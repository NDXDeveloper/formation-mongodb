🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.10 Monitoring et Maintenance

## Introduction

Le monitoring et la maintenance d'un cluster shardé MongoDB sont des activités critiques qui nécessitent une approche proactive et méthodique. Contrairement à un simple Replica Set, un cluster shardé introduit de multiples points de surveillance : les shards individuels, les config servers, les mongos, et les interactions entre ces composants. Une défaillance non détectée ou une maintenance mal planifiée peut entraîner des dégradations de performance significatives, voire une indisponibilité du service.

Cette section présente les stratégies, outils et bonnes pratiques pour maintenir un cluster shardé en production dans un état optimal, détecter les problèmes avant qu'ils n'impactent les utilisateurs, et effectuer les opérations de maintenance avec un minimum d'interruption.

---

## Architecture de Monitoring

### Composants à Surveiller

```
┌─────────────────────────────────────────────────────────────┐
│                    MONITORING STACK                         │
│              (Prometheus + Grafana + Alertmanager)          │
└──────────────────────┬──────────────────────────────────────┘
                       │
         ┌─────────────┼─────────────┬─────────────┐
         │             │             │             │
    ┌────▼────┐   ┌────▼───┐   ┌─────▼──┐      ┌───▼────┐
    │ Config  │   │Mongos 1│   │Shard A │      │Shard B │
    │Servers  │   │Mongos 2│   │ (RS)   │      │ (RS)   │
    │  (RS)   │   │Mongos 3│   │        │      │        │
    └─────────┘   └────────┘   └────────┘      └────────┘
         │             │             │             │
    Exporter      Exporter      Exporter        Exporter
    :9216         :9216         :9216           :9216
```

**Métriques par composant** :

| Composant | Métriques Critiques |
|-----------|-------------------|
| **Config Servers** | État replica set, latence réplication, oplog size |
| **Mongos** | Connexions actives, requêtes/sec, latence moyenne |
| **Shards** | CPU, RAM, IOPS, distribution chunks, jumbo chunks |
| **Balancer** | Migrations actives, taux d'échec, durée moyenne |
| **Réseau** | Bande passante inter-shards, latence, packet loss |

### Niveaux de Monitoring

```
┌──────────────────────────────────────────────────────────┐
│ Niveau 1 : Infrastructure (OS/Hardware)                  │
│ - CPU, RAM, Disque, Réseau                               │
│ - Outils : node_exporter, collectd, CloudWatch           │
└──────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────┐
│ Niveau 2 : MongoDB Internals                             │
│ - Opcounters, connections, queue depth                   │
│ - Outils : mongodb_exporter, mongostat                   │
└──────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────┐
│ Niveau 3 : Sharding Specifics                            │
│ - Distribution chunks, migrations, balancer              │
│ - Outils : sh.status(), scripts custom                   │
└──────────────────────────────────────────────────────────┘
                          │
┌──────────────────────────────────────────────────────────┐
│ Niveau 4 : Application Performance                       │
│ - Query latency, error rate, throughput                  │
│ - Outils : APM (New Relic, DataDog), logs applicatifs    │
└──────────────────────────────────────────────────────────┘
```

---

## Métriques Essentielles

### 1. Métriques de Distribution

#### Distribution des Chunks

```javascript
// Script de monitoring de la distribution
function checkChunkDistribution() {
  print("=== Distribution des Chunks ===\n");

  var collections = db.getSiblingDB("config").collections.find({
    dropped: false
  }).toArray();

  collections.forEach(function(coll) {
    if (coll.key) {  // Collection shardée
      var distribution = db.getSiblingDB("config").chunks.aggregate([
        { $match: { ns: coll._id } },
        { $group: {
            _id: "$shard",
            numChunks: { $sum: 1 }
          }
        },
        { $sort: { numChunks: -1 } }
      ]).toArray();

      if (distribution.length > 0) {
        var total = distribution.reduce((sum, d) => sum + d.numChunks, 0);
        var max = Math.max(...distribution.map(d => d.numChunks));
        var min = Math.min(...distribution.map(d => d.numChunks));
        var imbalance = ((max - min) / min) * 100;

        print("Collection : " + coll._id);
        print("  Total chunks : " + total);

        distribution.forEach(function(dist) {
          var percent = ((dist.numChunks / total) * 100).toFixed(1);
          print("  " + dist._id + " : " + dist.numChunks + " chunks (" + percent + "%)");
        });

        if (imbalance > 20) {
          print("  ⚠️  ALERTE : Déséquilibre de " + imbalance.toFixed(1) + "%");
        } else {
          print("  ✅ Distribution équilibrée (" + imbalance.toFixed(1) + "%)");
        }
        print("");
      }
    }
  });
}

// Exécuter
checkChunkDistribution();
```

**Seuils recommandés** :

```javascript
// Indicateurs de santé de la distribution
var healthMetrics = {
  balanceThreshold: 20,      // % de déséquilibre acceptable
  minChunksPerShard: 10,     // Minimum de chunks par shard
  maxJumboChunks: 0,         // Nombre acceptable de jumbo chunks
  migrationSuccessRate: 95   // % de succès des migrations
};
```

#### Jumbo Chunks

```javascript
// Détection et analyse des jumbo chunks
function analyzeJumboChunks() {
  print("=== Analyse des Jumbo Chunks ===\n");

  var jumboChunks = db.getSiblingDB("config").chunks.aggregate([
    { $match: { jumbo: true } },
    { $group: {
        _id: "$ns",
        count: { $sum: 1 },
        shards: { $addToSet: "$shard" }
      }
    },
    { $sort: { count: -1 } }
  ]).toArray();

  if (jumboChunks.length === 0) {
    print("✅ Aucun jumbo chunk détecté\n");
    return;
  }

  print("⚠️  " + jumboChunks.length + " collection(s) avec jumbo chunks :\n");

  jumboChunks.forEach(function(jc) {
    print("Collection : " + jc._id);
    print("  Nombre : " + jc.count);
    print("  Shards affectés : " + jc.shards.join(", "));

    // Analyser la shard key de la collection
    var collInfo = db.getSiblingDB("config").collections.findOne({ _id: jc._id });
    print("  Shard key : " + JSON.stringify(collInfo.key));

    // Recommandations
    print("  Recommandation :");
    print("    1. Vérifier la cardinalité de la shard key");
    print("    2. Considérer refineCollectionShardKey (MongoDB 5.0+)");
    print("    3. Ou forcer migration avec attemptToBalanceJumboChunks");
    print("");
  });
}

// Exécuter
analyzeJumboChunks();
```

