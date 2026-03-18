# Lab 02 — Cluster Creation with eksctl & AWS CLI 🟢

## Goal
Create a production-ready EKS cluster with a managed node group using both `eksctl` (declarative YAML) and raw AWS CLI commands. Configure `kubectl`, verify cluster health, and understand the CloudFormation stacks eksctl creates behind the scenes.

---

## Concepts

### eksctl vs AWS CLI vs Console

| Method | Pros | Cons |
|--------|------|------|
| **eksctl** | Single command, manages CloudFormation stacks, idempotent | Abstraction hides details |
| **AWS CLI** | Full control over every API call | Verbose; must manage VPC/node group manually |
| **Console** | Visual, guided wizard | Not repeatable, not IaC |
| **Terraform** | Full IaC, state management | Requires EKS Terraform module knowledge |

### Cluster Networking — Key Decisions

| Decision | Recommended Setting | Why |
|----------|--------------------|----|
| VPC CIDR | /16 (e.g. 10.0.0.0/16) | Enough IPs for pods (VPC CNI) |
| Private subnets | Yes (for nodes) | Nodes not directly exposed to internet |
| Public subnets | Yes (for load balancers) | ALB/NLB need public subnets |
| API endpoint | Public + Private | Allows kubectl from internet; internal traffic stays private |
| Node group subnet | Private | Worker nodes in private subnets |

### Managed Node Group AMI Types

| AMI Type | Description |
|----------|-------------|
| `AL2_x86_64` | Amazon Linux 2 (default, most common) |
| `AL2_ARM_64` | Amazon Linux 2 for Graviton (arm64) |
| `BOTTLEROCKET_x86_64` | AWS Bottlerocket OS — minimal, container-focused |
| `WINDOWS_CORE_2022_x86_64` | Windows Server 2022 Core |

---

## Architecture

```
eksctl create cluster
        │
        ▼
CloudFormation Stack: eksctl-my-eks-cluster-cluster
  ├── VPC (10.0.0.0/16)
  │   ├── Public Subnet AZ-a   (10.0.0.0/19)
  │   ├── Public Subnet AZ-b   (10.0.32.0/19)
  │   ├── Private Subnet AZ-a  (10.0.96.0/19)
  │   └── Private Subnet AZ-b  (10.0.128.0/19)
  ├── Internet Gateway
  ├── NAT Gateways (one per AZ)
  └── EKS Cluster (Control Plane)

CloudFormation Stack: eksctl-my-eks-cluster-nodegroup-ng-workers
  ├── IAM Role for Nodes (AmazonEKSWorkerNodePolicy, etc.)
  ├── Launch Template (AMI, instance type, user-data)
  └── Auto Scaling Group (2–4 nodes across private subnets)
```

---

## PoC Steps

### Step 1 — Create the eksctl cluster config file

```bash
cat > /tmp/eks-cluster.yaml <<'EOF'
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-eks-cluster
  region: us-east-1
  version: "1.31"
  tags:
    Project: eks-how-to-guide
    Owner: naveen
    Environment: lab

managedNodeGroups:
  - name: ng-workers
    instanceType: t3.medium
    minSize: 1
    maxSize: 4
    desiredCapacity: 2
    privateNetworking: true          # nodes in private subnets
    amiFamily: AmazonLinux2
    iam:
      withAddonPolicies:
        autoScaler: true             # needed for Cluster Autoscaler (Lab 06)
        albIngress: true             # needed for AWS Load Balancer Controller (Lab 04)
        cloudWatch: true             # needed for Container Insights (Lab 07)
        ebs: true                    # needed for EBS CSI driver (Lab 06)
    labels:
      role: worker
    tags:
      Project: eks-how-to-guide
      k8s.io/cluster-autoscaler/enabled: "true"
      k8s.io/cluster-autoscaler/my-eks-cluster: "owned"

cloudWatch:
  clusterLogging:
    enableTypes:
      - api
      - audit
      - authenticator
      - controllerManager
      - scheduler
EOF

echo "Config file created at /tmp/eks-cluster.yaml"
cat /tmp/eks-cluster.yaml
```

### Step 2 — Create the EKS cluster

```bash
eksctl create cluster -f /tmp/eks-cluster.yaml
```

This takes **15–20 minutes**. Expected final output:
```
[✓]  EKS cluster "my-eks-cluster" in "us-east-1" region is ready
```

### Step 3 — Update kubeconfig

