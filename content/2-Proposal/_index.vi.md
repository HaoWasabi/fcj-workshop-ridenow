---
title: "Bản đề xuất"
date: 2026-09-09
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# RideNow
## Triển khai nền tảng gọi xe trên AWS
### 1. Tổng quan dự án

RideNow là nền tảng gọi xe gồm bảy dịch vụ triển khai độc lập (user, driver, trip, payment, matching, notification và realtime gateway) cùng một giao diện web dạng single-page. Mỗi dịch vụ có cơ sở dữ liệu riêng và trao đổi sự kiện qua một topic và các hàng đợi.

Bản đề xuất này chỉ nói về hạ tầng: cách chạy RideNow trên Amazon Web Services trong một môi trường production (`prod`) tại Region Singapore (`ap-southeast-1`). Môi trường được dựng thủ công, từng service console một, theo các hướng dẫn từng bước trong `guides/cloudformation/`. Các template chỉ đọc trong `iac/stacks/` và các giá trị trong `iac/params/prod.json` quy định mọi tên và cấu hình.

- **Môi trường:** `prod`, region `ap-southeast-1`
- **Thời gian dự án:** [OWNER: điền lịch trình]
- **Quy mô nhóm và vai trò:** [OWNER: điền quy mô nhóm và vai trò]

---

### 2. Vấn đề cần giải quyết

#### Hiện trạng
RideNow đã chạy được như một hệ thống cục bộ hoàn chỉnh (`docker compose up --build`): sáu container PostgreSQL, một container Valkey, LocalStack cho SNS và SQS, và bảy dịch vụ. Chuyển lên cloud chỉ là đổi cấu hình ở từng ranh giới, vì ranh giới giữa các dịch vụ không thay đổi. Điều còn thiếu là một môi trường cloud an toàn, lặp lại được để chạy hệ thống.

#### Mục tiêu
- Chạy mọi dịch vụ dưới dạng container trên **Amazon EKS với AWS Fargate**, không phải quản lý node group.
- Dùng một instance **Amazon RDS for PostgreSQL** cho mỗi dịch vụ sở hữu dữ liệu (tổng cộng sáu), Multi-AZ, đặt trong private subnet.
- Truyền sự kiện nghiệp vụ qua **Amazon SNS** và **Amazon SQS**, mỗi hàng đợi có một dead-letter queue.
- Lưu trạng thái tài xế trực tiếp, chỉ mục GEO, Pub/Sub và luồng vị trí trong **Amazon ElastiCache for Valkey**.
- Lưu thông tin đăng nhập cơ sở dữ liệu trong **AWS Secrets Manager** và mã hóa dữ liệu bằng **AWS KMS**. Mỗi workload chỉ có quyền IAM cần thiết (IRSA).
- Chỉ có một điểm vào từ internet: **Application Load Balancer** do AWS Load Balancer Controller tạo, đặt sau **Amazon CloudFront**, với SPA lưu trong bucket **Amazon S3** riêng tư.

#### Giải pháp
- Dựng mạng trước (hai public subnet và hai private subnet trên hai Availability Zone, mỗi zone một NAT gateway), sau đó là IAM, ECR, dữ liệu, messaging và EKS cluster. Các lớp dữ liệu, messaging và cluster chỉ phụ thuộc IAM và mạng, nên có thể dựng theo thứ tự bất kỳ.
- Đẩy bảy image dịch vụ lên Amazon ECR, rồi triển khai trên Fargate.
- Tạo frontend sau cùng, khi Ingress đã có địa chỉ ALB. CloudFront chuyển `/api/*` và `/live*` tới ALB đó và phục vụ SPA từ S3.

#### Lợi ích đầu tư
- **Lặp lại được:** mọi tài nguyên, tên và cấu hình đều được ghi trong hướng dẫn và template, nên có thể dựng lại hoặc gỡ bỏ môi trường bất cứ lúc nào.
- **Quyền tối thiểu:** mỗi dịch vụ có IRSA role riêng, chỉ truy cập hàng đợi của mình và topic dùng chung. Các tầng dữ liệu được giữ riêng tư.
- **Kiểm soát chi phí:** thứ tự gỡ bỏ được ghi lại giúp xóa mọi tài nguyên tính phí khi không cần môi trường.

---

### 3. Sơ đồ kiến trúc

#### Kiến trúc tổng thể

```text
Browser --> CloudFront --> S3 (bucket SPA riêng tư)
               |
               +-- /api/*, /live* --> ALB --> EKS on Fargate (7 dịch vụ)
                                                 |-- RDS PostgreSQL x6 (private subnet)
                                                 |-- ElastiCache for Valkey
                                                 +-- SNS topic --> SQS queue x3 (+ DLQ)
```

