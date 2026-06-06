# ☁️ DynamoDB Deep Dive — Partition Keys, GSI, LSI & DAX — AWS Mastery

> **The database that scales to millions of requests per second with single-digit millisecond latency — but only if you model your data correctly. Get the key design wrong and you've built an expensive, throttled disaster.**

---

## 📖 Concept

### Partition Keys and the Hot Partition Problem

DynamoDB distributes data across partitions based on the partition key hash. Each partition handles up to 3,000 RCU (Read Capacity Units) and 1,000 WCU (Write Capacity Units). If your partition key has low cardinality — like `status` with values `active`/`inactive`, or `date` written as `2024-01-01` — most traffic goes to the same partition(s). This is the hot partition problem: throughput exceeds per-partition limits and you get `ProvisionedThroughputExceededException` even when total table capacity is ample.

Good partition key design principles: high cardinality (user IDs, UUIDs, order numbers), even distribution (avoid sequential IDs that concentrate traffic on one partition), and write distribution (for time-series, add a random suffix or shard number: `sensor#2024-01-01#shard3`).

### Global Secondary Indexes (GSI) and Local Secondary Indexes (LSI)

DynamoDB only supports efficient querying on the primary key (partition + optional sort key). For any other access pattern, you need a secondary index.

**LSI (Local Secondary Index):** Must be defined at table creation. Uses the same partition key as the main table but a different sort key. Shares throughput with the base table. Max 5 per table. Max item collection size: 10 GB per partition key value (if this limit is hit, writes fail). LSIs give you an alternate sort order within the same partition.

**GSI (Global Secondary Index):** Can be created on any attribute, at any time. Has its own partition key and optional sort key — completely separate from the base table keys. Has its own provisioned throughput (or shares in on-demand mode). Max 20 per table. GSIs use eventual consistency for reads. GSIs are the standard access pattern solution: design one GSI per alternate query access pattern.

### DAX (DynamoDB Accelerator)

DAX is a fully managed, in-memory cache for DynamoDB that's API-compatible — your existing DynamoDB SDK calls work without code changes (just change the endpoint URL to the DAX cluster endpoint). DAX provides microsecond read latency (vs millisecond for DynamoDB) and dramatically reduces RCU consumption for read-heavy workloads.

DAX has a write-through cache: writes go to DynamoDB and DAX simultaneously. Item-level cache TTL is configurable (default: 5 minutes). For strongly consistent reads, DAX bypasses cache and goes directly to DynamoDB. DAX clusters are VPC-resident — they live in your private subnets.

---

## 🏗️ Architecture Snapshot

```
DynamoDB Data Modelling — Orders Table Example
──────────────────────────────────────────────────────────────────

  Base Table:
  PK: customerId (partition)   SK: orderId#timestamp (sort)
  ┌──────────────┬──────────────────┬──────────┬────────────┐
  │ customerId   │ orderId#ts        │ status   │ total      │
  ├──────────────┼──────────────────┼──────────┼────────────┤
  │ cust-001     │ ord-A#2024-01-01  │ shipped  │ 99.99      │
  │ cust-001     │ ord-B#2024-01-15  │ pending  │ 45.00      │
  │ cust-002     │ ord-C#2024-01-02  │ shipped  │ 200.00     │
  └──────────────┴──────────────────┴──────────┴────────────┘
  Query: "All orders for cust-001" ← efficient (PK match)

  GSI-1: status-index
  PK: status             SK: timestamp
  ┌──────────────┬──────────────────┬──────────────┐
  │ status       │ timestamp         │ customerId   │
  ├──────────────┼──────────────────┼──────────────┤
  │ shipped      │ 2024-01-01       │ cust-001     │
  │ shipped      │ 2024-01-02       │ cust-002     │
  │ pending      │ 2024-01-15       │ cust-001     │
  └──────────────┴──────────────────┴──────────────┘
  Query: "All pending orders, newest first" ← efficient (GSI)

  DAX Cluster (in-memory, microsecond reads):
  App → DAX Cluster (VPC) → DynamoDB (on cache miss)
             ↑ 5min TTL cache
```

---

## 💡 Real-World Use Cases

- **E-commerce order management:** Base table keyed on `customerId` + `orderId`. GSI-1 on `status` + `createdAt` for "show all pending orders" admin view. GSI-2 on `productId` for "show all orders containing this product" for inventory. Three access patterns, one table, two GSIs.
- **Session store with DAX:** A high-traffic web app stores user sessions in DynamoDB. With DAX, session lookups drop from 3ms to 100μs. RCU consumption drops 80%. The savings on RCU pays for the DAX cluster — net cost reduction.
- **IoT time-series data:** Sensor readings keyed on `sensorId` (partition) + `timestamp` (sort). Avoid using `date` alone as partition key — add a shard suffix: `sensorId#shard1`, `sensorId#shard2` distributed via round-robin. Prevents hot partition on high-frequency sensors.

