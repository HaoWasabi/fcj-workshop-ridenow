---
title : "Rule Fargate pod và IAM Roles for Service Accounts"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.8.2. </b> "
---

## Create the resources

Mở AWS console ở region Asia Pacific (Singapore) `ap-southeast-1`. Tạo các tài nguyên theo thứ tự này. Các role cần OIDC provider.

### 1. Inbound rule: PostgreSQL từ các Fargate pod (`DbIngressFromFargatePods`)

1. Vào **EC2 -> Security Groups**.
2. Tìm `ridenow-prod-db-sg` và chọn nó.
3. Mở tab **Inbound rules** và chọn **Edit inbound rules -> Add rule**.
4. Đặt **Type** là `PostgreSQL` (mục này tự điền **Protocol** `TCP` và **Port range** `5432`).
5. Đặt **Source** là `Custom`.
6. Trong ô tìm kiếm, dán giá trị của `EKS_CLUSTER_SG_ID` và chọn group `eks-cluster-sg-ridenow-prod-<number>`.
7. Đặt **Description** là `PostgreSQL 5432 from the EKS cluster security group (Fargate pods)`.
8. Chọn **Save rules**.

Group lúc này có hai rule ở cổng 5432: rule từ mục 5.4 và rule này.

### 2. Inbound rule: Redis từ các Fargate pod (`RedisIngressFromFargatePods`)

1. Vào **EC2 -> Security Groups**.
2. Tìm `ridenow-prod-redis-sg` và chọn nó.
3. Mở tab **Inbound rules** và chọn **Edit inbound rules -> Add rule**.
4. Đặt **Type** là `Custom TCP`.
5. Đặt **Port range** là `6379`.
6. Đặt **Source** là `Custom`.
7. Dán giá trị của `EKS_CLUSTER_SG_ID` và chọn `eks-cluster-sg-ridenow-prod-<number>`.
8. Đặt **Description** là `Redis 6379 from the EKS cluster security group (Fargate pods)`.
9. Chọn **Save rules**.

Group lúc này có hai rule ở cổng 6379.


### 3. Thêm OIDC identity provider

Đầu tiên cần lấu issuer URL của cluster:

Vào AWS Console → tìm Elastic Kubernetes Service (EKS) → chọn Clusters.

Chọn cluster ridenow-prod ở region ap-southeast-1.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-49.png)

Trong tab Overview, tìm phần OpenID Connect provider URL (hoặc Identity).

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-50.png)
URL sẽ có dạng:
https://oidc.eks.ap-southeast-1.amazonaws.com/id/XXXXXXXXXXXX

Thiết lập Provider

Vào **IAM -> Identity providers -> Add provider**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-51.png)

Đặt **Provider type** là `OpenID Connect`.
Dán URL vừa in ra vào **Provider URL**.
Đặt **Audience** là `sts.amazonaws.com`.
Chọn **Add provider**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-52.png)

## 5.8.4 Tạo IAM Roles for Service Accounts


Đầu tiên tạo ridenow-prod-alb-controller 

Vào **IAM -> Roles -> Create role**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-48.png)

Đặt **Trusted entity type** là `Web identity`.

Đặt **Identity provider** là OIDC provider đã tạo

Đặt **Audience** là `sts.amazonaws.com`.

Chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-53.png)

Chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-54.png)

Đặt **Role name** là tên trong bảng.

ridenow-prod-alb-controller

Để trống **Description**.

Chọn **Create role**.

tạo ridenow-prod-external-secrets tương tự nhưng chúng ta thêm  pocily

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-55.png)
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-56.png)

```json

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": [
        "arn:aws:secretsmanager:ap-southeast-1:492767006533:secret:ridenow-prod-*",
        "arn:aws:secretsmanager:ap-southeast-1:492767006533:secret:rds!db-*"
      ]
    }
  ]
}
```
thay 492767006533 là Account ID của bạn 

Tiếp theo Tạo role lần lượt, với các giá trị sau:

| Role name | Service account (`namespace:name`) | Permissions |
|---|---|---|
| `ridenow-prod-trip-service` | `ridenow:trip-service` | managed policy `ridenow-prod-trip-service-messaging` |
| `ridenow-prod-payment-service` | `ridenow:payment-service` | managed policy `ridenow-prod-payment-service-messaging` |
| `ridenow-prod-matching-service` | `ridenow:matching-service` | managed policy `ridenow-prod-matching-service-messaging` |
| `ridenow-prod-notification-worker` | `ridenow:notification-worker` | managed policy `ridenow-prod-notification-worker-messaging` |


Lặp lại danh sách này cho từng dòng của bảng.

Vào **IAM -> Roles -> Create role**.

Đặt **Trusted entity type** là `Web identity`.

Đặt **Identity provider** là OIDC provider ở bước 3 (mục kết thúc bằng `OIDC_ID` của bạn).

Đặt **Audience** là `sts.amazonaws.com`.

Chọn **Next**.

Ở **Add permissions** tìm policy trong bảng và tick nó.

Chọn **Next**.

Đặt **Role name** là tên trong bảng.

Để trống **Description**.

Chọn **Create role**.

#### 4.2 Thay trust policy

Lặp lại danh sách này cho từng role trong sáu role.

Trong danh sách role, tìm tên role và mở nó.

Mở tab **Trust relationships** và chọn **Edit trust policy**.

Xóa toàn bộ nội dung.

Dán trust policy mà khối lệnh ở trên đã in cho role này.

Chọn **Update policy**.
