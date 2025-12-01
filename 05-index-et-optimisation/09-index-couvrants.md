🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.9 Index couvrants (Covered Queries)

## Introduction

Les **index couvrants** (covered queries) représentent le **niveau ultime d'optimisation** des requêtes MongoDB. C'est une technique qui permet à MongoDB de répondre à une requête **uniquement avec l'index**, sans jamais avoir besoin de lire les documents complets.

Imaginez pouvoir obtenir vos résultats sans jamais toucher aux données réelles, en consultant uniquement une table des matières enrichie. C'est exactement ce que font les covered queries : elles offrent des performances exceptionnelles en évitant complètement l'étape coûteuse de récupération des documents.

Dans ce chapitre, nous allons découvrir :
- 🎯 Ce qu'est une **covered query**
- ⚡ Pourquoi c'est **si rapide**
- 🔑 Les **conditions** pour en créer une
- 🛠️ Comment **concevoir** des index couvrants
- 📊 Comment **vérifier** avec explain()
- ⚠️ Les **limitations** à connaître

Maîtriser les covered queries vous permettra d'atteindre des performances maximales pour vos requêtes les plus critiques.

---

## Qu'est-ce qu'une Covered Query ?

### Définition

Une **covered query** (requête couverte) est une requête dont **tous les champs nécessaires** sont contenus dans l'index lui-même. MongoDB peut donc répondre à la requête en lisant uniquement l'index, sans avoir besoin de récupérer les documents complets.

### Analogie

```
Analogie avec un annuaire téléphonique
══════════════════════════════════════

Annuaire standard :
─────────────────
Index : Nom → Page
"Dupont" → page 147

Vous cherchez le téléphone de "Dupont Pierre" :
1. Consultez l'index → page 147
2. Allez à la page 147 dans le livre
3. Trouvez "Dupont Pierre"
4. Lisez son numéro : 01-23-45-67-89

Annuaire enrichi (covered query) :
──────────────────────────────────
Index enrichi : Nom → Téléphone
"Dupont Pierre" → 01-23-45-67-89

Vous cherchez le téléphone de "Dupont Pierre" :
1. Consultez l'index → 01-23-45-67-89
2. FIN ! (pas besoin d'ouvrir le livre)

Vous avez toutes les infos dans l'index !
```

### Visualisation

```
Requête NORMALE avec index
══════════════════════════

1. Recherche dans INDEX
   ┌─────────────────────┐
   │ email → _id         │
   │ ────────────────    │
   │ alice@.. → doc1     │ ← Trouve la référence
   │ bob@...  → doc2     │
   └─────────────────────┘
        ↓
2. FETCH du document complet
   ┌─────────────────────────────────┐
   │ { _id: doc1,                    │
   │   email: "alice@...",           │
   │   name: "Alice",                │
   │   age: 30,                      │
   │   address: { ... },             │ ← Lit tout le document
   │   ... 50 autres champs ...      │
   │ }                               │
   └─────────────────────────────────┘
        ↓
3. Retourne les champs demandés


Requête COUVERTE (Covered Query)
═════════════════════════════════

1. Recherche dans INDEX enrichi
   ┌────────────────────────────┐
   │ email → email + name       │
   │ ─────────────────────      │
   │ alice@.. → alice@.., Alice │ ← Trouve TOUT
   │ bob@...  → bob@..., Bob    │
   └────────────────────────────┘
        ↓
2. FIN ! (pas de FETCH)
   Toutes les infos sont dans l'index
        ↓
3. Retourne directement
```

---

## Pourquoi les Covered Queries sont-elles si rapides ?

### Comparaison des performances

```
Performance relative
════════════════════

COLLSCAN (scan complet)
████████████████████████████████████████████ 5000ms
└─ Lit tous les documents un par un

IXSCAN + FETCH (index normal)
████ 50ms
└─ Lit l'index + récupère les documents

IXSCAN COVERED (covered query)
█ 5ms
└─ Lit uniquement l'index

Amélioration : 1000x vs COLLSCAN, 10x vs index normal !
```

### Raisons de la rapidité

#### 1. Pas de lecture disque des documents

