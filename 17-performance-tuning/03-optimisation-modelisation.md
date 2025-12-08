🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.3 Optimisation de la Modélisation

## Introduction

La modélisation des données est la décision architecturale la plus critique pour les performances MongoDB. Contrairement aux bases relationnelles où la normalisation est la norme, MongoDB exige une approche radicalement différente, optimisée pour les patterns d'accès spécifiques de l'application. Une mauvaise modélisation ne peut être compensée par l'indexation ou le hardware : elle est la fondation sur laquelle repose toute la performance du système.

Cette section présente les méthodologies expertes pour concevoir, analyser et optimiser les modèles de données MongoDB en production, en équilibrant performance, scalabilité et maintenabilité.

## Principes Fondamentaux de Modélisation Orientée Performance

### Paradigme Document-First

Le paradigme document-first inverse la logique relationnelle :

**Approche relationnelle** :
```
1. Identifier les entités
2. Normaliser pour éliminer la redondance
3. Joindre à la lecture
```

**Approche MongoDB optimale** :
```
1. Identifier les patterns d'accès (queries patterns)
2. Modéliser pour optimiser les lectures fréquentes
3. Accepter la redondance contrôlée
4. Gérer la cohérence en écriture
```

### La Règle du 80/20 Appliquée

**Principe** : 80% des requêtes accèdent à 20% des données de manière prédictible.

**Implication** :
Optimiser le modèle pour ces 80% de requêtes, même si cela pénalise les 20% restants.

**Exemple concret** :
```javascript
// Cas : Application e-commerce
// 80% : Affichage produit avec stock, prix, images
// 20% : Mise à jour stock, modification prix, analytics

// Modèle optimisé pour les 80%
{
  _id: ObjectId("..."),
  name: "Laptop Pro 2025",
  price: 1299.99,
  stock: 45,
  category: "Electronics",
  images: ["url1.jpg", "url2.jpg"],  // Embedded pour lecture rapide
  specifications: {                   // Embedded, accédé fréquemment
    cpu: "Intel i9",
    ram: "32GB",
    storage: "1TB SSD"
  },
  // Référence pour données rarement accédées
  reviewsId: ObjectId("..."),         // Lookup seulement si demandé
  supplierDetailsId: ObjectId("...")  // Admin uniquement
}
```

### Data Locality : Le Principe Cardinal

**Définition** :
Co-localiser les données fréquemment accédées ensemble minimise les I/O et maximise l'efficacité du cache.

**Impact sur les performances** :
```
Lecture document unique : 1 I/O
Lecture + 5 références : 6 I/O
Latence multiplicative : 6×
Cache miss probability : 6×
```

**Métrique de localité** :
```javascript
// Calcul du score de localité
dataLocality = (fieldsAccessedInSingleDoc / totalFieldsNeeded) × 100

// Excellent : > 90%
// Bon : 70-90%
// Médiocre : 50-70%
// Mauvais : < 50%
```

## Analyse des Patterns d'Accès

### Méthodologie d'Analyse des Requêtes

#### Phase 1 : Capture des Patterns

**Profiling étendu** :
```javascript
// Activer profiler niveau 2 temporairement (1-2 heures aux heures de pointe)
db.setProfilingLevel(2, { slowms: 0 });

// Analyser après collecte
db.system.profile.aggregate([
  {
    $match: {
      ts: { $gte: ISODate("2025-01-15T08:00:00Z") },
      op: { $in: ["query", "update", "remove"] }
    }
  },
  {
    $group: {
      _id: {
        ns: "$ns",
        operation: "$op",
        // Normaliser les query patterns
        pattern: {
          $reduce: {
            input: { $objectToArray: "$command.filter" },
            initialValue: {},
            in: {
              $mergeObjects: [
                "$$value",
                { $arrayToObject: [[{ k: "$$this.k", v: "VALUE" }]] }
              ]
            }
          }
        }
      },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" },
      totalMs: { $sum: "$millis" }
    }
  },
  {
    $addFields: {
      impact: { $multiply: ["$count", "$avgMs"] }
    }
  },
  { $sort: { impact: -1 } },
  { $limit: 50 }
])
```

#### Phase 2 : Classification des Patterns

**Matrice de classification** :

| Pattern | Fréquence | Latence Critique | Optimisation Prioritaire |
|---------|-----------|------------------|--------------------------|
| **Type A** | Très élevée (>1000/s) | Oui (<50ms) | **CRITICAL** - Embedded mandatory |
| **Type B** | Élevée (100-1000/s) | Oui (<100ms) | **HIGH** - Embedded recommended |
| **Type C** | Moyenne (10-100/s) | Non (<500ms) | **MEDIUM** - Balance |
| **Type D** | Faible (<10/s) | Non | **LOW** - Reference acceptable |

