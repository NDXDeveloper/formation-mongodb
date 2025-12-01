🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.3 Opérateurs Logiques

## Introduction

Dans les chapitres précédents, nous avons appris à créer des requêtes simples et à utiliser les opérateurs de comparaison. Cependant, les besoins réels sont souvent plus complexes : trouver des utilisateurs qui sont **soit** actifs **soit** en période d'essai, des produits qui **ne sont pas** en rupture de stock, ou des commandes qui correspondent à **l'une ou l'autre** de plusieurs conditions.

Les **opérateurs logiques** de MongoDB vous permettent de combiner plusieurs conditions de manière flexible pour créer des requêtes sophistiquées. Ils fonctionnent de manière similaire aux opérateurs logiques en programmation ou en SQL.

Dans ce chapitre, nous allons explorer les quatre opérateurs logiques principaux de MongoDB : `$and`, `$or`, `$not` et `$nor`.

---

## Vue d'Ensemble des Opérateurs Logiques

MongoDB propose quatre opérateurs logiques principaux :

| Opérateur | Signification | Description |
|-----------|---------------|-------------|
| `$and` | AND (ET logique) | Toutes les conditions doivent être vraies |
| `$or` | OR (OU logique) | Au moins une condition doit être vraie |
| `$not` | NOT (NON logique) | Inverse le résultat d'une condition |
| `$nor` | NOR (NI...NI) | Aucune des conditions ne doit être vraie |

---

## L'Opérateur `$and`

L'opérateur `$and` effectue une opération **ET logique** sur un tableau de conditions. Un document doit correspondre à **toutes les conditions** pour être retourné.

### Syntaxe

```javascript
{ $and: [ { condition1 }, { condition2 }, ... ] }
```

### Comportement Implicite

**Important** : MongoDB applique automatiquement un `$and` implicite lorsque vous spécifiez plusieurs champs dans un même document :

```javascript
// Ces deux requêtes sont équivalentes :

// Version implicite (préférée)
db.users.find({
    status: "active",
    age: { $gte: 18 }
})

// Version explicite avec $and
db.users.find({
    $and: [
        { status: "active" },
        { age: { $gte: 18 } }
    ]
})
```

La version implicite est plus simple et généralement préférée.

### Quand Utiliser `$and` Explicitement ?

Vous devez utiliser `$and` explicitement dans deux cas principaux :

#### Cas 1 : Plusieurs Conditions sur le Même Champ

```javascript
// Produits avec un prix entre 50 et 100, ET un rabais entre 10 et 20%
db.products.find({
    $and: [
        { price: { $gte: 50 } },
        { price: { $lte: 100 } },
        { discount: { $gte: 10 } },
        { discount: { $lte: 20 } }
    ]
})

// Sans $and, la version courte fonctionne aussi :
db.products.find({
    price: { $gte: 50, $lte: 100 },
    discount: { $gte: 10, $lte: 20 }
})
```

#### Cas 2 : Combiner avec d'Autres Opérateurs Logiques

```javascript
// Utilisateurs actifs ET (de France OU de Belgique)
db.users.find({
    $and: [
        { status: "active" },
        { $or: [
            { country: "France" },
            { country: "Belgium" }
        ]}
    ]
})
```

### Exemples Pratiques

```javascript
// Produits en stock ET en promotion
db.products.find({
    $and: [
        { stock: { $gt: 0 } },
        { onSale: true }
    ]
})

// Utilisateurs majeurs ET vérifiés
db.users.find({
    $and: [
        { age: { $gte: 18 } },
        { verified: true }
    ]
})

// Commandes complétées ET livrées en 2024
db.orders.find({
    $and: [
        { status: "completed" },
        { deliveryDate: { $gte: ISODate("2024-01-01") } },
        { deliveryDate: { $lt: ISODate("2025-01-01") } }
    ]
})
```

---

## L'Opérateur `$or`

L'opérateur `$or` effectue une opération **OU logique** sur un tableau de conditions. Un document est retourné si **au moins une** des conditions est vraie.

### Syntaxe

```javascript
{ $or: [ { condition1 }, { condition2 }, ... ] }
```

### Concept

