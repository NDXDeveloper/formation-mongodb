🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.11 Requêtes sur Tableaux

## Introduction

Les tableaux sont une structure de données fondamentale dans MongoDB. Contrairement aux bases de données relationnelles où stocker plusieurs valeurs nécessite des tables séparées et des jointures, MongoDB permet de stocker directement des **tableaux** dans les documents.

Les tableaux dans MongoDB peuvent contenir :
- **Valeurs simples** : nombres, chaînes, dates, etc.
- **Objets** : structures complexes imbriquées
- **Tableaux imbriqués** : tableaux dans des tableaux
- **Types mixtes** : différents types dans le même tableau

Cette flexibilité offre de grandes possibilités mais nécessite également des techniques de requêtage spécifiques que nous allons explorer dans ce chapitre.

### Exemples de Tableaux dans MongoDB

```javascript
// Tableau de chaînes
{
    name: "Alice",
    hobbies: ["reading", "swimming", "coding"]
}

// Tableau de nombres
{
    name: "Bob",
    scores: [85, 92, 78, 95, 88]
}

// Tableau d'objets
{
    name: "Charlie",
    addresses: [
        { type: "home", city: "Paris" },
        { type: "work", city: "Lyon" }
    ]
}

// Tableau de dates
{
    name: "David",
    loginDates: [
        ISODate("2024-01-15"),
        ISODate("2024-02-20"),
        ISODate("2024-03-10")
    ]
}

// Tableau mixte (possible mais déconseillé)
{
    name: "Eve",
    data: [1, "two", true, { key: "value" }]
}
```

---

## Requêtes de Base sur Tableaux

### Vérifier si un Tableau Contient une Valeur

MongoDB vérifie automatiquement dans les tableaux lors des recherches par égalité.

```javascript
// Documents
{ name: "Alice", tags: ["mongodb", "database", "nosql"] }
{ name: "Bob", tags: ["javascript", "nodejs"] }
{ name: "Charlie", tags: ["python", "django"] }

// Trouver les documents où tags contient "mongodb"
db.articles.find({ tags: "mongodb" })
// Retourne Alice

// Trouver les documents où tags contient "javascript"
db.articles.find({ tags: "javascript" })
// Retourne Bob
```

**Important** : MongoDB vérifie si la valeur existe **n'importe où** dans le tableau.

### Recherche avec Plusieurs Valeurs Possibles (`$in`)

```javascript
// Trouver les articles avec tag "mongodb" OU "python"
db.articles.find({
    tags: { $in: ["mongodb", "python"] }
})
// Retourne Alice ET Charlie

// Trouver les produits avec couleur rouge OU bleu
db.products.find({
    colors: { $in: ["red", "blue"] }
})
```

**Rappel** : `$in` vérifie si **au moins une** valeur du tableau `$in` existe dans le tableau du document.

### Exclusion de Valeurs (`$nin`)

```javascript
// Trouver les articles qui ne contiennent NI "deprecated" NI "outdated"
db.articles.find({
    tags: { $nin: ["deprecated", "outdated"] }
})

// Produits sans couleurs rouge ou noir
db.products.find({
    colors: { $nin: ["red", "black"] }
})
```

---

## L'Opérateur `$all` - Toutes les Valeurs Doivent Être Présentes

L'opérateur `$all` sélectionne les documents où un tableau contient **tous** les éléments spécifiés.

### Syntaxe

```javascript
{ champ: { $all: [valeur1, valeur2, ...] } }
```

### Exemples

```javascript
// Documents
{ name: "Product A", features: ["wireless", "bluetooth", "waterproof"] }
{ name: "Product B", features: ["wireless", "bluetooth"] }
{ name: "Product C", features: ["bluetooth", "waterproof"] }

// Trouver les produits ayant TOUS ces features
db.products.find({
    features: { $all: ["wireless", "bluetooth"] }
})
// Retourne Product A ET Product B

// Trouver les produits ayant les 3 features
db.products.find({
    features: { $all: ["wireless", "bluetooth", "waterproof"] }
})
// Retourne seulement Product A
```

