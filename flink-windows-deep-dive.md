# Apache Flink Windows — Deep Dive with Real-Life Examples

---

## The Core Concept: Why Do We Need Windows?

Imagine you work at a toll booth on a highway. Cars keep coming — one after another, infinitely. Your boss asks:

> "How many cars passed every hour?"
> "What's the average speed of cars in the last 10 minutes?"

You can't answer that by looking at ONE car. You need to group cars over a time period, then compute. That grouping is exactly what a **Window** does in stream processing.

A Window takes an infinite stream of events and carves it into finite, bounded chunks so you can run computations on them.

---

## Step 1: Keyed vs Non-Keyed Windows

Before windowing, you decide: do you want to process groups separately, or everything together?

### Keyed Windows — Process each group independently

```java
stream
  .keyBy(event -> event.userId)   // group by user
  .window(...)
  .reduce(...);
```

**Real-life analogy:** A bank has thousands of customers. You want to compute each customer's total transactions per day. You `keyBy(customerId)` so each customer's events go to their own parallel bucket. 10,000 customers → 10,000 parallel windows running simultaneously.

### Non-Keyed Windows — Everything in one bucket

```java
stream
  .windowAll(...)   // no keyBy
  .reduce(...);
```

**Real-life analogy:** A traffic camera counts ALL vehicles passing through one intersection every 5 minutes, regardless of vehicle type. No grouping needed — just count everything together.

> **Warning:** This runs on a single machine with `parallelism=1`, so use only for low-volume global aggregations.

---

## Step 2: Window Assigners (The 4 Window Types)

This is the heart of windows. The assigner decides: **which window does each event belong to?**

---

### 1. Tumbling Windows — Non-overlapping, fixed-size buckets

```
Time:  |--0-1-2-3-4--|--5-6-7-8-9--|--10-11-12--|
         Window 1       Window 2      Window 3
```

Each event belongs to exactly **one** window. Windows don't overlap.

```java
input.keyBy(e -> e.city)
     .window(TumblingEventTimeWindows.of(Duration.ofHours(1)))
     .sum("sales");
```

**Real-life use cases:**
- **Hourly sales reports:** "Total revenue per city every hour" — each hour is one window, completely independent of the next.
- **Daily active users:** Count unique logins per day. Midnight resets the window.
- **Billing cycles:** Telecom companies compute data usage in 30-day tumbling windows for billing.
- **Stock market OHLC bars:** Open/High/Low/Close prices for every 1-minute candle — each minute is a tumbling window.

**When to use:** When you want clean, non-overlapping time periods with no event double-counting.

---

### 2. Sliding Windows — Overlapping windows with a slide interval

```
Window size = 10 min, Slide = 5 min

Time:  0----5----10----15----20
       [----W1----]
            [----W2----]
                 [----W3----]
```

An event can belong to **multiple windows** simultaneously. More computationally expensive but gives smoother aggregations.

```java
input.keyBy(e -> e.sensor)
     .window(SlidingEventTimeWindows.of(
         Duration.ofMinutes(10),   // window size
         Duration.ofMinutes(5)))   // slide interval
     .average("temperature");
```

**Real-life use cases:**
- **Moving average stock price:** "What's the average stock price over the last 10 minutes, updated every 1 minute?" — classic sliding window. Financial dashboards use this everywhere.
- **Health monitoring:** A smartwatch computes your average heart rate over the last 5 minutes, updated every 30 seconds. If it spikes, alert immediately.
- **Fraud detection:** "Has this card done more than 5 transactions in any 10-minute rolling window?" — a transaction belongs to multiple overlapping 10-min windows.
- **Network DDoS detection:** Count packets from an IP in the last 60 seconds, checked every 10 seconds.

**When to use:** When you need smooth, continuously-updated aggregations and can tolerate that one event contributes to multiple results.

---

### 3. Session Windows — Gaps of inactivity define boundaries

```
Events:  e1 e2   e3     [gap > 5min]   e4 e5 e6  [gap > 5min]  e7
         [---Session 1---]              [--Session 2--]           [S3]
```

No fixed size. A session starts when an event arrives, ends when nothing arrives for longer than the gap duration. **Variable length.**

```java
input.keyBy(e -> e.userId)
     .window(EventTimeSessionWindows.withGap(Duration.ofMinutes(30)))
     .sum("pageviews");
```

