🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.2 Commandes d'administration

## Introduction

Les commandes d'administration MongoDB constituent l'arsenal essentiel pour les SRE et administrateurs système qui doivent superviser, diagnostiquer et maintenir des déploiements en production. Contrairement aux métriques collectées passivement, ces commandes permettent une interaction directe avec le serveur pour obtenir des informations détaillées en temps réel, identifier les problèmes et prendre des décisions éclairées.

Cette section présente les commandes d'administration fondamentales, leur utilisation, leurs permissions requises et les bonnes pratiques pour leur exploitation en production.

## Vue d'ensemble des commandes d'administration

### Architecture des commandes

MongoDB expose ses commandes via plusieurs mécanismes :

```
┌────────────────────────────────────────────────────────────┐
│                    POINTS D'ENTRÉE                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   mongosh    │    │  db.command  │    │  HTTP API    │  │
│  │   (Shell)    │    │   (Driver)   │    │   (Atlas)    │  │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘  │
│         │                   │                   │          │
│         └───────────────────┴───────────────────┘          │
│                             │                              │
│                   ┌─────────▼─────────┐                    │
│                   │  COMMAND PARSER   │                    │
│                   └─────────┬─────────┘                    │
│                             │                              │
│         ┌───────────────────┼───────────────────┐          │
│         │                   │                   │          │
│    ┌────▼────┐       ┌──────▼──────┐      ┌─────▼───┐      │
│    │ Admin   │       │  Database   │      │ Diag.   │      │
│    │Commands │       │  Commands   │      │Commands │      │
│    └─────────┘       └─────────────┘      └─────────┘      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Catégories de commandes

Les commandes d'administration se répartissent en plusieurs catégories selon leur fonction :

#### 1. Commandes de diagnostic et monitoring

| Commande | Scope | Fréquence | Usage principal |
|----------|-------|-----------|-----------------|
| `serverStatus` | Instance | Temps réel | État complet du serveur |
| `dbStats` | Database | Périodique | Statistiques par base |
| `collStats` | Collection | À la demande | Statistiques par collection |
| `currentOp` | Instance | Diagnostic | Opérations en cours |
| `top` | Instance | Diagnostic | Usage CPU par collection |
| `replSetGetStatus` | Replica Set | Temps réel | État de réplication |
| `shardingStatistics` | Cluster | Périodique | Statistiques de sharding |

#### 2. Commandes d'intervention

| Commande | Impact | Utilisation | Précautions |
|----------|--------|-------------|-------------|
| `killOp` | Opération | Terminer une opération | Vérifier dépendances |
| `fsync` | Serveur | Forcer flush disque | Peut bloquer écritures |
| `compact` | Collection | Défragmentation | Downtime possible |
| `reIndex` | Collection | Reconstruction index | Locking intensif |
| `shutdown` | Serveur | Arrêt propre | Planifier maintenance |

#### 3. Commandes de configuration

| Commande | Niveau | Persistance | Redémarrage requis |
|----------|--------|-------------|-------------------|
| `setParameter` | Instance | Runtime ou config | Selon paramètre |
| `setProfilingLevel` | Database | Runtime | Non |
| `setFeatureCompatibilityVersion` | Cluster | Persistant | Non |
| `enableSharding` | Database | Persistant | Non |
| `shardCollection` | Collection | Persistant | Non |

#### 4. Commandes d'information

| Commande | Retour | Usage | Performance |
|----------|--------|-------|-------------|
| `listDatabases` | Liste | Inventaire | Léger |
| `listCollections` | Liste | Inventaire | Léger |
| `listIndexes` | Liste | Audit index | Léger |
| `getCmdLineOpts` | Config | Démarrage | Instantané |
| `buildInfo` | Version | Compatibilité | Instantané |
| `hostInfo` | Système | Diagnostic | Instantané |

---

## Syntaxe et utilisation des commandes

### Méthode 1 : Helpers mongosh

MongoDB Shell fournit des helpers qui encapsulent les commandes :

```javascript
// Via helpers (recommandé pour utilisation interactive)
db.serverStatus()
db.stats()
db.collection.stats()
rs.status()
sh.status()

