# Leveraging Openshift and Luigi for Marketing Ops
The Marketing Operations group at Red Hat recently began moving some of our operational data pipelines into Openshift 3, and in the process relied heavily on Luigi for workflow orchestration.

In this case study, we walk through our reasons for moving marketing data pipelines into scripted workflows, our technology choices, a high-level overview of the solution architecture, and a brief overview of what pipelines we have implemented (and are planning to implement).

# Data Engineering for Marketing Automation
Most modern marketing automation platforms include some built-in functionality for data flow, data manipulation, and business rule implementation. However, they are not necessarily built with programmers in mind, and pass up some functionality in favor of robustness and user-friendliness. For basic data pipelines, this is an ideal solution. As our processes grew more complex, so did our implementation requirements. We needed to become more rigorous in our testing and version control, and expand our available toolkit to match business needs.

In moving to scripted workflows, we intentionally gave up user-friendliness to gain a wider set of tools, version control, unit testing, and other things that programmers love. For Eloqua (our marketing automation platform of choice), we chose to move away from the graphical Program Builder and Program Canvas interfaces toward the robust set of REST and Bulk APIs they provide for application development.

## Example processes
- Enriching operational data with info from Salesforce
  + Custom Data Object records which quickly require data so that lead dev reps and automated nurture programs can make the appropriate decisions about a marketing contact
  + Mainly data from the Campaign object
  + Our use case required more complex filtering not currently available via Auto Syncs
- Database cleanup
  + Data enrichment/cleaning for both Eloqua and Salesforce when an email address recieves a hard bounce
  + Deleting invalid or inactive contacts to remain below our database limit
  + Restoring profile data for previously deleted contacts who "come back to life"

# Technology Choices
## Why Openshift?
Previously, we had used an internal implementation of Openshift v2 (called ITOS, for short) to run scripted processes for marketing data. This platform was available for all Red Hatters to run POC and low-criticality apps, and it was very easy to spin up a new Python gear when we needed to script a new process.

Recently we began moving to Openshift Container Application Platform v3 (internally, our instance is called Open Platform). OS3 is a modern platform for building and deploying Docker containers using Kubernetes. This is very exciting for us as it removes some of the limitations we previously experienced, especially around memory management, job scheduling, and available frameworks.

Also a bonus is the built-in Source-to-Image tool, which allowed a much easier migration for us. With S2I, our team just worries about writing scripts, not about building containers.

## Why Luigi?
The move to OS3 presented some interesting problems not present in OS2. Mainly, in OS2, an app could be treated as a static remote VM. Moving to Kubernetes required a shift in thinking, as containers may potentially be short-lived; hence, our processes needed to be atomic and idempotent.

We had previously designed very complex logic in Eloqua's Program Builder; when moving to Python we started with the basics, but quickly grew beyond that. This led to multiple concurrent jobs running against the Bulk API and using server resources. An early solution was a simple `scheduler.sh` script which was run every minute, then evaluated the current time and running processes to determine what to run next. It worked well for a while, but when adding new processes (especially with a shared dependency management requirement), it did not scale.

We evaluated a few options:
- Do nothing and continue to use `scheduler.sh`
- Luigi by Spotify
- Conductor by Netflix
- Apache Oozie

against evaluation criteria:
- Ease of migration
- Fit with current engineer skill sets
- Support for modern frameworks (i.e., Spark)
- Running application size

Obviously, given the reasons above, we couldn't keep using `scheduler.sh`.
Conductor was very appealing because of it's support for JSON workflow definitions, but lost points because the application size was too large for our Open Platform implementation (currently have an instance-specific size of 1GB per container), required each task to be built as a microservice (too large a change to make in our available migration time), and is based mainly in Java (not something our team has much experience with). Oozie similarly lost points due to being Java based (also for being primarily Hadoop-oriented, which is a move we were not immediately ready to make).

In the end, the best long-term solution that allowed for lightweight deployment and an easy migration was Luigi.

(Also a plus was the built-in contrib module for Salesforce)

# Application Architecture - SCC
Our application is named SCC, the initials for the birth name of rapper Jay Z. We got 99 problems, but a batch ain't one.

## Overview
![alt text](/SCC-Luigi-Overview.png "Overview")

## Source-to-Image (S2I) builds
As mentioned earlier, we leverage the built-in S2I tool for easier container development. The internally-hosted Docker repo and project image stream allows very easy tagging of images for deployment to production. We build all our containers using the default Python 3.5 builder, which uses pip for dependency management.

## SCC-Luigi / SCC-Luigi-DB: Central Scheduler and task history
We run the central scheduler (`luigid`) on an independent deployment/service/route. This lets multiple jobs connect to it from within the project, and exposes the web UI to internal users. The build/startup for this is merely running `pip install` to get Luigi, then a bash script to initiate the daemon.

We also run a persistent PostgreSQL pod which is used to track the (experimental) task history. Eventually we plan to move this to a separate internal database.

## Scheduled Jobs
The bulk of our dev work does not use constantly-running containers. Instead, we build images for logically-grouped processes, then launch the containers as Kubernetes Jobs. In Kubernetes, a Job is a standalone pod running an instance of a container whose job is to execute one command. In our case, it launches a bash script `run.sh` which prepares config files, runs a Luigi wrapper task (which triggers dependent tasks), and sends a heartbeat to a monitoring system. This provides a lighter development load because we don't have to worry about configuring services, routes, or deployment controllers - just the image build and job definition.

Jobs are very convenient in that, by default, they have re-try logic to ensure whatever command entered finishes without errors. Scheduled Jobs (Cron Jobs in Kubernetes >= 1.5) are individual Jobs launched by a central Kubernetes scheduler.

## Output: Persistent Volume Claims
One of the early shakeups that Kubernetes introduced to application development was the idea of stateless, cloud-native apps. Best practices for cloud apps involve keeping persistent storage of your application separate from the actual running services. In our case, to track task status and provide robustness, we needed the ability to write Luigi outputs as static files to persistent storage. Thankfully, more recent versions of Kubernetes provide support for this.

Our Openshift instance is configured to use GlusterFS for persistent volumes; in our app, we simply create a Persistent Volume Claim (PVC) which is then mounted by our Scheduled Job pods. Output is maintained there for a reasonable amount of time (to ensure the task successfully completed), then is cleaned up later by another scripted process.

## Monitoring and reporting
We use Prometheus (also running on Openshift) to collect start/end times for Jobs (via a Pushgateway). This helps us monitor whether a Job started and completed in the expected time frames.
Also, we leverage the `luigi-slack` extension to collect stats and logs on failed Luigi tasks.

# What's next?

- Exploring uses of Spark, HDFS, and other integrations
- Moving batch job components from other apps into Luigi to get closer to a microservices model, including:
  + Data Washing Machine
  + Lead Scoring
  + Integration Lead Management (create/update Salesforce leads)
  + Campaign Response and Engagement processing
