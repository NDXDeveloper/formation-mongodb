🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Requêtes et Filtres

## Bienvenue dans l'art de l'interrogation de données ! 🔍

Vous savez maintenant **créer, lire, mettre à jour et supprimer** des documents dans MongoDB. C'est excellent ! Mais dans le monde réel, vous aurez besoin de bien plus que de simples requêtes basiques. Comment rechercher tous les clients d'une ville spécifique ? Comment trouver les produits dont le prix est compris entre 10 et 50 euros ? Comment filtrer les articles publiés le mois dernier ?

Ce chapitre va transformer vos compétences de recherche de base en une maîtrise approfondie des requêtes MongoDB. Vous allez découvrir la richesse et la puissance du langage de requête MongoDB.

## Où en sommes-nous dans votre parcours ?

Vous avez complété les chapitres 1 et 2 et vous maîtrisez maintenant :
- ✅ Les concepts fondamentaux de MongoDB (chapitre 1)
- ✅ La structure des documents BSON et les types de données
- ✅ Les opérations CRUD de base : `insertOne()`, `find()`, `updateOne()`, `deleteOne()`
- ✅ L'utilisation de mongosh et MongoDB Compass

**Parfait !** Vous êtes maintenant prêt à approfondir vos capacités de recherche et à écrire des requêtes sophistiquées.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- ✅ **Construire** des requêtes complexes avec de multiples critères
- ✅ **Utiliser** tous les opérateurs de comparaison ($gt, $lt, $in, etc.)
- ✅ **Combiner** des conditions avec les opérateurs logiques ($and, $or, $not)
- ✅ **Interroger** des tableaux et des documents imbriqués
- ✅ **Maîtriser** les projections pour sélectionner uniquement les champs nécessaires
- ✅ **Optimiser** vos requêtes avec tri, limite et pagination
- ✅ **Rechercher** avec des expressions régulières et des conditions complexes
- ✅ **Compter** efficacement les documents selon différents critères

## Du simple au complexe : l'évolution de vos requêtes

### Vos requêtes actuelles (Chapitre 2)

```javascript
// Recherche simple par égalité
db.produits.find({ nom: "Ordinateur portable" })

// Recherche d'un seul document
db.utilisateurs.findOne({ email: "alice@example.com" })

// Recherche de tous les documents
db.articles.find()
```

**Ces requêtes sont un bon début**, mais elles sont limitées. Dans la vraie vie, vous aurez besoin de bien plus !

### Vos futures requêtes (Après ce chapitre)

```javascript
// Recherche de produits dans une fourchette de prix
db.produits.find({
    prix: { $gte: 100, $lte: 500 },
    categorie: { $in: ["Électronique", "Informatique"] },
    stock: { $gt: 0 }
})

// Recherche d'utilisateurs actifs avec conditions multiples
db.utilisateurs.find({
    $and: [
        { dateInscription: { $gte: new Date("2024-01-01") } },
        { $or: [
            { statut: "premium" },
            { achats: { $gte: 10 } }
        ]}
    ]
})

// Recherche avec projection (sélection de champs)
db.articles.find(
    { vues: { $gt: 1000 } },
    { titre: 1, auteur: 1, datePublication: 1, _id: 0 }
)

// Recherche avec tri et pagination
db.produits.find({ categorie: "Livres" })
    .sort({ prix: -1 })  // Tri décroissant par prix
    .skip(20)            // Sauter les 20 premiers
    .limit(10)           // Limiter à 10 résultats
```

**Impressionnant, n'est-ce pas ?** C'est exactement ce que vous saurez faire à la fin de ce chapitre !

## Vue d'ensemble du chapitre

Ce chapitre est organisé en 11 sections progressives qui couvrent tous les aspects des requêtes et filtres :

### 🎯 Partie 1 : Fondamentaux des requêtes (Section 3.1)
La **syntaxe de base** et les conventions d'écriture des requêtes MongoDB.

