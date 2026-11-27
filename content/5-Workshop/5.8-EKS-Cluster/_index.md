---
title : "EKS Cluster"
date : 2024-01-01
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
---

This page creates the Kubernetes control plane of the `prod` environment: a KMS key for secrets encryption, the EKS cluster `ridenow-prod` and three Fargate profiles. After that you connect to the cluster, install the two controllers, and prepare the namespace, pod networking and database secrets.

## Set variables

Run this block in a new shell. It repeats the shared variables from 5.2, and it looks up the resources from 5.3, 5.4 and 5.6:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

CLUSTER_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-eks-cluster-role" \
  --query Role.Arn --output text)
FARGATE_POD_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-eks-fargate-pod-execution-role" \
  --query Role.Arn --output text)

[ -n "$CLUSTER_ROLE_ARN" ] || echo "STOP: CLUSTER_ROLE_ARN is empty. Create the role ridenow-prod-eks-cluster-role first."
[ -n "$FARGATE_POD_ROLE_ARN" ] || echo "STOP: FARGATE_POD_ROLE_ARN is empty. Create the role ridenow-prod-eks-fargate-pod-execution-role first."

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)
[ -n "$VPC_ID" ] && [ "$VPC_ID" != "None" ] || echo "STOP: VPC_ID is empty. Create the network first (5.4)."

PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-private-*" --query 'Subnets[].SubnetId' --output text)
[ -n "$PRIVATE_SUBNET_IDS" ] || echo "STOP: PRIVATE_SUBNET_IDS is empty. Create the private subnets first (5.4)."

CLUSTER_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-cluster-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$CLUSTER_SG_ID" ] && [ "$CLUSTER_SG_ID" != "None" ] || echo "STOP: CLUSTER_SG_ID is empty. Create the cluster security group first (5.4)."

RDS_KMS_KEY_ARN=$(aws kms describe-key --key-id "alias/${CLUSTER_NAME}-rds" --query KeyMetadata.Arn --output text 2>/dev/null)
[ -n "$RDS_KMS_KEY_ARN" ] && [ "$RDS_KMS_KEY_ARN" != "None" ] || echo "STOP: RDS_KMS_KEY_ARN is empty. Create the KMS key and alias first (5.6)."

USER_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-user-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$USER_DB_SECRET_ARN" ] && [ "$USER_DB_SECRET_ARN" != "None" ] || echo "STOP: USER_DB_SECRET_ARN is empty. Create ridenow-prod-user-db with Managed in AWS Secrets Manager first (5.6)."

DRIVER_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-driver-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$DRIVER_DB_SECRET_ARN" ] && [ "$DRIVER_DB_SECRET_ARN" != "None" ] || echo "STOP: DRIVER_DB_SECRET_ARN is empty. Create ridenow-prod-driver-db with Managed in AWS Secrets Manager first (5.6)."

TRIP_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-trip-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$TRIP_DB_SECRET_ARN" ] && [ "$TRIP_DB_SECRET_ARN" != "None" ] || echo "STOP: TRIP_DB_SECRET_ARN is empty. Create ridenow-prod-trip-db with Managed in AWS Secrets Manager first (5.6)."

PAYMENT_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-payment-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$PAYMENT_DB_SECRET_ARN" ] && [ "$PAYMENT_DB_SECRET_ARN" != "None" ] || echo "STOP: PAYMENT_DB_SECRET_ARN is empty. Create ridenow-prod-payment-db with Managed in AWS Secrets Manager first (5.6)."

MATCHING_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-matching-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$MATCHING_DB_SECRET_ARN" ] && [ "$MATCHING_DB_SECRET_ARN" != "None" ] || echo "STOP: MATCHING_DB_SECRET_ARN is empty. Create ridenow-prod-matching-db with Managed in AWS Secrets Manager first (5.6)."

