---
title : "Các bước chuẩn bị"
date : 2024-01-01 
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

# 5.2. CÁC BƯỚC CHUẨN BỊ TRIỂN KHAI RIDENOW TRÊN AWS
## 5.2.1. Giới thiệu
Trước khi bắt đầu triển khai hệ thống RideNow lên AWS, em cần chuẩn bị môi trường làm việc, cài đặt các công cụ cần thiết và kiểm tra quyền truy cập vào tài khoản AWS.
Mục đích của giai đoạn này là đảm bảo máy tính có thể kết nối và tương tác với các dịch vụ AWS, đồng thời thống nhất các thông số cấu hình được sử dụng xuyên suốt quá trình triển khai.
Trong phần này, em thực hiện các công việc chính bao gồm:
- Chuẩn bị các công cụ triển khai.
- Thiết lập môi trường Bash trên Windows.
- Kiểm tra thông tin xác thực AWS.
- Cấu hình Region và môi trường triển khai.
- Khai báo và kiểm tra các biến dùng chung.

---
## 5.2.2. Chuẩn bị các công cụ triển khai
Đầu tiên, em tiến hành cài đặt và kiểm tra các công cụ cần thiết để phục vụ việc triển khai hệ thống RideNow.
Các công cụ được sử dụng bao gồm:

|Công cụ|Chức năng|
|---|---|
|AWS CLI v2|Tương tác và quản lý các tài nguyên AWS thông qua dòng lệnh|
|Docker|Đóng gói các service thành Docker image và đẩy lên Amazon ECR|
|kubectl|Quản lý các tài nguyên Kubernetes trên Amazon EKS|
|Helm|Cài đặt và quản lý các ứng dụng trên Kubernetes|
|jq|Đọc và xử lý dữ liệu JSON|
|Bash|Thực thi các câu lệnh triển khai|
Trong đó, AWS CLI được sử dụng để tương tác với AWS; Docker phục vụ quá trình đóng gói ứng dụng; còn kubectl và Helm được sử dụng để quản lý các thành phần Kubernetes.
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-deployment-tools-overview.png)
Việc chuẩn bị đầy đủ những công cụ này giúp quá trình triển khai diễn ra thống nhất và hạn chế các lỗi phát sinh do thiếu công cụ.

**Kiểm tra các công cụ bằng lệnh:**
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-check-tool-versions.png)

---
## 5.2.3. Thiết lập môi trường Bash trên Windows
Do các câu lệnh trong workshop sử dụng cú pháp Bash, trong khi máy tính của em chạy hệ điều hành Windows, nên em sử dụng Windows Subsystem for Linux, gọi tắt là WSL.
WSL cho phép chạy môi trường Linux trực tiếp trên Windows mà không cần cài đặt hệ điều hành Linux riêng biệt.
Trong workshop này, em sử dụng Ubuntu thông qua WSL để thực hiện các lệnh AWS CLI và các thao tác triển khai.
Để cài đặt Ubuntu, em mở PowerShell với quyền Administrator và thực hiện:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-install-ubuntu-wsl.png)
Sau khi cài đặt, em mở Ubuntu Terminal và kiểm tra môi trường bằng lệnh:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-verify-linux-bash-environment.png)
Nếu kết quả hiển thị thông tin Linux thì môi trường Bash đã hoạt động.

---
## 5.2.4. Kiểm tra danh tính và quyền truy cập AWS
Sau khi chuẩn bị môi trường làm việc, em tiến hành kiểm tra thông tin xác thực AWS.
Mục đích của bước này là xác nhận AWS CLI đang sử dụng đúng tài khoản và danh tính IAM được cấp quyền để thực hiện các thao tác triển khai.
Em chạy câu lệnh:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-aws-sts-get-caller-identity-command.png)

![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-aws-sts-caller-identity-result.png)
Lệnh này sử dụng dịch vụ AWS Security Token Service để truy vấn thông tin danh tính hiện tại.
Kết quả trả về gồm ba trường chính:
- **UserId:** Mã định danh của danh tính AWS hiện tại.
- **Account:** ID tài khoản AWS đang sử dụng.
- **Arn:** Định danh đầy đủ của người dùng hoặc vai trò IAM.
Thông qua các thông tin này, em xác nhận mình đang thao tác trên đúng tài khoản AWS.
Ngoài ra, em sử dụng danh tính IAM có quyền quản trị phù hợp thay vì sử dụng trực tiếp tài khoản root, nhằm giảm thiểu rủi ro bảo mật.

---
## 5.2.5. Cấu hình AWS Region và môi trường triển khai
Sau khi kiểm tra thành công quyền truy cập, em tiến hành cấu hình khu vực triển khai cho hệ thống RideNow.
Trong workshop này, em sử dụng AWS Region:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-aws-region-singapore.png)
Đây là Region Singapore của AWS.
Em thiết lập Region mặc định bằng câu lệnh:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-configure-default-aws-region.png)
Sau đó kiểm tra lại:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-verify-aws-region.png)

