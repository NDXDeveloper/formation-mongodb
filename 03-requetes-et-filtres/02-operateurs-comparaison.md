🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.2 Opérateurs de Comparaison

## Introduction

Dans le chapitre précédent, nous avons appris à effectuer des recherches simples par égalité. Cependant, dans la réalité, vous aurez souvent besoin de critères plus complexes : trouver des produits dont le prix est supérieur à un certain montant, des utilisateurs dont l'âge est inférieur à 18 ans, ou des commandes créées avant une certaine date.

MongoDB fournit un ensemble d'**opérateurs de comparaison** qui permettent d'exprimer ces conditions avancées. Ces opérateurs sont préfixés par le symbole dollar (`$`) et s'utilisent dans vos documents de requête.

Dans ce chapitre, nous allons explorer en détail les opérateurs de comparaison les plus utilisés.

---

## Vue d'Ensemble des Opérateurs de Comparaison

MongoDB propose les opérateurs de comparaison suivants :

| Opérateur | Signification | Description |
|-----------|---------------|-------------|
| `$eq` | Equal (égal) | Correspond aux valeurs égales à une valeur spécifiée |
| `$ne` | Not Equal (différent) | Correspond aux valeurs différentes d'une valeur spécifiée |
| `$gt` | Greater Than (supérieur) | Correspond aux valeurs strictement supérieures |
| `$gte` | Greater Than or Equal (supérieur ou égal) | Correspond aux valeurs supérieures ou égales |
| `$lt` | Less Than (inférieur) | Correspond aux valeurs strictement inférieures |
| `$lte` | Less Than or Equal (inférieur ou égal) | Correspond aux valeurs inférieures ou égales |
| `$in` | In (dans) | Correspond à n'importe quelle valeur dans un tableau |
| `$nin` | Not In (pas dans) | Ne correspond à aucune valeur dans un tableau |

---

## Syntaxe Générale

La syntaxe générale pour utiliser un opérateur de comparaison est :

```javascript
{ champ: { $operateur: valeur } }
```

**Exemple** :
```javascript
// Trouver les produits avec un prix supérieur à 100
db.products.find({ price: { $gt: 100 } })
```

---

## L'Opérateur `$eq` (Equal)

L'opérateur `$eq` correspond aux documents où la valeur d'un champ est **égale** à la valeur spécifiée.

### Syntaxe

```javascript
{ champ: { $eq: valeur } }
```

### Équivalence avec la Syntaxe Courte

En réalité, `$eq` est rarement utilisé car il existe une syntaxe plus courte et équivalente :

```javascript
// Avec $eq (explicite)
db.users.find({ age: { $eq: 25 } })

// Sans $eq (implicite, préféré)
db.users.find({ age: 25 })
```

Les deux requêtes sont strictement équivalentes. La version sans `$eq` est plus concise et généralement préférée.

### Exemples

```javascript
// Trouver les utilisateurs avec le statut "active"
db.users.find({ status: { $eq: "active" } })
// Équivalent à :
db.users.find({ status: "active" })

// Trouver les produits avec un prix exactement de 99.99
db.products.find({ price: { $eq: 99.99 } })

// Trouver les commandes avec exactement 5 articles
db.orders.find({ itemCount: { $eq: 5 } })
```

### Quand Utiliser `$eq` ?

`$eq` devient utile dans des situations spécifiques :

1. **Pour la cohérence avec d'autres opérateurs** dans des requêtes complexes
2. **Dans les pipelines d'agrégation** où la syntaxe explicite est parfois nécessaire
3. **Pour la lisibilité** dans du code généré automatiquement

---

## L'Opérateur `$ne` (Not Equal)

L'opérateur `$ne` correspond aux documents où la valeur d'un champ est **différente** de la valeur spécifiée.

### Syntaxe

```javascript
{ champ: { $ne: valeur } }
```

### Exemples

```javascript
// Trouver tous les utilisateurs qui ne sont pas actifs
db.users.find({ status: { $ne: "active" } })

// Trouver les produits dont le prix n'est pas 0
db.products.find({ price: { $ne: 0 } })

// Trouver les commandes dont le statut n'est pas "cancelled"
db.orders.find({ status: { $ne: "cancelled" } })
```

