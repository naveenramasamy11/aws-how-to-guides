# Lab 01 — Lambda Overview: Architecture, Pricing & Quotas

🟢 **Difficulty:** Beginner | ⏱️ **Time:** ~20 min | **No AWS resources created**

---

## 🎯 Goal

Understand how AWS Lambda works under the hood — the execution model, billing dimensions, cold/warm start mechanics, and hard limits — so every decision in later labs is grounded in first principles.

---

## 📖 Concepts

### What Is Lambda?

AWS Lambda is a **Function-as-a-Service (FaaS)** compute platform. You upload code, define a trigger, and AWS handles provisioning, scaling, patching, and fault tolerance. Functions scale horizontally from 0 to thousands of concurrent executions automatically.

### Execution Model

```
Invocation request
       │
       ▼
 ┌─────────────────────────────────────────────────────────────┐
 │                  Lambda Service (Control Plane)              │
 │                                                              │
 │  1. Find available warm execution environment                │
 │     OR spin up a new one (cold start)                        │
 │                                                              │
 │  ┌───────────────────────────────────────────────────────┐  │
 │  │          Execution Environment (Firecracker MVM)       │  │
 │  │                                                        │  │
 │  │  Phase 1 — INIT (cold start only):                    │  │
 │  │    • Download code/layers from S3                      │  │
 │  │    • Start runtime (Python/Node/JVM…)                  │  │
 │  │    • Run initialization code (outside handler)         │  │
 │  │                                                        │  │
 │  │  Phase 2 — INVOKE (every call):                       │  │
 │  │    • Pass event JSON to handler                        │  │
 │  │    • Execute handler logic                             │  │
 │  │    • Return response or error                          │  │
 │  │                                                        │  │
 │  │  Phase 3 — SHUTDOWN (after idle timeout ~15 min):     │  │
 │  │    • Runtime extension SHUTDOWN event                  │  │
 │  │    • Environment destroyed                             │  │
 │  └───────────────────────────────────────────────────────┘  │
 └─────────────────────────────────────────────────────────────┘
```

### Cold Start vs. Warm Start

| Factor | Cold Start | Warm Start |
|--------|-----------|------------|
| Frequency | First invoke + after idle | Subsequent invokes |
| Extra latency | 100 ms – 3 s (JVM: up to 10 s) | ~1–5 ms |
| Cause | New execution environment initialization | Reuse of existing environment |
| Mitigation | Provisioned Concurrency, SnapStart (Java), keep-alive pings | N/A — already warm |

**Key insight:** Code outside the handler runs only during cold starts. Put SDK clients, DB connections, and config loading there to reuse across warm invocations.

```python
import boto3

# This runs ONCE per cold start — reused across warm invocations
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('my-table')

def lambda_handler(event, context):
    # This runs on EVERY invocation
    item = table.get_item(Key={'id': event['id']})
    return item
```

### Invocation Types

| Type | Description | Use Case |
|------|-------------|----------|
| **RequestResponse** (sync) | Caller waits for response | API Gateway, direct SDK calls |
| **Event** (async) | Lambda queues the request, returns 202 immediately | S3, SNS, EventBridge |
| **DryRun** | Validate permissions without executing | IAM testing |

### Concurrency Model

```
Total Account Concurrency (default 1,000 per region)
         │
         ├── Function A — Reserved: 200
         │     ├── Provisioned: 50 (always warm)
         │     └── On-demand: up to 150
         │
         ├── Function B — Reserved: 100
         │
         └── Unreserved pool — shared by all other functions
                 (account limit − all reserved concurrency)
```

- **Throttle** occurs when concurrency limit is hit → 429 error
- **Burst limit**: 3,000 initial burst, then +500/minute per region

---

## 💰 Pricing Model

Lambda pricing has two components:

### 1. Requests
| Tier | Price |
|------|-------|
| First 1M requests/month | **Free** |
| After 1M | $0.20 per 1M requests |

