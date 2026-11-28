---
title : "Container Image"
date : 2024-01-01
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

# PHẦN 5.5 – TRIỂN KHAI CONTAINER IMAGE TRÊN AMAZON ECR

## 1. Giới thiệu Amazon ECR

Sau khi hoàn thành việc cấu hình hệ thống mạng và các thành phần bảo mật trên AWS, em tiếp tục thực hiện bước chuẩn bị nơi lưu trữ Docker Image cho các service của hệ thống RideNow.
Ở bước này, em sử dụng dịch vụ **Amazon Elastic Container Registry**, hay còn gọi là Amazon ECR.
Amazon ECR là dịch vụ của AWS cho phép lưu trữ và quản lý các container image, chẳng hạn như Docker Image.
Trong hệ thống RideNow, mỗi service sẽ được đóng gói thành một Docker Image riêng. Những image này sau đó được đẩy lên Amazon ECR để phục vụ quá trình triển khai ứng dụng lên Amazon EKS.
Việc sử dụng ECR giúp hệ thống có một nơi lưu trữ image tập trung, đồng thời hỗ trợ kiểm soát truy cập, mã hóa dữ liệu và quản lý các phiên bản image.

## 2. Tạo các ECR Repository cho hệ thống RideNow

Sau khi truy cập vào AWS Management Console, em tiếp tục thực hiện bước tạo các repository để lưu trữ Docker Image cho hệ thống RideNow.
Đầu tiên, em tìm kiếm dịch vụ **Amazon Elastic Container Registry**, hay còn gọi là Amazon ECR. Đây là dịch vụ của AWS cho phép lưu trữ và quản lý các Docker Image, phục vụ cho quá trình triển khai ứng dụng.

Tiếp theo, tại giao diện Amazon ECR, em truy cập vào mục **Private Repositories** và chọn **Create repository** để bắt đầu tạo repository mới.
Trong phần cấu hình, em lựa chọn chế độ **Private** nhằm giới hạn quyền truy cập vào các Docker Image của hệ thống.
Tại mục Repository name, em nhập `ridenow/user-service` để tạo repository đầu tiên, dành cho service quản lý người dùng.
Ngoài ra, em thiết lập **Image Tag Mutability là Immutable** nhằm ngăn chặn việc ghi đè các Docker Image đã sử dụng cùng một tag phiên bản.
Ở phần Encryption Settings, em lựa chọn **KMS với khóa AWS Managed Key** `**aws/ecr**` để mã hóa dữ liệu image khi lưu trữ.
Sau khi hoàn tất các thông số, em chọn Create repository để tiến hành tạo tài nguyên.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-create-private-ecr-repository.png)

Sau khi tạo thành công repository đầu tiên, em tiếp tục thực hiện tương tự cho các service còn lại.
Tổng cộng, hệ thống RideNow được chuẩn bị **7 ECR Repository**, bao gồm:
- `ridenow/user-service`: Quản lý thông tin người dùng.
- `ridenow/driver-service`: Quản lý thông tin tài xế.
- `ridenow/trip-service`: Xử lý thông tin chuyến đi.
- `ridenow/payment-service`: Xử lý các chức năng thanh toán.
- `ridenow/matching-service`: Xử lý việc ghép nối tài xế với khách hàng.
- `ridenow/notification-worker`: Xử lý các thông báo của hệ thống.
- `ridenow/realtime-gateway`: Phục vụ các chức năng giao tiếp thời gian thực.
Tất cả repository đều được cấu hình thống nhất với chế độ Private, Image Tag Immutable và mã hóa bằng AWS KMS.

![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ridenow-ecr-repositories-list.png)

Ở đây, em sử dụng quy ước đặt tên `ridenow/tên-service` để dễ dàng nhận biết các repository thuộc dự án RideNow.
Việc tạo repository riêng cho từng service cũng phù hợp với **kiến trúc Microservices** mà hệ thống đang sử dụng.
Cụ thể, mỗi service sẽ có Docker Image và các phiên bản độc lập. Khi cần cập nhật một service, em có thể build và push phiên bản image mới vào repository tương ứng mà không cần thay đổi image của những service khác.
Ví dụ, nếu em cập nhật chức năng quản lý chuyến đi, em chỉ cần tạo phiên bản Docker Image mới cho `trip-service` và đẩy lên repository `ridenow/trip-service`.
Nhờ đó, việc quản lý, cập nhật và triển khai các thành phần của hệ thống trở nên rõ ràng và thuận tiện hơn.
**Như vậy, sau bước này, em đã chuẩn bị đầy đủ 7 ECR Repository cho hệ thống RideNow.** Các repository này sẽ được sử dụng để lưu trữ Docker Image và phục vụ quá trình triển khai các service lên Amazon EKS ở những bước tiếp theo.

Tiếp theo, em sẽ trình bày về **cấu hình Scan on Push**, nhằm hỗ trợ tự động kiểm tra lỗ hổng bảo mật của Docker Image khi được đẩy lên Amazon ECR.

Trong quá trình tạo repository, tại mục Visibility Settings, em chọn chế độ Private.
Điều này có nghĩa là các Docker Image của hệ thống không được công khai để bất kỳ người dùng nào trên Internet cũng có thể tải về.
Thay vào đó, việc truy cập repository sẽ được kiểm soát thông qua cơ chế phân quyền của AWS, chẳng hạn như IAM.
Khi triển khai ứng dụng lên Amazon EKS, các thành phần được cấp quyền phù hợp mới có thể lấy Docker Image từ ECR.
Nhờ đó, em có thể hạn chế các truy cập không được phép vào những image của hệ thống.

Tiếp theo, tại mục Image Tag Mutability, em lựa chọn chế độ Immutable.
Thông thường, khi xây dựng Docker Image, chúng ta sẽ sử dụng các tag để xác định phiên bản, ví dụ như `v1.0.0`, `v1.0.1` hoặc `v2.0.0`.
Khi bật Immutable, một tag đã được sử dụng trong repository sẽ không thể bị ghi đè bằng một image khác.
Ví dụ, nếu em đã push một Docker Image với tag `v1.0.0`, thì những lần push tiếp theo không thể thay thế image của tag này.
Nếu cần triển khai phiên bản mới, em phải sử dụng một tag khác, chẳng hạn như `v1.0.1`.
Cấu hình này giúp hạn chế việc vô tình ghi đè phiên bản đã triển khai, đồng thời hỗ trợ quản lý và truy vết các phiên bản ứng dụng.

Sau khi thiết lập Immutable, em tiếp tục cấu hình mã hóa dữ liệu cho repository.
Tại phần Encryption Settings, em chọn KMS và sử dụng khóa do AWS quản lý có tên `aws/ecr`.
AWS KMS là dịch vụ quản lý khóa mã hóa của AWS.
Khi sử dụng khóa này, dữ liệu Docker Image được lưu trữ trong Amazon ECR sẽ được mã hóa khi lưu trữ.
Điều này giúp tăng cường khả năng bảo vệ dữ liệu của hệ thống.
Ngoài ra, việc sử dụng AWS Managed Key cũng giúp em không cần tự tạo và quản lý một khóa KMS riêng.
Em áp dụng cùng một cấu hình mã hóa cho cả 7 repository của RideNow.

## 3. Cấu hình Scan on Push – Kiểm tra bảo mật Docker Image trên Amazon ECR

### 3.1. Tổng quan về Scan on Push

