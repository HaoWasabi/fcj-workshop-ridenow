---
title : "Post Deploy"
date : 2024-01-01
weight : 3
chapter : false
pre : " <b> 5.8.3. </b> "
---


## Post-deploy steps

kiểm tra đã đăng nhập aws
```bash

aws sts get-caller-identity
```
![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-57.png)
1. Cấu hình tới Kubeconfig
```bash
aws eks update-kubeconfig --name ridenow-prod --region ap-southeast-1

kubectl get svc kubernetes
```


![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-58.png)

như vậy là đã kết nối tới kubernetes

2. CoreDNS on Fargate

CoreDNS là DNS server nội bộ của Kubernetes. Nó giúp các Pod tìm thấy nhau thông qua tên Service, ví dụ backend.default.svc.cluster.local.

Mục tiêu của bước CoreDNS on Fargate là chuyển các Pod CoreDNS đang chạy trên EC2 sang chạy trên AWS Fargate.

Kiểm tra và xóa annotation ghim EC2

Remove the EC2 pin only if it is still there (the patch fails when it is not).
```bash

if kubectl get deployment coredns -n kube-system \
     -o jsonpath='{.spec.template.metadata.annotations}' | grep -q compute-type; then
  kubectl patch deployment coredns -n kube-system --type=json \
    -p='[{"op":"remove","path":"/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
fi

kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status  deployment coredns -n kube-system

```

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-61.png)

![alt text](/fcj-workshop-ridenow/images/5-Workshop/5.8-Eks/image-62.png)

1. cài AWS Load Balancer Controller vào EKS để Kubernetes có thể tự động tạo và quản lý AWS Application Load Balancer (ALB).

AWS Load Balancer Controller là một Kubernetes Controller chạy bên trong Amazon EKS, có nhiệm vụ tự động tạo, cấu hình và quản lý AWS Load Balancer dựa trên các tài nguyên Kubernetes như Ingress và Service.

Nói đơn giản: Nó là cầu nối giữa Kubernetes và AWS Load Balancing.



Đây là một CLI exception: controller được phát hành dưới dạng Helm chart, và tệp policy phụ thuộc vào phiên bản chart. Cài controller bằng Helm, dùng role ARN và VPC id từ Set variables:

```bash
helm repo add eks https://aws.github.io/eks-charts && helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=$ALB_CONTROLLER_ROLE_ARN" \
  --set region="$AWS_REGION" \
  --set vpcId="$VPC_ID"
```

Controller chưa có quyền. Nếu thiếu quyền, Ingress không có địa chỉ và controller ghi log `AccessDenied ... elasticloadbalancing:DescribeLoadBalancers`. Permissions policy phụ thuộc vào phiên bản controller, nên hãy tải đúng bản khớp với phiên bản Helm đã cài:

```bash
mkdir -p .tmp
CONTROLLER_VERSION=$(helm list -n kube-system -o json \
  | jq -r '.[] | select(.name=="aws-load-balancer-controller") | .app_version')
echo "controller version: $CONTROLLER_VERSION"
curl -fsSL -o .tmp/alb-controller-iam-policy.json \
  "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/${CONTROLLER_VERSION}/docs/install/iam_policy.json"
```

Kết quả mong đợi: một dòng `controller version: v...` và tệp `.tmp/alb-controller-iam-policy.json`.

Tạo và gắn policy trong IAM console:

1. Vào **IAM -> Policies -> Create policy -> JSON** tab. Dán nội dung của `.tmp/alb-controller-iam-policy.json`.
2. Đặt tên policy là `AWSLoadBalancerControllerIAMPolicy`, rồi tạo. Nếu policy đã có từ cluster trước, hãy dùng lại.
3. Vào **IAM -> Roles -> `ridenow-prod-alb-controller` -> Add permissions -> Attach policies**.
4. Chọn `AWSLoadBalancerControllerIAMPolicy`, rồi chọn **Add permissions**.

