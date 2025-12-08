🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.12 Bonnes Pratiques de Sharding

## Introduction

Le sharding MongoDB est une technologie puissante qui permet de faire évoluer horizontalement votre base de données pour gérer des volumes massifs de données et de trafic. Cependant, sa complexité inhérente peut conduire à des erreurs coûteuses si les bonnes pratiques ne sont pas respectées dès la conception et tout au long du cycle de vie du cluster.

Cette section synthétise l'ensemble des meilleures pratiques accumulées au fil des chapitres précédents et ajoute des recommandations éprouvées en production. Elle constitue un guide de référence pour architectes, développeurs et administrateurs travaillant avec des clusters shardés MongoDB.

---

## Bonnes Pratiques de Conception

### 1. Choix de la Shard Key : Le Fondement Critique

La shard key est **la décision la plus importante** lors du déploiement d'un cluster shardé. Une mauvaise shard key peut rendre le cluster inefficace et difficile à corriger.

#### Critères d'Évaluation

| Critère | Description | Méthode de Validation |
|---------|-------------|----------------------|
| **Cardinalité élevée** | Nombre de valeurs distinctes > 10 000 | `db.collection.distinct("field").length` |
| **Distribution uniforme** | Pas de valeurs ultra-dominantes | Analyse avec agrégation $group |
| **Localité des requêtes** | Requêtes fréquentes incluent la shard key | Analyse des logs applicatifs |
| **Immutabilité** | Valeur ne change pas après insertion | Revue du modèle de données |
| **Monotonie évitée** | Pas de timestamps ou _id séquentiels seuls | Utiliser hashed ou compound |

#### Processus de Décision

```javascript
// Étape 1 : Identifier les candidats
// Analyser les patterns d'accès de l'application

// Étape 2 : Pour chaque candidat, exécuter
function evaluateShardKey(dbName, collName, candidateField) {
  print("=== Évaluation : " + candidateField + " ===\n");

  var coll = db.getSiblingDB(dbName)[collName];
  var totalDocs = coll.countDocuments({});

  // 1. Cardinalité
  var distinct = coll.distinct(candidateField).length;
  var cardinalityScore = distinct > 10000 ? 10 : distinct > 1000 ? 7 : distinct > 100 ? 4 : 1;
  print("Cardinalité : " + distinct + " valeurs distinctes (Score: " + cardinalityScore + "/10)");

  // 2. Distribution
  var topValues = coll.aggregate([
    { $group: { _id: "$" + candidateField, count: { $sum: 1 } } },
    { $sort: { count: -1 } },
    { $limit: 5 }
  ]).toArray();

  var maxPct = (topValues[0].count / totalDocs * 100).toFixed(2);
  var distributionScore = maxPct < 10 ? 10 : maxPct < 25 ? 7 : maxPct < 50 ? 4 : 1;
  print("Distribution : Top valeur = " + maxPct + "% (Score: " + distributionScore + "/10)");

  // 3. Nullité
  var nullCount = coll.countDocuments({ [candidateField]: null });
  var nullPct = (nullCount / totalDocs * 100).toFixed(2);
  var nullScore = nullPct < 1 ? 10 : nullPct < 5 ? 7 : nullPct < 20 ? 4 : 1;
  print("Nullité : " + nullPct + "% nulls (Score: " + nullScore + "/10)");

  // Score final
  var totalScore = (cardinalityScore + distributionScore + nullScore) / 3;
  print("\n📊 Score Total : " + totalScore.toFixed(1) + "/10");

  if (totalScore >= 8) {
    print("✅ Excellente shard key candidate");
  } else if (totalScore >= 6) {
    print("⚠️  Acceptable, mais considérer une compound key");
  } else {
    print("❌ Non recommandée, chercher alternative");
  }

  print("\n" + "=".repeat(50) + "\n");

  return totalScore;
}

// Exemple d'utilisation
evaluateShardKey("mydb", "orders", "customer_id");
evaluateShardKey("mydb", "orders", "status");
evaluateShardKey("mydb", "orders", "order_date");
```

#### Décisions par Type d'Application

```yaml
e_commerce:
  collections:
    orders:
      recommended: { customer_id: 1, order_date: 1 }
      rationale: "Localité par client, évite monotonie"

    products:
      recommended: { category: 1, product_id: 1 }
      rationale: "Distribution par catégorie, granularité par produit"

    sessions:
      recommended: { session_id: "hashed" }
      rationale: "Distribution uniforme garantie"

saas_multi_tenant:
  collections:
    documents:
      recommended: { tenant_id: 1, document_id: 1 }
      rationale: "Isolation par tenant, évite jumbo chunks gros clients"

    events:
      recommended: { tenant_id: "hashed", timestamp: 1 }
      rationale: "Distribution uniforme même pour gros tenants"

iot_time_series:
  collections:
    sensor_data:
      recommended: { sensor_id: "hashed", timestamp: 1 }
      rationale: "Évite hot spots sur timestamps récents"

    metrics:
      recommended: { metric_name: 1, timestamp: 1 }
      rationale: "Regroupement par métrique, ordre temporel"

social_media:
  collections:
    posts:
      recommended: { author_id: "hashed" }
      rationale: "Distribution uniforme des utilisateurs populaires"

    messages:
      recommended: { conversation_id: 1, timestamp: 1 }
      rationale: "Localité des conversations"

logs_analytics:
  collections:
    logs:
      recommended: { application: 1, timestamp: 1 }
      rationale: "Isolation par application, ordre temporel"

    events:
      recommended: { event_type: "hashed", timestamp: 1 }
      rationale: "Distribution uniforme par type"
```

