🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.6 MongoDB Connector for BI

## Introduction

MongoDB Connector for BI (anciennement BI Connector) est un pont SQL qui permet aux outils de Business Intelligence traditionnels (Tableau, Power BI, Looker, QlikView, etc.) de se connecter à MongoDB comme s'il s'agissait d'une base de données relationnelle. Il traduit les requêtes SQL en opérations MongoDB, permettant ainsi aux analystes métier d'exploiter les données MongoDB sans apprendre le langage d'agrégation MongoDB.

Cette section explore l'architecture, les stratégies de modélisation analytics, les optimisations performance et les patterns d'intégration pour des environnements de production à grande échelle.

---

## 🎯 Architecture et Fonctionnement

### Vue d'ensemble

**Principe fondamental**
```
[BI Tool] ──SQL──▶ [BI Connector] ──MongoDB Query──▶ [MongoDB]
    ↓                    ↓                                ↓
 Tableau            Translate SQL              mongod instance
 Power BI           to MongoDB                 (primary/secondary)
 Looker             aggregation
```

Le BI Connector agit comme un **proxy MySQL** :
- Les outils BI se connectent via protocole MySQL (port 3307 par défaut)
- Le connector analyse les requêtes SQL
- Traduit en opérations MongoDB (aggregation pipelines)
- Retourne les résultats au format tabulaire

### Architecture de Production

```
┌───────────────────────────────────────────────────────────────┐
│                  Production BI Architecture                   │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │              BI Tools Layer                             │  │
│  │  • Tableau Server (50 users)                            │  │
│  │  • Power BI Service (100 users)                         │  │
│  │  • Looker (20 users)                                    │  │
│  └────────────┬────────────────────────────────────────────┘  │
│               │ SQL queries (MySQL protocol)                  │
│               ↓                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │    BI Connector Cluster (High Availability)             │  │
│  │                                                         │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │  │
│  │  │  Node 1  │  │  Node 2  │  │  Node 3  │               │  │
│  │  │  Active  │  │  Active  │  │  Active  │               │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘               │  │
│  │       │             │             │                     │  │
│  │       └─────────────┴─────────────┘                     │  │
│  │                     │                                   │  │
│  │           [Load Balancer: HAProxy]                      │  │
│  └─────────────────────┬───────────────────────────────────┘  │
│                        │ MongoDB queries                      │
│                        ↓                                      │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │         MongoDB Atlas Analytics Cluster                 │  │
│  │                                                         │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │  │
│  │  │ Primary  │  │Secondary │  │Secondary │               │  │
│  │  │(no read) │  │ Analytics│  │ Analytics│               │  │
│  │  └──────────┘  └────┬─────┘  └────┬─────┘               │  │
│  │                     │             │                     │  │
│  │      [Read from secondaries only]                       │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │            Metadata & Caching Layer                     │  │
│  │  • Schema sampling cache                                │  │
│  │  • Query result cache (Redis)                           │  │
│  │  • Pre-aggregated tables (materialized views)           │  │
│  └─────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### Composants clés

**1. BI Connector (mongosqld)**
- Processus standalone qui écoute sur port MySQL (3307)
- Traduit SQL → MongoDB aggregation
- Gère connexions concurrentes
- Cache schema metadata

**2. Schema Discovery**
- Analyse documents MongoDB pour inférer schema
- Génère DRDL (Document Relational Definition Language)
- Mapping flexible/embedded → tables relationnelles

**3. Query Translation Engine**
- Parse SQL queries
- Optimise et traduit en aggregation pipeline
- Gère JOINs via $lookup
- Supporte subqueries

---

## 📦 Installation et Configuration

### Installation (Linux)

```bash
# Download
wget https://downloads.mongodb.com/bi-connector/mongodb-bi-linux-x86_64-ubuntu2004-v2.14.9.tgz

# Extract
tar -xzf mongodb-bi-linux-x86_64-ubuntu2004-v2.14.9.tgz

# Move to system path
sudo mv mongodb-bi-linux-x86_64-ubuntu2004-v2.14.9 /opt/mongodb-bi

# Create symlinks
sudo ln -s /opt/mongodb-bi/bin/mongosqld /usr/local/bin/
sudo ln -s /opt/mongodb-bi/bin/mongodrdl /usr/local/bin/
```

### Configuration de base

**Fichier de configuration (mongosqld.conf)**

```yaml
# /etc/mongosqld.conf

# Network settings
net:
  bindIp: 0.0.0.0  # Listen on all interfaces (adjust for security)
  port: 3307
  ssl:
    mode: requireSSL
    PEMKeyFile: /path/to/ssl/server.pem
    CAFile: /path/to/ssl/ca.pem

# MongoDB connection
mongodb:
  net:
    uri: "mongodb://analytics-user:password@mongodb-cluster.example.com:27017/production?ssl=true&replicaSet=rs0"
    auth:
      username: analytics_user
      password: ${MONGO_PASSWORD}
      source: admin

  # Connection pool
  maxPoolSize: 100
  minPoolSize: 10

