---
title : "Các bước chuẩn bị"
date : 2024-01-01 
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

Trang này liệt kê những gì bạn cần chuẩn bị trước khi dựng RideNow trên AWS và thiết lập các biến mà mọi trang sau đều dùng.

## Công cụ và quyền truy cập

Bạn cần một tài khoản AWS và một danh tính quản trị không phải root (người dùng IAM hoặc vai trò IAM Identity Center có quyền quản trị). Không dùng người dùng root.

Cài các công cụ sau trên máy của bạn:

| Công cụ | Dùng để |
|---|---|
| AWS CLI v2 | Tra cứu giá trị tài nguyên và chạy các bước kiểm tra trên mỗi trang |
| docker | Build và đẩy image của các service lên ECR |
| kubectl | Làm việc với cụm EKS |
| helm | Cài AWS Load Balancer Controller và External Secrets |
| jq | Đọc kết quả JSON trong các bước kiểm tra |
| Bash | Chạy mọi khối lệnh (trên Windows, dùng WSL) |

## Kiểm tra danh tính

Chạy lệnh sau bằng thông tin xác thực quản trị của bạn:

```bash
aws sts get-caller-identity
```

Kết quả hiển thị tài khoản và người dùng hoặc vai trò của bạn. Các giá trị dưới đây là ví dụ đã che:

```text
{
    "UserId": "AIDAEXAMPLEEXAMPLE",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/workshop-admin"
}
```

Nếu lệnh lỗi hoặc hiển thị người dùng root, hãy sửa thông tin xác thực trước khi tiếp tục.

## Region và môi trường

Toàn bộ workshop dựng môi trường `prod` trong region `ap-southeast-1`. Hãy đặt `ap-southeast-1` làm region mặc định của profile AWS CLI và mở bảng điều khiển AWS (console) ở cùng region.

## Cảnh báo chi phí

Một số tài nguyên tính phí theo giờ ngay từ lúc tạo, dù có ai dùng hay không:

- control plane của EKS
- các NAT gateway
- các instance RDS Multi-AZ
- replication group của ElastiCache

Nếu có thể, hãy làm workshop trong một lần và kết thúc bằng 5.11 Dọn dẹp để ngừng tính phí.

## Quy ước

- **Auto_Variable.** Khi một lệnh CLI tra được giá trị (ID tài khoản, VPC ID, URL của queue), trang sẽ lưu giá trị đó vào biến Bash giúp bạn. Bạn không bao giờ phải sao chép các giá trị này bằng tay. Mỗi trang bắt đầu bằng một khối đặt các biến mà trang cần, theo sau là kết quả mong đợi với giá trị đã che.
- **Placeholder.** Một vài giá trị không lệnh nào tra được, ví dụ lựa chọn do chính bạn quyết định. Trang viết các giá trị này dưới dạng `<GITHUB_REPO>` và cho biết nơi sao chép giá trị. Hãy thay cả token, gồm cả dấu ngoặc nhọn, bằng giá trị của bạn.
- **Tệp ghi chú.** Hãy mở một tệp văn bản trong lúc làm. Ghi lại mọi giá trị bạn sao chép bằng tay cho Placeholder để dán lại ở các trang sau.
- **Chỉ dùng Bash.** Mọi lệnh dùng cú pháp Bash. Cú pháp PowerShell không được hỗ trợ. Trên Windows, hãy chạy lệnh trong Bash bên trong WSL.
- **Chỉ trong một phiên.** Biến chỉ tồn tại trong phiên shell hiện tại. Sau khi mở shell mới, hãy chạy lại khối đặt biến ở đầu trang bạn đang làm.

## Đặt các biến dùng chung

Mọi trang sau đều dùng các biến này. Chạy khối lệnh, rồi so sánh kết quả với kết quả mong đợi:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod
IMAGE_TAG=v1.0.0

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "IMAGE_TAG=$IMAGE_TAG"
```

Kết quả mong đợi (ID tài khoản đã che):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
IMAGE_TAG=v1.0.0
```

Nếu một dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh.