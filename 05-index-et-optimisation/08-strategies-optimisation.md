🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.8 Stratégies d'optimisation des requêtes

## Introduction

Maintenant que vous comprenez les index et le Query Planner, il est temps d'apprendre les **stratégies concrètes** pour optimiser vos requêtes MongoDB. L'optimisation n'est pas une science exacte, mais plutôt un ensemble de techniques et de bonnes pratiques à appliquer selon votre contexte.

Dans ce chapitre, nous allons explorer :
- 🎯 Les **principes fondamentaux** d'optimisation
- 🔧 Les **techniques pratiques** éprouvées
- 📊 Les **patterns** d'optimisation courants
- ⚡ Les **anti-patterns** à éviter
- 🚀 Des **exemples concrets** avant/après

L'objectif : rendre vos requêtes MongoDB rapides, efficaces et scalables.

---

## Les 10 principes fondamentaux d'optimisation

### Principe 1 : Mesurer avant d'optimiser

```
"On ne peut pas améliorer ce qu'on ne mesure pas"

Le cycle d'optimisation :
═════════════════════════

1. MESURER
   └─ Utiliser explain("executionStats")
   └─ Noter les métriques actuelles

2. IDENTIFIER
   └─ Trouver le goulot d'étranglement
   └─ Comprendre le problème

3. OPTIMISER
   └─ Appliquer une technique
   └─ Modifier un index

4. VALIDER
   └─ Re-mesurer avec explain()
   └─ Comparer avant/après

5. DOCUMENTER
   └─ Noter l'amélioration
   └─ Expliquer le changement
```

**Exemple** :

```javascript
// 1. MESURER (avant)
const before = db.orders.find({ status: "pending" })
  .explain("executionStats")
console.log(`Avant : ${before.executionStats.executionTimeMillis}ms`)

// 2. IDENTIFIER le problème
// → COLLSCAN sur 1M documents

// 3. OPTIMISER
db.orders.createIndex({ status: 1 })

// 4. VALIDER (après)
const after = db.orders.find({ status: "pending" })
  .explain("executionStats")
console.log(`Après : ${after.executionStats.executionTimeMillis}ms`)
console.log(`Amélioration : ${before.executionStats.executionTimeMillis / after.executionStats.executionTimeMillis}x`)

// 5. DOCUMENTER
// Index créé le 2024-12-01
// Amélioration : 3500ms → 15ms (233x plus rapide)
// Raison : Élimination du COLLSCAN sur 1M documents
```

### Principe 2 : L'index approprié avant tout

```
Un bon index > Tout autre optimisation

Priorité des optimisations :
═══════════════════════════

1. Index approprié         ████████████ Impact : 100x - 1000x
2. Index composé optimal   ██████████   Impact : 10x - 100x
3. Index avec options      ████████     Impact : 5x - 20x
4. Optimisation requête    ████         Impact : 2x - 5x
5. Paramètres serveur      ██           Impact : 1.1x - 2x
```

### Principe 3 : La règle ESR

Pour les index composés, suivez toujours la règle **ESR** :

```
E = Equality (Égalité)
S = Sort (Tri)
R = Range (Plage)

Ordre optimal dans l'index :
════════════════════════════

1. EQUALITY - Les filtres d'égalité exacte
   { status: "pending" }
   { userId: 12345 }

2. SORT - Les champs de tri
   .sort({ createdAt: -1 })

3. RANGE - Les filtres de plage
   { price: { $gte: 10, $lte: 100 } }
   { age: { $gt: 18 } }
```

**Exemple** :

```javascript
// Requête
db.orders.find({
  userId: 12345,              // E - Equality
  status: "pending",          // E - Equality
  amount: { $gte: 100 }       // R - Range
}).sort({
  createdAt: -1               // S - Sort
})

// Index optimal selon ESR
db.orders.createIndex({
  userId: 1,        // E - Equality en premier
  status: 1,        // E - Equality en second
  createdAt: -1,    // S - Sort en troisième
  amount: 1         // R - Range en dernier
})
```

### Principe 4 : Viser un ratio de 100%

```
Ratio d'efficacité = nReturned / totalDocsExamined

Objectifs :
═══════════

100%     ★★★★★ PARFAIT - Chaque document examiné est retourné
80-99%   ★★★★  EXCELLENT
50-79%   ★★★   BON
20-49%   ★★    ACCEPTABLE
< 20%    ★     MAUVAIS - Beaucoup de gaspillage
< 5%           CRITIQUE - Optimisation urgente
```