```
Index normal :
├─ Lecture index (en RAM)      : 1ms
├─ Lecture documents (disque)  : 45ms  ← COÛTEUX
└─ Total                       : 46ms

Covered query :
├─ Lecture index (en RAM)      : 5ms
├─ Lecture documents           : 0ms   ← ÉVITÉ !
└─ Total                       : 5ms

Gain : 9x plus rapide
```

#### 2. Moins de données transférées

```
Document complet : 5 Ko
├─ _id: ObjectId(...)
├─ email: "user@example.com"
├─ name: "John Doe"
├─ age: 30
├─ address: { ... }
├─ preferences: { ... }
├─ history: [ ... ]
└─ ... 50 autres champs

Index uniquement : 100 octets
├─ email: "user@example.com"
└─ name: "John Doe"

Réduction : 98% de données en moins !
```

#### 3. Les index sont en mémoire (RAM)

```
Hiérarchie mémoire (du plus rapide au plus lent) :
═══════════════════════════════════════════════════

L1 Cache CPU    : 1 ns      │ [Rarement utilisé par MongoDB]
L2 Cache CPU    : 10 ns     │
L3 Cache CPU    : 50 ns     │
RAM             : 100 ns    │ ← Index ici
SSD             : 100 µs    │ ← Documents ici
HDD             : 10 ms     │

Index (RAM)      : ~100 ns
Documents (SSD)  : ~100,000 ns
Différence       : 1000x plus lent
```

#### 4. Totalité de la requête en une seule opération

```
Index normal :
1. Cherche dans l'index          → 1ms
2. Pour chaque résultat :
   - Lecture document 1          → 0.5ms
   - Lecture document 2          → 0.5ms
   - ...
   - Lecture document 100        → 0.5ms
Total : 1ms + (100 × 0.5ms) = 51ms

Covered query :
1. Cherche dans l'index          → 5ms
Total : 5ms

Pas de va-et-vient index ↔ documents !
```

---

## Conditions pour une Covered Query

Pour qu'une requête soit "couverte", **5 conditions** doivent être remplies :

### Condition 1 : Tous les champs retournés sont dans l'index

```javascript
// Index créé
db.users.createIndex({ email: 1, name: 1, age: 1 })

// ✅ COUVERT : Tous les champs sont dans l'index
db.users.find(
  { email: "alice@example.com" },
  { email: 1, name: 1, age: 1, _id: 0 }  // ← Projection
)

// ❌ NON COUVERT : "address" n'est pas dans l'index
db.users.find(
  { email: "alice@example.com" },
  { email: 1, name: 1, address: 1, _id: 0 }
)
```

### Condition 2 : Le filtre utilise l'index

```javascript
// Index créé
db.users.createIndex({ city: 1, name: 1 })

// ✅ COUVERT : Le filtre utilise "city" (dans l'index)
db.users.find(
  { city: "Paris" },
  { city: 1, name: 1, _id: 0 }
)

// ❌ NON COUVERT : Le filtre utilise "country" (pas dans l'index)
db.users.find(
  { country: "France" },
  { city: 1, name: 1, _id: 0 }
)
```

### Condition 3 : Le champ _id est exclu (ou dans l'index)

```javascript
// Index créé
db.users.createIndex({ email: 1, name: 1 })

// ✅ COUVERT : _id exclu
db.users.find(
  { email: "alice@example.com" },
  { email: 1, name: 1, _id: 0 }  // ← _id: 0
)

// ❌ NON COUVERT : _id inclus (par défaut)
db.users.find(
  { email: "alice@example.com" },
  { email: 1, name: 1 }  // ← _id inclus par défaut
)

// ✅ COUVERT : _id dans l'index
db.users.createIndex({ _id: 1, email: 1, name: 1 })
db.users.find(
  { email: "alice@example.com" },
  { _id: 1, email: 1, name: 1 }
)
```

### Condition 4 : Aucun champ indexé n'est un tableau

```javascript
// Index avec tableau
db.articles.createIndex({ tags: 1, title: 1 })

// ❌ NON COUVERT : "tags" est un champ tableau (multikey index)
db.articles.find(
  { tags: "mongodb" },
  { tags: 1, title: 1, _id: 0 }
)

// Les index multikey ne peuvent pas être couvrants
// Raison : Complexité de gestion des tableaux
```

### Condition 5 : La requête ne contient pas certains opérateurs

Certains opérateurs empêchent les covered queries :

