# Terraform: Infrastructure as Code (IaC) for AI

## Overview

**Terraform** is an open-source, declarative Infrastructure as Code (IaC) tool developed by HashiCorp. In MLOps and AI platform engineering, Terraform is used to provision, version, and manage cloud infrastructure—such as VPC networks, GPU node pools, storage buckets, and IAM roles—in a repeatable and auditable manner. By writing infrastructure as code, teams prevent "configuration drift" and can replicate staging environments to production with zero manual click-ops.

---

## Problem Statement

Provisioning AI infrastructure using cloud console click-ops or custom scripting shells introducing operational bottlenecks:
1. **Configuration Drift**: Manual changes made to cluster configurations (e.g., modifying security group ports or upgrading GPU driver nodes) are undocumented, making it impossible to audit or rebuild the cluster from scratch during an outage.
2. **Race Conditions in State**: If multiple engineers attempt to update cloud infrastructure configurations concurrently without a centralized state lock, resources can become corrupted or duplicated.
3. **IAM Permissions Leakage**: Manually attaching overly broad IAM policies (e.g., `AdministratorAccess`) to GPU compute nodes because "it was the only way to get S3 downloads working" violates security compliance audits.

---

## Terraform Core Mechanics for ML

Terraform manages cloud API resources by compiling declarative files (`.tf`) into a physical state tracking graph:

```
                  [Developer / CI/CD pipeline]
                               │
                               ▼
               [Terraform Code Configuration (.tf)]
                               │
                               ▼
               [Terraform Core Engine Execution]
             (Resolves dependencies & resource graphs)
                               │
         ┌─────────────────────┴─────────────────────┐
         ▼                                           ▼
[State Backend Store]                      [Target Cloud Providers]
(S3 / GCS bucket)                          (AWS / GCP / Azure APIs)
   │                                                 │
   ▼                                                 ▼
[State Lock (DynamoDB)]                    [Physical Infrastructure]
(Prevents race conditions)                 (GPU Clusters, VPCs, Buckets)
```

- **Declarative Approach**: Developers define the *desired end-state* of the infrastructure (e.g., "I want a GKE cluster with 8 H100 GPUs and a private network"). Terraform calculates the dependency graph and executes the API requests to transition the cloud state.
- **State File (`terraform.tfstate`)**: A JSON file that acts as the single source of truth, mapping logical Terraform resources to physical cloud IDs.
- **State Locking**: State files are stored in remote object storage (S3/GCS) with a centralized lock database (e.g., DynamoDB or Firestore). When an engineer runs `terraform apply`, Terraform locks the state, preventing other executions from running concurrently and avoiding race conditions.

---

## Terraform ML Code Blueprint (AWS Example)

The following Terraform blueprint provisions a secure private S3 bucket for model checkpoints, an IAM role with OIDC trust federation (IRSA), and a GPU-accelerated EKS node group:

```hcl
# 1. Secure Model Checkpoint S3 Bucket with Lifecycle Rules
resource "aws_s3_bucket" "model_checkpoints" {
  bucket        = "ml-production-model-checkpoints-12345"
  force_destroy = false
}

resource "aws_s3_bucket_lifecycle_configuration" "checkpoint_lifecycle" {
  bucket = aws_s3_bucket.model_checkpoints.id

  rule {
    id     = "archive_old_checkpoints"
    status = "Enabled"

    # Transition intermediate checkpoints to Glacier deep archive after 30 days
    transition {
      days          = 30
      storage_class = "GLACIER_IR"
    }

    # Delete checkpoints after 90 days to control storage costs
    expiration {
      days = 90
    }
  }
}

# 2. IAM Role for EKS Service Account (OIDC IRSA)
resource "aws_iam_role" "eks_ml_role" {
  name = "eks-ml-s3-reader-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-west-2.amazonaws.com/id/CLUSTER_OIDC_ID"
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "oidc.eks.us-west-2.amazonaws.com/id/CLUSTER_OIDC_ID:sub" = "system:serviceaccount:ml-workloads:ml-service-account"
          }
        }
      }
    ]
  })
}

# 3. GPU Node Group for Model Serving (Karpenter or EKS Node Group)
resource "aws_eks_node_group" "gpu_node_group" {
  cluster_name    = "ml-production-cluster"
  node_group_name = "eks-gpu-workers"
  node_role_arn   = "arn:aws:iam::123456789012:role/eks-node-role"
  subnet_ids      = ["subnet-private-1", "subnet-private-2"]

  # Spin up p4d instances hosting A100 GPUs
  instance_types = ["p4d.24xlarge"]

  scaling_config {
    desired_size = 2
    max_size     = 10
    min_size     = 0 # Karpenter can scale down to zero to save costs
  }

  labels = {
    "hardware-type" = "gpu"
    "gpu-model"     = "nvidia-a100"
  }

  # Add taint to prevent standard CPU applications from running here
  taint {
    key    = "sku"
    value  = "gpu"
    effect = "NO_SCHEDULE"
  }
}
```

