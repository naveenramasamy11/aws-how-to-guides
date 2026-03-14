# Lab 03 — Security Groups & NACLs 🟡

## 🎯 Goal
Build a layered defence-in-depth network security model using both Security Groups (stateful, instance-level) and NACLs (stateless, subnet-level), and understand how their rules interact.

---

## 🧠 Concepts

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | ENI (instance) | Subnet |
| **State** | Stateful — return traffic auto-allowed | Stateless — must explicitly allow both directions |
| **Rules** | Allow only (no deny) | Allow AND Deny |
| **Rule evaluation** | All rules evaluated together | Rules evaluated in order (lowest number wins) |
| **Default** | Deny all inbound, allow all outbound | Allow all in/out (default NACL) |
| **Association** | Many-to-many (SG ↔ instance) | One NACL per subnet |
| **Best for** | Application-layer firewall | Broad subnet-level blocks (e.g., block a CIDR) |

### Security Group Referencing
SGs can reference **other SGs** as source/destination — this is more flexible and maintainable than using CIDRs:
```
# Instead of: allow 10.10.1.0/24 on port 3306
# Use:        allow sg-web-tier on port 3306
```

This means "any instance in the web-tier SG can access MySQL" — membership-driven, not IP-driven.

### NACL Rule Numbering Convention
```
100  — Allow known-good traffic (application ports)
200  — Allow ephemeral ports (return traffic)
900  — Explicit deny specific bad actors
*    — Implicit deny all (always last, cannot be removed)
```

---

## 🏗️ Architecture

```
                   Internet
                      │
                  [NACL-Public]
                 Allow: 80,443 in
                 Deny: 1024-65535 blocked CIDR
                      │
            ┌─────────┴─────────┐
            │   Public Subnet   │
            │   [sg-alb]        │ ← Allow 80,443 from 0.0.0.0/0
            │   Load Balancer   │
            └─────────┬─────────┘
                      │
                  [NACL-Private]
                 Allow: 8080 from public subnet CIDR only
                      │
            ┌─────────┴─────────┐
            │  Private Subnet   │
            │  [sg-app]         │ ← Allow 8080 from sg-alb only
            │  App Server       │
            └─────────┬─────────┘
                      │
                 [sg-rds]        ← Allow 5432 from sg-app only
                      │
            ┌─────────┴─────────┐
            │  Private Subnet   │
            │  RDS PostgreSQL   │
            └───────────────────┘
```

---

## 🧪 PoC

```bash
source ~/.vpc_lab_vars
```

### Step 1: Create the ALB Security Group (public-facing)
```bash
SG_ALB=$(aws ec2 create-security-group \
  --group-name sg-alb \
  --description "ALB: allow HTTP and HTTPS from internet" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-alb},{Key=Tier,Value=public}]' \
  --query 'GroupId' --output text)
echo "ALB SG: $SG_ALB ✅"

# Allow HTTP from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ALB \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Allow HTTPS from anywhere
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ALB \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

echo "HTTP/HTTPS rules added to sg-alb ✅"
```

### Step 2: Create the App Server Security Group (SG referencing)
```bash
SG_APP=$(aws ec2 create-security-group \
  --group-name sg-app \
  --description "App tier: allow 8080 from ALB SG only" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-app},{Key=Tier,Value=app}]' \
  --query 'GroupId' --output text)
echo "App SG: $SG_APP ✅"

# Allow port 8080 from the ALB SG (not a CIDR — uses SG reference)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 8080 \
  --source-group $SG_ALB
echo "Port 8080 from sg-alb allowed on sg-app ✅"

# Allow SSH from within the VPC (for bastion access in Lab 04)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_APP \
  --protocol tcp --port 22 \
  --cidr 10.10.0.0/16
echo "SSH from VPC CIDR allowed on sg-app ✅"
```

### Step 3: Create the RDS Security Group
```bash
SG_RDS=$(aws ec2 create-security-group \
  --group-name sg-rds \
  --description "RDS: allow PostgreSQL from app SG only" \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=sg-rds},{Key=Tier,Value=data}]' \
  --query 'GroupId' --output text)
echo "RDS SG: $SG_RDS ✅"

# Allow PostgreSQL (5432) from app SG only
aws ec2 authorize-security-group-ingress \
  --group-id $SG_RDS \
  --protocol tcp --port 5432 \
  --source-group $SG_APP
echo "Port 5432 from sg-app allowed on sg-rds ✅"
```

