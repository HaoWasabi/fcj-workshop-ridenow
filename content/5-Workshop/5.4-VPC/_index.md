---
title : "VPC: Networking"
date : 2024-01-01
weight : 4
chapter : false
pre : " <b> 5.4. </b> "
---

This page creates the network of the `prod` environment: one VPC with two public and two private subnets, an internet gateway, two NAT gateways, and two security groups. Every later page (data, messaging, EKS) places its resources in this network.

## Set variables

Run the first block in a new shell. It repeats the shared variables from 5.2:

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
AWS_REGION=$(aws configure get region)
export AWS_DEFAULT_REGION="$AWS_REGION"
ENV=prod
CLUSTER_NAME=ridenow-prod

[ -n "$ACCOUNT_ID" ] || echo "STOP: ACCOUNT_ID is empty. Check your AWS credentials."
[ -n "$AWS_REGION" ] || echo "STOP: AWS_REGION is empty. Set a default region for your AWS CLI profile."
```

The five network variables look up resources that you create on this page. Run the second block after you finish Create the resources, not before. Each lookup finds its resource by its `Name` tag.

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=${CLUSTER_NAME}-vpc" --query 'Vpcs[0].VpcId' --output text)
[ -n "$VPC_ID" ] && [ "$VPC_ID" != "None" ] || echo "STOP: VPC_ID is empty. Create the VPC first."

PUBLIC_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-public-*" --query 'Subnets[].SubnetId' --output text)
[ -n "$PUBLIC_SUBNET_IDS" ] || echo "STOP: PUBLIC_SUBNET_IDS is empty. Create the public subnets first."

PRIVATE_SUBNET_IDS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-private-*" --query 'Subnets[].SubnetId' --output text)
[ -n "$PRIVATE_SUBNET_IDS" ] || echo "STOP: PRIVATE_SUBNET_IDS is empty. Create the private subnets first."

CLUSTER_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-cluster-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$CLUSTER_SG_ID" ] && [ "$CLUSTER_SG_ID" != "None" ] || echo "STOP: CLUSTER_SG_ID is empty. Create the cluster security group first."

DB_SG_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=${CLUSTER_NAME}-db-sg" --query 'SecurityGroups[0].GroupId' --output text)
[ -n "$DB_SG_ID" ] && [ "$DB_SG_ID" != "None" ] || echo "STOP: DB_SG_ID is empty. Create the database security group first."

echo "ACCOUNT_ID=$ACCOUNT_ID"
echo "AWS_REGION=$AWS_REGION"
echo "ENV=$ENV"
echo "CLUSTER_NAME=$CLUSTER_NAME"
echo "VPC_ID=$VPC_ID"
echo "PUBLIC_SUBNET_IDS=$PUBLIC_SUBNET_IDS"
echo "PRIVATE_SUBNET_IDS=$PRIVATE_SUBNET_IDS"
echo "CLUSTER_SG_ID=$CLUSTER_SG_ID"
echo "DB_SG_ID=$DB_SG_ID"
```

Expected result (the IDs are masked, and the two subnet lists show two IDs each):

```text
ACCOUNT_ID=123456789012
AWS_REGION=ap-southeast-1
ENV=prod
CLUSTER_NAME=ridenow-prod
VPC_ID=vpc-0abc...
PUBLIC_SUBNET_IDS=subnet-0aaa... subnet-0bbb...
PRIVATE_SUBNET_IDS=subnet-0ccc... subnet-0ddd...
CLUSTER_SG_ID=sg-0eee...
DB_SG_ID=sg-0fff...
```

If a line starts with `STOP`, fix that problem and run the block again.

## What it creates

| Resource | AWS resource type | Name |
|---|---|---|
| VPC | VPC | `ridenow-prod-vpc` |
| Internet gateway | Internet gateway | `ridenow-prod-igw` |
| Gateway attachment | Internet gateway attachment | `ridenow-prod-igw` attached to `ridenow-prod-vpc` |
| Public subnet 1 | Subnet | `ridenow-prod-public-1` |
| Public subnet 2 | Subnet | `ridenow-prod-public-2` |
| Private subnet 1 | Subnet | `ridenow-prod-private-1` |
| Private subnet 2 | Subnet | `ridenow-prod-private-2` |
| Public route table | Route table | `ridenow-prod-public-rt` |
| Public default route | Route | `0.0.0.0/0` to `ridenow-prod-igw` |
| Public subnet associations | Route table association | `ridenow-prod-public-1` and `ridenow-prod-public-2` with `ridenow-prod-public-rt` |
| Elastic IP 1 | Elastic IP | `ridenow-prod-nat-eip-1` |
| NAT gateway 1 | NAT gateway | `ridenow-prod-nat-1` |
| Elastic IP 2 | Elastic IP | `ridenow-prod-nat-eip-2` |
| NAT gateway 2 | NAT gateway | `ridenow-prod-nat-2` |
| Private route table 1 | Route table | `ridenow-prod-private-rt-1` |
| Private default route 1 | Route | `0.0.0.0/0` to `ridenow-prod-nat-1` |
| Private route table 2 | Route table | `ridenow-prod-private-rt-2` |
| Private default route 2 | Route | `0.0.0.0/0` to `ridenow-prod-nat-2` |
| Cluster security group | Security group | `ridenow-prod-cluster-sg` |
| Database security group | Security group | `ridenow-prod-db-sg` |

