---
title : "Network"
date : 2024-01-01 
weight : 4 
chapter : false
pre : " <b> 5.4. </b> "
---

Trang này tạo mạng của môi trường `prod`: một VPC với hai public subnet và hai private subnet, một internet gateway, hai NAT gateway và hai security group. Mọi trang sau (dữ liệu, nhắn tin, EKS) đều đặt tài nguyên của chúng trong mạng này.

## Set variables

Chạy khối đầu tiên trong một shell mới. Khối này lặp lại các biến dùng chung từ mục 5.2:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."
```

Năm biến mạng tra cứu các tài nguyên mà bạn tạo ngay trên trang này. Hãy chạy khối thứ hai sau khi hoàn thành phần Create the resources, không chạy trước. Mỗi lệnh tra cứu tìm tài nguyên theo tag `Name` của nó.

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)
[ -n "$VPC_ID" ] && [ "$VPC_ID" != "None" ] || echo "STOP: VPC_ID is empty. Create the VPC first."

PUBLIC_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-public-*" --query 'Subnets[].SubnetId' --output text)
[ -n "$PUBLIC_SUBNET_IDS" ] || echo "STOP: PUBLIC_SUBNET_IDS is empty. Create the public subnets first."

PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-private-*" --query 'Subnets[].SubnetId' --output text)
[ -n "$PRIVATE_SUBNET_IDS" ] || echo "STOP: PRIVATE_SUBNET_IDS is empty. Create the private subnets first."

CLUSTER_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-cluster-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$CLUSTER_SG_ID" ] && [ "$CLUSTER_SG_ID" != "None" ] || echo "STOP: CLUSTER_SG_ID is empty. Create the cluster security group first."

DB_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-db-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$DB_SG_ID" ] && [ "$DB_SG_ID" != "None" ] || echo "STOP: DB_SG_ID is empty. Create the database security group first."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "VPC_ID=$VPC_ID"
echo "PUBLIC_SUBNET_IDS=$PUBLIC_SUBNET_IDS"
echo "PRIVATE_SUBNET_IDS=$PRIVATE_SUBNET_IDS"
echo "CLUSTER_SG_ID=$CLUSTER_SG_ID"
echo "DB_SG_ID=$DB_SG_ID"
```