### Comportement Particulier

**Important** : `$ne` retourne également les documents où le champ **n'existe pas**.

```javascript
// Document 1: { name: "Alice", status: "active" }
// Document 2: { name: "Bob", status: "inactive" }
// Document 3: { name: "Charlie" } // Pas de champ status

// Cette requête retournera les documents 2 ET 3
db.users.find({ status: { $ne: "active" } })
```

Si vous voulez exclure les documents sans le champ, combinez avec `$exists` :

```javascript
// Ne retourner que les documents avec status != "active" ET status existe
db.users.find({
    status: { $ne: "active", $exists: true }
})
```

---

## L'Opérateur `$gt` (Greater Than)

L'opérateur `$gt` correspond aux documents où la valeur d'un champ est **strictement supérieure** à la valeur spécifiée.

### Syntaxe

```javascript
{ champ: { $gt: valeur } }
```

### Exemples avec des Nombres

```javascript
// Trouver les produits avec un prix supérieur à 100
db.products.find({ price: { $gt: 100 } })

// Trouver les utilisateurs de plus de 18 ans
db.users.find({ age: { $gt: 18 } })

// Trouver les commandes avec un montant supérieur à 500
db.orders.find({ amount: { $gt: 500 } })

// Trouver les articles avec plus de 1000 vues
db.articles.find({ views: { $gt: 1000 } })
```

### Exemples avec des Dates

`$gt` fonctionne également avec les dates :

```javascript
// Trouver les commandes créées après le 1er janvier 2024
db.orders.find({
    createdAt: { $gt: ISODate("2024-01-01T00:00:00Z") }
})

// Trouver les utilisateurs inscrits après une date spécifique
db.users.find({
    registrationDate: { $gt: new Date("2024-06-01") }
})
```

### Exemples avec des Chaînes

`$gt` peut aussi comparer des chaînes (ordre lexicographique) :

```javascript
// Trouver les utilisateurs dont le nom vient après "M" dans l'alphabet
db.users.find({ lastName: { $gt: "M" } })
// Retournera : "Martin", "Smith", "Zhang", etc.
// N'inclura pas : "Adams", "Lee", "Martin" (car "Martin" > "M" mais commence par M)
```

**Note** : La comparaison de chaînes est sensible à la casse (majuscules/minuscules).

---

## L'Opérateur `$gte` (Greater Than or Equal)

L'opérateur `$gte` correspond aux documents où la valeur d'un champ est **supérieure ou égale** à la valeur spécifiée.

### Syntaxe

```javascript
{ champ: { $gte: valeur } }
```

### Différence avec `$gt`

```javascript
// $gt : strictement supérieur (n'inclut pas la valeur)
db.products.find({ price: { $gt: 100 } })
// Retourne : 100.01, 150, 200, etc.
// N'inclut PAS : 100

// $gte : supérieur ou égal (inclut la valeur)
db.products.find({ price: { $gte: 100 } })
// Retourne : 100, 100.01, 150, 200, etc.
// Inclut : 100
```

### Exemples

```javascript
// Trouver les utilisateurs de 18 ans ou plus (majeurs)
db.users.find({ age: { $gte: 18 } })

// Trouver les produits avec un stock d'au moins 10 unités
db.products.find({ stock: { $gte: 10 } })

// Trouver les notes supérieures ou égales à 10/20
db.exams.find({ score: { $gte: 10 } })

// Trouver les commandes à partir du 1er janvier 2024 (inclus)
db.orders.find({
    orderDate: { $gte: ISODate("2024-01-01T00:00:00Z") }
})
```

---

## L'Opérateur `$lt` (Less Than)

L'opérateur `$lt` correspond aux documents où la valeur d'un champ est **strictement inférieure** à la valeur spécifiée.

### Syntaxe

```javascript
{ champ: { $lt: valeur } }
```

### Exemples

