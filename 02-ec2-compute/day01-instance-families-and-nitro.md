# ☁️ EC2 Instance Families & the Nitro System — AWS Mastery

> **Picking the wrong instance type is like bringing a lawnmower to a Formula 1 race — it'll run, just not well.**

---

## 📖 Concept

AWS has over 600 EC2 instance types across dozens of families, and the choice you make at launch has direct cost and performance consequences. The naming convention is a map: `m7g.2xlarge` = General Purpose (m), 7th generation, Graviton processor (g), 2x the baseline memory/CPU of xlarge. Read the name, understand the machine.

The Nitro System is the foundation of modern EC2. Introduced in 2017 and now powering virtually all current-generation instances, Nitro offloads virtualization functions (networking, storage, security) to dedicated hardware ASICs and a lightweight hypervisor. The result: near-bare-metal performance, hardware-enforced security isolation (even AWS operators cannot access Nitro instance memory), and consistent low-latency networking. When a customer says "we want bare-metal performance in the cloud," the answer is: you basically already have it on any Nitro instance.

For EKS workloads, instance family selection directly impacts pod density, networking throughput, and cost. `m7g` (Graviton3 general purpose) delivers ~40% better price/performance than comparable x86 instances for containerized workloads that have been compiled for ARM64. `c7g` wins for CPU-intensive microservices. `r7g` for in-memory caches and JVM apps. For GPU inference, `g5` (A10G) or `inf2` (Inferentia2) for ML-specific workloads.

In migration engagements, the biggest cost saving comes not from Reserved Instances or Savings Plans, but from rightsizing — customers who migrated on-prem VMs as `m5.4xlarge` because the VM was 16 vCPUs/64GB, when actual utilization was 8%/12%. CloudWatch metrics + Compute Optimizer tells the real story.

---

## 🏗️ Architecture Snapshot

```
  EC2 Instance Family Decision Tree
  ──────────────────────────────────

  What's your workload?
       │
       ├──▶ Balanced CPU/Memory (web, app servers, EKS nodes)
       │         └──▶ m7g (Graviton, best price-perf) / m7i (Intel)
       │
       ├──▶ CPU-intensive (batch, encoding, microservices)
       │         └──▶ c7g / c7i / c7a (AMD)
       │
       ├──▶ Memory-intensive (JVM, Redis, SAP, databases)
       │         └──▶ r7g / r7i / x2idn (ultra-high mem)
       │
       ├──▶ Storage-optimized (Cassandra, Kafka, Elasticsearch)
       │         └──▶ i4i (NVMe SSD) / d3 (HDD dense)
       │
       ├──▶ GPU workloads (ML training/inference)
       │         └──▶ p4d/p4de (A100) / g5 (A10G) / inf2
       │
       └──▶ Network-intensive (HPC, MPI)
                 └──▶ hpc7g / c7gn (200 Gbps EFA)

  Nitro System Components:
  ┌────────────────────────────────────┐
  │         EC2 Instance               │
  │  ┌──────────┐  ┌────────────────┐  │
  │  │  Guest   │  │  Nitro Card    │  │
  │  │    OS    │  │ (NVMe/ENA/EFA) │  │
  │  └──────────┘  └────────────────┘  │
  │  ┌─────────────────────────────┐   │
  │  │   Nitro Hypervisor (thin)   │   │
  │  └─────────────────────────────┘   │
  │  ┌─────────────────────────────┐   │
  │  │   Nitro Security Chip       │   │
  │  │ (hardware root of trust)    │   │
  └──┴─────────────────────────────┴───┘
```

---

## 💡 Real-World Use Cases

- **EKS node group rightsizing:** A payment gateway migrating 150+ microservices to EKS needs to balance pod density with CPU burst capacity — `m7g.2xlarge` gives 8 vCPU / 32 GB with Graviton3, fitting ~20 medium pods per node vs ~12 on older m5.
- **Spot Instances for batch processing:** A data pipeline runs nightly ETL jobs on `c7g.4xlarge` Spot instances at 70% discount vs On-Demand — with a Spot interruption handler that checkpoints to S3 every 2 minutes, termination is painless.
- **IMDSv2 enforcement in migration:** When rehosting legacy apps via MGN, enforcing `HttpTokens: required` at the AMI launch template level prevents SSRF vulnerabilities that existed in the source environment from being carried forward into AWS.

---

## 🔧 AWS CLI & Console Examples

### Listing and Filtering Instance Types

```bash
# Find all Graviton3 instances with 8+ vCPUs available in ap-south-1
aws ec2 describe-instance-types \
  --region ap-south-1 \
  --filters \
    "Name=processor-info.supported-architecture,Values=arm64" \
    "Name=vcpu-info.default-vcpus,Values=8,16,32" \
  --query 'InstanceTypes[].{Type:InstanceType,vCPU:VCpuInfo.DefaultVCpus,MemGB:MemoryInfo.SizeInMiB}' \
  --output table

# Check if an instance type supports EFA (for HPC/ML)
aws ec2 describe-instance-types \
  --instance-types c7gn.16xlarge \
  --query 'InstanceTypes[].NetworkInfo.EfaSupported'
```

### Launching with IMDSv2 Enforced (always do this)

```bash
# Create a Launch Template with IMDSv2 required — NEVER skip this
aws ec2 create-launch-template \
  --launch-template-name my-secure-template \
  --version-description "IMDSv2 enforced" \
  --launch-template-data '{
    "ImageId": "ami-0abcdef1234567890",
    "InstanceType": "m7g.xlarge",
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpPutResponseHopLimit": 1,
      "HttpEndpoint": "enabled"
    },
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{"Key": "Environment", "Value": "production"}]
    }]
  }'
```

