---
title : "EKS Addons"
date : 2024-01-01
weight : 9
chapter : false
pre : " <b> 5.9. </b> "
---

This page opens the database and cache to the Fargate pods, registers the cluster OIDC provider in IAM, and creates six IRSA roles (IAM roles for service accounts). Then it deploys the seven workloads of the `prod` environment. The roles for the Load Balancer Controller and External Secrets must exist before you install those two add-ons in 5.8.

## Set variables

Run this block in a new shell. It repeats the shared variables from 5.2, and it looks up the network, security group, and cluster values from 5.4, 5.7, and 5.8:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod
IMAGE_TAG=v1.0.0

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)
[ -n "$VPC_ID" ] && [ "$VPC_ID" != "None" ] || echo "STOP: VPC_ID is empty. Create the network first (5.4)."

DB_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-db-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$DB_SG_ID" ] && [ "$DB_SG_ID" != "None" ] || echo "STOP: DB_SG_ID is empty. Create the database security group first (5.4)."

REDIS_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-redis-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$REDIS_SG_ID" ] && [ "$REDIS_SG_ID" != "None" ] || echo "STOP: REDIS_SG_ID is empty. Create the cache security group first (5.7)."

OIDC_ISSUER_URL=$(aws eks describe-cluster --name "$CLUSTER_NAME" --query cluster.identity.oidc.issuer --output text 2>/dev/null)
[ -n "$OIDC_ISSUER_URL" ] && [ "$OIDC_ISSUER_URL" != "None" ] || echo "STOP: OIDC_ISSUER_URL is empty. Create the EKS cluster first (5.8)."

OIDC_ID="${OIDC_ISSUER_URL##*/}"
[ -n "$OIDC_ID" ] || echo "STOP: OIDC_ID is empty. Create the EKS cluster first (5.8)."

EKS_CLUSTER_SG_ID=$(aws eks describe-cluster --name "$CLUSTER_NAME" --query cluster.resourcesVpcConfig.clusterSecurityGroupId --output text 2>/dev/null)
[ -n "$EKS_CLUSTER_SG_ID" ] && [ "$EKS_CLUSTER_SG_ID" != "None" ] || echo "STOP: EKS_CLUSTER_SG_ID is empty. Create the EKS cluster first (5.8)."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "IMAGE_TAG=$IMAGE_TAG"
echo "VPC_ID=$VPC_ID"
echo "DB_SG_ID=$DB_SG_ID"
echo "REDIS_SG_ID=$REDIS_SG_ID"
echo "OIDC_ISSUER_URL=$OIDC_ISSUER_URL"
echo "OIDC_ID=$OIDC_ID"
echo "EKS_CLUSTER_SG_ID=$EKS_CLUSTER_SG_ID"
```

Expected result (the IDs are masked):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
IMAGE_TAG=v1.0.0
VPC_ID=vpc-0abc...
DB_SG_ID=sg-0fff...
REDIS_SG_ID=sg-0aaa...
OIDC_ISSUER_URL=https://oidc.eks.ap-southeast-1.amazonaws.com/id/ABCD...
OIDC_ID=ABCD...
EKS_CLUSTER_SG_ID=sg-0bbb...
```

If a line starts with `STOP`, fix that problem and run the block again. The role ARN variables look up roles that you create on this page. Run the block in Values to record after you finish Create the resources, not before.

## What it creates

| Template logical ID | AWS resource type | Resolved name |
|---|---|---|
| `DbIngressFromFargatePods` | EC2 security group inbound rule | on `ridenow-prod-db-sg`, TCP 5432 |
| `RedisIngressFromFargatePods` | EC2 security group inbound rule | on `ridenow-prod-redis-sg`, TCP 6379 |
| `OidcProvider` | IAM OIDC identity provider | the issuer URL of the cluster `ridenow-prod` |
| `AlbControllerRole` | IAM role | `ridenow-prod-alb-controller` |
| `ExternalSecretsRole` | IAM role | `ridenow-prod-external-secrets` |
| `TripServiceRole` | IAM role | `ridenow-prod-trip-service` |
| `PaymentServiceRole` | IAM role | `ridenow-prod-payment-service` |
| `MatchingServiceRole` | IAM role | `ridenow-prod-matching-service` |
| `NotificationWorkerRole` | IAM role | `ridenow-prod-notification-worker` |

