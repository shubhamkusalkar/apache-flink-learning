# Apache Flink — Stateful Stream Processing: Deep Dive with Real-Life Examples

---

## The Foundation: Why Does "State" Even Matter?

Before diving in, let's understand the problem state solves.

Imagine you're a cashier at a supermarket. A customer puts 5 items on the belt one by one:

- Item 1: ₹50
- Item 2: ₹120
- Item 3: ₹30
- Item 4: ₹200
- Item 5: ₹80

To give the customer their total bill, you **cannot** just look at item 5 in isolation (₹80). You need to **remember** the running total across all previous items.

That memory — that "running total you keep in your head" — is **state**.

Now imagine a streaming system where millions of events (like items) flow in per second. The system needs to "remember things" between events to answer questions like:

- "How many times has this user logged in this week?"
- "Has this credit card made more than 5 transactions in the last 10 minutes?"
- "What is the average temperature of this sensor over the last hour?"

None of these can be answered by looking at just one event. You need **state**.

---

## Part 1: Stateless vs Stateful Operations

### Stateless Operations — Each event is processed in isolation

```
Event In → [Process] → Event Out
```

Examples:
- **Filter:** "Drop all events where amount < 0" — you just look at the current event.
- **Map/Transform:** "Convert temperature from Celsius to Fahrenheit" — no memory needed.
- **Parse:** "Extract the userId field from a JSON string" — purely per-event.

Real-life analogy: A **security scanner at an airport**. It scans each bag independently. It doesn't need to remember the previous bag to scan the current one.

---

### Stateful Operations — Events are processed in context of past events

```
Event In → [Process + State] → Event Out
               ↕
           State Store
        (remembers past)
```

Examples:
- **Aggregations:** Sum, average, count over a time window.
- **Pattern detection:** "Alert if login fails 3 times in a row."
- **Joins:** Join a stream of orders with a stream of payments.
- **Machine learning:** Update a model's weights as new training data arrives.

Real-life analogy: A **bank teller**. When you walk in, they pull up your account history. They can't just look at your current request — they need the context of all your past transactions.

---

## Part 2: Keyed State — The Most Important Concept

### The Core Idea

In Flink, most state is **Keyed State**. Think of it like this:

You have a giant hash map (key → value store) distributed across many machines. Each event carries a key. When the event arrives, it goes to the machine responsible for that key, and only that machine holds the state for that key.

```
Event: { userId: "alice", amount: 100 }
  → goes to the partition/machine that owns key "alice"
  → reads/writes state only for "alice"

Event: { userId: "bob", amount: 50 }
  → goes to the partition/machine that owns key "bob"
  → reads/writes state only for "bob"
```

### Real-Life Analogy: Post Office Sorting

Imagine a large post office with 10 counters. Letters (events) arrive and are sorted by the first letter of the recipient's name:

- Counter 1: A–C
- Counter 2: D–F
- ...and so on

Each counter only manages mail for its assigned range. Counter 1 has no idea what Counter 5 is doing — and that's fine. Each counter works independently, in parallel, with no shared coordination needed.

This is exactly how Flink's keyed state works. Events with key "alice" always go to the same partition. No locking, no cross-machine transactions — just local reads and writes.

### Why Local Operations Matter

Without keyed state:
> Every aggregation would require talking to all machines → massive network overhead → slow and complex.

With keyed state:
> The state for a key lives exactly where the event is processed → no network call → fast and simple.

### Code Example

```java
// Count how many times each user clicked a button
DataStream<ClickEvent> clicks = ...;

clicks
  .keyBy(click -> click.userId)   // partition by userId
  .process(new RichFlatMapFunction<ClickEvent, String>() {

      // This ValueState holds one integer per userId
      private ValueState<Integer> clickCount;

      @Override
      public void open(Configuration config) {
          // Each userId gets its own independent counter
          clickCount = getRuntimeContext()
              .getState(new ValueStateDescriptor<>("count", Integer.class));
      }

      @Override
      public void flatMap(ClickEvent event, Collector<String> out) throws Exception {
          Integer count = clickCount.value();
          if (count == null) count = 0;
          count++;
          clickCount.update(count);
  
          if (count == 10) {
              out.collect("User " + event.userId + " clicked 10 times!");
          }
      }
  });
```