### 🎯 Partie 2 : Opérateurs de recherche (Sections 3.2 à 3.6)
Les différents types d'opérateurs pour construire vos filtres :
- **3.2** : Opérateurs de comparaison ($eq, $gt, $lt, $in, etc.)
- **3.3** : Opérateurs logiques ($and, $or, $not, $nor)
- **3.4** : Opérateurs d'éléments ($exists, $type)
- **3.5** : Opérateurs d'évaluation ($regex, $expr, $text, etc.)
- **3.6** : Opérateurs de tableaux ($all, $elemMatch, $size)

### 🎯 Partie 3 : Optimisation et présentation (Sections 3.7 à 3.9)
Comment façonner et optimiser vos résultats :
- **3.7** : Projections (sélection des champs)
- **3.8** : Tri, limite et pagination
- **3.9** : Comptage de documents

### 🎯 Partie 4 : Cas spéciaux (Sections 3.10 et 3.11)
Requêtes sur structures complexes :
- **3.10** : Requêtes sur documents imbriqués
- **3.11** : Requêtes sur tableaux

## Comprendre la philosophie des requêtes MongoDB

### Principe 1 : Les documents de requête

Dans MongoDB, les requêtes sont elles-mêmes des **documents** (objets JSON) :

```javascript
// Ceci est un document de requête
{
    age: { $gte: 18 },      // Condition 1
    ville: "Paris",          // Condition 2
    actif: true             // Condition 3
}

// Équivalent SQL conceptuel :
// WHERE age >= 18 AND ville = 'Paris' AND actif = true
```

**Avantage :** Cette approche est naturelle, lisible et suit la même structure que vos documents de données.

### Principe 2 : Les opérateurs préfixés par $

MongoDB utilise le symbole `$` pour tous ses opérateurs spéciaux :

```javascript
// $ indique un opérateur MongoDB
{
    prix: { $gt: 100 }        // $gt = "greater than" (plus grand que)
}

// Sans $, c'est une égalité stricte
{
    prix: 100                 // Cherche prix exactement égal à 100
}
```

**Important :** Le `$` distingue les opérateurs MongoDB des noms de champs ordinaires.

### Principe 3 : Composition et imbrication

Les conditions peuvent être imbriquées et combinées :

```javascript
// Conditions imbriquées
{
    prix: { $gte: 50, $lte: 100 },    // 50 <= prix <= 100
    stock: { $gt: 0 }                  // stock > 0
}

// Combinaison avec $and explicite
{
    $and: [
        { prix: { $gte: 50 } },
        { prix: { $lte: 100 } }
    ]
}
```

## Exemple progressif : une collection e-commerce

Pour illustrer l'évolution de vos compétences, utilisons un exemple concret. Imaginons une collection `produits` pour un site e-commerce :

```javascript
// Exemple de documents dans la collection produits
db.produits.insertMany([
    {
        _id: 1,
        nom: "Ordinateur portable Dell XPS",
        categorie: "Informatique",
        prix: 1299.99,
        stock: 15,
        marque: "Dell",
        caracteristiques: {
            processeur: "Intel i7",
            ram: 16,
            stockage: 512
        },
        tags: ["bureautique", "gaming", "portable"],
        dateAjout: new Date("2024-01-15"),
        promotion: false,
        note: 4.5
    },
    {
        _id: 2,
        nom: "Clavier mécanique RGB",
        categorie: "Accessoires",
        prix: 89.99,
        stock: 42,
        marque: "Corsair",
        caracteristiques: {
            type: "mécanique",
            switches: "Cherry MX Red",
            retro: true
        },
        tags: ["gaming", "RGB"],
        dateAjout: new Date("2024-02-01"),
        promotion: true,
        note: 4.8
    },
    {
        _id: 3,
        nom: "Souris sans fil Logitech",
        categorie: "Accessoires",
        prix: 39.99,
        stock: 0,  // Rupture de stock
        marque: "Logitech",
        caracteristiques: {
            type: "sans fil",
            dpi: 2400,
            batterie: "rechargeable"
        },
        tags: ["bureautique", "sans-fil"],
        dateAjout: new Date("2024-01-20"),
        promotion: false,
        note: 4.2
    }
])
```

