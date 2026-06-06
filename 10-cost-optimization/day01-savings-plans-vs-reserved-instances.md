# ☁️ AWS Savings Plans vs Reserved Instances — Cost Optimization Masterclass — AWS Mastery

> **Savings Plans are to Reserved Instances what a gym membership is to paying per visit — flexible, cheaper, and you still feel guilty if you don't use them.**

---

## 📖 Concept

AWS offers two primary commitment-based discount mechanisms: **Savings Plans** and **Reserved Instances (RIs)**. Both offer up to 72% discount compared to On-Demand pricing in exchange for a 1 or 3-year commitment. The key difference is flexibility: RIs commit to a specific instance type, region, OS, and tenancy. Savings Plans commit only to a spend level ($/hour).

There are three Savings Plan types: **Compute Savings Plans** (most flexible — applies to any EC2, Lambda, and Fargate usage, any region, any instance family, any OS — ~66% discount), **EC2 Instance Savings Plans** (commit to a specific instance family in a specific region — ~72% discount, but more restrictive), and **SageMaker Savings Plans** (for ML workloads only).

The right strategy depends on your workload stability: for a microservices EKS cluster where instance types change as you rightsize, Compute Savings Plans are better. For a stable, well-understood fleet that's been running the same instance type for 2+ years (SAP, Oracle, legacy apps), Standard Reserved Instances at 3 years get you the maximum discount.

Spot Instances sit in a separate category — up to 90% discount, but instances can be interrupted with 2 minutes notice. For EKS worker nodes running fault-tolerant, stateless microservices, a mixed node group (On-Demand base + Spot workers via Karpenter) is the cost-optimal pattern. A common ProServe recommendation is 20% On-Demand (base capacity, Savings Plan coverage) + 80% Spot (burstable, Karpenter-managed).

---

## 🏗️ Architecture Snapshot

```
  Cost Optimization Strategy — Coverage Layers
  ─────────────────────────────────────────────────────

  EC2 / EKS Compute Spend:
  ┌─────────────────────────────────────────────────┐
  │                                                 │
  │  ████████████████████  Compute Savings Plans    │
  │  (covers 60-70% of steady-state base usage)     │
  │                                                 │
  │  ██████████████        EC2 Ins. Savings Plans   │
  │  (covers stable, known instance families)       │
  │                                                 │
  │  ████████             Spot Instances            │
  │  (burst workloads, batch, fault-tolerant)       │
  │                                                 │
  │  ██                   On-Demand                 │
  │  (emergency headroom, unpredictable peaks)      │
  │                                                 │
  └─────────────────────────────────────────────────┘

  Savings Plan Coverage Model:
  ┌────────────────────────────────────────────────────┐
  │ Commitment: $X/hour for 1 or 3 years               │
  │                                                    │
  │ Your usage:  EC2 (any type) + Lambda + Fargate     │
  │              ←────── Savings Plan covers ────────→ │
  │                                                    │
  │ Overage:     On-Demand rates                       │
  └────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **EKS cluster cost optimization:** A payment gateway's EKS cluster runs `m7g.2xlarge` as base capacity — covered by Compute Savings Plans. Burst pods use Karpenter to provision Spot instances, achieving 55% overall cost reduction vs On-Demand.
- **Migration cost baselining:** During AWS MAP engagement, using Cost Explorer's Savings Plan recommendations to determine the right commitment level after 2 months of On-Demand baseline data — never buy SPs before you understand your usage pattern.
- **Multi-account Savings Plan sharing:** In AWS Organizations, Savings Plans purchased in the payer account automatically share across all member accounts — centralizing purchasing improves utilization rates.

---

## 🔧 AWS CLI & Console Examples

### Cost Explorer Analysis

```bash
# Get current Savings Plan coverage (what % of your usage is covered)
aws ce get-savings-plans-coverage \
  --time-period Start=2026-05-01,End=2026-06-01 \
  --granularity MONTHLY \
  --region us-east-1 \
  --query 'SavingsPlansCoverages[].{Coverage:Coverage.CoverageHours.CoverageHoursPercentage,OnDemandCost:Coverage.CoverageHours.OnDemandHours}' \
  --output table

