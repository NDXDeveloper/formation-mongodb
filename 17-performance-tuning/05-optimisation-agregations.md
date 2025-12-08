🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.5 Optimisation des Agrégations

## Introduction

Le framework d'agrégation MongoDB est un outil extrêmement puissant pour le traitement et l'analyse de données, mais sa flexibilité s'accompagne de défis de performance significatifs. Les pipelines d'agrégation peuvent varier de quelques millisecondes à plusieurs heures d'exécution selon leur conception. L'optimisation des agrégations en production nécessite une compréhension approfondie des optimisations automatiques, des patterns efficaces, et des limites du système.

Cette section explore les méthodologies expertes pour concevoir, analyser et optimiser les pipelines d'agrégation dans des environnements de production à grande échelle.

## Anatomie d'un Pipeline d'Agrégation

### Architecture d'Exécution

Un pipeline d'agrégation traverse plusieurs phases d'exécution :

```javascript
// Pipeline example
db.orders.aggregate([
  { $match: { status: "completed" } },           // Stage 1
  { $lookup: { /* join customers */ } },         // Stage 2
  { $unwind: "$items" },                         // Stage 3
  { $group: { _id: "$customerId", total: { $sum: "$items.price" } } }, // Stage 4
  { $sort: { total: -1 } },                      // Stage 5
  { $limit: 100 }                                // Stage 6
])

// Phases d'exécution :
// 1. Query Planning : Analyse et optimisation du pipeline
// 2. Index Selection : Choix des index pour stages initiaux
// 3. Pipeline Optimization : Réorganisation automatique des stages
// 4. Execution : Traitement stage par stage
// 5. Result Return : Retour des résultats
```

**Flow de données** :
```
Collection → Stage 1 → Stage 2 → ... → Stage N → Results

Chaque stage :
- Input : Documents du stage précédent
- Processing : Transformation/filtrage/agrégation
- Output : Documents pour le stage suivant
```

### Optimisations Automatiques du Query Optimizer

MongoDB applique automatiquement plusieurs optimisations au pipeline. Comprendre ces optimisations permet de concevoir des pipelines plus efficaces.

#### Optimisation 1 : $match Pushdown

**Principe** : Déplacer $match le plus tôt possible dans le pipeline.

```javascript
// Pipeline initial (sous-optimal)
db.orders.aggregate([
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
  }},
  { $unwind: "$customer" },
  { $match: {
      status: "completed",
      "customer.country": "France"
  }}
])

// Optimisé automatiquement par MongoDB
db.orders.aggregate([
  // $match sur orders déplacé avant $lookup
  { $match: { status: "completed" } },
  { $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer",
      // $match sur customers intégré dans le lookup pipeline
      pipeline: [
        { $match: { country: "France" } }
      ]
  }},
  { $unwind: "$customer" }
])

// Gain :
// - Moins de documents passent par $lookup
// - $lookup traite moins de documents customers
// - Réduction drastique de la charge
```

**Impact mesurable** :
```
Avant optimisation :
- Orders matched : 1,000,000
- Lookups performed : 1,000,000
- Filters applied after : 100,000
- Time : 45 seconds

Après optimisation :
- Orders matched : 100,000 (90% filtré avant)
- Lookups performed : 100,000
- Filters applied after : 100,000
- Time : 4.5 seconds (10× improvement)
```

#### Optimisation 2 : $sort + $limit Fusion

```javascript
// Pipeline initial
db.products.aggregate([
  { $sort: { sales: -1 } },
  { $limit: 100 }
])

// Optimisé automatiquement en "top-K sort"
// Au lieu de trier tous les documents puis prendre 100,
// maintient seulement les 100 meilleurs en mémoire pendant le tri
// = Beaucoup plus efficace

// Algorithme :
// 1. Maintenir heap de 100 documents
// 2. Pour chaque nouveau document :
//    - Si meilleur que le pire du heap : remplacer
//    - Sinon : ignorer
// 3. Retourner les 100 du heap

// Complexité :
// Sans fusion : O(N log N) + mémoire pour N documents
// Avec fusion : O(N log K) + mémoire pour K documents (K=100)
```

**Performance comparison** :
```javascript
// Collection : 10,000,000 documents
// Query : Top 100

// Sans fusion (sort all then limit) :
// - Memory : ~2GB (tout en mémoire)
// - Time : 25 seconds
// - Risk : Memory overflow si > 32MB sans allowDiskUse

// Avec fusion (top-K algorithm) :
// - Memory : ~500KB (seulement 100 docs)
// - Time : 3 seconds
// - Risk : None
```

#### Optimisation 3 : Pipeline Sequence Optimization

```javascript
// Pipeline initial
db.orders.aggregate([
  { $project: { customerId: 1, total: 1, items: 1 } },
  { $match: { total: { $gte: 100 } } }
])

// Optimisé automatiquement
db.orders.aggregate([
  { $match: { total: { $gte: 100 } } },  // Déplacé avant $project
  { $project: { customerId: 1, total: 1, items: 1 } }
])

// Gain : Filter avant projection réduit le nombre de documents à projeter
```