### 2. Modélisation des Données pour le Sharding

#### Principes de Modélisation

```javascript
// ✅ BON : Documents auto-suffisants
{
  "_id": ObjectId("..."),
  "order_id": "ORD12345",
  "customer": {
    "id": "CUST789",
    "name": "Alice Dupont",
    "email": "alice@example.com"
  },
  "items": [
    {
      "product_id": "PROD001",
      "name": "Laptop",
      "quantity": 1,
      "price": 999.99
    }
  ],
  "total_amount": 999.99,
  "created_at": ISODate("2024-01-15T10:30:00Z")
}
// Requête ciblée : find({ order_id: "ORD12345" })
// → 1 seul document, 1 seul shard

// ❌ MAUVAIS : Documents nécessitant $lookup
{
  "_id": ObjectId("..."),
  "order_id": "ORD12345",
  "customer_id": "CUST789",  // Référence
  "items": ["PROD001"]       // Références
}
// Requête : find({ order_id: "ORD12345" }) + $lookup customers + $lookup products
// → Plusieurs shards, $lookup coûteux
```

#### Stratégies de Dénormalisation

```javascript
// Pattern Extended Reference : Embarquer les données fréquemment accédées

// Au lieu de :
// Collection orders : { customer_id: "CUST123" }
// Collection customers : { _id: "CUST123", name: "...", email: "...", ... }

// Préférer :
{
  "order_id": "ORD12345",
  "customer": {
    "id": "CUST123",
    "name": "Alice Dupont",      // Données fréquentes
    "email": "alice@example.com", // Données fréquentes
    // "full_address": "..."      // Données rares → pas incluses
  }
}

// Mise à jour : Si customer.name change
// Option 1 : Accepter l'incohérence temporaire (eventually consistent)
// Option 2 : Mettre à jour tous les orders (coûteux mais cohérent)
// Option 3 : Hybrid : n'embarquer que les données immuables

// Décision dépend du cas d'usage
```

### 3. Taille des Documents et des Chunks

#### Limites et Recommandations

| Élément | Limite MongoDB | Recommandation | Rationale |
|---------|---------------|---------------|-----------|
| **Taille document** | 16 MB (hard limit) | < 1 MB | Performance et flexibilité |
| **Taille chunk** | Configurable | 64-128 MB | Équilibre migration/métadonnées |
| **Documents par chunk** | Aucune limite | 50K-500K | Dépend taille moyenne documents |
| **Chunks par shard** | Aucune limite | 100-1000 | Équilibre distribution/overhead |

#### Configuration de la Taille des Chunks

```javascript
// Déterminer la taille optimale selon le cas d'usage

// Petits documents (< 10 KB), haute fréquence d'insertion
// → Chunks plus gros (128 MB)
db.getSiblingDB("config").settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 128 } },
  { upsert: true }
)

// Gros documents (> 100 KB), faible cardinalité shard key
// → Chunks plus petits (32 MB)
db.getSiblingDB("config").settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 32 } },
  { upsert: true }
)

// Standard (documents moyens 1-50 KB)
// → Défaut (64 MB) convient
```

---

## Bonnes Pratiques de Déploiement

### 1. Topologie Recommandée

#### Environnement de Production Minimal

```
Composants Minimums (Haute Disponibilité) :
┌──────────────────────────────────────────────────┐
│ 3 Config Servers (Replica Set 3 membres)         │
│ 2+ Shards (chacun Replica Set 3 membres)         │
│ 2+ Mongos (load balanced)                        │
└──────────────────────────────────────────────────┘

Total Minimum : 12 serveurs
- 3 config servers
- 6 shard members (2 shards × 3)
- 3 mongos

Recommandation Production : 15-20 serveurs
- 3 config servers
- 9-12 shard members (3-4 shards × 3)
- 3-5 mongos
```

#### Distribution Multi-Datacenter

```javascript
// Configuration avec 3 datacenters pour haute disponibilité

// Stratégie 1 : Priorités d'élection
// DC1 (primary), DC2 (secondary), DC3 (disaster recovery)

// Config Servers
configReplSet = {
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "cfg1-dc1:27019", priority: 3 },  // DC1
    { _id: 1, host: "cfg2-dc2:27019", priority: 2 },  // DC2
    { _id: 2, host: "cfg3-dc3:27019", priority: 1 }   // DC3
  ]
}

// Shard A : Primary préféré DC1
shardA = {
  _id: "shardA",
  members: [
    { _id: 0, host: "shardA1-dc1:27018", priority: 3 },
    { _id: 1, host: "shardA2-dc2:27018", priority: 2 },
    { _id: 2, host: "shardA3-dc3:27018", priority: 1 }
  ]
}

// Shard B : Primary préféré DC2 (distribution de charge)
shardB = {
  _id: "shardB",
  members: [
    { _id: 0, host: "shardB1-dc1:27018", priority: 2 },
    { _id: 1, host: "shardB2-dc2:27018", priority: 3 },
    { _id: 2, host: "shardB3-dc3:27018", priority: 1 }
  ]
}

// Mongos : Un ou plusieurs par datacenter
// Application DC1 → mongos DC1
// Application DC2 → mongos DC2
```

### 2. Dimensionnement Matériel

#### Calcul des Ressources par Composant

