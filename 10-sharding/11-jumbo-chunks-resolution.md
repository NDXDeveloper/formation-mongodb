🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.11 Jumbo Chunks et Résolution

## Introduction

Les **jumbo chunks** représentent l'un des problèmes les plus courants et les plus frustrants dans la gestion d'un cluster shardé MongoDB. Un jumbo chunk est un chunk qui dépasse la taille maximale configurée (par défaut 64 Mo) mais qui **ne peut pas être divisé** par le processus normal de splitting. Cette situation paralyse le balancer, empêche une distribution équilibrée des données, et peut conduire à des déséquilibres massifs de charge entre les shards.

Comprendre les causes profondes des jumbo chunks, savoir les détecter rapidement, et maîtriser les différentes stratégies de résolution sont des compétences essentielles pour tout administrateur de cluster shardé MongoDB en production. Cette section explore en profondeur ce phénomène et fournit des solutions éprouvées pour le résoudre et, surtout, le prévenir.

---

## Qu'est-ce qu'un Jumbo Chunk ?

### Définition Technique

```javascript
// Un chunk devient "jumbo" quand :
// 1. Sa taille dépasse le seuil configuré (chunkSize)
// 2. Il ne peut pas être divisé (split) car toutes ses valeurs de shard key sont identiques

// Exemple de jumbo chunk dans les métadonnées
db.getSiblingDB("config").chunks.findOne({
  ns: "mydb.orders",
  jumbo: true
})

// Résultat :
{
  "_id": ObjectId("..."),
  "ns": "mydb.orders",
  "min": { "status": "active" },
  "max": { "status": "inactive" },
  "shard": "shardA",
  "jumbo": true,          // ⚠️ Marqué comme jumbo
  "lastmod": Timestamp(123, 45),
  "history": [...]
}
```

### Anatomie d'un Jumbo Chunk

```
┌─────────────────────────────────────────────────────────┐
│                    CHUNK NORMAL                         │
│  min: { user_id: "user_1000" }                          │
│  max: { user_id: "user_2000" }                          │
│  Taille: 45 MB                                          │
│  Documents: 50,000 avec user_id différents              │
│  ✅ Peut être split en 2 morceaux                       │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                    JUMBO CHUNK                          │
│  min: { status: "active" }                              │
│  max: { status: "inactive" }                            │
│  Taille: 2.5 GB (> 64 MB)                               │
│  Documents: 3,000,000 TOUS avec status="active"         │
│  ❌ IMPOSSIBLE à split (même valeur de shard key)       │
│  jumbo: true                                            │
└─────────────────────────────────────────────────────────┘
```

### Mécanisme de Création

```javascript
// Scénario typique de création d'un jumbo chunk

// 1. Collection avec shard key de faible cardinalité
sh.shardCollection("mydb.orders", { status: 1 })

// 2. Distribution initiale (3 valeurs possibles)
// Chunk 1: { status: MinKey } → { status: "active" }    → shardA
// Chunk 2: { status: "active" } → { status: "inactive" } → shardB
// Chunk 3: { status: "inactive" } → { status: MaxKey }   → shardC

// 3. Insertion massive de documents avec status="active"
for (var i = 0; i < 3000000; i++) {
  db.orders.insertOne({
    status: "active",  // Même valeur pour tous !
    customer_id: "CUST" + i,
    amount: Math.random() * 1000
  });
}

// 4. Le chunk 2 grossit démesurément
// Taille: 2.5 GB
// MongoDB tente de le splitter mais ÉCHOUE
// → Toutes les valeurs sont identiques (status="active")
// → Chunk marqué "jumbo"

// 5. Le balancer ne peut plus migrer ce chunk
// Log: "Cannot move chunk: chunk is jumbo"
```

---

## Causes des Jumbo Chunks

### 1. Shard Key de Faible Cardinalité

**Cause principale** : Le champ choisi comme shard key n'a que peu de valeurs distinctes.

```javascript
// ❌ Exemples de shard keys problématiques

// Exemple 1 : Statut (3 valeurs)
sh.shardCollection("mydb.orders", { status: 1 })
// status ∈ {"pending", "completed", "cancelled"}

// Exemple 2 : Booléen (2 valeurs)
sh.shardCollection("mydb.users", { is_active: 1 })
// is_active ∈ {true, false}

// Exemple 3 : Type (5 valeurs)
sh.shardCollection("mydb.products", { category: 1 })
// category ∈ {"electronics", "clothing", "books", "home", "sports"}

// Exemple 4 : Région (4 valeurs)
sh.shardCollection("mydb.customers", { region: 1 })
// region ∈ {"north", "south", "east", "west"}
```

**Impact** :

```javascript
// Si 90% des documents ont status="active"
// Le chunk correspondant contiendra 90% des données

// Exemple avec 10M de documents :
// - Chunk "active" : 9M documents (2.7 GB) → JUMBO
// - Chunk "completed" : 800K documents (240 MB)
// - Chunk "cancelled" : 200K documents (60 MB)

// Distribution catastrophique :
// shardA : 2.7 GB (90%)
// shardB : 240 MB (8%)
// shardC : 60 MB (2%)
```

### 2. Distribution Non Uniforme (Skewed Data)

**Cause** : Même avec cardinalité correcte, certaines valeurs sont beaucoup plus fréquentes.

