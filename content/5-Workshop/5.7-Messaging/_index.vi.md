---
title : "Messaging"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

Trang này tạo lớp nhắn tin và cache của môi trường `prod`: một SNS topic, ba SQS queue cùng các dead-letter queue, ba subscription và một Valkey replication group. Các service publish sự kiện chuyến đi lên topic, và mỗi consumer đọc queue riêng của nó.

## Set variables

Chạy khối lệnh này trong một shell mới. Khối này lặp lại các biến dùng chung từ mục 5.2, và tra cứu các tài nguyên mạng từ mục 5.4:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)
[ -n "$VPC_ID" ] && [ "$VPC_ID" != "None" ] || echo "STOP: VPC_ID is empty. Create the network first (5.4)."

PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-private-*" --query 'Subnets[].SubnetId' --output text)
[ -n "$PRIVATE_SUBNET_IDS" ] || echo "STOP: PRIVATE_SUBNET_IDS is empty. Create the private subnets first (5.4)."

CLUSTER_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-cluster-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$CLUSTER_SG_ID" ] && [ "$CLUSTER_SG_ID" != "None" ] || echo "STOP: CLUSTER_SG_ID is empty. Create the cluster security group first (5.4)."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "VPC_ID=$VPC_ID"
echo "PRIVATE_SUBNET_IDS=$PRIVATE_SUBNET_IDS"
echo "CLUSTER_SG_ID=$CLUSTER_SG_ID"
```

Kết quả mong đợi (các ID đã được che, danh sách subnet có hai ID):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
CLUSTER_SG_ID=sg-0eee...
```

Nếu có dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh.

Các biến nhắn tin tra cứu những tài nguyên mà bạn tạo ngay trên trang này. Hãy chạy khối tiếp theo sau khi hoàn thành phần Create the resources, không chạy trước. Mỗi lệnh tra cứu tìm tài nguyên theo tên của nó.

```bash
TRIP_EVENTS_TOPIC_ARN=$(aws sns list-topics --query "Topics[?ends_with(TopicArn, ':${CLUSTER_NAME}-events')].TopicArn | [0]" --output text)
[ -n "$TRIP_EVENTS_TOPIC_ARN" ] && [ "$TRIP_EVENTS_TOPIC_ARN" != "None" ] || echo "STOP: TRIP_EVENTS_TOPIC_ARN is empty. Create the SNS topic first."

MATCHING_QUEUE_URL=$(aws sqs get-queue-url --queue-name "${CLUSTER_NAME}-matching" --query QueueUrl --output text 2>/dev/null)
[ -n "$MATCHING_QUEUE_URL" ] || echo "STOP: MATCHING_QUEUE_URL is empty. Create the matching queue first."
MATCHING_QUEUE_ARN=$(aws sqs get-queue-attributes --queue-url "$MATCHING_QUEUE_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text 2>/dev/null)
[ -n "$MATCHING_QUEUE_ARN" ] || echo "STOP: MATCHING_QUEUE_ARN is empty. Create the matching queue first."

PAYMENT_QUEUE_URL=$(aws sqs get-queue-url --queue-name "${CLUSTER_NAME}-payment" --query QueueUrl --output text 2>/dev/null)
[ -n "$PAYMENT_QUEUE_URL" ] || echo "STOP: PAYMENT_QUEUE_URL is empty. Create the payment queue first."
PAYMENT_QUEUE_ARN=$(aws sqs get-queue-attributes --queue-url "$PAYMENT_QUEUE_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text 2>/dev/null)
[ -n "$PAYMENT_QUEUE_ARN" ] || echo "STOP: PAYMENT_QUEUE_ARN is empty. Create the payment queue first."

NOTIFICATIONS_QUEUE_URL=$(aws sqs get-queue-url --queue-name "${CLUSTER_NAME}-notifications" --query QueueUrl --output text 2>/dev/null)
[ -n "$NOTIFICATIONS_QUEUE_URL" ] || echo "STOP: NOTIFICATIONS_QUEUE_URL is empty. Create the notifications queue first."
NOTIFICATIONS_QUEUE_ARN=$(aws sqs get-queue-attributes --queue-url "$NOTIFICATIONS_QUEUE_URL" --attribute-names QueueArn --query Attributes.QueueArn --output text 2>/dev/null)
[ -n "$NOTIFICATIONS_QUEUE_ARN" ] || echo "STOP: NOTIFICATIONS_QUEUE_ARN is empty. Create the notifications queue first."

REDIS_ENDPOINT=$(aws elasticache describe-replication-groups --replication-group-id "${CLUSTER_NAME}-redis" --query 'ReplicationGroups[0].NodeGroups[0].PrimaryEndpoint.Address' --output text 2>/dev/null)
[ -n "$REDIS_ENDPOINT" ] && [ "$REDIS_ENDPOINT" != "None" ] || echo "STOP: REDIS_ENDPOINT is empty. Create the Valkey replication group and wait until it is Available."

REDIS_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-redis-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$REDIS_SG_ID" ] && [ "$REDIS_SG_ID" != "None" ] || echo "STOP: REDIS_SG_ID is empty. Create the cache security group first."

echo "TRIP_EVENTS_TOPIC_ARN=$TRIP_EVENTS_TOPIC_ARN"
echo "MATCHING_QUEUE_URL=$MATCHING_QUEUE_URL"
echo "MATCHING_QUEUE_ARN=$MATCHING_QUEUE_ARN"
echo "PAYMENT_QUEUE_URL=$PAYMENT_QUEUE_URL"
echo "PAYMENT_QUEUE_ARN=$PAYMENT_QUEUE_ARN"
echo "NOTIFICATIONS_QUEUE_URL=$NOTIFICATIONS_QUEUE_URL"
echo "NOTIFICATIONS_QUEUE_ARN=$NOTIFICATIONS_QUEUE_ARN"
echo "REDIS_ENDPOINT=$REDIS_ENDPOINT"
echo "REDIS_SG_ID=$REDIS_SG_ID"
```