// Avantages :
// - Syntaxe simplifiée
// - Formatage automatique
// - Validation des paramètres
```

### Méthode 2 : runCommand (Programmatique)

Pour un contrôle fin ou l'automatisation :

```javascript
// Via runCommand
db.runCommand({ serverStatus: 1 })
db.runCommand({ dbStats: 1, scale: 1024 })
db.runCommand({ collStats: "myCollection" })

// Avantages :
// - Contrôle des options
// - Parsing JSON standard
// - Portable entre drivers
```

### Méthode 3 : adminCommand (Commandes admin)

Certaines commandes nécessitent le contexte admin :

```javascript
// Commandes au niveau admin
db.adminCommand({ serverStatus: 1 })
db.adminCommand({ replSetGetStatus: 1 })
db.adminCommand({ listDatabases: 1 })
db.adminCommand({ getCmdLineOpts: 1 })

// Ou via getSiblingDB
db.getSiblingDB("admin").runCommand({ listDatabases: 1 })
```

### Comparaison des syntaxes

```javascript
// Exemple : Obtenir les statistiques du serveur

// Méthode 1 : Helper (plus lisible)
var stats = db.serverStatus()

// Méthode 2 : runCommand (plus flexible)
var stats = db.runCommand({ serverStatus: 1 })

// Méthode 3 : adminCommand (contexte explicite)
var stats = db.adminCommand({ serverStatus: 1 })

// Toutes trois retournent le même résultat
// Choisir selon le contexte :
// - Interactive : Helper
// - Script : runCommand
// - Admin explicite : adminCommand
```

---

## Gestion des permissions

### Rôles et privilèges requis

Les commandes d'administration nécessitent des privilèges spécifiques :

```javascript
// Commandes de lecture (read-only monitoring)
// Rôle minimum : clusterMonitor ou readAnyDatabase

db.adminCommand({ serverStatus: 1 })        // clusterMonitor
db.adminCommand({ replSetGetStatus: 1 })    // clusterMonitor
db.adminCommand({ listDatabases: 1 })       // readAnyDatabase

// Commandes d'administration
// Rôle minimum : dbAdmin, clusterAdmin selon scope

db.adminCommand({ setParameter: 1, ... })   // clusterAdmin
db.runCommand({ killOp: 1, op: 123 })      // clusterAdmin
db.runCommand({ compact: "collection" })    // dbAdmin

// Commandes de configuration critique
// Rôle : root ou permissions spécifiques

db.adminCommand({ shutdown: 1 })            // shutdown action
db.adminCommand({ replSetReconfig: ... })   // replSetReconfig action
```

### Audit des permissions

Vérifier les permissions d'un utilisateur :

```javascript
// Afficher les rôles de l'utilisateur courant
db.runCommand({ connectionStatus: 1, showPrivileges: true })

// Résultat (extrait) :
{
  "authInfo": {
    "authenticatedUsers": [
      { "user": "admin", "db": "admin" }
    ],
    "authenticatedUserRoles": [
      { "role": "clusterMonitor", "db": "admin" },
      { "role": "readAnyDatabase", "db": "admin" }
    ],
    "authenticatedUserPrivileges": [
      {
        "resource": { "cluster": true },
        "actions": [ "serverStatus", "top", "currentOp", ... ]
      }
    ]
  }
}
```

### Créer un utilisateur de monitoring

```javascript
// Utilisateur dédié au monitoring (read-only)
use admin
db.createUser({
  user: "monitoring_user",
  pwd: "secure_password",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }  // Pour accès oplog
  ]
})

