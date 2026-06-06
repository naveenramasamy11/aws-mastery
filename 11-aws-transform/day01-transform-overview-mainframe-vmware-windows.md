# ☁️ AWS Transform — Mainframe, VMware & Windows Modernization — AWS Mastery

> **AWS Transform is what happens when AWS got tired of watching customers spend 5 years manually rewriting COBOL — so they built a machine to do it.**

---

## 📖 Concept

AWS Transform is AWS's automated modernization service that accelerates the conversion of legacy workloads to cloud-native equivalents. It currently covers three major modernization patterns: **Mainframe modernization** (COBOL/PL/I to Java on AWS), **VMware migration** (vSphere VMs to EC2 via automated conversion), and **Windows to Linux** (Windows workloads refactored to Linux-native equivalents).

The core of AWS Transform is automated code conversion powered by generative AI and static analysis. For mainframe workloads, Transform analyzes COBOL programs, understands business logic, and generates equivalent Java code that runs on Amazon EC2 or containers. This replaces years of manual re-platforming work. The output is not just a mechanical translation — the AI understands patterns like COBOL copybooks, JCL batch jobs, DB2 stored procedures, and CICS transaction flows.

The Transform engagement model is typically a **RAPID Assessment** first — a 2-4 week analysis of the source environment that produces a modernization roadmap, complexity scoring per application, and an estimated LOE (Level of Effort). For mainframe, this involves scanning COBOL source libraries, JCL, and data definitions. For VMware, it's discovery of vSphere clusters, VM inventory, and dependency mapping.