```javascript
// ❌ NON COUVERT : $text (recherche full-text)
db.articles.find(
  { $text: { $search: "mongodb" } },
  { title: 1, _id: 0 }
)

// ❌ NON COUVERT : $geoWithin, $near (géospatial)
db.places.find(
  { location: { $near: [2.35, 48.86] } },
  { name: 1, _id: 0 }
)

// ❌ NON COUVERT : $exists avec false
db.users.find(
  { deletedAt: { $exists: false } },
  { name: 1, _id: 0 }
)
```

---

## Comment créer une Covered Query

### Étape 1 : Identifier la requête à optimiser

```javascript
// Requête fréquente dans l'application
db.users.find(
  { email: "user@example.com" },
  { name: 1, email: 1, age: 1 }
)
```

### Étape 2 : Analyser les champs nécessaires

```
Champs utilisés :
═════════════════

Filtre :     email
Projection : name, email, age, _id (par défaut)

Pour couvrir la requête, l'index doit contenir :
→ email, name, age

Et la projection doit exclure _id OU l'inclure dans l'index
```

### Étape 3 : Créer l'index approprié

```javascript
// Créer l'index avec tous les champs nécessaires
db.users.createIndex({ email: 1, name: 1, age: 1 })
```

### Étape 4 : Modifier la requête pour exclure _id

```javascript
// Requête modifiée (exclure _id)
db.users.find(
  { email: "user@example.com" },
  { name: 1, email: 1, age: 1, _id: 0 }  // ← _id: 0
)
```

### Étape 5 : Vérifier avec explain()

```javascript
db.users.find(
  { email: "user@example.com" },
  { name: 1, email: 1, age: 1, _id: 0 }
).explain("executionStats")
```

**Résultat attendu** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "PROJECTION_COVERED",    // ✅ COVERED !
      "transformBy": {
        "name": 1,
        "email": 1,
        "age": 1,
        "_id": 0
      },
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "email_1_name_1_age_1"
      }
    }
  },
  "executionStats": {
    "nReturned": 1,
    "totalDocsExamined": 0,             // ✅ 0 documents !
    "totalKeysExamined": 1,
    "executionTimeMillis": 2
  }
}
```

**Indicateurs d'une covered query** :
- ✅ `stage: "PROJECTION_COVERED"` ou `stage: "IXSCAN"` sans `FETCH`
- ✅ `totalDocsExamined: 0` (aucun document lu)
- ✅ Temps d'exécution très faible

---

## Exemples concrets

### Exemple 1 : Recherche utilisateur par email

#### Sans covered query

```javascript
// Index simple
db.users.createIndex({ email: 1 })

// Requête
db.users.find({ email: "alice@example.com" })
  .explain("executionStats")
```

**Résultat** :

```json
{
  "executionStats": {
    "nReturned": 1,
    "totalDocsExamined": 1,        // 1 document lu
    "totalKeysExamined": 1,
    "executionTimeMillis": 15
  }
}
```

#### Avec covered query

```javascript
// Index enrichi
db.users.createIndex({ email: 1, name: 1 })

// Requête avec projection
db.users.find(
  { email: "alice@example.com" },
  { email: 1, name: 1, _id: 0 }
).explain("executionStats")
```

**Résultat** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "PROJECTION_COVERED"  // ✅ Covered !
    }
  },
  "executionStats": {
    "nReturned": 1,
    "totalDocsExamined": 0,         // ✅ 0 documents lus !
    "totalKeysExamined": 1,
    "executionTimeMillis": 2        // 7.5x plus rapide
  }
}
```

### Exemple 2 : Liste des produits par catégorie

#### Configuration

```javascript
// Collection de produits
{
  _id: ObjectId("..."),
  sku: "PROD-001",
  name: "Laptop Pro",
  category: "Electronics",
  price: 999,
  stock: 50,
  description: "...",  // Beaucoup de texte
  reviews: [ ... ],     // Gros tableau
  specifications: { ... }  // Objet complexe
}
```

#### Index et requête couverts

```javascript
// Index couvrant
db.products.createIndex({
  category: 1,
  name: 1,
  price: 1
})

// Requête couverte
db.products.find(
  { category: "Electronics" },
  { category: 1, name: 1, price: 1, _id: 0 }
).sort({ price: 1 })
  .limit(20)
```

