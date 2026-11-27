---
title : "Container Image"
date : 2024-01-01
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

Trang này tạo bảy repository ECR riêng tư chứa image của các service RideNow. Mỗi repository quét image khi push và có chính sách vòng đời giữ tối đa 50 image, đồng thời xoá image không có tag sau 14 ngày.

## Set variables

Chạy khối lệnh này trong phiên Bash của bạn, rồi so sánh kết quả với kết quả mong đợi:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
IMAGE_TAG=v1.0.0

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "IMAGE_TAG=$IMAGE_TAG"
```

Kết quả mong đợi (ID tài khoản đã che):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
IMAGE_TAG=v1.0.0
```

Nếu một dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh. Các biến này đã được định nghĩa ở 5.2, nên bạn có thể bỏ qua khối lệnh này nếu vẫn đang dùng phiên shell của 5.2.

## What it creates

| Logical ID trong template | Loại tài nguyên AWS | Tên repository |
|---|---|---|
| `UserServiceRepo` | `AWS::ECR::Repository` | `ridenow/user-service` |
| `DriverServiceRepo` | `AWS::ECR::Repository` | `ridenow/driver-service` |
| `TripServiceRepo` | `AWS::ECR::Repository` | `ridenow/trip-service` |
| `PaymentServiceRepo` | `AWS::ECR::Repository` | `ridenow/payment-service` |
| `MatchingServiceRepo` | `AWS::ECR::Repository` | `ridenow/matching-service` |
| `NotificationWorkerRepo` | `AWS::ECR::Repository` | `ridenow/notification-worker` |
| `RealtimeGatewayRepo` | `AWS::ECR::Repository` | `ridenow/realtime-gateway` |

Mọi repository dùng cùng một cấu hình:

| Thiết lập | Giá trị |
|---|---|
| Visibility | Private |
| Image tag mutability | Immutable |
| Mã hoá | KMS, khoá do AWS quản lý `aws/ecr` |
| Scan on push | Bật |
| Lifecycle rule 1 | Xoá image không có tag sau 14 ngày kể từ lúc push |
| Lifecycle rule 2 | Xoá image khi repository chứa hơn 50 image |

## Depends on

Không có. Bạn có thể tạo các repository bất kỳ lúc nào trước 5.9, nơi bạn build và push image.

## Create the resources

Mở bảng điều khiển AWS (console) ở region Asia Pacific (Singapore) `ap-southeast-1`. Mỗi repository dưới đây có một khối các bước đầy đủ riêng. Hãy làm lần lượt từng repository.

Nếu trang tạo repository không có tuỳ chọn **Scan on push**, hãy bật sau khi tạo xong các repository: vào **ECR -> Private registry -> Settings -> Scanning configuration**, chọn **Basic scanning**, rồi bật **Scan on push** cho bộ lọc `ridenow/*`.

### 1. `ridenow/user-service` (`UserServiceRepo`)

**Tạo repository**

1. Vào **ECR -> Private registry -> Repositories -> Create repository**.
2. Đặt **Visibility settings** là `Private`.
3. Ở **Repository name**, nhập `ridenow/user-service` (gõ dấu gạch chéo như một phần của tên).
4. Đặt **Image tag mutability** là `Immutable`.
5. Chọn **Customize encryption settings (KMS)** trong phần **Encryption settings**.
6. Đặt **Encryption key** là `AWS managed key (aws/ecr)`.
7. Trong **Image scan settings**, đặt **Scan on push** là `Enabled`.
8. Chọn **Create repository**.

**Lifecycle rule 1**

9. Mở `ridenow/user-service`, rồi chọn **Lifecycle Policy** ở menu bên trái.
10. Chọn **Create rule**.
11. Đặt **Rule priority** là `1`.
12. Đặt **Rule description** là `Expire untagged images`.
13. Đặt **Image status** là `Untagged`.
14. Trong **Match criteria**, chọn **Since image pushed** với **Days** là `14`.
15. Đặt **Rule action** là `expire`.
16. Chọn **Save**.

**Lifecycle rule 2**

17. Chọn **Create rule** lần nữa.
18. Đặt **Rule priority** là `2`.
19. Đặt **Rule description** là `Keep only the most recent tagged images`.
20. Đặt **Image status** là `Any`.
21. Trong **Match criteria**, chọn **Image count more than** với **Count** là `50`.
22. Đặt **Rule action** là `expire`.
23. Chọn **Save**.