# Schema
schema:
  # Path to DRDL schema file (generated or custom)
  path: /var/schema/schema.drdl

  # Auto-refresh schema periodically
  refresh: true
  refreshIntervalSecs: 3600  # 1 hour

# Logging
systemLog:
  verbosity: 1  # 0=quiet, 1=normal, 2=verbose, 3=debug
  path: /var/log/mongosqld/mongosqld.log
  logAppend: true
  logRotate: reopen

# Performance
runtime:
  memory:
    maxPerStage: 134217728  # 128 MB per aggregation stage
  maxVarcharLength: 1024

# Security
security:
  enabled: true
  defaultMechanism: SCRAM-SHA-256
  defaultSource: admin
```

### Démarrage du service

```bash
# Démarrage manuel
mongosqld --config /etc/mongosqld.conf

# Systemd service
sudo cat > /etc/systemd/system/mongosqld.service << 'EOF'
[Unit]
Description=MongoDB BI Connector
After=network.target

[Service]
Type=simple
User=mongosqld
Group=mongosqld
Environment="MONGO_PASSWORD=SecurePassword123"
ExecStart=/usr/local/bin/mongosqld --config /etc/mongosqld.conf
Restart=on-failure
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable mongosqld
sudo systemctl start mongosqld

# Check status
sudo systemctl status mongosqld
```

---

## 🗂️ Schema Discovery et DRDL

### Génération automatique du schema

**Utilisation de mongodrdl**

```bash
# Générer DRDL depuis MongoDB
mongodrdl \
  --host mongodb-cluster.example.com:27017 \
  --ssl \
  --username analytics_user \
  --password SecurePassword123 \
  --authenticationDatabase admin \
  --db production \
  --collection orders \
  --out /var/schema/schema.drdl \
  --sampleSize 10000
```

**DRDL généré (exemple)**

```yaml
# schema.drdl
schema:
- db: production
  tables:

  # Collection: orders
  - table: orders
    collection: orders
    pipeline: []
    columns:
    - Name: _id
      MongoType: bson.ObjectId
      SqlName: _id
      SqlType: varchar

    - Name: order_number
      MongoType: string
      SqlName: order_number
      SqlType: varchar

    - Name: customer_id
      MongoType: int
      SqlName: customer_id
      SqlType: int

    - Name: order_date
      MongoType: bson.Date
      SqlName: order_date
      SqlType: timestamp

    - Name: status
      MongoType: string
      SqlName: status
      SqlType: varchar

    - Name: total
      MongoType: decimal
      SqlName: total
      SqlType: decimal(10,2)

    # Embedded customer (denormalized)
    - Name: customer.name
      MongoType: string
      SqlName: customer_name
      SqlType: varchar

    - Name: customer.email
      MongoType: string
      SqlName: customer_email
      SqlType: varchar

    # Array items (flattened)
    - Name: items
      MongoType: array
      SqlName: items
      SqlType: varchar
      # Array mapping handled specially

  # Virtual table for order items (flattening)
  - table: order_items
    collection: orders
    pipeline:
    - $unwind: $items
    columns:
    - Name: _id
      MongoType: bson.ObjectId
      SqlName: order_id
      SqlType: varchar

    - Name: items.product_id
      MongoType: string
      SqlName: product_id
      SqlType: varchar

    - Name: items.product_name
      MongoType: string
      SqlName: product_name
      SqlType: varchar

    - Name: items.quantity
      MongoType: int
      SqlName: quantity
      SqlType: int

    - Name: items.unit_price
      MongoType: decimal
      SqlName: unit_price
      SqlType: decimal(10,2)

    - Name: items.subtotal
      MongoType: decimal
      SqlName: subtotal
      SqlType: decimal(10,2)
