---
title : "Clean up"
date : 2024-01-01
weight : 11
chapter : false
pre : " <b> 5.11. </b> "
---

This page deletes everything you built in 5.3 to 5.10, so the hourly charges stop. Work from the top: delete what runs inside the cluster first, then delete the AWS resources in the reverse of the order you created them.

## Set variables

Run this block in a new shell. It sets the account, the Region, the cluster name and the VPC ID that the checks on this page use:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "VPC_ID=$VPC_ID"
```

Expected result (the account ID and the VPC ID are masked):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
```

If a line starts with `STOP`, fix that problem and run the block again. If `VPC_ID` prints `None`, the network is already deleted and you can skip the load balancer check.

## What gets deleted

| Group | What is deleted | Created on |
|---|---|---|
| In the cluster | Ingress, the seven workloads, pod security group policies, External Secrets, the Helm releases, the namespaces | 5.8, 5.9 |
| AWS resources | Frontend, IAM roles, security group rules, EKS, KMS keys, Valkey, SNS, SQS, RDS, ECR, network, IAM | 5.3 to 5.10 |

## Delete the in-cluster resources

> **Warning.** Do this before you delete anything in the console. The Ingress creates an Application Load Balancer. That load balancer holds network interfaces in the VPC subnets, so the VPC cannot be deleted while it exists. Deleting the Ingress is what removes the load balancer.

Run these commands from the repository root:

```bash
# Run from the repository root.
kubectl delete -f iac/k8s/30-ingress.yaml --ignore-not-found
for F in 26-realtime-gateway 25-notification-worker 24-matching-service \
         22-trip-service 21-driver-service 20-user-service 23-payment-service; do
  kubectl delete -f "iac/k8s/$F.yaml" --ignore-not-found
done
kubectl delete securitygrouppolicy --all -n ridenow --ignore-not-found
kubectl delete -f iac/k8s/10-external-secrets.yaml --ignore-not-found
kubectl delete namespace ridenow aws-observability --ignore-not-found
helm uninstall external-secrets -n external-secrets
helm uninstall aws-load-balancer-controller -n kube-system
kubectl delete namespace external-secrets --ignore-not-found
```

Wait until the load balancer is gone. This check lists the load balancers in the VPC:

```bash
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?VpcId=='${VPC_ID}'].[LoadBalancerName,State.Code]" --output text
```

Expected result: no output. If a load balancer is still listed, wait a minute and run the check again. In the EC2 console, open **Load Balancers** and confirm that none is tagged `elbv2.k8s.aws/cluster = ridenow-prod`.

If you built the layers from the CloudFormation templates instead of the console, this command prints the stack delete plan (a dry run that deletes nothing). Add `--apply` to delete the stacks, and run the check at the end of this page afterwards:

```bash
bash iac/scripts/teardown.sh "$ENV"
```

## Delete the AWS resources in the console

> **Warning.** Delete the Ingress-created load balancer first (the previous section). If it still exists, step 21 fails with a dependency error and the VPC stays, together with its NAT gateways.

Delete in the order of this table, from top to bottom. A resource goes only after everything that uses it is gone. The order is the reverse of the order you created them, from 5.10 down to 5.3.

