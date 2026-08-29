# ☁️ Service Control Policies in AWS Organizations — AWS Mastery

> **IAM policies say what a principal CAN do. SCPs say what NO principal in that account can EVER do — not even the root user.**

---

## 📖 Concept

Every ProServe engagement that touches multi-account landing zones eventually runs into the same question from a customer's security team: "How do we stop anyone — including account admins — from disabling CloudTrail, leaving approved regions, or spinning up unencrypted resources?" IAM policies can't answer that fully, because IAM policies live *inside* an account and a sufficiently privileged principal (or the root user) can always rewrite them. Service Control Policies (SCPs) live one layer up, at the AWS Organizations level, and set the maximum available permissions for every principal in an account — including its root user. They are a permission *ceiling*, never a grant.

The mental model that finally makes SCPs click: an action is only allowed if it passes through **every** applicable policy layer — SCPs at the Org root, SCPs at each OU level down to the account, the account's own IAM policies, permission boundaries, and resource policies. If any layer denies it (explicitly or by omission for SCPs, since SCPs default to implicit deny once attached), the action fails. SCPs never grant permissions by themselves; attaching an SCP that "allows" S3 does nothing useful unless IAM inside the account also allows S3.

In practice, SCPs are how you encode non-negotiable guardrails at scale: region restrictions for data residency, blocking the disabling of GuardDuty/CloudTrail/Config, preventing leaving the Organization, and preventing IAM users from bypassing MFA. I've used SCPs on EKS migration engagements specifically to lock worker-node IAM roles out of ever calling `iam:CreateUser` or `iam:CreatePolicy` — since a compromised pod shouldn't be able to escalate to create new IAM principals, only ever assume the roles it was explicitly granted.

---

## 🏗️ Architecture Snapshot

```
                     AWS Organizations
                     ┌─────────────────────────────┐
                     │   Root OU                    │
                     │   SCP: DenyLeaveOrganization  │
                     └──────────────┬───────────────┘
                                    │
          ┌─────────────────────────┼─────────────────────────┐
          │                         │                         │
   ┌──────▼──────┐          ┌───────▼───────┐         ┌───────▼───────┐
   │ Security OU │          │ Workloads OU  │         │ Sandbox OU    │
   │ SCP: DenyRegion       │ SCP: DenyRegion │         │ SCP: DenyEC2  │
   │      OutsideApproved  │      OutsideApproved       │  LargeInstances│
   └──────┬──────┘          └───────┬───────┘         └───────┬───────┘
          │                         │                         │
   ┌──────▼──────┐          ┌───────▼───────┐         ┌───────▼───────┐
   │ Log Archive │          │  Prod Account │         │ Dev Account   │
   │ Account     │          │  IAM: eks-node-role  │  │ IAM: dev-role │
   └─────────────┘          └───────────────┘         └───────────────┘

Effective permission = SCP(root) ∩ SCP(OU) ∩ SCP(account) ∩ IAM policy ∩ boundary
```

---

## 💡 Real-World Use Cases

- **Data residency enforcement:** A financial-services customer required every workload to stay in `ap-south-1`. One SCP attached at the Workloads OU denies any API call where the region isn't in the approved list — impossible to bypass even by an account admin.
- **Guardrail against security-tooling tampering:** Deny `cloudtrail:StopLogging`, `guardduty:DeleteDetector`, and `config:DeleteConfigurationRecorder` org-wide so no compromised or careless account can go dark on logging.
- **Blast-radius reduction for EKS worker roles:** Deny IAM principal creation and policy attachment actions in workload accounts so a compromised container can never self-escalate by minting new IAM identities.

---

## 🔧 AWS CLI & Console Examples

### Create and attach an SCP

```bash
# Create the policy document as JSON, then create the SCP
aws organizations create-policy \
  --name "DenyRegionOutsideApproved" \
  --description "Deny all actions outside ap-south-1 and us-east-1 (global services)" \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-region.json \
  --region ap-south-1

# Attach it to an OU
aws organizations attach-policy \
  --policy-id p-examplescp01 \
  --target-id ou-abcd-11111111
```

