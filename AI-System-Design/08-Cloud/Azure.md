# Azure: Microsoft Azure Machine Learning Infrastructure

## Overview

**Microsoft Azure** provides a robust, enterprise-grade cloud platform optimized for artificial intelligence and deep learning workloads. Azure’s AI offerings are divided between **Azure Machine Learning (Azure ML)**—a managed platform for workspace collaboration and deployment—and **Azure Kubernetes Service (AKS)**—for custom container orchestration. This guide details Azure's high-performance **ND-series GPU Virtual Machines**, InfiniBand clustering architectures, and enterprise security models.

---

## Problem Statement

Deploying AI systems on Microsoft Azure requires managing specific scale and security constraints:
1. **Network Sync Bottlenecks**: Distributed training across multi-node virtual machines can face high network latency if using standard TCP networks, degrading GPU scaling efficiency.
2. **Object Storage Bottlenecks**: Accessing training data stored in Azure Blob Storage from GPU VMs can introduce latency if access policies, private endpoints, or container mounts are sub-optimal.
3. **Identity Governance**: Managing static connection strings or Shared Access Signatures (SAS tokens) to access storage accounts from pods creates security compliance risks.

---

## Azure AI/ML Infrastructure Components

Azure provides a tightly integrated suite of managed developer tools, compute instances, and storage fabrics:

```
                         [Azure Portal / Entra ID]
                                     │
         ┌───────────────────────────┴───────────────────────────┐
         ▼                                                       ▼
 [Managed ML Workspaces]                                [Container Orchestration]
      (Azure ML)                                              (Azure AKS)
  ├─ Azure ML Compute Clusters                            ├─ AKS GPU Node Pools
  └─ Prompt Flow Orchestration                            └─ Entra ID Workload Identities
         │                                                       │
         └───────────────────────────┬───────────────────────────┘
                                     ▼
                      [Microsoft Virtual Hardware Layer]
                      ├─ ND H100 v5 Virtual Machines
                      ├─ Quantum-2 InfiniBand (NDv4+)
                      └─ Blob Storage (Premium Block Blob)
```

### 1. Azure Machine Learning (Azure ML)
Azure ML provides an end-to-end framework to build, train, and deploy models:
- **Azure ML Workspaces**: A collaborative workspace hosting data assets, model registries, running experiments, and serving endpoints.
- **Compute Clusters**: Managed clusters of GPU Virtual Machines that auto-scale based on training queue submissions and terminate automatically upon run completion.
- **Prompt Flow**: A visual and code-based orchestrator designed to chain LLMs, prompts, python code, and vector stores into production-ready RAG applications.

---

### 2. AKS GPU Orchestration & NVIDIA Operator
- **Azure Kubernetes Service (AKS)** supports deploying containerized ML serving models (e.g., Triton or vLLM).
- **NVIDIA GPU Operator**: AKS utilizes the GPU Operator to automatically install NVIDIA drivers, container runtimes, and monitoring utilities on GPU nodes, simplifying node lifecycle management.
- **Node Pools**: AKS allows segregating CPU system workloads from specialized GPU node pools, utilizing Kubernetes labels and taints to route model workloads.

---

### 3. ND-Series GPU VMs & InfiniBand Networking
- **ND H100 v5 VM Series**: Azure's flagship AI virtual machines:
  * Packed with **8x NVIDIA H100 (80GB) Tensor Core GPUs** connected via internal NVLink.
  * Features 400 Gbps **NVIDIA Quantum-2 InfiniBand** networking per VM, scaling to thousands of interconnected GPUs in a single cluster.
- **GPUDirect RDMA**: InfiniBand bypasses the hypervisor and OS kernel, enabling direct memory transfers between GPU memories on separate VMs, ensuring high scaling efficiency during distributed training.

---

### 4. Blob Storage & Access Optimization
- **Premium Block Blob Storage**: Low-latency, high-throughput SSD-backed object storage designed for high-concurrency training access.
- **Blobfuse2**: A virtual filesystem driver that mounts Azure Blob storage containers directly to AKS pod directories, enabling models to stream training files using standard POSIX read/write functions.

---

## Security: Microsoft Entra ID Workload ID

Azure uses **Microsoft Entra ID (formerly Azure AD) Workload Identities** to authenticate AKS pods:
- **Mechanism**:
  - An **User-Assigned Managed Identity** is created in Azure and granted specific role-based access control (RBAC) permissions (e.g., *Storage Blob Data Reader*).
  - A federated credential is created, establishing a trust relationship between the AKS cluster's OIDC issuer and the Managed Identity.
  - The Kubernetes pod runs using a ServiceAccount bound to this Managed Identity.
  - When the container makes SDK calls, the Azure Identity library retrieves a federated token from the AKS metadata endpoint and exchanges it for an Entra ID access token, eliminating the need to store hardcoded secret keys inside the code.

---

## Interview Questions

### Q1: What is InfiniBand, and how does it compare to standard Ethernet for distributed training on Azure?
**Answer**:
1. **InfiniBand** is a high-speed, low-latency communication link used in supercomputing clusters. Ethernet is a general-purpose networking protocol designed for reliable, packet-switched routing across internet and local networks.
2. **Key Differences**:
   - **Latency**: InfiniBand delivers sub-microsecond latency, whereas Ethernet latency ranges from tens of microseconds to milliseconds due to operating system kernel TCP/IP stack overhead.
   - **Throughput**: Azure's ND H100 v5 instances provide up to 3.2 Tbps of InfiniBand bandwidth.
   - **RDMA (Remote Direct Memory Access)**: InfiniBand natively supports GPUDirect RDMA. This allows the GPU memory on VM A to be read/written directly from VM B over the physical network interface, completely bypassing the host CPU and OS kernel. Ethernet requires packaging data into TCP packets, routing them through the CPU memory, and traversing the OS network layer, which creates bottlenecks during weight synchronization.

### Q2: How does AKS authenticate pods to read data from an Azure Storage Account using Entra ID Workload Identity?
**Answer**:
1. **Managed Identity Creation**: A User-Assigned Managed Identity is created in Azure and assigned the *Storage Blob Data Reader* role on the target storage account.
2. **Federated Credential**: A trust federation is established in Entra ID, linking the managed identity to the specific AKS cluster OIDC issuer, Namespace, and Kubernetes ServiceAccount name.
3. **Pod Configuration**: The Kubernetes ServiceAccount is annotated with the client ID of the Managed Identity:
   ```yaml
   metadata:
     annotations:
       azure.workload.identity/client-id: <MANAGED_IDENTITY_CLIENT_ID>
   ```
4. **Execution**: When the pod launches, the AKS Workload Identity webhook injects a signed token into the container. The Azure SDK (e.g., `azure-storage-blob`) detects the token, exchanges it with Entra ID for an access token, and reads the storage bucket securely without any static keys.

---

## References

1. **Azure ML Pipelines**: *Create and run machine learning pipelines in Azure ML*. (Microsoft Learn).
2. **AKS GPU Operator**: *Use NVIDIA GPU Operator on Azure Kubernetes Service*. (AKS Technical Documentation).
3. **Azure ND H100 v5 Series**: *Virtual Machine sizes for High Performance Computing and AI on Azure*. (Microsoft Learn).
