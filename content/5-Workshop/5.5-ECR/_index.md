---
title : "ECR: Image Registry"
date : 2024-01-01
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

This page creates the seven private ECR repositories that hold the RideNow service images. Each repository scans images on push and has a lifecycle policy that keeps at most 50 images and expires untagged images after 14 days.

## Set variables

Run this block in your Bash session, then compare the output with the expected result:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
IMAGE_TAG=v1.0.0

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "IMAGE_TAG=$IMAGE_TAG"
```

Expected result (the account ID is masked):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
IMAGE_TAG=v1.0.0
```

If a line starts with `STOP`, fix that problem and run the block again. These variables are defined on 5.2, so you can skip this block if your shell session is still the one from 5.2.

## What it creates

| Template logical ID | AWS resource type | Repository name |
|---|---|---|
| `UserServiceRepo` | `AWS::ECR::Repository` | `ridenow/user-service` |
| `DriverServiceRepo` | `AWS::ECR::Repository` | `ridenow/driver-service` |
| `TripServiceRepo` | `AWS::ECR::Repository` | `ridenow/trip-service` |
| `PaymentServiceRepo` | `AWS::ECR::Repository` | `ridenow/payment-service` |
| `MatchingServiceRepo` | `AWS::ECR::Repository` | `ridenow/matching-service` |
| `NotificationWorkerRepo` | `AWS::ECR::Repository` | `ridenow/notification-worker` |
| `RealtimeGatewayRepo` | `AWS::ECR::Repository` | `ridenow/realtime-gateway` |

Every repository uses the same settings:

| Setting | Value |
|---|---|
| Visibility | Private |
| Image tag mutability | Immutable |
| Encryption | KMS, AWS managed key `aws/ecr` |
| Scan on push | Enabled |
| Lifecycle rule 1 | Expire untagged images 14 days after they are pushed |
| Lifecycle rule 2 | Expire images when the repository holds more than 50 |

## Depends on

None. You can create the repositories at any time before 5.9, where you build and push the images.

## Create the resources

Open the AWS console in the Asia Pacific (Singapore) `ap-southeast-1` region. Each repository below has its own complete block of steps. Do them one after another.

If the create page has no **Scan on push** option, turn it on after you create the repositories: go to **ECR -> Private registry -> Settings -> Scanning configuration**, choose **Basic scanning**, and enable **Scan on push** for the filter `ridenow/*`.

### 1. `ridenow/user-service` (`UserServiceRepo`)

**Create the repository**

1. Go to **ECR -> Private registry -> Repositories -> Create repository**.
2. Set **Visibility settings** to `Private`.
3. In **Repository name**, type `ridenow/user-service` (type the slash as part of the name).
4. Set **Image tag mutability** to `Immutable`.
5. Tick **Customize encryption settings (KMS)** under **Encryption settings**.
6. Set **Encryption key** to `AWS managed key (aws/ecr)`.
7. Under **Image scan settings**, set **Scan on push** to `Enabled`.
8. Choose **Create repository**.

**Lifecycle rule 1**

9. Open `ridenow/user-service`, then choose **Lifecycle Policy** in the left menu.
10. Choose **Create rule**.
11. Set **Rule priority** to `1`.
12. Set **Rule description** to `Expire untagged images`.
13. Set **Image status** to `Untagged`.
14. Under **Match criteria**, set **Since image pushed** with **Days** `14`.
15. Set **Rule action** to `expire`.
16. Choose **Save**.

**Lifecycle rule 2**

17. Choose **Create rule** again.
18. Set **Rule priority** to `2`.
19. Set **Rule description** to `Keep only the most recent tagged images`.
20. Set **Image status** to `Any`.
21. Under **Match criteria**, set **Image count more than** with **Count** `50`.
22. Set **Rule action** to `expire`.
23. Choose **Save**.

### 2. `ridenow/driver-service` (`DriverServiceRepo`)

