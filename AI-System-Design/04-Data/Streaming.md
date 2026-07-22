# Streaming & Real-Time Analytics

## Overview

**Real-Time Streaming** is the continuous ingestion, processing, and analysis of data points as they are generated. In modern AI architectures (such as real-time fraud detection, dynamic pricing, and online personalization), streaming engines (like **Apache Flink**) process infinite event logs. They execute low-latency stateful computations, manage out-of-order event arrivals using watermarks, and guarantee transaction semantics (such as Exactly-Once) across distributed failures.

---

## Problem Statement

Processing infinite streams of data in real time introduces several architectural challenges:
1. **Handling Latency vs. Out-of-Order Data**: Network disruptions cause events to arrive out of order (e.g., an event occurring at 12:00 PM is delayed and arrives at 12:05 PM). Closing processing windows too early results in incorrect aggregates, while waiting too long increases system latency.
2. **Dynamic State Management**: Calculating running aggregates (e.g., "count the number of clicks by user $U$ in the last 2 hours") requires keeping state in memory across millions of users. If a worker node crashes, this state can be lost, causing incorrect output.
3. **Strict Delivery Guarantees**: Under failure scenarios (node restarts), network retries can duplicate events. If a financial transaction or click-rate metric is counted twice, it violates business logic rules.

---

## Stream Processing Semantics

When events are processed across distributed nodes, systems guarantee different delivery semantics:

| Semantic | Processing Guarantee | Overhead | Mechanism |
| :--- | :--- | :--- | :--- |
| **At-Most-Once** | Each event is processed once or zero times. Data can be lost but never duplicated. | Lowest | No retries. If a node crashes, the data currently in transit is discarded. |
| **At-Least-Once** | Each event is guaranteed to be processed. No data is lost, but duplicates can occur. | Medium | Retries are enabled. If an acknowledgment is lost, the sender resends the event. |
| **Exactly-Once** | Each event is processed exactly once. No data is lost, and state changes behave as if there were zero duplicates. | High | Distributed snapshots (Chandy-Lamport algorithm) combined with two-phase commits to downstream sinks. |

---

## Windowing Strategies & Watermarks

To calculate aggregates on infinite data streams, the stream must be sliced into logical, finite blocks called **Windows**:

```
[Event Stream] ──(12:01)──(12:02)──(12:03)──(12:04)──(12:05)──(12:06)──>
                 │                            │              │
                 └───────── Window 1 ─────────┘              │
                           (12:00 - 12:05)                   └─ Window 2 ──...
                                                               (12:05 - 12:10)
```

### 1. Windowing Types
- **Tumbling Windows**: Fixed-size, non-overlapping time blocks (e.g., a 5-minute window). An event belongs to exactly one window.
- **Sliding Windows**: Fixed-size, overlapping time blocks (e.g., a 5-minute window that slides every 1 minute). An event can belong to multiple windows.
- **Session Windows**: Gap-driven windows triggered by user inactivity. If a user stops clicking for 30 minutes, the session window closes.

---

### 2. Watermarks (Late Data Handling)
To balance latency and data completeness, streaming engines use **Watermarks**:
- **Definition**: A watermark is a control timestamp injected into the stream that tells the engine: "We assume no more events will arrive with event timestamps older than $t$."
- **Execution**: If the current watermark is 12:05 PM, the engine closes the 12:00 - 12:05 PM window and outputs the result. If a late event with a timestamp of 12:04 PM arrives *after* the 12:05 PM watermark has passed, the engine routes it to a designated **Side Output** or discards it based on configuration rules.

---

## State Management & Checkpointing (Apache Flink)

