---
title : "RDS: Managed PostgreSQL"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

## 5.6.1. Objectives

This chapter walks through **creating and deploying** the data layer of the RideNow system on Amazon RDS (PostgreSQL). Once complete, the system will have:

- 1 KMS Key for encryption
- 1 DB Subnet Group
- 6 RDS PostgreSQL instances (one database per service)
- Schemas ready for Flyway migration

**Region:** `ap-southeast-1`  
**Environment:** `prod`

---

## 5.6.2. Problem Statement

A self-managed Postgres running inside a container **lacks**:

| Feature | Self-managed | Amazon RDS |
|---|---|---|
| Automatic backup | Must configure manually | Built-in |
| Multi-AZ Failover | Must set up manually | Built-in |
| Automated patching | Manual | Automatic |
| Integrated monitoring | Requires extra tooling | CloudWatch included |
| Encryption at rest | Must manage keys manually | KMS integrated |

RDS uses the **same PostgreSQL engine** → Flyway, JPA, and SQL run identically; only the connection string needs to change.

---

## 5.6.3. Dependencies

This chapter depends only on the Network layer (`02-network`). The following resources must already exist:

| Resource | Name |
|---|---|
| VPC | `ridenow-prod-vpc` |
| Private Subnet 1 | `ridenow-prod-private-1` |
| Private Subnet 2 | `ridenow-prod-private-2` |
| Security Group | `ridenow-prod-db-sg` |

> If these do not exist, complete the Network chapter before continuing.

---

## 5.6.4. Resources to Be Created

| Logical ID | AWS Type | Actual Name |
|---|---|---|
| `DbKmsKey` | KMS Key (Symmetric) | – |
| `DbKmsAlias` | KMS Alias | `alias/ridenow-prod-rds` |
| `DbSubnetGroup` | RDS DB Subnet Group | `ridenow-prod-db-subnet-group` |
| `UserDb` | RDS DB Instance | `ridenow-prod-user-db` |
| `DriverDb` | RDS DB Instance | `ridenow-prod-driver-db` |
| `TripDb` | RDS DB Instance | `ridenow-prod-trip-db` |
| `PaymentDb` | RDS DB Instance | `ridenow-prod-payment-db` |
| `MatchingDb` | RDS DB Instance | `ridenow-prod-matching-db` |
| `NotificationDb` | RDS DB Instance | `ridenow-prod-notification-db` |

---

## 5.6.5. Step 1: Create the KMS Key

Purpose: Encrypt data at rest for all RDS instances and Secrets Manager.

### Steps in the Console

1. Go to **KMS → Customer managed keys → Create key**
2. **Key type**: `Symmetric`
3. **Key usage**: `Encrypt and decrypt`
4. **Key material origin**: `KMS`
5. **Regionality**: `Single-Region key`
6. **Alias**: `ridenow-prod-rds`  
   (The console will display `alias/ridenow-prod-rds`)
7. **Description**: `ridenow-prod RDS encryption key`
8. **Key administrators**: select your IAM user/role
9. Leave **Allow key administrators to delete this key** checked
10. **Key users**: leave empty
11. **Finish**

### Update the Key Policy

