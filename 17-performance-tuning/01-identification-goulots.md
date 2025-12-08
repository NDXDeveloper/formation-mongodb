🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.1 Identification des Goulots d'Étranglement

## Introduction

L'identification précise des goulots d'étranglement est la première et la plus critique étape de toute démarche d'optimisation. Une analyse erronée conduit à des optimisations inefficaces, voire contre-productives. Dans un système MongoDB en production, les goulots peuvent se manifester à différents niveaux : applicatif, base de données, système, réseau, ou architecture distribuée.

Cette section présente une méthodologie systématique pour identifier, isoler et caractériser les goulots d'étranglement en environnement de production.

## Méthodologie USE (Utilization, Saturation, Errors)

La méthodologie USE, développée par Brendan Gregg, fournit un framework systématique pour l'analyse de performance. Pour chaque ressource du système, on examine trois dimensions :

### Utilization (Utilisation)

Mesure du temps pendant lequel la ressource est occupée (busy time), exprimée en pourcentage.

**Ressources à analyser :**
- **CPU** : Temps CPU utilisé vs idle
- **Mémoire** : RAM utilisée vs disponible
- **Disque** : Temps pendant lequel le disque traite des requêtes
- **Réseau** : Bande passante utilisée vs disponible
- **Connexions** : Connexions actives vs pool size maximum

**Interprétation des seuils :**
```
< 70%  : Utilisation normale, capacité suffisante
70-85% : Zone d'attention, surveiller les tendances
85-95% : Zone critique, planifier l'augmentation de capacité
> 95%  : Saturation imminente, action immédiate requise
```

### Saturation (Saturation)

Degré auquel une ressource a du travail en attente qu'elle ne peut traiter immédiatement.

**Indicateurs de saturation :**
- **CPU** : Load average, run queue length
- **Mémoire** : Page faults, swap activity
- **Disque** : Queue depth, await time
- **Réseau** : Dropped packets, retransmissions
- **MongoDB** : Queued operations (read/write queues)

**Saturation critique MongoDB :**
```javascript
// Vérification des queues MongoDB
db.serverStatus().globalLock.currentQueue
{
  total: 0,     // Queue totale
  readers: 0,   // Opérations de lecture en attente
  writers: 0    // Opérations d'écriture en attente
}

// Valeurs préoccupantes :
// readers/writers > 10 : Saturation légère
// readers/writers > 50 : Saturation sévère
// readers/writers > 100 : Saturation critique
```

### Errors (Erreurs)

Compteurs d'erreurs qui peuvent indiquer des problèmes de ressources ou de configuration.

**Erreurs critiques à surveiller :**
- **Connexions refusées** : Pool exhaustion, limite de connexions
- **Timeouts** : Network, operation, election
- **Assertion failures** : Bugs, corruption, invariants violés
- **Replication errors** : Lag excessif, oplog overflow
- **OOM (Out of Memory)** : Killer process, allocation failures

## Identification par Symptôme

### Symptôme : Latence Élevée

La latence élevée peut avoir de multiples causes. Une approche méthodique est nécessaire pour identifier la vraie cause racine.

#### Analyse de la Latence

**1. Localisation de la latence**

Décomposer la latence totale en ses composantes :

```
Latence totale = Latence réseau + Latence queue + Latence traitement + Latence I/O
```

**Outils de mesure :**
- **Application side** : APM (New Relic, Datadog), instrumentation custom
- **MongoDB side** : Profiler, currentOp, system.profile collection
- **Infrastructure** : Network monitoring, traceroute, ping latency

**2. Analyse des percentiles**

Ne jamais se fier uniquement à la moyenne, qui masque les outliers :

```javascript
// Analyse des percentiles dans le profiler
db.system.profile.aggregate([
  { $match: { ts: { $gte: ISODate("2025-01-01") } } },
  { $group: {
      _id: "$command.find",
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" },
      maxMs: { $max: "$millis" },
      percentiles: { $push: "$millis" }
  }},
  { $project: {
      count: 1,
      avgMs: 1,
      maxMs: 1,
      p50: { $arrayElemAt: ["$percentiles", { $multiply: [0.50, "$count"] }] },
      p95: { $arrayElemAt: ["$percentiles", { $multiply: [0.95, "$count"] }] },
      p99: { $arrayElemAt: ["$percentiles", { $multiply: [0.99, "$count"] }] }
  }}
])
```

