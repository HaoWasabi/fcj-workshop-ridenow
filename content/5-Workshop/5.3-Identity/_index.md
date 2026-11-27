---
title : "Identity"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

This page creates the two IAM roles that the EKS cluster needs and five customer managed policies in the IAM console. IAM is a global service, so the region you open in the console does not matter.

## Set variables

Run this block in your Bash shell. It repeats the shared variables from [5.2](../5.2-Prerequiste/). The policy JSON below uses your account ID and region, so keep the output in view.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
```

Expected result (the account ID is masked):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
```

Wherever a policy below shows `{ACCOUNT_ID}` or `{AWS_REGION}`, replace the whole token, including the braces, with the value printed above.

## What it creates

| AWS resource type | Name |
|---|---|
| IAM role | `ridenow-prod-eks-cluster-role` |
| IAM role | `ridenow-prod-eks-fargate-pod-execution-role` |
| IAM customer managed policy | `ridenow-prod-app-access` |
| IAM customer managed policy | `ridenow-prod-trip-service-messaging` |
| IAM customer managed policy | `ridenow-prod-payment-service-messaging` |
| IAM customer managed policy | `ridenow-prod-matching-service-messaging` |
| IAM customer managed policy | `ridenow-prod-notification-worker-messaging` |

## Depends on

Nothing. You need only the tools and identity from 5.2. The seven resources can be created in any order.

## Create the resources

### 1. Role `ridenow-prod-eks-cluster-role`

1. Open **IAM**, choose **Roles**, then choose **Create role**.
2. Set **Trusted entity type** to **Custom trust policy**.
3. Delete the default JSON in the **Custom trust policy** box and paste the JSON below. Choose **Next**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }
  ]
}
```

4. Search for each of these AWS managed policies and tick it: `AmazonEKSClusterPolicy`, `AmazonEKSComputePolicy`, `AmazonEKSBlockStoragePolicyV2`, `AmazonEKSLoadBalancingPolicy`, `AmazonEKSNetworkingPolicy`. Choose **Next**.
5. Set **Role name** to `ridenow-prod-eks-cluster-role`. Leave **Description** at the default.
6. Choose **Create role**.

### 2. Role `ridenow-prod-eks-fargate-pod-execution-role`

1. Open **IAM**, choose **Roles**, then choose **Create role**.
2. Set **Trusted entity type** to **Custom trust policy**.
3. Delete the default JSON and paste the JSON below. Choose **Next**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "eks-fargate-pods.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

4. Search for `AmazonEKSFargatePodExecutionRolePolicy` and tick it. Choose **Next**.
5. Set **Role name** to `ridenow-prod-eks-fargate-pod-execution-role`. Leave **Description** at the default.
6. Choose **Create role**.

### 3. Policy `ridenow-prod-app-access`

1. Open **IAM**, choose **Policies**, then choose **Create policy**.
2. Set **Policy editor** to **JSON**, delete the default JSON, and paste the JSON below.
3. Replace each `{ACCOUNT_ID}` with your account ID. Leave the `*` region parts unchanged. Choose **Next**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcrAuthToken",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "EcrPullRideNow",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:*:{ACCOUNT_ID}:repository/ridenow/*"
    },
    {
      "Sid": "ReadRideNowSecrets",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:*:{ACCOUNT_ID}:secret:ridenow-prod-*"
    }
  ]
}
```

4. Set **Policy name** to `ridenow-prod-app-access`.
5. Set **Description** to the text below. The description cannot be edited later, so type it exactly.

```text
Scoped runtime access for RideNow pods via IRSA: pull images, read DB secrets, and (where present) use the environment's queues and event bus.
```

6. Choose **Create policy**.

### 4. Policy `ridenow-prod-trip-service-messaging`

1. Open **IAM**, choose **Policies**, then choose **Create policy**.
2. Set **Policy editor** to **JSON**, delete the default JSON, and paste the JSON below.
3. Replace each `{ACCOUNT_ID}` and `{AWS_REGION}` with your values. Choose **Next**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishToRideNowTopic",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:{AWS_REGION}:{ACCOUNT_ID}:ridenow-prod-events"
    }
  ]
}
```

4. Set **Policy name** to `ridenow-prod-trip-service-messaging`.
5. Set **Description** to `trip-service publishes trip events to the RideNow topic.`
6. Choose **Create policy**.

### 5. Policy `ridenow-prod-payment-service-messaging`

1. Open **IAM**, choose **Policies**, then choose **Create policy**.
2. Set **Policy editor** to **JSON**, delete the default JSON, and paste the JSON below.
3. Replace each `{ACCOUNT_ID}` and `{AWS_REGION}` with your values. Choose **Next**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishToRideNowTopic",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:{AWS_REGION}:{ACCOUNT_ID}:ridenow-prod-events"
    },
    {
      "Sid": "ConsumePaymentQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:{AWS_REGION}:{ACCOUNT_ID}:ridenow-prod-payment"
    }
  ]
}
```