Sau đó restart controller để nó nhận quyền:

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
kubectl rollout status  deployment aws-load-balancer-controller -n kube-system --timeout=300s
kubectl get deploy -n kube-system aws-load-balancer-controller
```

Kết quả mong đợi: lệnh cuối hiển thị `1/1` ở cột READY.










Chạy khối thứ hai của Set variables ngay bây giờ. Các biến dưới đây được tra cứu giúp bạn, nên bạn không sao chép gì bằng tay.

| Biến | Giá trị | Dùng bởi |
|---|---|---|
| `EKS_ENDPOINT` | API server endpoint của `ridenow-prod` | Các lệnh kiểm tra kết nối trong Post-deploy steps |
| `OIDC_ISSUER_URL` | URL của OpenID Connect provider của cluster | 5.9 (OIDC provider của các IRSA role) |
| `OIDC_ID` | Id ở cuối URL của OIDC provider | 5.9 (trust policy của các IRSA role) |
| `EKS_CLUSTER_SG_ID` | ID của security group mà EKS tạo cho cluster | 5.9 (nguồn của các rule database 5432 và Redis 6379) |


Chạy các bước này từ thư mục gốc của repository, theo thứ tự. Chạy các khối của Set variables trước trong mỗi terminal mới.

Các bước dùng `kubectl`, Helm và AWS CLI. Đây là các CLI exception, vì các đối tượng Kubernetes và việc cài controller không có trang console để tạo. Chỉ các thay đổi IAM policy và security group dùng console.

Các IRSA role `ridenow-prod-alb-controller` và `ridenow-prod-external-secrets` từ 5.9 phải tồn tại trước bước 3 và 4. Hãy tạo các role đó trước, rồi chạy khối thứ ba của Set variables và kiểm tra cả hai role ARN đều có giá trị. Làm bước 3 trước bước 4.

### 1. Kubeconfig

Đây là một CLI exception: kubeconfig là tệp cục bộ nên chỉ CLI mới ghi được. Lệnh dùng danh tính IAM đã tạo cluster.

```bash
aws eks update-kubeconfig --name "$CLUSTER_NAME"

kubectl get svc kubernetes
```

Kết quả mong đợi: `Updated context ...cluster/ridenow-prod`, rồi một dòng `kubernetes   ClusterIP ... 443/TCP`.

### 2. CoreDNS on Fargate

Đây là một CLI exception: patch và restart deployment chỉ có trong Kubernetes API. Fargate không có node. Pod CoreDNS ở trạng thái `Pending` ("no nodes available") trong hai trường hợp. Cả hai được khắc phục bằng cách tạo lại pod khi Fargate profile `ridenow-prod-system` đã **Active**:

- Template CoreDNS vẫn còn annotation ghim nó vào EC2. Hãy xóa annotation đó.
- Pod được tạo trước khi Fargate profile tồn tại. Pod chỉ được ghép với profile lúc nó được tạo, nên các pod này không tự chuyển sang Fargate. Trong trường hợp này annotation có thể đã mất, và chỉ cần restart là đủ.

```bash
# Remove the EC2 pin only if it is still there (the patch fails when it is not).
if kubectl get deployment coredns -n kube-system \
     -o jsonpath='{.spec.template.metadata.annotations}' | grep -q compute-type; then
  kubectl patch deployment coredns -n kube-system --type=json \
    -p='[{"op":"remove","path":"/spec/template/metadata/annotations/eks.amazonaws.com~1compute-type"}]'
fi
# Always recreate the pods, so they are scheduled on the now-Active profile.
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status  deployment coredns -n kube-system
```

Kết quả mong đợi: `deployment "coredns" successfully rolled out`.

### 3. AWS Load Balancer Controller

Đây là một CLI exception: controller được phát hành dưới dạng Helm chart, và tệp policy phụ thuộc vào phiên bản chart. Cài controller bằng Helm, dùng role ARN và VPC id từ Set variables:

```bash
helm repo add eks https://aws.github.io/eks-charts && helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName="$CLUSTER_NAME" \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set "serviceAccount.annotations.eks\.amazonaws\.com/role-arn=$ALB_CONTROLLER_ROLE_ARN" \
  --set region="$AWS_REGION" \
  --set vpcId="$VPC_ID"
```

Controller chưa có quyền. Nếu thiếu quyền, Ingress không có địa chỉ và controller ghi log `AccessDenied ... elasticloadbalancing:DescribeLoadBalancers`. Permissions policy phụ thuộc vào phiên bản controller, nên hãy tải đúng bản khớp với phiên bản Helm đã cài:

```bash
mkdir -p .tmp
CONTROLLER_VERSION=$(helm list -n kube-system -o json \
  | jq -r '.[] | select(.name=="aws-load-balancer-controller") | .app_version')
echo "controller version: $CONTROLLER_VERSION"
curl -fsSL -o .tmp/alb-controller-iam-policy.json \
  "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/${CONTROLLER_VERSION}/docs/install/iam_policy.json"
