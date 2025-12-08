🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.7 Dimensionnement Matériel

## Introduction

Le dimensionnement matériel est une décision architecturale fondamentale qui impacte directement les performances, les coûts et la scalabilité de MongoDB. Contrairement aux bases relationnelles traditionnelles, MongoDB tire pleinement parti du matériel moderne (multi-core, SSD, réseau haute vitesse) grâce à son architecture lock-free et son moteur de stockage WiredTiger.

Cette section explore les méthodologies de dimensionnement matériel pour MongoDB, en équilibrant performance, coût et évolutivité. Un dimensionnement optimal peut réduire les coûts de 30-50% tout en améliorant les performances de 2-5×.

## Méthodologie de Dimensionnement

### Approche Top-Down

```
1. ANALYSER LE WORKLOAD
   └─> Volume de données (actuel + croissance)
   └─> Patterns d'accès (read/write ratio)
   └─> Latence requise (P95, P99)
   └─> Throughput cible (ops/sec)
   └─> Durabilité et disponibilité (SLA)

2. DIMENSIONNER LES RESSOURCES
   └─> RAM (working set + overhead)
   └─> CPU (concurrence + calculs)
   └─> Storage (IOPS + capacité + latence)
   └─> Network (bandwidth + latency)

3. VALIDER PAR BENCHMARKING
   └─> POC avec données réelles
   └─> Tests de charge
   └─> Mesure vs requirements

4. PROVISIONNER AVEC MARGE
   └─> Croissance (6-12 mois)
   └─> Pics de charge (2-3×)
   └─> Incidents (N-1 capacity)
```

### Métrique de Sizing Fondamentale

**Working Set Size (WSS)** : Le volume de données fréquemment accédées.

```javascript
// Calcul du Working Set
function calculateWorkingSet() {
  const collections = db.getCollectionNames();
  let totalWss = 0;

  collections.forEach(collName => {
    const stats = db[collName].stats();

    // Données chaudes (heuristic : 30% du dataset)
    const hotDataRatio = 0.3;  // À ajuster selon le workload
    const hotData = stats.size * hotDataRatio;

    // Index (généralement tous en RAM)
    const indexSize = stats.totalIndexSize;

    const collectionWss = hotData + indexSize;

    print(`${collName}:`);
    print(`  Total Data: ${(stats.size / 1024 / 1024 / 1024).toFixed(2)} GB`);
    print(`  Hot Data (30%): ${(hotData / 1024 / 1024 / 1024).toFixed(2)} GB`);
    print(`  Index Size: ${(indexSize / 1024 / 1024 / 1024).toFixed(2)} GB`);
    print(`  WSS: ${(collectionWss / 1024 / 1024 / 1024).toFixed(2)} GB`);

    totalWss += collectionWss;
  });

  return {
    totalWssGB: (totalWss / 1024 / 1024 / 1024).toFixed(2),
    recommendation: getRAMRecommendation(totalWss)
  };
}

function getRAMRecommendation(wss) {
  // RAM = WSS + WiredTiger overhead + OS + Connections
  const wtCacheSize = wss * 1.2;  // +20% buffer
  const osMemory = 2 * 1024 * 1024 * 1024;  // 2GB pour OS
  const connectionOverhead = 1 * 1024 * 1024 * 1024;  // 1GB pour connections

  const totalRAM = wtCacheSize + osMemory + connectionOverhead;

  return {
    minimumRAM: (totalRAM / 1024 / 1024 / 1024).toFixed(2) + " GB",
    recommendedRAM: (totalRAM * 1.3 / 1024 / 1024 / 1024).toFixed(2) + " GB"  // +30% marge
  };
}

// Exécution
const wss = calculateWorkingSet();
printjson(wss);
```

## Dimensionnement de la RAM

### Calcul de la RAM Nécessaire

La RAM est la ressource la plus critique pour MongoDB.

**Formule de base** :
```
RAM Total = WiredTiger Cache + OS Cache + Connections Overhead + Safety Margin

Où :
- WiredTiger Cache = Working Set × 1.2 (minimum)
- OS Cache = 1-2 GB
- Connections Overhead = (Max Connections × 1 MB)
- Safety Margin = 20-30% du total
```

**Exemple de calcul** :
```
Dataset : 500 GB
Working Set : 150 GB (30% hot data + tous les index)

WiredTiger Cache : 150 GB × 1.2 = 180 GB
OS Cache : 2 GB
Connections : 1000 × 1 MB = 1 GB
Subtotal : 183 GB

RAM recommandée : 183 × 1.3 = 238 GB
→ Provisionner : 256 GB (taille standard)
```

### Cas Particuliers par Workload

#### Workload Read-Heavy

```
Caractéristiques :
- 90% reads, 10% writes
- Latence critique : P99 < 50ms
- Cache hit ratio target : >99%

Dimensionnement :
RAM = (Hot Data + All Indexes) × 1.3 + Overhead

Exemple :
- Dataset : 1 TB
- Hot Data : 200 GB (20%)
- Indexes : 100 GB
- RAM = (200 + 100) × 1.3 + 3 GB = 393 GB
→ Provisionner : 384-512 GB
```

