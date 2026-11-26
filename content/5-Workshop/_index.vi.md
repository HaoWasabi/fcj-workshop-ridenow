---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# DevOps on AWS with RakiBookery - Bookery Website

### Tổng quan

Workshop này hướng dẫn toàn bộ quy trình xây dựng, triển khai và vận hành **RakiBookery** - website nhà sách online - trên nền tảng Amazon Web Services (AWS). Dự án áp dụng mô hình Cloud & DevOps hiện đại với pipeline CI/CD tự động hoàn toàn, khả năng sẵn sàng cao theo Multi-AZ, và hệ thống giám sát toàn diện.

Workshop được chia thành bốn giai đoạn chính:

- **Hạ tầng**: Thiết lập mạng và các dịch vụ AWS nền tảng (VPC, RDS, S3, IAM)
- **Triển khai**: Xây dựng pipeline CI/CD bằng GitHub Actions và triển khai ứng dụng lên ECS Fargate
- **Vận hành**: Cấu hình auto-scaling, phân phối CDN, giám sát, cảnh báo chi phí và kiểm thử end-to-end
- **Hình ảnh minh họa**: Danh sách tham chiếu toàn bộ ảnh chụp màn hình sử dụng trong workshop

### Tổng quan kiến trúc

Hệ thống được tổ chức theo năm lớp chính:

| Lớp | Thành phần |
|-----|-----------|
| CI/CD | GitHub Actions, Docker, Amazon ECR |
| Presentation | Amazon CloudFront, Amazon S3 (Frontend) |
| Application | Application Load Balancer, Amazon ECS Fargate |
| Data | Amazon RDS MySQL (Multi-AZ) |
| Monitoring | Amazon CloudWatch, Amazon SNS, AWS Budgets |

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
10. [Dọn dẹp tài nguyên](5.10-Cleanup/)