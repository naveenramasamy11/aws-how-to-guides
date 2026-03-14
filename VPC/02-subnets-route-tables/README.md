# Lab 02 — Subnets & Route Tables 🟢

## 🎯 Goal
Create public and private subnets across two AZs, attach an Internet Gateway, and configure route tables to make public subnets internet-routable while keeping private subnets isolated.

---

## 🧠 Concepts

### Public vs Private Subnet
| | Public Subnet | Private Subnet |
|--|--------------|---------------|
| **Route to internet** | `0.0.0.0/0 → IGW` | `0.0.0.0/0 → NAT GW` (or none) |
| **Auto-assign public IP** | Yes (recommended) | No |
| **Used for** | Load balancers, bastion, NAT GW | App servers, databases, Lambda |

### Internet Gateway (IGW)
- **One per VPC**, horizontally scaled and HA by AWS
- Enables bidirectional internet access for resources with public IPs
- Must be attached to the VPC AND referenced in the route table

### Route Tables
- Every subnet must be associated with exactly **one route table**
- A VPC has a **Main Route Table** — subnets not explicitly associated use this
- Local route (`10.10.0.0/16 → local`) is always present and cannot be removed

### Subnet CIDR Planning

```
VPC: 10.10.0.0/16

AZ: us-east-1a
  Public:  10.10.1.0/24   (251 IPs)
  Private: 10.10.2.0/24   (251 IPs)

AZ: us-east-1b
  Public:  10.10.3.0/24   (251 IPs)
  Private: 10.10.4.0/24   (251 IPs)

Reserved for future AZs / expansion:
  10.10.10.0/24 ... 10.10.255.0/24
```

---

## 🏗️ Architecture

```
prod-vpc (10.10.0.0/16)
│
├── Internet Gateway (igw-prod)
│
├── AZ: us-east-1a
│   ├── Public Subnet  (10.10.1.0/24) ──► Public Route Table ──► IGW
│   └── Private Subnet (10.10.2.0/24) ──► Private Route Table (no internet yet)
│
└── AZ: us-east-1b
    ├── Public Subnet  (10.10.3.0/24) ──► Public Route Table ──► IGW
    └── Private Subnet (10.10.4.0/24) ──► Private Route Table (no internet yet)
```

---

## 🧪 PoC

```bash
# Load VPC ID from Lab 01
source ~/.vpc_lab_vars
echo "Working with VPC: $VPC_ID"
```

### Step 1: Create subnets across two AZs
```bash
# Public subnet — AZ 1a
PUB_SUBNET_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a},{Key=Type,Value=public}]' \
  --query 'Subnet.SubnetId' --output text)
echo "Public subnet 1a: $PUB_SUBNET_1A ✅"

# Private subnet — AZ 1a
PRIV_SUBNET_1A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a},{Key=Type,Value=private}]' \
  --query 'Subnet.SubnetId' --output text)
echo "Private subnet 1a: $PRIV_SUBNET_1A ✅"

# Public subnet — AZ 1b
PUB_SUBNET_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.3.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1b},{Key=Type,Value=public}]' \
  --query 'Subnet.SubnetId' --output text)
echo "Public subnet 1b: $PUB_SUBNET_1B ✅"

# Private subnet — AZ 1b
PRIV_SUBNET_1B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.10.4.0/24 \
  --availability-zone us-east-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1b},{Key=Type,Value=private}]' \
  --query 'Subnet.SubnetId' --output text)
echo "Private subnet 1b: $PRIV_SUBNET_1B ✅"

# Save for later labs
cat >> ~/.vpc_lab_vars << EOF
export PUB_SUBNET_1A=$PUB_SUBNET_1A
export PRIV_SUBNET_1A=$PRIV_SUBNET_1A
export PUB_SUBNET_1B=$PUB_SUBNET_1B
export PRIV_SUBNET_1B=$PRIV_SUBNET_1B
EOF
```

### Step 2: Enable auto-assign public IPs on public subnets
```bash
aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_1A \
  --map-public-ip-on-launch

aws ec2 modify-subnet-attribute \
  --subnet-id $PUB_SUBNET_1B \
  --map-public-ip-on-launch

echo "Public subnets will auto-assign public IPs ✅"
```

