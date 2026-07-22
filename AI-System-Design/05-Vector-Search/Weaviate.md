# Weaviate (Hybrid & Graph Search)

## Overview

**Weaviate** is an open-source, Go-based, GraphQL-first vector database. It is designed to store both data objects and vector embeddings, allowing developers to execute hybrid searches combining keyword-based search (BM25) and semantic vector search in a single query. Weaviate provides a modular design that integrates with external machine learning inference providers (like OpenAI, Hugging Face, Cohere) directly inside the database pipeline.

---

## Problem Statement

Modern search applications face challenges when using vector-only retrieval:
1. **Vocabulary Mismatch**: Vector search is excellent for conceptual or semantic queries, but performs poorly on specific search targets like product serial numbers, exact names, or phone numbers.
2. **Merging Disjoint Search Results**: Separately querying a BM25 engine (like Elasticsearch) and a vector search engine (like FAISS) and then manual blending is complex and mathematically inconsistent.
3. **Inference Pipeline Friction**: Generating embeddings and running models in the application layer adds network hops, slow serialization, and coding complexity.

---

## Hybrid Search & Reciprocal Rank Fusion (RRF)

Weaviate addresses these issues by offering **Hybrid Search** out-of-the-box. It performs dense vector search and sparse keyword search (BM25) in parallel, then blends the results using **Reciprocal Rank Fusion (RRF)**:

```
                  [Client Search Query]
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
      [Dense Search]               [Sparse Search]
      (HNSW Vector Index)          (BM25 Inverted Index)
              │                           │
              ▼                           ▼
      Ranked List A                Ranked List B
      (e.g., Doc1, Doc2)           (e.g., Doc2, Doc3)
              │                           │
              └─────────────┬─────────────┘
                            ▼
                [Reciprocal Rank Fusion]
                            │
                            ▼
                   [Final Blended Rank]
                   (e.g., Doc2, Doc1, Doc3)
```

### Reciprocal Rank Fusion (RRF) Math
RRF is an empirical ranking algorithm that scores documents based on their position in the dense and sparse search result lists, without needing to normalize scores between the two methods:
$$RRF(d) = \sum_{m \in M} \frac{1}{k + r_m(d)}$$
- $M$: The set of ranking systems (in this case, $M = \{\text{Dense}, \text{Sparse}\}$).
- $r_m(d)$: The rank position of document $d$ in system $m$ (1-indexed).
- $k$: A constant smoothing parameter (typically set to $60$).
- **Intuition**: Documents that appear high in both lists (small $r_m$) receive the highest RRF scores. A document ranked #1 in both systems gets a higher final score than one ranked #1 in only one system and #100 in another.

---

## Architecture: Go-Based HNSW

Weaviate implements **HNSW** directly in Go, bypassing external C++ libraries:
- **Garbage Collection Optimization**: Go is a garbage-collected language. To prevent memory scanning pauses (which would degrade query latencies when managing millions of graph nodes), Weaviate uses custom **off-heap storage** and memory-mapped files (mmap) to store vector indices.
- **Inverted Index Pairing**: Every class in Weaviate contains both a standard inverted index (for BM25 queries) and an HNSW graph index. When a write occurs, Weaviate updates both index structures transactionally.

---

## Components

1. **GraphQL API Gateway**: The main query ingress that compiles GraphQL schemas into execution plans.
2. **Vector Index (HNSW)**: The spatial index handling semantic vector retrieval.
3. **Inverted Index (BM25)**: The token-based dictionary indexing words and calculating Term Frequency-Inverse Document Frequency (TF-IDF) weights.
4. **Vector Modules (External/Local Inference)**: Plugins (e.g., `text2vec-openai`) that call API models to embed text during data ingestion and query execution.

---

## Design Decisions & Trade-offs

### Hybrid vs. Vector-Only Search

- **Hybrid Search (Weaviate)**:
  * *Pros*: High query accuracy. Captures both precise keyword hits and abstract conceptual meanings. Easy RRF tuning.
  * *Cons*: Higher CPU and RAM usage. Updates require maintaining two distinct indexes (graph and inverted list).
- **Vector-Only Search**:
  * *Pros*: Faster query times, smaller index footprint.
  * *Cons*: Poor performance on typos, abbreviations, codes, or exact match strings.

---

## Cost Optimization

- **Flat Vector Index**: If storage cost is a priority and query volume is low, Weaviate allows configuring flat vector indexing (bypassing graph building) which eliminates memory overhead at the expense of search speed ($O(N)$ scanning).
- **Replication Factor Tuning**: Use multi-node replication only for critical search indexes, keeping non-critical logging datasets on single-node shards.

---

## Interview Questions

### Q1: What is Reciprocal Rank Fusion (RRF)? Why is it used instead of adding normalized distance scores?
**Answer**:
1. **Reciprocal Rank Fusion (RRF)** is a rank-aggregation method that combines rankings from multiple retrieval systems (like dense vector search and sparse keyword search) by summing the reciprocals of their rank positions.
2. **Why not add normalized scores?**: 
   - Vector search returns distance metrics like Cosine Similarity ($[-1, 1]$) or L2 Distance ($[0, \infty)$).
   - BM25 keyword search returns open-ended relevance scores based on term frequency ($[0, \infty)$).
   - These scores are scale-independent and cannot be directly compared or added without complex normalization functions.
   - RRF avoids this issue by ignoring the raw scores entirely and focusing strictly on the **rank order** of documents. This makes it highly robust to changes in models, datasets, or scoring ranges.

### Q2: Why is implementing HNSW in Go challenging? How does Weaviate overcome it?
**Answer**:
1. **Garbage Collection (GC) Overhead**: Go is a garbage-collected language. If Weaviate represented millions of HNSW graph nodes and edges as standard Go pointer-based structs on the heap, the Go GC would have to traverse millions of objects during every collection cycle. This would cause massive CPU spikes and pause-the-world latencies.
2. **Weaviate's Solution**: Weaviate uses **LSM-Tree based storage (BoltDB/RocksDB derivatives)** and stores the vector arrays and graph connectivity lists off-heap or in memory-mapped files (`mmap`). This tells the Go runtime to ignore these large chunks of memory, keeping heap size small and GC pause times under a millisecond.

---

## References

1. **Reciprocal Rank Fusion**: Cormack, G. V., Clarke, C. L., & Buettcher, S. (2009). *Reciprocal Rank Fusion in a Link-Based Multi-Retrieval Environment*. ACM SIGIR.
2. **BM25 Search Algorithm**: Robertson, S., & Zaragoza, H. (2009). *The Probabilistic Relevance Framework: BM25 and Beyond*. Information Retrieval.