**Create the repository**

1. Go to **ECR -> Private registry -> Repositories -> Create repository**.
2. Set **Visibility settings** to `Private`.
3. In **Repository name**, type `ridenow/driver-service` (type the slash as part of the name).
4. Set **Image tag mutability** to `Immutable`.
5. Tick **Customize encryption settings (KMS)** under **Encryption settings**.
6. Set **Encryption key** to `AWS managed key (aws/ecr)`.
7. Under **Image scan settings**, set **Scan on push** to `Enabled`.
8. Choose **Create repository**.

**Lifecycle rule 1**

9. Open `ridenow/driver-service`, then choose **Lifecycle Policy** in the left menu.
10. Choose **Create rule**.
11. Set **Rule priority** to `1`.
12. Set **Rule description** to `Expire untagged images`.
13. Set **Image status** to `Untagged`.
14. Under **Match criteria**, set **Since image pushed** with **Days** `14`.
15. Set **Rule action** to `expire`.
16. Choose **Save**.

**Lifecycle rule 2**

17. Choose **Create rule** again.
18. Set **Rule priority** to `2`.
19. Set **Rule description** to `Keep only the most recent tagged images`.
20. Set **Image status** to `Any`.
21. Under **Match criteria**, set **Image count more than** with **Count** `50`.
22. Set **Rule action** to `expire`.
23. Choose **Save**.

### 3. `ridenow/trip-service` (`TripServiceRepo`)

**Create the repository**

1. Go to **ECR -> Private registry -> Repositories -> Create repository**.
2. Set **Visibility settings** to `Private`.
3. In **Repository name**, type `ridenow/trip-service` (type the slash as part of the name).
4. Set **Image tag mutability** to `Immutable`.
5. Tick **Customize encryption settings (KMS)** under **Encryption settings**.
6. Set **Encryption key** to `AWS managed key (aws/ecr)`.
7. Under **Image scan settings**, set **Scan on push** to `Enabled`.
8. Choose **Create repository**.

**Lifecycle rule 1**

9. Open `ridenow/trip-service`, then choose **Lifecycle Policy** in the left menu.
10. Choose **Create rule**.
11. Set **Rule priority** to `1`.
12. Set **Rule description** to `Expire untagged images`.
13. Set **Image status** to `Untagged`.
14. Under **Match criteria**, set **Since image pushed** with **Days** `14`.
15. Set **Rule action** to `expire`.
16. Choose **Save**.

**Lifecycle rule 2**

17. Choose **Create rule** again.
18. Set **Rule priority** to `2`.
19. Set **Rule description** to `Keep only the most recent tagged images`.
20. Set **Image status** to `Any`.
21. Under **Match criteria**, set **Image count more than** with **Count** `50`.
22. Set **Rule action** to `expire`.
23. Choose **Save**.

### 4. `ridenow/payment-service` (`PaymentServiceRepo`)

**Create the repository**

1. Go to **ECR -> Private registry -> Repositories -> Create repository**.
2. Set **Visibility settings** to `Private`.
3. In **Repository name**, type `ridenow/payment-service` (type the slash as part of the name).
4. Set **Image tag mutability** to `Immutable`.
5. Tick **Customize encryption settings (KMS)** under **Encryption settings**.
6. Set **Encryption key** to `AWS managed key (aws/ecr)`.
7. Under **Image scan settings**, set **Scan on push** to `Enabled`.
8. Choose **Create repository**.

**Lifecycle rule 1**

9. Open `ridenow/payment-service`, then choose **Lifecycle Policy** in the left menu.
10. Choose **Create rule**.
11. Set **Rule priority** to `1`.
12. Set **Rule description** to `Expire untagged images`.
13. Set **Image status** to `Untagged`.
14. Under **Match criteria**, set **Since image pushed** with **Days** `14`.
15. Set **Rule action** to `expire`.
16. Choose **Save**.

**Lifecycle rule 2**