// Tester les permissions
db.auth("monitoring_user", "secure_password")
db.serverStatus()  // ✅ Autorisé
db.adminCommand({ shutdown: 1 })  // ❌ Non autorisé
```

---

## Bonnes pratiques d'utilisation

### 1. Impact sur les performances

Certaines commandes peuvent impacter les performances :

```javascript
// Commandes LÉGÈRES (< 10ms, utilisables en production)
db.serverStatus()              // Lecture de métriques en mémoire
db.currentOp()                 // Snapshot des opérations
db.adminCommand({ isMaster: 1 })  // Check rapide de l'état

// Commandes MOYENNES (10-100ms, à espacer)
db.stats()                     // Parcours des métadonnées
db.collection.stats()          // Stats d'une collection
db.collection.getIndexes()     // Liste des index

// Commandes LOURDES (> 100ms, éviter en production)
db.collection.validate()       // Validation complète de collection
db.collection.stats({ indexDetails: true })  // Détails de tous les index
db.currentOp({ $all: true })   // Toutes les opérations système incluses
```

**Recommandation** : Établir une baseline de latence pour chaque commande dans votre environnement.

### 2. Fréquence de collecte

```yaml
Intervalles recommandés:
  Temps réel (1-10s):
    - serverStatus (métriques de base)
    - currentOp (si investigation en cours)
    - replSetGetStatus (monitoring continu)

  Périodique (1-5min):
    - dbStats (toutes les bases)
    - top (usage par collection)
    - shardingStatistics

  À la demande:
    - collStats (analyse ciblée)
    - explain() (optimisation queries)
    - validate() (maintenance)

  Rare (journalier/hebdomadaire):
    - compact (défragmentation)
    - reIndex (reconstruction index)
```

### 3. Stockage et rétention des résultats

```javascript
// Exemple : Collecte automatisée de serverStatus

function collectServerStats() {
  var stats = db.serverStatus()

  // Extraire uniquement les métriques nécessaires
  var metrics = {
    timestamp: new Date(),
    host: stats.host,
    version: stats.version,
    uptime: stats.uptime,
    connections: stats.connections,
    opcounters: stats.opcounters,
    memory: stats.mem,
    network: stats.network,
    opcountersRepl: stats.opcountersRepl || null
  }

  // Stocker dans une collection de monitoring
  db.getSiblingDB("monitoring").serverStats.insertOne(metrics)

  // Ou exporter vers système externe (Prometheus, Datadog, etc.)
  // exportToPrometheus(metrics)
}

// Exécuter toutes les 60 secondes
setInterval(collectServerStats, 60000)
```

**Rétention recommandée** :
- Métriques haute fréquence (1s) : 24-48 heures
- Métriques moyennes (1min) : 7-30 jours
- Métriques agrégées (1h) : 90-365 jours

### 4. Gestion des erreurs

```javascript
// Enveloppe robuste pour commandes d'administration

function runAdminCommandSafe(command, options = {}) {
  try {
    var startTime = Date.now()
    var result = db.adminCommand(command)
    var duration = Date.now() - startTime

    if (!result.ok) {
      console.error(`Command failed: ${JSON.stringify(command)}`)
      console.error(`Error: ${result.errmsg}`)
      console.error(`Code: ${result.code}`)
      return { success: false, error: result.errmsg, code: result.code }
    }

    if (options.logDuration && duration > (options.slowThreshold || 100)) {
      console.warn(`Slow command (${duration}ms): ${JSON.stringify(command)}`)
    }

    return { success: true, data: result, duration: duration }

  } catch (e) {
    console.error(`Exception executing command: ${e.message}`)
    return { success: false, error: e.message, exception: true }
  }
}

// Utilisation
var result = runAdminCommandSafe(
  { serverStatus: 1 },
  { logDuration: true, slowThreshold: 50 }
)

if (result.success) {
  // Traiter result.data
} else {
  // Gérer l'erreur
  if (result.code === 13) {
    console.log("Permission denied - check user privileges")
  }
}
```

### 5. Automatisation et scripting

```bash
#!/bin/bash
# Script de diagnostic rapide MongoDB

MONGO_HOST="localhost:27017"
MONGO_USER="admin"
MONGO_PASS="password"

