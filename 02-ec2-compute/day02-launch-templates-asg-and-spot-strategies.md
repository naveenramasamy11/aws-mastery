# ☁️ Launch Templates, Auto Scaling Groups & Spot Strategies — AWS Mastery

> **The trifecta that separates fixed-cost static infrastructure from elastic, cost-optimized fleets that scale with demand and survive spot interruptions.**

---

## 📖 Concept

### Launch Templates

Launch Templates (LTs) are the modern successor to Launch Configurations. They're version-controlled, reusable blueprints that define everything about how an EC2 instance should be launched: AMI, instance type, key pair, security groups, IAM role, user data, EBS volumes, placement group, metadata options, and more. Unlike Launch Configurations, LTs support versioning (you can have v1, v2, v3 and roll back), can specify multiple instance types for Spot/On-Demand mixed fleets, and are required for newer features like Karpenter and EC2 Fleet.

The migration to Launch Templates from Launch Configurations is not optional — AWS stopped accepting new Launch Configurations in 2023. If you're still running LCs in production, you're on borrowed time.

### Auto Scaling Groups

An ASG is the fleet manager. It uses a Launch Template to spin up instances, monitors their health via EC2 status checks and optionally ELB health checks, and maintains a desired capacity within min/max bounds. ASGs integrate with CloudWatch alarms (scale out when CPU > 70%, scale in when CPU < 30%), scheduled scaling (predictable traffic patterns), and predictive scaling (ML-based, learns your traffic curve).

The critical architectural decision: **attach your ASG to a Target Group** (ALB/NLB), not directly to instances. This enables health-check-based termination — if an instance fails the ALB health check, the ASG replaces it automatically.

### Spot Instance Strategies

Spot Instances offer up to 90% cost savings over On-Demand but can be interrupted with 2 minutes notice when EC2 needs capacity back. The key is designing for interruption:

**Spot Allocation Strategies (choose one):**
- `capacity-optimized` — picks the instance pool with the most available capacity, minimizing interruptions. Best for batch workloads and EKS node groups.
- `price-capacity-optimized` — similar to above but also considers price (AWS recommendation since 2022).
- `lowest-price` — picks cheapest pool. Higher interruption risk. Only useful if cost is the only concern and workloads are fully fault-tolerant.
- `diversified` — spreads across multiple pools. Good for long-running fleets that need sustained capacity.

For EKS, Karpenter uses `price-capacity-optimized` by default and diversifies across instance families automatically.

---

## 🏗️ Architecture Snapshot

```
Mixed On-Demand + Spot ASG with ALB
──────────────────────────────────────────────────────────────────

Internet
   │
   ▼
┌──────────────────────────────────────────┐
│  Application Load Balancer               │
│  (ap-south-1, Multi-AZ)                  │
└───────────────────┬──────────────────────┘
                    │  Target Group
                    ▼
┌──────────────────────────────────────────┐
│  Auto Scaling Group                      │
│  ┌─────────────────────────────────────┐ │
│  │  Launch Template v3                 │ │
│  │  AMI: ami-0abc123 (Golden AMI)      │ │
│  │  Overrides: [t3.medium, t3.large,   │ │
│  │             m5.large, m5.xlarge]    │ │
│  └─────────────────────────────────────┘ │
│                                          │
│  Instance Refresh ──▶ rolling replace    │
│                                          │
│  ┌────────────────────┐                  │
│  │  On-Demand Base    │  (always-on: 2)  │
│  │  [m5.large x2]     │                  │
│  └────────────────────┘                  │
│  ┌────────────────────┐                  │
│  │  Spot Fleet        │  (elastic: 0-18) │
│  │  [t3/m5 mixed]     │  price-capacity  │
│  │  ap-south-1a,1b,1c │  optimized       │
│  └────────────────────┘                  │
└──────────────────────────────────────────┘

Spot Interruption Flow:
  EC2 → 2-min warning → instance metadata → 
  your drain script → ASG replaces → ALB deregisters
```

---

## 💡 Real-World Use Cases

- **Web tier cost reduction:** Run 2 On-Demand instances as a base (handle interruption gaps), then 80% Spot for the rest. A production e-commerce platform can cut compute cost by 60-70% with `capacity-optimized` strategy and 4+ instance type overrides.
- **EKS worker nodes:** Karpenter provisioners use Spot with `price-capacity-optimized` across c5/m5/r5 families. When a Spot node gets interrupted, Karpenter launches a replacement before the 2-minute window closes, scheduling pods gracefully.
- **Migration wave testing:** During an AWS migration, use ASG Launch Template versioning to test new AMIs: deploy v2 to 10% of instances via Instance Refresh with `MinHealthyPercentage=90`, validate, then roll to 100%.

