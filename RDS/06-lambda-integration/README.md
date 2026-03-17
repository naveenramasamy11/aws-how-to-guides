# 🔴 Lab 06 — Lambda → RDS Integration Pipeline

**Difficulty:** 🔴 Advanced
**Time:** ~55 minutes
**Goal:** Build a serverless pipeline where an S3 PUT event triggers a Lambda function that reads a CSV file, validates rows, writes valid records to RDS via RDS Proxy, and publishes a summary to SNS.

---

## 🎯 Goal

End-to-end event-driven pipeline: CSV files dropped in S3 are automatically processed, persisted to RDS, and reported via SNS — all without managing servers.

---

## 📖 Concepts

### Why RDS Proxy with Lambda?

Lambda can scale to thousands of concurrent invocations. Each invocation opening its own database connection would exhaust RDS's connection limit almost immediately. RDS Proxy multiplexes these connections:

```
1000 Lambda invocations
    │  1000 short-lived connections
    ▼
RDS Proxy  (connection pool)
    │  10-20 real DB connections
    ▼
RDS MySQL (handles load normally)
```

### Pipeline Architecture

```
                ┌─────────────────────────────────────────────────────┐
                │                  AWS                                 │
                │                                                       │
S3 bucket       │   S3 Event        Lambda          RDS Proxy   RDS   │
orders-bucket   │   Notification    processor       (pool)      MySQL │
   │            │       │               │               │          │   │
   │ PUT         │       │               │               │          │   │
   │ orders/     │       ▼               │               │          │   │
   └──────────────────▶ Trigger ────────▶│               │          │   │
                │                       │ Get CSV        │          │   │
                │                       │◀──────────────S3          │   │
                │                       │ Parse + validate          │   │
                │                       │──────────────▶│──────────▶│   │
                │                       │ INSERT rows               │   │
                │                       │                           │   │
                │                       │──▶ SNS topic              │   │
                │                       │   "N rows saved"          │   │
                └─────────────────────────────────────────────────────┘
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                              VPC                                      │
│                                                                        │
│  ┌───────────────┐   VPC Interface   ┌──────────────────────────┐   │
│  │  Lambda       │◀─── Endpoints ───▶│  S3 VPC Endpoint (GW)    │   │
│  │  (VPC-attached│                   │  SM VPC Endpoint (Iface) │   │
│  │   subnet)     │                   └──────────────────────────┘   │
│  └───────┬───────┘                                                    │
│          │ TCP 3306 via proxy                                          │
│          ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  RDS Proxy  ──▶  RDS MySQL  (private subnet)                    │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  IAM:  Lambda Execution Role                                           │
│        - s3:GetObject (orders-bucket/*)                                │
│        - secretsmanager:GetSecretValue (db secret)                     │
│        - sns:Publish (orders-topic)                                    │
│        - rds-db:connect (proxy IAM auth)                               │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🔬 PoC Steps

### Step 1 — Environment setup

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export DB_ID=my-rds-pipeline
export DB_USER=admin
export DB_PASS="PipelinePass456!"
export BUCKET_NAME=orders-pipeline-$(echo $ACCOUNT_ID | cut -c1-8)
export SNS_TOPIC_NAME=orders-processed-topic

echo "Account: $ACCOUNT_ID | Bucket: $BUCKET_NAME"
```

### Step 2 — Create S3 bucket

```bash
aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region $AWS_REGION \
  --create-bucket-configuration LocationConstraint=$AWS_REGION \
  --tags Key=Project,Value=rds-howto 2>/dev/null || \
aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region $AWS_REGION \
  --tags Key=Project,Value=rds-howto

# Block all public access
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "Bucket: $BUCKET_NAME created"
```

### Step 3 — Create SNS topic