```javascript
// Formules de dimensionnement

// === CONFIG SERVERS ===
// Besoins minimaux (métadonnées uniquement)
configServerResources = {
  cpu: "2-4 cores",
  ram: "8-16 GB",
  storage: {
    size: "50-100 GB SSD",
    iops: "1000+",
    calculation: function(numCollections, numChunks) {
      // ~1 KB par chunk en métadonnées
      var metadataGB = (numChunks * 1024) / (1024 * 1024 * 1024);
      var withOverhead = metadataGB * 2;  // 2x pour overhead
      return Math.max(50, withOverhead);
    }
  }
};

// === SHARDS ===
// Dépend du working set
shardResources = {
  cpu: "8-32 cores",
  ram: function(workingSetGB) {
    // Formule : RAM >= 1.5 × working set (pour cache WiredTiger)
    return Math.max(32, workingSetGB * 1.5);
  },
  storage: {
    size: function(dataGB, compressionRatio) {
      // Formule : Storage = (Data / compression ratio) × 1.5 (overhead)
      // WiredTiger compression ~3-5x pour documents texte
      var compressed = dataGB / compressionRatio;
      return compressed * 1.5;
    },
    iops: "5000-10000+ (SSD/NVMe recommandé)",
    type: "SSD minimum, NVMe pour haute performance"
  }
};

// === MONGOS ===
// Léger (pas de stockage)
mongosResources = {
  cpu: "4-8 cores",
  ram: "8-16 GB",
  storage: "20 GB (logs uniquement)",
  network: "10 Gbps (goulot potentiel)"
};

// Exemple de calcul pour un cluster
function calculateClusterResources(config) {
  print("=== Dimensionnement du Cluster ===\n");

  var totalDataGB = config.totalDataGB;
  var compressionRatio = config.compressionRatio || 3;
  var workingSetPct = config.workingSetPct || 0.3;  // 30% du dataset actif
  var numShards = config.numShards;

  // Working set par shard
  var workingSetPerShardGB = (totalDataGB * workingSetPct) / numShards;

  // RAM par shard
  var ramPerShardGB = Math.max(32, workingSetPerShardGB * 1.5);

  // Stockage par shard
  var compressedDataGB = totalDataGB / compressionRatio;
  var storagePerShardGB = (compressedDataGB / numShards) * 1.5;

  print("Configuration :");
  print("  Données totales : " + totalDataGB + " GB");
  print("  Nombre de shards : " + numShards);
  print("  Working set : " + (workingSetPct * 100) + "%");
  print("");

  print("Ressources par Shard :");
  print("  CPU : 16+ cores");
  print("  RAM : " + Math.ceil(ramPerShardGB) + " GB");
  print("  Stockage : " + Math.ceil(storagePerShardGB) + " GB SSD/NVMe");
  print("");

  print("Ressources Config Servers (3 membres) :");
  print("  CPU : 4 cores");
  print("  RAM : 16 GB");
  print("  Stockage : 100 GB SSD");
  print("");

  print("Ressources Mongos (3 instances) :");
  print("  CPU : 8 cores");
  print("  RAM : 16 GB");
  print("  Stockage : 20 GB");
}

// Exemple : Cluster avec 10 TB de données
calculateClusterResources({
  totalDataGB: 10000,
  compressionRatio: 3,
  workingSetPct: 0.3,
  numShards: 5
});
```

### 3. Pré-splitting et Distribution Initiale

#### Stratégie de Pré-splitting

```javascript
// TOUJOURS pré-splitter avant import massif

function presplitForImport(namespace, shardKey, config) {
  print("=== Pré-splitting pour Import ===\n");

  var numShards = db.getSiblingDB("config").shards.count();
  var targetChunksPerShard = config.targetChunksPerShard || 4;
  var totalChunks = numShards * targetChunksPerShard;

  print("Shards : " + numShards);
  print("Chunks par shard : " + targetChunksPerShard);
  print("Total chunks : " + totalChunks);
  print("");

  // Activer sharding
  var dbName = namespace.split(".")[0];
  var collName = namespace.split(".")[1];

  sh.enableSharding(dbName);

  // Pour hashed shard key
  if (Object.values(shardKey)[0] === "hashed") {
    sh.shardCollection(namespace, shardKey);

    // MongoDB crée automatiquement 2 chunks par shard avec hashed
    // Pour plus de granularité :
    for (var i = 0; i < totalChunks; i++) {
      try {
        sh.splitAt(namespace, { [Object.keys(shardKey)[0]]: "split_" + i });
      } catch (e) {
        // Ignore les erreurs de split (normal avec hashed)
      }
    }
  }
  // Pour range shard key
  else {
    sh.shardCollection(namespace, shardKey);

    // Créer les split points basés sur les données existantes ou estimées
    var shardKeyField = Object.keys(shardKey)[0];

    // Si données existantes, utiliser percentiles
    if (config.existingData) {
      var coll = db.getSiblingDB(dbName)[collName];
      var total = coll.countDocuments({});
      var step = Math.floor(total / totalChunks);

      for (var i = 1; i < totalChunks; i++) {
        var doc = coll.find().sort({ [shardKeyField]: 1 }).skip(i * step).limit(1).toArray()[0];
        if (doc) {
          sh.splitAt(namespace, { [shardKeyField]: doc[shardKeyField] });
        }
      }
    }
    // Sinon, splits uniforms sur la plage attendue
    else if (config.range) {
      var min = config.range.min;
      var max = config.range.max;
      var step = (max - min) / totalChunks;

      for (var i = 1; i < totalChunks; i++) {
        sh.splitAt(namespace, { [shardKeyField]: min + (i * step) });
      }
    }
  }

  print("✅ Pré-splitting terminé");

  // Distribuer les chunks initialement
  var shards = db.getSiblingDB("config").shards.find().toArray();
  var chunks = db.getSiblingDB("config").chunks.find({ ns: namespace }).sort({ min: 1 }).toArray();

  print("\nDistribution des " + chunks.length + " chunks sur " + shards.length + " shards...");

  for (var i = 0; i < chunks.length; i++) {
    var targetShard = shards[i % shards.length]._id;

    try {
      sh.moveChunk(namespace, chunks[i].min, targetShard);
      print("  Chunk " + (i+1) + " → " + targetShard);
    } catch (e) {
      // Ignore si déjà sur le bon shard
    }
  }

  print("\n✅ Distribution initiale terminée");
}

// Exemple : Import de 100M de commandes
presplitForImport("ecommerce.orders", { customer_id: 1 }, {
  targetChunksPerShard: 5,
  existingData: false,
  range: { min: 0, max: 100000000 }
});
```