```javascript
// Exemple : E-commerce avec quelques gros clients

// Shard key : customer_id (bonne cardinalité : 1M clients)
sh.shardCollection("mydb.orders", { customer_id: 1 })

// Mais distribution réelle :
// - customer_AMAZON : 2M commandes (1.5 GB) → JUMBO
// - customer_WALMART : 1M commandes (750 MB) → JUMBO
// - 999,998 autres clients : 7M commandes au total

// "Hot customers" créent des jumbo chunks
```

### 3. Shard Key Composée avec Préfixe de Faible Cardinalité

```javascript
// ❌ Mauvais ordre des champs

// Exemple 1 : Type d'abord (faible cardinalité)
sh.shardCollection("mydb.events", { event_type: 1, timestamp: 1 })
// Si event_type a 5 valeurs, seulement 5 chunks initiaux possibles

// Exemple 2 : Région d'abord
sh.shardCollection("mydb.sales", { region: 1, customer_id: 1 })
// Si région EU a 80% des ventes → jumbo chunk région EU
```

### 4. Absence de Pré-splitting lors de l'Import

```javascript
// ❌ Scénario problématique

// 1. Activer le sharding sans pré-splitting
sh.shardCollection("mydb.products", { category: 1, product_id: 1 })

// Chunk initial unique :
// min: { category: MinKey, product_id: MinKey }
// max: { category: MaxKey, product_id: MaxKey }

// 2. Import massif (10M produits)
// Tous les documents vont dans le chunk initial

// 3. MongoDB tente de splitter pendant l'import
// Mais si beaucoup de produits ont category="electronics"
// → Création de jumbo chunk
```

### 5. Croissance Organique Non Contrôlée

```javascript
// Chunk initialement normal qui devient jumbo avec le temps

// T0 : Chunk avec 50K documents, 30 MB
{
  min: { category: "electronics", product_id: "PROD_0000" },
  max: { category: "electronics", product_id: "PROD_5000" }
}

// T+6 mois : Croissance du catalogue électronique
// 500K documents, 350 MB → JUMBO

// Cause : Pas de monitoring de la croissance des chunks
```

---

## Impact des Jumbo Chunks

### 1. Paralysie du Balancer

```javascript
// Le balancer ne peut pas migrer les jumbo chunks

// Tentative de migration :
sh.moveChunk("mydb.orders", { status: "active" }, "shardB")

// Erreur :
{
  "ok": 0,
  "errmsg": "chunk too large to move: 2684354560 bytes > 268435456 bytes (configured chunk size)",
  "code": 13439
}

// Log du balancer :
"Skipping chunk migration because chunk is jumbo: { ns: 'mydb.orders', min: { status: 'active' }, max: { status: 'inactive' } }"
```

**Conséquence** : Le déséquilibre s'aggrave avec le temps.

### 2. Déséquilibre de Charge Permanent

```javascript
// État du cluster avec jumbo chunks

sh.status()

// Output :
shards:
  { "_id": "shardA", "host": "...", "state": 1 }
  { "_id": "shardB", "host": "...", "state": 1 }
  { "_id": "shardC", "host": "...", "state": 1 }

databases:
  mydb.orders
    shard key: { "status": 1 }
    unique: false
    balancing: true
    chunks:
      shardA  1       // ⚠️ 1 chunk mais 2.5 GB !
      shardB  45      // 45 chunks mais seulement 500 MB
      shardC  45      // 45 chunks mais seulement 500 MB

    // Distribution :
    // shardA : 2.5 GB (83%)  ← JUMBO CHUNK
    // shardB : 250 MB (8.3%)
    // shardC : 250 MB (8.3%)
```

### 3. Performances Dégradées

```javascript
// Impact sur les performances

// Requêtes ciblant le jumbo chunk (shardA surchargé)
db.orders.find({ status: "active", amount: { $gt: 1000 } })

// Métriques sur shardA :
// - CPU: 85% (vs 20% sur shardB et shardC)
// - RAM: Cache WiredTiger saturé (90%)
// - IOPS: 5000 read/s (vs 500 sur les autres shards)
// - Latence: 150ms (vs 20ms sur les autres shards)

// Impact utilisateur :
// - Timeouts applicatifs
// - Expérience dégradée
```

### 4. Scaling Inefficace

```javascript
// Ajout d'un nouveau shard pour résoudre le problème

sh.addShard("shardD/shardD1:27018,shardD2:27018,shardD3:27018")

// Résultat attendu : Redistribution de la charge
// Résultat réel : shardD reste presque vide

// Le jumbo chunk reste bloqué sur shardA
// Les nouveaux chunks (petits) se redistribuent
// Mais le problème de fond persiste

sh.status()
// shardA : 2.5 GB (jumbo)
// shardB : 200 MB
// shardC : 200 MB
// shardD : 100 MB
// → shardA toujours surchargé !
```

### 5. Risque lors des Maintenances

```bash
# Scénario : Maintenance sur shardA (qui contient le jumbo chunk)

# 1. Tentative de drain avant maintenance
db.adminCommand({ removeShard: "shardA" })

# 2. Le balancer tente de migrer les chunks
# ✅ Chunks normaux : migrés en quelques heures
# ❌ Jumbo chunk : BLOQUÉ

# Résultat :
{
  "msg": "draining ongoing",
  "state": "ongoing",
  "remaining": {
    "chunks": 1,  // Le jumbo chunk !
    "dbs": 0
  },
  "note": "Cannot move jumbo chunk. Manual intervention required."
}

# 3. Maintenance IMPOSSIBLE sans résoudre le jumbo chunk d'abord
```

---

## Détection des Jumbo Chunks

### Méthode 1 : Requête Directe sur Config

