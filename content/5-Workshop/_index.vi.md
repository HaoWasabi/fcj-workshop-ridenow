---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Triển khai RideNow trên AWS

### Tổng quan

Workshop này hướng dẫn tạo và cấu hình, lần lượt trong từng service console, các dịch vụ AWS mà **RideNow** chạy trên đó. Bạn xây dựng môi trường production (`prod`) tại region `ap-southeast-1` (Singapore). Workshop chỉ bao gồm phần thiết lập AWS và không giải thích mã nguồn ứng dụng.

Mọi trang đều dùng AWS Management Console để tạo tài nguyên. AWS CLI, `kubectl` và Helm chỉ xuất hiện ở những bước console không làm được, như kubeconfig, cài add-on, Kubernetes secret, build và push image, và các bước kiểm tra. Mọi lệnh đều là Bash (dùng WSL trên Windows).

> Chi phí: EKS control plane, NAT gateway, RDS Multi-AZ và ElastiCache tính tiền theo giờ kể từ lúc tạo. Hãy kết thúc bằng mục 5.11 Dọn dẹp tài nguyên.

### Tổng quan kiến trúc

Hệ thống được tổ chức theo các lớp sau:

| Lớp | Dịch vụ AWS |
|-----|-------------|
| Định danh | IAM role và policy |
| Mạng | VPC, subnet, internet gateway, NAT gateway, route table, security group |
| Container image | Amazon ECR |
| Dữ liệu | Amazon RDS PostgreSQL, AWS Secrets Manager, AWS KMS |
| Messaging và cache | Amazon SNS, Amazon SQS, Amazon ElastiCache for Valkey |
| Compute | Amazon EKS với AWS Fargate |
| Ingress | Application Load Balancer do AWS Load Balancer Controller tạo |
| Frontend | Amazon S3 với Amazon CloudFront |

### Những gì bạn xây dựng

| Hạng mục | Số lượng |
|----------|----------|
| RDS PostgreSQL instance (mỗi service một instance) | 6 |
| ECR repository | 7 |
| SNS topic (`ridenow-prod-events`) | 1 |
| SQS queue, mỗi queue có một dead-letter queue | 3 queue, 3 dead-letter queue |
| Subnet | 2 public, 2 private |
| Fargate profile | 3 |

### Nội dung

1. [Giới thiệu](5.1-Workshop-overview/)
2. [Các bước chuẩn bị](5.2-Prerequiste/)
3. [IAM](5.3-IAM/)
4. [Network](5.4-Network/)
5. [ECR](5.5-ECR/)
6. [Data](5.6-Data/)
7. [Messaging](5.7-Messaging/)
8. [EKS Cluster](5.8-EKS-Cluster/)
9. [EKS Addons](5.9-EKS-Addons/)
10. [Frontend](5.10-Frontend/)
11. [Dọn dẹp tài nguyên](5.11-Cleanup/)

### Thứ tự tạo tài nguyên

Thực hiện lần lượt từ 5.3 đến 5.10, sau đó dọn dẹp ở 5.11 khi hoàn tất. Có hai quy tắc về thứ tự:

- **EKS Addons (5.9) làm sau EKS Cluster (5.8) và trước khi cài AWS Load Balancer Controller và External Secrets.** Các IRSA role tạo ở 5.9 chính là role mà hai bản cài này chạy bằng. Vì vậy: tạo cluster ở 5.8, tạo tài nguyên của 5.9, quay lại các bước post-deploy của 5.8 để cài hai add-on, rồi triển khai bảy service từ 5.9.
- **Frontend (5.10) làm sau cùng,** sau khi các service đã được triển khai và Ingress `ridenow` đã có địa chỉ ALB, vì CloudFront dùng ALB đó làm API origin.

Data (5.6), Messaging (5.7) và EKS Cluster (5.8) chỉ phụ thuộc vào IAM (5.3) và Network (5.4). Khi hai phần này xong, bạn có thể xây ba phần còn lại theo thứ tự bất kỳ.
