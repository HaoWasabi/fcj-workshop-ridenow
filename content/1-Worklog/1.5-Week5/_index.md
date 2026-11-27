---
title: "Week 5 Worklog"
date: 2026-11-16
weight: 5
chapter: false
pre: " <b> 1.5. </b> "
---

### Week 5 Objectives:

* Build the data layer: databases, secrets, encryption.
* Build the messaging layer: SNS, SQS and the cache for live driver positions.
* Check that events flow the way we designed them.

| Day | Task | Start Date | Completion Date | Reference Material |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Create the KMS key and the DB subnet group. <br> - Create the first RDS PostgreSQL instance by hand and check it only accepts traffic from inside the VPC. | 16/11/2026 | 16/11/2026 | [Amazon RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html) <br> [AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html) |
| 3 | - Create the other five databases. <br> - Use RDS-managed master secrets in Secrets Manager so nobody ever sees or types a password. | 17/11/2026 | 17/11/2026 | [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) |
| 4 | - Create the SNS topic, three queues with dead-letter queues, and the subscriptions with filters. <br> - Write the queue policies so only the topic can send messages. | 18/11/2026 | 18/11/2026 | [Amazon SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) |
| 5 | - Create the Valkey replication group for live driver state, geo lookup and the location stream. <br> - Open the right security group rules for it. | 19/11/2026 | 19/11/2026 | [Amazon ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html) |
| 6 | - Send test events through the topic and watch them land in the right queues. <br> - Review the Week 5 pages that Hào and Thuận started for the workshop. | 20/11/2026 | 20/11/2026 | |

### Week 5 Achievements:

&emsp;◉ Monday:
<br>&emsp;&emsp;○ First database up. Multi-AZ takes a long time to create, so I learned to start it and do something else.

&emsp;◉ Tuesday:
<br>&emsp;&emsp;○ Six databases, six secrets, zero passwords in our notes.

&emsp;◉ Wednesday:
<br>&emsp;&emsp;○ A message did not arrive because my queue policy had the wrong topic ARN. The dead-letter queue helped me notice it.

&emsp;◉ Thursday:
<br>&emsp;&emsp;○ Cache reachable from the cluster security group only.

&emsp;◉ Friday:
<br>&emsp;&emsp;○ Events flow the way we drew them on the whiteboard in week 2, which felt very good.