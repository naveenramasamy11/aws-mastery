# ☁️ S3 Replication, Object Lock & Access Points — AWS Mastery

> **Three features that turn S3 from "a bucket of files" into a compliance-grade, geo-distributed, multi-tenant storage platform.**

---

## 📖 Concept

### S3 Replication (CRR & SRR)

S3 replication asynchronously copies objects from a source bucket to one or more destination buckets. Cross-Region Replication (CRR) targets buckets in different AWS regions — the classic pattern for DR, latency reduction, and regulatory data residency. Same-Region Replication (SRR) targets buckets in the same region — used for log aggregation, test environment seeding from prod, or maintaining a separate compliance copy.

Key replication behaviours to know: only **new objects** after replication is enabled are replicated by default (use S3 Batch Replication to replicate existing objects). Delete markers are **not replicated** by default (you must enable delete marker replication explicitly). Replicated objects retain their encryption and storage class unless you override it in the replication rule.

Replication Time Control (RTC) adds a 99.99% SLA guarantee that objects are replicated within 15 minutes, with replication metrics visible in CloudWatch. Essential for RPO-sensitive workloads.

### S3 Object Lock

Object Lock implements WORM (Write Once Read Many) — once an object is locked, it cannot be deleted or overwritten for the lock duration. Two modes: **Governance** (users with `s3:BypassGovernanceRetention` can override the lock — useful for testing), and **Compliance** (no one, not even root, can delete before the retention period expires — required for SEC 17a-4 and similar regulations).

Object Lock has two protection levels: **Retention Periods** (time-based locks) and **Legal Holds** (indefinite locks, toggled on/off with `s3:PutObjectLegalHold`). Versioning is required — Object Lock only locks specific object versions, so deleting the bucket or the object version marker is blocked but adding a new version is still allowed.

### S3 Access Points

Access Points are named entry points to an S3 bucket with their own bucket-policy-equivalent (access point policy) and VPC binding capability. Instead of one monolithic bucket policy trying to handle 20 different team's access patterns, you create 20 access points — one per team — each with its own policy scoped to that team's prefixes.

The VPC binding is powerful for EKS: create an access point bound to your VPC and reference it in pod code via `arn:aws:s3:region:account:accesspoint/name`. Traffic never leaves the VPC even if the bucket is in a different account.

---

## 🏗️ Architecture Snapshot

```
S3 Multi-Region Setup with Replication + Access Points
─────────────────────────────────────────────────────────────────

  ap-south-1 (Primary)                us-east-1 (DR)
  ┌─────────────────────┐             ┌─────────────────────┐
  │  S3 Bucket: prod    │──── CRR ───▶│  S3 Bucket: dr-prod │
  │  Versioning: ON     │   (async,   │  Versioning: ON     │
  │  Object Lock: ON    │  <15 min    │  Object Lock: ON    │
  │                     │  with RTC)  │                     │
  │  Access Points:     │             └─────────────────────┘
  │  ┌─────────────────┐│
  │  │ ap/payments     ││◀── VPC-bound: only from payments VPC
  │  │ ap/analytics    ││◀── Public (analytics team)
  │  │ ap/audit        ││◀── VPC-bound: only from compliance VPC
  │  └─────────────────┘│
  └─────────────────────┘

  Object Lock Timeline:
  ┌──────────────────────────────────────────────────────────┐
  │  Object uploaded → Governance mode, 7yr retention       │
  │  ────────────────────────────────────────────────────    │
  │  Day 1    Day 30   Year 1   Year 7                       │
  │    │        │        │        │                          │
  │  Upload  Can't    Can't    Lock                          │
  │         delete  overwrite  expires                       │
  └──────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Compliance archive:** A financial services firm puts trade records in S3 with Compliance-mode Object Lock for 7 years to meet SEC 17a-4. Even a compromised root account cannot delete them. CRR copies them to a second region for DR — both copies are locked.
- **Migration staging:** During an AWS migration, CRR from an existing prod bucket to a new account's bucket validates data integrity before cutover. SRR seeds a test bucket from prod daily without manual processes.
- **Multi-team EKS data lake:** 15 data engineering teams write to the same S3 data lake bucket. 15 Access Points — one per team, each scoped to `data/team-<name>/` prefix, with VPC binding ensuring no cross-team prefix access. One bucket, zero bucket policy complexity.

---

## 🔧 AWS CLI & Console Examples

### Enable CRR Between Buckets

```bash
# Prerequisites: versioning enabled on both buckets
aws s3api put-bucket-versioning \
  --bucket source-prod-bucket \
  --versioning-configuration Status=Enabled \
  --region ap-south-1