The VPC uses the CIDR block `10.30.0.0/16`. The two public subnets and the two private subnets are spread over two Availability Zones. There are three route tables in total: one public table shared by both public subnets, and one private table for each private subnet.

## Depends on

Nothing. This is the first network page, and you can build it right after 5.3.

## Create the resources

Open the AWS console in the Asia Pacific (Singapore) `ap-southeast-1` region. Use the first two Availability Zones, `ap-southeast-1a` and `ap-southeast-1b`.

### 1. VPC (`Vpc`)

1. Go to **VPC -> Your VPCs -> Create VPC**.
2. Set **Resources to create** to `VPC only`.
3. Set **Name tag - optional** to `ridenow-prod-vpc`.
4. Set **IPv4 CIDR block** to `IPv4 CIDR manual input`.
5. Set **IPv4 CIDR** to `10.30.0.0/16`.
6. Set **IPv6 CIDR block** to `No IPv6 CIDR block`.
7. Set **Tenancy** to `Default`.
8. Choose **Create VPC**.

Enable DNS:

1. Select `ridenow-prod-vpc`.
2. Go to **Actions -> Edit VPC settings**.
3. Under **DNS settings**, tick **Enable DNS resolution**.
4. Tick **Enable DNS hostnames**.
5. Choose **Save**.

### 2. Internet gateway (`InternetGateway`)

1. Go to **VPC -> Internet gateways -> Create internet gateway**.
2. Set **Name tag** to `ridenow-prod-igw`.
3. Choose **Create internet gateway**.
4. Go to **Actions -> Attach to VPC**.
5. Set **Available VPCs** to `ridenow-prod-vpc`.
6. Choose **Attach internet gateway**.

### 3. Subnets (`Subnets`)

Start the form:

1. Go to **VPC -> Subnets -> Create subnet**.
2. Set **VPC ID** to `ridenow-prod-vpc`.

Subnet 1:

1. Set **Subnet name** to `ridenow-prod-public-1`.
2. Set **Availability Zone** to `ap-southeast-1a`.
3. Set **IPv4 subnet CIDR block** to `10.30.0.0/20`.
4. Under **Tags - optional**, choose **Add new tag**. Set **Key** to `kubernetes.io/role/elb` and **Value** to `1`. The `Name` tag is already filled from **Subnet name**.

Subnet 2 (choose **Add new subnet** first):

1. Set **Subnet name** to `ridenow-prod-public-2`.
2. Set **Availability Zone** to `ap-southeast-1b`.
3. Set **IPv4 subnet CIDR block** to `10.30.16.0/20`.
4. Under **Tags - optional**, choose **Add new tag**. Set **Key** to `kubernetes.io/role/elb` and **Value** to `1`.

Subnet 3 (choose **Add new subnet** first):

1. Set **Subnet name** to `ridenow-prod-private-1`.
2. Set **Availability Zone** to `ap-southeast-1a`.
3. Set **IPv4 subnet CIDR block** to `10.30.48.0/20`.
4. Under **Tags - optional**, choose **Add new tag**. Set **Key** to `kubernetes.io/role/internal-elb` and **Value** to `1`.

Subnet 4 (choose **Add new subnet** first):

1. Set **Subnet name** to `ridenow-prod-private-2`.
2. Set **Availability Zone** to `ap-southeast-1b`.
3. Set **IPv4 subnet CIDR block** to `10.30.64.0/20`.
4. Under **Tags - optional**, choose **Add new tag**. Set **Key** to `kubernetes.io/role/internal-elb` and **Value** to `1`.

Finish:

1. Choose **Create subnet**.

