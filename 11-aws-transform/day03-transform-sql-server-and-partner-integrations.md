# ☁️ Transform for SQL Server Workloads & Partner Integrations — AWS Mastery

> **Modernizing a legacy .NET/SQL Server estate used to mean a multi-year rewrite; Transform is trying to compress that into months.**

---

## 📖 Concept

A huge share of enterprise application estates run on Windows and SQL Server — often licensed under aging, expensive Microsoft Enterprise Agreements, running on hardware nearing end-of-life, and built by teams that have long since moved on. The traditional modernization paths were binary and both painful: rehost as-is (keeping the licensing cost and Windows dependency) or commit to a from-scratch rewrite (multi-year, high-risk, frequently over budget). AWS Transform for .NET and SQL Server sits in between: it uses AI-assisted code analysis to assess a .NET Framework application's cross-platform compatibility, automatically ports significant portions of the codebase to .NET (cross-platform, Linux-compatible), and — for the database layer — assesses and assists SQL Server schema and stored procedure conversion toward Aurora PostgreSQL or Aurora MySQL via the Schema Conversion Tool's assessment engine.

The realistic expectation to set with customers is that Transform automates the mechanical, repetitive 60-80% of a conversion — namespace changes, API surface differences between .NET Framework and modern .NET, straightforward T-SQL to PostgreSQL/MySQL syntax translation — while the remaining, harder 20-40% (complex stored procedures with vendor-specific extensions, tightly-coupled COM interop, custom Windows-specific integrations) still requires engineering judgment. Selling Transform as "100% automated migration" sets an engagement up to disappoint; selling it as "dramatically compresses the mechanical labor so your engineers spend their time on the genuinely hard 20%" sets the right expectation and is also simply the accurate one.

Partner integrations matter here because AWS explicitly designed Transform to work alongside specialized ISVs rather than replace them entirely for the hardest cases — vendors like Micro Focus (mainframe-adjacent legacy transformation) and LzLabs round out coverage for workload types Transform's native tooling doesn't fully address. A realistic modernization program treats Transform as the first-pass accelerator and engages the right partner for the residual complexity Transform's assessment surfaces, rather than assuming one tool covers the entire estate.

---

## 🏗️ Architecture Snapshot

```
┌────────────────────────────────────────────────────────────────────┐
│  AWS Transform for .NET & SQL Server                              │
│                                                                    │
│  .NET Framework App + SQL Server DB (on-prem or EC2)               │
│              │                                                     │
│              ▼                                                     │
│  ┌────────────────────────────┐    ┌────────────────────────────┐        │
│  │  Code Assessment         │    │  Database Assessment       │        │
│  │  (AI-assisted analysis    │    │  (SCT-powered schema/proc   │        │
│  │   of cross-platform       │    │   compatibility scoring)    │        │
│  │   compatibility)          │    │                             │        │
│  └────────────┬─────────────┘    └─────────────┬──────────────┘        │
│              │                                │                     │
│              ▼                                ▼                     │
│  ┌────────────────────────────┐    ┌────────────────────────────┐        │
│  │  Automated .NET porting  │    │  Automated schema/SQL       │        │
│  │  (60-80% mechanical)      │    │  conversion (60-80%)        │        │
│  └────────────┬─────────────┘    └────────────┬─────────────┘        │
│              │                                │                     │
│              ▼                                ▼                     │
│  Remaining manual effort (complex stored procs, COM interop,      │
│  vendor-specific extensions) ── engaged via partner (Micro Focus, │
│  LzLabs) where Transform's native coverage stops                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **License cost avoidance:** A customer running 200 SQL Server instances under an expiring Enterprise Agreement uses Transform's assessment to prioritize which databases are realistic Aurora PostgreSQL conversion candidates, avoiding a costly EA renewal for that portion of the estate.
- **Windows-to-Linux compute savings:** A .NET Framework application ported to cross-platform .NET can run on Linux EC2 instances instead of Windows, removing Windows Server licensing costs entirely for that workload.
- **Phased modernization with partner backstop:** Transform's assessment identifies 70% of an application portfolio as good automated-conversion candidates, with the remaining 30% (complex, tightly-coupled legacy code) routed to a specialized modernization partner engagement.

---

## 🔧 AWS CLI & Console Examples

### Start a Transform assessment (via console-driven workflow, CLI check on progress)

```bash
aws transform list-projects --query 'projects[].{Name:name,Status:status}'
```

### Use SCT's assessment engine for SQL Server schema conversion scoring

```bash
# SCT assessment reports are generated via the SCT desktop application,
# but the resulting Aurora target schema can be validated via CLI:
aws rds describe-db-clusters \
  --db-cluster-identifier converted-aurora-pg \
  --query 'DBClusters[].{Engine:Engine,EngineVersion:EngineVersion}'