### 2. Métriques du Balancer

#### État et Performance du Balancer

```javascript
// Monitoring complet du balancer
function monitorBalancer() {
  print("=== Monitoring du Balancer ===\n");

  // 1. État global
  var isEnabled = sh.getBalancerState();
  var isRunning = sh.isBalancerRunning();

  print("État :");
  print("  Activé : " + isEnabled);
  print("  En cours : " + isRunning);

  // 2. Fenêtre active
  var balancerConfig = db.getSiblingDB("config").settings.findOne({ _id: "balancer" });

  if (balancerConfig && balancerConfig.activeWindow) {
    print("  Fenêtre : " + balancerConfig.activeWindow.start + " - " + balancerConfig.activeWindow.stop);
  } else {
    print("  Fenêtre : Actif 24/7");
  }
  print("");

  // 3. Statistiques (dernières 24h)
  var last24h = new Date(Date.now() - 86400000);

  var migrations = db.getSiblingDB("config").changelog.aggregate([
    {
      $match: {
        time: { $gte: last24h },
        what: { $in: ["moveChunk.start", "moveChunk.commit", "moveChunk.error"] }
      }
    },
    {
      $group: {
        _id: "$what",
        count: { $sum: 1 }
      }
    }
  ]).toArray();

  print("Statistiques (24h) :");

  var stats = {
    started: 0,
    committed: 0,
    errors: 0
  };

  migrations.forEach(function(m) {
    if (m._id === "moveChunk.start") stats.started = m.count;
    if (m._id === "moveChunk.commit") stats.committed = m.count;
    if (m._id === "moveChunk.error") stats.errors = m.count;
  });

  print("  Migrations initiées : " + stats.started);
  print("  Migrations réussies : " + stats.committed);
  print("  Migrations échouées : " + stats.errors);

  if (stats.committed > 0) {
    var successRate = ((stats.committed / (stats.committed + stats.errors)) * 100).toFixed(2);
    print("  Taux de succès : " + successRate + "%");

    if (successRate < 95) {
      print("  ⚠️  ALERTE : Taux de succès faible (< 95%)");
    }
  }
  print("");

  // 4. Durée moyenne des migrations
  var avgDuration = db.getSiblingDB("config").changelog.aggregate([
    {
      $match: {
        what: "moveChunk.commit",
        time: { $gte: last24h }
      }
    },
    {
      $project: {
        duration: {
          $subtract: ["$time", "$details.step1of6"]
        }
      }
    },
    {
      $group: {
        _id: null,
        avgDurationMs: { $avg: "$duration" },
        maxDurationMs: { $max: "$duration" },
        count: { $sum: 1 }
      }
    }
  ]).toArray();

  if (avgDuration.length > 0) {
    var avg = avgDuration[0];
    print("Durée des migrations :");
    print("  Moyenne : " + (avg.avgDurationMs / 1000).toFixed(2) + "s");
    print("  Maximum : " + (avg.maxDurationMs / 1000).toFixed(2) + "s");
    print("  Échantillon : " + avg.count + " migrations");

    if (avg.avgDurationMs > 60000) {  // > 1 minute
      print("  ⚠️  ATTENTION : Durée moyenne élevée (> 1min)");
    }
  }
  print("");

  // 5. Migrations en cours
  var activeMigrations = db.currentOp({
    $or: [
      { op: "command", "command.moveChunk": { $exists: true } },
      { desc: /^migrateThread/ }
    ]
  }).inprog;

  print("Migrations actives : " + activeMigrations.length);

  activeMigrations.forEach(function(mig) {
    print("  Collection : " + mig.ns);
    print("  Durée : " + (mig.microsecs_running / 1000000).toFixed(2) + "s");
    if (mig.progress) {
      var percent = ((mig.progress.done / mig.progress.total) * 100).toFixed(1);
      print("  Progression : " + percent + "%");
    }
    print("");
  });
}

// Exécuter
monitorBalancer();
```

### 3. Métriques de Performance

#### Latence des Requêtes

```javascript
// Analyse des performances par type de requête
function analyzeQueryPerformance() {
  print("=== Analyse des Performances des Requêtes ===\n");

  // 1. Requêtes lentes en cours
  var slowQueries = db.currentOp({
    "secs_running": { $gte: 3 },
    "op": { $in: ["query", "update", "remove", "command"] }
  }).inprog;

  print("Requêtes lentes (>3s) : " + slowQueries.length + "\n");

  if (slowQueries.length > 0) {
    slowQueries.forEach(function(query) {
      print("Namespace : " + query.ns);
      print("Opération : " + query.op);
      print("Durée : " + query.secs_running + "s");

      if (query.command) {
        // Déterminer si c'est une requête targeted ou broadcast
        var filter = query.command.filter || query.command.query || {};
        print("Filtre : " + JSON.stringify(filter));
      }

      print("");
    });
  }

  // 2. Statistiques du profiler (si activé)
  // Activer le profiler : db.setProfilingLevel(1, { slowms: 100 })

  try {
    var profileStats = db.system.profile.aggregate([
      { $match: { ts: { $gte: new Date(Date.now() - 3600000) } } },  // 1h
      {
        $group: {
          _id: "$op",
          count: { $sum: 1 },
          avgDuration: { $avg: "$millis" },
          maxDuration: { $max: "$millis" }
        }
      },
      { $sort: { avgDuration: -1 } }
    ]).toArray();

    if (profileStats.length > 0) {
      print("Statistiques Profiler (1h) :");
      profileStats.forEach(function(stat) {
        print("  " + stat._id + " :");
        print("    Count : " + stat.count);
        print("    Avg : " + stat.avgDuration.toFixed(2) + "ms");
        print("    Max : " + stat.maxDuration.toFixed(2) + "ms");
      });
      print("");
    }
  } catch (e) {
    print("Profiler non activé ou non accessible\n");
  }
}

// Exécuter
analyzeQueryPerformance();
```

#### Métriques par Shard

