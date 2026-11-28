---
title : "Container Image"
date : 2024-01-01
weight : 5
chapter : false
pre : " <b> 5.5. </b> "
---

# PART 5.5 – DEPLOYING CONTAINER IMAGES ON AMAZON ECR

## 1. Introduction to Amazon ECR

After completing the configuration of the networking system and security components on AWS, I proceeded to prepare a storage location for the Docker images of the RideNow services.
For this step, I used **Amazon Elastic Container Registry**, also known as Amazon ECR.
Amazon ECR is an AWS service for storing and managing container images, such as Docker images.
In the RideNow system, each service is packaged as a separate Docker image. These images are then pushed to Amazon ECR for subsequent application deployment to Amazon EKS.
Using ECR provides the system with centralized image storage while supporting access control, data encryption, and image version management.

## 2. Creating ECR Repositories for the RideNow System

After accessing the AWS Management Console, I proceeded to create repositories to store Docker images for the RideNow system.
First, I searched for **Amazon Elastic Container Registry**, or Amazon ECR. This AWS service stores and manages Docker images used for application deployment.

Next, in the Amazon ECR interface, I opened **Private Repositories** and selected **Create repository** to create a new repository.
In the configuration section, I selected **Private** to restrict access to the system's Docker images.
Under Repository name, I entered `ridenow/user-service` to create the first repository, dedicated to the user management service.
I also set **Image Tag Mutability to Immutable** to prevent Docker images associated with an existing version tag from being overwritten.
Under Encryption Settings, I selected **KMS with the AWS Managed Key** `**aws/ecr**` to encrypt image data at rest.
After completing the settings, I selected Create repository to create the resource.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-create-private-ecr-repository.png)

After successfully creating the first repository, I repeated the process for the remaining services.
In total, **7 ECR repositories** were prepared for the RideNow system:
- `ridenow/user-service`: Manages user information.
- `ridenow/driver-service`: Manages driver information.
- `ridenow/trip-service`: Processes trip information.
- `ridenow/payment-service`: Handles payment functions.
- `ridenow/matching-service`: Matches drivers with passengers.
- `ridenow/notification-worker`: Processes system notifications.
- `ridenow/realtime-gateway`: Supports real-time communication functions.
All repositories were consistently configured as Private, with Immutable image tags and encryption using AWS KMS.

![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ridenow-ecr-repositories-list.png)

Here, I used the naming convention `ridenow/service-name` to make it easy to identify repositories belonging to the RideNow project.
Creating a separate repository for each service also aligns with the system's **Microservices architecture**.
Specifically, each service has its own Docker image and independent versions. When a service needs to be updated, I can build and push a new image version to its corresponding repository without modifying the images of other services.
For example, if I update the trip management functionality, I only need to build a new Docker image for `trip-service` and push it to `ridenow/trip-service`.
This makes managing, updating, and deploying the system's components clearer and more convenient.
**Thus, after this step, I had prepared all 7 ECR repositories for the RideNow system.** These repositories will store Docker images for deploying services to Amazon EKS in the following steps.

Next, I will explain the **Scan on Push configuration**, which helps automatically check Docker images for security vulnerabilities when they are pushed to Amazon ECR.

When creating a repository, under Visibility Settings, I selected Private.
This means the system's Docker images are not publicly available for anyone on the Internet to download.
Instead, repository access is controlled through AWS permissions mechanisms, such as IAM.
When deploying the application to Amazon EKS, only components with appropriate permissions can pull Docker images from ECR.
This helps limit unauthorized access to system images.

Next, under Image Tag Mutability, I selected Immutable.
Normally, when building Docker images, we use tags to identify versions, such as `v1.0.0`, `v1.0.1`, or `v2.0.0`.
With Immutable enabled, a tag that has already been used in a repository cannot be overwritten by another image.
For example, if I have pushed a Docker image with the tag `v1.0.0`, subsequent pushes cannot replace the image associated with that tag.
To deploy a new version, I must use a different tag, such as `v1.0.1`.
This setting reduces the risk of accidentally overwriting deployed versions and supports application version management and traceability.

After configuring Immutable, I proceeded to configure repository encryption.
Under Encryption Settings, I selected KMS and used the AWS-managed key named `aws/ecr`.
AWS KMS is AWS's key management service.
With this key, Docker image data stored in Amazon ECR is encrypted at rest.
This helps strengthen the protection of system data.
Using an AWS Managed Key also means I do not need to create and manage a separate KMS key myself.
I applied the same encryption configuration to all 7 RideNow repositories.

