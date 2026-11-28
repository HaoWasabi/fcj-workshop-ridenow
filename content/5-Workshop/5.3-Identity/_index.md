---
title : "Identity and Access Management"
date : 2024-01-01 
weight : 3
chapter : false
pre : " <b> 5.3. </b> "
---

# PART 01. CONFIGURING IAM ON AWS CONSOLE
## 1. Introduction to IAM
First, I will configure IAM on the AWS Console to prepare the access permissions required for the RideNow system.
IAM stands for Identity and Access Management, an AWS service for managing identities and controlling access to AWS resources.
In RideNow, components such as the EKS Cluster, Trip Service, Payment Service, Matching Service, and Notification Worker require different access permissions.
Therefore, I will first create 2 IAM Roles and 5 IAM Policies based on the `iac/stacks/01-iam.yaml` configuration file.
**Illustration:** IAM Dashboard on the AWS Console.
## 2. Creating an IAM Role for the EKS Cluster
First, I open the AWS Console, search for the IAM service, and select Roles.
Then, I select Create role to create a new IAM Role.

Under Trusted entity type, I select AWS service. Next, I select EKS and the EKS - Cluster use case so AWS can configure the appropriate trust relationship.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-eks-cluster-role-trusted-entity.png)

![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-eks-cluster-role-use-case.png)
In the permissions configuration step, I use the `AmazonEKSClusterPolicy` policy required for the EKS Cluster.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-eks-cluster-role-attach-policy.png)

Then, I name the Role:
`ridenow-prod-eks-cluster-role`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-eks-cluster-role-name.png)

This Role will be used when creating the EKS Cluster, allowing EKS to perform the AWS resource operations needed to operate and manage the Cluster.

After opening the Role, I select the **Permissions** tab to check the assigned permissions.
Here, I verify whether the `AmazonEKSClusterPolicy` policy has been attached to the Role.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-eks-cluster-role-permissions.png)

![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-eks-cluster-policy-details.png)
This AWS-managed policy provides the permissions Amazon EKS needs to perform Cluster management tasks.
If the Role does not have this policy, I select **Add permissions** and then **Attach policies**.
Next, I search for `AmazonEKSClusterPolicy`, select it, and click **Add permissions** to attach it to the Role.
Afterward, I check the Permissions list again to confirm that the policy has been attached successfully.

Next, I switch to the **Trust relationships** tab.

For the EKS Cluster Role, Amazon EKS must be the service allowed to assume the Role.
Therefore, I check two important settings in the Trust policy.
The first is `eks.amazonaws.com`, which identifies Amazon EKS as the trusted service.
The second is `sts:AssumeRole`, which allows EKS to assume the IAM Role and use its assigned permissions.
With this configuration, Amazon EKS can assume the Role to perform the operations needed to run the Cluster.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-eks-cluster-role-trust-policy.png)

If the Trust policy is not configured correctly, I select **Edit trust policy** to modify it.
Here, I use JSON to specify Amazon EKS as the trusted service.
Specifically, the `Effect` property is set to `Allow`, indicating that the action is permitted.
The `Principal` property is set to `eks.amazonaws.com`, identifying Amazon EKS as the service allowed to assume the Role.
Finally, the `Action` property is set to `sts:AssumeRole`, allowing EKS to assume the Role through AWS Security Token Service.

After verifying these settings, I click **Update policy** to save the configuration.
I have now completed the configuration and verification of the IAM Role for the EKS Cluster.
This Role will be used in the next Amazon EKS deployment step, providing the Cluster with the permissions needed to operate on AWS infrastructure.

## 3. Creating an IAM Role for Fargate Pod Execution
Next, I will create the second IAM Role for AWS Fargate.
Fargate allows containers to run without directly managing EC2 servers.
In RideNow, Fargate can be used to run application Pods inside EKS.
To create the Role, I go to IAM, select Roles, and click Create role.
Under Trusted entity type, I select AWS service and choose EKS - Fargate pod.
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-fargate-pod-role-trusted-entity.png)

Next, I attach the policy:
`AmazonEKSFargatePodExecutionRolePolicy`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-fargate-pod-role-attach-policy.png)
Then, I enter the name:
`ridenow-prod-eks-fargate-pod-execution-role`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-fargate-pod-execution-role-name.png)
Finally, I select Create role.

This Role allows Fargate infrastructure to perform operations needed to run Pods, such as pulling container images from Amazon ECR.
This Role does not directly grant AWS resource access permissions to the application code inside the Pods.

## 4. Creating the App Access Policy
After creating the two IAM Roles for the EKS Cluster and Fargate, I will create 5 IAM Policies for the microservices in RideNow.
RideNow microservices need to interact with AWS services such as Amazon ECR, Secrets Manager, SQS, and SNS.  
Therefore, I create Customer Managed Policies to control access for each service instead of granting full permissions to all microservices.  
The system includes 5 policies:
- AppAccessPolicy
- TripServiceMessagingPolicy
- PaymentServiceMessagingPolicy
- MatchingServiceMessagingPolicy
- NotificationWorkerMessagingPolicy  
    Each policy manages a separate group of access permissions, making the system easier to manage and limiting unnecessary permissions.  
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-app-access-policy-configuration.png)

