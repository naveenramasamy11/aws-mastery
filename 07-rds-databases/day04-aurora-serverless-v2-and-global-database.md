# ☁️ Aurora Serverless v2 & Aurora Global Database — AWS Mastery

> **Scaling a database used to mean a maintenance window and a prayer; Aurora Serverless v2 made it a background thread.**

---

## 📖 Concept

Aurora Serverless v1 was a genuinely useful idea with a rough execution: it scaled in discrete steps, paused entirely during idle periods (adding cold-start latency that broke connection-pooling assumptions), and didn't support many Aurora features like Global Database or Performance Insights. Aurora Serverless v2 is effectively a rewrite: it scales capacity in fine-grained increments (0.5 ACU steps) in seconds rather than minutes, scales up and down within a single second of load change, and — critically for migration work — supports the full feature set of provisioned Aurora, including Global Database, Multi-AZ, and reader instances mixed with provisioned instances in the same cluster.

The practical migration pattern this enables is a mixed-capacity cluster: a provisioned writer instance sized for baseline load, paired with Serverless v2 reader replicas that automatically absorb reporting-query spikes or batch-job load without needing to right-size the whole cluster for the worst case. For workloads migrated from on-prem with unpredictable or highly seasonal traffic (retail during sale events, payroll systems at month-end), this removes an entire category of capacity planning that used to require guessing peak load six months in advance.

Aurora Global Database solves an orthogonal problem: cross-region disaster recovery and read locality for globally distributed applications. It replicates a primary cluster's data to up to five secondary regions with typical replication lag under one second, using dedicated infrastructure (not logical replication) that doesn't add load to the primary cluster's compute. In a DR failover, `switchover` (planned) or `failover` (unplanned) promotes a secondary region to primary in under a minute for switchover, which is an entirely different RTO conversation than the hours-long restore-from-snapshot approach still common with on-prem-style DR runbooks.

For a migration factory, the combination of these two features is what makes "lift and shift the database, then modernize the scaling story" a realistic phased approach — you don't have to solve global resilience and elastic scaling on day one of the migration.

---

## 🏗️ Architecture Snapshot

```
┌────────────────────────────────────────────────────────────────────┐
│  Aurora Cluster (mixed capacity)                                 │
│                                                                    │
│  ┌───────────────────┐        ┌──────────────────────────────┐      │
│  │  Writer            │        │  Reader (Serverless v2)     │      │
│  │  (Provisioned,      │◀──────▶│  0.5–16 ACU, auto-scales    │      │
│  │   r6g.xlarge)        │        │  within seconds              │      │
│  └───────────────────┘        └───────────────────────────────┘      │
└───────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────┐
│  Aurora Global Database                                           │
│                                                                     │
│  Primary Region (ap-south-1)         Secondary Region (ap-southeast-1)│
│  ┌───────────────────┐   dedicated    ┌──────────────────┐         │
│  │  Primary Cluster   │   replication │  Secondary Cluster │         │
│  │  (writer + readers) │ ──────────▶  │  (read-only,        │         │
│  │                     │   <1s lag    │   promotable on     │         │
│  └───────────────────┘                │   failover)          │         │
│                                        └───────────────────┘         │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Seasonal retail spikes:** An e-commerce platform's Aurora reader fleet scales from 2 ACU baseline to 64 ACU during a flash sale automatically, then scales back down within minutes — no pre-provisioning for peak.
- **Regional DR compliance requirement:** A financial services customer requires an RPO under 5 seconds and RTO under 5 minutes for a critical transactional database — Aurora Global Database with planned switchover testing satisfies both.
- **Read locality for global users:** A SaaS application serving both Indian and Southeast Asian customers routes reads to the nearest Aurora Global Database secondary region, cutting read latency significantly versus a single-region deployment.

---

## 🔧 AWS CLI & Console Examples

### Create an Aurora Serverless v2 reader instance in an existing cluster

```bash
aws rds create-db-instance \
  --db-instance-identifier prod-cluster-reader-1 \
  --db-cluster-identifier prod-cluster \
  --engine aurora-postgresql \
  --db-instance-class db.serverless \
  --region ap-south-1
```

### Set the ACU scaling range for a Serverless v2 cluster

```bash
aws rds modify-db-cluster \
  --db-cluster-identifier prod-cluster \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=32 \
  --apply-immediately
```

### Create an Aurora Global Database

```bash
aws rds create-global-cluster \
  --global-cluster-identifier global-prod-db \
  --source-db-cluster-identifier arn:aws:rds:ap-south-1:111122223333:cluster:prod-cluster

