🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.2 - CRUD Rapide

## Table des matières

1. [Create - Insertion](#create---insertion)
2. [Read - Lecture](#read---lecture)
3. [Update - Mise à jour](#update---mise-%C3%A0-jour)
4. [Delete - Suppression](#delete---suppression)
5. [Replace - Remplacement](#replace---remplacement)
6. [Bulk Operations](#bulk-operations)
7. [Options communes](#options-communes)
8. [Opérateurs essentiels](#op%C3%A9rateurs-essentiels)

---

## Create - Insertion

### insertOne()

Insère un seul document dans une collection.

```javascript
db.<collection>.insertOne(<document>)
```

**Exemples :**

```javascript
// Insertion simple
db.users.insertOne({
  name: "Alice",
  email: "alice@example.com",
  age: 30
})

// Résultat :
{
  acknowledged: true,
  insertedId: ObjectId("507f1f77bcf86cd799439011")
}
```

```javascript
// Avec _id personnalisé
db.users.insertOne({
  _id: "user001",
  name: "Bob",
  email: "bob@example.com"
})
```

⚠️ **Erreur** : Si le `_id` existe déjà, erreur de duplication.

---

### insertMany()

Insère plusieurs documents en une seule opération.

```javascript
db.<collection>.insertMany([<document1>, <document2>, ...])
```

**Exemples :**

```javascript
// Insertion multiple
db.users.insertMany([
  { name: "Alice", email: "alice@example.com", age: 30 },
  { name: "Bob", email: "bob@example.com", age: 25 },
  { name: "Charlie", email: "charlie@example.com", age: 35 }
])

// Résultat :
{
  acknowledged: true,
  insertedIds: {
    '0': ObjectId("..."),
    '1': ObjectId("..."),
    '2': ObjectId("...")
  }
}
```

---

### Options d'insertion

```javascript
// Ordre non garanti (plus rapide)
db.users.insertMany([...], { ordered: false })

// Write Concern personnalisé
db.users.insertOne(
  { name: "Alice" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)
```

💡 **ordered: false** : Continue l'insertion même en cas d'erreur sur certains documents.

---

## Read - Lecture

### find()

Recherche tous les documents correspondant à un filtre.

```javascript
db.<collection>.find(<filtre>, <projection>)
```

**Exemples :**

```javascript
// Tous les documents
db.users.find()

// Avec filtre
db.users.find({ age: 30 })

// Avec projection (sélection de champs)
db.users.find(
  { age: { $gte: 25 } },
  { name: 1, email: 1, _id: 0 }
)

// Résultat :
[
  { name: "Alice", email: "alice@example.com" },
  { name: "Bob", email: "bob@example.com" }
]
```

---

### findOne()

Retourne un seul document (le premier correspondant).

```javascript
db.<collection>.findOne(<filtre>, <projection>)
```

**Exemples :**

```javascript
// Premier document de la collection
db.users.findOne()

// Premier utilisateur actif
db.users.findOne({ active: true })

// Avec projection
db.users.findOne(
  { email: "alice@example.com" },
  { name: 1, age: 1 }
)

// Résultat :
{
  _id: ObjectId("..."),
  name: "Alice",
  age: 30
}
```

💡 **Note** : Retourne `null` si aucun document trouvé.

---

### Méthodes de curseur

#### sort()

Trie les résultats.

```javascript
// Tri croissant (1) ou décroissant (-1)
db.users.find().sort({ age: 1 })         // Âge croissant
db.users.find().sort({ age: -1 })        // Âge décroissant
db.users.find().sort({ name: 1, age: -1 }) // Multi-critères
```

---

#### limit()

Limite le nombre de résultats.

```javascript
// 10 premiers documents
db.users.find().limit(10)

// Les 5 plus âgés
db.users.find().sort({ age: -1 }).limit(5)
```

---

#### skip()

Saute les N premiers résultats (pagination).

```javascript
// Sauter les 10 premiers
db.users.find().skip(10)

// Pagination : page 2, 10 résultats par page
db.users.find().skip(10).limit(10)

// Page N (N commence à 1)
const page = 3;
const perPage = 10;
db.users.find()
  .skip((page - 1) * perPage)
  .limit(perPage)
```

⚠️ **Performance** : `skip()` sur de grandes valeurs est lent. Préférer la pagination par curseur pour de gros volumes.

---

#### count()

Compte les documents.

```javascript
// Compte total (déprécié, utiliser countDocuments)
db.users.find({ active: true }).count()

// Méthode recommandée
db.users.countDocuments({ active: true })

// Estimation rapide (sans filtre)
db.users.estimatedDocumentCount()
```

---

### Filtres avancés

#### Opérateurs de comparaison

```javascript
// Égalité
db.users.find({ age: 30 })

// Différent de
db.users.find({ age: { $ne: 30 } })

// Supérieur à
db.users.find({ age: { $gt: 25 } })

// Supérieur ou égal à
db.users.find({ age: { $gte: 25 } })

// Inférieur à
db.users.find({ age: { $lt: 40 } })

// Inférieur ou égal à
db.users.find({ age: { $lte: 40 } })

// Dans une liste
db.users.find({ status: { $in: ["active", "pending"] } })

// Pas dans une liste
db.users.find({ status: { $nin: ["deleted", "banned"] } })
```

---

#### Opérateurs logiques

```javascript
// ET (implicite par défaut)
db.users.find({ age: 30, active: true })

// ET (explicite)
db.users.find({
  $and: [
    { age: { $gte: 25 } },
    { age: { $lte: 35 } }
  ]
})

// OU
db.users.find({
  $or: [
    { age: { $lt: 25 } },
    { age: { $gt: 35 } }
  ]
})

// NON
db.users.find({ age: { $not: { $eq: 30 } } })

// NI (NOR)
db.users.find({
  $nor: [
    { age: { $lt: 25 } },
    { status: "banned" }
  ]
})
```

---

#### Opérateurs d'éléments

```javascript
// Champ existe
db.users.find({ phone: { $exists: true } })

// Champ n'existe pas
db.users.find({ phone: { $exists: false } })

// Type de données
db.users.find({ age: { $type: "number" } })
db.users.find({ age: { $type: 16 } })  // 16 = int32
```

---

#### Opérateurs de tableaux

```javascript
// Contient une valeur
db.users.find({ tags: "mongodb" })

// Contient toutes les valeurs
db.users.find({ tags: { $all: ["mongodb", "database"] } })

// Taille du tableau
db.users.find({ tags: { $size: 3 } })

// Élément du tableau correspond
db.users.find({
  scores: { $elemMatch: { $gte: 80, $lte: 90 } }
})
```

---

#### Requêtes sur documents imbriqués

```javascript
// Notation pointée
db.users.find({ "address.city": "Paris" })

// Document imbriqué complet
db.users.find({
  address: { street: "5 rue de la Paix", city: "Paris" }
})
```

⚠️ **Attention** : La deuxième méthode requiert une correspondance exacte (ordre et champs).

---

#### Expressions régulières

```javascript
// Commence par
db.users.find({ name: /^A/ })
db.users.find({ name: { $regex: "^A" } })

// Contient (insensible à la casse)
db.users.find({ email: { $regex: "gmail", $options: "i" } })

// Se termine par
db.users.find({ email: /\.com$/ })
```

---

## Update - Mise à jour

### updateOne()

Met à jour le premier document correspondant.

```javascript
db.<collection>.updateOne(<filtre>, <update>, <options>)
```

**Exemples :**

```javascript
// Modifier un champ
db.users.updateOne(
  { email: "alice@example.com" },
  { $set: { age: 31 } }
)

// Résultat :
{
  acknowledged: true,
  matchedCount: 1,
  modifiedCount: 1
}
```

```javascript
// Incrémenter une valeur
db.users.updateOne(
  { email: "alice@example.com" },
  { $inc: { loginCount: 1 } }
)

// Ajouter un champ s'il n'existe pas
db.users.updateOne(
  { email: "alice@example.com" },
  { $setOnInsert: { createdAt: new Date() } },
  { upsert: true }
)
```

---

### updateMany()

Met à jour tous les documents correspondants.

```javascript
db.<collection>.updateMany(<filtre>, <update>, <options>)
```

**Exemples :**

```javascript
// Mettre à jour tous les utilisateurs inactifs
db.users.updateMany(
  { lastLogin: { $lt: new Date("2023-01-01") } },
  { $set: { active: false } }
)

// Résultat :
{
  acknowledged: true,
  matchedCount: 150,
  modifiedCount: 150
}
```

```javascript
// Ajouter un champ à tous les documents
db.users.updateMany(
  {},
  { $set: { version: 2 } }
)
```

---

### Opérateurs de mise à jour

#### $set

Définit la valeur d'un champ.

```javascript
db.users.updateOne(
  { _id: "user001" },
  { $set: { age: 31, city: "Paris" } }
)
```

---

#### $unset

Supprime un champ.

```javascript
db.users.updateOne(
  { _id: "user001" },
  { $unset: { tempField: "" } }
)
```

---

#### $inc

Incrémente/décrémente une valeur numérique.

```javascript
// Incrémenter
db.users.updateOne(
  { _id: "user001" },
  { $inc: { age: 1 } }
)

// Décrémenter
db.products.updateOne(
  { sku: "ABC123" },
  { $inc: { stock: -5 } }
)
```

---

#### $mul

Multiplie une valeur.

```javascript
db.products.updateOne(
  { sku: "ABC123" },
  { $mul: { price: 1.1 } }  // +10%
)
```

---

#### $rename

Renomme un champ.

```javascript
db.users.updateOne(
  { _id: "user001" },
  { $rename: { "name": "fullName" } }
)
```

---

#### $min / $max

Met à jour uniquement si la nouvelle valeur est plus petite/grande.

```javascript
// Ne met à jour que si 25 < valeur actuelle
db.users.updateOne(
  { _id: "user001" },
  { $min: { age: 25 } }
)

// Ne met à jour que si 40 > valeur actuelle
db.users.updateOne(
  { _id: "user001" },
  { $max: { age: 40 } }
)
```

---

#### $currentDate

Définit la date/heure actuelle.

```javascript
db.users.updateOne(
  { _id: "user001" },
  { $currentDate: {
      lastModified: true,
      lastLogin: { $type: "timestamp" }
  } }
)
```

---

### Opérateurs de tableaux

#### $push

Ajoute un élément à un tableau.

```javascript
// Ajouter un élément
db.users.updateOne(
  { _id: "user001" },
  { $push: { tags: "mongodb" } }
)

// Ajouter plusieurs éléments
db.users.updateOne(
  { _id: "user001" },
  { $push: { tags: { $each: ["database", "nosql"] } } }
)

// Ajouter avec tri et limitation
db.users.updateOne(
  { _id: "user001" },
  {
    $push: {
      scores: {
        $each: [85, 92],
        $sort: -1,
        $slice: 5  // Garde seulement les 5 meilleurs
      }
    }
  }
)
```

---

#### $pull

Supprime des éléments d'un tableau.

```javascript
// Supprimer une valeur
db.users.updateOne(
  { _id: "user001" },
  { $pull: { tags: "obsolete" } }
)

// Supprimer selon condition
db.users.updateOne(
  { _id: "user001" },
  { $pull: { scores: { $lt: 50 } } }
)
```

---

#### $pop

Supprime le premier ou dernier élément.

```javascript
// Supprimer le dernier
db.users.updateOne(
  { _id: "user001" },
  { $pop: { tags: 1 } }
)

// Supprimer le premier
db.users.updateOne(
  { _id: "user001" },
  { $pop: { tags: -1 } }
)
```

---

#### $addToSet

Ajoute un élément uniquement s'il n'existe pas (ensemble).

```javascript
// Ajouter si non présent
db.users.updateOne(
  { _id: "user001" },
  { $addToSet: { tags: "mongodb" } }
)

// Ajouter plusieurs éléments uniques
db.users.updateOne(
  { _id: "user001" },
  { $addToSet: { tags: { $each: ["mongodb", "database"] } } }
)
```

---

### Options d'update

#### upsert

Insère si le document n'existe pas.

```javascript
db.users.updateOne(
  { email: "new@example.com" },
  {
    $set: { name: "New User" },
    $setOnInsert: { createdAt: new Date() }
  },
  { upsert: true }
)
```

💡 **$setOnInsert** : Champs définis uniquement lors de l'insertion (pas lors de la mise à jour).

---

#### arrayFilters

Filtre pour mettre à jour des éléments spécifiques d'un tableau.

```javascript
// Mettre à jour les scores > 80
db.students.updateOne(
  { _id: "student001" },
  { $set: { "grades.$[elem].grade": "A" } },
  { arrayFilters: [{ "elem.score": { $gte: 80 } }] }
)
```

---

## Delete - Suppression

### deleteOne()

Supprime le premier document correspondant.

```javascript
db.<collection>.deleteOne(<filtre>)
```

**Exemples :**

```javascript
// Supprimer par _id
db.users.deleteOne({ _id: "user001" })

// Résultat :
{
  acknowledged: true,
  deletedCount: 1
}

// Supprimer par critère
db.users.deleteOne({ email: "user@example.com" })
```

---

### deleteMany()

Supprime tous les documents correspondants.

```javascript
db.<collection>.deleteMany(<filtre>)
```

**Exemples :**

```javascript
// Supprimer tous les utilisateurs inactifs
db.users.deleteMany({ active: false })

// Résultat :
{
  acknowledged: true,
  deletedCount: 42
}

// Supprimer tous les documents (⚠️ DANGER)
db.users.deleteMany({})
```

⚠️ **ATTENTION** : `deleteMany({})` supprime TOUS les documents de la collection !

---

## Replace - Remplacement

### replaceOne()

Remplace complètement un document (sauf `_id`).

```javascript
db.<collection>.replaceOne(<filtre>, <nouveauDocument>, <options>)
```

**Exemples :**

```javascript
// Remplacer complètement un document
db.users.replaceOne(
  { _id: "user001" },
  {
    name: "Alice Updated",
    email: "alice.new@example.com",
    age: 31,
    active: true
  }
)

// Résultat :
{
  acknowledged: true,
  matchedCount: 1,
  modifiedCount: 1
}
```

⚠️ **Différence avec update** :
- `replaceOne()` : Remplace tout le document (pas d'opérateurs $set)
- `updateOne()` : Modifie des champs spécifiques avec opérateurs

---

## Bulk Operations

### bulkWrite()

Exécute plusieurs opérations en une seule requête.

```javascript
db.<collection>.bulkWrite([
  <operation1>,
  <operation2>,
  ...
], <options>)
```

**Exemples :**

```javascript
db.users.bulkWrite([
  // Insertion
  {
    insertOne: {
      document: { name: "Alice", email: "alice@example.com" }
    }
  },

  // Mise à jour
  {
    updateOne: {
      filter: { email: "bob@example.com" },
      update: { $set: { age: 31 } }
    }
  },

  // Suppression
  {
    deleteOne: {
      filter: { email: "old@example.com" }
    }
  },

  // Remplacement
  {
    replaceOne: {
      filter: { _id: "user001" },
      replacement: { name: "Charlie", email: "charlie@example.com" }
    }
  }
])

// Résultat :
{
  acknowledged: true,
  insertedCount: 1,
  insertedIds: { '0': ObjectId("...") },
  matchedCount: 2,
  modifiedCount: 2,
  deletedCount: 1,
  upsertedCount: 0
}
```

---

### Options de bulkWrite

```javascript
// Mode non ordonné (continue en cas d'erreur)
db.users.bulkWrite([...], { ordered: false })

// Write Concern personnalisé
db.users.bulkWrite([...], {
  writeConcern: { w: "majority" }
})
```

💡 **ordered: false** : Plus rapide, exécution parallèle, continue même si erreurs.

---

## Options communes

### Write Concern

Niveau de confirmation d'écriture.

```javascript
// Confirmation de la majorité
db.users.insertOne(
  { name: "Alice" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)

// Options :
// w: 1 (défaut) - Primary uniquement
// w: "majority" - Majorité des nœuds
// w: <nombre> - Nombre spécifique de nœuds
// j: true - Écriture dans le journal
// wtimeout: <ms> - Timeout
```

---

### Collation

Options de tri et comparaison linguistiques.

```javascript
// Recherche insensible à la casse
db.users.find(
  { name: "alice" },
  { collation: { locale: "en", strength: 2 } }
)

// Update avec collation
db.users.updateOne(
  { name: "alice" },
  { $set: { verified: true } },
  { collation: { locale: "en", strength: 2 } }
)
```

💡 **strength: 2** : Ignore la casse et les accents.

---

### Hint

Force l'utilisation d'un index spécifique.

```javascript
db.users.find({ age: 30 }).hint({ age: 1 })

db.users.updateMany(
  { status: "active" },
  { $set: { notified: true } },
  { hint: { status: 1 } }
)
```

💡 **Usage** : Optimisation manuelle, debugging de requêtes.

---

## Opérateurs essentiels

### Tableau récapitulatif

#### Comparaison

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$eq` | Égal à | `{ age: { $eq: 30 } }` |
| `$ne` | Différent de | `{ age: { $ne: 30 } }` |
| `$gt` | Supérieur à | `{ age: { $gt: 25 } }` |
| `$gte` | Supérieur ou égal | `{ age: { $gte: 25 } }` |
| `$lt` | Inférieur à | `{ age: { $lt: 40 } }` |
| `$lte` | Inférieur ou égal | `{ age: { $lte: 40 } }` |
| `$in` | Dans liste | `{ status: { $in: ["active"] } }` |
| `$nin` | Pas dans liste | `{ status: { $nin: ["banned"] } }` |

#### Logiques

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$and` | ET logique | `{ $and: [{...}, {...}] }` |
| `$or` | OU logique | `{ $or: [{...}, {...}] }` |
| `$not` | NON logique | `{ age: { $not: { $eq: 30 } } }` |
| `$nor` | NI logique | `{ $nor: [{...}, {...}] }` |

#### Éléments

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$exists` | Champ existe | `{ phone: { $exists: true } }` |
| `$type` | Type BSON | `{ age: { $type: "number" } }` |

#### Mise à jour

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$set` | Définir valeur | `{ $set: { age: 31 } }` |
| `$unset` | Supprimer champ | `{ $unset: { temp: "" } }` |
| `$inc` | Incrémenter | `{ $inc: { age: 1 } }` |
| `$mul` | Multiplier | `{ $mul: { price: 1.1 } }` |
| `$rename` | Renommer | `{ $rename: { "old": "new" } }` |
| `$min` | Minimum | `{ $min: { age: 25 } }` |
| `$max` | Maximum | `{ $max: { age: 40 } }` |
| `$currentDate` | Date actuelle | `{ $currentDate: { updated: true } }` |

#### Tableaux

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$push` | Ajouter élément | `{ $push: { tags: "new" } }` |
| `$pull` | Retirer élément | `{ $pull: { tags: "old" } }` |
| `$pop` | Retirer premier/dernier | `{ $pop: { tags: 1 } }` |
| `$addToSet` | Ajouter si unique | `{ $addToSet: { tags: "new" } }` |
| `$all` | Contient tous | `{ tags: { $all: ["a", "b"] } }` |
| `$size` | Taille tableau | `{ tags: { $size: 3 } }` |
| `$elemMatch` | Élément correspond | `{ scores: { $elemMatch: {...} } }` |

---

## Workflows complets

### Workflow CRUD standard

```javascript
// 1. CREATE - Créer des utilisateurs
db.users.insertMany([
  { name: "Alice", email: "alice@example.com", age: 30, active: true },
  { name: "Bob", email: "bob@example.com", age: 25, active: true },
  { name: "Charlie", email: "charlie@example.com", age: 35, active: false }
])

// 2. READ - Lire les utilisateurs actifs
db.users.find({ active: true }).sort({ age: 1 })

// 3. UPDATE - Mettre à jour l'âge d'Alice
db.users.updateOne(
  { email: "alice@example.com" },
  { $inc: { age: 1 } }
)

// 4. DELETE - Supprimer les utilisateurs inactifs
db.users.deleteMany({ active: false })

// 5. VERIFY - Vérifier le résultat
db.users.countDocuments()
db.users.find()
```

---

### Migration de données

```javascript
// Ajouter un nouveau champ à tous les documents
db.users.updateMany(
  { version: { $exists: false } },
  { $set: { version: 2, migratedAt: new Date() } }
)

// Renommer un champ
db.users.updateMany(
  {},
  { $rename: { "oldField": "newField" } }
)

// Nettoyer les champs obsolètes
db.users.updateMany(
  {},
  { $unset: { deprecatedField: "" } }
)
```

---

### Gestion des compteurs

```javascript
// Incrémenter un compteur de vues
db.articles.updateOne(
  { _id: "article123" },
  {
    $inc: { views: 1 },
    $currentDate: { lastViewed: true }
  }
)

// Incrémenter plusieurs compteurs
db.stats.updateOne(
  { _id: "global" },
  {
    $inc: {
      totalViews: 1,
      uniqueVisitors: 1,
      todayViews: 1
    }
  },
  { upsert: true }
)
```

---

### Gestion de tags

```javascript
// Ajouter un tag unique
db.articles.updateOne(
  { _id: "article123" },
  { $addToSet: { tags: "mongodb" } }
)

// Ajouter plusieurs tags uniques
db.articles.updateOne(
  { _id: "article123" },
  { $addToSet: { tags: { $each: ["database", "nosql", "tutorial"] } } }
)

// Supprimer un tag
db.articles.updateOne(
  { _id: "article123" },
  { $pull: { tags: "obsolete" } }
)

// Supprimer plusieurs tags
db.articles.updateOne(
  { _id: "article123" },
  { $pull: { tags: { $in: ["old", "deprecated"] } } }
)
```

---

### Pagination efficace

```javascript
// Pagination classique (page 2, 10 par page)
const page = 2;
const limit = 10;
db.users.find()
  .sort({ _id: 1 })
  .skip((page - 1) * limit)
  .limit(limit)

// Pagination par curseur (meilleure performance)
const lastId = ObjectId("..."); // Dernier _id de la page précédente
db.users.find({ _id: { $gt: lastId } })
  .sort({ _id: 1 })
  .limit(10)
```

💡 **Pagination par curseur** : Plus efficace sur grandes collections.

---

### Upsert pattern

```javascript
// Mettre à jour ou créer si n'existe pas
db.userStats.updateOne(
  { userId: "user123", date: "2024-01-15" },
  {
    $inc: { pageViews: 1, clicks: 1 },
    $setOnInsert: {
      userId: "user123",
      date: "2024-01-15",
      createdAt: new Date()
    }
  },
  { upsert: true }
)
```

---

### Gestion des versions de documents

```javascript
// Mise à jour avec vérification de version (optimistic locking)
const currentVersion = 5;

const result = db.documents.updateOne(
  {
    _id: "doc123",
    version: currentVersion  // Condition : version actuelle
  },
  {
    $set: { content: "New content", updatedAt: new Date() },
    $inc: { version: 1 }  // Incrémenter la version
  }
)

if (result.modifiedCount === 0) {
  print("Conflit de version ! Document modifié par un autre processus.");
} else {
  print("Mise à jour réussie.");
}
```

---

## Bonnes pratiques

### ✅ Faire

```javascript
// Utiliser des projections
db.users.find({ active: true }, { name: 1, email: 1, _id: 0 })

// Limiter les résultats
db.logs.find().sort({ date: -1 }).limit(100)

// Utiliser updateOne/deleteOne quand un seul document attendu
db.users.updateOne({ _id: "user123" }, { $set: { active: false } })

// Valider l'existence avant suppression
if (db.users.countDocuments({ _id: "user123" }) > 0) {
  db.users.deleteOne({ _id: "user123" })
}

// Utiliser bulkWrite pour plusieurs opérations
db.users.bulkWrite([...])
```

---

### ❌ Éviter

```javascript
// Éviter find() sans limite sur grosses collections
db.hugeLogs.find()  // Peut charger des millions de documents

// Éviter updateMany sans filtre (sauf intentionnel)
db.users.updateMany({}, { $set: { migrated: true } })  // Tous les documents !

// Éviter skip() avec grandes valeurs
db.users.find().skip(1000000).limit(10)  // Très lent

// Éviter les regex complexes sans index
db.users.find({ email: /.*complex.*pattern.*/i })

// Éviter les tableaux de mise à jour non bornés
db.docs.updateOne({ _id: "doc1" }, { $push: { history: {...} } })
// Sans limite, le tableau peut croître indéfiniment
```

---

## Aide-mémoire rapide

### Syntaxe générale

```javascript
// CREATE
db.collection.insertOne({...})
db.collection.insertMany([{...}, {...}])

// READ
db.collection.find({...})
db.collection.findOne({...})
db.collection.find().sort({...}).limit(N).skip(N)

// UPDATE
db.collection.updateOne({filter}, {$set: {...}})
db.collection.updateMany({filter}, {$set: {...}})

// DELETE
db.collection.deleteOne({filter})
db.collection.deleteMany({filter})

// REPLACE
db.collection.replaceOne({filter}, {...})

// BULK
db.collection.bulkWrite([{...}, {...}])
```

---

**💡 Conseil** : Testez toujours vos requêtes de mise à jour et suppression avec `find()` avant d'exécuter !

```javascript
// 1. Tester avec find()
db.users.find({ lastLogin: { $lt: new Date("2020-01-01") } })

// 2. Si OK, exécuter la suppression
db.users.deleteMany({ lastLogin: { $lt: new Date("2020-01-01") } })
```

⏭️ [Administration (rs.status(), sh.status(), etc.)](/annexes/commandes-mongosh/03-administration.md)