#### Optimisation 4 : $skip + $limit Coalescence

```javascript
// Pipeline initial
db.collection.aggregate([
  { $skip: 100 },
  { $limit: 50 }
])

// Optimisé automatiquement
// Internalement traité comme :
// - Skip les 100 premiers
// - Retourner les 50 suivants
// - Ignorer le reste (early termination)
```

### Limites des Optimisations Automatiques

**Ce que MongoDB NE peut PAS optimiser automatiquement** :

```javascript
// 1. $lookup coûteux AVANT un $match sélectif
db.orders.aggregate([
  { $lookup: { /* millions de lookups */ } },
  { $match: { /* filtre 99% */ } }
])
// Solution : Inverse l'ordre manuellement

// 2. Multiples $project consécutifs
db.collection.aggregate([
  { $project: { field1: 1, field2: 1, computed1: { $add: ["$a", "$b"] } } },
  { $project: { field1: 1, computed2: { $multiply: ["$computed1", 2] } } }
])
// Solution : Fusionner en un seul $project

// 3. $unwind suivi de $match qui pourrait être avant
db.collection.aggregate([
  { $unwind: "$items" },  // Explose les documents
  { $match: { status: "active" } }  // Filtre sur le doc parent
])
// Solution : $match avant $unwind si le filtre est sur le document parent

// 4. Répétition de calculs coûteux
db.collection.aggregate([
  { $addFields: { expensive: { /* calcul complexe */ } } },
  { $match: { expensive: { $gt: 100 } } },
  { $group: { _id: "$category", avgExpensive: { $avg: "$expensive" } } }
])
// Optimal : Le calcul est fait une fois et réutilisé
```

## Stratégies d'Optimisation des Stages Critiques

### Optimisation de $match

**Règle fondamentale** : $match doit être le plus tôt possible et le plus sélectif possible.

#### $match avec Index

```javascript
// Pipeline optimal : $match peut utiliser un index
db.orders.aggregate([
  {
    $match: {
      customerId: ObjectId("..."),  // Index utilisable
      status: "completed",
      createdAt: { $gte: ISODate("2025-01-01") }
    }
  },
  { $group: { /* ... */ } }
])

// Vérifier l'utilisation de l'index avec explain()
const explain = db.orders.explain("executionStats").aggregate([...]);

// Chercher dans explain :
// - executionStats.executionStages.stage === "IXSCAN"
// - Pas de COLLSCAN
```

**Index optimal pour $match** :
```javascript
// Créer index compound suivant la règle ESR
db.orders.createIndex({
  customerId: 1,    // Equality
  status: 1,        // Equality
  createdAt: -1     // Range
})

// Le $match utilise l'index efficacement
// - Scan direct aux documents du customerId
// - Filtre status dans ce subset
// - Range sur createdAt
```

#### $match Early Filtering

```javascript
// Calcul de la sélectivité
const totalDocs = db.orders.countDocuments();
const matchedDocs = db.orders.countDocuments({ status: "completed" });
const selectivity = matchedDocs / totalDocs;

print(`Selectivity: ${(selectivity * 100).toFixed(2)}%`);

// Si selectivity < 10% : $match est très bénéfique
// Si selectivity > 50% : Bénéfice moindre mais toujours positif
```

**Pattern anti-performance** :
```javascript
// ❌ MAUVAIS : $match tardif
db.orders.aggregate([
  { $lookup: { /* join expensive */ } },
  { $unwind: "$items" },
  { $addFields: { /* calculations */ } },
  { $match: { status: "completed" } }  // Devrait être en premier !
])

// Impact :
// - Tous les stages précédents traitent 100% des documents
// - Puis 90% sont jetés par le $match
// - Gaspillage massif de CPU et mémoire

// ✅ BON : $match en premier
db.orders.aggregate([
  { $match: { status: "completed" } },  // Filter 90% immédiatement
  { $lookup: { /* 10× moins de lookups */ } },
  { $unwind: "$items" },
  { $addFields: { /* 10× moins de calculs */ } }
])
```

### Optimisation de $lookup

Le $lookup est souvent le stage le plus coûteux d'un pipeline.

#### $lookup avec Pipeline vs Basic

**Basic $lookup** :
```javascript
// Syntaxe simple (foreign key join)
{
  $lookup: {
    from: "customers",
    localField: "customerId",
    foreignField: "_id",
    as: "customer"
  }
}

// Comportement :
// - Pour CHAQUE document d'orders
// - Exécute : db.customers.find({ _id: order.customerId })
// - Performance : O(N) où N = nombre d'orders
```

**Pipeline $lookup** (plus flexible et optimisable) :
```javascript
{
  $lookup: {
    from: "customers",
    let: { custId: "$customerId" },
    pipeline: [
      { $match: {
          $expr: { $eq: ["$_id", "$$custId"] },
          status: "active"  // Filter additionnel côté customers
      }},
      { $project: { name: 1, email: 1 } }  // Projection pour réduire data transfer
    ],
    as: "customer"
  }
}

// Avantages :
// - Filtrage côté foreign collection (moins de data transferrée)
// - Projection (documents plus petits)
// - Peut utiliser index sur customers
```