Bên cạnh đó, hệ thống RideNow sử dụng môi trường `prod`, đại diện cho môi trường production.
Việc thống nhất Region và môi trường triển khai giúp các tài nguyên AWS được quản lý nhất quán, tránh trường hợp tạo nhầm tài nguyên ở các Region khác nhau.

---
## 5.2.6. Khai báo các biến cấu hình dùng chung
Thay vì nhập thủ công các thông tin như ID tài khoản AWS, Region hoặc tên cụm Kubernetes trong từng câu lệnh, em sử dụng biến Bash để lưu trữ và tái sử dụng các giá trị này.
Đầu tiên, em lấy ID tài khoản AWS:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-set-aws-account-id-variable.png)

Tiếp theo, em lấy Region đã cấu hình:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-set-aws-region-variable.png)

Sau đó, em khai báo các biến còn lại:
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-set-deployment-environment-variables.png)

Ý nghĩa của từng biến như sau:

|Biến|Ý nghĩa|
|---|---|
|ACCOUNT_ID|ID tài khoản AWS hiện tại|
|AWS_REGION|Region triển khai hệ thống|
|AWS_DEFAULT_REGION|Region mặc định cho AWS CLI|
|ENV|Tên môi trường triển khai|
|CLUSTER_NAME|Tên cụm Amazon EKS|
|IMAGE_TAG|Phiên bản Docker image|
Trong đó, `CLUSTER_NAME` được thiết lập là `ridenow-prod`, còn `IMAGE_TAG` được đặt là `v1.0.0`.
Việc sử dụng các biến cấu hình giúp các câu lệnh triển khai ngắn gọn, dễ quản lý và hạn chế sai sót khi nhập dữ liệu.

---
## 5.2.7. Kiểm tra kết quả khai báo biến
Sau khi khai báo các biến, em tiến hành kiểm tra lại toàn bộ giá trị bằng các câu lệnh `echo`.
![Ảnh minh họa các bước chuẩn bị RideNow](/fcj-workshop-ridenow/images/5.2-verify-deployment-variables.png)

Trong đó, ACCOUNT_ID sẽ hiển thị ID tài khoản AWS thực tế của em.
Thông qua kết quả này, em xác nhận các biến đã được khai báo đúng và có thể sử dụng trong những bước triển khai tiếp theo.

Ngoài ra, em cũng kiểm tra trường hợp ACCOUNT_ID hoặc AWS_REGION bị rỗng. Nếu có thông báo `STOP`, em sẽ kiểm tra lại thông tin xác thực hoặc cấu hình Region trước khi tiếp tục.
Một điểm cần lưu ý là các biến Bash chỉ tồn tại trong phiên shell hiện tại. Vì vậy, nếu mở phiên terminal mới, em cần thực hiện lại phần khai báo biến.

---
## 5.2.9. Kết quả đạt được
Sau khi hoàn thành các bước chuẩn bị, em đã thiết lập được môi trường làm việc phục vụ cho quá trình triển khai hệ thống RideNow lên AWS.
Cụ thể, các kết quả đạt được bao gồm:
1. Chuẩn bị đầy đủ các công cụ AWS CLI, Docker, kubectl, Helm, jq và Bash.
2. Thiết lập môi trường Ubuntu thông qua WSL trên Windows.
3. Kiểm tra thành công danh tính AWS bằng AWS CLI.
4. Cấu hình Region triển khai là `ap-southeast-1`.
5. Thiết lập môi trường `prod` và tên cụm `ridenow-prod`.
6. Khai báo và kiểm tra thành công các biến cấu hình dùng chung.
Như vậy, máy tính đã được chuẩn bị để thực hiện các bước triển khai hạ tầng và ứng dụng RideNow trong những phần tiếp theo.
Ở giai đoạn này, em mới hoàn thành việc chuẩn bị môi trường, chưa trực tiếp tạo cụm EKS hay triển khai các service của hệ thống.

---
## 5.2.10. Kết luận
Tóm lại, phần 5.2 tập trung vào việc chuẩn bị các điều kiện cần thiết trước khi triển khai hệ thống RideNow lên AWS.
Thông qua việc cài đặt công cụ, thiết lập môi trường Bash, kiểm tra danh tính IAM và cấu hình các biến dùng chung, em đã xây dựng được một môi trường làm việc thống nhất để thực hiện những bước triển khai tiếp theo.
Đây là bước nền tảng quan trọng nhằm đảm bảo các thao tác triển khai hạ tầng và ứng dụng trên AWS được thực hiện đúng tài khoản, đúng Region và đúng cấu hình.
Sau khi hoàn tất phần chuẩn bị, em sẽ tiếp tục chuyển sang giai đoạn triển khai các tài nguyên hạ tầng AWS cho hệ thống RideNow.