```javascript
// Trouver les produits avec un prix inférieur à 50
db.products.find({ price: { $lt: 50 } })

// Trouver les utilisateurs de moins de 18 ans (mineurs)
db.users.find({ age: { $lt: 18 } })

// Trouver les articles avec moins de 100 vues
db.articles.find({ views: { $lt: 100 } })

// Trouver les commandes créées avant le 1er janvier 2024
db.orders.find({
    createdAt: { $lt: ISODate("2024-01-01T00:00:00Z") }
})

// Trouver les produits avec un stock faible (moins de 5)
db.products.find({ stock: { $lt: 5 } })
```

---

## L'Opérateur `$lte` (Less Than or Equal)

L'opérateur `$lte` correspond aux documents où la valeur d'un champ est **inférieure ou égale** à la valeur spécifiée.

### Syntaxe

```javascript
{ champ: { $lte: valeur } }
```

### Exemples

```javascript
// Trouver les produits avec un prix de 50 euros ou moins
db.products.find({ price: { $lte: 50 } })

// Trouver les utilisateurs de 17 ans ou moins
db.users.find({ age: { $lte: 17 } })

// Trouver les examens avec une note de 10 ou moins
db.exams.find({ score: { $lte: 10 } })

// Trouver les commandes jusqu'au 31 décembre 2023 (inclus)
db.orders.find({
    orderDate: { $lte: ISODate("2023-12-31T23:59:59Z") }
})
```

---

## Combiner Plusieurs Opérateurs sur le Même Champ

Vous pouvez combiner plusieurs opérateurs de comparaison sur un même champ pour créer des **plages de valeurs**.

### Syntaxe pour les Plages

```javascript
{ champ: { $gte: valeurMin, $lte: valeurMax } }
```

### Exemples de Plages

```javascript
// Trouver les produits entre 50 et 100 euros (inclus)
db.products.find({
    price: { $gte: 50, $lte: 100 }
})

// Trouver les adultes de 18 à 65 ans
db.users.find({
    age: { $gte: 18, $lte: 65 }
})

// Trouver les commandes de janvier 2024
db.orders.find({
    orderDate: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2024-02-01T00:00:00Z")
    }
})

// Trouver les notes entre 10 et 15 (exclus)
db.exams.find({
    score: { $gt: 10, $lt: 15 }
})
```

### Différentes Combinaisons Possibles

```javascript
// Strictement entre 50 et 100 (exclus)
db.products.find({ price: { $gt: 50, $lt: 100 } })

// Entre 50 (inclus) et 100 (exclus)
db.products.find({ price: { $gte: 50, $lt: 100 } })

// Entre 50 (exclus) et 100 (inclus)
db.products.find({ price: { $gt: 50, $lte: 100 } })

// Entre 50 et 100 (tous deux inclus)
db.products.find({ price: { $gte: 50, $lte: 100 } })
```

---

## L'Opérateur `$in` (In)

L'opérateur `$in` permet de rechercher des documents où la valeur d'un champ correspond à **n'importe quelle valeur** dans un tableau spécifié.

### Syntaxe

```javascript
{ champ: { $in: [valeur1, valeur2, valeur3, ...] } }
```

### Concept

`$in` est l'équivalent d'un **OU logique** pour un même champ. Au lieu d'écrire plusieurs conditions `OR`, vous utilisez `$in`.

### Exemples Simples

```javascript
// Trouver les utilisateurs avec le statut "active" OU "pending"
db.users.find({
    status: { $in: ["active", "pending"] }
})

// Trouver les produits des catégories "Electronics", "Books" ou "Toys"
db.products.find({
    category: { $in: ["Electronics", "Books", "Toys"] }
})

// Trouver les commandes avec les statuts "shipped", "delivered"
db.orders.find({
    status: { $in: ["shipped", "delivered"] }
})

// Trouver les produits avec un prix de 10, 20 ou 30
db.products.find({
    price: { $in: [10, 20, 30] }
})
```

### `$in` avec des Tableaux

`$in` fonctionne aussi avec des champs de type tableau :

```javascript
// Documents
{ name: "Alice", hobbies: ["reading", "swimming"] }
{ name: "Bob", hobbies: ["gaming", "cooking"] }
{ name: "Charlie", hobbies: ["reading", "gaming"] }

// Trouver les utilisateurs ayant "reading" OU "gaming" comme hobby
db.users.find({
    hobbies: { $in: ["reading", "gaming"] }
})
// Retourne Alice, Bob et Charlie
```

