🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.8 Paramètres de Configuration Avancés

## Introduction

Au-delà du dimensionnement matériel et de l'optimisation des requêtes, les paramètres de configuration système et MongoDB jouent un rôle crucial dans les performances en production. Une configuration optimale peut améliorer les performances de 20-50% et prévenir des problèmes critiques de stabilité et de scalabilité.

Cette section explore les paramètres avancés de MongoDB et du système d'exploitation, leurs impacts sur les performances, et les méthodologies pour les optimiser selon différents profils de charge. Une configuration incorrecte peut dégrader les performances ou causer des instabilités subtiles difficiles à diagnostiquer.

## Architecture de Configuration MongoDB

### Hiérarchie de Configuration

MongoDB utilise plusieurs sources de configuration avec ordre de priorité :

```
1. Command-line options (--parameter)
   ↓ (override)
2. Configuration file (mongod.conf)
   ↓ (override)
3. Runtime parameters (setParameter)
   ↓ (override)
4. Default values (compiled-in)
```

**Persistance** :
- Command-line : Non persistant (redémarrage requis)
- Configuration file : Persistant
- Runtime parameters : Non persistant (sauf si setParameter: { persist: true })

### Structure mongod.conf

```yaml
# /etc/mongod.conf - Structure complète

# Processus et systemd
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

# Réseau
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536
  ipv6: false
  compression:
    compressors: snappy,zstd,zlib

# Sécurité
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile

# Stockage
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100
  directoryPerDB: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 20
      journalCompressor: snappy
      directoryForIndexes: false
    collectionConfig:
      blockCompressor: zstd
    indexConfig:
      prefixCompression: true

# Opérations
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 1.0

# Réplication
replication:
  oplogSizeMB: 51200
  replSetName: rs0
  enableMajorityReadConcern: true

# Sharding
sharding:
  clusterRole: shardsvr

# Logging
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  verbosity: 0
  component:
    accessControl:
      verbosity: 0
    command:
      verbosity: 0

# SetParameter
setParameter:
  enableLocalhostAuthBypass: false
  authenticationMechanisms: SCRAM-SHA-256
```

## Paramètres de Performance Critiques

### Connexions et Concurrence

#### maxIncomingConnections

**Description** : Nombre maximum de connexions simultanées acceptées.

```yaml
net:
  maxIncomingConnections: 65536  # Défaut : 65536 (ou ulimit)
```

**Calcul optimal** :
```
maxIncomingConnections = (Expected Peak Connections × 1.5) + Reserved

Où :
- Expected Peak : Connexions applicatives + admin + monitoring
- 1.5 : Marge pour pics
- Reserved : 50-100 pour admin/monitoring

Exemple :
- Application : 2000 connexions peak
- Monitoring : 20
- Admin : 10
- Calcul : (2000 × 1.5) + 30 = 3030
→ Configurer : 4000
```

**Impact sur les ressources** :
```javascript
// Chaque connexion consomme :
// - Thread : 1 (OS thread)
// - RAM : 1-2 MB (stack + buffers)
// - File descriptors : 1-3

// Exemple avec 4000 connexions :
// - Threads : 4000
// - RAM : 4-8 GB
// - FDs : 4000-12000

// Vérifier limites OS
db.serverStatus().connections
{
  current: 2345,
  available: 1655,
  totalCreated: 45678,
  active: 892,
  threaded: 2345
}
```

**Problèmes courants** :
```javascript
// Connection exhaustion
if (serverStatus.connections.available < 100) {
  print("WARNING: Less than 100 connections available");
  print("Consider:");
  print("1. Increase maxIncomingConnections");
  print("2. Optimize connection pooling in apps");
  print("3. Investigate connection leaks");
}

// Trop de connexions idle
const idleRatio = (serverStatus.connections.current - serverStatus.connections.active)
                  / serverStatus.connections.current;
if (idleRatio > 0.7) {
  print("INFO: >70% connections idle");
  print("Consider reducing pool size in applications");
}
```

#### Connection Pool (Application Side)

```javascript
// Configuration driver (exemple Node.js)
const client = new MongoClient(uri, {
  maxPoolSize: 100,          // Maximum connections dans le pool
  minPoolSize: 10,           // Minimum connections maintenues
  maxIdleTimeMS: 300000,     // 5 minutes idle avant fermeture
  waitQueueTimeoutMS: 10000, // Timeout si pool exhausted
  serverSelectionTimeoutMS: 30000
});

// Calcul optimal poolSize par instance app :
// maxPoolSize = (Expected Concurrent Ops × 1.2) / Number of App Instances

// Exemple :
// - 1000 ops/sec concurrent
// - 10 app instances
// - maxPoolSize = (1000 × 1.2) / 10 = 120 par instance
```

