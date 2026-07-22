# Data Pipelines & Orchestration (DAGs)

## Overview

**Data Pipelines** are structured networks of data processing tasks that move data from source systems to downstream analytics databases and feature stores. Managing large-scale data flows requires **Workflow Orchestrators** (such as Apache Airflow or Prefect) that model tasks as Directed Acyclic Graphs (DAGs), manage execution schedules, allocate compute pools, and handle task failures and backfills in an automated, reliable manner.

---

## Problem Statement

As data systems scale, managing processing scripts manually introduces architectural instability:
1. **Ad-hoc Cron Spaghettis**: Scheduling individual tasks via cron (e.g., Run Step A at 1:00 AM, Step B at 2:00 AM) fails when Step A runs slow. Step B will launch before its input is ready, polluting downstream databases.
2. **Brittle Error Recovery**: When a middle task in a pipeline fails (e.g., API rate-limiting), the entire process stops. Recovering manually requires logging into servers, checking raw logs, and restarting scripts from the point of failure.
3. **Lack of Dependency Transparency**: Understanding the execution lineage—which datasets depend on which transformation tasks—becomes impossible without a centralized orchestration engine.

---

## Directed Acyclic Graphs (DAGs) & Orchestration Engines

To guarantee execution order, workflow engines compile tasks into **Directed Acyclic Graphs (DAGs)**:

```
                              [Start Pipeline]
                                     │
                                     ▼
                            [Task A: Fetch API]
                                     │
                  ┌──────────────────┴──────────────────┐
                  ▼                                     ▼
      [Task B: Clean Tables]                 [Task C: Schema Parse]
                  │                                     │
                  └──────────────────┬──────────────────┘
                                     │
                                     ▼
                           [Task D: Join & Sync]
                                     │
                                     ▼
                              [End Pipeline]
```

- **Directed**: Edges between tasks have a defined direction, showing execution flow (Task A must complete before Task B begins).
- **Acyclic**: The graph has no loops (cycles). A task cannot depend on its own downstream outputs, preventing infinite loops.
- **State Machine Control**: The scheduler monitors the state of each task: `No Status` $\rightarrow$ `Queued` $\rightarrow$ `Running` $\rightarrow$ `Success` or `Failed`. Downstream tasks are only triggered when all of their parent tasks transition to `Success`.

---

## Components of an Orchestration System (Apache Airflow Architecture)

1. **Scheduler**: The central brain. It constantly monitors the DAG directory, evaluates task dependencies, checks cron/time schedules, and pushes runnable tasks to the queue.
2. **Metadata Database**: Typically PostgreSQL. Stores run histories, task state variables, and connection credentials.
3. **Queue / Executor**: Allocates work to machines (e.g., Celery Queue, or Kubernetes Executor which spawns a dedicated pod for each task).
4. **Workers**: The physical execution units that run the task code.
5. **Webserver**: The user interface to visualize DAG runs, inspect logs, and manually trigger backfills.

---

## Design Decisions & Trade-offs

### Apache Airflow vs. Prefect

- **Apache Airflow**:
  * *Pros*: Highly mature, industry standard, massive community, rich provider library. Strong static scheduling control.
  * *Cons*: Heavy infrastructure overhead. DAGs are statically compiled, making dynamic parameters or run-time conditional loops difficult to execute.
- **Prefect**:
  * *Pros*: Code-first approach (any Python code can be a task via decorators). Supports dynamic runtime DAGs. Native serverless/agent model.
  * *Cons*: Less mature community, changing APIs between versions.

---

## Failure Handling & Resiliency

Data pipelines must expect failures (API timeouts, query blocks). Michelangelo and similar platforms enforce:
- **Exponential Backoff Retries**: If a task fails, do not retry immediately (which can worsen downstream API rate-limiting). Delay retries exponentially:
  $$\text{Delay} = \text{base\_delay} \times 2^{\text{retry\_count}}$$
- **Idempotency**: Every task must be idempotent—running the same task multiple times with the same input parameters must yield the identical database state, preventing duplicate records during retries.
- **Alerting & Dead-Letter Queues**: Failed tasks trigger webhooks to Slack/PagerDuty, or route un-parseable data rows to dead-letter storage for manual inspection.

---

## Interview Questions

### Q1: What is a DAG? Why is the "Acyclic" property critical in data pipeline design?
**Answer**:
1. **DAG (Directed Acyclic Graph)**: A mathematical representation of a set of tasks (vertices) and their dependencies (directed edges).
2. **Acyclic Importance**:
   - The acyclic property guarantees that there are no circular dependencies (e.g., Task A depends on Task B, which depends on Task C, which depends on Task A).
   - If circular dependencies existed, the scheduling engine would enter a deadlock state, waiting infinitely for parent tasks to succeed, preventing the pipeline from ever starting or completing.
   - It ensures that the execution path has a clear beginning and end, enabling the scheduler to calculate static topological sort orders to schedule tasks deterministically.

### Q2: What is an Idempotent Task in a data pipeline? Why is it crucial for error recovery?
**Answer**:
1. **Idempotency**: A property where an operation can be applied multiple times without changing the result beyond the initial application. In data pipelines, running an idempotent pipeline run 10 times results in the same database state as running it once.
2. **Crucial for Error Recovery**:
   - If a pipeline fails at Step 3 (writing data to a SQL database) due to a network timeout, the task must be retried.
   - If the task is **not** idempotent, retrying it will append duplicate rows to the database, causing data pollution.
   - An **idempotent** task uses strategies like `UPSERT` (update on conflict), overwrite writes (`INSERT OVERWRITE`), or checks for existing unique UUID hashes to ensure that repeated executions simply heal the state rather than duplicate data.

---

## References

1. **Apache Airflow Architecture**: *Apache Airflow Documentation: Architectural Overview*. (Astronomer Guides).
2. **Workflow Orchestration Systems**: Casado, R., & Younas, M. (2015). *Emerging Trends and Technologies in Big Data Pipeline Orchestration*. Journal of Computer and System Sciences.
3. **Idempotent Data Design**: *Designing Data-Intensive Applications: The Principles of Idempotency*. (O'Reilly).