```

### Schema personnalisé (optimisé pour BI)

**DRDL custom avec pre-aggregations**

```yaml
# schema-optimized.drdl
schema:
- db: production
  tables:

  # Table principale orders (dénormalisée pour performance)
  - table: orders
    collection: orders
    pipeline: []
    columns:
    - Name: _id
      MongoType: bson.ObjectId
      SqlName: order_id
      SqlType: varchar
    - Name: order_number
      SqlName: order_number
      SqlType: varchar
    - Name: order_date
      SqlName: order_date
      SqlType: timestamp
    - Name: customer.name
      SqlName: customer_name
      SqlType: varchar
    - Name: customer.email
      SqlName: customer_email
      SqlType: varchar
    - Name: customer.tier
      SqlName: customer_tier
      SqlType: varchar
    - Name: total
      SqlName: total
      SqlType: decimal(10,2)
    - Name: status
      SqlName: status
      SqlType: varchar

    # Computed columns
    - Name: computed.year
      MongoType: int
      SqlName: order_year
      SqlType: int
      # Utilise $project dans pipeline

    - Name: computed.month
      MongoType: int
      SqlName: order_month
      SqlType: int

    - Name: computed.quarter
      MongoType: int
      SqlName: order_quarter
      SqlType: int

  # Table items (flattened pour analyses détaillées)
  - table: order_items
    collection: orders
    pipeline:
    - $unwind: $items
    - $project:
        order_id: $_id
        order_number: 1
        order_date: 1
        product_id: $items.product_id
        product_name: $items.product_name
        product_category: $items.category
        quantity: $items.quantity
        unit_price: $items.unit_price
        subtotal: $items.subtotal
        discount: $items.discount
    columns:
    - Name: order_id
      SqlName: order_id
      SqlType: varchar
    - Name: order_number
      SqlName: order_number
      SqlType: varchar
    - Name: order_date
      SqlName: order_date
      SqlType: timestamp
    - Name: product_id
      SqlName: product_id
      SqlType: varchar
    - Name: product_name
      SqlName: product_name
      SqlType: varchar
    - Name: product_category
      SqlName: product_category
      SqlType: varchar
    - Name: quantity
      SqlName: quantity
      SqlType: int
    - Name: unit_price
      SqlName: unit_price
      SqlType: decimal(10,2)
    - Name: subtotal
      SqlName: subtotal
      SqlType: decimal(10,2)
    - Name: discount
      SqlName: discount
      SqlType: decimal(10,2)

  # Table aggregated (pré-calculée pour dashboards)
  - table: orders_daily_summary
    collection: orders
    pipeline:
    - $match:
        order_date:
          $gte: { $date: "2024-01-01T00:00:00Z" }
    - $project:
        date: { $dateToString: { format: "%Y-%m-%d", date: "$order_date" } }
        total: 1
        status: 1
        customer_tier: $customer.tier
    - $group:
        _id:
          date: $date
          status: $status
          tier: $customer_tier
        order_count: { $sum: 1 }
        revenue: { $sum: $total }
        avg_order_value: { $avg: $total }
    - $project:
        _id: 0
        date: $_id.date
        status: $_id.status
        customer_tier: $_id.tier
        order_count: 1
        revenue: 1
        avg_order_value: 1
    columns:
    - Name: date
      SqlName: date
      SqlType: date
    - Name: status
      SqlName: status
      SqlType: varchar
    - Name: customer_tier
      SqlName: customer_tier
      SqlType: varchar
    - Name: order_count
      SqlName: order_count
      SqlType: int
    - Name: revenue
      SqlName: revenue
      SqlType: decimal(15,2)
    - Name: avg_order_value
      SqlName: avg_order_value
      SqlType: decimal(10,2)
```

### Rechargement du schema

```bash
# Signaler mongosqld pour recharger schema
kill -SIGHUP $(pgrep mongosqld)

# Ou redémarrage
sudo systemctl restart mongosqld
```

---

## 🔌 Intégration avec Outils BI

### 1. Tableau

**Configuration connexion**

```
Server: bi-connector.example.com
Port: 3307
Database: production
Username: tableau_user
Password: ••••••••

Advanced Options:
  - Initial SQL: SET time_zone = '+00:00'
  - Use SSL: Yes
  - Require SSL: Yes
```

**Optimisations Tableau**

```sql
-- Utiliser extracts (TDE) plutôt que live connections pour grandes volumétries
-- Dans Tableau Desktop:
-- Data → [Connection] → Extract Data
-- Schedule: Daily refresh at 2 AM

-- Custom SQL pour pré-filtrer
SELECT
  order_id,
  order_date,
  customer_name,
  customer_tier,
  total,
  status
FROM orders
WHERE order_date >= DATE_SUB(CURDATE(), INTERVAL 90 DAY)
  AND status IN ('completed', 'shipped')
```

**Calculated Fields (Tableau-side)**

```
// Éviter calculations côté BI, privilégier MongoDB
// Mais si nécessaire:

// Year
YEAR([order_date])

// Quarter
"Q" + STR(DATEPART('quarter', [order_date]))

// Revenue per Customer Tier (LOD expression)
{ FIXED [customer_tier] : SUM([total]) }
```

### 2. Power BI

**Configuration connexion (MySQL connector)**

```
Data Source Settings:
  Server: bi-connector.example.com:3307
  Database: production

Advanced Options:
  SQL Statement:
    SELECT * FROM orders WHERE order_date >= DATE_SUB(NOW(), INTERVAL 365 DAY)

  Connection timeout: 30
  Command timeout: 600
```

**Power Query M customizations**

```m
let
    Source = MySQL.Database("bi-connector.example.com:3307", "production"),
    orders = Source{[Schema="production",Item="orders"]}[Data],

    // Filtrer côté serveur (push-down)
    filtered = Table.SelectRows(orders, each [order_date] >= Date.AddDays(DateTime.LocalNow(), -365)),

    // Typage
    typed = Table.TransformColumnTypes(filtered, {
        {"order_date", type datetime},
        {"total", type number},
        {"order_year", Int64.Type},
        {"order_month", Int64.Type}
    }),

    // Calculated columns (éviter si possible)
    withCalcs = Table.AddColumn(typed, "revenue_tier", each
        if [total] > 1000 then "High"
        else if [total] > 500 then "Medium"
        else "Low"
    )