#### $lookup Performance Optimization

**1. Index sur foreignField** :
```javascript
// ESSENTIEL : Index sur le champ de jointure
db.customers.createIndex({ _id: 1 })  // _id déjà indexé par défaut

// Pour autres champs :
db.customers.createIndex({ customerId: 1 })

// Sans index : COLLSCAN pour chaque lookup = catastrophique
// Avec index : IXSCAN = rapide
```

**2. Reduce Lookup Count** :
```javascript
// Stratégie : Filter AVANT le $lookup
db.orders.aggregate([
  // ✅ Filter first
  { $match: {
      status: "completed",
      createdAt: { $gte: ISODate("2025-01-01") }
  }},  // Réduit de 1M à 100k orders

  // Puis lookup seulement sur 100k au lieu de 1M
  { $lookup: { from: "customers", ... } }
])
```

**3. Batch Lookups avec $group** :
```javascript
// Anti-pattern : Lookup individuel pour chaque document
db.orders.aggregate([
  { $lookup: { from: "products", localField: "productId", ... } }
])
// = N lookups (N = nombre d'orders)

// Pattern optimisé : Group puis lookup une fois
db.orders.aggregate([
  { $group: {
      _id: "$productId",
      orders: { $push: "$$ROOT" }
  }},
  { $lookup: {
      from: "products",
      localField: "_id",
      foreignField: "_id",
      as: "product"
  }},
  { $unwind: "$orders" },
  { $replaceRoot: {
      newRoot: {
        $mergeObjects: ["$orders", { product: { $arrayElemAt: ["$product", 0] } }]
      }
  }}
])
// = M lookups (M = nombre de produits uniques, M << N)
```

**4. Avoid Multiple $lookups** :
```javascript
// ❌ MAUVAIS : Multiple lookups séquentiels
db.orders.aggregate([
  { $lookup: { from: "customers", ... } },    // N lookups
  { $lookup: { from: "products", ... } },     // N lookups
  { $lookup: { from: "shipping", ... } }      // N lookups
])
// Total : 3N lookups

// ✅ MEILLEUR : Embedded ou denormalization
// Option A : Embed données fréquentes dans orders
{
  _id: ObjectId("orderId"),
  customerName: "John",  // Embedded
  customerEmail: "john@example.com",
  productName: "Widget",
  // Lookup seulement pour détails additionnels si nécessaire
}

// Option B : Pré-join dans une vue matérialisée
db.createView("orders_enriched", "orders", [
  { $lookup: { from: "customers", ... } },
  { $lookup: { from: "products", ... } }
])

// Queries utilisent la vue pré-joinée
db.orders_enriched.aggregate([...])
```

#### $lookup dans Sharded Clusters

**Problématique** : $lookup peut déclencher des broadcasts coûteux.

```javascript
// Sur un cluster shardé
db.orders.aggregate([
  { $match: { customerId: ObjectId("...") } },  // Targeted à 1 shard
  { $lookup: {
      from: "products",  // Collection products shardée
      localField: "productId",
      foreignField: "_id",
      as: "product"
  }}
])

// Comportement :
// 1. $match exécuté sur le shard contenant l'order
// 2. $lookup doit contacter TOUS les shards de products
//    (car productId distribution inconnue)
// 3. = Scatter-gather pour chaque lookup

// Solution : Colocation ou denormalization
// Option 1 : Shard products par même clé que orders
// Option 2 : Embed product info dans orders (denormalization)
```

### Optimisation de $group

$group est souvent le stage le plus consommateur de mémoire.

#### Memory Management

**Limite par défaut** : 100 MB par stage

```javascript
// Pipeline dépassant la limite
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      orders: { $push: "$$ROOT" }  // Accumule tous les documents
    }
  }
])

// Si trop de customers ou documents volumineux :
// Error : "Exceeded memory limit for $group"
```

**Solutions** :

**1. allowDiskUse** :
```javascript
db.orders.aggregate(
  [...],
  { allowDiskUse: true }
)

// Permet de déborder sur disque si nécessaire
// Trade-off :
// - Évite les erreurs mémoire
// - Mais significativement plus lent (10-100×)
// - Utilise le temporary directory
```

**2. Reduce Document Size** :
```javascript
// ❌ MAUVAIS : Push documents complets
{
  $group: {
    _id: "$category",
    products: { $push: "$$ROOT" }  // Document entier
  }
}

// ✅ BON : Push seulement les champs nécessaires
{
  $group: {
    _id: "$category",
    products: {
      $push: {
        productId: "$_id",
        name: "$name",
        price: "$price"
        // Seulement ce qui est nécessaire
      }
    }
  }
}

// Réduction de mémoire : 80-90% typiquement
```

**3. Pre-aggregation avec $project** :
```javascript
db.orders.aggregate([
  // Réduire taille des documents AVANT $group
  { $project: {
      customerId: 1,
      total: 1,
      date: 1
      // Exclure champs non nécessaires
  }},
  { $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$total" },
      orderCount: { $sum: 1 }
  }}
])
```

