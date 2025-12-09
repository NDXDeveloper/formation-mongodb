🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.4 - Audit d'Infrastructure

## Introduction

L'audit d'infrastructure évalue les **ressources matérielles, la configuration système et l'architecture** de votre déploiement MongoDB. Une infrastructure bien dimensionnée et configurée est essentielle pour des performances optimales et une haute disponibilité.

### 🎯 Objectif

Vérifier que l'infrastructure supporte efficacement la charge actuelle et future, et qu'elle est configurée selon les meilleures pratiques MongoDB.

### ⏱️ Durée estimée
- Audit rapide : 45 minutes - 1 heure
- Audit complet : 3-5 heures

---

## Ressources Système

### 💻 CPU (Processeur)

#### ✅ Checklist CPU

| Point de vérification | Valeur Cible | Priorité | Action |
|----------------------|--------------|----------|--------|
| Utilisation moyenne | < 70% | 🟠 | Dimensionnement |
| Utilisation en pointe | < 85% | 🟠 | Planifier scaling |
| I/O Wait | < 10% | 🔴 | Problème disque |
| Steal (VM) | < 5% | 🟠 | Ressources partagées |
| Nombre de cœurs | ≥ 4 (prod) | 🟡 | Performance |

**Commandes de vérification** :
```bash
# Utilisation CPU en temps réel
top -b -n 1 | head -20

# Vue détaillée
htop

# Utilisation moyenne sur 1 minute
uptime

# Statistiques CPU détaillées
mpstat 1 5

# I/O Wait
iostat -x 1 5

# Pour VM : vérifier steal time
top
# Regarder la ligne "%Cpu(s)" -> "st" (steal time)
```

**Analyse** :
```bash
# Script de monitoring CPU
#!/bin/bash
echo "=== Audit CPU ==="
echo "Load Average:"
uptime | awk -F'load average:' '{print $2}'

echo -e "\nCPU Usage:"
top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print "CPU Usage: " 100 - $1"%"}'

echo -e "\nI/O Wait:"
iostat -c | awk 'NR==4 {print "I/O Wait: "$4"%"}'

echo -e "\nCores:"
nproc
```

**Symptômes de problème** :
```markdown
⚠️ CPU > 80% constant → Sous-dimensionnement
⚠️ I/O Wait > 20% → Disque trop lent
⚠️ Load average > 2x nombre de cœurs → Surcharge
⚠️ Steal time > 10% (VM) → Contention ressources host
```

**Actions correctives** :
- Scale vertical (plus de CPU)
- Scale horizontal (sharding)
- Optimiser les requêtes lentes
- Vérifier les process non-MongoDB

---

### 💾 RAM (Mémoire)

#### ✅ Checklist RAM

| Point de vérification | Valeur Cible | Priorité | Action |
|----------------------|--------------|----------|--------|
| RAM totale | ≥ 8 GB (prod) | 🔴 | Minimum absolu |
| Utilisation RAM | < 80% | 🟠 | Éviter swap |
| Working Set < RAM | Oui | 🔴 | Performance critique |
| Index size < RAM | Oui | 🔴 | Performance critique |
| Swap utilisé | 0 ou minimal | 🔴 | Performance |
| Cache WiredTiger | 50% RAM ou config | 🟡 | Optimisation |

**Commandes de vérification** :
```bash
# RAM totale et disponible
free -h

# Détail de l'utilisation
cat /proc/meminfo | grep -E 'MemTotal|MemFree|MemAvailable|Cached|Buffers|SwapTotal|SwapFree'

# Swap actif
swapon -s

# Monitoring continu
vmstat 1 5
```

**Vérifications MongoDB** :
```javascript
// Statistiques mémoire MongoDB
db.serverStatus().mem

// Résultat :
{
  bits: 64,
  resident: 2048,     // RAM utilisée par MongoDB (Mo)
  virtual: 4096,      // Mémoire virtuelle
  supported: true
}

// Working Set
db.serverStatus().wiredTiger.cache

// Taille des index
db.collection.stats().indexSizes

// Total index de la DB
let totalIndexSize = 0;
db.getCollectionNames().forEach(coll => {
  totalIndexSize += db[coll].stats().totalIndexSize;
});
print("Total index size: " + (totalIndexSize / 1024 / 1024).toFixed(2) + " Mo");
```