---

## 🔧 AWS CLI & Console Examples

### Create a DynamoDB Table with GSI

```bash
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=customerId,AttributeType=S \
    AttributeName=orderId,AttributeType=S \
    AttributeName=status,AttributeType=S \
    AttributeName=createdAt,AttributeType=S \
  --key-schema \
    AttributeName=customerId,KeyType=HASH \
    AttributeName=orderId,KeyType=RANGE \
  --global-secondary-indexes '[
    {
      "IndexName": "status-index",
      "KeySchema": [
        {"AttributeName": "status", "KeyType": "HASH"},
        {"AttributeName": "createdAt", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ]' \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1

# Wait for table to be active
aws dynamodb wait table-exists --table-name Orders --region ap-south-1
```

### Write and Query Items

```bash
# Put an item
aws dynamodb put-item \
  --table-name Orders \
  --item '{
    "customerId": {"S": "cust-001"},
    "orderId": {"S": "ord-A"},
    "status": {"S": "pending"},
    "createdAt": {"S": "2024-01-15T10:00:00Z"},
    "total": {"N": "99.99"},
    "items": {"L": [{"S": "item-1"}, {"S": "item-2"}]}
  }' \
  --region ap-south-1

# Query base table: all orders for a customer, newest first
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression "customerId = :cid" \
  --expression-attribute-values '{":cid": {"S": "cust-001"}}' \
  --scan-index-forward false \
  --region ap-south-1

# Query GSI: all pending orders
aws dynamodb query \
  --table-name Orders \
  --index-name status-index \
  --key-condition-expression "#s = :status" \
  --expression-attribute-names '{"#s": "status"}' \
  --expression-attribute-values '{":status": {"S": "pending"}}' \
  --region ap-south-1
```

### Enable DynamoDB Streams (for triggers and replication)

```bash
aws dynamodb update-table \
  --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES \
  --region ap-south-1

# Get the stream ARN
aws dynamodb describe-table \
  --table-name Orders \
  --query 'Table.LatestStreamArn' \
  --output text \
  --region ap-south-1
```

### Create a DAX Cluster

```bash
# Create DAX subnet group first
aws dax create-subnet-group \
  --subnet-group-name prod-dax-subnet-group \
  --subnet-ids subnet-priv-a subnet-priv-b subnet-priv-c \
  --region ap-south-1

# Create DAX cluster (3-node for HA)
aws dax create-cluster \
  --cluster-name prod-dax-cluster \
  --node-type dax.r6g.large \
  --replication-factor 3 \
  --iam-role-arn arn:aws:iam::123456789012:role/DAXRole \
  --subnet-group-name prod-dax-subnet-group \
  --security-group-ids sg-dax-sg \
  --region ap-south-1

# Get DAX endpoint for your app
aws dax describe-clusters \
  --cluster-names prod-dax-cluster \
  --query 'Clusters[0].ClusterDiscoveryEndpoint' \
  --region ap-south-1
```

### Python — Using DAX Client

```python
import amazondax
import boto3

# Regular DynamoDB client
dynamodb = boto3.resource('dynamodb', region_name='ap-south-1')

# DAX client (same API, different endpoint)
dax = amazondax.AmazonDaxClient.resource(
    endpoint_url='daxs://prod-dax-cluster.xyz.dax-clusters.ap-south-1.amazonaws.com:9111',
    region_name='ap-south-1'
)

# Use dax exactly like dynamodb
table = dax.Table('Orders')
response = table.query(
    KeyConditionExpression='customerId = :cid',
    ExpressionAttributeValues={':cid': 'cust-001'}
)
```

### Terraform — DynamoDB Table with GSI

```hcl
resource "aws_dynamodb_table" "orders" {
  name         = "Orders"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "customerId"
  range_key    = "orderId"

  attribute {
    name = "customerId"
    type = "S"
  }
  attribute {
    name = "orderId"
    type = "S"
  }
  attribute {
    name = "status"
    type = "S"
  }
  attribute {
    name = "createdAt"
    type = "S"
  }

  global_secondary_index {
    name            = "status-index"
    hash_key        = "status"
    range_key       = "createdAt"
    projection_type = "ALL"
  }

  point_in_time_recovery { enabled = true }

  server_side_encryption { enabled = true }

  tags = { Environment = "prod" }
}
```

---

## 🔐 Security Best Practices

