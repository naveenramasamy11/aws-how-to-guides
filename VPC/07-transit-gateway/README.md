# Lab 07 — Transit Gateway (Hub-and-Spoke) 🔴

## 🎯 Goal
Replace a mesh of VPC peering connections with a **Transit Gateway** — a single hub that connects multiple VPCs, allowing centralized routing, shared services, and on-premises connectivity.

---

## 🧠 Concepts

### VPC Peering vs Transit Gateway

| | VPC Peering | Transit Gateway |
|--|------------|----------------|
| **Topology** | Point-to-point (mesh) | Hub-and-spoke |
| **Scale** | N×(N-1)/2 connections | 1 hub, N spokes |
| **Transitive routing** | ❌ Not supported | ✅ Supported |
| **Cross-account** | Yes | Yes |
| **Cross-region** | Yes | Yes (TGW peering) |
| **VPN/Direct Connect** | Not supported | Supported |
| **Cost** | Per-connection + data | Per-attachment + data |
| **Best for** | 2–4 VPCs | 5+ VPCs, hybrid cloud |

### TGW Components

| Component | Description |
|-----------|-------------|
| **Transit Gateway** | The central hub router |
| **TGW Attachment** | A connection from a VPC (or VPN) to the TGW |
| **TGW Route Table** | Controls which attachments can talk to each other |
| **TGW Association** | Links an attachment to a route table |
| **TGW Propagation** | Auto-learns routes from an attachment into a route table |

### Routing Modes
- **Flat routing** — all VPCs share one route table (everyone talks to everyone)
- **Segmented routing** — separate route tables per environment (prod cannot reach dev)

---

## 🏗️ Architecture

```
                  Transit Gateway
                  ┌─────────────┐
                  │   TGW-hub   │
                  └──────┬──────┘
          ┌───────────────┼───────────────┐
          │               │               │
   TGW Attachment   TGW Attachment   TGW Attachment
          │               │               │
   prod-vpc           dev-vpc        shared-vpc
   10.10.0.0/16    10.20.0.0/16    10.30.0.0/16
                                  (DNS, monitoring)

TGW Route Table: rt-flat
  10.10.0.0/16 → prod-vpc attachment
  10.20.0.0/16 → dev-vpc attachment
  10.30.0.0/16 → shared-vpc attachment
```

---

## 🧪 PoC

```bash
source ~/.vpc_lab_vars
echo "Prod VPC: $VPC_ID"
```

### Step 1: Create two additional VPCs (dev and shared-services)
```bash
# Dev VPC
VPC_DEV=$(aws ec2 create-vpc \
  --cidr-block 10.20.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=dev-vpc}]' \
  --query 'Vpc.VpcId' --output text)
echo "Dev VPC: $VPC_DEV ✅"

# Shared Services VPC
VPC_SHARED=$(aws ec2 create-vpc \
  --cidr-block 10.30.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=shared-vpc}]' \
  --query 'Vpc.VpcId' --output text)
echo "Shared VPC: $VPC_SHARED ✅"

# Enable DNS on both
for VPC in $VPC_DEV $VPC_SHARED; do
  aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-support '{"Value":true}'
  aws ec2 modify-vpc-attribute --vpc-id $VPC --enable-dns-hostnames '{"Value":true}'
done

# Subnets
SUBNET_DEV=$(aws ec2 create-subnet \
  --vpc-id $VPC_DEV --cidr-block 10.20.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=dev-subnet}]' \
  --query 'Subnet.SubnetId' --output text)

SUBNET_SHARED=$(aws ec2 create-subnet \
  --vpc-id $VPC_SHARED --cidr-block 10.30.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=shared-subnet}]' \
  --query 'Subnet.SubnetId' --output text)

# Route tables
RT_DEV=$(aws ec2 create-route-table --vpc-id $VPC_DEV \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-dev}]' \
  --query 'RouteTable.RouteTableId' --output text)
RT_SHARED=$(aws ec2 create-route-table --vpc-id $VPC_SHARED \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-shared}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 associate-route-table --route-table-id $RT_DEV --subnet-id $SUBNET_DEV
aws ec2 associate-route-table --route-table-id $RT_SHARED --subnet-id $SUBNET_SHARED

cat >> ~/.vpc_lab_vars << EOF
export VPC_DEV=$VPC_DEV
export VPC_SHARED=$VPC_SHARED
export SUBNET_DEV=$SUBNET_DEV
export SUBNET_SHARED=$SUBNET_SHARED
export RT_DEV=$RT_DEV
export RT_SHARED=$RT_SHARED
EOF
```

### Step 2: Create the Transit Gateway
```bash
TGW_ID=$(aws ec2 create-transit-gateway \
  --description "Central TGW hub for prod/dev/shared VPCs" \
  --options "AmazonSideAsn=64512,AutoAcceptSharedAttachments=enable,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable,DnsSupport=enable,VpnEcmpSupport=enable" \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=tgw-hub}]' \
  --query 'TransitGateway.TransitGatewayId' --output text)

echo "Transit Gateway creating: $TGW_ID"
echo "Waiting for TGW to become available (takes ~3 minutes)..."
aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW_ID
echo "Transit Gateway ready ✅"

echo "export TGW_ID=$TGW_ID" >> ~/.vpc_lab_vars
```

