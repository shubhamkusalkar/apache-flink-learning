# Apache Flink — Time & Watermarks: Deep Dive with Real-Life Examples

---

## The Core Problem: Time Is Complicated in Distributed Systems

Before anything else, understand **why time is hard**.

Imagine you run an online store. Customers click "Buy" all day. You want to know:

> "How many orders were placed between 2:00 PM and 3:00 PM?"

Simple question. But here's what actually happens:

- A customer in Mumbai clicks "Buy" at **2:47 PM** (their device's clock)
- Their mobile network is slow — the event reaches your Flink cluster at **3:02 PM**
- Your Flink job is processing it at **3:02 PM** (the machine's wall clock)
- Kafka ingested it at **2:49 PM** (Kafka's timestamp)

So — which "time" do you use?
- **2:47 PM** — when the customer actually placed the order (**event time**)
- **2:49 PM** — when your pipeline first saw it (**ingestion time**)
- **3:02 PM** — when Flink actually processed it (**processing time**)

This question is not trivial. Your answer determines whether the order counts in the 2–3 PM window or not. And it determines whether your reports are **correct** or just **fast**.

---

## Part 1: The Three Notions of Time

---

### 1. Event Time — "When it actually happened"

**Definition:** The timestamp embedded in the event itself, set by the originating device or application.

```
{ "orderId": "A123", "userId": "alice", "amount": 499, "eventTimestamp": "2024-03-15T14:47:00Z" }
                                                             ↑
                                                    Event Time lives HERE
                                                    (inside the event payload)
```

**How Flink uses it:**
```java
WatermarkStrategy
    .<Order>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((order, recordTimestamp) -> order.eventTimestamp);
```

You tell Flink: "Use this field from the event as the timestamp."

**Real-life analogy:** A **postmark on a letter**. The date stamped on the envelope is when the letter was written and posted — regardless of when it arrived at your house or when you opened it.

**Advantages:**
- Results are **deterministic** — reprocessing the same data gives the same result
- Correctly handles out-of-order data
- Historical data analysis works perfectly

**Disadvantages:**
- Requires waiting for late events → introduces latency
- Needs watermarks (explained below) to know when a time period is "complete"

**Best for:** Any scenario where event timing matters for business logic.
- Financial reports: "Revenue between 9 AM and 10 AM" should use when transactions occurred, not when servers processed them
- IoT analytics: Sensor readings should be analyzed based on measurement time
- User behavior: "Sessions between 8 PM and 9 PM" = when users actually used the app

---

### 2. Processing Time — "When Flink saw it"

**Definition:** The wall-clock time of the machine executing the Flink operator at the moment it processes an event.

```
Event arrives at Flink TaskManager at 3:02:15 PM
→ Processing Time = 3:02:15 PM
→ (regardless of what timestamp is inside the event)
```

**How Flink uses it:**
```java
stream
    .keyBy(e -> e.city)
    .window(TumblingProcessingTimeWindows.of(Duration.ofMinutes(5)))
    .sum("amount");
```

No timestamp extraction needed. Flink just uses `System.currentTimeMillis()`.

**Real-life analogy:** A **receptionist's log book**. They write down the time when a guest checks in, not when the guest left home. Two guests who left home at the same time but arrived at different times get logged at different times.

**Advantages:**
- Simplest to implement — no timestamp logic needed
- Lowest latency — no waiting for late events
- Best performance

**Disadvantages:**
- **Non-deterministic** — reprocessing the same data gives different results (because the wall clock is different)
- If a Flink job has backpressure, events arriving late due to system slowness get placed in wrong windows
- If you replay historical data, all events get dumped into "now" windows

**Best for:**
- Approximate real-time metrics where slight inaccuracy is fine
- Monitoring dashboards where you want "what's happening right now"
- Cases where your events have no embedded timestamps

**When it breaks down (critical example):**

```
Real scenario:
  - Event was produced at 2:58 PM
  - Kafka consumer lag = 5 minutes (system under load)
  - Event arrives at Flink at 3:03 PM
  - Processing time window says: "This belongs to the 3:00-3:05 PM window"
  - But the event actually happened in the 2:55-3:00 PM window!

Result: Your 2:55-3:00 PM window is missing data. Your 3:00-3:05 PM window has phantom data.
Report is WRONG.
```

This is why for any business-critical reporting, you should use event time.

---

### 3. Ingestion Time — "When Flink's source first saw it"

**Definition:** The timestamp assigned by the Flink **source operator** when the event first enters the Flink pipeline (not when an upstream system like Kafka received it, not when a downstream operator processes it).

```
Kafka message created at:   2:47 PM  (Kafka's timestamp)
Flink source reads it at:   2:49 PM  ← Ingestion Time assigned HERE
Flink operator processes:   2:49 PM  (same as ingestion time, propagated)
```

**How Flink uses it:**
```java
WatermarkStrategy.forMonotonousTimestamps()
// Combined with: source assigns ingestion timestamp automatically
```

Ingestion time is essentially processing time at the source — but the timestamp is fixed once and propagated through the entire pipeline consistently.

**Real-life analogy:** A **warehouse receiving dock**. The moment a truck arrives and items are logged, a "received at" timestamp is stamped on every item. Whether that item sits in the warehouse for 2 hours before a worker processes it, the item's official received time is when the truck docked — not when the worker touched it.

**Advantages:**
- More consistent than pure processing time (no skew from operator processing delays)
- Simpler than event time (no watermarks needed)
- Automatic watermarks (always monotonically increasing)

**Disadvantages:**
- Still wrong if Kafka has consumer lag (events that were produced long ago all get "now" as ingestion time)
- Reprocessing historical data doesn't work correctly

**When to use:** A middle ground when you don't have reliable event timestamps but still want consistent behavior across operators.

---

### Visual Comparison

```
TIMELINE:
  2:47 PM ──── 2:49 PM ──────────────────── 3:02 PM
    │              │                            │
    │              │                            │
  Event          Kafka / Source             Flink Operator
  Created        Ingested                   Processes It

  Event Time =   2:47 PM  (timestamp inside the event payload)
  Ingestion Time = 2:49 PM  (when Flink source first saw it)
  Processing Time = 3:02 PM  (when the operator ran on it)
```

---

### Quick Decision Guide

| Question | Use |
|----------|-----|
| Do my events have reliable timestamps? | Event Time |
| Do I need consistent, replayable results? | Event Time |
| Do I care about business time (when things happened)? | Event Time |
| Do I just want "what's happening right now" dashboards? | Processing Time |
| Am I prototyping and don't have timestamps? | Processing Time |
| My events have no timestamp but I want cross-operator consistency? | Ingestion Time |

> **Rule of thumb:** 90% of production use cases should use Event Time. Processing Time is for quick dashboards and prototypes.

---

## Part 2: Watermarks — The Most Important Concept in Flink Time

This is the hardest concept to grasp but also the most important. Once you truly understand watermarks, everything else clicks.

---

### The Core Problem Watermarks Solve

You're using event time. Events are flowing in. You're computing a window from **2:00 PM to 3:00 PM**.

**How does Flink know when the 2:00–3:00 PM window is "complete"?**

In batch processing, you'd wait until ALL data is available. But in streaming, data comes in forever. You can't wait forever.

In a perfect world, events arrive in order:
```
2:00:01 → 2:00:02 → 2:00:03 → ... → 2:59:58 → 2:59:59 → 3:00:00
```
Easy! When you see the first event with timestamp ≥ 3:00:00, close the 2:00–3:00 window.

But in the real world, events arrive **out of order**:
```
2:45:00 → 2:47:00 → 2:44:00 → 2:49:00 → 2:43:00 → 3:02:00 → 2:48:00 → ...
```

When you see `3:02:00`, can you close the 2:00–3:00 window? Maybe. But `2:48:00` arrived AFTER `3:02:00`! If you'd already closed the window, you'd miss it.

**The dilemma:**
- Close windows too early → miss late events → wrong results
- Never close windows → never get results → useless

**Watermarks are the solution.** They are Flink's mechanism for saying:

> "I am confident that all events with timestamp ≤ T have arrived. Time has progressed to T. Windows ending at or before T can now fire safely."

---

### What Exactly Is a Watermark?

A watermark is a **special marker** — a timestamp value — that flows **inline with your data** through the Flink pipeline.

```
Data stream:

... [event ts=2:47] [event ts=2:45] [WATERMARK t=2:40] [event ts=2:50] [WATERMARK t=2:45] ...
```

`WATERMARK t=2:40` means: **"I guarantee no more events with timestamp ≤ 2:40 will ever arrive."**

When an operator's watermark advances past a window's end time, that window fires.

```
Window: [2:00 PM – 2:30 PM]

Data arrives:
  event ts=2:15  → add to window
  event ts=2:25  → add to window
  WATERMARK t=2:35  → watermark > window end (2:30) → FIRE the window!
  event ts=2:28  → TOO LATE (window already fired) → late event
```

---

### Real-Life Analogy: The Train Station Departure Board

Imagine you're a train station manager. Trains are scheduled to depart at :00 every hour. Before a train departs, you wait for all passengers booked on that train to board.

But you can't wait forever — someone might be stuck in traffic.

So you use an **announcement system** (the watermark):

> "The boarding gate for the 2:00 PM train will close at 2:05 PM."

This is your watermark: `WATERMARK = 2:05 PM (departure time + 5 minute grace)`

- If a passenger arrives at 2:03 PM → they board (event within allowed lateness)
- If a passenger arrives at 2:07 PM → they missed the train (event is too late)
- At 2:05 PM → train departs (window fires) regardless

The "5 minute grace" = your allowed out-of-orderness tolerance.

---

### How Watermarks Are Generated

Flink generates watermarks at the **source**, and they flow downstream with the data.

#### Strategy 1: Bounded Out-of-Orderness (Most Common)

"I expect events to be at most X time late."

```java
WatermarkStrategy
    .<SensorReading>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((event, ts) -> event.timestamp);
```

**What this does:**
- Flink tracks the maximum event timestamp seen so far: `maxTimestamp`
- Watermark = `maxTimestamp - 5 seconds`

```
Events arrive:
  ts=10:00:01 → maxTs=10:00:01, watermark=09:59:56
  ts=10:00:05 → maxTs=10:00:05, watermark=10:00:00
  ts=10:00:03 → maxTs=10:00:05, watermark=10:00:00 (out of order, ignored for max)
  ts=10:00:10 → maxTs=10:00:10, watermark=10:00:05

When watermark=10:00:05 fires:
  → Window [09:59:00 - 10:00:00] fires (watermark 10:00:05 > window end 10:00:00)
```

**Real-life use case:** IoT sensors in a factory. Sensor readings might arrive 3–5 seconds late due to network jitter. Use `forBoundedOutOfOrderness(Duration.ofSeconds(5))`.

---

#### Strategy 2: Monotonous Timestamps (Strictly Ordered Data)

"My events always arrive in order. No late events."

```java
WatermarkStrategy.forMonotonousTimestamps()
    .withTimestampAssigner((event, ts) -> event.timestamp);
```

Watermark = every event's timestamp. No delay. Windows fire as soon as their end timestamp passes.

**Real-life use case:** Database CDC (Change Data Capture). Database transaction logs are guaranteed to be in order (transaction IDs are sequential). No out-of-orderness.

---

#### Strategy 3: Custom Watermark Generator

You have full control over when and how watermarks are emitted.

```java
// Periodic watermark: emitted every 200ms (default)
public class BoundedOutOfOrdernessGenerator
    implements WatermarkGenerator<MyEvent> {

    private long maxTimestamp = Long.MIN_VALUE;
    private final long maxOutOfOrderness = 5000; // 5 seconds

    @Override
    public void onEvent(MyEvent event, long eventTimestamp, WatermarkOutput output) {
        maxTimestamp = Math.max(maxTimestamp, eventTimestamp);
        // Don't emit watermark here — emit periodically
    }

    @Override
    public void onPeriodicEmit(WatermarkOutput output) {
        // Emit watermark every 200ms (configured interval)
        output.emitWatermark(new Watermark(maxTimestamp - maxOutOfOrderness - 1));
    }
}
```

**Periodic vs Punctuated emission:**

| Type | When emitted | Use case |
|------|-------------|----------|
| **Periodic** | Every N milliseconds (configurable, default 200ms) | Most cases — regular watermark progress |
| **Punctuated** | After every specific event (e.g., events with a "flush" flag) | Special protocols where events themselves signal time progress |

---

### The Watermark Equation — Memorize This

```
Watermark = max(event_timestamp_seen_so_far) - allowed_lateness
```

- **Higher allowed lateness** → watermark lags further behind → windows fire later → more late events captured → higher latency
- **Lower allowed lateness** → watermark stays close to real time → windows fire quickly → more events missed → lower latency

**It's always a trade-off: Correctness vs Latency.**

---

## Part 3: Watermarks in Parallel Streams — The Minimum Rule

This is where things get interesting. Flink runs in parallel — multiple sources, multiple operators.

### Single Source, Multiple Partitions

```
Kafka Topic: "orders" with 4 partitions

Source Subtask 1 (reads partition 0):  generates its own watermark W1
Source Subtask 2 (reads partition 1):  generates its own watermark W2
Source Subtask 3 (reads partition 2):  generates its own watermark W3
Source Subtask 4 (reads partition 3):  generates its own watermark W4
```

Each source partition has its own stream of events. Partition 0 might be receiving orders from fast US servers. Partition 3 might be receiving orders from a slow server in a remote region.

### The Minimum Watermark Rule

When a downstream operator (like a window operator) receives from **multiple input streams**, it sets its event time to the **minimum** watermark across all inputs.

```
Downstream Join/Window Operator receives from:
  Source Subtask 1: Watermark = 10:05:00
  Source Subtask 2: Watermark = 10:03:00  ← this one is behind
  Source Subtask 3: Watermark = 10:06:00
  Source Subtask 4: Watermark = 10:04:00

Operator's effective watermark = min(10:05, 10:03, 10:06, 10:04) = 10:03:00
```

**The operator can only advance to 10:03:00.** Even though subtask 1 has advanced to 10:05, the operator won't fire any window that ends after 10:03, because subtask 2 might still send events with timestamps before 10:03.

### Real-Life Analogy: A Meeting That Can't Start Until Everyone Arrives

You're hosting a 4-person meeting. The rule is: "We start when everyone is seated."

- Person 1 is seated at 10:05
- Person 2 is seated at 10:03 ← slowest
- Person 3 is seated at 10:06
- Person 4 is seated at 10:04

Meeting starts at **10:03** — when the last person sat down (the minimum).

The meeting (window computation) waits for the slowest participant (the slowest input stream's watermark).

---

### The Idle Source Problem

What if one of your Kafka partitions goes completely silent? No events, no watermarks.

```
Source Subtask 1: Watermark = 10:05:00 → advances to 10:06, 10:07...
Source Subtask 2: Watermark = 10:03:00 → STUCK! No new events
Source Subtask 3: Watermark = 10:06:00 → advances to 10:07, 10:08...
Source Subtask 4: Watermark = 10:04:00 → advances to 10:05, 10:06...

Downstream watermark = min(10:07, 10:03, 10:08, 10:06) = 10:03 → FOREVER STUCK
```

The entire downstream pipeline freezes because one idle partition doesn't advance its watermark. Windows never fire.

**Solution: Mark idle sources**

```java
WatermarkStrategy
    .<Order>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((e, ts) -> e.timestamp)
    .withIdleness(Duration.ofMinutes(1));  // ← if no events for 1 minute, mark as idle
```

When marked idle, that subtask's watermark is **excluded** from the minimum calculation. The other subtasks' watermarks can advance freely.

```
After idleness timeout:
  Source Subtask 2 marked as IDLE → excluded from minimum

  Downstream watermark = min(10:07, [IDLE], 10:08, 10:06) = 10:06 → advances!
```

**Real-life use case:** A retail chain with 100 stores. Each store is a Kafka partition. At night, many stores close and stop sending events. Without idle source handling, your morning report windows would never fire because the closed stores' partitions are stuck. With `withIdleness`, closed-store partitions are ignored and other stores' data flows through.

---

### How Watermarks Propagate Through the Graph

```
SOURCE OPERATORS
├── Source 1: W=10:05 ─────────────────────────────────┐
├── Source 2: W=10:03 ────────────────────────────────┐ │
│                                                      ↓ ↓
INTERMEDIATE OPERATORS                            [  Map  ]   W = min(10:05, 10:03) = 10:03
│                                                      │
│                                                      ↓
WINDOW OPERATOR                               [  Window  ]    fires when W > window_end
│                                                      │
│                                                      ↓
SINK                                              [  Sink  ]
```

At each operator:
1. Operator advances its event-time clock to the received watermark
2. Checks if any windows can now fire (is watermark > window end?)
3. Fires eligible windows
4. Forwards the watermark downstream

---

## Part 4: Event Time Windows in Action

Now that you understand watermarks, let's see how they power event time windows.

### Tumbling Event Time Window — Step by Step

```java
stream
    .keyBy(e -> e.city)
    .window(TumblingEventTimeWindows.of(Duration.ofMinutes(5)))
    .sum("sales");
```

**Scenario: Sales data for city "NYC"**

```
Events arrive (out of order):
  ts=10:02, sales=100  → assigned to window [10:00–10:05]
  ts=10:07, sales=200  → assigned to window [10:05–10:10]
  ts=10:03, sales=150  → assigned to window [10:00–10:05]  (late but within watermark lag)
  ts=10:01, sales=80   → assigned to window [10:00–10:05]
  ts=10:11, sales=300  → assigned to window [10:10–10:15]

Watermark progression (5-second lag):
  After ts=10:07: watermark = 10:02
  After ts=10:11: watermark = 10:06

When watermark reaches 10:06:
  → 10:06 > window end 10:05 → window [10:00–10:05] FIRES
  → Result: NYC sales = 100 + 150 + 80 = 330

Window [10:05–10:10] stays open — watermark hasn't passed 10:10 yet.
```

---

### Event Time vs Processing Time — The Critical Difference

**Same data, different results:**

```
Real event times:
  09:59:30 — event A (produced just before 10:00)
  10:00:15 — event B
  09:59:45 — event C (produced before 10:00, arrived late)
  10:00:30 — event D

Kafka consumer lag: 2 minutes
All events processed by Flink at 10:02 PM
```

**Processing Time window [10:00–10:05]:**
```
All 4 events are processed at ~10:02 → all 4 fall in [10:00-10:05] window
Window [09:55-10:00] = EMPTY (wrong! A and C should be here)
Window [10:00-10:05] = A + B + C + D (wrong! A and C happened before 10:00)
```

**Event Time window [10:00–10:05]:**
```
A (ts=09:59:30) → [09:55–10:00] window
B (ts=10:00:15) → [10:00–10:05] window
C (ts=09:59:45) → [09:55–10:00] window
D (ts=10:00:30) → [10:00–10:05] window

Window [09:55-10:00] = A + C ✓ (correct!)
Window [10:00-10:05] = B + D ✓ (correct!)
```

**Event time gives you the right answer regardless of when events arrive.**

---

## Part 5: Late Events — What Happens After a Window Fires?

Even with the best watermark strategy, some events arrive after their window has fired. This is called a **late event**.

### Without Any Lateness Handling

```
Watermark passes 10:05 → window [10:00–10:05] fires with data: {A, B, C}
Event arrives: ts=10:03 (late — watermark already at 10:07)
→ EVENT IS DROPPED SILENTLY
```

This is the default behavior. Fine for some use cases, problematic for others.

---

### With Allowed Lateness

```java
stream
    .keyBy(e -> e.city)
    .window(TumblingEventTimeWindows.of(Duration.ofMinutes(5)))
    .allowedLateness(Duration.ofMinutes(2))   // keep window alive for 2 extra minutes
    .sum("sales");
```

```
Watermark passes 10:05 → window [10:00–10:05] FIRST FIRING → emit result
  Result: NYC sales = 330

Watermark = 10:06 → late event ts=10:03 arrives → window still alive (10:07 < 10:05+2min)
  → UPDATE: NYC sales = 330 + 50 = 380 → LATE FIRING (second result emitted)

Watermark = 10:07 → another late event ts=10:04 arrives → window still alive
  → UPDATE: NYC sales = 380 + 70 = 450 → LATE FIRING (third result emitted)

Watermark = 10:07:01 → window [10:00–10:05] DESTROYED (10:07:01 > 10:05 + 2min)
  → Any events with ts in [10:00–10:05] from now on are truly dropped
```

**Important:** With `allowedLateness`, your downstream system will receive **multiple results for the same window**. You must handle this — either by treating each result as an "update/correction" or by using upsert semantics in your sink.

---

### With Side Outputs for Very Late Events

For events that arrive even after the allowed lateness period, capture them in a separate stream instead of silently dropping them:

```java
OutputTag<SalesEvent> lateTag = new OutputTag<SalesEvent>("late-sales"){};

SingleOutputStreamOperator<CitySales> mainStream = stream
    .keyBy(e -> e.city)
    .window(TumblingEventTimeWindows.of(Duration.ofMinutes(5)))
    .allowedLateness(Duration.ofMinutes(2))
    .sideOutputLateData(lateTag)   // capture events beyond allowed lateness
    .sum("sales");

// Main stream: on-time + slightly-late results
DataStream<CitySales> sales = mainStream;

// Side stream: very late events (more than 2 minutes late)
DataStream<SalesEvent> veryLateEvents = mainStream.getSideOutput(lateTag);
veryLateEvents.addSink(new ReconciliationDatabase()); // handle separately
```

**Real-life use case:** A retail chain's daily sales report.
- On-time events (within 30 min) → normal daily totals
- Slightly late events (within 2 hours) → correction to daily totals
- Very late events (next day or later) → manual reconciliation queue for accountants

---

## Part 6: Watermark Strategies — Choosing the Right One

### Strategy Comparison

| Strategy | When watermark advances | Use case |
|----------|------------------------|----------|
| `forMonotonousTimestamps()` | After every event (no lag) | Ordered sources: DB CDC, ordered Kafka topics |
| `forBoundedOutOfOrderness(D)` | maxTimestamp - D | Most cases: IoT, app events, user clicks |
| Custom periodic | Every N ms based on your logic | Complex scenarios with domain-specific knowledge |
| Custom punctuated | Only on specific events | Protocol-driven sources with explicit "progress" markers |

---

### Configuring Watermark Interval

How often Flink emits periodic watermarks:

```java
env.getConfig().setAutoWatermarkInterval(200); // emit watermark every 200ms (default)
```

Lower interval = watermarks advance faster = lower latency but more overhead.
Higher interval = watermarks advance slower = higher latency but less overhead.

**Tuning guidance:**
- Default (200ms) is fine for most applications
- For very high-throughput (millions/sec), increase to 500ms–1s to reduce watermark overhead
- For very low-latency requirements, decrease to 50ms

---

## Part 7: Putting It All Together — A Complete Real-World Scenario

### Scenario: Real-Time Taxi Ride Analytics (Uber-like system)

**Business requirements:**
- Count completed rides per city every 5 minutes
- Handle GPS event delays (rides can complete their data up to 30 seconds late)
- Handle occasional very late events (driver app goes offline and reconnects)
- Multiple Kafka partitions (one per region); some regions have low traffic at night

```java
// Step 1: Define watermark strategy
WatermarkStrategy<RideEvent> watermarkStrategy =
    WatermarkStrategy
        .<RideEvent>forBoundedOutOfOrderness(Duration.ofSeconds(30))  // 30s out-of-orderness
        .withTimestampAssigner((ride, ts) -> ride.rideCompletedAt)    // use ride completion time
        .withIdleness(Duration.ofMinutes(5));                         // handle inactive regions

// Step 2: Read from Kafka
DataStream<RideEvent> rides = env.fromSource(
    kafkaSource,
    watermarkStrategy,
    "Ride Events"
);

// Step 3: Late events side output tag
OutputTag<RideEvent> lateRidesTag = new OutputTag<RideEvent>("late-rides"){};

// Step 4: Window and aggregate
SingleOutputStreamOperator<CityRideCount> rideCounts = rides
    .keyBy(ride -> ride.city)
    .window(TumblingEventTimeWindows.of(Duration.ofMinutes(5)))
    .allowedLateness(Duration.ofMinutes(2))      // accept events up to 2 min late
    .sideOutputLateData(lateRidesTag)            // capture very late events
    .aggregate(
        new RideCountAggregator(),               // count rides per city (memory efficient)
        new EnrichWithWindowTimestamp()          // add window start/end to output
    );

// Step 5: Handle very late rides
rideCounts.getSideOutput(lateRidesTag)
          .addSink(new LateRideReconciliationSink());

// Step 6: Send results to dashboard
rideCounts.addSink(new DashboardSink());
```

**What happens with a real sequence of events:**

```
Events received by Flink (out of order due to GPS lag):

10:02:15 - Ride A completed in NYC (ts=10:02:15) → window [10:00–10:05]
10:02:30 - Ride B completed in NYC (ts=10:02:10) → window [10:00–10:05] ← out of order
10:03:00 - Ride C completed in LA  (ts=10:02:55) → LA window [10:00–10:05]
10:03:30 - Ride D completed in NYC (ts=10:03:20) → window [10:00–10:05]
10:05:45 - Ride E completed in NYC (ts=10:05:30) → window [10:05–10:10]

Watermark progression (30s lag):
  After Ride A: W = 10:02:15 - 30s = 10:01:45
  After Ride B: W = 10:02:15 - 30s = 10:01:45 (10:02:10 < maxTs, ignored)
  After Ride D: W = 10:03:20 - 30s = 10:02:50
  After Ride E: W = 10:05:30 - 30s = 10:05:00

When W = 10:05:00:
  → W > window end 10:05:00? No, not yet (equal, not greater)

10:06:05 - Ride F completed in NYC (ts=10:05:50)
  W = 10:05:50 - 30s = 10:05:20
  → W (10:05:20) > window end (10:05:00) → FIRE window [10:00–10:05]
  → NYC rides in 10:00–10:05 = A + B + D = 3 rides emitted to dashboard

10:06:45 - Late arrival: Ride G (ts=10:04:30) arrives (GPS delay)
  → ts=10:04:30 is within window [10:00–10:05] that already fired
  → But allowed lateness = 2 min, and W=10:05:20 < 10:05:00 + 2min = 10:07:00
  → Window [10:00–10:05] still alive → LATE FIRING: NYC rides = 3 + 1 = 4
  → Dashboard receives correction: "NYC 10:00-10:05: 4 rides (updated)"

10:08:00 - Very late arrival: Ride H (ts=10:03:00)
  → W is now past 10:07:00 → window [10:00–10:05] is DESTROYED
  → Ride H goes to SIDE OUTPUT → ReconciliationSink handles it
```

---

## Part 8: Common Pitfalls and How to Avoid Them

---

### Pitfall 1: Watermark Never Advances (Silent Window Stall)

**Symptom:** Windows never fire. Your application produces no output.

**Cause:** No events are flowing, so no new max timestamps, so watermark doesn't advance.

**Solution:**
```java
// Always configure idleness timeout
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withIdleness(Duration.ofMinutes(1)); // ← this saves you
```

---

### Pitfall 2: Wrong Timestamp Extractor

**Symptom:** All events go to windows in 1970 (Unix epoch time 0) or windows never fire.

**Cause:** Returning milliseconds where Flink expects milliseconds (correct) but your field is in seconds.

```java
// WRONG: if eventTimestamp is in SECONDS
.withTimestampAssigner((e, ts) -> e.eventTimestamp);
// This gives 1710000000 (seconds) instead of 1710000000000 (ms) → year 1970!

// CORRECT: convert to milliseconds
.withTimestampAssigner((e, ts) -> e.eventTimestamp * 1000L);
```

---

### Pitfall 3: Watermark Lag Too Small — Many Events Dropped

**Symptom:** Your results look correct but you're missing ~5-10% of events.

**Cause:** Your `forBoundedOutOfOrderness` duration is too small for your actual network jitter.

**Solution:**
1. Measure actual lateness distribution in your data:
```java
// Log the actual out-of-orderness you observe
long lag = maxSeenTimestamp - event.timestamp; // how late is this event?
```
2. Set your `forBoundedOutOfOrderness` to the 99th percentile of observed lag.
3. Use `sideOutputLateData` to monitor how many events are still late after your watermark.

---

### Pitfall 4: Processing Time Windows in Production Reporting

**Symptom:** Your hourly sales report shows slightly different numbers every time you rerun it.

**Cause:** You're using processing time windows. Reprocessing historical data reassigns events to different windows (based on when you reran, not when events happened).

**Solution:** Always use event time for business reporting.

---

### Pitfall 5: Forgetting to Handle Multiple Results from Late Firings

**Symptom:** Your downstream database has duplicate or conflicting records for the same window.

**Cause:** With `allowedLateness`, the same window fires multiple times.

**Solution:** Use upsert semantics in your sink:
```java
// Instead of INSERT, use INSERT OR UPDATE (upsert) by window key
// Or use a MERGE statement
// Or use a downstream deduplication step
```

---

## Summary Cheat Sheet

### The Three Times

| Time Type | What it is | Deterministic? | Use case |
|-----------|-----------|---------------|----------|
| **Event Time** | Timestamp in the event | ✅ Yes | Business reporting, correctness-critical |
| **Processing Time** | Server wall clock when processed | ❌ No | Live dashboards, low latency |
| **Ingestion Time** | When source first saw the event | Partially | When events have no timestamps |

### Watermark Formula

```
Watermark = max(event_timestamp_seen) - allowed_out_of_orderness
```

### Watermarks in Parallel

```
Downstream watermark = min(watermark across all input subtasks)
```

### Key Rules

1. **Watermarks flow inline with data** — they're not a side channel, they flow in the stream itself.
2. **Watermark = promise** — "I guarantee no events with timestamp ≤ watermark will arrive after this."
3. **Window fires when watermark > window end time** — not when the window's time is reached on the wall clock.
4. **Minimum watermark rule** — a slow input stream blocks ALL downstream windows, not just its own.
5. **Always set idleness** — silent partitions will freeze your entire pipeline without it.
6. **Lateness vs watermark lag are different settings:**
   - `forBoundedOutOfOrderness(D)` — how much to delay watermarks to absorb out-of-order events
   - `allowedLateness(L)` — how long to keep a window alive after it first fires
7. **Higher watermark lag = lower latency risk of dropping events, but higher output latency.**
8. **Event time is always correct; processing time is always fast — choose based on your requirement.**

### Decision Flowchart

```
Do your events have reliable timestamps?
  ├── YES → Use Event Time
  │     ├── How late can events arrive? → Set forBoundedOutOfOrderness(D)
  │     ├── Need to catch even later events? → Set allowedLateness(L)
  │     └── What to do with very late events? → sideOutputLateData(tag)
  └── NO
        ├── Need consistent cross-operator behavior? → Ingestion Time
        └── Just want "right now" metrics? → Processing Time
```
