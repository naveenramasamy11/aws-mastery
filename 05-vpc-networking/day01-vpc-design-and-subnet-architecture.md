# ☁️ VPC Design & Subnet Architecture — AWS Mastery

> **Your VPC CIDR is a tattoo — plan it before you ink it, because changing it later hurts.**

---

## 📖 Concept

A VPC (Virtual Private Cloud) is your private, logically isolated network inside AWS. Every resource — EC2 instances, EKS nodes, RDS databases, Lambda functions in a VPC — lives in a subnet inside a VPC. The design decisions you make at VPC creation time (CIDR ranges, subnet sizing, AZ distribution) are extremely hard to change later and directly impact your ability to scale, connect to on-premises networks, and comply with security requirements.

The three-tier subnet model is the AWS standard: **Public subnets** (internet-facing load balancers, NAT Gateways), **Private subnets** (application servers, EKS nodes, Lambda), and **Isolated/Data subnets** (RDS, ElastiCache, with no route to internet at all). Each tier exists in every AZ you deploy to — for ap-south-1, that means 3 tiers × 3 AZs = 9 subnets minimum per VPC.

Security Groups and NACLs both filter traffic, but they work differently in a critical way: Security Groups are **stateful** (return traffic is automatically allowed), while NACLs are **stateless** (you must explicitly allow both inbound AND outbound traffic, including ephemeral ports 1024-65535). NACLs are evaluated before Security Groups. Most teams only configure NACLs for broad subnet-level blocking (e.g., blocking a malicious IP range) and rely on Security Groups for application-level control.

Transit Gateway (TGW) replaces the old VPC Peering mesh for multi-VPC and hybrid connectivity. Where VPC Peering requires a peering connection between every pair of VPCs (N×(N-1)/2 connections for N VPCs), TGW acts as a central hub — all VPCs attach to TGW, and routing is managed centrally via TGW route tables. For enterprises with 10+ VPCs, TGW is not optional.

---

## 🏗️ Architecture Snapshot

```
  AWS VPC — Three-Tier Architecture (Multi-AZ)
  ──────────────────────────────────────────────────────

  VPC: 10.0.0.0/16
  ┌──────────────────────────────────────────────────────┐
  │  AZ-A (ap-south-1a)  │  AZ-B (ap-south-1b)          │
  │  ┌──────────────────┐│  ┌──────────────────────────┐ │
  │  │ Public           ││  │ Public                   │ │
  │  │ 10.0.1.0/24      ││  │ 10.0.2.0/24              │ │
  │  │ [ALB] [NAT GW]   ││  │ [ALB] [NAT GW]           │ │
  │  └────────┬─────────┘│  └──────────┬───────────────┘ │
  │           │           │             │                  │
  │  ┌────────▼─────────┐│  ┌──────────▼───────────────┐ │
  │  │ Private          ││  │ Private                   │ │
  │  │ 10.0.11.0/24     ││  │ 10.0.12.0/24              │ │
  │  │ [EKS Nodes]      ││  │ [EKS Nodes]               │ │
  │  │ [EC2 App Servers]││  │ [EC2 App Servers]         │ │
  │  └────────┬─────────┘│  └──────────┬───────────────┘ │
  │           │           │             │                  │
  │  ┌────────▼─────────┐│  ┌──────────▼───────────────┐ │
  │  │ Isolated/Data    ││  │ Isolated/Data             │ │
  │  │ 10.0.21.0/24     ││  │ 10.0.22.0/24              │ │
  │  │ [RDS] [Cache]    ││  │ [RDS] [Cache]             │ │
  │  └──────────────────┘│  └──────────────────────────┘ │
  └──────────────────────┴──────────────────────────────┘
        │ Internet Gateway (IGW)       │ Transit Gateway
        ▼                              ▼
    Internet                    On-Premises / Other VPCs
```

---

## 💡 Real-World Use Cases

