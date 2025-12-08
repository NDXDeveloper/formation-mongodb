🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.6 Configuration du Moteur WiredTiger

## Introduction

WiredTiger est le moteur de stockage par défaut de MongoDB depuis la version 3.2, remplaçant MMAPv1. Sa conception moderne offre des performances supérieures, une meilleure concurrence, et des fonctionnalités avancées de compression. Cependant, sa configuration fine nécessite une compréhension approfondie de son architecture et de ses paramètres pour exploiter pleinement son potentiel en production.

Cette section explore les aspects techniques de WiredTiger, ses paramètres de configuration critiques, et les méthodologies d'optimisation pour différents profils de charge. Une configuration optimale de WiredTiger peut améliorer les performances de 2 à 10 fois selon le workload.

## Architecture de WiredTiger

### Composants Fondamentaux

WiredTiger repose sur plusieurs composants clés qui interagissent pour assurer performance et durabilité.

```
┌─────────────────────────────────────────┐
│         Application (mongod)            │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         WiredTiger API Layer            │
├─────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  Cache   │  │ Journal  │  │ Eviction │
│  │ Manager  │  │  (WAL)   │  │ Thread   │
│  └──────────┘  └──────────┘  └────────┘ │
├─────────────────────────────────────────┤
│  ┌──────────────────────────────────┐   │
│  │    Transaction Manager           │   │
│  │    (MVCC + Snapshot Isolation)   │   │
│  └──────────────────────────────────┘   │
├─────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │ B-Tree   │  │Checkpoint│  │Compress│ │
│  │ Indexes  │  │ Manager  │  │Engine  │ │
│  └──────────┘  └──────────┘  └────────┘ │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         File System (Data Files)        │
│  *.wt (data) | journal/ (WAL logs)      │
└─────────────────────────────────────────┘
```

### Cache WiredTiger

Le cache est le composant le plus critique pour les performances.

**Fonctionnement** :
- Cache unifié pour données et index (contrairement à MMAPv1)
- Gestion LRU (Least Recently Used) pour éviction
- Pages de taille variable (4KB à 32KB typiquement)
- Dirty pages tracking pour checkpoints
- Multi-threaded eviction

**Configuration par défaut** :
```javascript
// Taille cache = max(0.5 × (RAM - 1GB), 256MB)
// Exemple avec 16GB RAM :
// Cache = max(0.5 × (16 - 1), 0.256) = 7.5GB

db.serverStatus().wiredTiger.cache
{
  "maximum bytes configured": 8053063680,  // 7.5GB
  "bytes currently in the cache": 6442450944,
  "tracked dirty bytes in the cache": 524288000,
  "bytes read into cache": 4567890123456,
  "bytes written from cache": 2345678901234,
  "pages evicted by application threads": 12345,
  "pages read into cache": 234567,
  "pages written from cache": 123456
}
```

### Journal (Write-Ahead Log)

Le journal assure la durabilité des écritures en cas de crash.

**Mécanisme** :
```
1. Write Operation
   ↓
2. Write to Journal (WAL) - On disk
   ↓
3. Write acknowledged (if j:true)
   ↓
4. Write to Cache (in-memory)
   ↓
5. Eventually → Checkpoint (flush to data files)
```

**Fichiers journal** :
```bash
ls -lh /var/lib/mongodb/journal/
-rw------- 1 mongodb mongodb 100M Jan 15 10:00 WiredTigerLog.0000000001
-rw------- 1 mongodb mongodb 100M Jan 15 10:05 WiredTigerLog.0000000002
-rw------- 1 mongodb mongodb  45M Jan 15 10:08 WiredTigerLog.0000000003
```

Chaque fichier journal : 100MB par défaut
Rotation automatique lors des checkpoints

### Checkpoints

Les checkpoints écrivent périodiquement le cache dirty sur disque.

**Processus de checkpoint** :
```
1. Mark all dirty pages (snapshot)
2. Write dirty pages to data files
   (while allowing new operations to continue)
3. Update metadata (checkpoint complete)
4. Truncate old journal files
5. Start new checkpoint interval
```

**Fréquence par défaut** :
- Tous les 60 secondes OU
- Quand 2GB de journal accumulé

### MVCC et Isolation

WiredTiger utilise MVCC (Multi-Version Concurrency Control).

**Principe** :
```javascript
// Chaque document a des versions multiples
Document v1: { _id: 1, value: 100 } @ timestamp T1
Document v2: { _id: 1, value: 150 } @ timestamp T2
Document v3: { _id: 1, value: 200 } @ timestamp T3

// Transaction T (débutée à T2) voit v2
// Transaction T' (débutée à T3) voit v3
// Pas de lock nécessaire pour read-read

// Avantages :
// - Readers ne bloquent jamais writers
// - Writers ne bloquent jamais readers
// - Haute concurrence
```

**Snapshot isolation** :
- Chaque transaction voit une snapshot cohérente
- Isolation level : Read Committed par défaut
- Snapshot isolation pour transactions multi-documents

