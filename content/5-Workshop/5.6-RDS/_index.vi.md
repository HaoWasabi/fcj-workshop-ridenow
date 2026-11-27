---
title : "RDS: Quản lý PostgreSQL"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

## 5.6.1. Mục tiêu

Chương này hướng dẫn **khởi tạo và triển khai** tầng dữ liệu của hệ thống RideNow trên Amazon RDS (PostgreSQL). Sau khi hoàn thành, hệ thống sẽ có:

- 1 KMS Key dùng để mã hóa
- 1 DB Subnet Group
- 6 RDS PostgreSQL instance (mỗi service một database riêng)
- Schema đã sẵn sàng cho Flyway migration

**Vùng:** `ap-southeast-1`  
**Môi trường:** `prod`

---

## 5.6.2. Vấn đề cần giải quyết

Self-managed Postgres chạy trong container **thiếu**:

| Tính năng | Self-managed | Amazon RDS |
|---|---|---|
| Automatic backup | Phải tự cấu hình | Có sẵn |
| Multi-AZ Failover | Phải tự dựng | Có sẵn |
| Automated patching | Thủ công | Tự động |
| Monitoring tích hợp | Phải gắn thêm | CloudWatch sẵn |
| Encryption at rest | Phải tự quản lý key | KMS tích hợp |

RDS dùng **cùng PostgreSQL engine** → Flyway, JPA, SQL chạy y hệt, chỉ cần đổi connection string.

---

## 5.6.3. Phụ thuộc

Chỉ phụ thuộc vào tầng Network (`02-network`). Cần có sẵn:

| Resource | Tên |
|---|---|
| VPC | `ridenow-prod-vpc` |
| Private Subnet 1 | `ridenow-prod-private-1` |
| Private Subnet 2 | `ridenow-prod-private-2` |
| Security Group | `ridenow-prod-db-sg` |

> Nếu chưa có, hoàn thành chương Network trước khi tiếp tục.

---

## 5.6.4. Các resource sẽ được tạo

| Logical ID | Loại AWS | Tên thực tế |
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

## 5.6.5. Bước 1: Tạo KMS Key

Mục đích: Mã hóa dữ liệu at-rest cho tất cả RDS instance và Secrets Manager.

### Các bước trên Console

1. Vào **KMS → Customer managed keys → Create key**
2. **Key type**: `Symmetric`
3. **Key usage**: `Encrypt and decrypt`
4. **Key material origin**: `KMS`
5. **Regionality**: `Single-Region key`
6. **Alias**: `ridenow-prod-rds`  
   (Console sẽ hiển thị `alias/ridenow-prod-rds`)
7. **Description**: `ridenow-prod RDS encryption key`
8. **Key administrators**: chọn IAM user/role của bạn
9. Để nguyên **Allow key administrators to delete this key**
10. **Key users**: để trống
11. **Finish**

### Cập nhật Key Policy

Sau khi tạo xong:
1. Mở key vừa tạo → **Key policy → Edit**
2. Thay toàn bộ policy bằng nội dung sau (thay `<ACCOUNT_ID>` bằng 12-digit account id của bạn):

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

### Bật Key Rotation

1. Tab **Key rotation → Edit**
2. Tick **Enable automatic key rotation**
3. **Save**

---

## 5.6.6. Bước 2: Tạo DB Subnet Group

Mục đích: Chỉ định RDS chạy trong 2 private subnet (Multi-AZ).

### Các bước trên Console

1. Vào **RDS → Subnet groups → Create DB subnet group**
2. **Name**: `ridenow-prod-db-subnet-group`
3. **Description**: `ridenow-prod RDS private subnets`
4. **VPC**: `ridenow-prod-vpc`
5. **Availability Zones**: chọn 2 AZ tương ứng với private subnet
6. **Subnets**: tick `ridenow-prod-private-1` và `ridenow-prod-private-2`  
   (kiểm tra CIDR, **không** chọn public subnet)
7. **Create**

---

## 5.6.7. Bước 3: Tạo RDS Instance

Tạo lần lượt 6 database. Mỗi instance mất khoảng 10–20 phút → có thể tạo instance tiếp theo trong lúc instance trước đang `Creating`.

### Cấu hình chung cho tất cả instance

| Thuộc tính | Giá trị |
|---|---|
| Engine type | PostgreSQL |
| Engine version | Latest `17.x` |
| Templates | Dev/Test |
| Deployment options | **Multi-AZ DB instance** (không chọn Multi-AZ DB cluster) |
| Instance class | Burstable classes → `db.t4g.micro` |
| Storage type | General Purpose SSD (gp3) |
| Allocated storage | 20 GiB |
| Storage autoscaling | **Tắt** |
| Compute resource | Don't connect to an EC2 compute resource |
| Network type | IPv4 |
| VPC | `ridenow-prod-vpc` |
| DB subnet group | `ridenow-prod-db-subnet-group` |
| Public access | **No** |
| VPC security group | Choose existing → `ridenow-prod-db-sg` (xóa `default`) |
| Availability Zone | No preference |
| Database authentication | Password authentication |
| Performance Insights | **Tắt** |
| Enhanced monitoring | **Tắt** |
| Automated backups | **Bật** |
| Backup retention | 30 days |
| Encryption | **Bật** → KMS key = `alias/ridenow-prod-rds` |
| Auto minor version upgrade | Bật |
| Deletion protection | **Tắt** |
| Credentials management | **Managed in AWS Secrets Manager** |
| Encryption key (cho secret) | `alias/ridenow-prod-rds` |

