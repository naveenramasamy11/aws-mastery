# ☁️ EventBridge Rules, Pipes & Event-Driven Lambda — AWS Mastery

> **The difference between a pile of Lambda functions and an actual event-driven architecture is EventBridge.**

---

## 📖 Concept

A common anti-pattern in early serverless migrations is Lambda-to-Lambda direct invocation — Function A calls Function B synchronously, which calls Function C, and now you have a tightly-coupled call chain wearing a serverless costume. Amazon EventBridge exists to break that coupling by giving you a real event bus: producers publish events without knowing who (if anyone) consumes them, and consumers subscribe via rules that pattern-match on event content, without knowing who produced them. This is the architectural shift from "my code calls your code" to "something happened, and anyone who cares can react."

EventBridge rules match on event source, detail-type, and arbitrary fields inside the event's JSON payload using content-based filtering — you can match "any S3 object created event where the key starts with `incoming/` and the size is over 10MB" without writing a single line of filtering code; the pattern-matching happens in the EventBridge service itself. Rules can fan out to multiple targets (Lambda, Step Functions, SQS, SNS, Kinesis, API destinations for third-party webhooks) with independent retry policies and dead-letter queues per target, so one failing consumer doesn't block others.

EventBridge Pipes, the newer addition, solves a narrower but very common problem: point-to-point event routing with built-in transformation and enrichment, without needing a Lambda function purely to reshape data between a source (like a DynamoDB stream or SQS queue) and a target. A Pipe can filter, transform via a lightweight JMESPath expression, and optionally enrich the payload via a Lambda or Step Functions call, then deliver directly to the target — cutting out an entire "glue Lambda" that used to exist only to reshape JSON.

For migration and modernization work, this pairing is the backbone of decoupling monolithic on-prem batch jobs into event-driven serverless pipelines: a file lands in S3, EventBridge fires an event, a Pipe filters and enriches it, and it lands in exactly the downstream services that need it — with no custom orchestration code to maintain.

---

## 🏗️ Architecture Snapshot

```
┌────────────────────────────────────────────────────────────────────┐
│  Event-Driven Pipeline                                          │
│                                                                    │
│  S3 ObjectCreated ──▶ EventBridge Bus ──▶ Rule (pattern match)     │
│                                              │                     │
│                    ┌────────────────────────────┼──────────────┐     │
│                    ▼                         ▼              ▼     │
│              Lambda (process)          SQS (buffer)    SNS (alert)│
│                                                                    │
│  DynamoDB Stream ──▶ EventBridge Pipe ──▶ filter/enrich ──▶ Step   │
│                       (no glue Lambda)                  Functions │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Decoupled order processing:** An "OrderPlaced" event fans out independently to inventory reservation (Lambda), customer notification (SNS), and analytics ingestion (Kinesis) — adding a fourth consumer later requires zero changes to the producer.
- **Legacy batch job decomposition:** A monolithic nightly batch job gets broken into discrete steps triggered by EventBridge rules reacting to S3 file arrivals, replacing a fragile cron-based sequence with independently observable, retryable stages.
- **Cross-account event routing:** A central Security account's EventBridge bus receives GuardDuty findings from every member account and routes critical findings to a Slack webhook via API destinations.

---

## 🔧 AWS CLI & Console Examples

### Create an EventBridge rule with content-based filtering

```bash
aws events put-rule \
  --name large-incoming-files \
  --event-pattern '{"source":["aws.s3"],"detail-type":["Object Created"],"detail":{"bucket":{"name":["incoming-data"]},"object":{"size":[{"numeric":[">",10485760]}]}}}' \
  --state ENABLED

# Expected output:
# { "RuleArn": "arn:aws:events:ap-south-1:111122223333:rule/large-incoming-files" }
```

### Add a Lambda target with a dead-letter queue

```bash
aws events put-targets \
  --rule large-incoming-files \
  --targets '[{"Id":"process-large-file","Arn":"arn:aws:lambda:ap-south-1:111122223333:function:process-large-file","DeadLetterConfig":{"Arn":"arn:aws:sqs:ap-south-1:111122223333:eventbridge-dlq"},"RetryPolicy":{"MaximumRetryAttempts":2}}]'
