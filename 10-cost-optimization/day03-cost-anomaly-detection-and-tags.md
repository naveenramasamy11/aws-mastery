# ☁️ Cost Anomaly Detection & Cost Allocation Tags — AWS Mastery

> **Rightsizing tells you what you're overpaying for on purpose. Anomaly Detection tells you what you're overpaying for by accident — and accidents are usually bigger.**

---

## 📖 Concept

Most cost-optimization conversations focus on the known knobs: Savings Plans, Spot, rightsizing. But the bill events that actually cause panic — a misconfigured Lambda in an infinite retry loop, a forgotten NAT Gateway processing terabytes, a runaway data transfer bill from a misrouted replication job — aren't about picking the wrong pricing model, they're about something going quietly, expensively wrong. AWS Cost Anomaly Detection uses machine learning against your historical spend patterns per service, account, or cost-allocation-tag dimension, and alerts you when spend deviates meaningfully from what the model expects — catching the "why did our bill triple overnight" problem in hours instead of at the end of the billing cycle.

The other half of cost visibility that's easy to underinvest in is Cost Allocation Tags. Without consistent tagging (`Environment`, `Team`, `Project`, `CostCenter`), Cost Explorer and anomaly detection can tell you *what service* got expensive, but not *whose* workload it was or *which team* to route the alert to. On ProServe engagements, the single highest-leverage FinOps activity is almost never a Savings Plan purchase — it's enforcing tag policies at the Organizations level so every resource is attributable to a team from day one, because retrofitting tags onto years of untagged resources is a much bigger project than tagging correctly from the start.

Anomaly Detection monitors work at the granularity you configure: a monitor scoped to `AWS Service` catches "EC2 spend spiked," while a monitor scoped to a `Cost Allocation Tag` value catches "the `team:data-platform` tag's spend spiked," which is the more actionable signal for actually routing the alert to the right people.

---

## 🏗️ Architecture Snapshot

```
┌────────────────────────────────────────────┐
│         AWS Cost & Usage Data             │
│   (tagged by Environment/Team/Project)    │
└────────────────────┬───────────────────────────┘
                     │
        ┌────────────▼────────────┐
        │  Cost Anomaly Detection   │
        │  ML model per monitor     │
        │  dimension (service/tag)  │
        └────────────┬─────────────┘
                      │ deviation detected
        ┌─────────────▼─────────────┐
        │   SNS Topic / Email/Slack  │
        │   Alert with root-cause    │
        │   service breakdown        │
        └───────────────────────────────┘

Cost Allocation Tags flow:
Resource created → Tagged (Team=data-platform) → Cost Explorer / Anomaly
Detection can now filter, alert, and attribute spend by that tag value.
```

---

## 💡 Real-World Use Cases

- **Catching a runaway Lambda retry storm same-day:** A misconfigured DLQ causes a Lambda to retry indefinitely; Cost Anomaly Detection flags the spend spike within hours instead of at month-end reconciliation.
- **Attributing cost to the right team automatically:** With `Team` cost allocation tags enforced via SCP, a spend anomaly alert routes directly to the owning team's Slack channel instead of a generic FinOps inbox.
- **Validating migration cost assumptions:** During a migration wave, tag-scoped anomaly monitors catch unexpectedly high data-transfer costs from a newly cut-over workload before it accumulates into a quarter-ending surprise.

---

## 🔧 AWS CLI & Console Examples

### Create a cost anomaly monitor scoped to a service

```bash
aws ce create-anomaly-monitor \
  --anomaly-monitor file://monitor.json \
  --region us-east-1
```

### Create a subscription that alerts on anomalies over a threshold

```bash
aws ce create-anomaly-subscription \
  --anomaly-subscription file://subscription.json
```

### Enforce mandatory cost allocation tags via SCP

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyResourceCreationWithoutTeamTag",
      "Effect": "Deny",
      "Action": ["ec2:RunInstances", "rds:CreateDBInstance"],
      "Resource": "*",
      "Condition": {
        "Null": { "aws:RequestTag/Team": "true" }
      }
    }
  ]
}
```

---

## 🔐 Security Best Practices

- **Restrict who can modify or delete anomaly monitors/subscriptions** — treat `ce:UpdateAnomalyMonitor`/`ce:DeleteAnomalySubscription` as sensitive; a disabled monitor is a blind spot nobody notices until the bill arrives.
- **Don't rely solely on tag-based cost attribution for security-sensitive spend review** — a compromised account could still tag resources deceptively; corroborate cost anomalies with CloudTrail activity in the same time window during investigation.
- **Route cost anomaly alerts to a monitored channel, not just email** — SNS-to-Slack integration ensures alerts don't sit unread in an inbox during exactly the kind of fast-moving incident that needs same-day response.

---

## 😄 Funny Things to Try

```bash
# Get your top cost anomalies from the last 30 days, sorted by impact
aws ce get-anomalies \
  --date-interval Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --query 'Anomalies | sort_by(@, &Impact.TotalImpact) | reverse(@) | [0:5]'

# Find every untagged resource type contributing to your bill (the FinOps walk of shame)
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --metrics UnblendedCost \
  --group-by Type=TAG,Key=Team
```

---

## ⚠️ Gotchas & Tricky Bits

- **Anomaly Detection needs historical data to build a baseline** — a brand-new monitor on a brand-new account has nothing to compare against yet; expect a couple of weeks before detection becomes meaningful.
- **Cost allocation tags must be explicitly activated in Billing preferences** — tagging a resource isn't enough; the tag key itself must be activated as a cost allocation tag in the Billing console before it appears in Cost Explorer filters.
- **SCP-enforced tagging only prevents new untagged resources, not existing ones** — retrofitting tags onto pre-existing resources requires a separate remediation pass (Config rules + Lambda auto-remediation is the common pattern).
- **Pro Tip:** Anomaly thresholds set too low generate alert fatigue fast — start with a dollar-impact threshold, not a percentage, since a 200% spike on a $2/day service is noise while a 15% spike on a $50,000/month service is real money.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → Cost Management → Cost Anomaly Detection → Monitors`
2. **Look for:** the monitor type dropdown — Dimensional (by service) vs Cost Category vs Cost Allocation Tag.
3. **Key field:** `Alert threshold` on the subscription — set as an absolute dollar amount for high-value services to avoid noise from small percentage swings.
4. **Common mistake here:** creating monitors but never creating a subscription — a monitor with no subscription silently detects anomalies that nobody is ever notified about.
5. **Confirm with CLI:**
   ```bash
   aws ce get-anomaly-subscriptions --query 'AnomalySubscriptions[].SubscriptionName'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Budgets | Complementary threshold-based alerting alongside ML-based anomaly detection |
| AWS Organizations | Enforces mandatory tagging policies at the OU level via SCPs |
| AWS Config | Detects and can auto-remediate resources created without required tags |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