**Interprétation :**
- P50 élevé : Problème systémique affectant la majorité des requêtes
- P95-P99 élevés mais P50 normal : Outliers, possiblement contention ou cold cache
- Max >> P99 : Requêtes pathologiques à investiguer individuellement

#### Corrélation avec les Ressources Système

**Matrice de corrélation latence-ressources :**

| Symptôme | CPU Élevé | RAM Faible | I/O Élevé | Network Lent |
|----------|-----------|------------|-----------|--------------|
| Latence constante haute | ✓ | ✓ | ✓ | ✓ |
| Pics de latence sporadiques | - | ✓ (page faults) | ✓ (contention) | ✓ (congestion) |
| Latence croissante dans le temps | - | ✓ (leak) | ✓ (fragmentation) | - |
| Latence variable selon l'heure | ✓ (batch jobs) | - | ✓ (backups) | ✓ (peak hours) |

### Symptôme : Faible Throughput

Le throughput faible indique que le système ne traite pas suffisamment d'opérations par unité de temps.

#### Analyse du Throughput

**1. Mesure du throughput actuel**

```javascript
// Throughput via serverStatus
const stats1 = db.serverStatus();
// Attendre 60 secondes
sleep(60000);
const stats2 = db.serverStatus();

const opsPerSecond = {
  insert: (stats2.opcounters.insert - stats1.opcounters.insert) / 60,
  query: (stats2.opcounters.query - stats1.opcounters.query) / 60,
  update: (stats2.opcounters.update - stats1.opcounters.update) / 60,
  delete: (stats2.opcounters.delete - stats1.opcounters.delete) / 60,
  command: (stats2.opcounters.command - stats1.opcounters.command) / 60
};
```

**2. Identification des limiteurs de throughput**

**Lock contention** :
```javascript
db.serverStatus().locks
// Analyse de :
// - acquireCount : Nombre de fois où le lock a été acquis
// - acquireWaitCount : Nombre de fois où il a fallu attendre
// - timeAcquiringMicros : Temps total d'attente

// Ratio critique :
const lockContentionRatio = timeAcquiringMicros / (acquireCount * avgOperationTime);
// > 10% : Contention significative
```

**Connection exhaustion** :
```javascript
db.serverStatus().connections
{
  current: 850,      // Connexions actuelles
  available: 150,    // Connexions disponibles
  totalCreated: 12500
}

// Saturation si available < 10% de la limite configurée
```

**Write concern timeout** :
```javascript
// Recherche des timeouts de write concern dans les logs
db.adminCommand({
  getLog: "global"
}).log.filter(entry => entry.includes("writeConcern"))
```

### Symptôme : Utilisation Mémoire Excessive

L'utilisation mémoire excessive peut conduire à du swapping et une dégradation catastrophique des performances.

#### Analyse de la Mémoire

**1. Décomposition de l'utilisation mémoire**

```javascript
const memStats = db.serverStatus().mem;
{
  bits: 64,           // Architecture
  resident: 4096,     // RAM physique utilisée (MB)
  virtual: 8192,      // Mémoire virtuelle totale (MB)
  mapped: 0,          // Fichiers mappés (MMAPv1 uniquement)
  mappedWithJournal: 0
}

const wtCache = db.serverStatus().wiredTiger.cache;
{
  "maximum bytes configured": 5368709120,  // 5 GB cache WiredTiger
  "bytes currently in the cache": 4829847552,  // Utilisation actuelle
  "tracked dirty bytes in the cache": 524288000,  // Données modifiées non écrites
  "pages evicted by application threads": 1234,
  "pages read into cache": 567890,
  "pages written from cache": 123456
}
```

**2. Identification des fuites mémoire**

**Méthode de détection :**
```bash
# Surveillance continue de la mémoire résidente
while true; do
  echo "$(date): $(mongo --quiet --eval 'db.serverStatus().mem.resident')" >> mem_tracking.log
  sleep 300  # Toutes les 5 minutes
done

# Analyse de la tendance
awk '{print $2, $3}' mem_tracking.log | \
  gnuplot -e "set terminal dumb; plot '-' with lines"
```