## 3. Configuring Scan on Push – Checking Docker Image Security on Amazon ECR

### 3.1. Overview of Scan on Push

After creating all 7 ECR repositories for the RideNow system, I proceeded to configure Scan on Push to enhance Docker image security.
First, we need to understand why Docker images require security scanning.
In a Microservices architecture, each service is usually packaged into a separate Docker image. However, a Docker image contains not only application code but also components such as the runtime environment, libraries, and dependent software packages.
These components may have security vulnerabilities, particularly when older or unpatched software versions are used.
For this reason, Amazon ECR provides **Scan on Push**, which automatically triggers a security scan whenever a new Docker image is pushed to a repository.
For example, when I build a new version of `user-service` and push its Docker image to Amazon ECR, the system automatically scans it to detect known security vulnerabilities.
Developers can then review and assess security issues before using the image to deploy an application on Amazon EKS.
However, Scan on Push only helps detect vulnerabilities; it does not automatically fix them or guarantee that a Docker image is completely secure.

### 3.2. How Basic Scanning Works in Amazon ECR

To implement Scan on Push, Amazon ECR provides a mechanism called Basic Scanning.
Basic Scanning is a security scanning method that helps identify known vulnerabilities in supported software components inside container images.
The scanning process examines software package information and compares it against published vulnerability data.
An important concept in this process is **CVE**, which stands for Common Vulnerabilities and Exposures.
CVE is a system of identifiers for publicly disclosed security vulnerabilities.
For example, if the Docker image for `payment-service` uses a library version with a known vulnerability, Basic Scanning may detect it and provide information about that vulnerability.
Security findings can be classified by severity, such as Critical, High, Medium, or Low, helping developers prioritize issues for remediation.
For RideNow, I chose Basic Scanning because it meets the basic vulnerability scanning requirements for Docker images.
This configuration adds a layer of security to container management without requiring a separate scanning system.

### 3.3. Configuring Basic Scanning in the AWS Console

After understanding how the mechanism works, I applied it to the RideNow system on AWS.
First, I accessed the AWS Management Console and selected **Amazon Elastic Container Registry**, or Amazon ECR.
I used the **Asia Pacific (Singapore)** Region, corresponding to `ap-southeast-1`, where the system's resources are deployed.
In Amazon ECR, I opened **Private Registry**, then selected **Scanning** to access the Scanning Configuration page.
This is where AWS allows security scanning to be configured for container images stored in the Private Registry.

Under Scan Type, I selected **Basic Scanning**.
This means I use Amazon ECR's basic scanning mechanism to check for known vulnerabilities in supported software components.
After selecting Basic Scanning, I configured Scan on Push so that a scan is automatically triggered when a new Docker image is pushed to a repository.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-basic-scanning-configuration.png)

### 3.4. Setting Up Repository Filters for the RideNow System

Next, I needed to specify which repositories would have Scan on Push enabled.
The RideNow system has 7 ECR repositories corresponding to its 7 services: user management, driver management, trip processing, payments, driver-passenger matching, notification processing, and real-time communication.
These repositories share the naming convention `ridenow/service-name`.
Therefore, instead of setting up scanning individually for each repository, I used a **Repository Filter** to apply one common policy.
Under Scan on Push Filters, I entered:
`ridenow/*`
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-scan-on-push-repository-filter.png)
Here, `ridenow/` is the common prefix for RideNow repositories, while `*` represents the service-name portion that follows.
For example, this filter applies to `ridenow/user-service`, `ridenow/trip-service`, `ridenow/payment-service`, and other repositories with the same prefix.
The key advantage is that **one filter is enough to enable Scan on Push for all 7 repositories**.
In addition, if the system adds another service in the future and its repository also uses the `ridenow/` prefix, the existing Scan on Push configuration can apply to the new repository.
This makes security management simpler, more consistent, and easier to scale as the system grows.

### 3.5. How Scan on Push Works in the RideNow System

At this point, Amazon ECR has been configured to automatically scan new Docker images pushed to repositories matching the `ridenow/*` filter.
To illustrate this, I will describe a practical example from the RideNow system.
Suppose the development team has just updated the trip processing functionality in `trip-service`.
First, the service's source code is packaged into a Docker image using Docker Build.
The image is then assigned a version tag, such as `v1.0.0`, and pushed to the `ridenow/trip-service` repository using Docker Push.
When Amazon ECR receives the new image, it recognizes that the repository matches the `ridenow/*` filter and automatically triggers a security scan.
During scanning, Amazon ECR checks supported software components inside the image for known vulnerabilities.
Once the scan finishes, the results are recorded for developers to review.
If vulnerabilities are detected, the team can investigate the cause, update libraries, or change the base image, then build and push a new Docker image version.
The overall workflow can therefore be described as follows:
**Source Code → Docker Build → Docker Push → Amazon ECR → Basic Scanning → Scan Results**
With this mechanism, security checks are integrated directly into the process of uploading Docker images to the registry.
However, Scan on Push does not automatically block deployment of an image containing vulnerabilities. To achieve that, additional control policies must be added to the CI/CD or deployment process.

