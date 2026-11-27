---
title : "Dọn dẹp tài nguyên"
date : 2024-01-01
weight : 11
chapter : false
pre : " <b> 5.11. </b> "
---

Trang này xóa mọi thứ bạn đã dựng ở 5.3 đến 5.10 để chi phí tính theo giờ dừng lại. Làm từ trên xuống: xóa trước những gì chạy bên trong cluster, rồi xóa các tài nguyên AWS theo thứ tự ngược với thứ tự bạn đã tạo.

## Set variables

Chạy khối lệnh này trong một shell mới. Nó đặt account, Region, tên cluster và VPC ID mà các bước kiểm tra trên trang này dùng:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "VPC_ID=$VPC_ID"
```

Kết quả mong đợi (account ID và VPC ID đã được che):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
```

Nếu một dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh. Nếu `VPC_ID` in ra `None`, mạng đã bị xóa và bạn có thể bỏ qua bước kiểm tra load balancer.

## What gets deleted

| Nhóm | Những gì bị xóa | Tạo ở |
|---|---|---|
| Trong cluster | Ingress, bảy workload, pod security group policy, External Secrets, các Helm release, các namespace | 5.8, 5.9 |
| Tài nguyên AWS | Frontend, IAM role, security group rule, EKS, KMS key, Valkey, SNS, SQS, RDS, ECR, mạng, IAM | 5.3 đến 5.10 |

## Delete the in-cluster resources

> **Cảnh báo.** Hãy làm bước này trước khi xóa bất cứ thứ gì trong console. Ingress tạo ra một Application Load Balancer. Load balancer này giữ các network interface trong subnet của VPC, nên không thể xóa VPC khi nó còn tồn tại. Xóa Ingress là cách để xóa load balancer.

Chạy các lệnh sau từ thư mục gốc của repository:

```bash
# Run from the repository root.
kubectl delete -f iac/k8s/30-ingress.yaml --ignore-not-found
for F in 26-realtime-gateway 25-notification-worker 24-matching-service \
         22-trip-service 21-driver-service 20-user-service 23-payment-service; do
  kubectl delete -f "iac/k8s/$F.yaml" --ignore-not-found
done
kubectl delete securitygrouppolicy --all -n ridenow --ignore-not-found
kubectl delete -f iac/k8s/10-external-secrets.yaml --ignore-not-found
kubectl delete namespace ridenow aws-observability --ignore-not-found
helm uninstall external-secrets -n external-secrets
helm uninstall aws-load-balancer-controller -n kube-system
kubectl delete namespace external-secrets --ignore-not-found
```

Chờ đến khi load balancer biến mất. Lệnh kiểm tra này liệt kê các load balancer trong VPC:

```bash
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?VpcId=='${VPC_ID}'].[LoadBalancerName,State.Code]" --output text
```

Kết quả mong đợi: không có output. Nếu load balancer vẫn còn trong danh sách, hãy chờ một phút rồi chạy lại. Trong EC2 console, mở **Load Balancers** và xác nhận không còn load balancer nào có tag `elbv2.k8s.aws/cluster = ridenow-prod`.

Nếu bạn dựng các layer bằng CloudFormation template thay vì console, lệnh này in ra kế hoạch xóa stack (chạy thử, không xóa gì). Thêm `--apply` để xóa stack, rồi chạy bước kiểm tra ở cuối trang này:

```bash
bash iac/scripts/teardown.sh "$ENV"
```

## Delete the AWS resources in the console

> **Cảnh báo.** Hãy xóa load balancer do Ingress tạo ra trước (phần trên). Nếu nó còn tồn tại, bước 21 sẽ lỗi vì phụ thuộc và VPC cùng các NAT gateway vẫn còn.

Xóa theo thứ tự trong bảng, từ trên xuống dưới. Một tài nguyên chỉ được xóa sau khi mọi thứ đang dùng nó đã bị xóa. Thứ tự này ngược với thứ tự bạn đã tạo, từ 5.10 xuống 5.3.

