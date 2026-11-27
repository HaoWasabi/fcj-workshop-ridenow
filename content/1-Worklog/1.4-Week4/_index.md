---
title: "Week 4 Worklog"
date: 2026-11-09
weight: 4
chapter: false
pre: " <b> 1.4. </b> "
---

### Week 4 Objectives:

* Containerize every service and push the images to a registry.
* Design the AWS network and permissions that everything else will sit on.
* Start turning my console clicks into CloudFormation templates.
* Keep supporting the teammates while they finish the services.

| Day | Task | Start Date | Completion Date | Reference Material |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Check each Dockerfile with the teammates so images stay small and run as a normal user. <br> - Build all seven images locally. | 09/11/2026 | 09/11/2026 | [Dockerfile best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/) |
| 3 | - Design the network: one VPC, two public and two private subnets across two Availability Zones, a NAT gateway per zone. <br> - Write the security group rules for the cluster and the databases. | 10/11/2026 | 10/11/2026 | [VPC and subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html) |
| 4 | - Write the first CloudFormation templates: network and IAM roles for the cluster. <br> - Deploy them in the real account and fix the errors. | 11/11/2026 | 11/11/2026 | [CloudFormation User Guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) |
| 5 | - Write the ECR template with seven repositories, scan on push and a lifecycle rule. <br> - Push the first images. | 12/11/2026 | 12/11/2026 | [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) |
| 6 | - Agree on naming rules so every resource is called `ridenow-prod-...`. <br> - Weekly check-in with the mentor and update the plan. | 13/11/2026 | 13/11/2026 | |

### Week 4 Achievements:

&emsp;◉ Monday:
<br>&emsp;&emsp;○ Seven images built. Two of them were over a gigabyte, so we trimmed them.

&emsp;◉ Tuesday:
<br>&emsp;&emsp;○ Network design done. I underestimated how much a NAT gateway costs per hour, so I wrote it into the budget notes.

&emsp;◉ Wednesday:
<br>&emsp;&emsp;○ My first template failed three times because of a missing dependency and a wrong policy ARN. After that I started reading CloudFormation events before guessing.

&emsp;◉ Thursday:
<br>&emsp;&emsp;○ Images are in ECR and tagged.

&emsp;◉ Friday:
<br>&emsp;&emsp;○ A single naming rule saved us from a lot of confusion later when the workshop pages had to name every resource.