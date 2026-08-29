# ☁️ SSM Session Manager — No Bastion Hosts, No Open SSH — AWS Mastery

> **The day you delete your last bastion host security group rule is the day your attack surface actually shrinks.**

---

## 📖 Concept

Every migration wave plan I've built eventually hits the "how do operators reach private EC2 instances" question, and for years the answer was a bastion host: a jump box sitting in a public subnet with port 22 open to a VPN CIDR, patched religiously, and still somehow the first thing a pentest flags. AWS Systems Manager Session Manager removes the bastion entirely. The SSM Agent running on the instance makes an **outbound** HTTPS connection to the Systems Manager service — no inbound port needs to be open, not even from a VPN range. Authentication and authorization happen through IAM, not SSH keys, which means access is centrally auditable, revocable in seconds, and covered by CloudTrail and Session Manager logging by default.

The architectural shift that matters here: the instance now needs an IAM instance profile with `AmazonSSMManagedInstanceCore` attached, and it needs network reachability to the SSM, SSM messages, and EC2 messages endpoints — either via public internet through a NAT Gateway, or, in a fully private subnet, via VPC Interface Endpoints for those three services. On EKS-adjacent migration engagements, I've used SSM Session Manager on worker nodes specifically so no node ever needs a public IP or an SSH key distributed to a build pipeline — access is IAM-role-based and every session is logged to CloudWatch or S3 automatically.

The other underrated win: `port forwarding` through Session Manager gives you a tunnel to private resources (an RDS instance, an internal load balancer) without a bastion or a VPN client, which is exactly the kind of thing a security team stops fighting you about once they see the audit trail it produces.

---

## 🏗️ Architecture Snapshot

```
┌─────────────────────────────── VPC ─────────────────────────────────┐
│                                                               │
│   Private Subnet                                             │
│   ┌───────────────┐        Outbound HTTPS (443)              │
│   │  EC2 Instance │────────────────────┐                      │
│   │  SSM Agent    │                   │                      │
│   │  IAM Role:    │                   ▼                      │
│   │  SSMManaged   │        ┌──────────────────────┐           │
│   │  InstanceCore │        │ VPC Interface        │          │
│   └───────────────┘        │ Endpoints:           │          │
│                             │ ssm / ssmmessages /  │          │
│                             │ ec2messages          │          │
│                             └───────────┬───────────┘          │
└───────────────────────────────────────────┼────────────────────────┘
                                          │ PrivateLink
                                          ▼
                          ┌───────────────────────────────┐
                          │  AWS Systems Manager        │
                          │  (Session Manager backend)  │
                          └───────────────┬───────────────┘
                                          │
                                          ▼
                          ┌───────────────────────────────┐
                          │  Operator (IAM user/role)   │
                          │  aws ssm start-session       │
                          └───────────────────────────────┘

No inbound port 22. No bastion. Every session logged to CloudWatch/S3.
```

---

## 💡 Real-World Use Cases

- **Zero-bastion EKS worker node access:** Worker nodes in fully private subnets are reached only via `aws ssm start-session` — no SSH keys ever touch a CI/CD pipeline or a laptop.
- **Emergency database access without a VPN client:** Port-forward to a private RDS endpoint from a laptop with `aws ssm start-session --document-name AWS-StartPortForwardingSession` during an incident, with zero infrastructure changes.
- **Compliance-driven session recording:** A regulated customer required every privileged shell session to be recorded; Session Manager logs stdout/stdin to S3 automatically, satisfying the control without a third-party PAM tool.

---

## 🔧 AWS CLI & Console Examples

### Start an interactive session

```bash
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --region ap-south-1
```

### Port-forward to a private resource (e.g. RDS on 5432)

```bash
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.abc123.ap-south-1.rds.amazonaws.com"],"portNumber":["5432"],"localPortNumber":["9999"]}' \
  --region ap-south-1

psql -h localhost -p 9999 -U dbadmin
```

### Required IAM instance role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Principal": { "Service": "ec2.amazonaws.com" }
    }
  ]
}
```
Attach the managed policy `AmazonSSMManagedInstanceCore` to the role.

### Terraform: fully private SSM access via VPC endpoints

```hcl
resource "aws_vpc_endpoint" "ssm" {
  for_each          = toset(["ssm", "ssmmessages", "ec2messages"])
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.ap-south-1.${each.key}"
  vpc_endpoint_type = "Interface"
  subnet_ids        = aws_subnet.private[*].id
  security_group_ids = [aws_security_group.ssm_endpoints.id]
  private_dns_enabled = true
}
```

---

## 🔐 Security Best Practices

- **Scope `ssm:StartSession` with `ssm:ResourceTag` conditions** — never grant blanket `ssm:StartSession` on `*`; tag instances by environment and restrict who can session into `Environment=prod`.
- **Enable session logging to S3 and CloudWatch with encryption** — set this once at the Systems Manager preferences level and every session is captured automatically.
- **Require MFA for the calling identity** — combine with `aws:MultiFactorAuthPresent` in the IAM policy for prod access.

---

## 😄 Funny Things to Try

```bash
aws ssm describe-instance-information \
  --query 'InstanceInformationList[].[InstanceId,PingStatus,PlatformName]' \
  --output table

aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=staging" \
  --parameters 'commands=["uptime"]'
# Congratulations, you just fleet-SSH'd without SSH.
```

---

## ⚠️ Gotchas & Tricky Bits

- **The SSM Agent must actually be running and current** — older hardened AMIs sometimes ship an outdated agent that silently fails to register.
- **Fully private subnets need all three VPC endpoints, not just one** — missing `ec2messages` causes sessions to hang at "Starting session" with no useful error.
- **Session Manager preferences are region-specific** — logging/encryption settings don't carry across regions.
- **Pro Tip:** `aws ssm start-session` respects your local CLI profile and MFA session — check `aws sts get-caller-identity` first when troubleshooting access denied.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → Systems Manager → Session Manager`
2. **Look for:** the "Preferences" tab — S3/CloudWatch logging and KMS encryption.
3. **Key field:** `S3 bucket name` under Preferences.
4. **Common mistake here:** forgetting `AmazonSSMManagedInstanceCore` on the instance role.
5. **Confirm with CLI:**
   ```bash
   aws ssm describe-instance-information --filters "Key=InstanceIds,Values=i-0123456789abcdef0"
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| VPC Interface Endpoints | Required for SSM access from fully private subnets |
| CloudTrail | Records every StartSession/TerminateSession call |
| IAM Access Analyzer | Verify the SSM role isn't over-permissioned |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
