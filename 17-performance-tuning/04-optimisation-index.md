🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.4 Optimisation des Index

## Introduction

Les index sont l'outil le plus puissant pour améliorer les performances de lecture dans MongoDB, mais ils représentent également un compromis complexe : chaque index améliore certaines requêtes tout en pénalisant les écritures et en consommant des ressources précieuses. L'art de l'optimisation des index réside dans l'identification du jeu d'index minimal qui maximise les performances globales du système.

Cette section explore les méthodologies expertes pour concevoir, analyser, maintenir et optimiser les index en environnement de production, où chaque décision a un impact mesurable sur les performances et les coûts opérationnels.

## Principes Fondamentaux de l'Indexation

### Coût Réel des Index

Chaque index a un coût qui doit être quantifié et justifié.

**Coûts directs** :

```javascript
// Analyse du coût d'un index
const indexStats = db.collection.aggregate([
  { $indexStats: {} }
]).toArray();

indexStats.forEach(index => {
  const cost = {
    // Stockage disque
    diskUsage: index.size,  // Bytes

    // RAM (si index resident)
    ramUsage: index.size,  // Idéalement tout l'index en RAM

    // Coût écriture (estimation)
    // Chaque insert/update touche tous les index
    writePenalty: "~10-15% par index",

    // Coût maintenance
    fragmentationRisk: index.accesses.ops < 100 ? "HIGH" : "LOW"
  };

  print(`Index ${index.name}:`);
  printjson(cost);
});
```

**Formule de coût total** :
```
IndexCost = StorageCost + WritePenalty + MaintenanceCost + OpportunityCost

Où :
- StorageCost = Size × StoragePrice
- WritePenalty = WriteOps/sec × IndexCount × 0.12ms
- MaintenanceCost = RebuildFrequency × RebuildDuration × DowntimeCost
- OpportunityCost = RAMUsed × RAMPrice (RAM non disponible pour le cache)
```

**Règle de décision** :
```javascript
// Un index est justifié si :
ReadImprovement × ReadFrequency > IndexCost

// Exemple :
// Sans index : 500ms × 100 queries/s = 50,000ms/s de latence
// Avec index : 5ms × 100 queries/s = 500ms/s de latence
// Gain : 49,500ms/s
//
// Coût index :
// - Write penalty : 1000 writes/s × 0.12ms = 120ms/s
// - Storage : 500MB × $0.10/GB/month = $0.05/month (négligeable)
//
// ROI : 49,500 / 120 = 412× → Index largement justifié
```

### Index Selectivity

La sélectivité d'un index est cruciale pour son efficacité.

**Définition** :
```javascript
selectivity = uniqueValues / totalDocuments

// Interprétation :
// 1.0 (100%) : Parfaite (ex: _id, email unique)
// 0.5 (50%)  : Bonne (ex: userId dans une collection de commandes)
// 0.1 (10%)  : Moyenne (ex: status avec 10 valeurs possibles)
// 0.01 (1%)  : Faible (ex: boolean field)
// 0.001 (<1%): Très faible (ex: isDeleted où 99.9% sont false)
```

**Calcul de sélectivité** :
```javascript
function calculateSelectivity(collection, field) {
  const total = db[collection].countDocuments();
  const distinct = db[collection].distinct(field).length;

  const selectivity = distinct / total;

  return {
    field: field,
    totalDocs: total,
    distinctValues: distinct,
    selectivity: selectivity,
    recommendation: getRecommendation(selectivity)
  };
}

function getRecommendation(selectivity) {
  if (selectivity > 0.5) return "Excellent candidate for index";
  if (selectivity > 0.1) return "Good candidate for index";
  if (selectivity > 0.01) return "Consider compound index or partial index";
  return "Poor selectivity - consider alternatives";
}

// Usage
const result = calculateSelectivity("orders", "status");
printjson(result);
```

**Impact sur les performances** :
```javascript
// Index sur champ à faible sélectivité
db.users.find({ isPremium: true })  // isPremium: 1% true, 99% false

// Sans index : COLLSCAN
// - Examine : 1,000,000 documents
// - Returns : 10,000 documents (1%)
// - Time : ~2000ms

// Avec index sur isPremium
// - Examine : 10,000 index entries
// - Returns : 10,000 documents
// - Time : ~50ms
// Amélioration : 40× (justifie l'index malgré faible sélectivité)

// Mais...
db.users.find({ isPremium: false })  // 99% des documents
// Avec index : Moins efficace qu'un COLLSCAN
// Query planner peut choisir COLLSCAN automatiquement
```

### Working Set et Index Residency

**Principe cardinal** :
> Tous les index fréquemment utilisés doivent tenir en RAM pour des performances optimales.