NOTIFICATION_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-notification-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$NOTIFICATION_DB_SECRET_ARN" ] && [ "$NOTIFICATION_DB_SECRET_ARN" != "None" ] || echo "STOP: NOTIFICATION_DB_SECRET_ARN is empty. Create ridenow-prod-notification-db with Managed in AWS Secrets Manager first (5.6)."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "CLUSTER_ROLE_ARN=$CLUSTER_ROLE_ARN"
echo "FARGATE_POD_ROLE_ARN=$FARGATE_POD_ROLE_ARN"
echo "VPC_ID=$VPC_ID"
echo "PRIVATE_SUBNET_IDS=$PRIVATE_SUBNET_IDS"
echo "CLUSTER_SG_ID=$CLUSTER_SG_ID"
echo "RDS_KMS_KEY_ARN=$RDS_KMS_KEY_ARN"
echo "USER_DB_SECRET_ARN=$USER_DB_SECRET_ARN"
echo "DRIVER_DB_SECRET_ARN=$DRIVER_DB_SECRET_ARN"
echo "TRIP_DB_SECRET_ARN=$TRIP_DB_SECRET_ARN"
echo "PAYMENT_DB_SECRET_ARN=$PAYMENT_DB_SECRET_ARN"
echo "MATCHING_DB_SECRET_ARN=$MATCHING_DB_SECRET_ARN"
echo "NOTIFICATION_DB_SECRET_ARN=$NOTIFICATION_DB_SECRET_ARN"
```

Expected result (the IDs are masked and the values are shortened):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
CLUSTER_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-eks-cluster-role
FARGATE_POD_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-eks-fargate-pod-execution-role
VPC_ID=vpc-0abc...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
CLUSTER_SG_ID=sg-0eee...
RDS_KMS_KEY_ARN=arn:aws:kms:ap-southeast-1:123456789012:key/1234abcd-...
USER_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
DRIVER_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
TRIP_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
PAYMENT_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
MATCHING_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
NOTIFICATION_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
```

If a line starts with `STOP`, fix that problem and run the block again.

The cluster variables look up resources that you create on this page. Run the next block after you finish Create the resources, not before. Each lookup finds the cluster by name.

```bash
EKS_ENDPOINT=$(aws eks describe-cluster --name "$CLUSTER_NAME" --query cluster.endpoint --output text 2>/dev/null)
[ -n "$EKS_ENDPOINT" ] && [ "$EKS_ENDPOINT" != "None" ] || echo "STOP: EKS_ENDPOINT is empty. Create the cluster first and wait until it is Active."

OIDC_ISSUER_URL=$(aws eks describe-cluster --name "$CLUSTER_NAME" --query cluster.identity.oidc.issuer --output text 2>/dev/null)
[ -n "$OIDC_ISSUER_URL" ] && [ "$OIDC_ISSUER_URL" != "None" ] || echo "STOP: OIDC_ISSUER_URL is empty. Create the cluster first and wait until it is Active."

OIDC_ID="${OIDC_ISSUER_URL##*/}"
[ -n "$OIDC_ID" ] || echo "STOP: OIDC_ID is empty. Check OIDC_ISSUER_URL."

EKS_CLUSTER_SG_ID=$(aws eks describe-cluster --name "$CLUSTER_NAME" --query cluster.resourcesVpcConfig.clusterSecurityGroupId --output text 2>/dev/null)
[ -n "$EKS_CLUSTER_SG_ID" ] && [ "$EKS_CLUSTER_SG_ID" != "None" ] || echo "STOP: EKS_CLUSTER_SG_ID is empty. Create the cluster first and wait until it is Active."

echo "EKS_ENDPOINT=$EKS_ENDPOINT"
echo "OIDC_ISSUER_URL=$OIDC_ISSUER_URL"
echo "OIDC_ID=$OIDC_ID"
echo "EKS_CLUSTER_SG_ID=$EKS_CLUSTER_SG_ID"
```

Expected result (the IDs are masked):

```text
EKS_ENDPOINT=https://ABCD....gr7.ap-southeast-1.eks.amazonaws.com
OIDC_ISSUER_URL=https://oidc.eks.ap-southeast-1.amazonaws.com/id/ABCD...
OIDC_ID=ABCD...
EKS_CLUSTER_SG_ID=sg-0fff...
```

The two role ARNs of the next block belong to the roles that you create in 5.9. Run the block only after those roles exist, before the two installs in Post-deploy steps.

