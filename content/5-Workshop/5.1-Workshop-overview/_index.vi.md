---
title : "Giới thiệu"
date : 2024-01-01 
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

#### Purpose of this chapter

Chương này hướng dẫn tạo và cấu hình, trên AWS Management Console, các dịch vụ AWS mà môi trường production của RideNow (`prod`, region `ap-southeast-1`) sử dụng. Mỗi phần xử lý một lớp hạ tầng, từ IAM đến lưu trữ frontend bằng S3 và CloudFront, và kết thúc bằng một bước kiểm tra để xác nhận lớp đó hoạt động.

#### Scope

Chương chỉ bao gồm việc tạo và cấu hình các dịch vụ AWS: tên của từng tài nguyên, cách thiết lập, tài nguyên phụ thuộc, và các giá trị mà phần sau dùng lại.

Chương không bao gồm mã nguồn ứng dụng. Chương không mô tả, trích dẫn hay giải thích mã nguồn của các dịch vụ RideNow hoặc frontend, các endpoint hay định dạng thông điệp của chúng.

Các dịch vụ ứng dụng RideNow chỉ xuất hiện với vai trò là bên sử dụng tài nguyên AWS. Ví dụ, `ridenow/trip-service` là một ECR repository, và `ridenow-prod-trip-service` là IAM role mà trip service dùng để truy cập SNS và SQS.

#### What you will build

| Layer | AWS services |
|---|---|
| Identity | IAM roles và policies |
| Network | VPC, subnets, NAT gateways, security groups |
| Images | ECR repositories |
| Data | RDS PostgreSQL, Secrets Manager, KMS |
| Messaging and cache | SNS, SQS, ElastiCache for Valkey |
| Compute | EKS với Fargate |
| Ingress | Application Load Balancer thông qua AWS Load Balancer Controller |
| Frontend | S3 và CloudFront |

#### How the chapter is organized

Các phần 5.3 đến 5.10 mỗi phần tạo một lớp hạ tầng và dùng cùng các đề mục: set variables, what it creates, depends on, create the resources, values to record, post-deploy steps và verify. Phần 5.11 xóa toàn bộ theo thứ tự ngược lại để dừng phát sinh chi phí.

#### Next step

Chuyển sang [5.2 Prerequisites](../5.2-Prerequiste/) để chuẩn bị tài khoản, công cụ và đặt các biến dùng chung cho mọi phần sau.
