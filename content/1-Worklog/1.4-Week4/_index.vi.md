---
title: "Worklog Tuần 4"
date: 2026-11-09
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Mục tiêu tuần 4:

* Đóng gói container cho mọi service và đẩy image lên registry.
* Thiết kế mạng và quyền trên AWS làm nền cho mọi thứ còn lại.
* Bắt đầu chuyển các thao tác trên console thành template CloudFormation.
* Tiếp tục hỗ trợ các bạn hoàn thiện service.

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Tài liệu tham khảo |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Cùng các bạn xem lại từng Dockerfile để image gọn nhẹ và chạy bằng user thường. <br> - Build cả bảy image ở máy local. | 09/11/2026 | 09/11/2026 | [Dockerfile best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) |
| 3 | - Thiết kế mạng: một VPC, hai public subnet và hai private subnet trên hai Availability Zone, mỗi zone một NAT gateway. <br> - Viết rule security group cho cluster và database. | 10/11/2026 | 10/11/2026 | [VPC and subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html) |
| 4 | - Viết các template CloudFormation đầu tiên: mạng và IAM role cho cluster. <br> - Triển khai trên tài khoản thật và sửa lỗi. | 11/11/2026 | 11/11/2026 | [CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) |
| 5 | - Viết template ECR gồm bảy repository, quét khi push và lifecycle rule. <br> - Đẩy các image đầu tiên. | 12/11/2026 | 12/11/2026 | [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) |
| 6 | - Thống nhất quy tắc đặt tên để mọi tài nguyên đều có dạng `ridenow-prod-...`. <br> - Họp tiến độ hằng tuần với mentor và cập nhật kế hoạch. | 13/11/2026 | 13/11/2026 | |

### Kết quả tuần 4:

&emsp;◉ Thứ Hai:
<br>&emsp;&emsp;○ Build xong bảy image. Hai image nặng hơn một gigabyte nên cả nhóm cắt bớt.

&emsp;◉ Thứ Ba:
<br>&emsp;&emsp;○ Thiết kế mạng xong. Mình đã đánh giá thấp chi phí NAT gateway mỗi giờ, nên ghi nó vào phần ghi chú ngân sách.

&emsp;◉ Thứ Tư:
<br>&emsp;&emsp;○ Template đầu tiên lỗi ba lần vì thiếu dependency và sai ARN của policy. Từ đó mình đọc CloudFormation events trước khi đoán mò.

&emsp;◉ Thứ Năm:
<br>&emsp;&emsp;○ Image đã nằm trong ECR và có tag.

&emsp;◉ Thứ Sáu:
<br>&emsp;&emsp;○ Một quy tắc đặt tên giúp tránh rất nhiều nhầm lẫn về sau, khi các trang workshop phải gọi tên từng tài nguyên.