### Niveau 1 : Requêtes simples (ce que vous savez déjà)

```javascript
// Rechercher un produit par nom exact
db.produits.find({ nom: "Ordinateur portable Dell XPS" })

// Rechercher tous les produits d'une catégorie
db.produits.find({ categorie: "Accessoires" })

// Rechercher les produits en promotion
db.produits.find({ promotion: true })
```

**Limitation :** Ces requêtes ne fonctionnent que pour des égalités exactes.

### Niveau 2 : Comparaisons (ce que vous allez apprendre)

```javascript
// Produits de moins de 100€
db.produits.find({
    prix: { $lt: 100 }
})
// Retourne : Clavier (89.99€) et Souris (39.99€)

// Produits entre 50€ et 100€
db.produits.find({
    prix: { $gte: 50, $lte: 100 }
})
// Retourne : Clavier (89.99€)

// Produits avec stock supérieur à 10
db.produits.find({
    stock: { $gt: 10 }
})
// Retourne : Ordinateur (15) et Clavier (42)
```

**Explication :**
- `$lt` = "less than" (plus petit que)
- `$lte` = "less than or equal" (plus petit ou égal)
- `$gt` = "greater than" (plus grand que)
- `$gte` = "greater than or equal" (plus grand ou égal)

### Niveau 3 : Conditions multiples (avancé)

```javascript
// Produits en stock ET à moins de 100€
db.produits.find({
    stock: { $gt: 0 },
    prix: { $lt: 100 }
})
// Retourne : Clavier uniquement (stock: 42, prix: 89.99)

// Produits de certaines marques
db.produits.find({
    marque: { $in: ["Dell", "Logitech"] }
})
// Retourne : Ordinateur Dell et Souris Logitech

// Produits en promotion OU avec note >= 4.5
db.produits.find({
    $or: [
        { promotion: true },
        { note: { $gte: 4.5 } }
    ]
})
// Retourne : Ordinateur (note: 4.5) et Clavier (promotion: true)
```

**Nouveaux opérateurs :**
- `$in` : valeur parmi une liste
- `$or` : condition OU logique

### Niveau 4 : Requêtes complexes (expert)

```javascript
// Produits gaming disponibles, avec note élevée, à prix raisonnable
db.produits.find({
    $and: [
        { tags: "gaming" },
        { stock: { $gt: 0 } },
        { note: { $gte: 4.0 } },
        { prix: { $lte: 1500 } }
    ]
})

// Produits avec caractéristiques spécifiques (document imbriqué)
db.produits.find({
    "caracteristiques.ram": { $gte: 16 }
})
// Retourne : Ordinateur (RAM: 16)

// Recherche textuelle dans le nom
db.produits.find({
    nom: { $regex: /clavier/i }  // i = insensible à la casse
})
// Retourne : Clavier mécanique RGB
```

**Concepts avancés :**
- `$and` explicite pour clarifier la logique
- Notation pointée pour documents imbriqués
- `$regex` pour recherche par motif

## Les grandes familles d'opérateurs

MongoDB organise ses opérateurs en plusieurs catégories. Voici un aperçu de ce que vous allez apprendre :

### 1. Opérateurs de comparaison (Section 3.2)

| Opérateur | Signification | Exemple |
|-----------|---------------|---------|
| `$eq` | Égal à (equal) | `{ age: { $eq: 25 } }` |
| `$ne` | Différent de (not equal) | `{ statut: { $ne: "inactif" } }` |
| `$gt` | Plus grand que (greater than) | `{ prix: { $gt: 100 } }` |
| `$gte` | Plus grand ou égal (≥) | `{ age: { $gte: 18 } }` |
| `$lt` | Plus petit que (less than) | `{ stock: { $lt: 5 } }` |
| `$lte` | Plus petit ou égal (≤) | `{ prix: { $lte: 50 } }` |
| `$in` | Dans une liste | `{ ville: { $in: ["Paris", "Lyon"] } }` |
| `$nin` | Pas dans une liste | `{ statut: { $nin: ["annulé", "remboursé"] } }` |

