# ☁️ CloudWatch Alarms, Composite Alarms & Anomaly Detection — AWS Mastery

> **Moving from "my phone rang at 3 AM because CPU hit 70% for one second" to "my phone rings only when something is actually, genuinely, on fire."**

---

## 📖 Concept

### CloudWatch Alarms

A CloudWatch alarm watches a single metric over a specified time period. When the metric crosses a threshold for a number of consecutive evaluation periods, the alarm transitions: OK → ALARM → OK. Each state transition can trigger an action: SNS notification, Auto Scaling action, EC2 action (stop/start/reboot/terminate), or Systems Manager OpsCenter OpsItem.

**Key alarm configuration parameters:**
- **Period** — how long (seconds) to aggregate the metric (60s, 300s, etc.)
- **Evaluation Periods** — how many consecutive periods must breach before state changes
- **Datapoints to Alarm** — out of the evaluation periods, how many must breach (M of N evaluation)
- **Treat Missing Data** — what happens when no data is reported: `breaching`, `notBreaching`, `ignore`, `missing`

**M of N evaluation** is the most underused feature. Instead of "alarm if CPU > 80% for 3 consecutive 5-minute periods" (alarm after 15 solid minutes), use "5 of 5 datapoints" for more conservative or "1 of 5 datapoints" for immediate alerting.

### Composite Alarms

Composite alarms combine multiple child alarms using boolean logic (AND, OR, NOT). The composite alarm doesn't evaluate metrics itself — it evaluates the states of other alarms. This enables alert suppression and correlation: only page on-call if the app alarm is in ALARM state AND the database alarm is also in ALARM state. If only the app alarm fires, it might be a flap — don't wake anyone.

Composite alarms also enable maintenance windows: create a "maintenance mode" alarm that's always in ALARM state during your maintenance window, then wrap your real alarms with `NOT (ALARM("maintenance-mode-alarm")) AND ALARM("real-alarm")`.

### CloudWatch Anomaly Detection

Anomaly Detection uses ML to model the expected behavior of a metric based on historical data (uses up to 2 weeks of history). It creates a band of expected values accounting for time of day, day of week, and seasonal patterns. You alarm when the metric falls outside the band — no static threshold needed.

This is powerful for metrics with natural diurnal patterns: web traffic is low at 3 AM and high at noon. A static threshold alarm on request rate will either miss nighttime incidents (threshold too high) or false-alarm during lunch peaks (threshold too low). Anomaly Detection handles both.

---

## 🏗️ Architecture Snapshot

```
Alarm Hierarchy — From Noise to Signal
──────────────────────────────────────────────────────────────────

Individual Metric Alarms (child alarms):
  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────────┐
  │ CPU-High Alarm   │  │ Memory-High Alarm│  │ DB-Latency Alarm  │
  │ > 85% for 3 min  │  │ > 90% for 5 min  │  │ > 500ms p99       │
  └────────┬─────────┘  └────────┬─────────┘  └─────────┬─────────┘
           │                     │                       │
           └────────────┬────────┘                       │
                        ▼                                │
              ┌─────────────────────┐                    │
              │ Composite: App      │                    │
              │ CPU OR Memory       │◀───────────────────┘
              │ ALARM state         │
              └──────────┬──────────┘
                         │ AND
              ┌──────────▼──────────┐
              │ Composite: AppDB    │  ← Only this fires SNS/PagerDuty
              │ App Composite AND   │
              │ DB Latency Alarm    │
              └─────────────────────┘

Anomaly Detection Band:
  Request Rate (req/s)
  │
  │   Expected band ╔═══════╗       ╔═══════════╗
  │   (2σ range)    ║░░░░░░░║       ║░░░░░░░░░░░║
  │                 ║░░░░░░░║       ║░░░░░░░░░░░║
  │   Actual ───────╫───────╫───────╫──────╳────╫─── ALARM (anomaly)
  │                 ╚═══════╝       ╚═══════════╝
  └───────────────────────────────────────────────▶ Time
              3AM        6AM       12PM
```

---

## 💡 Real-World Use Cases

- **EKS node pressure alerting:** Per-node CPU and memory alarms roll into composite alarms per node group. Node group composite rolls into cluster-level composite. PagerDuty only fires when cluster-level composite is in ALARM — eliminates transient single-node blips from waking on-call.
- **Migration validation monitoring:** During a migration wave cutover, set anomaly detection alarms on response time and error rate for the migrated service. The ML model trains on the source-system baseline; deviation post-cutover triggers immediate alerting without needing to know the "right" threshold.
- **Cost anomaly detection + alarm:** Use AWS Cost Anomaly Detection (separate from CloudWatch) to set up cost alerts, then route them through SNS to the same alerting path as infrastructure alarms. Unified view of "something broke" and "something is spending unexpectedly."

---