echo "=== MongoDB Diagnostic Report ==="
echo "Date: $(date)"
echo "Host: $MONGO_HOST"
echo ""

# 1. Vérifier connectivité
echo "1. Connection status:"
mongosh "$MONGO_HOST" -u "$MONGO_USER" -p "$MONGO_PASS" --quiet --eval "
  try {
    db.adminCommand({ ping: 1 })
    print('✅ Connected')
  } catch(e) {
    print('❌ Connection failed: ' + e.message)
  }
"

# 2. Version et uptime
echo -e "\n2. Server info:"
mongosh "$MONGO_HOST" -u "$MONGO_USER" -p "$MONGO_PASS" --quiet --eval "
  var status = db.serverStatus()
  print('Version: ' + status.version)
  print('Uptime: ' + Math.floor(status.uptime / 3600) + ' hours')
"

# 3. Replica Set status
echo -e "\n3. Replica Set status:"
mongosh "$MONGO_HOST" -u "$MONGO_USER" -p "$MONGO_PASS" --quiet --eval "
  try {
    var rsStatus = rs.status()
    print('Set: ' + rsStatus.set)
    rsStatus.members.forEach(m => {
      var icon = m.health === 1 ? '✅' : '❌'
      print(icon + ' ' + m.name + ' - ' + m.stateStr)
    })
  } catch(e) {
    print('Not a replica set or no permission')
  }
"

# 4. Connexions
echo -e "\n4. Connections:"
mongosh "$MONGO_HOST" -u "$MONGO_USER" -p "$MONGO_PASS" --quiet --eval "
  var conn = db.serverStatus().connections
  print('Current: ' + conn.current)
  print('Available: ' + conn.available)
  var pct = (conn.current / (conn.current + conn.available) * 100).toFixed(2)
  print('Usage: ' + pct + '%')
"

# 5. Opérations lentes
echo -e "\n5. Current slow operations (> 5s):"
mongosh "$MONGO_HOST" -u "$MONGO_USER" -p "$MONGO_PASS" --quiet --eval "
  db.currentOp({
    active: true,
    secs_running: { \$gt: 5 }
  }).inprog.forEach(op => {
    print(op.op + ' on ' + op.ns + ' - ' + op.secs_running + 's')
  })
" | head -10

echo -e "\n=== End of Report ==="
```

---

## Commandes par cas d'usage

### Diagnostic de performance

```javascript
// Workflow de diagnostic complet

// 1. Vue d'ensemble de la charge
db.serverStatus().opcounters       // Throughput global
db.serverStatus().connections      // Charge connexions
db.currentOp({ active: true })     // Opérations actives

// 2. Identifier les goulots d'étranglement
db.serverStatus().globalLock       // Contentions
db.serverStatus().wiredTiger.cache // Pression mémoire
db.serverStatus().network          // Saturation réseau

// 3. Identifier les queries problématiques
db.currentOp({
  active: true,
  secs_running: { $gt: 5 },
  $or: [
    { "locks.Global": "w" },       // Write locks
    { "planSummary": "COLLSCAN" }  // Collection scans
  ]
})

// 4. Analyser par collection
db.runCommand({ top: 1 })          // Usage CPU par collection
```

### Monitoring de réplication

```javascript
// Vérification santé Replica Set

// 1. État global
rs.status()

// 2. Lag de réplication
rs.printSlaveReplicationInfo()

// 3. Configuration du Replica Set
rs.conf()

// 4. Oplog window
db.printReplicationInfo()

// 5. Vérifier si un membre est en sync
rs.status().members.forEach(m => {
  if (m.stateStr === "RECOVERING") {
    print(`⚠️ ${m.name} is recovering`)
    print(`  Last heartbeat: ${m.lastHeartbeat}`)
    print(`  Optime: ${m.optimeDate}`)
  }
})
```

### Audit de sécurité

```javascript
// Commandes pour audit de sécurité