### Ticket System (Concurrency Control)

WiredTiger utilise un système de tickets pour contrôler la concurrence.

**Configuration** :
```javascript
// Paramètres par défaut (MongoDB 4.0+)
db.adminCommand({
  setParameter: 1,
  wiredTigerConcurrentReadTransactions: 128,  // Read tickets
  wiredTigerConcurrentWriteTransactions: 128  // Write tickets
})

// MongoDB 3.x avait des valeurs plus basses (128 par défaut)
// MongoDB 4.0+ : Valeurs dynamiques basées sur CPU cores
```

**Calcul automatique** :
```
MongoDB 4.0+ :
Read tickets = 128 (fixe)
Write tickets = 128 (fixe)

MongoDB 5.0+ considère les cores mais garde 128 comme optimal
```

**Monitoring des tickets** :
```javascript
function analyzeTicketUtilization() {
  const serverStatus = db.serverStatus();
  const wt = serverStatus.wiredTiger;

  const analysis = {
    readTickets: {
      available: wt.concurrentTransactions.read.available,
      out: wt.concurrentTransactions.read.out,
      totalTickets: wt.concurrentTransactions.read.totalTickets,
      utilization: ((wt.concurrentTransactions.read.out /
                     wt.concurrentTransactions.read.totalTickets) * 100).toFixed(2) + "%"
    },

    writeTickets: {
      available: wt.concurrentTransactions.write.available,
      out: wt.concurrentTransactions.write.out,
      totalTickets: wt.concurrentTransactions.write.totalTickets,
      utilization: ((wt.concurrentTransactions.write.out /
                     wt.concurrentTransactions.write.totalTickets) * 100).toFixed(2) + "%"
    }
  };

  // Assessment
  const issues = [];

  if (wt.concurrentTransactions.read.available < 10) {
    issues.push("⚠️ Low read tickets available - potential queueing");
  }

  if (wt.concurrentTransactions.write.available < 10) {
    issues.push("⚠️ Low write tickets available - potential queueing");
  }

  analysis.assessment = issues.length === 0 ?
    "✅ Ticket system healthy" : issues;

  return analysis;
}

printjson(analyzeTicketUtilization());
```

**Tuning tickets** (Rare) :
```javascript
// Augmenter seulement si :
// 1. Tickets constamment exhausted (available = 0)
// 2. Queue depth élevé persistant
// 3. Serveur sous-utilisé (CPU < 50%)

// Augmentation progressive
db.adminCommand({
  setParameter: 1,
  wiredTigerConcurrentReadTransactions: 256,  // Doubled
  wiredTigerConcurrentWriteTransactions: 256
})

// Monitoring après changement :
// - CPU usage (devrait augmenter)
// - Latency (devrait baisser si CPU available)
// - Throughput (devrait augmenter)

// ATTENTION : Trop de tickets = CPU thrashing
// Ne jamais dépasser 512 sans testing approfondi
```

### Journal et Durabilité

#### commitIntervalMs

**Impact** : Fréquence de flush du journal sur disque.

```yaml
storage:
  journal:
    commitIntervalMs: 100  # 50-500ms
```

**Trade-offs par valeur** :

| Valeur | Durabilité | Write Latency | Throughput | Use Case |
|--------|------------|---------------|------------|----------|
| 50ms | Excellent | +20% | -15% | Financial, critical data |
| 100ms | Bon | Baseline | Baseline | **Défaut recommandé** |
| 200ms | Acceptable | -15% | +20% | Write-heavy, tolère perte 200ms |
| 500ms | Risqué | -30% | +40% | Non recommandé production |

**Combinaison avec Write Concern** :
```javascript
// j:true force flush immédiat (ignore commitIntervalMs)
db.collection.insertOne(
  { data: "critical" },
  { writeConcern: { w: "majority", j: true } }
)
// Latency : +5-10ms mais garantie durabilité maximale

// j:false (default) respecte commitIntervalMs
db.collection.insertOne(
  { data: "non-critical" },
  { writeConcern: { w: 1, j: false } }
)
// Latency minimale, perte possible en cas de crash
```

#### directoryPerDB et directoryForIndexes

**directoryPerDB** :
```yaml
storage:
  directoryPerDB: true  # Une directory par database
```

Avantages :
- Organisation claire
- Backup par database facilité
- Peut utiliser différents filesystems par database

Inconvénients :
- Plus de file descriptors
- Fragmentation filesystem possible

**directoryForIndexes** :
```yaml
storage:
  wiredTiger:
    engineConfig:
      directoryForIndexes: true  # Sépare index des données
```

Avantages :
- Peut placer index sur storage plus rapide
- Meilleure organisation
- Debug facilité

