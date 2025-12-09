🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.4 - Configuration Haute Performance

## Présentation

### Objectif
Configuration MongoDB optimisée pour maximiser les performances en lecture et écriture, adaptée aux applications à forte charge transactionnelle, temps réel, IoT, analytics et Big Data.

### Cas d'usage typiques

- **Applications temps réel** : Trading, gaming, IoT
- **Analytics haute fréquence** : Dashboards, monitoring
- **E-commerce** : Catalogues, sessions, carts
- **Réseaux sociaux** : Feeds, notifications, messages
- **Logs et télémétrie** : Collecte massive de données
- **API haute disponibilité** : Forte concurrence, latence faible

### Caractéristiques

- **Throughput élevé** : 10,000+ ops/sec
- **Latence faible** : P95 < 10ms, P99 < 50ms
- **Cache optimisé** : Hit ratio > 95%
- **I/O optimisé** : NVMe, RAID, compression
- **Indexation stratégique** : Covered queries, compound indexes
- **Concurrence maximale** : Connection pooling, write batching

---

## Architecture matérielle optimale

### Serveur haute performance (référence)

| Composant | Spécification | Justification |
|-----------|---------------|---------------|
| **CPU** | 32 cores @ 3.5+ GHz (AMD EPYC / Intel Xeon) | Parallélisation, agrégations |
| **RAM** | 128 GB DDR4 ECC | Cache WiredTiger maximal |
| **Stockage primaire** | 2x 2TB NVMe SSD (RAID 1) | IOPS élevées, latence faible |
| **Stockage logs** | 500 GB SSD SATA | Séparation I/O |
| **Réseau** | 2x 25 Gbps (bonding) | Bande passante, redondance |
| **Alimentation** | Dual PSU | Disponibilité |

### Dimensionnement mémoire

#### Règle de calcul

```
Working Set < 70% RAM disponible pour WiredTiger Cache
Cache WiredTiger = 50% RAM totale - 1 GB
```

#### Exemples pratiques

| RAM Totale | Cache WiredTiger | OS + Buffers | Index | Overhead |
|------------|------------------|--------------|-------|----------|
| 64 GB | 31 GB | 8 GB | 20 GB | 5 GB |
| 128 GB | 63 GB | 15 GB | 40 GB | 10 GB |
| 256 GB | 127 GB | 25 GB | 80 GB | 24 GB |
| 512 GB | 255 GB | 50 GB | 160 GB | 47 GB |

### Stockage

#### Choix du système de fichiers

```bash
# XFS recommandé pour MongoDB
mkfs.xfs -f -d agcount=32 -l size=128m,lazy-count=1 /dev/nvme0n1

# Options de montage optimales
mount -t xfs -o noatime,nobarrier,logbufs=8,logbsize=256k,allocsize=16m /dev/nvme0n1 /data/mongodb
```

#### /etc/fstab

```bash
# Montage optimisé pour performances
/dev/nvme0n1  /data/mongodb  xfs  noatime,nobarrier,logbufs=8,logbsize=256k,allocsize=16m,inode64  0 0

# Logs sur volume séparé
/dev/sda1     /var/log/mongodb  xfs  noatime,nodiratime  0 0
```

#### Configuration RAID

```bash
# RAID 10 pour performances + redondance
# 4x NVMe en RAID 10 = ~800k IOPS lecture, ~400k IOPS écriture

mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1

# Optimisation stripe
echo 8192 > /sys/block/md0/md/stripe_cache_size
```

---

## Configuration système Linux

### Paramètres kernel (/etc/sysctl.conf)

```bash
# === Mémoire et swap ===
vm.swappiness = 1
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
vm.dirty_expire_centisecs = 500
vm.dirty_writeback_centisecs = 100
vm.zone_reclaim_mode = 0
vm.vfs_cache_pressure = 50

# === Réseau ===
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_congestion_control = bbr
net.ipv4.ip_local_port_range = 10000 65535
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728

# === Fichiers ===
fs.file-max = 2097152
fs.nr_open = 2097152
fs.aio-max-nr = 1048576

# === Performances ===
kernel.sched_migration_cost_ns = 5000000
kernel.sched_autogroup_enabled = 0

# Appliquer
sudo sysctl -p
```