The Fargate pods run in the security group that EKS creates for the cluster. The database and cache security groups were created before that group existed, so the two inbound rules close the gap. Without them every pod times out when it connects to PostgreSQL or Redis.

Each role trusts the cluster OIDC provider for one Kubernetes service account. `user-service`, `driver-service`, and `realtime-gateway` have no role.

## Depends on

5.1 to 5.8. You need these resources before you start:

| Resource | Name | Created in | Used for |
|---|---|---|---|
| Security group | `ridenow-prod-db-sg` | 5.4 | Target of the port 5432 rule |
| Security group | `ridenow-prod-redis-sg` | 5.7 | Target of the port 6379 rule |
| EKS cluster | `ridenow-prod` and its cluster security group `eks-cluster-sg-ridenow-prod-<number>` | 5.8 | Source of both rules, and the OIDC issuer |
| Managed policy | `ridenow-prod-trip-service-messaging` | 5.3 | Permissions of `ridenow-prod-trip-service` |
| Managed policy | `ridenow-prod-payment-service-messaging` | 5.3 | Permissions of `ridenow-prod-payment-service` |
| Managed policy | `ridenow-prod-matching-service-messaging` | 5.3 | Permissions of `ridenow-prod-matching-service` |
| Managed policy | `ridenow-prod-notification-worker-messaging` | 5.3 | Permissions of `ridenow-prod-notification-worker` |

## Create the resources

Open the AWS console in the Asia Pacific (Singapore) `ap-southeast-1` region. Create the resources in this order. The roles need the OIDC provider.

### 1. Inbound rule: PostgreSQL from the Fargate pods (`DbIngressFromFargatePods`)

1. Go to **EC2 -> Security Groups**.
2. Search for `ridenow-prod-db-sg` and select it.
3. Open the **Inbound rules** tab and choose **Edit inbound rules -> Add rule**.
4. Set **Type** to `PostgreSQL` (this fills **Protocol** `TCP` and **Port range** `5432`).
5. Set **Source** to `Custom`.
6. In the search box, paste the value of `EKS_CLUSTER_SG_ID` and pick the group `eks-cluster-sg-ridenow-prod-<number>`.
7. Set **Description** to `PostgreSQL 5432 from the EKS cluster security group (Fargate pods)`.
8. Choose **Save rules**.

The group now has two rules on port 5432: the one from 5.4 and this one.

### 2. Inbound rule: Redis from the Fargate pods (`RedisIngressFromFargatePods`)

1. Go to **EC2 -> Security Groups**.
2. Search for `ridenow-prod-redis-sg` and select it.
3. Open the **Inbound rules** tab and choose **Edit inbound rules -> Add rule**.
4. Set **Type** to `Custom TCP`.
5. Set **Port range** to `6379`.
6. Set **Source** to `Custom`.
7. Paste the value of `EKS_CLUSTER_SG_ID` and pick `eks-cluster-sg-ridenow-prod-<number>`.
8. Set **Description** to `Redis 6379 from the EKS cluster security group (Fargate pods)`.
9. Choose **Save rules**.

The group now has two rules on port 6379.

### 3. OIDC identity provider (`OidcProvider`)

Print the issuer URL of the cluster:

```bash
echo "$OIDC_ISSUER_URL"
```

Expected result: a URL that ends with `/id/` and 32 hexadecimal characters. If it is empty, the cluster does not exist yet (5.8).

1. Go to **IAM -> Identity providers -> Add provider**.
2. Set **Provider type** to `OpenID Connect`.
3. Paste the URL printed above into **Provider URL**.
4. Set **Audience** to `sts.amazonaws.com`.
5. Choose **Get thumbprint**. The console fills `9e99a48a9960b14926bb7f3b02e22da2b0ab7280`.
6. Choose **Add provider**.

### 4. Six IRSA roles

Each role has a trust policy that names the OIDC provider and one service account. Print all six trust policies and the inline policy now. They contain your account ID, Region, and OIDC ID, so you do not type them by hand:

```bash
set +H
OIDC_HOST="oidc.eks.${AWS_REGION}.amazonaws.com/id/${OIDC_ID}"
while read -r ROLE NS SA; do
  echo "Trust policy for ridenow-${ENV}-${ROLE}:"
  cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_HOST}" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_HOST}:sub": "system:serviceaccount:${NS}:${SA}",
          "${OIDC_HOST}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF
done <<'LIST'
alb-controller kube-system aws-load-balancer-controller
external-secrets external-secrets external-secrets
trip-service ridenow trip-service
payment-service ridenow payment-service
matching-service ridenow matching-service
notification-worker ridenow notification-worker
LIST
echo "Inline policy read-ridenow-secrets for ridenow-${ENV}-external-secrets:"
cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:*:${ACCOUNT_ID}:secret:ridenow-${ENV}-*",
        "arn:aws:secretsmanager:*:${ACCOUNT_ID}:secret:rds!db-*"
      ]
    }
  ]
}
EOF
```

Create the six roles one at a time, with these values:

| Role name | Service account (`namespace:name`) | Permissions |
|---|---|---|
| `ridenow-prod-alb-controller` | `kube-system:aws-load-balancer-controller` | none at creation |
| `ridenow-prod-external-secrets` | `external-secrets:external-secrets` | inline policy `read-ridenow-secrets` |
| `ridenow-prod-trip-service` | `ridenow:trip-service` | managed policy `ridenow-prod-trip-service-messaging` |
| `ridenow-prod-payment-service` | `ridenow:payment-service` | managed policy `ridenow-prod-payment-service-messaging` |
| `ridenow-prod-matching-service` | `ridenow:matching-service` | managed policy `ridenow-prod-matching-service-messaging` |
| `ridenow-prod-notification-worker` | `ridenow:notification-worker` | managed policy `ridenow-prod-notification-worker-messaging` |

#### 4.1 Create the role

Repeat this list for each row of the table.

1. Go to **IAM -> Roles -> Create role**.
2. Set **Trusted entity type** to `Web identity`.
3. Set **Identity provider** to the OIDC provider from step 3 (the entry that ends with your `OIDC_ID`).
4. Set **Audience** to `sts.amazonaws.com`.
5. Choose **Next**.
6. On **Add permissions**, for the two roles without a managed policy (`ridenow-prod-alb-controller` and `ridenow-prod-external-secrets`) tick no policy. For the other four, set the filter to `Customer managed`, search for the policy in the table, and tick it.
7. Choose **Next**.
8. Set **Role name** to the name from the table.
9. Leave **Description** blank.
10. Choose **Create role**.

#### 4.2 Replace the trust policy

Repeat this list for each of the six roles.

1. In the roles list, search for the role name and open it.
2. Open the **Trust relationships** tab and choose **Edit trust policy**.
3. Delete the whole document.
4. Paste the trust policy that the block above printed for this role.
5. Choose **Update policy**.

#### 4.3 Add the inline policy to `ridenow-prod-external-secrets`

1. Open the role `ridenow-prod-external-secrets` and its **Permissions** tab.
2. Choose **Add permissions -> Create inline policy**.
3. Choose the **JSON** tab and delete the whole document.
4. Paste the inline policy that the block above printed.
5. Choose **Next**.
6. Set **Policy name** to `read-ridenow-secrets`.
7. Choose **Create policy**.

## Values to record

Run this block now. It reads the ARN of each role by its name, so you copy nothing by hand:

```bash
ALB_CONTROLLER_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-alb-controller" \
  --query Role.Arn --output text 2>/dev/null)
EXTERNAL_SECRETS_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-external-secrets" \
  --query Role.Arn --output text 2>/dev/null)
TRIP_SERVICE_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-trip-service" \
  --query Role.Arn --output text 2>/dev/null)
PAYMENT_SERVICE_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-payment-service" \
  --query Role.Arn --output text 2>/dev/null)
MATCHING_SERVICE_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-matching-service" \
  --query Role.Arn --output text 2>/dev/null)
NOTIFICATION_WORKER_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-notification-worker" \
  --query Role.Arn --output text 2>/dev/null)

for V in ALB_CONTROLLER_ROLE_ARN EXTERNAL_SECRETS_ROLE_ARN TRIP_SERVICE_ROLE_ARN PAYMENT_SERVICE_ROLE_ARN \
         MATCHING_SERVICE_ROLE_ARN NOTIFICATION_WORKER_ROLE_ARN; do
  [ -n "${!V}" ] && [ "${!V}" != "None" ] || echo "STOP: $V is empty. Create that role first."
  echo "$V=${!V}"
done
```

Expected result (the account ID is masked):

```text
ALB_CONTROLLER_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-alb-controller
EXTERNAL_SECRETS_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-external-secrets
TRIP_SERVICE_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-trip-service
PAYMENT_SERVICE_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-payment-service
MATCHING_SERVICE_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-matching-service
NOTIFICATION_WORKER_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-notification-worker
```

