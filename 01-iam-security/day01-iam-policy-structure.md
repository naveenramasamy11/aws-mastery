# ☁️ IAM Policy Structure & Evaluation Logic — AWS Mastery

> **The policy that denies you is always somewhere you didn't look — until now.**

---

## 📖 Concept

IAM is the nervous system of AWS security. Every API call, every SDK request, every `aws s3 cp` you run travels through the IAM policy evaluation engine before it reaches the actual service. Understanding this engine isn't optional for anyone operating AWS at scale — it's the difference between a 2am incident caused by an overly permissive role and a clean, auditable access model.

An IAM policy is JSON with five key components: `Version`, `Statement`, `Effect` (Allow/Deny), `Action`, and `Resource`. Optionally a `Condition` block adds attribute-based controls. The `Version` should always be `"2012-10-17"` — using the older `"2008-10-17"` silently disables policy variables like `${aws:username}`, which has caused many a confused support ticket.

The evaluation order is: **Explicit Deny wins always → Service Control Policies (SCPs) → Permission Boundaries → Resource-based policies → Identity-based policies → Session policies**. If no explicit Allow is found at the end of this chain, the result is an implicit Deny. This is why "my role has AdministratorAccess but it still can't do X" is almost always an SCP or Permission Boundary problem — not an IAM bug.

In ProServe migrations, IAM is often the last thing teams think about and the first thing that breaks. When migrating 150+ microservices into EKS, every pod's service account needs the right IRSA role, and getting that wrong means silent failures at runtime — not helpful build errors at deploy time.

---

## 🏗️ Architecture Snapshot

```
  IAM Policy Evaluation — Order of Operations
  ─────────────────────────────────────────────

  API Request arrives
        │
        ▼
  ┌─────────────────────┐
  │  Explicit DENY?     │──YES──▶ DENY (game over)
  └──────────┬──────────┘
             │ NO
             ▼
  ┌─────────────────────┐
  │  SCP allows it?     │──NO───▶ DENY
  └──────────┬──────────┘
             │ YES
             ▼
  ┌─────────────────────┐
  │ Permission Boundary │──NO───▶ DENY
  │    allows it?       │
  └──────────┬──────────┘
             │ YES
             ▼
  ┌─────────────────────┐
  │  Resource policy    │
  │  + Identity policy  │──NO───▶ DENY (implicit)
  │  has Allow?         │
  └──────────┬──────────┘
             │ YES
             ▼
           ALLOW ✅
```

---

## 💡 Real-World Use Cases

- **EKS IRSA:** A pod needs read access to a specific S3 bucket. Rather than giving the EC2 node's instance role access (which all pods share), IRSA lets you attach a scoped IAM role to just that pod's service account via OIDC federation.
- **Multi-account SCP guardrails:** In AWS Organizations, SCPs prevent even root-level account admins from disabling CloudTrail or leaving the organization — critical for compliance in large enterprises with 50+ accounts.
- **Permission Boundaries for developer self-service:** Allow developers to create their own IAM roles for Lambda functions, but enforce that those roles can never exceed a pre-approved set of permissions — no accidental AdministratorAccess Lambda roles.

---

## 🔧 AWS CLI & Console Examples

### Viewing and Testing Policies

```bash
# See all policies attached to a role
aws iam list-attached-role-policies \
  --role-name my-eks-node-role \
  --region ap-south-1

# Simulate whether an action is allowed (IAM Policy Simulator via CLI)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/my-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/my-key \
  --region ap-south-1

# Get the full policy document of an inline policy
aws iam get-role-policy \
  --role-name my-eks-node-role \
  --policy-name my-inline-policy

# List all roles that have a specific managed policy attached
aws iam list-entities-for-policy \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --entity-filter Role
```

### Writing a Scoped Least-Privilege Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificS3BucketRead",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-app-bucket",
        "arn:aws:s3:::my-app-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-south-1"
        }
      }
    }
  ]
}
```

### Condition Keys — The Underused Power Move

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```
> This denies all actions if MFA is not present. Attach to an IAM group. Your security team will love you.

### IRSA Trust Policy for EKS Pod

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:default:my-service-account",
          "oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
```

### Permission Boundary — Cap What a Developer Can Create

```bash
# Create a permission boundary policy
aws iam create-policy \
  --policy-name DeveloperBoundary \
  --policy-document file://dev-boundary.json

# Attach it when creating a new role
aws iam create-role \
  --role-name dev-lambda-role \
  --assume-role-policy-document file://trust.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/DeveloperBoundary
