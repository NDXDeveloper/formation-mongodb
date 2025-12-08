🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.10 Atlas Data Lake

## Introduction

**Atlas Data Lake** transforme votre stockage cloud (S3, Azure Blob, GCS) en source de données interrogeable via le langage MongoDB. Plus besoin d'ETL complexe ou de duplication : interrogez directement vos données archivées, logs historiques, ou datasets massifs avec la syntaxe MongoDB familière. C'est la solution idéale pour l'analytics sur données froides, l'archivage économique, et les requêtes fédérées (MongoDB + S3).

### 🎯 Objectifs de cette Section

- Comprendre l'architecture Data Lake
- Configurer un Data Lake avec S3/Azure/GCS
- Maîtriser les requêtes fédérées (MongoDB + S3)
- Implémenter une stratégie de data tiering (hot/cold)
- Optimiser les coûts avec cold storage
- Utiliser Data Lake pour analytics et compliance

---

## 🏗️ Architecture Atlas Data Lake

### Vue d'Ensemble

```
┌────────────────────────────────────────────────────────────────────────┐
│                   ATLAS DATA LAKE ARCHITECTURE                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   APPLICATION                                                          │
│   ┌──────────────────────────────────────────────────────────────────┐ │
│   │  MongoDB Query (Standard Syntax):                                │ │
│   │  db.orders.find({ createdAt: { $gte: ISODate("2023-01-01") } })  │ │
│   │                                                                  │ │
│   │  Single connection string for both:                              │ │
│   │  • Hot data (MongoDB Atlas cluster)                              │ │
│   │  • Cold data (S3/Azure/GCS)                                      │ │
│   └────────────────────────────────┬─────────────────────────────────┘ │
│                                    │                                   │
│                                    ▼                                   │
│   ATLAS DATA LAKE SERVICE (MongoDB-managed)                            │
│   ┌──────────────────────────────────────────────────────────────────┐ │
│   │  Query Router & Optimizer                                        │ │
│   │  • Parses MongoDB queries                                        │ │
│   │  • Routes to appropriate storage                                 │ │
│   │  • Optimizes execution plan                                      │ │
│   │  • Merges results from multiple sources                          │ │
│   └───────────────┬────────────────────────────────┬─────────────────┘ │
│                   │                                │                   │
│         ┌─────────┴────────┐              ┌────────┴─────────┐         │
│         ▼                  ▼              ▼                  ▼         │
│   ┌──────────┐      ┌──────────┐    ┌──────────┐      ┌──────────┐     │
│   │ MONGODB  │      │ MONGODB  │    │   AWS    │      │  AZURE   │     │
│   │ CLUSTER  │      │ CLUSTER  │    │    S3    │      │   BLOB   │     │
│   │  (Hot)   │      │  (Warm)  │    │  (Cold)  │      │  (Cold)  │     │
│   └──────────┘      └──────────┘    └──────────┘      └──────────┘     │
│   Recent data       Last 90 days     Archives         Compliance       │
│   < 30 days         Real-time        Historical       Long-term        │
│   High IOPS         queries          analytics        retention        │
│                                                                        │
│   SUPPORTED FORMATS IN S3/AZURE/GCS:                                   │
│   • JSON (plain text or gzip)                                          │
│   • BSON (MongoDB native)                                              │
│   • Parquet (columnar, optimal for analytics)                          │
│   • CSV / TSV                                                          │
│   • Avro                                                               │
│                                                                        │
│   KEY BENEFITS:                                                        │
│   ✅ Query S3 with MongoDB syntax (no Athena/Spark needed)             │
│   ✅ Federated queries (join MongoDB + S3 data)                        │
│   ✅ Cost optimization: S3 ~$0.023/GB vs MongoDB ~$0.25/GB             │
│   ✅ No data duplication                                               │
│   ✅ Automatic schema inference                                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Data Lake vs Alternatives

```
┌────────────────────────────────────────────────────────────────────────┐
│         ATLAS DATA LAKE vs ALTERNATIVES (Analytics on S3)              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  FEATURE           DATA LAKE    ATHENA      SPARK        REDSHIFT      │
│  ───────────────────────────────────────────────────────────────────── │
│  Query Language    MongoDB      SQL         SQL/Scala    SQL           │
│  Setup Time        Minutes      Minutes     Hours        Hours         │
│  Infrastructure    Zero         Zero        Cluster      Cluster       │
│  MongoDB Native    ✅ Yes       ❌ No       ⚠️ Via       ❌ No         │
│                                             Connector                  │
│  Federated Query   ✅ Yes       ❌ No       ✅ Yes       ⚠️ Limited    │
│  (MongoDB+S3)                                                          │
│                                                                        │
│  JSON Support      ✅ Native    ⚠️ Limited  ✅ Good      ⚠️ Limited    │
│  Nested Docs       ✅ Native    ❌ No       ⚠️ Complex   ❌ No         │
│  Aggregations      ✅ Full      ⚠️ Basic    ✅ Full      ✅ Full       │
│                                                                        │
│  Cost Model        Compute      Scanned     Cluster      Cluster       │
│                    hours        data        hours        hours         │
│  Typical Cost      $2-5/TB      $5/TB       $15-30/TB    $25+/TB       │
│                                                                        │
│  Best For          MongoDB      Ad-hoc SQL  Complex      High-perf     │
│                    users        queries     ETL          BI/DW         │
│                                                                        │
│  RECOMMENDATION:                                                       │
│  • Data Lake: If using MongoDB, want unified query experience          │
│  • Athena: SQL team, simple ad-hoc queries                             │
│  • Spark: Complex transformations, ML pipelines                        │
│  • Redshift: Traditional data warehouse, heavy BI workloads            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Configuration