**Calcul du Working Set** :
```javascript
function calculateWorkingSet() {
  const stats = db.serverStatus().wiredTiger.cache;

  print("=== Working Set Analysis ===");
  print("Bytes in cache: " + (stats["bytes currently in the cache"] / 1024 / 1024).toFixed(2) + " Mo");
  print("Max cache size: " + (stats["maximum bytes configured"] / 1024 / 1024).toFixed(2) + " Mo");
  print("Pages read into cache: " + stats["pages read into cache"]);
  print("Pages written from cache: " + stats["pages written from cache"]);

  const ratio = (stats["bytes currently in the cache"] / stats["maximum bytes configured"]) * 100;
  print("Cache utilization: " + ratio.toFixed(1) + "%");

  if (ratio > 95) {
    print("⚠️  Cache presque plein - Working Set > RAM");
  }
}

calculateWorkingSet();
```

**Symptômes de problème** :
```markdown
⚠️ Swap utilisé > 100 Mo → Performance dégradée
⚠️ RAM disponible < 20% → Risque de swap
⚠️ Cache hit ratio < 90% → Working Set > RAM
⚠️ Page faults fréquents → Données swappées
```

**Actions correctives** :
- Augmenter la RAM
- Réduire la taille du working set (archivage)
- Optimiser les index
- Désactiver le swap (`swapoff -a`)
- Ajuster cache WiredTiger

---

### 💿 Disque (Stockage)

#### ✅ Checklist Disque

| Point de vérification | Valeur Cible | Priorité | Action |
|----------------------|--------------|----------|--------|
| Type de disque | SSD/NVMe | 🟠 | Performance |
| IOPS disponibles | > 3000 (SSD) | 🟠 | Charge I/O |
| Latence disque | < 10ms | 🟠 | Performance |
| Espace libre | > 20% | 🔴 | Éviter saturation |
| Système de fichiers | XFS ou ext4 | 🟡 | Recommandé |
| Point de montage séparé | Oui | 🟡 | Isolation |

**Commandes de vérification** :
```bash
# Espace disque
df -h

# Inodes (important)
df -i

# Type de disque et partitions
lsblk
fdisk -l

# Système de fichiers
mount | grep mongo

# Performance I/O
iostat -x 1 5
# Regarder : %util, await, r/s, w/s

# Test performance disque
fio --name=random-write --ioengine=libaio --iodepth=32 --rw=randwrite --bs=4k --direct=1 --size=1G --numjobs=4 --runtime=60 --time_based --group_reporting

# Latence disque
ioping /path/to/mongodb/data
```

**Analyse I/O MongoDB** :
```javascript
// Statistiques I/O
db.serverStatus().wiredTiger.concurrentTransactions

// Background flush
db.serverStatus().wiredTiger.log

// Data file statistics
db.serverStatus().wiredTiger["data-handle"]
```

**Calcul de l'espace nécessaire** :
```javascript
function estimateStorage() {
  let total = 0;

  db.getCollectionNames().forEach(coll => {
    const stats = db[coll].stats();
    total += stats.size + stats.totalIndexSize;
  });

  const current = total / 1024 / 1024 / 1024;  // Go
  const withJournal = current * 1.1;  // +10% pour journal
  const withOplog = withJournal * 1.05;  // +5% pour oplog
  const recommended = withOplog * 1.5;  // +50% buffer

  print("=== Estimation Stockage ===");
  print("Données actuelles: " + current.toFixed(2) + " Go");
  print("Avec journal: " + withJournal.toFixed(2) + " Go");
  print("Avec oplog: " + withOplog.toFixed(2) + " Go");
  print("Recommandé (buffer 50%): " + recommended.toFixed(2) + " Go");
}

estimateStorage();
```

**Symptômes de problème** :
```markdown
⚠️ Espace libre < 10% → Critique
⚠️ %util > 80% constant → Disque saturé
⚠️ await > 50ms → Latence élevée
⚠️ IOPS < 1000 (HDD) → Trop lent
⚠️ Inodes < 10% libre → Risque saturation
```

**Actions correctives** :
- Migrer vers SSD/NVMe
- Ajouter du stockage
- Archiver anciennes données
- Compression WiredTiger
- Sharding pour distribuer