### SSM Session Manager — No Bastion Needed

```bash
# Start a session to an EC2 instance with NO open ports, NO SSH key
aws ssm start-session \
  --target i-0abc123def456789 \
  --region ap-south-1

# Port forwarding via SSM (access RDS on private subnet from your laptop)
aws ssm start-session \
  --target i-0abc123def456789 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["5432"],"localPortNumber":["15432"]}'
# Now connect: psql -h localhost -p 15432 -U postgres
```

### Compute Optimizer Rightsizing Recommendations

```bash
# Get EC2 rightsizing recommendations
aws compute-optimizer get-ec2-instance-recommendations \
  --region ap-south-1 \
  --query 'instanceRecommendations[].{
    Instance:instanceArn,
    Finding:finding,
    Recommended:recommendationOptions[0].instanceType,
    SavingsPercent:recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table
```

### Spot Instance with Interruption Handler

```bash
# Check if your Spot instance is about to be terminated (run this in a cron every 5s)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/spot/termination-time
# Returns empty if safe, returns timestamp if termination is imminent
# Use this to trigger checkpoint/drain logic in your app
```

---

## 🔐 Security Best Practices

- **Always enforce IMDSv2:** Set `HttpTokens: required` and `HttpPutResponseHopLimit: 1` in every Launch Template. Hop limit of 1 prevents containers from reaching the metadata service — critical in EKS where a compromised pod could otherwise steal the node's IAM role credentials.
- **Never store credentials in User Data:** User Data is readable by any process on the instance and stored in plaintext in the EC2 API. Use SSM Parameter Store or Secrets Manager with instance role permissions instead.
- **Use SSM Session Manager instead of SSH:** Eliminates the need for open port 22, bastion hosts, and SSH key management. All sessions are logged to CloudTrail and optionally to S3/CloudWatch Logs.
- **Tag everything at launch:** Use Tag-based SCPs to prevent operations on untagged resources. `aws:RequestTag` conditions in IAM policies enforce tagging at creation time.

---

## 😄 Funny Things to Try

```bash
# Check what instance type you're running on FROM INSIDE the instance
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-type
# The machine knows what it is. Do you?

# Count how many EC2 instances you have running RIGHT NOW across all regions
aws ec2 describe-regions --query 'Regions[].RegionName' --output text | \
  tr '\t' '\n' | \
  xargs -I{} aws ec2 describe-instances \
    --region {} \
    --filters "Name=instance-state-name,Values=running" \
    --query 'length(Reservations[].Instances[])' \
    --output text 2>/dev/null | \
  awk '{s+=$1} END {print "Total running instances:", s}'
# Surprise yourself. Or terrify yourself. Either way, knowledge is power.

# Find instances with public IPs that probably shouldn't have them
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[?PublicIpAddress!=null].[InstanceId,PublicIpAddress,Tags[?Key==`Name`].Value|[0]]' \
  --output table --region ap-south-1
# The "who left the door open?" audit.
```

---

## ⚠️ Gotchas & Tricky Bits

- **EBS-optimized is not automatic on all types:** Older instance families require explicitly enabling EBS optimization. On modern Nitro instances it's always on. Check before blaming your database for slow I/O.
- **Graviton is ARM64 — your Docker images must be multi-arch:** If you run `m7g` nodes in EKS and your container images are only built for `amd64`, pods will fail to schedule with an obscure image pull error. Build with `docker buildx` for `linux/arm64,linux/amd64` and push a multi-arch manifest.
- **Burstable instances (t3/t4g) have CPU credits:** A `t3.medium` running at 100% CPU will burn through its credit balance and throttle to baseline (20% CPU). In production, always set `cpu-credits: unlimited` or switch to a fixed-performance family for steady-state workloads.
- **Spot interruption gives 2 minutes notice — not more:** Plan your checkpointing and drain logic around a hard 2-minute window. EventBridge + Lambda or a metadata polling script are the two patterns. Anyone who says "we'll just handle it manually" has not been interrupted yet.
- **Pro Tip:** Use `ec2-instance-selector` (open source CLI from AWS) to filter instance types by vCPU, memory, architecture, network performance, and price in one command — vastly better than scrolling the console pricing page.

---

## 📸 Console Walkthrough

> *Enabling Compute Optimizer and finding your first rightsizing opportunity*

1. **Navigate to:** `AWS Console → Compute Optimizer → Get Started`
2. **Opt in:** Click `Opt in to Compute Optimizer` (free service, no agent needed)
3. **Wait:** Recommendations take 12–24 hours to generate (needs CloudWatch metric history)
4. **Navigate to:** `Compute Optimizer → EC2 Instances`
5. **Look for:** Instances marked `Over-provisioned` — these have recommendations
6. **Key field:** `Estimated monthly savings` — sort by this column descending
7. **Common mistake here:** Accepting `Savings Plans` recommendations from Compute Optimizer without checking if the instance change also requires OS/application validation
8. **Confirm with CLI:**
   ```bash
   aws compute-optimizer get-enrollment-status
   # Should return: {"status": "Active"}
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| Auto Scaling Groups | EC2 instances are managed at scale via ASGs with Launch Templates |
| EKS Managed Node Groups | Backed by ASGs — instance family choice drives pod density and cost |
| SSM Parameter Store | Secure config injection for EC2 at launch via User Data + IAM role |
| Compute Optimizer | ML-driven rightsizing recommendations based on CloudWatch utilization |
| AWS Cost Explorer | RI and Savings Plan purchase decisions based on EC2 usage patterns |
| EC2 Image Builder | Automated, pipelined AMI creation with CIS hardening and patching |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
