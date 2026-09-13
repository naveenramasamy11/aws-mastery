# ☁️ IAM Access Analyzer & Cross-Account Trust Validation — AWS Mastery

> **The tool that finds the S3 bucket policy you forgot about — before an attacker does.**

---

## 📖 Concept

Every large-scale migration engagement eventually hits the same moment: a customer asks "which of our resources are reachable from outside our AWS Organization?" and nobody has a confident answer. IAM policies, S3 bucket policies, KMS key policies, Secrets Manager resource policies, SQS queue policies, Lambda resource policies, and IAM role trust policies all grant cross-account access independently, and each one is a place where "just add my account for testing" silently becomes permanent. IAM Access Analyzer exists to close that gap by continuously reasoning over every resource-based policy in an account or organization and flagging any that grant access to a principal outside a defined "zone of trust."

What makes Access Analyzer different from a linter is that it uses automated reasoning (a form of formal, mathematical logic verification built on Zelkova, the same engine behind AWS's policy validation tooling) rather than pattern matching. It doesn't just look for `"Principal": "*"` — it actually evaluates the full policy, including conditions, to determine whether an external principal could genuinely access the resource under any circumstance. This means it correctly ignores a wildcard principal that's locked down by an `aws:SourceArn` condition, and correctly flags a policy that looks safe on the surface but has a logic hole in a `NotPrincipal` clause.

In migration and modernization work, Access Analyzer earns its keep in two very different moments. During a migration wave, it's how you validate that newly provisioned cross-account roles (for MGN replication, DMS endpoints, or Landing Zone account vending) aren't accidentally broader than intended. Post-migration, it's how you run continuous drift detection — because trust boundaries erode over time as engineers add "temporary" access that never gets removed. I've seen findings surface access granted 14 months earlier for a one-day debugging session that nobody remembered to revoke.

The other half of Access Analyzer — the "custom policy checks" and "unused access" features — deserves equal attention. Unused access analysis flags IAM roles and permissions that haven't been exercised in 90+ days, which is the single most effective lever for shrinking a customer's blast radius without asking anyone to guess what "least privilege" should look like from scratch.

---

## 🏗️ Architecture Snapshot

```
┌──────────────────────────────────────────────────────────┐
│                     AWS Organization                              │
│                                                                    │
│  ┌────────────────────┐        ┌───────────────────────┐    │
│  │   Security/Audit        │        │   Member Account (Prod)  │    │
│  │   Account                │        │                          │    │
│  │                          │        │  ┌────────────────────┐  │    │
│  │  ┌──────────────────┐  │        │  │ S3 Bucket Policy    │  │    │
│  │  │ Access Analyzer     │◀─┼────────┼──│ Principal: acct-999 │  │    │
│  │  │ (Organization scope)│  │ scans  │  └────────────────────┘  │    │
│  │  └─────────┬────────┘  │        │  ┌────────────────────┐  │    │
│  │            │              │        │  │ IAM Role Trust      │  │    │
│  │            ▼              │        │  │ Policy (external)   │  │    │
│  │  ┌──────────────────┐  │        │  └────────────────────┘  │    │
│  │  │ Findings:           │  │        │  ┌────────────────────┐  │    │
│  │  │ - External access   │  │        │  │ KMS Key Policy       │  │    │
│  │  │ - Unused access      │  │        │  └────────────────────┘  │    │
│  │  └─────────┬────────┘  │        └─────────────────────┐    │
│  │            │              │                                       │
│  │            ▼              │                                       │
│  │  EventBridge → SNS/Slack   │                                       │
│  │  (findings notification)   │                                       │
│  └────────────────────┘                                       │
└─────────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Landing Zone validation:** After a Control Tower account-vending run, sweep every new account with organization-scoped Access Analyzer to confirm no default cross-account trust leaked in from a copied CloudFormation template.
- **Migration wave sign-off:** Before cutting over a migration wave, run Access Analyzer against the target account to prove the MGN/DMS replication roles only trust the exact source-account ARNs they need — not `*`.
- **Continuous compliance evidence:** Feed Access Analyzer findings into Security Hub so auditors get a running log of external-access findings and their resolution time, instead of a point-in-time manual review.

---

## 🔧 AWS CLI & Console Examples

### Create an organization-level analyzer

```bash
# Run this from the Security/Audit (delegated admin) account
aws accessanalyzer create-analyzer \
  --analyzer-name org-external-access \
  --type ORGANIZATION \
  --region ap-south-1

# Expected output:
# {
#   "arn": "arn:aws:access-analyzer:ap-south-1:111122223333:analyzer/org-external-access"
# }
```

### List and triage active findings

```bash
aws accessanalyzer list-findings-v2 \
  --analyzer-arn arn:aws:access-analyzer:ap-south-1:111122223333:analyzer/org-external-access \
  --filter '{"status":{"eq":["ACTIVE"]}}' \
  --region ap-south-1

# Each finding includes resourceType, isPublic, principal, and condition —
# isPublic:true means literally anyone on the internet, not just another AWS account.
```

### Archive a known-good finding (so it stops re-alerting)

```bash
aws accessanalyzer update-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-south-1:111122223333:analyzer/org-external-access \
  --status ARCHIVED \
  --ids "12345678-90ab-cdef-1234-567890abcdef"
```

### Run unused-access analysis (the underrated feature)

```bash
aws accessanalyzer create-analyzer \
  --analyzer-name org-unused-access \
  --type ORGANIZATION_UNUSED_ACCESS \
  --configuration '{"unusedAccess":{"unusedAccessAge":90}}' \
  --region ap-south-1

# Surfaces IAM roles/users with permissions unused for 90+ days —
# gold for a least-privilege remediation backlog.
```

### Terraform — analyzer with EventBridge notification

```hcl
resource "aws_accessanalyzer_analyzer" "org" {
  analyzer_name = "org-external-access"
  type          = "ORGANIZATION"
}

resource "aws_cloudwatch_event_rule" "analyzer_findings" {
  name        = "access-analyzer-new-findings"
  description = "Route new Access Analyzer findings to Slack"
  event_pattern = jsonencode({
    source      = ["aws.access-analyzer"]
    detail-type = ["Access Analyzer Finding"]
  })
}
```

---

## 🔐 Security Best Practices

- **Scope at the organization level, not per-account:** A single delegated-admin analyzer covering the whole Organization catches drift you'd otherwise need hundreds of per-account analyzers to see.
- **Treat "public" and "cross-account" findings differently in your SLA:** A truly public S3 bucket is a page-someone-now event; a cross-account finding to a known partner account might just need documentation. Don't let the noise from the second category bury the first.
- **Wire findings into Security Hub, not just email:** Security Hub gives you finding aggregation, suppression rules, and a real remediation workflow — a mailbox full of JSON does not.
- **Run unused-access analysis before every major IAM cleanup project:** It turns "let's review all our roles" from a guessing exercise into a prioritized, evidence-backed list.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# Find out if you've accidentally made yourself internet-famous
aws accessanalyzer list-findings-v2 \
  --analyzer-arn <your-analyzer-arn> \
  --filter '{"isPublic":{"eq":["true"]}}' \
  --region ap-south-1
# If this returns anything, congratulations, you're on Shodan.

# Ask Access Analyzer to policy-check a policy BEFORE you attach it
aws accessanalyzer validate-policy \
  --policy-document file://my-risky-policy.json \
  --policy-type IDENTITY_POLICY \
  --region ap-south-1
# It will cheerfully tell you that your "Resource": "*" with no condition
# is, in fact, a bad idea. Access Analyzer: the friend who reads the fine print.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Analyzer findings are eventually consistent, not real-time:** A policy change can take up to 30 minutes to reflect in findings — don't build a hard SLA gate on "zero findings" that runs seconds after a Terraform apply.
- **Archived findings can silently reappear:** If the underlying policy changes and then reverts, an archived finding gets re-evaluated and can come back ACTIVE — archiving isn't a permanent "ignore."
- **Organization analyzers require a delegated administrator:** Trying to create one from the management account directly (instead of delegating) is a common first-run mistake that fails with a permissions error that doesn't clearly say why.
- **Pro Tip:** Use `validate-policy` in your CI/CD pipeline (as a pre-commit or pre-apply check for Terraform-generated IAM policies) so bad policies get caught before they're ever attached, not after Access Analyzer finds them in production.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → IAM → Access Analyzer → Analyzers`
2. **Look for:** The "Create analyzer" button, and choose zone of trust: Account or Organization.
3. **Key field:** `Type` — set to `Organization` for delegated-admin, org-wide coverage because per-account analyzers miss cross-account drift between sibling accounts.
4. **Common mistake here:** Creating the analyzer in the management account instead of a delegated administrator (usually the Security/Audit account) — AWS recommends never running workloads or tooling from the management account.
5. **Confirm with CLI:**
   ```bash
   aws accessanalyzer list-analyzers --query 'analyzers[].{Name:name,Type:type,Status:status}'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Security Hub | Aggregates Access Analyzer findings alongside GuardDuty and Config for a single pane of glass |
| AWS Organizations | Provides the "zone of trust" boundary that Access Analyzer reasons against |
| AWS Config | Complements Access Analyzer with continuous configuration compliance beyond just access policies |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