```javascript
// Identifier tous les jumbo chunks
db.getSiblingDB("config").chunks.find({ jumbo: true })

// Avec détails
db.getSiblingDB("config").chunks.find({ jumbo: true }).forEach(function(chunk) {
  printjson({
    namespace: chunk.ns,
    shard: chunk.shard,
    min: chunk.min,
    max: chunk.max,
    lastModified: chunk.lastmod
  });
});
```

### Méthode 2 : Agrégation par Collection

```javascript
// Compter les jumbo chunks par collection
db.getSiblingDB("config").chunks.aggregate([
  { $match: { jumbo: true } },
  { $group: {
      _id: "$ns",
      count: { $sum: 1 },
      shards: { $addToSet: "$shard" }
    }
  },
  { $sort: { count: -1 } }
])

// Résultat :
[
  {
    "_id": "mydb.orders",
    "count": 1,
    "shards": ["shardA"]
  },
  {
    "_id": "mydb.events",
    "count": 3,
    "shards": ["shardB", "shardC"]
  }
]
```

### Méthode 3 : Via sh.status()

```javascript
// sh.status() indique les jumbo chunks
sh.status()

// Output (extrait) :
mydb.orders
  shard key: { "status": 1 }
  chunks:
    shardA  1
      { "status": { "$minKey": 1 } } -->> { "status": "active" } on: shardA Timestamp(1, 0) jumbo
    shardB  45
      ...
```

### Méthode 4 : Script de Diagnostic Complet

```javascript
// Script complet pour analyser les jumbo chunks
function diagnoseJumboChunks() {
  print("=== DIAGNOSTIC DES JUMBO CHUNKS ===\n");

  var jumboChunks = db.getSiblingDB("config").chunks.find({
    jumbo: true
  }).toArray();

  if (jumboChunks.length === 0) {
    print("✅ Aucun jumbo chunk détecté\n");
    return;
  }

  print("⚠️  " + jumboChunks.length + " jumbo chunk(s) détecté(s)\n");

  // Grouper par collection
  var byCollection = {};

  jumboChunks.forEach(function(chunk) {
    if (!byCollection[chunk.ns]) {
      byCollection[chunk.ns] = [];
    }
    byCollection[chunk.ns].push(chunk);
  });

  // Analyser chaque collection
  Object.keys(byCollection).forEach(function(ns) {
    var chunks = byCollection[ns];

    print("Collection : " + ns);
    print("  Jumbo chunks : " + chunks.length);

    // Récupérer la shard key
    var collInfo = db.getSiblingDB("config").collections.findOne({ _id: ns });
    print("  Shard key : " + JSON.stringify(collInfo.key));

    // Analyser chaque jumbo chunk
    chunks.forEach(function(chunk, index) {
      print("\n  Jumbo Chunk #" + (index + 1) + " :");
      print("    Shard : " + chunk.shard);
      print("    Min : " + JSON.stringify(chunk.min));
      print("    Max : " + JSON.stringify(chunk.max));

      // Estimer la taille (approximatif)
      try {
        var dbName = ns.split(".")[0];
        var collName = ns.split(".")[1];

        var count = db.getSiblingDB(dbName)[collName].countDocuments({
          $and: [
            { [Object.keys(chunk.min)[0]]: { $gte: chunk.min[Object.keys(chunk.min)[0]] } },
            { [Object.keys(chunk.max)[0]]: { $lt: chunk.max[Object.keys(chunk.max)[0]] } }
          ]
        });

        print("    Documents (estimés) : " + count);

        // Estimer taille (1 KB par document en moyenne)
        var estimatedSizeMB = (count * 1024) / 1024 / 1024;
        print("    Taille estimée : " + estimatedSizeMB.toFixed(2) + " MB");

      } catch (e) {
        print("    ⚠️  Impossible d'estimer la taille");
      }

      // Analyser la cardinalité de la shard key
      try {
        var shardKeyField = Object.keys(collInfo.key)[0];
        var distinctCount = db.getSiblingDB(dbName)[collName].distinct(shardKeyField, {
          $and: [
            { [shardKeyField]: { $gte: chunk.min[shardKeyField] } },
            { [shardKeyField]: { $lt: chunk.max[shardKeyField] } }
          ]
        }).length;

        print("    Valeurs distinctes (shard key) : " + distinctCount);

        if (distinctCount === 1) {
          print("    ⚠️  PROBLÈME : Toutes les valeurs sont identiques !");
          print("    → Ce chunk ne peut PAS être splitté");
        } else if (distinctCount < 10) {
          print("    ⚠️  ATTENTION : Très faible cardinalité (" + distinctCount + " valeurs)");
        }

      } catch (e) {
        print("    ⚠️  Impossible d'analyser la cardinalité");
      }
    });

    // Recommandations
    print("\n  Recommandations :");

    var shardKeyFields = Object.keys(collInfo.key);

    if (shardKeyFields.length === 1) {
      print("    1. ⚠️  Shard key simple → Considérer une shard key composée");
      print("       Exemple : " + JSON.stringify({
        [shardKeyFields[0]]: 1,
        "_id": 1
      }));
    }

    print("    2. Utiliser refineCollectionShardKey (MongoDB 4.4+)");
    print("       db.adminCommand({");
    print("         refineCollectionShardKey: \"" + ns + "\",");
    print("         key: { " + shardKeyFields[0] + ": 1, additionalField: 1 }");
    print("       })");

    print("    3. Ou forcer la migration avec attemptToBalanceJumboChunks (MongoDB 4.4+)");

    print("    4. En dernier recours : Resharding complet (MongoDB 5.0+)");
    print("       db.adminCommand({");
    print("         reshardCollection: \"" + ns + "\",");
    print("         key: { newShardKey: 1 }");
    print("       })");

    print("\n" + "=".repeat(60) + "\n");
  });
}

// Exécuter
diagnoseJumboChunks();
```