### 1. Création Data Lake

```bash
# Via Atlas UI:
# 1. Atlas Dashboard → Data Federation (left menu)
# 2. Create Data Lake
# 3. Configure:
#    - Data Lake name
#    - Cloud provider (AWS/Azure/GCP)
#    - Region (same as cluster for best perf)
#    - S3 bucket or Azure container

# Via Atlas CLI
atlas dataLakes create \
  --projectId <PROJECT_ID> \
  --name production-datalake \
  --cloudProviderConfig \
    provider=AWS \
    roleId=<IAM_ROLE_ARN> \
    testS3Bucket=my-bucket

# Via Terraform
resource "mongodbatlas_data_lake" "production" {
  project_id = var.atlas_project_id
  name       = "production-datalake"

  cloud_provider_config {
    aws {
      role_id        = aws_iam_role.atlas_data_lake.arn
      test_s3_bucket = aws_s3_bucket.datalake.id
    }
  }
}
```

### 2. Configuration IAM (AWS)

```hcl
# Terraform: IAM Role for Data Lake to access S3
resource "aws_iam_role" "atlas_data_lake" {
  name = "atlas-data-lake-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.atlas_aws_account_id}:root"
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "sts:ExternalId" = var.atlas_external_id
          }
        }
      }
    ]
  })
}

# IAM Policy: Read access to S3 bucket
resource "aws_iam_role_policy" "atlas_data_lake_s3" {
  name = "atlas-data-lake-s3-policy"
  role = aws_iam_role.atlas_data_lake.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket",
          "s3:GetBucketLocation"
        ]
        Resource = [
          aws_s3_bucket.datalake.arn,
          "${aws_s3_bucket.datalake.arn}/*"
        ]
      }
    ]
  })
}

# S3 Bucket for Data Lake
resource "aws_s3_bucket" "datalake" {
  bucket = "company-mongodb-datalake"

  versioning {
    enabled = true
  }

  lifecycle_rule {
    enabled = true

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}
```

### 3. Définition du Virtual Database

```javascript
// Virtual database configuration
// Maps S3 paths to MongoDB collections

// Option 1: Via Atlas UI
// Data Federation → Databases → Add Database
// Configure paths and mappings

// Option 2: Via API
{
  "databases": [
    {
      "name": "analytics",
      "collections": [
        {
          "name": "orders",
          "dataSources": [
            {
              "storeName": "s3Store",
              "path": "/orders/{year}-{month}-{day}/*.json",
              "defaultFormat": "json"
            }
          ]
        },
        {
          "name": "logs",
          "dataSources": [
            {
              "storeName": "s3Store",
              "path": "/logs/{year}/{month}/{day}/*.parquet",
              "defaultFormat": "parquet"
            }
          ]
        }
      ]
    }
  ],
  "stores": [
    {
      "name": "s3Store",
      "provider": "s3",
      "region": "us-east-1",
      "bucket": "company-mongodb-datalake"
    }
  ]
}
```