Each `userId` has its own independent `clickCount`. "alice" reaching 10 clicks doesn't affect "bob"'s counter at all.

---

### Types of Keyed State

Flink gives you different "shapes" of state storage depending on what you need to remember:

| State Type | What it stores | Real-life analogy |
|------------|---------------|-------------------|
| `ValueState<T>` | One single value per key | A sticky note with one number |
| `ListState<T>` | A list of values per key | A notepad with multiple lines |
| `MapState<K, V>` | A map of sub-keys to values per key | A small spreadsheet per customer |
| `ReducingState<T>` | One value, auto-reduced on each update | A running total that updates itself |
| `AggregatingState<IN, OUT>` | Like ReducingState but with type flexibility | Average calculator (sum+count auto-maintained) |

#### Real-life use cases for each:

**ValueState** — Fraud detection: Remember the last transaction amount per card. If the new amount is 10x larger than the last, flag it.

```java
// Last transaction amount per credit card
ValueState<Double> lastAmount; // per card number
```

**ListState** — Session tracking: Keep a list of all pages a user visited in their current session.

```java
// Pages visited this session per user
ListState<String> visitedPages; // per userId
```

**MapState** — Inventory management: For each warehouse (key), track stock levels of each product (sub-key).

```java
// Stock count per product, per warehouse
MapState<String, Integer> stock; // per warehouseId, productId → quantity
```

**ReducingState** — Running totals: Automatically maintain the sum of all transaction amounts for a customer.

```java
// Running total spend per customer
ReducingState<Double> totalSpend; // per customerId, auto-sums
```

---

## Part 3: Key Groups — How Flink Scales State

### The Problem with Scaling

Suppose you start with 4 parallel workers, and each worker handles some keys. Now you want to scale up to 8 workers. How do you redistribute the existing state without losing data?

You can't just randomly reassign keys — worker 5 needs to pick up exactly the state that was in worker 2 for its share of keys.

### The Solution: Key Groups

Flink pre-divides all possible keys into a fixed number of **Key Groups** when you define `maxParallelism`. Think of these as "slots" or "buckets".

```
maxParallelism = 128  →  128 Key Groups

4 workers:  Each worker handles 32 Key Groups
8 workers:  Each worker handles 16 Key Groups
16 workers: Each worker handles 8 Key Groups
```

Real-life analogy: **A hotel with 128 rooms**. When you hire 4 housekeepers, each cleans 32 rooms. When you hire 8, each cleans 16. The rooms (Key Groups) themselves never change — only the assignment of which housekeeper covers which rooms changes.

This is why Flink requires `maxParallelism` to be set upfront — it defines how many "rooms" exist. You can never exceed that number of workers.

---

## Part 4: State Persistence — How Flink Never Loses Your State

This is the **most critical** part of stateful stream processing. What happens when a machine crashes?

In a bank, if the cashier's till breaks mid-transaction, you don't want to start counting from zero. You need to restore from the last known good state.

Flink solves this with **Checkpointing**.

---

### Checkpointing: The Core Idea

Periodically, Flink takes a consistent snapshot of all operator states across all machines and saves it to durable storage (like HDFS or S3).

If a failure happens, Flink:
1. Goes back to the last successful checkpoint
2. Restores all operator states to what they were at that point
3. Replays the input data from that point forward

Real-life analogy: **Saving a video game**.

You're playing a long RPG. Every 10 minutes, the game auto-saves. If your computer crashes, you don't start from the beginning — you start from the last save point. You might replay 5 minutes of game, but you don't lose hours of progress.

The "save file" = Flink checkpoint  
The "replay 5 minutes" = Flink replaying events from the last checkpoint position in Kafka  

---

### The Hard Problem: Consistent Snapshots in a Distributed System

Here's the challenge. Flink is running on, say, 10 machines, all processing data simultaneously. You can't just say "pause everything and take a snapshot" — that would stop processing.

You need a way to take a consistent snapshot **while the data keeps flowing**. This is where **Stream Barriers** come in.

---

### Stream Barriers: The Brilliant Solution

A **barrier** is a special marker injected into the data stream by the source. It flows along with the data, in order, never overtaking records.