### Limites système (/etc/security/limits.conf)

```bash
# Limites élevées pour MongoDB
mongod soft nofile 1000000
mongod hard nofile 1000000
mongod soft nproc 64000
mongod hard nproc 64000
mongod soft memlock unlimited
mongod hard memlock unlimited
mongod soft stack 10240
mongod hard stack 32768

# Vérifier
ulimit -a
```

### Désactivation Transparent Huge Pages (THP)

```bash
# /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=mongod.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never | tee /sys/kernel/mm/transparent_hugepage/enabled > /dev/null'
ExecStart=/bin/sh -c 'echo never | tee /sys/kernel/mm/transparent_hugepage/defrag > /dev/null'
ExecStart=/bin/sh -c 'echo 0 | tee /sys/kernel/mm/transparent_hugepage/khugepaged/defrag > /dev/null'

[Install]
WantedBy=basic.target

# Activer
sudo systemctl daemon-reload
sudo systemctl enable disable-thp
sudo systemctl start disable-thp

# Vérifier
cat /sys/kernel/mm/transparent_hugepage/enabled  # Doit afficher: always madvise [never]
```

### NUMA (serveurs multi-socket)

```bash
# Désactiver NUMA zone reclaim
echo 0 > /proc/sys/vm/zone_reclaim_mode

# Interleave NUMA pour MongoDB
numactl --interleave=all mongod --config /etc/mongod.conf

# OU dans le service systemd
# ExecStart=/usr/bin/numactl --interleave=all /usr/bin/mongod --config /etc/mongod.conf
```

### I/O Scheduler

```bash
# Utiliser 'none' ou 'deadline' pour NVMe
echo none > /sys/block/nvme0n1/queue/scheduler

# Rendre permanent (/etc/udev/rules.d/60-scheduler.rules)
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/scheduler}="deadline"

# Augmenter la queue depth pour NVMe
echo 1024 > /sys/block/nvme0n1/queue/nr_requests
```

---

## Configuration MongoDB optimale

### Configuration haute performance - Lectures intensives

```yaml
# /etc/mongod.conf - Optimisé pour lectures (analytics, queries complexes)

storage:
  dbPath: /data/mongodb
  directoryPerDB: true  # Séparation physique par DB
  journal:
    enabled: true
    commitIntervalMs: 100  # Balance durabilité/performance
  wiredTiger:
    engineConfig:
      cacheSizeGB: 96  # 75% de 128 GB RAM
      journalCompressor: snappy  # Balance compression/CPU
      directoryForIndexes: true  # Index séparés des données
      statisticsLogDelaySecs: 0  # Désactiver en prod
    collectionConfig:
      blockCompressor: snappy  # Rapide, bon ratio
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc
  verbosity: 0
  component:
    query:
      verbosity: 0
    write:
      verbosity: 0

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 100000
  compression:
    compressors: snappy,zstd,zlib
  serviceExecutor: adaptive  # Gestion adaptive des threads

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

replication:
  replSetName: rs-prod
  oplogSizeMB: 51200  # 50 GB pour grande fenêtre

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 50  # Identifier les requêtes > 50ms
  filter: '{ millis: { $gt: 50 } }'

setParameter:
  # Connexions
  maxConns: 100000

  # Performance curseurs
  cursorTimeoutMillis: 600000
  internalQueryExecMaxBlockingSortBytes: 104857600  # 100 MB

  # Cache et mémoire
  wiredTigerConcurrentReadTransactions: 256
  wiredTigerConcurrentWriteTransactions: 256

  # Planificateur de requêtes
  internalQueryPlannerMaxIndexedSolutions: 128
  internalQueryEnumerationMaxOrSolutions: 10
  internalQueryEnumerationMaxIntersectPerAnd: 3

  # Performance agrégations
  internalPipelineLengthLimit: 200

  # Diagnostics
  diagnosticDataCollectionEnabled: true
```