**Exemple d'analyse** :
```javascript
// Pattern identifié : Affichage page utilisateur
// Fréquence : 2500 req/s
// Latence cible : < 30ms
// Données accédées : user profile + recent orders (5 dernières)

// Classification : Type A
// Décision : Embedded les 5 dernières commandes dans le document user
```

### Analyse de la Cardinalité des Relations

La cardinalité influence fondamentalement la stratégie de modélisation.

#### One-to-Few (1:N où N < 100)

**Caractéristiques** :
- N typiquement < 100 éléments
- Croissance bornée et prévisible
- Accès fréquent simultané

**Stratégie optimale** : **Embedding**

**Exemple** :
```javascript
// User avec adresses de livraison (max ~10)
{
  _id: ObjectId("..."),
  name: "John Doe",
  email: "john@example.com",
  addresses: [  // Embedded array
    {
      type: "home",
      street: "123 Main St",
      city: "Paris",
      zipCode: "75001",
      isDefault: true
    },
    {
      type: "work",
      street: "456 Office Blvd",
      city: "Paris",
      zipCode: "75008",
      isDefault: false
    }
  ]
}

// Performance :
// - 1 seule lecture pour user + toutes ses adresses
// - Pas de $lookup nécessaire
// - Atomicité des updates garantie
```

#### One-to-Many (1:N où N = 100-10,000)

**Caractéristiques** :
- N variable, peut croître significativement
- Accès partiel (pagination, filtrage)
- Relation forte mais volumineuse

**Stratégie optimale** : **Hybrid (Subset Pattern)**

**Pattern Subset** :
```javascript
// Collection users
{
  _id: ObjectId("userId"),
  name: "Jane Smith",
  email: "jane@example.com",
  // Subset : Les N plus récents/importants
  recentOrders: [
    {
      orderId: ObjectId("..."),
      date: ISODate("2025-01-15"),
      total: 299.99,
      status: "shipped"
    }
    // ... 10-20 dernières commandes
  ],
  ordersSummary: {
    total: 342,
    totalSpent: 45234.67,
    lastOrderDate: ISODate("2025-01-15")
  }
}

// Collection orders (complète)
{
  _id: ObjectId("orderId"),
  userId: ObjectId("userId"),
  date: ISODate("2025-01-15"),
  items: [ /* détails complets */ ],
  total: 299.99,
  status: "shipped"
  // ... toutes les données
}

// Stratégie d'accès :
// - Page principale : recentOrders (embedded, 1 query)
// - Historique complet : orders collection ($match userId, paginated)
// - Statistiques : ordersSummary (embedded, pré-calculé)
```

**Avantages** :
- Performance optimale pour 90% des accès (embedded subset)
- Scalabilité pour données complètes (separate collection)
- Coût d'écriture modéré (update subset + insert orders)

#### One-to-Millions (1:N où N > 10,000)

**Caractéristiques** :
- N très grand, potentiellement millions
- Croissance non bornée
- Accès toujours partiel

**Stratégie optimale** : **Reference avec Extended Reference Pattern**

**Extended Reference Pattern** :
```javascript
// Collection products
{
  _id: ObjectId("productId"),
  name: "Popular Gadget",
  price: 99.99,
  // Extended reference : Données agrégées des reviews
  reviewsSummary: {
    count: 12847,
    averageRating: 4.6,
    distribution: {
      5: 8234,
      4: 3012,
      3: 1204,
      2: 287,
      1: 110
    },
    // Top reviews pour affichage rapide
    featured: [
      {
        reviewId: ObjectId("..."),
        author: "John D.",
        rating: 5,
        snippet: "Excellent product, highly recommend...",
        helpful: 234
      }
      // ... 3-5 top reviews
    ]
  }
}

// Collection reviews (séparée)
{
  _id: ObjectId("reviewId"),
  productId: ObjectId("productId"),
  userId: ObjectId("..."),
  rating: 5,
  title: "Excellent product",
  content: "Full review text here...",
  date: ISODate("2025-01-10"),
  helpful: 234
}

// Accès :
// - Affichage produit : reviewsSummary embedded (1 query)
// - Toutes les reviews : query reviews collection (paginated)
// - Maintien cohérence : Update summary via change streams ou batch job
```

### Analyse de la Fréquence Lecture/Écriture

**Ratio Read/Write** est déterminant pour la stratégie de denormalisation.

```javascript
// Calcul du ratio
readWriteRatio = readOpsPerSecond / writeOpsPerSecond

// Interprétation :
// > 100:1  → Denormalisation agressive acceptable
// 10-100:1 → Denormalisation sélective
// 1-10:1   → Normalisation privilégiée
// < 1:1    → Write-optimized (éviter denormalisation)
```

**Exemple : Système de blog**