Sau khi hoàn thành việc tạo 7 ECR Repository cho hệ thống RideNow, em tiếp tục thực hiện cấu hình Scan on Push nhằm tăng cường khả năng bảo mật cho các Docker Image.
Trước tiên, chúng ta cần hiểu tại sao Docker Image cần được kiểm tra bảo mật.
Trong kiến trúc Microservices, mỗi service thường được đóng gói thành một Docker Image riêng biệt. Tuy nhiên, một Docker Image không chỉ chứa mã nguồn ứng dụng mà còn bao gồm các thành phần như môi trường thực thi, thư viện và các gói phần mềm phụ thuộc.
Những thành phần này có thể tồn tại các lỗ hổng bảo mật, đặc biệt khi sử dụng những phiên bản phần mềm cũ hoặc chưa được cập nhật.
Chính vì vậy, Amazon ECR cung cấp tính năng **Scan on Push**, cho phép tự động kích hoạt quá trình quét bảo mật khi một Docker Image mới được đẩy lên repository.
Ví dụ, khi em xây dựng một phiên bản mới của `user-service` và đẩy Docker Image lên Amazon ECR, hệ thống sẽ tự động thực hiện quá trình quét nhằm phát hiện những lỗ hổng bảo mật đã được nhận diện.
Nhờ đó, người phát triển có thể kiểm tra và đánh giá các vấn đề bảo mật trước khi sử dụng image để triển khai ứng dụng trên Amazon EKS.
Tuy nhiên, Scan on Push chỉ hỗ trợ phát hiện lỗ hổng, chứ không tự động sửa chữa hoặc đảm bảo Docker Image hoàn toàn an toàn.

### 3.2. Cơ chế Basic Scanning trong Amazon ECR

Để triển khai Scan on Push, Amazon ECR cung cấp cơ chế Basic Scanning.
Basic Scanning là phương thức quét bảo mật giúp phát hiện các lỗ hổng đã biết trong những thành phần phần mềm được hỗ trợ bên trong Container Image.
Quá trình quét sẽ kiểm tra thông tin về các gói phần mềm và đối chiếu với dữ liệu lỗ hổng bảo mật đã được công bố.
Một khái niệm quan trọng trong quá trình này là **CVE**, viết tắt của Common Vulnerabilities and Exposures.
CVE là hệ thống định danh các lỗ hổng bảo mật đã được công khai.
Ví dụ, nếu Docker Image của `payment-service` sử dụng một phiên bản thư viện có lỗ hổng đã biết, Basic Scanning có thể phát hiện và cung cấp thông tin về lỗ hổng đó.
Các phát hiện bảo mật có thể được phân loại theo mức độ nghiêm trọng như Critical, High, Medium hoặc Low, giúp người phát triển xác định vấn đề cần ưu tiên xử lý.
Trong hệ thống RideNow, em lựa chọn Basic Scanning vì đây là cơ chế phù hợp với yêu cầu kiểm tra lỗ hổng cơ bản của các Docker Image.
Cấu hình này giúp bổ sung một lớp bảo mật trong quá trình quản lý container mà không yêu cầu xây dựng một hệ thống quét riêng biệt.

### 3.3. Cấu hình Basic Scanning trên AWS Console

Sau khi tìm hiểu cơ chế hoạt động, em tiến hành áp dụng vào hệ thống RideNow trên AWS.
Đầu tiên, em truy cập AWS Management Console và lựa chọn dịch vụ **Amazon Elastic Container Registry**, hay Amazon ECR.
Tại đây, em sử dụng Region **Asia Pacific (Singapore)**, tương ứng với `ap-southeast-1`, là khu vực triển khai tài nguyên của hệ thống.
Trong giao diện Amazon ECR, em truy cập vào phần **Private Registry**, sau đó chọn **Scanning** để mở trang Scanning Configuration.
Đây là nơi AWS cho phép thiết lập cơ chế quét bảo mật đối với các Container Image được lưu trữ trong Private Registry.

Tại phần Scan Type, em lựa chọn **Basic Scanning**.
Việc lựa chọn này có nghĩa là em sử dụng cơ chế quét cơ bản của Amazon ECR để kiểm tra những lỗ hổng bảo mật đã biết trong các thành phần phần mềm được hỗ trợ.
Sau khi lựa chọn Basic Scanning, em tiếp tục cấu hình Scan on Push để quá trình quét được kích hoạt tự động khi có Docker Image mới được đẩy lên repository.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-basic-scanning-configuration.png)

### 3.4. Thiết lập Repository Filter cho hệ thống RideNow

Tiếp theo, em cần xác định những repository nào sẽ được áp dụng tính năng Scan on Push.
Trong hệ thống RideNow, em đã tạo 7 ECR Repository tương ứng với 7 service, bao gồm quản lý người dùng, tài xế, chuyến đi, thanh toán, ghép nối tài xế, xử lý thông báo và giao tiếp thời gian thực.
Các repository này đều sử dụng một quy ước đặt tên chung là `ridenow/tên-service`.
Vì vậy, thay vì cấu hình quét riêng lẻ cho từng repository, em sử dụng **Repository Filter** để áp dụng chung một chính sách.
Tại mục Scan on Push Filters, em thiết lập giá trị:
`ridenow/*`
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-scan-on-push-repository-filter.png)
Trong đó, `ridenow/` là tiền tố chung của các repository thuộc hệ thống RideNow, còn ký tự `*` đại diện cho phần tên service phía sau.
Chẳng hạn, bộ lọc này sẽ áp dụng cho `ridenow/user-service`, `ridenow/trip-service`, `ridenow/payment-service` và những repository khác có cùng tiền tố.
Điểm quan trọng của cách cấu hình này là **chỉ cần một bộ lọc để áp dụng Scan on Push cho toàn bộ 7 repository**.
Ngoài ra, nếu trong tương lai hệ thống phát triển thêm một service mới và repository của service đó cũng sử dụng tiền tố `ridenow/`, cấu hình Scan on Push hiện tại vẫn có thể áp dụng cho repository mới.
Điều này giúp việc quản lý bảo mật trở nên đơn giản, nhất quán và dễ mở rộng hơn khi hệ thống phát triển.

### 3.5. Cơ chế hoạt động của Scan on Push trong hệ thống RideNow

Lúc này, Amazon ECR đã được cấu hình để tự động quét những Docker Image mới được đẩy lên các repository phù hợp với bộ lọc `ridenow/*`.
Để hiểu rõ hơn, em xin trình bày một ví dụ thực tế trong hệ thống RideNow.
Giả sử nhóm phát triển vừa hoàn thành việc cập nhật chức năng xử lý chuyến đi trong `trip-service`.
Đầu tiên, mã nguồn của service được đóng gói thành một Docker Image thông qua quá trình Docker Build.
Sau đó, Docker Image được gắn một tag phiên bản, chẳng hạn như `v1.0.0`, rồi được đẩy lên repository `ridenow/trip-service` bằng Docker Push.
Khi Amazon ECR tiếp nhận image mới, hệ thống nhận diện repository này thuộc bộ lọc `ridenow/*` và tự động kích hoạt quá trình quét bảo mật.
Trong quá trình quét, Amazon ECR kiểm tra các thành phần phần mềm được hỗ trợ bên trong image để tìm kiếm những lỗ hổng đã biết.
Sau khi quá trình quét hoàn tất, kết quả sẽ được ghi nhận để người phát triển có thể xem xét.
Nếu phát hiện lỗ hổng bảo mật, nhóm phát triển có thể kiểm tra nguyên nhân, cập nhật thư viện hoặc thay đổi Base Image, sau đó xây dựng và đẩy một phiên bản Docker Image mới.
Như vậy, toàn bộ luồng xử lý được mô tả như sau:
**Source Code → Docker Build → Docker Push → Amazon ECR → Basic Scanning → Scan Results**
Thông qua cơ chế này, việc kiểm tra bảo mật được tích hợp trực tiếp vào quá trình đưa Docker Image lên môi trường lưu trữ.
Tuy nhiên, cần lưu ý rằng Scan on Push không tự động chặn việc triển khai một image có lỗ hổng. Để thực hiện điều đó, hệ thống cần bổ sung các chính sách kiểm soát trong quy trình CI/CD hoặc triển khai.

