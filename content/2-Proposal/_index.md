---
title: "Proposal"
date: 2026-09-09
weight: 2
chapter: false
pre: "<b>2. </b>"
---

# RideNow
## Deploying a ride-hailing platform on AWS
### 1. Project Overview

RideNow is a ride-hailing platform made of seven deployable services (user, driver, trip, payment, matching, notification, and a realtime gateway) and a single-page web frontend. The services keep one database each and exchange events through a message topic and queues.

This proposal covers the infrastructure only: how to run RideNow on Amazon Web Services in one production environment (`prod`) in the Singapore Region (`ap-southeast-1`). The environment is built by hand, one service console at a time, following the step-by-step guides in `guides/cloudformation/`. The read-only templates in `iac/stacks/` and the values in `iac/params/prod.json` define every name and setting.

- **Environment:** `prod`, region `ap-southeast-1`
- **Project duration:** [OWNER: fill in the schedule]
- **Team size and roles:** [OWNER: fill in team size and roles]

---

### 2. Problem Statement

#### Current State
RideNow already runs as a complete local system (`docker compose up --build`): six PostgreSQL containers, one Valkey container, LocalStack for SNS and SQS, and the seven services. Moving it to the cloud is a configuration swap at each boundary, because the service boundaries do not change. What is missing is a repeatable, secure cloud environment to run it in.

#### Objectives
- Run every service as a container on **Amazon EKS with AWS Fargate**, with no node groups to manage.
- Store one **Amazon RDS for PostgreSQL** instance per data-owning service (six in total), Multi-AZ, in private subnets.
- Carry domain events over **Amazon SNS** and **Amazon SQS**, with a dead-letter queue on every queue.
- Hold live driver state, the GEO index, Pub/Sub and a location stream in **Amazon ElastiCache for Valkey**.
- Keep database credentials in **AWS Secrets Manager** and encrypt data with **AWS KMS**. Give each workload only the IAM permissions it needs (IRSA).
- Publish one internet-facing entry: an **Application Load Balancer** created by the AWS Load Balancer Controller, behind **Amazon CloudFront** with the SPA stored in a private **Amazon S3** bucket.

#### Solution
- Build the network first (two public and two private subnets across two Availability Zones, one NAT gateway per zone), then IAM, ECR, data, messaging and the EKS cluster. The data, messaging and cluster layers depend only on IAM and network, so they can be built in any order.
- Push seven service images to Amazon ECR, then deploy them on Fargate.
- Create the frontend last, once the Ingress has an ALB address. CloudFront routes `/api/*` and `/live*` to that ALB and serves the SPA from S3.

#### Return on Investment
- **Repeatable:** every resource, name and setting is written down in the guides and templates, so the environment can be rebuilt or torn down on demand.
- **Least privilege:** each service has its own IRSA role limited to its own queue and the shared topic. Data tiers stay private.
- **Cost control:** a documented teardown order removes every billed resource when the environment is not needed.

---

### 3. Architecture Diagrams

#### Overall Architecture

```text
Browser --> CloudFront --> S3 (private SPA bucket)
               |
               +-- /api/*, /live* --> ALB --> EKS on Fargate (7 services)
                                                 |-- RDS PostgreSQL x6 (private subnets)
                                                 |-- ElastiCache for Valkey
                                                 +-- SNS topic --> SQS queues x3 (+ DLQs)
```

[OWNER: add an architecture diagram image if required]

#### Service Connection Architecture

The rule is: if the caller needs the answer to continue, it calls over HTTP. If something merely happened and others react later, an event is published.

| Interaction | Mode |
| --- | --- |
| Driver creation validates the user (driver to user) | Synchronous HTTP |
| Trip request authorizes the fare (trip to payment) | Synchronous HTTP |
| Trip request to nearby-driver matching | Event `trip.requested` through SNS to the matching queue |
| Trip completion or cancellation to payment capture or release | Events `trip.completed` and `trip.cancelled` through SNS to the payment queue |
| Every event to the notification inbox | SNS to the notifications queue (no filter) |
| Live driver position to the passenger | Valkey Pub/Sub to the realtime gateway, then SSE through the ALB `/live` path |
| Driver location to the matching index | Valkey Stream read through a consumer group |

Messaging resources: the SNS topic `ridenow-prod-events`, the queues `ridenow-prod-matching`, `ridenow-prod-payment` and `ridenow-prod-notifications`, each with a dead-letter queue (`maxReceiveCount` 5), and three subscriptions with raw message delivery.

#### Architecture Components