**Validation** :
```javascript
// Après déploiement, vérifier cache hit ratio
const cacheStats = db.serverStatus().wiredTiger.cache;
const totalReads = cacheStats["pages read into cache"];
const totalRequests = cacheStats["pages requested from the cache"];
const hitRatio = ((totalRequests - totalReads) / totalRequests * 100).toFixed(2);

print(`Cache Hit Ratio: ${hitRatio}%`);
// Target : >99% pour read-heavy
```

#### Workload Write-Heavy

```
Caractéristiques :
- 30% reads, 70% writes
- Dirty page ratio élevé
- Checkpoints fréquents

Dimensionnement :
RAM = Working Set × 1.5 + OS Cache (2-4GB) + Overhead

Ratio plus élevé car :
- Dirty pages occupent le cache
- OS cache buffer pour writes
- Checkpoints nécessitent buffer

Exemple :
- Dataset : 2 TB
- Working Set : 400 GB
- RAM = 400 × 1.5 + 4 GB + 3 GB = 607 GB
→ Provisionner : 512-768 GB
```

#### Workload Analytics

```
Caractéristiques :
- Agrégations complexes
- Full table scans fréquents
- Allowdiskuse nécessaire

Dimensionnement :
RAM = max(Working Set × 1.3, Dataset si < 500GB) + Overhead

Analytics bénéficie de dataset complet en RAM si possible

Exemple 1 (Small dataset) :
- Dataset : 200 GB
- RAM = 200 × 1.3 + 3 GB = 263 GB
→ Provisionner : 256 GB (dataset presque complet en RAM)

Exemple 2 (Large dataset) :
- Dataset : 5 TB
- Working Set : 500 GB
- RAM = 500 × 1.3 + 3 GB = 653 GB
→ Provisionner : 512-768 GB (accepter cache misses)
```

### Formules par Réplication/Sharding

#### Replica Set (3 nœuds)

```
Chaque membre : RAM identique
- Primary : Doit gérer la charge complète
- Secondaries : Doivent pouvoir devenir Primary

RAM par nœud = RAM calculée

Coût total = 3 × RAM par nœud

Exemple :
- RAM requise : 256 GB
- Configuration : 3 × 256 GB = 768 GB total
```

#### Sharded Cluster

```
Calcul par shard :
RAM par shard = (Working Set Total / Nombre de Shards) × 1.3 + Overhead

Exemple :
- Working Set Total : 1 TB
- Shards : 4
- RAM par shard = (1000 / 4) × 1.3 + 3 = 328 GB
→ 4 shards × 384 GB = 1.5 TB total

Config Servers : 16-32 GB (metadata seulement)
Mongos : 8-16 GB (routing seulement)
```

### RAM Monitoring et Ajustement

**Métriques de validation** :

```javascript
function validateRAMSizing() {
  const serverStatus = db.serverStatus();
  const cacheStats = serverStatus.wiredTiger.cache;
  const mem = serverStatus.mem;

  const metrics = {
    // RAM physique
    totalRAMGB: (mem.resident / 1024).toFixed(2),

    // Cache WiredTiger
    cacheSizeGB: (cacheStats["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2),
    cacheUsedGB: (cacheStats["bytes currently in the cache"] / 1024 / 1024 / 1024).toFixed(2),
    cacheUsagePercent: ((cacheStats["bytes currently in the cache"] /
                         cacheStats["maximum bytes configured"]) * 100).toFixed(2),

    // Cache performance
    hitRatio: (() => {
      const reads = cacheStats["pages read into cache"];
      const requests = cacheStats["pages requested from the cache"];
      return ((requests - reads) / requests * 100).toFixed(2);
    })(),

    // Éviction pressure
    evictionsByApp: cacheStats["pages evicted by application threads"],

    // Page faults (OS level)
    pageFaults: serverStatus.extra_info.page_faults
  };

  // Assessment
  const issues = [];

  if (metrics.cacheUsagePercent > 95) {
    issues.push("⚠️ Cache constantly full - Consider increasing RAM");
  }

  if (metrics.hitRatio < 95) {
    issues.push("⚠️ Low cache hit ratio - Working set > RAM");
  }

  if (metrics.evictionsByApp > 10000) {
    issues.push("⚠️ High eviction pressure - RAM undersized");
  }

  if (metrics.pageFaults > 1000) {
    issues.push("⚠️ High page faults - OS swapping (critical)");
  }

  metrics.assessment = issues.length === 0 ? "✅ RAM sizing appears adequate" : issues;

  return metrics;
}

printjson(validateRAMSizing());
```

**Critères d'augmentation RAM** :

```
Augmenter RAM si :
1. Cache hit ratio < 95% pendant >1 semaine
2. Éviction pressure élevée constante
3. Page faults en augmentation
4. Latency P99 dégradée et corrélée au cache

Formule d'augmentation :
Nouvelle RAM = RAM actuelle × (1 + (100 - Hit Ratio) / 100)

Exemple :
RAM actuelle : 256 GB
Hit ratio : 92%
Nouvelle RAM = 256 × (1 + (100-92)/100) = 276 GB
→ Augmenter à 384 GB (taille standard suivante)
```

## Dimensionnement du CPU

### Calcul des Cores Nécessaires

Le CPU impacte la concurrence et le throughput.

**Facteurs déterminants** :
1. Concurrence (connexions simultanées)
2. Complexité des queries (agrégations, sorts)
3. Compression/décompression (WiredTiger)
4. Réplication (oplog application)