**Real-life use cases:**
- **Web analytics (classic use case):** A user visits your website. They browse pages, then go idle for 30 minutes → session ends. This is literally how Google Analytics defines a session! You compute pages viewed, time on site, etc. per session.
- **ATM transactions:** A customer uses an ATM, does a few operations, then leaves. If no activity for 2 minutes → session closed. Detect if the same card reopens a session from a different ATM 5 minutes later (fraud!).
- **Gaming sessions:** Player logs in, plays for a while, goes AFK for 15 min → session closed. Compute score, duration, achievements per game session.
- **Call center:** A customer service call with pauses — group all interactions within a 5-minute gap as one session.

**Dynamic gap example** (different users get different gaps):

```java
.window(EventTimeSessionWindows.withDynamicGap(element -> {
    // VIP users get a 60-minute session timeout
    // Regular users get 20 minutes
    return element.isVIP ? 3_600_000 : 1_200_000;
}))
```

**When to use:** When activity has natural start/end patterns based on user behavior, not clock time.

---

### 4. Global Windows — One window for all time

```
Events:  e1 e2 e3 e4 e5 e6 e7 e8 ...
         [----------All in one window-----------]
```

All events with the same key go into a single, never-ending window. Requires a **custom trigger** because it has no natural end.

```java
input.keyBy(e -> e.userId)
     .window(GlobalWindows.create())
     .trigger(new CountTrigger<>(100))  // fire every 100 events
     .process(new MyWindowFunction());
```

**Real-life use cases:**
- **Process every N events:** A sensor sends readings; trigger an alert after exactly 1000 readings regardless of time.
- **Batch-style processing on streams:** Process every 500 orders together for bulk discount calculation.
- **Event-count-based billing:** Bill a customer every 1000 API calls, regardless of when they happen.

**When to use:** Rarely. Only when your window boundary is defined by count or custom logic, not time.

---

## Step 3: Window Functions — What to compute inside a window

Once you have a window, you need to compute something. Three options, each with different trade-offs.

---

### ReduceFunction — Incremental, memory-efficient

Processes elements two at a time, keeps only the running result. Like a running total.

```java
// Real example: Running total of sales per city
input.keyBy(e -> e.city)
     .window(TumblingEventTimeWindows.of(Duration.ofHours(1)))
     .reduce((sale1, sale2) -> new Sale(sale1.city, sale1.amount + sale2.amount));
```

**How it works internally:**
```
Event 1: city=NYC, amount=100  → state: (NYC, 100)
Event 2: city=NYC, amount=200  → state: (NYC, 300)   ← reduce called
Event 3: city=NYC, amount=50   → state: (NYC, 350)   ← reduce called
Window fires → emit (NYC, 350)
```

**Memory:** Only stores ONE value in state, no matter how many events arrive. Extremely efficient.

**When to use:** Sum, min, max, count — any commutative associative operation where input type = output type.

---

### AggregateFunction — Incremental, flexible types

Like `ReduceFunction` but your accumulator can be a completely different type from input/output.

```java
// Real example: Average order value per product category
class AverageAggregate
    implements AggregateFunction<Order, Tuple2<Double, Long>, Double> {

    // Accumulator: (sum, count)
    public Tuple2<Double, Long> createAccumulator() {
        return Tuple2.of(0.0, 0L);
    }

    public Tuple2<Double, Long> add(Order order, Tuple2<Double, Long> acc) {
        return Tuple2.of(acc.f0 + order.amount, acc.f1 + 1);
    }

    public Double getResult(Tuple2<Double, Long> acc) {
        return acc.f0 / acc.f1;  // compute average at the end
    }

    public Tuple2<Double, Long> merge(Tuple2<Double, Long> a, Tuple2<Double, Long> b) {
        return Tuple2.of(a.f0 + b.f0, a.f1 + b.f1);
    }
}
```

**Real-life use cases:**
- Average order value per category per hour
- Percentage of failed requests (count failures / count total)
- Standard deviation of sensor readings

**Memory:** Also only stores one accumulator value. Very efficient.

---

### ProcessWindowFunction — Full access, all elements buffered

Gets access to **ALL elements** in the window at once, plus metadata (window start/end time, watermark, state). Most powerful but most memory-intensive.

```java
// Real example: Detect the peak traffic minute within a 1-hour window
public class PeakTrafficDetector
    extends ProcessWindowFunction<PageView, String, String, TimeWindow> {

    @Override
    public void process(String page, Context ctx,
                        Iterable<PageView> views, Collector<String> out) {

        // Access window metadata
        long windowStart = ctx.window().getStart();
        long windowEnd = ctx.window().getEnd();

        // Need ALL elements to find the peak minute
        Map<Long, Integer> minuteCounts = new HashMap<>();
        for (PageView view : views) {
            long minute = view.timestamp / 60_000;
            minuteCounts.merge(minute, 1, Integer::sum);
        }

        long peakMinute = Collections.max(minuteCounts.entrySet(),
                          Map.Entry.comparingByValue()).getKey();

        out.collect("Page: " + page +
                    " | Window: " + windowStart + "-" + windowEnd +
                    " | Peak minute: " + peakMinute);
    }
}
```

