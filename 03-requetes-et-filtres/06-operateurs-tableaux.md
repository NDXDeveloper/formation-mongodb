🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.6 Opérateurs de Tableaux

## Introduction

MongoDB est particulièrement puissant pour travailler avec des **tableaux** stockés directement dans les documents. Contrairement aux bases de données relationnelles où les relations un-à-plusieurs nécessitent des tables séparées et des jointures, MongoDB permet de stocker des collections d'éléments directement dans un document.

Cependant, interroger des tableaux nécessite des opérateurs spécialisés. Par exemple :
- Trouver des documents dont un tableau contient **tous** les éléments d'une liste
- Rechercher des éléments de tableau correspondant à **plusieurs conditions**
- Filtrer par la **taille** d'un tableau

MongoDB fournit des **opérateurs de tableaux** spécifiquement conçus pour ces besoins. Dans ce chapitre, nous allons explorer les trois opérateurs principaux : `$all`, `$elemMatch` et `$size`.

---

## Vue d'Ensemble des Opérateurs de Tableaux

| Opérateur | Description | Usage Principal |
|-----------|-------------|-----------------|
| `$all` | Vérifie qu'un tableau contient tous les éléments spécifiés | Recherche par ensemble de valeurs |
| `$elemMatch` | Correspond à des éléments de tableau satisfaisant plusieurs conditions | Requêtes complexes sur éléments de tableau |
| `$size` | Vérifie la taille (nombre d'éléments) d'un tableau | Filtrage par longueur de tableau |

---

## Rappel : Requêtes de Base sur Tableaux

Avant d'explorer les opérateurs spécialisés, rappelons les requêtes de base sur tableaux :

### Recherche Simple dans un Tableau

```javascript
// Document exemple
{
    name: "Alice",
    hobbies: ["reading", "swimming", "coding"]
}

// Trouver les documents où hobbies contient "reading"
db.users.find({ hobbies: "reading" })
// Retourne le document (MongoDB vérifie automatiquement dans le tableau)

// Trouver les documents où hobbies contient "reading" OU "gaming"
db.users.find({
    hobbies: { $in: ["reading", "gaming"] }
})
```

### Correspondance Exacte de Tableau

```javascript
// Tableau exact dans le même ordre
db.users.find({
    hobbies: ["reading", "swimming", "coding"]
})
// Ne correspond qu'aux documents avec exactement ce tableau dans cet ordre
```

Maintenant, voyons les opérateurs spécialisés.

---

## L'Opérateur `$all`

L'opérateur `$all` sélectionne les documents où un champ de type tableau contient **tous** les éléments spécifiés, **peu importe l'ordre**.

### Syntaxe

```javascript
{ champ: { $all: [valeur1, valeur2, ...] } }
```

### Différence avec la Recherche Simple

```javascript
// Documents de la collection
{ name: "Alice", tags: ["mongodb", "database", "nosql"] }
{ name: "Bob", tags: ["mongodb", "nosql"] }
{ name: "Charlie", tags: ["database", "sql"] }

// Avec $in : au moins UN élément présent
db.articles.find({
    tags: { $in: ["mongodb", "database"] }
})
// Retourne Alice, Bob ET Charlie (car au moins un tag correspond)

// Avec $all : TOUS les éléments doivent être présents
db.articles.find({
    tags: { $all: ["mongodb", "database"] }
})
// Retourne SEULEMENT Alice (car elle a les deux tags)
```

### Exemples de Base

#### Recherche avec Tous les Éléments

```javascript
// Trouver les utilisateurs ayant TOUS ces hobbies
db.users.find({
    hobbies: { $all: ["reading", "swimming"] }
})

// Trouver les produits avec TOUS ces tags
db.products.find({
    tags: { $all: ["wireless", "bluetooth"] }
})

// Trouver les articles avec TOUS ces mots-clés
db.articles.find({
    keywords: { $all: ["mongodb", "tutorial", "beginner"] }
})
```

#### L'Ordre n'a Pas d'Importance

```javascript
// Documents
{ name: "Product A", tags: ["red", "large", "cotton"] }
{ name: "Product B", tags: ["large", "cotton"] }

// Ces deux requêtes sont équivalentes
db.products.find({ tags: { $all: ["cotton", "large"] } })
db.products.find({ tags: { $all: ["large", "cotton"] } })
// Les deux retournent Product A (Product B n'a que 2 des tags)
```

#### Éléments Supplémentaires Autorisés

```javascript
// Document
{ name: "Alice", skills: ["JavaScript", "Python", "MongoDB", "React"] }

// Cette requête correspond car le tableau contient au moins les éléments spécifiés
db.developers.find({
    skills: { $all: ["JavaScript", "MongoDB"] }
})
// Retourne Alice (même si elle a aussi Python et React)
```

### Cas d'Usage Pratiques

#### E-commerce : Filtrage Multi-critères

```javascript
// Produits ayant TOUS ces attributs
db.products.find({
    features: { $all: ["waterproof", "bluetooth", "wireless"] }
})

// Vêtements disponibles en TOUTES ces tailles
db.clothing.find({
    availableSizes: { $all: ["M", "L", "XL"] }
})

// Produits avec TOUTES ces certifications
db.products.find({
    certifications: { $all: ["ISO9001", "CE", "FCC"] }
})
```

#### Gestion de Compétences

```javascript
// Développeurs ayant TOUTES les compétences requises
db.developers.find({
    skills: { $all: ["JavaScript", "React", "Node.js"] }
})

// Candidats parlant TOUTES ces langues
db.candidates.find({
    languages: { $all: ["English", "French", "Spanish"] }
})
```

#### Gestion de Contenus

```javascript
// Articles couvrant TOUS ces sujets
db.articles.find({
    topics: { $all: ["MongoDB", "Performance", "Indexing"] }
})

// Vidéos avec TOUS ces tags
db.videos.find({
    tags: { $all: ["tutorial", "beginner", "mongodb"] }
})
```

#### Gestion de Permissions

```javascript
// Utilisateurs ayant TOUS ces rôles
db.users.find({
    roles: { $all: ["admin", "moderator"] }
})

// Documents accessibles par TOUS ces départements
db.documents.find({
    allowedDepartments: { $all: ["HR", "Finance"] }
})
```

### Combinaison avec d'Autres Opérateurs

```javascript
// Produits avec certains tags ET prix < 100
db.products.find({
    tags: { $all: ["wireless", "bluetooth"] },
    price: { $lt: 100 }
})

// Utilisateurs avec compétences ET expérience suffisante
db.developers.find({
    skills: { $all: ["JavaScript", "MongoDB"] },
    experience: { $gte: 3 }
})

// Articles avec tags ET publiés récemment
db.articles.find({
    tags: { $all: ["mongodb", "tutorial"] },
    publishedDate: { $gte: ISODate("2024-01-01") }
})
```

---

## L'Opérateur `$elemMatch`

L'opérateur `$elemMatch` sélectionne les documents contenant un champ de type tableau avec **au moins un élément** qui satisfait **toutes les conditions** spécifiées.

### Syntaxe

```javascript
{ champ: { $elemMatch: { condition1, condition2, ... } } }
```

### Pourquoi `$elemMatch` ?

Sans `$elemMatch`, les conditions multiples sur un tableau peuvent donner des résultats inattendus :

```javascript
// Documents
{
    name: "Product A",
    reviews: [
        { rating: 5, verified: true },
        { rating: 2, verified: false }
    ]
}
{
    name: "Product B",
    reviews: [
        { rating: 4, verified: true },
        { rating: 5, verified: true }
    ]
}

// ❌ Sans $elemMatch : conditions séparées
db.products.find({
    "reviews.rating": { $gte: 4 },
    "reviews.verified": true
})
// Retourne Product A ET Product B
// Car Product A a un review avec rating >= 4 (5) ET un review verified (même si ce n'est pas le même)

// ✅ Avec $elemMatch : même élément doit satisfaire toutes les conditions
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})
// Retourne SEULEMENT Product B
// Car seul Product B a un review qui est à la fois >= 4 ET verified
```

### Exemples de Base

#### Tableaux d'Objets

```javascript
// Trouver les produits avec au moins un review vérifié et bien noté
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})

// Trouver les commandes avec au moins un article cher et en quantité importante
db.orders.find({
    items: {
        $elemMatch: {
            price: { $gte: 100 },
            quantity: { $gte: 5 }
        }
    }
})

// Trouver les utilisateurs avec au moins une adresse en France
db.users.find({
    addresses: {
        $elemMatch: {
            country: "France",
            type: "shipping"
        }
    }
})
```

#### Tableaux de Valeurs Simples

`$elemMatch` fonctionne aussi avec des tableaux de valeurs simples, bien que ce soit moins courant :

```javascript
// Documents
{ name: "Alice", scores: [85, 92, 78, 95] }
{ name: "Bob", scores: [65, 70, 68, 72] }

// Trouver les étudiants avec au moins un score entre 90 et 100
db.students.find({
    scores: {
        $elemMatch: {
            $gte: 90,
            $lte: 100
        }
    }
})
// Retourne Alice (a 92 et 95)
```

### Cas d'Usage Pratiques

#### E-commerce : Analyse de Reviews

```javascript
// Produits avec au moins un review récent et bien noté
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            date: { $gte: ISODate("2024-01-01") },
            verified: true
        }
    }
})

// Produits avec au moins un review négatif d'un acheteur vérifié
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $lte: 2 },
            verified: true,
            helpful: { $gte: 5 }
        }
    }
})
```

#### Gestion de Commandes

```javascript
// Commandes avec au moins un article en rupture de stock
db.orders.find({
    items: {
        $elemMatch: {
            status: "out_of_stock",
            priority: "high"
        }
    }
})

// Commandes avec au moins un article cher et en promo
db.orders.find({
    items: {
        $elemMatch: {
            price: { $gte: 100 },
            discount: { $gte: 20 }
        }
    }
})
```

#### Gestion de Projets

```javascript
// Projets avec au moins une tâche urgente et non assignée
db.projects.find({
    tasks: {
        $elemMatch: {
            priority: "urgent",
            status: "unassigned",
            dueDate: { $lte: new Date() }
        }
    }
})

// Projets avec au moins un membre senior et disponible
db.projects.find({
    team: {
        $elemMatch: {
            role: "senior",
            available: true,
            skills: { $in: ["MongoDB", "Node.js"] }
        }
    }
})
```

#### Gestion d'Événements

```javascript
// Événements avec au moins une session le week-end et en français
db.events.find({
    sessions: {
        $elemMatch: {
            dayOfWeek: { $in: ["Saturday", "Sunday"] },
            language: "French",
            seatsAvailable: { $gt: 10 }
        }
    }
})

// Conférences avec au moins un speaker expert et disponible
db.conferences.find({
    speakers: {
        $elemMatch: {
            level: "expert",
            available: true,
            topics: { $in: ["MongoDB", "Database Design"] }
        }
    }
})
```

### `$elemMatch` dans les Projections

`$elemMatch` peut aussi être utilisé dans les **projections** pour ne retourner que les éléments correspondants :

```javascript
// Retourner seulement les reviews vérifiées et bien notées
db.products.find(
    { productId: "PROD-123" },
    {
        name: 1,
        reviews: {
            $elemMatch: {
                rating: { $gte: 4 },
                verified: true
            }
        }
    }
)

// Résultat : seuls les reviews correspondants sont inclus dans le tableau reviews
```

### Combinaison avec d'Autres Opérateurs

```javascript
// Produits avec reviews positifs ET en stock
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    },
    stock: { $gt: 0 }
})

// Commandes avec items chers ET client VIP
db.orders.find({
    items: {
        $elemMatch: {
            price: { $gte: 100 },
            quantity: { $gte: 2 }
        }
    },
    customerType: "VIP"
})
```

---

## L'Opérateur `$size`

L'opérateur `$size` sélectionne les documents où un champ de type tableau a exactement le **nombre d'éléments** spécifié.

### Syntaxe

```javascript
{ champ: { $size: nombre } }
```

### Exemples de Base

```javascript
// Trouver les utilisateurs avec exactement 3 hobbies
db.users.find({ hobbies: { $size: 3 } })

// Trouver les produits avec exactement 5 images
db.products.find({ images: { $size: 5 } })

// Trouver les commandes avec exactement 1 article
db.orders.find({ items: { $size: 1 } })

// Trouver les listes vides
db.tasks.find({ subtasks: { $size: 0 } })
```

### Limitation Importante

**`$size` n'accepte PAS d'opérateurs de comparaison** comme `$gt`, `$lt`, etc.

```javascript
// ❌ INCORRECT : ne fonctionne pas
db.users.find({ hobbies: { $size: { $gte: 3 } } })

// ✅ CORRECT : utiliser $expr pour les comparaisons
db.users.find({
    $expr: { $gte: [{ $size: "$hobbies" }, 3] }
})
```

### Utilisation de `$expr` pour les Plages

Pour rechercher des tableaux avec une taille dans une plage, utilisez `$expr` :

```javascript
// Tableaux avec au moins 3 éléments
db.users.find({
    $expr: { $gte: [{ $size: "$hobbies" }, 3] }
})

// Tableaux avec moins de 5 éléments
db.users.find({
    $expr: { $lt: [{ $size: "$tags" }, 5] }
})

// Tableaux avec entre 2 et 5 éléments
db.products.find({
    $expr: {
        $and: [
            { $gte: [{ $size: "$images" }, 2] },
            { $lte: [{ $size: "$images" }, 5] }
        ]
    }
})

// Tableaux non vides
db.documents.find({
    $expr: { $gt: [{ $size: "$attachments" }, 0] }
})
```

### Cas d'Usage Pratiques

#### Validation de Données

```javascript
// Produits avec exactement le nombre requis d'images
db.products.find({
    images: { $size: 4 }
})

// Formulaires avec toutes les réponses (10 questions)
db.surveys.find({
    answers: { $size: 10 }
})

// Commandes avec un seul article (commandes simples)
db.orders.find({
    items: { $size: 1 }
})
```

#### Détection d'Anomalies

```javascript
// Utilisateurs sans aucun hobby (potentiel problème)
db.users.find({
    hobbies: { $size: 0 }
})

// Produits sans images (données incomplètes)
db.products.find({
    images: { $size: 0 }
})

// Commandes vides (erreur potentielle)
db.orders.find({
    items: { $size: 0 }
})
```

#### Analyse de Données

```javascript
// Articles avec exactement 5 tags (bien catégorisés)
db.articles.find({
    tags: { $size: 5 }
})

// Projets avec exactement 3 membres (petites équipes)
db.projects.find({
    team: { $size: 3 }
})

// Paniers abandonnés avec un seul article
db.carts.find({
    status: "abandoned",
    items: { $size: 1 }
})
```

#### Avec `$expr` pour Logique Complexe

```javascript
// Produits populaires (beaucoup de reviews)
db.products.find({
    $expr: { $gte: [{ $size: "$reviews" }, 50] }
})

// Utilisateurs actifs (au moins 10 activités)
db.users.find({
    $expr: { $gte: [{ $size: "$activities" }, 10] }
})

// Projets petits à moyens (2 à 10 membres)
db.projects.find({
    $expr: {
        $and: [
            { $gte: [{ $size: "$team" }, 2] },
            { $lte: [{ $size: "$team" }, 10] }
        ]
    }
})

// Articles bien documentés (plus de 3 images)
db.articles.find({
    $expr: { $gt: [{ $size: "$images" }, 3] }
})
```

### Alternative : Stocker la Taille

Pour de meilleures performances sur de grandes collections, envisagez de stocker la taille dans un champ séparé :

```javascript
// Structure du document
{
    name: "Product A",
    images: ["img1.jpg", "img2.jpg", "img3.jpg"],
    imageCount: 3  // Stocké explicitement
}

// Requête plus rapide
db.products.find({ imageCount: { $gte: 3 } })

// Au lieu de
db.products.find({
    $expr: { $gte: [{ $size: "$images" }, 3] }
})
```

---

## Combinaison des Opérateurs de Tableaux

Les opérateurs de tableaux peuvent être combinés entre eux et avec d'autres opérateurs pour des requêtes sophistiquées.

### `$all` + `$size`

```javascript
// Produits avec exactement 3 tags et contenant tous ces tags spécifiques
db.products.find({
    tags: { $all: ["wireless", "bluetooth", "portable"] },
    $expr: { $eq: [{ $size: "$tags" }, 3] }
})
// Retourne seulement les produits ayant exactement ces 3 tags, pas plus

// Articles avec au moins ces 2 catégories et exactement 2 catégories
db.articles.find({
    categories: { $all: ["MongoDB", "Tutorial"] },
    $expr: { $eq: [{ $size: "$categories" }, 2] }
})
```

### `$elemMatch` + `$size`

```javascript
// Produits avec au moins 5 reviews ET au moins un review récent bien noté
db.products.find({
    $expr: { $gte: [{ $size: "$reviews" }, 5] },
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            date: { $gte: ISODate("2024-01-01") }
        }
    }
})

// Commandes avec exactement 3 articles ET au moins un article cher
db.orders.find({
    items: { $size: 3 },
    items: {
        $elemMatch: {
            price: { $gte: 100 }
        }
    }
})
```

### `$all` + `$elemMatch`

```javascript
// Produits avec certains tags ET reviews positifs
db.products.find({
    tags: { $all: ["wireless", "premium"] },
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})

// Projets avec compétences requises ET membres seniors
db.projects.find({
    requiredSkills: { $all: ["MongoDB", "Node.js"] },
    team: {
        $elemMatch: {
            level: "senior",
            available: true
        }
    }
})
```

### Exemple Complexe

```javascript
// Recherche sophistiquée de produits
db.products.find({
    // Doit avoir TOUS ces tags
    tags: { $all: ["wireless", "bluetooth", "premium"] },

    // Doit avoir au moins 10 reviews
    $expr: { $gte: [{ $size: "$reviews" }, 10] },

    // Au moins un review vérifié et bien noté
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true,
            date: { $gte: ISODate("2024-01-01") }
        }
    },

    // Prix dans une plage
    price: { $gte: 50, $lte: 200 },

    // En stock
    stock: { $gt: 0 }
})
```

---

## Tableaux Imbriqués

Les opérateurs de tableaux fonctionnent également avec des tableaux imbriqués.

### Structure de Document Imbriqué

```javascript
{
    name: "Online Store",
    departments: [
        {
            name: "Electronics",
            products: [
                { name: "Laptop", price: 999 },
                { name: "Mouse", price: 29 }
            ]
        },
        {
            name: "Books",
            products: [
                { name: "MongoDB Guide", price: 45 }
            ]
        }
    ]
}
```

### Requêtes sur Tableaux Imbriqués

```javascript
// Trouver les magasins avec un département ayant au moins un produit cher
db.stores.find({
    "departments.products": {
        $elemMatch: {
            price: { $gte: 500 }
        }
    }
})

// Département avec exactement 2 produits
db.stores.find({
    departments: {
        $elemMatch: {
            $expr: { $eq: [{ $size: "$products" }, 2] }
        }
    }
})
```

---

## Comparaison avec SQL

Les tableaux dans MongoDB n'ont pas d'équivalent direct en SQL. Voici des approximations conceptuelles :

| Concept | SQL (approximation) | MongoDB |
|---------|---------------------|---------|
| Vérifier contenu de liste | `WHERE tags LIKE '%tag1%' AND tags LIKE '%tag2%'` | `{ tags: { $all: ["tag1", "tag2"] } }` |
| Élément avec conditions | Jointure avec sous-requête complexe | `{ items: { $elemMatch: { ... } } }` |
| Compter éléments | `SELECT COUNT(*) FROM table_liee` | `{ items: { $size: 3 } }` |

**Note** : En SQL, ces opérations nécessiteraient généralement des tables séparées et des jointures.

---

## Bonnes Pratiques

### 1. Utiliser `$all` pour Filtres Multi-tags

```javascript
// ✅ Bon : pour recherche ET logique sur tags
db.products.find({
    tags: { $all: ["wireless", "bluetooth"] }
})

// ⚠️ Moins clair : avec $and explicite
db.products.find({
    $and: [
        { tags: "wireless" },
        { tags: "bluetooth" }
    ]
})
```

### 2. Préférer `$elemMatch` pour Objets Complexes

```javascript
// ✅ Bon : garantit que les conditions s'appliquent au même élément
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})

// ❌ Incorrect : les conditions peuvent s'appliquer à différents éléments
db.products.find({
    "reviews.rating": { $gte: 4 },
    "reviews.verified": true
})
```

### 3. Stocker la Taille pour les Requêtes Fréquentes

```javascript
// Si vous recherchez souvent par taille de tableau
{
    name: "Product A",
    images: ["img1.jpg", "img2.jpg"],
    imageCount: 2  // Maintenu à jour par l'application
}

// Requête optimisée
db.products.find({ imageCount: { $gte: 3 } })
```

### 4. Indexer les Champs de Tableaux

```javascript
// Créer un index multikey pour les recherches dans tableaux
db.products.createIndex({ tags: 1 })

// Les requêtes utilisent l'index efficacement
db.products.find({ tags: "wireless" })
db.products.find({ tags: { $all: ["wireless", "bluetooth"] } })
```

### 5. Limiter la Taille des Tableaux

Les très grands tableaux peuvent impacter les performances :

```javascript
// ⚠️ Éviter les tableaux avec des milliers d'éléments
// Envisager de restructurer les données ou d'utiliser des références
```

### 6. Utiliser `$expr` pour Comparaisons de Taille

```javascript
// ✅ Bon : pour recherches par plage de taille
db.users.find({
    $expr: { $gte: [{ $size: "$hobbies" }, 3] }
})

// ❌ Ne fonctionne pas
db.users.find({ hobbies: { $size: { $gte: 3 } } })
```

---

## Pièges Courants à Éviter

### 1. Confusion entre `$in` et `$all`

```javascript
// ❌ Erreur : utiliser $in quand on veut $all
db.products.find({
    tags: { $in: ["wireless", "bluetooth"] }
})
// Retourne les produits avec "wireless" OU "bluetooth"

// ✅ Correct : utiliser $all pour ET logique
db.products.find({
    tags: { $all: ["wireless", "bluetooth"] }
})
// Retourne seulement les produits avec les DEUX tags
```

### 2. Oublier `$elemMatch` pour Objets

```javascript
// ❌ Incorrect : conditions peuvent s'appliquer à différents éléments
db.products.find({
    "reviews.rating": { $gte: 4 },
    "reviews.verified": true
})

// ✅ Correct : même élément doit satisfaire toutes les conditions
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})
```

### 3. Utiliser `$size` avec Opérateurs de Comparaison

```javascript
// ❌ Ne fonctionne pas
db.users.find({
    hobbies: { $size: { $gte: 3 } }
})

// ✅ Utiliser $expr
db.users.find({
    $expr: { $gte: [{ $size: "$hobbies" }, 3] }
})
```

### 4. Performance avec `$size` et `$expr`

```javascript
// ⚠️ Peut être lent sur de grandes collections
db.products.find({
    $expr: { $gte: [{ $size: "$reviews" }, 100] }
})

// ✅ Alternative : stocker la taille
db.products.find({ reviewCount: { $gte: 100 } })
```

### 5. `$all` avec Tableau Vide

```javascript
// ⚠️ Attention : $all avec tableau vide correspond à TOUS les documents
db.users.find({ tags: { $all: [] } })
// Retourne tous les documents (généralement pas l'intention)
```

---

## Performance et Optimisation

### Impact des Opérateurs

| Opérateur | Performance | Peut Utiliser Index | Notes |
|-----------|-------------|---------------------|-------|
| `$all` | Bonne | Oui (multikey) | Efficace avec index multikey |
| `$elemMatch` | Moyenne | Partiel | Peut utiliser index pour certaines conditions |
| `$size` | Moyenne | Non | Nécessite un scan des documents |
| `$size` avec `$expr` | Faible | Non | Éviter sur grandes collections |

### Optimisation avec Index Multikey

```javascript
// Créer un index multikey
db.products.createIndex({ tags: 1 })

// Ces requêtes utilisent l'index efficacement
db.products.find({ tags: "wireless" })
db.products.find({ tags: { $all: ["wireless", "bluetooth"] } })
db.products.find({ tags: { $in: ["wireless", "bluetooth"] } })
```

### Index Composés avec Tableaux

```javascript
// Index composé
db.products.createIndex({ category: 1, tags: 1, price: 1 })

// Requête optimisée
db.products.find({
    category: "Electronics",
    tags: { $all: ["wireless", "bluetooth"] },
    price: { $lt: 100 }
})
```

### Optimisation de `$elemMatch`

```javascript
// Index sur champs imbriqués
db.products.createIndex({ "reviews.rating": 1, "reviews.verified": 1 })

// La requête peut utiliser l'index
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})
```

### Vérification avec `explain()`

```javascript
// Analyser $all
db.products.find({
    tags: { $all: ["wireless", "bluetooth"] }
}).explain("executionStats")

// Analyser $elemMatch
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
}).explain("executionStats")

// Analyser $size avec $expr
db.products.find({
    $expr: { $gte: [{ $size: "$reviews" }, 10] }
}).explain("executionStats")
```

---

## Points Clés à Retenir

✅ **`$all`** vérifie qu'un tableau contient **tous** les éléments spécifiés (ET logique)

✅ L'**ordre n'a pas d'importance** avec `$all`

✅ **`$elemMatch`** garantit que **le même élément** satisfait toutes les conditions

✅ Utilisez `$elemMatch` pour des **tableaux d'objets** avec conditions multiples

✅ **`$size`** vérifie la taille exacte d'un tableau

✅ `$size` **n'accepte pas** les opérateurs de comparaison directement

✅ Utilisez **`$expr`** avec `$size` pour des comparaisons de taille (>=, <=, etc.)

✅ Les **index multikey** améliorent les performances des requêtes sur tableaux

✅ **Stocker la taille** dans un champ séparé pour de meilleures performances

✅ Ne confondez pas **`$in`** (OU) et **`$all`** (ET)

---

## Résumé

Dans ce chapitre, vous avez appris :

- Comment utiliser `$all` pour vérifier qu'un tableau contient tous les éléments requis
- Comment utiliser `$elemMatch` pour appliquer plusieurs conditions au même élément
- Comment utiliser `$size` pour filtrer par la longueur des tableaux
- La différence cruciale entre `$in` (OU) et `$all` (ET)
- L'importance de `$elemMatch` pour les tableaux d'objets
- Comment combiner `$size` avec `$expr` pour des comparaisons de taille
- Les bonnes pratiques d'indexation et d'optimisation
- Les pièges courants à éviter

Ces opérateurs sont essentiels pour exploiter pleinement la puissance des tableaux dans MongoDB. Ils vous permettent d'interroger efficacement des structures de données complexes qui nécessiteraient des jointures multiples dans une base de données relationnelle.

Dans le prochain chapitre, nous explorerons les **projections** qui vous permettront de contrôler précisément quels champs sont retournés dans les résultats de vos requêtes.

---


⏭️ [Projections : Sélection des champs](/03-requetes-et-filtres/07-projections.md)