Inconvénients :
- Complexité accrue
- Nécessite configuration filesystem

**Configuration optimale pour I/O** :
```yaml
# Scénario : NVMe pour index, SATA SSD pour data
storage:
  dbPath: /data/mongodb  # SATA SSD
  directoryPerDB: true
  wiredTiger:
    engineConfig:
      directoryForIndexes: true

# Puis créer symlink des index vers NVMe
# ln -s /nvme/mongodb/mydb/index /data/mongodb/mydb/index
```

### Oplog Configuration

#### oplogSizeMB

**Calcul optimal** :
```javascript
// Formule
oplogSizeMB = Write Rate (MB/h) × Window (hours) × Safety Factor

// Exemple
const writeRateMBPerHour = 2048;  // 2 GB/h
const windowHours = 48;           // 48h window target
const safetyFactor = 1.5;

const oplogSize = writeRateMBPerHour × windowHours × safetyFactor;
// = 2048 × 48 × 1.5 = 147,456 MB = 144 GB

// Configuration
replication:
  oplogSizeMB: 147456
```

**Validation** :
```javascript
function analyzeOplogCapacity() {
  const oplogs = db.getSiblingDB("local").oplog.rs;
  const oplogStats = oplogs.stats();

  // Window actuelle
  const firstEntry = oplogs.find().sort({$natural: 1}).limit(1).next();
  const lastEntry = oplogs.find().sort({$natural: -1}).limit(1).next();
  const windowHours = (lastEntry.ts.getTime() - firstEntry.ts.getTime()) / 3600000;

  // Taux de remplissage
  const oplogSizeGB = oplogStats.maxSize / 1024 / 1024 / 1024;
  const oplogUsedGB = oplogStats.size / 1024 / 1024 / 1024;
  const fillRate = (oplogUsedGB / oplogSizeGB * 100).toFixed(2);

  // Write rate
  const writeMBPerHour = (oplogUsedGB × 1024) / windowHours;

  const analysis = {
    oplogSizeGB: oplogSizeGB.toFixed(2),
    currentWindowHours: windowHours.toFixed(2),
    fillRate: fillRate + "%",
    writeMBPerHour: writeMBPerHour.toFixed(2),

    // Projections
    timeToFull: ((oplogSizeGB - oplogUsedGB) / (writeMBPerHour / 1024)).toFixed(2) + " hours",

    recommendation: ""
  };

  if (windowHours < 24) {
    analysis.recommendation = "⚠️ Increase oplog - window < 24h";
  } else if (windowHours < 48) {
    analysis.recommendation = "Consider increasing oplog to 48h+ window";
  } else {
    analysis.recommendation = "✅ Oplog size adequate";
  }

  return analysis;
}

printjson(analyzeOplogCapacity());
```

**Redimensionnement de l'oplog** :
```javascript
// MongoDB 4.0+ : Redimensionnement online
db.adminCommand({
  replSetResizeOplog: 1,
  size: 147456  // Nouvelle taille en MB
})

// Pré-4.0 : Nécessite shutdown et rebuild
// 1. Backup
// 2. Shutdown secondary
// 3. Démarrer en standalone
// 4. Recréer oplog avec nouvelle taille
// 5. Redémarrer en replica set
// 6. Répéter pour chaque membre
```

## Paramètres de Profiling et Monitoring

### Operation Profiling

```yaml
operationProfiling:
  mode: slowOp           # off | slowOp | all
  slowOpThresholdMs: 100 # Seuil en ms
  slowOpSampleRate: 1.0  # 0.0-1.0 (100% = tout profiler)
  filter: '{ op: { $in: ["query", "update", "remove"] } }'
```

**Configuration par environnement** :

```yaml
# Production
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 0.1   # 10% sampling pour réduire overhead

# Staging / Debug
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 50
  slowOpSampleRate: 1.0   # 100% pour debugging

# Performance testing
operationProfiling:
  mode: all               # Tous les ops
  slowOpThresholdMs: 0
  slowOpSampleRate: 1.0
```

**Runtime adjustment** :
```javascript
// Activer temporairement profiling complet
db.setProfilingLevel(2, { slowms: 0, sampleRate: 1.0 })

// Après 5 minutes, revenir à slowOp
db.setProfilingLevel(1, { slowms: 100, sampleRate: 0.1 })

// Analyse des ops
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()
```

### Logging Verbosity

```yaml
systemLog:
  verbosity: 0  # 0-5, 0 = minimal
  component:
    accessControl:
      verbosity: 1    # Auth logs
    command:
      verbosity: 0
    control:
      verbosity: 0
    ftdc:
      verbosity: 0
    geo:
      verbosity: 0
    index:
      verbosity: 0
    network:
      verbosity: 0
    query:
      verbosity: 1    # Query logs pour debugging
    replication:
      verbosity: 0
    sharding:
      verbosity: 0
    storage:
      verbosity: 0
    write:
      verbosity: 0
    transaction:
      verbosity: 0
```

