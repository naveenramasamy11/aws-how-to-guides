# EC2 — Amazon Elastic Compute Cloud

> **Module goal:** Go from zero to production-ready EC2 deployments.
> Seven hands-on labs take you from launching your first instance through building a fully monitored, auto-scaled, cost-optimised compute tier.

---

## Learning Path

```
🟢 Lab 01 → Service overview, instance types, pricing
🟢 Lab 02 → Launch first instance, SSH, user-data bootstrap
🟡 Lab 03 → EBS volumes, instance store, snapshots
🟡 Lab 04 → AMIs, Auto Scaling Groups, Launch Templates
🔴 Lab 05 → IAM roles, Security Groups, key pairs, SSM Session Manager
🔴 Lab 06 → ALB + ASG + SNS pipeline (event-driven scale)
🔴 Lab 07 → CloudWatch dashboards, Cost Explorer, Savings Plans, resilience
```

---

## Labs Index

| # | Lab | Difficulty |
|---|-----|-----------|
| 01 | [Service Overview — Concepts, Instance Types & Pricing](./01-overview/README.md) | 🟢 Beginner |
| 02 | [Launch Your First Instance — CLI, SSH & User-Data](./02-launch-first-instance/README.md) | 🟢 Beginner |
| 03 | [Storage Deep-Dive — EBS, Instance Store & Snapshots](./03-storage-and-ebs/README.md) | 🟡 Intermediate |
| 04 | [AMIs, Auto Scaling Groups & Launch Templates](./04-ami-snapshots-autoscaling/README.md) | 🟡 Intermediate |
| 05 | [IAM Roles, Security Groups & SSM Session Manager](./05-iam-security/README.md) | 🔴 Advanced |
| 06 | [Integration Pipeline — ALB → ASG → Lambda → SNS](./06-integration-pipeline/README.md) | 🔴 Advanced |
| 07 | [Production Patterns — Monitoring, Cost & Resilience](./07-production-patterns/README.md) | 🔴 Advanced |

---

## Core Concepts Quick-Reference

| Concept | Description |
|---------|-------------|
| **Instance Type** | vCPU + memory + network profile (e.g. `t3.micro`, `m6i.xlarge`, `c7g.2xlarge`) |
| **AMI** | Amazon Machine Image — OS + software snapshot used to launch instances |
| **Key Pair** | RSA/ED25519 keypair for SSH; store private key securely |
| **Security Group** | Stateful virtual firewall — rules evaluated per packet direction |
| **EBS** | Elastic Block Store — persistent network-attached volumes |
| **Instance Store** | Ephemeral NVMe storage local to the host — lost on stop/terminate |
| **Placement Group** | Cluster / Partition / Spread — control physical placement of instances |
| **User Data** | Cloud-init script executed once at first boot |
| **Instance Metadata** | Available at `http://169.254.169.254/latest/meta-data/` (IMDSv2 recommended) |
| **Spot Instance** | Up to 90 % savings; can be interrupted with 2-min notice |
| **Savings Plan** | 1- or 3-year commitment for On-Demand discount |

---

## ASCII Architecture Diagram

```
Internet
    │
    ▼
┌───────────────────────────────────────────────────────┐
│  VPC  10.0.0.0/16                                     │
│                                                       │
│  ┌──────────────────────────────────────────────┐    │
│  │  Public Subnet  10.0.1.0/24                  │    │
│  │                                              │    │
│  │  ┌─────────────────┐                        │    │
│  │  │  ALB (Internet  │                        │    │
│  │  │  facing)        │                        │    │
│  │  └────────┬────────┘                        │    │
│  └───────────┼──────────────────────────────────┘   │
│              │                                       │
│  ┌───────────▼──────────────────────────────────┐   │
│  │  Private Subnet  10.0.2.0/24                 │   │
│  │                                              │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │   │
│  │  │ EC2 (AZ-a│  │ EC2 (AZ-b│  │ EC2 (AZ-c│   │   │
│  │  │  ASG)    │  │  ASG)    │  │  ASG)    │   │   │
│  │  └────┬─────┘  └──────────┘  └──────────┘   │   │
│  └───────┼──────────────────────────────────────┘   │
│          │                                           │
│  ┌───────▼──────────────────────────────────────┐   │
│  │  Data Subnet  10.0.3.0/24                    │   │
│  │  RDS / ElastiCache                           │   │
│  └──────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────┘
```

---

## One-Time Setup

```bash
# Set your default region
export AWS_DEFAULT_REGION=us-east-1

# Confirm identity
aws sts get-caller-identity --output table

# Create a key pair for the labs (save the .pem!)
aws ec2 create-key-pair \
  --key-name ec2-labs-key \
  --query 'KeyMaterial' \
  --output text > ~/ec2-labs-key.pem

chmod 400 ~/ec2-labs-key.pem
```

---

## CLI Cheat Sheet

```bash
# List running instances
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# Start / Stop / Terminate
aws ec2 start-instances  --instance-ids i-0123456789abcdef0
aws ec2 stop-instances   --instance-ids i-0123456789abcdef0
aws ec2 terminate-instances --instance-ids i-0123456789abcdef0

# Describe Security Groups
aws ec2 describe-security-groups \
  --query 'SecurityGroups[*].[GroupId,GroupName,Description]' \
  --output table

# Get latest Amazon Linux 2023 AMI
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
             "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text

# Describe EBS volumes
aws ec2 describe-volumes \
  --query 'Volumes[*].[VolumeId,Size,VolumeType,State,AvailabilityZone]' \
  --output table
```

---

➡️ Start with [Lab 01 — Service Overview](./01-overview/README.md)