```javascript
// Collecte de métriques détaillées par shard
function collectShardMetrics() {
  print("=== Métriques par Shard ===\n");

  var shards = db.getSiblingDB("config").shards.find().toArray();

  shards.forEach(function(shard) {
    print("Shard : " + shard._id);

    try {
      // Connexion au primary du shard
      var shardHost = shard.host.split("/")[1].split(",")[0];
      var shardConn = new Mongo(shardHost);

      // 1. État du replica set
      var rsStatus = shardConn.getDB("admin").adminCommand({ replSetGetStatus: 1 });
      var primary = rsStatus.members.find(m => m.state === 1);
      var secondaries = rsStatus.members.filter(m => m.state === 2);

      print("  Replica Set :");
      print("    Primary : " + primary.name);
      print("    Secondaries : " + secondaries.length);

      // Lag de réplication
      if (secondaries.length > 0) {
        var maxLag = 0;
        secondaries.forEach(function(sec) {
          var lag = (primary.optimeDate - sec.optimeDate) / 1000;
          maxLag = Math.max(maxLag, lag);
        });
        print("    Replication lag (max) : " + maxLag.toFixed(2) + "s");

        if (maxLag > 10) {
          print("    ⚠️  ATTENTION : Lag élevé (> 10s)");
        }
      }

      // 2. Statistiques du serveur
      var serverStatus = shardConn.getDB("admin").serverStatus();

      print("  Performance :");
      print("    Connexions : " + serverStatus.connections.current + "/" + serverStatus.connections.available);
      print("    Ops/sec : ");
      print("      Insert : " + serverStatus.opcounters.insert);
      print("      Query : " + serverStatus.opcounters.query);
      print("      Update : " + serverStatus.opcounters.update);
      print("      Delete : " + serverStatus.opcounters.delete);

      // 3. Mémoire WiredTiger
      var wtMem = serverStatus.wiredTiger.cache;
      var cachePct = ((wtMem["bytes currently in the cache"] / wtMem["maximum bytes configured"]) * 100).toFixed(1);

      print("  Mémoire (WiredTiger Cache) :");
      print("    Taille : " + (wtMem["bytes currently in the cache"] / 1024 / 1024 / 1024).toFixed(2) + " GB");
      print("    Max : " + (wtMem["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2) + " GB");
      print("    Utilisation : " + cachePct + "%");

      if (cachePct > 90) {
        print("    ⚠️  ATTENTION : Cache presque plein (> 90%)");
      }

      // 4. Stockage
      var dbStats = shardConn.getDB("admin").adminCommand({
        listDatabases: 1,
        nameOnly: false
      });

      var totalSize = dbStats.databases.reduce((sum, db) => sum + (db.sizeOnDisk || 0), 0);

      print("  Stockage :");
      print("    Taille totale : " + (totalSize / 1024 / 1024 / 1024).toFixed(2) + " GB");

    } catch (e) {
      print("  ❌ Erreur de connexion : " + e.message);
    }

    print("");
  });
}

// Exécuter
collectShardMetrics();
```

### 4. Métriques des Config Servers

```javascript
// Monitoring spécifique des config servers
function monitorConfigServers() {
  print("=== Monitoring Config Servers ===\n");

  // Récupérer les config servers depuis la connection string de mongos
  var adminDB = db.getSiblingDB("admin");
  var cmdLineOpts = adminDB.adminCommand({ getCmdLineOpts: 1 });
  var configDBString = cmdLineOpts.parsed.sharding.configDB;

  var configServers = configDBString.split("/")[1].split(",");

  print("Config Servers : " + configServers.length + "\n");

  configServers.forEach(function(cs) {
    print("Config Server : " + cs);

    try {
      var csConn = new Mongo(cs);

      // 1. État du membre
      var isMaster = csConn.getDB("admin").isMaster();
      print("  État : " + (isMaster.ismaster ? "PRIMARY" : isMaster.secondary ? "SECONDARY" : "OTHER"));
      print("  Replica Set : " + isMaster.setName);

      // 2. Statistiques des métadonnées
      var configDB = csConn.getDB("config");

      var chunksCount = configDB.chunks.countDocuments({});
      var shardsCount = configDB.shards.countDocuments({});
      var collectionsCount = configDB.collections.countDocuments({ dropped: false });

      print("  Métadonnées :");
      print("    Chunks : " + chunksCount);
      print("    Shards : " + shardsCount);
      print("    Collections shardées : " + collectionsCount);

      // 3. Taille de l'oplog
      var localDB = csConn.getDB("local");
      var oplogStats = localDB.oplog.rs.stats();

      var oplogSizeGB = (oplogStats.maxSize / 1024 / 1024 / 1024).toFixed(2);
      var oplogUsedGB = (oplogStats.size / 1024 / 1024 / 1024).toFixed(2);
      var oplogPct = ((oplogStats.size / oplogStats.maxSize) * 100).toFixed(1);

      print("  Oplog :");
      print("    Taille : " + oplogUsedGB + " GB / " + oplogSizeGB + " GB (" + oplogPct + "%)");

      // Durée couverte par l'oplog
      var firstOplog = localDB.oplog.rs.find().sort({ $natural: 1 }).limit(1).toArray()[0];
      var lastOplog = localDB.oplog.rs.find().sort({ $natural: -1 }).limit(1).toArray()[0];

      if (firstOplog && lastOplog) {
        var oplogDurationHours = ((lastOplog.ts.getTime() - firstOplog.ts.getTime()) / 1000 / 3600).toFixed(1);
        print("    Durée couverte : " + oplogDurationHours + " heures");

        if (oplogDurationHours < 24) {
          print("    ⚠️  ATTENTION : Oplog couvre moins de 24h");
        }
      }

    } catch (e) {
      print("  ❌ Erreur : " + e.message);
    }

    print("");
  });
}

// Exécuter
monitorConfigServers();
```

---

## Outils de Monitoring

### 1. MongoDB Native Tools

#### mongostat

```bash
# Monitoring en temps réel via mongos
mongostat --host mongos1.example.com:27017 -u admin -p password --authenticationDatabase admin

# Output :
# insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in net_out conn                time
#   *456   789   *234    12       0   1567  3.4% 80.2%       0  8.3G  4.2G 0|0 1|0  157mb   89mb  128 Dec  8 10:30:15.123

# Colonnes importantes :
# - insert, query, update, delete : Opcounters par seconde
# - command : Commandes par seconde
# - dirty : % de données modifiées en cache
# - used : % d'utilisation du cache WiredTiger
# - qrw : Queue depth (read|write)
# - conn : Nombre de connexions actives
```

#### mongotop

```bash
# Identifier les collections les plus actives
mongotop --host mongos1.example.com:27017 -u admin -p password --authenticationDatabase admin 30

# Output :
#                     ns    total    read    write
# mydb.orders           450ms   120ms    330ms
# mydb.users            230ms   230ms      0ms
# mydb.analytics       1200ms  1200ms      0ms

# Utile pour :
# - Identifier les collections "hot"
# - Détecter les patterns d'accès anormaux
# - Valider la distribution de charge
```

### 2. MongoDB Exporter + Prometheus + Grafana

#### Installation de mongodb_exporter