### 3.6. The Role of Scan on Push in a Microservices Architecture

In the RideNow system, each microservice is developed and deployed relatively independently.
This provides flexibility in scaling and maintenance but also increases the number of Docker images that must be managed.
Each service may use different libraries and software versions, resulting in different security risks.
Without centralized scanning, vulnerability detection may depend heavily on manual work by individual development teams.
With Scan on Push, Amazon ECR provides a consistent scanning mechanism for new images pushed to RideNow repositories.
Whenever a service is updated and a new Docker image is created, a security scan is automatically triggered according to the configuration.
This helps detect issues in the software components inside the image before the development team decides to deploy that version.
Furthermore, combining Scan on Push with Private Repositories, Immutable image tags, and encryption using AWS KMS provides multiple layers of protection.
Private Repositories help control access, Immutable tags prevent existing tagged images from being overwritten, KMS protects stored image data, and Scan on Push helps identify security vulnerabilities.
Each mechanism serves a distinct purpose and contributes to stronger security when managing and deploying RideNow services.

## 4. Configuring the First Lifecycle Rule – Automatically Deleting Untagged Docker Images After 14 Days

### 4.1. Overview of Lifecycle Policies in Amazon ECR

After configuring Scan on Push to check Docker images for security vulnerabilities, I proceeded to configure a **Lifecycle Policy** in Amazon Elastic Container Registry.
First, we need to understand what a Lifecycle Policy is and why it is useful.
During software development, services are frequently updated to add features, fix bugs, or improve performance.
With each update, developers may build a new Docker image and push it to Amazon ECR for deployment.
For example, in RideNow, `user-service` may go through multiple versions such as `v1.0.0`, `v1.0.1`, and `v1.0.2`.
As versions accumulate, old Docker images may remain in the repository even when they are no longer used.
This increases storage consumption and can generate additional costs.
To address this, Amazon ECR offers **Lifecycle Policies**, which let administrators define rules for automatically managing and deleting Docker images that meet specified conditions.
These conditions can be based on tag status, image age, or the number of images in a repository.
This allows the system to automatically clean up images that are no longer needed instead of requiring administrators to review and delete them manually.

### 4.2. Understanding Tagged and Untagged Images

To understand how Lifecycle Policies work, we need to distinguish between two important concepts: **Tagged Images** and **Untagged Images**.
A Tagged Image is a Docker image with one or more tags identifying its version or purpose.
For example:
`ridenow/user-service:v1.0.0`
Here, `ridenow/user-service` is the repository name and `v1.0.0` is the image tag.
Tags help developers identify which version is in use and simplify application deployment.
By contrast, an Untagged Image is an image that currently has no associated tags.
During image updates, particularly when a tag is reassigned to another image in a repository that allows tag overwriting, the old image may become Untagged if no other tags remain.
An image can also become Untagged when its tags are deleted.

However, **Untagged does not necessarily mean an image is no longer being used**.
A workload may reference an image by digest instead of by tag. Therefore, before applying a deletion policy, administrators must ensure that images eligible for cleanup are no longer needed by the system.
In RideNow, repositories use Immutable image tags to prevent version overwriting. Thus, images becoming Untagged through tag overwriting is not a typical scenario, although untagged images may still appear during repository management.
The first Lifecycle Rule is designed to handle this group of images.

### 4.3. How a Time-Based Lifecycle Rule Works

Amazon ECR allows Lifecycle Rules to be configured based on how long Docker images have been stored.
Here, I use the **Untagged condition combined with a 14-day threshold**.
This means the system considers only images that are currently untagged and older than the configured period.
For example, suppose the `ridenow/user-service` repository contains three images.
The first image has the tag `v1.0.0` and has been stored for 30 days.
The second image has no tag and has been stored for 5 days.
The third image has no tag and has been stored for 20 days.
When the Lifecycle Policy is evaluated, the first image is excluded because it still has a tag.
The second image is untagged but has not exceeded 14 days, so it is not yet eligible for deletion.
The third image is untagged and older than 14 days, so it becomes eligible for deletion under the Lifecycle Policy.
Importantly, this period is calculated from the image storage timestamp used by ECR, such as when the image was pushed, rather than when its tag was removed.
Also, Amazon ECR processes deletions automatically after images meet the criteria; this does not mean an image is deleted at the exact moment it enters its 15th day.
Therefore, the Lifecycle Rule uses two combined conditions: **the image must be Untagged and must be older than 14 days**.