**Calcul du working set d'index** :
```javascript
// 1. Taille totale des index
const stats = db.collection.stats();
const totalIndexSize = stats.totalIndexSize;

// 2. RAM disponible pour les index
const serverStatus = db.serverStatus();
const cacheSize = serverStatus.wiredTiger.cache["maximum bytes configured"];
const cacheCurrent = serverStatus.wiredTiger.cache["bytes currently in the cache"];

// 3. Ratio d'index residency
const indexResidency = cacheCurrent / totalIndexSize;

print(`Index Residency: ${(indexResidency * 100).toFixed(2)}%`);

if (indexResidency < 0.9) {
  print("WARNING: Indexes not fully resident in RAM");
  print("Consider:");
  print("- Increasing cache size");
  print("- Removing unused indexes");
  print("- Using partial indexes to reduce size");
}
```

**Impact du cache miss** :
```
Index fully in RAM : ~0.1ms per lookup
Index on SSD      : ~1-5ms per lookup
Index on HDD      : ~10-50ms per lookup

Multiplier pour requête : 10-500×
```

## Stratégies d'Optimisation des Index Composés

### La Règle ESR (Equality, Sort, Range)

Pour les index composés, l'ordre des champs est critique.

**Principe ESR** :
```
1. Equality : Champs avec conditions d'égalité (=, $in)
2. Sort : Champs de tri
3. Range : Champs avec range queries (>, <, $gte, $lte)
```

**Exemple illustratif** :
```javascript
// Requête type
db.orders.find({
  status: "completed",      // Equality
  customerId: { $in: [...] }, // Equality ($in)
  total: { $gte: 100 }      // Range
}).sort({
  orderDate: -1              // Sort
})

// Analyse des différents ordres d'index :

// ❌ BAD : Range avant Sort
db.orders.createIndex({ total: 1, orderDate: -1, status: 1, customerId: 1 })
// - Range scan sur total : Examine beaucoup de documents
// - Sort doit s'appliquer après → In-memory sort ou partial index usage
// - Equality filters appliqués après → Post-filtering inefficace

// ⚠️ MEDIOCRE : Sort avant Equality
db.orders.createIndex({ orderDate: -1, status: 1, customerId: 1, total: 1 })
// - Scan tout l'index trié par date
// - Filter sur status ensuite → Examine beaucoup de documents

// ✅ OPTIMAL : ESR order
db.orders.createIndex({
  status: 1,        // E: Equality first
  customerId: 1,    // E: Equality (even with $in)
  orderDate: -1,    // S: Sort
  total: 1          // R: Range last
})
// - Index scan directement aux documents status="completed"
// - Filtre customerId dans ce subset
// - Déjà trié par orderDate (pas de sort stage)
// - Range filter sur total appliqué last
```

**Validation avec explain()** :
```javascript
const explain = db.orders.find({
  status: "completed",
  customerId: { $in: [id1, id2, id3] },
  total: { $gte: 100 }
}).sort({ orderDate: -1 }).explain("executionStats");

// Métriques à vérifier :
const metrics = {
  // Pas de SORT stage
  hasInMemorySort: explain.executionStats.executionStages.stage === "SORT",

  // Index bounds utilisés correctement
  indexBounds: explain.executionStats.executionStages.inputStage.indexBounds,

  // Ratio efficiency
  examined: explain.executionStats.totalKeysExamined,
  returned: explain.executionStats.nReturned,
  efficiency: returned / examined  // Doit être > 0.1
};

printjson(metrics);
```

### Cas Particuliers et Exceptions ESR

#### Exception 1 : Faible Cardinalité Equality

```javascript
// Requête
db.products.find({
  isActive: true,      // Equality mais faible cardinalité (50/50)
  category: "Electronics", // Equality haute cardinalité
  price: { $gte: 100, $lte: 500 }  // Range
}).sort({ rating: -1 })

// ESR strict suggèrerait : { isActive: 1, category: 1, rating: -1, price: 1 }
// Mais isActive a faible sélectivité

// MEILLEUR : Commencer par haute sélectivité
db.products.createIndex({
  category: 1,     // Haute sélectivité first
  isActive: 1,     // Faible sélectivité after
  rating: -1,      // Sort
  price: 1         // Range
})
```

**Règle affinée** :
```
1. High-selectivity Equality
2. Low-selectivity Equality
3. Sort
4. Range
```

#### Exception 2 : Sort très sélectif

```javascript
// Requête : Top 10 des dernières commandes d'un user
db.orders.find({
  userId: ObjectId("...")  // Equality
}).sort({
  orderDate: -1            // Sort
}).limit(10)               // Early termination possible

// ESR suggère : { userId: 1, orderDate: -1 }
// C'est correct !

// Mais si on ajoute un range :
db.orders.find({
  userId: ObjectId("..."),
  total: { $gte: 100 }     // Range
}).sort({
  orderDate: -1
}).limit(10)

// ESR suggère : { userId: 1, orderDate: -1, total: 1 }
// Le range est APRÈS le sort pour permettre early termination
```