---

## Stratégies de Résolution

### Stratégie 1 : Refine Collection Shard Key (MongoDB 4.4+)

**Principe** : Ajouter un champ à la shard key existante pour augmenter la granularité.

```javascript
// Situation initiale
sh.shardCollection("mydb.orders", { status: 1 })
// → Jumbo chunk sur status="active"

// Solution : Ajouter un champ à la shard key
db.adminCommand({
  refineCollectionShardKey: "mydb.orders",
  key: { status: 1, customer_id: 1 }  // Ajouter customer_id
})

// Processus :
// 1. MongoDB ajoute customer_id à la shard key
// 2. Les chunks existants peuvent maintenant être splittés
// 3. Chunk { status: "active" } devient splittable en :
//    - { status: "active", customer_id: MinKey } → { status: "active", customer_id: "CUST_5000" }
//    - { status: "active", customer_id: "CUST_5000" } → { status: "active", customer_id: "CUST_10000" }
//    - etc.

// Validation
sh.status()
// Le jumbo chunk n'est plus marqué jumbo
// Il peut maintenant être splitté et migré
```

**Prérequis** :

```javascript
// 1. Créer un index compatible
db.orders.createIndex({ status: 1, customer_id: 1 })

// 2. Vérifier la cardinalité du nouveau champ
db.orders.aggregate([
  { $match: { status: "active" } },
  { $group: { _id: "$customer_id" } },
  { $count: "distinctCustomers" }
])
// Doit retourner un nombre élevé (idéalement > 1000)

// 3. Exécuter refineCollectionShardKey
// (Voir commande ci-dessus)
```

**Limitations** :

```javascript
// - Nécessite MongoDB 4.4+
// - Le nouveau champ doit exister dans tous les documents
// - L'index doit être créé avant
// - Pas de downgrade possible après
```

### Stratégie 2 : Splitter Manuellement avec un Suffix

**Principe** : Si le chunk contient plusieurs valeurs distinctes (mais pas assez pour un split auto), forcer le split.

```javascript
// Situation : Chunk avec 2 valeurs distinctes

// Chunk actuel :
{
  min: { category: "electronics", product_id: MinKey },
  max: { category: "home", product_id: MinKey },
  jumbo: true
}

// Contient :
// - category="electronics" : 1.8 GB
// - category="food" : 700 MB

// Solution : Splitter entre les deux valeurs
sh.splitAt("mydb.products", {
  category: "food",
  product_id: MinKey
})

// Résultat : 2 chunks
// Chunk 1 : { category: "electronics" } → { category: "food" }     (1.8 GB)
// Chunk 2 : { category: "food" } → { category: "home" }            (700 MB)

// Chunk 1 reste gros mais plus petit
// Chunk 2 est de taille acceptable

// Si chunk 1 est encore trop gros et contient uniquement "electronics"
// → Nécessite stratégie 1 (refine) ou stratégie 4 (reshape)
```

### Stratégie 3 : Forcer la Migration (MongoDB 4.4+)

**Principe** : Autoriser MongoDB à migrer les jumbo chunks malgré leur taille.

```javascript
// Activer l'option globale
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  { $set: { attemptToBalanceJumboChunks: true } },
  { upsert: true }
)

// Le balancer tentera maintenant de migrer les jumbo chunks
// ⚠️ ATTENTION : Migration très lente et coûteuse en ressources

// Monitoring de la migration
db.currentOp({
  op: "command",
  "command.moveChunk": { $exists: true }
})

// Résultat :
{
  "opid": "shardA:12345",
  "op": "command",
  "ns": "mydb.orders",
  "command": { "moveChunk": "mydb.orders", ... },
  "msg": "Cloning phase: 1500000/3000000 documents",  // Progression
  "microsecs_running": 1800000000  // 30 minutes !
}

// Une fois terminé, vérifier
sh.status()
// Le jumbo chunk devrait être migré sur un autre shard

// Désactiver après (recommandé)
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  { $set: { attemptToBalanceJumboChunks: false } }
)
```

**Considérations** :

```javascript
// Avantages :
// - Pas de modification de la shard key
// - Le balancer gère automatiquement

// Inconvénients :
// - Très lent (plusieurs heures pour GB de données)
// - Impact performance important pendant la migration
// - Risque de timeout si chunk trop volumineux (> 5 GB)
// - Ne résout pas le problème de fond (shard key inadéquate)

// Recommandé uniquement pour :
// - Jumbo chunks temporaires (pic de données)
// - Migration one-shot avant refactoring
// - Cluster avec beaucoup de ressources disponibles
```

### Stratégie 4 : Resharding Complet (MongoDB 5.0+)

**Principe** : Changer complètement la shard key de la collection.

