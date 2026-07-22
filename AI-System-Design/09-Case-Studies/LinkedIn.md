# Case Study: LinkedIn (PYMK & Feed Ranking Graph Architectures)

## Overview

**LinkedIn** is the world's largest professional social network. Its platform architecture relies on two key recommendation systems: **People You May Know (PYMK)**—which suggests professional connections using Graph Neural Networks (GNNs) and ego-graph traversals—and the **Homepage Feed Ranking** engine—which uses multi-objective optimization models to rank updates, articles, and posts in real time.

---

## Problem Statement

Scaling a professional social graph and feed ranking engine introduces unique architectural challenges:
1. **Massive Graph Operations**: Resolving second-degree connections (friends-of-friends) across a graph of hundreds of millions of members and billions of edges can cause severe database bottlenecks if calculated dynamically.
2. **Multi-Objective Value Balancing**: Optimizing the feed strictly for clicks leads to clickbait. The system must balance multiple, competing user actions: likes, comments, shares, long-form reading time (dwell time), and professional relevance.
3. **Graph Clustering Latency**: Graph Neural Networks are computationally expensive. Generating real-time graph node embeddings for active connection recommendations cannot run synchronously during the user query path.

---

## High-Level System Architecture

LinkedIn's architecture splits operations between a graph-processing pipeline and a multi-stage feed ranking serving engine:

```
  [Social Graph Database] ──> [Offline Graph Cluster] ──> [GNN Node Embeddings]
  (Member nodes & edges)     (GNNs / Ego-Graph walks)    (Stored in HDFS / Redis)
                                                                    │
  [Feed Content Pool] ──────────────────────────────────────────────┼──┐
           │                                                        │  │
           ▼                                                        ▼  ▼
   [Retrieval Layer] ───────────────────────────────────> [Feature Extraction]
   (Retrieves posts from connections & hashtags)          (Loads GNN vectors)
           │                                                        │
           ▼                                                        ▼
   [Multi-Stage Ranker Engine] ◄────────────────────────────────────┘
   ├─ Stage 1: Fast Scoring (Linear Models / LightGBM)
   └─ Stage 2: Heavy Scoring (Deep Multi-Task Neural Net)
           │
           ▼
    [Client UI Feed]
```

### 1. PYMK & Graph Neural Networks (GNNs)
- **The Social Graph**: Members are represented as **Nodes**; connections, messages, and follows are represented as **Edges**.
- **GNN Embedding Generation**: LinkedIn uses Graph Neural Networks to compile node embeddings. The GNN aggregates feature information from a member's immediate neighbors (first-degree connections) and passes it through neural layers to represent the member's graph location:
  $$h_v^{(k)} = \sigma \left( W_k \cdot \text{Aggregate} \left( h_v^{(k-1)}, \{h_u^{(k-1)} : u \in \mathcal{N}(v)\} \right) \right)$$
- **Link Prediction (PYMK)**: To suggest new connections, the system computes the cosine similarity between node embeddings. Node pairs with high similarity that do not have an active edge are flagged as candidate connections.
- **Offline Processing**: GNN embedding generation runs in batch on distributed GPU clusters, saving completed vectors to an online low-latency lookup store (**Redis**).

---

### 2. Homepage Feed Ranking & Multi-Objective Optimization
The LinkedIn feed is compiled via a multi-stage pipeline:
- **Retrieval**: Gathers thousands of candidate posts from a member's network, followed hashtags, and channels.
- **First Stage (Lightweight Ranking)**: A fast model (e.g., LightGBM) filters the candidates down to the top $100$ posts.
- **Second Stage (Multi-Task Deep Learning)**: A deep neural network processes the top $100$ posts. It has multiple output heads, each predicting a different probability:
  * $P(\text{Like})$: Probability of user liking the post.
  * $P(\text{Comment})$: Probability of user commenting on the post.
  * $P(\text{Share})$: Probability of user sharing the post.
  * $E(\text{DwellTime})$: Expected reading time spent on the post.
- **Utility Function Blend**: The individual probabilities are merged into a single score using customizable business weights:
  $$\text{Score} = w_{\text{like}} P(\text{Like}) + w_{\text{comment}} P(\text{Comment}) + w_{\text{share}} P(\text{Share}) + w_{\text{dwell}} E(\text{DwellTime})$$
  Posts are sorted by this score before rendering.

---

## Design Decisions & Trade-offs

### Multi-Task Neural Networks vs. Individual Single-Metric Models

- **Multi-Task Neural Network (LinkedIn)**:
  * *Pros*: Single forward pass computes predictions for all target actions (likes, comments, shares, dwell time), saving CPU cycles. Shared lower layers learn general representations, preventing overfitting.
  * *Cons*: Difficult to train. Balancing gradients from different tasks (e.g., clicks vs. shares) requires complex optimizer configurations.
- **Individual Models**:
  * *Pros*: Simple to build and deploy. Teams can iterate on the "Like model" without affecting the "Share model".
  * *Cons*: Running multiple separate models in parallel on the serving path increases CPU latency and database lookup overhead.

---

## Interview Questions

### Q1: How does LinkedIn's People You May Know (PYMK) recommend connections? Walk through the graph traversal mechanics.
**Answer**:
1. **Candidate Retrieval**: The system does not scan the entire database. It targets the user's **Ego-Graph** (first-degree connections and their immediate connections - second-degree friends).
2. **GNN Embeddings**: Offline batch pipelines run Graph Neural Networks. These networks aggregate node attributes (industry, company, job title) and graph connectivity patterns to produce a dense vector for each member.
3. **Similarity Scoring**: For the retrieved candidates, the system looks up their GNN vectors from a Redis database and calculates the cosine similarity.
4. **Ranking & Filtering**: Candidates are ranked by cosine similarity and filtered to exclude existing connections or rejected invitations, returning the final People You May Know list to the user interface.

### Q2: Why is "Dwell Time" (time spent reading a post) monitored and optimized in feed ranking algorithms instead of just clicks?
**Answer**:
1. **Clickbait Prevention**: Optimizing feed models strictly for clicks (CTR) encourages sensationalist headlines or sensational images. Users click, realize the content is low-quality, and leave immediately, leading to a poor user experience.
2. **Engagement Quality Indicator**: **Dwell Time** measures active reading or video watch duration. A long dwell time is a strong indicator of genuine user engagement and content relevance.
3. **Multi-Objective Balance**: By including Expected Dwell Time ($E(\text{DwellTime})$) as a core term in the multi-objective utility scoring function, the feed optimizer balances click-through rates with high-retention content, promoting professional value and long-term user retention.

---

## References

1. **LinkedIn PYMK Engine**: *Scaling People You May Know Recommendations*. (LinkedIn Engineering Blog).
2. **Graph Neural Networks**: Hamilton, W., Ying, Z., & Leskovec, J. (2017). *Inductive Representation Learning on Large Graphs (GraphSAGE)*. NeurIPS.
3. **Multi-Objective Feed Ranking**: *Multi-Objective Optimization in Homepage Feed Ranking at LinkedIn*. (LinkedIn Engineering Publications).