### Configuration haute performance - Écritures intensives

```yaml
# /etc/mongod.conf - Optimisé pour écritures (IoT, logs, événements)

storage:
  dbPath: /data/mongodb
  directoryPerDB: true
  journal:
    enabled: true
    commitIntervalMs: 30  # Commit plus fréquent pour durabilité
  wiredTiger:
    engineConfig:
      cacheSizeGB: 96
      journalCompressor: none  # Pas de compression pour vitesse
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: none  # Trade: espace vs CPU
    indexConfig:
      prefixCompression: false  # Plus rapide sans compression

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc
  verbosity: 0

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 100000
  compression:
    compressors: snappy
  serviceExecutor: adaptive

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

replication:
  replSetName: rs-prod
  oplogSizeMB: 102400  # 100 GB pour absorber burst

operationProfiling:
  mode: off  # Désactiver en écriture intensive

setParameter:
  maxConns: 100000

  # Optimisations écriture
  wiredTigerConcurrentWriteTransactions: 512  # Plus de transactions parallèles
  wiredTigerConcurrentReadTransactions: 128

  # Journal
  journalCommitIntervalMs: 30

  # Checkpoints (flush disque)
  syncdelay: 60  # 60 secondes entre checkpoints

  # Désactiver diagnostics coûteux
  diagnosticDataCollectionEnabled: false

  # Flow control (éviter surcharge)
  flowControlTargetLagSeconds: 10
  flowControlThresholdLagPercentage: 0.5
```

### Configuration équilibrée (mixte)

```yaml
# /etc/mongod.conf - Balance lecture/écriture

storage:
  dbPath: /data/mongodb
  directoryPerDB: true
  journal:
    enabled: true
    commitIntervalMs: 50
  wiredTiger:
    engineConfig:
      cacheSizeGB: 96
      journalCompressor: snappy
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc
  verbosity: 0

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536
  compression:
    compressors: snappy,zstd
  serviceExecutor: adaptive

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

replication:
  replSetName: rs-prod
  oplogSizeMB: 51200

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

setParameter:
  maxConns: 65536
  wiredTigerConcurrentReadTransactions: 256
  wiredTigerConcurrentWriteTransactions: 256
  internalQueryExecMaxBlockingSortBytes: 104857600
  cursorTimeoutMillis: 600000
```

---

## Optimisation des index

### Stratégie d'indexation haute performance

#### Principes

1. **Index couvrants** : Toutes les données dans l'index
2. **Compound index** : Ordre ESR (Equality, Sort, Range)
3. **Index partiels** : Réduire la taille
4. **Index sparse** : Ignorer null/missing
5. **Éviter les index inutiles** : Coût maintenance

#### Exemples d'index optimaux

```javascript
// Collection: users (100M documents)
// Requêtes typiques:
// 1. Recherche par email
// 2. Liste des users actifs par date de création
// 3. Recherche par pays + statut

use myapp

// Index 1: Email unique (équivalent à login)
db.users.createIndex(
  { email: 1 },
  { unique: true, name: "idx_email" }
)

// Index 2: Compound pour requête courante (ESR)
db.users.createIndex(
  { status: 1, createdAt: -1, country: 1 },
  { name: "idx_status_created_country" }
)

// Index 3: Partiel pour users actifs seulement
db.users.createIndex(
  { country: 1, lastLoginAt: -1 },
  {
    partialFilterExpression: { status: "active" },
    name: "idx_active_country_lastlogin"
  }
)

// Index 4: TTL pour sessions expirées
db.sessions.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0, name: "idx_ttl_sessions" }
)
```

#### Index couvrant (Covered Query)

