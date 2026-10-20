# Day 46 – Automate S3-to-S3 File Copy Using AWS Lambda

## 📌 Project Overview

This lab demonstrates how to automate copying files from a public Amazon S3 bucket to a private S3 bucket using AWS Lambda. Amazon DynamoDB stores logs of the copy operation.

When a new object is uploaded to the source bucket, Amazon S3 sends an event to Lambda. The function copies the object to the destination bucket and writes a log entry to DynamoDB.

## 🎯 Objectives

* Configure public and private S3 buckets.
* Create an S3 bucket policy for public read access.
* Configure IAM trust and permissions policies.
* Automate file copying using AWS Lambda.
* Configure S3 event notifications.
* Store copy-operation logs in DynamoDB.
* Verify the workflow using AWS CLI.

## 🛠️ AWS Services Used

* Amazon S3
* AWS Lambda
* Amazon DynamoDB
* AWS IAM
* Amazon CloudWatch

## 🏗️ Architecture

```text
User uploads a file
        |
        v
Public S3 Bucket
        |
        | ObjectCreated Event
        v
AWS Lambda Function
datacenter-copyfunction
        |
        +----------------------+
        |                      |
        v                      v
Private S3 Bucket       DynamoDB Table
Copied file             Operation logs
```

## 📋 Resource Configuration

| Resource           | Name / Configuration             |
| ------------------ | -------------------------------- |
| Source bucket      | `datacenter-public-186065528`    |
| Destination bucket | `datacenter-private-186065528`   |
| Lambda function    | `datacenter-copyfunction`        |
| Runtime            | Python 3.12                      |
| Handler            | `lambda_function.lambda_handler` |
| Memory             | 128 MB                           |
| Timeout            | 10 seconds                       |
| IAM role           | `lambda_execution_role`          |
| DynamoDB table     | `datacenter-S3CopyLogs`          |
| Partition key      | `LogID` — String                 |
| S3 trigger         | `s3:ObjectCreated:*`             |

---

## 1. Verify Configure AWS CLI

Go to AWS client and verify that AWS CLI is installed

```bash
aws --version
aws configure
aws sts get-caller-identity
```

Set your AWS region and account ID:

```bash
export AWS_REGION=$(aws configure get region)
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "$AWS_REGION"
echo "$ACCOUNT_ID"
```

Use the same region for the S3 buckets, Lambda function, and DynamoDB table where required by your lab configuration.

**Note:** The following commands configure the named resources. If you already created them, do not create duplicate resources.

## 2. Create the S3 Buckets (It may be different name in your case)

For a bucket in your configured region, the creation command is:

```bash
aws s3api create-bucket \
  --bucket datacenter-public-186065528 \
  --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"
```

Create the private bucket:

```bash
aws s3api create-bucket \
  --bucket datacenter-private-186065528 \
  --region "$AWS_REGION" \
  --create-bucket-configuration LocationConstraint="$AWS_REGION"
```

**Important:** For `us-east-1`, omit `--create-bucket-configuration LocationConstraint=...`. Bucket names must be globally unique, and S3 bucket names cannot be changed after creation.

Verify the buckets:

```bash
aws s3api list-buckets \
  --query 'Buckets[].Name' \
  --output table
```

## 3. S3 Bucket Policy JSON — Public Read Access

Go to permission and edit the bucket policy: 

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowPublicRead",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::datacenter-public-186065528/*"
    }
  ]
}
```

This policy permits anyone to read objects in the source bucket. It does not grant permission to upload, delete, or modify objects.

Apply the policy:

```bash
aws s3api put-bucket-policy \
  --bucket datacenter-public-186065528 \
  --policy file://public-bucket-policy.json \
  --region "$AWS_REGION"
```

If your lab specifically requires public object access, configure the source bucket's Block Public Access settings to permit the policy:

```bash
aws s3api put-public-access-block \
  --bucket datacenter-public-186065528 \
  --public-access-block-configuration \
  'BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=false,RestrictPublicBuckets=false' \
  --region "$AWS_REGION"
```

This configuration allows the bucket policy to provide public read access without allowing public ACLs. An organization-level or account-level Block Public Access setting may still override this configuration.

Verify the bucket policy:

```bash
aws s3api get-bucket-policy \
  --bucket datacenter-public-186065528 \
  --region "$AWS_REGION"
```

**Security note:** Public read access is used here to satisfy the lab requirement. Do not use it for private data or production buckets. Keep the destination bucket private.

## 4. Create the DynamoDB Table

The table uses `LogID` as its partition key, with type String.

```bash
aws dynamodb create-table \
  --table-name datacenter-S3CopyLogs \
  --attribute-definitions AttributeName=LogID,AttributeType=S \
  --key-schema AttributeName=LogID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region "$AWS_REGION"
