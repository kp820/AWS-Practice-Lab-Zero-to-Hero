# Day 47: Implement Priority Queuing with Amazon SNS, SQS, Lambda, and CloudFormation

## Overview

In Day 47 of my 50 Days AWS Practice Lab Challenge, I deployed an AWS message-priority workflow using AWS CloudFormation.

A hands-on AWS lab by TriCloud Ops. We use one AWS CloudFormation stack to build a priority queuing system: messages are published to an SNS topic, routed into a high-priority or low-priority SQS queue, and a Lambda function processes the high-priority messages first.

- `datacenter-High-Priority-Queue`
- `datacenter-Low-Priority-Queue`

SNS subscription filter policies route messages to the appropriate queue based on the `priority` message attribute. A Lambda function named `datacenter-priorities-queue-function` consumes messages from the queues. The supplied Lambda code is located at `/root/index.py` on the AWS client host.

The CloudFormation stack is named `datacenter-priority-stack`, and its template is created at `/root/datacenter-priority-stack.yml`.

> **Priority behavior:** Separate SQS queues do not automatically create a global priority order. The Lambda code must explicitly check and process the high-priority queue before processing the low-priority queue. Review `/root/index.py` to confirm that behavior before testing.

## Architecture

```text
Publisher
   |
   v
SNS Topic: datacenter-Priority-Queues-Topic
   |
   +-- priority=high --> SQS: datacenter-High-Priority-Queue --+
   |                                                        |
   +-- priority=low ---> SQS: datacenter-Low-Priority-Queue --+--> Lambda
                                                               datacenter-priorities-queue-function
                                                               (checks high priority first)
```

## Resources Created

| Resource | Name / configuration |
|---|---|
| CloudFormation stack | `datacenter-priority-stack` |
| High-priority SQS queue | `datacenter-High-Priority-Queue` |
| Low-priority SQS queue | `datacenter-Low-Priority-Queue` |
| SNS topic | `datacenter-Priority-Queues-Topic` |
| Lambda function | `datacenter-priorities-queue-function` |
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
**Step-by-step**
## Step 1: Check your setup

Run these commands on the AWS client host:

```bash
aws sts get-caller-identity
aws configure get region
ls -l /root/index.py
cat /root/index.py
```

The first command confirms which AWS identity is active. The second displays the configured default region.

If the region command returns no value, set a region for the current shell, for example:

```bash
export AWS_REGION=us-east-1
```

## Step 2: Create or select your S3 artifact bucket

If you already created an artifact bucket, use that bucket. Otherwise, create a globally unique bucket in us-east-1.

Create S3 global bucket by below command as well

```bash
aws s3api create-bucket \
  --bucket datacenter-priority-artifacts-YOUR-UNIQUE-ID \
  --region us-east-1
```
Replace YOUR-UNIQUE-ID with a unique suffix before running the command.

Set (export) the real bucket name:

```bash
export ARTIFACT_BUCKET=datacenter-priority-artifacts-YOUR-UNIQUE-ID
```
Verify the bucket:
```bash
aws s3api head-bucket \
  --bucket "$ARTIFACT_BUCKET" \
  --region us-east-1
```
## Step 3: Package and upload the Lambda code

Create the ZIP file from your existing Python code:

```bash
cd /root

zip -j -FS /root/datacenter-priorities-function.zip /root/index.py
```
Upload the zip file in the s3 bucket
```bash
aws s3 cp \
  /root/datacenter-priorities-function.zip \
  "s3://${ARTIFACT_BUCKET}/datacenter-priorities-function.zip" \
  --region us-east-1
```
## Step 4: Create the CloudFormation Template

Create the required file:
```bash
touch /root/datacenter-priority-stack.yml
```
open the yml file by VI editor then press i to edit mode 

