🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.4 Options et Modificateurs d'Index

## Introduction

Jusqu'à présent, nous avons exploré les **types d'index** (simple, composé, multiclé, texte, géospatial, haché, wildcard, TTL). Maintenant, nous allons découvrir les **options et modificateurs** qui permettent de personnaliser le comportement de ces index.

Les options d'index sont des **propriétés additionnelles** que vous pouvez ajouter lors de la création d'un index pour modifier son comportement :
- **Unique** : Garantir l'unicité des valeurs
- **Partial** : N'indexer qu'un sous-ensemble de documents
- **Sparse** : Exclure les documents sans le champ indexé
- **Hidden** : Rendre l'index invisible pour le query planner

Ces options transforment un index basique en un outil puissant et flexible, adapté à vos besoins spécifiques.

---

## Pourquoi des Options d'Index ?

### Index Basique vs Index avec Options

**Sans options** : Index standard
```javascript
// Index simple basique
db.users.createIndex({ email: 1 })

// Problèmes potentiels :
// - Aucune garantie d'unicité (doublons possibles)
// - Indexe TOUS les documents (même sans email)
// - Utilise de l'espace pour documents non pertinents
// - Pas de contrôle sur la visibilité
```

**Avec options** : Index personnalisé
```javascript
// Index avec options
db.users.createIndex(
  { email: 1 },
  {
    unique: true,      // Garantit l'unicité
    sparse: true       // N'indexe que si email présent
  }
)

// Avantages :
// ✅ Unicité garantie (pas de doublons)
// ✅ Index plus petit (seulement documents avec email)
// ✅ Adapté au cas d'usage (email optionnel mais unique)
```

### Analogie : Les Options d'une Voiture

Imaginez acheter une voiture :

**Voiture de base** = Index sans options
```
- Moteur standard
- Aucune option
- Fonctionnelle mais basique
```

**Voiture avec options** = Index avec options
```
- GPS intégré (unique: trouve destination unique)
- Radar de recul (partial: aide dans certaines situations)
- Système start-stop (sparse: actif seulement quand nécessaire)
- Mode discret (hidden: invisible mais fonctionnel)
```

Les options personnalisent l'index pour vos besoins spécifiques.

---

## Vue d'Ensemble des Options Principales

### 1. Index Unique (unique)

**Objectif** : Garantir qu'aucune valeur dupliquée n'existe pour le champ indexé

**Syntaxe** :
```javascript
db.collection.createIndex(
  { champ: 1 },
  { unique: true }
)
```

**Cas d'usage** :
- 📧 Emails utilisateurs
- 👤 Usernames
- 🏷️ SKU produits
- 📋 Numéros de commande
- 🔑 Codes de vérification

**Comportement** :
```javascript
db.users.createIndex({ email: 1 }, { unique: true })

// ✅ Premier email
db.users.insertOne({ email: "alice@example.com" })

// ❌ Email dupliqué
db.users.insertOne({ email: "alice@example.com" })
// ERREUR : E11000 duplicate key error
```

**Avantages** :
- ✅ Intégrité des données garantie
- ✅ Détection automatique des doublons
- ✅ Pas besoin de vérification manuelle
- ✅ Combine optimisation + contrainte

**Limitation** :
- ⚠️ Un seul document avec `null` autorisé (sauf avec sparse)

---

### 2. Index Partiel (partial)

**Objectif** : N'indexer qu'un sous-ensemble de documents selon une condition

**Syntaxe** :
```javascript
db.collection.createIndex(
  { champ: 1 },
  {
    partialFilterExpression: {
      condition: valeur
    }
  }
)
```

**Cas d'usage** :
- 🔄 Documents actifs uniquement (status: "active")
- 📦 Produits en stock (stock > 0)
- 📋 Commandes non finalisées
- ✅ Utilisateurs vérifiés
- 💎 Produits premium

**Comportement** :
```javascript
// Index partiel : seulement commandes actives
db.orders.createIndex(
  { customerId: 1 },
  {
    partialFilterExpression: {
      status: { $in: ["pending", "processing"] }
    }
  }
)

// Collection : 10M commandes
// - 9M completed (90%)
// - 1M pending/processing (10%)
// → Index 10x plus petit !
```

**Avantages** :
- ✅ Taille d'index réduite (jusqu'à 90%)
- ✅ Performances améliorées
- ✅ Économie d'espace disque
- ✅ Impact réduit sur les écritures

**Limitation** :
- ⚠️ Requête doit inclure la condition du filtre