**Avantages** :
- Ne charge pas les descriptions volumineuses
- Ne charge pas les reviews (tableaux)
- Ne charge pas les specifications
- **Extrêmement rapide** : 3-5ms au lieu de 80-100ms

### Exemple 3 : Statistiques par plage de dates

#### Scénario

```javascript
// Beaucoup de logs par jour
// Requête fréquente : compter par statut et date

// Index couvrant
db.logs.createIndex({
  date: 1,
  status: 1
})

// Agrégation couverte
db.logs.aggregate([
  { $match: {
      date: {
        $gte: ISODate("2024-12-01"),
        $lt: ISODate("2024-12-02")
      }
  }},
  { $project: {
      date: 1,
      status: 1,
      _id: 0
  }},
  { $group: {
      _id: "$status",
      count: { $sum: 1 }
  }}
])
```

**Performance** :
- Sans covered query : 2500ms (lit 1M de documents complets)
- Avec covered query : 80ms (lit seulement l'index)
- **Amélioration : 31x plus rapide**

### Exemple 4 : Vérification d'existence

```javascript
// Vérifier si un email existe déjà

// Index
db.users.createIndex({ email: 1 })

// Requête couverte
const exists = db.users.findOne(
  { email: "test@example.com" },
  { _id: 1 }  // Seulement _id (qui est dans tous les index)
)

// Ou encore plus optimal (sans _id)
db.users.createIndex({ email: 1 })
const exists = db.users.findOne(
  { email: "test@example.com" },
  { email: 1, _id: 0 }
)

// Usage
if (exists) {
  console.log("Email déjà utilisé")
} else {
  console.log("Email disponible")
}
```

---

## Stratégies pour maximiser les Covered Queries

### Stratégie 1 : Ajouter des champs fréquemment consultés

```javascript
// Requête fréquente
db.orders.find({ userId: 123 })

// Au lieu de retourner tout le document, créer un index couvrant
db.orders.createIndex({
  userId: 1,
  status: 1,
  amount: 1,
  createdAt: 1
})

// Requête couverte
db.orders.find(
  { userId: 123 },
  { userId: 1, status: 1, amount: 1, createdAt: 1, _id: 0 }
)
```

### Stratégie 2 : Index dédiés aux dashboards

```javascript
// Dashboard affiche : nom, email, lastLogin, status

// Index spécifique pour le dashboard
db.users.createIndex({
  status: 1,        // Filtre fréquent
  lastLogin: -1,    // Tri fréquent
  name: 1,          // Affiché
  email: 1          // Affiché
})

// Requête dashboard (couverte)
db.users.find(
  { status: "active" },
  { name: 1, email: 1, lastLogin: 1, status: 1, _id: 0 }
).sort({ lastLogin: -1 })
  .limit(100)
```

### Stratégie 3 : Séparation lecture/écriture

```javascript
// Pour des données fréquemment lues mais rarement mises à jour

// Collection "profiles" avec champs souvent lus
{
  userId: 123,
  displayName: "John Doe",
  avatar: "url",
  level: 5,
  badges: 12
}

// Index couvrant pour affichage
db.profiles.createIndex({
  userId: 1,
  displayName: 1,
  avatar: 1,
  level: 1
})

// Requête ultra-rapide
db.profiles.find(
  { userId: 123 },
  { displayName: 1, avatar: 1, level: 1, _id: 0 }
)
```

### Stratégie 4 : Index pour APIs

```javascript
// API endpoint : GET /users/search?email=...

// Index couvrant pour l'API
db.users.createIndex({
  email: 1,
  name: 1,
  username: 1,
  createdAt: 1
})

// Réponse API (rapide, couverte)
app.get('/users/search', async (req, res) => {
  const user = await db.users.findOne(
    { email: req.query.email },
    { email: 1, name: 1, username: 1, createdAt: 1, _id: 0 }
  )
  res.json(user)
})
```

---

## Limitations et contraintes

### Limitation 1 : Index multikey (tableaux)

```javascript
// Index avec champ tableau
db.articles.createIndex({ tags: 1, title: 1 })

// ❌ Ne peut JAMAIS être couvert
db.articles.find(
  { tags: "mongodb" },
  { tags: 1, title: 1, _id: 0 }
)

// Raison : Complexité de gestion des entrées multiples
// Un document avec 5 tags = 5 entrées dans l'index
```

**Solution** : Dénormaliser si critique

```javascript
// Ajouter un champ non-tableau pour les cas fréquents
{
  tags: ["mongodb", "database", "nosql"],
  primaryTag: "mongodb"  // ← Champ simple
}

// Index sans multikey
db.articles.createIndex({ primaryTag: 1, title: 1 })

// Requête couverte
db.articles.find(
  { primaryTag: "mongodb" },
  { primaryTag: 1, title: 1, _id: 0 }
)
```

### Limitation 2 : Taille de l'index

```javascript
// Index avec beaucoup de champs
db.products.createIndex({
  category: 1,
  brand: 1,
  name: 1,
  description: 1,  // ← Champ long !
  price: 1,
  stock: 1
})

// Problème : L'index devient énorme
// → Peut ne plus tenir en RAM
// → Performances dégradées
```

**Solution** : Équilibre entre couverture et taille

```javascript
// Index plus petit
db.products.createIndex({
  category: 1,
  brand: 1,
  name: 1,     // Champs courts seulement
  price: 1,
  stock: 1
  // description exclu volontairement
})
```

### Limitation 3 : Coût des écritures

```javascript
// Index couvrant avec 10 champs
db.collection.createIndex({
  field1: 1, field2: 1, field3: 1, field4: 1, field5: 1,
  field6: 1, field7: 1, field8: 1, field9: 1, field10: 1
})

// Problème : Chaque insertion/mise à jour doit mettre à jour
// toutes les valeurs dans l'index
// → Écritures plus lentes
```

**Compromis** :
```
Lectures très fréquentes (10000/s)     → Index couvrant justifié ✅
Écritures fréquentes (1000/s)          → Évaluer le compromis ⚠️
Écritures > Lectures                    → Peut-être éviter ❌
```

### Limitation 4 : Champs géospatiaux

```javascript
// Index géospatial
db.places.createIndex({ location: "2dsphere", name: 1 })

// ❌ Ne peut pas être couvert
db.places.find(
  { location: { $near: { ... } } },
  { name: 1, _id: 0 }
)

// Les index géospatiaux ne supportent pas les covered queries
```

### Limitation 5 : Collections shardées

```javascript
// Sur cluster shardé, les covered queries sont possibles
// MAIS uniquement si :
// - Le filtre inclut la shard key
// - OU la requête va sur un seul shard

// ✅ COUVERT si filtre sur shard key
db.orders.find(
  { userId: 123 },  // userId = shard key
  { userId: 1, status: 1, _id: 0 }
)

// ⚠️ PEUT NE PAS ÊTRE COUVERT si broadcast
db.orders.find(
  { status: "pending" },  // Pas la shard key
  { status: 1, amount: 1, _id: 0 }
)
// → Query envoyée à tous les shards
// → Peut ne pas être optimisée en covered query
```

---

## Vérifier une Covered Query avec explain()

### Indicateurs dans explain()

```javascript
const result = db.users.find(
  { email: "user@example.com" },
  { email: 1, name: 1, _id: 0 }
).explain("executionStats")
```

### Signes d'une covered query

#### 1. Stage PROJECTION_COVERED

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "PROJECTION_COVERED",  // ✅ Indicateur #1
      "transformBy": { ... },
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "email_1_name_1"
      }
    }
  }
}
```

#### 2. Pas de stage FETCH

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "IXSCAN",            // ✅ IXSCAN sans FETCH
      "indexName": "email_1_name_1"
      // Pas de "inputStage" avec FETCH
    }
  }
}
```

