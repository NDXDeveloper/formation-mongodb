🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.10 Monitoring d'un Replica Set

## Introduction

Le monitoring d'un Replica Set est essentiel pour garantir la haute disponibilité, détecter les problèmes avant qu'ils n'impactent les utilisateurs, et optimiser les performances. Un système de monitoring efficace combine la surveillance des métriques système, des métriques MongoDB spécifiques, et des métriques métier pour fournir une vue complète de la santé du cluster.

## Métriques Clés

### 1. Métriques de Disponibilité

#### État du Replica Set

```javascript
// Vérification de base
rs.status()
```

**Métriques critiques** :

| Métrique | Description | Valeur Saine | Alerte si |
|----------|-------------|--------------|-----------|
| `myState` | État du membre actuel | 1 (PRIMARY) ou 2 (SECONDARY) | 8 (DOWN), 3 (RECOVERING) |
| `members[].state` | État de chaque membre | 1 ou 2 | 8, 6 (UNKNOWN) |
| `members[].health` | Santé du membre | 1 | 0 |
| `set` | Nom du Replica Set | Nom configuré | Mismatch |
| `ok` | Statut de la commande | 1 | 0 |

#### Extraction des Métriques d'État

```javascript
function checkReplicaSetHealth() {
  const status = rs.status()

  const metrics = {
    timestamp: new Date(),
    replicaSet: status.set,

    // État global
    ok: status.ok,

    // Membres
    totalMembers: status.members.length,
    healthyMembers: status.members.filter(m => m.health === 1).length,

    // Primary
    primaryCount: status.members.filter(m => m.state === 1).length,
    primaryName: status.members.find(m => m.state === 1)?.name || 'NONE',

    // Secondary
    secondaryCount: status.members.filter(m => m.state === 2).length,

    // Membres problématiques
    downMembers: status.members.filter(m => m.state === 8).map(m => m.name),
    recoveringMembers: status.members.filter(m => m.state === 3).map(m => m.name),
    unknownMembers: status.members.filter(m => m.state === 6).map(m => m.name),

    // Élections
    term: status.term,
    lastElection: status.members.find(m => m.state === 1)?.electionDate
  }

  return metrics
}

// Utilisation
const health = checkReplicaSetHealth()
printjson(health)
```

### 2. Métriques de Réplication

#### Replication Lag

**Définition** : Délai entre le Primary et les Secondary

```javascript
function calculateReplicationLag() {
  const status = rs.status()
  const now = status.date

  const lagMetrics = status.members
    .filter(m => m.state === 2)  // SECONDARY uniquement
    .map(m => ({
      member: m.name,
      lagSeconds: m.optimeDate ? (now - m.optimeDate) / 1000 : null,
      lastHeartbeat: m.lastHeartbeat,
      pingMs: m.pingMs,
      syncSourceHost: m.syncSourceHost
    }))

  const maxLag = Math.max(...lagMetrics.map(m => m.lagSeconds || 0))
  const avgLag = lagMetrics.reduce((sum, m) => sum + (m.lagSeconds || 0), 0) / lagMetrics.length

  return {
    members: lagMetrics,
    maxLag: maxLag,
    avgLag: avgLag,
    critical: maxLag > 60,  // > 1 minute
    warning: maxLag > 10    // > 10 secondes
  }
}

// Utilisation
const lag = calculateReplicationLag()
if (lag.critical) {
  print(`CRITICAL: Max replication lag is ${lag.maxLag.toFixed(2)}s`)
} else if (lag.warning) {
  print(`WARNING: Max replication lag is ${lag.maxLag.toFixed(2)}s`)
}
```

#### Oplog Window

```javascript
function checkOplogWindow() {
  const replInfo = db.getSiblingDB('local').oplog.rs.stats()
  const firstEntry = db.getSiblingDB('local').oplog.rs.find().sort({$natural: 1}).limit(1).next()
  const lastEntry = db.getSiblingDB('local').oplog.rs.find().sort({$natural: -1}).limit(1).next()

  const windowSeconds = (lastEntry.ts.getTime() - firstEntry.ts.getTime())
  const windowHours = windowSeconds / 3600

  return {
    oplogSizeMB: (replInfo.maxSize / 1024 / 1024).toFixed(2),
    usedMB: (replInfo.size / 1024 / 1024).toFixed(2),
    usedPercent: ((replInfo.size / replInfo.maxSize) * 100).toFixed(2),
    windowHours: windowHours.toFixed(2),
    windowDays: (windowHours / 24).toFixed(2),
    firstOpTime: firstEntry.ts,
    lastOpTime: lastEntry.ts,
    warning: windowHours < 24,
    critical: windowHours < 12
  }
}

// Utilisation
const oplog = checkOplogWindow()
print(`Oplog window: ${oplog.windowHours} hours (${oplog.usedPercent}% used)`)

if (oplog.critical) {
  print(`CRITICAL: Oplog window < 12 hours`)
} else if (oplog.warning) {
  print(`WARNING: Oplog window < 24 hours`)
}
```

### 3. Métriques de Performance

#### Opérations par Seconde

```javascript
function getOperationMetrics() {
  const status1 = db.serverStatus()
  const opcounters1 = status1.opcounters
  const timestamp1 = new Date()

  // Attendre 1 seconde
  sleep(1000)

  const status2 = db.serverStatus()
  const opcounters2 = status2.opcounters
  const timestamp2 = new Date()

  const duration = (timestamp2 - timestamp1) / 1000

  return {
    insert: ((opcounters2.insert - opcounters1.insert) / duration).toFixed(2),
    query: ((opcounters2.query - opcounters1.query) / duration).toFixed(2),
    update: ((opcounters2.update - opcounters1.update) / duration).toFixed(2),
    delete: ((opcounters2.delete - opcounters1.delete) / duration).toFixed(2),
    getmore: ((opcounters2.getmore - opcounters1.getmore) / duration).toFixed(2),
    command: ((opcounters2.command - opcounters1.command) / duration).toFixed(2),
    total: function() {
      return (
        parseFloat(this.insert) +
        parseFloat(this.query) +
        parseFloat(this.update) +
        parseFloat(this.delete)
      ).toFixed(2)
    }
  }
}

// Utilisation
const ops = getOperationMetrics()
print(`Operations/sec - Insert: ${ops.insert}, Query: ${ops.query}, Update: ${ops.update}, Delete: ${ops.delete}`)
```

