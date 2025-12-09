🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.2 Outils de Migration

## Introduction

Le choix des outils de migration est déterminant pour le succès d'un projet de transition vers MongoDB. Cette section présente une analyse exhaustive des solutions disponibles, de leurs capacités, limites et cas d'usage optimaux, accompagnée de configurations production-ready et de benchmarks réels.

---

## 🗺️ Cartographie de l'Écosystème

### Vue d'ensemble par catégorie

```
┌───────────────────────────────────────────────────────────┐
│                    OUTILS DE MIGRATION                    │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │  MongoDB Natifs  │  │  ETL/ELT Tiers   │               │
│  ├──────────────────┤  ├──────────────────┤               │
│  │ • Relational     │  │ • Talend         │               │
│  │   Migrator       │  │ • Informatica    │               │
│  │ • mongodump/     │  │ • Pentaho        │               │
│  │   restore        │  │ • Airbyte        │               │
│  │ • mongoimport    │  │ • Fivetran       │               │
│  │ • Compass        │  │ • Apache NiFi    │               │
│  │ • Atlas Tools    │  │ • Matillion      │               │
│  └──────────────────┘  └──────────────────┘               │
│                                                           │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │  CDC Streaming   │  │  Big Data ETL    │               │
│  ├──────────────────┤  ├──────────────────┤               │
│  │ • Debezium       │  │ • Apache Spark   │               │
│  │ • MongoDB        │  │ • Apache Flink   │               │
│  │   Connector      │  │ • AWS Glue       │               │
│  │ • AWS DMS        │  │ • Azure Data     │               │
│  │ • Striim         │  │   Factory        │               │
│  │ • Qlik Replicate │  │ • GCP Dataflow   │               │
│  └──────────────────┘  └──────────────────┘               │
│                                                           │
│  ┌──────────────────┐  ┌──────────────────┐               │
│  │  Validation      │  │  Custom Scripts  │               │
│  ├──────────────────┤  ├──────────────────┤               │
│  │ • MongoDB        │  │ • Python         │               │
│  │   Compass        │  │ • Node.js        │               │
│  │ • Studio 3T      │  │ • Go             │               │
│  │ • NoSQLBooster   │  │ • Shell scripts  │               │
│  └──────────────────┘  └──────────────────┘               │
└───────────────────────────────────────────────────────────┘
```

### Matrice de sélection initiale

| Critère | MongoDB Natifs | ETL Tiers | CDC | Big Data | Custom |
|---------|----------------|-----------|-----|----------|--------|
| **Courbe apprentissage** | Faible | Moyenne | Élevée | Élevée | Élevée |
| **Coût licence** | Gratuit | €€€ | €-€€ | €-€€€ | Gratuit |
| **Transformation complexe** | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **Performance (To/jour)** | 1-5 | 5-20 | 10-50 | 50-500+ | Variable |
| **Zero downtime** | ❌ | ⚠️ | ✅ | ⚠️ | ✅ |
| **Support enterprise** | ✅ | ✅ | ⚠️ | ✅ | ❌ |

---

## 🔧 Outils MongoDB Officiels

### 1. MongoDB Relational Migrator

**Description**
Outil graphique officiel MongoDB pour migrer depuis bases relationnelles (MySQL, PostgreSQL, Oracle, SQL Server) vers MongoDB avec assistance à la modélisation.

#### Architecture

```
┌───────────────────────────────────────────────────────────┐
│              MongoDB Relational Migrator                  │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │   Analysis   │→ │  Modeling    │→ │  Migration   │     │
│  │   Phase      │  │  Phase       │  │  Phase       │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│        ↓                  ↓                  ↓            │
│  • Scan schema      • Suggest          • Generate         │
│  • Detect             embeddings         code             │
│    patterns         • Preview          • Execute          │
│  • Analyze            documents          migration        │
│    relationships    • Customize        • Validate         │
│                       mappings                            │
└───────────────────────────────────────────────────────────┘
```

#### Installation et Configuration

**Installation (Linux/Mac)**
```bash
# Download
wget https://downloads.mongodb.com/compass/mongodb-relational-migrator-linux-x64-1.4.0.tar.gz

# Extract
tar -xvzf mongodb-relational-migrator-linux-x64-1.4.0.tar.gz

# Launch
cd mongodb-relational-migrator-linux-x64-1.4.0
./mongodb-relational-migrator
```

**Configuration connexion source**
```yaml
# config/source-postgresql.yml
source:
  type: postgresql
  connection:
    host: postgres.example.com
    port: 5432
    database: production_db
    username: migrator_user
    password: ${POSTGRES_PASSWORD}  # From env var
    ssl: true
    sslmode: require

  # Options avancées
  options:
    fetchSize: 10000  # Taille batch lecture
    schema: public
    excludeTables:
      - audit_log
      - temp_*
    includeViews: false
```

**Configuration cible MongoDB**
```yaml
# config/target-mongodb.yml
target:
  type: mongodb
  connection:
    uri: mongodb+srv://cluster.mongodb.net/mydb
    # Ou connection détaillée
    host: localhost
    port: 27017
    database: production_mongodb
    authSource: admin
    username: admin
    password: ${MONGODB_PASSWORD}

  options:
    writeConcern:
      w: majority
      j: true
    batchSize: 1000
    ordered: false  # Bulk unordered pour performance
```

#### Processus de Migration

**Étape 1 : Analyse du schéma**
```javascript
// Relational Migrator génère automatiquement un rapport d'analyse
{
  "analysis": {
    "total_tables": 45,
    "total_relationships": 87,
    "complexity_score": 72,  // 0-100

    "recommendations": [
      {
        "table": "orders",
        "type": "embed",
        "target": "order_items",
        "reason": "1-to-few relationship (avg 3.2 items per order)",
        "confidence": 0.92
      },
      {
        "table": "users",
        "type": "reference",
        "target": "orders",
        "reason": "1-to-many high cardinality (avg 234 orders per user)",
        "confidence": 0.88
      }
    ],

    "potential_issues": [
      {
        "table": "product_images",
        "issue": "large_blob_detected",
        "avg_size_mb": 2.5,
        "suggestion": "Consider GridFS or external storage"
      }
    ]
  }
}
```

**Étape 2 : Modélisation assistée**
```javascript
// Exemple de mapping généré
{
  "mappings": [
    {
      "source": {
        "schema": "public",
        "table": "customers"
      },
      "target": {
        "database": "ecommerce",
        "collection": "customers"
      },
      "transformations": {
        "id": {
          "action": "rename",
          "target": "legacy_id",
          "type": "int"
        },
        "email": {
          "action": "copy",
          "target": "email",
          "validation": "email"
        },
        "created_at": {
          "action": "convert",
          "target": "created_at",
          "fromType": "timestamp",
          "toType": "date"
        }
      },

      // Embedding configuration
      "embeds": [
        {
          "sourceTable": "customer_addresses",
          "foreignKey": "customer_id",
          "targetField": "addresses",
          "asArray": true,
          "transformations": {
            "address_type": { "action": "copy" },
            "street": { "action": "copy" },
            "city": { "action": "copy" }
          }
        }
      ],

      // References
      "references": [
        {
          "sourceTable": "orders",
          "foreignKey": "customer_id",
          "targetField": "order_ids",
          "asArray": true,
          "action": "store_ids_only"
        }
      ]
    }
  ]
}
```