aws s3api put-bucket-versioning \
  --bucket dr-prod-bucket \
  --versioning-configuration Status=Enabled \
  --region us-east-1

# Create replication configuration
cat > replication.json << 'EOF'
{
  "Role": "arn:aws:iam::123456789012:role/S3ReplicationRole",
  "Rules": [
    {
      "ID": "prod-to-dr",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "Destination": {
        "Bucket": "arn:aws:s3:::dr-prod-bucket",
        "StorageClass": "STANDARD_IA",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": { "Minutes": 15 }
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": { "Minutes": 15 }
        }
      },
      "DeleteMarkerReplication": {
        "Status": "Enabled"
      }
    }
  ]
}
EOF

aws s3api put-bucket-replication \
  --bucket source-prod-bucket \
  --replication-configuration file://replication.json \
  --region ap-south-1
```

### Replicate Existing Objects with S3 Batch Replication

```bash
# Create a batch replication job (replicates objects that existed before replication was enabled)
aws s3control create-job \
  --account-id 123456789012 \
  --operation '{"S3ReplicateObject": {}}' \
  --report '{"Bucket":"arn:aws:s3:::batch-reports-bucket","Format":"Report_CSV_20180820","Enabled":true,"Prefix":"batch-replication","ReportScope":"AllTasks"}' \
  --manifest-generator '{"S3JobManifestGenerator": {"SourceBucket": "arn:aws:s3:::source-prod-bucket", "EnableManifestOutput": false, "Filter": {"EligibleForReplication": true}}}' \
  --priority 10 \
  --role-arn arn:aws:iam::123456789012:role/S3BatchRole \
  --no-confirmation-required \
  --region ap-south-1
```

### Enable Object Lock (must be done at bucket creation)

```bash
# Object Lock MUST be enabled at bucket creation — cannot enable on existing bucket
aws s3api create-bucket \
  --bucket compliance-archive-2024 \
  --create-bucket-configuration LocationConstraint=ap-south-1 \
  --object-lock-enabled-for-bucket \
  --region ap-south-1

# Set default retention (applies to all new objects)
aws s3api put-object-lock-configuration \
  --bucket compliance-archive-2024 \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "GOVERNANCE",
        "Years": 7
      }
    }
  }' \
  --region ap-south-1

# Lock a specific object (override default)
aws s3api put-object-retention \
  --bucket compliance-archive-2024 \
  --key "trades/2024/q1-report.csv" \
  --retention '{"Mode": "COMPLIANCE", "RetainUntilDate": "2031-01-01T00:00:00Z"}' \
  --region ap-south-1

# Add a legal hold (indefinite, independent of retention period)
aws s3api put-object-legal-hold \
  --bucket compliance-archive-2024 \
  --key "trades/2024/q1-report.csv" \
  --legal-hold '{"Status": "ON"}' \
  --region ap-south-1
```

### Create S3 Access Points

```bash
# Create a VPC-bound access point for the payments team
aws s3control create-access-point \
  --name payments-ap \
  --account-id 123456789012 \
  --bucket source-prod-bucket \
  --vpc-configuration VpcId=vpc-0abc123def456 \
  --region ap-south-1

