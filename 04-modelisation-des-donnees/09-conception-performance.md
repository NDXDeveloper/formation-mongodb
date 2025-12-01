🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.9 Conception pour la performance

## Introduction

La **performance** est un critère essentiel dans la modélisation de données MongoDB. Un schéma mal conçu peut entraîner des temps de réponse catastrophiques, même avec le matériel le plus puissant. À l'inverse, un schéma bien pensé peut offrir d'excellentes performances avec des ressources modestes.

Dans ce chapitre, nous allons explorer comment concevoir vos schémas MongoDB en gardant la performance au cœur de vos décisions. Nous verrons comment optimiser pour différents types de charges de travail et comment mesurer l'impact de vos choix.

---

## Comprendre la performance dans MongoDB

### Les trois piliers de la performance

La performance dans MongoDB repose sur trois aspects fondamentaux :

#### 1. **Vitesse des requêtes** (Latence)
- Temps pour exécuter une requête unique
- Mesuré en millisecondes (ms)
- Objectif : < 10 ms pour les requêtes simples, < 100 ms pour les requêtes complexes

#### 2. **Débit** (Throughput)
- Nombre de requêtes traitées par seconde
- Mesuré en opérations/seconde (ops/s)
- Objectif : dépend de votre application (centaines à dizaines de milliers d'ops/s)

#### 3. **Utilisation des ressources**
- CPU, RAM, I/O disque, bande passante réseau
- Objectif : utilisation optimale sans saturation

### Le triangle de la performance

```
        Performance
           /\
          /  \
         /    \
        /      \
       /________\
  Vitesse    Ressources
              Débit
```

**Compromis à considérer :**
- ✅ Optimiser pour la vitesse → peut augmenter l'utilisation mémoire (caches, index)
- ✅ Optimiser pour le débit → peut nécessiter plus de CPU
- ✅ Réduire les ressources → peut impacter vitesse et débit

---

## Principes de conception orientée performance

### Principe 1 : Modéliser selon les patterns d'accès

**Règle d'or :** Votre schéma doit refléter **comment l'application accède aux données**, pas une structure abstraite idéale.

#### Questions à se poser :

1. **Quelles sont les 20% de requêtes qui représentent 80% du trafic ?**
   - Optimisez d'abord ces requêtes critiques

2. **Les données sont-elles lues plus qu'écrites, ou l'inverse ?**
   - Ratio lecture/écriture : 90/10, 50/50, 10/90 ?

3. **Quels champs sont toujours consultés ensemble ?**
   - Ces champs doivent être dans le même document

4. **Quelles requêtes doivent être les plus rapides ?**
   - Page d'accueil, recherche, profil utilisateur ?

#### Exemple : Application e-commerce

**Pattern d'accès identifié :**
```
80% des requêtes :
- Afficher liste de produits avec catégorie (50%)
- Afficher détails produit (20%)
- Rechercher produits (10%)

20% des requêtes :
- Créer commande (5%)
- Modifier produit (2%)
- Autres... (13%)
```

**Optimisation pour ces patterns :**

```javascript
// ✅ Optimisé pour affichage liste et recherche
{
  _id: ObjectId("..."),
  nom: "Smartphone XYZ Pro",
  prix: 899.99,
  categorieId: ObjectId("..."),
  categorieNom: "Électronique",  // ← Dénormalisé pour vitesse
  categoriePath: "Électronique > Smartphones",
  image: "https://cdn.exemple.com/smartphone-xyz.jpg",
  stock: 45,
  noteMoyenne: 4.5,  // ← Précalculé
  nombreAvis: 234,   // ← Précalculé

  // Détails chargés seulement sur la page produit
  description: "Smartphone haute performance...",
  specifications: {
    ecran: "6.7 pouces",
    memoire: "8 GB"
  }
}
```

**Index pour ces patterns :**

```javascript
// Pour la recherche et le tri
db.produits.createIndex({ categorieId: 1, prix: 1 })
db.produits.createIndex({ categorieId: 1, noteMoyenne: -1 })

// Pour la recherche texte
db.produits.createIndex({ nom: "text", description: "text" })
```

---

### Principe 2 : Minimiser le nombre de requêtes

**Impact du nombre de requêtes :**

```javascript
// ❌ LENT : 3 requêtes
const utilisateur = db.utilisateurs.findOne({ _id: userId })        // 10 ms
const profil = db.profils.findOne({ utilisateurId: userId })        // 10 ms
const preferences = db.preferences.findOne({ utilisateurId: userId }) // 10 ms
// Total : 30 ms + overhead réseau (× 3)

// ✅ RAPIDE : 1 requête
const utilisateur = db.utilisateurs.findOne({ _id: userId })  // 15 ms
// Total : 15 ms + overhead réseau (× 1)
```

**Stratégies pour réduire les requêtes :**

#### 1. Embedding (Imbrication)

```javascript
// ✅ Tout en un document
{
  _id: ObjectId("..."),
  nom: "Sophie Martin",
  email: "sophie@example.com",
  profil: {  // ← Imbriqué
    photo: "https://...",
    bio: "Développeuse...",
    dateNaissance: ISODate("1995-03-15")
  },
  preferences: {  // ← Imbriqué
    langue: "fr",
    theme: "sombre",
    notifications: true
  }
}
```

#### 2. Dénormalisation sélective

```javascript
// Collection "commandes"
{
  _id: ObjectId("..."),
  numeroCommande: "CMD-001",
  clientId: ObjectId("..."),
  // ↓ Infos client dénormalisées (évite une jointure)
  clientNom: "Sophie Martin",
  clientEmail: "sophie@example.com",
  articles: [ /* ... */ ],
  total: 299.99
}
```

#### 3. Agrégation avec $lookup (quand nécessaire)

```javascript
// Au lieu de 2 requêtes séparées
db.commandes.aggregate([
  { $match: { clientId: ObjectId("...") } },
  {
    $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },
  { $unwind: "$client" }
])
```

---

### Principe 3 : Optimiser la taille des documents

**Impact de la taille :**

```javascript
// Document de 1 Ko × 1000 requêtes = 1 Mo transféré
// Document de 100 Ko × 1000 requêtes = 100 Mo transféré
```

#### Stratégie 1 : Projections

Ne chargez que les champs nécessaires :

```javascript
// ❌ Charger tout (500 Ko par document)
db.articles.find({ categorie: "tech" })

// ✅ Projeter seulement ce qui est nécessaire (5 Ko par document)
db.articles.find(
  { categorie: "tech" },
  {
    titre: 1,
    resume: 1,
    auteur: 1,
    datePublication: 1,
    _id: 1
  }
)
```

**Gain :** 100x moins de données transférées !

#### Stratégie 2 : Lazy loading

Charger les détails seulement quand nécessaire :

```javascript
// Collection "produits"
{
  _id: ObjectId("..."),
  nom: "Smartphone XYZ",
  prix: 899.99,
  image: "https://...",
  // Infos légères pour la liste

  // ↓ Détails volumineux dans un sous-document séparé
  detailsId: ObjectId("...")
}

// Collection "detailsProduits"
{
  _id: ObjectId("..."),
  produitId: ObjectId("..."),
  description: "Description très longue...",  // 50 Ko
  specifications: { /* détails complets */ },  // 20 Ko
  avisDetailles: [ /* tous les avis */ ]       // 100 Ko
}
```

**Usage :**

```javascript
// Étape 1 : Afficher la liste (rapide)
const produits = db.produits.find({ categorie: "smartphones" })

// Étape 2 : Charger les détails seulement si l'utilisateur clique (à la demande)
const details = db.detailsProduits.findOne({ produitId: produit._id })
```

---

## Optimisation des lectures

### 1. Index stratégiques

**Impact des index :**

```javascript
// Sans index
db.produits.find({ categorieId: ObjectId("...") }).explain("executionStats")
// → COLLSCAN : 1000 ms pour 1 million de documents

// Avec index
db.produits.createIndex({ categorieId: 1 })
db.produits.find({ categorieId: ObjectId("...") }).explain("executionStats")
// → IXSCAN : 5 ms pour les mêmes données
```

**Gain :** 200x plus rapide !

#### Index simples vs composés

```javascript
// ❌ Index simples séparés (sous-optimal)
db.produits.createIndex({ categorieId: 1 })
db.produits.createIndex({ prix: 1 })

// Requête ne peut utiliser qu'un seul index
db.produits.find({
  categorieId: ObjectId("..."),
  prix: { $gte: 100, $lte: 500 }
})
// → Utilise l'index categorieId, puis filtre le prix en mémoire

// ✅ Index composé (optimal)
db.produits.createIndex({ categorieId: 1, prix: 1 })

// Requête utilise l'index complet
db.produits.find({
  categorieId: ObjectId("..."),
  prix: { $gte: 100, $lte: 500 }
})
// → IXSCAN sur les deux champs
```

**Performance :**
```
Index séparés : 50 ms (utilise 1 index + filtrage)
Index composé : 5 ms (utilise l'index complet)
Gain : 10x plus rapide
```

#### Ordre des champs dans l'index composé

**Règle ESR (Equality, Sort, Range) :**

1. **E**quality : Champs avec égalité (=)
2. **S**ort : Champs de tri
3. **R**ange : Champs avec plage ($gt, $lt, etc.)

```javascript
// Requête typique
db.produits.find({
  categorieId: ObjectId("..."),  // ← Equality
  prix: { $gte: 100, $lte: 500 } // ← Range
})
.sort({ dateAjout: -1 })         // ← Sort

// ✅ Index optimal (ordre ESR)
db.produits.createIndex({
  categorieId: 1,   // E
  dateAjout: -1,    // S
  prix: 1           // R
})
```

### 2. Covered Queries (Requêtes couvertes)

Une requête **covered** obtient toutes les données directement de l'index, sans lire le document.

```javascript
// Créer un index qui contient tous les champs nécessaires
db.produits.createIndex({
  categorieId: 1,
  nom: 1,
  prix: 1
})

// ✅ Requête couverte (ultra-rapide)
db.produits.find(
  { categorieId: ObjectId("...") },
  { nom: 1, prix: 1, _id: 0 }  // ← _id: 0 important !
)
```

**Vérification avec explain() :**

```javascript
db.produits.find(
  { categorieId: ObjectId("...") },
  { nom: 1, prix: 1, _id: 0 }
).explain("executionStats")

// Résultat recherché :
{
  stage: "IXSCAN",
  totalDocsExamined: 0,  // ← 0 = requête couverte !
  totalKeysExamined: 123
}
```

**Performance :**
```
Requête normale : 10 ms (lecture index + documents)
Requête couverte : 1 ms (lecture index uniquement)
Gain : 10x plus rapide
```

### 3. Computed fields (Champs calculés)

Précalculer au lieu de calculer à chaque lecture.

#### Exemple 1 : Statistiques d'article

```javascript
// ❌ LENT : Calculer à chaque affichage
const article = db.articles.findOne({ _id: articleId })

// Compter les likes à chaque fois
const nombreLikes = db.likes.countDocuments({ articleId: article._id })
// Compter les commentaires à chaque fois
const nombreCommentaires = db.commentaires.countDocuments({ articleId: article._id })

// Total : 3 requêtes

// ✅ RAPIDE : Précalculé
{
  _id: ObjectId("..."),
  titre: "Article",
  contenu: "...",
  statistiques: {  // ← Précalculé à l'écriture
    vues: 5234,
    likes: 342,
    commentaires: 89,
    partages: 23
  }
}

// Total : 1 requête
```

**Mise à jour des compteurs :**

```javascript
// Incrémenter lors d'un like
db.articles.updateOne(
  { _id: articleId },
  { $inc: { "statistiques.likes": 1 } }
)

// Incrémenter lors d'un commentaire
db.articles.updateOne(
  { _id: articleId },
  { $inc: { "statistiques.commentaires": 1 } }
)
```

#### Exemple 2 : Total de commande

```javascript
// ✅ Totaux précalculés
{
  _id: ObjectId("..."),
  numeroCommande: "CMD-001",
  articles: [
    { nom: "Produit A", quantite: 2, prix: 29.99, sousTotal: 59.98 },
    { nom: "Produit B", quantite: 1, prix: 89.99, sousTotal: 89.99 }
  ],
  sousTotal: 149.97,    // ← Précalculé
  tva: 29.99,           // ← Précalculé (20%)
  fraisPort: 5.00,      // ← Précalculé
  total: 184.96         // ← Précalculé
}
```

**Avantage :** Affichage instantané, pas de calcul côté application.

---

## Optimisation des écritures

### 1. Batch operations (Opérations par lot)

**Impact des opérations unitaires :**

```javascript
// ❌ LENT : Insertions une par une
for (let i = 0; i < 1000; i++) {
  db.logs.insertOne({ message: `Log ${i}`, date: new Date() })
}
// Temps : ~5 secondes (5 ms × 1000)

// ✅ RAPIDE : Insertion par lot
const logs = []
for (let i = 0; i < 1000; i++) {
  logs.push({ message: `Log ${i}`, date: new Date() })
}
db.logs.insertMany(logs)
// Temps : ~50 ms

// Gain : 100x plus rapide !
```

**Règle :** Toujours grouper les opérations quand possible.

### 2. Bulk operations

Pour les mises à jour massives :

```javascript
// ❌ LENT : Mises à jour une par une
const produits = db.produits.find({ stock: { $lt: 10 } })
produits.forEach(produit => {
  db.produits.updateOne(
    { _id: produit._id },
    { $set: { statut: "rupture_stock" } }
  )
})
// Temps : 10 secondes pour 1000 produits

// ✅ RAPIDE : Mise à jour groupée
db.produits.updateMany(
  { stock: { $lt: 10 } },
  { $set: { statut: "rupture_stock" } }
)
// Temps : 50 ms

// ✅ RAPIDE : Bulk write pour opérations complexes
const operations = [
  {
    updateOne: {
      filter: { _id: ObjectId("...") },
      update: { $set: { stock: 0 } }
    }
  },
  {
    updateOne: {
      filter: { _id: ObjectId("...") },
      update: { $inc: { stock: 10 } }
    }
  }
  // ... 1000 opérations
]

db.produits.bulkWrite(operations)
// Temps : 100 ms
```

### 3. Write Concern adapté

Compromis entre **vitesse** et **durabilité** :

```javascript
// ✅ Écriture rapide (pas d'attente de confirmation)
db.logs.insertOne(
  { message: "Log", date: new Date() },
  { writeConcern: { w: 0 } }  // Fire and forget
)
// Temps : 1 ms (pas d'attente)

// ✅ Écriture normale (attente confirmation primary)
db.commandes.insertOne(
  { numeroCommande: "CMD-001", total: 299.99 },
  { writeConcern: { w: 1 } }  // Défaut
)
// Temps : 5 ms

// ✅ Écriture sécurisée (attente réplication + journal)
db.transactions.insertOne(
  { type: "paiement", montant: 1000 },
  { writeConcern: { w: "majority", j: true } }
)
// Temps : 20 ms (attente réplication)
```

**Choix du write concern :**

| Type de données | Write Concern recommandé |
|-----------------|-------------------------|
| Logs, metrics non critiques | `{ w: 0 }` ou `{ w: 1 }` |
| Données applicatives standard | `{ w: 1 }` (défaut) |
| Transactions financières | `{ w: "majority", j: true }` |
| Données critiques | `{ w: "majority", j: true }` |

---

## Équilibrer lecture et écriture

### Ratio lecture/écriture

Votre modélisation doit s'adapter au ratio lecture/écriture de votre application.

#### Scénario 1 : Read-heavy (90% lectures / 10% écritures)

**Exemple :** Site d'actualités, catalogue produits

**Optimisation :**
- ✅ Dénormaliser agressivement
- ✅ Précalculer toutes les statistiques
- ✅ Imbriquer les données fréquemment consultées
- ✅ Index généreux sur les champs de recherche

```javascript
// ✅ Optimisé pour lecture
{
  _id: ObjectId("..."),
  titre: "Article",
  auteur: {  // ← Imbriqué (évite jointure)
    id: ObjectId("..."),
    nom: "Jean Martin",
    photo: "https://..."
  },
  statistiques: {  // ← Précalculé
    vues: 5234,
    likes: 342,
    commentaires: 89
  },
  commentairesRecents: [ /* top 5 */ ],  // ← Subset
  nombreCommentairesTotal: 89
}
```

**Compromis accepté :** Écritures plus complexes pour des lectures ultra-rapides.

#### Scénario 2 : Write-heavy (10% lectures / 90% écritures)

**Exemple :** Logs, métriques IoT, analytics

**Optimisation :**
- ✅ Éviter la dénormalisation excessive
- ✅ Limiter les index (ralentissent les écritures)
- ✅ Utiliser le bucketing pour réduire le nombre de documents
- ✅ Write concern relaxé (w: 0 ou w: 1)

```javascript
// ✅ Optimisé pour écriture
{
  _id: ObjectId("..."),
  capteurId: "SENSOR-001",
  date: ISODate("2024-01-15T10:00:00Z"),
  mesures: [  // ← Bucket (60 mesures par document)
    { ts: ISODate("..."), temp: 22.5 },
    { ts: ISODate("..."), temp: 22.6 }
    // ... 58 autres
  ]
}

// Index minimal (seulement ce qui est nécessaire)
db.mesures.createIndex({ capteurId: 1, date: -1 })
```

#### Scénario 3 : Balanced (50% lectures / 50% écritures)

**Exemple :** Réseau social, messagerie

**Optimisation :**
- ✅ Dénormalisation sélective (champs fréquents seulement)
- ✅ Index sur les champs de requête courants
- ✅ Computed fields pour statistiques importantes
- ✅ Références pour données volumineuses

```javascript
// ✅ Équilibré
{
  _id: ObjectId("..."),
  auteurId: ObjectId("..."),
  auteurNom: "Sophie",  // ← Dénormalisé (lecture)
  texte: "Mon message",
  date: ISODate("..."),
  likes: 42,  // ← Compteur (lecture/écriture équilibrées)
  nombreCommentaires: 12  // ← Compteur
}

// Collection séparée pour commentaires (volume)
```

---

## Mesurer et optimiser

### 1. explain() - Votre meilleur ami

```javascript
// Analyser une requête
db.produits.find({
  categorieId: ObjectId("..."),
  prix: { $gte: 100, $lte: 500 }
}).explain("executionStats")
```

**Métriques clés à surveiller :**

```javascript
{
  executionTimeMillis: 5,      // ← Temps d'exécution (objectif : < 10 ms)
  totalDocsExamined: 123,      // ← Documents examinés
  totalKeysExamined: 123,      // ← Clés d'index examinées
  nReturned: 123,              // ← Documents retournés

  executionStages: {
    stage: "IXSCAN",           // ← IXSCAN = bon, COLLSCAN = mauvais
    indexName: "categorieId_1_prix_1"
  }
}
```

**Indicateurs de problème :**

```javascript
// 🔴 Problème : Collection scan
stage: "COLLSCAN"
→ Solution : Créer un index

// 🔴 Problème : Beaucoup de documents examinés
totalDocsExamined: 10000
nReturned: 10
→ Solution : Améliorer l'index (trop de documents scannés)

// 🔴 Problème : Temps trop long
executionTimeMillis: 1523
→ Solution : Analyser le plan d'exécution, vérifier les index

// 🔴 Problème : Index non utilisé
stage: "COLLSCAN"
indexesAvailable: ["categorieId_1"]
→ Solution : Requête ne correspond pas à l'index
```

### 2. Database Profiler

Identifier les requêtes lentes automatiquement :

```javascript
// Activer le profiler (niveau 1 = requêtes > 100 ms)
db.setProfilingLevel(1, { slowms: 100 })

// Activer le profiler (niveau 2 = toutes les requêtes)
db.setProfilingLevel(2)

// Consulter les requêtes lentes
db.system.profile.find({
  millis: { $gt: 100 }
}).sort({ ts: -1 }).limit(10)
```

**Exemple de résultat :**

```javascript
{
  op: "query",
  ns: "mondb.produits",
  command: {
    find: "produits",
    filter: { categorieId: ObjectId("...") }
  },
  keysExamined: 0,      // ← 0 = pas d'index utilisé !
  docsExamined: 50000,  // ← Tous les documents scannés
  nreturned: 1234,
  millis: 1523,         // ← 1.5 secondes !
  planSummary: "COLLSCAN"
}
```

### 3. Monitoring continu

```javascript
// serverStatus pour métriques globales
db.serverStatus().opcounters
// {
//   insert: 123456,
//   query: 987654,
//   update: 456789,
//   delete: 12345
// }

// collStats pour une collection
db.produits.stats()
// {
//   count: 100000,
//   size: 52428800,        // 50 Mo
//   avgObjSize: 524,       // 524 octets en moyenne
//   storageSize: 26214400,
//   nindexes: 5,
//   totalIndexSize: 5242880
// }
```

---

## Cas d'usage et patterns de performance

### Cas 1 : E-commerce - Liste de produits

**Objectif :** Afficher 20 produits en < 10 ms

**Schéma optimisé :**

```javascript
{
  _id: ObjectId("..."),
  nom: "Smartphone XYZ",
  prix: 899.99,
  image: "https://cdn.exemple.com/thumb-smartphone.jpg",  // ← Miniature
  categorieId: ObjectId("..."),
  categorieNom: "Smartphones",  // ← Dénormalisé
  stock: 45,
  noteMoyenne: 4.5,  // ← Précalculé
  nombreAvis: 234,   // ← Précalculé
  badges: ["nouveau", "promo"]  // ← Précalculé
}

// Index composé
db.produits.createIndex({ categorieId: 1, prix: 1 })

// Requête optimisée avec projection
db.produits.find(
  { categorieId: ObjectId("...") },
  {
    nom: 1,
    prix: 1,
    image: 1,
    noteMoyenne: 1,
    nombreAvis: 1,
    badges: 1
  }
)
.sort({ prix: 1 })
.limit(20)
```

**Performance :** 3-5 ms

### Cas 2 : Blog - Page article

**Objectif :** Afficher article avec commentaires en < 20 ms

**Schéma hybride :**

```javascript
// Collection "articles"
{
  _id: ObjectId("..."),
  titre: "Guide MongoDB",
  contenu: "...",
  auteur: {  // ← Imbriqué
    id: ObjectId("..."),
    nom: "Jean Martin",
    photo: "https://..."
  },
  commentairesRecents: [  // ← Subset : 5 derniers
    {
      auteur: "Sophie",
      texte: "Super article !",
      date: ISODate("...")
    }
    // ... 4 autres
  ],
  nombreCommentairesTotal: 247,
  statistiques: {  // ← Précalculé
    vues: 5234,
    likes: 342
  },
  datePublication: ISODate("...")
}

// Collection "commentaires" (tous)
{
  _id: ObjectId("..."),
  articleId: ObjectId("..."),
  auteur: "Sophie",
  texte: "Super article !",
  date: ISODate("...")
}
```

**Requête optimisée :**

```javascript
// Affichage initial (1 requête)
const article = db.articles.findOne({ slug: "guide-mongodb" })

// Charger plus de commentaires seulement si demandé (lazy loading)
if (utilisateurCliqueSurVoirPlus) {
  const commentaires = db.commentaires
    .find({ articleId: article._id })
    .sort({ date: -1 })
    .skip(5)  // Sauter les 5 déjà affichés
    .limit(20)
}
```

**Performance :** 10-15 ms (affichage initial)

### Cas 3 : Analytics - Dashboard temps réel

**Objectif :** Afficher métriques en < 50 ms

**Schéma avec agrégations précalculées :**

```javascript
// Collection "metrics_hourly" (pré-agrégées par heure)
{
  _id: ObjectId("..."),
  date: ISODate("2024-01-15T10:00:00Z"),
  metrics: {
    vues: 12340,
    visiteursUniques: 8234,
    conversions: 123,
    revenu: 45678.90,
    tauxConversion: 1.49  // Précalculé
  },
  parSource: [
    { source: "organic", vues: 7000 },
    { source: "social", vues: 3000 },
    { source: "direct", vues: 2340 }
  ]
}

// Index pour requêtes temporelles
db.metrics_hourly.createIndex({ date: -1 })

// Requête dashboard (dernières 24 heures)
db.metrics_hourly.find({
  date: {
    $gte: ISODate("2024-01-14T10:00:00Z"),
    $lt: ISODate("2024-01-15T10:00:00Z")
  }
}).sort({ date: -1 })
```

**Performance :** 20-30 ms

---

## Checklist d'optimisation

### ✅ Avant le déploiement

- [ ] **Patterns d'accès documentés** : Quelles sont les requêtes critiques ?
- [ ] **Index créés** : Tous les champs de requête sont indexés
- [ ] **explain() vérifié** : Toutes les requêtes critiques utilisent des index (IXSCAN)
- [ ] **Taille des documents** : Aucun document > 1 Mo (objectif : < 100 Ko)
- [ ] **Computed fields** : Statistiques et totaux précalculés
- [ ] **Projections** : Charger seulement les champs nécessaires
- [ ] **Dénormalisation** : Champs fréquents dénormalisés stratégiquement
- [ ] **Tests de charge** : Performance validée avec volumes réalistes

### ✅ En production

- [ ] **Profiler activé** : Surveiller les requêtes lentes
- [ ] **Monitoring** : Métriques de performance collectées
- [ ] **Alertes** : Notifications si requêtes > 100 ms
- [ ] **Revue régulière** : Analyser les requêtes lentes mensuellement
- [ ] **Index maintenance** : Supprimer les index inutilisés

---

## Outils et ressources

### Outils de mesure

1. **MongoDB Compass**
   - Analyse visuelle des requêtes
   - Plan d'exécution graphique
   - Statistiques de collection

2. **explain() dans mongosh**
   - Analyse détaillée des requêtes
   - Utilisation des index
   - Temps d'exécution

3. **Database Profiler**
   - Détection automatique des requêtes lentes
   - Historique des performances

4. **mongostat / mongotop**
   - Monitoring en temps réel
   - Opérations par seconde
   - Utilisation des ressources

### Métriques clés à surveiller

```javascript
// Opérations par seconde
db.serverStatus().opcounters

// Utilisation des index
db.produits.aggregate([{ $indexStats: {} }])

// Taille des collections
db.stats()

// Performance des requêtes
db.system.profile.find().sort({ ts: -1 })
```

---

## Conclusion

La conception pour la performance dans MongoDB repose sur des principes fondamentaux :

**Les 7 commandements de la performance :**

1. 🎯 **Modéliser selon les patterns d'accès** : Votre schéma doit refléter comment l'application utilise les données
2. 🎯 **Minimiser les requêtes** : Embedding et dénormalisation stratégique
3. 🎯 **Créer des index appropriés** : Sur tous les champs de requête
4. 🎯 **Précalculer** : Computed fields pour les valeurs fréquentes
5. 🎯 **Projeter** : Charger seulement ce qui est nécessaire
6. 🎯 **Mesurer** : explain() et profiler sont vos amis
7. 🎯 **Itérer** : Optimiser basé sur les données réelles

**Règles d'or :**

- ✅ **Optimiser pour le cas d'usage principal** (80/20)
- ✅ **Mesurer avant d'optimiser** : Ne pas optimiser à l'aveugle
- ✅ **Accepter les compromis** : Équilibrer lecture vs écriture
- ✅ **Tester en conditions réelles** : Volume et charge réalistes
- ✅ **Surveiller en continu** : La performance évolue avec les données

N'oubliez pas : **Un schéma bien conçu peut offrir d'excellentes performances même sur du matériel modeste, tandis qu'un schéma mal conçu sera lent même sur du matériel haut de gamme.**

---

**Points clés à retenir :**

- ✅ Modéliser selon les patterns d'accès, pas selon une structure idéale
- ✅ Minimiser le nombre de requêtes avec embedding et dénormalisation
- ✅ Créer des index sur tous les champs de requête
- ✅ Utiliser explain() pour analyser toutes les requêtes critiques
- ✅ Précalculer les valeurs fréquemment utilisées
- ✅ Projeter seulement les champs nécessaires
- ✅ Équilibrer optimisation lecture vs écriture selon le ratio
- ✅ Mesurer et monitorer continuellement les performances

---


⏭️ [Index et Optimisation](/05-index-et-optimisation/README.md)
