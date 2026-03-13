# Lab 06 — Permission Boundaries (Delegated Admin) 🔴

## 🎯 Goal
Implement **Permission Boundaries** to allow a team lead to create IAM roles for their team without being able to grant more permissions than they themselves have. This is the **delegated admin** pattern.

---

## 🧠 Concepts

### What is a Permission Boundary?
A Permission Boundary is an **AWS Managed or Customer Managed policy** that you attach to an IAM entity to set the **maximum permissions** that entity can ever have, regardless of what identity-based policies are attached.

```
Effective Permissions = Identity Policy ∩ Permission Boundary
```

This means even if a policy says `Allow: *`, if the boundary only allows S3 and EC2, the entity can only access S3 and EC2.

### The Delegated Admin Problem
Without boundaries, if you grant someone `iam:CreateRole` and `iam:AttachRolePolicy`, they can:
1. Create a role
2. Attach `AdministratorAccess` to it
3. Assume that role → **privilege escalation!**

Permission Boundaries solve this — the created role can only have permissions up to what the boundary allows.

### Key IAM Conditions for Delegated Admin Pattern
| Condition | Purpose |
|-----------|---------|
| `iam:PermissionsBoundary` | Enforce boundary on create/update |
| `iam:PassedToService` | Control which services a role can be passed to |
| `aws:TagKeys` | Enforce tagging (for ownership tracking) |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Admin (full IAM)                                         │
│  └── Creates: TeamLeadPolicy (with boundary enforcement)│
│  └── Creates: TeamBoundary (the max permissions)         │
│  └── Assigns: TeamLeadPolicy → TeamLead user            │
└─────────────────────────────────────────────────────────┘
         │
         ▼ delegates
┌─────────────────────────────────────────────────────────┐
│ TeamLead (can create roles/policies)                     │
│  └── CAN create roles — IF they attach TeamBoundary     │
│  └── CANNOT create roles without TeamBoundary attached   │
│  └── CANNOT create roles with permissions > TeamBoundary│
└─────────────────────────────────────────────────────────┘
         │
         ▼ creates
┌─────────────────────────────────────────────────────────┐
│ AppRole (created by TeamLead)                            │
│  ├── Permission Policy: AllowS3Access                    │
│  └── Permission Boundary: TeamBoundary ← ENFORCED       │
│      Effective: S3 only (limited by boundary)            │
└─────────────────────────────────────────────────────────┘
```

---

## 🧪 PoC

### Step 1: [Admin] Create the Permission Boundary policy
This defines the **maximum** permissions any role in this team can ever have:

```bash
cat > /tmp/team-boundary.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServicesForTeam",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "logs:*",
        "cloudwatch:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowIAMReadOnly",
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyPrivilegeEscalation",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeleteUser",
        "iam:AttachUserPolicy",
        "iam:PutUserPolicy",
        "organizations:*"
      ],
      "Resource": "*"
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

BOUNDARY_ARN=$(aws iam create-policy \
  --policy-name TeamBoundary \
  --description "Permission boundary: max permissions for app team roles" \
  --policy-document file:///tmp/team-boundary.json \
  --tags Key=Purpose,Value=PermissionBoundary \
  --query 'Policy.Arn' \
  --output text)

echo "Created boundary policy: $BOUNDARY_ARN ✅"
```

### Step 2: [Admin] Create the TeamLead policy (with boundary enforcement)
This policy allows the team lead to create roles, **but only if they attach the boundary**:

```bash
cat > /tmp/team-lead-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowIAMRoleManagement",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:TagRole",
        "iam:UntagRole",
        "iam:UpdateRole"
      ],
      "Resource": "arn:aws:iam::${ACCOUNT_ID}:role/team-*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "${BOUNDARY_ARN}"
        }
      }
    },
    {
      "Sid": "AllowPassRole",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::${ACCOUNT_ID}:role/team-*",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": [
            "lambda.amazonaws.com",
            "ec2.amazonaws.com"
          ]
        }
      }
    },
    {
      "Sid": "AllowPolicyManagement",
      "Effect": "Allow",
      "Action": [
        "iam:CreatePolicy",
        "iam:DeletePolicy",
        "iam:CreatePolicyVersion",
        "iam:DeletePolicyVersion",
        "iam:GetPolicy",
        "iam:GetPolicyVersion",
        "iam:ListPolicyVersions"
      ],
      "Resource": "arn:aws:iam::${ACCOUNT_ID}:policy/team-*"
    },
    {
      "Sid": "AllowReadIAM",
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyBoundaryModification",
      "Effect": "Deny",
      "Action": [
        "iam:DeletePolicy",
        "iam:CreatePolicyVersion",
        "iam:DeletePolicyVersion",
        "iam:SetDefaultPolicyVersion"
      ],
      "Resource": "${BOUNDARY_ARN}"
    }
  ]
}
EOF

TEAM_LEAD_POLICY_ARN=$(aws iam create-policy \
  --policy-name TeamLeadPolicy \
  --description "Allows team lead to manage roles with permission boundary enforcement" \
  --policy-document file:///tmp/team-lead-policy.json \
  --query 'Policy.Arn' \
  --output text)

echo "Created TeamLeadPolicy: $TEAM_LEAD_POLICY_ARN ✅"
```

### Step 3: [Admin] Create the TeamLead user and attach policy
```bash
aws iam create-user --user-name team-lead \
  --tags Key=Role,Value=TeamLead

aws iam attach-user-policy \
  --user-name team-lead \
  --policy-arn $TEAM_LEAD_POLICY_ARN

