🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.7 Activer le Sharding sur une Base et une Collection

## Introduction

L'activation du sharding sur une base de données et ses collections est une étape critique qui détermine la réussite ou l'échec de votre stratégie de distribution horizontale. Une fois le sharding activé avec une shard key donnée, **modifier cette clé nécessite des opérations complexes** (bien que MongoDB 5.0+ facilite ce processus avec la fonctionnalité de resharding).

Cette section détaille les mécanismes d'activation du sharding, les stratégies de choix de shard key, les techniques de pré-splitting, et les anti-patterns à éviter absolument pour garantir des performances optimales à long terme.

---

## Prérequis et Considérations

### Avant d'Activer le Sharding

#### 1. Cluster Shardé Opérationnel

```javascript
// Vérifier l'état du cluster
mongosh --host mongos1.example.com --port 27017
sh.status()

// Résultat attendu :
// - Config servers en Replica Set (3 membres)
// - Au moins 2 shards actifs
// - Balancer activé
```

#### 2. Connexion via Mongos

**IMPORTANT** : Le sharding s'active **uniquement via un mongos**, jamais directement sur un shard.

```javascript
// ✅ Correct : Via mongos
mongosh --host mongos1.example.com --port 27017

// ❌ Incorrect : Directement sur un shard
mongosh --host shardA1.example.com --port 27018
```

#### 3. Analyse de la Charge de Travail

Avant d'activer le sharding, analysez :

| Critère | Questions à se poser |
|---------|---------------------|
| **Volume de données** | Combien de données actuelles ? Croissance prévue ? |
| **Patterns de requêtes** | Quelles sont les requêtes les plus fréquentes ? |
| **Distribution** | Les données sont-elles uniformément distribuées ? |
| **Hot spots** | Y a-t-il des valeurs très populaires (hot keys) ? |
| **Cardinalité** | La shard key candidate a-t-elle une cardinalité élevée ? |
| **Monotonie** | Les insertions sont-elles séquentielles (timestamps, _id) ? |

#### 4. Sauvegarde Préventive

```bash
# Toujours sauvegarder avant d'activer le sharding
mongodump --host mongos1.example.com --port 27017 \
  --db mydb \
  --out /backup/pre-sharding-$(date +%Y%m%d)
```

---

## Phase 1 : Activation du Sharding sur une Base de Données

### Commande de Base

```javascript
// Se connecter au mongos
mongosh --host mongos1.example.com --port 27017

// Activer le sharding sur la base de données
sh.enableSharding("mydb")

// Ou syntaxe alternative
db.adminCommand({ enableSharding: "mydb" })
```

### Vérification

```javascript
// Vérifier que le sharding est activé
use config
db.databases.find({ _id: "mydb" })

// Résultat attendu :
{
  "_id" : "mydb",
  "primary" : "shardA",
  "partitioned" : true,
  "version" : {
    "uuid" : UUID("..."),
    "timestamp" : Timestamp(1640000000, 1),
    "lastMod" : 1
  }
}
```

### Shard Primaire

Lors de l'activation du sharding :
- MongoDB assigne un **shard primaire** à la base de données
- Ce shard stocke les collections **non-shardées** de cette base
- Les collections shardées seront distribuées sur tous les shards

```javascript
// Voir le shard primaire
db.getSiblingDB("config").databases.findOne({ _id: "mydb" }).primary

// Changer le shard primaire (si nécessaire)
db.adminCommand({ movePrimary: "mydb", to: "shardB" })
```

---

## Phase 2 : Choix de la Shard Key

### Critères d'une Bonne Shard Key

Une shard key efficace doit respecter trois propriétés fondamentales :

#### 1. Cardinalité Élevée (High Cardinality)

**Définition** : Nombre de valeurs distinctes possibles pour la shard key.

```javascript
// ✅ Bonne cardinalité : user_id (UUID)
// Millions de valeurs possibles
{ user_id: UUID("550e8400-e29b-41d4-a716-446655440000") }

// ❌ Mauvaise cardinalité : status (3 valeurs seulement)
{ status: "active" }  // 90% des documents
{ status: "inactive" }  // 8% des documents
{ status: "deleted" }  // 2% des documents
```

**Impact** :
- Faible cardinalité → Peu de chunks → Distribution déséquilibrée
- Haute cardinalité → Nombreux chunks → Distribution uniforme