First, I go to IAM, select Policies, and click Create policy.
Here, I switch to JSON mode to define access permissions.

I use the PolicyDocument corresponding to `AppAccessPolicy` in `iac/stacks/01-iam.yaml`.

After entering the JSON, I select Next and enter the name:
`ridenow-prod-app-access`
![Ảnh minh họa cấu hình IAM](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-app-access-policy-name.png)

Finally, I select Create policy.
This policy defines AWS resource access permissions for the RideNow application.
The specific allowed actions and resources depend on the configured JSON content.

## 5. Creating the Trip Service Messaging Policy
Next, I will create a policy for Trip Service.

![IAM configuration screenshot](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-trip-service-messaging-policy-configuration.png)

Trip Service is responsible for handling ride-related business operations.
In a microservices architecture, Trip Service may need to exchange messages with other components through AWS messaging services.
I go to IAM, select Policies, and click Create policy.
Then, I select JSON and enter the permissions defined in `TripServiceMessagingPolicy` in the YAML file.

![IAM configuration screenshot](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-trip-service-messaging-policy-creation.png)

I enter the policy name:
`ridenow-prod-trip-service-messaging`
Then, I select Create policy.
This policy restricts access to the messaging resources required by Trip Service rather than granting broad access to the entire system.
**Illustrations:**
- JSON content of the Trip Service Messaging Policy.
- Successful policy creation screen.
## 6. Creating the Payment Service Messaging Policy
Next, I will configure a policy for Payment Service.

![IAM configuration screenshot](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-payment-service-messaging-policy-configuration.png)

Payment Service is responsible for handling payment-related business operations in RideNow.
Therefore, it may need the messaging system to send or receive payment-processing messages.
As in the previous steps, I go to IAM, select Policies, and click Create policy.
I enter the PolicyDocument corresponding to `PaymentServiceMessagingPolicy`.
The policy name is:
`ridenow-prod-payment-service-messaging`

![IAM configuration screenshot](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-payment-service-messaging-policy-name.png)

After reviewing the content, I select Create policy to finish.
Creating a separate policy for Payment Service separates payment-related access permissions from those of other microservices.
**Illustrations:**
- JSON content of the Payment Service Messaging Policy.
- Successful policy creation screen.
## 7. Creating the Matching Service Messaging Policy
Next, I create a policy for Matching Service.

![IAM configuration screenshot](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-matching-service-messaging-policy-configuration.png)

Matching Service is responsible for matching drivers with customer ride requests.
During operation, Matching Service may need to exchange messages with other services to receive matching requests or send processing results.
I go to IAM, select Policies, and click Create policy.
Then, I enter the JSON defined in `MatchingServiceMessagingPolicy`.
I enter the name:
`ridenow-prod-matching-service-messaging`

![IAM configuration screenshot](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-matching-service-messaging-policy-name.png)

Finally, I select Create policy.
This policy defines the scope of messaging permissions specifically for Matching Service.
**Illustrations:**
- JSON content of the Matching Service Messaging Policy.
- Successful policy creation screen.
## 8. Creating the Notification Worker Messaging Policy
The last policy I need to create is the Notification Worker Messaging Policy.

![IAM configuration screenshot](/fcj-workshop-ridenow/images/5-Workshop/5.3/5.3-notification-worker-messaging-policy-configuration.png)

Notification Worker handles notification tasks in the RideNow system.
For example, when a ride is confirmed or its status changes, the system may generate a message for Notification Worker to receive and process.
To create the policy, I go to IAM, select Policies, and click Create policy.
I enter the JSON content corresponding to `NotificationWorkerMessagingPolicy` from the YAML file.
Then, I enter the name:
`ridenow-prod-notification-worker-messaging`
Finally, I select Create policy.
This policy manages the messaging permissions required by Notification Worker.
**Illustrations:**
- JSON content of the Notification Worker Messaging Policy.
- Successful policy creation screen.
## 9. Verifying the IAM Configuration
After completing the configuration steps, I will verify all resources created on the AWS Console.
First, under IAM Roles, I check the two Roles:
- `ridenow-prod-eks-cluster-role`
- `ridenow-prod-eks-fargate-pod-execution-role`
Next, under IAM Policies, I check the five policies:
- `ridenow-prod-app-access`
- `ridenow-prod-trip-service-messaging`
- `ridenow-prod-payment-service-messaging`
- `ridenow-prod-matching-service-messaging`
- `ridenow-prod-notification-worker-messaging`
In addition to checking their names, I review the defined permissions, the trust relationships of both Roles, and the attached policies.
If all resources match the configuration, I have completed IAM preparation for the RideNow system.
**Illustrations:**
- List of the 2 created IAM Roles.
- List of the 5 created customer managed policies.
## 10. Conclusion
In the first step of deploying RideNow to AWS, I have configured a total of 7 IAM resources: 2 IAM Roles and 5 IAM Policies.
The two IAM Roles provide permissions for the EKS Cluster and Fargate infrastructure.
The five IAM Policies define access permissions for the application and microservices, including Trip Service, Payment Service, Matching Service, and Notification Worker.
Separating these policies makes access management clearer and supports the principle of least privilege, meaning each component receives only the permissions it actually needs.
After completing IAM configuration, the system has the foundational access controls needed to deploy additional AWS resources in the next steps.
