# Data Ingestion & Storage Architecture

Welcome to the **Data Ingestion & Storage Architecture** module. This section provides an in-depth, production-focused engineering guide for designing database architectures, real-time data pipelines, analytical warehouses, and unstructured data lakes.

These guides are written for senior software engineers, detailing indexing schemes, database layouts, transactional guarantees, pipeline flows, and relevant system design interview questions.

---

## 🗺️ Module Learning Roadmap

Data architectures start with transaction writes (OLTP) and progress into transformation pipelines (ETL) and analytical query layers (OLAP):

```
     [SQL / NoSQL (OLTP)] ──> [Data Pipelines & ETL] ──> [Streaming / Real-Time]
              │                                                     │
              ▼                                                     ▼
     [Data Lake (S3/Iceberg)] ─────────────────────────────> [Data Warehouse (OLAP)]
```

---

## 📂 Topic Breakdown

Click on any topic below to access the deep-dive architectural guide:

| Topic | Primary Focus | Core Engineering Challenges |
| :--- | :--- | :--- |
| 🗄️ **[SQL](SQL.md)** | Relational Database Systems | B+ Tree indexes, ACID guarantees, transaction isolation, sharding, replication. |
| 🗃️ **[NoSQL](NoSQL.md)** | Non-Relational Storage | Document stores, Key-Value stores, Wide-Column engines, consistent hashing. |
| 🧬 **[Data Pipelines](Data_Pipelines.md)** | Data flow orchestration | Batch architectures, orchestrators (Airflow), fault tolerance, logging. |
| 🔄 **[ETL](ETL.md)** | Extract, Transform, Load | Schema validation, deduplication, backfills, data validation tests. |
| 🌊 **[Streaming](Streaming.md)** | Real-time processing | Event processing (Flink/Spark), windowing (sliding/session), watermark rules. |
| ❄️ **[Data Warehouse](Data_Warehouse.md)** | Analytical Query (OLAP) | Columnar storage, compression, MPP engines, compute-storage decoupling. |
| 🪸 **[Data Lake](Data_Lake.md)** | Object Table Formats | Delta Lake vs. Apache Iceberg, ACID metadata logs on S3, parquet compaction. |

---

## 📐 Core Data Architecture Goals

Every database or pipeline design must evaluate:
1. **Consistency vs. Latency**: Choosing between strong transactional consistency (ACID) and scale-out eventual consistency (BASE / CAP Theorem).
2. **Read vs. Write Path Optimization**: Tuning indices (B-Trees vs. LSM-Trees) to match the storage write/read ratio.
3. **Compute and Storage Decoupling**: Offloading analytical scaling by separating query engine processing from underlying compressed object storage (e.g., S3).
4. **Data Durability**: Guaranteeing zero data loss via write-ahead logging (WAL), multi-region replications, and robust backup pipelines.
