---
title : "Messaging & Cache"
date : 2024-01-01
weight : 7
chapter : false
pre : " <b> 5.7. </b> "
---

In this section, we build the **messaging** and **cache** layers of the RideNow `prod` environment. This is where the services exchange trip events, and where the live location of each driver is stored.

## Overview

When a trip changes state, `trip-service` does not call every other service directly. Instead, it publishes an event to the **Amazon SNS topic** `ridenow-prod-events`. SNS copies the event to several **Amazon SQS queues**, and each service reads its own queue:

| Event (`type`) | `ridenow-prod-matching` | `ridenow-prod-payment` | `ridenow-prod-notifications` |
|---|---|---|---|
| `trip.requested` | ✅ matching-service finds a driver | | ✅ |
| `trip.completed` | | ✅ payment-service handles the payment | ✅ |
| `trip.cancelled` | | ✅ | ✅ |

This design has three benefits:

- **Decoupled services**: `trip-service` does not need to know how many services are listening. To add a new consumer, we only add a subscription.
- **No lost events**: if a service stops for a short time, the events wait in the queue until the service runs again.
- **Isolated failures**: a message that fails 5 times is moved to a **dead-letter queue (DLQ)**, so it does not block the other messages, and we have 14 days to investigate it.

We also create an **ElastiCache for Valkey** cluster (it is compatible with the Redis protocol and about 20% cheaper than Redis OSS). Valkey stores the GEO position and live state of the drivers, the location Pub/Sub channel, and the signal that a trip has ended.

> **Cost:** SNS and SQS are almost free when there is no traffic. Valkey (two `cache.t4g.small` nodes) is billed per hour from the moment it is created. Delete it with [5.11](../5.11-Cleanup/) when you no longer need it.

## Set variables

**1.** Open a new Bash shell and run the block below. It sets the shared variables from [5.2](../5.2-Prerequiste/) again, and looks up the network resources created in [5.4](../5.4-Network/):

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