`$or` est l'un des opérateurs les plus utiles car il permet de rechercher des documents correspondant à l'une quelconque de plusieurs conditions.

### Exemples de Base

```javascript
// Utilisateurs actifs OU en période d'essai
db.users.find({
    $or: [
        { status: "active" },
        { status: "trial" }
    ]
})

// Produits en promotion OU nouveaux
db.products.find({
    $or: [
        { onSale: true },
        { isNew: true }
    ]
})

// Commandes urgentes OU express
db.orders.find({
    $or: [
        { priority: "urgent" },
        { shippingMethod: "express" }
    ]
})
```

### Comparaison avec `$in`

Pour les conditions sur le **même champ**, utilisez `$in` au lieu de `$or` :

```javascript
// ❌ Moins optimal avec $or
db.users.find({
    $or: [
        { status: "active" },
        { status: "trial" },
        { status: "pending" }
    ]
})

// ✅ Meilleur avec $in
db.users.find({
    status: { $in: ["active", "trial", "pending"] }
})
```

**`$or` est nécessaire** quand les conditions portent sur **différents champs** :

```javascript
// Cas où $or est nécessaire
db.products.find({
    $or: [
        { category: "Electronics" },
        { price: { $lt: 20 } }
    ]
})
```

### Exemples avec Plusieurs Champs

```javascript
// Produits électroniques OU prix inférieur à 50
db.products.find({
    $or: [
        { category: "Electronics" },
        { price: { $lt: 50 } }
    ]
})

// Utilisateurs VIP OU avec plus de 1000 points
db.users.find({
    $or: [
        { membershipLevel: "VIP" },
        { points: { $gt: 1000 } }
    ]
})

// Commandes récentes OU de montant élevé
db.orders.find({
    $or: [
        { orderDate: { $gte: ISODate("2024-01-01") } },
        { amount: { $gt: 1000 } }
    ]
})
```

### `$or` avec Plus de Deux Conditions

Vous pouvez avoir autant de conditions que nécessaire :

```javascript
// Produits en promotion OU nouveaux OU bestsellers
db.products.find({
    $or: [
        { onSale: true },
        { isNew: true },
        { isBestseller: true }
    ]
})

// Utilisateurs selon plusieurs critères de recherche
db.users.find({
    $or: [
        { email: "john@example.com" },
        { username: "johndoe" },
        { phone: "+33612345678" }
    ]
})
```

---

## Combiner `$and` et `$or`

La vraie puissance des opérateurs logiques apparaît lorsqu'on les combine pour créer des requêtes complexes.

### Syntaxe Générale

```javascript
{
    $and: [
        { condition1 },
        { $or: [ { condition2 }, { condition3 } ] },
        { condition4 }
    ]
}
```

### Exemples Pratiques

#### Exemple 1 : Produits avec Conditions Complexes

```javascript
// Produits électroniques en stock ET (en promotion OU nouveaux)
db.products.find({
    $and: [
        { category: "Electronics" },
        { stock: { $gt: 0 } },
        { $or: [
            { onSale: true },
            { isNew: true }
        ]}
    ]
})
```

#### Exemple 2 : Utilisateurs Éligibles

```javascript
// Utilisateurs majeurs ET (vérifiés OU VIP)
db.users.find({
    $and: [
        { age: { $gte: 18 } },
        { $or: [
            { verified: true },
            { membershipLevel: "VIP" }
        ]}
    ]
})

// Version simplifiée (implicite AND)
db.users.find({
    age: { $gte: 18 },
    $or: [
        { verified: true },
        { membershipLevel: "VIP" }
    ]
})
```

#### Exemple 3 : Recherche de Commandes

```javascript
// Commandes complétées ET (montant élevé OU client VIP)
db.orders.find({
    status: "completed",
    $or: [
        { amount: { $gt: 500 } },
        { customerType: "VIP" }
    ]
})
```

### Imbrication Profonde

Vous pouvez imbriquer les opérateurs logiques à plusieurs niveaux :

