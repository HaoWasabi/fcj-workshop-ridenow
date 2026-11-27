---
title : "Data"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

This page creates the data layer of the `prod` environment: one KMS key, one DB subnet group, and six encrypted PostgreSQL instances on Amazon RDS, one per service. RDS creates and stores each master password in AWS Secrets Manager, so you never type or save a password. Each service owns its own database and no database references another.

## Set variables

Run the first block in a new shell. It repeats the shared variables from 5.2 and the network variables from 5.4, each with its own lookup:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)
[ -n "$VPC_ID" ] && [ "$VPC_ID" != "None" ] || echo "STOP: VPC_ID is empty. Create the VPC first (5.4)."

PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-private-*" --query 'Subnets[].SubnetId' --output text)
[ -n "$PRIVATE_SUBNET_IDS" ] || echo "STOP: PRIVATE_SUBNET_IDS is empty. Create the private subnets first (5.4)."

DB_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-db-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$DB_SG_ID" ] && [ "$DB_SG_ID" != "None" ] || echo "STOP: DB_SG_ID is empty. Create the database security group first (5.4)."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "VPC_ID=$VPC_ID"
echo "PRIVATE_SUBNET_IDS=$PRIVATE_SUBNET_IDS"
echo "DB_SG_ID=$DB_SG_ID"
```

Expected result (the account ID and resource IDs are masked):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
DB_SG_ID=sg-0fff...
```

If a line starts with `STOP`, fix that problem and run the block again.

The data variables look up resources that you create on this page. Run the second block after you finish Create the resources, not before. Each lookup finds its resource by name.