| Layer | AWS Service | Role in the system |
| --- | --- | --- |
| Identity | **AWS IAM** | Two EKS roles (`ridenow-prod-eks-cluster-role`, `ridenow-prod-eks-fargate-pod-execution-role`), five customer managed policies, and six IRSA roles for the add-ons and services. |
| Network | **Amazon VPC** | VPC `10.30.0.0/16`, two public and two private subnets, one internet gateway, two NAT gateways, and the security groups `ridenow-prod-cluster-sg` and `ridenow-prod-db-sg`. |
| Registry | **Amazon ECR** | Seven repositories (`ridenow/<service>`), scan on push, at most 50 images, untagged images expire after 14 days. |
| Data | **Amazon RDS for PostgreSQL, AWS Secrets Manager, AWS KMS** | Six `ridenow-prod-<service>-db` instances (`db.t4g.micro`, 20 GB, Multi-AZ, 30-day backups), encrypted with `alias/ridenow-prod-rds`, with RDS-managed master secrets. |
| Messaging | **Amazon SNS, Amazon SQS** | One topic, three queues, three dead-letter queues, three subscriptions and queue policies that admit only the topic. |
| Cache | **Amazon ElastiCache for Valkey** | Replication group `ridenow-prod-redis` (`cache.t4g.small`) with its subnet group and `ridenow-prod-redis-sg`. |
| Compute | **Amazon EKS on AWS Fargate** | Cluster `ridenow-prod` on Kubernetes 1.31 with the Fargate profiles `ridenow-prod-system`, `ridenow-prod-app` and `ridenow-prod-observability`. |
| Ingress | **Elastic Load Balancing (ALB)** | Application Load Balancer created from the `ridenow` Ingress by the AWS Load Balancer Controller. Five API paths plus `/live`. |
| Frontend | **Amazon S3, Amazon CloudFront** | Private bucket, origin access control `ridenow-prod-frontend-oac`, function `ridenow-prod-spa-rewrite`, and the distribution `RideNow prod frontend`. An optional GitHub OIDC role `ridenow-prod-frontend-deploy` publishes the site. |

---

### 4. Timeline & Milestones

The build follows the creation order of the guides. Dates and durations are not recorded in the repository.

| Step | Work items | Schedule |
| :--- | :--- | :--- |
| **1. Identity and network** | IAM roles and policies; VPC, subnets, NAT gateways, route tables, security groups. | [OWNER: fill in dates] |
| **2. Registry, data, messaging, cluster** | ECR repositories and image push; KMS, RDS and Secrets Manager; SNS, SQS and Valkey; EKS cluster and Fargate profiles. These four can run in any order after step 1. | [OWNER: fill in dates] |
| **3. Add-ons and services** | Pod network access rules, OIDC provider, IRSA roles; AWS Load Balancer Controller and External Secrets; deploy the seven services. | [OWNER: fill in dates] |
| **4. Frontend** | S3, CloudFront, optional deploy role; first publish of the site. Done last, after the Ingress has an ALB address. | [OWNER: fill in dates] |
| **5. Verification and teardown** | Run the smoke test and the guide checks; delete resources in reverse dependency order when finished. | [OWNER: fill in dates] |

---

### 5. Estimated Budget

The repository names the services that bill by the hour from the moment they are created. It contains no price figures, so the cost totals are left for the owner.

| Cost driver | Configuration |
| --- | --- |
| **EKS control plane** | One cluster, `ridenow-prod`. |
| **NAT gateways** | Two, one per Availability Zone, plus their Elastic IPs. |
| **Amazon RDS for PostgreSQL** | Six Multi-AZ `db.t4g.micro` instances with 20 GB storage and 30-day backups. |
| **Amazon ElastiCache for Valkey** | One replication group on `cache.t4g.small`. |
| **AWS Fargate** | Pods in the system, app and observability profiles. |
| **Application Load Balancer** | One ALB created by the Ingress. |
| **Other services** | SNS, SQS, ECR storage, Secrets Manager, KMS keys, S3 and CloudFront, charged by use. |

| Figure | Value |
| --- | --- |
| Estimated monthly cost | [OWNER: fill in the cost estimate] |
| Budget limit | [OWNER: fill in the budget limit] |

---

### 6. Risk Assessment

| Identified risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| **Hourly charges left running** | [OWNER: rate] | [OWNER: rate] | Follow the teardown order. Delete the Ingress ALB before the VPC. |
| **Wrong creation order** | [OWNER: rate] | [OWNER: rate] | Create the add-ons resources before installing the Load Balancer Controller and External Secrets. Create the frontend last. |
| **Database credentials exposed** | [OWNER: rate] | [OWNER: rate] | RDS-managed secrets in Secrets Manager, delivered through External Secrets and IRSA. Databases stay in private subnets. |
| **Open edge** | [OWNER: rate] | [OWNER: rate] | The ALB is HTTP on port 80 with no WAF and no source restriction. `docs/ARCHITECTURE.md` lists TLS, WAF and authentication as blockers before production use. |
| **Event loss or duplicates** | [OWNER: rate] | [OWNER: rate] | Dead-letter queues on every queue, idempotent consumers, and events published after commit. |

---

### 7. Expected Outcomes

- **A running `prod` environment** in `ap-southeast-1`: six databases, one topic with three queues and their dead-letter queues, one Valkey group, one EKS cluster with three Fargate profiles, seven images in ECR, an ALB, and the S3 and CloudFront frontend.
- **A repeatable record:** every resource name and setting is documented in the guides, and the workshop chapter walks through them one console at a time.
- **A clean exit:** a teardown order that removes every billed resource.

---

### 8. References

[1]: [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

[2]: [Amazon EKS Documentation](https://docs.aws.amazon.com/eks/)

[3]: [AWS Secrets Manager Developer Guide](https://docs.aws.amazon.com/secretsmanager/)
