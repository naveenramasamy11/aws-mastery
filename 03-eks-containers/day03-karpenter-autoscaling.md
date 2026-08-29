# ☁️ Karpenter Autoscaling — AWS Mastery

> **Cluster Autoscaler asks "which ASG has room?" Karpenter asks "what's the cheapest instance that fits this pod?" — and that difference is where the cost savings live.**

---

## 📖 Concept

On every EKS migration I've run, the moment Cluster Autoscaler stops being enough is the moment workloads become heterogeneous — some pods need GPU, some need high memory, some are fine on Spot, and pre-defining a Managed Node Group per shape becomes an operational tax nobody wants to pay. Karpenter flips the model: instead of scaling predefined node groups up and down, it watches for unschedulable pods and directly provisions the right EC2 instance type, size, and purchase option (On-Demand or Spot) for exactly what's pending — often within seconds, and without you ever defining an Auto Scaling Group.

The core objects are `NodePool` (what kinds of nodes are allowed — instance families, zones, capacity type) and `EC2NodeClass` (the AWS-specific details — AMI, subnets, security groups, IAM role). Karpenter uses a bin-packing algorithm that considers actual pod resource requests, not just node counts, which is why a well-tuned Karpenter setup routinely runs 20-30% cheaper than an equivalent Cluster Autoscaler + fixed node group setup on the same workload — it isn't rounding up to the next node group size, it's picking the instance that fits.

The other structural win for ProServe migration work: Karpenter's **consolidation** feature continuously looks for cheaper ways to run the current pod set and proactively replaces nodes — draining and terminating an underutilized `m5.2xlarge` in favor of two `m5.large` instances if that's cheaper, without anyone opening a ticket. Combined with Spot interruption handling built directly into the controller, this is the autoscaler I now default to on any EKS workload that isn't tightly latency-bound to specific pinned node shapes.

---

## 🏗️ Architecture Snapshot

```
                    Unschedulable Pod
                          │
                          ▼
              ┌───────────────────────┐
              │   Karpenter Controller │
              │   (watches API server) │
              └───────────┬────────────┘
                          │ evaluates
                          ▼
       ┌──────────────────────────────────┐
       │  NodePool: general-purpose        │
       │  - instance families: m,c,r       │
       │  - capacity type: spot > on-demand│
       │  - zones: ap-south-1a/1b/1c       │
       └──────────────┬─────────────────────┘
                          │ references
                          ▼
       ┌──────────────────────────────────┐
       │  EC2NodeClass                    │
       │  - AMI family: AL2023             │
       │  - subnetSelector, sgSelector     │
       │  - instanceProfile (IAM)          │
       └──────────────┬─────────────────────┘
                          │ provisions
                          ▼
              ┌───────────────────────┐
              │  Right-sized EC2 node  │
              │  joins cluster in <60s │
              └───────────────────────┘

Consolidation loop: continuously replaces nodes with cheaper-fit alternatives.
```

---

## 💡 Real-World Use Cases

- **Bursty batch workloads:** A data-processing pipeline that spikes from 5 to 200 pods for an hour nightly gets exactly the nodes it needs for that hour, then Karpenter consolidates back down — no pre-provisioned headroom sitting idle all day.
- **Mixed Spot/On-Demand fleets without separate node groups:** One NodePool with `capacity-type: [spot, on-demand]` lets Karpenter prefer Spot for stateless workloads and fall back automatically when Spot capacity is unavailable.
- **Multi-architecture migration (x86 to Graviton):** A NodePool constrained to `arm64` instance families lets you migrate workloads to Graviton incrementally, pod by pod, without touching existing node groups.

---

## 🔧 AWS CLI & Console Examples

### Minimal NodePool + EC2NodeClass

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose
spec:
  template:
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values: ["m5", "m6i", "c5", "r5"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot", "on-demand"]
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: KarpenterNodeRole-my-cluster
  subnetSelectorTerms:
    - tags: { karpenter.sh/discovery: my-cluster }
  securityGroupSelectorTerms:
    - tags: { karpenter.sh/discovery: my-cluster }
```

### Check what Karpenter is about to provision

```bash
kubectl get nodepool -o wide
kubectl describe nodeclaim
```

### IAM role trust for the Karpenter controller (IRSA)

```bash
eksctl create iamserviceaccount \
  --cluster my-cluster \
  --namespace karpenter \
  --name karpenter \
  --role-name KarpenterControllerRole-my-cluster \
  --attach-policy-arn arn:aws:iam::123456789012:policy/KarpenterControllerPolicy \
  --approve \
  --region ap-south-1
```

---

## 🔐 Security Best Practices

- **Scope the Karpenter node IAM role tightly** — it only needs permissions to launch/terminate instances it created and pull from ECR; never attach broad EC2 admin permissions to `KarpenterNodeRole`.
- **Tag subnets and security groups with `karpenter.sh/discovery`** deliberately and audit that tag — an overly broad discovery tag can let Karpenter place nodes in subnets you didn't intend.
- **Enforce Pod Security Standards on Karpenter-provisioned nodes** — new nodes join with default kubelet settings; don't assume node-level hardening from your old Managed Node Groups carries over automatically.

---

## 😄 Funny Things to Try

```bash
# Watch Karpenter provision a node in near-real-time
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter

# Force a consolidation decision and watch it right-size your cluster
kubectl get nodeclaims -o custom-columns=NAME:.metadata.name,TYPE:.status.allocatable
# Then delete a deployment's replica and watch Karpenter quietly
# consolidate two half-empty nodes into one — no ticket required.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Karpenter does not manage the control plane or CoreDNS placement** — it's purely a node-provisioning controller; you still need topology spread constraints for critical add-ons.
- **Spot interruption notices give only a 2-minute warning** — Karpenter handles the drain automatically, but pods without `PodDisruptionBudget`s can still be killed ungracefully; always set PDBs on anything running on Spot NodePools.
- **Consolidation can be surprisingly aggressive** — `WhenEmptyOrUnderutilized` will happily churn nodes during low-traffic windows; for latency-sensitive workloads, use `WhenEmpty` instead to avoid mid-day node replacement.
- **Pro Tip:** Run Karpenter alongside Cluster Autoscaler only during migration cutover, never long-term — the two controllers can fight over the same unschedulable pods and cause duplicate node launches.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → EKS → Clusters → [cluster] → Compute` (Karpenter nodes appear as standard EC2 instances, not a distinct EKS object).
2. **Look for:** instances tagged `karpenter.sh/nodepool` — this confirms Karpenter, not a Managed Node Group, launched them.
3. **Key field:** the `karpenter.sh/capacity-type` tag on each instance — confirms whether it landed on Spot or On-Demand.
4. **Common mistake here:** assuming Karpenter nodes show up under "Node groups" in the EKS console — they don't; they're plain EC2 instances outside that view.
5. **Confirm with CLI:**
   ```bash
   kubectl get nodes -L karpenter.sh/capacity-type,node.kubernetes.io/instance-type
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| EC2 Spot | Karpenter's primary lever for cost savings on interruption-tolerant workloads |
| IRSA | Grants the Karpenter controller pod scoped IAM permissions to launch/terminate nodes |
| CloudWatch Container Insights | Surfaces per-node utilization that validates consolidation decisions |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
