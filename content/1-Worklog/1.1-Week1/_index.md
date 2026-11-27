---
title: "Week 1 Worklog"
date: 2026-10-19
weight: 1
chapter: false
pre: " <b> 1.1. </b> "
---

### Week 1 Objectives:

* Meet the team and the mentors, and understand how the internship works.
* Get a working AWS account with MFA, IAM users and a budget alarm, so we cannot get a surprise bill.
* Learn the core ideas: Regions, Availability Zones, VPC, IAM, and what "container on AWS" really means.
* Throw ideas around with the team and agree on one project to build for the next eight weeks.

| Day | Task | Start Date | Completion Date | Reference Material |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - First meeting with the mentors and the other four members (Thuận, Đào, Hào, Nhi). <br> - Go through the internship rules, the schedule up to 11/12 and how we report progress. <br> - Create the shared group chat and the team repository. | 19/10/2026 | 19/10/2026 | [FCJ - Explore AWS Services](https://cloudjourney.awsstudygroup.com/1-explore/) |
| 3 | - Create my AWS account, turn on MFA for the root user and make an admin IAM user for daily work. <br> - Set up AWS Budgets and a billing alarm. <br> - Read about Regions and Availability Zones. | 20/10/2026 | 20/10/2026 | [Create Your First AWS Account](https://000001.awsstudygroup.com/) |
| 4 | - Learn the basics of VPC: subnets, route tables, internet gateway, NAT gateway. <br> - Click through a first VPC in the console to see how the pieces connect. | 21/10/2026 | 21/10/2026 | [Amazon VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/what-is-amazon-vpc.html) |
| 5 | - Learn IAM properly: users, roles, policies, and why a role is better than an access key. <br> - Each member writes down two or three project ideas. | 22/10/2026 | 22/10/2026 | [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |
| 6 | - Team brainstorm: compare the ideas by "can we finish it in eight weeks" and "does it use enough AWS services". <br> - Agree on a ride-hailing platform and name it RideNow. <br> - Agree that I will lead the project and own the cloud side. | 23/10/2026 | 23/10/2026 | |

### Week 1 Achievements:

&emsp;◉ Monday:
<br>&emsp;&emsp;○ Met everyone and set up the repository and group chat.

&emsp;◉ Tuesday:
<br>&emsp;&emsp;○ Account, MFA, admin user and budget alarm were ready before I touched any other service.

&emsp;◉ Wednesday:
<br>&emsp;&emsp;○ VPC finally made sense after I drew it on paper and then built it by hand once.

&emsp;◉ Thursday:
<br>&emsp;&emsp;○ Understood why we should never put access keys in code, which later shaped how the whole project handles credentials.

&emsp;◉ Friday:
<br>&emsp;&emsp;○ The team chose RideNow. Ride-hailing felt right because it needs many services talking to each other, which is a good reason to use AWS properly.
<br>&emsp;&emsp;○ I took the lead role. It felt a bit scary, so I wrote down what I needed to decide next week.