**Principe** :
Si LIMIT est présent avec SORT, privilégier le sort tôt pour permettre l'early termination.

### Index Prefix Utilization

Un index composé peut servir pour les requêtes utilisant ses préfixes.

```javascript
// Index composé
db.collection.createIndex({ a: 1, b: 1, c: 1, d: 1 })

// Préfixes utilisables :
// { a: 1 }
// { a: 1, b: 1 }
// { a: 1, b: 1, c: 1 }
// { a: 1, b: 1, c: 1, d: 1 }

// Requêtes supportées :
db.collection.find({ a: 1 })                    // ✅ Prefix { a: 1 }
db.collection.find({ a: 1, b: 2 })              // ✅ Prefix { a: 1, b: 1 }
db.collection.find({ a: 1, b: 2, c: 3 })        // ✅ Prefix { a: 1, b: 1, c: 1 }
db.collection.find({ a: 1, c: 3 })              // ⚠️ Partial use { a: 1 } only
db.collection.find({ b: 2 })                    // ❌ No prefix match
db.collection.find({ a: 1, b: 2, d: 4 })        // ⚠️ Prefix { a: 1, b: 1 } only

// Sort supportés :
db.collection.find({ a: 1 }).sort({ b: 1 })           // ✅
db.collection.find({ a: 1 }).sort({ b: 1, c: 1 })    // ✅
db.collection.find({ a: 1, b: 2 }).sort({ c: 1 })    // ✅
db.collection.find({ a: 1 }).sort({ c: 1 })          // ❌ Gap in prefix
```

**Stratégie de consolidation** :

```javascript
// Avant : Index redondants
db.collection.createIndex({ userId: 1 })
db.collection.createIndex({ userId: 1, status: 1 })
db.collection.createIndex({ userId: 1, status: 1, createdAt: -1 })

// Après : Un seul index optimal
db.collection.createIndex({ userId: 1, status: 1, createdAt: -1 })

// Couvre toutes les requêtes :
// - { userId } → Prefix
// - { userId, status } → Prefix
// - { userId, status, createdAt } → Full

// Économies :
// - Storage : -66% (3 index → 1 index)
// - Write penalty : -66% (3 updates → 1 update)
// - Maintenance : -66% (3 rebuilds → 1 rebuild)
```

### Covered Queries : Index Couvrant

Une covered query est satisfaite entièrement par l'index sans accès au document.

**Conditions pour une covered query** :
1. Tous les champs du `find()` sont dans l'index
2. Tous les champs de la `projection` sont dans l'index
3. Aucun champ dans le query est un array (index multikey incompatible)
4. `_id` doit être explicitement exclu si pas dans l'index

**Exemple optimal** :
```javascript
// Index
db.users.createIndex({
  email: 1,
  status: 1,
  lastLogin: -1,
  _id: 1  // Include _id explicitement pour covering
})

// Covered query
db.users.find(
  {
    email: "user@example.com",
    status: "active"
  },
  {
    email: 1,
    status: 1,
    lastLogin: 1,
    _id: 0  // IMPORTANT : Exclure _id si pas utilisé
  }
)

// explain() montre :
// - stage: "IXSCAN" (pas de FETCH)
// - totalDocsExamined: 0 (aucun document accédé)
// - Latency : ~0.5ms vs ~5ms avec FETCH
```

**Gains de performance** :
```
Non-covered query :
1. Index scan : 0.5ms
2. FETCH documents : 4.5ms
Total : 5ms

Covered query :
1. Index scan : 0.5ms
Total : 0.5ms

Amélioration : 10× pour queries simples
```

**Stratégie de conception** :
```javascript
// Analyser les requêtes fréquentes pour identifier opportunités
db.system.profile.aggregate([
  {
    $match: {
      "command.find": { $exists: true },
      millis: { $gt: 10 }  // Requêtes > 10ms
    }
  },
  {
    $project: {
      collection: 1,
      filter: "$command.filter",
      projection: "$command.projection",
      docsExamined: 1
    }
  },
  {
    $match: {
      docsExamined: { $gt: 0 }  // Potential covering candidates
    }
  }
])

// Pour chaque query fréquente, évaluer si un covering index est possible
```

## Index Partiels et Sparse : Optimisation de la Taille

### Partial Index : Indexer un Sous-Ensemble

Les partial indexes réduisent drastiquement la taille en indexant uniquement les documents pertinents.

**Cas d'usage classique** : Status flags