### `$in` avec des ObjectId

```javascript
// Trouver les commandes de plusieurs clients spécifiques
db.orders.find({
    customerId: {
        $in: [
            ObjectId("507f1f77bcf86cd799439011"),
            ObjectId("507f191e810c19729de860ea"),
            ObjectId("507f1f77bcf86cd799439012")
        ]
    }
})
```

### Avantages de `$in`

**Au lieu de** :
```javascript
// Version longue avec $or
db.products.find({
    $or: [
        { category: "Electronics" },
        { category: "Books" },
        { category: "Toys" }
    ]
})
```

**Utilisez** :
```javascript
// Version courte avec $in
db.products.find({
    category: { $in: ["Electronics", "Books", "Toys"] }
})
```

La version avec `$in` est plus lisible, plus concise et plus performante.

---

## L'Opérateur `$nin` (Not In)

L'opérateur `$nin` est l'opposé de `$in`. Il correspond aux documents où la valeur d'un champ **ne correspond à aucune** des valeurs dans un tableau spécifié.

### Syntaxe

```javascript
{ champ: { $nin: [valeur1, valeur2, valeur3, ...] } }
```

### Exemples

```javascript
// Trouver les utilisateurs qui ne sont ni "banned" ni "deleted"
db.users.find({
    status: { $nin: ["banned", "deleted"] }
})

// Trouver les produits qui ne sont pas dans les catégories "Food" ou "Drinks"
db.products.find({
    category: { $nin: ["Food", "Drinks"] }
})

// Trouver les commandes qui ne sont ni "cancelled" ni "refunded"
db.orders.find({
    status: { $nin: ["cancelled", "refunded"] }
})

// Exclure plusieurs prix spécifiques
db.products.find({
    price: { $nin: [9.99, 19.99, 29.99] }
})
```

### Comportement Important

**`$nin` retourne également les documents où le champ n'existe pas**, tout comme `$ne`.

```javascript
// Document 1: { name: "Laptop", category: "Electronics" }
// Document 2: { name: "Book", category: "Books" }
// Document 3: { name: "Misc" } // Pas de champ category

// Cette requête retourne les documents 2 ET 3
db.products.find({
    category: { $nin: ["Electronics", "Toys"] }
})
```

Pour exclure les documents sans le champ :

```javascript
db.products.find({
    category: { $nin: ["Electronics", "Toys"], $exists: true }
})
```

---

## Combinaison d'Opérateurs de Comparaison

Vous pouvez combiner différents opérateurs de comparaison pour créer des requêtes sophistiquées.

### Combiner sur Plusieurs Champs

```javascript
// Trouver les produits "Electronics" avec un prix entre 100 et 500
db.products.find({
    category: "Electronics",
    price: { $gte: 100, $lte: 500 }
})

// Trouver les utilisateurs actifs de plus de 18 ans
db.users.find({
    status: "active",
    age: { $gte: 18 }
})

// Trouver les commandes complétées avec un montant supérieur à 1000
db.orders.find({
    status: "completed",
    amount: { $gt: 1000 }
})
```

### Combiner `$in` avec d'autres Opérateurs

```javascript
// Produits "Electronics" ou "Books" avec prix < 50
db.products.find({
    category: { $in: ["Electronics", "Books"] },
    price: { $lt: 50 }
})

// Utilisateurs actifs ou pending, de plus de 18 ans
db.users.find({
    status: { $in: ["active", "pending"] },
    age: { $gte: 18 }
})
```

### Exclusions Multiples

```javascript
// Produits ni "Food" ni "Drinks", avec prix différent de 0
db.products.find({
    category: { $nin: ["Food", "Drinks"] },
    price: { $ne: 0 }
})

// Utilisateurs ni "banned" ni "deleted", de moins de 65 ans
db.users.find({
    status: { $nin: ["banned", "deleted"] },
    age: { $lt: 65 }
})
```

---