**Configuration recommandée** :
```bash
# XFS (recommandé)
mkfs.xfs -f /dev/sdb1
mount -o noatime,nodiratime /dev/sdb1 /var/lib/mongodb

# Ajout dans /etc/fstab
/dev/sdb1 /var/lib/mongodb xfs noatime,nodiratime 0 0

# Désactiver le readahead
blockdev --setra 32 /dev/sdb1

# Vérifier
blockdev --getra /dev/sdb1
```

---

### 🌐 Réseau

#### ✅ Checklist Réseau

| Point de vérification | Valeur Cible | Priorité | Action |
|----------------------|--------------|----------|--------|
| Latence intra-cluster | < 1ms | 🟠 | Même datacenter |
| Bande passante | ≥ 1 Gbps | 🟠 | Performance |
| Connexions actives | < 80% max | 🟠 | Pool sizing |
| Firewall configuré | Oui | 🔴 | Sécurité |
| Latence client-serveur | < 50ms | 🟡 | UX |

**Commandes de vérification** :
```bash
# Latence réseau
ping -c 10 mongodb-server

# Entre membres replica set
ping -c 10 replica-member-2

# Bande passante
iftop
# ou
nload

# Connexions MongoDB
netstat -an | grep 27017 | wc -l

# Connexions établies
ss -s

# Test latence avec nc
time echo "test" | nc mongodb-server 27017

# MTU (taille paquet)
ip link show | grep mtu
```

**Vérifications MongoDB** :
```javascript
// Connexions actives
db.serverStatus().connections

// Résultat :
{
  current: 52,        // Connexions actuelles
  available: 51148,   // Connexions disponibles
  totalCreated: 1234  // Total créé depuis démarrage
}

// Limite de connexions
db.adminCommand({ getParameter: 1, maxIncomingConnections: 1 })

// Statistiques réseau
db.serverStatus().network
{
  bytesIn: NumberLong("..."),
  bytesOut: NumberLong("..."),
  numRequests: NumberLong("...")
}
```

**Configuration des connexions** :
```yaml
# mongod.conf
net:
  port: 27017
  bindIp: 0.0.0.0  # ⚠️ Sécuriser avec firewall
  maxIncomingConnections: 65536

  # Compression réseau (MongoDB 3.4+)
  compression:
    compressors: snappy,zlib,zstd
```

**Connection Pooling (application)** :
```javascript
// Node.js example
const client = new MongoClient(uri, {
  maxPoolSize: 100,        // Max connexions dans le pool
  minPoolSize: 10,         // Min connexions maintenues
  maxIdleTimeMS: 30000,    // Timeout connexion idle
  waitQueueTimeoutMS: 5000 // Timeout attente connexion
});
```

**Symptômes de problème** :
```markdown
⚠️ Latence > 10ms intra-cluster → Problème réseau
⚠️ Connexions > 90% du max → Augmenter limite
⚠️ Connexions croissantes → Fuite connexions app
⚠️ Bytes In/Out déséquilibrés → Vérifier pattern
⚠️ Packet loss > 0.1% → Problème réseau
```

**Actions correctives** :
- Augmenter maxIncomingConnections
- Optimiser connection pooling app
- Vérifier configuration réseau
- Utiliser compression réseau
- Placer membres dans même zone

---

## Configuration MongoDB

### ⚙️ Configuration WiredTiger

#### ✅ Checklist WiredTiger

| Paramètre | Valeur Recommandée | Description |
|-----------|-------------------|-------------|
| **cacheSizeGB** | 50-60% RAM | Cache mémoire |
| **directoryForIndexes** | true | Index séparés |
| **journalCompressor** | snappy | Compression journal |
| **collectionConfig.blockCompressor** | snappy | Compression données |

**Configuration fichier** :
```yaml
# mongod.conf
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2  # 50% de 4GB RAM
      journalCompressor: snappy
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true
```

**Vérification runtime** :
```javascript
// Configuration actuelle
db.serverStatus().wiredTiger.cache

// Ajuster le cache (sans redémarrage)
db.adminCommand({
  setParameter: 1,
  wiredTigerEngineRuntimeConfig: "cache_size=2GB"
})

// Statistiques de compression
db.collection.stats().wiredTiger
```