```javascript
// Articles : Read-heavy (ratio ~1000:1)
// Écriture : Rare (publication/modification)
// Lecture : Très fréquente (vues, listing)

// Modèle optimisé :
{
  _id: ObjectId("articleId"),
  title: "MongoDB Performance Tuning",
  content: "Full article text...",

  // Denormalisé : author info (read-heavy)
  author: {
    id: ObjectId("authorId"),
    name: "John Expert",
    avatar: "url",
    bio: "Expert MongoDB developer"
  },

  // Pré-calculé : stats (read-heavy)
  stats: {
    views: 45234,
    likes: 1234,
    commentsCount: 89
  },

  // Référence : comments (1:many, accès pagination)
  // Stockés dans collection séparée
}

// Maintenance :
// - Author change : Batch update nécessaire mais rare
// - Stats update : Increment operators, très efficaces
// - Comments : Séparés, pas d'impact sur article reads
```

## Stratégies d'Optimisation Avancées

### Pattern 1 : Attribute Pattern

**Problématique** :
Documents avec un grand nombre de champs similaires mais peu utilisés ensemble, créant des index inefficaces.

**Cas d'usage** :
Caractéristiques produits variables, métadonnées, configurations.

**Avant (sous-optimal)** :
```javascript
{
  _id: ObjectId("productId"),
  name: "Laptop",
  // Centaines de specs possibles
  spec_cpu: "Intel i9",
  spec_ram: "32GB",
  spec_storage: "1TB SSD",
  spec_screen: "15.6 inch",
  spec_weight: "1.8kg",
  // ... 100+ spec_ fields
}

// Problème :
// - Index sur chaque spec_* impraticable
// - Requête type "find products with spec_X = value" inefficace
// - Sparse documents (la plupart des specs null pour un produit)
```

**Après (optimisé avec Attribute Pattern)** :
```javascript
{
  _id: ObjectId("productId"),
  name: "Laptop",
  category: "Electronics",

  // Transformation en array de key-value
  specs: [
    { k: "cpu", v: "Intel i9" },
    { k: "ram", v: "32GB" },
    { k: "storage", v: "1TB SSD" },
    { k: "screen", v: "15.6 inch" },
    { k: "weight", v: "1.8kg" }
  ]
}

// Index efficace :
db.products.createIndex({ "specs.k": 1, "specs.v": 1 })

// Requête optimisée :
db.products.find({
  "specs": {
    $elemMatch: {
      k: "cpu",
      v: /Intel i9/
    }
  }
})
```

**Gains de performance** :
- 1 index pour tous les attributs vs N index
- Documents plus denses (moins de null fields)
- Requêtes plus rapides sur attributs dynamiques

**Trade-off** :
- Requêtes légèrement plus complexes
- Perte du type checking strict par field

### Pattern 2 : Bucket Pattern

**Problématique** :
Très grand nombre de documents de petite taille (IoT, time-series, logs).

**Problème de performance** :
- Overhead de stockage (chaque document a un _id, index entries)
- Fragmentation
- Index gigantesques

**Cas d'usage** :
Données IoT, metrics, logs, événements temporels.

**Avant (sous-optimal)** :
```javascript
// 1 document par mesure
{
  _id: ObjectId("..."),
  sensorId: "sensor_123",
  timestamp: ISODate("2025-01-15T10:00:00Z"),
  temperature: 22.5
}

// Problème :
// - 86,400 documents/jour/sensor (1 par seconde)
// - 31.5M documents/an/sensor
// - Index énorme sur sensorId + timestamp
```

**Après (optimisé avec Bucket Pattern)** :
```javascript
// Grouper par buckets (ex: 1 heure)
{
  _id: ObjectId("..."),
  sensorId: "sensor_123",
  bucketStart: ISODate("2025-01-15T10:00:00Z"),
  bucketEnd: ISODate("2025-01-15T11:00:00Z"),

  // Mesures groupées
  measurements: [
    { ts: ISODate("2025-01-15T10:00:00Z"), temp: 22.5 },
    { ts: ISODate("2025-01-15T10:00:01Z"), temp: 22.6 },
    // ... 3600 mesures pour l'heure
  ],

  // Statistiques pré-calculées
  stats: {
    count: 3600,
    avgTemp: 22.7,
    minTemp: 21.8,
    maxTemp: 23.9
  }
}

// Réduction :
// - 86,400 → 24 documents/jour/sensor
// - 31.5M → 8,760 documents/an/sensor
// - Index size réduit de ~3600×
```

**Index optimal** :
```javascript
db.sensor_data.createIndex({
  sensorId: 1,
  bucketStart: 1
})
```

**Variations** :
- **Fixed bucket size** : 1 hour, 1 day (prévisible)
- **Dynamic bucket size** : Remplir jusqu'à limite (16MB), puis nouveau bucket
- **Hybrid** : Bucket fixe avec rollover si limite atteinte

