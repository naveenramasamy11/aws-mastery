# ☁️ IMDSv2 Enforcement, EC2 Image Builder & AMI Lifecycle — AWS Mastery

> **The metadata service was designed under 1990s trust assumptions; IMDSv2 is AWS finally patching that.**

---

## 📖 Concept

The Instance Metadata Service (IMDS) is how an EC2 instance discovers its own identity — instance ID, IAM role temporary credentials, user data script, network configuration — by calling `http://169.254.169.254` from inside the instance. For years this endpoint required no authentication (IMDSv1): any process on the box, including code executing through a server-side request forgery (SSRF) vulnerability in a web application, could simply curl that address and walk out with the instance's IAM role credentials. This was the exact mechanism behind the 2019 Capital One breach, where a misconfigured WAF allowed SSRF that reached IMDS and exfiltrated credentials for an over-permissioned role.

IMDSv2 closes this with a session-oriented, token-based handshake: a `PUT` request fetches a token (with a hop-limit header that most SSRF proxies never forward), and subsequent `GET` requests must include that token. A classic SSRF exploit — which typically issues only simple `GET` requests with attacker-controlled URLs — usually cannot forge the initial `PUT`, so IMDSv2 alone kills a huge class of exploitation without any application code changes. Enforcing IMDSv2 fleet-wide is one of the highest-leverage, lowest-effort security wins available on any EC2 assessment.

The other half of this topic is AMI lifecycle management — standardizing how golden images get built, hardened, tested, versioned, shared, and eventually deprecated. EC2 Image Builder automates this: it takes a base AMI, applies hardening components (CIS benchmarks, patch baselines, agent installs), runs validation tests, and publishes a new AMI version on a schedule, so nobody hand-rolls AMIs with undocumented scripts. Combined with AMI deprecation and cross-account sharing via AWS Organizations, this turns AMI sprawl into a managed, auditable pipeline — every migration wave launches from an approved, dated AMI instead of whatever the source server happened to be running.

---

## 🏗️ Architecture Snapshot

```
┌────────────────────────────────────────────────────────────────┐
│  EC2 Image Builder Pipeline                                     │
│                                                                   │
│  Base AMI → Build Component → Test Component → Distribution      │
│  (Amazon    (CIS hardening,    (Inspector scan,  (share to        │
│   Linux)     patch, agents)     smoke tests)      accounts)       │
│                                                        │           │
│                                                        ▼           │
│                                          ┌─────────────────────┐  │
│                                          │  New AMI version    │  │
│                                          │  (tagged, dated)    │  │
│                                          └─────────────────────┘  │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  IMDSv2 Handshake (per instance)                                  │
│                                                                     │
│  App Process       PUT /latest/api/token  (hop-limit: 1)           │
│  inside EC2  ────────────────────────────▶  IMDS (169.254.169.254) │
│               ◀──────────────────────────   returns session token  │
│                                                                     │
│  GET /latest/meta-data/iam/... + token header                      │
│  ─────────────────────────────▶ credentials returned only w/ token │
│                                                                     │
│  SSRF exploit (attacker GET only, no PUT) → BLOCKED                 │
└────────────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Post-incident hardening:** After any SSRF finding in a customer's web application, mandate IMDSv2 fleet-wide as an immediate mitigating control while the code fix ships.
- **Migration factory golden images:** Every migration wave launches from an Image Builder-produced AMI with CIS hardening baked in, so security review happens once per pipeline run, not once per server.
- **Regulated workload compliance:** Financial services customers require documented, versioned AMI provenance — Image Builder's build history and SBOM output satisfies this directly.

---

## 🔧 AWS CLI & Console Examples

### Enforce IMDSv2 on a running instance (retroactive)

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-0123456789abcdef0 \
  --http-tokens required \
  --http-put-response-hop-limit 1 \
  --region ap-south-1

# Expected output includes:
# "HttpTokens": "required"   <- this is the enforcement flag
```

### Enforce IMDSv2 account-wide for all NEW instances

