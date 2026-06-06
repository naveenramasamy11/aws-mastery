# 🏗️ AWS Mainframe Modernization & RAPID Assessment — AWS Mastery

> **"The COBOL is 40 years old. The business logic is irreplaceable. The mainframe bill is $4M/year. This is what an exit looks like."**

---

## 📖 Concept

### AWS Mainframe Modernization Service

AWS Mainframe Modernization (M2) is a managed service that enables you to migrate and modernize mainframe workloads to AWS — without requiring a complete rewrite. It supports two modernization patterns:

**Automated Refactoring** — M2 converts COBOL, PL/I, and JCL code to Java (via Micro Focus) or to Blu Age-generated modern Java microservices. The converted code runs on the M2 managed runtime environment, which emulates VSAM file systems, CICS transaction processing, and JES2 batch scheduling. You get cloud-native deployment without touching the application logic.

**Replatforming** — Run existing mainframe runtimes (Micro Focus Enterprise Server, LzLabs Software Defined Mainframe) on EC2 without code conversion. The applications run as-is; you just move the execution environment to AWS. Lower risk, faster timeline, but you're still paying COBOL license fees instead of getting a modern stack.

The M2 service manages the runtime environment (ECS-based), storage (via VSAM-to-EFS mapping), batch scheduling (using AWS Batch as the backend), and provides metrics and monitoring through CloudWatch.

### RAPID Assessment

RAPID (Rapid Assessment for Portfolio Inventory and Discovery) is AWS's methodology — delivered as a set of tools and a structured engagement model — for assessing a mainframe portfolio before committing to a migration path.

RAPID produces:
1. **Portfolio inventory** — automated discovery of programs, transactions, JCL procedures, datasets, and copybooks
2. **Complexity scoring** — each program rated by lines of code, COBOL constructs used, dependencies, and cross-program coupling
3. **Effort estimation** — working hours per program for automated conversion + manual remediation
4. **Modernization roadmap** — sequenced wave plan grouping programs by business function and dependency

The assessment typically takes 4-8 weeks for a mid-size mainframe (1M-5M lines of COBOL). The output directly feeds the M2 automated refactoring pipeline.

---

## 🏗️ Architecture Snapshot