```bash
RDS_KMS_KEY_ARN=$(aws kms describe-key --key-id "alias/${CLUSTER_NAME}-rds" --query KeyMetadata.Arn --output text 2>/dev/null)
[ -n "$RDS_KMS_KEY_ARN" ] && [ "$RDS_KMS_KEY_ARN" != "None" ] || echo "STOP: RDS_KMS_KEY_ARN is empty. Create the KMS key and alias first."

USER_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-user-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
[ -n "$USER_DB_ENDPOINT" ] && [ "$USER_DB_ENDPOINT" != "None" ] || echo "STOP: USER_DB_ENDPOINT is empty. Create ridenow-prod-user-db first and wait until it is Available."

USER_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-user-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$USER_DB_SECRET_ARN" ] && [ "$USER_DB_SECRET_ARN" != "None" ] || echo "STOP: USER_DB_SECRET_ARN is empty. Create ridenow-prod-user-db with Managed in AWS Secrets Manager first."

DRIVER_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-driver-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
[ -n "$DRIVER_DB_ENDPOINT" ] && [ "$DRIVER_DB_ENDPOINT" != "None" ] || echo "STOP: DRIVER_DB_ENDPOINT is empty. Create ridenow-prod-driver-db first and wait until it is Available."

DRIVER_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-driver-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$DRIVER_DB_SECRET_ARN" ] && [ "$DRIVER_DB_SECRET_ARN" != "None" ] || echo "STOP: DRIVER_DB_SECRET_ARN is empty. Create ridenow-prod-driver-db with Managed in AWS Secrets Manager first."

TRIP_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-trip-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
[ -n "$TRIP_DB_ENDPOINT" ] && [ "$TRIP_DB_ENDPOINT" != "None" ] || echo "STOP: TRIP_DB_ENDPOINT is empty. Create ridenow-prod-trip-db first and wait until it is Available."

TRIP_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-trip-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$TRIP_DB_SECRET_ARN" ] && [ "$TRIP_DB_SECRET_ARN" != "None" ] || echo "STOP: TRIP_DB_SECRET_ARN is empty. Create ridenow-prod-trip-db with Managed in AWS Secrets Manager first."

PAYMENT_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-payment-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
[ -n "$PAYMENT_DB_ENDPOINT" ] && [ "$PAYMENT_DB_ENDPOINT" != "None" ] || echo "STOP: PAYMENT_DB_ENDPOINT is empty. Create ridenow-prod-payment-db first and wait until it is Available."

PAYMENT_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-payment-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$PAYMENT_DB_SECRET_ARN" ] && [ "$PAYMENT_DB_SECRET_ARN" != "None" ] || echo "STOP: PAYMENT_DB_SECRET_ARN is empty. Create ridenow-prod-payment-db with Managed in AWS Secrets Manager first."

MATCHING_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-matching-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
[ -n "$MATCHING_DB_ENDPOINT" ] && [ "$MATCHING_DB_ENDPOINT" != "None" ] || echo "STOP: MATCHING_DB_ENDPOINT is empty. Create ridenow-prod-matching-db first and wait until it is Available."

MATCHING_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-matching-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$MATCHING_DB_SECRET_ARN" ] && [ "$MATCHING_DB_SECRET_ARN" != "None" ] || echo "STOP: MATCHING_DB_SECRET_ARN is empty. Create ridenow-prod-matching-db with Managed in AWS Secrets Manager first."

NOTIFICATION_DB_ENDPOINT=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-notification-db" --query 'DBInstances[0].Endpoint.Address' --output text 2>/dev/null)
[ -n "$NOTIFICATION_DB_ENDPOINT" ] && [ "$NOTIFICATION_DB_ENDPOINT" != "None" ] || echo "STOP: NOTIFICATION_DB_ENDPOINT is empty. Create ridenow-prod-notification-db first and wait until it is Available."

NOTIFICATION_DB_SECRET_ARN=$(aws rds describe-db-instances --db-instance-identifier "${CLUSTER_NAME}-notification-db" --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text 2>/dev/null)
[ -n "$NOTIFICATION_DB_SECRET_ARN" ] && [ "$NOTIFICATION_DB_SECRET_ARN" != "None" ] || echo "STOP: NOTIFICATION_DB_SECRET_ARN is empty. Create ridenow-prod-notification-db with Managed in AWS Secrets Manager first."

echo "RDS_KMS_KEY_ARN=$RDS_KMS_KEY_ARN"
echo "USER_DB_ENDPOINT=$USER_DB_ENDPOINT"
echo "USER_DB_SECRET_ARN=$USER_DB_SECRET_ARN"
echo "DRIVER_DB_ENDPOINT=$DRIVER_DB_ENDPOINT"
echo "DRIVER_DB_SECRET_ARN=$DRIVER_DB_SECRET_ARN"
echo "TRIP_DB_ENDPOINT=$TRIP_DB_ENDPOINT"
echo "TRIP_DB_SECRET_ARN=$TRIP_DB_SECRET_ARN"
echo "PAYMENT_DB_ENDPOINT=$PAYMENT_DB_ENDPOINT"
echo "PAYMENT_DB_SECRET_ARN=$PAYMENT_DB_SECRET_ARN"
echo "MATCHING_DB_ENDPOINT=$MATCHING_DB_ENDPOINT"
echo "MATCHING_DB_SECRET_ARN=$MATCHING_DB_SECRET_ARN"
echo "NOTIFICATION_DB_ENDPOINT=$NOTIFICATION_DB_ENDPOINT"
echo "NOTIFICATION_DB_SECRET_ARN=$NOTIFICATION_DB_SECRET_ARN"
```

Expected result (the account ID is masked and the values are shortened):

```text
RDS_KMS_KEY_ARN=arn:aws:kms:ap-southeast-1:123456789012:key/1234abcd-...
USER_DB_ENDPOINT=ridenow-prod-user-db.xxxx.ap-southeast-1.rds.amazonaws.com
USER_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
DRIVER_DB_ENDPOINT=ridenow-prod-driver-db.xxxx.ap-southeast-1.rds.amazonaws.com
DRIVER_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
TRIP_DB_ENDPOINT=ridenow-prod-trip-db.xxxx.ap-southeast-1.rds.amazonaws.com
TRIP_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
PAYMENT_DB_ENDPOINT=ridenow-prod-payment-db.xxxx.ap-southeast-1.rds.amazonaws.com
PAYMENT_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
MATCHING_DB_ENDPOINT=ridenow-prod-matching-db.xxxx.ap-southeast-1.rds.amazonaws.com
MATCHING_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
NOTIFICATION_DB_ENDPOINT=ridenow-prod-notification-db.xxxx.ap-southeast-1.rds.amazonaws.com
NOTIFICATION_DB_SECRET_ARN=arn:aws:secretsmanager:ap-southeast-1:123456789012:secret:rds!db-...
```

If a line starts with `STOP`, fix that problem and run the block again.

## What it creates