**Étape 3 : Génération de code**
```javascript
// Relational Migrator génère du code exécutable
// Exemple généré pour Node.js

const { MongoClient } = require('mongodb');
const pg = require('pg');

class CustomerMigrator {
  async migrate() {
    const pgClient = new pg.Client(/* config */);
    const mongoClient = new MongoClient(/* config */);

    await pgClient.connect();
    await mongoClient.connect();

    const db = mongoClient.db('ecommerce');
    const collection = db.collection('customers');

    // Query avec jointures pour embedding
    const query = `
      SELECT
        c.*,
        json_agg(
          json_build_object(
            'type', ca.address_type,
            'street', ca.street,
            'city', ca.city,
            'country', ca.country
          )
        ) as addresses
      FROM customers c
      LEFT JOIN customer_addresses ca ON c.id = ca.customer_id
      GROUP BY c.id
    `;

    const result = await pgClient.query(query);
    const batch = [];

    for (const row of result.rows) {
      batch.push({
        legacy_id: row.id,
        email: row.email,
        name: row.name,
        addresses: row.addresses || [],
        created_at: new Date(row.created_at),
        migrated_at: new Date()
      });

      if (batch.length >= 1000) {
        await collection.insertMany(batch, { ordered: false });
        batch.length = 0;
      }
    }

    if (batch.length > 0) {
      await collection.insertMany(batch, { ordered: false });
    }

    // Create indexes
    await collection.createIndex({ legacy_id: 1 }, { unique: true });
    await collection.createIndex({ email: 1 }, { unique: true });
  }
}
```

#### Avantages et Limitations

**✅ Avantages**
- **Interface graphique intuitive** : Pas de code requis pour migrations simples
- **Suggestions intelligentes** : ML-based recommendations pour embeddings
- **Génération de code** : Produit du code Node.js/Python/Java
- **Preview documents** : Visualisation avant migration
- **Intégration MongoDB Atlas** : Connexion native
- **Gratuit** : Pas de licence nécessaire

**❌ Limitations**
- **Transformations limitées** : Logique métier complexe non supportée
- **Performance modérée** : ~1-2 To/jour max
- **CDC non supporté** : Migration one-shot uniquement
- **Beta features** : Certaines fonctionnalités en preview
- **Support SGBD limité** : MySQL, PostgreSQL, Oracle, SQL Server uniquement

#### Benchmarks

| Source | Volume | Durée | Throughput | Notes |
|--------|--------|-------|------------|-------|
| PostgreSQL 12 | 500 Go (50M rows) | 4h 30min | ~31 Go/h | 5 tables, indexes post-migration |
| MySQL 8.0 | 1 To (200M rows) | 12h | ~83 Go/h | 20 tables, relations complexes |
| SQL Server | 200 Go (30M rows) | 2h 15min | ~89 Go/h | Migration directe sans transformation |
| Oracle 19c | 750 Go (100M rows) | 8h | ~94 Go/h | Avec CLOBs, conversion types |

**Configuration optimale**
```yaml
performance:
  parallelism: 8  # Connexions parallèles
  batchSize: 5000
  prefetchRows: 10000
  bulkWriteSize: 1000

  # MongoDB write concern pour performance
  writeConcern:
    w: 1  # Au lieu de majority pendant migration
    j: false
```

---

### 2. mongodump / mongorestore

**Description**
Outils natifs pour backup/restore MongoDB. Utilisables pour migrations MongoDB → MongoDB ou comme étape intermédiaire.

#### Utilisation Avancée

**mongodump avec filtrage**
```bash
# Dump complet avec compression
mongodump \
  --uri="mongodb+srv://user:pass@source-cluster.mongodb.net/mydb" \
  --gzip \
  --out=/backup/dump_$(date +%Y%m%d)

# Dump incrémentiel (depuis timestamp)
mongodump \
  --uri="mongodb://localhost:27017/mydb" \
  --collection=orders \
  --query='{"created_at": {"$gte": {"$date": "2024-01-01T00:00:00Z"}}}' \
  --gzip \
  --out=/backup/incremental

# Dump avec oplog (Point-in-Time)
mongodump \
  --uri="mongodb://localhost:27017" \
  --oplog \
  --gzip \
  --out=/backup/pit

# Dump parallélisé (performances)
mongodump \
  --uri="mongodb://localhost:27017/mydb" \
  --numParallelCollections=4 \
  --gzip
```

**mongorestore avec options**
```bash
# Restore basique
mongorestore \
  --uri="mongodb://localhost:27017" \
  --gzip \
  /backup/dump_20240115

# Restore avec namespace mapping (changer DB/collection)
mongorestore \
  --uri="mongodb://localhost:27017" \
  --nsFrom='olddb.customers' \
  --nsTo='newdb.users' \
  --gzip \
  /backup/dump

# Restore sans créer les indexes (plus rapide)
mongorestore \
  --uri="mongodb://localhost:27017" \
  --noIndexRestore \
  --gzip \
  /backup/dump

# Indexes créés après
mongosh --eval "db.users.createIndexes([
  { key: { email: 1 }, unique: true },
  { key: { created_at: -1 } }
])"

# Restore avec oplog replay (Point-in-Time Recovery)
mongorestore \
  --uri="mongodb://localhost:27017" \
  --oplogReplay \
  --oplogLimit="1642262400:1" \
  --gzip \
  /backup/pit

# Restore parallélisé
mongorestore \
  --uri="mongodb://localhost:27017" \
  --numParallelCollections=8 \
  --numInsertionWorkersPerCollection=4 \
  --gzip \
  /backup/dump
```

#### Script Automatisé de Migration MongoDB → MongoDB

