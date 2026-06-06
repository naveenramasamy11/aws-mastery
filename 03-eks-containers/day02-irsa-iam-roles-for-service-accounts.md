# ☁️ IRSA — IAM Roles for Service Accounts — AWS Mastery

> **The feature that finally ended the era of "just mount AWS credentials as a Kubernetes secret" and gave pods cryptographically verifiable, short-lived, pod-scoped AWS identities.**

---

## 📖 Concept

IRSA (IAM Roles for Service Accounts) is the mechanism that lets EKS pods assume IAM roles without any static credentials. It works via OpenID Connect (OIDC) federation: the EKS cluster has an OIDC provider endpoint, and IAM trusts that endpoint to validate JWT tokens issued by Kubernetes.

When a pod with an annotated Service Account starts, the EKS Pod Identity Webhook intercepts the pod creation and automatically injects two things: a projected Service Account token (a signed JWT with short TTL, typically 24h), and the environment variables `AWS_ROLE_ARN` and `AWS_WEB_IDENTITY_TOKEN_FILE`. The AWS SDK picks these up automatically and calls `sts:AssumeRoleWithWebIdentity`, exchanging the Kubernetes JWT for temporary AWS credentials (15-minute TTL by default, auto-renewed by the SDK).

The trust relationship on the IAM role uses conditions to ensure only the specific service account in the specific namespace in the specific cluster can assume it — not any random pod in the cluster. This gives you **pod-level granularity** for IAM permissions, which is critical for multi-tenant EKS clusters.

**IRSA vs EC2 Instance Profile:** Before IRSA, pods running on EC2 nodes inherited the node's IAM instance profile. This meant every pod on that node had the same permissions as the node itself — a massive over-permission problem. A compromised pod could do anything the node could do. IRSA completely decouples pod identity from node identity.

**EKS Pod Identity (2023):** AWS introduced EKS Pod Identity as a simpler alternative to IRSA that doesn't require OIDC setup per cluster. With Pod Identity, you use the EKS API to directly associate IAM roles to service accounts — no trust policy editing needed. For new clusters, consider Pod Identity; for existing IRSA setups, migration is not urgent.

---

## 🏗️ Architecture Snapshot

```
IRSA Flow — Pod Assuming IAM Role
────────────────────────────────────────────────────────────────────

  EKS Cluster (ap-south-1)
  ┌────────────────────────────────────────────────────────────┐
  │  OIDC Provider: oidc.eks.ap-south-1.amazonaws.com/id/XXX  │
  │                                                            │
  │  Pod (payments-service)                                    │
  │  ┌──────────────────────────────────────────────────────┐  │
  │  │  ServiceAccount: payments-sa                         │  │
  │  │  Annotation: eks.amazonaws.com/role-arn:             │  │
  │  │    arn:aws:iam::123456789012:role/payments-pod-role  │  │
  │  │                                                      │  │
  │  │  Injected by Webhook:                                │  │
  │  │  AWS_ROLE_ARN=arn:aws:iam::...                       │  │
  │  │  AWS_WEB_IDENTITY_TOKEN_FILE=/var/run/secrets/...    │  │
  │  └──────────────────────────────────────────────────────┘  │
  └────────────────────────────────────────────────────────────┘
           │
           │  JWT Token (signed by k8s)
           ▼
  ┌────────────────────────┐
  │  AWS STS               │
  │  AssumeRoleWithWebIdentity
  │  Validates JWT against │
  │  OIDC provider         │
  └──────────┬─────────────┘
             │
             │  Temporary credentials (15 min)
             ▼
  ┌────────────────────────┐
  │  IAM Role              │
  │  payments-pod-role     │
  │  Trust: sub=system:    │
  │  serviceaccount:       │
  │  payments:payments-sa  │
  └──────────┬─────────────┘
             │
             ▼
  ┌────────────────────────────────────────┐
  │  AWS Services (S3, DynamoDB, SQS, etc) │
  │  Scoped to only what payments needs    │
  └────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Microservice per-service permissions:** Each microservice's pods get their own IAM role. The payments service gets access to `payments-*` DynamoDB tables and `payments-receipts` S3 bucket. The catalog service gets a completely different role scoped to catalog resources. Zero blast radius if one pod is compromised.
- **External Secrets Operator:** ESO uses IRSA to call Secrets Manager and SSM Parameter Store and sync secrets into Kubernetes. The ESO service account gets a role with `secretsmanager:GetSecretValue` on specific secrets — no credentials in the cluster at all.
- **Velero backups:** Velero's service account assumes an IRSA role with access to an S3 bucket for cluster backup/restore. Works across clusters and accounts when properly configured.

---

## 🔧 AWS CLI & Console Examples

### Step 1: Ensure OIDC Provider is Associated with Cluster

```bash
# Check if OIDC provider is already set up
aws eks describe-cluster \
  --name my-eks-cluster \
  --query 'cluster.identity.oidc.issuer' \
  --output text \
  --region ap-south-1