```

Kết quả mong đợi: một dòng `controller version: v...` và tệp `.tmp/alb-controller-iam-policy.json`.

Tạo và gắn policy trong IAM console:

1. Vào **IAM -> Policies -> Create policy -> JSON** tab. Dán nội dung của `.tmp/alb-controller-iam-policy.json`.
2. Đặt tên policy là `AWSLoadBalancerControllerIAMPolicy`, rồi tạo. Nếu policy đã có từ cluster trước, hãy dùng lại.
3. Vào **IAM -> Roles -> `ridenow-prod-alb-controller` -> Add permissions -> Attach policies**.
4. Chọn `AWSLoadBalancerControllerIAMPolicy`, rồi chọn **Add permissions**.

Sau đó restart controller để nó nhận quyền:

```bash
kubectl rollout restart deployment aws-load-balancer-controller -n kube-system
kubectl rollout status  deployment aws-load-balancer-controller -n kube-system --timeout=300s
kubectl get deploy -n kube-system aws-load-balancer-controller
```

Kết quả mong đợi: lệnh cuối hiển thị `1/1` ở cột READY.

### 4. External Secrets Operator

External Secrets phải giải mã các secret RDS bằng khóa KMS từ 5.6 (alias `ridenow-prod-rds`). In ra tài liệu quyền, rồi thêm vào role trong IAM console:

```bash
cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "$RDS_KMS_KEY_ARN"
    }
  ]
}
EOF
```

1. Vào **IAM -> Roles -> `ridenow-prod-external-secrets` -> Add permissions -> Create inline policy -> JSON** tab.
2. Dán tài liệu mà khối ở trên đã in.
3. Đặt tên policy là `ridenow-prod-external-secrets-kms`, rồi tạo. Bỏ qua bước này nếu inline policy đã tồn tại.

Đây là một CLI exception: operator được phát hành dưới dạng Helm chart.

```bash
helm repo add external-secrets https://charts.external-secrets.io && helm repo update

# webhook.port=9443 is required on Fargate (the default 10250 clashes with the kubelet)
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="$EXTERNAL_SECRETS_ROLE_ARN" \
  --set webhook.port=9443 \
  --wait

kubectl get pods -n external-secrets
kubectl get crd | grep external-secrets.io
```

Kết quả mong đợi: ba pod `Running` và các CRD `external-secrets.io` được liệt kê.

### 5. Namespace

Đây là một CLI exception: namespace là đối tượng Kubernetes lấy từ một manifest trong repository.

```bash
kubectl apply -f iac/k8s/00-namespace.yaml
kubectl get namespace ridenow
```

Kết quả mong đợi: namespace `ridenow` có trạng thái `Active`.

### 6. Pod networking (per-pod security groups)

Mỗi service có security group riêng, chỉ được phép truy cập database của chính nó. Tạo sáu group và các rule database của chúng trong EC2 console. Sau đó gán chúng cho pod bằng `SecurityGroupPolicy`.

1. Vào **EC2 -> Security Groups -> Create security group**, mỗi service trong bảng dưới một lần.
2. Đặt **Security group name** và **Description** theo bảng.
3. Đặt **VPC** là `ridenow-prod-vpc`.
4. Để trống **Inbound rules** và giữ outbound rule mặc định (all traffic).
5. Ở **Tags**, thêm `Name` với giá trị bằng tên group. Lệnh tra cứu bên dưới tìm từng group theo tag này.
6. Chọn **Create security group**.

| Tên | Mô tả |
|---|---|
| `ridenow-prod-user-service-sg` | `user-service pods` |
| `ridenow-prod-driver-service-sg` | `driver-service pods` |
| `ridenow-prod-trip-service-sg` | `trip-service pods` |
| `ridenow-prod-payment-service-sg` | `payment-service pods` |
| `ridenow-prod-matching-service-sg` | `matching-service pods` |
| `ridenow-prod-notification-service-sg` | `notification-service pods` |

Sau đó thêm các rule database:

1. Vào **EC2 -> Security Groups -> `ridenow-prod-db-sg` -> Inbound rules -> Edit inbound rules**.
2. Thêm sáu rule. Đặt **Type** là `PostgreSQL`, **Port** là `5432`, và **Source** là từng group trong sáu group ở trên. Bỏ qua rule đã tồn tại.
3. Chọn **Save rules**.

Đây là một CLI exception: `SecurityGroupPolicy` là đối tượng Kubernetes. Khối này đọc sáu group id và áp dụng các policy. `EKS_CLUSTER_SG_ID` vẫn nằm trên mọi pod vì DNS và Redis cần nó:

```bash
mkdir -p .tmp
: > .tmp/sgp.yaml
for SVC in user driver trip payment matching notification; do
  SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=ridenow-${ENV}-${SVC}-service-sg" --query 'SecurityGroups[0].GroupId' --output text)
  if [ -z "$SG_ID" ] || [ "$SG_ID" = "None" ]; then echo "MISSING: ridenow-${ENV}-${SVC}-service-sg (create it in the EC2 console)"; continue; fi
  APP=${SVC}-service; [ $SVC = notification ] && APP=notification-worker
  cat >> .tmp/sgp.yaml <<EOF
