# Lab 05 — IAM Roles, Security Groups & SSM Session Manager

🔴 **Difficulty:** Advanced | **Time:** 60 min | **Cost:** ~$0.02 (t3.micro, ~20 min)

---

## Goal

Apply EC2 security best practices end-to-end: attach a least-privilege IAM instance role, enforce IMDSv2, remove all inbound SSH rules, and connect using SSM Session Manager instead of a key pair.

---

## Concepts

### IAM Instance Profile vs IAM Role

```
IAM Role (ec2-lab05-role)
    └── Trust Policy: "ec2.amazonaws.com may assume this role"
    └── Permission Policies (what the role can do)
              │
              │  wrapped in
              ▼
IAM Instance Profile (ec2-lab05-profile)
              │
              │  attached to
              ▼
       EC2 Instance
              │
              │  fetches temporary credentials via
              ▼
       IMDSv2 endpoint
   http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>
```

### IMDSv2 — Why and How

| Version | Token Required | Risk |
|---------|---------------|------|
| IMDSv1 | No — any process can GET metadata | SSRF attacks can steal instance credentials |
| IMDSv2 | Yes — PUT request required first | SSRF cannot complete PUT; credentials protected |

Always enforce IMDSv2 on new instances via `--metadata-options HttpTokens=required`.

### SSM Session Manager vs SSH

| Aspect | SSH | SSM Session Manager |
|--------|-----|-------------------|
| Inbound port | TCP 22 open | No inbound rules required |
| Key management | PEM file required | No key pair needed |
| Audit trail | None by default | Full session logs to S3/CloudWatch |
| Network path | Public IP required | Outbound HTTPS only via SSM endpoints |
| IAM control | OS user + key | IAM policy (`ssm:StartSession`) |

### Least-Privilege Security Group Pattern

```
Inbound:   NONE (zero rules — no SSH, no HTTP open to world)
Outbound:  HTTPS 443 → 0.0.0.0/0  (for SSM endpoints and package updates)
           HTTP  80  → 0.0.0.0/0  (for yum/dnf repos)
```

---

## Architecture

```
Your Laptop
    │  aws ssm start-session
    ▼
SSM Service (AWS-managed)
    │  HTTPS outbound from instance
    ▼
EC2 Instance (NO public SSH port open)
    │
    ├── IAM Role: ec2-lab05-role
    │   └── AmazonSSMManagedInstanceCore
    │   └── Custom S3 read policy (example)
    │
    ├── IMDSv2 enforced
    │
    └── Security Group: outbound 443/80 only (no inbound)
```

---

## PoC Steps

### Step 1 — Create the IAM role and instance profile

```bash
export AWS_DEFAULT_REGION=us-east-1

# Trust policy document
cat > /tmp/ec2-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name ec2-lab05-role \
  --assume-role-policy-document file:///tmp/ec2-trust.json \
  --description "EC2 Lab05 role for SSM + least-privilege access" \
  --tags Key=Lab,Value=EC2-05

# Attach AWS-managed policy for SSM Session Manager
aws iam attach-role-policy \
  --role-name ec2-lab05-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# Add a least-privilege inline policy: read-only access to a specific S3 prefix
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > /tmp/ec2-s3-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "ReadLabBucket",
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::ec2-lab05-${ACCOUNT_ID}",
      "arn:aws:s3:::ec2-lab05-${ACCOUNT_ID}/*"
    ]
  }]
}
EOF

aws iam put-role-policy \
  --role-name ec2-lab05-role \
  --policy-name ec2-lab05-s3-readonly \
  --policy-document file:///tmp/ec2-s3-policy.json

# Create the instance profile and attach the role
aws iam create-instance-profile \
  --instance-profile-name ec2-lab05-profile \
  --tags Key=Lab,Value=EC2-05

aws iam add-role-to-instance-profile \
  --instance-profile-name ec2-lab05-profile \
  --role-name ec2-lab05-role

echo "IAM role and profile ready"
sleep 10  # Allow IAM to propagate
```

### Step 2 — Create a restrictive security group (no inbound rules)

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)

SG_ID=$(aws ec2 create-security-group \
  --group-name "ec2-lab05-sg" \
  --description "Lab 05 - no inbound, SSM only" \
  --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ec2-lab05-sg},{Key=Lab,Value=EC2-05}]" \
  --query 'GroupId' --output text)

# Remove the default outbound "allow all" rule
aws ec2 revoke-security-group-egress \
  --group-id "$SG_ID" \
  --protocol -1 \
  --port -1 \
  --cidr 0.0.0.0/0

# Allow only HTTPS and HTTP outbound (SSM endpoints + package repos)
aws ec2 authorize-security-group-egress \
  --group-id "$SG_ID" \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-egress \
  --group-id "$SG_ID" \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

echo "Security group $SG_ID created (no inbound, limited outbound)"

