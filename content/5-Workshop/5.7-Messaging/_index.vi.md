---
title : "Messaging & Cache"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

Trong phần này, chúng ta xây dựng lớp **messaging** và **cache** cho môi trường `prod` của RideNow. Đây là nơi các service trao đổi sự kiện chuyến đi với nhau, và là nơi lưu vị trí trực tiếp của tài xế.

## Tổng quan

Khi một chuyến đi thay đổi trạng thái, `trip-service` không gọi trực tiếp từng service khác. Thay vào đó, nó publish một sự kiện lên **Amazon SNS topic** `ridenow-prod-events`. SNS sao chép sự kiện sang các **Amazon SQS queue**, và mỗi service đọc queue riêng của nó:

| Sự kiện (`type`) | `ridenow-prod-matching` | `ridenow-prod-payment` | `ridenow-prod-notifications` |
|---|---|---|---|
| `trip.requested` | ✅ matching-service tìm tài xế | | ✅ |
| `trip.completed` | | ✅ payment-service tính tiền | ✅ |
| `trip.cancelled` | | ✅ | ✅ |

Cách tổ chức này mang lại ba lợi ích:

- **Tách rời các service**: `trip-service` không cần biết có bao nhiêu service đang lắng nghe. Muốn thêm một consumer mới, chúng ta chỉ cần thêm một subscription.
- **Không mất sự kiện**: nếu một service tạm ngừng, sự kiện vẫn nằm chờ trong queue cho đến khi service chạy lại.
- **Cô lập lỗi**: một message xử lý thất bại 5 lần sẽ được chuyển sang **dead-letter queue (DLQ)**, nên nó không chặn các message khác, và chúng ta có 14 ngày để điều tra.

Bên cạnh đó, chúng ta tạo một **ElastiCache for Valkey** (tương thích giao thức Redis, rẻ hơn Redis OSS khoảng 20%). Valkey lưu vị trí GEO và trạng thái trực tiếp của tài xế, kênh Pub/Sub vị trí, và tín hiệu kết thúc chuyến đi.

> **Chi phí:** SNS và SQS gần như miễn phí khi không có traffic. Valkey (hai node `cache.t4g.small`) bắt đầu tính tiền theo giờ ngay khi được tạo. Hãy xoá nó theo [5.11](../5.11-Cleanup/) khi không dùng nữa.

## Thiết lập biến

**1.** Mở một shell Bash mới và chạy khối lệnh dưới đây. Khối này gán lại các biến dùng chung từ [5.2](../5.2-Prerequiste/), và tra cứu các tài nguyên mạng đã tạo ở [5.4](../5.4-Network/):

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

Chúng ta gán lại biến ở mỗi phần vì bạn có thể làm từng phần trong một shell khác nhau. Nếu có dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại. Dòng `STOP` về `VPC_ID`, `PRIVATE_SUBNET_IDS` hoặc `CLUSTER_SG_ID` nghĩa là bạn cần hoàn thành 5.4 trước.