### Principe 5 : Privilégier la lecture sur l'écriture

```
Les index accélèrent les lectures mais ralentissent les écritures

Équation d'optimisation :
═════════════════════════

Bénéfice index = (Lectures × Gain lecture) - (Écritures × Coût écriture)

Si Lectures >> Écritures :
└─ ✅ Index très bénéfique

Si Lectures ≈ Écritures :
└─ ⚠️ Évaluer le compromis

Si Écritures >> Lectures :
└─ ❌ Peut-être éviter l'index
```

### Principe 6 : Éviter les requêtes sur tableaux quand possible

```javascript
// ❌ LENT : Requête sur tableau
{
  tags: ["mongodb", "database", "nosql"]  // Tableau
}
db.articles.find({ tags: "mongodb" })
// → Index multikey, moins efficace

// ✅ RAPIDE : Champ dédupliqué
{
  tags: ["mongodb", "database", "nosql"],
  primaryTag: "mongodb"                    // Champ simple
}
db.articles.createIndex({ primaryTag: 1 })
db.articles.find({ primaryTag: "mongodb" })
// → Index simple, plus efficace
```

### Principe 7 : Limiter les résultats tôt

```javascript
// ❌ MAUVAIS : Limite appliquée tard
db.posts.find()
  .sort({ publishedAt: -1 })
  .skip(100)
  .limit(10)
// → Trie TOUS les documents puis skip

// ✅ BON : Index pour tri + limite
db.posts.createIndex({ publishedAt: -1 })
db.posts.find()
  .sort({ publishedAt: -1 })
  .skip(100)
  .limit(10)
// → Parcourt l'index directement, s'arrête à 110
```

### Principe 8 : Projections pour réduire les données

```javascript
// ❌ LENT : Récupère tous les champs
db.users.find({ city: "Paris" })
// → Transfère beaucoup de données

// ✅ RAPIDE : Ne récupère que le nécessaire
db.users.find(
  { city: "Paris" },
  { name: 1, email: 1, _id: 0 }
)
// → Transfère moins de données
// → Peut être une covered query
```

### Principe 9 : Éviter les expressions coûteuses

```javascript
// ❌ TRÈS LENT : $where avec JavaScript
db.users.find({
  $where: function() {
    return this.age > 18 && this.status === "active"
  }
})
// → Exécute JavaScript pour chaque document
// → Ne peut pas utiliser d'index

// ✅ RAPIDE : Opérateurs natifs
db.users.find({
  age: { $gt: 18 },
  status: "active"
})
// → Peut utiliser un index
```

### Principe 10 : Penser en termes de batch

```javascript
// ❌ LENT : Une requête par document
for (let userId of userIds) {
  db.users.findOne({ _id: userId })
}
// → 1000 requêtes pour 1000 utilisateurs

// ✅ RAPIDE : Batch avec $in
db.users.find({
  _id: { $in: userIds }
})
// → 1 requête pour 1000 utilisateurs
```

---

## Stratégies d'optimisation par scénario

### Scénario 1 : Recherche simple lente

**Problème** :

```javascript
// Requête lente
db.products.find({ sku: "PROD-12345" })

// explain() montre :
{
  "stage": "COLLSCAN",
  "executionTimeMillis": 3500,
  "totalDocsExamined": 5000000
}
```

**Solution** :

```javascript
// Créer un index unique sur SKU
db.products.createIndex({ sku: 1 }, { unique: true })

// Résultat :
{
  "stage": "IXSCAN",
  "indexName": "sku_1",
  "executionTimeMillis": 2,
  "totalDocsExamined": 1
}

// Amélioration : 3500ms → 2ms (1750x plus rapide)
```

### Scénario 2 : Filtres multiples

**Problème** :

```javascript
// Requête avec plusieurs filtres
db.orders.find({
  userId: 12345,
  status: "pending",
  amount: { $gte: 100 }
})

// Avec index simple sur userId
{
  "stage": "FETCH",
  "inputStage": { "stage": "IXSCAN", "indexName": "userId_1" },
  "executionTimeMillis": 145,
  "totalDocsExamined": 5000,
  "nReturned": 50
}
// Ratio : 50/5000 = 1% (mauvais)
```

