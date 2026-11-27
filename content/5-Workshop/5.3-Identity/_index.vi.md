---
title : "Định danh"
date : 2024-01-01 
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

# PHẦN 01. CẤU HÌNH IAM TRÊN AWS CONSOLE
## 1. Giới thiệu IAM
Đầu tiên, em sẽ tiến hành cấu hình IAM trên AWS Console để chuẩn bị các quyền truy cập cần thiết cho hệ thống RideNow.
IAM là viết tắt của Identity and Access Management, đây là dịch vụ của AWS dùng để quản lý danh tính và phân quyền truy cập vào các tài nguyên AWS.
Trong hệ thống RideNow, các thành phần như EKS Cluster, Trip Service, Payment Service, Matching Service và Notification Worker cần có những quyền truy cập khác nhau.
Vì vậy, ở bước đầu tiên, em sẽ tạo 2 IAM Roles và 5 IAM Policies dựa trên file cấu hình `iac/stacks/01-iam.yaml`.
**Hình ảnh minh họa:** Giao diện IAM Dashboard trên AWS Console.
## 2. Tạo IAM Role cho EKS Cluster
Đầu tiên, em vào AWS Console, tìm kiếm dịch vụ IAM và chọn mục Roles.
Sau đó, em chọn Create role để tạo một IAM Role mới.

Ở phần Trusted entity type, em chọn AWS service. Tiếp theo, em chọn EKS và trường hợp sử dụng EKS - Cluster để AWS thiết lập quan hệ tin cậy phù hợp.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-eks-cluster-role-trusted-entity.png)

![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-eks-cluster-role-use-case.png)
Ở bước cấu hình quyền, em sử dụng policy `AmazonEKSClusterPolicy` theo yêu cầu dành cho EKS Cluster.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-eks-cluster-role-attach-policy.png)

Sau đó, em đặt tên Role là:
`ridenow-prod-eks-cluster-role`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-eks-cluster-role-name.png)

Role này sẽ được sử dụng khi tạo EKS Cluster, cho phép dịch vụ EKS thực hiện các thao tác cần thiết với tài nguyên AWS để vận hành và quản lý Cluster.

Sau khi truy cập vào Role, em chọn tab **Permissions** để kiểm tra những quyền đã được cấp.
Tại đây, em kiểm tra xem Role đã được gắn policy `AmazonEKSClusterPolicy` hay chưa.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-eks-cluster-role-permissions.png)

![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-eks-cluster-policy-details.png)
Đây là policy do AWS quản lý, cung cấp những quyền cần thiết để Amazon EKS thực hiện các tác vụ quản lý Cluster.
Nếu Role chưa có policy này, em chọn **Add permissions**, sau đó chọn **Attach policies**.
Tiếp theo, em tìm kiếm `AmazonEKSClusterPolicy`, tích chọn và nhấn **Add permissions** để gắn policy vào Role.
Sau khi hoàn thành, em kiểm tra lại danh sách Permissions để bảo đảm policy đã được gắn thành công.

Tiếp theo, em chuyển sang tab **Trust relationships**.

Đối với EKS Cluster Role, dịch vụ được phép đảm nhận Role phải là Amazon EKS.
Vì vậy, trong phần Trust policy, em kiểm tra hai thông tin quan trọng.
Thông tin thứ nhất là `eks.amazonaws.com`, xác định Amazon EKS là dịch vụ được tin cậy.
Thông tin thứ hai là `sts:AssumeRole`, cho phép dịch vụ EKS đảm nhận IAM Role để sử dụng các quyền đã được cấp.
Nhờ cấu hình này, Amazon EKS có thể đảm nhận Role để thực hiện các thao tác cần thiết trong quá trình vận hành Cluster.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-eks-cluster-role-trust-policy.png)

Trong trường hợp Trust policy chưa được cấu hình chính xác, em chọn **Edit trust policy** để chỉnh sửa.
Tại đây, em sử dụng cấu hình JSON để khai báo dịch vụ Amazon EKS là đối tượng được tin cậy.
Cụ thể, thuộc tính `Effect` có giá trị `Allow`, thể hiện việc cho phép thực hiện hành động.
Thuộc tính `Principal` được cấu hình là `eks.amazonaws.com`, xác định dịch vụ Amazon EKS được phép đảm nhận Role.
Cuối cùng, thuộc tính `Action` có giá trị `sts:AssumeRole`, cho phép dịch vụ EKS sử dụng cơ chế AWS Security Token Service để đảm nhận Role.

