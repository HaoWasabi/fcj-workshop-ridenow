---
title : "Giới thiệu"
date : 2024-01-01 
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

# 5.1. Giới thiệu – Xây dựng hạ tầng Production cho hệ thống RideNow trên AWS
## 5.1.1. Tổng quan về hệ thống và mục tiêu triển khai

RideNow là hệ thống được triển khai theo kiến trúc Microservices, trong đó các chức năng ứng dụng được tổ chức thành những service riêng biệt.
Để các service này có thể hoạt động trong môi trường thực tế, ngoài mã nguồn ứng dụng, chúng ta cần xây dựng một hệ thống hạ tầng hỗ trợ đầy đủ các chức năng như mạng, bảo mật, lưu trữ dữ liệu, giao tiếp giữa các service và triển khai container.
Trong chương này, em tập trung vào môi trường **Production**, được đặt tên là `prod`, và lựa chọn AWS Region **Asia Pacific (Singapore)**, tương ứng với mã `ap-southeast-1`.
Production là môi trường được sử dụng để vận hành hệ thống trong điều kiện thực tế, vì vậy các thành phần hạ tầng cần được tổ chức rõ ràng, có cơ chế kiểm soát truy cập, bảo vệ dữ liệu và hỗ trợ quá trình vận hành.
Mục tiêu của chương là xây dựng các lớp hạ tầng cần thiết cho RideNow bằng AWS Management Console, đồng thời kiểm tra từng lớp sau khi cấu hình để đảm bảo các tài nguyên đáp ứng yêu cầu của hệ thống.
**[Chèn hình 1: Tổng quan AWS Management Console hoặc sơ đồ kiến trúc hạ tầng RideNow trên AWS]**

## 5.1.2. Các lớp hạ tầng chính của hệ thống RideNow
Để triển khai một hệ thống Microservices trên AWS, chúng ta cần kết hợp nhiều dịch vụ khác nhau.
Trong chương này, em chia hạ tầng thành tám lớp chính. Mỗi lớp đảm nhận một vai trò riêng, nhưng có sự liên kết với các lớp còn lại để hình thành một hệ thống hoàn chỉnh.

**Đầu tiên là lớp Identity – Quản lý danh tính và phân quyền.**
Ở lớp này, em sử dụng AWS IAM, bao gồm IAM Roles và IAM Policies.
IAM cho phép xác định những tài nguyên hoặc service nào được phép thực hiện các hành động cụ thể trên AWS.
Ví dụ, một service chỉ cần đọc thông tin từ SQS thì sẽ được cấp các quyền cần thiết liên quan đến SQS, thay vì sử dụng quyền quản trị toàn bộ tài khoản.
Điều này giúp áp dụng nguyên tắc Least Privilege, nghĩa là chỉ cấp quyền tối thiểu cần thiết cho từng thành phần.
**Lớp thứ hai là Network – Hạ tầng mạng.**
Ở lớp này, em sử dụng Amazon VPC, Subnets, NAT Gateways và Security Groups.
VPC cung cấp một môi trường mạng riêng để tổ chức các tài nguyên AWS.
Subnets được sử dụng để phân chia mạng thành những khu vực khác nhau, chẳng hạn như Public Subnet và Private Subnet.
NAT Gateway cho phép tài nguyên trong Private Subnet chủ động truy cập các dịch vụ bên ngoài khi cần thiết mà không phải trực tiếp nhận kết nối từ Internet thông qua NAT Gateway.
Trong khi đó, Security Groups đóng vai trò như các bộ lọc lưu lượng mạng, giúp kiểm soát những kết nối được phép đi vào và đi ra khỏi tài nguyên.
Việc kết hợp các thành phần này giúp phân tách khu vực công khai và khu vực nội bộ, hạn chế những truy cập không cần thiết.
**[Chèn hình 2: Sơ đồ VPC với Public Subnet, Private Subnet, NAT Gateway và Security Groups]**

**Lớp thứ ba là Images – Quản lý Container Image.**
Ở lớp này, em sử dụng Amazon Elastic Container Registry, hay Amazon ECR.
Amazon ECR là dịch vụ lưu trữ và quản lý Container Image của AWS.
Trong hệ thống RideNow, mỗi Microservice được đóng gói thành một Docker Image và lưu trữ trong một ECR Repository riêng.
Ví dụ, `ridenow/user-service` được sử dụng để lưu trữ image của User Service, còn `ridenow/driver-service` được sử dụng cho Driver Service.
Ngoài việc lưu trữ image, Amazon ECR còn hỗ trợ quản lý phiên bản, kiểm soát truy cập, mã hóa và quét lỗ hổng bảo mật.
Điều này giúp các Docker Image được quản lý tập trung, phục vụ quá trình triển khai lên Amazon EKS.
**Lớp thứ tư là Data – Lưu trữ và bảo vệ dữ liệu.**
Đối với lớp dữ liệu, em sử dụng Amazon RDS PostgreSQL, AWS Secrets Manager và AWS KMS.
Amazon RDS PostgreSQL cung cấp dịch vụ cơ sở dữ liệu quan hệ được AWS quản lý.
AWS Secrets Manager được sử dụng để lưu trữ và quản lý những thông tin nhạy cảm, chẳng hạn như thông tin xác thực kết nối cơ sở dữ liệu.
Trong khi đó, AWS KMS hỗ trợ quản lý khóa mã hóa nhằm bảo vệ dữ liệu.
Việc kết hợp các dịch vụ này giúp tách riêng cơ sở dữ liệu khỏi các service ứng dụng, đồng thời tăng cường khả năng bảo vệ thông tin quan trọng của hệ thống.

