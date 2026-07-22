# Kubernetes for ML Infrastructure

## Overview

**Kubernetes (K8s)** is the industry-standard container orchestration platform used to deploy, scale, and manage machine learning workloads in production. When hosting ML models—particularly LLMs and deep learning networks—Kubernetes is configured to schedule GPU accelerators, manage multi-tenant node pools, autoscale pods based on inference queue traffic (using KEDA), and handle seamless rolling updates.

---

## Problem Statement

Standard Kubernetes cluster configurations are optimized for CPU-based web microservices, presenting challenges when applied to ML inference and training:
1. **GPU Allocation Blindness**: Kubernetes cannot natively schedule or isolate GPU resources. Without plugins, it treats host nodes as standard CPU/RAM resource blocks.
2. **Resource Starvation and Spikes**: Model loading is VRAM-intensive. Spinning up a new replica of a 14 GB model causes a transient load spike that can trigger resource limit crashes.
3. **Autoscaling Inefficiencies**: Standard autoscalers scale pods based on CPU or Memory usage. However, during ML inference, a GPU can show $100\%$ utilization while processing a single request, triggering unnecessary, expensive pod scaling.
4. **VRAM Load Delay (Cold Starts)**: During a rolling update, a model serving container starts running immediately, but loading the model file from S3 into VRAM can take several minutes. Routing traffic to the pod before VRAM loading finishes leads to HTTP 500 timeouts.

---

## Kubernetes ML Core Mechanics

To support high-performance ML workloads, Kubernetes clusters integrate specialized device controllers and scaling architectures:

```
                         [Client HTTP/gRPC Requests]
                                     │
                                     ▼
                        [Ingress Controller (K8s)]
                                     │
       ┌─────────────────────────────┴─────────────────────────────┐
       ▼                                                           ▼
 [KEDA Scheduler] (Queue-based)                            [Control Plane]
       │                                                           │
       ▼                                                           ▼
 [Inference Pods]                                       [NVIDIA Device Plugin]
 (Readiness Probes verify model loading)                 (Exposes Host GPU cores)
       │                                                           │
       └─────────────────────────────┬─────────────────────────────┘
                                     ▼
                        [Kubernetes Worker Nodes]
                        ├─ GPU Node Pool (A100/H100)
                        └─ CPU Node Pool (API/Ingress)
```

### 1. GPU Scheduling & NVIDIA Device Plugin
- Kubernetes uses the **NVIDIA Container Toolkit** and **NVIDIA Device Plugin** to discover and schedule GPU accelerators.
- **Resource Limits**: Pods request GPUs in their YAML configuration:
  ```yaml
  resources:
    limits:
      nvidia.com/gpu: 1
  ```
- **Fractional GPU Sharing (MIG)**: For smaller models, Multi-Instance GPU (MIG) partitions a single physical H100/A100 into up to 7 isolated hardware instances, allowing multiple pods to share one GPU card with strict memory and compute isolation.

---

### 2. Heterogeneous Node Pools
- Clusters are split into distinct **Node Pools**:
  * **System Node Pool**: Low-cost CPU-only instances (e.g., `m5.large`) that run Kubernetes system agents, logging, and lightweight API gateways.
  * **GPU Node Pool**: Specialized accelerated instances (e.g., `g5.4xlarge` with A10G, or `p4de.24xlarge` with A100s) reserved strictly for training or heavy model serving.
- **Taints & Tolerations**: Prevents non-ML pods from scheduling on expensive GPU nodes:
  - GPU nodes are tainted with: `sku=gpu:NoSchedule`.
  - ML pods must declare matching tolerations in their spec to be scheduled on those nodes.

---

### 3. Queue-Based Autoscaling (KEDA)
- **KEDA (Kubernetes Event-driven Autoscaling)** replaces standard CPU-based scaling.
- KEDA monitors external event sources (e.g., Apache Kafka lag or RabbitMQ queue length).
- **Scale-to-Zero**: If the inference request queue is empty, KEDA can scale GPU model serving pods down to $0$ to eliminate idle GPU costs. As soon as a request enters the queue, KEDA spins up a pod to handle it.

