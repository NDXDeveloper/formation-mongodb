🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.1 Métriques clés à surveiller

## Introduction

La surveillance efficace de MongoDB repose sur la compréhension et le suivi de métriques critiques qui révèlent l'état de santé, les performances et la capacité du système. Cette section détaille les métriques essentielles que tout SRE ou administrateur système doit surveiller en production, avec des seuils recommandés et des stratégies d'analyse.

## Classification des métriques

Les métriques MongoDB se répartissent en plusieurs catégories interdépendantes :

```
┌────────────────────────────────────────────────────────┐
│                    MÉTRIQUES MONGODB                   │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ SYSTÈME      │  │ CONNEXIONS   │  │ OPÉRATIONS   │  │
│  │ CPU, RAM     │  │ Active/Total │  │ CRUD, Aggr.  │  │
│  │ Disk I/O     │  │ Pool exhaust │  │ Throughput   │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ MÉMOIRE      │  │ RÉPLICATION  │  │ SHARDING     │  │
│  │ Cache WT     │  │ Lag, Health  │  │ Chunks       │  │
│  │ Page faults  │  │ Oplog window │  │ Balancing    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ LATENCE      │  │ LOCKS        │  │ STOCKAGE     │  │
│  │ P50/P95/P99  │  │ Contentions  │  │ Taille DB    │  │
│  │ Query time   │  │ Wait time    │  │ Fragmentation│  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└────────────────────────────────────────────────────────┘
```

---

## 1. Métriques système et infrastructure

### 1.1 Utilisation CPU

**Définition** : Pourcentage d'utilisation du processeur par le processus mongod.

**Sources de collecte** :
```bash
# Via top/htop
top -p $(pgrep mongod)

# Via serverStatus
db.serverStatus().extra_info.user_time_us
db.serverStatus().extra_info.system_time_us

# Via node_exporter (Prometheus)
process_cpu_seconds_total{job="mongodb"}
```

**Métriques détaillées** :

| Métrique | Description | Seuil Warning | Seuil Critical |
|----------|-------------|---------------|----------------|
| `%user` | CPU utilisateur (MongoDB) | > 70% | > 85% |
| `%system` | CPU système (I/O, kernel) | > 30% | > 50% |
| `%iowait` | Attente I/O disque | > 20% | > 40% |
| `%steal` | CPU volé (cloud/VM) | > 10% | > 20% |

**Analyse d'exemple** :

```javascript
// Scénario : CPU user à 85%, iowait à 5%
// Diagnostic : Charge CPU liée au traitement, pas aux I/O

db.currentOp({
  active: true,
  secs_running: { $gt: 5 }
})

// Rechercher :
// - Collection scans (planSummary: "COLLSCAN")
// - Agrégations complexes
// - Index manquants
```

**Actions recommandées** :
- **70-85% CPU** : Analyser les slow queries, optimiser les index
- **> 85% CPU** : Scaling vertical (plus de CPU) ou horizontal (sharding)
- **iowait > 20%** : Vérifier les disques (latence, IOPS), considérer des SSD

### 1.2 Mémoire système

**Définition** : Utilisation de la RAM par le système et MongoDB.

**Métriques clés** :

```javascript
db.serverStatus().mem
{
  "bits": 64,
  "resident": 4567,      // RAM physique utilisée (MB)
  "virtual": 8234,       // Mémoire virtuelle
  "supported": true,
  "mapped": 0,
  "mappedWithJournal": 0
}
```

**Sources complémentaires** :
```bash
# Mémoire disponible sur le système
free -h

# RSS (Resident Set Size) du processus mongod
ps aux | grep mongod | awk '{print $6/1024 " MB"}'

# Pression mémoire (Linux)
cat /proc/pressure/memory
```

**Seuils recommandés** :

| Métrique | Warning | Critical | Action |
|----------|---------|----------|--------|
| RAM disponible | < 20% | < 10% | Augmenter RAM ou optimiser cache |
| Swap utilisé | > 0 MB | > 100 MB | Désactiver swap ou augmenter RAM |
| Page faults/sec | > 100 | > 500 | Cache undersized, voir section 1.4 |

**Exemple d'analyse - Swap détecté** :

```bash
# Vérifier si MongoDB utilise le swap
cat /proc/$(pgrep mongod)/status | grep Swap
# Swap: 256000 kB  ⚠️ CRITIQUE

# Diagnostic : Cache WiredTiger trop grand ou RAM insuffisante
```

**Actions correctives** :
1. Réduire la taille du cache WiredTiger (50% RAM par défaut)
2. Ajouter de la RAM physique
3. Désactiver le swap complètement : `swapoff -a` (après validation)

### 1.3 I/O disque

**Définition** : Opérations de lecture/écriture sur le stockage persistant.

**Métriques principales** :

```bash
# IOPS (I/O Operations Per Second)
iostat -x 1

# Résultat typique :
Device    r/s    w/s    rkB/s    wkB/s   await  %util
nvme0n1   450    1200   12800    48000   8.5    78%

# Latence de lecture/écriture
# await = temps moyen par opération (ms)
# %util = saturation du disque
```

**Métriques WiredTiger** :

```javascript
db.serverStatus().wiredTiger.block_manager
{
  "blocks read": 245789234,
  "blocks written": 567891234,
  "bytes read": 1234567890123,
  "bytes written": 4567890123456,
  "bytes written for checkpoint": 3456789012345
}
```

**Seuils critiques** :

| Métrique | SSD (NVMe) | SSD (SATA) | HDD | Action |
|----------|------------|------------|-----|--------|
| Latence lecture | < 5ms | < 10ms | < 20ms | Acceptable |
| Latence écriture | < 10ms | < 20ms | < 50ms | Acceptable |
| %util | < 80% | < 70% | < 60% | Saturation proche |
| IOPS | > 50K | > 10K | > 200 | Performances typiques |

**Analyse d'exemple - Disque saturé** :