In ProServe engagements, AWS Transform is positioned as the third option alongside manual refactoring (expensive, slow) and lift-and-shift via MGN (fast but doesn't modernize). Transform occupies the middle ground: automated code conversion at a fraction of the manual cost, producing cloud-native output rather than just rehosted legacy code.

---

## 🏗️ Architecture Snapshot

```
  AWS Transform — Modernization Pathways
  ──────────────────────────────────────────────────────

  Mainframe Modernization:
  ┌────────────────────┐     Transform      ┌─────────────────────┐
  │  Legacy Mainframe  │  ──────────────▶   │  AWS Cloud Native   │
  │  - COBOL/PL1 code  │  AI-powered        │  - Java on EC2      │
  │  - JCL batch jobs  │  code conversion   │  - Lambda functions │
  │  - DB2 / VSAM      │                    │  - Aurora/DynamoDB  │
  │  - CICS online     │                    │  - Step Functions   │
  └────────────────────┘                    └─────────────────────┘

  VMware Migration:
  ┌────────────────────┐     Transform      ┌─────────────────────┐
  │  VMware vSphere    │  ──────────────▶   │  AWS EC2            │
  │  - VMs (any OS)    │  Automated         │  - EC2 instances    │
  │  - vCenter mgmt    │  conversion        │  - EBS volumes      │
  │  - vSAN storage    │                    │  - EKS (optional)   │
  └────────────────────┘                    └─────────────────────┘

  Windows to Linux:
  ┌────────────────────┐     Transform      ┌─────────────────────┐
  │  Windows Workload  │  ──────────────▶   │  Linux on AWS       │
  │  - .NET Framework  │  Code analysis     │  - .NET on Linux    │
  │  - SQL Server      │  + conversion      │  - Aurora PostgreSQL│
  │  - IIS             │                    │  - Nginx/ALB        │
  └────────────────────┘                    └─────────────────────┘

  RAPID Assessment Flow:
  [Discover] → [Analyze] → [Score complexity] → [Roadmap] → [Pilot]
     2 days      1 week         3 days            2 days     ongoing
```

---

## 💡 Real-World Use Cases

- **Bank mainframe modernization:** A large bank running 50M lines of COBOL for core banking — AWS Transform converts the highest-value transaction processing programs to Java microservices, reducing mainframe MIPS costs by 60% in Year 1.
- **VMware estate migration:** An enterprise with 500 VMs on vSphere 6.7 facing vSphere 8 license costs — Transform automates the discovery and migration planning, integrating with MGN for the actual rehost while identifying candidates for EKS containerization.
- **Windows Server 2012 EOL migration:** Thousands of Windows Server 2012 instances reaching end-of-life — Transform identifies which can be migrated to Linux (.NET 6+ apps) vs which need to stay Windows (Win32 dependencies), reducing Windows licensing costs by 40%.

---

## 🔧 AWS CLI & Console Examples

### AWS Mainframe Modernization Service

```bash
# List mainframe modernization environments
aws m2 list-environments \
  --region us-east-1 \
  --query 'environments[].{ID:id,Name:name,Engine:engineType,Status:status}' \
  --output table

# Create a runtime environment for converted COBOL apps (Blu Age engine)
aws m2 create-environment \
  --name my-mainframe-env \
  --instance-type M2.m5.xlarge \
  --engine-type bluage \
  --engine-version 3.7.0 \
  --subnet-ids subnet-0abc123 subnet-0def456 \
  --security-group-ids sg-0abc123 \
  --storage-configurations '[{
    "efs": {
      "fileSystemId": "fs-0abc123",
      "mountPoint": "/m2/mount"
    }
  }]' \
  --region us-east-1

# Deploy a converted application to the runtime environment
aws m2 create-application \
  --name converted-banking-app \
  --engine-type bluage \
  --definition '{
    "content": "version: 2.0\napplication: my-banking-app\nresources:\n  - type: java\n    s3Location: s3://my-transform-bucket/converted-app.zip"
  }' \
  --region us-east-1
```

### Transform Assessment Utilities

```bash
# Use AWS Application Discovery Service to inventory VMware VMs
aws discovery start-continuous-export \
  --region us-east-1

# List discovered servers (after ADS agent/connector deployment)
aws discovery describe-agents \
  --region us-east-1 \
  --query 'agentsInfo[].{AgentID:agentId,Hostname:agentNetworkInfoList[0].hostName,Status:health}' \
  --output table

# Export discovery data for Transform analysis
aws discovery start-export-task \
  --region us-east-1
```

### Migration Hub Integration

```bash
# List migration tasks tracked in Migration Hub
aws migrationhub list-migration-tasks \
  --region us-west-2 \
  --query 'MigrationTaskSummaryList[].{Task:MigrationTaskName,Tool:ProgressUpdateStream,Status:Status}' \
  --output table

# Get details on a specific application being transformed
aws migrationhub describe-application-state \
  --application-id d-application-0abc123 \
  --region us-west-2
```

---

## 🔐 Security Best Practices

- **Isolate Transform workloads in separate VPC:** The converted applications should land in a new, purpose-built VPC with proper segmentation — don't deploy transformed mainframe apps directly into your production VPC on day one.
- **Validate converted code in an isolated environment first:** Transform's AI-generated Java code should go through static analysis (SonarQube), security scanning (Amazon CodeGuru Security), and functional testing in a non-production environment before promotion.
- **Encrypt all source code and data in S3:** Upload COBOL source, JCL, and data files to an S3 bucket with SSE-KMS, VPC endpoint access only, and S3 Block Public Access. Source code is your most sensitive IP.
- **Use SCPs to prevent production deployment until validation gates pass:** In a Transform engagement, use AWS Config rules and SCPs to enforce that no converted application reaches production without passing through your test and security review stages.

---

## 😄 Funny Things to Try

```bash
# Check how many lines of COBOL a company had to deal with historically
# (Mainframe Modernization Service — list applications)
aws m2 list-applications \
  --region us-east-1 \
  --query 'applications[].{Name:name,Engine:engineType,Status:status,LastUpdated:lastStartTime}' \
  --output table
# If you've ever deployed a COBOL app: "I survived this"
# If you haven't: "why does this runtime environment have 40GB of RAM allocated"

# The VMware inventory reality check — how many VMs do we actually have?
aws discovery describe-agents \
  --region us-east-1 \
  --query 'length(agentsInfo)' \
  --output text
# Multiply by average 3-5 VMs per host = estimated VM count
# Then check vCenter... "oh that's where the other 200 went"

# Find Windows instances still running in your AWS account (pre-Transform targets)
aws ec2 describe-instances \
  --filters "Name=platform,Values=windows" \
             "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Name:Tags[?Key==`Name`].Value|[0]}' \
  --output table --region ap-south-1
# These are your Transform candidates for Windows-to-Linux.
```

---

## ⚠️ Gotchas & Tricky Bits

- **AWS Transform is not a magic "press button, get Java" machine:** The AI conversion produces a starting point — typically 60-80% functional code. The remaining 20-40% requires human review, especially for complex business rules, CICS screen layouts, and performance-critical paths.
- **Blu Age vs Micro Focus (COBOL runtime engines):** AWS Mainframe Modernization supports two engines: Blu Age (converts COBOL to Java — true refactor) and Micro Focus (runs COBOL natively on EC2 — replatform). They serve different use cases. Blu Age is for long-term modernization; Micro Focus is for faster migration with deferred conversion.
- **VMware Transform requires Application Discovery Service:** You can't use Transform for VMware without first deploying the ADS Discovery Connector to vCenter. This connector requires network access to all vCenter endpoints.
- **Windows-to-Linux compatibility matrix is narrow:** Transform supports .NET 4.5+ to .NET 6+ migration. Apps using Win32 APIs, COM objects, Windows Registry, or MSMQ cannot be automatically converted. Inventory carefully before estimating scope.
- **Pro Tip:** Always start a Transform engagement with a **pilot group** — pick 2-3 representative applications, run the full Transform cycle, and measure the quality/effort ratio. Use this to calibrate the rest of the program estimate. Customers who skip the pilot consistently underestimate the human review effort.

---

## 📸 Console Walkthrough

> *Setting up AWS Mainframe Modernization environment and deploying a converted app*

1. **Navigate to:** `AWS Console → AWS Mainframe Modernization`
2. **Create environment:** Click `Create runtime environment`
3. **Engine type:** Choose `Blu Age` (for converted Java) or `Micro Focus` (for native COBOL runtime)
4. **Instance type:** Start with `M2.m5.xlarge` for testing — scale up based on application TPS requirements
5. **Networking:** Choose the VPC and private subnets where the runtime will operate
6. **Key field:** `EFS mount point` — the runtime environment needs EFS for application artifacts storage
7. **Common mistake here:** Deploying the runtime in public subnets — mainframe workloads should always be in private subnets with ALB in front
8. **Deploy application:** `Applications → Create application → Upload your converted app package from S3`
9. **Confirm with CLI:**
   ```bash
   aws m2 list-deployments \
     --application-id app-0abc123 \
     --region us-east-1 \
     --query 'deployments[].{ID:deploymentId,Status:status,Created:creationTime}'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| MGN (Application Migration Service) | Complementary — MGN rehosted VMs can then be modernized via Transform |
| AWS Mainframe Modernization | Runtime environment for Blu Age (Java) and Micro Focus (COBOL) converted apps |
| Application Discovery Service | Inventory source for VMware and on-premises servers before Transform |
| Amazon CodeGuru Security | Security scanning for AI-generated code from Transform conversions |
| DMS | Database migration component — VSAM/DB2 to Aurora/DynamoDB alongside code conversion |
| Migration Hub | Centralized tracking of Transform progress alongside other migration tools |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