**4. Streaming Aggregation** :
```javascript
// Pour agrégations sur énormes datasets
// Stratégie : Process par chunks

async function streamingAggregation() {
  const categories = await db.products.distinct("category");
  const results = [];

  for (const category of categories) {
    const categoryResult = await db.products.aggregate([
      { $match: { category: category } },  // Process une category à la fois
      { $group: {
          _id: null,
          avgPrice: { $avg: "$price" },
          count: { $sum: 1 }
      }}
    ]).toArray();

    results.push({ category, ...categoryResult[0] });
  }

  return results;
}

// Avantages :
// - Mémoire constante (pas de pic)
// - Parallélisable
// Inconvénients :
// - Plus lent (multiple queries)
// - Code plus complexe
```

#### Accumulateurs Performants

Tous les accumulateurs n'ont pas le même coût.

**Performance par accumulateur** :

| Accumulateur | Coût | Mémoire | Note |
|--------------|------|---------|------|
| $sum | Très faible | O(1) | ✅ Optimal |
| $avg | Faible | O(1) | ✅ Optimal |
| $min / $max | Faible | O(1) | ✅ Optimal |
| $first / $last | Faible | O(1) | ✅ Optimal |
| $push | Élevé | O(N) | ⚠️ Limiter la taille |
| $addToSet | Très élevé | O(N) | ⚠️ Dédupe coûteuse |
| $stdDevPop | Moyen | O(1) | Acceptable |

**Optimisations** :

```javascript
// ❌ MAUVAIS : $addToSet sur grandes collections
{
  $group: {
    _id: "$category",
    uniqueCustomers: { $addToSet: "$customerId" }  // Peut avoir millions
  }
}
// Coût : O(N²) pour la déduplication

// ✅ MEILLEUR : Limiter ou utiliser alternative
{
  $group: {
    _id: "$category",
    customerCount: { $sum: 1 }  // Si seul le count est nécessaire
  }
}

// Ou si vraiment besoin de la liste :
db.orders.aggregate([
  { $group: {
      _id: { category: "$category", customerId: "$customerId" }
  }},
  { $group: {
      _id: "$_id.category",
      uniqueCustomers: { $push: "$_id.customerId" }
  }}
])
// Déduplication faite par le premier $group (plus efficace)
```

### Optimisation de $sort

$sort sans index supportant est très coûteux.

#### Index-Supported Sort

```javascript
// Index existant
db.orders.createIndex({ customerId: 1, orderDate: -1 })

// ✅ Sort supporté par index
db.orders.aggregate([
  { $match: { customerId: ObjectId("...") } },
  { $sort: { orderDate: -1 } }
])

// explain() montre :
// - Pas de stage SORT
// - IXSCAN avec direction: "backward"
// - executionTimeMillis très faible

// ❌ Sort NON supporté par index
db.orders.aggregate([
  { $match: { customerId: ObjectId("...") } },
  { $sort: { totalAmount: -1 } }  // Pas dans l'index
])

// explain() montre :
// - Stage SORT présent
// - memLimit: 33554432 (32 MB)
// - Risque d'échec si dataset > 32MB sans allowDiskUse
```

**Stratégie d'index pour sorts fréquents** :
```javascript
// Identifier les sorts fréquents
db.system.profile.aggregate([
  { $match: { "command.aggregate": { $exists: true } } },
  { $project: {
      collection: "$command.aggregate",
      sortFields: "$command.pipeline.$sort"
  }},
  { $unwind: "$sortFields" },
  { $group: {
      _id: "$sortFields",
      count: { $sum: 1 }
  }},
  { $sort: { count: -1 } }
])

// Créer index pour les sorts les plus fréquents
```

#### Sort + Limit Optimization

```javascript
// Pipeline optimal : Sort + Limit fusionnés
db.products.aggregate([
  { $match: { category: "Electronics" } },
  { $sort: { sales: -1 } },
  { $limit: 100 }
])

// Algorithme top-K automatique
// Mémoire : Seulement 100 documents
// Pas de risque de memory overflow

// ❌ MAUVAIS : Sort après des stages coûteux
db.products.aggregate([
  { $lookup: { /* expensive */ } },
  { $unwind: "$details" },
  { $addFields: { /* calculations */ } },
  { $sort: { calculatedField: -1 } },  // Sort sur tous les documents expansés
  { $limit: 100 }
])

// ✅ MEILLEUR : Reduce dataset avant sort si possible
db.products.aggregate([
  { $match: { /* filter */ } },
  { $sort: { baseField: -1 } },
  { $limit: 100 },  // Réduire à 100 tôt
  { $lookup: { /* lookup sur 100 au lieu de millions */ } },
  { $unwind: "$details" },
  { $addFields: { /* calculations sur 100 */ } }
])
```

### Optimisation de $unwind

$unwind peut exploser le nombre de documents à traiter.