```javascript
// Scénario : 95% des orders sont "completed", 5% sont "pending" ou "processing"
// Queries fréquentes : Rechercher orders actives (non-completed)

// Mauvais : Index complet
db.orders.createIndex({ status: 1 })
// - Index 100% des documents
// - Taille : Large
// - 95% de l'index rarement utilisé

// Optimal : Partial index
db.orders.createIndex(
  { status: 1, createdAt: -1 },
  {
    partialFilterExpression: {
      status: { $ne: "completed" }
    },
    name: "idx_active_orders"
  }
)

// - Index seulement 5% des documents
// - Taille : 95% plus petit
// - Queries sur orders actives : Très rapide
// - RAM savings : Significatif
```

**Attention : Utilisation limitée**
```javascript
// ✅ Query utilise le partial index
db.orders.find({
  status: "pending",      // Correspond au filter
  createdAt: { $gte: date }
})

// ✅ Query utilise le partial index
db.orders.find({
  status: { $in: ["pending", "processing"] },  // Subset du filter
  createdAt: { $gte: date }
})

// ❌ Query N'utilise PAS le partial index
db.orders.find({
  status: "completed"     // Exclut par le filter
})

// ❌ Query N'utilise PAS le partial index
db.orders.find({
  createdAt: { $gte: date }  // Pas de condition sur status
})
```

**Stratégies avancées** :

**1. Date-based partial index** :
```javascript
// Indexer seulement les données récentes (hot data)
db.logs.createIndex(
  { level: 1, timestamp: -1, message: "text" },
  {
    partialFilterExpression: {
      timestamp: {
        $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)  // 30 jours
      }
    },
    name: "idx_recent_logs"
  }
)

// Combiné avec TTL pour cleanup automatique
db.logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 90 * 24 * 60 * 60 }  // 90 jours
)
```

**2. Multi-condition partial index** :
```javascript
// Indexer seulement les documents premium actifs
db.users.createIndex(
  { email: 1, lastActivity: -1 },
  {
    partialFilterExpression: {
      $and: [
        { isPremium: true },
        { isActive: true },
        { lastActivity: { $gte: new Date("2024-01-01") } }
      ]
    },
    name: "idx_premium_active_users"
  }
)

// Taille : ~5% de la collection vs 100%
// Performance : Identique pour queries matching le filter
// RAM savings : 95%
```

### Sparse Index : Ignorer les Null

Les sparse indexes excluent les documents où le champ indexé est absent ou null.

```javascript
// Cas : Champ optionnel utilisé rarement
// Ex: 5% des users ont un referralCode

// Sparse index
db.users.createIndex(
  { referralCode: 1 },
  { sparse: true }
)

// Effet :
// - Index seulement les 5% avec referralCode
// - Taille : 95% plus petit
// - Queries sur referralCode : Rapides

// ⚠️ ATTENTION : Comportement subtil
db.users.find({ referralCode: { $exists: false } })
// N'utilisera PAS le sparse index (logique : ces docs ne sont pas indexés)

db.users.find({ referralCode: "ABC123" })
// Utilisera le sparse index (ces docs sont indexés)
```

**Différence Sparse vs Partial** :

| Critère | Sparse | Partial |
|---------|--------|---------|
| Condition | Automatique (null/absent) | Explicite (expression) |
| Flexibilité | Limitée | Complète |
| Cas d'usage | Champs optionnels | Filtrages complexes |
| Unique + Sparse | Permet multiple null | N/A |

**Combinaison Sparse + Unique** :
```javascript
// Use case : Email optionnel mais unique si présent
db.profiles.createIndex(
  { email: 1 },
  {
    unique: true,
    sparse: true
  }
)

// Comportement :
// - Multiple documents peuvent avoir email: null ou absent
// - Mais chaque email non-null doit être unique
// - Index très petit si peu de users ont un email
```

## Audit et Maintenance des Index

### Identification des Index Inutilisés

```javascript
// Collecter les statistiques d'utilisation
const indexStats = db.collection.aggregate([
  { $indexStats: {} }
]).toArray();

// Analyser l'utilisation
const analysis = indexStats.map(index => {
  const daysSinceLastUse = index.accesses.since
    ? (Date.now() - index.accesses.since.getTime()) / (1000 * 60 * 60 * 24)
    : Infinity;

  return {
    name: index.name,
    accesses: index.accesses.ops,
    lastUsed: index.accesses.since,
    daysSinceLastUse: daysSinceLastUse,
    size: index.size || 0,
    recommendation: getIndexRecommendation(index.accesses.ops, daysSinceLastUse)
  };
}).sort((a, b) => a.accesses - b.accesses);

function getIndexRecommendation(accesses, daysSinceLastUse) {
  if (accesses === 0 && daysSinceLastUse > 30) {
    return "DROP - Never used in 30 days";
  }
  if (accesses < 10 && daysSinceLastUse > 7) {
    return "CONSIDER DROP - Rarely used";
  }
  if (accesses < 100) {
    return "MONITOR - Low usage";
  }
  return "KEEP - Actively used";
}

printjson(analysis);

// Automatisation : Script de cleanup
const indexesToDrop = analysis.filter(
  idx => idx.recommendation === "DROP - Never used in 30 days"
);

indexesToDrop.forEach(idx => {
  if (idx.name !== "_id_") {  // Never drop _id index
    print(`Dropping unused index: ${idx.name}`);
    // db.collection.dropIndex(idx.name);  // Uncomment to execute
  }
});
```