// 1. Lister tous les utilisateurs
db.getSiblingDB("admin").system.users.find({}, {
  user: 1,
  db: 1,
  roles: 1
})

// 2. Vérifier les connexions actives
db.currentOp(true).inprog.forEach(op => {
  if (op.client) {
    print(`User: ${op.effectiveUsers ? op.effectiveUsers[0].user : 'N/A'}`)
    print(`Client: ${op.client}`)
    print(`Operation: ${op.op} on ${op.ns}`)
    print("---")
  }
})

// 3. Audit des commandes récentes (si audit activé)
db.getSiblingDB("admin").system.audit.find({
  "param.command": { $exists: true }
}).sort({ ts: -1 }).limit(20)

// 4. Vérifier la configuration de sécurité
db.adminCommand({ getCmdLineOpts: 1 }).parsed.security
```

### Maintenance préventive

```javascript
// Checklist de maintenance hebdomadaire

function weeklyMaintenanceCheck() {
  print("=== Weekly Maintenance Check ===\n")

  // 1. Espace disque
  print("1. Disk Space:")
  db.adminCommand({ listDatabases: 1 }).databases.forEach(dbInfo => {
    var sizeGB = (dbInfo.sizeOnDisk / (1024*1024*1024)).toFixed(2)
    print(`  ${dbInfo.name}: ${sizeGB} GB`)
  })

  // 2. Fragmentation par collection
  print("\n2. Fragmentation Check:")
  db.getCollectionNames().forEach(collName => {
    var stats = db[collName].stats()
    var fragmentation = (stats.storageSize / stats.size).toFixed(2)
    if (fragmentation > 1.5) {
      print(`  ⚠️ ${collName}: ${fragmentation}x (consider compact)`)
    }
  })

  // 3. Index non utilisés
  print("\n3. Unused Indexes:")
  db.getCollectionNames().forEach(collName => {
    var indexStats = db[collName].aggregate([
      { $indexStats: {} }
    ]).toArray()

    indexStats.forEach(idx => {
      if (idx.accesses.ops === 0 && idx.name !== "_id_") {
        print(`  ⚠️ ${collName}.${idx.name}: Never used`)
      }
    })
  })

  // 4. Oplog window
  print("\n4. Oplog Window:")
  db.printReplicationInfo()

  print("\n=== End of Check ===")
}

// Exécuter
weeklyMaintenanceCheck()
```

---

## Format de retour et parsing

### Structure de réponse standard

Toutes les commandes MongoDB retournent un document avec au minimum :

```javascript
{
  "ok": 1,           // 1 = succès, 0 = échec
  // ... données spécifiques à la commande
}

// En cas d'erreur :
{
  "ok": 0,
  "errmsg": "auth failed",
  "code": 18,
  "codeName": "AuthenticationFailed"
}
```

### Parsing sécurisé

```javascript
function parseCommandResult(result, requiredFields = []) {
  // Vérifier le succès
  if (!result.ok) {
    throw new Error(`Command failed: ${result.errmsg} (code: ${result.code})`)
  }

  // Vérifier les champs requis
  for (let field of requiredFields) {
    if (!(field in result)) {
      throw new Error(`Missing required field: ${field}`)
    }
  }

  return result
}

// Utilisation
try {
  var result = db.serverStatus()
  var parsed = parseCommandResult(result, ['connections', 'opcounters'])

  // Extraction sécurisée avec valeurs par défaut
  var connections = parsed.connections || { current: 0, available: 0 }
  var opcounters = parsed.opcounters || {}

} catch (e) {
  console.error(`Error: ${e.message}`)
}
```

### Extraction de métriques spécifiques

```javascript
// Extraire uniquement les données nécessaires