### 3.6. Vai trò của Scan on Push trong kiến trúc Microservices

Trong hệ thống RideNow, mỗi Microservice được phát triển và triển khai tương đối độc lập.
Điều này mang lại khả năng mở rộng và bảo trì linh hoạt, nhưng đồng thời cũng làm tăng số lượng Docker Image cần quản lý.
Mỗi service có thể sử dụng các thư viện và phiên bản phần mềm khác nhau, dẫn đến những rủi ro bảo mật khác nhau.
Nếu không có cơ chế kiểm tra tập trung, việc phát hiện lỗ hổng có thể phụ thuộc nhiều vào thao tác thủ công của từng nhóm phát triển.
Với Scan on Push, Amazon ECR cung cấp một cơ chế quét thống nhất cho các image mới được đẩy lên repository của RideNow.
Khi một service được cập nhật và tạo ra Docker Image mới, quá trình quét bảo mật sẽ được tự động kích hoạt theo cấu hình.
Điều này giúp phát hiện sớm các vấn đề liên quan đến thành phần phần mềm bên trong image, trước khi nhóm phát triển quyết định sử dụng phiên bản đó để triển khai.
Ngoài ra, việc kết hợp Scan on Push với Private Repository, Image Tag Immutable và mã hóa bằng AWS KMS giúp hệ thống có nhiều lớp bảo vệ khác nhau.
Trong đó, Private Repository hỗ trợ kiểm soát quyền truy cập, Immutable hạn chế việc ghi đè image đã có tag, KMS bảo vệ dữ liệu image khi lưu trữ, còn Scan on Push hỗ trợ phát hiện lỗ hổng bảo mật.
Mỗi cơ chế đảm nhận một chức năng riêng, góp phần tăng cường khả năng bảo mật trong quá trình quản lý và triển khai các service của RideNow.

## 4. Cấu hình Lifecycle Rule thứ nhất – Tự động xóa Docker Image không có tag sau 14 ngày

### 4.1. Tổng quan về Lifecycle Policy trong Amazon ECR

Sau khi hoàn thành việc cấu hình Scan on Push nhằm kiểm tra các lỗ hổng bảo mật của Docker Image, em tiếp tục thực hiện cấu hình **Lifecycle Policy** trong Amazon Elastic Container Registry.
Trước tiên, chúng ta cần hiểu Lifecycle Policy là gì và tại sao cần sử dụng tính năng này.
Trong quá trình phát triển phần mềm, các service thường xuyên được cập nhật để bổ sung tính năng, sửa lỗi hoặc cải thiện hiệu suất.
Mỗi lần cập nhật, lập trình viên có thể xây dựng một Docker Image mới và đẩy lên Amazon ECR để phục vụ quá trình triển khai.
Ví dụ, trong hệ thống RideNow, `user-service` có thể được phát triển qua nhiều phiên bản như `v1.0.0`, `v1.0.1` và `v1.0.2`.
Khi số lượng phiên bản ngày càng tăng, các Docker Image cũ có thể tiếp tục tồn tại trong repository ngay cả khi không còn được sử dụng.
Điều này khiến dung lượng lưu trữ tăng lên và có thể làm phát sinh thêm chi phí.
Để giải quyết vấn đề này, Amazon ECR cung cấp tính năng **Lifecycle Policy**, cho phép thiết lập các quy tắc tự động quản lý và xóa những Docker Image đáp ứng các điều kiện nhất định.
Các điều kiện có thể dựa trên trạng thái tag, thời gian tồn tại hoặc số lượng image trong repository.
Nhờ đó, hệ thống có thể tự động dọn dẹp những image không còn cần thiết, thay vì yêu cầu người quản trị phải kiểm tra và xóa thủ công.”

### 4.2. Khái niệm Tagged Image và Untagged Image

Để hiểu cách hoạt động của Lifecycle Policy, chúng ta cần phân biệt hai khái niệm quan trọng là **Tagged Image** và **Untagged Image**.
Tagged Image là Docker Image được gắn một hoặc nhiều tag để xác định phiên bản hoặc mục đích sử dụng.
Ví dụ:
`ridenow/user-service:v1.0.0`
Trong đó, `ridenow/user-service` là tên repository và `v1.0.0` là tag của image.
Tag giúp người phát triển xác định phiên bản nào đang được sử dụng và thuận tiện cho quá trình triển khai ứng dụng.
Ngược lại, Untagged Image là image hiện không còn tag nào được gắn với nó.
Trong quá trình cập nhật image, đặc biệt khi một tag được chuyển sang image khác ở repository cho phép ghi đè tag, image cũ có thể trở thành Untagged nếu không còn tag nào khác.
Ngoài ra, trạng thái Untagged cũng có thể xuất hiện khi các tag của image bị xóa.

Tuy nhiên, cần lưu ý rằng **Untagged không đồng nghĩa với việc image chắc chắn không còn được sử dụng**.
Một workload có thể tham chiếu image bằng digest thay vì tag. Vì vậy, trước khi áp dụng chính sách xóa, người quản trị cần đảm bảo các image thuộc diện dọn dẹp không còn cần thiết cho hệ thống.
Trong dự án RideNow, các repository được cấu hình Image Tag Immutable để hạn chế việc ghi đè phiên bản. Do đó, việc tạo ra Untagged Image do ghi đè tag không phải là tình huống thông thường, nhưng các image không có tag vẫn có thể xuất hiện trong quá trình quản lý repository.
Lifecycle Rule thứ nhất được xây dựng để xử lý nhóm image này.

### 4.3. Nguyên lý hoạt động của Lifecycle Rule theo thời gian

Amazon ECR cho phép thiết lập Lifecycle Rule dựa trên thời gian tồn tại của Docker Image.
Ở đây, em sử dụng điều kiện **Untagged kết hợp với thời hạn 14 ngày**.
Điều đó có nghĩa là hệ thống chỉ xem xét những image đang ở trạng thái không có tag và đã vượt quá khoảng thời gian được cấu hình.
Ví dụ, giả sử repository `ridenow/user-service` có ba image.
Image thứ nhất có tag `v1.0.0` và đã được lưu trữ 30 ngày.
Image thứ hai không có tag và đã được lưu trữ 5 ngày.
Image thứ ba không có tag và đã được lưu trữ 20 ngày.
Khi Lifecycle Policy được đánh giá, image thứ nhất không thuộc phạm vi của quy tắc vì vẫn có tag.
Image thứ hai tuy không có tag nhưng chưa vượt quá thời hạn 14 ngày nên chưa đủ điều kiện xóa.
Trong khi đó, image thứ ba vừa không có tag vừa vượt quá thời hạn 14 ngày, vì vậy sẽ được đưa vào diện xóa theo Lifecycle Policy.
Điểm quan trọng là thời gian này được tính dựa trên mốc thời gian lưu trữ image mà ECR sử dụng, chẳng hạn thời điểm image được push, chứ không phải thời điểm tag bị xóa.
Ngoài ra, việc xóa được Amazon ECR xử lý tự động sau khi image đáp ứng điều kiện, không có nghĩa là image sẽ bị xóa ngay lập tức tại đúng thời điểm bước sang ngày thứ 15.
Như vậy, Lifecycle Rule hoạt động dựa trên hai điều kiện kết hợp: **image phải ở trạng thái Untagged và phải vượt quá thời hạn 14 ngày**.