[OWNER: thêm ảnh sơ đồ kiến trúc nếu cần]

#### Kiến trúc kết nối dịch vụ

Quy tắc: nếu bên gọi cần câu trả lời để tiếp tục thì gọi đồng bộ qua HTTP. Nếu chỉ là một việc đã xảy ra và các bên khác phản ứng sau thì phát sự kiện.

| Tương tác | Kiểu |
| --- | --- |
| Tạo tài xế kiểm tra người dùng (driver gọi user) | HTTP đồng bộ |
| Yêu cầu chuyến đi xin giữ tiền (trip gọi payment) | HTTP đồng bộ |
| Yêu cầu chuyến đi tới ghép tài xế gần nhất | Sự kiện `trip.requested` qua SNS tới hàng đợi matching |
| Chuyến đi hoàn tất hoặc bị hủy tới thu tiền hoặc nhả tiền | Sự kiện `trip.completed` và `trip.cancelled` qua SNS tới hàng đợi payment |
| Mọi sự kiện tới hộp thư thông báo | SNS tới hàng đợi notifications (không lọc) |
| Vị trí tài xế trực tiếp tới hành khách | Valkey Pub/Sub tới realtime gateway, rồi SSE qua đường dẫn ALB `/live` |
| Vị trí tài xế tới chỉ mục ghép chuyến | Valkey Stream đọc qua consumer group |

Tài nguyên messaging: SNS topic `ridenow-prod-events`, các hàng đợi `ridenow-prod-matching`, `ridenow-prod-payment` và `ridenow-prod-notifications`, mỗi hàng đợi có một dead-letter queue (`maxReceiveCount` 5), và ba subscription dùng raw message delivery.

#### Các thành phần kiến trúc

| Lớp | Dịch vụ AWS | Vai trò trong hệ thống |
| --- | --- | --- |
| Danh tính | **AWS IAM** | Hai role cho EKS (`ridenow-prod-eks-cluster-role`, `ridenow-prod-eks-fargate-pod-execution-role`), năm customer managed policy và sáu IRSA role cho add-on và dịch vụ. |
| Mạng | **Amazon VPC** | VPC `10.30.0.0/16`, hai public và hai private subnet, một internet gateway, hai NAT gateway, và các security group `ridenow-prod-cluster-sg`, `ridenow-prod-db-sg`. |
| Registry | **Amazon ECR** | Bảy repository (`ridenow/<service>`), quét khi push, tối đa 50 image, image không gắn tag hết hạn sau 14 ngày. |
| Dữ liệu | **Amazon RDS for PostgreSQL, AWS Secrets Manager, AWS KMS** | Sáu instance `ridenow-prod-<service>-db` (`db.t4g.micro`, 20 GB, Multi-AZ, sao lưu 30 ngày), mã hóa bằng `alias/ridenow-prod-rds`, secret master do RDS quản lý. |
| Messaging | **Amazon SNS, Amazon SQS** | Một topic, ba hàng đợi, ba dead-letter queue, ba subscription và queue policy chỉ cho phép topic. |
| Cache | **Amazon ElastiCache for Valkey** | Replication group `ridenow-prod-redis` (`cache.t4g.small`) cùng subnet group và `ridenow-prod-redis-sg`. |
| Tính toán | **Amazon EKS trên AWS Fargate** | Cluster `ridenow-prod` chạy Kubernetes 1.31 với các Fargate profile `ridenow-prod-system`, `ridenow-prod-app` và `ridenow-prod-observability`. |
| Ingress | **Elastic Load Balancing (ALB)** | Application Load Balancer do AWS Load Balancer Controller tạo từ Ingress `ridenow`. Năm đường dẫn API cùng `/live`. |
| Frontend | **Amazon S3, Amazon CloudFront** | Bucket riêng tư, origin access control `ridenow-prod-frontend-oac`, function `ridenow-prod-spa-rewrite` và distribution `RideNow prod frontend`. Role GitHub OIDC tùy chọn `ridenow-prod-frontend-deploy` dùng để publish site. |

---

### 4. Tiến độ & Cột mốc

Quy trình dựng theo thứ tự tạo trong các hướng dẫn. Ngày tháng và thời lượng không được ghi trong repository.

