# 💰 Spot Instance Strategies & AWS Compute Optimizer — AWS Mastery

> **"Running 70% of your EKS compute on Spot while maintaining 99.9% application availability. Not luck — architecture."**

---

## 📖 Concept

### Spot Instances

Spot Instances use unused EC2 capacity at up to 90% discount vs On-Demand pricing. The trade-off: AWS can reclaim them with a 2-minute warning (the "interruption notice") when capacity is needed elsewhere. This makes Spot fundamentally unsuitable for stateful, non-interruptible workloads — and extremely powerful for everything else.

**Spot pricing** is now mostly stable (AWS changed the model in 2017 — price fluctuations are rare, not constant). What causes interruptions isn't price changes but capacity reclamation. The interruption rate varies dramatically by instance type and Availability Zone — `m5.large` in `us-east-1a` might have <5% interruption rate, while `c5.xlarge` in the same AZ might be 15-20%. The Spot Instance Advisor shows historical interruption frequency — use it to pick low-interruption-rate combinations.

**Spot Fleet vs EC2 Auto Scaling Group with Spot:**
- **Spot Fleet** is the older API — supports mixed instance types, diversification, and allocation strategies. Still valid but ASG with mixed instances is now preferred for most use cases.
- **ASG with mixed instances policy** — the modern approach. Define a base of On-Demand (e.g., 20%) and fill the rest with Spot across multiple instance families and sizes. Use `price-capacity-optimized` allocation strategy (introduced 2022) — it selects from the pools with lowest interruption risk AND lowest price, not just lowest price.

**Spot for EKS/containers:** Karpenter and Managed Node Groups both support Spot natively. With Karpenter, you define NodePools that allow Spot, and Karpenter diversifies across 20+ instance types automatically, reducing the blast radius of any single Spot interruption.

### AWS Compute Optimizer

Compute Optimizer uses ML to analyze 14 days of CloudWatch utilization metrics and produce rightsizing recommendations for EC2, EBS volumes, Lambda functions, ECS tasks on Fargate, and Auto Scaling Groups. It outputs a finding (Under-provisioned, Over-provisioned, Optimized, Not enough data) with specific recommended instance types and projected savings.

Compute Optimizer integrates with AWS Organizations to produce account-wide recommendations. The free tier covers basic recommendations; the paid enhanced infrastructure metrics tier extends the analysis window to 3 months and improves recommendation accuracy by 20-30% for variable workloads.

---

## 🏗️ Architecture Snapshot

```
EKS Cluster — Spot-Heavy Compute Architecture with Karpenter
────────────────────────────────────────────────────────────────

  EKS Control Plane (AWS managed)
  │
  ├── Node Group: Critical (On-Demand)
  │     m5.xlarge x2 (for system workloads, PodDisruptionBudgets)
  │
  ├── NodePool: General (Spot via Karpenter)
  │     Instance types: [m5, m5a, m5d, m6i, m6a, c5, c6i]
  │     Diversified across 3 AZs
  │     90% of total capacity
  │
  └── NodePool: GPU (Spot)
        g4dn.xlarge, g5.xlarge
        ML inference batch jobs

Compute Optimizer Recommendation Flow:
  CloudWatch Metrics (14 days)
        │
        ▼
  Compute Optimizer
  ┌─────────────────────────────────────────┐
  │  EC2 Analysis                           │
  │  ┌──────────────────┬────────────────┐  │
  │  │ Current         │ Recommended    │  │
  │  │ m5.4xlarge      │ m5.2xlarge     │  │
  │  │ CPU: avg 12%    │ Save: $180/mo  │  │
  │  │ Mem: avg 18%    │ Perf risk: Low │  │
  │  └──────────────────┴────────────────┘  │
  │                                         │
  │  Lambda Analysis                        │
  │  128MB → 512MB (+$3/mo saves 40% time)  │
  └─────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **EKS batch workloads on Spot:** CI/CD build runners, ML training jobs, and data processing pipelines run on Spot-backed Karpenter node pools. When a Spot interruption occurs, Karpenter drains the node, the pod is rescheduled on a new Spot node (different type/AZ), and the job resumes from a checkpoint. Effective cost reduction: 60-75%.
- **Compute Optimizer for post-migration rightsizing:** After a lift-and-shift migration (rehost all servers 1:1), run Compute Optimizer across the migrated account. Typically finds 30-40% of instances are over-provisioned. Rightsizing recommendations + RI/Savings Plan purchases on the rightsized baseline can cut compute spend by 50% vs the original on-prem sizing.
- **Mixed On-Demand + Spot for stateless APIs:** REST API servers behind an ALB run on an ASG with 25% On-Demand base and 75% Spot. `price-capacity-optimized` strategy across 6 instance types. Even with a 10% Spot interruption rate, the ASG automatically replaces interrupted instances. P99 latency is unaffected — ALB health checks remove unhealthy targets before the 2-minute warning expires.

---

## 🔧 AWS CLI & Console Examples

### Create an ASG with Mixed On-Demand + Spot

```bash
# First create a launch template (IMDSv2 required)
aws ec2 create-launch-template \
  --launch-template-name "webapp-lt" \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "KeyName": "my-key",
    "SecurityGroupIds": ["sg-0abc123def456"],
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpEndpoint": "enabled"
    },
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Name", "Value": "webapp-spot"}]
    }]
  }' \
  --region ap-south-1

