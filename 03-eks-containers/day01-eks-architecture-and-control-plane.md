# ☁️ EKS Architecture & Control Plane Deep Dive — AWS Mastery

> **EKS is not just "Kubernetes on AWS" — it's Kubernetes where AWS manages the part that wakes you up at 3am.**

---

## 📖 Concept

Amazon EKS separates Kubernetes into two distinct planes: the **control plane** (managed by AWS) and the **data plane** (your EC2 nodes or Fargate). The control plane — API server, etcd, controller manager, scheduler — runs in AWS-managed VPCs across multiple AZs. You never SSH into it, you never patch it, and if it fails, AWS fixes it under the EKS SLA. This is the core value proposition: you get Kubernetes without the operational burden of running Kubernetes masters.

The data plane is where your workloads actually run. You choose: Managed Node Groups (AWS-controlled ASG with automatic AMI updates), Self-Managed Node Groups (you control everything, including node draining), or Fargate (serverless pods — no nodes at all). For most production workloads, Managed Node Groups with Karpenter for autoscaling is the modern standard.

EKS networking uses the VPC CNI plugin (`aws-node` DaemonSet), which assigns actual VPC IP addresses to pods. Each pod gets a real ENI secondary IP — not an overlay network. This has a critical implication: **you can run out of VPC IPs if you don't plan your CIDR blocks properly**. A `/24` subnet (256 IPs) can't support 200 pods plus nodes. Plan for `/19` or larger per AZ for production clusters.

Add-ons are the managed components AWS can version and update for you: VPC CNI, CoreDNS, kube-proxy, EBS CSI Driver, EFS CSI Driver. Always use managed add-ons in production — they integrate with EKS upgrade windows and eliminate the "did we update kube-proxy after the cluster upgrade?" question.

---

## 🏗️ Architecture Snapshot

```
  Amazon EKS — Full Architecture
  ────────────────────────────────────────────────────────

  ┌──────────────────────────────────────────────────────┐
  │                  AWS Managed VPC                      │
  │  ┌────────────────────────────────────────────────┐  │
  │  │             EKS Control Plane                   │  │
  │  │  ┌──────────┐ ┌──────────┐ ┌────────────────┐  │  │
  │  │  │ API      │ │  etcd    │ │  Controller    │  │  │
  │  │  │ Server   │ │ (HA)     │ │  Manager /     │  │  │
  │  │  └──────────┘ └──────────┘ │  Scheduler     │  │  │
  │  │                            └────────────────┘  │  │
  │  └──────────────────┬─────────────────────────────┘  │
  └─────────────────────│────────────────────────────────┘
                        │ kubectl / API calls
  ┌─────────────────────▼────────────────────────────────┐
  │                  Customer VPC                         │
  │  ┌────────────────────┐  ┌────────────────────────┐  │
  │  │  Private Subnet    │  │  Private Subnet         │  │
  │  │  AZ-A              │  │  AZ-B                   │  │
  │  │  ┌──────────────┐  │  │  ┌──────────────────┐  │  │
  │  │  │ Node Group   │  │  │  │  Node Group      │  │  │
  │  │  │ m7g.2xlarge  │  │  │  │  m7g.2xlarge     │  │  │
  │  │  │ ┌──┐ ┌──┐   │  │  │  │  ┌──┐ ┌──┐      │  │  │
  │  │  │ │P1│ │P2│   │  │  │  │  │P3│ │P4│      │  │  │
  │  │  │ └──┘ └──┘   │  │  │  │  └──┘ └──┘      │  │  │
  │  │  └──────────────┘  │  │  └──────────────────┘  │  │
  │  └────────────────────┘  └────────────────────────┘  │
  │                                                       │
  │  Add-ons: VPC CNI │ CoreDNS │ kube-proxy │ EBS CSI   │
  └───────────────────────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Payment gateway microservices migration:** Migrating 150+ microservices to EKS using Managed Node Groups with `m7g.2xlarge` — IRSA scopes each service's pod to exactly the S3/DynamoDB/SQS permissions it needs, replacing blanket instance role permissions.
- **Multi-tenant cluster with namespace isolation:** Using Kubernetes Network Policies + Security Groups for Pods to isolate tenant workloads at the pod level, with separate IAM roles per namespace via IRSA.
- **ROSA to EKS migration:** Moving OpenShift workloads to native EKS while preserving RBAC mappings using `aws-auth` ConfigMap migration to EKS Access Entries (the modern approach).

---

## 🔧 AWS CLI & Console Examples

### Creating an EKS Cluster

```bash
# Create cluster with eksctl (recommended for initial setup)
eksctl create cluster \
  --name prod-cluster \
  --region ap-south-1 \
  --version 1.30 \
  --nodegroup-name standard-workers \
  --node-type m7g.2xlarge \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 10 \
  --managed \
  --asg-access \
  --with-oidc \
  --ssh-access=false  # No SSH — use SSM
