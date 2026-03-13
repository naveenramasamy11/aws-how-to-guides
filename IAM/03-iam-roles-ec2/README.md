# Lab 03 — IAM Roles for EC2 (Instance Profiles) 🟡

## 🎯 Goal
Attach an IAM role to an EC2 instance so it can access AWS services (S3) without hardcoding access keys. Understand the EC2 Instance Metadata Service (IMDS) and how temporary credentials work.

---

## 🧠 Concepts

### Why Roles for EC2 — Not Access Keys
Hardcoding access keys in EC2 instances is a critical security anti-pattern:
- Keys can be leaked via code repos, logs, or snapshots
- Key rotation is manual and error-prone
- Keys have permanent access until explicitly revoked

IAM Roles solve this by:
- Providing **temporary credentials** (rotated every ~1 hour automatically)
- Using the **EC2 Instance Metadata Service (IMDS)** to deliver creds
- No key management overhead

### Instance Profile
An **Instance Profile** is a container for an IAM Role that can be associated with an EC2 instance. When you create a role for EC2 via Console, the instance profile is created automatically. Via CLI, you must create it separately.

### How Credentials Flow on EC2
```
EC2 Instance
    │
    ▼
IMDS (169.254.169.254)
    │  GET /latest/meta-data/iam/security-credentials/<role-name>
    ▼
Temporary Credentials
    ├── AccessKeyId
    ├── SecretAccessKey
    ├── Token (session token)
    └── Expiration (~1 hour, auto-rotated)
    │
    ▼
AWS SDK / CLI uses these automatically
```

---

## 🏗️ Architecture

```
          IAM Role: EC2-S3-ReadRole
          ├── Trust: ec2.amazonaws.com
          └── Policy: AmazonS3ReadOnlyAccess
                │
                ▼
     Instance Profile: EC2-S3-ReadRole-Profile
                │
                ▼
         EC2 Instance (t3.micro)
                │
         (calls IMDS for temp creds)
                │
                ▼
          S3 Bucket (read-only)
```

---

## 🧪 PoC

### Step 1: Create the IAM Role with EC2 Trust Policy
```bash
# Trust policy allowing EC2 to assume this role
cat > /tmp/ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
ROLE_ARN=$(aws iam create-role \
  --role-name EC2-S3-ReadRole \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json \
  --description "Allows EC2 instances to read from S3" \
  --max-session-duration 3600 \
  --tags Key=Purpose,Value=EC2S3Access \
  --query 'Role.Arn' \
  --output text)

echo "Created role: $ROLE_ARN ✅"
```

### Step 2: Attach Permissions Policy to the Role
```bash
# Attach AWS Managed policy
aws iam attach-role-policy \
  --role-name EC2-S3-ReadRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

echo "S3 ReadOnly policy attached ✅"

# Verify
aws iam list-attached-role-policies --role-name EC2-S3-ReadRole \
  --query 'AttachedPolicies[*].[PolicyName,PolicyArn]' --output table
```

### Step 3: Create the Instance Profile
```bash
# Create instance profile (same name as role for clarity)
aws iam create-instance-profile \
  --instance-profile-name EC2-S3-ReadRole-Profile

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-ReadRole-Profile \
  --role-name EC2-S3-ReadRole

echo "Instance profile created and role attached ✅"

# Verify
aws iam get-instance-profile --instance-profile-name EC2-S3-ReadRole-Profile
```

### Step 4: Launch EC2 Instance with the Instance Profile
```bash
# Get latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
            "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

echo "Using AMI: $AMI_ID"

# Get default VPC and subnet
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0].SubnetId' --output text)

echo "VPC: $VPC_ID, Subnet: $SUBNET_ID"

# Launch instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t3.micro \
  --iam-instance-profile Name=EC2-S3-ReadRole-Profile \
  --subnet-id $SUBNET_ID \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=iam-role-poc}]' \
  --user-data '#!/bin/bash
echo "Testing S3 access via IAM Role" > /tmp/test.log
aws s3 ls >> /tmp/test.log 2>&1
echo "Done" >> /tmp/test.log' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Launched instance: $INSTANCE_ID ✅"

# Wait for it to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
echo "Instance is running ✅"
```

### Step 5: Verify credentials from IMDS (SSM Session or userdata)
If you have SSM access to the instance, run:
```bash
# Connect via SSM (no SSH key required)
aws ssm start-session --target $INSTANCE_ID

# Inside the instance shell, run:
# Check what role the instance has
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Get the actual temporary credentials
ROLE_NAME=$(curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/)
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME | python3 -m json.tool

# IMDSv2 (more secure, token-based)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Test S3 access (should work via role)
aws s3 ls

# Test EC2 describe (should be denied - role only has S3)
aws ec2 describe-instances --region us-east-1
# Error: User is not authorized to perform: ec2:DescribeInstances
```

### Step 6: Enforce IMDSv2 (Security Best Practice)
```bash
# Modify existing instance to require IMDSv2
aws ec2 modify-instance-metadata-options \
  --instance-id $INSTANCE_ID \
  --http-tokens required \
  --http-endpoint enabled

echo "IMDSv2 enforced on instance ✅"

# Launch new instances with IMDSv2 enforced from the start
# Add to run-instances command:
# --metadata-options "HttpTokens=required,HttpEndpoint=enabled"
```

### Step 7: Update role on a running instance (hot-swap)
```bash
# Disassociate current profile
ASSOC_ID=$(aws ec2 describe-iam-instance-profile-associations \
  --filters "Name=instance-id,Values=$INSTANCE_ID" \
  --query 'IamInstanceProfileAssociations[0].AssociationId' \
  --output text)

# Replace with a different profile (if needed)
# aws ec2 replace-iam-instance-profile-association \
#   --association-id $ASSOC_ID \
#   --iam-instance-profile Name=NewProfileName
```

---

## ✅ Cleanup

```bash
# Terminate EC2 instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID
echo "Instance terminated ✅"

# Remove role from instance profile
aws iam remove-role-from-instance-profile \
  --instance-profile-name EC2-S3-ReadRole-Profile \
  --role-name EC2-S3-ReadRole

# Delete instance profile
aws iam delete-instance-profile \
  --instance-profile-name EC2-S3-ReadRole-Profile

# Detach policy from role
aws iam detach-role-policy \
  --role-name EC2-S3-ReadRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Delete role
aws iam delete-role --role-name EC2-S3-ReadRole

echo "All IAM resources cleaned up ✅"
```

---

## 🔒 Security Best Practices
1. **Always use roles, never access keys on EC2**
2. **Enforce IMDSv2** (`--http-tokens required`) to prevent SSRF attacks
3. **Scope down permissions** — avoid wildcards in Resource
4. **Use VPC endpoints** for S3/DynamoDB to avoid traffic leaving the VPC
5. **Enable CloudTrail** to audit all API calls made by the role

---

## ➡️ Next Lab
[Lab 04 — IAM Roles for Lambda →](../04-iam-roles-lambda/README.md)