```javascript
// Requête avec projection limitée
db.users.find(
  { email: "john@example.com" },
  { _id: 0, email: 1, name: 1, status: 1 }
)

// Index couvrant optimal
db.users.createIndex(
  { email: 1, name: 1, status: 1 },
  { name: "idx_covered_user_info" }
)

// Vérification
db.users.find(
  { email: "john@example.com" },
  { _id: 0, email: 1, name: 1, status: 1 }
).explain("executionStats")

// totalDocsExamined doit être 0 (covered query)
```

### Maintenance des index

```javascript
// Identifier les index non utilisés
db.users.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 10 } } }
])

// Supprimer les index inutiles
db.users.dropIndex("idx_unused")

// Reconstruire les index fragmentés (offline)
db.users.reIndex()  // ⚠️ Bloquant, à éviter en prod

// Rebuild en arrière-plan (MongoDB 4.2+)
db.users.createIndex({ field: 1 }, { background: true })
```

---

## Read/Write Concerns pour performance

### Read Concern

```javascript
// Lecture locale (plus rapide, moins cohérente)
db.users.find({}).readConcern("local")

// Lecture majoritaire (cohérente, un peu plus lente)
db.users.find({}).readConcern("majority")

// Configuration par défaut dans le driver
const client = new MongoClient(uri, {
  readConcernLevel: "local"  // Privilégier performance
});
```

### Write Concern

```javascript
// Écriture rapide (acknowledge immédiat)
db.users.insertOne(doc, { writeConcern: { w: 1, j: false } })

// Écriture sécurisée (majorité + journal)
db.users.insertOne(doc, { writeConcern: { w: "majority", j: true, wtimeout: 5000 } })

// Batch inserts pour performance
db.users.insertMany(docs, {
  ordered: false,  // Parallélisation
  writeConcern: { w: 1, j: false }
})
```

### Read Preference (Replica Set)

```javascript
// Lecture depuis PRIMARY (cohérence maximale)
db.getMongo().setReadPref("primary")

// Lecture depuis SECONDARY (décharger PRIMARY)
db.getMongo().setReadPref("secondary")

// Lecture depuis le plus proche (latence minimale)
db.getMongo().setReadPref("nearest")

// Préférence avec tags (distribution géographique)
db.getMongo().setReadPref("nearest", [{ datacenter: "eu-west" }])
```

---

## Connection Pooling

### Configuration Node.js

```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb://user:pass@host1,host2,host3/myapp?replicaSet=rs0";

const client = new MongoClient(uri, {
  // Pool de connexions
  maxPoolSize: 200,        // 200 connexions max
  minPoolSize: 50,         // 50 connexions toujours ouvertes

  // Timeouts
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
  connectTimeoutMS: 10000,

  // Keep-alive
  keepAlive: true,
  keepAliveInitialDelay: 300000,

  // Retry
  retryWrites: true,
  retryReads: true,

  // Compression
  compressors: ['snappy', 'zstd'],
  zlibCompressionLevel: 6,

  // Performance
  readConcernLevel: 'local',
  w: 1,
  journal: false
});
```

### Configuration Python

```python
from pymongo import MongoClient
from pymongo.read_preferences import ReadPreference

client = MongoClient(
    "mongodb://user:pass@host1,host2,host3/myapp?replicaSet=rs0",
    maxPoolSize=200,
    minPoolSize=50,
    maxIdleTimeMS=300000,
    serverSelectionTimeoutMS=5000,
    socketTimeoutMS=45000,
    connectTimeoutMS=10000,
    retryWrites=True,
    retryReads=True,
    compressors='snappy,zstd',
    readPreference=ReadPreference.NEAREST
)
```

### Configuration Java (Spring Boot)