### 2. Duration (GB-seconds)
| Tier | Price |
|------|-------|
| First 400,000 GB-seconds/month | **Free** |
| After free tier | $0.0000166667 per GB-second |

**Duration = (memory allocated in GB) × (execution time in seconds)**

**Example calculation:**

```
Function: 512 MB memory, runs for 200 ms per invocation
1M invocations/month

Duration = 0.5 GB × 0.2 s = 0.1 GB-s per invocation
Monthly GB-s = 1,000,000 × 0.1 = 100,000 GB-s

Cost breakdown:
  Requests:  $0.20 (after free tier)
  Duration:  100,000 × $0.0000166667 = $1.67
  Total:     ~$1.87/month for 1M invocations
```

### Additional Cost Factors
- **Provisioned concurrency**: $0.000004646 per GB-second allocated
- **Data transfer**: Standard EC2 data transfer rates
- **SnapStart** (Java): Free — snapshot storage included

---

## 📏 Key Quotas & Limits

| Limit | Default | Adjustable? |
|-------|---------|-------------|
| Concurrent executions (per region) | 1,000 | ✅ Yes (support ticket) |
| Function timeout | Max 15 minutes | ❌ (15 min is the hard max) |
| Deployment package size (zip, direct upload) | 50 MB | ❌ |
| Deployment package size (via S3) | 250 MB unzipped | ❌ |
| `/tmp` ephemeral storage | 512 MB – 10,240 MB | ✅ Configurable |
| Memory allocation | 128 MB – 10,240 MB | ✅ Configurable |
| Environment variables | 4 KB total | ❌ |
| Layers per function | 5 | ❌ |
| Function payload (sync) | 6 MB request + 6 MB response | ❌ |
| Function payload (async) | 256 KB | ❌ |
| Burst concurrency | 3,000 initial | Per-region |

---

## 🔍 Runtime Lifecycle Detail

```
┌──────────────────────────────────────────────────────────────┐
│                  Extension Init Phase                         │
│  (Lambda Extensions API — monitoring tools attach here)       │
└──────────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────────┐
│                  Runtime Init Phase                           │
│  • Runtime bootstraps (python3.12, node, JVM start)          │
│  • Your module-level code executes                            │
│  • SDK clients initialized                                    │
└──────────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────────┐
│                  Function Init Phase (if SnapStart)           │
│  • Java: snapshot taken here, restored on next cold start     │
└──────────────────────────────────────────────────────────────┘
                          │
                    INVOKE loop
                    (handler called N times)
                          │
┌──────────────────────────────────────────────────────────────┐
│                  Shutdown Phase                               │
│  • Extensions receive SHUTDOWN signal                         │
│  • Flush buffered logs/metrics                                │
└──────────────────────────────────────────────────────────────┘
```

---

## 📊 Choosing the Right Memory Setting

Lambda allocates CPU proportionally to memory. More memory = faster CPU = shorter duration. The **cost-optimal** setting is not always the minimum memory.

| Memory | vCPU Allocation | Typical Use |
|--------|----------------|-------------|
| 128 MB | 0.0625 vCPU | Tiny I/O functions |
| 512 MB | 0.25 vCPU | Light compute |
| 1,769 MB | 1 full vCPU | Full single-core workloads |
| 3,538 MB | 2 vCPU | Parallel compute |
| 10,240 MB | 6 vCPU | Memory/CPU intensive |

**Tip:** Use [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) (open-source Step Functions state machine) to find the optimal memory setting automatically.

---

## ✅ Key Takeaways

- Lambda bills on **requests + GB-seconds** with a generous free tier
- Cold starts happen on new environments; warm starts reuse existing ones
- Code outside the handler runs once per cold start — use it for expensive initializations
- Concurrency is regional and shared; use reserved concurrency to protect critical functions
- The 15-minute timeout is a hard limit — use Step Functions or Batch for longer workloads

---

➡️ Next: [Lab 02 — Deploy Your First Lambda Function](../02-first-function/README.md)
