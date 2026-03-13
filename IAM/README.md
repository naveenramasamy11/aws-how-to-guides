# AWS IAM — How-To Guide 🔐

> Identity and Access Management (IAM) is the foundation of AWS security.
> This guide takes you from zero to advanced IAM patterns with hands-on PoC labs.

---

## 🗺️ Learning Path

```
🟢 Beginner
  └── Lab 01 — IAM Fundamentals (Concepts & Console Tour)
  └── Lab 02 — Users, Groups & Managed Policies

🟡 Intermediate
  └── Lab 03 — IAM Roles for EC2 (Instance Profiles)
  └── Lab 04 — IAM Roles for Lambda (Execution Roles)

🔴 Advanced
  └── Lab 05 — Cross-Account Access with IAM Roles
  └── Lab 06 — Permission Boundaries (Delegated Admin)
  └── Lab 07 — ABAC — Attribute-Based Access Control
```

---

## 📋 Labs Index

| # | Lab | Key Concepts | Difficulty |
|---|-----|-------------|------------|
| 01 | [IAM Fundamentals](./01-iam-fundamentals/README.md) | Users, Groups, Roles, Policies, ARN | 🟢 |
| 02 | [Users, Groups & Policies](./02-users-groups-policies/README.md) | MFA, Inline vs Managed, Least Privilege | 🟢 |
| 03 | [IAM Roles for EC2](./03-iam-roles-ec2/README.md) | Instance Profiles, AssumeRole, Metadata API | 🟡 |
| 04 | [IAM Roles for Lambda](./04-iam-roles-lambda/README.md) | Execution Role, Resource-Based Policy | 🟡 |
| 05 | [Cross-Account Access](./05-cross-account-access/README.md) | Trust Policy, sts:AssumeRole, ExternalId | 🔴 |
| 06 | [Permission Boundaries](./06-permission-boundaries/README.md) | Delegated Admin, Max Permissions | 🔴 |
| 07 | [ABAC](./07-abac-attribute-based-access/README.md) | Tags, Conditions, aws:PrincipalTag | 🔴 |

---

## 🔑 Core IAM Concepts (Quick Reference)

| Concept | Description |
|---------|-------------|
| **User** | Human or service identity with long-term credentials |
| **Group** | Collection of users sharing the same permissions |
| **Role** | Temporary identity assumed by users, services, or accounts |
| **Policy** | JSON document defining permissions (Allow/Deny) |
| **ARN** | Amazon Resource Name — unique identifier for any AWS resource |
| **Trust Policy** | Defines WHO can assume a role |
| **Permission Policy** | Defines WHAT actions the role can perform |
| **Permission Boundary** | Sets the MAX permissions an entity can ever have |
| **SCP** | Service Control Policy — org-level guardrails (AWS Organizations) |

---

## 🛡️ IAM Policy Evaluation Logic

```
Request comes in
      │
      ▼
1. Explicit DENY in any policy?  ──── YES ──► DENY
      │ NO
      ▼
2. SCP (Org) allows the action?  ──── NO  ──► DENY
      │ YES
      ▼
3. Permission Boundary allows?   ──── NO  ──► DENY
      │ YES
      ▼
4. Explicit ALLOW in policy?     ──── NO  ──► DENY
      │ YES
      ▼
            ALLOW ✅
```

---

## ⚙️ Setup

All labs use AWS CLI. Configure it before starting:

```bash
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name: us-east-1
# Default output format: json
```

Verify:
```bash
aws sts get-caller-identity
```

---

## 📎 Useful IAM CLI Commands (Cheat Sheet)

```bash
# List all users
aws iam list-users

# List all roles
aws iam list-roles

# List policies attached to a user
aws iam list-attached-user-policies --user-name <USERNAME>

# Simulate a policy (what can a user do?)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::<ACCOUNT_ID>:user/<USERNAME> \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/*

# Get the current caller identity
aws sts get-caller-identity
```
