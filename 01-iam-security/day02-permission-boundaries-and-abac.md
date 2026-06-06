# ☁️ Permission Boundaries & ABAC — AWS Mastery

> **The two IAM features that separate "cloud admin who googles everything" from "cloud architect who designs delegation safely."**

---

## 📖 Concept

### Permission Boundaries

A Permission Boundary is a managed IAM policy attached to a role or user that sets the *maximum* permissions that entity can ever have — regardless of what identity-based policies are attached. Think of it as a ceiling. If the boundary doesn't allow `s3:DeleteBucket`, the identity can't delete buckets even if every inline and managed policy in the account says it can.

This is the single most powerful feature for **safe delegation**. In a ProServe or platform-engineering context, you want to let dev teams create their own IAM roles without being able to escalate to full admin. Without permission boundaries, a developer who can `iam:CreateRole` + `iam:AttachRolePolicy` can trivially create an `AdministratorAccess` role and assume it. With a boundary that caps at, say, `s3:*` and `ec2:*`, that escalation path is gone.

The effective permissions of a principal are always the **intersection** of:
1. Identity-based policies
2. Permission boundary (if any)
3. SCPs (if the account is in an AWS Organization)
4. Resource-based policies (evaluated separately, can grant access even when identity policy doesn't)

### ABAC — Attribute-Based Access Control

ABAC lets you write IAM policies that make access decisions based on **tags** — on the principal (user/role) AND on the resource being accessed. Instead of maintaining a separate policy per team/environment, you write one policy that says: "Allow access to any resource where the resource tag `Environment` equals the caller's tag `Environment`."

This scales beautifully in large multi-team accounts. In a migration wave context where you have 50 application teams each deploying to their own tagged environments, ABAC means you write ONE policy instead of 50. New teams get access automatically when you tag them correctly.

The IAM condition key `aws:PrincipalTag/<key>` gives you the tag on the caller. `aws:ResourceTag/<key>` gives you the tag on the resource. The magic happens when you combine them in a `StringEquals` condition.

---

## 🏗️ Architecture Snapshot

```
IAM Effective Permission Evaluation
────────────────────────────────────────────────────────────────

  Developer Role
  ┌─────────────────────────────────────────────────────────┐
  │  Identity Policy: Allow s3:*, ec2:*, iam:CreateRole    │
  │  Permission Boundary: Allow s3:*, ec2:* only           │
  │                                                         │
  │  Effective = INTERSECTION                               │
  │  → s3:* ✅  ec2:* ✅  iam:CreateRole ❌               │
  └─────────────────────────────────────────────────────────┘

  ABAC Tag-Based Access
  ┌──────────────────────┐      ┌─────────────────────────┐
  │  Principal (Role)    │      │  Resource (EC2/S3/RDS)  │
  │  Tag: Team=payments  │─────▶│  Tag: Team=payments     │
  │  Tag: Env=prod       │      │  Tag: Env=prod          │
  └──────────────────────┘      └─────────────────────────┘
           │
           ▼
  Policy Condition:
  aws:PrincipalTag/Team == aws:ResourceTag/Team
  aws:PrincipalTag/Env  == aws:ResourceTag/Env
           │
           ▼
        ✅ ALLOW — same team, same env

  ┌──────────────────────┐      ┌─────────────────────────┐
  │  Principal           │      │  Resource               │
  │  Tag: Team=payments  │─────▶│  Tag: Team=platform     │
  └──────────────────────┘      └─────────────────────────┘
           │
           ▼
        ❌ DENY — different team tag
```

---

## 💡 Real-World Use Cases

- **Platform-team delegation:** Give dev teams an IAM role with `iam:CreateRole` but attach a permission boundary that prevents them from granting themselves or others more than dev-tier access — no more "oops I gave myself AdministratorAccess."
- **Multi-tenant EKS workloads:** Tag each EKS namespace's IRSA role with `Team=<team>` and `Env=<env>`. One ABAC policy governs which S3 buckets, Parameter Store paths, and DynamoDB tables each team's pods can access — no per-team policies needed.
- **Migration wave governance:** During an AWS migration wave, each application has an `AppID` tag. ABAC ensures the migration service role for App-A can only touch App-A's resources, even within the same account.

---

## 🔧 AWS CLI & Console Examples

### Attach a Permission Boundary to a Role

```bash
# Create the boundary policy first
aws iam create-policy \
  --policy-name DevRoleBoundary \
  --policy-document file://boundary.json \
  --region ap-south-1

# boundary.json
# {
#   "Version": "2012-10-17",
#   "Statement": [{
#     "Effect": "Allow",
#     "Action": ["s3:*", "ec2:*", "cloudwatch:*", "logs:*"],
#     "Resource": "*"
#   }]
# }

# Attach to new role at creation time
aws iam create-role \
  --role-name dev-app-role \
  --assume-role-policy-document file://trust.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/DevRoleBoundary \
  --region ap-south-1
```

### Update Boundary on Existing Role

```bash
aws iam put-role-permissions-boundary \
  --role-name dev-app-role \
  --permissions-boundary arn:aws:iam::123456789012:policy/DevRoleBoundary

# Remove boundary (careful — this removes the cap!)
aws iam delete-role-permissions-boundary \
  --role-name dev-app-role
```

### ABAC Policy — Tag-Based Access to EC2

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2SameTeamSameEnv",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Team": "${aws:PrincipalTag/Team}",
          "aws:ResourceTag/Environment": "${aws:PrincipalTag/Environment}"
        }
      }
    },
    {
      "Sid": "DenyUntaggedEC2Actions",
      "Effect": "Deny",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "*",
      "Condition": {
        "Null": {
          "aws:ResourceTag/Team": "true"
        }
      }
    }
  ]
}
```

```bash
# Tag a role for ABAC
aws iam tag-role \
  --role-name payments-service-role \
  --tags Key=Team,Value=payments Key=Environment,Value=prod

