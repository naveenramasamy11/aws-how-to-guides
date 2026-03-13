# Lab 07 — ABAC: Attribute-Based Access Control 🔴

## 🎯 Goal
Use **resource tags and principal tags** to dynamically control access — so a single IAM policy grants the right permissions to the right people without managing per-user or per-role policies.

---

## 🧠 Concepts

### ABAC vs RBAC

| | RBAC (Role-Based) | ABAC (Attribute-Based) |
|--|------------------|----------------------|
| **Access defined by** | Role/group membership | Tags on principal + resource |
| **Policy count** | One policy per role | One policy for all users |
| **Scaling** | Grows with team size | Scales automatically via tags |
| **New hire** | Create user, assign role | Create user, tag it — done |
| **Best for** | Static, well-defined roles | Dynamic, growing teams |

### Key IAM Condition Keys for ABAC

| Condition Key | Meaning |
|--------------|---------|
| `aws:PrincipalTag/<key>` | Tag on the IAM user/role making the request |
| `aws:ResourceTag/<key>` | Tag on the AWS resource being accessed |
| `aws:RequestTag/<key>` | Tag included in a create/tag API request |
| `aws:TagKeys` | Set of tag keys in the request |

### How ABAC Works
```
IAM User (dev-alice)                 S3 Bucket (app-data)
  Tags:                                Tags:
    Team = backend                       Team = backend  ✅ match → ALLOW
    Env  = dev                           Env  = dev      ✅ match → ALLOW

IAM User (dev-bob)                   S3 Bucket (app-data)
  Tags:                                Tags:
    Team = frontend                      Team = backend  ❌ mismatch → DENY
    Env  = dev                           Env  = dev
```

---

## 🏗️ Architecture

```
IAM Policy: ABACDeveloperPolicy (ONE policy for all devs)
  └── Allow s3:GetObject, s3:PutObject IF
        aws:PrincipalTag/Team == aws:ResourceTag/Team
        AND aws:PrincipalTag/Env == aws:ResourceTag/Env

Users:
  alice → Tags: Team=backend, Env=dev
  bob   → Tags: Team=frontend, Env=dev
  carol → Tags: Team=backend, Env=prod

S3 Buckets:
  backend-dev-bucket  → Tags: Team=backend, Env=dev   ← alice can access
  frontend-dev-bucket → Tags: Team=frontend, Env=dev  ← bob can access
  backend-prod-bucket → Tags: Team=backend, Env=prod  ← carol can access
```

---

## 🧪 PoC

### Step 1: Create the ABAC Policy (single policy for all devs)
```bash
cat > /tmp/abac-developer-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ABACAllowS3SameTeamAndEnv",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}",
          "aws:ResourceTag/Env":  "${aws:PrincipalTag/Env}"
        }
      }
    },
    {
      "Sid": "ABACAllowListAllBuckets",
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Sid": "ABACAllowEC2SameTeam",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}",
          "aws:ResourceTag/Env":  "${aws:PrincipalTag/Env}"
        }
      }
    },
    {
      "Sid": "ABACAllowTaggingOwnResources",
      "Effect": "Allow",
      "Action": [
        "s3:PutBucketTagging",
        "ec2:CreateTags"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}",
          "aws:ResourceTag/Env":  "${aws:PrincipalTag/Env}"
        },
        "ForAllValues:StringEquals": {
          "aws:TagKeys": ["Name", "Project", "CostCenter"]
        }
      }
    },
    {
      "Sid": "DenyChangingTeamOrEnvTags",
      "Effect": "Deny",
      "Action": [
        "s3:PutBucketTagging",
        "ec2:CreateTags",
        "ec2:DeleteTags"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringEquals": {
          "aws:TagKeys": ["Team", "Env"]
        }
      }
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

ABAC_POLICY_ARN=$(aws iam create-policy \
  --policy-name ABACDeveloperPolicy \
  --description "ABAC policy: grants access to resources matching user tags (Team+Env)" \
  --policy-document file:///tmp/abac-developer-policy.json \
  --tags Key=Purpose,Value=ABAC \
  --query 'Policy.Arn' \
  --output text)

echo "Created ABAC policy: $ABAC_POLICY_ARN ✅"
```

### Step 2: Create IAM Users with Team/Env Tags
```bash
# Alice — backend developer, dev environment
aws iam create-user \
  --user-name alice \
  --tags Key=Team,Value=backend Key=Env,Value=dev Key=Role,Value=developer

# Bob — frontend developer, dev environment
aws iam create-user \
  --user-name bob \
  --tags Key=Team,Value=frontend Key=Env,Value=dev Key=Role,Value=developer

# Carol — backend developer, prod environment
aws iam create-user \
  --user-name carol \
  --tags Key=Team,Value=backend Key=Env,Value=prod Key=Role,Value=developer

echo "Users alice, bob, carol created ✅"

# Verify tags
for user in alice bob carol; do
  echo "--- $user ---"
  aws iam list-user-tags --user-name $user \
    --query 'Tags[*].[Key,Value]' --output table
done
```