| Variable | What it holds | Used by |
|---|---|---|
| `ALB_CONTROLLER_ROLE_ARN` | ARN of `ridenow-prod-alb-controller` | 5.8 (Load Balancer Controller install) |
| `EXTERNAL_SECRETS_ROLE_ARN` | ARN of `ridenow-prod-external-secrets` | 5.8 (External Secrets install) |
| `TRIP_SERVICE_ROLE_ARN` | ARN of `ridenow-prod-trip-service` | `22-trip-service.yaml` |
| `PAYMENT_SERVICE_ROLE_ARN` | ARN of `ridenow-prod-payment-service` | `23-payment-service.yaml` |
| `MATCHING_SERVICE_ROLE_ARN` | ARN of `ridenow-prod-matching-service` | `24-matching-service.yaml` |
| `NOTIFICATION_WORKER_ROLE_ARN` | ARN of `ridenow-prod-notification-worker` | `25-notification-worker.yaml` |

## Post-deploy steps

Run these steps from the root of the RideNow repository, in the shell where you ran the blocks above. You need images for `IMAGE_TAG` in ECR (5.5), a kubeconfig, the `ridenow` namespace, and the database secrets (5.8), and the Load Balancer Controller and External Secrets installed (5.8).

### 1. Check the IRSA roles

```bash
for ROLE in alb-controller external-secrets trip-service payment-service matching-service notification-worker; do
  aws iam get-role --role-name "ridenow-${ENV}-${ROLE}" --query 'Role.AssumeRolePolicyDocument.Statement[0].Condition.StringEquals' --output json \
    | jq -r --arg r "$ROLE" 'to_entries[] | select(.key | endswith(":sub")) | "\($r)  \(.value)"'
done
```

Expected result: six lines, each with the role and its `system:serviceaccount:<namespace>:<service account>` from the table in step 4.

```text
alb-controller  system:serviceaccount:kube-system:aws-load-balancer-controller
external-secrets  system:serviceaccount:external-secrets:external-secrets
trip-service  system:serviceaccount:ridenow:trip-service
payment-service  system:serviceaccount:ridenow:payment-service
matching-service  system:serviceaccount:ridenow:matching-service
notification-worker  system:serviceaccount:ridenow:notification-worker
```

### 2. Build and push the images

If you pushed the images in 5.5 for the same `IMAGE_TAG`, skip this step. Otherwise run this block with Docker running:

```bash
REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$REGISTRY"
for SVC in user-service driver-service trip-service payment-service \
           matching-service notification-worker realtime-gateway; do
  docker buildx build --platform linux/amd64 \
    -t "$REGISTRY/ridenow/$SVC:$IMAGE_TAG" \
    --push "./$SVC"
done
```

Expected result: `Login Succeeded`, then seven successful pushes.

### 3. Set the workload variables

The seven manifests contain values that you fill in with variables. Look up the database, queue, and cache values that earlier pages created:

```bash
USER_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-user-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
DRIVER_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-driver-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
TRIP_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-trip-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
PAYMENT_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-payment-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
MATCHING_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-matching-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
NOTIFICATION_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-notification-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
MATCHING_QUEUE_URL=$(aws sqs get-queue-url --queue-name "${CLUSTER_NAME}-matching" --query QueueUrl --output text 2>/dev/null)
PAYMENT_QUEUE_URL=$(aws sqs get-queue-url --queue-name "${CLUSTER_NAME}-payment" --query QueueUrl --output text 2>/dev/null)
NOTIFICATIONS_QUEUE_URL=$(aws sqs get-queue-url --queue-name "${CLUSTER_NAME}-notifications" --query QueueUrl --output text 2>/dev/null)
REDIS_ENDPOINT=$(aws elasticache describe-replication-groups --replication-group-id "${CLUSTER_NAME}-redis" --query 'ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address' --output text 2>/dev/null)
```

The services accept requests only from one browser origin, and the value is never `*`. This is a Manual_Value. You get it from the frontend page, because the CloudFront domain exists only after you create it. Copy the origin from 5.10 (the CloudFront domain, written as `https://` followed by the domain) into `<APP_ORIGIN>` in the next block. If you have not done 5.10 yet, use `https://app.example.com` and apply the manifests again after 5.10.