function extractKeyMetrics(serverStatus) {
  return {
    timestamp: new Date(),

    // Connexions
    connections: {
      current: serverStatus.connections?.current || 0,
      available: serverStatus.connections?.available || 0,
      totalCreated: serverStatus.connections?.totalCreated || 0
    },

    // Opérations
    operations: {
      insert: serverStatus.opcounters?.insert || 0,
      query: serverStatus.opcounters?.query || 0,
      update: serverStatus.opcounters?.update || 0,
      delete: serverStatus.opcounters?.delete || 0
    },

    // Mémoire
    memory: {
      resident: serverStatus.mem?.resident || 0,
      virtual: serverStatus.mem?.virtual || 0
    },

    // Cache WiredTiger
    cache: {
      bytesInCache: serverStatus.wiredTiger?.cache?.["bytes currently in the cache"] || 0,
      maxBytes: serverStatus.wiredTiger?.cache?.["maximum bytes configured"] || 0
    },

    // Réseau
    network: {
      bytesIn: serverStatus.network?.bytesIn || 0,
      bytesOut: serverStatus.network?.bytesOut || 0,
      numRequests: serverStatus.network?.numRequests || 0
    }
  }
}

// Utilisation
var fullStats = db.serverStatus()
var keyMetrics = extractKeyMetrics(fullStats)
print(JSON.stringify(keyMetrics, null, 2))
```

---

## Commandes dépréciées et alternatives

### Commandes obsolètes

Certaines commandes sont dépréciées et doivent être remplacées :

```javascript
// ❌ DÉPRÉCIÉ : group (supprimé en MongoDB 4.2)
db.collection.group({
  key: { category: 1 },
  reduce: function(curr, result) { result.total += curr.amount },
  initial: { total: 0 }
})

// ✅ ALTERNATIVE : aggregate avec $group
db.collection.aggregate([
  { $group: {
    _id: "$category",
    total: { $sum: "$amount" }
  }}
])

// ❌ DÉPRÉCIÉ : mapReduce (déprécié en 5.0)
db.collection.mapReduce(
  function() { emit(this.category, this.amount) },
  function(key, values) { return Array.sum(values) },
  { out: "results" }
)

// ✅ ALTERNATIVE : aggregate pipeline
db.collection.aggregate([
  { $group: { _id: "$category", total: { $sum: "$amount" }}},
  { $out: "results" }
])

// ❌ DÉPRÉCIÉ : eval (supprimé en 4.2)
db.eval("function() { return db.collection.count() }")

// ✅ ALTERNATIVE : utiliser les commandes natives
db.collection.countDocuments()
```

### Évolution des commandes

```javascript
// Anciennes versions vs versions récentes

// AVANT (< 4.0) : isMaster
db.adminCommand({ isMaster: 1 })

// MAINTENANT (>= 4.4) : hello (isMaster toujours supporté mais hello préféré)
db.adminCommand({ hello: 1 })

// AVANT : count() (peut être imprécis)
db.collection.count({ status: "active" })

// MAINTENANT : countDocuments() (précis mais plus lent)
db.collection.countDocuments({ status: "active" })

// Ou estimatedDocumentCount() (rapide mais approximatif)
db.collection.estimatedDocumentCount()
```

---

## Intégration avec outils de monitoring

### Export vers Prometheus

```javascript
// Formatter les métriques pour Prometheus

function formatPrometheusMetrics(serverStatus) {
  var metrics = []
  var timestamp = Date.now()

  // Connexions
  metrics.push(`mongodb_connections_current ${serverStatus.connections.current} ${timestamp}`)
  metrics.push(`mongodb_connections_available ${serverStatus.connections.available} ${timestamp}`)

  // Opérations
  for (let op in serverStatus.opcounters) {
    metrics.push(`mongodb_opcounters_${op} ${serverStatus.opcounters[op]} ${timestamp}`)
  }

  // Mémoire
  metrics.push(`mongodb_memory_resident_mb ${serverStatus.mem.resident} ${timestamp}`)

  // Cache WiredTiger
  var cacheUsed = serverStatus.wiredTiger.cache["bytes currently in the cache"]
  var cacheMax = serverStatus.wiredTiger.cache["maximum bytes configured"]
  metrics.push(`mongodb_wiredtiger_cache_used_bytes ${cacheUsed} ${timestamp}`)
  metrics.push(`mongodb_wiredtiger_cache_max_bytes ${cacheMax} ${timestamp}`)

  return metrics.join("\n")
}