---

## Bonnes Pratiques Opérationnelles

### 1. Monitoring et Alerting

#### Dashboard KPI Essentiels

```yaml
metriques_critiques:
  cluster_health:
    - metric: "mongodb_up"
      alert_threshold: "< 1 for 1m"
      severity: "critical"

    - metric: "config_server_majority_available"
      alert_threshold: "< 2 for 30s"
      severity: "critical"

  distribution:
    - metric: "chunk_imbalance_percentage"
      alert_threshold: "> 20% for 4h"
      severity: "warning"

    - metric: "jumbo_chunks_count"
      alert_threshold: "> 0 for 1h"
      severity: "warning"

  balancer:
    - metric: "migration_failure_rate_24h"
      alert_threshold: "> 10% for 30m"
      severity: "warning"

    - metric: "migration_duration_p95"
      alert_threshold: "> 600s for 1h"
      severity: "warning"

  performance:
    - metric: "query_latency_p99"
      alert_threshold: "> 500ms for 5m"
      severity: "warning"

    - metric: "wiredtiger_cache_usage"
      alert_threshold: "> 90% for 10m"
      severity: "warning"

    - metric: "replication_lag_max"
      alert_threshold: "> 30s for 5m"
      severity: "critical"

  capacity:
    - metric: "disk_usage_percentage"
      alert_threshold: "> 80% for 1h"
      severity: "warning"

    - metric: "connection_usage_percentage"
      alert_threshold: "> 80% for 10m"
      severity: "warning"
```

#### Script de Health Check Quotidien

```javascript
// health-check-daily.js
// À exécuter chaque matin pour validation du cluster

function dailyHealthCheck() {
  var report = {
    date: new Date(),
    cluster: "production",
    status: "healthy",
    warnings: [],
    errors: []
  };

  print("=== Health Check Quotidien ===");
  print("Date : " + report.date);
  print("");

  // 1. Santé du cluster
  try {
    var shards = db.getSiblingDB("config").shards.find().toArray();
    print("✅ Shards : " + shards.length + " actifs");

    shards.forEach(function(shard) {
      try {
        var shardConn = new Mongo(shard.host.split("/")[1].split(",")[0]);
        shardConn.getDB("admin").ping();
      } catch (e) {
        report.errors.push("Shard " + shard._id + " inaccessible");
        report.status = "critical";
      }
    });
  } catch (e) {
    report.errors.push("Impossible de vérifier les shards : " + e.message);
    report.status = "critical";
  }

  // 2. Distribution des chunks
  var jumboCount = db.getSiblingDB("config").chunks.countDocuments({ jumbo: true });

  if (jumboCount > 0) {
    report.warnings.push(jumboCount + " jumbo chunks détectés");
    if (report.status === "healthy") report.status = "warning";
  }

  print(jumboCount === 0 ? "✅" : "⚠️  " + " Jumbo chunks : " + jumboCount);

  // 3. État du balancer
  var balancerEnabled = sh.getBalancerState();
  var balancerRunning = sh.isBalancerRunning();

  print("✅ Balancer : " + (balancerEnabled ? "Activé" : "Désactivé"));

  // 4. Migrations récentes
  var last24h = new Date(Date.now() - 86400000);
  var migrations = db.getSiblingDB("config").changelog.aggregate([
    { $match: {
        time: { $gte: last24h },
        what: { $in: ["moveChunk.commit", "moveChunk.error"] }
      }
    },
    { $group: { _id: "$what", count: { $sum: 1 } } }
  ]).toArray();

  var committed = 0, errors = 0;
  migrations.forEach(function(m) {
    if (m._id === "moveChunk.commit") committed = m.count;
    if (m._id === "moveChunk.error") errors = m.count;
  });

  if (errors > committed * 0.1) {
    report.warnings.push("Taux d'échec des migrations élevé : " + (errors / (committed + errors) * 100).toFixed(1) + "%");
    if (report.status === "healthy") report.status = "warning";
  }

  print((errors === 0 ? "✅" : "⚠️  ") + " Migrations (24h) : " + committed + " succès, " + errors + " échecs");

  // 5. Espace disque
  shards.forEach(function(shard) {
    try {
      var shardConn = new Mongo(shard.host.split("/")[1].split(",")[0]);
      var dbStats = shardConn.getDB("admin").adminCommand({ listDatabases: 1 });

      var totalSizeGB = dbStats.databases.reduce((sum, db) => sum + (db.sizeOnDisk || 0), 0) / (1024 * 1024 * 1024);

      if (totalSizeGB > 500) {  // Seuil configurable
        report.warnings.push("Shard " + shard._id + " utilise " + totalSizeGB.toFixed(0) + " GB");
      }

    } catch (e) {
      // Ignorer si inaccessible (déjà signalé)
    }
  });

  // Résumé
  print("");
  print("=== Résumé ===");
  print("Statut : " + report.status.toUpperCase());

  if (report.errors.length > 0) {
    print("\n🔴 ERREURS :");
    report.errors.forEach(e => print("  - " + e));
  }

  if (report.warnings.length > 0) {
    print("\n⚠️  AVERTISSEMENTS :");
    report.warnings.forEach(w => print("  - " + w));
  }

  if (report.status === "healthy") {
    print("\n✅ Cluster en bonne santé");
  }

  // Enregistrer le rapport
  db.getSiblingDB("admin").health_reports.insertOne(report);

  return report;
}

// Exécuter
dailyHealthCheck();
```