### 4. Connection String

```javascript
// Data Lake connection string (separate from cluster)
const dataLakeUri = "mongodb://user:password@data-lake-0.mongodb.net/analytics";

// OR unified connection string (federated)
const federatedUri = "mongodb+srv://user:password@cluster0.mongodb.net/";

// Connect
const client = new MongoClient(dataLakeUri);
await client.connect();

// Query S3 data with MongoDB syntax
const db = client.db('analytics');
const orders = await db.collection('orders').find({
  createdAt: { $gte: new Date('2023-01-01') }
}).toArray();

console.log(`Found ${orders.length} orders from 2023`);
```

---

## 🔍 Requêtes Data Lake

### Requêtes Basiques

```javascript
// Standard MongoDB queries work on S3 data
const db = client.db('analytics');

// 1. Find documents
const recentOrders = await db.collection('orders').find({
  createdAt: { $gte: new Date('2024-01-01') },
  status: 'completed'
}).toArray();

// 2. Aggregation pipeline
const salesByMonth = await db.collection('orders').aggregate([
  {
    $match: {
      createdAt: { $gte: new Date('2024-01-01') }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      totalSales: { $sum: "$amount" },
      orderCount: { $count: {} }
    }
  },
  {
    $sort: { "_id.year": 1, "_id.month": 1 }
  }
]).toArray();

// 3. Complex aggregations
const topProducts = await db.collection('orders').aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.productId",
      totalQuantity: { $sum: "$items.quantity" },
      totalRevenue: { $sum: { $multiply: ["$items.quantity", "$items.price"] } }
    }
  },
  { $sort: { totalRevenue: -1 } },
  { $limit: 10 }
]).toArray();
```

### Federated Queries (MongoDB + S3)

```javascript
// Query that joins hot data (MongoDB) with cold data (S3)

// Scenario: Recent orders in MongoDB, historical in S3
// Query last 30 days + historical data together

const allOrders = await db.collection('orders').aggregate([
  {
    $unionWith: {
      coll: "orders_archive",  // S3-backed collection
      pipeline: [
        {
          $match: {
            createdAt: {
              $gte: new Date('2023-01-01'),
              $lt: new Date('2024-01-01')
            }
          }
        }
      ]
    }
  },
  {
    $match: {
      customerId: "12345"
    }
  },
  {
    $sort: { createdAt: -1 }
  }
]).toArray();

// This query:
// 1. Searches recent orders in MongoDB cluster
// 2. Searches historical orders in S3
// 3. Merges results
// 4. Filters by customerId
// 5. Sorts by date
```

### Partitioning pour Performance

