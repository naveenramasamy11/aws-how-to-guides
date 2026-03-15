# Lab 01 — EC2 Service Overview: Concepts, Instance Types & Pricing

🟢 **Difficulty:** Beginner | **Time:** 30 min | **Cost:** $0 (read-only API calls)

---

## Goal

Understand the EC2 service model — instance families, purchasing options, the Nitro hypervisor architecture, and how to reason about cost before you launch a single instance.

---

## Concepts

### Instance Families

| Family | Optimised For | Example Types |
|--------|--------------|---------------|
| **T** (Burstable) | Dev/test, low baseline CPU | t3.micro, t4g.small |
| **M** (General Purpose) | Balanced CPU/RAM | m6i.large, m7g.xlarge |
| **C** (Compute) | CPU-intensive workloads | c7i.2xlarge, c6g.4xlarge |
| **R** (Memory) | In-memory DBs, caches | r6i.4xlarge, r7g.8xlarge |
| **G / P** (GPU) | ML training, inference, graphics | g4dn.xlarge, p4d.24xlarge |
| **I** (Storage) | High IOPS NVMe | i4i.xlarge |
| **X** (Memory Extreme) | SAP HANA, in-memory analytics | x2idn.16xlarge |
| **Inf / Trn** (Inferentia/Trainium) | AWS custom ML chips | inf2.xlarge, trn1.2xlarge |

### Processor Generation Suffixes

| Suffix | Processor |
|--------|-----------|
| (none) | Intel Xeon (older generation) |
| `i` | Intel Ice Lake / Sapphire Rapids |
| `a` | AMD EPYC |
| `g` | AWS Graviton (ARM) — best price/performance |
| `n` | High network bandwidth variant |
| `d` | Local NVMe storage |
| `z` | High clock speed |

### Purchasing Options Compared

| Option | Discount | Interruption | Best Use |
|--------|----------|-------------|----------|
| **On-Demand** | 0 % | None | Short-term, unpredictable |
| **Reserved (1yr)** | ~40 % | None | Steady-state production |
| **Reserved (3yr)** | ~60 % | None | Long-running, committed |
| **Savings Plans** | ~40-60 % | None | Flexible commitment |
| **Spot** | up to 90 % | 2-min notice | Fault-tolerant batch/ML |
| **Dedicated Host** | varies | None | Compliance, licensing |

### Nitro Hypervisor Architecture

```
┌────────────────────────────────────────────┐
│           Customer EC2 Instance             │
│  ┌──────────────────────────────────────┐  │
│  │  Guest OS (Amazon Linux / Ubuntu / …)│  │
│  └──────────────────────────────────────┘  │
│  ┌──────────────────────────────────────┐  │
│  │  Nitro Hypervisor (lightweight KVM)  │  │
│  └──────────────────────────────────────┘  │
└────────────────────────────────────────────┘
       │              │              │
       ▼              ▼              ▼
 Nitro Card       Nitro Card    Nitro Card
 (Network I/O)   (EBS I/O)     (Security)
```

Key Nitro benefits: near bare-metal performance, hardware-enforced isolation, offloaded I/O.

---

## Architecture — What You'll Build Across All Labs

```
         Internet Gateway
               │
          (Lab 02)
               │
         Public Subnet
          EC2 Instance
         (SSH / web app)
               │
          Private Subnet
          EC2 + EBS vol      ← Lab 03
               │
       AMI  ASG  ALB         ← Lab 04
               │
      IAM Role + SSM         ← Lab 05
               │
     ALB → ASG → Lambda      ← Lab 06
               │
    CloudWatch + Cost        ← Lab 07
```

---

## PoC Steps

### Step 1 — Confirm EC2 service limits in your account

```bash
aws service-quotas list-service-quotas \
  --service-code ec2 \
  --query 'Quotas[?contains(QuotaName, `Running On-Demand`)].{Name:QuotaName,Value:Value}' \
  --output table
```