```bash
SNS_ARN=$(aws sns create-topic \
  --name $SNS_TOPIC_NAME \
  --tags Key=Project,Value=rds-howto \
  --query 'TopicArn' --output text)
echo "SNS Topic ARN: $SNS_ARN"

# Subscribe your email for notifications
aws sns subscribe \
  --topic-arn $SNS_ARN \
  --protocol email \
  --notification-endpoint naveenramasamy11@gmail.com
echo "Check your email to confirm SNS subscription"
```

### Step 4 — Set up RDS instance and Proxy (abbreviated — full steps in Lab 04)

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0:2].SubnetId' --output text | tr '\t' ' ')

aws rds create-db-subnet-group \
  --db-subnet-group-name rds-pipeline-subnet-group \
  --db-subnet-group-description "Pipeline subnet group" \
  --subnet-ids $SUBNET_IDS \
  --tags Key=Project,Value=rds-howto 2>/dev/null || true

SG_ID=$(aws ec2 create-security-group \
  --group-name rds-pipeline-sg --description "RDS pipeline SG" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=rds-howto}]" \
  --query 'GroupId' --output text 2>/dev/null || \
  aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=rds-pipeline-sg" "Name=vpc-id,Values=$VPC_ID" \
    --query 'SecurityGroups[0].GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 3306 --source-group $SG_ID 2>/dev/null || true

aws rds create-db-instance \
  --db-instance-identifier $DB_ID \
  --db-instance-class db.t3.micro \
  --engine mysql --engine-version 8.0.36 \
  --master-username $DB_USER --master-user-password "$DB_PASS" \
  --allocated-storage 20 --storage-type gp3 \
  --no-multi-az --no-publicly-accessible \
  --db-subnet-group-name rds-pipeline-subnet-group \
  --vpc-security-group-ids $SG_ID \
  --backup-retention-period 1 \
  --tags Key=Project,Value=rds-howto 2>/dev/null || echo "DB may already exist"

aws rds wait db-instance-available --db-instance-identifier $DB_ID

# Create orders table
DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].Endpoint.Address' --output text)

mysql -h $DB_ENDPOINT -u $DB_USER -p"$DB_PASS" <<'SQL'
CREATE DATABASE IF NOT EXISTS ordersdb;
USE ordersdb;
CREATE TABLE IF NOT EXISTS orders (
  id          INT AUTO_INCREMENT PRIMARY KEY,
  order_id    VARCHAR(50) UNIQUE NOT NULL,
  customer    VARCHAR(100),
  product     VARCHAR(100),
  quantity    INT,
  unit_price  DECIMAL(10,2),
  total       DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,
  status      ENUM('pending','processing','shipped','delivered') DEFAULT 'pending',
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
SQL
echo "Schema created"
```

### Step 5 — Store DB credentials in Secrets Manager

```bash
SECRET_ARN=$(aws secretsmanager create-secret \
  --name rds-pipeline-db-secret \
  --description "Pipeline RDS credentials" \
  --secret-string "{\"username\":\"${DB_USER}\",\"password\":\"${DB_PASS}\",\"host\":\"${DB_ENDPOINT}\",\"dbname\":\"ordersdb\"}" \
  --tags Key=Project,Value=rds-howto \
  --query 'ARN' --output text)
echo "Secret ARN: $SECRET_ARN"
```

### Step 6 — Create Lambda execution IAM role

```bash
cat > /tmp/lambda-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

LAMBDA_ROLE_ARN=$(aws iam create-role \
  --role-name rds-pipeline-lambda-role \
  --assume-role-policy-document file:///tmp/lambda-trust.json \
  --tags Key=Project,Value=rds-howto \
  --query 'Role.Arn' --output text)
echo "Lambda Role: $LAMBDA_ROLE_ARN"

# Attach managed policies
aws iam attach-role-policy --role-name rds-pipeline-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole

# Custom policy for S3, SNS, Secrets Manager
cat > /tmp/lambda-custom-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*"
    },
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "${SECRET_ARN}"
    },
    {
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "${SNS_ARN}"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name rds-pipeline-lambda-role \
  --policy-name lambda-pipeline-policy \
  --policy-document file:///tmp/lambda-custom-policy.json

sleep 10   # wait for IAM role propagation
```

### Step 7 — Write the Lambda function

```python
# lambda_function.py
import json
import csv
import io
import os
import boto3
import pymysql

s3_client  = boto3.client("s3")
sm_client  = boto3.client("secretsmanager")
sns_client = boto3.client("sns")

SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]
SECRET_ARN    = os.environ["SECRET_ARN"]