**Debugging specific components** :
```javascript
// Augmenter verbosity temporairement
db.adminCommand({
  setParameter: 1,
  logComponentVerbosity: {
    query: { verbosity: 2 },
    replication: { verbosity: 1 }
  }
})

// Après investigation
db.adminCommand({
  setParameter: 1,
  logComponentVerbosity: {
    query: { verbosity: 0 },
    replication: { verbosity: 0 }
  }
})
```

**Log rotation** :
```yaml
systemLog:
  logRotate: reopen  # rename | reopen

# reopen : Compatible avec logrotate
# rename : MongoDB gère la rotation (limite à 10 fichiers)
```

Configuration logrotate :
```bash
# /etc/logrotate.d/mongodb
/var/log/mongodb/*.log {
  daily
  rotate 7
  compress
  delaycompress
  notifempty
  create 0640 mongodb mongodb
  sharedscripts
  postrotate
    /bin/kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid)
  endscript
}
```

## Paramètres Runtime (setParameter)

### Paramètres Persistants

MongoDB 4.2+ supporte setParameter persistant :

```javascript
// Persistent (survit au redémarrage)
db.adminCommand({
  setParameter: 1,
  internalQueryExecMaxBlockingSortBytes: 335544320,  // 320 MB
  persist: true
})

// Vérifier la persistance
db.adminCommand({ getParameter: "*" })
```

### Paramètres de Query Execution

#### internalQueryExecMaxBlockingSortBytes

**Description** : Mémoire maximale pour in-memory sorts.

```javascript
// Défaut : 33554432 (32 MB)
// Recommandé pour analytics : 100-500 MB

db.adminCommand({
  setParameter: 1,
  internalQueryExecMaxBlockingSortBytes: 335544320,  // 320 MB
  persist: true
})

// Impact :
// - Sorts plus volumineux possibles sans allowDiskUse
// - Mais risque OOM si trop élevé et nombreux sorts concurrents

// Calcul optimal :
// maxSortMemory = Available RAM / Expected Concurrent Sorts / 2

// Exemple :
// - RAM disponible : 64 GB
// - Sorts concurrents : 50
// - maxSortMemory = (64 × 1024) / 50 / 2 = 655 MB
```

#### internalQueryPlannerMaxIndexedSolutions

**Description** : Nombre max de plans d'index considérés par le query planner.

```javascript
// Défaut : 64
// Augmenter si beaucoup d'index et queries complexes

db.adminCommand({
  setParameter: 1,
  internalQueryPlannerMaxIndexedSolutions: 128
})

// Impact :
// + : Meilleur plan trouvé si nombreux index
// - : Planning time augmente
```

#### internalQueryExecYieldIterations

**Description** : Nombre d'itérations avant yielding.

```javascript
// Défaut : 128
// Yielding permet à d'autres opérations de s'exécuter

db.adminCommand({
  setParameter: 1,
  internalQueryExecYieldIterations: 64  // Plus de yielding
})

// Augmenter (256, 512) si :
// - Queries rapides dominant
// - Contention faible

// Diminuer (32, 64) si :
// - Queries longues monopolisent
// - Contention élevée
```

### Paramètres de Réplication

#### replWriterThreadCount

**Description** : Nombre de threads pour appliquer l'oplog sur secondaries.

```javascript
// Défaut : 16 (ajusté automatiquement)
// Range : 1-256

db.adminCommand({
  setParameter: 1,
  replWriterThreadCount: 32
})

// Calcul :
// replWriterThreadCount = min(Cores / 2, 32)

// Exemple : 48 cores
// = min(24, 32) = 24

// Augmenter si :
// - Replication lag sur secondaries
// - Write heavy workload
// - Secondaries ont CPU disponible

// Ne pas augmenter excessivement :
// - Overhead de coordination
// - Rendements décroissants au-delà de 32
```

#### replBatchLimitBytes

**Description** : Taille max d'un batch d'oplog.

```javascript
// Défaut : 100 MB

db.adminCommand({
  setParameter: 1,
  replBatchLimitBytes: 209715200  // 200 MB
})

// Augmenter pour :
// - Réduire le nombre de batches
// - Améliorer throughput si bandwidth élevé

// Attention :
// - Plus de mémoire utilisée
// - Latency accrue si batch trop gros
```

### Paramètres de Sharding

#### chunkSize

**Description** : Taille des chunks en MB.

