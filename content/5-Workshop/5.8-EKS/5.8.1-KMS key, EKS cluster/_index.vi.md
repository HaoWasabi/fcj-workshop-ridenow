---
title : "KMS key, EKS cluster, Fargate profiles"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.8.1. </b> "
---


Trong phần này, chúng ta sẽ tiến hành tạo và cấu hình các tài nguyên AWS cần thiết cho EKS, bao gồm KMS Key, EKS Cluster, IAM Roles for Service Accounts (IRSA) và các Fargate Profiles.

## 5.8.1 Tạo KMS key
AWS Key Management Service (AWS KMS) là dịch vụ quản lý khóa mã hóa của AWS. Trong EKS, KMS được sử dụng để mã hóa Kubernetes Secrets, giúp bảo vệ các thông tin nhạy cảm như credentials, tokens và passwords được lưu trữ trong cluster.

Truy cập vào Key Management Service trên AWS Console.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-1.png)

Đây là giao diện quản lý các KMS Keys trong AWS.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-2.png)

CHúng ta sẽ tạo một KMS Key mới với các thông số sau:

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

Sau khi tạo Key, chúng ta cần cấu hình Key Policy để xác định những IAM Principal nào được phép sử dụng và quản lý KMS Key.

Sử dụng policy JSON sau:

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

Policy này cấp quyền quản lý KMS Key ở cấp tài khoản. Các IAM User và Role vẫn cần có IAM permissions phù hợp để thực hiện các thao tác tương ứng.


![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-12.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-13.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-14.png)

Tiếp theo, bật tính năng Automatic Key Rotation cho KMS Key.

Tính năng này cho phép AWS tự động tạo phiên bản cryptographic key material mới theo chu kỳ, giúp giảm rủi ro khi sử dụng một khóa mã hóa trong thời gian dài.

Việc xoay vòng khóa không làm mất khả năng giải mã dữ liệu đã được mã hóa bằng phiên bản khóa cũ. AWS KMS vẫn giữ lại các phiên bản key material cần thiết để giải mã dữ liệu trước đó.

Thực hiện bật Automatic Key Rotation trong phần cấu hình của KMS Key.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-15.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-16.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-17.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-18.png)

## 5.8.2 Tạo EKS Cluster

Amazon Elastic Kubernetes Service (Amazon EKS) là dịch vụ Kubernetes được quản lý bởi AWS. EKS chịu trách nhiệm quản lý Kubernetes Control Plane, bao gồm API Server, Scheduler và các thành phần điều phối của Kubernetes.

Trong phần này, chúng ta sẽ tạo EKS Cluster có tên ridenow-prod, sử dụng Kubernetes version 1.34, triển khai trên các Private Subnets và sử dụng AWS KMS để mã hóa Kubernetes Secrets.

Truy cập vào Amazon Elastic Kubernetes Service (EKS) trên AWS Console và chọn Create Cluster.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-28.png)


Thiết lập các thông số sau:

- **Configuration options:** Chọn `Custom configuration`.
- **Use EKS Auto Mode:** Tắt.

Chúng ta sử dụng Custom configuration để tự quản lý cấu hình Cluster và lựa chọn cách triển khai Compute thông qua Fargate Profiles ở bước tiếp theo. Vì vậy, không sử dụng EKS Auto Mode.

Tiếp theo, cấu hình thông tin Cluster:

| Setting | Value |
|---|---|
| Name | `ridenow-prod` |
| Cluster IAM role | `ridenow-prod-eks-cluster-role` |

**Cluster IAM role** là IAM Role cho phép EKS Control Plane gọi các AWS API cần thiết để quản lý tài nguyên và vận hành Cluster.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-29.png)

Thiết lập các thông số:

| Setting | Value |
|---|---|
| Kubernetes version | `1.34` |
| Upgrade policy | `Standard` |

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-30.png)


Tiếp theo, cấu hình quyền truy cập và mã hóa:

- **Bootstrap cluster administrator access:** Giữ `Allow cluster administrator access`.
- **Cluster authentication mode:** Giữ `EKS API and ConfigMap`.
- **Turn on envelope encryption of Kubernetes secrets using KMS:** Bật.
- **KMS key:** Chọn alias `ridenow-prod-eks-secrets`.

Sau khi hoàn tất, chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-31.png)


Networking xác định cách EKS Cluster kết nối với VPC, các Subnets và những nguồn có thể truy cập Kubernetes API Server.

Cấu hình các thông số sau:

| Setting | Value |
|---|---|
| VPC | `ridenow-prod-vpc` |
| Subnets | `ridenow-prod-private-1`, `ridenow-prod-private-2` |
| Security groups | `ridenow-prod-cluster-sg` |
| Cluster IP address family | `IPv4` |

Ở phần **Subnets**, bỏ chọn tất cả các Subnets ngoại trừ:

- `ridenow-prod-private-1`
- `ridenow-prod-private-2`

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-59.png)

Tiếp theo, cấu hình **Cluster endpoint access**:

| Setting | Value |
|---|---|
| Cluster endpoint access | `Public and private` |
| Public access source allowlist | `0.0.0.0/0` |

Mở rộng **Advanced settings** và giữ **Public access source allowlist** ở `0.0.0.0/0`.

Sau khi hoàn tất, chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-33.png)



**Control Plane Logging** cho phép EKS gửi các loại log hoạt động của Kubernetes Control Plane đến Amazon CloudWatch Logs.

Trong phần **Configure observability**, tắt tất cả các loại Control Plane Logs.

