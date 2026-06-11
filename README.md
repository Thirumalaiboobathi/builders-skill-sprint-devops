# AWS DevOps Agent Sprint 2026

### Incident Response, Root Cause Analysis & Agentic Operations on AWS

<p align="center">
  <img src="https://img.shields.io/badge/AWS-Cloud-orange" />
  <img src="https://img.shields.io/badge/DevOps-Agentic%20Operations-blue" />
  <img src="https://img.shields.io/badge/GenAI-Bedrock-green" />
  <img src="https://img.shields.io/badge/Status-Completed-success" />
</p>

---

## Executive Summary

This repository documents my completion of the **AWS User Group Madurai – Builders Skill Sprint (DevOps Month 2026)**.

The sprint focused on using the **AWS DevOps Agent** to investigate, diagnose, remediate, and validate failures across modern cloud-native and AI-powered architectures.

Across five progressively difficult challenges, I performed:

* Infrastructure discovery
* Incident investigation
* Root cause analysis
* IAM troubleshooting
* Lambda debugging
* EC2 performance diagnostics
* CloudWatch alarm analysis
* SQS and DLQ troubleshooting
* Step Functions debugging
* Amazon Bedrock integration recovery
* End-to-end AI pipeline remediation

---

# Sprint Challenges

| Challenge   | Scenario            | AWS Services                                                | Difficulty | Status      |
| ----------- | ------------------- | ----------------------------------------------------------- | ---------- | ----------- |
| Challenge 1 | Meet Your Agent     | DevOps Agent                                                | ⭐          | ✅ Completed |
| Challenge 2 | First Investigation | Lambda, CloudWatch                                          | ⭐⭐         | ✅ Completed |
| Challenge 3 | Stress & Diagnose   | EC2, SSM, CloudWatch                                        | ⭐⭐⭐        | ✅ Completed |
| Challenge 4 | Broken Application  | Lambda, DynamoDB, IAM                                       | ⭐⭐⭐        | ✅ Completed |
| Challenge 5 | Silent AI Pipeline  | EventBridge, SQS, Lambda, Bedrock, Step Functions, DynamoDB | ⭐⭐⭐⭐⭐      | ✅ Completed |

---

# Skills Demonstrated

| Category               | Technologies                  |
| ---------------------- | ----------------------------- |
| Cloud Platform         | AWS                           |
| Observability          | CloudWatch, CloudWatch Alarms |
| Compute                | Lambda, EC2                   |
| Messaging              | SQS, DLQ                      |
| Workflow Orchestration | Step Functions                |
| Databases              | DynamoDB                      |
| AI Services            | Amazon Bedrock                |
| Event Processing       | EventBridge                   |
| Operations             | Systems Manager               |
| Security               | IAM Policies, Roles           |
| Infrastructure as Code | CloudFormation                |
| Investigation          | AWS DevOps Agent              |

---

# Challenge Highlights

## Challenge 1 – Meet Your Agent

### Objective

Validate AWS DevOps Agent connectivity and infrastructure visibility.

### Activities

| Activity                 | Outcome    |
| ------------------------ | ---------- |
| Resource Discovery       | Successful |
| Environment Health Check | Successful |
| Infrastructure Summary   | Generated  |

### Result

The DevOps Agent successfully analyzed the AWS environment, identified deployed resources, and provided a human-readable health summary.

---

## Challenge 2 – First Investigation

### Scenario

A newly deployed Lambda function failed on every invocation.

### Root Cause

| Component  | Issue                       |
| ---------- | --------------------------- |
| Lambda     | Undefined variable `config` |
| Error Type | NameError                   |

### Resolution

```python
def handler(event, context):
    config = {"value": "Challenge 2 Fixed"}
    return {"result": config["value"]}
}
```

### Outcome

✅ Lambda execution restored
✅ CloudWatch alarm recovered

---

## Challenge 3 – Stress & Diagnose

### Scenario

An EC2 instance became extremely slow and triggered a high CPU alarm.