```javascript
// Logique complexe : (A ET B) OU (C ET D)
db.products.find({
    $or: [
        {
            $and: [
                { category: "Electronics" },
                { price: { $lt: 500 } }
            ]
        },
        {
            $and: [
                { category: "Books" },
                { rating: { $gte: 4.5 } }
            ]
        }
    ]
})
```

---

## L'Opérateur `$not`

L'opérateur `$not` effectue une opération **NON logique** qui inverse le résultat d'une expression. Il retourne les documents qui **ne correspondent pas** à l'expression spécifiée.

### Syntaxe

```javascript
{ champ: { $not: { expression } } }
```

**Important** : `$not` s'applique à une **expression** (avec un opérateur), pas directement à une valeur.

### Différence avec `$ne`

```javascript
// $ne : différent de (s'applique à une valeur)
db.users.find({ status: { $ne: "active" } })

// $not : inverse une expression (s'applique à un opérateur)
db.users.find({ age: { $not: { $gte: 18 } } })
```

### Exemples de Base

```javascript
// Utilisateurs qui ne sont PAS majeurs (< 18 ans)
db.users.find({
    age: { $not: { $gte: 18 } }
})
// Équivalent à : age: { $lt: 18 }

// Produits qui ne sont PAS en promotion (prix > 100)
db.products.find({
    price: { $not: { $lte: 100 } }
})
// Équivalent à : price: { $gt: 100 }

// Commandes qui ne sont PAS récentes
db.orders.find({
    orderDate: { $not: { $gte: ISODate("2024-01-01") } }
})
// Équivalent à : orderDate: { $lt: ISODate("2024-01-01") }
```

### `$not` avec Expressions Régulières

`$not` est particulièrement utile avec les expressions régulières :

```javascript
// Utilisateurs dont l'email ne se termine PAS par "@gmail.com"
db.users.find({
    email: { $not: /gmail\.com$/ }
})

// Produits dont le nom ne contient PAS le mot "pro"
db.products.find({
    name: { $not: /pro/i }
})
```

### `$not` avec d'Autres Opérateurs

```javascript
// Produits dont le prix n'est PAS dans la plage 50-100
db.products.find({
    $and: [
        { price: { $not: { $gte: 50 } } },
        { price: { $not: { $lte: 100 } } }
    ]
})
// Équivalent à : price < 50 OU price > 100

// Utilisateurs qui n'ont PAS entre 18 et 65 ans
db.users.find({
    age: { $not: { $gte: 18, $lte: 65 } }
})
```

### Comportement Important

`$not` retourne également les documents où **le champ n'existe pas** :

```javascript
// Retourne les documents où age < 18 OU age n'existe pas
db.users.find({
    age: { $not: { $gte: 18 } }
})

// Pour exclure les documents sans le champ :
db.users.find({
    age: { $not: { $gte: 18 }, $exists: true }
})
```

---

## L'Opérateur `$nor`

L'opérateur `$nor` effectue une opération **NI...NI logique** sur un tableau de conditions. Un document est retourné si **aucune** des conditions n'est vraie.

### Syntaxe

```javascript
{ $nor: [ { condition1 }, { condition2 }, ... ] }
```

### Concept

`$nor` est l'inverse de `$or` : un document doit **échouer toutes les conditions** pour être retourné.

### Exemples de Base

```javascript
// Utilisateurs qui ne sont NI actifs NI en période d'essai
db.users.find({
    $nor: [
        { status: "active" },
        { status: "trial" }
    ]
})
// Équivalent à : status n'est ni "active" ni "trial"

// Produits qui ne sont NI en promotion NI nouveaux
db.products.find({
    $nor: [
        { onSale: true },
        { isNew: true }
    ]
})

// Commandes qui ne sont NI annulées NI remboursées
db.orders.find({
    $nor: [
        { status: "cancelled" },
        { status: "refunded" }
    ]
})
```

### Comparaison `$nor` vs `$nin`

Pour le **même champ**, utilisez `$nin` :

```javascript
// ❌ Avec $nor (fonctionne mais verbose)
db.users.find({
    $nor: [
        { status: "banned" },
        { status: "deleted" },
        { status: "suspended" }
    ]
})

// ✅ Avec $nin (préféré pour le même champ)
db.users.find({
    status: { $nin: ["banned", "deleted", "suspended"] }
})
```