## Comparaisons avec Différents Types de Données

### Nombres

```javascript
// Entiers et décimaux
db.products.find({ price: { $gt: 10.5 } })
db.stats.find({ count: { $gte: 1000 } })
```

### Chaînes de Caractères

```javascript
// Comparaison lexicographique (ordre alphabétique)
db.users.find({ lastName: { $gte: "M" } })
db.products.find({ name: { $lt: "C" } })
```

**Attention** : La comparaison de chaînes est sensible à la casse.

```javascript
// "Apple" < "apple" (les majuscules viennent avant)
db.products.find({ name: { $gt: "A" } })
// Ne retournera pas les noms commençant par "a" minuscule
```

### Dates

```javascript
// Comparaison de dates
db.orders.find({
    createdAt: { $gte: ISODate("2024-01-01T00:00:00Z") }
})

// Plage de dates
db.events.find({
    eventDate: {
        $gte: ISODate("2024-06-01T00:00:00Z"),
        $lt: ISODate("2024-07-01T00:00:00Z")
    }
})
```

### Booléens

```javascript
// Les booléens peuvent aussi être comparés
db.products.find({ inStock: { $eq: true } })
// Équivalent à :
db.products.find({ inStock: true })

// Avec $ne
db.products.find({ isDiscontinued: { $ne: true } })
```

### Null

```javascript
// Trouver les documents où un champ n'est pas null
db.users.find({ email: { $ne: null } })

// Avec $in pour null et undefined
db.users.find({ email: { $in: [null, undefined] } })
```

---

## Cas d'Usage Pratiques

### Cas 1 : E-commerce - Filtrage de Produits

```javascript
// Collection: products
{
    _id: ObjectId("..."),
    name: "Laptop",
    category: "Electronics",
    price: 899.99,
    stock: 15,
    rating: 4.5
}
```

**Requêtes** :

```javascript
// Produits en promotion (moins de 50€)
db.products.find({ price: { $lt: 50 } })

// Produits disponibles (stock positif)
db.products.find({ stock: { $gt: 0 } })

// Produits bien notés (4 étoiles ou plus)
db.products.find({ rating: { $gte: 4 } })

// Produits dans une gamme de prix (50-200€)
db.products.find({
    price: { $gte: 50, $lte: 200 }
})

// Produits de plusieurs catégories spécifiques
db.products.find({
    category: { $in: ["Electronics", "Books", "Toys"] }
})

// Produits hors stock critique (plus de 10 unités)
db.products.find({ stock: { $gte: 10 } })
```

### Cas 2 : Gestion d'Utilisateurs

```javascript
// Collection: users
{
    _id: ObjectId("..."),
    username: "johndoe",
    age: 30,
    status: "active",
    registrationDate: ISODate("2023-06-15T10:00:00Z"),
    role: "user"
}
```

**Requêtes** :

```javascript
// Utilisateurs majeurs
db.users.find({ age: { $gte: 18 } })

// Utilisateurs actifs ou en attente
db.users.find({
    status: { $in: ["active", "pending"] }
})

// Utilisateurs inscrits en 2024
db.users.find({
    registrationDate: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2025-01-01T00:00:00Z")
    }
})

// Utilisateurs qui ne sont pas bannis ou supprimés
db.users.find({
    status: { $nin: ["banned", "deleted"] }
})

// Utilisateurs seniors (65 ans ou plus)
db.users.find({ age: { $gte: 65 } })

// Utilisateurs non-administrateurs
db.users.find({
    role: { $ne: "admin" }
})
```

### Cas 3 : Analyse de Commandes

```javascript
// Collection: orders
{
    _id: ObjectId("..."),
    orderNumber: "ORD-12345",
    customerId: ObjectId("..."),
    amount: 150.00,
    status: "completed",
    orderDate: ISODate("2024-01-15T10:30:00Z"),
    itemCount: 3
}
```

**Requêtes** :