#### 2. Distribution Uniforme (Write Scaling)

**Définition** : Les écritures doivent être réparties uniformément entre les shards.

```javascript
// ❌ Distribution non uniforme : timestamp (monotone)
{ timestamp: ISODate("2024-01-15T10:30:00Z") }
// Toutes les insertions récentes vont sur le même chunk

// ✅ Distribution uniforme : hashed _id
{ _id: "hashed" }
// Les insertions sont distribuées aléatoirement
```

#### 3. Localité des Requêtes (Query Isolation)

**Définition** : Les requêtes fréquentes doivent cibler un seul shard.

```javascript
// ✅ Bonne localité : Requêtes par customer_id
db.orders.find({ customer_id: "CUST12345" })
// Si shard key = { customer_id: 1 } → 1 seul shard interrogé

// ❌ Mauvaise localité : Requêtes multi-shards
db.orders.find({ order_date: { $gte: ISODate("2024-01-01") } })
// Si shard key = { customer_id: 1 } → TOUS les shards interrogés
```

### Tableau de Décision : Choix de Shard Key

| Pattern d'Accès | Shard Key Recommandée | Justification |
|-----------------|----------------------|---------------|
| Recherche par utilisateur | `{ user_id: 1 }` ou `{ user_id: "hashed" }` | Localité des requêtes |
| Séries temporelles | `{ application: 1, timestamp: 1 }` | Évite monotonie + localité |
| Données géographiques | `{ region: 1, user_id: 1 }` | Zone sharding possible |
| Catalogues produits | `{ category: 1, product_id: 1 }` | Distribution + requêtes |
| Logs applicatifs | `{ service: 1, timestamp: 1 }` | Isolation par service |
| Compteurs/métriques | `{ metric_name: "hashed" }` | Distribution uniforme |

---

## Phase 3 : Activation du Sharding sur une Collection

### Syntaxe de Base

```javascript
// Format général
sh.shardCollection("<database>.<collection>", { <shard_key> })

// Exemples
sh.shardCollection("mydb.users", { user_id: 1 })
sh.shardCollection("mydb.events", { timestamp: "hashed" })
sh.shardCollection("mydb.orders", { customer_id: 1, order_id: 1 })
```

### Avec Options Avancées

```javascript
// Syntaxe complète avec options
db.adminCommand({
  shardCollection: "mydb.orders",
  key: { customer_id: 1 },
  unique: false,
  numInitialChunks: 4,
  collation: { locale: "simple" }
})
```

**Options disponibles** :

| Option | Description | Valeur par défaut |
|--------|-------------|-------------------|
| `unique` | Contrainte d'unicité sur la shard key | `false` |
| `numInitialChunks` | Nombre de chunks initiaux (hashed uniquement) | 2 |
| `collation` | Collation pour comparaisons de chaînes | Héritée de la collection |
| `timeseries` | Options pour collections time series | - |

---

## Stratégies de Sharding : Range vs Hashed vs Compound

### 1. Range Sharding (Partitionnement par Plage)

#### Principe

Les données sont partitionnées en **plages contiguës** de valeurs de la shard key.

```javascript
// Exemple : Range sharding sur user_id
sh.shardCollection("mydb.users", { user_id: 1 })

// Distribution des chunks :
// Chunk 1 : { user_id: MinKey } → { user_id: "user_5000" }  → Shard A
// Chunk 2 : { user_id: "user_5000" } → { user_id: "user_10000" }  → Shard B
// Chunk 3 : { user_id: "user_10000" } → { user_id: MaxKey }  → Shard A
```

#### Avantages

✅ **Requêtes par plage efficaces** :
```javascript
// Requête ciblant un seul shard ou peu de shards
db.users.find({ user_id: { $gte: "user_5000", $lt: "user_7000" } })
// → Interroge uniquement le Shard B
```

✅ **Tri naturel** : Les données sont ordonnées sur chaque shard

✅ **Géolocalisation** : Parfait pour le zone sharding

#### Inconvénients

❌ **Hot spots possibles** : Si les écritures se concentrent sur une plage

❌ **Distribution dépendante des données** : Peut devenir déséquilibrée

#### Cas d'Usage Idéaux

