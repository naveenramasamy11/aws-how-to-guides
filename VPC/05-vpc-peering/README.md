# Lab 05 — VPC Peering 🔴

## 🎯 Goal
Connect two VPCs (in the same region) so their instances can communicate using private IP addresses — without traffic ever leaving the AWS backbone.

---

## 🧠 Concepts

### What is VPC Peering?
A **VPC Peering connection** is a networking connection between two VPCs using AWS internal infrastructure. Traffic stays on the AWS network (never traverses the internet).

### Key Rules & Limitations
| Rule | Detail |
|------|--------|
| **Non-transitive** | A→B peered, B→C peered, A cannot reach C through B |
| **No overlapping CIDRs** | Peer VPCs must have non-overlapping CIDR blocks |
| **Cross-region supported** | Latency increases, inter-region data transfer fees apply |
| **Cross-account supported** | Requires explicit acceptance in the target account |
| **Route tables required** | Peering doesn't auto-update routes — you must add them |
| **Security groups** | Can reference peer SGs in the same region |

### Non-Transitive Peering (Important!)
```
VPC-A ──── peered ──── VPC-B ──── peered ──── VPC-C

VPC-A CANNOT talk to VPC-C through VPC-B.
Solution: peer A↔C directly, or use Transit Gateway (Lab 07).
```

---

## 🏗️ Architecture

```
VPC-A (prod-vpc)   10.10.0.0/16    ←── already created (Lab 01)
        │
  VPC Peering Connection
        │
VPC-B (dev-vpc)    10.20.0.0/16    ←── create in this lab

After peering:
  VPC-A instance (10.10.2.x) → reaches → VPC-B instance (10.20.2.x)
  VPC-B instance (10.20.2.x) → reaches → VPC-A instance (10.10.2.x)
```

---

## 🧪 PoC

```bash
source ~/.vpc_lab_vars
echo "Prod VPC (VPC-A): $VPC_ID"
```

### Step 1: Create the second VPC (dev-vpc)
```bash
VPC_B=$(aws ec2 create-vpc \
  --cidr-block 10.20.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=dev-vpc}]' \
  --query 'Vpc.VpcId' --output text)
echo "Dev VPC (VPC-B): $VPC_B ✅"

# Enable DNS
aws ec2 modify-vpc-attribute --vpc-id $VPC_B --enable-dns-support '{"Value":true}'
aws ec2 modify-vpc-attribute --vpc-id $VPC_B --enable-dns-hostnames '{"Value":true}'

# Create a private subnet in VPC-B
SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_B \
  --cidr-block 10.20.2.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=dev-private-1a}]' \
  --query 'Subnet.SubnetId' --output text)
echo "Dev subnet: $SUBNET_B ✅"

# Route table for VPC-B
RT_B=$(aws ec2 create-route-table \
  --vpc-id $VPC_B \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=rt-dev-private}]' \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 associate-route-table --route-table-id $RT_B --subnet-id $SUBNET_B

cat >> ~/.vpc_lab_vars << EOF
export VPC_B=$VPC_B
export SUBNET_B=$SUBNET_B
export RT_B=$RT_B
EOF
```

### Step 2: Create the VPC Peering Connection
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_ID \
  --peer-vpc-id $VPC_B \
  --peer-owner-id $ACCOUNT_ID \
  --peer-region us-east-1 \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=prod-to-dev-peering}]' \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text)
echo "Peering connection created: $PEERING_ID (pending-acceptance) ✅"

echo "export PEERING_ID=$PEERING_ID" >> ~/.vpc_lab_vars
```

### Step 3: Accept the Peering Connection (in same account, different VPC)
```bash
# In same account: accept it programmatically
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID

echo "Peering connection accepted ✅"

# Verify it's active
aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_ID \
  --query 'VpcPeeringConnections[0].{Id:VpcPeeringConnectionId,Status:Status.Code,VPC_A:RequesterVpcInfo.CidrBlock,VPC_B:AccepterVpcInfo.CidrBlock}' \
  --output table
```

### Step 4: Update Route Tables in both VPCs
```bash
# VPC-A (prod): add route to VPC-B's CIDR via peering
aws ec2 create-route \
  --route-table-id $PRIV_RT_1A \
  --destination-cidr-block 10.20.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID
echo "Route to VPC-B added in VPC-A private RT ✅"

# VPC-B (dev): add route to VPC-A's CIDR via peering
aws ec2 create-route \
  --route-table-id $RT_B \
  --destination-cidr-block 10.10.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID
echo "Route to VPC-A added in VPC-B private RT ✅"

# Verify routes
echo "=== VPC-A Private Route Table ==="
aws ec2 describe-route-tables --route-table-ids $PRIV_RT_1A \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,GatewayId,State]' \
  --output table

echo "=== VPC-B Route Table ==="
aws ec2 describe-route-tables --route-table-ids $RT_B \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,GatewayId,State]' \
  --output table
```

### Step 5: Create Security Groups that allow cross-VPC traffic
```bash
# SG for VPC-B test instance — allow ICMP and SSH from VPC-A CIDR
SG_B=$(aws ec2 create-security-group \
  --group-name sg-dev-instance \
  --description "Dev instance: allow traffic from prod VPC" \
  --vpc-id $VPC_B \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_B \
  --protocol icmp --port -1 \
  --cidr 10.10.0.0/16
echo "ICMP from prod VPC allowed on dev SG ✅"

aws ec2 authorize-security-group-ingress \
  --group-id $SG_B \
  --protocol tcp --port 22 \
  --cidr 10.10.0.0/16
echo "SSH from prod VPC allowed on dev SG ✅"

echo "export SG_B=$SG_B" >> ~/.vpc_lab_vars
```

### Step 6: Launch test instances and verify connectivity
```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
            "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

# Instance in VPC-A private subnet
INST_A=$(aws ec2 run-instances \
  --image-id $AMI_ID --instance-type t3.micro \
  --subnet-id $PRIV_SUBNET_1A \
  --security-group-ids $SG_APP \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=prod-test-instance}]' \
  --query 'Instances[0].InstanceId' --output text)

# Instance in VPC-B subnet
INST_B=$(aws ec2 run-instances \
  --image-id $AMI_ID --instance-type t3.micro \
  --subnet-id $SUBNET_B \
  --security-group-ids $SG_B \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=dev-test-instance}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $INST_A $INST_B
echo "Both test instances running ✅"

IP_A=$(aws ec2 describe-instances --instance-ids $INST_A \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)
IP_B=$(aws ec2 describe-instances --instance-ids $INST_B \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

echo "VPC-A instance IP: $IP_A"
echo "VPC-B instance IP: $IP_B"

# From VPC-A instance (via SSM), ping VPC-B instance:
# aws ssm start-session --target $INST_A
# ping -c 3 $IP_B   ← should succeed ✅
```

### Step 7: Enable DNS Resolution across peered VPCs
```bash
# Without this, DNS names in VPC-B won't resolve from VPC-A
aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id $PEERING_ID \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

echo "Cross-VPC DNS resolution enabled ✅"
```

---

## ✅ Cleanup

```bash
source ~/.vpc_lab_vars

# Terminate test instances
aws ec2 terminate-instances --instance-ids $INST_A $INST_B
aws ec2 wait instance-terminated --instance-ids $INST_A $INST_B

# Remove peering routes
aws ec2 delete-route --route-table-id $PRIV_RT_1A --destination-cidr-block 10.20.0.0/16
aws ec2 delete-route --route-table-id $RT_B --destination-cidr-block 10.10.0.0/16

# Delete peering connection
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PEERING_ID

# Delete VPC-B resources
aws ec2 delete-security-group --group-id $SG_B
aws ec2 disassociate-route-table \
  --association-id $(aws ec2 describe-route-tables --route-table-ids $RT_B \
    --query 'RouteTables[0].Associations[0].RouteTableAssociationId' --output text)
aws ec2 delete-route-table --route-table-id $RT_B
aws ec2 delete-subnet --subnet-id $SUBNET_B
aws ec2 delete-vpc --vpc-id $VPC_B

echo "All peering resources cleaned up ✅"
```

---

## ➡️ Next Lab
[Lab 06 — VPC Endpoints →](../06-vpc-endpoints/README.md)
