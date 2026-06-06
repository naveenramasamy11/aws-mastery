# ☁️ Transit Gateway & AWS PrivateLink — AWS Mastery

> **The two networking primitives that replace "just peer everything" with hub-and-spoke at scale and "just expose the service publicly" with private, VPC-native endpoints.**

---

## 📖 Concept

### Transit Gateway (TGW)

VPC Peering is great for connecting 2-3 VPCs. When you have 20 VPCs, peering becomes a full-mesh nightmare — N*(N-1)/2 connections, no transitive routing, separate route table entries for every peer. Transit Gateway solves this with a hub-and-spoke model: all VPCs attach to TGW, and routing is managed centrally via route tables on TGW itself.

TGW supports attachments for VPCs, VPN connections, Direct Connect Gateways, and other TGWs (TGW Peering for multi-region). Each attachment gets associated with a TGW route table. You can have multiple route tables on one TGW to implement routing isolation — e.g., prod VPCs can't reach dev VPCs even through TGW, because prod attaches to the "prod" route table that has no route to dev.

**Key TGW concepts:**
- **Attachments** — what connects to TGW (VPCs, VPNs, DX Gateways, other TGWs)
- **Route Tables** — TGW-side routing tables that determine where traffic goes
- **Associations** — which route table an attachment uses for routing decisions
- **Propagations** — automatic advertisement of an attachment's CIDR into a route table

### AWS PrivateLink

PrivateLink lets you expose services (your own, or AWS services, or partner services) to consumers via Elastic Network Interfaces (ENIs) in the consumer's VPC — without VPC peering, internet gateways, or public IPs. The consumer creates a VPC Endpoint (Interface type), which creates ENIs in their subnets with private IPs. Traffic flows entirely within the AWS network.

From the service side: you put your service behind a Network Load Balancer, then create a VPC Endpoint Service. Consumer accounts then create Interface VPC Endpoints pointing to your endpoint service. You control which accounts/principals can connect.

This is the architecture behind every `*.amazonaws.com` service endpoint — S3 Interface endpoints, SSM, Secrets Manager, ECR — all are PrivateLink under the hood.

---

## 🏗️ Architecture Snapshot

```
Transit Gateway Hub-and-Spoke + PrivateLink
──────────────────────────────────────────────────────────────────

  Multi-Account AWS Environment (ap-south-1)
  
  ┌────────────────────────────────────────────────────────────┐
  │                  Transit Gateway                           │
  │  ┌─────────────────┐    ┌─────────────────────────────┐   │
  │  │ Route Table:    │    │ Route Table:                │   │
  │  │ PROD            │    │ SHARED                      │   │
  │  │ 10.0.0.0/8→VPC  │    │ 172.16.0.0/12→Shared       │   │
  │  │ 0.0.0.0/0→Egress│    │                             │   │
  │  └─────────────────┘    └─────────────────────────────┘   │
  └──────────┬─────────────────────────┬──────────────────────┘
             │                         │
     ┌───────┼────────┐        ┌───────┼────────┐
     ▼       ▼        ▼        ▼       ▼        ▼
  VPC-Prod VPC-Prod  Egress   Shared  Tools    Dev
  App-A    App-B     VPC      Services VPC     VPCs
  (10.1)   (10.2)   (Internet)(172.16) (172.17)
  
  
  PrivateLink for Internal Service Exposure:
  
  Provider Account               Consumer Account
  ┌────────────────────┐         ┌────────────────────┐
  │  VPC (10.1.0.0/16) │         │  VPC (10.2.0.0/16) │
  │                    │         │                    │
  │  App Servers       │         │  Interface VPC     │
  │  [EC2/ECS pods]    │         │  Endpoint          │
  │       │            │         │  (ENI: 10.2.1.45)  │
  │       ▼            │         │       │            │
  │  Network Load      │PrivateLink│      │            │
  │  Balancer ─────────┼─────────┼───────┘            │
  │                    │ (no     │  DNS: service.vpce  │
  │  Endpoint Service  │  internet│  .amazonaws.com    │
  └────────────────────┘)        └────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Multi-account landing zone hub-and-spoke:** 50 product VPCs all attach to TGW. Centralized egress VPC with NAT Gateway handles all outbound internet traffic — no NAT in each VPC, saving $30k+/year. Centralized inspection VPC with Network Firewall inspects east-west traffic between prod and shared services.
- **PrivateLink for cross-account microservices:** A shared authentication service (Account A) is exposed via PrivateLink. All 20 product accounts (Account B-U) create Interface Endpoints to consume it. No VPC peering, no shared network, no public endpoints — each consumer sees it as a private IP in their own subnet.
- **Migration connectivity:** During a migration wave, TGW connects the on-premises Direct Connect/VPN to all AWS VPCs simultaneously. As workloads migrate, new VPCs are attached to TGW without reconfiguring any on-premises routing.

---

## 🔧 AWS CLI & Console Examples

### Create Transit Gateway

```bash
aws ec2 create-transit-gateway \
  --description "Central TGW for ap-south-1" \
  --options '{
    "AmazonSideAsn": 64512,
    "AutoAcceptSharedAttachments": "disable",
    "DefaultRouteTableAssociation": "disable",
    "DefaultRouteTablePropagation": "disable",
    "VpnEcmpSupport": "enable",
    "DnsSupport": "enable",
    "MulticastSupport": "disable"
  }' \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=prod-tgw}]' \
  --region ap-south-1

