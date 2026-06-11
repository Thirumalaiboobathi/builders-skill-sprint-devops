# Challenge 5 — Findings
**The Silent Pipeline: A Broken Agentic AI Workflow**

---

## Architecture

I built a **five-service agentic AI pipeline** — a real-world GenAI backend pattern where an EventBridge schedule pumps work into a queue, a Lambda calls Amazon Bedrock to summarise documents, and Step Functions orchestrates the post-processing before writing results to DynamoDB.

```
EventBridge (rate: 2 min)
       │  injects document summarisation task
       ▼
SQS Work Queue ──────────────────────────── Dead Letter Queue
       │  triggers on message arrival              ▲
       ▼                                           │ (on failure)
Lambda: challenge5-pipeline-processor             │
       │  calls Amazon Bedrock Nova Lite           │
       │  for document summarisation               │
       ▼                                           │
Step Functions: challenge5-orchestrator ──────────┘
       │
       ├─ State 1: ValidateInput  (Pass)
       ├─ State 2: EnrichMetadata (Pass)
       └─ State 3: StoreResult    (Task → DynamoDB PutItem)
                         │
                         ▼
              DynamoDB: challenge5-agent-results
```

**What makes this scenario unique:** Five AWS services chained together with three injected faults — one per layer. Each fault masks the one below it. No single alarm identifies the full problem. The DevOps Agent has to trace the entire chain autonomously.

---

## What I Built and How I Broke It

Three faults were injected at design time across three different service layers:

| # | Fault Injected | Service | How It Was Broken |
|---|---------------|---------|-------------------|
| 1 | `MessageRetentionPeriod: 60s` | SQS | Messages expire before Lambda can reliably process them, accumulating in the DLQ |
| 2 | `bedrock:InvokeModel` absent from Lambda IAM role | Lambda IAM | Lambda gets `AccessDeniedException` every time it tries to invoke Bedrock |
| 3 | `dynamodb:PutItem` absent from Step Functions IAM role | Step Functions IAM | Every execution fails at the `StoreResult` state — DynamoDB never receives data |

All three were injected via CloudFormation — the IAM roles were deliberately created with incomplete policies, and the SQS queue with an aggressive retention window.

A CloudWatch alarm `challenge5-dlq-messages-high` was configured to fire when the DLQ depth reaches 1 — giving the DevOps Agent a visible entry point to start its investigation.

---

## Challenges Faced During Deployment

Getting the stack to deploy cleanly was non-trivial. Two blockers were hit before the stack stabilised:

**Blocker 1 — AWS API validation on SQS VisibilityTimeout**

The original Fault 1 was a `VisibilityTimeout: 3s` on the SQS queue (Lambda timeout was 30s). AWS has since added hard API-level validation that **blocks** creating a Lambda Event Source Mapping when queue visibility timeout is less than the function timeout. The stack rolled back with:

```
Resource handler returned message: "Invalid request provided: Queue visibility timeout:
3 seconds is less than Function timeout: 30 seconds"
```

The fault was redesigned to use `MessageRetentionPeriod: 60s` instead — equally valid as a real-world misconfiguration, and not blocked by the API.

**Blocker 2 — Circular IAM dependency in CloudFormation**

The original template had `LambdaRole` referencing `OrchestratorStateMachine.Arn` in its inline policy, while the state machine depended on `StepFunctionsRole`. CloudFormation could not resolve the dependency order, causing `CREATE_FAILED` on the `SQSTrigger` resource. Fixed by restructuring resource declaration order with explicit `DependsOn` chains:

```
ResultTable → DeadLetterQueue → WorkQueue → StepFunctionsRole
→ OrchestratorStateMachine → LambdaRole → PipelineProcessor → SQSTrigger
```

**Blocker 3 — Deprecated Bedrock model**

The original template used `amazon.titan-text-express-v1`. By the time the stack deployed, AWS had retired this model. The Lambda was getting:

```
ResourceNotFoundException: This model version has reached the end of its life.
```

This was actually discovered by the DevOps Agent during its investigation — not caught during design. The model was updated to `amazon.nova-lite-v1:0` with the correct Nova request schema.

---

## What the Agent Found

### Initial Prompt

