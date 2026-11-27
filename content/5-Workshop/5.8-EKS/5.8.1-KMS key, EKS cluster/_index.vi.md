---
title : "KMS key, EKS cluster"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.8.1. </b> "
---


Trong phần này chúng ta sẽ tạo tài nguyên KMS key, EKS cluster  và các Fargate profiles.

## 5.8.1 Tạo KMS key
1. Truy cập vào trang Key Management Service

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-1.png)

Đây là trang Key Management Service
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-2.png)

2. Chúng ta sẽ tạo Key với thông số sau 

| Setting | Value |
|---|---|
| Key type | `Symmetric` |
| Key usage | `Encrypt and decrypt` |
| Alias | `Alias ridenow-prod-eks-secrets` |
| Description | `ridenow-prod EKS secrets encryption key` |


![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-3.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-4.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-10.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-6.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-7.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-11.png)

3. thiết lập với json sau
   
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountAdmin",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    }
  ]
}
```

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-12.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-13.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-14.png)

4. Tiếp theo cần bật rotation

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-15.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-16.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-17.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-18.png)

## 5.8.2 Tạo EKS Cluster
Truy cập vào trang EKS chọn Create Cluster

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-28.png)


2. Đặt **Configuration options** là `Custom configuration`.
3. Tắt **Use EKS Auto Mode**.
4. Đặt **Name** là `ridenow-prod`.
5. Đặt **Cluster IAM role** là `ridenow-prod-eks-cluster-role`.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-29.png)

Đặt Kubernetes version là 1.31.
Đặt Upgrade policy là Standard.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-30.png)

3. Giữ **Bootstrap cluster administrator access** ở `Allow cluster administrator access`.
4. Giữ **Cluster authentication mode** ở `EKS API and ConfigMap`.
5.  Tick **Turn on envelope encryption of Kubernetes secrets using KMS**.
6.  Đặt **KMS key** là alias `ridenow-prod-eks-secrets`.
7.  Chọn **Next**.
   
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-31.png)

1.  Đặt **VPC** là `ridenow-prod-vpc`.
2.  Ở **Subnets**, bỏ tick mọi subnet trừ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
3.  Đặt **Security groups** là `ridenow-prod-cluster-sg`.
4.  Đặt **Cluster IP address family** là `IPv4`.
5.  
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-59.png)
6.  Đặt **Cluster endpoint access** là `Public and private`.
7.  Mở rộng **Advanced settings**. Giữ **Public access source allowlist** ở `0.0.0.0/0`.
8.  Chọn **Next**.
9.  
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-33.png)


10. Ở **Configure observability**, để tắt mọi loại log của control plane. Chọn **Next**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-34.png)
11. Ở **Select add-ons**, giữ tick `kube-proxy`, `CoreDNS` và `Amazon VPC CNI`. Chọn **Next**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-35.png)

12. Ở **Configure selected add-ons settings**, giữ mặc định. Chọn **Next**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-36.png)
13. Chọn **Create**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-37.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-39.png)
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-40.png)
14. Chờ 10 đến 15 phút cho đến khi **Status** là `Active`.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-41.png)

## 5.8.3 Tạo Fargate profile
bước này chúng ta sẽ tạo Fargate profile `ridenow-prod-system` 

1. Vào **EKS -> Clusters -> `ridenow-prod` -> Compute** tab **-> Fargate profiles -> Add Fargate profile**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-42.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-43.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-44.png)


2. Đặt **Name** là `ridenow-prod-system`.
3. Đặt **Pod execution role** là `ridenow-prod-eks-fargate-pod-execution-role`.
4. Ở **Subnets**, chỉ giữ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
5. Chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-45.png)


6. Ở **Pod selectors**, đặt **Namespace** là `kube-system`. Để trống **Match labels**.
7. Chọn **Add selector**. Đặt **Namespace** là `external-secrets`. Để trống **Match labels**.
8. Chọn **Next**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-46.png)

9.  Chọn **Create**.
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-47.png)

Chờ 1 đến 2 phút cho đến khi **Status** là `Active` thì mới tạo tiếp Fargate profile khác.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-60.png)

Tương tự chúng ta sẽ tạo Fargate profile `ridenow-prod-app` (`AppFargateProfile`) và `ridenow-prod-observability` (`ObservabilityFargateProfile`) với thông số sau 

ridenow-prod-app

1. Đặt **Name** là `ridenow-prod-app`.
2. Đặt **Pod execution role** là `ridenow-prod-eks-fargate-pod-execution-role`.
3. Ở **Subnets**, chỉ giữ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
4. Chọn **Next**.
5. Ở **Pod selectors**, đặt **Namespace** là `ridenow`. Để trống **Match labels**.

ridenow-prod-observability

6. Đặt **Name** là `ridenow-prod-observability`.
7. Đặt **Pod execution role** là `ridenow-prod-eks-fargate-pod-execution-role`.
8. Ở **Subnets**, chỉ giữ `ridenow-prod-private-1` và `ridenow-prod-private-2`.
9.  Chọn **Next**.
10. Ở **Pod selectors**, đặt **Namespace** là `aws-observability`. Để trống **Match labels**.