**Indicateurs de fuite :**
- Croissance linéaire continue sans plateau
- Pas de corrélation avec le volume de données ou le nombre de connexions
- Pas de récupération mémoire après périodes de faible activité

**3. Working Set Analysis**

```javascript
// Estimation du working set
db.serverStatus().wiredTiger.cache["bytes currently in the cache"]
+ db.serverStatus().wiredTiger.cache["tracked dirty bytes in the cache"]

// Comparaison avec la taille des données accédées fréquemment
db.stats().dataSize + db.stats().indexSize
```

**Diagnostic :**
- Working set > RAM disponible : Thrashing inevitable, nécessite scaling vertical
- Cache eviction rate élevé : Cache sous-dimensionné
- Dirty ratio > 20% : Write pressure élevée, checkpoint contention possible

### Symptôme : I/O Disque Élevé

Le disque est souvent le composant le plus lent et peut devenir un goulot majeur.

#### Analyse des I/O

**1. Métriques système I/O**

```bash
# iostat - Analyse détaillée
iostat -x 5 3
# Métriques critiques :
# - %util : Utilisation du disque (> 80% = problématique)
# - await : Latence moyenne des requêtes I/O (> 20ms = lent)
# - svctm : Temps de service (deprecated mais informatif)
# - r/s, w/s : Reads et writes par seconde
# - rkB/s, wkB/s : Throughput en KB/s

# Patterns problématiques :
# - High await + Low throughput : Contention ou disque lent
# - High util + High r/s : Read-heavy workload mal optimisé
# - High util + High w/s : Write-heavy workload ou checkpoint contention
```

**2. Analyse des patterns d'accès MongoDB**

```javascript
// Statistiques de stockage WiredTiger
const storageStats = db.serverStatus().wiredTiger.blockManager;
{
  "blocks read": 1234567,
  "blocks written": 987654,
  "blocks read per second": 45.2,
  "blocks written per second": 67.8
}

// Ratio read/write
const rwRatio = storageStats["blocks read"] / storageStats["blocks written"];
// Indique le type de workload dominant
```

**3. Identification des requêtes I/O intensives**

```javascript
// Recherche des requêtes avec beaucoup de documents examinés
db.system.profile.find({
  docsExamined: { $gt: 10000 },
  millis: { $gt: 100 }
}).sort({ docsExamined: -1 }).limit(10)

// Analyse des collection scans
db.system.profile.find({
  "planSummary": "COLLSCAN"
}).count()
```

**Patterns typiques :**
- **Cold cache après redémarrage** : Pics I/O initiaux puis stabilisation
- **Collection scans fréquents** : Absence d'index appropriés
- **Hot documents** : Petite portion de données accédée très fréquemment
- **Working set too large** : Cache misses constants

### Symptôme : CPU Élevé

L'utilisation CPU élevée peut indiquer des requêtes inefficaces ou un volume de traitement excessif.

#### Analyse CPU

**1. Décomposition de l'utilisation CPU**

```bash
# top/htop - Vue processus
# Identifier les threads MongoDB consommateurs :
# - mongod : Thread principal
# - conn* : Threads de connexion (un par connexion active)
# - ftdc : Full Time Diagnostic Data Capture
# - snapshot : Snapshot threads

# Analyse par thread
top -H -p $(pidof mongod)
```

**2. Analyse du CPU dans MongoDB**

```javascript
// Pas de métrique CPU directe dans serverStatus
// Mais analyse indirecte via :

// Operations lentes indiquant CPU bound
db.currentOp({
  "active": true,
  "secs_running": { "$gt": 1 },
  "microsecs_running": { "$gt": 1000000 }
})

// Agrégations complexes
db.system.profile.find({
  "command.aggregate": { $exists: true },
  "millis": { $gt: 1000 }
})
```

**3. Identification des causes**

**In-Memory Sorting** :
```javascript
// Sorts sans index, forcés en mémoire
db.system.profile.find({
  "planSummary": /SORT/,
  "executionStats.executionStages.stage": "SORT",
  "executionStats.executionStages.memLimit": 33554432  // 32 MB limit
})
```

**Regex non-optimisées** :
```javascript
// Recherche des regex sans ancrage
db.system.profile.find({
  "command.filter": {
    $exists: true
  }
}).forEach(doc => {
  const filter = doc.command.filter;
  for (let key in filter) {
    if (filter[key].$regex && !filter[key].$regex.startsWith("^")) {
      printjson(doc);
    }
  }
})
```

