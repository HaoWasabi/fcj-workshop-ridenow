---
title : "IAM Roles for Service Accounts và các Fargate profiles"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.8.2. </b> "
---


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

1. Vào **IAM -> Identity providers -> Add provider**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-51.png)
2. Đặt **Provider type** là `OpenID Connect`.
3. Dán URL vừa in ra vào **Provider URL**.
4. Đặt **Audience** là `sts.amazonaws.com`.
6. Chọn **Add provider**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-52.png)

## 5.8.4 Tạo IAM Roles for Service Accounts


Đầu tiên tạo ridenow-prod-alb-controller 

1. Vào **IAM -> Roles -> Create role**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-48.png)

2. Đặt **Trusted entity type** là `Web identity`.
3. Đặt **Identity provider** là OIDC provider đã tạo
4. Đặt **Audience** là `sts.amazonaws.com`.
5. Chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-53.png)
5. Chọn **Next**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-54.png)
8. Đặt **Role name** là tên trong bảng.
ridenow-prod-alb-controller
9. Để trống **Description**.
10. Chọn **Create role**.


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

1. Vào **IAM -> Roles -> Create role**.
2. Đặt **Trusted entity type** là `Web identity`.
3. Đặt **Identity provider** là OIDC provider ở bước 3 (mục kết thúc bằng `OIDC_ID` của bạn).
4. Đặt **Audience** là `sts.amazonaws.com`.
5. Chọn **Next**.
6. Ở **Add permissions** tìm policy trong bảng và tick nó.
7. Chọn **Next**.
8. Đặt **Role name** là tên trong bảng.
9. Để trống **Description**.
10. Chọn **Create role**.

#### 4.2 Thay trust policy

Lặp lại danh sách này cho từng role trong sáu role.

1. Trong danh sách role, tìm tên role và mở nó.
2. Mở tab **Trust relationships** và chọn **Edit trust policy**.
3. Xóa toàn bộ nội dung.
4. Dán trust policy mà khối lệnh ở trên đã in cho role này.
5. Chọn **Update policy**.


