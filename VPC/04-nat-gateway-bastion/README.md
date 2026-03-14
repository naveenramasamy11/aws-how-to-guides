# Lab 04 — NAT Gateway & Bastion Host 🟡

## 🎯 Goal
Give private subnet instances **outbound internet access** (for package installs, AWS API calls) using a NAT Gateway — without exposing them to inbound connections. Set up a Bastion Host as the only SSH entry point.

---

## 🧠 Concepts

### NAT Gateway
- Deployed in a **public subnet** with an Elastic IP
- Routes outbound traffic from private subnets to the internet
- **Fully managed** by AWS — HA within an AZ, scales automatically
- Traffic flow: `Private Instance → NAT GW → IGW → Internet`
- **One NAT GW per AZ** is best practice (AZ resilience)

### NAT Gateway vs NAT Instance

| | NAT Gateway | NAT Instance |
|--|------------|-------------|
| **Management** | Fully managed | You manage the EC2 |
| **Availability** | HA within AZ | Single point of failure |
| **Bandwidth** | Up to 100 Gbps | Limited by instance type |
| **Cost** | Per-hour + per-GB | EC2 instance cost |
| **Security Groups** | Not applicable | Can attach SGs |
| **Use case** | Production | Lab / cost-sensitive |

### Bastion Host (Jump Server)
A hardened EC2 instance in the **public subnet** that acts as the single SSH entry point to private instances. Best practices:
- Use **EC2 Instance Connect** or **SSM Session Manager** instead of traditional bastion (no port 22 needed)
- If traditional bastion is required: restrict port 22 to your IP only

---

## 🏗️ Architecture

```
Internet
    │
    ▼
Internet Gateway
    │
    ▼
Public Subnet (10.10.1.0/24 — us-east-1a)
    ├── Bastion Host (sg-bastion: allow SSH from your-ip/32)
    └── NAT Gateway (Elastic IP)
         │
         ▼ (outbound internet for private instances)
Private Subnet (10.10.2.0/24 — us-east-1a)
    └── App Server (sg-app: no public IP, no inbound from internet)
         │ SSH via bastion ──────────────────────────────────────┘
```

---

## 🧪 PoC

```bash
source ~/.vpc_lab_vars
```

### Step 1: Allocate an Elastic IP for the NAT Gateway
```bash
EIP_ALLOC=$(aws ec2 allocate-address \
  --domain vpc \
  --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=eip-nat-1a}]' \
  --query 'AllocationId' --output text)
echo "Elastic IP allocated: $EIP_ALLOC ✅"

echo "export EIP_ALLOC=$EIP_ALLOC" >> ~/.vpc_lab_vars
```

### Step 2: Create the NAT Gateway in the public subnet
```bash
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUBNET_1A \
  --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=nat-gw-1a}]' \
  --query 'NatGateway.NatGatewayId' --output text)
echo "NAT Gateway creating: $NAT_GW_ID"

# Wait until it's available (takes ~1 minute)
echo "Waiting for NAT Gateway to become available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID
echo "NAT Gateway ready ✅"

echo "export NAT_GW_ID=$NAT_GW_ID" >> ~/.vpc_lab_vars
```

### Step 3: Add default route via NAT to the private route table
```bash
# Private subnet in 1a routes outbound via NAT GW in 1a (same AZ = no AZ data charges)
aws ec2 create-route \
  --route-table-id $PRIV_RT_1A \
  --destination-cidr-block 0.0.0.0/0 \
  --nat-gateway-id $NAT_GW_ID
echo "Default route → NAT GW added to private RT 1a ✅"

# Verify
aws ec2 describe-route-tables --route-table-ids $PRIV_RT_1A \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,NatGatewayId,GatewayId,State]' \
  --output table
```

### Step 4: Create the Bastion Security Group
```bash
# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "Your public IP: $MY_IP"

SG_BASTION=$(aws ec2 create-security-group \
  --group-name sg-bastion \
  --description "Bastion: SSH from admin IP only" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-bastion}]' \
  --query 'GroupId' --output text)
echo "Bastion SG: $SG_BASTION ✅"

# Allow SSH from your IP only
aws ec2 authorize-security-group-ingress \
  --group-id $SG_BASTION \
  --protocol tcp --port 22 \
  --cidr "${MY_IP}/32"
echo "SSH from ${MY_IP}/32 allowed ✅"

echo "export SG_BASTION=$SG_BASTION" >> ~/.vpc_lab_vars
```