---

### 3. Index Sparse (sparse)

**Objectif** : Exclure les documents où le champ est absent ou `null`

**Syntaxe** :
```javascript
db.collection.createIndex(
  { champ: 1 },
  { sparse: true }
)
```

**Cas d'usage** :
- 📱 Numéros de téléphone optionnels
- ✉️ Emails secondaires
- 🌐 IDs connexions sociales (Google, Facebook)
- 🏢 Numéros de TVA (entreprises)
- 📅 Dates optionnelles

**Comportement** :
```javascript
db.users.createIndex({ phoneNumber: 1 }, { sparse: true })

// Documents
{ username: "alice", phoneNumber: "+33612345678" }  // ✅ Indexé
{ username: "bob" }                                  // ❌ Non indexé
{ username: "charlie", phoneNumber: null }           // ❌ Non indexé
```

**Avantages** :
- ✅ Index plus petit (seulement documents avec valeur)
- ✅ Unicité conditionnelle (avec unique)
- ✅ Parfait pour champs optionnels
- ✅ Simple à utiliser

**Limitation** :
- ⚠️ Ne peut pas optimiser recherches de `null`

---

### 4. Index Caché (hidden)

**Objectif** : Rendre l'index invisible pour le query planner sans le supprimer

**Syntaxe** :
```javascript
// Cacher un index existant
db.collection.hideIndex("nomIndex")

// Afficher un index caché
db.collection.unhideIndex("nomIndex")
```

**Cas d'usage** :
- 🧪 Tester suppression d'un index suspect
- 🔄 Migration d'index
- 🐛 Debug de performances
- ✅ Validation prudente en production
- 📊 A/B testing

**Comportement** :
```javascript
db.users.createIndex({ lastLoginDate: 1 })

// Cacher l'index
db.users.hideIndex("lastLoginDate_1")

// Requête n'utilise PAS l'index caché
db.users.find({ lastLoginDate: { $gte: yesterday } })
// → COLLSCAN (comme si l'index n'existait pas)

// Réactiver instantanément si nécessaire
db.users.unhideIndex("lastLoginDate_1")
// → Index utilisable à nouveau
```

**Avantages** :
- ✅ Test sans risque
- ✅ Rollback instantané (millisecondes)
- ✅ Pas de recréation coûteuse
- ✅ Validation en conditions réelles

**Limitation** :
- ⚠️ Index toujours maintenu (impact écritures + espace)

---

## Tableau Comparatif des Options

| Option | Objectif | Impact Taille | Impact Requêtes | Cas d'Usage Principal |
|--------|----------|---------------|-----------------|----------------------|
| **unique** | Garantir unicité | ≈ Index classique | ✅ Optimise + Valide | Identifiants uniques |
| **partial** | Sous-ensemble | 🔽 Très réduite | ✅ Si condition incluse | Filtres fréquents |
| **sparse** | Exclure null/absent | 🔽 Réduite | ✅ Sauf recherche null | Champs optionnels |
| **hidden** | Invisible temporaire | ≈ Index classique | ❌ Pas utilisé | Test/Debug |

### Économies d'Espace par Option

```
Collection : 1 million de documents

Index classique (baseline) :
├─ Taille : 20 MB
└─ Documents indexés : 1M (100%)

Index unique :
├─ Taille : ~20 MB (identique)
└─ Documents indexés : 1M (100%)
└─ Bonus : + Contrainte d'unicité

Index partial (10% des docs) :
├─ Taille : ~2 MB (90% d'économie)
└─ Documents indexés : 100k (10%)

Index sparse (30% ont valeur) :
├─ Taille : ~6 MB (70% d'économie)
└─ Documents indexés : 300k (30%)

Index hidden :
├─ Taille : ~20 MB (aucune économie)
└─ Documents indexés : 1M (mais invisible)
```

---

## Comment Choisir la Bonne Option ?

### Arbre de Décision

```
Quel est votre besoin ?

├─ Garantir l'UNICITÉ des valeurs ?
│  └─→ unique: true
│     "Email, username, SKU doivent être uniques"
│
├─ Champ OPTIONNEL mais unique si présent ?
│  └─→ unique: true, sparse: true
│     "Téléphone optionnel mais pas de doublons"
│
├─ Indexer seulement un SOUS-ENSEMBLE selon condition ?
│  └─→ partialFilterExpression: {...}
│     "Seulement documents actifs, en stock, etc."
│
├─ Champ OPTIONNEL (beaucoup de null/absents) ?
│  └─→ sparse: true
│     "30% ont une valeur, 70% n'ont pas le champ"
│
└─ TESTER avant suppression d'index ?
   └─→ hideIndex()
      "Vérifier impact sans supprimer définitivement"
```