in
    withCalcs
```

**DirectQuery vs Import**

```
Import Mode (recommandé pour < 1 GB):
✓ Performance queries rapides
✓ Toutes fonctionnalités DAX
✗ Data pas real-time
✗ Refresh nécessaire

DirectQuery Mode:
✓ Data real-time
✓ Pas de refresh
✗ Performance variable
✗ Fonctionnalités DAX limitées
```

### 3. Looker

**Configuration connexion (looker.yml)**

```yaml
# looker.yml
connections:
  mongodb_production:
    host: bi-connector.example.com
    port: 3307
    database: production
    username: looker_user
    password: ${LOOKER_DB_PASSWORD}
    dialect: mysql

    # Connection pool
    max_connections: 20
    timeout: 60

    # SSL
    ssl: true
    ssl_mode: require
```

**LookML model**

```lookml
# models/orders.model.lkml
connection: "mongodb_production"

include: "/views/*.view"

explore: orders {
  join: order_items {
    type: left_outer
    sql_on: ${orders.order_id} = ${order_items.order_id} ;;
    relationship: one_to_many
  }

  join: customers {
    type: left_outer
    sql_on: ${orders.customer_email} = ${customers.email} ;;
    relationship: many_to_one
  }
}
```

**LookML views**

```lookml
# views/orders.view.lkml
view: orders {
  sql_table_name: production.orders ;;

  dimension: order_id {
    type: string
    sql: ${TABLE}.order_id ;;
    primary_key: yes
  }

  dimension_group: order {
    type: time
    timeframes: [date, week, month, quarter, year]
    sql: ${TABLE}.order_date ;;
  }

  dimension: customer_name {
    type: string
    sql: ${TABLE}.customer_name ;;
  }

  dimension: customer_tier {
    type: string
    sql: ${TABLE}.customer_tier ;;
  }

  dimension: status {
    type: string
    sql: ${TABLE}.status ;;
  }

  dimension: total {
    type: number
    sql: ${TABLE}.total ;;
    value_format_name: usd
  }

  # Measures
  measure: count {
    type: count
    drill_fields: [order_id, order_date, customer_name, total]
  }

  measure: total_revenue {
    type: sum
    sql: ${total} ;;
    value_format_name: usd
  }

  measure: average_order_value {
    type: average
    sql: ${total} ;;
    value_format_name: usd
  }

  # Filtered measures
  measure: completed_orders {
    type: count
    filters: [status: "completed"]
  }

  measure: completed_revenue {
    type: sum
    sql: ${total} ;;
    filters: [status: "completed"]
  }
}
```

---

## ⚡ Optimisations Performance

### 1. Indexes MongoDB

**Indexes critiques pour BI queries**

```javascript
// orders collection
db.orders.createIndex({ order_date: -1 });
db.orders.createIndex({ status: 1, order_date: -1 });
db.orders.createIndex({ "customer.tier": 1, order_date: -1 });
db.orders.createIndex({ order_date: -1, total: -1 });

// Compound index pour drill-downs fréquents
db.orders.createIndex({
  order_date: -1,
  status: 1,
  "customer.tier": 1
});

// Text index pour recherches
db.orders.createIndex({
  order_number: "text",
  "customer.name": "text"
});

// Index sur champs embedded souvent filtrés
db.orders.createIndex({ "items.product_id": 1 });
db.orders.createIndex({ "items.category": 1, order_date: -1 });
```

### 2. Pre-aggregated Views (Materialized Views)

**Architecture avec aggregation pipeline**

```javascript
// Créer une collection pré-agrégée (mise à jour quotidienne)
db.orders.aggregate([
  {
    $match: {
      order_date: { $gte: ISODate("2024-01-01") }
    }
  },
  {
    $project: {
      date: { $dateToString: { format: "%Y-%m-%d", date: "$order_date" } },
      year: { $year: "$order_date" },
      month: { $month: "$order_date" },
      week: { $week: "$order_date" },
      status: 1,
      customer_tier: "$customer.tier",
      total: 1,
      item_count: { $size: "$items" }
    }
  },
  {
    $group: {
      _id: {
        date: "$date",
        year: "$year",
        month: "$month",
        week: "$week",
        status: "$status",
        tier: "$customer_tier"
      },
      order_count: { $sum: 1 },
      revenue: { $sum: "$total" },
      avg_order_value: { $avg: "$total" },
      total_items: { $sum: "$item_count" },
      avg_items_per_order: { $avg: "$item_count" }
    }
  },
  {
    $project: {
      _id: 0,
      date: "$_id.date",
      year: "$_id.year",
      month: "$_id.month",
      week: "$_id.week",
      status: "$_id.status",
      customer_tier: "$_id.tier",
      order_count: 1,
      revenue: 1,
      avg_order_value: 1,
      total_items: 1,
      avg_items_per_order: 1
    }
  },
  {
    $out: "orders_daily_aggregated"
  }
]);