**Lớp thứ năm là Messaging and Cache – Truyền thông điệp và bộ nhớ đệm.**
Ở lớp này, em sử dụng Amazon SNS, Amazon SQS và Amazon ElastiCache for Valkey.
SNS hỗ trợ mô hình Publish/Subscribe, cho phép một sự kiện được phân phối đến nhiều thành phần đăng ký nhận thông tin.
SQS cung cấp hàng đợi thông điệp, giúp các service giao tiếp bất đồng bộ và giảm sự phụ thuộc trực tiếp vào nhau.
Trong khi đó, ElastiCache for Valkey cung cấp bộ nhớ đệm có tốc độ truy xuất cao, hỗ trợ lưu trữ những dữ liệu cần được truy cập thường xuyên.
Những dịch vụ này đặc biệt phù hợp với kiến trúc Microservices, bởi vì các service có thể trao đổi dữ liệu hoặc xử lý công việc thông qua thông điệp thay vì luôn phải gọi trực tiếp đến nhau.”
**[Chèn hình 3: Sơ đồ thể hiện các lớp ECR, RDS, SNS, SQS và ElastiCache của RideNow]**

**Lớp thứ sáu là Compute – Nền tảng thực thi ứng dụng.**
Ở lớp này, em sử dụng Amazon Elastic Kubernetes Service, hay Amazon EKS, kết hợp với AWS Fargate.
Amazon EKS là dịch vụ Kubernetes được AWS quản lý, cho phép triển khai và điều phối các ứng dụng dưới dạng container.
AWS Fargate cung cấp khả năng chạy container mà không cần trực tiếp quản lý các máy chủ EC2 nền bên dưới.
Nhờ đó, nhóm phát triển có thể tập trung nhiều hơn vào việc cấu hình và vận hành các workload ứng dụng thay vì tự quản lý toàn bộ hạ tầng máy chủ.

**Lớp thứ bảy là Ingress – Quản lý lưu lượng truy cập vào hệ thống.**
Ở lớp này, em sử dụng Application Load Balancer, kết hợp với AWS Load Balancer Controller.
Application Load Balancer có nhiệm vụ tiếp nhận và phân phối các HTTP hoặc HTTPS request đến những thành phần ứng dụng phù hợp.
AWS Load Balancer Controller hỗ trợ quản lý các tài nguyên Load Balancer của AWS dựa trên cấu hình Kubernetes.
Nhờ đó, lưu lượng truy cập từ bên ngoài có thể được định tuyến đến những service tương ứng trong hệ thống RideNow.

**Cuối cùng là lớp Frontend – Phân phối giao diện người dùng.**
Ở lớp này, em sử dụng Amazon S3 và Amazon CloudFront.
Amazon S3 được sử dụng để lưu trữ các tập tin tĩnh của frontend, chẳng hạn như HTML, CSS và JavaScript.
Trong khi đó, Amazon CloudFront là dịch vụ Content Delivery Network, hay CDN, hỗ trợ phân phối nội dung thông qua hệ thống các điểm hiện diện.
Việc kết hợp S3 và CloudFront giúp xây dựng một cơ chế lưu trữ và phân phối frontend phù hợp với môi trường Cloud.
Như vậy, tám lớp hạ tầng này đảm nhận những chức năng khác nhau, từ phân quyền, kết nối mạng, lưu trữ dữ liệu cho đến triển khai container và phân phối giao diện người dùng.
Khi được kết hợp, chúng tạo thành nền tảng hạ tầng phục vụ quá trình vận hành hệ thống RideNow trên AWS.
**[Chèn hình 4: Sơ đồ kiến trúc tổng thể của RideNow gồm 8 lớp hạ tầng AWS]**
## 5.1.3. Mối quan hệ giữa các lớp hạ tầng
Các lớp hạ tầng vừa trình bày không hoạt động hoàn toàn độc lập mà có mối quan hệ phụ thuộc với nhau.
Ví dụ, trước khi triển khai các Microservice lên Amazon EKS, chúng ta cần chuẩn bị hạ tầng mạng như VPC và Subnets.
Đồng thời, các service cũng cần được cấp quyền IAM phù hợp để truy cập những tài nguyên AWS cần thiết.
Khi triển khai ứng dụng dưới dạng container, các Docker Image được lưu trữ trên Amazon ECR sẽ được sử dụng làm nguồn image cho các workload trên EKS.
Trong quá trình hoạt động, các service có thể kết nối đến cơ sở dữ liệu PostgreSQL trên Amazon RDS, sử dụng SNS và SQS để trao đổi thông điệp hoặc truy cập ElastiCache để sử dụng dữ liệu được lưu trong bộ nhớ đệm.
Đối với những yêu cầu truy cập từ người dùng, Application Load Balancer đảm nhận vai trò tiếp nhận và định tuyến lưu lượng đến các thành phần ứng dụng phù hợp.
Ở phía frontend, các tập tin giao diện được lưu trữ trên Amazon S3 và phân phối thông qua CloudFront.
Có thể thấy, việc triển khai cần được tổ chức theo một thứ tự hợp lý vì cấu hình của một lớp hạ tầng có thể phụ thuộc vào tài nguyên hoặc thông tin được tạo ở lớp trước.
Đây cũng là lý do chương được chia thành nhiều phần nhỏ, thay vì tạo tất cả các tài nguyên cùng một lúc.