### deny-region.json — region restriction with global-service exceptions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideApprovedRegions",
      "Effect": "Deny",
      "NotAction": [
        "iam:*", "organizations:*", "route53:*", "sts:*",
        "cloudfront:*", "waf:*", "support:*", "budgets:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": ["ap-south-1", "us-east-1"]
        }
      }
    }
  ]
}
```

### Guardrail: block disabling security services

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenySecurityToolingTamper",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "guardduty:DeleteDetector",
        "guardduty:DisassociateFromMasterAccount",
        "config:DeleteConfigurationRecorder",
        "config:StopConfigurationRecorder"
      ],
      "Resource": "*"
    }
  ]
}
```

### Terraform for repeatable SCP management

```hcl
resource "aws_organizations_policy" "deny_region" {
  name    = "DenyRegionOutsideApproved"
  type    = "SERVICE_CONTROL_POLICY"
  content = file("${path.module}/policies/deny-region.json")
}

resource "aws_organizations_policy_attachment" "workloads_ou" {
  policy_id = aws_organizations_policy.deny_region.id
  target_id = aws_organizations_organizational_unit.workloads.id
}
```

---

## 🔐 Security Best Practices

- **Never attach an SCP with only Allow statements to a new OU without testing** — the moment you attach *any* SCP, it switches that OU/account from "implicit allow everything" to "default deny everything not explicitly allowed" for that policy's scope. A poorly scoped Allow-list SCP has locked out entire migration teams mid-cutover.
- **Always keep a FullAWSAccess policy attached alongside custom deny-list SCPs** — Organizations' default `FullAWSAccess` should remain attached unless you're deliberately running an allow-list model; removing it while attaching an incomplete allow-list SCP is the single most common self-inflicted outage in SCP rollouts.
- **Use `aws:PrincipalArn` conditions to exempt break-glass roles** — every SCP that blocks security-tooling changes should have an explicit exception for a tightly controlled incident-response role, or your own security team will lock themselves out during an actual incident.

---

## 😄 Funny Things to Try

```bash
# See every SCP affecting your current account, layered top to bottom
aws organizations list-policies-for-target \
  --target-id $(aws sts get-caller-identity --query Account --output text) \
  --filter SERVICE_CONTROL_POLICY

# Simulate whether an action would actually be allowed, factoring in SCPs
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/eks-node-role \
  --action-names s3:DeleteBucket \
  --resource-arns "*"
# Note: this simulates IAM only — it does NOT factor in SCPs, which is
# exactly the gotcha that catches people who trust this tool blindly.
```

---

## ⚠️ Gotchas & Tricky Bits

- **`simulate-principal-policy` ignores SCPs entirely** — it only evaluates IAM. Teams routinely "verify" an action is allowed via the simulator, then get denied in production because an SCP at the OU level silently blocked it. Always test with a real API call in a sandbox account under the same OU.
- **SCP propagation is eventually consistent** — after attaching or detaching an SCP, allow a few minutes before assuming the change is live everywhere; I've seen engineers panic-debug a "still blocked" issue that was just propagation lag.
- **The 5KB (attached) / 5,120-character SCP size limit adds up fast** — Deny lists with many actions and long ARNs hit this ceiling quicker than expected; use policy variables and wildcards deliberately, and split logically unrelated guardrails into separate SCPs rather than one mega-policy.
- **Pro Tip:** Roll out new SCPs to a single "canary" OU (a low-stakes sandbox account) for 24–48 hours before organization-wide attachment. It's the cheapest insurance against an org-wide lockout.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → AWS Organizations → Policies → Service control policies`
2. **Look for:** the `FullAWSAccess` default policy — confirm it's still attached to OUs you don't intend to fully lock down.
3. **Key field:** `Targets` tab on any SCP — shows every OU/account it's attached to; always review this before editing an existing SCP's JSON.
4. **Common mistake here:** editing a live SCP directly instead of duplicating it, testing the duplicate on a sandbox OU, then swapping — live edits to attached SCPs affect every account under that OU immediately.
5. **Confirm with CLI:**
   ```bash
   aws organizations list-targets-for-policy --policy-id p-examplescp01
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Config | Detects and reports drift/violations that SCPs prevent proactively |
| IAM Access Analyzer | Surfaces unused permissions that a tighter SCP could safely restrict |
| AWS Control Tower | Ships pre-built "guardrail" SCPs as part of landing zone setup |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*