```javascript
// Exemple combiné
db.commandes.find({
    montant: { $gte: 100, $lte: 500 },    // Entre 100 et 500
    statut: { $in: ["payé", "expédié"] }   // Statut payé ou expédié
})
```

### 2. Opérateurs logiques (Section 3.3)

| Opérateur | Signification | Usage |
|-----------|---------------|-------|
| `$and` | ET logique | `{ $and: [condition1, condition2] }` |
| `$or` | OU logique | `{ $or: [condition1, condition2] }` |
| `$not` | NON logique (négation) | `{ age: { $not: { $gt: 18 } } }` |
| `$nor` | NI l'un NI l'autre | `{ $nor: [condition1, condition2] }` |

```javascript
// Client VIP : Premium OU plus de 50 commandes
db.clients.find({
    $or: [
        { statut: "premium" },
        { nombreCommandes: { $gte: 50 } }
    ]
})

// Produits ni en rupture ni en précommande
db.produits.find({
    $nor: [
        { stock: 0 },
        { statut: "précommande" }
    ]
})
```

### 3. Opérateurs d'éléments (Section 3.4)

| Opérateur | Signification | Usage |
|-----------|---------------|-------|
| `$exists` | Vérifie l'existence d'un champ | `{ email: { $exists: true } }` |
| `$type` | Vérifie le type d'un champ | `{ age: { $type: "number" } }` |

```javascript
// Utilisateurs qui ont fourni un numéro de téléphone
db.utilisateurs.find({
    telephone: { $exists: true, $ne: null }
})

// Documents où age est un nombre (et non une chaîne)
db.personnes.find({
    age: { $type: "number" }
})
```

### 4. Opérateurs d'évaluation (Section 3.5)

| Opérateur | Signification | Usage |
|-----------|---------------|-------|
| `$regex` | Recherche par expression régulière | `{ nom: { $regex: /^A/ } }` |
| `$expr` | Expression permettant d'utiliser des opérateurs d'agrégation | `{ $expr: { $gt: ["$stock", "$seuilAlerte"] } }` |
| `$text` | Recherche full-text | `{ $text: { $search: "mongodb" } }` |
| `$mod` | Modulo | `{ age: { $mod: [2, 0] } }` |

```javascript
// Noms commençant par "Mar"
db.clients.find({
    nom: { $regex: /^Mar/i }  // i = insensible à la casse
})

// Comparer deux champs du même document
db.produits.find({
    $expr: { $gt: ["$stock", "$seuilAlerte"] }
})
```

### 5. Opérateurs de tableaux (Section 3.6)

| Opérateur | Signification | Usage |
|-----------|---------------|-------|
| `$all` | Contient tous les éléments | `{ tags: { $all: ["gaming", "RGB"] } }` |
| `$elemMatch` | Au moins un élément satisfait les conditions | `{ scores: { $elemMatch: { $gte: 80, $lt: 90 } } }` |
| `$size` | Taille exacte du tableau | `{ tags: { $size: 3 } }` |

```javascript
// Produits avec les tags "gaming" ET "portable"
db.produits.find({
    tags: { $all: ["gaming", "portable"] }
})

// Étudiants avec au moins une note entre 15 et 18
db.etudiants.find({
    notes: { $elemMatch: { $gte: 15, $lte: 18 } }
})
```

## Projections : sélectionner uniquement ce dont vous avez besoin

Les projections vous permettent de contrôler quels champs sont retournés :

