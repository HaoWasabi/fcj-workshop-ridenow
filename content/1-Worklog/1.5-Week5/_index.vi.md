---
title: "Worklog Tuần 5"
date: 2026-11-16
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Mục tiêu tuần 5:

* Dựng tầng dữ liệu: database, secret, mã hóa.
* Dựng tầng nhắn tin: SNS, SQS và cache cho vị trí tài xế theo thời gian thực.
* Kiểm tra sự kiện chảy đúng như đã thiết kế.

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Tài liệu tham khảo |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Tạo KMS key và DB subnet group. <br> - Tạo instance RDS PostgreSQL đầu tiên bằng tay và kiểm tra nó chỉ nhận kết nối từ bên trong VPC. | 16/11/2026 | 16/11/2026 | [Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) <br> [AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) |
| 3 | - Tạo năm database còn lại. <br> - Dùng master secret do RDS quản lý trong Secrets Manager để không ai phải nhìn hay gõ mật khẩu. | 17/11/2026 | 17/11/2026 | [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) |
| 4 | - Tạo SNS topic, ba queue kèm dead-letter queue và các subscription có filter. <br> - Viết queue policy để chỉ topic mới gửi được tin. | 18/11/2026 | 18/11/2026 | [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) |
| 5 | - Tạo Valkey replication group cho trạng thái tài xế, tra cứu vị trí và luồng vị trí. <br> - Mở đúng rule security group cho nó. | 19/11/2026 | 19/11/2026 | [Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html) |
| 6 | - Gửi sự kiện thử qua topic và xem chúng vào đúng queue. <br> - Xem lại các trang workshop mà Hào và Thuận bắt đầu viết. | 20/11/2026 | 20/11/2026 | |

### Kết quả tuần 5:

&emsp;◉ Thứ Hai:
<br>&emsp;&emsp;○ Database đầu tiên chạy. Multi-AZ tạo rất lâu, nên mình học cách bấm tạo rồi đi làm việc khác.

&emsp;◉ Thứ Ba:
<br>&emsp;&emsp;○ Sáu database, sáu secret, không có mật khẩu nào nằm trong ghi chú của nhóm.

&emsp;◉ Thứ Tư:
<br>&emsp;&emsp;○ Một tin nhắn không đến nơi vì queue policy của mình ghi sai ARN của topic. Dead-letter queue giúp mình nhận ra lỗi này.

&emsp;◉ Thứ Năm:
<br>&emsp;&emsp;○ Cache chỉ truy cập được từ security group của cluster.

&emsp;◉ Thứ Sáu:
<br>&emsp;&emsp;○ Sự kiện chảy đúng như hình vẽ trên bảng ở tuần 2, cảm giác rất đã.