Kết quả mong đợi (các ID đã được che):

```text
TRIP_EVENTS_TOPIC_ARN=arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events
MATCHING_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-matching
MATCHING_QUEUE_ARN=arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-matching
PAYMENT_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-payment
PAYMENT_QUEUE_ARN=arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-payment
NOTIFICATIONS_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-notifications
NOTIFICATIONS_QUEUE_ARN=arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-notifications
REDIS_ENDPOINT=ridenow-prod-redis.abc123.apse1.cache.amazonaws.com
REDIS_SG_ID=sg-0aaa...
```

## What it creates

| Template logical ID | AWS resource type | Resolved name |
|---|---|---|
| `TripEventTopic` | SNS topic | `ridenow-prod-events` |
| `NotificationDlq` | SQS queue | `ridenow-prod-notifications-dlq` |
| `NotificationQueue` | SQS queue | `ridenow-prod-notifications` |
| `MatchingDlq` | SQS queue | `ridenow-prod-matching-dlq` |
| `MatchingQueue` | SQS queue | `ridenow-prod-matching` |
| `PaymentDlq` | SQS queue | `ridenow-prod-payment-dlq` |
| `PaymentQueue` | SQS queue | `ridenow-prod-payment` |
| `MatchingSubscription` | SNS subscription | topic `ridenow-prod-events` to `ridenow-prod-matching` |
| `PaymentSubscription` | SNS subscription | topic `ridenow-prod-events` to `ridenow-prod-payment` |
| `NotificationSubscription` | SNS subscription | topic `ridenow-prod-events` to `ridenow-prod-notifications` |
| `MatchingQueuePolicy` | SQS queue policy | on `ridenow-prod-matching` |
| `PaymentQueuePolicy` | SQS queue policy | on `ridenow-prod-payment` |
| `NotificationQueuePolicy` | SQS queue policy | on `ridenow-prod-notifications` |
| `RedisSubnetGroup` | ElastiCache subnet group | `ridenow-prod-redis-subnets` |
| `RedisSecurityGroup` | EC2 security group | `ridenow-prod-redis-sg` |
| `RedisReplicationGroup` | ElastiCache replication group | `ridenow-prod-redis` |

Ba work queue có visibility timeout 60 giây và chuyển message sang dead-letter queue sau 5 lần nhận. Mỗi dead-letter queue giữ message 14 ngày. Subscription matching chỉ cho qua sự kiện `trip.requested`. Subscription payment cho qua `trip.completed` và `trip.cancelled`. Subscription notifications cho qua mọi sự kiện. Nhóm Valkey có một node primary và một node replica loại `cache.t4g.small`, với Multi-AZ và automatic failover.

## Depends on

5.4 Network. Bạn cần các tài nguyên sau trước khi bắt đầu:

| Tài nguyên | Tên | Dùng cho |
|---|---|---|
| VPC | `ridenow-prod-vpc` | Security group của cache |
| Private subnet | `ridenow-prod-private-1`, `ridenow-prod-private-2` | Subnet group của cache |
| Security group | `ridenow-prod-cluster-sg` | Nguồn của rule cổng 6379 |

## Create the resources

Mở AWS console ở region Asia Pacific (Singapore) `ap-southeast-1`. Làm theo các bước theo thứ tự.

### 1. SNS topic (`TripEventTopic`)

1. Vào **Amazon SNS -> Topics -> Create topic**.
2. Đặt **Type** là `Standard`.
3. Đặt **Name** là `ridenow-prod-events`.
4. Để **Encryption** ở trạng thái tắt (khoá `aws/sns` chặn việc gửi sang SQS).
5. Giữ **Access policy** ở mặc định.
6. Chọn **Create topic**.

### 2. Dead-letter queue (`NotificationDlq`, `MatchingDlq`, `PaymentDlq`)

Lặp lại danh sách này ba lần, với các tên `ridenow-prod-notifications-dlq`, `ridenow-prod-matching-dlq` và `ridenow-prod-payment-dlq`.

1. Vào **Amazon SQS -> Queues -> Create queue**.
2. Đặt **Type** là `Standard`.
3. Đặt **Name** là một trong ba tên DLQ.
4. Đặt **Visibility timeout** là `30` **Seconds** (mặc định).
5. Đặt **Message retention period** là `14` **Days**.
6. Đặt **Server-side encryption** là `Enabled`.
7. Đặt **Encryption key type** là `Amazon SQS key (SSE-SQS)`.
8. Giữ **Access policy** ở `Basic` (mặc định).
9. Đặt **Dead-letter queue** là `Disabled`.
10. Chọn **Create queue**.

### 3. Work queue và access policy (`NotificationQueue`, `MatchingQueue`, `PaymentQueue`)

In ra ba access policy. Chúng chứa account ID và Region của bạn, nên bạn không phải gõ tay:

```bash
for Q in matching payment notifications; do
  echo "Access policy for ${CLUSTER_NAME}-${Q}:"
  cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRideNowTopic",
      "Effect": "Allow",
      "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:${CLUSTER_NAME}-${Q}",
      "Condition": { "ArnEquals": { "aws:SourceArn": "arn:aws:sns:${AWS_REGION}:${ACCOUNT_ID}:${CLUSTER_NAME}-events" } }
    }
  ]
}
EOF
done
```

Lặp lại danh sách dưới đây ba lần, với các cặp queue và dead-letter queue sau:

| Tên queue | Dead-letter queue |
|---|---|
| `ridenow-prod-matching` | `ridenow-prod-matching-dlq` |
| `ridenow-prod-payment` | `ridenow-prod-payment-dlq` |
| `ridenow-prod-notifications` | `ridenow-prod-notifications-dlq` |

1. Vào **Amazon SQS -> Queues -> Create queue**.
2. Đặt **Type** là `Standard`.
3. Đặt **Name** là tên queue trong bảng.
4. Đặt **Visibility timeout** là `1` **Minutes** (60 giây).
5. Đặt **Message retention period** là `4` **Days** (mặc định).
6. Đặt **Server-side encryption** là `Enabled`.
7. Đặt **Encryption key type** là `Amazon SQS key (SSE-SQS)`.
8. Trong **Access policy**, chọn **Advanced**.
9. Thay JSON bằng policy mà khối lệnh ở trên đã in ra cho queue này.
10. Đặt **Dead-letter queue** là `Enabled`.
11. Trong **Choose queue**, chọn dead-letter queue trong bảng.
12. Đặt **Maximum receives** là `5`.
13. Chọn **Create queue**.

### 4. Subscription (`MatchingSubscription`, `PaymentSubscription`, `NotificationSubscription`)

In ra ba ARN của queue. Bạn dán từng ARN vào **Endpoint** của subscription tương ứng:

```bash
for Q in matching payment notifications; do
  echo "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:${CLUSTER_NAME}-${Q}"
done
```

Subscription matching:

