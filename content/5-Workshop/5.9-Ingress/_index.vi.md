---
title : "Ingress"
date : 2024-01-01
weight : 9
chapter : false
pre : " <b> 5.9. </b> "
---

Trang này mở cơ sở dữ liệu và cache cho các Fargate pod, đăng ký OIDC provider của cluster trong IAM, và tạo sáu IRSA role (IAM role cho service account). Sau đó trang triển khai bảy workload của môi trường `prod`. Role cho Load Balancer Controller và External Secrets phải tồn tại trước khi bạn cài hai add-on đó ở mục 5.8.

## Set variables

Chạy khối lệnh này trong một shell mới. Khối này lặp lại các biến dùng chung từ mục 5.2, và tra cứu các giá trị mạng, security group và cluster từ mục 5.4, 5.7 và 5.8:

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

Kết quả mong đợi (các ID đã được che):

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

Nếu một dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh. Các biến ARN của role tra cứu những role mà bạn tạo ngay trên trang này. Hãy chạy khối lệnh ở phần Values to record sau khi hoàn thành Create the resources, không chạy trước.

## What it creates

| Template logical ID | AWS resource type | Resolved name |
|---|---|---|
| `DbIngressFromFargatePods` | EC2 security group inbound rule | trên `ridenow-prod-db-sg`, TCP 5432 |
| `RedisIngressFromFargatePods` | EC2 security group inbound rule | trên `ridenow-prod-redis-sg`, TCP 6379 |
| `OidcProvider` | IAM OIDC identity provider | issuer URL của cluster `ridenow-prod` |
| `AlbControllerRole` | IAM role | `ridenow-prod-alb-controller` |
| `ExternalSecretsRole` | IAM role | `ridenow-prod-external-secrets` |
| `TripServiceRole` | IAM role | `ridenow-prod-trip-service` |
| `PaymentServiceRole` | IAM role | `ridenow-prod-payment-service` |
| `MatchingServiceRole` | IAM role | `ridenow-prod-matching-service` |
| `NotificationWorkerRole` | IAM role | `ridenow-prod-notification-worker` |

Các Fargate pod chạy trong security group mà EKS tạo cho cluster. Security group của database và cache được tạo trước khi group đó tồn tại, nên hai inbound rule này lấp khoảng trống đó. Thiếu chúng, mọi pod đều bị timeout khi kết nối tới PostgreSQL hoặc Redis.

Mỗi role tin cậy OIDC provider của cluster cho đúng một Kubernetes service account. `user-service`, `driver-service` và `realtime-gateway` không có role.

## Depends on

5.1 đến 5.8. Bạn cần các tài nguyên sau trước khi bắt đầu:

| Resource | Name | Created in | Used for |
|---|---|---|---|
| Security group | `ridenow-prod-db-sg` | 5.4 | Đích của rule cổng 5432 |
| Security group | `ridenow-prod-redis-sg` | 5.7 | Đích của rule cổng 6379 |
| EKS cluster | `ridenow-prod` và cluster security group `eks-cluster-sg-ridenow-prod-<number>` | 5.8 | Nguồn của cả hai rule, và OIDC issuer |
| Managed policy | `ridenow-prod-trip-service-messaging` | 5.3 | Quyền của `ridenow-prod-trip-service` |
| Managed policy | `ridenow-prod-payment-service-messaging` | 5.3 | Quyền của `ridenow-prod-payment-service` |
| Managed policy | `ridenow-prod-matching-service-messaging` | 5.3 | Quyền của `ridenow-prod-matching-service` |
| Managed policy | `ridenow-prod-notification-worker-messaging` | 5.3 | Quyền của `ridenow-prod-notification-worker` |

## Create the resources

Mở AWS console ở region Asia Pacific (Singapore) `ap-southeast-1`. Tạo các tài nguyên theo thứ tự này. Các role cần OIDC provider.

### 1. Inbound rule: PostgreSQL từ các Fargate pod (`DbIngressFromFargatePods`)

