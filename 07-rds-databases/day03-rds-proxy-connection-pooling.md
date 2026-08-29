# ☁️ RDS Proxy & Connection Pooling — AWS Mastery

> **A Lambda function that opens a new database connection per invocation is a denial-of-service attack you wrote yourself.**

---

## 📖 Concept

Every serverless-meets-relational-database migration hits the same wall: Lambda scales by running many concurrent short-lived execution environments, and each one, left to its own devices, opens its own connection to RDS or Aurora. At a few hundred concurrent invocations, you exhaust `max_connections` on the database and start seeing connection errors that have nothing to do with query load — the database is fine, it's just out of connection slots. RDS Proxy sits between your application (Lambda, ECS, EC2) and the database, maintaining a warm pool of actual database connections and multiplexing many client connections onto a much smaller number of real backend connections.

The part that matters architecturally: RDS Proxy is fully managed and highly available across AZs by default, and it also gives you IAM authentication as an option instead of embedding database passwords in Lambda environment variables — pulling credentials from Secrets Manager and rotating them without any application code change or connection-dropping event. On Aurora specifically, RDS Proxy also improves failover behavior: application connections stay open against the Proxy while it transparently re-establishes the backend connection to the new writer, which for many workloads cuts failover-related downtime from tens of seconds to a couple of seconds.

On a migration engagement moving a monolith's Lambda-based API layer onto Aurora, we introduced RDS Proxy specifically because Lambda concurrency during traffic spikes was periodically exhausting `max_connections` on a fairly generously-sized Aurora instance — the database wasn't underpowered, it was just being asked to hold open far more connections than it could concurrently execute queries for.

---

## 🏗️ Architecture Snapshot

```
        Many Lambda invocations (100s concurrent)
                    │  │  │  │  │
                    ▼  ▼  ▼  ▼  ▼
        ┌─────────────────────────────────────┐
        │           RDS Proxy              │
        │  - Connection pooling/multiplexing│
        │  - IAM auth via Secrets Manager   │
        │  - Multi-AZ, fully managed        │
        └────────────────┬───────────────────┘
                          │  (small pool of real connections)
                          ▼
        ┌─────────────────────────────────────┐
        │      Aurora / RDS Cluster         │
        │      Writer + Read Replicas       │
        └─────────────────────────────────────┘
```

---

## 💡 Real-World Use Cases

- **Serverless API layer with bursty traffic:** Lambda functions behind API Gateway connect through RDS Proxy instead of directly to Aurora, avoiding connection exhaustion during traffic spikes.
- **Faster failover for latency-sensitive apps:** RDS Proxy shortens Aurora failover impact by holding client connections open and re-pointing them to the new writer transparently.
- **Removing database passwords from Lambda code entirely:** IAM authentication through RDS Proxy means credentials never appear in environment variables or code — Secrets Manager rotation happens invisibly to the application.

---

## 🔧 AWS CLI & Console Examples

### Create an RDS Proxy

```bash
aws rds create-db-proxy \
  --db-proxy-name my-app-proxy \
  --engine-family MYSQL \
  --auth file://proxy-auth.json \
  --role-arn arn:aws:iam::123456789012:role/rds-proxy-role \
  --vpc-subnet-ids subnet-abc123 subnet-def456 \
  --region ap-south-1
```

### Register the target (the actual DB cluster)

```bash
aws rds register-db-proxy-targets \
  --db-proxy-name my-app-proxy \
  --db-cluster-identifiers my-aurora-cluster \
  --region ap-south-1
```

### Lambda connecting via IAM auth (Python)

```python
import boto3, pymysql

rds = boto3.client('rds', region_name='ap-south-1')
token = rds.generate_db_auth_token(
    DBHostname='my-app-proxy.proxy-abc123.ap-south-1.rds.amazonaws.com',
    Port=3306, DBUsername='app_user'
)
conn = pymysql.connect(
    host='my-app-proxy.proxy-abc123.ap-south-1.rds.amazonaws.com',
    user='app_user', password=token, port=3306,
    ssl={'ca': 'rds-ca-bundle.pem'}
)
```

---

## 🔐 Security Best Practices

- **Always enable IAM authentication through the Proxy for Lambda workloads** — eliminates long-lived database passwords from function code and environment variables entirely.
- **Restrict `rds-db:connect` IAM permission to specific proxy/user combinations** — don't grant it broadly; scope by resource ARN to the specific proxy and database user the function actually needs.
- **Enable TLS enforcement on the Proxy** (`RequireTLS: true`) — without it, connections between clients and the Proxy can be unencrypted even though Proxy-to-database traffic is protected.

---

## 😄 Funny Things to Try

```bash
# Watch how few real connections the database actually needs
aws rds describe-db-proxy-target-groups --db-proxy-name my-app-proxy \
  --query 'TargetGroups[].ConnectionPoolConfig'

# Compare connection counts: direct vs through Proxy under load
mysql -h my-aurora-cluster.cluster-abc123.ap-south-1.rds.amazonaws.com -e "SHOW PROCESSLIST" | wc -l
mysql -h my-app-proxy.proxy-abc123.ap-south-1.rds.amazonaws.com -e "SHOW PROCESSLIST" | wc -l
```

---

## ⚠️ Gotchas & Tricky Bits

- **RDS Proxy doesn't help with query performance, only connection management** — it's not a caching layer or a query accelerator; slow queries are still slow.
- **Pinning can silently defeat pooling** — session-level state (temp tables, `SET` statements, multi-statement transactions) can pin a client connection to one backend connection, reducing multiplexing efficiency; check `DatabaseConnectionsCurrentlySessionPinned` in CloudWatch if pooling seems less effective than expected.
- **RDS Proxy adds a small amount of latency per query** — usually single-digit milliseconds, but worth measuring for extremely latency-sensitive workloads before assuming it's free.
- **Pro Tip:** RDS Proxy requires the database to already exist and be reachable from the same VPC — plan subnet/security-group access explicitly; a common setup mistake is forgetting the Proxy's own security group needs egress to the database's security group.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → RDS → Proxies → Create proxy`
2. **Look for:** the "Require Transport Layer Security" toggle — enable this for production every time.
3. **Key field:** `IAM authentication` — set to Required if the goal is fully removing embedded passwords.
4. **Common mistake here:** forgetting to register the actual DB cluster as a target after creating the Proxy — a Proxy with no registered targets simply can't route traffic anywhere.
5. **Confirm with CLI:**
   ```bash
   aws rds describe-db-proxy-targets --db-proxy-name my-app-proxy
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| Secrets Manager | Stores and rotates the database credentials RDS Proxy authenticates with |
| Lambda | The most common high-concurrency client that benefits from Proxy pooling |
| CloudWatch | Exposes `DatabaseConnections` and pinning metrics to validate pooling effectiveness |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