### 4.4. Cấu hình Lifecycle Rule trên AWS Console

Em vào danh sách Private Repositories và lựa chọn repository `ridenow/user-service`.
Tại giao diện quản lý repository, em truy cập phần **Lifecycle Policy** để cấu hình quy tắc quản lý vòng đời Docker Image.
Đây là nơi Amazon ECR cho phép người quản trị thiết lập những điều kiện tự động xác định image cần được xóa.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-lifecycle-policy-untagged-14-days.png)
Sau đó, em chọn Create Rule để tạo quy tắc đầu tiên.
Tại mục Rule Priority, em thiết lập giá trị là `1`.
Rule Priority được sử dụng để xác định thứ tự ưu tiên giữa các quy tắc trong cùng một Lifecycle Policy.
Thông thường, giá trị càng nhỏ thì mức ưu tiên càng cao. Vì đây là quy tắc đầu tiên của hệ thống, em lựa chọn Priority bằng 1.

Tại mục Image Status, em chọn **Untagged**.
Cấu hình này giúp giới hạn phạm vi áp dụng, chỉ xem xét những image hiện không được gắn tag.
Tại Match Criteria, em lựa chọn điều kiện thời gian kể từ khi image được đưa vào repository và thiết lập giá trị **14 ngày**.
Cuối cùng, tại Rule Action, em chọn **Expire**.
Expire là hành động yêu cầu Amazon ECR xóa các image đáp ứng điều kiện của quy tắc.

### 4.5. Vai trò của Lifecycle Policy trong tối ưu chi phí và quản lý hệ thống

Lifecycle Policy không chỉ giúp tự động hóa quá trình dọn dẹp Docker Image mà còn đóng vai trò quan trọng trong việc quản lý tài nguyên trên AWS.
Thứ nhất là **tối ưu dung lượng lưu trữ**.
Amazon ECR lưu trữ các image và dữ liệu liên quan. Khi số lượng image tăng lên, dung lượng lưu trữ cũng có thể tăng theo, dẫn đến chi phí phát sinh.
Việc tự động xóa những image không còn cần thiết giúp hạn chế tình trạng tích lũy dữ liệu không sử dụng.
Thứ hai là **giảm công việc quản trị thủ công**.
Nếu không sử dụng Lifecycle Policy, người quản trị phải thường xuyên kiểm tra repository, xác định image cũ và thực hiện xóa bằng tay.
Khi có nhiều Microservice, công việc này có thể trở nên phức tạp và tốn thời gian.
Thứ ba là **đảm bảo tính nhất quán trong quản lý tài nguyên**.
Với hệ thống RideNow, cả 7 repository đều sử dụng cùng một quy tắc xóa image không có tag sau 14 ngày.
Nhờ đó, các service được quản lý theo cùng một chính sách, giúp việc bảo trì và mở rộng hệ thống trở nên thuận tiện hơn.
Tuy nhiên, Lifecycle Policy chỉ giúp quản lý vòng đời image trong repository. Nó không thay thế các cơ chế sao lưu, quản lý phiên bản triển khai hoặc kiểm soát bảo mật của ứng dụng.

### 4.6. Kết luận

Như vậy, trong phần này, em đã tìm hiểu và thực hiện cấu hình Lifecycle Rule thứ nhất trong Amazon ECR.
Về mặt lý thuyết, Lifecycle Policy là cơ chế cho phép tự động quản lý vòng đời Docker Image dựa trên các điều kiện được thiết lập trước.
Đối với quy tắc thứ nhất, em sử dụng Image Status là Untagged, thời hạn 14 ngày và hành động Expire.
Về mặt thực hành, em cấu hình quy tắc với Rule Priority bằng 1 trên repository `ridenow/user-service`, sau đó áp dụng tương tự cho các repository còn lại của RideNow.
Cấu hình này giúp tự động dọn dẹp những Docker Image không có tag và đã vượt quá thời hạn quy định, từ đó hạn chế việc tích lũy image không cần thiết, tiết kiệm dung lượng lưu trữ và giảm công việc quản trị.
**Có thể thấy, Lifecycle Policy là một cơ chế quan trọng giúp quản lý Container Registry hiệu quả hơn, đặc biệt đối với hệ thống Microservices có nhiều service và thường xuyên cập nhật phiên bản như RideNow.**
Tuy nhiên, quy tắc thứ nhất mới chỉ xử lý những Docker Image không có tag.
Trong thực tế, các image vẫn được gắn tag cũng có thể tích lũy qua nhiều lần cập nhật phiên bản.
Vì vậy, ở phần tiếp theo, em sẽ trình bày về **Lifecycle Rule thứ hai – Giới hạn tối đa 50 Docker Image trong mỗi repository**, nhằm quản lý số lượng image lưu trữ một cách toàn diện hơn.

## 5. Cấu hình Lifecycle Rule thứ hai – Giữ tối đa 50 Docker Image trên Amazon ECR

### 5.1. Tổng quan về quản lý số lượng Docker Image

Sau khi hoàn thành Lifecycle Rule thứ nhất, với mục đích tự động xóa những Docker Image không có tag sau 14 ngày, em tiếp tục thực hiện cấu hình Lifecycle Rule thứ hai.
Trước tiên, chúng ta cần hiểu tại sao phải giới hạn số lượng Docker Image được lưu trữ trong Amazon ECR.
Trong quá trình phát triển phần mềm, mỗi khi một service được cập nhật hoặc sửa lỗi, lập trình viên thường xây dựng một Docker Image mới.
Ví dụ, trong hệ thống RideNow, `trip-service` có thể được phát triển qua nhiều phiên bản như `v1.0.0`, `v1.0.1`, `v1.0.2` và tiếp tục tăng lên theo thời gian.
Mặc dù những phiên bản cũ có thể không còn được sử dụng trong quá trình triển khai, chúng vẫn có thể được lưu trữ trong Amazon ECR.
Nếu không có chính sách quản lý, số lượng Docker Image sẽ ngày càng tăng, gây khó khăn cho việc quản trị và làm phát sinh thêm chi phí lưu trữ.
Đặc biệt, RideNow được xây dựng theo kiến trúc Microservices với 7 service độc lập. Điều này đồng nghĩa với việc mỗi service đều có thể tạo ra nhiều phiên bản Docker Image khác nhau.
Chính vì vậy, ngoài việc xóa image không có tag, hệ thống còn cần một cơ chế giới hạn tổng số lượng image được lưu trữ trong từng repository.
Đây là mục đích của Lifecycle Rule thứ hai.”

### 5.2. Khái niệm Image Count trong Amazon ECR