**`$nor` est nécessaire** quand les conditions portent sur **différents champs** :

```javascript
// Produits qui ne sont NI électroniques NI chers
db.products.find({
    $nor: [
        { category: "Electronics" },
        { price: { $gt: 1000 } }
    ]
})
```

### Exemples avec Plusieurs Champs

```javascript
// Utilisateurs qui ne sont NI VIP NI avec beaucoup de points
db.users.find({
    $nor: [
        { membershipLevel: "VIP" },
        { points: { $gt: 1000 } }
    ]
})

// Produits qui ne sont NI en rupture de stock NI discontinués
db.products.find({
    $nor: [
        { stock: { $lte: 0 } },
        { discontinued: true }
    ]
})

// Articles qui ne sont NI archivés NI supprimés
db.articles.find({
    $nor: [
        { archived: true },
        { deleted: true }
    ]
})
```

### Comportement Important

`$nor` retourne également les documents où **les champs n'existent pas** :

```javascript
// Retourne les documents où :
// - status n'est pas "active" ET status n'est pas "trial"
// - OU status n'existe pas
db.users.find({
    $nor: [
        { status: "active" },
        { status: "trial" }
    ]
})
```

---

## Combinaisons Complexes d'Opérateurs

### Exemple 1 : Recherche Avancée de Produits

```javascript
// Trouver des produits qui sont :
// - En stock ET
// - (En promotion OU nouveaux) ET
// - PAS discontinués
db.products.find({
    stock: { $gt: 0 },
    $or: [
        { onSale: true },
        { isNew: true }
    ],
    discontinued: { $ne: true }
})
```

### Exemple 2 : Filtrage d'Utilisateurs Complexe

```javascript
// Utilisateurs qui sont :
// - Majeurs ET
// - (Vérifiés OU VIP) ET
// - NI bannis NI supprimés
db.users.find({
    age: { $gte: 18 },
    $or: [
        { verified: true },
        { membershipLevel: "VIP" }
    ],
    $nor: [
        { status: "banned" },
        { status: "deleted" }
    ]
})
```

### Exemple 3 : Requête de Commandes Sophistiquée

```javascript
// Commandes qui sont :
// - Complétées OU expédiées ET
// - (Montant > 100 OU client VIP) ET
// - PAS en retard de livraison
db.orders.find({
    $or: [
        { status: "completed" },
        { status: "shipped" }
    ],
    $or: [
        { amount: { $gt: 100 } },
        { customerType: "VIP" }
    ],
    deliveryDate: { $not: { $lt: ISODate("2024-01-01") } }
})
```

### Exemple 4 : Logique Métier Complexe

```javascript
// Produits éligibles à une promotion spéciale :
// (Catégorie Electronics OU Books) ET
// (Prix entre 20 et 200) ET
// (Stock > 5) ET
// NI en promotion NI discontinué
db.products.find({
    $or: [
        { category: "Electronics" },
        { category: "Books" }
    ],
    price: { $gte: 20, $lte: 200 },
    stock: { $gt: 5 },
    $nor: [
        { onSale: true },
        { discontinued: true }
    ]
})
```

---

## Cas d'Usage Pratiques

### Cas 1 : Système de Recherche d'Utilisateurs

```javascript
// Collection: users
{
    _id: ObjectId("..."),
    username: "johndoe",
    email: "john@example.com",
    status: "active",
    verified: true,
    membershipLevel: "standard",
    age: 30,
    country: "France"
}
```

**Requêtes** :

```javascript
// Recherche simple : utilisateurs actifs OU vérifiés
db.users.find({
    $or: [
        { status: "active" },
        { verified: true }
    ]
})

// Recherche avancée : utilisateurs majeurs ET (de France OU Belgique)
db.users.find({
    age: { $gte: 18 },
    $or: [
        { country: "France" },
        { country: "Belgium" }
    ]
})

// Exclusion : utilisateurs NI bannis NI supprimés NI suspendus
db.users.find({
    $nor: [
        { status: "banned" },
        { status: "deleted" },
        { status: "suspended" }
    ]
})

// Complexe : utilisateurs actifs ET majeurs ET (VIP OU > 500 points)
db.users.find({
    status: "active",
    age: { $gte: 18 },
    $or: [
        { membershipLevel: "VIP" },
        { points: { $gt: 500 } }
    ]
})
```