17. Choose **Create rule** again.
18. Set **Rule priority** to `2`.
19. Set **Rule description** to `Keep only the most recent tagged images`.
20. Set **Image status** to `Any`.
21. Under **Match criteria**, set **Image count more than** with **Count** `50`.
22. Set **Rule action** to `expire`.
23. Choose **Save**.

### 5. `ridenow/matching-service` (`MatchingServiceRepo`)

**Create the repository**

1. Go to **ECR -> Private registry -> Repositories -> Create repository**.
2. Set **Visibility settings** to `Private`.
3. In **Repository name**, type `ridenow/matching-service` (type the slash as part of the name).
4. Set **Image tag mutability** to `Immutable`.
5. Tick **Customize encryption settings (KMS)** under **Encryption settings**.
6. Set **Encryption key** to `AWS managed key (aws/ecr)`.
7. Under **Image scan settings**, set **Scan on push** to `Enabled`.
8. Choose **Create repository**.

**Lifecycle rule 1**

9. Open `ridenow/matching-service`, then choose **Lifecycle Policy** in the left menu.
10. Choose **Create rule**.
11. Set **Rule priority** to `1`.
12. Set **Rule description** to `Expire untagged images`.
13. Set **Image status** to `Untagged`.
14. Under **Match criteria**, set **Since image pushed** with **Days** `14`.
15. Set **Rule action** to `expire`.
16. Choose **Save**.

**Lifecycle rule 2**

17. Choose **Create rule** again.
18. Set **Rule priority** to `2`.
19. Set **Rule description** to `Keep only the most recent tagged images`.
20. Set **Image status** to `Any`.
21. Under **Match criteria**, set **Image count more than** with **Count** `50`.
22. Set **Rule action** to `expire`.
23. Choose **Save**.

### 6. `ridenow/notification-worker` (`NotificationWorkerRepo`)

**Create the repository**

1. Go to **ECR -> Private registry -> Repositories -> Create repository**.
2. Set **Visibility settings** to `Private`.
3. In **Repository name**, type `ridenow/notification-worker` (type the slash as part of the name).
4. Set **Image tag mutability** to `Immutable`.
5. Tick **Customize encryption settings (KMS)** under **Encryption settings**.
6. Set **Encryption key** to `AWS managed key (aws/ecr)`.
7. Under **Image scan settings**, set **Scan on push** to `Enabled`.
8. Choose **Create repository**.

**Lifecycle rule 1**

9. Open `ridenow/notification-worker`, then choose **Lifecycle Policy** in the left menu.
10. Choose **Create rule**.
11. Set **Rule priority** to `1`.
12. Set **Rule description** to `Expire untagged images`.
13. Set **Image status** to `Untagged`.
14. Under **Match criteria**, set **Since image pushed** with **Days** `14`.
15. Set **Rule action** to `expire`.
16. Choose **Save**.

**Lifecycle rule 2**

17. Choose **Create rule** again.
18. Set **Rule priority** to `2`.
19. Set **Rule description** to `Keep only the most recent tagged images`.
20. Set **Image status** to `Any`.
21. Under **Match criteria**, set **Image count more than** with **Count** `50`.
22. Set **Rule action** to `expire`.
23. Choose **Save**.

### 7. `ridenow/realtime-gateway` (`RealtimeGatewayRepo`)

**Create the repository**

1. Go to **ECR -> Private registry -> Repositories -> Create repository**.
2. Set **Visibility settings** to `Private`.
3. In **Repository name**, type `ridenow/realtime-gateway` (type the slash as part of the name).
4. Set **Image tag mutability** to `Immutable`.
5. Tick **Customize encryption settings (KMS)** under **Encryption settings**.
6. Set **Encryption key** to `AWS managed key (aws/ecr)`.
7. Under **Image scan settings**, set **Scan on push** to `Enabled`.
8. Choose **Create repository**.

**Lifecycle rule 1**