**Solution** :

```javascript
// Index composé selon ESR
db.orders.createIndex({
  userId: 1,      // E - Equality
  status: 1,      // E - Equality
  amount: 1       // R - Range
})

// Résultat :
{
  "stage": "IXSCAN",
  "indexName": "userId_1_status_1_amount_1",
  "executionTimeMillis": 8,
  "totalDocsExamined": 50,
  "nReturned": 50
}
// Ratio : 50/50 = 100% (parfait)
// Amélioration : 145ms → 8ms (18x plus rapide)
```

### Scénario 3 : Tri sans index

**Problème** :

```javascript
// Tri sur grande collection
db.posts.find({ status: "published" })
  .sort({ publishedAt: -1 })
  .limit(20)

// explain() montre :
{
  "stage": "LIMIT",
  "inputStage": {
    "stage": "SORT",              // ⚠️ Tri en mémoire !
    "sortPattern": { "publishedAt": -1 },
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "status_1"
    }
  },
  "executionTimeMillis": 1250
}
```

**Solution** :

```javascript
// Index composé incluant le champ de tri
db.posts.createIndex({
  status: 1,        // E - Equality
  publishedAt: -1   // S - Sort
})

// Résultat :
{
  "stage": "LIMIT",
  "inputStage": {
    "stage": "IXSCAN",            // Tri via l'index !
    "indexName": "status_1_publishedAt_-1"
  },
  "executionTimeMillis": 5
}
// Amélioration : 1250ms → 5ms (250x plus rapide)
```

### Scénario 4 : Requêtes OR inefficaces

**Problème** :

```javascript
// OR sur champs différents
db.users.find({
  $or: [
    { email: "user@example.com" },
    { username: "user123" }
  ]
})

// Sans index : COLLSCAN
// Avec index sur email seulement : Inefficace
```

**Solution A : Deux index** :

```javascript
// Créer un index sur chaque champ
db.users.createIndex({ email: 1 })
db.users.createIndex({ username: 1 })

// MongoDB utilisera les deux index et fusionnera les résultats
// explain() montre :
{
  "stage": "SUBPLAN",
  "inputStage": {
    "stage": "OR",
    "inputStages": [
      { "stage": "IXSCAN", "indexName": "email_1" },
      { "stage": "IXSCAN", "indexName": "username_1" }
    ]
  }
}
```

**Solution B : Dénormalisation** (si applicable) :

```javascript
// Si les requêtes OR sont très fréquentes,
// considérer une dénormalisation

// Document étendu
{
  _id: ObjectId("..."),
  email: "user@example.com",
  username: "user123",
  loginIdentifiers: [          // ← Tableau combiné
    "user@example.com",
    "user123"
  ]
}

// Index multikey
db.users.createIndex({ loginIdentifiers: 1 })

// Requête simplifiée
db.users.find({ loginIdentifiers: "user123" })
```

### Scénario 5 : Comptage lent

**Problème** :

```javascript
// Comptage sur grande collection
db.orders.countDocuments({ status: "pending" })

// Sans index : Très lent (COLLSCAN sur millions de docs)
```

**Solution** :

```javascript
// Index sur le champ de filtre
db.orders.createIndex({ status: 1 })

// Ou utiliser estimatedDocumentCount() si précision pas critique
db.orders.estimatedDocumentCount()
// → Utilise les métadonnées, instantané
// → Mais compte TOUS les documents (pas de filtre)
```

### Scénario 6 : Pagination inefficace

**Problème** :

```javascript
// Pagination avec skip/limit
// Page 1000 : skip(50000).limit(50)
db.posts.find()
  .sort({ createdAt: -1 })
  .skip(50000)      // ⚠️ Parcourt 50000 docs !
  .limit(50)

// Très lent sur les pages éloignées
```

**Solution A : Pagination par curseur** :

```javascript
// Première page
const page1 = db.posts.find()
  .sort({ createdAt: -1 })
  .limit(50)

const lastDoc = page1[page1.length - 1]

// Page suivante (avec curseur)
const page2 = db.posts.find({
  createdAt: { $lt: lastDoc.createdAt }
})
  .sort({ createdAt: -1 })
  .limit(50)

// Toujours rapide, même pour page 1000
// Car ne parcourt que 50 documents
```