**2.** Khối lệnh thứ hai tra cứu các tài nguyên mà chúng ta sẽ tạo trong phần này. **Chưa chạy nó bây giờ.** Chúng ta sẽ quay lại chạy nó ở mục Giá trị cần ghi lại, sau khi tạo xong mọi tài nguyên.

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
REDIS_ENDPOINT=master.ridenow-prod-redis.abc123.apse1.cache.amazonaws.com
REDIS_SG_ID=sg-0aaa...
```

## Tài nguyên được tạo

Sau phần này, tài khoản của bạn có 16 tài nguyên sau:

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

## Điều kiện tiên quyết

Phần này cần ba tài nguyên từ [5.4 Network](../5.4-Network/):

| Tài nguyên | Tên | Dùng cho |
|---|---|---|
| VPC | `ridenow-prod-vpc` | Subnet group và security group của Valkey |
| Private subnet | `ridenow-prod-private-1` (`ap-southeast-1a`), `ridenow-prod-private-2` (`ap-southeast-1b`) | Subnet group của Valkey |
| Security group | `ridenow-prod-cluster-sg` | Nguồn được phép kết nối tới Valkey |

SNS và SQS không nằm trong VPC, nên bạn có thể làm bước 1 đến bước 4 trước khi 5.4 xong. Từ bước 5 (Valkey) trở đi, cả ba tài nguyên trên phải có sẵn.

## Tạo các tài nguyên

Mở AWS console và chọn region **Asia Pacific (Singapore)** `ap-southeast-1` ở góc trên bên phải. Hãy làm theo các bước theo đúng thứ tự.

### 1. Tạo SNS topic

**1.** Mở **Amazon SNS**, chọn **Topics** ở menu bên trái, rồi chọn **Create topic**.

![SNS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/01-01-sns-topics-list.png)

**2.** Điền form như sau:

- **Type**: `Standard`
- **Name**: `ridenow-prod-events`
- **Encryption**: để **tắt**
- **Access policy**: giữ mặc định

![SNS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/01-02-sns-create-form_1.png)

Chúng ta chọn `Standard` vì các consumer không cần thứ tự tuyệt đối và cần thông lượng cao. Encryption phải tắt vì khoá mặc định `aws/sns` không cho SNS gửi message sang SQS: nếu bật, các queue sẽ không nhận được gì mà console cũng không báo lỗi.

**3.** Chọn **Create topic**. Trang chi tiết hiện ra với **ARN** có dạng `arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events`.

![SNS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/01-03-sns-topic-created.png)

### 2. Tạo ba dead-letter queue

Chúng ta tạo DLQ trước, vì khi tạo work queue ở bước 3, mỗi queue phải chọn DLQ của nó từ danh sách có sẵn.

**1.** Mở **Amazon SQS**, chọn **Queues**, rồi chọn **Create queue**.

![SQS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-01-sqs-queues-list.png)

**2.** Điền phần **Details** và **Configuration**:

- **Type**: `Standard`
- **Name**: `ridenow-prod-notifications-dlq`
- **Visibility timeout**: `30` **Seconds** (mặc định)
- **Message retention period**: `14` **Days**

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-02-dlq-notifications.png)

Mặc định SQS chỉ giữ message 4 ngày. Với DLQ, chúng ta tăng lên mức tối đa 14 ngày để có đủ thời gian điều tra những message lỗi trước khi chúng bị xoá.

> **Lưu ý:** tên queue và **Type** không đổi được sau khi tạo. Nếu gõ sai tên, bạn phải xoá queue rồi tạo lại. Hãy kiểm tra kỹ tên trước khi bấm **Create queue**.

**3.** Trong phần **Encryption**, đặt **Server-side encryption** là `Enabled` và **Encryption key type** là `Amazon SQS key (SSE-SQS)`.

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-03-dlq-encryption.png)

SSE-SQS mã hoá message khi lưu trữ bằng khoá do SQS quản lý, không tốn thêm chi phí và không cần cấu hình quyền KMS.

**4.** Giữ **Access policy** ở `Basic`, đặt **Dead-letter queue** là `Disabled`, rồi chọn **Create queue**. Bản thân DLQ không cần một DLQ khác.

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-04-dlq-created_1.png)
![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-04-dlq-created_2.png)

**5.** Lặp lại bước 1 đến bước 4 thêm hai lần, với tên `ridenow-prod-matching-dlq` và `ridenow-prod-payment-dlq`. Danh sách **Queues** giờ có ba DLQ:

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-05-dlq-list.png)

### 3. Tạo ba work queue

Work queue là nơi các service thật sự đọc sự kiện. Mỗi queue cần một **access policy** chỉ cho phép topic `ridenow-prod-events` gửi message vào nó.

**1.** Trên console, chọn **Create queue** và điền phần **Details** và **Configuration**:

- **Type**: `Standard`
- **Name**: `ridenow-prod-matching`
- **Visibility timeout**: `1` **Minutes**
- **Message retention period**: `4` **Days** (mặc định)

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-01-queue-configuration.png)

Visibility timeout 60 giây là khoảng thời gian một message bị ẩn khỏi các consumer khác trong lúc một consumer đang xử lý nó. Nếu consumer không xử lý xong trong 60 giây, message sẽ hiện lại để được thử lại.

**2.** Trong phần **Encryption**, đặt **Server-side encryption** là `Enabled` và **Encryption key type** là `Amazon SQS key (SSE-SQS)`.

**3.** Trong phần **Access policy**, chọn **Advanced**. Xoá toàn bộ JSON có sẵn, rồi dán policy đã in ra cho `ridenow-prod-matching`.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRideNowTopic",
      "Effect": "Allow",
      "Principal": { "Service": "sns.amazonaws.com" },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:ap-southeast-1:ACCOUNT_ID:ridenow-prod-matching",
      "Condition": { "ArnEquals": { "aws:SourceArn": "arn:aws:sns:ap-southeast-1:ACCOUNT_ID:ridenow-prod-events" } }
    }
  ]
}
```

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-02-queue-access-policy.png)

