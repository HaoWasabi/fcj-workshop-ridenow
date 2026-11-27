---
title: "Week 6 Worklog"
date: 2026-11-23
weight: 6
chapter: false
pre: " <b> 1.6. </b> "
---

### Week 6 Objectives:

* Create the EKS cluster on Fargate and get it ready for real traffic.
* Give each workload only the permissions it needs.
* Deploy the seven services and expose them through one load balancer.

| Day | Task | Start Date | Completion Date | Reference Material |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Create the cluster, its KMS key and the three Fargate profiles. <br> - Connect with kubectl and look at what runs by default. | 23/11/2026 | 23/11/2026 | [Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html) |
| 3 | - Patch CoreDNS to run on Fargate. <br> - Create the OIDC provider and the IAM roles for service accounts (IRSA). | 24/11/2026 | 24/11/2026 | [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) |
| 4 | - Install the AWS Load Balancer Controller and External Secrets with Helm. <br> - Open the database and cache ports to the pods. | 25/11/2026 | 25/11/2026 | [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/) |
| 5 | - Deploy the seven services with their manifests and check the pods. <br> - Fix the pods that stayed Pending or crashed. | 26/11/2026 | 26/11/2026 | |
| 6 | - Create the Ingress, wait for the ALB and run the API test against it. <br> - Write down every command that worked, in order. | 27/11/2026 | 27/11/2026 | |

### Week 6 Achievements:

&emsp;◉ Monday:
<br>&emsp;&emsp;○ Cluster created. The control plane takes about ten minutes and I stared at the screen for most of it.

&emsp;◉ Tuesday:
<br>&emsp;&emsp;○ CoreDNS stayed Pending until I patched it for Fargate. This took me half a day to find.

&emsp;◉ Wednesday:
<br>&emsp;&emsp;○ Controller and secrets operator running. I learned that the order of steps matters: the IAM roles must exist before the installs.

&emsp;◉ Thursday:
<br>&emsp;&emsp;○ Pods could not read their database secrets at first. The cause was a missing security group rule and a wrong service account name.

&emsp;◉ Friday:
<br>&emsp;&emsp;○ The API test passed through the ALB. I felt proud, and then immediately thought about the bill and wrote the teardown notes.