# Challenge 1 — Findings

## What I asked the agent

1. What resources do I have in this account?
2. Is anything unhealthy right now?

## What the agent told me

The AWS DevOps Agent analyzed the AWS account and provided a high-level overview of the infrastructure. It identified resources across the us-east-1 and ap-south-1 regions, including networking components such as VPCs, subnets, internet gateways, security groups, route tables, and network ACLs.

The agent also discovered MemoryDB and ElastiCache configurations, App Runner auto-scaling settings, EventBridge event buses, Athena workgroups, X-Ray sampling rules, and Resource Explorer resources.

When asked about the health of the environment, the agent reported that there were no active issues. No CloudWatch alarms were in the ALARM state, and no unhealthy compute, database, caching, or networking resources were detected. The environment was assessed as healthy and operating normally.

## Evidence

* [x] Screenshot: AWS resource inventory discovered by the DevOps Agent (`screenshots/ss1.png`)
* [x] Screenshot: Environment health status reported by the DevOps Agent (`screenshots/ss2.png`)
