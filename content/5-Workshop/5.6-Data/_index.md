---
title : "Data"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

## 5.6.1. Overview

### Problem Statement

A self-managed Postgres instance (running inside a Docker container) **lacks**:

- Automatic backup
- Failover
- Automated patching
- Built-in monitoring

**Amazon RDS** provides all of these capabilities while running the same PostgreSQL engine, so Flyway migrations, JPA, and SQL queries work identically — no code changes required.

### Design Principles

The system follows **Database-per-Service**:

| Principle | Detail |
|---|---|
| Ownership | Each microservice owns its own database/schema |
| Cross-service FK | **No** physical foreign keys between services |
| Reference | Only UUIDs are stored (logical references) |
| Real-time location | Not stored in PostgreSQL — uses Redis GEO instead |

Deployment region: `ap-southeast-1`  
Environment: `prod`

---

## 5.6.2. Data Architecture

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  User Service   │     │ Driver Service  │     │  Trip Service   │
│  user_service   │     │ driver_service  │     │  trip_service   │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ Payment Service │     │Matching Service │     │Notification Svc │
│payment_service  │     │matching_service │     │notification_svc │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

Each service reads and writes only its own database. Cross-service references are UUID-only.

---

## 5.6.3. AWS RDS Infrastructure (Production)

### 5.6.3.1. Resources Created

| Logical ID | Resource Type | Actual Name |
|---|---|---|
| `DbKmsKey` | KMS Key (Symmetric) | – |
| `DbKmsAlias` | KMS Alias | `alias/ridenow-prod-rds` |
| `DbSubnetGroup` | RDS DB Subnet Group | `ridenow-prod-db-subnet-group` |
| `UserDb` | RDS PostgreSQL | `ridenow-prod-user-db` |
| `DriverDb` | RDS PostgreSQL | `ridenow-prod-driver-db` |
| `TripDb` | RDS PostgreSQL | `ridenow-prod-trip-db` |
| `PaymentDb` | RDS PostgreSQL | `ridenow-prod-payment-db` |
| `MatchingDb` | RDS PostgreSQL | `ridenow-prod-matching-db` |
| `NotificationDb` | RDS PostgreSQL | `ridenow-prod-notification-db` |

### 5.6.3.2. Common Configuration (Production)

| Property | Value |
|---|---|
| Engine | PostgreSQL 17.x |
| Deployment | Multi-AZ DB instance |
| Instance class | `db.t4g.micro` |
| Storage | gp3, 20 GiB |
| Public access | No |
| Encryption | Yes (`alias/ridenow-prod-rds`) |
| Credentials | AWS Secrets Manager |
| Backup retention | 30 days |
| VPC | `ridenow-prod-vpc` |
| Security group | `ridenow-prod-db-sg` |

### 5.6.3.3. Per-Database Details

| Service | DB Identifier | Master username | Initial DB name |
|---|---|---|---|
| User | `ridenow-prod-user-db` | `app_user` | `user_service` |
| Driver | `ridenow-prod-driver-db` | `driver` | `driver_service` |
| Trip | `ridenow-prod-trip-db` | `trip` | `trip_service` |
| Payment | `ridenow-prod-payment-db` | `payment` | `payment_service` |
| Matching | `ridenow-prod-matching-db` | `matching` | `matching_service` |
| Notification | `ridenow-prod-notification-db` | `notification` | `notification_service` |

---

## 5.6.4. Hands-on: Migrate driver-service to RDS

This section walks through migrating from a self-managed Postgres container to a managed RDS instance, using **driver-service** as the example.

### 5.6.4.1. Create the RDS Instance (AWS Console)

1. Go to **RDS Console → Create database**
2. Select **Standard create**
3. **Engine**: PostgreSQL, Version **17.x** (matching docker-compose)
4. **Template**: Free tier (or Dev/Test)
5. **DB instance identifier**: `ridenow-driver-db`
6. **Master username**: `driver`
7. **Master password**: set a strong password → **save it**
8. **VPC**: select the project VPC
9. **Subnet group**: create new or use default
10. **Public access**: **No** (access from within VPC only)
11. **VPC security groups**: select `ridenow-db-sg`
12. **Initial database name**: `driver_service`
13. **Backup retention**: 7 days (Free tier) or 30 days (prod)
14. **Create database** (takes roughly 5–10 minutes)

### 5.6.4.2. Get the Endpoint

Go to **RDS → Databases → ridenow-driver-db → Connectivity & security**

- **Endpoint**: `ridenow-driver-db.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com`
- **Port**: `5432`

### 5.6.4.3. Test Connection from Local Machine

> Temporarily open the security group to allow your IP.

```bash
psql -h ridenow-driver-db.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com \
     -U driver \
     -d driver_service \
     -p 5432
```

### 5.6.4.4. Point driver-service to RDS

Only **one env var** needs to change — **no image rebuild required**:

```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://ridenow-driver-db.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com:5432/driver_service \
SPRING_DATASOURCE_USERNAME=driver \
SPRING_DATASOURCE_PASSWORD=<password> \
./mvnw spring-boot:run
```

Flyway will automatically run `V1__init_schema.sql` on RDS when the service boots for the first time.

### 5.6.4.5. Verify End-to-End

```bash
# Create a driver → data is stored in RDS
curl -X POST http://localhost:8080/api/drivers \
  -H "Content-Type: application/json" \
  -d '{"userId":"...","licenseNumber":"B2-RDS-TEST"}'
```

Verify in `psql`:

```sql
SELECT id, license_number FROM drivers;
```

### 5.6.4.6. Key Points to Understand