```javascript
// Commandes importantes (montant > 1000€)
db.orders.find({ amount: { $gt: 1000 } })

// Commandes du dernier trimestre 2024
db.orders.find({
    orderDate: {
        $gte: ISODate("2024-10-01T00:00:00Z"),
        $lt: ISODate("2025-01-01T00:00:00Z")
    }
})

// Commandes complétées ou expédiées
db.orders.find({
    status: { $in: ["completed", "shipped"] }
})

// Commandes moyennes (entre 50 et 200€)
db.orders.find({
    amount: { $gte: 50, $lte: 200 }
})

// Grosses commandes (plus de 10 articles)
db.orders.find({ itemCount: { $gt: 10 } })

// Commandes qui ne sont pas annulées
db.orders.find({
    status: { $ne: "cancelled" }
})
```

---

## Comparaison avec SQL

Pour les développeurs venant du SQL, voici les équivalences :

| SQL | MongoDB |
|-----|---------|
| `WHERE age = 25` | `{ age: 25 }` ou `{ age: { $eq: 25 } }` |
| `WHERE age != 25` | `{ age: { $ne: 25 } }` |
| `WHERE age > 18` | `{ age: { $gt: 18 } }` |
| `WHERE age >= 18` | `{ age: { $gte: 18 } }` |
| `WHERE age < 65` | `{ age: { $lt: 65 } }` |
| `WHERE age <= 65` | `{ age: { $lte: 65 } }` |
| `WHERE status IN ('active', 'pending')` | `{ status: { $in: ["active", "pending"] } }` |
| `WHERE status NOT IN ('banned', 'deleted')` | `{ status: { $nin: ["banned", "deleted"] } }` |
| `WHERE age BETWEEN 18 AND 65` | `{ age: { $gte: 18, $lte: 65 } }` |
| `WHERE price > 50 AND price < 100` | `{ price: { $gt: 50, $lt: 100 } }` |

---

## Bonnes Pratiques

### 1. Créer des Index sur les Champs de Comparaison

Les requêtes avec opérateurs de comparaison bénéficient grandement des index :

```javascript
// Créer un index sur le champ price pour optimiser les requêtes
db.products.createIndex({ price: 1 })

// Créer un index sur le champ age
db.users.createIndex({ age: 1 })

// Index composé pour des requêtes fréquentes
db.products.createIndex({ category: 1, price: 1 })
```

### 2. Utiliser `$in` au Lieu de Multiples `$or`

**❌ Moins optimal** :
```javascript
db.users.find({
    $or: [
        { status: "active" },
        { status: "pending" },
        { status: "trial" }
    ]
})
```

**✅ Meilleur** :
```javascript
db.users.find({
    status: { $in: ["active", "pending", "trial"] }
})
```

### 3. Combiner Opérateurs Intelligemment

```javascript
// Efficient : combine filtres sur le même document
db.products.find({
    category: "Electronics",
    price: { $gte: 100, $lte: 500 },
    stock: { $gt: 0 }
})
```

### 4. Attention aux Types de Données

MongoDB compare des types différents selon un ordre spécifique. Assurez-vous que vos valeurs sont du bon type :

```javascript
// ✅ Correct : nombre vs nombre
db.products.find({ price: { $gt: 100 } })

// ❌ Attention : chaîne vs nombre
db.products.find({ price: { $gt: "100" } })
// Cela comparera lexicographiquement, pas numériquement
```

### 5. Utiliser des Plages pour les Dates

```javascript
// ✅ Bonne pratique : définir une plage claire
db.orders.find({
    orderDate: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2024-02-01T00:00:00Z")
    }
})
```

### 6. Tester les Limites avec `$gte` vs `$gt`

Choisissez le bon opérateur selon que vous voulez inclure ou exclure la valeur limite :

```javascript
// Majeurs (18 ans inclus)
db.users.find({ age: { $gte: 18 } })

// Plus de 18 ans (18 exclu)
db.users.find({ age: { $gt: 18 } })
```

---

## Pièges Courants à Éviter

### 1. Confusion entre `$gt` et `$gte`

```javascript
// ❌ Erreur courante : exclure la valeur de référence
db.users.find({ age: { $gt: 18 } })
// N'inclut PAS les utilisateurs de 18 ans exactement

// ✅ Correct pour "18 ans ou plus"
db.users.find({ age: { $gte: 18 } })
```

