# ☁️ CloudWatch Logs Insights — Query Language & Production Patterns — AWS Mastery

> **CloudWatch Logs Insights is the AWS console's best-kept secret — most engineers don't know it until 2am when something breaks.**

---

## 📖 Concept

CloudWatch Logs Insights is a fully managed, interactive log analytics service built directly into CloudWatch. You query log groups with a purpose-built query language — no infrastructure to manage, no Elasticsearch cluster to maintain, no Kibana to configure. For most AWS workloads, it replaces the need for a separate log aggregation system for operational investigations.

The query language is SQL-adjacent but optimized for log analytics. The five core commands: `fields` (select columns), `filter` (WHERE clause), `stats` (aggregations), `sort` (ORDER BY), and `limit` (LIMIT). The `parse` command extracts unstructured values from log lines using glob or regex patterns. The `|` pipe chains commands.

Logs Insights queries run across multiple log groups simultaneously — critical for microservices architectures where you need to correlate events across Lambda functions, EKS pod logs, API Gateway access logs, and RDS slow query logs in a single query.

CloudWatch Metric Filters extract metrics from log data in real time — turning log patterns into numeric metrics that can drive alarms and dashboards. Combined with Composite Alarms (alarms that evaluate other alarms), you can build sophisticated alerting: "alert only if error rate is high AND latency is high AND this isn't during a deployment window."

For EKS workloads, Container Insights provides automatic log and metric collection from pods and nodes via the CloudWatch agent DaemonSet — no custom instrumentation needed. Lambda Insights does the same for Lambda functions via a Lambda Layer.

---

## 🏗️ Architecture Snapshot

```
  CloudWatch Observability Stack
  ──────────────────────────────────────────────────────

  Data Sources:
  ┌───────────┐ ┌───────────┐ ┌──────────┐ ┌──────────┐
  │  Lambda   │ │  EKS Pods │ │   EC2    │ │   RDS    │
  │  /aws/    │ │ Container │ │  Agents  │ │  Slow    │
  │  lambda/  │ │  Insights │ │          │ │  Query   │
  └─────┬─────┘ └─────┬─────┘ └────┬─────┘ └────┬─────┘
        └─────────────┴────────────┴─────────────┘
                              │
                    ┌─────────▼──────────┐
                    │  CloudWatch Logs   │
                    │  (Log Groups)      │
                    └─────────┬──────────┘
               ┌──────────────┼──────────────┐
               ▼              ▼              ▼
        ┌─────────┐   ┌──────────┐   ┌─────────────┐
        │  Logs   │   │  Metric  │   │ Contributor  │
        │Insights │   │ Filters  │   │  Insights   │
        │(Query)  │   │(Metrics) │   │ (Top-N)     │
        └─────────┘   └────┬─────┘   └─────────────┘
                           │
                    ┌──────▼──────┐
                    │  Alarms     │
                    │ Dashboards  │
                    └─────────────┘
```

---

## 💡 Real-World Use Cases

- **Lambda error investigation:** Query `/aws/lambda/my-function` for all ERROR messages in the last hour, extract request IDs, and correlate with downstream service logs — identifying root cause in minutes vs hours of manual log grepping.
- **EKS pod crash root cause:** Container Insights stores pod logs — query `kubectl logs` equivalent directly in CloudWatch Logs Insights across all pods in a deployment simultaneously.
- **API Gateway latency spikes:** Parse API Gateway access logs to find the P95 and P99 latency per endpoint, identify the slow endpoints, and correlate with Lambda duration metrics.

---

## 🔧 CloudWatch Logs Insights Queries

### Basic Error Analysis (Lambda)

```
# Find all Lambda errors in the last hour
fields @timestamp, @message, @logStream
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50
```

### Latency Analysis (API Gateway)

```
# P50/P95/P99 latency by route
fields @timestamp, httpMethod, resourcePath, responseLatency
| filter status >= 200
| stats
    pct(responseLatency, 50) as p50,
    pct(responseLatency, 95) as p95,
    pct(responseLatency, 99) as p99,
    count(*) as requests
  by concat(httpMethod, " ", resourcePath)
| sort p99 desc
```

### Top Error Sources (EKS Container Insights)

```
# Top pods by error log count
fields @timestamp, @logStream, @message
| filter @message like /Exception|Error|FATAL/
| parse @logStream "*/*/*" as cluster, namespace, podName
| stats count(*) as errorCount by podName, namespace
| sort errorCount desc
| limit 20
```

### VPC Flow Logs — Top Rejected Traffic Sources

```
# Top IPs generating rejected inbound traffic
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| filter dstPort in [22, 3389, 443, 80]
| stats count(*) as rejectCount by srcAddr, dstPort
| sort rejectCount desc
| limit 25
```

### CLI — Running Queries Programmatically

```bash
# Start a Logs Insights query
QUERY_ID=$(aws logs start-query \
  --log-group-name /aws/lambda/my-function \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | limit 20' \
  --region ap-south-1 \
  --query queryId --output text)

echo "Query ID: $QUERY_ID"

# Wait and get results
sleep 5
aws logs get-query-results \
  --query-id $QUERY_ID \
  --region ap-south-1 \
  --query 'results[][*].{field:field,value:value}'
```