#### 3. totalDocsExamined = 0

```json
{
  "executionStats": {
    "nReturned": 1,
    "totalDocsExamined": 0,         // ✅ Indicateur #3
    "totalKeysExamined": 1,
    "executionTimeMillis": 2
  }
}
```

### Comparaison visuelle

```javascript
// Test avec et sans covered query

// SANS covered query (avec _id)
const without = db.users.find(
  { email: "user@example.com" },
  { email: 1, name: 1 }  // _id inclus par défaut
).explain("executionStats")

console.log("SANS covered query :")
console.log(`  Stage : ${without.queryPlanner.winningPlan.stage}`)
console.log(`  Docs examinés : ${without.executionStats.totalDocsExamined}`)
console.log(`  Temps : ${without.executionStats.executionTimeMillis}ms`)

// AVEC covered query (sans _id)
const with_covered = db.users.find(
  { email: "user@example.com" },
  { email: 1, name: 1, _id: 0 }  // _id exclu
).explain("executionStats")

console.log("\nAVEC covered query :")
console.log(`  Stage : ${with_covered.queryPlanner.winningPlan.stage}`)
console.log(`  Docs examinés : ${with_covered.executionStats.totalDocsExamined}`)
console.log(`  Temps : ${with_covered.executionStats.executionTimeMillis}ms`)
```