### Questions à Se Poser

#### 1. Le champ doit-il être unique ?

**OUI** → `unique: true`
```javascript
db.users.createIndex({ email: 1 }, { unique: true })
```

**NON** → Continuer...

#### 2. Le champ est-il optionnel ?

**OUI** → `sparse: true` (ou `partial`)
```javascript
db.users.createIndex({ phoneNumber: 1 }, { sparse: true })
```

**NON** → Continuer...

#### 3. Interrogez-vous toujours le même sous-ensemble ?

**OUI** → `partialFilterExpression`
```javascript
db.orders.createIndex(
  { customerId: 1 },
  {
    partialFilterExpression: { status: "active" }
  }
)
```

**NON** → Continuer...

#### 4. Voulez-vous tester l'impact de la suppression ?

**OUI** → `hideIndex()`
```javascript
db.users.hideIndex("suspectIndex")
// Test 24-48h
// Décision : drop ou unhide
```

**NON** → Index classique suffit

---

## Combinaison d'Options

### Options Compatibles

Certaines options peuvent être combinées pour créer des solutions encore plus puissantes :

#### unique + sparse

**Cas d'usage** : Champ optionnel unique
```javascript
// Téléphone optionnel mais unique si présent
db.users.createIndex(
  { phoneNumber: 1 },
  {
    unique: true,
    sparse: true
  }
)

// ✅ Plusieurs utilisateurs sans téléphone
db.users.insertOne({ username: "alice" })
db.users.insertOne({ username: "bob" })
db.users.insertOne({ username: "charlie" })

// ✅ Téléphones uniques
db.users.insertOne({ username: "dave", phoneNumber: "+33612345678" })
db.users.insertOne({ username: "eve", phoneNumber: "+33698765432" })

// ❌ Téléphone dupliqué
db.users.insertOne({ username: "frank", phoneNumber: "+33612345678" })
// ERREUR
```

#### unique + partial

**Cas d'usage** : Unicité conditionnelle
```javascript
// Email unique SEULEMENT pour utilisateurs actifs
db.users.createIndex(
  { email: 1 },
  {
    unique: true,
    partialFilterExpression: { status: "active" }
  }
)

// ✅ Utilisateurs actifs : emails uniques
{ email: "alice@example.com", status: "active" }
{ email: "bob@example.com", status: "active" }

// ✅ Utilisateurs inactifs : emails peuvent être dupliqués
{ email: "old@example.com", status: "inactive" }
{ email: "old@example.com", status: "inactive" }  // OK
```

#### partial + sparse

**Cas d'usage** : Double filtrage
```javascript
// Seulement produits publiés ET avec SKU
db.products.createIndex(
  { sku: 1 },
  {
    unique: true,
    sparse: true,
    partialFilterExpression: { status: "published" }
  }
)
```

### Options Incompatibles

Certaines combinaisons ne sont **pas supportées** :

#### TTL + partial ❌

```javascript
// ❌ Non supporté
db.sessions.createIndex(
  { createdAt: 1 },
  {
    expireAfterSeconds: 3600,
    partialFilterExpression: { status: "active" }
  }
)
// ERREUR : TTL et partial incompatibles
```

#### TTL + sparse ⚠️ (problématique)

```javascript
// ⚠️ Techniquement possible mais problématique
db.sessions.createIndex(
  { expiresAt: 1 },
  {
    expireAfterSeconds: 0,
    sparse: true
  }
)

// Problème : Documents sans expiresAt ne sont PAS dans l'index
// → Ne seront JAMAIS supprimés par TTL !
```

---

## Exemples Pratiques par Scénario

### Scénario 1 : Plateforme E-commerce

```javascript
// Collection users
{
  _id: ObjectId("..."),
  username: "alice",
  email: "alice@example.com",      // Obligatoire, unique
  phoneNumber: "+33612345678",     // Optionnel, unique si présent
  status: "active",                // active, inactive, banned
  googleId: "google-123456"        // Optionnel, unique si présent
}

// Index 1 : Email (unique, obligatoire)
db.users.createIndex({ email: 1 }, { unique: true })

// Index 2 : Username (unique, obligatoire)
db.users.createIndex({ username: 1 }, { unique: true })

// Index 3 : Téléphone (unique, optionnel)
db.users.createIndex({ phoneNumber: 1 }, { unique: true, sparse: true })

// Index 4 : Google ID (unique, optionnel)
db.users.createIndex({ googleId: 1 }, { unique: true, sparse: true })

// Index 5 : Recherche dans utilisateurs actifs
db.users.createIndex(
  { email: 1 },
  {
    partialFilterExpression: { status: "active" }
  }
)
```

