---
title : "Introduction"
date : 2024-01-01 
weight : 1 
chapter : false
pre : " <b> 5.1. </b> "
---

#### Purpose of this chapter

This chapter shows how to create and configure, in the AWS Management Console, the AWS services that the RideNow production environment (`prod`, region `ap-southeast-1`) runs on. Each section covers one layer of the infrastructure, from IAM to the S3 and CloudFront frontend hosting, and ends with a check you can run to confirm the layer works.

#### Scope

The chapter covers only the creation and configuration of AWS services: what each resource is called, how it is set up, what it depends on, and which values later sections reuse.

The chapter does not cover application code. It does not describe, list, or explain the source code of the RideNow services or the frontend, their endpoints, or their message formats.

RideNow application services appear only as consumers of AWS resources. For example, `ridenow/trip-service` is an ECR repository, and `ridenow-prod-trip-service` is the IAM role that the trip service uses to reach SNS and SQS.

#### What you will build

| Layer | AWS services |
|---|---|
| Identity | IAM roles and policies |
| Network | VPC, subnets, NAT gateways, security groups |
| Images | ECR repositories |
| Data | RDS PostgreSQL, Secrets Manager, KMS |
| Messaging and cache | SNS, SQS, ElastiCache for Valkey |
| Compute | EKS with Fargate |
| Ingress | Application Load Balancer through the AWS Load Balancer Controller |
| Frontend | S3 and CloudFront |

#### How the chapter is organized

Sections 5.3 to 5.10 each create one layer and follow the same headings: set variables, what it creates, depends on, create the resources, values to record, post-deploy steps, and verify. Section 5.11 deletes everything in the reverse order so that charges stop.

#### Next step

Go to [5.2 Prerequisites](../5.2-Prerequiste/) to prepare your account and tools and to set the variables shared by all later sections.