After creation:
1. Open the key → **Key policy → Edit**
2. Replace the entire policy with the following (replace `<ACCOUNT_ID>` with your 12-digit account ID):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountAdmin",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::<ACCOUNT_ID>:root" },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
```

3. **Save changes**

### Enable Key Rotation

1. Go to the **Key rotation** tab → **Edit**
2. Check **Enable automatic key rotation**
3. **Save**

---

## 5.6.6. Step 2: Create the DB Subnet Group

Purpose: Specify that RDS runs inside two private subnets (Multi-AZ).

### Steps in the Console

1. Go to **RDS → Subnet groups → Create DB subnet group**
2. **Name**: `ridenow-prod-db-subnet-group`
3. **Description**: `ridenow-prod RDS private subnets`
4. **VPC**: `ridenow-prod-vpc`
5. **Availability Zones**: select the 2 AZs matching the private subnets
6. **Subnets**: check `ridenow-prod-private-1` and `ridenow-prod-private-2`  
   (verify the CIDRs — do **not** select public subnets)
7. **Create**

---

## 5.6.7. Step 3: Create the RDS Instances

Create all 6 databases one by one. Each instance takes roughly 10–20 minutes — you can start the next instance while the previous one is still in `Creating` state.

### Common Configuration for All Instances

| Property | Value |
|---|---|
| Engine type | PostgreSQL |
| Engine version | Latest `17.x` |
| Templates | Dev/Test |
| Deployment options | **Multi-AZ DB instance** (do not select Multi-AZ DB cluster) |
| Instance class | Burstable classes → `db.t4g.micro` |
| Storage type | General Purpose SSD (gp3) |
| Allocated storage | 20 GiB |
| Storage autoscaling | **Disabled** |
| Compute resource | Don't connect to an EC2 compute resource |
| Network type | IPv4 |
| VPC | `ridenow-prod-vpc` |
| DB subnet group | `ridenow-prod-db-subnet-group` |
| Public access | **No** |
| VPC security group | Choose existing → `ridenow-prod-db-sg` (remove `default`) |
| Availability Zone | No preference |
| Database authentication | Password authentication |
| Performance Insights | **Disabled** |
| Enhanced monitoring | **Disabled** |
| Automated backups | **Enabled** |
| Backup retention | 30 days |
| Encryption | **Enabled** → KMS key = `alias/ridenow-prod-rds` |
| Auto minor version upgrade | Enabled |
| Deletion protection | **Disabled** |
| Credentials management | **Managed in AWS Secrets Manager** |
| Encryption key (for secret) | `alias/ridenow-prod-rds` |

### Per-Instance Details

| # | Logical ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|---|
| 1 | `UserDb` | `ridenow-prod-user-db` | `app_user` | `user_service` |
| 2 | `DriverDb` | `ridenow-prod-driver-db` | `driver` | `driver_service` |
| 3 | `TripDb` | `ridenow-prod-trip-db` | `trip` | `trip_service` |
| 4 | `PaymentDb` | `ridenow-prod-payment-db` | `payment` | `payment_service` |
| 5 | `MatchingDb` | `ridenow-prod-matching-db` | `matching` | `matching_service` |
| 6 | `NotificationDb` | `ridenow-prod-notification-db` | `notification` | `notification_service` |

### Creation Procedure (repeat 6 times)

1. **RDS → Databases → Create database**
2. Select **Standard create**
3. Fill in values from the common configuration table and the per-instance table above
4. **Create database**

> **Note**: When deleting an instance later, keep the **Create final snapshot** option selected.

---

## 5.6.8. Values to Record After Creation

Once all instances reach `available` status, record the following values:

| Output | Where to Find It in the Console | Shell Variable |
|---|---|---|
| UserDbEndpoint | `ridenow-prod-user-db` → Connectivity & security → Endpoint | `USER_DB_ENDPOINT` |
| UserDbSecretArn | Configuration → Master credentials ARN | `USER_DB_SECRET_ARN` |
| DriverDbEndpoint | Endpoint | `DRIVER_DB_ENDPOINT` |
| DriverDbSecretArn | Master credentials ARN | `DRIVER_DB_SECRET_ARN` |
| TripDbEndpoint | Endpoint | `TRIP_DB_ENDPOINT` |
| TripDbSecretArn | Master credentials ARN | `TRIP_DB_SECRET_ARN` |
| PaymentDbEndpoint | Endpoint | `PAYMENT_DB_ENDPOINT` |
| PaymentDbSecretArn | Master credentials ARN | `PAYMENT_DB_SECRET_ARN` |
| MatchingDbEndpoint | Endpoint | `MATCHING_DB_ENDPOINT` |
| MatchingDbSecretArn | Master credentials ARN | `MATCHING_DB_SECRET_ARN` |
| NotificationDbEndpoint | Endpoint | `NOTIFICATION_DB_ENDPOINT` |
| NotificationDbSecretArn | Master credentials ARN | `NOTIFICATION_DB_SECRET_ARN` |

These values will be used in chapter **07 – EKS Add-ons**.

---

## 5.6.9. Verification

Run the following commands:

```bash
# 1. Check all 6 RDS instances
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].[DBInstanceIdentifier,DBInstanceStatus,StorageEncrypted,MultiAZ,PubliclyAccessible,BackupRetentionPeriod]" \
  --output table

# 2. Check KMS key rotation
aws kms get-key-rotation-status \
  --key-id alias/ridenow-prod-rds \
  --query KeyRotationEnabled \
  --output text
```

### Expected Results

- 6 rows, each showing:
  - Status = `available`
  - StorageEncrypted = `True`
  - MultiAZ = `True`
  - PubliclyAccessible = `False`
  - BackupRetentionPeriod = `30`
- Key rotation = `True`

If any instance is missing or shows a status other than `available`, go to **RDS → Databases → [instance] → Logs & events** to investigate.

---

## 5.6.10. Data Schema

Once RDS is ready, each service uses **Flyway** to run migrations that create the schema.

### Key Principles

- Each service owns only its own schema.
- Cross-service references **store UUIDs only** — no physical foreign keys.
- Real-time driver location is **not** stored in PostgreSQL — Redis GEO is used instead.

### 5.6.10.1. User Service (`user_service`)

```dbml
Table users {
  id uuid [primary key]
  cognito_sub varchar [unique, not null]
  email varchar [unique, not null]
  phone varchar [unique]
  full_name varchar [not null]
  avatar_url varchar
  status varchar [not null, default: 'ACTIVE']  // ACTIVE | SUSPENDED
  created_at timestamp [not null]
  updated_at timestamp [not null]
}
```

### 5.6.10.2. Driver Service (`driver_service`)

```dbml
Table drivers {
  id uuid [primary key]
  user_id uuid [unique, not null]          // Logical ref → User
  license_number varchar [unique, not null]
  status varchar [not null, default: 'ACTIVE']
  average_rating decimal(3,2)
  total_trips bigint [not null, default: 0]
  created_at timestamp [not null]
  updated_at timestamp [not null]
}

