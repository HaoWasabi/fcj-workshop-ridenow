---
title : "Networking"
date : 2024-01-01 
weight : 4 
chapter : false
pre : " <b> 5.4. </b> "
---

## 1. Giới thiệu tổng quan

Mục tiêu chính của phần này là tạo ra một môi trường mạng riêng biệt, có khả năng kết nối Internet và đảm bảo an toàn cho các dịch vụ trong hệ thống.
Trong phần này, em đã cấu hình một VPC, bốn Subnet, một Internet Gateway, hai NAT Gateway, ba Route Table và hai Security Group.
Các tài nguyên này sẽ tạo thành nền tảng mạng để triển khai những thành phần tiếp theo như EKS, cơ sở dữ liệu RDS và hệ thống cache.
Toàn bộ tài nguyên được triển khai trong Region Singapore, với hai Availability Zone là ap-southeast-1a và ap-southeast-1b.

## 2. Tạo VPC – Mạng riêng của hệ thống

### 2.1. Giới thiệu VPC

Đầu tiên, em tiến hành xây dựng mạng riêng cho hệ thống RideNow bằng cách sử dụng dịch vụ VPC trên AWS.

Trong hệ thống RideNow, VPC đóng vai trò là nền tảng mạng để triển khai những thành phần tiếp theo như EKS, cơ sở dữ liệu RDS và hệ thống cache.

### 2.2. Các bước tạo VPC

Để bắt đầu, em đăng nhập vào AWS Management Console và chọn Region **Asia Pacific (Singapore)**, có mã là `ap-southeast-1`.
Sau đó, em tìm kiếm dịch vụ VPC, truy cập vào mục **Your VPCs** và chọn **Create VPC** để tiến hành tạo mạng riêng.

![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-vpc.png)
Trong đó, dải CIDR `10.30.0.0/16` xác định phạm vi địa chỉ IP của VPC. Từ dải địa chỉ này, em có thể tiếp tục phân chia thành các mạng con, hay còn gọi là Subnet, để phục vụ các thành phần khác nhau của hệ thống.

### 2.3. Cấu hình DNS cho VPC

Sau khi VPC được tạo thành công, em tiếp tục cấu hình DNS để hỗ trợ việc phân giải tên miền trong mạng.
Đầu tiên, em truy cập **Your VPCs**, chọn VPC `ridenow-prod-vpc`, sau đó vào **Actions → Edit VPC settings**.
Tại phần DNS Settings, em bật hai tính năng:
**Thứ nhất là DNS Resolution.**
Tính năng này cho phép các tài nguyên trong VPC sử dụng dịch vụ phân giải DNS do AWS cung cấp, giúp chuyển đổi tên miền thành địa chỉ IP khi cần thiết.
**Thứ hai là DNS Hostnames.**
Tính năng này hỗ trợ việc cấp tên DNS cho những tài nguyên đủ điều kiện trong VPC.
Việc bật hai tính năng DNS giúp các thành phần trong hệ thống thuận tiện hơn khi cần giao tiếp thông qua tên miền, thay vì phải sử dụng trực tiếp địa chỉ IP.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-enable-vpc-dns-settings.png)

### 2.4. Kiểm tra VPC bằng AWS CLI trên Ubuntu WSL

Ngoài việc kiểm tra trực tiếp trên AWS Console, em còn sử dụng AWS CLI trong môi trường Ubuntu WSL để xác minh các thông tin của VPC thông qua dòng lệnh.
Cách kiểm tra này giúp em xác nhận cấu hình tài nguyên trực tiếp từ AWS API, thay vì chỉ dựa vào giao diện quản lý.
**Bước đầu tiên, em lấy VPC ID theo tên đã cấu hình.**
Em sử dụng lệnh:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-find-vpc-id.png)

Trong đó, `describe-vpcs` được sử dụng để truy vấn thông tin VPC, còn `--filters` giúp tìm đúng VPC thông qua tag Name.
Sau đó, em chạy:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-display-vpc-id.png)
Lệnh này hiển thị VPC ID mà AWS đã tự động cấp cho tài nguyên.

**Tiếp theo, em kiểm tra dải địa chỉ CIDR và trạng thái của VPC.**
Em thực hiện lệnh:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-verify-vpc-cidr-and-state.png)
Kết quả này cho thấy VPC có dải địa chỉ CIDR đúng như thiết kế và đang ở trạng thái Available.

**Cuối cùng, em kiểm tra hai tính năng DNS.**
Để kiểm tra DNS Resolution, em sử dụng lệnh:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-verify-vpc-dns-resolution.png)

Tiếp theo, để kiểm tra DNS Hostnames, em sử dụng lệnh:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cli-verify-vpc-dns-hostnames.png)

Với cả hai lệnh, em cần xác nhận thuộc tính `Value` trả về là `true`.
Điều này chứng tỏ hai tính năng DNS đã được bật thành công trên VPC.

### 2.5. Kết quả và chuyển sang bước tiếp theo

Như vậy, sau khi hoàn thành quá trình triển khai và kiểm tra, em đã tạo được một VPC có tên `ridenow-prod-vpc`, với dải địa chỉ mạng là `10.30.0.0/16`, nằm tại Region Singapore.
Bên cạnh đó, em cũng đã cấu hình thành công hai tính năng DNS Resolution và DNS Hostnames, đồng thời sử dụng AWS CLI để xác minh trạng thái và các thuộc tính của VPC.
VPC này sẽ đóng vai trò là nền tảng mạng cho toàn bộ hệ thống RideNow.
Tuy nhiên, tại thời điểm này, em mới chỉ tạo mạng riêng và chưa thiết lập đường kết nối Internet cho VPC.