### Step 4: Remove default outbound rules and add explicit ones (best practice)
```bash
# By default, all SGs allow all outbound. Lock it down for the app tier:
# First get the default egress rule ID
EGRESS_RULE_ID=$(aws ec2 describe-security-group-rules \
  --filters "Name=group-id,Values=$SG_APP" \
  --query 'SecurityGroupRules[?IsEgress==`true`].SecurityGroupRuleId' \
  --output text)

aws ec2 revoke-security-group-egress \
  --group-id $SG_APP \
  --security-group-rule-ids $EGRESS_RULE_ID

# Allow only outbound to RDS on 5432
aws ec2 authorize-security-group-egress \
  --group-id $SG_APP \
  --protocol tcp --port 5432 \
  --destination-group $SG_RDS

# Allow outbound HTTPS (for AWS API calls, SSM)
aws ec2 authorize-security-group-egress \
  --group-id $SG_APP \
  --protocol tcp --port 443 --cidr 0.0.0.0/0

echo "App SG egress locked down ✅"
```

### Step 5: Create a custom NACL for public subnets
```bash
# Create the public NACL
NACL_PUBLIC=$(aws ec2 create-network-acl \
  --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=nacl-public}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)
echo "Public NACL: $NACL_PUBLIC ✅"

# INBOUND rules
# Rule 100: Allow HTTP
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_PUBLIC \
  --ingress --rule-number 100 \
  --protocol tcp --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Rule 110: Allow HTTPS
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_PUBLIC \
  --ingress --rule-number 110 \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Rule 200: Allow ephemeral/return traffic ports (stateless requirement!)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_PUBLIC \
  --ingress --rule-number 200 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Rule 900: Block a specific bad CIDR (example)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_PUBLIC \
  --ingress --rule-number 900 \
  --protocol -1 \
  --cidr-block 192.0.2.0/24 --rule-action deny

echo "NACL inbound rules added ✅"

# OUTBOUND rules
# Rule 100: Allow HTTP outbound
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_PUBLIC \
  --egress --rule-number 100 \
  --protocol tcp --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Rule 110: Allow HTTPS outbound
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_PUBLIC \
  --egress --rule-number 110 \
  --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow

# Rule 200: Allow ephemeral ports outbound (return traffic to clients)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_PUBLIC \
  --egress --rule-number 200 \
  --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow

echo "NACL outbound rules added ✅"
```

### Step 6: Associate the NACL with public subnets
```bash
# Disassociate from default NACL and associate with custom
for SUBNET in $PUB_SUBNET_1A $PUB_SUBNET_1B; do
  # Get current association ID
  ASSOC_ID=$(aws ec2 describe-network-acls \
    --filters "Name=association.subnet-id,Values=$SUBNET" \
    --query 'NetworkAcls[0].Associations[?SubnetId==`'$SUBNET'`].NetworkAclAssociationId' \
    --output text)

  aws ec2 replace-network-acl-association \
    --association-id $ASSOC_ID \
    --network-acl-id $NACL_PUBLIC
  echo "Subnet $SUBNET associated with nacl-public ✅"
done
```

### Step 7: Verify the full setup
```bash
echo "=== Security Groups ==="
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'SecurityGroups[*].[GroupId,GroupName,Description]' \
  --output table

echo "=== Inbound rules for sg-app ==="
aws ec2 describe-security-group-rules \
  --filters "Name=group-id,Values=$SG_APP" \
  --query 'SecurityGroupRules[?IsEgress==`false`].[IpProtocol,FromPort,ToPort,CidrIpv4,ReferencedGroupInfo.GroupId]' \
  --output table

echo "=== NACL rules ==="
aws ec2 describe-network-acls --network-acl-ids $NACL_PUBLIC \
  --query 'NetworkAcls[0].Entries[*].[RuleNumber,Protocol,RuleAction,CidrBlock,PortRange.From,PortRange.To,Egress]' \
  --output table

# Save SG IDs
cat >> ~/.vpc_lab_vars << EOF
export SG_ALB=$SG_ALB
export SG_APP=$SG_APP
export SG_RDS=$SG_RDS
export NACL_PUBLIC=$NACL_PUBLIC
EOF
```

---

## ⚠️ Common Mistakes

| Mistake | Impact | Fix |
|---------|--------|-----|
| Forgetting ephemeral ports in NACL | Connections time out | Add rule for `1024-65535` both in/out |
| Using CIDRs instead of SG refs for inter-tier traffic | Breaks on instance replacement (IP changes) | Use SG-to-SG references |
| Overly permissive `0.0.0.0/0` outbound on all SGs | Wide blast radius | Scope egress to required destinations |
| Single NACL for all subnets | Can't apply different policies | One NACL per tier |

---

## ✅ Cleanup
> Keep SGs and NACL for subsequent labs.

---

## ➡️ Next Lab
[Lab 04 — NAT Gateway & Bastion Host →](../04-nat-gateway-bastion/README.md)