# Output: https://oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

# Get the OIDC ID (part after /id/)
OIDC_ID=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --query 'cluster.identity.oidc.issuer' \
  --output text \
  --region ap-south-1 | cut -d '/' -f 5)

echo $OIDC_ID

# Check if the IAM OIDC provider exists
aws iam list-open-id-connect-providers \
  --query "OpenIDConnectProviderList[?contains(Arn, '$OIDC_ID')]"

# Create the OIDC provider if it doesn't exist (eksctl is easiest)
eksctl utils associate-iam-oidc-provider \
  --cluster my-eks-cluster \
  --region ap-south-1 \
  --approve
```

### Step 2: Create the IAM Role with OIDC Trust Policy

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_PROVIDER=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --query 'cluster.identity.oidc.issuer' \
  --output text \
  --region ap-south-1 | sed 's|https://||')

# Create trust policy
cat > trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_PROVIDER}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:payments:payments-sa",
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name payments-pod-role \
  --assume-role-policy-document file://trust-policy.json

# Attach a permissions policy
aws iam attach-role-policy \
  --role-name payments-pod-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBReadOnlyAccess
```

### Step 3: Annotate the Kubernetes Service Account

```bash
# Create the service account with annotation
kubectl create serviceaccount payments-sa -n payments

kubectl annotate serviceaccount payments-sa \
  -n payments \
  eks.amazonaws.com/role-arn=arn:aws:iam::${ACCOUNT_ID}:role/payments-pod-role

# Verify
kubectl describe serviceaccount payments-sa -n payments
```

### Step 4: Use in Pod/Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payments-service
  namespace: payments
spec:
  replicas: 2
  selector:
    matchLabels:
      app: payments
  template:
    metadata:
      labels:
        app: payments
    spec:
      serviceAccountName: payments-sa  # ← this is all you need
      containers:
      - name: payments
        image: payments:v1.2.3
        # AWS SDK auto-detects IRSA env vars — no config needed
        env:
        - name: AWS_REGION
          value: ap-south-1
```

### Verify IRSA is Working from Inside the Pod

```bash
kubectl exec -it deployment/payments-service -n payments -- bash

# Inside the pod:
echo $AWS_ROLE_ARN           # should show the IAM role ARN
echo $AWS_WEB_IDENTITY_TOKEN_FILE  # should show /var/run/secrets/...

# Verify which identity the pod is using
aws sts get-caller-identity
# Output:
# {
#   "UserId": "AROA...:aws-sdk-go-v2-1234",
#   "Account": "123456789012",
#   "Arn": "arn:aws:sts::123456789012:assumed-role/payments-pod-role/aws-sdk-go-v2-..."
# }
```

### Terraform — IRSA Setup End-to-End

```hcl
data "aws_eks_cluster" "main" {
  name = var.cluster_name
}

data "aws_iam_openid_connect_provider" "eks" {
  url = data.aws_eks_cluster.main.identity[0].oidc[0].issuer
}

