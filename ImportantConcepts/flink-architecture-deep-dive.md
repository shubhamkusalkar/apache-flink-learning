# Apache Flink Architecture — Deep Dive with Real-Life Examples

---

## The Big Picture: What Is Flink Made Of?

Before going deep, let's understand the whole system at 30,000 feet.

When you run a Flink job, three types of processes are involved:

```
┌─────────────────────────────────────────────────────────────────┐
│                        FLINK CLUSTER                            │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    JOB MANAGER                           │   │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐ │   │
│  │  │ Dispatcher   │ │ResourceMgr   │ │   JobMaster(s)   │ │   │
│  │  │ (REST + UI)  │ │(slot alloc.) │ │  (one per job)   │ │   │
│  │  └──────────────┘ └──────────────┘ └──────────────────┘ │   │
│  └──────────────────────────────────────────────────────────┘   │
│                            │ coordinates                        │
│           ┌────────────────┼────────────────┐                   │
│           ▼                ▼                ▼                   │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐            │
│  │ TaskManager 1│ │ TaskManager 2│ │ TaskManager 3│            │
│  │ [slot][slot] │ │ [slot][slot] │ │ [slot][slot] │            │
│  └──────────────┘ └──────────────┘ └──────────────┘            │
└─────────────────────────────────────────────────────────────────┘
         ▲
         │ submits job
┌──────────────┐
│    CLIENT    │
│ (your code)  │
└──────────────┘
```

Think of it like a **restaurant**:
- **Client** = A customer who places an order
- **JobManager** = The restaurant manager who coordinates everything
- **TaskManagers** = The kitchen workers who actually cook the food
- **Task Slots** = Individual cooking stations in the kitchen

---

## Part 1: The Client — Where Your Job Begins

The client is NOT part of the running Flink cluster. It's your application or the CLI tool that:

1. Compiles your Flink program into a **JobGraph** (a logical execution plan)
2. Submits the JobGraph to the JobManager
3. Optionally stays connected to display progress/logs, or disconnects immediately

```
Your Code (Java/Python/SQL)
        │
        ▼
   ExecutionEnvironment.execute()
        │
        ▼
  Build JobGraph (what operations, how they connect)
        │
        ▼
  Submit to JobManager via REST API
        │
        ▼
  Optionally wait for results / disconnect
```

**Real-life analogy:** You (the client) call a restaurant and place an order. You describe exactly what you want (the JobGraph). The restaurant takes over from there. You can either stay on hold to hear progress updates, or hang up and trust they'll deliver.

### CLI Submission Example

```bash
# Submit and wait (client stays connected)
./bin/flink run myFlinkJob.jar

# Submit and detach (client disconnects immediately)
./bin/flink run --detached myFlinkJob.jar
```

---

## Part 2: The JobManager — The Brain of the Operation

The JobManager is the **master process** that controls the entire Flink job lifecycle. It has three sub-components, each with a specific responsibility.

---

### 2.1 The Dispatcher — The Front Door

The Dispatcher is the **entry point** for job submissions. It's responsible for:

1. **Exposing a REST API** — The client talks to the Dispatcher via HTTP
2. **Hosting the Web UI** — The Flink dashboard you see at `localhost:8081`
3. **Creating a new JobMaster** for each submitted job

```
Client ──(REST)──► Dispatcher ──► creates ──► JobMaster (for Job A)
                                  ──► creates ──► JobMaster (for Job B)
                                  ──► creates ──► JobMaster (for Job C)
```

**Real-life analogy:** The **restaurant receptionist**. When a customer calls, the receptionist picks up (REST API), notes the order (receives the JobGraph), and hands it off to the right kitchen manager (JobMaster). The receptionist also runs the display board showing current orders (Web UI).

---

### 2.2 The ResourceManager — The Resource Allocator

The ResourceManager is responsible for **allocating Task Slots** (the unit of compute resources in Flink).

When a JobMaster needs resources to run tasks, it asks the ResourceManager: "Give me 12 slots."