| Template logical ID | AWS resource type | Name |
|---|---|---|
| `DbKmsKey` | KMS key (symmetric) | No name; the alias below points to it |
| `DbKmsAlias` | KMS alias | `alias/ridenow-prod-rds` |
| `DbSubnetGroup` | RDS DB subnet group | `ridenow-prod-db-subnet-group` |
| `UserDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-user-db` |
| `DriverDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-driver-db` |
| `TripDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-trip-db` |
| `PaymentDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-payment-db` |
| `MatchingDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-matching-db` |
| `NotificationDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-notification-db` |
| RDS-managed master secret (one per instance) | `AWS::SecretsManager::Secret` | Created by RDS in Secrets Manager, encrypted with `alias/ridenow-prod-rds` |

Every instance uses the same settings:

| Setting | Value |
|---|---|
| Engine | PostgreSQL, latest `17.x` |
| Deployment | Multi-AZ DB instance |
| Instance class | `db.t4g.micro` |
| Storage | gp3, 20 GiB, no storage autoscaling |
| Public access | No |
| Encryption | Yes, with `alias/ridenow-prod-rds` |
| Credentials | Managed in AWS Secrets Manager |
| Backup retention | 30 days |
| Deletion protection | Off, and `DeletionPolicy` keeps a final snapshot |
| VPC | `ridenow-prod-vpc` |
| DB subnet group | `ridenow-prod-db-subnet-group` |
| Security group | `ridenow-prod-db-sg` |

Each instance has its own master user and initial database:

| Service | DB instance identifier | Master username | Initial database name |
|---|---|---|---|
| User | `ridenow-prod-user-db` | `app_user` | `user_service` |
| Driver | `ridenow-prod-driver-db` | `driver` | `driver_service` |
| Trip | `ridenow-prod-trip-db` | `trip` | `trip_service` |
| Payment | `ridenow-prod-payment-db` | `payment` | `payment_service` |
| Matching | `ridenow-prod-matching-db` | `matching` | `matching_service` |
| Notification | `ridenow-prod-notification-db` | `notification` | `notification_service` |

The databases follow the database-per-service rule: each service reads and writes only its own database, and no physical foreign key crosses a service boundary. Live driver locations are not stored in PostgreSQL.

## Depends on

Only 5.4. Select these by name in the console:

| Needed from 5.4 | Select in the console | Variable |
|---|---|---|
| Private subnets | `ridenow-prod-private-1`, `ridenow-prod-private-2` | `PRIVATE_SUBNET_IDS` |
| Database security group | `ridenow-prod-db-sg` in `ridenow-prod-vpc` | `DB_SG_ID` |

## Create the resources

Open the AWS console in the Asia Pacific (Singapore) `ap-southeast-1` region. Do the three steps in order.

### 1. KMS key (`DbKmsKey`, `DbKmsAlias`)

1. Go to **KMS -> Customer managed keys -> Create key**.
2. Set **Key type** to `Symmetric`.
3. Set **Key usage** to `Encrypt and decrypt`.
4. Under **Advanced options**, set **Key material origin** to `KMS`.
5. Under **Advanced options**, set **Regionality** to `Single-Region key`.
6. Select **Next**.
7. Set **Alias** to `ridenow-prod-rds`. The console shows it as `alias/ridenow-prod-rds`.
8. Set **Description** to `ridenow-prod RDS encryption key`.
9. Select **Next**.
10. Under **Key administrators**, tick your own IAM user or role.
11. Leave **Allow key administrators to delete this key** ticked, then select **Next**.
12. Under **Key users**, tick nothing, then select **Next**.
13. On **Review**, select **Finish**.
14. Open the new key and go to **Key policy -> Edit**.
15. Run the command below in your shell, copy the JSON it prints, and paste it over the whole policy.

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

16. Select **Save changes**.
17. Go to the **Key rotation** tab and select **Edit**.
18. Tick **Enable automatic key rotation**, then select **Save**.

The command prints the policy with your own account ID filled in. The wizard writes its own default policy, so this replacement is what matches the template.

### 2. DB subnet group (`DbSubnetGroup`)