```
Data stream from Kafka:

... [event] [event] [event] [BARRIER n] [event] [event] [event] [BARRIER n+1] ...
                             ↑ checkpoint n starts here
```

The barrier divides the stream into two parts:
- Everything **before** the barrier = belongs to checkpoint n
- Everything **after** the barrier = belongs to checkpoint n+1

**How it flows through the system:**

```
Source                Operator A              Operator B
  │                      │                       │
  ├── event 1 ──────────►│                       │
  ├── event 2 ──────────►│                       │
  ├── BARRIER ──────────►│                       │
  │              A takes snapshot                │
  │              A forwards BARRIER ────────────►│
  ├── event 3 ──────────►│               B takes snapshot
  ...                                    B forwards BARRIER ──► Sink
```

Each operator, upon receiving a barrier:
1. Takes a snapshot of its current state
2. Saves the snapshot to the state backend
3. Passes the barrier downstream

When the barrier reaches the sink (final operator), the checkpoint is complete!

Real-life analogy: **Taking a company-wide audit photo**.

The CEO sends a signal down the org chart: "Freeze your department's records right now and send me a summary, then pass the signal down."

- VP of Sales freezes their pipeline report and sends it up
- Then passes "freeze" signal to their managers
- Each manager freezes their team data and reports up
- Eventually all summaries arrive at the CEO = one consistent company snapshot

Nobody stopped working — they just took a quick snapshot and continued.

---

### Barrier Alignment: For Multiple Input Streams

What if an operator has two input streams (like a join)?

```
Stream A: ... event_a1 ... [BARRIER] ... event_a2 ...
Stream B: ... event_b1 ... event_b2 ... [BARRIER] ...

                     ↓
                 Join Operator
```

The join operator must wait until it receives the barrier from **both** streams before taking its snapshot. This is called **barrier alignment**.

```
Step 1: Barrier arrives from Stream A
        → Operator BUFFERS further events from A (don't process yet)
        → Continues consuming from Stream B

Step 2: Barrier arrives from Stream B
        → Operator takes snapshot
        → Processes buffered events from A
        → Continues normally
```

Real-life analogy: **Synchronizing two assembly lines**.

Line A makes car frames. Line B makes engines. A "snapshot" audit requires counting both at the same point. When Line A reaches the audit mark, it pauses. Line B catches up to the audit mark. Now both are at the same point → take the audit → both resume.