1. Vào **EC2 -> Security Groups**.
2. Tìm `ridenow-prod-db-sg` và chọn nó.
3. Mở tab **Inbound rules** và chọn **Edit inbound rules -> Add rule**.
4. Đặt **Type** là `PostgreSQL` (mục này tự điền **Protocol** `TCP` và **Port range** `5432`).
5. Đặt **Source** là `Custom`.
6. Trong ô tìm kiếm, dán giá trị của `EKS_CLUSTER_SG_ID` và chọn group `eks-cluster-sg-ridenow-prod-<number>`.
7. Đặt **Description** là `PostgreSQL 5432 from the EKS cluster security group (Fargate pods)`.
8. Chọn **Save rules**.

Group lúc này có hai rule ở cổng 5432: rule từ mục 5.4 và rule này.

### 2. Inbound rule: Redis từ các Fargate pod (`RedisIngressFromFargatePods`)

1. Vào **EC2 -> Security Groups**.
2. Tìm `ridenow-prod-redis-sg` và chọn nó.
3. Mở tab **Inbound rules** và chọn **Edit inbound rules -> Add rule**.
4. Đặt **Type** là `Custom TCP`.
5. Đặt **Port range** là `6379`.
6. Đặt **Source** là `Custom`.
7. Dán giá trị của `EKS_CLUSTER_SG_ID` và chọn `eks-cluster-sg-ridenow-prod-<number>`.
8. Đặt **Description** là `Redis 6379 from the EKS cluster security group (Fargate pods)`.
9. Chọn **Save rules**.

Group lúc này có hai rule ở cổng 6379.

### 3. OIDC identity provider (`OidcProvider`)

In ra issuer URL của cluster:

```bash
echo "$OIDC_ISSUER_URL"
```

Kết quả mong đợi: một URL kết thúc bằng `/id/` và 32 ký tự hexa. Nếu kết quả rỗng, cluster chưa tồn tại (mục 5.8).

1. Vào **IAM -> Identity providers -> Add provider**.
2. Đặt **Provider type** là `OpenID Connect`.
3. Dán URL vừa in ra vào **Provider URL**.
4. Đặt **Audience** là `sts.amazonaws.com`.
5. Chọn **Get thumbprint**. Console tự điền `9e99a48a9960b14926bb7f3b02e22da2b0ab7280`.
6. Chọn **Add provider**.

### 4. Sáu IRSA role

Mỗi role có một trust policy nêu OIDC provider và một service account. Hãy in ra cả sáu trust policy và inline policy ngay bây giờ. Chúng chứa account ID, Region và OIDC ID của bạn, nên bạn không phải gõ tay:

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

Tạo sáu role lần lượt, với các giá trị sau:

| Role name | Service account (`namespace:name`) | Permissions |
|---|---|---|
| `ridenow-prod-alb-controller` | `kube-system:aws-load-balancer-controller` | không có khi tạo |
| `ridenow-prod-external-secrets` | `external-secrets:external-secrets` | inline policy `read-ridenow-secrets` |
| `ridenow-prod-trip-service` | `ridenow:trip-service` | managed policy `ridenow-prod-trip-service-messaging` |
| `ridenow-prod-payment-service` | `ridenow:payment-service` | managed policy `ridenow-prod-payment-service-messaging` |
| `ridenow-prod-matching-service` | `ridenow:matching-service` | managed policy `ridenow-prod-matching-service-messaging` |
| `ridenow-prod-notification-worker` | `ridenow:notification-worker` | managed policy `ridenow-prod-notification-worker-messaging` |

#### 4.1 Tạo role

Lặp lại danh sách này cho từng dòng của bảng.