1. Go to **RDS -> Subnet groups -> Create DB subnet group**.
2. Set **Name** to `ridenow-prod-db-subnet-group`.
3. Set **Description** to `ridenow-prod RDS private subnets`.
4. Set **VPC** to `ridenow-prod-vpc`.
5. Under **Availability Zones**, tick the two zones of `ridenow-prod-private-1` and `ridenow-prod-private-2`.
6. Under **Subnets**, tick the two private subnets. Check the CIDRs in **VPC -> Subnets** and do not pick `ridenow-prod-public-*`.
7. Select **Create**.

### 3. The six RDS instances

Create the six databases one after another. Each has its own heading below with every value filled in. Creation takes 10 to 20 minutes per instance, so start the next one while the previous one is Creating.

#### 3.1 `UserDb` (ridenow-prod-user-db)

| Template ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|
| `UserDb` | `ridenow-prod-user-db` | `app_user` | `user_service` |

**Start**

1. Go to **RDS -> Databases -> Create database**.
2. Select **Standard create**.

**Engine**

3. Set **Engine type** to `PostgreSQL`.
4. Set **Engine version** to the latest `17.x` in the list.
5. Set **Templates** to `Dev/Test`.

**Availability and durability**

6. Under **Deployment options**, select **Multi-AZ DB instance**. Do not pick Multi-AZ DB cluster.

**Settings**

7. Set **DB instance identifier** to `ridenow-prod-user-db`.
8. Set **Master username** to `app_user`.
9. Set **Credentials management** to **Managed in AWS Secrets Manager**.
10. Set **Encryption key** (for the secret) to `alias/ridenow-prod-rds`.

**Instance configuration**

11. Select **Burstable classes** and choose `db.t4g.micro`.

**Storage**

12. Set **Storage type** to `General Purpose SSD (gp3)`.
13. Set **Allocated storage** to `20` GiB.
14. Untick **Enable storage autoscaling**.

**Connectivity**

15. Set **Compute resource** to **Don't connect to an EC2 compute resource**.
16. Set **Network type** to `IPv4`.
17. Set **Virtual private cloud (VPC)** to `ridenow-prod-vpc`.
18. Set **DB subnet group** to `ridenow-prod-db-subnet-group`.
19. Set **Public access** to `No`.
20. Under **VPC security group (firewall)**, select **Choose existing** and pick `ridenow-prod-db-sg`.
21. Remove `default` from the selected security groups.
22. Set **Availability Zone** to `No preference`.

**Database authentication**

23. Select **Password authentication**.

**Monitoring**

24. Untick **Turn on Performance Insights**.
25. Untick **Enable Enhanced monitoring**.

**Additional configuration**

26. Set **Initial database name** to `user_service`.
27. Tick **Enable automated backups**.
28. Set **Backup retention period** to `30` days.
29. Tick **Enable encryption**.
30. Set **AWS KMS key** to `alias/ridenow-prod-rds`.
31. Leave **Enable auto minor version upgrade** ticked.
32. Untick **Enable deletion protection**.
33. Select **Create database**.

#### 3.2 `DriverDb` (ridenow-prod-driver-db)

| Template ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|
| `DriverDb` | `ridenow-prod-driver-db` | `driver` | `driver_service` |

**Start**

1. Go to **RDS -> Databases -> Create database**.
2. Select **Standard create**.

**Engine**

3. Set **Engine type** to `PostgreSQL`.
4. Set **Engine version** to the latest `17.x` in the list.
5. Set **Templates** to `Dev/Test`.

**Availability and durability**

6. Under **Deployment options**, select **Multi-AZ DB instance**. Do not pick Multi-AZ DB cluster.

**Settings**

7. Set **DB instance identifier** to `ridenow-prod-driver-db`.
8. Set **Master username** to `driver`.
9. Set **Credentials management** to **Managed in AWS Secrets Manager**.
10. Set **Encryption key** (for the secret) to `alias/ridenow-prod-rds`.

**Instance configuration**

11. Select **Burstable classes** and choose `db.t4g.micro`.

**Storage**

12. Set **Storage type** to `General Purpose SSD (gp3)`.
13. Set **Allocated storage** to `20` GiB.
14. Untick **Enable storage autoscaling**.

**Connectivity**