Table vehicles {
  id uuid [primary key]
  driver_id uuid [not null]
  plate_number varchar [unique, not null]
  vehicle_type varchar [not null]           // MOTORBIKE | CAR
  brand varchar
  model varchar
  color varchar
  active boolean [not null, default: true]
  created_at timestamp [not null]
  updated_at timestamp [not null]
}
```

### 5.6.10.3. Trip Service (`trip_service`)

```dbml
Table trips {
  id uuid [primary key]
  passenger_id uuid [not null]
  driver_id uuid
  vehicle_id uuid
  ride_type varchar [not null]
  pickup_lat decimal(9,6) [not null]
  pickup_lng decimal(9,6) [not null]
  pickup_address varchar
  destination_lat decimal(9,6) [not null]
  destination_lng decimal(9,6) [not null]
  destination_address varchar
  status varchar [not null, default: 'REQUESTED']
  estimated_fare decimal(12,2)
  final_fare decimal(12,2)
  cancellation_reason varchar
  version bigint [not null, default: 0]
  requested_at timestamp [not null]
  accepted_at timestamp
  arrived_at timestamp
  started_at timestamp
  completed_at timestamp
  cancelled_at timestamp
  created_at timestamp [not null]
  updated_at timestamp [not null]
}

Table trip_offers {
  id uuid [primary key]
  trip_id uuid [not null]
  driver_id uuid [not null]
  status varchar [not null, default: 'PENDING']
  distance_meters integer
  offered_at timestamp [not null]
  responded_at timestamp
  expires_at timestamp
}
```

### 5.6.10.4. Matching Service (`matching_service`)

```dbml
Table matching_driver_positions {
  driver_id uuid [primary key]
  vehicle_type varchar [not null]
  lat decimal(9,6) [not null]
  lng decimal(9,6) [not null]
  reported_at timestamp [not null]
}
```

> This table is a working copy only. The primary store for driver location is Redis GEO.

### 5.6.10.5. Payment Service (`payment_service`)

```dbml
Table payments {
  id uuid [primary key]
  trip_id uuid [unique, not null]
  amount decimal(12,2) [not null]
  currency varchar(3) [not null, default: 'VND']
  method varchar [not null]
  status varchar [not null, default: 'PENDING']
  idempotency_key varchar [unique, not null]
  failure_reason varchar
  created_at timestamp [not null]
  updated_at timestamp [not null]
  paid_at timestamp
  failed_at timestamp
}
```

### 5.6.10.6. Notification Service (`notification_service`)

```dbml
Table notifications {
  id uuid [primary key]
  user_id uuid [not null]
  event_id uuid [not null]
  type varchar [not null]
  message varchar [not null]
  trip_id uuid
  occurred_at timestamp
  created_at timestamp [not null]
  read_at timestamp
}
```

### 5.6.10.7. Rating (Logical Domain)

```dbml
Table ratings {
  id uuid [primary key]
  trip_id uuid [not null]
  reviewer_user_id uuid [not null]
  target_user_id uuid [not null]
  rating_type varchar [not null]
  score smallint [not null]                // 1–5
  comment text
  created_at timestamp [not null]
  updated_at timestamp [not null]
}
```

---

## 5.6.11. Definition of Done

- [ ] KMS Key `alias/ridenow-prod-rds` created with rotation enabled
- [ ] DB Subnet Group `ridenow-prod-db-subnet-group` created with 2 private subnets
- [ ] All 6 RDS instances are in `available` state
- [ ] All instances: Encrypted = True, Multi-AZ = True, Public = False
- [ ] Backup retention = 30 days
- [ ] All 12 Endpoint + SecretArn values recorded
- [ ] Verification commands return the expected results

---

## 5.6.12. Summary

| Item | Value |
|---|---|
| Number of RDS instances | 6 |
| Engine | PostgreSQL 17.x |
| Multi-AZ | Yes |
| Encryption | Customer Managed KMS |
| Public access | No |
| Backup | 30 days |
| Credentials | AWS Secrets Manager |
| Schema ownership | One database per service |
| Cross-service FK | No (logical UUID references only) |

**Key takeaway:**  
RDS provides managed PostgreSQL with backup, failover, and encryption out of the box. Application code (Flyway + JPA) **requires no changes** — just point the connection string to the new endpoint.