```
My agentic AI pipeline is completely silent. I have an EventBridge rule firing every
2 minutes that pushes messages into an SQS queue called challenge5-work-queue, which
triggers a Lambda called challenge5-pipeline-processor. The Lambda is supposed to call
Bedrock and then kick off a Step Functions state machine called challenge5-orchestrator,
which writes results to a DynamoDB table called challenge5-agent-results. But nothing
ever lands in DynamoDB. The CloudWatch alarm challenge5-dlq-messages-high is red.
Investigate the full pipeline — SQS, Lambda, Step Functions, DynamoDB — and tell me
every root cause you find.
```

### Agent Investigation — Root Causes Identified

The agent autonomously traced the full pipeline and surfaced **four root causes** in a single investigation pass:

---

**Root Cause 1 — Deprecated Bedrock Model (Unplanned — Agent Discovered)**

The agent pulled CloudWatch Logs for `challenge5-pipeline-processor` and found:

```
Bedrock error: An error occurred (ResourceNotFoundException) when calling the
InvokeModel operation: This model version has reached the end of its life.
```

Both recent Lambda invocations had failed with this error. The pipeline was dying at the very first step — Step Functions was never triggered. This fault was not injected intentionally — the Titan model was retired by AWS after the template was written. The agent caught it before any human noticed.

---

**Root Cause 2 — Lambda Missing `bedrock:InvokeModel` Permission (Injected Fault 2)**

Even with the correct model, the Lambda would fail. The agent inspected `challenge5-lambda-role` and confirmed the `PipelineAccess` inline policy only contained:

```
sqs:ReceiveMessage, sqs:DeleteMessage, sqs:GetQueueAttributes
states:StartExecution
```

No `bedrock:InvokeModel`. The agent identified this as a secondary blocker that would surface immediately after fixing the model ID.

---

**Root Cause 3 — Step Functions Missing `dynamodb:PutItem` (Injected Fault 3)**

The agent examined `challenge5-sfn-role` and found only a `LogsOnly` policy with CloudWatch permissions. No DynamoDB write permission. The `StoreResult` state in the orchestrator would fail with `AccessDeniedException` for every execution — even if Lambda succeeded.

The agent noted: *"The Step Functions state machine has 0 executions in its history — it's correctly configured but has never been invoked because Lambda fails first."*

---

**Root Cause 4 — Aggressive SQS Message Retention (Injected Fault 1)**

The agent flagged `MessageRetentionPeriod: 120 seconds` as operationally dangerous:

> *"Setting matches your EventBridge rate — messages could expire before debugging or retry processing completes. Consider increasing to 1800s for operational safety."*

---

### Agent's Fix Priority Checklist

| Priority | Action |
|----------|--------|
| 1 | Update Lambda code to use an active Bedrock model |
| 2 | Add `bedrock:InvokeModel` to `challenge5-lambda-role` |
| 3 | Add `dynamodb:PutItem` to `challenge5-sfn-role` |
| 4 | Increase SQS retention period |

---

## Fixes Applied

### Fix 1 — Updated Bedrock Model + Request Schema

Lambda code updated from deprecated `amazon.titan-text-express-v1` to `amazon.nova-lite-v1:0` with the correct Nova request format:

```python
resp = bedrock.invoke_model(
    modelId="amazon.nova-lite-v1:0",
    contentType="application/json",
    accept="application/json",
    body=json.dumps({
        "messages": [
            {
                "role": "user",
                "content": [{"text": f"Summarise: {payload.get('message', 'hello')}"}]
            }
        ]
    })
)
summary = json.loads(resp["body"].read())["output"]["message"]["content"][0]["text"]
```

Note: Two schema iterations were needed. Nova Lite uses `{"text": "..."}` content blocks — not `{"type": "text", "text": "..."}` as Anthropic models use, and not `results[0].outputText` as Titan used.

### Fix 2 — Lambda IAM: Added `bedrock:InvokeModel`

IAM → Roles → `challenge5-lambda-role` → Added inline policy `BedrockInvokeAccess`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "bedrock:InvokeModel",
    "Resource": "*"
  }]
}
```

### Fix 3 — Step Functions IAM: Added `dynamodb:PutItem`

IAM → Roles → `challenge5-sfn-role` → Added inline policy `DynamoDBWriteAccess`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "dynamodb:PutItem",
    "Resource": "arn:aws:dynamodb:us-east-1:675613597178:table/challenge5-agent-results"
  }]
}
```

