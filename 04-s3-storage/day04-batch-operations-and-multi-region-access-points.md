# ☁️ S3 Batch Operations & Multi-Region Access Points — AWS Mastery

> **When you need to touch a billion objects, you don't write a loop — you write a manifest.**

---

## 📖 Concept

At small scale, changing metadata or ACLs on S3 objects is a script problem: list objects, loop, call the API, done. At migration scale — tens or hundreds of millions of objects inherited from a decommissioned on-prem NAS or a legacy S3 bucket with inconsistent tagging — that loop becomes a multi-day operation prone to throttling, partial failure, and no clean audit trail of what happened. S3 Batch Operations solves this by turning bulk object operations (copy, tag, restore from Glacier, invoke a Lambda per object, apply Object Lock retention, replace ACLs) into a managed job: you supply a manifest (a CSV or S3 Inventory report listing object keys), Batch Operations handles the parallelism, retries, and produces a completion report showing exactly which objects succeeded and which failed and why.

Multi-Region Access Points (MRAPs) solve a different but related problem: applications that need to read/write S3 data across multiple regions without hardcoding a bucket-region pairing into every client. An MRAP presents a single global endpoint that routes requests to the closest or most appropriate underlying bucket based on latency, with S3 Cross-Region Replication (CRR) keeping the underlying regional buckets in sync. This matters enormously for global migration engagements where a customer wants active-active resilience across, say, `ap-south-1` and `eu-west-1` without their application code knowing which region it's actually talking to.

Together, these two features answer the two hardest S3-at-scale questions ProServe gets asked: "how do I fix metadata on everything I already have" and "how do I make what I have resilient across regions without a rewrite."

---

## 🏗️ Architecture Snapshot

```
┌─────────────────────────────────────────────┐
│  S3 Batch Operations                                       │
│                                                              │
│  S3 Inventory report (CSV/Parquet manifest, millions of rows)│
│              │                                               │
│              ▼                                               │
│  ┌──────────────────────┐                                    │
│  │  Batch Job          │  --> Retag / Restore / Copy / Lambda │
│  │  (parallel workers) │                                    │
│  └──────────────────────┘                                    │
│              │                                               │
│              ▼                                               │
│   Completion report (success/failure per object)              │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Multi-Region Access Point (MRAP)                           │
│                                                              │
│   Application ──▶  mrap-alias.accesspoint.s3-global.amazonaws.com│
│                              │ (routes by latency/policy)   │
│                ┌────────────┼─────────────┐              │
│                ▼                          ▼              │
│     Bucket (ap-south-1)         Bucket (eu-west-1)             │
│            ▲─────────────── CRR ──────────────▼                     │
└─────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Legacy NAS migration cleanup:** After migrating 50 million files off an on-prem NAS into S3, run a Batch Operations job driven by an inventory manifest to apply standardized tags (`department`, `retention-class`) based on original folder path.
- **Bulk Glacier restore for an audit:** A compliance team needs 2 million archived objects restored to Standard for a 30-day review window — one Batch Operations restore job instead of 2 million individual API calls.
- **Global active-active application:** An application serving users in India and Europe reads/writes through a single MRAP endpoint, with CRR keeping both regional buckets consistent, so no client-side region logic is needed.

---

## 🔧 AWS CLI & Console Examples

### Create a Batch Operations job to retag objects from a manifest

```bash
aws s3control create-job \
  --account-id 111122223333 \
  --operation '{"S3PutObjectTagging":{"TagSet":[{"Key":"retention-class","Value":"7yr"}]}}' \
  --manifest '{"Spec":{"Format":"S3BatchOperations_CSV_20180820","Fields":["Bucket","Key"]},"Location":{"ObjectArn":"arn:aws:s3:::migration-inventory/manifest.csv","ETag":"abc123"}}' \
  --report '{"Bucket":"arn:aws:s3:::migration-reports","Prefix":"retag-job","Format":"Report_CSV_20180820","Enabled":true,"ReportScope":"AllTasks"}' \
  --priority 10 \
  --role-arn arn:aws:iam::111122223333:role/S3BatchOperationsRole