```javascript
// Symptômes : await > 50ms, %util > 90%

// 1. Identifier les opérations I/O intensives
db.currentOp({
  waitingForLock: false,
  secs_running: { $gt: 3 }
}).inprog.forEach(op => {
  print(`Op: ${op.op}, NS: ${op.ns}, Query: ${JSON.stringify(op.command)}`)
})

// 2. Vérifier la compression (taux de lecture/écriture)
var stats = db.serverStatus().wiredTiger.block_manager
var readGB = stats["bytes read"] / (1024*1024*1024)
var writtenGB = stats["bytes written"] / (1024*1024*1024)
print(`Read: ${readGB.toFixed(2)} GB, Written: ${writtenGB.toFixed(2)} GB`)

// 3. Vérifier les checkpoints
db.serverStatus().wiredTiger.transaction.transaction_checkpoint_currently_running
// Si true pendant longtemps → Problème de performance disque
```

**Actions correctives** :
- Migrer vers des disques plus rapides (SSD NVMe)
- Activer/optimiser la compression WiredTiger
- Ajouter des index pour réduire les scans
- Augmenter le cache pour réduire les disk reads

---

## 2. Métriques de connexions

### 2.1 Connexions actives vs disponibles

**Définition** : Nombre de connexions clients au serveur MongoDB.

**Collecte** :

```javascript
db.serverStatus().connections
{
  "current": 847,        // Connexions actuellement établies
  "available": 51153,    // Connexions restantes disponibles
  "totalCreated": 245789, // Total créé depuis le démarrage
  "active": 23,          // Connexions actives (requêtes en cours)
  "threaded": 847,
  "exhaustIsMaster": 0,
  "exhaustHello": 0,
  "awaitingTopologyChanges": 0
}
```

**Métriques dérivées** :

```javascript
// Taux d'utilisation des connexions
var connStats = db.serverStatus().connections
var usage = (connStats.current / (connStats.current + connStats.available)) * 100
print(`Connection pool usage: ${usage.toFixed(2)}%`)

// Taux de création de connexions (churn)
// Collecter totalCreated à T0 et T1 (1 minute d'écart)
var churnRate = (totalCreatedT1 - totalCreatedT0) / 60  // connexions/sec
```

**Seuils d'alerte** :

| Métrique | Warning | Critical | Impact |
|----------|---------|----------|--------|
| Usage pool | > 70% | > 85% | Nouvelles connexions refusées |
| Connexions actives | > 100 | > 500 | Charge élevée ou slow queries |
| Churn rate | > 10/s | > 50/s | Connection pooling mal configuré |

**Analyse d'exemple - Pool exhaustion** :

```javascript
// Scénario : current = 50500, available = 500 (98% utilisé)

// 1. Identifier les connexions inactives
db.currentOp({ active: false, secs_running: { $gt: 300 } }).inprog.length

// 2. Identifier les connexions par client
db.currentOp().inprog.reduce((acc, op) => {
  var client = op.client || "unknown"
  acc[client] = (acc[client] || 0) + 1
  return acc
}, {})

// Résultat :
// { "10.0.1.45:34567": 1234,  ⚠️ Un client monopolise les connexions
//   "10.0.2.78:45678": 45,
//   "10.0.3.12:56789": 23 }
```

**Actions correctives** :
1. **Court terme** : Tuer les connexions inactives avec `db.killOp(opId)`
2. **Moyen terme** : Optimiser le connection pooling applicatif
3. **Long terme** : Augmenter `maxIncomingConnections` (par défaut 65536)

**Configuration recommandée** :

```yaml
# mongod.conf
net:
  maxIncomingConnections: 10000  # Ajuster selon la charge

# Application (exemple Node.js)
const client = new MongoClient(uri, {
  maxPoolSize: 100,        // Connexions max par instance
  minPoolSize: 10,         // Connexions min maintenues
  maxIdleTimeMS: 30000,    // Fermer connexions inactives après 30s
  waitQueueTimeoutMS: 5000 // Timeout si pool saturé
})
```

### 2.2 Connexions actives et queued

**Définition** : Connexions traitant activement des opérations vs en attente.

```javascript
// Via currentOp
var active = db.currentOp({ active: true }).inprog.length
var waiting = db.currentOp({ waitingForLock: true }).inprog.length

// Via serverStatus
db.serverStatus().globalLock.activeClients
{
  "total": 45,     // Clients actifs
  "readers": 32,   // Opérations de lecture
  "writers": 13    // Opérations d'écriture
}

db.serverStatus().globalLock.currentQueue
{
  "total": 12,     // Opérations en attente de lock
  "readers": 8,
  "writers": 4
}
```

**Seuils d'alerte** :

| Métrique | Normal | Warning | Critical |
|----------|--------|---------|----------|
| Active clients | < 100 | 100-500 | > 500 |
| Queued operations | 0-5 | 5-20 | > 20 |
| Ratio queued/active | < 0.1 | 0.1-0.3 | > 0.3 |

**Analyse d'exemple - Queue saturation** :

```javascript
// Scénario : currentQueue.total = 45, activeClients = 60

// Identifier la cause de la contention
db.currentOp({
  $or: [
    { waitingForLock: true },
    { "locks.Global": { $eq: "w" } }  // Write lock global
  ]
}).inprog.forEach(op => {
  print(`${op.op} on ${op.ns} - Waiting: ${op.waitingForLock}`)
  print(`Lock stats: ${JSON.stringify(op.locks)}`)
  print(`Secs running: ${op.secs_running}`)
  print("---")
})

// Résultat typique :
// remove on mydb.largeColl - Waiting: false
// Lock stats: {"Global":"w","Database":"w","Collection":"w"}
// Secs running: 45  ⚠️ Opération lente bloquant les autres
```

**Actions correctives** :
- Identifier et optimiser l'opération lente (`db.killOp()` si nécessaire)
- Vérifier les index manquants
- Éviter les opérations bloquantes aux heures de pointe
- Considérer le read/write splitting (lire depuis secondaries)

---

## 3. Métriques d'opérations (Throughput)