Sau khi kiểm tra các thông tin, em nhấn **Update policy** để lưu cấu hình.
Như vậy, em đã hoàn thành việc kiểm tra và cấu hình IAM Role cho EKS Cluster.
Role này sẽ được sử dụng trong bước triển khai Amazon EKS tiếp theo, giúp Cluster có được những quyền cần thiết để hoạt động trên hạ tầng AWS.

## 3. Tạo IAM Role cho Fargate Pod Execution
Tiếp theo, em sẽ tạo IAM Role thứ hai dành cho AWS Fargate.
Fargate là dịch vụ cho phép chạy các container mà không cần trực tiếp quản lý máy chủ EC2.
Trong hệ thống RideNow, Fargate có thể được sử dụng để chạy các Pod của ứng dụng bên trong EKS.
Để tạo Role, em tiếp tục vào IAM, chọn Roles và Create role.
Ở phần Trusted entity type, em chọn AWS service và chọn EKS - Fargate pod.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-fargate-pod-role-trusted-entity.png)

Tiếp theo, em gắn policy:
`AmazonEKSFargatePodExecutionRolePolicy`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-fargate-pod-role-attach-policy.png)
Sau đó, em đặt tên:
`ridenow-prod-eks-fargate-pod-execution-role`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-fargate-pod-execution-role-name.png)
Cuối cùng, em chọn Create role.

Role này cho phép hạ tầng Fargate thực hiện những thao tác cần thiết khi chạy Pod, chẳng hạn như tải container image từ Amazon ECR.
Role này không trực tiếp cấp quyền truy cập tài nguyên AWS cho mã nguồn ứng dụng bên trong Pod.

## 4. Tạo App Access Policy
Sau khi hoàn thành việc tạo hai IAM Roles cho EKS Cluster và Fargate, tiếp theo em tiến hành tạo 5 IAM Policies dành cho các microservices trong hệ thống RideNow.
Trong hệ thống RideNow, các microservices cần tương tác với nhiều dịch vụ AWS như Amazon ECR, Secrets Manager, SQS và SNS.  
Vì vậy, em tạo các Customer Managed Policies để phân quyền truy cập cho từng dịch vụ, thay vì cấp toàn bộ quyền cho tất cả microservices.  
Hệ thống gồm 5 Policies:
- AppAccessPolicy
- TripServiceMessagingPolicy
- PaymentServiceMessagingPolicy
- MatchingServiceMessagingPolicy
- NotificationWorkerMessagingPolicy  
    Mỗi Policy sẽ đảm nhiệm một nhóm quyền truy cập riêng, giúp hệ thống dễ quản lý và hạn chế việc cấp quyền không cần thiết.  
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-app-access-policy-configuration.png)

Đầu tiên, em vào IAM, chọn Policies và chọn Create policy.
Tại đây, em chuyển sang chế độ JSON để khai báo các quyền truy cập.

Em sẽ sử dụng phần PolicyDocument tương ứng với `AppAccessPolicy` trong file `iac/stacks/01-iam.yaml`.

Sau khi nhập nội dung JSON, em chọn Next và đặt tên:
`ridenow-prod-app-access`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-app-access-policy-name.png)

Cuối cùng, em chọn Create policy.
Policy này được sử dụng để định nghĩa các quyền truy cập tài nguyên AWS dành cho ứng dụng RideNow.
Các hành động và tài nguyên cụ thể mà policy cho phép sẽ phụ thuộc vào nội dung JSON đã cấu hình.

## 5. Tạo Trip Service Messaging Policy
Tiếp theo, em sẽ tạo policy dành cho Trip Service.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-trip-service-messaging-policy-configuration.png)


