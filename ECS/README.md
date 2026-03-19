# 🐳 Amazon ECS — Elastic Container Service

Run Docker containers on AWS without managing servers (Fargate) or on your own EC2 fleet — at any scale.

---

## Learning Path

| Difficulty | Labs |
|------------|------|
| 🟢 Beginner | Lab 01 · Lab 02 |
| 🟡 Intermediate | Lab 03 · Lab 04 |
| 🔴 Advanced | Lab 05 · Lab 06 · Lab 07 |

---

## Labs Index

| Lab | Title | Difficulty |
|-----|-------|------------|
| [01](./01-ecs-overview/README.md) | ECS Concepts & Architecture | 🟢 |
| [02](./02-first-cluster-and-task/README.md) | First Cluster, Task Definition & Run Task | 🟢 |
| [03](./03-fargate-service/README.md) | Fargate Service with ALB | 🟡 |
| [04](./04-advanced-deployments/README.md) | Rolling Updates, Blue/Green & Auto Scaling | 🟡 |
| [05](./05-iam-and-security/README.md) | IAM Roles, Secrets & VPC Isolation | 🔴 |
| [06](./06-integration-patterns/README.md) | CI/CD Pipeline: ECR → ECS Blue/Green via CodeDeploy | 🔴 |
| [07](./07-production-patterns/README.md) | Observability, Cost Optimisation & Resilience | 🔴 |

---

## Core Concepts Quick-Reference

| Concept | Description |
|---------|-------------|
| **Cluster** | Logical grouping of tasks/services (Fargate or EC2 launch type) |
| **Task Definition** | Blueprint: Docker image, CPU/memory, env vars, volumes, IAM role |
| **Task** | Running instance of a task definition (ephemeral) |
| **Service** | Long-running controller — maintains desired task count, integrates with ALB |
| **Launch Type** | **Fargate** = serverless; **EC2** = self-managed nodes |
| **Container Agent** | Daemon on EC2 nodes that communicates with ECS control plane |
| **Task Role** | IAM role assumed by the containers inside a task |
| **Execution Role** | IAM role used by ECS to pull images and write logs |
| **Capacity Provider** | Abstraction over launch types; supports FARGATE_SPOT for cost savings |
| **Service Discovery** | AWS Cloud Map integration for service-to-service DNS resolution |

---

## ASCII Architecture Diagram

```
  Developer                AWS Cloud
  ─────────                ──────────────────────────────────────────────
  docker push ──► ECR (image registry)
                                │
                                ▼
                     ECS Cluster (Fargate)
                     ┌──────────────────────────────────┐
                     │  Service  (desired count = 3)     │
                     │  ┌──────┐ ┌──────┐ ┌──────┐     │
                     │  │Task 1│ │Task 2│ │Task 3│     │
                     │  │:8080 │ │:8080 │ │:8080 │     │
                     │  └──┬───┘ └──┬───┘ └──┬───┘     │
                     └─────┼────────┼────────┼──────────┘
                           │        │        │
                     ┌─────▼────────▼────────▼──────────┐
                     │       Application Load Balancer    │
                     │         Target Group (IP mode)     │
                     └──────────────┬───────────────────┘
                                    │
                             Internet / VPC
```

---

## Prerequisites & Setup

```bash
# Install / update AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install --update

# Verify
aws --version        # aws-cli/2.x.x

# Set default region
export AWS_DEFAULT_REGION=us-east-1
aws configure set region us-east-1

# Install ECS CLI helper (ecs-cli) — optional but useful
sudo curl -Lo /usr/local/bin/ecs-cli \
  https://amazon-ecs-cli.s3.amazonaws.com/ecs-cli-linux-amd64-latest
sudo chmod +x /usr/local/bin/ecs-cli

# Verify account
aws sts get-caller-identity --output table
```

---

## CLI Cheat Sheet

```bash
# --- Clusters ---
aws ecs list-clusters --output table
aws ecs describe-clusters --clusters my-cluster --output table

# --- Task Definitions ---
aws ecs list-task-definitions --output table
aws ecs describe-task-definition --task-definition my-task:1 --output table

# --- Run a one-off task ---
aws ecs run-task \
  --cluster my-cluster \
  --launch-type FARGATE \
  --task-definition my-task:1 \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc],securityGroups=[sg-abc],assignPublicIp=ENABLED}"

# --- Services ---
aws ecs list-services --cluster my-cluster --output table
aws ecs describe-services --cluster my-cluster --services my-svc --output table
aws ecs update-service --cluster my-cluster --service my-svc --desired-count 3

# --- Exec into a running container ---
aws ecs execute-command \
  --cluster my-cluster \
  --task <task-id> \
  --container app \
  --interactive \
  --command "/bin/sh"

# --- View logs ---
aws logs get-log-events \
  --log-group /ecs/my-task \
  --log-stream-name ecs/app/<task-id> \
  --output table
```

---

## Next Step

Start with [Lab 01 → ECS Concepts & Architecture](./01-ecs-overview/README.md)