#### Connexions

```javascript
function getConnectionMetrics() {
  const status = db.serverStatus()
  const conn = status.connections

  return {
    current: conn.current,
    available: conn.available,
    totalCreated: conn.totalCreated,
    active: conn.active,
    threaded: conn.threaded,

    // Pourcentages
    usagePercent: ((conn.current / (conn.current + conn.available)) * 100).toFixed(2),

    // Alertes
    warning: conn.current > (conn.current + conn.available) * 0.7,
    critical: conn.current > (conn.current + conn.available) * 0.9
  }
}

// Utilisation
const conn = getConnectionMetrics()
print(`Connections: ${conn.current} / ${conn.current + conn.available} (${conn.usagePercent}%)`)

if (conn.critical) {
  print(`CRITICAL: Connection usage > 90%`)
} else if (conn.warning) {
  print(`WARNING: Connection usage > 70%`)
}
```

#### Mémoire et Cache

```javascript
function getMemoryMetrics() {
  const status = db.serverStatus()

  return {
    // Mémoire système
    resident: status.mem.resident,
    virtual: status.mem.virtual,
    mapped: status.mem.mapped,

    // WiredTiger cache
    cacheSizeGB: (status.wiredTiger.cache['maximum bytes configured'] / 1024 / 1024 / 1024).toFixed(2),
    cacheUsedGB: (status.wiredTiger.cache['bytes currently in the cache'] / 1024 / 1024 / 1024).toFixed(2),
    cacheUsedPercent: ((status.wiredTiger.cache['bytes currently in the cache'] /
                       status.wiredTiger.cache['maximum bytes configured']) * 100).toFixed(2),

    cacheDirtyGB: (status.wiredTiger.cache['tracked dirty bytes in the cache'] / 1024 / 1024 / 1024).toFixed(2),
    cacheDirtyPercent: ((status.wiredTiger.cache['tracked dirty bytes in the cache'] /
                        status.wiredTiger.cache['maximum bytes configured']) * 100).toFixed(2),

    // Évictions
    evictions: status.wiredTiger.cache['pages evicted by application threads'],
    evictionsModified: status.wiredTiger.cache['modified pages evicted'],

    // Alertes
    warning: (status.wiredTiger.cache['bytes currently in the cache'] /
             status.wiredTiger.cache['maximum bytes configured']) > 0.8,
    critical: (status.wiredTiger.cache['bytes currently in the cache'] /
              status.wiredTiger.cache['maximum bytes configured']) > 0.95
  }
}

// Utilisation
const mem = getMemoryMetrics()
print(`Cache: ${mem.cacheUsedGB} GB / ${mem.cacheSizeGB} GB (${mem.cacheUsedPercent}%)`)
print(`Dirty: ${mem.cacheDirtyGB} GB (${mem.cacheDirtyPercent}%)`)
```

#### Disque I/O

```javascript
function getDiskMetrics() {
  const status = db.serverStatus()

  return {
    // Flush
    flushes: status.wiredTiger.log['log flush operations'],
    flushTime: status.wiredTiger.log['total time spent flushing the log (usecs)'],

    // Sync
    syncs: status.wiredTiger.log['log sync operations'],
    syncTime: status.wiredTiger.log['total time spent synchronizing the log (usecs)'],

    // Checkpoints
    checkpoints: status.wiredTiger.transaction.checkpoint['transaction checkpoints'],
    checkpointTime: status.wiredTiger.transaction.checkpoint['transaction checkpoint total time (msecs)'],

    // Transactions
    transactionsBegun: status.wiredTiger.transaction['transactions begun'],
    transactionsCommitted: status.wiredTiger.transaction['transactions committed'],
    transactionsRolledBack: status.wiredTiger.transaction['transactions rolled back']
  }
}

// Utilisation
const disk = getDiskMetrics()
print(`Flushes: ${disk.flushes}, Syncs: ${disk.syncs}, Checkpoints: ${disk.checkpoints}`)
```

### 4. Métriques Réseau

```javascript
function getNetworkMetrics() {
  const status = db.serverStatus()
  const network = status.network

  return {
    bytesIn: network.bytesIn,
    bytesOut: network.bytesOut,
    numRequests: network.numRequests,

    // Convertir en MB
    bytesInMB: (network.bytesIn / 1024 / 1024).toFixed(2),
    bytesOutMB: (network.bytesOut / 1024 / 1024).toFixed(2),

    // Latence réseau (depuis rs.status())
    memberLatencies: rs.status().members.map(m => ({
      member: m.name,
      pingMs: m.pingMs || 'N/A',
      state: m.stateStr
    }))
  }
}

// Utilisation
const network = getNetworkMetrics()
print(`Network In: ${network.bytesInMB} MB, Out: ${network.bytesOutMB} MB`)
print(`Member latencies:`)
network.memberLatencies.forEach(m => {
  print(`  ${m.member}: ${m.pingMs} ms`)
})
```

## Commandes MongoDB Natives

### rs.status()

Commande principale pour surveiller l'état du Replica Set.

