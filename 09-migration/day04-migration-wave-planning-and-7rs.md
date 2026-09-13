# ☁️ Migration Wave Planning & the 7 Rs Framework — AWS Mastery

> **The technology was never the hard part of migration. The sequencing is.**

---

## 📖 Concept

Every migration tool covered so far in this series — MGN, DMS, SCT, Snow Family — answers "how do I move a workload." Wave planning answers the much harder question: "in what order, and grouped how." Get the technology right and the sequencing wrong, and you still fail the engagement: a wave that migrates an application without also migrating the database it depends on breaks on day one; a wave sized too large overwhelms the customer's change-management capacity; a wave sized too small drags a migration out for years and bleeds budget on running duplicate infrastructure.

The 7 Rs framework (Rehost, Replatform, Repurchase, Refactor, Retire, Retain, Relocate) is the classification lens applied to every discovered application before it's ever assigned to a wave. Rehost ("lift and shift," typically via MGN) moves fastest with least risk but least modernization. Replatform makes minimal changes to gain a cloud benefit (e.g., moving to RDS instead of self-managed MySQL). Repurchase swaps for a SaaS alternative. Refactor is the highest-effort, highest-reward path — re-architecting for cloud-native patterns. Retire and Retain are equally important and chronically underused: retiring genuinely dead applications and explicitly deciding NOT to migrate certain workloads (yet, or ever) both reduce the total migration surface area, which is a bigger lever on schedule and cost than almost any technical optimization.

Wave planning then sequences the surviving applications using dependency mapping (via Application Discovery Service or a CMDB import) as the primary constraint, migration complexity as the secondary constraint, and business risk tolerance as the tie-breaker. The pattern that consistently works: front-load 1-2 "pathfinder" waves of low-risk, low-dependency applications to prove the migration factory's tooling and runbooks work end-to-end, then scale wave size up once the process is validated, then handle the highest-complexity, highest-dependency applications last, when the team has the most institutional experience with the customer's specific environment.

---

## 🏗️ Architecture Snapshot

```
┌────────────────────────────────────────────────────────────────────┐
│  Migration Wave Plan                                             │
│                                                                    │
│  Discovery (Application Discovery Service / CMDB import)          │
│              │                                                    │
│              ▼                                                    │
│  Application Portfolio ──▶ 7 Rs Classification per app            │
│              │                                                    │
│              ▼                                                    │
│  Dependency Graph (which apps talk to which databases/services)   │
│              │                                                    │
│              ▼                                                    │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐          │
│  │ Wave 1         │  │ Wave 2         │  │ Wave 3 (n)     │          │
│  │ (pathfinder,   │─▶│ (scaled,       │─▶│ (highest        │          │
│  │  low risk)     │  │  validated     │  │  complexity,    │          │
│  │                │  │  tooling)      │  │  most           │          │
│  │                │  │                │  │  dependencies)  │          │
│  └───────────────┘  └────────────────┘  └────────────────┘          │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Pathfinder wave validation:** A migration factory runs its first wave against 5 stateless, low-dependency web applications to validate MGN replication, cutover runbooks, and rollback procedures before committing to a 200-server program.
- **Dependency-aware sequencing:** An application and its database are always assigned to the same wave (or the database migrates first) — splitting them across waves is one of the most common causes of post-cutover outages.
- **Portfolio rationalization:** Applying Retire and Retain decisively during discovery can shrink a nominal 500-application portfolio to 380 applications that actually need active migration effort, materially compressing the program timeline.

---

## 🔧 AWS CLI & Console Examples

### Start Application Discovery Service data collection

```bash
aws discovery start-data-collection-by-agent-ids \
  --agent-ids AGENT-ID-1 AGENT-ID-2 AGENT-ID-3
```

### Export discovered application dependency data

```bash
aws discovery start-export-task \
  --export-data-format CSV \
  --filters name=inventoryType,values=CONNECTION
```

### Create a Migration Hub wave grouping (via Migration Hub Strategy Recommendations)

```bash
aws migrationhub-strategy get-portfolio-summary
# Returns portfolio-level R-type breakdown to inform wave sizing decisions
```

### Tag applications by wave assignment for tracking

```bash
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list arn:aws:ec2:ap-south-1:111122223333:instance/i-0123456789abcdef0 \
  --tags MigrationWave=Wave1,MigrationStrategy=Rehost
```

### Query MGN for wave cutover readiness

```bash
aws mgn describe-source-servers \
  --filters '{"tags":{"MigrationWave":["Wave1"]}}' \
  --query 'items[].{Server:sourceProperties.identificationHints.hostname,LagDuration:dataReplicationInfo.lagDuration}'
```

---

## 🔐 Security Best Practices

- **Apply consistent security baselines per wave, not per application:** Standardizing IAM roles, security groups, and logging configuration at the wave level (via a shared landing zone blueprint) prevents inconsistent security posture from creeping in application-by-application.
- **Validate IAM permissions for cross-account replication before wave kickoff:** A wave stalling because MGN's replication role lacks a needed permission is a common, avoidable schedule slip — validate this in the pathfinder wave, not wave 3.
- **Treat wave-level cutover runbooks as auditable artifacts:** Regulated customers often need documented evidence of change approval per wave — build this into the wave plan template from day one rather than retrofitting it later.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# Count how many "temporary" Retain decisions have quietly become permanent
aws resourcegroupstaggingapi get-resources \
  --tag-filters Key=MigrationStrategy,Values=Retain \
  --query 'length(ResourceTagMappingList)'
# The "we'll migrate that one later" pile is always bigger than remembered.

# Ask Migration Hub how confident it is your discovery data is complete
aws discovery describe-agents --query 'agentsInfo[].{Health:health,Hostname:hostName}'
```

---

## ⚠️ Gotchas & Tricky Bits

- **Dependency mapping from network flow data alone misses application-layer dependencies:** A shared file mount or an undocumented API call between two servers won't show up in NetFlow-style discovery — pair automated discovery with structured application-owner interviews.
- **Wave size is a change-management constraint, not just a technical one:** A customer's operations team can usually absorb far fewer simultaneous cutovers than the migration tooling could technically execute — plan wave cadence around the customer's actual operational capacity.
- **"Rehost now, refactor later" commitments frequently never happen:** Once an application is stable and running on EC2 post-rehost, the business case to circle back and refactor evaporates — if refactoring is truly required, consider doing it before migration, not as a deferred phase.
- **Pro Tip:** Build a rollback runbook for every wave with the same rigor as the cutover runbook — the pathfinder wave is exactly where you want to practice a rollback in a low-risk setting, before a high-complexity wave forces you to improvise one under pressure.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → Migration Hub → Migration Hub Strategy Recommendations → Portfolio`
2. **Look for:** The R-type distribution chart — a portfolio skewed heavily toward Rehost usually signals more refactoring opportunity was left unexplored.
3. **Key field:** `Complexity score` per application — cross-reference this against dependency count to decide wave placement, not complexity alone.
4. **Common mistake here:** Assigning wave order purely by "easiest first" without checking dependency chains — an easy application blocked on a hard one's database migration still can't cut over first.
5. **Confirm with CLI:**
   ```bash
   aws migrationhub-strategy get-application-component-strategies --application-component-id <id>
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Application Discovery Service | Supplies the dependency and utilization data wave planning is built on |
| AWS Application Migration Service (MGN) | Executes the actual server-level migration for Rehost-classified applications per wave |
| AWS Migration Hub | Centralizes tracking of wave status and R-type classification across the whole portfolio |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