### Différence avec `$in`

```javascript
// $in : AU MOINS UNE valeur présente (OU logique)
db.products.find({
    features: { $in: ["wireless", "bluetooth"] }
})
// Retourne A, B, C (car tous ont au moins une des features)

// $all : TOUTES les valeurs présentes (ET logique)
db.products.find({
    features: { $all: ["wireless", "bluetooth"] }
})
// Retourne A, B (qui ont les deux features)
```

### L'Ordre n'a Pas d'Importance

```javascript
// Ces deux requêtes sont équivalentes
db.products.find({ features: { $all: ["wireless", "bluetooth"] } })
db.products.find({ features: { $all: ["bluetooth", "wireless"] } })
```

---

## L'Opérateur `$size` - Taille du Tableau

L'opérateur `$size` sélectionne les documents où un tableau a exactement le nombre d'éléments spécifié.

### Syntaxe

```javascript
{ champ: { $size: nombre } }
```

### Exemples

```javascript
// Trouver les utilisateurs avec exactement 3 hobbies
db.users.find({ hobbies: { $size: 3 } })

// Trouver les produits avec exactement 5 images
db.products.find({ images: { $size: 5 } })

// Trouver les tableaux vides
db.documents.find({ tags: { $size: 0 } })

// Trouver les listes avec un seul élément
db.lists.find({ items: { $size: 1 } })
```

### Limitation : Pas de Comparaisons

`$size` n'accepte **pas** les opérateurs de comparaison (`$gt`, `$lt`, etc.).

```javascript
// ❌ Ne fonctionne PAS
db.users.find({ hobbies: { $size: { $gte: 3 } } })

// ✅ Solution : utiliser $expr
db.users.find({
    $expr: { $gte: [{ $size: "$hobbies" }, 3] }
})
```

### Plages de Taille avec `$expr`

```javascript
// Au moins 3 éléments
db.users.find({
    $expr: { $gte: [{ $size: "$tags" }, 3] }
})

// Moins de 10 éléments
db.products.find({
    $expr: { $lt: [{ $size: "$images" }, 10] }
})

// Entre 5 et 10 éléments
db.articles.find({
    $expr: {
        $and: [
            { $gte: [{ $size: "$comments" }, 5] },
            { $lte: [{ $size: "$comments" }, 10] }
        ]
    }
})

// Tableaux non vides
db.documents.find({
    $expr: { $gt: [{ $size: "$attachments" }, 0] }
})
```

---

## L'Opérateur `$elemMatch` - Conditions sur le Même Élément

Pour les tableaux d'objets, `$elemMatch` garantit que **toutes les conditions** s'appliquent au **même élément** du tableau.

### Problème Sans `$elemMatch`

```javascript
// Documents
{
    name: "Product A",
    reviews: [
        { rating: 5, verified: true },
        { rating: 2, verified: false }
    ]
}

// ❌ Sans $elemMatch : conditions peuvent s'appliquer à différents éléments
db.products.find({
    "reviews.rating": { $gte: 4 },
    "reviews.verified": true
})
// Correspond à Product A car :
// - Un review a rating >= 4 (le premier)
// - Un review est verified (le premier)
// Même si c'est le même élément, ce n'est pas garanti !
```

### Solution avec `$elemMatch`

```javascript
// ✅ Avec $elemMatch : même élément doit satisfaire toutes les conditions
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})
// Ne retourne que les produits ayant AU MOINS UN review vérifié ET bien noté
```

### Exemples Pratiques

```javascript
// Produits avec au moins un review récent et positif
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            date: { $gte: ISODate("2024-01-01") },
            verified: true
        }
    }
})

// Utilisateurs avec au moins une adresse en France de type "home"
db.users.find({
    addresses: {
        $elemMatch: {
            country: "France",
            type: "home"
        }
    }
})

// Commandes avec au moins un article cher et en stock
db.orders.find({
    items: {
        $elemMatch: {
            price: { $gte: 100 },
            inStock: true
        }
    }
})
```