```javascript
// Structure complète
const status = rs.status()

// Informations clés
const analysis = {
  // Méta
  set: status.set,
  date: status.date,
  term: status.term,

  // Membres
  members: status.members.map(m => ({
    name: m.name,
    state: m.state,
    stateStr: m.stateStr,
    health: m.health,
    uptime: m.uptime,
    optime: m.optimeDate,
    lastHeartbeat: m.lastHeartbeat,
    lastHeartbeatRecv: m.lastHeartbeatRecv,
    pingMs: m.pingMs,
    syncSourceHost: m.syncSourceHost,
    configVersion: m.configVersion
  })),

  // Analyse
  isPrimaryPresent: status.members.some(m => m.state === 1),
  primaryName: status.members.find(m => m.state === 1)?.name,
  secondaryCount: status.members.filter(m => m.state === 2).length,
  healthyCount: status.members.filter(m => m.health === 1).length,
  unhealthyMembers: status.members.filter(m => m.health !== 1).map(m => m.name)
}

printjson(analysis)
```

### rs.conf()

Configuration du Replica Set.

```javascript
const config = rs.conf()

const configAnalysis = {
  _id: config._id,
  version: config.version,
  protocolVersion: config.protocolVersion,

  // Membres
  totalMembers: config.members.length,
  votingMembers: config.members.filter(m => m.votes === 1).length,
  arbiters: config.members.filter(m => m.arbiterOnly).length,
  hiddenMembers: config.members.filter(m => m.hidden).length,
  delayedMembers: config.members.filter(m => m.slaveDelay > 0).length,

  // Settings
  settings: {
    electionTimeoutMillis: config.settings?.electionTimeoutMillis,
    catchUpTimeoutMillis: config.settings?.catchUpTimeoutMillis,
    chainingAllowed: config.settings?.chainingAllowed,
    writeConcernDefault: config.settings?.getLastErrorDefaults,
    customWriteConcernModes: config.settings?.getLastErrorModes
  },

  // Validation
  isOddVotingMembers: (config.members.filter(m => m.votes === 1).length % 2) === 1,
  hasPriorityZeroVotingMember: config.members.some(m => m.priority === 0 && m.votes === 1)
}

printjson(configAnalysis)
```

### rs.printReplicationInfo()

Information sur l'oplog.

```javascript
// Sortie formatée
rs.printReplicationInfo()

// Exemple de sortie :
// configured oplog size:   10240MB
// log length start to end: 86394secs (23.99hrs)
// oplog first event time:  Mon Jan 15 2024 10:00:00 GMT+0000 (UTC)
// oplog last event time:   Tue Jan 16 2024 10:59:54 GMT+0000 (UTC)
// now:                     Tue Jan 16 2024 11:00:00 GMT+0000 (UTC)
```

### rs.printSecondaryReplicationInfo()

Lag de réplication pour chaque Secondary.

```javascript
// Sortie formatée
rs.printSecondaryReplicationInfo()

// Exemple de sortie :
// source: mongodb-02:27017
//   syncedTo: Tue Jan 16 2024 10:59:50 GMT+0000 (UTC)
//   0 secs (0 hrs) behind the primary
//
// source: mongodb-03:27017
//   syncedTo: Tue Jan 16 2024 10:59:45 GMT+0000 (UTC)
//   5 secs (0 hrs) behind the primary
```

### db.serverStatus()

Statistiques détaillées du serveur.

```javascript
const serverStatus = db.serverStatus()

// Sections importantes pour monitoring
const relevantMetrics = {
  // Informations de base
  host: serverStatus.host,
  version: serverStatus.version,
  uptime: serverStatus.uptime,

  // Réplication
  repl: {
    setName: serverStatus.repl.setName,
    ismaster: serverStatus.repl.ismaster,
    secondary: serverStatus.repl.secondary,
    rbid: serverStatus.repl.rbid,
    hosts: serverStatus.repl.hosts,
    primary: serverStatus.repl.primary
  },

  // Opcounters
  opcounters: serverStatus.opcounters,
  opcountersRepl: serverStatus.opcountersRepl,

  // Connexions
  connections: serverStatus.connections,

  // Mémoire
  mem: serverStatus.mem,

  // Réseau
  network: serverStatus.network,

  // WiredTiger
  wiredTiger: {
    cache: {
      currentlyInCache: serverStatus.wiredTiger.cache['bytes currently in the cache'],
      maxConfigured: serverStatus.wiredTiger.cache['maximum bytes configured'],
      dirtyBytes: serverStatus.wiredTiger.cache['tracked dirty bytes in the cache']
    },
    concurrentTransactions: serverStatus.wiredTiger.concurrentTransactions,
    transaction: {
      begun: serverStatus.wiredTiger.transaction['transactions begun'],
      committed: serverStatus.wiredTiger.transaction['transactions committed']
    }
  },

  // Locks
  locks: serverStatus.locks,

  // Métriques de latence
  opLatencies: serverStatus.opLatencies
}

printjson(relevantMetrics)
```

### db.currentOp()

Opérations en cours.

```javascript
// Opérations actives
const activeOps = db.currentOp({
  "active": true,
  "secs_running": { $gte: 5 }
})

activeOps.inprog.forEach(op => {
  print(`Operation: ${op.op}`)
  print(`  Namespace: ${op.ns}`)
  print(`  Running for: ${op.secs_running}s`)
  print(`  Query: ${JSON.stringify(op.query || op.command)}`)
  print(`  OpID: ${op.opid}`)
  print(`  Client: ${op.client}`)
  print(`---`)
})

// Statistiques
const opStats = {
  total: activeOps.inprog.length,
  byOperation: {},
  longRunning: activeOps.inprog.filter(op => op.secs_running > 10).length
}

activeOps.inprog.forEach(op => {
  opStats.byOperation[op.op] = (opStats.byOperation[op.op] || 0) + 1
})

printjson(opStats)
```

## Outils de Monitoring

### mongostat

Statistiques en temps réel.

```bash
# Usage basique
mongostat --host mongodb-01:27017

# Avec authentification
mongostat --host mongodb-01:27017 -u admin -p password --authenticationDatabase admin

# Intervalle personnalisé (toutes les 5 secondes)
mongostat --host mongodb-01:27017 5

# Colonnes spécifiques
mongostat --host mongodb-01:27017 -O 'host,repl,set,time,command,query,update,delete'

# Monitoring de tous les membres
mongostat --host mongodb-01:27017,mongodb-02:27017,mongodb-03:27017 \
  --discover \
  -O 'host,repl,set,time,command,query,update,delete,qrw,arw,net_in,net_out,conn'
```

