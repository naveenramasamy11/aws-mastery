# ☁️ CloudWatch Synthetics Canaries & Container Insights — AWS Mastery

> **Alarms tell you something broke. Canaries tell you before your customers do.**

---

## 📖 Concept

Most CloudWatch alarms are reactive — they fire once a real user has already hit an error or a latency spike, which means by the time the alarm pages someone, actual customers already had a bad experience. CloudWatch Synthetics flips this by running scripted canaries — headless browser scripts (via Puppeteer/Playwright runtime) or simple API-check scripts — on a schedule from AWS's own infrastructure, simulating real user journeys (login, checkout, search) every one to sixty minutes, regardless of whether real traffic is currently flowing. A canary failure means "our service is broken" before a single real customer has necessarily noticed, which is the entire point: synthetic monitoring buys you lead time.

Container Insights solves the parallel problem for containerized workloads: standard CloudWatch metrics tell you node-level CPU and memory, but they don't natively understand what "a pod" or "a Kubernetes namespace" is. Container Insights (backed by the CloudWatch agent or, more recently, an OpenTelemetry-based collector) enriches metrics with Kubernetes/ECS-aware dimensions — namespace, pod, service, task — so you can actually answer "which namespace is eating all the memory on this node" instead of staring at an aggregate node metric with no way to attribute it. For an EKS migration, this is usually the single tool that turns "the cluster feels slow" into "namespace X's pod Y has a memory leak" in one dashboard.

Together, canaries and Container Insights close the loop between "did something break" (synthetic, proactive) and "why did it break, specifically" (Container Insights, diagnostic) — which is exactly the pairing a consulting engagement needs to hand a customer a genuinely operable observability stack instead of a wall of raw CloudWatch metrics nobody knows how to read.

---

## 🏗️ Architecture Snapshot

```
┌───────────────────────────────────────────────────────┐
│  CloudWatch Synthetics                                           │
│                                                                    │
│  Scheduled Canary (every 5 min)                                   │
│  ┌───────────────────┐                                            │
│  │ Headless browser     │ ──▶ Login → Add to cart → Checkout        │
│  │ script (Puppeteer)   │      (simulates real user journey)         │
│  └───────────┬───────┐                                            │
│             │                                                       │
│             ▼                                                       │
│   CloudWatch Alarm (on canary failure) ──▶ SNS ──▶ PagerDuty/Slack   │
└──────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────┐
│  Container Insights                                                │
│                                                                     │
│  EKS Node ──▶ CloudWatch Agent / OTel Collector ──▶ enriched metrics│
│                                                       (namespace,   │
│                                                        pod, service)│
│                            │                                        │
│                            ▼                                        │
│              CloudWatch Container Insights Dashboard                │
│              (per-namespace CPU/memory, pod restarts)               │
└────────────────────────────────────────────────────────────────┐
```

---

## 💡 Real-World Use Cases

- **Post-migration SLA proof:** Run a heartbeat canary hitting the login page every minute immediately after cutover, giving concrete uptime evidence for the first 30 days instead of relying on "no tickets means it's fine."
- **Third-party dependency monitoring:** A canary script that calls a partner API on a schedule catches upstream outages before they cascade into your own application's error rate.
- **EKS resource rightsizing:** Container Insights' per-namespace/per-pod memory and CPU data feeds directly into Karpenter/HPA tuning decisions, replacing guesswork with actual utilization history.

---

## 🔧 AWS CLI & Console Examples

### Create a heartbeat canary

```bash
aws synthetics create-canary \
  --name login-heartbeat \
  --code S3Bucket=canary-scripts,S3Key=login-check.zip \
  --artifact-s3-location s3://canary-artifacts/login-heartbeat \
  --execution-role-arn arn:aws:iam::111122223333:role/CanaryExecutionRole \
  --schedule Expression="rate(5 minutes)" \
  --runtime-version syn-nodejs-puppeteer-9.0 \
  --region ap-south-1
```

### Start the canary running on its schedule

```bash
aws synthetics start-canary --name login-heartbeat
```

### Enable Container Insights on an existing EKS cluster

```bash
aws eks update-cluster-config \
  --name prod-eks \
  --logging '{"clusterLogging":[{"types":["api","audit"],"enabled":true}]}'

# Container Insights itself is enabled via the CloudWatch Observability
# add-on, not a cluster config flag:
aws eks create-addon \
  --cluster-name prod-eks \
  --addon-name amazon-cloudwatch-observability \
  --region ap-south-1
```