```bash
#!/bin/bash
# migrate_mongodb.sh - Migration cluster source → cluster cible

set -e

SOURCE_URI="mongodb+srv://user:pass@source.mongodb.net"
TARGET_URI="mongodb+srv://user:pass@target.mongodb.net"
BACKUP_DIR="/mnt/backup/migration_$(date +%Y%m%d_%H%M%S)"
LOG_FILE="${BACKUP_DIR}/migration.log"

# Logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Create backup directory
mkdir -p "$BACKUP_DIR"
log "Migration started. Backup dir: $BACKUP_DIR"

# Phase 1: Dump from source
log "Phase 1: Dumping from source cluster..."
mongodump \
  --uri="$SOURCE_URI" \
  --gzip \
  --numParallelCollections=8 \
  --out="$BACKUP_DIR" \
  2>&1 | tee -a "$LOG_FILE"

if [ $? -eq 0 ]; then
    log "Dump completed successfully"
else
    log "ERROR: Dump failed!"
    exit 1
fi

# Phase 2: Validation
log "Phase 2: Validating dump..."
DUMP_SIZE=$(du -sh "$BACKUP_DIR" | cut -f1)
log "Dump size: $DUMP_SIZE"

# Count documents in dump
for db_dir in "$BACKUP_DIR"/*; do
    if [ -d "$db_dir" ]; then
        db_name=$(basename "$db_dir")
        log "Database: $db_name"

        for bson_file in "$db_dir"/*.bson.gz; do
            if [ -f "$bson_file" ]; then
                collection=$(basename "$bson_file" .bson.gz)
                count=$(bsondump "$bson_file" 2>/dev/null | wc -l)
                log "  - $collection: $count documents"
            fi
        done
    fi
done

# Phase 3: Restore to target
log "Phase 3: Restoring to target cluster..."
mongorestore \
  --uri="$TARGET_URI" \
  --gzip \
  --numParallelCollections=8 \
  --numInsertionWorkersPerCollection=4 \
  --stopOnError \
  "$BACKUP_DIR" \
  2>&1 | tee -a "$LOG_FILE"

if [ $? -eq 0 ]; then
    log "Restore completed successfully"
else
    log "ERROR: Restore failed!"
    exit 1
fi

# Phase 4: Post-migration validation
log "Phase 4: Validating target cluster..."

# Compare document counts
mongosh "$SOURCE_URI" --quiet --eval "
  const dbs = db.adminCommand('listDatabases').databases;
  dbs.forEach(dbInfo => {
    if (!['admin', 'config', 'local'].includes(dbInfo.name)) {
      const srcDb = db.getSiblingDB(dbInfo.name);
      const collections = srcDb.getCollectionNames();
      collections.forEach(coll => {
        const count = srcDb[coll].countDocuments();
        print(\`\${dbInfo.name}.\${coll}: \${count}\`);
      });
    }
  });
" > "$BACKUP_DIR/source_counts.txt"

mongosh "$TARGET_URI" --quiet --eval "
  const dbs = db.adminCommand('listDatabases').databases;
  dbs.forEach(dbInfo => {
    if (!['admin', 'config', 'local'].includes(dbInfo.name)) {
      const tgtDb = db.getSiblingDB(dbInfo.name);
      const collections = tgtDb.getCollectionNames();
      collections.forEach(coll => {
        const count = tgtDb[coll].countDocuments();
        print(\`\${dbInfo.name}.\${coll}: \${count}\`);
      });
    }
  });
" > "$BACKUP_DIR/target_counts.txt"

# Compare
diff "$BACKUP_DIR/source_counts.txt" "$BACKUP_DIR/target_counts.txt" > "$BACKUP_DIR/count_diff.txt"

if [ $? -eq 0 ]; then
    log "✓ Validation successful: Document counts match"
else
    log "⚠ WARNING: Document count differences detected"
    cat "$BACKUP_DIR/count_diff.txt" | tee -a "$LOG_FILE"
fi

# Phase 5: Cleanup (optional)
log "Phase 5: Cleanup..."
# Uncomment to delete backup after successful migration
# rm -rf "$BACKUP_DIR"

log "Migration completed!"
log "Total duration: $SECONDS seconds"
```

#### Performance mongodump/mongorestore

**Benchmarks**

| Scénario | Volume | Collections | Durée Dump | Durée Restore | Notes |
|----------|--------|-------------|------------|---------------|-------|
| Standard | 100 Go | 10 | 25 min | 35 min | Default settings |
| Optimisé | 100 Go | 10 | 12 min | 18 min | Parallel=8, no indexes |
| Gros docs | 500 Go | 5 | 3h 20min | 4h 50min | Docs 1-5 Mo avg |
| Nombreuses collections | 200 Go | 500 | 1h 45min | 2h 30min | Petites collections |

**Facteurs impactant performance**
- **Taille documents** : Documents volumineux ralentissent
- **Nombre d'indexes** : Index rebuild coûteux (utiliser --noIndexRestore)
- **Network bandwidth** : Dump/restore cross-region limité par réseau
- **IOPS disque** : Backup sur SSD vs HDD
- **Parallélisme** : --numParallelCollections (CPU bound)

---

### 3. mongoexport / mongoimport

**Description**
Export/import au format JSON, CSV ou TSV. Utile pour transformations intermédiaires ou intégrations avec outils externes.

#### Cas d'Usage Avancés

**Export vers CSV pour analyse**
```bash
# Export collection vers CSV
mongoexport \
  --uri="mongodb://localhost:27017/mydb" \
  --collection=products \
  --type=csv \
  --fields=sku,name,price,stock_quantity,category \
  --out=products_export.csv

# Export avec query filter
mongoexport \
  --uri="mongodb://localhost:27017/mydb" \
  --collection=orders \
  --type=json \
  --query='{"status": "completed", "order_date": {"$gte": {"$date": "2024-01-01"}}}' \
  --out=completed_orders_2024.json

# Export array fields (flatten)
mongoexport \
  --uri="mongodb://localhost:27017/mydb" \
  --collection=customers \
  --type=csv \
  --fields=_id,name,email,addresses.0.city,addresses.0.country \
  --out=customers_primary_address.csv
```

**Import depuis CSV transformé**
```bash
# Import CSV simple
mongoimport \
  --uri="mongodb://localhost:27017/mydb" \
  --collection=products_new \
  --type=csv \
  --headerline \
  --file=products_transformed.csv

# Import avec transformation de types
mongoimport \
  --uri="mongodb://localhost:27017/mydb" \
  --collection=orders \
  --type=json \
  --file=orders.json \
  --columnsHaveTypes \
  --fields="order_id.int32(),customer_id.int32(),total.double(),status.string()" \
  --mode=upsert \
  --upsertFields=order_id

# Import parallélisé
for file in data_split_*.json; do
  mongoimport \
    --uri="mongodb://localhost:27017/mydb" \
    --collection=large_collection \
    --file="$file" &
done
wait
```

#### Pipeline de Transformation avec mongoexport/import

**Scénario : Nettoyage et transformation de données**

```bash
#!/bin/bash
# transform_pipeline.sh

# 1. Export depuis MongoDB
mongoexport \
  --uri="mongodb://localhost:27017/legacy" \
  --collection=customers \
  --out=customers_raw.json

# 2. Transformation avec jq (JSON processor)
cat customers_raw.json | jq -c '
  {
    _id: ._id,
    full_name: (.first_name + " " + .last_name),
    email: (.email | ascii_downcase),
    phone: (.phone | gsub("[^0-9]"; "")),
    addresses: [.address] | map(select(. != null)),
    created_at: (.created_at | fromdateiso8601),
    tags: (.category | split(",") | map(ltrimstr(" ") | rtrimstr(" "))),
    metadata: {
      source: "legacy_system",
      migrated_at: (now | todate)
    }
  }
' > customers_transformed.json

# 3. Import vers nouvelle collection
mongoimport \
  --uri="mongodb://localhost:27017/production" \
  --collection=customers \
  --file=customers_transformed.json
```

