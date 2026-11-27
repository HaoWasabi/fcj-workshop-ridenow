---
title : "Frontend"
date : 2024-01-01
weight : 10
chapter : false
pre : " <b> 5.10. </b> "
---

This page publishes the web app. It creates a private S3 bucket, a CloudFront distribution that serves the site and sends `/api/*` and `/live*` to the load balancer, and an optional IAM role that GitHub Actions uses to publish. Do this page last, after the `ridenow` Ingress has an ALB address (5.8) and the seven workloads run (5.9). The distribution needs that ALB address as its API origin.

## Set variables

Run this block in a new shell. It repeats the shared variables from 5.2 and looks up the ALB address of the `ridenow` Ingress from 5.8:

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

Expected result (the account ID is masked):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
BUCKET=ridenow-prod-frontend-123456789012-ap-southeast-1
ALB_DNS_NAME=k8s-ridenow-ridenow-0abc1234.elb.amazonaws.com
```

If a line starts with `STOP`, fix that problem and run the block again. Do not go on while `ALB_DNS_NAME` is empty, because the distribution cannot reach the API without it.

Two values on this page cannot be looked up by a command, so they are Placeholders:

- `<GITHUB_REPO>` is the owner and name of your GitHub repository, in the form `owner/name`. You need it only if you create the optional deploy role. Copy it from the address bar of your repository on GitHub (the part after `github.com/`), and write it in your notes file.
- `<APP_ORIGIN>` is the address of the site. It does not exist until you create the distribution. You copy it from the values you record at the end of Create the resources, then paste it into 5.9.

## What it creates

| Template logical ID | AWS resource type | Resolved name |
|---|---|---|
| `SiteBucket` | S3 bucket | `ridenow-prod-frontend-<account id>-ap-southeast-1` (your account ID is part of the name) |
| `SiteBucketPolicy` | S3 bucket policy | on the bucket above |
| `OriginAccessControl` | CloudFront origin access control | `ridenow-prod-frontend-oac` |
| `SpaRewriteFunction` | CloudFront function | `ridenow-prod-spa-rewrite` |
| `Distribution` | CloudFront distribution | comment `RideNow prod frontend` |
| `DeployRole` | IAM role (optional) | `ridenow-prod-frontend-deploy` |

The bucket stays private. Only the distribution can read it, through the origin access control. The function rewrites paths without a file extension to `/index.html`, so the single-page app router works on reload. The distribution has two origins: the bucket (`site`) and the ALB (`backend`). It sends `/api/*` and `/live*` (server-sent events from `realtime-gateway`) to the ALB and everything else to the bucket.

Create the deploy role only if you want GitHub Actions to publish the site. If you publish from your own machine, skip steps 6 and 7.

## Depends on

5.1 to 5.9. You need these before you start:

| Resource | Name | Created in | Used for |
|---|---|---|---|
| Ingress with an ALB address | `ridenow` in namespace `ridenow` | 5.8 and 5.9 | Origin `backend` of the distribution |
| Running workloads | the seven services of the `ridenow` namespace | 5.9 | Answer `/api/*` and `/live*` |

## Create the resources

Open the AWS console in the Asia Pacific (Singapore) `ap-southeast-1` region. Create the resources in this order: bucket, origin access control, function, distribution, bucket policy, then the optional OIDC provider and role.

### 1. S3 bucket (`SiteBucket`)

**Create the bucket**

1. Go to **S3 -> Buckets -> Create bucket**.
2. Set **Bucket type** to `General purpose`.
3. Set **Bucket name** to the value of `BUCKET` (run `echo "$BUCKET"` to print it).
4. Set **AWS Region** to `Asia Pacific (Singapore) ap-southeast-1`.
5. Under **Object Ownership**, pick `ACLs disabled (recommended)`.
6. Under **Block Public Access settings for this bucket**, tick **Block all public access** (all four boxes on).
7. Set **Bucket Versioning** to `Enable`.
8. Under **Default encryption**, pick `Server-side encryption with Amazon S3 managed keys (SSE-S3)`. Leave **Bucket Key** as is.
9. Choose **Create bucket**.

**Add the lifecycle rule**

1. Open the bucket, then the **Management** tab -> **Lifecycle rules** -> **Create lifecycle rule**.
2. Set **Lifecycle rule name** to `expire-old-versions`.
3. Under **Choose a rule scope**, pick **Apply to all objects in the bucket**.
4. Tick **I acknowledge that this rule will apply to all objects in the bucket**.
5. Under **Lifecycle rule actions**, tick **Permanently delete noncurrent versions of objects**.
6. Set **Days after objects become noncurrent** to `30`.
7. Choose **Create rule**.

Check: the Permissions tab shows Block all public access `On`, the Properties tab shows Versioning `Enabled` and SSE-S3, and the Management tab shows `expire-old-versions` `Enabled`.

### 2. Origin access control (`OriginAccessControl`)

1. Go to **CloudFront -> Security -> Origin access -> Create control setting**.
2. Set **Name** to `ridenow-prod-frontend-oac`.
3. Leave **Description** empty.
4. Set **Origin type** to `S3`.
5. Set **Signing behavior** to `Sign requests (recommended)`.
6. Choose **Create**.

### 3. SPA rewrite function (`SpaRewriteFunction`)

1. Go to **CloudFront -> Functions -> Create function**.
2. Set **Name** to `ridenow-prod-spa-rewrite`.
3. Set **Description** to `Rewrite extensionless paths to /index.html for the SPA router`.
4. Set **Runtime** to `cloudfront-js-2.0`.
5. Choose **Create function**.
6. On the **Build** tab, in **Function code**, select all, delete, and paste the code below.
7. Choose **Save changes**.
8. Open the **Publish** tab and choose **Publish function**. The distribution can use the function only after you publish it.

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

**Create the distribution**

1. Go to **CloudFront -> Distributions -> Create distribution**.
2. Under **Choose a plan**, pick `Pay as you go`.
3. Set **Distribution name** to `ridenow-prod-frontend-cdn`.
4. Set **Distribution type** to `Single website or app`, then choose **Next**.
5. Set **Origin type** to `Amazon S3`.
6. For **S3 origin**, choose **Browse S3** and pick your frontend bucket (the bucket, not the website endpoint).
7. Under **Origin settings**, keep `Use recommended origin settings`. The wizard creates an origin access control for you. Choose **Next**.
8. Under **Enable security**, do not enable AWS WAF protections. Choose **Next**.
9. On the review page, choose **Create distribution**.

The wizard creates its own origin access control and may rewrite the bucket policy. You fix both below. The distribution takes 5 to 10 minutes to deploy, and you can edit it while it deploys.

**Fix the S3 origin**

1. Open the distribution, then the **Origins** tab. Select the S3 origin and choose **Edit**.
2. Set **Name** to `site`.
3. Set **Origin access** to `Origin access control settings (recommended)`.
4. Set **Origin access control** to `ridenow-prod-frontend-oac` (from step 2).
5. Choose **Save changes**.

**Add the ALB origin**

1. On the **Origins** tab, choose **Create origin**.
2. Set **Origin domain** to the value of `ALB_DNS_NAME` (the plain DNS name, no `http://`, no path).
3. Set **Protocol** to `HTTP only`.
4. Set **HTTP port** to `80`.
5. Set **Name** to `backend`.
6. Set **Origin access** to `Public`.
7. Under **Additional settings**, set **Response timeout** to `60`.
8. Set **Keep-alive timeout** to `60`.
9. Choose **Create origin**.

**Edit the default behavior**

1. Open the **Behaviors** tab, select **Default (*)**, and choose **Edit**.
2. Set **Origin and origin groups** to `site`.
3. Set **Compress objects automatically** to `Yes`.
4. Set **Viewer protocol policy** to `Redirect HTTP to HTTPS`.
5. Set **Allowed HTTP methods** to `GET, HEAD, OPTIONS`.
6. Set **Cache HTTP methods** to `GET, HEAD` only (leave OPTIONS unticked).
7. Under **Cache key and origin requests**, set **Cache policy** to `CachingOptimized` (Managed).
8. Leave **Origin request policy** empty.
9. Under **Function associations**, set **Viewer request** to **Function type** `CloudFront Functions` and **Function ARN / name** `ridenow-prod-spa-rewrite`.
10. Leave all other function associations empty.
11. Choose **Save changes**.

**Create the `/api/*` behavior**

1. On the **Behaviors** tab, choose **Create behavior**.
2. Set **Path pattern** to `/api/*`.
3. Set **Origin and origin groups** to `backend`.
4. Set **Compress objects automatically** to `No`.
5. Set **Viewer protocol policy** to `Redirect HTTP to HTTPS`.
6. Set **Allowed HTTP methods** to `GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE`.
7. Set **Cache HTTP methods** to `GET, HEAD` only.
8. Set **Cache policy** to `CachingDisabled` (Managed).
9. Set **Origin request policy** to `AllViewerExceptHostHeader` (Managed).
10. Leave **Function associations** empty.
11. Choose **Create behavior**.

**Create the `/live*` behavior** (server-sent events from `realtime-gateway`)

1. On the **Behaviors** tab, choose **Create behavior**.
2. Set **Path pattern** to `/live*`.
3. Set **Origin and origin groups** to `backend`.
4. Set **Compress objects automatically** to `No`.
5. Set **Viewer protocol policy** to `Redirect HTTP to HTTPS`.
6. Set **Allowed HTTP methods** to `GET, HEAD, OPTIONS`.
7. Set **Cache HTTP methods** to `GET, HEAD` only.
8. Set **Cache policy** to `CachingDisabled`.
9. Set **Origin request policy** to `AllViewerExceptHostHeader`.
10. Leave **Function associations** empty.
11. Choose **Create behavior**.
12. Check the order in the **Behaviors** list: `/api/*` first, `/live*` second, **Default (*)** last.

**Edit the general settings**

1. Open the **General** tab, then **Settings -> Edit**.
2. Set **Description** to `RideNow prod frontend`.
3. Set **Price class** to `Use only North America and Europe`.
4. Set **Supported HTTP versions** to `HTTP/2` and `HTTP/3` (tick both).
5. Set **IPv6** to `On`.
6. Set **Default root object** to `index.html`.
7. Set **Standard logging** to `Off`.
8. Leave **AWS WAF web ACL** empty.
9. Set **Distribution state** to `Enabled`.
10. Choose **Save changes**.
11. Leave the **Error pages** tab empty, because the function handles the single-page app routes.

Now look up the distribution ID and domain name. The next steps use them:

```bash
CLOUDFRONT_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='RideNow prod frontend'].Id | [0]" --output text)
CLOUDFRONT_DOMAIN=$(aws cloudfront list-distributions --query "DistributionList.Items[?Comment=='RideNow prod frontend'].DomainName | [0]" --output text)
[ -n "$CLOUDFRONT_ID" ] && [ "$CLOUDFRONT_ID" != "None" ] || echo "STOP: CLOUDFRONT_ID is empty. Create the distribution first."
[ -n "$CLOUDFRONT_DOMAIN" ] && [ "$CLOUDFRONT_DOMAIN" != "None" ] || echo "STOP: CLOUDFRONT_DOMAIN is empty. Create the distribution first."
echo "CLOUDFRONT_ID=$CLOUDFRONT_ID"
echo "CLOUDFRONT_DOMAIN=$CLOUDFRONT_DOMAIN"
```

Expected result (the values are masked):

```text
CLOUDFRONT_ID=E1ABC....
CLOUDFRONT_DOMAIN=d1abc....cloudfront.net
```

### 5. Bucket policy (`SiteBucketPolicy`)

Print the policy. It names your bucket, account, and distribution, so you do not type them by hand:

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

1. Go to **S3 -> Buckets** and open your frontend bucket.
2. Open the **Permissions** tab, then **Bucket policy -> Edit**.
3. Select all, delete, and paste the JSON that the block printed.
4. Choose **Save changes**.

### 6. GitHub OIDC provider (optional, only for GitHub publishing)

Check whether the provider already exists in the account:

```bash
aws iam list-open-id-connect-providers --query "OpenIDConnectProviderList[?contains(Arn, 'token.actions.githubusercontent.com')].Arn" --output text
```

Expected result: an ARN if the provider exists, and an empty line if not. If you get an ARN, skip to step 7. Otherwise create it:

1. Go to **IAM -> Identity providers -> Add provider**.
2. Set **Provider type** to `OpenID Connect`.
3. Set **Provider URL** to `https://token.actions.githubusercontent.com`. Choose **Get thumbprint** if shown.
4. Set **Audience** to `sts.amazonaws.com`.
5. Choose **Add provider**.

### 7. Deploy role (`DeployRole`, optional)

Copy `<GITHUB_REPO>` from your GitHub repository address (see Set variables) before you run the next block. The trust policy allows only jobs of that repository that run in the GitHub Environment `prod`. Print both policies:

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

The first document is the trust policy and the second is the permissions policy.

**Create the role**

1. Go to **IAM -> Roles -> Create role**.
2. Set **Trusted entity type** to `Custom trust policy`.
3. Paste the first document (the trust policy). Choose **Next**.
4. On **Add permissions**, select nothing. Choose **Next**.
5. Set **Role name** to `ridenow-prod-frontend-deploy`.
6. Set **Description** to `Assumed by GitHub Actions (OIDC) to publish the RideNow frontend.`
7. Choose **Create role**.

**Add the inline policy**

1. Open the role, then the **Permissions** tab -> **Add permissions -> Create inline policy** -> **JSON**.
2. Paste the second document (the permissions policy). Choose **Next**.
3. Set **Policy name** to `publish-frontend`.
4. Choose **Create policy**.

**Set the session length**

1. On the role **Summary**, choose **Maximum session duration -> Edit**.
2. Set it to `1 hour`.
3. Choose **Save changes**.

## Values to record

Run this block now. It reads the deploy role ARN if the role exists, and prints every value of this page:

```bash
DEPLOY_ROLE_ARN=$(aws iam get-role --role-name "ridenow-${ENV}-frontend-deploy" --query Role.Arn --output text 2>/dev/null)
[ -n "$DEPLOY_ROLE_ARN" ] && [ "$DEPLOY_ROLE_ARN" != "None" ] || echo "NOTE: DEPLOY_ROLE_ARN is empty. This is fine if you skipped the optional deploy role."

echo "CLOUDFRONT_ID=$CLOUDFRONT_ID"
echo "CLOUDFRONT_DOMAIN=$CLOUDFRONT_DOMAIN"
echo "DEPLOY_ROLE_ARN=$DEPLOY_ROLE_ARN"
echo "APP_ORIGIN to copy into 5.9: https://$CLOUDFRONT_DOMAIN"
```

Expected result (the values are masked):

```text
CLOUDFRONT_ID=E1ABC....
CLOUDFRONT_DOMAIN=d1abc....cloudfront.net
DEPLOY_ROLE_ARN=arn:aws:iam::123456789012:role/ridenow-prod-frontend-deploy
APP_ORIGIN to copy into 5.9: https://d1abc....cloudfront.net
```

| Value | Type | What it holds | Used by |
|---|---|---|---|
| `CLOUDFRONT_ID` | Auto_Variable | ID of the distribution | Steps 5 and 7, Post-deploy steps, Verify |
| `CLOUDFRONT_DOMAIN` | Auto_Variable | Domain name of the distribution | Post-deploy steps, Verify |
| `DEPLOY_ROLE_ARN` | Auto_Variable | ARN of `ridenow-prod-frontend-deploy` (empty if you skipped the role) | GitHub variable `AWS_DEPLOY_ROLE_ARN` |
| `<APP_ORIGIN>` | Placeholder | `https://` followed by the value of `CLOUDFRONT_DOMAIN` | 5.9, where the services accept requests only from this origin |
| `<GITHUB_REPO>` | Placeholder | Your repository, as `owner/name` | Step 7 and the GitHub setup below |

Why these two are Placeholders: `<APP_ORIGIN>` is the one value that 5.9 needs from this page, and 5.9 comes earlier in the order, so you carry it back by hand. Copy it from the last line of the block above into your notes file, and paste it into `<APP_ORIGIN>` on 5.9. `<GITHUB_REPO>` is your own choice of repository, so no command can look it up. Copy it from the repository address on GitHub.

## Post-deploy steps

Run these steps from the root of the RideNow repository, in the shell where you ran the blocks above. You need `pnpm` for the build. The GitHub setup also needs the `gh` CLI signed in to your account.

### 1. First publish

Build with the CloudFront domain as the API base, upload `assets/*` with a long cache time and the rest with `no-cache`, then invalidate `/index.html`:

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

Expected result: the build finishes, the two syncs list the uploaded files, and the last line is an invalidation ID with `InProgress`.

### 2. GitHub setup (only if you created the deploy role)

Copy `<GITHUB_REPO>` from your repository address (see Set variables) before you run this block. It creates the GitHub Environment `prod` with you as required reviewer, then sets the two variables that the publish workflow reads:

```bash
REVIEWER_ID=$(gh api user --jq .id)
jq -n --argjson id "$REVIEWER_ID" '{reviewers: [{type: "User", id: $id}]}' \
  | gh api -X PUT "repos/<GITHUB_REPO>/environments/${ENV}" --input - --jq '.name'
gh variable set AWS_DEPLOY_ROLE_ARN --env "$ENV" --repo "<GITHUB_REPO>" --body "$DEPLOY_ROLE_ARN"
gh variable set AWS_REGION --env "$ENV" --repo "<GITHUB_REPO>" --body "$AWS_REGION"
gh variable list --env "$ENV" --repo "<GITHUB_REPO>"
```

Expected result: `prod`, then a list with `AWS_DEPLOY_ROLE_ARN` and `AWS_REGION`.

Required reviewers need a public repository or a paid GitHub plan. The workflow job must run in the GitHub Environment `prod`, because the trust policy of the role accepts only that subject.

### 3. Roll back

The bucket is versioned, and a publish never deletes files from `assets/`, so a rollback only has to restore `index.html`.

List the versions of `index.html`, newest first:

```bash
aws s3api list-object-versions --bucket "$BUCKET" --prefix index.html \
  --query "Versions[?Key=='index.html'].[VersionId,LastModified,IsLatest]" --output table
```

The second row is the previous release. Copy that version over the current object, and invalidate the cache so viewers get it:

```bash
PREV=$(aws s3api list-object-versions --bucket "$BUCKET" --prefix index.html \
  --query "Versions[?Key=='index.html'].VersionId | [1]" --output text)
aws s3api copy-object --bucket "$BUCKET" --key index.html \
  --copy-source "$BUCKET/index.html?versionId=$PREV" \
  --query 'VersionId' --output text
aws cloudfront create-invalidation --distribution-id "$CLOUDFRONT_ID" --paths "/index.html" \
  --query 'Invalidation.[Id,Status]' --output text
```

Expected result: a new version ID, then an invalidation ID with `InProgress`. Noncurrent versions expire after 30 days. To go back further, check out the older commit and repeat the first publish.

## Verify

Run these commands in the same shell:

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://${CLOUDFRONT_DOMAIN}/"
curl -s -o /dev/null -w "HTTP %{http_code}\n" "https://${CLOUDFRONT_DOMAIN}/api/drivers"
aws cloudfront get-distribution-config --id "$CLOUDFRONT_ID" --query 'DistributionConfig.CacheBehaviors.Items[].PathPattern' --output json
aws s3api get-public-access-block --bucket "$BUCKET" --query 'PublicAccessBlockConfiguration' --output json
```

Expected result: `HTTP 200` for the site, `HTTP 200` for `/api/drivers`, the list `["/api/*", "/live*"]`, and all four public access settings `true`.

- `HTTP 403` on the site means the bucket is empty (run the first publish) or the bucket policy names another distribution (print and apply step 5 again).
- `HTTP 000` means the distribution is still deploying. Wait a few minutes.
- `502` or `504` on `/api/drivers` means CloudFront cannot reach the ALB. Open the distribution, then **Origins**, edit `backend`, and set **Origin domain** to the current `ALB_DNS_NAME`.
- A page that shows an old version means the cache still holds the previous `index.html`. Wait for the invalidation to finish.

The next page is 5.11 Cleanup.