**Sortie exemple** :
```
insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in net_out conn    set repl                time
     0     0      0      0       0     2|0  0.0% 3.2%       0  1.5G  75M 0|0 1|0  158b   38.4k    11    rs0  PRI Jan 15 10:30:00.000
```

**Métriques importantes** :

| Colonne | Description | Alerte si |
|---------|-------------|-----------|
| `insert` | Insertions/sec | Spike anormal |
| `query` | Requêtes/sec | Spike anormal |
| `update` | Updates/sec | Spike anormal |
| `delete` | Suppressions/sec | Spike anormal |
| `command` | Commandes/sec | - |
| `dirty` | % cache dirty | > 20% |
| `used` | % cache utilisé | > 80% |
| `qrw` | Queue read/write | > 10 |
| `arw` | Active read/write | Élevé soutenu |
| `net_in` | Trafic entrant | - |
| `net_out` | Trafic sortant | - |
| `conn` | Connexions | > 80% max |
| `repl` | État (PRI/SEC) | DOWN, UNK |

### mongotop

Temps passé par collection.

```bash
# Usage basique
mongotop --host mongodb-01:27017

# Avec intervalle
mongotop --host mongodb-01:27017 5

# Avec authentification
mongotop --host mongodb-01:27017 -u admin -p password --authenticationDatabase admin

# JSON output (pour parsing)
mongotop --host mongodb-01:27017 --json
```

**Sortie exemple** :
```
                    ns    total    read    write    2024-01-15T10:30:00Z
    mydb.users       145ms    23ms    122ms
    mydb.orders       89ms    67ms     22ms
    mydb.products     12ms    12ms      0ms
```

### MongoDB Exporter (Prometheus)

Exposition des métriques pour Prometheus.

#### Installation

```bash
# Télécharger MongoDB Exporter
wget https://github.com/percona/mongodb_exporter/releases/download/v0.40.0/mongodb_exporter-0.40.0.linux-amd64.tar.gz

# Extraire
tar -xzf mongodb_exporter-0.40.0.linux-amd64.tar.gz

# Déplacer
sudo mv mongodb_exporter /usr/local/bin/

# Créer service systemd
sudo cat > /etc/systemd/system/mongodb_exporter.service <<EOF
[Unit]
Description=MongoDB Exporter
After=network.target

[Service]
Type=simple
User=mongodb_exporter
Environment="MONGODB_URI=mongodb://monitoring:password@localhost:27017"
ExecStart=/usr/local/bin/mongodb_exporter \
  --mongodb.uri=\${MONGODB_URI} \
  --mongodb.collstats-colls=mydb.users,mydb.orders \
  --web.listen-address=:9216 \
  --web.telemetry-path=/metrics

Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Créer utilisateur
sudo useradd --no-create-home --shell /bin/false mongodb_exporter

# Démarrer
sudo systemctl daemon-reload
sudo systemctl start mongodb_exporter
sudo systemctl enable mongodb_exporter

# Vérifier
curl http://localhost:9216/metrics
```

#### Configuration Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets:
          - 'mongodb-01:9216'
          - 'mongodb-02:9216'
          - 'mongodb-03:9216'
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
```

#### Métriques Exportées

```
# Réplication
mongodb_mongod_replset_member_state
mongodb_mongod_replset_member_health
mongodb_mongod_replset_oplog_head_timestamp
mongodb_mongod_replset_oplog_tail_timestamp

# Performance
mongodb_mongod_op_counters_total
mongodb_mongod_op_counters_repl_total
mongodb_connections

# Cache WiredTiger
mongodb_mongod_wiredtiger_cache_bytes{type="total"}
mongodb_mongod_wiredtiger_cache_bytes{type="dirty"}
mongodb_mongod_wiredtiger_cache_bytes_total{type="read"}
mongodb_mongod_wiredtiger_cache_bytes_total{type="written"}

# Réseau
mongodb_network_bytes_total{state="in_bytes"}
mongodb_network_bytes_total{state="out_bytes"}
```

### Grafana Dashboards

#### Configuration de la Source de Données

```json
{
  "name": "Prometheus",
  "type": "prometheus",
  "url": "http://prometheus:9090",
  "access": "proxy",
  "isDefault": true
}
```

#### Dashboard Replica Set - Panneaux Essentiels

**Panel 1 : État des Membres**
```
Query: mongodb_mongod_replset_member_state
Type: Stat
Thresholds: 1 (PRIMARY) = Green, 2 (SECONDARY) = Blue, autres = Red
```

**Panel 2 : Replication Lag**
```
Query:
  (mongodb_mongod_replset_oplog_head_timestamp -
   mongodb_mongod_replset_my_state_optime_timestamp)
Type: Graph
Alert: > 10 secondes
```

**Panel 3 : Oplog Window**
```
Query:
  (mongodb_mongod_replset_oplog_head_timestamp -
   mongodb_mongod_replset_oplog_tail_timestamp) / 3600
Type: Stat
Unit: hours
Alert: < 24 heures
```

**Panel 4 : Operations/sec**
```
Query:
  rate(mongodb_mongod_op_counters_total[1m])
Type: Graph
Legend: {{type}}
```

**Panel 5 : Connexions**
```
Query: mongodb_connections{state="current"}
Type: Gauge
Max: mongodb_connections{state="available"} + mongodb_connections{state="current"}
Thresholds: 70% = Yellow, 90% = Red
```

**Panel 6 : Cache WiredTiger**
```
Query:
  (mongodb_mongod_wiredtiger_cache_bytes{type="dirty"} /
   mongodb_mongod_wiredtiger_cache_bytes{type="total"}) * 100
