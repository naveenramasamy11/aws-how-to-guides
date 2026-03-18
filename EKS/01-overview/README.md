# Lab 01 — EKS Overview & Architecture 🟢

## Goal
Understand what Amazon EKS is, how it differs from self-managed Kubernetes, navigate the EKS service in the AWS Console and CLI, and list the key concepts, pricing dimensions, and service quotas you'll encounter in every lab.

---

## Concepts

### What is EKS?
Amazon Elastic Kubernetes Service (EKS) is a fully managed Kubernetes control plane. AWS runs, scales, and upgrades the API server, etcd, and scheduler across three Availability Zones. You are responsible only for worker nodes and the workloads you deploy on them.

### EKS vs Self-Managed Kubernetes

| Dimension | Amazon EKS | Self-Managed K8s (e.g. kubeadm) |
|-----------|-----------|----------------------------------|
| Control plane ops | AWS-managed | Your team manages |
| etcd backups | Automatic | Manual |
| Control-plane HA | Multi-AZ by default | You design it |
| Kubernetes upgrades | 1-click (in-place) | Manual drain/upgrade |
| SLA | 99.95% API availability | None |
| Cost | $0.10/hr per cluster + nodes | Only EC2/infra cost |

### EKS Data Plane Options

| Option | Description | Best For |
|--------|-------------|----------|
| **Managed Node Groups** | AWS manages AMI, lifecycle, drain-on-update | Most workloads |
| **Self-Managed Node Groups** | You manage EC2 launch templates, AMI | Custom AMI requirements |
| **AWS Fargate** | Serverless per-pod; no nodes to manage | Batch / isolated workloads |

### Kubernetes Architecture on EKS

```
Developer  ──► kubectl ──► EKS API Server (AWS-managed, multi-AZ)
                                │
                     ┌──────────▼──────────┐
                     │     etcd (AWS)       │  ← Encrypted at rest (KMS)
                     └──────────┬──────────┘
                                │
              ┌─────────────────▼──────────────────┐
              │           Worker Nodes (EC2)         │
              │  kubelet ◄── kube-proxy ◄── VPC CNI │
              │  ┌──────────┐  ┌──────────┐          │
              │  │  Pod     │  │  Pod     │          │
              │  │ (IP from │  │ (IP from │          │
              │  │  subnet) │  │  subnet) │          │
              │  └──────────┘  └──────────┘          │
              └────────────────────────────────────── ┘
```

### Pricing Model

| Component | Price (us-east-1) |
|-----------|------------------|
| EKS cluster | $0.10 / hour (~$72/month) |
| Managed node group EC2 | Standard EC2 pricing for instance type |
| Fargate pod (vCPU) | $0.04048 / vCPU-hour |
| Fargate pod (GB RAM) | $0.004445 / GB-hour |
| EKS Anywhere | License-based (on-premises) |

### Key Service Quotas (us-east-1 defaults)

| Quota | Default | Adjustable |
|-------|---------|-----------|
| EKS clusters per region | 100 | Yes |
| Managed node groups per cluster | 30 | Yes |
| Nodes per managed node group | 450 | Yes |
| Fargate profiles per cluster | 10 | Yes |
| Maximum pods per node (VPC CNI) | Depends on instance type | Via prefix delegation |

### Supported Kubernetes Versions (as of 2025)
EKS supports 4 minor versions at any time. Check current versions:
```bash
aws eks describe-addon-versions --query 'addons[0].addonVersions[0].compatibilities[*].clusterVersion' \
  --output table
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     AWS Account                                       │
│                                                                       │
│  IAM User / Role ──► AWS CLI / Console / SDK                         │
│         │                                                             │
│         ▼                                                             │
│  EKS Control Plane (Multi-AZ, AWS-managed)                           │
│  ┌─────────────────────────────────────────┐                         │
│  │  API Server  │  Scheduler  │  Controller │                        │
│  │              │             │  Manager    │                        │
│  │         etcd (encrypted with KMS)        │                        │
│  └─────────────────────────────────────────┘                         │
│         │                                                             │
│  VPC ───┼──────────────────────────────────────────                  │
│         │                                                             │
│  ┌──────▼──────────────────────────────────┐                         │
│  │  Worker Nodes (Managed Node Group / EC2) │                        │
│  │  kubelet  kube-proxy  aws-vpc-cni        │                        │
│  └─────────────────────────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## PoC Steps

### Step 1 — Verify prerequisites are installed

```bash
aws --version
# Expected: aws-cli/2.x.x Python/3.x.x ...

eksctl version
# Expected: 0.x.x