aws rds create-db-cluster \
  --db-cluster-identifier prod-cluster-secondary \
  --engine aurora-postgresql \
  --global-cluster-identifier global-prod-db \
  --region ap-southeast-1
```

### Perform a planned switchover (DR test)

```bash
aws rds switchover-global-cluster \
  --global-cluster-identifier global-prod-db \
  --target-db-cluster-identifier arn:aws:rds:ap-southeast-1:111122223333:cluster:prod-cluster-secondary

# Expected output includes:
# "Status": "switching-over"  <- promotes secondary to primary in place
```

### Terraform — mixed-capacity Aurora cluster

```hcl
resource "aws_rds_cluster" "prod" {
  cluster_identifier = "prod-cluster"
  engine             = "aurora-postgresql"

  serverlessv2_scaling_configuration {
    min_capacity = 0.5
    max_capacity = 32
  }
}

resource "aws_rds_cluster_instance" "writer" {
  cluster_identifier = aws_rds_cluster.prod.id
  instance_class     = "db.r6g.xlarge"
}

resource "aws_rds_cluster_instance" "reader_serverless" {
  cluster_identifier = aws_rds_cluster.prod.id
  instance_class     = "db.serverless"
}
```

---

## 🔐 Security Best Practices

- **Encrypt Global Database secondary regions with region-specific KMS keys:** Cross-region replication requires the secondary cluster to have its own KMS key — you cannot share a single-region key across regions.
- **Restrict `rds:FailoverGlobalCluster` permissions tightly:** An unplanned failover is a significant operational event; this permission should sit with a break-glass role, not standard operator access.
- **Enable IAM database authentication on both primary and secondary clusters:** Keeps credential management consistent across regions instead of maintaining separate secrets per region.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# Watch your Serverless v2 cluster breathe in near real-time
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name ServerlessDatabaseCapacity \
  --dimensions Name=DBClusterIdentifier,Value=prod-cluster \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 --statistics Average
# Watching ACU rise and fall in real-time is oddly satisfying.

# Check exactly how far behind your Global Database secondary is right now
aws cloudwatch get-metric-statistics --namespace AWS/RDS \
  --metric-name AuroraGlobalDBReplicationLag \
  --dimensions Name=DBClusterIdentifier,Value=prod-cluster-secondary \
  --start-time $(date -u -d '5 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) --period 60 --statistics Maximum
```

---

## ⚠️ Gotchas & Tricky Bits

- **Serverless v2 minimum ACU still costs money even at zero load:** Unlike v1, Serverless v2 never fully pauses — a MinCapacity of 0.5 ACU is billed continuously, so set it deliberately rather than defaulting to a comfortable-sounding number.
- **Scaling speed depends on buffer pool size:** A cluster scaling from a very low ACU floor to a much higher ceiling under sudden load can briefly see degraded performance while the buffer pool warms — set MinCapacity higher for spiky-but-predictable workloads.
- **Global Database failover is NOT automatic:** Unlike Multi-AZ within a region, cross-region failover for Global Database requires an explicit `failover-global-cluster` API call — plan and rehearse this in a runbook, don't assume it's automatic.
- **Pro Tip:** Test `switchover-global-cluster` (the planned, zero-data-loss path) regularly in a non-prod global cluster — it's the operation teams almost never rehearse until the unplanned failover day arrives.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → RDS → Databases → [cluster] → Actions → Add reader`
2. **Look for:** The "DB instance class" dropdown — Serverless v2 appears as a distinct option, not a size within provisioned classes.
3. **Key field:** `Capacity range (ACUs)` — set MinCapacity based on steady-state load, not zero, to avoid cold-scaling latency on every quiet period.
4. **Common mistake here:** Setting MaxCapacity too close to MinCapacity, which defeats the purpose of elastic scaling under a real traffic spike.
5. **Confirm with CLI:**
   ```bash
   aws rds describe-db-clusters --db-cluster-identifier prod-cluster \
     --query 'DBClusters[].ServerlessV2ScalingConfiguration'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| Amazon RDS Proxy | Pools connections in front of Aurora Serverless v2, smoothing connection churn during rapid scale events |
| AWS KMS | Provides the region-specific encryption keys required for Global Database secondary clusters |
| Amazon CloudWatch | Tracks `ServerlessDatabaseCapacity` and `AuroraGlobalDBReplicationLag` for both features |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