```javascript
// Défaut : 64 MB depuis MongoDB 3.4
// Range : 1-1024 MB

// Sur mongos ou config server
db.getSiblingDB("config").settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 128 } },
  { upsert: true }
)

// Calcul optimal :
// chunkSize = Dataset per Shard / Target Chunks per Shard

// Exemple :
// - 1 TB par shard
// - Target : 10,000 chunks
// - chunkSize = 1024 GB / 10000 = 102 MB → 128 MB

// Augmenter (128-256 MB) si :
// - Large documents
// - Migration overhead trop élevé

// Diminuer (32-64 MB) si :
// - Small documents
// - Meilleure distribution nécessaire
// - Jumbo chunks problème
```

#### mongos Query Parameters

```javascript
// Sur mongos
db.adminCommand({
  setParameter: 1,

  // Timeout pour queries scatter-gather
  cursorTimeoutMillis: 600000,  // 10 minutes

  // Taille max du batch retourné
  internalQueryExecYieldPeriodMS: 10,

  // Connection pool vers shards
  ShardingTaskExecutorPoolMaxSize: 500
})
```

### Paramètres de Sécurité

#### maxSessionsPerUser

**Description** : Sessions max par user (MongoDB 3.6+).

```javascript
db.adminCommand({
  setParameter: 1,
  maxSessions: 1000000,        // Global max sessions
  maxSessionsPerUser: 10000    // Par user
})

// Ajuster selon :
// - Nombre d'utilisateurs
// - Pattern d'utilisation (web vs batch)
```

## Configuration Système d'Exploitation

### Filesystem

**XFS recommandé** (vs ext4) :

```bash
# Création filesystem XFS
mkfs.xfs -f /dev/sdb

# Mount options optimales
mount -o noatime,nodiratime,nobarrier /dev/sdb /var/lib/mongodb

# /etc/fstab
/dev/sdb /var/lib/mongodb xfs noatime,nodiratime,nobarrier 0 0

# noatime : Ne pas update access time (réduit writes)
# nodiratime : Ne pas update directory access time
# nobarrier : Désactive les barrières (si UPS ou RAID avec BBU)
```

**Filesystem performance verification** :
```bash
# Test I/O
fio --name=mongodb-io-test \
    --filename=/var/lib/mongodb/test \
    --size=10G \
    --direct=1 \
    --rw=randrw \
    --rwmixread=60 \
    --bs=16k \
    --ioengine=libaio \
    --iodepth=64 \
    --runtime=60 \
    --numjobs=4 \
    --group_reporting

# Targets :
# - IOPS : >10,000 (SSD), >50,000 (NVMe)
# - Latency avg : <10ms (SSD), <1ms (NVMe)
```

### Kernel Parameters (sysctl)

```bash
# /etc/sysctl.d/mongodb.conf

# Network tuning
net.core.somaxconn = 4096                    # Socket listen backlog
net.ipv4.tcp_fin_timeout = 30                # TIME_WAIT timeout
net.ipv4.tcp_keepalive_time = 120            # Keepalive interval
net.ipv4.tcp_max_syn_backlog = 4096          # SYN backlog
net.ipv4.tcp_syncookies = 1                  # SYN flood protection

# Memory management
vm.swappiness = 1                            # Minimize swapping
vm.dirty_ratio = 15                          # Dirty pages threshold
vm.dirty_background_ratio = 5                # Background flush threshold
vm.zone_reclaim_mode = 0                     # NUMA memory reclaim

# File descriptors
fs.file-max = 98000                          # Global max FDs

# Apply
sysctl -p /etc/sysctl.d/mongodb.conf
```

**Explication des paramètres** :

```yaml
vm.swappiness = 1:
  Description: Tendance du kernel à swapper
  Impact: 0 = jamais swap sauf OOM (risqué)
         1 = swap minimal (recommandé)
         60 = défaut Linux (trop agressif)

vm.dirty_ratio = 15:
  Description: % RAM de dirty pages avant flush synchrone
  Impact: Plus bas = flush plus fréquent, moins de cache
         Plus haut = risque de flush brusque
  Recommandé: 10-15% pour databases

net.core.somaxconn = 4096:
  Description: Queue de connexions en attente
  Impact: Doit être >= maxIncomingConnections
         Trop bas = connexions rejetées sous charge
```

### Ulimits

```bash
# /etc/security/limits.d/mongodb.conf

mongodb soft nofile 64000    # File descriptors (soft limit)
mongodb hard nofile 64000    # File descriptors (hard limit)
mongodb soft nproc 64000     # Processes/threads
mongodb hard nproc 64000
mongodb soft memlock unlimited  # Memory locking
mongodb hard memlock unlimited
mongodb soft fsize unlimited    # File size
mongodb hard fsize unlimited

# Vérification
sudo -u mongodb bash -c 'ulimit -a'
```