Stateful streaming engines (like Flink) store intermediate metrics (e.g., active user sessions) dynamically:
- **Local State Backends**: Flink workers store active state in local memory or high-speed embedded databases (**RocksDB**) located on the physical worker nodes.
- **Asynchronous Barrier Snapshotting (ABS)**: Flink injects "checkpoint barriers" into the input stream. When a barrier passes through a worker node, the worker takes an asynchronous snapshot of its local state and uploads it to persistent object storage (S3).
- **Failure Recovery**: If a worker node crashes, the scheduler spins up a replacement node, pulls the latest snapshot from S3, aligns the input stream to the checkpoint barrier, and resumes execution, guaranteeing Exactly-Once state recovery.

---

## Design Decisions & Trade-offs

### Apache Flink vs. Spark Structured Streaming

- **Apache Flink (Continuous Processing)**:
  * *Mechanism*: Processes events one-by-one as they arrive (native streaming).
  * *Pros*: Lowest possible latency (sub-millisecond), advanced state and watermark control, powerful session windowing.
  * *Cons*: High deployment complexity, requires managing dedicated state backends.
- **Spark Structured Streaming (Micro-Batch)**:
  * *Mechanism*: Groups streaming events into small micro-batches (e.g., every 100ms) and processes them using the Spark engine.
  * *Pros*: Shared APIs with Spark batch processing, simpler operations, high throughput.
  * *Cons*: Higher latency (typically $>10\text{ ms}$ due to micro-batch grouping), limited support for complex session windows.

---

## Interview Questions

### Q1: What is a Watermark in stream processing? How does it handle out-of-order data?
**Answer**:
1. **Concept**: A Watermark is a monotonically increasing timestamp injected into a stream that acts as a progress metric. It informs the processing engine that all events prior to the watermark timestamp have likely been processed.
2. **Handling Out-of-Order Data**:
   - Due to network lag, events can arrive out of order (e.g., Event $A$ with timestamp 12:01 arrives at 12:06, after Event $B$ with timestamp 12:05).
   - If we define a watermark strategy with a **bounded-out-of-orderness** of $3\text{ minutes}$, the watermark value will equal $\max(\text{event\_time}) - 3\text{ minutes}$.
   - If the maximum event timestamp seen is 12:08, the watermark is 12:05.
   - Any incoming event with a timestamp older than 12:05 is considered "late data". The 12:00-12:05 window remains open until the watermark crosses 12:05, allowing out-of-order events (like the 12:04 event) to be merged correctly.
   - Once the watermark passes 12:05, the window is finalized, and any subsequent matching late data is routed to side outputs.

### Q2: How does Apache Flink achieve Exactly-Once processing semantics under worker node failures?
**Answer**:
Flink achieves Exactly-Once semantics using **Asynchronous Barrier Snapshotting (ABS)** and **Two-Phase Commit (2PC) Sinks**:
1. **Checkpoint Barriers**: Flink injects barrier packets into the source stream. These barriers flow through the graph alongside standard events, dividing the stream into epochs.
2. **State Snapshotting**: When an operator receives a barrier, it pauses incoming processing, takes an asynchronous snapshot of its current state (e.g., running counters), writes the snapshot to S3, and propagates the barrier downstream.
3. **Sink Coordination (Two-Phase Commit)**: When the final output sink receives the barrier, it prepares a transaction block. Once the job manager confirms that all operators successfully saved their snapshots, the sink commits the transaction to the target database.
4. **Recovery**: If a node crashes, Flink rolls back the entire cluster state to the last successful checkpoint, resets the read pointer in the source message broker (e.g., Kafka offset) to the checkpoint location, and reprocesses the data, ensuring the final output matches exactly-once execution.

---

## References

1. **Flink Architecture**: Carbone, P., et al. (2015). *Apache Flink: Stateful Stream Processing on a Global Scale*. Bulletin of the IEEE Computer Society Technical Committee on Data Engineering.
2. **Exactly-Once Semantics**: Akidau, T., et al. (2015). *The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale, Unbounded, Out-of-Order Data Processing*. VLDB 2015.
3. **Chandy-Lamport Algorithm**: Chandy, K. M., & Lamport, L. (1985). *Distributed Snapshots: Determining Global States of Distributed Systems*. ACM Transactions on Computer Systems.
