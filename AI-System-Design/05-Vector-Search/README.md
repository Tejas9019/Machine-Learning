# Vector Search & High-Dimensional Databases

Welcome to the **Vector Search & High-Dimensional Databases** module. This section provides an in-depth, production-focused engineering guide for designing, indexing, and scaling similarity search systems, vector database architectures, and hybrid search pipelines.

These guides are written for senior engineers, detailing quantization math, index construction (HNSW, IVF), metadata filtering, hardware acceleration, and relevant system design interview questions.

---

## 🗺️ Module Learning Roadmap

Vector search systems transition from basic mathematical similarity metrics to complex distributed databases and hybrid search pipelines:

```
 [Vector Similarity Metrics] ──> [In-Memory Indexes (FAISS)] ──> [Metadata Filtering]
              │                                                         │
              ▼                                                         ▼
 [Embedded Stores (Chroma)] ──> [Distributed Databases (Milvus)] ──> [Hybrid Search (Weaviate)]
```

---

## 📂 Topic Breakdown

Click on any topic below to access the deep-dive architectural guide:

| Topic | Primary Focus | Core Engineering Challenges |
| :--- | :--- | :--- |
| ⚡ **[FAISS](FAISS.md)** | Core Indexing Library | IVF clustering, Product Quantization (PQ) math, HNSW construction, GPU acceleration. |
| 🌲 **[Pinecone](Pinecone.md)** | Managed Serverless Cloud | Decoupling read/write paths, metadata indexing, multi-tenant isolation, serverless scale. |
| 🦁 **[Milvus](Milvus.md)** | Distributed Enterprise DB | Log-broker architecture, QueryNode/IndexNode separation, segments, Kubernetes scaling. |
| 🪱 **[Weaviate](Weaviate.md)** | Hybrid & Graph Search | HNSW in Go, Sparse+Dense Hybrid search, Reciprocal Rank Fusion (RRF), custom integrations. |
| 🎨 **[Chroma](Chroma.md)** | Developer-Friendly Embedded DB | SQLite metadata storage, HNSWlib integration, local prototyping, scaling bottlenecks. |

---

## 📐 Core Vector Search Principles

Every vector database or similarity indexing pipeline must balance:

1. **Precision vs. Latency Recall Trade-off**: Approximate Nearest Neighbors (ANN) indexes sacrifice absolute accuracy (recall) to achieve sub-millisecond query latencies.
2. **Memory Footprint**: High-dimensional vectors (e.g., 1536 dimensions) consume massive RAM. Compression techniques like Product Quantization (PQ) are crucial to reduce cost.
3. **Filtering Strategies**: Combining vector similarity with metadata constraints (e.g., `user_id = 123`) requires selecting between *Pre-filtering* (high latency/low recall), *Post-filtering* (low recall), and *Single-stage Filtering* (hybrid indexing).
4. **Read vs. Write Path Separation**: High-performance indexes like HNSW are slow to construct. Systems must isolate write-heavy ingestion and compilation paths from read-heavy query endpoints.
