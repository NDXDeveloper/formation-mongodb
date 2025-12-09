🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.2 Problèmes de Performance

## Vue d'ensemble

Les problèmes de performance sont parmi les incidents les plus fréquents et peuvent avoir des impacts critiques sur l'expérience utilisateur et la stabilité du système. Cette section fournit des méthodologies complètes pour diagnostiquer et résoudre les problèmes de performance MongoDB.

---

## Table des Matières

1. [Requêtes Lentes](#1-requ%C3%AAtes-lentes)
2. [Utilisation CPU Élevée](#2-utilisation-cpu-%C3%A9lev%C3%A9e)
3. [Saturation Mémoire](#3-saturation-m%C3%A9moire)
4. [Goulots d'Étranglement I/O](#4-goulots-d%C3%A9tranglement-io)
5. [Lock Contention](#5-lock-contention)
6. [Problèmes de Cache](#6-probl%C3%A8mes-de-cache)
7. [Problèmes d'Agrégation](#7-probl%C3%A8mes-dagr%C3%A9gation)
8. [Monitoring et Métriques](#8-monitoring-et-m%C3%A9triques)

---

## 1. Requêtes Lentes

### Symptômes

```
Application timeouts
Response times > 1 second
High queue depth
User complaints about slowness
```

### Causes Possibles

- Absence d'index appropriés
- Index non utilisés (query planner inefficace)
- Collection scans complets
- Requêtes sur documents trop volumineux
- Projection insuffisante
- Requêtes complexes non optimisées

---

### Diagnostic Pas à Pas

#### Étape 1 : Activer et Analyser le Profiler

```javascript
// Activer le profiler pour les requêtes > 100ms
db.setProfilingLevel(1, {slowms: 100})

// Vérifier le niveau actuel
db.getProfilingStatus()

// Voir les requêtes les plus lentes
db.system.profile.find()
  .sort({millis: -1})
  .limit(10)
  .pretty()

// Analyse statistique des requêtes lentes
db.system.profile.aggregate([
  {$match: {millis: {$gt: 100}}},
  {$group: {
    _id: "$ns",
    count: {$sum: 1},
    avgMillis: {$avg: "$millis"},
    maxMillis: {$max: "$millis"},
    operations: {$push: {
      op: "$op",
      millis: "$millis",
      ts: "$ts"
    }}
  }},
  {$sort: {avgMillis: -1}}
])
```

**Interpréter les résultats du profiler :**

```javascript
// Exemple de sortie
{
  "op" : "query",                    // Type d'opération
  "ns" : "mydb.users",               // Collection
  "command" : {
    "find" : "users",
    "filter" : { "age" : { "$gt" : 25 } }
  },
  "keysExamined" : 0,                // ⚠️ 0 = pas d'index utilisé
  "docsExamined" : 1000000,          // ⚠️ Scan complet !
  "nreturned" : 50000,               // Documents retournés
  "responseLength" : 5000000,        // Taille de la réponse
  "millis" : 2543,                   // ⚠️ 2.5 secondes !
  "execStats" : {
    "stage" : "COLLSCAN",            // ⚠️ Collection scan
    "nReturned" : 50000,
    "executionTimeMillis" : 2543
  }
}
```

**Signaux d'alerte :**
- `keysExamined: 0` → Aucun index utilisé
- `docsExamined >> nreturned` → Scan inefficace
- `stage: "COLLSCAN"` → Scan complet de collection
- `millis > 1000` → Requête très lente

#### Étape 2 : Analyser avec explain()

```javascript
// Analyse détaillée d'une requête
db.users.find({age: {$gt: 25}}).explain("executionStats")

// Analyse d'une requête complexe
db.users.find({
  age: {$gt: 25},
  city: "Paris"
}).sort({created: -1}).limit(10).explain("executionStats")

// Pour les agrégations
db.orders.explain("executionStats").aggregate([
  {$match: {status: "completed"}},
  {$group: {_id: "$customerId", total: {$sum: "$amount"}}}
])
```

**Interpréter explain() :**

```javascript
{
  "queryPlanner" : {
    "winningPlan" : {
      "stage" : "FETCH",             // Étape de récupération
      "inputStage" : {
        "stage" : "IXSCAN",          // ✅ Index scan (bon)
        "keyPattern" : { "age" : 1 },
        "indexName" : "age_1"
      }
    }
  },
  "executionStats" : {
    "executionTimeMillis" : 45,      // Temps d'exécution
    "totalKeysExamined" : 50000,     // Clés d'index examinées
    "totalDocsExamined" : 50000,     // Documents examinés
    "nReturned" : 50000,             // Documents retournés
    "executionStages" : {
      "stage" : "FETCH",
      "nReturned" : 50000,
      "executionTimeMillisEstimate" : 40,
      "inputStage" : {
        "stage" : "IXSCAN",          // Index utilisé
        "nReturned" : 50000,
        "executionTimeMillisEstimate" : 10
      }
    }
  }
}
```

**Métriques importantes :**

```
Ratio d'efficacité = nReturned / totalDocsExamined

✅ Excellent : ratio > 0.9 (90%+ des documents examinés sont retournés)
⚠️ Moyen : ratio 0.5-0.9
❌ Mauvais : ratio < 0.5 (beaucoup de documents examinés inutilement)
```

#### Étape 3 : Identifier les Index Manquants

```javascript
// Vérifier les index existants
db.users.getIndexes()

// Analyser l'utilisation des index
db.users.aggregate([{$indexStats: {}}])

// Identifier les collections sans index (sauf _id)
db.getCollectionNames().forEach(function(collection) {
  var indexes = db[collection].getIndexes();
  if (indexes.length === 1) {  // Seulement _id
    print(collection + " has only _id index");
  }
})

// Voir les index inutilisés
db.users.aggregate([
  {$indexStats: {}},
  {$match: {"accesses.ops": 0}},
  {$project: {name: 1, "accesses.since": 1}}
])
```

#### Étape 4 : Analyser les Patterns d'Accès

```javascript
// Top des requêtes par opération
db.system.profile.aggregate([
  {$group: {
    _id: {
      ns: "$ns",
      op: "$op",
      query: "$command.filter"
    },
    count: {$sum: 1},
    avgTime: {$avg: "$millis"},
    maxTime: {$max: "$millis"}
  }},
  {$sort: {count: -1}},
  {$limit: 20}
])

// Identifier les requêtes avec projections manquantes
db.system.profile.find({
  "command.projection": {$exists: false},
  "responseLength": {$gt: 1000000}  // > 1MB
})
```

#### Étape 5 : Mesurer l'Impact sur les Ressources

```javascript
// Vérifier les opérations en cours
db.currentOp({
  "active": true,
  "secs_running": {$gt: 5}
})

// Voir les opérations longues avec détails
db.currentOp(true).inprog.filter(op =>
  op.secs_running > 5 && op.op === "query"
).map(op => ({
  opid: op.opid,
  secs_running: op.secs_running,
  ns: op.ns,
  query: op.command
}))
```

---

### Résolution Pas à Pas

#### Solution 1 : Créer les Index Appropriés

**Analyse et création :**

```javascript
// 1. Identifier le pattern de requête
db.system.profile.aggregate([
  {$match: {ns: "mydb.users"}},
  {$group: {
    _id: "$command.filter",
    count: {$sum: 1},
    avgTime: {$avg: "$millis"}
  }},
  {$sort: {count: -1}}
])

// 2. Créer un index simple
db.users.createIndex({age: 1})

// 3. Vérifier l'amélioration
db.users.find({age: {$gt: 25}}).explain("executionStats")

// 4. Index composé pour requêtes multiples
db.users.createIndex({city: 1, age: 1})

// 5. Index avec ordre de tri
db.users.createIndex({status: 1, created: -1})

// 6. Créer en arrière-plan (production)
db.users.createIndex(
  {email: 1},
  {background: true, name: "email_idx"}
)
```

**Ordre des champs dans l'index composé (ESR Rule) :**

```
E - Equality (égalité)
S - Sort (tri)
R - Range (plage)

Exemple :
Query: {status: "active", age: {$gt: 25}}, sort: {created: -1}
Index optimal : {status: 1, created: -1, age: 1}
             (Equality) (Sort)      (Range)
```

**Exemple complet :**

```javascript
// Requête problématique
db.orders.find({
  status: "completed",
  customerId: {$in: [1, 2, 3]},
  amount: {$gt: 100}
}).sort({orderDate: -1})

// Analyse
db.orders.find({
  status: "completed",
  customerId: {$in: [1, 2, 3]},
  amount: {$gt: 100}
}).sort({orderDate: -1}).explain("executionStats")

// Création de l'index optimal
db.orders.createIndex({
  status: 1,           // Equality
  orderDate: -1,       // Sort
  customerId: 1,       // Range ($in)
  amount: 1            // Range ($gt)
})

// Vérification de l'amélioration
// AVANT : 2500ms, COLLSCAN
// APRÈS : 15ms, IXSCAN
```

#### Solution 2 : Optimiser les Projections

```javascript
// ❌ MAUVAIS : Récupère tous les champs
db.users.find({age: {$gt: 25}})

// ✅ BON : Projection explicite
db.users.find(
  {age: {$gt: 25}},
  {name: 1, email: 1, _id: 0}
)

// ❌ MAUVAIS : Documents volumineux
db.articles.find({category: "tech"})

// ✅ BON : Exclure les champs volumineux
db.articles.find(
  {category: "tech"},
  {content: 0, comments: 0}  // Exclure gros champs
)

// Index couvrant (covered query)
db.users.createIndex({age: 1, name: 1, email: 1})

db.users.find(
  {age: {$gt: 25}},
  {name: 1, email: 1, _id: 0}  // Tous les champs sont dans l'index
)
// Résultat : Pas de FETCH, seulement IXSCAN !
```

#### Solution 3 : Optimiser la Pagination

**❌ Pagination inefficace avec skip() :**

```javascript
// PROBLÈME : Skip examine tous les documents précédents
db.products.find()
  .sort({_id: 1})
  .skip(10000)   // ⚠️ Examine 10000 docs !
  .limit(20)

// Performance se dégrade avec l'offset
// Page 1 : 10ms
// Page 100 : 150ms
// Page 1000 : 2500ms
```

**✅ Pagination optimale avec range query :**

```javascript
// Première page
db.products.find()
  .sort({_id: 1})
  .limit(20)

// Sauvegarder le dernier _id : lastId = <dernier document _id>

// Pages suivantes
db.products.find({_id: {$gt: lastId}})
  .sort({_id: 1})
  .limit(20)

// Performance constante quelle que soit la page !
```

**Pagination avec index composé :**

```javascript
// Index pour tri + filtrage
db.products.createIndex({category: 1, created: -1, _id: 1})

// Première page
var results = db.products.find({category: "electronics"})
  .sort({created: -1, _id: 1})
  .limit(20)

// Récupérer les marqueurs
var lastCreated = results[results.length - 1].created
var lastId = results[results.length - 1]._id

// Pages suivantes
db.products.find({
  category: "electronics",
  $or: [
    {created: {$lt: lastCreated}},
    {created: lastCreated, _id: {$gt: lastId}}
  ]
})
.sort({created: -1, _id: 1})
.limit(20)
```

#### Solution 4 : Limiter les Résultats

```javascript
// ❌ MAUVAIS : Récupère potentiellement millions de docs
db.logs.find({level: "info"})

// ✅ BON : Toujours limiter
db.logs.find({level: "info"}).limit(1000)

// ✅ BON : Compter sans tout charger
db.logs.countDocuments({level: "info"})

// ✅ BON : Estimation rapide (moins précis mais instantané)
db.logs.estimatedDocumentCount()

// ❌ MAUVAIS : Sort en mémoire sur résultats non indexés
db.logs.find({message: /error/}).sort({timestamp: -1})

// ✅ BON : Sort avec index
db.logs.createIndex({timestamp: -1})
db.logs.find().sort({timestamp: -1}).limit(100)
```

#### Solution 5 : Reécrire les Requêtes Inefficaces

**Exemple 1 : $where → Opérateurs standards**

```javascript
// ❌ TRÈS LENT : $where exécute du JavaScript
db.users.find({
  $where: "this.age > 25 && this.status === 'active'"
})

// ✅ RAPIDE : Opérateurs natifs
db.users.find({
  age: {$gt: 25},
  status: "active"
})
```

**Exemple 2 : $regex non ancré → $regex ancré**

```javascript
// ❌ LENT : Scan complet même avec index
db.users.find({email: /example\.com/})

// ✅ RAPIDE : Peut utiliser l'index
db.users.find({email: /^.*@example\.com$/})

// ✅ ENCORE MIEUX : Recherche exacte
db.users.find({email: "user@example.com"})

// ✅ OPTIMAL : Index texte pour recherche
db.users.createIndex({email: "text"})
db.users.find({$text: {$search: "example.com"}})
```

**Exemple 3 : $in avec beaucoup de valeurs → Alternative**

```javascript
// ⚠️ PEUT ÊTRE LENT : $in avec 1000+ valeurs
db.orders.find({customerId: {$in: [/* 10000 IDs */]}})

// ✅ MIEUX : Si possible, inverser la relation
// Plutôt que chercher dans orders
// Ajouter les orderIds dans customer
db.customers.findOne({_id: customerId}).orderIds

// ✅ OU : Chunker les requêtes
const chunkSize = 1000
for (let i = 0; i < customerIds.length; i += chunkSize) {
  const chunk = customerIds.slice(i, i + chunkSize)
  db.orders.find({customerId: {$in: chunk}})
}
```

#### Solution 6 : Utiliser les Hints pour Forcer un Index

```javascript
// Query planner choisit mal
db.users.find({
  age: {$gt: 25},
  city: "Paris"
}).explain("executionStats")
// Utilise age_1 mais city_1_age_1 serait meilleur

// Forcer l'utilisation d'un index spécifique
db.users.find({
  age: {$gt: 25},
  city: "Paris"
}).hint({city: 1, age: 1})

// Forcer par nom d'index
db.users.find({
  age: {$gt: 25},
  city: "Paris"
}).hint("city_1_age_1")

// Désactiver l'utilisation d'index (forcer COLLSCAN)
db.users.find({age: {$gt: 25}}).hint({$natural: 1})
```

---

## 2. Utilisation CPU Élevée

### Symptômes

```
CPU usage > 80% sustained
High load average
System unresponsive
Slow query execution even with indexes
```

### Causes Possibles

- Requêtes complexes non optimisées
- Agrégations lourdes
- Scan de collections volumineuses
- Tri en mémoire
- Calculs JavaScript côté serveur
- Lock contention
- Trop d'opérations concurrentes

---

### Diagnostic Pas à Pas

#### Étape 1 : Identifier la Cause de la Charge CPU

```bash
# Vérifier l'utilisation CPU globale
top -p $(pgrep -d',' mongod)

# Détails par thread MongoDB
top -H -p $(pgrep mongod)

# Statistiques détaillées
pidstat -t -p $(pgrep mongod) 1 10

# Tracer les appels système (attention : overhead)
strace -c -p $(pgrep mongod)
```

```javascript
// Identifier les opérations consommatrices
db.currentOp({
  "secs_running": {$gt: 5},
  "microsecs_running": {$gt: 5000000}
}).inprog.forEach(op => {
  printjson({
    opid: op.opid,
    op: op.op,
    ns: op.ns,
    secs: op.secs_running,
    query: op.command
  })
})

// Voir les opérations par type
db.currentOp().inprog.reduce((acc, op) => {
  acc[op.op] = (acc[op.op] || 0) + 1
  return acc
}, {})
```

#### Étape 2 : Analyser les Métriques Serveur

```javascript
// Métriques globales
var stats = db.serverStatus()

// Compteurs d'opérations
printjson(stats.opcounters)
// {
//   insert: 123456,
//   query: 987654,
//   update: 456789,
//   delete: 12345,
//   getmore: 234567,
//   command: 876543
// }

// Métriques réseau
printjson(stats.network)
// Identifier si beaucoup de données transférées

// Connexions
printjson(stats.connections)

// WiredTiger cache
printjson(stats.wiredTiger.cache)
```

#### Étape 3 : Identifier les Requêtes Problématiques

```javascript
// Top des requêtes CPU-intensives
db.system.profile.aggregate([
  {$match: {
    millis: {$gt: 1000},
    ts: {$gt: new Date(Date.now() - 3600000)}  // Dernière heure
  }},
  {$group: {
    _id: {
      ns: "$ns",
      op: "$op",
      planSummary: "$planSummary"
    },
    count: {$sum: 1},
    avgTime: {$avg: "$millis"},
    maxTime: {$max: "$millis"},
    totalTime: {$sum: "$millis"}
  }},
  {$sort: {totalTime: -1}},
  {$limit: 10}
])
```

#### Étape 4 : Analyser les Patterns de Tri

```javascript
// Identifier les sorts en mémoire (très coûteux)
db.system.profile.find({
  "execStats.stage": "SORT",
  "execStats.sortPattern": {$exists: true}
})

// Taille des sorts en mémoire
db.system.profile.find({
  "execStats.memUsage": {$gt: 33554432}  // > 32MB
}).forEach(doc => {
  print("Sort memory: " + (doc.execStats.memUsage / 1024 / 1024) + " MB")
  printjson(doc.command)
})
```

---

### Résolution Pas à Pas

#### Solution 1 : Optimiser les Tris avec Index

```javascript
// ❌ PROBLÈME : Sort en mémoire
db.orders.find({status: "completed"})
  .sort({orderDate: -1})
  .limit(100)

// Explain montre :
// "stage": "SORT"
// "memUsage": 52428800  // 50MB en mémoire !
// "inputStage": {"stage": "COLLSCAN"}

// ✅ SOLUTION : Index sur le champ de tri
db.orders.createIndex({status: 1, orderDate: -1})

// Après :
// "stage": "FETCH"
// "inputStage": {"stage": "IXSCAN"}
// Pas de sort en mémoire !
```

**Index pour tri composé :**

```javascript
// Query avec filtrage et tri
db.orders.find({
  status: "completed",
  customerId: 123
}).sort({orderDate: -1, amount: 1})

// Index optimal
db.orders.createIndex({
  status: 1,
  customerId: 1,
  orderDate: -1,
  amount: 1
})
```

#### Solution 2 : Limiter les Calculs JavaScript

```javascript
// ❌ TRÈS LENT : $where avec JavaScript
db.products.find({
  $where: function() {
    return this.price * this.quantity > 1000
  }
})

// ✅ RAPIDE : $expr avec opérateurs natifs
db.products.find({
  $expr: {
    $gt: [{$multiply: ["$price", "$quantity"]}, 1000]
  }
})

// ✅ ENCORE MIEUX : Champ calculé stocké
// Ajouter un champ "totalValue" lors de l'insertion/update
db.products.updateMany({}, [{
  $set: {
    totalValue: {$multiply: ["$price", "$quantity"]}
  }
}])

// Créer un index sur le champ calculé
db.products.createIndex({totalValue: 1})

// Query devient simple
db.products.find({totalValue: {$gt: 1000}})
```

#### Solution 3 : Augmenter les Ressources Serveur

**Vérifier la configuration WiredTiger :**

```javascript
// Voir le cache actuel
db.serverStatus().wiredTiger.cache

// Ajuster le cache (50% de RAM par défaut)
// /etc/mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  // Ajuster selon RAM disponible

// Redémarrer MongoDB
// sudo systemctl restart mongod
```

**Augmenter les ressources système :**

```bash
# Vérifier les limites actuelles
ulimit -a

# Augmenter les limites
# /etc/security/limits.conf
mongodb soft nofile 64000
mongodb hard nofile 64000
mongodb soft nproc 32000
mongodb hard nproc 32000
```

#### Solution 4 : Distribuer la Charge

**Read Preference sur Secondaries :**

```javascript
// Node.js - Lire depuis secondaries
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred',
  readConcern: { level: 'available' }
})

// Requêtes analytics sur secondary
db.orders.find({status: "completed"}).readPref("secondary")
```

**Sharding pour distribution horizontale :**

```javascript
// Activer le sharding
sh.enableSharding("mydb")

// Sharder une collection volumineuse
sh.shardCollection("mydb.orders", {customerId: "hashed"})

// Vérifier la distribution
db.orders.getShardDistribution()
```

#### Solution 5 : Implémenter le Caching Applicatif

```javascript
// Node.js avec Redis
const redis = require('redis')
const client = redis.createClient()

async function getUser(userId) {
  // 1. Vérifier le cache
  const cached = await client.get(`user:${userId}`)
  if (cached) {
    return JSON.parse(cached)
  }

  // 2. Query MongoDB
  const user = await db.users.findOne({_id: userId})

  // 3. Mettre en cache (TTL 5 min)
  await client.setex(
    `user:${userId}`,
    300,
    JSON.stringify(user)
  )

  return user
}

// Invalider le cache lors des updates
async function updateUser(userId, data) {
  await db.users.updateOne(
    {_id: userId},
    {$set: data}
  )

  // Invalider le cache
  await client.del(`user:${userId}`)
}
```

---

## 3. Saturation Mémoire

### Symptômes

```
High memory usage (> 90%)
OOM (Out of Memory) errors
Swapping activity
Slow performance despite good queries
MongoDB restarts unexpectedly
```

### Causes Possibles

- Cache WiredTiger trop grand
- Documents très volumineux
- Résultats de requêtes non paginés
- Accumulation de connexions
- Memory leaks dans l'application
- Agrégations volumineuses

---

### Diagnostic Pas à Pas

#### Étape 1 : Analyser l'Utilisation Mémoire

```bash
# Mémoire globale du système
free -h
vmstat 1 10

# Mémoire MongoDB spécifique
ps aux | grep mongod | awk '{print $6}'  # RSS en KB

# Détails de la mémoire MongoDB
pmap -x $(pgrep mongod) | tail -1

# Vérifier le swapping
vmstat 1 5
# si: swap in, so: swap out
# Si > 0 : système utilise le swap (MAUVAIS)
```

```javascript
// Métriques mémoire MongoDB
var mem = db.serverStatus().mem
printjson(mem)
// {
//   bits: 64,
//   resident: 2048,    // Mémoire résidente (MB)
//   virtual: 4096,     // Mémoire virtuelle (MB)
//   mapped: 0,
//   mappedWithJournal: 0
// }

// Cache WiredTiger
var cache = db.serverStatus().wiredTiger.cache
printjson({
  "maximum bytes configured": cache["maximum bytes configured"],
  "bytes currently in cache": cache["bytes currently in the cache"],
  "pages evicted by application threads": cache["pages evicted by application threads"],
  "percentage overhead": cache["percentage overhead"]
})
```

#### Étape 2 : Identifier les Collections Volumineuses

```javascript
// Taille des collections
db.getCollectionNames().map(name => ({
  collection: name,
  size: db[name].stats().size / 1024 / 1024,  // MB
  storageSize: db[name].stats().storageSize / 1024 / 1024,  // MB
  count: db[name].count(),
  avgObjSize: db[name].stats().avgObjSize
})).sort((a, b) => b.size - a.size)

// Identifier les documents très volumineux
db.collection.find().forEach(doc => {
  var size = Object.bsonsize(doc)
  if (size > 1024 * 1024) {  // > 1MB
    print(doc._id + ": " + (size / 1024 / 1024) + " MB")
  }
})
```

#### Étape 3 : Analyser les Évictions de Cache

```javascript
// Statistiques d'éviction du cache
var cache = db.serverStatus().wiredTiger.cache

var evictionRate = cache["pages evicted by application threads"] /
                   cache["pages read into cache"]

print("Eviction rate: " + (evictionRate * 100).toFixed(2) + "%")

// Taux élevé (> 10%) = cache trop petit ou workload inadapté
```

#### Étape 4 : Vérifier les Requêtes Gourmandes

```javascript
// Requêtes retournant beaucoup de données
db.system.profile.find({
  "responseLength": {$gt: 16777216}  // > 16MB
}).sort({responseLength: -1}).forEach(doc => {
  printjson({
    ns: doc.ns,
    op: doc.op,
    responseLength: (doc.responseLength / 1024 / 1024).toFixed(2) + " MB",
    nreturned: doc.nreturned,
    millis: doc.millis
  })
})
```

---

### Résolution Pas à Pas

#### Solution 1 : Ajuster le Cache WiredTiger

```yaml
# /etc/mongod.conf
storage:
  wiredTiger:
    engineConfig:
      # Par défaut : 50% de (RAM - 1GB) ou 256MB
      # Recommandations :
      # - Serveur dédié : 50-60% de la RAM
      # - Serveur partagé : 30-40% de la RAM
      # - Laisser au moins 2-4GB pour l'OS
      cacheSizeGB: 4
```

**Calcul du cache optimal :**

```
RAM totale : 16GB
MongoDB dédié : cacheSizeGB = (16 - 2) * 0.5 = 7GB

RAM totale : 16GB
Serveur partagé : cacheSizeGB = (16 - 4) * 0.4 = 4.8GB
```

```bash
# Appliquer la configuration
sudo systemctl restart mongod

# Vérifier
mongosh --eval "db.serverStatus().wiredTiger.cache['maximum bytes configured']"
```

#### Solution 2 : Optimiser les Documents Volumineux

**Pattern Extended Reference :**

```javascript
// ❌ PROBLÈME : Documents énormes avec historique complet
{
  _id: ObjectId("..."),
  userId: 123,
  orders: [
    {orderId: 1, items: [...], total: 150, date: ISODate("...")},
    {orderId: 2, items: [...], total: 200, date: ISODate("...")},
    // ... 1000 orders
  ],
  reviews: [/* 500 reviews */],
  preferences: {/* data */}
}

// ✅ SOLUTION : Séparer en collections
// Collection users (petite)
{
  _id: 123,
  name: "John Doe",
  email: "john@example.com",
  preferences: {/* data */}
}

// Collection orders (séparée)
{
  _id: ObjectId("..."),
  userId: 123,
  items: [...],
  total: 150,
  date: ISODate("...")
}

// Index pour requêtes rapides
db.orders.createIndex({userId: 1, date: -1})
```

**Pattern Subset :**

```javascript
// Garder seulement un subset dans le document principal
{
  _id: 123,
  name: "John Doe",
  email: "john@example.com",
  recentOrders: [  // Seulement les 5 derniers
    {orderId: 998, total: 150, date: ISODate("...")},
    {orderId: 999, total: 200, date: ISODate("...")},
    {orderId: 1000, total: 180, date: ISODate("...")}
  ],
  totalOrders: 1000  // Compteur
}

// Historique complet dans collection séparée
```

#### Solution 3 : Implémenter la Pagination Correcte

```javascript
// ❌ PROBLÈME : Charger tous les résultats
const allUsers = await db.users.find({status: "active"}).toArray()
// Peut charger des millions de documents en mémoire !

// ✅ SOLUTION : Pagination avec cursor
async function processAllUsers() {
  const cursor = db.users.find({status: "active"}).batchSize(100)

  while (await cursor.hasNext()) {
    const user = await cursor.next()
    // Traiter un utilisateur à la fois
    await processUser(user)
  }
}

// ✅ SOLUTION : Pagination par batch
async function processUsersBatch() {
  const batchSize = 1000
  let lastId = null

  while (true) {
    const query = lastId
      ? {status: "active", _id: {$gt: lastId}}
      : {status: "active"}

    const batch = await db.users
      .find(query)
      .sort({_id: 1})
      .limit(batchSize)
      .toArray()

    if (batch.length === 0) break

    // Traiter le batch
    await processBatch(batch)

    lastId = batch[batch.length - 1]._id
  }
}
```

#### Solution 4 : Nettoyer les Données Obsolètes

```javascript
// TTL Index pour suppression automatique
db.sessions.createIndex(
  {createdAt: 1},
  {expireAfterSeconds: 3600}  // 1 heure
)

// Archivage des anciennes données
// Déplacer vers collection archive
db.orders.aggregate([
  {$match: {
    orderDate: {$lt: new Date('2023-01-01')}
  }},
  {$out: "orders_archive"}
])

// Supprimer de la collection principale
db.orders.deleteMany({
  orderDate: {$lt: new Date('2023-01-01')}
})

// Compacter la collection
db.runCommand({compact: "orders"})
```

#### Solution 5 : Limiter la Taille des Connexions

```javascript
// Node.js - Limiter le pool
const client = new MongoClient(uri, {
  maxPoolSize: 50,           // Max connexions (défaut: 100)
  minPoolSize: 10,           // Min connexions
  maxIdleTimeMS: 30000,      // Fermer après 30s idle
  maxConnecting: 2           // Max connexions simultanées
})

// Monitoring du pool
client.on('connectionCreated', (event) => {
  console.log('Connections:', event.connectionId)
})

client.on('connectionClosed', (event) => {
  console.log('Connection closed:', event.connectionId)
})
```

---

## 4. Goulots d'Étranglement I/O

### Symptômes

```
High disk I/O wait
Slow read/write operations
Disk queue depth > 10
iostat shows high %util
```

### Causes Possibles

- Disques lents (HDD vs SSD)
- RAID mal configuré
- Trop d'écritures simultanées
- Journal trop volumineux
- Snapshots fréquents
- Pas assez de RAM (cache insuffisant)

---

### Diagnostic Pas à Pas

#### Étape 1 : Analyser l'I/O Disque

```bash
# I/O par device
iostat -x 1 10

# Sortie importante :
# %util  : Utilisation du disque (> 80% = saturé)
# await  : Temps d'attente moyen (> 10ms = problème)
# r/s, w/s : Opérations lecture/écriture par seconde
# rMB/s, wMB/s : Débit

# I/O par processus
iotop -p $(pgrep mongod)

# Latence des I/O
ioping /var/lib/mongodb
```

```javascript
// Métriques I/O MongoDB
var wtStats = db.serverStatus().wiredTiger

printjson({
  "bytes read": wtStats.block_cache["bytes read into cache"],
  "bytes written": wtStats.block_cache["bytes written from cache"],
  "pages read": wtStats.data_handle["pages read into cache"],
  "pages written": wtStats.data_handle["pages written from cache"]
})
```

#### Étape 2 : Identifier les Opérations I/O-Intensives

```javascript
// Opérations avec beaucoup de lectures
db.system.profile.find({
  "docsExamined": {$gt: 100000}
}).sort({ts: -1})

// Opérations d'écriture volumineuses
db.system.profile.find({
  "op": {$in: ["insert", "update", "delete"]},
  "nreturned": {$gt: 1000}
})
```

#### Étape 3 : Analyser le Journal

```bash
# Taille du journal
du -sh /var/lib/mongodb/journal/

# Vérifier la configuration
mongosh --eval "db.serverStatus().dur"
```

---

### Résolution Pas à Pas

#### Solution 1 : Optimiser la Configuration Storage

```yaml
# /etc/mongod.conf
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100    # Défaut: 100ms (réduit l'I/O)

  wiredTiger:
    engineConfig:
      cacheSizeGB: 8
      journalCompressor: snappy  # Compression du journal
      directoryForIndexes: true   # Séparer indexes et données

    collectionConfig:
      blockCompressor: snappy      # Compression des données

    indexConfig:
      prefixCompression: true      # Compression des index
```

#### Solution 2 : Utiliser des Disques Plus Rapides

**Migration vers SSD :**

```bash
# 1. Arrêter MongoDB proprement
mongosh --eval "db.adminCommand({shutdown: 1})"

# 2. Copier les données vers le nouveau disque
rsync -av /var/lib/mongodb/ /mnt/ssd/mongodb/

# 3. Mettre à jour la configuration
# /etc/mongod.conf
storage:
  dbPath: /mnt/ssd/mongodb

# 4. Ajuster les permissions
chown -R mongodb:mongodb /mnt/ssd/mongodb

# 5. Redémarrer
systemctl start mongod
```

**Configuration RAID :**

```bash
# RAID 10 recommandé pour MongoDB
# - RAID 0 : Performance mais pas de redondance
# - RAID 1 : Redondance mais pas de performance
# - RAID 5/6 : MAUVAIS pour MongoDB (pénalité écriture)
# - RAID 10 : OPTIMAL (performance + redondance)

# Exemple création RAID 10 avec 4 disques
mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sdb /dev/sdc /dev/sdd /dev/sde
```

#### Solution 3 : Optimiser les Patterns d'Écriture

**Bulk Operations :**

```javascript
// ❌ LENT : Insertions individuelles
for (let i = 0; i < 10000; i++) {
  await db.logs.insertOne({
    timestamp: new Date(),
    message: `Log ${i}`
  })
}
// 10000 I/O writes !

// ✅ RAPIDE : Bulk insert
const docs = []
for (let i = 0; i < 10000; i++) {
  docs.push({
    timestamp: new Date(),
    message: `Log ${i}`
  })
}
await db.logs.insertMany(docs, {ordered: false})
// 1 I/O write (ou quelques-uns avec gros volumes)
```

**Write Concern :**

```javascript
// Write concern moins strict = moins d'I/O
await db.logs.insertMany(docs, {
  writeConcern: {
    w: 1,           // Acknowledge du primary seulement
    j: false        // Pas de journal (ATTENTION : risque de perte)
  }
})

// Pour données critiques : write concern strict
await db.orders.insertOne(order, {
  writeConcern: {
    w: "majority",  // Majorité du replica set
    j: true,        // Avec journal
    wtimeout: 5000  // Timeout 5s
  }
})
```

#### Solution 4 : Partitionnement I/O

**Séparer Journal et Données :**

```yaml
# /etc/mongod.conf
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true

  # WiredTiger peut séparer journal sur autre disque
  wiredTiger:
    engineConfig:
      # Journal sur disque rapide dédié
      journalPath: /mnt/journal-ssd
```

**Volumes séparés :**

```bash
# Monter les disques
/dev/sda1 -> /var/lib/mongodb       # Données (RAID 10)
/dev/sdb1 -> /var/log/mongodb       # Logs (disque séparé)
/dev/sdc1 -> /mnt/journal           # Journal (SSD)
```

#### Solution 5 : Monitorer et Alerter

```javascript
// Script de monitoring I/O
function checkIOPerformance() {
  const stats = db.serverStatus().wiredTiger.block_cache

  const readLatency = stats["bytes read into cache"] /
                      stats["reads"]

  if (readLatency > 10) {  // > 10ms moyen
    console.error(`HIGH I/O LATENCY: ${readLatency.toFixed(2)}ms`)
    // Envoyer alerte
  }
}

setInterval(checkIOPerformance, 60000)
```

---

## 5. Lock Contention

### Symptômes

```
Queries waiting for locks
High lock wait time
Write conflicts
Timeouts on writes
```

### Causes Possibles

- Transactions longues
- Index builds bloquants
- Opérations DDL (drop, rename)
- Write conflicts sur documents
- Lock escalation

---

### Diagnostic Pas à Pas

#### Étape 1 : Identifier les Locks

```javascript
// Voir les opérations en attente de locks
db.currentOp({
  "waitingForLock": true
})

// Détails des locks
db.currentOp().inprog.filter(op =>
  op.waitingForLock
).map(op => ({
  opid: op.opid,
  op: op.op,
  ns: op.ns,
  secs_running: op.secs_running,
  locks: op.locks
}))

// Statistiques de locks globales
db.serverStatus().locks
```

#### Étape 2 : Analyser les Transactions

```javascript
// Transactions actives
db.currentOp({
  "active": true,
  "transaction": {$exists: true}
})

// Transactions qui durent longtemps
db.currentOp({
  "transaction": {$exists: true},
  "transaction.timeActiveMicros": {$gt: 5000000}  // > 5s
})
```

---

### Résolution Pas à Pas

#### Solution 1 : Optimiser les Transactions

```javascript
// ❌ MAUVAIS : Transaction trop longue
const session = client.startSession()
session.startTransaction()

try {
  // Beaucoup d'opérations
  for (let i = 0; i < 10000; i++) {
    await db.orders.updateOne({_id: i}, {$set: {status: "processed"}}, {session})
  }

  await session.commitTransaction()
} catch (error) {
  await session.abortTransaction()
}

// ✅ BON : Transactions courtes et ciblées
const session = client.startSession()
session.startTransaction()

try {
  // Opérations limitées et rapides
  await db.accounts.updateOne(
    {_id: sourceId},
    {$inc: {balance: -amount}},
    {session}
  )

  await db.accounts.updateOne(
    {_id: targetId},
    {$inc: {balance: amount}},
    {session}
  )

  await session.commitTransaction()
} catch (error) {
  await session.abortTransaction()
}
```

#### Solution 2 : Index Builds Non-Bloquants

```javascript
// ❌ Bloquant (par défaut avant MongoDB 4.2)
db.users.createIndex({email: 1})

// ✅ Non-bloquant (background)
db.users.createIndex(
  {email: 1},
  {background: true}
)

// ✅ MongoDB 4.2+ : Hybrid index build (non-bloquant par défaut)
db.users.createIndex({email: 1})
// Ne bloque pas les lectures/écritures
```

#### Solution 3 : Réduire la Contention avec Sharding

```javascript
// Distribuer les écritures sur plusieurs shards
sh.shardCollection("mydb.orders", {userId: "hashed"})

// Les écritures vont sur différents shards
// Réduit la contention sur un seul serveur
```

---

## 6. Problèmes de Cache

### Symptômes

```
Cache hit rate < 90%
Frequent page evictions
Slow queries despite indexes
High I/O despite good cache size
```

### Diagnostic

```javascript
// Cache statistics
var cache = db.serverStatus().wiredTiger.cache

var hitRate = (cache["pages read into cache"] - cache["pages requested from the cache"]) /
              cache["pages read into cache"]

print("Cache hit rate: " + (hitRate * 100).toFixed(2) + "%")

// Évictions
print("Pages evicted:", cache["pages evicted by application threads"])
```

### Résolution

**Augmenter le cache ou optimiser le working set :**

```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 12  # Augmenter si possible
```

---

## 7. Problèmes d'Agrégation

### Diagnostic

```javascript
// Agrégations lentes
db.system.profile.find({
  "command.aggregate": {$exists: true},
  "millis": {$gt: 1000}
})
```

### Résolution

**Optimiser le pipeline :**

```javascript
// ❌ LENT : $lookup puis $match
db.orders.aggregate([
  {$lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customer"
  }},
  {$match: {"customer.country": "France"}}
])

// ✅ RAPIDE : $match avant $lookup
db.orders.aggregate([
  {$match: {country: "France"}},  // Filtrer d'abord
  {$lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customer"
  }}
])
```

---

## 8. Monitoring et Métriques

### Dashboard de Performance Essentiel

```javascript
// Script de monitoring complet
function performanceReport() {
  const stats = db.serverStatus()

  return {
    // Connexions
    connections: {
      current: stats.connections.current,
      available: stats.connections.available,
      usage: `${(stats.connections.current / (stats.connections.current + stats.connections.available) * 100).toFixed(1)}%`
    },

    // Opérations
    ops: {
      insert: stats.opcounters.insert,
      query: stats.opcounters.query,
      update: stats.opcounters.update,
      delete: stats.opcounters.delete
    },

    // Mémoire
    memory: {
      resident: `${stats.mem.resident} MB`,
      virtual: `${stats.mem.virtual} MB`
    },

    // Cache
    cache: {
      size: `${(stats.wiredTiger.cache["bytes currently in the cache"] / 1024 / 1024).toFixed(0)} MB`,
      maxConfig: `${(stats.wiredTiger.cache["maximum bytes configured"] / 1024 / 1024).toFixed(0)} MB`,
      usage: `${(stats.wiredTiger.cache["bytes currently in the cache"] / stats.wiredTiger.cache["maximum bytes configured"] * 100).toFixed(1)}%`
    },

    // Performance
    performance: {
      activeReads: stats.globalLock.activeClients.readers,
      activeWrites: stats.globalLock.activeClients.writers,
      queuedReads: stats.globalLock.currentQueue.readers,
      queuedWrites: stats.globalLock.currentQueue.writers
    }
  }
}

print(JSON.stringify(performanceReport(), null, 2))
```

### Alertes Recommandées

```javascript
// Seuils d'alerte
const THRESHOLDS = {
  connectionUsage: 80,      // %
  cacheUsage: 90,           // %
  queueDepth: 10,           // opérations
  slowQuery: 1000,          // ms
  lockWaitTime: 5000        // ms
}

function checkAlerts() {
  const report = performanceReport()
  const alerts = []

  if (parseFloat(report.connections.usage) > THRESHOLDS.connectionUsage) {
    alerts.push(`HIGH CONNECTION USAGE: ${report.connections.usage}`)
  }

  if (parseFloat(report.cache.usage) > THRESHOLDS.cacheUsage) {
    alerts.push(`HIGH CACHE USAGE: ${report.cache.usage}`)
  }

  if (report.performance.queuedReads + report.performance.queuedWrites > THRESHOLDS.queueDepth) {
    alerts.push(`HIGH QUEUE DEPTH: ${report.performance.queuedReads + report.performance.queuedWrites}`)
  }

  return alerts
}
```

---

## Checklist Globale de Performance

### Diagnostic Rapide (5 minutes)

```bash
# 1. Opérations lentes actuelles
mongosh --eval "db.currentOp({secs_running: {\$gt: 5}})"

# 2. Top 10 requêtes lentes (profiler)
mongosh --eval "db.system.profile.find().sort({millis: -1}).limit(10)"

# 3. Utilisation ressources
top -p $(pgrep mongod)
iostat -x 1 3

# 4. Métriques MongoDB
mongosh --eval "db.serverStatus().opcounters"
mongosh --eval "db.serverStatus().connections"

# 5. Cache hit rate
mongosh --eval "var c = db.serverStatus().wiredTiger.cache; print('Hit rate: ' + ((1 - c['pages read into cache'] / c['pages requested from the cache']) * 100).toFixed(2) + '%')"
```

### Optimisation Progressive

```
Phase 1: Quick Wins (1 jour)
□ Créer les index manquants évidents
□ Ajouter projections aux requêtes
□ Limiter les résultats non paginés
□ Activer le profiler

Phase 2: Optimisation Moyenne (1 semaine)
□ Revoir la modélisation des données
□ Optimiser les agrégations
□ Implémenter le caching applicatif
□ Ajuster les paramètres WiredTiger

Phase 3: Architecture (1 mois)
□ Évaluer le sharding
□ Implémenter le read splitting
□ Optimiser l'infrastructure (SSD, RAM)
□ Mettre en place monitoring avancé
```

---

## Conclusion

Les problèmes de performance MongoDB nécessitent une approche systématique :

1. **Mesurer** avec précision (profiler, explain, metrics)
2. **Identifier** les goulots (CPU, mémoire, I/O, requêtes)
3. **Optimiser** de manière ciblée (index, queries, config)
4. **Valider** l'amélioration (avant/après)
5. **Monitorer** en continu

**Points clés :**
- ✅ Index appropriés sur tous les champs de requête/tri
- ✅ Projections pour limiter les données transférées
- ✅ Pagination correcte (pas de skip)
- ✅ Cache WiredTiger bien dimensionné
- ✅ Monitoring proactif avec alertes

---


⏭️ [Problèmes de réplication](/22-depannage-resolution-problemes/03-problemes-replication.md)
