---
title : "Prerequisites"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

This page lists what you need before you build RideNow on AWS and sets the variables that every later page uses.

## Tools and access

You need an AWS account and a non-root administrator identity (an IAM user or an IAM Identity Center role with administrator permissions). Do not use the root user.

Install these tools on your machine:

| Tool | Used for |
|---|---|
| AWS CLI v2 | Look up resource values and run the checks on each page |
| docker | Build and push the service images to ECR |
| kubectl | Work with the EKS cluster |
| helm | Install the AWS Load Balancer Controller and External Secrets |
| jq | Read JSON output in the checks |
| Bash | Run every command block (on Windows, use WSL) |

## Check your identity

Run this command with your administrator credentials:

```bash
aws sts get-caller-identity
```

The result shows your account and your user or role. The values below are masked examples:

```text
{
    "UserId": "AIDAEXAMPLEEXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/workshop-admin"
}
```

If the command fails or shows the root user, fix your credentials before you continue.

## Region and environment

The whole workshop builds the `prod` environment in the `ap-southeast-1` region. Set `ap-southeast-1` as the default region of your AWS CLI profile and open the AWS console in the same region.

## Cost warning

Several resources bill by the hour from the moment you create them, whether or not anything uses them:

- the EKS control plane
- the NAT gateways
- the Multi-AZ RDS instances
- the ElastiCache replication group

Follow the workshop in one sitting if you can, and finish with 5.11 Clean up so the charges stop.

## Conventions

- **Auto_Variable.** When one CLI command can look up a value (an account ID, a VPC ID, a queue URL), the page saves it into a Bash variable for you. You never copy these values by hand. Each page starts with a block that sets the variables the page needs, followed by an expected result with masked values.
- **Placeholder.** A few values cannot be looked up by any command, for example a choice you make yourself. The page writes these in the form `<GITHUB_REPO>` and tells you where to copy the value from. Replace the whole token, including the angle brackets, with your value.
- **Notes file.** Keep a text file open while you work. Write down every value you copy by hand for a Placeholder, so you can paste it again on a later page.
- **Bash only.** All commands use Bash syntax. PowerShell syntax is not supported. On Windows, run the commands in Bash inside WSL.
- **Session only.** Variables last only for the current shell session. After you open a new shell, run the variables block at the top of the page you are working on again.

## Set the shared variables

These variables are used by every later page. Run the block, then compare the output with the expected result:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod
IMAGE_TAG=v1.0.0

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "IMAGE_TAG=$IMAGE_TAG"
```

Expected result (the account ID is masked):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
IMAGE_TAG=v1.0.0
```

If either line starts with `STOP`, fix that problem and run the block again.