```bash
vi /root/datacenter-priority-stack.yml
```
Copy the cloudformation template and paste in yml then press Esc and type qw to save the file
```bash
AWSTemplateFormatVersion: '2010-09-09'
Description: Priority queuing with SNS, SQS and Lambda (datacenter lab)

Resources:
  # ---------- SQS queues ----------
  HighPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: datacenter-High-Priority-Queue

  LowPriorityQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: datacenter-Low-Priority-Queue

  # ---------- SNS topic ----------
  PriorityTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: datacenter-Priority-Queues-Topic

  # ---------- Allow SNS to write into the queues ----------
  HighQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref HighPriorityQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt HighPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PriorityTopic

  LowQueuePolicy:
    Type: AWS::SQS::QueuePolicy
    Properties:
      Queues:
        - !Ref LowPriorityQueue
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: sns.amazonaws.com
            Action: sqs:SendMessage
            Resource: !GetAtt LowPriorityQueue.Arn
            Condition:
              ArnEquals:
                aws:SourceArn: !Ref PriorityTopic

  # ---------- SNS -> SQS subscriptions (routed by the "priority" attribute) ----------
  HighQueueSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityTopic
      Protocol: sqs
      Endpoint: !GetAtt HighPriorityQueue.Arn
      FilterPolicy:
        priority:
          - high

  LowQueueSubscription:
    Type: AWS::SNS::Subscription
    Properties:
      TopicArn: !Ref PriorityTopic
      Protocol: sqs
      Endpoint: !GetAtt LowPriorityQueue.Arn
      FilterPolicy:
        priority:
          - low

  # ---------- Customer managed policy (SQS + SNS access) ----------
  LambdaCustomManagedPolicy:
    Type: AWS::IAM::ManagedPolicy
    Properties:
      ManagedPolicyName: datacenter-lambda-sqs-sns-policy
      Description: Allows the Lambda function to use the priority queues and topic
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Action:
              - sqs:ReceiveMessage
              - sqs:DeleteMessage
              - sqs:GetQueueAttributes
              - sqs:GetQueueUrl
              - sqs:SendMessage
            Resource:
              - !GetAtt HighPriorityQueue.Arn
              - !GetAtt LowPriorityQueue.Arn
          - Effect: Allow
            Action:
              - sns:Publish
              - sns:Subscribe
            Resource: !Ref PriorityTopic

  # ---------- IAM role for Lambda ----------
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
        - !Ref LambdaCustomManagedPolicy

  # ---------- Lambda function ----------
  PriorityFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: datacenter-priorities-queue-function
      Runtime: python3.12
      Handler: index.lambda_handler   # change if your /root/index.py uses a different function name
      Role: !GetAtt LambdaExecutionRole.Arn
      MemorySize: 128
      Timeout: 10
      Environment:
        Variables:
          HIGH_PRIORITY_QUEUE_URL: !Ref HighPriorityQueue
          LOW_PRIORITY_QUEUE_URL: !Ref LowPriorityQueue
      Code:
        ZipFile: |
            # PASTE_INDEX_PY_HERE

Outputs:
  TopicArn:
    Value: !Ref PriorityTopic
  HighQueueUrl:
    Value: !Ref HighPriorityQueue
  LowQueueUrl:
    Value: !Ref LowPriorityQueue
  FunctionName:
    Value: !Ref PriorityFunction

```

## Step 5: Validate and deploy the stack

Validate the template first:

```bash
aws cloudformation validate-template \
  --template-body file:///root/datacenter-priority-stack.yml \
  --region us-east-1
```

If validation succeeds, deploy:

```bash
  aws cloudformation deploy \
  --template-file /root/datacenter-priority-stack.yml \
  --stack-name datacenter-priority-stack \
  --parameter-overrides \
    LambdaCodeBucket="$ARTIFACT_BUCKET" \
    LambdaCodeKey=datacenter-priorities-function.zip \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```
Wait for deployment to finish. If it fails, inspect the failure events:

## Step 6: Verify all resources
Check stack status:
Run:

```bash
aws cloudformation describe-stacks \
  --stack-name datacenter-priority-stack \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Expected status:

CREATE_COMPLETE

## Step 7: Verify the queues, topic, Lambda, and IAM policy

You can manual verify 
1. SNS topic
2. SQS service
3. Lambda function
4. IAM role and custom manage policy are attached on the role
5. Lambda function timeout should be 10 seconds
6. Expected configuration includes 128 MB memory and a 10-second timeout.

## Step 8: Verify the lab test

copy the test command from the lab and hit in the aws-client 

## Step 9: Check the queue message counts

Run:

```bash
HIGH_QUEUE_URL=$(aws sqs get-queue-url \
  --queue-name datacenter-High-Priority-Queue \
  --region "$AWS_REGION" \
  --query QueueUrl --output text)

LOW_QUEUE_URL=$(aws sqs get-queue-url \
  --queue-name datacenter-Low-Priority-Queue \
  --region "$AWS_REGION" \
  --query QueueUrl --output text)

aws sqs get-queue-attributes \
  --queue-url "$HIGH_QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
  --region "$AWS_REGION"

aws sqs get-queue-attributes \
  --queue-url "$LOW_QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
  --region "$AWS_REGION"
```

Expected result, assuming no messages have been consumed:
- High-priority queue: approximately 2 available messages.
- Low-priority queue: approximately 2 available messages.
SQS counts are approximate, so allow a few seconds after publishing.

## Step 10: Test Lambda

Your Lambda checks the high-priority queue first and processes at most one message per invocation.

```bash
aws lambda invoke \
  --function-name datacenter-priorities-queue-function \
  --payload '{}' \
  /root/lambda-test-1.json \
  --region us-east-1

cat /root/lambda-test-1.json
```
Your CLI should accept this command. The Lambda function should process one high-priority message and delete it.

## Step 11: Process the remaining messages

Run this loop:
```bash
for i in 2 3 4; do
  aws lambda invoke \
    --function-name datacenter-priorities-queue-function \
    --payload '{}' \
    "/root/lambda-test-$i.json" \
    --region us-east-1

  echo "Lambda response $i:"
  cat "/root/lambda-test-$i.json"
  echo
done
```

Expected responses should include messages being deleted. Since your function checks the high-priority queue first, it should process both high-priority messages before the low-priority messages, provided they are visible when polled.

## Step 12: Verify queue counts again

```bash
aws sqs get-queue-attributes \
  --queue-url "$HIGH_QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
  --region us-east-1

aws sqs get-queue-attributes \
  --queue-url "$LOW_QUEUE_URL" \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible \
  --region us-east-1
```

Expected after all four messages are processed: both queues eventually show 
ApproximateNumberOfMessages: 0 
ApproximateNumberOfMessagesNotVisible: 0.


Excellent! 🎉 Your Datacenter Priority Queue lab is successfully working. Both SQS queues are empty after processing, and Lambda successfully deleted the messages.

=============