```javascript
// Document avec array
{
  _id: 1,
  customer: "John",
  items: [
    { product: "A", qty: 2 },
    { product: "B", qty: 1 },
    { product: "C", qty: 5 }
  ]
}

// Après $unwind
db.orders.aggregate([
  { $unwind: "$items" }
])

// Résultat : 3 documents
{ _id: 1, customer: "John", items: { product: "A", qty: 2 } }
{ _id: 1, customer: "John", items: { product: "B", qty: 1 } }
{ _id: 1, customer: "John", items: { product: "C", qty: 5 } }

// Si array moyen de 10 items :
// - Input : 1,000,000 orders
// - After $unwind : 10,000,000 documents
// = 10× augmentation
```

**Optimisations** :

**1. Filter BEFORE $unwind** :
```javascript
// ❌ MAUVAIS : Unwind puis filter
db.orders.aggregate([
  { $unwind: "$items" },
  { $match: { "items.product": "A" } }
])
// Unwind tous les items, puis garde seulement "A"

// ✅ BON : Filter l'array avant unwind
db.orders.aggregate([
  { $addFields: {
      items: {
        $filter: {
          input: "$items",
          as: "item",
          cond: { $eq: ["$$item.product", "A"] }
        }
      }
  }},
  { $match: { items: { $ne: [] } } },  // Garde seulement orders avec items filtrés
  { $unwind: "$items" }
])
// Unwind seulement les items pertinents
```

**2. preserveNullAndEmptyArrays : false** :
```javascript
// Par défaut : false (bon)
{ $unwind: "$items" }

// Si array vide ou absent : document supprimé
// = Moins de documents à traiter dans stages suivants

// preserveNullAndEmptyArrays: true
{ $unwind: { path: "$items", preserveNullAndEmptyArrays: true } }

// Garde les documents même si array vide
// = Plus de documents (performance impact)
// Utiliser seulement si nécessaire pour la logique métier
```

**3. Avoid Multiple $unwinds** :
```javascript
// ❌ TRÈS MAUVAIS : Double $unwind
db.orders.aggregate([
  { $unwind: "$items" },        // Array de 10 items
  { $unwind: "$items.options" } // Chaque item a 5 options
])

// Explosion : 1 order → 50 documents (10 × 5)
// 1,000,000 orders → 50,000,000 documents

// ✅ Alternative : Aggregation différente
// Utiliser $reduce, $map, ou $filter pour traiter les arrays sans unwind
```

## Stratégies Avancées d'Optimisation

### Pipeline Splitting

Pour les pipelines longs, considérer le splitting en plusieurs étapes.

```javascript
// Pipeline monolithique (dur à optimiser et debug)
db.orders.aggregate([
  { $match: { /* ... */ } },
  { $lookup: { /* ... */ } },
  { $unwind: "$items" },
  { $group: { /* complex aggregation */ } },
  { $lookup: { /* another join */ } },
  { $project: { /* complex transformations */ } },
  { $sort: { /* ... */ } },
  { $limit: 100 }
])

// Stratégie : Split en étapes logiques avec materialization
// Étape 1 : Pré-agrégation et storage
db.orders.aggregate([
  { $match: { /* ... */ } },
  { $lookup: { /* ... */ } },
  { $unwind: "$items" },
  { $group: { /* complex aggregation */ } },
  { $out: "orders_preaggregated" }  // Materialise dans collection temp
])

// Étape 2 : Processing final sur dataset réduit
db.orders_preaggregated.aggregate([
  { $lookup: { /* another join - sur dataset plus petit */ } },
  { $project: { /* transformations */ } },
  { $sort: { /* ... */ } },
  { $limit: 100 }
])

// Avantages :
// - Chaque étape optimisable indépendamment
// - Debugging plus facile
// - Réutilisation de l'étape 1 pour différentes queries
// - Résultats intermédiaires indexables

// Inconvénients :
// - Overhead du write intermédiaire
// - Pas suitable pour real-time (latency accrue)
```

### Vues Matérialisées avec $merge et $out

Pour agrégations coûteuses exécutées fréquemment, matérialiser les résultats.

#### $out : Remplacement Complet

```javascript
// Recalcul complet et remplacement
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$total" },
      orderCount: { $sum: 1 },
      lastOrderDate: { $max: "$orderDate" }
    }
  },
  { $out: "customer_stats" }  // Remplace la collection entièrement
])

// Utilisation :
// Au lieu de ré-agréger à chaque fois :
db.customer_stats.find({ totalSpent: { $gte: 10000 } })

// Stratégie de refresh :
// - Cron job nocturne pour recalcul complet
// - Ou trigger sur changements significatifs
```

#### $merge : Mise à Jour Incrémentale