# Get Savings Plan recommendations from AWS
aws ce get-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days THIRTY_DAYS \
  --region us-east-1 \
  --query 'Recommendations.SavingsPlansPurchaseRecommendationDetails[0].{HourlyCommitment:HourlyCommitmentToPurchase,EstimatedSavings:EstimatedSavingsAmount,SavingsPercentage:EstimatedSavingsPercentage}'

# Get Reserved Instance recommendations
aws ce get-reservation-purchase-recommendation \
  --service EC2 \
  --lookback-period-in-days THIRTY_DAYS \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --region us-east-1 \
  --query 'Recommendations[0].RecommendationDetails[0].{InstanceType:InstanceDetails.EC2InstanceDetails.InstanceType,Recommended:RecommendedNumberOfInstancesToPurchase,EstimatedBreakeven:EstimatedBreakevenInMonths}'
```

### Savings Plan Purchase and Management

```bash
# List existing Savings Plans
aws savingsplans list-savings-plans \
  --states ACTIVE \
  --query 'savingsPlans[].{ID:savingsPlanId,Type:savingsPlanType,Commitment:commitment,Start:start,End:end,Savings:savingsPlanArn}' \
  --output table

# Get utilization rate (are you using what you bought?)
aws ce get-savings-plans-utilization \
  --time-period Start=2026-05-01,End=2026-06-01 \
  --granularity MONTHLY \
  --region us-east-1 \
  --query 'SavingsPlansUtilizationsByTime[].{Utilization:Utilization.UtilizationPercentage,Savings:Savings.TotalSavings,UnusedCommitment:Unused.UnusedCommitment}'
```

### Spot Instance Cost Analysis

```bash
# Get Spot price history for m7g.2xlarge in ap-south-1
aws ec2 describe-spot-price-history \
  --instance-types m7g.2xlarge \
  --product-descriptions "Linux/UNIX" \
  --region ap-south-1 \
  --max-items 10 \
  --query 'SpotPriceHistory[].{AZ:AvailabilityZone,Price:SpotPrice,Time:Timestamp}' \
  --output table

# Compare On-Demand vs Spot savings
# On-Demand m7g.2xlarge ap-south-1: ~$0.3584/hr
# Spot (typical):                   ~$0.1075/hr (~70% savings)
echo "On-Demand monthly: $(echo '0.3584 * 720' | bc) USD"
echo "Spot monthly:      $(echo '0.1075 * 720' | bc) USD"
```

### AWS Budgets Setup

```bash
# Create a monthly cost budget with alert at 80%
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "Monthly-AWS-Spend",
    "BudgetLimit": {"Amount": "5000", "Unit": "USD"},
    "BudgetType": "COST",
    "TimeUnit": "MONTHLY",
    "TimePeriod": {"Start": "2026-06-01T00:00:00Z"}
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "naveenramasamy11@gmail.com"}]
  }]'
```

---

## 🔐 Security Best Practices

- **Centralize Savings Plans in the payer account:** Savings Plans purchased in member accounts only apply to that account. Payer account SPs share across the Org. For maximum utilization, buy in the payer account.
- **Use Cost Allocation Tags for chargeback:** Tag all resources with `Team`, `Project`, and `Environment`. Enable Cost Allocation Tags in Billing and you can break down costs per team in Cost Explorer — critical for internal chargeback.
- **Set Budgets with forecasted spend alerts:** Use both ACTUAL (what you've spent) and FORECASTED (what you're projected to spend) alerts. Forecasted alerts catch runaway costs before the bill arrives.
- **Use AWS Organizations consolidated billing:** All accounts in an Org share volume discounts (S3, data transfer tiers) and Savings Plan/RI benefits automatically — critical at enterprise scale.

---

## 😄 Funny Things to Try

```bash
# The "how much did I spend today?" anxiety check
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-%d),End=$(date -d '+1 day' +%Y-%m-%d) \
  --granularity DAILY \
  --metrics BlendedCost \
  --query 'ResultsByTime[0].Total.BlendedCost.Amount' \
  --output text \
  --region us-east-1