# Create access key for team-lead
TEAM_LEAD_CREDS=$(aws iam create-access-key --user-name team-lead)
echo "Team lead credentials:"
echo $TEAM_LEAD_CREDS | python3 -m json.tool

# Configure a separate profile for team-lead
# aws configure --profile team-lead
# (Enter the access key and secret from above)
echo "Configure team-lead profile with above credentials ✅"
```

### Step 4: [TeamLead] Attempt to create a role WITHOUT boundary (should fail)
```bash
# As team-lead, try to create a role without the permission boundary
# This should fail due to the iam:PermissionsBoundary condition

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
  --profile team-lead \
  --role-name team-app-role-noboundary \
  --assume-role-policy-document file:///tmp/lambda-trust.json 2>&1

# Expected: An error occurred (AccessDenied) — condition requires boundary ✅
```

### Step 5: [TeamLead] Create a role WITH the boundary (should succeed)
```bash
# This time, attach the permission boundary — should succeed
ROLE_ARN=$(aws iam create-role \
  --profile team-lead \
  --role-name team-app-lambda-role \
  --assume-role-policy-document file:///tmp/lambda-trust.json \
  --permissions-boundary $BOUNDARY_ARN \
  --description "App Lambda role created by team-lead with boundary" \
  --tags Key=ManagedBy,Value=team-lead Key=Team,Value=appteam \
  --query 'Role.Arn' \
  --output text)

echo "Created role with boundary: $ROLE_ARN ✅"
```

### Step 6: [TeamLead] Attach a policy to the role
```bash
# Create a policy (scoped to team-* prefix)
cat > /tmp/team-app-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::app-data-bucket",
        "arn:aws:s3:::app-data-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/app-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
EOF

APP_POLICY_ARN=$(aws iam create-policy \
  --profile team-lead \
  --policy-name team-app-s3-dynamo \
  --policy-document file:///tmp/team-app-policy.json \
  --query 'Policy.Arn' \
  --output text)

aws iam attach-role-policy \
  --profile team-lead \
  --role-name team-app-lambda-role \
  --policy-arn $APP_POLICY_ARN

echo "Policy attached to role ✅"
```

### Step 7: [TeamLead] Try to attach AdministratorAccess (should fail)
```bash
# Team lead tries to attach AdministratorAccess to the app role
# Even if this were allowed by the team-lead policy, the BOUNDARY blocks it
aws iam attach-role-policy \
  --profile team-lead \
  --role-name team-app-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess 2>&1

# Expected: AccessDenied — the boundary enforces resource naming pattern (team-*) ✅
```

### Step 8: Verify the effective permissions of the app role
```bash
# Check what policies the role has
echo "=== Attached Policies ==="
aws iam list-attached-role-policies \
  --role-name team-app-lambda-role \
  --query 'AttachedPolicies[*].[PolicyName,PolicyArn]' \
  --output table

# Check the permission boundary
echo "=== Permission Boundary ==="
aws iam get-role \
  --role-name team-app-lambda-role \
  --query 'Role.PermissionsBoundary' \
  --output table

# Even if the role has a policy allowing ec2:*, the boundary doesn't include EC2
# Simulate: try to describe EC2 instances with the role (should be denied by boundary)
aws iam simulate-principal-policy \
  --policy-source-arn $ROLE_ARN \
  --action-names s3:GetObject s3:DeleteBucket dynamodb:GetItem ec2:DescribeInstances \
  --resource-arns "*" \
  --query 'EvaluationResults[*].[ActionName,EvalDecision,MatchedStatements[0].SourcePolicyId]' \
  --output table
```

---

## 📊 Permission Boundary Decision Matrix

| Identity Policy | Permission Boundary | Effective Permission |
|----------------|--------------------|--------------------|
| Allow S3:* | Allow S3:* | Allow S3:* |
| Allow S3:* | Allow EC2:* | **DENY** (no overlap) |
| Allow `*` | Allow S3:*, EC2:* | Allow S3:* + EC2:* only |
| Deny S3:Delete | Allow S3:* | Deny S3:Delete |

---

## ✅ Cleanup

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Detach and cleanup team-lead resources
aws iam detach-user-policy --user-name team-lead --policy-arn $TEAM_LEAD_POLICY_ARN

LEAD_KEY=$(aws iam list-access-keys --user-name team-lead \
  --query 'AccessKeyMetadata[0].AccessKeyId' --output text)
[ "$LEAD_KEY" != "None" ] && \
  aws iam delete-access-key --user-name team-lead --access-key-id $LEAD_KEY

aws iam delete-user --user-name team-lead

# Cleanup role
aws iam detach-role-policy \
  --role-name team-app-lambda-role \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/team-app-s3-dynamo

aws iam delete-role --role-name team-app-lambda-role
aws iam delete-policy --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/team-app-s3-dynamo

# Cleanup policies
aws iam delete-policy --policy-arn $TEAM_LEAD_POLICY_ARN
aws iam delete-policy --policy-arn $BOUNDARY_ARN

echo "All resources cleaned up ✅"
```

---

## 🔒 Security Best Practices
1. **Always include a Deny on modifying the boundary itself** (prevent self-escalation)
2. **Scope by naming convention** — use resource prefixes like `team-*`
3. **Combine with SCPs** at the Organizations level for defense-in-depth
4. **Tag enforcement** in conditions ensures ownership tracking
5. **Use AWS Access Analyzer** to validate and review permission boundaries

---

## ➡️ Next Lab
[Lab 07 — ABAC (Attribute-Based Access Control) →](../07-abac-attribute-based-access/README.md)