### 3.1 Compteurs d'opérations (opcounters)

**Définition** : Nombre cumulatif d'opérations par type depuis le démarrage.

```javascript
db.serverStatus().opcounters
{
  "insert": 45789234,
  "query": 234567890,
  "update": 67890123,
  "delete": 4567890,
  "getmore": 12345678,
  "command": 456789012
}

// Pour Replica Set, opcounters depuis réplication
db.serverStatus().opcountersRepl
{
  "insert": 23456789,
  "query": 0,           // Les secondaires ne reçoivent pas de queries directes
  "update": 34567890,
  "delete": 2345678,
  "getmore": 0,
  "command": 567890
}
```

**Calcul du throughput (ops/sec)** :

```javascript
// Collecter opcounters à T0 et T1 (intervalles de 60 secondes)
function calculateOpsPerSec(t0, t1, deltaTime) {
  var ops = {}
  for (var key in t1) {
    ops[key] = (t1[key] - t0[key]) / deltaTime
  }
  return ops
}

// Exemple de résultat :
// { insert: 450, query: 1200, update: 350, delete: 50, command: 2500 }
// Throughput total : ~4550 ops/sec
```

**Patterns et anomalies à surveiller** :

| Pattern | Caractéristique | Action |
|---------|-----------------|--------|
| **Spike soudain** | `insert` passe de 100/s à 5000/s | Vérifier batch job ou incident |
| **Query ratio élevé** | `query` >> `insert + update` | Vérifier caching applicatif |
| **Delete massif** | `delete` > 1000/s prolongé | Peut créer fragmentation |
| **Command explosif** | `command` >> autres ops | Analyser avec profiler |

**Analyse d'exemple - Insert spike** :

```javascript
// Scénario : insert passe de 200/s à 8000/s brutalement

// 1. Identifier la collection cible
db.currentOp({ op: "insert" }).inprog.forEach(op => {
  print(`Insert on: ${op.ns}, docs: ${op.command.documents.length}`)
})

// 2. Vérifier l'impact sur la performance
var wt = db.serverStatus().wiredTiger
print(`Cache usage: ${wt.cache["bytes currently in the cache"] / wt.cache["maximum bytes configured"] * 100}%`)
print(`Pages evicted: ${wt.cache["pages evicted by application threads"]}`)

// 3. Vérifier si c'est du bulk insert
db.getSiblingDB("admin").aggregate([
  { $currentOp: { allUsers: true } },
  { $match: { op: "insert" } },
  { $group: { _id: "$client", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
```

### 3.2 Latences des opérations

**Définition** : Temps d'exécution des opérations par type.

```javascript
db.serverStatus().opLatencies
{
  "reads": {
    "latency": 234567890,  // Latence cumulée en microsecondes
    "ops": 12345678        // Nombre total d'opérations
  },
  "writes": {
    "latency": 456789012,
    "ops": 8901234
  },
  "commands": {
    "latency": 789012345,
    "ops": 23456789
  }
}
```

**Calcul de latence moyenne** :

```javascript
function calculateAvgLatency(opLatencies) {
  var results = {}
  for (var opType in opLatencies) {
    var data = opLatencies[opType]
    // Convertir microsecondes en millisecondes
    var avgMs = (data.latency / data.ops / 1000).toFixed(2)
    results[opType] = avgMs + " ms"
  }
  return results
}

// Résultat :
// { reads: "19.00 ms", writes: "51.30 ms", commands: "33.60 ms" }
```

**Seuils recommandés (P95)** :

| Type opération | Acceptable | Warning | Critical |
|----------------|-----------|---------|----------|
| Reads | < 50ms | 50-100ms | > 100ms |
| Writes | < 100ms | 100-200ms | > 200ms |
| Commands | < 50ms | 50-150ms | > 150ms |

**Important** : Ces latences sont des **moyennes globales**. Pour les percentiles (P95, P99), utiliser le profiler ou Atlas Performance Advisor.

**Analyse d'exemple - Write latency élevée** :

```javascript
// Scénario : writes latency avg = 250ms (> 200ms threshold)

// 1. Activer le profiler pour capturer les slow writes
db.setProfilingLevel(1, { slowms: 100 })

// 2. Analyser les slow operations récentes
db.system.profile.find({
  op: { $in: ["insert", "update", "delete"] },
  millis: { $gt: 100 }
}).sort({ ts: -1 }).limit(10).forEach(op => {
  print(`Op: ${op.op} on ${op.ns}`)
  print(`Duration: ${op.millis}ms`)
  print(`Plan: ${op.execStats ? op.execStats.stage : "N/A"}`)
  print(`Docs examined: ${op.docsExamined}`)
  print("---")
})

// 3. Vérifier Write Concern
db.serverStatus().wc
// Si writeConcern: { w: "majority", j: true }
// → Latence attendue plus élevée pour durabilité
```

---

## 4. Métriques de mémoire et cache WiredTiger

### 4.1 Cache WiredTiger

**Définition** : Mémoire dédiée au cache de données pour éviter les lectures disque.

```javascript
db.serverStatus().wiredTiger.cache
{
  "maximum bytes configured": 4294967296,          // 4 GB (50% RAM par défaut)
  "bytes currently in the cache": 3758096384,     // 3.5 GB utilisé
  "tracked dirty bytes in the cache": 524288000,  // 500 MB de données modifiées
  "bytes read into cache": 123456789012,
  "bytes written from cache": 234567890123,
  "pages evicted by application threads": 45678,  // Évictions forcées
  "pages read into cache": 12345678,
  "pages written from cache": 23456789,
  "eviction worker thread active": 4,
  "eviction server candidate queue empty": 89,
  "eviction server candidate queue not empty": 11
}
```

**Métriques clés** :

