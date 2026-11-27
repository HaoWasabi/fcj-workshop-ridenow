---
title : "Frontend"
date : 2024-01-01
weight : 10
chapter : false
pre : " <b> 5.10. </b> "
---

Trang này xuất bản ứng dụng web. Trang tạo một bucket S3 riêng tư, một distribution CloudFront phục vụ website và chuyển `/api/*` cùng `/live*` tới load balancer, và một IAM role tùy chọn để GitHub Actions dùng khi xuất bản. Hãy làm trang này sau cùng, sau khi Ingress `ridenow` đã có địa chỉ ALB (5.8) và bảy workload đang chạy (5.9). Distribution cần địa chỉ ALB đó làm API origin.

## Set variables

Chạy khối lệnh này trong một shell mới. Khối lệnh lặp lại các biến dùng chung từ 5.2 và tra địa chỉ ALB của Ingress `ridenow` từ 5.8:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

BUCKET="ridenow-${ENV}-frontend-${ACCOUNT_ID}-${AWS_REGION}"

ALB_DNS_NAME=$(kubectl get ingress ridenow -n ridenow -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
[ -n "$ALB_DNS_NAME" ] || echo "STOP: ALB_DNS_NAME is empty. The ridenow Ingress has no ALB address yet. Finish 5.8 first."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "BUCKET=$BUCKET"
echo "ALB_DNS_NAME=$ALB_DNS_NAME"
```

Kết quả mong đợi (ID tài khoản đã được che):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
BUCKET=ridenow-prod-frontend-123456789012-ap-southeast-1
ALB_DNS_NAME=k8s-ridenow-ridenow-0abc1234.elb.amazonaws.com
```

Nếu một dòng bắt đầu bằng `STOP`, hãy sửa lỗi đó rồi chạy lại khối lệnh. Đừng làm tiếp khi `ALB_DNS_NAME` còn trống, vì distribution không thể gọi tới API nếu thiếu giá trị này.

Hai giá trị trên trang này không lệnh nào tra được, nên chúng là Placeholder:

- `<GITHUB_REPO>` là chủ sở hữu và tên repository GitHub của bạn, theo dạng `owner/name`. Bạn chỉ cần giá trị này nếu tạo deploy role tùy chọn. Hãy sao chép giá trị từ thanh địa chỉ của repository trên GitHub (phần sau `github.com/`) và ghi vào tệp ghi chú.
- `<APP_ORIGIN>` là địa chỉ của website. Giá trị này chưa tồn tại cho đến khi bạn tạo distribution. Bạn sao chép nó từ các giá trị cần ghi lại ở cuối phần Create the resources, rồi dán vào 5.9.

## What it creates

| Template logical ID | AWS resource type | Resolved name |
|---|---|---|
| `SiteBucket` | S3 bucket | `ridenow-prod-frontend-<account id>-ap-southeast-1` (ID tài khoản của bạn nằm trong tên) |
| `SiteBucketPolicy` | S3 bucket policy | trên bucket ở trên |
| `OriginAccessControl` | CloudFront origin access control | `ridenow-prod-frontend-oac` |
| `SpaRewriteFunction` | CloudFront function | `ridenow-prod-spa-rewrite` |
| `Distribution` | CloudFront distribution | comment `RideNow prod frontend` |
| `DeployRole` | IAM role (tùy chọn) | `ridenow-prod-frontend-deploy` |

Bucket luôn riêng tư. Chỉ distribution đọc được bucket, thông qua origin access control. Function viết lại các đường dẫn không có phần mở rộng tệp thành `/index.html`, để router của single-page app hoạt động khi tải lại trang. Distribution có hai origin: bucket (`site`) và ALB (`backend`). Distribution chuyển `/api/*` và `/live*` (server-sent events từ `realtime-gateway`) tới ALB, và mọi đường dẫn còn lại tới bucket.

Chỉ tạo deploy role nếu bạn muốn GitHub Actions xuất bản website. Nếu bạn xuất bản từ máy của mình, hãy bỏ qua bước 6 và 7.

## Depends on

5.1 đến 5.9. Bạn cần những thứ sau trước khi bắt đầu:

| Resource | Name | Created in | Used for |
|---|---|---|---|
| Ingress có địa chỉ ALB | `ridenow` trong namespace `ridenow` | 5.8 và 5.9 | Origin `backend` của distribution |
| Workload đang chạy | bảy service của namespace `ridenow` | 5.9 | Trả lời `/api/*` và `/live*` |

## Create the resources

Mở AWS console ở Region Asia Pacific (Singapore) `ap-southeast-1`. Tạo các tài nguyên theo thứ tự: bucket, origin access control, function, distribution, bucket policy, rồi OIDC provider và role tùy chọn.

### 1. S3 bucket (`SiteBucket`)

**Tạo bucket**

1. Vào **S3 -> Buckets -> Create bucket**.
2. Đặt **Bucket type** là `General purpose`.
3. Đặt **Bucket name** là giá trị của `BUCKET` (chạy `echo "$BUCKET"` để in ra).
4. Đặt **AWS Region** là `Asia Pacific (Singapore) ap-southeast-1`.
5. Ở **Object Ownership**, chọn `ACLs disabled (recommended)`.
6. Ở **Block Public Access settings for this bucket**, đánh dấu **Block all public access** (bật cả bốn ô).
7. Đặt **Bucket Versioning** là `Enable`.
8. Ở **Default encryption**, chọn `Server-side encryption with Amazon S3 managed keys (SSE-S3)`. Giữ nguyên **Bucket Key**.
9. Chọn **Create bucket**.

**Thêm lifecycle rule**

1. Mở bucket, chọn tab **Management** -> **Lifecycle rules** -> **Create lifecycle rule**.
2. Đặt **Lifecycle rule name** là `expire-old-versions`.
3. Ở **Choose a rule scope**, chọn **Apply to all objects in the bucket**.
4. Đánh dấu **I acknowledge that this rule will apply to all objects in the bucket**.
5. Ở **Lifecycle rule actions**, đánh dấu **Permanently delete noncurrent versions of objects**.
6. Đặt **Days after objects become noncurrent** là `30`.
7. Chọn **Create rule**.

Kiểm tra: tab Permissions hiện Block all public access là `On`, tab Properties hiện Versioning là `Enabled` và SSE-S3, tab Management hiện `expire-old-versions` là `Enabled`.

### 2. Origin access control (`OriginAccessControl`)

1. Vào **CloudFront -> Security -> Origin access -> Create control setting**.
2. Đặt **Name** là `ridenow-prod-frontend-oac`.
3. Để trống **Description**.
4. Đặt **Origin type** là `S3`.
5. Đặt **Signing behavior** là `Sign requests (recommended)`.
6. Chọn **Create**.

### 3. SPA rewrite function (`SpaRewriteFunction`)

1. Vào **CloudFront -> Functions -> Create function**.
2. Đặt **Name** là `ridenow-prod-spa-rewrite`.
3. Đặt **Description** là `Rewrite extensionless paths to /index.html for the SPA router`.
4. Đặt **Runtime** là `cloudfront-js-2.0`.
5. Chọn **Create function**.
6. Ở tab **Build**, trong **Function code**, chọn tất cả, xóa, rồi dán đoạn mã bên dưới.
7. Chọn **Save changes**.
8. Mở tab **Publish** và chọn **Publish function**. Distribution chỉ dùng được function sau khi bạn publish nó.

```javascript
function handler(event) {
  var request = event.request;
  var lastSegment = request.uri.split('/').pop();
  if (lastSegment.indexOf('.') === -1) {
    request.uri = '/index.html';
  }
  return request;
}
```

### 4. CloudFront distribution (`Distribution`)

**Tạo distribution**

1. Vào **CloudFront -> Distributions -> Create distribution**.
2. Ở **Choose a plan**, chọn `Pay as you go`.
3. Đặt **Distribution name** là `ridenow-prod-frontend-cdn`.
4. Đặt **Distribution type** là `Single website or app`, rồi chọn **Next**.
5. Đặt **Origin type** là `Amazon S3`.
6. Ở **S3 origin**, chọn **Browse S3** và chọn bucket frontend của bạn (chọn bucket, không chọn website endpoint).
7. Ở **Origin settings**, giữ `Use recommended origin settings`. Wizard tự tạo một origin access control cho bạn. Chọn **Next**.
8. Ở **Enable security**, không bật AWS WAF protections. Chọn **Next**.
9. Ở trang xem lại, chọn **Create distribution**.

Wizard tạo origin access control riêng và có thể ghi đè bucket policy. Bạn sửa cả hai ở dưới. Distribution cần 5 đến 10 phút để triển khai, và bạn có thể sửa nó trong lúc chờ.

**Sửa S3 origin**

1. Mở distribution, chọn tab **Origins**. Chọn S3 origin và chọn **Edit**.
2. Đặt **Name** là `site`.
3. Đặt **Origin access** là `Origin access control settings (recommended)`.
4. Đặt **Origin access control** là `ridenow-prod-frontend-oac` (từ bước 2).
5. Chọn **Save changes**.

**Thêm ALB origin**

1. Ở tab **Origins**, chọn **Create origin**.
2. Đặt **Origin domain** là giá trị của `ALB_DNS_NAME` (tên DNS thuần, không có `http://`, không có đường dẫn).
3. Đặt **Protocol** là `HTTP only`.
4. Đặt **HTTP port** là `80`.
5. Đặt **Name** là `backend`.
6. Đặt **Origin access** là `Public`.
7. Ở **Additional settings**, đặt **Response timeout** là `60`.
8. Đặt **Keep-alive timeout** là `60`.
9. Chọn **Create origin**.

**Sửa default behavior**

1. Mở tab **Behaviors**, chọn **Default (*)** và chọn **Edit**.
2. Đặt **Origin and origin groups** là `site`.
3. Đặt **Compress objects automatically** là `Yes`.
4. Đặt **Viewer protocol policy** là `Redirect HTTP to HTTPS`.
5. Đặt **Allowed HTTP methods** là `GET, HEAD, OPTIONS`.
6. Đặt **Cache HTTP methods** chỉ là `GET, HEAD` (không đánh dấu OPTIONS).
7. Ở **Cache key and origin requests**, đặt **Cache policy** là `CachingOptimized` (Managed).
8. Để trống **Origin request policy**.
9. Ở **Function associations**, đặt **Viewer request** với **Function type** là `CloudFront Functions` và **Function ARN / name** là `ridenow-prod-spa-rewrite`.
10. Để trống mọi function association còn lại.
11. Chọn **Save changes**.

**Tạo behavior `/api/*`**

1. Ở tab **Behaviors**, chọn **Create behavior**.
2. Đặt **Path pattern** là `/api/*`.
3. Đặt **Origin and origin groups** là `backend`.
4. Đặt **Compress objects automatically** là `No`.
5. Đặt **Viewer protocol policy** là `Redirect HTTP to HTTPS`.
6. Đặt **Allowed HTTP methods** là `GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE`.
7. Đặt **Cache HTTP methods** chỉ là `GET, HEAD`.
8. Đặt **Cache policy** là `CachingDisabled` (Managed).
9. Đặt **Origin request policy** là `AllViewerExceptHostHeader` (Managed).
10. Để trống **Function associations**.
11. Chọn **Create behavior**.

**Tạo behavior `/live*`** (server-sent events từ `realtime-gateway`)

1. Ở tab **Behaviors**, chọn **Create behavior**.
2. Đặt **Path pattern** là `/live*`.
3. Đặt **Origin and origin groups** là `backend`.
4. Đặt **Compress objects automatically** là `No`.
5. Đặt **Viewer protocol policy** là `Redirect HTTP to HTTPS`.
6. Đặt **Allowed HTTP methods** là `GET, HEAD, OPTIONS`.
7. Đặt **Cache HTTP methods** chỉ là `GET, HEAD`.
8. Đặt **Cache policy** là `CachingDisabled`.
9. Đặt **Origin request policy** là `AllViewerExceptHostHeader`.
10. Để trống **Function associations**.
11. Chọn **Create behavior**.
12. Kiểm tra thứ tự trong danh sách **Behaviors**: `/api/*` đầu tiên, `/live*` thứ hai, **Default (*)** cuối cùng.

**Sửa general settings**

1. Mở tab **General**, chọn **Settings -> Edit**.
2. Đặt **Description** là `RideNow prod frontend`.
3. Đặt **Price class** là `Use only North America and Europe`.
4. Đặt **Supported HTTP versions** là `HTTP/2` và `HTTP/3` (đánh dấu cả hai).
5. Đặt **IPv6** là `On`.
6. Đặt **Default root object** là `index.html`.
7. Đặt **Standard logging** là `Off`.
8. Để trống **AWS WAF web ACL**.
9. Đặt **Distribution state** là `Enabled`.
10. Chọn **Save changes**.
11. Để trống tab **Error pages**, vì function đã xử lý các route của single-page app.

Bây giờ hãy tra ID và domain name của distribution. Các bước sau dùng chúng:

```bash
CLOUDFRONT_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='RideNow prod frontend'].Id | [0]" --output text)
CLOUDFRONT_DOMAIN=$(aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='RideNow prod frontend'].DomainName | [0]" --output text)
[ -n "$CLOUDFRONT_ID" ] && [ "$CLOUDFRONT_ID" != "None" ] || echo "STOP: CLOUDFRONT_ID is empty. Create the distribution first."
[ -n "$CLOUDFRONT_DOMAIN" ] && [ "$CLOUDFRONT_DOMAIN" != "None" ] || echo "STOP: CLOUDFRONT_DOMAIN is empty. Create the distribution first."
echo "CLOUDFRONT_ID=$CLOUDFRONT_ID"
echo "CLOUDFRONT_DOMAIN=$CLOUDFRONT_DOMAIN"
```

Kết quả mong đợi (các giá trị đã được che):

```text
CLOUDFRONT_ID=E1ABC....
CLOUDFRONT_DOMAIN=d1abc....cloudfront.net
```

### 5. Bucket policy (`SiteBucketPolicy`)

In policy ra. Policy chứa bucket, tài khoản và distribution của bạn, nên bạn không phải gõ tay:

```bash
cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontRead",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${BUCKET}/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${CLOUDFRONT_ID}"
        }
      }
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::${BUCKET}",
        "arn:aws:s3:::${BUCKET}/*"
      ],
      "Condition": {
        "Bool": { "aws:SecureTransport": "false" }
      }
    }
  ]
}
EOF
```

1. Vào **S3 -> Buckets** và mở bucket frontend của bạn.
2. Mở tab **Permissions**, chọn **Bucket policy -> Edit**.
3. Chọn tất cả, xóa, rồi dán JSON mà khối lệnh vừa in.
4. Chọn **Save changes**.

### 6. GitHub OIDC provider (tùy chọn, chỉ khi xuất bản bằng GitHub)

Kiểm tra provider đã có trong tài khoản chưa:

```bash
aws iam list-open-id-connect-providers --query "OpenIDConnectProviderList[?contains(Arn, 'token.actions.githubusercontent.com')].Arn" --output text
```

Kết quả mong đợi: một ARN nếu provider đã tồn tại, hoặc một dòng trống nếu chưa. Nếu có ARN, hãy chuyển sang bước 7. Nếu không, hãy tạo provider:

1. Vào **IAM -> Identity providers -> Add provider**.
2. Đặt **Provider type** là `OpenID Connect`.
3. Đặt **Provider URL** là `https://token.actions.githubusercontent.com`. Chọn **Get thumbprint** nếu có hiện.
4. Đặt **Audience** là `sts.amazonaws.com`.
5. Chọn **Add provider**.

### 7. Deploy role (`DeployRole`, tùy chọn)

Hãy sao chép `<GITHUB_REPO>` từ địa chỉ repository GitHub của bạn (xem Set variables) trước khi chạy khối lệnh tiếp theo. Trust policy chỉ cho phép các job của repository đó chạy trong GitHub Environment `prod`. In cả hai policy:

```bash
cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_REPO>:environment:${ENV}"
        }
      }
    }
  ]
}
EOF
cat <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::${BUCKET}"
    },
    {
      "Sid": "WriteObjects",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::${BUCKET}/*"
    },
    {
      "Sid": "Invalidate",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::${ACCOUNT_ID}:distribution/${CLOUDFRONT_ID}"
    }
  ]
}
EOF
```

Tài liệu đầu tiên là trust policy, tài liệu thứ hai là permissions policy.

**Tạo role**

1. Vào **IAM -> Roles -> Create role**.
2. Đặt **Trusted entity type** là `Custom trust policy`.
3. Dán tài liệu đầu tiên (trust policy). Chọn **Next**.
4. Ở **Add permissions**, không chọn gì. Chọn **Next**.
5. Đặt **Role name** là `ridenow-prod-frontend-deploy`.
6. Đặt **Description** là `Assumed by GitHub Actions (OIDC) to publish the RideNow frontend.`
7. Chọn **Create role**.

**Thêm inline policy**

1. Mở role, chọn tab **Permissions** -> **Add permissions -> Create inline policy** -> **JSON**.
2. Dán tài liệu thứ hai (permissions policy). Chọn **Next**.
3. Đặt **Policy name** là `publish-frontend`.
4. Chọn **Create policy**.

**Đặt thời lượng phiên**

1. Ở **Summary** của role, chọn **Maximum session duration -> Edit**.
2. Đặt là `1 hour`.
3. Chọn **Save changes**.

## Values to record

Chạy khối lệnh này ngay bây giờ. Khối lệnh đọc ARN của deploy role nếu role tồn tại, và in mọi giá trị của trang này:

```bash
DEPLOY_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-frontend-deploy" --query Role.Arn --output text 2>/dev/null)
[ -n "$DEPLOY_ROLE_ARN" ] && [ "$DEPLOY_ROLE_ARN" != "None" ] || echo "NOTE: DEPLOY_ROLE_ARN is empty. This is fine if you skipped the optional deploy role."

echo "CLOUDFRONT_ID=$CLOUDFRONT_ID"
echo "CLOUDFRONT_DOMAIN=$CLOUDFRONT_DOMAIN"
echo "DEPLOY_ROLE_ARN=$DEPLOY_ROLE_ARN"
echo "APP_ORIGIN to copy into 5.9: https://$CLOUDFRONT_DOMAIN"
```

Kết quả mong đợi (các giá trị đã được che):

```text
CLOUDFRONT_ID=E1ABC....
CLOUDFRONT_DOMAIN=d1abc....cloudfront.net
DEPLOY_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-frontend-deploy
APP_ORIGIN to copy into 5.9: https://d1abc....cloudfront.net
```

| Value | Type | What it holds | Used by |
|---|---|---|---|
| `CLOUDFRONT_ID` | Auto_Variable | ID của distribution | Bước 5 và 7, Post-deploy steps, Verify |
| `CLOUDFRONT_DOMAIN` | Auto_Variable | Domain name của distribution | Post-deploy steps, Verify |
| `DEPLOY_ROLE_ARN` | Auto_Variable | ARN của `ridenow-prod-frontend-deploy` (trống nếu bạn bỏ qua role) | Biến GitHub `AWS_DEPLOY_ROLE_ARN` |
| `<APP_ORIGIN>` | Placeholder | `https://` theo sau là giá trị của `CLOUDFRONT_DOMAIN` | 5.9, nơi các service chỉ nhận request từ origin này |
| `<GITHUB_REPO>` | Placeholder | Repository của bạn, dạng `owner/name` | Bước 7 và phần thiết lập GitHub bên dưới |

Lý do hai giá trị này là Placeholder: `<APP_ORIGIN>` là giá trị duy nhất 5.9 cần từ trang này, mà 5.9 đứng trước trong thứ tự thực hiện, nên bạn mang nó quay lại bằng tay. Hãy sao chép giá trị từ dòng cuối của khối lệnh trên vào tệp ghi chú, rồi dán vào `<APP_ORIGIN>` ở 5.9. `<GITHUB_REPO>` là repository do bạn tự chọn, nên không lệnh nào tra được. Hãy sao chép nó từ địa chỉ repository trên GitHub.

## Post-deploy steps

Chạy các bước này từ thư mục gốc của repository RideNow, trong shell mà bạn đã chạy các khối lệnh ở trên. Bạn cần `pnpm` để build. Phần thiết lập GitHub còn cần CLI `gh` đã đăng nhập vào tài khoản của bạn.

### 1. First publish

Build với domain CloudFront làm API base, tải `assets/*` lên với thời gian cache dài và phần còn lại với `no-cache`, rồi invalidate `/index.html`:

```bash
( cd frontend && pnpm install --frozen-lockfile && VITE_API_BASE_URL="https://$CLOUDFRONT_DOMAIN" pnpm build )
aws s3 sync frontend/dist/ "s3://$BUCKET/" \
  --exclude "*" --include "assets/*" \
  --cache-control "public,max-age=31536000,immutable"
aws s3 sync frontend/dist/ "s3://$BUCKET/" \
  --exclude "assets/*" \
  --cache-control "no-cache" \
  --delete
aws cloudfront create-invalidation --distribution-id "$CLOUDFRONT_ID" --paths "/index.html" \
  --query 'Invalidation.[Id,Status]' --output text
```

Kết quả mong đợi: build hoàn tất, hai lệnh sync liệt kê các tệp đã tải lên, và dòng cuối là ID của invalidation cùng `InProgress`.

### 2. GitHub setup (chỉ khi bạn đã tạo deploy role)

Hãy sao chép `<GITHUB_REPO>` từ địa chỉ repository của bạn (xem Set variables) trước khi chạy khối lệnh này. Khối lệnh tạo GitHub Environment `prod` với bạn là người duyệt bắt buộc, rồi đặt hai biến mà workflow xuất bản đọc:

```bash
REVIEWER_ID=$(gh api user --jq .id)
jq -n --argjson id "$REVIEWER_ID" '{reviewers: [{type: "User", id: $id}]}' \
  | gh api -X PUT "repos/<GITHUB_REPO>/environments/${ENV}" --input - --jq '.name'
gh variable set AWS_DEPLOY_ROLE_ARN --env "$ENV" --repo "<GITHUB_REPO>" --body "$DEPLOY_ROLE_ARN"
gh variable set AWS_REGION --env "$ENV" --repo "<GITHUB_REPO>" --body "$AWS_REGION"
gh variable list --env "$ENV" --repo "<GITHUB_REPO>"
```

Kết quả mong đợi: `prod`, rồi một danh sách có `AWS_DEPLOY_ROLE_ARN` và `AWS_REGION`.

Người duyệt bắt buộc cần repository công khai hoặc gói GitHub trả phí. Job của workflow phải chạy trong GitHub Environment `prod`, vì trust policy của role chỉ chấp nhận subject đó.

### 3. Roll back

Bucket bật versioning và mỗi lần xuất bản không bao giờ xóa tệp trong `assets/`, nên rollback chỉ cần khôi phục `index.html`.

Liệt kê các phiên bản của `index.html`, mới nhất trước:

```bash
aws s3api list-object-versions --bucket "$BUCKET" --prefix index.html \
  --query "Versions[?Key=='index.html'].[VersionId,LastModified,IsLatest]" --output table
```

Hàng thứ hai là bản phát hành trước. Sao chép phiên bản đó đè lên object hiện tại, rồi invalidate cache để người xem nhận được bản mới:

```bash
PREV=$(aws s3api list-object-versions --bucket "$BUCKET" --prefix index.html \
  --query "Versions[?Key=='index.html'].VersionId | [1]" --output text)
aws s3api copy-object --bucket "$BUCKET" --key index.html \
  --copy-source "$BUCKET/index.html?versionId=$PREV" \
  --query 'VersionId' --output text
aws cloudfront create-invalidation --distribution-id "$CLOUDFRONT_ID" --paths "/index.html" \
  --query 'Invalidation.[Id,Status]' --output text
```

Kết quả mong đợi: một version ID mới, rồi ID của invalidation cùng `InProgress`. Các phiên bản noncurrent hết hạn sau 30 ngày. Để quay lại xa hơn, hãy checkout commit cũ hơn và lặp lại bước first publish.

## Verify

Chạy các lệnh sau trong cùng shell:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://${CLOUDFRONT_DOMAIN}/"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://${CLOUDFRONT_DOMAIN}/api/drivers"
aws cloudfront get-distribution-config --id "$CLOUDFRONT_ID" --query 'DistributionConfig.CacheBehaviors.Items[].PathPattern' --output json
aws s3api get-public-access-block --bucket "$BUCKET" --query 'PublicAccessBlockConfiguration' --output json
```

Kết quả mong đợi: `HTTP 200` cho website, `HTTP 200` cho `/api/drivers`, danh sách `["/api/*", "/live*"]`, và cả bốn thiết lập public access đều là `true`.

- `HTTP 403` ở website nghĩa là bucket đang trống (hãy chạy first publish) hoặc bucket policy trỏ tới distribution khác (in và áp dụng lại bước 5).
- `HTTP 000` nghĩa là distribution vẫn đang triển khai. Hãy chờ vài phút.
- `502` hoặc `504` ở `/api/drivers` nghĩa là CloudFront không tới được ALB. Mở distribution, chọn **Origins**, sửa `backend` và đặt **Origin domain** là `ALB_DNS_NAME` hiện tại.
- Trang hiện phiên bản cũ nghĩa là cache vẫn giữ `index.html` trước đó. Hãy chờ invalidation hoàn tất.

Trang tiếp theo là 5.11 Cleanup.