- **Enable encryption at rest.** DynamoDB encrypts with AWS-managed keys by default (free). For compliance, use customer-managed KMS keys — adds a small per-request KMS cost but gives you key rotation control and CloudTrail visibility on key usage.
- **Use fine-grained access control with IAM conditions.** You can restrict DynamoDB access to specific items using `dynamodb:LeadingKeys` condition key: `"dynamodb:LeadingKeys": ["${aws:PrincipalTag/UserId}"]` — users can only read/write their own items. Perfect for multi-tenant Lambda + DynamoDB architectures.
- **Enable Point-in-Time Recovery (PITR).** PITR gives you continuous backups with 35-day restore window — costs 0.2 cents per GB-month. Always enable on production tables. The alternative (on-demand backup) is cheaper per backup but doesn't protect against "I accidentally deleted all orders 2 hours ago."
- **Avoid Scan operations on large tables in production.** A Scan reads every item and consumes full table RCU. One accidental full-table scan on a large table can exhaust provisioned capacity and cause cascading throttling. Use Query (always), and set up CloudWatch alarms on `SystemErrors` and `ThrottledRequests`.

---

## 😄 Funny Things to Try

```bash
# Check your table's current consumed capacity (is anyone doing a full scan?!)
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum \
  --region ap-south-1 \
  --query 'Datapoints[*].[Timestamp,Sum]' \
  --output table
# Spike at 3:47 AM? Someone ran a full Scan. Find them.

# Estimate item size before you regret it
aws dynamodb describe-table \
  --table-name Orders \
  --region ap-south-1 \
  --query '[Table.TableSizeBytes, Table.ItemCount]' \
  --output table
# DynamoDB charges per item per operation. An average item size of 400KB
# means each read costs 50 RCU (1 RCU = 4KB strongly consistent).
# Math is humbling.
```

---

## ⚠️ Gotchas & Tricky Bits

- **GSIs have eventual consistency only.** Reads from GSIs are always eventually consistent — you cannot request strongly consistent reads from a GSI. If your app needs read-your-writes after an update, read from the base table, not the GSI.
- **GSI write capacity must be provisioned separately** (in provisioned mode). If your GSI runs out of WCU, writes to the base table succeed but GSI updates are dropped — silently! Your GSI becomes stale and queries return wrong data. In on-demand mode, this doesn't apply.
- **Item collection limit on LSIs.** All items with the same partition key value across the base table and all LSIs combined cannot exceed 10 GB. Hit this and writes fail with `ItemCollectionSizeLimitExceededException`. Monitor `AccountMaxTableLevelThrottling` and `ReturnItemCollectionMetrics`.
- **DynamoDB `batchWrite` is not atomic.** Unlike a SQL transaction, `BatchWriteItem` can partially succeed. Use `TransactWriteItems` (up to 25 items) for atomic multi-item writes — critical for anything financial.
- **Pro Tip:** For time-series data where you need to delete old items, use TTL (Time To Live) with a Unix timestamp attribute. DynamoDB automatically deletes expired items within 48 hours — no Lambda trigger, no cost, no operations overhead. Set it once and forget.

---

## 📸 Console Walkthrough

1. **Navigate to:** `AWS Console → DynamoDB → Tables → Create table`
2. **Key design:** Partition key is the most critical decision — choose it carefully. Sort key is optional but enables range queries
3. **Capacity mode:** Choose "On-demand" for new/unpredictable workloads; switch to "Provisioned" once you know your steady-state throughput (30-60% cheaper)
4. **Add GSI:** Scroll to "Indexes" section → "Create global index" → choose new partition key + optional sort key
5. **Common mistake here:** Setting GSI projection to "Keys only" and then trying to query non-key attributes — you'll get partial responses. Use "All" for simplicity; use "Include" for specific attributes to save costs on large tables
6. **Enable PITR:** `Table → Backups → Enable Point-in-time recovery`
7. **Check table metrics:**
   ```bash
   aws dynamodb describe-table \
     --table-name Orders \
     --region ap-south-1 \
     --query 'Table.[TableStatus,ItemCount,TableSizeBytes,BillingModeSummary]'
   ```

---

## 🔗 Related Services

| Service | Why it connects |
|---------|-----------------|
| DAX (DynamoDB Accelerator) | In-memory cache for read-heavy workloads — microsecond latency, same API |
| DynamoDB Streams + Lambda | Change data capture for event-driven architectures, cross-region replication triggers |
| AWS AppSync | GraphQL API that can use DynamoDB as a data source with automatic resolver generation |
| Amazon Kinesis Data Streams | Alternative to DynamoDB Streams for higher-throughput CDC — supports up to 24h retention vs 24h for Streams |
| AWS Backup | Centralized backup management for DynamoDB across accounts and regions — complements PITR |

---

*Part of the [AWS Mastery](https://github.com/naveenramasamy11/aws-mastery) series.*