### 2. Backup et Disaster Recovery

#### Stratégie de Backup 3-2-1

```yaml
strategy_3_2_1:
  # 3 copies des données
  copies:
    - location: "Production cluster"
      type: "Live data"
      retention: "N/A"

    - location: "Backup storage (local)"
      type: "Daily snapshots"
      retention: "30 days"

    - location: "Backup storage (off-site)"
      type: "Weekly snapshots"
      retention: "1 year"

  # 2 types de média
  media:
    - type: "Disk (SAN/NAS)"
      usage: "Daily backups, fast restore"

    - type: "Cloud Object Storage (S3/GCS)"
      usage: "Long-term retention, DR"

  # 1 copie off-site
  offsite:
    location: "Different region/datacenter"
    sync: "Automated"
    test_frequency: "Monthly"

implementation:
  daily_backup:
    time: "02:00 UTC"
    method: "Snapshot or mongodump"
    script: "/scripts/backup-cluster.sh"

  weekly_backup:
    time: "Sunday 01:00 UTC"
    method: "Full snapshot"
    upload: "S3 bucket (encrypted)"

  monthly_restore_test:
    environment: "Staging cluster"
    validation: "Data integrity + query tests"
    documentation: "Test report required"
```

### 3. Maintenance Windows

#### Planification des Maintenances

```javascript
// Définir des fenêtres de maintenance standard

// 1. Fenêtre de balancing (quotidienne)
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "02:00",  // Heures creuses
        stop: "06:00"
      }
    }
  },
  { upsert: true }
)

// 2. Fenêtre de maintenance planifiée (mensuelle)
var maintenanceSchedule = {
  frequency: "monthly",
  dayOfWeek: "Sunday",  // Dimanche
  weekOfMonth: "first",  // Premier dimanche du mois
  startTime: "01:00 UTC",
  duration: "4 hours",
  activities: [
    "Rolling restart for patches",
    "Preventive maintenance script",
    "Backup verification",
    "Performance review"
  ],
  notification: {
    advance: "7 days",
    channels: ["email", "slack", "status-page"]
  }
};

// 3. Communication template
var maintenanceNotification = `
📅 MAINTENANCE PLANIFIÉE

Date : ${maintenanceSchedule.dayOfWeek} ${maintenanceSchedule.weekOfMonth} [DATE]
Heure : ${maintenanceSchedule.startTime} (durée estimée: ${maintenanceSchedule.duration})

Impact attendu : Aucun (rolling maintenance)

Activités :
${maintenanceSchedule.activities.map(a => "- " + a).join("\n")}

En cas de problème : Contact équipe DBA
`;
```

---

## Bonnes Pratiques de Performance

### 1. Optimisation des Requêtes

#### Patterns de Requêtes Efficaces

```javascript
// ✅ BON : Requête ciblée incluant shard key
db.orders.find({
  customer_id: "CUST12345",  // Shard key
  status: "pending"
})
.explain("executionStats")

// Résultat :
// - winningPlan.stage: "SINGLE_SHARD"
// - nReturned: 5
// - executionTimeMillis: 10ms

// ❌ MAUVAIS : Requête sans shard key
db.orders.find({
  status: "pending"  // Pas de shard key
})
.explain("executionStats")

// Résultat :
// - winningPlan.stage: "SHARD_MERGE"
// - nReturned: 5000
// - executionTimeMillis: 250ms (25x plus lent)

// 🔧 SOLUTION : Remodeler la requête ou créer index secondaire
db.orders.createIndex({ status: 1, created_at: -1 })

// Accepter le broadcast mais optimiser avec index
db.orders.find({ status: "pending" })
  .sort({ created_at: -1 })
  .limit(100)
// Index réduit les documents examinés sur chaque shard
```

#### Agrégations Optimisées

```javascript
// Pattern d'optimisation : Push down aggregations

// ❌ Inefficace : Sort global sans limite
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $sort: { total_amount: -1 } },  // Sort global coûteux
  { $project: { customer_id: 1, total_amount: 1 } }
])

// ✅ Efficace : Sort + Limit (permet push-down)
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $sort: { total_amount: -1 } },
  { $limit: 100 },  // Limite appliquée sur chaque shard
  { $project: { customer_id: 1, total_amount: 1 } }
])

// ✅ Encore mieux : Match avec shard key
db.orders.aggregate([
  { $match: {
      customer_id: { $in: listOfCustomers },  // Shard key
      status: "completed"
    }
  },
  { $sort: { total_amount: -1 } },
  { $limit: 10 }
])
```

### 2. Stratégies d'Indexation

#### Index Strategy Matrix

| Scénario | Type d'Index | Justification |
|----------|-------------|---------------|
| Shard key seule | Automatique | Créé par MongoDB lors du sharding |
| Champs fréquents dans filtres | Single field | Pour requêtes ciblées |
| Requêtes multi-champs | Compound | Ordre selon sélectivité |
| Arrays (tags, categories) | Multikey | Automatique sur arrays |
| Recherche textuelle | Text | Full-text search |
| Géolocalisation | 2dsphere | Queries géospatiales |
| Unicité globale | Unique + shard key | Include shard key pour unicité |
| TTL (logs, sessions) | TTL | Suppression automatique |