### `$elemMatch` avec Tableaux de Valeurs Simples

Bien que moins courant, `$elemMatch` fonctionne aussi avec des tableaux de valeurs simples :

```javascript
// Documents
{ name: "Alice", scores: [85, 92, 78, 95] }
{ name: "Bob", scores: [65, 70, 68, 72] }

// Trouver les étudiants avec au moins un score entre 90 et 100
db.students.find({
    scores: {
        $elemMatch: { $gte: 90, $lte: 100 }
    }
})
// Retourne Alice (a 92 et 95)
```

---

## Requêtes sur Position dans un Tableau

### Accès par Index

Vous pouvez accéder à un élément spécifique d'un tableau par son index (commence à 0).

```javascript
// Documents
{ name: "Alice", scores: [85, 92, 78] }
{ name: "Bob", scores: [95, 88, 91] }

// Trouver où le premier score est supérieur à 90
db.students.find({ "scores.0": { $gt: 90 } })
// Retourne Bob (son premier score est 95)

// Trouver où le deuxième score est 88
db.students.find({ "scores.1": 88 })
// Retourne Bob
```

### Exemples Pratiques

```javascript
// Premier tag est "mongodb"
db.articles.find({ "tags.0": "mongodb" })

// Première adresse est de type "home"
db.users.find({ "addresses.0.type": "home" })

// Premier élément du tableau items avec prix > 100
db.orders.find({ "items.0.price": { $gt: 100 } })
```

---

## Requêtes sur Tableaux Imbriqués

MongoDB supporte les tableaux dans des tableaux.

### Structure Exemple

```javascript
{
    name: "Product A",
    variations: [
        {
            size: "M",
            colors: ["red", "blue", "green"]
        },
        {
            size: "L",
            colors: ["black", "white"]
        }
    ]
}
```

### Requêtes

```javascript
// Trouver les produits avec une variation incluant "red"
db.products.find({ "variations.colors": "red" })

// Avec $elemMatch pour garantir size ET colors dans même variation
db.products.find({
    variations: {
        $elemMatch: {
            size: "M",
            colors: "red"
        }
    }
})

// Variation avec size "L" ET au moins une couleur dans la liste
db.products.find({
    variations: {
        $elemMatch: {
            size: "L",
            colors: { $in: ["black", "white", "gray"] }
        }
    }
})
```

---

## L'Opérateur Positionnel `$` (pour les Mises à Jour)

L'opérateur `$` identifie un élément du tableau qui correspond à la condition de la requête.

### Mise à Jour du Premier Élément Correspondant

```javascript
// Document
{
    name: "Alice",
    scores: [
        { subject: "Math", grade: 85 },
        { subject: "Physics", grade: 90 },
        { subject: "Chemistry", grade: 78 }
    ]
}

// Mettre à jour le grade de Math
db.students.updateOne(
    {
        name: "Alice",
        "scores.subject": "Math"
    },
    {
        $set: { "scores.$.grade": 95 }
    }
)
// Résultat : le grade de Math passe de 85 à 95
```

**Important** : `$` met à jour seulement le **premier** élément correspondant.

### Exemples

```javascript
// Mettre à jour le statut d'une commande spécifique
db.users.updateOne(
    {
        email: "alice@example.com",
        "orders.orderId": "ORD-001"
    },
    {
        $set: { "orders.$.status": "shipped" }
    }
)

// Augmenter le prix d'un produit spécifique dans le panier
db.carts.updateOne(
    {
        userId: 123,
        "items.productId": "PROD-456"
    },
    {
        $inc: { "items.$.quantity": 1 }
    }
)

// Marquer un tag comme featured
db.articles.updateOne(
    {
        _id: ObjectId("..."),
        tags: "mongodb"
    },
    {
        $set: { "tags.$": "mongodb-featured" }
    }
)
```

---

## L'Opérateur `$[]` - Tous les Éléments du Tableau

L'opérateur `$[]` met à jour **tous les éléments** d'un tableau.

### Syntaxe