**Formule de base** :
```
Cores requis = (Connections concurrentes / Connections par core) + Overhead

Où :
- Connections par core : 50-100 (selon workload)
- Overhead : 2-4 cores (OS, monitoring, background tasks)
```

**Exemple de calcul** :

```
Workload OLTP :
- Peak connections : 2000
- Moyenne sustained : 1000
- Connections par core : 75 (workload mixte)

Cores = (1000 / 75) + 4 = 17 cores
→ Provisionner : 16-24 cores

Workload Analytics :
- Peak connections : 200
- Agrégations complexes (CPU-intensive)
- Connections par core : 25 (agrégations lourdes)

Cores = (200 / 25) + 4 = 12 cores
→ Provisionner : 12-16 cores
```

### Recommandations par Workload

| Workload | Cores Min | Cores Recommandé | Rationale |
|----------|-----------|------------------|-----------|
| **Read-Heavy Simple** | 8 | 12-16 | Simple queries, peu de CPU |
| **Write-Heavy** | 12 | 16-24 | Compression, checkpoints |
| **OLTP Mixte** | 16 | 24-32 | Haute concurrence |
| **Analytics** | 16 | 32-48 | Agrégations CPU-intensive |
| **Sharded Cluster (par shard)** | 12 | 16-24 | Distribution de charge |

### CPU Architecture Considerations

**Clock Speed vs Core Count** :

```
Single-thread performance :
- MongoDB opérations bénéficient de clock speed élevé
- Préférer : 3.0+ GHz base, 4.0+ GHz turbo

Multi-core scaling :
- Excellent scaling jusqu'à 24-32 cores
- Au-delà : Rendements décroissants
- Optimal : 16-32 cores à haute fréquence

Exemple de choix :
❌ Mauvais : 64 cores @ 2.0 GHz
✅ Bon : 24 cores @ 3.5 GHz (turbo 4.2 GHz)
```

**NUMA Architecture** :

```yaml
# Problématique : NUMA (Non-Uniform Memory Access)
# Sur serveurs multi-socket, memory access latency varie

# Solution : Désactiver NUMA ou configurer interleaving
# /etc/default/grub
GRUB_CMDLINE_LINUX="numa=off"

# Ou via numactl
numactl --interleave=all mongod --config /etc/mongod.conf

# Impact :
# Sans optimisation : 20-30% performance loss
# Avec optimisation : Performance uniforme
```

### CPU Monitoring

```javascript
function analyzeCPUUtilization() {
  // Note : MongoDB ne fournit pas directement CPU usage
  // Utiliser outils OS : top, htop, vmstat

  const serverStatus = db.serverStatus();

  // Indicateurs indirects de CPU pressure
  const metrics = {
    // Connexions actives (proxy de CPU usage)
    activeConnections: serverStatus.connections.active,

    // Operations en cours
    currentOps: db.currentOp({ "$all": true }).inprog.length,

    // Queued operations (sign de saturation)
    queuedReads: serverStatus.globalLock.currentQueue.readers,
    queuedWrites: serverStatus.globalLock.currentQueue.writers,

    // Opérations par seconde (throughput)
    opsPerSecond: {
      insert: serverStatus.opcounters.insert,
      query: serverStatus.opcounters.query,
      update: serverStatus.opcounters.update,
      delete: serverStatus.opcounters.delete,
      command: serverStatus.opcounters.command
    }
  };

  // Assessment
  const issues = [];

  if (metrics.queuedReads + metrics.queuedWrites > 100) {
    issues.push("⚠️ High queue depth - Possible CPU saturation");
  }

  if (metrics.currentOps > 1000) {
    issues.push("⚠️ Very high concurrent operations - Monitor CPU");
  }

  metrics.assessment = issues.length === 0 ?
    "✅ No obvious CPU pressure indicators" : issues;

  return metrics;
}

printjson(analyzeCPUUtilization());
```

**Monitoring externe** :

```bash
# CPU utilization par core
mpstat -P ALL 5

# Si %idle < 10% sustained : CPU saturation
# Si %iowait > 20% : I/O bottleneck (pas CPU)
# Si %sys > 30% : Kernel overhead (check context switches)

# Context switches (indicateur de contention)
vmstat 5

# Si cs > 100000 : Haute contention
# Solution : Augmenter cores ou optimiser queries
```

**Critères d'augmentation CPU** :

```
Augmenter CPU si :
1. CPU usage sustained > 80%
2. Queue depth > 50 régulièrement
3. Latency P99 corrélée avec CPU spikes
4. Agrégations timeouts fréquents

Scaling :
- CPU est moins coûteux que RAM
- Doubler cores améliore typiquement throughput de 1.7-1.9×
- Rendements décroissants au-delà de 32 cores
```

## Dimensionnement du Stockage

### Types de Stockage

| Type | IOPS | Latency | Throughput | Coût | Use Case |
|------|------|---------|------------|------|----------|
| **HDD (7200 RPM)** | 100-200 | 10-20ms | 100-200 MB/s | $ | Archives, cold data |
| **HDD (15K RPM)** | 200-400 | 5-10ms | 200-300 MB/s | $$ | Legacy, non recommandé |
| **SATA SSD** | 10K-50K | 0.1-1ms | 500-600 MB/s | $$$ | Dev/Test, budget |
| **NVMe SSD** | 100K-1M+ | 0.01-0.1ms | 3-7 GB/s | $$$$ | **Production** |
| **Cloud EBS (gp3)** | 16K | 1-5ms | 1 GB/s | $$$ | AWS standard |
| **Cloud EBS (io2)** | 64K+ | <1ms | 4 GB/s | $$$$ | AWS premium |

