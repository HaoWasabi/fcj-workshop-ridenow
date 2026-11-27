---
title : "Computing"
date : 2024-01-01
weight : 8
chapter : false
pre : " <b> 5.8. </b> "
---

Trang này tạo control plane Kubernetes của môi trường `prod`: một khóa KMS để mã hóa secret, EKS cluster `ridenow-prod` và ba Fargate profile. Sau đó bạn kết nối vào cluster, cài hai controller, rồi chuẩn bị namespace, mạng cho pod và secret của database.

## Set variables

Chạy khối này trong một shell mới. Khối này lặp lại các biến dùng chung từ 5.2, và tra cứu các tài nguyên từ 5.3, 5.4 và 5.6:

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

Kết quả mong đợi (các ID được che và giá trị được rút gọn):

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

Nếu một dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó và chạy lại khối.

Các biến của cluster tra cứu tài nguyên mà bạn tạo trên trang này. Chạy khối tiếp theo sau khi bạn hoàn tất Create the resources, không chạy trước. Mỗi lệnh tra cứu tìm cluster theo tên.

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

Kết quả mong đợi (các ID được che):

```text
EKS_ENDPOINT=https://ABCD....gr7.ap-southeast-1.eks.amazonaws.com
OIDC_ISSUER_URL=https://oidc.eks.ap-southeast-1.amazonaws.com/id/ABCD...
OIDC_ID=ABCD...
EKS_CLUSTER_SG_ID=sg-0fff...
```

Hai role ARN của khối tiếp theo thuộc về các role mà bạn tạo ở 5.9. Chỉ chạy khối này sau khi các role đó đã tồn tại, trước hai bước cài đặt trong Post-deploy steps.

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

Kết quả mong đợi (account ID được che):

```text
ALB_CONTROLLER_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-alb-controller
EXTERNAL_SECRETS_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-external-secrets
```

## What it creates

| Template logical ID | AWS resource type | Resolved name |
|---|---|---|
| `ClusterKmsKey` | KMS key | alias `ridenow-prod-eks-secrets`, mô tả `ridenow-prod EKS secrets encryption key` |
| `Cluster` | EKS cluster | `ridenow-prod` (Kubernetes `1.31`) |
| `SystemFargateProfile` | EKS Fargate profile | `ridenow-prod-system` |
| `AppFargateProfile` | EKS Fargate profile | `ridenow-prod-app` |
| `ObservabilityFargateProfile` | EKS Fargate profile | `ridenow-prod-observability` |

## Depends on

5.3 IAM và 5.4 Network. Bạn cần các tài nguyên sau trước khi bắt đầu. Bạn chọn từng tài nguyên theo tên trong console.

| Tài nguyên | Tên | Dùng cho |
|---|---|---|
| IAM role (5.3) | `ridenow-prod-eks-cluster-role` | Service role của cluster |
| IAM role (5.3) | `ridenow-prod-eks-fargate-pod-execution-role` | Pod execution role của ba profile |
| Private subnet (5.4) | `ridenow-prod-private-1`, `ridenow-prod-private-2` | Subnet của cluster và của Fargate profile |
| Security group (5.4) | `ridenow-prod-cluster-sg` | Security group bổ sung của cluster |

Các Post-deploy steps còn cần secret database và khóa KMS `ridenow-prod-rds` từ 5.6, cùng các IRSA role `ridenow-prod-alb-controller` và `ridenow-prod-external-secrets` từ 5.9. Hai role này phải tồn tại trước hai lần cài Helm (bước 3 và 4).

## Create the resources

Mở AWS console ở Region Asia Pacific (Singapore) `ap-southeast-1`. Làm theo thứ tự các bước. Console yêu cầu alias cho khóa, và EKS chỉ cho phép một Fargate profile đang được tạo tại một thời điểm, nên hãy chờ **Active** giữa các profile.

### 1. KMS key (`ClusterKmsKey`)

In ra key policy. Policy chứa account ID của bạn, nên bạn không gõ tay:

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