**Real-life use cases:**
- Find the top-N most purchased items in a window (need all elements to rank)
- Detect anomalies that require comparing all readings together
- Computing percentiles (median, p99 latency) — you need all data points

> **Memory warning:** If 1 million events arrive in a window, all 1 million are held in memory until the window fires. Use with caution on high-volume streams.

---

### Best of Both: AggregateFunction + ProcessWindowFunction

```java
// Real example: Average temperature per sensor + include window timestamp in output
input.keyBy(e -> e.sensorId)
     .window(TumblingEventTimeWindows.of(Duration.ofMinutes(5)))
     .aggregate(
         new AverageAggregate(),          // incremental — only accumulator in memory
         new EnrichWithWindowTime()        // gets the pre-aggregated average + window metadata
     );

class EnrichWithWindowTime
    extends ProcessWindowFunction<Double, SensorReport, String, TimeWindow> {

    public void process(String sensorId, Context ctx,
                        Iterable<Double> avgIterable, Collector<SensorReport> out) {
        Double avg = avgIterable.iterator().next(); // only 1 element — the aggregated result
        out.collect(new SensorReport(sensorId, avg, ctx.window().getEnd()));
    }
}
```

> **Recommended pattern** when you need window metadata (timestamps) AND efficient memory usage. The `AggregateFunction` handles incremental aggregation; `ProcessWindowFunction` just enriches the final result.

---

## Step 4: Triggers — When does a window fire?

Default behavior: windows fire when the watermark passes the window end time. But you can customize this.

**TriggerResult options:**