| Bước | Trang | Service console | Xóa | Ghi chú |
|------|-------|-----------------|-----|---------|
| 1 | 5.10 | CloudFront > **Distributions** | Distribution `RideNow prod frontend` | **Disable** distribution và chờ đến khi **Last modified** hiện ngày, rồi **Delete**. Sau đó mở **Functions** và xóa function SPA rewrite `ridenow-prod-spa-rewrite`. Sau đó mở **Origin access** và xóa `ridenow-prod-frontend-oac`. |
| 2 | 5.10 | S3 > **Buckets** | `ridenow-prod-frontend-<account id>-<region>` | **Empty** bucket (mọi version và delete marker), rồi **Delete**. |
| 3 | 5.10 | IAM > **Roles** | `ridenow-prod-frontend-deploy` (chỉ khi bạn đã tạo) | Xóa inline policy của role trước, rồi xóa role. |
| 4 | 5.8 | EC2 > **Security Groups** | Sáu nhóm `ridenow-prod-<service>-service-sg` | Trước tiên gỡ sáu inbound rule tương ứng khỏi `ridenow-prod-db-sg`, rồi xóa các nhóm. |
| 5 | 5.9 | EC2 > **Security Groups** | Hai inbound rule cho phép pod truy cập | Trên `ridenow-prod-db-sg` (cổng 5432) và `ridenow-prod-redis-sg` (cổng 6379), xóa các rule có source là security group của EKS cluster. |
| 6 | 5.9 | IAM > **Roles** | Sáu IRSA role: `ridenow-prod-alb-controller`, `ridenow-prod-external-secrets`, `ridenow-prod-trip-service`, `ridenow-prod-payment-service`, `ridenow-prod-matching-service`, `ridenow-prod-notification-worker` | Gỡ hoặc xóa các policy của chúng trước. |
| 7 | 5.9 | IAM > **Identity providers** | OIDC provider của cluster | Xóa sau bước 6. |
| 8 | 5.9 | IAM > **Policies** | `AWSLoadBalancerControllerIAMPolicy` | Xóa sau khi policy đã được gỡ khỏi role ở bước 6. |
| 9 | 5.8 | EKS > **Clusters** > `ridenow-prod` > **Compute** | Ba Fargate profile | Xóa từng profile một, theo thứ tự: `ridenow-prod-observability`, `ridenow-prod-app`, `ridenow-prod-system`. Chờ mỗi lần xóa hoàn tất. |
| 10 | 5.8 | EKS > **Clusters** | `ridenow-prod` | Xóa sau khi các profile đã bị xóa. |
| 11 | 5.8 | KMS > **Customer managed keys** | Key có alias `ridenow-prod-eks-secrets` | **Schedule key deletion** (tối thiểu 7 ngày). |
| 12 | 5.7 | ElastiCache > **Valkey caches** | `ridenow-prod-redis` | Xóa. Sau đó mở **Subnet groups** và xóa subnet group của Valkey. Rồi trong EC2 xóa `ridenow-prod-redis-sg`. |
| 13 | 5.7 | SNS > **Subscriptions**, rồi **Topics** | Ba subscription, rồi `ridenow-prod-events` | |
| 14 | 5.7 | SQS > **Queues** | `ridenow-prod-matching`, `ridenow-prod-payment`, `ridenow-prod-notifications` và ba DLQ của chúng | Xóa từng queue trước DLQ của nó. |
| 15 | 5.6 | RDS > **Databases** | Sáu instance `ridenow-prod-<service>-db` | Tạo final snapshot khi được hỏi. Deletion protection đang tắt. Secret master-user trong Secrets Manager bị xóa cùng instance. |
| 16 | 5.6 | RDS > **Subnet groups** | DB subnet group | Xóa sau khi cả sáu instance đã bị xóa. |
| 17 | 5.6 | KMS > **Customer managed keys** | Key có alias `ridenow-prod-rds` | **Schedule key deletion**. |
| 18 | 5.5 | ECR > **Repositories** | Bảy repository `ridenow/<service>` | Xóa cùng với các image. |
| 19 | 5.4 | EC2 > **Security Groups** | `ridenow-prod-db-sg`, rồi `ridenow-prod-cluster-sg` | Xóa sau khi các tài nguyên RDS, Valkey và EKS dùng chúng đã bị xóa. |
| 20 | 5.4 | VPC > **NAT gateways** | Cả hai NAT gateway | Chờ đến khi hiện **Deleted**. Sau đó mở VPC > **Elastic IPs** và **Release** cả hai địa chỉ. |
| 21 | 5.4 | VPC > **Route tables**, **Subnets**, **Internet gateways**, **Your VPCs** | Các route table, bốn subnet, internet gateway (**Detach**, rồi xóa), rồi `ridenow-prod-vpc` | Xóa theo đúng thứ tự này. |
| 22 | 5.3 | IAM > **Roles**, **Policies** | Hai EKS role, rồi app-access policy và bốn messaging policy | Xóa cuối cùng: không còn gì dùng các tài nguyên của 5.3. |
| 23 | 5.8 | CloudWatch > **Log groups** | `/aws/eks/ridenow/fargate` và log group của cluster | Tùy chọn. |

Nếu console từ chối xóa và báo lỗi phụ thuộc, thông báo sẽ nêu tên tài nguyên đang dùng nó. Hãy xóa tài nguyên đó trước.

## Check that everything is gone

Chạy khối lệnh này. Nó liệt kê mọi tài nguyên RideNow còn tồn tại:

```bash
left() {
  local OUT
  OUT=$(aws "$@" --output text 2>/dev/null)
  [ -n "$OUT" ] && [ "$OUT" != "None" ] && echo "  $OUT"
}

echo "Still present:"
left eks list-clusters --query "clusters[?@=='${CLUSTER_NAME}']"
left rds describe-db-instances --query "DBInstances[?starts_with(DBInstanceIdentifier, '${CLUSTER_NAME}-')].DBInstanceIdentifier"
left elasticache describe-replication-groups --query "ReplicationGroups[?starts_with(ReplicationGroupId, '${CLUSTER_NAME}-')].ReplicationGroupId"
left sns list-topics --query "Topics[?contains(TopicArn, ':${CLUSTER_NAME}-')].TopicArn"
left sqs list-queues --queue-name-prefix "${CLUSTER_NAME}-" --query 'QueueUrls'
left elbv2 describe-load-balancers --query "LoadBalancers[?VpcId=='${VPC_ID}'].LoadBalancerArn"
left ec2 describe-nat-gateways --filter "Name=tag:Name,Values=${CLUSTER_NAME}-*" "Name=state,Values=pending,available" --query 'NatGateways[].NatGatewayId'
left ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[].VpcId'
left ecr describe-repositories --query "repositories[?starts_with(repositoryName, 'ridenow/')].repositoryName"
left iam list-roles --query "Roles[?starts_with(RoleName, '${CLUSTER_NAME}-')].RoleName"
left cloudfront list-distributions --query "DistributionList.Items[?Comment=='RideNow ${ENV} frontend'].Id"
left s3api list-buckets --query "Buckets[?starts_with(Name, '${CLUSTER_NAME}-frontend-')].Name"
echo "Done."
```

Kết quả mong đợi:

```text
Still present:
Done.
```

Danh sách rỗng nghĩa là không còn gì để trả phí. Các KMS key đang chờ xóa không được liệt kê và không tốn phí sau khi bạn đặt lịch xóa.