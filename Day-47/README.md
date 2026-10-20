# Day 47: Implement Priority Queuing with Amazon SNS, SQS, Lambda, and CloudFormation

## Overview

In Day 47 of my 50 Days AWS Practice Lab Challenge, I deployed an AWS message-priority workflow using AWS CloudFormation.

The solution uses one Amazon SNS topic to receive published messages and two Amazon SQS queues to separate messages by priority:

- `nautilus-High-Priority-Queue`
- `nautilus-Low-Priority-Queue`

SNS subscription filter policies route messages to the appropriate queue based on the `priority` message attribute. A Lambda function named `nautilus-priorities-queue-function` consumes messages from the queues. The supplied Lambda code is located at `/root/index.py` on the AWS client host.

The CloudFormation stack is named `nautilus-priority-stack`, and its template is created at `/root/nautilus-priority-stack.yml`.

> **Priority behavior:** Separate SQS queues do not automatically create a global priority order. The Lambda code must explicitly check and process the high-priority queue before processing the low-priority queue. Review `/root/index.py` to confirm that behavior before testing.

## Architecture

```text
Publisher
   |
   v
SNS Topic: nautilus-Priority-Queues-Topic
   |
   +-- priority=high --> SQS: nautilus-High-Priority-Queue --+
   |                                                        |
   +-- priority=low ---> SQS: nautilus-Low-Priority-Queue --+--> Lambda
                                                               nautilus-priorities-queue-function
                                                               (checks high priority first)
```

## Resources Created

| Resource | Name / configuration |
|---|---|
| CloudFormation stack | `nautilus-priority-stack` |
| High-priority SQS queue | `nautilus-High-Priority-Queue` |
| Low-priority SQS queue | `nautilus-Low-Priority-Queue` |
| SNS topic | `nautilus-Priority-Queues-Topic` |
| Lambda function | `nautilus-priorities-queue-function` |
| Lambda execution role | `lambda_execution_role` |
| Lambda runtime | Python 3.12 |
| Lambda memory | 128 MB |
| Lambda timeout | 10 seconds |
| SNS filter attributes | `priority=high` and `priority=low` |

## Prerequisites

- Access to the AWS client host.
- AWS CLI already configured with credentials and a default region.
- The provided Lambda source file at `/root/index.py`.
- Permission to create CloudFormation, IAM, Lambda, SNS, and SQS resources.
- The required role name `lambda_execution_role` must not already be in use by an unrelated resource in the account.

Do not paste AWS access keys or secret credentials into this repository.

## Step 1: Verify AWS CLI Access and the Lambda Source

Run these commands on the AWS client host:

```bash
aws sts get-caller-identity
aws configure get region
ls -l /root/index.py
```

The first command confirms which AWS identity is active. The second displays the configured default region.

If the region command returns no value, set a region for the current shell, for example:

```bash
export AWS_REGION=ap-south-1
```

Use the region assigned to your lab, not necessarily the example above. If a region is configured, you can set it explicitly for consistency:

```bash
export AWS_REGION="$(aws configure get region)"
```

## Step 2: Review the Lambda Code

Inspect the supplied code before deployment:

```bash
sed -n '1,240p' /root/index.py
```

Confirm that the handler function is named `lambda_handler`, because the template uses `index.lambda_handler`.

Also verify that the code reads from the high-priority queue before the low-priority queue. If the code expects environment variables, make sure their names match the template:

- `HIGH_PRIORITY_QUEUE_URL`
- `LOW_PRIORITY_QUEUE_URL`
- `PRIORITY_TOPIC_ARN`

If the supplied code uses different variable names or queue lookup logic, update the template's environment variables or the code to match. Do not assume that merely creating two queues guarantees high-priority-first processing.

## Step 3: Package the Supplied Lambda Code

CloudFormation needs the Lambda deployment package in an artifact location. Create a ZIP file from the supplied Python source:

```bash
cd /root
zip -j /root/index.zip /root/index.py
```

If `zip` is not installed, install it using the package manager available on the AWS client host, or create the ZIP file using an available Python utility.

## Step 4: Create the CloudFormation Template

Create the required file:

```bash
vi /root/nautilus-priority-stack.yml
```

