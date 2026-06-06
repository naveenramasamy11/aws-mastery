# ☁️ Lambda Cold Starts, Concurrency & Execution Model — AWS Mastery

> **Lambda cold starts are like airport security — you only notice them when you're already late.**

---

## 📖 Concept

AWS Lambda runs your code in execution environments — lightweight VMs called MicroVMs (powered by Firecracker). The first time a function is invoked (or after a period of inactivity), Lambda must download the deployment package, start the MicroVM, initialize the runtime, and run your init code outside the handler. This is a **cold start**. Subsequent invocations reuse the warm environment — just the handler runs. Cold start durations vary: ~100ms for Node.js/Python on small packages, to 1–3 seconds for Java with large JARs.

Lambda concurrency is the number of function instances running simultaneously. **Reserved concurrency** caps the maximum instances for a specific function — useful to protect a downstream database from being overwhelmed. **Provisioned concurrency** pre-initializes execution environments, eliminating cold starts entirely — but you pay for those environments even when they're idle. For latency-sensitive APIs, provisioned concurrency on a Lambda + ALB or API Gateway is the pattern.

The execution model matters for stateful vs stateless code. The `/tmp` directory (512MB–10GB) and any global variables persist across warm invocations in the same execution environment. This is both a feature (connection pooling, cached SDK clients) and a footgun (global state from previous invocations can bleed into current ones if you're not careful).

Lambda SnapStart (for Java) takes a snapshot of an initialized execution environment and restores from it for new invocations — reducing Java cold starts from 3s to under 1s. It's one of the biggest Lambda improvements in years and is underutilized.

---

## 🏗️ Architecture Snapshot

```
  Lambda Execution Model & Concurrency
  ──────────────────────────────────────────────────

  Invoke Request
       │
       ▼
  ┌────────────────────────────────────────────────┐
  │            Lambda Service                       │
  │                                                 │
  │  Is a warm execution environment available?     │
  │       │ YES                │ NO                 │
  │       ▼                    ▼                    │
  │  [WARM START]         [COLD START]              │
  │  Run handler only     Download package          │
  │  ~1-10ms overhead     Init runtime              │
  │                       Run init code             │
  │                       Run handler               │
  │                       ~100ms–3s overhead        │
  └─────────────────────────────────────────────────┘

  Concurrency Model:
  Account Limit: 1000 concurrent (default, can increase)
  ├── Function A: Reserved=200 (hard cap at 200)
  ├── Function B: Reserved=100
  └── Unreserved pool: 700 (shared across all other functions)

  Provisioned Concurrency:
  [Pre-warmed environment] ──▶ Always ready, zero cold start
  Cost: ~$0.000004646 per GB-second of provisioned time
```

---

## 💡 Real-World Use Cases

- **API backend with SLA requirements:** A payments API needs P99 latency under 500ms — provisioned concurrency of 50 on the Lambda + API Gateway caching eliminates cold starts from the critical path.
- **Event-driven microservices:** SQS → Lambda for async order processing — reserved concurrency prevents Lambda from scaling to 1000 instances and overwhelming an RDS database with connections (use RDS Proxy too).
- **Scheduled batch jobs:** EventBridge rule → Lambda every 5 minutes for data sync — cold starts don't matter, cost optimization matters — use ARM64 (Graviton2) runtime for 20% cheaper compute.

---

## 🔧 AWS CLI & Console Examples

### Checking Concurrency

```bash
# Get current concurrency settings for a function
aws lambda get-function-concurrency \
  --function-name my-api-function \
  --region ap-south-1

# Set reserved concurrency (limits max concurrent executions)
aws lambda put-function-concurrency \
  --function-name my-api-function \
  --reserved-concurrent-executions 100 \
  --region ap-south-1

# Put provisioned concurrency (eliminates cold starts)
aws lambda put-provisioned-concurrency-config \
  --function-name my-api-function \
  --qualifier prod  \
  --provisioned-concurrent-executions 20 \
  --region ap-south-1
```

### Invoking and Monitoring

```bash
# Invoke Lambda synchronously and see response
aws lambda invoke \
  --function-name my-api-function \
  --payload '{"action": "health-check"}' \
  --cli-binary-format raw-in-base64-out \
  --region ap-south-1 \
  response.json && cat response.json

# Check recent Lambda errors via CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=my-api-function \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region ap-south-1
```

### Lambda Power Tuning (ARM64 for cost)

```bash
# Update function to use ARM64 (Graviton2) — 20% cheaper, often faster
aws lambda update-function-configuration \
  --function-name my-api-function \
  --architectures arm64 \
  --region ap-south-1

# Set memory (also controls CPU allocation proportionally)
aws lambda update-function-configuration \
  --function-name my-api-function \
  --memory-size 512 \
  --timeout 30 \
  --region ap-south-1
```

---

## 🔐 Security Best Practices

- **Never hardcode credentials in Lambda environment variables:** Use IAM execution roles + Secrets Manager. Environment variables are visible in the console and in CloudTrail.
- **Set a tight timeout:** Default is 3 seconds, max is 15 minutes. Set the tightest timeout your function can handle — it limits blast radius if something goes wrong (infinite loop, hung DB call).
- **Use Lambda resource-based policies carefully:** A Lambda function's resource policy controls who can invoke it. `aws:SourceAccount` and `aws:SourceArn` conditions prevent confused deputy attacks from cross-account EventBridge or S3 triggers.
- **Enable X-Ray tracing in production:** `aws lambda update-function-configuration --tracing-config Mode=Active` — it costs pennies and gives you distributed traces across API Gateway → Lambda → DynamoDB.

---

## 😄 Funny Things to Try

```bash
# The "how many Lambdas do I have?" existential inventory
aws lambda list-functions \
  --query 'Functions[].{Name:FunctionName,Runtime:Runtime,Memory:MemorySize,LastModified:LastModified}' \
  --output table --region ap-south-1 | head -40
# Spoiler: there are always more than you think.

# Find all Lambda functions that haven't been invoked in 30 days (zombie Lambdas)
aws lambda list-functions \
  --query 'Functions[].FunctionName' --output text --region ap-south-1 | \
  tr '\t' '\n' | while read fn; do
    last=$(aws cloudwatch get-metric-statistics \
      --namespace AWS/Lambda --metric-name Invocations \
      --dimensions Name=FunctionName,Value=$fn \
      --start-time $(date -u -d '30 days ago' +%Y-%m-%dT%H:%M:%S) \
      --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
      --period 2592000 --statistics Sum \
      --query 'Datapoints[0].Sum' --output text 2>/dev/null)
    [ "$last" = "None" ] || [ "$last" = "0" ] && echo "ZOMBIE: $fn"
  done
# Every enterprise has a graveyard of forgotten Lambdas. Find yours.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Lambda + VPC = extra cold start:** When Lambda is configured in a VPC, it needs to attach an ENI — this used to add 10+ seconds to cold start. With Hyperplane ENIs (2019+), it's fixed (~500ms). But Lambda in a VPC still requires subnets with enough free IPs and the right security group rules.
- **The 15-minute timeout is a hard limit:** If your workload needs to run longer, it's not a Lambda workload. Use ECS Fargate tasks, Step Functions, or EC2.
- **Synchronous invocation retries are YOUR responsibility:** API Gateway → Lambda has no automatic retry. SQS → Lambda retries on failure (up to maxReceiveCount on the DLQ). Design error handling explicitly.
- **Lambda@Edge runs in the viewer's nearest region — not your region:** Your function code is replicated globally. It has tighter limits (5 seconds timeout for viewer requests, 30 seconds for origin requests) and cannot use VPCs or Provisioned Concurrency.
- **Pro Tip:** Use AWS Lambda Power Tuning (open source Step Functions state machine) to automatically test your function at different memory sizes and find the optimal cost/performance ratio. Often functions are set to 128MB out of habit when 512MB runs faster AND cheaper per invocation.

---

## 📸 Console Walkthrough

> *Enabling Provisioned Concurrency on a Lambda function alias*

1. **Navigate to:** `AWS Console → Lambda → [function-name] → Configuration → Concurrency`
2. **Publish a version:** Before setting provisioned concurrency, you need a published version or alias — go to `Actions → Publish new version`
3. **Create an alias:** `Aliases → Create alias → Name: prod → Version: [latest published version]`
4. **Set provisioned concurrency on the alias:** `Aliases → prod → Concurrency → Edit`
5. **Key field:** `Provisioned concurrency` — set to the number of pre-warmed environments (start with your P95 concurrent requests)
6. **Common mistake here:** Setting provisioned concurrency on `$LATEST` — it only works on published versions or aliases
7. **Monitor:** Check `ProvisionedConcurrencyUtilization` metric in CloudWatch to right-size
8. **Confirm with CLI:**
   ```bash
   aws lambda get-provisioned-concurrency-config \
     --function-name my-api-function \
     --qualifier prod \
     --query 'Status'
   # Should return "READY"
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| API Gateway | HTTP/REST/WebSocket triggers for Lambda — the most common invocation path |
| EventBridge | Scheduled and event-driven triggers — Lambda as the target |
| SQS | Async, buffered Lambda invocation with backpressure and DLQ support |
| Step Functions | Orchestrate multiple Lambdas into stateful, visual workflows |
| RDS Proxy | Connection pooling between Lambda (which scales massively) and RDS |
| X-Ray | Distributed tracing across Lambda invocations and downstream calls |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