1. Vào **KMS -> Customer managed keys -> Create key**.
2. Đặt **Key type** là `Symmetric`.
3. Đặt **Key usage** là `Encrypt and decrypt`.
4. Mở rộng **Advanced options**. Đặt **Key material origin** là `KMS`.
5. Đặt **Regionality** là `Single-Region key`.
6. Chọn **Next**.
7. Đặt **Alias** là `ridenow-prod-eks-secrets`.
8. Đặt **Description** là `ridenow-prod EKS secrets encryption key`.
9. Chọn **Next**.
10. Ở **Key administrators**, không tick gì. Chọn **Next**.
11. Ở **Key users**, không tick gì. Chọn **Next**.
12. Ở **Key policy**, xóa JSON hiện có. Dán policy mà khối ở trên đã in.
13. Chọn **Next**, rồi **Finish**.
14. Mở khóa. Vào tab **Key rotation** và chọn **Edit**.
15. Tick **Automatically rotate this KMS key every year**.
16. Chọn **Save**.

### 2. EKS cluster (`Cluster`)

1. Vào **EKS -> Clusters -> Create cluster**.
2. Đặt **Configuration options** là `Custom configuration`.
3. Tắt **Use EKS Auto Mode**.
4. Đặt **Name** là `ridenow-prod`.
5. Đặt **Cluster IAM role** là `ridenow-prod-eks-cluster-role`.
6. Đặt **Kubernetes version** là `1.31`.
7. Đặt **Upgrade policy** là `Standard`.
8. Giữ **Bootstrap cluster administrator access** ở `Allow cluster administrator access`.
9. Giữ **Cluster authentication mode** ở `EKS API and ConfigMap`.
10. Tick **Turn on envelope encryption of Kubernetes secrets using KMS**.
11. Đặt **KMS key** là alias `ridenow-prod-eks-secrets`.
12. Chọn **Next**.
13. Đặt **VPC** là `ridenow-prod-vpc`.
14. Ở **Subnets**, bỏ tick mọi subnet trừ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
15. Đặt **Security groups** là `ridenow-prod-cluster-sg`.
16. Đặt **Cluster IP address family** là `IPv4`.
17. Đặt **Cluster endpoint access** là `Public and private`.
18. Mở rộng **Advanced settings**. Giữ **Public access source allowlist** ở `0.0.0.0/0`.
19. Chọn **Next**.
20. Ở **Configure observability**, để tắt mọi loại log của control plane. Chọn **Next**.
21. Ở **Select add-ons**, giữ tick `kube-proxy`, `CoreDNS` và `Amazon VPC CNI`. Chọn **Next**.
22. Ở **Configure selected add-ons settings**, giữ mặc định. Chọn **Next**.
23. Chọn **Create**.
24. Chờ 10 đến 15 phút cho đến khi **Status** là `Active`.

### 3. Fargate profile `ridenow-prod-system` (`SystemFargateProfile`)

1. Vào **EKS -> Clusters -> `ridenow-prod` -> Compute** tab **-> Fargate profiles -> Add Fargate profile**.
2. Đặt **Name** là `ridenow-prod-system`.
3. Đặt **Pod execution role** là `ridenow-prod-eks-fargate-pod-execution-role`.
4. Ở **Subnets**, chỉ giữ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
5. Chọn **Next**.
6. Ở **Pod selectors**, đặt **Namespace** là `kube-system`. Để trống **Match labels**.
7. Chọn **Add selector**. Đặt **Namespace** là `external-secrets`. Để trống **Match labels**.
8. Chọn **Next**.
9. Chọn **Create**.
10. Chờ 1 đến 2 phút cho đến khi **Status** là `Active`.

### 4. Fargate profile `ridenow-prod-app` (`AppFargateProfile`)

Chỉ bắt đầu profile này sau khi `ridenow-prod-system` ở trạng thái **Active**.

1. Vào **EKS -> Clusters -> `ridenow-prod` -> Compute** tab **-> Fargate profiles -> Add Fargate profile**.
2. Đặt **Name** là `ridenow-prod-app`.
3. Đặt **Pod execution role** là `ridenow-prod-eks-fargate-pod-execution-role`.
4. Ở **Subnets**, chỉ giữ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
5. Chọn **Next**.
6. Ở **Pod selectors**, đặt **Namespace** là `ridenow`. Để trống **Match labels**.
7. Chọn **Next**.
8. Chọn **Create**.
9. Chờ cho đến khi **Status** là `Active`.