**Requête sur bucket** :
```javascript
// Requête : Données sensor entre 10h et 11h
db.sensor_data.find({
  sensorId: "sensor_123",
  bucketStart: { $gte: ISODate("2025-01-15T10:00:00Z") },
  bucketEnd: { $lte: ISODate("2025-01-15T11:00:00Z") }
})

// Si besoin de filtrer dans le bucket :
db.sensor_data.aggregate([
  {
    $match: {
      sensorId: "sensor_123",
      bucketStart: { $gte: ISODate("2025-01-15T10:00:00Z") }
    }
  },
  { $unwind: "$measurements" },
  {
    $match: {
      "measurements.ts": {
        $gte: ISODate("2025-01-15T10:15:00Z"),
        $lte: ISODate("2025-01-15T10:30:00Z")
      }
    }
  }
])
```

### Pattern 3 : Computed Pattern

**Problématique** :
Calculs coûteux répétés à chaque lecture (agrégations, statistics).

**Stratégie** :
Pré-calculer et stocker les résultats, mettre à jour de manière asynchrone ou incrémentale.

**Exemple : Dashboard e-commerce**

```javascript
// Sans computed pattern (requête coûteuse à chaque affichage)
db.orders.aggregate([
  {
    $match: {
      userId: ObjectId("userId"),
      status: "completed"
    }
  },
  {
    $group: {
      _id: null,
      totalOrders: { $sum: 1 },
      totalSpent: { $sum: "$total" },
      avgOrderValue: { $avg: "$total" },
      // ... plus de calculs
    }
  }
])

// Problème :
// - Calcul à chaque chargement dashboard
// - Agrégation sur potentiellement millions de commandes
// - Latence > 1000ms inacceptable
```

**Avec Computed Pattern** :
```javascript
// Collection users avec computed fields
{
  _id: ObjectId("userId"),
  name: "John Doe",
  email: "john@example.com",

  // Statistiques pré-calculées
  orderStats: {
    totalOrders: 47,
    totalSpent: 12456.78,
    avgOrderValue: 264.82,
    lastOrderDate: ISODate("2025-01-14"),
    lastOrderValue: 299.99,

    // Breakdown par période
    currentYear: {
      orders: 12,
      spent: 3456.78
    },
    lastMonth: {
      orders: 3,
      spent: 789.50
    },

    // Calculé : timestamp de dernière mise à jour
    lastUpdated: ISODate("2025-01-15T10:30:00Z")
  }
}

// Dashboard : 1 seule requête instantanée
db.users.findOne(
  { _id: ObjectId("userId") },
  { orderStats: 1 }
)
// Latence : < 5ms
```

**Stratégies de mise à jour** :

**1. Synchrone (Immediate Consistency)** :
```javascript
// Lors de chaque nouvelle commande
db.users.updateOne(
  { _id: userId },
  {
    $inc: {
      "orderStats.totalOrders": 1,
      "orderStats.totalSpent": orderTotal,
      "orderStats.currentYear.orders": 1,
      "orderStats.currentYear.spent": orderTotal
    },
    $set: {
      "orderStats.lastOrderDate": new Date(),
      "orderStats.lastOrderValue": orderTotal,
      "orderStats.lastUpdated": new Date()
    }
  }
)

// Avantages : Toujours à jour
// Inconvénients : Overhead d'écriture, deux updates par commande
```

**2. Asynchrone via Change Streams** :
```javascript
// Worker séparé écoute les changements sur orders
const changeStream = db.orders.watch([
  { $match: { "operationType": "insert" } }
]);

changeStream.on("change", async (change) => {
  const order = change.fullDocument;

  await db.users.updateOne(
    { _id: order.userId },
    {
      $inc: { /* updates */ },
      $set: { /* updates */ }
    }
  );
});

// Avantages : Découplé, pas d'impact sur insert orders
// Inconvénients : Eventual consistency (quelques ms de délai)
```

**3. Batch periodique** :
```javascript
// Cron job toutes les N minutes
async function updateUserStats() {
  // Utilise aggregation pour calculer stats
  const stats = await db.orders.aggregate([
    {
      $match: {
        createdAt: { $gte: lastRunTime }  // Seulement nouveaux
      }
    },
    {
      $group: {
        _id: "$userId",
        newOrders: { $sum: 1 },
        newSpent: { $sum: "$total" }
      }
    }
  ]);

  // Bulk update users
  const bulkOps = stats.map(stat => ({
    updateOne: {
      filter: { _id: stat._id },
      update: {
        $inc: {
          "orderStats.totalOrders": stat.newOrders,
          "orderStats.totalSpent": stat.newSpent
        }
      }
    }
  }));

  await db.users.bulkWrite(bulkOps);
}

// Avantages : Efficace, batch processing
// Inconvénients : Staleness (délai = intervalle batch)
```

