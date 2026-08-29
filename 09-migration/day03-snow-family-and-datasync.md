# ☁️ Snow Family & AWS DataSync — AWS Mastery

> **When the data is too big for the network, you stop thinking about bandwidth and start thinking about a truck.**

---

## 📖 Concept

Every migration plan eventually runs the math: at a given available bandwidth, how long would it take to move a petabyte to S3? The answer is often "longer than the migration project's entire timeline," which is exactly the gap the AWS Snow Family closes. Snowball Edge (tens of TB to 80TB per device, with onboard compute) and Snowmobile (an actual shipping-container-sized data transfer truck, for exabyte-scale) let you physically ship storage hardware to AWS instead of pushing data over a wire. It sounds almost primitive next to the rest of AWS's networking story, but for genuinely large one-time bulk transfers, it remains the fastest and often cheapest option — "never underestimate the bandwidth of a truck full of hard drives" is a real engineering principle, not just a joke.

AWS DataSync solves an adjacent but different problem: ongoing or one-time online data transfer between on-premises storage (NFS, SMB, or an on-prem object store) and S3/EFS/FSx, with built-in verification, incremental sync, and bandwidth throttling — the kind of transfer that's large but not truck-large, and where you want the actual online path to be efficient rather than requiring physical hardware. DataSync agents run on-prem as a VM and only transfer changed data on subsequent syncs, which is what makes it viable for pre-cutover data validation runs during a migration, not just a single one-shot copy.

The decision framework I use on ProServe engagements: DataSync for anything under roughly 100TB with adequate bandwidth and where incremental resync matters before cutover; Snowball Edge for larger one-time transfers or environments with limited/unreliable connectivity; Snowmobile only for truly exabyte-scale data center migrations, which in practice is rare outside the largest enterprise engagements.

---

## 🏗️ Architecture Snapshot

```
DataSync (online, incremental):
┌─────────────────┐   NFS/SMB    ┌──────────────────┐   Encrypted   ┌─────────────┐
│  On-prem storage │──────────────▶│  DataSync Agent   │───────────────▶│  S3 / EFS / │
│                  │              │  (on-prem VM)      │   transfer    │  FSx        │
└─────────────────┘              └──────────────────┘               └─────────────┘
                                    Incremental, verified, throttled

Snow Family (offline, bulk):
┌─────────────────┐   Ship device  ┌──────────────────┐   Ingest at    ┌─────────────┐
│  On-prem data    │────────────────▶│  Snowball Edge /  │────────────────▶│  S3         │
│  center          │   (physically) │  Snowmobile        │   AWS DC       │             │
└─────────────────┘                └──────────────────┘                └─────────────┘
                                    Encrypted at rest with KMS during transit
```

---

## 💡 Real-World Use Cases

- **Data center exit under a hard deadline:** A customer closing a data center in 6 weeks with 200TB and limited uplink bandwidth uses Snowball Edge devices shipped in parallel to hit the deadline that online transfer couldn't meet.
- **Pre-cutover incremental sync:** DataSync runs nightly incremental syncs from on-prem NFS to S3 for weeks before a migration cutover, so the final cutover-day sync only needs to move a small delta.
- **Edge compute with limited connectivity:** Snowball Edge with compute is deployed to a remote site to preprocess and compress data locally before eventual shipment, when the site itself has minimal internet connectivity.

---

## 🔧 AWS CLI & Console Examples

### Create a Snowball Edge job

```bash
aws snowball create-job \
  --job-type IMPORT \
  --resources 'S3Resources=[{BucketArn=arn:aws:s3:::my-migration-bucket}]' \
  --snowball-type EDGE \
  --shipping-option NEXT_DAY \
  --region ap-south-1
```

### Create and start a DataSync task

```bash
aws datasync create-task \
  --source-location-arn arn:aws:datasync:ap-south-1:123456789012:location/loc-onprem \
  --destination-location-arn arn:aws:datasync:ap-south-1:123456789012:location/loc-s3 \
  --options 'VerifyMode=POINT_IN_TIME_CONSISTENT,TransferMode=CHANGED' \
  --region ap-south-1

aws datasync start-task-execution \
  --task-arn arn:aws:datasync:ap-south-1:123456789012:task/task-abc123 \
  --region ap-south-1
```

### Check DataSync task execution progress

```bash
aws datasync describe-task-execution \
  --task-execution-arn arn:aws:datasync:ap-south-1:123456789012:execution/exec-abc123 \
  --query '{Status:Status,BytesTransferred:BytesTransferred,FilesTransferred:FilesTransferred}'
```

---

## 🔐 Security Best Practices

- **Snowball Edge devices are encrypted with KMS keys you control** — the physical device is tamper-evident and encrypted, but ensure the KMS key policy is scoped correctly before shipping; a lost device without proper encryption controls is a real data-exposure risk.
- **Enable DataSync task verification (`VerifyMode`)** — always verify transferred data integrity, especially for the final pre-cutover sync where a silent corruption would only surface after the legacy system is decommissioned.
- **Scope IAM permissions for DataSync locations narrowly** — the IAM role backing an S3 location should only have access to the specific bucket/prefix involved in that migration, not broad S3 access.

---

## 😄 Funny Things to Try

```bash
# Check the shipping status of your Snowball Edge like a package
aws snowball describe-job --job-id JID1234567890 \
  --query '{Status:JobState,Shipping:ShippingDetails}'

# Watch a DataSync task's transfer rate in near-real-time
watch -n 5 'aws datasync describe-task-execution --task-execution-arn <arn> --query BytesTransferred'
```

---

## ⚠️ Gotchas & Tricky Bits

- **Snowball Edge has a hard capacity limit per device** — plan the number of devices needed up front; ordering one device and discovering mid-transfer it's not enough adds a full shipping round-trip to your timeline.
- **DataSync doesn't sync file permissions/ACLs perfectly across all protocol combinations** — verify how POSIX permissions map to S3 object metadata before relying on it for permission-sensitive workloads.
- **Snowmobile requires significant physical site prep** — power, network drop, and physical security requirements are non-trivial; this isn't a same-week engagement, plan months ahead.
- **Pro Tip:** Run DataSync in "verify only" mode against a completed Snowball transfer to reconcile that everything landed correctly — combining both tools for validation is common on large hybrid migrations.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → AWS Snow Family → Create job`
2. **Look for:** the device type selector — Snowball Edge Storage Optimized vs Compute Optimized, chosen based on whether preprocessing is needed on-site.
3. **Key field:** `KMS key` for the job — always set an explicit customer-managed key rather than relying on defaults for sensitive data.
4. **Common mistake here:** underestimating shipping lead time when planning a hard cutover date — always add buffer for shipping delays in both directions.
5. **Confirm with CLI:**
   ```bash
   aws snowball list-jobs --query 'JobListEntries[].[JobId,JobState]'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS MGN | Often used alongside Snow Family for compute migration while Snow handles bulk data |
| KMS | Encrypts Snowball devices and DataSync transfers in transit and at rest |
| S3 | The most common destination for both DataSync and Snow Family transfers |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
