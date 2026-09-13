# ☁️ Route 53 Resolver Rules & Hybrid DNS — AWS Mastery

> **DNS is the thing nobody thinks about until a migrated app can't find its own database.**

---

## 📖 Concept

Every migration engagement eventually runs into a version of the same problem: an application newly running in a VPC needs to resolve a hostname that lives in the customer's on-prem Active Directory DNS, and an on-prem application needs to resolve a hostname for a service that just moved into AWS. The default VPC resolver (the `.2` address at the base of the VPC CIDR) only knows about records inside that VPC and public DNS — it has no idea an on-prem DNS server exists. Route 53 Resolver endpoints and rules are the bridge that makes hybrid DNS resolution actually work during the (often years-long) period where workloads are split between on-prem and cloud.

The architecture has two halves. An inbound resolver endpoint lets on-prem DNS servers forward queries for AWS-hosted private hosted zone records into the VPC resolver — so on-prem machines can resolve `db.internal.corp` that now points to an RDS instance in AWS. An outbound resolver endpoint paired with a resolver rule does the reverse: it forwards DNS queries for a specific domain (say, `corp.local`) from inside the VPC out to the customer's on-prem DNS servers over Direct Connect or VPN, so newly-migrated EC2 instances can still resolve legacy on-prem hostnames they depend on.

Resolver rules are shared across accounts via AWS RAM (Resource Access Manager), which is what makes centralized DNS architecture possible in a multi-account Landing Zone: one Network/Shared Services account owns the resolver endpoints and rules, and every spoke VPC in the organization associates with those shared rules instead of each team building their own hybrid DNS from scratch. Getting this wrong is one of the most common causes of "it works from my laptop but not from the migrated server" tickets in the first 90 days after a cutover.

---

## 🏗️ Architecture Snapshot

```
On-Premises Data Center                    AWS (Shared Services VPC)
┌───────────────────┐                    ┌────────────────────────┐
│  On-prem DNS Server │                    │  Route 53 Resolver        │
│  (corp.local zone)   │                    │  Inbound Endpoint (ENIs)  │
│           │           │  DNS query over    │           ▲              │
│           │           │  Direct Connect/VPN │           │              │
│           └───────────┼──────────────────┼───────────┘              │
│                        │                    │  resolves *.aws.corp     │
│                        │                    │  (private hosted zone)   │
│  On-prem App           │                    │                          │
│                        │                    │  Route 53 Resolver        │
│                        │  ◀────────────────────│──  Outbound Endpoint     │
│                        │   query forwarded   │   + Resolver Rule        │
│                        │   for corp.local     │   (corp.local → on-prem)│
│                        │                    │           ▲              │
│                        │                    │           │              │
│                        │                    │   EC2 instance in VPC    │
│                        │                    │   (needs corp.local DNS) │
└───────────────────┘                    └────────────────────────┘
                                             Rule shared via AWS RAM to
                                             every spoke VPC in the org
```

---

## 💡 Real-World Use Cases

- **Phased migration DNS continuity:** During a multi-month migration wave plan, newly-migrated EC2 instances resolve legacy on-prem service names via outbound resolver rules, avoiding a "big bang" DNS cutover that breaks everything at once.
- **Centralized hybrid DNS in a Landing Zone:** A Network Account owns all resolver endpoints and rules; every new spoke account/VPC just associates with shared rules via RAM, giving new accounts working hybrid DNS on day one.
- **Active Directory-integrated workloads:** Applications freshly migrated to EC2 still need to authenticate against an on-prem Active Directory domain controller — resolver rules ensure the AD SRV records resolve correctly from inside the VPC.

---

## 🔧 AWS CLI & Console Examples

### Create an outbound resolver endpoint

```bash
aws route53resolver create-resolver-endpoint \
  --name to-onprem-dns \
  --direction OUTBOUND \
  --security-group-ids sg-0123456789abcdef0 \
  --ip-addresses SubnetId=subnet-aaa,Ip=10.0.1.10 SubnetId=subnet-bbb,Ip=10.0.2.10 \
  --region ap-south-1
```