**Calcul cache optimal** :
```javascript
function recommendCacheSize() {
  const totalRAM = db.serverStatus().mem.resident;
  const indexSize = /* calculer total des index */;

  // Formule : 50% RAM ou (Index Size + Working Set), au plus grand
  const fiftyPercent = totalRAM * 0.5;
  const recommended = Math.max(fiftyPercent, indexSize * 1.2);

  print("RAM totale: " + totalRAM + " Mo");
  print("50% RAM: " + fiftyPercent + " Mo");
  print("Taille index: " + indexSize + " Mo");
  print("Cache recommandé: " + recommended + " Mo");
}
```

---

### ⚙️ Paramètres Système (OS)

#### ✅ Checklist OS

| Paramètre | Valeur Recommandée | Priorité |
|-----------|-------------------|----------|
| **Transparent Huge Pages** | disabled | 🔴 |
| **NUMA** | interleave (si multi-socket) | 🟠 |
| **ulimit (files)** | ≥ 64000 | 🔴 |
| **ulimit (processes)** | ≥ 64000 | 🔴 |
| **Swappiness** | 1 | 🟠 |
| **TCP keepalive** | Configuré | 🟡 |

**Vérifications** :
```bash
# Transparent Huge Pages (doit être disabled)
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag

# Désactiver THP
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/defrag

# Permanent (ajouter à /etc/rc.local)
cat << 'EOF' > /etc/init.d/disable-transparent-hugepages
#!/bin/bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF

chmod 755 /etc/init.d/disable-transparent-hugepages
update-rc.d disable-transparent-hugepages defaults

# ulimit
ulimit -a

# Configurer dans /etc/security/limits.conf
mongod soft nofile 64000
mongod hard nofile 64000
mongod soft nproc 64000
mongod hard nproc 64000

# Swappiness
cat /proc/sys/vm/swappiness
# Définir à 1
echo 1 > /proc/sys/vm/swappiness
# Permanent dans /etc/sysctl.conf
vm.swappiness = 1

# TCP keepalive
cat /proc/sys/net/ipv4/tcp_keepalive_time
# Recommandé : 120
sysctl -w net.ipv4.tcp_keepalive_time=120
```

**Script de vérification** :
```bash
#!/bin/bash
echo "=== MongoDB System Configuration Audit ==="

echo -e "\n1. Transparent Huge Pages:"
cat /sys/kernel/mm/transparent_hugepage/enabled | grep -o "\[.*\]"

echo -e "\n2. ulimit (nofile):"
ulimit -n

echo -e "\n3. Swappiness:"
cat /proc/sys/vm/swappiness

echo -e "\n4. TCP Keepalive:"
cat /proc/sys/net/ipv4/tcp_keepalive_time

echo -e "\n5. NUMA (si applicable):"
numactl --hardware 2>/dev/null || echo "Non applicable"

echo -e "\n=== Recommendations ==="
[[ $(cat /sys/kernel/mm/transparent_hugepage/enabled) != *"[never]"* ]] && echo "⚠️  Disable Transparent Huge Pages"
[[ $(ulimit -n) -lt 64000 ]] && echo "⚠️  Increase ulimit nofile to 64000"
[[ $(cat /proc/sys/vm/swappiness) -gt 1 ]] && echo "⚠️  Set swappiness to 1"
```

---

## Architecture Distribuée

### 🔄 Replica Set

#### ✅ Checklist Replica Set

| Point de vérification | Recommandation | Priorité |
|----------------------|----------------|----------|
| Nombre de membres | 3 (min) ou 5 | 🔴 |
| Nombre impair | Oui (ou arbiter) | 🔴 |
| Distribution géographique | Multi-AZ/Multi-DC | 🟠 |
| Priority configurées | Oui | 🟡 |
| Hidden members | Pour backup/analytics | 🟡 |
| Oplog size | ≥ 5% du dataset | 🟠 |

**Vérifications** :
```javascript
// Status du replica set
rs.status()

// Configuration
rs.conf()

// Membre actuel
db.isMaster()

// Oplog
db.getReplicationInfo()
{
  logSizeMB: 1024,
  usedMB: 512,
  timeDiff: 3600,  // Secondes de données dans oplog
  tFirst: ISODate("..."),
  tLast: ISODate("..."),
  now: ISODate("...")
}

// Replication lag
rs.printSlaveReplicationInfo()
```

