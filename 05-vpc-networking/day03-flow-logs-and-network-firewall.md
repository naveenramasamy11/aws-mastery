# ☁️ VPC Flow Logs & AWS Network Firewall — AWS Mastery

> **Security Groups and NACLs tell you what's ALLOWED. Flow Logs tell you what actually HAPPENED — and only one of those catches the incident after the fact.**

---

## 📖 Concept

On more than one migration engagement, the moment of truth wasn't designing the VPC — it was three weeks after cutover when the customer's security team asked "can you show us exactly which IPs talked to this instance on this date?" Security Groups and NACLs are enforcement mechanisms; they don't retain history. VPC Flow Logs are the audit trail: metadata about every accepted and rejected IP flow at the ENI, subnet, or VPC level — source/destination IP and port, protocol, packet/byte counts, and crucially, whether the traffic was ACCEPTed or REJECTed. Sent to CloudWatch Logs or S3, they become queryable with Logs Insights or Athena, which is how you answer "who talked to what, when" after the fact instead of only being able to say "here's what's currently allowed."

AWS Network Firewall sits at a different layer entirely — it's a stateful, deep-packet-inspection firewall you deploy at the VPC boundary (typically in a dedicated subnet behind a Gateway Load Balancer or as a Transit Gateway attachment) to enforce domain-based and signature-based rules that Security Groups simply can't express: blocking outbound traffic to known malicious domains, restricting egress to an explicit allow-list of external APIs, or inspecting east-west traffic between VPCs in a hub-and-spoke topology. Where a Security Group can only reason about IP/port, Network Firewall can reason about domain names, TLS SNI, and Suricata-compatible IDS/IPS rules.

On one ProServe engagement moving a customer off a legacy on-prem firewall appliance during an EKS migration, we replaced perimeter domain-filtering entirely with Network Firewall rule groups, then used Flow Logs on the EKS worker node subnets to validate, before cutover, that nothing outside the approved domain list was actually being used in production — turning a risky "trust the migration" moment into a data-backed one.

---

## 🏗️ Architecture Snapshot

```
                     Internet
                        │
              ┌──────────▼──────────┐
              │  Internet Gateway  │
              └──────────┬──────────┘
                         │
              ┌──────────▼───────────────────┐
              │  Firewall Subnet            │
              │  ┌────────────────────────┐ │
              │  │  AWS Network Firewall   │ │  ← domain filtering,
              │  │  (stateful rule groups) │ │    Suricata IDS/IPS
              │  └──────────────────────────┘ │
              └──────────┬────────────────────┘
                         │
              ┌──────────▼──────────┐
              │  NAT Gateway        │
              └──────────┬──────────┘
                         │
        ┌────────────────┼──────────────────┐
        ▼                                  ▼
┌───────────────┐                 ┌───────────────┐
│ Private Subnet │                 │ Private Subnet │
│ EKS Node Group │                 │ RDS Instance   │
│  [Flow Logs]───┼──▶ CloudWatch   │  [Flow Logs]───┼──▶ S3 (Athena queries)
└───────────────┘    Logs Insights └───────────────┘
```

---

## 💡 Real-World Use Cases

- **Post-incident forensics:** Query Flow Logs with CloudWatch Logs Insights to reconstruct exactly which source IP hit a compromised instance and on which port, within minutes of the SOC raising the alert.
- **Egress allow-listing for compliance:** Network Firewall domain rules restrict outbound traffic from a PCI-scoped subnet to only the payment processor's known domains — blocking everything else by default.
- **Pre-cutover traffic validation:** Flow Logs on a shadow/parallel environment confirm the actual dependency graph of a legacy app before its firewall rules are decommissioned during migration.

---

## 🔧 AWS CLI & Console Examples

### Enable VPC Flow Logs to CloudWatch

```bash
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-0123456789abcdef0 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/flow-logs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flow-logs-role \
  --region ap-south-1
```

### Query rejected traffic with Logs Insights

```
fields @timestamp, srcAddr, dstAddr, dstPort, action
| filter action = "REJECT"
| sort @timestamp desc
| limit 50
```

### Create a Network Firewall domain-filtering rule group

```bash
aws network-firewall create-rule-group \
  --rule-group-name "allow-approved-domains" \
  --type STATEFUL \
  --capacity 100 \
  --rule-group file://rule-group.json \
  --region ap-south-1
```

---

## 🔐 Security Best Practices

- **Log REJECTed traffic, not just accepted** — the default `ALL` traffic type captures both; teams that only log ACCEPT miss the exact evidence needed for intrusion investigation.
- **Restrict who can modify Network Firewall rule groups via IAM** — a firewall rule change is as sensitive as an SCP change; scope `network-firewall:UpdateRuleGroup` tightly and require change review.
- **Send Flow Logs to S3 for long-term retention, CloudWatch for near-real-time alerting** — use both destinations; CloudWatch retention gets expensive fast at scale, S3 with lifecycle rules is the durable archive.

---

## 😄 Funny Things to Try

```bash
# Find your own noisiest talker in the last hour
aws logs start-query \
  --log-group-name /vpc/flow-logs \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'stats sum(bytes) as total by srcAddr | sort total desc | limit 5'
# Nine times out of ten it's a health check. The tenth time is why you log this.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Flow Logs don't capture DNS traffic or traffic to/from the AWS DNS resolver** — for domain-level visibility, pair Flow Logs with Route 53 Resolver query logging, not Flow Logs alone.
- **Flow Log records can arrive with several minutes of delay** — don't rely on them for real-time blocking decisions; that's Network Firewall's job, not Flow Logs'.
- **Network Firewall adds real latency and cost per GB inspected** — it's not free-to-deploy-everywhere; scope it to the perimeter and inter-VPC boundaries that actually need deep inspection, not every subnet.
- **Pro Tip:** Flow Log fields differ between the default format and the custom format — always specify a custom format explicitly if you need fields like `pkt-srcaddr` or `tcp-flags` for advanced correlation, since the default format omits them.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → VPC → Your VPCs → [VPC] → Flow logs tab`
2. **Look for:** the "Create flow log" button — note the destination options (CloudWatch Logs vs S3) presented here.
3. **Key field:** `Log record format` — switch from AWS default to Custom to capture fields needed for deeper analysis.
4. **Common mistake here:** enabling Flow Logs after an incident instead of proactively — there's no retroactive capture, so this is a "turn it on before you need it" control.
5. **Confirm with CLI:**
   ```bash
   aws ec2 describe-flow-logs --filter "Name=resource-id,Values=vpc-0123456789abcdef0"
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| Athena | Queries Flow Logs stored in S3 at scale, cheaper than CloudWatch for historical analysis |
| GuardDuty | Consumes VPC Flow Logs as one of its primary threat-detection data sources |
| Transit Gateway | Common attachment point for centralizing Network Firewall inspection across multiple VPCs |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
