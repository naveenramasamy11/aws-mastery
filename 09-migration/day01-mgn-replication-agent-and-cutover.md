# ☁️ AWS MGN — Application Migration Service Deep Dive — AWS Mastery

> **MGN is lift-and-shift done right — continuous replication, cutover in minutes, no proprietary agents that break your app.**

---

## 📖 Concept

AWS Application Migration Service (MGN), formerly CloudEndure Migration, is AWS's primary service for rehosting (lift-and-shift) workloads from on-premises or other clouds to AWS. MGN uses continuous block-level replication to keep a synchronized copy of your source servers in AWS — at cutover, the lag between source and target is typically under a minute.

The MGN workflow has four phases: **Install** (deploy the replication agent on source servers), **Replicate** (continuous block-level sync to staging area in AWS), **Test** (launch test instances, validate application behavior without interrupting source), **Cutover** (launch production instances, finalize replication, decommission source). The staging area uses low-cost EC2 instances and EBS volumes — you only pay for full-sized target instances during test and cutover windows.

Migration waves are how you organize which servers to cut over together. A typical ProServe engagement groups servers by application dependency — all servers in a microservice group cut over in the same wave. Wave planning is where most migration projects fail: cutting over a database before its application, or an application before its load balancer, causes outages.

For the 7 Rs of migration (Retire, Retain, Rehost, Replatform, Repurchase, Refactor, Relocate), MGN handles Rehost. For Replatform, you typically use MGN to get to AWS first, then optimize (switch to RDS, enable EKS, etc.) after the initial migration. "Lift, then shift the lift" is a common real-world pattern.

---

## 🏗️ Architecture Snapshot

```
  AWS MGN Migration Architecture
  ─────────────────────────────────────────────────────────

  SOURCE ENVIRONMENT              AWS TARGET ENVIRONMENT
  (On-Premises / Other Cloud)

  ┌─────────────────────┐         ┌───────────────────────────┐
  │  Source Server      │         │  AWS Account               │
  │  (Any OS/workload)  │         │                           │
  │  ┌───────────────┐  │  TCP    │  ┌────────────────────┐   │
  │  │ MGN Repl.     │──┼──1500──▶│  │ Replication Server │   │
  │  │ Agent         │  │ (443)   │  │ (staging area)     │   │
  │  └───────────────┘  │         │  └─────────┬──────────┘   │
  └─────────────────────┘         │            │               │
                                  │  ┌─────────▼──────────┐   │
                                  │  │ EBS Staging Volumes│   │
                                  │  │ (compressed, enc.) │   │
                                  │  └─────────┬──────────┘   │
                                  │            │               │
                                  │  Test / Cutover:           │
                                  │  ┌─────────▼──────────┐   │
                                  │  │ Target EC2 Instance │   │
                                  │  │ (converted to AWS  │   │
                                  │  │  drivers/boot)     │   │
                                  │  └────────────────────┘   │
                                  └───────────────────────────┘

  MGN Phases:
  [Install Agent] → [Replication] → [Testing] → [Cutover] → [Decommission]
         │               │               │           │
      minutes         continuous     non-disruptive  final
                      replication    validation      lag <1min
```

---

## 💡 Real-World Use Cases

- **Large-scale factory migration:** Migrating 200+ servers in waves of 20-30 per weekend window — MGN's continuous replication means each cutover takes 5-10 minutes, not hours, enabling tight migration windows.
- **Payment gateway rehost:** Migrating PCI-DSS in-scope servers with minimal downtime using MGN test launches to validate compliance configuration in AWS before final cutover.
- **Cloud-to-cloud migration:** Moving workloads from Azure or GCP to AWS — MGN agent supports any OS, making it cloud-agnostic for source.

---

## 🔧 AWS CLI & Console Examples

### MGN Setup and Server Management

```bash
# Initialize MGN service in a region (first-time setup)
aws mgn initialize-service --region ap-south-1

# List all source servers being replicated
aws mgn describe-source-servers \
  --region ap-south-1 \
  --query 'items[].{ID:sourceServerID,Hostname:sourceProperties.identificationHints.hostname,State:dataReplicationInfo.dataReplicationState,LagMinutes:dataReplicationInfo.lagDuration}' \
  --output table

# Get replication status for a specific server
aws mgn describe-source-servers \
  --region ap-south-1 \
  --filters '{"sourceServerIDs": ["s-0abc123def456789"]}' \
  --query 'items[0].dataReplicationInfo'
```

### Launch Test and Cutover

```bash
# Launch test instances (non-disruptive — source keeps running)
aws mgn start-test \
  --source-server-ids '["s-0abc123def456789", "s-0def456789abc012"]' \
  --region ap-south-1

# Mark test as successful (required before cutover)
aws mgn finish-test \
  --source-server-ids '["s-0abc123def456789"]' \
  --region ap-south-1

# Start cutover (last sync + launch production instances)
aws mgn start-cutover \
  --source-server-ids '["s-0abc123def456789"]' \
  --region ap-south-1

# Mark cutover as complete (stops replication, finalizes)
aws mgn finish-cutover \
  --source-server-ids '["s-0abc123def456789"]' \
  --region ap-south-1
```

