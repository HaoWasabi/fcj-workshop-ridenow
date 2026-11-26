---
title : "Data"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

## 5.6.1. Tổng quan

### Vấn đề cần giải quyết

Self-managed Postgres (chạy trong container Docker) **không có**:

- Automatic backup
- Failover
- Automated patching
- Monitoring tích hợp

**Amazon RDS** cung cấp tất cả các tính năng trên với cùng PostgreSQL engine nên Flyway migration, JPA, SQL đều chạy y hệt, không cần thay đổi code.

### Nguyên tắc thiết kế

Hệ thống tuân thủ **Database-per-Service**:

| Nguyên tắc | Chi tiết |
|---|---|
| Ownership | Mỗi microservice sở hữu database/schema riêng |
| Cross-service FK | **Không** tạo foreign key vật lý giữa các service |
| Reference | Chỉ lưu UUID (logical reference) |
| Real-time location | Không lưu trong PostgreSQL → dùng Redis GEO |

Vùng triển khai: `ap-southeast-1`  
Môi trường: `prod`

---

## 5.6.2. Kiến trúc dữ liệu

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

Mỗi service chỉ đọc/ghi database của chính nó. Cross-service reference chỉ là UUID.

---

## 5.6.3. Hạ tầng AWS RDS (Production)

### 5.6.3.1. Các resource được tạo

| Logical ID | Loại resource | Tên thực tế |
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

### 5.6.3.2. Cấu hình chung (Production)

| Thuộc tính | Giá trị |
|---|---|
| Engine | PostgreSQL 17.x |
| Deployment | Multi-AZ DB instance |
| Instance class | `db.t4g.micro` |
| Storage | gp3, 20 GiB |
| Public access | No |
| Encryption | Có (`alias/ridenow-prod-rds`) |
| Credentials | AWS Secrets Manager |
| Backup retention | 30 ngày |
| VPC | `ridenow-prod-vpc` |
| Security group | `ridenow-prod-db-sg` |

### 5.6.3.3. Chi tiết từng database

| Service | DB Identifier | Master username | Initial DB name |
|---|---|---|---|
| User | `ridenow-prod-user-db` | `app_user` | `user_service` |
| Driver | `ridenow-prod-driver-db` | `driver` | `driver_service` |
| Trip | `ridenow-prod-trip-db` | `trip` | `trip_service` |
| Payment | `ridenow-prod-payment-db` | `payment` | `payment_service` |
| Matching | `ridenow-prod-matching-db` | `matching` | `matching_service` |
| Notification | `ridenow-prod-notification-db` | `notification` | `notification_service` |

---

## 5.6.4. Thực hành: Migrate driver-service sang RDS

Phần này hướng dẫn chuyển từ self-managed Postgres container sang RDS managed, lấy **driver-service** làm ví dụ.

### 5.6.4.1. Tạo RDS instance (AWS Console)

1. Vào **RDS Console → Create database**
2. Chọn **Standard create**
3. **Engine**: PostgreSQL, Version **17.x** (khớp với docker-compose)
4. **Template**: Free tier (hoặc Dev/Test)
5. **DB instance identifier**: `ridenow-driver-db`
6. **Master username**: `driver`
7. **Master password**: đặt mật khẩu mạnh → **lưu lại**
8. **VPC**: chọn VPC của dự án
9. **Subnet group**: tạo mới hoặc dùng default
10. **Public access**: **No** (chỉ access từ trong VPC)
11. **VPC security groups**: chọn `ridenow-db-sg`
12. **Initial database name**: `driver_service`
13. **Backup retention**: 7 days (Free tier) hoặc 30 days (prod)
14. **Create database** (mất khoảng 5–10 phút)

### 5.6.4.2. Lấy Endpoint

Vào **RDS → Databases → ridenow-driver-db → Connectivity & security**

- **Endpoint**: `ridenow-driver-db.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com`
- **Port**: `5432`

### 5.6.4.3. Test kết nối từ laptop

> Tạm thời mở security group cho phép IP của bạn.

```bash
psql -h ridenow-driver-db.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com \
     -U driver \
     -d driver_service \
     -p 5432
```

### 5.6.4.4. Trỏ driver-service local đến RDS

Chỉ cần thay đổi **1 env var**, **không rebuild image**:

```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://ridenow-driver-db.xxxxxxxxxx.ap-southeast-1.rds.amazonaws.com:5432/driver_service \
SPRING_DATASOURCE_USERNAME=driver \
SPRING_DATASOURCE_PASSWORD=<password> \
./mvnw spring-boot:run
```

Flyway sẽ tự chạy `V1__init_schema.sql` trên RDS khi service boot lần đầu.

### 5.6.4.5. Verify end-to-end

```bash
# Tạo driver → data lưu vào RDS
curl -X POST http://localhost:8080/api/drivers \
  -H "Content-Type: application/json" \
  -d '{"userId":"...","licenseNumber":"B2-RDS-TEST"}'
```

Kiểm tra trong `psql`:

```sql
SELECT id, license_number FROM drivers;
```

### 5.6.4.6. Điểm quan trọng cần giải thích

| Điểm | Giải thích |
|---|---|
| Cùng schema | Flyway V1 script, JPA entity, SQL query chạy y hệt |
| Chỉ khác nơi lưu data | Từ container Postgres → RDS managed |
| Password | Sẽ chuyển vào **AWS Secrets Manager** khi lên EKS (không commit vào git) |
| Backup & Failover | RDS tự backup hàng ngày; Multi-AZ failover là option (không free tier) |

### Mục tiêu thực hành

1) `driver-service` chạy và tạo được driver
2) Data nằm trong RDS (kiểm tra bằng `psql`)
3) Flyway migration chạy thành công trên RDS

---

## 5.6.5. Schema dữ liệu (DBML)

Schema tuân thủ nguyên tắc microservice. Các quan hệ cross-service chỉ là logical.

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

> Đây không phải store chính của vị trí tài xế. Store chính sẽ là **Redis GEO**. Bảng này chỉ phục vụ matching local-first và debug.

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

### 5.6.5.7. Rating (Logical domain)

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

## 5.6.6. Quan hệ logic giữa các entity

Các quan hệ dưới đây **chỉ mang tính mô tả domain**, không tạo foreign key vật lý:

| Quan hệ | Mô tả |
|---|---|
| `drivers.user_id` → `users.id` | Tài xế gắn với user |
| `vehicles.driver_id` → `drivers.id` | Xe thuộc tài xế |
| `trips.passenger_id` → `users.id` | Hành khách |
| `trips.driver_id` → `drivers.id` | Tài xế được gán |
| `trips.vehicle_id` → `vehicles.id` | Xe được sử dụng |
| `trip_offers.trip_id` → `trips.id` | Offer thuộc chuyến |
| `trip_offers.driver_id` → `drivers.id` | Offer gửi tới tài xế |
| `payments.trip_id` → `trips.id` | 1-1 với chuyến |
| `ratings.trip_id` → `trips.id` | Đánh giá theo chuyến |
| `ratings.reviewer_user_id` → `users.id` | Người đánh giá |
| `ratings.target_user_id` → `users.id` | Người được đánh giá |

---

## 5.6.7. Kiểm tra sau khi triển khai

```bash
# Kiểm tra 6 RDS instance
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].[DBInstanceIdentifier,DBInstanceStatus,StorageEncrypted,MultiAZ,PubliclyAccessible,BackupRetentionPeriod]" \
  --output table

# Kiểm tra KMS key rotation
aws kms get-key-rotation-status \
  --key-id alias/ridenow-prod-rds \
  --query KeyRotationEnabled \
  --output text
```

**Kết quả mong đợi:**

- 6 dòng: `available`, `True` (encrypted), `True` (Multi-AZ), `False` (public), `30`
- Key rotation: `True`

---

## 5.6.8. Tóm tắt

| Hạng mục | Chi tiết |
|---|---|
| Số database (prod) | 6 (User, Driver, Trip, Payment, Matching, Notification) |
| Engine | PostgreSQL 17.x |
| Multi-AZ | Có (prod) |
| Encryption | Có (Customer Managed KMS) |
| Public access | Không |
| Backup | 7 ngày (stage) / 30 ngày (prod) |
| Schema ownership | Mỗi service tự quản lý |
| Cross-service FK | Không sử dụng (chỉ logical UUID) |
| Real-time location | Redis GEO (không lưu PostgreSQL) |
| Migration tool | Flyway (chạy y hệt trên container lẫn RDS) |

**Kết luận**:  
Cùng một Flyway script + JPA entity, chạy được cả trên Docker Postgres lẫn RDS. Chỉ cần đổi connection string.