```javascript
// Mise à jour incrémentale (MongoDB 4.2+)
db.orders.aggregate([
  {
    $match: {
      orderDate: { $gte: lastProcessedDate }  // Seulement nouveaux orders
    }
  },
  {
    $group: {
      _id: "$customerId",
      newSpent: { $sum: "$total" },
      newOrders: { $sum: 1 }
    }
  },
  {
    $merge: {
      into: "customer_stats",
      on: "_id",  // Match sur customerId
      whenMatched: [  // Update des stats existantes
        {
          $set: {
            totalSpent: { $add: ["$totalSpent", "$newSpent"] },
            orderCount: { $add: ["$orderCount", "$newOrders"] },
            lastUpdated: "$$NOW"
          }
        }
      ],
      whenNotMatched: "insert"  // Insert si nouveau customer
    }
  }
])

// Avantages :
// - Incrémental = beaucoup plus rapide
// - Pas de recalcul complet
// - Stats toujours relativement à jour

// Usage :
// Exécution fréquente (ex: toutes les 5 minutes)
// Charge distribuée vs batch nocturne
```

**Exemple complet : Dashboard stats** :

```javascript
// Collection stats matérialisée
db.createCollection("daily_stats", {
  validator: {
    $jsonSchema: {
      required: ["date", "metrics"],
      properties: {
        date: { bsonType: "date" },
        metrics: {
          bsonType: "object",
          properties: {
            totalRevenue: { bsonType: "double" },
            orderCount: { bsonType: "int" },
            avgOrderValue: { bsonType: "double" }
          }
        }
      }
    }
  }
})

// Index pour queries rapides
db.daily_stats.createIndex({ date: -1 })

// Pipeline de calcul (exécuté quotidiennement)
db.orders.aggregate([
  {
    $match: {
      orderDate: {
        $gte: ISODate("2025-01-15T00:00:00Z"),
        $lt: ISODate("2025-01-16T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: null,
      totalRevenue: { $sum: "$total" },
      orderCount: { $sum: 1 }
    }
  },
  {
    $addFields: {
      avgOrderValue: { $divide: ["$totalRevenue", "$orderCount"] }
    }
  },
  {
    $project: {
      _id: 0,
      date: ISODate("2025-01-15"),
      metrics: {
        totalRevenue: "$totalRevenue",
        orderCount: "$orderCount",
        avgOrderValue: "$avgOrderValue"
      }
    }
  },
  {
    $merge: {
      into: "daily_stats",
      on: "date",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])

// Dashboard query : Instantané !
db.daily_stats.find().sort({ date: -1 }).limit(30)
// vs agrégation en temps réel : 10,000× plus rapide
```

### Optimisation dans Sharded Clusters

Les pipelines sur clusters shardés ont des considérations spéciales.

#### Pipeline Targeting

```javascript
// ✅ TARGETED : Query contient la shard key
db.orders.aggregate([
  { $match: { customerId: ObjectId("...") } }  // Shard key
  // ...
])

// Exécution :
// - mongos route vers 1 seul shard
// - Pipeline exécuté entièrement sur ce shard
// - Résultat retourné directement
// Performance : Optimale

// ❌ SCATTER-GATHER : Query sans shard key
db.orders.aggregate([
  { $match: { status: "completed" } }  // Pas la shard key
  // ...
])

// Exécution :
// - mongos envoie le pipeline à TOUS les shards
// - Chaque shard exécute et retourne résultats partiels
// - mongos merge les résultats
// Performance : N× plus lent (N = nombre de shards)
```

#### Split Pipeline

MongoDB divise automatiquement certains pipelines entre shards et mongos.

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$category", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
])

// Exécution splitée :

// Sur chaque shard :
[
  { $match: { status: "completed" } },
  { $group: { _id: "$category", total: { $sum: "$amount" } } }
]
// Retour au mongos : Résultats partiels par category

// Sur mongos :
[
  { $group: { _id: "$_id", total: { $sum: "$total" } } },  // Merge des groupes
  { $sort: { total: -1 } },
  { $limit: 10 }
]

// Optimisations :
// - Agrégation parallèle sur les shards
// - Mongos ne merge que les résultats agrégés (petit dataset)
```

**Stages non-distribuables** (toujours sur mongos) :

- $out / $merge (avant MongoDB 4.4)
- $lookup (si foreign collection non sharded)
- $graphLookup
- $facet
- $collStats

```javascript
// Pipeline avec $lookup sur collection non-shardée
db.sharded_orders.aggregate([
  { $match: { /* ... */ } },  // Exécuté sur shards
  { $lookup: {                 // FORCE tout à remonter à mongos
      from: "unsharded_customers",
      // ...
  }},
  { $group: { /* ... */ } }   // Exécuté sur mongos
])

// Impact :
// - Tous les documents matchés remontent à mongos
// - Network overhead élevé
// - Mongos devient bottleneck

// Solution : Shard aussi customers, ou embed customer data
```

## Analyse et Debugging des Pipelines

### explain() pour Aggregation

```javascript
const explain = db.collection.explain("executionStats").aggregate([...])

// Structure spécifique aux aggregations
{
  "stages": [
    {
      "$cursor": {
        "queryPlanner": { /* plan pour le $cursor stage */ },
        "executionStats": { /* stats pour requête initiale */ }
      }
    },
    { "$stage2": { /* info sur stage 2 */ } },
    // ...
  ],
  "serverInfo": { /* ... */ },
  "ok": 1
}
```

**Analyse du $cursor stage** :

```javascript
// Le $cursor stage représente la partie du pipeline
// qui peut utiliser les index (généralement $match initial)

