---
title : "SQS: Messaging"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

This page creates the messaging and cache layer of the `prod` environment: one SNS topic, three SQS queues with their dead-letter queues, three subscriptions, and a Valkey replication group. The services publish trip events to the topic, and each consumer reads its own queue.

## Set variables

Run this block in a new shell. It repeats the shared variables from 5.2, and it looks up the network resources from 5.4:

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

Expected result (the IDs are masked, and the subnet list shows two IDs):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
CLUSTER_SG_ID=sg-0eee...
```

If a line starts with `STOP`, fix that problem and run the block again.

The messaging variables look up resources that you create on this page. Run the next block after you finish Create the resources, not before. Each lookup finds its resource by name.

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

Expected result (the IDs are masked):

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

The three work queues have a visibility timeout of 60 seconds and move a message to their dead-letter queue after 5 receives. Each dead-letter queue keeps messages for 14 days. The matching subscription passes only `trip.requested` events. The payment subscription passes `trip.completed` and `trip.cancelled`. The notifications subscription passes every event. The Valkey group has one primary and one replica node of type `cache.t4g.small`, with Multi-AZ and automatic failover.

## Depends on

5.4 Network. You need these resources before you start:

| Resource | Name | Used for |
|---|---|---|
| VPC | `ridenow-prod-vpc` | Cache security group |
| Private subnets | `ridenow-prod-private-1`, `ridenow-prod-private-2` | Cache subnet group |
| Security group | `ridenow-prod-cluster-sg` | Source of the port 6379 rule |

## Create the resources

Open the AWS console in the Asia Pacific (Singapore) `ap-southeast-1` region. Follow the steps in order.

### 1. SNS topic (`TripEventTopic`)

1. Go to **Amazon SNS -> Topics -> Create topic**.
2. Set **Type** to `Standard`.
3. Set **Name** to `ridenow-prod-events`.
4. Leave **Encryption** disabled (the `aws/sns` key blocks delivery to SQS).
5. Leave **Access policy** at the default.
6. Choose **Create topic**.

### 2. Dead-letter queues (`NotificationDlq`, `MatchingDlq`, `PaymentDlq`)

Repeat this list three times, with the names `ridenow-prod-notifications-dlq`, `ridenow-prod-matching-dlq` and `ridenow-prod-payment-dlq`.

1. Go to **Amazon SQS -> Queues -> Create queue**.
2. Set **Type** to `Standard`.
3. Set **Name** to one of the three DLQ names.
4. Set **Visibility timeout** to `30` **Seconds** (the default).
5. Set **Message retention period** to `14` **Days**.
6. Set **Server-side encryption** to `Enabled`.
7. Set **Encryption key type** to `Amazon SQS key (SSE-SQS)`.
8. Leave **Access policy** at `Basic` (the default).
9. Set **Dead-letter queue** to `Disabled`.
10. Choose **Create queue**.

### 3. Work queues and access policies (`NotificationQueue`, `MatchingQueue`, `PaymentQueue`)

Print the three access policies. They contain your account ID and Region, so you do not type them by hand:

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

Repeat the list below three times, with these queue and dead-letter queue pairs:

| Queue name | Dead-letter queue |
|---|---|
| `ridenow-prod-matching` | `ridenow-prod-matching-dlq` |
| `ridenow-prod-payment` | `ridenow-prod-payment-dlq` |
| `ridenow-prod-notifications` | `ridenow-prod-notifications-dlq` |

1. Go to **Amazon SQS -> Queues -> Create queue**.
2. Set **Type** to `Standard`.
3. Set **Name** to the queue name from the table.
4. Set **Visibility timeout** to `1` **Minutes** (60 seconds).
5. Set **Message retention period** to `4` **Days** (the default).
6. Set **Server-side encryption** to `Enabled`.
7. Set **Encryption key type** to `Amazon SQS key (SSE-SQS)`.
8. Under **Access policy**, select **Advanced**.
9. Replace the JSON with the policy that the block above printed for this queue.
10. Set **Dead-letter queue** to `Enabled`.
11. In **Choose queue**, pick the dead-letter queue from the table.
12. Set **Maximum receives** to `5`.
13. Choose **Create queue**.

### 4. Subscriptions (`MatchingSubscription`, `PaymentSubscription`, `NotificationSubscription`)

Print the three queue ARNs. You paste one as the **Endpoint** of each subscription:

```bash
for Q in matching payment notifications; do
  echo "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:${CLUSTER_NAME}-${Q}"