- Requêtes par plage fréquentes
- Zone sharding (données géographiques)
- Données avec distribution naturellement uniforme

### 2. Hashed Sharding (Partitionnement par Hachage)

#### Principe

La shard key est **hachée** avant distribution, garantissant une distribution uniforme.

```javascript
// Hashed sharding sur _id
sh.shardCollection("mydb.events", { _id: "hashed" })

// Distribution :
// hash("event_001") = 123456 → Shard A
// hash("event_002") = 789012 → Shard B
// hash("event_003") = 345678 → Shard A
```

#### Avantages

✅ **Distribution uniforme garantie** : Pas de hot spots

✅ **Parfait pour les clés monotones** : _id, timestamp

✅ **Scaling prévisible** : Équilibrage automatique

#### Inconvénients

❌ **Pas de requêtes par plage efficaces** :
```javascript
// ❌ Requête broadcast (tous les shards interrogés)
db.events.find({ _id: { $gte: "event_1000", $lt: "event_2000" } })
```

❌ **Perte de localité** : Données connexes dispersées

#### Cas d'Usage Idéaux

- Insertions à haut débit avec clé monotone
- Distribution uniforme prioritaire
- Pas de requêtes par plage nécessaires

### 3. Compound Sharding (Shard Key Composée)

#### Principe

Combine **plusieurs champs** pour obtenir le meilleur des deux mondes.

```javascript
// Shard key composée : application + timestamp
sh.shardCollection("mydb.logs", { application: 1, timestamp: 1 })

// Distribution :
// { application: "webserver", timestamp: ISODate("2024-01-15T10:00:00Z") } → Shard A
// { application: "database", timestamp: ISODate("2024-01-15T10:00:00Z") } → Shard B
// { application: "webserver", timestamp: ISODate("2024-01-15T11:00:00Z") } → Shard A
```

#### Avantages

✅ **Localité + Distribution** :
```javascript
// Requête ciblée : 1 seul shard
db.logs.find({ application: "webserver", timestamp: { $gte: ISODate("2024-01-15T10:00:00Z") } })

// Évite les hot spots sur timestamp
```

✅ **Flexibilité** : Équilibre entre les propriétés

✅ **Multi-tenancy** : Parfait pour les applications SaaS

#### Inconvénients

❌ **Requêtes sans préfixe** : Moins efficaces
```javascript
// ❌ Requête sans 'application' → broadcast
db.logs.find({ timestamp: ISODate("2024-01-15T10:00:00Z") })
```

❌ **Cardinalité du préfixe critique** : Si peu de valeurs pour `application`

#### Cas d'Usage Idéaux

- Applications multi-tenant (tenant_id + autre champ)
- Séries temporelles par catégorie
- Besoin de localité ET distribution

---

## Exemples Concrets par Type de Données

### Exemple 1 : E-commerce - Collection Orders

**Contexte** :
- 10M de commandes
- Requêtes principalement par `customer_id`
- Certains clients très actifs (hot customers)

**Stratégie** :

```javascript
// Option 1 : Range sharding simple (risque de hot spots)
sh.shardCollection("ecommerce.orders", { customer_id: 1 })

// ✅ Option 2 : Hashed (distribution uniforme)
sh.shardCollection("ecommerce.orders", { customer_id: "hashed" })

// ✅✅ Option 3 : Compound (meilleur compromis)
sh.shardCollection("ecommerce.orders", { customer_id: 1, order_id: 1 })
```

**Justification Option 3** :
- `customer_id` assure la localité des requêtes
- `order_id` ajoute de la granularité pour éviter les hot spots
- Requêtes par customer_id restent efficaces

### Exemple 2 : IoT - Collection Sensor Data

**Contexte** :
- Millions de capteurs
- Insertions continues avec timestamp
- Requêtes par sensor_id et plage de temps

**Stratégie** :

```javascript
// ❌ Mauvais : Timestamp seul (hot spot sur dernières données)
sh.shardCollection("iot.sensor_data", { timestamp: 1 })

// ✅ Bon : Compound avec sensor_id
sh.shardCollection("iot.sensor_data", { sensor_id: 1, timestamp: 1 })

// ✅✅ Meilleur : Hashed sensor_id + timestamp
sh.shardCollection("iot.sensor_data", { sensor_id: "hashed", timestamp: 1 })
```