### Cas 2 : Filtrage de Produits E-commerce

```javascript
// Collection: products
{
    _id: ObjectId("..."),
    name: "Laptop Pro",
    category: "Electronics",
    price: 899.99,
    stock: 15,
    onSale: false,
    isNew: true,
    rating: 4.5,
    discontinued: false
}
```

**Requêtes** :

```javascript
// Produits disponibles : en stock ET non discontinués
db.products.find({
    stock: { $gt: 0 },
    discontinued: false
})

// Produits attractifs : en promotion OU nouveaux OU bien notés
db.products.find({
    $or: [
        { onSale: true },
        { isNew: true },
        { rating: { $gte: 4.5 } }
    ]
})

// Produits abordables : prix < 100 OU en promotion
db.products.find({
    $or: [
        { price: { $lt: 100 } },
        { onSale: true }
    ]
})

// Produits éligibles livraison gratuite :
// (Electronics OU Books) ET prix > 50 ET en stock
db.products.find({
    $or: [
        { category: "Electronics" },
        { category: "Books" }
    ],
    price: { $gt: 50 },
    stock: { $gt: 0 }
})

// Exclusion : produits NI en rupture NI discontinués
db.products.find({
    $nor: [
        { stock: { $lte: 0 } },
        { discontinued: true }
    ]
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
    priority: "normal",
    orderDate: ISODate("2024-01-15T10:30:00Z"),
    shippingMethod: "standard"
}
```

**Requêtes** :

```javascript
// Commandes à traiter en priorité :
// (urgentes OU express) ET non complétées
db.orders.find({
    $or: [
        { priority: "urgent" },
        { shippingMethod: "express" }
    ],
    status: { $ne: "completed" }
})

// Commandes problématiques :
// (anciennes ET non livrées) OU (montant élevé ET en retard)
db.orders.find({
    $or: [
        {
            $and: [
                { orderDate: { $lt: ISODate("2024-01-01") } },
                { status: { $ne: "delivered" } }
            ]
        },
        {
            $and: [
                { amount: { $gt: 500 } },
                { status: "late" }
            ]
        }
    ]
})

// Commandes valides : NI annulées NI remboursées NI frauduleuses
db.orders.find({
    $nor: [
        { status: "cancelled" },
        { status: "refunded" },
        { flagged: "fraud" }
    ]
})

// Commandes à facturer :
// complétées ET (montant > 100 OU client entreprise)
db.orders.find({
    status: "completed",
    $or: [
        { amount: { $gt: 100 } },
        { customerType: "business" }
    ]
})
```

---

## Comparaison avec SQL

Pour les développeurs venant du SQL, voici les équivalences :

| SQL | MongoDB |
|-----|---------|
| `WHERE a = 1 AND b = 2` | `{ a: 1, b: 2 }` (AND implicite) |
| `WHERE a = 1 AND b = 2` | `{ $and: [{ a: 1 }, { b: 2 }] }` (AND explicite) |
| `WHERE a = 1 OR b = 2` | `{ $or: [{ a: 1 }, { b: 2 }] }` |
| `WHERE NOT (a = 1)` | `{ a: { $not: { $eq: 1 } } }` |
| `WHERE a != 1 AND b != 2` | `{ $nor: [{ a: 1 }, { b: 2 }] }` |
| `WHERE (a = 1 OR a = 2) AND b = 3` | `{ $or: [{ a: 1 }, { a: 2 }], b: 3 }` |
| `WHERE a IN (1,2,3) OR b > 10` | `{ $or: [{ a: { $in: [1,2,3] } }, { b: { $gt: 10 } }] }` |

### Exemples de Conversion SQL → MongoDB

#### SQL Exemple 1
```sql
SELECT * FROM products
WHERE (category = 'Electronics' OR category = 'Books')
  AND price < 100
  AND stock > 0;
```