```javascript
// Situation : Shard key fondamentalement inadéquate
// Shard key actuelle : { status: 1 }
// → Impossible à corriger avec refine

// Solution : Resharding complet
db.adminCommand({
  reshardCollection: "mydb.orders",
  key: { customer_id: "hashed" },  // Nouvelle shard key complètement différente
  numInitialChunks: 10  // Nombre de chunks initiaux
})

// Processus interne (automatisé par MongoDB) :
// 1. Création d'une collection temporaire avec la nouvelle shard key
// 2. Copie de toutes les données vers la nouvelle collection
// 3. Synchronisation incrémentale des modifications
// 4. Basculement atomique
// 5. Suppression de l'ancienne collection

// Monitoring du resharding
db.getSiblingDB("config").reshardingOperations.find().pretty()

// Résultat (après quelques heures) :
{
  "ns": "mydb.orders",
  "key": { "customer_id": "hashed" },
  "state": "done",
  "durationMillis": 14400000  // 4 heures
}

// Validation
sh.status()
// Collection maintenant shardée sur customer_id
// Plus de jumbo chunks
```

**Considérations** :

```javascript
// Avantages :
// - Résolution définitive du problème
// - Nouvelle shard key optimale
// - Distribution équilibrée garantie

// Inconvénients :
// - Très lourd (copie complète des données)
// - Plusieurs heures à jours selon volume
// - Consommation importante de ressources (CPU, disque, réseau)
// - Nécessite MongoDB 5.0+

// Planification recommandée :
// 1. Tester en staging avec données réelles
// 2. Planifier une fenêtre de maintenance longue
// 3. Augmenter temporairement les ressources du cluster
// 4. Monitoring continu pendant l'opération
// 5. Plan de rollback préparé
```

### Stratégie 5 : Recréation de la Collection

**Principe** : Solution radicale quand les autres stratégies ne sont pas disponibles (MongoDB < 4.4).

```javascript
// ⚠️ DERNIÈRE OPTION : Downtime important

// Étape 1 : Exporter les données
mongodump \
  --host mongos1.example.com \
  --db mydb \
  --collection orders \
  --out /backup/orders-resharding

// Étape 2 : Créer une nouvelle collection avec bonne shard key
db.orders_new.createIndex({ customer_id: "hashed" })

sh.shardCollection("mydb.orders_new", { customer_id: "hashed" })

// Étape 3 : Pré-splitter pour distribution optimale
for (var i = 0; i < 20; i++) {
  sh.splitAt("mydb.orders_new", { customer_id: "hash_value_" + i });
}

// Étape 4 : Importer les données
mongorestore \
  --host mongos1.example.com \
  --db mydb \
  --collection orders_new \
  /backup/orders-resharding/mydb/orders.bson

// Étape 5 : Basculer les applications
// Modifier la connection string pour pointer vers orders_new

// Étape 6 : Renommer (nécessite downtime)
db.adminCommand({
  renameCollection: "mydb.orders",
  to: "mydb.orders_old",
  dropTarget: false
})

db.adminCommand({
  renameCollection: "mydb.orders_new",
  to: "mydb.orders",
  dropTarget: false
})

// Étape 7 : Valider et supprimer l'ancienne
db.orders_old.drop()
```

---

## Prévention des Jumbo Chunks

### 1. Choix d'une Shard Key Appropriée

```javascript
// ✅ Critères d'une bonne shard key (révision)

// 1. Cardinalité élevée
// ✅ Bon : user_id (millions de valeurs)
// ❌ Mauvais : status (3 valeurs)

// 2. Distribution uniforme
// ✅ Bon : UUID v4, hashed _id
// ❌ Mauvais : timestamp (monotone), région (skewed)

// 3. Localité des requêtes
// ✅ Bon : Requêtes fréquentes incluent la shard key
// ❌ Mauvais : Requêtes n'utilisent jamais la shard key

// Exemples de bonnes shard keys :
sh.shardCollection("users", { user_id: "hashed" })
sh.shardCollection("orders", { customer_id: 1, order_date: 1 })
sh.shardCollection("events", { application: 1, timestamp: 1 })
sh.shardCollection("iot_data", { sensor_id: "hashed", timestamp: 1 })
```

### 2. Analyse Préalable des Données

```javascript
// Avant de sharder, analyser la distribution

function analyzeShardKeyCandidate(dbName, collName, shardKeyField) {
  print("=== Analyse de la Shard Key Candidate ===\n");

  var coll = db.getSiblingDB(dbName)[collName];

  // 1. Cardinalité
  var distinctCount = coll.distinct(shardKeyField).length;
  var totalDocs = coll.countDocuments({});

  print("Cardinalité :");
  print("  Valeurs distinctes : " + distinctCount);
  print("  Total documents : " + totalDocs);
  print("  Ratio : " + (distinctCount / totalDocs * 100).toFixed(2) + "%");

  if (distinctCount < 100) {
    print("  ⚠️  ATTENTION : Cardinalité très faible (< 100)");
    print("  → Risque élevé de jumbo chunks !");
  } else if (distinctCount < 1000) {
    print("  ⚠️  ATTENTION : Cardinalité faible (< 1000)");
    print("  → Considérer une shard key composée");
  } else {
    print("  ✅ Cardinalité acceptable");
  }
  print("");

  // 2. Distribution
  var distribution = coll.aggregate([
    { $group: { _id: "$" + shardKeyField, count: { $sum: 1 } } },
    { $sort: { count: -1 } },
    { $limit: 10 }
  ]).toArray();

  print("Distribution (Top 10) :");

  var maxCount = distribution[0].count;
  var minCount = distribution[distribution.length - 1].count;

  distribution.forEach(function(item) {
    var percent = (item.count / totalDocs * 100).toFixed(2);
    print("  " + item._id + " : " + item.count + " docs (" + percent + "%)");
  });

  var skew = (maxCount / (totalDocs / distinctCount)).toFixed(2);
  print("\n  Skew factor : " + skew);

  if (skew > 5) {
    print("  ⚠️  ALERTE : Distribution très déséquilibrée (skew > 5)");
    print("  → Une valeur représente " + skew + "x la moyenne");
    print("  → Risque élevé de jumbo chunks !");
  } else if (skew > 2) {
    print("  ⚠️  ATTENTION : Distribution déséquilibrée (skew > 2)");
  } else {
    print("  ✅ Distribution uniforme");
  }
  print("");

  // 3. Recommandation
  print("Recommandation :");

  if (distinctCount < 1000 || skew > 5) {
    print("  ❌ Shard key NON RECOMMANDÉE");
    print("  → Utiliser une shard key composée ou hashed");
    print("  Exemples :");
    print("    - { " + shardKeyField + ": \"hashed\" }");
    print("    - { " + shardKeyField + ": 1, _id: 1 }");
  } else if (distinctCount < 10000 || skew > 2) {
    print("  ⚠️  Shard key ACCEPTABLE mais non optimale");
    print("  → Envisager une shard key composée");
  } else {
    print("  ✅ Shard key RECOMMANDÉE");
  }
}

// Exemple d'utilisation
analyzeShardKeyCandidate("mydb", "orders", "status");
// → Révèle les problèmes potentiels AVANT le sharding
```