**Solution B : Index avec _id** (si tri naturel) :

```javascript
// Utiliser _id pour pagination
// _id contient un timestamp

// Page suivante
db.posts.find({
  _id: { $lt: lastSeenId }
})
  .sort({ _id: -1 })
  .limit(50)

// L'index _id est automatique et toujours trié
```

### Scénario 7 : Regex non optimisé

**Problème** :

```javascript
// Regex qui ne peut pas utiliser d'index
db.users.find({
  email: { $regex: /example.com$/ }    // ⚠️ Fin de chaîne
})

// OU
db.users.find({
  name: { $regex: /.*john.*/i }        // ⚠️ Wildcard au début
})

// Ne peut pas utiliser l'index → COLLSCAN
```

**Solution** :

```javascript
// Pour préfixes : Regex optimisé
db.users.find({
  email: { $regex: /^john/ }           // ✅ Début de chaîne
})
// Peut utiliser l'index email_1

// Pour recherche full-text : Index texte
db.users.createIndex({ name: "text" })
db.users.find({
  $text: { $search: "john" }
})

// Pour suffixes : Dénormalisation
{
  email: "john@example.com",
  emailDomain: "example.com"           // ← Champ dédié
}
db.users.createIndex({ emailDomain: 1 })
db.users.find({ emailDomain: "example.com" })
```

### Scénario 8 : Agrégations lentes

**Problème** :

```javascript
// Agrégation sans optimisation
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: {
      _id: "$userId",
      total: { $sum: "$amount" }
  }},
  { $sort: { total: -1 } },
  { $limit: 10 }
])

// Lent si pas d'index
```

**Solution** :

```javascript
// Index pour le $match
db.orders.createIndex({ status: 1 })

// Réorganiser le pipeline (MongoDB le fait automatiquement, mais...)
db.orders.aggregate([
  { $match: { status: "completed" } },     // Filtrage en premier
  { $sort: { amount: -1 } },               // Tri avant group si possible
  { $group: {
      _id: "$userId",
      total: { $sum: "$amount" }
  }},
  { $limit: 10 }                           // Limite après group
])

// Index composé pour $match + champs du $group
db.orders.createIndex({ status: 1, userId: 1, amount: 1 })
```

---

## Patterns d'optimisation avancés

### Pattern 1 : Covered Query (requête couverte)

**Objectif** : Répondre à la requête uniquement avec l'index, sans lire les documents.

```javascript
// Créer un index avec tous les champs nécessaires
db.users.createIndex({ email: 1, name: 1, age: 1 })

// Requête qui utilise UNIQUEMENT les champs indexés
db.users.find(
  { email: "user@example.com" },
  { _id: 0, email: 1, name: 1, age: 1 }   // Projection sur index
)

// explain() montre :
{
  "stage": "PROJECTION_COVERED",          // ✅ Covered query !
  "totalDocsExamined": 0                  // ✅ 0 documents lus !
}

// Encore plus rapide qu'une requête normale avec index
```

### Pattern 2 : Index partiel pour cas spécifiques

**Objectif** : Indexer seulement les documents pertinents.

```javascript
// Problème : 95% des commandes sont "completed"
// Seules les "pending" sont souvent recherchées

// Solution : Index partiel
db.orders.createIndex(
  { status: 1, createdAt: -1 },
  {
    partialFilterExpression: {
      status: { $in: ["pending", "processing"] }
    }
  }
)

// Avantages :
// - Index 20x plus petit
// - Écritures plus rapides
// - Toujours performant pour les requêtes pertinentes
```

### Pattern 3 : Index composé avec unique

**Objectif** : Garantir l'unicité d'une combinaison de champs.

```javascript
// Un user peut avoir plusieurs emails
// Mais chaque email doit être unique globalement

db.userEmails.createIndex(
  { userId: 1, email: 1 },
  { unique: true }
)

// Permet :
// - userId: 1, email: "a@ex.com" ✅
// - userId: 1, email: "b@ex.com" ✅
// - userId: 2, email: "a@ex.com" ✅

// Interdit :
// - userId: 1, email: "a@ex.com" (doublon) ❌
```

### Pattern 4 : Dénormalisation calculée

**Objectif** : Pré-calculer et stocker des valeurs souvent utilisées.