| Step | Page | Service console | Delete | Notes |
|------|------|-----------------|--------|-------|
| 1 | 5.10 | CloudFront > **Distributions** | The `RideNow prod frontend` distribution | **Disable** it and wait until **Last modified** shows a date, then **Delete**. Then open **Functions** and delete the SPA rewrite function `ridenow-prod-spa-rewrite`. Then open **Origin access** and delete `ridenow-prod-frontend-oac`. |
| 2 | 5.10 | S3 > **Buckets** | `ridenow-prod-frontend-<account id>-<region>` | **Empty** it (all versions and delete markers), then **Delete**. |
| 3 | 5.10 | IAM > **Roles** | `ridenow-prod-frontend-deploy` (only if you created it) | Delete its inline policy first, then the role. |
| 4 | 5.8 | EC2 > **Security Groups** | The six `ridenow-prod-<service>-service-sg` groups | First remove the six matching inbound rules from `ridenow-prod-db-sg`, then delete the groups. |
| 5 | 5.9 | EC2 > **Security Groups** | The two pod-access inbound rules | On `ridenow-prod-db-sg` (port 5432) and `ridenow-prod-redis-sg` (port 6379), delete the rules whose source is the EKS cluster security group. |
| 6 | 5.9 | IAM > **Roles** | The six IRSA roles: `ridenow-prod-alb-controller`, `ridenow-prod-external-secrets`, `ridenow-prod-trip-service`, `ridenow-prod-payment-service`, `ridenow-prod-matching-service`, `ridenow-prod-notification-worker` | Detach or delete their policies first. |
| 7 | 5.9 | IAM > **Identity providers** | The OIDC provider of the cluster | Delete it after step 6. |
| 8 | 5.9 | IAM > **Policies** | `AWSLoadBalancerControllerIAMPolicy` | Delete it after it is detached from the role in step 6. |
| 9 | 5.8 | EKS > **Clusters** > `ridenow-prod` > **Compute** | The three Fargate profiles | Delete one at a time, in this order: `ridenow-prod-observability`, `ridenow-prod-app`, `ridenow-prod-system`. Wait for each deletion to finish. |
| 10 | 5.8 | EKS > **Clusters** | `ridenow-prod` | Delete it after the profiles are gone. |
| 11 | 5.8 | KMS > **Customer managed keys** | The key with alias `ridenow-prod-eks-secrets` | **Schedule key deletion** (7 days minimum). |
| 12 | 5.7 | ElastiCache > **Valkey caches** | `ridenow-prod-redis` | Delete it. Then open **Subnet groups** and delete the Valkey subnet group. Then in EC2 delete `ridenow-prod-redis-sg`. |
| 13 | 5.7 | SNS > **Subscriptions**, then **Topics** | The three subscriptions, then `ridenow-prod-events` | |
| 14 | 5.7 | SQS > **Queues** | `ridenow-prod-matching`, `ridenow-prod-payment`, `ridenow-prod-notifications` and their three DLQs | Delete each queue before its DLQ. |
| 15 | 5.6 | RDS > **Databases** | The six `ridenow-prod-<service>-db` instances | Take a final snapshot when asked. Deletion protection is off. The master-user secrets in Secrets Manager are removed with the instances. |
| 16 | 5.6 | RDS > **Subnet groups** | The DB subnet group | Delete it after all six instances are gone. |
| 17 | 5.6 | KMS > **Customer managed keys** | The key with alias `ridenow-prod-rds` | **Schedule key deletion**. |
| 18 | 5.5 | ECR > **Repositories** | The seven `ridenow/<service>` repositories | Delete them together with their images. |
| 19 | 5.4 | EC2 > **Security Groups** | `ridenow-prod-db-sg`, then `ridenow-prod-cluster-sg` | Delete them after the RDS, Valkey and EKS resources that use them are gone. |
| 20 | 5.4 | VPC > **NAT gateways** | Both NAT gateways | Wait for **Deleted**. Then open VPC > **Elastic IPs** and **Release** both addresses. |
| 21 | 5.4 | VPC > **Route tables**, **Subnets**, **Internet gateways**, **Your VPCs** | The route tables, the four subnets, the internet gateway (**Detach**, then delete), then `ridenow-prod-vpc` | Delete in this order. |
| 22 | 5.3 | IAM > **Roles**, **Policies** | The two EKS roles, then the app-access policy and the four messaging policies | Last: nothing else uses the 5.3 resources now. |
| 23 | 5.8 | CloudWatch > **Log groups** | `/aws/eks/ridenow/fargate` and the cluster log group | Optional. |

If a console refuses a deletion with a dependency message, it names the resource that still uses it. Delete that one first.

## Check that everything is gone

Run this block. It lists every RideNow resource that still exists:

```bash
left() {
  local OUT
  OUT=$(aws "$@" --output text 2>/dev/null)
  [ -n "$OUT" ] && [ "$OUT" != "None" ] && echo "  $OUT"
}

echo "Still present:"
left eks list-clusters --query "clusters[?@=='${CLUSTER_NAME}']"
left rds describe-db-instances --query "DBInstances[?starts_with(DBInstanceIdentifier, '${CLUSTER_NAME}-')].DBInstanceIdentifier"
left elasticache describe-replication-groups --query "ReplicationGroups[?starts_with(ReplicationGroupId, '${CLUSTER_NAME}-')].ReplicationGroupId"
left sns list-topics --query "Topics[?contains(TopicArn, ':${CLUSTER_NAME}-')].TopicArn"
left sqs list-queues --queue-name-prefix "${CLUSTER_NAME}-" --query 'QueueUrls'
left elbv2 describe-load-balancers --query "LoadBalancers[?VpcId=='${VPC_ID}'].LoadBalancerArn"
left ec2 describe-nat-gateways --filter "Name=tag:Name,Values=${CLUSTER_NAME}-*" "Name=state,Values=pending,available" --query 'NatGateways[].NatGatewayId'
left ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[].VpcId'
left ecr describe-repositories --query "repositories[?starts_with(repositoryName, 'ridenow/')].repositoryName"
left iam list-roles --query "Roles[?starts_with(RoleName, '${CLUSTER_NAME}-')].RoleName"
left cloudfront list-distributions --query "DistributionList.Items[?Comment=='RideNow ${ENV} frontend'].Id"
left s3api list-buckets --query "Buckets[?starts_with(Name, '${CLUSTER_NAME}-frontend-')].Name"
echo "Done."
```

Expected result:

```text
Still present:
Done.
```

An empty list means nothing is left to pay for. KMS keys scheduled for deletion are not listed, and they cost nothing after you schedule the deletion.