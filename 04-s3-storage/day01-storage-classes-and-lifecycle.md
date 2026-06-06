# ☁️ S3 Storage Classes & Lifecycle Management — AWS Mastery

> **S3 is not a bucket — it's a tiered cost machine you forgot to configure.**

---

## 📖 Concept

Amazon S3 offers multiple storage classes designed for different access patterns and cost profiles. The default class, S3 Standard, gives 99.99% availability and is optimised for frequently accessed data — but it's also the most expensive per GB. Most workloads have data that ages: logs from yesterday are accessed hourly, logs from last year are accessed never. Without lifecycle policies, you're paying Standard prices for cold data indefinitely.

S3 Intelligent-Tiering (S3-IT) is the "set it and forget it" answer for unpredictable access patterns. It monitors access patterns per object and automatically moves objects between Frequent Access, Infrequent Access, Archive Instant Access, Archive Access, and Deep Archive tiers. There's a small monitoring charge per object (around $0.0025 per 1,000 objects/month) but zero retrieval fees — making it ideal for data lakes, ML datasets, and long-term logs where access patterns are unknown.

Lifecycle policies let you define rules that transition or expire objects automatically. A common pattern for application logs: transition to S3-IA after 30 days, to Glacier Instant Retrieval after 90 days, delete after 365 days. This can reduce storage costs by 60–80% vs leaving everything in Standard.

Cross-Region Replication (CRR) copies objects asynchronously to a bucket in another region. It's used for disaster recovery, compliance (data residency requirements), and latency reduction. Same-Region Replication (SRR) copies to another bucket in the same region — useful for log aggregation, dev/test data sharing, or keeping a versioned copy in a separate account.

---

## 🏗️ Architecture Snapshot

```
  S3 Storage Classes — Cost vs Access Speed
  ─────────────────────────────────────────────────────

  Access      ┌───────────────────────────────────────┐
  Frequency   │  S3 Standard          ($0.023/GB)     │ ← Hot data
     ▲        │  Frequent access, <ms latency          │
     │        ├───────────────────────────────────────┤
     │        │  S3 Standard-IA       ($0.0125/GB)    │ ← Warm data
     │        │  ≥30 day min, retrieval fee            │
     │        ├───────────────────────────────────────┤
     │        │  S3 Glacier Instant   ($0.004/GB)     │ ← Cool data
     │        │  ms retrieval, archive use cases       │
     │        ├───────────────────────────────────────┤
     │        │  S3 Glacier Flexible  ($0.0036/GB)    │ ← Cold data
     │        │  minutes to hours retrieval            │
     │        ├───────────────────────────────────────┤
  Low │        │  S3 Glacier Deep Archive ($0.00099/GB)│ ← Frozen data
     ▼        │  12h retrieval, compliance/archive     │
              └───────────────────────────────────────┘

  Lifecycle Policy Flow:
  Day 0 ──▶ Standard ──▶ [30d] ──▶ Standard-IA ──▶ [90d] ──▶ Glacier ──▶ [365d] DELETE
```

---

## 💡 Real-World Use Cases

- **Application log cost reduction:** A fintech company keeping 2TB of daily application logs in S3 Standard — adding a lifecycle policy (IA at 30d, Glacier at 90d, delete at 365d) reduces monthly storage cost from ~$46 to ~$8.
- **ML training dataset management:** Large ML datasets accessed heavily during training runs then rarely touched — S3 Intelligent-Tiering automatically moves them to Archive Access tier and back, without manual intervention.
- **Cross-account DR replication:** CRR from production account's S3 bucket to a separate DR account bucket, with different KMS keys — meets compliance requirements for data isolation between environments.

---

## 🔧 AWS CLI & Console Examples

### Checking Object Storage Classes

```bash
# List all objects in a bucket with their storage class
aws s3api list-objects-v2 \
  --bucket my-app-logs \
  --query 'Contents[].{Key:Key,Class:StorageClass,SizeGB:Size}' \
  --output table \
  --region ap-south-1

# Find all Standard objects larger than 1GB (candidates for tiering)
aws s3api list-objects-v2 \
  --bucket my-app-logs \
  --query 'Contents[?StorageClass==`STANDARD` && Size>`1073741824`].Key' \
  --output text
```

### Creating a Lifecycle Policy

```bash
# Apply a lifecycle policy via CLI
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-app-logs \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "log-tiering-and-expiry",
      "Status": "Enabled",
      "Filter": {"Prefix": "logs/"},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"},
        {"Days": 180, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 365}
    }]
  }'

# Verify the policy was applied
aws s3api get-bucket-lifecycle-configuration \
  --bucket my-app-logs
```

### Enabling S3 Intelligent-Tiering

```bash
# Enable S3 Intelligent-Tiering with Archive tiers
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket my-data-lake \
  --id full-tiering-config \
  --intelligent-tiering-configuration '{
    "Id": "full-tiering-config",
    "Status": "Enabled",
    "Tierings": [
      {"Days": 90, "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'
```

### Cross-Region Replication Setup

