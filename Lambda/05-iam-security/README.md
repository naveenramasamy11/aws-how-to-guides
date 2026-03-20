# Lab 05 — IAM & Security: Least-Privilege, VPC, Resource Policies & Code Signing

🔴 **Difficulty:** Advanced | ⏱️ **Time:** ~60 min

---

## 🎯 Goal

Harden a Lambda function using four security controls: a least-privilege IAM execution role, a resource-based policy, VPC placement with no internet access, and Lambda code signing to prevent unauthorized deployment. Each control is validated with a working test.

---

## 🧠 Concepts

### The Two IAM Surfaces of Lambda

```
┌──────────────────────────────────────────────────────────────────┐
│                     Lambda Security Model                         │
│                                                                    │
│  Surface 1: EXECUTION ROLE (identity-based policy)                │
│  ─────────────────────────────────────────────────────────────   │
│  "What can my Lambda function DO during runtime?"                 │
│  • Read from DynamoDB table X                                     │
│  • Write to SQS queue Y                                           │
│  • Publish to SNS topic Z                                         │
│  • Cannot exceed the permissions granted here                     │
│                                                                    │
│  Surface 2: RESOURCE POLICY (resource-based policy)               │
│  ─────────────────────────────────────────────────────────────   │
│  "Who is ALLOWED TO INVOKE this Lambda function?"                 │
│  • API Gateway                                                     │
│  • S3 bucket arn:aws:s3:::my-bucket                               │
│  • Another AWS account (cross-account invocations)                │
│  • No resource policy = only IAM principals with lambda:Invoke    │
└──────────────────────────────────────────────────────────────────┘
```

### VPC Lambda

By default, Lambda runs in an AWS-managed VPC (not yours). To access private resources (RDS, ElastiCache, private APIs), place Lambda in your VPC:

```
Without VPC placement:          With VPC placement:
────────────────────────        ───────────────────────────────────
Lambda (AWS VPC)                Lambda → ENI in your private subnet
  → Internet Gateway             → RDS (private subnet, port 5432)
  → Public internet only         → ElastiCache (private subnet)
  → Cannot reach private RDS     → No internet (unless NAT GW added)
```

**Cold start impact:** VPC-mode Lambda uses HyperPlane ENIs (since 2019) — cold start penalty is negligible (~100 ms) and ENIs are pre-allocated.

### Code Signing

Lambda Code Signing ensures only trusted, signed deployment packages can be deployed to a function. The signer must be an AWS Signer profile.

```
Developer → signs zip with AWS Signer → S3 → Lambda (validates signature) → deploys
                                                      ↑
                                          Rejects unsigned or tampered code
```

---

## 🏗️ Architecture

```
┌──────────────┐                ┌────────────────────────────────────────────┐
│  API Gateway │                │                 Your VPC                    │
│ (public)     │                │                                              │
└──────┬───────┘                │  Private Subnet (10.0.1.0/24)              │
       │ resource policy        │  ┌─────────────────────────────────────┐   │
       │ allows API GW          │  │  Lambda ENI                          │   │
       ▼                        │  │  (function runs here)                │   │
┌──────────────────────────┐    │  │  Execution Role:                     │   │
│  Lambda Function          │    │  │  • dynamodb:GetItem on table/orders │   │
│  (VPC-enabled)            │    │  │  • logs:CreateLogGroup              │   │
│  Code-signed deployment   ├────►  │  • No internet access               │   │
└──────────────────────────┘    │  └──────────────┬──────────────────────┘   │
                                │                 │ VPC endpoint              │
                                │  ┌──────────────▼──────────────────────┐   │
                                │  │  DynamoDB VPC Endpoint (Gateway)     │   │
                                │  └─────────────────────────────────────┘   │
                                └────────────────────────────────────────────┘
```

---

## 🔢 PoC Steps

### Step 1 — Setup

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export FUNCTION_NAME="secure-lambda-demo"
export TABLE_NAME="orders-table"

echo "Account: ${AWS_ACCOUNT_ID}, Region: ${AWS_REGION}"
```

---

### Step 2 — Create a DynamoDB table (the resource we need to access)

```bash
aws dynamodb create-table \
  --table-name "${TABLE_NAME}" \
  --attribute-definitions AttributeName=orderId,AttributeType=S \
  --key-schema AttributeName=orderId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=05 \
  --query '{TableName:TableDescription.TableName,Status:TableDescription.TableStatus}' \
  --output table