```yaml
# application.yml
spring:
  data:
    mongodb:
      uri: mongodb://user:pass@host1,host2,host3/myapp?replicaSet=rs0

      # Connection Pool
      options:
        max-connection-pool-size: 200
        min-connection-pool-size: 50
        max-connection-idle-time: 300000

        # Timeouts
        server-selection-timeout: 5000
        socket-timeout: 45000
        connect-timeout: 10000

        # Performance
        retry-writes: true
        retry-reads: true
        compressors: snappy,zstd
```

---

## Patterns d'optimisation applicative

### Batch Operations

```javascript
// ❌ MAUVAIS: Inserts individuels
for (let i = 0; i < 10000; i++) {
  await db.collection.insertOne({ data: i });
}
// Temps: ~30 secondes

// ✅ BON: Batch insert
const batch = [];
for (let i = 0; i < 10000; i++) {
  batch.push({ data: i });

  if (batch.length === 1000) {
    await db.collection.insertMany(batch, { ordered: false });
    batch.length = 0;
  }
}
if (batch.length > 0) {
  await db.collection.insertMany(batch, { ordered: false });
}
// Temps: ~2 secondes
```

### Bulk Operations

```javascript
// Utiliser bulkWrite pour operations mixtes
const operations = [
  { insertOne: { document: { name: "John" } } },
  { updateOne: { filter: { _id: 1 }, update: { $set: { status: "active" } } } },
  { deleteOne: { filter: { _id: 2 } } }
];

db.collection.bulkWrite(operations, { ordered: false });
```

### Pagination efficace

```javascript
// ❌ MAUVAIS: Skip/Limit (lent sur grands datasets)
db.users.find().skip(1000000).limit(20)

// ✅ BON: Range queries avec index
db.users.find({ _id: { $gt: lastSeenId } }).limit(20).sort({ _id: 1 })

// ✅ BON: Pagination par timestamp
db.users.find({ createdAt: { $lt: lastSeenDate } }).limit(20).sort({ createdAt: -1 })
```

### Projection stratégique

```javascript
// ❌ MAUVAIS: Récupérer tout le document
db.users.find({ email: "john@example.com" })

// ✅ BON: Récupérer uniquement les champs nécessaires
db.users.find(
  { email: "john@example.com" },
  { _id: 1, name: 1, email: 1 }
)
```

### Caching applicatif

```javascript
// Utiliser Redis/Memcached devant MongoDB
const redis = require('redis').createClient();

async function getUser(userId) {
  // Vérifier cache
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached);

  // Sinon, récupérer de MongoDB
  const user = await db.users.findOne({ _id: userId });

  // Mettre en cache (TTL 5 min)
  await redis.setex(`user:${userId}`, 300, JSON.stringify(user));

  return user;
}
```

---

## Monitoring des performances

### Métriques clés à surveiller

```javascript
// 1. Statistiques serveur
db.serverStatus()

// Métriques importantes:
// - opcounters.query, insert, update, delete
// - connections.current, available
// - network.bytesIn, bytesOut
// - mem.resident (RAM utilisée)
// - wiredTiger.cache.bytes currently in the cache
// - wiredTiger.cache.pages evicted by application threads

// 2. Top des opérations lentes
db.currentOp({ secs_running: { $gt: 1 } })

// 3. Profiler (activer temporairement)
db.setProfilingLevel(1, { slowms: 50 })
db.system.profile.find().limit(10).sort({ ts: -1 }).pretty()

// 4. Hit ratio du cache
const stats = db.serverStatus().wiredTiger.cache
const hitRatio = (stats["pages read into cache"] / (stats["pages read into cache"] + stats["pages requested from the cache"])) * 100
print("Cache Hit Ratio: " + hitRatio + "%")  // Objectif: > 95%
```

### Queries de monitoring

```javascript
// Collection stats avec cache info
db.users.stats()

// Index usage statistics
db.users.aggregate([ { $indexStats: {} } ])

// Taille des index
db.users.stats().indexSizes

// Fragmentation
db.users.stats().wiredTiger
```

### Script de monitoring automatisé

