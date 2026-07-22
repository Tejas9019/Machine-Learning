# Milvus (Distributed Enterprise-Grade DB)

## Overview

**Milvus** is an open-source, highly distributed, enterprise-grade vector database built from the ground up to handle billions of vector embeddings. Unlike single-node libraries or simple wrappers, Milvus separates storage, compute, and transaction coordination into independent microservices, making it a highly scalable solution for large-scale production workloads running on Kubernetes.

---

## Problem Statement

When operating a vector search engine at extreme scales (e.g., billions of vectors with high ingestion rates):
1. **Monolithic Scaling Bottlenecks**: Tightly coupled index building and query search resources lead to CPU starvation. Building an HNSW graph is extremely CPU/RAM intensive; doing so on search nodes degrades search throughput.
2. **Data Durability**: Storing massive graph indexes entirely in-memory risks severe data loss during node crashes, requiring slow index rebuilds from scratch.
3. **High Write Throughput**: Real-time applications require streaming ingestion, whereas traditional vector indices are designed for batch builds.

---

## Decoupled Architecture

Milvus solves these challenges by splitting the system into four distinct layers: coordination, worker nodes, log broker, and persistent object storage.

```
                  [Client Request]
                         │
                         ▼
                     [Proxy]
                         │
       ┌─────────────────┴─────────────────┐
       ▼                                   ▼
[Log Broker (Pulsar/Kafka)]        [Coordinators] (Root/Query/Index/Data)
       │
       ├─────────────────┬─────────────────┐
       ▼                 ▼                 ▼
  [QueryNode]       [DataNode]       [IndexNode] (Ephemerally builds indices)
  (Holds RAM        (Flushes to S3)
   segments)
       │                 │                 │
       ▼                 ▼                 ▼
 ─────────────────────────────────────────────
              [Object Storage (MinIO/S3)]
 ─────────────────────────────────────────────
```

### 1. Ingress & Coordination Layer
- **Proxy**: The gateway that authenticates clients, hashes partition keys, routes queries to the correct QueryNodes, and merges final search results.
- **Coordinators (Root, Query, Index, Data)**: Manage cluster metadata, assign segment workloads to worker nodes, coordinate index builds, and track background health.

### 2. Log Broker (WAL)
- Milvus uses **Apache Pulsar** or **Apache Kafka** as its Write-Ahead Log (WAL).
- All inserts, deletes, and updates are appended to the Log Broker. This ensures data persistence before any worker node parses the data.

### 3. Worker Node Layer (Compute)
- **QueryNode**: Subscribes to the Log Broker to consume real-time inserts, maintains in-memory vector segments, and executes vector similarity queries.
- **DataNode**: Subscribes to the Log Broker, groups streaming vectors into chunks (segments), and flushes them as raw files to Object Storage.
- **IndexNode**: Pulls unindexed segments from Object Storage, builds vector indexes (e.g., HNSW or IVF-PQ) in the background, and writes the completed index files back to Object Storage.

### 4. Storage Layer
- **Object Storage**: Consists of MinIO, AWS S3, or Google Cloud Storage. It stores the source vectors (as binlog files) and completed index files.

---

## Segment Management: Growing vs. Historical

To support real-time querying alongside heavy batch indexing, Milvus divides data into two types of segments:
- **Growing Segments**: Newly inserted vectors are appended to in-memory growing segments. Because these segments are too small or updated too frequently to build a graph, QueryNodes perform a brute-force Flat search over them to ensure real-time data visibility.
- **Historical Segments**: Once a growing segment reaches its limit (typically 512 MB or 1 million vectors), it is sealed, written to Object Storage, and picked up by an IndexNode. The IndexNode compiles it into a static HNSW/IVF index. QueryNodes then load this completed index into memory, executing high-speed graph traversals instead of brute-force scans.

---

## Design Decisions & Trade-offs

### Decoupled Microservices vs. Single-Node Databases

- **Decoupled Architecture (Milvus)**:
  * *Pros*: Infinite horizontal scaling. You can scale QueryNodes independently to handle search traffic spikes, scale IndexNodes to handle heavy index builds, and scale DataNodes for ingestion peaks. Highly resilient.
  * *Cons*: Extreme operational complexity. Requires maintaining Kubernetes, Zookeeper (metadata storage), a Log Broker (Kafka/Pulsar), and Object Storage (MinIO/S3).
- **Single-Node DB (e.g., Chroma, FAISS)**:
  * *Pros*: Simple, lightweight, zero-configuration setup, ideal for local development.
  * *Cons*: Poor horizontal scalability, single point of failure (SPOF), resource contention during index building.

---

## Scaling & High Availability

- **Kubernetes Operator**: Milvus is deployed via the `milvus-operator`, which automatically scales QueryNodes based on CPU/Memory usage or search queue latencies.
- **Sharding**: Collections are split into multiple physical shards (vChannels), allowing QueryNodes to process subsets of data in parallel.
- **Query Replicas**: To increase search throughput (QPS), Milvus loads the same historical segment index into multiple independent QueryNodes, distributing read queries across replicas.

---

## Interview Questions

### Q1: Explain the role of the Log Broker in Milvus. Why doesn't Milvus write directly to memory segments?
**Answer**:
1. **Durability & Crash Recovery**: Writing directly to memory segments is volatile. If a QueryNode crashes, data in memory is lost. The Log Broker (Kafka/Pulsar) acts as a persistent Write-Ahead Log (WAL).
2. **Decoupled Processing (Pub-Sub)**: The Log Broker allows Milvus to decouple write ingestion from write processing. An ingestion API request writes immediately to Kafka and returns success. DataNodes and QueryNodes then consume from Kafka asynchronously, preventing heavy index building or query loads from blocking the ingestion pipeline.
3. **Eventual Consistency Coordination**: By using the Log Broker as the single source of truth, different components can consume data at their own pace without needing complex peer-to-peer sync protocols.

### Q2: How does Milvus handle real-time search queries on data that has just been inserted but not yet indexed?
**Answer**:
1. Milvus partitions data into **Growing Segments** and **Historical Segments**.
2. **Growing Segments** hold newly inserted records. Because they don't have a compiled index, QueryNodes execute a brute-force Flat L2/IP search over them.
3. **Historical Segments** contain older data that has been sealed and compiled into a graph/vector index (e.g., HNSW). QueryNodes run high-speed index queries over these segments.
4. When a user submits a search query, the Proxy broadcasts the request to QueryNodes holding both the historical indexes and the growing segments. The QueryNode merges the results from both segments before returning the final top-$K$ list.

---

## References

1. **Milvus Architecture Design**: Xie, F., et al. (2021). *Milvus: A Purpose-Built Vector Data Management System*. ACM SIGMOD.
2. **Distributed Log-Broker Patterns**: Kreps, J., Narkhede, N., & Rao, J. (2011). *Kafka: a Distributed Messaging System for Log Processing*. NetDB.