```bash
ALB_CONTROLLER_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-alb-controller" \
  --query Role.Arn --output text)
EXTERNAL_SECRETS_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-external-secrets" \
  --query Role.Arn --output text)

[ -n "$ALB_CONTROLLER_ROLE_ARN" ] || echo "STOP: ALB_CONTROLLER_ROLE_ARN is empty. Create the role ridenow-prod-alb-controller first (5.9)."
[ -n "$EXTERNAL_SECRETS_ROLE_ARN" ] || echo "STOP: EXTERNAL_SECRETS_ROLE_ARN is empty. Create the role ridenow-prod-external-secrets first (5.9)."

echo "ALB_CONTROLLER_ROLE_ARN=$ALB_CONTROLLER_ROLE_ARN"
echo "EXTERNAL_SECRETS_ROLE_ARN=$EXTERNAL_SECRETS_ROLE_ARN"
```

Expected result (the account ID is masked):

```text
ALB_CONTROLLER_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-alb-controller
EXTERNAL_SECRETS_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-external-secrets
```

## What it creates

| Template logical ID | AWS resource type | Resolved name |
|---|---|---|
| `ClusterKmsKey` | KMS key | alias `ridenow-prod-eks-secrets`, description `ridenow-prod EKS secrets encryption key` |
| `Cluster` | EKS cluster | `ridenow-prod` (Kubernetes `1.31`) |
| `SystemFargateProfile` | EKS Fargate profile | `ridenow-prod-system` |
| `AppFargateProfile` | EKS Fargate profile | `ridenow-prod-app` |
| `ObservabilityFargateProfile` | EKS Fargate profile | `ridenow-prod-observability` |

## Depends on

5.3 IAM and 5.4 Network. You need these resources before you start. You select each one by name in the console.

| Resource | Name | Used for |
|---|---|---|
| IAM role (5.3) | `ridenow-prod-eks-cluster-role` | Cluster service role |
| IAM role (5.3) | `ridenow-prod-eks-fargate-pod-execution-role` | Pod execution role of the three profiles |
| Private subnets (5.4) | `ridenow-prod-private-1`, `ridenow-prod-private-2` | Cluster subnets and Fargate profile subnets |
| Security group (5.4) | `ridenow-prod-cluster-sg` | Additional cluster security group |

The Post-deploy steps also need the database secrets and the KMS key `ridenow-prod-rds` from 5.6, and the IRSA roles `ridenow-prod-alb-controller` and `ridenow-prod-external-secrets` from 5.9. Those two roles must exist before the two Helm installs (steps 3 and 4).

## Create the resources

Open the AWS console in the Asia Pacific (Singapore) `ap-southeast-1` region. Follow the steps in order. The console needs an alias for the key, and EKS allows only one Fargate profile in creation at a time, so wait for **Active** between profiles.

### 1. KMS key (`ClusterKmsKey`)

Print the key policy. It contains your account ID, so you do not type it by hand:

```bash
cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountAdmin",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::${ACCOUNT_ID}:root" },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
EOF
```

1. Go to **KMS -> Customer managed keys -> Create key**.
2. Set **Key type** to `Symmetric`.
3. Set **Key usage** to `Encrypt and decrypt`.
4. Expand **Advanced options**. Set **Key material origin** to `KMS`.
5. Set **Regionality** to `Single-Region key`.
6. Choose **Next**.
7. Set **Alias** to `ridenow-prod-eks-secrets`.
8. Set **Description** to `ridenow-prod EKS secrets encryption key`.
9. Choose **Next**.
10. On **Key administrators**, tick nothing. Choose **Next**.
11. On **Key users**, tick nothing. Choose **Next**.
12. On **Key policy**, delete the existing JSON. Paste the policy that the block above printed.
13. Choose **Next**, then **Finish**.
14. Open the key. Go to the **Key rotation** tab and choose **Edit**.
15. Tick **Automatically rotate this KMS key every year**.
16. Choose **Save**.

### 2. EKS cluster (`Cluster`)