---

### 4. Safe Rolling Updates & Readiness Probes
To prevent routing user requests to a container that is still loading model weights into VRAM, Kubernetes uses **Readiness Probes**:
- **Execution**: The serving pod starts, but is marked as `Not Ready` in the load balancer.
- **Probe Check**: Kubernetes executes HTTP or command checks (e.g., hitting a `/health` or `/ready` endpoint exposed by Triton/vLLM) to verify the model is fully loaded into VRAM.
- **Traffic Routing**: Only when the readiness probe returns an HTTP 200 OK does Kubernetes add the pod to the service endpoint, enabling zero-downtime rolling updates.

---

## Design Decisions & Trade-offs

### Kubernetes vs. Serverless Container Platforms (e.g., AWS ECS, Google Cloud Run)

- **Kubernetes**:
  * *Pros*: Maximum control over scheduling, native GPU partitioning, multi-tenant network isolation, integrations with Kubeflow/KEDA, vendor-agnostic.
  * *Cons*: Extreme operational complexity, high maintenance cost, requires dedicated platform engineering teams.
- **Serverless Containers (ECS/Fargate)**:
  * *Pros*: Zero cluster maintenance, simple deployments, pay-per-use configurations.
  * *Cons*: Limited GPU control, lacks advanced scheduling capabilities, vendor lock-in.

---

## Interview Questions

### Q1: Why is CPU/Memory-based scaling (HPA) sub-optimal for ML inference? What is the alternative?
**Answer**:
1. **HPA Limitation**: The standard Horizontal Pod Autoscaler (HPA) monitors CPU and RAM usage. 
2. During ML inference (e.g., running llama-3-8B), the GPU handles the bulk of the computations, while the host CPU remains relatively idle. Consequently, CPU usage will not trigger scaling even when the server is overloaded.
3. Conversely, if a single request arrives, the GPU might spike to $100\%$ compute utilization to run tensor calculations. If scaling was based on GPU utilization, a single request would trigger the cluster to spin up duplicate, expensive GPU pods, leading to resource waste.
4. **Alternative**: **Queue-Based Scaling (KEDA)**. KEDA scales pods based on the number of pending requests in the queue (e.g., queue lag in Kafka). If there are 100 requests in the queue and each pod can process 10 requests concurrently, KEDA scales the deployment to exactly 10 pods, matching resource allocation directly to user demand.

### Q2: What is MIG (Multi-Instance GPU)? Why is it important in Kubernetes ML deployments?
**Answer**:
1. **Multi-Instance GPU (MIG)** is a hardware-level technology developed by NVIDIA (supported on Ampere and newer architectures) that allows partitioning a single physical GPU into multiple independent GPU instances.
2. **Importance in K8s**:
   - **Cost Efficiency**: Without MIG, requesting `nvidia.com/gpu: 1` allocates an entire physical GPU (e.g., 80 GB A100) to one pod. If that pod only runs a small embedding model consuming 4 GB of VRAM, the remaining 76 GB and compute cores are wasted.
   - **Hardware Isolation**: MIG partitions the physical GPU's memory and compute SMs at the hardware silicon level.
   - This allows Kubernetes to schedule up to 7 separate pods on a single physical A100 GPU card, with guaranteed performance isolation, preventing one pod's memory leak or CPU spike from crashing neighboring workloads.

---

## References

1. **NVIDIA Device Plugin for Kubernetes**: *NVIDIA Container Toolkit & Device Plugin Documentation*. (NVIDIA GitHub).
2. **KEDA Scaling Patterns**: *KEDA: Kubernetes Event-driven Autoscaling Core Concepts*. (KEDA project docs).
3. **MIG Architecture**: *NVIDIA Multi-Instance GPU User Guide*. (NVIDIA Developer Portal).