9. Open `ridenow/realtime-gateway`, then choose **Lifecycle Policy** in the left menu.
10. Choose **Create rule**.
11. Set **Rule priority** to `1`.
12. Set **Rule description** to `Expire untagged images`.
13. Set **Image status** to `Untagged`.
14. Under **Match criteria**, set **Since image pushed** with **Days** `14`.
15. Set **Rule action** to `expire`.
16. Choose **Save**.

**Lifecycle rule 2**

17. Choose **Create rule** again.
18. Set **Rule priority** to `2`.
19. Set **Rule description** to `Keep only the most recent tagged images`.
20. Set **Image status** to `Any`.
21. Under **Match criteria**, set **Image count more than** with **Count** `50`.
22. Set **Rule action** to `expire`.
23. Choose **Save**.

## Values to record

You do not need to copy anything by hand. Every later page builds the registry address from `ACCOUNT_ID` and `AWS_REGION`. The URI of each repository has the form `<account-id>.dkr.ecr.<region>.amazonaws.com/<repository name>`. To list the URIs, go to **ECR -> Private registry -> Repositories** and read the **URI** column, or run:

```bash
aws ecr describe-repositories \
  --query "repositories[?starts_with(repositoryName,'ridenow/')].[repositoryName,repositoryUri]" \
  --output table
```

| Repository | Used for |
|---|---|
| `ridenow/user-service` | Image push below and the user-service workload in 5.9 |
| `ridenow/driver-service` | Image push below and the driver-service workload in 5.9 |
| `ridenow/trip-service` | Image push below and the trip-service workload in 5.9 |
| `ridenow/payment-service` | Image push below and the payment-service workload in 5.9 |
| `ridenow/matching-service` | Image push below and the matching-service workload in 5.9 |
| `ridenow/notification-worker` | Image push below and the notification-worker workload in 5.9 |
| `ridenow/realtime-gateway` | Image push below and the realtime-gateway workload in 5.9 |

## Post-deploy steps

Build the seven service images and push them to the repositories. Run this block from the root of the RideNow repository, with Docker running and the variables from the first block set:

```bash
REGISTRY="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
aws ecr get-login-password --region "$AWS_REGION" \
  | docker login --username AWS --password-stdin "$REGISTRY"
for SVC in user-service driver-service trip-service payment-service \
           matching-service notification-worker realtime-gateway; do
  docker buildx build --platform linux/amd64 \
    -t "$REGISTRY/ridenow/$SVC:$IMAGE_TAG" \
    --push "./$SVC"
done
```

Expected result: `Login Succeeded`, then seven successful pushes. Tags are immutable, so to push again, set a new `IMAGE_TAG` first.

## Verify

Check the scan, encryption, and tag settings of all seven repositories:

```bash
aws ecr describe-repositories \
  --query "repositories[?starts_with(repositoryName,'ridenow/')].[repositoryName,imageScanningConfiguration.scanOnPush,encryptionConfiguration.encryptionType,imageTagMutability]" \
  --output table
```

Expected result: seven rows, each with `True`, `KMS`, and `IMMUTABLE`. If a row differs, delete that repository while it is empty and create it again, because the encryption setting cannot change after creation.

Check the lifecycle rules of all seven repositories:

```bash
for REPO in user-service driver-service trip-service payment-service \
            matching-service notification-worker realtime-gateway; do
  echo "ridenow/$REPO"
  aws ecr get-lifecycle-policy --repository-name "ridenow/$REPO" \
    --query lifecyclePolicyText --output text \
    | jq -c '.rules[] | {rulePriority, countNumber: .selection.countNumber}'
done
```

Expected result: for each repository, rule priority `1` with `countNumber` `14` and rule priority `2` with `countNumber` `50`:

```text
ridenow/user-service
{"rulePriority":1,"countNumber":14}
{"rulePriority":2,"countNumber":50}
```

The next page is 5.6 Data.
