---
title: "Workshop"
date: 2024-01-01
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

# DevOps on AWS with RakiBookery - Bookery Website

### Overview

This workshop guides you through the entire process of building, developing, and operating **RakiBookery** - an online bookery website - on the Amazon Web Services (AWS) platform. The project employs modern Cloud & DevOps models, featuring a fully automated CI/CD pipeline, Multi-AZ readiness, and a comprehensive monitoring system.

The workshop is divided into key stages:

- **Infrastructure**: Setting up the network and AWS service foundation (VPC, RDS, S3, IAM)
- **Deployment**: Building a CI/CD pipeline using GitHub Actions and deploying the application to ECS Fargate
- **Operations**: Configuring auto-scaling, CDN distribution, monitoring, cost alerts, and end-to-end testing
- **Illustrations**: Reference list of all screenshots used in the workshop

### Architecture Summary

The system is organized into the following main layers:

| Layer | Components |
|-------|------------|
| CI/CD | GitHub Actions, Docker, Amazon ECR |
| Presentation | Amazon CloudFront, Amazon S3 (Frontend) |
| Application | Application Load Balancer, Amazon ECS Fargate |
| Data | Amazon RDS MySQL (Multi-AZ) |
| Monitoring | Amazon CloudWatch, Amazon SNS, AWS Budgets |

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
10. [Clean up](5.10-Cleanup/)