### 2. `ridenow/driver-service` (`DriverServiceRepo`)

**Tạo repository**

1. Vào **ECR -> Private registry -> Repositories -> Create repository**.
2. Đặt **Visibility settings** là `Private`.
3. Ở **Repository name**, nhập `ridenow/driver-service` (gõ dấu gạch chéo như một phần của tên).
4. Đặt **Image tag mutability** là `Immutable`.
5. Chọn **Customize encryption settings (KMS)** trong phần **Encryption settings**.
6. Đặt **Encryption key** là `AWS managed key (aws/ecr)`.
7. Trong **Image scan settings**, đặt **Scan on push** là `Enabled`.
8. Chọn **Create repository**.

**Lifecycle rule 1**

9. Mở `ridenow/driver-service`, rồi chọn **Lifecycle Policy** ở menu bên trái.
10. Chọn **Create rule**.
11. Đặt **Rule priority** là `1`.
12. Đặt **Rule description** là `Expire untagged images`.
13. Đặt **Image status** là `Untagged`.
14. Trong **Match criteria**, chọn **Since image pushed** với **Days** là `14`.
15. Đặt **Rule action** là `expire`.
16. Chọn **Save**.

**Lifecycle rule 2**

17. Chọn **Create rule** lần nữa.
18. Đặt **Rule priority** là `2`.
19. Đặt **Rule description** là `Keep only the most recent tagged images`.
20. Đặt **Image status** là `Any`.
21. Trong **Match criteria**, chọn **Image count more than** với **Count** là `50`.
22. Đặt **Rule action** là `expire`.
23. Chọn **Save**.

### 3. `ridenow/trip-service` (`TripServiceRepo`)

**Tạo repository**

1. Vào **ECR -> Private registry -> Repositories -> Create repository**.
2. Đặt **Visibility settings** là `Private`.
3. Ở **Repository name**, nhập `ridenow/trip-service` (gõ dấu gạch chéo như một phần của tên).
4. Đặt **Image tag mutability** là `Immutable`.
5. Chọn **Customize encryption settings (KMS)** trong phần **Encryption settings**.
6. Đặt **Encryption key** là `AWS managed key (aws/ecr)`.
7. Trong **Image scan settings**, đặt **Scan on push** là `Enabled`.
8. Chọn **Create repository**.

**Lifecycle rule 1**

9. Mở `ridenow/trip-service`, rồi chọn **Lifecycle Policy** ở menu bên trái.
10. Chọn **Create rule**.
11. Đặt **Rule priority** là `1`.
12. Đặt **Rule description** là `Expire untagged images`.
13. Đặt **Image status** là `Untagged`.
14. Trong **Match criteria**, chọn **Since image pushed** với **Days** là `14`.
15. Đặt **Rule action** là `expire`.
16. Chọn **Save**.

**Lifecycle rule 2**

17. Chọn **Create rule** lần nữa.
18. Đặt **Rule priority** là `2`.
19. Đặt **Rule description** là `Keep only the most recent tagged images`.
20. Đặt **Image status** là `Any`.
21. Trong **Match criteria**, chọn **Image count more than** với **Count** là `50`.
22. Đặt **Rule action** là `expire`.
23. Chọn **Save**.

### 4. `ridenow/payment-service` (`PaymentServiceRepo`)

**Tạo repository**

1. Vào **ECR -> Private registry -> Repositories -> Create repository**.
2. Đặt **Visibility settings** là `Private`.
3. Ở **Repository name**, nhập `ridenow/payment-service` (gõ dấu gạch chéo như một phần của tên).
4. Đặt **Image tag mutability** là `Immutable`.
5. Chọn **Customize encryption settings (KMS)** trong phần **Encryption settings**.
6. Đặt **Encryption key** là `AWS managed key (aws/ecr)`.
7. Trong **Image scan settings**, đặt **Scan on push** là `Enabled`.
8. Chọn **Create repository**.

**Lifecycle rule 1**

9. Mở `ridenow/payment-service`, rồi chọn **Lifecycle Policy** ở menu bên trái.
10. Chọn **Create rule**.
11. Đặt **Rule priority** là `1`.
12. Đặt **Rule description** là `Expire untagged images`.
13. Đặt **Image status** là `Untagged`.
14. Trong **Match criteria**, chọn **Since image pushed** với **Days** là `14`.
15. Đặt **Rule action** là `expire`.
16. Chọn **Save**.