# Expected output:
# { "JobId": "abcd1234-ef56-7890-abcd-ef1234567890" }
```

### Bulk restore from Glacier via Batch Operations

```bash
aws s3control create-job \
  --account-id 111122223333 \
  --operation '{"S3InitiateRestoreObject":{"ExpirationInDays":30,"GlacierJobTier":"BULK"}}' \
  --manifest '{"Spec":{"Format":"S3BatchOperations_CSV_20180820","Fields":["Bucket","Key"]},"Location":{"ObjectArn":"arn:aws:s3:::migration-inventory/glacier-manifest.csv","ETag":"def456"}}' \
  --report '{"Bucket":"arn:aws:s3:::migration-reports","Prefix":"restore-job","Format":"Report_CSV_20180820","Enabled":true,"ReportScope":"AllTasks"}' \
  --priority 5 \
  --role-arn arn:aws:iam::111122223333:role/S3BatchOperationsRole
```

### Terraform — Multi-Region Access Point

```hcl
resource "aws_s3control_multi_region_access_point" "global" {
  details {
    name = "global-app-data"

    region {
      bucket = aws_s3_bucket.ap_south_1.id
    }
    region {
      bucket = aws_s3_bucket.eu_west_1.id
    }
  }
}
```

### Check a Batch Operations job's progress

```bash
aws s3control describe-job --account-id 111122223333 --job-id abcd1234-ef56-7890-abcd-ef1234567890 \
  --query 'Job.{Status:Status,Progress:ProgressSummary}'
```

---

## 🔐 Security Best Practices

- **Scope the Batch Operations IAM role tightly to the exact operation and bucket:** A role with blanket `s3:*` for a retagging job is far broader than the job needs — grant only `s3:PutObjectTagging` on the target bucket ARN.
- **Always enable job completion reports:** Without `Enabled: true` on the report config, a partially-failed job leaves you with no record of which objects didn't get processed.
- **Use Object Lock-aware operations carefully:** `S3PutObjectRetention` via Batch Operations can extend retention on millions of objects in one job — a single manifest error can have a very large, hard-to-reverse blast radius.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# See how many Batch Operations jobs you've quietly run and forgotten
aws s3control list-jobs --account-id 111122223333 --query 'length(Jobs)'

# Ask S3 how many objects are secretly still in Glacier Deep Archive
aws s3api list-objects-v2 --bucket my-bucket \
  --query "Contents[?StorageClass=='DEEP_ARCHIVE'] | length(@)"
# The answer is always "more than you remember archiving."
```

---

## ⚠️ Gotchas & Tricky Bits

- **Manifest ETag must match exactly:** The `ETag` in the manifest location must be the exact ETag of the manifest object at job creation time — if the manifest file changes after upload, job creation fails with a cryptic mismatch error.
- **Glacier BULK tier restores are cheap but slow:** Bulk retrieval can take 5–12 hours; if the audit deadline is tomorrow morning, pay for `Expedited` or `Standard` tier instead.
- **MRAP routing is NOT the same as bucket location:** An MRAP alias looks like a single bucket but silently routes to whichever region policy dictates — debugging "why did my write land in the wrong region" means checking the MRAP routing configuration, not the bucket itself.
- **Pro Tip:** Use S3 Inventory (Parquet format) instead of a hand-built CSV manifest for jobs over a few million objects — it's already in the exact schema Batch Operations expects and avoids manual manifest-generation bugs.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → S3 → Batch Operations → Create job`
2. **Look for:** The manifest source selector — "S3 Inventory report" vs "CSV".
3. **Key field:** `IAM role` — must have exactly the permissions the chosen operation needs, or job creation fails validation immediately.
4. **Common mistake here:** Forgetting to enable the completion report, leaving no audit trail for a job affecting millions of objects.
5. **Confirm with CLI:**
   ```bash
   aws s3control describe-job --account-id 111122223333 --job-id <job-id>
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| S3 Inventory | Supplies the ready-made manifest format Batch Operations consumes directly |
| AWS Lambda | Batch Operations can invoke a Lambda function per object for custom logic beyond built-in operations |
| Amazon S3 Cross-Region Replication | The mechanism keeping MRAP's underlying regional buckets synchronized |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