```
┌───────────────────────────────────────────────────────────────────────┐
│                    S3 PARTITIONING STRATEGY                           │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  PARTITIONING BY DATE (Most Common)                                   │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  s3://bucket/orders/                                             │ │
│  │    ├── year=2024/                                                │ │
│  │    │   ├── month=01/                                             │ │
│  │    │   │   ├── day=01/                                           │ │
│  │    │   │   │   └── data.parquet                                  │ │
│  │    │   │   ├── day=02/                                           │ │
│  │    │   │   │   └── data.parquet                                  │ │
│  │    │   │   └── ...                                               │ │
│  │    │   ├── month=02/                                             │ │
│  │    │   │   └── ...                                               │ │
│  │                                                                  │ │
│  │  Virtual collection mapping:                                     │ │
│  │  path: "/orders/year={year}/month={month}/day={day}/*.parquet"   │ │
│  │                                                                  │ │
│  │  Query with partition pruning:                                   │ │
│  │  db.orders.find({                                                │ │
│  │    createdAt: {                                                  │ │
│  │      $gte: ISODate("2024-01-15"),                                │ │
│  │      $lt: ISODate("2024-01-20")                                  │ │
│  │    }                                                             │ │
│  │  })                                                              │ │
│  │                                                                  │ │
│  │  Only scans:                                                     │ │
│  │  • year=2024/month=01/day=15/                                    │ │
│  │  • year=2024/month=01/day=16/                                    │ │
│  │  • ...                                                           │ │
│  │  • year=2024/month=01/day=19/                                    │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  BENEFITS:                                                            │
│  ✅ Partition pruning: Only scan relevant directories                 │
│  ✅ Faster queries: 10-100x faster with proper partitioning           │
│  ✅ Lower cost: Less data scanned = lower compute cost                │
│  ✅ Parallel processing: Multiple partitions read in parallel         │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🔥 Data Tiering Strategies

### Hot/Warm/Cold Architecture

```
┌───────────────────────────────────────────────────────────────────────┐
│                  DATA TIERING ARCHITECTURE                            │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  HOT TIER (MongoDB Atlas Cluster)                                     │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Data Age:      0-30 days                                         │ │
│  │ Access:        Very frequent (100s/sec)                          │ │
│  │ Latency:       < 10ms                                            │ │
│  │ Size:          50 GB                                             │ │
│  │ Cost:          $0.25/GB-month = $12.50/month                     │ │
│  │ Use Cases:     • Real-time queries                               │ │
│  │                • Transactional operations                        │ │
│  │                • User-facing features                            │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                ↓                                      │
│                         TTL Index (30 days)                           │
│                         Auto-archive to S3                            │
│                                ↓                                      │
│  WARM TIER (S3 Standard)                                              │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Data Age:      30-90 days                                        │ │
│  │ Access:        Occasional (10s/day)                              │ │
│  │ Latency:       100-500ms                                         │ │
│  │ Size:          200 GB                                            │ │
│  │ Cost:          $0.023/GB-month = $4.60/month                     │ │
│  │ Use Cases:     • Recent analytics                                │ │
│  │                • Reporting                                       │ │
│  │                • Audit queries                                   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                ↓                                      │
│                      S3 Lifecycle (90 days)                           │
│                      Transition to Glacier                            │
│                                ↓                                      │
│  COLD TIER (S3 Glacier)                                               │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Data Age:      > 90 days                                         │ │
│  │ Access:        Rare (few/month)                                  │ │
│  │ Latency:       Minutes-hours (retrieval)                         │ │
│  │ Size:          2 TB                                              │ │
│  │ Cost:          $0.004/GB-month = $8.20/month                     │ │
│  │ Use Cases:     • Compliance                                      │ │
│  │                • Historical analysis                             │ │
│  │                • Long-term archival                              │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  TOTAL COST ANALYSIS (2.25 TB total):                                 │
│  ──────────────────────────────────────────────────────────────────── │
│  Scenario A: All in MongoDB                                           │
│  2.25 TB × $0.25/GB = $576/month                                      │
│                                                                       │
│  Scenario B: Tiered (Hot/Warm/Cold)                                   │
│  Hot:   50 GB × $0.25  = $12.50/month                                 │
│  Warm: 200 GB × $0.023 = $4.60/month                                  │
│  Cold:   2 TB × $0.004 = $8.20/month                                  │
│  Total: $25.30/month                                                  │
│                                                                       │
│  SAVINGS: $550.70/month (96% reduction!)                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Automated Archiving

```javascript
// Automated archiving strategy using TTL + Change Streams

// 1. TTL Index on MongoDB (auto-delete after 30 days)
db.orders.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 2592000 }  // 30 days
);

// 2. Change Stream to capture deletions and archive to S3
const changeStream = db.orders.watch([
  { $match: { operationType: 'delete' } }
]);

changeStream.on('change', async (change) => {
  // Archive deleted document to S3
  const doc = change.fullDocument;

  const year = doc.createdAt.getFullYear();
  const month = String(doc.createdAt.getMonth() + 1).padStart(2, '0');
  const day = String(doc.createdAt.getDate()).padStart(2, '0');

  const s3Key = `orders/year=${year}/month=${month}/day=${day}/${doc._id}.json`;

  await s3Client.putObject({
    Bucket: 'company-mongodb-datalake',
    Key: s3Key,
    Body: JSON.stringify(doc),
    ContentType: 'application/json'
  });

  console.log(`Archived order ${doc._id} to S3`);
});

// 3. Or use Atlas Scheduled Trigger for batch archiving
// Atlas UI → Triggers → Scheduled → Run daily at 2 AM
exports = async function() {
  const collection = context.services.get("mongodb-atlas")
    .db("production").collection("orders");

  const thirtyDaysAgo = new Date();
  thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

  const oldOrders = await collection.find({
    createdAt: { $lt: thirtyDaysAgo }
  }).toArray();

  // Archive to S3
  for (const order of oldOrders) {
    await archiveToS3(order);
  }

  // Delete from MongoDB
  await collection.deleteMany({
    createdAt: { $lt: thirtyDaysAgo }
  });

  return `Archived ${oldOrders.length} orders`;
};
```