Vì vậy, **ở bước tiếp theo, em sẽ tiến hành tạo Internet Gateway và gắn vào VPC**, nhằm chuẩn bị cho việc cấu hình kết nối Internet của các Public Subnet trong hệ thống.

## 3. Tạo Internet Gateway – Kết nối VPC với Internet

### 3.1. Giới thiệu Internet Gateway

Sau khi hoàn thành việc tạo VPC, em tiếp tục cấu hình Internet Gateway cho hệ thống RideNow.
Internet Gateway là thành phần đóng vai trò như một cổng kết nối giữa mạng VPC và Internet.

Trong hệ thống RideNow, Internet Gateway được sử dụng để hỗ trợ kết nối Internet cho các tài nguyên trong Public Subnet, khi những tài nguyên này được cấu hình địa chỉ IP, định tuyến và quyền truy cập phù hợp.

### 3.2. Các bước tạo Internet Gateway

Đầu tiên, em đăng nhập vào AWS Management Console và chọn Region **Asia Pacific (Singapore)**, có mã là `ap-southeast-1`.

Tại giao diện tạo Internet Gateway, em nhập tên tài nguyên là `ridenow-prod-igw`.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-internet-gateway.png)

Trong đó, `ridenow` là tên hệ thống, `prod` đại diện cho môi trường production và `igw` là viết tắt của Internet Gateway.
Sau khi kiểm tra thông tin, em nhấn **Create internet gateway** để AWS tiến hành tạo tài nguyên.

### 3.3. Gắn Internet Gateway vào VPC

Sau khi Internet Gateway được tạo thành công, tài nguyên ban đầu có trạng thái **Detached**, nghĩa là chưa được gắn vào VPC nào.
Vì vậy, em tiếp tục thực hiện thao tác liên kết Internet Gateway với VPC đã tạo ở bước trước.
Đầu tiên, em chọn Internet Gateway `ridenow-prod-igw`, sau đó truy cập **Actions → Attach to VPC**.
Tại phần Available VPCs, em chọn VPC có tên `ridenow-prod-vpc` và nhấn **Attach internet gateway**.
Sau khi thao tác thành công, trạng thái Internet Gateway chuyển từ **Detached** sang **Attached**.
Điều này cho biết Internet Gateway đã được gắn vào đúng mạng VPC của hệ thống RideNow.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-attach-internet-gateway-to-vpc.png)

### 3.4. Kết quả và chuyển sang bước tiếp theo

Như vậy, sau khi hoàn thành quá trình triển khai và kiểm tra, em đã tạo được Internet Gateway có tên `ridenow-prod-igw` trong Region Singapore.
Internet Gateway này đã được liên kết với VPC `ridenow-prod-vpc`.
Bên cạnh đó, em cũng sử dụng AWS CLI để kiểm tra và xác nhận trạng thái liên kết giữa hai tài nguyên.
Tuy nhiên, **việc tạo và gắn Internet Gateway vào VPC chưa đồng nghĩa với việc các tài nguyên trong VPC có thể truy cập Internet ngay lập tức**.
Để thực hiện được điều đó, hệ thống cần có các Subnet phù hợp, Route Table định tuyến lưu lượng đến Internet Gateway và những cấu hình mạng cần thiết khác.

Vì vậy, ở bước tiếp theo, em sẽ tiến hành tạo các Public Subnet và Private Subnet để phân chia mạng, sau đó cấu hình Route Table nhằm thiết lập đường kết nối Internet cho các Public Subnet của hệ thống RideNow.

## 4. Tạo Subnet – Phân chia mạng

### 4.1. Giới thiệu Subnet

Sau khi hoàn thành việc tạo VPC và cấu hình Internet Gateway, em tiếp tục tiến hành phân chia mạng bằng cách tạo các Subnet.
Subnet là một mạng con nằm bên trong VPC, giúp chúng ta chia dải địa chỉ IP thành nhiều khu vực mạng nhỏ hơn để phục vụ các mục đích khác nhau.

Trong hệ thống RideNow, em chia VPC `ridenow-prod-vpc` thành **bốn Subnet**, bao gồm hai Public Subnet và hai Private Subnet.
Public Subnet được sử dụng cho những tài nguyên cần có đường kết nối trực tiếp với Internet, chẳng hạn như NAT Gateway và Load Balancer công khai.
Trong khi đó, Private Subnet được sử dụng cho các thành phần nội bộ như ứng dụng và cơ sở dữ liệu, nhằm hạn chế việc truy cập trực tiếp từ Internet.

Việc phân chia này giúp hệ thống có cấu trúc mạng rõ ràng hơn, đồng thời tạo điều kiện thuận lợi cho việc quản lý và bảo mật các tài nguyên.

### 4.2. Các bước tạo Subnet trên AWS Console

Đầu tiên, em đăng nhập vào AWS Management Console và chọn Region **Asia Pacific (Singapore)**, có mã là `ap-southeast-1`.

Tại phần VPC ID, em chọn VPC `ridenow-prod-vpc` đã tạo ở bước trước.
Tiếp theo, em lần lượt cấu hình bốn Subnet như sau:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-1.png)

![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-2.png)



![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-3.png)

![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-4.png)

