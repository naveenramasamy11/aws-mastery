# 🗄️ AWS Database Migration Service (DMS) & Schema Conversion Tool (SCT) — AWS Mastery

> **"Migrating a 10TB Oracle DB to Aurora PostgreSQL while it's still serving prod traffic — with zero data loss and a 4-hour cutover window. This is how."**

---

## 📖 Concept

### AWS Database Migration Service (DMS)

AWS DMS replicates data from a source database to a target database — continuously, with minimal downtime. It uses a replication instance (an EC2-based managed service) to read from the source endpoint, transform in-flight if needed, and write to the target endpoint.

DMS supports three migration task types:
- **Full Load** — one-time bulk transfer of all existing data
- **Change Data Capture (CDC)** — continuous replication of ongoing changes using the source DB's native transaction log (binlog for MySQL, WAL for PostgreSQL, LogMiner for Oracle, CDC for SQL Server)
- **Full Load + CDC** — migrate existing data AND stream ongoing changes, enabling a near-zero-downtime cutover

**How CDC works:** DMS reads the source transaction log and replays INSERTs, UPDATEs, and DELETEs on the target. The source DB must have binary logging / supplemental logging enabled — a common setup step people forget before the migration window.

**Replication instance sizing** is critical. Under-provision it and DMS becomes the bottleneck. Multi-AZ replication instances provide failover for the replication tier itself (though the migration task must be restarted on failover). For large migrations (>500GB), use `r5.4xlarge` or larger, and separate the full load from the CDC phase.

### AWS Schema Conversion Tool (SCT)

SCT is a desktop application (not a cloud service) that automates the conversion of database schemas between heterogeneous engines. It converts table DDL, views, stored procedures, functions, triggers, and sequences. For objects it can't convert automatically (complex Oracle PL/SQL packages, SQL Server CLR assemblies), it generates action items with effort estimates.

SCT produces a migration assessment report: a breakdown of automatically convertible objects vs objects requiring manual intervention, categorized by complexity. This is the first deliverable in every database migration engagement — it tells you the LOE before you commit to the migration approach.

---

## 🏗️ Architecture Snapshot

```
DMS Full Load + CDC Migration Architecture
──────────────────────────────────────────────────────────────

Source Side (on-prem / EC2 / RDS)        Target Side (RDS / Aurora)
┌──────────────────────────────┐         ┌─────────────────────────┐
│   Oracle / MySQL / SQL Server │         │   Aurora PostgreSQL      │
│   ┌────────────────────────┐ │         │   ┌───────────────────┐  │
│   │   Transaction Log /    │ │         │   │   Tables          │  │
│   │   WAL / binlog         │ │         │   │   (converted      │  │
│   └───────────┬────────────┘ │         │   │    schema)        │  │
└───────────────│──────────────┘         └───────────┬───────────┘  │
                │                                    │
                │         AWS Network               │
                │    ┌──────────────────────┐       │
                │    │  DMS Replication     │       │
                └───▶│  Instance            │───────┘
                     │  (r5.2xlarge)        │
                     │                      │
                     │  ┌────────────────┐  │
                     │  │ Migration Task │  │
                     │  │ Phase 1: Full  │  │
                     │  │ Load           │  │
                     │  │ Phase 2: CDC   │  │
                     │  └────────────────┘  │
                     └──────────────────────┘

SCT Workflow (run locally before DMS):
  SCT Desktop App
  │
  ├── Connect to Source (Oracle) → Scan schema
  ├── Generate Assessment Report
  │     ├── Auto-convertible objects: 87%
  │     └── Action items (manual effort): 13%
  ├── Convert schema → PostgreSQL DDL
  └── Apply to target DB (before DMS task starts)
```

---

## 💡 Real-World Use Cases

- **Oracle to Aurora PostgreSQL migration:** SCT converts table DDL and most PL/SQL stored procedures. DMS full load + CDC replicates data while the app is live. Cutover = redirect the connection string, stop CDC task, promote read replica.
- **SQL Server to Aurora MySQL (SaaS homogenisation):** A SaaS company running 50 customer databases on SQL Server uses SCT to batch-convert schemas, DMS to replicate each tenant DB in parallel (one task per tenant), and Migration Hub to track per-tenant cutover status.
- **On-prem MySQL to RDS MySQL (homogeneous migration):** Even same-engine migrations benefit from DMS CDC for zero-downtime cutover — especially when the source is on-prem and a direct app-level cutover is risky.