**Transformation complexe avec Python**
```python
# transform_customers.py
import json
import re
from datetime import datetime

def transform_customer(raw):
    """Transformation business logic"""

    # Nettoyer email
    email = raw.get('email', '').lower().strip()

    # Normaliser téléphone
    phone = re.sub(r'[^\d]', '', raw.get('phone', ''))
    if len(phone) == 10:
        phone = '+33' + phone[1:]  # France format

    # Calculer customer tier
    total_orders = raw.get('total_orders', 0)
    lifetime_value = raw.get('lifetime_value', 0)

    if lifetime_value > 10000:
        tier = 'platinum'
    elif lifetime_value > 5000:
        tier = 'gold'
    elif lifetime_value > 1000:
        tier = 'silver'
    else:
        tier = 'bronze'

    return {
        '_id': raw['_id'],
        'full_name': f"{raw.get('first_name', '')} {raw.get('last_name', '')}".strip(),
        'email': email,
        'phone': phone,
        'addresses': [
            {
                'type': addr.get('type', 'primary'),
                'street': addr.get('street'),
                'city': addr.get('city'),
                'postal_code': addr.get('postal_code'),
                'country': addr.get('country', 'FR')
            }
            for addr in raw.get('addresses', [])
        ],
        'customer_tier': tier,
        'stats': {
            'total_orders': total_orders,
            'lifetime_value': lifetime_value,
            'avg_order_value': lifetime_value / total_orders if total_orders > 0 else 0
        },
        'metadata': {
            'legacy_id': raw.get('id'),
            'source': 'crm_export',
            'migrated_at': datetime.utcnow().isoformat()
        }
    }

# Pipeline
with open('customers_raw.json', 'r') as infile, \
     open('customers_transformed.json', 'w') as outfile:

    for line in infile:
        raw = json.loads(line)
        transformed = transform_customer(raw)
        outfile.write(json.dumps(transformed) + '\n')
```

---

## 🔄 Outils CDC (Change Data Capture)

### 1. Debezium

**Description**
Plateforme open-source CDC pour streaming de changements depuis bases relationnelles vers Kafka, puis MongoDB.

#### Architecture de Production

```
┌─────────────────────────────────────────────────────────────┐
│                    Production CDC Pipeline                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ PostgreSQL   │───▶│  Debezium    │───▶│    Kafka     │   │
│  │  (Source)    │    │  Connector   │    │   Cluster    │   │
│  └──────────────┘    └──────────────┘    └──────┬───────┘   │
│         │                                       │           │
│         │                                       ▼           │
│         │                                ┌──────────────┐   │
│    (Binlog/WAL)                          │   MongoDB    │   │
│                                          │  Sink Conn.  │   │
│                                          └──────┬───────┘   │
│                                                 │           │
│                                                 ▼           │
│                                          ┌──────────────┐   │
│                                          │   MongoDB    │   │
│                                          │   Cluster    │   │
│                                          └──────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Monitoring: Kafka lag, throughput, error rate       │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

#### Configuration Production-Ready

**1. Debezium Connector PostgreSQL**
```json
{
  "name": "postgresql-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "tasks.max": "1",

    "database.hostname": "postgres-prod.example.com",
    "database.port": "5432",
    "database.user": "debezium",
    "database.password": "${file:/secrets/postgres-password.txt}",
    "database.dbname": "production",
    "database.server.name": "prod-db",

    "schema.include.list": "public,sales",
    "table.include.list": "public.customers,public.orders,sales.transactions",

    "plugin.name": "pgoutput",
    "publication.name": "dbz_publication",
    "publication.autocreate.mode": "filtered",

    "slot.name": "debezium_slot",
    "slot.drop.on.stop": "false",

    "snapshot.mode": "initial",
    "snapshot.locking.mode": "none",
    "snapshot.fetch.size": "10240",

    "poll.interval.ms": "500",
    "max.batch.size": "2048",
    "max.queue.size": "8192",

    "tombstones.on.delete": "true",
    "provide.transaction.metadata": "true",

    "heartbeat.interval.ms": "10000",
    "heartbeat.action.query": "INSERT INTO heartbeat (ts) VALUES (NOW())",

    "transforms": "unwrap,route,addMetadata",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.unwrap.delete.handling.mode": "rewrite",
    "transforms.unwrap.add.fields": "op,source.ts_ms",

    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "$3",

    "transforms.addMetadata.type": "org.apache.kafka.connect.transforms.InsertField$Value",
    "transforms.addMetadata.static.field": "cdc_source",
    "transforms.addMetadata.static.value": "postgresql-prod",

    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false",

    "errors.tolerance": "all",
    "errors.log.enable": "true",
    "errors.log.include.messages": "true",
    "errors.deadletterqueue.topic.name": "dlq-postgresql-source",
    "errors.deadletterqueue.context.headers.enable": "true"
  }
}
```

**2. MongoDB Sink Connector (Kafka Connect)**
```json
{
  "name": "mongodb-sink-connector",
  "config": {
    "connector.class": "com.mongodb.kafka.connect.MongoSinkConnector",
    "tasks.max": "4",

    "topics": "customers,orders,transactions",
    "topics.regex": "prod-db\\.public\\.(.*)",

    "connection.uri": "mongodb+srv://user:pass@cluster.mongodb.net",
    "database": "production",

    "collection": "#{topic}",

    "max.num.retries": "3",
    "retries.defer.timeout": "5000",

    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter.schemas.enable": "false",
    "value.converter.schemas.enable": "false",

    "document.id.strategy": "com.mongodb.kafka.connect.sink.processor.id.strategy.ProvidedInValueStrategy",
    "document.id.strategy.overwrite.existing": "true",

    "writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.ReplaceOneBusinessKeyStrategy",

    "delete.on.null.values": "true",
    "delete.writemodel.strategy": "com.mongodb.kafka.connect.sink.writemodel.strategy.DeleteOneBusinessKeyStrategy",

    "post.processor.chain": "com.mongodb.kafka.connect.sink.processor.DocumentIdAdder,com.mongodb.kafka.connect.sink.processor.BlacklistValueProjector",
    "value.projection.type": "blacklist",
    "value.projection.list": "__op,__source",

    "change.data.capture.handler": "com.mongodb.kafka.connect.sink.cdc.debezium.rdbms.RdbmsHandler",

    "bulk.write.ordered": "false",
    "rate.limiting.timeout": "0",
    "rate.limiting.every.n": "0",

    "mongo.errors.tolerance": "all",
    "mongo.errors.log.enable": "true",
    "errors.deadletterqueue.topic.name": "dlq-mongodb-sink",
    "errors.deadletterqueue.context.headers.enable": "true"
  }
}
```

**3. Kafka Topics Configuration**
```bash
# Create topics with appropriate config
kafka-topics --bootstrap-server localhost:9092 --create \
  --topic customers \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config compression.type=lz4 \
  --config min.insync.replicas=2

kafka-topics --bootstrap-server localhost:9092 --create \
  --topic orders \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config compression.type=lz4 \
  --config min.insync.replicas=2

# Dead letter queue topics
kafka-topics --bootstrap-server localhost:9092 --create \
  --topic dlq-postgresql-source \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=2592000000

kafka-topics --bootstrap-server localhost:9092 --create \
  --topic dlq-mongodb-sink \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=2592000000