### Calcul de la Capacité Nécessaire

**Formule de base** :
```
Storage Total = Data Size × (1 + Growth Rate × Projection Period) × Replication Factor × Overhead

Où :
- Data Size : Taille actuelle compressée
- Growth Rate : % mensuel de croissance
- Projection Period : 12-24 mois
- Replication Factor : 3 pour replica set
- Overhead : 1.3 (journal, oplog, fragmentation, index)
```

**Exemple de calcul** :

```
Scénario :
- Data Size actuel : 500 GB
- Growth Rate : 5% / mois
- Projection : 12 mois
- Replica Set : 3 membres

Calcul :
Croissance = 500 × (1 + 0.05)^12 = 500 × 1.796 = 898 GB
Par membre = 898 × 1.3 = 1167 GB
Total cluster = 1167 × 3 = 3501 GB

→ Provisionner : 1.5 TB par nœud × 3 = 4.5 TB total
```

**Oplog sizing** :

```javascript
// Calcul oplog size recommandé
function calculateOplogSize() {
  const replStatus = rs.status();
  const oplogs = db.getSiblingDB("local").oplog.rs;

  // Taille actuelle oplog
  const oplogStats = oplogs.stats();
  const oplogSizeGB = oplogStats.size / 1024 / 1024 / 1024;

  // Window actuelle
  const firstEntry = oplogs.find().sort({$natural: 1}).limit(1).next();
  const lastEntry = oplogs.find().sort({$natural: -1}).limit(1).next();
  const windowHours = (lastEntry.ts.getTime() - firstEntry.ts.getTime()) / 3600000;

  // Write rate
  const writeOps = db.serverStatus().opcounters.insert +
                   db.serverStatus().opcounters.update +
                   db.serverStatus().opcounters.delete;

  const analysis = {
    currentOplogSizeGB: oplogSizeGB.toFixed(2),
    currentWindowHours: windowHours.toFixed(2),

    // Recommandation : 24-48h window
    recommendedFor24h: (oplogSizeGB * 24 / windowHours).toFixed(2) + " GB",
    recommendedFor48h: (oplogSizeGB * 48 / windowHours).toFixed(2) + " GB",

    assessment: windowHours < 24 ?
      "⚠️ Oplog window < 24h - Consider increasing" :
      "✅ Oplog window adequate"
  };

  return analysis;
}

printjson(calculateOplogSize());
```

**Recommandation oplog** :
```
Minimum : 24h window (maintenance/backup)
Recommandé : 48-72h window
Write-heavy : 96h+ window

Formule :
Oplog Size = Write Rate (GB/h) × Window (hours) × 1.5

Exemple :
- Write rate : 2 GB/h
- Window target : 48h
- Oplog = 2 × 48 × 1.5 = 144 GB
→ Configurer : 150-200 GB
```

### Calcul des IOPS Requis

**Formule** :
```
IOPS Required = (Reads/sec × Read IOPS per op) + (Writes/sec × Write IOPS per op)

Où :
- Read IOPS per op : 1-5 (index lookup + fetch)
- Write IOPS per op : 2-10 (data + index + journal)
```

**Exemple de calcul** :

```
Workload :
- Reads : 5000 ops/sec
- Writes : 2000 ops/sec

Calcul :
Read IOPS = 5000 × 3 = 15,000 IOPS
Write IOPS = 2000 × 6 = 12,000 IOPS
Total = 27,000 IOPS

→ Provisionner : 30,000-40,000 IOPS (avec marge)

Choix de storage :
- NVMe SSD : 100K+ IOPS → Largement suffisant
- SATA SSD : 10-50K IOPS → Limite
- HDD : 200 IOPS → Totalement insuffisant
```

### Performance par Type de Storage

**Benchmark interne** (résultats typiques MongoDB) :

```
Test Setup : Mixed workload (60% read, 40% write), 1000 connections

┌─────────────┬──────────────┬──────────────┬─────────────┐
│ Storage     │ Read Latency │ Write Latency│ Throughput  │
│             │ (P99)        │ (P99)        │ (ops/sec)   │
├─────────────┼──────────────┼──────────────┼─────────────┤
│ HDD 7.2K    │ 45ms         │ 85ms         │ 500         │
│ SATA SSD    │ 5ms          │ 12ms         │ 8,000       │
│ NVMe SSD    │ 0.8ms        │ 2.5ms        │ 45,000      │
│ AWS gp3     │ 3ms          │ 8ms          │ 12,000      │
│ AWS io2     │ 1.2ms        │ 3.5ms        │ 35,000      │
└─────────────┴──────────────┴──────────────┴─────────────┘

Conclusion :
- NVMe : Meilleur choix performance pure
- gp3 : Bon compromis cloud (coût/performance)
- SATA SSD : Acceptable dev/test
- HDD : À éviter absolument pour production
```

### RAID Configuration

**RAID recommandés pour MongoDB** :