```javascript
// Calcul du taux d'utilisation du cache
var cache = db.serverStatus().wiredTiger.cache
var usagePercent = (cache["bytes currently in the cache"] / cache["maximum bytes configured"]) * 100
print(`Cache usage: ${usagePercent.toFixed(2)}%`)

// Calcul du taux de dirty data
var dirtyPercent = (cache["tracked dirty bytes in the cache"] / cache["bytes currently in the cache"]) * 100
print(`Dirty data: ${dirtyPercent.toFixed(2)}%`)

// Hit ratio (nécessite monitoring dans le temps)
// hit_ratio = (pages_in_cache - pages_evicted) / pages_in_cache
```

**Seuils d'alerte** :

| Métrique | Warning | Critical | Impact |
|----------|---------|----------|--------|
| Cache usage | > 85% | > 95% | Évictions fréquentes |
| Dirty data % | > 20% | > 50% | Checkpoints lents |
| Pages evicted/sec | > 100 | > 1000 | Cache undersized |
| Read into cache GB/h | Établir baseline | Spike x2 | Changement de workload |

**Analyse d'exemple - Cache thrashing** :

```javascript
// Scénario : Cache à 98%, pages evicted en constante augmentation

// 1. Identifier le working set size
db.serverStatus().wiredTiger.cache["bytes currently in the cache"] / (1024*1024*1024)
// Si > 90% de la RAM allouée → Working set trop grand

// 2. Analyser les collections qui consomment le plus de cache
db.adminCommand({ serverStatus: 1 }).wiredTiger.cache

// 3. Vérifier si des scans de collections entières se produisent
db.currentOp({
  "planSummary": "COLLSCAN",
  secs_running: { $gt: 5 }
}).inprog.forEach(op => {
  print(`Collection scan on: ${op.ns}`)
  print(`Documents examined: ${op.docsExamined}`)
})
```

**Actions correctives** :
1. **Court terme** : Ajouter des index pour éviter les scans
2. **Moyen terme** : Augmenter la RAM disponible
3. **Long terme** : Archiver les données anciennes, considérer le sharding

**Configuration du cache** :

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # Par défaut : 50% RAM - 1GB ou 256MB
```

### 4.2 Page Faults

**Définition** : Accès à des pages mémoire qui ne sont pas en RAM (nécessitent lecture disque).

```javascript
db.serverStatus().extra_info
{
  "note": "fields vary by platform",
  "page_faults": 45678   // Nombre de page faults depuis le démarrage
}
```

**Calcul du taux de page faults** :

```bash
# Collecter page_faults à intervalle régulier
# Rate = (page_faults_t1 - page_faults_t0) / delta_time_seconds

# Exemple avec script :
#!/bin/bash
PREV_FAULTS=$(mongo --quiet --eval "db.serverStatus().extra_info.page_faults")
sleep 60
CURR_FAULTS=$(mongo --quiet --eval "db.serverStatus().extra_info.page_faults")
RATE=$(( ($CURR_FAULTS - $PREV_FAULTS) / 60 ))
echo "Page faults/sec: $RATE"
```

**Seuils** :

| Taux page faults/sec | État | Action |
|----------------------|------|--------|
| 0-10 | ✅ Normal | Aucune |
| 10-100 | ⚠️ Warning | Monitorer tendance |
| 100-500 | 🔥 Critical | Augmenter cache/RAM |
| > 500 | 💀 Emergency | Working set >> RAM disponible |

**Corrélation avec autres métriques** :

```javascript
// Script d'analyse de corrélation
function analyzeMemoryPressure() {
  var status = db.serverStatus()
  var cache = status.wiredTiger.cache
  var pageFaults = status.extra_info.page_faults

  return {
    cacheUsagePercent: (cache["bytes currently in the cache"] / cache["maximum bytes configured"] * 100).toFixed(2),
    pagesEvicted: cache["pages evicted by application threads"],
    pageFaults: pageFaults,
    diagnosis: cache["bytes currently in the cache"] / cache["maximum bytes configured"] > 0.9 &&
               cache["pages evicted by application threads"] > 10000 ?
               "CRITICAL: Cache under pressure, frequent evictions and page faults" : "Normal"
  }
}

analyzeMemoryPressure()
```

---

## 5. Métriques de réplication

### 5.1 Replication Lag

**Définition** : Retard entre le primary et les secondaries dans l'application des opérations.

```javascript
// Sur le PRIMARY
rs.printSlaveReplicationInfo()

// Résultat :
source: secondary1.example.com:27017
    syncedTo: Mon Dec 08 2025 15:42:30 GMT+0100
    0 secs (0 hrs) behind the primary

source: secondary2.example.com:27017
    syncedTo: Mon Dec 08 2025 15:42:15 GMT+0100
    15 secs (0.00 hrs) behind the primary  ⚠️
```

**Méthode programmatique** :

```javascript
// Récupérer le lag pour chaque membre
function getReplicationLag() {
  var status = rs.status()
  var primaryOptime = null
  var lags = []

  status.members.forEach(member => {
    if (member.stateStr === "PRIMARY") {
      primaryOptime = member.optimeDate
    }
  })

  status.members.forEach(member => {
    if (member.stateStr === "SECONDARY") {
      var lagMs = primaryOptime - member.optimeDate
      lags.push({
        host: member.name,
        lagSeconds: (lagMs / 1000).toFixed(2),
        state: member.stateStr,
        health: member.health
      })
    }
  })

  return lags
}

getReplicationLag()
```

**Seuils d'alerte** :

| Lag | Niveau | Impact | Action |
|-----|--------|--------|--------|
| < 5s | ✅ Normal | Aucun | Monitoring continu |
| 5-30s | ⚠️ Warning | Read preference peut être impacté | Investiguer |
| 30s-5min | 🔥 Critical | Données obsolètes sur secondaries | Escalade |
| > 5min | 💀 Emergency | Risque de rollback si failover | Incident majeur |

**Causes communes de replication lag** :

```javascript
// 1. Vérifier la charge sur le secondary
db.serverStatus().metrics.repl.network
{
  "bytes": 123456789012,
  "getmores": {
    "num": 234567,
    "totalMillis": 45678901  // Temps passé à récupérer oplog
  },
  "ops": 456789012
}