```bash
# Installer mongodb_exporter sur chaque shard et config server

# 1. Télécharger
wget https://github.com/percona/mongodb_exporter/releases/download/v0.40.0/mongodb_exporter-0.40.0.linux-amd64.tar.gz
tar xvzf mongodb_exporter-0.40.0.linux-amd64.tar.gz

# 2. Créer un utilisateur de monitoring
mongosh --host localhost:27018
use admin
db.createUser({
  user: "mongodb_exporter",
  pwd: "SecureExporterPassword",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})

# 3. Configurer le exporter
cat > /etc/systemd/system/mongodb_exporter.service <<EOF
[Unit]
Description=MongoDB Exporter
After=network.target

[Service]
Type=simple
User=mongodb_exporter
Environment="MONGODB_URI=mongodb://mongodb_exporter:SecureExporterPassword@localhost:27018/admin"
ExecStart=/usr/local/bin/mongodb_exporter
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 4. Démarrer
sudo systemctl daemon-reload
sudo systemctl start mongodb_exporter
sudo systemctl enable mongodb_exporter

# Vérifier
curl http://localhost:9216/metrics | grep mongodb_up
# mongodb_up 1
```

#### Configuration Prometheus

```yaml
# /etc/prometheus/prometheus.yml

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  # Config Servers
  - job_name: 'mongodb-config'
    static_configs:
      - targets:
        - 'cfg1.example.com:9216'
        - 'cfg2.example.com:9216'
        - 'cfg3.example.com:9216'
        labels:
          cluster: 'production'
          component: 'config-server'

  # Mongos
  - job_name: 'mongodb-mongos'
    static_configs:
      - targets:
        - 'mongos1.example.com:9216'
        - 'mongos2.example.com:9216'
        - 'mongos3.example.com:9216'
        labels:
          cluster: 'production'
          component: 'mongos'

  # Shards
  - job_name: 'mongodb-shards'
    static_configs:
      - targets:
        - 'shardA1.example.com:9216'
        - 'shardA2.example.com:9216'
        - 'shardA3.example.com:9216'
        labels:
          cluster: 'production'
          component: 'shard'
          shard: 'shardA'

      - targets:
        - 'shardB1.example.com:9216'
        - 'shardB2.example.com:9216'
        - 'shardB3.example.com:9216'
        labels:
          cluster: 'production'
          component: 'shard'
          shard: 'shardB'
```

#### Requêtes PromQL Essentielles

```promql
# 1. Connexions actives (par composant)
mongodb_connections{state="current", component="mongos"}

# 2. Opcounters (taux par seconde)
rate(mongodb_op_counters_total[5m])

# 3. Lag de réplication (maximum)
max(mongodb_mongod_replset_member_replication_lag) by (set)

# 4. Utilisation du cache WiredTiger
(mongodb_wiredtiger_cache_bytes{type="total"} / mongodb_wiredtiger_cache_max_bytes) * 100

# 5. Queue depth (lectures + écritures)
mongodb_mongod_global_lock_current_queue{type="reader"} +
mongodb_mongod_global_lock_current_queue{type="writer"}

# 6. Distribution des chunks (nécessite custom exporter)
# Voir section "Custom Exporters" ci-dessous

# 7. Durée moyenne des requêtes
rate(mongodb_mongod_op_latencies_latency_total[5m]) /
rate(mongodb_mongod_op_latencies_ops_total[5m])

# 8. Taux d'erreur des commandes
rate(mongodb_mongod_metrics_commands_failed_total[5m]) /
rate(mongodb_mongod_metrics_commands_total[5m]) * 100
```

#### Dashboard Grafana pour Cluster Shardé

```json
{
  "dashboard": {
    "title": "MongoDB Sharded Cluster Overview",
    "panels": [
      {
        "title": "Cluster Health",
        "targets": [
          {
            "expr": "mongodb_up{component=\"config-server\"}",
            "legendFormat": "Config: {{instance}}"
          },
          {
            "expr": "mongodb_up{component=\"mongos\"}",
            "legendFormat": "Mongos: {{instance}}"
          },
          {
            "expr": "mongodb_up{component=\"shard\"}",
            "legendFormat": "Shard: {{shard}} - {{instance}}"
          }
        ]
      },
      {
        "title": "Operations per Second",
        "targets": [
          {
            "expr": "sum(rate(mongodb_op_counters_total[5m])) by (type)",
            "legendFormat": "{{type}}"
          }
        ]
      },
      {
        "title": "Replication Lag",
        "targets": [
          {
            "expr": "max(mongodb_mongod_replset_member_replication_lag) by (set)",
            "legendFormat": "{{set}}"
          }
        ],
        "thresholds": [
          { "value": 10, "color": "orange" },
          { "value": 30, "color": "red" }
        ]
      },
      {
        "title": "WiredTiger Cache Usage",
        "targets": [
          {
            "expr": "(mongodb_wiredtiger_cache_bytes{type=\"total\"} / mongodb_wiredtiger_cache_max_bytes) * 100",
            "legendFormat": "{{instance}}"
          }
        ]
      },
      {
        "title": "Connections",
        "targets": [
          {
            "expr": "mongodb_connections{state=\"current\"}",
            "legendFormat": "{{instance}} - {{component}}"
          }
        ]
      },
      {
        "title": "Slow Queries (>100ms)",
        "targets": [
          {
            "expr": "rate(mongodb_mongod_metrics_query_executor_total{state=\"scanned\"}[5m])",
            "legendFormat": "{{instance}}"
          }
        ]
      }
    ]
  }
}
```

### 3. Custom Exporters pour Métriques Sharding