1. Vào **Amazon SNS -> Topics -> `ridenow-prod-events` -> Subscriptions** tab -> **Create subscription**.
2. Đặt **Protocol** là `Amazon SQS`.
3. Đặt **Endpoint** là ARN đã in ra cho `ridenow-prod-matching`.
4. Đánh dấu **Enable raw message delivery**.
5. Mở rộng **Subscription filter policy** và bật nó.
6. Đặt **Filter policy scope** là `Message body`.
7. Dán bộ lọc dưới đây vào trình soạn JSON.
8. Để **Redrive policy (dead-letter queue)** tắt.
9. Chọn **Create subscription**.

```json
{
  "type": ["trip.requested"]
}
```

Subscription payment:

1. Vào **Amazon SNS -> Topics -> `ridenow-prod-events` -> Subscriptions** tab -> **Create subscription**.
2. Đặt **Protocol** là `Amazon SQS`.
3. Đặt **Endpoint** là ARN đã in ra cho `ridenow-prod-payment`.
4. Đánh dấu **Enable raw message delivery**.
5. Mở rộng **Subscription filter policy** và bật nó.
6. Đặt **Filter policy scope** là `Message body`.
7. Dán bộ lọc dưới đây vào trình soạn JSON.
8. Để **Redrive policy (dead-letter queue)** tắt.
9. Chọn **Create subscription**.

```json
{
  "type": ["trip.completed", "trip.cancelled"]
}
```

Subscription notification:

1. Vào **Amazon SNS -> Topics -> `ridenow-prod-events` -> Subscriptions** tab -> **Create subscription**.
2. Đặt **Protocol** là `Amazon SQS`.
3. Đặt **Endpoint** là ARN đã in ra cho `ridenow-prod-notifications`.
4. Đánh dấu **Enable raw message delivery**.
5. Để **Subscription filter policy** tắt (nó nhận mọi sự kiện).
6. Để **Redrive policy (dead-letter queue)** tắt.
7. Chọn **Create subscription**.

Mỗi subscription hiện Status `Confirmed`. Nếu có subscription hiện `Pending confirmation`, hãy kiểm tra lại access policy của queue ở bước 3.

### 5. Subnet group của cache (`RedisSubnetGroup`)

1. Vào **Amazon ElastiCache -> Subnet groups -> Create subnet group**.
2. Đặt **Name** là `ridenow-prod-redis-subnets`.
3. Đặt **Description** là `ridenow-prod Redis private subnets`.
4. Đặt **VPC ID** là `ridenow-prod-vpc`.
5. Trong **Selected subnets**, chọn các Availability Zone trước, rồi đánh dấu `ridenow-prod-private-1` và `ridenow-prod-private-2`.
6. Chọn **Create**.

### 6. Security group của cache (`RedisSecurityGroup`)

1. Vào **EC2 -> Security Groups -> Create security group**.
2. Đặt **Security group name** là `ridenow-prod-redis-sg`.
3. Đặt **Description** là `ridenow-prod Redis (private)`.
4. Đặt **VPC** là `ridenow-prod-vpc`.
5. Trong **Inbound rules**, chọn **Add rule**.
6. Đặt **Type** là `Custom TCP`.
7. Đặt **Port range** là `6379`.
8. Đặt **Source** là `Custom`, rồi chọn security group `ridenow-prod-cluster-sg`.
9. Đặt **Description** là `Redis 6379 from EKS nodes only`.
10. Giữ **Outbound rules** ở mặc định (`All traffic`, `0.0.0.0/0`).
11. Trong **Tags**, chọn **Add new tag**. Đặt **Key** là `Name` và **Value** là `ridenow-prod-redis-sg`. Lệnh tra cứu biến tìm nhóm theo tag này.
12. Chọn **Create security group**.

### 7. Valkey replication group (`RedisReplicationGroup`)

1. Vào **Amazon ElastiCache -> Valkey caches -> Create Valkey cache**.
2. Đặt **Deployment option** là `Design your own cache`.
3. Đặt **Creation method** là `Cluster cache`.
4. Đặt **Cluster mode** là `Disabled`.
5. Đặt **Name** là `ridenow-prod-redis`.
6. Đặt **Description** là `ridenow-prod driver GEO / live state`.
7. Đặt **Engine version** là `8.0`.
8. Đặt **Port** là `6379`.
9. Đặt **Parameter group** là `default.valkey8`.
10. Đặt **Node type** là `cache.t4g.small`.
11. Đặt **Number of replicas** là `1`.
12. Đặt **Multi-AZ** là `Enable`.
13. Đặt **Auto-failover** là `Enable`.
14. Trong **Connectivity**, đặt **Subnet group** là **Choose existing subnet group**, rồi chọn `ridenow-prod-redis-subnets`.
15. Đặt **Availability Zone placements** là `No preference`.
16. Trong **Security**, đặt **Encryption at rest** là `Enable` (khoá mặc định).
17. Đặt **Encryption in transit** là `Enable`.
18. Đặt **Access control** là `No access control`.
19. Trong **Selected security groups**, chọn **Manage**.
20. Đánh dấu `ridenow-prod-redis-sg` và bỏ `default`.
21. Trong **Backup**, bỏ đánh dấu **Enable automatic backups**.
22. Chọn **Create**.
23. Chờ 10 đến 15 phút đến khi **Status** là `Available`.