> **Lưu ý:** dòng `Resource` trong policy phải kết thúc bằng đúng tên queue đang tạo. Nếu dán nhầm policy của queue khác, console vẫn tạo queue mà không báo lỗi, nhưng SNS sẽ không gửi được message vào queue này.

**4.** Trong phần **Dead-letter queue**, đặt nó là `Enabled`, chọn `ridenow-prod-matching-dlq` ở **Choose queue**, và đặt **Maximum receives** là `5`.

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-03-queue-dlq-setting.png)

Với cấu hình này, một message bị nhận 5 lần mà vẫn chưa được xoá (tức là xử lý thất bại 5 lần) sẽ được SQS tự động chuyển sang `ridenow-prod-matching-dlq`.

**5.** Chọn **Create queue**.

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-04-queue-created.png)

**6.** Lặp lại bước 2 đến bước 6 cho hai queue còn lại, mỗi queue dùng policy và DLQ của chính nó:

| Tên queue | Dead-letter queue |
|---|---|
| `ridenow-prod-payment` | `ridenow-prod-payment-dlq` |
| `ridenow-prod-notifications` | `ridenow-prod-notifications-dlq` |

Danh sách **Queues** giờ có đủ sáu queue:

<!-- IMAGE: 03-05-queue-list.png | danh sách Queues với sáu queue | path gợi ý: /images/5-Workshop/5.7-Messaging/03-05-queue-list.png -->
![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-05-queue-list.png)

### 4. Tạo ba subscription

Subscription nối topic với từng queue. **Filter policy** của subscription quyết định queue nào nhận loại sự kiện nào.

**1.** Chạy lệnh sau để in ra ARN của ba queue. Bạn sẽ dán chúng vào ô **Endpoint**:

```bash
for Q in matching payment notifications; do
  echo "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:${CLUSTER_NAME}-${Q}"
done
```

**2.** Mở **Amazon SNS**, chọn **Topics**, chọn `ridenow-prod-events`, mở tab **Subscriptions**, rồi chọn **Create subscription**.

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-01-topic-subscription-click-create.png)

**3.** Điền form cho subscription **matching**:

- **Topic ARN**: đã được điền sẵn là ARN của `ridenow-prod-events`
- **Protocol**: `Amazon SQS`
- **Endpoint**: ARN đã in ra cho `ridenow-prod-matching`
- **Enable raw message delivery**: đánh dấu
- **Subscription filter policy**: bật, đặt **Filter policy scope** là `Message body`, rồi dán bộ lọc:

```json
{
  "type": ["trip.requested"]
}
```

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-01-topic-subscriptions-settings.png)
![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-02-matching-sub-form.png)

**Raw message delivery** giúp body của message trong SQS chính là JSON của sự kiện, không bị bọc trong lớp vỏ SNS, nên service đọc được ngay. **Filter policy scope** phải là `Message body` vì bộ lọc so sánh trường `type` trong body. Nếu để mặc định `Message attributes`, bộ lọc sẽ tìm trong attribute (mà publisher không gửi), và queue sẽ không bao giờ nhận được message nào.

**4.** Để **Redrive policy (dead-letter queue)** tắt, rồi chọn **Create subscription**. **Status** của subscription phải là `Confirmed`.
![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-03-matching-sub-created.png)