```bash
aws ec2 modify-instance-metadata-defaults \
  --http-tokens required \
  --http-put-response-hop-limit 2 \
  --http-endpoint enabled \
  --region ap-south-1

# Sets the region-level default so every future launch is
# IMDSv2-only unless a launch template explicitly overrides it.
```

### Audit which instances are still on IMDSv1

```bash
aws ec2 describe-instances \
  --filters "Name=metadata-options.http-tokens,Values=optional" \
  --query 'Reservations[].Instances[].[InstanceId,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

### Create an Image Builder pipeline (Terraform)

```hcl
resource "aws_imagebuilder_image_pipeline" "golden_al2023" {
  name                             = "golden-al2023-pipeline"
  image_recipe_arn                 = aws_imagebuilder_image_recipe.al2023.arn
  infrastructure_configuration_arn = aws_imagebuilder_infrastructure_configuration.default.arn
  distribution_configuration_arn   = aws_imagebuilder_distribution_configuration.multi_account.arn

  schedule {
    schedule_expression                = "cron(0 3 ? * SUN *)"
    pipeline_execution_start_condition = "EXPRESSION_MATCH_ONLY"
  }
}
```

### Deprecate an old AMI (stop new launches without deleting it)

```bash
aws ec2 deprecate-image \
  --image-id ami-0123456789abcdef0 \
  --deprecate-at "2026-12-31T00:00:00"
```

---

## 🔐 Security Best Practices

- **Set `http-tokens required` at the account/region default level, not just per launch template:** New instances launched by anyone inherit the safer default automatically.
- **Set a hop-limit of 1 unless you have a real reason not to:** A hop-limit of 2+ is required for containerized workloads that proxy IMDS requests, but it also widens the SSRF attack surface — only raise it when confirmed necessary.
- **Bake CIS hardening into the Image Builder recipe, not a post-launch script:** Anything applied post-launch can be skipped under time pressure; anything baked into the golden AMI cannot.
- **Sign and version every AMI with immutable tags:** `pipeline-run-id`, `source-commit`, and `build-date` tags turn "which AMI is this?" from a Slack thread into a `describe-images` call.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# Ask an instance who it thinks it is (only works from inside the instance)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
# If this returns a role name with NO token required, you're still on
# IMDSv1 and should fix that today, not tomorrow.

# Count how many AMIs you personally own and have quietly forgotten about
aws ec2 describe-images --owners self --query 'length(Images)'
# The number is always bigger than you think.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Enforcing IMDSv2 can break legacy SDKs:** Very old AWS SDK versions don't speak the token protocol and will silently fail to fetch credentials — test on a canary instance before a fleet-wide rollout.
- **`modify-instance-metadata-defaults` only affects future launches:** Existing running instances need `modify-instance-metadata-options` applied individually or via SSM Run Command at scale.
- **Deprecated AMIs are NOT deleted:** `deprecate-image` only blocks new launches by default users; an admin with explicit AMI ID can still launch from a deprecated AMI unless also restricted via SCP.
- **Pro Tip:** Combine AMI deprecation with an AWS Config custom rule checking `metadata-options.http-tokens` to get continuous compliance instead of a one-time audit.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → EC2 → Instances → select an instance → Actions → Modify instance metadata options`
2. **Look for:** The "IMDSv2" radio button set — it defaults to "Optional" on older instances.
3. **Key field:** `IMDSv2` — set to `Required` because "Optional" still allows the vulnerable v1 path to work alongside v2.
4. **Common mistake here:** Leaving the hop limit at the default of 1 when running containerized workloads that need the extra hop — causing container tasks to silently fail to fetch credentials.
5. **Confirm with CLI:**
   ```bash
   aws ec2 describe-instances --instance-ids i-0123456789abcdef0 \
     --query 'Reservations[].Instances[].MetadataOptions'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Systems Manager | Runs fleet-wide `modify-instance-metadata-options` across hundreds of instances at once |
| Amazon Inspector | Scans Image Builder-produced AMIs for CVEs before distribution |
| AWS Config | Provides continuous compliance checking that IMDSv2 enforcement hasn't drifted |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