```python
# custom_sharding_exporter.py
# Exporter personnalisé pour métriques de sharding

from prometheus_client import start_http_server, Gauge
from pymongo import MongoClient
import time

# Métriques Prometheus
chunk_distribution = Gauge('mongodb_chunk_distribution', 'Number of chunks per shard', ['namespace', 'shard'])
jumbo_chunks = Gauge('mongodb_jumbo_chunks', 'Number of jumbo chunks', ['namespace'])
migration_success_rate = Gauge('mongodb_migration_success_rate', 'Migration success rate (24h)', [])
balancer_state = Gauge('mongodb_balancer_state', 'Balancer state (1=enabled, 0=disabled)', [])

def collect_sharding_metrics(mongos_uri):
    client = MongoClient(mongos_uri)
    config_db = client['config']

    # 1. Distribution des chunks
    chunk_dist = config_db.chunks.aggregate([
        {'$group': {
            '_id': {'ns': '$ns', 'shard': '$shard'},
            'count': {'$sum': 1}
        }}
    ])

    for doc in chunk_dist:
        chunk_distribution.labels(
            namespace=doc['_id']['ns'],
            shard=doc['_id']['shard']
        ).set(doc['count'])

    # 2. Jumbo chunks
    jumbo_dist = config_db.chunks.aggregate([
        {'$match': {'jumbo': True}},
        {'$group': {
            '_id': '$ns',
            'count': {'$sum': 1}
        }}
    ])

    for doc in jumbo_dist:
        jumbo_chunks.labels(namespace=doc['_id']).set(doc['count'])

    # 3. Taux de succès des migrations (24h)
    last_24h = time.time() - 86400

    success = config_db.changelog.count_documents({
        'what': 'moveChunk.commit',
        'time': {'$gte': last_24h}
    })

    errors = config_db.changelog.count_documents({
        'what': 'moveChunk.error',
        'time': {'$gte': last_24h}
    })

    if success + errors > 0:
        rate = (success / (success + errors)) * 100
        migration_success_rate.set(rate)

    # 4. État du balancer
    admin_db = client['admin']
    balancer_status = admin_db.command('balancerStatus')
    balancer_state.set(1 if balancer_status['mode'] == 'full' else 0)

if __name__ == '__main__':
    MONGOS_URI = 'mongodb://monitor_user:password@mongos1.example.com:27017/admin'

    # Démarrer le serveur HTTP pour Prometheus
    start_http_server(9217)

    print("Custom sharding exporter started on port 9217")

    # Boucle de collecte
    while True:
        try:
            collect_sharding_metrics(MONGOS_URI)
        except Exception as e:
            print(f"Error collecting metrics: {e}")

        time.sleep(60)  # Collecter toutes les minutes
```

```bash
# Déployer le custom exporter

# 1. Installer dépendances
pip3 install prometheus_client pymongo

# 2. Créer un service systemd
cat > /etc/systemd/system/mongodb-sharding-exporter.service <<EOF
[Unit]
Description=MongoDB Sharding Custom Exporter
After=network.target

[Service]
Type=simple
User=prometheus
ExecStart=/usr/bin/python3 /opt/custom_sharding_exporter.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# 3. Démarrer
sudo systemctl daemon-reload
sudo systemctl start mongodb-sharding-exporter
sudo systemctl enable mongodb-sharding-exporter

# 4. Ajouter à Prometheus
# scrape_configs:
#   - job_name: 'mongodb-sharding-metrics'
#     static_configs:
#       - targets: ['localhost:9217']
```

---

## Alerting : Règles Critiques

### Configuration Alertmanager

```yaml
# /etc/alertmanager/alertmanager.yml

route:
  receiver: 'team-database'
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'team-database'
    email_configs:
      - to: 'dba-team@example.com'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'your-pagerduty-key'

  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#mongodb-alerts'
```

### Règles d'Alerte Prometheus

```yaml
# /etc/prometheus/rules/mongodb-sharded.yml

groups:
  - name: mongodb_sharded_cluster
    interval: 30s
    rules:

      # === Alertes Critiques ===

      - alert: ConfigServerDown
        expr: mongodb_up{component="config-server"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Config Server down: {{ $labels.instance }}"
          description: "Config server {{ $labels.instance }} is unreachable. Cluster may become read-only."

      - alert: MajorityConfigServersDown
        expr: count(mongodb_up{component="config-server"} == 0) >= 2
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Majority of Config Servers down"
          description: "{{ $value }} config servers are down. Cluster is READ-ONLY!"

      - alert: ShardDown
        expr: mongodb_up{component="shard"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Shard node down: {{ $labels.shard }} - {{ $labels.instance }}"
          description: "Shard member {{ $labels.instance }} of {{ $labels.shard }} is down."

      - alert: HighReplicationLag
        expr: mongodb_mongod_replset_member_replication_lag > 30
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High replication lag: {{ $labels.set }}"
          description: "Replication lag on {{ $labels.set }} is {{ $value }}s (threshold: 30s)"

      - alert: MigrationFailureRateHigh
        expr: |
          (
            rate(mongodb_changelog_count{what="moveChunk.error"}[1h]) /
            (rate(mongodb_changelog_count{what="moveChunk.commit"}[1h]) +
             rate(mongodb_changelog_count{what="moveChunk.error"}[1h]))
          ) * 100 > 20
        for: 30m
        labels:
          severity: critical
        annotations:
          summary: "High migration failure rate"
          description: "Migration failure rate is {{ $value }}% (threshold: 20%)"

      # === Alertes Warning ===

      - alert: WiredTigerCacheHighUsage
        expr: |
          (mongodb_wiredtiger_cache_bytes{type="total"} /
           mongodb_wiredtiger_cache_max_bytes) * 100 > 90
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "WiredTiger cache high usage: {{ $labels.instance }}"
          description: "Cache usage is {{ $value }}% on {{ $labels.instance }}"

      - alert: HighConnectionCount
        expr: |
          (mongodb_connections{state="current"} /
           mongodb_connections{state="available"}) * 100 > 80
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High connection count: {{ $labels.instance }}"
          description: "Connection usage is {{ $value }}% on {{ $labels.instance }}"

      - alert: JumboChunksDetected
        expr: mongodb_jumbo_chunks > 0
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Jumbo chunks detected: {{ $labels.namespace }}"
          description: "{{ $value }} jumbo chunks on {{ $labels.namespace }}"

      - alert: ClusterImbalance
        expr: |
          (
            max(mongodb_chunk_distribution) by (namespace) -
            min(mongodb_chunk_distribution) by (namespace)
          ) / min(mongodb_chunk_distribution) by (namespace) * 100 > 30
        for: 4h
        labels:
          severity: warning
        annotations:
          summary: "Cluster imbalance: {{ $labels.namespace }}"
          description: "Chunk distribution imbalance is {{ $value }}%"

      - alert: SlowQueries
        expr: rate(mongodb_mongod_metrics_query_executor_total{state="scanned"}[5m]) > 100
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "High rate of slow queries: {{ $labels.instance }}"
          description: "{{ $value }} slow queries/sec on {{ $labels.instance }}"

      - alert: LongRunningMigration
        expr: mongodb_current_op_duration_seconds{op="command", command="moveChunk"} > 1800
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Long-running migration detected"
          description: "Migration running for {{ $value }}s (>30min)"
```

---

## Stratégies de Maintenance

### 1. Maintenance Planifiée : Rolling Restart