**Agrégations complexes** :
```javascript
// Pipelines avec nombreux stages
db.system.profile.find({
  "command.pipeline": { $exists: true }
}).sort({ "command.pipeline.length": -1 })
```

## Analyse des Goulots Spécifiques MongoDB

### Goulot : Index Inefficace ou Manquant

#### Détection

**1. Index Usage Statistics**

```javascript
// Statistiques d'utilisation des index
db.collection.aggregate([
  { $indexStats: {} }
])

// Indicateurs :
// - accesses.ops : Nombre d'accès
// - accesses.since : Date du dernier accès
// Index non utilisés : candidates pour suppression
```

**2. Analyse explain() systématique**

```javascript
// Audit automatisé des requêtes profilées
db.system.profile.find({
  millis: { $gt: 100 }
}).forEach(function(doc) {
  if (doc.command.find) {
    const explainOutput = db[doc.ns.split('.')[1]]
      .find(doc.command.filter)
      .explain("executionStats");

    // Calcul du ratio examined/returned
    const examined = explainOutput.executionStats.totalDocsExamined;
    const returned = explainOutput.executionStats.nReturned;
    const ratio = examined / (returned || 1);

    if (ratio > 10) {
      print(`Inefficient query - Ratio: ${ratio}`);
      printjson(doc.command);
    }
  }
});
```

**Seuils critiques :**
```
Ratio examined/returned :
< 2    : Excellent (index couvrant ou très sélectif)
2-10   : Acceptable
10-100 : Préoccupant (index sous-optimal)
> 100  : Critique (collection scan ou index inefficace)
```

### Goulot : Replication Lag

Le lag de réplication affecte la cohérence et peut saturer la bande passante réseau.

#### Détection et Analyse

**1. Mesure du lag**

```javascript
// Sur chaque membre du replica set
rs.status().members.forEach(function(member) {
  if (member.state === 2) {  // SECONDARY
    const lag = rs.status().members.find(m => m.state === 1).optimeDate - member.optimeDate;
    print(`Member ${member.name}: Lag = ${lag / 1000} seconds`);
  }
})
```

**2. Analyse de l'oplog**

```javascript
// Taille et fenêtre de l'oplog
const replSetGetStatus = rs.status();
const oplogs = db.getSiblingDB("local").oplog.rs;

const firstEntry = oplogs.find().sort({$natural: 1}).limit(1).next();
const lastEntry = oplogs.find().sort({$natural: -1}).limit(1).next();

const windowHours = (lastEntry.ts.getTime() - firstEntry.ts.getTime()) / 3600;

print(`Oplog window: ${windowHours} hours`);
// Recommandation : > 24h pour tolérer les maintenances
```

**3. Identification des causes**

**Write volume excessif** :
```javascript
// Taux d'écriture sur le primary
db.serverStatus().opcounters.insert +
db.serverStatus().opcounters.update +
db.serverStatus().opcounters.delete

// Comparaison avec la capacité de réplication des secondaries
```

**Secondary performance** :
```javascript
// Sur un secondary, vérifier la vitesse d'application de l'oplog
db.serverStatus().repl.apply.ops
```

**Network bandwidth** :
```bash
# Monitoring de la bande passante
iftop -i eth0
# ou
nload eth0

# Vérifier si saturation réseau corrèle avec augmentation du lag
```

### Goulot : Balancer Activity (Sharding)

Les migrations de chunks peuvent impacter significativement les performances.

#### Détection

**1. État du balancer**

```javascript
sh.status()
// Vérifier :
// - balancer state : actif ou non
// - currently-enabled : balancing windows
// - migrations : en cours ou en queue

// Détails des migrations
db.getSiblingDB("config").locks.find({state: 2})  // Locks de migration
```

**2. Impact des migrations**

```javascript
// Logs de migration
db.adminCommand({
  getLog: "global"
}).log.filter(entry => entry.includes("moveChunk"))

// Métriques de migration
sh.status().shards.forEach(function(shard) {
  const shardStats = db.getSiblingDB("admin").runCommand({
    serverStatus: 1,
    sharding: 1
  });

  if (shardStats.sharding) {
    print(`Shard ${shard._id}:`);
    print(`  Migration in progress: ${shardStats.sharding.migrationInProgress}`);
  }
})
```