def get_db_connection():
    """Retrieve credentials from Secrets Manager and return a PyMySQL connection."""
    secret = json.loads(
        sm_client.get_secret_value(SecretId=SECRET_ARN)["SecretString"]
    )
    return pymysql.connect(
        host=secret["host"],
        user=secret["username"],
        password=secret["password"],
        database=secret["dbname"],
        connect_timeout=5,
        ssl={"ssl": {}},
        cursorclass=pymysql.cursors.DictCursor,
    )

def lambda_handler(event, context):
    """Process S3 CSV upload: validate rows, insert to RDS, notify via SNS."""
    results = {"inserted": 0, "skipped": 0, "errors": []}

    for record in event["Records"]:
        bucket = record["s3"]["bucket"]["name"]
        key    = record["s3"]["object"]["key"]
        print(f"Processing s3://{bucket}/{key}")

        # Read CSV from S3
        response = s3_client.get_object(Bucket=bucket, Key=key)
        content  = response["Body"].read().decode("utf-8")
        reader   = csv.DictReader(io.StringIO(content))

        rows_to_insert = []
        for row in reader:
            # Validate required fields
            try:
                order_id  = row["order_id"].strip()
                customer  = row["customer"].strip()
                product   = row["product"].strip()
                quantity  = int(row["quantity"])
                unit_price = float(row["unit_price"])
                if quantity <= 0 or unit_price <= 0:
                    raise ValueError("quantity and unit_price must be positive")
                rows_to_insert.append((order_id, customer, product, quantity, unit_price))
            except (KeyError, ValueError) as e:
                results["skipped"] += 1
                results["errors"].append(str(e))

        # Insert valid rows
        if rows_to_insert:
            conn = get_db_connection()
            try:
                with conn.cursor() as cursor:
                    cursor.executemany(
                        "INSERT IGNORE INTO orders (order_id, customer, product, quantity, unit_price) "
                        "VALUES (%s, %s, %s, %s, %s)",
                        rows_to_insert,
                    )
                conn.commit()
                results["inserted"] += cursor.rowcount
            finally:
                conn.close()

    # Publish summary to SNS
    message = (
        f"Order CSV processing complete.\n"
        f"Inserted: {results['inserted']} rows\n"
        f"Skipped:  {results['skipped']} rows\n"
        f"Errors:   {results['errors']}"
    )
    sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="RDS Pipeline: Orders Processed",
        Message=message,
    )
    print(message)
    return {"statusCode": 200, "body": results}
```

### Step 8 — Package and deploy Lambda

```bash
mkdir -p /tmp/lambda_pkg
cp lambda_function.py /tmp/lambda_pkg/

# Install pymysql layer
pip3 install pymysql -t /tmp/lambda_pkg/ --quiet

cd /tmp/lambda_pkg && zip -r /tmp/lambda_pipeline.zip . -q
cd -

LAMBDA_ARN=$(aws lambda create-function \
  --function-name orders-rds-pipeline \
  --runtime python3.12 \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/lambda_pipeline.zip \
  --role $LAMBDA_ROLE_ARN \
  --timeout 60 \
  --memory-size 256 \
  --vpc-config SubnetIds=$(echo $SUBNET_IDS | tr ' ' ','),SecurityGroupIds=$SG_ID \
  --environment "Variables={SNS_TOPIC_ARN=${SNS_ARN},SECRET_ARN=${SECRET_ARN}}" \
  --tags Key=Project,Value=rds-howto \
  --query 'FunctionArn' --output text)