![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-5.png)

![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-subnets-step-6.png)

| Subnet                   | Availability Zone | IPv4 CIDR       |
| ------------------------ | ----------------- | --------------- |
| `ridenow-prod-public-1`  | ap-southeast-1a   | `10.30.0.0/20`  |
| `ridenow-prod-public-2`  | ap-southeast-1b   | `10.30.16.0/20` |
| `ridenow-prod-private-1` | ap-southeast-1a   | `10.30.48.0/20` |
| `ridenow-prod-private-2` | ap-southeast-1b   | `10.30.64.0/20` |

Trong đó, hai Public Subnet được đặt ở hai Availability Zone khác nhau, tương tự với hai Private Subnet.
Availability Zone là các khu vực hạ tầng độc lập trong cùng một AWS Region.
Việc phân chia Subnet trên hai Availability Zone giúp hệ thống có nền tảng để triển khai kiến trúc có tính sẵn sàng cao. Khi một Availability Zone gặp sự cố, những thành phần được triển khai phù hợp ở Availability Zone còn lại có thể tiếp tục phục vụ hệ thống.
Sau khi nhập thông tin Subnet name, Availability Zone và IPv4 CIDR, em chọn **Add new subnet** để tiếp tục thêm các Subnet còn lại.
Cuối cùng, em nhấn **Create subnet** để AWS tiến hành tạo cả bốn Subnet.

### 4.3. Cấu hình Kubernetes Tags cho các Subnet

Sau khi tạo các Subnet, em tiếp tục cấu hình Kubernetes Tags.
Mục đích của việc thêm các tag này là giúp AWS Load Balancer Controller nhận diện các Subnet phù hợp khi triển khai Load Balancer trên EKS ở những bước tiếp theo.
Đầu tiên, em truy cập **VPC → Subnets**, chọn từng Subnet và mở tab **Tags** để kiểm tra hoặc bổ sung tag.
Đối với hai Public Subnet, em sử dụng tag:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-subnet-kubernetes-tags.png)

Tag này đánh dấu các Subnet phù hợp để triển khai Load Balancer công khai, hay còn gọi là Internet-facing Load Balancer.

Đối với hai Private Subnet, em sử dụng tag:

![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-subnet-kubernetes-tags.png)
Tag này đánh dấu các Subnet phù hợp để triển khai Load Balancer nội bộ, hay còn gọi là Internal Load Balancer.

### 4.4. Cấu hình tự động cấp phát Public IP

Tiếp theo, em cấu hình tính năng tự động cấp phát địa chỉ Public IPv4 cho các Subnet.
Đối với hai Public Subnet, em bật tính năng **Enable auto-assign public IPv4 address**.
Để thực hiện, em truy cập **VPC → Subnets**, chọn `ridenow-prod-public-1`, sau đó vào **Actions → Edit subnet settings**.
Tại đây, em đánh dấu tùy chọn **Enable auto-assign public IPv4 address** và nhấn **Save**.
Sau đó, em thực hiện tương tự đối với `ridenow-prod-public-2`.
Việc bật tính năng này cho phép các EC2 instance được khởi tạo trong Public Subnet có thể tự động nhận địa chỉ Public IPv4 theo cấu hình mặc định, nếu không có thiết lập khác ghi đè.

Ngược lại, đối với hai Private Subnet là `ridenow-prod-private-1` và `ridenow-prod-private-2`, em giữ tắt tính năng tự động cấp phát Public IPv4.
Mục đích là hạn chế việc các tài nguyên nội bộ được cấp địa chỉ IP công khai ngoài ý muốn.
Tuy nhiên, việc bật Public IP không có nghĩa là các tài nguyên đã có thể kết nối Internet ngay lập tức. Để thực hiện điều đó, hệ thống vẫn cần cấu hình Route Table và các điều kiện mạng phù hợp.

### 4.5. Kết quả và chuyển sang bước tiếp theo

Như vậy, sau khi hoàn thành quá trình triển khai và kiểm tra, em đã tạo được bốn Subnet nằm trong VPC `ridenow-prod-vpc`, bao gồm hai Public Subnet và hai Private Subnet.
Các Subnet được phân chia trên hai Availability Zone là `ap-southeast-1a` và `ap-southeast-1b`.
Bên cạnh đó, em cũng đã cấu hình Kubernetes Tags để hỗ trợ việc triển khai Load Balancer trên EKS sau này, đồng thời thiết lập tính năng tự động cấp Public IP cho hai Public Subnet.
Sau khi hoàn thành, em sử dụng AWS Console và AWS CLI để kiểm tra lại tên, CIDR, Availability Zone, trạng thái, cấu hình Public IP và các tag của từng Subnet.
Việc phân chia mạng này giúp hệ thống RideNow có cấu trúc rõ ràng hơn giữa các thành phần công khai và nội bộ, đồng thời tạo nền tảng cho việc triển khai các dịch vụ trên nhiều Availability Zone.
Tuy nhiên, ở thời điểm này, các Subnet mới chỉ được tạo và cấu hình các thuộc tính cơ bản, chưa có đầy đủ đường định tuyến Internet theo thiết kế.
Vì vậy, **ở bước tiếp theo, em sẽ tiến hành tạo Public Route Table**, cấu hình tuyến đường mặc định `0.0.0.0/0` trỏ đến Internet Gateway `ridenow-prod-igw`, sau đó liên kết Route Table với hai Public Subnet để hoàn thiện phần định tuyến Internet cho khu vực mạng công khai của hệ thống.

