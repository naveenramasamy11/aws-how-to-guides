# AWS Lambda — Module Overview

> **Build event-driven, serverless compute at any scale — pay only for what you use.**
> This module takes you from deploying your first function to running production-grade Lambda architectures with observability, security, and cost controls built in.

---

## 🗺️ Learning Path

```
🟢 Lab 01 → Overview & concepts
🟢 Lab 02 → First function (CLI + Python/Node)
🟡 Lab 03 → Triggers & event sources
🟡 Lab 04 → Advanced features (Layers, env vars, concurrency, SnapStart)
🔴 Lab 05 → IAM & security (least-privilege, resource policies, VPC, signing)
🔴 Lab 06 → Integration pattern (S3 → Lambda → DynamoDB/SNS pipeline)
🔴 Lab 07 → Production patterns (monitoring, cost, retries, failure handling)
```

---

## 📋 Labs Index

| Lab | Title | Difficulty | Time |
|-----|-------|------------|------|
| [01](./01-lambda-overview/README.md) | Lambda Overview — Architecture, Pricing & Quotas | 🟢 Beginner | ~20 min |
| [02](./02-first-function/README.md) | First Function — Deploy via CLI & Boto3 | 🟢 Beginner | ~30 min |
| [03](./03-triggers-and-events/README.md) | Triggers & Event Sources — S3, API GW, SQS, EventBridge | 🟡 Intermediate | ~45 min |
| [04](./04-advanced-features/README.md) | Advanced Features — Layers, Env Vars, Concurrency, SnapStart | 🟡 Intermediate | ~60 min |
| [05](./05-iam-security/README.md) | IAM & Security — Least-Privilege, VPC, Code Signing | 🔴 Advanced | ~60 min |
| [06](./06-integration-patterns/README.md) | Integration Pattern — S3 → Lambda → DynamoDB Pipeline | 🔴 Advanced | ~75 min |
| [07](./07-production-patterns/README.md) | Production Patterns — Monitoring, Cost, Retries, DLQs | 🔴 Advanced | ~75 min |

---

## 🧠 Core Concepts Quick-Reference

| Concept | Description |
|---------|-------------|
| **Function** | Unit of deployment — your code + runtime + config |
| **Handler** | Entry-point method invoked by the runtime (e.g. `lambda_function.lambda_handler`) |
| **Event** | JSON payload passed to your handler from the trigger source |
| **Context** | Object providing runtime metadata (remaining time, request ID, log stream) |
| **Execution environment** | Managed micro-VM (Firecracker) that runs your function |
| **Cold start** | First invocation latency — environment init + function init |
| **Warm start** | Reuse of a running environment — much faster |
| **Trigger** | AWS service or resource that invokes your function |
| **Destination** | Success/failure routing target (SQS, SNS, EventBridge, Lambda) |
| **Layer** | Shared .zip containing libraries/dependencies attached to functions |
| **Alias** | Named pointer to a function version (e.g. `PROD` → v42) |
| **Reserved concurrency** | Hard cap on simultaneous executions for a function |
| **Provisioned concurrency** | Pre-warmed environments — eliminates cold starts |
| **SnapStart** | Java init snapshot caching — up to 10x faster cold starts |

---

## 🏗️ ASCII Architecture Overview

```
                    ┌─────────────────────────────────────────────────┐
                    │                  TRIGGERS                        │
                    │  API Gateway  S3  SQS  SNS  EventBridge  DynamoDB│
                    └────────────────────┬────────────────────────────┘
                                         │ Event JSON
                                         ▼
                    ┌────────────────────────────────────────────────┐
                    │              AWS LAMBDA SERVICE                  │
                    │                                                  │
                    │  ┌──────────────────────────────────────────┐   │
                    │  │         Execution Environment             │   │
                    │  │  ┌─────────────────────────────────────┐ │   │
                    │  │  │  Runtime (Python / Node / Java / Go) │ │   │
                    │  │  │  + Your Code + Layers                │ │   │
                    │  │  └──────────────┬──────────────────────┘ │   │
                    │  │                 │ return value / error     │   │
                    │  └─────────────────┼────────────────────────┘   │
                    │                    │                              │
                    │  Concurrency mgmt  │  Aliases & Versions         │
                    │  Scaling           │  Reserved / Provisioned     │
                    └────────────────────┼────────────────────────────┘
                                         │
                    ┌────────────────────▼────────────────────────────┐
                    │                 DESTINATIONS                      │
                    │   CloudWatch Logs  SQS (DLQ)  SNS  EventBridge  │
                    └─────────────────────────────────────────────────┘
```

---

## ⚙️ Setup Commands

```bash
# Verify AWS CLI is configured
aws sts get-caller-identity --output table

# Set a convenience variable for the labs
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create a reusable deployment S3 bucket (used in several labs)
aws s3 mb s3://lambda-labs-${AWS_ACCOUNT_ID}-${AWS_REGION} \
  --region ${AWS_REGION}
```

---

## 🔧 CLI Cheat Sheet

```bash
# List all functions
aws lambda list-functions --query 'Functions[*].[FunctionName,Runtime,LastModified]' --output table

# Invoke a function synchronously
aws lambda invoke \
  --function-name my-function \
  --payload '{"key":"value"}' \
  --cli-binary-format raw-in-base64-out \
  response.json && cat response.json

# Get function config
aws lambda get-function-configuration \
  --function-name my-function --output table

# Update function code from a local zip
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Tail live logs
aws logs tail /aws/lambda/my-function --follow

# List event source mappings (SQS, DynamoDB, Kinesis triggers)
aws lambda list-event-source-mappings --output table

# Publish a new version
aws lambda publish-version --function-name my-function

# Create/update alias
aws lambda create-alias \
  --function-name my-function \
  --name PROD \
  --function-version 3

# Put concurrency limit
aws lambda put-function-concurrency \
  --function-name my-function \
  --reserved-concurrent-executions 50
```

---

## 📐 Runtimes Supported (as of 2024)

| Runtime | Identifier |
|---------|-----------|
| Python 3.12 | `python3.12` |
| Python 3.11 | `python3.11` |
| Node.js 20.x | `nodejs20.x` |
| Java 21 | `java21` |
| Go 1.x | `provided.al2023` |
| .NET 8 | `dotnet8` |
| Ruby 3.3 | `ruby3.3` |
| Custom runtime | `provided.al2023` |

---

➡️ Start with [Lab 01 — Lambda Overview](./01-lambda-overview/README.md)