---

## 📊 Cas d'Usage

### 1. Historical Analytics

```javascript
// Use case: Analyze sales trends over 3 years (mostly in S3)

const salesTrend = await db.collection('orders').aggregate([
  {
    $match: {
      createdAt: {
        $gte: new Date('2022-01-01'),
        $lt: new Date('2025-01-01')
      }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      totalSales: { $sum: "$amount" },
      orderCount: { $count: {} },
      avgOrderValue: { $avg: "$amount" }
    }
  },
  {
    $sort: { "_id.year": 1, "_id.month": 1 }
  }
]).toArray();

// This query:
// - Scans 3 years of data
// - Most data in S3 (cost-effective)
// - Returns monthly aggregates
// - No need to move data to MongoDB
```

### 2. Customer 360 View

```javascript
// Combine recent orders (MongoDB) with historical (S3)

async function getCustomer360(customerId) {
  // Recent orders from MongoDB
  const recentOrders = await mongoDb.collection('orders').find({
    customerId: customerId,
    createdAt: { $gte: thirtyDaysAgo }
  }).toArray();

  // Historical orders from Data Lake (S3)
  const historicalOrders = await dataLakeDb.collection('orders_archive').find({
    customerId: customerId,
    createdAt: { $lt: thirtyDaysAgo }
  }).toArray();

  // Combine
  const allOrders = [...recentOrders, ...historicalOrders];

  // Calculate metrics
  const totalSpent = allOrders.reduce((sum, order) => sum + order.amount, 0);
  const orderCount = allOrders.length;
  const avgOrderValue = totalSpent / orderCount;

  return {
    customerId,
    totalSpent,
    orderCount,
    avgOrderValue,
    firstOrderDate: allOrders[allOrders.length - 1].createdAt,
    lastOrderDate: allOrders[0].createdAt
  };
}
```

### 3. Compliance & Audit

```javascript
// Use case: Query 7 years of data for compliance audit

const auditQuery = await db.collection('transactions').aggregate([
  {
    $match: {
      userId: "suspicious-user-123",
      createdAt: {
        $gte: new Date('2018-01-01'),  // 7 years ago
        $lt: new Date('2025-01-01')
      }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" }
      },
      transactionCount: { $count: {} },
      totalAmount: { $sum: "$amount" },
      avgAmount: { $avg: "$amount" }
    }
  }
]).toArray();

// Benefits:
// - 7 years of data queried seamlessly
// - Most data in Glacier (pennies per GB)
// - Meets compliance requirements
// - No need to restore from backups
```

### 4. Log Analytics

```javascript
// Use case: Analyze application logs stored in S3 as JSON

// S3 structure:
// s3://logs/app-logs/year=2024/month=12/day=08/hour=14/*.json

const errorAnalysis = await db.collection('app_logs').aggregate([
  {
    $match: {
      timestamp: {
        $gte: new Date('2024-12-01'),
        $lt: new Date('2024-12-08')
      },
      level: 'ERROR'
    }
  },
  {
    $group: {
      _id: {
        errorType: "$error.type",
        errorMessage: "$error.message"
      },
      count: { $count: {} },
      firstOccurrence: { $min: "$timestamp" },
      lastOccurrence: { $max: "$timestamp" }
    }
  },
  {
    $sort: { count: -1 }
  },
  {
    $limit: 20
  }
]).toArray();

// Analyze millions of log entries
// Stored cheaply in S3
// Query with familiar MongoDB syntax
// No need for ELK stack
```

---

## 🚀 Performance et Optimisation

### File Formats