Paste the template from the companion file [`nautilus-priority-stack.yml`](./nautilus-priority-stack.yml).

The template creates both queues, the SNS topic, filtered SNS subscriptions, queue permissions allowing SNS to send messages, the Lambda execution role, and the Lambda function.

The Lambda role includes CloudWatch Logs permissions plus the SQS and SNS permissions needed by the function. The role is named exactly `lambda_execution_role`.

## Step 5: Create an S3 Artifact Bucket

The Lambda ZIP file must be uploaded to S3 so CloudFormation can deploy it. Use a unique bucket name. The following example derives a name from your AWS account and region:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ARTIFACT_BUCKET="nautilus-priority-artifacts-${ACCOUNT_ID}-${AWS_REGION}"

echo "$ARTIFACT_BUCKET"
```

Create the bucket if it does not already exist:

```bash
if [ "$AWS_REGION" = "us-east-1" ]; then
  aws s3api create-bucket --bucket "$ARTIFACT_BUCKET" --region "$AWS_REGION"
else
  aws s3api create-bucket \
    --bucket "$ARTIFACT_BUCKET" \
    --region "$AWS_REGION" \
    --create-bucket-configuration LocationConstraint="$AWS_REGION"
fi
```

S3 bucket names are globally unique. If the name is already taken or the bucket already exists under a different owner, choose another unique name. Do not assume that a bucket you cannot access is yours.

## Step 6: Package the CloudFormation Template

Run:

```bash
aws cloudformation package \
  --template-file /root/nautilus-priority-stack.yml \
  --s3-bucket "$ARTIFACT_BUCKET" \
  --output-template-file /root/nautilus-priority-stack-packaged.yml \
  --region "$AWS_REGION"
```

The packaging command uploads the local Lambda ZIP artifact to S3 and writes a packaged CloudFormation template.

## Step 7: Deploy the Stack

Deploy using the required stack name:

```bash
aws cloudformation deploy \
  --template-file /root/nautilus-priority-stack-packaged.yml \
  --stack-name nautilus-priority-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region "$AWS_REGION"
```

The `CAPABILITY_NAMED_IAM` acknowledgement is required because the template creates an IAM role with a specified name.

Check the stack status:

```bash
aws cloudformation describe-stacks \
  --stack-name nautilus-priority-stack \
  --query 'Stacks[0].StackStatus' \
  --output text \
  --region "$AWS_REGION"
```

The deployment should finish with `CREATE_COMPLETE`. If it fails, inspect the stack events:

```bash
aws cloudformation describe-stack-events \
  --stack-name nautilus-priority-stack \
  --query 'StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table \
  --region "$AWS_REGION"
```

## Step 8: Get the SNS Topic ARN

Retrieve the topic ARN as required by the lab:

```bash
topicarn=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn, 'nautilus-Priority-Queues-Topic')].TopicArn | [0]" \
  --output text \
  --region "$AWS_REGION")

echo "$topicarn"
```

Confirm that the variable contains an ARN before publishing messages.

## Step 9: Publish Test Messages

Publish the high-priority messages first:

```bash
aws sns publish \
  --topic-arn "$topicarn" \
  --message 'High Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}' \
  --region "$AWS_REGION"

aws sns publish \
  --topic-arn "$topicarn" \
  --message 'High Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"high"}}' \
  --region "$AWS_REGION"
```

Publish the low-priority messages:

```bash
aws sns publish \
  --topic-arn "$topicarn" \
  --message 'Low Priority message 1' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}' \
  --region "$AWS_REGION"

aws sns publish \
  --topic-arn "$topicarn" \
  --message 'Low Priority message 2' \
  --message-attributes '{"priority":{"DataType":"String","StringValue":"low"}}' \
  --region "$AWS_REGION"
```

SNS subscription filter policies route messages with `priority=high` to the high-priority queue and messages with `priority=low` to the low-priority queue.

## Step 10: Invoke the Lambda Function

Invoke the function using an empty JSON event if the supplied handler is designed to poll the SQS queues itself:

```bash
aws lambda invoke \
  --function-name nautilus-priorities-queue-function \
  --invocation-type RequestResponse \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/nautilus-priority-response.json \
  --region "$AWS_REGION"