# Create ASG with mixed instances policy
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "webapp-mixed-asg" \
  --min-size 4 \
  --max-size 40 \
  --desired-capacity 10 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb,subnet-ccc" \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "webapp-lt",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "m5.xlarge"},
        {"InstanceType": "m5a.xlarge"},
        {"InstanceType": "m5d.xlarge"},
        {"InstanceType": "m6i.xlarge"},
        {"InstanceType": "m6a.xlarge"},
        {"InstanceType": "c5.2xlarge"},
        {"InstanceType": "c6i.2xlarge"}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 25,
      "SpotAllocationStrategy": "price-capacity-optimized"
    }
  }' \
  --region ap-south-1
```

### Handle Spot Interruption Notices with EventBridge

```bash
# Create EventBridge rule to catch Spot interruption warnings
aws events put-rule \
  --name "SpotInterruptionWarning" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Spot Instance Interruption Warning"]
  }' \
  --state ENABLED \
  --region ap-south-1

# Route to Lambda for graceful drain logic
aws events put-targets \
  --rule "SpotInterruptionWarning" \
  --targets '[{
    "Id": "spot-drain-lambda",
    "Arn": "arn:aws:lambda:ap-south-1:123456789012:function:spot-drain"
  }]' \
  --region ap-south-1

# Lambda payload will contain:
# {
#   "detail": {
#     "instance-id": "i-0abc123def456",
#     "instance-action": "terminate"
#   }
# }
# Lambda should: deregister from target group, drain connections, checkpoint state
```

### Query Compute Optimizer Recommendations

```bash
# Enable Compute Optimizer for the account (one-time)
aws compute-optimizer update-enrollment-status \
  --status Active \
  --include-member-accounts \
  --region ap-south-1

# Get EC2 instance recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --region ap-south-1 \
  --query 'instanceRecommendations[?finding==`OVER_PROVISIONED`].[instanceArn,finding,recommendationOptions[0].instanceType,recommendationOptions[0].projectedUtilizationMetrics]' \
  --output table

# Get recommendations with estimated savings
aws compute-optimizer get-ec2-instance-recommendations \
  --region ap-south-1 \
  --query 'instanceRecommendations[*].{
    Instance: instanceArn,
    Finding: finding,
    CurrentType: currentInstanceType,
    RecommendedType: recommendationOptions[0].instanceType,
    SavingsOpportunity: recommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value
  }' \
  --output table

# Get Lambda function recommendations
aws compute-optimizer get-lambda-function-recommendations \
  --region ap-south-1 \
  --query 'lambdaFunctionRecommendations[?finding==`OVER_PROVISIONED`].[functionArn,memorySizeRecommendationOptions[0].memorySize,memorySizeRecommendationOptions[0].projectedUtilizationMetrics]' \
  --output table

# Export all recommendations to S3 (for bulk analysis)
aws compute-optimizer export-ec2-instance-recommendations \
  --s3-destination-config '{
    "bucket": "my-cost-optimization-reports",
    "keyPrefix": "compute-optimizer/ec2/"
  }' \
  --region ap-south-1
```

### Check Spot Interruption Rate via Spot Advisor API

```bash
# Fetch Spot interruption frequency data (public API, no auth needed)
curl -s https://spot-price.s3.amazonaws.com/spot.js | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
# Find ap-south-1 Linux m5.xlarge pricing
for region in data['config']['regions']:
    if region['region'] == 'ap-south-1':
        for type_group in region['instanceTypes']:
            for size in type_group['sizes']:
                if 'xlarge' in size['size']:
                    print(f\"{size['size']}: \${size['valueColumns'][0]['prices']['USD']}/hr\")
        break
" 2>/dev/null | head -20
```

### Karpenter NodePool for Spot-Optimized EKS

```yaml
# karpenter-nodepool-spot.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-general
spec:
  template:
    spec:
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values:
            - m5.xlarge
            - m5a.xlarge
            - m5d.xlarge
            - m6i.xlarge
            - m6a.xlarge
            - c5.2xlarge
            - c6i.2xlarge
            - c6a.2xlarge
        - key: topology.kubernetes.io/zone
          operator: In
          values: ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      # Spot interruption handling — drain before termination
      terminationGracePeriod: 120s
  limits:
    cpu: 1000
    memory: 4000Gi
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
```

### Terraform — Compute Optimizer Enrollment

```hcl
resource "aws_computeoptimizer_enrollment_status" "org" {
  status                    = "Active"
  include_member_accounts   = true
}

# Export recommendations to S3 for analysis
resource "aws_computeoptimizer_recommendation_preferences" "ec2" {
  resource_type = "Ec2Instance"

  scope {
    name  = "AccountId"
    value = data.aws_caller_identity.current.account_id
  }

  enhanced_infrastructure_metrics = "Active"  # Paid tier — 3 months history
  inferred_workload_types          = "Active"
}
```

---

## 🔐 Security Best Practices

- **Use IMDSv2 on all Spot instances.** Instance metadata access from within a Spot instance pod (in EKS) is a common credential theft vector. Enforce `HttpTokens=required` in your launch template and the EC2NodeClass in Karpenter. Block pod-level IMDS access entirely with a network policy or by setting `--deny-imds-access` in EKS node group config.
- **Tag Spot instances with capacity-type.** Use `karpenter.sh/capacity-type: spot` node labels and node selectors/tolerations to ensure only interruption-tolerant workloads land on Spot nodes. Critical workloads should use `nodeAffinity` to require `on-demand`.
- **Don't store ephemeral Spot state in instance storage without checkpointing.** If your batch job writes intermediate state to the Spot node's local disk, you lose it on interruption. Use S3, EFS, or a distributed cache for checkpoint storage.

---

## 😄 Funny Things to Try

```bash
# Find your most over-provisioned EC2 instance
aws compute-optimizer get-ec2-instance-recommendations \
  --region ap-south-1 \
  --query 'sort_by(instanceRecommendations[?finding==`OVER_PROVISIONED`], &recommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value)[-1].{
    Instance: instanceArn,
    CurrentType: currentInstanceType,
    AvgCPU: utilizationMetrics[?name==`CPU`].value | [0],
    WastedMoney: recommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value
  }' \
  --output json