| Result | Behavior |
|--------|----------|
| `CONTINUE` | Do nothing yet |
| `FIRE` | Compute and emit results (keep the window) |
| `PURGE` | Clear window contents (don't emit) |
| `FIRE_AND_PURGE` | Emit AND clear |

**Real-life use cases for custom triggers:**

**CountTrigger — Fire every N events regardless of time:**
```java
// E-commerce: process orders in micro-batches of 100
input.keyBy(e -> e.region)
     .window(GlobalWindows.create())
     .trigger(new CountTrigger<>(100))
     .process(new BulkOrderProcessor());
```

**Custom trigger — Fire early every minute but also fire at window end:**
```java
// Dashboard: Show live preview every 60 seconds, final result at window end
// This is what "early firings" patterns implement
```

**PurgingTrigger — Wrap any trigger to clear elements after firing:**
```java
// Fire every 50 events AND clear the window (don't accumulate)
.trigger(new PurgingTrigger<>(new CountTrigger<>(50)))
```

---

## Step 5: Evictors — Remove elements before/after computation

Evictors let you filter out elements from a window before the function runs.

```java
// Keep only the 100 most recent elements in the window
input.keyBy(e -> e.sensor)
     .window(GlobalWindows.create())
     .trigger(new CountTrigger<>(100))
     .evictor(new CountEvictor<>(100))   // only keep last 100
     .process(new SensorAnalyzer());
```

**Built-in evictors:**

| Evictor | Behavior | Use case |
|---------|----------|----------|
| `CountEvictor(N)` | Keep only the last N elements | Sliding window on latest N readings |
| `TimeEvictor(interval)` | Remove elements older than `max_ts - interval` | Keep only recent events regardless of window size |
| `DeltaEvictor(threshold, fn)` | Remove elements too different from the latest | Anomaly filtering — drop outliers before computing |

> **Warning:** Evictors prevent incremental aggregation (`ReduceFunction`/`AggregateFunction` can't pre-aggregate). All elements are buffered in memory. Use sparingly.

---

## Step 6: Allowed Lateness — Handling late-arriving events

In the real world, events arrive out of order. A mobile app event might arrive 2 minutes late due to network issues.

```
Watermark advances to 10:05:00

Event arrives: timestamp = 10:03:45  ← This is LATE (before watermark)
```

**Without allowed lateness:** The event is dropped silently.

**With allowed lateness:**
```java
input.keyBy(e -> e.userId)
     .window(TumblingEventTimeWindows.of(Duration.ofMinutes(5)))
     .allowedLateness(Duration.ofMinutes(2))   // keep window alive 2 extra minutes
     .reduce(new SumSales())
```

**What happens:**
1. Watermark reaches window end (e.g., `10:05:00`) → **First firing** (regular result)
2. Late event arrives at `10:06:30` (1.5 min late, within 2 min lateness) → window is still alive → **Late firing** (updated result)
3. Watermark reaches `10:07:00` (window end + 2 min lateness) → window is destroyed

**Real-life use cases:**
- **Mobile apps:** Users in poor network areas send events that arrive late. Without allowed lateness, their data is lost. With 5-minute lateness, you capture most stragglers.
- **IoT sensors:** Field devices with intermittent connectivity.
- **Distributed systems:** Kafka consumer lag can cause events to arrive minutes late.

> **Important:** You'll get multiple results for the same window (one regular + possibly multiple late firings). Your downstream system must handle "update" semantics, not just "append."

---

## Step 7: Side Outputs for Late Data

Instead of just allowing or dropping late events, route them to a **separate stream** for special handling:

```java
// Tag for the late-data side stream
OutputTag<Transaction> lateTag = new OutputTag<Transaction>("late-transactions"){};

SingleOutputStreamOperator<DailySales> mainResult = input
    .keyBy(e -> e.store)
    .window(TumblingEventTimeWindows.of(Duration.ofDays(1)))
    .allowedLateness(Duration.ofHours(1))
    .sideOutputLateData(lateTag)      // route very late events here
    .reduce(new SumSales());

// Main stream: on-time results
DataStream<DailySales> sales = mainResult;

// Side stream: events that arrived after the 1-hour grace period
DataStream<Transaction> lateTransactions = mainResult.getSideOutput(lateTag);

// Handle late transactions separately — maybe write to a reconciliation table
lateTransactions.addSink(new ReconciliationDatabase());
```

**Real-life use cases:**
- **Financial reconciliation:** Late transactions that missed the daily window go to a "pending reconciliation" queue for manual review.
- **Analytics:** Late events are written to a separate "corrections" table in the data warehouse.

---

## Putting It All Together — A Complete Real-World Scenario

**Scenario:** Uber-like ride-sharing platform

You want to compute: *"Average ride fare per city, every 5 minutes, with handling for GPS event delays"*

```java
DataStream<Ride> rideStream = ...;  // Kafka source

SingleOutputStreamOperator<CityFareReport> result = rideStream
    .keyBy(ride -> ride.city)                           // group by city
    .window(TumblingEventTimeWindows.of(               // 5-minute tumbling windows
        Duration.ofMinutes(5)))
    .allowedLateness(Duration.ofMinutes(2))             // GPS events can be 2 min late
    .sideOutputLateData(lateRidesTag)                  // capture very late rides
    .aggregate(
        new AverageAggregator(),                        // memory-efficient avg computation
        new EnrichWithCityAndWindow()                   // add city name + window timestamp
    );

// Handle very late rides separately
result.getSideOutput(lateRidesTag)
      .addSink(new LateRideReconciliationSink());

// Main output: real-time dashboard feed
result.addSink(new DashboardSink());
```

---

## Summary Cheat Sheet

| Concept | What it is | Real-life analogy |
|---------|-----------|-------------------|
| Tumbling Window | Fixed, non-overlapping buckets | Hourly payroll runs |
| Sliding Window | Overlapping, rolling buckets | Moving average on stock chart |
| Session Window | Activity-gap-based buckets | Browser session timeout |
| Global Window | One window forever, custom trigger | Process every N orders |
| ReduceFunction | Incremental same-type aggregation | Running sum on a calculator |
| AggregateFunction | Incremental flexible-type aggregation | Computing an average (sum + count → divide) |
| ProcessWindowFunction | Full access to all elements + metadata | Batch job over collected data |
| Trigger | When to fire the window | A timer going off |
| Evictor | Remove elements before/after compute | Only keep last 100 readings |
| Allowed Lateness | Grace period for late events | "Late submissions accepted until Friday" |
| Side Output | Separate stream for late/special events | A reject bin for out-of-spec items |

---

## Key Rules to Remember

1. **Always use event-time windows** (not processing-time) when your data has meaningful timestamps — processing-time gives inconsistent results under backpressure or reprocessing.
2. **`ReduceFunction`/`AggregateFunction` = memory efficient** (only stores accumulator). **`ProcessWindowFunction` = memory heavy** (stores all events).
3. **Session windows require mergeable state** — Flink merges overlapping sessions automatically.
4. **Global windows do nothing without a custom trigger** — they have no natural end.
5. **Allowed lateness means you can get multiple results for the same window** — design your downstream to handle updates.
6. **Evictors disable pre-aggregation** — avoid unless truly necessary.
