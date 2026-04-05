---
name: eks-best-practices
description: Core EKS patterns and best practices for production-ready clusters using Terraform, Pod Identity, and modern add-ons.
---

# EKS Production-Ready Best Practices

This skill outlines the standard patterns for deploying a secure, scalable, and manageable Amazon EKS cluster (v1.32+) using `terraform-aws-modules/eks/aws`.

## 1. Network Foundation (VPC)
*   **Private Subnets**: Always place worker nodes in private subnets.
*   **NAT Gateway**: Use a Single NAT Gateway for cost-efficiency in dev/test, or Multi-NAT for production high availability.
*   **Subnet Tagging**:
    *   `kubernetes.io/role/internal-elb = 1` (Private subnets)
    *   `kubernetes.io/role/elb = 1` (Public subnets)
*   **VPC CNI**: Use the latest version and configure `before_compute = true` to ensure networking is ready when nodes join.

## 2. Cluster Configuration
*   **Version Management**: Use the most recent stable Kubernetes version (e.g., 1.32).
*   **Security**:
    *   `enable_cluster_creator_admin_permissions = true` for easy initial access.
    *   `cluster_endpoint_public_access = true` (restrict by CIDR in production).
*   **Storage**: Encrypt EKS volumes using AWS KMS.

## 3. Managed Node Groups
*   **AMI Type**: Use `AL2023_x86_64` for the latest security patches and performance (AL2023 is the modern standard).
*   **Taints & Labels**: Use taints to isolate specialized workloads (e.g., `nvidia.com/gpu: NoSchedule` for GPU nodes) and labels for node affinity.
*   **Instance Store**: For high-performance workloads, use NVMe instance store volumes in a RAID-0 configuration.

## 4. Modern IAM (Pod Identity)
Always prefer **EKS Pod Identity** over the older IRSA (IAM Roles for Service Accounts) where possible.
*   Install the `eks-pod-identity-agent` add-on.
*   Use `aws_eks_pod_identity_association` to map IAM roles to ServiceAccounts.
*   This removes the need for OIDC provider complexity and manual annotation of ServiceAccounts.

## 5. Essential Add-ons & Scaling
*   **Cluster Autoscaler**: Deploy via Helm to manage Auto Scaling Group (ASG) capacity.
*   **CoreDNS/Kube-Proxy**: Keep these updated via EKS Managed Add-ons.
*   **Load Balancer Controller**: Deploy the `aws-load-balancer-controller` for ALB/NLB integration.

## 6. Maintenance & Logs
*   **CloudWatch Logging**: Enable cluster control plane logs (API, Audit, Authenticator) for troubleshooting and compliance.
*   **Node Repair**: Enable `node_repair_config` in managed node groups to auto-replace unhealthy nodes.
