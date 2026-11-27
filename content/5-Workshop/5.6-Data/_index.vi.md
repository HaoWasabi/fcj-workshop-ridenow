---
title : "Database"
date : 2024-01-01
weight : 6
chapter : false
pre : " <b> 5.6. </b> "
---

Trang này tạo tầng dữ liệu của môi trường `prod`: một khoá KMS, một DB subnet group và sáu instance PostgreSQL được mã hoá trên Amazon RDS, mỗi service một instance. RDS tự tạo và lưu mật khẩu master trong AWS Secrets Manager, nên bạn không phải gõ hay lưu mật khẩu. Mỗi service sở hữu database riêng và không database nào tham chiếu đến database khác.

## Set variables

Trước khi thao tác, hãy mở một phiên shell mới để chạy khối lệnh kiểm tra các tài nguyên nền tảng từ mục 5.2 và các biến mạng từ mục 5.4, mỗi biến có một lệnh con kiểm tra riêng. Cơ chế kiểm tra `STOP` giúp phát hiện sớm biến thiếu hoặc sai cấu hình trước khi tạo tài nguyên.

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

Kết quả mong đợi (ID tài khoản và ID tài nguyên đã che):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
DB_SG_ID=sg-0fff...
```

Nếu có dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh.

Các biến dữ liệu tra cứu những tài nguyên bạn tạo ở trang này. Hãy chạy khối thứ hai sau khi hoàn tất phần Create the resources, không chạy trước. Mỗi lệnh tra cứu tìm tài nguyên theo tên.

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

Kết quả mong đợi (ID tài khoản đã che và các giá trị được rút gọn):

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

Nếu có dòng bắt đầu bằng `STOP`, tiếp tục vòng lặp sửa lỗi - chạy lại khối lệnh.

## What it creates

| Logical ID trong template | Loại tài nguyên AWS | Tên |
|---|---|---|
| `DbKmsKey` | KMS key (symmetric) | Không có tên; alias bên dưới trỏ đến khoá này |
| `DbKmsAlias` | KMS alias | `alias/ridenow-prod-rds` |
| `DbSubnetGroup` | RDS DB subnet group | `ridenow-prod-db-subnet-group` |
| `UserDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-user-db` |
| `DriverDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-driver-db` |
| `TripDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-trip-db` |
| `PaymentDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-payment-db` |
| `MatchingDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-matching-db` |
| `NotificationDb` | RDS DB instance (PostgreSQL) | `ridenow-prod-notification-db` |
| Secret master do RDS quản lý (mỗi instance một secret) | `AWS::SecretsManager::Secret` | RDS tạo trong Secrets Manager, mã hoá bằng `alias/ridenow-prod-rds` |

#### Bảng cấu hình chung mọi instance:

| Thiết lập | Giá trị |
|---|---|
| Engine | PostgreSQL, bản `17.x` mới nhất |
| Deployment | Multi-AZ DB instance |
| Instance class | `db.t4g.micro` |
| Storage | gp3, 20 GiB, không bật storage autoscaling |
| Public access | Không |
| Mã hoá | Có, dùng `alias/ridenow-prod-rds` |
| Credentials | Managed in AWS Secrets Manager |
| Backup retention | 30 ngày |
| Deletion protection | Tắt, và `DeletionPolicy` giữ lại một snapshot cuối |
| VPC | `ridenow-prod-vpc` |
| DB subnet group | `ridenow-prod-db-subnet-group` |
| Security group | `ridenow-prod-db-sg` |

#### Bảng thông tin riêng mỗi instance:

| Service | DB instance identifier | Master username | Tên database khởi tạo |
|---|---|---|---|
| User | `ridenow-prod-user-db` | `app_user` | `user_service` |
| Driver | `ridenow-prod-driver-db` | `driver` | `driver_service` |
| Trip | `ridenow-prod-trip-db` | `trip` | `trip_service` |
| Payment | `ridenow-prod-payment-db` | `payment` | `payment_service` |
| Matching | `ridenow-prod-matching-db` | `matching` | `matching_service` |
| Notification | `ridenow-prod-notification-db` | `notification` | `notification_service` |

Các database theo nguyên tắc mỗi service một database: mỗi service chỉ đọc và ghi database của mình, và không có foreign key vật lý nào đi qua ranh giới giữa các service. Vị trí tài xế theo thời gian thực không được lưu trong PostgreSQL.

## Depends on

Chỉ phụ thuộc mục 5.4. Chọn các tài nguyên này theo tên trong console:

| Cần từ 5.4 | Chọn trong console | Biến |
|---|---|---|
| Private subnet | `ridenow-prod-private-1`, `ridenow-prod-private-2` | `PRIVATE_SUBNET_IDS` |
| Security group của database | `ridenow-prod-db-sg` trong `ridenow-prod-vpc` | `DB_SG_ID` |

## Create the resources

Mở bảng điều khiển AWS (console) ở region Asia Pacific (Singapore) `ap-southeast-1`. Làm ba bước theo thứ tự.

### 1. KMS key (`DbKmsKey`, `DbKmsAlias`)

1. Vào **KMS -> Customer managed keys -> Create key**.
2. Đặt **Key type** là `Symmetric`.
3. Đặt **Key usage** là `Encrypt and decrypt`.
4. Trong **Advanced options**, đặt **Key material origin** là `KMS`.
5. Trong **Advanced options**, đặt **Regionality** là `Single-Region key`.
6. Chọn **Next**.
7. Đặt **Alias** là `ridenow-prod-rds`. Console hiển thị là `alias/ridenow-prod-rds`.
8. Đặt **Description** là `ridenow-prod RDS encryption key`.
9. Chọn **Next**.
10. Trong **Key administrators**, chọn IAM user hoặc role của chính bạn.
11. Giữ nguyên **Allow key administrators to delete this key** đã chọn, rồi chọn **Next**.
12. Trong **Key users**, không chọn gì, rồi chọn **Next**.
13. Ở **Review**, chọn **Finish**.
14. Mở khoá vừa tạo và vào **Key policy -> Edit**.
15. Chạy lệnh bên dưới trong shell, sao chép JSON được in ra và dán đè lên toàn bộ policy.

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

16. Chọn **Save changes**.
17. Vào tab **Key rotation** và chọn **Edit**.
18. Chọn **Enable automatic key rotation**, rồi chọn **Save**.

Lệnh in ra policy với ID tài khoản của chính bạn. Wizard tự ghi một policy mặc định, nên phần thay thế này là phần khớp với template.

### 2. DB subnet group (`DbSubnetGroup`)

1. Vào **RDS -> Subnet groups -> Create DB subnet group**.
2. Đặt **Name** là `ridenow-prod-db-subnet-group`.
3. Đặt **Description** là `ridenow-prod RDS private subnets`.
4. Đặt **VPC** là `ridenow-prod-vpc`.
5. Trong **Availability Zones**, chọn hai zone của `ridenow-prod-private-1` và `ridenow-prod-private-2`.
6. Trong **Subnets**, chọn hai private subnet. Kiểm tra CIDR trong **VPC -> Subnets** và không chọn `ridenow-prod-public-*`.
7. Chọn **Create**.

### 3. Sáu RDS instance

Tạo sáu database lần lượt từng cái. Mỗi database có một đề mục riêng bên dưới với đầy đủ giá trị. Mỗi instance mất 10 đến 20 phút để tạo, nên hãy bắt đầu instance tiếp theo khi instance trước đang ở trạng thái Creating.

#### Quy trình thực hiện

Lần lượt khởi tạo các database *user_service, driver_service, trip_service, payment_service, matching_service, notification_service.* theo quy trình.

1. **Khởi tạo & Engine:** Vào RDS -> Databases -> Create database -> Standard create. Chọn PostgreSQL bản 17.x mới nhất, template Dev/Test.

2. **Availability & Instance:** Chọn `Multi-AZ DB instance` (không chọn DB cluster); instance class `Burstable classes` -> `db.t4g.micro.`

3. **Storage:** Chọn `General Purpose SSD (gp3)`, 20 GiB, tắt storage autoscaling.

4. **Connectivity:** Không gắn EC2, mạng IPv4, VPC: `ridenow-prod-vpc`, DB subnet group: `ridenow-prod-db-subnet-group`, Public access: `No`. Chọn SG `ridenow-prod-db-sg`, xoá SG `default`, Availability Zone: `No preference`.

5. **Authentication & Monitoring:** Chọn `Password authentication`; tắt `Performance Insights` và `Enhanced monitoring`.

6. **Additional configuration & Tạo:**
    - **Đặt Initial database name** theo bảng trên (ví dụ: user_service).
    - **Settings:** chọn Managed in AWS Secrets Manager, secret encryption key `alias/ridenow-prod-rds`.
    - **Backup & Encryption:** bật automated backups (30 ngày), bật encryption với KMS key `alias/ridenow-prod-rds`, giữ Enable auto minor version upgrade, tắt deletion protection.
    - Nhấn **Create database.**

> Với **NotificationDb:** Giữ mặc định cho mọi mục không được liệt kê. Khi xoá instance sau này, hãy giữ **Create final snapshot** được chọn.

## Values to record

Hãy chạy khối thứ hai của phần Set variables ngay bây giờ. Các giá trị dưới đây được tra cứu tự động, nên bạn không phải sao chép gì bằng tay. Các trang sau lặp lại cùng các lệnh tra cứu ở đầu trang.

| Biến | Giá trị | Dùng ở |
|---|---|---|
| `RDS_KMS_KEY_ARN` | ARN của khoá phía sau `alias/ridenow-prod-rds` | 5.8 (policy cho phép External Secrets giải mã secret database) |
| `USER_DB_ENDPOINT`, `DRIVER_DB_ENDPOINT`, `TRIP_DB_ENDPOINT`, `PAYMENT_DB_ENDPOINT`, `MATCHING_DB_ENDPOINT`, `NOTIFICATION_DB_ENDPOINT` | Tên host của từng instance | 5.9 (kết nối database của từng workload) |
| `USER_DB_SECRET_ARN`, `DRIVER_DB_SECRET_ARN`, `TRIP_DB_SECRET_ARN`, `PAYMENT_DB_SECRET_ARN`, `MATCHING_DB_SECRET_ARN`, `NOTIFICATION_DB_SECRET_ARN` | ARN của secret master do RDS quản lý của từng instance | 5.8 (secret database do External Secrets đọc) |

Cả mười ba giá trị đều có lệnh tra cứu, nên không giá trị nào là Manual_Value. Console hiển thị cùng dữ liệu ở **RDS -> Databases -> instance -> Connectivity & security** (endpoint) và **Configuration** (master credentials ARN).

## Post-deploy steps

Trang này không cần bước nào sau khi triển khai. Rule inbound cho phép cluster truy cập database qua cổng 5432 được tạo ở 5.9.

## Verify

Chạy các lệnh sau trong cùng shell:

```bash
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].[DBInstanceIdentifier,DBInstanceStatus,StorageEncrypted,MultiAZ,PubliclyAccessible,BackupRetentionPeriod]" \
  --output table
aws kms get-key-rotation-status --key-id "alias/${CLUSTER_NAME}-rds" --query KeyRotationEnabled --output text
```

Kết quả mong đợi: sáu dòng, mỗi dòng có `available`, `True`, `True`, `False` và `30`; sau đó là `True` cho key rotation. Nếu thiếu dòng hoặc trạng thái chưa là `available`, hãy mở **RDS -> Databases -> instance đó -> Logs & events**.

```bash
aws rds describe-db-instances \
  --query "DBInstances[?starts_with(DBInstanceIdentifier,'ridenow-prod-')].VpcSecurityGroups[].VpcSecurityGroupId" \
  --output text | tr '\t' '\n' | sort -u
echo "DB_SG_ID=$DB_SG_ID"
aws secretsmanager describe-secret --secret-id "$USER_DB_SECRET_ARN" --query KmsKeyId --output text
```

Kết quả mong đợi: dòng đầu là ID security group trùng với `DB_SG_ID` (chỉ một dòng), và dòng cuối là key ARN của `alias/ridenow-prod-rds`:

```text
sg-0fff...
DB_SG_ID=sg-0fff...
arn:aws:kms:ap-southeast-1:123456789012:key/1234abcd-...
```

