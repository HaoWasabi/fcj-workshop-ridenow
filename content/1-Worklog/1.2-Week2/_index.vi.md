---
title: "Worklog Tuần 2"
date: 2026-10-26
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Mục tiêu tuần 2:

* Biến ý tưởng RideNow thành bản yêu cầu bằng văn bản mà cả nhóm đồng ý.
* Quyết định ranh giới giữa các service và dữ liệu mà mỗi service sở hữu.
* Phác thảo kiến trúc đầu tiên trên AWS và chọn cách chạy các service.
* Chia việc cho năm người.

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Tài liệu tham khảo |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Viết danh sách đầu tiên về những gì hành khách, tài xế và admin cần làm được. <br> - Cắt gọn xuống phiên bản có thể làm kịp trong thời gian cho phép. | 26/10/2026 | 26/10/2026 | |
| 3 | - Tách hệ thống thành bảy service: user, driver, trip, payment, matching, notification và một realtime gateway. <br> - Quyết định mỗi service sở hữu một database riêng. | 27/10/2026 | 27/10/2026 | [Microservices on AWS](https://docs.aws.amazon.com/whitepapers/latest/microservices-on-aws/microservices-on-aws.html) |
| 4 | - Xác định cuộc gọi nào phải là HTTP đồng bộ, việc nào nên là sự kiện. <br> - Viết quy tắc đơn giản để áp dụng và danh sách các sự kiện. | 28/10/2026 | 28/10/2026 | [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) <br> [Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) |
| 5 | - So sánh các cách chạy container (EC2, Fargate, EKS) và chọn EKS trên Fargate. <br> - Vẽ sơ đồ kiến trúc đầu tiên gồm VPC, private subnet, RDS, cache và load balancer. | 29/10/2026 | 29/10/2026 | [Amazon EKS on AWS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html) |
| 6 | - Chia việc: các bạn viết service, mình phụ trách phần cloud và điều phối. <br> - Viết kế hoạch tám tuần và xem lại cùng mentor. | 30/10/2026 | 30/10/2026 | |

### Kết quả tuần 2:

&emsp;◉ Thứ Hai:
<br>&emsp;&emsp;○ Danh sách yêu cầu đầu tiên quá lớn. Cắt bớt tính năng khó hơn mình nghĩ nhiều so với việc viết ra chúng.

&emsp;◉ Thứ Ba:
<br>&emsp;&emsp;○ Bảy service, mỗi service một database. Nghe thì nhiều cho năm người, nhưng mỗi service đều nhỏ.

&emsp;◉ Thứ Tư:
<br>&emsp;&emsp;○ Có một quy tắc ai cũng nhớ được: nếu bên gọi cần câu trả lời để đi tiếp thì gọi HTTP, còn nếu chỉ là một việc vừa xảy ra và bên khác có thể phản ứng sau thì phát sự kiện.

&emsp;◉ Thứ Năm:
<br>&emsp;&emsp;○ Chọn EKS trên Fargate để không ai phải vá server. Mình biết đây sẽ là phần khó nhất với mình, nhưng cũng là phần học được nhiều nhất.

&emsp;◉ Thứ Sáu:
<br>&emsp;&emsp;○ Cuối ngày ai cũng biết phần việc của mình. Mentor nhắc cả nhóm nghĩ đến chi phí từ sớm nên mình thêm việc đó vào danh sách.