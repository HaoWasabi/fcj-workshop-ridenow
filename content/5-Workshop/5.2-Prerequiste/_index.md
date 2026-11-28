
---
title : "Preparation Steps"
date : 2024-01-01
weight : 2
chapter : false
pre : " <b> 5.2. </b> "
---

# 5.2. PREPARATION STEPS FOR DEPLOYING RIDENOW ON AWS

## 5.2.1. Introduction

Before deploying the RideNow system on AWS, I need to prepare the working environment, install the necessary tools, and verify access permissions to the AWS account.

The purpose of this phase is to ensure that my computer can connect to and interact with AWS services while establishing consistent configuration parameters throughout the deployment process.

In this section, I will perform the following main tasks:

- Prepare the required deployment tools.
- Set up a Bash environment on Windows.
- Verify AWS credentials.
- Configure the AWS Region and deployment environment.
- Define and verify shared configuration variables.

---

## 5.2.2. Preparing the Deployment Tools

First, I install and verify the tools required to deploy the RideNow system.

The tools used include:

| Tool | Function |
|---|---|
| AWS CLI v2 | Interact with and manage AWS resources through the command line |
| Docker | Package services into Docker images and push them to Amazon ECR |
| kubectl | Manage Kubernetes resources on Amazon EKS |
| Helm | Install and manage applications on Kubernetes |
| jq | Read and process JSON data |
| Bash | Execute deployment commands |

AWS CLI is used to interact with AWS, Docker is used to package applications, while kubectl and Helm are used to manage Kubernetes components.

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-deployment-tools-overview.png)

Preparing all these tools helps ensure a consistent deployment process and minimizes errors caused by missing dependencies.

**Verify the installed tools using the following commands:**

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-check-tool-versions.png)

---

## 5.2.3. Setting Up the Bash Environment on Windows

Since the commands used in this workshop follow Bash syntax, while my computer runs the Windows operating system, I use Windows Subsystem for Linux (WSL).

WSL allows a Linux environment to run directly on Windows without installing a separate Linux operating system.

In this workshop, I use Ubuntu through WSL to execute AWS CLI commands and perform deployment operations.

To install Ubuntu, I open PowerShell with Administrator privileges and execute the following command:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-install-ubuntu-wsl.png)

After installation, I open the Ubuntu Terminal and verify the environment using the following command:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-verify-linux-bash-environment.png)

If the output displays Linux system information, the Bash environment is working properly.

---

## 5.2.4. Verifying AWS Identity and Access Permissions

After preparing the working environment, I verify the AWS credentials.

The purpose of this step is to confirm that AWS CLI is using the correct AWS account and an IAM identity with sufficient permissions to perform deployment operations.

I run the following command:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-aws-sts-get-caller-identity-command.png)

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-aws-sts-caller-identity-result.png)

This command uses AWS Security Token Service (STS) to retrieve information about the current identity.

The output contains three main fields:

- **UserId:** The unique identifier of the current AWS identity.
- **Account:** The ID of the AWS account currently being used.
- **Arn:** The full Amazon Resource Name of the IAM user or role.

Using this information, I confirm that I am working with the correct AWS account.

In addition, I use an IAM identity with appropriate administrative permissions instead of directly using the root account to minimize security risks.

---

## 5.2.5. Configuring the AWS Region and Deployment Environment

After successfully verifying access permissions, I configure the AWS Region for deploying the RideNow system.

In this workshop, I use the following AWS Region:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-aws-region-singapore.png)

This is the AWS Singapore Region.

I configure the default Region using the following command:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-configure-default-aws-region.png)

Then, I verify the configuration:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-verify-aws-region.png)

In addition, the RideNow system uses the `prod` environment, which represents the production environment.

Using a consistent Region and deployment environment helps ensure that AWS resources are managed consistently and prevents resources from being accidentally created in different Regions.

---

## 5.2.6. Defining Shared Configuration Variables

Instead of manually entering information such as the AWS account ID, Region, or Kubernetes cluster name in every command, I use Bash variables to store and reuse these values.

First, I retrieve the AWS account ID:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-set-aws-account-id-variable.png)

Next, I retrieve the configured AWS Region:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-set-aws-region-variable.png)

Then, I define the remaining variables:

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-set-deployment-environment-variables.png)

The meaning of each variable is described below:

| Variable | Description |
|---|---|
| ACCOUNT_ID | Current AWS account ID |
| AWS_REGION | AWS Region where the system is deployed |
| AWS_DEFAULT_REGION | Default AWS Region for AWS CLI |
| ENV | Deployment environment name |
| CLUSTER_NAME | Amazon EKS cluster name |
| IMAGE_TAG | Docker image version |

In this configuration, `CLUSTER_NAME` is set to `ridenow-prod`, while `IMAGE_TAG` is set to `v1.0.0`.

Using configuration variables makes deployment commands shorter, easier to manage, and less prone to input errors.

---

## 5.2.7. Verifying the Configuration Variables

After defining the variables, I verify their values using `echo` commands.

![RideNow preparation steps illustration](/fcj-workshop-ridenow/images/5.2-verify-deployment-variables.png)

The `ACCOUNT_ID` variable displays my actual AWS account ID.

Based on these results, I confirm that the variables have been defined correctly and can be used in subsequent deployment steps.

Additionally, I check whether `ACCOUNT_ID` or `AWS_REGION` is empty. If a `STOP` message appears, I will review the AWS credentials or Region configuration before proceeding.

One important point is that Bash variables only exist within the current shell session. Therefore, if I open a new terminal session, I need to define these variables again.

---

## 5.2.9. Results Achieved

After completing the preparation steps, I have successfully established the working environment required to deploy the RideNow system on AWS.

Specifically, the achieved results include:

1. Installed and prepared the required tools: AWS CLI, Docker, kubectl, Helm, jq, and Bash.
2. Set up the Ubuntu environment through WSL on Windows.
3. Successfully verified the AWS identity using AWS CLI.
4. Configured the deployment Region as `ap-southeast-1`.
5. Set up the `prod` environment and the cluster name `ridenow-prod`.
6. Successfully defined and verified the shared configuration variables.

As a result, my computer is now prepared for the infrastructure and application deployment steps of the RideNow system.

At this stage, I have only completed the environment preparation. The EKS Cluster and application services have not yet been deployed.

---

## 5.2.10. Conclusion

In summary, Section 5.2 focuses on preparing the necessary requirements before deploying the RideNow system on AWS.

By installing the required tools, setting up the Bash environment, verifying the IAM identity, and configuring shared variables, I have established a consistent working environment for the next deployment steps.

This is an important foundational step to ensure that AWS infrastructure and application deployment operations are performed using the correct account, Region, and configuration.

After completing the preparation phase, I will proceed to deploy the AWS infrastructure resources required for the RideNow system.