## 🔧 AWS CLI & Console Examples

### Create a Standard Metric Alarm

```bash
# CPU utilization alarm with M of N evaluation (3 of 5 datapoints)
aws cloudwatch put-metric-alarm \
  --alarm-name "webapp-cpu-high" \
  --alarm-description "EC2 CPU > 85% for 3 of 5 periods" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc123def456 \
  --statistic Average \
  --period 60 \
  --evaluation-periods 5 \
  --datapoints-to-alarm 3 \
  --threshold 85 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:ops-alerts \
  --ok-actions arn:aws:sns:ap-south-1:123456789012:ops-alerts \
  --region ap-south-1
```

### Create an Anomaly Detection Alarm

```bash
# First, create an anomaly detector for the metric
aws cloudwatch put-anomaly-detector \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/webapp-alb/abc123 \
  --stat Average \
  --region ap-south-1

# Create alarm based on anomaly detection band (2 standard deviations)
aws cloudwatch put-metric-alarm \
  --alarm-name "alb-response-time-anomaly" \
  --alarm-description "ALB response time outside expected range" \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=LoadBalancer,Value=app/webapp-alb/abc123 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 2 \
  --threshold-metric-id ad1 \
  --comparison-operator GreaterThanUpperThreshold \
  --metrics '[{
    "Id": "m1",
    "MetricStat": {
      "Metric": {
        "Namespace": "AWS/ApplicationELB",
        "MetricName": "TargetResponseTime",
        "Dimensions": [{"Name": "LoadBalancer", "Value": "app/webapp-alb/abc123"}]
      },
      "Period": 300,
      "Stat": "Average"
    }
  },{
    "Id": "ad1",
    "Expression": "ANOMALY_DETECTION_BAND(m1, 2)"
  }]' \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:ops-alerts \
  --region ap-south-1
```

### Create a Composite Alarm

```bash
# Composite: app is in trouble only if CPU AND memory are both high
aws cloudwatch put-composite-alarm \
  --alarm-name "webapp-composite" \
  --alarm-description "App is degraded — CPU AND memory high" \
  --alarm-rule "ALARM(\"webapp-cpu-high\") AND ALARM(\"webapp-memory-high\")" \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:ops-pagerduty \
  --region ap-south-1

# Composite with suppression: don't alarm during maintenance
aws cloudwatch put-composite-alarm \
  --alarm-name "webapp-prod-alert" \
  --alarm-description "Production alert with maintenance suppression" \
  --alarm-rule "ALARM(\"webapp-composite\") AND NOT ALARM(\"maintenance-window-alarm\")" \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:ops-pagerduty \
  --region ap-south-1
```

### Check Alarm States

```bash
# See all alarms in ALARM state
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --region ap-south-1 \
  --query 'MetricAlarms[*].[AlarmName,StateReason,StateUpdatedTimestamp]' \
  --output table

# Get alarm history for a specific alarm (for post-mortem)
aws cloudwatch describe-alarm-history \
  --alarm-name "webapp-cpu-high" \
  --history-item-type StateUpdate \
  --start-date 2024-01-01 \
  --region ap-south-1 \
  --query 'AlarmHistoryItems[*].[Timestamp,HistorySummary]' \
  --output table
```

### Terraform — Composite Alarm with Anomaly Detection

```hcl
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "webapp-cpu-high"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 5
  datapoints_to_alarm = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Average"
  threshold           = 85
  treat_missing_data  = "notBreaching"

  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.webapp.name
  }

  alarm_actions = [aws_sns_topic.ops_alerts.arn]
  ok_actions    = [aws_sns_topic.ops_alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "response_time_anomaly" {
  alarm_name          = "alb-response-time-anomaly"
  comparison_operator = "GreaterThanUpperThreshold"
  evaluation_periods  = 3
  datapoints_to_alarm = 2
  threshold_metric_id = "ad1"
  treat_missing_data  = "notBreaching"

  metric_query {
    id = "m1"
    metric {
      metric_name = "TargetResponseTime"
      namespace   = "AWS/ApplicationELB"
      period      = 300
      stat        = "Average"
      dimensions = {
        LoadBalancer = aws_lb.webapp.arn_suffix
      }
    }
  }

  metric_query {
    id          = "ad1"
    expression  = "ANOMALY_DETECTION_BAND(m1, 2)"
    label       = "TargetResponseTime (expected)"
    return_data = true
  }

  alarm_actions = [aws_sns_topic.ops_alerts.arn]
}

resource "aws_cloudwatch_composite_alarm" "webapp_prod" {
  alarm_name = "webapp-prod-composite"
  alarm_rule = "ALARM(\"${aws_cloudwatch_metric_alarm.cpu_high.alarm_name}\") OR ALARM(\"${aws_cloudwatch_metric_alarm.response_time_anomaly.alarm_name}\")"

  alarm_actions = [aws_sns_topic.ops_pagerduty.arn]
}
```