```

### Checking Cluster and Add-on Status

```bash
# List all clusters
aws eks list-clusters --region ap-south-1

# Describe cluster and get OIDC issuer URL (needed for IRSA)
aws eks describe-cluster \
  --name prod-cluster \
  --region ap-south-1 \
  --query 'cluster.{Status:status,Version:version,OIDCIssuer:identity.oidc.issuer,Endpoint:endpoint}'

# List installed add-ons and their versions
aws eks list-addons \
  --cluster-name prod-cluster \
  --region ap-south-1

# Check if a newer add-on version is available
aws eks describe-addon-versions \
  --addon-name vpc-cni \
  --kubernetes-version 1.30 \
  --query 'addons[].addonVersions[0].addonVersion'
```

### Updating Add-ons (never skip this after cluster upgrades)

```bash
# Update VPC CNI to latest compatible version
aws eks update-addon \
  --cluster-name prod-cluster \
  --addon-name vpc-cni \
  --addon-version v1.18.3-eksbuild.1 \
  --resolve-conflicts OVERWRITE \
  --region ap-south-1

# Check update status
aws eks describe-addon \
  --cluster-name prod-cluster \
  --addon-name vpc-cni \
  --region ap-south-1 \
  --query 'addon.{Status:status,Version:addonVersion}'
```

### Node Group Operations

```bash
# List node groups
aws eks list-nodegroups \
  --cluster-name prod-cluster \
  --region ap-south-1

# Update node group AMI (triggers rolling update)
aws eks update-nodegroup-version \
  --cluster-name prod-cluster \
  --nodegroup-name standard-workers \
  --region ap-south-1

# Scale node group manually
aws eks update-nodegroup-config \
  --cluster-name prod-cluster \
  --nodegroup-name standard-workers \
  --scaling-config minSize=3,maxSize=15,desiredSize=5 \
  --region ap-south-1
```

### OIDC Provider Setup for IRSA

```bash
# Create OIDC provider for the cluster (one-time setup)
eksctl utils associate-iam-oidc-provider \
  --cluster prod-cluster \
  --region ap-south-1 \
  --approve

# Verify OIDC provider exists
aws iam list-open-id-connect-providers \
  --query 'OpenIDConnectProviderList[].Arn'
```

---

## 🔐 Security Best Practices

- **Use EKS Access Entries instead of aws-auth ConfigMap:** The ConfigMap approach is error-prone (a YAML typo can lock you out of your cluster). EKS Access Entries (GA in 2024) manage cluster access via IAM — use `aws eks create-access-entry` for all new clusters.
- **Enable envelope encryption for EKS secrets:** By default, Kubernetes Secrets are base64 in etcd — not encrypted. Enable KMS envelope encryption at cluster creation: `--secrets-encryption-key-arn arn:aws:kms:...`
- **Network Policies are not enabled by default:** The VPC CNI supports Network Policies but they must be enabled explicitly. Without them, all pods in the cluster can talk to each other. Enable: `aws eks update-addon --addon-name vpc-cni --configuration-values '{"enableNetworkPolicy":"true"}'`
- **Restrict public API endpoint access:** Set `publicAccessCidrs` to your corporate NAT gateway IPs only, or disable the public endpoint entirely and use private endpoint via VPN/Direct Connect.

---

## 😄 Funny Things to Try

```bash
# The "how many pods am I running?" existential question
kubectl get pods --all-namespaces --no-headers | wc -l
# Then: kubectl get pods --all-namespaces --field-selector=status.phase=Pending --no-headers
# If the second number > 0, you have a problem. Or you're Karpenter and it's already fixing it.