# Get the TGW ID
TGW_ID=$(aws ec2 describe-transit-gateways \
  --filters "Name=tag:Name,Values=prod-tgw" \
  --query 'TransitGateways[0].TransitGatewayId' \
  --output text \
  --region ap-south-1)
```

### Attach VPC to TGW

```bash
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id vpc-0prod123 \
  --subnet-ids subnet-a1 subnet-b1 subnet-c1 \
  --options "DnsSupport=enable,Ipv6Support=disable,ApplianceModeSupport=disable" \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=prod-app-a-attach}]' \
  --region ap-south-1
```

### Create TGW Route Table and Associate

```bash
# Create a prod route table
aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id $TGW_ID \
  --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=prod-rt}]' \
  --region ap-south-1

PROD_RT_ID=$(aws ec2 describe-transit-gateway-route-tables \
  --filters "Name=tag:Name,Values=prod-rt" \
  --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' \
  --output text \
  --region ap-south-1)

# Associate attachment with prod route table
aws ec2 associate-transit-gateway-route-table \
  --transit-gateway-route-table-id $PROD_RT_ID \
  --transit-gateway-attachment-id tgw-attach-0prod123 \
  --region ap-south-1

# Enable propagation (auto-publish VPC CIDR to route table)
aws ec2 enable-transit-gateway-route-table-propagation \
  --transit-gateway-route-table-id $PROD_RT_ID \
  --transit-gateway-attachment-id tgw-attach-0prod123 \
  --region ap-south-1

# Add default route pointing to Egress VPC
aws ec2 create-transit-gateway-route \
  --transit-gateway-route-table-id $PROD_RT_ID \
  --destination-cidr-block "0.0.0.0/0" \
  --transit-gateway-attachment-id tgw-attach-0egress123 \
  --region ap-south-1
```

### Share TGW Across Accounts via RAM

```bash
aws ram create-resource-share \
  --name "TGW-Share" \
  --resource-arns "arn:aws:ec2:ap-south-1:123456789012:transit-gateway/$TGW_ID" \
  --principals "arn:aws:organizations::123456789012:organization/o-abc123" \
  --region ap-south-1
```

### Create VPC Endpoint (PrivateLink for AWS Services)

```bash
# Interface endpoint for SSM (no more public internet for SSM access)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0prod123 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.ap-south-1.ssm \
  --subnet-ids subnet-priv-a subnet-priv-b \
  --security-group-ids sg-endpoint-sg \
  --private-dns-enabled \
  --region ap-south-1

# Interface endpoint for Secrets Manager
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0prod123 \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.ap-south-1.secretsmanager \
  --subnet-ids subnet-priv-a subnet-priv-b \
  --security-group-ids sg-endpoint-sg \
  --private-dns-enabled \
  --region ap-south-1

# Gateway endpoint for S3 (free, route-table based)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-0prod123 \
  --vpc-endpoint-type Gateway \
  --service-name com.amazonaws.ap-south-1.s3 \
  --route-table-ids rtb-private \
  --region ap-south-1
```

### Terraform — TGW + VPC Attachment

```hcl
resource "aws_ec2_transit_gateway" "main" {
  description                     = "Central TGW"
  amazon_side_asn                 = 64512
  auto_accept_shared_attachments  = "disable"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  dns_support                     = "enable"
  vpn_ecmp_support                = "enable"

  tags = { Name = "prod-tgw" }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "prod_app" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = var.prod_vpc_id
  subnet_ids         = var.prod_private_subnet_ids
  dns_support        = "enable"

  transit_gateway_default_route_table_association = false
  transit_gateway_default_route_table_propagation = false

  tags = { Name = "prod-app-tgw-attach" }
}

resource "aws_vpc_endpoint" "ssm" {
  vpc_id              = var.prod_vpc_id
  service_name        = "com.amazonaws.${var.region}.ssm"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.endpoint_sg.id]
  private_dns_enabled = true

  tags = { Name = "ssm-endpoint" }
}
```

---

## 🔐 Security Best Practices

- **Disable default route table association and propagation on TGW.** The default is to auto-associate and propagate, meaning every VPC can route to every other VPC. This defeats the purpose of segmentation. Always manage associations explicitly.
- **Use TGW route tables for network segmentation instead of Security Groups alone.** Security Groups are stateful and instance-level. TGW route table isolation prevents traffic from even reaching the VPC — defense in depth.
- **Always add Security Groups on Interface Endpoints.** Interface endpoints create ENIs in your subnets. Apply Security Groups that only allow traffic from your application subnets/SGs. Allowing 0.0.0.0/0 on endpoint SGs defeats private access.
- **Enable Private DNS on Interface Endpoints.** With private DNS enabled, `ssm.ap-south-1.amazonaws.com` resolves to the endpoint's private IP inside the VPC — existing code doesn't need to change endpoint URLs.

---

## 😄 Funny Things to Try

```bash
# See all your VPC endpoints and their states
aws ec2 describe-vpc-endpoints \
  --region ap-south-1 \
  --query 'VpcEndpoints[*].[VpcEndpointId,ServiceName,State,VpcId]' \
  --output table