---

## 🔧 AWS CLI & Console Examples

### Step 1: Enable Supplemental Logging on Oracle Source

```sql
-- On the source Oracle DB (run as SYSDBA before DMS task starts)
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (PRIMARY KEY) COLUMNS;
ALTER DATABASE ADD SUPPLEMENTAL LOG DATA (UNIQUE INDEX) COLUMNS;

-- Verify:
SELECT LOG_MODE, SUPPLEMENTAL_LOG_DATA_MIN, SUPPLEMENTAL_LOG_DATA_PK
FROM V$DATABASE;
```

### Step 2: Create a DMS Replication Instance

```bash
# Create subnet group first (must span ≥2 AZs)
aws dms create-replication-subnet-group \
  --replication-subnet-group-identifier "dms-subnet-group-prod" \
  --replication-subnet-group-description "DMS prod subnet group" \
  --subnet-ids subnet-abc123 subnet-def456 \
  --region ap-south-1

# Create replication instance (Multi-AZ for resilience)
aws dms create-replication-instance \
  --replication-instance-identifier "dms-prod-instance" \
  --replication-instance-class "dms.r5.2xlarge" \
  --allocated-storage 200 \
  --multi-az \
  --replication-subnet-group-identifier "dms-subnet-group-prod" \
  --vpc-security-group-ids sg-0abc123def456 \
  --engine-version "3.5.3" \
  --region ap-south-1

# Wait until available (takes 5-10 minutes)
aws dms wait replication-instance-available \
  --filters Name=replication-instance-id,Values=dms-prod-instance \
  --region ap-south-1
```

### Step 3: Create Source and Target Endpoints

```bash
# Source: Oracle on-prem via VPN/Direct Connect
aws dms create-endpoint \
  --endpoint-identifier "oracle-source" \
  --endpoint-type source \
  --engine-name oracle \
  --server-name "10.0.0.5" \
  --port 1521 \
  --database-name "ORCL" \
  --username "dms_user" \
  --password "CHANGE_ME" \
  --oracle-settings '{
    "UseLogminerReader": true,
    "ArchivedLogsOnly": false,
    "AddSupplementalLogging": true
  }' \
  --region ap-south-1

# Target: Aurora PostgreSQL
aws dms create-endpoint \
  --endpoint-identifier "aurora-pg-target" \
  --endpoint-type target \
  --engine-name aurora-postgresql \
  --server-name "mydb.cluster-xyz.ap-south-1.rds.amazonaws.com" \
  --port 5432 \
  --database-name "myapp_prod" \
  --username "dms_target_user" \
  --password "CHANGE_ME" \
  --region ap-south-1
```

### Step 4: Test Endpoint Connections

```bash
# Get replication instance ARN first
RI_ARN=$(aws dms describe-replication-instances \
  --filters Name=replication-instance-id,Values=dms-prod-instance \
  --query 'ReplicationInstances[0].ReplicationInstanceArn' \
  --output text --region ap-south-1)

# Test both endpoints
aws dms test-connection \
  --replication-instance-arn "$RI_ARN" \
  --endpoint-arn "arn:aws:dms:ap-south-1:123456789012:endpoint:oracle-source" \
  --region ap-south-1

# Check result
aws dms describe-connections \
  --filters Name=endpoint-id,Values=oracle-source \
  --query 'Connections[0].[Status,LastFailureMessage]' \
  --output table --region ap-south-1
```

### Step 5: Create and Start the Migration Task