## 5. Tạo Route Table – Điều hướng lưu lượng mạng

### 5.1. Giới thiệu Route Table

Sau khi hoàn thành việc tạo bốn Subnet, em tiếp tục tiến hành cấu hình Route Table cho hệ thống RideNow.
Route Table là bảng định tuyến trong AWS VPC, có nhiệm vụ xác định đường đi của các gói tin khi được gửi từ các Subnet đến những địa chỉ mạng khác.

Trong hệ thống RideNow, em thiết kế tổng cộng ba Route Table, bao gồm một Public Route Table và hai Private Route Table.
Trong đó, Public Route Table được sử dụng để định tuyến lưu lượng Internet của hai Public Subnet thông qua Internet Gateway.
Còn hai Private Route Table sẽ được cấu hình riêng cho từng Private Subnet, sử dụng NAT Gateway để các tài nguyên nội bộ có thể chủ động truy cập Internet mà không cần có Public IP.

Ở bước này, em tiến hành tạo và cấu hình Public Route Table trước. Hai Private Route Table sẽ được hoàn thiện sau khi triển khai NAT Gateway.

### 5.2. Các bước tạo Public Route Table trên AWS Console

Đầu tiên, em đăng nhập vào AWS Management Console và chọn Region **Asia Pacific (Singapore)**, có mã là `ap-southeast-1`.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-public-route-table.png)
Tại giao diện tạo Route Table, em thực hiện các cấu hình như sau:
- **Name:** `ridenow-prod-public-rt`.
- **VPC:** Chọn `ridenow-prod-vpc`.

Sau khi tạo thành công, em mở tab **Routes** để kiểm tra các tuyến đường hiện có.
Lúc này, Route Table đã có một tuyến mặc định của VPC:
```
Destination: 10.30.0.0/16
Target: local
```
Trong đó, `10.30.0.0/16` là dải địa chỉ mạng của VPC RideNow, còn `local` cho phép các tài nguyên trong VPC định tuyến lưu lượng đến nhau.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-route-table-local-route.png)
Tuyến đường này được AWS tự động tạo khi Route Table được khởi tạo.

### 5.3. Cấu hình tuyến đường kết nối Internet

Sau khi tạo Public Route Table, em tiếp tục cấu hình tuyến đường để các Public Subnet có thể sử dụng Internet Gateway.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-route-table-internet-gateway-route.png)
Trong cấu hình này, `0.0.0.0/0` đại diện cho tuyến mặc định của toàn bộ địa chỉ IPv4.
Điều này có nghĩa là khi một gói tin được gửi đến địa chỉ không khớp với tuyến đường cụ thể hơn, Route Table sẽ chuyển lưu lượng đó đến Internet Gateway.
Còn Internet Gateway `ridenow-prod-igw` là thành phần đã được tạo và gắn vào VPC ở bước trước, đóng vai trò kết nối giữa VPC và Internet.
Sau khi lưu, Public Route Table có hai tuyến đường chính:

|Destination|Target|
|---|---|
|`10.30.0.0/16`|`local`|
|`0.0.0.0/0`|`ridenow-prod-igw`|
Trong đó, tuyến `local` phục vụ việc giao tiếp trong VPC, còn tuyến `0.0.0.0/0` định tuyến lưu lượng Internet thông qua Internet Gateway.

### 5.4. Liên kết Public Route Table với hai Public Subnet

Sau khi cấu hình tuyến Internet, em tiếp tục liên kết Public Route Table với hai Public Subnet đã tạo trước đó.

Tại phần Explicit subnet associations, em liên kết cả hai Public Subnet với cùng một Public Route Table.
- `ridenow-prod-public-1`.
- `ridenow-prod-public-2`.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-public-route-table-subnet-associations.png)

Khi các tài nguyên trong hai Public Subnet cần gửi lưu lượng ra Internet, Route Table sẽ cung cấp tuyến đường mặc định đến Internet Gateway.
Tuy nhiên, để kết nối Internet thực tế, tài nguyên vẫn cần có địa chỉ IP công khai phù hợp và các cấu hình Security Group, Network ACL cho phép lưu lượng.

### 5.5. Kết quả và chuyển sang bước tiếp theo

Như vậy, sau khi hoàn thành quá trình triển khai và kiểm tra, em đã tạo được Public Route Table có tên `ridenow-prod-public-rt` trong VPC `ridenow-prod-vpc`.
Trong Route Table này, em đã cấu hình tuyến mặc định `0.0.0.0/0` trỏ đến Internet Gateway `ridenow-prod-igw`.
Bên cạnh đó, em cũng đã liên kết Public Route Table với hai Public Subnet là `ridenow-prod-public-1` và `ridenow-prod-public-2`.
Cuối cùng, em sử dụng AWS Console và AWS CLI trên Ubuntu WSL để xác minh các tuyến đường và mối liên kết giữa Route Table với Subnet.
Việc cấu hình Public Route Table giúp thiết lập đường định tuyến Internet cho hai Public Subnet, đồng thời phân biệt rõ khu vực mạng công khai và khu vực mạng nội bộ của hệ thống RideNow.
Tuy nhiên, tại thời điểm này, **em mới hoàn thành Public Route Table, còn hai Private Route Table chưa được cấu hình đầy đủ**.
Để các tài nguyên trong Private Subnet có thể chủ động truy cập Internet mà không cần sử dụng Public IP, hệ thống cần có NAT Gateway.
Vì vậy, ở bước tiếp theo, em sẽ tiến hành **cấp phát hai Elastic IP và tạo hai NAT Gateway**, mỗi NAT Gateway được đặt trong một Public Subnet thuộc Availability Zone riêng.

