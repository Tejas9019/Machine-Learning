# AWS: Machine Learning Infrastructure

## Overview

**Amazon Web Services (AWS)** offers a comprehensive suite of cloud infrastructure components designed to support large-scale machine learning systems. AWS provides two primary deployment paths for AI: fully managed services through **Amazon SageMaker** and container orchestration via **Amazon Elastic Kubernetes Service (EKS)**. This guide details the architectural building blocks of AWS ML, including EC2 GPU instance specifications, high-performance networking, and S3 storage optimizations.

---

## Problem Statement

Designing ML architectures on AWS without understanding resource and networking constraints leads to:
1. **Slow Distributed Training**: Large models (like Llama-3-70B) cannot fit in a single GPU node. Distributing training across nodes using standard TCP networking causes communication bottlenecks, severely limiting throughput.
2. **High Managed Service Costs**: Amazon SageMaker Endpoints provide a simple deployment path, but they charge a premium. Keeping endpoints running $24/7$ without proper scaling policies is expensive.
3. **IAM Credential Leakage**: Hardcoding AWS access keys inside model training containers to read/write to S3 is a major security risk.

---

## AWS AI/ML Infrastructure Components

AWS provides a layered stack of compute, orchestration, and storage services:

```
                          [AWS API / CLI / SDK]
                                     │
         ┌───────────────────────────┴───────────────────────────┐
         ▼                                                       ▼
[Managed Machine Learning]                              [Container Orchestration]
  (Amazon SageMaker)                                      (Amazon EKS / Karpenter)
  ├─ SageMaker Training Jobs                              ├─ EKS GPU Node Groups (NVIDIA AMI)
  └─ SageMaker Real-Time Endpoints                         └─ IRSA (IAM Roles for Service Accounts)
         │                                                       │
         └───────────────────────────┬───────────────────────────┘
                                     ▼
                      [AWS Virtual Hardware Layer]
                      ├─ EC2 GPU Instances (p4d/p5)
                      ├─ Elastic Fabric Adapter (EFA)
                      └─ S3 Object Storage (Express One Zone)
```

### 1. Amazon SageMaker (Managed Workloads)
SageMaker provides abstract serverless wrappers for ML:
- **Training Jobs**: Spin up ephemeral EC2 GPU instances, pull training code containers, mount S3 data volumes, run the job, save the checkpoints back to S3, and terminate the compute resources automatically, charging only for the training duration.
- **SageMaker Endpoints**: Host model containers behind a managed load balancer. Supports auto-scaling, shadow variants (split traffic for testing), and multi-model endpoints (MME) to pack multiple models onto a single instance.

---

### 2. Amazon EKS GPU Orchestration (Karpenter)
For custom platforms, EKS manages containerized workloads on Kubernetes:
- **NVIDIA AMI**: EKS worker nodes use the official NVIDIA Amazon Machine Image (AMI), pre-loaded with container runtimes, GPU drivers, and CUDA tools.
- **Karpenter**: Karpenter is a high-performance Kubernetes cluster autoscaler. It monitors unschedulable pods, evaluates their resource requirements (e.g., requesting a pod with 4 GPUs and 128 GB RAM), dynamically provisions the exact optimal EC2 instance (e.g., a `p4d.24xlarge`), and terminates the instance when the work is done, scaling much faster than standard Kubernetes cluster autoscalers.

---

### 3. EC2 GPU Instances & High-Performance Networking
- **p4d.24xlarge**: Includes **8x NVIDIA A100 (40GB/80GB) GPUs** connected via high-speed NVLink. It features 400 Gbps network bandwidth using Elastic Fabric Adapter (EFA).
- **p5.48xlarge**: Includes **8x NVIDIA H100 (80GB) GPUs** with 3200 Gbps of EFA network bandwidth, designed for massive LLM training runs.
- **EFA & GPUDirect RDMA**: Elastic Fabric Adapter (EFA) bypasses the host operating system's network stack, allowing GPUs on different servers to read and write directly to each other's memory (Remote Direct Memory Access, or RDMA). This eliminates CPU overhead during gradient synchronization, enabling linear scaling across hundreds of nodes.

---

### 4. S3 Storage Optimizations
- **S3 Express One Zone**: A high-performance, single-AZ storage class designed to deliver consistent single-digit millisecond latency. It is ideal for storing active training data and checkpoints, allowing GPU nodes to download weights at maximum speed.
- **VPC Gateway Endpoints**: Ensures traffic between EKS/SageMaker and S3 remains within the private network, avoiding NAT Gateway processing charges.

---

## Security: IAM Roles for Service Accounts (IRSA)

- Hardcoding AWS credentials inside containers is forbidden.
- EKS uses **IRSA (IAM Roles for Service Accounts)** to establish trust between the Kubernetes OpenID Connect (OIDC) provider and AWS IAM.
- **Mechanism**: A service account inside Kubernetes is annotated with an IAM Role ARN. When a pod runs, the EKS controller injects a temporary AWS STS token into the pod's container. The AWS SDK reads this token automatically to authenticate with S3, scoping permissions to the specific pod.

---

## Interview Questions

### Q1: What is EFA (Elastic Fabric Adapter) and why is it critical for distributed LLM training on AWS?
**Answer**:
1. **EFA** is a custom network interface developed by AWS that provides high-performance, low-latency, and high-throughput networking for distributed computing.
2. **Why it's critical**:
   - In distributed model training (e.g., training a 70B parameter model across multiple nodes), GPUs must continuously exchange weight gradients using algorithms like Ring-AllReduce.
   - Standard TCP/IP networking introduces significant OS kernel overhead and packet routing latency, causing GPUs to sit idle.
   - EFA supports **OS-bypass** and **GPUDirect RDMA (Remote Direct Memory Access)**. This allows the GPU on Host A to write directly into the VRAM of a GPU on Host B over the physical network interface, bypassing the host CPU and operating system kernel.
   - This reduces network latency to sub-microseconds and scales network bandwidth to 3200 Gbps (on p5 instances), preventing network transfer times from bottlenecking GPU compute cycles.

### Q2: How does IRSA (IAM Roles for Service Accounts) secure access to AWS resources from Kubernetes pods?
**Answer**:
1. Instead of using static AWS Access Keys, **IRSA** uses OpenID Connect (OIDC) federation.
2. The EKS cluster hosts an OIDC provider endpoint.
3. An IAM Role is created with a trust policy that trusts the EKS OIDC provider.
4. In Kubernetes, a ServiceAccount is created and annotated with the IAM Role's ARN:
   ```yaml
   metadata:
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::12345:role/ml-s3-access-role
   ```
5. When a pod is scheduled, the EKS Pod Identity Webhook dynamically injects:
   - An AWS security token file (`/var/run/secrets/eks.amazonaws.com/serviceaccount/token`).
   - Environment variables pointing to the token and the role ARN (`AWS_ROLE_ARN`, `AWS_WEB_IDENTITY_TOKEN_FILE`).
6. The AWS SDK inside the container automatically detects these variables, exchanges the OIDC token for temporary, short-lived AWS IAM credentials via AWS STS, securing all calls to S3/DynamoDB.

---

## References

1. **SageMaker Endpoints**: *Amazon SageMaker Developer Guide: Host Models for Real-Time Inference*. (AWS Documentation).
2. **Karpenter Autoscaler**: *Karpenter: Kubernetes-native autoscaler for AWS*. (Karpenter project documentation).
3. **GPUDirect RDMA over EFA**: *Elastic Fabric Adapter Architecture and Performance*. (AWS Whitepapers).