# The number that appears is what you've spent today. Deep breath.

# The "what's my most expensive service this month?" audit
aws ce get-cost-and-usage \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics BlendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups | sort_by(@, &Keys[0]) | reverse(@) | [0:5].{Service:Keys[0],Cost:Metrics.BlendedCost.Amount}' \
  --output table \
  --region us-east-1
# Spoiler: it's almost always EC2 + NAT Gateway data transfer.

# Find your Savings Plan utilization rate — are you getting your money's worth?
aws ce get-savings-plans-utilization \
  --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY --region us-east-1 \
  --query 'Total.Utilization.UtilizationPercentage' --output text
# Under 70%? You over-committed. Under 100%? You're leaving money on the table.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Never buy Savings Plans before establishing an On-Demand baseline:** AWS recommends at least 30 days of On-Demand data before purchasing. Buying SPs on day 1 of a new workload almost always results in over-commitment.
- **Compute Savings Plans apply in a specific priority order:** Smallest commitment first, then by region. If you have multiple SPs, AWS automatically applies the most beneficial one. You can't control which SP covers which usage.
- **3-year No-Upfront SPs don't save as much as you think:** The maximum discount comes from 3-year All-Upfront. But All-Upfront locks cash. No-Upfront 1-year Compute SPs offer a good balance of flexibility and savings (~40% discount) with annual renewal flexibility.
- **Reserved Instances for RDS don't transfer between instance families:** An RDS RI for `db.r6g.2xlarge` doesn't apply to `db.r7g.2xlarge` (Convertible RI does, with a conversion process). Plan DB generation upgrades before buying long-term RIs.
- **Pro Tip:** Set up AWS Cost Anomaly Detection (free) — it uses ML to detect unusual spending patterns and alerts you via SNS. In an enterprise with 50+ accounts, it's caught runaway Lambdas, misconfigured NAT Gateways, and forgotten dev environments many times.

---

## 📸 Console Walkthrough

> *Using Cost Explorer to get Savings Plan recommendations*

1. **Navigate to:** `AWS Console → Cost Explorer → Savings Plans → Recommendations`
2. **Configure:** Set `Savings Plans type: Compute`, `Term: 1 year`, `Payment: No upfront`, `Based on last 30 days`
3. **Review recommendations:** Shows `Estimated hourly commitment`, `Estimated monthly savings`, and `Coverage percentage`
4. **Key field:** `Estimated savings` — this is the dollar amount you'd save per month vs your current On-Demand spending
5. **Look at coverage:** If current coverage is already >80%, adding more SPs may over-commit you
6. **Common mistake here:** Buying the maximum recommended amount immediately — start at 70% of the recommendation to maintain flexibility
7. **After purchase:** Check `Savings Plans → Utilization` weekly for the first month to confirm you're using the commitment
8. **Confirm via CLI:**
   ```bash
   aws ce get-savings-plans-utilization \
     --time-period Start=YYYY-MM-01,End=YYYY-MM-DD \
     --granularity MONTHLY --region us-east-1
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Cost Explorer | Visualization, trend analysis, and SP/RI recommendations |
| AWS Budgets | Proactive alerts on actual and forecasted spend |
| Compute Optimizer | Rightsizing before purchasing SPs — always rightsize first |
| AWS Cost Anomaly Detection | ML-based detection of unexpected cost spikes |
| Trusted Advisor | Flags unused RIs, underutilized instances, and cost optimization opportunities |
| Organizations | Consolidated billing and Savings Plan sharing across all member accounts |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