**Calcul nofile (file descriptors)** :
```
FD required = Connections + Data files + Index files + Journal + System overhead

Example :
- Connections : 5000
- Data files : 500
- Index files : 1000
- Journal : 10
- System : 500
Total = 7010

Recommandé : 64000 (largement suffisant pour la plupart des cas)
```

**Validation des limites** :
```javascript
// Dans MongoDB
db.serverStatus().connections
// Si current approche de available, vérifier ulimits

// Vérifier les FDs utilisés
db.adminCommand({ serverStatus: 1 }).connections

// Check OS level
// lsof -u mongodb | wc -l
// Should be well below ulimit nofile
```

### Transparent Huge Pages (THP)

**Désactivation obligatoire** :

```bash
# Check status
cat /sys/kernel/mm/transparent_hugepage/enabled
# [always] madvise never  → MAUVAIS (enabled)
# always madvise [never]  → BON (disabled)

# Désactivation temporaire
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Désactivation permanente : systemd unit
# /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=mongod.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target

# Activer
sudo systemctl daemon-reload
sudo systemctl enable disable-thp.service
sudo systemctl start disable-thp.service
```

**Pourquoi désactiver THP** :
```
THP (2MB pages) vs Regular pages (4KB)

Problèmes avec THP pour MongoDB :
1. Compaction stalls : Kernel tente de créer huge pages
   → Freeze applicatif de plusieurs secondes

2. Memory fragmentation : Difficile de maintenir huge pages
   → Performance imprévisible

3. WiredTiger incompatibilité : Utilise fine-grained memory
   → Overhead de gestion THP > bénéfices

Impact mesuré :
- Avec THP : Latency spikes jusqu'à 10+ secondes
- Sans THP : Latency stable et prévisible

Conclusion : TOUJOURS désactiver THP pour MongoDB
```

### NUMA (Non-Uniform Memory Access)

**Configuration pour serveurs multi-socket** :

```bash
# Check NUMA topology
numactl --hardware

# Option 1 : Désactiver NUMA (simple mais pas optimal)
# /etc/default/grub
GRUB_CMDLINE_LINUX="numa=off"
# update-grub && reboot

# Option 2 : Interleave NUMA (meilleur)
# /etc/systemd/system/mongod.service.d/numa.conf
[Service]
ExecStart=
ExecStart=/usr/bin/numactl --interleave=all /usr/bin/mongod --config /etc/mongod.conf

# Reload systemd
systemctl daemon-reload
systemctl restart mongod

# Vérification
ps aux | grep mongod
# Should show : numactl --interleave=all mongod
```

**Impact NUMA** :
```
Sans optimisation NUMA :
- Memory access latency variable (local vs remote)
- Performance dégradation 20-40%
- Comportement imprévisible

Avec interleaving :
- Memory distribué uniformément
- Latency prévisible
- Performance stable

Note : Certains benchmarks montrent que numa=off peut être
légèrement meilleur que interleave pour MongoDB, mais
interleave est généralement plus safe.
```

## Configurations par Environnement

### Production (High Availability)

```yaml
# mongod.conf - Production HA

# Network
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 10000
  compression:
    compressors: snappy,zstd

# Security
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile

# Storage
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100
  directoryPerDB: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 40
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: zstd
    indexConfig:
      prefixCompression: true

# Replication
replication:
  oplogSizeMB: 102400  # 100 GB
  replSetName: prod-rs0
  enableMajorityReadConcern: true

# Operations
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 0.05  # 5% sampling

# Logging
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  verbosity: 0

# setParameter
setParameter:
  enableLocalhostAuthBypass: false
  cursorTimeoutMillis: 600000
  internalQueryExecMaxBlockingSortBytes: 104857600  # 100 MB
```

### Development / Testing

```yaml
# mongod.conf - Development

net:
  port: 27017
  bindIp: 127.0.0.1
  maxIncomingConnections: 1000

security:
  authorization: disabled  # Faciliter dev

storage:
  dbPath: /data/mongodb-dev
  journal:
    enabled: true
    commitIntervalMs: 100
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2
    collectionConfig:
      blockCompressor: snappy

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 50  # Plus strict pour identifier problèmes
  slowOpSampleRate: 1.0   # 100% pour debugging

systemLog:
  destination: file
  path: /var/log/mongodb/mongod-dev.log
  verbosity: 1  # Plus verbose
  component:
    query:
      verbosity: 2  # Debug queries
```

### Analytics / Reporting

```yaml
# mongod.conf - Analytics

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 2000

storage:
  dbPath: /data/mongodb-analytics
  journal:
    enabled: true
    commitIntervalMs: 200  # Moins critique
  wiredTiger:
    engineConfig:
      cacheSizeGB: 120  # Maximiser pour datasets
    collectionConfig:
      blockCompressor: zlib  # Haute compression

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 5000  # Queries longues normales

setParameter:
  internalQueryExecMaxBlockingSortBytes: 524288000  # 500 MB
  internalQueryExecYieldIterations: 256  # Plus de yielding
```