**Choix de stratégie** :

| Critère | Synchrone | Change Streams | Batch |
|---------|-----------|----------------|-------|
| Cohérence | Immediate | Near real-time (< 1s) | Eventual (minutes) |
| Performance Writes | Impact élevé | Impact faible | Impact très faible |
| Complexité | Faible | Moyenne | Moyenne |
| Use Case | Critical stats | User-facing stats | Analytics, reporting |

### Pattern 4 : Polymorphic Pattern

**Problématique** :
Documents de types similaires mais structures légèrement différentes dans la même collection.

**Cas d'usage** :
Single Table Inheritance, Content Management, Event Sourcing.

**Exemple : Système de notifications**

```javascript
// Différents types de notifications avec champs spécifiques
{
  _id: ObjectId("..."),
  userId: ObjectId("userId"),
  type: "email",  // Discriminator field
  createdAt: ISODate("2025-01-15T10:00:00Z"),
  status: "sent",

  // Champs communs
  title: "Welcome to our platform",
  priority: "high",

  // Champs spécifiques au type email
  email: {
    from: "noreply@example.com",
    to: "user@example.com",
    subject: "Welcome!",
    body: "...",
    deliveryStatus: "delivered"
  }
}

{
  _id: ObjectId("..."),
  userId: ObjectId("userId"),
  type: "push",  // Différent type
  createdAt: ISODate("2025-01-15T10:05:00Z"),
  status: "sent",

  // Champs communs
  title: "New message received",
  priority: "normal",

  // Champs spécifiques au type push
  push: {
    deviceToken: "...",
    badge: 1,
    sound: "default",
    clickAction: "/messages/123"
  }
}

{
  _id: ObjectId("..."),
  userId: ObjectId("userId"),
  type: "sms",
  createdAt: ISODate("2025-01-15T10:10:00Z"),
  status: "pending",

  // Champs communs
  title: "Verification code",
  priority: "critical",

  // Champs spécifiques au type sms
  sms: {
    phoneNumber: "+33612345678",
    message: "Your code is: 123456",
    provider: "twilio"
  }
}
```

**Index strategy** :
```javascript
// Index commun pour tous les types
db.notifications.createIndex({ userId: 1, createdAt: -1 })
db.notifications.createIndex({ type: 1, status: 1 })

// Index spécifiques aux types (sparse)
db.notifications.createIndex(
  { "email.deliveryStatus": 1 },
  { sparse: true }  // Seulement pour documents avec ce champ
)
```

**Requêtes** :
```javascript
// Toutes les notifications d'un user
db.notifications.find({ userId: ObjectId("userId") })

// Notifications email non délivrées
db.notifications.find({
  type: "email",
  "email.deliveryStatus": { $ne: "delivered" }
})

// Polymorphic aggregation
db.notifications.aggregate([
  { $match: { userId: ObjectId("userId") } },
  {
    $group: {
      _id: "$type",
      count: { $sum: 1 },
      // Conditional stats par type
      avgDeliveryTime: {
        $avg: {
          $cond: [
            { $eq: ["$type", "email"] },
            "$email.deliveryTimeMs",
            null
          ]
        }
      }
    }
  }
])
```

**Avantages** :
- Queries simplifiées (1 collection)
- Évolution flexible du schéma par type
- Bon pour données liées conceptuellement

**Attention** :
- Documenter clairement les schémas par type
- Validation schema par type avec discriminator
- Monitoring de la distribution des types

### Pattern 5 : Outlier Pattern

**Problématique** :
Quelques documents ("outliers") ont des caractéristiques exceptionnelles qui pénalisent la majorité.

**Exemple typique** :
99% des users ont < 100 commandes, mais 1% ont > 10,000 commandes.

**Stratégie** :
Gérer différemment les outliers.

```javascript
// Documents normaux (99%)
{
  _id: ObjectId("userId"),
  name: "Regular User",
  hasExcessiveOrders: false,  // Flag

  // Embedded orders (< 100)
  orders: [
    { orderId: ObjectId("..."), date: ISODate("..."), total: 99.99 },
    // ... < 100 orders
  ]
}

// Documents outliers (1%)
{
  _id: ObjectId("powerUserId"),
  name: "Power User",
  hasExcessiveOrders: true,  // Flag

  // Pas d'embedded orders
  // → Utiliser collection séparée
  ordersCount: 15234,
  totalSpent: 234567.89
}

// Collection orders pour outliers
{
  _id: ObjectId("orderId"),
  userId: ObjectId("powerUserId"),
  date: ISODate("..."),
  total: 99.99,
  // ... détails complets
}
```