```bash
#!/bin/bash
# /usr/local/bin/mongodb-perf-monitor.sh

MONGO_URI="mongodb://admin:password@localhost:27017/admin"

echo "=== MongoDB Performance Report ==="
echo "Date: $(date)"
echo ""

# Connexions
echo "--- Connexions ---"
mongosh "$MONGO_URI" --quiet --eval "
  const stats = db.serverStatus();
  print('Actives: ' + stats.connections.current);
  print('Disponibles: ' + stats.connections.available);
  print('Taux utilisation: ' + (stats.connections.current / (stats.connections.current + stats.connections.available) * 100).toFixed(2) + '%');
"
echo ""

# Operations/sec
echo "--- Operations/sec (dernière minute) ---"
mongosh "$MONGO_URI" --quiet --eval "
  const before = db.serverStatus().opcounters;
  sleep(1000);
  const after = db.serverStatus().opcounters;
  print('Queries: ' + (after.query - before.query));
  print('Inserts: ' + (after.insert - before.insert));
  print('Updates: ' + (after.update - before.update));
  print('Deletes: ' + (after.delete - before.delete));
"
echo ""

# Cache WiredTiger
echo "--- Cache WiredTiger ---"
mongosh "$MONGO_URI" --quiet --eval "
  const wt = db.serverStatus().wiredTiger.cache;
  const cacheSize = wt['bytes currently in the cache'] / (1024**3);
  const maxCache = wt['maximum bytes configured'] / (1024**3);
  print('Utilisé: ' + cacheSize.toFixed(2) + ' GB / ' + maxCache.toFixed(2) + ' GB');
  print('Évictions: ' + wt['pages evicted by application threads']);
  print('Dirty pages: ' + wt['tracked dirty bytes in the cache'] / (1024**2) + ' MB');
"
echo ""

# Slow queries
echo "--- Slow Queries (> 100ms) ---"
mongosh "$MONGO_URI" --quiet --eval "
  db.currentOp({ 'secs_running': { \$gt: 0.1 } }).inprog.forEach(op => {
    if (op.op === 'query' || op.op === 'command') {
      print(op.opid + ': ' + op.op + ' - ' + op.secs_running + 's');
    }
  });
"
```

---

## Benchmarking

### Outil: YCSB (Yahoo! Cloud Serving Benchmark)

```bash
# Installation
git clone https://github.com/brianfrankcooper/YCSB.git
cd YCSB
mvn clean package

# Configuration
cat > mongodb.properties << EOF
mongodb.url=mongodb://localhost:27017/ycsb
mongodb.writeConcern=acknowledged
mongodb.readConcern=local
EOF

# Charger les données (1M records)
./bin/ycsb load mongodb -s -P workloads/workloada -P mongodb.properties -p recordcount=1000000

# Benchmark workload A (50% read, 50% update)
./bin/ycsb run mongodb -s -P workloads/workloada -P mongodb.properties -p operationcount=1000000 -threads 64

# Résultats attendus (serveur performant):
# [OVERALL] Throughput(ops/sec): 80000+
# [READ] AverageLatency(us): < 1000
# [UPDATE] AverageLatency(us): < 1500
```

### Outil: mongoperf

```bash
# Test I/O brut
cat > mongoperf.json << EOF
{
  "nThreads": 16,
  "fileSizeMB": 10000,
  "sleepMicros": 0,
  "mmf": true,
  "r": true,
  "w": true
}
EOF

mongoperf < mongoperf.json

# Résultats attendus (NVMe):
# ops/sec: 100000+
```

### Benchmark custom