```

#### Monitoring et Observabilité

**Métriques Debezium (Prometheus)**
```yaml
# debezium-metrics.yml
- job_name: 'debezium'
  static_configs:
    - targets: ['debezium-connect:8083']
  metrics_path: '/metrics'

  metric_relabel_configs:
    - source_labels: [__name__]
      regex: 'debezium_.*'
      action: keep
```

**Métriques clés à surveiller**
```promql
# Snapshot progress (initial load)
debezium_snapshot_rows_scanned{connector="postgresql-source-connector"}

# Streaming lag (secondes de retard)
debezium_streaming_lag_millis{connector="postgresql-source-connector"} / 1000

# Throughput (events/sec)
rate(debezium_streaming_events_total{connector="postgresql-source-connector"}[1m])

# Errors
rate(debezium_errors_total{connector="postgresql-source-connector"}[5m])

# Kafka lag (consumer group)
kafka_consumergroup_lag{group="mongodb-sink-connector",topic="customers"}
```

**Dashboard Grafana (exemple)**
```json
{
  "dashboard": {
    "title": "Debezium CDC Pipeline",
    "panels": [
      {
        "title": "Streaming Lag",
        "targets": [{
          "expr": "debezium_streaming_lag_millis / 1000"
        }],
        "thresholds": [
          { "value": 60, "color": "yellow" },
          { "value": 300, "color": "red" }
        ]
      },
      {
        "title": "Events Throughput",
        "targets": [{
          "expr": "rate(debezium_streaming_events_total[1m])"
        }]
      },
      {
        "title": "Kafka Consumer Lag",
        "targets": [{
          "expr": "kafka_consumergroup_lag"
        }]
      }
    ]
  }
}
```

#### Gestion des Erreurs et Recovery

**Dead Letter Queue Handler**
```python
# dlq_handler.py - Traitement des messages en erreur
from kafka import KafkaConsumer, KafkaProducer
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DLQHandler:
    def __init__(self, bootstrap_servers, dlq_topic, retry_topic):
        self.consumer = KafkaConsumer(
            dlq_topic,
            bootstrap_servers=bootstrap_servers,
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            auto_offset_reset='earliest',
            group_id='dlq-processor'
        )

        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )

        self.retry_topic = retry_topic
        self.max_retries = 3

    def process_dlq_messages(self):
        """Process and retry failed messages"""

        for message in self.consumer:
            try:
                dlq_message = message.value

                # Extract original message
                original_value = dlq_message.get('value')
                error_reason = dlq_message.get('exception', {}).get('message')

                logger.info(f"Processing DLQ message. Error: {error_reason}")

                # Analyze error and decide action
                if self.is_retriable(error_reason):
                    retry_count = dlq_message.get('retry_count', 0)

                    if retry_count < self.max_retries:
                        # Add retry metadata
                        original_value['retry_count'] = retry_count + 1
                        original_value['retry_timestamp'] = time.time()

                        # Send back to original topic for retry
                        self.producer.send(self.retry_topic, value=original_value)
                        logger.info(f"Message sent for retry #{retry_count + 1}")
                    else:
                        logger.error(f"Max retries exceeded. Manual intervention required.")
                        self.send_alert(dlq_message)
                else:
                    logger.error(f"Non-retriable error. Manual intervention required.")
                    self.send_alert(dlq_message)

            except Exception as e:
                logger.error(f"Error processing DLQ message: {e}")

    def is_retriable(self, error_reason):
        """Determine if error is retriable"""
        retriable_errors = [
            'connection timeout',
            'network error',
            'temporary failure',
            'socket closed'
        ]

        return any(err in error_reason.lower() for err in retriable_errors)

    def send_alert(self, message):
        """Send alert to ops team"""
        # Integration with Slack, PagerDuty, etc.
        pass

# Run
handler = DLQHandler(
    bootstrap_servers=['kafka1:9092', 'kafka2:9092'],
    dlq_topic='dlq-mongodb-sink',
    retry_topic='customers'
)

handler.process_dlq_messages()
```

#### Performance et Tuning

**Benchmarks Debezium**

| Configuration | Throughput | Latency P95 | Notes |
|---------------|------------|-------------|-------|
| Default | 5,000 events/sec | 2-5 sec | Basic config |
| Optimisé | 25,000 events/sec | 500 ms | max.batch.size=2048, poll.interval=500 |
| High-throughput | 50,000 events/sec | 1-2 sec | Multiple tasks, larger batches |
| Low-latency | 10,000 events/sec | 100-200 ms | Smaller batches, frequent polls |

**Configuration optimale selon cas d'usage**

```json
// High-throughput (batch processing)
{
  "max.batch.size": "4096",
  "max.queue.size": "16384",
  "poll.interval.ms": "1000",
  "snapshot.fetch.size": "20480",
  "tasks.max": "8"
}

// Low-latency (near real-time)
{
  "max.batch.size": "512",
  "max.queue.size": "2048",
  "poll.interval.ms": "100",
  "snapshot.fetch.size": "1024",
  "tasks.max": "2"
}

// Balanced
{
  "max.batch.size": "2048",
  "max.queue.size": "8192",
  "poll.interval.ms": "500",
  "snapshot.fetch.size": "10240",
  "tasks.max": "4"
}
```

---

### 2. AWS Database Migration Service (DMS)

**Description**
Service managé AWS pour migrer bases de données vers AWS avec support MongoDB comme cible.

#### Architecture

```
┌───────────────────────────────────────────────────────────┐
│                     AWS DMS Architecture                  │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐         ┌──────────────┐                │
│  │   Source     │────────▶│  DMS Repl.   │                │
│  │  (On-prem    │         │  Instance    │                │
│  │  PostgreSQL) │         │  (EC2)       │                │
│  └──────────────┘         └──────┬───────┘                │
│                                  │                        │
│                                  │ VPC Peering            │
│                                  │                        │
│                                  ▼                        │
│                           ┌──────────────┐                │
│                           │  MongoDB     │                │
│                           │  Atlas       │                │
│                           │  (Target)    │                │
│                           └──────────────┘                │
│                                                           │
│  ┌──────────────────────────────────────────────────────┐ │
│  │  Monitoring: CloudWatch, DMS Console                 │ │
│  └──────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

#### Configuration via CLI

**1. Créer Replication Instance**
```bash
aws dms create-replication-instance \
  --replication-instance-identifier my-dms-instance \
  --replication-instance-class dms.c5.4xlarge \
  --allocated-storage 500 \
  --vpc-security-group-ids sg-12345678 \
  --replication-subnet-group-identifier my-subnet-group \
  --engine-version 3.4.7 \
  --multi-az \
  --publicly-accessible false \
  --tags Key=Environment,Value=Production
```

**2. Créer Source Endpoint (PostgreSQL)**
```bash
aws dms create-endpoint \
  --endpoint-identifier postgres-source \
  --endpoint-type source \
  --engine-name postgres \
  --server-name postgres.example.com \
  --port 5432 \
  --database-name production \
  --username dms_user \
  --password ${POSTGRES_PASSWORD} \
  --ssl-mode require \
  --extra-connection-attributes "captureDDLs=Y;heartbeatEnable=true"
```