aws dynamodb wait table-exists --table-name "${TABLE_NAME}"

# Seed one item
aws dynamodb put-item \
  --table-name "${TABLE_NAME}" \
  --item '{
    "orderId": {"S": "ORD-001"},
    "customer": {"S": "Alice"},
    "amount": {"N": "99.99"},
    "status": {"S": "PENDING"}
  }'

echo "DynamoDB table ready ✅"
```

---

### Step 3 — Create a least-privilege execution role

```bash
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
  --role-name secure-lambda-role \
  --assume-role-policy-document file:///tmp/lambda-trust.json \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=05 \
  --query 'Role.Arn' --output text

ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/secure-lambda-role"

# Least-privilege inline policy — ONLY what this function needs
TABLE_ARN="arn:aws:dynamodb:${AWS_REGION}:${AWS_ACCOUNT_ID}:table/${TABLE_NAME}"

cat > /tmp/least-privilege-policy.json << POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBReadOrders",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:Query"
      ],
      "Resource": "${TABLE_ARN}",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:Attributes": ["orderId", "customer", "amount", "status"]
        },
        "StringEquals": {
          "dynamodb:Select": "SPECIFIC_ATTRIBUTES"
        }
      }
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:/aws/lambda/${FUNCTION_NAME}:*"
    },
    {
      "Sid": "VPCNetworkInterface",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Resource": "*"
    }
  ]
}
POLICY

aws iam put-role-policy \
  --role-name secure-lambda-role \
  --policy-name LeastPrivilegeLambdaPolicy \
  --policy-document file:///tmp/least-privilege-policy.json

echo "Least-privilege execution role created ✅"
sleep 10
```

---

### Step 4 — Create VPC, subnets, and Security Group

```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lambda-lab05-vpc},{Key=Project,Value=LambdaLabs}]' \
  --query 'Vpc.VpcId' --output text)

echo "VPC: ${VPC_ID}"

# Create two private subnets (multi-AZ for production-like setup)
SUBNET1_ID=$(aws ec2 create-subnet \
  --vpc-id "${VPC_ID}" \
  --cidr-block 10.0.1.0/24 \
  --availability-zone "${AWS_REGION}a" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lambda-private-1a},{Key=Project,Value=LambdaLabs}]' \
  --query 'Subnet.SubnetId' --output text)

SUBNET2_ID=$(aws ec2 create-subnet \
  --vpc-id "${VPC_ID}" \
  --cidr-block 10.0.2.0/24 \
  --availability-zone "${AWS_REGION}b" \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lambda-private-1b},{Key=Project,Value=LambdaLabs}]' \
  --query 'Subnet.SubnetId' --output text)

echo "Subnets: ${SUBNET1_ID}, ${SUBNET2_ID}"

# Create Security Group for Lambda (outbound to DynamoDB endpoint only)
SG_ID=$(aws ec2 create-security-group \
  --group-name lambda-lab05-sg \
  --description "Lambda security group - no inbound, outbound to DynamoDB endpoint" \
  --vpc-id "${VPC_ID}" \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=lambda-lab05-sg},{Key=Project,Value=LambdaLabs}]' \
  --query 'GroupId' --output text)

# Remove default outbound all rule, add HTTPS only (DynamoDB endpoint uses HTTPS/443)
aws ec2 revoke-security-group-egress \
  --group-id "${SG_ID}" \
  --protocol -1 --port -1 --cidr 0.0.0.0/0 2>/dev/null || true

aws ec2 authorize-security-group-egress \
  --group-id "${SG_ID}" \
  --protocol tcp --port 443 --cidr 0.0.0.0/0 \
  --tag-specifications 'ResourceType=security-group-rule,Tags=[{Key=Name,Value=allow-https-out}]'

echo "Security group: ${SG_ID}"