```
┌────────────────────────────────────────────────────────────────────────┐
│                      FILE FORMAT COMPARISON                            │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  FORMAT      SIZE    READ PERF    WRITE PERF    USE CASE               │
│  ───────────────────────────────────────────────────────────────────── │
│  JSON        ⭐      ⭐⭐         ⭐⭐⭐         • Human-readable
│              Large   Slow        Fast          • Compatibility         │
│                                                • Simple use cases      │
│                                                                        │
│  BSON        ⭐⭐     ⭐⭐⭐        ⭐⭐⭐         • MongoDB native
│              Medium  Fast        Fast          • Binary efficiency     │
│                                                • Nested docs           │
│                                                                        │
│  Parquet     ⭐⭐⭐   ⭐⭐⭐⭐⭐     ⭐            • Analytics (BEST)
│              Small   Very fast   Slow          • Columnar storage      │
│              (10x)   (100x)                    • Compression           │
│                                                • Large datasets        │
│                                                                        │
│  CSV          ⭐⭐    ⭐⭐         ⭐⭐⭐         • Flat data
│              Small   Medium      Fast          • Excel compatibility   │
│                                                • Simple structure      │
│                                                                        │
│  RECOMMENDATION FOR DATA LAKE:                                         │
│  ✅ Parquet for analytics workloads (10-100x better performance)       │
│  ✅ BSON for MongoDB-native data                                       │
│  ✅ JSON for human-readable archives                                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Query Optimization Tips

```javascript
// ❌ BAD: Full collection scan
db.orders.find({
  amount: { $gt: 1000 }
}).toArray();
// Scans all files in S3

// ✅ GOOD: Partition pruning with date filter
db.orders.find({
  createdAt: {
    $gte: new Date('2024-12-01'),
    $lt: new Date('2024-12-08')
  },
  amount: { $gt: 1000 }
}).toArray();
// Only scans relevant date partitions

// ❌ BAD: Projection after full document fetch
db.orders.find({}).toArray().then(docs =>
  docs.map(d => ({ _id: d._id, amount: d.amount }))
);

// ✅ GOOD: Projection in query (especially for Parquet)
db.orders.find(
  {},
  { projection: { _id: 1, amount: 1 } }
).toArray();
// Columnar format reads only needed columns

// ✅ BEST: Combine partition pruning + projection + filter
db.orders.find(
  {
    createdAt: { $gte: new Date('2024-12-01') },
    status: 'completed',
    amount: { $gt: 1000 }
  },
  {
    projection: { _id: 1, customerId: 1, amount: 1 }
  }
).toArray();
```

### Costs Monitoring

```javascript
// Data Lake costs are based on:
// 1. Compute time (processing)
// 2. Data scanned (S3 reads)

// Example cost calculation:
const costs = {
  // Compute: $5 per 1M document-reads
  compute: {
    documentsRead: 10_000_000,  // 10M docs
    costPerMillion: 5,
    total: (10_000_000 / 1_000_000) * 5  // $50
  },

  // Data transfer: $0.09/GB (if crossing regions)
  dataTransfer: {
    gigabytesTransferred: 100,  // 100GB
    costPerGB: 0.09,
    total: 100 * 0.09  // $9
  },

  // S3 storage: $0.023/GB-month
  storage: {
    gigabytesStored: 1000,  // 1TB
    costPerGB: 0.023,
    total: 1000 * 0.023  // $23
  }
};

const totalMonthlyCost =
  costs.compute.total +
  costs.dataTransfer.total +
  costs.storage.total;

console.log(`Estimated monthly cost: $${totalMonthlyCost}`);
// Output: Estimated monthly cost: $82
```

---

## 📋 Best Practices

### Data Lake Checklist

```
┌───────────────────────────────────────────────────────────────────────┐
│                   DATA LAKE BEST PRACTICES                            │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  DATA ORGANIZATION                                                    │
│  ☐ Partition by date (year/month/day) for time-series data            │
│  ☐ Use Parquet format for analytics workloads                         │
│  ☐ Consistent naming conventions (lowercase, hyphens)                 │
│  ☐ Compress files (gzip for JSON, snappy for Parquet)                 │
│  ☐ Optimal file sizes: 128MB - 1GB per file                           │
│                                                                       │
│  QUERY OPTIMIZATION                                                   │
│  ☐ Always include partition keys in queries                           │
│  ☐ Use projection to limit columns (especially Parquet)               │
│  ☐ Limit result sets with $limit                                      │
│  ☐ Test queries on small date ranges first                            │
│  ☐ Monitor query performance in Atlas UI                              │
│                                                                       │
│  DATA LIFECYCLE                                                       │
│  ☐ Implement TTL indexes on hot data                                  │
│  ☐ Automate archiving to S3                                           │
│  ☐ Use S3 lifecycle policies (Standard → Glacier)                     │
│  ☐ Test restore procedures for Glacier data                           │
│  ☐ Document retention policies                                        │
│                                                                       │
│  COST MANAGEMENT                                                      │
│  ☐ Monitor Data Lake compute costs                                    │
│  ☐ Optimize queries to minimize data scanned                          │
│  ☐ Use same region for cluster and S3 (avoid transfer)                │
│  ☐ Archive rarely-accessed data to Glacier                            │
│  ☐ Review and delete unused virtual collections                       │
│                                                                       │
│  SECURITY                                                             │
│  ☐ Use IAM roles (not access keys) for S3 access                      │
│  ☐ Encrypt data at rest (S3 encryption)                               │
│  ☐ Encrypt data in transit (TLS)                                      │
│  ☐ Implement least privilege access                                   │
│  ☐ Enable S3 versioning for data protection                           │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Common Pitfalls