### Step 3: Attach all three VPCs to the Transit Gateway
```bash
# Prod VPC attachment
ATTACH_PROD=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $VPC_ID \
  --subnet-ids $PRIV_SUBNET_1A \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=attach-prod}]' \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)
echo "Prod attachment: $ATTACH_PROD ✅"

# Dev VPC attachment
ATTACH_DEV=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $VPC_DEV \
  --subnet-ids $SUBNET_DEV \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=attach-dev}]' \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)
echo "Dev attachment: $ATTACH_DEV ✅"

# Shared VPC attachment
ATTACH_SHARED=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $VPC_SHARED \
  --subnet-ids $SUBNET_SHARED \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=attach-shared}]' \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)
echo "Shared attachment: $ATTACH_SHARED ✅"

# Wait for all attachments to be available
echo "Waiting for attachments..."
for ATTACH in $ATTACH_PROD $ATTACH_DEV $ATTACH_SHARED; do
  aws ec2 wait transit-gateway-attachment-available \
    --transit-gateway-attachment-ids $ATTACH
done
echo "All attachments available ✅"

cat >> ~/.vpc_lab_vars << EOF
export ATTACH_PROD=$ATTACH_PROD
export ATTACH_DEV=$ATTACH_DEV
export ATTACH_SHARED=$ATTACH_SHARED
EOF
```

### Step 4: Update VPC route tables to send cross-VPC traffic via TGW
```bash
# Prod VPC: route to dev and shared via TGW
aws ec2 create-route --route-table-id $PRIV_RT_1A \
  --destination-cidr-block 10.20.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $PRIV_RT_1A \
  --destination-cidr-block 10.30.0.0/16 --transit-gateway-id $TGW_ID
echo "Prod RT updated ✅"

# Dev VPC: route to prod and shared via TGW
aws ec2 create-route --route-table-id $RT_DEV \
  --destination-cidr-block 10.10.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $RT_DEV \
  --destination-cidr-block 10.30.0.0/16 --transit-gateway-id $TGW_ID
echo "Dev RT updated ✅"

# Shared VPC: route to prod and dev via TGW
aws ec2 create-route --route-table-id $RT_SHARED \
  --destination-cidr-block 10.10.0.0/16 --transit-gateway-id $TGW_ID
aws ec2 create-route --route-table-id $RT_SHARED \
  --destination-cidr-block 10.20.0.0/16 --transit-gateway-id $TGW_ID
echo "Shared RT updated ✅"
```

### Step 5: View the TGW Route Table (flat routing)
```bash
# Get the default TGW route table ID
TGW_RT_ID=$(aws ec2 describe-transit-gateway-route-tables \
  --filters "Name=transit-gateway-id,Values=$TGW_ID" \
  --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' \
  --output text)
echo "TGW Route Table: $TGW_RT_ID"

# View routes (auto-propagated from attachments)
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id $TGW_RT_ID \
  --filters "Name=state,Values=active" \
  --query 'Routes[*].[DestinationCidrBlock,Type,State,TransitGatewayAttachments[0].TransitGatewayAttachmentId]' \
  --output table
```

### Step 6: Segmented Routing (Prod cannot reach Dev)
```bash
# Create a separate TGW route table for prod (isolated from dev)
TGW_RT_PROD=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=rt-tgw-prod}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

TGW_RT_DEV=$(aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=rt-tgw-dev}]' \
  --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

# Associate prod attachment with prod RT
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id $TGW_RT_PROD \
  --transit-gateway-attachment-id $ATTACH_PROD

# Propagate shared attachment into BOTH route tables
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $TGW_RT_PROD \
  --transit-gateway-attachment-id $ATTACH_SHARED

# Prod can ONLY reach shared — not dev
# Dev can ONLY reach shared — not prod
# Shared can reach both
echo "Segmented routing configured: prod ↔ shared, dev ↔ shared, prod ✗ dev ✅"
```

---

## ✅ Cleanup

```bash
source ~/.vpc_lab_vars

# Delete attachments
for ATTACH in $ATTACH_PROD $ATTACH_DEV $ATTACH_SHARED; do
  aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $ATTACH
done
echo "Waiting for attachments to detach..."
sleep 60

# Delete TGW
aws ec2 delete-transit-gateway --transit-gateway-id $TGW_ID

# Delete dev VPC
aws ec2 delete-route --route-table-id $RT_DEV --destination-cidr-block 10.10.0.0/16 2>/dev/null
aws ec2 delete-route --route-table-id $RT_DEV --destination-cidr-block 10.30.0.0/16 2>/dev/null
aws ec2 disassociate-route-table \
  --association-id $(aws ec2 describe-route-tables --route-table-ids $RT_DEV \
    --query 'RouteTables[0].Associations[0].RouteTableAssociationId' --output text)
aws ec2 delete-route-table --route-table-id $RT_DEV
aws ec2 delete-subnet --subnet-id $SUBNET_DEV
aws ec2 delete-vpc --vpc-id $VPC_DEV

# Delete shared VPC
aws ec2 disassociate-route-table \
  --association-id $(aws ec2 describe-route-tables --route-table-ids $RT_SHARED \
    --query 'RouteTables[0].Associations[0].RouteTableAssociationId' --output text)
aws ec2 delete-route-table --route-table-id $RT_SHARED
aws ec2 delete-subnet --subnet-id $SUBNET_SHARED
aws ec2 delete-vpc --vpc-id $VPC_SHARED

echo "All TGW resources cleaned up ✅"
```

---

## 🎓 VPC Learning Path Complete!

| Lab | Topic | Status |
|-----|-------|--------|
| 01 | VPC Fundamentals | ✅ |
| 02 | Subnets & Route Tables | ✅ |
| 03 | Security Groups & NACLs | ✅ |
| 04 | NAT Gateway & Bastion | ✅ |
| 05 | VPC Peering | ✅ |
| 06 | VPC Endpoints | ✅ |
| 07 | Transit Gateway | ✅ |

**Next up:** S3 Security, KMS, AWS Organizations & SCPs.