# The "pending" state on new endpoints can last 2-3 minutes.
# During that time, SSM/Secrets Manager calls will fail.
# Great for "why is my Lambda timing out RIGHT after deployment?" debugging.

# Check TGW route table to see all routes (the network map)
aws ec2 search-transit-gateway-routes \
  --transit-gateway-route-table-id $PROD_RT_ID \
  --filters "Name=state,Values=active" \
  --region ap-south-1 \
  --query 'Routes[*].[DestinationCidrBlock,TransitGatewayAttachments[0].TransitGatewayAttachmentId,State]' \
  --output table
# Every CIDR your prod VPCs can reach, in one table.
# The "wait, how does 10.99.0.0/16 get there?" conversation ends here.
```

---

## ⚠️ Gotchas & Tricky Bits

- **TGW routing is not transitive by default.** Attaching VPC-A and VPC-B to TGW doesn't mean A and B can talk — you need propagation enabled on the route table AND routes pointing to each other's attachments. TGW just provides the transit fabric; routing must be configured explicitly.
- **Overlapping CIDRs on TGW will break routing.** TGW cannot route to overlapping CIDRs across attachments. If VPC-A (10.0.0.0/16) and VPC-B (10.0.0.0/16) both attach to TGW, only one gets a route. Plan CIDRs before you build — this is the migration pre-work nobody wants to do but everyone has to.
- **Interface Endpoints cost money even when idle.** Each Interface Endpoint has an hourly charge (~$0.01/hr) plus data processing. In a large landing zone with 15 VPCs × 10 endpoints each, this adds up. Use centralised endpoints in a shared services VPC and route through TGW instead of per-VPC endpoints.
- **VPC Endpoint SG must allow HTTPS (443).** All AWS service Interface Endpoints use HTTPS. If your endpoint SG blocks port 443, calls silently time out — not a clear auth error. Always allow 443 inbound from your application subnets.
- **Pro Tip:** For isolated dev environments, use Gateway Endpoints (S3 and DynamoDB only — free) plus SSM Session Manager through a shared TGW path to a centralised bastion VPC. This eliminates the need for VPN or public bastions in dev VPCs and saves ~$40/month in NAT Gateway costs per small dev VPC.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → VPC → Transit Gateways`
2. **Create:** Click "Create transit gateway" → disable default route table association and propagation
3. **Attach VPCs:** Go to "Transit gateway attachments" → "Create transit gateway attachment" → select TGW and VPC
4. **Key field:** Select subnets in each AZ for the attachment — these subnets become TGW "attachment subnets" and need enough IPs (TGW uses one IP per subnet)
5. **Common mistake here:** Using public subnets for TGW attachments — use private subnets; the attachment subnet should not need internet access
6. **For PrivateLink:** Go to `VPC → Endpoints → Create endpoint` → search service name → select Interface type → choose subnets
7. **Verify connectivity:**
   ```bash
   # From an EC2 instance, verify SSM endpoint resolution
   nslookup ssm.ap-south-1.amazonaws.com
   # Should resolve to 10.x.x.x (your VPC CIDR) not a public IP
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| AWS RAM (Resource Access Manager) | Shares TGW across accounts in an AWS Organization without per-account VPC peering |
| AWS Network Firewall | Deploy in a centralised "inspection VPC" attached to TGW to inspect all east-west traffic |
| Direct Connect Gateway | Attach to TGW to extend on-premises connectivity to all VPCs through a single DX attachment |
| Route 53 Resolver | Combine with TGW for cross-VPC DNS resolution; resolver rules propagate via shared services VPC |
| AWS PrivateLink (VPC Endpoint Services) | Your own services behind NLB exposed privately across accounts — the provider side of PrivateLink |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