Type: Gauge
Unit: percent
Alert: > 20%
```

**Panel 7 : Réseau I/O**
```
Query:
  rate(mongodb_network_bytes_total[1m])
Type: Graph
Legend: {{state}}
```

**Dashboard JSON complet** :
```json
{
  "dashboard": {
    "title": "MongoDB Replica Set",
    "panels": [
      {
        "title": "Replica Set Members",
        "targets": [
          {
            "expr": "mongodb_mongod_replset_member_state"
          }
        ],
        "type": "stat"
      },
      {
        "title": "Replication Lag",
        "targets": [
          {
            "expr": "(mongodb_mongod_replset_oplog_head_timestamp - mongodb_mongod_replset_my_state_optime_timestamp)"
          }
        ],
        "type": "graph",
        "alert": {
          "conditions": [
            {
              "type": "query",
              "query": {
                "params": ["A", "5m", "now"]
              },
              "reducer": {
                "type": "avg"
              },
              "evaluator": {
                "type": "gt",
                "params": [10]
              }
            }
          ]
        }
      }
      // ... autres panels
    ]
  }
}
```

## Scripts de Monitoring Personnalisés

### Script de Monitoring Complet

```javascript
// monitoring.js - Script de monitoring complet

class ReplicaSetMonitor {
  constructor(alertThresholds = {}) {
    this.thresholds = {
      replicationLagWarning: alertThresholds.replicationLagWarning || 10,
      replicationLagCritical: alertThresholds.replicationLagCritical || 60,
      oplogWindowWarning: alertThresholds.oplogWindowWarning || 24,
      oplogWindowCritical: alertThresholds.oplogWindowCritical || 12,
      connectionUsageWarning: alertThresholds.connectionUsageWarning || 0.7,
      connectionUsageCritical: alertThresholds.connectionUsageCritical || 0.9,
      cacheUsageWarning: alertThresholds.cacheUsageWarning || 0.8,
      cacheUsageCritical: alertThresholds.cacheUsageCritical || 0.95
    }

    this.alerts = []
  }

  checkReplicaSetHealth() {
    const status = rs.status()

    // Check 1: Primary présent
    const primaryCount = status.members.filter(m => m.state === 1).length
    if (primaryCount === 0) {
      this.alerts.push({
        severity: 'CRITICAL',
        component: 'ReplicaSet',
        message: 'No PRIMARY member found',
        timestamp: new Date()
      })
    } else if (primaryCount > 1) {
      this.alerts.push({
        severity: 'CRITICAL',
        component: 'ReplicaSet',
        message: `Multiple PRIMARY members detected: ${primaryCount}`,
        timestamp: new Date()
      })
    }

    // Check 2: Membres unhealthy
    const unhealthyMembers = status.members.filter(m => m.health !== 1)
    if (unhealthyMembers.length > 0) {
      unhealthyMembers.forEach(m => {
        this.alerts.push({
          severity: m.state === 8 ? 'CRITICAL' : 'WARNING',
          component: 'Member',
          message: `Member ${m.name} is ${m.stateStr} (health: ${m.health})`,
          timestamp: new Date()
        })
      })
    }

    // Check 3: Replication lag
    const lagCheck = this.checkReplicationLag(status)

    // Check 4: Nombre de votants
    const cfg = rs.conf()
    const votingMembers = cfg.members.filter(m => m.votes === 1).length
    if (votingMembers % 2 === 0) {
      this.alerts.push({
        severity: 'WARNING',
        component: 'Configuration',
        message: `Even number of voting members: ${votingMembers}`,
        timestamp: new Date()
      })
    }

    return {
      healthy: this.alerts.filter(a => a.severity === 'CRITICAL').length === 0,
      alerts: this.alerts
    }
  }

  checkReplicationLag(status) {
    const now = status.date

    status.members.forEach(m => {
      if (m.state === 2 && m.optimeDate) {  // SECONDARY
        const lagSeconds = (now - m.optimeDate) / 1000

        if (lagSeconds > this.thresholds.replicationLagCritical) {
          this.alerts.push({
            severity: 'CRITICAL',
            component: 'Replication',
            message: `${m.name} lag: ${lagSeconds.toFixed(2)}s`,
            value: lagSeconds,
            threshold: this.thresholds.replicationLagCritical,
            timestamp: new Date()
          })
        } else if (lagSeconds > this.thresholds.replicationLagWarning) {
          this.alerts.push({
            severity: 'WARNING',
            component: 'Replication',
            message: `${m.name} lag: ${lagSeconds.toFixed(2)}s`,
            value: lagSeconds,
            threshold: this.thresholds.replicationLagWarning,
            timestamp: new Date()
          })
        }
      }
    })
  }

  checkOplogWindow() {
    const replInfo = db.getReplicationInfo()
    const windowHours = replInfo.timeDiff / 3600

    if (windowHours < this.thresholds.oplogWindowCritical) {
      this.alerts.push({
        severity: 'CRITICAL',
        component: 'Oplog',
        message: `Oplog window: ${windowHours.toFixed(2)} hours`,
        value: windowHours,
        threshold: this.thresholds.oplogWindowCritical,
        timestamp: new Date()
      })
    } else if (windowHours < this.thresholds.oplogWindowWarning) {
      this.alerts.push({
        severity: 'WARNING',
        component: 'Oplog',
        message: `Oplog window: ${windowHours.toFixed(2)} hours`,
        value: windowHours,
        threshold: this.thresholds.oplogWindowWarning,
        timestamp: new Date()
      })
    }

    return { windowHours, alerts: this.alerts.filter(a => a.component === 'Oplog') }
  }