```bash
# Procédure de rolling restart sans downtime

# ========================================
# PHASE 1 : Préparation
# ========================================

# 1. Notification
echo "MAINTENANCE : Rolling restart prévu le 2024-01-15 02:00-06:00 UTC"

# 2. Désactiver le balancer
mongosh --host mongos1.example.com --eval "sh.stopBalancer()"

# Attendre que les migrations en cours se terminent
while mongosh --host mongos1.example.com --eval "sh.isBalancerRunning()" | grep -q "true"; do
  echo "Attente fin migrations..."
  sleep 10
done

echo "Balancer arrêté, aucune migration en cours"

# 3. Sauvegarde de précaution
./backup-cluster.sh

# ========================================
# PHASE 2 : Restart des Mongos
# ========================================

# Pour chaque mongos (mongos1, mongos2, mongos3)

for MONGOS in mongos1 mongos2 mongos3; do
  echo "Restart de $MONGOS..."

  # Retirer du load balancer (si applicable)
  # aws elb deregister-instances-from-load-balancer --load-balancer-name my-lb --instances $MONGOS

  # Attendre drain des connexions
  sleep 30

  # Restart
  ssh $MONGOS.example.com "sudo systemctl restart mongos"

  # Vérifier
  sleep 10
  mongosh --host $MONGOS.example.com --eval "db.adminCommand('ping')"

  if [ $? -eq 0 ]; then
    echo "✅ $MONGOS redémarré avec succès"
  else
    echo "❌ ERREUR : $MONGOS ne répond pas !"
    exit 1
  fi

  # Réintégrer au load balancer
  # aws elb register-instances-with-load-balancer --load-balancer-name my-lb --instances $MONGOS

  sleep 60  # Pause entre chaque mongos
done

# ========================================
# PHASE 3 : Restart des Config Servers
# ========================================

# Ordre : SECONDARIES d'abord, PRIMARY en dernier

for CFG in cfg2 cfg3 cfg1; do
  echo "Restart de $CFG..."

  # Vérifier le rôle
  ROLE=$(mongosh --host $CFG.example.com:27019 --quiet --eval "rs.isMaster().ismaster")

  if [ "$ROLE" == "true" ] && [ "$CFG" != "cfg1" ]; then
    echo "⚠️  $CFG est PRIMARY, step down..."
    mongosh --host $CFG.example.com:27019 --eval "rs.stepDown(60)"
    sleep 10
  fi

  # Restart
  ssh $CFG.example.com "sudo systemctl restart mongod-configsvr"

  # Attendre réintégration au replica set
  sleep 30

  # Vérifier
  mongosh --host $CFG.example.com:27019 --eval "rs.status()"

  if [ $? -eq 0 ]; then
    echo "✅ $CFG redémarré avec succès"
  else
    echo "❌ ERREUR : $CFG ne répond pas !"
    exit 1
  fi

  sleep 60
done

# ========================================
# PHASE 4 : Restart des Shards (un à un)
# ========================================

for SHARD in shardA shardB; do
  echo "Restart du shard $SHARD..."

  # Membres du shard
  MEMBERS=("${SHARD}1" "${SHARD}2" "${SHARD}3")

  # Secondaries d'abord
  for MEMBER in "${MEMBERS[@]}"; do
    echo "  Restart de $MEMBER..."

    ROLE=$(mongosh --host $MEMBER.example.com:27018 --quiet --eval "rs.isMaster().ismaster")

    if [ "$ROLE" == "true" ]; then
      echo "  $MEMBER est PRIMARY, skip pour l'instant"
      PRIMARY=$MEMBER
      continue
    fi

    # Restart secondary
    ssh $MEMBER.example.com "sudo systemctl restart mongod-shard"
    sleep 30

    # Vérifier réintégration
    mongosh --host $MEMBER.example.com:27018 --eval "rs.status()"

    if [ $? -eq 0 ]; then
      echo "  ✅ $MEMBER redémarré"
    else
      echo "  ❌ ERREUR : $MEMBER ne répond pas !"
      exit 1
    fi

    sleep 30
  done

  # Primary en dernier
  echo "  Restart du PRIMARY : $PRIMARY..."

  # Step down
  mongosh --host $PRIMARY.example.com:27018 --eval "rs.stepDown(60)"
  sleep 15

  # Restart
  ssh $PRIMARY.example.com "sudo systemctl restart mongod-shard"
  sleep 30

  # Vérifier
  mongosh --host $PRIMARY.example.com:27018 --eval "rs.status()"

  if [ $? -eq 0 ]; then
    echo "  ✅ $PRIMARY redémarré"
  else
    echo "  ❌ ERREUR : $PRIMARY ne répond pas !"
    exit 1
  fi

  sleep 60
done

# ========================================
# PHASE 5 : Validation et Réactivation
# ========================================

echo "Validation du cluster..."

# 1. Vérifier la santé globale
mongosh --host mongos1.example.com --eval "sh.status()"

# 2. Réactiver le balancer
mongosh --host mongos1.example.com --eval "sh.startBalancer()"

echo "✅ Rolling restart terminé avec succès"
echo "Balancer réactivé, cluster opérationnel"
```

### 2. Maintenance Non Planifiée : Gestion d'Incident

```bash
# Procédure d'urgence : Shard Primary down

# ========================================
# DIAGNOSTIC
# ========================================

# 1. Identifier le problème
mongosh --host mongos1.example.com --eval "sh.status()"

# Output montre :
# shardA: "PRIMARY : UNKNOWN"

# 2. Connexion au replica set du shard
mongosh --host shardA1.example.com:27018 --eval "rs.status()"

# Primary est down, une élection devrait avoir lieu

# ========================================
# ACTIONS
# ========================================

# Si élection bloquée (pas de majorité) :

# Option 1 : Redémarrer le primary
ssh shardA1.example.com "sudo systemctl start mongod-shard"

# Attendre et vérifier
sleep 30
mongosh --host shardA1.example.com:27018 --eval "rs.status()"

# Option 2 : Si hardware HS, forcer élection sur secondary
mongosh --host shardA2.example.com:27018

# Reconfig pour retirer le membre HS
cfg = rs.conf()
cfg.members = cfg.members.filter(m => m.host !== "shardA1.example.com:27018")
rs.reconfig(cfg, {force: true})

# Option 3 : Promouvoir un secondary manuellement
mongosh --host shardA2.example.com:27018
rs.stepDown(0)  # Sur le current primary (si accessible)

# Ou forcer la priorité
cfg = rs.conf()
cfg.members[1].priority = 10  # shardA2
rs.reconfig(cfg)

# ========================================
# VALIDATION
# ========================================

# Vérifier que le shard a un primary
mongosh --host mongos1.example.com --eval "sh.status()"

# Tester une opération CRUD sur le shard
mongosh --host mongos1.example.com
use mydb
db.test_collection.insertOne({test: "recovery", timestamp: new Date()})

# Si succès, cluster recovered

# ========================================
# POST-INCIDENT
# ========================================

# 1. Analyser les logs pour comprendre la cause
ssh shardA1.example.com "sudo tail -n 500 /var/log/mongodb/shard.log"

# 2. Documenter l'incident
# - Heure de début
# - Cause identifiée
# - Actions prises
# - Durée d'impact
# - Leçons apprises

# 3. Améliorer le monitoring/alerting
# Ajouter une alerte pour ce scénario spécifique
```

