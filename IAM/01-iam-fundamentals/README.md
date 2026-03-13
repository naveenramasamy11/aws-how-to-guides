# Lab 01 — IAM Fundamentals 🟢

## 🎯 Goal
Understand the core IAM building blocks — Users, Groups, Roles, and Policies — by exploring them via the AWS CLI and Console.

---

## 🧠 Concepts

### IAM Users
A **user** is a permanent identity in your AWS account. It can have:
- **Console password** — for AWS Management Console
- **Access keys** — for programmatic (CLI/API) access
- **MFA device** — for added security

### IAM Groups
A **group** is a logical collection of users. Policies attached to a group apply to all its members. Groups cannot be nested (no group inside a group).

### IAM Roles
A **role** is a temporary identity with no long-term credentials. It's *assumed* by:
- AWS services (EC2, Lambda, ECS)
- Other AWS accounts
- Federated users (SAML/OIDC)
- Users in the same account (role chaining)

### IAM Policies
A **policy** is a JSON document defining permissions. Types:
| Type | Description |
|------|-------------|
| AWS Managed | Pre-built by AWS (`AdministratorAccess`, `ReadOnlyAccess`) |
| Customer Managed | Custom policies you create and own |
| Inline | Embedded directly into a single user/group/role |
| Resource-Based | Attached to a resource (e.g., S3 bucket policy) |

### ARN Format
```
arn:partition:service:region:account-id:resource-type/resource-id

# Examples
arn:aws:iam::123456789012:user/alice
arn:aws:iam::123456789012:role/MyLambdaRole
arn:aws:iam::aws:policy/AdministratorAccess  ← AWS Managed (no account-id)
```

---

## 🏗️ Architecture

```
AWS Account (123456789012)
├── IAM Users
│   ├── alice (Developer)
│   └── bob (Developer)
├── IAM Groups
│   └── Developers
│       └── Policy: ReadOnlyAccess
├── IAM Roles
│   └── EC2-S3-ReadRole
│       ├── Trust Policy: ec2.amazonaws.com
│       └── Permission Policy: AmazonS3ReadOnlyAccess
└── IAM Policies
    ├── AWS Managed: AdministratorAccess
    └── Customer Managed: MyCustomPolicy
```

---

## 🧪 PoC — Explore IAM via CLI

### Step 1: Verify your identity
```bash
aws sts get-caller-identity
```
**Expected output:**
```json
{
    "UserId": "AIDA...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

### Step 2: List IAM users in your account
```bash
aws iam list-users --query 'Users[*].[UserName,CreateDate,PasswordLastUsed]' --output table
```

### Step 3: List all IAM roles
```bash
aws iam list-roles --query 'Roles[*].[RoleName,CreateDate]' --output table
```

### Step 4: View a specific AWS Managed policy
```bash
# Get the policy ARN
POLICY_ARN="arn:aws:iam::aws:policy/ReadOnlyAccess"

# Get policy metadata
aws iam get-policy --policy-arn $POLICY_ARN

# Get the actual policy document
VERSION=$(aws iam get-policy --policy-arn $POLICY_ARN --query 'Policy.DefaultVersionId' --output text)
aws iam get-policy-version --policy-arn $POLICY_ARN --version-id $VERSION \
  --query 'PolicyVersion.Document' --output json
```

### Step 5: List all AWS Managed policies (filter by service)
```bash
# Show only IAM-related managed policies
aws iam list-policies --scope AWS \
  --query "Policies[?contains(PolicyName, 'IAM')].[PolicyName, Arn]" \
  --output table
```

### Step 6: Check what policies are attached to your own user
```bash
# Replace with your IAM username
MY_USER=$(aws sts get-caller-identity --query 'Arn' --output text | cut -d'/' -f2)
echo "My user: $MY_USER"

aws iam list-attached-user-policies --user-name $MY_USER
aws iam list-user-policies --user-name $MY_USER  # inline policies
```

### Step 7: Explore account-level IAM summary
```bash
aws iam get-account-summary --query 'SummaryMap' --output table
```
This gives you counts of users, groups, roles, policies, and MFA devices.

---

## 🔍 IAM Policy Anatomy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        }
      }
    }
  ]
}
```

| Field | Description |
|-------|-------------|
| `Version` | Always `"2012-10-17"` (the policy language version) |
| `Sid` | Optional statement ID for readability |
| `Effect` | `Allow` or `Deny` |
| `Action` | AWS API actions (e.g., `s3:GetObject`, `ec2:*`) |
| `Resource` | ARN(s) the action applies to (`*` = all) |
| `Condition` | Optional constraints (IP, time, tags, MFA, etc.) |

---

## ✅ Cleanup
No resources were created in this lab. All commands were read-only.

---

## ➡️ Next Lab
[Lab 02 — Users, Groups & Policies →](../02-users-groups-policies/README.md)