**3. Identification des problèmes**

**Jumbo chunks** :
```javascript
// Chunks non-splittables
db.getSiblingDB("config").chunks.find({
  jumbo: true
})

// Distribution des tailles de chunks
db.getSiblingDB("config").chunks.aggregate([
  { $group: {
      _id: "$ns",
      avgSize: { $avg: "$estimatedDataSize" },
      maxSize: { $max: "$estimatedDataSize" },
      jumboCount: {
        $sum: { $cond: ["$jumbo", 1, 0] }
      }
  }}
])
```

**Hotspots** :
```javascript
// Identification des shards surchargés
sh.status().shards.forEach(function(shard) {
  const stats = db.getSiblingDB(shard.host).serverStatus();
  print(`${shard._id}: ${stats.opcounters.insert + stats.opcounters.update} writes/sec`);
})
```

### Goulot : Connection Pool Exhaustion

L'épuisement du pool de connexions crée du queuing côté application.

#### Détection

**1. Monitoring des connexions**

```javascript
// Connexions actuelles vs limites
db.serverStatus().connections
{
  current: 850,
  available: 150,
  totalCreated: 15234,
  rejected: 12,  // Connexions rejetées
  active: 342    // Connexions avec opération en cours
}

// Alerter si :
// - available < 10% de la limite
// - rejected > 0
// - active/current ratio < 0.3 (beaucoup de connexions idle)
```

**2. Analyse des connexions actives**

```javascript
// Détail des connexions
db.currentOp({ "$all": true }).inprog.forEach(function(op) {
  if (op.client) {
    print(`Client: ${op.client}, Op: ${op.op}, Secs: ${op.secs_running}`);
  }
})

// Regroupement par client
db.currentOp({ "$all": true }).inprog.reduce(function(acc, op) {
  if (op.client) {
    acc[op.client] = (acc[op.client] || 0) + 1;
  }
  return acc;
}, {})
```

**3. Configuration et tuning**

**Limites système** :
```bash
# Vérification des limites OS
ulimit -n  # File descriptors
cat /proc/sys/net/ipv4/ip_local_port_range  # Ports disponibles
cat /proc/sys/net/ipv4/tcp_fin_timeout  # TIME_WAIT timeout
```

**Configuration MongoDB** :
```javascript
// Limite de connexions configurée
db.serverStatus().connections.maxPoolSize ||
  "unlimited (défaut: 65536)"

// Configurable via :
// mongod.conf : net.maxIncomingConnections
// ou --maxConns parameter
```

## Outils de Diagnostic Avancés

### mongotop et mongostat

**mongotop** : Monitoring en temps réel du temps passé par collection

```bash
mongotop 5  # Refresh toutes les 5 secondes

# Interpréter :
# - Total time : Temps total par collection
# - Read time : Temps en lecture
# - Write time : Temps en écriture

# Identifier les collections "hot" consommant le plus de ressources
```

**mongostat** : Vue globale des opérations

```bash
mongostat --discover 5  # Tous les membres du replica set

# Métriques clés :
# - insert/query/update/delete : Ops par seconde par type
# - flushes : Nombre de flush du journal
# - mapped/vsize/res : Utilisation mémoire
# - qrw : Queue read/write (indicateur de contention)
# - arw : Active reads/writes
# - net_in/net_out : Trafic réseau

# Pattern critique :
# qrw élevé + arw faible = Bottleneck applicatif ou connexions
# qrw élevé + arw élevé = Saturation ressources MongoDB
```

### FTDC (Full Time Diagnostic Data Capture)

FTDC collecte automatiquement des métriques détaillées toutes les secondes.

**Localisation des fichiers** :
```bash
# Emplacement par défaut
ls -lh /var/log/mongodb/diagnostic.data/

# Fichiers metrics.* : Données brutes BSON compressées
```

**Extraction et analyse** :
```python
# Script Python pour parser FTDC
import bson
import gzip

def parse_ftdc(filename):
    with gzip.open(filename, 'rb') as f:
        while True:
            try:
                doc = bson.decode(f.read(4096))
                # Traiter doc...
                yield doc
            except:
                break

# Analyser les patterns temporels
for doc in parse_ftdc('metrics.2025-01-15T10-00-00Z-00000'):
    if 'serverStatus' in doc:
        print(doc['serverStatus']['opcounters'])
```