### 4.4. Configuring the Lifecycle Rule in the AWS Console

I opened the Private Repositories list and selected `ridenow/user-service`.
Within the repository management interface, I opened **Lifecycle Policy** to configure Docker image lifecycle management rules.
Here, Amazon ECR allows administrators to define conditions that automatically identify images for deletion.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-lifecycle-policy-untagged-14-days.png)
I then selected Create Rule to create the first rule.
Under Rule Priority, I entered `1`.
Rule Priority determines the priority order of rules within the same Lifecycle Policy.
Generally, a lower number indicates a higher priority. Since this is the first rule, I assigned Priority 1.

Under Image Status, I selected **Untagged**.
This limits the rule to images that currently have no tags.
Under Match Criteria, I selected the condition based on time since the image was added to the repository and set it to **14 days**.
Finally, under Rule Action, I selected **Expire**.
Expire instructs Amazon ECR to delete images that meet the rule's criteria.

### 4.5. The Role of Lifecycle Policies in Cost Optimization and System Management

Lifecycle Policies not only automate Docker image cleanup but also play an important role in managing AWS resources.
The first benefit is **optimizing storage usage**.
Amazon ECR stores images and related data. As the number of images increases, storage usage may also grow, resulting in additional costs.
Automatically deleting unnecessary images helps prevent unused data from accumulating.
The second benefit is **reducing manual administrative work**.
Without Lifecycle Policies, administrators must regularly inspect repositories, identify old images, and delete them manually.
When there are many microservices, this work can become complicated and time-consuming.
The third benefit is **ensuring consistency in resource management**.
In RideNow, all 7 repositories use the same rule to delete untagged images after 14 days.
As a result, the services follow a consistent management policy, making maintenance and scaling easier.
However, a Lifecycle Policy only manages the lifecycle of images within a repository. It does not replace backups, deployment version management, or application security controls.

### 4.6. Conclusion

In this section, I explored and configured the first Lifecycle Rule in Amazon ECR.
In theory, a Lifecycle Policy is a mechanism that automatically manages Docker image lifecycles based on predefined conditions.
For the first rule, I used an Image Status of Untagged, a 14-day threshold, and the Expire action.
In practice, I configured the rule with Priority 1 on the `ridenow/user-service` repository and then applied the same configuration to the remaining RideNow repositories.
This configuration automatically cleans up untagged Docker images older than the specified period, reducing unnecessary image accumulation, saving storage space, and lowering administrative effort.
**Lifecycle Policies are therefore an important mechanism for managing container registries efficiently, especially in Microservices systems like RideNow that contain many services and frequently release new versions.**
However, the first rule only handles untagged Docker images.
In practice, tagged images can also accumulate over multiple application updates.
Therefore, in the next section, I will explain the **second Lifecycle Rule – Limiting Each Repository to a Maximum of 50 Docker Images**, which provides more comprehensive control over stored image counts.

## 5. Configuring the Second Lifecycle Rule – Keeping a Maximum of 50 Docker Images in Amazon ECR

### 5.1. Overview of Docker Image Count Management

After configuring the first Lifecycle Rule to automatically delete untagged Docker images after 14 days, I proceeded to set up the second Lifecycle Rule.
First, we need to understand why limiting the number of Docker images stored in Amazon ECR is necessary.
During software development, whenever a service is updated or a bug is fixed, developers typically build a new Docker image.
For example, in RideNow, `trip-service` may evolve through versions such as `v1.0.0`, `v1.0.1`, `v1.0.2`, and many more over time.
Although older versions may no longer be used for deployment, they can remain stored in Amazon ECR.
Without a management policy, the number of Docker images can keep growing, making administration more difficult and increasing storage costs.
In particular, RideNow uses a Microservices architecture with 7 independent services. Each service can therefore produce multiple versions of its own Docker image.
For this reason, in addition to deleting untagged images, the system needs a mechanism to limit the total number of images stored in each repository.
This is the purpose of the second Lifecycle Rule.

### 5.2. Understanding Image Count in Amazon ECR