15. Set **Compute resource** to **Don't connect to an EC2 compute resource**.
16. Set **Network type** to `IPv4`.
17. Set **Virtual private cloud (VPC)** to `ridenow-prod-vpc`.
18. Set **DB subnet group** to `ridenow-prod-db-subnet-group`.
19. Set **Public access** to `No`.
20. Under **VPC security group (firewall)**, select **Choose existing** and pick `ridenow-prod-db-sg`.
21. Remove `default` from the selected security groups.
22. Set **Availability Zone** to `No preference`.

**Database authentication**

23. Select **Password authentication**.

**Monitoring**

24. Untick **Turn on Performance Insights**.
25. Untick **Enable Enhanced monitoring**.

**Additional configuration**

26. Set **Initial database name** to `driver_service`.
27. Tick **Enable automated backups**.
28. Set **Backup retention period** to `30` days.
29. Tick **Enable encryption**.
30. Set **AWS KMS key** to `alias/ridenow-prod-rds`.
31. Leave **Enable auto minor version upgrade** ticked.
32. Untick **Enable deletion protection**.
33. Select **Create database**.

#### 3.3 `TripDb` (ridenow-prod-trip-db)

| Template ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|
| `TripDb` | `ridenow-prod-trip-db` | `trip` | `trip_service` |

**Start**

1. Go to **RDS -> Databases -> Create database**.
2. Select **Standard create**.

**Engine**

3. Set **Engine type** to `PostgreSQL`.
4. Set **Engine version** to the latest `17.x` in the list.
5. Set **Templates** to `Dev/Test`.

**Availability and durability**

6. Under **Deployment options**, select **Multi-AZ DB instance**. Do not pick Multi-AZ DB cluster.

**Settings**

7. Set **DB instance identifier** to `ridenow-prod-trip-db`.
8. Set **Master username** to `trip`.
9. Set **Credentials management** to **Managed in AWS Secrets Manager**.
10. Set **Encryption key** (for the secret) to `alias/ridenow-prod-rds`.

**Instance configuration**

11. Select **Burstable classes** and choose `db.t4g.micro`.

**Storage**

12. Set **Storage type** to `General Purpose SSD (gp3)`.
13. Set **Allocated storage** to `20` GiB.
14. Untick **Enable storage autoscaling**.

**Connectivity**

15. Set **Compute resource** to **Don't connect to an EC2 compute resource**.
16. Set **Network type** to `IPv4`.
17. Set **Virtual private cloud (VPC)** to `ridenow-prod-vpc`.
18. Set **DB subnet group** to `ridenow-prod-db-subnet-group`.
19. Set **Public access** to `No`.
20. Under **VPC security group (firewall)**, select **Choose existing** and pick `ridenow-prod-db-sg`.
21. Remove `default` from the selected security groups.
22. Set **Availability Zone** to `No preference`.

**Database authentication**

23. Select **Password authentication**.

**Monitoring**

24. Untick **Turn on Performance Insights**.
25. Untick **Enable Enhanced monitoring**.

**Additional configuration**

26. Set **Initial database name** to `trip_service`.
27. Tick **Enable automated backups**.
28. Set **Backup retention period** to `30` days.
29. Tick **Enable encryption**.
30. Set **AWS KMS key** to `alias/ridenow-prod-rds`.
31. Leave **Enable auto minor version upgrade** ticked.
32. Untick **Enable deletion protection**.
33. Select **Create database**.

#### 3.4 `PaymentDb` (ridenow-prod-payment-db)

| Template ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|
| `PaymentDb` | `ridenow-prod-payment-db` | `payment` | `payment_service` |

**Start**

1. Go to **RDS -> Databases -> Create database**.
2. Select **Standard create**.

**Engine**

3. Set **Engine type** to `PostgreSQL`.
4. Set **Engine version** to the latest `17.x` in the list.
5. Set **Templates** to `Dev/Test`.

**Availability and durability**

6. Under **Deployment options**, select **Multi-AZ DB instance**. Do not pick Multi-AZ DB cluster.

**Settings**

7. Set **DB instance identifier** to `ridenow-prod-payment-db`.
8. Set **Master username** to `payment`.
9. Set **Credentials management** to **Managed in AWS Secrets Manager**.
10. Set **Encryption key** (for the secret) to `alias/ridenow-prod-rds`.

**Instance configuration**

11. Select **Burstable classes** and choose `db.t4g.micro`.

**Storage**

12. Set **Storage type** to `General Purpose SSD (gp3)`.
13. Set **Allocated storage** to `20` GiB.
14. Untick **Enable storage autoscaling**.