```javascript
// Par défaut, tous les champs sont retournés
db.produits.find({ categorie: "Informatique" })
// Retourne : { _id, nom, categorie, prix, stock, marque, ... }

// Projection : seulement nom et prix
db.produits.find(
    { categorie: "Informatique" },
    { nom: 1, prix: 1, _id: 0 }  // 1 = inclure, 0 = exclure
)
// Retourne : { nom: "...", prix: ... }
```

**Pourquoi c'est important ?**
- ⚡ Réduit la quantité de données transférées
- 🚀 Améliore les performances
- 📊 Facilite le traitement côté application

## Tri, limite et pagination : contrôler les résultats

```javascript
// Tri par prix croissant
db.produits.find().sort({ prix: 1 })  // 1 = ascendant

// Tri par prix décroissant
db.produits.find().sort({ prix: -1 })  // -1 = descendant

// Les 5 produits les moins chers
db.produits.find().sort({ prix: 1 }).limit(5)

// Pagination : page 2 (éléments 11 à 20)
db.produits.find()
    .sort({ dateAjout: -1 })  // Plus récents d'abord
    .skip(10)                  // Sauter les 10 premiers
    .limit(10)                 // Prendre les 10 suivants
```

**Cas d'usage réel :** Afficher des résultats de recherche page par page.

## Documents imbriqués et tableaux : cas spéciaux

### Requêtes sur documents imbriqués

```javascript
// Structure du document
{
    _id: 1,
    nom: "Ordinateur",
    caracteristiques: {
        processeur: "Intel i7",
        ram: 16,
        stockage: 512
    }
}

// Recherche avec notation pointée
db.produits.find({
    "caracteristiques.ram": { $gte: 16 }
})

// Recherche avec document complet (égalité stricte)
db.produits.find({
    caracteristiques: {
        processeur: "Intel i7",
        ram: 16,
        stockage: 512
    }
})
// ⚠️ Doit correspondre EXACTEMENT (ordre et champs)
```

### Requêtes sur tableaux

```javascript
// Tableau dans le document
{
    _id: 1,
    nom: "Ordinateur",
    tags: ["gaming", "portable", "bureautique"]
}

// Contient au moins un élément
db.produits.find({ tags: "gaming" })

// Contient tous les éléments
db.produits.find({ tags: { $all: ["gaming", "portable"] } })

// Taille exacte du tableau
db.produits.find({ tags: { $size: 3 } })

// Position spécifique
db.produits.find({ "tags.0": "gaming" })  // Premier élément
```

## Exemple réel complet : système de blog

Voyons un exemple concret qui combine plusieurs concepts :

```javascript
// Collection d'articles de blog
db.articles.insertMany([
    {
        titre: "Introduction à MongoDB",
        auteur: "Alice Dupont",
        categorie: "Tutoriels",
        tags: ["mongodb", "database", "nosql"],
        datePublication: new Date("2024-01-15"),
        vues: 1250,
        likes: 42,
        commentaires: [
            { auteur: "Bob", texte: "Super article !", date: new Date("2024-01-16") },
            { auteur: "Charlie", texte: "Très utile", date: new Date("2024-01-17") }
        ],
        statut: "publié",
        premium: false
    },
    {
        titre: "Guide avancé des agrégations",
        auteur: "Alice Dupont",
        categorie: "Tutoriels",
        tags: ["mongodb", "agregation", "avancé"],
        datePublication: new Date("2024-02-01"),
        vues: 850,
        likes: 28,
        commentaires: [
            { auteur: "David", texte: "Excellent !", date: new Date("2024-02-02") }
        ],
        statut: "publié",
        premium: true
    },
    {
        titre: "Actualités MongoDB 7.0",
        auteur: "Bob Martin",
        categorie: "News",
        tags: ["mongodb", "version", "news"],
        datePublication: new Date("2024-02-10"),
        vues: 320,
        likes: 15,
        commentaires: [],
        statut: "brouillon",
        premium: false
    }
])
```

### Requêtes pratiques sur ce blog

