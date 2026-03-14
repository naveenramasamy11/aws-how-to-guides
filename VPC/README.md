# AWS VPC — How-To Guide 🌐

> Amazon Virtual Private Cloud (VPC) is the networking foundation of AWS.
> Everything you deploy lives inside a VPC. This guide takes you from a blank network to production-grade multi-VPC architectures with hands-on PoC labs.

---

## 🗺️ Learning Path

```
🟢 Beginner
  └── Lab 01 — VPC Fundamentals (CIDR, AZs, components)
  └── Lab 02 — Subnets & Route Tables (public vs private)

🟡 Intermediate
  └── Lab 03 — Security Groups & NACLs (stateful vs stateless)
  └── Lab 04 — NAT Gateway & Bastion Host (outbound internet, SSH jump)

🔴 Advanced
  └── Lab 05 — VPC Peering (cross-VPC private connectivity)
  └── Lab 06 — VPC Endpoints (private AWS service access, no internet)
  └── Lab 07 — Transit Gateway (hub-and-spoke, multi-VPC routing)
```

---

## 📋 Labs Index

| # | Lab | Key Concepts | Difficulty |
|---|-----|-------------|------------|
| 01 | [VPC Fundamentals](./01-vpc-fundamentals/README.md) | CIDR, Tenancy, DNS, Default VPC | 🟢 |
| 02 | [Subnets & Route Tables](./02-subnets-route-tables/README.md) | Public/Private subnets, IGW, Route Tables | 🟢 |
| 03 | [Security Groups & NACLs](./03-security-groups-nacls/README.md) | Stateful vs Stateless, Rule evaluation | 🟡 |
| 04 | [NAT Gateway & Bastion](./04-nat-gateway-bastion/README.md) | Outbound internet, SSH jump host | 🟡 |
| 05 | [VPC Peering](./05-vpc-peering/README.md) | Requester/Accepter, Route propagation | 🔴 |
| 06 | [VPC Endpoints](./06-vpc-endpoints/README.md) | Gateway (S3/DynamoDB), Interface (PrivateLink) | 🔴 |
| 07 | [Transit Gateway](./07-transit-gateway/README.md) | Hub-and-spoke, TGW route tables, attachments | 🔴 |

---

## 🔑 Core VPC Concepts (Quick Reference)

| Concept | Description |
|---------|-------------|
| **VPC** | Isolated virtual network scoped to a region |
| **Subnet** | Sub-division of a VPC scoped to a single AZ |
| **CIDR** | IP address range (e.g., `10.0.0.0/16`) |
| **IGW** | Internet Gateway — enables public internet access |
| **NAT Gateway** | Outbound-only internet for private subnets |
| **Route Table** | Rules determining where network traffic is directed |
| **Security Group** | Stateful firewall at the ENI level |
| **NACL** | Stateless firewall at the subnet level |
| **VPC Peering** | Private connection between two VPCs (non-transitive) |
| **VPC Endpoint** | Private access to AWS services without internet |
| **Transit Gateway** | Scalable hub connecting many VPCs and on-prem |
| **ENI** | Elastic Network Interface — virtual network card |

---

## 🏗️ VPC Component Map

```
Region: us-east-1
└── VPC: 10.0.0.0/16
    ├── AZ: us-east-1a
    │   ├── Public Subnet:  10.0.1.0/24  ──► IGW ──► Internet
    │   └── Private Subnet: 10.0.2.0/24  ──► NAT ──► Internet (outbound only)
    ├── AZ: us-east-1b
    │   ├── Public Subnet:  10.0.3.0/24
    │   └── Private Subnet: 10.0.4.0/24
    ├── Internet Gateway (IGW)
    ├── NAT Gateway (in public subnet, Elastic IP)
    ├── Route Tables
    │   ├── Public RT:  0.0.0.0/0 → IGW
    │   └── Private RT: 0.0.0.0/0 → NAT GW
    ├── Security Groups (stateful, ENI-level)
    └── NACLs (stateless, subnet-level)
```

---

## ⚙️ Setup

```bash
# Confirm AWS CLI and region
aws configure get region
aws ec2 describe-availability-zones --query 'AvailabilityZones[*].ZoneName' --output table

# Set a working region variable used across all labs
export AWS_REGION=us-east-1
export AWS_DEFAULT_REGION=us-east-1
```

---

## 📎 VPC CLI Cheat Sheet

```bash
# List all VPCs
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,CidrBlock,IsDefault,Tags[?Key==`Name`].Value|[0]]' --output table

# List subnets in a VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' --output table

# List route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query 'RouteTables[*].[RouteTableId,Routes[*].[DestinationCidrBlock,GatewayId]]' --output json

# List security groups
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query 'SecurityGroups[*].[GroupId,GroupName,Description]' --output table

# List NACLs
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=<VPC_ID>" \
  --query 'NetworkAcls[*].[NetworkAclId,IsDefault]' --output table
```