**Métriques clés dans FTDC** :
- serverStatus complet toutes les secondes
- replSetGetStatus pour replica sets
- Métriques OS (CPU, mémoire, disque)
- Métriques réseau

### Profiler Granulaire

Configuration avancée du profiler pour analyse ciblée.

**Profiler avec filtres** :
```javascript
// Profiler niveau 1 avec seuil et filtres
db.setProfilingLevel(1, {
  slowms: 100,
  filter: {
    op: { $in: ["query", "update", "remove"] },
    ns: /^mydb\.important/,  // Seulement certaines collections
    millis: { $gt: 50 }
  }
})

// Profiler avec sampling
db.setProfilingLevel(1, {
  slowms: 50,
  sampleRate: 0.1  // Profiler seulement 10% des opérations
})
```

**Analyse avancée du profiler** :
```javascript
// Top requêtes par temps total consommé
db.system.profile.aggregate([
  { $match: { ts: { $gte: new ISODate("2025-01-15T00:00:00Z") } } },
  { $group: {
      _id: {
        op: "$op",
        ns: "$ns",
        pattern: "$command"
      },
      count: { $sum: 1 },
      totalMs: { $sum: "$millis" },
      avgMs: { $avg: "$millis" },
      maxMs: { $max: "$millis" }
  }},
  { $sort: { totalMs: -1 } },
  { $limit: 20 }
])
```

### Performance Advisor (Atlas)

Pour les déploiements Atlas, le Performance Advisor fournit des recommandations automatiques.

**Types de recommandations** :
- **Index suggestions** : Index recommandés basés sur les query patterns
- **Schema suggestions** : Anti-patterns détectés dans la modélisation
- **Query suggestions** : Requêtes sous-optimales identifiées

**Accès programmatique** :
```javascript
// Via Atlas API
curl --user "$ATLAS_PUBLIC_KEY:$ATLAS_PRIVATE_KEY" \
  --digest \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/$GROUP_ID/processes/$HOST:$PORT/performanceAdvisor/suggestedIndexes"
```

## Méthodologie de Diagnostic Holistique

### Approche en Couches

L'analyse doit progresser méthodiquement à travers les couches du système :

**Couche 1 : Application**
- Patterns d'utilisation : Normal vs anomal
- Connection pooling configuration
- Query patterns et fréquence
- Retry logic et timeouts

**Couche 2 : Driver MongoDB**
- Configuration du driver (read preference, write concern)
- Connection pool metrics
- Command monitoring
- Driver version et compatibilité

**Couche 3 : MongoDB Server**
- Query performance (explain, profiler)
- Index effectiveness
- Lock contention
- Oplog et replication

**Couche 4 : Système d'Exploitation**
- CPU, mémoire, disque, réseau
- Kernel parameters (TCP tuning, file descriptors)
- Transparent Huge Pages (THP) disabled
- NUMA architecture considerations

**Couche 5 : Infrastructure**
- Stockage (type, IOPS, latence)
- Réseau (bande passante, latence inter-datacenter)
- Virtualisation overhead
- Cloud provider specifics

### Workflow de Diagnostic

```
1. OBSERVER
   └─> Collecter métriques de toutes les couches
   └─> Identifier les anomalies et patterns

2. ORIENTER
   └─> Hypothèses sur la cause racine
   └─> Prioriser par probabilité et impact

3. DÉCIDER
   └─> Sélectionner l'hypothèse à tester
   └─> Planifier le test de validation

4. AGIR
   └─> Implémenter le changement
   └─> Mesurer l'impact
   └─> Documenter les résultats

5. RÉPÉTER
   └─> Si non résolu, retour à étape 1 avec nouvelles données
```

### Checklist de Diagnostic Rapide

Pour un diagnostic initial en situation d'urgence :

