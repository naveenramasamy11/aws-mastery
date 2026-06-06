# ☁️ Aurora vs RDS — Architecture, Failover & When to Choose — AWS Mastery

> **Aurora is not just a faster RDS — it's a fundamentally different storage architecture that laughs at your failover drills.**

---

## 📖 Concept

Amazon RDS (Relational Database Service) gives you managed MySQL, PostgreSQL, Oracle, SQL Server, and MariaDB on EC2 instances — AWS handles backups, patching, and Multi-AZ failover. Amazon Aurora is AWS's cloud-native database engine, compatible with MySQL and PostgreSQL but built on a completely different storage architecture. Understanding the difference is critical for architecture decisions.

**RDS Multi-AZ** replicates data synchronously to a standby instance in another AZ. On failure, DNS flips to the standby — typical failover time is 60–120 seconds. RDS Read Replicas use async replication and can be promoted in a DR scenario, but are primarily for read scaling.

**Aurora's storage layer** is shared across all instances and consists of 6 copies of data across 3 AZs — written in quorums (4/6 writes, 3/6 reads). The compute layer (Aurora instances) is separate from storage. Aurora Writer failover takes 30 seconds or less. Aurora Global Database extends this to cross-region — a secondary region can take over in under a minute with RPO near-zero.

**Aurora Serverless v2** scales the Aurora compute capacity in fine-grained Aurora Capacity Units (ACUs) — from 0.5 ACU to 128 ACU — in milliseconds. It's ideal for dev/test environments, unpredictable workloads, and applications with intermittent traffic. Unlike v1, it doesn't go to zero (minimum 0.5 ACU), which means no cold starts for production use.

For migration engagements, the choice often comes down to: existing workload compatibility (RDS is simpler to migrate to), performance requirements (Aurora is consistently 3–5× faster for MySQL workloads), and cost (Aurora costs more per instance but saves on storage and I/O at scale).

---

## 🏗️ Architecture Snapshot

```
  RDS Multi-AZ vs Aurora Cluster
  ─────────────────────────────────────────────────────

  RDS Multi-AZ:
  ┌──────────────────┐    sync     ┌──────────────────┐
  │  Primary RDS     │────repl────▶│  Standby RDS     │
  │  (AZ-A)          │             │  (AZ-B)          │
  │  EBS Storage     │             │  EBS Storage     │
  └──────────────────┘             └──────────────────┘
  Failover: 60-120 seconds, DNS flip

  Aurora Cluster:
  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
  │ Aurora Writer │   │ Aurora Reader │   │ Aurora Reader │
  │  (AZ-A)       │   │  (AZ-B)       │   │  (AZ-C)       │
  └───────┬───────┘   └───────┬───────┘   └───────┬───────┘
          │                   │                   │
          └──────────┬────────┘───────────────────┘
                     ▼
          ┌──────────────────────────────────────┐
          │  Aurora Distributed Storage Layer     │
          │  6 copies × 3 AZs (quorum writes)    │
          │  Auto-scales to 128TB, no provisioning│
          └──────────────────────────────────────┘
  Failover: <30 seconds, same endpoint
```

---

## 💡 Real-World Use Cases

- **SAP workload migration:** Moving SAP HANA-adjacent OLTP workloads to Aurora PostgreSQL — Aurora's 3-5× performance improvement over standard PostgreSQL RDS means right-sizing down while maintaining throughput.
- **Dev/test cost optimization:** Aurora Serverless v2 dev clusters scale to 0.5 ACU when idle (nights/weekends), reducing dev database costs by 80% vs keeping a `db.r6g.large` running 24/7.
- **Multi-region active-passive DR:** Aurora Global Database with a secondary region in `us-east-1` (primary in `ap-south-1`) — RPO of typically <1 second, RTO under 1 minute for planned failovers.

---

## 🔧 AWS CLI & Console Examples

### RDS Instance Operations

```bash
# List all RDS instances with engine and status
aws rds describe-db-instances \
  --region ap-south-1 \
  --query 'DBInstances[].{ID:DBInstanceIdentifier,Engine:Engine,Class:DBInstanceClass,Status:DBInstanceStatus,MultiAZ:MultiAZ}' \
  --output table

# Create an Aurora PostgreSQL Serverless v2 cluster
aws rds create-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --master-username admin \
  --master-user-password "$(aws secretsmanager get-random-password --password-length 16 --query RandomPassword --output text)" \
  --vpc-security-group-ids sg-0abc123 \
  --db-subnet-group-name my-db-subnet-group \
  --backup-retention-period 7 \
  --storage-encrypted \
  --region ap-south-1

# Failover an Aurora cluster to a specific reader
aws rds failover-db-cluster \
  --db-cluster-identifier my-aurora-cluster \
  --target-db-instance-identifier my-aurora-reader-1 \
  --region ap-south-1
```

### Performance Insights

```bash
# Get Performance Insights data for a specific RDS instance
aws pi get-resource-metrics \
  --service-type RDS \
  --identifier db-ABC123DEFGHIJKLMNOPQRSTU \
  --metric-queries '[{"Metric":"db.load.avg","GroupBy":{"Group":"db.wait_event"}}]' \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period-in-seconds 60 \
  --region ap-south-1
```