Trong Amazon ECR, Image Count là số lượng Docker Image được lưu trữ trong một repository.
Khi xây dựng và đẩy các phiên bản Docker Image lên Amazon ECR, số lượng image có thể tăng lên theo thời gian.
Ví dụ, một repository có thể lưu trữ 10, 30, 50 hoặc 100 image tùy thuộc vào tần suất cập nhật của service.
Amazon ECR hỗ trợ Lifecycle Policy dựa trên số lượng image, cho phép người quản trị xác định một ngưỡng lưu trữ và tự động xóa các image cũ khi vượt ngưỡng.
Trong phần thực hành này, em lựa chọn giới hạn là **50 Docker Image cho mỗi repository**.
Điều này có nghĩa là khi số lượng image vượt quá 50, Amazon ECR sẽ đánh giá những image cũ thuộc diện cần xóa theo điều kiện của Lifecycle Policy.
Ví dụ, nếu repository `ridenow/trip-service` chứa 60 image, hệ thống sẽ xác định 10 image cũ nhất thuộc diện xóa khi xét riêng quy tắc giới hạn số lượng.
Nhờ đó, repository có thể duy trì số lượng image trong phạm vi được cấu hình, tránh tích lũy quá nhiều phiên bản cũ.
Tuy nhiên, cần lưu ý rằng giới hạn 50 image là chính sách quản lý lưu trữ, không có nghĩa Amazon ECR sẽ từ chối Docker Image mới khi repository đã có đủ 50 image.
Thay vào đó, image mới vẫn có thể được push lên repository, sau đó Lifecycle Policy sẽ đánh giá và xử lý những image thuộc diện xóa.”

### 5.3. Nguyên lý hoạt động của Lifecycle Rule với Image Status Any

Để hiểu rõ hơn về quy tắc thứ hai, chúng ta cần tìm hiểu tùy chọn Image Status trong Amazon ECR.
Ở Lifecycle Rule thứ nhất, em đã lựa chọn Image Status là Untagged. Điều này có nghĩa quy tắc chỉ áp dụng cho những Docker Image không có tag.
Tuy nhiên, nếu chỉ sử dụng quy tắc này thì những Docker Image vẫn còn tag sẽ không bị xóa theo điều kiện 14 ngày.
Ví dụ, các phiên bản `v1.0.0`, `v1.0.1` và `v1.0.2` vẫn được giữ lại vì chúng đang có tag.
Khi các phiên bản tiếp tục tăng lên, repository vẫn có thể tích lũy rất nhiều Docker Image.
Vì vậy, ở Lifecycle Rule thứ hai, em lựa chọn **Image Status là Any**.
Any có nghĩa là quy tắc có thể áp dụng cho cả Docker Image có tag và không có tag.
Khi kết hợp Image Status Any với điều kiện Image Count bằng 50, hệ thống sẽ đánh giá tổng số image trong repository và xác định các image cũ cần được xóa khi số lượng vượt ngưỡng.
Một điểm quan trọng là quy tắc này không chỉ xóa image không có tag mà còn có thể xóa những image vẫn còn tag phiên bản.
Do đó, trước khi áp dụng trong môi trường Production, người quản trị cần kiểm tra để tránh xóa những phiên bản vẫn cần thiết cho hoạt động hoặc khôi phục hệ thống.”

### 5.4. Cấu hình Lifecycle Rule thứ hai trên AWS Console

Đầu tiên, em truy cập dịch vụ Amazon Elastic Container Registry tại Region Asia Pacific Singapore, tương ứng với mã `ap-southeast-1`.
Tiếp theo, em vào phần Private Repositories và lựa chọn repository `ridenow/user-service`.
Tại giao diện quản lý repository, em truy cập phần Lifecycle Policy.
Ở đây, em đã có Lifecycle Rule thứ nhất với độ ưu tiên là 1, dùng để xóa image không có tag sau 14 ngày.
Để bổ sung quy tắc giới hạn số lượng image, em chọn Create Rule.

Trong giao diện tạo Lifecycle Rule, đầu tiên em thiết lập **Rule Priority bằng 2**.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-lifecycle-rule-keep-50-images.png)
Rule Priority là mức độ ưu tiên của một quy tắc trong Lifecycle Policy.
Giá trị càng nhỏ thì độ ưu tiên càng cao. Vì vậy, Rule 1 có độ ưu tiên cao hơn Rule 2.
Ngoài ra, trong Amazon ECR, quy tắc sử dụng Image Status Any phải có mức ưu tiên thấp nhất trong cùng một Lifecycle Policy.
Do hệ thống RideNow sử dụng hai quy tắc, việc đặt Priority bằng 2 giúp đáp ứng yêu cầu này.

Ở phần Match Criteria, em lựa chọn điều kiện **Image Count More Than**, sau đó nhập số lượng là `50`.
Cuối cùng, tại Rule Action, em lựa chọn Expire.
Expire có nghĩa Amazon ECR sẽ thực hiện xóa các Docker Image đáp ứng điều kiện của Lifecycle Rule.
Như vậy, cấu hình của quy tắc thứ hai bao gồm:
- Rule Priority: `2`
- Rule Description: `Keep only the most recent tagged images`
- Image Status: `Any`
- Match Criteria: `Image count more than 50`
- Rule Action: `Expire`

### 5.5. Kết hợp hai Lifecycle Rule trong hệ thống RideNow

Sau khi lưu thành công, repository `ridenow/user-service` sẽ có tổng cộng hai Lifecycle Rule.
Quy tắc thứ nhất có Priority bằng 1, áp dụng cho những image không có tag và đã tồn tại quá 14 ngày.
Quy tắc thứ hai có Priority bằng 2, áp dụng cho tất cả image và sử dụng điều kiện giới hạn số lượng là 50.
Hai quy tắc này bổ sung cho nhau trong quá trình quản lý Docker Image.
Quy tắc thứ nhất tập trung xử lý những image không còn tag và đã vượt quá thời gian quy định.
Trong khi đó, quy tắc thứ hai giúp hạn chế số lượng image tích lũy trong repository, kể cả những image vẫn còn tag phiên bản.
Ví dụ, nếu một repository có nhiều image không có tag đã tồn tại quá 14 ngày, những image này có thể được đưa vào diện xóa theo Rule 1.
Nếu repository vẫn có số lượng image vượt quá 50, Rule 2 sẽ tiếp tục được sử dụng để xác định những image cũ thuộc diện xóa theo điều kiện số lượng và quy tắc ưu tiên của Amazon ECR.
Cần lưu ý rằng Amazon ECR đánh giá các quy tắc trong cùng một Lifecycle Policy theo cơ chế ưu tiên và lựa chọn image, chứ không đơn giản thực hiện từng quy tắc như hai câu lệnh xóa độc lập.
Sau khi kiểm tra cấu hình, em áp dụng cùng bộ hai Lifecycle Rule cho 6 repository còn lại trong hệ thống RideNow.
Như vậy, cả 7 repository sẽ có cùng một cơ chế quản lý vòng đời Docker Image, với tổng cộng 14 Lifecycle Rule.

### 5.6. Lợi ích của việc giới hạn số lượng Docker Image