Expected output (the IDs are masked, and the subnet list has two IDs):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
CLUSTER_SG_ID=sg-0eee...
```

We set the variables again in every section, because you may work on each section in a different shell. If a line starts with `STOP`, fix that problem and run the block again. A `STOP` line about `VPC_ID`, `PRIVATE_SUBNET_IDS`, or `CLUSTER_SG_ID` means you need to finish 5.4 first.

**2.** The second block looks up the resources that we create in this section. **Do not run it yet.** We come back to it in the Values to record section, after all the resources are created.

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

Expected output (the IDs are masked):

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

## What it creates

After this section, your account has the following 16 resources:

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

## Depends on

This section needs three resources from [5.4 Network](../5.4-Network/):

| Resource | Name | Used for |
|---|---|---|
| VPC | `ridenow-prod-vpc` | The subnet group and the security group of Valkey |
| Private subnet | `ridenow-prod-private-1` (`ap-southeast-1a`), `ridenow-prod-private-2` (`ap-southeast-1b`) | The subnet group of Valkey |
| Security group | `ridenow-prod-cluster-sg` | The source that is allowed to connect to Valkey |

SNS and SQS are not inside the VPC, so you can do steps 1 to 4 before 5.4 is finished. From step 5 (Valkey), all three resources must exist.

## Create the resources

Open the AWS console and choose the region **Asia Pacific (Singapore)** `ap-southeast-1` in the top-right corner. Follow the steps in order.

### 1. Create the SNS topic

**1.** Open **Amazon SNS**, choose **Topics** in the left menu, and then choose **Create topic**.

![SNS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/01-01-sns-topics-list.png)

**2.** Fill in the form as follows:

- **Type**: `Standard`
- **Name**: `ridenow-prod-events`
- **Encryption**: leave it **off**
- **Access policy**: keep the default

![SNS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/01-02-sns-create-form_1.png)

We choose `Standard` because the consumers do not need strict ordering, and they need high throughput. Encryption must stay off, because the default key `aws/sns` does not allow SNS to send messages to SQS. If you turn it on, the queues receive nothing, and the console shows no error.

**3.** Choose **Create topic**. The detail page shows the **ARN**, which looks like `arn:aws:sns:ap-southeast-1:123456789012:ridenow-prod-events`.

![SNS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/01-03-sns-topic-created.png)

### 2. Create three dead-letter queues

We create the DLQs first, because in step 3 each work queue must pick its DLQ from an existing list.

**1.** Open **Amazon SQS**, choose **Queues**, and then choose **Create queue**.

![SQS](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-01-sqs-queues-list.png)

**2.** Fill in **Details** and **Configuration**:

- **Type**: `Standard`
- **Name**: `ridenow-prod-notifications-dlq`
- **Visibility timeout**: `30` **Seconds** (default)
- **Message retention period**: `14` **Days**

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-02-dlq-notifications.png)

By default, SQS keeps messages for only 4 days. For a DLQ, we increase this to the maximum of 14 days, so we have enough time to investigate the failed messages before they are deleted.

> **Note:** the queue name and the **Type** cannot be changed after you create the queue. If you type the name wrong, you must delete the queue and create it again. Check the name carefully before you choose **Create queue**.

**3.** In **Encryption**, set **Server-side encryption** to `Enabled` and **Encryption key type** to `Amazon SQS key (SSE-SQS)`.

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-03-dlq-encryption.png)

SSE-SQS encrypts the messages at rest with a key that SQS manages. It costs nothing extra and needs no KMS permissions.

**4.** Keep **Access policy** as `Basic`, set **Dead-letter queue** to `Disabled`, and then choose **Create queue**. A DLQ does not need another DLQ.

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-04-dlq-created_1.png)
![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-04-dlq-created_2.png)

**5.** Repeat steps 1 to 4 two more times, with the names `ridenow-prod-matching-dlq` and `ridenow-prod-payment-dlq`. The **Queues** list now shows three DLQs:

![DLQ](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/02-05-dlq-list.png)

### 3. Create three work queues

Work queues are where the services read the events. Each queue needs an **access policy** that allows only the topic `ridenow-prod-events` to send messages to it.

**1.** In the console, choose **Create queue** and fill in **Details** and **Configuration**:

- **Type**: `Standard`
- **Name**: `ridenow-prod-matching`
- **Visibility timeout**: `1` **Minutes**
- **Message retention period**: `4` **Days** (default)

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-01-queue-configuration.png)

A visibility timeout of 60 seconds is the time that a message stays hidden from the other consumers while one consumer processes it. If the consumer does not finish in 60 seconds, the message becomes visible again and SQS tries it again.

**2.** In **Encryption**, set **Server-side encryption** to `Enabled` and **Encryption key type** to `Amazon SQS key (SSE-SQS)`.

**3.** In **Access policy**, choose **Advanced**. Delete all the existing JSON, and then paste the policy that was printed for `ridenow-prod-matching`.

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

> **Note:** the `Resource` line in the policy must end with the exact name of the queue that you are creating. If you paste the policy of another queue, the console still creates the queue and shows no error, but SNS cannot send messages to this queue.

**4.** In **Dead-letter queue**, set it to `Enabled`, choose `ridenow-prod-matching-dlq` in **Choose queue**, and set **Maximum receives** to `5`.

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-03-queue-dlq-setting.png)

With this setting, if a message is received 5 times and is still not deleted (that is, it failed 5 times), SQS automatically moves it to `ridenow-prod-matching-dlq`.

**5.** Choose **Create queue**.

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-04-queue-created.png)

**6.** Repeat step 2 to step 6 for the other two queues. Each queue uses its own policy and its own DLQ:

| Queue name | Dead-letter queue |
|---|---|
| `ridenow-prod-payment` | `ridenow-prod-payment-dlq` |
| `ridenow-prod-notifications` | `ridenow-prod-notifications-dlq` |

The **Queues** list now shows all six queues:

![queue](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/03-05-queue-list.png)

### 4. Create three subscriptions

A subscription connects the topic to a queue. The **filter policy** of a subscription decides which queue receives which type of event.

**1.** Run the command below to print the ARNs of the three queues. You paste them into the **Endpoint** field:

```bash
for Q in matching payment notifications; do
  echo "arn:aws:sqs:${AWS_REGION}:${ACCOUNT_ID}:${CLUSTER_NAME}-${Q}"
