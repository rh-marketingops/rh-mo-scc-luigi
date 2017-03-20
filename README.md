# Leveraging Openshift and Luigi for Marketing Ops
The Marketing Operations group at Red Hat recently began moving some of our operational data pipelines into Openshift 3, and in the process relied heavily on Luigi for workflow orchestration.

# Problems

## Data Engineering for Marketing Automation
Most modern marketing automation platforms include some built-in functionality for data flow, data manipulation, and business rule implementation. However, they are not necessarily built with programmers in mind, and pass up some functionality in favor of robustness and user-friendliness. For basic data pipelines, this is an ideal solution. 


# Technology Choices
## Why Openshift?
Previously, we had used an internal implementation of Openshift v2 (called ITOS, for short) to run scripted processes for marketing data. This platform was available for all Red Hatters to run POC and low-criticality apps, and it was very easy to spin up a new Python gear when we needed to script a new process.

Recently we began moving to Openshift Container Application Platform v3 (internally, this instance is called Open Platform). OS3 is a modern platform for building and deploying Docker containers using Kubernetes. This is very exciting for us as it removes some of the limitations we previously experienced, especially around memory management, job scheduling, and available frameworks.

## Why Luigi?
The move to OS3 presented some interesting problems not present in OS2. Mainly, in OS2, an app could be treated as a static remote VM. Moving to Kubernetes required a shift in thinking, as containers may potentially be shorter-lived than our processes.

Another reason Luigi was appealing was the growing complexity of our data pipelines. We had previously designed very complex logic in Eloqua's Program Builder; when moving to Python we started with the basics, but quickly grew beyond that. This led to multiple concurrent jobs running against the bulk API and using server resources. An early solution was a simple `scheduler.sh` script which was run every minute, then evaluated the current time and running processes to determine what to run next. It worked well for a while, but when adding new processes (especially with a shared dependency management requirement), it did not scale.

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
Conductor was very appealing because of it's support for JSON workflow definitions, but lost points because the application size was too large for our Open Platform implementation (currently have an instance-specific size of 1GB per container), required each task to be built as a microservice (too large a change to make in our available migration time), and is based mainly in Java (not something our team has much experience with). Oozie similarly lost points due to being Java based.

In the end, the best long-term solution that allowed for lightweight deployment and an easy migration was Luigi.

# Application Architecture

## SCC
Our application is named SCC, the initials for the birth name of rapper Jay Z. We got 99 problems, but a batch ain't one.

## Archimate diagram

# What's next?

- Exploring uses of Spark integrations
- Moving other batch job