```bash
# Enable versioning first (required for CRR)
aws s3api put-bucket-versioning \
  --bucket source-bucket \
  --versioning-configuration Status=Enabled

# Create replication configuration
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789012:role/s3-replication-role",
    "Rules": [{
      "ID": "replicate-all",
      "Status": "Enabled",
      "Filter": {},
      "Destination": {
        "Bucket": "arn:aws:s3:::dest-bucket-us-east-1",
        "StorageClass": "STANDARD_IA"
      }
    }]
  }'
```

### Block Public Access (Always Enable This)

```bash
# Block all public access — do this on every bucket you create
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true

# Enforce at account level (blocks all new public buckets)
aws s3control put-public-access-block \
  --account-id 123456789012 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,\
    BlockPublicPolicy=true,RestrictPublicBuckets=true
```

---

## 🔐 Security Best Practices

- **Block Public Access at the account level:** Enable all four Block Public Access settings at the S3 account level, not just per-bucket. This prevents any developer from accidentally making a new bucket public.
- **Use S3 Object Lock for compliance data:** WORM (Write Once Read Many) protection prevents deletion or overwriting for a specified retention period. Required for SEC 17a-4, CFTC, and HIPAA compliance.
- **Encrypt with SSE-KMS, not SSE-S3:** SSE-S3 uses AWS-managed keys with no audit trail. SSE-KMS gives you KMS key policies, CloudTrail logging of every decrypt operation, and customer-controlled key rotation.
- **Presigned URLs expire — set short TTLs:** Default presigned URL TTL can be up to 7 days (for IAM role-based credentials). For sharing sensitive documents, set TTL to 15 minutes and regenerate on demand.

---

## 😄 Funny Things to Try

```bash
# The "how much am I actually storing?" reality check
aws s3 ls s3://my-bucket --recursive --human-readable --summarize 2>/dev/null | tail -2
# The summary at the bottom: Total Objects and Total Size
# Surprise yourself. Then look at your AWS bill.

# Find your largest S3 objects across ALL buckets (top 10)
for bucket in $(aws s3api list-buckets --query 'Buckets[].Name' --output text); do
  aws s3api list-objects-v2 --bucket $bucket \
    --query 'Contents[].[Size,Key]' --output text 2>/dev/null
done | sort -rn | head -10 | awk '{printf "%.2f GB  %s\n", $1/1073741824, $2}'
# The "what is taking up all my space?" audit.

# Check if you have any buckets with versioning DISABLED (danger zone)
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  tr '\t' '\n' | \
  xargs -I{} bash -c 'status=$(aws s3api get-bucket-versioning --bucket {} \
    --query "Status" --output text 2>/dev/null); \
    [ "$status" != "Enabled" ] && echo "NO VERSIONING: {}"'
# Buckets with no versioning = no recovery if someone runs `aws s3 rm --recursive`
```

---

## ⚠️ Gotchas & Tricky Bits

- **Minimum storage duration charges:** S3-IA charges a minimum of 30 days even if you delete the object on day 2. Glacier Flexible = 90 days, Deep Archive = 180 days. Moving small, frequently changing files to these classes can cost MORE than Standard.
- **Lifecycle transitions don't apply retroactively to existing objects:** When you add a lifecycle rule, it applies going forward. To transition existing objects, you need S3 Batch Operations or a script.
- **S3 Replication doesn't replicate existing objects:** CRR only replicates new objects after the rule is enabled. Use S3 Batch Replication to copy pre-existing objects.
- **Versioning cannot be disabled once enabled:** You can only suspend it. Suspended versioning still keeps all previous versions (and you pay for them). Use lifecycle rules to expire non-current versions.
- **Pro Tip:** Enable S3 Storage Lens at the organization level — it gives you a free dashboard showing storage distribution by class, replication status, encryption coverage, and cost efficiency metrics across all accounts and buckets in your Org.

---

## 📸 Console Walkthrough

> *Setting up a lifecycle policy to move logs to Glacier*

1. **Navigate to:** `AWS Console → S3 → [bucket-name] → Management tab`
2. **Click:** `Create lifecycle rule`
3. **Rule name:** Something meaningful like `logs-tiering-365d`
4. **Filter:** Set prefix to `logs/` to scope the rule
5. **Transitions:** Add three transitions — 30d to Standard-IA, 90d to Glacier Instant Retrieval, 180d to Glacier Deep Archive
6. **Expiration:** Enable `Expire current versions` at 365 days
7. **Key field:** Enable `Delete expired delete markers` to clean up versioning clutter
8. **Common mistake here:** Forgetting to enable for both current AND non-current versions when versioning is on — you end up keeping every old version forever
9. **Verify with CLI:**
   ```bash
   aws s3api get-bucket-lifecycle-configuration \
     --bucket my-app-logs \
     --query 'Rules[].{ID:ID,Status:Status}'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| CloudTrail | Logs all S3 API calls (data events) — enable for sensitive buckets |
| Macie | ML-based PII/sensitive data detection in S3 objects |
| KMS | Customer-managed encryption keys for SSE-KMS |
| Athena | Query S3 data directly with SQL — no ETL required |
| S3 Storage Lens | Organization-wide visibility into storage metrics and costs |
| EventBridge | S3 event notifications trigger Lambda, SQS, or SNS on object operations |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