```bash
ORIGIN_URL="<APP_ORIGIN>"
[[ "$ORIGIN_URL" == https://* ]] || echo "STOP: ORIGIN_URL must start with https://. Replace the placeholder with your origin."

SUBST=()
for P in ACCOUNT_ID:ACCOUNT_ID REGION:AWS_REGION ENV:ENV IMAGE_TAG:IMAGE_TAG APP_ORIGIN:ORIGIN_URL \
         REDIS_ENDPOINT:REDIS_ENDPOINT USER_DB_ENDPOINT:USER_DB_ENDPOINT DRIVER_DB_ENDPOINT:DRIVER_DB_ENDPOINT \
         TRIP_DB_ENDPOINT:TRIP_DB_ENDPOINT PAYMENT_DB_ENDPOINT:PAYMENT_DB_ENDPOINT \
         MATCHING_DB_ENDPOINT:MATCHING_DB_ENDPOINT NOTIFICATION_DB_ENDPOINT:NOTIFICATION_DB_ENDPOINT \
         MATCHING_QUEUE_URL:MATCHING_QUEUE_URL PAYMENT_QUEUE_URL:PAYMENT_QUEUE_URL \
         NOTIFICATIONS_QUEUE_URL:NOTIFICATIONS_QUEUE_URL TRIP_SERVICE_ROLE_ARN:TRIP_SERVICE_ROLE_ARN \
         PAYMENT_SERVICE_ROLE_ARN:PAYMENT_SERVICE_ROLE_ARN MATCHING_SERVICE_ROLE_ARN:MATCHING_SERVICE_ROLE_ARN \
         NOTIFICATION_WORKER_ROLE_ARN:NOTIFICATION_WORKER_ROLE_ARN; do
  K=${P%%:*}
  V=${P#*:}
  [ -n "${!V}" ] && [ "${!V}" != "None" ] || echo "STOP: $V is empty. Create that resource first, then run this block again."
  SUBST+=(-e "s|<${K}>|${!V}|g")
done
```

Expected result: no `STOP` line.

### 4. Deploy the seven workloads

Deploy in this order: payment before trip, and driver before matching. Fargate starts one node for each pod, so a rollout can take several minutes. The files are `23-payment-service.yaml`, `20-user-service.yaml`, `21-driver-service.yaml`, `22-trip-service.yaml`, `24-matching-service.yaml`, `25-notification-worker.yaml`, and `26-realtime-gateway.yaml`, all in `iac/k8s/`.

```bash
for F in 23-payment-service 20-user-service 21-driver-service 22-trip-service \
         24-matching-service 25-notification-worker 26-realtime-gateway; do
  sed "${SUBST[@]}" iac/k8s/$F.yaml | kubectl apply -f -
  kubectl rollout status deployment/${F#*-} -n ridenow --timeout=900s
done
```

Expected result: seven `deployment "<service>" successfully rolled out` lines. If a rollout fails, run `kubectl describe pod -n ridenow -l app=<service>` and `kubectl logs -n ridenow deployment/<service> --tail=50`. A timeout to PostgreSQL or Redis means the inbound rules from steps 1 and 2 are missing, or the pod networking from 5.8 is missing.

## Verify

Run these commands in the same shell:

```bash
aws iam list-open-id-connect-providers --query "OpenIDConnectProviderList[?contains(Arn, '$OIDC_ID')].Arn" --output text
aws ec2 describe-security-groups --group-ids "$DB_SG_ID" --query "SecurityGroups[0].IpPermissions[?FromPort==\`5432\`].UserIdGroupPairs[].GroupId" --output text
aws ec2 describe-security-groups --group-ids "$REDIS_SG_ID" --query "SecurityGroups[0].IpPermissions[?FromPort==\`6379\`].UserIdGroupPairs[].GroupId" --output text
echo "cluster SG: $EKS_CLUSTER_SG_ID"
kubectl get pods -n ridenow
```

Expected result: one provider ARN that ends with `/id/` and your `OIDC_ID`. Both security group lines contain the `cluster SG:` value. The role checks in Post-deploy steps print six lines. The pod list shows 13 pods `1/1 Running`: two each for `user-service`, `driver-service`, `trip-service`, `payment-service`, `matching-service`, and `realtime-gateway`, and one for `notification-worker`.

The next page is 5.10 Frontend. Do it after the `ridenow` Ingress has an ALB address.