**Lifecycle rule 2**

17. Chọn **Create rule** lần nữa.
18. Đặt **Rule priority** là `2`.
19. Đặt **Rule description** là `Keep only the most recent tagged images`.
20. Đặt **Image status** là `Any`.
21. Trong **Match criteria**, chọn **Image count more than** với **Count** là `50`.
22. Đặt **Rule action** là `expire`.
23. Chọn **Save**.

### 5. `ridenow/matching-service` (`MatchingServiceRepo`)

**Tạo repository**

1. Vào **ECR -> Private registry -> Repositories -> Create repository**.
2. Đặt **Visibility settings** là `Private`.
3. Ở **Repository name**, nhập `ridenow/matching-service` (gõ dấu gạch chéo như một phần của tên).
4. Đặt **Image tag mutability** là `Immutable`.
5. Chọn **Customize encryption settings (KMS)** trong phần **Encryption settings**.
6. Đặt **Encryption key** là `AWS managed key (aws/ecr)`.
7. Trong **Image scan settings**, đặt **Scan on push** là `Enabled`.
8. Chọn **Create repository**.

**Lifecycle rule 1**

9. Mở `ridenow/matching-service`, rồi chọn **Lifecycle Policy** ở menu bên trái.
10. Chọn **Create rule**.
11. Đặt **Rule priority** là `1`.
12. Đặt **Rule description** là `Expire untagged images`.
13. Đặt **Image status** là `Untagged`.
14. Trong **Match criteria**, chọn **Since image pushed** với **Days** là `14`.
15. Đặt **Rule action** là `expire`.
16. Chọn **Save**.

**Lifecycle rule 2**

17. Chọn **Create rule** lần nữa.
18. Đặt **Rule priority** là `2`.
19. Đặt **Rule description** là `Keep only the most recent tagged images`.
20. Đặt **Image status** là `Any`.
21. Trong **Match criteria**, chọn **Image count more than** với **Count** là `50`.
22. Đặt **Rule action** là `expire`.
23. Chọn **Save**.

### 6. `ridenow/notification-worker` (`NotificationWorkerRepo`)

**Tạo repository**

1. Vào **ECR -> Private registry -> Repositories -> Create repository**.
2. Đặt **Visibility settings** là `Private`.
3. Ở **Repository name**, nhập `ridenow/notification-worker` (gõ dấu gạch chéo như một phần của tên).
4. Đặt **Image tag mutability** là `Immutable`.
5. Chọn **Customize encryption settings (KMS)** trong phần **Encryption settings**.
6. Đặt **Encryption key** là `AWS managed key (aws/ecr)`.
7. Trong **Image scan settings**, đặt **Scan on push** là `Enabled`.
8. Chọn **Create repository**.

**Lifecycle rule 1**

9. Mở `ridenow/notification-worker`, rồi chọn **Lifecycle Policy** ở menu bên trái.
10. Chọn **Create rule**.
11. Đặt **Rule priority** là `1`.
12. Đặt **Rule description** là `Expire untagged images`.
13. Đặt **Image status** là `Untagged`.
14. Trong **Match criteria**, chọn **Since image pushed** với **Days** là `14`.
15. Đặt **Rule action** là `expire`.
16. Chọn **Save**.

**Lifecycle rule 2**

17. Chọn **Create rule** lần nữa.
18. Đặt **Rule priority** là `2`.
19. Đặt **Rule description** là `Keep only the most recent tagged images`.
20. Đặt **Image status** là `Any`.
21. Trong **Match criteria**, chọn **Image count more than** với **Count** là `50`.
22. Đặt **Rule action** là `expire`.
23. Chọn **Save**.

### 7. `ridenow/realtime-gateway` (`RealtimeGatewayRepo`)

**Tạo repository**

1. Vào **ECR -> Private registry -> Repositories -> Create repository**.
2. Đặt **Visibility settings** là `Private`.
3. Ở **Repository name**, nhập `ridenow/realtime-gateway` (gõ dấu gạch chéo như một phần của tên).
4. Đặt **Image tag mutability** là `Immutable`.
5. Chọn **Customize encryption settings (KMS)** trong phần **Encryption settings**.
6. Đặt **Encryption key** là `AWS managed key (aws/ecr)`.
7. Trong **Image scan settings**, đặt **Scan on push** là `Enabled`.
8. Chọn **Create repository**.