```
AWS Mainframe Modernization — Target Architecture
──────────────────────────────────────────────────────────────────

Source (On-Prem Mainframe)            Target (AWS M2 Environment)
┌───────────────────────────┐         ┌───────────────────────────┐
│  IBM z/OS                 │         │  AWS M2 Runtime           │
│  ┌─────────────────────┐  │         │  ┌─────────────────────┐  │
│  │  CICS Transactions  │  │  RAPID  │  │  Blu Age / MF Java  │  │
│  │  COBOL Programs     │──│──────── │─▶│  Runtime (ECS)      │  │
│  │  JCL Batch Jobs     │  │  SCM    │  │  VSAM → EFS/S3      │  │
│  │  VSAM Datasets      │  │         │  │  JCL → AWS Batch    │  │
│  │  DB2 Database       │  │         │  │  DB2 → Aurora PG    │  │
│  └─────────────────────┘  │         │  └─────────────────────┘  │
└───────────────────────────┘         │                           │
                                      │  Supporting Services:     │
                                      │  ┌─────────────────────┐  │
                                      │  │ DMS (DB2→Aurora)    │  │
                                      │  │ AWS Batch (JCL)     │  │
                                      │  │ EFS (VSAM files)    │  │
                                      │  │ CloudWatch (metrics)│  │
                                      │  └─────────────────────┘  │
                                      └───────────────────────────┘

RAPID Assessment Flow:
  Week 1-2: Discovery
  ┌──────────────────────────────────────────────────────────┐
  │ Install RAPID discovery agent on mainframe               │
  │ Agent scans: PDS libraries, JCL, COBOL source, DB2 DDL   │
  │ Output: Program inventory CSV + dependency graph         │
  └──────────────────────────────────────────────────────────┘
  Week 3-4: Analysis
  ┌──────────────────────────────────────────────────────────┐
  │ M2 Assessment Tool scores each program                   │
  │ Complexity tiers: Simple / Medium / Complex / Excluded   │
  │ Excluded: programs using unsupported constructs          │
  └──────────────────────────────────────────────────────────┘
  Week 5-8: Roadmap
  ┌──────────────────────────────────────────────────────────┐
  │ Wave planning: group by business domain                  │
  │ Wave 1: Batch reporting (lowest risk)                    │
  │ Wave 2: Online transactions (CICS)                       │
  │ Wave 3: Core banking / highest complexity                │
  └──────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Bank COBOL modernization:** A retail bank with 3M lines of COBOL and 800 batch jobs uses RAPID assessment to discover that 65% of code is auto-convertible via Blu Age, 25% needs manual remediation (embedded assembler, unsupported CICS macros), and 10% is dead code. Wave 1 migrates overnight batch reporting — zero user impact. Waves 2-5 migrate online banking transactions with parallel run validation before cutover.
- **Insurance policy processing replatforming:** An insurer runs Micro Focus Enterprise Server on z/OS. RAPID identifies the code is Micro Focus-compatible — replatforming (not refactoring) is the fastest path. The COBOL runs on M2 with Micro Focus runtime on AWS within 6 months. No code changes, mainframe decommissioned, 60% infrastructure cost reduction.
- **Government batch modernization:** A government agency migrates 400 JCL batch jobs to AWS Batch. RAPID maps JCL dependencies, identifies JCL procedures that can be converted to Step Functions, and produces a topological sort of batch execution order for wave planning.

---

## 🔧 AWS CLI & Console Examples

### Create an M2 Runtime Environment

```bash
# Create a Blu Age runtime environment (for automated refactoring path)
aws m2 create-environment \
  --name "mainframe-prod" \
  --description "Production Blu Age runtime for banking COBOL" \
  --engine-type bluage \
  --engine-version "3.7.0" \
  --instance-type M2.m5.xlarge \
  --high-availability-config '{
    "desiredCapacity": 2
  }' \
  --subnet-ids '["subnet-aaa", "subnet-bbb"]' \
  --security-group-ids '["sg-0abc123def456"]' \
  --storage-configurations '[{
    "efs": {
      "fileSystemId": "fs-0abc123",
      "mountPoint": "/m2/mount"
    }
  }]' \
  --tags '{"Environment": "prod", "Project": "mainframe-migration"}' \
  --region ap-south-1

# Check environment status
aws m2 get-environment \
  --environment-id "env-0abc123def456" \
  --region ap-south-1 \
  --query '{Status: status, EngineType: engineType, InstanceType: instanceType}' \
  --output json
```

### Deploy an Application to M2

```bash
# Create an application definition (points to S3 artifact)
aws m2 create-application \
  --name "banking-batch" \
  --description "Core banking batch application" \
  --engine-type bluage \
  --definition '{
    "content": "{\"definition\":{\"listeners\":[{\"port\":8196,\"type\":\"http\"}],\"ba-application\":{\"app-location\":\"s3://my-m2-artifacts/banking-batch-v1.zip\"}}}"
  }' \
  --tags '{"Application": "banking-batch"}' \
  --region ap-south-1

# Deploy application to environment
aws m2 create-deployment \
  --application-id "app-0abc123def456" \
  --application-version 1 \
  --environment-id "env-0abc123def456" \
  --region ap-south-1

# List deployments and check status
aws m2 list-deployments \
  --application-id "app-0abc123def456" \
  --region ap-south-1 \
  --query 'deployments[*].[deploymentId,status,applicationVersion]' \
  --output table
```

### Run a Batch Job

```bash
# Start a batch job (JCL-equivalent in M2)
aws m2 start-batch-job \
  --application-id "app-0abc123def456" \
  --batch-job-identifier '{
    "fileBatchJobIdentifier": {
      "fileName": "MONTHLYRPT",
      "folderPath": "/m2/jobs"
    }
  }' \
  --job-params '{
    "REPORT_DATE": "20240601",
    "OUTPUT_PATH": "/m2/output/monthly"
  }' \
  --region ap-south-1