```javascript
{ $set: { "tableau.$[].champ": valeur } }
```

### Exemples

```javascript
// Document
{
    name: "Product A",
    reviews: [
        { author: "Alice", rating: 5, verified: false },
        { author: "Bob", rating: 4, verified: false }
    ]
}

// Marquer tous les reviews comme vérifiés
db.products.updateOne(
    { name: "Product A" },
    {
        $set: { "reviews.$[].verified": true }
    }
)
// Résultat : tous les reviews ont verified: true
```

### Autres Exemples

```javascript
// Augmenter tous les scores de 10%
db.students.updateOne(
    { name: "Alice" },
    {
        $mul: { "scores.$[]": 1.1 }
    }
)

// Appliquer une réduction sur tous les items
db.carts.updateOne(
    { userId: 123 },
    {
        $set: { "items.$[].discount": 10 }
    }
)

// Mettre à jour le statut de toutes les tâches
db.projects.updateOne(
    { projectId: "PROJ-001" },
    {
        $set: { "tasks.$[].status": "completed" }
    }
)
```

---

## L'Opérateur `$[<identifier>]` - Éléments Filtrés

L'opérateur `$[<identifier>]` met à jour **les éléments qui correspondent à un filtre** spécifié.

### Syntaxe

```javascript
db.collection.updateOne(
    { query },
    { $set: { "tableau.$[identifier].champ": valeur } },
    { arrayFilters: [{ "identifier.condition": valeur }] }
)
```

### Exemples

```javascript
// Document
{
    name: "Product A",
    reviews: [
        { author: "Alice", rating: 5, helpful: 10 },
        { author: "Bob", rating: 2, helpful: 3 },
        { author: "Charlie", rating: 4, helpful: 8 }
    ]
}

// Marquer comme "featured" seulement les reviews bien notés
db.products.updateOne(
    { name: "Product A" },
    {
        $set: { "reviews.$[elem].featured": true }
    },
    {
        arrayFilters: [{ "elem.rating": { $gte: 4 } }]
    }
)
// Résultat : Alice et Charlie ont featured: true, pas Bob
```

### Cas d'Usage Pratiques

```javascript
// Marquer comme "verified" seulement les reviews avec 5+ likes
db.products.updateOne(
    { _id: ObjectId("...") },
    {
        $set: { "reviews.$[elem].verified": true }
    },
    {
        arrayFilters: [{ "elem.helpful": { $gte: 5 } }]
    }
)

// Appliquer une réduction seulement sur les articles chers
db.carts.updateOne(
    { userId: 123 },
    {
        $set: { "items.$[item].discount": 15 }
    },
    {
        arrayFilters: [{ "item.price": { $gte: 100 } }]
    }
)

// Augmenter le salaire des employés seniors
db.departments.updateOne(
    { name: "IT" },
    {
        $inc: { "employees.$[emp].salary": 5000 }
    },
    {
        arrayFilters: [{ "emp.level": "senior" }]
    }
)

// Marquer les tâches urgentes comme prioritaires
db.projects.updateOne(
    { _id: ObjectId("...") },
    {
        $set: { "tasks.$[task].priority": "high" }
    },
    {
        arrayFilters: [{
            "task.dueDate": { $lte: new Date() },
            "task.status": "pending"
        }]
    }
)
```

---

## Opérations de Modification de Tableaux

### Ajouter des Éléments

#### `$push` - Ajouter un Élément

```javascript
// Ajouter un nouveau tag
db.articles.updateOne(
    { _id: ObjectId("...") },
    { $push: { tags: "new-tag" } }
)

// Ajouter un nouvel objet à un tableau
db.users.updateOne(
    { email: "alice@example.com" },
    {
        $push: {
            orders: {
                orderId: "ORD-003",
                date: new Date(),
                amount: 150.00
            }
        }
    }
)
```

#### `$push` avec `$each` - Ajouter Plusieurs Éléments

