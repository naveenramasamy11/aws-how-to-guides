# AWS How-To Guides 🚀

> Hands-on, PoC-driven AWS guides for engineers — from beginner to advanced.
> Each module is structured to help you **understand the concept**, **run the PoC**, and **extend to real-world use cases**.

---

## 📚 Modules

### ☁️ Core Infrastructure

| Module | Topics Covered | Difficulty |
|--------|---------------|------------|
| [IAM](./IAM/README.md) | Users, Groups, Roles, Policies, Cross-Account, Permission Boundaries, ABAC | 🟢 Beginner → 🔴 Advanced |
| [VPC](./VPC/README.md) | Subnets, Route Tables, Security Groups, NACLs, NAT GW, Peering, Endpoints, Transit Gateway | 🟢 Beginner → 🔴 Advanced |
| [EC2](./EC2/README.md) | Instance types, Launch Templates, ASG, EBS, AMI baking, SSM, ALB pipeline, CloudWatch, cost optimisation | 🟢 Beginner → 🔴 Advanced |
| [S3](./S3/README.md) | Buckets, storage classes, lifecycle rules, encryption (SSE-S3/KMS/C), versioning, CRR, Object Lock, S3 Select, event-driven pipelines | 🟢 Beginner → 🔴 Advanced |
| [RDS](./RDS/README.md) | DB engines, Multi-AZ, Read Replicas, RDS Proxy, KMS encryption, IAM DB auth, backups/PITR, Blue/Green deployments, CloudWatch alarms, production runbooks | 🟢 Beginner → 🔴 Advanced |
| [EKS](./EKS/README.md) | Cluster creation, managed node groups, Deployments, Ingress/ALB, IRSA, RBAC, HPA, Cluster Autoscaler, EBS/EFS CSI, Container Insights, cluster upgrades | 🟢 Beginner → 🔴 Advanced |
| [ECS](./ECS/README.md) | Clusters, Task Definitions, Fargate services, ALB integration, rolling & Blue/Green deployments, auto scaling, IAM task roles, Secrets Manager, ECR CI/CD, Container Insights, FARGATE_SPOT cost optimisation | 🟢 Beginner → 🔴 Advanced |
| [Lambda](./Lambda/README.md) | Function lifecycle, cold/warm starts, S3/SQS/API GW/EventBridge triggers, Layers, env vars, concurrency controls, VPC placement, ETL pipeline pattern, structured logging, X-Ray tracing, arm64 cost optimisation | 🟢 Beginner → 🔴 Advanced |

> More modules coming soon: CloudFormation, Organizations, Bedrock, etc.

---

## 🧰 Prerequisites

Before starting any lab, ensure you have the following tools installed:

```bash
# AWS CLI v2
aws --version

# Optional but recommended
jq --version        # JSON parsing
terraform --version # For IaC labs
```

- An AWS account (Free Tier works for most labs)
- AWS CLI configured with admin or appropriate permissions
- Basic understanding of Linux CLI

---

## 🗂️ How Labs Are Structured

Each lab follows this format:

```
📁 module/
├── README.md          ← Module overview & index
└── XX-lab-name/
    ├── README.md      ← Lab goal, concepts, architecture
    ├── commands.sh    ← All CLI commands (copy-paste ready)
    └── policies/      ← JSON policy documents
```

---

## ⚡ Quick Start

```bash
git clone https://github.com/naveenramasamy11/aws-how-to-guides.git
cd aws-how-to-guides/IAM
```

Pick a lab, read the README, and follow along!

---

## 🤝 Contributing

Pull requests are welcome! Please open an issue first to discuss what you'd like to add or change.

---

## 📝 License

MIT License — see [LICENSE](./LICENSE) for details.