### RDS Proxy Setup (for Lambda/EKS)

```bash
# Create an RDS Proxy for connection pooling
aws rds create-db-proxy \
  --db-proxy-name my-rds-proxy \
  --engine-family POSTGRESQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"arn:aws:secretsmanager:ap-south-1:123456789012:secret:rds-creds","IAMAuth":"REQUIRED"}]' \
  --role-arn arn:aws:iam::123456789012:role/rds-proxy-role \
  --vpc-subnet-ids subnet-0abc subnet-0def \
  --vpc-security-group-ids sg-0abc123 \
  --require-tls \
  --region ap-south-1
```

---

## 🔐 Security Best Practices

- **Use IAM database authentication for Aurora PostgreSQL/MySQL:** Eliminates static passwords — use `aws rds generate-db-auth-token` for short-lived tokens. Combine with RDS Proxy for Lambda workloads.
- **Enable Deletion Protection on production clusters:** `aws rds modify-db-cluster --deletion-protection` — one CloudFormation `terraform destroy` shouldn't take out your production database.
- **Encrypt at rest with KMS CMK:** Use customer-managed KMS keys, not AWS-managed. This lets you control key rotation, audit usage, and revoke access if needed.
- **Put RDS in isolated subnets with no internet route:** Database subnets should have no route to internet gateway or NAT gateway. The only access should be from application Security Groups on the database port.

---

## 😄 Funny Things to Try

```bash
# Check what's actually connecting to your RDS right now
aws rds describe-db-instances \
  --query 'DBInstances[].{ID:DBInstanceIdentifier,Connections:Endpoint.Address}' \
  --output table --region ap-south-1
# Then on the DB: SELECT client_addr, count(*) FROM pg_stat_activity GROUP BY client_addr;
# How many Lambda functions have you left connected? All of them? All of them.

# Check if your RDS automated backups are actually happening
aws rds describe-db-instances \
  --query 'DBInstances[?BackupRetentionPeriod==`0`].DBInstanceIdentifier' \
  --output text --region ap-south-1
# Output: any RDS with ZERO backup retention. That's a resume-generating incident waiting.

# Find RDS instances NOT using encrypted storage
aws rds describe-db-instances \
  --query 'DBInstances[?StorageEncrypted==`false`].{ID:DBInstanceIdentifier,Engine:Engine}' \
  --output table --region ap-south-1
# In 2024, unencrypted RDS instances are like driving without a seatbelt.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Aurora storage is billed per GB-month — there's no pre-provisioning:** You don't choose storage size for Aurora. It auto-scales. But you pay for every GB used, including deleted rows until VACUUM runs. Monitor `FreeLocalStorage` and `VolumeBytesUsed`.
- **Aurora Serverless v2 doesn't pause like v1:** Serverless v2 minimum is 0.5 ACU (not 0). If you need true zero-cost when idle, use Serverless v1 (MySQL 5.7/PostgreSQL 10 only). v1 has cold start latency on first query after pause.
- **RDS Multi-AZ failover changes the underlying instance — endpoint stays the same:** Applications must handle TCP reconnection. A properly configured connection pool with retry logic handles this automatically. Hard-coded connection creation without retry will fail.
- **Parameter Group changes require a reboot:** Dynamic parameters apply immediately. Static parameters (like `max_connections`, `shared_buffers`) require an instance reboot — schedule this in a maintenance window.
- **Pro Tip:** Enable Enhanced Monitoring (1-second granularity) and Performance Insights on all production RDS instances. They're cheap ($0/month for 7-day retention) and give you OS-level and query-level visibility that basic CloudWatch metrics don't provide.

---

## 📸 Console Walkthrough

> *Setting up Aurora Serverless v2 for a dev environment*

1. **Navigate to:** `AWS Console → RDS → Create database`
2. **Engine:** Select `Aurora (MySQL Compatible)` or `Aurora (PostgreSQL Compatible)`
3. **Templates:** Select `Dev/Test`
4. **DB cluster:** Enter a cluster identifier like `dev-aurora-cluster`
5. **Instance configuration:** Select `Serverless v2`
6. **Key field:** `Minimum ACUs = 0.5, Maximum ACUs = 4` for dev — this keeps cost minimal while allowing bursts
7. **Connectivity:** Select your VPC, choose private subnets, and the database security group
8. **Common mistake here:** Not setting a DB subnet group with subnets in at least 2 AZs — Aurora requires multi-AZ even for dev clusters
9. **Confirm with CLI:**
   ```bash
   aws rds describe-db-clusters \
     --db-cluster-identifier dev-aurora-cluster \
     --query 'DBClusters[].{Status:Status,Engine:Engine,Capacity:ServerlessV2ScalingConfiguration}'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| RDS Proxy | Manages connection pools — critical for Lambda and EKS workloads |
| Secrets Manager | Stores and rotates RDS credentials automatically |
| Performance Insights | Query-level performance visibility at 1-second granularity |
| CloudWatch | DB-level metrics, Enhanced Monitoring, alarms on CPU/connections/storage |
| DMS | Migrate data into Aurora from on-premises databases or other engines |
| Aurora Global Database | Cross-region replication with <1s RPO for DR or global low-latency reads |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