Trong môi trường Lab này, chúng ta tắt các loại log để tránh phát sinh chi phí CloudWatch Logs không cần thiết.

Sau đó, chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-34.png)


EKS Add-ons là các thành phần mở rộng giúp Kubernetes Cluster thực hiện những chức năng cơ bản như quản lý DNS, networking và network proxy.

Trong phần **Select add-ons**, giữ chọn các Add-ons sau:

- `kube-proxy`
- `CoreDNS`
- `Amazon VPC CNI`

**Giải thích:**

- **kube-proxy:** Quản lý các quy tắc networking để hỗ trợ giao tiếp giữa các Kubernetes Services và Pods.
- **CoreDNS:** Cung cấp dịch vụ DNS nội bộ cho Kubernetes Cluster, giúp các Pods và Services có thể phân giải tên miền.
- **Amazon VPC CNI:** Cho phép Kubernetes Pods sử dụng địa chỉ IP từ VPC và giao tiếp với các tài nguyên AWS thông qua mạng VPC.

Đây là các thành phần quan trọng để Cluster có thể vận hành các Kubernetes Workloads.

Sau khi hoàn tất, chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-35.png)


Ở phần **Configure selected add-ons settings**, giữ nguyên các cấu hình mặc định cho cả ba Add-ons.

Chọn **Next** để tiếp tục.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-36.png)

Kiểm tra lại toàn bộ cấu hình đã thiết lập. Nếu các thông số chính xác, chọn **Create** để bắt đầu khởi tạo EKS Cluster.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-37.png)

AWS sẽ tiến hành tạo các tài nguyên cần thiết cho Kubernetes Control Plane và cấu hình các thành phần liên quan.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-39.png)
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-40.png)

Quá trình khởi tạo thường mất khoảng 10–15 phút. Chờ cho đến khi **Status** chuyển sang `Active`.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-41.png)


Sau khi Cluster có trạng thái `Active`, chúng ta đã hoàn thành việc khởi tạo EKS Control Plane.

## 5.8.3 Tạo Fargate profile

Amazon EKS Fargate là dịch vụ cho phép chạy Kubernetes Pods trên AWS Fargate mà không cần tự quản lý EC2 Worker Nodes.

**Fargate Profile** xác định những Kubernetes Pods nào sẽ được triển khai trên Fargate thông qua Namespace và Labels Selector.


Trong Workshop này, chúng ta sẽ tạo 3 Fargate Profiles để phân chia các Pods theo mục đích sử dụng:

| Fargate Profile | Namespace | Mục đích |
|---|---|---|
| `ridenow-prod-system` | `kube-system`, `external-secrets` | Chạy các System Pods và External Secrets |
| `ridenow-prod-app` | `ridenow` | Chạy các Application Pods của RideNow |
| `ridenow-prod-observability` | `aws-observability` | Chạy các Pods phục vụ Observability |



Đầu tiên, chúng ta sẽ tạo Fargate Profile dành cho các thành phần hệ thống và External Secrets.

Truy cập vào:

**EKS → Clusters → `ridenow-prod` → Compute → Fargate profiles → Add Fargate profile**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-42.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-43.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-44.png)


Thiết lập các thông số sau:

| Setting | Value |
|---|---|
| Name | `ridenow-prod-system` |
| Pod execution role | `ridenow-prod-eks-fargate-pod-execution-role` |
| Subnets | `ridenow-prod-private-1`, `ridenow-prod-private-2` |


Sau khi hoàn tất, chọn **Next**.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-45.png)


Tiếp theo, cấu hình **Pod selectors**.

Chúng ta sẽ thêm hai Selectors:

| Namespace | Match labels |
|---|---|
| `kube-system` | Để trống |
| `external-secrets` | Để trống |

Chọn **Add selector** để thêm Namespace `external-secrets`, sau đó chọn **Next**.


![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-46.png)

Chọn **Create** để tạo Fargate Profile.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-47.png)


Chờ khoảng 1–2 phút cho đến khi **Status** chuyển sang `Active`.

Chỉ tiếp tục tạo Fargate Profile tiếp theo sau khi Profile hiện tại đã ở trạng thái `Active`.

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-60.png)

**Tạo Fargate Profile `ridenow-prod-app`**

Fargate Profile này dành cho các Application Pods của hệ thống RideNow.

Thực hiện tương tự bước trên và thiết lập các thông số sau:

| Setting | Value |
|---|---|
| Name | `ridenow-prod-app` |
| Pod execution role | `ridenow-prod-eks-fargate-pod-execution-role` |
| Subnets | `ridenow-prod-private-1`, `ridenow-prod-private-2` |

Ở phần **Pod selectors**, cấu hình:

| Namespace | Match labels |
|---|---|
| `ridenow` | Để trống |

Chọn **Next** và **Create** để hoàn tất.


**Tạo Fargate Profile `ridenow-prod-observability`**

Fargate Profile cuối cùng dành cho các thành phần phục vụ Observability.

Thực hiện tương tự và thiết lập các thông số:

| Setting | Value |
|---|---|
| Name | `ridenow-prod-observability` |
| Pod execution role | `ridenow-prod-eks-fargate-pod-execution-role` |
| Subnets | `ridenow-prod-private-1`, `ridenow-prod-private-2` |

Ở phần **Pod selectors**, cấu hình:

| Namespace | Match labels |
|---|---|
| `aws-observability` | Để trống |

Namespace `aws-observability` được sử dụng cho các thành phần liên quan đến cấu hình và thu thập dữ liệu Observability trong môi trường EKS.

Chọn **Next** và **Create** để hoàn tất.