echo "Lambda ARN: $LAMBDA_ARN"
```

### Step 9 — Add S3 trigger

```bash
# Allow S3 to invoke Lambda
aws lambda add-permission \
  --function-name orders-rds-pipeline \
  --statement-id s3-invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::$BUCKET_NAME \
  --source-account $ACCOUNT_ID

# Create S3 event notification
aws s3api put-bucket-notification-configuration \
  --bucket $BUCKET_NAME \
  --notification-configuration "{
    \"LambdaFunctionConfigurations\": [{
      \"LambdaFunctionArn\": \"$LAMBDA_ARN\",
      \"Events\": [\"s3:ObjectCreated:*\"],
      \"Filter\": {
        \"Key\": {\"FilterRules\": [{\"Name\": \"prefix\", \"Value\": \"orders/\"},{\"Name\": \"suffix\", \"Value\": \".csv\"}]}
      }
    }]
  }"
echo "S3 trigger configured"
```

### Step 10 — Test the pipeline end-to-end

```bash
# Create a sample CSV
cat > /tmp/orders_sample.csv <<'CSV'
order_id,customer,product,quantity,unit_price
ORD-001,Alice Johnson,Laptop Pro 15,1,1299.99
ORD-002,Bob Smith,Wireless Mouse,3,29.99
ORD-003,Carol White,USB-C Hub,2,49.99
INVALID-ROW,,,bad,-5
ORD-004,Dave Brown,Mechanical Keyboard,1,89.99
CSV

# Upload to S3 — this triggers Lambda
aws s3 cp /tmp/orders_sample.csv \
  s3://$BUCKET_NAME/orders/orders_$(date +%Y%m%d_%H%M%S).csv

echo "File uploaded. Waiting 15s for Lambda to process..."
sleep 15

# Check Lambda logs
LOG_GROUP="/aws/lambda/orders-rds-pipeline"
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime --descending \
  --limit 1 --query 'logStreams[0].logStreamName' --output text)

aws logs get-log-events \
  --log-group-name $LOG_GROUP \
  --log-stream-name "$LOG_STREAM" \
  --limit 20 \
  --query 'events[*].message' \
  --output text
```

**Expected output:**
```
Processing s3://orders-pipeline-12345678/orders/orders_20260317_080000.csv
Order CSV processing complete.
Inserted: 4 rows
Skipped:  1 rows
Errors:   ["quantity and unit_price must be positive"]
```

---

## 🧹 Cleanup

```bash
aws lambda delete-function --function-name orders-rds-pipeline
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3api delete-bucket --bucket $BUCKET_NAME
aws sns delete-topic --topic-arn $SNS_ARN
aws secretsmanager delete-secret --secret-id rds-pipeline-db-secret --force-delete-without-recovery
aws iam detach-role-policy --role-name rds-pipeline-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
aws iam delete-role-policy --role-name rds-pipeline-lambda-role --policy-name lambda-pipeline-policy
aws iam delete-role --role-name rds-pipeline-lambda-role
aws rds delete-db-instance --db-instance-identifier $DB_ID --skip-final-snapshot
aws rds wait db-instance-deleted --db-instance-identifier $DB_ID
aws rds delete-db-subnet-group --db-subnet-group-name rds-pipeline-subnet-group
aws ec2 delete-security-group --group-id $SG_ID
echo "Cleanup complete"
```

---

## ✅ What You Learned

- How to build a fully serverless event-driven pipeline with S3, Lambda, RDS, and SNS
- Why RDS Proxy is essential for Lambda-to-RDS architectures
- How to use Secrets Manager to securely provide DB credentials to Lambda
- How to write Lambda code that validates, bulk-inserts, and reports via SNS

---

> ➡️ Next: [Lab 07 — Production Patterns](../07-production-patterns/README.md)