**Résultat attendu** :

```
SANS covered query :
  Stage : FETCH
  Docs examinés : 1
  Temps : 15ms

AVEC covered query :
  Stage : PROJECTION_COVERED
  Docs examinés : 0
  Temps : 2ms

Amélioration : 7.5x plus rapide
```

---

## Cas d'usage idéaux

### 1. APIs haute performance

```javascript
// Endpoint critique : /api/users/:email

// Index couvrant
db.users.createIndex({
  email: 1,
  name: 1,
  username: 1,
  status: 1
})

// Handler ultra-rapide
app.get('/api/users/:email', async (req, res) => {
  const user = await db.users.findOne(
    { email: req.params.email },
    { email: 1, name: 1, username: 1, status: 1, _id: 0 }
  )
  res.json(user)  // Réponse < 5ms
})
```

### 2. Recherche autocomplete

```javascript
// Autocomplete sur noms d'utilisateurs

// Index couvrant
db.users.createIndex({
  username: 1,
  displayName: 1,
  avatar: 1
})

// Recherche rapide
db.users.find(
  { username: { $regex: /^joh/i } },
  { username: 1, displayName: 1, avatar: 1, _id: 0 }
).limit(10)
```

### 3. Listes de sélection

```javascript
// Dropdown : Sélection de catégories

// Index dédié
db.categories.createIndex({
  active: 1,
  name: 1,
  slug: 1
})

// Liste pour UI (couverte)
db.categories.find(
  { active: true },
  { name: 1, slug: 1, _id: 0 }
).sort({ name: 1 })
```

### 4. Vérifications d'existence

```javascript
// Vérifier si username déjà pris

// Index
db.users.createIndex({ username: 1 })

// Vérification ultra-rapide
async function isUsernameTaken(username) {
  const exists = await db.users.findOne(
    { username: username },
    { username: 1, _id: 0 }
  )
  return exists !== null
}
```

### 5. Compteurs et statistiques

```javascript
// Compter commandes par statut

// Index pour stats
db.orders.createIndex({
  status: 1,
  createdAt: 1
})

// Agrégation couverte
db.orders.aggregate([
  { $match: {
      createdAt: { $gte: startDate, $lte: endDate }
  }},
  { $project: {
      status: 1,
      createdAt: 1,
      _id: 0
  }},
  { $group: {
      _id: "$status",
      count: { $sum: 1 }
  }}
])
```

---

## Bonnes pratiques

### ✅ À faire

```
1. Identifier les requêtes les plus fréquentes
   └─ Utiliser le profiler MongoDB

2. Analyser les champs réellement nécessaires
   └─ Souvent, seulement 3-5 champs sur 50

3. Créer des index couvrants ciblés
   └─ Pas besoin de couvrir toutes les requêtes

4. Toujours exclure _id dans les projections
   └─ Sauf si vraiment nécessaire

5. Vérifier avec explain("executionStats")
   └─ Confirmer totalDocsExamined = 0

6. Documenter l'intention
   └─ Expliquer pourquoi l'index est structuré ainsi

7. Surveiller la taille des index
   └─ Équilibre entre couverture et espace

8. Privilégier pour les lectures fréquentes
   └─ Ratio lectures/écritures > 10:1
```

### ❌ À éviter

```
1. Inclure des champs longs dans l'index
   └─ Descriptions, textes longs, etc.

2. Créer des index couvrants sur champs tableau
   └─ Impossible avec multikey

3. Forcer la covered query partout
   └─ Compromis avec coût d'écriture

4. Oublier d'exclure _id
   └─ Erreur la plus fréquente !

5. Index couvrant trop large (10+ champs)
   └─ Coût en espace et écritures

6. Ne pas mesurer le gain réel
   └─ Toujours valider avec explain()
```

---