```
❌ DATA LAKE ANTI-PATTERNS:

1. No Partitioning
   • Storing all data in single directory
   • Every query scans entire dataset
   → Solution: Partition by date or other key dimension

2. Too Many Small Files
   • Millions of < 1MB files
   • High overhead, slow queries
   → Solution: Combine into 128MB-1GB files

3. Wrong File Format
   • Using JSON for analytics (10-100x slower than Parquet)
   → Solution: Convert to Parquet for analytics

4. No Projection
   • Fetching full documents when only need few fields
   • Wastes bandwidth and compute
   → Solution: Always use projection

5. Cross-Region Data Transfer
   • Cluster in us-east-1, S3 in eu-west-1
   • High transfer costs ($0.09/GB)
   → Solution: Same region for cluster and S3

6. No Lifecycle Management
   • Keeping all data in S3 Standard
   • 5x more expensive than Glacier
   → Solution: Lifecycle policy to Glacier after 90 days
```

---

## 🏁 Résumé

### Points Clés

1. **Architecture**
   - Query S3/Azure/GCS with MongoDB syntax
   - Federated queries (MongoDB + cloud storage)
   - Near real-time sync
   - No data duplication

2. **File Formats**
   - Parquet: Best for analytics (10-100x faster)
   - BSON: MongoDB native
   - JSON: Human-readable, compatibility
   - CSV: Flat data, Excel

3. **Data Tiering**
   - Hot (0-30d): MongoDB ($0.25/GB)
   - Warm (30-90d): S3 Standard ($0.023/GB)
   - Cold (>90d): S3 Glacier ($0.004/GB)
   - 96% cost savings possible

4. **Performance**
   - Partition by date (year/month/day)
   - Use Parquet for analytics
   - Always include partition keys in queries
   - Use projection for columnar formats

5. **Use Cases**
   - Historical analytics
   - Compliance (7+ years retention)
   - Log analytics
   - Customer 360 views

### Configuration Minimale Production

```javascript
// 1. S3 bucket with partitioning
// s3://bucket/collection/year=2024/month=12/day=08/*.parquet

// 2. Virtual database config
{
  "databases": [{
    "name": "analytics",
    "collections": [{
      "name": "orders",
      "dataSources": [{
        "storeName": "s3Store",
        "path": "/orders/year={year}/month={month}/day={day}/*.parquet",
        "defaultFormat": "parquet"
      }]
    }]
  }]
}

// 3. Optimized query pattern
db.orders.find(
  {
    createdAt: {
      $gte: new Date('2024-12-01'),  // Partition pruning
      $lt: new Date('2024-12-08')
    },
    status: 'completed'
  },
  {
    projection: { _id: 1, amount: 1, customerId: 1 }  // Column pruning
  }
).limit(1000)  // Limit results
```

### Cost Example

```
Scenario: 10TB total data, 1 year retention

Option A: All in MongoDB
10,000 GB × $0.25/GB = $2,500/month

Option B: Data Lake (Hot/Warm/Cold)
Hot (50GB, 30d):    50 × $0.25 = $12.50
Warm (500GB, 60d): 500 × $0.023 = $11.50
Cold (9.5TB, 9m): 9,500 × $0.004 = $38.00
Total: $62/month

Savings: $2,438/month (97%)
```

---


⏭️ [Atlas Charts (visualisation)](/14-mongodb-atlas/11-atlas-charts.md)