## Validation et Testing de Configuration

### Health Check Automatisé

```javascript
function comprehensiveConfigAudit() {
  const serverStatus = db.serverStatus();
  const buildInfo = db.adminCommand({ buildInfo: 1 });
  const params = db.adminCommand({ getParameter: "*" });

  const audit = {
    timestamp: new Date(),
    version: buildInfo.version,

    // Check critiques
    checks: {
      thp: {
        description: "Transparent Huge Pages",
        // Note : Doit être vérifié via OS, pas disponible dans MongoDB
        status: "MANUAL CHECK REQUIRED",
        command: "cat /sys/kernel/mm/transparent_hugepage/enabled"
      },

      ulimits: {
        description: "File descriptors",
        current: serverStatus.connections.current,
        available: serverStatus.connections.available,
        total: serverStatus.connections.current + serverStatus.connections.available,
        status: serverStatus.connections.available > 1000 ? "OK" : "WARNING"
      },

      numa: {
        description: "NUMA interleaving",
        status: "MANUAL CHECK REQUIRED",
        command: "ps aux | grep -E 'numactl.*mongod'"
      },

      cache: {
        description: "WiredTiger cache size",
        configuredGB: (serverStatus.wiredTiger.cache["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2),
        usagePercent: ((serverStatus.wiredTiger.cache["bytes currently in the cache"] /
                        serverStatus.wiredTiger.cache["maximum bytes configured"]) * 100).toFixed(2),
        status: "OK"
      },

      journal: {
        description: "Journal enabled and configured",
        enabled: params.storage ? params.storage.journal.enabled : "CHECK mongod.conf",
        commitIntervalMs: params.storage ? params.storage.journal.commitIntervalMs : "CHECK mongod.conf",
        status: "OK"
      },

      profiling: {
        description: "Operation profiling",
        level: db.getProfilingLevel(),
        slowms: db.getProfilingStatus().slowms,
        status: db.getProfilingLevel() > 0 ? "ENABLED" : "DISABLED"
      },

      connections: {
        description: "Connection configuration",
        maxIncoming: "CHECK mongod.conf",  // Not available via serverStatus
        current: serverStatus.connections.current,
        available: serverStatus.connections.available,
        status: serverStatus.connections.available < 100 ? "WARNING - Low available" : "OK"
      },

      oplog: {
        description: "Oplog sizing",
        sizeGB: "CHECK via rs.printReplicationInfo()",
        windowHours: "CHECK via rs.printReplicationInfo()",
        status: "MANUAL CHECK REQUIRED"
      }
    },

    // Recommandations
    recommendations: []
  };

  // Generate recommendations
  if (audit.checks.cache.usagePercent > 95) {
    audit.recommendations.push("⚠️ Cache usage >95% - Consider increasing cache size");
  }

  if (audit.checks.ulimits.available < 1000) {
    audit.recommendations.push("⚠️ Low available connections - Check maxIncomingConnections and ulimits");
  }

  if (audit.checks.profiling.level === 0) {
    audit.recommendations.push("ℹ️ Profiling disabled - Consider enabling slowOp profiling");
  }

  if (audit.recommendations.length === 0) {
    audit.recommendations.push("✅ Configuration appears healthy");
  }

  return audit;
}

printjson(comprehensiveConfigAudit());
```

### Performance Regression Testing

```javascript
// Framework de test de configuration
class ConfigurationTest {
  constructor(testName) {
    this.testName = testName;
    this.results = [];
  }

  async runLoadTest(operations = 10000) {
    const start = Date.now();

    // Mixed workload
    for (let i = 0; i < operations; i++) {
      if (i % 10 === 0) {
        // 10% writes
        await db.testCollection.insertOne({
          _id: i,
          data: "test" + i,
          timestamp: new Date()
        });
      } else {
        // 90% reads
        await db.testCollection.findOne({ _id: Math.floor(Math.random() * i) });
      }
    }

    const duration = Date.now() - start;
    const opsPerSecond = (operations / duration * 1000).toFixed(2);

    return {
      operations: operations,
      durationMs: duration,
      opsPerSecond: opsPerSecond
    };
  }

  async compareConfigurations(config1, config2) {
    print(`Testing configuration: ${config1.name}`);
    // Apply config1
    // Run test
    const result1 = await this.runLoadTest();

    print(`Testing configuration: ${config2.name}`);
    // Apply config2
    // Run test
    const result2 = await this.runLoadTest();

    // Compare
    const improvement = ((result2.opsPerSecond - result1.opsPerSecond) /
                         result1.opsPerSecond * 100).toFixed(2);

    return {
      config1: config1.name,
      result1: result1,
      config2: config2.name,
      result2: result2,
      improvement: improvement + "%"
    };
  }
}
```