done
```

Matching subscription:

1. Go to **Amazon SNS -> Topics -> `ridenow-prod-events` -> Subscriptions** tab -> **Create subscription**.
2. Set **Protocol** to `Amazon SQS`.
3. Set **Endpoint** to the ARN printed for `ridenow-prod-matching`.
4. Check **Enable raw message delivery**.
5. Expand **Subscription filter policy** and turn it on.
6. Set **Filter policy scope** to `Message body`.
7. Paste the filter below into the JSON editor.
8. Leave **Redrive policy (dead-letter queue)** off.
9. Choose **Create subscription**.

```json
{
  "type": ["trip.requested"]
}
```

Payment subscription:

1. Go to **Amazon SNS -> Topics -> `ridenow-prod-events` -> Subscriptions** tab -> **Create subscription**.
2. Set **Protocol** to `Amazon SQS`.
3. Set **Endpoint** to the ARN printed for `ridenow-prod-payment`.
4. Check **Enable raw message delivery**.
5. Expand **Subscription filter policy** and turn it on.
6. Set **Filter policy scope** to `Message body`.
7. Paste the filter below into the JSON editor.
8. Leave **Redrive policy (dead-letter queue)** off.
9. Choose **Create subscription**.

```json
{
  "type": ["trip.completed", "trip.cancelled"]
}
```

Notification subscription:

1. Go to **Amazon SNS -> Topics -> `ridenow-prod-events` -> Subscriptions** tab -> **Create subscription**.
2. Set **Protocol** to `Amazon SQS`.
3. Set **Endpoint** to the ARN printed for `ridenow-prod-notifications`.
4. Check **Enable raw message delivery**.
5. Leave **Subscription filter policy** off (it receives every event).
6. Leave **Redrive policy (dead-letter queue)** off.
7. Choose **Create subscription**.

Each subscription shows Status `Confirmed`. If one shows `Pending confirmation`, recheck the queue access policy from step 3.

### 5. Cache subnet group (`RedisSubnetGroup`)

1. Go to **Amazon ElastiCache -> Subnet groups -> Create subnet group**.
2. Set **Name** to `ridenow-prod-redis-subnets`.
3. Set **Description** to `ridenow-prod Redis private subnets`.
4. Set **VPC ID** to `ridenow-prod-vpc`.
5. Under **Selected subnets**, pick the Availability Zones first, then tick `ridenow-prod-private-1` and `ridenow-prod-private-2`.
6. Choose **Create**.

### 6. Cache security group (`RedisSecurityGroup`)

1. Go to **EC2 -> Security Groups -> Create security group**.
2. Set **Security group name** to `ridenow-prod-redis-sg`.
3. Set **Description** to `ridenow-prod Redis (private)`.
4. Set **VPC** to `ridenow-prod-vpc`.
5. Under **Inbound rules**, choose **Add rule**.
6. Set **Type** to `Custom TCP`.
7. Set **Port range** to `6379`.
8. Set **Source** to `Custom`, then pick the security group `ridenow-prod-cluster-sg`.
9. Set **Description** to `Redis 6379 from EKS nodes only`.
10. Leave **Outbound rules** at the default (`All traffic`, `0.0.0.0/0`).
11. Under **Tags**, choose **Add new tag**. Set **Key** to `Name` and **Value** to `ridenow-prod-redis-sg`. The variable lookup finds the group by this tag.
12. Choose **Create security group**.

### 7. Valkey replication group (`RedisReplicationGroup`)

1. Go to **Amazon ElastiCache -> Valkey caches -> Create Valkey cache**.
2. Set **Deployment option** to `Design your own cache`.
3. Set **Creation method** to `Cluster cache`.
4. Set **Cluster mode** to `Disabled`.
5. Set **Name** to `ridenow-prod-redis`.
6. Set **Description** to `ridenow-prod driver GEO / live state`.
7. Set **Engine version** to `8.0`.
8. Set **Port** to `6379`.
9. Set **Parameter group** to `default.valkey8`.
10. Set **Node type** to `cache.t4g.small`.
11. Set **Number of replicas** to `1`.
12. Set **Multi-AZ** to `Enable`.
13. Set **Auto-failover** to `Enable`.
14. Under **Connectivity**, set **Subnet group** to **Choose existing subnet group**, then pick `ridenow-prod-redis-subnets`.
15. Set **Availability Zone placements** to `No preference`.
16. Under **Security**, set **Encryption at rest** to `Enable` (default key).
17. Set **Encryption in transit** to `Enable`.
18. Set **Access control** to `No access control`.
19. Under **Selected security groups**, choose **Manage**.
20. Tick `ridenow-prod-redis-sg` and remove `default`.
21. Under **Backup**, clear **Enable automatic backups**.
22. Choose **Create**.
23. Wait 10 to 15 minutes until **Status** is `Available`.

## Values to record

Run the second block of Set variables now. The variables below are looked up for you, so you copy nothing by hand.

| Variable | What it holds | Used by |
|---|---|---|
| `TRIP_EVENTS_TOPIC_ARN` | ARN of the topic `ridenow-prod-events` | 5.9 (service workloads publish events to it) |
| `MATCHING_QUEUE_URL` | URL of `ridenow-prod-matching` | 5.9 (matching-service workload) |
| `MATCHING_QUEUE_ARN` | ARN of `ridenow-prod-matching` | Connection checks |
| `PAYMENT_QUEUE_URL` | URL of `ridenow-prod-payment` | 5.9 (payment-service workload) |
| `PAYMENT_QUEUE_ARN` | ARN of `ridenow-prod-payment` | Connection checks |
| `NOTIFICATIONS_QUEUE_URL` | URL of `ridenow-prod-notifications` | 5.9 (notification-worker workload) |
| `NOTIFICATIONS_QUEUE_ARN` | ARN of `ridenow-prod-notifications` | Connection checks |
| `REDIS_ENDPOINT` | Primary endpoint host of `ridenow-prod-redis` (no port) | 5.9 (driver, matching, notification and realtime workloads) |
| `REDIS_SG_ID` | ID of `ridenow-prod-redis-sg` | 5.9 (ingress rule from the EKS cluster security group) |

## Post-deploy steps

No post-deploy step is needed for this page. In 5.9 you open `ridenow-prod-redis-sg` to the Fargate pods.

## Verify

Run these commands in the same shell, after the second block of Set variables:

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

Expected result: six variables with a value (none `MISSING`), three subscriptions with full ARNs (not `PendingConfirmation`), and a cache group that is available with Multi-AZ and automatic failover enabled.

```text
TRIP_EVENTS_TOPIC_ARN=arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events
MATCHING_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-matching
PAYMENT_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-payment
NOTIFICATIONS_QUEUE_URL=https://sqs.ap-southeast-1.amazonaws.com/123456789012/ridenow-prod-notifications
REDIS_ENDPOINT=ridenow-prod-redis.abc123.apse1.cache.amazonaws.com
REDIS_SG_ID=sg-0aaa...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-matchingarn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:1111...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-paymentarn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:2222...
arn:aws:sqs:ap-southeast-1:123456789012:ridenow-prod-notificationsarn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events:3333...
availableenabledenabled
```

If a value is missing, open the console page for that resource and check the name.