// Utilisation
var stats = db.serverStatus()
var promMetrics = formatPrometheusMetrics(stats)
print(promMetrics)
```

### Export vers format JSON structuré

```javascript
// Format pour ingestion par systèmes de monitoring

function exportMonitoringData() {
  var status = db.serverStatus()
  var rsStatus = null

  try {
    rsStatus = rs.status()
  } catch (e) {
    // Pas un replica set
  }

  return {
    metadata: {
      timestamp: new Date().toISOString(),
      host: status.host,
      version: status.version,
      process: status.process
    },

    metrics: {
      connections: status.connections,
      operations: status.opcounters,
      memory: status.mem,
      network: status.network,
      locks: status.locks,
      cache: {
        wiredTiger: status.wiredTiger?.cache
      }
    },

    replication: rsStatus ? {
      set: rsStatus.set,
      members: rsStatus.members.map(m => ({
        name: m.name,
        state: m.stateStr,
        health: m.health,
        uptime: m.uptime,
        optimeDate: m.optimeDate
      }))
    } : null,

    health: {
      ok: status.ok === 1,
      uptime: status.uptime
    }
  }
}

// Export
print(JSON.stringify(exportMonitoringData(), null, 2))
```

---

## Vue d'ensemble des commandes détaillées

Les sections suivantes approfondissent chaque commande majeure d'administration :

### 13.2.1 - serverStatus

La commande la plus importante pour le monitoring, retournant des centaines de métriques sur :
- État du serveur et processus
- Compteurs d'opérations (opcounters)
- Connexions
- Mémoire et cache WiredTiger
- Réseau
- Réplication
- Locks et contentions
- Métadonnées de stockage

**Usage typique** : Collecte toutes les 10-60 secondes pour monitoring continu.

### 13.2.2 - dbStats

Statistiques détaillées par base de données :
- Nombre de collections
- Taille des données
- Taille des index
- Nombre de documents
- Espace disque alloué

**Usage typique** : Collecte périodique (5-15 minutes) pour tracking de croissance.

### 13.2.3 - collStats

Métriques au niveau collection :
- Nombre de documents
- Taille moyenne des documents
- Statistiques d'index
- Fragmentation
- Détails WiredTiger

**Usage typique** : Diagnostic ciblé, audit de performance.

### 13.2.4 - currentOp

Opérations en cours d'exécution :
- Type d'opération (query, insert, update, etc.)
- Collection cible
- Durée d'exécution
- Plan d'exécution
- Locks tenus ou attendus

**Usage typique** : Diagnostic temps réel de problèmes de performance.

### 13.2.5 - killOp

Terminer une opération spécifique :
- Arrêt d'une query longue
- Libération de locks
- Intervention d'urgence

**Usage typique** : Intervention manuelle en cas de problème critique.

---

## Checklist de commandes essentielles

### Pour le monitoring quotidien

```javascript
// Script de monitoring rapide (< 1 minute)

print("=== Daily MongoDB Health Check ===\n")

// 1. Connexion et version
print("1. Server Info:")
var status = db.serverStatus()
print(`   Host: ${status.host}`)
print(`   Version: ${status.version}`)
print(`   Uptime: ${Math.floor(status.uptime / 3600)} hours`)

// 2. Connexions
print("\n2. Connections:")
print(`   Current: ${status.connections.current}`)
print(`   Available: ${status.connections.available}`)
var connPct = (status.connections.current /
  (status.connections.current + status.connections.available) * 100).toFixed(2)
print(`   Usage: ${connPct}%`)

// 3. Opérations
print("\n3. Operations (last minute):")
// Nécessite 2 collectes espacées de 60s
// Simplifié ici pour l'exemple