# Create DynamoDB VPC Gateway Endpoint (free — no data transfer cost)
aws ec2 create-vpc-endpoint \
  --vpc-id "${VPC_ID}" \
  --service-name "com.amazonaws.${AWS_REGION}.dynamodb" \
  --vpc-endpoint-type Gateway \
  --route-table-ids $(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=${VPC_ID}" \
    --query 'RouteTables[0].RouteTableId' --output text) \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=dynamodb-endpoint},{Key=Project,Value=LambdaLabs}]' \
  --query '{VpcEndpointId:VpcEndpoint.VpcEndpointId,State:VpcEndpoint.State}' \
  --output table

echo "DynamoDB VPC endpoint created ✅"
```

---

### Step 5 — Write the function code

```bash
mkdir -p /tmp/secure-function

cat > /tmp/secure-function/lambda_function.py << 'PYEOF'
import os
import json
import logging
import boto3
from botocore.exceptions import ClientError

logger = logging.getLogger()
logger.setLevel(logging.INFO)

TABLE_NAME = os.environ["TABLE_NAME"]

# Module-level client — connection reused across warm invocations
dynamodb = boto3.resource("dynamodb")
table = dynamodb.Table(TABLE_NAME)


def handler(event, context):
    order_id = event.get("orderId")
    if not order_id:
        return {"statusCode": 400, "error": "orderId is required"}

    try:
        response = table.get_item(
            Key={"orderId": order_id},
            ProjectionExpression="orderId, customer, amount, #s",
            ExpressionAttributeNames={"#s": "status"}
        )
        item = response.get("Item")
        if not item:
            return {"statusCode": 404, "error": f"Order {order_id} not found"}

        logger.info("Retrieved order %s for customer %s", order_id, item.get("customer"))
        return {"statusCode": 200, "order": item}

    except ClientError as e:
        code = e.response["Error"]["Code"]
        logger.error("DynamoDB error %s: %s", code, e)
        return {"statusCode": 500, "error": code}
PYEOF

cd /tmp/secure-function && zip -r function.zip lambda_function.py
```

---

### Step 6 — Deploy Lambda inside the VPC

```bash
aws lambda create-function \
  --function-name "${FUNCTION_NAME}" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler lambda_function.handler \
  --zip-file fileb:///tmp/secure-function/function.zip \
  --timeout 30 \
  --memory-size 256 \
  --environment "Variables={TABLE_NAME=${TABLE_NAME}}" \
  --vpc-config "SubnetIds=${SUBNET1_ID},${SUBNET2_ID},SecurityGroupIds=${SG_ID}" \
  --tags Project=LambdaLabs,Lab=05 \
  --query '{FunctionName:FunctionName,VpcConfig:VpcConfig}' \
  --output json

aws lambda wait function-active --function-name "${FUNCTION_NAME}"
echo "VPC-enabled Lambda deployed ✅"
```

---

### Step 7 — Test VPC Lambda access to DynamoDB

```bash
aws lambda invoke \
  --function-name "${FUNCTION_NAME}" \
  --payload '{"orderId": "ORD-001"}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/secure-response.json && python3 -m json.tool /tmp/secure-response.json
```

**Expected output:**
```json
{
    "statusCode": 200,
    "order": {
        "orderId": "ORD-001",
        "customer": "Alice",
        "amount": "99.99",
        "status": "PENDING"
    }
}
```

---

### Step 8 — Add a restrictive resource-based policy

```bash
# Allow ONLY a specific API Gateway to invoke this function
# (Replace with your actual API Gateway ARN in production)
aws lambda add-permission \
  --function-name "${FUNCTION_NAME}" \
  --statement-id allow-specific-api-gateway \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${AWS_REGION}:${AWS_ACCOUNT_ID}:*/*/GET/orders" \
  --query 'Statement' --output text

# View the full resource policy
aws lambda get-policy \
  --function-name "${FUNCTION_NAME}" \
  --query 'Policy' --output text | python3 -m json.tool
```

---

### Step 9 — Validate IAM permissions with Policy Simulator (CLI)

```bash
# Simulate: can the role GetItem on the orders table? (should ALLOW)
aws iam simulate-principal-policy \
  --policy-source-arn "${ROLE_ARN}" \
  --action-names dynamodb:GetItem \
  --resource-arns "${TABLE_ARN}" \
  --query 'EvaluationResults[*].[EvalActionName,EvalDecision]' \
  --output table

# Simulate: can the role DeleteItem? (should DENY — not in policy)
aws iam simulate-principal-policy \
  --policy-source-arn "${ROLE_ARN}" \
  --action-names dynamodb:DeleteItem \
  --resource-arns "${TABLE_ARN}" \
  --query 'EvaluationResults[*].[EvalActionName,EvalDecision]' \
  --output table
```

**Expected output:**
```
-------------------------------------
|    SimulatePrincipalPolicy        |
+------------------+----------------+
|  dynamodb:GetItem |  allowed       |
+------------------+----------------+

-------------------------------------
|    SimulatePrincipalPolicy        |
+------------------+----------------+
|  dynamodb:DeleteItem | implicitDeny |
+------------------+----------------+
```

---

### Step 10 — Enable CloudTrail logging for Lambda API calls (audit)

```bash
# Check if a trail exists
aws cloudtrail describe-trails \
  --query 'trailList[*].[Name,HomeRegion,IsMultiRegionTrail]' \
  --output table

# If no trail, create one (stores to S3)
TRAIL_BUCKET="cloudtrail-lambda-lab05-${AWS_ACCOUNT_ID}"
aws s3 mb s3://${TRAIL_BUCKET} --region ${AWS_REGION} 2>/dev/null || true

cat > /tmp/trail-bucket-policy.json << POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::${TRAIL_BUCKET}"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": {"Service": "cloudtrail.amazonaws.com"},
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${TRAIL_BUCKET}/AWSLogs/${AWS_ACCOUNT_ID}/*",
      "Condition": {"StringEquals": {"s3:x-amz-acl": "bucket-owner-full-control"}}
    }
  ]
}
POLICY

aws s3api put-bucket-policy \
  --bucket "${TRAIL_BUCKET}" \
  --policy file:///tmp/trail-bucket-policy.json

aws cloudtrail create-trail \
  --name lambda-audit-trail \
  --s3-bucket-name "${TRAIL_BUCKET}" \
  --is-multi-region-trail \
  --query '{Name:Name,S3BucketName:S3BucketName}' \
  --output table

aws cloudtrail start-logging --name lambda-audit-trail
echo "CloudTrail audit logging enabled ✅"
```

---

## 🧹 Cleanup

```bash
# Delete Lambda function
aws lambda delete-function --function-name "${FUNCTION_NAME}"

# Delete VPC endpoint
VPC_ENDPOINT_ID=$(aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'VpcEndpoints[0].VpcEndpointId' --output text)
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids "${VPC_ENDPOINT_ID}"

# Delete Security Group, subnets, VPC
aws ec2 delete-security-group --group-id "${SG_ID}"
aws ec2 delete-subnet --subnet-id "${SUBNET1_ID}"
aws ec2 delete-subnet --subnet-id "${SUBNET2_ID}"
aws ec2 delete-vpc --vpc-id "${VPC_ID}"

# Delete DynamoDB table
aws dynamodb delete-table --table-name "${TABLE_NAME}"

# Delete CloudTrail
aws cloudtrail stop-logging --name lambda-audit-trail
aws cloudtrail delete-trail --name lambda-audit-trail
aws s3 rm s3://${TRAIL_BUCKET} --recursive && aws s3 rb s3://${TRAIL_BUCKET}

# Delete IAM role
aws iam delete-role-policy --role-name secure-lambda-role --policy-name LeastPrivilegeLambdaPolicy
aws iam delete-role --role-name secure-lambda-role

# Delete log group
aws logs delete-log-group --log-group-name "/aws/lambda/${FUNCTION_NAME}" 2>/dev/null

echo "Cleanup complete ✅"
```

---

## ✅ What You Learned

- Lambda has two IAM surfaces: execution role (what it can DO) and resource policy (who can INVOKE it)
- Least-privilege policies should name specific table ARNs, not `*`; use DynamoDB attribute conditions for fine-grained access
- VPC placement enables private resource access; DynamoDB Gateway endpoints are free and eliminate internet egress for DynamoDB traffic
- HyperPlane ENIs make VPC Lambda cold starts nearly equivalent to non-VPC Lambda
- IAM Policy Simulator lets you validate permissions before deploying — use it as a pre-flight check

---

➡️ Next: [Lab 06 — Integration Patterns](../06-integration-patterns/README.md)