Trip Service là microservice chịu trách nhiệm xử lý các nghiệp vụ liên quan đến chuyến xe.
Trong kiến trúc microservices, Trip Service có thể cần trao đổi thông điệp với các thành phần khác thông qua dịch vụ messaging của AWS.
Em tiếp tục vào IAM, chọn Policies và Create policy.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-trip-service-messaging-policy-creation.png)
Sau đó, em chọn JSON và nhập nội dung quyền từ `TripServiceMessagingPolicy` trong file YAML.
Em đặt tên policy là:
`ridenow-prod-trip-service-messaging`
Sau đó, em chọn Create policy.
Mục đích của policy này là giới hạn quyền truy cập vào các tài nguyên messaging cần thiết cho Trip Service, thay vì cấp quyền truy cập rộng cho toàn bộ hệ thống.

## 6. Tạo Payment Service Messaging Policy
Tiếp theo, em sẽ cấu hình policy dành cho Payment Service.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-payment-service-messaging-policy-configuration.png)
Payment Service chịu trách nhiệm xử lý các nghiệp vụ liên quan đến thanh toán trong hệ thống RideNow.
Do đó, service này có thể cần sử dụng hệ thống messaging để gửi hoặc nhận các thông điệp liên quan đến quá trình thanh toán.
Tương tự các bước trước, em vào IAM, chọn Policies, sau đó chọn Create policy.
Em nhập phần PolicyDocument tương ứng với `PaymentServiceMessagingPolicy`.
Tên policy là:
`ridenow-prod-payment-service-messaging`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-payment-service-messaging-policy-name.png)
Sau khi kiểm tra lại nội dung, em chọn Create policy để hoàn tất.
Việc tạo policy riêng cho Payment Service giúp phân tách quyền truy cập của dịch vụ thanh toán với các microservices khác.

## 7. Tạo Matching Service Messaging Policy
Tiếp theo, em tạo policy dành cho Matching Service.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-matching-service-messaging-policy-configuration.png)
Matching Service là thành phần chịu trách nhiệm xử lý nghiệp vụ ghép tài xế với yêu cầu đặt xe của khách hàng.
Trong quá trình hoạt động, Matching Service có thể cần trao đổi thông điệp với các service khác để nhận yêu cầu ghép xe hoặc gửi kết quả xử lý.
Em vào IAM, chọn Policies và Create policy.
Sau đó, em nhập JSON được định nghĩa trong `MatchingServiceMessagingPolicy`.
Em đặt tên:
`ridenow-prod-matching-service-messaging`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-matching-service-messaging-policy-name.png)
Cuối cùng, em chọn Create policy.
Policy này giúp xác định phạm vi quyền truy cập messaging dành riêng cho Matching Service.

## 8. Tạo Notification Worker Messaging Policy
Policy cuối cùng em cần tạo là Notification Worker Messaging Policy.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5.3-notification-worker-messaging-policy-configuration.png)
Notification Worker là thành phần xử lý các tác vụ thông báo trong hệ thống RideNow.
Ví dụ, khi một chuyến xe được xác nhận hoặc trạng thái chuyến xe thay đổi, hệ thống có thể phát sinh thông điệp để Notification Worker tiếp nhận và xử lý.
Để tạo policy, em tiếp tục vào IAM, chọn Policies và Create policy.
Em nhập nội dung JSON tương ứng với `NotificationWorkerMessagingPolicy` trong file YAML.
Sau đó, em đặt tên:
`ridenow-prod-notification-worker-messaging`
Cuối cùng, em chọn Create policy.
Policy này được thiết kế để quản lý các quyền messaging cần thiết cho Notification Worker.

## Tổng kết
Như vậy, ở bước đầu tiên của quá trình triển khai hệ thống RideNow lên AWS, em đã tiến hành cấu hình IAM với tổng cộng 7 tài nguyên, bao gồm 2 IAM Roles và 5 IAM Policies.
Hai IAM Roles được sử dụng để cấp quyền cho EKS Cluster và hạ tầng Fargate.
Năm IAM Policies được sử dụng để định nghĩa quyền truy cập dành cho ứng dụng và các microservices như Trip Service, Payment Service, Matching Service và Notification Worker.
Việc tách riêng các policy giúp quản lý quyền truy cập rõ ràng hơn và hỗ trợ áp dụng nguyên tắc phân quyền tối thiểu, nghĩa là mỗi thành phần chỉ được cấp những quyền thực sự cần thiết.
Sau khi hoàn thành phần IAM, hệ thống đã có các thành phần phân quyền nền tảng để tiếp tục triển khai những tài nguyên AWS ở các bước tiếp theo.
