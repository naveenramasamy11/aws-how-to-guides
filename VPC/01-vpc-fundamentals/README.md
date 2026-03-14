# Lab 01 — VPC Fundamentals 🟢

## 🎯 Goal
Understand the core building blocks of a VPC — CIDR sizing, tenancy, DNS settings, and the difference between the Default VPC and a custom VPC — and create your first custom VPC from scratch.

---

## 🧠 Concepts

### What is a VPC?
A VPC is a **logically isolated section of the AWS cloud** where you launch AWS resources. It spans all Availability Zones in a region but its subnets are scoped to individual AZs.

### CIDR Sizing Guide

| CIDR | Usable IPs | Use Case |
|------|-----------|----------|
| `/28` | 11 | Tiny (Lambda, test) |
| `/24` | 251 | Small subnet |
| `/22` | 1,019 | Medium VPC |
| `/20` | 4,091 | Large VPC |
| `/16` | 65,531 | Max recommended |

> AWS **reserves 5 IPs per subnet**: network address, VPC router, DNS, future use, broadcast.
> So a `/24` gives you 256 - 5 = **251 usable IPs**.

### Default VPC vs Custom VPC

| | Default VPC | Custom VPC |
|--|------------|-----------|
| **CIDR** | `172.31.0.0/16` (fixed) | You choose |
| **Subnets** | Auto-created in every AZ | You create |
| **Internet access** | Yes (all subnets public) | You control |
| **Best for** | Quick testing | Production workloads |

### DNS in VPC
Two key settings on every VPC:

| Setting | Description |
|---------|-------------|
| `enableDnsSupport` | Enables the Route 53 resolver (169.254.169.253) |
| `enableDnsHostnames` | Assigns public DNS names to instances with public IPs |

Both must be `true` for Route 53 private hosted zones to work.

### Tenancy
| Tenancy | Description |
|---------|-------------|
| `default` | Shared hardware (multi-tenant) — cost-effective |
| `dedicated` | Single-tenant hardware — compliance/licensing use cases |

---

## 🏗️ Architecture

```
Region: us-east-1
└── Custom VPC: prod-vpc (10.10.0.0/16)
    ├── DNS Support: enabled
    ├── DNS Hostnames: enabled
    ├── Tenancy: default
    └── (no subnets yet — added in Lab 02)
```

---

## 🧪 PoC

### Step 1: Inspect the Default VPC
```bash
# View the Default VPC
aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].{VpcId:VpcId,CIDR:CidrBlock,DNS:EnableDnsSupport,Hostnames:EnableDnsHostnames}' \
  --output table

# Count auto-created default subnets (one per AZ)
aws ec2 describe-subnets \
  --filters "Name=defaultForAz,Values=true" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone]' \
  --output table
```

### Step 2: Create a custom VPC
```bash
# Create the VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.10.0.0/16 \
  --instance-tenancy default \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc},{Key=Env,Value=prod}]' \
  --query 'Vpc.VpcId' \
  --output text)

echo "Created VPC: $VPC_ID ✅"
```

### Step 3: Enable DNS support and hostnames
```bash
aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-support '{"Value":true}'

aws ec2 modify-vpc-attribute \
  --vpc-id $VPC_ID \
  --enable-dns-hostnames '{"Value":true}'

echo "DNS support and hostnames enabled ✅"

# Verify
aws ec2 describe-vpcs --vpc-ids $VPC_ID \
  --query 'Vpcs[0].{VpcId:VpcId,CIDR:CidrBlock,DNS:EnableDnsSupport,Hostnames:EnableDnsHostnames}' \
  --output table
```

### Step 4: Add a secondary CIDR block (IPv4 extension)
```bash
# Extend the VPC with a secondary CIDR (useful when primary is exhausted)
aws ec2 associate-vpc-cidr-block \
  --vpc-id $VPC_ID \
  --cidr-block 10.20.0.0/16

echo "Secondary CIDR 10.20.0.0/16 associated ✅"

# View all CIDRs on the VPC
aws ec2 describe-vpcs --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[*].[CidrBlock,CidrBlockState.State]' \
  --output table
```

### Step 5: Enable IPv6 (dual-stack)
```bash
# Add an Amazon-provided IPv6 CIDR
aws ec2 associate-vpc-cidr-block \
  --vpc-id $VPC_ID \
  --amazon-provided-ipv6-cidr-block

echo "IPv6 CIDR block requested ✅"

# Check IPv6 CIDR (takes a moment to assign)
sleep 5
aws ec2 describe-vpcs --vpc-ids $VPC_ID \
  --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[*].[Ipv6CidrBlock,Ipv6CidrBlockState.State]' \
  --output table
```

### Step 6: Save VPC ID for use in subsequent labs
```bash
# Persist for use in later labs
echo "export VPC_ID=$VPC_ID" >> ~/.vpc_lab_vars
echo "VPC_ID=$VPC_ID saved to ~/.vpc_lab_vars ✅"

# To reload in a new terminal:
# source ~/.vpc_lab_vars
```

### Step 7: View full VPC details
```bash
aws ec2 describe-vpcs --vpc-ids $VPC_ID
```

---

## ✅ Cleanup
> Keep this VPC — it's used in all subsequent VPC labs.
> Run this only if you want to destroy everything:
```bash
# Disassociate secondary CIDR first
ASSOC_ID=$(aws ec2 describe-vpcs --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[?CidrBlock==`10.20.0.0/16`].AssociationId' \
  --output text)
[ -n "$ASSOC_ID" ] && aws ec2 disassociate-vpc-cidr-block --association-id $ASSOC_ID

# Delete VPC (only works after all dependent resources removed)
aws ec2 delete-vpc --vpc-id $VPC_ID
echo "VPC deleted ✅"
```

---

## ➡️ Next Lab
[Lab 02 — Subnets & Route Tables →](../02-subnets-route-tables/README.md)
