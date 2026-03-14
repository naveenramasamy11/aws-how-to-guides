# Lab 06 — VPC Endpoints (Gateway & Interface) 🔴

## 🎯 Goal
Enable private EC2 instances to access **S3** and **AWS Systems Manager** without internet, NAT Gateway, or public IPs — using VPC Endpoints. This reduces cost, latency, and attack surface.

---

## 🧠 Concepts

### Why VPC Endpoints?
Without endpoints, private instances need a NAT Gateway to call AWS APIs (S3, SSM, CloudWatch, etc.) — incurring NAT data processing charges and routing traffic through the internet path.

With VPC Endpoints, traffic stays **entirely within the AWS network**.

### Two Types of VPC Endpoints

| Type | Protocol | Services | Cost |
|------|----------|----------|------|
| **Gateway** | Route table entry | S3, DynamoDB only | Free |
| **Interface** (PrivateLink) | ENI + private IP | 150+ AWS services | Hourly + per-GB |

### Gateway Endpoint
- Updates the route table: `pl-xxxxxx (S3 prefix list) → vpce-xxxx`
- No ENI, no DNS change, no SG needed
- Works only within the same region

### Interface Endpoint (PrivateLink)
- Creates an **ENI in your subnet** with a private IP
- Traffic resolves to `vpce-xxx.s3.us-east-1.vpce.amazonaws.com` via Route 53
- Needs a **VPC Endpoint Policy** (resource-based) to control access
- Needs **endpoint-specific SG** to control which instances can connect

---

## 🏗️ Architecture

```
Private Instance (10.10.2.x)
    │
    ├── S3 access via GATEWAY endpoint
    │     Route: pl-xxxxx (S3 prefixes) → vpce-gateway
    │     No ENI, no SG, free
    │
    └── SSM access via INTERFACE endpoints
          ENI in private subnet (10.10.2.x) → PrivateLink → SSM
          Requires 3 endpoints: ssm, ssmmessages, ec2messages
```

---

## 🧪 PoC

```bash
source ~/.vpc_lab_vars
```

### Step 1: Create a Gateway Endpoint for S3
```bash
S3_ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $PRIV_RT_1A $PRIV_RT_1B \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=vpce-s3-gateway}]' \
  --query 'VpcEndpoint.VpcEndpointId' --output text)

echo "S3 Gateway Endpoint: $S3_ENDPOINT_ID ✅"
echo "export S3_ENDPOINT_ID=$S3_ENDPOINT_ID" >> ~/.vpc_lab_vars
```

### Step 2: Verify the S3 prefix list route appears in the private route table
```bash
aws ec2 describe-route-tables --route-table-ids $PRIV_RT_1A \
  --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,DestinationPrefixListId,GatewayId,State]' \
  --output table

# You should see a new route:
# DestinationPrefixListId = pl-xxxxxxxx  GatewayId = vpce-xxxxxxxx
```

### Step 3: Restrict the S3 Gateway Endpoint policy (optional but recommended)
```bash
# By default the endpoint allows all S3 actions. Lock it down:
cat > /tmp/s3-endpoint-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::your-app-bucket",
        "arn:aws:s3:::your-app-bucket/*",
        "arn:aws:s3:::aws-ssm-*",
        "arn:aws:s3:::amazon-ssm-*",
        "arn:aws:s3:::aws-windows-downloads-*",
        "arn:aws:s3:::amazonlinux-*",
        "arn:aws:s3:::amazonlinux-2-repos-*"
      ]
    }
  ]
}
EOF

aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id $S3_ENDPOINT_ID \
  --policy-document file:///tmp/s3-endpoint-policy.json

echo "S3 endpoint policy applied ✅"
```

### Step 4: Create a Security Group for Interface Endpoints
```bash
SG_ENDPOINTS=$(aws ec2 create-security-group \
  --group-name sg-vpc-endpoints \
  --description "Allow HTTPS from VPC CIDR to interface endpoints" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

# Interface endpoints communicate over port 443
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ENDPOINTS \
  --protocol tcp --port 443 \
  --cidr 10.10.0.0/16

echo "VPC endpoint SG: $SG_ENDPOINTS ✅"
echo "export SG_ENDPOINTS=$SG_ENDPOINTS" >> ~/.vpc_lab_vars
```