| RAID | Caractéristiques | Recommandation |
|------|------------------|----------------|
| **RAID 10** | Mirror + Stripe | **Optimal production** |
| | - Read: Excellent (stripe) | - Haute performance |
| | - Write: Bon (mirror) | - Redondance |
| | - Capacity: 50% | - Coût : 2× |
| **RAID 6** | Double parity | Acceptable si budget limité |
| | - Read: Bon | - Moins de performance |
| | - Write: Moyen (parity) | - Meilleure capacité |
| | - Capacity: (n-2)/n | |
| **RAID 0** | Stripe only | ⚠️ Seulement avec réplication |
| | - Read: Excellent | - Aucune redondance |
| | - Write: Excellent | - Perte d'un disque = perte node |
| | - Capacity: 100% | |

**Configuration RAID 10 optimale** :

```bash
# Exemple : 4 disques NVMe
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/nvme[0-3]n1

# Filesystem : XFS (recommandé pour MongoDB)
mkfs.xfs /dev/md0

# Mount options
mount -o noatime,nodiratime /dev/md0 /var/lib/mongodb

# Vérification performance
fio --name=mongodb-test \
    --ioengine=libaio \
    --direct=1 \
    --bs=16k \
    --iodepth=64 \
    --rw=randrw \
    --rwmixread=60 \
    --size=10G \
    --filename=/var/lib/mongodb/test

# Target :
# - IOPS : >50,000
# - Latency : <1ms
```

### Storage Monitoring

```javascript
// Métriques I/O MongoDB
function analyzeStoragePerformance() {
  const serverStatus = db.serverStatus();
  const wt = serverStatus.wiredTiger;

  const metrics = {
    // Block manager stats
    blocksRead: wt.block_manager["blocks read"],
    blocksWritten: wt.block_manager["blocks written"],
    bytesReadGB: (wt.block_manager["bytes read"] / 1024 / 1024 / 1024).toFixed(2),
    bytesWrittenGB: (wt.block_manager["bytes written"] / 1024 / 1024 / 1024).toFixed(2),

    // Checkpoint performance
    avgCheckpointMs: (wt.transaction["transaction checkpoint total time (msecs)"] /
                      wt.transaction["transaction checkpoints"]).toFixed(2),
    maxCheckpointMs: wt.transaction["transaction checkpoint max time (msecs)"],

    // Cache performance (indirect I/O indicator)
    cacheEvictions: wt.cache["pages evicted by application threads"]
  };

  // Assessment
  const issues = [];

  if (metrics.maxCheckpointMs > 30000) {
    issues.push("⚠️ Very slow checkpoints - Storage bottleneck");
  }

  if (metrics.cacheEvictions > 100000) {
    issues.push("⚠️ High eviction rate - Possible I/O pressure");
  }

  metrics.assessment = issues.length === 0 ?
    "✅ Storage performance appears adequate" : issues;

  return metrics;
}

printjson(analyzeStoragePerformance());
```

**Monitoring système** :

```bash
# iostat - Monitoring I/O détaillé
iostat -x 5

# Métriques critiques :
# - %util : >80% = saturation
# - await : >10ms = lent
# - r/s + w/s : IOPS actuel

# Si problème détecté :
# 1. Vérifier si I/O séquentiel vs random
# 2. Corréler avec checkpoint timing
# 3. Analyser queries (explain()) pour optimiser
```

## Dimensionnement du Réseau

### Bandwidth Requirements

**Calcul de bandwidth** :

```
Bandwidth = Data Transfer Rate + Replication Traffic + Cluster Communication

Où :
- Data Transfer : Query results (read) + Documents (write)
- Replication : Oplog sync (primary → secondaries)
- Cluster Comm : Heartbeats, elections, chunk migrations
```

**Exemple de calcul** :

```
Scénario Replica Set :
- Read throughput : 50 MB/s
- Write throughput : 20 MB/s
- Replication factor : 3
- Average document size : 5 KB

Calcul :
Application traffic = 50 + 20 = 70 MB/s
Replication traffic = 20 MB/s × 2 (primary → 2 secondaries) = 40 MB/s
Total = 110 MB/s = 880 Mbps

→ Provisionner : 1 Gbps minimum, 10 Gbps recommandé
```

**Scénario Sharded Cluster** :

```
Cluster : 4 shards, chaque shard = replica set 3 membres

Traffic :
- Application → Mongos : 200 MB/s
- Mongos → Shards (scatter-gather) : 200 MB/s × 4 = 800 MB/s
- Replication per shard : 50 MB/s × 2 = 100 MB/s
- Inter-shard (chunk migration) : 50 MB/s

Total intra-cluster : ~1 GB/s = 8 Gbps

→ Provisionner : 10 Gbps minimum, 25 Gbps recommandé
```

### Latency Requirements

**Network latency impact** :

```
Latency entre membres du replica set :
< 1ms   : Idéal (same datacenter, same rack optimal)
1-5ms   : Acceptable (same datacenter, different racks)
5-50ms  : Tolérable (same region, different AZs)
>50ms   : Problématique (différentes régions)

Impact sur write latency :
Write Latency = Application → Primary + Primary → Secondaries + Ack
              = 1ms + 2ms + 1ms = 4ms (avec writeConcern majority)

Si network latency augmente de 10ms :
Write Latency = 1ms + 12ms + 1ms = 14ms (+350%)
```

**Recommandations par déploiement** :