### 2. Oublier que `$ne` et `$nin` Incluent les Documents Sans le Champ

```javascript
// Retourne aussi les documents sans le champ "status"
db.users.find({ status: { $ne: "active" } })

// Solution : ajouter $exists
db.users.find({
    status: { $ne: "active", $exists: true }
})
```

### 3. Comparaison de Types Différents

```javascript
// Document avec price: "100" (string)
// Document avec price: 100 (number)

db.products.find({ price: { $gt: 50 } })
// Ne retournera que les documents avec price numérique
```

### 4. Ordre de Comparaison des Chaînes et Casse

```javascript
// "Apple" est différent de "apple"
db.products.find({ name: { $gt: "A" } })
// Les majuscules ont un ordre différent des minuscules dans l'ASCII
```

Pour une comparaison insensible à la casse, utilisez les expressions régulières (voir chapitre 3.5).

---

## Performance et Optimisation

### Index et Opérateurs de Comparaison

Les opérateurs de comparaison sont **très efficaces** avec des index appropriés :

```javascript
// Sans index : scan complet de la collection (COLLSCAN)
db.products.find({ price: { $gte: 100, $lte: 500 } })

// Avec index : utilisation optimale de l'index (IXSCAN)
db.products.createIndex({ price: 1 })
db.products.find({ price: { $gte: 100, $lte: 500 } })
```

### Vérifier l'Utilisation des Index avec `explain()`

```javascript
// Analyser la requête
db.products.find({
    price: { $gte: 100, $lte: 500 }
}).explain("executionStats")

// Cherchez "IXSCAN" pour confirmer l'utilisation d'index
// "COLLSCAN" indique un scan complet (pas optimal)
```

### Ordre des Opérateurs

MongoDB optimise automatiquement l'ordre des opérateurs, mais pour les index composés, l'ordre compte :

```javascript
// Index composé
db.products.createIndex({ category: 1, price: 1 })

// ✅ Bien : utilise l'index efficacement
db.products.find({
    category: "Electronics",
    price: { $gte: 100 }
})

// ⚠️ Moins optimal : ne peut utiliser que la première partie de l'index
db.products.find({
    price: { $gte: 100 }
})
```

---

## Points Clés à Retenir

✅ Les **opérateurs de comparaison** permettent des requêtes sophistiquées au-delà de l'égalité simple

✅ **`$gt`, `$gte`, `$lt`, `$lte`** sont utilisés pour les comparaisons numériques, de dates et de chaînes

✅ **`$in`** et **`$nin`** permettent de rechercher ou exclure des valeurs dans une liste

✅ Vous pouvez **combiner plusieurs opérateurs** sur le même champ pour créer des plages

✅ **`$ne`** et **`$nin`** incluent les documents où le champ n'existe pas

✅ Les opérateurs fonctionnent avec différents types : **nombres, chaînes, dates, booléens**

✅ **Les index** améliorent considérablement les performances des requêtes avec opérateurs de comparaison

✅ **`$in`** est préférable à plusieurs conditions **`$or`** sur le même champ

✅ Attention aux **types de données** : MongoDB compare selon des règles strictes

---

## Résumé

Dans ce chapitre, vous avez appris :

- L'utilisation des 8 principaux opérateurs de comparaison de MongoDB
- Comment combiner plusieurs opérateurs pour créer des plages de valeurs
- Les différences subtiles entre `$gt`/`$gte` et `$lt`/`$lte`
- L'utilisation de `$in` et `$nin` pour les listes de valeurs
- Les comportements particuliers de `$ne` et `$nin` avec les champs manquants
- Les bonnes pratiques pour optimiser vos requêtes
- Les pièges courants à éviter

Ces opérateurs constituent la base de la plupart des requêtes MongoDB. Dans le prochain chapitre, nous découvrirons les **opérateurs logiques** qui vous permettront de combiner ces conditions de manière encore plus flexible avec `$and`, `$or`, `$not` et `$nor`.

---


⏭️ [Opérateurs logiques ($and, $or, $not, $nor)](/03-requetes-et-filtres/03-operateurs-logiques.md)
