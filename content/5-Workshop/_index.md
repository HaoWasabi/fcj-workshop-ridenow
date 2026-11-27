---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# Deploying RideNow on AWS

### Overview

This workshop shows how to create and configure, one service console at a time, the AWS services that **RideNow** runs on. You build the production environment (`prod`) in the region `ap-southeast-1` (Singapore). The workshop covers AWS setup only: it does not explain application code.

Every page uses the AWS Management Console for resource creation. The AWS CLI, `kubectl` and Helm appear only for steps the console cannot do, such as kubeconfig, add-on installs, Kubernetes secrets, image build and push, and verification checks. All commands are Bash (use WSL on Windows).

> Cost: the EKS control plane, NAT gateways, Multi-AZ RDS and ElastiCache bill by the hour from the moment they are created. Finish with 5.11 Clean up.

### Architecture Summary

The system is organized into the following layers:

| Layer | AWS services |
|-------|--------------|
| Identity | IAM roles and policies |
| Network | VPC, subnets, internet gateway, NAT gateways, route tables, security groups |
| Container images | Amazon ECR |
| Data | Amazon RDS PostgreSQL, AWS Secrets Manager, AWS KMS |
| Messaging and cache | Amazon SNS, Amazon SQS, Amazon ElastiCache for Valkey |
| Compute | Amazon EKS with AWS Fargate |
| Ingress | Application Load Balancer created by the AWS Load Balancer Controller |
| Frontend | Amazon S3 with Amazon CloudFront |

### What you build

| Item | Count |
|------|-------|
| RDS PostgreSQL instances (one per service) | 6 |
| ECR repositories | 7 |
| SNS topic (`ridenow-prod-events`) | 1 |
| SQS queues, each with a dead-letter queue | 3 queues, 3 dead-letter queues |
| Subnets | 2 public, 2 private |
| Fargate profiles | 3 |

### Contents

1. [Introduction](5.1-Workshop-overview/)
2. [Prerequisites](5.2-Prerequiste/)
3. [IAM](5.3-IAM/)
4. [Network](5.4-Network/)
5. [ECR](5.5-ECR/)
6. [Data](5.6-Data/)
7. [Messaging](5.7-Messaging/)
8. [EKS Cluster](5.8-EKS-Cluster/)
9. [EKS Addons](5.9-EKS-Addons/)
10. [Frontend](5.10-Frontend/)
11. [Clean up](5.11-Cleanup/)

### Creation order

Work through 5.3 to 5.10 in order, then clean up in 5.11 when you finish. Two ordering rules apply:

- **EKS Addons (5.9) comes after EKS Cluster (5.8) and before the AWS Load Balancer Controller and External Secrets installs.** The IRSA roles created in 5.9 are the roles those two installs run as. So: create the cluster in 5.8, create the 5.9 resources, return to the post-deploy steps of 5.8 to install the two add-ons, then deploy the seven services from 5.9.
- **Frontend (5.10) comes last,** after the services are deployed and the `ridenow` Ingress has an ALB address, because CloudFront uses that ALB as its API origin.

Data (5.6), Messaging (5.7) and EKS Cluster (5.8) depend only on IAM (5.3) and Network (5.4). Once those two are done, you can build these three in any order.
