---
title: "Week 3 Worklog"
date: 2026-11-02
weight: 3
chapter: false
pre: " <b> 1.3. </b> "
---

### Week 3 Objectives:

* Design the code side so five people can work in parallel without stepping on each other.
* Agree on API contracts, events and database ownership.
* Make the whole system run on one laptop before we touch the cloud.

| Day | Task | Start Date | Completion Date | Reference Material |
| :-: | :--- | :----------: | :-------------: | :--- |
| 2 | - Set up the repository layout, branch rules and the pull request habit. <br> - Write a short guide on how each service should be structured. | 02/11/2026 | 02/11/2026 | |
| 3 | - Write the API contract for each service so the frontend and the other services know what to expect. <br> - Define the event names and what each event carries. | 03/11/2026 | 03/11/2026 | |
| 4 | - Draw the database schema for each service and write it down in one DBML file. <br> - Review the schemas with each teammate. | 04/11/2026 | 04/11/2026 | [DBML](https://dbml.dbdiagram.io/docs/) |
| 5 | - Write the Docker Compose setup: PostgreSQL per service, Valkey, and LocalStack to fake SNS and SQS. <br> - Help teammates get their service to start in a container. | 05/11/2026 | 05/11/2026 | [Docker Compose](https://docs.docker.com/compose/) <br> [LocalStack](https://docs.localstack.cloud/) |
| 6 | - Review the first pull requests. <br> - Run the whole system locally for the first time and note what broke. | 06/11/2026 | 06/11/2026 | |

### Week 3 Achievements:

&emsp;◉ Monday:
<br>&emsp;&emsp;○ Simple branch rules and small pull requests. It saved us a lot of merge pain later.

&emsp;◉ Tuesday:
<br>&emsp;&emsp;○ The contracts were the most useful document of the week, because the teammates could build without waiting for each other.

&emsp;◉ Wednesday:
<br>&emsp;&emsp;○ One schema file for every database. Reviewing them one by one with each person took a long day, but it caught two services that wanted to read each other's tables.

&emsp;◉ Thursday:
<br>&emsp;&emsp;○ LocalStack let us test the event flow without spending a cent on AWS.

&emsp;◉ Friday:
<br>&emsp;&emsp;○ First full local run. Three services failed to start because of wrong environment variables. We fixed them before lunch and wrote the names down for the guides.