---

## 🔧 AWS CLI & Console Examples

### Create a Launch Template

```bash
aws ec2 create-launch-template \
  --launch-template-name "webapp-lt" \
  --version-description "v1 with IMDSv2 and EBS encryption" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "m5.large",
    "KeyName": "my-keypair",
    "SecurityGroupIds": ["sg-0123456789abcdef0"],
    "IamInstanceProfile": {
      "Arn": "arn:aws:iam::123456789012:instance-profile/webapp-profile"
    },
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpPutResponseHopLimit": 1
    },
    "BlockDeviceMappings": [{
      "DeviceName": "/dev/xvda",
      "Ebs": {
        "VolumeSize": 30,
        "VolumeType": "gp3",
        "Encrypted": true,
        "DeleteOnTermination": true
      }
    }],
    "UserData": "IyEvYmluL2Jhc2gKZWNobyBIZWxsbyBXb3JsZA=="
  }' \
  --region ap-south-1
```

### Create a New LT Version

```bash
# Create v2 from v1, only changing AMI
aws ec2 create-launch-template-version \
  --launch-template-name "webapp-lt" \
  --source-version 1 \
  --version-description "v2 updated AMI" \
  --launch-template-data '{"ImageId": "ami-0newami123456789"}' \
  --region ap-south-1

# Set v2 as default
aws ec2 modify-launch-template \
  --launch-template-name "webapp-lt" \
  --default-version 2 \
  --region ap-south-1
```

### Create Mixed On-Demand + Spot ASG

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "webapp-asg" \
  --min-size 2 \
  --max-size 20 \
  --desired-capacity 4 \
  --vpc-zone-identifier "subnet-aaa,subnet-bbb,subnet-ccc" \
  --target-group-arns "arn:aws:elasticloadbalancing:ap-south-1:123456789012:targetgroup/webapp-tg/abc123" \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "webapp-lt",
        "Version": "$Default"
      },
      "Overrides": [
        {"InstanceType": "m5.large"},
        {"InstanceType": "m5.xlarge"},
        {"InstanceType": "t3.large"},
        {"InstanceType": "t3.xlarge"}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 20,
      "SpotAllocationStrategy": "price-capacity-optimized"
    }
  }' \
  --region ap-south-1
```

### Trigger Instance Refresh (Rolling AMI Update)

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name "webapp-asg" \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300,
    "CheckpointPercentages": [20, 50, 100],
    "CheckpointDelay": 600
  }' \
  --region ap-south-1

# Monitor refresh status
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name "webapp-asg" \
  --query 'InstanceRefreshes[0].[Status,PercentageComplete]' \
  --output table \
  --region ap-south-1
```

### Terraform — Full Mixed ASG

```hcl
resource "aws_launch_template" "webapp" {
  name_prefix   = "webapp-"
  image_id      = var.ami_id
  instance_type = "m5.large"  # default, overridden by ASG

  metadata_options {
    http_tokens                 = "required"  # IMDSv2
    http_put_response_hop_limit = 1
  }

  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = 30
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }

  tag_specifications {
    resource_type = "instance"
    tags = { Name = "webapp", Environment = var.env }
  }

  lifecycle { create_before_destroy = true }
}

resource "aws_autoscaling_group" "webapp" {
  name                = "webapp-asg"
  min_size            = 2
  max_size            = 20
  desired_capacity    = 4
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.webapp.arn]
  health_check_type   = "ELB"

  mixed_instances_policy {
    launch_template {
      launch_template_specification {
        launch_template_id = aws_launch_template.webapp.id
        version            = "$Latest"
      }
      override {
        instance_type = "m5.large"
      }
      override {
        instance_type = "m5.xlarge"
      }
      override {
        instance_type = "t3.large"
      }
    }

    instances_distribution {
      on_demand_base_capacity                  = 2
      on_demand_percentage_above_base_capacity = 20
      spot_allocation_strategy                 = "price-capacity-optimized"
    }
  }

  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
      instance_warmup        = 300
    }
  }
}
```

---

## 🔐 Security Best Practices