```javascript
// Ajouter plusieurs tags
db.articles.updateOne(
    { _id: ObjectId("...") },
    {
        $push: {
            tags: {
                $each: ["mongodb", "database", "nosql"]
            }
        }
    }
)

// Ajouter plusieurs scores
db.students.updateOne(
    { name: "Alice" },
    {
        $push: {
            scores: {
                $each: [95, 88, 92]
            }
        }
    }
)
```

#### `$push` avec `$position` - Insérer à une Position

```javascript
// Insérer au début du tableau (position 0)
db.articles.updateOne(
    { _id: ObjectId("...") },
    {
        $push: {
            tags: {
                $each: ["featured"],
                $position: 0
            }
        }
    }
)

// Insérer à la position 2
db.lists.updateOne(
    { name: "Todo List" },
    {
        $push: {
            items: {
                $each: ["New Task"],
                $position: 2
            }
        }
    }
)
```

#### `$push` avec `$sort` et `$slice`

```javascript
// Ajouter et garder seulement les 5 plus hauts scores
db.students.updateOne(
    { name: "Alice" },
    {
        $push: {
            scores: {
                $each: [95, 88],
                $sort: -1,      // Trier décroissant
                $slice: 5       // Garder seulement les 5 premiers
            }
        }
    }
)

// Ajouter des reviews et garder les 10 plus récentes
db.products.updateOne(
    { _id: ObjectId("...") },
    {
        $push: {
            reviews: {
                $each: [{ rating: 5, date: new Date() }],
                $sort: { date: -1 },
                $slice: 10
            }
        }
    }
)
```

#### `$addToSet` - Ajouter Seulement si Unique

```javascript
// Ajouter un tag seulement s'il n'existe pas déjà
db.articles.updateOne(
    { _id: ObjectId("...") },
    { $addToSet: { tags: "mongodb" } }
)

// Ajouter plusieurs valeurs uniques
db.articles.updateOne(
    { _id: ObjectId("...") },
    {
        $addToSet: {
            tags: {
                $each: ["mongodb", "database", "nosql"]
            }
        }
    }
)
```

### Supprimer des Éléments

#### `$pull` - Supprimer par Valeur

```javascript
// Supprimer un tag spécifique
db.articles.updateOne(
    { _id: ObjectId("...") },
    { $pull: { tags: "outdated" } }
)

// Supprimer tous les scores inférieurs à 70
db.students.updateOne(
    { name: "Alice" },
    { $pull: { scores: { $lt: 70 } } }
)

// Supprimer des objets correspondant à des critères
db.users.updateOne(
    { email: "alice@example.com" },
    {
        $pull: {
            orders: {
                status: "cancelled"
            }
        }
    }
)
```

#### `$pop` - Supprimer le Premier ou Dernier Élément

```javascript
// Supprimer le dernier élément
db.lists.updateOne(
    { name: "Todo List" },
    { $pop: { items: 1 } }
)

// Supprimer le premier élément
db.lists.updateOne(
    { name: "Todo List" },
    { $pop: { items: -1 } }
)
```

#### `$pullAll` - Supprimer Plusieurs Valeurs

```javascript
// Supprimer plusieurs tags
db.articles.updateOne(
    { _id: ObjectId("...") },
    {
        $pullAll: {
            tags: ["deprecated", "old", "outdated"]
        }
    }
)

// Supprimer plusieurs scores spécifiques
db.students.updateOne(
    { name: "Alice" },
    {
        $pullAll: {
            scores: [65, 70, 75]
        }
    }
)
```

---

## Projections sur Tableaux

### Limiter les Éléments avec `$slice`

```javascript
// Retourner les 3 premiers éléments
db.articles.find(
    {},
    {
        title: 1,
        comments: { $slice: 3 }
    }
)

// Retourner les 5 derniers éléments (nombre négatif)
db.articles.find(
    {},
    {
        title: 1,
        comments: { $slice: -5 }
    }
)

// Sauter 2 éléments et prendre les 3 suivants
db.articles.find(
    {},
    {
        title: 1,
        comments: { $slice: [2, 3] }
    }
)
```

### Projeter avec `$elemMatch`

