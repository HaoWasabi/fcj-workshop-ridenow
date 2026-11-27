---
title: "Worklog Tuần 3"
date: 2026-11-02
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Mục tiêu tuần 3:

* Thiết kế phần code để năm người làm song song mà không giẫm chân nhau.
* Thống nhất API contract, sự kiện và quyền sở hữu database.
* Chạy được cả hệ thống trên một máy trước khi đụng tới cloud.

| Thứ | Công việc | Ngày bắt đầu | Ngày hoàn thành | Tài liệu tham khảo |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Dựng cấu trúc repository, quy tắc nhánh và thói quen tạo pull request. <br> - Viết hướng dẫn ngắn về cách tổ chức một service. | 02/11/2026 | 02/11/2026 | |
| 3 | - Viết API contract cho từng service để frontend và các service khác biết cần mong đợi gì. <br> - Định nghĩa tên sự kiện và dữ liệu mà mỗi sự kiện mang theo. | 03/11/2026 | 03/11/2026 | |
| 4 | - Vẽ schema database cho từng service và gom vào một file DBML. <br> - Rà soát schema cùng từng bạn. | 04/11/2026 | 04/11/2026 | [DBML](https://dbml.dbdiagram.io/docs/) |
| 5 | - Viết cấu hình Docker Compose: PostgreSQL cho từng service, Valkey và LocalStack để giả lập SNS, SQS. <br> - Giúp các bạn chạy được service của mình trong container. | 05/11/2026 | 05/11/2026 | [Docker Compose](https://docs.docker.com/compose/) <br> [LocalStack](https://docs.localstack.cloud/) |
| 6 | - Review các pull request đầu tiên. <br> - Chạy toàn bộ hệ thống ở máy local lần đầu và ghi lại những chỗ lỗi. | 06/11/2026 | 06/11/2026 | |

### Kết quả tuần 3:

&emsp;◉ Thứ Hai:
<br>&emsp;&emsp;○ Quy tắc nhánh đơn giản và pull request nhỏ. Nhờ vậy sau này đỡ đau đầu khi merge rất nhiều.

&emsp;◉ Thứ Ba:
<br>&emsp;&emsp;○ Các contract là tài liệu hữu ích nhất tuần, vì các bạn có thể làm mà không phải chờ nhau.

&emsp;◉ Thứ Tư:
<br>&emsp;&emsp;○ Một file schema cho mọi database. Ngồi rà từng người mất cả ngày, nhưng bắt được hai service định đọc thẳng bảng của nhau.

&emsp;◉ Thứ Năm:
<br>&emsp;&emsp;○ LocalStack cho phép thử luồng sự kiện mà không tốn đồng nào trên AWS.

&emsp;◉ Thứ Sáu:
<br>&emsp;&emsp;○ Lần chạy local đầy đủ đầu tiên. Ba service không khởi động được do sai biến môi trường. Cả nhóm sửa xong trước giờ nghỉ trưa và ghi lại tên các biến để dùng cho các guide.