**Oplog sizing** :
```javascript
// Vérifier la taille actuelle
db.getReplicationInfo()

// Changer la taille (nécessite redémarrage du membre)
// 1. Redémarrer en standalone
// 2. Changer la taille
db.adminCommand({ replSetResizeOplog: 1, size: 2048 })  // 2 Go
```

**Configuration optimale** :
```javascript
// Configuration type pour 3 membres
cfg = rs.conf()
cfg.members = [
  {
    _id: 0,
    host: "mongo1.example.com:27017",
    priority: 2  // Primary préféré
  },
  {
    _id: 1,
    host: "mongo2.example.com:27017",
    priority: 1
  },
  {
    _id: 2,
    host: "mongo3.example.com:27017",
    priority: 1
  }
]
cfg.settings = {
  chainingAllowed: true,
  heartbeatTimeoutSecs: 10,
  electionTimeoutMillis: 10000
}
rs.reconfig(cfg)
```

**Symptômes de problème** :
```markdown
⚠️ Lag secondaires > 10 secondes → Performance
⚠️ Oplog < 1 heure de données → Trop petit
⚠️ Élections fréquentes → Problème réseau
⚠️ Membres DOWN → Haute disponibilité compromise
⚠️ Oplog wrapping → Resync nécessaire
```

---

### 📊 Sharded Cluster

#### ✅ Checklist Sharding

| Point de vérification | Recommandation | Priorité |
|----------------------|----------------|----------|
| Shard key choisie | Cardinalité élevée | 🔴 |
| Distribution équilibrée | Oui | 🟠 |
| Jumbo chunks | Aucun | 🔴 |
| Config servers | 3 (CSRS) | 🔴 |
| Mongos instances | ≥ 2 | 🟠 |
| Balancer actif | Oui (fenêtre définie) | 🟡 |

**Vérifications** :
```javascript
// Status général
sh.status()

// Distribution des chunks
db.getSiblingDB("config").chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])

// Jumbo chunks
db.getSiblingDB("config").chunks.find({ jumbo: true })

// Balancer status
sh.getBalancerState()
sh.isBalancerRunning()

// Statistiques par shard
db.printShardingStatus()

// Config servers
sh.status().configsvr
```

**Analyse de la shard key** :
```javascript
function analyzeShardKey(namespace) {
  const [dbName, collName] = namespace.split('.');
  const coll = db.getSiblingDB(dbName).getCollection(collName);

  print("=== Analyse Shard Key : " + namespace + " ===");

  // Récupérer la shard key
  const collInfo = db.getSiblingDB("config").collections.findOne({
    _id: namespace
  });

  if (!collInfo) {
    print("Collection non shardée");
    return;
  }

  const shardKey = collInfo.key;
  print("Shard Key: " + JSON.stringify(shardKey));

  // Distribution
  const chunks = db.getSiblingDB("config").chunks.aggregate([
    { $match: { ns: namespace } },
    { $group: {
        _id: "$shard",
        chunks: { $sum: 1 },
        jumbo: { $sum: { $cond: ["$jumbo", 1, 0] } }
      }
    }
  ]).toArray();

  print("\nDistribution des chunks:");
  chunks.forEach(s => {
    print("  " + s._id + ": " + s.chunks + " chunks" +
          (s.jumbo > 0 ? " (⚠️ " + s.jumbo + " jumbo)" : ""));
  });

  // Cardinalité
  const keyField = Object.keys(shardKey)[0];
  const cardinality = coll.distinct(keyField).length;
  print("\nCardinalité (" + keyField + "): " + cardinality);
}

analyzeShardKey("mydb.users");
```

**Configuration du balancer** :
```javascript
// Désactiver pendant maintenance
sh.stopBalancer()

// Réactiver
sh.startBalancer()

// Définir fenêtre de balancing
db.getSiblingDB("config").settings.update(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "23:00",  // 11 PM
        stop: "06:00"    // 6 AM
      }
    }
  },
  { upsert: true }
)

// Vérifier
sh.getBalancerWindow()
```