```javascript
// Retourner seulement le premier review vérifié
db.products.find(
    { _id: ObjectId("...") },
    {
        name: 1,
        reviews: {
            $elemMatch: {
                verified: true,
                rating: { $gte: 4 }
            }
        }
    }
)
```

### Projeter des Champs Spécifiques dans un Tableau

```javascript
// Retourner seulement certains champs des objets du tableau
db.users.find(
    {},
    {
        name: 1,
        "orders.orderId": 1,
        "orders.amount": 1
    }
)
```

---

## Cas d'Usage Pratiques

### Cas 1 : E-commerce - Gestion de Panier

```javascript
// Structure
{
    userId: 123,
    cart: {
        items: [
            { productId: "PROD-001", name: "Laptop", price: 999, quantity: 1 },
            { productId: "PROD-002", name: "Mouse", price: 29, quantity: 2 }
        ],
        total: 1057
    }
}

// Ajouter un produit au panier
db.carts.updateOne(
    { userId: 123 },
    {
        $push: {
            "cart.items": {
                productId: "PROD-003",
                name: "Keyboard",
                price: 79,
                quantity: 1
            }
        }
    }
)

// Augmenter la quantité d'un produit existant
db.carts.updateOne(
    {
        userId: 123,
        "cart.items.productId": "PROD-001"
    },
    {
        $inc: { "cart.items.$.quantity": 1 }
    }
)

// Supprimer un produit du panier
db.carts.updateOne(
    { userId: 123 },
    {
        $pull: {
            "cart.items": { productId: "PROD-002" }
        }
    }
)

// Trouver les paniers avec des articles chers
db.carts.find({
    "cart.items": {
        $elemMatch: {
            price: { $gte: 500 }
        }
    }
})
```

### Cas 2 : Blog - Gestion de Commentaires

```javascript
// Structure
{
    title: "Introduction to MongoDB",
    author: "Alice",
    comments: [
        {
            commentId: "C001",
            author: "Bob",
            text: "Great article!",
            date: ISODate("2024-01-15"),
            likes: 5,
            replies: [
                {
                    author: "Alice",
                    text: "Thank you!",
                    date: ISODate("2024-01-16")
                }
            ]
        }
    ]
}

// Ajouter un commentaire
db.articles.updateOne(
    { _id: ObjectId("...") },
    {
        $push: {
            comments: {
                commentId: "C002",
                author: "Charlie",
                text: "Very helpful!",
                date: new Date(),
                likes: 0,
                replies: []
            }
        }
    }
)

// Incrémenter les likes d'un commentaire
db.articles.updateOne(
    {
        _id: ObjectId("..."),
        "comments.commentId": "C001"
    },
    {
        $inc: { "comments.$.likes": 1 }
    }
)

// Ajouter une réponse à un commentaire
db.articles.updateOne(
    {
        _id: ObjectId("..."),
        "comments.commentId": "C001"
    },
    {
        $push: {
            "comments.$.replies": {
                author: "David",
                text: "I agree!",
                date: new Date()
            }
        }
    }
)

// Supprimer les commentaires anciens
db.articles.updateMany(
    {},
    {
        $pull: {
            comments: {
                date: { $lt: ISODate("2023-01-01") },
                likes: { $lt: 5 }
            }
        }
    }
)
```

### Cas 3 : Réseau Social - Gestion d'Amis

```javascript
// Structure
{
    userId: 123,
    username: "alice",
    friends: [
        {
            userId: 456,
            username: "bob",
            since: ISODate("2024-01-15"),
            status: "active"
        },
        {
            userId: 789,
            username: "charlie",
            since: ISODate("2024-02-20"),
            status: "active"
        }
    ],
    friendRequests: [
        {
            userId: 321,
            username: "david",
            date: ISODate("2024-03-10")
        }
    ]
}

// Ajouter un ami
db.users.updateOne(
    { userId: 123 },
    {
        $push: {
            friends: {
                userId: 999,
                username: "eve",
                since: new Date(),
                status: "active"
            }
        },
        $pull: {
            friendRequests: { userId: 999 }
        }
    }
)

// Retirer un ami
db.users.updateOne(
    { userId: 123 },
    {
        $pull: {
            friends: { userId: 456 }
        }
    }
)

// Trouver les utilisateurs avec plus de 50 amis
db.users.find({
    $expr: { $gte: [{ $size: "$friends" }, 50] }
})

// Trouver les utilisateurs ayant un ami spécifique
db.users.find({
    "friends.userId": 456
})

// Trouver les utilisateurs avec des demandes en attente
db.users.find({
    $expr: { $gt: [{ $size: "$friendRequests" }, 0] }
})
```

