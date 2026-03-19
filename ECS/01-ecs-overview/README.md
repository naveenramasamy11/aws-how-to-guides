# Lab 01 — ECS Concepts & Architecture 🟢

## Goal

Build a solid mental model of ECS: how clusters, task definitions, services, launch types, and networking fit together before writing a single line of CLI.

---

## Concepts

### Launch Types Compared

| Feature | Fargate | EC2 |
|---------|---------|-----|
| Node management | None (AWS-managed) | You manage EC2 instances |
| Billing unit | vCPU + memory per second | EC2 instance hours |
| Startup time | ~30 s | Depends on AMI/agent |
| Spot savings | FARGATE_SPOT (~70% off) | EC2 Spot instances |
| Custom AMI | Not supported | Supported |
| GPU workloads | Not supported | Supported (p3/g4) |
| Best for | Most microservices, APIs | GPU, custom kernel, very high density |

### Networking Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| `awsvpc` | Each task gets its own ENI and private IP | **Recommended** — full VPC isolation |
| `bridge` | Container port mapped to host port (EC2 only) | Legacy |
| `host` | Container shares host network namespace (EC2 only) | High-throughput, latency-sensitive |
| `none` | No external connectivity | Batch / isolated jobs |

### Task Definition Anatomy

```json
{
  "family": "web-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn":      "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/web-app:latest",
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group":  "/ecs/web-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "environment": [
        {"name": "APP_ENV", "value": "production"}
      ]
    }
  ]
}
```

### ECS Control Plane Components

```
┌─────────────────────────────────────────────────────────┐
│                    ECS Control Plane                     │
│  ┌──────────────┐  ┌─────────────┐  ┌───────────────┐  │
│  │  Scheduler   │  │  API Server │  │ State Store   │  │
│  │  (placement) │  │  (AWS SDK)  │  │ (desired vs   │  │
│  └──────────────┘  └─────────────┘  │  running)     │  │
│                                      └───────────────┘  │
└────────────────────────┬────────────────────────────────┘
                         │ control channel (HTTPS)
              ┌──────────┴──────────┐
              │   Fargate Data Plane │   or   EC2 Container Agent
              │  (AWS-managed VMs)   │         (ecs-agent daemon)
              └──────────────────────┘
```

### Service Scheduler Strategies

| Strategy | Behaviour |
|----------|-----------|
| `REPLICA` | Maintain N running tasks across AZs (most common) |
| `DAEMON` | Run exactly one task per EC2 node (logging agents, monitoring) |

### Capacity Providers

| Provider | Description |
|----------|-------------|
| `FARGATE` | On-demand Fargate — guaranteed capacity |
| `FARGATE_SPOT` | Spot Fargate — up to 70% cheaper, can be interrupted |
| Custom (EC2 ASG) | EC2 Auto Scaling Group managed by ECS |

---

## Architecture: Request Flow

```
  User Request (HTTPS)
        │
        ▼
  Route 53 (DNS)
        │
        ▼
  Application Load Balancer
  ┌─────────────────────┐
  │  Listener :443       │
  │  Target Group (IP)   │
  └──────┬──────────────┘
         │ health-checked
  ┌──────▼──────────────────────────────┐
  │  ECS Cluster (Fargate)               │
  │  VPC: 10.0.0.0/16                    │
  │  ┌────────────────┐ ┌──────────────┐ │
  │  │ Task (AZ-a)    │ │ Task (AZ-b)  │ │
  │  │ 10.0.1.12:8080 │ │10.0.2.15:8080│ │
  │  └────────────────┘ └──────────────┘ │
  └──────────────────────────────────────┘
         │ IAM Task Role
         ▼
  AWS Services (S3, DynamoDB, SQS…)
```

---

## Key Pricing Factors

| Resource | Fargate Price (us-east-1) |
|----------|--------------------------|
| vCPU per second | $0.04048 / vCPU-hour |
| Memory per second | $0.004445 / GB-hour |
| FARGATE_SPOT vCPU | ~$0.01219 / vCPU-hour |
| FARGATE_SPOT memory | ~$0.00133 / GB-hour |
| ECR storage | $0.10 / GB-month |
| Data transfer out | Standard EC2 rates |

**Estimate**: A 0.25 vCPU / 0.5 GB task running 24 h = ~$0.27/day (~$8/month) on on-demand Fargate.

---

## Service Limits (Default, us-east-1)

| Limit | Default |
|-------|---------|
| Clusters per account | 10,000 |
| Services per cluster | 5,000 |
| Tasks per service (desired) | 5,000 |
| Task definitions (families) | 1,000,000 |
| Container definitions per task def | 10 |

---

## Cleanup

This lab is conceptual — no AWS resources were created.

---

➡️ [Lab 02 → First Cluster, Task Definition & Run Task](../02-first-cluster-and-task/README.md)
