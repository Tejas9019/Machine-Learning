# Vector Databases

## Overview

A Vector Database is a specialized storage engine designed to store, index, and query high-dimensional vector representations. Unlike relational databases that perform exact matching on discrete keys, vector databases execute **Approximate Nearest Neighbor (ANN)** search to retrieve vectors based on spatial similarity.

---

## Problem Statement

To find the nearest neighbor of a query vector in a database of size $N$ using exact search (flat index), we must calculate the distance to all $N$ vectors. At scale ($N > 10^7$ vectors), this $O(N)$ linear scan becomes computationally expensive and introduces latencies of seconds, making it unusable for real-time production SLAs ($< 50\text{ms}$).

Vector databases solve this by organizing spatial indices that trade absolute accuracy (recall) for logarithmic $O(\log N)$ query speeds.

---

## Architecture

The dual pipeline structure of a production-grade Vector Database (Write Path vs. Read Path):

<img src="../diagrams/vector_db_pipelines.png" alt="Vector Database Ingestion and Query Pipelines Diagram" width="100%" />

---

## Components

### 1. HNSW (Hierarchical Navigable Small World) Graph Indexing
HNSW is a multi-layer graph-based index. Upper layers contain sparse connections for fast routing across large distances, while lower layers contain dense clusters for fine-grained search. 
* **Layer Assignment**: When a new vector is inserted, its maximum layer $l$ is determined using an exponential decay probability distribution to ensure upper layers remain sparse:
$$l = \lfloor -\ln(\text{uniform}(0, 1)) \cdot m_L \rfloor$$
  * *Where $m_L$ is a normalization parameter regulating layer density.*
* **Search Execution**: 
  1. Search starts at the entry point of the top layer.
  2. The algorithm greedily traverses nodes, moving to neighbors that are closer to the query vector.
  3. When it reaches a local minimum (no closer neighbors), it steps down to the next layer and resumes search.
  4. At Layer 0, it performs a search using a priority queue of size `efSearch` to find the final candidate list.

```
HNSW Multi-Layer Graph structure:
Layer 2 (Sparse)      ●───────────────●
                      │               │
Layer 1 (Medium)      ●───────●───────●───────●
                      │       │       │       │
Layer 0 (Dense)       ●───●───●───●───●───●───●
```

### 2. IVF (Inverted File) Indexing
IVF partitions the high-dimensional vector space using K-Means clustering.
* **Posting Lists**: During indexing, vectors are assigned to the nearest centroid. The index stores a map of centroid IDs to list of vectors (posting lists).
* **Search Execution**: 
  1. The query vector is compared against all cluster centroids.
  2. The search is restricted to the posting lists of the $P$ closest centroids (`nprobe` parameter).
  3. Restricting search to these centroids yields $O(K \cdot \frac{N}{C})$ search time rather than $O(N)$ (where $C$ is total centroids).

### 3. Product Quantization (PQ) Compression & Asymmetric Distance (ADC)
PQ divides a $D$-dimensional vector into $M$ smaller sub-vectors of dimension $D/M$. For each sub-space, K-Means clustering is run to find $K^*$ centroids.
* **Codebook Generation**: Each vector is compressed into a sequence of $M$ integers, where each integer points to the index of the nearest sub-space centroid. For $K^* = 256$, each sub-vector is represented by an 8-bit integer.
* **Asymmetric Distance Computation (ADC)**: During query time, the query vector is not quantized. Instead, the distances between the unquantized query sub-vectors and all $K^*$ codebook centroids are computed and stored in a lookup table. The distance to any quantized database vector is calculated by summing values from the lookup table, eliminating vector decompression overhead:
$$d(q, y_{\text{quantized}}) \approx \sum_{i=1}^M \|q_i - \mathbf{c}_{i, y_i}\|^2$$
  * *Where $q_i$ is query sub-vector $i$, and $\mathbf{c}_{i, y_i}$ is centroid $y_i$ in sub-space $i$.*

### 4. Metadata Filtering Strategies
* **Pre-Filtering**: Evaluates metadata filters first, then runs vector search on the matching subset. If the filter is highly restrictive (e.g., matching only 0.1% of vectors), graph connections can be broken, preventing HNSW traversal.
* **Post-Filtering**: Runs the vector search first to find the top $K$ nearest neighbors, then discards results that do not match metadata criteria. This risks returning fewer than the requested $K$ results.
* **Single-Stage (Iterative) Filtering**: Traverses the graph and checks metadata validity on the fly during search. If a neighbor does not match metadata filters, it is skipped but its neighbors are still traversed.