---

## Comparaison avec SQL

En SQL, les tableaux nécessiteraient généralement des tables séparées avec des jointures.

### Approche SQL

```sql
-- Table principale
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

-- Table de relation
CREATE TABLE user_hobbies (
    user_id INT,
    hobby VARCHAR(50),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Requête avec jointure
SELECT u.name
FROM users u
JOIN user_hobbies h ON u.id = h.user_id
WHERE h.hobby IN ('reading', 'coding');
```

### Approche MongoDB

```javascript
// Tout dans un document
{
    _id: 1,
    name: "Alice",
    hobbies: ["reading", "swimming", "coding"]
}

// Requête simple, sans jointure
db.users.find({
    hobbies: { $in: ["reading", "coding"] }
})
```

---

## Bonnes Pratiques

### 1. Limiter la Taille des Tableaux

```javascript
// ⚠️ Éviter les tableaux très volumineux (> 1000 éléments)
{
    userId: 123,
    orders: [/* 10,000 commandes */]
}

// ✅ Pour grandes quantités, utiliser une collection séparée
// Collection users
{ userId: 123, name: "Alice" }

// Collection orders
{ userId: 123, orderId: "ORD-001", ... }
```

### 2. Utiliser `$addToSet` pour Éviter les Doublons

```javascript
// ✅ Évite automatiquement les doublons
db.articles.updateOne(
    { _id: ObjectId("...") },
    { $addToSet: { tags: "mongodb" } }
)

// ⚠️ Peut créer des doublons
db.articles.updateOne(
    { _id: ObjectId("...") },
    { $push: { tags: "mongodb" } }
)
```

### 3. Indexer les Tableaux pour les Recherches

```javascript
// Créer un index multikey
db.articles.createIndex({ tags: 1 })
db.products.createIndex({ "features": 1 })

// Les requêtes utilisent l'index efficacement
db.articles.find({ tags: "mongodb" })
```

### 4. Utiliser `$elemMatch` pour Tableaux d'Objets

```javascript
// ✅ Garantit les conditions sur le même élément
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 },
            verified: true
        }
    }
})

// ❌ Peut donner des résultats inattendus
db.products.find({
    "reviews.rating": { $gte: 4 },
    "reviews.verified": true
})
```

### 5. Limiter les Projections de Tableaux

```javascript
// ✅ Ne récupérer que les éléments nécessaires
db.articles.find(
    {},
    {
        title: 1,
        comments: { $slice: 10 }  // Seulement 10 commentaires
    }
)

// ⚠️ Récupère tous les commentaires (peut être lourd)
db.articles.find({}, { title: 1, comments: 1 })
```

### 6. Maintenir la Cohérence des Types

```javascript
// ✅ Bon : types cohérents
{
    scores: [85, 92, 78, 95]
}

// ❌ À éviter : types mixtes
{
    scores: [85, "92", 78, null, "N/A"]
}
```

---

## Pièges Courants à Éviter

### 1. Confusion entre `$in` et `$all`

```javascript
// ❌ Confusion
db.products.find({
    features: { $in: ["wireless", "bluetooth"] }
})
// Retourne produits avec wireless OU bluetooth

// ✅ Clair
db.products.find({
    features: { $all: ["wireless", "bluetooth"] }
})
// Retourne produits avec wireless ET bluetooth
```

### 2. Oublier `$elemMatch` pour Tableaux d'Objets