```

If the table already exists, verify it instead of creating it again:

```bash
aws dynamodb describe-table \
  --table-name datacenter-S3CopyLogs \
  --region "$AWS_REGION"
```

Wait until the table is active:

```bash
aws dynamodb wait table-exists \
  --table-name datacenter-S3CopyLogs \
  --region "$AWS_REGION"
```

## 5. IAM Trust Policy JSON

The trust policy allows the Lambda service to assume the IAM execution role.

Create `trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Create the role if it does not already exist:

```bash
aws iam create-role \
  --role-name lambda_execution_role \
  --assume-role-policy-document file://trust-policy.json
```

Attach the AWS-managed policy that allows Lambda to write execution logs to CloudWatch:

```bash
aws iam attach-role-policy \
  --role-name lambda_execution_role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

## 6. IAM Permissions Policy JSON

Go to IAM and create a policy:
* Read objects from the source bucket.
* Write objects to the private destination bucket.
* Insert log records into DynamoDB.

Create `lambda-permissions.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadSourceObjects",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject"
      ],
      "Resource": "arn:aws:s3:::datacenter-public-186065528/*"
    },
    {
      "Sid": "WriteDestinationObjects",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::datacenter-private-186065528/*"
    },
    {
      "Sid": "WriteDynamoDBLogs",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem"
      ],
      "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/datacenter-S3CopyLogs"
    }
  ]
}
```

Replace `REGION` and `ACCOUNT_ID` in the DynamoDB ARN with your actual AWS region and account ID before applying the policy.

For example, you can generate the policy using your current CLI configuration:

```bash
python3 - <<'PY'
import json
import os

region = os.environ["AWS_REGION"]
account_id = os.environ["ACCOUNT_ID"]

policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ReadSourceObjects",
            "Effect": "Allow",
            "Action": ["s3:GetObject"],
            "Resource": "arn:aws:s3:::datacenter-public-186065528/*"
        },
        {
            "Sid": "WriteDestinationObjects",
            "Effect": "Allow",
            "Action": ["s3:PutObject"],
            "Resource": "arn:aws:s3:::datacenter-private-186065528/*"
        },
        {
            "Sid": "WriteDynamoDBLogs",
            "Effect": "Allow",
            "Action": ["dynamodb:PutItem"],
            "Resource": (
                f"arn:aws:dynamodb:{region}:{account_id}:"
                "table/datacenter-S3CopyLogs"
            )
        }
    ]
}

with open("lambda-permissions.json", "w") as f:
    json.dump(policy, f, indent=2)

print("Created lambda-permissions.json")
PY
```

Attach the permissions as an inline policy:

```bash
aws iam put-role-policy \
  --role-name lambda_execution_role \
  --policy-name datacenter-copy-permissions \
  --policy-document file://lambda-permissions.json
```

Verify the inline policy:

```bash
aws iam get-role-policy \
  --role-name lambda_execution_role \
  --policy-name datacenter-copy-permissions
```

These permissions are limited to the lab's source bucket, destination bucket, and DynamoDB table. The separate AWS-managed policy provides CloudWatch logging permissions.

## 7. Configure the Lambda Function

The lab's Python source file is `/root/lambda-function.py`. It should use the destination bucket name `datacenter-private-186065528` and DynamoDB table `datacenter-S3CopyLogs`.

Check that the placeholders in the supplied source code have been replaced before packaging it.

Create a deployment package:

```bash
mkdir -p /tmp/datacenter-lambda

cp /root/lambda-function.py \
  /tmp/datacenter-lambda/lambda_function.py

cd /tmp/datacenter-lambda

zip /tmp/datacenter-copyfunction.zip lambda_function.py
```

Get the IAM role ARN:

```bash
ROLE_ARN=$(aws iam get-role \
  --role-name lambda_execution_role \
  --query 'Role.Arn' \
  --output text)

echo "$ROLE_ARN"
```

Create the function if it does not already exist:

```bash
aws lambda create-function \
  --function-name datacenter-copyfunction \
  --runtime python3.12 \
  --handler lambda_function.lambda_handler \
  --role "$ROLE_ARN" \
  --zip-file fileb:///tmp/datacenter-copyfunction.zip \
  --timeout 10 \
  --memory-size 128 \
  --region "$AWS_REGION"
```

If the function already exists, update its code instead:

```bash
aws lambda update-function-code \
  --function-name datacenter-copyfunction \
  --zip-file fileb:///tmp/datacenter-copyfunction.zip \
  --region "$AWS_REGION"
```

Verify the function configuration:

```bash
aws lambda get-function-configuration \
  --function-name datacenter-copyfunction \
  --region "$AWS_REGION"
```

Wait for the function's update to finish before testing or changing its configuration again.

## 8. Configure the S3 Event Notification

First, retrieve the Lambda function ARN:

```bash
LAMBDA_ARN=$(aws lambda get-function \
  --function-name datacenter-copyfunction \
  --query 'Configuration.FunctionArn' \
  --output text \
  --region "$AWS_REGION")

echo "$LAMBDA_ARN"
```

Allow the source S3 bucket to invoke Lambda:

```bash
aws lambda add-permission \
  --function-name datacenter-copyfunction \
  --statement-id allow-public-bucket-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::datacenter-public-186065528 \
  --source-account "$ACCOUNT_ID" \
  --region "$AWS_REGION"
```

If the statement already exists, inspect the existing Lambda resource policy instead of adding a duplicate statement.

Create `notification.json`:

```json
{
  "LambdaFunctionConfigurations": [
    {
      "Id": "datacenter-copy-trigger",
      "LambdaFunctionArn": "REPLACE_WITH_LAMBDA_ARN",
      "Events": [
        "s3:ObjectCreated:*"
      ]
    }
  ]
}
```

Insert the actual Lambda ARN into the JSON file:

```bash
python3 - <<'PY'
import json
import os

config = {
    "LambdaFunctionConfigurations": [
        {
            "Id": "datacenter-copy-trigger",
            "LambdaFunctionArn": os.environ["LAMBDA_ARN"],
            "Events": ["s3:ObjectCreated:*"]
        }
    ]
}

with open("notification.json", "w") as f:
    json.dump(config, f, indent=2)

print("Created notification.json")
PY
```

Apply the notification configuration:

```bash
aws s3api put-bucket-notification-configuration \
  --bucket datacenter-public-186065528 \
  --notification-configuration file://notification.json \
  --region "$AWS_REGION"
```

Verify the trigger:

```bash
aws s3api get-bucket-notification-configuration \
  --bucket datacenter-public-186065528 \
  --region "$AWS_REGION"
```

**Important:** `put-bucket-notification-configuration` replaces the bucket's existing notification configuration. If other notifications already exist, preserve them in `notification.json` before applying the change.

## 9. Test the Workflow

Upload the test file:

```bash
aws s3 cp /root/sample.zip \
  s3://datacenter-public-186065528/sample.zip \
  --region "$AWS_REGION"
```

### Verify the copied file

```bash
aws s3api head-object \
  --bucket datacenter-private-186065528 \
  --key sample.zip \
  --region "$AWS_REGION"
```

If the object exists, the command returns its metadata.

### Verify the DynamoDB logs

```bash
aws dynamodb scan \
  --table-name datacenter-S3CopyLogs \
  --region "$AWS_REGION" \
  --query 'Items'
```

Check for a log entry corresponding to `sample.zip`. The available fields depend on the Lambda function's logging implementation.

### Check Lambda execution logs

```bash
aws logs describe-log-streams \
  --log-group-name /aws/lambda/datacenter-copyfunction \
  --order-by LastEventTime \
  --descending \
  --limit 5 \
  --region "$AWS_REGION"
```

To view recent log events, use the AWS Console or retrieve events from the relevant log stream using `aws logs get-log-events`.

## 10. Troubleshooting

| Problem                      | What to check                                                                       |
| ---------------------------- | ----------------------------------------------------------------------------------- |
| Lambda is not triggered      | S3 notification configuration and Lambda resource-based permission                  |
| `AccessDenied` while reading | IAM `s3:GetObject` permission and source bucket policy                              |
| `AccessDenied` while copying | IAM `s3:PutObject` permission for the destination bucket                            |
| DynamoDB logging fails       | Table name, partition key, region, and `dynamodb:PutItem` permission                |
| Lambda times out             | Function logs, network access, and configured timeout                               |
| Copied object is missing     | Lambda invocation status and CloudWatch logs                                        |
| Public policy is rejected    | S3 Block Public Access settings and applicable account or organization restrictions |

## 🔐 Security Best Practices

* Keep the destination bucket private.
* Use least-privilege IAM permissions.
* Never upload AWS access keys, secret keys, or passwords to GitHub.
* Use public bucket policies only when required for a specific learning exercise.
* Avoid exposing sensitive information through publicly readable objects.
* Delete unnecessary AWS resources after completing the lab to reduce costs.

## 📚 Key Learnings

* Creating S3 bucket policies.
* Configuring IAM trust and permissions policies.
* Automating S3 operations using AWS Lambda.
* Triggering Lambda from S3 object creation events.
* Recording operation logs in DynamoDB.
* Testing and troubleshooting AWS serverless applications using CLI commands.

## 🏁 Conclusion

This lab combines Amazon S3, AWS Lambda, IAM, and DynamoDB to demonstrate an event-driven file-copy workflow.

**Day 46 of my 50 Days AWS Practice Lab Challenge — Zero to Hero!** ☁️🚀