// 4. Mémoire
print("\n4. Memory:")
print(`   Resident: ${status.mem.resident} MB`)
var cacheUsage = (status.wiredTiger.cache["bytes currently in the cache"] /
  status.wiredTiger.cache["maximum bytes configured"] * 100).toFixed(2)
print(`   Cache usage: ${cacheUsage}%`)

// 5. Réplication
print("\n5. Replication:")
try {
  var rsStatus = rs.status()
  print(`   Set: ${rsStatus.set}`)
  rsStatus.members.forEach(m => {
    var icon = m.health === 1 ? "✅" : "❌"
    print(`   ${icon} ${m.name}: ${m.stateStr}`)
  })
} catch(e) {
  print("   Not a replica set")
}

// 6. Alertes
print("\n6. Alerts:")
var alerts = []

if (connPct > 80) alerts.push("⚠️ Connection pool > 80%")
if (cacheUsage > 90) alerts.push("⚠️ Cache usage > 90%")

if (alerts.length === 0) {
  print("   ✅ No alerts")
} else {
  alerts.forEach(a => print(`   ${a}`))
}

print("\n=== End of Check ===")
```

### Pour le troubleshooting

```javascript
// Script d'investigation d'un problème de performance

print("=== Performance Troubleshooting ===\n")

// 1. Opérations lentes en cours
print("1. Slow operations (> 5s):")
var slowOps = db.currentOp({
  active: true,
  secs_running: { $gt: 5 }
}).inprog

if (slowOps.length === 0) {
  print("   ✅ No slow operations")
} else {
  slowOps.slice(0, 5).forEach(op => {
    print(`   ${op.op} on ${op.ns} - ${op.secs_running}s`)
    print(`   Plan: ${op.planSummary || 'N/A'}`)
    print(`   Docs examined: ${op.docsExamined || 'N/A'}`)
  })
}

// 2. Locks en attente
print("\n2. Operations waiting for locks:")
var waitingOps = db.currentOp({
  waitingForLock: true
}).inprog

if (waitingOps.length === 0) {
  print("   ✅ No lock contention")
} else {
  print(`   ⚠️ ${waitingOps.length} operations waiting`)
}

// 3. Cache pressure
print("\n3. Cache pressure:")
var cache = db.serverStatus().wiredTiger.cache
var evictions = cache["pages evicted by application threads"]
print(`   Evictions: ${evictions}`)
if (evictions > 10000) {
  print("   ⚠️ High eviction rate - consider more RAM")
}

// 4. Top collections par usage
print("\n4. Top collections by time:")
var topOutput = db.adminCommand({ top: 1 })
// Parser et afficher top 5
// (parsing complexe omis pour concision)

print("\n=== End of Troubleshooting ===")
```

---

## Résumé

Les commandes d'administration MongoDB constituent un outil puissant pour :

✅ **Monitoring temps réel** : Visibilité instantanée sur l'état du système

✅ **Diagnostic proactif** : Identification précoce des problèmes

✅ **Intervention ciblée** : Actions précises sur opérations ou ressources

✅ **Automatisation** : Intégration dans pipelines de monitoring

✅ **Audit et conformité** : Traçabilité des opérations et accès

**Principes clés** :
- Toujours vérifier les permissions avant l'exécution
- Comprendre l'impact performance de chaque commande
- Automatiser la collecte pour tendances historiques
- Combiner plusieurs commandes pour diagnostic complet
- Documenter les seuils et procédures d'intervention

---

## Prochaines sections

Les sous-sections suivantes détaillent chaque commande majeure :

- **13.2.1** serverStatus - Métrique exhaustive du serveur
- **13.2.2** dbStats - Statistiques par base de données
- **13.2.3** collStats - Statistiques par collection
- **13.2.4** currentOp - Opérations en cours
- **13.2.5** killOp - Terminer des opérations

Chaque section fournit :
- Syntaxe complète et options
- Structure détaillée du retour
- Cas d'usage pratiques
- Exemples d'analyse
- Alertes recommandées

---


⏭️ [serverStatus](/13-monitoring-administration/02.1-serverstatus.md)