In Amazon ECR, Image Count refers to the number of Docker images stored in a repository.
As Docker image versions are built and pushed to Amazon ECR, the number of images can increase over time.
For example, a repository might store 10, 30, 50, or 100 images depending on how frequently the service is updated.
Amazon ECR supports count-based Lifecycle Policies, allowing administrators to define a storage threshold and automatically delete older images when the threshold is exceeded.
For this exercise, I chose a limit of **50 Docker images per repository**.
This means that when a repository contains more than 50 images, Amazon ECR evaluates older images for deletion according to the Lifecycle Policy.
For example, if `ridenow/trip-service` contains 60 images, the 10 oldest images would be identified for deletion when considering the image-count rule on its own.
This helps keep the number of stored images within the configured limit and prevents excessive accumulation of old versions.
However, the 50-image limit is a storage management policy; it does not mean Amazon ECR rejects new Docker images when a repository already contains 50 images.
Instead, new images can still be pushed, after which the Lifecycle Policy evaluates and processes images eligible for deletion.

### 5.3. How a Lifecycle Rule with Image Status Any Works

To better understand the second rule, we need to examine the Image Status option in Amazon ECR.
In the first Lifecycle Rule, I selected Untagged, meaning the rule only applies to Docker images without tags.
However, if only that rule is used, images that still have tags will not be deleted by the 14-day condition.
For example, versions `v1.0.0`, `v1.0.1`, and `v1.0.2` would remain because they still have tags.
As more versions are released, the repository can still accumulate many Docker images.
Therefore, for the second Lifecycle Rule, I selected **Image Status: Any**.
Any means the rule can apply to both tagged and untagged Docker images.
When Image Status Any is combined with an Image Count threshold of 50, the system evaluates the total number of images in the repository and identifies older images for deletion when the limit is exceeded.
An important point is that this rule can delete not only untagged images but also images that still have version tags.
Therefore, before applying this rule in Production, administrators should check carefully to avoid deleting versions still needed for operation or system recovery.

### 5.4. Configuring the Second Lifecycle Rule in the AWS Console

First, I opened Amazon Elastic Container Registry in the Asia Pacific (Singapore) Region, `ap-southeast-1`.
Next, I navigated to Private Repositories and selected `ridenow/user-service`.
In the repository management interface, I opened Lifecycle Policy.
Here, the first Lifecycle Rule already exists with Priority 1, deleting untagged images older than 14 days.
To add a rule limiting the image count, I selected Create Rule.

In the rule creation interface, I first set **Rule Priority to 2**.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-lifecycle-rule-keep-50-images.png)
Rule Priority indicates a rule's priority within a Lifecycle Policy.
A lower number has a higher priority, so Rule 1 takes precedence over Rule 2.
In addition, Amazon ECR requires a rule using Image Status Any to have the lowest priority in its Lifecycle Policy.
Since RideNow uses two rules, assigning Priority 2 satisfies this requirement.

Under Match Criteria, I selected **Image Count More Than** and entered `50`.
Finally, under Rule Action, I selected Expire.
Expire means Amazon ECR deletes Docker images that meet the conditions of the Lifecycle Rule.
Thus, the second rule is configured as follows:
- Rule Priority: `2`
- Rule Description: `Keep only the most recent tagged images`
- Image Status: `Any`
- Match Criteria: `Image count more than 50`
- Rule Action: `Expire`

### 5.5. Combining Both Lifecycle Rules in the RideNow System

After saving, the `ridenow/user-service` repository will have two Lifecycle Rules in total.
The first rule has Priority 1 and applies to untagged images older than 14 days.
The second rule has Priority 2, applies to all images, and uses an image-count limit of 50.
These two rules complement each other in Docker image management.
The first rule focuses on images without tags that have exceeded the specified age threshold.
Meanwhile, the second rule limits the number of images accumulated in the repository, including images that still have version tags.
For example, if a repository contains many untagged images older than 14 days, those images may be eligible for deletion under Rule 1.
If the repository still contains more than 50 images, Rule 2 is used to identify older images eligible for deletion according to the image-count condition and Amazon ECR's rule priority logic.
Note that Amazon ECR evaluates rules within a Lifecycle Policy according to image selection and priority rules; it does not simply run each rule as an independent deletion command.
After checking the configuration, I applied the same two Lifecycle Rules to the remaining 6 RideNow repositories.
Thus, all 7 repositories use the same Docker image lifecycle management mechanism, with a total of 14 Lifecycle Rules.

### 5.6. Benefits of Limiting the Number of Docker Images