**Symptômes de problème** :
```markdown
⚠️ Distribution déséquilibrée (ratio > 2x) → Mauvaise shard key
⚠️ Jumbo chunks présents → Split manuel nécessaire
⚠️ Balancer constamment actif → Shard key problématique
⚠️ Migrations lentes → Jumbo chunks ou réseau
⚠️ Shard unique surchargé → Redistribution nécessaire
```

---

## Monitoring et Alertes

### 📈 Métriques à Surveiller

#### ✅ Checklist Monitoring

| Catégorie | Métriques | Fréquence | Outil |
|-----------|-----------|-----------|-------|
| **Système** | CPU, RAM, Disque, Réseau | 1 min | Prometheus, Datadog |
| **MongoDB** | Ops/sec, Latence, Connexions | 1 min | mongostat, Atlas |
| **Réplication** | Lag, Oplog, Elections | 5 min | rs.status() |
| **Sharding** | Distribution, Migrations | 10 min | sh.status() |
| **Sécurité** | Auth failures, Connections | 5 min | Logs |

**Outils essentiels** :
```bash
# mongostat - Vue d'ensemble
mongostat --host localhost:27017 --rowcount 10

# mongotop - Top collections
mongotop --host localhost:27017 10

# Logs en temps réel
tail -f /var/log/mongodb/mongod.log

# Avec grep pour erreurs
tail -f /var/log/mongodb/mongod.log | grep -i error
```

**Configuration Prometheus** :
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']

# mongodb-exporter
docker run -d \
  --name mongodb-exporter \
  -p 9216:9216 \
  percona/mongodb_exporter:0.40 \
  --mongodb.uri=mongodb://user:pass@localhost:27017
```

**Grafana Dashboard** :
```markdown
Dashboards recommandés :
- MongoDB Overview (ID: 2583)
- MongoDB Exporter (ID: 7353)
- WiredTiger (ID: 12079)
```

---

### 🚨 Alertes Critiques

#### Configuration des Seuils

```yaml
# Exemple AlertManager
groups:
  - name: mongodb_alerts
    rules:
      # CPU
      - alert: HighCPU
        expr: cpu_usage > 80
        for: 5m
        annotations:
          summary: "High CPU on {{ $labels.instance }}"

      # RAM
      - alert: LowMemory
        expr: memory_available_percent < 20
        for: 5m

      # Disque
      - alert: DiskSpaceLow
        expr: disk_free_percent < 20
        for: 5m

      # Réplication lag
      - alert: ReplicationLag
        expr: mongodb_replset_member_replication_lag > 10
        for: 5m

      # Connexions
      - alert: HighConnections
        expr: mongodb_connections_current / mongodb_connections_available > 0.8
        for: 5m
```

---

## Dimensionnement

### 📊 Calcul des Ressources

#### RAM Sizing

```
RAM nécessaire = Working Set + Index Size + Buffer

Formule détaillée :
RAM = (Données actives * 1.2) + (Total index * 1.1) + 2 GB (OS + buffer)

Minimum absolu : 8 GB
Recommandé production : 16-32 GB
Enterprise / Volume élevé : 64-128 GB+
```

#### CPU Sizing

```
Calcul basé sur :
- Ops/seconde attendues
- Complexité des requêtes
- Agrégations

Guide :
< 1000 ops/sec : 4 cores
1000-5000 ops/sec : 8 cores
5000-20000 ops/sec : 16 cores
> 20000 ops/sec : 32+ cores ou sharding
```

#### Disque Sizing

```
Capacité = (Données + Index) * 1.5 * Croissance

Exemple :
Données actuelles : 100 GB
Index : 20 GB
Croissance 12 mois : 2x
Buffer sécurité : 50%