**Application code** :
```javascript
async function getUserWithOrders(userId) {
  const user = await db.users.findOne({ _id: userId });

  if (user.hasExcessiveOrders) {
    // Outlier : fetch from separate collection
    const orders = await db.orders.find({ userId: userId })
                                   .sort({ date: -1 })
                                   .limit(20)
                                   .toArray();
    user.orders = orders;
  }
  // Sinon, orders déjà embedded

  return user;
}
```

**Avantages** :
- 99% des cas optimisés (1 query)
- 1% gérés correctement sans impacter les autres
- Documents normaux ne dépassent pas 16MB

## Anti-Patterns et Leurs Solutions

### Anti-Pattern 1 : Massive Arrays

**Problème** :
Arrays non bornés qui croissent indéfiniment.

```javascript
// ❌ MAUVAIS : Array unbounded
{
  _id: ObjectId("userId"),
  name: "John Doe",
  activityLog: [
    { action: "login", ts: ISODate("...") },
    { action: "view_page", ts: ISODate("...") },
    // ... potentiellement des millions d'entrées
    // → Document size explosion
    // → Index size explosion sur activityLog
    // → Update performance degradation
  ]
}
```

**Impacts** :
- Document peut atteindre 16MB limit
- Index multikey énorme
- Chaque update réécrit le document entier
- Fragmentation mémoire

**Solution 1 : Capped Array avec rotation**
```javascript
{
  _id: ObjectId("userId"),
  name: "John Doe",
  // Garder seulement les N derniers
  recentActivity: [
    { action: "login", ts: ISODate("...") },
    // ... max 100 entries
  ]
}

// Update avec $push et $slice
db.users.updateOne(
  { _id: userId },
  {
    $push: {
      recentActivity: {
        $each: [{ action: "login", ts: new Date() }],
        $sort: { ts: -1 },
        $slice: 100  // Garde seulement 100 derniers
      }
    }
  }
)
```

**Solution 2 : Collection séparée**
```javascript
// Collection users (sans activity)
{
  _id: ObjectId("userId"),
  name: "John Doe"
}

// Collection activity_log
{
  _id: ObjectId("activityId"),
  userId: ObjectId("userId"),
  action: "login",
  timestamp: ISODate("..."),
  details: { /* ... */ }
}

// Index pour accès rapide
db.activity_log.createIndex({ userId: 1, timestamp: -1 })

// TTL pour rotation automatique (garder 90 jours)
db.activity_log.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 7776000 }  // 90 days
)
```

### Anti-Pattern 2 : Excessive Denormalization

**Problème** :
Dupliquer trop de données, rendant la cohérence difficile à maintenir.

```javascript
// ❌ MAUVAIS : Denormalisation excessive
{
  _id: ObjectId("orderId"),
  orderDate: ISODate("..."),
  total: 299.99,

  // Toutes les données user dupliquées
  customer: {
    id: ObjectId("userId"),
    name: "John Doe",
    email: "john@example.com",
    phone: "+33612345678",
    address: {
      street: "123 Main St",
      city: "Paris",
      zipCode: "75001",
      country: "France"
    },
    preferences: { /* ... */ },
    loyaltyPoints: 1234,
    memberSince: ISODate("..."),
    // ... des dizaines d'autres champs
  }
}

// Problèmes :
// - Update user.email → devoir mettre à jour tous les orders
// - Incohérence potentielle (quelle version de email est correcte?)
// - Gaspillage de stockage
// - Documents orders volumineux
```

**Solution : Denormalisation sélective (Extended Reference)**
```javascript
{
  _id: ObjectId("orderId"),
  orderDate: ISODate("..."),
  total: 299.99,

  // Seulement les données nécessaires et rarement changées
  customer: {
    id: ObjectId("userId"),
    name: "John Doe",           // Nécessaire pour affichage, rarement changé
    email: "john@example.com"   // Nécessaire pour confirmation, peut changer
  },

  // Adresse de livraison snapshot (immutable pour cet order)
  shippingAddress: {
    street: "123 Main St",
    city: "Paris",
    zipCode: "75001",
    country: "France"
  }
}

// Règle : Embedd seulement ce qui est :
// 1. Nécessaire pour l'affichage courant
// 2. Immutable dans ce contexte (snapshot)
// 3. Rarement modifié ou tolérant staleness
```

### Anti-Pattern 3 : Bloated Documents

**Problème** :
Documents approchant la limite de 16MB, ralentissant toutes les opérations.

```javascript
// ❌ MAUVAIS : Document bloated
{
  _id: ObjectId("productId"),
  name: "Product",
  // Très grandes données embedded
  largeImage: "base64encodedimage...",  // 5 MB
  video: "base64encodedvideo...",       // 8 MB
  documentation: "very long text...",   // 2 MB
  reviews: [ /* 1000s of reviews */ ]   // 1 MB
  // Total: 16 MB limit atteinte
}

// Problèmes :
// - Tout fetch de ce produit charge 16MB
// - Network transfer lent
// - Memory pressure
// - Cache inefficace
```