  checkPerformance() {
    const status = db.serverStatus()

    // Check 1: Connexions
    const conn = status.connections
    const connUsage = conn.current / (conn.current + conn.available)

    if (connUsage > this.thresholds.connectionUsageCritical) {
      this.alerts.push({
        severity: 'CRITICAL',
        component: 'Connections',
        message: `Connection usage: ${(connUsage * 100).toFixed(2)}%`,
        value: connUsage,
        threshold: this.thresholds.connectionUsageCritical,
        timestamp: new Date()
      })
    } else if (connUsage > this.thresholds.connectionUsageWarning) {
      this.alerts.push({
        severity: 'WARNING',
        component: 'Connections',
        message: `Connection usage: ${(connUsage * 100).toFixed(2)}%`,
        value: connUsage,
        threshold: this.thresholds.connectionUsageWarning,
        timestamp: new Date()
      })
    }

    // Check 2: Cache WiredTiger
    if (status.wiredTiger) {
      const cacheUsage = status.wiredTiger.cache['bytes currently in the cache'] /
                         status.wiredTiger.cache['maximum bytes configured']

      if (cacheUsage > this.thresholds.cacheUsageCritical) {
        this.alerts.push({
          severity: 'CRITICAL',
          component: 'Cache',
          message: `Cache usage: ${(cacheUsage * 100).toFixed(2)}%`,
          value: cacheUsage,
          threshold: this.thresholds.cacheUsageCritical,
          timestamp: new Date()
        })
      } else if (cacheUsage > this.thresholds.cacheUsageWarning) {
        this.alerts.push({
          severity: 'WARNING',
          component: 'Cache',
          message: `Cache usage: ${(cacheUsage * 100).toFixed(2)}%`,
          value: cacheUsage,
          threshold: this.thresholds.cacheUsageWarning,
          timestamp: new Date()
        })
      }
    }
  }

  checkAll() {
    this.alerts = []  // Reset alerts

    this.checkReplicaSetHealth()
    this.checkOplogWindow()
    this.checkPerformance()

    return {
      timestamp: new Date(),
      healthy: this.alerts.filter(a => a.severity === 'CRITICAL').length === 0,
      alertCount: {
        critical: this.alerts.filter(a => a.severity === 'CRITICAL').length,
        warning: this.alerts.filter(a => a.severity === 'WARNING').length,
        total: this.alerts.length
      },
      alerts: this.alerts
    }
  }

  getMetrics() {
    const status = rs.status()
    const serverStatus = db.serverStatus()
    const replInfo = db.getReplicationInfo()

    return {
      timestamp: new Date(),

      // Replica Set
      replicaSet: {
        name: status.set,
        term: status.term,
        members: {
          total: status.members.length,
          healthy: status.members.filter(m => m.health === 1).length,
          primary: status.members.filter(m => m.state === 1).length,
          secondary: status.members.filter(m => m.state === 2).length,
          arbiter: status.members.filter(m => m.state === 7).length,
          down: status.members.filter(m => m.state === 8).length
        }
      },

      // Replication
      replication: {
        lag: {
          max: Math.max(...status.members
            .filter(m => m.state === 2 && m.optimeDate)
            .map(m => (status.date - m.optimeDate) / 1000)),
          members: status.members
            .filter(m => m.state === 2)
            .map(m => ({
              name: m.name,
              lagSeconds: m.optimeDate ? (status.date - m.optimeDate) / 1000 : null
            }))
        },
        oplog: {
          windowHours: replInfo.timeDiff / 3600,
          sizeMB: replInfo.logSizeMB,
          usedMB: replInfo.usedMB
        }
      },

      // Performance
      performance: {
        operations: serverStatus.opcounters,
        connections: serverStatus.connections,
        memory: {
          resident: serverStatus.mem.resident,
          virtual: serverStatus.mem.virtual
        },
        network: {
          bytesIn: serverStatus.network.bytesIn,
          bytesOut: serverStatus.network.bytesOut,
          requests: serverStatus.network.numRequests
        }
      },

      // WiredTiger
      wiredTiger: serverStatus.wiredTiger ? {
        cache: {
          currentBytes: serverStatus.wiredTiger.cache['bytes currently in the cache'],
          maxBytes: serverStatus.wiredTiger.cache['maximum bytes configured'],
          dirtyBytes: serverStatus.wiredTiger.cache['tracked dirty bytes in the cache'],
          usagePercent: (serverStatus.wiredTiger.cache['bytes currently in the cache'] /
                        serverStatus.wiredTiger.cache['maximum bytes configured'] * 100)
        }
      } : null
    }
  }
}

// Utilisation
const monitor = new ReplicaSetMonitor({
  replicationLagWarning: 10,
  replicationLagCritical: 60,
  oplogWindowWarning: 24,
  oplogWindowCritical: 12
})

const result = monitor.checkAll()

if (result.healthy) {
  print('✓ All checks passed')
} else {
  print(`✗ ${result.alertCount.critical} critical alerts, ${result.alertCount.warning} warnings`)
  print('\nAlerts:')
  result.alerts.forEach(alert => {
    print(`[${alert.severity}] ${alert.component}: ${alert.message}`)
  })
}

// Métriques
const metrics = monitor.getMetrics()
printjson(metrics)
```

### Script d'Exportation des Métriques

```javascript
// export-metrics.js - Exporter les métriques vers un format standard