```javascript
// benchmark.js - Script de test de charge
const { MongoClient } = require('mongodb');
const client = new MongoClient('mongodb://localhost:27017');

async function benchmark() {
  await client.connect();
  const db = client.db('benchmark');
  const collection = db.collection('test');

  // Préparer les données
  console.log('Préparation...');
  await collection.deleteMany({});
  const docs = Array.from({ length: 100000 }, (_, i) => ({
    _id: i,
    name: `User ${i}`,
    email: `user${i}@example.com`,
    age: Math.floor(Math.random() * 50) + 20,
    tags: ['tag1', 'tag2', 'tag3']
  }));
  await collection.insertMany(docs, { ordered: false });

  // Benchmark lectures
  console.log('Benchmark lectures...');
  const readStart = Date.now();
  for (let i = 0; i < 10000; i++) {
    await collection.findOne({ _id: Math.floor(Math.random() * 100000) });
  }
  const readTime = Date.now() - readStart;
  console.log(`Lectures: ${(10000 / (readTime / 1000)).toFixed(0)} ops/sec`);

  // Benchmark écritures
  console.log('Benchmark écritures...');
  const writeStart = Date.now();
  for (let i = 0; i < 10000; i++) {
    await collection.updateOne(
      { _id: Math.floor(Math.random() * 100000) },
      { $set: { lastUpdate: new Date() } }
    );
  }
  const writeTime = Date.now() - writeStart;
  console.log(`Écritures: ${(10000 / (writeTime / 1000)).toFixed(0)} ops/sec`);

  await client.close();
}

benchmark().catch(console.error);
```

---

## Cas d'usage spécifiques

### IoT / Time Series (écritures massives)

```javascript
// Collection optimisée pour time series
db.createCollection("sensor_data", {
  timeseries: {
    timeField: "timestamp",
    metaField: "sensorId",
    granularity: "seconds"
  },
  expireAfterSeconds: 2592000  // 30 jours
})

// Index minimal
db.sensor_data.createIndex({ "sensorId": 1, "timestamp": -1 })

// Insertion par batch
const batch = [];
for (let i = 0; i < 1000; i++) {
  batch.push({
    sensorId: "SENSOR_001",
    timestamp: new Date(),
    temperature: Math.random() * 30,
    humidity: Math.random() * 100
  });
}
db.sensor_data.insertMany(batch, { ordered: false, writeConcern: { w: 1, j: false } })
```

### Analytics / Dashboards (lectures complexes)

```javascript
// Vues matérialisées pour agrégations fréquentes
db.createView("daily_stats", "events", [
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 86400000) }
    }
  },
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
      count: { $sum: 1 },
      totalValue: { $sum: "$value" }
    }
  }
])

// Utilisation
db.daily_stats.find()  // Beaucoup plus rapide que l'agrégation complète

// Index pour agrégations
db.events.createIndex({ createdAt: -1, category: 1, value: 1 })
```

### E-commerce (mixte, latence critique)

```javascript
// Modélisation optimisée
// Document produit avec données dénormalisées
{
  _id: ObjectId("..."),
  sku: "PROD-001",
  name: "Laptop Pro 15",
  price: 1299.99,
  stock: 50,
  category: "Electronics",
  // Dénormaliser les infos fréquemment accédées
  images: ["url1", "url2"],
  ratings: {
    average: 4.5,
    count: 1234
  },
  // Index pour recherche
  tags: ["laptop", "computer", "pro"]
}

// Index optimaux
db.products.createIndex({ sku: 1 }, { unique: true })
db.products.createIndex({ category: 1, price: 1 })
db.products.createIndex({ tags: 1 })
db.products.createIndex({ "ratings.average": -1 })

// Read preference secondary pour recherche
db.getMongo().setReadPref("secondaryPreferred")
```

---

## Checklist optimisation

### Infrastructure

- [ ] SSD NVMe pour données
- [ ] RAID 10 ou supérieur
- [ ] 128 GB+ RAM (cache >= 64 GB)
- [ ] 32+ CPU cores
- [ ] Réseau 10+ Gbps
- [ ] XFS avec options optimisées

### Système

- [ ] THP désactivé
- [ ] NUMA configuré (si multi-socket)
- [ ] ulimit ajusté (1M+ files)
- [ ] vm.swappiness = 1
- [ ] I/O scheduler = none (NVMe)
- [ ] TCP/IP tuning appliqué