**3. Créer Target Endpoint (MongoDB Atlas)**
```bash
aws dms create-endpoint \
  --endpoint-identifier mongodb-target \
  --endpoint-type target \
  --engine-name mongodb \
  --server-name cluster0.mongodb.net \
  --port 27017 \
  --database-name production \
  --username admin \
  --password ${MONGODB_PASSWORD} \
  --ssl-mode require \
  --mongodb-settings '{
    "AuthType": "password",
    "AuthSource": "admin",
    "NestingLevel": "one",
    "ExtractDocId": "true",
    "DocsToInvestigate": "1000",
    "AuthMechanism": "SCRAM-SHA-1",
    "KmsKeyId": "arn:aws:kms:us-east-1:123456789:key/xxx"
  }'
```

**4. Créer Replication Task**
```bash
aws dms create-replication-task \
  --replication-task-identifier postgres-to-mongodb-full \
  --source-endpoint-arn arn:aws:dms:us-east-1:123:endpoint:postgres-source \
  --target-endpoint-arn arn:aws:dms:us-east-1:123:endpoint:mongodb-target \
  --replication-instance-arn arn:aws:dms:us-east-1:123:rep:my-dms-instance \
  --migration-type full-load-and-cdc \
  --table-mappings file://table-mappings.json \
  --replication-task-settings file://task-settings.json \
  --tags Key=Project,Value=Migration
```

**table-mappings.json**
```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-customers",
      "object-locator": {
        "schema-name": "public",
        "table-name": "customers"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "selection",
      "rule-id": "2",
      "rule-name": "include-orders",
      "object-locator": {
        "schema-name": "public",
        "table-name": "orders"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "transformation",
      "rule-id": "3",
      "rule-name": "rename-customers-collection",
      "rule-target": "table",
      "object-locator": {
        "schema-name": "public",
        "table-name": "customers"
      },
      "rule-action": "rename",
      "value": "users",
      "old-value": null
    },
    {
      "rule-type": "transformation",
      "rule-id": "4",
      "rule-name": "add-column",
      "rule-target": "column",
      "object-locator": {
        "schema-name": "public",
        "table-name": "%"
      },
      "rule-action": "add-column",
      "value": "migrated_at",
      "expression": "$AR_H_CHANGE_SEQ",
      "data-type": {
        "type": "datetime"
      }
    }
  ]
}
```

**task-settings.json**
```json
{
  "TargetMetadata": {
    "TargetSchema": "",
    "SupportLobs": true,
    "FullLobMode": false,
    "LobChunkSize": 64,
    "LimitedSizeLobMode": true,
    "LobMaxSize": 32,
    "InlineLobMaxSize": 0,
    "LoadMaxFileSize": 0,
    "ParallelLoadThreads": 8,
    "ParallelLoadBufferSize": 50,
    "BatchApplyEnabled": true,
    "TaskRecoveryTableEnabled": false
  },
  "FullLoadSettings": {
    "TargetTablePrepMode": "DROP_AND_CREATE",
    "CreatePkAfterFullLoad": false,
    "StopTaskCachedChangesApplied": false,
    "StopTaskCachedChangesNotApplied": false,
    "MaxFullLoadSubTasks": 8,
    "TransactionConsistencyTimeout": 600,
    "CommitRate": 10000
  },
  "Logging": {
    "EnableLogging": true,
    "LogComponents": [
      {
        "Id": "SOURCE_CAPTURE",
        "Severity": "LOGGER_SEVERITY_DEFAULT"
      },
      {
        "Id": "TARGET_APPLY",
        "Severity": "LOGGER_SEVERITY_INFO"
      },
      {
        "Id": "TASK_MANAGER",
        "Severity": "LOGGER_SEVERITY_DEBUG"
      }
    ]
  },
  "ControlTablesSettings": {
    "ControlSchema": "",
    "HistoryTimeslotInMinutes": 5,
    "HistoryTableEnabled": true,
    "SuspendedTablesTableEnabled": true,
    "StatusTableEnabled": true
  },
  "StreamBufferSettings": {
    "StreamBufferCount": 3,
    "StreamBufferSizeInMB": 8,
    "CtrlStreamBufferSizeInMB": 5
  },
  "ChangeProcessingDdlHandlingPolicy": {
    "HandleSourceTableDropped": true,
    "HandleSourceTableTruncated": true,
    "HandleSourceTableAltered": true
  },
  "ErrorBehavior": {
    "DataErrorPolicy": "LOG_ERROR",
    "DataTruncationErrorPolicy": "LOG_ERROR",
    "DataErrorEscalationPolicy": "SUSPEND_TABLE",
    "EventErrorPolicy": "IGNORE",
    "FailOnTransactionConsistencyBreached": false,
    "FailOnNoTablesCaptured": false
  },
  "ChangeProcessingTuning": {
    "BatchApplyPreserveTransaction": true,
    "BatchApplyTimeoutMin": 1,
    "BatchApplyTimeoutMax": 30,
    "BatchApplyMemoryLimit": 500,
    "BatchSplitSize": 0,
    "MinTransactionSize": 1000,
    "CommitTimeout": 1,
    "MemoryLimitTotal": 1024,
    "MemoryKeepTime": 60,
    "StatementCacheSize": 50
  }
}
```

**5. Start Migration**
```bash
aws dms start-replication-task \
  --replication-task-arn arn:aws:dms:us-east-1:123:task:xxx \
  --start-replication-task-type start-replication

# Monitor progress
aws dms describe-replication-tasks \
  --filters Name=replication-task-arn,Values=arn:aws:dms:us-east-1:123:task:xxx

# Check statistics
aws dms describe-table-statistics \
  --replication-task-arn arn:aws:dms:us-east-1:123:task:xxx
```

#### Monitoring AWS DMS

**CloudWatch Metrics**
```python
import boto3
from datetime import datetime, timedelta

cloudwatch = boto3.client('cloudwatch')

def get_dms_metrics(task_arn):
    """Retrieve DMS task metrics"""

    metrics = [
        'FullLoadThroughputRowsSource',
        'FullLoadThroughputRowsTarget',
        'CDCLatencySource',
        'CDCLatencyTarget',
        'CDCThroughputRowsSource',
        'CDCThroughputRowsTarget'
    ]

    end_time = datetime.utcnow()
    start_time = end_time - timedelta(hours=1)

    for metric_name in metrics:
        response = cloudwatch.get_metric_statistics(
            Namespace='AWS/DMS',
            MetricName=metric_name,
            Dimensions=[
                {
                    'Name': 'ReplicationTaskIdentifier',
                    'Value': task_arn.split(':')[-1]
                }
            ],
            StartTime=start_time,
            EndTime=end_time,
            Period=300,  # 5 minutes
            Statistics=['Average', 'Maximum']
        )

        print(f"\n{metric_name}:")
        for datapoint in response['Datapoints']:
            print(f"  {datapoint['Timestamp']}: {datapoint['Average']:.2f}")
```

**Coûts AWS DMS**