```

### Create an EventBridge Pipe (DynamoDB Stream → filter → Step Functions)

```bash
aws pipes create-pipe \
  --name orders-to-fulfillment \
  --source arn:aws:dynamodb:ap-south-1:111122223333:table/orders/stream/2026-01-01T00:00:00.000 \
  --target arn:aws:states:ap-south-1:111122223333:stateMachine:fulfillment \
  --role-arn arn:aws:iam::111122223333:role/PipeExecutionRole \
  --source-parameters '{"DynamoDBStreamParameters":{"StartingPosition":"LATEST"},"FilterCriteria":{"Filters":[{"Pattern":"{\"eventName\":[\"INSERT\"]}"}]}}'
```

### Terraform — EventBridge rule with SQS target

```hcl
resource "aws_cloudwatch_event_rule" "order_events" {
  name           = "order-placed-events"
  event_bus_name = "default"
  event_pattern  = jsonencode({
    "source"      = ["custom.orders"]
    "detail-type" = ["OrderPlaced"]
  })
}

resource "aws_cloudwatch_event_target" "to_sqs" {
  rule = aws_cloudwatch_event_rule.order_events.name
  arn  = aws_sqs_queue.order_queue.arn
}
```

---

## 🔐 Security Best Practices

- **Scope resource-based policies on the event bus tightly for cross-account event routing:** A bus policy allowing `events:PutEvents` from `*` accounts is an easy way to accidentally accept forged events from anywhere.
- **Always attach a dead-letter queue to every rule target:** Without one, a permanently failing target silently drops events after retries exhaust, with zero visibility.
- **Use resource policies, not IAM user credentials, for API destinations calling third-party webhooks:** Store the webhook auth token in Secrets Manager and reference it via the API destination's connection, never hardcoded in a Lambda environment variable.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# See every event pattern currently listening on your default bus
aws events list-rules --query 'Rules[].{Name:Name,State:State}'

# Send a test event and watch which rules fire
aws events put-events --entries '[{"Source":"custom.test","DetailType":"TestEvent","Detail":"{\"hello\":\"world\"}"}]'
# If nothing reacts, either your pattern is wrong or nobody's listening —
# EventBridge won't complain either way, which is exactly the gotcha below.
```

---

## ⚠️ Gotchas & Tricky Bits

- **EventBridge does not error on unmatched events:** If a rule pattern doesn't match, the event is simply dropped with no error — always test patterns with `test-event-pattern` before deploying.
- **Content-based filtering has depth limits:** Deeply nested JSON detail fields beyond a certain depth aren't matchable — flatten critical filter fields closer to the top of the payload.
- **Pipes enrichment adds latency, not just transformation:** An enrichment Lambda call on every single event in a high-throughput pipe can become the bottleneck — batch or sample where exact enrichment isn't required.
- **Pro Tip:** Use `aws events test-event-pattern` locally against sample event JSON before deploying a rule — it catches malformed patterns before they silently drop production events.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → EventBridge → Rules → Create rule`
2. **Look for:** The "Event pattern" builder — it has a live sample-event matcher.
3. **Key field:** `Sample event` — paste a real event from CloudTrail/S3/etc. to validate your pattern against actual production data, not a guess.
4. **Common mistake here:** Forgetting to add a dead-letter queue on the target — silent event loss is the #1 EventBridge debugging headache.
5. **Confirm with CLI:**
   ```bash
   aws events describe-rule --name large-incoming-files --query '{State:State,Pattern:EventPattern}'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Step Functions | Common EventBridge target for orchestrating multi-step workflows triggered by events |
| Amazon SQS | Standard dead-letter queue target and buffer for EventBridge rule failures |
| AWS Lambda | The most common compute target invoked by both EventBridge rules and Pipes |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