### Scénario 2 : Système de Commandes

```javascript
// Collection orders
{
  _id: ObjectId("..."),
  orderNumber: "ORD-2024-001234",  // Unique
  customerId: 12345,
  status: "pending",               // pending, processing, shipped, completed
  total: 299.99
}

// Index 1 : Numéro de commande unique
db.orders.createIndex({ orderNumber: 1 }, { unique: true })

// Index 2 : Commandes actives (pending, processing, shipped)
db.orders.createIndex(
  { customerId: 1, createdAt: -1 },
  {
    partialFilterExpression: {
      status: { $in: ["pending", "processing", "shipped"] }
    }
  }
)
```

### Scénario 3 : Application avec Connexions Sociales

```javascript
// Collection users
{
  _id: ObjectId("..."),
  username: "alice",
  email: "alice@example.com",
  googleId: "google-123",       // Optionnel
  facebookId: "fb-456",         // Optionnel
  githubId: "github-789"        // Optionnel
}

// Index unique + sparse pour chaque provider
db.users.createIndex({ googleId: 1 }, { unique: true, sparse: true })
db.users.createIndex({ facebookId: 1 }, { unique: true, sparse: true })
db.users.createIndex({ githubId: 1 }, { unique: true, sparse: true })
```

---

## Stratégie de Sélection d'Options

### Processus de Décision

```javascript
// Étape 1 : Analyser le champ
let field = {
  name: "phoneNumber",
  mandatory: false,        // Optionnel
  mustBeUnique: true,      // Doit être unique si présent
  percentageWithValue: 30  // 30% des documents ont le champ
}

// Étape 2 : Déterminer les options nécessaires
let options = {}

// Unicité requise ?
if (field.mustBeUnique) {
  options.unique = true
  console.log("Option : unique")
}

// Champ optionnel ?
if (!field.mandatory) {
  options.sparse = true
  console.log("Option : sparse")
}

// Filtrer sur sous-ensemble fréquent ?
let filterCondition = null  // À définir selon cas d'usage
if (filterCondition) {
  options.partialFilterExpression = filterCondition
  console.log("Option : partial")
}

// Étape 3 : Créer l'index
db.users.createIndex({ [field.name]: 1 }, options)
console.log("Index créé avec options :", options)
```

### Checklist de Validation

Avant de créer un index avec options, vérifiez :

**Pour unique** :
- [ ] Le champ doit vraiment être unique dans toute la collection ?
- [ ] S'il est optionnel, ai-je ajouté `sparse: true` ?
- [ ] Les doublons existants ont été nettoyés ?
- [ ] La validation est en place côté application ?

**Pour partial** :
- [ ] Au moins 20% des documents sont exclus (économies significatives) ?
- [ ] Mes requêtes incluent toujours la condition du filtre ?
- [ ] Le filtre est stable (pas de changements fréquents) ?
- [ ] La condition est supportée (pas de regex, géospatial, etc.) ?

**Pour sparse** :
- [ ] Au moins 30% des documents n'ont pas le champ ?
- [ ] Je ne recherche jamais les documents avec `null` ?
- [ ] Je ne trie pas tous les documents sur ce champ ?
- [ ] Simple à utiliser et maintenir ?

**Pour hidden** :
- [ ] Je veux tester avant suppression définitive ?
- [ ] J'ai un plan de surveillance (24-48h) ?
- [ ] J'ai un plan de rollback (unhideIndex) ?
- [ ] Durée de test définie avec décision à prendre ?

---

## Outils et Commandes

### Créer un Index avec Options

```javascript
// Syntaxe générale
db.collection.createIndex(
  { champ: 1 },           // Clés d'index
  {
    unique: true,         // Option 1
    sparse: true,         // Option 2
    name: "custom_name",  // Nom personnalisé
    background: true      // Déprécié depuis 4.2
  }
)
```

### Vérifier les Options d'un Index

```javascript
// Lister tous les index avec leurs options
db.collection.getIndexes()

// Résultat exemple :
// {
//   "v": 2,
//   "key": { "email": 1 },
//   "name": "email_1",
//   "unique": true,    // ← Option unique
//   "sparse": true     // ← Option sparse
// }
```