### 3. Maintenance Préventive : Nettoyage et Optimisation

```javascript
// Script de maintenance préventive (à exécuter mensuellement)

function preventiveMaintenance() {
  print("=== MAINTENANCE PRÉVENTIVE ===\n");

  // 1. Nettoyer les collections système obsolètes
  print("1. Nettoyage des logs système\n");

  // Config changelog (garder 30 jours)
  var cutoffDate = new Date(Date.now() - 30 * 86400000);

  var deletedChangelog = db.getSiblingDB("config").changelog.deleteMany({
    time: { $lt: cutoffDate }
  });

  print("  Entrées changelog supprimées : " + deletedChangelog.deletedCount);

  // 2. Compacter les collections (si espace disque faible)
  print("\n2. Vérification de l'espace disque\n");

  var shards = db.getSiblingDB("config").shards.find().toArray();

  shards.forEach(function(shard) {
    try {
      var shardConn = new Mongo(shard.host.split("/")[1].split(",")[0]);
      var dbStats = shardConn.getDB("admin").adminCommand({
        listDatabases: 1,
        nameOnly: false
      });

      dbStats.databases.forEach(function(dbInfo) {
        if (dbInfo.name !== "admin" && dbInfo.name !== "local" && dbInfo.name !== "config") {
          var sizeGB = (dbInfo.sizeOnDisk / 1024 / 1024 / 1024).toFixed(2);
          print("  " + shard._id + " - " + dbInfo.name + " : " + sizeGB + " GB");

          // Si plus de 500 GB, considérer compact
          if (dbInfo.sizeOnDisk > 500 * 1024 * 1024 * 1024) {
            print("    ⚠️  Considérer compact pour " + dbInfo.name);
          }
        }
      });

    } catch (e) {
      print("  ❌ Erreur sur " + shard._id + " : " + e.message);
    }
  });

  // 3. Vérifier les index inutilisés
  print("\n3. Analyse des index inutilisés\n");

  var collections = db.getSiblingDB("config").collections.find({
    dropped: false
  }).toArray();

  collections.forEach(function(coll) {
    if (coll.key) {
      print("  Collection : " + coll._id);

      try {
        // Obtenir les stats d'index
        var dbName = coll._id.split(".")[0];
        var collName = coll._id.split(".")[1];

        var indexStats = db.getSiblingDB(dbName)[collName].aggregate([
          { $indexStats: {} }
        ]).toArray();

        indexStats.forEach(function(idx) {
          if (idx.accesses && idx.accesses.ops === 0) {
            print("    ⚠️  Index inutilisé : " + idx.name);
            print("       Envisager suppression avec : db." + collName + ".dropIndex(\"" + idx.name + "\")");
          }
        });

      } catch (e) {
        // $indexStats non disponible sur toutes les versions
      }
    }
  });

  // 4. Vérifier la fragmentation
  print("\n4. Vérification de la fragmentation\n");

  shards.forEach(function(shard) {
    try {
      var shardConn = new Mongo(shard.host.split("/")[1].split(",")[0]);
      var serverStatus = shardConn.getDB("admin").serverStatus();

      if (serverStatus.wiredTiger && serverStatus.wiredTiger.block) {
        var bytesRead = serverStatus.wiredTiger.block["bytes read"];
        var bytesWritten = serverStatus.wiredTiger.block["bytes written"];

        print("  " + shard._id + " :");
        print("    Bytes lus : " + (bytesRead / 1024 / 1024 / 1024).toFixed(2) + " GB");
        print("    Bytes écrits : " + (bytesWritten / 1024 / 1024 / 1024).toFixed(2) + " GB");
      }

    } catch (e) {
      print("  ❌ Erreur sur " + shard._id);
    }
  });

  // 5. Vérifier l'état des connexions
  print("\n5. État des connexions\n");

  var mongosConnections = db.serverStatus().connections;
  print("  Mongos :");
  print("    Actives : " + mongosConnections.current);
  print("    Disponibles : " + mongosConnections.available);

  var pct = (mongosConnections.current / (mongosConnections.current + mongosConnections.available) * 100).toFixed(1);
  print("    Utilisation : " + pct + "%");

  if (pct > 80) {
    print("    ⚠️  ATTENTION : Utilisation élevée des connexions (> 80%)");
  }

  print("\n=== FIN MAINTENANCE PRÉVENTIVE ===");
}

// Exécuter
preventiveMaintenance();
```

---

## Anti-Patterns de Monitoring et Maintenance

### ❌ Anti-Pattern 1 : Monitoring Insuffisant

**Problème** :

```javascript
// Uniquement vérifier que les serveurs répondent au ping
// Pas de monitoring des métriques MongoDB spécifiques
```

**Conséquence** :
- Problèmes détectés trop tard (après impact utilisateurs)
- Pas de visibilité sur les tendances
- Impossible de faire du capacity planning

**Solution** :

```yaml
# Stack de monitoring complète
monitoring:
  niveau_1_infrastructure:
    - CPU, RAM, Disque, Réseau
    - Outils: node_exporter, CloudWatch

  niveau_2_mongodb:
    - Opcounters, connections, cache
    - Outils: mongodb_exporter

  niveau_3_sharding:
    - Distribution chunks, migrations
    - Outils: custom exporter

  niveau_4_application:
    - Latence requêtes, taux d'erreur
    - Outils: APM (New Relic, DataDog)
```

### ❌ Anti-Pattern 2 : Alertes Sans Action

**Problème** :

```yaml
# Des dizaines d'alertes configurées
# Mais personne ne les traite
# → Alerte fatigue, vraies alertes ignorées
```

**Conséquence** :
- Désensibilisation aux alertes
- Incidents critiques passés inaperçus
- Perte de confiance dans le système d'alerting

**Solution** :