1. Go to **EKS -> Clusters -> Create cluster**.
2. Set **Configuration options** to `Custom configuration`.
3. Turn **Use EKS Auto Mode** off.
4. Set **Name** to `ridenow-prod`.
5. Set **Cluster IAM role** to `ridenow-prod-eks-cluster-role`.
6. Set **Kubernetes version** to `1.31`.
7. Set **Upgrade policy** to `Standard`.
8. Leave **Bootstrap cluster administrator access** on `Allow cluster administrator access`.
9. Leave **Cluster authentication mode** on `EKS API and ConfigMap`.
10. Tick **Turn on envelope encryption of Kubernetes secrets using KMS**.
11. Set **KMS key** to the alias `ridenow-prod-eks-secrets`.
12. Choose **Next**.
13. Set **VPC** to `ridenow-prod-vpc`.
14. In **Subnets**, untick every subnet except `ridenow-prod-private-1` and `ridenow-prod-private-2`.
15. Set **Security groups** to `ridenow-prod-cluster-sg`.
16. Set **Cluster IP address family** to `IPv4`.
17. Set **Cluster endpoint access** to `Public and private`.
18. Expand **Advanced settings**. Leave **Public access source allowlist** on `0.0.0.0/0`.
19. Choose **Next**.
20. On **Configure observability**, leave every control plane log type off. Choose **Next**.
21. On **Select add-ons**, leave `kube-proxy`, `CoreDNS` and `Amazon VPC CNI` ticked. Choose **Next**.
22. On **Configure selected add-ons settings**, leave the defaults. Choose **Next**.
23. Choose **Create**.
24. Wait 10 to 15 minutes until **Status** is `Active`.

### 3. Fargate profile `ridenow-prod-system` (`SystemFargateProfile`)

1. Go to **EKS -> Clusters -> `ridenow-prod` -> Compute** tab **-> Fargate profiles -> Add Fargate profile**.
2. Set **Name** to `ridenow-prod-system`.
3. Set **Pod execution role** to `ridenow-prod-eks-fargate-pod-execution-role`.
4. In **Subnets**, keep only `ridenow-prod-private-1` and `ridenow-prod-private-2`.
5. Choose **Next**.
6. Under **Pod selectors**, set **Namespace** to `kube-system`. Leave **Match labels** empty.
7. Choose **Add selector**. Set **Namespace** to `external-secrets`. Leave **Match labels** empty.
8. Choose **Next**.
9. Choose **Create**.
10. Wait 1 to 2 minutes until **Status** is `Active`.

### 4. Fargate profile `ridenow-prod-app` (`AppFargateProfile`)

Start this profile only after `ridenow-prod-system` is **Active**.

1. Go to **EKS -> Clusters -> `ridenow-prod` -> Compute** tab **-> Fargate profiles -> Add Fargate profile**.
2. Set **Name** to `ridenow-prod-app`.
3. Set **Pod execution role** to `ridenow-prod-eks-fargate-pod-execution-role`.
4. In **Subnets**, keep only `ridenow-prod-private-1` and `ridenow-prod-private-2`.
5. Choose **Next**.
6. Under **Pod selectors**, set **Namespace** to `ridenow`. Leave **Match labels** empty.
7. Choose **Next**.
8. Choose **Create**.
9. Wait until **Status** is `Active`.

### 5. Fargate profile `ridenow-prod-observability` (`ObservabilityFargateProfile`)

Start this profile only after `ridenow-prod-app` is **Active**.

1. Go to **EKS -> Clusters -> `ridenow-prod` -> Compute** tab **-> Fargate profiles -> Add Fargate profile**.
2. Set **Name** to `ridenow-prod-observability`.
3. Set **Pod execution role** to `ridenow-prod-eks-fargate-pod-execution-role`.
4. In **Subnets**, keep only `ridenow-prod-private-1` and `ridenow-prod-private-2`.
5. Choose **Next**.
6. Under **Pod selectors**, set **Namespace** to `aws-observability`. Leave **Match labels** empty.
7. Choose **Next**.
8. Choose **Create**.
9. Wait until **Status** is `Active`.

## Values to record

Run the second block of Set variables now. The variables below are looked up for you, so you copy nothing by hand.

| Variable | What it holds | Used by |
|---|---|---|
| `EKS_ENDPOINT` | API server endpoint of `ridenow-prod` | Connection checks in the Post-deploy steps |
| `OIDC_ISSUER_URL` | OpenID Connect provider URL of the cluster | 5.9 (OIDC provider of the IRSA roles) |
| `OIDC_ID` | The id at the end of the OIDC provider URL | 5.9 (trust policies of the IRSA roles) |
| `EKS_CLUSTER_SG_ID` | ID of the security group that EKS creates for the cluster | 5.9 (source of the database 5432 and Redis 6379 rules) |

## Post-deploy steps

Run these steps from the repository root, in order. Run the Set variables blocks first in each new terminal.

The steps use `kubectl`, Helm and the AWS CLI. They are CLI exceptions, because Kubernetes objects and the controller installs have no console page to create them. Only the IAM policy and security group changes use the console.