**MongoDB** :
```javascript
db.products.find({
    $or: [
        { category: "Electronics" },
        { category: "Books" }
    ],
    price: { $lt: 100 },
    stock: { $gt: 0 }
})
```

#### SQL Exemple 2
```sql
SELECT * FROM users
WHERE age >= 18
  AND (verified = true OR membershipLevel = 'VIP')
  AND status NOT IN ('banned', 'deleted');
```

**MongoDB** :
```javascript
db.users.find({
    age: { $gte: 18 },
    $or: [
        { verified: true },
        { membershipLevel: "VIP" }
    ],
    status: { $nin: ["banned", "deleted"] }
})
```

---

## Bonnes Pratiques

### 1. Privilégier la Syntaxe Implicite pour `$and`

```javascript
// ✅ Préféré : AND implicite
db.users.find({
    status: "active",
    age: { $gte: 18 }
})

// ⚠️ Verbose : AND explicite (inutile ici)
db.users.find({
    $and: [
        { status: "active" },
        { age: { $gte: 18 } }
    ]
})
```

### 2. Utiliser `$in` au Lieu de `$or` pour le Même Champ

```javascript
// ❌ Moins optimal
db.users.find({
    $or: [
        { status: "active" },
        { status: "trial" },
        { status: "pending" }
    ]
})

// ✅ Meilleur
db.users.find({
    status: { $in: ["active", "trial", "pending"] }
})
```

### 3. Créer des Index pour les Requêtes Complexes

```javascript
// Pour des requêtes fréquentes avec $or
db.products.createIndex({ category: 1, price: 1 })
db.products.createIndex({ onSale: 1 })

// La requête bénéficiera des index
db.products.find({
    $or: [
        { category: "Electronics" },
        { onSale: true }
    ],
    price: { $lt: 500 }
})
```

### 4. Simplifier la Logique Quand Possible

```javascript
// ❌ Complexe
db.users.find({
    $and: [
        { age: { $not: { $lt: 18 } } }
    ]
})

// ✅ Simplifié
db.users.find({
    age: { $gte: 18 }
})
```

### 5. Placer les Filtres Sélectifs en Premier

```javascript
// ✅ Bon : filtre sélectif d'abord
db.products.find({
    sku: "PROD-12345", // Très sélectif (peu de résultats)
    $or: [
        { category: "Electronics" },
        { onSale: true }
    ]
})

// ⚠️ Moins optimal : filtres larges d'abord
db.products.find({
    $or: [
        { category: "Electronics" },
        { onSale: true }
    ],
    sku: "PROD-12345"
})
```

### 6. Utiliser `explain()` pour Vérifier les Performances

```javascript
// Analyser la requête
db.products.find({
    $or: [
        { category: "Electronics" },
        { price: { $lt: 50 } }
    ]
}).explain("executionStats")
```

---

## Pièges Courants à Éviter

### 1. Confusion entre `$nin` et `$nor`

```javascript
// ❌ Incorrect : $nin pour plusieurs champs
db.products.find({
    $nin: [
        { category: "Electronics" },
        { onSale: true }
    ]
})
// Ceci est invalide !

// ✅ Correct : utiliser $nor pour plusieurs champs
db.products.find({
    $nor: [
        { category: "Electronics" },
        { onSale: true }
    ]
})

// ✅ Correct : utiliser $nin pour un seul champ
db.products.find({
    category: { $nin: ["Electronics", "Toys"] }
})
```

### 2. Oublier les Crochets avec `$or`

```javascript
// ❌ Incorrect : objet au lieu de tableau
db.users.find({
    $or: {
        { status: "active" },
        { verified: true }
    }
})

// ✅ Correct : tableau de conditions
db.users.find({
    $or: [
        { status: "active" },
        { verified: true }
    ]
})
```

### 3. Logique `$not` Mal Appliquée

```javascript
// ❌ Incorrect : $not sur une valeur directe
db.users.find({
    age: { $not: 18 }
})

// ✅ Correct : $not sur une expression
db.users.find({
    age: { $not: { $eq: 18 } }
})
// Ou simplement :
db.users.find({
    age: { $ne: 18 }
})
```

### 4. Surcharge de Conditions `$or`