#### Checklist d'Index par Collection

```javascript
// Template de stratégie d'indexation

function defineIndexStrategy(collName, config) {
  print("=== Stratégie d'Index : " + collName + " ===\n");

  var strategy = {
    collection: collName,
    shardKey: config.shardKey,
    indexes: []
  };

  // 1. Shard key index (automatique)
  strategy.indexes.push({
    name: "Shard Key (auto)",
    fields: config.shardKey,
    type: "automatic",
    purpose: "Sharding routing"
  });

  // 2. Index pour requêtes fréquentes
  config.commonQueries.forEach(function(query) {
    strategy.indexes.push({
      name: "Query: " + JSON.stringify(query.filter),
      fields: query.filter,
      type: query.unique ? "unique" : "standard",
      purpose: "Query performance",
      estimatedUsage: query.frequencyPerDay + " queries/day"
    });
  });

  // 3. Index pour agrégations
  if (config.aggregations) {
    config.aggregations.forEach(function(agg) {
      strategy.indexes.push({
        name: "Aggregation: " + agg.name,
        fields: agg.fields,
        type: "compound",
        purpose: "Aggregation pipeline",
        estimatedUsage: agg.frequencyPerDay + " pipelines/day"
      });
    });
  }

  // 4. TTL index si applicable
  if (config.ttl) {
    strategy.indexes.push({
      name: "TTL: " + config.ttl.field,
      fields: { [config.ttl.field]: 1 },
      type: "TTL",
      purpose: "Automatic cleanup",
      expireAfterSeconds: config.ttl.seconds
    });
  }

  // Afficher la stratégie
  print("Shard Key : " + JSON.stringify(strategy.shardKey));
  print("\nIndex proposés :");

  strategy.indexes.forEach(function(idx, i) {
    print("\n" + (i+1) + ". " + idx.name);
    print("   Fields : " + JSON.stringify(idx.fields));
    print("   Type : " + idx.type);
    print("   Purpose : " + idx.purpose);
    if (idx.estimatedUsage) print("   Usage : " + idx.estimatedUsage);
    if (idx.expireAfterSeconds) print("   TTL : " + idx.expireAfterSeconds + "s");
  });

  print("\n" + "=".repeat(60));

  return strategy;
}

// Exemple : Collection orders
defineIndexStrategy("orders", {
  shardKey: { customer_id: 1, order_date: 1 },
  commonQueries: [
    {
      filter: { customer_id: 1, status: 1 },
      frequencyPerDay: 50000,
      unique: false
    },
    {
      filter: { order_id: 1 },
      frequencyPerDay: 100000,
      unique: true
    }
  ],
  aggregations: [
    {
      name: "Daily sales by customer",
      fields: { customer_id: 1, order_date: -1 },
      frequencyPerDay: 1000
    }
  ],
  ttl: null
});
```

---

## Bonnes Pratiques de Sécurité

### 1. Defense in Depth

```yaml
security_layers:

  layer_1_network:
    - Firewalls: "Restrict to application IPs only"
    - VPC/Private Network: "No public internet access"
    - TLS/SSL: "Encrypted connections (certificates)"
    - IP Whitelisting: "Mongos and internal IPs only"

  layer_2_authentication:
    - Method: "SCRAM-SHA-256 (minimum)"
    - x.509: "For inter-cluster communication"
    - Keyfile: "For replica set and config servers"
    - Password Policy: "Strong passwords, rotation every 90 days"

  layer_3_authorization:
    - RBAC: "Role-Based Access Control"
    - Principle of Least Privilege: "Minimum permissions"
    - Application Accounts: "Read-only when possible"
    - Admin Accounts: "MFA required"

  layer_4_encryption:
    - At Rest: "Encrypted storage volumes (LUKS, BitLocker)"
    - In Transit: "TLS 1.2+ only"
    - Client-Side: "Field Level Encryption (FLE) for PII"
    - Key Management: "External KMS (AWS KMS, Azure Key Vault)"

  layer_5_auditing:
    - Audit Log: "Enabled for all DDL and authentication"
    - Retention: "90 days minimum"
    - SIEM Integration: "Real-time alerts on anomalies"
    - Regular Review: "Weekly audit log analysis"
```

### 2. Configuration Sécurisée

```javascript
// Template de configuration sécurisée

// 1. Création des utilisateurs avec rôles appropriés
use admin

// Admin cluster (équipe DBA uniquement)
db.createUser({
  user: "clusterAdmin",
  pwd: passwordPrompt(),  // Ne jamais mettre en clair
  roles: [
    { role: "clusterAdmin", db: "admin" },
    { role: "userAdminAnyDatabase", db: "admin" }
  ]
})

// Backup user (pour mongodump)
db.createUser({
  user: "backupUser",
  pwd: passwordPrompt(),
  roles: [
    { role: "backup", db: "admin" },
    { role: "restore", db: "admin" }
  ]
})

// Application user (lecture/écriture sur DB spécifique)
use mydb
db.createUser({
  user: "appUser",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "mydb" }
  ]
})

// Monitoring user (lecture seule, métriques)
use admin
db.createUser({
  user: "monitoringUser",
  pwd: passwordPrompt(),
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})

// 2. Activer l'audit
// Dans mongod.conf :
```

```yaml
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{
    atype: { $in: ["authenticate", "createUser", "dropUser", "dropDatabase", "dropCollection", "createIndex", "dropIndex"] }
  }'
```

---

## Anti-Patterns Récapitulatifs

### Top 10 des Erreurs à Éviter