### Launch Configuration (instance type, networking)

```bash
# Update launch template for a source server
aws mgn update-launch-configuration \
  --source-server-id s-0abc123def456789 \
  --launch-disposition STARTED \
  --target-instance-type-right-sizing-method BASIC \
  --ec2-launch-template-id lt-0abc123def456789 \
  --region ap-south-1

# Check current launch settings
aws mgn get-launch-configuration \
  --source-server-id s-0abc123def456789 \
  --region ap-south-1
```

---

## 🔐 Security Best Practices

- **Use a dedicated replication security group:** MGN replication traffic goes from agent to replication servers on port 1500 (TCP). Create a security group that allows only this port from your source IP ranges, not 0.0.0.0/0.
- **Enable EBS volume encryption for staging:** All staging volumes should be encrypted with KMS. Set this in the MGN Replication Settings before starting replication.
- **IAM least privilege for MGN agent:** The MGN agent needs a specific IAM user or role. Use the AWS-provided `AWSApplicationMigrationAgentInstallationPolicy` — don't give it AdministratorAccess.
- **Test cutover before the real thing:** Always do at least one test cutover per server group. The test launch is non-disruptive and reveals networking/application issues before the production window.

---

## 😄 Funny Things to Try

```bash
# Check how many servers are still in "initial sync" (the "are we there yet?")
aws mgn describe-source-servers \
  --region ap-south-1 \
  --query 'items[?dataReplicationInfo.dataReplicationState==`INITIAL_SYNC`].{Server:sourceServerID,Progress:dataReplicationInfo.dataReplicationInitiation.steps[-1].status}' \
  --output table
# Initial sync on a 10TB server: "not there yet. come back tomorrow."

# The "how much data am I replicating?" sanity check
aws mgn describe-source-servers \
  --region ap-south-1 \
  --query 'items[].{Server:sourceServerID,ReplicatedGB:dataReplicationInfo.replicatedStorageBytes,TotalGB:dataReplicationInfo.dataReplicationInitiation.steps[0].nextAttemptDateTime}' \
  --output table

# Find servers that have been in "stalled" replication
aws mgn describe-source-servers \
  --region ap-south-1 \
  --query 'items[?dataReplicationInfo.dataReplicationState==`STALLED`].sourceServerID' \
  --output text
# STALLED = network issue, agent problem, or source server rebooted. Investigate!
```

---

## ⚠️ Gotchas & Tricky Bits

- **Boot conversion takes time at cutover:** MGN converts the source OS boot drivers to AWS-compatible ones (virtio, ENA). For Windows, this can take 3-5 minutes after instance launch before the OS boots. Plan for this in your cutover window.
- **Antivirus blocks the MGN agent:** Windows Defender and third-party AV often block the MGN replication agent. Add exclusions for `C:\Program Files (x86)\AWS Replication Agent\` before installation.
- **MGN replication uses the public replication server endpoint by default:** If your source environment has no internet access, you need to configure MGN to use a private replication server (via VPN or Direct Connect). Check this before installing agents.
- **Test instances count against EC2 limits:** During testing, full-sized EC2 instances are launched. If you're testing 50 servers simultaneously, ensure your EC2 vCPU service limits are sufficient.
- **Pro Tip:** Use MGN Application Groups to track which source servers belong to the same application. This makes wave planning visual and ensures you cut over interdependent servers together. Navigate to `MGN → Applications` in the console.

---

## 📸 Console Walkthrough

> *Installing the MGN agent and monitoring replication progress*

1. **Navigate to:** `AWS Console → Application Migration Service → Get started`
2. **Replication settings:** Configure staging subnet, replication security group, and EBS encryption before adding any servers
3. **Agent installation:** Navigate to `Source servers → Add servers → Show installation instructions`
4. **Copy the agent installer command:** It includes your account-specific token — run this on each source server
5. **Monitor:** `Source servers` tab shows replication state: `Not started → Initial sync → Continuous replication`
6. **Key metric:** `Replication lag` — should be under 1 minute for servers in continuous replication state
7. **Common mistake here:** Not configuring the launch template (instance type, subnet, security groups) before test cutover — the test will use defaults which may not match your target architecture
8. **Confirm replication health with CLI:**
   ```bash
   aws mgn describe-source-servers --region ap-south-1 \
     --query 'items[].{Host:sourceProperties.identificationHints.hostname,State:dataReplicationInfo.dataReplicationState,Lag:dataReplicationInfo.lagDuration}'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Migration Hub | Central tracking dashboard for MGN migrations across multiple regions |
| DMS | Database migrations — complement to MGN for replatforming database engines |
| Schema Conversion Tool (SCT) | Convert Oracle/SQL Server schemas to Aurora PostgreSQL/MySQL |
| CloudEndure DR | Disaster recovery for mission-critical workloads — sister product to MGN |
| AWS DataSync | File-based data migration for NAS/NFS shares alongside server replication |
| AWS Transform | Automated modernization after initial rehost via MGN |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