### Step 3: Attach the SINGLE ABAC policy to all three users
```bash
for user in alice bob carol; do
  aws iam attach-user-policy \
    --user-name $user \
    --policy-arn $ABAC_POLICY_ARN
  echo "ABACDeveloperPolicy attached to $user ✅"
done
```

### Step 4: Create S3 Buckets with matching tags
```bash
# Bucket for backend team, dev env
aws s3api create-bucket \
  --bucket "backend-dev-data-${ACCOUNT_ID}" \
  --region us-east-1

aws s3api put-bucket-tagging \
  --bucket "backend-dev-data-${ACCOUNT_ID}" \
  --tagging 'TagSet=[{Key=Team,Value=backend},{Key=Env,Value=dev}]'

echo "backend-dev bucket created with tags ✅"

# Bucket for frontend team, dev env
aws s3api create-bucket \
  --bucket "frontend-dev-data-${ACCOUNT_ID}" \
  --region us-east-1

aws s3api put-bucket-tagging \
  --bucket "frontend-dev-data-${ACCOUNT_ID}" \
  --tagging 'TagSet=[{Key=Team,Value=frontend},{Key=Env,Value=dev}]'

echo "frontend-dev bucket created with tags ✅"

# Bucket for backend team, prod env
aws s3api create-bucket \
  --bucket "backend-prod-data-${ACCOUNT_ID}" \
  --region us-east-1

aws s3api put-bucket-tagging \
  --bucket "backend-prod-data-${ACCOUNT_ID}" \
  --tagging 'TagSet=[{Key=Team,Value=backend},{Key=Env,Value=prod}]'

echo "backend-prod bucket created with tags ✅"
```

### Step 5: Simulate access for each user (Policy Simulator)
```bash
# Test: alice (backend/dev) accessing backend-dev bucket → ALLOW
aws iam simulate-principal-policy \
  --policy-source-arn "arn:aws:iam::${ACCOUNT_ID}:user/alice" \
  --action-names s3:GetObject s3:PutObject \
  --resource-arns "arn:aws:s3:::backend-dev-data-${ACCOUNT_ID}/*" \
  --context-entries \
    "ContextKeyName=aws:PrincipalTag/Team,ContextKeyValues=backend,ContextKeyType=string" \
    "ContextKeyName=aws:PrincipalTag/Env,ContextKeyValues=dev,ContextKeyType=string" \
    "ContextKeyName=aws:ResourceTag/Team,ContextKeyValues=backend,ContextKeyType=string" \
    "ContextKeyName=aws:ResourceTag/Env,ContextKeyValues=dev,ContextKeyType=string" \
  --query 'EvaluationResults[*].[ActionName,EvalDecision]' \
  --output table
# Expected: ALLOW ✅

echo "---"

# Test: alice (backend/dev) accessing frontend-dev bucket → DENY
aws iam simulate-principal-policy \
  --policy-source-arn "arn:aws:iam::${ACCOUNT_ID}:user/alice" \
  --action-names s3:GetObject \
  --resource-arns "arn:aws:s3:::frontend-dev-data-${ACCOUNT_ID}/*" \
  --context-entries \
    "ContextKeyName=aws:PrincipalTag/Team,ContextKeyValues=backend,ContextKeyType=string" \
    "ContextKeyName=aws:PrincipalTag/Env,ContextKeyValues=dev,ContextKeyType=string" \
    "ContextKeyName=aws:ResourceTag/Team,ContextKeyValues=frontend,ContextKeyType=string" \
    "ContextKeyName=aws:ResourceTag/Env,ContextKeyValues=dev,ContextKeyType=string" \
  --query 'EvaluationResults[*].[ActionName,EvalDecision]' \
  --output table
# Expected: DENY (Team mismatch: backend != frontend) ✅

echo "---"

# Test: alice (backend/dev) accessing backend-prod bucket → DENY
aws iam simulate-principal-policy \
  --policy-source-arn "arn:aws:iam::${ACCOUNT_ID}:user/alice" \
  --action-names s3:GetObject \
  --resource-arns "arn:aws:s3:::backend-prod-data-${ACCOUNT_ID}/*" \
  --context-entries \
    "ContextKeyName=aws:PrincipalTag/Team,ContextKeyValues=backend,ContextKeyType=string" \
    "ContextKeyName=aws:PrincipalTag/Env,ContextKeyValues=dev,ContextKeyType=string" \
    "ContextKeyName=aws:ResourceTag/Team,ContextKeyValues=backend,ContextKeyType=string" \
    "ContextKeyName=aws:ResourceTag/Env,ContextKeyValues=prod,ContextKeyType=string" \
  --query 'EvaluationResults[*].[ActionName,EvalDecision]' \
  --output table
# Expected: DENY (Env mismatch: dev != prod) ✅
```