# List batch job executions
aws m2 list-batch-job-executions \
  --application-id "app-0abc123def456" \
  --region ap-south-1 \
  --query 'batchJobExecutions[*].[executionId,status,startTime,endTime]' \
  --output table

# Get job details (for debugging a failed run)
aws m2 get-batch-job-execution \
  --application-id "app-0abc123def456" \
  --execution-id "exe-0abc123def456" \
  --region ap-south-1
```

### Monitor M2 Environment via CloudWatch

```bash
# Check M2 metrics (active transactions, batch job queue depth)
aws cloudwatch get-metric-statistics \
  --namespace "AWS/M2" \
  --metric-name "ActiveConnections" \
  --dimensions Name=ApplicationId,Value=app-0abc123def456 \
  --start-time "$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ)" \
  --end-time "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --period 300 \
  --statistics Average \
  --region ap-south-1

# Create alarm for batch job failures
aws cloudwatch put-metric-alarm \
  --alarm-name "m2-batch-job-failures" \
  --namespace "AWS/M2" \
  --metric-name "BatchJobFailures" \
  --dimensions Name=ApplicationId,Value=app-0abc123def456 \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --datapoints-to-alarm 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-south-1:123456789012:mainframe-ops \
  --region ap-south-1
```

### Terraform — M2 Environment

```hcl
resource "aws_m2_environment" "prod" {
  name         = "mainframe-prod"
  engine_type  = "bluage"
  engine_version = "3.7.0"
  instance_type = "M2.m5.xlarge"

  high_availability_config {
    desired_capacity = 2
  }

  network_type       = "dual"
  subnet_ids         = var.private_subnet_ids
  security_group_ids = [aws_security_group.m2.id]

  storage_configuration {
    efs {
      file_system_id = aws_efs_file_system.m2_vsam.id
      mount_point    = "/m2/mount"
    }
  }

  tags = {
    Environment = "prod"
    Project     = "mainframe-modernization"
  }
}

resource "aws_efs_file_system" "m2_vsam" {
  creation_token = "m2-vsam-storage"
  encrypted      = true
  kms_key_id     = aws_kms_key.m2.arn

  lifecycle_policy {
    transition_to_ia = "AFTER_30_DAYS"
  }
}

resource "aws_m2_application" "banking_batch" {
  name        = "banking-batch"
  engine_type = "bluage"

  definition {
    content = jsonencode({
      definition = {
        ba-application = {
          app-location = "s3://${aws_s3_bucket.m2_artifacts.bucket}/banking-batch-v1.zip"
        }
      }
    })
  }
}
```

---

## 🔐 Security Best Practices

- **Use VPC-only M2 environments.** M2 environments should never be publicly accessible — all CICS transaction traffic comes through an ALB or NLB in front of the runtime. The M2 listener port (8196 default) should only be accessible from within the VPC.
- **Encrypt VSAM-equivalent EFS storage.** The EFS file system attached to M2 for VSAM dataset emulation must use KMS encryption. For financial workloads, use a customer-managed KMS key and enable EFS access points with POSIX user enforcement per application.
- **IAM role separation for M2 service roles.** M2 requires a service role (`aws-service-role-for-m2`) for control plane operations, plus an execution role for the application. The execution role should have least-privilege access to S3 artifacts, EFS mounts, and Secrets Manager (for DB2→Aurora connection strings). Never use the same role for both.
- **Audit COBOL converted code before production.** Automated refactoring produces Java code, but the conversion isn't guaranteed to be functionally identical for all edge cases (packed decimal arithmetic, sign handling, REDEFINES clauses). Mandate a parallel run period (run legacy and converted code simultaneously, compare outputs) before decommissioning the mainframe.

---

## 😄 Funny Things to Try

```bash
# List all M2 environments in your account — and their costs
aws m2 list-environments \
  --region ap-south-1 \
  --query 'environments[*].[name,status,instanceType,engineType]' \
  --output table