Sau khi hoàn thành NAT Gateway, em sẽ tiếp tục tạo hai Private Route Table và cấu hình đường định tuyến Internet thông qua NAT Gateway tương ứng.

## 6. Tạo NAT Gateway – Cho Private Subnet truy cập Internet

### 6.1. Giới thiệu NAT Gateway

Sau khi hoàn thành việc cấu hình Public Route Table, em tiếp tục triển khai NAT Gateway cho hệ thống RideNow.

NAT Gateway là một dịch vụ mạng của AWS, giúp các tài nguyên nằm trong Private Subnet có thể chủ động kết nối ra Internet mà không cần được cấp địa chỉ Public IP trực tiếp.
Trong hệ thống RideNow, các thành phần nội bộ như ứng dụng chạy trên EKS có thể cần truy cập Internet để tải thư viện, lấy container image hoặc kết nối đến các dịch vụ bên ngoài.

Tuy nhiên, để đảm bảo tính bảo mật, các thành phần này được thiết kế nằm trong Private Subnet và không trực tiếp sử dụng Public IP.
Vì vậy, em sử dụng NAT Gateway để cung cấp đường kết nối Internet ra ngoài cho những tài nguyên này.
Cụ thể, em tiến hành cấp phát hai Elastic IP, tạo hai NAT Gateway và cấu hình hai Private Route Table.
Mỗi NAT Gateway được đặt trong một Public Subnet thuộc một Availability Zone khác nhau, nhằm giảm sự phụ thuộc vào một NAT Gateway duy nhất.

### 6.2. Cấp phát hai Elastic IP

Đầu tiên, em tiến hành cấp phát hai Elastic IP để sử dụng cho hai NAT Gateway.
Elastic IP là một địa chỉ IPv4 công khai tĩnh được AWS cấp phát, cho phép NAT Gateway sử dụng địa chỉ IP công khai ổn định khi kết nối ra Internet.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-allocate-nat-elastic-ip-1.png)

Sau khi nhập đầy đủ thông tin, em nhấn **Allocate** để AWS cấp phát Elastic IP thứ nhất.

Tiếp theo, em thực hiện tương tự để cấp phát Elastic IP thứ hai, nhưng thay đổi Name tag thành `ridenow-prod-nat-eip-2`.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-allocate-nat-elastic-ip-2.png)

Sau khi hoàn thành, em quay lại danh sách Elastic IPs để kiểm tra.
Tại đây, em xác nhận đã có hai Elastic IP được cấp phát, mỗi địa chỉ có một Allocation ID riêng do AWS tạo ra.
Hai Elastic IP này sẽ lần lượt được gắn vào NAT Gateway thứ nhất và NAT Gateway thứ hai.

### 6.3. Tạo NAT Gateway thứ nhất

Sau khi cấp phát hai Elastic IP, em tiếp tục tạo NAT Gateway thứ nhất.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-nat-gateway-1.png)

Trong đó, NAT Gateway thứ nhất được đặt tại Public Subnet 1, thuộc Availability Zone `ap-southeast-1a`.
Em chọn Connectivity type là Public vì NAT Gateway này cần có khả năng kết nối ra Internet thông qua Internet Gateway của VPC.
Đồng thời, em sử dụng Elastic IP thứ nhất để NAT Gateway có địa chỉ IPv4 công khai.

### 6.4. Tạo NAT Gateway thứ hai

Tiếp theo, em tạo NAT Gateway thứ hai để phục vụ cho Private Subnet nằm ở Availability Zone còn lại.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-nat-gateway-2.png)

Khác với NAT Gateway thứ nhất, NAT Gateway thứ hai được đặt trong Public Subnet 2, thuộc Availability Zone `ap-southeast-1b`.
Mỗi NAT Gateway sử dụng một Elastic IP riêng và được triển khai trên một Availability Zone khác nhau.

Cách thiết kế này giúp hệ thống hạn chế sự phụ thuộc vào một NAT Gateway duy nhất, đồng thời cho phép mỗi Private Subnet sử dụng NAT Gateway trong cùng Availability Zone.

### 6.5. Tạo Private Route Table thứ nhất

Sau khi hai NAT Gateway đã sẵn sàng, em tiếp tục cấu hình Route Table để các Private Subnet có thể sử dụng NAT Gateway khi cần kết nối ra Internet.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-private-route-table-1.png)


Em cấu hình tuyến đường:
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-route-table-1-nat-default-route.png)
Trong đó, `0.0.0.0/0` là tuyến mặc định dành cho những địa chỉ IPv4 không khớp với tuyến cụ thể hơn.
Còn NAT Gateway `ridenow-prod-nat-1` là thành phần tiếp nhận lưu lượng Internet được gửi từ Private Subnet 1.

Cuối cùng, em liên kết Route Table này với `ridenow-prod-private-1`.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-route-table-1-subnet-association.png)

Như vậy, Private Subnet 1 đã có đường định tuyến mặc định thông qua NAT Gateway 1.

### 6.6. Tạo Private Route Table thứ hai

Tiếp theo, em tiến hành tạo Private Route Table thứ hai cho Private Subnet 2.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-create-private-route-table-2.png)