### Fix 4 — SQS Retention Period

SQS → `challenge5-work-queue` → Edit → Message retention period updated from `60s` to `300s`.

---

## Recovery Verification

After all fixes, a test message was injected manually:

```bash
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/675613597178/challenge5-work-queue \
  --message-body '{"message": "Final pipeline test after all fixes."}' \
  --region us-east-1
```

DynamoDB scan confirmed **2 items written successfully** — including a full AI-generated healthcare document summary produced by Amazon Nova Lite via Bedrock:

```json
{
  "Items": [
    {
      "summary": { "S": "\"Test\" is a term used to evaluate the performance, quality..." },
      "run_id": { "S": "a0c6b9a6-7ae2-422d-9980-78bd09087486" },
      "source": { "S": "challenge5-pipeline" }
    },
    {
      "summary": { "S": "To summarize the task of analyzing a healthcare report..." },
      "run_id": { "S": "2a674c69-350c-43c4-8272-6e4467297dde" },
      "source": { "S": "challenge5-pipeline" }
    }
  ],
  "Count": 2
}
```

Full pipeline confirmed healthy: `EventBridge ✅ → SQS ✅ → Lambda ✅ → Bedrock ✅ → Step Functions ✅ → DynamoDB ✅`

---

## Evidence

- [x] `screenshots/01-agent-investigation-root-causes.png` — Agent chat showing all 4 root causes
- [x] `screenshots/02-dlq-alarm-red.png` — CloudWatch alarm in ALARM state
- [x] `screenshots/03-lambda-iam-bedrock-fix.png` — BedrockInvokeAccess policy added
- [x] `screenshots/04-sfn-iam-dynamodb-fix.png` — DynamoDBWriteAccess policy added
- [x] `screenshots/05-lambda-test-succeeded.png` — Lambda test execution succeeded
- [x] `screenshots/06-dynamodb-scan-recovery.png` — DynamoDB showing 2 items with AI summaries
- [x] `screenshots/07-agent-final-health-check.png` — Agent confirms pipeline healthy

---

## Bonus: Runbook (Agent Skill)

The following runbook was added to the DevOps Agent as a **Skill** in the `bss-may-2026` Agent Space, enabling the agent to follow structured investigation steps for any future agentic pipeline failure:

```markdown
# Runbook: Agentic AI Pipeline Investigation

## When to use
Fire this runbook when: DLQ alarm is red, DynamoDB table is empty,
or Step Functions executions are failing.

## Step 1 — Check SQS health
- Verify MessageRetentionPeriod is sufficient for your processing time (minimum 5x Lambda timeout).
- Check DLQ depth — if > 0, messages are failing processing.
- Verify VisibilityTimeout ≥ Lambda timeout + 5s.

## Step 2 — Check Lambda errors
- Pull CloudWatch Logs for the processor Lambda.
- Look for: ResourceNotFoundException (EOL model), AccessDeniedException
  (missing IAM), ValidationException (wrong request schema).
- If ResourceNotFoundException on InvokeModel → model is deprecated, update model ID.
- If AccessDeniedException on bedrock:InvokeModel → fix Lambda IAM role.

## Step 3 — Check Step Functions executions
- List recent executions. If 0 executions → Lambda is failing before StartExecution.
- If FAILED at StoreResult → check SFN IAM role for dynamodb:PutItem.

## Step 4 — Verify end-to-end
- Send one test message manually to SQS.
- Wait 60s → check DynamoDB for new item.
- If item present → pipeline is healthy.
```

---

## Key Takeaway

This challenge demonstrated something that goes beyond typical DevOps troubleshooting: **the AWS DevOps Agent can reason across service boundaries autonomously.** It didn't just find the alarm that was firing — it traced the full chain from SQS through Lambda logs through IAM policies through Step Functions execution history, and even surfaced a real production issue (deprecated Bedrock model) that wasn't part of the original design.

The agent identified 4 root causes in a single investigation pass, provided a prioritised fix checklist, and confirmed recovery end-to-end. That's the value of agentic SRE.

---

*Challenge 5 — AWS User Group Madurai | Builders Skill Sprint May 2026 | DevOps Month*