### Index Consolidation Strategy

```javascript
// Analyser les queries pour identifier les overlaps
db.system.profile.aggregate([
  {
    $match: {
      "command.find": { $exists: true },
      ns: "mydb.orders"
    }
  },
  {
    $group: {
      _id: {
        filter: "$command.filter",
        sort: "$command.sort"
      },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 20 }
])

// Exemple de résultat :
// 1. { userId: X } + sort { createdAt: -1 } : 10k/day
// 2. { userId: X, status: Y } : 5k/day
// 3. { userId: X, status: Y } + sort { createdAt: -1 } : 3k/day

// Index actuels (sous-optimaux) :
// - { userId: 1 }
// - { userId: 1, createdAt: -1 }
// - { userId: 1, status: 1 }
// - { status: 1, createdAt: -1 }

// Index consolidé optimal :
// { userId: 1, status: 1, createdAt: -1 }
// Couvre queries 1, 2, 3 via prefix utilization

// Migration :
// 1. Créer le nouvel index
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 })

// 2. Monitorer les performances (quelques jours)

// 3. Drop les anciens index si performances OK
// db.orders.dropIndex({ userId: 1, createdAt: -1 })
// db.orders.dropIndex({ userId: 1, status: 1 })
```

### Fragmentation et Rebuild

Les index peuvent se fragmenter avec le temps, impactant les performances.

**Détection de fragmentation** :
```javascript
const stats = db.collection.stats();

// Métriques de fragmentation
const fragmentation = {
  collection: stats.ns,

  // Taille logique vs physique
  dataSize: stats.size,
  storageSize: stats.storageSize,
  dataFragmentation: ((stats.storageSize - stats.size) / stats.size * 100).toFixed(2) + '%',

  // Index size
  totalIndexSize: stats.totalIndexSize,
  indexSizes: stats.indexSizes,

  // Indicateur de fragmentation
  // Si storageSize >> size, fragmentation possible
  needsCompaction: (stats.storageSize / stats.size) > 1.5
};

printjson(fragmentation);

if (fragmentation.needsCompaction) {
  print("Consider running compact or reindexing");
}
```

**Stratégies de défragmentation** :

**1. Compact (collection entière)** :
```javascript
// ⚠️ LOCKS la collection, à faire en maintenance window
db.runCommand({ compact: "collection" })

// Ou avec options
db.runCommand({
  compact: "collection",
  force: true,
  // Note : Peut prendre plusieurs heures sur grandes collections
})
```

**2. Reindex (tous les index)** :
```javascript
// ⚠️ LOCKS la collection pendant le rebuild
db.collection.reIndex()

// Alternative : Rebuild index par index
db.collection.dropIndex("index_name")
db.collection.createIndex({ field: 1 }, { name: "index_name" })
```

**3. Rolling index rebuild (replica set)** :

Stratégie sans downtime pour replica sets :

```javascript
// Processus sur chaque secondary puis primary :
// 1. Connexion au secondary
mongo --host secondary1:27017

// 2. Stop la réplication
rs.secondaryOk()
db.adminCommand({ fsync: 1, lock: true })

// 3. Rebuild indexes
db.collection.reIndex()

// 4. Resume réplication
db.adminCommand({ fsyncUnlock: 1 })

// 5. Attendre rattrapage du replication lag
// 6. Répéter pour autres secondaries
// 7. Stepdown primary, répéter sur l'ancien primary
```

**4. Background rebuild (MongoDB 4.2+)** :
```javascript
// Rebuild en background (pas de lock complet)
db.collection.createIndex(
  { field: 1 },
  {
    background: true,
    name: "new_index_name"
  }
)

// Note : Plus lent mais n'impacte pas les lectures/écritures
// Une fois créé, drop l'ancien et rename
```

### Monitoring de Performance des Index

**Métriques continues à surveiller** :