# Set the access point policy (scope to payments/ prefix only)
aws s3control put-access-point-policy \
  --name payments-ap \
  --account-id 123456789012 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/payments-app-role"
      },
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:ap-south-1:123456789012:accesspoint/payments-ap/object/payments/*"
    }]
  }' \
  --region ap-south-1
```

### Terraform — Replication + Object Lock

```hcl
resource "aws_s3_bucket" "source" {
  bucket = "prod-source-bucket"
}

resource "aws_s3_bucket_versioning" "source" {
  bucket = aws_s3_bucket.source.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_object_lock_configuration" "source" {
  bucket = aws_s3_bucket.source.id
  object_lock_enabled = "Enabled"
  rule {
    default_retention {
      mode  = "GOVERNANCE"
      years = 7
    }
  }
  depends_on = [aws_s3_bucket_versioning.source]
}

resource "aws_s3_bucket_replication_configuration" "to_dr" {
  bucket = aws_s3_bucket.source.id
  role   = aws_iam_role.replication.arn
  rule {
    id     = "replicate-all"
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.dr.arn
      storage_class = "STANDARD_IA"
    }
    delete_marker_replication { status = "Enabled" }
  }
  depends_on = [aws_s3_bucket_versioning.source]
}
```

---

## 🔐 Security Best Practices

- **Block public access at the account level.** Go to `S3 → Block Public Access settings for this account` and enable all four options. This prevents any bucket in the account from being accidentally made public, regardless of individual bucket settings.
- **Always use Compliance mode for regulatory data, Governance for operational data.** Governance mode lets you delete objects if needed (with the right permission) — useful for test/QA locks. Compliance mode should be reserved for data you legally cannot touch.
- **Bucket policy must allow the replication role.** The destination bucket needs a bucket policy allowing `s3:ReplicateObject` and `s3:ReplicateDelete` from the replication IAM role. Forgetting this causes silent replication failures.
- **Access Points don't bypass bucket policies.** The effective permissions for an access point request are the intersection of the access point policy AND the bucket policy. The bucket policy needs to either be permissive or explicitly allow the access point ARN.

---

## 😄 Funny Things to Try

```bash
# Check replication status on an object (did it actually replicate?)
aws s3api head-object \
  --bucket source-prod-bucket \
  --key "some-important-file.csv" \
  --region ap-south-1 \
  --query 'ReplicationStatus'
# Returns: "COMPLETED", "PENDING", "FAILED", or "REPLICA"
# "FAILED" appearing here is the fastest path to a bad Monday.

# Try to delete an Object Lock-protected object (spoiler: it won't work)
aws s3api delete-object \
  --bucket compliance-archive-2024 \
  --key "trades/2024/q1-report.csv" \
  --region ap-south-1
# Error: "An error occurred (AccessDenied) when calling the DeleteObject operation"
# This is exactly the point. Try to explain that to an impatient manager. 😄
```

---

## ⚠️ Gotchas & Tricky Bits

- **Object Lock cannot be enabled on an existing bucket.** It must be set at bucket creation. Many teams learn this the hard way when a compliance requirement arrives 2 years after the bucket was created. The workaround: create a new bucket with Object Lock, copy objects with S3 Batch Operations, update application configs, then delete the old bucket.
- **Replication does NOT replicate existing objects.** This is the #1 replication gotcha. Enable replication before your data arrives, or use S3 Batch Replication for existing objects. Silent failure: you enable CRR and assume historical data is replicated — it's not.
- **Versioning suspension doesn't delete versions.** If you suspend versioning on a bucket with Object Lock, existing locked versions remain locked. But new objects written while suspended don't get Object Lock applied. Only fully enabled versioning works with Object Lock.
- **Access Point ARN format in code.** When using Access Points in code, the ARN format is `arn:aws:s3:region:account:accesspoint/name` not the bucket name. The AWS SDK accepts both interchangeably as the `Bucket` parameter.
- **Pro Tip:** Use Multi-Region Access Points (MRAPs) when your application needs to read from the nearest S3 bucket automatically without any application-layer logic. MRAPs route `GetObject` requests to the nearest bucket with the object — ideal for globally distributed apps migrated to AWS with data locality requirements.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → S3 → [source bucket] → Management → Replication rules`
2. **Click:** "Create replication rule" → give it a name, choose scope (All objects vs prefix/tag filter)
3. **Key field:** "Replication Time Control" — enable this if you need the 15-minute SLA guarantee (costs extra per GB transferred)
4. **Common mistake here:** Forgetting to enable versioning on the destination bucket before setting up replication — the console warns you but many click past it
5. **For Object Lock:** Go to `S3 → Create bucket → Advanced settings → Object Lock → Enable` — once the bucket exists, this cannot be changed
6. **Verify replication metrics:**
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/S3 \
     --metric-name ReplicationLatency \
     --dimensions Name=SourceBucket,Value=source-prod-bucket Name=RuleId,Value=prod-to-dr \
     --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
     --period 300 \
     --statistics Average \
     --region ap-south-1
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Backup | Can back up S3 buckets (including Object Lock state) separately from replication — useful for accidental deletion scenarios |
| S3 Batch Operations | Required for retroactive replication of existing objects and bulk Object Lock application |
| AWS Macie | Scans S3 buckets for sensitive data (PII, credentials) — critical before enabling Cross-Account replication |
| CloudTrail Data Events | Log every S3 `GetObject`/`PutObject`/`DeleteObject` — combine with Object Lock to prove data integrity chain of custody |
| AWS Config | `s3-bucket-replication-enabled` and `s3-object-lock-enabled` managed rules detect compliance gaps across all buckets |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