Em tiếp tục vào liên kết với `ridenow-prod-private-2`.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-private-route-table-2-subnet-association.png)

Như vậy, hai Private Subnet đã được cấu hình sử dụng hai NAT Gateway riêng biệt.
Cụ thể, Private Subnet 1 sử dụng NAT Gateway 1 ở Availability Zone 1a, còn Private Subnet 2 sử dụng NAT Gateway 2 ở Availability Zone 1b.

### 6.7. Kết quả và chuyển sang bước tiếp theo

Như vậy, sau khi hoàn thành quá trình triển khai và kiểm tra, em đã cấp phát hai Elastic IP và tạo được hai NAT Gateway cho hệ thống RideNow.
Trong đó, NAT Gateway thứ nhất được đặt trong Public Subnet 1 thuộc Availability Zone `ap-southeast-1a`, còn NAT Gateway thứ hai được đặt trong Public Subnet 2 thuộc Availability Zone `ap-southeast-1b`.
Mỗi NAT Gateway sử dụng một Elastic IP riêng để phục vụ kết nối Internet.
Bên cạnh đó, em cũng đã tạo hai Private Route Table và cấu hình tuyến mặc định `0.0.0.0/0` đến NAT Gateway tương ứng.
Nhờ cấu hình này, các tài nguyên trong Private Subnet có đường định tuyến để chủ động kết nối ra Internet mà không cần được cấp Public IP trực tiếp, khi các điều kiện mạng và bảo mật khác được đáp ứng.
Việc triển khai NAT Gateway trên hai Availability Zone cũng giúp giảm sự phụ thuộc vào một NAT Gateway duy nhất và tạo nền tảng cho kiến trúc có tính sẵn sàng cao.
Cuối cùng, em đã sử dụng AWS Console và AWS CLI trên Ubuntu WSL để kiểm tra trạng thái NAT Gateway và cấu hình Route Table.
**Sau khi hoàn thành phần NAT Gateway, ở bước tiếp theo, em sẽ tiến hành cấu hình Security Group** nhằm kiểm soát lưu lượng truy cập giữa các thành phần ứng dụng và cơ sở dữ liệu trong hệ thống RideNow.

## 7. Tạo Cluster Security Group – Bảo vệ ứng dụng

### 7.1. Giới thiệu Security Group

Sau khi hoàn thành việc cấu hình NAT Gateway và các Route Table, em tiếp tục tiến hành thiết lập Security Group để kiểm soát lưu lượng mạng cho hệ thống RideNow.

Security Group có thể hiểu là một lớp tường lửa ảo của AWS, được sử dụng để kiểm soát các kết nối đi vào và đi ra những tài nguyên được gắn Security Group.
Trong đó, **Inbound Rules** quy định những lưu lượng nào được phép đi vào, còn **Outbound Rules** quy định những lưu lượng nào được phép đi ra.

Đối với hệ thống RideNow, em tạo một Security Group có tên `ridenow-prod-cluster-sg`, được thiết kế để sử dụng cho EKS và các thành phần ứng dụng trong những bước triển khai tiếp theo.
Mục tiêu của bước này là cho phép các tài nguyên sử dụng cùng Security Group giao tiếp nội bộ với nhau, đồng thời không mở quyền truy cập trực tiếp từ toàn bộ Internet.

### 7.2. Các bước tạo Cluster Security Group

Inbound Rules là tập hợp các quy tắc xác định những kết nối nào được phép đi vào tài nguyên sử dụng Security Group.
Ở bước tạo ban đầu, em **không thêm bất kỳ Inbound Rule nào**.
Điều này có nghĩa là Security Group chưa cho phép các kết nối mới đi vào từ các nguồn bên ngoài theo bất kỳ rule nào.
Mục đích là hạn chế những kết nối không cần thiết và chỉ bổ sung các quyền truy cập phù hợp với yêu cầu của hệ thống.
Sau khi tạo Security Group thành công, em sẽ bổ sung một rule riêng để cho phép các tài nguyên thuộc cùng Security Group giao tiếp với nhau.

Outbound Rules quy định những kết nối mà tài nguyên sử dụng Security Group được phép gửi ra bên ngoài.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cluster-sg-outbound-rules.png)

Với cấu hình này, các tài nguyên sử dụng Security Group được phép khởi tạo lưu lượng IPv4 đi ra ngoài trên tất cả các giao thức và cổng.
Điều này hỗ trợ ứng dụng kết nối đến những dịch vụ cần thiết, chẳng hạn như cơ sở dữ liệu RDS hoặc các dịch vụ bên ngoài thông qua NAT Gateway.
Tuy nhiên, khả năng kết nối thực tế còn phụ thuộc vào Route Table, Network ACL và các cấu hình mạng liên quan.
Sau khi hoàn tất, em giữ nguyên cấu hình Outbound Rule để sử dụng cho Cluster Security Group.

Sau khi cấu hình Inbound và Outbound Rules, em tiếp tục thêm tag để phục vụ việc quản lý và truy vấn tài nguyên.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cluster-sg-name-tag.png)

Tag Name giúp em nhận diện Security Group trên AWS Console, đồng thời có thể sử dụng để truy vấn tài nguyên bằng AWS CLI.

### 7.3. Thêm Inbound Rule tự tham chiếu

Sau khi tạo Security Group thành công, em tiếp tục bổ sung một Inbound Rule tự tham chiếu, hay còn gọi là **Self-referencing Rule**.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-cluster-sg-self-referencing-inbound-rule.png)