Việc thiết lập Lifecycle Rule thứ hai mang lại một số lợi ích quan trọng đối với quá trình vận hành hệ thống RideNow.
Đầu tiên là **kiểm soát dung lượng lưu trữ**.
Khi một service được cập nhật thường xuyên, nhiều phiên bản Docker Image sẽ được tạo ra. Nếu không có cơ chế dọn dẹp, repository có thể tích lũy ngày càng nhiều image.
Việc giới hạn số lượng image giúp loại bỏ các phiên bản cũ không còn cần thiết, từ đó góp phần tiết kiệm dung lượng lưu trữ.
Lợi ích thứ hai là **tối ưu chi phí AWS**.
Amazon ECR có thể phát sinh chi phí dựa trên dung lượng dữ liệu được lưu trữ. Vì vậy, việc loại bỏ những image không cần thiết có thể góp phần giảm chi phí vận hành hệ thống.
Lợi ích thứ ba là **đơn giản hóa công việc quản trị**.
Thay vì phải thường xuyên kiểm tra và xóa image thủ công, người quản trị có thể thiết lập các điều kiện để Amazon ECR tự động xử lý.
Điều này đặc biệt có ý nghĩa trong hệ thống Microservices như RideNow, vì mỗi service đều có vòng đời phát triển và cập nhật độc lập.
Cuối cùng là **duy trì tính nhất quán trong quản lý hệ thống**.
Khi cả 7 repository sử dụng cùng một Lifecycle Policy, việc kiểm soát số lượng và thời gian lưu trữ Docker Image sẽ trở nên đồng bộ và dễ quản lý hơn.
Tuy nhiên, không nên xem việc giới hạn 50 image là một biện pháp bảo mật hay sao lưu dữ liệu.
Lifecycle Policy chỉ quản lý việc lưu trữ và xóa image trong repository. Do đó, hệ thống vẫn cần có chiến lược quản lý phiên bản và khôi phục phù hợp, đặc biệt đối với những image đang được sử dụng trong môi trường Production.

## 6. Áp dụng Lifecycle Policy cho toàn bộ 7 repository

Sau khi hoàn thành hai Lifecycle Rule cho repository đầu tiên, em tiếp tục áp dụng cấu hình tương tự cho 6 repository còn lại.
Như vậy, mỗi repository của hệ thống RideNow đều có hai quy tắc quản lý vòng đời.
Quy tắc thứ nhất là xóa image không có tag sau 14 ngày.
Quy tắc thứ hai là duy trì tối đa 50 image trong mỗi repository.
Với 7 repository, tổng cộng em cấu hình 14 lifecycle rule.
Việc áp dụng cùng một bộ quy tắc giúp các repository có cách quản lý thống nhất, thuận tiện cho việc bảo trì và kiểm soát dung lượng lưu trữ.”

## 7. Kiểm tra URI, xây dựng và triển khai Docker Image lên Amazon ECR

### 7.1. Tổng quan về quá trình triển khai Docker Image

Sau khi hoàn thành việc tạo 7 ECR Repository, cấu hình Scan on Push và thiết lập các Lifecycle Policy cho hệ thống RideNow, em tiếp tục thực hiện quá trình xây dựng và đưa Docker Image lên Amazon ECR.
Trước tiên, chúng ta cần hiểu vai trò của Docker Image trong quá trình triển khai ứng dụng.
Trong kiến trúc Microservices, mỗi service được phát triển và quản lý tương đối độc lập. Để có thể triển khai các service trên những môi trường khác nhau, chúng ta cần một phương pháp đóng gói ứng dụng cùng với các thành phần cần thiết để ứng dụng hoạt động.
Docker là nền tảng hỗ trợ đóng gói ứng dụng thành các Container Image.
Một Docker Image có thể chứa mã ứng dụng, môi trường thực thi, thư viện và các thành phần phụ thuộc. Từ image này, hệ thống có thể tạo ra các container để chạy ứng dụng.
Trong dự án RideNow, hệ thống gồm 7 service, tương ứng với 7 Docker Image cần được xây dựng và quản lý độc lập.
Sau khi các image được xây dựng, chúng sẽ được đẩy lên Amazon Elastic Container Registry, hay Amazon ECR.
Amazon ECR đóng vai trò là nơi lưu trữ tập trung các Container Image, giúp hệ thống quản lý phiên bản, kiểm soát truy cập và cung cấp image cho quá trình triển khai lên Amazon EKS.
Như vậy, quy trình triển khai Container Image của RideNow có thể được mô tả như sau:
**Source Code → Docker Build → Docker Image → Docker Push → Amazon ECR → Amazon EKS**
Trong phần thực hành này, em tập trung vào việc kiểm tra các repository, xây dựng Docker Image, đẩy image lên Amazon ECR và kiểm tra kết quả sau khi triển khai lên registry.

### 7.2. Tìm hiểu Repository URI trong Amazon ECR

Trước khi xây dựng và đẩy Docker Image, em cần xác định địa chỉ của các repository đã được tạo trong Amazon ECR.
Mỗi repository trong ECR đều có một địa chỉ riêng, được gọi là **Repository URI**.
URI là viết tắt của Uniform Resource Identifier, được sử dụng để định danh một tài nguyên.
Trong Amazon ECR, Repository URI cho phép Docker xác định chính xác nơi lưu trữ và truy xuất Container Image.
Một Repository URI thường có cấu trúc:
`<account-id>.dkr.ecr.<region>.amazonaws.com/<repository-name>`
Trong đó:
- `account-id` là mã định danh tài khoản AWS.
- `dkr.ecr` thể hiện Docker Registry thuộc Amazon ECR.
- `region` là khu vực AWS nơi repository được tạo.
- `repository-name` là tên repository cần truy cập.
Ví dụ, đối với hệ thống RideNow, URI của service quản lý người dùng sẽ có cấu trúc:
`<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/ridenow/user-service`
Địa chỉ này giúp Docker xác định repository cần sử dụng khi thực hiện thao tác Push hoặc Pull Image.
Khi kết hợp URI với một tag phiên bản, chúng ta sẽ có tên đầy đủ của Docker Image.
Ví dụ:
`<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/ridenow/user-service:v1.0.0`
Trong đó, `v1.0.0` là tag được sử dụng để xác định phiên bản image.
Việc sử dụng tag giúp quản lý các phiên bản ứng dụng rõ ràng hơn, đồng thời hỗ trợ triển khai hoặc quay lại một phiên bản đã được lưu trữ khi cần thiết.”

### 7.3. Kiểm tra Repository URI trên AWS Console và AWS CLI

Tại danh sách Private Repositories, em kiểm tra các repository đã được tạo cho 7 service của hệ thống.
Ở phần thông tin repository, Amazon ECR cung cấp URI tương ứng để phục vụ quá trình Push và Pull Docker Image.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-repository-uri.png)

Ngoài việc kiểm tra trực tiếp trên Console, em còn sử dụng AWS CLI để truy xuất các thông tin này.
AWS CLI là công cụ dòng lệnh cho phép tương tác với những dịch vụ AWS thông qua các câu lệnh.
Do môi trường phát triển của em sử dụng Windows, em thực hiện các câu lệnh trong PowerShell.
Đầu tiên, em sử dụng lệnh `aws sts get-caller-identity` để xác định tài khoản AWS hiện đang được sử dụng.
Sau đó, em khai báo các biến cần thiết, bao gồm Account ID, Region và Image Tag.
Trong đó, Region được đặt là `ap-southeast-1` và Image Tag được sử dụng là `v1.0.0`.
Tiếp theo, em sử dụng lệnh `aws ecr describe-repositories` để truy xuất danh sách các repository cùng URI tương ứng.
Thông qua kết quả trả về, em có thể kiểm tra xem hệ thống đã có đầy đủ 7 repository và các địa chỉ URI đã được tạo chính xác hay chưa.
Việc kiểm tra này rất quan trọng vì nếu sử dụng sai Account ID, Region hoặc Repository URI, Docker có thể không tìm thấy repository khi thực hiện thao tác Push Image.