### Step 5: Create Interface Endpoints for SSM (3 required)
```bash
# SSM requires THREE interface endpoints to function:
# 1. com.amazonaws.region.ssm
# 2. com.amazonaws.region.ssmmessages
# 3. com.amazonaws.region.ec2messages

for SERVICE in ssm ssmmessages ec2messages; do
  ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id $VPC_ID \
    --service-name com.amazonaws.us-east-1.${SERVICE} \
    --vpc-endpoint-type Interface \
    --subnet-ids $PRIV_SUBNET_1A $PRIV_SUBNET_1B \
    --security-group-ids $SG_ENDPOINTS \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=vpce-${SERVICE}}]" \
    --query 'VpcEndpoint.VpcEndpointId' --output text)
  echo "Interface endpoint for $SERVICE: $ENDPOINT_ID ✅"
done
```

### Step 6: Verify private DNS for SSM endpoint
```bash
# Interface endpoints create Route 53 private hosted zone entries
# Private DNS means: ssm.us-east-1.amazonaws.com resolves to the endpoint ENI IP

# Check endpoint details (DNS names)
aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=service-name,Values=com.amazonaws.us-east-1.ssm" \
  --query 'VpcEndpoints[0].{Id:VpcEndpointId,State:State,DNS:DnsEntries[0].DnsName}' \
  --output table
```

### Step 7: Launch a private instance and test SSM access (no internet needed)
```bash
# Create IAM role for SSM
aws iam create-role \
  --role-name SSMPrivateInstanceRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam attach-role-policy \
  --role-name SSMPrivateInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam create-instance-profile --instance-profile-name SSMPrivateInstanceProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name SSMPrivateInstanceProfile \
  --role-name SSMPrivateInstanceRole

sleep 10  # IAM propagation

AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
            "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

PRIV_INST=$(aws ec2 run-instances \
  --image-id $AMI_ID --instance-type t3.micro \
  --subnet-id $PRIV_SUBNET_1A \
  --security-group-ids $SG_APP \
  --iam-instance-profile Name=SSMPrivateInstanceProfile \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled" \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=private-ssm-test}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $PRIV_INST
echo "Private instance: $PRIV_INST ✅"

# Wait a minute for SSM agent to register
sleep 60

# Connect via SSM — no internet, no bastion, no public IP!
aws ssm start-session --target $PRIV_INST
# Once inside:
# aws s3 ls  ← works via gateway endpoint
# curl https://ssm.us-east-1.amazonaws.com  ← resolves to endpoint ENI IP
echo "export PRIV_INST=$PRIV_INST" >> ~/.vpc_lab_vars
```

### Step 8: Bucket Policy to Enforce VPC Endpoint Access
```bash
# Lock down an S3 bucket so it can ONLY be accessed from within the VPC
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/bucket-policy-endpoint-only.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonVPCEndpointAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::your-private-bucket",
        "arn:aws:s3:::your-private-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:SourceVpce": "${S3_ENDPOINT_ID}"
        }
      }
    }
  ]
}
EOF

# Apply to your bucket (replace bucket name)
# aws s3api put-bucket-policy \
#   --bucket your-private-bucket \
#   --policy file:///tmp/bucket-policy-endpoint-only.json
echo "Bucket policy template created (update bucket name and apply) ✅"
```

---

## ✅ Cleanup

```bash
source ~/.vpc_lab_vars

# Terminate private instance
aws ec2 terminate-instances --instance-ids $PRIV_INST
aws ec2 wait instance-terminated --instance-ids $PRIV_INST

# Delete all VPC endpoints in this VPC
ENDPOINT_IDS=$(aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" \
  --query 'VpcEndpoints[*].VpcEndpointId' --output text)
[ -n "$ENDPOINT_IDS" ] && aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $ENDPOINT_IDS
echo "VPC endpoints deleted ✅"

# Delete endpoint SG
aws ec2 delete-security-group --group-id $SG_ENDPOINTS

# Cleanup SSM role
aws iam remove-role-from-instance-profile \
  --instance-profile-name SSMPrivateInstanceProfile \
  --role-name SSMPrivateInstanceRole
aws iam delete-instance-profile --instance-profile-name SSMPrivateInstanceProfile
aws iam detach-role-policy \
  --role-name SSMPrivateInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam delete-role --role-name SSMPrivateInstanceRole

echo "All resources cleaned up ✅"
```

---

## ➡️ Next Lab
[Lab 07 — Transit Gateway →](../07-transit-gateway/README.md)
