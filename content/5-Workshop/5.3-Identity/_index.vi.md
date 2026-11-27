---
title : "Định danh"
date : 2024-01-01 
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

Trang này tạo hai IAM role mà cluster EKS cần và năm customer managed policy trong IAM console. IAM là dịch vụ toàn cầu, nên region bạn mở trong console không ảnh hưởng.

## Set variables

Chạy khối lệnh này trong shell Bash. Khối này lặp lại các biến dùng chung từ [5.2](../5.2-Prerequiste/). JSON của policy bên dưới dùng account ID và region của bạn, nên hãy giữ kết quả hiển thị để đối chiếu.

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

Kết quả mong đợi (account ID đã được che):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
```

Ở chỗ nào policy bên dưới có `{ACCOUNT_ID}` hoặc `{AWS_REGION}`, hãy thay toàn bộ token, gồm cả dấu ngoặc nhọn, bằng giá trị đã in ở trên.

## What it creates

| Loại tài nguyên AWS | Tên |
|---|---|
| IAM role | `ridenow-prod-eks-cluster-role` |
| IAM role | `ridenow-prod-eks-fargate-pod-execution-role` |
| IAM customer managed policy | `ridenow-prod-app-access` |
| IAM customer managed policy | `ridenow-prod-trip-service-messaging` |
| IAM customer managed policy | `ridenow-prod-payment-service-messaging` |
| IAM customer managed policy | `ridenow-prod-matching-service-messaging` |
| IAM customer managed policy | `ridenow-prod-notification-worker-messaging` |

## Depends on

Không có. Bạn chỉ cần công cụ và danh tính đã chuẩn bị ở 5.2. Bảy tài nguyên này có thể tạo theo thứ tự bất kỳ.

## Create the resources

### 1. Role `ridenow-prod-eks-cluster-role`

1. Mở **IAM**, chọn **Roles**, rồi chọn **Create role**.
2. Đặt **Trusted entity type** là **Custom trust policy**.
3. Xóa JSON mặc định trong ô **Custom trust policy** và dán JSON bên dưới. Chọn **Next**.

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

4. Tìm từng AWS managed policy sau và chọn: `AmazonEKSClusterPolicy`, `AmazonEKSComputePolicy`, `AmazonEKSBlockStoragePolicyV2`, `AmazonEKSLoadBalancingPolicy`, `AmazonEKSNetworkingPolicy`. Chọn **Next**.
5. Đặt **Role name** là `ridenow-prod-eks-cluster-role`. Giữ nguyên **Description** mặc định.
6. Chọn **Create role**.

### 2. Role `ridenow-prod-eks-fargate-pod-execution-role`

1. Mở **IAM**, chọn **Roles**, rồi chọn **Create role**.
2. Đặt **Trusted entity type** là **Custom trust policy**.
3. Xóa JSON mặc định và dán JSON bên dưới. Chọn **Next**.

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

4. Tìm `AmazonEKSFargatePodExecutionRolePolicy` và chọn. Chọn **Next**.
5. Đặt **Role name** là `ridenow-prod-eks-fargate-pod-execution-role`. Giữ nguyên **Description** mặc định.
6. Chọn **Create role**.

### 3. Policy `ridenow-prod-app-access`

1. Mở **IAM**, chọn **Policies**, rồi chọn **Create policy**.
2. Đặt **Policy editor** là **JSON**, xóa JSON mặc định và dán JSON bên dưới.
3. Thay mỗi `{ACCOUNT_ID}` bằng account ID của bạn. Giữ nguyên các phần region là `*`. Chọn **Next**.

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

4. Đặt **Policy name** là `ridenow-prod-app-access`.
5. Đặt **Description** là đoạn văn bản bên dưới. Không sửa được description sau khi tạo, nên hãy gõ chính xác.

```text
Scoped runtime access for RideNow pods via IRSA: pull images, read DB secrets, and (where present) use the environment's queues and event bus.
```

6. Chọn **Create policy**.

### 4. Policy `ridenow-prod-trip-service-messaging`

1. Mở **IAM**, chọn **Policies**, rồi chọn **Create policy**.
2. Đặt **Policy editor** là **JSON**, xóa JSON mặc định và dán JSON bên dưới.
3. Thay mỗi `{ACCOUNT_ID}` và `{AWS_REGION}` bằng giá trị của bạn. Chọn **Next**.

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

4. Đặt **Policy name** là `ridenow-prod-trip-service-messaging`.
5. Đặt **Description** là `trip-service publishes trip events to the RideNow topic.`
6. Chọn **Create policy**.

### 5. Policy `ridenow-prod-payment-service-messaging`

1. Mở **IAM**, chọn **Policies**, rồi chọn **Create policy**.
2. Đặt **Policy editor** là **JSON**, xóa JSON mặc định và dán JSON bên dưới.
3. Thay mỗi `{ACCOUNT_ID}` và `{AWS_REGION}` bằng giá trị của bạn. Chọn **Next**.

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

4. Đặt **Policy name** là `ridenow-prod-payment-service-messaging`.
5. Đặt **Description** là `payment-service publishes payment events and consumes the payment queue.`
6. Chọn **Create policy**.

### 6. Policy `ridenow-prod-matching-service-messaging`

1. Mở **IAM**, chọn **Policies**, rồi chọn **Create policy**.
2. Đặt **Policy editor** là **JSON**, xóa JSON mặc định và dán JSON bên dưới.
3. Thay mỗi `{ACCOUNT_ID}` và `{AWS_REGION}` bằng giá trị của bạn. Chọn **Next**.

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

4. Đặt **Policy name** là `ridenow-prod-matching-service-messaging`.
5. Đặt **Description** là `matching-service publishes match events and consumes the matching queue.`
6. Chọn **Create policy**.

### 7. Policy `ridenow-prod-notification-worker-messaging`

1. Mở **IAM**, chọn **Policies**, rồi chọn **Create policy**.
2. Đặt **Policy editor** là **JSON**, xóa JSON mặc định và dán JSON bên dưới.
3. Thay mỗi `{ACCOUNT_ID}` và `{AWS_REGION}` bằng giá trị của bạn. Chọn **Next**.

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

4. Đặt **Policy name** là `ridenow-prod-notification-worker-messaging`.
5. Đặt **Description** là `notification-worker consumes the notifications queue.`
6. Chọn **Create policy**.

## Values to record

Hai ARN của role được dùng khi bạn tạo cluster EKS ở 5.8. Hãy nạp chúng vào biến ngay bây giờ. Khối lệnh tra cứu từng role theo tên.

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

Kết quả mong đợi (account ID đã được che):

```text
CLUSTER_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-eks-cluster-role
FARGATE_POD_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-eks-fargate-pod-execution-role
```

Nếu thiếu một role, lệnh in lỗi của AWS và biến vẫn rỗng. Hãy sửa tên role rồi chạy lại khối lệnh.

Năm policy chưa cần biến nào. Các messaging policy được gắn vào service role ở 5.9, nơi bạn chọn chúng theo tên trong console.

## Post-deploy steps

Trang này không cần bước nào sau khi tạo.

## Verify

Chạy khối lệnh này:

```bash
aws iam list-policies --scope Local \
  --query "Policies[?starts_with(PolicyName,'ridenow-$ENV-')].PolicyName" --output text | tr '\t' '\n'
echo "$CLUSTER_ROLE_ARN"
echo "$FARGATE_POD_ROLE_ARN"
```

Kết quả mong đợi (account ID đã được che; thứ tự các policy có thể khác). Nếu thiếu dòng nào, tài nguyên đó chưa được tạo hoặc bị sai tên:

```text
ridenow-prod-app-access
ridenow-prod-trip-service-messaging
ridenow-prod-payment-service-messaging
ridenow-prod-matching-service-messaging
ridenow-prod-notification-worker-messaging
arn:aws:iam::123456789012:role/ridenow-prod-eks-cluster-role
arn:aws:iam::123456789012:role/ridenow-prod-eks-fargate-pod-execution-role
```

Tiếp theo: [5.4 Network](../5.4-Network/).