Calcul : (100 + 20) * 2 * 1.5 = 360 GB
```

**Script de recommandation** :
```javascript
function recommendInfrastructure() {
  // Collecter les stats
  let totalData = 0;
  let totalIndex = 0;

  db.getCollectionNames().forEach(coll => {
    const stats = db[coll].stats();
    totalData += stats.size;
    totalIndex += stats.totalIndexSize;
  });

  const dataGB = totalData / 1024 / 1024 / 1024;
  const indexGB = totalIndex / 1024 / 1024 / 1024;
  const ops = db.serverStatus().opcounters;

  print("=== Recommandations Infrastructure ===");
  print("\nDonnées actuelles: " + dataGB.toFixed(2) + " GB");
  print("Index: " + indexGB.toFixed(2) + " GB");

  // RAM
  const workingSet = (dataGB * 0.3) + indexGB;  // 30% données actives
  const ramRec = Math.max(16, workingSet * 1.2);
  print("\nRAM recommandée: " + Math.ceil(ramRec) + " GB");

  // CPU
  const totalOps = ops.insert + ops.query + ops.update + ops.delete;
  let cpuRec = 4;
  if (totalOps > 1000) cpuRec = 8;
  if (totalOps > 5000) cpuRec = 16;
  if (totalOps > 20000) cpuRec = 32;
  print("CPU recommandé: " + cpuRec + " cores");

  // Disque
  const diskRec = (dataGB + indexGB) * 2 * 1.5;  // 2x croissance, 1.5x buffer
  print("Disque recommandé: " + Math.ceil(diskRec) + " GB SSD");
}

recommendInfrastructure();
```

---

## Checklist par Environnement

### 🧪 Développement

```markdown
□ Ressources minimales (2-4 GB RAM, 2 cores)
□ Standalone ou RS 1 membre
□ Pas de haute disponibilité requise
□ Désactiver authentication (optionnel)
□ Logging verbose pour debug
□ Profiler niveau 2 acceptable
```

### 🧪 Staging/QA

```markdown
□ Configuration proche de production
□ Replica Set 3 membres
□ Même version MongoDB que prod
□ Données anonymisées de production
□ Monitoring basique
□ Backup réguliers
```

### 🏭 Production

```markdown
□ Replica Set 3-5 membres (minimum)
□ Multi-AZ ou Multi-DC
□ Authentication activée
□ TLS/SSL configuré
□ Monitoring complet + alertes
□ Backup automatisés + tests restauration
□ Oplog ≥ 24h de données
□ Ressources dimensionnées + 50% buffer
□ Documentation à jour
□ Procédures disaster recovery
```

---

## Actions Prioritaires

### 🔴 Critique - À corriger immédiatement

```markdown
□ Transparent Huge Pages activé
□ RAM < 8 GB en production
□ Disque plein > 90%
□ Swap utilisé > 1 GB
□ Replica Set < 3 membres en prod
□ Pas de monitoring
□ Pas de backup
□ Sécurité non configurée
```

### 🟠 Important - À planifier sous 2 semaines

```markdown
□ CPU > 80% constant
□ RAM utilisée > 80%
□ Disque > 80%
□ I/O Wait > 20%
□ Replication lag > 30 secondes
□ Jumbo chunks présents
□ ulimits non configurés
□ Distribution sharding déséquilibrée (ratio > 2x)
```

### 🟡 Modéré - À améliorer progressivement

```markdown
□ CPU > 70%
□ Pas de members hidden pour backup
□ Oplog < 5% du dataset
□ Balancer sans fenêtre définie
□ Configuration système non optimisée
□ Monitoring partiel
□ Documentation incomplète
```

---

## Template de Rapport d'Audit

```markdown
# Rapport d'Audit d'Infrastructure
**Date** : [DATE]
**Environnement** : [DEV/STAGING/PROD]
**Auditeur** : [NOM]

## Résumé Exécutif
- Architecture : [Standalone/RS/Sharded]
- Membres : [NOMBRE]
- Problèmes critiques : X
- Recommandations prioritaires : X

## Ressources Système

### Serveur 1 : [HOSTNAME]
| Ressource | Actuel | Recommandé | Statut |
|-----------|--------|------------|--------|
| CPU | X cores, Y% util | Z cores | 🟢/🟡/🔴 |
| RAM | X GB (Y% util) | Z GB | 🟢/🟡/🔴 |
| Disque | X GB (Y% libre) | Z GB | 🟢/🟡/🔴 |
| IOPS | X | Y | 🟢/🟡/🔴 |

[Répéter pour chaque serveur]

## Configuration MongoDB

### WiredTiger
- Cache Size : X GB (Y% RAM)
- Compression : [TYPE]
- Journal : [ENABLED/DISABLED]