const cursorStats = explain.stages[0].$cursor.executionStats;

// Métriques critiques :
{
  totalDocsExamined: 150000,    // Documents examinés
  totalKeysExamined: 150000,    // Clés d'index examinées
  nReturned: 10000,             // Documents retournés au pipeline
  executionTimeMillis: 234,     // Temps du cursor

  executionStages: {
    stage: "IXSCAN",            // ✅ Utilise un index
    indexName: "status_1_date_-1",
    // ...
  }
}

// Ratio d'efficacité
const efficiency = cursorStats.nReturned / cursorStats.totalDocsExamined;
// > 0.1 : Bon
// < 0.01 : Problématique
```

**Analyse des stages suivants** :

```javascript
// Stages après $cursor n'ont généralement pas de stats détaillées
// mais on peut inférer :

explain.stages.forEach((stage, idx) => {
  const stageName = Object.keys(stage)[0];

  if (stageName === "$group") {
    // Vérifier si allowDiskUse a été nécessaire
    // Vérifier la mémoire utilisée (non directement disponible)
    print(`Stage ${idx}: $group - Monitor memory usage`);
  }

  if (stageName === "$lookup") {
    // Lookup est coûteux
    // Vérifier si foreign collection est indexée
    print(`Stage ${idx}: $lookup - Verify foreign index`);
  }

  if (stageName === "$sort") {
    // Sort sans index est coûteux
    print(`Stage ${idx}: $sort - Check if index-supported`);
  }
});
```

### Profiling des Agrégations

```javascript
// Activer profiler
db.setProfilingLevel(2)

// Exécuter aggregation
db.orders.aggregate([...])

// Analyser dans system.profile
db.system.profile.find({
  "command.aggregate": { $exists: true },
  ns: "mydb.orders"
}).sort({ ts: -1 }).limit(10).pretty()

// Métriques importantes :
{
  op: "command",
  ns: "mydb.orders",
  command: {
    aggregate: "orders",
    pipeline: [ /* ... */ ],
    cursor: {},
    allowDiskUse: false
  },

  // Métriques critiques
  millis: 5234,              // Temps total
  planSummary: "IXSCAN ...", // Plan utilisé
  docsExamined: 1500000,     // Documents examinés
  nreturned: 100,            // Documents retournés

  // Ressources
  cursorExhausted: true,
  numYield: 234,             // Nombre de yields (contention)
  locks: { /* ... */ },      // Lock stats

  // Flow control (si cluster)
  flowControl: { /* ... */ }
}
```

**Identifier les agrégations lentes** :

```javascript
// Top 10 agrégations les plus lentes
db.system.profile.aggregate([
  {
    $match: {
      "command.aggregate": { $exists: true },
      millis: { $gte: 1000 }  // > 1 seconde
    }
  },
  {
    $project: {
      collection: "$command.aggregate",
      pipeline: "$command.pipeline",
      millis: 1,
      docsExamined: 1,
      nreturned: 1,
      planSummary: 1
    }
  },
  { $sort: { millis: -1 } },
  { $limit: 10 }
])
```

### Performance Testing Méthodique

```javascript
// Framework de test de performance
class AggregationPerformanceTest {
  constructor(collection, pipeline, name) {
    this.collection = collection;
    this.pipeline = pipeline;
    this.name = name;
  }

  async run(iterations = 5) {
    const results = [];

    // Warm-up run
    await db[this.collection].aggregate(this.pipeline).toArray();

    // Timed runs
    for (let i = 0; i < iterations; i++) {
      const start = Date.now();
      const result = await db[this.collection].aggregate(this.pipeline).toArray();
      const duration = Date.now() - start;

      results.push({
        iteration: i + 1,
        duration: duration,
        resultCount: result.length
      });
    }

    // Statistics
    const durations = results.map(r => r.duration);
    const avg = durations.reduce((a, b) => a + b) / durations.length;
    const min = Math.min(...durations);
    const max = Math.max(...durations);

    return {
      name: this.name,
      iterations: iterations,
      avgMs: avg.toFixed(2),
      minMs: min,
      maxMs: max,
      resultCount: results[0].resultCount,
      results: results
    };
  }
}

// Usage
const test = new AggregationPerformanceTest(
  "orders",
  [
    { $match: { status: "completed" } },
    { $group: { _id: "$customerId", total: { $sum: "$amount" } } }
  ],
  "Customer Totals Aggregation"
);

