# CI/CD for Machine Learning (Continuous Training & deployment)

## Overview

**CI/CD in MLOps** extends traditional software delivery pipelines (Continuous Integration and Continuous Delivery) to handle the unique lifecycles of machine learning systems. In addition to code, MLOps pipelines must orchestrate data validation, automated model retraining (Continuous Training, or CT), model evaluation benchmarks, and safe model promotion strategies (Shadow and Canary deployments).

---

## Problem Statement

Traditional software CI/CD pipelines fail when applied directly to machine learning deployments:
1. **Silent Failures**: Traditional software fails noisily due to syntax bugs or runtime crashes. ML systems fail silently when data distributions shift, producing mathematically incorrect predictions while returning HTTP 200 OK codes.
2. **Resource-Heavy Tests**: Running model evaluations on thousands of inputs during every Git commit commit is computationally expensive and slow, stalling developer velocity.
3. **High-Risk Promotions**: Swapping a new model into production immediately (blue-green cutover) can trigger cascading downstream failures if the new model's output schema or prediction distribution differs from the baseline.

---

## End-to-End MLOps GitOps Pipeline

A production-grade CI/CD and CT pipeline operates as an integrated feedback loop:

```
[Git Push / Commit] ──> [Code CI Tests] ──> [Docker Build & Push]
                                                  │
                                                  ▼
[Data Drift Alert] ──> [Continuous Training (CT)] ──> [Model Evaluation Suite]
                                                              │
                                                              ▼
[Canary Promotion] <── [Shadow Deploy] <── [MLflow Registry Promotion]
```

### 1. Continuous Integration (CI) for ML
When a developer modifies code or a CT trigger fires:
- **Code CI**: Runs standard unit tests (e.g., PyTest), linting, and security audits.
- **Data Validation (Sanity Checks)**: Validates incoming training datasets using schema testing libraries (like Great Expectations) to catch missing values, incorrect data types, or out-of-bounds values.
- **Model Evaluation**: Runs the newly trained model against a locked **Gold Evaluation Dataset**. It evaluates performance metrics (accuracy, F1-score, latency, memory consumption) and compares them to the active production model.

---

### 2. Continuous Training (CT) Triggers
Rather than retraining models manually, CT pipelines (orchestrated by Kubeflow or Airflow) are triggered by:
- **Schedule-Based**: Retraining daily or weekly.
- **Event-Driven**: Triggered when new training data partitions are registered in the Data Lake.
- **Drift-Driven**: Triggered automatically when monitoring systems detect significant data or concept drift in production logs.

---

### 3. Continuous Delivery & Deployment (CD) Strategies
To mitigate deployment risks, models are promoted using safe routing patterns:

#### A. Shadow Deployment
- **Mechanism**: The Ingress Gateway clones incoming live production requests. It sends one request copy to the Active Production Model (which returns the response to the user) and another copy to the New Candidate Model.
- **Benefits**: Validates the candidate model's real-time latency, memory usage, and prediction output profile under real production load without exposing users to potential model errors.
- **Drawbacks**: Requires twice the GPU infrastructure compute capacity since every request is executed twice.

#### B. Canary Deployment
- **Mechanism**: The router directs a small, configurable slice of production traffic (e.g., $5\%$) to the new model, while the remaining $95\%$ goes to the baseline model.
- **Execution**: If error metrics, latencies, and feedback indicators remain normal over a validation window (e.g., 24 hours), the traffic allocation is stepped up ($10\%, 25\%, 50\%, 100\%$).

#### C. Blue-Green Deployment
- **Mechanism**: Maintains two identical environments (Blue hosting the active model, Green hosting the new candidate). Once the Green environment is verified, the Ingress router swaps the target pointer instantly.
- **Benefits**: Instant rollback capability by swapping the pointer back if errors spike.

---

## Design Decisions & Trade-offs

### Shadow Deployment vs. Canary Deployment

- **Shadow Deployment**:
  * *Pros*: Zero risk to end users. Collects $100\%$ prediction logs for comparison. Great for verifying system-level load stability.
  * *Cons*: Double execution costs (GPU/CPU), cannot evaluate user-behavior feedback (e.g., click-through rate) since outputs are discarded.
- **Canary Deployment**:
  * *Pros*: Cost-effective (traffic is split, not duplicated), allows evaluating real user feedback and business conversion metrics.
  * *Cons*: Exposes a subset of real users ($5\%$) to potential model regression bugs.

---

## Interview Questions

### Q1: Describe the concept of Shadow Deployment. Why is it used in model serving, and what are its hardware limitations?
**Answer**:
1. **Concept**: Shadow Deployment duplicates production traffic at the API gateway. The active production model serves the user's request. In parallel, the new model processes a copy of the request, but its outputs are logged internally and discarded rather than returned to the user.
2. **Why it's used**: It is used to validate the new model's system performance (time-to-first-token, peak VRAM utilization, latency under peak traffic hours) and mathematical output correctness using live inputs, with zero risk of impacting end users.
3. **Hardware Limitations**:
   - **Compute Overhead**: Requires scaling up duplicate GPU nodes to run identical workloads in parallel, doubling infrastructure costs.
   - **Write Safety**: If the model pipeline writes to databases or triggers state changes (e.g., invoking payment gateways or sending notifications), the shadow path must be configured with mock APIs to prevent duplicate actions.

### Q2: What is Continuous Training (CT)? Walk through an automated trigger lifecycle.
**Answer**:
1. **Continuous Training (CT)** is the automation of the model retraining lifecycle, moving from manual, developer-triggered training runs to automated pipelines that react to data or performance updates.
2. **Trigger Lifecycle**:
   - **Step 1: Detection**: Model monitoring detects that production data distributions have shifted (data drift alert).
   - **Step 2: Trigger**: The monitoring alert triggers a webhook call to an orchestration engine (e.g., Kubeflow Pipelines).
   - **Step 3: Ingestion & Training**: The pipeline downloads the latest ground-truth dataset from the feature store, spins up a distributed `PyTorchJob`, and trains the model.
   - **Step 4: Evaluation**: The pipeline compares the new model's evaluation metrics against the active production model.
   - **Step 5: Registration**: If the new model beats the production baseline, it is registered in the Model Registry and flagged for approval, initiating the CD rollout.

---

## References

1. **Continuous Delivery for ML**: *Continuous Delivery for Machine Learning: Practical Guide to MLOps*. (Martin Fowler Blogs).
2. **Google Cloud MLOps whitepaper**: *MLOps: Continuous delivery and automation pipelines in machine learning*. (Google Cloud Architecture Guides).