### 3. Pré-splitting Intelligent

```javascript
// Créer les chunks à l'avance pour distribution optimale

// Approche 1 : Pré-split basé sur les valeurs existantes
function presplitCollection(dbName, collName, shardKey, numSplits) {
  var ns = dbName + "." + collName;

  print("Pré-splitting de " + ns + " en " + numSplits + " chunks...\n");

  // Activer le sharding
  sh.enableSharding(dbName);
  sh.shardCollection(ns, shardKey);

  // Obtenir les valeurs pour splits
  var shardKeyField = Object.keys(shardKey)[0];
  var coll = db.getSiblingDB(dbName)[collName];

  // Calculer les percentiles
  var total = coll.countDocuments({});
  var step = Math.floor(total / numSplits);

  var splitPoints = [];
  for (var i = 1; i < numSplits; i++) {
    var doc = coll.find().sort({ [shardKeyField]: 1 }).skip(i * step).limit(1).toArray()[0];
    if (doc) {
      splitPoints.push(doc[shardKeyField]);
    }
  }

  // Effectuer les splits
  splitPoints.forEach(function(point) {
    try {
      sh.splitAt(ns, { [shardKeyField]: point });
      print("✅ Split à : " + point);
    } catch (e) {
      print("❌ Erreur split à " + point + " : " + e.message);
    }
  });

  print("\nPré-splitting terminé");
  sh.status();
}

// Exemple
presplitCollection("mydb", "orders", { customer_id: 1 }, 20);
```

### 4. Monitoring Continu

```javascript
// Script de monitoring automatisé (à exécuter quotidiennement)

function monitorChunkGrowth() {
  print("=== Monitoring de la Croissance des Chunks ===\n");

  var collections = db.getSiblingDB("config").collections.find({
    dropped: false
  }).toArray();

  var warnings = [];

  collections.forEach(function(coll) {
    if (coll.key) {
      // Analyser la distribution des chunks
      var chunks = db.getSiblingDB("config").chunks.aggregate([
        { $match: { ns: coll._id } },
        { $group: {
            _id: "$shard",
            numChunks: { $sum: 1 }
          }
        }
      ]).toArray();

      if (chunks.length > 0) {
        var max = Math.max(...chunks.map(c => c.numChunks));
        var min = Math.min(...chunks.map(c => c.numChunks));
        var imbalance = ((max - min) / min) * 100;

        if (imbalance > 30) {
          warnings.push({
            collection: coll._id,
            type: "imbalance",
            severity: "warning",
            message: "Déséquilibre de " + imbalance.toFixed(1) + "%"
          });
        }
      }

      // Vérifier les jumbo chunks
      var jumboCount = db.getSiblingDB("config").chunks.countDocuments({
        ns: coll._id,
        jumbo: true
      });

      if (jumboCount > 0) {
        warnings.push({
          collection: coll._id,
          type: "jumbo",
          severity: "critical",
          message: jumboCount + " jumbo chunk(s) détecté(s)"
        });
      }

      // Vérifier les chunks proches de la limite
      // (nécessite un custom exporter pour la taille réelle)
    }
  });

  // Afficher les alertes
  if (warnings.length === 0) {
    print("✅ Aucune alerte\n");
  } else {
    print("⚠️  " + warnings.length + " alerte(s) détectée(s) :\n");

    warnings.forEach(function(warning) {
      var icon = warning.severity === "critical" ? "🔴" : "⚠️ ";
      print(icon + " " + warning.collection);
      print("   Type : " + warning.type);
      print("   " + warning.message);
      print("");
    });

    // Envoyer une notification (email, Slack, etc.)
    // sendAlert(warnings);
  }
}

// Exécuter
monitorChunkGrowth();

// À automatiser via cron :
// 0 9 * * * mongosh --host mongos1 --eval "load('/scripts/monitor_chunks.js'); monitorChunkGrowth()"
```

### 5. Limites de Taille Proactives

```javascript
// Configurer une taille de chunk plus petite pour collections critiques

// Par défaut : 64 MB
// Pour collections sensibles : 32 MB ou moins

db.getSiblingDB("config").settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 32 } },  // 32 MB
  { upsert: true }
)

// ⚠️ Impact :
// - Plus de chunks → Plus de métadonnées
// - Migrations plus fréquentes mais plus rapides
// - Meilleure granularité de distribution

// Recommandé pour :
// - Collections avec croissance rapide
// - Shard keys avec cardinalité moyenne (1000-10000)
// - Clusters avec beaucoup de shards (> 5)
```