### 7.4. Khái niệm Dockerfile và Docker Build

Sau khi xác định được Repository URI, em tiếp tục thực hiện việc xây dựng Docker Image cho các service của RideNow.
Để hiểu quá trình này, trước tiên chúng ta cần phân biệt Dockerfile, Docker Image và Docker Container.
**Dockerfile** là một tập tin chứa các chỉ dẫn để Docker xây dựng image.
Trong Dockerfile, lập trình viên có thể khai báo môi trường thực thi, thư mục làm việc, các tập tin cần sao chép, lệnh cài đặt phụ thuộc và lệnh khởi chạy ứng dụng.
**Docker Image** là sản phẩm được tạo ra sau khi Docker thực hiện các chỉ dẫn trong Dockerfile.
Image được sử dụng như một khuôn mẫu để tạo ra container.
Trong khi đó, **Docker Container** là một phiên bản chạy của Docker Image, có môi trường thực thi và tiến trình riêng.
Có thể hiểu đơn giản rằng Dockerfile là bản hướng dẫn xây dựng, Docker Image là sản phẩm đã được đóng gói, còn Docker Container là môi trường nơi ứng dụng được thực thi.
Quá trình tạo Docker Image từ Dockerfile được gọi là **Docker Build**.
Trong hệ thống RideNow, mỗi service sẽ được đóng gói thành một image tương ứng, giúp các thành phần ứng dụng có thể được quản lý và triển khai độc lập.
Điều này đặc biệt phù hợp với kiến trúc Microservices, bởi vì khi một service được cập nhật, nhóm phát triển có thể xây dựng image mới cho riêng service đó mà không bắt buộc phải xây dựng lại toàn bộ hệ thống.

### 7.5. Xây dựng 7 Docker Image cho hệ thống RideNow

Đầu tiên, em sử dụng Docker Desktop để cung cấp Docker Engine phục vụ quá trình xây dựng Container Image.
Tiếp theo, em mở mã nguồn hệ thống RideNow trong Visual Studio Code và sử dụng PowerShell để thực hiện các câu lệnh Docker.
Trước khi build, em kiểm tra vị trí các Dockerfile và xác định Build Context phù hợp với từng service.

Build Context là phạm vi các tập tin và thư mục được cung cấp cho Docker trong quá trình xây dựng image.
Điều này đặc biệt quan trọng đối với những dự án có nhiều module hoặc thư viện dùng chung, vì Dockerfile có thể cần truy cập các tập tin nằm ngoài thư mục của một service.
Sau khi xác định đúng Dockerfile và Build Context, em sử dụng lệnh `docker buildx build` để xây dựng Docker Image.
Trong đó, tham số `--platform linux/amd64` được sử dụng để chỉ định kiến trúc nền tảng mục tiêu là Linux AMD64.
Tham số `-t` cho phép gắn tên và phiên bản cho image.

![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-build-service-docker-images.png)

Sau khi image đầu tiên được xây dựng thành công, em tiếp tục thực hiện tương tự đối với 6 service còn lại, bao gồm Driver Service, Trip Service, Payment Service, Matching Service, Notification Worker và Realtime Gateway.

Mỗi service được xây dựng thành một Docker Image riêng biệt.
Sau quá trình build, em sử dụng lệnh `docker images` để kiểm tra danh sách các image đã có trong môi trường Docker cục bộ.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-verify-local-docker-images.png)

Như vậy, sau giai đoạn này, các thành phần của RideNow đã được đóng gói thành những Container Image, sẵn sàng để đưa lên Amazon ECR.

### 7.6. Cơ chế xác thực Docker với Amazon ECR

Trước khi thực hiện Push Docker Image, em cần thiết lập kết nối xác thực giữa Docker và Amazon ECR.
Do các repository của RideNow được cấu hình ở chế độ Private, Docker không thể tự do đẩy hoặc tải image nếu không có quyền truy cập phù hợp.
Amazon ECR sử dụng cơ chế phân quyền của AWS, trong đó IAM được sử dụng để quản lý những hành động mà một danh tính AWS được phép thực hiện.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-docker-authenticate-with-ecr.png)

Để Docker có thể đăng nhập vào Amazon ECR, em sử dụng AWS CLI với lệnh `aws ecr get-login-password`.
Lệnh này lấy mật khẩu xác thực tạm thời từ Amazon ECR dựa trên thông tin đăng nhập AWS hiện tại.
Sau đó, mật khẩu được truyền vào lệnh `docker login` để Docker thực hiện xác thực với registry.

Token xác thực của ECR có thời hạn hiệu lực là 12 giờ, vì vậy Docker có thể cần thực hiện đăng nhập lại khi token hết hạn.

Tuy nhiên, việc đăng nhập thành công không có nghĩa là tài khoản được phép thực hiện mọi thao tác. Khả năng Push hoặc Pull Image vẫn phụ thuộc vào những quyền IAM và chính sách truy cập có hiệu lực.
Nhờ cơ chế này, hệ thống có thể kiểm soát những danh tính được phép đưa Docker Image lên ECR, hạn chế các thao tác không được cấp quyền.

### 7.7. Đẩy Docker Image lên Amazon ECR bằng Docker Push

Sau khi hoàn thành việc xác thực, em tiếp tục đẩy các Docker Image lên Amazon ECR.
Thao tác này được gọi là **Docker Push**.
Docker Push là quá trình chuyển dữ liệu của Container Image từ môi trường Docker cục bộ lên Container Registry.
Docker Image được cấu thành từ các layer. Trong quá trình push, Docker sẽ kiểm tra và tải lên những layer cần thiết để registry có thể lưu trữ image.
Để push đúng repository, image cần được gắn tên đầy đủ dựa trên Repository URI và Image Tag.
Ví dụ, với `user-service`, em sử dụng URI của repository `ridenow/user-service` và tag `v1.0.0`.
Sau đó, em thực hiện lệnh `docker push` để đưa image lên Amazon ECR.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-push-docker-images-to-ecr.png)

Sau khi push thành công image đầu tiên, em thực hiện tương tự cho 6 image còn lại.
Mỗi image được đẩy lên repository tương ứng với service của mình.
Một điểm quan trọng là trong các repository RideNow, em đã cấu hình **Image Tag Mutability là Immutable**.
Điều này có nghĩa là khi tag `v1.0.0` đã tồn tại, hệ thống không cho phép ghi đè tag đó bằng một image khác.
Nếu muốn triển khai phiên bản mới, em cần sử dụng tag mới, chẳng hạn như `v1.0.1`.
Cấu hình này giúp hạn chế việc vô tình ghi đè các phiên bản đã lưu trữ và hỗ trợ quá trình quản lý phiên bản ứng dụng.
Sau khi hoàn thành Docker Push, các image của RideNow sẽ được lưu trữ trên Amazon ECR thay vì chỉ tồn tại trên máy phát triển.

### 7.8. Kiểm tra Docker Image sau khi Push