data "aws_iam_policy_document" "payments_trust" {
  statement {
    effect = "Allow"
    principals {
      type        = "Federated"
      identifiers = [data.aws_iam_openid_connect_provider.eks.arn]
    }
    actions = ["sts:AssumeRoleWithWebIdentity"]
    condition {
      test     = "StringEquals"
      variable = "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub"
      values   = ["system:serviceaccount:payments:payments-sa"]
    }
    condition {
      test     = "StringEquals"
      variable = "${replace(data.aws_iam_openid_connect_provider.eks.url, "https://", "")}:aud"
      values   = ["sts.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "payments_pod" {
  name               = "payments-pod-role"
  assume_role_policy = data.aws_iam_policy_document.payments_trust.json
}
```

---

## 🔐 Security Best Practices

- **Always scope the trust condition to a specific namespace AND service account.** A trust policy that only checks `aud` (audience) but not `sub` (subject) allows any service account in the cluster to assume the role. Always include `StringEquals` on both `:sub` and `:aud`.
- **Never reuse IRSA roles across namespaces or clusters.** One role per service account per cluster. If you share roles, you lose the blast radius isolation that makes IRSA valuable in the first place.
- **Use token expiration.** The projected service account token has a 24-hour TTL by default. You can set it shorter for high-security workloads using `expirationSeconds` in the projected volume spec.
- **Enable EKS audit logging.** All `sts:AssumeRoleWithWebIdentity` calls from IRSA appear in CloudTrail. Set up CloudWatch Insights queries to alert on unexpected role assumptions from unusual pod names or namespaces.
- **Pin the OIDC thumbprint.** When creating the OIDC provider via Terraform, pin the thumbprint to prevent MITM attacks on the OIDC endpoint. The eksctl-created provider does this automatically.

---

## 😄 Funny Things to Try

```bash
# Look at the actual JWT token the pod is using (decode it live)
kubectl exec -it deployment/payments-service -n payments -- \
  cat $AWS_WEB_IDENTITY_TOKEN_FILE | \
  cut -d. -f2 | base64 -d 2>/dev/null | python3 -m json.tool

# The token contains:
# "sub": "system:serviceaccount:payments:payments-sa"
# "iss": "https://oidc.eks.ap-south-1.amazonaws.com/id/..."
# "exp": 1234567890  ← when it expires (usually +24h)
# It's just a JWT. You can decode it anywhere. But only AWS can VALIDATE it. 🔐

# Find all service accounts with IRSA annotations (org-wide audit)
kubectl get serviceaccounts \
  --all-namespaces \
  -o jsonpath='{range .items[?(@.metadata.annotations.eks\.amazonaws\.com/role-arn)]}{.metadata.namespace}/{.metadata.name}: {.metadata.annotations.eks\.amazonaws\.com/role-arn}{"\n"}{end}'
# Maps every namespace/SA to its IAM role.
# Great for "wait, does anything ELSE have access to this role?"
```

---

## ⚠️ Gotchas & Tricky Bits

- **The OIDC provider must exist before the IAM role trust policy is evaluated.** If you delete and recreate a cluster, the OIDC provider URL changes (new ID suffix). Any existing IRSA trust policies referencing the old URL stop working — you must update them.
- **SDK version matters.** IRSA automatic credential discovery requires AWS SDK v2 (or v1 ≥ 1.25 for Go, ≥ 1.9 for Python boto3). Older SDKs won't pick up `AWS_WEB_IDENTITY_TOKEN_FILE`. Check SDK versions in your container images if IRSA isn't working.
- **`kubectl exec` doesn't inherit IRSA automatically.** When you exec into a pod and run AWS CLI, it uses the pod's injected env vars and service account token — but only if the AWS CLI version is recent enough to handle Web Identity. Older awscli v1 may not. Use `aws sts get-caller-identity` as the first debug step.
- **Fargate profiles need OIDC too.** IRSA works the same way on Fargate pods. Ensure the OIDC provider is set up — Fargate doesn't have a node instance profile to fall back on, so missing IRSA = complete auth failure.
- **Pro Tip:** Use `eksctl create iamserviceaccount` for the most reliable IRSA setup — it creates the OIDC provider, IAM role with correct trust policy, and annotates the K8s service account in one command. Terraform is more flexible but error-prone on the trust condition formatting.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → IAM → Roles → [your IRSA role]`
2. **Look for:** "Trust relationships" tab — you should see `sts:AssumeRoleWithWebIdentity` with the OIDC provider as Principal
3. **Key field:** The `Condition` block — verify it has the exact `sub` matching `system:serviceaccount:<namespace>:<sa-name>`
4. **Common mistake here:** Using `StringLike` instead of `StringEquals` in the sub condition — wildcards here defeat the purpose
5. **Verify in EKS:** `AWS Console → EKS → Clusters → [cluster] → Configuration → Authentication` — OIDC provider URL should be listed
6. **CloudTrail check:**
   ```bash
   aws cloudtrail lookup-events \
     --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRoleWithWebIdentity \
     --region ap-south-1 \
     --query 'Events[*].[EventTime,Username,CloudTrailEvent]' \
     --output table
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS STS | All IRSA auth flows through `sts:AssumeRoleWithWebIdentity` — understanding STS token lifecycle explains credential expiry |
| External Secrets Operator | ESO's most common deployment uses IRSA to pull from Secrets Manager and sync to K8s secrets |
| Karpenter | Karpenter's controller uses IRSA to call EC2/SQS APIs for node provisioning — one of the first IRSA roles you set up |
| AWS Load Balancer Controller | Requires an IRSA role to manage ALB/NLB resources on behalf of Ingress objects |
| Velero | Backup/restore tool that uses IRSA to access S3 for cluster state snapshots |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