| # | Anti-Pattern | Impact | Solution |
|---|--------------|--------|----------|
| 1 | Shard key monotone (timestamp, _id) | Hot spots, jumbo chunks | Hashed ou compound avec préfixe non-monotone |
| 2 | Shard key de faible cardinalité | Jumbo chunks, déséquilibre | Cardinalité > 10K, ou compound |
| 3 | Pas de pré-splitting avant import | Distribution déséquilibrée initiale | Pré-splitter en 4-10 chunks par shard |
| 4 | Requêtes sans shard key | Broadcast coûteux | Inclure shard key ou accepter + optimiser |
| 5 | Ignorer les jumbo chunks | Déséquilibre croissant | Traiter immédiatement (refine/reshape) |
| 6 | Config servers non-redondants | Point de défaillance unique | Toujours 3 config servers minimum |
| 7 | Pas de monitoring du balancer | Migrations en heures de pointe | Fenêtres de balancing + alerting |
| 8 | $lookup intensif | Performance dégradée | Dénormaliser les données fréquentes |
| 9 | Sharder trop tôt | Complexité inutile | Attendre 100+ GB ou 10K+ ops/sec |
| 10 | Pas de tests en staging | Surprises en production | Environnement staging identique |

---

## Checklist de Production

### Avant le Go-Live

```yaml
architecture:
  - [ ] Topologie définie et documentée
  - [ ] Shard key validée (cardinalité, distribution, localité)
  - [ ] Pré-splitting planifié
  - [ ] Sizing matériel vérifié (CPU, RAM, Storage)
  - [ ] Tests de charge effectués en staging

sécurité:
  - [ ] Authentification activée (SCRAM-SHA-256+)
  - [ ] Keyfile déployé sur tous les composants
  - [ ] Utilisateurs créés avec principe du moindre privilège
  - [ ] TLS/SSL activé pour toutes les connexions
  - [ ] Audit log configuré et testé
  - [ ] Firewall rules appliquées

monitoring:
  - [ ] Prometheus + Grafana déployés
  - [ ] Alertes configurées (critical + warning)
  - [ ] Dashboard cluster opérationnel
  - [ ] PagerDuty / on-call intégré
  - [ ] Runbooks documentés

backup:
  - [ ] Stratégie de backup définie (3-2-1)
  - [ ] Scripts de backup automatisés
  - [ ] Test de restauration effectué
  - [ ] Backup off-site configuré
  - [ ] Documentation de DR à jour

opérations:
  - [ ] Fenêtre de balancing définie
  - [ ] Procédure de rolling restart documentée
  - [ ] Plan de scaling (vertical et horizontal)
  - [ ] Health check script déployé
  - [ ] Équipe formée et on-call défini

application:
  - [ ] Connection strings avec failover
  - [ ] Read/Write concerns appropriés
  - [ ] Retry logic implémenté
  - [ ] Timeout configurés
  - [ ] Monitoring APM activé
```

### Post Go-Live (première semaine)

```yaml
jour_1:
  - Monitoring continu 24/7
  - Vérifier distribution des chunks
  - Analyser latence des requêtes
  - Valider comportement du balancer

jour_3:
  - Review des alertes déclenchées
  - Analyse des slow queries
  - Vérification de la croissance des données
  - Ajustements de configuration si nécessaire

jour_7:
  - Post-mortem de la mise en production
  - Documentation des leçons apprises
  - Optimisations identifiées
  - Plan d'amélioration continue
```

---

## Recommandations par Cas d'Usage

### E-commerce

```yaml
shard_keys:
  orders: { customer_id: 1, order_date: 1 }
  products: { category: 1, product_id: 1 }
  cart: { session_id: "hashed" }
  reviews: { product_id: 1, created_at: 1 }

sizing:
  shards_minimum: 3
  growth_projection: "Plan for 3x growth first year"

特points_clés:
  - Pics de charge prévisibles (Black Friday, promotions)
  - Considérer read preference secondary pour reporting
  - TTL pour sessions expirées
  - Monitoring spécifique sur latence checkout
```

### SaaS Multi-Tenant

```yaml
shard_keys:
  documents: { tenant_id: 1, document_id: 1 }
  users: { tenant_id: 1, user_id: 1 }
  events: { tenant_id: "hashed", timestamp: 1 }

considerations:
  - Isolation par tenant garantie
  - Attention aux "whale customers" (gros clients)
  - Zone sharding potentiel pour compliance (GDPR)
  - Metrics par tenant pour billing

best_practices:
  - Alertes sur déséquilibre par tenant
  - Possibilité de shard dédié pour gros clients
  - Rate limiting par tenant au niveau applicatif
```

### IoT / Time Series

```yaml
shard_keys:
  sensor_data: { sensor_id: "hashed", timestamp: 1 }
  aggregates: { metric_name: 1, interval: 1 }

特considerations:
  - Volume massif d'écritures
  - TTL agressif sur données brutes (7-30 jours)
  - Pré-agrégation (hourly, daily) dans collections séparées
  - Compression importante (WiredTiger snappy)

optimizations:
  - Batch inserts (1000+ documents)
  - Write concern w:1 pour throughput
  - Read preference secondary pour analytics
  - Capped collections pour last-N-values
```

### Analytics / Logs

```yaml
shard_keys:
  logs: { application: 1, timestamp: 1 }
  events: { event_type: "hashed", timestamp: 1 }

特considerations:
  - Ingestion très haute (100K+ ops/sec)
  - Requêtes analytiques lourdes
  - Rétention définie (30-90 jours)

best_practices:
  - Shards dédiés aux écritures vs lectures (zone sharding)
  - allowDiskUse pour agrégations volumineuses
  - Index partiel sur champs rares
  - Considérer Time Series Collections (MongoDB 5.0+)
```

---

