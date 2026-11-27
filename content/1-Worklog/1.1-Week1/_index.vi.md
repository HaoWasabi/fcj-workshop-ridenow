---
title: "Worklog Tuần 1"
date: 2026-10-19
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Mục tiêu tuần 1:

* Làm quen với team, các mentor và cách vận hành của kỳ thực tập.
* Có tài khoản AWS dùng được, bật MFA, tạo IAM user và cảnh báo ngân sách để không bị bất ngờ về hóa đơn.
* Nắm các khái niệm cốt lõi: Region, Availability Zone, VPC, IAM và việc chạy container trên AWS nghĩa là gì.
* Cùng team bàn ý tưởng và chốt một dự án để làm trong tám tuần.

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Tài liệu tham khảo |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Buổi gặp đầu tiên với mentor và bốn bạn trong nhóm (Thuận, Đào, Hào, Nhi). <br> - Nghe phổ biến quy định thực tập, lịch trình đến 11/12 và cách báo cáo tiến độ. <br> - Tạo nhóm chat và repository chung của team. | 19/10/2026 | 19/10/2026 | [FCJ - Explore AWS Services](https://cloudjourney.awsstudygroup.com/1-explore/) |
| 3 | - Tạo tài khoản AWS, bật MFA cho root user và tạo một IAM user admin để dùng hằng ngày. <br> - Thiết lập AWS Budgets và billing alarm. <br> - Đọc về Region và Availability Zone. | 20/10/2026 | 20/10/2026 | [Create Your First AWS Account](https://000001.awsstudygroup.com/) |
| 4 | - Học căn bản về VPC: subnet, route table, internet gateway, NAT gateway. <br> - Tự bấm tạo một VPC đầu tiên trên console để xem các phần ghép với nhau thế nào. | 21/10/2026 | 21/10/2026 | [Amazon VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) |
| 5 | - Học IAM cho đàng hoàng: user, role, policy và vì sao dùng role tốt hơn access key. <br> - Mỗi thành viên ghi ra hai, ba ý tưởng dự án. | 22/10/2026 | 22/10/2026 | [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |
| 6 | - Cả nhóm brainstorm: so sánh ý tưởng theo tiêu chí "có làm xong trong tám tuần không" và "có dùng đủ nhiều dịch vụ AWS không". <br> - Chốt làm nền tảng gọi xe và đặt tên là RideNow. <br> - Thống nhất mình làm trưởng nhóm và phụ trách phần cloud. | 23/10/2026 | 23/10/2026 | |

### Kết quả tuần 1:

&emsp;◉ Thứ Hai:
<br>&emsp;&emsp;○ Gặp mọi người, dựng repository và nhóm chat.

&emsp;◉ Thứ Ba:
<br>&emsp;&emsp;○ Tài khoản, MFA, admin user và cảnh báo ngân sách đã xong trước khi đụng tới bất kỳ dịch vụ nào khác.

&emsp;◉ Thứ Tư:
<br>&emsp;&emsp;○ Mình chỉ thật sự hiểu VPC sau khi vẽ ra giấy rồi tự dựng bằng tay một lần.

&emsp;◉ Thứ Năm:
<br>&emsp;&emsp;○ Hiểu vì sao không bao giờ để access key trong code, và điều này về sau định hình cách cả dự án xử lý thông tin xác thực.

&emsp;◉ Thứ Sáu:
<br>&emsp;&emsp;○ Nhóm chọn RideNow. Gọi xe hợp vì cần nhiều dịch vụ nói chuyện với nhau, một lý do tốt để dùng AWS cho nghiêm túc.
<br>&emsp;&emsp;○ Mình nhận vai trò trưởng nhóm. Hơi run nên mình ghi sẵn những việc cần quyết định trong tuần sau.