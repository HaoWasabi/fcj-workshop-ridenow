---
title: "Worklog Tuần 7"
date: 2026-11-30
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Mục tiêu tuần 7:

* Host frontend trên S3 và CloudFront và thiết lập cách triển khai nó.
* Làm cho môi trường dễ xóa và dựng lại.
* Biến những gì đã dựng thành các guide từng bước trên console và giao phần cho cả nhóm.

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Tài liệu tham khảo |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Tạo S3 bucket riêng tư, origin access control và CloudFront distribution. <br> - Định tuyến `/api/*` và `/live*` tới load balancer. | 30/11/2026 | 30/11/2026 | [Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) |
| 3 | - Thiết lập GitHub Actions với OIDC role để publish site mà không cần access key. <br> - Thử publish và rollback. | 01/12/2026 | 01/12/2026 | [GitHub OIDC with AWS](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) |
| 4 | - Viết thứ tự xóa tài nguyên và thử trên một môi trường đầy đủ. <br> - Dựng lại toàn bộ từ đầu để chứng minh các bước lặp lại được. | 02/12/2026 | 02/12/2026 | |
| 5 | - Viết tám guide trên console, mỗi tầng một guide, từ các template CloudFormation của mình. <br> - Giao guide: Nhi lo IAM, network và ECR, Hào lo data, Thuận lo messaging, Đào lo EKS cluster và add-ons. | 03/12/2026 | 03/12/2026 | |
| 6 | - Viết script kiểm tra guide để lỗi lộ ra sớm. <br> - Trả lời những câu hỏi đầu tiên của các bạn khi bắt đầu viết trang của mình. | 04/12/2026 | 04/12/2026 | |

### Kết quả tuần 7:

&emsp;◉ Thứ Hai:
<br>&emsp;&emsp;○ Site mở được qua CloudFront ngay lần đầu sau khi mình sửa origin path.

&emsp;◉ Thứ Ba:
<br>&emsp;&emsp;○ Publish hoàn toàn không cần key. Mình thích cảm giác gọn gàng đó.

&emsp;◉ Thứ Tư:
<br>&emsp;&emsp;○ Lần xóa đầu tiên còn sót một load balancer nên không xóa được VPC. Mình sửa thứ tự và ghi chú in đậm.

&emsp;◉ Thứ Năm:
<br>&emsp;&emsp;○ Ai cũng có phần guide rõ ràng. Chia theo tầng giúp dễ nói ai sở hữu phần nào.

&emsp;◉ Thứ Sáu:
<br>&emsp;&emsp;○ Script đã bắt được vài tên bị thiếu trong các bản nháp đầu.