## Checklist de Configuration Production

### Pre-Deployment

```
☐ OS Configuration
  ☐ Disable THP (verify: cat /sys/kernel/mm/transparent_hugepage/enabled)
  ☐ Configure NUMA (numactl --interleave=all ou numa=off)
  ☐ Set ulimits (nofile: 64000, nproc: 64000)
  ☐ Sysctl tuning (swappiness=1, dirty_ratio=15)
  ☐ XFS filesystem with noatime,nodiratime

☐ MongoDB Configuration
  ☐ WiredTiger cache sized appropriately (working set × 1.3)
  ☐ Journal enabled with appropriate commitIntervalMs
  ☐ Oplog sized for 48h+ window
  ☐ maxIncomingConnections appropriate
  ☐ directoryPerDB enabled for organization
  ☐ Compression configured (zstd recommended)

☐ Security
  ☐ Authorization enabled
  ☐ KeyFile configured (replica sets)
  ☐ TLS/SSL configured
  ☐ Network encryption enabled
  ☐ Audit logging configured

☐ Monitoring
  ☐ Profiling configured (slowOp with sampling)
  ☐ Log rotation configured
  ☐ Metrics collection (Prometheus/Datadog)
  ☐ Alerting rules defined

☐ Replication
  ☐ Replica set configured with odd members
  ☐ Write concern defaults set
  ☐ Read preference strategy defined
  ☐ Priority and votes configured
```

### Post-Deployment Validation

```
☐ Health checks
  ☐ THP disabled verification
  ☐ NUMA configuration active
  ☐ Ulimits effective
  ☐ Disk I/O performance (fio test)

☐ MongoDB status
  ☐ Replica set status healthy
  ☐ Replication lag < 5s
  ☐ Cache hit ratio > 95%
  ☐ No connection exhaustion
  ☐ Checkpoint duration < 10s

☐ Load testing
  ☐ Baseline performance established
  ☐ P99 latency < target
  ☐ Throughput meets requirements
  ☐ No errors under load

☐ Monitoring active
  ☐ Metrics flowing to monitoring system
  ☐ Alerts firing correctly
  ☐ Dashboards populated
  ☐ On-call procedures documented
```

## Conclusion

La configuration optimale de MongoDB nécessite une approche holistique combinant :

1. **Configuration MongoDB** : Paramètres internes adaptés au workload
2. **Configuration OS** : Kernel tuning, filesystem, ulimits
3. **Validation** : Testing et monitoring continus
4. **Ajustement** : Itération basée sur métriques production

**Paramètres les plus impactants** (par ordre d'importance) :
1. THP disabled (impact : 10-50% performance)
2. WiredTiger cache size (impact : 2-10× sur latence)
3. NUMA configuration (impact : 20-40% sur serveurs multi-socket)
4. ulimits appropriate (impact : Stabilité)
5. Filesystem et mount options (impact : 10-20% I/O)
6. Journal commitIntervalMs (impact : 15-30% write latency)
7. Connection limits (impact : Scalabilité)

**Erreurs courantes** :
- THP enabled (cause #1 de latency spikes)
- Cache undersized (thrashing)
- ulimits trop bas (connection failures)
- NUMA mal configuré (performance imprévisible)
- commitIntervalMs trop élevé (perte de données)
- Profiling en mode "all" en production (overhead 20%+)

**Bonnes pratiques** :
- Documenter toute configuration non-standard
- Tester changements en staging d'abord
- Monitorer l'impact de chaque changement
- Utiliser configuration management (Ansible, Chef, Puppet)
- Maintenir des configurations de référence par environnement
- Auditer configuration régulièrement (monthly)

La configuration optimale évolue avec le workload et la croissance. Un audit périodique et des ajustements basés sur les métriques de production sont essentiels pour maintenir des performances optimales.

---

**Points clés à retenir :**
- THP disabled est OBLIGATOIRE (impact critique sur latence)
- WiredTiger cache = working set × 1.3 minimum
- ulimits : nofile 64000, nproc 64000
- NUMA : interleave ou désactivé pour serveurs multi-socket
- XFS avec noatime recommandé
- commitIntervalMs : 100ms optimal pour la plupart des cas
- Profiling : slowOp avec sampling (0.05-0.1) en production
- Configuration doit être versionnée et testée avant déploiement
- Monitoring des paramètres aussi important que configuration initiale

⏭️ [Compression des données](/17-performance-tuning/09-compression-donnees.md)