// Index sur la collection agrégée
db.orders_daily_aggregated.createIndex({ date: -1 });
db.orders_daily_aggregated.createIndex({ year: -1, month: -1 });
db.orders_daily_aggregated.createIndex({ date: -1, customer_tier: 1 });
```

**Automatisation refresh (cron job)**

```bash
#!/bin/bash
# /usr/local/bin/refresh_aggregated_views.sh

MONGO_URI="mongodb://admin:password@mongodb-cluster.example.com:27017/production?authSource=admin"

# Refresh daily aggregations
mongosh "$MONGO_URI" --quiet --eval '
db.orders.aggregate([
  /* aggregation pipeline here */
  { $out: "orders_daily_aggregated" }
]);

db.products.aggregate([
  /* product aggregations */
  { $out: "products_summary" }
]);

print("Aggregated views refreshed successfully");
'

# Cron: Daily at 2 AM
# 0 2 * * * /usr/local/bin/refresh_aggregated_views.sh >> /var/log/aggregated_refresh.log 2>&1
```

### 3. Query Result Caching (Redis)

**Architecture avec cache**

```typescript
// bi-cache-layer.ts
import { createClient } from 'redis';
import { MongoClient } from 'mongodb';
import crypto from 'crypto';

interface CacheConfig {
  ttl: number;  // seconds
  keyPrefix: string;
}

class BIQueryCache {
  private redis: ReturnType<typeof createClient>;
  private mongo: MongoClient;
  private config: CacheConfig;

  constructor(config: CacheConfig) {
    this.config = config;
    this.redis = createClient({ url: 'redis://localhost:6379' });
    this.mongo = new MongoClient(process.env.MONGODB_URI);
  }

  async connect(): Promise<void> {
    await this.redis.connect();
    await this.mongo.connect();
  }

  async executeQuery(sql: string): Promise<any[]> {
    // Generate cache key from SQL
    const cacheKey = this.generateCacheKey(sql);

    // Check cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      console.log('Cache hit:', cacheKey);
      return JSON.parse(cached);
    }

    console.log('Cache miss:', cacheKey);

    // Execute query (via BI Connector or direct MongoDB)
    const results = await this.executeMongoQuery(sql);

    // Store in cache
    await this.redis.setEx(
      cacheKey,
      this.config.ttl,
      JSON.stringify(results)
    );

    return results;
  }

  private generateCacheKey(sql: string): string {
    const hash = crypto
      .createHash('sha256')
      .update(sql)
      .digest('hex');

    return `${this.config.keyPrefix}:${hash}`;
  }

  private async executeMongoQuery(sql: string): Promise<any[]> {
    // Simplified - in reality, use BI Connector translation
    // or direct aggregation pipeline

    // Example: Parse simple SELECT
    const match = sql.match(/SELECT .* FROM (\w+) WHERE (.*)/i);
    if (!match) {
      throw new Error('Unsupported query');
    }

    const [, collection, whereClause] = match;

    // Execute on MongoDB
    const db = this.mongo.db('production');
    const results = await db
      .collection(collection)
      .find(this.parseWhereClause(whereClause))
      .toArray();

    return results;
  }

  private parseWhereClause(whereClause: string): any {
    // Simplified parser - production would be more robust
    return {};
  }

  async invalidateCache(pattern: string): Promise<void> {
    const keys = await this.redis.keys(`${this.config.keyPrefix}:${pattern}*`);
    if (keys.length > 0) {
      await this.redis.del(keys);
      console.log(`Invalidated ${keys.length} cache keys`);
    }
  }
}

// Usage
const cache = new BIQueryCache({
  ttl: 3600,  // 1 hour
  keyPrefix: 'bi_query'
});

await cache.connect();

const results = await cache.executeQuery(`
  SELECT
    DATE(order_date) as date,
    COUNT(*) as order_count,
    SUM(total) as revenue
  FROM orders
  WHERE order_date >= DATE_SUB(NOW(), INTERVAL 7 DAY)
  GROUP BY DATE(order_date)
`);
```

### 4. Read Preferences et Analytics Nodes

**Configuration MongoDB pour analytics**

```javascript
// Configuration Replica Set pour analytics
// mongodb.conf sur secondary dédiés analytics

replication:
  replSetName: rs0

# Tag ce node comme analytics
replSetName: rs0
tags:
  region: us-east
  usage: analytics
```

**BI Connector read preference**

```yaml
# mongosqld.conf
mongodb:
  net:
    uri: "mongodb://analytics-user:password@mongodb-cluster.example.com:27017/production?replicaSet=rs0&readPreference=secondary&readPreferenceTags=usage:analytics"