**Connectivity**

15. Set **Compute resource** to **Don't connect to an EC2 compute resource**.
16. Set **Network type** to `IPv4`.
17. Set **Virtual private cloud (VPC)** to `ridenow-prod-vpc`.
18. Set **DB subnet group** to `ridenow-prod-db-subnet-group`.
19. Set **Public access** to `No`.
20. Under **VPC security group (firewall)**, select **Choose existing** and pick `ridenow-prod-db-sg`.
21. Remove `default` from the selected security groups.
22. Set **Availability Zone** to `No preference`.

**Database authentication**

23. Select **Password authentication**.

**Monitoring**

24. Untick **Turn on Performance Insights**.
25. Untick **Enable Enhanced monitoring**.

**Additional configuration**

26. Set **Initial database name** to `payment_service`.
27. Tick **Enable automated backups**.
28. Set **Backup retention period** to `30` days.
29. Tick **Enable encryption**.
30. Set **AWS KMS key** to `alias/ridenow-prod-rds`.
31. Leave **Enable auto minor version upgrade** ticked.
32. Untick **Enable deletion protection**.
33. Select **Create database**.

#### 3.5 `MatchingDb` (ridenow-prod-matching-db)

| Template ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|
| `MatchingDb` | `ridenow-prod-matching-db` | `matching` | `matching_service` |

**Start**

1. Go to **RDS -> Databases -> Create database**.
2. Select **Standard create**.

**Engine**

3. Set **Engine type** to `PostgreSQL`.
4. Set **Engine version** to the latest `17.x` in the list.
5. Set **Templates** to `Dev/Test`.

**Availability and durability**

6. Under **Deployment options**, select **Multi-AZ DB instance**. Do not pick Multi-AZ DB cluster.

**Settings**

7. Set **DB instance identifier** to `ridenow-prod-matching-db`.
8. Set **Master username** to `matching`.
9. Set **Credentials management** to **Managed in AWS Secrets Manager**.
10. Set **Encryption key** (for the secret) to `alias/ridenow-prod-rds`.

**Instance configuration**

11. Select **Burstable classes** and choose `db.t4g.micro`.

**Storage**

12. Set **Storage type** to `General Purpose SSD (gp3)`.
13. Set **Allocated storage** to `20` GiB.
14. Untick **Enable storage autoscaling**.

**Connectivity**

15. Set **Compute resource** to **Don't connect to an EC2 compute resource**.
16. Set **Network type** to `IPv4`.
17. Set **Virtual private cloud (VPC)** to `ridenow-prod-vpc`.
18. Set **DB subnet group** to `ridenow-prod-db-subnet-group`.
19. Set **Public access** to `No`.
20. Under **VPC security group (firewall)**, select **Choose existing** and pick `ridenow-prod-db-sg`.
21. Remove `default` from the selected security groups.
22. Set **Availability Zone** to `No preference`.

**Database authentication**

23. Select **Password authentication**.

**Monitoring**

24. Untick **Turn on Performance Insights**.
25. Untick **Enable Enhanced monitoring**.

**Additional configuration**

26. Set **Initial database name** to `matching_service`.
27. Tick **Enable automated backups**.
28. Set **Backup retention period** to `30` days.
29. Tick **Enable encryption**.
30. Set **AWS KMS key** to `alias/ridenow-prod-rds`.
31. Leave **Enable auto minor version upgrade** ticked.
32. Untick **Enable deletion protection**.
33. Select **Create database**.

#### 3.6 `NotificationDb` (ridenow-prod-notification-db)

| Template ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|
| `NotificationDb` | `ridenow-prod-notification-db` | `notification` | `notification_service` |

**Start**

1. Go to **RDS -> Databases -> Create database**.
2. Select **Standard create**.

**Engine**

3. Set **Engine type** to `PostgreSQL`.
4. Set **Engine version** to the latest `17.x` in the list.
5. Set **Templates** to `Dev/Test`.

**Availability and durability**

6. Under **Deployment options**, select **Multi-AZ DB instance**. Do not pick Multi-AZ DB cluster.

**Settings**

7. Set **DB instance identifier** to `ridenow-prod-notification-db`.
8. Set **Master username** to `notification`.
9. Set **Credentials management** to **Managed in AWS Secrets Manager**.
10. Set **Encryption key** (for the secret) to `alias/ridenow-prod-rds`.