4. Set **Policy name** to `ridenow-prod-payment-service-messaging`.
5. Set **Description** to `payment-service publishes payment events and consumes the payment queue.`
6. Choose **Create policy**.

### 6. Policy `ridenow-prod-matching-service-messaging`

1. Open **IAM**, choose **Policies**, then choose **Create policy**.
2. Set **Policy editor** to **JSON**, delete the default JSON, and paste the JSON below.
3. Replace each `{ACCOUNT_ID}` and `{AWS_REGION}` with your values. Choose **Next**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishToRideNowTopic",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:{AWS_REGION}:{ACCOUNT_ID}:ridenow-prod-events"
    },
    {
      "Sid": "ConsumeMatchingQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:{AWS_REGION}:{ACCOUNT_ID}:ridenow-prod-matching"
    }
  ]
}
```

4. Set **Policy name** to `ridenow-prod-matching-service-messaging`.
5. Set **Description** to `matching-service publishes match events and consumes the matching queue.`
6. Choose **Create policy**.

### 7. Policy `ridenow-prod-notification-worker-messaging`

1. Open **IAM**, choose **Policies**, then choose **Create policy**.
2. Set **Policy editor** to **JSON**, delete the default JSON, and paste the JSON below.
3. Replace each `{ACCOUNT_ID}` and `{AWS_REGION}` with your values. Choose **Next**.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ConsumeNotificationsQueue",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:{AWS_REGION}:{ACCOUNT_ID}:ridenow-prod-notifications"
    }
  ]
}
```

4. Set **Policy name** to `ridenow-prod-notification-worker-messaging`.
5. Set **Description** to `notification-worker consumes the notifications queue.`
6. Choose **Create policy**.

## Values to record

The two role ARNs are used when you create the EKS cluster in 5.8. Load them into variables now. The block looks each role up by name.

```bash
CLUSTER_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-eks-cluster-role" \
  --query Role.Arn --output text)
FARGATE_POD_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-eks-fargate-pod-execution-role" \
  --query Role.Arn --output text)

[ -n "$CLUSTER_ROLE_ARN" ] || echo "STOP: CLUSTER_ROLE_ARN is empty. Create the role ridenow-prod-eks-cluster-role first."
[ -n "$FARGATE_POD_ROLE_ARN" ] || echo "STOP: FARGATE_POD_ROLE_ARN is empty. Create the role ridenow-prod-eks-fargate-pod-execution-role first."

echo "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN"
echo "FARGATE_POD_ROLE_ARN=$FARGATE_POD_ROLE_ARN"
```

Expected result (the account ID is masked):

```text
CLUSTER_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-eks-cluster-role
FARGATE_POD_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-eks-fargate-pod-execution-role
```

If a role is missing, the command prints an AWS error and the variable stays empty. Fix the role name and run the block again.

The five policies need no variable now. The messaging policies are attached to service roles in 5.9, where you select them by name in the console.

## Post-deploy steps

No post-deploy step is needed for this page.

## Verify

Run this block:

```bash
aws iam list-policies --scope Local \
  --query "Policies[?starts_with(PolicyName,'ridenow-$ENV-')].PolicyName" --output text | tr '\t' '\n'
echo "$CLUSTER_ROLE_ARN"
echo "$FARGATE_POD_ROLE_ARN"
```

Expected result (the account ID is masked; the policy order can differ). A missing line means that resource was not created or is misspelled:

```text
ridenow-prod-app-access
ridenow-prod-trip-service-messaging
ridenow-prod-payment-service-messaging
ridenow-prod-matching-service-messaging
ridenow-prod-notification-worker-messaging
arn:aws:iam::123456789012:role/ridenow-prod-eks-cluster-role
arn:aws:iam::123456789012:role/ridenow-prod-eks-fargate-pod-execution-role
```

Next: [5.4 Network](../5.4-Network/).