Sau khi đẩy Docker Image lên Amazon ECR, em tiến hành kiểm tra kết quả.
Đầu tiên, em truy cập AWS Management Console và mở dịch vụ Amazon ECR.
Tại danh sách Private Repositories, em lần lượt truy cập từng repository của hệ thống RideNow.
Trong phần Images, Amazon ECR cung cấp thông tin về những image đang được lưu trữ.
Các thông tin quan trọng bao gồm Image Tag, thời gian push, dung lượng image và Image Digest.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-verify-pushed-images-in-ecr.png)

Trong đó, **Image Tag** được sử dụng để đặt tên và quản lý phiên bản image.
Còn **Image Digest** là giá trị định danh nội dung của image, thường được biểu diễn dưới dạng mã băm SHA-256.
Điểm khác biệt là tag là một tên tham chiếu phiên bản, còn digest được xác định từ nội dung image theo định dạng manifest tương ứng.
Vì vậy, digest giúp xác định chính xác image được lưu trữ, tránh nhầm lẫn giữa những phiên bản có tên tag giống nhau ở các thời điểm hoặc môi trường khác nhau.

Bên cạnh việc kiểm tra trên AWS Console, em cũng có thể sử dụng AWS CLI với lệnh `aws ecr describe-images`.
Lệnh này cho phép truy xuất thông tin image, bao gồm tag, thời gian push và digest.
Thông qua đó, em có thể xác minh Docker Image đã thực sự được lưu trữ trong đúng repository của hệ thống.

### 7.9. Kiểm tra kết quả Scan on Push sau khi triển khai Image lên ECR

Sau khi các Docker Image được push lên Amazon ECR, em tiếp tục kiểm tra kết quả quét bảo mật.
Ở phần trước, em đã cấu hình **Basic Scanning kết hợp với Scan on Push** cho các repository có tiền tố `ridenow/`.
Với cấu hình này, khi Docker Image mới được đẩy lên repository phù hợp, Amazon ECR sẽ tự động kích hoạt quá trình quét các lỗ hổng bảo mật đã biết.
Để kiểm tra kết quả, em truy cập vào repository trên AWS Console, lựa chọn image vừa được push, sau đó mở phần Scan Results hoặc Image Scan Findings.
Tại đây, hệ thống cung cấp thông tin về những lỗ hổng được phát hiện trong các thành phần phần mềm được hỗ trợ của image.
Kết quả có thể được phân loại theo các mức độ nghiêm trọng như Critical, High, Medium và Low.
![Ảnh minh họa cấu hình Amazon ECR](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-image-scan-findings.png)

Ví dụ, nếu một Docker Image sử dụng phiên bản thư viện có lỗ hổng đã được công bố, công cụ quét có thể ghi nhận thông tin lỗ hổng để người phát triển đánh giá.

Ngoài ra, em có thể sử dụng lệnh `aws ecr describe-image-scan-findings` để kiểm tra kết quả trực tiếp trong PowerShell.
Thông qua lệnh này, em có thể xem trạng thái quá trình quét và tổng hợp số lượng phát hiện theo mức độ nghiêm trọng.
Nếu xuất hiện các lỗ hổng, nhóm phát triển cần xác định thành phần bị ảnh hưởng và đánh giá mức độ rủi ro.
Tùy trường hợp, nhóm phát triển có thể cập nhật Base Image, nâng cấp thư viện hoặc sửa các thành phần phần mềm liên quan.
Sau đó, một Docker Image mới được xây dựng và đẩy lên ECR để thực hiện kiểm tra lại.
Điều quan trọng cần lưu ý là **Scan on Push chỉ hỗ trợ phát hiện lỗ hổng, không tự động sửa chữa và cũng không mặc định ngăn chặn image có lỗ hổng được triển khai**.
Do đó, kết quả quét vẫn cần được xem xét và kết hợp với các cơ chế kiểm soát trong quy trình CI/CD để đảm bảo yêu cầu bảo mật trước khi triển khai ứng dụng.

### 7.10. Vai trò của quy trình Build và Push trong kiến trúc Microservices

Trong hệ thống RideNow, việc xây dựng và lưu trữ Docker Image trên Amazon ECR đóng vai trò quan trọng trong quá trình triển khai Microservices.
Thứ nhất là **đảm bảo tính độc lập giữa các service**.
Mỗi service được đóng gói thành một image riêng biệt. Khi một service thay đổi, nhóm phát triển có thể xây dựng và triển khai phiên bản mới cho service đó mà không bắt buộc phải thay đổi image của các service khác.
Thứ hai là **quản lý phiên bản ứng dụng**.
Thông qua Image Tag và Image Digest, hệ thống có thể xác định phiên bản được lưu trữ và sử dụng trong quá trình triển khai.
Thứ ba là **tập trung hóa việc lưu trữ Container Image**.
Thay vì lưu trữ các image phân tán trên máy của từng lập trình viên, Amazon ECR cung cấp một registry tập trung, giúp các môi trường triển khai có thể truy xuất image từ cùng một nguồn.
Thứ tư là **tăng cường kiểm soát bảo mật**.
Các repository được cấu hình Private, Image Tag Immutable, mã hóa bằng AWS KMS và Scan on Push.
Những cơ chế này hỗ trợ kiểm soát truy cập, bảo vệ dữ liệu image khi lưu trữ, quản lý phiên bản và phát hiện các lỗ hổng bảo mật đã biết.
Cuối cùng là **hỗ trợ triển khai trên Amazon EKS**.
Sau khi Docker Image được lưu trữ trên ECR, hệ thống Kubernetes trên Amazon EKS có thể sử dụng image tương ứng để khởi chạy các container, với điều kiện các thành phần triển khai đã được cấp quyền và cấu hình phù hợp.
Nhờ đó, quá trình đưa ứng dụng từ mã nguồn lên môi trường vận hành trở nên có tổ chức, dễ quản lý và thuận tiện hơn khi hệ thống phát triển.

## 8. Kết luận

Như vậy, trong phần này, em đã tìm hiểu và thực hiện quy trình xây dựng, lưu trữ và kiểm tra Docker Image trên Amazon ECR cho hệ thống RideNow.
Về mặt lý thuyết, em đã trình bày các khái niệm Repository URI, Dockerfile, Docker Image, Docker Build, Docker Push, Image Tag và Image Digest.
Đồng thời, em cũng giải thích cơ chế xác thực giữa Docker và Amazon ECR thông qua AWS CLI và IAM.
Về mặt thực hành, quy trình gồm việc kiểm tra URI của 7 repository, xác định Dockerfile và Build Context, xây dựng 7 Docker Image, xác thực với ECR và push các image lên repository tương ứng.
Sau đó, em kiểm tra lại thông tin các image đã được lưu trữ và kết quả Scan on Push để xác nhận quá trình triển khai lên registry diễn ra đúng yêu cầu.
Thông qua quá trình này, hệ thống RideNow có thể quản lý Docker Image của từng Microservice một cách độc lập, đồng thời tận dụng những cơ chế kiểm soát phiên bản và bảo mật do AWS cung cấp.
**Có thể thấy, việc kết hợp Docker với Amazon ECR là một phần quan trọng trong quy trình triển khai ứng dụng theo kiến trúc Microservices, giúp tạo ra cầu nối giữa môi trường phát triển và môi trường vận hành trên nền tảng Cloud.**
Sau khi hoàn thành giai đoạn này, các Docker Image của hệ thống sẽ sẵn sàng để được sử dụng trong quá trình triển khai các service lên Amazon EKS ở những phần tiếp theo.