// 2. Vérifier l'oplog window
var oplog = db.getSiblingDB("local").oplog.rs
var firstOp = oplog.find().sort({$natural: 1}).limit(1).next()
var lastOp = oplog.find().sort({$natural: -1}).limit(1).next()
var oplogWindowHours = (lastOp.ts.getTime() - firstOp.ts.getTime()) / 1000 / 3600

print(`Oplog window: ${oplogWindowHours.toFixed(2)} hours`)

// Si < 24h → Oplog trop petit pour le taux de write

// 3. Vérifier si le secondary est overloaded
db.currentOp({
  secs_running: { $gt: 10 },
  $or: [
    { op: "query" },
    { op: "getmore" }
  ]
})
// Des queries lentes peuvent ralentir la réplication
```

**Actions correctives** :
1. **Augmenter la taille de l'oplog** (nécessite downtime par membre)
2. **Réduire la charge de lecture sur secondaries** (read preference)
3. **Optimiser les index** sur les secondaries
4. **Upgrader le matériel** (CPU, disque) du secondary lent

### 5.2 Oplog Window

**Définition** : Durée pendant laquelle les opérations sont conservées dans l'oplog.

```javascript
db.printReplicationInfo()

// Résultat :
configured oplog size:   10240MB
log length start to end: 25634secs (7.12hrs)
oplog first event time:  Mon Dec 08 2025 08:30:15 GMT+0100
oplog last event time:   Mon Dec 08 2025 15:42:29 GMT+0100
now:                     Mon Dec 08 2025 15:42:30 GMT+0100
```

**Calcul du taux de consommation** :

```javascript
// Collecter à T0 et T1 (intervalle 1 heure)
function calculateOplogGrowthRate() {
  var oplog = db.getSiblingDB("local").oplog.rs
  var stats = oplog.stats()

  return {
    sizeGB: (stats.size / (1024*1024*1024)).toFixed(2),
    maxSizeGB: (stats.maxSize / (1024*1024*1024)).toFixed(2),
    utilizationPercent: (stats.size / stats.maxSize * 100).toFixed(2)
  }
}

// Après 1 heure, recalculer et comparer
// Growth rate = (sizeT1 - sizeT0) / intervalle
```

**Recommandations** :

| Workload | Oplog size | Window cible |
|----------|------------|--------------|
| Lecture intensive | 5-10% RAM | 24-48h |
| Écriture intensive | 10-20% RAM | 48-72h |
| Batch jobs | 20-30% RAM | 72h+ |

**Alerte critique** :

```javascript
// Si oplog window < 2x le replication lag maximum toléré
// Exemple : Si lag max acceptable = 1h, window doit être > 2h

function checkOplogHealth() {
  var repl = db.printReplicationInfo()
  var windowHours = /* extraire de repl */
  var maxLagSeconds = getReplicationLag().reduce((max, m) =>
    Math.max(max, parseFloat(m.lagSeconds)), 0
  )

  if (windowHours < 2 * (maxLagSeconds / 3600)) {
    return {
      status: "CRITICAL",
      message: "Oplog window too small relative to replication lag",
      windowHours: windowHours,
      maxLagHours: maxLagSeconds / 3600
    }
  }
  return { status: "OK" }
}
```

### 5.3 État des membres (Replica Set Health)

**Définition** : Santé de chaque membre du Replica Set.

```javascript
rs.status()

// Résultat condensé :
{
  "set": "rs0",
  "members": [
    {
      "name": "primary.example.com:27017",
      "health": 1,              // 1 = healthy, 0 = down
      "state": 1,               // 1 = PRIMARY
      "stateStr": "PRIMARY",
      "uptime": 3456789,
      "lastHeartbeat": ISODate("2025-12-08T14:42:28Z"),
      "lastHeartbeatRecv": ISODate("2025-12-08T14:42:29Z"),
      "pingMs": 0
    },
    {
      "name": "secondary1.example.com:27017",
      "health": 1,
      "state": 2,               // 2 = SECONDARY
      "stateStr": "SECONDARY",
      "uptime": 3456789,
      "lastHeartbeat": ISODate("2025-12-08T14:42:28Z"),
      "lastHeartbeatRecv": ISODate("2025-12-08T14:42:29Z"),
      "pingMs": 1,
      "syncSourceHost": "primary.example.com:27017"
    },
    {
      "name": "secondary2.example.com:27017",
      "health": 0,              // ⚠️ DOWN
      "state": 8,               // 8 = DOWN
      "stateStr": "DOWN",
      "lastHeartbeat": ISODate("2025-12-08T14:40:15Z"),
      "lastHeartbeatRecv": ISODate("2025-12-08T14:40:10Z"),
      "pingMs": 0
    }
  ]
}
```

**États possibles** :

| State | stateStr | Signification | Action |
|-------|----------|---------------|--------|
| 0 | STARTUP | Initialisation | Normal au démarrage |
| 1 | PRIMARY | Nœud primaire | Monitoring normal |
| 2 | SECONDARY | Réplique secondaire | Monitoring normal |
| 3 | RECOVERING | Resynchronisation | Attendre, surveiller |
| 5 | STARTUP2 | Initial sync en cours | Peut être long |
| 6 | UNKNOWN | État inconnu | Investigation |
| 7 | ARBITER | Arbitre | Monitoring normal |
| 8 | DOWN | Hors ligne | ⚠️ Alerte immédiate |
| 9 | ROLLBACK | Rollback en cours | ⚠️ Perte de données |
| 10 | REMOVED | Retiré du RS | Configuration change |

**Monitoring automatisé** :

```javascript
function monitorReplicaSetHealth() {
  var status = rs.status()
  var alerts = []

  status.members.forEach(member => {
    // Alerte si membre down
    if (member.health === 0) {
      alerts.push({
        severity: "CRITICAL",
        member: member.name,
        issue: "Member is DOWN",
        lastSeen: member.lastHeartbeat
      })
    }

    // Alerte si recovering trop longtemps
    if (member.stateStr === "RECOVERING" && member.uptime > 3600) {
      alerts.push({
        severity: "WARNING",
        member: member.name,
        issue: "Member stuck in RECOVERING for > 1h",
        uptime: member.uptime
      })
    }

    // Alerte si ping latency élevée
    if (member.pingMs > 100) {
      alerts.push({
        severity: "WARNING",
        member: member.name,
        issue: `High network latency: ${member.pingMs}ms`,
        ping: member.pingMs
      })
    }
  })

  return alerts
}
```

---

## 6. Métriques de Sharding

### 6.1 Distribution des chunks

**Définition** : Répartition des chunks de données entre les shards.

```javascript
// Statistiques globales du cluster
sh.status()