**5.** Lặp lại bước 2 đến bước 4 cho subscription **payment**. Đặt **Endpoint** là ARN của `ridenow-prod-payment`, và dùng bộ lọc sau (scope vẫn là `Message body`):

```json
{
  "type": ["trip.completed", "trip.cancelled"]
}
```

**6.** Lặp lại bước 2 đến bước 4 cho subscription **notification**. Đặt **Endpoint** là ARN của `ridenow-prod-notifications`, vẫn đánh dấu **Enable raw message delivery**, nhưng **để Subscription filter policy tắt**. Queue này cần nhận mọi sự kiện để `notification-worker` gửi thông báo cho người dùng.

Tab **Subscriptions** giờ có ba dòng, tất cả ở trạng thái `Confirmed`:

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-04-topic-subscriptions.png)

Nếu có subscription hiện `Pending confirmation`, hãy kiểm tra lại access policy của queue đó ở bước 3.

### 5. Tạo subnet group cho Valkey

Subnet group cho ElastiCache biết được đặt các node Valkey vào những subnet nào. Chúng ta dùng hai private subnet ở hai Availability Zone khác nhau để chạy Multi-AZ.

**1.** Mở **Amazon ElastiCache**, chọn **Subnet groups**, rồi chọn **Create subnet group**.

![subnet](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/05-01-subnet-groups-list.png)

**2.** Điền form như sau:

- **Name**: `ridenow-prod-redis-subnets`
- **Description**: `ridenow-prod Redis private subnets`
- **VPC ID**: `ridenow-prod-vpc`
- **Selected subnets**: chọn **Manage**, chỉ giữ `ridenow-prod-private-1` (`ap-southeast-1a`) và `ridenow-prod-private-2` (`ap-southeast-1b`)

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/05-02-subnet-group-form.png)

Console có thể tự chọn sẵn mọi subnet trong VPC. Hãy bỏ các subnet public nếu có, vì Valkey không được phép truy cập từ internet.

**3.** Chọn **Create**.

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/05-03-subnet-group-created.png)

### 6. Tạo security group cho Valkey

Security group này đóng vai trò tường lửa cho Valkey: chỉ mở cổng `6379` cho các tài nguyên mang security group của cluster, tức các pod EKS ở 5.8 và 5.9.

**1.** Mở **EC2**, chọn **Security Groups**, rồi chọn **Create security group**.

**2.** Điền phần **Basic details**:

- **Security group name**: `ridenow-prod-redis-sg`
- **Description**: `ridenow-prod Redis (private)`
- **VPC**: `ridenow-prod-vpc`

**3.** Trong **Inbound rules**, chọn **Add rule** và điền:

- **Type**: `Custom TCP`
- **Port range**: `6379`
- **Source**: `Custom`, rồi chọn `ridenow-prod-cluster-sg` (gõ `sg-` hoặc tên để tìm)
- **Description**: `Redis 6379 from EKS nodes only`

Giữ **Outbound rules** ở mặc định (`All traffic`, `0.0.0.0/0`).

![security](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/06-01-redis-sg-form_1.png)

**4.** Trong **Tags**, chọn **Add new tag**, đặt **Key** là `Name` và **Value** là `ridenow-prod-redis-sg`. Khối lệnh tra cứu biến tìm security group theo tag này.

![security](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/06-01-redis-sg-form_2.png)

Chúng ta dùng **một security group** làm source thay vì dải CIDR như `10.30.0.0/16`. Như vậy chỉ những tài nguyên mang `ridenow-prod-cluster-sg` mới kết nối được, chứ không phải mọi máy trong VPC.

**5.** Chọn **Create security group**.

![security](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/06-02-redis-sg-created.png)

### 7. Tạo Valkey replication group

> **Lưu ý:** từ bước này AWS bắt đầu tính tiền theo giờ cho hai node Valkey.

**1.** Mở **Amazon ElastiCache**, chọn **Valkey caches**, rồi chọn **Create Valkey cache**.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-01-valkey-caches-list.png)

**2.** Chọn kiểu triển khai:

- **Engine**: Chọn `Valkey`
- **Deployment option**: `Design your own cache` (không chọn `Serverless`)
- **Creation method**: `Cluster cache` (không chọn `Easy create`)
- **Cluster mode**: `Disabled`

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-02-valkey-deployment-options.png)

Chúng ta tắt cluster mode vì dữ liệu của RideNow nhỏ, một shard là đủ, và client không cần hỗ trợ chế độ cluster.

**3.** Điền phần **Cluster info** và **Location**:

- **Name**: `ridenow-prod-redis`
- **Description**: `ridenow-prod driver GEO / live state`
- **Location**: `AWS Cloud`
- **Multi-AZ**: `Enable`
- **Auto-failover**: kiểm tra đã được bật

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-03-valkey-cluster-info.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-03-valkey-cluster-settings-locations.png)

Multi-AZ bắt buộc phải có auto-failover, nên console tự bật ô **Auto-failover** và có thể khoá nó lại. Ô này chỉ bật được khi có ít nhất một replica. Nếu ô bị mờ và đang tắt, hãy đặt **Number of replicas** ở bước tiếp theo trước, rồi quay lại kiểm tra.

**4.** Điền phần **Cluster settings**:

- **Engine version**: `8.0`
- **Port**: `6379`
- **Parameter group**: `default.valkey8`
- **Node type**: `cache.t4g.small`
- **Number of replicas**: `1`

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-03-valkey-cluster-settings-cache.png)

Với một replica, chúng ta có hai node: primary ở một Availability Zone và replica ở AZ còn lại. Khi primary gặp sự cố, replica được tự động nâng lên làm primary.

**5.** Trong phần **Connectivity**, chọn **Choose existing subnet group**, chọn `ridenow-prod-redis-subnets`, và để **Availability Zone placements** là `No preference`.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-04-valkey-connectivity.png)

**6.** Điền phần **Security**:

- **Encryption at rest**: `Enable` (khoá mặc định)
- **encryption key**: `Default key`
- **Encryption in transit**: `Enable`
- **Access control**: `No access control`
- **Selected security groups**: chọn **Manage**, đánh dấu `ridenow-prod-redis-sg` và bỏ `default`

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-manage.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-choose.png)

Khi bật **Encryption in transit**, mọi client phải kết nối bằng TLS. Các service của RideNow đã được cấu hình sẵn cho việc này ở 5.9.

**7.** Trong phần **Backup**, bỏ đánh dấu **Enable automatic backups**. Dữ liệu trong Valkey là trạng thái tạm thời và được ghi lại liên tục, nên không cần backup. Phần **Maintenance** để mặc định.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-backup.png)

**8.** Trong phần **Tags**, chọn **Add new tag**, đặt **Key** là `Environment` và **Value** là `prod`.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-tag.png)

**9.** Xem lại trang tóm tắt, rồi chọn **Create**.

**10.** Chờ khoảng 10 đến 15 phút cho đến khi **Status** chuyển từ `Creating` sang `Available`.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-07-valkey-creating.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-08-valkey-available.png)

### 8. Kiểm tra luồng sự kiện

Trước khi chuyển sang phần tiếp theo, chúng ta publish thử hai sự kiện để chắc chắn filter policy hoạt động đúng.

**1.** Mở topic `ridenow-prod-events` trên **Amazon SNS** và chọn **Publish message**. Để **Subject** trống, đặt **Message structure** là `Identical payload for all delivery protocols`, rồi dán vào **Message body**:

```json
{"type":"trip.requested","tripId":"test-001"}
```

![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check.png)
![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check-message-body.png)

**2.** Chọn **Publish message**. Sau đó lặp lại bước 1 với một sự kiện thứ hai:

```json
{"type":"trip.completed","tripId":"test-001"}
```

![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check-message-body-2.png)

**3.** Mở **Amazon SQS**, chọn **Queues**, và xem cột **Messages available** (chọn nút refresh nếu số chưa cập nhật):

| Queue | Messages available | Lý do |
|---|---|---|
| `ridenow-prod-matching` | `1` | chỉ nhận `trip.requested` |
| `ridenow-prod-payment` | `1` | chỉ nhận `trip.completed` |
| `ridenow-prod-notifications` | `2` | nhận mọi sự kiện |
| Ba DLQ | `0` | chưa có message nào bị lỗi |