| Déploiement | Latency Target | Bandwidth | Configuration |
|-------------|----------------|-----------|---------------|
| **Single DC** | <1ms | 1-10 Gbps | Optimal |
| **Multi-AZ** | <5ms | 1-10 Gbps | Recommandé production |
| **Multi-Region (sync)** | <50ms | 1 Gbps+ | Haute disponibilité |
| **Multi-Region (async)** | <100ms | 100 Mbps+ | Disaster recovery |

### Network Monitoring

```javascript
function analyzeReplicationLag() {
  const replStatus = rs.status();

  const primary = replStatus.members.find(m => m.state === 1);
  const secondaries = replStatus.members.filter(m => m.state === 2);

  const analysis = {
    primary: primary.name,
    secondaries: secondaries.map(sec => ({
      name: sec.name,
      lagSeconds: ((primary.optimeDate - sec.optimeDate) / 1000).toFixed(2),
      pingMs: sec.pingMs || 0,
      health: sec.health === 1 ? "OK" : "ISSUE"
    }))
  };

  // Assessment
  const maxLag = Math.max(...analysis.secondaries.map(s => parseFloat(s.lagSeconds)));
  const maxPing = Math.max(...analysis.secondaries.map(s => s.pingMs));

  const issues = [];

  if (maxLag > 10) {
    issues.push("⚠️ High replication lag (>10s) - Check network or load");
  }

  if (maxPing > 50) {
    issues.push("⚠️ High network latency (>50ms) - Check connectivity");
  }

  analysis.assessment = issues.length === 0 ?
    "✅ Replication healthy" : issues;

  return analysis;
}

printjson(analyzeReplicationLag());
```

## Cloud vs On-Premise

### Comparaison Dimensionnement

**On-Premise** :

Avantages :
- Contrôle total du hardware
- Optimisation possible (RAID, réseau)
- Coût fixe à long terme
- Performance maximale (NVMe, réseau dédié)

Inconvénients :
- Capex élevé initial
- Scaling plus lent
- Maintenance hardware
- Disaster recovery complexe

**Dimensionnement type** :
```
Server : Dell PowerEdge R750
- CPU : 2× Intel Xeon Gold 6338 (32 cores @ 2.0 GHz, turbo 3.5)
- RAM : 512 GB DDR4-3200
- Storage : 4× 3.84TB NVMe SSD (RAID 10)
- Network : 2× 25 Gbps

Coût : ~$25,000 initial + $3,000/an maintenance
Performance : Excellent
TCO 3 ans : ~$34,000
```

**Cloud (AWS)** :

Avantages :
- Opex (pay-as-you-go)
- Scaling rapide (minutes)
- Haute disponibilité intégrée
- Disaster recovery simple

Inconvénients :
- Coût cumulatif élevé long terme
- Performance variable (noisy neighbors)
- Vendor lock-in
- Moins d'optimisation possible

**Dimensionnement type** :
```
Instance : r6i.4xlarge (memory-optimized)
- CPU : 16 vCPUs (Intel Ice Lake)
- RAM : 128 GB
- Storage : 2× 1TB gp3 (16,000 IOPS)
- Network : 12.5 Gbps

Coût : ~$1,200/mois ($14,400/an)
TCO 3 ans : ~$43,200

Instance : r6i.8xlarge (pour workload plus lourd)
- CPU : 32 vCPUs
- RAM : 256 GB
- Storage : 4× 1TB gp3
- Network : 25 Gbps

Coût : ~$2,400/mois ($28,800/an)
TCO 3 ans : ~$86,400
```

### Recommandations Cloud par Provider

#### AWS

**Instance families** :

| Family | Use Case | CPU | RAM | Storage | Network |
|--------|----------|-----|-----|---------|---------|
| **r6i** | Memory-optimized | Ice Lake | High | gp3/io2 | Up to 25 Gbps |
| **r7i** | Latest gen | Sapphire Rapids | High | gp3/io2 | Up to 25 Gbps |
| **i4i** | Storage-optimized | Ice Lake | Med | NVMe local | Up to 75 Gbps |
| **c6i** | Compute-optimized | Ice Lake | Med | gp3/io2 | Up to 25 Gbps |

**Configurations recommandées** :

```yaml
# Read-Heavy Production
Instance: r6i.4xlarge
  vCPUs: 16
  RAM: 128 GB
  Storage: 2× 1TB gp3 (16,000 IOPS each)
  Cost: ~$1,200/month

# Write-Heavy / OLTP
Instance: r6i.8xlarge
  vCPUs: 32
  RAM: 256 GB
  Storage: 2× 2TB io2 (32,000 IOPS each)
  Cost: ~$3,000/month

# Analytics / Large Dataset
Instance: r6i.16xlarge
  vCPUs: 64
  RAM: 512 GB
  Storage: 4× 2TB gp3 (16,000 IOPS each)
  Cost: ~$4,800/month
```

#### Azure

**VM Series** :

| Series | Use Case | Specifications |
|--------|----------|----------------|
| **Esv5** | Memory-optimized | Intel Ice Lake, up to 672 GB RAM |
| **Dasv5** | Balanced | AMD EPYC, good price/performance |
| **Lsv3** | Storage-optimized | NVMe local storage |

**Configurations recommandées** :

