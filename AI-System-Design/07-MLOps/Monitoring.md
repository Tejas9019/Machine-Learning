# Model Monitoring & Observability

## Overview

**Model Monitoring & Observability** is the practice of tracking, analyzing, and alerting on the performance of machine learning models in production. Because machine learning models are dependent on external real-world data, they degrade over time due to shifts in input patterns. Observability architectures collect inference logs, calculate statistical distance metrics (such as the Kolmogorov-Smirnov test and Population Stability Index), and track both system-level and model-level performance indicators to detect issues before they impact business metrics.

---

## Problem Statement

Serving models in production exposes them to silent degradation:
1. **Silent Performance Decay**: Traditional software crashes or logs errors. An ML model continues to predict values with high confidence, but the underlying accuracy degrades because user behaviors have changed (e.g., a recommendation model failing during a sudden holiday season).
2. **Delayed Ground Truth**: In tasks like fraud detection or loan risk assessment, the actual outcome (ground truth) may not be known for months. The system cannot rely on traditional accuracy metrics to detect failures and must monitor intermediate statistical shifts instead.
3. **High-Volume Logging Latency**: Writing every input text, vector embedding, and output prediction to a database during the query path adds severe network latency and risks blocking the core model serving engine.

---

## Observability Architecture & Log Ingestion

To monitor models without introducing latency to the serving path, logging is executed asynchronously:

```
[Inference Request] ──> [Model Serving Engine] ──> [Response to User]
                                │ (Asynchronous payload clone)
                                ▼
                       [Message Queue (Kafka)]
                                │
                                ▼
                    [Log Ingestion Processor]
                                │
         ┌──────────────────────┴──────────────────────┐
         ▼                                             ▼
 [System Health Metrics]                      [Analytical Data Lake]
 (Prometheus / Grafana)                       (Parquet files on S3)
                                                       │
                                                       ▼
                                            [Drift Detection Service]
                                            (Daily Spark jobs / KS-Tests)
```

1. **API Ingress & Asynchronous Payload Copy**: When Triton or vLLM executes a request, it sends a payload clone (containing inputs, outputs, model version, and timestamp) asynchronously to a message broker (e.g., Apache Kafka), keeping the user's critical path fast.
2. **Log Ingestion Processor**: A streaming consumer reads the payload logs, parses the raw features, and writes them to:
   - **TimeSeries Databases (Prometheus)**: For short-term aggregate metrics (e.g., average prediction scores, request counts).
   - **Data Lake (S3/Iceberg)**: For long-term historical analytical logging.
3. **Drift Detection Service**: A daily batch job (e.g., running on Spark) compares the distribution of the previous day's production inputs against the baseline training validation dataset.

---

## Types of Model Drift

Mathematical degradation is categorized into three types:

### 1. Data Drift (Covariate Shift)
- **Definition**: A change in the distribution of the input features over time, while the relationship between input features and target outputs remains constant:
  $$P(X_{\text{production}}) \neq P(X_{\text{training}})$$
- *Example*: An image classification model trained on high-resolution daytime photos is deployed to cameras that capture low-resolution nighttime photos.

### 2. Concept Drift
- **Definition**: A change in the relationship between input features and the target label, while the input distribution itself may remain unchanged:
  $$P(Y|X_{\text{production}}) \neq P(Y|X_{\text{training}})$$
- *Example*: A house price prediction model where the physical characteristics of houses ($X$) remain identical, but a sudden economic crisis changes their market value ($Y$).

### 3. Prior Probability Shift
- **Definition**: A change in the distribution of the target labels:
  $$P(Y_{\text{production}}) \neq P(Y_{\text{training}})$$
- *Example*: A spam classifier deployed during an election year where the percentage of spam emails in the wild doubles.

---

## Mathematical Drift Detection Algorithms

To flag data drift, systems calculate statistical distances between production sample distributions ($P$) and training baseline distributions ($Q$):

### 1. Kolmogorov-Smirnov (KS) Test
- **Use Case**: Used for continuous numerical features (e.g., confidence scores, user age).
- **Math**: A non-parametric test comparing the cumulative empirical distribution functions (CDFs) of two samples:
  $$D = \sup_x |F_1(x) - F_2(x)|$$
- **Alert**: If the maximum vertical distance $D$ exceeds a threshold corresponding to a low p-value (typically $p < 0.05$), the null hypothesis is rejected, indicating that the production dataset does not follow the training distribution.

### 2. Population Stability Index (PSI)
- **Use Case**: Used to measure shifts in categorical variables or binned continuous variables.
- **Formula**:
  $$PSI = \sum_{i=1}^{B} \left( P_i - Q_i \right) \times \ln\left(\frac{P_i}{Q_i}\right)$$
  Where $P_i$ is the actual production percentage in bin $i$, and $Q_i$ is the baseline training percentage in bin $i$.
- **Thresholds**:
  * $PSI < 0.1$: No significant change.
  * $0.1 \le PSI < 0.25$: Moderate shift; triggers warning alerts.
  * $PSI \ge 0.25$: Significant shift; triggers automated model retraining pipelines.

---

## Interview Questions

### Q1: If the ground truth labels are delayed by 3 months, how can you detect if a model's performance has degraded in production?
**Answer**:
When ground truth is delayed, we cannot calculate direct performance metrics (such as Accuracy, F1-Score, or Precision) in real time. We must use **proxy indicators**:
1. **Monitor Data Drift (Covariate Shift)**: Use statistical tests like the Kolmogorov-Smirnov (KS) test (for continuous numerical inputs) or the Chi-Square test (for categorical inputs) to compare the distribution of incoming production features against the training baseline. If the inputs shift significantly, it is a strong proxy that the model's accuracy is likely degrading.
2. **Monitor Output Prediction Distribution**: Track the distribution of the model's output prediction probabilities. For instance, if a loan approval model historically predicts a $10\%$ default rate but suddenly starts predicting a $35\%$ default rate, this indicates an anomaly in input data or model behavior.
3. **Monitor System Metrics**: Track change signals in upstream pipelines (e.g., changes in feature schemas or data source systems).

### Q2: Why is logging inference payloads asynchronously preferred over synchronous logging in model serving?
**Answer**:
1. **Latency Overhead**: Deep learning and LLM serving applications operate under strict latency budgets (SLAs). Writing payload logs synchronously to a database during the request path adds network round-trips and disk write times to the user's query path, directly increasing response latency.
2. **System Availability**: If the monitoring database goes down or experiences write spikes, a synchronous logger would block the model serving engine, causing user requests to fail.
3. **Asynchronous Architecture**: ROUTING a cloned copy of the request/response payload to an asynchronous broker (such as Kafka or RabbitMQ) ensures that the serving engine returns the prediction to the user immediately. The logging and analytics processing run in the background, decoupling system reliability and serving speed from observability tasks.

---

## References

1. **Data Drift Analysis**: Tasche, D. (2017). *Fisher Consistency for Prior Probability Shift*. Journal of Machine Learning Research.
2. **PSI Formulation**: Yurdakul, B. (2018). *Statistical Properties of Population Stability Index*. Graduate Thesis, Western Michigan University.
3. **ML Observability Patterns**: *Monitoring Machine Learning Pipelines: System Metrics and Data Quality*. (Google Cloud Architecture Center).