```bash
# Table mappings JSON — migrate all tables in schema MYAPP
cat > /tmp/table-mappings.json << 'EOF'
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-myapp-schema",
      "object-locator": {
        "schema-name": "MYAPP",
        "table-name": "%"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "transformation",
      "rule-id": "2",
      "rule-name": "lowercase-schema",
      "rule-action": "convert-lowercase",
      "rule-target": "schema",
      "object-locator": {
        "schema-name": "%"
      }
    }
  ]
}
EOF

# Create Full Load + CDC task
aws dms create-replication-task \
  --replication-task-identifier "oracle-to-aurora-prod" \
  --source-endpoint-arn "arn:aws:dms:ap-south-1:123456789012:endpoint:oracle-source" \
  --target-endpoint-arn "arn:aws:dms:ap-south-1:123456789012:endpoint:aurora-pg-target" \
  --replication-instance-arn "$RI_ARN" \
  --migration-type "full-load-and-cdc" \
  --table-mappings file:///tmp/table-mappings.json \
  --replication-task-settings '{
    "FullLoadSettings": {
      "TargetTablePrepMode": "DROP_AND_CREATE",
      "MaxFullLoadSubTasks": 8,
      "TransactionConsistencyTimeout": 600
    },
    "Logging": {
      "EnableLogging": true,
      "LogComponents": [
        {"Id": "SOURCE_UNLOAD", "Severity": "LOGGER_SEVERITY_DEFAULT"},
        {"Id": "TARGET_LOAD", "Severity": "LOGGER_SEVERITY_DEFAULT"},
        {"Id": "TASK_MANAGER", "Severity": "LOGGER_SEVERITY_DEFAULT"}
      ]
    }
  }' \
  --region ap-south-1

# Start the task
aws dms start-replication-task \
  --replication-task-arn "arn:aws:dms:ap-south-1:123456789012:task:oracle-to-aurora-prod" \
  --start-replication-task-type "start-replication" \
  --region ap-south-1
```

### Step 6: Monitor Migration Progress

```bash
# Check task status and table statistics
aws dms describe-table-statistics \
  --replication-task-arn "arn:aws:dms:ap-south-1:123456789012:task:oracle-to-aurora-prod" \
  --region ap-south-1 \
  --query 'TableStatistics[*].[TableName,TableState,FullLoadRows,InsertCount,UpdateCount,DeleteCount]' \
  --output table

# Check for errors in task logs (from CloudWatch)
aws logs filter-log-events \
  --log-group-name "dms-tasks-oracle-to-aurora-prod" \
  --filter-pattern "ERROR" \
  --region ap-south-1 \
  --query 'events[*].[timestamp,message]' \
  --output table
```

### Terraform — DMS Replication Instance

```hcl
resource "aws_dms_replication_subnet_group" "prod" {
  replication_subnet_group_description = "DMS prod subnet group"
  replication_subnet_group_id          = "dms-prod-subnet-group"
  subnet_ids                           = var.private_subnet_ids
}

resource "aws_dms_replication_instance" "prod" {
  replication_instance_id    = "dms-prod"
  replication_instance_class = "dms.r5.2xlarge"
  allocated_storage          = 200
  multi_az                   = true
  engine_version             = "3.5.3"

  replication_subnet_group_id    = aws_dms_replication_subnet_group.prod.id
  vpc_security_group_ids         = [aws_security_group.dms.id]

  auto_minor_version_upgrade  = true
  publicly_accessible         = false
}

resource "aws_dms_endpoint" "target_aurora" {
  endpoint_id   = "aurora-pg-target"
  endpoint_type = "target"
  engine_name   = "aurora-postgresql"

  server_name   = aws_rds_cluster.target.endpoint
  port          = 5432
  database_name = "myapp_prod"
  username      = var.db_user
  password      = var.db_password
}
```

---

## 🔐 Security Best Practices

- **Never hardcode credentials in DMS endpoint configuration.** Use AWS Secrets Manager — DMS supports `SecretsManagerSecretId` and `SecretsManagerAccessRoleArn` parameters on endpoint creation. The DMS replication instance role assumes the Secrets Manager role to fetch credentials at task start.
- **Run DMS replication instances in private subnets.** They should never be publicly accessible. Use VPC peering, Direct Connect, or VPN for connectivity to on-prem sources. The `publicly-accessible` flag defaults to `false` — keep it that way.
- **Encrypt replication storage.** DMS replication instances have local storage for buffering. Enable encryption at rest using KMS. Set `--kms-key-id` on `create-replication-instance`.
- **Scope the DMS source user's permissions minimally.** For Oracle, the DMS user needs `SELECT ANY TABLE`, `SELECT_CATALOG_ROLE`, and LogMiner permissions — not DBA. Document the exact grant set and don't use SYS or SYSTEM.