```javascript
// Script de monitoring (à exécuter périodiquement)
function monitorIndexPerformance(collection) {
  const stats = db[collection].stats();
  const indexStats = db[collection].aggregate([{ $indexStats: {} }]).toArray();

  const report = {
    timestamp: new Date(),
    collection: collection,

    // Métriques globales
    totalDocs: stats.count,
    totalIndexSize: stats.totalIndexSize,
    avgDocSize: stats.avgObjSize,

    // Ratio index/data
    indexToDataRatio: (stats.totalIndexSize / stats.size).toFixed(2),

    // Par index
    indexes: indexStats.map(idx => ({
      name: idx.name,
      size: idx.size || 0,
      usageCount: idx.accesses.ops,
      usageRate: (idx.accesses.ops / ((Date.now() - idx.accesses.since) / 1000)).toFixed(2) + " ops/sec",

      // Health indicators
      isHealthy: idx.accesses.ops > 100,
      recommendation: idx.accesses.ops === 0 ? "Consider dropping" : "Keep"
    }))
  };

  // Alertes
  if (report.indexToDataRatio > 1) {
    report.alert = "Index size > data size - Review index strategy";
  }

  if (report.totalIndexSize > 10 * 1024 * 1024 * 1024) {  // 10GB
    report.alert = "Large index size - Ensure RAM capacity";
  }

  return report;
}

// Exécution et logging
const report = monitorIndexPerformance("orders");
printjson(report);

// Export vers système de monitoring
// sendToPrometheus(report);
// sendToDatadog(report);
```

## Index Build Strategies en Production

### Background vs Foreground Builds

**Foreground build** (défaut avant MongoDB 4.2) :
```javascript
db.collection.createIndex({ field: 1 })

// Caractéristiques :
// - Prend un WRITE LOCK sur la collection
// - Très rapide (5-10× plus rapide que background)
// - Bloque toutes les écritures et lectures
// - Adapté seulement en maintenance window
```

**Background build** :
```javascript
db.collection.createIndex(
  { field: 1 },
  { background: true }
)

// Caractéristiques :
// - Pas de lock exclusif
// - Plus lent (peut prendre heures sur grandes collections)
// - Yields périodiquement pour permettre autres opérations
// - Adapté pour production sans downtime
```

**MongoDB 4.2+ : Hybrid build** :
```javascript
// Nouveau comportement par défaut (depuis 4.2)
db.collection.createIndex({ field: 1 })

// Caractéristiques :
// - Commence en "background-like"
// - Prend un SHORT exclusive lock à la fin pour finaliser
// - Meilleur compromis : rapide + minimal downtime
// - Lock final : généralement < 1 seconde
```

### Rolling Index Build (Zero Downtime)

Pour les replica sets, stratégie de build sans impact :

```javascript
// ÉTAPE 1 : Build sur tous les secondaries
// Connexion à secondary1
const secondary1 = connect("secondary1:27017/admin");
secondary1.auth("admin", "password");

// Build index en background
secondary1.getSiblingDB("mydb").collection.createIndex(
  { field: 1 },
  { background: true }
)

// Attendre completion
while (secondary1.getSiblingDB("mydb").currentOp({
  "command.createIndexes": "collection"
}).inprog.length > 0) {
  sleep(5000);
  print("Index build in progress...");
}

// Répéter pour secondary2, secondary3, etc.

// ÉTAPE 2 : Stepdown du primary
rs.stepDown(120)  // 120 seconds

// ÉTAPE 3 : Build sur l'ancien primary (maintenant secondary)
// Une fois nouveau primary élu, build sur l'ancien primary

// ÉTAPE 4 : Validation
// Tous les membres ont maintenant l'index
rs.status().members.forEach(member => {
  const conn = connect(member.name + "/mydb");
  const indexes = conn.collection.getIndexes();
  print(`${member.name}: ${indexes.length} indexes`);
});
```

### Index Build sur Cluster Shardé

Sur un cluster shardé, l'index doit être créé sur tous les shards.

```javascript
// Connexion au mongos
const mongos = connect("mongos:27017/admin");

// Créer l'index (propagé automatiquement à tous les shards)
mongos.getSiblingDB("mydb").collection.createIndex(
  { field: 1 },
  { background: true }
)

// Monitoring de la progression sur chaque shard
sh.status().shards.forEach(shard => {
  const shardConn = connect(shard.host + "/mydb");

  const inprogOps = shardConn.currentOp({
    "command.createIndexes": "collection"
  }).inprog;

  if (inprogOps.length > 0) {
    print(`Shard ${shard._id}: Index build in progress`);
    printjson(inprogOps[0]);
  } else {
    print(`Shard ${shard._id}: Index build complete or not started`);
  }
});

// Note : La commande peut retourner avant que tous les shards
// aient fini le build. Toujours vérifier l'état.
```

**Stratégie optimale pour sharded clusters** :