---

## Design Decisions

| Metric / Feature | HNSW (Graph-based) | IVF-PQ (Quantized Cluster) |
| :--- | :--- | :--- |
| **Search Speed** | **Extremely Fast** ($O(\log N)$) | Moderate (Scale depends on `nprobe` settings) |
| **RAM Footprint** | **High**: Storing the graph structure and vectors in RAM is required. | **Low**: Compressed vectors and indices can be stored on disk. |
| **Recall Rate** | **95% - 99%** | 80% - 90% |
| **Write Latency** | Slow: Requires updating graph edges for each new node. | Medium: Requires periodically recalculating K-means clusters. |

---

## Scaling

### RAM Sizing Calculations for HNSW Indices
For a database containing $N$ vectors of dimension $d$ in an HNSW index, the required RAM is calculated as:
$$\text{Memory}_{\text{HNSW}} \approx N \times (d \times 4 + M \times 8) \text{ bytes}$$
* $d \times 4$: Raw float32 vector coordinates.
* $M \times 8$: The graph connections. Each node has $M$ bi-directional connections, and each connection stores an 8-byte memory address pointing to a neighbor node.
* *Example (10 million vectors, 1536 dims, $M=16$)*:
$$\text{RAM} \approx 10^7 \times (1536 \times 4 + 16 \times 8) = 10^7 \times (6144 + 128) \approx 62.7 \text{ GB}$$

### Distributed Sharding Protocols
* **Scatter-Gather (Document-ID Sharding)**: Distributes vectors randomly across nodes. Query is broadcast to all shards, and the master node merges results. High network traffic bottleneck.
* **Routing Sharding (Centroid-based)**: Groups similar vectors on the same nodes. Queries are routed only to the nodes storing vectors close to the query space, minimizing network overhead.

---

## Failure Handling

* **Index Degradation (Drift)**: As new vectors are inserted, the graph topology can degrade, leading to lower recall rates.
  * *Mitigation*: Implement a scheduled background job to reconstruct graph segments asynchronously.
* **OOM (Out-of-Memory) Crashes**: As data grows, HNSW can exceed RAM capacity.
  * *Mitigation*: Configure index storage to write raw vectors to NVMe SSDs while holding only the HNSW graph index in RAM.

---

## Security

* **Multi-Tenant Leakage**: Users accessing document embeddings belonging to other organizations.
  * *Mitigation*: Implement namespace isolation or partition vectors physically at the storage layer.
* **Vector Hijacking (Evasion)**: Crafting queries designed to exploit HNSW traversal logic, trapping the search in local minima to hide target data.
  * *Mitigation*: Introduce random restarts during graph traversal to ensure coverage.

---

## Cost Optimization

* **Product Quantization (PQ)**: Compressing 1536-dimension float32 vectors (6144 bytes) to 96 bytes by quantizing them. This reduces memory footprint by up to **98%**, allowing large databases to run on significantly cheaper hardware.

---

## Interview Questions

### 1. Explain the role of the parameters M, efConstruction, and efSearch in HNSW index configuration.
**Answer**:
* **M**: The maximum number of connection links per node on each layer. Higher $M$ values improve recall on high-dimensional data but increase index size.
* **efConstruction**: The size of the dynamic candidate list evaluated during index construction. Increasing it builds higher-quality graph indices but increases indexing time.
* **efSearch**: The size of the dynamic candidate list evaluated during query time. Increasing it improves query recall at the cost of higher query latency.

### 2. How does Single-Stage (Iterative) Filtering solve the graph disconnection problem in HNSW search?
**Answer**: In Pre-filtering, nodes that fail metadata checks are removed from the search space before traversal. If a filter matches only a few documents, the graph becomes fragmented, and search cannot traverse the disconnected components. Single-Stage filtering keeps the entire graph connected. The search traverses all nodes but checks metadata criteria dynamically. If a node fails, it is excluded from the final results, but the search can still traverse through it to reach other nodes.

---

## References

* [Malkov & Yashunin (2016): Efficient and Robust Approximate Nearest Neighbor Search Using HNSW Graphs](https://arxiv.org/abs/1603.09320)
* [Milvus Architecture: Vector Database Design Patterns](https://milvus.io/docs/architecture_overview.md)
* [Pinecone Engineering Blog: Spatial Indexing & Quantization](https://www.pinecone.io/learn/)