kubectl version --client --output=yaml
# Expected: clientVersion: major: "1" minor: "31"...

helm version --short
# Expected: v3.x.x+...
```

### Step 2 — Verify AWS credentials and permissions

```bash
aws sts get-caller-identity --output table
# ┌──────────────────────────────────────────────────────────────────────┐
# │                       GetCallerIdentity                              │
# ├──────────────┬───────────────────────────────────────────────────────┤
# │  Account     │  123456789012                                         │
# │  Arn         │  arn:aws:iam::123456789012:user/naveen               │
# │  UserId      │  AIDA...                                              │
# └──────────────┴───────────────────────────────────────────────────────┘
```

### Step 3 — List EKS supported Kubernetes versions

```bash
aws eks describe-addon-versions \
  --region us-east-1 \
  --query 'addons[0].addonVersions[*].compatibilities[*].clusterVersion' \
  --output json | python3 -c "import sys,json; versions=list({v for sub in json.load(sys.stdin) for v in sub}); versions.sort(); [print(v) for v in versions]"
```

Expected output:
```
1.27
1.28
1.29
1.30
1.31
```

### Step 4 — List available EKS add-ons

```bash
aws eks describe-addon-versions \
  --region us-east-1 \
  --query 'addons[*].addonName' \
  --output table
```

Expected output (partial):
```
------------------------------
|   DescribeAddonVersions    |
+----------------------------+
|  aws-ebs-csi-driver        |
|  aws-efs-csi-driver        |
|  coredns                   |
|  kube-proxy                |
|  vpc-cni                   |
|  aws-load-balancer-controller |
|  ...                       |
+----------------------------+
```

### Step 5 — Check max pods per instance type

The VPC CNI limits pods per node based on the number of ENIs and IPs an instance type supports.

```bash
# Check limits for t3.medium (3 ENIs × 6 IPs = 17 pods max by default)
python3 - <<'EOF'
# ENI limits per instance type (from AWS documentation)
eni_limits = {
    "t3.micro":   {"max_enis": 2, "ips_per_eni": 2},
    "t3.small":   {"max_enis": 3, "ips_per_eni": 4},
    "t3.medium":  {"max_enis": 3, "ips_per_eni": 6},
    "t3.large":   {"max_enis": 3, "ips_per_eni": 12},
    "t3.xlarge":  {"max_enis": 4, "ips_per_eni": 15},
    "m5.large":   {"max_enis": 3, "ips_per_eni": 10},
    "m5.xlarge":  {"max_enis": 4, "ips_per_eni": 15},
    "m5.2xlarge": {"max_enis": 4, "ips_per_eni": 15},
    "c5.large":   {"max_enis": 3, "ips_per_eni": 10},
}

print(f"{'Instance Type':<15} {'Max ENIs':>8} {'IPs/ENI':>8} {'Max Pods':>10}")
print("-" * 45)
for itype, limits in eni_limits.items():
    # Formula: (max_enis * (ips_per_eni - 1)) + 2
    max_pods = (limits["max_enis"] * (limits["ips_per_eni"] - 1)) + 2
    print(f"{itype:<15} {limits['max_enis']:>8} {limits['ips_per_eni']:>8} {max_pods:>10}")
EOF
```

Expected output:
```
Instance Type   Max ENIs  IPs/ENI    Max Pods
---------------------------------------------
t3.micro               2        2           4
t3.small               3        4          11
t3.medium              3        6          17
t3.large               3       12          35
t3.xlarge              4       15          59
m5.large               3       10          29
m5.xlarge              4       15          59
m5.2xlarge             4       15          59
c5.large               3       10          29
```

### Step 6 — Review IAM permissions required for EKS

```bash
# List the permissions your user/role needs for EKS operations
cat <<'EOF'
Minimum IAM permissions for EKS cluster admin:
  eks:*                          - All EKS actions
  ec2:*                          - VPC, subnets, security groups, instances
  iam:CreateRole                 - For node group roles
  iam:AttachRolePolicy           - For node group roles
  iam:PassRole                   - Pass roles to EKS service
  iam:CreateInstanceProfile      - For EC2 node groups
  iam:AddRoleToInstanceProfile   - For EC2 node groups
  cloudformation:*               - eksctl uses CloudFormation stacks
  autoscaling:*                  - Managed node group scaling
EOF
```

---

## Cleanup
No AWS resources are created in this lab — it is entirely read-only and educational.

---

## Next Lab
➡️ [Lab 02 — Cluster Creation with eksctl & CLI](../02-cluster-creation/README.md)