---

## 😄 Funny Things to Try

```bash
# Check how many rows DMS has migrated so far — the "are we there yet" command
aws dms describe-table-statistics \
  --replication-task-arn "arn:aws:dms:ap-south-1:123456789012:task:oracle-to-aurora-prod" \
  --region ap-south-1 \
  --query 'sum(TableStatistics[].FullLoadRows)' \
  --output text
# Run this every 5 minutes during a full load.
# It's the production equivalent of watching a progress bar.
# If it stops increasing, DMS is stuck. Check the logs.

# Count tables still loading vs completed
aws dms describe-table-statistics \
  --replication-task-arn "arn:aws:dms:ap-south-1:123456789012:task:oracle-to-aurora-prod" \
  --region ap-south-1 \
  --query '{Loading: length(TableStatistics[?TableState==`Table loading`]), Done: length(TableStatistics[?TableState==`Table completed`])}' \
  --output json
# Watching the "Loading" count drop to 0 is genuinely satisfying at 2 AM.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Enable supplemental logging BEFORE creating the DMS task, not after.** If LogMiner or binlog doesn't have full supplemental logging, CDC will miss UPDATE and DELETE operations on tables without primary keys. You'll have a target DB with stale rows and no errors — the silent killer.
- **DMS CDC latency is NOT milliseconds — plan for seconds to minutes.** Under load, CDC can lag 30-60 seconds behind the source. Build this into your cutover plan: after stopping writes to the source, wait until CDC latency reaches zero before promoting the target.
- **LOBs (Large Objects / BLOBs / CLOBs) dramatically slow DMS.** Full LOB mode reads each LOB inline; limited LOB mode truncates above a threshold. Profile your LOB sizes before choosing the LOB setting — a 100MB BLOB in "inline" mode will stall DMS for seconds per row.
- **The SCT assessment report is optimistic.** Automated conversion percentages of 90%+ don't mean 90% of migration effort is done — complex PL/SQL packages with dynamic SQL, cursor sharing, or Oracle-specific types can take weeks to rewrite manually.
- **Pro Tip:** For cutover validation, run `SELECT COUNT(*)` and checksum queries on every table on both source and target after full load completes. `pt-table-checksum` (Percona) or AWS DMS data validation (built-in, set in task settings `ValidationSettings.EnableValidation: true`) catches row-level discrepancies automatically.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → Database Migration Service → Replication instances → Create replication instance`
2. **Sizing:** Choose instance class — `r5.2xlarge` for prod migrations >50GB. Enable Multi-AZ only if you need the replication instance to be HA (task restarts on failover regardless).
3. **Endpoints:** `DMS → Endpoints → Create endpoint`. Select engine, fill connectivity details, click "Test endpoint connection" before saving — saves you 20 minutes of debugging why the task won't start.
4. **Migration task:** `DMS → Database migration tasks → Create task`. Key settings:
   - **Migration type:** `Full load and change data capture (CDC)` for live migrations
   - **CDC stop position:** Leave blank for continuous; set only for point-in-time recovery scenarios
   - **Table mappings:** Use the wizard or paste JSON directly — JSON gives more control over schema name transformations (critical for Oracle UPPER_CASE → PostgreSQL lowercase)
5. **Monitoring:** After task starts, go to `Task → Table statistics` tab. Watch table states cycle: `Before load → Table loading → Table completed`. If any table stays in `Table error`, expand it — 95% of the time it's a missing permission or a type incompatibility that SCT didn't flag.

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS SCT | Converts schema DDL and stored procedures before DMS migrates data — run SCT first, always |
| AWS Migration Hub | Tracks DMS task progress alongside MGN server migrations — unified view of the whole wave |
| Amazon CloudWatch | DMS task logs and metrics (CDCLatencySource, CDCLatencyTarget, FullLoadThroughputRowsSource) are in CloudWatch — set alarms on latency |
| AWS Secrets Manager | Store source and target DB credentials; DMS can pull them directly via IAM role — no plaintext passwords |
| Amazon RDS / Aurora | Primary DMS migration targets — Aurora Serverless v2 is a popular target for on-prem databases migrating to managed |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