```yaml
# Standard Production
VM: Standard_E16s_v5
  vCPUs: 16
  RAM: 128 GB
  Storage: 2× Premium SSD (P40, 7,500 IOPS each)
  Cost: ~$1,100/month

# High-Performance
VM: Standard_E32s_v5
  vCPUs: 32
  RAM: 256 GB
  Storage: 2× Ultra SSD (20,000 IOPS each)
  Cost: ~$2,800/month
```

#### GCP

**Machine types** :

| Type | Use Case | Specifications |
|------|----------|----------------|
| **n2-highmem** | Memory-optimized | Intel Cascade Lake |
| **c2** | Compute-optimized | High single-thread performance |
| **n2d** | AMD option | Good price/performance |

**Configurations recommandées** :

```yaml
# Standard Production
Machine: n2-highmem-16
  vCPUs: 16
  RAM: 128 GB
  Storage: 2× 1TB SSD persistent disk (30,000 IOPS)
  Cost: ~$1,000/month

# High-Performance
Machine: n2-highmem-32
  vCPUs: 32
  RAM: 256 GB
  Storage: 2× 2TB SSD persistent disk
  Cost: ~$2,000/month
```

## Capacity Planning

### Projection de Croissance

**Modèle de prévision** :

```javascript
function projectCapacity(currentGB, monthlyGrowthRate, months) {
  const projections = [];

  for (let month = 1; month <= months; month++) {
    const projectedSize = currentGB * Math.pow(1 + monthlyGrowthRate, month);
    projections.push({
      month: month,
      sizeGB: projectedSize.toFixed(2),
      ramRequired: (projectedSize * 0.3 * 1.3 + 3).toFixed(2),  // 30% working set
      storageRequired: (projectedSize * 1.3 * 3).toFixed(2)  // 3× replica set
    });
  }

  return projections;
}

// Exemple : 500GB actuels, 5% de croissance mensuelle, projection 24 mois
const projection = projectCapacity(500, 0.05, 24);
printjson(projection);

// Résultat :
// Mois 12 : 898 GB → RAM : 352 GB, Storage : 3.5 TB
// Mois 24 : 1611 GB → RAM : 631 GB, Storage : 6.3 TB
```

### Scaling Triggers

**Seuils d'action** :

```yaml
# RAM
Warning: Usage > 85%
  Action: Plan scaling dans 30 jours

Critical: Usage > 95%
  Action: Scale immédiatement

# Storage
Warning: Usage > 70%
  Action: Plan expansion dans 60 jours

Critical: Usage > 85%
  Action: Expand dans 7 jours

# CPU
Warning: Usage > 70% sustained
  Action: Optimize queries ou plan scaling

Critical: Usage > 85% sustained
  Action: Scale cores dans 14 jours

# IOPS
Warning: Latency P99 > 10ms
  Action: Review queries et index

Critical: Latency P99 > 50ms
  Action: Upgrade storage tier
```

### Right-Sizing Strategy

**Méthodologie de redimensionnement** :

```
Étape 1 : Analyse utilisation actuelle (30 jours)
- RAM : Average, P95, P99
- CPU : Average, P95, P99
- Storage : Growth rate
- I/O : IOPS, latency

Étape 2 : Identification over/under-provisioning
Over-provisioned si :
- RAM < 50% pendant 30 jours
- CPU < 30% pendant 30 jours
- Storage < 50% avec croissance lente

Under-provisioned si :
- RAM > 90% fréquent
- CPU > 80% fréquent
- Storage > 80%

Étape 3 : Ajustement
Downsizing : Réduire de 30-50%
Upsizing : Augmenter de 50-100%

Étape 4 : Validation (7-14 jours)
Monitorer métriques clés
Rollback si dégradation
```

## Checklist de Dimensionnement

### Phase de Design

```
☐ Analyser le workload (read/write ratio, latency requirements)
☐ Calculer le working set size
☐ Dimensionner RAM (WSS × 1.3 + overhead + 30% marge)
☐ Dimensionner CPU (concurrence + complexité queries)
☐ Dimensionner Storage (capacité + IOPS + croissance)
☐ Dimensionner Network (bandwidth + latency requirements)
☐ Choisir architecture (replica set vs sharded)
☐ Définir stratégie cloud vs on-premise
☐ Calculer TCO sur 3 ans
```

### Phase de Validation

```
☐ POC avec données réelles
☐ Load testing avec profil production
☐ Mesurer latency (P50, P95, P99)
☐ Mesurer throughput (ops/sec)
☐ Vérifier cache hit ratio (target >95%)
☐ Vérifier checkpoint performance (<10s)
☐ Vérifier replication lag (<5s)
☐ Stress test (2-3× charge nominale)
```

### Phase de Production

```
☐ Monitoring continu (métriques de dimensionnement)
☐ Alerting configuré (seuils de saturation)
☐ Capacity planning quarterly
☐ Performance review mensuel
☐ Right-sizing review bi-annuel
☐ Disaster recovery tested
☐ Scaling procedure documented
```

## Cas Pratiques de Dimensionnement

### Cas 1 : Startup E-commerce

**Requirements** :
- Users : 100K actifs
- Transactions : 10K/jour
- Data : 50 GB actuels
- Growth : 10%/mois
- SLA : 99.9% uptime, <100ms P99