---

## Anti-Patterns à Éviter

### ❌ Anti-Pattern 1 : Ignorer les Alertes Jumbo Chunks

**Problème** :

```javascript
// Jumbo chunks détectés mais ignorés
// "On verra plus tard, ça fonctionne pour l'instant"

db.getSiblingDB("config").chunks.find({ jumbo: true }).count()
// 3 jumbo chunks

// 6 mois plus tard :
db.getSiblingDB("config").chunks.find({ jumbo: true }).count()
// 15 jumbo chunks !

// Problème devenu ingérable
```

**Conséquence** :
- Déséquilibre croissant
- Résolution de plus en plus complexe
- Coût de correction augmente exponentiellement

**Solution** :
```javascript
// Traiter immédiatement au premier jumbo chunk
// Mise en place d'alertes automatiques
// Review hebdomadaire de la distribution
```

### ❌ Anti-Pattern 2 : Tenter de Splitter un Jumbo Chunk Monovalué

**Problème** :

```javascript
// Jumbo chunk avec une seule valeur de shard key
// Tentative de split manuel

sh.splitAt("mydb.orders", { status: "active", _id: ObjectId("...") })

// Erreur :
{
  "ok": 0,
  "errmsg": "split point must be different from existing boundaries",
  "code": 141
}

// Ou pire : split réussit mais crée 2 chunks dont 1 reste jumbo
```

**Conséquence** :
- Perte de temps
- Faux espoir de résolution
- Chunk toujours bloqué

**Solution** :
```javascript
// Diagnostiquer d'abord la cardinalité
// Si cardinalité = 1 → Refine ou resharding obligatoire
// Pas de raccourci possible
```

### ❌ Anti-Pattern 3 : Migration Forcée Sans Monitoring

**Problème** :

```bash
# Activer attemptToBalanceJumboChunks
# Puis partir en weekend sans surveillance

db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  { $set: { attemptToBalanceJumboChunks: true } }
)

# Lundi matin : Cluster saturé
# Migration a pris 48h et n'est pas terminée
# Performance dégradée pendant tout le weekend
```

**Conséquence** :
- Impact utilisateurs prolongé
- Saturation des ressources
- Possibles timeouts et échecs

**Solution** :
```bash
# Migration forcée uniquement pendant fenêtre de maintenance
# Monitoring continu
# Ressources cluster augmentées temporairement
# Plan de rollback préparé
```

### ❌ Anti-Pattern 4 : Resharding en Production Sans Test

**Problème** :

```javascript
// Resharding directement en production
// Sans test en staging

db.adminCommand({
  reshardCollection: "mydb.orders",
  key: { customer_id: "hashed" }
})

// Découverte après coup :
// - Durée : 12h (au lieu de 4h estimées)
// - Nouvelle shard key inadéquate (requêtes broadcast)
// - Downgrade impossible
```

**Conséquence** :
- Opération irréversible
- Problèmes découverts trop tard
- Downtime prolongé

**Solution** :
```bash
# 1. Restaurer backup production en staging
# 2. Tester resharding complet
# 3. Valider performances avec nouvelle shard key
# 4. Estimer durée réelle
# 5. Planifier fenêtre de maintenance adaptée
# 6. Exécuter en production avec équipe complète
```

### ❌ Anti-Pattern 5 : Shard Key "Temporaire"

**Problème** :

```javascript
// "On choisit cette shard key pour l'instant"
// "On changera plus tard si nécessaire"

sh.shardCollection("mydb.orders", { status: 1 })

// 2 ans plus tard :
// - 500M de documents
// - Jumbo chunks ingérables
// - Resharding nécessiterait 1 semaine de downtime
// - Impossible à corriger sans impact majeur
```

**Conséquence** :
- Dette technique croissante
- Correction devient prohibitive
- Cluster bloqué avec mauvaise architecture

**Solution** :
```javascript
// Choisir la BONNE shard key dès le départ
// Analyse approfondie avant sharding
// Tests avec données réalistes
// "Mesurer deux fois, couper une fois"
```

---

## Cas Pratiques de Résolution

### Cas 1 : E-commerce avec Catégories

**Contexte** :

```javascript
// Collection products shardée sur category
sh.shardCollection("ecommerce.products", { category: 1 })

// Après 1 an :
// - category="Electronics" : 2M produits (1.5 GB) → JUMBO
// - Autres catégories : 500K produits au total
```

**Résolution** :

```javascript
// Étape 1 : Diagnostic
diagnoseJumboChunks()
// → 1 jumbo chunk sur category="Electronics"
// → Cardinalité du chunk : 1 (toutes les valeurs identiques)

// Étape 2 : Analyse des requêtes applicatives
// Requêtes principales :
// - find({ category: "Electronics", brand: "Samsung" })
// - find({ category: "Electronics", price: { $lt: 500 } })
// → brand et price sont des champs pertinents

// Étape 3 : Refine avec brand
db.products.createIndex({ category: 1, brand: 1 })

db.adminCommand({
  refineCollectionShardKey: "ecommerce.products",
  key: { category: 1, brand: 1 }
})

// Étape 4 : Validation
sh.status()
// Chunk "Electronics" maintenant splittable :
// - { category: "Electronics", brand: "Apple" }
// - { category: "Electronics", brand: "Samsung" }
// - { category: "Electronics", brand: "Sony" }
// etc.

// Distribution équilibrée retrouvée
```

