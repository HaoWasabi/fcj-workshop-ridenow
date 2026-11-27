---
title: "Worklog Tuần 6"
date: 2026-11-23
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Mục tiêu tuần 6:

* Tạo cluster EKS trên Fargate và chuẩn bị sẵn sàng nhận traffic thật.
* Chỉ cấp cho mỗi workload đúng những quyền nó cần.
* Triển khai bảy service và public ra ngoài qua một load balancer.

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Tài liệu tham khảo |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Tạo cluster, KMS key của nó và ba Fargate profile. <br> - Kết nối bằng kubectl và xem những gì chạy sẵn. | 23/11/2026 | 23/11/2026 | [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) |
| 3 | - Patch CoreDNS để chạy trên Fargate. <br> - Tạo OIDC provider và IAM role cho service account (IRSA). | 24/11/2026 | 24/11/2026 | [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) |
| 4 | - Cài AWS Load Balancer Controller và External Secrets bằng Helm. <br> - Mở cổng database và cache cho các pod. | 25/11/2026 | 25/11/2026 | [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) |
| 5 | - Triển khai bảy service bằng manifest và kiểm tra pod. <br> - Sửa các pod bị Pending hoặc crash. | 26/11/2026 | 26/11/2026 | |
| 6 | - Tạo Ingress, chờ ALB và chạy API test qua nó. <br> - Ghi lại mọi lệnh đã chạy được, theo đúng thứ tự. | 27/11/2026 | 27/11/2026 | |

### Kết quả tuần 6:

&emsp;◉ Thứ Hai:
<br>&emsp;&emsp;○ Tạo xong cluster. Control plane mất khoảng mười phút và mình ngồi nhìn màn hình gần hết thời gian đó.

&emsp;◉ Thứ Ba:
<br>&emsp;&emsp;○ CoreDNS cứ Pending cho tới khi mình patch cho Fargate. Mình mất nửa ngày mới tìm ra.

&emsp;◉ Thứ Tư:
<br>&emsp;&emsp;○ Controller và secrets operator đã chạy. Mình rút ra thứ tự các bước rất quan trọng: IAM role phải có trước khi cài.

&emsp;◉ Thứ Năm:
<br>&emsp;&emsp;○ Ban đầu pod không đọc được secret của database. Nguyên nhân là thiếu một rule security group và sai tên service account.

&emsp;◉ Thứ Sáu:
<br>&emsp;&emsp;○ API test chạy qua ALB thành công. Mình mừng, rồi lập tức nghĩ đến hóa đơn và viết ghi chú dọn dẹp.