```bash
aws eks update-kubeconfig \
  --name my-eks-cluster \
  --region us-east-1

# Verify context was added
kubectl config current-context
# Expected: arn:aws:eks:us-east-1:123456789012:cluster/my-eks-cluster
```

### Step 4 — Verify cluster and nodes

```bash
# Cluster info
kubectl cluster-info
# Expected:
# Kubernetes control plane is running at https://XXXX.gr7.us-east-1.eks.amazonaws.com
# CoreDNS is running at https://XXXX.gr7.us-east-1.eks.amazonaws.com/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

# Node status
kubectl get nodes -o wide
# NAME                          STATUS   ROLES    AGE   VERSION    INTERNAL-IP    EXTERNAL-IP   OS-IMAGE
# ip-10-0-96-12.ec2.internal    Ready    <none>   5m    v1.31.x    10.0.96.12     <none>        Amazon Linux 2
# ip-10-0-128-45.ec2.internal   Ready    <none>   5m    v1.31.x    10.0.128.45    <none>        Amazon Linux 2

# All system pods
kubectl get pods -n kube-system
# NAME                       READY   STATUS    RESTARTS   AGE
# aws-node-xxxxx             1/1     Running   0          5m    <- VPC CNI
# coredns-xxxxxxxx           1/1     Running   0          5m    <- DNS
# kube-proxy-xxxxx           1/1     Running   0          5m    <- iptables rules
```

### Step 5 — Inspect the CloudFormation stacks eksctl created

```bash
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --query 'StackSummaries[?starts_with(StackName, `eksctl-my-eks-cluster`)].{Name:StackName, Status:StackStatus, Created:CreationTime}' \
  --output table
# ┌─────────────────────────────────────────────────────────────────────────┐
# │  eksctl-my-eks-cluster-cluster                         │ CREATE_COMPLETE │
# │  eksctl-my-eks-cluster-nodegroup-ng-workers            │ CREATE_COMPLETE │
# └─────────────────────────────────────────────────────────────────────────┘
```

### Step 6 — Check cluster details via AWS CLI

```bash
aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query 'cluster.{Name:name, Version:version, Endpoint:endpoint, Status:status, PlatformVersion:platformVersion}' \
  --output table
```

Expected output:
```
----------------------------------------------------------------------
|                        DescribeCluster                              |
+-------------------+-------------------------------------------------+
|  Name             |  my-eks-cluster                                 |
|  Version          |  1.31                                           |
|  Status           |  ACTIVE                                         |
|  PlatformVersion  |  eks.x                                          |
|  Endpoint         |  https://XXXX.gr7.us-east-1.eks.amazonaws.com  |
+-------------------+-------------------------------------------------+
```

### Step 7 — List managed node groups

```bash
eksctl get nodegroup \
  --cluster my-eks-cluster \
  --region us-east-1 \
  --output table
# ┌──────────────────────┬────────┬──────┬────────┬──────┬──────────────────┐
# │ CLUSTER              │ NAME   │ MIN  │ MAX    │ DESI │  STATUS          │
# ├──────────────────────┼────────┼──────┼────────┼──────┼──────────────────┤
# │ my-eks-cluster       │ng-workers│ 1  │  4     │  2   │  CREATE_COMPLETE │
# └──────────────────────┴────────┴──────┴────────┴──────┴──────────────────┘
```

### Step 8 — Check EKS add-ons installed

```bash
aws eks list-addons \
  --cluster-name my-eks-cluster \
  --region us-east-1 \
  --query 'addons' \
  --output table
# ┌─────────────────────────────────────────────────────────────────────────┐
# │                            ListAddons                                   │
# ├─────────────────────────────────────────────────────────────────────────┤
# │  coredns                                                                │
# │  kube-proxy                                                             │
# │  vpc-cni                                                                │
# └─────────────────────────────────────────────────────────────────────────┘
```

---

## Cleanup

> **Keep the cluster running** — all subsequent labs (03–07) depend on it. Run cleanup only when you have completed all labs.

```bash
# Full cluster + VPC teardown (takes ~10 minutes)
eksctl delete cluster --name my-eks-cluster --region us-east-1

# Verify deletion
aws eks describe-cluster --name my-eks-cluster --region us-east-1 2>&1 | grep -i "not found"
# ResourceNotFoundException: No cluster found for name: my-eks-cluster
```

---

## Next Lab
➡️ [Lab 03 — Deploying Workloads](../03-deploy-workloads/README.md)