### Filtrer par Option

```javascript
// Index uniques
db.collection.getIndexes().filter(idx => idx.unique === true)

// Index sparse
db.collection.getIndexes().filter(idx => idx.sparse === true)

// Index partiels
db.collection.getIndexes().filter(idx =>
  idx.partialFilterExpression !== undefined
)

// Index cachés
db.collection.getIndexes().filter(idx => idx.hidden === true)
```

### Modifier un Index Existant

Certaines options peuvent être modifiées après création :

```javascript
// Cacher un index
db.collection.hideIndex("indexName")

// Afficher un index
db.collection.unhideIndex("indexName")

// Modifier TTL (expireAfterSeconds)
db.runCommand({
  collMod: "collection",
  index: {
    keyPattern: { createdAt: 1 },
    expireAfterSeconds: 7200  // Nouvelle valeur
  }
})
```

⚠️ **Note** : On ne peut PAS modifier `unique`, `sparse`, ou `partialFilterExpression` après création. Il faut supprimer et recréer l'index.

---

## Bonnes Pratiques Générales

### ✅ À Faire

1. **Choisir l'option la plus simple qui répond au besoin**
   ```javascript
   // ✅ Besoin simple = option simple
   // Champ optionnel → sparse (pas partial avec exists)
   db.users.createIndex({ phoneNumber: 1 }, { sparse: true })
   ```

2. **Documenter les choix d'options**
   ```javascript
   // Commentaire expliquant pourquoi chaque option
   // Index unique + sparse : phoneNumber optionnel mais pas de doublons
   // Économie : ~70% (70% utilisateurs sans téléphone)
   db.users.createIndex({ phoneNumber: 1 }, { unique: true, sparse: true })
   ```

3. **Tester en environnement de dev/staging**
   ```javascript
   // Créer l'index en staging
   // Mesurer impact sur performances
   // Valider que les requêtes utilisent bien l'index
   ```

4. **Valider les données avant index unique**
   ```javascript
   // Vérifier les doublons AVANT de créer l'index unique
   db.users.aggregate([
     { $group: { _id: "$email", count: { $sum: 1 } } },
     { $match: { count: { $gt: 1 } } }
   ])
   ```

5. **Combiner judicieusement les options**
   ```javascript
   // unique + sparse pour champs optionnels uniques
   db.users.createIndex({ googleId: 1 }, { unique: true, sparse: true })
   ```

6. **Nommer clairement les index avec options**
   ```javascript
   // Nom descriptif incluant les options
   db.users.createIndex(
     { email: 1 },
     {
       unique: true,
       name: "email_unique_active_users",
       partialFilterExpression: { status: "active" }
     }
   )
   ```

### ❌ À Éviter

1. **Ne pas sur-utiliser les options**
   ```javascript
   // ❌ Options inutiles ajoutent complexité
   // Si le champ est obligatoire, pas besoin de sparse
   db.users.createIndex({ email: 1 }, { sparse: true })  // Inutile
   ```

2. **Ne pas oublier sparse avec unique pour optionnels**
   ```javascript
   // ❌ Unique sans sparse sur champ optionnel
   db.users.createIndex({ phoneNumber: 1 }, { unique: true })
   // Problème : 1 seul document sans phoneNumber autorisé !

   // ✅ Ajouter sparse
   db.users.createIndex({ phoneNumber: 1 }, { unique: true, sparse: true })
   ```

3. **Ne pas créer partial si > 50% des documents**
   ```javascript
   // ❌ Si 80% des documents correspondent au filtre
   // → Économies limitées, complexité inutile

   // ✅ Utiliser partial seulement si < 20-30% correspondent
   ```

4. **Ne pas laisser un index hidden indéfiniment**
   ```javascript
   // ❌ Index caché depuis 6 mois
   // Consomme espace et impact écritures pour rien

   // ✅ Décider dans les 48-72h : drop ou unhide
   ```

5. **Ne pas combiner options incompatibles**
   ```javascript
   // ❌ TTL + partial
   db.sessions.createIndex(
     { createdAt: 1 },
     {
       expireAfterSeconds: 3600,
       partialFilterExpression: { status: "active" }
     }
   )
   // ERREUR
   ```

---

## Progression dans l'Apprentissage

Cette section introduit les quatre options principales d'index et leur combinaison. Pour maîtriser chacune en profondeur, consultez les sections détaillées suivantes :

### 📖 Sections Détaillées