---

## Design Decisions & Trade-offs

### Declarative IaC (Terraform) vs. Imperative CLI/SDK Scripts

- **Declarative IaC (Terraform)**:
  * *Pros*: Self-documenting, manages dependencies automatically (knows that the IAM Role must be created *before* the Node Group that uses it), supports atomic plans (visualizing changes using `terraform plan` before applying them).
  * *Cons*: Learning curve for HCL (HashiCorp Configuration Language), state files can get out of sync if manual changes are made in the cloud console.
- **Imperative Scripts (Boto3 / Bash)**:
  * *Pros*: Full programmatic logic (if/else loops), easy to write for quick one-off tasks.
  * *Cons*: Hard to manage resource updates (must write custom checks to see if a resource already exists to avoid duplicate errors), hard to clean up resources, lacks state dependency tracking.

---

## Interview Questions

### Q1: What is the role of the State File in Terraform? Why is State Locking critical when working in a DevOps/MLOps team?
**Answer**:
1. **State File (`terraform.tfstate`)**:
   - Acts as Terraform's private database.
   - Maps the logical resource names declared in `.tf` files to real, physical resource IDs in the cloud API (e.g., mapping `aws_s3_bucket.model_checkpoints` to `arn:aws:s3:::ml-production-model-checkpoints-12345`).
   - Tracks metadata and dependencies between resources.
2. **State Locking Importance**:
   - If two engineers run `terraform apply` concurrently, they will read the same initial state.
   - If Engineer A updates a security group and Engineer B deletes a subnet, their operations will conflict.
   - Without locking, the second write will overwrite the first, leaving resources corrupted or leaving orphaned, un-tracked resources running in the cloud (cost leakage).
   - State Locking (using DynamoDB on AWS or Firestore on GCP) ensures only one write operation is active at a time, rejecting concurrent executions and maintaining state integrity.

### Q2: Why do we apply taints to GPU node groups in Terraform? How does this impact pod deployment?
**Answer**:
1. **Why we apply taints**: GPU nodes are extremely expensive (e.g., a `p4d` instance costs over $32/hour). We must ensure that only workloads that require GPU acceleration run on these nodes.
2. If the node pool has no taints, Kubernetes can schedule CPU-based applications (like Nginx, Prometheus logging agents, or lightweight web microservices) on the GPU nodes, occupying memory and blocking subsequent ML training jobs.
3. **Impact on pod deployment**:
   - By applying a taint (e.g., `sku=gpu:NoSchedule`) to the node group in Terraform, the scheduler blocks all pods from running on these nodes by default.
   - To deploy a model serving engine on the GPU nodes, developers must add a matching **Toleration** in their pod's YAML configuration:
     ```yaml
     tolerations:
     - key: "sku"
       operator: "Equal"
       value: "gpu"
       effect: "NoSchedule"
     ```
   - This ensures resource segregation and cost control.

---

## References

1. **Terraform Core Architecture**: *Terraform Architecture & Dependency Graph Resolution*. (HashiCorp Developer Guides).
2. **IaC Best Practices**: *Managing AWS Infrastructure with Terraform: State Management*. (AWS Whitepapers).
3. **EKS Terraform Blueprinting**: *Deploying GPU-accelerated Kubernetes clusters on AWS EKS using Terraform*. (NVIDIA Deep Learning Portal).