```

---

## 🔐 Security Best Practices

- **Never use wildcards in Resource for sensitive actions:** `"Resource": "*"` on `iam:PassRole` or `sts:AssumeRole` is a privilege escalation vector. Always scope to specific ARNs.
- **Enforce IMDSv2 everywhere:** Set `HttpTokens: required` in all Launch Templates. IMDSv1 is a SSRF attack's best friend — it was behind the Capital One breach.
- **Use `aws:PrincipalOrgID` condition:** Prevent cross-account access from outside your AWS Organization. Add `"aws:PrincipalOrgID": "o-xxxxxxxxxx"` to S3 bucket policies and KMS key policies.
- **Rotate access keys — or better, eliminate them:** Use IAM roles for EC2, EKS IRSA for pods, and IAM Identity Center for humans. If you still have long-lived access keys, `aws iam list-access-keys` and start the rotation conversation.
- **Enable AWS IAM Access Analyzer:** It uses formal reasoning to flag any policy that grants access to principals outside your account or organization. Free, instant value.

---

## 😄 Funny Things to Try

```bash
# "Who am I?" — the most useful AWS command you'll run daily
aws sts get-caller-identity
# Returns your Account ID, User ID, and ARN.
# Run this before anything else when something mysteriously fails.
# AWS's version of "have you tried turning it off and on again?"

# List every IAM user who has NEVER used their password
aws iam generate-credential-report && sleep 5
aws iam get-credential-report \
  --query 'Content' --output text | base64 -d | \
  awk -F',' 'NR>1 && $5=="false" {print $1, "has never logged in 👻"}'

# Find all roles that trust EVERYONE (misconfigured trust policies)
aws iam list-roles \
  --query 'Roles[?contains(AssumeRolePolicyDocument, `"AWS":"*"`)].RoleName' \
  --output table
# If this returns anything... you have a bad day incoming.

# The "oops" detector — find your most permissive policies
aws iam list-policies --scope Local \
  --query 'Policies[].PolicyName' --output text | \
  tr '\t' '\n' | head -20
# Then check each one. Somewhere in there is an "AdministratorAccess-copy" someone made.
```

---

## ⚠️ Gotchas & Tricky Bits

- **`NotAction` vs `Action` with Deny:** Using `"Effect": "Deny", "NotAction": "s3:*"` does NOT mean "deny everything except S3." It means "deny everything that is NOT an S3 action" — which is the opposite of what most people intend. Use with extreme caution.
- **IAM propagation delay:** IAM changes are eventually consistent. After attaching a policy or updating a role, wait 5–10 seconds before testing — "it doesn't work" often means "it hasn't propagated yet." In automated pipelines, add a short sleep or retry loop.
- **`iam:PassRole` is the sneaky one:** Allowing a user to create Lambda functions without restricting `iam:PassRole` means they can pass any role to that Lambda — including roles more powerful than their own. Always scope `iam:PassRole` with a condition on the role ARN pattern.
- **SCP `Deny` beats account root:** SCPs can deny actions even to the root user of a member account. Many customers are surprised to discover they can't disable CloudTrail in a child account because of an SCP applied at the OU level.
- **Pro Tip:** Use the IAM Policy Simulator in the AWS Console (`IAM → Policy Simulator`) before attaching complex policies. It runs the full evaluation engine and tells you exactly which statement caused an Allow or Deny — saves hours of debugging.

---

## 📸 Console Walkthrough

> *Creating a scoped IAM role for an EKS service account (IRSA)*

1. **Navigate to:** `AWS Console → IAM → Roles → Create Role`
2. **Select trusted entity:** Choose `Web identity`
3. **Identity provider:** Select your EKS cluster's OIDC provider (format: `oidc.eks.<region>.amazonaws.com/id/<hash>`)
4. **Audience:** Set to `sts.amazonaws.com`
5. **Key field:** `Condition → StringEquals → :sub` — set this to `system:serviceaccount:<namespace>:<serviceaccount-name>` — this is what binds the role to exactly one Kubernetes service account
6. **Common mistake here:** Leaving the `:sub` condition blank makes the role assumable by ANY service account in the cluster — not just yours
7. **Attach permissions policy:** Scope it tightly — only the S3/DynamoDB/SSM actions that pod actually needs
8. **Confirm with CLI:**
   ```bash
   # Verify the OIDC trust is wired correctly
   aws iam get-role \
     --role-name my-irsa-role \
     --query 'Role.AssumeRolePolicyDocument' \
     --output json
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Organizations / SCPs | SCPs act as the outer boundary before IAM policies are evaluated |
| AWS Secrets Manager | Rotate credentials automatically; IAM controls who can access secrets |
| EKS / IRSA | Pod-level IAM via OIDC — eliminates node-level overpermissioning |
| AWS Config | Detects IAM drift — e.g., MFA disabled, root access keys present |
| GuardDuty | Uses CloudTrail to detect anomalous API calls by IAM principals |
| IAM Access Analyzer | Flags policies granting external access using formal verification |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