Configuring the second Lifecycle Rule provides several benefits for operating the RideNow system.
The first is **controlling storage usage**.
When a service is updated frequently, many Docker image versions are created. Without cleanup, a repository can accumulate an increasing number of images.
Limiting the number of images helps remove old versions that are no longer needed, which can save storage space.
The second benefit is **optimizing AWS costs**.
Amazon ECR may incur charges based on the amount of stored data. Removing unnecessary images can therefore help lower operating costs.
The third benefit is **simplifying administration**.
Instead of repeatedly reviewing repositories and deleting images manually, administrators can define conditions for Amazon ECR to handle automatically.
This is especially useful in a Microservices system like RideNow, where each service has an independent development and update lifecycle.
The final benefit is **maintaining consistency in system management**.
When all 7 repositories use the same Lifecycle Policy, image retention periods and counts can be managed consistently.
However, a 50-image limit should not be considered a security measure or a backup strategy.
Lifecycle Policies only manage image storage and deletion within repositories. The system still needs appropriate version management and recovery strategies, especially for images in use in Production.

## 6. Applying Lifecycle Policies to All 7 Repositories

After configuring both Lifecycle Rules for the first repository, I applied the same configuration to the remaining 6 repositories.
As a result, each RideNow repository has two lifecycle management rules.
The first rule deletes untagged images older than 14 days.
The second rule keeps a maximum of 50 images in each repository.
Across 7 repositories, I configured a total of 14 Lifecycle Rules.
Applying the same set of rules ensures consistent repository management, simplifies maintenance, and helps control storage consumption.

## 7. Checking Repository URIs, Building Docker Images, and Pushing Them to Amazon ECR

### 7.1. Overview of the Docker Image Deployment Process

After creating all 7 ECR repositories, configuring Scan on Push, and setting up Lifecycle Policies for RideNow, I proceeded to build and upload Docker images to Amazon ECR.
First, we need to understand the role of Docker images in application deployment.
In a Microservices architecture, each service is developed and managed relatively independently. To deploy these services across different environments, we need a way to package each application with the components required to run it.
Docker is a platform that supports packaging applications into container images.
A Docker image can contain application code, a runtime environment, libraries, and dependencies. Containers can then be created from these images to run the applications.
In the RideNow project, there are 7 services, corresponding to 7 Docker images that must be built and managed independently.
Once built, the images are pushed to Amazon Elastic Container Registry, or Amazon ECR.
Amazon ECR serves as centralized storage for container images, helping the system manage versions, control access, and provide images for deployment to Amazon EKS.
The RideNow container image deployment workflow can therefore be described as follows:
**Source Code → Docker Build → Docker Image → Docker Push → Amazon ECR → Amazon EKS**
In this practical section, I focus on checking repositories, building Docker images, pushing them to Amazon ECR, and verifying the results after uploading them to the registry.

### 7.2. Understanding Repository URIs in Amazon ECR

Before building and pushing Docker images, I need to determine the addresses of the repositories created in Amazon ECR.
Each ECR repository has its own address, called a **Repository URI**.
URI stands for Uniform Resource Identifier and is used to identify a resource.
In Amazon ECR, a Repository URI tells Docker exactly where a container image is stored and retrieved.
A Repository URI generally follows this structure:
`<account-id>.dkr.ecr.<region>.amazonaws.com/<repository-name>`
Where:
- `account-id` is the AWS account identifier.
- `dkr.ecr` indicates the Docker registry provided by Amazon ECR.
- `region` is the AWS Region in which the repository was created.
- `repository-name` is the name of the repository being accessed.
For example, in RideNow, the user management service repository URI has this structure:
`<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/ridenow/user-service`
This address helps Docker identify the correct repository when pushing or pulling images.
Combining the URI with a version tag gives us the complete Docker image name.
For example:
`<account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/ridenow/user-service:v1.0.0`
Here, `v1.0.0` is the tag identifying the image version.
Using tags makes application version management clearer and supports deploying or reverting to a stored version when needed.

### 7.3. Checking Repository URIs in the AWS Console and AWS CLI

In the Private Repositories list, I checked the repositories created for all 7 services.
Amazon ECR displays the corresponding URI in each repository's information so that Docker can push and pull images.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-repository-uri.png)

In addition to checking directly in the Console, I used AWS CLI to retrieve this information.
AWS CLI is a command-line tool for interacting with AWS services.
Because my development environment uses Windows, I ran the commands in PowerShell.
First, I used `aws sts get-caller-identity` to identify the currently active AWS account.
I then defined the necessary variables, including Account ID, Region, and Image Tag.
I set the Region to `ap-southeast-1` and the Image Tag to `v1.0.0`.
Next, I used `aws ecr describe-repositories` to retrieve the repository list and corresponding URIs.
From the returned results, I could verify that all 7 repositories existed and that their URIs were correct.
This check is very important because using an incorrect Account ID, Region, or Repository URI can prevent Docker from finding the repository when pushing images.