### Step 3: Create and attach an Internet Gateway
```bash
# Create IGW
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=igw-prod}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)
echo "Created IGW: $IGW_ID ✅"

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID
echo "IGW attached to VPC ✅"

echo "export IGW_ID=$IGW_ID" >> ~/.vpc_lab_vars
```

### Step 4: Create the Public Route Table
```bash
# Create public route table
PUB_RT_ID=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-public}]' \
  --query 'RouteTable.RouteTableId' --output text)
echo "Public route table: $PUB_RT_ID ✅"

# Add default route to IGW
aws ec2 create-route \
  --route-table-id $PUB_RT_ID \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id $IGW_ID
echo "Default route → IGW added ✅"

# Associate public subnets with this route table
aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET_1A
aws ec2 associate-route-table --route-table-id $PUB_RT_ID --subnet-id $PUB_SUBNET_1B
echo "Public subnets associated with public route table ✅"

echo "export PUB_RT_ID=$PUB_RT_ID" >> ~/.vpc_lab_vars
```

### Step 5: Create Private Route Tables (one per AZ for AZ-isolation)
```bash
# Private RT for AZ 1a
PRIV_RT_1A=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-private-1a}]' \
  --query 'RouteTable.RouteTableId' --output text)

# Private RT for AZ 1b
PRIV_RT_1B=$(aws ec2 create-route-table \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-private-1b}]' \
  --query 'RouteTable.RouteTableId' --output text)

# Associate private subnets
aws ec2 associate-route-table --route-table-id $PRIV_RT_1A --subnet-id $PRIV_SUBNET_1A
aws ec2 associate-route-table --route-table-id $PRIV_RT_1B --subnet-id $PRIV_SUBNET_1B
echo "Private subnets associated with private route tables ✅"

cat >> ~/.vpc_lab_vars << EOF
export PRIV_RT_1A=$PRIV_RT_1A
export PRIV_RT_1B=$PRIV_RT_1B
EOF
```

### Step 6: Verify routing
```bash
echo "=== Public Route Table ==="
aws ec2 describe-route-tables --route-table-ids $PUB_RT_ID \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,GatewayId,State]' \
  --output table

echo "=== Private Route Table 1a ==="
aws ec2 describe-route-tables --route-table-ids $PRIV_RT_1A \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,GatewayId,State]' \
  --output table
```

Expected output for public RT:
```
--------------------------------------------
| DestinationCidrBlock | GatewayId  | State |
+----------------------+------------+-------+
| 0.0.0.0/0            | igw-xxxxxx | active|
| 10.10.0.0/16         | local      | active|
--------------------------------------------
```

### Step 7: Launch a test EC2 instance in the public subnet
```bash
source ~/.vpc_lab_vars

# Get latest AL2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
            "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

# Create a minimal security group
SG_TEST=$(aws ec2 create-security-group \
  --group-name sg-vpc-test \
  --description "Temporary SG for routing test" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-vpc-test}]' \
  --query 'GroupId' --output text)

# Allow outbound (default) and ICMP inbound for ping test
aws ec2 authorize-security-group-ingress \
  --group-id $SG_TEST \
  --protocol icmp --port -1 --cidr 0.0.0.0/0

# Launch instance in public subnet
INST_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $PUB_SUBNET_1A \
  --security-group-ids $SG_TEST \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=vpc-test-instance}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INST_ID
echo "Instance $INST_ID running in public subnet ✅"

# Get public IP
PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INST_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Public IP: $PUBLIC_IP"

# Ping test (from your machine)
ping -c 3 $PUBLIC_IP
```

---

## ✅ Cleanup
> Keep VPC, subnets, IGW, and route tables — used in later labs.
> Clean up only the test instance and security group:

```bash
source ~/.vpc_lab_vars

aws ec2 terminate-instances --instance-ids $INST_ID
aws ec2 wait instance-terminated --instance-ids $INST_ID
aws ec2 delete-security-group --group-id $SG_TEST
echo "Test resources cleaned up ✅"
```

---

## ➡️ Next Lab
[Lab 03 — Security Groups & NACLs →](../03-security-groups-nacls/README.md)
