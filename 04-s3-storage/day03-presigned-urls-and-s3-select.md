# ☁️ Presigned URLs & S3 Select — AWS Mastery

> **Give someone temporary access to one object without an IAM user, and let them query a 10GB file without downloading it — both are the same underlying idea: don't move more than you have to.**

---

## 📖 Concept

A recurring pattern across customer engagements: an application needs to let an end user upload or download a specific S3 object, but creating an IAM user or handing out long-lived credentials per end user doesn't scale and is a security nightmare. Presigned URLs solve this by letting a principal with S3 permissions generate a time-limited URL that embeds a signature proving "whoever holds this link is allowed to do this one operation, until this timestamp." The browser or client never needs AWS credentials at all — it just performs a normal HTTPS GET or PUT against the presigned URL, and S3 validates the signature server-side.

The mental model: the signature is computed using SigV4 against the *generating* principal's credentials, so the presigned URL only ever grants what that principal already had — it's a delegation mechanism, not a privilege escalation path. Expiration is enforced at the S3 service level regardless of client-side tampering, and once expired, the URL simply 403s.

S3 Select solves a different but related problem: querying a subset of a large CSV/JSON/Parquet object stored in S3 without downloading and parsing the whole file client-side. Send a SQL-like expression, and S3 filters and returns only the matching rows/columns, streamed back — which on large migration and data-lake engagements has cut both egress cost and processing time dramatically compared to pulling the entire object into Lambda or EC2 memory first.

---

## 🏗️ Architecture Snapshot

```
Presigned URL flow:
┌────────────┐  1. Request signed URL   ┌──────────────────┐
│  End User  │───────────────────────────▶│  Your Backend     │
│  (browser) │                           │  (holds IAM creds)│
└─────┬──────┘  2. Signed URL returned   └──────────────────┘
      │◀────────────────────────────────────────────┘
      │  3. PUT/GET directly to S3 (no AWS creds needed)
      ▼
┌────────────────────────────────────────────┐
│              Amazon S3                   │
│  Validates SigV4 signature + expiry      │
└────────────────────────────────────────────────┘

S3 Select flow:
┌──────────────┐   SQL expression    ┌─────────────────────────┐
│  Lambda/App  │──────────────────────▶│  S3 object (CSV/Parquet)│
│              │◀─────────────────────│  Filtered rows only     │
└──────────────┘  Streamed results   └─────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **User-uploaded profile photos without exposing AWS credentials:** The backend generates a presigned PUT URL scoped to one object key with a 5-minute expiry; the mobile app uploads directly to S3, bypassing the application server entirely for the heavy bytes.
- **Time-limited invoice downloads:** A finance portal generates presigned GET URLs valid for 15 minutes so a customer can download their own invoice PDF without a long-lived public link ever existing.
- **Filtering log archives in place:** An incident-response workflow uses S3 Select to pull only log lines matching a specific `request_id` out of a 50GB compressed CloudTrail archive, instead of downloading the whole object.

---

## 🔧 AWS CLI & Console Examples

### Generate a presigned URL (GET, 15-minute expiry)

```bash
aws s3 presign s3://my-bucket/invoices/inv-2026-001.pdf \
  --expires-in 900 \
  --region ap-south-1
```

### Generate a presigned PUT URL (Python/boto3, common in backends)

```python
import boto3
s3 = boto3.client('s3', region_name='ap-south-1')
url = s3.generate_presigned_url(
    'put_object',
    Params={'Bucket': 'my-bucket', 'Key': 'uploads/photo123.jpg', 'ContentType': 'image/jpeg'},
    ExpiresIn=300
)
```

### S3 Select — filter a CSV without downloading it

```bash
aws s3api select-object-content \
  --bucket my-data-lake \
  --key logs/access-2026-08.csv \
  --expression "SELECT * FROM s3object s WHERE s.status_code = '500'" \
  --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
  --output-serialization '{"CSV": {}}' \
  output.csv \
  --region ap-south-1
```

---

## 🔐 Security Best Practices

- **Set the shortest expiry that still works for the use case** — 5–15 minutes for uploads, not the default long windows some SDK examples show; a leaked presigned URL is only dangerous for as long as it's valid.
- **Always scope the generating IAM principal's own permissions tightly** — a presigned URL can never grant more than the generator already has, so an overly broad backend role makes every presigned URL it issues correspondingly riskier.
- **Use `Content-Type` and `Content-Length-Range` conditions on presigned PUTs** — prevents someone reusing a presigned upload URL to upload an unexpected file type or an oversized payload.

---

## 😄 Funny Things to Try

```bash
# Generate a presigned URL, then watch it die exactly on schedule
aws s3 presign s3://my-bucket/test.txt --expires-in 10
sleep 11 && curl -s -o /dev/null -w "%{http_code}\n" "<paste-url-here>"
# 403. Right on time. S3 does not do grace periods.

# S3 Select an aggregate without ever downloading the object
aws s3api select-object-content \
  --bucket my-data-lake --key big-file.csv \
  --expression "SELECT COUNT(*) FROM s3object" --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
  --output-serialization '{"CSV": {}}' count.csv
cat count.csv
```

---

## ⚠️ Gotchas & Tricky Bits

- **Presigned URL expiry is capped by the credentials behind it** — if you sign with temporary STS credentials (like an assumed role), the URL cannot outlive the credentials' own session, even if you request a longer `--expires-in`.
- **S3 Select only supports a subset of SQL and specific formats (CSV/JSON/Parquet)** — no joins, no subqueries; it's a filter/projection tool, not a query engine.
- **Presigned URLs don't survive a bucket policy that denies the action** — a restrictive bucket policy can override what the presigned URL otherwise allows, which surprises people who assume the signature alone is authoritative.
- **Pro Tip:** For very large or many-object transfers, presigned URLs are the wrong tool — use S3 Transfer Acceleration or multipart upload with presigned URLs per part instead of one giant presigned PUT.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → S3 → [bucket] → Objects → [object] → Object actions → Share with a presigned URL`
2. **Look for:** the expiry dropdown — max is 7 days when generated from a role, 12 hours for IAM user-based signatures using SigV4 with certain configurations.
3. **Key field:** the generated URL's `X-Amz-Expires` query parameter — confirms the actual TTL that was baked in.
4. **Common mistake here:** generating a presigned URL from the console (as an admin) for a workload meant to run unattended — always generate these programmatically from the actual application's scoped role.
5. **Confirm with CLI:**
   ```bash
   curl -I "<presigned-url>"
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| CloudFront Signed URLs | The CDN-layer equivalent for cached content, with additional geo/IP restrictions |
| IAM STS | Presigned URL lifetime is bounded by the temporary credentials used to sign it |
| Athena | The heavier-weight alternative when S3 Select's simple filtering isn't enough |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