```javascript
// 1. Build sur un shard à la fois pour contrôler la charge
// Désactiver le balancer temporairement
sh.stopBalancer()

// 2. Build shard par shard
sh.status().shards.forEach(shard => {
  print(`Building index on shard: ${shard._id}`);

  const shardConn = connect(shard.host + "/mydb");
  shardConn.collection.createIndex(
    { field: 1 },
    { background: true }
  );

  // Attendre completion
  while (shardConn.currentOp({
    "command.createIndexes": "collection"
  }).inprog.length > 0) {
    sleep(10000);
  }

  print(`Shard ${shard._id} complete`);
});

// 3. Réactiver le balancer
sh.startBalancer()
```

## Optimisations Avancées

### Index Intersection vs Compound Index

MongoDB peut utiliser plusieurs index via intersection (depuis 2.6).

```javascript
// Scénario :
db.products.find({
  category: "Electronics",
  brand: "Samsung"
})

// Option A : Deux index simples
db.products.createIndex({ category: 1 })
db.products.createIndex({ brand: 1 })

// Query planner peut utiliser intersection :
// 1. Scan category index → Set A (ex: 10k docs)
// 2. Scan brand index → Set B (ex: 5k docs)
// 3. Intersection A ∩ B → Result (ex: 500 docs)

// explain() montre :
"stage": "AND_SORTED"  // ou "AND_HASH"
"inputStages": [
  { "stage": "IXSCAN", "indexName": "category_1" },
  { "stage": "IXSCAN", "indexName": "brand_1" }
]

// Option B : Compound index
db.products.createIndex({ category: 1, brand: 1 })

// Query planner utilise compound index :
// 1. Direct scan sur { category: X, brand: Y } → Result (500 docs)

// explain() montre :
"stage": "IXSCAN"
"indexName": "category_1_brand_1"
```

**Performance comparison** :

| Métrique | Index Intersection | Compound Index |
|----------|-------------------|----------------|
| Index scans | 2 | 1 |
| Index entries examined | 15,000 (10k + 5k) | 500 |
| Latency | ~10ms | ~2ms |
| Memory | Higher (merge sets) | Lower |
| **Recommendation** | Avoid | **Preferred** |

**Quand l'intersection est acceptable** :
- Queries très rares et spécifiques
- Contraintes sur nombre d'index (limite atteinte)
- Chaque index simple utilisé fréquemment seul

### Wildcard Index : Flexible mais Coûteux

Les wildcard indexes (MongoDB 4.2+) indexent tous les champs ou sous-champs.

```javascript
// Index tous les champs
db.collection.createIndex({ "$**": 1 })

// Index tous les sous-champs d'un objet
db.collection.createIndex({ "attributes.$**": 1 })

// Cas d'usage : Schémas très dynamiques
// Ex: Attributs produits variables
{
  _id: ObjectId("..."),
  name: "Product",
  attributes: {
    color: "blue",
    size: "large",
    weight: "1kg",
    // ... dizaines d'attributs variables
  }
}

// Queries supportées :
db.collection.find({ "attributes.color": "blue" })
db.collection.find({ "attributes.size": "large" })
// Sans créer un index pour chaque attribut possible
```

**Trade-offs** :

**Avantages** :
- Flexibilité maximale
- Pas besoin de prévoir les queries
- Un seul index pour multiples champs