---

## 🔐 Security Best Practices

- **Use SNS with access policies on alarm topics.** CloudWatch publishes to SNS on alarm. Ensure the SNS topic policy allows `cloudwatch.amazonaws.com` as principal with `sns:Publish`. Without this, alarm actions silently fail.
- **Encrypt SNS topics with KMS.** If your alarm notifications contain sensitive metric names or instance IDs, encrypt the SNS topic. Use AWS-managed key or customer-managed key based on compliance requirements.
- **Set OK actions, not just ALARM actions.** An alarm that only notifies on ALARM state leaves on-call guessing whether the issue resolved. Always set `ok-actions` to notify recovery — gives you an automatic all-clear signal.
- **Audit alarm configurations with AWS Config.** The `cloudwatch-alarm-action-check` Config rule detects alarms with no actions configured — the silent alarm that nobody notices until an incident reveals it.

---

## 😄 Funny Things to Try

```bash
# Count how many alarms are currently in ALARM state
aws cloudwatch describe-alarms \
  --state-value ALARM \
  --region ap-south-1 \
  --query 'length(MetricAlarms)'
# If this number is > 0 during business hours, someone should be looking at it.
# If it's always > 0, the alarms are broken (misconfigured thresholds).
# "We just ignore that alarm, it's always on" = technical debt notification.

# Find alarms with NO actions configured (the silent killers)
aws cloudwatch describe-alarms \
  --region ap-south-1 \
  --query 'MetricAlarms[?AlarmActions==`[]`].[AlarmName]' \
  --output table
# An alarm with no actions is like a smoke detector with no speaker.
# It detects the fire. Nobody knows.
```

---

## ⚠️ Gotchas & Tricky Bits

- **`treat-missing-data = breaching` can cause false alarms.** If your Lambda function only runs hourly, it has no metrics for most of the hour. With `breaching`, the alarm fires during the idle period. Use `notBreaching` for sporadic metrics and `breaching` only for metrics that should always be reporting (EC2 CPU, ALB requests).
- **Composite alarms don't support actions on INSUFFICIENT_DATA.** Only OK and ALARM state transitions can trigger actions on composite alarms. Plan your notification flow accordingly.
- **Anomaly Detection needs 2 weeks of history for accuracy.** If you enable it on a new metric or a metric with irregular patterns (e.g., just launched), the band will be wide and alarms unreliable for the first 14 days. Start in ALARM-disabled mode and monitor.
- **Cross-account alarms require CloudWatch cross-account observability setup.** A composite alarm in Account A cannot directly reference a metric alarm in Account B — you need to set up cross-account observability (sharing CloudWatch data) first.
- **Pro Tip:** For EKS/container workloads, use Container Insights metrics (`EKS/ContainerInsights` namespace) for node and pod-level alarms. `node_cpu_utilization`, `pod_memory_utilization`, and `pod_number_of_container_restarts` are the three alarms every EKS cluster should have.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → CloudWatch → Alarms → Create alarm`
2. **Select metric:** Choose namespace → metric → add dimensions (e.g., EC2 InstanceId)
3. **Configure:** Set period (300s recommended for most), statistic (Average, p99 for latency), threshold
4. **Key field:** "Additional configuration" → set `Datapoints to alarm` differently from `Evaluation periods` for M of N evaluation — this is hidden under "Additional configuration" and most people miss it
5. **For Composite:** Go to `Alarms → Create alarm → Composite alarm` → write alarm rule using alarm names
6. **Common mistake here:** Writing composite alarm rules with alarm ARNs instead of alarm names — the `ALARM()` function in the rule takes the alarm **name**, not the ARN
7. **Verify alarm is working:**
   ```bash
   # Manually set alarm to ALARM state for testing
   aws cloudwatch set-alarm-state \
     --alarm-name "webapp-cpu-high" \
     --state-value ALARM \
     --state-reason "Testing alarm" \
     --region ap-south-1
   # Check if SNS delivered, then reset:
   aws cloudwatch set-alarm-state \
     --alarm-name "webapp-cpu-high" \
     --state-value OK \
     --state-reason "Test complete" \
     --region ap-south-1
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| SNS | Primary action target for CloudWatch alarms — distributes to email, Lambda, SQS, PagerDuty, Slack |
| AWS Systems Manager OpsCenter | CloudWatch alarms can create OpsItems directly — integrates alerting into the incident management workflow |
| EventBridge | Route CloudWatch alarm state changes to EventBridge for complex routing, multi-target actions, or cross-account delivery |
| AWS Chatbot | Bridges SNS → Slack/Teams — CloudWatch alarms appear as formatted messages in your ops channel |
| Auto Scaling | CloudWatch alarm threshold breaches trigger scale-out/scale-in actions on ASGs |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