```javascript
// 1. Articles populaires (plus de 1000 vues)
db.articles.find({
    vues: { $gte: 1000 }
})

// 2. Articles publiés par Alice en 2024
db.articles.find({
    auteur: "Alice Dupont",
    statut: "publié",
    datePublication: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2025-01-01")
    }
})

// 3. Articles avec "mongodb" dans les tags ET plus de 30 likes
db.articles.find({
    tags: "mongodb",
    likes: { $gt: 30 }
})

// 4. Articles premium OU avec plus de 2 commentaires
db.articles.find({
    $or: [
        { premium: true },
        { "commentaires.2": { $exists: true } }  // 3ème commentaire existe
    ]
})

// 5. Articles récents avec sélection de champs
db.articles.find(
    {
        datePublication: { $gte: new Date("2024-02-01") },
        statut: "publié"
    },
    {
        titre: 1,
        auteur: 1,
        vues: 1,
        _id: 0
    }
).sort({ vues: -1 })

// 6. Recherche textuelle dans le titre
db.articles.find({
    titre: { $regex: /mongodb/i }
})

// 7. Articles sans commentaires
db.articles.find({
    $or: [
        { commentaires: { $size: 0 } },
        { commentaires: { $exists: false } }
    ]
})
```

## Comptage de documents

```javascript
// Compter tous les articles
db.articles.countDocuments()

// Compter les articles publiés
db.articles.countDocuments({ statut: "publié" })

// Estimation rapide (moins précis, plus rapide)
db.articles.estimatedDocumentCount()
```

## Points d'attention pour ce chapitre

### 1. L'ordre des conditions importe (parfois)

```javascript
// Ces deux requêtes sont équivalentes (ET implicite)
db.produits.find({ prix: { $lt: 100 }, stock: { $gt: 0 } })
db.produits.find({ stock: { $gt: 0 }, prix: { $lt: 100 } })

// Mais pour les performances avec des index, l'ordre peut avoir un impact
```

### 2. Égalité implicite vs explicite

```javascript
// Égalité implicite (recommandée quand c'est simple)
db.users.find({ age: 25 })

// Égalité explicite (utile pour la cohérence)
db.users.find({ age: { $eq: 25 } })

// Les deux sont identiques
```

### 3. Notation pointée pour documents imbriqués

```javascript
// ❌ Ne fonctionne pas
db.produits.find({ caracteristiques.ram: 16 })

// ✅ Correct (guillemets nécessaires)
db.produits.find({ "caracteristiques.ram": 16 })
```

### 4. Tableaux : contient vs égalité

```javascript
// Document
{ tags: ["a", "b", "c"] }

// Contient "a" (un seul élément suffit)
db.collection.find({ tags: "a" })  // ✅ Match

// Égalité stricte du tableau complet
db.collection.find({ tags: ["a", "b", "c"] })  // ✅ Match
db.collection.find({ tags: ["a", "b"] })      // ❌ Pas de match
```

## Conseils d'apprentissage pour ce chapitre

### 🎯 Méthodologie recommandée

1. **Section par section** : Ne sautez pas d'étapes, chaque section introduit de nouveaux opérateurs
2. **Testez chaque exemple** : Créez une collection test et expérimentez
3. **Combinez progressivement** : Commencez simple, puis combinez plusieurs opérateurs
4. **Utilisez Compass** : L'interface graphique aide à comprendre les résultats
5. **Lisez la documentation** : Chaque opérateur a des subtilités

### 💡 Astuces pratiques

```javascript
// Aide en ligne dans mongosh
db.collection.find.help()

// Compter les résultats
db.produits.find({ categorie: "Livres" }).count()

// Formater joliment
db.produits.find().pretty()

// Voir le plan d'exécution (pour l'optimisation, chapitre 5)
db.produits.find({ prix: { $gt: 100 } }).explain()
```

### 🔗 Connexion avec les chapitres futurs