- **EKS subnet sizing:** For a cluster running 150+ microservices on `m7g.2xlarge` nodes, each node needs ~20 pod IPs (VPC CNI). With 10 nodes per AZ, that's 200+ IPs per AZ — private subnets need `/19` (8,190 IPs) or larger, not `/24` (256 IPs).
- **Transit Gateway for multi-account architecture:** A bank with separate AWS accounts for dev, staging, prod, shared-services, and security connects all of them through a centralized TGW in the network account — all routing controlled by the network team, no ad-hoc VPC peering.
- **PrivateLink for SaaS integration:** A SaaS vendor exposes their API as a VPC Endpoint Service (PrivateLink). Customer connects via Interface VPC Endpoint — traffic never leaves the AWS network, no internet exposure, no NAT Gateway charges.

---

## 🔧 AWS CLI & Console Examples

### VPC and Subnet Inspection

```bash
# List all VPCs with their CIDR and name
aws ec2 describe-vpcs \
  --region ap-south-1 \
  --query 'Vpcs[].{ID:VpcId,CIDR:CidrBlock,Name:Tags[?Key==`Name`].Value|[0]}' \
  --output table

# List subnets in a VPC with AZ and available IPs
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=vpc-0abc123" \
  --query 'Subnets[].{ID:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock,AvailableIPs:AvailableIpAddressCount,Public:MapPublicIpOnLaunch}' \
  --output table \
  --region ap-south-1

# Check route tables for a subnet
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-0abc123" \
  --query 'RouteTables[].Routes[].{Dest:DestinationCidrBlock,Target:GatewayId,NatGW:NatGatewayId}' \
  --output table
```

### Security Group Audit

```bash
# Find all security groups with 0.0.0.0/0 inbound (the "open to world" audit)
aws ec2 describe-security-groups \
  --region ap-south-1 \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].{ID:GroupId,Name:GroupName,Ports:IpPermissions[].FromPort}' \
  --output table

# Find security groups with port 22 open to world (SSH exposure)
aws ec2 describe-security-groups \
  --filters "Name=ip-permission.from-port,Values=22" \
              "Name=ip-permission.cidr,Values=0.0.0.0/0" \
  --query 'SecurityGroups[].{ID:GroupId,Name:GroupName,VPC:VpcId}' \
  --output table \
  --region ap-south-1
```

### VPC Flow Logs Setup

```bash
# Enable VPC Flow Logs to CloudWatch Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-0abc123 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/flowlogs-role \
  --region ap-south-1

# Query flow logs in CloudWatch Logs Insights (rejected traffic to port 22)
# Use in CloudWatch Logs Insights console:
# fields @timestamp, srcAddr, dstPort, action
# | filter dstPort = 22 and action = "REJECT"
# | stats count(*) by srcAddr
# | sort count desc
# | limit 20
```

### Transit Gateway Attachment

```bash
# List TGW attachments
aws ec2 describe-transit-gateway-attachments \
  --region ap-south-1 \
  --query 'TransitGatewayAttachments[].{ID:TransitGatewayAttachmentId,Type:ResourceType,State:State,VPCID:ResourceId}' \
  --output table

# Check TGW route table
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id tgw-rtb-0abc123 \
  --filters "Name=state,Values=active" \
  --query 'Routes[].{CIDR:DestinationCidrBlock,Attachment:TransitGatewayAttachments[0].TransitGatewayAttachmentId}' \
  --output table
```

---

## 🔐 Security Best Practices

- **Never use 0.0.0.0/0 in Security Group inbound rules except for public ALBs on port 443/80:** Everything else should be scoped to specific CIDR ranges or other Security Group IDs. Reference Security Group to Security Group (not CIDR) wherever possible within the same VPC.
- **Enable VPC Flow Logs on all production VPCs:** They're your network forensics tool. Without them, you can't answer "was this IP ever in our environment?" after a security incident. Send to S3 with lifecycle policy, not just CloudWatch Logs.
- **Use NAT Gateway per AZ, not one shared:** A single NAT Gateway is a single point of failure and an AZ bottleneck. For production, deploy one NAT Gateway per AZ and update private route tables to route to the NAT GW in the same AZ.
- **Use Interface VPC Endpoints for AWS services:** Traffic to S3, DynamoDB, ECR, SSM, and other AWS services shouldn't go through the internet (and through your NAT Gateway). Interface endpoints keep traffic on the AWS backbone and eliminate NAT Gateway data processing charges.

