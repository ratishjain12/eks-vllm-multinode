---
name: eks-vllm-setup
description: Guide and patterns for setting up Amazon EKS with GPU node groups, EFA support, Cluster Autoscaler, and LeaderWorkerSet (LWS) for vLLM inference.
---

# EKS vLLM Multi-Node Inference Setup

This skill provides the necessary patterns and instructions to deploy a high-performance LLM inference cluster on Amazon EKS.

## Core Architecture

1.  **VPC**: 3 AZs, private/public subnets, single NAT gateway.
2.  **EKS Cluster**: v1.32, with EKS Pod Identity agent.
3.  **GPU Node Group**: `g6e.8xlarge` (L40S GPUs) with:
    *   **EFA** enabled for high-speed node-to-node communication.
    *   **RAID-0 NVMe** ephemeral storage for fast model loading/weights.
    *   **NVIDIA/EFA device plugins** via Helm.
4.  **Scaling**: Cluster Autoscaler with EKS Pod Identity (IAM role `autoscaling:*`).
5.  **Inference**: LeaderWorkerSet (LWS) for multi-node Pipeline Parallelism (PP) or Tensor Parallelism (TP).

## Setup Steps

### 1. Networking (VPC)
Use the `terraform-aws-modules/vpc/aws` module. Tag private subnets with `kubernetes.io/role/internal-elb = 1`.

### 2. GPU Node Configuration
Ensure the `ami_type` is `AL2023_x86_64_NVIDIA` and `enable_efa_support = true`. Apply a `nvidia.com/gpu: NoSchedule` taint to isolate GPU workloads.

### 3. Scaling Infrastructure
Deploy the **Cluster Autoscaler** via Helm. 
*   **IAM Role**: Use EKS Pod Identity for the service account `cluster-autoscaler-aws-cluster-autoscaler`.
*   **Discovery**: Set `autoDiscovery.clusterName` to your EKS cluster name.

### 4. Weights & Storage
Mount instance store volumes in RAID-0 using `cloudinit_pre_nodeadm` to provide high-speed ephemeral storage (80Gi-160Gi) for vLLM.

### 5. vLLM Configuration (Llama-3.1/3.3)
Deploy via `LeaderWorkerSet` (LWS).
*   **Size**: Set based on Model size vs. GPU memory (48GB per L40S).
    *   Llama-3.1-8B: Size=2 (PP=2, TP=1).
    *   Llama-3.3-70B: Size=4 (PP=4, TP=1).
*   **Command**: Use `vllm.entrypoints.openai.api_server` with `--pipeline-parallel-size`.

## Reusable Snippets

### Pod Identity Association
```hcl
resource "aws_eks_pod_identity_association" "this" {
  cluster_name    = module.eks.cluster_name
  namespace       = "kube-system"
  service_account = "SERVICE_ACCOUNT_NAME"
  role_arn        = aws_iam_role.this.arn
}
```

### vLLM Command (PP=2)
```bash
python3 -m vllm.entrypoints.openai.api_server --port 8080 --model <MODEL_ID> --tensor-parallel-size 1 --pipeline-parallel-size 2
```
