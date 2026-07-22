# Kubeflow: ML Orchestration on Kubernetes

## Overview

**Kubeflow** is an open-source, cloud-native machine learning platform built on top of Kubernetes. It acts as an orchestrator for ML workloads, converting complex training pipelines into directed acyclic graphs (DAGs), coordinating distributed training jobs across multi-GPU nodes, and automating hyperparameter tuning sweeps.

---

## Problem Statement

Running machine learning workloads in production introduces unique infrastructure challenges:
1. **Ad-hoc Pipeline Fragility**: Running data prep, model training, and evaluation as a series of manual scripts or disjointed cron jobs makes pipelines brittle and hard to trace.
2. **Distributed Training Orchestration**: Configuring multi-node distributed training (e.g., PyTorch DistributedDataParallel) on Kubernetes requires manual configuration of head and worker pod IP addresses, environment variables, and network configurations.
3. **Hyperparameter Tuning Waste**: Running grid searches manually across dynamic GPU resources leads to poor GPU utilization, resource leakage, and slower experimentation cycles.

---

## Kubeflow Architecture & Components

Kubeflow integrates multiple dedicated Kubernetes Custom Resources (CRDs) and operators into a cohesive ML control plane:

```
                      [Kubeflow Pipelines SDK]
                                  │
                                  ▼
                    [Kubeflow Pipelines Engine]
                    (Argo Workflows / Tekton DAGs)
                                  │
         ┌────────────────────────┼────────────────────────┐
         ▼                        ▼                        ▼
[Training Operators]       [Katib Optimizer]       [Metadata Store]
(PyTorchJob / TFJob)      (AutoML Hyperparameters) (Artifact Lineage DB)
         │                        │                        │
         └────────────────────────┼────────────────────────┘
                                  ▼
                     [Kubernetes GPU Cluster]
```

### 1. Kubeflow Pipelines (KFP)
Kubeflow Pipelines allows engineers to build end-to-end, reusable workflows.
- **Python SDK**: Developers write pipelines as Python functions decorated with `@pipeline`. The SDK compiles the Python code into a YAML file representing a Directed Acyclic Graph (DAG).
- **Execution Engine**: The compiled DAG is run by a workflow engine (like Argo Workflows). Each step in the pipeline is executed inside its own isolated Docker container.
- **Artifact Passing**: Data between steps (e.g., a path to a generated dataset or model checkpoint) is passed via Kubernetes Persistent Volume Claims (PVCs) or cloud object storage, tracked in the Kubeflow Metadata store.

---

### 2. Training Operators (Distributed Training)
To run distributed training (where multiple GPUs across different servers train a single model in parallel), Kubeflow provides custom operators:
- **PyTorchJob / TFJob / MPIJob**:
  - Instead of manually setting up multi-node configurations, engineers write a YAML declaration defining the master/worker replica count and container details.
  - The operator spins up the pods, mounts GPU devices, sets up network discovery (creating service endpoints for workers), and injects clustering variables (e.g., `MASTER_ADDR`, `MASTER_PORT`, `WORLD_SIZE`).
  - Supports **Ring-AllReduce** algorithms (like PyTorch NCCL) to synchronize weight gradients across nodes efficiently.

---

### 3. Katib (AutoML & Hyperparameter Tuning)
Katib automates search sweeps to find the best model configuration:
- **Trial Orchestration**: Katib schedules multiple pods (Trials) running parallel training experiments with varying configurations.
- **Search Algorithms**: Supports Random Search, Bayesian Optimization, and **Hyperband** (which cuts short poorly performing trials early to save compute budget).
- **Control Loop**: Monitors trial metrics (e.g., validation accuracy) reported by pods, dynamically spawning new trials using updated parameter configurations until convergence.

---

## Design Decisions & Trade-offs

### Kubeflow Pipelines vs. Apache Airflow

- **Kubeflow Pipelines (KFP)**:
  * *Pros*: Kubernetes-native. Every task runs in its own isolated container, avoiding dependency conflicts. Out-of-the-box support for distributed GPU training operators and Kubernetes node scheduling.
  * *Cons*: Heavy infrastructure overhead. Requires running a full Kubernetes cluster with complex CRDs. Overkill for simple data transformation pipelines.
- **Apache Airflow**:
  * *Pros*: Excellent for general data engineering, large community, rich integration with external SaaS products (Snowflake, Databricks).
  * *Cons*: Task execution runs on shared workers by default (risk of dependency conflicts), lacks native Kubernetes GPU scheduling and orchestration features.

---

## Scaling & Fault Tolerance

- **Checkpoint Resiliency**: Distributed training jobs are configured to write checkpoints to shared persistent volumes (e.g., AWS EFS). If a worker pod crashes, the coordinator schedules a replacement pod, which loads the latest checkpoint to resume training without starting over.
- **Node Taints & Tolerations**: Set up GPU nodes with taints (e.g., `gpu=true:NoSchedule`) and model training pods with matching tolerations. This prevents general CPU-based web server workloads from occupying expensive GPU compute nodes.

---

## Interview Questions

### Q1: How does Kubeflow Pipelines pass data between two sequential steps in a workflow if they run in completely isolated container pods?
**Answer**:
1. In Kubeflow Pipelines, steps run in isolated pods and cannot share local memory or disk storage directly.
2. Data is passed using **Cloud Object Storage (S3/GCS)** or **Kubernetes Persistent Volumes (PV/PVC)**.
3. When Step A finishes (e.g., data preprocessing), it writes its output (e.g., a `.parquet` file) to a designated object storage path (e.g., `s3://ml-bucket/pipeline-run-123/step-a-output/`).
4. Step A then returns this path as an output string/metadata.
5. The Kubeflow Pipeline orchestrator reads this metadata, updates the input parameters of Step B with the S3 URI, and launches Step B's pod.
6. Step B reads the URI parameter, downloads the data from S3, and starts its processing step.

### Q2: What is the benefit of the PyTorchJob Custom Resource in Kubeflow compared to deploying standard Kubernetes Pods?
**Answer**:
1. Deploying multi-node distributed training manually using standard Kubernetes Pods requires writing complex configuration code:
   - Defining a coordinator pod, creating service endpoints to expose it, and spinning up multiple worker pods.
   - Manually calculating and injecting IP addresses and rank configurations (`RANK`, `WORLD_SIZE`) into each worker pod's environment.
2. **PyTorchJob Operator**:
   - Automates this entire setup process.
   - The developer writes a single `PyTorchJob` YAML spec defining the replicas (e.g., 1 Master, 4 Workers).
   - The operator dynamically configures service names, mounts physical GPUs to correct worker containers, handles mutual authentication, injects system variables (`MASTER_ADDR`, `MASTER_PORT`, `RANK`), and cleans up all pod resources once training succeeds.

---

## References

1. **Kubeflow Platform Overview**: *Kubeflow: The Machine Learning Toolkit for Kubernetes*. (Kubeflow Open Source Project Documentation).
2. **Hyperparameter Tuning (Katib)**: *Katib: Scalable and Cloud-Native Hyperparameter Tuning System*. (KubeCon & CloudNativeCon).
3. **Argo Workflows Engine**: *Argo Workflows: Kubernetes-Native Workflow Engine*. (Argoproj Documentation).