Kết quả mong đợi (các ID đã được che, hai danh sách subnet mỗi danh sách có hai ID):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
PUBLIC_SUBNET_IDS=subnet-0aaa... subnet-0bbb...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
CLUSTER_SG_ID=sg-0eee...
DB_SG_ID=sg-0fff...
```

Nếu có dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh.

## What it creates

| Tài nguyên | Loại tài nguyên AWS | Tên |
|---|---|---|
| VPC | VPC | `ridenow-prod-vpc` |
| Internet gateway | Internet gateway | `ridenow-prod-igw` |
| Gắn gateway | Internet gateway attachment | `ridenow-prod-igw` gắn vào `ridenow-prod-vpc` |
| Public subnet 1 | Subnet | `ridenow-prod-public-1` |
| Public subnet 2 | Subnet | `ridenow-prod-public-2` |
| Private subnet 1 | Subnet | `ridenow-prod-private-1` |
| Private subnet 2 | Subnet | `ridenow-prod-private-2` |
| Public route table | Route table | `ridenow-prod-public-rt` |
| Route mặc định public | Route | `0.0.0.0/0` tới `ridenow-prod-igw` |
| Liên kết subnet public | Route table association | `ridenow-prod-public-1` và `ridenow-prod-public-2` với `ridenow-prod-public-rt` |
| Elastic IP 1 | Elastic IP | `ridenow-prod-nat-eip-1` |
| NAT gateway 1 | NAT gateway | `ridenow-prod-nat-1` |
| Elastic IP 2 | Elastic IP | `ridenow-prod-nat-eip-2` |
| NAT gateway 2 | NAT gateway | `ridenow-prod-nat-2` |
| Private route table 1 | Route table | `ridenow-prod-private-rt-1` |
| Route mặc định private 1 | Route | `0.0.0.0/0` tới `ridenow-prod-nat-1` |
| Private route table 2 | Route table | `ridenow-prod-private-rt-2` |
| Route mặc định private 2 | Route | `0.0.0.0/0` tới `ridenow-prod-nat-2` |
| Security group của cluster | Security group | `ridenow-prod-cluster-sg` |
| Security group của cơ sở dữ liệu | Security group | `ridenow-prod-db-sg` |

VPC dùng khối CIDR `10.30.0.0/16`. Hai public subnet và hai private subnet được chia trên hai Availability Zone. Có tổng cộng ba route table: một public route table dùng chung cho cả hai public subnet, và một private route table cho mỗi private subnet.

## Depends on

Không có. Đây là trang mạng đầu tiên, bạn có thể làm ngay sau mục 5.3.

## Create the resources

Mở AWS console ở Region Asia Pacific (Singapore) `ap-southeast-1`. Dùng hai Availability Zone đầu tiên, `ap-southeast-1a` và `ap-southeast-1b`.

### 1. VPC (`Vpc`)

1. Vào **VPC -> Your VPCs -> Create VPC**.
2. Đặt **Resources to create** là `VPC only`.
3. Đặt **Name tag - optional** là `ridenow-prod-vpc`.
4. Đặt **IPv4 CIDR block** là `IPv4 CIDR manual input`.
5. Đặt **IPv4 CIDR** là `10.30.0.0/16`.
6. Đặt **IPv6 CIDR block** là `No IPv6 CIDR block`.
7. Đặt **Tenancy** là `Default`.
8. Chọn **Create VPC**.

Bật DNS:

1. Chọn `ridenow-prod-vpc`.
2. Vào **Actions -> Edit VPC settings**.
3. Trong **DNS settings**, đánh dấu **Enable DNS resolution**.
4. Đánh dấu **Enable DNS hostnames**.
5. Chọn **Save**.

### 2. Internet gateway (`InternetGateway`)

1. Vào **VPC -> Internet gateways -> Create internet gateway**.
2. Đặt **Name tag** là `ridenow-prod-igw`.
3. Chọn **Create internet gateway**.
4. Vào **Actions -> Attach to VPC**.
5. Đặt **Available VPCs** là `ridenow-prod-vpc`.
6. Chọn **Attach internet gateway**.

### 3. Subnet (`Subnets`)

Mở biểu mẫu:

1. Vào **VPC -> Subnets -> Create subnet**.
2. Đặt **VPC ID** là `ridenow-prod-vpc`.

Subnet 1:

1. Đặt **Subnet name** là `ridenow-prod-public-1`.
2. Đặt **Availability Zone** là `ap-southeast-1a`.
3. Đặt **IPv4 subnet CIDR block** là `10.30.0.0/20`.
4. Trong **Tags - optional**, chọn **Add new tag**. Đặt **Key** là `kubernetes.io/role/elb` và **Value** là `1`. Tag `Name` đã được điền sẵn từ **Subnet name**.

Subnet 2 (chọn **Add new subnet** trước):

1. Đặt **Subnet name** là `ridenow-prod-public-2`.
2. Đặt **Availability Zone** là `ap-southeast-1b`.
3. Đặt **IPv4 subnet CIDR block** là `10.30.16.0/20`.
4. Trong **Tags - optional**, chọn **Add new tag**. Đặt **Key** là `kubernetes.io/role/elb` và **Value** là `1`.

Subnet 3 (chọn **Add new subnet** trước):

1. Đặt **Subnet name** là `ridenow-prod-private-1`.
2. Đặt **Availability Zone** là `ap-southeast-1a`.
3. Đặt **IPv4 subnet CIDR block** là `10.30.48.0/20`.
4. Trong **Tags - optional**, chọn **Add new tag**. Đặt **Key** là `kubernetes.io/role/internal-elb` và **Value** là `1`.

Subnet 4 (chọn **Add new subnet** trước):

1. Đặt **Subnet name** là `ridenow-prod-private-2`.
2. Đặt **Availability Zone** là `ap-southeast-1b`.
3. Đặt **IPv4 subnet CIDR block** là `10.30.64.0/20`.
4. Trong **Tags - optional**, chọn **Add new tag**. Đặt **Key** là `kubernetes.io/role/internal-elb` và **Value** là `1`.

Hoàn tất:

1. Chọn **Create subnet**.

Kiểm tra các tag. Mở tab **Tags** của từng subnet. Hai public subnet phải có `kubernetes.io/role/elb` = `1`, và hai private subnet phải có `kubernetes.io/role/internal-elb` = `1`. AWS Load Balancer Controller tìm subnet cho load balancer theo các tag này. Nếu thiếu tag, chọn **Manage tags** và thêm vào.

Bật public IP cho `ridenow-prod-public-1`:

1. Vào **VPC -> Subnets** và chọn `ridenow-prod-public-1`.
2. Vào **Actions -> Edit subnet settings**.
3. Đánh dấu **Enable auto-assign public IPv4 address**.
4. Chọn **Save**.

Bật public IP cho `ridenow-prod-public-2`:

1. Vào **VPC -> Subnets** và chọn `ridenow-prod-public-2`.
2. Vào **Actions -> Edit subnet settings**.
3. Đánh dấu **Enable auto-assign public IPv4 address**.
4. Chọn **Save**.

Giữ tắt auto-assign ở cả hai private subnet.

### 4. Public route table (`PublicRouteTable`)

Tạo bảng:

1. Vào **VPC -> Route tables -> Create route table**.
2. Đặt **Name** là `ridenow-prod-public-rt`.
3. Đặt **VPC** là `ridenow-prod-vpc`.
4. Chọn **Create route table**.

Thêm route mặc định:

1. Mở tab **Routes** và chọn **Edit routes -> Add route**.
2. Đặt **Destination** là `0.0.0.0/0`.
3. Đặt **Target** là **Internet Gateway** và chọn `ridenow-prod-igw`.
4. Chọn **Save changes**.

Liên kết các subnet:

1. Mở tab **Subnet associations** và chọn **Edit subnet associations**.
2. Đánh dấu `ridenow-prod-public-1`.
3. Đánh dấu `ridenow-prod-public-2`.
4. Chọn **Save associations**.

### 5. Elastic IP và NAT gateway (`NatGateway`)

Cấp phát Elastic IP thứ nhất:

1. Vào **VPC -> Elastic IPs -> Allocate Elastic IP address**.
2. Đặt **Public IPv4 address pool** là `Amazon's pool of IPv4 addresses`.
3. Đặt **Network border group** là `ap-southeast-1`.
4. Trong **Tags - optional**, chọn **Add new tag**. Đặt **Key** là `Name` và **Value** là `ridenow-prod-nat-eip-1`.
5. Chọn **Allocate**.