The buffering adds a tiny bit of latency — but ensures the snapshot is **consistent** (you won't accidentally capture state from events of different "generations").

---

### Unaligned Checkpointing: Speed Over Strictness

For applications that need very low latency checkpoints (e.g., high-backpressure scenarios where alignment takes too long), Flink offers **unaligned checkpointing**.

Instead of waiting for barriers to align:
- The operator takes a snapshot immediately when the first barrier arrives
- All in-flight events (those already in the network buffer) are included in the snapshot as part of the state

```
Aligned:    Wait for all barriers → snapshot state only
Unaligned:  Snapshot state + all in-flight data immediately
```

**Trade-off table:**

| | Aligned Checkpointing | Unaligned Checkpointing |
|--|--|--|
| **Latency** | Higher (must wait for alignment) | Lower (snapshot immediately) |
| **State size** | Smaller (only operator state) | Larger (includes in-flight data) |
| **Best for** | Normal throughput | High backpressure scenarios |
| **I/O pressure** | Lower | Higher |

Real-life analogy (unaligned): **Taking a photo of a busy kitchen mid-service**.

Aligned = Wait for all chefs to finish their current dish, then photograph.  
Unaligned = Photograph immediately, but include notes saying "Chef A had half-finished pasta in transit."

The unaligned photo is faster but requires more information to capture the full state.

---

## Part 5: State Backends — Where Does State Actually Live?

The **state backend** decides two things:
1. Where state is stored **during computation** (in-memory vs disk)
2. How snapshots (checkpoints) are taken

Flink offers three backends:

---

### 1. HashMapStateBackend (Default)

State lives in the **Java heap** (in memory) of the TaskManager.

```
TaskManager JVM Heap
├── Key "alice"  → { clickCount: 42, lastLogin: 1710000000 }
├── Key "bob"    → { clickCount: 7,  lastLogin: 1710000100 }
└── Key "carol"  → { clickCount: 123, lastLogin: 1710000200 }
```

**Checkpoints** are written to a distributed file system (HDFS, S3, etc.).

**Best for:**
- Small to medium state sizes
- Low-latency access requirements
- Development and testing

**Real-life analogy:** Your work desk. Everything is within arm's reach (fast), but your desk has limited space. The filing cabinet (HDFS) stores the archived copy.

**Risk:** If state grows too large, the JVM runs out of heap → OutOfMemoryError.

---

### 2. EmbeddedRocksDBStateBackend

State lives in **RocksDB** (an embedded key-value store on disk) on each TaskManager machine.

```
TaskManager Disk (RocksDB)
├── /state/alice  →  serialized bytes
├── /state/bob    →  serialized bytes
└── /state/carol  →  serialized bytes
```

**Checkpoints** are incrementally uploaded to HDFS/S3 — only changed state is uploaded, not everything.

**Best for:**
- Very large state (TBs of state per job)
- State that doesn't fit in memory
- Production workloads with long-running jobs

**Real-life analogy:** A library. You can't keep all books on your desk (memory), so they live on shelves (disk). Accessing a book takes a bit longer than grabbing something off your desk, but the library can hold millions of books.

**Trade-off:** ~10x slower reads/writes vs in-memory, but can handle virtually unlimited state size.

---

### Choosing a State Backend

| Scenario | Recommended Backend |
|----------|---------------------|
| State < few GB, latency critical | HashMapStateBackend |
| State > tens of GB, large production job | EmbeddedRocksDBStateBackend |
| Rapid development / testing | HashMapStateBackend |
| Long-running jobs (days/weeks) | EmbeddedRocksDBStateBackend |

---

## Part 6: Savepoints — The "Manual Save" Feature

### What is a Savepoint?

A **savepoint** is a manually-triggered checkpoint that:
- **Never expires** automatically (unlike regular checkpoints which can be auto-deleted)
- Is **intentionally created** by an operator (not automatically by Flink)
- Can be used to **restart, upgrade, or migrate** a Flink job

```
Regular Checkpoint:  Auto-triggered by Flink → kept for N checkpoints → auto-deleted
Savepoint:           Manually triggered by you → kept forever → deleted only when you say so
```

### Real-Life Use Cases for Savepoints

**1. Zero-downtime code upgrades**

You have a Flink job running in production for 6 months with terabytes of state. You want to fix a bug in your `ProcessFunction`.

Without savepoints: Stop the job → deploy new code → start fresh → lose all state → wait months to rebuild.

With savepoints:
```bash
# Step 1: Take a savepoint before stopping
flink savepoint <jobId> hdfs:///flink/savepoints/

# Step 2: Stop the old job
flink cancel <jobId>

# Step 3: Deploy the new code with the saved state
flink run -s hdfs:///flink/savepoints/savepoint-xxx newJob.jar
```
The new version of the job picks up exactly where the old one left off.

**2. A/B testing new logic**

Take a savepoint of the production job. Start two jobs from the same savepoint:
- Job A: Old logic (production)
- Job B: New logic (testing)

Compare outputs without affecting production.

**3. Scaling up or down**

Job is running with parallelism=4 but traffic increased. Take a savepoint, stop the job, restart with parallelism=8. Flink redistributes the Key Groups across the new workers.

**4. Migrating to a new Flink version**

Old cluster: Flink 1.18. New cluster: Flink 2.0. Take a savepoint on the old cluster. Start the job on the new cluster from that savepoint.

---

### Savepoint vs Checkpoint — Quick Comparison

| | Checkpoint | Savepoint |
|--|--|--|
| **Triggered by** | Flink automatically | You manually |
| **Purpose** | Fault tolerance (auto-recovery) | Upgrades, migrations, testing |
| **Lifespan** | Auto-deleted after N checkpoints | Lives forever until you delete it |
| **Size** | Optimized for speed | Slightly larger (more metadata) |
| **Restore** | Automatic on failure | Manual |

---

## Part 7: Processing Guarantees — Exactly-Once vs At-Least-Once

This is about what happens to events when a failure occurs and Flink replays from a checkpoint.

### The Three Guarantees

**At-Most-Once:** Events might be lost. No retry. Fastest, but unreliable.

> Real-life: A sports commentator who misses a goal and never mentions it. Fast, but inaccurate.

**At-Least-Once:** Events are never lost, but might be processed more than once after a failure.

> Real-life: A journalist who sometimes reports the same news story twice to make sure nobody misses it. Safe, but redundant.

**Exactly-Once:** Every event is processed exactly once — no loss, no duplicates.

> Real-life: A bank ledger. Every transaction is recorded once, precisely. If a system crashes mid-entry, it rolls back and retries — but the transaction appears exactly once in the final record.

---

### How Flink Achieves Exactly-Once

Flink's barrier alignment (Step 4 above) is the key. By waiting for barriers from all inputs before snapshotting, Flink ensures that:

- The snapshot captures a **consistent** point across all operators
- On recovery, events are replayed from that consistent point
- No event from before the checkpoint is processed twice (because state already includes it)
- No event after the checkpoint is missed (because replay starts from there)

```
Timeline:

Checkpoint at T=100 captures state S100

Failure at T=150

Recovery:
  → Restore state to S100
  → Replay all events from T=100 onwards
  → Events from T=100 to T=150 are replayed
  → But state was already at S100 (not S150), so the replay produces correct results
  → No duplicates, no gaps
```

---

### When Exactly-Once Breaks Down

Exactly-once is a **processing guarantee**, not an output guarantee. If your sink (e.g., writing to a database) doesn't support transactions or idempotent writes, you might still get duplicates in the output even if Flink processed each event exactly once.

For end-to-end exactly-once, your sink must support:
- **Transactional sinks:** Kafka (with transactions), JDBC databases
- **Idempotent sinks:** Writes with the same key overwrite each other (like updating a DB row by primary key)

---

### At-Least-Once Mode (Skipping Alignment)

If you don't need exactly-once and want lower checkpoint latency, you can disable barrier alignment:

```java
env.getCheckpointConfig()
   .setCheckpointingConsistencyMode(CheckpointingMode.AT_LEAST_ONCE);
```

What happens on recovery: Some events that were processed before the checkpoint might be processed again (because they were in-flight during alignment and are replayed).

When is this OK? If your operation is idempotent (e.g., SET a value rather than INCREMENT it), duplicates don't matter.

---

## Part 8: How State Connects to Windows

Now that you understand state, windows become much clearer. **Every window is built on top of state.**

When you define a window:

```java
stream
  .keyBy(e -> e.userId)
  .window(TumblingEventTimeWindows.of(Duration.ofMinutes(5)))
  .sum("amount")
```

What Flink actually does internally:

1. **For each key (userId)**, Flink maintains state
2. The state holds the window's accumulated data (or accumulator for incremental functions)
3. When the watermark passes the window end time, Flink reads the state, computes the result, emits it, and **clears the state**
4. This entire state is included in checkpoints → if a failure happens mid-window, the partial accumulation is restored

```
State for key "alice":
  Window [10:00 - 10:05]:  accumulator = { sum: 340, count: 7 }

Checkpoint taken at 10:03 →
  saves: { alice → Window[10:00-10:05] → {sum: 340, count: 7} }

Failure at 10:04 →
  Restore state → alice's window still has {sum: 340, count: 7}
  Replay events from 10:03 → window gets updated correctly
  At 10:05 → window fires → correct result
```

Without stateful processing, you could never build reliable windows on streams.

---

## Part 9: Batch Processing as a Special Case

Flink can also process bounded (finite) data sets — like reading a CSV file or a bounded Kafka topic. In this case, the entire input is finite and known upfront.

**Key difference in recovery:**
- Streaming: Uses checkpoints → on failure, restore from last checkpoint and replay a portion
- Batch: No checkpoints during processing → on failure, replay the **entire** input from the beginning

Why? Batch jobs are typically faster and checkpointing overhead is not worth it. If a batch job reading 10 million records fails at record 9.5 million, it's cheaper to just restart from record 1 than to take 9.5 million checkpoints.

**State backend difference:**
- Batch jobs use simpler, optimized in-memory structures (no need for distributed key-value index since data is bounded)
- This gives better performance for batch workloads

---

## Part 10: Putting It All Together — A Real-World End-to-End Example

**Scenario: Real-time fraud detection for a payment network**

Requirements:
- Flag any card that makes more than 5 transactions in any 10-minute rolling window
- Never lose transaction data if a server crashes
- The system processes 100,000 transactions/second

```java
DataStream<Transaction> transactions = env
    .fromSource(kafkaSource, WatermarkStrategy.forBoundedOutOfOrderness(
        Duration.ofSeconds(5)), "Kafka Transactions");

transactions
    .keyBy(txn -> txn.cardNumber)       // ← Keyed State: each card has its own counter
    .window(SlidingEventTimeWindows.of(  // ← Window built on state
        Duration.ofMinutes(10),
        Duration.ofMinutes(1)))
    .aggregate(new TransactionCounter())  // ← State backend holds accumulator
    .filter(result -> result.count > 5)
    .addSink(new FraudAlertSink());

// Checkpointing config
env.enableCheckpointing(30_000);  // checkpoint every 30 seconds
env.getCheckpointConfig()
   .setCheckpointStorage("hdfs:///flink/checkpoints");
```

**What's happening under the hood:**

1. **Keyed State:** Each `cardNumber` gets its own isolated state. Card "4111..." and card "5500..." never interfere.

2. **Window State:** For each card, Flink maintains a sliding window accumulator. Every minute, a new window opens. Every minute, the oldest window closes.

3. **Checkpointing:** Every 30 seconds, Flink injects barriers into the Kafka stream. All operators snapshot their current window accumulators to HDFS.

4. **Failure scenario:** A TaskManager crashes at second 45. Flink detects the failure, restores state from the 30-second checkpoint, resets Kafka offsets to the checkpoint position, and replays 15 seconds of transactions. The window accumulators are correct. No transaction is double-counted, none is missed.

5. **Scaling:** Traffic doubles. You take a savepoint, stop the job, and restart with twice the parallelism. Key Groups are redistributed. Processing continues from exactly where it left off.

---

## Summary Cheat Sheet

| Concept | What it is | Key insight |
|---------|-----------|-------------|
| **State** | Memory across events | Without state, every event is independent — no aggregations possible |
| **Keyed State** | Per-key partitioned state | Each key's state lives on one machine — no coordination needed |
| **Key Groups** | Atomic unit for redistributing state | Enables scaling without losing state |
| **ValueState** | One value per key | Single counter, flag, or timestamp per entity |
| **ListState** | List of values per key | Sequence of events per entity |
| **MapState** | Map per key | Sub-key indexing within an entity |
| **Checkpointing** | Periodic consistent snapshots | Like auto-save in a game — recover from here on failure |
| **Stream Barriers** | Markers that divide streams into snapshot epochs | The mechanism that makes consistent distributed snapshots possible |
| **Barrier Alignment** | Waiting for barriers from all inputs | Ensures exactly-once by preventing split-brain snapshots |
| **Unaligned Checkpointing** | Include in-flight data in snapshot | Faster checkpoints under backpressure, but larger state |
| **HashMapStateBackend** | State in JVM heap | Fast, limited by memory |
| **RocksDBStateBackend** | State on local disk | Slower but handles TB-scale state |
| **Savepoint** | Manual, persistent checkpoint | For upgrades, migrations, A/B tests |
| **Exactly-Once** | Each event processed precisely once | Requires barrier alignment + transactional/idempotent sink |
| **At-Least-Once** | No event lost, but duplicates possible | Faster, OK for idempotent operations |

---

## Key Rules to Remember

1. **State is the foundation** — everything in Flink (windows, joins, pattern detection) is built on state.
2. **Always keyBy before accessing keyed state** — non-keyed state is global and can't scale.
3. **RocksDB for production, HashMap for development** — unless you're confident your state fits in heap.
4. **Checkpoints save you from failures; savepoints save you from yourself** (upgrades, mistakes).
5. **Exactly-once processing ≠ exactly-once output** — your sink also needs to be transactional or idempotent.
6. **Barrier alignment adds latency** — if your job has high backpressure, consider unaligned checkpointing.
7. **maxParallelism sets your Key Group ceiling** — you can never scale beyond this value, so set it generously upfront (e.g., 128 or 256).
8. **Windows are just state with a trigger** — understanding state makes windows completely transparent.