### 7.4. Understanding Dockerfiles and Docker Build

After identifying the Repository URIs, I proceeded to build Docker images for the RideNow services.
To understand this process, we first need to distinguish between a Dockerfile, a Docker Image, and a Docker Container.
A **Dockerfile** is a file containing instructions that Docker follows to build an image.
In a Dockerfile, developers can specify the runtime environment, working directory, files to copy, commands to install dependencies, and the command used to start the application.
A **Docker Image** is the result of Docker executing the instructions in a Dockerfile.
An image acts as a template for creating containers.
A **Docker Container**, meanwhile, is a running instance of a Docker image with its own runtime environment and processes.
Simply put, a Dockerfile is the build instruction document, a Docker image is the packaged product, and a Docker container is the environment in which the application runs.
The process of creating a Docker image from a Dockerfile is called **Docker Build**.
In RideNow, each service is packaged into its own image so that application components can be managed and deployed independently.
This is particularly suitable for a Microservices architecture: when one service is updated, the development team can build a new image for that service without necessarily rebuilding the entire system.

### 7.5. Building 7 Docker Images for the RideNow System

First, I used Docker Desktop to provide the Docker Engine needed to build container images.
Next, I opened the RideNow source code in Visual Studio Code and used PowerShell to run Docker commands.
Before building, I checked the locations of the Dockerfiles and identified the appropriate Build Context for each service.

Build Context is the set of files and directories made available to Docker during the image build process.
This is especially important for projects with multiple modules or shared libraries because a Dockerfile may need to access files located outside a service's directory.
After identifying the correct Dockerfile and Build Context, I used `docker buildx build` to build the Docker image.
The `--platform linux/amd64` parameter specifies Linux AMD64 as the target platform architecture.
The `-t` parameter assigns a name and version tag to the image.

![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-build-service-docker-images.png)

After successfully building the first image, I repeated the process for the remaining 6 services: Driver Service, Trip Service, Payment Service, Matching Service, Notification Worker, and Realtime Gateway.

Each service was built as a separate Docker image.
After building, I used `docker images` to check the images available in the local Docker environment.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-verify-local-docker-images.png)

At this stage, the RideNow components had been packaged into container images and were ready to be uploaded to Amazon ECR.

### 7.6. How Docker Authentication with Amazon ECR Works

Before pushing Docker images, I needed to establish authentication between Docker and Amazon ECR.
Because RideNow repositories are configured as Private, Docker cannot freely push or pull images without appropriate access permissions.
Amazon ECR uses the AWS permission system, in which IAM manages the actions an AWS identity is allowed to perform.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-docker-authenticate-with-ecr.png)

To log Docker in to Amazon ECR, I used the AWS CLI command `aws ecr get-login-password`.
This command retrieves a temporary authentication password from Amazon ECR using the current AWS credentials.
The password is then passed to `docker login` to authenticate Docker with the registry.

An ECR authentication token remains valid for 12 hours, so Docker may need to log in again after the token expires.

However, successful login does not mean that the account can perform every operation. Permission to push or pull images still depends on the applicable IAM permissions and access policies.
This mechanism allows the system to control which identities can upload Docker images to ECR and helps prevent unauthorized operations.

### 7.7. Pushing Docker Images to Amazon ECR with Docker Push

After completing authentication, I proceeded to push the Docker images to Amazon ECR.
This operation is called **Docker Push**.
Docker Push transfers container image data from the local Docker environment to a container registry.
Docker images consist of layers. During a push, Docker checks and uploads the layers needed for the registry to store the image.
To push to the correct repository, the image must be tagged with the full name based on its Repository URI and Image Tag.
For example, for `user-service`, I used the URI of `ridenow/user-service` and the `v1.0.0` tag.
I then ran `docker push` to upload the image to Amazon ECR.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-push-docker-images-to-ecr.png)

After successfully pushing the first image, I repeated the process for the remaining 6 images.
Each image was pushed to the repository corresponding to its service.
One important point is that I configured **Image Tag Mutability as Immutable** for RideNow repositories.
This means that once the tag `v1.0.0` already exists, the system does not allow another image to overwrite that tag.
To deploy a new version, I must use a new tag, such as `v1.0.1`.
This setting reduces accidental overwriting of stored versions and supports application version management.
After Docker Push is complete, the RideNow images are stored in Amazon ECR rather than existing only on the development machine.

### 7.8. Verifying Docker Images After Pushing