The IRSA roles `ridenow-prod-alb-controller` and `ridenow-prod-external-secrets` from 5.9 must exist before steps 3 and 4. Create those roles first, then run the third block of Set variables and check that both role ARNs have a value. Do step 3 before step 4.

### 1. Kubeconfig

This is a CLI exception: the kubeconfig is a local file, so only the CLI can write it. It uses the IAM identity that created the cluster.

```bash
aws eks update-kubeconfig --name "$CLUSTER_NAME"
kubectl get svc kubernetes
```

Expected result: `Updated context ...cluster/ridenow-prod`, then one `kubernetes   ClusterIP ... 443/TCP` row.

### 2. CoreDNS on Fargate

This is a CLI exception: a deployment patch and restart exist only in the Kubernetes API. Fargate has no nodes. CoreDNS pods stay `Pending` ("no nodes available") in two cases. Both are fixed by recreating them once the `ridenow-prod-system` Fargate profile is **Active**:

- The CoreDNS template still carries the annotation that pins it to EC2. Remove it.
- The pods were created before the Fargate profile existed. A pod is matched to a profile only when it is created, so these pods never move to Fargate on their own. The annotation can already be gone in this case, and the restart alone is the fix.

```bash
# Remove the EC2 pin only if it is still there (the patch fails when it is not).
if kubectl get deployment coredns -n kube-system \
     -o jsonpath='{.spec.template.metadata.annotations}' | grep -q compute-type; then
  kubectl patch deployment coredns -n kube-system --type=json \
    -p='[{"op":"remove","path":"/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
fi
# Always recreate the pods, so they are scheduled on the now-Active profile.
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status  deployment coredns -n kube-system
```

Expected result: `deployment "coredns" successfully rolled out`.

### 3. AWS Load Balancer Controller

This is a CLI exception: the controller ships as a Helm chart, and the policy file depends on the chart version. Install the controller with Helm, using the role ARN and VPC id from Set variables:

```bash
helm repo add eks https://aws.github.io/eks-charts && helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=$ALB_CONTROLLER_ROLE_ARN" \
  --set region="$AWS_REGION" \
  --set vpcId="$VPC_ID"
```

The controller has no permissions yet. Without them the Ingress gets no address and the controller logs `AccessDenied ... elasticloadbalancing:DescribeLoadBalancers`. The permissions policy depends on the controller version, so download the one that matches the version Helm installed:

```bash
mkdir -p .tmp
CONTROLLER_VERSION=$(helm list -n kube-system -o json \
  | jq -r '.[] | select(.name=="aws-load-balancer-controller") | .app_version')
echo "controller version: $CONTROLLER_VERSION"
curl -fsSL -o .tmp/alb-controller-iam-policy.json \
  "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/${CONTROLLER_VERSION}/docs/install/iam_policy.json"
```

Expected result: a line `controller version: v...` and a file `.tmp/alb-controller-iam-policy.json`.

Create and attach the policy in the IAM console:

1. Go to **IAM -> Policies -> Create policy -> JSON** tab. Paste the contents of `.tmp/alb-controller-iam-policy.json`.
2. Name the policy `AWSLoadBalancerControllerIAMPolicy`, then create it. If it already exists from an earlier cluster, reuse it.
3. Go to **IAM -> Roles -> `ridenow-prod-alb-controller` -> Add permissions -> Attach policies**.
4. Select `AWSLoadBalancerControllerIAMPolicy`, then choose **Add permissions**.

Then restart the controller so it picks the permissions up:

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
kubectl rollout status  deployment aws-load-balancer-controller -n kube-system --timeout=300s
kubectl get deploy -n kube-system aws-load-balancer-controller
```

Expected result: the last command shows `1/1` under READY.

### 4. External Secrets Operator

External Secrets must decrypt the RDS secrets with the KMS key from 5.6 (alias `ridenow-prod-rds`). Print the permission document, then add it to the role in the IAM console:

```bash
cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "$RDS_KMS_KEY_ARN"
    }
  ]
}
EOF
```

1. Go to **IAM -> Roles -> `ridenow-prod-external-secrets` -> Add permissions -> Create inline policy -> JSON** tab.
2. Paste the document that the block above printed.
3. Name the policy `ridenow-prod-external-secrets-kms`, then create it. Skip this if the inline policy already exists.

This is a CLI exception: the operator ships as a Helm chart.

```bash
helm repo add external-secrets https://charts.external-secrets.io && helm repo update

