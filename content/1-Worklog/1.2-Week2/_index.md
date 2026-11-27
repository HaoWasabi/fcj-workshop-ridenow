---
title: "Week 2 Worklog"
date: 2026-10-26
weight: 2
chapter: false
pre: " <b> 1.2. </b> "
---

### Week 2 Objectives:

* Turn the RideNow idea into written requirements the whole team agrees on.
* Decide the service boundaries and which data each service owns.
* Sketch a first architecture on AWS and choose how the services will run.
* Split the work between the five of us.

| Day | Task | Start Date | Completion Date | Reference Material |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Write the first list of what riders, drivers and admins need to do. <br> - Cut it down to a version we can build in the time we have. | 26/10/2026 | 26/10/2026 | |
| 3 | - Split the system into seven services: user, driver, trip, payment, matching, notification and a realtime gateway. <br> - Decide that each service owns its own database. | 27/10/2026 | 27/10/2026 | [Microservices on AWS](https://docs.aws.amazon.com/whitepapers/latest/microservices-on-aws/microservices-on-aws.html) |
| 4 | - Work out which calls must be synchronous HTTP and which things should be events. <br> - Write the simple rule we use for it and the list of events. | 28/10/2026 | 28/10/2026 | [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) <br> [Amazon SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) |
| 5 | - Compare ways to run containers (EC2, Fargate, EKS) and pick EKS on Fargate. <br> - Draw the first architecture diagram with VPC, private subnets, RDS, a cache and a load balancer. | 29/10/2026 | 29/10/2026 | [Amazon EKS on AWS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html) |
| 6 | - Split the work: teammates build the services, and I take the cloud side and lead. <br> - Write the plan for the eight weeks and review it with the mentor. | 30/10/2026 | 30/10/2026 | |

### Week 2 Achievements:

&emsp;◉ Monday:
<br>&emsp;&emsp;○ Our first requirement list was far too big. Cutting features was harder than writing them.

&emsp;◉ Tuesday:
<br>&emsp;&emsp;○ Seven services with their own databases. It sounded like a lot for five people, but each service stays small.

&emsp;◉ Wednesday:
<br>&emsp;&emsp;○ Got a rule everyone can remember: if the caller needs the answer to continue, call over HTTP. If something just happened and others can react later, publish an event.

&emsp;◉ Thursday:
<br>&emsp;&emsp;○ Chose EKS on Fargate so nobody has to patch servers. I knew it would be the hardest part for me, but also the part I would learn the most from.

&emsp;◉ Friday:
<br>&emsp;&emsp;○ Everyone knew their part by the end of the day. The mentor asked us to think about cost early, so I added it to my list.