Cấp phát Elastic IP thứ hai:

1. Chọn **Allocate Elastic IP address** lần nữa.
2. Dùng cùng thiết lập với địa chỉ thứ nhất, nhưng đặt **Value** của tag `Name` là `ridenow-prod-nat-eip-2`.
3. Chọn **Allocate**.

Tạo NAT gateway 1:

1. Vào **VPC -> NAT gateways -> Create NAT gateway**.
2. Đặt **Name** là `ridenow-prod-nat-1`.
3. Đặt **Subnet** là `ridenow-prod-public-1`.
4. Đặt **Connectivity type** là `Public`.
5. Đặt **Elastic IP allocation ID** là Elastic IP thứ nhất bạn vừa cấp phát.
6. Chọn **Create NAT gateway**.

Tạo NAT gateway 2:

1. Chọn **Create NAT gateway** lần nữa.
2. Đặt **Name** là `ridenow-prod-nat-2`.
3. Đặt **Subnet** là `ridenow-prod-public-2`.
4. Đặt **Connectivity type** là `Public`.
5. Đặt **Elastic IP allocation ID** là Elastic IP thứ hai (địa chỉ chưa được gắn).
6. Chọn **Create NAT gateway**.

Chờ:

1. Chờ đến khi cả hai gateway có **State** là `Available` (3 đến 5 phút).

### 6. Private route table (`PrivateRouteTable`)

Route table 1:

1. Vào **VPC -> Route tables -> Create route table**.
2. Đặt **Name** là `ridenow-prod-private-rt-1`.
3. Đặt **VPC** là `ridenow-prod-vpc`.
4. Chọn **Create route table**.
5. Mở tab **Routes** và chọn **Edit routes -> Add route**.
6. Đặt **Destination** là `0.0.0.0/0`.
7. Đặt **Target** là **NAT Gateway** và chọn `ridenow-prod-nat-1`.
8. Chọn **Save changes**.
9. Mở tab **Subnet associations** và chọn **Edit subnet associations**.
10. Đánh dấu `ridenow-prod-private-1`.
11. Chọn **Save associations**.

Route table 2:

1. Vào **VPC -> Route tables -> Create route table**.
2. Đặt **Name** là `ridenow-prod-private-rt-2`.
3. Đặt **VPC** là `ridenow-prod-vpc`.
4. Chọn **Create route table**.
5. Mở tab **Routes** và chọn **Edit routes -> Add route**.
6. Đặt **Destination** là `0.0.0.0/0`.
7. Đặt **Target** là **NAT Gateway** và chọn `ridenow-prod-nat-2`.
8. Chọn **Save changes**.
9. Mở tab **Subnet associations** và chọn **Edit subnet associations**.
10. Đánh dấu `ridenow-prod-private-2`.
11. Chọn **Save associations**.

### 7. Security group của cluster (`ClusterSecurityGroup`)

Tạo nhóm:

1. Vào **VPC -> Security groups -> Create security group**.
2. Đặt **Security group name** là `ridenow-prod-cluster-sg`.
3. Đặt **Description** là `ridenow-prod EKS / application pods`.
4. Đặt **VPC** là `ridenow-prod-vpc`.
5. Để trống **Inbound rules**.
6. Giữ dòng **Outbound rules** mặc định: **Type** `All traffic`, **Protocol** `All`, **Port range** `All`, **Destination** `Anywhere-IPv4` `0.0.0.0/0`.
7. Đặt **Description** của dòng outbound đó là `Allow all egress (NAT to internet, RDS, etc.)`.
8. Trong **Tags - optional**, chọn **Add new tag**. Đặt **Key** là `Name` và **Value** là `ridenow-prod-cluster-sg`. Các lệnh tra cứu biến tìm nhóm theo tag này, không theo tên nhóm.
9. Chọn **Create security group**.

Thêm rule tự tham chiếu:

1. Mở `ridenow-prod-cluster-sg`.
2. Mở tab **Inbound rules** và chọn **Edit inbound rules -> Add rule**.
3. Đặt **Type** là `All traffic`.
4. Đặt **Source** là **Custom** và chọn `ridenow-prod-cluster-sg` (chính nhóm này).
5. Đặt **Description** là `Allow all traffic between members of this SG`.
6. Chọn **Save rules**.

### 8. Security group của cơ sở dữ liệu (`DbSecurityGroup`)

1. Vào **VPC -> Security groups -> Create security group**.
2. Đặt **Security group name** là `ridenow-prod-db-sg`.
3. Đặt **Description** là `ridenow-prod RDS PostgreSQL (private)`.
4. Đặt **VPC** là `ridenow-prod-vpc`.
5. Trong **Inbound rules**, chọn **Add rule**.
6. Đặt **Type** là `PostgreSQL` (Protocol `TCP`, Port range `5432`).
7. Đặt **Source** là **Custom** và chọn `ridenow-prod-cluster-sg`.
8. Đặt **Description** của rule là `Postgres 5432 from the RideNow cluster SG only`.
9. Giữ dòng **Outbound rules** mặc định: **Type** `All traffic`, **Protocol** `All`, **Port range** `All`, **Destination** `Anywhere-IPv4` `0.0.0.0/0`.
10. Đặt **Description** của dòng outbound đó là `Allow all egress`.
11. Trong **Tags - optional**, chọn **Add new tag**. Đặt **Key** là `Name` và **Value** là `ridenow-prod-db-sg`.
12. Chọn **Create security group**.

Không thêm rule inbound theo CIDR.

## Values to record

Hãy chạy khối thứ hai của phần Set variables ngay bây giờ. Năm biến dưới đây được tra cứu tự động, nên bạn không phải sao chép gì bằng tay. Các trang sau lặp lại cùng các lệnh tra cứu ở đầu trang.

| Biến | Giá trị | Được dùng bởi |
|---|---|---|
| `VPC_ID` | ID của `ridenow-prod-vpc` | 5.7 (security group của cache), 5.8 (load balancer controller) |
| `PUBLIC_SUBNET_IDS` | ID của `ridenow-prod-public-1` và `ridenow-prod-public-2` | Load balancer do Ingress tạo, tìm các public subnet theo tag |
| `PRIVATE_SUBNET_IDS` | ID của `ridenow-prod-private-1` và `ridenow-prod-private-2` | 5.6 (DB subnet group), 5.7 (cache subnet group), 5.8 (cluster và các Fargate profile) |
| `CLUSTER_SG_ID` | ID của `ridenow-prod-cluster-sg` | 5.7 (nguồn ingress của cache), 5.8 (security group của cluster) |
| `DB_SG_ID` | ID của `ridenow-prod-db-sg` | 5.6 (sáu instance cơ sở dữ liệu), 5.9 (rule ingress từ security group của EKS cluster) |

CIDR của VPC `10.30.0.0/16` là giá trị cố định bạn đã nhập ở bước 1, chỉ cần để tham khảo.

## Post-deploy steps

Trang này không cần bước nào sau khi triển khai.

## Verify

Chạy các lệnh sau trong cùng shell:

```bash
aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --query 'Vpcs[0].[CidrBlock,State]' --output text
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'length(Subnets)'
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" --query 'length(NatGateways)'
aws ec2 describe-security-groups --group-ids "$DB_SG_ID" --query 'SecurityGroups[0].IpPermissions[].[IpProtocol,FromPort,ToPort]' --output text
```

Kết quả mong đợi:

```text
10.30.0.0/16	available
4
2
tcp	5432	5432
```

Nếu có giá trị sai, hãy kiểm tra lại bước tương ứng trong VPC console.