| Bước | Hạng mục công việc | Lịch trình |
| :--- | :--- | :--- |
| **1. Danh tính và mạng** | IAM role và policy; VPC, subnet, NAT gateway, route table, security group. | [OWNER: điền ngày] |
| **2. Registry, dữ liệu, messaging, cluster** | ECR repository và đẩy image; KMS, RDS và Secrets Manager; SNS, SQS và Valkey; EKS cluster và Fargate profile. Bốn nhóm này làm theo thứ tự bất kỳ sau bước 1. | [OWNER: điền ngày] |
| **3. Add-on và dịch vụ** | Quy tắc truy cập mạng của pod, OIDC provider, IRSA role; AWS Load Balancer Controller và External Secrets; triển khai bảy dịch vụ. | [OWNER: điền ngày] |
| **4. Frontend** | S3, CloudFront, role deploy tùy chọn; publish site lần đầu. Làm sau cùng, khi Ingress đã có địa chỉ ALB. | [OWNER: điền ngày] |
| **5. Kiểm tra và gỡ bỏ** | Chạy smoke test và các bước kiểm tra trong hướng dẫn; xóa tài nguyên theo thứ tự phụ thuộc ngược khi hoàn tất. | [OWNER: điền ngày] |

---

### 5. Ngân sách dự kiến

Repository nêu các dịch vụ tính phí theo giờ kể từ lúc được tạo. Repository không có số liệu giá, nên tổng chi phí để chủ báo cáo điền.

| Yếu tố chi phí | Cấu hình |
| --- | --- |
| **EKS control plane** | Một cluster, `ridenow-prod`. |
| **NAT gateway** | Hai cái, mỗi Availability Zone một cái, cùng Elastic IP. |
| **Amazon RDS for PostgreSQL** | Sáu instance `db.t4g.micro` Multi-AZ, 20 GB lưu trữ, sao lưu 30 ngày. |
| **Amazon ElastiCache for Valkey** | Một replication group `cache.t4g.small`. |
| **AWS Fargate** | Các pod trong profile system, app và observability. |
| **Application Load Balancer** | Một ALB do Ingress tạo. |
| **Dịch vụ khác** | SNS, SQS, dung lượng ECR, Secrets Manager, khóa KMS, S3 và CloudFront, tính theo mức dùng. |

| Số liệu | Giá trị |
| --- | --- |
| Chi phí hằng tháng ước tính | [OWNER: điền ước tính chi phí] |
| Hạn mức ngân sách | [OWNER: điền hạn mức ngân sách] |

---

### 6. Đánh giá rủi ro

| Rủi ro | Khả năng | Tác động | Biện pháp giảm thiểu |
| --- | --- | --- | --- |
| **Tài nguyên tính phí theo giờ bị để chạy** | [OWNER: đánh giá] | [OWNER: đánh giá] | Làm theo thứ tự gỡ bỏ. Xóa ALB của Ingress trước khi xóa VPC. |
| **Sai thứ tự tạo** | [OWNER: đánh giá] | [OWNER: đánh giá] | Tạo tài nguyên add-on trước khi cài Load Balancer Controller và External Secrets. Tạo frontend sau cùng. |
| **Lộ thông tin đăng nhập cơ sở dữ liệu** | [OWNER: đánh giá] | [OWNER: đánh giá] | Secret do RDS quản lý trong Secrets Manager, cấp qua External Secrets và IRSA. Cơ sở dữ liệu nằm trong private subnet. |
| **Điểm vào để mở** | [OWNER: đánh giá] | [OWNER: đánh giá] | ALB dùng HTTP cổng 80, không có WAF và không giới hạn nguồn. `docs/ARCHITECTURE.md` liệt kê TLS, WAF và xác thực là các điểm chặn trước khi dùng thật. |
| **Mất hoặc trùng sự kiện** | [OWNER: đánh giá] | [OWNER: đánh giá] | Dead-letter queue cho mọi hàng đợi, consumer idempotent, và sự kiện chỉ phát sau khi commit. |

---

### 7. Kết quả kỳ vọng

- **Môi trường `prod` đang chạy** tại `ap-southeast-1`: sáu cơ sở dữ liệu, một topic với ba hàng đợi và các dead-letter queue, một nhóm Valkey, một EKS cluster với ba Fargate profile, bảy image trong ECR, một ALB, và frontend trên S3 và CloudFront.
- **Tài liệu lặp lại được:** mọi tên và cấu hình tài nguyên được ghi trong hướng dẫn, và chương workshop đi qua từng bước trên từng console.
- **Gỡ bỏ gọn:** một thứ tự gỡ bỏ xóa mọi tài nguyên tính phí.

---

### 8. Tài liệu tham khảo

[1]: [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

[2]: [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)

[3]: [AWS Secrets Manager Developer Guide](https://docs.aws.amazon.com/secretsmanager/)