# Find all pods NOT on the nodes you expected
kubectl get pods --all-namespaces -o wide | grep -v "Running" | grep -v "Completed"
# The "hall of shame" — everything that's not where it should be.

# Check what's eating your node's resources
kubectl describe node <node-name> | grep -A 20 "Allocated resources"
# Surprise: it's always that one JVM app someone said "only needs 512Mi"

# The "oops I forgot resource limits" finder
kubectl get pods --all-namespaces -o json | \
  jq '.items[] | select(.spec.containers[].resources.limits == null) | .metadata.name'
# Pods with no limits can eat the entire node. Find them before they find you.
```

---

## ⚠️ Gotchas & Tricky Bits

- **VPC CNI IP exhaustion:** Each node pre-warms a pool of IPs (controlled by `WARM_IP_TARGET` and `MINIMUM_IP_TARGET` env vars on `aws-node`). On large instances, this can consume 15–20 IPs per node even with few pods. Plan your subnets with `/19` or larger per AZ.
- **CoreDNS is a bottleneck at scale:** Default CoreDNS has 2 replicas. At 500+ pods with heavy DNS lookups, you'll see intermittent timeouts. Scale CoreDNS replicas, enable NodeLocal DNSCache, and set `ndots:2` in pod DNS config to reduce unnecessary upstream queries.
- **EKS upgrades are in-place but sequential:** You must upgrade one minor version at a time (1.27 → 1.28 → 1.29). You cannot skip versions. Always upgrade add-ons AFTER the control plane upgrade, and node groups AFTER add-ons.
- **Fargate pods have no DaemonSets:** If your observability agent, security scanner, or log forwarder runs as a DaemonSet, it won't run on Fargate pods. Use sidecar containers or Fargate-compatible alternatives (like AWS Distro for OpenTelemetry as a sidecar).
- **Pro Tip:** Use `kubectl-node-shell` (open source) to get a shell on a node without SSH — it spawns a privileged pod that mounts the host filesystem. Useful for debugging node-level issues without opening any ports.

---

## 📸 Console Walkthrough

> *Checking cluster health and upgrading an add-on after a cluster version upgrade*

1. **Navigate to:** `AWS Console → EKS → Clusters → [your-cluster]`
2. **Overview tab:** Check `Cluster status` — should be `Active`
3. **Add-ons tab:** Look for add-ons showing `Update available` badge in orange
4. **Click the add-on:** e.g., `vpc-cni` → `Edit`
5. **Key field:** `Version` dropdown — select the latest version compatible with your cluster version
6. **Conflict resolution:** Set to `Overwrite` for managed add-ons (preserves your custom config where possible)
7. **Common mistake here:** Updating add-ons before upgrading the control plane — always control plane first, then add-ons, then node groups
8. **Confirm with CLI:**
   ```bash
   aws eks describe-addon \
     --cluster-name prod-cluster \
     --addon-name vpc-cni \
     --query 'addon.status'
   # Should return "ACTIVE" when done
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| IAM / IRSA | Pod-level AWS permissions via OIDC federation — the EKS security cornerstone |
| VPC / Subnets | VPC CNI assigns pod IPs from your subnets — CIDR planning is critical |
| ALB / AWS Load Balancer Controller | Ingress resources create ALBs automatically via Kubernetes annotations |
| Karpenter | Node autoscaling — provisions right-sized nodes in seconds based on pending pods |
| CloudWatch Container Insights | Node and pod-level metrics, logs, and performance dashboards |
| ECR | Private container registry — integrates natively with EKS node IAM roles |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