# webhook.port=9443 is required on Fargate (the default 10250 clashes with the kubelet)
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$EXTERNAL_SECRETS_ROLE_ARN" \
  --set webhook.port=9443 \
  --wait

kubectl get pods -n external-secrets
kubectl get crd | grep external-secrets.io
```

Expected result: three pods `Running` and the `external-secrets.io` CRDs listed.

### 5. Namespace

This is a CLI exception: the namespace is a Kubernetes object that comes from a manifest in the repository.

```bash
kubectl apply -f iac/k8s/00-namespace.yaml
kubectl get namespace ridenow
```

Expected result: namespace `ridenow` with status `Active`.

### 6. Pod networking (per-pod security groups)

Each service gets its own security group, allowed only to its own database. Create the six groups and their database rules in the EC2 console. Then give them to the pods with a `SecurityGroupPolicy`.

1. Go to **EC2 -> Security Groups -> Create security group**, once per service in the table below.
2. Set **Security group name** and **Description** from the table.
3. Set **VPC** to `ridenow-prod-vpc`.
4. Leave **Inbound rules** empty and keep the default outbound rule (all traffic).
5. Under **Tags**, add `Name` with a value equal to the group name. The lookup below finds each group by this tag.
6. Choose **Create security group**.

| Name | Description |
|---|---|
| `ridenow-prod-user-service-sg` | `user-service pods` |
| `ridenow-prod-driver-service-sg` | `driver-service pods` |
| `ridenow-prod-trip-service-sg` | `trip-service pods` |
| `ridenow-prod-payment-service-sg` | `payment-service pods` |
| `ridenow-prod-matching-service-sg` | `matching-service pods` |
| `ridenow-prod-notification-service-sg` | `notification-service pods` |

Then add the database rules:

1. Go to **EC2 -> Security Groups -> `ridenow-prod-db-sg` -> Inbound rules -> Edit inbound rules**.
2. Add six rules. Set **Type** to `PostgreSQL`, **Port** to `5432`, and **Source** to each of the six groups above. Skip a rule that already exists.
3. Choose **Save rules**.

This is a CLI exception: a `SecurityGroupPolicy` is a Kubernetes object. The block reads the six group ids and applies the policies. `EKS_CLUSTER_SG_ID` stays on every pod because DNS and Redis need it:

```bash
mkdir -p .tmp
: > .tmp/sgp.yaml
for SVC in user driver trip payment matching notification; do
  SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=ridenow-${ENV}-${SVC}-service-sg" --query 'SecurityGroups[0].GroupId' --output text)
  if [ -z "$SG_ID" ] || [ "$SG_ID" = "None" ]; then echo "MISSING: ridenow-${ENV}-${SVC}-service-sg (create it in the EC2 console)"; continue; fi
  APP=${SVC}-service; [ $SVC = notification ] && APP=notification-worker
  cat >> .tmp/sgp.yaml <<EOF
---
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: ${APP}-sgp
  namespace: ridenow
spec:
  podSelector:
    matchLabels:
      app: ${APP}
  securityGroups:
    groupIds:
      - ${SG_ID}
      - ${EKS_CLUSTER_SG_ID}
EOF
  echo "$APP -> $SG_ID"
done

kubectl apply -f .tmp/sgp.yaml
kubectl rollout restart deploy -n ridenow
kubectl get securitygrouppolicy -n ridenow
```

Pods started earlier keep the old security group until restarted, which is why the rollout restart is there. It does nothing on a first install. Expected result: six lines `<name> -> sg-...` and six `SecurityGroupPolicy` rows.

### 7. Fargate log router

This is a CLI exception: the log router reads a Kubernetes ConfigMap.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: aws-observability
  labels:
    aws-observability: enabled
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-logging
  namespace: aws-observability
data:
  flb_log_cw: "true"
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match kube.*
        region ${AWS_REGION}
        log_group_name /aws/eks/ridenow/fargate
        log_stream_prefix fargate-
        auto_create_group true
EOF
```

Expected result: `namespace/aws-observability created` and `configmap/aws-logging created`.

### 8. Database secrets (ExternalSecrets)