**Instance configuration**

11. Select **Burstable classes** and choose `db.t4g.micro`.

**Storage**

12. Set **Storage type** to `General Purpose SSD (gp3)`.
13. Set **Allocated storage** to `20` GiB.
14. Untick **Enable storage autoscaling**.

**Connectivity**

15. Set **Compute resource** to **Don't connect to an EC2 compute resource**.
16. Set **Network type** to `IPv4`.
17. Set **Virtual private cloud (VPC)** to `ridenow-prod-vpc`.
18. Set **DB subnet group** to `ridenow-prod-db-subnet-group`.
19. Set **Public access** to `No`.
20. Under **VPC security group (firewall)**, select **Choose existing** and pick `ridenow-prod-db-sg`.
21. Remove `default` from the selected security groups.
22. Set **Availability Zone** to `No preference`.

**Database authentication**

23. Select **Password authentication**.

**Monitoring**

24. Untick **Turn on Performance Insights**.
25. Untick **Enable Enhanced monitoring**.

**Additional configuration**

26. Set **Initial database name** to `notification_service`.
27. Tick **Enable automated backups**.
28. Set **Backup retention period** to `30` days.
29. Tick **Enable encryption**.
30. Set **AWS KMS key** to `alias/ridenow-prod-rds`.
31. Leave **Enable auto minor version upgrade** ticked.
32. Untick **Enable deletion protection**.
33. Select **Create database**.

Leave everything not listed at its default. When you delete an instance later, keep **Create final snapshot** ticked.

## Values to record

Run the second block of Set variables now. The values below are looked up for you, so you copy nothing by hand. Later pages restate the same lookups at their top.

| Variable | What it holds | Used by |
|---|---|---|
| `RDS_KMS_KEY_ARN` | ARN of the key behind `alias/ridenow-prod-rds` | 5.8 (policy that lets External Secrets decrypt the database secrets) |
| `USER_DB_ENDPOINT`, `DRIVER_DB_ENDPOINT`, `TRIP_DB_ENDPOINT`, `PAYMENT_DB_ENDPOINT`, `MATCHING_DB_ENDPOINT`, `NOTIFICATION_DB_ENDPOINT` | Host name of each instance | 5.9 (database connection of each workload) |
| `USER_DB_SECRET_ARN`, `DRIVER_DB_SECRET_ARN`, `TRIP_DB_SECRET_ARN`, `PAYMENT_DB_SECRET_ARN`, `MATCHING_DB_SECRET_ARN`, `NOTIFICATION_DB_SECRET_ARN` | ARN of the RDS-managed master secret of each instance | 5.8 (database secrets read by External Secrets) |

All thirteen values have a lookup command, so none of them is a Manual_Value. The console shows the same data under **RDS -> Databases -> the instance -> Connectivity & security** (endpoint) and **Configuration** (master credentials ARN).

## Post-deploy steps

No post-deploy step is needed for this page. The inbound rule that lets the cluster reach the databases on port 5432 is created in 5.9.

## Verify

Run these commands in the same shell:

```bash
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].[DBInstanceIdentifier,DBInstanceStatus,StorageEncrypted,MultiAZ,PubliclyAccessible,BackupRetentionPeriod]" \
  --output table
aws kms get-key-rotation-status --key-id "alias/${CLUSTER_NAME}-rds" --query KeyRotationEnabled --output text
```

Expected result: six rows, each with `available`, `True`, `True`, `False`, and `30`; then `True` for key rotation. If a row is missing or not `available`, open **RDS -> Databases -> that instance -> Logs & events**.

```bash
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].VpcSecurityGroups[].VpcSecurityGroupId" \
  --output text | tr '\t' '\n' | sort -u
echo "DB_SG_ID=$DB_SG_ID"
aws secretsmanager describe-secret --secret-id "$USER_DB_SECRET_ARN" --query KmsKeyId --output text
```

Expected result: the first line is the same security group ID as `DB_SG_ID` (one line only), and the last line is the key ARN of `alias/ridenow-prod-rds`:

```text
sg-0fff...
DB_SG_ID=sg-0fff...
arn:aws:kms:ap-southeast-1:123456789012:key/1234abcd-...
```

The next page is 5.7 Messaging.