```javascript
// Au lieu de calculer à chaque requête
db.orders.find({
  $expr: {
    $gt: [{ $multiply: ["$quantity", "$price"] }, 1000]
  }
})
// → Calcul sur chaque document, lent

// Stocker la valeur calculée
{
  quantity: 10,
  price: 150,
  totalAmount: 1500           // ← Calculé à l'insertion
}

// Index sur la valeur calculée
db.orders.createIndex({ totalAmount: 1 })

// Requête rapide
db.orders.find({ totalAmount: { $gt: 1000 } })
```

### Pattern 5 : Index pour lookups fréquents

**Objectif** : Optimiser les jointures (agrégation $lookup).

```javascript
// Agrégation avec $lookup
db.orders.aggregate([
  { $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
  }}
])

// Optimisation : Index sur le foreignField
db.users.createIndex({ _id: 1 })  // Déjà présent
// Mais pour autres cas :
db.products.createIndex({ categoryId: 1 })  // Pour lookups sur category
```

### Pattern 6 : Index sparse pour champs optionnels

**Objectif** : Économiser de l'espace pour champs peu remplis.

```javascript
// Seulement 5% des users ont un phoneNumber
// 95% n'en ont pas (null ou absent)

// Index sparse
db.users.createIndex(
  { phoneNumber: 1 },
  { sparse: true }
)

// Économie : Index 95% plus petit
// Les requêtes sur phoneNumber restent rapides
// Les autres requêtes ne sont pas affectées
```

---

## Techniques d'optimisation applicative

### 1. Caching intelligent

```javascript
// Cache applicatif pour données rarement modifiées
const cache = new Map()

async function getUser(userId) {
  // Vérifier le cache
  if (cache.has(userId)) {
    return cache.get(userId)
  }

  // Sinon, requête DB
  const user = await db.users.findOne({ _id: userId })

  // Mettre en cache (avec TTL)
  cache.set(userId, user)
  setTimeout(() => cache.delete(userId), 60000)  // 1 minute

  return user
}
```

### 2. Batch processing

```javascript
// Au lieu de requêtes individuelles
for (const orderId of orderIds) {
  const order = await db.orders.findOne({ _id: orderId })
  // Traiter...
}

// Batch avec $in
const orders = await db.orders.find({
  _id: { $in: orderIds }
}).toArray()
```

### 3. Projection minimale

```javascript
// Ne charger que les champs nécessaires

// ❌ Charge tout (peut-être 50 champs)
const users = db.users.find({ city: "Paris" })

// ✅ Charge seulement ce qui est nécessaire
const users = db.users.find(
  { city: "Paris" },
  { name: 1, email: 1, _id: 0 }
)

// Économie de bande passante et mémoire
```

### 4. Éviter les grandes transactions

```javascript
// ❌ Transaction énorme
const session = client.startSession()
session.startTransaction()
for (let i = 0; i < 100000; i++) {
  await db.collection.insertOne({ ... }, { session })
}
await session.commitTransaction()
// → Très lent, risque de timeout

// ✅ Batches plus petits
for (let i = 0; i < 100000; i += 1000) {
  const batch = data.slice(i, i + 1000)
  await db.collection.insertMany(batch)
}
```

### 5. Utiliser allowDiskUse pour agrégations

```javascript
// Agrégation qui dépasse la limite mémoire

// ❌ Erreur si > 100 Mo
db.orders.aggregate([
  { $group: { ... } },
  { $sort: { ... } }
])

// ✅ Autorise l'utilisation du disque
db.orders.aggregate(
  [
    { $group: { ... } },
    { $sort: { ... } }
  ],
  { allowDiskUse: true }
)
```

---

## Anti-patterns à éviter

### Anti-pattern 1 : Index sur chaque champ

```javascript
// ❌ MAUVAIS : Trop d'index
db.users.createIndex({ email: 1 })
db.users.createIndex({ username: 1 })
db.users.createIndex({ firstName: 1 })
db.users.createIndex({ lastName: 1 })
db.users.createIndex({ city: 1 })
db.users.createIndex({ country: 1 })
db.users.createIndex({ age: 1 })
db.users.createIndex({ status: 1 })
// → 9 index ! (avec _id)
// → Écritures très lentes
// → Beaucoup d'espace disque

// ✅ BON : Index ciblés sur requêtes réelles
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ city: 1, age: 1 })
db.users.createIndex({ status: 1, lastLoginAt: -1 })
// → 4 index (avec _id)
// → Couvre les requêtes importantes
```