### Step 6: ABAC with IAM Roles (for services like Lambda/EC2)
```bash
# Create a role for the backend dev-env Lambda function
cat > /tmp/lambda-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name backend-dev-lambda-role \
  --assume-role-policy-document file:///tmp/lambda-trust.json \
  --tags Key=Team,Value=backend Key=Env,Value=dev \
  --description "Lambda role for backend/dev — ABAC controlled"

aws iam attach-role-policy \
  --role-name backend-dev-lambda-role \
  --policy-arn $ABAC_POLICY_ARN

echo "ABAC Lambda role created for backend/dev ✅"
# This role will ONLY access backend+dev tagged resources automatically
```

### Step 7: Enforce tags on new resource creation (ABAC for Create)
```bash
# Policy addition: only allow creating EC2 instances if they include required tags
cat > /tmp/enforce-tags-on-create.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireTagsOnEC2Create",
      "Effect": "Allow",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/Team": "${aws:PrincipalTag/Team}",
          "aws:RequestTag/Env":  "${aws:PrincipalTag/Env}"
        },
        "ForAllValues:StringEquals": {
          "aws:TagKeys": ["Team", "Env", "Name"]
        }
      }
    },
    {
      "Sid": "AllowOtherEC2Resources",
      "Effect": "Allow",
      "Action": "ec2:RunInstances",
      "Resource": [
        "arn:aws:ec2:*:*:subnet/*",
        "arn:aws:ec2:*:*:network-interface/*",
        "arn:aws:ec2:*:*:security-group/*",
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*::image/*",
        "arn:aws:ec2:*:*:key-pair/*"
      ]
    }
  ]
}
EOF

ENFORCE_TAG_POLICY_ARN=$(aws iam create-policy \
  --policy-name EnforceTagsOnCreate \
  --description "ABAC: enforce Team+Env tags on new EC2 instances" \
  --policy-document file:///tmp/enforce-tags-on-create.json \
  --query 'Policy.Arn' \
  --output text)

echo "Created tag-enforcement policy: $ENFORCE_TAG_POLICY_ARN ✅"
```

---

## 📊 ABAC Tag Strategy — Best Practices

```
Recommended Tag Schema:
┌──────────────┬────────────────────────────────────┐
│ Tag Key      │ Example Values                     │
├──────────────┼────────────────────────────────────┤
│ Team         │ backend, frontend, platform, data  │
│ Env          │ dev, staging, prod                 │
│ Project      │ payment-service, auth-api          │
│ CostCenter   │ cc-1001, cc-1002                   │
│ Owner        │ alice@company.com                  │
└──────────────┴────────────────────────────────────┘
```

---

## ✅ Cleanup

```bash
# Delete S3 buckets
for bucket in backend-dev-data frontend-dev-data backend-prod-data; do
  aws s3 rb "s3://${bucket}-${ACCOUNT_ID}" --force 2>/dev/null && \
    echo "Deleted bucket: ${bucket}-${ACCOUNT_ID}"
done

# Detach policies and delete users
for user in alice bob carol; do
  aws iam detach-user-policy --user-name $user --policy-arn $ABAC_POLICY_ARN
  aws iam delete-user --user-name $user
  echo "Deleted user: $user"
done

# Cleanup role
aws iam detach-role-policy \
  --role-name backend-dev-lambda-role \
  --policy-arn $ABAC_POLICY_ARN
aws iam delete-role --role-name backend-dev-lambda-role

# Delete policies
aws iam delete-policy --policy-arn $ABAC_POLICY_ARN
aws iam delete-policy --policy-arn $ENFORCE_TAG_POLICY_ARN

echo "All ABAC resources cleaned up ✅"
```

---

## 🔒 Security Best Practices
1. **Treat tags as security controls** — use SCPs to prevent users from modifying Team/Env tags
2. **Enforce tags at creation time** with `aws:RequestTag` conditions
3. **Combine ABAC + RBAC** — ABAC for resource access, RBAC for service-level permissions
4. **Audit with AWS Config** — rule `required-tags` ensures compliance
5. **Use IAM Access Analyzer** to detect overly permissive ABAC policies

---

## 🎓 Congratulations!
You've completed the full IAM learning path:

| Lab | Topic | Status |
|-----|-------|--------|
| 01 | IAM Fundamentals | ✅ |
| 02 | Users, Groups & Policies | ✅ |
| 03 | IAM Roles for EC2 | ✅ |
| 04 | IAM Roles for Lambda | ✅ |
| 05 | Cross-Account Access | ✅ |
| 06 | Permission Boundaries | ✅ |
| 07 | ABAC | ✅ |

**Next up:** VPC, EC2 deep-dive, S3 security, AWS Organizations & SCPs.