**Inconvénients** :
- Taille d'index très large (tous les champs)
- Performance inférieure aux index spécifiques
- Coût écriture élevé (update de n'importe quel champ)
- Ne supporte pas les compound queries efficacement

**Recommandation** :
```javascript
// ❌ À éviter en production générale
db.collection.createIndex({ "$**": 1 })

// ✅ Acceptable pour cas spécifiques
db.collection.createIndex(
  { "attributes.$**": 1 },
  {
    wildcardProjection: {
      "attributes.internalField": 0  // Exclure certains champs
    }
  }
)

// ✅ Meilleur : Index spécifiques basés sur query patterns
db.collection.createIndex({ "attributes.color": 1 })
db.collection.createIndex({ "attributes.size": 1 })
```

### Index Hashed pour Sharding

Les hashed indexes sont utilisés principalement pour les shard keys.

```javascript
// Création
db.collection.createIndex({ _id: "hashed" })

// Utilisation comme shard key
sh.shardCollection("mydb.collection", { _id: "hashed" })

// Caractéristiques :
// - Distribue uniformément les données (pas de hotspots)
// - Pas de range queries possible sur la shard key
// - Scatter-gather pour toutes les queries sans shard key
```

**Performance considerations** :

```javascript
// Point queries : Efficaces
db.collection.find({ _id: ObjectId("...") })
// → Targeted query (1 shard)

// Range queries : Inefficaces
db.collection.find({
  _id: {
    $gte: ObjectId("..."),
    $lte: ObjectId("...")
  }
})
// → Scatter-gather (tous les shards)

// Queries sans _id : Inefficaces
db.collection.find({ status: "active" })
// → Scatter-gather (tous les shards)
```

**Recommandation** :
- Utiliser hashed shard key pour write scaling uniforme
- Toujours inclure la shard key dans les queries critiques
- Considérer compound shard keys pour query targeting

### Text Index : Full-Text Search

```javascript
// Création
db.articles.createIndex({
  title: "text",
  content: "text"
})

// Avec poids
db.articles.createIndex(
  {
    title: "text",
    content: "text",
    tags: "text"
  },
  {
    weights: {
      title: 10,
      content: 5,
      tags: 1
    },
    name: "articles_text_index"
  }
)
```

**Limitations** :
- Un seul text index par collection
- Ne supporte pas les stemming avancés (langues complexes)
- Performance limitée sur grandes collections (> 10M docs)
- Pas de phrase search exacte avancée

**Alternative pour production** :
```javascript
// Pour search avancé : Utiliser Atlas Search (Lucene-based)
// Offre :
// - Fuzzy matching
// - Faceting
// - Autocomplete
// - Synonyms
// - Better relevance scoring
```

## Checklist d'Optimisation des Index

### Audit Périodique (Mensuel)

```
☐ Analyser index usage avec $indexStats
  └─> Identifier index avec 0 accesses
  └─> Drop les index inutilisés depuis > 30 jours

☐ Review top slow queries (profiler)
  └─> Identifier missing indexes
  └─> Créer index appropriés

☐ Vérifier index size vs RAM
  └─> Calculer working set
  └─> Alerter si index > 80% RAM

☐ Analyser fragmentation
  └─> storageSize vs size ratio
  └─> Planifier compact/reindex si > 1.5×

☐ Review compound index order
  └─> Vérifier ESR rule respectée
  └─> Identifier opportunités de consolidation

☐ Monitoring write performance
  └─> Vérifier impact des index sur write latency
  └─> Considérer drop si write-heavy + index peu utilisé
```

### Nouveau Feature Deployment

```
☐ Avant déploiement :
  └─> Analyser query patterns du nouveau feature
  └─> Concevoir index appropriés
  └─> Créer index en background sur production

☐ Jour du déploiement :
  └─> Vérifier index status (complete sur tous les membres)
  └─> Monitoring serré des query times
  └─> Rollback plan si performances dégradées

☐ Post-déploiement (J+7) :
  └─> Review index usage réel
  └─> Ajuster si nécessaire
  └─> Documenter décisions
```

### Scale-up Decision

```
☐ Si query latency augmente :
  └─> Vérifier index coverage
  └─> Analyser explain() des slow queries
  └─> Ajouter index manquants AVANT scale hardware

☐ Si write latency augmente :
  └─> Review nombre d'index
  └─> Considérer drop index peu utilisés
  └─> Évaluer partial/sparse pour réduire taille

☐ Si RAM saturé :
  └─> Calculer index working set
  └─> Prioriser index critiques en RAM
  └─> Scale vertical si index essentiels > RAM
```

## Conclusion

L'optimisation des index MongoDB est un processus continu nécessitant :

1. **Analyse rigoureuse** des patterns d'accès réels
2. **Application méthodique** des règles ESR et prefix utilization
3. **Équilibre** entre performance lecture et coût écriture
4. **Monitoring constant** de l'utilisation et de l'efficacité
5. **Maintenance proactive** (cleanup, rebuild, consolidation)

**Principes directeurs** :
- Créer le minimum d'index nécessaires
- Chaque index doit avoir un ROI démontrable
- Préférer compound indexes aux index simples multiples
- Utiliser partial/sparse pour optimiser la taille
- Monitorer et adapter en continu

**Métriques de succès** :
- 95% des queries utilisent un index efficacement (ratio < 10)
- Pas d'index inutilisé depuis > 30 jours
- Working set d'index < 80% RAM disponible
- Write latency impact < 15% par index

L'optimisation des index est la pierre angulaire de la performance MongoDB en production. Un jeu d'index bien conçu et maintenu est le meilleur investissement pour des performances durables et prévisibles.

---

**Points clés à retenir :**
- Ordre des champs dans compound indexes : ESR rule (Equality, Sort, Range)
- Covered queries éliminent le document fetch (10× faster)
- Partial indexes réduisent drastiquement la taille (jusqu'à 95%)
- Un seul index bien conçu vaut mieux que multiples index sous-optimaux
- Audit régulier et cleanup des index inutilisés est essentiel
- Index build en production : rolling strategy pour zero downtime

⏭️ [Optimisation des agrégations](/17-performance-tuning/05-optimisation-agregations.md)