## Checklist : Créer une Covered Query

### ✅ Checklist complète

```
□ J'ai identifié une requête fréquente et critique
□ J'ai listé tous les champs nécessaires (filtre + projection)
□ J'ai créé un index contenant tous ces champs
□ J'ai modifié la requête pour exclure _id (ou l'inclure dans l'index)
□ Aucun champ dans l'index n'est un tableau
□ La requête n'utilise pas d'opérateurs incompatibles ($text, $geo...)
□ J'ai testé avec explain("executionStats")
□ Le stage est "PROJECTION_COVERED" ou "IXSCAN" sans FETCH
□ totalDocsExamined = 0
□ J'ai mesuré l'amélioration de performance
□ J'ai vérifié l'impact sur les écritures
□ J'ai documenté l'index et sa raison d'être
```

---

## Concepts clés à retenir

### 🎯 Points essentiels

1. **Covered query** = Requête répondue uniquement avec l'index
   - Pas de lecture des documents
   - Performance maximale

2. **5 conditions obligatoires** :
   - Tous les champs retournés dans l'index
   - Le filtre utilise l'index
   - _id exclu (ou dans l'index)
   - Pas de champ tableau
   - Pas d'opérateurs incompatibles

3. **Performance** :
   - 10x plus rapide qu'index normal
   - 1000x plus rapide que COLLSCAN
   - totalDocsExamined = 0

4. **Création** :
   - Index avec tous les champs nécessaires
   - Projection excluant _id
   - Vérification avec explain()

5. **Cas d'usage idéaux** :
   - APIs haute performance
   - Autocomplete
   - Vérifications d'existence
   - Listes de sélection
   - Statistiques

6. **Compromis** :
   - Espace disque (index plus gros)
   - Écritures plus lentes
   - À réserver aux requêtes critiques

---

## Ressources et commandes utiles

### Commandes essentielles

```javascript
// Vérifier si covered
db.collection.find({ ... }, { ..., _id: 0 })
  .explain("executionStats")

// Chercher PROJECTION_COVERED ou totalDocsExamined: 0

// Taille de l'index
db.collection.stats().indexSizes

// Utilisation de l'index
db.collection.aggregate([{ $indexStats: {} }])
```

### Script de validation

```javascript
function isCoveredQuery(explainResult) {
  const plan = explainResult.queryPlanner.winningPlan
  const stats = explainResult.executionStats

  // Méthode 1 : Vérifier le stage
  const hasCoveredStage = plan.stage === "PROJECTION_COVERED" ||
    (plan.stage === "IXSCAN" && !plan.inputStage)

  // Méthode 2 : Vérifier totalDocsExamined
  const noDocsExamined = stats.totalDocsExamined === 0

  if (hasCoveredStage && noDocsExamined) {
    console.log("✅ COVERED QUERY !")
    console.log(`   Index utilisé : ${plan.inputStage?.indexName || plan.indexName}`)
    console.log(`   Temps : ${stats.executionTimeMillis}ms`)
    return true
  } else {
    console.log("❌ NOT COVERED")
    console.log(`   Stage : ${plan.stage}`)
    console.log(`   Docs examinés : ${stats.totalDocsExamined}`)
    return false
  }
}

// Usage
const result = db.users.find(
  { email: "test@example.com" },
  { email: 1, name: 1, _id: 0 }
).explain("executionStats")

isCoveredQuery(result)
```

---

## Analogie finale

> **Les covered queries, c'est comme un menu fast-food :**
>
> **Restaurant normal** (requête normale avec index) :
> - Vous commandez un burger
> - Le serveur note votre commande (index)
> - Va en cuisine chercher votre burger (FETCH du document)
> - Vous sert votre burger
> - Temps total : 5 minutes
>
> **Menu pré-emballé** (covered query) :
> - Vous commandez un menu
> - Le serveur prend un menu déjà prêt sur l'étagère (index complet)
> - Vous donne directement (pas de cuisine)
> - Temps total : 10 secondes
>
> **Les covered queries = Tout ce dont vous avez besoin, déjà prêt dans l'index !** 🍔

---

**Vous maîtrisez maintenant les covered queries, le niveau ultime d'optimisation MongoDB !** 🚀

---


⏭️ [Gestion des index en production](/05-index-et-optimisation/10-gestion-index-production.md)