```

**Avantages**
- ✅ Isolation des requêtes analytics (pas d'impact sur primary)
- ✅ Performance reads optimisée (secondary dédiés)
- ✅ Scaling horizontal (ajout secondaries analytics)

---

## 📊 Scénario Réel : Retail Chain Analytics (6 mois)

**Contexte**
- Chaîne de magasins retail, 500 magasins
- MongoDB : 2 To (transactions, inventaire, clients)
- Utilisateurs BI : 80 analystes métier (Tableau) + 20 managers (Power BI)
- Objectif : Remplacer data warehouse Oracle par MongoDB + BI Connector

### Architecture implémentée

```
┌──────────────────────────────────────────────────────────┐
│              Production Architecture                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  BI Users (100 users)                               │ │
│  │  • Tableau Server (80 analystes)                    │ │
│  │  • Power BI Service (20 managers)                   │ │
│  └────────────┬────────────────────────────────────────┘ │
│               │                                          │
│               ↓                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │  HAProxy Load Balancer                              │ │
│  └────────────┬────────────────────────────────────────┘ │
│               │                                          │
│       ┌───────┴───────┬───────────┐                      │
│       ↓               ↓           ↓                      │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐                 │
│  │   BI    │   │   BI    │   │   BI    │                 │
│  │Connector│   │Connector│   │Connector│                 │
│  │  Node 1 │   │  Node 2 │   │  Node 3 │                 │
│  └────┬────┘   └────┬────┘   └────┬────┘                 │
│       │             │             │                      │
│       └─────────────┴─────────────┘                      │
│                     │                                    │
│                     ↓                                    │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Redis Cluster (Query Cache)                     │    │
│  │  • 3 masters + 3 replicas                        │    │
│  │  • 1 hour TTL                                    │    │
│  │  • Eviction: LRU                                 │    │
│  └──────────────────────────────────────────────────┘    │
│                     │                                    │
│                     ↓                                    │
│  ┌──────────────────────────────────────────────────┐    │
│  │  MongoDB Atlas M100 (3 regions)                  │    │
│  │                                                  │    │
│  │  Primary (us-east-1)                             │    │
│  │  Secondary Analytics 1 (us-east-1)               │    │
│  │  Secondary Analytics 2 (us-east-1)               │    │
│  │  Secondary (us-west-2, disaster recovery)        │    │
│  │                                                  │    │
│  │  Aggregated Collections:                         │    │
│  │  • transactions_daily (1 year)                   │    │
│  │  • inventory_snapshot (90 days)                  │    │
│  │  • customer_360 (materialized view)              │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Modélisation données

**Collection transactions (source)**

```javascript
{
  _id: ObjectId("..."),
  transaction_id: "TXN-2024-00123456",
  timestamp: ISODate("2024-01-15T14:32:10Z"),

  store: {
    store_id: "STR-001",
    name: "Manhattan Store",
    region: "Northeast",
    city: "New York"
  },

  customer: {
    customer_id: "CUST-789",
    tier: "gold",
    age_group: "35-44",
    gender: "F"
  },

  items: [
    {
      product_id: "PROD-123",
      name: "Laptop",
      category: "Electronics",
      subcategory: "Computers",
      brand: "Apple",
      quantity: 1,
      unit_price: 1299.99,
      discount: 100.00,
      subtotal: 1199.99
    },
    {
      product_id: "PROD-456",
      name: "Mouse",
      category: "Electronics",
      subcategory: "Accessories",
      brand: "Logitech",
      quantity: 2,
      unit_price: 29.99,
      discount: 0,
      subtotal: 59.98
    }
  ],

  payment: {
    method: "credit_card",
    amount: 1259.97,
    currency: "USD"
  },

  computed: {
    total_items: 3,
    total_discount: 100.00,
    total_revenue: 1259.97
  }
}
```

**Collection aggregated (pour BI)**

```javascript
// transactions_daily
{
  _id: {
    date: "2024-01-15",
    store_id: "STR-001",
    category: "Electronics"
  },

  date: ISODate("2024-01-15T00:00:00Z"),
  year: 2024,
  month: 1,
  week: 3,
  day_of_week: 1,  // Monday

  store: {
    store_id: "STR-001",
    name: "Manhattan Store",
    region: "Northeast",
    city: "New York"
  },

  category: "Electronics",

  metrics: {
    transaction_count: 234,
    total_revenue: 45678.90,
    total_items_sold: 567,
    unique_customers: 198,
    avg_transaction_value: 195.21,
    avg_items_per_transaction: 2.42,
    total_discount: 3456.78,
    discount_rate: 0.0757  // 7.57%
  },

  customer_breakdown: {
    by_tier: {
      gold: { count: 89, revenue: 23456.78 },
      silver: { count: 67, revenue: 14567.89 },
      bronze: { count: 42, revenue: 7654.23 }
    },
    by_age_group: {
      "18-24": { count: 23, revenue: 3456.78 },
      "25-34": { count: 67, revenue: 16789.01 },
      "35-44": { count: 78, revenue: 19012.34 },
      "45-54": { count: 45, revenue: 8901.23 }
    }
  }
}
```

### DRDL Schema optimisé