// Distribution des chunks par shard
db.getSiblingDB("config").chunks.aggregate([
  { $group: {
    _id: "$shard",
    count: { $sum: 1 }
  }},
  { $sort: { count: -1 }}
])

// Résultat :
// { "_id": "shard01", "count": 450 }
// { "_id": "shard02", "count": 445 }
// { "_id": "shard03", "count": 280 }  ⚠️ Déséquilibre

// Détail par collection
db.getSiblingDB("config").chunks.aggregate([
  { $group: {
    _id: { ns: "$ns", shard: "$shard" },
    count: { $sum: 1 }
  }},
  { $sort: { "_id.ns": 1, count: -1 }}
])
```

**Métriques de distribution** :

```javascript
function analyzeChunkDistribution() {
  var chunks = db.getSiblingDB("config").chunks.aggregate([
    { $group: { _id: "$shard", count: { $sum: 1 },
                collections: { $addToSet: "$ns" } }},
    { $sort: { count: -1 }}
  ]).toArray()

  var totalChunks = chunks.reduce((sum, s) => sum + s.count, 0)
  var avgChunks = totalChunks / chunks.length

  return chunks.map(shard => ({
    shard: shard._id,
    chunks: shard.count,
    deviation: ((shard.count - avgChunks) / avgChunks * 100).toFixed(2) + "%",
    collectionsCount: shard.collections.length
  }))
}

analyzeChunkDistribution()
// Idéal : deviation < 10% entre shards
```

**Seuils d'alerte** :

| Métrique | Warning | Critical | Impact |
|----------|---------|----------|--------|
| Déviation chunks | > 15% | > 30% | Hotspots possibles |
| Jumbo chunks | > 0 | > 5 | Migrations bloquées |
| Migrations échouées | > 5/jour | > 20/jour | Problème de balancing |

### 6.2 Migrations de chunks

**Définition** : Déplacements de chunks entre shards pour équilibrer la charge.

```javascript
// Migrations en cours
sh.isBalancerRunning()
// true ou false

// Statistiques de migration
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.commit",
  time: { $gt: ISODate("2025-12-08T00:00:00Z") }
}).count()

// Détail des migrations récentes
db.getSiblingDB("config").changelog.find({
  what: { $in: ["moveChunk.start", "moveChunk.commit", "moveChunk.error"] }
}).sort({ time: -1 }).limit(10).forEach(entry => {
  print(`${entry.time}: ${entry.what} - ${entry.ns}`)
  if (entry.details) {
    print(`  From: ${entry.details.from} → To: ${entry.details.to}`)
    print(`  Duration: ${entry.details.step5 - entry.details.step1}ms`)
  }
})
```

**Métriques de performance des migrations** :

```javascript
function analyzeMigrationPerformance(hoursBack = 24) {
  var cutoff = new Date(Date.now() - hoursBack * 3600 * 1000)

  var migrations = db.getSiblingDB("config").changelog.aggregate([
    { $match: {
      what: "moveChunk.commit",
      time: { $gt: cutoff }
    }},
    { $project: {
      ns: 1,
      duration: {
        $subtract: ["$details.step5", "$details.step1"]
      },
      from: "$details.from",
      to: "$details.to"
    }},
    { $group: {
      _id: null,
      totalMigrations: { $sum: 1 },
      avgDuration: { $avg: "$duration" },
      maxDuration: { $max: "$duration" },
      minDuration: { $min: "$duration" }
    }}
  ]).toArray()[0]

  return {
    period: `Last ${hoursBack} hours`,
    totalMigrations: migrations.totalMigrations,
    avgDurationMs: Math.round(migrations.avgDuration),
    maxDurationMs: migrations.maxDuration,
    migrationsPerHour: (migrations.totalMigrations / hoursBack).toFixed(2)
  }
}

analyzeMigrationPerformance(24)
```

**Alertes recommandées** :
- Migration duration > 5 minutes → Chunk trop grand ou réseau lent
- Migrations échouées > 5% → Problème de configuration ou ressources
- Aucune migration pendant 24h avec déséquilibre > 20% → Balancer arrêté

### 6.3 Jumbo Chunks

**Définition** : Chunks dépassant la taille maximale configurée (64 MB par défaut).

```javascript
// Identifier les jumbo chunks
db.getSiblingDB("config").chunks.find({ jumbo: true }).forEach(chunk => {
  print(`Jumbo chunk found:`)
  print(`  Collection: ${chunk.ns}`)
  print(`  Shard: ${chunk.shard}`)
  print(`  Range: ${JSON.stringify(chunk.min)} → ${JSON.stringify(chunk.max)}`)
})

// Compter les jumbo chunks par collection
db.getSiblingDB("config").chunks.aggregate([
  { $match: { jumbo: true }},
  { $group: { _id: "$ns", count: { $sum: 1 }}},
  { $sort: { count: -1 }}
])
```

**Impact des jumbo chunks** :
- ❌ Ne peuvent pas être migrés automatiquement
- ❌ Causent des déséquilibres persistants
- ❌ Peuvent créer des hotspots sur un shard

**Résolution** :

```javascript
// 1. Activer le splitting manuel
sh.enableAutoSplit()

