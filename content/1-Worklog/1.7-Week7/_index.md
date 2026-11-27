---
title: "Week 7 Worklog"
date: 2026-11-30
weight: 7
chapter: false
pre: " <b> 1.7. </b> "
---

### Week 7 Objectives:

* Host the frontend on S3 and CloudFront and set up its deployment.
* Make the environment easy to delete and rebuild.
* Turn what I built into step-by-step console guides and hand the pieces to the team.

| Day | Task | Start Date | Completion Date | Reference Material |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Create the private S3 bucket, origin access control and CloudFront distribution. <br> - Route `/api/*` and `/live*` to the load balancer. | 30/11/2026 | 30/11/2026 | [Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html) |
| 3 | - Set up GitHub Actions with an OIDC role so the site can be published without access keys. <br> - Test a publish and a rollback. | 01/12/2026 | 01/12/2026 | [GitHub OIDC with AWS](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) |
| 4 | - Write the teardown order and test it on a full environment. <br> - Rebuild everything from scratch to prove the steps are repeatable. | 02/12/2026 | 02/12/2026 | |
| 5 | - Write the eight console guides, one per layer, from my CloudFormation templates. <br> - Hand out the guides: Nhi takes IAM, network and ECR, Hào takes data, Thuận takes messaging, Đào takes the EKS cluster and add-ons. | 03/12/2026 | 03/12/2026 | |
| 6 | - Write a checking script for the guides so mistakes show up early. <br> - Answer the first questions from teammates who started their pages. | 04/12/2026 | 04/12/2026 | |

### Week 7 Achievements:

&emsp;◉ Monday:
<br>&emsp;&emsp;○ The site opened through CloudFront on the first try after I fixed the origin path.

&emsp;◉ Tuesday:
<br>&emsp;&emsp;○ Publishing needs no keys at all. I liked how clean that felt.

&emsp;◉ Wednesday:
<br>&emsp;&emsp;○ The first teardown left a load balancer behind, which blocked deleting the VPC. I fixed the order and wrote it down in bold.

&emsp;◉ Thursday:
<br>&emsp;&emsp;○ Everyone got a clear piece of the guides. Splitting by layer made it easy to say who owns what.

&emsp;◉ Friday:
<br>&emsp;&emsp;○ The script already caught a few missing names in the first drafts.