| Point | Explanation |
|---|---|
| Same schema | Flyway V1 script, JPA entity, and SQL queries run identically |
| Only the data location changes | From a Postgres container → RDS managed |
| Password | Will be moved to **AWS Secrets Manager** when deploying to EKS (never commit to git) |
| Backup & Failover | RDS auto-backs up daily; Multi-AZ failover is an option (not in free tier) |

### Practice Goals

1) `driver-service` runs and can create a driver
2) Data resides in RDS (verified via `psql`)
3) Flyway migration runs successfully on RDS

---

## 5.6.5. Data Schema (DBML)

The schema follows microservice principles. All cross-service relationships are logical only.

### 5.6.5.1. User Service

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

### 5.6.5.2. Driver Service

```dbml
Table drivers {
  id uuid [primary key]
  user_id uuid [unique, not null]          // Logical ref → User Service
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

### 5.6.5.3. Trip Service

```dbml
Table trips {
  id uuid [primary key]
  passenger_id uuid [not null]
  driver_id uuid
  vehicle_id uuid
  ride_type varchar [not null]             // MOTORBIKE | CAR
  pickup_lat decimal(9,6) [not null]
  pickup_lng decimal(9,6) [not null]
  pickup_address varchar
  destination_lat decimal(9,6) [not null]
  destination_lng decimal(9,6) [not null]
  destination_address varchar
  status varchar [not null, default: 'REQUESTED']
  // REQUESTED | SEARCHING | DRIVER_ASSIGNED | DRIVER_ARRIVED
  // IN_PROGRESS | COMPLETED | CANCELLED
  estimated_fare decimal(12,2)
  final_fare decimal(12,2)
  cancellation_reason varchar
  version bigint [not null, default: 0]    // Optimistic locking
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
  // PENDING | ACCEPTED | DECLINED | EXPIRED
  distance_meters integer
  offered_at timestamp [not null]
  responded_at timestamp
  expires_at timestamp
}
```

### 5.6.5.4. Matching Service

```dbml
Table matching_driver_positions {
  driver_id uuid [primary key]
  vehicle_type varchar [not null]
  lat decimal(9,6) [not null]
  lng decimal(9,6) [not null]
  reported_at timestamp [not null]
}
```

> This is not the primary store for driver location. The primary store is **Redis GEO**. This table serves local-first matching and debugging only.

### 5.6.5.5. Payment Service

```dbml
Table payments {
  id uuid [primary key]
  trip_id uuid [unique, not null]
  amount decimal(12,2) [not null]
  currency varchar(3) [not null, default: 'VND']
  method varchar [not null]                // CASH | SIMULATED_CARD | PAYPAL
  status varchar [not null, default: 'PENDING']
  // PENDING → PROCESSING → AUTHORIZED → PAID / RELEASED / FAILED
  idempotency_key varchar [unique, not null]
  failure_reason varchar
  created_at timestamp [not null]
  updated_at timestamp [not null]
  paid_at timestamp
  failed_at timestamp
}
```

### 5.6.5.6. Notification Worker

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

### 5.6.5.7. Rating (Logical Domain)

```dbml
Table ratings {
  id uuid [primary key]
  trip_id uuid [not null]
  reviewer_user_id uuid [not null]
  target_user_id uuid [not null]
  rating_type varchar [not null]
  // PASSENGER_TO_DRIVER | DRIVER_TO_PASSENGER
  score smallint [not null]                // 1–5
  comment text
  created_at timestamp [not null]
  updated_at timestamp [not null]
}
```

---

## 5.6.6. Logical Relationships Between Entities

The relationships below are **domain-level descriptions only** — no physical foreign keys are created:

| Relationship | Description |
|---|---|
| `drivers.user_id` → `users.id` | Driver is linked to a user |
| `vehicles.driver_id` → `drivers.id` | Vehicle belongs to a driver |
| `trips.passenger_id` → `users.id` | Passenger on the trip |
| `trips.driver_id` → `drivers.id` | Assigned driver |
| `trips.vehicle_id` → `vehicles.id` | Vehicle used for the trip |
| `trip_offers.trip_id` → `trips.id` | Offer belongs to a trip |
| `trip_offers.driver_id` → `drivers.id` | Offer sent to a driver |
| `payments.trip_id` → `trips.id` | 1-to-1 with a trip |
| `ratings.trip_id` → `trips.id` | Rating tied to a trip |
| `ratings.reviewer_user_id` → `users.id` | The reviewer |
| `ratings.target_user_id` → `users.id` | The reviewed user |

---

## 5.6.7. Post-Deployment Verification

```bash
# Check all 6 RDS instances
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].[DBInstanceIdentifier,DBInstanceStatus,StorageEncrypted,MultiAZ,PubliclyAccessible,BackupRetentionPeriod]" \
  --output table

# Check KMS key rotation
aws kms get-key-rotation-status \
  --key-id alias/ridenow-prod-rds \
  --query KeyRotationEnabled \
  --output text
```

**Expected results:**

- 6 rows: `available`, `True` (encrypted), `True` (Multi-AZ), `False` (public), `30`
- Key rotation: `True`

---

## 5.6.8. Summary

| Item | Detail |
|---|---|
| Number of databases (prod) | 6 (User, Driver, Trip, Payment, Matching, Notification) |
| Engine | PostgreSQL 17.x |
| Multi-AZ | Yes (prod) |
| Encryption | Yes (Customer Managed KMS) |
| Public access | No |
| Backup | 7 days (stage) / 30 days (prod) |
| Schema ownership | Each service manages its own |
| Cross-service FK | Not used (logical UUID references only) |
| Real-time location | Redis GEO (not stored in PostgreSQL) |
| Migration tool | Flyway (runs identically on Docker Postgres and RDS) |

**Conclusion:**  
The same Flyway scripts and JPA entities work on both Docker Postgres and RDS. The only change needed is the connection string.