### Chi tiết từng instance

| # | Template ID | DB instance identifier | Master username | Initial database name |
|---|---|---|---|---|
| 1 | `UserDb` | `ridenow-prod-user-db` | `app_user` | `user_service` |
| 2 | `DriverDb` | `ridenow-prod-driver-db` | `driver` | `driver_service` |
| 3 | `TripDb` | `ridenow-prod-trip-db` | `trip` | `trip_service` |
| 4 | `PaymentDb` | `ridenow-prod-payment-db` | `payment` | `payment_service` |
| 5 | `MatchingDb` | `ridenow-prod-matching-db` | `matching` | `matching_service` |
| 6 | `NotificationDb` | `ridenow-prod-notification-db` | `notification` | `notification_service` |

### Quy trình tạo từng instance (lặp lại 6 lần)

1. **RDS → Databases → Create database**
2. Chọn **Standard create**
3. Điền các giá trị theo bảng cấu hình chung + bảng chi tiết ở trên
4. **Create database**

> **Lưu ý**: Khi xóa instance sau này, giữ nguyên tùy chọn **Create final snapshot** (tương ứng `DeletionPolicy: Snapshot` trong template).

---

## 5.6.8. Giá trị cần ghi nhận sau khi tạo

Sau khi tất cả instance chuyển sang trạng thái `available`, ghi lại các giá trị sau (Discovery_Block sẽ tự load nếu dùng script):

| Output | Nơi lấy trên Console | Biến shell |
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

Các giá trị này sẽ được dùng ở chương **07 – EKS Add-ons**.

---

## 5.6.9. Kiểm tra

Chạy các lệnh sau:

```bash
# 1. Kiểm tra 6 RDS instance
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].[DBInstanceIdentifier,DBInstanceStatus,StorageEncrypted,MultiAZ,PubliclyAccessible,BackupRetentionPeriod]" \
  --output table

# 2. Kiểm tra KMS key rotation
aws kms get-key-rotation-status \
  --key-id alias/ridenow-prod-rds \
  --query KeyRotationEnabled \
  --output text
```

### Kết quả mong đợi

- 6 dòng, mỗi dòng:
  - Status = `available`
  - StorageEncrypted = `True`
  - MultiAZ = `True`
  - PubliclyAccessible = `False`
  - BackupRetentionPeriod = `30`
- Key rotation = `True`

Nếu thiếu instance hoặc status khác `available` → mở **RDS → Databases → instance → Logs & events** để kiểm tra.

---

## 5.6.10. Schema dữ liệu

Sau khi RDS sẵn sàng, mỗi service sẽ dùng **Flyway** để chạy migration tạo schema.

### Nguyên tắc quan trọng

- Mỗi service chỉ sở hữu schema của mình.
- Cross-service reference **chỉ lưu UUID** (không tạo foreign key vật lý).
- Vị trí tài xế real-time **không** lưu trong PostgreSQL → dùng Redis GEO.

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

> Bảng này chỉ là working copy. Store chính của vị trí tài xế là Redis GEO.

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

### 5.6.10.6. Notification (`notification_service`)

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

### 5.6.10.7. Rating (logical domain)

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

- [ ] KMS Key `alias/ridenow-prod-rds` đã tạo và bật rotation
- [ ] DB Subnet Group `ridenow-prod-db-subnet-group` đã tạo với 2 private subnet
- [ ] 6 RDS instance đều ở trạng thái `available`
- [ ] Tất cả instance: Encrypted = True, Multi-AZ = True, Public = False
- [ ] Backup retention = 30 days
- [ ] Đã ghi nhận 12 giá trị Endpoint + SecretArn
- [ ] Lệnh verify trả về kết quả đúng

---

## 5.6.12. Tóm tắt

| Hạng mục | Giá trị |
|---|---|
| Số RDS instance | 6 |
| Engine | PostgreSQL 17.x |
| Multi-AZ | Có |
| Encryption | Customer Managed KMS |
| Public access | Không |
| Backup | 30 ngày |
| Credentials | AWS Secrets Manager |
| Schema ownership | Mỗi service một database |
| Cross-service FK | Không (chỉ logical UUID) |

**Bài học chính:**  
RDS cung cấp managed PostgreSQL với backup, failover và encryption sẵn. Application code (Flyway + JPA) **không cần thay đổi**, chỉ cần trỏ connection string sang endpoint mới.