**Justification** :
- `sensor_id: "hashed"` distribue uniformément les capteurs
- `timestamp` permet les requêtes temporelles par capteur
- Pas de hot spot sur les insertions récentes

### Exemple 3 : Réseaux Sociaux - Collection Posts

**Contexte** :
- Posts de millions d'utilisateurs
- Requêtes par author_id (timeline de l'auteur)
- Requêtes par hashtags (moins fréquentes)

**Stratégie** :

```javascript
// ✅ Stratégie choisie
sh.shardCollection("social.posts", { author_id: "hashed" })

// Créer un index secondaire pour les hashtags
db.posts.createIndex({ hashtags: 1 })
```

**Justification** :
- `author_id: "hashed"` distribue uniformément les auteurs populaires
- Les requêtes par auteur restent efficaces (1 shard)
- Les requêtes par hashtags seront broadcast mais sont moins fréquentes

### Exemple 4 : Logs Applicatifs - Collection Logs

**Contexte** :
- Logs de plusieurs microservices
- Volume massif (100 Go/jour)
- Requêtes par service + plage temporelle
- Rétention de 30 jours (TTL)

**Stratégie** :

```javascript
// ✅ Shard key composée
sh.shardCollection("platform.logs", { service_name: 1, timestamp: 1 })

// Créer un index TTL pour suppression automatique
db.logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 2592000 }  // 30 jours
)
```

**Justification** :
- `service_name` isole les logs par service (localité)
- `timestamp` évite les hot spots et permet les requêtes temporelles
- TTL index gère automatiquement la rétention

### Exemple 5 : Analytics - Collection Page Views

**Contexte** :
- Tracking des pages vues
- Requêtes analytiques par date
- Insertions massives en temps réel

**Stratégie** :

```javascript
// ✅ Hashed sur un identifiant unique
sh.shardCollection("analytics.page_views", { session_id: "hashed" })

// Alternative avec pre-aggregation
sh.shardCollection("analytics.daily_stats", { date: 1, page_url: 1 })
```

**Justification** :
- `session_id: "hashed"` distribue uniformément les insertions
- Pour l'analytics, pré-agréger dans `daily_stats` shardé par date

---

## Pré-splitting et Distribution Initiale

### Pourquoi Pré-splitter ?

Lors du sharding d'une collection existante ou avant une insertion massive :

1. **Sans pré-splitting** :
   - Tous les documents vont initialement dans un seul chunk
   - Le chunk se divise progressivement (overhead)
   - Migrations nombreuses et coûteuses

2. **Avec pré-splitting** :
   - Chunks créés à l'avance
   - Distribution immédiate des insertions
   - Pas de surcharge de splitting/migration

### Pré-splitting pour Range Sharding

```javascript
// Activer le sharding
sh.shardCollection("mydb.users", { user_id: 1 })

// Pré-splitter en 10 chunks
for (var i = 1; i <= 9; i++) {
  sh.splitAt("mydb.users", { user_id: "user_" + (i * 10000) });
}

// Résultat : 10 chunks
// Chunk 1 : MinKey → user_10000
// Chunk 2 : user_10000 → user_20000
// ...
// Chunk 10 : user_90000 → MaxKey
```

### Pré-splitting pour Hashed Sharding

```javascript
// Avec numInitialChunks (MongoDB 4.4+)
sh.shardCollection(
  "mydb.events",
  { _id: "hashed" },
  false,  // unique
  {
    numInitialChunks: 16  // 16 chunks initiaux
  }
)

// Ou manuellement pour versions antérieures
sh.shardCollection("mydb.events", { _id: "hashed" })
for (var i = 0; i < 16; i++) {
  sh.splitFind("mydb.events", { _id: ObjectId() });
}
```

### Distribution Initiale des Chunks

Après le pré-splitting, distribuer manuellement :

```javascript
// Lister les shards disponibles
db.getSiblingDB("config").shards.find()

// Déplacer des chunks vers différents shards
sh.moveChunk("mydb.users", { user_id: "user_10000" }, "shardB")
sh.moveChunk("mydb.users", { user_id: "user_30000" }, "shardC")
sh.moveChunk("mydb.users", { user_id: "user_50000" }, "shardB")

// Vérifier la distribution
db.users.getShardDistribution()
```

### Calcul du Nombre de Chunks Optimal

```javascript
// Formule recommandée :
// numChunks = numShards * 2 * N
// Où N = facteur d'expansion (2-4)

// Exemple : 3 shards, facteur 3
var numShards = 3;
var expansionFactor = 3;
var numChunks = numShards * 2 * expansionFactor;  // 18 chunks

// Pré-splitter
for (var i = 1; i < numChunks; i++) {
  var splitPoint = Math.floor((i / numChunks) * 100000);
  sh.splitAt("mydb.users", { user_id: "user_" + splitPoint });
}
```

---

## Validation et Monitoring

### Vérification Post-Sharding

```javascript
// 1. Confirmer que la collection est shardée
db.collection.stats()
// Rechercher : "sharded" : true

// 2. Voir la distribution des chunks
db.users.getShardDistribution()

// Output :
// Shard shardA at shardA/...
//  data : 1.5GiB docs : 1000000 chunks : 5
//  estimated data per chunk : 300MiB
//  estimated docs per chunk : 200000
//
// Shard shardB at shardB/...
//  data : 1.4GiB docs : 980000 chunks : 5
//  ...
//
// Totals
//  data : 2.9GiB docs : 1980000 chunks : 10

// 3. Détails des chunks
db.getSiblingDB("config").chunks.find({ ns: "mydb.users" }).pretty()

// 4. Index utilisés
db.users.getIndexes()
```

### Métriques à Surveiller

```javascript
// Distribution des documents par shard
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "mydb.users" } },
  { $group: {
      _id: "$shard",
      numChunks: { $sum: 1 }
    }
  }
])

// Taille des données par shard
db.users.stats().shards

// Identifier les jumbo chunks
db.getSiblingDB("config").chunks.find({
  ns: "mydb.users",
  jumbo: true
})
```

### Tests de Performance

```javascript
// Test 1 : Requête ciblée (targeted query)
db.users.find({ user_id: "user_12345" }).explain("executionStats")
// Vérifier : "totalDocsExamined", "executionTimeMillis", "nReturned"

// Test 2 : Requête broadcast (scatter-gather)
db.users.find({ email: /.*@example.com/ }).explain("executionStats")
// Comparer avec la requête ciblée

// Test 3 : Insertion
var startTime = new Date();
for (var i = 0; i < 10000; i++) {
  db.users.insertOne({
    user_id: "user_" + (100000 + i),
    name: "Test User " + i,
    email: "test" + i + "@example.com"
  });
}
var endTime = new Date();
print("Temps d'insertion : " + (endTime - startTime) + " ms");
```

---

## Anti-Patterns et Erreurs Fatales

### ❌ Anti-Pattern 1 : Shard Key de Faible Cardinalité

**Problème** :

```javascript
// Mauvais : Seulement 3 valeurs possibles
sh.shardCollection("mydb.orders", { status: 1 })

// Distribution :
// status: "pending" → 60% des documents → Shard A (surcharge)
// status: "completed" → 35% des documents → Shard B
// status: "cancelled" → 5% des documents → Shard C
```

**Conséquence** :
- Jumbo chunks impossibles à diviser
- Déséquilibre permanent
- Hot spot sur Shard A

**Solution** :

```javascript
// Utiliser une shard key composée avec haute cardinalité
sh.shardCollection("mydb.orders", { customer_id: 1, order_id: 1 })

// Ou hashed
sh.shardCollection("mydb.orders", { customer_id: "hashed" })
```

### ❌ Anti-Pattern 2 : Shard Key Monotone sans Hashing

**Problème** :

```javascript
// Mauvais : _id auto-incrémenté ou timestamp
sh.shardCollection("mydb.events", { _id: 1 })
sh.shardCollection("mydb.logs", { timestamp: 1 })
```

**Conséquence** :
- Toutes les insertions sur le dernier chunk (hot spot)
- Un seul shard reçoit toute la charge d'écriture
- Migrations constantes

**Solution** :

```javascript
// ✅ Hashed sur la clé monotone
sh.shardCollection("mydb.events", { _id: "hashed" })

// ✅ Ou compound avec un préfixe non-monotone
sh.shardCollection("mydb.logs", { application: 1, timestamp: 1 })
```

### ❌ Anti-Pattern 3 : Shard Key Non Incluse dans les Requêtes

**Problème** :

```javascript
// Shard key : { user_id: 1 }
sh.shardCollection("mydb.posts", { user_id: 1 })

// Mais les requêtes principales sont :
db.posts.find({ hashtag: "#mongodb" })  // ❌ Broadcast query
db.posts.find({ created_at: { $gte: ISODate("...") } })  // ❌ Broadcast query
```

**Conséquence** :
- Toutes les requêtes interrogent tous les shards
- Aucun bénéfice du sharding pour les lectures
- Latence multipliée par le nombre de shards

**Solution** :

```javascript
// Choisir une shard key alignée avec les patterns de requêtes
sh.shardCollection("mydb.posts", { hashtag: 1, created_at: 1 })

// Ou créer des index secondaires (mais requêtes restent broadcast)
```

### ❌ Anti-Pattern 4 : Shard Key avec Valeur Null

**Problème** :

```javascript
// Shard key : { category: 1 }
sh.shardCollection("mydb.products", { category: 1 })

// Mais certains documents n'ont pas de category
db.products.insertOne({
  name: "Product X",
  price: 99.99
  // category manquant → null
})
```

**Conséquence** :
- Tous les documents sans `category` vont dans le même chunk
- Jumbo chunk potentiel si beaucoup de documents sans category

**Solution** :

```javascript
// Option 1 : Valeur par défaut
db.products.insertOne({
  name: "Product X",
  category: "uncategorized",  // Valeur par défaut
  price: 99.99
})

// Option 2 : Validation de schéma
db.createCollection("products", {
  validator: {
    $jsonSchema: {
      required: ["category"],
      properties: {
        category: { bsonType: "string" }
      }
    }
  }
})

// Option 3 : Shard key composée avec _id
sh.shardCollection("mydb.products", { category: 1, _id: 1 })
```

### ❌ Anti-Pattern 5 : Sharding Prématuré

**Problème** :

```javascript
// Sharder une collection avec seulement 1 Go de données
sh.shardCollection("mydb.small_collection", { _id: 1 })
```

**Conséquence** :
- Overhead de gestion (chunks, balancer, métadonnées)
- Complexité opérationnelle
- Pas de bénéfice réel

**Solution** :

```javascript
// Règles empiriques pour sharder :
// - Volume > 100 Go
// - Taux d'écriture > 10 000 ops/sec
// - Croissance prévue importante
//
// Sinon : rester sur un Replica Set simple
```

### ❌ Anti-Pattern 6 : Modifier les Données de Shard Key

**Problème** :

```javascript
// Shard key : { user_id: 1 }
sh.shardCollection("mydb.sessions", { user_id: 1 })

// Puis tenter de modifier le user_id
db.sessions.updateOne(
  { _id: ObjectId("...") },
  { $set: { user_id: "new_user_id" } }  // ❌ Erreur dans MongoDB < 4.2
)
```

**Conséquence** :
- MongoDB < 4.2 : Erreur `ImmutableField`
- MongoDB ≥ 4.2 : Migration coûteuse du document entre shards

**Solution** :

```javascript
// Option 1 : Supprimer et recréer (MongoDB < 4.2)
var doc = db.sessions.findOne({ _id: ObjectId("...") });
doc.user_id = "new_user_id";
db.sessions.deleteOne({ _id: ObjectId("...") });
db.sessions.insertOne(doc);

// Option 2 : Éviter de modifier la shard key
// Choisir une shard key immuable dès le départ
```

### ❌ Anti-Pattern 7 : Ignorer le Balancer lors de Maintenances

**Problème** :

```bash
# Effectuer une maintenance sans arrêter le balancer
# Exemple : Ajout de RAM, mise à jour OS

# Le balancer migre des chunks pendant la maintenance
# → Impact performance, risque de timeout
```

**Solution** :

```javascript
// Toujours arrêter le balancer pour les maintenances
sh.stopBalancer()

// Attendre que les migrations en cours se terminent
while (sh.isBalancerRunning()) {
  print("Attente arrêt balancer...");
  sleep(1000);
}

// Effectuer la maintenance...

// Réactiver après
sh.startBalancer()
```

---

## Migration d'une Collection Non-Shardée

### Scénario : Collection Existante à Sharder

**Contexte** :
- Collection `products` avec 50M documents (200 Go)
- Actuellement sur un Replica Set standalone
- Besoin de sharder pour scaling

### Étape 1 : Analyse Préalable

```javascript
// Connexion au Replica Set existant
mongosh --host primary.example.com --port 27017

// Analyser la collection
use mydb
db.products.stats()

// Analyser les index existants
db.products.getIndexes()

// Identifier la meilleure shard key candidate
// Exemple : Requêtes principales par category
db.products.find({ category: "Electronics" }).explain("executionStats")

// Vérifier la cardinalité
db.products.distinct("category").length
```

### Étape 2 : Préparation

```bash
# 1. Sauvegarde complète
mongodump --host primary.example.com --db mydb --collection products \
  --out /backup/pre-sharding-products

# 2. Créer l'index pour la shard key (si inexistant)
mongosh --host primary.example.com
use mydb
db.products.createIndex({ category: 1, product_id: 1 })
```

### Étape 3 : Intégration au Cluster Shardé

```javascript
// Connexion au cluster shardé
mongosh --host mongos1.example.com --port 27017

// Ajouter le Replica Set existant comme shard
sh.addShard("existingRS/primary.example.com:27017,secondary1.example.com:27017,secondary2.example.com:27017")

// Vérifier
sh.status()
```

### Étape 4 : Activation du Sharding

```javascript
// Activer le sharding sur la base
sh.enableSharding("mydb")

// Arrêter le balancer temporairement
sh.stopBalancer()

// Pré-splitter (critique pour une grosse collection)
sh.shardCollection("mydb.products", { category: 1, product_id: 1 })

// Pré-splitter manuellement par catégorie
var categories = ["Electronics", "Clothing", "Books", "Home", "Sports", "Toys"];
categories.forEach(function(cat) {
  sh.splitAt("mydb.products", { category: cat, product_id: MinKey });
});

// Réactiver le balancer
sh.startBalancer()
```

### Étape 5 : Monitoring de la Migration

```javascript
// Surveiller la distribution
db.products.getShardDistribution()

// Surveiller les migrations
db.getSiblingDB("config").changelog.find({
  ns: "mydb.products",
  what: "moveChunk.commit"
}).sort({ time: -1 }).limit(10)

// Surveiller le balancer
sh.isBalancerRunning()

// Métriques détaillées
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "mydb.products" } },
  { $group: {
      _id: "$shard",
      chunks: { $sum: 1 }
    }
  }
])
```

### Étape 6 : Validation

```javascript
// Test de lecture ciblée
db.products.find({ category: "Electronics", product_id: "PROD12345" }).explain("executionStats")
// Vérifier : 1 seul shard interrogé

// Test de lecture broadcast
db.products.find({ price: { $lt: 100 } }).explain("executionStats")
// Comparer performance avant/après

// Test d'écriture
db.products.insertOne({
  category: "Electronics",
  product_id: "PROD99999",
  name: "Test Product",
  price: 99.99
})
```

---

## Bonnes Pratiques de Production

### 1. Planification de la Shard Key

```javascript
// Checklist avant activation :
// ✅ Cardinalité analysée (>1000 valeurs distinctes idéal)
// ✅ Patterns de requêtes documentés
// ✅ Distribution des valeurs vérifiée (pas de skew)
// ✅ Tests effectués en staging avec données réelles
// ✅ Plan de rollback défini
```

### 2. Pré-splitting Systématique

```javascript
// Pour toute collection > 1 Go
// Règle : numChunks = numShards * 4 (minimum)

var numShards = db.getSiblingDB("config").shards.count();
var numChunks = numShards * 4;

// Pré-splitter avant insertion massive
```

### 3. Monitoring Continu

```javascript
// Script de monitoring quotidien
function checkShardingHealth(dbName, collName) {
  var ns = dbName + "." + collName;

  // 1. Distribution des chunks
  var chunkDist = db.getSiblingDB("config").chunks.aggregate([
    { $match: { ns: ns } },
    { $group: { _id: "$shard", count: { $sum: 1 } } }
  ]).toArray();

  print("=== Distribution des chunks ===");
  printjson(chunkDist);

  // 2. Jumbo chunks
  var jumboCount = db.getSiblingDB("config").chunks.count({
    ns: ns,
    jumbo: true
  });

  if (jumboCount > 0) {
    print("⚠️  ATTENTION : " + jumboCount + " jumbo chunks détectés !");
  }

  // 3. Migrations récentes (dernières 24h)
  var migrations = db.getSiblingDB("config").changelog.count({
    ns: ns,
    what: "moveChunk.commit",
    time: { $gte: new Date(Date.now() - 86400000) }
  });

  print("Migrations (24h) : " + migrations);
}

// Exécuter
checkShardingHealth("mydb", "users");
```

### 4. Documentation

Maintenir une documentation à jour pour chaque collection shardée :

```yaml
# collection-sharding-metadata.yml

collections:
  - name: mydb.users
    shard_key:
      fields: { user_id: "hashed" }
      reason: "Distribution uniforme des utilisateurs"
      decided_by: "team-backend"
      date: "2024-01-15"

    statistics:
      num_chunks: 48
      size: "500 GB"
      num_docs: 50000000

    query_patterns:
      - pattern: "find({ user_id: ... })"
        frequency: "90%"
        targeted: true

      - pattern: "find({ email: ... })"
        frequency: "8%"
        targeted: false

    indexes:
      - { user_id: "hashed" }  # Shard key index
      - { email: 1 }           # Secondary index
      - { created_at: -1 }     # Secondary index
```

### 5. Tests de Charge

```javascript
// Avant de passer en production
// Test avec 10x le volume prévu

// Test d'écriture
var startTime = new Date();
var numInserts = 1000000;

for (var i = 0; i < numInserts; i++) {
  db.users.insertOne({
    user_id: UUID(),
    name: "User " + i,
    email: "user" + i + "@example.com",
    created_at: new Date()
  });

  if (i % 10000 == 0) {
    print("Inserted " + i + " documents");
  }
}

var endTime = new Date();
var duration = (endTime - startTime) / 1000;
var throughput = numInserts / duration;

print("Durée : " + duration + " secondes");
print("Throughput : " + throughput.toFixed(2) + " insertions/sec");
```

---

## Résumé : Processus Décisionnel

```
┌─────────────────────────────────────────┐
│  Analyse des Besoins                    │
│  - Volume actuel et prévu               │
│  - Patterns de requêtes                 │
│  - Taux d'écriture/lecture              │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Choix de la Shard Key                  │
│  - Cardinalité élevée ?                 │
│  - Distribution uniforme ?              │
│  - Localité des requêtes ?              │
│                                         │
│  Range / Hashed / Compound ?            │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Tests en Staging                       │
│  - Pré-splitting                        │
│  - Tests de charge                      │
│  - Validation des performances          │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Activation en Production               │
│  - Sauvegarde                           │
│  - sh.enableSharding()                  │
│  - sh.shardCollection()                 │
│  - Pré-splitting                        │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│  Monitoring et Ajustements              │
│  - Distribution des chunks              │
│  - Jumbo chunks                         │
│  - Performances des requêtes            │
│  - Balancer activity                    │
└─────────────────────────────────────────┘
```

---

## Conclusion

L'activation du sharding sur une base et une collection est une décision stratégique qui impacte durablement les performances et l'évolutivité de votre système. Les points clés à retenir :

- ✅ **Choisir une shard key optimale** : Cardinalité, distribution, localité
- ✅ **Pré-splitter systématiquement** : Pour collections volumineuses
- ✅ **Tester exhaustivement** : En staging avec données réalistes
- ✅ **Monitorer activement** : Distribution, jumbo chunks, migrations
- ✅ **Éviter les anti-patterns** : Shard keys monotones, faible cardinalité, non-alignées

Une shard key bien choisie = performances optimales à long terme.
Une shard key mal choisie = problèmes croissants et migration coûteuse.

**Investissez le temps nécessaire dans la phase d'analyse et de tests.**

---

## Ressources

- [MongoDB Documentation - Shard Keys](https://docs.mongodb.com/manual/core/sharding-shard-key/)
- [MongoDB Documentation - Choose a Shard Key](https://docs.mongodb.com/manual/core/sharding-choose-a-shard-key/)
- [MongoDB Best Practices - Sharding](https://docs.mongodb.com/manual/core/sharding-data-partitioning/)
- [MongoDB Blog - Shard Key Selection](https://www.mongodb.com/blog)

---


⏭️ [Migration des chunks](/10-sharding/08-migration-chunks.md)