// 2. Forcer le split sur un chunk spécifique
sh.splitAt("mydb.mycollection", { shardKey: "someValue" })

// 3. Si le split échoue, vérifier la distribution des valeurs
db.mycollection.aggregate([
  { $group: { _id: "$shardKey", count: { $sum: 1 }}},
  { $sort: { count: -1 }},
  { $limit: 10 }
])
// Si une valeur domine → Problème de shard key
```

---

## 7. Métriques réseau

### 7.1 Bande passante réseau

**Définition** : Volume de données échangées sur le réseau.

```javascript
db.serverStatus().network
{
  "bytesIn": 123456789012,      // Bytes reçus depuis le démarrage
  "bytesOut": 234567890123,     // Bytes envoyés depuis le démarrage
  "physicalBytesIn": 98765432109,
  "physicalBytesOut": 187654321098,
  "numRequests": 45678901,      // Nombre de requêtes reçues
  "numSlowDNSOperations": 0,
  "numSlowSSLOperations": 0
}
```

**Calcul du débit** :

```bash
#!/bin/bash
# Collecter bytesIn et bytesOut à T0 et T1

function calculateNetworkThroughput() {
  BYTES_IN_T0=$1
  BYTES_OUT_T0=$2
  BYTES_IN_T1=$3
  BYTES_OUT_T1=$4
  INTERVAL=$5  # en secondes

  IN_MBPS=$(echo "scale=2; ($BYTES_IN_T1 - $BYTES_IN_T0) / $INTERVAL / 1024 / 1024" | bc)
  OUT_MBPS=$(echo "scale=2; ($BYTES_OUT_T1 - $BYTES_OUT_T0) / $INTERVAL / 1024 / 1024" | bc)

  echo "Inbound: ${IN_MBPS} MB/s"
  echo "Outbound: ${OUT_MBPS} MB/s"
}
```

**Seuils selon le type de réseau** :

| Réseau | Capacité | Utilisation Warning | Utilisation Critical |
|--------|----------|---------------------|----------------------|
| 1 Gbps | 125 MB/s | > 80 MB/s (64%) | > 100 MB/s (80%) |
| 10 Gbps | 1250 MB/s | > 800 MB/s (64%) | > 1000 MB/s (80%) |
| 25 Gbps | 3125 MB/s | > 2000 MB/s (64%) | > 2500 MB/s (80%) |

**Analyse d'exemple - Saturation réseau** :

```javascript
// Scénario : Débit > 80% de la capacité

// 1. Identifier les opérations consommatrices
db.currentOp({
  $or: [
    { "command.getMore": { $exists: true }},  // Curseurs
    { op: "query", docsExamined: { $gt: 10000 }}  // Large queries
  ]
}).inprog.forEach(op => {
  print(`Op: ${op.op}, NS: ${op.ns}`)
  print(`Docs examined: ${op.docsExamined}`)
  print(`Duration: ${op.secs_running}s`)
})

// 2. Vérifier les projections (récupérer moins de données)
// 3. Vérifier compression réseau (snappy activé?)
db.serverStatus().network.compression
```

**Optimisations** :
- Activer la compression réseau (snappy, zstd)
- Optimiser les projections (récupérer moins de champs)
- Utiliser l'agrégation côté serveur
- Implémenter pagination côté client

---

## 8. Métriques de locks et contentions

### 8.1 Global Lock

**Définition** : Verrous globaux sur le serveur MongoDB.

```javascript
db.serverStatus().globalLock
{
  "totalTime": 123456789012,  // Temps total depuis démarrage (µs)
  "currentQueue": {
    "total": 5,                // Opérations en attente de lock
    "readers": 3,
    "writers": 2
  },
  "activeClients": {
    "total": 45,               // Clients actifs
    "readers": 32,
    "writers": 13
  }
}
```

**Métriques de contention** :

```javascript
db.serverStatus().locks
{
  "Global": {
    "acquireCount": { "r": 234567890, "w": 12345678 },
    "acquireWaitCount": { "r": 234, "w": 567 },  // Fois où il a fallu attendre
    "timeAcquiringMicros": { "r": 45678, "w": 123456 }  // Temps d'attente total
  },
  "Database": {
    "acquireCount": { "r": 123456789, "w": 9876543 },
    "acquireWaitCount": { "r": 123, "w": 456 },
    "timeAcquiringMicros": { "r": 23456, "w": 89012 }
  },
  "Collection": {
    "acquireCount": { "r": 987654321, "w": 8765432 },
    "acquireWaitCount": { "r": 89, "w": 234 },
    "timeAcquiringMicros": { "r": 12345, "w": 56789 }
  }
}
```

**Calcul du taux de contention** :

```javascript
function calculateLockContention() {
  var locks = db.serverStatus().locks
  var results = {}

  for (var lockType in locks) {
    var data = locks[lockType]
    if (data.acquireCount && data.acquireWaitCount) {
      var readContention = data.acquireWaitCount.r / data.acquireCount.r * 100
      var writeContention = data.acquireWaitCount.w / data.acquireCount.w * 100

      results[lockType] = {
        readContention: readContention.toFixed(4) + "%",
        writeContention: writeContention.toFixed(4) + "%",
        avgWaitTimeRead: data.acquireWaitCount.r > 0 ?
          (data.timeAcquiringMicros.r / data.acquireWaitCount.r / 1000).toFixed(2) + "ms" : "0ms",
        avgWaitTimeWrite: data.acquireWaitCount.w > 0 ?
          (data.timeAcquiringMicros.w / data.acquireWaitCount.w / 1000).toFixed(2) + "ms" : "0ms"
      }
    }
  }

  return results
}

calculateLockContention()
```

**Seuils d'alerte** :

| Métrique | Warning | Critical | Action |
|----------|---------|----------|--------|
| Queue total | > 10 | > 50 | Identifier opérations lentes |
| Write contention % | > 1% | > 5% | Optimiser écritures |
| Avg wait time | > 50ms | > 200ms | Contention sévère |

**Analyse d'exemple - Write contention élevée** :

```javascript
// Scénario : writeContention = 8%, avgWaitTime = 350ms