**Solution : Référencer les grandes données**
```javascript
// Document principal (lean)
{
  _id: ObjectId("productId"),
  name: "Product",
  description: "Short description",
  price: 99.99,

  // Références aux grandes données
  mediaIds: {
    images: [ObjectId("img1"), ObjectId("img2")],
    videos: [ObjectId("vid1")],
    documentation: ObjectId("doc1")
  },

  // Summary seulement
  reviewsSummary: {
    count: 1247,
    averageRating: 4.6
  }
}

// Collections séparées
// media_files
{
  _id: ObjectId("img1"),
  type: "image",
  url: "https://cdn.example.com/img1.jpg",
  size: "large",
  mimeType: "image/jpeg"
}

// reviews (voir pattern subset précédent)
```

### Anti-Pattern 4 : Documents as Collections

**Problème** :
Utiliser des documents comme des collections (clés dynamiques).

```javascript
// ❌ MAUVAIS : Clés dynamiques comme collection
{
  _id: ObjectId("statsId"),
  type: "daily_stats",
  // Clé = date, dynamique
  "2025-01-01": { views: 1234, clicks: 567 },
  "2025-01-02": { views: 1456, clicks: 634 },
  "2025-01-03": { views: 1123, clicks: 512 },
  // ... des centaines de dates
}

// Problèmes :
// - Impossibilité d'indexer sur les dates
// - Queries complexes et inefficaces
// - Document size growth unbounded
// - Difficile à query une range de dates
```

**Solution : Structure appropriée**
```javascript
// Collection avec documents par période
{
  _id: ObjectId("statsId"),
  date: ISODate("2025-01-01"),
  views: 1234,
  clicks: 567
}

// Index pour range queries
db.daily_stats.createIndex({ date: 1 })

// Query efficace sur range
db.daily_stats.find({
  date: {
    $gte: ISODate("2025-01-01"),
    $lte: ISODate("2025-01-31")
  }
})
```

**Alternative avec Bucket Pattern** (si très nombreux) :
```javascript
// Un document par mois (bucket)
{
  _id: ObjectId("statsId"),
  month: "2025-01",
  dailyStats: [
    { day: 1, views: 1234, clicks: 567 },
    { day: 2, views: 1456, clicks: 634 },
    // ... 31 days max
  ],
  monthSummary: {
    totalViews: 38456,
    totalClicks: 17234,
    avgDailyViews: 1240
  }
}
```

## Refactoring de Modèle en Production

### Méthodologie de Migration

#### Phase 1 : Analyse de l'Impact

**Audit préalable** :
```javascript
// 1. Taille actuelle de la collection
db.collection.stats()
// Vérifier : size, count, avgObjSize

// 2. Volumétrie des données à migrer
const affectedDocs = db.collection.countDocuments(
  { /* critère de migration */ }
)

// 3. Estimation du temps
// Throughput typique : 1000-5000 docs/sec
const estimatedMinutes = affectedDocs / 3000 / 60

// 4. Impact sur les applications
// Identifier toutes les requêtes affectées
db.system.profile.distinct("command.find", {
  ns: "mydb.collection"
})
```

#### Phase 2 : Stratégie de Migration

**Option A : Migration Big Bang (petit dataset < 1M docs)**
```javascript
// Downtime accepté : Migration complète d'un coup
// 1. Backup
// 2. Maintenance mode
// 3. Migration complète
// 4. Reindex
// 5. Tests
// 6. Restoration du service
```

**Option B : Migration Blue-Green (dataset moyen)**
```javascript
// 1. Créer nouvelle collection avec nouveau schéma
// 2. Dual-write : écrire dans les deux collections
// 3. Backfill ancien vers nouveau (batch processing)
// 4. Validation des données
// 5. Switch read traffic vers nouvelle collection
// 6. Arrêt dual-write
// 7. Cleanup ancienne collection
```

**Option C : Migration Progressive (large dataset > 10M docs)**