### Query Container Insights for the top memory-consuming pods

```bash
aws cloudwatch get-metric-data \
  --metric-data-queries '[{"Id":"m1","MetricStat":{"Metric":{"Namespace":"ContainerInsights","MetricName":"pod_memory_utilization","Dimensions":[{"Name":"ClusterName","Value":"prod-eks"}]},"Period":300,"Stat":"Maximum"}}]' \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S)
```

### Terraform — Synthetics canary with alarm

```hcl
resource "aws_synthetics_canary" "login_heartbeat" {
  name                 = "login-heartbeat"
  artifact_s3_location = "s3://canary-artifacts/login-heartbeat"
  execution_role_arn   = aws_iam_role.canary_role.arn
  runtime_version      = "syn-nodejs-puppeteer-9.0"
  handler              = "index.handler"
  zip_file             = "login-check.zip"

  schedule {
    expression = "rate(5 minutes)"
  }
}

resource "aws_cloudwatch_metric_alarm" "canary_failure" {
  alarm_name          = "login-heartbeat-failing"
  namespace           = "CloudWatchSynthetics"
  metric_name         = "SuccessPercent"
  dimensions          = { CanaryName = aws_synthetics_canary.login_heartbeat.name }
  comparison_operator = "LessThanThreshold"
  threshold           = 100
  evaluation_periods  = 2
  period              = 300
  statistic           = "Average"
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

---

## 🔐 Security Best Practices

- **Scope the canary execution role to only what the test script needs:** A canary that just checks a login page needs S3 read/write for artifacts and nothing else — avoid attaching broad permissions "just in case."
- **Never hardcode test credentials in the canary script:** Pull test-account credentials from Secrets Manager at runtime so they can be rotated without redeploying the canary.
- **Restrict Container Insights CloudWatch agent IAM permissions to metrics/logs write only:** The agent doesn't need any permissions beyond `cloudwatch:PutMetricData` and log group write access.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# Ask your canary how confident it is in your own application right now
aws synthetics get-canary-runs --name login-heartbeat --max-results 5 \
  --query 'CanaryRuns[].{Status:Status.State,Time:Timeline.Started}'
# A string of PASSED entries is oddly reassuring at 2am.

# Find the single hungriest pod in your cluster right now
aws cloudwatch get-metric-data --metric-data-queries '[{"Id":"m1","MetricStat":{"Metric":{"Namespace":"ContainerInsights","MetricName":"pod_memory_utilization"},"Period":300,"Stat":"Maximum"}}]' \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --query 'MetricDataResults[0].Values | max(@)'
```

---

## ⚠️ Gotchas & Tricky Bits

- **Canary Lambda cold starts can produce false-positive failures:** A canary that times out on its first run after a code update isn't necessarily a real outage — check whether it's a cold-start artifact before paging anyone.
- **Container Insights adds real CloudWatch cost at scale:** Enhanced observability mode collects fine-grained metrics per pod, which can meaningfully increase CloudWatch billing on large clusters — right-size the collection interval.
- **Canary scripts silently break on unannounced UI changes:** A canary hardcoded to a CSS selector for the login button breaks the moment a frontend redeploy changes that selector — treat canary scripts as code that needs the same review discipline as the app itself.
- **Pro Tip:** Run canaries from multiple AWS regions when checking a global-facing endpoint — a single-region canary can pass while users in another region are experiencing a regional network issue you'd otherwise miss entirely.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → CloudWatch → Application monitoring → Synthetics Canaries → Create canary`
2. **Look for:** The blueprint selector — "Heartbeat monitoring" is the simplest starting point for a first canary.
3. **Key field:** `Schedule` — set frequency based on how quickly you need to detect an outage versus how much you want to spend; 1-minute intervals cost more but catch issues faster.
4. **Common mistake here:** Leaving the canary's VPC configuration blank when testing an internal-only endpoint — it will simply fail to connect, and the failure looks identical to a real outage.
5. **Confirm with CLI:**
   ```bash
   aws synthetics get-canary --name login-heartbeat --query 'Canary.Status'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| Amazon SNS | Delivers canary failure alarms to on-call channels (PagerDuty, Slack, email) |
| Amazon EKS | The cluster whose pod/namespace-level metrics Container Insights enriches |
| AWS X-Ray | Complements Container Insights with distributed tracing for root-causing the "why" behind a resource spike |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