1. Vào **IAM -> Roles -> Create role**.
2. Đặt **Trusted entity type** là `Web identity`.
3. Đặt **Identity provider** là OIDC provider ở bước 3 (mục kết thúc bằng `OIDC_ID` của bạn).
4. Đặt **Audience** là `sts.amazonaws.com`.
5. Chọn **Next**.
6. Ở **Add permissions**, với hai role không có managed policy (`ridenow-prod-alb-controller` và `ridenow-prod-external-secrets`) thì không tick policy nào. Với bốn role còn lại, đặt bộ lọc là `Customer managed`, tìm policy trong bảng và tick nó.
7. Chọn **Next**.
8. Đặt **Role name** là tên trong bảng.
9. Để trống **Description**.
10. Chọn **Create role**.

#### 4.2 Thay trust policy

Lặp lại danh sách này cho từng role trong sáu role.

1. Trong danh sách role, tìm tên role và mở nó.
2. Mở tab **Trust relationships** và chọn **Edit trust policy**.
3. Xóa toàn bộ nội dung.
4. Dán trust policy mà khối lệnh ở trên đã in cho role này.
5. Chọn **Update policy**.

#### 4.3 Thêm inline policy cho `ridenow-prod-external-secrets`

1. Mở role `ridenow-prod-external-secrets` và tab **Permissions** của nó.
2. Chọn **Add permissions -> Create inline policy**.
3. Chọn tab **JSON** và xóa toàn bộ nội dung.
4. Dán inline policy mà khối lệnh ở trên đã in.
5. Chọn **Next**.
6. Đặt **Policy name** là `read-ridenow-secrets`.
7. Chọn **Create policy**.

## Values to record

Chạy khối lệnh này ngay bây giờ. Khối này đọc ARN của từng role theo tên, nên bạn không phải sao chép gì bằng tay:

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

Kết quả mong đợi (account ID đã được che):

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
| `ALB_CONTROLLER_ROLE_ARN` | ARN của `ridenow-prod-alb-controller` | 5.8 (cài Load Balancer Controller) |
| `EXTERNAL_SECRETS_ROLE_ARN` | ARN của `ridenow-prod-external-secrets` | 5.8 (cài External Secrets) |
| `TRIP_SERVICE_ROLE_ARN` | ARN của `ridenow-prod-trip-service` | `22-trip-service.yaml` |
| `PAYMENT_SERVICE_ROLE_ARN` | ARN của `ridenow-prod-payment-service` | `23-payment-service.yaml` |
| `MATCHING_SERVICE_ROLE_ARN` | ARN của `ridenow-prod-matching-service` | `24-matching-service.yaml` |
| `NOTIFICATION_WORKER_ROLE_ARN` | ARN của `ridenow-prod-notification-worker` | `25-notification-worker.yaml` |

## Post-deploy steps

Chạy các bước này từ thư mục gốc của repository RideNow, trong shell mà bạn đã chạy các khối lệnh ở trên. Bạn cần có image cho `IMAGE_TAG` trong ECR (mục 5.5), kubeconfig, namespace `ridenow` và các database secret (mục 5.8), và đã cài Load Balancer Controller cùng External Secrets (mục 5.8).

### 1. Kiểm tra các IRSA role

```bash
for ROLE in alb-controller external-secrets trip-service payment-service matching-service notification-worker; do
  aws iam get-role --role-name "ridenow-${ENV}-${ROLE}" --query 'Role.AssumeRolePolicyDocument.Statement[0].Condition.StringEquals' --output json \
    | jq -r --arg r "$ROLE" 'to_entries[] | select(.key | endswith(":sub")) | "\($r)  \(.value)"'
done
```

Kết quả mong đợi: sáu dòng, mỗi dòng gồm role và `system:serviceaccount:<namespace>:<service account>` của nó theo bảng ở bước 4.

```text
alb-controller  system:serviceaccount:kube-system:aws-load-balancer-controller
external-secrets  system:serviceaccount:external-secrets:external-secrets
trip-service  system:serviceaccount:ridenow:trip-service
payment-service  system:serviceaccount:ridenow:payment-service
matching-service  system:serviceaccount:ridenow:matching-service
notification-worker  system:serviceaccount:ridenow:notification-worker
```

