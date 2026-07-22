# GCP: Google Cloud Machine Learning Infrastructure

## Overview

**Google Cloud Platform (GCP)** provides a highly specialized ecosystem for machine learning, driven by custom-designed hardware accelerators and enterprise container orchestration. GCP's ML stack centers around two primary platforms: **Vertex AI** (for managed model development and pipelines) and **Google Kubernetes Engine (GKE)** (for cloud-native custom deployments). This guide details GCP's unique **Tensor Processing Units (TPUs)**, high-speed pod networks, and workload security models.

---

## Problem Statement

Running machine learning workloads on GCP requires optimizing resources and security interfaces:
1. **Inefficient Data Ingestion**: Accessing training datasets (e.g., millions of small image files) stored in Cloud Storage directly from training pods can cause I/O bottlenecks, leaving TPUs/GPUs underutilized.
2. **TPU Network Latency**: Scale-out TPU training requires specialized network switches. Standard networking configurations fail to leverage the physical torus topology of TPU pods, slowing down multi-node training.
3. **Complex IAM Secrets Management**: Passing static service account JSON key files to Kubernetes worker pods creates security vulnerabilities and maintenance overhead.

---

## GCP AI/ML Infrastructure Components

GCP provides an integrated suite of managed AutoML, container, and hardware acceleration services:

```
                          [GCP Console / gcloud CLI]
                                     │
         ┌───────────────────────────┴───────────────────────────┐
         ▼                                                       ▼
  [Managed AI Platform]                                 [Kubernetes Containers]
     (Vertex AI)                                             (Google GKE)
  ├─ Vertex AI Pipelines (Kubeflow)                       ├─ Dynamic Node Auto-provisioning
  └─ Vertex AI Feature Store                              └─ Workload Identity Federation
         │                                                       │
         └───────────────────────────┬───────────────────────────┘
                                     ▼
                      [Google Custom Hardware Layer]
                      ├─ Cloud TPU Pods (v4/v5e/v5p)
                      ├─ Google ICI Network (Optical Switches)
                      └─ Cloud Storage (GCS Fuse mounts)
```

### 1. Vertex AI (Managed Platform)
Vertex AI brings Google's ML tools under a single API:
- **Vertex AI Pipelines**: Serverless execution of Kubeflow or TFX pipelines.
- **Custom Training Jobs**: Provision worker pools of GPUs or TPUs to run python scripts, abstracting away underlying node infrastructure.
- **Vertex AI Feature Store**: A managed repository to store, serve, and share ML features with low-latency online serving (based on Bigtable) and batch offline capabilities.

---

### 2. GKE GPU & TPU Orchestration
Google Kubernetes Engine (GKE) is widely considered the most advanced managed Kubernetes engine, offering features tailored for ML:
- **TPU in GKE**: GKE natively schedules Cloud TPU slices using standard Kubernetes pod YAML definitions.
- **GCSFuse**: GKE supports mounting Cloud Storage buckets directly into containers as local folders. This enables training scripts to read training files with standard Python file I/O operations (`open()`), with caching and pre-fetching handled automatically by Google's storage driver.
- **Node Auto-Provisioning (NAP)**: Automatically spins up optimal GPU node groups based on pending pod requirements, matching driver versions with container needs.

---

### 3. Cloud TPUs (Tensor Processing Units)
TPUs are Google's custom Application-Specific Integrated Circuits (ASICs) designed from the ground up for deep learning:
- **TPU v4**: Features 275 TFLOPS per chip, connected in a 3D Torus topology.
- **TPU v5p**: Google's most powerful TPU, delivering 459 TFLOPS of BF16 performance and 95 GB of HBM3 memory per chip, designed for massive LLM training.
- **Interconnect (ICI)**: TPU chips inside a Pod are connected via **Inter-Chassis Interconnect (ICI)** copper/optical cables, bypassing traditional network protocols to execute high-speed matrix additions directly.
- **Optical Circuit Switches (OCS)**: Dynamically routes connections between TPU nodes using light beams, allowing the cluster topology to be reconfigured programmatically to isolate failed chips.

---

## Security: Workload Identity

GCP uses **Workload Identity** to securely bind Kubernetes service accounts to GCP service accounts:
- **Mechanism**:
  - A Google Service Account (GSA) is created with specific permissions (e.g., read access to a GCS bucket).
  - A Kubernetes Service Account (KSA) is annotated with the GSA's email address.
  - GKE's metadata server intercepts authentication calls from the pod. When the model serving code calls `storage.Client()`, the metadata server exchanges the Kubernetes token for a temporary GCP OAuth2 access token, eliminating the need to store JSON secret keys inside the container.

---

## Interview Questions

### Q1: What is Google's Workload Identity and why is it preferred over using static Service Account JSON keys?
**Answer**:
1. **Workload Identity** is the recommended method for applications running on GKE to access Google Cloud services (such as Cloud Storage, BigQuery, or Vertex AI) securely.
2. **Why it is preferred**:
   - **No Secret Management Overhead**: Static JSON keys do not expire automatically and must be rotated manually, creating a significant security risk if a key is committed to Git or leaked.
   - **Automated Token Management**: Workload Identity binds a Kubernetes Service Account (KSA) directly to a Google Service Account (GSA). GKE dynamically requests short-lived, self-rotating credentials (valid for 1 hour) from the GCP Security Token Service.
   - **Fine-Grained IAM Control**: Security teams can restrict access at the individual Pod level, rather than assigning broad VM-level IAM roles to the underlying GKE host nodes.

### Q2: How do Cloud TPUs differ from NVIDIA GPUs in architecture and workload suitabilities?
**Answer**:
1. **ASIC vs. General GPU**: NVIDIA GPUs are general-purpose processors designed to handle graphics, rendering, and scientific computing alongside deep learning. Google TPUs are Application-Specific Integrated Circuits (ASICs) designed *strictly* for tensor matrix operations.
2. **Matrix Multiply Unit (MXU)**: TPUs feature a hardware-level coprocessor called the Matrix Multiply Unit, which processes matrix multiplications in a **Systolic Array** structure. Data flows through a grid of ALU nodes continuously without needing to write intermediate results back to registers, making it highly efficient.
3. **Network Interconnect (ICI)**: TPUs are designed to operate as a single unified system (TPU Pods) connected via custom high-speed OCS optical links.
4. **Workload Suitability**:
   - **TPUs**: Ideal for massive, long-running transformer training and fine-tuning workloads that can leverage large TPU pod configurations.
   - **GPUs**: Better suited for model serving, image/vision models, and workloads that require dynamic operations or custom CUDA kernels that are not supported by the Google XLA compiler.

---

## References

1. **Vertex AI Pipelines**: *Vertex AI Pipelines Architecture and Orchestration*. (Google Cloud Documentation).
2. **GKE GCSFuse Mounts**: *Mount Cloud Storage buckets in GKE workloads*. (GKE Guides).
3. **TPU v5p Architecture**: *Cloud TPU v5p Overview and Performance Benchmarks*. (Google Cloud Technical Blog).