Check the tags. Open each subnet's **Tags** tab. The two public subnets must show `kubernetes.io/role/elb` = `1`, and the two private subnets must show `kubernetes.io/role/internal-elb` = `1`. The AWS Load Balancer Controller finds the subnets for the load balancer by these tags. If a tag is missing, choose **Manage tags** and add it.

Enable public IPs on `ridenow-prod-public-1`:

1. Go to **VPC -> Subnets** and select `ridenow-prod-public-1`.
2. Go to **Actions -> Edit subnet settings**.
3. Tick **Enable auto-assign public IPv4 address**.
4. Choose **Save**.

Enable public IPs on `ridenow-prod-public-2`:

1. Go to **VPC -> Subnets** and select `ridenow-prod-public-2`.
2. Go to **Actions -> Edit subnet settings**.
3. Tick **Enable auto-assign public IPv4 address**.
4. Choose **Save**.

Leave auto-assign off on both private subnets.

### 4. Public route table (`PublicRouteTable`)

Create the table:

1. Go to **VPC -> Route tables -> Create route table**.
2. Set **Name** to `ridenow-prod-public-rt`.
3. Set **VPC** to `ridenow-prod-vpc`.
4. Choose **Create route table**.

Add the default route:

1. Open the **Routes** tab and choose **Edit routes -> Add route**.
2. Set **Destination** to `0.0.0.0/0`.
3. Set **Target** to **Internet Gateway** and pick `ridenow-prod-igw`.
4. Choose **Save changes**.

Associate the subnets:

1. Open the **Subnet associations** tab and choose **Edit subnet associations**.
2. Tick `ridenow-prod-public-1`.
3. Tick `ridenow-prod-public-2`.
4. Choose **Save associations**.

### 5. Elastic IPs and NAT gateways (`NatGateway`)

Allocate the first Elastic IP:

1. Go to **VPC -> Elastic IPs -> Allocate Elastic IP address**.
2. Set **Public IPv4 address pool** to `Amazon's pool of IPv4 addresses`.
3. Set **Network border group** to `ap-southeast-1`.
4. Under **Tags - optional**, choose **Add new tag**. Set **Key** to `Name` and **Value** to `ridenow-prod-nat-eip-1`.
5. Choose **Allocate**.

Allocate the second Elastic IP:

1. Choose **Allocate Elastic IP address** again.
2. Use the same settings as the first address, but set the `Name` tag **Value** to `ridenow-prod-nat-eip-2`.
3. Choose **Allocate**.

Create NAT gateway 1:

1. Go to **VPC -> NAT gateways -> Create NAT gateway**.
2. Set **Name** to `ridenow-prod-nat-1`.
3. Set **Subnet** to `ridenow-prod-public-1`.
4. Set **Connectivity type** to `Public`.
5. Set **Elastic IP allocation ID** to the first Elastic IP you allocated.
6. Choose **Create NAT gateway**.

Create NAT gateway 2:

1. Choose **Create NAT gateway** again.
2. Set **Name** to `ridenow-prod-nat-2`.
3. Set **Subnet** to `ridenow-prod-public-2`.
4. Set **Connectivity type** to `Public`.
5. Set **Elastic IP allocation ID** to the second Elastic IP (the one not yet associated).
6. Choose **Create NAT gateway**.

Wait:

1. Wait until both gateways show **State** `Available` (3 to 5 minutes).

### 6. Private route tables (`PrivateRouteTable`)

Route table 1:

1. Go to **VPC -> Route tables -> Create route table**.
2. Set **Name** to `ridenow-prod-private-rt-1`.
3. Set **VPC** to `ridenow-prod-vpc`.
4. Choose **Create route table**.
5. Open the **Routes** tab and choose **Edit routes -> Add route**.
6. Set **Destination** to `0.0.0.0/0`.
7. Set **Target** to **NAT Gateway** and pick `ridenow-prod-nat-1`.
8. Choose **Save changes**.
9. Open the **Subnet associations** tab and choose **Edit subnet associations**.
10. Tick `ridenow-prod-private-1`.
11. Choose **Save associations**.

Route table 2:

1. Go to **VPC -> Route tables -> Create route table**.
2. Set **Name** to `ridenow-prod-private-rt-2`.
3. Set **VPC** to `ridenow-prod-vpc`.
4. Choose **Create route table**.
5. Open the **Routes** tab and choose **Edit routes -> Add route**.
6. Set **Destination** to `0.0.0.0/0`.
7. Set **Target** to **NAT Gateway** and pick `ridenow-prod-nat-2`.
8. Choose **Save changes**.
9. Open the **Subnet associations** tab and choose **Edit subnet associations**.
10. Tick `ridenow-prod-private-2`.
11. Choose **Save associations**.