### Root Cause

A bootstrap script launched infinite CPU-intensive loops during startup.

### Investigation Findings

| Metric          | Observation                 |
| --------------- | --------------------------- |
| CPU Utilization | ~95%                        |
| Alarm           | challenge3-high-cpu         |
| Root Process    | Multiple runaway bash loops |

### Resolution

```bash
pkill -f 'while true; do :; done'
```

### Outcome

✅ CPU returned to normal
✅ Alarm cleared
✅ Instance performance restored

---

## Challenge 4 – IAM Access Recovery

### Scenario

A Lambda application began failing after deployment even though the code was correct.

### Root Cause

Missing DynamoDB permissions on the Lambda execution role.

### Error

```text
AccessDeniedException:
dynamodb:GetItem
```

### Resolution

Added:

* dynamodb:GetItem
* dynamodb:Query
* dynamodb:Scan

permissions to the Lambda execution role.

### Outcome

✅ Application restored
✅ DynamoDB access verified

---

# Challenge 5 – The Silent Pipeline (Capstone Challenge)

## Architecture

```text
EventBridge
      │
      ▼
SQS Work Queue
      │
      ▼
Lambda Processor
      │
      ▼
Amazon Bedrock
      │
      ▼
Step Functions
      │
      ▼
DynamoDB
```

---

## Objective

Diagnose a silent multi-service AI pipeline where messages were entering the system but no results reached DynamoDB.

---

## Services Involved

| Layer        | Service        |
| ------------ | -------------- |
| Event Source | EventBridge    |
| Messaging    | SQS            |
| Compute      | Lambda         |
| AI Inference | Amazon Bedrock |
| Workflow     | Step Functions |
| Storage      | DynamoDB       |
| Monitoring   | CloudWatch     |
| Security     | IAM            |

---

## Root Causes Identified

| Priority | Root Cause                             | Impact                             |
| -------- | -------------------------------------- | ---------------------------------- |
| P1       | Deprecated Bedrock Model               | Lambda failed before orchestration |
| P1       | Missing bedrock:InvokeModel            | AccessDeniedException              |
| P2       | Missing dynamodb:PutItem               | Step Functions execution failure   |
| P3       | Aggressive SQS retention configuration | Increased risk of message loss     |

---

## Remediation Actions

| Fix                | Description                      |
| ------------------ | -------------------------------- |
| Bedrock Update     | Migrated from Titan to Nova Lite |
| Lambda IAM         | Added bedrock:InvokeModel        |
| Step Functions IAM | Added dynamodb:PutItem           |
| Queue Reliability  | Increased SQS retention period   |

---

## Validation Results

| Component        | Status    |
| ---------------- | --------- |
| EventBridge      | ✅ Healthy |
| SQS              | ✅ Healthy |
| Lambda           | ✅ Healthy |
| Bedrock          | ✅ Healthy |
| Step Functions   | ✅ Healthy |
| DynamoDB         | ✅ Healthy |
| CloudWatch Alarm | ✅ Cleared |

---

## End-to-End Recovery

The pipeline successfully processed messages through all stages and stored AI-generated summaries inside DynamoDB.

```text
EventBridge
→ SQS
→ Lambda
→ Bedrock
→ Step Functions
→ DynamoDB
```

All components were validated successfully after remediation.

---

# Repository Structure

```text
.
├── challenge-1-meet-your-agent
├── challenge-2-first-investigation
├── challenge-3-stress-and-diagnose
├── challenge-4-broken-application
├── challenge-5-silent-pipeline
│   ├── FINDINGS.md
│   ├── screenshots
│   └── template.yaml
└── README.md
```

---

# Key Takeaways

* AWS DevOps Agent can perform cross-service root cause analysis.
* Modern AI systems require observability across multiple AWS services.
* IAM misconfigurations remain one of the most common production issues.
* Agentic troubleshooting significantly reduces investigation time.
* End-to-end visibility is critical in event-driven AI architectures.

---