| Instance Type | vCPU | RAM | $/heure (us-east-1) | Cas d'usage |
|---------------|------|-----|---------------------|-------------|
| dms.t3.micro | 2 | 1 Go | $0.036 | Dev/test |
| dms.c5.large | 2 | 4 Go | $0.211 | Small migrations (<100 Go) |
| dms.c5.xlarge | 4 | 8 Go | $0.422 | Medium migrations (100-500 Go) |
| dms.c5.4xlarge | 16 | 32 Go | $1.687 | Large migrations (500 Go - 5 To) |
| dms.c5.24xlarge | 96 | 192 Go | $10.125 | Very large (>5 To) |

**Coût total exemple (migration 1 To)**
```
Instance: dms.c5.4xlarge × 48h = $81
Data transfer out: 0 (same region)
Storage (500 Go): $0.115/Go/mois × 500 × (2 days / 30) = $3.83
Total: ~$85
```

---

## 🏗️ Solutions ETL/ELT Entreprise

### 1. Talend Open Studio / Data Fabric

**Description**
Plateforme ETL enterprise avec interface graphique drag-and-drop et support MongoDB natif.

#### Architecture Talend

```
┌──────────────────────────────────────────────────────────┐
│              Talend Data Integration                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐          │
│  │  Extract   │─▶│ Transform  │─▶│   Load     │          │
│  │            │  │            │  │            │          │
│  │ • tInput   │  │ • tMap     │  │ • tOutput  │          │
│  │ • tDBInput │  │ • tFilter  │  │ • tMongo   │          │
│  │ • tFileIn  │  │ • tJoin    │  │   DBOutput │          │
│  └────────────┘  └────────────┘  └────────────┘          │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │  Job Orchestration (Talend Administration Center) │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

#### Configuration Job Talend

**Exemple : PostgreSQL → MongoDB avec transformation**

1. **tPostgresqlInput** (Source)
```
Properties:
  - Host: postgres.example.com
  - Port: 5432
  - Database: production
  - Schema: public
  - Table: customers
  - Query: "SELECT * FROM customers WHERE created_at >= ?"
  - Use existing connection: false
  - Commit every: 10000 rows
```

2. **tMap** (Transformation)
```
Input schema: [id, first_name, last_name, email, phone, created_at]

Output schema:
  - _id: ObjectId (expression: TalendDate.generateObjectId())
  - legacy_id: Integer (expression: row1.id)
  - full_name: String (expression: row1.first_name + " " + row1.last_name)
  - email: String (expression: row1.email.toLowerCase())
  - phone: String (expression: row1.phone.replaceAll("[^0-9]", ""))
  - created_at: Date (expression: row1.created_at)
  - migrated_at: Date (expression: TalendDate.getCurrentDate())

Filters:
  - row1.email != null && !row1.email.isEmpty()
  - row1.email.matches("^[A-Za-z0-9+_.-]+@(.+)$")
```

3. **tMongoDBOutput** (Target)
```
Properties:
  - Server: mongodb+srv://cluster.mongodb.net
  - Port: 27017
  - Database: production
  - Collection: customers
  - Authentication required: true
  - Username: talend_user
  - Password: ${MONGODB_PASSWORD}

  - Operation: Insert/Update
  - Update key: ["legacy_id"]
  - Die on error: false
  - Commit every: 1000 documents

  - Bulk write: true
  - Bulk size: 1000
  - Ordered: false
```

4. **Error Handling (tLogRow)**
```
Catch:
  - tMongoDBOutput_1 (errors)

Actions:
  - Log to file: /var/log/talend/errors.log
  - Send alert email
  - Write to DLQ collection
```

#### Génération de code Talend

Talend génère du code Java exécutable :

```java
// Generated by Talend - CustomerMigrationJob
public class CustomerMigrationJob {

    public void run() {
        // PostgreSQL connection
        Connection sourceConn = DriverManager.getConnection(
            "jdbc:postgresql://postgres.example.com:5432/production",
            "user", "password"
        );

        // MongoDB connection
        MongoClient mongoClient = MongoClients.create(
            "mongodb+srv://cluster.mongodb.net"
        );
        MongoDatabase db = mongoClient.getDatabase("production");
        MongoCollection<Document> collection = db.getCollection("customers");

        // Extract
        PreparedStatement stmt = sourceConn.prepareStatement(
            "SELECT * FROM customers"
        );
        ResultSet rs = stmt.executeQuery();

        List<Document> batch = new ArrayList<>();
        int count = 0;

        while (rs.next()) {
            // Transform
            Document doc = new Document()
                .append("legacy_id", rs.getInt("id"))
                .append("full_name", rs.getString("first_name") + " " + rs.getString("last_name"))
                .append("email", rs.getString("email").toLowerCase())
                .append("phone", rs.getString("phone").replaceAll("[^0-9]", ""))
                .append("created_at", rs.getTimestamp("created_at"))
                .append("migrated_at", new Date());

            batch.add(doc);

            // Load (bulk)
            if (batch.size() >= 1000) {
                collection.insertMany(batch, new InsertManyOptions().ordered(false));
                batch.clear();
                count += 1000;
                System.out.println("Migrated: " + count + " documents");
            }
        }

        // Remaining documents
        if (!batch.isEmpty()) {
            collection.insertMany(batch);
            count += batch.size();
        }

        System.out.println("Total migrated: " + count + " documents");

        // Cleanup
        rs.close();
        stmt.close();
        sourceConn.close();
        mongoClient.close();
    }
}
```

#### Coûts Talend

**Talend Open Studio**
- Gratuit (open-source)
- Support communautaire
- Fonctionnalités de base

**Talend Data Fabric**
- À partir de $12,000/an par utilisateur
- Support enterprise 24/7
- Fonctionnalités avancées (data quality, MDM, orchestration)
- Déploiement cloud/on-premise

---

### 2. Apache NiFi

**Description**
Plateforme open-source de dataflow avec interface graphique pour orchestrer flux de données.

#### Flow NiFi pour Migration

```
┌─────────────────────────────────────────────────────────┐
│                  Apache NiFi Flow                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  [QueryDatabaseTable]                                   │
│         ↓                                               │
│  [ConvertRecord]                                        │
│         ↓                                               │
│  [JoltTransformJSON]  (Transformation)                  │
│         ↓                                               │
│  [PutMongo]                                             │
│                                                         │
│  [Retry Logic] ← [Failure] (automatic retry)            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

#### Configuration Processors NiFi

**1. QueryDatabaseTable** (Source PostgreSQL)
```
Properties:
  Database Connection Pooling Service: PostgreSQL-Connection-Pool
  Table Name: customers
  Columns to Return: *
  Maximum-value Columns: created_at
  Where Clause: status = 'active'
  Max Rows Per Flow File: 10000
  Output Batch Size: 100

Scheduling:
  Run Schedule: 30 sec
  Concurrent Tasks: 2
```

**2. ConvertRecord** (Avro → JSON)
```
Properties:
  Record Reader: AvroReader
  Record Writer: JsonRecordSetWriter
  Include Zero Record FlowFiles: false
```