### 7. Cluster security group (`ClusterSecurityGroup`)

Create the group:

1. Go to **VPC -> Security groups -> Create security group**.
2. Set **Security group name** to `ridenow-prod-cluster-sg`.
3. Set **Description** to `ridenow-prod EKS / application pods`.
4. Set **VPC** to `ridenow-prod-vpc`.
5. Leave **Inbound rules** empty.
6. Keep the default **Outbound rules** row: **Type** `All traffic`, **Protocol** `All`, **Port range** `All`, **Destination** `Anywhere-IPv4` `0.0.0.0/0`.
7. Set that outbound row **Description** to `Allow all egress (NAT to internet, RDS, etc.)`.
8. Under **Tags - optional**, choose **Add new tag**. Set **Key** to `Name` and **Value** to `ridenow-prod-cluster-sg`. The variable lookups find the group by this tag, not by the group name.
9. Choose **Create security group**.

Add the self-referencing rule:

1. Open `ridenow-prod-cluster-sg`.
2. Open the **Inbound rules** tab and choose **Edit inbound rules -> Add rule**.
3. Set **Type** to `All traffic`.
4. Set **Source** to **Custom** and pick `ridenow-prod-cluster-sg` (the group itself).
5. Set **Description** to `Allow all traffic between members of this SG`.
6. Choose **Save rules**.

### 8. Database security group (`DbSecurityGroup`)

1. Go to **VPC -> Security groups -> Create security group**.
2. Set **Security group name** to `ridenow-prod-db-sg`.
3. Set **Description** to `ridenow-prod RDS PostgreSQL (private)`.
4. Set **VPC** to `ridenow-prod-vpc`.
5. Under **Inbound rules**, choose **Add rule**.
6. Set **Type** to `PostgreSQL` (Protocol `TCP`, Port range `5432`).
7. Set **Source** to **Custom** and pick `ridenow-prod-cluster-sg`.
8. Set the rule **Description** to `Postgres 5432 from the RideNow cluster SG only`.
9. Keep the default **Outbound rules** row: **Type** `All traffic`, **Protocol** `All`, **Port range** `All`, **Destination** `Anywhere-IPv4` `0.0.0.0/0`.
10. Set that outbound row **Description** to `Allow all egress`.
11. Under **Tags - optional**, choose **Add new tag**. Set **Key** to `Name` and **Value** to `ridenow-prod-db-sg`.
12. Choose **Create security group**.

Add no CIDR inbound rule.

## Values to record

Run the second block of Set variables now. The five variables below are looked up for you, so you copy nothing by hand. Later pages restate the same lookups at their top.

| Variable | What it holds | Used by |
|---|---|---|
| `VPC_ID` | ID of `ridenow-prod-vpc` | 5.7 (cache security group), 5.8 (load balancer controller) |
| `PUBLIC_SUBNET_IDS` | IDs of `ridenow-prod-public-1` and `ridenow-prod-public-2` | The load balancer created by the Ingress, which finds the public subnets by tag |
| `PRIVATE_SUBNET_IDS` | IDs of `ridenow-prod-private-1` and `ridenow-prod-private-2` | 5.6 (database subnet group), 5.7 (cache subnet group), 5.8 (cluster and Fargate profiles) |
| `CLUSTER_SG_ID` | ID of `ridenow-prod-cluster-sg` | 5.7 (cache ingress source), 5.8 (cluster security group) |
| `DB_SG_ID` | ID of `ridenow-prod-db-sg` | 5.6 (the six database instances), 5.9 (ingress rule from the EKS cluster security group) |

The VPC CIDR `10.30.0.0/16` is a fixed value that you entered in step 1 and need only for reference.

## Post-deploy steps

No post-deploy step is needed for this page.

## Verify

Run these commands in the same shell:

```bash
aws ec2 describe-vpcs --vpc-ids "$VPC_ID" --query 'Vpcs[0].[CidrBlock,State]' --output text
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'length(Subnets)'
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" --query 'length(NatGateways)'
aws ec2 describe-security-groups --group-ids "$DB_SG_ID" --query 'SecurityGroups[0].IpPermissions[].[IpProtocol,FromPort,ToPort]' --output text
```

Expected result:

```text
10.30.0.0/16	available
4
2
tcp	5432	5432
```

If a value is wrong, recheck the matching step in the VPC console.