### MongoDB

- [ ] Cache WiredTiger >= 50% RAM
- [ ] Journal compression appropriée
- [ ] Block compression évaluée
- [ ] maxConns dimensionné
- [ ] serviceExecutor = adaptive
- [ ] Profiler désactivé (ou slowms optimisé)

### Indexation

- [ ] Index sur tous les champs de requête
- [ ] Compound index respectant ESR
- [ ] Index couvrants pour requêtes fréquentes
- [ ] Index partiels si applicable
- [ ] Index inutiles supprimés
- [ ] Index statistiques vérifiées

### Application

- [ ] Connection pooling configuré
- [ ] Batch operations utilisées
- [ ] Projection limitée aux champs nécessaires
- [ ] Pagination efficace (range queries)
- [ ] Write concern optimisé
- [ ] Read preference appropriée
- [ ] Caching applicatif en place

### Monitoring

- [ ] Métriques temps réel activées
- [ ] Alertes configurées
- [ ] Cache hit ratio > 95%
- [ ] P95 latency < 50ms
- [ ] Slow queries identifiées
- [ ] Dashboards opérationnels

---

## Dépannage performance

### Symptôme : Latence élevée

```javascript
// 1. Identifier les slow queries
db.currentOp({ secs_running: { $gt: 1 } })

// 2. Analyser avec explain
db.collection.find(query).explain("executionStats")

// 3. Vérifier les index
db.collection.getIndexes()

// 4. Statistiques d'utilisation des index
db.collection.aggregate([ { $indexStats: {} } ])
```

### Symptôme : CPU élevé

```bash
# Vérifier les opérations actives
mongosh --eval "db.currentOp({ secs_running: { \$gt: 0.5 } })"

# Profiler les requêtes
db.setProfilingLevel(2, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 }).limit(10)

# Causes communes:
# - Absence d'index
# - Collection scans
# - Agrégations complexes sans index
# - Sorts en mémoire
```

### Symptôme : Cache hit ratio faible

```javascript
// Vérifier la taille du cache
const stats = db.serverStatus().wiredTiger.cache
print("Cache size: " + (stats['bytes currently in the cache'] / (1024**3)).toFixed(2) + " GB")
print("Max cache: " + (stats['maximum bytes configured'] / (1024**3)).toFixed(2) + " GB")

// Si cache trop petit: augmenter cacheSizeGB
// Si working set > cache: ajouter RAM ou optimiser requêtes
```

### Symptôme : Throughput limité

```bash
# Vérifier les limites I/O
iostat -x 1

# Vérifier les connexions
mongosh --eval "db.serverStatus().connections"

# Solutions:
# - Augmenter maxConns
# - Optimiser connection pooling
# - Utiliser batch operations
# - Sharding si single server saturé
```

---

## Ressources

### Documentation officielle

- [Production Notes](https://docs.mongodb.com/manual/administration/production-notes/)
- [Performance Best Practices](https://www.mongodb.com/basics/best-practices)
- [WiredTiger Tuning](https://docs.mongodb.com/manual/core/wiredtiger/)

### Outils

- [YCSB](https://github.com/brianfrankcooper/YCSB)
- [mongoperf](https://docs.mongodb.com/database-tools/mongoperf/)
- [pt-mongodb-query-digest](https://www.percona.com/doc/percona-toolkit/LATEST/pt-mongodb-query-digest.html)

### Commandes rapides

```bash
# Top opérations
mongosh --eval "db.currentOp({ secs_running: { \$gt: 0.1 } })"

# Cache stats
mongosh --eval "db.serverStatus().wiredTiger.cache"

# Kill slow query
mongosh --eval "db.killOp(12345)"
```

---

**Configuration optimisée pour** : Haute performance (10k+ ops/sec)
**Versions testées** : MongoDB 6.0+, 7.0+, 8.0+

⏭️ [Checklist de Performance](/annexes/checklist-performance/README.md)