function exportMetricsToPrometheus() {
  const monitor = new ReplicaSetMonitor()
  const metrics = monitor.getMetrics()
  const alerts = monitor.checkAll()

  const prometheusMetrics = []

  // Replica Set state
  prometheusMetrics.push(`# HELP mongodb_rs_members_total Total number of replica set members`)
  prometheusMetrics.push(`# TYPE mongodb_rs_members_total gauge`)
  prometheusMetrics.push(`mongodb_rs_members_total{set="${metrics.replicaSet.name}"} ${metrics.replicaSet.members.total}`)

  prometheusMetrics.push(`# HELP mongodb_rs_members_healthy Number of healthy members`)
  prometheusMetrics.push(`# TYPE mongodb_rs_members_healthy gauge`)
  prometheusMetrics.push(`mongodb_rs_members_healthy{set="${metrics.replicaSet.name}"} ${metrics.replicaSet.members.healthy}`)

  // Replication lag
  prometheusMetrics.push(`# HELP mongodb_rs_replication_lag_seconds Replication lag in seconds`)
  prometheusMetrics.push(`# TYPE mongodb_rs_replication_lag_seconds gauge`)
  metrics.replication.lag.members.forEach(m => {
    if (m.lagSeconds !== null) {
      prometheusMetrics.push(`mongodb_rs_replication_lag_seconds{member="${m.name}"} ${m.lagSeconds}`)
    }
  })

  // Oplog
  prometheusMetrics.push(`# HELP mongodb_rs_oplog_window_hours Oplog window in hours`)
  prometheusMetrics.push(`# TYPE mongodb_rs_oplog_window_hours gauge`)
  prometheusMetrics.push(`mongodb_rs_oplog_window_hours{set="${metrics.replicaSet.name}"} ${metrics.replication.oplog.windowHours}`)

  // Connections
  prometheusMetrics.push(`# HELP mongodb_connections_current Current connections`)
  prometheusMetrics.push(`# TYPE mongodb_connections_current gauge`)
  prometheusMetrics.push(`mongodb_connections_current ${metrics.performance.connections.current}`)

  // Operations
  Object.entries(metrics.performance.operations).forEach(([op, count]) => {
    prometheusMetrics.push(`# HELP mongodb_operations_total{type="${op}"} Total operations`)
    prometheusMetrics.push(`# TYPE mongodb_operations_total counter`)
    prometheusMetrics.push(`mongodb_operations_total{type="${op}"} ${count}`)
  })

  return prometheusMetrics.join('\n')
}

// Exporter
print(exportMetricsToPrometheus())
```

## Alerting

### Définition des Alertes Critiques

```yaml
# alerts.yml - Configuration Prometheus Alertmanager

groups:
  - name: mongodb_replica_set
    interval: 30s
    rules:
      # Alerte 1: Pas de Primary
      - alert: MongoDBNoPrimary
        expr: mongodb_mongod_replset_member_state{state="1"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "No PRIMARY member in replica set {{ $labels.set }}"
          description: "Replica set {{ $labels.set }} has no PRIMARY member for more than 1 minute."

      # Alerte 2: Membre Down
      - alert: MongoDBMemberDown
        expr: mongodb_mongod_replset_member_health == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB member {{ $labels.name }} is DOWN"
          description: "Member {{ $labels.name }} in replica set {{ $labels.set }} is DOWN."

      # Alerte 3: Replication Lag critique
      - alert: MongoDBReplicationLagCritical
        expr: |
          (mongodb_mongod_replset_oplog_head_timestamp -
           mongodb_mongod_replset_my_state_optime_timestamp) > 60
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High replication lag on {{ $labels.instance }}"
          description: "Replication lag is {{ $value }}s on {{ $labels.instance }}."

      # Alerte 4: Replication Lag warning
      - alert: MongoDBReplicationLagWarning
        expr: |
          (mongodb_mongod_replset_oplog_head_timestamp -
           mongodb_mongod_replset_my_state_optime_timestamp) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Moderate replication lag on {{ $labels.instance }}"
          description: "Replication lag is {{ $value }}s on {{ $labels.instance }}."

      # Alerte 5: Oplog window faible
      - alert: MongoDBOplogWindowLow
        expr: |
          ((mongodb_mongod_replset_oplog_head_timestamp -
            mongodb_mongod_replset_oplog_tail_timestamp) / 3600) < 24
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low oplog window on {{ $labels.instance }}"
          description: "Oplog window is {{ $value }} hours on {{ $labels.instance }}."

      # Alerte 6: Utilisation connexions élevée
      - alert: MongoDBHighConnectionUsage
        expr: |
          (mongodb_connections{state="current"} /
           (mongodb_connections{state="current"} + mongodb_connections{state="available"})) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High connection usage on {{ $labels.instance }}"
          description: "Connection usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}."

      # Alerte 7: Cache WiredTiger saturé
      - alert: MongoDBCacheHighUsage
        expr: |
          (mongodb_mongod_wiredtiger_cache_bytes{type="dirty"} /
           mongodb_mongod_wiredtiger_cache_bytes{type="total"}) > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High WiredTiger cache usage on {{ $labels.instance }}"
          description: "Cache usage is {{ $value | humanizePercentage }} on {{ $labels.instance }}."
```

### Configuration Alertmanager

```yaml
# alertmanager.yml

global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

    - match:
        severity: warning
      receiver: 'slack'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@company.com'
        from: 'alertmanager@company.com'
        smarthost: 'smtp.company.com:587'
        auth_username: 'alertmanager'
        auth_password: 'password'

  - name: 'slack'
    slack_configs:
      - channel: '#mongodb-alerts'
        title: 'MongoDB Alert'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
```

## Logging

### Configuration des Logs MongoDB

```yaml
# mongod.conf - Configuration avancée des logs

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc

  # Verbosité par composant
  component:
    accessControl:
      verbosity: 0
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
      verbosity: 0
    replication:
      verbosity: 1      # Augmenter pour réplication
    storage:
      verbosity: 0
      journal:
        verbosity: 0
    write:
      verbosity: 0
```

### Analyse des Logs

```bash
# Rechercher les erreurs
grep -i "error" /var/log/mongodb/mongod.log | tail -20

# Rechercher les warnings
grep -i "warning" /var/log/mongodb/mongod.log | tail -20

# Rechercher les élections
grep -i "election" /var/log/mongodb/mongod.log | tail -20

# Rechercher les rollbacks
grep -i "rollback" /var/log/mongodb/mongod.log | tail -20

# Rechercher les connexions refusées
grep -i "refused" /var/log/mongodb/mongod.log | tail -20

# Analyser les slow queries
grep -i "slow query" /var/log/mongodb/mongod.log | tail -20
```

### Centralisation des Logs (ELK Stack)

```yaml
# filebeat.yml - Configuration Filebeat

filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/mongodb/mongod.log
    fields:
      service: mongodb
      environment: production
      replica_set: rs0

    # Parser les logs MongoDB
    multiline:
      pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}T'
      negate: true
      match: after

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "mongodb-logs-%{+yyyy.MM.dd}"