---

## 😄 Funny Things to Try

```bash
# Find out how many IPs you've "wasted" — subnets where first 4 and last 1 are reserved
# AWS reserves 5 IPs per subnet (.0 network, .1 router, .2 DNS, .3 reserved, .255 broadcast)
aws ec2 describe-subnets --region ap-south-1 \
  --query 'Subnets[].{Subnet:SubnetId,Total:to_number(split(`/`,CidrBlock)[1]),Available:AvailableIpAddressCount}' \
  --output table
# Note: every /24 you create gives you 251 usable IPs, not 256. AWS took 5.

# The "how many NAT Gateways am I paying for?" check
aws ec2 describe-nat-gateways \
  --filter "Name=state,Values=available" \
  --query 'NatGateways[].{ID:NatGatewayId,AZ:SubnetId,State:State}' \
  --output table --region ap-south-1
# Each NAT GW costs ~$32/month + $0.045/GB. If you have one per AZ × 3 AZs = ~$96/month
# just to exist, before any data transfer.

# Find unused Elastic IPs (you're paying for these doing nothing)
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocationId:AllocationId}' \
  --output table --region ap-south-1
# Unassociated EIPs cost $0.005/hour = ~$3.60/month each. Delete them.
```

---

## ⚠️ Gotchas & Tricky Bits

- **Security Groups are VPC-scoped, not global:** You can reference a Security Group from another Security Group only within the same VPC (or with VPC peering). Across TGW, you must use CIDR-based rules.
- **NACL rule evaluation stops at first match:** NACLs are evaluated in numerical order — rule 100 is evaluated before rule 200. If rule 100 allows all traffic, rules 200+ are never reached. Plan rule numbers with gaps (100, 200, 300) so you can insert rules later.
- **Default VPC and default subnets are public:** The AWS default VPC has all subnets with `MapPublicIpOnLaunch=true`. Never use the default VPC for production workloads.
- **VPC Peering is not transitive:** If VPC-A peers with VPC-B, and VPC-B peers with VPC-C, VPC-A cannot reach VPC-C through VPC-B. You need direct peering or Transit Gateway.
- **Pro Tip:** Use the AWS VPC IP Address Manager (IPAM) to centrally plan, track, and audit IP address usage across all VPCs in your AWS Organization. It prevents CIDR overlaps that make VPC peering and on-premises connectivity impossible later.

---

## 📸 Console Walkthrough

> *Enabling VPC Flow Logs and querying them in CloudWatch Logs Insights*

1. **Navigate to:** `AWS Console → VPC → Your VPCs → [vpc-id] → Flow Logs tab`
2. **Click:** `Create flow log`
3. **Filter:** Select `All` (capture Accept and Reject traffic)
4. **Destination:** Choose `CloudWatch Logs` and create a new log group `/aws/vpc/flowlogs`
5. **IAM Role:** Create or select a role that allows `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`
6. **Key field:** `Log format` — use the default format first; add `pkt-srcaddr` and `pkt-dstaddr` for NAT gateway scenarios
7. **Common mistake here:** Setting filter to `Reject` only — you miss all the allowed traffic, which is what you need for baseline profiling
8. **Query in CloudWatch Logs Insights:**
   ```
   Navigate to: CloudWatch → Logs Insights → select /aws/vpc/flowlogs
   Query: fields srcAddr, dstPort, action | filter action="REJECT" | stats count(*) by srcAddr | sort count desc
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| Transit Gateway | Central hub for multi-VPC and hybrid network routing |
| AWS PrivateLink | Private connectivity to services without internet exposure |
| Route 53 Resolver | DNS resolution between VPCs and on-premises via inbound/outbound endpoints |
| Network Firewall | Stateful, managed firewall deployed in a VPC for deep packet inspection |
| Direct Connect | Dedicated network connection from on-premises to AWS — bypasses internet |
| Global Accelerator | Routes user traffic to nearest AWS edge, improves latency for global apps |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
