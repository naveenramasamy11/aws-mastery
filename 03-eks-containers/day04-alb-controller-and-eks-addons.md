# ☁️ AWS Load Balancer Controller & EKS Managed Add-ons — AWS Mastery

> **The add-ons you don't manage are the ones that break your cluster at 2am.**

---

## 📖 Concept

Every EKS cluster ships with a handful of components that aren't optional even though nobody explicitly asked for them: the VPC CNI (pod networking), CoreDNS (cluster DNS), and kube-proxy (service routing). For years these were self-managed — you installed them once and then forgot about them, which is exactly how clusters end up running a VPC CNI version from two years ago with known IP-exhaustion bugs. EKS managed add-ons turn these into first-class, versioned, upgradable EKS resources that you can update through the EKS API (or Terraform) instead of `kubectl apply`-ing YAML you copy-pasted from a GitHub issue three years ago.

The AWS Load Balancer Controller (ALB/NLB Controller) is the piece that turns Kubernetes Ingress and Service objects into real Application Load Balancers and Network Load Balancers. Before this controller existed, exposing a Kubernetes service externally on AWS meant either a classic ELB with poor Kubernetes-native features, or hand-managing target groups outside the cluster. The controller watches Ingress/Service resources and reconciles them into ALB/NLB target groups automatically, including IP-mode target registration (pods register directly, bypassing the extra NodePort hop) which meaningfully cuts latency for high-throughput services.

In migration work, this combination — managed add-ons plus the LB controller — is what separates an EKS cluster that a customer can operate themselves from one that requires a phone call to the platform team every time there's a networking hiccup. Getting IRSA (IAM Roles for Service Accounts) wired correctly for the LB controller's IAM permissions is usually the single trickiest step in a first EKS deployment, and getting it wrong produces a controller pod that starts fine but silently fails to reconcile any Ingress — a classic "working as intended, doing nothing" failure mode.

---

## 🏗️ Architecture Snapshot

```
┌───────────────────────────────────────────────────────────┐
│  EKS Cluster                                                   │
│                                                                  │
│  kube-system namespace                                          │
│  ┌───────────┐ ┌───────────┐ ┌─────────────────────┐  │
│  │ VPC CNI      │ │ CoreDNS      │ │ AWS LB Controller        │  │
│  │ (addon)      │ │ (addon)      │ │ (Helm/addon, uses IRSA)  │  │
│  └───────────┘ └───────────┘ └───────────┬───────────┘  │
│                                                │ watches         │
│                                                ▼                 │
│                                  ┌─────────────────────────┐  │
│                                  │ Ingress / Service (LB type)│  │
│                                  └─────────────┬─────────┘  │
│                                                │ reconciles       │
└───────────────────────────────────────┼─────────────┘
                                                   ▼
                                     ┌───────────────────┐
                                     │  Application Load        │
                                     │  Balancer (target: pods) │
                                     └───────────────────┘
```

---

## 💡 Real-World Use Cases

- **Zero-downtime add-on upgrades:** Use EKS managed add-ons' rolling update capability to patch VPC CNI security issues without a full node replacement or cluster downtime.
- **Cost-efficient ALB consolidation:** Use a single ALB with IngressGroup annotations across multiple Ingress resources to avoid provisioning one ALB per microservice.
- **IP-mode target registration for latency-sensitive APIs:** Switch `alb.ingress.kubernetes.io/target-type` to `ip` for services where the extra NodePort hop measurably affects p99 latency.

---

## 🔧 AWS CLI & Console Examples

### Check current add-on versions and available updates

```bash
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.30 \
  --query 'addons[].addonVersions[:3].addonVersion'
```

### Update the VPC CNI add-on in place

```bash
aws eks update-addon \
  --cluster-name prod-eks \
  --addon-name vpc-cni \
  --addon-version v1.18.3-eksbuild.1 \
  --resolve-conflicts OVERWRITE
```

### Install AWS Load Balancer Controller via Helm (with IRSA)

```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=prod-eks \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Terraform — IRSA role for the LB controller

```hcl
module "lb_controller_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  role_name = "eks-alb-controller"

  attach_load_balancer_controller_policy = true

  oidc_providers = {
    main = {
      provider_arn               = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}
```

---

## 🔐 Security Best Practices

- **Always use IRSA for the LB controller's IAM permissions, never node-instance-role permissions:** Attaching ALB permissions to the node role means every pod on every node inherits the ability to create load balancers — a huge blast radius for zero benefit.
- **Pin add-on versions explicitly in Terraform/IaC:** Letting add-ons silently auto-update on cluster upgrade can introduce breaking API changes without a change window.
- **Restrict Ingress annotations via OPA/Kyverno policy:** Without a guardrail, any team can set `scheme: internet-facing` on an Ingress meant to stay internal.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# Ask the LB controller what it thinks it's watching
kubectl logs -n kube-system deployment/aws-load-balancer-controller | grep -i "reconcile"
# If you see nothing, it's not broken — it's just bored. No Ingress yet.

# Count how many orphaned target groups are sitting around unused
aws elbv2 describe-target-groups --query 'TargetGroups[?length(TargetHealthDescriptions)==`0`].TargetGroupName' 2>/dev/null
```

---

## ⚠️ Gotchas & Tricky Bits

- **IRSA trust policy mismatches fail silently:** A wrong `namespace:serviceaccount` in the OIDC trust condition produces a controller pod that starts, logs no errors, and simply never reconciles anything — always check `kubectl describe sa` against the trust policy condition.
- **VPC CNI IP exhaustion is still the #1 EKS networking issue in 2026:** Prefix delegation (`ENABLE_PREFIX_DELEGATION=true`) massively increases pod density per node, but must be enabled before nodes join, not after.
- **Add-on conflict resolution can silently drop custom config:** `--resolve-conflicts OVERWRITE` will wipe manually-applied CNI config changes — use `PRESERVE` if you've hand-tuned anything.
- **Pro Tip:** Run `eksctl utils describe-addon-versions` alongside a Kubernetes version upgrade plan — add-on compatibility windows are often narrower than the EKS version support window itself.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → EKS → Clusters → [cluster] → Add-ons`
2. **Look for:** The version badge next to each add-on — a yellow triangle means an update is available.
3. **Key field:** `Conflict resolution method` — set to `Preserve` if you have any custom ConfigMap edits you don't want overwritten.
4. **Common mistake here:** Clicking "Update now" on VPC CNI during business hours on a cluster with in-flight connections — schedule this in a maintenance window.
5. **Confirm with CLI:**
   ```bash
   aws eks describe-addon --cluster-name prod-eks --addon-name vpc-cni --query 'addon.addonVersion'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| IAM Roles for Service Accounts (IRSA) | Provides the scoped, pod-level AWS permissions the LB controller needs |
| Elastic Load Balancing | The actual ALB/NLB resources the controller creates and manages |
| Amazon VPC | Prefix delegation and ENI limits directly shape how many pods a node can host |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