done
```

**2.** Open **Amazon SNS**, choose **Topics**, choose `ridenow-prod-events`, open the **Subscriptions** tab, and then choose **Create subscription**.

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-01-topic-subscription-click-create.png)

**3.** Fill in the form for the **matching** subscription:

- **Topic ARN**: already filled in with the ARN of `ridenow-prod-events`
- **Protocol**: `Amazon SQS`
- **Endpoint**: the ARN that was printed for `ridenow-prod-matching`
- **Enable raw message delivery**: select it
- **Subscription filter policy**: turn it on, set **Filter policy scope** to `Message body`, and then paste the filter:

```json
{
  "type": ["trip.requested"]
}
```

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-01-topic-subscriptions-settings.png)
![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-02-matching-sub-form.png)

**Raw message delivery** makes the message body in SQS exactly the JSON of the event, without the SNS envelope, so the service can read it directly. **Filter policy scope** must be `Message body`, because the filter compares the `type` field in the body. If you keep the default `Message attributes`, the filter looks in the attributes (which the publisher does not send), and the queue never receives any message.

**4.** Keep **Redrive policy (dead-letter queue)** off, and then choose **Create subscription**. The **Status** of the subscription must be `Confirmed`.

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-03-matching-sub-created.png)

**5.** Repeat step 2 to step 4 for the **payment** subscription. Set **Endpoint** to the ARN of `ridenow-prod-payment`, and use the filter below (the scope is still `Message body`):

```json
{
  "type": ["trip.completed", "trip.cancelled"]
}
```

**6.** Repeat step 2 to step 4 for the **notification** subscription. Set **Endpoint** to the ARN of `ridenow-prod-notifications`, select **Enable raw message delivery** again, but **keep Subscription filter policy off**. This queue must receive every event, so `notification-worker` can send notifications to the users.

The **Subscriptions** tab now shows three rows, and all of them have the status `Confirmed`:

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/04-04-topic-subscriptions.png)

If a subscription shows `Pending confirmation`, check the access policy of that queue in step 3.

### 5. Create the subnet group for Valkey

A subnet group tells ElastiCache which subnets it can use for the Valkey nodes. We use two private subnets in two different Availability Zones, so we can run Multi-AZ.

**1.** Open **Amazon ElastiCache**, choose **Subnet groups**, and then choose **Create subnet group**.

![subnet](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/05-01-subnet-groups-list.png)

**2.** Fill in the form as follows:

- **Name**: `ridenow-prod-redis-subnets`
- **Description**: `ridenow-prod Redis private subnets`
- **VPC ID**: `ridenow-prod-vpc`
- **Selected subnets**: choose **Manage**, and keep only `ridenow-prod-private-1` (`ap-southeast-1a`) and `ridenow-prod-private-2` (`ap-southeast-1b`)

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/05-02-subnet-group-form.png)

The console may select every subnet in the VPC for you. Remove the public subnets if there are any, because Valkey must not be reachable from the internet.

**3.** Choose **Create**.

![topic](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/05-03-subnet-group-created.png)

### 6. Create the security group for Valkey

This security group works as a firewall for Valkey. It opens port `6379` only to resources that have the cluster security group, which means the EKS pods in 5.8 and 5.9.

**1.** Open **EC2**, choose **Security Groups**, and then choose **Create security group**.

**2.** Fill in **Basic details**:

- **Security group name**: `ridenow-prod-redis-sg`
- **Description**: `ridenow-prod Redis (private)`
- **VPC**: `ridenow-prod-vpc`

**3.** In **Inbound rules**, choose **Add rule** and fill in:

- **Type**: `Custom TCP`
- **Port range**: `6379`
- **Source**: `Custom`, and then choose `ridenow-prod-cluster-sg` (type `sg-` or the name to search)
- **Description**: `Redis 6379 from EKS nodes only`

Keep **Outbound rules** at the default (`All traffic`, `0.0.0.0/0`).

![security](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/06-01-redis-sg-form_1.png)

**4.** In **Tags**, choose **Add new tag**, and set **Key** to `Name` and **Value** to `ridenow-prod-redis-sg`. The lookup block for the variables finds the security group by this tag.

![security](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/06-01-redis-sg-form_2.png)

We use **a security group** as the source instead of a CIDR range such as `10.30.0.0/16`. In this way, only resources that have `ridenow-prod-cluster-sg` can connect, and not every machine in the VPC.

**5.** Choose **Create security group**.

![security](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/06-02-redis-sg-created.png)

### 7. Create the Valkey replication group

> **Note:** from this step, AWS charges per hour for the two Valkey nodes.

**1.** Open **Amazon ElastiCache**, choose **Valkey caches**, and then choose **Create Valkey cache**.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-01-valkey-caches-list.png)

**2.** Choose the deployment type:

- **Engine**: choose `Valkey`
- **Deployment option**: `Design your own cache` (do not choose `Serverless`)
- **Creation method**: `Cluster cache` (do not choose `Easy create`)
- **Cluster mode**: `Disabled`

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-02-valkey-deployment-options.png)

We turn cluster mode off because the RideNow data is small, one shard is enough, and the clients do not need to support cluster mode.

**3.** Fill in **Cluster info** and **Location**:

- **Name**: `ridenow-prod-redis`
- **Description**: `ridenow-prod driver GEO / live state`
- **Location**: `AWS Cloud`
- **Multi-AZ**: `Enable`
- **Auto-failover**: check that it is turned on

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-03-valkey-cluster-info.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-03-valkey-cluster-settings-locations.png)

Multi-AZ requires auto-failover, so the console turns on **Auto-failover** for you and may lock it. This option can be turned on only when there is at least one replica. If the option is grey and turned off, set **Number of replicas** in the next step first, and then come back to check it.

**4.** Fill in **Cluster settings**:

- **Engine version**: `8.0`
- **Port**: `6379`
- **Parameter group**: `default.valkey8`
- **Node type**: `cache.t4g.small`
- **Number of replicas**: `1`

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-03-valkey-cluster-settings-cache.png)

With one replica, we have two nodes: the primary in one Availability Zone and the replica in the other one. If the primary fails, the replica is promoted to primary automatically.

**5.** In **Connectivity**, choose **Choose existing subnet group**, choose `ridenow-prod-redis-subnets`, and keep **Availability Zone placements** at `No preference`.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-04-valkey-connectivity.png)

**6.** Fill in **Security**:

- **Encryption at rest**: `Enable` (default key)
- **Encryption key**: `Default key`
- **Encryption in transit**: `Enable`
- **Access control**: `No access control`
- **Selected security groups**: choose **Manage**, select `ridenow-prod-redis-sg`, and clear `default`

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-manage.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-choose.png)

When you turn on **Encryption in transit**, every client must connect with TLS. The RideNow services are already configured for this in 5.9.

**7.** In **Backup**, clear **Enable automatic backups**. The data in Valkey is temporary state that is written again all the time, so we do not need a backup. Keep the **Maintenance** section at the default.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-backup.png)

**8.** In **Tags**, choose **Add new tag**, and set **Key** to `Environment` and **Value** to `prod`.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-05-valkey-security-tag.png)

**9.** Review the summary page, and then choose **Create**.

**10.** Wait about 10 to 15 minutes until **Status** changes from `Creating` to `Available`.

![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-07-valkey-creating.png)
![valkey](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/07-08-valkey-available.png)

### 8. Check the event flow

Before we move to the next section, we publish two test events to make sure that the filter policies work correctly.

**1.** Open the topic `ridenow-prod-events` in **Amazon SNS** and choose **Publish message**. Keep **Subject** empty, set **Message structure** to `Identical payload for all delivery protocols`, and then paste this into **Message body**:

```json
{"type":"trip.requested","tripId":"test-001"}
```

![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check.png)
![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check-message-body.png)

**2.** Choose **Publish message**. Then repeat step 1 with a second event:

```json
{"type":"trip.completed","tripId":"test-001"}
```

![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check-message-body-2.png)

**3.** Open **Amazon SQS**, choose **Queues**, and look at the **Messages available** column (choose the refresh button if the numbers do not update):

| Queue | Messages available | Reason |
|---|---|---|
| `ridenow-prod-matching` | `1` | it receives only `trip.requested` |
| `ridenow-prod-payment` | `1` | it receives only `trip.completed` |
| `ridenow-prod-notifications` | `2` | it receives every event |
| Three DLQs | `0` | no message has failed yet |

![check](/fcj-workshop-ridenow/images/5-Workshop/5.7-Messaging/check-queue-message-available.png)

This result shows that each queue receives only the type of event that its service handles. If `ridenow-prod-matching` or `ridenow-prod-payment` shows `0`, check that **Filter policy scope** is `Message body`. If all three queues show `0`, check the access policy in step 3 and the Encryption of the topic in step 1.

**4.** Open `ridenow-prod-matching`, choose **Send and receive messages**, and then choose **Poll for messages**. Choose the message to see its body.

The body is exactly the JSON that you sent, with no SNS envelope (`"Type": "Notification"`, `"TopicArn"`), thanks to raw message delivery.

**5.** Delete the test messages: on each queue, choose **Purge** and type `purge` to confirm. Every poll increases the receive count of a message, and after 5 times the message is moved to the DLQ. If you do not delete the messages, the services in 5.9 will read the `test-001` messages.

## Values to record

**1.** Now run the second block of the Set variables section. The variables below are looked up automatically by resource name, so you do not need to copy anything by hand:

| Variable | Value | Used by |
|---|---|---|
| `TRIP_EVENTS_TOPIC_ARN` | The ARN of the topic `ridenow-prod-events` | 5.9 (the services publish events to it) |
| `MATCHING_QUEUE_URL` | The URL of `ridenow-prod-matching` | 5.9 (matching-service) |
| `MATCHING_QUEUE_ARN` | The ARN of `ridenow-prod-matching` | Connection checks |
| `PAYMENT_QUEUE_URL` | The URL of `ridenow-prod-payment` | 5.9 (payment-service) |
| `PAYMENT_QUEUE_ARN` | The ARN of `ridenow-prod-payment` | Connection checks |
| `NOTIFICATIONS_QUEUE_URL` | The URL of `ridenow-prod-notifications` | 5.9 (notification-worker) |
| `NOTIFICATIONS_QUEUE_ARN` | The ARN of `ridenow-prod-notifications` | Connection checks |
| `REDIS_ENDPOINT` | The host of the primary endpoint of `ridenow-prod-redis` (it starts with `master.`, without the port) | 5.9 (driver, matching, notification and realtime) |
| `REDIS_SG_ID` | The ID of `ridenow-prod-redis-sg` | 5.9 (the ingress rule from the security group of the EKS cluster) |

## Post-deploy steps

This section needs no post-deploy steps. In 5.9, we open `ridenow-prod-redis-sg` to the Fargate pods.

## Verify

**1.** In the same shell, after you run the second block of the Set variables section, run these commands:

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

Expected output:

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

The first six variables all have a value (no `MISSING`), the three subscriptions have full ARNs (not `PendingConfirmation`), and Valkey is `available` with Multi-AZ and auto-failover turned on. If a value is missing, open the console page of that resource and check the name again.

### What's next

- [Computing](../5.8-Compute/)