## Évolution et Scaling

### Quand Ajouter un Shard ?

```javascript
// Critères de décision pour scaling horizontal

function shouldAddShard(metrics) {
  var reasons = [];

  // 1. Utilisation du stockage
  if (metrics.avgStorageUsagePercent > 70) {
    reasons.push("Stockage > 70% sur average");
  }

  // 2. Charge CPU
  if (metrics.avgCpuPercent > 70) {
    reasons.push("CPU > 70% sur average");
  }

  // 3. Cache pressure
  if (metrics.wiredTigerCacheUsage > 90) {
    reasons.push("WiredTiger cache > 90%");
  }

  // 4. Distribution déséquilibrée
  if (metrics.chunkImbalance > 30) {
    reasons.push("Déséquilibre chunks > 30%");
  }

  // 5. Latence queries
  if (metrics.queryLatencyP95 > 200) {
    reasons.push("P95 latency > 200ms");
  }

  // Décision
  if (reasons.length >= 2) {
    print("⚠️  RECOMMANDATION : Ajouter un shard");
    print("\nRaisons :");
    reasons.forEach(r => print("  - " + r));
    print("\nActions :");
    print("  1. Provisionner nouveau shard (Replica Set)");
    print("  2. Ajouter au cluster : sh.addShard(...)");
    print("  3. Laisser le balancer redistribuer (plusieurs jours)");
    print("  4. Monitorer les migrations");
    return true;
  } else {
    print("✅ Capacité actuelle suffisante");
    return false;
  }
}

// Exemple
shouldAddShard({
  avgStorageUsagePercent: 75,
  avgCpuPercent: 65,
  wiredTigerCacheUsage: 85,
  chunkImbalance: 15,
  queryLatencyP95: 180
});
```

### Migration vers MongoDB Atlas

```yaml
migration_strategy:

  évaluation:
    - Estimer coût Atlas vs self-hosted
    - Vérifier features disponibles
    - Planifier downtime acceptable

  preparation:
    - Créer cluster Atlas (même version)
    - Tester connectivity
    - Répliquer en staging

  migration_methods:

    method_1_live_migration:
      tool: "mongomirror (Atlas)"
      downtime: "Minimal (< 1 minute)"
      steps:
        - Configure mongomirror
        - Start replication
        - Monitor lag
        - Cutover when lag < 1 second
      suitable: "Production avec HA requirement"

    method_2_backup_restore:
      tool: "mongodump + mongorestore"
      downtime: "Several hours"
      steps:
        - Stop writes
        - mongodump from source
        - mongorestore to Atlas
        - Validate
        - Switch connection string
      suitable: "Acceptable downtime window"

    method_3_change_streams:
      tool: "Custom script with change streams"
      downtime: "Minimal"
      complexity: "High"
      suitable: "Complex migrations with transformations"

  validation:
    - Compare document counts
    - Validate indexes
    - Test application queries
    - Performance benchmarks

  rollback_plan:
    - Keep source cluster running 7+ days
    - Document rollback procedure
    - Test rollback in staging
```

---

## Conclusion

Le sharding MongoDB est une technologie puissante mais exigeante qui nécessite :

- ✅ **Conception rigoureuse** : Shard key optimale dès le départ
- ✅ **Déploiement méthodique** : Pré-splitting, distribution, validation
- ✅ **Monitoring proactif** : Alertes, dashboards, health checks
- ✅ **Maintenance disciplinée** : Fenêtres planifiées, procédures documentées
- ✅ **Sécurité multi-niveaux** : Defense in depth, principe du moindre privilège
- ✅ **Performance continue** : Requêtes optimisées, index appropriés
- ✅ **Évolution planifiée** : Scaling horizontal quand nécessaire

**Règle d'or** : Le sharding n'est pas une solution miracle. Ne shardez que quand nécessaire (>100 GB ou >10K ops/sec), et uniquement après avoir optimisé votre Replica Set existant.

**Investissement requis** : Le sharding demande une expertise technique significative. Assurez-vous que votre équipe est formée et que vous avez les ressources pour maintenir un cluster shardé en production avant de vous lancer.

**Alternative moderne** : Pour beaucoup de cas d'usage, MongoDB Atlas avec auto-scaling peut être une alternative plus simple et plus fiable qu'un cluster shardé auto-géré.

---

## Ressources Finales

### Documentation Officielle

- [MongoDB Sharding Manual](https://docs.mongodb.com/manual/sharding/)
- [MongoDB Production Notes](https://docs.mongodb.com/manual/administration/production-notes/)
- [MongoDB Best Practices](https://docs.mongodb.com/manual/administration/production-checklist-operations/)

### Formation

- [MongoDB University - M103: Basic Cluster Administration](https://university.mongodb.com/courses/M103)
- [MongoDB University - M201: MongoDB Performance](https://university.mongodb.com/courses/M201)
- [MongoDB Certification](https://university.mongodb.com/certification)

### Communauté

- [MongoDB Community Forums](https://www.mongodb.com/community/forums/)
- [MongoDB User Groups](https://www.mongodb.com/user-groups)
- [Stack Overflow - mongodb tag](https://stackoverflow.com/questions/tagged/mongodb)

### Blogs et Articles

- [MongoDB Engineering Blog](https://www.mongodb.com/blog)
- [MongoDB Performance Best Practices](https://www.mongodb.com/basics/best-practices)

---

**Félicitations !** Vous avez terminé le chapitre sur le Sharding. Vous disposez maintenant des connaissances nécessaires pour concevoir, déployer, et maintenir un cluster shardé MongoDB en production avec confiance et expertise.

---


⏭️ [Sécurité](/11-securite/README.md)