```yaml
# schema.drdl
schema:
- db: retail
  tables:

  # Table principale (agrégée, pas source brute)
  - table: daily_sales
    collection: transactions_daily
    pipeline: []
    columns:
    - Name: date
      SqlName: date
      SqlType: date
    - Name: year
      SqlName: year
      SqlType: int
    - Name: month
      SqlName: month
      SqlType: int
    - Name: week
      SqlName: week
      SqlType: int
    - Name: day_of_week
      SqlName: day_of_week
      SqlType: int
    - Name: store.store_id
      SqlName: store_id
      SqlType: varchar
    - Name: store.name
      SqlName: store_name
      SqlType: varchar
    - Name: store.region
      SqlName: region
      SqlType: varchar
    - Name: store.city
      SqlName: city
      SqlType: varchar
    - Name: category
      SqlName: category
      SqlType: varchar
    - Name: metrics.transaction_count
      SqlName: transaction_count
      SqlType: int
    - Name: metrics.total_revenue
      SqlName: total_revenue
      SqlType: decimal(15,2)
    - Name: metrics.total_items_sold
      SqlName: total_items_sold
      SqlType: int
    - Name: metrics.unique_customers
      SqlName: unique_customers
      SqlType: int
    - Name: metrics.avg_transaction_value
      SqlName: avg_transaction_value
      SqlType: decimal(10,2)
    - Name: metrics.avg_items_per_transaction
      SqlName: avg_items_per_transaction
      SqlType: decimal(5,2)
    - Name: metrics.total_discount
      SqlName: total_discount
      SqlType: decimal(10,2)
    - Name: metrics.discount_rate
      SqlName: discount_rate
      SqlType: decimal(5,4)

  # Table détaillée pour drill-down (avec limite temporelle)
  - table: transactions_detail
    collection: transactions
    pipeline:
    - $match:
        timestamp:
          $gte: { $date: "2024-01-01T00:00:00Z" }
    columns:
    - Name: transaction_id
      SqlName: transaction_id
      SqlType: varchar
    - Name: timestamp
      SqlName: transaction_time
      SqlType: timestamp
    - Name: store.store_id
      SqlName: store_id
      SqlType: varchar
    - Name: store.name
      SqlName: store_name
      SqlType: varchar
    - Name: customer.customer_id
      SqlName: customer_id
      SqlType: varchar
    - Name: customer.tier
      SqlName: customer_tier
      SqlType: varchar
    - Name: payment.amount
      SqlName: total_amount
      SqlType: decimal(10,2)
    - Name: computed.total_items
      SqlName: total_items
      SqlType: int
```

### Timeline et résultats

**Mois 1 : Setup infrastructure**
- Déploiement BI Connector cluster (3 nodes)
- Configuration HAProxy
- Setup Redis cache
- Tests initiaux

**Mois 2-3 : Migration Tableau**
- Formation analystes (2 jours)
- Migration 50 dashboards prioritaires
- Tests performance
- Ajustements schema DRDL

**Mois 4 : Migration Power BI**
- Migration 15 reports managers
- Optimisation queries
- Configuration scheduled refresh

**Mois 5 : Optimisations**
- Création collections agrégées
- Tuning indexes
- Activation query cache
- Performance monitoring

**Mois 6 : Stabilisation**
- Decommissioning Oracle data warehouse
- Documentation complète
- Formation équipe support

### Résultats quantifiés

**Performance**
- Dashboard load time : 45s → 8s (Oracle → MongoDB)
- Query latency P95 : 12s → 2s
- Concurrent users supportés : 30 → 100
- Cache hit rate : 65% (Redis)

**Coûts**
- Infrastructure : -40% (vs Oracle warehouse)
- License BI : Aucune (Tableau/Power BI déjà existants)
- Maintenance : -60% (automation)

**Adoption**
- 80 analystes formés et actifs
- 150 dashboards migrés
- Satisfaction utilisateurs : 4.2/5 → 4.7/5

### Challenges rencontrés

**Challenge 1 : Performance queries avec JOINs**
- **Symptôme** : Queries Tableau avec multiples JOINs très lentes (30s+)
- **Cause** : $lookup sur grandes collections
- **Solution** : Dénormalisation données + pre-aggregation

**Challenge 2 : Schema drift**
- **Symptôme** : Nouveaux champs apparaissent, cassent dashboards
- **Cause** : MongoDB schema flexible, DRDL pas mis à jour
- **Solution** :
  - Validation schema (JSON Schema)
  - Refresh DRDL automatique quotidien
  - Alerting sur nouveaux champs

**Challenge 3 : Timeout queries lourdes**
- **Symptôme** : Certaines queries dépassent 5 min, timeout BI tool
- **Cause** : Aggregations complexes sans indexes
- **Solution** :
  - Indexes composés optimisés
  - Pre-aggregation collections
  - Incremental refresh (delta uniquement)

**Challenge 4 : Concurrence élevée**
- **Symptôme** : Dégradation performance aux heures de pointe (50+ users)
- **Cause** : BI Connector single node saturé
- **Solution** :
  - Cluster BI Connector (3 nodes + load balancer)
  - Connection pooling optimisé
  - Query result caching (Redis)

---

## 🎯 Bonnes Pratiques

### 1. Modélisation pour BI