### Creating a Metric Filter

```bash
# Create metric filter to count 5XX errors in API Gateway logs
aws logs put-metric-filter \
  --log-group-name /aws/apigateway/access-logs \
  --filter-name api-5xx-errors \
  --filter-pattern '[ip, identity, user, timestamp, request, status=5*, bytes, ...]' \
  --metric-transformations \
    metricName=API5xxErrors,metricNamespace=CustomApp,metricValue=1,defaultValue=0 \
  --region ap-south-1

# Create alarm on the metric
aws cloudwatch put-metric-alarm \
  --alarm-name api-high-error-rate \
  --metric-name API5xxErrors \
  --namespace CustomApp \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:alerts-topic \
  --region ap-south-1
```

---

## 🔐 Security Best Practices

- **Log group encryption with KMS:** By default, CloudWatch Logs are encrypted with AWS-managed keys. For sensitive logs (audit trails, API access logs), use customer-managed KMS keys: `aws logs associate-kms-key --log-group-name /audit --kms-key-id arn:aws:kms:...`
- **Set log retention policies:** By default, CloudWatch log groups have INFINITE retention. This costs money and creates compliance issues. Set retention on every log group — 90 days for application logs, 1 year for security/audit logs, export to S3 for longer retention.
- **Enable CloudTrail for all regions:** CloudTrail is not enabled in all regions by default. An attacker can operate in a region with no trail. Use a multi-region trail and send to a separate, locked-down S3 bucket in your security account.
- **Use subscription filters with Firehose for SIEM integration:** Stream security-relevant logs (CloudTrail, GuardDuty, VPC Flow Logs) to Splunk, Datadog, or a SIEM via Kinesis Firehose for real-time threat detection.

---

## 😄 Funny Things to Try

```bash
# List ALL log groups in your account and their retention settings
aws logs describe-log-groups \
  --query 'logGroups[].{Name:logGroupName,RetentionDays:retentionInDays,SizeGB:storedBytes}' \
  --output table --region ap-south-1 | head -50
# Spoiler: half of them say "None" for retention. That's infinite.
# Your bill has noticed even if you haven't.

# Find the log groups consuming the most storage
aws logs describe-log-groups \
  --query 'logGroups[].{Name:logGroupName,StoredGB:storedBytes}' \
  --output json --region ap-south-1 | \
  python3 -c "import json,sys; groups=json.load(sys.stdin); \
  sorted_groups=sorted(groups, key=lambda x: x.get('StoredGB',0), reverse=True); \
  [print(f\"{g.get('StoredGB',0)/1e9:.2f} GB  {g['Name']}\") for g in sorted_groups[:10]]"
# The "who's been logging everything?" champion will reveal itself.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Logs Insights queries have a 15-minute timeout:** Large queries across huge log groups can time out. Use time range filters and narrow your query scope. For long-term analysis, export to S3 and use Athena.
- **Metric Filters only apply to NEW log events after creation:** They don't backfill historical data. If you create a metric filter today, it only counts errors from now on.
- **Container Insights requires explicit enablement on EKS:** It's not on by default. Enable via: `aws eks update-addon --addon-name amazon-cloudwatch-observability`. Without it, you get no pod-level metrics or logs in CloudWatch.
- **CloudWatch Logs pricing is per GB ingested AND stored:** A verbose logging strategy (DEBUG level in production) can generate 100x the log volume of INFO-only logging. Set production log levels carefully and use sampling for high-volume endpoints.
- **Pro Tip:** Use CloudWatch Logs Insights saved queries. Store your most-used investigation queries (error rates, latency distributions, top error sources) and share them with your team. Navigate to `Logs Insights → History → Save query`.

---

## 📸 Console Walkthrough

> *Running a Logs Insights query to find Lambda errors with request IDs*

1. **Navigate to:** `AWS Console → CloudWatch → Logs Insights`
2. **Select log group:** Click `Select log group(s)` → search for `/aws/lambda/` → select your function
3. **Time range:** Set to `Last 1 hour` (or custom range for incident investigation)
4. **Enter query:**
   ```
   fields @timestamp, @requestId, @message
   | filter @message like /ERROR/
   | sort @timestamp desc
   | limit 100
   ```
5. **Click:** `Run query`
6. **Key column:** `@requestId` — copy this to correlate with API Gateway logs or downstream service logs
7. **Common mistake here:** Forgetting to select the right time range — Logs Insights defaults to 1 hour and your incident may have been last night
8. **Export results:** Click `Actions → Export results to CSV` for incident reports

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| CloudTrail | API audit logs sent to CloudWatch Logs for querying with Logs Insights |
| X-Ray | Distributed tracing complement to logs — use together for full observability |
| Container Insights | EKS/ECS metrics and logs automatically pushed to CloudWatch |
| Lambda Insights | Enhanced Lambda metrics (memory, CPU, init duration) via a Layer |
| Kinesis Firehose | Stream logs from CloudWatch to S3, Splunk, or Datadog via subscription filters |
| SNS / PagerDuty | CloudWatch Alarms trigger SNS topics for alerting and incident management |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