### Step 5: Create an SSH key pair
```bash
# Create key pair (save the private key!)
aws ec2 create-key-pair \
  --key-name vpc-lab-key \
  --key-type ed25519 \
  --query 'KeyMaterial' --output text > ~/.ssh/vpc-lab-key.pem

chmod 600 ~/.ssh/vpc-lab-key.pem
echo "Key pair created: ~/.ssh/vpc-lab-key.pem ✅"
```

### Step 6: Launch the Bastion Host
```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
            "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

BASTION_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $PUB_SUBNET_1A \
  --security-group-ids $SG_BASTION \
  --key-name vpc-lab-key \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=bastion-host}]' \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled" \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $BASTION_ID

BASTION_IP=$(aws ec2 describe-instances --instance-ids $BASTION_ID \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "Bastion running — IP: $BASTION_IP ✅"
echo "export BASTION_ID=$BASTION_ID" >> ~/.vpc_lab_vars
echo "export BASTION_IP=$BASTION_IP" >> ~/.vpc_lab_vars
```

### Step 7: Launch a Private App Instance
```bash
# Allow SSH from bastion SG on the app SG
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 22 \
  --source-group $SG_BASTION

APP_INST_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --subnet-id $PRIV_SUBNET_1A \
  --security-group-ids $SG_APP \
  --key-name vpc-lab-key \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=app-server-private}]' \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled" \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $APP_INST_ID

APP_PRIV_IP=$(aws ec2 describe-instances --instance-ids $APP_INST_ID \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

echo "Private app server running — Private IP: $APP_PRIV_IP ✅"
echo "export APP_INST_ID=$APP_INST_ID" >> ~/.vpc_lab_vars
```

### Step 8: Test SSH via Bastion (jump host)
```bash
# SSH to bastion, then hop to private instance
ssh -i ~/.ssh/vpc-lab-key.pem \
    -J ec2-user@${BASTION_IP} \
    ec2-user@${APP_PRIV_IP}

# Once inside the private instance, verify outbound internet via NAT:
curl -s https://checkip.amazonaws.com
# Should return the NAT Gateway's Elastic IP — NOT the private IP ✅

# Test package install works (requires outbound internet)
sudo dnf update -y
```

### Step 9: (Better Practice) Use SSM Session Manager instead of Bastion
```bash
# No SSH key, no port 22, no bastion needed
# Requires SSM Agent (pre-installed on AL2023) + AmazonSSMManagedInstanceCore policy on the role

# Attach SSM policy to an instance role
aws iam attach-role-policy \
  --role-name <instance-role-name> \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Connect directly from CLI (no public IP needed)
aws ssm start-session --target $APP_INST_ID

# Port forwarding via SSM (no bastion)
aws ssm start-session \
  --target $APP_INST_ID \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["22"],"localPortNumber":["2222"]}'
```

---

## ✅ Cleanup

```bash
source ~/.vpc_lab_vars

# Terminate instances
aws ec2 terminate-instances --instance-ids $BASTION_ID $APP_INST_ID
aws ec2 wait instance-terminated --instance-ids $BASTION_ID $APP_INST_ID

# Delete NAT Gateway
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_ID
echo "Waiting for NAT GW deletion..."
sleep 60

# Release Elastic IP
aws ec2 release-address --allocation-id $EIP_ALLOC

# Delete key pair
aws ec2 delete-key-pair --key-name vpc-lab-key
rm -f ~/.ssh/vpc-lab-key.pem

# Delete bastion SG
aws ec2 delete-security-group --group-id $SG_BASTION

# Remove NAT route from private RT
aws ec2 delete-route --route-table-id $PRIV_RT_1A --destination-cidr-block 0.0.0.0/0

echo "All resources cleaned up ✅"
```

---

## ➡️ Next Lab
[Lab 05 — VPC Peering →](../05-vpc-peering/README.md)