**Principes**
- ✅ **Dénormaliser** : Préférer embedded documents aux JOINs
- ✅ **Pre-aggregate** : Collections agrégées pour dashboards
- ✅ **Time partitioning** : Limiter scope temporel (90 jours courants)
- ✅ **Computed fields** : Calculs dans MongoDB, pas BI tool

**Anti-patterns**
- ❌ Over-normalisation (nombreux JOINs)
- ❌ Arrays volumineux (>1000 éléments)
- ❌ Calculations lourdes côté BI tool
- ❌ Full table scans sur collections massives

### 2. Index Strategy

```javascript
// Indexes pour queries BI typiques

// Time-series queries
db.transactions.createIndex({ timestamp: -1 });
db.transactions.createIndex({ date: -1, store_id: 1 });

// Drill-down paths
db.transactions.createIndex({
  date: -1,
  "store.region": 1,
  "store.city": 1
});

// Filters fréquents
db.transactions.createIndex({ "customer.tier": 1, date: -1 });
db.transactions.createIndex({ category: 1, date: -1 });

// Text search (si nécessaire)
db.transactions.createIndex({
  transaction_id: "text",
  "customer.name": "text"
});
```

### 3. Monitoring

**Métriques clés**

```javascript
// Monitoring BI Connector performance
{
  "bi_connector": {
    "active_connections": 45,
    "queries_per_second": 12.5,
    "avg_query_duration_ms": 850,
    "p95_query_duration_ms": 2300,
    "cache_hit_rate": 0.65,
    "error_rate": 0.002
  },

  "mongodb": {
    "reads_per_second": 450,
    "avg_read_latency_ms": 25,
    "working_set_size_gb": 45,
    "cache_usage_percent": 78,
    "disk_io_percent": 23
  },

  "redis_cache": {
    "hit_rate": 0.65,
    "miss_rate": 0.35,
    "evictions_per_minute": 12,
    "memory_usage_percent": 68,
    "keys_count": 125000
  }
}
```

### 4. Security

```yaml
# Security best practices

1. Dedicated BI user avec permissions limitées:
   - READ-ONLY sur collections nécessaires
   - Pas d'accès write
   - Pas d'accès collections sensibles (users, passwords)

2. SSL/TLS obligatoire:
   - BI Connector ↔ BI Tools : SSL
   - BI Connector ↔ MongoDB : SSL/TLS

3. Network isolation:
   - BI Connector dans subnet privé
   - Accès uniquement via VPN ou bastion

4. Auditing:
   - Log toutes requêtes SQL
   - Alert sur queries anormales (full scans, > 1M docs)

5. Rate limiting:
   - Max queries per user per minute
   - Max concurrent connections per user
```

---

## 📚 Checklist BI Connector

**Installation**
- [ ] BI Connector installé et configuré
- [ ] SSL/TLS activé
- [ ] Connexion MongoDB validée
- [ ] Schema DRDL généré
- [ ] Service systemd configuré

**Performance**
- [ ] Indexes MongoDB optimisés
- [ ] Collections pré-agrégées créées
- [ ] Query cache (Redis) configuré
- [ ] Read preference = secondary (analytics nodes)
- [ ] Connection pooling optimisé

**BI Tools**
- [ ] Connexion Tableau/Power BI validée
- [ ] Dashboards prioritaires migrés
- [ ] Extracts configurés (si applicable)
- [ ] Scheduled refresh configuré
- [ ] Formation utilisateurs effectuée

**Monitoring**
- [ ] Métriques BI Connector collectées
- [ ] Alerts configurées (latency, errors)
- [ ] Dashboard monitoring créé
- [ ] Slow query logging activé

**Security**
- [ ] User dédié BI avec permissions minimales
- [ ] SSL/TLS obligatoire
- [ ] Network isolation configurée
- [ ] Audit logging activé
- [ ] Rate limiting configuré

---

## 🎓 Conclusion

MongoDB Connector for BI est un **pont essentiel** pour organisations utilisant outils BI traditionnels. Points clés :

**Quand utiliser ?**
- ✅ Analystes métier habitués SQL/BI tools
- ✅ Migration progressive (coexistence SQL/MongoDB)
- ✅ Dashboards existants à migrer rapidement
- ✅ Read-only analytics (pas de writes via BI)

**Limites**
- ❌ Performance inférieure à MongoDB native (overhead traduction)
- ❌ Toutes fonctionnalités SQL pas supportées
- ❌ Schema rigid (moins flexible que MongoDB pur)

**Alternatives**
- MongoDB Charts (natif, pas de SQL)
- Export vers data lake (Parquet) + BI tools
- API custom pour analytics

Le BI Connector est une **solution transitoire** vers adoption MongoDB native dans outils analytics modernes, mais reste précieux pour migration progressive et adoption équipes métier.

---

**Prochaine section** : 19.7 Intégration avec Data Lakes - Patterns d'intégration MongoDB avec architectures data lake (S3, Delta Lake, Iceberg).

⏭️ [Intégration avec Apache Kafka](/19-migration-integration/07-integration-apache-kafka.md)