**3. JoltTransformJSON** (Transformation)
```json
{
  "spec": [
    {
      "operation": "shift",
      "spec": {
        "id": "legacy_id",
        "first_name": "name.first",
        "last_name": "name.last",
        "email": "contact.email",
        "phone": "contact.phone",
        "created_at": "metadata.created_at"
      }
    },
    {
      "operation": "default",
      "spec": {
        "metadata": {
          "migrated_at": "${now():toNumber()}",
          "source": "postgresql"
        }
      }
    },
    {
      "operation": "modify-overwrite-beta",
      "spec": {
        "contact": {
          "email": "=toLower(@(1,email))",
          "phone": "=split('[^0-9]','',@(1,phone))"
        }
      }
    }
  ]
}
```

**4. PutMongo** (Target MongoDB)
```
Properties:
  Mongo URI: mongodb+srv://cluster.mongodb.net
  Mongo Database Name: production
  Mongo Collection Name: customers

  Mode: insert
  Upsert: true
  Update Query Key: legacy_id

  Write Concern: ACKNOWLEDGED
  Batch Size: 100

  Character Set: UTF-8
```

**5. Error Handling (UpdateAttribute + PutFile)**
```
On failure route:
  UpdateAttribute:
    - failure.timestamp: ${now()}
    - failure.reason: ${error.message}

  PutFile:
    - Directory: /data/nifi/failures
    - Conflict Resolution: replace
```

#### Avantages NiFi

- ✅ **Visual dataflow design** : Interface graphique intuitive
- ✅ **Backpressure handling** : Gestion automatique de la charge
- ✅ **Provenance tracking** : Traçabilité complète des données
- ✅ **Clusterable** : Scaling horizontal natif
- ✅ **Open-source** : Gratuit, communauté active
- ✅ **Connecteurs riches** : 300+ processors

---

## 📊 Comparatif Global des Outils

### Matrice de Décision

| Outil | Complexité | Performance | Coût | CDC | Transformation | Support |
|-------|-----------|-------------|------|-----|----------------|---------|
| **MongoDB Relational Migrator** | ⭐⭐ | ⭐⭐⭐ | Gratuit | ❌ | ⭐⭐ | MongoDB |
| **mongodump/restore** | ⭐ | ⭐⭐⭐⭐ | Gratuit | ❌ | ⭐ | MongoDB |
| **Debezium** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Gratuit | ✅ | ⭐⭐ | Community |
| **AWS DMS** | ⭐⭐⭐ | ⭐⭐⭐⭐ | €€ | ✅ | ⭐⭐⭐ | AWS |
| **Talend** | ⭐⭐⭐ | ⭐⭐⭐⭐ | €€€ | ⚠️ | ⭐⭐⭐⭐⭐ | Enterprise |
| **Apache NiFi** | ⭐⭐⭐ | ⭐⭐⭐⭐ | Gratuit | ⚠️ | ⭐⭐⭐⭐ | Community |
| **Airbyte** | ⭐⭐ | ⭐⭐⭐ | Gratuit/€€ | ✅ | ⭐⭐⭐ | Community/Enterprise |
| **Fivetran** | ⭐ | ⭐⭐⭐⭐ | €€€ | ✅ | ⭐⭐ | Enterprise |
| **Apache Spark** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | Gratuit/€€ | ❌ | ⭐⭐⭐⭐⭐ | Community |
| **Custom Scripts** | ⭐⭐⭐⭐ | Variable | Gratuit | ⚠️ | ⭐⭐⭐⭐⭐ | DIY |

### Guide de Sélection

#### Scénario 1 : Migration ponctuelle simple (<500 Go)
**Recommandation** : MongoDB Relational Migrator + mongodump/restore
- Gratuit
- Interface graphique
- Suggestions de modélisation
- Temps setup minimal

#### Scénario 2 : Migration avec zero downtime (To+)
**Recommandation** : Debezium + Kafka + MongoDB Sink
- CDC temps-réel
- Latence faible (<1 sec)
- Scalabilité élevée
- Open-source

**Alternative Enterprise** : AWS DMS (si AWS) ou Striim

#### Scénario 3 : Transformations complexes
**Recommandation** : Apache Spark + MongoDB Spark Connector
- Transformations avancées (ML, aggregations)
- Performance maximale (To/heure)
- Ecosystem riche

**Alternative avec UI** : Talend ou Pentaho

#### Scénario 4 : Budget limité, compétences techniques
**Recommandation** : Scripts Python/Node.js custom
- Contrôle total
- Pas de licence
- Optimisable à l'infini

#### Scénario 5 : SaaS, gestion simplifiée
**Recommandation** : Fivetran ou Airbyte Cloud
- Setup en minutes
- Maintenance minimale
- Support intégré

---

## 🎯 Recommandations Finales

### Critères de Choix Essentiels

1. **Volume de données**
   - <100 Go : Outils simples (Relational Migrator, mongoimport)
   - 100 Go - 1 To : ETL classiques (Talend, NiFi)
   - >1 To : Big Data (Spark, Debezium)

2. **Exigences de disponibilité**
   - Downtime acceptable : Migration batch
   - Zero downtime : CDC obligatoire

3. **Complexité transformations**
   - Simple (renaming, type conversion) : Relational Migrator, DMS
   - Complexe (business logic, ML) : Spark, Talend, Custom

4. **Budget**
   - Gratuit : Open-source (Debezium, NiFi, Spark)
   - €€ : AWS DMS, Airbyte Enterprise
   - €€€ : Talend, Informatica, Fivetran

5. **Compétences équipe**
   - Débutants : GUI tools (Relational Migrator, Talend)
   - Experts : Code-first (Spark, Custom scripts)

### Checklist Sélection Outil

- [ ] Volume de données estimé
- [ ] Exigences downtime
- [ ] Complexité transformations requises
- [ ] Budget disponible (licence + infra)
- [ ] Compétences équipe disponibles
- [ ] Support requis (community vs enterprise)
- [ ] Intégration écosystème existant (cloud, monitoring)
- [ ] Scalabilité future (croissance données)
- [ ] Conformité et sécurité (encryption, audit)

---

## 📚 Ressources et Documentation

### Documentation Officielle

**MongoDB**
- Relational Migrator: https://www.mongodb.com/products/relational-migrator
- Database Tools: https://www.mongodb.com/docs/database-tools/
- Kafka Connector: https://www.mongodb.com/docs/kafka-connector/

**CDC Tools**
- Debezium: https://debezium.io/documentation/
- AWS DMS: https://docs.aws.amazon.com/dms/

**ETL Platforms**
- Talend: https://www.talend.com/resources/
- Apache NiFi: https://nifi.apache.org/docs.html
- Airbyte: https://docs.airbyte.com/

### Communautés et Support

- MongoDB Community Forums: https://www.mongodb.com/community/forums/
- Stack Overflow: Tag `mongodb`
- Reddit: r/mongodb
- Debezium Mailing Lists: https://groups.google.com/g/debezium

---

**Prochaine section** : 19.3 Relational Migrator - Guide approfondi de l'outil officiel MongoDB avec cas d'usage détaillés et bonnes pratiques.

⏭️ [Relational Migrator](/19-migration-integration/03-relational-migrator.md)