# Verify there are zero inbound rules
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --query 'SecurityGroups[0].IpPermissions' \
  --output json
# Expected: []
```

### Step 3 — Launch an instance with IMDSv2 enforced and no key pair

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query 'Subnets[0].SubnetId' --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --iam-instance-profile Name=ec2-lab05-profile \
  --metadata-options "HttpTokens=required,HttpPutResponseHopLimit=1,HttpEndpoint=enabled" \
  --no-associate-public-ip-address \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=ec2-lab05},{Key=Lab,Value=EC2-05}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Instance: $INSTANCE_ID (no key pair, no public IP)"
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

# Verify IMDSv2 is enforced
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].MetadataOptions' \
  --output table
```

Expected output:
```
--------------------------------------------
|         MetadataOptions                  |
+----------------------+-------------------+
|  HttpEndpoint        |  enabled          |
|  HttpPutResponseHopLimit | 1             |
|  HttpTokens          |  required         |
|  State               |  applied          |
+----------------------+-------------------+
```

### Step 4 — Connect via SSM Session Manager (no SSH!)

```bash
# Install Session Manager plugin if not already installed
# On Amazon Linux / EC2:  sudo dnf install -y session-manager-plugin
# On macOS: brew install --cask session-manager-plugin

# Wait for SSM agent registration (may take 1-2 min after instance start)
echo "Waiting for SSM registration..."
for i in {1..12}; do
  STATUS=$(aws ssm describe-instance-information \
    --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
    --query 'InstanceInformationList[0].PingStatus' \
    --output text 2>/dev/null)
  [ "$STATUS" = "Online" ] && echo "SSM Online!" && break
  echo "  Attempt $i: $STATUS (waiting 10s)"
  sleep 10
done

# Start an interactive session
aws ssm start-session --target "$INSTANCE_ID"
```

Inside the SSM session:

```bash
# Confirm you are on the instance (no SSH required)
whoami
hostname
curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300" | head -c 20
# This should return a token string -- confirming IMDSv2 works

# Try IMDSv1 (should FAIL with 401 because HttpTokens=required)
curl -s -o /dev/null -w "%{http_code}" \
  http://169.254.169.254/latest/meta-data/instance-id
# Expected: 401

# Check the IAM role credentials available on the instance
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
ROLE_NAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/)
echo "Role on instance: $ROLE_NAME"
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE_NAME" \
  | python3 -m json.tool | grep -E "Type|Expiration"

exit
```

Expected output (inside SSM session):
```
Role on instance: ec2-lab05-role
    "Type": "AWS-HMAC",
    "Expiration": "2026-03-15T12:05:00Z",
```

### Step 5 — Verify least-privilege: test allowed and denied actions

```bash
# Start a new SSM session
aws ssm start-session --target "$INSTANCE_ID"
```

Inside:

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
ROLE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/)
CREDS=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE")

export AWS_ACCESS_KEY_ID=$(echo "$CREDS" | python3 -c "import sys,json;print(json.load(sys.stdin)['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo "$CREDS" | python3 -c "import sys,json;print(json.load(sys.stdin)['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo "$CREDS" | python3 -c "import sys,json;print(json.load(sys.stdin)['Token'])")

# This SHOULD succeed (SSM action allowed by AmazonSSMManagedInstanceCore)
aws ssm describe-instance-information --region us-east-1 2>&1 | head -5

# This SHOULD FAIL (EC2 describe not in the policy)
aws ec2 describe-instances --region us-east-1 2>&1 | head -3
# Expected: An error occurred (UnauthorizedOperation) ...

exit
```

---

## Cleanup

```bash
# Terminate instance
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"

# Delete security group
aws ec2 delete-security-group --group-id "$SG_ID"

# Detach policies and delete IAM objects
aws iam detach-role-policy \
  --role-name ec2-lab05-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

aws iam delete-role-policy \
  --role-name ec2-lab05-role \
  --policy-name ec2-lab05-s3-readonly

aws iam remove-role-from-instance-profile \
  --instance-profile-name ec2-lab05-profile \
  --role-name ec2-lab05-role

aws iam delete-instance-profile --instance-profile-name ec2-lab05-profile
aws iam delete-role --role-name ec2-lab05-role

echo "Cleanup complete"
```

---

## Key Takeaways

- Use IAM instance profiles instead of hardcoded credentials — credentials rotate automatically
- Always enforce IMDSv2 (`HttpTokens=required`) to defend against SSRF-based credential theft
- SSM Session Manager eliminates the need to open inbound port 22 — audit trail included for free
- Least-privilege: grant only the exact actions and resources the instance needs
- Test denied actions to confirm your policies are restrictive as intended

---

➡️ Next: [Lab 06 — Integration Pipeline: ALB + ASG + Lambda + SNS](../06-integration-pipeline/README.md)