const perfResults = await test.run(10);
printjson(perfResults);
```

## Patterns d'Optimisation par Cas d'Usage

### Real-Time Analytics

**Contrainte** : Latence < 100ms

```javascript
// ❌ MAUVAIS : Agrégation complexe en temps réel
db.events.aggregate([
  { $match: { timestamp: { $gte: last24Hours } } },
  { $group: { _id: "$eventType", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])
// Latence : 2-5 secondes sur dataset large

// ✅ BON : Pré-agrégation avec incremental update
// Collection pré-agrégée mise à jour en continu
db.event_stats_realtime.find().sort({ count: -1 })
// Latence : < 10ms

// Mise à jour via change stream
const changeStream = db.events.watch();
changeStream.on('change', (change) => {
  if (change.operationType === 'insert') {
    db.event_stats_realtime.updateOne(
      { _id: change.fullDocument.eventType },
      { $inc: { count: 1 } },
      { upsert: true }
    );
  }
});
```

### Reporting/BI Queries

**Contrainte** : Throughput élevé, latence acceptable (secondes)

```javascript
// Stratégie : Vues matérialisées avec refresh quotidien
// Vue : Sales by region, product category
db.orders.aggregate([
  {
    $match: {
      orderDate: { $gte: startOfMonth, $lt: endOfMonth }
    }
  },
  {
    $group: {
      _id: {
        region: "$shippingAddress.region",
        category: "$productCategory"
      },
      revenue: { $sum: "$total" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$total" }
    }
  },
  {
    $out: "sales_by_region_category_monthly"
  }
])

// Index pour queries BI rapides
db.sales_by_region_category_monthly.createIndex({ "_id.region": 1 })
db.sales_by_region_category_monthly.createIndex({ "_id.category": 1 })
db.sales_by_region_category_monthly.createIndex({ revenue: -1 })

// BI tools query la vue matérialisée : Instant !
```

### ETL / Batch Processing

**Contrainte** : Volume énorme, pas de contrainte temps réel

```javascript
// Stratégie : Process par chunks avec allowDiskUse
const batchSize = 10000;
let processed = 0;

while (true) {
  const batch = await db.raw_data.aggregate([
    { $skip: processed },
    { $limit: batchSize },
    {
      $lookup: { /* enrichment */ }
    },
    {
      $project: { /* transformation */ }
    }
  ], {
    allowDiskUse: true,
    maxTimeMS: 300000  // 5 minutes timeout par batch
  }).toArray();

  if (batch.length === 0) break;

  // Insert dans collection de destination
  await db.processed_data.insertMany(batch);

  processed += batch.length;
  print(`Processed: ${processed} documents`);
}
```

## Checklist d'Optimisation

### Avant Déploiement

```
☐ $match en premier stage (avec index supportant)
☐ Projections précoces pour réduire taille documents
☐ $lookup minimisé et avec index sur foreignField
☐ $unwind uniquement si nécessaire, après filtrage
☐ $group avec accumulateurs légers ($sum, $avg vs $push)
☐ $sort supporté par index ou fusionné avec $limit
☐ explain() analysé avec ratios acceptables
☐ Test avec dataset production-like (taille et distribution)
☐ allowDiskUse configuré si nécessaire
☐ Timeout approprié (maxTimeMS)
```

### Monitoring Production

```
☐ Latence P95 des agrégations < seuil défini
☐ Aucune agrégation dépassant memory limit
☐ Pas d'utilisation excessive de allowDiskUse (disque lent)
☐ Index usage validé via $indexStats
☐ Profiler pour identifier regressions
☐ Resource utilization (CPU, RAM, I/O) sous contrôle
```

### Maintenance Régulière

```
☐ Review pipelines lents (profiler monthly)
☐ Update vues matérialisées (si utilisées)
☐ Index optimization pour nouveaux patterns
☐ Cleanup de collections temporaires ($out/$merge)
☐ Vérifier evolution des query patterns
```

## Conclusion

L'optimisation des agrégations MongoDB nécessite une approche systématique :

1. **Conception** : Ordre optimal des stages (filter early, project early, reduce document count)
2. **Index** : Support pour $match et $sort stages
3. **Memory Management** : Accumulateurs légers, allowDiskUse quand nécessaire
4. **Materialization** : Vues pour queries fréquentes et coûteuses
5. **Monitoring** : explain(), profiler, métriques de production

**Principes directeurs** :
- Réduire le dataset le plus tôt possible
- Minimiser les stages coûteux ($lookup, $unwind)
- Exploiter les index au maximum
- Matérialiser pour queries répétitives
- Mesurer et itérer

**ROI typiques** :
- $match optimization : 10-100× improvement
- Index-supported $sort : 10-50× improvement
- Proper $lookup strategy : 5-20× improvement
- Materialized views : 100-1000× improvement

L'excellence en optimisation d'agrégations vient de la compréhension profonde du coût de chaque stage, de l'utilisation judicieuse des index, et de l'adaptation de la stratégie au cas d'usage spécifique (real-time vs batch, read-heavy vs write-heavy).

---

**Points clés à retenir :**
- $match et $project en premier : Réduire le dataset immédiatement
- Index pour $match et $sort : Éviter collection scans et in-memory sorts
- $lookup est coûteux : Minimiser, indexer foreignField, considérer denormalization
- $group memory limit : Utiliser accumulateurs légers, allowDiskUse si nécessaire
- Vues matérialisées : Pour agrégations coûteuses et fréquentes
- explain() et profiler : Essentiels pour l'analyse et l'optimisation
- Pipeline splitting : Pour pipelines complexes, décomposer et matérialiser

⏭️ [Configuration du moteur WiredTiger](/17-performance-tuning/06-configuration-wiredtiger.md)