The ResourceManager then:
- Checks if TaskManagers have free slots
- If yes: assigns them to the JobMaster
- If no (in YARN/Kubernetes): requests the cluster platform to start new TaskManagers
- If no (in standalone mode): waits until slots become available (can't start new machines itself)

```
JobMaster: "I need 8 slots"
    │
    ▼
ResourceManager:
    "TaskManager-1 has 3 free slots → assign"
    "TaskManager-2 has 3 free slots → assign"
    "TaskManager-3 has 2 free slots → assign"
    Total: 8 slots assigned ✓
```

**Real-life analogy:** The **restaurant supply manager / staffing coordinator**. When the kitchen needs more hands, they call this person. They either assign existing kitchen workers (free slots) or call a staffing agency to send more workers (request new TaskManagers from YARN/Kubernetes).

**Environment-specific behavior:**

| Environment | ResourceManager behavior |
|-------------|-------------------------|
| YARN | Can request YARN to start new TaskManager containers |
| Kubernetes | Can request K8s to spawn new TaskManager pods |
| Standalone | Can ONLY use pre-started TaskManagers — cannot spin up new ones |

---

### 2.3 The JobMaster — The Job Foreman

One JobMaster is created **per submitted job**. This means if 3 jobs are running in the same cluster, there are 3 JobMasters.

The JobMaster is responsible for **everything related to one specific job**:

1. **Scheduling tasks** — decides which subtask runs on which TaskManager slot
2. **Monitoring task progress** — tracks which tasks finished, which failed
3. **Coordinating checkpoints** — sends barrier injection signals to sources, collects acknowledgments
4. **Recovering from failures** — when a TaskManager dies, the JobMaster restarts the affected tasks from the last checkpoint

```
JobMaster (for "Sales Aggregation Job"):
  ├── Schedules: Source subtask → TaskManager-1 Slot-1
  ├── Schedules: Map subtask → TaskManager-1 Slot-1 (chained!)
  ├── Schedules: Window subtask → TaskManager-2 Slot-2
  ├── Coordinates: Checkpoint every 30s
  └── On failure: Restart from checkpoint, reschedule to available slot
```

**Real-life analogy:** The **head chef / kitchen manager** for a specific order. They direct the cooking process, watch each station, and if one cook goes on break (task failure), they reassign the work to another cook and pick up from where they left off (checkpoint recovery).

---

## Part 3: TaskManagers — The Workers

TaskManagers are the **worker processes** that actually execute your Flink code. This is where your `map()`, `filter()`, `window()`, `process()` functions run.

Each TaskManager:
- Runs as a **separate JVM process** (usually on a separate machine in production)
- Has a fixed number of **Task Slots**
- Executes one or more **subtasks** per slot
- Manages its own **state storage** (in memory or RocksDB)
- Sends **heartbeats** to the JobMaster to prove it's alive

```
TaskManager Machine (16 GB RAM, 8 CPU cores)
├── JVM Process
│   ├── Task Slot 1 (reserved: 4 GB managed memory)
│   │   └── Running: [Source] → [Filter] → [Map]  (chained into one thread)
│   ├── Task Slot 2 (reserved: 4 GB managed memory)
│   │   └── Running: [Window] → [Aggregate]
│   └── Task Slot 3 (reserved: 4 GB managed memory)
│       └── Running: [Join] → [Sink]
└── Shared: network buffers, JVM overhead, off-heap memory
```

**Real-life analogy:** A **kitchen worker**. They have a specific workstation (task slot) with dedicated tools and counter space. They follow the recipe steps (execute operators) assigned to them. Multiple kitchen workers are in the same kitchen (TaskManager machine), each at their own station.

---

## Part 4: Task Slots — The Unit of Parallelism

### What Is a Task Slot?

A task slot is a **reserved, isolated unit of resources** within a TaskManager. It's the smallest unit that Flink uses to schedule work.

Key properties:
- Each slot gets an equal share of the TaskManager's **managed memory**
- Slots provide **memory isolation** (one slot can't consume another's memory budget)
- Slots do **NOT** provide CPU isolation (all slots in a TaskManager share the same CPU cores)
- Each slot runs in its own **thread**

```
TaskManager with 3 slots, 12 GB managed memory:

  ┌─────────────────────────────────────────────┐
  │              TaskManager JVM                │
  │                                             │
  │ ┌──────────┐  ┌──────────┐  ┌──────────┐   │
  │ │  Slot 1  │  │  Slot 2  │  │  Slot 3  │   │
  │ │  4 GB    │  │  4 GB    │  │  4 GB    │   │
  │ │  memory  │  │  memory  │  │  memory  │   │
  │ └──────────┘  └──────────┘  └──────────┘   │
  │                                             │
  │     Shared: JVM heap, network buffers       │
  └─────────────────────────────────────────────┘
```

**Real-life analogy:** Think of a TaskManager as an **apartment building** and slots as **individual apartments**.

- Each apartment has its own dedicated square footage (managed memory)
- Residents in Apartment 1 can't overflow into Apartment 2's space
- But all apartments share the same building infrastructure (CPU, network)
- You can have 1 apartment building with 4 apartments, or 4 apartment buildings with 1 apartment each — same total capacity

### How Many Slots Should a TaskManager Have?

This is a tuning question. Common choices:

| Slots per TaskManager | Effect |
|----------------------|--------|
| **1 slot** | Maximum CPU isolation. Each TaskManager runs one thread. Simple to reason about. |
| **Equal to CPU cores** | Good balance. Each core gets a slot. Common in production. |
| **More than CPU cores** | Increases parallelism but causes CPU contention. Risky. |

**Rule of thumb:** Set slots = number of CPU cores per TaskManager machine.

---

## Part 5: Slot Sharing — The Game Changer

This is one of Flink's most clever architectural decisions. Without understanding it, you'll be confused about how parallelism relates to slots.

### The Naive Approach (Without Slot Sharing)

Consider this pipeline with parallelism=4:

```
Source (p=4) → Map (p=4) → Window (p=4) → Sink (p=4)

Without slot sharing: Need 4 + 4 + 4 + 4 = 16 slots
```

Problem: Source and Map are typically lightweight. Window is memory-intensive. If Source uses 4 full slots and Window also needs 4 full slots, but all 8 are equal size — you're wasting memory on Source tasks that don't need it.

### Flink's Solution: Slot Sharing

By default, **subtasks from different operators within the same job can share a slot**.

```
Job with parallelism=4:

  Slot 1 (on TM-1):  Source-1 → Map-1 → Window-1 → Sink-1
  Slot 2 (on TM-2):  Source-2 → Map-2 → Window-2 → Sink-2
  Slot 3 (on TM-3):  Source-3 → Map-3 → Window-3 → Sink-3
  Slot 4 (on TM-4):  Source-4 → Map-4 → Window-4 → Sink-4

With slot sharing: Need only 4 slots (= max parallelism = 4)
```

Each slot holds **an entire pipeline slice** — one subtask from every operator. This is called a **pipeline within a slot**.

**Real-life analogy:** An assembly line.

Without sharing: You have 4 stations for car frames, 4 for engines, 4 for painting = 12 stations, but they can't all run at once because each stage depends on the previous.

With sharing: You have 4 parallel **complete** assembly lines. Each line does frames → engines → painting end-to-end. 4 cars flow through 4 independent pipelines simultaneously. Much more efficient.

### The Slot Sharing Rule

```
Required slots = max parallelism across all operators in the job
```

Example:
```java
DataStream<Event> source = env.addSource(...).setParallelism(2);  // p=2
DataStream<Event> mapped = source.map(...).setParallelism(4);     // p=4 ← max
DataStream<Result> windowed = mapped.window(...).setParallelism(4); // p=4 ← max

Required slots = 4 (the maximum)
```

Not 2+4+4=10. Just **4**.

### Why Slot Sharing Improves Resource Utilization

Without sharing:
```
Source (light) occupies a full slot  ← wastes most of the slot's memory
Window (heavy) occupies a full slot  ← might not be enough memory!
```

With sharing:
```
Source + Window share a slot
The light source uses little memory → more memory available for the Window in the same slot
Resources are naturally balanced across the pipeline
```

**The heavy operator defines how much memory the slot needs. The light operators "hitchhike" for free.**

---

## Part 6: Tasks and Operator Chaining

### What Is a Task?

Flink groups multiple operator subtasks into a single **Task**, executed by a single thread. This grouping is called **operator chaining**.

```
Without chaining (3 threads):
Thread 1: Source → [network buffer] → Thread 2: Map → [network buffer] → Thread 3: Sink

With chaining (1 thread):
Thread 1: Source → Map → Sink  (no buffers, no serialization between them)
```

Chaining drastically reduces overhead: no serialization, no network buffers, no thread switching between these operators.

### When Can Operators Be Chained?

Flink chains operators automatically when:
1. They have the same parallelism
2. They are connected by a local forward (not a shuffle/keyBy)
3. No explicit `disableChaining()` is set

```java
// These chain together: same parallelism, forward connection
source.map(x -> x * 2).filter(x -> x > 0)  // → one chained task

// These DON'T chain: keyBy introduces a shuffle
source.map(...).keyBy(k -> k.id).window(...)  // → two separate tasks
```

**Real-life analogy:** An assembly line worker who does multiple steps without putting the part down.

Without chaining: Worker 1 picks up a car door → puts it on a conveyor → Worker 2 picks it up → does the next step → conveyor again. Every handoff has delay and cost.

With chaining: Worker 1 picks up the door, does Steps 1, 2, and 3 themselves, then passes. Far fewer handoffs.

### Visual Example

```
Job: Source → Map → Filter → keyBy → Window → Sink

Chained tasks:
  Task A (one thread): [Source → Map → Filter]
       │ (network shuffle via keyBy)
  Task B (one thread): [Window → Sink]
```

Slot layout with parallelism=3:
```
Slot 1: [Source-1 → Map-1 → Filter-1] ─── network ──► [Window-1 → Sink-1]
Slot 2: [Source-2 → Map-2 → Filter-2] ─── network ──► [Window-2 → Sink-2]
Slot 3: [Source-3 → Map-3 → Filter-3] ─── network ──► [Window-3 → Sink-3]
```

---

## Part 7: Job Submission — The Complete End-to-End Flow

Let's trace exactly what happens when you run `./bin/flink run myJob.jar`:

```
Step 1: CLIENT
  └─ Your main() method runs locally
  └─ DataStream API calls build a StreamGraph (logical DAG)
  └─ Optimizer converts it to a JobGraph (physical DAG with parallelism set)
  └─ JAR + JobGraph sent to Dispatcher via REST POST

Step 2: DISPATCHER
  └─ Receives JobGraph
  └─ Persists job info (for HA recovery)
  └─ Creates a new JobMaster for this job
  └─ Hands JobGraph to JobMaster

Step 3: JOB MASTER
  └─ Converts JobGraph to ExecutionGraph (one node per parallel subtask)
  └─ Asks ResourceManager for slots

Step 4: RESOURCE MANAGER
  └─ Checks available TaskManager slots
  └─ If not enough: requests new TaskManagers (YARN/K8s)
  └─ Returns slot offers to JobMaster

Step 5: JOB MASTER (slot assignment)
  └─ Deploys task binaries to TaskManagers
  └─ Assigns specific subtasks to specific slots
  └─ Starts execution

Step 6: TASK MANAGERS
  └─ Receive task deployment
  └─ Initialize state backends
  └─ Start executing subtasks
  └─ Subtasks connect to each other for data exchange (network channels)

Step 7: RUNNING
  └─ Data flows through operators
  └─ JobMaster coordinates checkpoints
  └─ TaskManagers send heartbeats to JobMaster
  └─ Results flow to sink
```

**Real-life analogy: Building a Pop-Up Restaurant**

1. **Client:** A catering company (you) plans the entire menu and staffing requirements (JobGraph).
2. **Dispatcher:** The event venue's coordinator receives the catering plan and assigns a head chef (JobMaster).
3. **JobMaster (head chef):** Reviews the plan, figures out exactly how many kitchen workers are needed, requests them from the staffing agency.
4. **ResourceManager:** The staffing agency assigns available workers (slots) or calls in new ones (new TaskManagers).
5. **TaskManagers:** Workers arrive, set up their stations, and start cooking according to the plan.
6. **Running:** Food flows from prep → cooking → plating → service continuously.

---

## Part 8: Parallelism — Running in Parallel

Parallelism is the number of parallel instances of a single operator. It directly determines how many slots are consumed.

### Setting Parallelism

**Level 1: Operator level** (most specific)
```java
stream.map(x -> x * 2).setParallelism(4)   // this map runs on 4 threads
```

**Level 2: Execution environment level** (default for all operators)
```java
env.setParallelism(8)   // all operators use parallelism=8 unless overridden
```

**Level 3: Cluster/job level** (via config or CLI)
```bash
./bin/flink run -p 16 myJob.jar   // override parallelism to 16
```

**Level 4: Flink configuration default**
```yaml
# flink-conf.yaml
parallelism.default: 1
```

Priority: Operator > Environment > CLI/Job > Config

### Parallelism vs Slots

```
Cluster: 4 TaskManagers × 3 slots each = 12 slots total

Job: parallelism=4
  Source (p=4) → Window (p=4) → Sink (p=4)
  Required slots = 4 (slot sharing!)
  Uses 4 of the 12 available slots
  Remaining 8 slots available for other jobs
```

```
Job: parallelism=12 (max out the cluster)
  Required slots = 12
  All 12 slots used
  Cluster is fully utilized
```

**Real-life analogy:** A warehouse with 12 packing stations. A small order needs 4 workers. A large order needs all 12. The number of packing stations you need = the parallelism of your job.

---

## Part 9: Deployment Modes — Where Does Your Job Run?

### Mode 1: Application Cluster (Recommended for Production)

**"One application, one cluster."**

The cluster is **dedicated** to your single Flink application. The cluster lives and dies with your job.

```
Your JAR ──► Cluster starts ──► Job runs ──► Job finishes ──► Cluster shuts down
```

Key feature: Your **main() method runs ON the cluster**, not on your local machine. The JobGraph is built inside the cluster.

```bash
# Kubernetes Application Mode
./bin/flink run-application \
    --target kubernetes-application \
    myFlinkJob.jar
```

**Advantages:**
- Complete resource isolation (no other jobs compete)
- Main() runs on-cluster (no fat client needed locally)
- Job failure only affects this cluster
- Easy resource accounting ("this job uses exactly these resources")

**Disadvantages:**
- Cluster startup time for every job submission
- One cluster per job = more overhead at large scale

**Real-life use case:** A bank's end-of-day settlement job that runs nightly, needs guaranteed isolated resources, and must not interfere with daytime risk calculation jobs.

---

### Mode 2: Session Cluster

**"One long-running cluster, many jobs."**

A Flink cluster runs continuously. Multiple jobs are submitted and run concurrently in the same cluster.

```
Cluster starts (once)
  ├── Job A submitted → runs → finishes → slots released
  ├── Job B submitted → runs → finishes → slots released
  ├── Job C submitted → runs → finishes → slots released
  └── Cluster keeps running indefinitely
```

```bash
# Start a session cluster
./bin/start-cluster.sh

# Submit jobs to it
./bin/flink run jobA.jar
./bin/flink run jobB.jar
./bin/flink run jobC.jar
```

**Advantages:**
- Near-instant job startup (cluster already warm)
- Efficient for many small/short jobs
- One cluster to manage

**Disadvantages:**
- **Resource competition:** Job A's poorly written code can starve Job B
- **Blast radius:** A TaskManager crash can kill tasks from **multiple jobs**
- **No isolation:** A memory leak in one job affects the whole cluster

**Real-life use case:** A data science team's development cluster. Multiple developers submit experimental jobs, explore data, and iterate quickly. Isolation isn't critical; fast iteration is.

---

### Side-by-Side Comparison

```
Scenario: 3 jobs running simultaneously

APPLICATION CLUSTER:
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │  Cluster A  │  │  Cluster B  │  │  Cluster C  │
  │   Job A     │  │   Job B     │  │   Job C     │
  │  (isolated) │  │  (isolated) │  │  (isolated) │
  └─────────────┘  └─────────────┘  └─────────────┘
  Job A crash → only Cluster A affected

SESSION CLUSTER:
  ┌─────────────────────────────────────────────────┐
  │                  Cluster XYZ                    │
  │   Job A    │    Job B    │    Job C              │
  │ (sharing resources with B and C)                │
  └─────────────────────────────────────────────────┘
  TaskManager crash → Job A, B, AND C all lose tasks
```

---

## Part 10: High Availability — What If the JobManager Crashes?

The JobManager is the brain. If it crashes, the entire job stops. Flink solves this with **High Availability (HA)**.

### How HA Works

```
Active JobManager ──────────────────────────────────┐
Standby JobManager 1 (watching via ZooKeeper)        │
Standby JobManager 2 (watching via ZooKeeper)        │
                                                     │
Active JM crashes! ──────────────────────────────────┘

ZooKeeper elects a new leader from standby managers
New active JobManager:
  1. Reads the persisted JobGraph from distributed storage (HDFS)
  2. Reads the last completed checkpoint from distributed storage
  3. Restarts the job from the last checkpoint
  4. Re-schedules tasks to available TaskManagers
```

**Real-life analogy:** A **backup conductor** at an orchestra.

The head conductor collapses mid-concert. The backup conductor (who's been watching) immediately steps up, checks their score (persisted JobGraph), finds where the music was (last checkpoint), and continues the concert from that measure. The audience barely notices a hiccup.

---

## Part 11: Fault Tolerance in Practice — What Happens When Things Fail

### Scenario 1: A TaskManager Crashes

```
Before: 3 TaskManagers, all running
  TM-1: Slot1[Source-1, Window-1], Slot2[Source-2, Window-2]
  TM-2: Slot1[Source-3, Window-3], Slot2[Source-4, Window-4]
  TM-3: Slot1[Sink-1], Slot2[Sink-2]

TM-2 crashes!
  Affected tasks: Source-3, Window-3, Source-4, Window-4 → all LOST

JobMaster detects loss (via missing heartbeats)
JobMaster checks: Is there a recent checkpoint?
  → Yes, checkpoint at T-30s exists

JobMaster action:
  1. Restart failed tasks on available slots (TM-1 or TM-3 have free capacity)
  2. Restore state from T-30s checkpoint
  3. Reset Kafka offsets to T-30s position
  4. Resume processing

Data from T-30s to T=now is replayed from Kafka
Result: Exactly-once processing restored, ~30s of replayed data, job continues
```

### Scenario 2: A JobMaster Crashes (with HA enabled)

```
JobMaster for "Sales Job" crashes
  → Checkpoint coordination stops
  → Task scheduling stops
  → Tasks in TaskManagers continue for a bit (they buffer data)

HA failover:
  → ZooKeeper detects leader loss (heartbeat timeout)
  → Standby JobMaster promoted to active
  → New JobMaster reads persisted execution plan + last checkpoint
  → New JobMaster reconnects to existing TaskManagers
  → Tasks that died are restarted from checkpoint
  → Job resumes
```

### Scenario 3: Slow TaskManager (Not Crashed, Just Slow)

Flink does NOT automatically handle slow tasks (stragglers) the way Spark does with speculative execution. A slow TaskManager:
- Reduces throughput of the entire job (backpressure propagates upstream)
- Can delay watermark advancement (minimum watermark rule!)
- Won't be automatically replaced

You need to monitor and handle this operationally.

---

## Part 12: Complete Architecture in Action — End-to-End Story

**Scenario: Real-time order processing for an e-commerce platform**

Requirements:
- 50,000 orders/second at peak
- Compute sales totals per category every 1 minute
- Must survive TaskManager failures without losing data
- Runs 24/7

**Infrastructure:**
- 5 TaskManagers, each with 4 slots = 20 slots total
- JobManager with HA (3 replicas on separate machines)

**Job setup:**
```java
env.setParallelism(4);  // 4 parallel pipelines

DataStream<Order> orders = env
    .fromSource(kafkaSource,
        WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10))
            .withTimestampAssigner((o, ts) -> o.timestamp)
            .withIdleness(Duration.ofMinutes(2)),
        "Orders")

orders
    .keyBy(o -> o.category)
    .window(TumblingEventTimeWindows.of(Duration.ofMinutes(1)))
    .aggregate(new SalesSumAggregator(), new CategoryWindowResult())
    .addSink(new DashboardSink());

env.enableCheckpointing(30_000);  // checkpoint every 30s
```

**What the cluster looks like at runtime:**

```
JobManager:
  └── JobMaster: manages "Order Aggregation" job
      └── Coordinates checkpoints every 30s
      └── Monitors 4 parallel pipelines

TaskManager-1 (slots 1-4):
  Slot 1: [KafkaSource-1 → WatermarkAssign-1] → [Window-1 → Aggregate-1 → Sink-1]
  Slot 2: [KafkaSource-2 → WatermarkAssign-2] → [Window-2 → Aggregate-2 → Sink-2]
  Slot 3: [KafkaSource-3 → WatermarkAssign-3] → [Window-3 → Aggregate-3 → Sink-3]
  Slot 4: [KafkaSource-4 → WatermarkAssign-4] → [Window-4 → Aggregate-4 → Sink-4]

TaskManagers 2-5: IDLE (16 slots available for other jobs in session mode,
                         or unused in application mode)

Checkpoints:
  Every 30s: All 4 pipelines snapshot their window accumulators to HDFS
  On TM crash: Job restores from last checkpoint, replays 30s of Kafka data
```

**Failure scenario at 2:47 PM:**
```
2:47:00 - Checkpoint N completes (all window accumulators saved to HDFS)
2:47:23 - TaskManager-1 JVM crashes (hardware failure)
2:47:24 - JobMaster detects missing heartbeat
2:47:25 - JobMaster declares TM-1's tasks as FAILED
2:47:26 - JobMaster checks: last checkpoint = N (at 2:47:00) ✓
2:47:27 - JobMaster requests 4 slots from ResourceManager (for new task replicas)
2:47:28 - ResourceManager assigns slots from TM-2 through TM-5
2:47:29 - JobMaster restores window accumulators from checkpoint N
2:47:30 - JobMaster resets Kafka consumer offsets to checkpoint N positions
2:47:31 - Tasks start running again from 2:47:00 state
2:47:32 - 23 seconds of Kafka data replayed
2:47:45 - Job fully caught up, processing new data in real-time

Total downtime: ~20 seconds
Data loss: ZERO
Duplicate processing: ZERO (exactly-once via checkpoint alignment)
```

---

## Summary Cheat Sheet

### Components

| Component | Role | Analogy |
|-----------|------|---------|
| **Client** | Builds and submits the job | Customer placing an order |
| **Dispatcher** | REST entry point + WebUI + creates JobMasters | Restaurant receptionist |
| **ResourceManager** | Allocates slots to jobs | Staffing coordinator |
| **JobMaster** | Manages one job's execution + checkpoints | Head chef for one order |
| **TaskManager** | Worker process that runs operator code | Kitchen worker |
| **Task Slot** | Reserved memory unit within a TaskManager | Individual workstation in the kitchen |

### Key Formulas

```
Slots per TaskManager = equal share of managed memory per slot

Required slots for a job = max parallelism across all operators (with slot sharing)

Slots needed without sharing = sum of all operator parallelisms

Checkpoint recovery window = last checkpoint time to failure time (must be replayed)
```

### Key Rules to Remember

1. **One JobMaster per job** — multiple jobs in the same cluster = multiple JobMasters, one ResourceManager.
2. **Slots = memory isolation, not CPU isolation** — all slots in a TaskManager share CPU cores.
3. **Slot sharing = max_parallelism slots needed** — not the sum of all parallelisms.
4. **Operator chaining = fewer threads and zero serialization overhead** between chained operators.
5. **Application Cluster = isolation** (one job, one cluster). **Session Cluster = sharing** (many jobs, one cluster).
6. **TaskManager crash = some tasks die** (recoverable via checkpoint). **JobManager crash = entire job pauses** (recoverable via HA).
7. **Parallelism hierarchy:** Operator setting > Env setting > CLI flag > Config default.
8. **The number of task slots limits your maximum parallelism** — you cannot run more subtasks than you have slots.
9. **ResourceManager behavior depends on the platform** — Kubernetes/YARN can elastically add TaskManagers; standalone cannot.
10. **Chaining is automatic** — Flink chains operators that have the same parallelism and are connected by forward edges. Use `disableChaining()` only when debugging.