After pushing Docker images to Amazon ECR, I verified the results.
First, I opened the AWS Management Console and navigated to Amazon ECR.
From the Private Repositories list, I opened each RideNow repository in turn.
In the Images section, Amazon ECR displays information about the stored images.
Important details include Image Tag, push time, image size, and Image Digest.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-verify-pushed-images-in-ecr.png)

An **Image Tag** is used to name and manage image versions.
An **Image Digest**, by contrast, is an identifier derived from image content, typically represented as a SHA-256 hash.
The difference is that a tag is a version reference name, while a digest is determined from the image content according to its manifest format.
The digest therefore helps identify the exact stored image, avoiding confusion between image versions that may have had the same tag at different times or in different environments.

In addition to checking the AWS Console, I can use the AWS CLI command `aws ecr describe-images`.
This command retrieves image information, including tags, push times, and digests.
Using this information, I can verify that each Docker image has actually been stored in the correct system repository.

### 7.9. Checking Scan on Push Results After Uploading Images to ECR

After pushing the Docker images to Amazon ECR, I checked the security scan results.
Earlier, I configured **Basic Scanning with Scan on Push** for repositories using the `ridenow/` prefix.
With this configuration, pushing a new Docker image to a matching repository automatically triggers scanning for known security vulnerabilities.
To review the results, I opened a repository in the AWS Console, selected the recently pushed image, and opened Scan Results or Image Scan Findings.
Here, the system provides information about vulnerabilities detected in supported software components of the image.
Findings may be categorized by severity levels such as Critical, High, Medium, and Low.
![Amazon ECR configuration illustration](/fcj-workshop-ridenow/images/5-Workshop/5.5/5.5-ecr-image-scan-findings.png)

For example, if a Docker image uses a library version with a publicly disclosed vulnerability, the scanner may record that finding for developers to evaluate.

I can also use `aws ecr describe-image-scan-findings` to check the results directly in PowerShell.
This command lets me view the scanning status and the total number of findings grouped by severity.
If vulnerabilities are found, the development team should identify the affected components and assess the risks.
Depending on the situation, the team may update the base image, upgrade libraries, or fix related software components.
A new Docker image can then be built and pushed to ECR to run another scan.
Importantly, **Scan on Push only detects vulnerabilities; it does not automatically remediate them or block the deployment of vulnerable images by default**.
Therefore, scan results must still be reviewed and combined with CI/CD controls to meet security requirements before application deployment.

### 7.10. The Role of the Build and Push Process in a Microservices Architecture

In RideNow, building and storing Docker images on Amazon ECR plays an important role in Microservices deployment.
First, it **ensures independence between services**.
Each service is packaged into a separate image. When one service changes, the development team can build and deploy a new version without necessarily changing the images of other services.
Second, it supports **application version management**.
Image Tags and Image Digests help the system identify versions that are stored and used during deployment.
Third, it **centralizes container image storage**.
Instead of distributing images across individual developers' machines, Amazon ECR provides a central registry from which deployment environments can retrieve images from a common source.
Fourth, it **strengthens security controls**.
Repositories are configured as Private, with Immutable image tags, encryption using AWS KMS, and Scan on Push.
These mechanisms support access control, protection of image data at rest, version management, and detection of known security vulnerabilities.
Finally, it **supports deployment to Amazon EKS**.
After Docker images are stored in ECR, Kubernetes on Amazon EKS can use them to launch containers, provided the deployment components have the necessary permissions and configuration.
This makes the process of moving applications from source code to the runtime environment more organized, manageable, and scalable as the system evolves.

## 8. Conclusion

In this section, I explored and implemented the process of building, storing, and inspecting Docker images on Amazon ECR for the RideNow system.
From a theoretical perspective, I explained Repository URIs, Dockerfiles, Docker Images, Docker Build, Docker Push, Image Tags, and Image Digests.
I also discussed how Docker authenticates with Amazon ECR through AWS CLI and IAM.
In practice, the workflow involved checking the URIs of 7 repositories, identifying Dockerfiles and Build Contexts, building 7 Docker images, authenticating with ECR, and pushing the images to their respective repositories.
I then verified the stored images and Scan on Push results to confirm that the upload process met the registry deployment requirements.
Through this process, RideNow can manage each microservice's Docker images independently while taking advantage of the version control and security mechanisms provided by AWS.
**Combining Docker with Amazon ECR is an important part of deploying applications based on a Microservices architecture, bridging the development environment and the operating environment in the Cloud.**
After this phase, the system's Docker images will be ready for use when deploying services to Amazon EKS in the next sections.