### Replica Set
- Membres : X
- Oplog Size : X GB (Y heures)
- Replication Lag : X secondes

### Sharding (si applicable)
- Shards : X
- Distribution : [BALANCED/UNBALANCED]
- Jumbo Chunks : X

## Problèmes Identifiés

### Critiques 🔴
1. [PROBLÈME]
   - Impact : [DESCRIPTION]
   - Action : [IMMÉDIATE]

### Importants 🟠
1. [PROBLÈME]
   - Impact : [DESCRIPTION]
   - Action : [SOUS 2 SEMAINES]

## Recommandations

### Court terme (< 1 semaine)
1. Désactiver Transparent Huge Pages
2. Augmenter RAM de X à Y GB
3. Configurer monitoring et alertes

### Moyen terme (1-4 semaines)
1. Migrer vers SSD
2. Ajouter membre hidden
3. Optimiser configuration OS

### Long terme (> 1 mois)
1. Planifier sharding
2. Multi-région pour DR
3. Migration Atlas

## Plan d'Action
| Action | Priorité | Responsable | Deadline |
|--------|----------|-------------|----------|
| [ACTION 1] | 🔴 | [NOM] | [DATE] |
| [ACTION 2] | 🟠 | [NOM] | [DATE] |

## Coûts Estimés
- Infrastructure : [MONTANT]
- Migration : [MONTANT]
- Formation : [MONTANT]
**Total** : [MONTANT]

## Annexes
- Captures screenshots
- Outputs de commandes
- Scripts utilisés
```

---

## Scripts Utilitaires

### Script Complet d'Audit

```bash
#!/bin/bash
# audit_infrastructure.sh

echo "========================================="
echo "AUDIT INFRASTRUCTURE MONGODB"
echo "Date: $(date)"
echo "========================================="

echo -e "\n### SYSTÈME ###"
echo "OS: $(cat /etc/os-release | grep PRETTY_NAME | cut -d'"' -f2)"
echo "Kernel: $(uname -r)"
echo "Uptime: $(uptime -p)"

echo -e "\n### CPU ###"
echo "Cores: $(nproc)"
echo "Load: $(uptime | awk -F'load average:' '{print $2}')"
top -bn1 | head -5

echo -e "\n### RAM ###"
free -h

echo -e "\n### DISQUE ###"
df -h | grep -v tmpfs

echo -e "\n### RÉSEAU ###"
echo "Connections MongoDB:"
netstat -an | grep 27017 | wc -l

echo -e "\n### CONFIGURATION OS ###"
echo "Transparent Huge Pages:"
cat /sys/kernel/mm/transparent_hugepage/enabled
echo "Swappiness:"
cat /proc/sys/vm/swappiness
echo "ulimit:"
ulimit -n

echo -e "\n### MONGODB ###"
mongo --quiet --eval "
  print('Version: ' + db.version());
  print('Uptime: ' + db.serverStatus().uptime + 's');
  var mem = db.serverStatus().mem;
  print('Memory - Resident: ' + mem.resident + 'MB');
  print('Connections: ' + db.serverStatus().connections.current + '/' + db.serverStatus().connections.available);
  if (rs.status().ok) {
    print('Replica Set: ' + rs.status().set);
    print('Members: ' + rs.status().members.length);
  }
"

echo -e "\n========================================="
echo "FIN AUDIT"
echo "========================================="
```

---

## Ressources Complémentaires

### Documentation Officielle
- [Production Notes](https://www.mongodb.com/docs/manual/administration/production-notes/)
- [Production Checklist](https://www.mongodb.com/docs/manual/administration/production-checklist-operations/)
- [Hardware and OS Configuration](https://www.mongodb.com/docs/manual/administration/production-notes/#hardware-considerations)

### Guides Avancés
- [Performance Best Practices](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)
- [Capacity Planning](https://www.mongodb.com/docs/manual/core/capacity-planning/)

### Outils
- **MongoDB Atlas** : Monitoring et alertes intégrés
- **Ops Manager** : Monitoring on-premise
- **Prometheus + Grafana** : Monitoring open-source
- **Percona Monitoring** : Suite complète

---


⏭️ [Docker et Docker Compose](/annexes/docker-compose/README.md)