```

### Check .NET porting job status

```bash
aws transform get-application-component \
  --application-component-id app-comp-0123456789abcdef0 \
  --query '{Status:transformationStatus,Percentage:transformationProgress}'
```

### Terraform — target Aurora cluster for a converted SQL Server workload

```hcl
resource "aws_rds_cluster" "converted_target" {
  cluster_identifier = "converted-aurora-pg"
  engine             = "aurora-postgresql"
  engine_version     = "16.4"
  master_username    = "admin"
  master_password    = var.db_master_password
}
```

---

## 🔐 Security Best Practices

- **Never let converted stored procedures inherit `sa`-equivalent privileges on the target database:** A common mistake during SQL Server-to-PostgreSQL conversion is preserving overly broad legacy database roles instead of re-scoping to least privilege on the new engine.
- **Re-audit connection strings and secrets after .NET porting:** Ported applications frequently carry forward hardcoded connection strings from the original codebase — this is the right moment to migrate them to Secrets Manager instead of copying the anti-pattern forward.
- **Validate TLS/encryption settings explicitly on the new Aurora target:** SQL Server's default encryption configuration doesn't map 1:1 to Aurora's — confirm encryption-at-rest and in-transit settings are explicitly configured, not assumed inherited.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# See how many of your applications Transform thinks are "easy mode"
aws transform list-application-components \
  --query 'applicationComponents[?compatibilityScore>=`80`].{Name:name,Score:compatibilityScore}'
# The ones scoring below 50 are your actual multi-week engineering projects.

# Count how many SQL Server instances you're STILL running after "finishing" a migration
aws rds describe-db-instances --query 'DBInstances[?Engine==`sqlserver-ee` || Engine==`sqlserver-se`] | length(@)'
```

---

## ⚠️ Gotchas & Tricky Bits

- **Compatibility scores measure mechanical convertibility, not business logic correctness:** A high compatibility score means the code will compile and run on the target platform — it does NOT mean the business logic behaves identically; regression testing is still mandatory.
- **T-SQL to PostgreSQL conversion is not symmetric across features:** Certain T-SQL constructs (cursors, specific system functions, some transaction isolation behaviors) have no clean PostgreSQL equivalent and require manual rewrite, not automated translation.
- **.NET Framework-specific libraries (especially anything touching Windows Registry, COM, or WMI) cannot be automatically ported:** These require either a rewrite or accepting the application stays on Windows.
- **Pro Tip:** Run Transform's assessment early in the discovery phase, even before final wave planning — its compatibility scoring is genuinely useful input into 7 Rs classification (a low-scoring application is a stronger Rehost or Retain candidate than a Refactor candidate).

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → AWS Transform → Projects → Create project`
2. **Look for:** The application source selector — connect to a Git repository or upload source code directly for assessment.
3. **Key field:** `Target framework version` — pick the specific .NET version to port toward; picking too aggressive a target version can inflate the manual-effort estimate unnecessarily.
4. **Common mistake here:** Skipping the database assessment step and only running the code assessment — database conversion complexity is often the larger of the two efforts and needs equal upfront visibility.
5. **Confirm with CLI:**
   ```bash
   aws transform get-project --project-id proj-0123456789abcdef0 --query 'project.status'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Schema Conversion Tool (SCT) | Powers the database assessment and conversion scoring underneath Transform's SQL Server workflow |
| Amazon Aurora | The typical target engine for converted SQL Server databases |
| AWS Application Migration Service (MGN) | Handles the underlying compute migration for applications not selected for code transformation |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
