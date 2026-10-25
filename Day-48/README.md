# Day 48: Deploy an AWS Lambda Function Using CloudFormation

## Overview

In this lab, we use AWS CloudFormation to create an AWS Lambda function
and its IAM execution role. We then invoke the function using the AWS
CLI and verify the response.

## Lab Requirements

  Component                Configuration
  ------------------------ ----------------------------
  CloudFormation stack     `devops-lambda-app`
  Template file            `/root/devops-lambda.yml`
  Lambda function          `devops-lambda`
  Runtime                  Python 3.12
  Handler                  `index.lambda_handler`
  Memory                   128 MB
  Timeout                  10 seconds
  IAM role                 `lambda_execution_role`
  Expected status code     `200`
  Expected response body   `Welcome to KKE AWS Labs!`

> **Note:** This template uses the inline `ZipFile` property, so a
> separate ZIP package or S3 code bucket is not required.

## Prerequisites

-   Access to the AWS lab environment through the `aws-client` host.
-   AWS CLI configured with the lab-provided credentials.
-   Permissions to create CloudFormation stacks, Lambda functions, IAM
    roles, and related resources.

Check the AWS identity and configured region:

``` bash
aws sts get-caller-identity
aws configure get region
```

If the region is not configured, set the region required by your lab.
For example:

``` bash
export AWS_DEFAULT_REGION=us-east-1
```

Do not add AWS access keys or secret keys to this repository.

## Step 1: Create the CloudFormation template

Create `/root/devops-lambda.yml`:

``` bash
cat > /root/devops-lambda.yml <<'EOF'
AWSTemplateFormatVersion: '2010-09-09'
Description: Day 48 - Deploy Lambda function using CloudFormation

Resources:
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: lambda_execution_role
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - lambda.amazonaws.com
            Action:
              - sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

  DevOpsLambda:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: devops-lambda
      Runtime: python3.12
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      MemorySize: 128
      Timeout: 10
      Code:
        ZipFile: |
          def lambda_handler(event, context):
              return {
                  'statusCode': 200,
                  'body': 'Welcome to KKE AWS Labs!'
              }
EOF
```

### What the template does

-   Creates the IAM role `lambda_execution_role`.
-   Allows the Lambda service to assume the role.
-   Attaches `AWSLambdaBasicExecutionRole` so the function can write
    logs to CloudWatch.
-   Creates the `devops-lambda` function using Python 3.12.
-   Sets memory to 128 MB and timeout to 10 seconds.
-   Returns status code `200` and the required response body.

## Step 2: Validate the template

``` bash
aws cloudformation validate-template \
  --template-body file:///root/devops-lambda.yml
```

Resolve any validation errors before deploying the stack.

## Step 3: Deploy the stack

``` bash
aws cloudformation deploy \
  --template-file /root/devops-lambda.yml \
  --stack-name devops-lambda-app \
  --capabilities CAPABILITY_NAMED_IAM
```

`CAPABILITY_NAMED_IAM` acknowledges that the template creates an IAM
role with a specific name.

Wait for deployment to finish successfully.

## Step 4: Verify the CloudFormation stack

``` bash
aws cloudformation describe-stacks \
  --stack-name devops-lambda-app \
  --query 'Stacks[0].StackStatus' \
  --output text
```

Expected result:

``` text
CREATE_COMPLETE
```

If the stack was previously created and updated, the status may instead
be `UPDATE_COMPLETE`.

## Step 5: Verify the Lambda configuration

``` bash
aws lambda get-function-configuration \
  --function-name devops-lambda \
  --query '{FunctionName:FunctionName,Runtime:Runtime,Handler:Handler,MemorySize:MemorySize,Timeout:Timeout,Role:Role}' \
  --output json
```

Confirm that the output shows:

-   Function name: `devops-lambda`
-   Runtime: `python3.12`
-   Handler: `index.lambda_handler`
-   Memory: `128`
-   Timeout: `10`
-   IAM role ARN ending in `role/lambda_execution_role`

## Step 6: Invoke and test the function

Some older AWS CLI versions do not support `--cli-binary-format`. Use
this invocation command in that environment:

``` bash
aws lambda invoke \
  --function-name devops-lambda \
  --payload '{}' \
  /root/devops-lambda-response.json
```

Inspect the returned function payload:

``` bash
cat /root/devops-lambda-response.json
```

Expected payload:

``` json
{
  "statusCode": 200,
  "body": "Welcome to KKE AWS Labs!"
}
```

The invocation command also prints metadata. Its `StatusCode` indicates
the Lambda invocation result, while the JSON file contains the
function's own response object.

## Troubleshooting

### `Unknown options: --cli-binary-format`

The installed AWS CLI is likely an older version. Remove
`--cli-binary-format raw-in-base64-out` and use the invocation command
shown in Step 6.

### `Insufficient permissions` or `AccessDenied`

Check whether the lab credentials have permissions to create
CloudFormation, Lambda, and IAM resources. Do not broaden permissions
beyond what the lab account authorizes.

### IAM role already exists

The role name `lambda_execution_role` must be unique in the AWS account.
Check whether an existing role is present and whether it is safe and
permitted to reuse it. Do not delete shared or pre-existing lab
resources without confirming they are no longer needed.

### Stack creation failed

Inspect the stack events to identify the resource and reason for
failure:

``` bash
aws cloudformation describe-stack-events \
  --stack-name devops-lambda-app \
  --query 'StackEvents[].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table
```

## Cleanup (Optional)

Only run cleanup if you no longer need the lab resources and the stack
belongs to this exercise:

``` bash
aws cloudformation delete-stack \
  --stack-name devops-lambda-app
```

Wait for deletion to complete:

``` bash
aws cloudformation wait stack-delete-complete \
  --stack-name devops-lambda-app
```

## Completion Checklist

-   [ ] Created `/root/devops-lambda.yml`.
-   [ ] Validated the CloudFormation template.
-   [ ] Deployed stack `devops-lambda-app`.
-   [ ] Created Lambda function `devops-lambda`.
-   [ ] Configured Python runtime, 128 MB memory, and 10-second timeout.
-   [ ] Used IAM role `lambda_execution_role`.
-   [ ] Invoked the function and verified status code `200`.
-   [ ] Verified response body `Welcome to KKE AWS Labs!`.

## Summary

This lab demonstrates how to define and deploy an AWS Lambda function
and its IAM execution role using AWS CloudFormation. The function is
tested through the AWS CLI to confirm the required response.