### 2. Build và push các image

Nếu bạn đã push image ở mục 5.5 với cùng `IMAGE_TAG`, hãy bỏ qua bước này. Nếu chưa, chạy khối lệnh này khi Docker đang chạy:

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

Kết quả mong đợi: `Login Succeeded`, rồi bảy lần push thành công.

### 3. Đặt các biến cho workload

Bảy manifest chứa các giá trị mà bạn điền bằng biến. Hãy tra cứu các giá trị database, queue và cache mà các trang trước đã tạo:

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

Các service chỉ nhận request từ một browser origin, và giá trị này không bao giờ là `*`. Đây là một Manual_Value. Bạn lấy nó từ trang frontend, vì domain CloudFront chỉ có sau khi bạn tạo nó. Hãy sao chép origin từ mục 5.10 (domain CloudFront, viết dưới dạng `https://` theo sau là domain) vào `<APP_ORIGIN>` trong khối lệnh tiếp theo. Nếu bạn chưa làm mục 5.10, hãy dùng `https://app.example.com` và áp dụng lại các manifest sau mục 5.10.

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

Kết quả mong đợi: không có dòng `STOP`.

### 4. Triển khai bảy workload

Triển khai theo thứ tự này: payment trước trip, và driver trước matching. Fargate khởi động một node cho mỗi pod, nên một lần rollout có thể mất vài phút. Các file là `23-payment-service.yaml`, `20-user-service.yaml`, `21-driver-service.yaml`, `22-trip-service.yaml`, `24-matching-service.yaml`, `25-notification-worker.yaml` và `26-realtime-gateway.yaml`, đều nằm trong `iac/k8s/`.

```bash
for F in 23-payment-service 20-user-service 21-driver-service 22-trip-service \
         24-matching-service 25-notification-worker 26-realtime-gateway; do
  sed "${SUBST[@]}" iac/k8s/$F.yaml | kubectl apply -f -
  kubectl rollout status deployment/${F#*-} -n ridenow --timeout=900s
done
```

Kết quả mong đợi: bảy dòng `deployment "<service>" successfully rolled out`. Nếu một rollout lỗi, hãy chạy `kubectl describe pod -n ridenow -l app=<service>` và `kubectl logs -n ridenow deployment/<service> --tail=50`. Timeout tới PostgreSQL hoặc Redis nghĩa là thiếu các inbound rule ở bước 1 và 2, hoặc thiếu phần pod networking ở mục 5.8.

## Verify

Chạy các lệnh sau trong cùng shell:

```bash
aws iam list-open-id-connect-providers --query "OpenIDConnectProviderList[?contains(Arn, '$OIDC_ID')].Arn" --output text
aws ec2 describe-security-groups --group-ids "$DB_SG_ID" --query "SecurityGroups[0].IpPermissions[?FromPort==\`5432\`].UserIdGroupPairs[].GroupId" --output text
aws ec2 describe-security-groups --group-ids "$REDIS_SG_ID" --query "SecurityGroups[0].IpPermissions[?FromPort==\`6379\`].UserIdGroupPairs[].GroupId" --output text
echo "cluster SG: $EKS_CLUSTER_SG_ID"
kubectl get pods -n ridenow
```

Kết quả mong đợi: một ARN provider kết thúc bằng `/id/` và `OIDC_ID` của bạn. Cả hai dòng security group đều chứa giá trị `cluster SG:`. Phần kiểm tra role ở Post-deploy steps in ra sáu dòng. Danh sách pod hiện 13 pod `1/1 Running`: hai pod cho mỗi service `user-service`, `driver-service`, `trip-service`, `payment-service`, `matching-service` và `realtime-gateway`, và một pod cho `notification-worker`.

Trang tiếp theo là 5.10 Frontend. Hãy làm sau khi Ingress `ridenow` đã có địa chỉ ALB.