// 1. Identifier les opérations qui prennent des write locks
db.currentOp({
  $or: [
    { op: "insert" },
    { op: "update" },
    { op: "remove" }
  ],
  waitingForLock: false,  // Celles qui TIENNENT le lock
  secs_running: { $gt: 5 }
}).inprog.forEach(op => {
  print(`${op.op} on ${op.ns} running for ${op.secs_running}s`)
  print(`Locks: ${JSON.stringify(op.locks)}`)
})

// 2. Vérifier si bulk operations ou transactions longues
db.currentOp({
  "transaction": { $exists: true },
  secs_running: { $gt: 10 }
})
```

**Actions correctives** :
- Réduire la durée des transactions
- Utiliser des batch plus petits pour les bulk operations
- Optimiser les index pour accélérer les updates
- Éviter les opérations DDL (createIndex) aux heures de pointe

---

## 9. Synthèse : Dashboard de métriques critiques

### Template de dashboard SRE

```
┌───────────────────────────────────────────────────────────────┐
│ MONGODB PRODUCTION HEALTH - Cluster: prod-rs0                 │
│ Updated: 2025-12-08 15:42:30 UTC           Status: 🟢 HEALTHY │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│ SYSTÈME                          CONNEXIONS                   │
│ ├─ CPU Usage:        45% 🟢      ├─ Current:     847 🟢       │
│ ├─ RAM Usage:        67% 🟢      ├─ Available:   50,153 🟢    │
│ ├─ Disk I/O Wait:    12% 🟢      ├─ Active:      23 🟢        │
│ └─ Disk Space:       72% 🟢      └─ Queued:      2 🟢         │
│                                                               │
│ OPÉRATIONS                       RÉPLICATION                  │
│ ├─ Read ops/s:      1,245 🟢     ├─ Lag Sec1:    0.2s 🟢      │
│ ├─ Write ops/s:      456 🟢      ├─ Lag Sec2:    15s ⚠️       │
│ ├─ P95 Latency:     47ms 🟢      ├─ Oplog Win:   48h 🟢       │
│ └─ Slow Queries:      12 ⚠️      └─ Members:     3/3 UP 🟢    │
│                                                               │
│ CACHE & MÉMOIRE                  SHARDING                     │
│ ├─ WT Cache:        85% ⚠️       ├─ Chunks:      Balanced 🟢  │
│ ├─ Dirty Pages:     15% 🟢       ├─ Migrations:  3 active 🟢  │
│ ├─ Page Faults/s:    8 🟢        ├─ Jumbo Chunks: 0 🟢        │
│ └─ Evictions/s:     45 🟢        └─ Balancer:    ON 🟢        │
│                                                               │
│ ALERTES ACTIVES (2)                                           │
│ ⚠️  Secondary2 replication lag: 15s (threshold: 10s)          │
│ ⚠️  WiredTiger cache usage: 85% (threshold: 85%)              │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Checklist de surveillance quotidienne

**Matin (9h)** :
- [ ] Vérifier disponibilité (rs.status(), tous membres UP)
- [ ] Vérifier replication lag (< 5s)
- [ ] Vérifier espace disque (< 80%)
- [ ] Vérifier alertes overnight
- [ ] Examiner slow queries log (top 10)

**Midi (12h)** :
- [ ] Vérifier métriques de charge (pics de trafic)
- [ ] Vérifier connexions (usage < 70%)
- [ ] Vérifier cache hit ratio
- [ ] Vérifier migrations en cours (sharding)

**Soir (18h)** :
- [ ] Vérifier oplog window (> 24h)
- [ ] Vérifier backup de la journée
- [ ] Analyser tendances de croissance
- [ ] Planifier maintenance si nécessaire

### Métriques par priorité

**P0 - Critique (Paging 24/7)** :
1. Primary down
2. Majorité de membres down (quorum perdu)
3. Disk full (> 95%)
4. Replication stopped (lag croissant indéfiniment)

**P1 - Urgent (Intervention dans l'heure)** :
1. Secondary down
2. Replication lag > 60s
3. Connection pool > 90%
4. Slow queries impact > 20% trafic

**P2 - Important (Intervention dans la journée)** :
1. Cache usage > 90%
2. Disk usage > 85%
3. CPU usage > 85% sustained
4. Jumbo chunks détectés

**P3 - Surveillance (Analyse hebdomadaire)** :
1. Index usage efficiency
2. Query patterns evolution
3. Growth rate projections
4. Fragmentation levels

---

## Prochaines étapes

Maintenant que vous maîtrisez les métriques clés, les prochaines sections approfondissent :

- **13.2** Commandes d'administration (serverStatus, dbStats, currentOp détaillés)
- **13.3** Profiler de requêtes (identification et optimisation)
- **13.4** Logs MongoDB (structure, parsing, alerting)
- **13.5** MongoDB Database Tools (mongodump, mongostat, mongotop)
- **13.6** mongostat et mongotop en détail
- **13.7** Intégration Prometheus + Grafana (stack complète)
- **13.8** MongoDB Ops Manager (solution entreprise)
- **13.9** Alerting et notifications (PagerDuty, OpsGenie, Slack)
- **13.10** Diagnostics FTDC (Full Time Diagnostic Data Capture)
- **13.11** Gestion mémoire et cache WiredTiger (tuning avancé)

---

**Points clés à retenir** :

✅ Établir une baseline avant de définir des seuils d'alerte

✅ Corréler plusieurs métriques pour diagnostic précis (CPU + I/O + Cache)

✅ Prioriser les métriques selon impact sur disponibilité et performance

✅ Automatiser la collecte avec Prometheus, Datadog, ou Atlas

✅ Documenter les seuils et runbooks pour chaque alerte

✅ Réviser régulièrement les seuils selon l'évolution du workload

---


⏭️ [Commandes d'administration](/13-monitoring-administration/02-commandes-administration.md)