Trop de conditions `$or` peuvent dégrader les performances :

```javascript
// ⚠️ Attention : beaucoup de conditions
db.products.find({
    $or: [
        { category: "Electronics" },
        { category: "Books" },
        { category: "Toys" },
        { category: "Clothing" },
        { category: "Sports" },
        // ... 20 autres catégories
    ]
})

// ✅ Meilleur : utiliser $in
db.products.find({
    category: { $in: ["Electronics", "Books", "Toys", ...] }
})
```

---

## Performance et Optimisation

### Impact des Opérateurs Logiques

Les opérateurs logiques peuvent avoir un impact significatif sur les performances :

#### `$and` (Implicite)
- Très performant avec des index
- MongoDB peut utiliser plusieurs index

#### `$or`
- Peut être coûteux sans index appropriés
- MongoDB doit évaluer chaque condition séparément
- Créer des index sur tous les champs utilisés dans `$or`

#### `$not`
- Ne peut pas utiliser d'index directement
- Peut forcer un scan complet de la collection (COLLSCAN)
- Utiliser avec précaution sur de grandes collections

#### `$nor`
- Similaire à `$not` en termes de performance
- Ne peut pas utiliser d'index efficacement
- Éviter sur de très grandes collections si possible

### Optimisation avec Index

```javascript
// Créer des index composés pour des requêtes fréquentes
db.products.createIndex({ category: 1, price: 1, stock: 1 })

// Cette requête utilisera l'index efficacement
db.products.find({
    category: "Electronics",
    price: { $lt: 500 },
    stock: { $gt: 0 }
})

// Pour $or, créer des index sur chaque champ
db.products.createIndex({ category: 1 })
db.products.createIndex({ onSale: 1 })

db.products.find({
    $or: [
        { category: "Electronics" },
        { onSale: true }
    ]
})
```

### Vérification avec `explain()`

```javascript
// Analyser une requête complexe
db.products.find({
    $or: [
        { category: "Electronics" },
        { price: { $lt: 50 } }
    ],
    stock: { $gt: 0 }
}).explain("executionStats")

// Vérifier :
// - "executionTimeMillis" : temps d'exécution
// - "totalDocsExamined" : nombre de documents examinés
// - "stage" : "IXSCAN" (index) vs "COLLSCAN" (scan complet)
```

---

## Points Clés à Retenir

✅ **`$and`** combine des conditions avec un ET logique (toutes doivent être vraies)

✅ **`$or`** combine des conditions avec un OU logique (au moins une doit être vraie)

✅ **`$not`** inverse le résultat d'une expression

✅ **`$nor`** vérifie qu'aucune condition n'est vraie (NI...NI)

✅ MongoDB applique un **`$and` implicite** entre les champs d'un même document

✅ Utilisez **`$in`** au lieu de **`$or`** pour plusieurs valeurs du même champ

✅ Les opérateurs logiques peuvent être **imbriqués** pour créer des requêtes complexes

✅ **`$not`** et **`$nor`** incluent les documents où les champs n'existent pas

✅ Les **index** sont cruciaux pour les performances des requêtes avec opérateurs logiques

✅ Utilisez **`explain()`** pour analyser et optimiser vos requêtes

---

## Résumé

Dans ce chapitre, vous avez appris :

- Les quatre opérateurs logiques principaux de MongoDB (`$and`, `$or`, `$not`, `$nor`)
- Comment combiner plusieurs conditions avec la logique booléenne
- La différence entre AND implicite et explicite
- Quand utiliser `$or` vs `$in`
- Comment imbriquer les opérateurs pour des requêtes complexes
- Les équivalences avec SQL
- Les bonnes pratiques d'optimisation
- Les pièges courants à éviter

Ces opérateurs logiques, combinés avec les opérateurs de comparaison du chapitre précédent, vous donnent un contrôle total sur vos requêtes MongoDB. Dans le prochain chapitre, nous explorerons les **opérateurs d'éléments** qui vous permettront de vérifier l'existence de champs et leurs types.

---


⏭️ [Opérateurs d'éléments ($exists, $type)](/03-requetes-et-filtres/04-operateurs-elements.md)