### Anti-pattern 2 : Index redondants

```javascript
// ❌ MAUVAIS : Index redondants
db.products.createIndex({ category: 1 })
db.products.createIndex({ category: 1, price: 1 })
// → L'index composé rend le premier inutile

// ✅ BON : Supprimer le redondant
db.products.dropIndex("category_1")
// Garder seulement : category_1_price_1
// Il peut servir pour :
// - { category: "X" }
// - { category: "X", price: { ... } }
```

### Anti-pattern 3 : Pagination avec skip() profond

```javascript
// ❌ MAUVAIS : skip() avec grande valeur
// Page 1000
db.posts.find()
  .sort({ _id: -1 })
  .skip(50000)
  .limit(50)
// → Parcourt et ignore 50,000 documents !

// ✅ BON : Pagination par curseur
const lastId = previousPage.lastId
db.posts.find({ _id: { $lt: lastId } })
  .sort({ _id: -1 })
  .limit(50)
```

### Anti-pattern 4 : $where pour logique simple

```javascript
// ❌ MAUVAIS : $where avec JavaScript
db.users.find({
  $where: "this.age > 18"
})
// → Ne peut pas utiliser d'index
// → Exécute JS pour chaque document

// ✅ BON : Opérateurs natifs
db.users.find({
  age: { $gt: 18 }
})
```

### Anti-pattern 5 : Requêtes dans des boucles

```javascript
// ❌ MAUVAIS : N+1 queries
const orders = await db.orders.find({ userId: 123 }).toArray()
for (const order of orders) {
  const product = await db.products.findOne({ _id: order.productId })
  // Traiter...
}
// → Si 100 orders : 101 requêtes !

// ✅ BON : Agrégation ou lookup
const orders = await db.orders.aggregate([
  { $match: { userId: 123 } },
  { $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
  }},
  { $unwind: "$product" }
]).toArray()
// → 1 seule requête
```

### Anti-pattern 6 : Tri sans index sur grande collection

```javascript
// ❌ MAUVAIS : Tri en mémoire
db.logs.find()
  .sort({ timestamp: -1 })
  .limit(100)
// → Charge et trie TOUS les documents
// → Peut échouer si > 100 Mo

// ✅ BON : Index sur le champ de tri
db.logs.createIndex({ timestamp: -1 })
```

### Anti-pattern 7 : Faible cardinalité comme seul index

```javascript
// ❌ MAUVAIS : Index sur champ boolean seul
db.users.createIndex({ isActive: 1 })
// → Seulement 2 valeurs (true/false)
// → Peut être un COLLSCAN serait plus rapide

// ✅ BON : Index composé
db.users.createIndex({ isActive: 1, lastLoginAt: -1 })
// → Plus sélectif
// → Supporte aussi le tri
```

---

## Checklist d'optimisation

### ✅ Avant de créer un index

```
□ J'ai identifié une requête lente avec explain()
□ J'ai calculé le ratio actuel (< 80%)
□ J'ai vérifié qu'un index similaire n'existe pas
□ J'ai déterminé les champs nécessaires (ESR)
□ J'ai estimé la taille de l'index
□ J'ai vérifié la fréquence de la requête
□ J'ai comparé avec le coût des écritures
□ J'ai testé en environnement de dev/staging
□ J'ai documenté la raison de l'index
```

### ✅ Après optimisation

```
□ J'ai re-testé avec explain("executionStats")
□ Le ratio est maintenant > 80%
□ Le stage est IXSCAN (pas COLLSCAN)
□ Le temps d'exécution est acceptable (< 100ms)
□ Il n'y a plus de SORT en mémoire si applicable
□ J'ai documenté l'amélioration (avant/après)
□ J'ai vérifié l'impact sur les écritures
□ J'ai surveillé en production pendant quelques jours
```

### ✅ Maintenance régulière

```
□ Analyser $indexStats pour identifier les index inutilisés
□ Vérifier les requêtes lentes (profiler)
□ Supprimer les index redondants
□ Mettre à jour les index selon l'évolution des requêtes
□ Surveiller la taille totale des index
□ Vérifier que les index tiennent en RAM
□ Documenter les changements
```