**Lifecycle rule 1**

9. Mở `ridenow/realtime-gateway`, rồi chọn **Lifecycle Policy** ở menu bên trái.
10. Chọn **Create rule**.
11. Đặt **Rule priority** là `1`.
12. Đặt **Rule description** là `Expire untagged images`.
13. Đặt **Image status** là `Untagged`.
14. Trong **Match criteria**, chọn **Since image pushed** với **Days** là `14`.
15. Đặt **Rule action** là `expire`.
16. Chọn **Save**.

**Lifecycle rule 2**

17. Chọn **Create rule** lần nữa.
18. Đặt **Rule priority** là `2`.
19. Đặt **Rule description** là `Keep only the most recent tagged images`.
20. Đặt **Image status** là `Any`.
21. Trong **Match criteria**, chọn **Image count more than** với **Count** là `50`.
22. Đặt **Rule action** là `expire`.
23. Chọn **Save**.

## Values to record

Bạn không cần sao chép giá trị nào bằng tay. Các trang sau tự dựng địa chỉ registry từ `ACCOUNT_ID` và `AWS_REGION`. URI của mỗi repository có dạng `<account-id>.dkr.ecr.<region>.amazonaws.com/<tên repository>`. Để liệt kê các URI, vào **ECR -> Private registry -> Repositories** và đọc cột **URI**, hoặc chạy:

```bash
aws ecr describe-repositories \
  --query "repositories[?starts_with(repositoryName,'ridenow/')].[repositoryName,repositoryUri]" \
  --output table
```

| Repository | Dùng cho |
|---|---|
| `ridenow/user-service` | Push image bên dưới và workload user-service ở 5.9 |
| `ridenow/driver-service` | Push image bên dưới và workload driver-service ở 5.9 |
| `ridenow/trip-service` | Push image bên dưới và workload trip-service ở 5.9 |
| `ridenow/payment-service` | Push image bên dưới và workload payment-service ở 5.9 |
| `ridenow/matching-service` | Push image bên dưới và workload matching-service ở 5.9 |
| `ridenow/notification-worker` | Push image bên dưới và workload notification-worker ở 5.9 |
| `ridenow/realtime-gateway` | Push image bên dưới và workload realtime-gateway ở 5.9 |

## Post-deploy steps

Build bảy image của service và push lên các repository. Chạy khối lệnh này từ thư mục gốc của repository RideNow, khi Docker đang chạy và các biến ở khối lệnh đầu tiên đã được đặt:

```bash
REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$REGISTRY"
for SVC in user-service driver-service trip-service payment-service \
           matching-service notification-worker realtime-gateway; do
  docker buildx build --platform linux/amd64 \
    -t "$REGISTRY/ridenow/$SVC:$IMAGE_TAG" \
    --push "./$SVC"
done
```

Kết quả mong đợi: `Login Succeeded`, sau đó là bảy lần push thành công. Tag là immutable, vì vậy muốn push lại, hãy đặt `IMAGE_TAG` mới trước.

## Verify

Kiểm tra thiết lập quét, mã hoá và tag của cả bảy repository:

```bash
aws ecr describe-repositories \
  --query "repositories[?starts_with(repositoryName,'ridenow/')].[repositoryName,imageScanningConfiguration.scanOnPush,encryptionConfiguration.encryptionType,imageTagMutability]" \
  --output table
```

Kết quả mong đợi: bảy dòng, mỗi dòng có `True`, `KMS` và `IMMUTABLE`. Nếu một dòng khác, hãy xoá repository đó khi nó còn trống rồi tạo lại, vì thiết lập mã hoá không đổi được sau khi tạo.

Kiểm tra lifecycle rule của cả bảy repository:

```bash
for REPO in user-service driver-service trip-service payment-service \
            matching-service notification-worker realtime-gateway; do
  echo "ridenow/$REPO"
  aws ecr get-lifecycle-policy --repository-name "ridenow/$REPO" \
    --query lifecyclePolicyText --output text \
    | jq -c '.rules[] | {rulePriority, countNumber: .selection.countNumber}'
done
```

Kết quả mong đợi: với mỗi repository, rule priority `1` có `countNumber` là `14` và rule priority `2` có `countNumber` là `50`:

```text
ridenow/user-service
{"rulePriority":1,"countNumber":14}
{"rulePriority":2,"countNumber":50}
```

Trang tiếp theo là 5.6 Dữ liệu.