### 5. Fargate profile `ridenow-prod-observability` (`ObservabilityFargateProfile`)

Chỉ bắt đầu profile này sau khi `ridenow-prod-app` ở trạng thái **Active**.

1. Vào **EKS -> Clusters -> `ridenow-prod` -> Compute** tab **-> Fargate profiles -> Add Fargate profile**.
2. Đặt **Name** là `ridenow-prod-observability`.
3. Đặt **Pod execution role** là `ridenow-prod-eks-fargate-pod-execution-role`.
4. Ở **Subnets**, chỉ giữ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
5. Chọn **Next**.
6. Ở **Pod selectors**, đặt **Namespace** là `aws-observability`. Để trống **Match labels**.
7. Chọn **Next**.
8. Chọn **Create**.
9. Chờ cho đến khi **Status** là `Active`.

## Values to record

Chạy khối thứ hai của Set variables ngay bây giờ. Các biến dưới đây được tra cứu giúp bạn, nên bạn không sao chép gì bằng tay.

| Biến | Giá trị | Dùng bởi |
|---|---|---|
| `EKS_ENDPOINT` | API server endpoint của `ridenow-prod` | Các lệnh kiểm tra kết nối trong Post-deploy steps |
| `OIDC_ISSUER_URL` | URL của OpenID Connect provider của cluster | 5.9 (OIDC provider của các IRSA role) |
| `OIDC_ID` | Id ở cuối URL của OIDC provider | 5.9 (trust policy của các IRSA role) |
| `EKS_CLUSTER_SG_ID` | ID của security group mà EKS tạo cho cluster | 5.9 (nguồn của các rule database 5432 và Redis 6379) |

## Post-deploy steps

Chạy các bước này từ thư mục gốc của repository, theo thứ tự. Chạy các khối của Set variables trước trong mỗi terminal mới.

Các bước dùng `kubectl`, Helm và AWS CLI. Đây là các CLI exception, vì các đối tượng Kubernetes và việc cài controller không có trang console để tạo. Chỉ các thay đổi IAM policy và security group dùng console.

Các IRSA role `ridenow-prod-alb-controller` và `ridenow-prod-external-secrets` từ 5.9 phải tồn tại trước bước 3 và 4. Hãy tạo các role đó trước, rồi chạy khối thứ ba của Set variables và kiểm tra cả hai role ARN đều có giá trị. Làm bước 3 trước bước 4.

### 1. Kubeconfig

Đây là một CLI exception: kubeconfig là tệp cục bộ nên chỉ CLI mới ghi được. Lệnh dùng danh tính IAM đã tạo cluster.

```bash
aws eks update-kubeconfig --name "$CLUSTER_NAME"
kubectl get svc kubernetes
```

Kết quả mong đợi: `Updated context ...cluster/ridenow-prod`, rồi một dòng `kubernetes   ClusterIP ... 443/TCP`.

### 2. CoreDNS on Fargate

Đây là một CLI exception: patch và restart deployment chỉ có trong Kubernetes API. Fargate không có node. Pod CoreDNS ở trạng thái `Pending` ("no nodes available") trong hai trường hợp. Cả hai được khắc phục bằng cách tạo lại pod khi Fargate profile `ridenow-prod-system` đã **Active**:

- Template CoreDNS vẫn còn annotation ghim nó vào EC2. Hãy xóa annotation đó.
- Pod được tạo trước khi Fargate profile tồn tại. Pod chỉ được ghép với profile lúc nó được tạo, nên các pod này không tự chuyển sang Fargate. Trong trường hợp này annotation có thể đã mất, và chỉ cần restart là đủ.

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

Kết quả mong đợi: `deployment "coredns" successfully rolled out`.

### 3. AWS Load Balancer Controller

Đây là một CLI exception: controller được phát hành dưới dạng Helm chart, và tệp policy phụ thuộc vào phiên bản chart. Cài controller bằng Helm, dùng role ARN và VPC id từ Set variables:

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

Controller chưa có quyền. Nếu thiếu quyền, Ingress không có địa chỉ và controller ghi log `AccessDenied ... elasticloadbalancing:DescribeLoadBalancers`. Permissions policy phụ thuộc vào phiên bản controller, nên hãy tải đúng bản khớp với phiên bản Helm đã cài:

```bash
mkdir -p .tmp
CONTROLLER_VERSION=$(helm list -n kube-system -o json \
  | jq -r '.[] | select(.name=="aws-load-balancer-controller") | .app_version')
echo "controller version: $CONTROLLER_VERSION"
curl -fsSL -o .tmp/alb-controller-iam-policy.json \
  "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/${CONTROLLER_VERSION}/docs/install/iam_policy.json"
```

Kết quả mong đợi: một dòng `controller version: v...` và tệp `.tmp/alb-controller-iam-policy.json`.

Tạo và gắn policy trong IAM console:

1. Vào **IAM -> Policies -> Create policy -> JSON** tab. Dán nội dung của `.tmp/alb-controller-iam-policy.json`.
2. Đặt tên policy là `AWSLoadBalancerControllerIAMPolicy`, rồi tạo. Nếu policy đã có từ cluster trước, hãy dùng lại.
3. Vào **IAM -> Roles -> `ridenow-prod-alb-controller` -> Add permissions -> Attach policies**.
4. Chọn `AWSLoadBalancerControllerIAMPolicy`, rồi chọn **Add permissions**.

Sau đó restart controller để nó nhận quyền:

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
kubectl rollout status  deployment aws-load-balancer-controller -n kube-system --timeout=300s
kubectl get deploy -n kube-system aws-load-balancer-controller
```

Kết quả mong đợi: lệnh cuối hiển thị `1/1` ở cột READY.

### 4. External Secrets Operator

External Secrets phải giải mã các secret RDS bằng khóa KMS từ 5.6 (alias `ridenow-prod-rds`). In ra tài liệu quyền, rồi thêm vào role trong IAM console:

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

1. Vào **IAM -> Roles -> `ridenow-prod-external-secrets` -> Add permissions -> Create inline policy -> JSON** tab.
2. Dán tài liệu mà khối ở trên đã in.
3. Đặt tên policy là `ridenow-prod-external-secrets-kms`, rồi tạo. Bỏ qua bước này nếu inline policy đã tồn tại.

Đây là một CLI exception: operator được phát hành dưới dạng Helm chart.

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

Kết quả mong đợi: ba pod `Running` và các CRD `external-secrets.io` được liệt kê.

### 5. Namespace

Đây là một CLI exception: namespace là đối tượng Kubernetes lấy từ một manifest trong repository.

```bash
kubectl apply -f iac/k8s/00-namespace.yaml
kubectl get namespace ridenow
```

Kết quả mong đợi: namespace `ridenow` có trạng thái `Active`.

### 6. Pod networking (per-pod security groups)

Mỗi service có security group riêng, chỉ được phép truy cập database của chính nó. Tạo sáu group và các rule database của chúng trong EC2 console. Sau đó gán chúng cho pod bằng `SecurityGroupPolicy`.

1. Vào **EC2 -> Security Groups -> Create security group**, mỗi service trong bảng dưới một lần.
2. Đặt **Security group name** và **Description** theo bảng.
3. Đặt **VPC** là `ridenow-prod-vpc`.
4. Để trống **Inbound rules** và giữ outbound rule mặc định (all traffic).
5. Ở **Tags**, thêm `Name` với giá trị bằng tên group. Lệnh tra cứu bên dưới tìm từng group theo tag này.
6. Chọn **Create security group**.

| Tên | Mô tả |
|---|---|
| `ridenow-prod-user-service-sg` | `user-service pods` |
| `ridenow-prod-driver-service-sg` | `driver-service pods` |
| `ridenow-prod-trip-service-sg` | `trip-service pods` |
| `ridenow-prod-payment-service-sg` | `payment-service pods` |
| `ridenow-prod-matching-service-sg` | `matching-service pods` |
| `ridenow-prod-notification-service-sg` | `notification-service pods` |

Sau đó thêm các rule database:

1. Vào **EC2 -> Security Groups -> `ridenow-prod-db-sg` -> Inbound rules -> Edit inbound rules**.
2. Thêm sáu rule. Đặt **Type** là `PostgreSQL`, **Port** là `5432`, và **Source** là từng group trong sáu group ở trên. Bỏ qua rule đã tồn tại.
3. Chọn **Save rules**.

Đây là một CLI exception: `SecurityGroupPolicy` là đối tượng Kubernetes. Khối này đọc sáu group id và áp dụng các policy. `EKS_CLUSTER_SG_ID` vẫn nằm trên mọi pod vì DNS và Redis cần nó:

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

Pod khởi động trước đó giữ security group cũ cho đến khi được restart, vì vậy có lệnh rollout restart. Lệnh này không làm gì ở lần cài đầu tiên. Kết quả mong đợi: sáu dòng `<name> -> sg-...` và sáu dòng `SecurityGroupPolicy`.

### 7. Fargate log router

Đây là một CLI exception: log router đọc một ConfigMap của Kubernetes.

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

Kết quả mong đợi: `namespace/aws-observability created` và `configmap/aws-logging created`.

### 8. Database secrets (ExternalSecrets)

Đây là một CLI exception: `ExternalSecret` là đối tượng Kubernetes, được điền từ manifest trong repository. Bước này cần sáu biến `*_DB_SECRET_ARN` từ Set variables (đều có giá trị) và bản cài External Secrets ở bước 4.

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

Kết quả mong đợi: sáu dòng có `SecretSynced` và READY `True`. Nếu một dòng hiển thị `SecretSyncedError`, hãy đọc lý do:

```bash
kubectl logs -n external-secrets deploy/external-secrets --tail=50 | grep -i "error\|denied"
```

### 9. ALB ingress and smoke test

Hãy triển khai bảy service từ 5.9 trước. Lệnh kiểm tra HTTP 200 bên dưới chỉ thành công sau khi các service đang chạy. Payment phải chạy trước, vì trip-service gọi nó khi tạo chuyến đi. Đây là một CLI exception: Ingress là đối tượng Kubernetes, và controller ở bước 3 tạo load balancer từ nó.

```bash
kubectl apply -f iac/k8s/30-ingress.yaml
kubectl get ingress ridenow -n ridenow
```

Chạy lại lệnh thứ hai cho đến khi cột ADDRESS hiển thị một host name. Sau đó chạy smoke test:

```bash
ALB_DNS_NAME=$(kubectl get ingress ridenow -n ridenow -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB_DNS_NAME=$ALB_DNS_NAME"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://${ALB_DNS_NAME}/api/drivers"
kubectl logs -n ridenow deploy/matching-service --tail=20
kubectl logs -n ridenow deploy/notification-worker --tail=20
```

Kết quả mong đợi: một host name, `HTTP 200`, và log của matching và notification cho thấy consumer SQS và Valkey đã kết nối. Hãy giữ `ALB_DNS_NAME`, vì 5.10 dùng nó làm API origin của CloudFront.

`503` nghĩa là load balancer không có target khỏe. Chạy `kubectl get pods -n ridenow` và chờ đến khi mọi pod `Ready`. Một pod kẹt ở `0/1` thường do sai database endpoint, secret, security group hoặc Redis endpoint. Chạy `kubectl describe pod` trên pod đó để xem lý do.

## Verify

Chạy các lệnh sau trong cùng shell, sau khối thứ hai của Set variables:

```bash
for V in EKS_ENDPOINT OIDC_ISSUER_URL OIDC_ID EKS_CLUSTER_SG_ID; do
  echo "$V=${!V:-MISSING}"
done
aws eks describe-cluster --name "$CLUSTER_NAME" \
  --query 'cluster.[status,version,resourcesVpcConfig.endpointPublicAccess,resourcesVpcConfig.endpointPrivateAccess,encryptionConfig[0].resources[0]]' --output text
aws eks list-fargate-profiles --cluster-name "$CLUSTER_NAME" --output text
```

Kết quả mong đợi: bốn biến có giá trị (không có `MISSING`), cluster `ACTIVE` ở `1.31` với truy cập public và private và `secrets` được mã hóa, cùng ba profile `ridenow-prod-system`, `ridenow-prod-app` và `ridenow-prod-observability`.

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

Nếu thiếu mục nào, hãy mở trang console của tài nguyên đó và kiểm tra tên.