This is a CLI exception: an `ExternalSecret` is a Kubernetes object, filled from the manifest in the repository. It needs the six `*_DB_SECRET_ARN` variables from Set variables (all with a value) and the External Secrets install from step 4.

```bash
sed -e "s|[<]REGION[>]|$AWS_REGION|g" \
    -e "s|[<]ENV[>]|$ENV|g" \
    -e "s|[<]USER_SECRET_ARN[>]|$USER_DB_SECRET_ARN|g" \
    -e "s|[<]DRIVER_SECRET_ARN[>]|$DRIVER_DB_SECRET_ARN|g" \
    -e "s|[<]TRIP_SECRET_ARN[>]|$TRIP_DB_SECRET_ARN|g" \
    -e "s|[<]PAYMENT_SECRET_ARN[>]|$PAYMENT_DB_SECRET_ARN|g" \
    -e "s|[<]MATCHING_SECRET_ARN[>]|$MATCHING_DB_SECRET_ARN|g" \
    -e "s|[<]NOTIFICATION_SECRET_ARN[>]|$NOTIFICATION_DB_SECRET_ARN|g" \
    iac/k8s/10-external-secrets.yaml | kubectl apply -f -

kubectl get externalsecret -n ridenow
```

Expected result: six rows with `SecretSynced` and READY `True`. If a row shows `SecretSyncedError`, read the reason:

```bash
kubectl logs -n external-secrets deploy/external-secrets --tail=50 | grep -i "error\|denied"
```

### 9. ALB ingress and smoke test

Deploy the seven services from 5.9 first. The HTTP 200 check below succeeds only after the services run. Payment must already be running, because trip-service calls it when a trip is created. This is a CLI exception: the Ingress is a Kubernetes object, and the controller from step 3 creates the load balancer from it.

```bash
kubectl apply -f iac/k8s/30-ingress.yaml
kubectl get ingress ridenow -n ridenow
```

Run the second command again until the ADDRESS column shows a host name. Then run the smoke test:

```bash
ALB_DNS_NAME=$(kubectl get ingress ridenow -n ridenow -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB_DNS_NAME=$ALB_DNS_NAME"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://${ALB_DNS_NAME}/api/drivers"
kubectl logs -n ridenow deploy/matching-service --tail=20
kubectl logs -n ridenow deploy/notification-worker --tail=20
```

Expected result: a host name, `HTTP 200`, and matching and notification logs that show their SQS and Valkey consumers connected. Keep `ALB_DNS_NAME`, because 5.10 uses it as the CloudFront API origin.

A `503` means the load balancer has no healthy target. Run `kubectl get pods -n ridenow` and wait until every pod is `Ready`. A pod stuck at `0/1` usually means a wrong database endpoint, secret, security group or Redis endpoint. Run `kubectl describe pod` on it to see why.

## Verify

Run these commands in the same shell, after the second block of Set variables:

```bash
for V in EKS_ENDPOINT OIDC_ISSUER_URL OIDC_ID EKS_CLUSTER_SG_ID; do
  echo "$V=${!V:-MISSING}"
done
aws eks describe-cluster --name "$CLUSTER_NAME" \
  --query 'cluster.[status,version,resourcesVpcConfig.endpointPublicAccess,resourcesVpcConfig.endpointPrivateAccess,encryptionConfig[0].resources[0]]' --output text
aws eks list-fargate-profiles --cluster-name "$CLUSTER_NAME" --output text
```

Expected result: four variables with a value (none `MISSING`), the cluster `ACTIVE` on `1.31` with public and private access and `secrets` encrypted, and the three profiles `ridenow-prod-system`, `ridenow-prod-app` and `ridenow-prod-observability`.

```text
EKS_ENDPOINT=https://ABCD....gr7.ap-southeast-1.eks.amazonaws.com
OIDC_ISSUER_URL=https://oidc.eks.ap-southeast-1.amazonaws.com/id/ABCD...
OIDC_ID=ABCD...
EKS_CLUSTER_SG_ID=sg-0fff...
ACTIVE  1.31  True  True  secrets
FARGATEPROFILES  ...  ridenow-prod-system
FARGATEPROFILES  ...  ridenow-prod-app
FARGATEPROFILES  ...  ridenow-prod-observability
```

If an item is missing, open the console page for that resource and check the name.
