
---
title : "Introduction"
date : 2024-01-01
weight : 1
chapter : false
pre : " <b> 5.1. </b> "
---

# 5.1. Introduction – Building the Production Infrastructure for RideNow on AWS

## 5.1.1. System Overview and Deployment Objectives

RideNow is a system deployed using a Microservices architecture, in which application functionalities are organized into separate services.

To enable these services to operate in a real-world environment, in addition to the application source code, we need to build a supporting infrastructure that provides essential capabilities such as networking, security, data storage, inter-service communication, and container deployment.

In this chapter, I focus on the **Production** environment, named `prod`, and select the AWS Region **Asia Pacific (Singapore)**, corresponding to the region code `ap-southeast-1`.

Production is the environment used to operate the system under real-world conditions. Therefore, infrastructure components must be clearly organized, with appropriate access control mechanisms, data protection measures, and operational support.

The objective of this chapter is to build the infrastructure layers required for RideNow using the AWS Management Console. Each layer will also be verified after configuration to ensure that the resources meet the system's requirements.

**[Insert Figure 1: Overview of the AWS Management Console or RideNow infrastructure architecture on AWS]**

## 5.1.2. Main Infrastructure Layers of the RideNow System

Deploying a Microservices system on AWS requires the integration of multiple AWS services.

In this chapter, I divide the infrastructure into eight main layers. Each layer performs a specific role while interacting with the others to form a complete system.

**The first layer is Identity – Identity and Access Management.**

In this layer, I use AWS IAM, including IAM Roles and IAM Policies.

IAM allows us to define which AWS resources or services are permitted to perform specific actions.

For example, a service that only needs to read messages from SQS will be granted the necessary SQS-related permissions instead of receiving administrative access to the entire AWS account.

This follows the principle of Least Privilege, which means granting each component only the minimum permissions required to perform its tasks.

**The second layer is Network – Network Infrastructure.**

In this layer, I use Amazon VPC, Subnets, NAT Gateways, and Security Groups.

Amazon VPC provides a private networking environment for organizing AWS resources.

Subnets are used to divide the network into different areas, such as Public Subnets and Private Subnets.

NAT Gateways allow resources in Private Subnets to initiate outbound connections to external services when necessary, without allowing unsolicited inbound connections from the Internet through the NAT Gateway.

Meanwhile, Security Groups act as network traffic filters, controlling which inbound and outbound connections are permitted for resources.

Combining these components helps separate public and internal network areas while limiting unnecessary access.

**[Insert Figure 2: VPC architecture showing Public Subnets, Private Subnets, NAT Gateway, and Security Groups]**

**The third layer is Images – Container Image Management.**

In this layer, I use Amazon Elastic Container Registry, also known as Amazon ECR.

Amazon ECR is an AWS service used to store and manage container images.

In the RideNow system, each Microservice is packaged as a Docker Image and stored in a separate ECR Repository.

For example, `ridenow/user-service` is used to store images for the User Service, while `ridenow/driver-service` is used for the Driver Service.

In addition to image storage, Amazon ECR supports image version management, access control, encryption, and vulnerability scanning.

This allows Docker Images to be managed centrally and used during deployment to Amazon EKS.

**The fourth layer is Data – Data Storage and Protection.**

For the data layer, I use Amazon RDS for PostgreSQL, AWS Secrets Manager, and AWS KMS.

Amazon RDS for PostgreSQL provides a managed relational database service.

AWS Secrets Manager is used to store and manage sensitive information, such as database connection credentials.

Meanwhile, AWS KMS supports encryption key management to help protect data.

Combining these services allows the database to be separated from application services while strengthening the protection of important system information.

**The fifth layer is Messaging and Cache – Messaging and Caching.**

In this layer, I use Amazon SNS, Amazon SQS, and Amazon ElastiCache for Valkey.

Amazon SNS supports the Publish/Subscribe model, allowing an event to be distributed to multiple subscribed components.

Amazon SQS provides message queues that enable asynchronous communication between services and reduce direct dependencies between them.

Meanwhile, Amazon ElastiCache for Valkey provides high-speed in-memory caching, supporting the storage of frequently accessed data.

These services are particularly suitable for a Microservices architecture because services can exchange information or process tasks through messages instead of always communicating through direct service-to-service calls.

**[Insert Figure 3: Diagram illustrating the ECR, RDS, SNS, SQS, and ElastiCache layers of RideNow]**

**The sixth layer is Compute – Application Execution Platform.**

In this layer, I use Amazon Elastic Kubernetes Service, also known as Amazon EKS, together with AWS Fargate.

Amazon EKS is a managed Kubernetes service provided by AWS that enables the deployment and orchestration of containerized applications.

AWS Fargate provides the ability to run containers without directly managing the underlying EC2 servers.

As a result, the development team can focus more on configuring and operating application workloads instead of managing the entire server infrastructure.

**The seventh layer is Ingress – Incoming Traffic Management.**

In this layer, I use Application Load Balancer together with AWS Load Balancer Controller.

Application Load Balancer is responsible for receiving and distributing HTTP or HTTPS requests to the appropriate application components.

AWS Load Balancer Controller helps manage AWS load balancing resources based on Kubernetes configurations.

As a result, incoming traffic from external users can be routed to the corresponding services within the RideNow system.

**The final layer is Frontend – User Interface Delivery.**

In this layer, I use Amazon S3 and Amazon CloudFront.

Amazon S3 is used to store static frontend files, such as HTML, CSS, and JavaScript.

Meanwhile, Amazon CloudFront is a Content Delivery Network, or CDN, service that distributes content through a network of edge locations.

Combining S3 and CloudFront provides a frontend storage and delivery mechanism suitable for a cloud environment.

In summary, these eight infrastructure layers perform different functions, ranging from access management and networking to data storage, container deployment, and user interface delivery.

When integrated, they form the infrastructure foundation required to operate the RideNow system on AWS.

**[Insert Figure 4: Overall RideNow architecture showing all eight AWS infrastructure layers]**

## 5.1.3. Relationships Between Infrastructure Layers

The infrastructure layers described above do not operate completely independently. Instead, they have dependencies on one another.

For example, before deploying Microservices to Amazon EKS, we need to prepare the networking infrastructure, including the VPC and Subnets.

At the same time, services must be assigned appropriate IAM permissions to access the required AWS resources.

When deploying applications as containers, Docker Images stored in Amazon ECR are used as image sources for workloads running on EKS.

During operation, services can connect to the PostgreSQL database hosted on Amazon RDS, use SNS and SQS to exchange messages, or access ElastiCache to retrieve cached data.

For incoming user requests, Application Load Balancer is responsible for receiving and routing traffic to the appropriate application components.

On the frontend side, user interface files are stored in Amazon S3 and distributed through Amazon CloudFront.

Therefore, the deployment process must follow a logical sequence because the configuration of one infrastructure layer may depend on resources or information created in a previous layer.

This is also why the chapter is divided into smaller sections instead of creating all infrastructure resources at once.
