# ETL: Extract, Transform, Load (Batch Processing)

## Overview

**ETL (Extract, Transform, Load)** is the foundational data integration pipeline pattern used to compile raw, heterogeneous data into clean, structured tables optimized for analytics and machine learning model training. In modern big data platforms, ETL pipelines run as batch processes utilizing distributed computing frameworks like **Apache Spark** to clean datasets, validate schemas, remove duplicates, and backfill historical records.

---

## Problem Statement

Designing batch-processing pipelines at petabyte scale introduces operational and computational challenges:
1. **Inefficient Disk I/O (MapReduce Bottleneck)**: Traditional batch frameworks (like Hadoop MapReduce) write intermediate states to disk after every step, causing severe latency overhead during multi-step transformations.
2. **Schema Ingestion Failures**: Upstream applications modify schemas (e.g., changing a column type or dropping a field) without warning, causing downstream ML models to fail during inference.
3. **Expensive Backfills**: When a transformation bug is fixed, re-running the pipeline over five years of historical data can consume massive compute budgets if the backfill strategy is not optimized.

---

## Batch Processing Architectures: MapReduce vs. Apache Spark

Distributed computing frameworks slice large datasets across clusters of worker nodes:

- **Hadoop MapReduce (Disk-Heavy)**:
  * **Map**: Reads input blocks from disk (HDFS), applies transformations, and writes intermediate key-value files to local disks.
  * **Shuffle & Sort**: Transports intermediate files across the network to group values by keys.
  * **Reduce**: Computes final aggregates and writes output back to HDFS disk.
  * *Bottleneck*: Continuous disk writes and network transfers limit execution speed.
- **Apache Spark (In-Memory)**:
  * Spark uses **Resilient Distributed Datasets (RDDs)** and DataFrames to execute transformations entirely in memory (RAM).
  * **DAG Optimization (Lazy Evaluation)**: Spark does not run transformation commands immediately. It builds an execution plan. It only runs the computation when an "action" (like `.save()` or `.count()`) is called, optimizing the query graph (e.g., fusing filter operations) to bypass unnecessary steps.

---

## Data Validation & Quality Pipelines
Before loading data into target tables, the pipeline must enforce strict quality guardrails:
- **Schema Enforcement**: Blocks writes if columns do not match defined parquet/table data types.
- **Great Expectations / Data Quality Frameworks**: Executes assertion tests on DataFrames:
  * `expect_column_values_to_not_be_null("user_id")`
  * `expect_column_values_to_be_between("age", 18, 120)`
- **Deduplication**: Drops duplicate records using unique primary keys (e.g., event ID + timestamp) using partition window functions.

---

## Backfilling Strategies

When model training code or feature definitions update, historical data must be reprocessed:
- **Partition Overwrites**: Instead of rewriting the entire table, structure the dataset by date partitions (e.g., `year=YYYY/month=MM/day=DD`). Reprocessing only overwrites the target partition directory, leaving other dates untouched.
- **Parallel Backfill Workers**: Spin up parallel Spark jobs, each assigned to process a distinct range of date partitions (e.g., Worker 1 processes 2025 data, Worker 2 processes 2026 data), leveraging cluster elasticity to speed up run time.

---

## Design Decisions & Trade-offs

### ETL vs. ELT (Extract, Load, Transform)

- **ETL (Extract, Transform, Load)**:
  * *Mechanism*: Data is transformed in-memory *before* loading into the target database.
  * *Pros*: High security (sensitive data like PII is redacted before writing to storage), clean target tables, optimized storage footprint.
  * *Cons*: Rigid pipelines. If business requirements change, raw data is lost unless saved separately.
- **ELT (Extract, Load, Transform)**:
  * *Mechanism*: Raw data is loaded directly into the target database/data lake (S3/Snowflake) and transformed on-demand using SQL/dbt.
  * *Pros*: High flexibility. Raw data is always preserved, allowing developers to write new transformations retrospectively.
  * *Cons*: High storage costs, potential security exposure of unredacted raw datasets.

---

## Interview Questions

### Q1: Compare Hadoop MapReduce and Apache Spark. Why is Spark significantly faster for multi-step ETL pipelines?
**Answer**:
1. **Memory vs. Disk**:
   - **Hadoop MapReduce** is designed with a state-persistent model. Every Map and Reduce step must read from and write to disk (HDFS) to ensure fault tolerance.
   - **Apache Spark** performs computations in-memory (RAM). It keeps intermediate data structures in memory across transformation steps, eliminating slow disk read/write cycles.
2. **Execution Graph Optimization**:
   - MapReduce runs tasks sequentially as a rigid Map $\rightarrow$ Shuffle $\rightarrow$ Reduce block.
   - Spark builds a **Directed Acyclic Graph (DAG)** of execution. Because of **Lazy Evaluation**, Spark's query optimizer (Catalyst) evaluates the entire chain of transformations before execution, combining filters, pruning unused columns, and optimizing partition joins in memory before performing physical operations.

### Q2: How do you design an ETL pipeline to be idempotent? Why is idempotency critical when backfilling data?
**Answer**:
1. **Design Patterns**:
   - **Write Overwrite**: When writing data for a specific partition (e.g., `day=2026-07-22`), configure the job to overwrite the partition folder rather than append. If the job runs twice, the second run simply overwrites the first, preventing duplicate records.
   - **Upsert Mechanics**: Use `UPSERT` or `MERGE INTO` SQL commands, matching records on unique business keys. Existing rows are updated with new values; missing rows are inserted.
2. **Critical for Backfilling**:
   - Backfills involve re-running pipelines over historical date ranges to correct bugs or update logic.
   - If the task is not idempotent, re-running a backfill job will write duplicate logs, corrupting historical aggregates and invalidating subsequent ML model training runs.

---

## References

1. **MapReduce Core Paper**: Dean, J., & Ghemawat, S. (2004). *MapReduce: Simplified Data Processing on Large Clusters*. ACM OSDI.
2. **Apache Spark Design**: Zaharia, M., et al. (2012). *Resilient Distributed Datasets: A Spack-Tolerant Abstraction for In-Memory Cluster Computing*. NSDI 2012.
3. **Data Quality Best Practices**: *Great Expectations: Declarative Data Quality Framework*. (Great Expectations project docs).