- **Always set `HttpTokens = required` in Launch Templates.** IMDSv2 prevents SSRF attacks from stealing instance credentials via the metadata service. Many CVEs in containerized environments exploited IMDSv2-disabled instances. Set `HttpPutResponseHopLimit = 1` to prevent containers from reaching IMDS through network hops.
- **Encrypt EBS volumes in Launch Templates.** Set `Encrypted: true` on every `BlockDeviceMapping`. Enable the account-level EBS encryption default in EC2 settings so new volumes are encrypted even if the LT forgets.
- **Use IAM instance profiles, not access keys.** Never put AWS credentials in User Data — they appear in the EC2 console in plaintext. Assign an instance profile with the least-privilege role needed.
- **Set `UpdateLaunchTemplateDefaultVersion` as a Config rule.** Detect LTs where the default version is out of date — stale LTs are a common source of "why is my ASG still launching the old AMI" incidents.

---

## 😄 Funny Things to Try

```bash
# List all launch template versions (great for "which AMI are we actually running?")
aws ec2 describe-launch-template-versions \
  --launch-template-name "webapp-lt" \
  --query 'LaunchTemplateVersions[*].[VersionNumber,VersionDescription,LaunchTemplateData.ImageId]' \
  --output table \
  --region ap-south-1
# Guaranteed to produce the "wait, we have HOW many versions?" moment in every team.

# Check what Spot price actually is right now vs On-Demand
aws ec2 describe-spot-price-history \
  --instance-types m5.large t3.large \
  --product-descriptions "Linux/UNIX" \
  --start-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --region ap-south-1 \
  --query 'SpotPriceHistory[*].[InstanceType,AvailabilityZone,SpotPrice]' \
  --output table
# On-Demand m5.large in ap-south-1 = ~$0.096/hr.
# Spot is often $0.015-0.030/hr. Do the math. 😮
```

---

## ⚠️ Gotchas & Tricky Bits

- **ELB health check grace period must be long enough.** If your app takes 3 minutes to start and the grace period is 60 seconds, the ASG will terminate healthy instances that are still booting. Set `HealthCheckGracePeriod` to your P95 startup time + 60 seconds buffer.
- **Launch Configurations cannot be modified.** Unlike LTs, you can't update an LC — you create a new one and update the ASG. This is why the migration to Launch Templates matters: LT versioning enables Instance Refresh without ASG recreation.
- **`desired_capacity` in Terraform conflicts with external scaling.** If CloudWatch scaling actions change desired capacity and Terraform then runs, it will reset to the Terraform value. Use `lifecycle { ignore_changes = [desired_capacity] }` to let ASG manage desired capacity independently.
- **Spot interruption notice is 2 minutes, but draining takes time.** If your connection drain on the ALB target group is set to 300 seconds, you'll lose in-flight requests. Set deregistration delay to 90 seconds maximum for Spot instances. Implement the EC2 metadata endpoint polling for interruption notice: `curl http://169.254.169.254/latest/meta-data/spot/termination-time`.
- **Pro Tip:** Tag your ASG with `aws:autoscaling:groupName` automatically — every EC2 instance gets this tag. Use it in CloudWatch dashboards to filter metrics per ASG without any custom setup. Also tag instances with `SpotOrOnDemand` using the `InstanceLifecycle` attribute via User Data to track cost by fleet type.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → EC2 → Launch Templates → Create launch template`
2. **Look for:** "Advanced details" section — this is where IMDSv2 and User Data live
3. **Key field:** `Metadata accessible` → set to `V2 only (token required)` — this is the IMDSv2 toggle that many people miss
4. **For ASG:** Go to `EC2 → Auto Scaling Groups → Create` → select your LT → choose "Mix purchase options and instance types" for Spot/OD mix
5. **Common mistake here:** Selecting "Adhere to launch template" instead of "Override launch template" — the latter enables the mixed instance policy with Spot
6. **Confirm instances are actually Spot:**
   ```bash
   aws ec2 describe-instances \
     --filters "Name=tag:aws:autoscaling:groupName,Values=webapp-asg" \
     --query 'Reservations[*].Instances[*].[InstanceId,InstanceLifecycle,InstanceType]' \
     --output table \
     --region ap-south-1
   # InstanceLifecycle = "spot" for Spot instances, null for On-Demand
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| Application Load Balancer | ASG registers instances into ALB target groups; ELB health checks drive ASG replacement |
| EC2 Image Builder | Produces golden AMIs that populate Launch Template ImageId — automate the whole chain |
| CloudWatch | Target tracking and step scaling policies drive ASG desired capacity changes |
| Karpenter | Modern EKS node autoscaler that uses Launch Templates internally, with smarter Spot diversification |
| AWS Systems Manager | Session Manager replaces bastion hosts for ASG instances — no key pair needed if configured in LT |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