# M2.m5.xlarge runs ~$0.50/hr — dramatically cheaper than $200/MIPS on a mainframe.
# The ROI calculation practically writes itself.

# Check how many batch jobs ran today
aws m2 list-batch-job-executions \
  --application-id "app-0abc123def456" \
  --region ap-south-1 \
  --query 'length(batchJobExecutions[?status==`Succeeded`])' \
  --output text
# Every successful batch job here represents a mainframe MSU (million service units) NOT consumed.
# Your finance team will want this number. Show them.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Not all COBOL is convertible by Blu Age.** Programs using IBM-specific macros, 390 assembler inlines, CICS SYNCPOINT with rollback, or IDMS/IMS database calls are not automatically handled. RAPID assessment surfaces these — they require manual rewrite, stay on legacy, or use a different pattern. Don't start the M2 project without running RAPID first.
- **VSAM-to-EFS mapping has performance implications.** VSAM KSDS (keyed) files are emulated over EFS with a proprietary indexing layer. Sequential batch read throughput is typically fine; random-access CICS transactions against large VSAM datasets can exhibit higher latency than on-mainframe. Benchmark with your actual data volumes during the parallel run.
- **M2 batch scheduling is not JES2.** JCL dependencies, GDG (Generation Data Groups), and SYSOUT disposition are emulated but not perfectly. Complex JCL with STEPLIB concatenations, conditional COND parameters, and in-stream data may require remediation even on the "replatforming" path.
- **Parallel run is non-negotiable for financial workloads.** Running legacy and converted code in parallel — comparing output datasets byte-for-byte — is the only way to prove correctness before cutover. Build parallel run automation into your migration plan from day one. It typically extends the project by 2-4 months but prevents post-cutover disasters.
- **Pro Tip:** The M2 "Micro Focus" engine path and the "Blu Age" path are architecturally different. Micro Focus replatforms existing compiled COBOL (near-zero code change); Blu Age refactors to Java (requires source code). Confirm which path applies to your workload before sizing the project — they have different timelines, costs, and license implications.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → AWS Mainframe Modernization → Environments → Create environment`
2. **Engine selection — the critical choice:**
   - `Blu Age` — if you have COBOL source code and want automated Java conversion
   - `Micro Focus` — if you have existing MF-compiled code or need minimal code change
3. **Instance type:** `M2.m5.xlarge` is a good start for dev/test. Production typically runs `M2.m5.4xlarge` or larger with HA (2+ instances). M2 instance types are separate from regular EC2 types — don't confuse them.
4. **Storage:** Attach EFS for VSAM dataset emulation. The mount point `/m2/mount` is the default — M2 maps VSAM dataset names to EFS paths automatically via the runtime configuration.
5. **Applications:** `M2 → Applications → Create application` → upload your converted artifact ZIP to S3 → reference the S3 path in the application definition JSON. Deploy to the environment → M2 handles the ECS task scheduling.
6. **Parallel run monitoring:** After deployment, navigate to `Applications → [App Name] → Batch Job Executions` to see job run history, status, and output. Compare against mainframe output using a checksum comparison tool — AWS provides integration with Step Functions for this.

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS DMS + SCT | Migrate the DB2 or VSAM-backed relational data to Aurora PostgreSQL in parallel with the COBOL migration |
| AWS Batch | Used by M2 as the underlying batch scheduler — complex JCL dependencies are modelled as Batch job queues and job definitions |
| Amazon EFS | VSAM dataset emulation storage — EFS Elastic mode handles the variable throughput of batch vs online workloads |
| AWS Migration Hub | Track M2 application deployments and batch job validation alongside other migration streams in a unified dashboard |
| AWS Transform | The broader umbrella program that includes mainframe modernization alongside VMware and Windows Server migration — RAPID is part of the Transform assessment toolkit |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