## Values to record

Hãy chạy khối thứ hai của phần Set variables ngay bây giờ. Các biến dưới đây được tra cứu tự động, nên bạn không phải sao chép gì bằng tay.

| Biến | Giá trị | Được dùng bởi |
|---|---|---|
| `TRIP_EVENTS_TOPIC_ARN` | ARN của topic `ridenow-prod-events` | 5.9 (các workload của service publish sự kiện lên đó) |
| `MATCHING_QUEUE_URL` | URL của `ridenow-prod-matching` | 5.9 (workload matching-service) |
| `MATCHING_QUEUE_ARN` | ARN của `ridenow-prod-matching` | Kiểm tra kết nối |
| `PAYMENT_QUEUE_URL` | URL của `ridenow-prod-payment` | 5.9 (workload payment-service) |
| `PAYMENT_QUEUE_ARN` | ARN của `ridenow-prod-payment` | Kiểm tra kết nối |
| `NOTIFICATIONS_QUEUE_URL` | URL của `ridenow-prod-notifications` | 5.9 (workload notification-worker) |
| `NOTIFICATIONS_QUEUE_ARN` | ARN của `ridenow-prod-notifications` | Kiểm tra kết nối |
| `REDIS_ENDPOINT` | Host primary endpoint của `ridenow-prod-redis` (không có cổng) | 5.9 (các workload driver, matching, notification và realtime) |
| `REDIS_SG_ID` | ID của `ridenow-prod-redis-sg` | 5.9 (rule ingress từ security group của EKS cluster) |

## Post-deploy steps

Trang này không cần bước nào sau khi triển khai. Ở 5.9 bạn mở `ridenow-prod-redis-sg` cho các Fargate pod.

## Verify

Chạy các lệnh sau trong cùng shell, sau khối thứ hai của phần Set variables:

```bash
echo "TRIP_EVENTS_TOPIC_ARN=${TRIP_EVENTS_TOPIC_ARN:-MISSING}"
echo "MATCHING_QUEUE_URL=${MATCHING_QUEUE_URL:-MISSING}"
echo "PAYMENT_QUEUE_URL=${PAYMENT_QUEUE_URL:-MISSING}"
echo "NOTIFICATIONS_QUEUE_URL=${NOTIFICATIONS_QUEUE_URL:-MISSING}"
echo "REDIS_ENDPOINT=${REDIS_ENDPOINT:-MISSING}"
echo "REDIS_SG_ID=${REDIS_SG_ID:-MISSING}"
aws sns list-subscriptions-by-topic --topic-arn "$TRIP_EVENTS_TOPIC_ARN" --query 'Subscriptions[].[Endpoint,SubscriptionArn]' --output text
aws elasticache describe-replication-groups --replication-group-id "${CLUSTER_NAME}-redis" --query 'ReplicationGroups[0].[Status,MultiAZ,AutomaticFailover]' --output text
```

Kết quả mong đợi: sáu biến có giá trị (không có `MISSING`), ba subscription với ARN đầy đủ (không phải `PendingConfirmation`), và nhóm cache ở trạng thái available với Multi-AZ và automatic failover được bật.

```text
TRIP_EVENTS_TOPIC_ARN=arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events
MATCHING_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-matching
PAYMENT_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-payment
NOTIFICATIONS_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-notifications
REDIS_ENDPOINT=ridenow-prod-redis.abc123.apse1.cache.amazonaws.com
REDIS_SG_ID=sg-0aaa...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-matching	arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:1111...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-payment	arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:2222...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-notifications	arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:3333...
available	enabled	enabled
```

Nếu thiếu giá trị nào, hãy mở trang console của tài nguyên đó và kiểm tra tên.
