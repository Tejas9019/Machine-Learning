# MLflow: Model Tracking & Registry

## Overview

**MLflow** is an open-source platform designed to manage the machine learning lifecycle. It standardizes model development by providing APIs to track experiments, package code into reproducible runs, manage model versions, and transition models through staging-to-production lifecycles.

---

## Problem Statement

Developing machine learning models without a centralized lifecycle tool introduces operational issues:
1. **Lost Experiments**: Data scientists run thousands of training runs with different hyperparameters (e.g., learning rates, batch sizes). Keeping track of metrics in spreadsheets leads to lost progress and un-reproducible models.
2. **Lack of Model Lineage**: When a model misbehaves in production, tracing it back to the exact code commit, training script, database query, and hyperparameters is nearly impossible.
3. **Deployment Spaghetti**: Translating research scripts (Jupyter notebooks) to deployable API microservices requires manual, error-prone recoding.

---

## MLflow Architecture & Key Components

MLflow is organized into four decoupled components that communicate via standard REST APIs:

```
            [Data Scientist / Training Job]
                           │
      ┌────────────────────┼────────────────────┐
      ▼                    ▼                    ▼
[MLflow Tracking]   [MLflow Projects]    [MLflow Models]
(Params, Metrics,   (Code Packaging,     (Standardized
 Artifacts Storage)  MLproject config)    Model Formats)
      │
      └────────────────────┬────────────────────┘
                           │
                           ▼
                   [MLflow Registry]
                   (Model Versioning, Stage Transitions)
```

### 1. MLflow Tracking
Logs training parameters (e.g., learning rate), metrics (e.g., loss, validation accuracy), and artifacts (e.g., confusion matrix plots, weights files) for every run.
- **Backend Store**: Stores run metadata (parameters, metrics, tags). Typically a relational database (PostgreSQL or MySQL).
- **Artifact Store**: Stores large binary files (model weights, plots). Typically an object storage bucket (Amazon S3, Google Cloud Storage, or Azure Blob).

### 2. MLflow Projects
Provides a self-contained, standardized packaging format for code.
- Uses an `MLproject` text file in the root directory to define:
  * **Environment**: Conda, Docker, or Virtualenv requirements.
  * **Entry Points**: Python commands to run, along with input parameter declarations and data types.

### 3. MLflow Models
Specifies a standard format for packaging machine learning models so they can be consumed by various downstream deployment tools (e.g., Triton, SageMaker, or local REST APIs).
- Every model includes a `MLmodel` file describing different "flavors" (e.g., `python_function`, `pytorch`, `onnx`) under which the model can be run.

### 4. MLflow Model Registry
A centralized model database providing a collaborative workspace to manage model lifecycles:
- **Versioning**: Registering a run creates a new model version (e.g., `Version 1`, `Version 2`).
- **Stage Progression**: Controls transitions between predefined stages: `None`, `Staging`, `Production`, and `Archived`. Stage changes trigger hooks/webhooks to run automated CI checks before promotion.

---

## Design Decisions & Trade-offs

### Centralized MLflow Server vs. Distributed Logging

- **Centralized Tracking Server**:
  * *Pros*: Single pane of glass for all team experiments. Easy comparisons between runs. Strong security and access control over model registry promotions.
  * *Cons*: Single point of failure (SPOF) if the DB or S3 goes down; network latency during heavy artifact uploading from remote Kubernetes workers.
- **Local SQLite / Local Folder Tracking**:
  * *Pros*: Zero network latency, offline capability, zero server cost.
  * *Cons*: No collaboration, team members cannot view each other's runs, high risk of data loss.

---

## Scaling MLflow for Production

- **High Availability Metadata Store**: Deploy the Tracking Server backed by a managed relational database cluster (e.g., AWS RDS Aurora PostgreSQL) with read replicas to handle massive query volumes from visualization dashboards.
- **Direct-to-S3 Artifact Upload**: Configure MLflow clients to upload model artifacts directly to Amazon S3 (using signed URLs) rather than routing heavy binary files through the MLflow Tracking Server proxy, preventing network bottlenecking.

---

## Interview Questions

### Q1: What is Model Lineage? How does MLflow implement it?
**Answer**:
1. **Model Lineage** is the chronological record of the components and steps that created a machine learning model, tracing it from raw training data and code versions to the final production artifact.
2. **MLflow Implementation**:
   - When a model is registered in the **Model Registry**, it is linked to a unique **Run ID**.
   - The **Run ID** contains pointers to the exact training Git commit hash, the training parameters (learning rate, epochs), and the metrics (val loss).
   - The run also records the URI of the **input training dataset snapshot** (e.g., a specific delta table version or DVC hash).
   - This metadata enables engineers to trace any model running in production back to its source code, dependencies, and training hyperparameters, guaranteeing auditability and reproducibility.

### Q2: Why is the MLflow Models "flavor" concept useful?
**Answer**:
1. An ML model can be run in different software environments.
2. The MLflow **Flavor** concept allows packaging a model with multiple dynamic execution options inside its metadata file (`MLmodel`).
3. For example, a PyTorch model can be saved with:
   - A `pytorch` flavor (enabling loading it as a native PyTorch object in a Python script for further training).
   - A `python_function` (pyfunc) flavor (enabling loading it as a generic wrapper to run inside Spark UDFs or generic API microservices without needing PyTorch-specific code).
4. This decouples model creation from model deployment, allowing downstream serving tools to load models using standard abstractions.

---

## References

1. **MLflow Project Paper**: Zaharia, M., et al. (2018). *Accelerating the Machine Learning Lifecycle with MLflow*. IEEE Micro.
2. **Experiment Tracking Best Practices**: *Managing Machine Learning Metadata & Lineage*. (Google Cloud Architecture Center).