- **Chapitre 4** : La modélisation influencera vos stratégies de requêtes
- **Chapitre 5** : Les index optimiseront drastiquement ces requêtes
- **Chapitre 6** : Les agrégations offrent des capacités encore plus puissantes

## Données de test pour pratiquer

Voici un jeu de données complet pour pratiquer :

```javascript
// Créer une base de test
use formation_requetes

// Insérer des données variées
db.pratique.insertMany([
    // Documents avec structures variées pour tester tous les opérateurs
    { _id: 1, nom: "Alice", age: 28, ville: "Paris", score: 85, actif: true,
      hobbies: ["lecture", "voyage"], dateInscription: new Date("2023-01-15") },
    { _id: 2, nom: "Bob", age: 35, ville: "Lyon", score: 92, actif: true,
      hobbies: ["sport", "cuisine", "photo"], dateInscription: new Date("2023-03-20") },
    { _id: 3, nom: "Charlie", age: 22, ville: "Paris", score: 78, actif: false,
      hobbies: ["gaming"], dateInscription: new Date("2023-06-10") },
    { _id: 4, nom: "Diana", age: 41, ville: "Marseille", score: 88, actif: true,
      hobbies: ["lecture", "musique"], dateInscription: new Date("2023-02-05") },
    { _id: 5, nom: "Étienne", age: 29, ville: "Paris", score: 95, actif: true,
      hobbies: ["sport", "voyage", "lecture"], dateInscription: new Date("2023-04-12") },
    { _id: 6, nom: "Fanny", ville: "Lyon", score: 70, actif: false,
      hobbies: [], dateInscription: new Date("2023-08-30") }  // age manquant
])
```

**Essayez ces requêtes :**
```javascript
// Personnes de Paris de plus de 25 ans
// Score entre 80 et 90
// Inscrites en 2023
// Ayant "lecture" dans leurs hobbies
// etc.
```

## Ce que vous allez maîtriser

À la fin de ce chapitre, vous serez capable de répondre à des questions comme :

- ❓ Trouver tous les produits entre 50€ et 150€ en stock
- ❓ Rechercher les utilisateurs actifs inscrits après janvier 2024
- ❓ Filtrer les articles contenant "MongoDB" dans le titre
- ❓ Compter les commandes supérieures à 100€ par statut
- ❓ Récupérer uniquement le nom et le prix des produits
- ❓ Afficher les 10 articles les plus récents
- ❓ Paginer des résultats de recherche
- ❓ Requêter des données dans des documents imbriqués
- ❓ Rechercher dans des tableaux avec conditions complexes

---

### 📌 Points clés à retenir de cette introduction

- MongoDB offre un langage de requête riche et expressif
- Les requêtes sont des documents JSON utilisant des opérateurs préfixés par $
- Il existe 5 grandes familles d'opérateurs (comparaison, logiques, éléments, évaluation, tableaux)
- Les projections permettent de sélectionner uniquement les champs nécessaires
- Le tri, la limite et la pagination contrôlent la présentation des résultats
- Les documents imbriqués utilisent la notation pointée
- Les tableaux ont des opérateurs spécifiques ($all, $elemMatch, $size)
- Combiner les opérateurs permet de construire des requêtes très sophistiquées

---

**Durée estimée du chapitre** : 6-8 heures de lecture et pratique
**Niveau** : Intermédiaire débutant
**Prérequis** : Chapitres 1 et 2 complétés, maîtrise des opérations CRUD

🎯 **Prochaine étape** : Dans la section 3.1, nous allons commencer par la syntaxe de base des requêtes et établir les fondations solides sur lesquelles nous construirons toutes vos compétences de recherche.

---

**Prochaine section** : 3.1 - Syntaxe des requêtes de base

Prêt à devenir un expert des requêtes MongoDB ? Allons-y ! 🚀

⏭️ [Syntaxe des requêtes de base](/03-requetes-et-filtres/01-syntaxe-requetes-base.md)