![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check-queue-message-available.png)

Kết quả này cho thấy mỗi queue chỉ nhận đúng loại sự kiện mà service của nó xử lý. Nếu `ridenow-prod-matching` hoặc `ridenow-prod-payment` có `0`, hãy kiểm tra **Filter policy scope** là `Message body`. Nếu cả ba queue đều có `0`, hãy kiểm tra access policy ở bước 3 và Encryption của topic ở bước 1.

**4.** Mở `ridenow-prod-matching`, chọn **Send and receive messages**, rồi chọn **Poll for messages**. Chọn message để xem body.

Body chính là JSON bạn đã gửi, không có lớp vỏ SNS (`"Type": "Notification"`, `"TopicArn"`), nhờ raw message delivery.

**5.** Xoá các message test: trên từng queue, chọn **Purge** và gõ `purge` để xác nhận. Mỗi lần poll làm tăng receive count của message, sau 5 lần message sẽ bị chuyển sang DLQ. Nếu không xoá, các service ở 5.9 sẽ đọc phải message `test-001`.

## Giá trị cần ghi lại

**1.** Bây giờ hãy chạy khối lệnh thứ hai của mục Thiết lập biến. Các biến dưới đây được tra cứu tự động theo tên tài nguyên, nên bạn không phải sao chép gì bằng tay:

| Biến | Giá trị | Được dùng bởi |
|---|---|---|
| `TRIP_EVENTS_TOPIC_ARN` | ARN của topic `ridenow-prod-events` | 5.9 (các service publish sự kiện lên đó) |
| `MATCHING_QUEUE_URL` | URL của `ridenow-prod-matching` | 5.9 (matching-service) |
| `MATCHING_QUEUE_ARN` | ARN của `ridenow-prod-matching` | Kiểm tra kết nối |
| `PAYMENT_QUEUE_URL` | URL của `ridenow-prod-payment` | 5.9 (payment-service) |
| `PAYMENT_QUEUE_ARN` | ARN của `ridenow-prod-payment` | Kiểm tra kết nối |
| `NOTIFICATIONS_QUEUE_URL` | URL của `ridenow-prod-notifications` | 5.9 (notification-worker) |
| `NOTIFICATIONS_QUEUE_ARN` | ARN của `ridenow-prod-notifications` | Kiểm tra kết nối |
| `REDIS_ENDPOINT` | Host primary endpoint của `ridenow-prod-redis` (bắt đầu bằng `master.`, không có cổng) | 5.9 (driver, matching, notification và realtime) |
| `REDIS_SG_ID` | ID của `ridenow-prod-redis-sg` | 5.9 (rule ingress từ security group của EKS cluster) |

## Các bước sau triển khai

Phần này không cần bước nào sau khi triển khai. Ở 5.9, chúng ta sẽ mở `ridenow-prod-redis-sg` cho các Fargate pod.

## Kiểm tra kết quả

**1.** Trong cùng shell, sau khi đã chạy khối thứ hai của mục Thiết lập biến, chạy các lệnh sau:

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

Kết quả mong đợi:

```text
TRIP_EVENTS_TOPIC_ARN=arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events
MATCHING_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-matching
PAYMENT_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-payment
NOTIFICATIONS_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-notifications
REDIS_ENDPOINT=master.ridenow-prod-redis.abc123.apse1.cache.amazonaws.com
REDIS_SG_ID=sg-0aaa...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-matching	arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:1111...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-payment	arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:2222...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-notifications	arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:3333...
available	enabled	enabled
```

Sáu biến đầu đều có giá trị (không có `MISSING`), ba subscription có ARN đầy đủ (không phải `PendingConfirmation`), và Valkey ở trạng thái `available` với Multi-AZ và auto-failover được bật. Nếu thiếu giá trị nào, hãy mở trang console của tài nguyên đó và kiểm tra lại tên.

### Nội dung tiếp theo

- [Computing](../5.8-Compute/)