```
☐ Métriques système basiques (CPU, RAM, I/O)
  └─> top, free, iostat, iftop

☐ État MongoDB global
  └─> db.serverStatus()
  └─> db.currentOp({ "$all": true })

☐ Réplication (si applicable)
  └─> rs.status()
  └─> rs.printSlaveReplicationInfo()

☐ Sharding (si applicable)
  └─> sh.status()
  └─> Balancer state

☐ Connexions
  └─> db.serverStatus().connections
  └─> currentOp pour voir connexions actives

☐ Requêtes lentes en cours
  └─> db.currentOp({ "active": true, "secs_running": { $gt: 1 } })

☐ Logs récents
  └─> db.adminCommand({ getLog: "global" })
  └─> Recherche d'erreurs, warnings, timeouts

☐ Profiler (si activé)
  └─> db.system.profile.find().sort({ts:-1}).limit(10)
```

## Patterns de Goulots Communs et Signatures

### Pattern : "Morning Rush"

**Signature** :
- Pics de latence systématiques à heures fixes (début journée)
- CPU et I/O élevés pendant 30-60 minutes
- Normalisation progressive ensuite

**Cause probable** :
- Cold cache après nuit de faible activité
- Batch processes schedulés
- Backup windows se terminant

**Diagnostic** :
```javascript
// Vérifier cache warming rate
db.serverStatus().wiredTiger.cache["pages read into cache"]

// Identifier batch jobs
db.currentOp({
  "secs_running": { $gt: 300 },
  "op": { $in: ["update", "remove"] }
})
```

### Pattern : "Death by Thousand Cuts"

**Signature** :
- Dégradation progressive sur plusieurs jours/semaines
- Aucun événement déclencheur identifiable
- Ressources semblent OK mais performance décline

**Causes probables** :
- Croissance des données au-delà du working set
- Index fragmention
- Oplog window shrinking
- Memory leak lent

**Diagnostic** :
```javascript
// Tendance croissance données
db.collection.stats().size
db.collection.stats().count

// Fragmentation index
db.collection.stats().indexSizes
// Comparer avec taille théorique basée sur le nombre de documents

// Memory leak check
// Monitoring continu de resident memory sur plusieurs jours
```

### Pattern : "Scatter-Gather Storm"

**Signature** :
- Latence élevée sur cluster shardé
- Requêtes simples prenant temps excessif
- Network traffic élevé entre mongos et shards

**Cause** :
- Requêtes ne contenant pas la shard key
- Forcing de broadcast queries

**Diagnostic** :
```javascript
// Sur mongos, analyse des requêtes
db.system.profile.find({
  "execStats.stage": "SHARD_MERGE"
}).forEach(function(doc) {
  print(`Query: ${JSON.stringify(doc.command.filter)}`);
  print(`Shards contacted: ${doc.execStats.nShards}`);
})
```

### Pattern : "Checkpoint Cascade"

**Signature** :
- Pics réguliers de write latency toutes les 60 secondes
- Corrélation avec checkpoint interval
- Dirty cache ratio élevé avant les pics

**Cause** :
- WiredTiger checkpoint overhead
- Write pressure dépassant checkpoint capacity

**Diagnostic** :
```javascript
db.serverStatus().wiredTiger.transaction.checkpoint
// Analyser :
// - "transaction checkpoint max time (msecs)"
// - "transaction checkpoint min time (msecs)"
// Si max >> average : checkpoints problématiques

db.serverStatus().wiredTiger.cache["tracked dirty bytes in the cache"]
// Si > 20% du cache : write pressure élevée
```

## Conclusion

L'identification précise des goulots d'étranglement requiert :

1. **Méthodologie rigoureuse** : Approche systématique USE et en couches
2. **Observation multi-dimensionnelle** : Métriques de toutes les couches du stack
3. **Pensée corrélative** : Lier les symptômes aux causes racines
4. **Outils appropriés** : Maîtrise de mongostat, profiler, explain, FTDC
5. **Pattern recognition** : Reconnaître les signatures de problèmes connus

L'identification n'est que la première étape. Une fois le goulot identifié avec certitude, les sections suivantes aborderont les techniques spécifiques d'optimisation pour chaque type de goulot.

---

**Points clés :**
- Utiliser la méthodologie USE pour l'analyse systématique
- Ne jamais se fier aux moyennes seules, analyser les percentiles
- Corréler les métriques de différentes couches pour identifier la vraie cause racine
- Documenter les patterns observés pour reconnaissance future
- Valider les hypothèses par des tests avant d'implémenter des changements en production

⏭️ [Analyse avec explain() approfondie](/17-performance-tuning/02-analyse-explain-approfondie.md)