**Dimensionnement** :
```yaml
Architecture: Replica Set (3 membres)

Hardware par membre:
  CPU: 8 cores @ 3.0+ GHz
  RAM: 32 GB
    - Working Set: 15 GB (30% de 50GB)
    - WiredTiger Cache: 19 GB
    - Overhead: 3 GB
    - Marge: 10 GB

  Storage: 500 GB NVMe SSD
    - Data: 50 GB
    - Growth (12 months): 50 × 1.1^12 = 157 GB
    - Replication: N/A (chaque membre a tout)
    - Overhead: 1.3×
    - Total: 204 GB × 1.5 marge = 306 GB

  Network: 1 Gbps

Coût estimé (AWS):
  Instance: r6i.2xlarge × 3
  Cost: ~$900/month
```

### Cas 2 : Enterprise SaaS Platform

**Requirements** :
- Users : 1M actifs
- Transactions : 1M/jour
- Data : 2 TB actuels
- Growth : 5%/mois
- SLA : 99.99% uptime, <50ms P99

**Dimensionnement** :
```yaml
Architecture: Sharded Cluster
  - 4 shards (replica set chacun)
  - 3 config servers
  - 2 mongos

Shard (3 membres chacun):
  CPU: 24 cores @ 3.5+ GHz
  RAM: 256 GB
    - Working Set per shard: 150 GB (30% de 500GB)
    - WiredTiger Cache: 195 GB
    - Overhead: 5 GB
    - Marge: 56 GB

  Storage: 2 TB NVMe SSD
    - Data per shard: 500 GB
    - Growth (12 months): 500 × 1.05^12 = 898 GB
    - Overhead: 1.3×
    - Total: 1167 GB × 1.3 marge = 1517 GB

  Network: 10 Gbps

Config Servers (3 membres):
  CPU: 4 cores
  RAM: 16 GB
  Storage: 100 GB SSD
  Network: 1 Gbps

Mongos (2 instances):
  CPU: 8 cores
  RAM: 16 GB
  Network: 10 Gbps

Coût estimé (AWS):
  Shards: r6i.8xlarge × 12 = ~$28,800/month
  Config: r6i.xlarge × 3 = ~$900/month
  Mongos: c6i.2xlarge × 2 = ~$600/month
  Total: ~$30,300/month
```

### Cas 3 : Analytics Platform

**Requirements** :
- Data : 10 TB
- Queries : Complex aggregations
- Users : 500 analysts
- Latency : <5s P99 acceptable
- Batch jobs : Nightly ETL

**Dimensionnement** :
```yaml
Architecture: Dedicated Replica Set

Primary + 2 Secondaries:
  CPU: 48 cores @ 3.0+ GHz
    - Agrégations CPU-intensive
    - Parallélisation importante

  RAM: 512 GB
    - Working Set: 3 TB dataset mais 30% hot = 900 GB
    - Impossible tout en RAM
    - Cache: 480 GB → 50% du hot data
    - Accepter cache misses

  Storage: 8 TB NVMe SSD
    - Data: 10 TB compressé (zlib) → ~4 TB on disk
    - Oplog: 500 GB (large pour batch)
    - Overhead: 1.3×
    - Total: 5.2 TB × 1.3 = 6.76 TB

  Network: 25 Gbps
    - Large result sets
    - Replication de bulk inserts

Hidden Secondary (pour analytics isolés):
  Same specs, hidden: true, priority: 0

Coût estimé (AWS):
  Instance: r6i.16xlarge × 4
  Cost: ~$19,200/month
```

## Conclusion

Le dimensionnement matériel optimal pour MongoDB repose sur :

1. **Analyse approfondie du workload** : Comprendre les patterns réels
2. **Priorité à la RAM** : Working set doit tenir en cache
3. **Storage moderne** : NVMe SSD pour production
4. **CPU scaling** : 16-32 cores optimal, haute fréquence
5. **Network adequate** : 10 Gbps minimum pour clusters

**Règles d'or** :
- RAM > Working Set × 1.3
- Storage = Data × Croissance × Replication × 1.3
- CPU = Concurrence / 75 + 4 overhead
- Network > Throughput × 2 (marge)

**Erreurs courantes** :
- Sous-estimer le working set (cause #1 de performance issues)
- Choisir HDD ou SATA SSD (10-100× plus lent que NVMe)
- Ignorer la croissance (saturation en 6 mois)
- Over-provisionner sans mesure (gaspillage de budget)

**Méthodologie** :
1. Start with calculations (formulas)
2. Validate with POC (real data)
3. Deploy with margin (30%)
4. Monitor continuously (adjust)
5. Right-size quarterly (optimize costs)

Le dimensionnement n'est pas une opération ponctuelle mais un processus continu d'ajustement basé sur les métriques de production et l'évolution du workload.

---

**Points clés à retenir :**
- RAM est la ressource la plus critique : Working Set doit tenir en cache
- NVMe SSD obligatoire pour production (10-100× plus rapide que SATA/HDD)
- CPU : 16-32 cores @ 3.0+ GHz optimal pour la plupart des workloads
- Network : 10 Gbps minimum pour clusters distribués
- Capacity planning : Projeter 12-24 mois avec marge 30%
- Cloud vs On-Premise : TCO à calculer sur 3 ans
- Monitoring continu essentiel pour right-sizing et optimisation coûts

⏭️ [Paramètres de configuration avancés](/17-performance-tuning/08-parametres-configuration-avances.md)