# Verify tags
aws iam list-role-tags \
  --role-name payments-service-role \
  --query 'Tags[*].[Key,Value]' \
  --output table
```

### Terraform — Permission Boundary + ABAC Role

```hcl
resource "aws_iam_policy" "dev_boundary" {
  name = "DevRoleBoundary"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:*", "ec2:*", "cloudwatch:*"]
      Resource = "*"
    }]
  })
}

resource "aws_iam_role" "dev_role" {
  name                 = "dev-${var.team}-role"
  permissions_boundary = aws_iam_policy.dev_boundary.arn

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { AWS = "arn:aws:iam::${var.account_id}:root" }
      Action    = "sts:AssumeRole"
    }]
  })

  tags = {
    Team        = var.team
    Environment = var.environment
  }
}
```

---

## 🔐 Security Best Practices

- **Always apply permission boundaries when delegating IAM role creation.** Any IAM principal that can create roles or attach policies is a privilege-escalation risk without a boundary capping what it can grant.
- **Include `iam:PassRole` in your boundary policy carefully.** If an entity needs to pass roles to services (EC2, Lambda), make sure the boundary allows it but scope it: `"Resource": "arn:aws:iam::*:role/dev-*"` not `*`.
- **Use `aws:RequestTag` in ABAC create policies.** Force callers to include required tags when creating resources: add a `Condition` with `Null: {"aws:RequestTag/Team": "true"}` and `Effect: Deny` to block untagged resource creation.
- **Tag governance is critical for ABAC.** ABAC is only as secure as your tagging discipline. Enforce required tags with AWS Config rule `required-tags` and deny creation of resources without mandatory tags via SCP.
- **Audit boundaries regularly with IAM Access Analyzer.** Use Access Analyzer to confirm boundaries are having the intended effect and detect any unintended access paths.

---

## 😄 Funny Things to Try

```bash
# The "ceiling check" — simulate what a bounded role can actually do
# Uses IAM policy simulation to test boundary effect
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/dev-app-role \
  --action-names iam:CreateRole s3:PutObject ec2:DescribeInstances \
  --region ap-south-1 \
  --query 'EvaluationResults[*].[EvalActionName,EvalDecision]' \
  --output table

# Expected output:
# --------------------------------
# | EvalActionName  | EvalDecision |
# |------------------------------ |
# | iam:CreateRole  | implicitDeny | ← boundary blocked it
# | s3:PutObject    | allowed      |
# | ec2:Describe... | allowed      |
# --------------------------------
# This is the fastest way to prove to an auditor that delegation is safe.

# ABAC debugging — who am I and what tags do I have?
aws iam get-role \
  --role-name payments-service-role \
  --query 'Role.Tags' \
  --output table
# Shows the tags that will be evaluated in ABAC conditions.
# "But why can't my Lambda access that S3 bucket?" → check these tags first.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Boundaries don't apply to resource-based policies.** If an S3 bucket policy grants access to a principal, the boundary on that principal does NOT block it. Resource-based policies are evaluated independently. This bites people who think "I put a boundary on that role, it can't access my bucket" — wrong, if the bucket policy allows it.
- **Boundary must explicitly allow actions for them to work.** A boundary is not a "deny list" — it's an "allow ceiling." If you forget to include `sts:AssumeRole` in the boundary and the role needs to assume other roles, it silently fails. Check boundaries when roles can't assume downstream roles.
- **ABAC conditions on `iam:PassRole` need `iam:PassedToService`.** If you use ABAC to control which roles can be passed to services, add the condition `iam:PassedToService` to scope to specific AWS services (e.g., `lambda.amazonaws.com`).
- **Tag propagation latency.** After tagging a role with `aws iam tag-role`, there's a brief propagation delay (usually < 1 minute) before the tags appear in access evaluations. Don't test ABAC policies milliseconds after tagging.
- **Pro Tip:** The most elegant permission boundary pattern is a single `BoundaryPolicy` per account, defined in CloudFormation StackSets, that limits all dev roles to a specific set of services + a specific resource name prefix (e.g., `dev-*`). Enforce this with an SCP that requires any `iam:CreateRole` or `iam:PutRolePermissionsBoundary` action to include your boundary ARN.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → IAM → Roles → [Select a role]`
2. **Look for:** The "Permissions boundary" tab under the role summary
3. **Key field:** `Permissions boundary` — if empty, no ceiling is applied. Click "Set boundary" to attach one.
4. **Common mistake here:** People set a boundary and wonder why allowed actions still fail — they forgot that the identity policy ALSO needs to allow it (it's an intersection, not a union)
5. **For ABAC:** Go to `IAM → Policies → Create Policy` → use the JSON editor to add `Condition` blocks with `aws:PrincipalTag` and `aws:ResourceTag`
6. **Confirm with CLI:**
   ```bash
   aws iam get-role \
     --role-name dev-app-role \
     --query 'Role.PermissionsBoundary'
   # Returns: {"PermissionsBoundaryType": "Policy", "PermissionsBoundaryArn": "..."}
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Organizations / SCPs | SCPs are the org-level ceiling above permission boundaries — both restrict max permissions, evaluated in sequence |
| IAM Access Analyzer | Validates that boundaries achieve intended restriction; generates access preview before you apply |
| AWS Config | `required-tags` rule enforces tag hygiene needed for ABAC to work correctly |
| AWS CloudTrail | Every `iam:PutRolePermissionsBoundary` action is logged — audit boundary changes here |
| AWS Control Tower | Enforces guardrails (SCPs) across landing zone; pairs with permission boundaries for defense in depth |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
