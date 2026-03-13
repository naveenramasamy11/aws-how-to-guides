# Lab 02 — IAM Users, Groups & Policies 🟢

## 🎯 Goal
Create IAM users, organize them into groups, attach managed and custom policies, and enforce least-privilege access. Enable MFA programmatically.

---

## 🧠 Concepts

### Least Privilege Principle
Grant only the permissions required to perform a task — nothing more. Start with deny-all and explicitly allow what's needed.

### Managed vs Inline Policies
| | AWS Managed | Customer Managed | Inline |
|--|------------|-----------------|--------|
| **Owner** | AWS | You | You (embedded) |
| **Reusable** | Yes | Yes | No (1:1 with entity) |
| **Versioning** | AWS handles | You handle | No versioning |
| **Best for** | Common use cases | Custom org needs | Strict 1:1 binding |

### Password Policy
Account-level settings enforcing complexity, rotation, and reuse rules for IAM user console passwords.

---

## 🏗️ Architecture

```
AWS Account
└── IAM
    ├── Groups
    │   ├── developers    ← S3 Read + EC2 Describe
    │   └── admins        ← AdministratorAccess
    ├── Users
    │   ├── dev-alice     ← member of: developers
    │   └── dev-bob       ← member of: developers
    └── Policies
        └── DeveloperPolicy (Customer Managed)
            ├── s3:Get*, s3:List* on all buckets
            └── ec2:Describe* on all resources
```

---

## 🧪 PoC

### Step 1: Set up account password policy
```bash
aws iam update-account-password-policy \
  --minimum-password-length 14 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 5 \
  --hard-expiry false

echo "Password policy updated ✅"

# Verify
aws iam get-account-password-policy
```

### Step 2: Create IAM Groups
```bash
# Create developers group
aws iam create-group --group-name developers
echo "Group 'developers' created ✅"

# Create admins group
aws iam create-group --group-name admins
echo "Group 'admins' created ✅"
```

### Step 3: Attach AWS Managed Policy to admins group
```bash
aws iam attach-group-policy \
  --group-name admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

echo "AdministratorAccess attached to admins ✅"
```

### Step 4: Create a Customer Managed Policy for developers
```bash
# Save the policy document
cat > /tmp/developer-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EC2DescribeAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "IAMReadSelf",
      "Effect": "Allow",
      "Action": [
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ListAccessKeys",
        "iam:ChangePassword"
      ],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    }
  ]
}
EOF

# Create the policy
POLICY_ARN=$(aws iam create-policy \
  --policy-name DeveloperPolicy \
  --description "Read access to S3 and EC2 describe for developers" \
  --policy-document file:///tmp/developer-policy.json \
  --query 'Policy.Arn' \
  --output text)

echo "Created policy: $POLICY_ARN ✅"
```

### Step 5: Attach the Customer Managed Policy to developers group
```bash
aws iam attach-group-policy \
  --group-name developers \
  --policy-arn $POLICY_ARN

echo "DeveloperPolicy attached to developers group ✅"
```

### Step 6: Create IAM Users
```bash
# Create dev-alice
aws iam create-user --user-name dev-alice \
  --tags Key=Team,Value=backend Key=Environment,Value=dev
echo "User dev-alice created ✅"

# Create dev-bob
aws iam create-user --user-name dev-bob \
  --tags Key=Team,Value=frontend Key=Environment,Value=dev
echo "User dev-bob created ✅"
```

### Step 7: Add users to group
```bash
aws iam add-user-to-group --user-name dev-alice --group-name developers
aws iam add-user-to-group --user-name dev-bob --group-name developers
echo "Users added to developers group ✅"

# Verify group membership
aws iam get-group --group-name developers \
  --query 'Users[*].UserName' --output table
```

### Step 8: Create access keys for programmatic access
```bash
# Create access key for dev-alice (store output securely!)
aws iam create-access-key --user-name dev-alice \
  --query 'AccessKey.{AccessKeyId:AccessKeyId,SecretAccessKey:SecretAccessKey}' \
  --output json
```
> ⚠️ **Security Note:** Save the SecretAccessKey immediately — it's shown only once!

### Step 9: Create a console login profile
```bash
aws iam create-login-profile \
  --user-name dev-alice \
  --password 'Temp@Pass123!' \
  --password-reset-required

echo "Console access created for dev-alice (password reset required on first login) ✅"
```

### Step 10: Test the policy with IAM Policy Simulator
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Simulate what dev-alice can do
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::${ACCOUNT_ID}:user/dev-alice \
  --action-names s3:GetObject s3:DeleteObject ec2:DescribeInstances ec2:TerminateInstances \
  --resource-arns "*" \
  --query 'EvaluationResults[*].[ActionName,EvalDecision]' \
  --output table
```

Expected output:
```
---------------------------------------------------
|           SimulatePrincipalPolicy               |
+----------------------+--------------------------+
|  s3:GetObject        | allowed                  |
|  s3:DeleteObject     | implicitDeny             |
|  ec2:DescribeInstances | allowed                |
|  ec2:TerminateInstances | implicitDeny          |
+----------------------+--------------------------+
```

### Step 11: Add an inline deny policy to a specific user (optional)
Inline policies are useful for exceptions — e.g., block a specific user from a specific action even though their group allows it:

```bash
cat > /tmp/deny-us-east-2.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUSEast2",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-2"
        }
      }
    }
  ]
}
EOF

aws iam put-user-policy \
  --user-name dev-bob \
  --policy-name DenyUSEast2 \
  --policy-document file:///tmp/deny-us-east-2.json

echo "Inline deny policy attached to dev-bob ✅"
```

---

## ✅ Cleanup

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
POLICY_ARN="arn:aws:iam::${ACCOUNT_ID}:policy/DeveloperPolicy"

# Remove users from group
aws iam remove-user-from-group --user-name dev-alice --group-name developers
aws iam remove-user-from-group --user-name dev-bob --group-name developers

# Delete inline policy
aws iam delete-user-policy --user-name dev-bob --policy-name DenyUSEast2

# Delete login profiles
aws iam delete-login-profile --user-name dev-alice

# Delete access keys (list first)
KEY_ID=$(aws iam list-access-keys --user-name dev-alice --query 'AccessKeyMetadata[0].AccessKeyId' --output text)
[ "$KEY_ID" != "None" ] && aws iam delete-access-key --user-name dev-alice --access-key-id $KEY_ID

# Delete users
aws iam delete-user --user-name dev-alice
aws iam delete-user --user-name dev-bob

# Detach & delete policies
aws iam detach-group-policy --group-name developers --policy-arn $POLICY_ARN
aws iam detach-group-policy --group-name admins --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam delete-policy --policy-arn $POLICY_ARN

# Delete groups
aws iam delete-group --group-name developers
aws iam delete-group --group-name admins

echo "All resources cleaned up ✅"
```

---

## ➡️ Next Lab
[Lab 03 — IAM Roles for EC2 →](../03-iam-roles-ec2/README.md)
