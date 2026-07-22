# Case Study: Uber (Michelangelo ML Platform)

## Overview

**Uber Michelangelo** is an enterprise-grade, end-to-end Machine Learning Platform designed to orchestrate ML workflows across Uber's global operations. It powers critical real-time features like dynamic pricing (surge), driver-rider matching, and high-precision Estimated Time of Arrival (ETA) predictions. The platform decouples training, feature engineering, and model serving while guaranteeing mathematical feature consistency between offline training and online inference.

---

## Problem Statement

Operating real-time machine learning services at global scale introduces spatial-temporal and operational challenges:
1. **Offline/Online Feature Drift**: Features computed in batch offline (e.g., a driver's average ride completion rate over the last 30 days) must match the mathematical representation of features calculated online in real time. Discrepancies lead to degraded model accuracy.
2. **Spatial-Temporal Density**: Modeling traffic, supply, and demand requires partitioning the physical world into logical, searchable grid cells that can resolve operations under millisecond latency SLA constraints.
3. **High-Throughput Model Serving**: ETA models are called millions of times per second. Routing requests to external REST model servers adds network overhead that can bottleneck the dispatch pipeline.

---

## Michelangelo Platform Architecture

Michelangelo divides the machine learning lifecycle into dedicated training, feature, and serving layers:

```
                            [Data Sources (Kafka / Hive)]
                                          │
                   ┌──────────────────────┴──────────────────────┐
                   ▼                                             ▼
       [Offline Batch Engine]                         [Real-time Stream Engine]
       (Apache Spark / Hive)                          (Apache Flink)
                   │                                             │
                   ▼                                             ▼
       [Offline Feature Store]                        [Online Feature Store]
       (Hive / HDFS tables)                           (Cassandra / Redis)
                   │                                             │
                   ▼                                             ▼
       [Offline Model Training]                      [Online Serving Gateway]
       (Spark ML / PyTorch / GPU)                     (Java Serving Container)
                   │                                             │
                   └──────────────────────┬──────────────────────┘
                                          ▼
                               [Model Registry & Deploy]
```

### 1. The Dual-Store Feature Architecture
To ensure feature consistency, Uber uses a **Shared Feature Store**:
- **Offline Feature Store**: Compiles complex historical aggregates (e.g., restaurant preparation times over the past month) in batch using Spark, saving them to Hive tables for model training.
- **Online Feature Store**: Loads a copy of the pre-compiled offline features into a low-latency database (**Apache Cassandra / Redis**). Real-time services look up these pre-computed features in milliseconds during the dispatch request.
- **Consistency Guarantee**: Features are declared using a unified schema config file, ensuring that the same feature extraction code is applied in both training and serving pipelines.

---

### 2. Spatial Indexing via H3
To resolve spatial features (e.g., calculating driver density around a rider), Uber uses **H3**:
- **Hexagonal Hierarchical Spatial Index**: H3 partitions the surface of the Earth into a grid of hexagonal cells.
- **Why Hexagons?**: Unlike squares (where the distance from the center to corners differs from center to edges), hexagons have equal distance between the center point and all adjacent neighbors. This simplifies spatial radius query calculations and smoothing algorithms.
- **Execution**: Coordinates (lat/lon) are binned into H3 cell IDs. Feature stores look up metrics (e.g., active drivers) mapped to specific H3 cell IDs in real time.

---

### 3. High-Throughput Serving (In-Process JVM)
To meet the extreme QPS demands of ETA calculations:
- Uber avoids calling remote model endpoints over HTTP/gRPC.
- **In-Process Java Serving**: Models are compiled into optimized Java formats (e.g., compiled tree structures) and loaded directly into the memory of the client dispatch services (JVM). This eliminates network serialization latency, reducing model evaluation times to $<5\text{ ms}$.

---

## Design Decisions & Trade-offs

### In-Process Model Serving vs. REST Microservice Serving

- **In-Process Model Serving (JVM Memory)**:
  * *Pros*: Lowest possible latency ($<1\text{ ms}$ evaluation). Zero network hops. Extremely reliable.
  * *Cons*: Language lock-in (serving applications must match the host JVM container). Updating a model requires redeploying or dynamically hot-reloading code in the active client application.
- **REST Microservice Serving (e.g., Triton, SageMaker)**:
  * *Pros*: Language-agnostic, decouples client app dependencies from model libraries. Simple scaling of model nodes independently.
  * *Cons*: Introduces network hop latency and serialization/deserialization overhead.

---

## Interview Questions

### Q1: What is the Offline/Online Feature Drift problem in MLOps, and how does Uber's Michelangelo Feature Store solve it?
**Answer**:
1. **The Problem**: Feature drift occurs when the mathematical logic used to compute features during model training (offline) differs from the logic used during model serving (online). For example, if the offline training pipeline calculates "average trip duration" using UTC time, but the online serving pipeline uses local server time, the model will receive incorrect inputs, leading to poor predictions.
2. **Uber's Solution**:
   - **Unified Schema Definition**: Feature configurations are defined once in a centralized registry using a domain-specific language (DSL).
   - **Dual-Store Sync**: The offline batch pipeline (Spark) computes features, saves them to the Offline Store (Hive), and automatically synchronizes the identical data assets to the Online Store (Cassandra/Redis).
   - **Shared Libraries**: Serving containers use the same feature computation libraries as the training pipeline to process real-time inputs, guaranteeing mathematical alignment.

### Q2: Why does Uber use H3 (Hexagons) instead of standard square grids for spatial indexing?
**Answer**:
1. **Equal Neighbor Distance**: In a square grid, a cell has two types of neighbors: those sharing an edge (distance $1$) and those sharing a corner (distance $\sqrt{2}$). In a hexagonal grid, every cell shares an edge with exactly 6 neighbors, and the distance from the center of a hexagon to the centers of all 6 neighbors is identical.
2. This simplifies spatial calculations (like k-nearest neighbors or grid smoothing algorithms) because there are no diagonal distortions.
3. **Hierarchical Indexing**: H3 supports nested resolutions, allowing grid cells to be aggregated from small meter-sized hexagons up to city-sized zones using simple integer bit-shifts, optimizing spatial database search performance.

---

## References

1. **Uber Michelangelo Platform**: *Meet Michelangelo: Uber's Machine Learning Platform*. (Uber Engineering Blog).
2. **H3 Spatial Index**: *H3: Uber's Hexagonal Hierarchical Spatial Index*. (Uber Open Source Project).
3. **Feature Store Design**: *Designing a Feature Store for Real-Time Machine Learning*. (ACM KDD).
