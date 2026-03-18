# EKS — Amazon Elastic Kubernetes Service

> **7 hands-on PoC labs** taking you from zero to production-ready Kubernetes on AWS.

---

## Learning Path

```
🟢 Lab 01  Overview & Architecture      → Concepts, pricing, quota limits, API anatomy
🟢 Lab 02  Cluster Creation             → eksctl + AWS CLI, kubeconfig, managed node groups
🟡 Lab 03  Deploying Workloads          → Deployments, Services, ConfigMaps, Secrets, rollouts
🟡 Lab 04  Networking & Ingress         → VPC CNI, ALB Ingress Controller, Network Policies
🔴 Lab 05  IAM & Security               → IRSA, RBAC, OPA Gatekeeper, Secrets Manager CSI
🔴 Lab 06  Autoscaling & Storage        → HPA, Cluster Autoscaler, EBS CSI, EFS CSI
🔴 Lab 07  Production Patterns          → CloudWatch Container Insights, logging, cluster upgrades
```

---

## Labs Index

| # | Lab | Difficulty | Duration |
|---|-----|-----------|----------|
| 01 | [EKS Overview & Architecture](./01-overview/README.md) | 🟢 Beginner | 20 min |
| 02 | [Cluster Creation with eksctl & CLI](./02-cluster-creation/README.md) | 🟢 Beginner | 30 min |
| 03 | [Deploying Workloads](./03-deploy-workloads/README.md) | 🟡 Intermediate | 40 min |
| 04 | [Networking & Ingress](./04-networking-ingress/README.md) | 🟡 Intermediate | 45 min |
| 05 | [IAM & Security (IRSA, RBAC)](./05-iam-security/README.md) | 🔴 Advanced | 50 min |
| 06 | [Autoscaling & Storage](./06-autoscaling-storage/README.md) | 🔴 Advanced | 50 min |
| 07 | [Production Patterns](./07-production-patterns/README.md) | 🔴 Advanced | 60 min |

---

## Core Concepts Quick-Reference

| Concept | Description |
|---------|-------------|
| **Control Plane** | AWS-managed Kubernetes API server, etcd, scheduler — spans 3 AZs by default |
| **Node Group** | EC2 Auto Scaling Group running the `kubelet` + `kube-proxy` + VPC CNI |
| **Managed Node Group** | AWS manages node lifecycle (AMI updates, drain, terminate) |
| **Fargate Profile** | Serverless pod execution — no node management, per-pod isolation |
| **Pod** | Smallest deployable unit; one or more containers sharing network/storage |
| **Deployment** | Declarative desired-state for a replica set of pods |
| **Service** | Stable DNS + VIP in front of pods; types: ClusterIP, NodePort, LoadBalancer |
| **Ingress** | L7 HTTP routing rule; requires an Ingress Controller (e.g. AWS Load Balancer Controller) |
| **IRSA** | IAM Roles for Service Accounts — fine-grained AWS permissions per pod |
| **VPC CNI** | AWS VPC CNI plug-in — each pod gets a real VPC IP from your subnet |

---

## ASCII Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          AWS Region                                   │
│                                                                       │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                    EKS Control Plane (AWS Managed)            │    │
│  │   kube-apiserver  ┃  etcd  ┃  controller-manager  ┃  scheduler│   │
│  └──────────────────────────┬───────────────────────────────────┘    │
│                             │ kubectl / API calls                     │
│        ┌────────────────────┼────────────────────┐                   │
│        │   VPC              │                    │                   │
│        │  ┌─────────────────▼──────────────────┐ │                   │
│        │  │   Managed Node Group (ASG)          │ │                   │
│        │  │  ┌─────────────┐ ┌────────────────┐ │ │                   │
│        │  │  │  Worker     │ │  Worker        │ │ │                   │
│        │  │  │  Node (EC2) │ │  Node (EC2)    │ │ │                   │
│        │  │  │ ┌─────────┐ │ │ ┌────────────┐ │ │ │                   │
│        │  │  │ │ Pod(s)  │ │ │ │  Pod(s)    │ │ │ │                   │
│        │  │  │ └─────────┘ │ │ └────────────┘ │ │ │                   │
│        │  │  └─────────────┘ └────────────────┘ │ │                   │
│        │  └───────────────────────────────────── ┘ │                   │
│        │                                            │                   │
│        │  ┌──────────────────────────────────────┐ │                   │
│        │  │ ALB (AWS Load Balancer Controller)   │ │                   │
│        │  └──────────────────────────────────────┘ │                   │
│        └────────────────────────────────────────────┘                   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Setup — Prerequisites

```bash
# 1. Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install
aws --version   # aws-cli/2.x.x

# 2. Install eksctl
ARCH=amd64
PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
tar -xzf eksctl_${PLATFORM}.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# 3. Install kubectl
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.31.0/2024-09-12/bin/linux/amd64/kubectl
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client

# 4. Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# 5. Configure AWS credentials
aws configure
# AWS Access Key ID:     <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name:   us-east-1
# Default output format: json
```

---

## CLI Cheat Sheet

```bash
# ── Cluster operations ──────────────────────────────────────────────
eksctl create cluster -f cluster.yaml
eksctl get cluster --name my-eks-cluster --region us-east-1
eksctl delete cluster --name my-eks-cluster --region us-east-1

# ── Node groups ─────────────────────────────────────────────────────
eksctl create nodegroup --cluster my-eks-cluster --name ng-extra --instance-types t3.medium --nodes 2
eksctl get nodegroup --cluster my-eks-cluster
eksctl delete nodegroup --cluster my-eks-cluster --name ng-extra

# ── kubeconfig ──────────────────────────────────────────────────────
aws eks update-kubeconfig --name my-eks-cluster --region us-east-1
kubectl config current-context
kubectl config get-contexts

# ── Workloads ────────────────────────────────────────────────────────
kubectl get nodes -o wide
kubectl get pods -A
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --tail=50 -f
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# ── Resource management ──────────────────────────────────────────────
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl rollout status deployment/<name>
kubectl rollout undo deployment/<name>

# ── IRSA ─────────────────────────────────────────────────────────────
eksctl create iamserviceaccount \
  --name my-sa \
  --namespace default \
  --cluster my-eks-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# ── Scaling ──────────────────────────────────────────────────────────
kubectl scale deployment my-app --replicas=5
kubectl autoscale deployment my-app --cpu-percent=50 --min=2 --max=10
```