# This is the instance where someone said "let's get a bigger one just in case"
# and then promptly forgot about it. The biggest number here is your hall of shame.

# Check what percentage of your instances are Spot right now
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query '{
    Total: length(Reservations[].Instances[]),
    Spot: length(Reservations[].Instances[?InstanceLifecycle==`spot`])
  }' \
  --region ap-south-1 \
  --output json
# If Spot is 0% of running instances, you're leaving significant money on the table.
# If Spot is 100%, you're either very brave or running all batch jobs.
```

---

## ⚠️ Gotchas & Tricky Bits

- **`lowest-price` Spot allocation strategy is a trap.** It concentrates all Spot instances in the cheapest pool, maximizing interruption risk. Use `price-capacity-optimized` or `capacity-optimized` — they spread across pools with available capacity. The price difference between strategies is typically <5%, but the reliability difference is huge.
- **Spot interruptions happen at the worst time.** AWS reclaims capacity when demand spikes — Friday afternoon, holiday season starts, major sporting events. If your "batch job that doesn't matter" suddenly matters on Black Friday, running it on Spot in `us-east-1` at peak time is brave.
- **Compute Optimizer needs 14 days of data.** Instances launched less than 2 weeks ago show "Not enough data." Don't act on recommendations for recently migrated instances — the utilization pattern from initial smoke testing doesn't represent steady-state load.
- **Rightsizing recommendations don't account for burst headroom.** A `t3.medium` at 15% average CPU might spike to 90% for 30 seconds every 5 minutes. Compute Optimizer sees average utilization, not peak. Check `CPUUtilization` p99 before downsizing — the recommendation might be technically correct but operationally wrong.
- **Pro Tip:** The Spot Instance Advisor website (https://aws.amazon.com/ec2/spot/instance-advisor/) shows interruption frequency by instance type and region. Cross-reference it when choosing instance type diversification for your Spot pools. `m5.large` consistently has one of the lowest interruption rates globally — it's the Spot workhorse.

---

## 📸 Console Walkthrough

1. **Spot guidance:** `AWS Console → EC2 → Spot Requests → Spot Instance Advisor` — check interruption frequency before choosing instance types. Aim for "<5% interruption rate" pool combinations.
2. **ASG with Spot:** `EC2 → Auto Scaling Groups → Create Auto Scaling Group`. On the "Choose instance launch options" step, select "Combine purchase options and instance types". Set On-Demand base capacity to protect critical minimum capacity, then configure Spot for the rest.
3. **Key Spot setting:** `Allocation strategy` — change from default "Lowest price" to "Price and capacity optimized." This single change improves Spot reliability without increasing cost meaningfully.
4. **Compute Optimizer:** `AWS Console → Compute Optimizer → Dashboard`. The dashboard shows a summary across all resource types. Navigate to "EC2 instances" → filter by `OVER_PROVISIONED` → sort by `Estimated monthly savings`. Start rightsizing from the highest savings items.
5. **Export for analysis:** `Compute Optimizer → EC2 instances → Export recommendations` → select S3 bucket → download CSV for spreadsheet analysis or feed into Cost Explorer.

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Cost Explorer | Analyze actual Spot vs On-Demand spend split; view Savings Plans utilization alongside Spot savings |
| Karpenter | The best way to use Spot in EKS — automatic diversification, interruption handling, and bin packing |
| EC2 Auto Scaling | ASG mixed instances policy is the primary mechanism for Spot + On-Demand mixing outside EKS |
| CloudWatch | Monitor `SpotInterruptionWarning` events and ASG scaling activities; set alarms on Spot capacity pool depletion |
| AWS Trusted Advisor | Overlaps with Compute Optimizer for rightsizing but also covers idle resources, underutilized EBS volumes, and unassociated Elastic IPs |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