---

## Outils et méthodes de monitoring

### 1. Profiler MongoDB

```javascript
// Activer le profiler pour requêtes > 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Analyser les requêtes lentes
db.system.profile.find({
  millis: { $gt: 100 }
}).sort({ ts: -1 }).limit(10).pretty()

// Identifier les patterns
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  { $group: {
      _id: "$command.find",
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" },
      maxTime: { $max: "$millis" }
  }},
  { $sort: { count: -1 } }
])
```

### 2. Index statistics

```javascript
// Utilisation des index
db.collection.aggregate([{ $indexStats: {} }])

// Identifier les index non utilisés
db.collection.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 100 } } }
])
```

### 3. Statistiques de collection

```javascript
// Taille des index
db.collection.stats()

// Vérifier la fragmentation
db.collection.stats().wiredTiger.metadata

// Ratio index/données
const stats = db.collection.stats()
const ratio = stats.totalIndexSize / stats.size
console.log(`Ratio index/données : ${(ratio * 100).toFixed(2)}%`)
```

### 4. Script d'analyse automatique

```javascript
// Script pour analyser toutes les collections
db.getCollectionNames().forEach(collName => {
  print(`\n=== ${collName} ===`)
  const stats = db[collName].stats()

  print(`Documents : ${stats.count}`)
  print(`Taille données : ${(stats.size / 1024 / 1024).toFixed(2)} Mo`)
  print(`Taille index : ${(stats.totalIndexSize / 1024 / 1024).toFixed(2)} Mo`)

  // Index stats
  const indexStats = db[collName].aggregate([{ $indexStats: {} }]).toArray()
  print(`\nIndex (usage) :`)
  indexStats.forEach(idx => {
    print(`  - ${idx.name} : ${idx.accesses.ops} utilisations`)
  })
})
```

---

## Processus d'optimisation complet

### Méthodologie en 7 étapes

```
Processus d'optimisation MongoDB
════════════════════════════════

Étape 1 : IDENTIFIER
├─ Activer le profiler
├─ Analyser les requêtes lentes
└─ Prioriser par impact (fréquence × temps)

Étape 2 : MESURER
├─ explain("executionStats") pour chaque requête
├─ Noter les métriques actuelles
└─ Calculer le ratio d'efficacité

Étape 3 : ANALYSER
├─ Identifier le goulot (COLLSCAN, SORT, ratio)
├─ Vérifier les index disponibles
└─ Déterminer la stratégie d'optimisation

Étape 4 : CONCEVOIR
├─ Définir le nouvel index (ESR)
├─ Ou modifier la requête
└─ Estimer l'impact

Étape 5 : TESTER
├─ Appliquer en dev/staging
├─ Mesurer l'amélioration
└─ Vérifier les effets secondaires

Étape 6 : DÉPLOYER
├─ Planifier le déploiement
├─ Créer l'index en production
└─ Surveiller les métriques

Étape 7 : VALIDER
├─ Confirmer l'amélioration
├─ Documenter le changement
└─ Surveiller à long terme
```

### Exemple complet