```javascript
// ❌ Incorrect pour tableaux d'objets
db.products.find({
    "reviews.rating": { $gte: 4 },
    "reviews.verified": true
})

// ✅ Correct
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
db.users.find({ hobbies: { $size: { $gte: 3 } } })

// ✅ Utiliser $expr
db.users.find({
    $expr: { $gte: [{ $size: "$hobbies" }, 3] }
})
```

### 4. Tableaux Trop Volumineux

```javascript
// ⚠️ Document trop grand (peut atteindre la limite de 16 Mo)
{
    userId: 123,
    activities: [/* milliers d'activités */]
}

// ✅ Collection séparée pour grandes quantités
```

### 5. Index Incorrects sur Tableaux

```javascript
// ⚠️ Index sur tableau crée un index multikey
db.articles.createIndex({ tags: 1 })

// Comprendre que cela indexe chaque élément du tableau
// Peut impacter les performances si tableaux très volumineux
```

---

## Performance et Optimisation

### Index Multikey

```javascript
// Créer un index sur un champ tableau
db.articles.createIndex({ tags: 1 })

// MongoDB crée automatiquement un index multikey
// Chaque valeur du tableau est indexée séparément
```

### Limiter les Projections

```javascript
// ✅ Performant : limite les données retournées
db.articles.find(
    { tags: "mongodb" },
    {
        title: 1,
        comments: { $slice: 5 }
    }
)

// ⚠️ Moins performant : retourne tout
db.articles.find({ tags: "mongodb" })
```

### Vérification avec `explain()`

```javascript
// Analyser les requêtes sur tableaux
db.articles.find({
    tags: { $all: ["mongodb", "database"] }
}).explain("executionStats")

// Vérifier l'utilisation d'index
db.products.find({
    reviews: {
        $elemMatch: {
            rating: { $gte: 4 }
        }
    }
}).explain("executionStats")
```

---

## Points Clés à Retenir

✅ MongoDB vérifie **automatiquement** dans les tableaux lors des recherches

✅ **`$in`** : au moins une valeur (OU) - **`$all`** : toutes les valeurs (ET)

✅ **`$size`** vérifie la taille exacte (pas de comparaisons directes)

✅ **`$elemMatch`** garantit les conditions sur le **même élément**

✅ **`$`** met à jour le **premier** élément correspondant

✅ **`$[]`** met à jour **tous** les éléments

✅ **`$[<identifier>]`** met à jour les éléments **filtrés**

✅ **`$push`** ajoute, **`$pull`** supprime, **`$addToSet`** ajoute sans doublons

✅ Limitez la **taille des tableaux** (< 1000 éléments idéalement)

✅ Les **index multikey** améliorent les performances sur les tableaux

---

## Résumé

Dans ce chapitre, vous avez appris :

- Les différents types de tableaux dans MongoDB
- Les requêtes de base sur tableaux (égalité, `$in`, `$nin`)
- Les opérateurs spécifiques : `$all`, `$size`, `$elemMatch`
- Comment accéder aux éléments par position
- Les opérateurs de mise à jour : `$`, `$[]`, `$[<identifier>]`
- Les opérations d'ajout et suppression : `$push`, `$pull`, `$addToSet`, `$pop`
- Les projections sur tableaux avec `$slice` et `$elemMatch`
- Les cas d'usage pratiques dans différents contextes
- Les bonnes pratiques et pièges à éviter
- L'optimisation des performances avec les index multikey

Les tableaux sont une fonctionnalité puissante de MongoDB qui permet de stocker des collections de données directement dans les documents. Une bonne maîtrise des requêtes sur tableaux est essentielle pour exploiter pleinement le potentiel de MongoDB et créer des applications performantes.

Avec ce chapitre, vous avez complété la **Partie 1** du tutoriel sur les requêtes et filtres MongoDB. Vous disposez maintenant de tous les outils nécessaires pour interroger efficacement vos données, quelle que soit leur complexité !

---


⏭️ [Modélisation des Données](/04-modelisation-des-donnees/README.md)