### Cas 2 : SaaS Multi-tenant

**Contexte** :

```javascript
// Application SaaS avec 10K tenants
sh.shardCollection("saas.documents", { tenant_id: 1 })

// Problème : 1 gros client (tenant_BIGCORP)
// - 5M documents (3 GB) → JUMBO
// - Autres clients : 10M documents au total mais bien distribués
```

**Résolution** :

```javascript
// Option 1 : Refine (si MongoDB 4.4+)
db.documents.createIndex({ tenant_id: 1, document_id: 1 })

db.adminCommand({
  refineCollectionShardKey: "saas.documents",
  key: { tenant_id: 1, document_id: 1 }
})

// Option 2 : Si MongoDB < 4.4, migration forcée
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  { $set: { attemptToBalanceJumboChunks: true } }
)

// Monitoring pendant 24h
// Migration du jumbo chunk vers un autre shard

// Option 3 : Isolation du gros client (solution architecturale)
// Créer un shard dédié pour BIGCORP
sh.addShardTag("shardBIGCORP", "bigcorp")

sh.addTagRange(
  "saas.documents",
  { tenant_id: "tenant_BIGCORP" },
  { tenant_id: "tenant_BIGCORP\xff" },  // \xff = valeur max
  "bigcorp"
)

// Le gros client reste isolé, ne perturbe plus le cluster
```

### Cas 3 : Logs Applicatifs

**Contexte** :

```javascript
// Logs shardés sur service_name
sh.shardCollection("platform.logs", { service_name: 1, timestamp: 1 })

// Service "api-gateway" génère 80% des logs
// Chunk { service_name: "api-gateway" } → 5 GB → JUMBO
```

**Résolution** :

```javascript
// Solution : Hashed sur service_name
// Nécessite resharding (MongoDB 5.0+)

db.adminCommand({
  reshardCollection: "platform.logs",
  key: { service_name: "hashed", timestamp: 1 },
  numInitialChunks: 16
})

// Résultat :
// - Logs api-gateway distribués uniformément sur tous les shards
// - Plus de jumbo chunks
// - Performance restaurée

// Alternative si MongoDB < 5.0 :
// Refine avec un champ supplémentaire
db.logs.createIndex({ service_name: 1, timestamp: 1, request_id: 1 })

db.adminCommand({
  refineCollectionShardKey: "platform.logs",
  key: { service_name: 1, timestamp: 1, request_id: 1 }
})
```

---

## Checklist de Gestion des Jumbo Chunks

```yaml
detection:
  quotidienne:
    - Vérifier présence de jumbo chunks
    - Commande: db.getSiblingDB("config").chunks.find({ jumbo: true }).count()
    - Alerte si count > 0

  hebdomadaire:
    - Analyser la distribution par collection
    - Script: checkChunkDistribution()
    - Alerte si déséquilibre > 20%

prevention:
  avant_sharding:
    - Analyser la cardinalité de la shard key candidate
    - Analyser la distribution des valeurs
    - Tester avec données réelles en staging
    - Pré-splitter pour distribution optimale

  en_production:
    - Monitoring continu de la croissance
    - Review mensuelle de la distribution
    - Alertes automatiques sur déséquilibre

resolution:
  immediat:
    - Diagnostiquer la cause (script diagnoseJumboChunks)
    - Analyser la cardinalité du chunk
    - Évaluer l'impact sur le cluster

  court_terme:
    - MongoDB 4.4+ : refineCollectionShardKey
    - MongoDB 5.0+ : reshardCollection si nécessaire
    - < 4.4 : Migration forcée ou recréation

  long_terme:
    - Documentation de l'incident
    - Post-mortem pour prévention future
    - Amélioration du monitoring
```

---

## Conclusion

Les jumbo chunks sont un symptôme, pas une maladie. Ils révèlent presque toujours un problème de conception : **une shard key inadéquate**. Les points clés à retenir :

- ✅ **Prévention > Correction** : Choisir la bonne shard key dès le départ
- ✅ **Détection précoce** : Monitoring automatisé et alertes
- ✅ **Action rapide** : Traiter au premier jumbo chunk détecté
- ✅ **Outils modernes** : refineCollectionShardKey (4.4+) et reshardCollection (5.0+)
- ✅ **Tests rigoureux** : Toujours tester en staging avant production
- ✅ **Documentation** : Apprendre de chaque incident

**Philosophie** : Les jumbo chunks ne sont pas une fatalité. Avec une conception soigneuse, une analyse préalable approfondie, et un monitoring proactif, ils sont largement évitables. Et quand ils surviennent malgré tout, les versions modernes de MongoDB offrent des outils puissants pour les résoudre sans downtime majeur.

**Investissez dans le choix de la shard key : c'est la décision la plus importante lors du déploiement d'un cluster shardé.**

---

## Ressources

- [MongoDB Documentation - Jumbo Chunks](https://docs.mongodb.com/manual/core/sharding-data-partitioning/#jumbo-chunks)
- [MongoDB Documentation - refineCollectionShardKey](https://docs.mongodb.com/manual/reference/command/refineCollectionShardKey/)
- [MongoDB Documentation - reshardCollection](https://docs.mongodb.com/manual/reference/command/reshardCollection/)
- [MongoDB Blog - Resharding in MongoDB 5.0](https://www.mongodb.com/blog)

---


⏭️ [Bonnes pratiques de sharding](/10-sharding/12-bonnes-pratiques-sharding.md)