```javascript
// ÉTAPE 1 : Identifier
// Profiler montre une requête lente

// ÉTAPE 2 : Mesurer
const before = db.orders.find({
  userId: 12345,
  status: "pending"
}).sort({ createdAt: -1 })
  .explain("executionStats")

console.log(`Avant :`)
console.log(`  Stage : ${before.queryPlanner.winningPlan.stage}`)
console.log(`  Temps : ${before.executionStats.executionTimeMillis}ms`)
console.log(`  Docs examinés : ${before.executionStats.totalDocsExamined}`)
console.log(`  Docs retournés : ${before.executionStats.nReturned}`)
console.log(`  Ratio : ${(before.executionStats.nReturned / before.executionStats.totalDocsExamined * 100).toFixed(2)}%`)

// ÉTAPE 3 : Analyser
// → IXSCAN sur userId_1
// → Mais SORT en mémoire
// → Ratio 2% (très inefficace)

// ÉTAPE 4 : Concevoir
// Index composé selon ESR :
// - userId (E)
// - status (E)
// - createdAt (S)

// ÉTAPE 5 : Tester en dev
db.orders.createIndex({
  userId: 1,
  status: 1,
  createdAt: -1
})

const after = db.orders.find({
  userId: 12345,
  status: "pending"
}).sort({ createdAt: -1 })
  .explain("executionStats")

console.log(`\nAprès :`)
console.log(`  Stage : ${after.queryPlanner.winningPlan.inputStage.stage}`)
console.log(`  Index : ${after.queryPlanner.winningPlan.inputStage.indexName}`)
console.log(`  Temps : ${after.executionStats.executionTimeMillis}ms`)
console.log(`  Docs examinés : ${after.executionStats.totalDocsExamined}`)
console.log(`  Docs retournés : ${after.executionStats.nReturned}`)
console.log(`  Ratio : ${(after.executionStats.nReturned / after.executionStats.totalDocsExamined * 100).toFixed(2)}%`)

const improvement = before.executionStats.executionTimeMillis / after.executionStats.executionTimeMillis
console.log(`\nAmélioration : ${improvement.toFixed(1)}x plus rapide`)

// ÉTAPE 6 : Déployer en production
// (selon processus de l'équipe)

// ÉTAPE 7 : Valider et documenter
/*
Index créé : userId_1_status_1_createdAt_-1
Date : 2024-12-01
Requête optimisée : Orders pending par user avec tri
Amélioration : 234ms → 8ms (29x plus rapide)
Ratio : 2% → 100%
Impact : ~10,000 req/jour
*/
```

---

## Concepts clés à retenir

### 🎯 Points essentiels

1. **Mesurer avant d'optimiser**
   - Toujours utiliser explain("executionStats")
   - Comparer avant/après
   - Documenter les résultats

2. **L'index approprié est la clé**
   - Un bon index > toute autre optimisation
   - Suivre la règle ESR
   - Viser un ratio de 100%

3. **Stratégies par scénario**
   - Recherche simple → Index simple
   - Filtres multiples → Index composé
   - Tri → Inclure dans l'index
   - Pagination → Curseur au lieu de skip()

4. **Patterns avancés**
   - Covered queries pour performances maximales
   - Index partiels pour économiser l'espace
   - Dénormalisation calculée si nécessaire

5. **Éviter les anti-patterns**
   - Pas d'index sur chaque champ
   - Pas de requêtes dans des boucles
   - Pas de $where pour logique simple
   - Pas de skip() profond

6. **Processus d'optimisation**
   - Identifier → Mesurer → Analyser
   - Concevoir → Tester → Déployer → Valider

7. **Maintenance continue**
   - Surveiller les requêtes lentes
   - Analyser l'utilisation des index
   - Supprimer les index inutilisés
   - Adapter aux évolutions

---

## Ressources pour aller plus loin

### Commandes essentielles

```javascript
// Analyse de requête
db.collection.find({ ... }).explain("executionStats")

// Index stats
db.collection.aggregate([{ $indexStats: {} }])

// Profiler
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 })

// Stats collection
db.collection.stats()

// Cache de plans
db.collection.getPlanCache().list()
```

### Scripts utiles

```javascript
// Trouver les requêtes les plus lentes
db.system.profile.aggregate([
  { $match: { ns: "mydb.collection" } },
  { $group: {
      _id: "$command.filter",
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" },
      maxTime: { $max: "$millis" }
  }},
  { $sort: { avgTime: -1 } },
  { $limit: 10 }
])

// Trouver les index inutilisés
db.collection.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $eq: 0 } } },
  { $project: { name: 1, key: 1 } }
])
```

---

## Analogie finale

> **Optimiser des requêtes MongoDB, c'est comme optimiser un trajet en voiture :**
>
> **Sans optimisation** = Routes secondaires, embouteillages
> - Vous arrivez, mais c'est long et inefficace
>
> **Avec un index simple** = Autoroute directe
> - Vous arrivez 10x plus vite
>
> **Avec un index composé optimal** = Autoroute + voie rapide + GPS optimisé
> - Vous arrivez 100x plus vite, trajet parfait
>
> **Avec toutes les optimisations** = Formule 1 sur circuit privé
> - Performance maximale, chaque détail compte
>
> **L'optimisation, c'est choisir le meilleur chemin avec les bons outils !** 🏎️

---

**Vous maîtrisez maintenant les stratégies d'optimisation des requêtes MongoDB !** 🚀

---


⏭️ [Index couvrants (Covered Queries)](/05-index-et-optimisation/09-index-couvrants.md)