---
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: ${APP}-sgp
  namespace: ridenow
spec:
  podSelector:
    matchLabels:
      app: ${APP}
  securityGroups:
    groupIds:
      - ${SG_ID}
      - ${EKS_CLUSTER_SG_ID}
EOF
  echo "$APP -> $SG_ID"
done

kubectl apply -f .tmp/sgp.yaml
kubectl rollout restart deploy -n ridenow
kubectl get securitygrouppolicy -n ridenow
```

Pod khởi động trước đó giữ security group cũ cho đến khi được restart, vì vậy có lệnh rollout restart. Lệnh này không làm gì ở lần cài đầu tiên. Kết quả mong đợi: sáu dòng `<name> -> sg-...` và sáu dòng `SecurityGroupPolicy`.

### 7. Fargate log router

Đây là một CLI exception: log router đọc một ConfigMap của Kubernetes.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: aws-observability
  labels:
    aws-observability: enabled
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-logging
  namespace: aws-observability
data:
  flb_log_cw: "true"
  output.conf: |
    [OUTPUT]
        Name cloudwatch_logs
        Match kube.*
        region ${AWS_REGION}
        log_group_name /aws/eks/ridenow/fargate
        log_stream_prefix fargate-
        auto_create_group true
EOF
```

Kết quả mong đợi: `namespace/aws-observability created` và `configmap/aws-logging created`.

### 8. Database secrets (ExternalSecrets)

Đây là một CLI exception: `ExternalSecret` là đối tượng Kubernetes, được điền từ manifest trong repository. Bước này cần sáu biến `*_DB_SECRET_ARN` từ Set variables (đều có giá trị) và bản cài External Secrets ở bước 4.

```bash
sed -e "s|[<]REGION[>]|$AWS_REGION|g" \
    -e "s|[<]ENV[>]|$ENV|g" \
    -e "s|[<]USER_SECRET_ARN[>]|$USER_DB_SECRET_ARN|g" \
    -e "s|[<]DRIVER_SECRET_ARN[>]|$DRIVER_DB_SECRET_ARN|g" \
    -e "s|[<]TRIP_SECRET_ARN[>]|$TRIP_DB_SECRET_ARN|g" \
    -e "s|[<]PAYMENT_SECRET_ARN[>]|$PAYMENT_DB_SECRET_ARN|g" \
    -e "s|[<]MATCHING_SECRET_ARN[>]|$MATCHING_DB_SECRET_ARN|g" \
    -e "s|[<]NOTIFICATION_SECRET_ARN[>]|$NOTIFICATION_DB_SECRET_ARN|g" \
    iac/k8s/10-external-secrets.yaml | kubectl apply -f -

kubectl get externalsecret -n ridenow
```

Kết quả mong đợi: sáu dòng có `SecretSynced` và READY `True`. Nếu một dòng hiển thị `SecretSyncedError`, hãy đọc lý do:

```bash
kubectl logs -n external-secrets deploy/external-secrets --tail=50 | grep -i "error\|denied"
```

### 9. ALB ingress and smoke test

Hãy triển khai bảy service từ 5.9 trước. Lệnh kiểm tra HTTP 200 bên dưới chỉ thành công sau khi các service đang chạy. Payment phải chạy trước, vì trip-service gọi nó khi tạo chuyến đi. Đây là một CLI exception: Ingress là đối tượng Kubernetes, và controller ở bước 3 tạo load balancer từ nó.

```bash
kubectl apply -f iac/k8s/30-ingress.yaml
kubectl get ingress ridenow -n ridenow
```

Chạy lại lệnh thứ hai cho đến khi cột ADDRESS hiển thị một host name. Sau đó chạy smoke test:

```bash
ALB_DNS_NAME=$(kubectl get ingress ridenow -n ridenow -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "ALB_DNS_NAME=$ALB_DNS_NAME"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "http://${ALB_DNS_NAME}/api/drivers"
kubectl logs -n ridenow deploy/matching-service --tail=20
kubectl logs -n ridenow deploy/notification-worker --tail=20
```

Kết quả mong đợi: một host name, `HTTP 200`, và log của matching và notification cho thấy consumer SQS và Valkey đã kết nối. Hãy giữ `ALB_DNS_NAME`, vì 5.10 dùng nó làm API origin của CloudFront.

`503` nghĩa là load balancer không có target khỏe. Chạy `kubectl get pods -n ridenow` và chờ đến khi mọi pod `Ready`. Một pod kẹt ở `0/1` thường do sai database endpoint, secret, security group hoặc Redis endpoint. Chạy `kubectl describe pod` trên pod đó để xem lý do.