## Configuration du Cache

### Dimensionnement Optimal

Le dimensionnement du cache est la décision la plus impactante.

**Formule de base** :
```
Cache Size = min(
  0.5 × (Total RAM - 1GB),
  Working Set Size + 20% buffer
)
```

**Analyse du Working Set** :
```javascript
// 1. Taille des données fréquemment accédées
const stats = db.collection.stats();
const dataSize = stats.size;
const indexSize = stats.totalIndexSize;

// 2. Estimation du working set (hot data)
// Méthode 1 : Via cache hit ratio
const cacheStats = db.serverStatus().wiredTiger.cache;
const totalRequests = cacheStats["pages read into cache"] +
                      cacheStats["pages requested from the cache"];
const cacheHits = cacheStats["pages requested from the cache"];
const hitRatio = cacheHits / totalRequests;

// Si hitRatio < 0.95 : Cache trop petit
// Si hitRatio > 0.99 : Cache possiblement surdimensionné

// Méthode 2 : Via page faults
const serverStatus = db.serverStatus();
const pageFaults = serverStatus.extra_info.page_faults;
// Si page_faults croissants : Cache insuffisant

// 3. Working set estimé
// Généralement : 20-40% du dataset total pour workloads typiques
const estimatedWorkingSet = (dataSize + indexSize) * 0.3;

print(`Data Size: ${(dataSize / 1024 / 1024 / 1024).toFixed(2)} GB`);
print(`Index Size: ${(indexSize / 1024 / 1024 / 1024).toFixed(2)} GB`);
print(`Estimated Working Set: ${(estimatedWorkingSet / 1024 / 1024 / 1024).toFixed(2)} GB`);
print(`Current Cache Size: ${(cacheStats["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2)} GB`);
print(`Cache Hit Ratio: ${(hitRatio * 100).toFixed(2)}%`);
```

**Configuration du cache** :

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 12  # Taille explicite en GB

      # Ou en pourcentage (MongoDB 3.4+)
      # cacheSizeGB n'est pas spécifié, utilise la formule par défaut
```

```bash
# Via ligne de commande
mongod --wiredTigerCacheSizeGB 12
```

**Recommandations par scénario** :

| Scénario | RAM Totale | Cache Recommandé | Rationale |
|----------|------------|------------------|-----------|
| **Dedicated MongoDB** | 32GB | 15GB (47%) | Laisser 17GB pour OS cache, connections |
| **Shared server** | 32GB | 10GB (31%) | Partage avec autres services |
| **Large dataset (> RAM)** | 64GB | 30GB (47%) | Maximize cache pour working set |
| **Small dataset (< RAM)** | 16GB | Data+Index+2GB | Pas besoin de plus |
| **Write-heavy** | 32GB | 12GB (37%) | Laisser plus pour OS buffers |
| **Read-heavy** | 32GB | 18GB (56%) | Maximiser cache pour reads |

### Gestion de l'Éviction

L'éviction se déclenche quand le cache atteint ses limites.

**Seuils d'éviction** :
```javascript
db.serverStatus().wiredTiger.cache
{
  "maximum bytes configured": 8053063680,        // 100% (7.5GB)
  "bytes currently in the cache": 7250000000,    // 90%

  // Seuils de déclenchement
  "eviction trigger": 6442450944,                // 80% - Eviction commence
  "eviction target": 6442450944,                 // 80% - Objectif éviction

  // Dirty data thresholds
  "tracked dirty bytes in the cache": 1610612736, // 20%
  "maximum dirty bytes": 2013265920,              // 25% du cache
  "dirty trigger": 1610612736,                    // 20% - Éviction dirty pages

  // Éviction stats
  "pages evicted by application threads": 1234,
  "pages evicted because they exceeded memory": 5678,
  "pages selected for eviction unable to be evicted": 42,  // Dirty pages actives

  // Performance indicators
  "eviction walks": 234,
  "eviction worker thread active": 4
}
```

**Tunables d'éviction** (MongoDB 4.0+) :
```javascript
// Configuration avancée (réservée aux experts)
db.adminCommand({
  setParameter: 1,

  // Nombre de threads d'éviction (défaut : 4)
  // Augmenter si cache saturation fréquente
  "wiredTigerEvictionThreadsMax": 8,

  // Seuil pour commencer éviction (défaut : 80%)
  "wiredTigerEngineRuntimeConfigSetting": "eviction_trigger=85"
})

// Note : Modifications risquées, tester en staging
```

**Symptômes de cache undersized** :
```javascript
// Indicateurs d'un cache trop petit
const cacheMetrics = db.serverStatus().wiredTiger.cache;

// 1. Éviction agressive
if (cacheMetrics["pages evicted by application threads"] > 10000) {
  print("WARNING: Application threads doing eviction (cache pressure)");
}

// 2. Cache toujours plein
const currentUsage = cacheMetrics["bytes currently in the cache"];
const maxCache = cacheMetrics["maximum bytes configured"];
const usagePercent = (currentUsage / maxCache) * 100;

if (usagePercent > 95) {
  print("WARNING: Cache constantly at 95%+ (undersized)");
}

// 3. Éviction de pages non-dirty
const cleanEvictions = cacheMetrics["unmodified pages evicted"];
if (cleanEvictions > 1000000) {
  print("INFO: High clean evictions (normal under load)");
}

// 4. Stalls causés par éviction
const evictionStalls = cacheMetrics["eviction server unable to reach target"];
if (evictionStalls > 100) {
  print("CRITICAL: Eviction stalls detected (cache severely undersized)");
}
```

**Symptômes de cache oversized** :
```javascript
// Cache trop grand gaspille de la RAM
const cacheMetrics = db.serverStatus().wiredTiger.cache;
const currentUsage = cacheMetrics["bytes currently in the cache"];
const maxCache = cacheMetrics["maximum bytes configured"];
const usagePercent = (currentUsage / maxCache) * 100;

if (usagePercent < 50) {
  print("INFO: Cache usage < 50% - Consider reducing cache size");
  print("      to free RAM for OS cache or other services");
}

// Vérifier la stabilité de l'utilisation
// Si utilisation stable à 40-50% pendant 24h :
// → Cache oversized, réduire de 20-30%
```

## Configuration des Checkpoints

### Intervalle et Déclenchement

Les checkpoints impactent directement la durabilité et les performances.

**Configuration par défaut** :
```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      # Pas de configuration = défauts
      # Checkpoint tous les 60 secondes OU 2GB de journal
```

**Monitoring des checkpoints** :
```javascript
db.serverStatus().wiredTiger.transaction
{
  "transaction checkpoint generation": 12345,
  "transaction checkpoint currently running": 0,

  // Durée des checkpoints
  "transaction checkpoint max time (msecs)": 15234,   // Pire cas
  "transaction checkpoint min time (msecs)": 1234,    // Meilleur cas
  "transaction checkpoint most recent time (msecs)": 3456,

  // Fréquence
  "transaction checkpoint total time (msecs)": 234567890,
  "transaction checkpoints": 12345,

  // Volume écrit
  "transaction checkpoint scrub dirty target": 0,
  "transaction checkpoint scrub time (msecs)": 0
}
```

**Analyse de la durée des checkpoints** :
```javascript
function analyzeCheckpointPerformance() {
  const txnStats = db.serverStatus().wiredTiger.transaction;

  const avgCheckpointDuration =
    txnStats["transaction checkpoint total time (msecs)"] /
    txnStats["transaction checkpoints"];

  const maxDuration = txnStats["transaction checkpoint max time (msecs)"];
  const recentDuration = txnStats["transaction checkpoint most recent time (msecs)"];

  const analysis = {
    avgDurationMs: avgCheckpointDuration.toFixed(2),
    maxDurationMs: maxDuration,
    recentDurationMs: recentDuration,
    totalCheckpoints: txnStats["transaction checkpoints"],

    assessment: ""
  };

  if (maxDuration > 30000) {  // 30 secondes
    analysis.assessment = "CRITICAL: Very slow checkpoints (>30s)";
  } else if (maxDuration > 10000) {  // 10 secondes
    analysis.assessment = "WARNING: Slow checkpoints (>10s)";
  } else if (avgCheckpointDuration > 5000) {  // 5 secondes
    analysis.assessment = "ATTENTION: Average checkpoint duration high";
  } else {
    analysis.assessment = "OK: Checkpoint performance acceptable";
  }

  return analysis;
}

printjson(analyzeCheckpointPerformance());
```

**Causes de checkpoints lents** :

1. **Cache dirty ratio élevé** :
```javascript
const cacheStats = db.serverStatus().wiredTiger.cache;
const dirtyBytes = cacheStats["tracked dirty bytes in the cache"];
const totalCache = cacheStats["maximum bytes configured"];
const dirtyPercent = (dirtyBytes / totalCache) * 100;

if (dirtyPercent > 20) {
  print(`High dirty ratio: ${dirtyPercent.toFixed(2)}%`);
  print("Checkpoint will write many pages (slow)");
}

// Solution : Increase cache size ou reduce write pressure
```

2. **I/O saturation** :
```bash
# Vérifier I/O lors d'un checkpoint
iostat -x 5

# Si %util proche de 100% pendant checkpoint :
# → I/O bottleneck
# Solutions :
# - Faster storage (SSD NVMe)
# - RAID configuration
# - Reduce checkpoint frequency (trade durability)
```

3. **Fragmentation du cache** :
```javascript
// Pages sales dispersées nécessitent beaucoup d'I/O random
// Solution : Compaction périodique
db.runCommand({ compact: "collection" })
```

### Trade-offs Checkpoint Frequency

**Checkpoint fréquent (< 60s)** :

Avantages :
- Journal files plus petits
- Récupération après crash plus rapide
- Moins de données à rejouer

Inconvénients :
- Overhead I/O plus fréquent
- Potentielle dégradation des write performances

**Checkpoint peu fréquent (> 60s)** :

Avantages :
- Moins d'overhead I/O
- Meilleures performances write

Inconvénients :
- Journal files plus volumineux
- Récupération après crash plus lente
- Plus de RAM pour journal buffer

**Configuration ajustée** :
```javascript
// Pour workload write-heavy : Augmenter l'intervalle
// Note : Non exposé directement dans MongoDB
// Nécessite recompilation WiredTiger ou utilisation de
// configuration file WiredTiger.
// Généralement, garder le défaut (60s) est optimal

// Alternative : Ajuster le journal file size trigger
// Augmenter le seuil de 2GB à 4GB si I/O impacté
// (Nécessite MongoDB 4.4+)
```

## Configuration de la Compression

WiredTiger supporte plusieurs algorithmes de compression pour données et index.

### Algorithmes Disponibles

| Algorithme | Ratio | CPU | Vitesse | Use Case |
|------------|-------|-----|---------|----------|
| **none** | 1× | Minimal | Fastest | Low latency, fast storage |
| **snappy** | 2-3× | Low | Fast | **DEFAULT** - Balanced |
| **zlib** | 3-5× | Medium | Medium | Storage-constrained |
| **zstd** | 3-6× | Low-Med | Fast | **Best balance** (MongoDB 4.2+) |

### Configuration de Compression

**Par défaut** :
```yaml
# mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy  # Compression des collections
    indexConfig:
      prefixCompression: true  # Compression des index (prefix)
```

**Changement global** :
```yaml
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd  # Plus efficace que snappy
    indexConfig:
      prefixCompression: true
```

**Changement par collection** :
```javascript
// Lors de la création
db.createCollection("myCollection", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
})

// Pour collection existante : Nécessite recréation
// 1. Créer nouvelle collection avec compression désirée
db.createCollection("myCollection_new", {
  storageEngine: { wiredTiger: { configString: "block_compressor=zstd" } }
})

// 2. Copier les données
db.myCollection.aggregate([
  { $match: {} },
  { $out: "myCollection_new" }
])

// 3. Renommer (downtime)
db.myCollection.renameCollection("myCollection_old")
db.myCollection_new.renameCollection("myCollection")

// 4. Rebuild index
db.myCollection.getIndexes().forEach(function(index) {
  if (index.name !== "_id_") {
    var key = index.key;
    delete index.key;
    delete index.ns;
    delete index.v;
    db.myCollection.createIndex(key, index);
  }
})
```

### Analyse de l'Efficacité de Compression

```javascript
function analyzeCompression(collectionName) {
  const stats = db[collectionName].stats();

  const size = stats.size;  // Données non compressées
  const storageSize = stats.storageSize;  // Données compressées
  const compressionRatio = size / storageSize;

  const indexSize = stats.totalIndexSize;

  const analysis = {
    collection: collectionName,

    // Données
    uncompressedDataGB: (size / 1024 / 1024 / 1024).toFixed(2),
    compressedDataGB: (storageSize / 1024 / 1024 / 1024).toFixed(2),
    dataCompressionRatio: compressionRatio.toFixed(2) + "×",
    dataSavingsGB: ((size - storageSize) / 1024 / 1024 / 1024).toFixed(2),

    // Index
    indexSizeGB: (indexSize / 1024 / 1024 / 1024).toFixed(2),

    // Total
    totalOnDiskGB: ((storageSize + indexSize) / 1024 / 1024 / 1024).toFixed(2),

    compressor: stats.wiredTiger.creationString.match(/block_compressor=(\w+)/)?.[1] || "unknown"
  };

  return analysis;
}

// Usage
printjson(analyzeCompression("orders"));

// Comparaison entre algorithmes
// Tester en staging avec données production
```

### Recommandations par Workload

**Read-heavy workload** :
```yaml
# Privilégier la vitesse de décompression
blockCompressor: snappy  # Ou zstd
# CPU décompression minimal, latency optimale
```

**Write-heavy workload** :
```yaml
# Balance compression / CPU
blockCompressor: zstd
# Bon ratio, CPU raisonnable, write amplification réduite
```

**Storage-constrained** :
```yaml
# Maximiser la compression
blockCompressor: zlib
# Meilleur ratio mais plus de CPU
# Trade-off acceptable si stockage coûteux
```

**Low-latency critical** :
```yaml
# Pas de compression
blockCompressor: none
# Zero CPU overhead, latency minimale
# Seulement si storage rapide et abondant
```

### Impact Performance de la Compression

**Benchmark interne** (résultats typiques) :

```
Test Setup:
- Collection : 10M documents, 5GB uncompressed
- Hardware : SSD NVMe, 32GB RAM, 8 cores
- Operations : 50% read, 50% write

Results:
┌───────────┬──────────┬─────────────┬──────────┬──────────┐
│Compressor │ On-Disk  │ Read Latency│Write Lat │ CPU Use  │
├───────────┼──────────┼─────────────┼──────────┼──────────┤
│ none      │ 5.0 GB   │ 1.2ms       │ 1.5ms    │ 15%      │
│ snappy    │ 1.8 GB   │ 1.4ms (+17%)│ 1.7ms    │ 18%      │
│ zstd      │ 1.2 GB   │ 1.5ms (+25%)│ 1.8ms    │ 22%      │
│ zlib      │ 1.0 GB   │ 2.1ms (+75%)│ 2.5ms    │ 35%      │
└───────────┴──────────┴─────────────┴──────────┴──────────┘

Conclusion :
- snappy : Best default (balance)
- zstd : Best modern choice (compression + speed)
- zlib : Only if storage critical and CPU available
- none : Only for ultra-low latency requirements
```

## Configuration du Journal

### Journal Settings

```yaml
# mongod.conf
storage:
  journal:
    enabled: true     # Toujours true en production
    commitIntervalMs: 100  # Défaut : 100ms (MongoDB 4.0+)
```

**commitIntervalMs** : Fréquence de flush du journal sur disque

```
┌────────────────────────────────────────────┐
│ Write Operation                            │
└──────────┬─────────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────────┐
│ Journal Buffer (in-memory)                 │
│ - Accumulates writes                       │
│ - Flushed every commitIntervalMs           │
└──────────┬─────────────────────────────────┘
           │ Every 100ms (défaut)
           ▼
┌────────────────────────────────────────────┐
│ Journal Files (on-disk)                    │
│ /var/lib/mongodb/journal/*.wt              │
└────────────────────────────────────────────┘
```

**Trade-off commitIntervalMs** :

| Valeur | Durabilité | Performance | Use Case |
|--------|------------|-------------|----------|
| 50ms | Meilleure | Plus lent | Données critiques |
| 100ms | Bonne | **Balanced** | **Défaut recommandé** |
| 200ms | Acceptable | Rapide | Write-heavy, données tolérantes |
| 500ms | Risquée | Très rapide | Non recommandé production |

**Configuration pour latence minimale** :
```yaml
# Write-heavy workload, acceptable de perdre 200ms de writes
storage:
  journal:
    enabled: true
    commitIntervalMs: 200
```

**Configuration pour durabilité maximale** :
```yaml
# Financial transactions, aucune perte acceptable
storage:
  journal:
    enabled: true
    commitIntervalMs: 50

# Combiné avec write concern :
# db.collection.insertOne({...}, { writeConcern: { w: "majority", j: true } })
# = Latency accrue mais garantie maximale
```

### Désactivation du Journal (Non Recommandé)

```yaml
# ⚠️ NE PAS FAIRE EN PRODUCTION
storage:
  journal:
    enabled: false
```

**Conséquences** :
- Perte de données possible en cas de crash
- Réparation database nécessaire après crash
- Récupération impossible
- **Gain de performance** : 10-30% sur writes
- **Acceptable uniquement** : Données non-critiques, réplicables

**Cas d'usage acceptable** :
- Cache layer (données reconstituables)
- Test/development environments
- Secondary read-only (avec copy de production)

## Tuning Avancé par Workload

### Read-Heavy Workload

**Caractéristiques** : 90% reads, 10% writes

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 24  # 75% de RAM (32GB total)
      # Maximiser cache pour working set

    collectionConfig:
      blockCompressor: snappy  # Fast décompression

    indexConfig:
      prefixCompression: true

# Journal moins critique pour performance
storage:
  journal:
    enabled: true
    commitIntervalMs: 100  # Défaut OK
```

**Paramètres complémentaires** :
```javascript
// Read Preference sur replica set : Distribuer reads
// Dans application :
db.collection.find({...}).readPref("secondaryPreferred")

// Augmenter connection pool size
// Pour absorber pic de read requests
```

**Monitoring focus** :
- Cache hit ratio (target : >98%)
- Read latency P95, P99
- Éviction rate (doit être minimal)

### Write-Heavy Workload

**Caractéristiques** : 30% reads, 70% writes

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 16  # 50% de RAM
      # Laisser RAM pour OS cache (write buffering)

    collectionConfig:
      blockCompressor: zstd  # Meilleure compression
      # Réduit write amplification

    indexConfig:
      prefixCompression: true

storage:
  journal:
    enabled: true
    commitIntervalMs: 150  # Légèrement augmenté
    # Trade durabilité/performance
```

**Paramètres WiredTiger avancés** :
```javascript
// Augmenter dirty page threshold
db.adminCommand({
  setParameter: 1,
  // Permettre plus de dirty pages avant éviction
  "wiredTigerEngineRuntimeConfigSetting": "eviction_dirty_trigger=25"
})

// Défaut : 20%
// Augmenter à 25% pour write-heavy
// Checkpoints écriront plus mais moins fréquemment
```

**Storage optimization** :
```bash
# Filesystem : XFS recommandé (meilleur que ext4 pour writes)
# Mount options :
# noatime : Ne pas update access time (réduit writes)
mkfs.xfs /dev/sdb
mount -o noatime /dev/sdb /var/lib/mongodb
```

**Monitoring focus** :
- Write latency P95, P99
- Checkpoint duration
- Journal size growth
- Dirty cache ratio

### Mixed Workload (OLTP)

**Caractéristiques** : 60% reads, 40% writes, latence critique

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 20  # 62% de RAM (32GB total)
      # Balance read cache et write buffer

    collectionConfig:
      blockCompressor: zstd  # Best balance

    indexConfig:
      prefixCompression: true

storage:
  journal:
    enabled: true
    commitIntervalMs: 100  # Défaut
```

**Optimisations** :
```javascript
// 1. Index optimization (covered queries)
db.collection.createIndex({ field1: 1, field2: 1, field3: 1 })

// 2. Write batching dans application
// Batch inserts au lieu de insertOne répétés
db.collection.insertMany(batch, { ordered: false })

// 3. Connection pooling approprié
// minPoolSize : 10
// maxPoolSize : 100
```

**Monitoring focus** :
- Overall latency P50, P95, P99
- Cache hit ratio
- Checkpoint impact sur latency
- Lock contention

### Analytics / Reporting Workload

**Caractéristiques** : Agrégations complexes, gros scans

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 28  # 87% de RAM
      # Maximiser pour datasets en mémoire

    collectionConfig:
      blockCompressor: zlib  # Haute compression
      # Datasets volumineux

    indexConfig:
      prefixCompression: true

storage:
  journal:
    enabled: true
    commitIntervalMs: 200  # Writes peu fréquents
```

**Stratégies complémentaires** :
```javascript
// 1. Agrégations avec allowDiskUse
db.collection.aggregate([...], { allowDiskUse: true })

// 2. Dedicated analytics secondary
// Replica set : 1 secondary hidden pour analytics
cfg = rs.conf()
cfg.members[2].priority = 0
cfg.members[2].hidden = true
rs.reconfig(cfg)

// 3. Read from secondary pour analytics
db.collection.find().readPref("secondary").hint(index)
```

## Monitoring et Métriques WiredTiger

### Métriques Critiques

```javascript
function wiredTigerHealthCheck() {
  const wt = db.serverStatus().wiredTiger;

  const health = {
    timestamp: new Date(),

    // Cache health
    cache: {
      sizeGB: (wt.cache["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2),
      usagePercent: ((wt.cache["bytes currently in the cache"] /
                      wt.cache["maximum bytes configured"]) * 100).toFixed(2),
      dirtyPercent: ((wt.cache["tracked dirty bytes in the cache"] /
                      wt.cache["maximum bytes configured"]) * 100).toFixed(2),

      // Indicateurs de santé
      evictionRate: wt.cache["pages evicted by application threads"],
      cacheOverflowEvictions: wt.cache["pages evicted because they exceeded memory"],

      assessment: ""
    },

    // Transaction health
    transaction: {
      checkpoints: wt.transaction["transaction checkpoints"],
      avgCheckpointMs: (wt.transaction["transaction checkpoint total time (msecs)"] /
                        wt.transaction["transaction checkpoints"]).toFixed(2),
      maxCheckpointMs: wt.transaction["transaction checkpoint max time (msecs)"],
      recentCheckpointMs: wt.transaction["transaction checkpoint most recent time (msecs)"],

      assessment: ""
    },

    // Block manager (I/O)
    blockManager: {
      blocksRead: wt.block_manager["blocks read"],
      blocksWritten: wt.block_manager["blocks written"],
      bytesReadMB: (wt.block_manager["bytes read"] / 1024 / 1024).toFixed(2),
      bytesWrittenMB: (wt.block_manager["bytes written"] / 1024 / 1024).toFixed(2),
    }
  };

  // Cache assessment
  if (health.cache.usagePercent > 95) {
    health.cache.assessment = "CRITICAL: Cache constantly full";
  } else if (health.cache.usagePercent > 85) {
    health.cache.assessment = "WARNING: Cache under pressure";
  } else if (health.cache.usagePercent < 50) {
    health.cache.assessment = "INFO: Cache underutilized";
  } else {
    health.cache.assessment = "OK";
  }

  if (health.cache.dirtyPercent > 25) {
    health.cache.assessment += " | High dirty ratio";
  }

  if (health.cache.evictionRate > 10000) {
    health.cache.assessment += " | High eviction pressure";
  }

  // Transaction assessment
  if (health.transaction.maxCheckpointMs > 30000) {
    health.transaction.assessment = "CRITICAL: Very slow checkpoints";
  } else if (health.transaction.avgCheckpointMs > 10000) {
    health.transaction.assessment = "WARNING: Slow checkpoints";
  } else {
    health.transaction.assessment = "OK";
  }

  return health;
}

// Exécuter périodiquement
printjson(wiredTigerHealthCheck());
```

### Alerting Rules

**Prometheus-style alerting rules** :

```yaml
# Cache alerts
- alert: WiredTigerCachePressure
  expr: |
    (mongodb_wiredtiger_cache_bytes_currently_in_cache /
     mongodb_wiredtiger_cache_maximum_bytes_configured) > 0.95
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "WiredTiger cache under pressure"
    description: "Cache usage > 95% for 5 minutes"

- alert: WiredTigerHighDirtyRatio
  expr: |
    (mongodb_wiredtiger_cache_tracked_dirty_bytes_in_cache /
     mongodb_wiredtiger_cache_maximum_bytes_configured) > 0.25
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "High dirty page ratio in cache"

- alert: WiredTigerSlowCheckpoint
  expr: |
    mongodb_wiredtiger_txn_checkpoint_most_recent_time_msecs > 15000
  labels:
    severity: warning
  annotations:
    summary: "Slow WiredTiger checkpoint detected"
    description: "Last checkpoint took > 15 seconds"

- alert: WiredTigerEvictionStorm
  expr: |
    rate(mongodb_wiredtiger_cache_pages_evicted_by_application_threads[5m]) > 1000
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High eviction rate - cache undersized"
```

### Diagnostic Scripts

```javascript
// Script de diagnostic complet
function fullWiredTigerDiagnostic() {
  const wt = db.serverStatus().wiredTiger;
  const serverStatus = db.serverStatus();

  print("=== WiredTiger Full Diagnostic ===\n");

  // 1. Cache Analysis
  print("1. CACHE ANALYSIS");
  print("-----------------");
  const cacheSize = wt.cache["maximum bytes configured"];
  const cacheUsed = wt.cache["bytes currently in the cache"];
  const cacheDirty = wt.cache["tracked dirty bytes in the cache"];

  print(`Cache Size: ${(cacheSize / 1024 / 1024 / 1024).toFixed(2)} GB`);
  print(`Cache Used: ${(cacheUsed / 1024 / 1024 / 1024).toFixed(2)} GB (${((cacheUsed/cacheSize)*100).toFixed(2)}%)`);
  print(`Dirty Pages: ${(cacheDirty / 1024 / 1024 / 1024).toFixed(2)} GB (${((cacheDirty/cacheSize)*100).toFixed(2)}%)`);

  const totalReads = wt.cache["pages read into cache"];
  const totalRequests = wt.cache["pages requested from the cache"];
  const hitRatio = ((totalRequests - totalReads) / totalRequests * 100).toFixed(2);
  print(`Cache Hit Ratio: ${hitRatio}%`);

  // 2. Eviction Analysis
  print("\n2. EVICTION ANALYSIS");
  print("--------------------");
  print(`Application thread evictions: ${wt.cache["pages evicted by application threads"]}`);
  print(`Eviction server evictions: ${wt.cache["pages evicted by the page eviction server"]}`);
  print(`Modified pages evicted: ${wt.cache["modified pages evicted"]}`);
  print(`Unmodified pages evicted: ${wt.cache["unmodified pages evicted"]}`);

  // 3. Checkpoint Analysis
  print("\n3. CHECKPOINT ANALYSIS");
  print("----------------------");
  const totalCheckpoints = wt.transaction["transaction checkpoints"];
  const totalCheckpointTime = wt.transaction["transaction checkpoint total time (msecs)"];
  const avgCheckpoint = (totalCheckpointTime / totalCheckpoints).toFixed(2);

  print(`Total Checkpoints: ${totalCheckpoints}`);
  print(`Average Duration: ${avgCheckpoint} ms`);
  print(`Max Duration: ${wt.transaction["transaction checkpoint max time (msecs)"]} ms`);
  print(`Most Recent: ${wt.transaction["transaction checkpoint most recent time (msecs)"]} ms`);

  // 4. I/O Analysis
  print("\n4. I/O ANALYSIS");
  print("---------------");
  print(`Blocks Read: ${wt.block_manager["blocks read"]}`);
  print(`Blocks Written: ${wt.block_manager["blocks written"]}`);
  print(`Bytes Read: ${(wt.block_manager["bytes read"] / 1024 / 1024 / 1024).toFixed(2)} GB`);
  print(`Bytes Written: ${(wt.block_manager["bytes written"] / 1024 / 1024 / 1024).toFixed(2)} GB`);

  // 5. Concurrency Analysis
  print("\n5. CONCURRENCY ANALYSIS");
  print("-----------------------");
  print(`Active Transactions: ${wt.concurrentTransactions.read.out + wt.concurrentTransactions.write.out}`);
  print(`Read Tickets Available: ${wt.concurrentTransactions.read.available}`);
  print(`Write Tickets Available: ${wt.concurrentTransactions.write.available}`);

  // 6. Recommendations
  print("\n6. RECOMMENDATIONS");
  print("------------------");

  const recommendations = [];

  if (cacheUsed / cacheSize > 0.95) {
    recommendations.push("⚠️  Cache constantly full - Consider increasing cache size");
  }

  if (hitRatio < 95) {
    recommendations.push("⚠️  Low cache hit ratio - Working set may exceed cache");
  }

  if (cacheDirty / cacheSize > 0.25) {
    recommendations.push("⚠️  High dirty ratio - Write pressure high");
  }

  if (wt.cache["pages evicted by application threads"] > 10000) {
    recommendations.push("⚠️  Application doing eviction - Cache pressure critical");
  }

  if (wt.transaction["transaction checkpoint max time (msecs)"] > 30000) {
    recommendations.push("⚠️  Very slow checkpoints - Check I/O performance");
  }

  if (avgCheckpoint > 10000) {
    recommendations.push("⚠️  Slow average checkpoint time - Consider faster storage");
  }

  if (recommendations.length === 0) {
    print("✅ WiredTiger configuration appears healthy");
  } else {
    recommendations.forEach(rec => print(rec));
  }

  print("\n=================================\n");
}

// Exécution
fullWiredTigerDiagnostic();
```

## Configuration de Référence par Environnement

### Production Standard

```yaml
# mongod.conf - Production 32GB RAM, SSD
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100

  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 15
      journalCompressor: snappy
      directoryForIndexes: false

    collectionConfig:
      blockCompressor: zstd

    indexConfig:
      prefixCompression: true

# Filesystem
# XFS, mounted with: noatime,nodiratime

# ulimits
# file descriptors: 64000
# processes: 64000
```

### High-Performance OLTP

```yaml
# mongod.conf - OLTP 64GB RAM, NVMe SSD
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100  # Peut descendre à 50 si nécessaire

  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 40  # 62% de RAM
      journalCompressor: snappy

    collectionConfig:
      blockCompressor: snappy  # Latence minimale

    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

# Hardware :
# - NVMe SSD (>500k IOPS)
# - Dedicated CPU cores
# - Low-latency network
```

### Analytics / DW

```yaml
# mongod.conf - Analytics 128GB RAM, SATA SSD
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
    commitIntervalMs: 200  # Writes peu fréquents

  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 80  # 62% de RAM
      journalCompressor: zlib  # Compression max

    collectionConfig:
      blockCompressor: zlib  # Compression max

    indexConfig:
      prefixCompression: true

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 1000  # Analytics queries sont longues

setParameter:
  internalQueryExecMaxBlockingSortBytes: 335544320  # 320MB pour sorts
```

### Development / Staging

```yaml
# mongod.conf - Development 8GB RAM
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100

  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 3

    collectionConfig:
      blockCompressor: snappy

    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  verbosity: 1  # Plus verbose pour debugging
```

## Conclusion

La configuration optimale de WiredTiger repose sur plusieurs piliers :

1. **Cache dimensioning** : Le paramètre le plus impactant
   - Target : 95%+ cache hit ratio
   - Working set doit tenir en cache
   - Laisser RAM pour OS et overhead

2. **Compression strategy** : Balance storage/CPU
   - zstd : Meilleur choix moderne
   - snappy : Si CPU limité
   - zlib : Si storage critique

3. **Checkpoint tuning** : Durabilité vs Performance
   - Défaut (60s) optimal pour la plupart
   - Augmenter si write-heavy et I/O limite
   - Diminuer si durabilité critique

4. **Journal configuration** : Garanties de durabilité
   - commitIntervalMs : 100ms défaut optimal
   - Toujours enabled en production
   - Ajuster selon criticité des données

5. **Monitoring continu** : Ajustement basé sur métriques
   - Cache hit ratio
   - Checkpoint duration
   - Eviction pressure
   - Dirty page ratio

**Méthodologie d'optimisation** :
1. Établir baseline avec configuration défaut
2. Identifier le goulot (cache, I/O, CPU)
3. Ajuster paramètres progressivement
4. Mesurer l'impact
5. Itérer

**Erreurs courantes à éviter** :
- Cache trop grand (gaspille RAM)
- Cache trop petit (thrashing)
- Désactiver journal en production
- Over-tuning sans mesure
- Ignorer les métriques WiredTiger

L'optimisation WiredTiger est un processus itératif basé sur les métriques réelles du workload. Les configurations de référence fournissent un point de départ, mais chaque système nécessite des ajustements spécifiques basés sur ses caractéristiques uniques.

---

**Points clés à retenir :**
- Cache size est le paramètre le plus critique (target : working set + 20%)
- zstd offre le meilleur ratio compression/performance (MongoDB 4.2+)
- Checkpoints de >10s indiquent un problème (I/O ou dirty ratio)
- Cache hit ratio < 95% suggère cache undersized ou index manquants
- Journal commitIntervalMs à 100ms est optimal pour la plupart des cas
- Monitoring WiredTiger metrics est essentiel pour optimisation continue
- Configuration doit être adaptée au workload spécifique (read/write ratio)

⏭️ [Dimensionnement matériel](/17-performance-tuning/07-dimensionnement-materiel.md)
