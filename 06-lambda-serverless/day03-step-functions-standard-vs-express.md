# ☁️ Step Functions — Standard vs Express — AWS Mastery

> **Standard Workflows remember everything for a year. Express Workflows forget on purpose — and that trade-off is the entire decision.**

---

## 📖 Concept

Every serverless orchestration problem eventually needs Step Functions once a single Lambda can't cleanly express "do A, then B, then either C or D depending on the result, and retry C up to 3 times with backoff." Step Functions is AWS's state machine service, defined in Amazon States Language (ASL), that coordinates Lambda functions, container tasks, human approvals, and dozens of other AWS service integrations without you writing the orchestration logic in code.

The decision that actually matters in production is Standard vs Express. Standard Workflows are built for durability: every execution is tracked with exactly-once semantics, can run up to a year, and gives you a full execution history in the console for debugging — but at a cost model based on state transitions, which gets expensive at high volume. Express Workflows are built for throughput: at-least-once semantics, a 5-minute maximum duration, no per-transition execution history in the console (only CloudWatch Logs if you enable them), but priced per invocation and duration like Lambda — which makes them dramatically cheaper for high-frequency, short-lived workflows.

On a migration engagement moving a customer's event-processing pipeline off a legacy message broker, we used Express Workflows for the high-volume per-event transformation (millions of short executions a day, at-least-once was acceptable because the pipeline was already idempotent) and Standard Workflows for the surrounding batch-orchestration layer that needed guaranteed exactly-once semantics and human-approval steps that could pause for days. Picking the wrong one in either direction either bankrupts you on state-transition costs or silently drops durability guarantees you actually needed.

---

## 🏗️ Architecture Snapshot

```
                    ┌──────────────────────────────────┐
                    │   Amazon EventBridge          │
                    └───────────────┬───────────────┘
                                    │
              ┌──────────────────────┼──────────────────────┐
              ▼                                           ▼
  ┌─────────────────────────┐                  ┌───────────────────────┐
  │  STANDARD Workflow     │                  │  EXPRESS Workflow      │
  │  - Exactly-once        │                  │  - At-least-once       │
  │  - Up to 1 year        │                  │  - Max 5 minutes       │
  │  - Full history in     │                  │  - CloudWatch Logs     │
  │    console             │                  │    only                │
  │  - Human approval steps│                  │  - High-volume, cheap  │
  └────────────┬───────────┘                  └────────────┬───────────┘
              │                                           │
      ┌────────▼────────┐                          ┌────────▼────────┐
      │ Lambda / ECS /  │                          │ Lambda (mostly) │
      │ SNS / SQS / etc │                          │ per-event work  │
      └────────────────┘                          └────────────────┘
```

---

## 💡 Real-World Use Cases

- **Order fulfillment with human approval:** A Standard Workflow pauses at a "manager approval" task for up to days, waiting on a callback token, then resumes the state machine exactly where it left off.
- **High-volume IoT telemetry transformation:** An Express Workflow processes millions of short-lived sensor-event transformations per day at a fraction of Standard's per-transition cost.
- **Migration wave orchestration:** A Standard Workflow coordinates a multi-hour sequence of MGN replication checks, DNS cutover, and validation steps — durability and full execution history matter more than throughput here.

---

## 🔧 AWS CLI & Console Examples

### Create a Standard state machine

```bash
aws stepfunctions create-state-machine \
  --name "order-fulfillment" \
  --type STANDARD \
  --definition file://order-fulfillment.asl.json \
  --role-arn arn:aws:iam::123456789012:role/step-functions-role \
  --region ap-south-1
```

### Create an Express state machine

```bash
aws stepfunctions create-state-machine \
  --name "event-transform" \
  --type EXPRESS \
  --definition file://event-transform.asl.json \
  --role-arn arn:aws:iam::123456789012:role/step-functions-role \
  --logging-configuration '{"level":"ALL","includeExecutionData":true,"destinations":[{"cloudWatchLogsLogGroup":{"logGroupArn":"arn:aws:logs:ap-south-1:123456789012:log-group:/sfn/event-transform:*"}}]}' \
  --region ap-south-1
```

### Minimal ASL with retry and choice

```json
{
  "StartAt": "ProcessOrder",
  "States": {
    "ProcessOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:ap-south-1:123456789012:function:process-order",
      "Retry": [{ "ErrorEquals": ["States.TaskFailed"], "MaxAttempts": 3, "BackoffRate": 2.0 }],
      "Next": "IsHighValue"
    },
    "IsHighValue": {
      "Type": "Choice",
      "Choices": [{ "Variable": "$.amount", "NumericGreaterThan": 10000, "Next": "ManagerApproval" }],
      "Default": "ShipOrder"
    },
    "ManagerApproval": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke.waitForTaskToken",
      "Next": "ShipOrder"
    },
    "ShipOrder": { "Type": "Task", "Resource": "arn:aws:lambda:ap-south-1:123456789012:function:ship-order", "End": true }
  }
}
```

---

## 🔐 Security Best Practices

- **Scope the Step Functions execution role per state machine, not one shared role for all workflows** — a compromised or misconfigured workflow shouldn't inherit permissions meant for an unrelated pipeline.
- **Enable CloudWatch Logs for Express Workflows explicitly** — without it, a failed Express execution leaves almost no debuggable trace; this is the single most common post-incident regret with Express.
- **Use resource-based policies on downstream Lambda functions** to restrict which state machines can invoke them, rather than relying solely on the state machine's IAM role being correctly scoped.

---

## 😄 Funny Things to Try

```bash
# Watch a Standard execution's full history like a replay
aws stepfunctions get-execution-history \
  --execution-arn arn:aws:states:ap-south-1:123456789012:execution:order-fulfillment:abc123 \
  --query 'events[].[timestamp,type]' --output table

# Start a Standard execution and immediately ask "are we there yet?"
aws stepfunctions start-execution --state-machine-arn arn:aws:states:ap-south-1:123456789012:stateMachine:order-fulfillment \
  --input '{"amount": 500}'
```

---

## ⚠️ Gotchas & Tricky Bits

- **Express Workflows can execute a task more than once** — at-least-once semantics mean your downstream Lambda MUST be idempotent, or duplicate side effects (double charges, duplicate emails) will happen eventually.
- **Standard Workflow costs scale with state transitions, not duration** — a workflow with a long `Wait` state costs the same as a short one, but a workflow with many small states gets expensive fast; consolidate trivial steps.
- **You cannot convert a Standard state machine to Express in place** — it's a new resource type; migrating requires recreating the state machine and redirecting triggers.
- **Pro Tip:** Use Synchronous Express Workflows (`StartSyncExecution`) when a caller needs the result inline — it's the closest Step Functions gets to "just call this like a function," without needing to poll for status.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → Step Functions → State machines → Create state machine`
2. **Look for:** the "Type" selector at the top — Standard vs Express — this cannot be changed after creation.
3. **Key field:** `Logging configuration` — for Express, set this to `ALL` during development; dial back to `ERROR` only once stable to control CloudWatch Logs cost.
4. **Common mistake here:** choosing Express by default for cost, then discovering mid-incident there's no execution history to debug a failure from three days ago.
5. **Confirm with CLI:**
   ```bash
   aws stepfunctions describe-state-machine --state-machine-arn <arn>
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| EventBridge | Common trigger source that starts Step Functions executions on a schedule or event pattern |
| Lambda | The most common task type invoked from within a state machine |
| CloudWatch Logs | The only debugging surface for Express Workflow execution history |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