Expected output (values vary by account):
```
-----------------------------------------------------------------
|                     ListServiceQuotas                         |
+-----------------------------------------------------------+---+
|  Name                                                     |Val|
+-----------------------------------------------------------+---+
|  Running On-Demand Standard (A, C, D, H, I, M, R, T, Z)  |32 |
|  Running On-Demand G and VT instances                     | 0 |
|  Running On-Demand P instances                            | 0 |
+-----------------------------------------------------------+---+
```

### Step 2 — Explore available instance types

```bash
# List all t3 and m6i types in your region with their vCPU and memory
aws ec2 describe-instance-types \
  --filters "Name=instance-type,Values=t3.*,m6i.*" \
  --query 'InstanceTypes[*].{Type:InstanceType,vCPU:VCpuInfo.DefaultVCpus,MemMiB:MemoryInfo.SizeInMiB,NetworkPerf:NetworkInfo.NetworkPerformance}' \
  --output table | sort
```

Expected output (partial):
```
---------------------------------------------------------------------
|                      DescribeInstanceTypes                        |
+-----------+---------+--------+----------------------------------+
|  MemMiB   | NetworkPerf | Type     | vCPU                     |
+-----------+---------+--------+----------------------------------+
|    1024   | Up to 5 Gigabit | t3.micro  |   2                 |
|    2048   | Up to 5 Gigabit | t3.small  |   2                 |
|    4096   | Up to 5 Gigabit | t3.medium |   2                 |
|   16384   | Up to 12.5 Gigabit | m6i.large | 2                |
|   32768   | Up to 12.5 Gigabit | m6i.xlarge| 4                |
```

### Step 3 — Check Spot price history for cost insight

```bash
# Get the last 10 spot price entries for t3.medium in us-east-1
aws ec2 describe-spot-price-history \
  --instance-types t3.medium \
  --product-descriptions "Linux/UNIX" \
  --max-items 10 \
  --query 'SpotPriceHistory[*].{AZ:AvailabilityZone,Price:SpotPrice,Time:Timestamp}' \
  --output table
```

### Step 4 — Estimate cost with the AWS Pricing Calculator

```bash
# On-Demand price for m6i.large in us-east-1 (requires pricing API)
aws pricing get-products \
  --service-code AmazonEC2 \
  --filters \
    "Type=TERM_MATCH,Field=instanceType,Value=m6i.large" \
    "Type=TERM_MATCH,Field=location,Value=US East (N. Virginia)" \
    "Type=TERM_MATCH,Field=operatingSystem,Value=Linux" \
    "Type=TERM_MATCH,Field=tenancy,Value=Shared" \
    "Type=TERM_MATCH,Field=capacitystatus,Value=Used" \
  --region us-east-1 \
  --query 'PriceList[0]' \
  --output text | python3 -c "
import sys, json
p = json.loads(sys.stdin.read())
od = p['terms']['OnDemand']
key1 = list(od.keys())[0]
key2 = list(od[key1]['priceDimensions'].keys())[0]
price = od[key1]['priceDimensions'][key2]['pricePerUnit']['USD']
print(f'm6i.large On-Demand: \${price}/hr (us-east-1 Linux)')
"
```

Expected output:
```
m6i.large On-Demand: $0.0960/hr (us-east-1 Linux)
```

### Step 5 — Get the latest Amazon Linux 2023 AMI ID

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023.*-x86_64" \
    "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text)

echo "Latest AL2023 AMI: $AMI_ID"
```

Expected output:
```
Latest AL2023 AMI: ami-0abcdef1234567890
```

---

## Cleanup

No resources were created in this lab. All commands were read-only API calls.

---

## Key Takeaways

- Instance type naming encodes processor architecture, size, and generation
- Graviton (`g`-suffix) instances offer the best price/performance ratio for most workloads
- Spot instances save up to 90 % but require fault-tolerant architecture
- Savings Plans are more flexible than Reserved Instances for mixed workloads
- Always retrieve the latest AMI ID dynamically — never hardcode AMI IDs

---

➡️ Next: [Lab 02 — Launch Your First Instance](../02-launch-first-instance/README.md)