```javascript
// Phase 1 : Ajout champ version au schéma
db.collection.updateMany(
  { schemaVersion: { $exists: false } },
  { $set: { schemaVersion: 1 } }
)

// Phase 2 : Application code supporte les deux versions
async function getUser(userId) {
  const user = await db.users.findOne({ _id: userId });

  if (user.schemaVersion === 1) {
    // Ancienne version : migrate on-read
    const migratedUser = migrateToV2(user);

    // Update in background (fire and forget)
    db.users.updateOne(
      { _id: userId },
      { $set: migratedUser }
    ).catch(err => logger.error(err));

    return migratedUser;
  }

  // Nouvelle version : retour direct
  return user;
}

// Phase 3 : Background job pour migration batch
async function migrateBatch() {
  const batch = await db.users.find({
    schemaVersion: 1
  }).limit(1000).toArray();

  const bulkOps = batch.map(user => ({
    updateOne: {
      filter: { _id: user._id },
      update: { $set: migrateToV2(user) }
    }
  }));

  await db.users.bulkWrite(bulkOps);
}

// Phase 4 : Une fois 100% migré, cleanup du code de compatibilité
```

### Cas Pratique : Refactoring E-commerce

**Situation initiale** :
```javascript
// Modèle sous-optimal
{
  _id: ObjectId("orderId"),
  userId: ObjectId("userId"),
  items: [
    {
      productId: ObjectId("p1"),
      quantity: 2,
      price: 49.99
    }
  ],
  total: 99.98,
  status: "completed",
  createdAt: ISODate("...")
}

// Problèmes identifiés :
// 1. Pas d'info produit embedded → $lookup nécessaire
// 2. Pas de pré-calculs → agrégations coûteuses pour analytics
// 3. Status changes → beaucoup de updates
```

**Analyse des patterns d'accès** :
```
- Affichage commande : 80% des requêtes → besoin nom/image produit
- Analytics commandes : 15% → besoin agrégations par produit/catégorie
- Updates status : 5%
```

**Nouveau modèle optimisé** :
```javascript
{
  _id: ObjectId("orderId"),
  userId: ObjectId("userId"),

  items: [
    {
      productId: ObjectId("p1"),
      // Extended reference : données pour affichage
      productName: "Gaming Mouse",
      productImage: "url",
      category: "Electronics",
      // Snapshot prix au moment de la commande
      priceAtOrder: 49.99,
      quantity: 2,
      subtotal: 99.98
    }
  ],

  // Pré-calculés pour analytics
  analytics: {
    totalItems: 2,
    categoriesBreakdown: {
      "Electronics": 99.98
    },
    productIds: [ObjectId("p1")]  // Pour queries
  },

  total: 99.98,

  // Status avec historique
  status: "completed",
  statusHistory: [
    { status: "pending", timestamp: ISODate("...") },
    { status: "processing", timestamp: ISODate("...") },
    { status: "shipped", timestamp: ISODate("...") },
    { status: "completed", timestamp: ISODate("...") }
  ],

  createdAt: ISODate("..."),
  schemaVersion: 2
}
```

**Index strategy** :
```javascript
// Accès user orders
db.orders.createIndex({ userId: 1, createdAt: -1 })

// Analytics par produit
db.orders.createIndex({ "analytics.productIds": 1, createdAt: -1 })

// Analytics par catégorie
db.orders.createIndex({
  "analytics.categoriesBreakdown": 1,
  createdAt: -1
})

// Status queries
db.orders.createIndex({ status: 1, createdAt: -1 })
```

**Gains mesurés** :
```
Avant :
- Affichage commande : 2 queries (order + products lookup), 45ms
- Analytics produit : aggregation full scan, 2500ms

Après :
- Affichage commande : 1 query, 8ms (82% improvement)
- Analytics produit : index scan, 150ms (94% improvement)

Trade-off :
- Document size : +30% (acceptable)
- Write cost : +15% (acceptable pour 5% des ops)
```

## Conclusion

L'optimisation de la modélisation MongoDB requiert une approche holistique :

1. **Analyse approfondie des patterns d'accès** : 80/20 rule
2. **Équilibre read/write performance** : Basé sur les ratios réels
3. **Application judicieuse des patterns** : Attribute, Bucket, Computed, etc.
4. **Évitement des anti-patterns** : Massive arrays, excessive denormalization
5. **Méthodologie rigoureuse de refactoring** : Progressive, testée, mesurée

**Principe cardinal** :
> "Optimize for the queries you run, not for the data you store."

La modélisation n'est jamais figée. Elle doit évoluer avec :
- Les patterns d'usage changeants
- La croissance des données
- Les nouvelles fonctionnalités applicatives
- Les contraintes de performance évolutives

L'excellence en modélisation MongoDB vient de l'expérience, de la mesure continue, et de l'adaptation pragmatique aux besoins réels du système en production.

---

**Points clés à retenir :**
- La modélisation est la fondation de la performance MongoDB
- Analyser les query patterns avant de concevoir le schéma
- Équilibrer denormalisation et cohérence selon read/write ratio
- Appliquer les design patterns appropriés au contexte
- Migrer progressivement avec stratégie définie et mesure d'impact
- Monitorer et adapter le modèle en continu

⏭️ [Optimisation des index](/17-performance-tuning/04-optimisation-index.md)