### Create a forwarding rule for the on-prem domain

```bash
aws route53resolver create-resolver-rule \
  --name corp-local-forward \
  --rule-type FORWARD \
  --domain-name "corp.local" \
  --target-ips Ip=10.100.0.10,Port=53 Ip=10.100.0.11,Port=53 \
  --resolver-endpoint-id rslvr-out-0123456789abcdef0
```

### Share the rule to spoke accounts via RAM

```bash
aws ram create-resource-share \
  --name shared-dns-rules \
  --resource-arns arn:aws:route53resolver:ap-south-1:111122223333:resolver-rule/rslvr-rr-0123456789abcdef0 \
  --principals 222233334444 333344445555
```

### Terraform — resolver rule association in a spoke VPC

```hcl
resource "aws_route53_resolver_rule_association" "spoke_assoc" {
  resolver_rule_id = "rslvr-rr-0123456789abcdef0"
  vpc_id           = aws_vpc.spoke.id
}
```

---

## 🔐 Security Best Practices

- **Restrict resolver endpoint security groups to only DNS ports (53 TCP/UDP) from known CIDRs:** An overly permissive security group on a resolver endpoint is a lateral-movement path into your DNS infrastructure.
- **Use Route 53 Resolver DNS Firewall alongside resolver rules:** Block known malicious domains and DNS tunneling attempts at the resolver layer — this is a near-zero-cost control most migration projects skip.
- **Centralize resolver rule ownership in the Network/Shared Services account:** Letting every spoke account create its own resolver rules leads to conflicting forwarding rules that are extremely hard to debug.

---

## 😄 Funny Things to Try

> *These are real commands that do unexpected/surprising/amusing things — always safe, always educational.*

```bash
# Ask a VPC instance to prove it can actually reach the on-prem domain
dig corp.local SOA +short
# If this times out, congratulations, you've found your first hybrid-DNS ticket.

# Count how many "temporary" resolver rules have quietly become permanent
aws route53resolver list-resolver-rules --query 'ResolverRules[?Status==`COMPLETE`] | length(@)'
```

---

## ⚠️ Gotchas & Tricky Bits

- **Resolver rule precedence isn't obvious:** More specific domain rules always win over less specific ones regardless of creation order — a rule for `db.corp.local` overrides a rule for `corp.local` even if created later.
- **Resolver endpoints need at least two subnets in different AZs:** A single-AZ resolver endpoint is a single point of failure that AWS won't let you create — plan subnet allocation ahead of time.
- **RAM-shared rules require explicit VPC association in each account:** Sharing the rule doesn't automatically apply it — every spoke account must still run `create-resolver-rule-association`.
- **Pro Tip:** DNS query logging (via Resolver Query Logging Config) is invaluable during a migration wave to prove which legacy DNS dependencies still exist before you can safely decommission on-prem DNS.

---

## 📸 Console Walkthrough

> *Step-by-step console path with what to look for at each step.*

1. **Navigate to:** `AWS Console → Route 53 → Resolver → Rules → Create rule`
2. **Look for:** The "Rule type" selector — Forward vs System vs Recursive.
3. **Key field:** `Outbound endpoint` — must already exist and be associated with subnets that route to on-prem via Direct Connect/VPN.
4. **Common mistake here:** Forgetting to associate the new rule with the VPCs that need it — a rule with zero associations does nothing.
5. **Confirm with CLI:**
   ```bash
   aws route53resolver list-resolver-rule-associations --query 'ResolverRuleAssociations[].{Rule:ResolverRuleId,VPC:VPCId,Status:Status}'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS Direct Connect / Site-to-Site VPN | Provides the network path resolver endpoints use to reach on-prem DNS servers |
| AWS Resource Access Manager (RAM) | Shares resolver rules centrally across every account in the Organization |
| Route 53 Resolver DNS Firewall | Adds malicious-domain blocking directly at the resolver layer |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