setup.kibana:
  host: "kibana:5601"

# Processeurs
processors:
  - decode_json_fields:
      fields: ["message"]
      target: "mongodb"

  - add_host_metadata: ~
  - add_cloud_metadata: ~
```

## Bonnes Pratiques de Monitoring

### 1. Niveaux de Monitoring

```
┌─────────────────────────────────────────────┐
│         Pyramide du Monitoring              │
├─────────────────────────────────────────────┤
│                                             │
│  Niveau 1: INFRASTRUCTURE                   │
│  - CPU, RAM, Disque, Réseau                 │
│  - Disponibilité serveurs                   │
│                                             │
│  Niveau 2: MONGODB (SYSTÈME)                │
│  - État Replica Set                         │
│  - Replication lag                          │
│  - Oplog window                             │
│                                             │
│  Niveau 3: MONGODB (PERFORMANCE)            │
│  - Operations/sec                           │
│  - Connexions                               │
│  - Cache WiredTiger                         │
│  - Slow queries                             │
│                                             │
│  Niveau 4: APPLICATION                      │
│  - Latence des requêtes                     │
│  - Taux d'erreur                            │
│  - Throughput métier                        │
│                                             │
└─────────────────────────────────────────────┘
```

### 2. Fréquence de Collecte

```javascript
const monitoringIntervals = {
  // Critique - Vérification fréquente
  replicaSetState: 10,         // 10 secondes
  replicationLag: 10,           // 10 secondes
  primaryPresence: 5,           // 5 secondes

  // Important - Vérification régulière
  connectionUsage: 30,          // 30 secondes
  cacheUsage: 30,               // 30 secondes
  operationsPerSecond: 30,      // 30 secondes

  // Monitoring général
  oplogWindow: 300,             // 5 minutes
  diskUsage: 300,               // 5 minutes
  networkIO: 60,                // 1 minute

  // Analyse
  slowQueries: 60,              // 1 minute
  indexUsage: 3600,             // 1 heure
  collectionStats: 3600         // 1 heure
}
```

### 3. Rétention des Données

```javascript
const retentionPolicies = {
  // Métriques haute fréquence
  realtimeMetrics: {
    resolution: '10s',
    retention: '7 days'
  },

  // Métriques normales
  regularMetrics: {
    resolution: '1m',
    retention: '30 days'
  },

  // Agrégées
  hourlyMetrics: {
    resolution: '1h',
    retention: '1 year'
  },

  dailyMetrics: {
    resolution: '1d',
    retention: '5 years'
  },

  // Logs
  logs: {
    retention: '90 days',
    archiveRetention: '7 years'
  }
}
```

### 4. Dashboard Essentiel

**Tableau de bord minimal pour production** :

```
┌──────────────────────────────────────────────────────────┐
│ MongoDB Replica Set - Production Dashboard               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ [Replica Set Status]    [Primary]    [Health: 3/3 UP]    │
│                                                          │
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│ │ Replication Lag │ │  Oplog Window   │ │ Operations  │  │
│ │     0.5s        │ │    48 hours     │ │  1,234/sec  │  │
│ │  ━━━━━━━━━░     │ │  ━━━━━━━━━━     │ │             │  │
│ └─────────────────┘ └─────────────────┘ └─────────────┘  │
│                                                          │
│ ┌─────────────────┐ ┌─────────────────┐ ┌─────────────┐  │
│ │  Connections    │ │      Cache      │ │   Network   │  │
│ │   450 / 1000    │ │    65% used     │ │  125 MB/s   │  │
│ │  ━━━━━━░░░░     │ │  ━━━━━━━━░░     │ │             │  │
│ └─────────────────┘ └─────────────────┘ └─────────────┘  │
│                                                          │
│ [Recent Alerts] ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │
│ ✓ No critical alerts                                     │
│ ⚠ Warning: mongodb-03 lag 8.2s (5m ago)                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5. Checklist de Monitoring

```javascript
const monitoringChecklist = {
  daily: [
    'Vérifier replication lag',
    'Vérifier état de tous les membres',
    'Vérifier oplog window',
    'Réviser les alertes de la journée',
    'Vérifier les slow queries'
  ],

  weekly: [
    'Analyser les tendances de performance',
    'Vérifier l\'utilisation du disque',
    'Réviser les logs d\'erreurs',
    'Tester les alertes',
    'Vérifier les backups'
  ],

  monthly: [
    'Analyser les patterns de croissance',
    'Réviser les seuils d\'alerting',
    'Valider les runbooks',
    'Test de failover',
    'Audit de sécurité'
  ]
}
```

## Conclusion

Le monitoring d'un Replica Set MongoDB repose sur :

- ✅ **Métriques multi-niveaux** : Infrastructure, système, performance, application
- ✅ **Outils variés** : Commandes natives, mongostat, Prometheus, Grafana
- ✅ **Alerting proactif** : Détection précoce des problèmes
- ✅ **Automation** : Scripts de monitoring et collecte automatique
- ✅ **Visualisation** : Dashboards clairs et actionnables

**Métriques critiques à surveiller en priorité** :
1. État du Replica Set (Primary présent, membres healthy)
2. Replication lag (< 10 secondes en normal)
3. Oplog window (≥ 24 heures recommandé)
4. Connexions (< 80% de la capacité)
5. Cache WiredTiger (< 80% utilisé)

Un système de monitoring bien conçu permet de détecter et résoudre les problèmes avant qu'ils n'impactent les utilisateurs, garantissant ainsi la haute disponibilité et les performances optimales du Replica Set.

⏭️ [Maintenance et opérations courantes](/09-replication/11-maintenance-operations-courantes.md)