1. **[5.4.1 Index Unique](04.1-index-unique.md)**
   - Garantir l'unicité des valeurs
   - Gestion des erreurs E11000
   - Unicité conditionnelle
   - Champs obligatoires vs optionnels
   - Cas d'usage : emails, SKU, codes

2. **[5.4.2 Index Partiel (Partial)](04.2-index-partiel.md)**
   - partialFilterExpression en détail
   - Opérateurs supportés
   - Économies d'espace (jusqu'à 90%)
   - Requêtes doivent inclure filtre
   - Cas d'usage : documents actifs, en stock

3. **[5.4.3 Index Sparse](04.3-index-sparse.md)**
   - Comportement avec null/undefined/absent
   - Différence avec partial
   - Combinaison unique + sparse
   - Impact sur requêtes
   - Cas d'usage : champs optionnels

4. **[5.4.4 Index Caché (Hidden)](04.4-index-cache.md)**
   - hideIndex() / unhideIndex()
   - Test avant suppression
   - Rollback instantané
   - Stratégie de test (24-48h)
   - Cas d'usage : validation prudente

5. **[5.4.5 Combinaison d'Options](04.5-combinaison-options.md)**
   - unique + sparse (⭐⭐⭐⭐⭐)
   - unique + partial
   - sparse + partial
   - Triple combinaisons
   - Matrice de compatibilité

### 🎯 Parcours Recommandé

**Pour les débutants** :
1. **Index Unique** (5.4.1) - Le plus courant et simple
2. **Index Sparse** (5.4.3) - Souvent utilisé avec unique
3. **Index Partiel** (5.4.2) - Plus avancé, grande flexibilité
4. **Index Caché** (5.4.4) - Pour gestion en production
5. **Combinaison** (5.4.5) - Synthèse et optimisation

**Pour les utilisateurs intermédiaires** :
1. Identifier vos besoins spécifiques
2. Approfondir les options correspondantes
3. Expérimenter les combinaisons
4. Tester en staging avant production

**Pour les experts** :
- Utilisez cette section comme référence rapide
- Consultez les sections détaillées pour cas avancés
- Optimisez les combinaisons d'options
- Passez aux sections suivantes sur la gestion

---

## Conclusion

Les **options d'index** (unique, partial, sparse, hidden) sont des outils puissants qui permettent de personnaliser le comportement des index MongoDB. Elles transforment un index basique en une solution optimisée pour vos besoins spécifiques, que ce soit pour garantir l'intégrité des données, réduire la taille des index, ou tester en toute sécurité.

### Points Clés à Retenir

- 🔑 Options = Personnalisation du comportement d'index
- 🔑 4 options principales : unique, partial, sparse, hidden
- 🔑 Peuvent être combinées (avec restrictions)
- 🔑 Chaque option résout un problème spécifique
- 🔑 Choisir l'option la plus simple qui répond au besoin
- 🔑 Documenter les choix d'options
- 🔑 Tester avant déploiement en production
- 🔑 Valider que les requêtes utilisent bien l'index

### Tableau Récapitulatif Rapide

| Si vous voulez... | Utilisez... |
|-------------------|-------------|
| Empêcher doublons | `unique: true` |
| Champ optionnel unique | `unique: true, sparse: true` |
| Indexer sous-ensemble | `partialFilterExpression: {...}` |
| Exclure null/absents | `sparse: true` |
| Tester avant supprimer | `hideIndex()` |

### Prochaines Étapes

Après avoir exploré les options d'index, vous serez prêt pour :
- **[5.5 Création et Suppression d'Index](./05-creation-suppression-index.md)** : Gestion complète
- **[5.6 Analyse avec explain()](./06-analyse-explain.md)** : Diagnostic approfondi
- **[5.7 Monitoring des Index](./07-monitoring-index.md)** : Surveillance en production
- **[5.8 Stratégies d'Optimisation](./08-strategies-optimisation.md)** : Techniques avancées

---

**📚 Ressources Complémentaires**
- [Documentation officielle MongoDB - Index Properties](https://docs.mongodb.com/manual/core/index-properties/)
- [Unique Indexes](https://docs.mongodb.com/manual/core/index-unique/)
- [Partial Indexes](https://docs.mongodb.com/manual/core/index-partial/)
- [Sparse Indexes](https://docs.mongodb.com/manual/core/index-sparse/)
- [Hidden Indexes](https://docs.mongodb.com/manual/core/index-hidden/)

⏭️ [Index unique](/05-index-et-optimisation/04.1-index-unique.md)