cat /tmp/nautilus-priority-response.json
```

Review the response for errors. If the supplied code expects an event with a different structure, use the event format expected by that code.

## Step 11: Review Lambda Logs and Processing Order

Open **CloudWatch → Log groups → `/aws/lambda/nautilus-priorities-queue-function`** in the AWS Management Console.

Inspect the log stream generated by the invocation. Confirm that the function processes the high-priority messages before processing the low-priority messages.

You can also query recent log events with the CLI:

```bash
aws logs filter-log-events \
  --log-group-name /aws/lambda/nautilus-priorities-queue-function \
  --region "$AWS_REGION"
```

If the invocation processes only a limited number of messages, invoke the function again as appropriate for the supplied code. Verify the actual log output instead of assuming all four messages were processed in one invocation.

### Important Priority Note

SQS standard queues do not provide a priority feature. Priority is implemented here by routing messages to separate queues and having the consumer process the high-priority queue first.

If the Lambda function is instead triggered independently by event source mappings for both queues, AWS does not guarantee that a high-priority message will be processed before a low-priority message. The consumer logic must enforce the desired order.

## Troubleshooting

| Problem | What to check |
|---|---|
| CloudFormation deployment fails | Review stack events and the failed resource's status reason. |
| Lambda code cannot be packaged | Confirm `/root/index.py` exists and the ZIP contains `index.py` at its root. |
| Lambda handler error | Confirm the handler is `index.lambda_handler` and the function exists in `index.py`. |
| SNS messages do not reach a queue | Verify topic subscriptions, the `priority` message attribute, filter policy, and queue policy. |
| Lambda cannot read messages | Verify `lambda_execution_role` has the required SQS permissions and the code uses the correct queue URLs. |
| Low-priority message is processed first | Verify that the Lambda code explicitly checks the high-priority queue first and that another independent consumer is not processing messages. |
| Lambda reports access denied | Review the role's SQS/SNS permissions and CloudWatch Logs permissions. |
| Lambda times out | The function has the required 10-second timeout; review its polling and processing logic. |

## Security and Cost Notes

- Do not commit credentials, access keys, or secret tokens.
- Keep IAM permissions limited to the resources required by the lab.
- SQS, SNS, Lambda, CloudWatch Logs, and S3 may incur charges depending on usage and account free-tier eligibility.
- The artifact bucket is used to store the Lambda deployment package. Keep it if you need it for future deployments, or remove the uploaded artifact and bucket after confirming they are no longer needed.

## Cleanup

Only clean up resources created specifically for this lab and only after saving any logs or evidence you need.

Delete the CloudFormation stack:

```bash
aws cloudformation delete-stack \
  --stack-name nautilus-priority-stack \
  --region "$AWS_REGION"

aws cloudformation wait stack-delete-complete \
  --stack-name nautilus-priority-stack \
  --region "$AWS_REGION"
```

After confirming that the stack has been deleted, optionally remove the artifact bucket if it was created only for this lab and contains no other required objects:

```bash
aws s3 rm "s3://$ARTIFACT_BUCKET" --recursive --region "$AWS_REGION"
aws s3api delete-bucket --bucket "$ARTIFACT_BUCKET" --region "$AWS_REGION"
```

Do not delete an existing bucket or resources belonging to another lab.

## Key Learnings

- Implementing priority routing with SNS subscription filter policies.
- Separating high-priority and low-priority messages into SQS queues.
- Using Lambda to consume messages from SQS.
- Deploying AWS resources using CloudFormation.
- Packaging Lambda code for CloudFormation deployment.
- Applying IAM permissions for SQS, SNS, Lambda, and CloudWatch Logs.
- Verifying processing order through Lambda output and CloudWatch Logs.

## Conclusion

In Day 47 of my 50 Days AWS Practice Lab Challenge, I used AWS CloudFormation to deploy a priority-based messaging workflow using Amazon SNS, Amazon SQS, and AWS Lambda.

SNS filters messages according to their priority attribute and routes them to separate SQS queues. The Lambda consumer must process the high-priority queue before the low-priority queue to enforce the required priority behavior.

This lab demonstrates infrastructure as code, event-driven messaging, IAM permissions, and priority-aware message processing in AWS.

**Day 47 completed — learning AWS one lab at a time! ☁️🚀**
