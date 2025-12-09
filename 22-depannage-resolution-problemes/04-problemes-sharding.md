🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.4 Problèmes de Sharding

## Vue d'ensemble

Le sharding MongoDB permet la distribution horizontale des données, mais introduit une complexité supplémentaire qui peut générer des problèmes spécifiques. Cette section fournit des méthodologies complètes pour diagnostiquer et résoudre les problèmes liés aux clusters shardés.

---

## Table des Matières

1. [Balancing Bloqué](#1-balancing-bloqu%C3%A9)
2. [Jumbo Chunks](#2-jumbo-chunks)
3. [Requêtes Scatter-Gather](#3-requ%C3%AAtes-scatter-gather)
4. [Hotspots sur Shards](#4-hotspots-sur-shards)
5. [Migration de Chunks Échouée](#5-migration-de-chunks-%C3%A9chou%C3%A9e)
6. [Problèmes de Config Servers](#6-probl%C3%A8mes-de-config-servers)
7. [Problèmes de Mongos](#7-probl%C3%A8mes-de-mongos)
8. [Shard Key Inadéquate](#8-shard-key-inad%C3%A9quate)

---

## 1. Balancing Bloqué

### Symptômes

```
Uneven data distribution across shards
Balancer not running
"Migrations paused" messages
Some shards full while others empty
Performance degradation on specific shards
```

### Causes Possibles

- Balancer désactivé
- Fenêtre de balancing restrictive
- Jumbo chunks bloquant le balancer
- Migrations échouées répétées
- Locks empêchant les migrations
- Ressources insuffisantes sur les shards

---

### Diagnostic Pas à Pas

#### Étape 1 : Vérifier l'État du Balancer

```javascript
// Se connecter à mongos
mongosh --host mongos:27017

// Vérifier l'état du balancer
sh.getBalancerState()
// true = activé, false = désactivé

// État détaillé du balancer
sh.status()

// Informations spécifiques sur le balancer
db.getSiblingDB("config").settings.findOne({_id: "balancer"})
// Sortie :
// {
//   _id: "balancer",
//   mode: "full",  // ou "off"
//   stopped: false
// }

// Vérifier si le balancer est en cours d'exécution
sh.isBalancerRunning()

// Voir les migrations récentes
db.getSiblingDB("config").changelog.find({
  what: {$in: ["moveChunk.start", "moveChunk.commit", "moveChunk.error"]}
}).sort({time: -1}).limit(20)
```

#### Étape 2 : Analyser la Distribution des Chunks

```javascript
// Distribution globale
sh.status()

// Distribution détaillée par collection
db.getSiblingDB("config").collections.find().forEach(coll => {
  print("\n=== " + coll._id + " ===")

  // Compter les chunks par shard
  db.getSiblingDB("config").chunks.aggregate([
    {$match: {ns: coll._id}},
    {$group: {
      _id: "$shard",
      count: {$sum: 1}
    }},
    {$sort: {count: -1}}
  ]).forEach(printjson)
})

// Voir la taille des données par shard
db.getSiblingDB("config").shards.find().forEach(shard => {
  print("\n=== " + shard._id + " ===")
  var conn = new Mongo(shard.host)
  var stats = conn.getDB("mydb").stats()
  print("Data size: " + (stats.dataSize / 1024 / 1024 / 1024).toFixed(2) + " GB")
  print("Storage size: " + (stats.storageSize / 1024 / 1024 / 1024).toFixed(2) + " GB")
})
```

#### Étape 3 : Identifier les Chunks Problématiques

```javascript
// Trouver les jumbo chunks
db.getSiblingDB("config").chunks.find({
  jumbo: true
}).forEach(printjson)

// Chunks les plus volumineux (estimation par range)
db.getSiblingDB("config").chunks.aggregate([
  {$project: {
    ns: 1,
    shard: 1,
    min: 1,
    max: 1,
    jumbo: 1,
    // Estimation grossière de la taille par la différence de clés
    rangeSize: {$subtract: ["$max", "$min"]}
  }},
  {$sort: {rangeSize: -1}},
  {$limit: 20}
])
```

#### Étape 4 : Vérifier les Locks et Opérations en Cours

```javascript
// Vérifier les locks de balancing
db.getSiblingDB("config").locks.find()

// Voir les migrations actives
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.start",
  time: {$gt: new Date(Date.now() - 3600000)}  // Dernière heure
}).count()

// Vérifier si des migrations sont bloquées
db.currentOp({
  desc: {$regex: /^migrateThread/}
})
```

#### Étape 5 : Analyser les Logs

```bash
# Logs du balancer (sur config server)
grep -i "balancer\|moveChunk" /var/log/mongodb/mongod.log | tail -100

# Messages clés à rechercher :
# - "balancer is disabled"
# - "no chunks need to be moved"
# - "migration failed"
# - "balancer window"
# - "jumbo chunk"
```

---

### Résolution Pas à Pas

#### Solution 1 : Activer le Balancer

```javascript
// Activer le balancer
sh.startBalancer()

// Vérifier
sh.getBalancerState()
// true

// Alternative : Modifier la configuration
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {stopped: false}},
  {upsert: true}
)

// Vérifier que le balancer démarre
sh.isBalancerRunning()
// Attendre quelques secondes et vérifier à nouveau

// Voir l'historique récent
db.getSiblingDB("config").changelog.find({
  what: {$in: ["balancer.start", "balancer.round"]}
}).sort({time: -1}).limit(5)
```

#### Solution 2 : Ajuster la Fenêtre de Balancing

```javascript
// Vérifier la fenêtre actuelle
db.getSiblingDB("config").settings.findOne({_id: "balancer"})

// Définir une fenêtre de balancing (heures creuses)
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {
    activeWindow: {
      start: "01:00",  // 1h du matin
      stop: "06:00"    // 6h du matin
    }
  }},
  {upsert: true}
)

// Supprimer la fenêtre (balancing 24/7)
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$unset: {activeWindow: ""}}
)

// Pour permettre le balancing immédiat (production avec précaution)
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {mode: "full"}},
  {upsert: true}
)
```

#### Solution 3 : Forcer le Balancing

```javascript
// Déplacer un chunk manuellement
sh.moveChunk(
  "mydb.mycollection",
  {shardKey: "valueInChunk"},
  "shard0001"  // Destination shard
)

// Exemple concret
sh.moveChunk(
  "mydb.orders",
  {customerId: 12345},
  "shard0002"
)

// Vérifier la progression
db.currentOp({
  desc: {$regex: /^migrateThread/}
})

// Diviser un gros chunk manuellement avant de le déplacer
sh.splitAt("mydb.orders", {customerId: 50000})
sh.splitAt("mydb.orders", {customerId: 75000})
```

#### Solution 4 : Nettoyer les Locks Orphelins

```javascript
// ATTENTION : À faire uniquement si aucune migration n'est en cours !

// Vérifier d'abord qu'aucune migration n'est active
db.currentOp({desc: {$regex: /^migrateThread/}})

// Si vide, nettoyer les locks
db.getSiblingDB("config").locks.find()

// Supprimer les locks obsolètes (avec prudence)
db.getSiblingDB("config").locks.remove({
  state: 0,  // Lock non actif
  ts: {$lt: new Date(Date.now() - 900000)}  // Older than 15 min
})

// Redémarrer le balancer
sh.stopBalancer()
sh.startBalancer()
```

#### Solution 5 : Optimiser les Paramètres du Balancer

```javascript
// Augmenter la taille des chunks (si beaucoup de petits chunks)
db.getSiblingDB("config").settings.updateOne(
  {_id: "chunksize"},
  {$set: {value: 128}},  // 128 MB (défaut: 64 MB)
  {upsert: true}
)

// Paramètres avancés du balancer
db.adminCommand({
  setParameter: 1,
  // Nombre de chunks à déplacer en parallèle
  migrateCloneInsertionBatchSize: 100,
  // Délai entre les migrations
  migrateCloneInsertionBatchDelayMS: 0
})

// Configuration pour environnements haute performance
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {
    // Permettre migrations pendant les pics (avec précaution)
    attemptToBalanceJumboChunks: true
  }},
  {upsert: true}
)
```

---

## 2. Jumbo Chunks

### Symptômes

```
"Jumbo chunk" warnings in logs
Uneven data distribution
Balancer unable to move chunks
Single shard overloaded
Performance issues on specific shard
```

### Causes Possibles

- Shard key avec faible cardinalité
- Documents volumineux
- Croissance rapide d'une plage de valeurs
- Split automatique désactivé
- Chunk size trop grand

---

### Diagnostic Pas à Pas

#### Étape 1 : Identifier les Jumbo Chunks

```javascript
// Trouver tous les jumbo chunks
db.getSiblingDB("config").chunks.find({
  jumbo: true
}).forEach(chunk => {
  printjson({
    ns: chunk.ns,
    shard: chunk.shard,
    min: chunk.min,
    max: chunk.max
  })
})

// Compter par collection
db.getSiblingDB("config").chunks.aggregate([
  {$match: {jumbo: true}},
  {$group: {
    _id: "$ns",
    count: {$sum: 1}
  }},
  {$sort: {count: -1}}
])

// Estimer la taille des jumbo chunks
db.getSiblingDB("config").chunks.find({jumbo: true}).forEach(chunk => {
  // Se connecter au shard
  var shardConn = new Mongo(
    db.getSiblingDB("config").shards.findOne({_id: chunk.shard}).host
  )

  // Compter les documents dans le chunk
  var dbName = chunk.ns.split('.')[0]
  var collName = chunk.ns.split('.').slice(1).join('.')

  var count = shardConn.getDB(dbName)[collName].count({
    $and: [
      chunk.min,
      {$not: chunk.max}  // Chunk range
    ]
  })

  print(`${chunk.ns} on ${chunk.shard}: ~${count} documents`)
})
```

#### Étape 2 : Analyser la Shard Key

```javascript
// Voir la shard key de la collection
db.getSiblingDB("config").collections.findOne({
  _id: "mydb.mycollection"
})

// Analyser la distribution des valeurs de shard key
db.mycollection.aggregate([
  {$group: {
    _id: "$shardKeyField",
    count: {$sum: 1}
  }},
  {$sort: {count: -1}},
  {$limit: 20}
])

// Vérifier la cardinalité
db.mycollection.aggregate([
  {$group: {_id: "$shardKeyField"}},
  {$count: "uniqueValues"}
])
```

#### Étape 3 : Analyser la Croissance

```javascript
// Voir la croissance récente par chunk
db.getSiblingDB("config").changelog.aggregate([
  {$match: {
    what: "split",
    time: {$gt: new Date(Date.now() - 86400000)}  // 24h
  }},
  {$group: {
    _id: "$ns",
    splits: {$sum: 1}
  }},
  {$sort: {splits: -1}}
])

// Identifier les plages qui ne se divisent pas
db.getSiblingDB("config").chunks.find({
  ns: "mydb.orders",
  jumbo: true
}).forEach(chunk => {
  print(`Range: ${tojson(chunk.min)} to ${tojson(chunk.max)}`)
})
```

---

### Résolution Pas à Pas

#### Solution 1 : Diviser Manuellement les Jumbo Chunks

```javascript
// Stratégie 1 : Split au milieu
// Identifier un chunk jumbo
var chunk = db.getSiblingDB("config").chunks.findOne({
  ns: "mydb.orders",
  jumbo: true
})

// Calculer une valeur de split (au milieu de la plage)
// Pour une shard key numérique
var midPoint = {
  customerId: (chunk.min.customerId + chunk.max.customerId) / 2
}

// Diviser
sh.splitAt("mydb.orders", midPoint)

// Stratégie 2 : Splits multiples
// Diviser en plusieurs chunks plus petits
sh.splitAt("mydb.orders", {customerId: 10000})
sh.splitAt("mydb.orders", {customerId: 20000})
sh.splitAt("mydb.orders", {customerId: 30000})
sh.splitAt("mydb.orders", {customerId: 40000})

// Stratégie 3 : Split automatique par find
// Laisser MongoDB trouver le meilleur point de division
sh.splitFind("mydb.orders", {customerId: 25000})

// Vérifier les nouveaux chunks
db.getSiblingDB("config").chunks.find({
  ns: "mydb.orders",
  "min.customerId": {$gte: chunk.min.customerId},
  "max.customerId": {$lte: chunk.max.customerId}
}).count()
```

#### Solution 2 : Refactorer la Shard Key (Migration)

```javascript
// ⚠️ IMPORTANT : Nécessite migration de données

// Approche 1 : Shard key composée (ajouter de la cardinalité)
// Ancienne shard key : {customerId: 1}
// Nouvelle shard key : {customerId: 1, orderDate: 1}

// 1. Créer une nouvelle collection avec meilleure shard key
db.adminCommand({
  shardCollection: "mydb.orders_new",
  key: {customerId: 1, orderDate: 1}
})

// 2. Migrer les données (en batch)
var cursor = db.orders.find().batchSize(1000)
var batch = []

cursor.forEach(doc => {
  batch.push(doc)

  if (batch.length >= 1000) {
    db.orders_new.insertMany(batch, {ordered: false})
    batch = []
  }
})

if (batch.length > 0) {
  db.orders_new.insertMany(batch, {ordered: false})
}

// 3. Vérifier l'intégrité
db.orders.count()
db.orders_new.count()

// 4. Renommer (pendant une fenêtre de maintenance)
db.orders.renameCollection("orders_old")
db.orders_new.renameCollection("orders")

// Approche 2 : Hashed shard key
db.adminCommand({
  shardCollection: "mydb.orders_new",
  key: {customerId: "hashed"}
})
// Distribution automatique uniforme
```

#### Solution 3 : Refactoring avec Resharding (MongoDB 5.0+)

```javascript
// MongoDB 5.0+ permet de changer la shard key en place !

// Vérifier la version
db.version()

// Resharding avec nouvelle shard key
db.adminCommand({
  reshardCollection: "mydb.orders",
  key: {customerId: 1, orderDate: 1}
})

// Suivre la progression
db.getSiblingDB("config").reshardingOperations.find()

// Note : C'est une opération lourde qui peut prendre du temps
// Planifier pendant une fenêtre de maintenance
```

#### Solution 4 : Optimiser les Données

```javascript
// Stratégie 1 : Agréger les petits documents
// Si beaucoup de petits documents créent un jumbo chunk

// Avant : 1 document par événement
{_id: 1, userId: 123, event: "click", timestamp: ...}
{_id: 2, userId: 123, event: "view", timestamp: ...}
// ... 10 millions de petits documents

// Après : Agréger par période
{
  _id: ObjectId(),
  userId: 123,
  date: ISODate("2024-01-15"),
  events: [
    {type: "click", timestamp: ...},
    {type: "view", timestamp: ...},
    // ... événements de la journée
  ],
  eventCount: 142
}

// Stratégie 2 : Externaliser les gros champs
// Si des documents individuels sont très gros

// Avant : Tout dans un document
{
  _id: 1,
  userId: 123,
  metadata: {...},
  largeData: "... plusieurs MB ..."
}

// Après : Séparer les gros champs
// Collection principale
{
  _id: 1,
  userId: 123,
  metadata: {...},
  largeDataRef: "large_1"
}

// Collection pour gros champs (séparée)
{
  _id: "large_1",
  data: "... plusieurs MB ..."
}
```

#### Solution 5 : Autoriser la Migration des Jumbo Chunks

```javascript
// ⚠️ ATTENTION : Peut causer des problèmes de performance

// Permettre temporairement la migration des jumbo chunks
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {attemptToBalanceJumboChunks: true}},
  {upsert: true}
)

// Surveiller attentivement
db.currentOp({
  desc: {$regex: /^migrateThread/}
})

// Désactiver après équilibrage
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {attemptToBalanceJumboChunks: false}}
)
```

---

## 3. Requêtes Scatter-Gather

### Symptômes

```
Slow queries despite indexes
High network traffic between mongos and shards
All shards contacted for simple queries
Poor query performance in sharded environment
```

### Causes Possibles

- Requêtes sans shard key
- Shard key non incluse dans les filtres
- Requêtes sur champs non-sharded
- Mauvaise conception de l'application

---

### Diagnostic Pas à Pas

#### Étape 1 : Identifier les Requêtes Scatter-Gather

```javascript
// Activer le profiler sur mongos
db.setProfilingLevel(1, {slowms: 100})

// Analyser les requêtes
db.system.profile.find({
  "command.shardVersion": {$exists: true}
}).sort({ts: -1}).limit(20).forEach(doc => {
  print("Query:", tojson(doc.command.filter))
  print("Shards used:", doc.nShards || "N/A")
  print("Millis:", doc.millis)
  print("---")
})

// Requêtes qui touchent tous les shards
db.system.profile.find({
  nShards: {$eq: db.getSiblingDB("config").shards.count()}
}).count()
```

#### Étape 2 : Analyser avec explain()

```javascript
// Requête sans shard key
db.orders.find({status: "pending"}).explain("executionStats")

// Sortie importante :
// {
//   "queryPlanner": {
//     "winningPlan": {
//       "stage": "SHARD_MERGE",  // ⚠️ Scatter-gather !
//       "shards": [
//         {shardName: "shard0000", ...},
//         {shardName: "shard0001", ...},
//         {shardName: "shard0002", ...}
//         // Tous les shards !
//       ]
//     }
//   },
//   "executionStats": {
//     "nReturned": 100,
//     "totalDocsExamined": 1000000,  // Examiner beaucoup de docs
//     "totalKeysExamined": 1000000,
//     "executionTimeMillis": 2543
//   }
// }

// Requête avec shard key
db.orders.find({
  customerId: 12345,  // Shard key !
  status: "pending"
}).explain("executionStats")

// Sortie avec targeting :
// {
//   "queryPlanner": {
//     "winningPlan": {
//       "stage": "SINGLE_SHARD",  // ✅ Un seul shard !
//       "shards": [
//         {shardName: "shard0001", ...}
//       ]
//     }
//   }
// }
```

#### Étape 3 : Analyser les Patterns d'Accès

```javascript
// Identifier les collections problématiques
db.system.profile.aggregate([
  {$match: {
    op: "query",
    nShards: {$gt: 1}  // Multi-shard queries
  }},
  {$group: {
    _id: "$ns",
    count: {$sum: 1},
    avgShards: {$avg: "$nShards"},
    avgTime: {$avg: "$millis"}
  }},
  {$sort: {count: -1}}
])
```

---

### Résolution Pas à Pas

#### Solution 1 : Inclure la Shard Key dans les Requêtes

```javascript
// ❌ MAUVAIS : Requête sans shard key
db.orders.find({status: "pending"})
// Scatter-gather sur tous les shards

// ✅ BON : Requête avec shard key
db.orders.find({
  customerId: 12345,  // Shard key
  status: "pending"
})
// Targeted query sur un seul shard

// ❌ MAUVAIS : Update sans shard key
db.orders.updateMany(
  {status: "pending"},
  {$set: {status: "processing"}}
)
// Impacte tous les shards

// ✅ BON : Update avec shard key
db.orders.updateMany(
  {
    customerId: 12345,  // Shard key
    status: "pending"
  },
  {$set: {status: "processing"}}
)
```

#### Solution 2 : Refactorer le Code Applicatif

```javascript
// Pattern 1 : Lookup par utilisateur
// Application Node.js

// ❌ MAUVAIS
async function getUserOrders(status) {
  // Cherche dans tous les shards !
  return await db.orders.find({status: status}).toArray()
}

// ✅ BON
async function getUserOrders(userId, status) {
  // Targeted query
  return await db.orders.find({
    customerId: userId,  // Shard key
    status: status
  }).toArray()
}

// Pattern 2 : Recherche globale nécessaire
// Utiliser l'agrégation avec $facet pour combiner résultats

// ✅ OPTIMAL : Pré-calculer les agrégations
// Collection orders_by_status (maintenue par triggers)
{
  _id: "pending",
  orderIds: [123, 456, 789, ...],
  count: 3,
  lastUpdated: ISODate("...")
}

// Requête rapide ciblée
db.orders_by_status.findOne({_id: "pending"})
```

#### Solution 3 : Créer des Index Secondaires

```javascript
// Pour requêtes sans shard key mais fréquentes

// Créer un index sur le champ fréquemment requêté
db.orders.createIndex({status: 1, orderDate: -1})

// Maintenant la requête est plus rapide même en scatter-gather
db.orders.find({status: "pending"}).sort({orderDate: -1})

// Note : Toujours moins performant qu'une targeted query,
// mais mieux que sans index
```

#### Solution 4 : Utiliser des Zones (Zone Sharding)

```javascript
// Pour isoler certaines requêtes sur des shards spécifiques

// Définir des zones géographiques
sh.addShardToZone("shard0000", "EU")
sh.addShardToZone("shard0001", "US")
sh.addShardToZone("shard0002", "ASIA")

// Définir les ranges
sh.updateZoneKeyRange(
  "mydb.orders",
  {customerId: MinKey, region: "EU"},
  {customerId: MaxKey, region: "EU"},
  "EU"
)

sh.updateZoneKeyRange(
  "mydb.orders",
  {customerId: MinKey, region: "US"},
  {customerId: MaxKey, region: "US"},
  "US"
)

// Requêtes avec région = targeted vers un zone
db.orders.find({
  region: "EU",
  status: "pending"
})
// Touche seulement les shards de la zone EU
```

#### Solution 5 : Monitoring et Alertes

```javascript
// Monitorer le ratio scatter-gather
function monitorScatterGather() {
  var stats = db.system.profile.aggregate([
    {$match: {op: "query"}},
    {$group: {
      _id: null,
      totalQueries: {$sum: 1},
      scatterGatherQueries: {
        $sum: {$cond: [{$gt: ["$nShards", 1]}, 1, 0]}
      }
    }},
    {$project: {
      totalQueries: 1,
      scatterGatherQueries: 1,
      percentage: {
        $multiply: [
          {$divide: ["$scatterGatherQueries", "$totalQueries"]},
          100
        ]
      }
    }}
  ]).toArray()[0]

  if (stats.percentage > 30) {  // > 30% scatter-gather
    console.warn(`WARNING: ${stats.percentage.toFixed(1)}% scatter-gather queries`)
  }

  return stats
}

monitorScatterGather()
```

---

## 4. Hotspots sur Shards

### Symptômes

```
One shard overloaded while others idle
Uneven write distribution
Performance bottleneck on specific shard
Increased latency for certain operations
CPU/Memory spike on one shard
```

### Causes Possibles

- Shard key monotone (auto-increment, timestamp)
- Concentration de valeurs populaires
- Écriture séquentielle
- Mauvais choix de shard key
- Zone sharding mal configuré

---

### Diagnostic Pas à Pas

#### Étape 1 : Identifier le Hotspot

```javascript
// Vérifier la distribution des opérations
db.getSiblingDB("config").shards.find().forEach(shard => {
  print("\n=== " + shard._id + " ===")

  var conn = new Mongo(shard.host.split(',')[0])  // Premier membre du replica set
  var serverStatus = conn.getDB("admin").serverStatus()

  print("Ops per second:")
  print("  Insert:", serverStatus.opcounters.insert)
  print("  Query:", serverStatus.opcounters.query)
  print("  Update:", serverStatus.opcounters.update)
  print("  Delete:", serverStatus.opcounters.delete)

  print("Connections:", serverStatus.connections.current)

  print("Memory:")
  print("  Resident:", serverStatus.mem.resident, "MB")
})

// Distribution des chunks (devrait être équilibrée)
db.getSiblingDB("config").chunks.aggregate([
  {$match: {ns: "mydb.orders"}},
  {$group: {
    _id: "$shard",
    chunks: {$sum: 1}
  }},
  {$sort: {chunks: -1}}
])

// Distribution de la taille des données
db.adminCommand({
  dbStats: 1,
  scale: 1024 * 1024 * 1024  // GB
})
```

#### Étape 2 : Analyser la Shard Key

```javascript
// Voir la shard key
db.getSiblingDB("config").collections.findOne({
  _id: "mydb.orders"
}).key

// Analyser la distribution des valeurs
db.orders.aggregate([
  {$sample: {size: 10000}},  // Échantillon
  {$group: {
    _id: "$customerId",  // Shard key
    count: {$sum: 1}
  }},
  {$sort: {count: -1}},
  {$limit: 20}
])

// Identifier les valeurs très fréquentes (hotspot potentiel)
```

#### Étape 3 : Analyser les Patterns d'Écriture

```javascript
// Voir où vont les nouvelles insertions
db.getSiblingDB("config").changelog.find({
  what: "multi-split",
  time: {$gt: new Date(Date.now() - 3600000)}  // Dernière heure
}).forEach(entry => {
  print("Namespace:", entry.ns)
  print("Shard:", entry.details.before.shard)
  print("Number of splits:", entry.details.number)
})

// Pour shard key monotone (problème)
db.orders.find().sort({_id: -1}).limit(10)
// Si tous les nouveaux docs vont sur le même chunk/shard = hotspot
```

---

### Résolution Pas à Pas

#### Solution 1 : Utiliser Hashed Shard Key

```javascript
// Pour éviter hotspots sur shard keys monotones

// ❌ PROBLÈME : Shard key séquentielle
db.adminCommand({
  shardCollection: "mydb.orders",
  key: {orderId: 1}  // Auto-increment = hotspot !
})
// Toutes les nouvelles insertions vont sur le dernier chunk

// ✅ SOLUTION : Hashed shard key
db.adminCommand({
  shardCollection: "mydb.orders_new",
  key: {orderId: "hashed"}
})
// Distribution automatique uniforme

// Migration des données (voir section Jumbo Chunks)
```

#### Solution 2 : Shard Key Composée

```javascript
// Ajouter de la cardinalité pour éviter hotspots

// ❌ PROBLÈME : Seulement timestamp
db.adminCommand({
  shardCollection: "mydb.events",
  key: {timestamp: 1}  // Monotone = hotspot
})

// ✅ SOLUTION : Composer avec un champ aléatoire/distribué
db.adminCommand({
  shardCollection: "mydb.events_new",
  key: {userId: 1, timestamp: 1}  // userId distribue la charge
})

// ✅ ALTERNATIVE : Ajouter un champ randomisé
// Ajouter un champ à l'insertion
{
  _id: ObjectId("..."),
  timestamp: ISODate("..."),
  shardBucket: Math.floor(Math.random() * 100),  // 0-99
  data: {...}
}

db.adminCommand({
  shardCollection: "mydb.events_new",
  key: {shardBucket: 1, timestamp: 1}
})
```

#### Solution 3 : Pré-split des Chunks

```javascript
// Pour nouvelles collections avec charge prévisible

// Créer des chunks vides répartis sur tous les shards
// Shard key : {userId: 1}

// Obtenir les limites de userId (min/max)
var minUserId = 1
var maxUserId = 1000000

// Calculer les points de split
var numShards = db.getSiblingDB("config").shards.count()
var numSplits = numShards * 10  // 10 chunks par shard

for (var i = 1; i < numSplits; i++) {
  var splitPoint = minUserId + (i * (maxUserId - minUserId) / numSplits)
  sh.splitAt("mydb.users", {userId: Math.floor(splitPoint)})
}

// Forcer la distribution
sh.startBalancer()
```

#### Solution 4 : Ajuster les Zones pour Équilibrer

```javascript
// Redéfinir les zones pour mieux distribuer

// Identifier les plages problématiques
db.getSiblingDB("config").chunks.aggregate([
  {$match: {ns: "mydb.orders"}},
  {$group: {
    _id: {
      shard: "$shard",
      min: "$min"
    },
    count: {$sum: 1}
  }},
  {$sort: {count: -1}}
])

// Ajuster les zones
sh.removeShardFromZone("shard0000", "HOT_ZONE")
sh.addShardToZone("shard0001", "HOT_ZONE")

// Redéfinir les ranges
sh.updateZoneKeyRange(
  "mydb.orders",
  {customerId: 0},
  {customerId: 500000},
  "ZONE_1"
)

sh.updateZoneKeyRange(
  "mydb.orders",
  {customerId: 500000},
  {customerId: 1000000},
  "ZONE_2"
)
```

#### Solution 5 : Scaling Horizontal du Shard

```javascript
// Si un shard est définitivement plus chargé

// Ajouter un nouveau shard
sh.addShard("shard0003/mongodb-shard03:27017")

// Forcer la redistribution des chunks du shard surchargé
var overloadedShard = "shard0000"
var targetShard = "shard0003"

// Déplacer manuellement des chunks
db.getSiblingDB("config").chunks.find({
  ns: "mydb.orders",
  shard: overloadedShard
}).limit(10).forEach(chunk => {
  sh.moveChunk(
    "mydb.orders",
    chunk.min,
    targetShard
  )
})

// Ou laisser le balancer faire le travail
sh.startBalancer()
```

---

## 5. Migration de Chunks Échouée

### Symptômes

```
"Migration failed" errors in logs
Chunks stuck in migration
Data temporarily unavailable
Balancer repeatedly failing
Rollback of migrations
```

### Causes Possibles

- Ressources insuffisantes (CPU, mémoire, disque)
- Timeout de migration
- Conflit de locks
- Erreurs réseau
- Documents orphelins

---

### Diagnostic Pas à Pas

#### Étape 1 : Identifier les Migrations Échouées

```javascript
// Voir l'historique des migrations
db.getSiblingDB("config").changelog.find({
  what: {$in: ["moveChunk.start", "moveChunk.commit", "moveChunk.error"]}
}).sort({time: -1}).limit(50).forEach(entry => {
  print(entry.time, "-", entry.what, "-", entry.ns)
  if (entry.details && entry.details.errmsg) {
    print("  Error:", entry.details.errmsg)
  }
})

// Compter les échecs récents
db.getSiblingDB("config").changelog.count({
  what: "moveChunk.error",
  time: {$gt: new Date(Date.now() - 3600000)}  // Dernière heure
})

// Migrations en cours
db.currentOp({
  desc: {$regex: /^migrateThread/}
})
```

#### Étape 2 : Analyser les Logs

```bash
# Sur le shard source et destination
grep -i "migrate\|moveChunk" /var/log/mongodb/mongod.log | tail -100

# Messages clés :
# - "migration failed"
# - "error applying batch"
# - "destination failed with"
# - "migration commit failed"
# - "range deletion timed out"
```

#### Étape 3 : Vérifier les Ressources

```bash
# Sur chaque shard pendant une migration
# CPU
top -p $(pgrep mongod)

# Mémoire
free -h

# Espace disque
df -h /var/lib/mongodb

# I/O
iostat -x 1 5

# Réseau
iftop -i eth0
```

---

### Résolution Pas à Pas

#### Solution 1 : Augmenter les Timeouts

```javascript
// Paramètres de migration
db.adminCommand({
  setParameter: 1,
  // Timeout pour la phase de clone (défaut: 30 min)
  migrationCloneTimeout: 7200000,  // 2 heures (ms)
  // Timeout pour le catchup (défaut: pas de timeout)
  maxTimeMS: 3600000  // 1 heure
})

// Sur le shard source et destination
```

#### Solution 2 : Nettoyer les Documents Orphelins

```javascript
// Documents orphelins = chunks migrés mais pas nettoyés

// Sur chaque shard, exécuter le nettoyage
db.adminCommand({
  cleanupOrphaned: "mydb.orders",
  startingFromKey: {customerId: MinKey}
})

// Répéter jusqu'à ce qu'il n'y ait plus d'orphelins
// Sortie :
// {
//   ok: 1,
//   stoppedAtKey: {customerId: ...}
// }

// Si stoppedAtKey existe, continuer depuis ce point
db.adminCommand({
  cleanupOrphaned: "mydb.orders",
  startingFromKey: {customerId: <dernière valeur>}
})
```

#### Solution 3 : Réduire la Charge Pendant Migration

```javascript
// Ralentir le balancer
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {
    // Limiter à 1 migration à la fois
    _secondaryThrottle: true,
    _waitForDelete: true
  }},
  {upsert: true}
)

// Définir une fenêtre de migration
db.getSiblingDB("config").settings.updateOne(
  {_id: "balancer"},
  {$set: {
    activeWindow: {
      start: "02:00",
      stop: "05:00"
    }
  }}
)
```

#### Solution 4 : Forcer la Fin d'une Migration Bloquée

```javascript
// ⚠️ ATTENTION : Peut causer des incohérences

// 1. Identifier la migration bloquée
db.currentOp({desc: {$regex: /^migrateThread/}})

// 2. Tuer l'opération
db.killOp(<opid>)

// 3. Nettoyer les métadonnées
db.getSiblingDB("config").migrations.deleteMany({
  _id: {$exists: true}
})

// 4. Redémarrer le balancer
sh.stopBalancer()
sh.startBalancer()
```

#### Solution 5 : Migration Manuelle Contrôlée

```javascript
// Pour migrations critiques, faire manuellement

// 1. Désactiver le balancer
sh.stopBalancer()

// 2. Identifier le chunk à migrer
var chunk = db.getSiblingDB("config").chunks.findOne({
  ns: "mydb.orders",
  shard: "shard0000",
  // Filtrer selon besoin
})

// 3. Migrer avec monitoring
sh.moveChunk("mydb.orders", chunk.min, "shard0001")

// 4. Vérifier
db.getSiblingDB("config").chunks.findOne({_id: chunk._id})

// 5. Nettoyer les orphelins
db.adminCommand({cleanupOrphaned: "mydb.orders"})

// 6. Réactiver le balancer
sh.startBalancer()
```

---

## 6. Problèmes de Config Servers

### Symptômes

```
Unable to connect to config server
Metadata operations failing
"No config server available" errors
Sharding operations blocked
Cannot add/remove shards
```

### Diagnostic

```javascript
// État du config server replica set
db.getSiblingDB("config").hello()

// Membres du config replica set
use config
rs.status()

// Vérifier la configuration
rs.conf()
```

### Résolution

```javascript
// Si config server down, démarrer en mode standalone
// Reconstruire le replica set si nécessaire

// Sur le config server survivant
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    {_id: 0, host: "config1:27019"},
    {_id: 1, host: "config2:27019"},
    {_id: 2, host: "config3:27019"}
  ]
})
```

---

## 7. Problèmes de Mongos

### Symptômes

```
Cannot connect to mongos
Routing errors
"No primary detected" errors
Slow query routing
Connection pool exhaustion
```

### Diagnostic

```javascript
// État de mongos
db.adminCommand({serverStatus: 1})

// Voir les shards connus
sh.status()

// Connexions
db.serverStatus().connections
```

### Résolution

```bash
# Redémarrer mongos
sudo systemctl restart mongos

# Vérifier la configuration
cat /etc/mongos.conf

# Tester la connexion
mongosh --host mongos:27017 --eval "db.adminCommand({hello: 1})"
```

---

## 8. Shard Key Inadéquate

### Symptômes

```
Poor query performance
Uneven data distribution
Frequent jumbo chunks
Many scatter-gather queries
Hotspots on specific shards
```

### Diagnostic

```javascript
// Analyser l'efficacité de la shard key actuelle
// 1. Cardinalité
db.orders.aggregate([
  {$group: {_id: "$customerId"}},
  {$count: "uniqueValues"}
])

// 2. Distribution
db.getSiblingDB("config").chunks.aggregate([
  {$match: {ns: "mydb.orders"}},
  {$group: {_id: "$shard", count: {$sum: 1}}},
  {$sort: {count: -1}}
])

// 3. Fréquence d'accès
db.system.profile.aggregate([
  {$match: {
    ns: "mydb.orders",
    "command.filter.customerId": {$exists: true}
  }},
  {$count: "targetedQueries"}
])
```

### Résolution

**Critères d'une bonne shard key :**

```
1. Cardinalité élevée (nombreuses valeurs uniques)
2. Fréquence faible (valeurs réparties uniformément)
3. Monotonie évitée (pas auto-increment ni timestamp seul)
4. Incluse dans la plupart des requêtes
```

**Options :**

```javascript
// Option 1 : Hashed shard key
{shardKeyField: "hashed"}

// Option 2 : Compound shard key
{field1: 1, field2: 1}

// Option 3 : Avec randomisation
{bucket: 1, timestamp: 1}
// où bucket = Math.floor(Math.random() * 100)
```

---

## Checklist Globale Sharding

### Diagnostic Rapide (10 minutes)

```bash
# 1. État du cluster
mongosh --host mongos --eval "sh.status()"

# 2. Balancer actif ?
mongosh --host mongos --eval "sh.isBalancerRunning()"

# 3. Distribution des chunks
mongosh --host mongos --eval "db.getSiblingDB('config').chunks.aggregate([{\$group: {_id: '\$shard', count: {\$sum: 1}}}, {\$sort: {count: -1}}])"

# 4. Jumbo chunks ?
mongosh --host mongos --eval "db.getSiblingDB('config').chunks.count({jumbo: true})"

# 5. Migrations récentes
mongosh --host mongos --eval "db.getSiblingDB('config').changelog.find({what: 'moveChunk.error'}).sort({time: -1}).limit(5)"
```

### Monitoring Continu

```javascript
// Script de monitoring sharding
function shardingHealthCheck() {
  print("=== Sharding Health Check ===\n")

  // 1. Nombre de shards
  var numShards = db.getSiblingDB("config").shards.count()
  print("Shards:", numShards)

  // 2. Balancer
  print("\nBalancer:")
  print("  Active:", sh.isBalancerRunning())
  print("  State:", sh.getBalancerState())

  // 3. Distribution des chunks
  print("\nChunk distribution:")
  db.getSiblingDB("config").chunks.aggregate([
    {$group: {_id: "$shard", chunks: {$sum: 1}}},
    {$sort: {chunks: -1}}
  ]).forEach(s => {
    print(`  ${s._id}: ${s.chunks} chunks`)
  })

  // 4. Jumbo chunks
  var jumboCount = db.getSiblingDB("config").chunks.count({jumbo: true})
  print("\nJumbo chunks:", jumboCount)
  if (jumboCount > 0) {
    print("  ⚠️  WARNING: Jumbo chunks detected")
  }

  // 5. Migrations récentes
  var recentErrors = db.getSiblingDB("config").changelog.count({
    what: "moveChunk.error",
    time: {$gt: new Date(Date.now() - 3600000)}
  })
  print("\nMigration errors (last hour):", recentErrors)
  if (recentErrors > 0) {
    print("  ⚠️  WARNING: Recent migration failures")
  }

  print("\n=== End of Health Check ===")
}

shardingHealthCheck()
```

---

## Conclusion

Le sharding MongoDB offre une scalabilité horizontale mais nécessite :

1. **Shard key bien choisie** (cardinalité, distribution, queries)
2. **Monitoring actif** (balancer, distribution, migrations)
3. **Maintenance régulière** (jumbo chunks, orphans)
4. **Configuration optimale** (balancer, timeouts, zones)

**Points critiques :**
- ✅ Shard key avec haute cardinalité et distribution uniforme
- ✅ Éviter les shard keys monotones (timestamp, auto-increment)
- ✅ Inclure la shard key dans les requêtes (éviter scatter-gather)
- ✅ Balancer activé avec fenêtre appropriée
- ✅ Monitoring continu des jumbo chunks
- ✅ Nettoyer les orphelins régulièrement

---


⏭️ [Corruption de données](/22-depannage-resolution-problemes/05-corruption-donnees.md)