Điểm quan trọng là ở phần Source, em không chọn `0.0.0.0/0`, mà chọn chính Security Group `ridenow-prod-cluster-sg`.
Điều này được gọi là rule tự tham chiếu vì Security Group sử dụng chính nó làm nguồn được phép truy cập.

### 7.4. Giải thích ý nghĩa của Self-referencing Rule

Self-referencing Rule cho phép các tài nguyên được gắn cùng một Security Group gửi lưu lượng đến nhau theo những giao thức và cổng được quy định trong rule.

Ví dụ, giả sử hệ thống có hai tài nguyên là Application A và Application B, cả hai đều sử dụng Security Group `ridenow-prod-cluster-sg`.
Khi Application A cần gửi yêu cầu đến Application B, rule tự tham chiếu sẽ cho phép lưu lượng này đi vào Application B nếu các điều kiện mạng khác cũng được đáp ứng.

Do em chọn Type là All traffic, nên rule này cho phép tất cả các giao thức và cổng giữa những tài nguyên thuộc cùng Security Group.
Tuy nhiên, điều đó không có nghĩa là mọi tài nguyên trong VPC đều được tự động giao tiếp với nhau. Rule chỉ áp dụng cho các tài nguyên thực sự sử dụng Security Group tương ứng.
Đồng thời, cấu hình này không cho phép toàn bộ Internet truy cập trực tiếp vào ứng dụng.
Như vậy, Self-referencing Rule hỗ trợ giao tiếp nội bộ giữa các tài nguyên ứng dụng và giúp giới hạn phạm vi nguồn truy cập.

### 7.5. Kết quả và chuyển sang bước tiếp theo

Như vậy, sau khi hoàn thành quá trình triển khai và kiểm tra, em đã tạo được Cluster Security Group có tên `ridenow-prod-cluster-sg` trong VPC `ridenow-prod-vpc`.
Đối với Inbound Rules, em đã cấu hình một rule tự tham chiếu, cho phép các tài nguyên sử dụng cùng Security Group giao tiếp với nhau.
Đối với Outbound Rules, em giữ cấu hình cho phép tất cả lưu lượng IPv4 đi ra ngoài, nhằm đáp ứng nhu cầu kết nối của các thành phần ứng dụng.
Bên cạnh đó, em cũng đã thêm tag Name để thuận tiện cho việc quản lý tài nguyên và truy vấn bằng AWS CLI.
Sau khi cấu hình, em sử dụng AWS Console và AWS CLI trên Ubuntu WSL để kiểm tra lại Security Group ID, Inbound Rules và Outbound Rules.
Việc thiết lập Security Group giúp hệ thống RideNow có thêm một lớp kiểm soát lưu lượng mạng, tạo nền tảng cho việc triển khai các dịch vụ ứng dụng trên EKS ở những bước tiếp theo.
Tuy nhiên, Security Group này mới chỉ được tạo và cấu hình. Các quy tắc sẽ phát huy tác dụng đối với những tài nguyên được gắn Security Group khi hệ thống được triển khai.
**Sau khi hoàn thành phần Cluster Security Group, ở bước tiếp theo, em sẽ tiến hành tạo Database Security Group** nhằm kiểm soát các kết nối đến cơ sở dữ liệu PostgreSQL, trong đó chỉ cho phép truy cập cổng TCP 5432 từ Cluster Security Group đã tạo.

## 8. Tạo Database Security Group – Bảo vệ cơ sở dữ liệu

### 8.1. Giới thiệu Database Security Group

Sau khi hoàn thành việc tạo Cluster Security Group cho các thành phần ứng dụng, em tiếp tục cấu hình một Security Group riêng để bảo vệ cơ sở dữ liệu PostgreSQL của hệ thống RideNow.
Security Group là một lớp tường lửa ảo trên AWS, có nhiệm vụ kiểm soát lưu lượng mạng đi vào và đi ra các tài nguyên được gắn Security Group.

Trong phần này, em tạo một Database Security Group có tên là `ridenow-prod-db-sg`.
Mục đích của Security Group này là giới hạn quyền truy cập vào cơ sở dữ liệu PostgreSQL, chỉ cho phép các tài nguyên thuộc Cluster Security Group đã được tạo ở bước trước kết nối thông qua cổng TCP 5432.
Đây là cổng mặc định được PostgreSQL sử dụng để tiếp nhận các kết nối từ ứng dụng.
Thay vì mở cổng cơ sở dữ liệu cho toàn bộ Internet, em sử dụng Security Group của ứng dụng làm nguồn truy cập nhằm tăng cường khả năng bảo vệ dữ liệu.

### 8.2. Cấu hình Inbound Rule cho PostgreSQL

Sau khi nhập các thông tin cơ bản, em tiếp tục cấu hình Inbound Rule.
Inbound Rule là quy tắc xác định những kết nối nào được phép đi vào tài nguyên sử dụng Security Group.
Đối với Database Security Group, em chỉ cho phép kết nối đến cổng PostgreSQL từ Cluster Security Group đã tạo trước đó.
![Ảnh minh họa cấu hình mạng RideNow](/fcj-workshop-ridenow/images/5-Workshop/5.4/5.4-database-sg-postgresql-inbound-rule.png)