```yaml
# Principe : Chaque alerte doit avoir un runbook

alertes:
  critiques:
    - nom: "ConfigServerDown"
      runbook: "docs/runbooks/config-server-down.md"
      responsable: "équipe-dba"
      escalation: "30 minutes → manager"

    - nom: "HighReplicationLag"
      runbook: "docs/runbooks/replication-lag.md"
      responsable: "équipe-dba"
      escalation: "1 heure → CTO"

  warnings:
    - nom: "JumboChunksDetected"
      runbook: "docs/runbooks/jumbo-chunks.md"
      responsable: "équipe-dba"
      review: "hebdomadaire"
```

### ❌ Anti-Pattern 3 : Maintenance Sans Test

**Problème** :

```bash
# Effectuer une maintenance majeure directement en production
# Sans l'avoir testée en staging
# Exemple : Upgrade MongoDB 5.0 → 7.0 sans test
```

**Conséquence** :
- Incompatibilités découvertes en production
- Rollback complexe
- Downtime prolongé

**Solution** :

```bash
# Pipeline de test rigoureux

# 1. DEV : Test sur cluster minimal
#    - 1 config server
#    - 1 shard (3 membres)
#    - 1 mongos
#    - Résultat : OK

# 2. STAGING : Test sur cluster identique à prod
#    - Restauration snapshot prod
#    - Upgrade complet
#    - Tests fonctionnels
#    - Tests de charge
#    - Résultat : OK, durée estimée: 4h

# 3. PRODUCTION : Exécution planifiée
#    - Fenêtre de maintenance: 02:00-06:00
#    - Équipe complète disponible
#    - Plan de rollback préparé
```

### ❌ Anti-Pattern 4 : Pas de Documentation

**Problème** :

```bash
# Connaissances uniquement dans la tête des admins
# Pas de runbooks
# Pas de documentation des incidents passés
```

**Conséquence** :
- Dépendance aux individus (single point of failure humain)
- Onboarding difficile des nouveaux membres
- Résolution d'incidents plus lente

**Solution** :

```markdown
# Structure de documentation recommandée

docs/
├── architecture/
│   ├── cluster-topology.md
│   ├── shard-key-decisions.md
│   └── network-diagram.png
│
├── runbooks/
│   ├── config-server-down.md
│   ├── shard-failover.md
│   ├── migration-failure.md
│   └── balancer-issues.md
│
├── procedures/
│   ├── rolling-restart.md
│   ├── add-shard.md
│   ├── backup-restore.md
│   └── upgrade-cluster.md
│
└── incidents/
    ├── 2024-01-15-replication-lag.md
    ├── 2024-02-03-jumbo-chunks.md
    └── template-incident-report.md
```

### ❌ Anti-Pattern 5 : Maintenance Pendant les Heures de Pointe

**Problème** :

```bash
# Effectuer un rolling restart à 14h
# Pendant le pic de charge quotidien
```

**Conséquence** :
- Dégradation de performance visible par les utilisateurs
- Timeout applicatifs
- Expérience utilisateur dégradée

**Solution** :

```bash
# Fenêtres de maintenance définies

# Production : 02:00-06:00 UTC (charge minimale)
# Staging : Flexible
# Dev : Flexible

# Exceptions : Incidents critiques uniquement

# Planification :
# - Notification 7 jours avant
# - Fenêtre de 4 heures
# - Équipe complète en standby
# - Rollback plan préparé
```

---

## Checklist de Monitoring

```yaml
checklist_monitoring:

  quotidien:
    - titre: "Santé du cluster"
      commande: "sh.status()"
      fréquence: "Matin et soir"

    - titre: "Distribution des chunks"
      commande: "checkChunkDistribution()"
      fréquence: "Matin"

    - titre: "État du balancer"
      commande: "monitorBalancer()"
      fréquence: "Matin"

    - titre: "Vérification des alertes"
      action: "Review Grafana + Alertmanager"
      fréquence: "Matin"

  hebdomadaire:
    - titre: "Analyse des performances"
      action: "Review slow queries logs"
      fréquence: "Lundi matin"

    - titre: "Jumbo chunks"
      commande: "analyzeJumboChunks()"
      fréquence: "Lundi"

    - titre: "Capacity planning"
      action: "Projection croissance données"
      fréquence: "Lundi"

    - titre: "Review des incidents"
      action: "Analyse incidents de la semaine"
      fréquence: "Vendredi"

  mensuel:
    - titre: "Maintenance préventive"
      commande: "preventiveMaintenance()"
      fréquence: "1er du mois"

    - titre: "Test de restauration"
      action: "Restaurer backup en staging"
      fréquence: "15 du mois"

    - titre: "Audit sécurité"
      action: "Review users, roles, network"
      fréquence: "Dernier vendredi"

    - titre: "Update documentation"
      action: "Mise à jour runbooks et procédures"
      fréquence: "Dernier vendredi"

  trimestriel:
    - titre: "Disaster recovery drill"
      action: "Simulation panne complète"
      fréquence: "T1, T2, T3, T4"

    - titre: "Performance benchmark"
      action: "Tests de charge"
      fréquence: "T1, T2, T3, T4"
```

---

## Conclusion

Le monitoring et la maintenance d'un cluster shardé MongoDB sont des activités continues qui nécessitent rigueur et proactivité. Les points clés à retenir :

- ✅ **Monitoring multi-niveaux** : Infrastructure, MongoDB, Sharding, Application
- ✅ **Alerting intelligent** : Chaque alerte avec runbook et responsable
- ✅ **Maintenance planifiée** : Procédures documentées et testées
- ✅ **Réactivité aux incidents** : Diagnostic rapide et actions documentées
- ✅ **Amélioration continue** : Post-mortems et optimisations
- ✅ **Documentation vivante** : Runbooks, procédures, incidents
- ✅ **Tests réguliers** : DR drills, restaurations, benchmarks

Un cluster bien monitoré et maintenu proactivement présente :
- **Moins d'incidents** grâce à la détection précoce
- **Résolution plus rapide** grâce aux runbooks
- **Meilleure disponibilité** grâce aux maintenances planifiées
- **Performance optimale** grâce au tuning continu

**Investissez dans le monitoring et la maintenance : c'est la clé d'un cluster shardé fiable en production.**

---

## Ressources

- [MongoDB Monitoring Best Practices](https://docs.mongodb.com/manual/administration/monitoring/)
- [Prometheus MongoDB Exporter](https://github.com/percona/mongodb_exporter)
- [MongoDB Ops Manager](https://www.mongodb.com/products/ops-manager)
- [MongoDB Cloud Manager](https://www.mongodb.com/cloud/cloud-manager)

---


⏭️ [Jumbo Chunks et résolution](/10-sharding/11-jumbo-chunks-resolution.md)