Trong đó, khi chọn Type là PostgreSQL, AWS sẽ tự động thiết lập giao thức TCP và cổng 5432.
Điểm quan trọng nhất nằm ở phần Source.
Thay vì chọn `Anywhere-IPv4` với dải địa chỉ `0.0.0.0/0`, em chọn chính Cluster Security Group có tên `ridenow-prod-cluster-sg`.
Điều này có nghĩa là rule chỉ cho phép các kết nối đến cổng TCP 5432 có nguồn là những tài nguyên được gắn Cluster Security Group.
Việc sử dụng Security Group làm Source giúp giới hạn phạm vi truy cập và tránh việc mở cổng PostgreSQL cho mọi địa chỉ IP trên Internet.
Khác với Self-referencing Rule ở bước trước, trong trường hợp này em có thể cấu hình Inbound Rule ngay khi tạo Database Security Group, vì Cluster Security Group đã tồn tại và có Security Group ID để lựa chọn.

### 8.3. Cấu hình Outbound Rules

Sau khi cấu hình Inbound Rule, em tiếp tục kiểm tra Outbound Rules.
Outbound Rules quy định những kết nối mà tài nguyên sử dụng Security Group được phép khởi tạo ra bên ngoài.
Theo thiết kế của hệ thống RideNow, em giữ cấu hình Outbound mặc định với các thông tin:
- **Type:** All traffic.
- **Protocol:** All.
- **Port range:** All.
- **Destination:** Anywhere-IPv4, tương ứng `0.0.0.0/0`.
- **Description:** `Allow all egress`.
Với cấu hình này, các tài nguyên sử dụng Database Security Group được phép khởi tạo lưu lượng IPv4 đi ra ngoài theo quy tắc của Security Group.
Tuy nhiên, việc có Outbound Rule cho phép tất cả lưu lượng không đồng nghĩa với việc cơ sở dữ liệu sẽ được truy cập trực tiếp từ Internet.
Khả năng truy cập thực tế còn phụ thuộc vào các yếu tố khác như Subnet, Route Table, địa chỉ IP và cấu hình của dịch vụ RDS.
Bên cạnh đó, Security Group trên AWS có tính chất **stateful**, nghĩa là lưu lượng phản hồi cho một kết nối đã được cho phép sẽ tự động được chấp nhận, không cần tạo thêm một rule riêng cho chiều phản hồi.

### 8.4. Kết quả và kết luận

Như vậy, sau khi hoàn thành quá trình triển khai và kiểm tra, em đã tạo được Database Security Group có tên `ridenow-prod-db-sg` trong VPC `ridenow-prod-vpc`.
Đối với Inbound Rules, em chỉ cho phép kết nối đến cổng TCP 5432 từ Cluster Security Group `ridenow-prod-cluster-sg`.
Điều này giúp giới hạn quyền truy cập vào cơ sở dữ liệu PostgreSQL và tránh việc mở cổng kết nối cơ sở dữ liệu cho toàn bộ Internet.
Đối với Outbound Rules, em giữ cấu hình cho phép tất cả lưu lượng IPv4 đi ra theo yêu cầu của tài liệu.
Bên cạnh đó, em cũng thêm tag Name để thuận tiện cho việc quản lý tài nguyên và truy vấn thông qua AWS CLI.
Sau khi cấu hình, em sử dụng AWS Console và AWS CLI trên Ubuntu WSL để kiểm tra Security Group ID, Inbound Rules và Outbound Rules.
Việc tạo Database Security Group giúp hệ thống RideNow có thêm một lớp bảo vệ mạng cho cơ sở dữ liệu PostgreSQL.
Tuy nhiên, tại thời điểm này, em mới chỉ tạo và cấu hình Security Group, chưa triển khai cơ sở dữ liệu RDS PostgreSQL.
Trong những bước tiếp theo, khi tạo các DB instance trên AWS RDS, em sẽ gắn Database Security Group này vào các instance tương ứng và sử dụng các Private Subnet theo thiết kế.
**Như vậy, em đã hoàn thành phần cấu hình hai Security Group trong mục 5.4 Networking, bao gồm Cluster Security Group dành cho ứng dụng và Database Security Group dành cho cơ sở dữ liệu.**
Hai Security Group này sẽ tạo nền tảng cho việc kiểm soát kết nối giữa các thành phần ứng dụng và cơ sở dữ liệu khi hệ thống RideNow được triển khai trên AWS.

## 9. Kết luận

Như vậy, trong phần 5.4, em đã xây dựng được hạ tầng mạng cơ bản cho hệ thống RideNow trên AWS.
Hệ thống sử dụng một VPC làm mạng riêng, được chia thành hai Public Subnet và hai Private Subnet trên hai Availability Zone.
Các Public Subnet có đường kết nối Internet thông qua Internet Gateway, trong khi Private Subnet có thể truy cập Internet ra ngoài thông qua NAT Gateway.
Bên cạnh đó, em sử dụng Route Table để điều hướng lưu lượng mạng và Security Group để kiểm soát các kết nối đến ứng dụng cũng như cơ sở dữ liệu.
Việc thiết kế mạng theo cách này giúp phân tách những thành phần công khai và nội bộ, đồng thời tạo nền tảng để xây dựng hệ thống có tính bảo mật và khả năng sẵn sàng cao hơn.
Sau khi hoàn thành phần Networking, em có thể tiếp tục triển khai các thành phần khác của RideNow như cơ sở dữ liệu, cache và Kubernetes trên EKS.
Em xin kết thúc phần trình bày về Networking tại đây.
