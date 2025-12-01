🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.7 Projections : Sélection des Champs

## Introduction

Jusqu'à présent, nous avons appris à **filtrer** les documents selon différents critères. Mais une fois que MongoDB a trouvé les documents correspondants, par défaut, il retourne **tous les champs** de ces documents.

Imaginons que vous ayez un document utilisateur avec 20 champs (nom, email, adresse, historique d'achats, préférences, etc.), mais que vous n'ayez besoin que du nom et de l'email. Récupérer tous les champs serait un gaspillage de bande passante et de mémoire.

C'est là qu'interviennent les **projections**. Une projection vous permet de spécifier **quels champs** vous souhaitez inclure ou exclure dans les résultats de vos requêtes.

Les projections offrent plusieurs avantages :
- **Réduction de la bande passante** : moins de données transférées sur le réseau
- **Amélioration des performances** : moins de données à traiter
- **Sécurité** : ne pas exposer des champs sensibles
- **Clarté du code** : récupérer uniquement ce dont vous avez besoin

---

## Syntaxe de Base

Les projections sont le **deuxième paramètre** de la méthode `find()` :

```javascript
db.collection.find(
    { filtres },      // Premier paramètre : critères de recherche
    { projection }    // Deuxième paramètre : champs à retourner
)
```

### Format de la Projection

Une projection est un document où :
- **`1`** ou **`true`** : inclure le champ
- **`0`** ou **`false`** : exclure le champ

```javascript
// Format général
{
    champ1: 1,  // Inclure
    champ2: 1,  // Inclure
    champ3: 0   // Exclure
}
```

---

## Inclusion de Champs

L'inclusion spécifie les champs que vous **voulez** dans les résultats. Tous les autres champs (sauf `_id`) sont automatiquement exclus.

### Exemples de Base

```javascript
// Document complet
{
    _id: ObjectId("..."),
    name: "Alice",
    email: "alice@example.com",
    age: 30,
    address: "123 Main St",
    phone: "555-0123"
}

// Inclure seulement name et email
db.users.find(
    {},
    { name: 1, email: 1 }
)

// Résultat
{
    _id: ObjectId("..."),  // _id est inclus par défaut
    name: "Alice",
    email: "alice@example.com"
}
```

### Syntaxe Alternative avec `true`

Vous pouvez utiliser `true` au lieu de `1` (équivalent) :

```javascript
// Ces deux projections sont identiques
db.users.find({}, { name: 1, email: 1 })
db.users.find({}, { name: true, email: true })
```

### Exemples Pratiques

```javascript
// Ne récupérer que le nom et le prix des produits
db.products.find(
    {},
    { name: 1, price: 1 }
)

// Récupérer seulement le titre et la date des articles
db.articles.find(
    { status: "published" },
    { title: 1, publishedDate: 1 }
)

// Récupérer le nom et le statut des commandes
db.orders.find(
    { customerId: ObjectId("...") },
    { orderNumber: 1, status: 1 }
)
```

---

## Exclusion de Champs

L'exclusion spécifie les champs que vous **ne voulez pas** dans les résultats. Tous les autres champs sont automatiquement inclus.

### Exemples de Base

```javascript
// Exclure seulement le champ password
db.users.find(
    {},
    { password: 0 }
)

// Résultat : tous les champs sauf password
{
    _id: ObjectId("..."),
    name: "Alice",
    email: "alice@example.com",
    age: 30,
    address: "123 Main St",
    phone: "555-0123"
    // password n'est pas inclus
}
```

### Syntaxe Alternative avec `false`

```javascript
// Ces deux projections sont identiques
db.users.find({}, { password: 0 })
db.users.find({}, { password: false })
```

### Exclure Plusieurs Champs

```javascript
// Exclure les champs sensibles
db.users.find(
    {},
    { password: 0, creditCard: 0, ssn: 0 }
)

// Exclure les métadonnées internes
db.products.find(
    {},
    { internalNotes: 0, cost: 0, supplierId: 0 }
)

// Exclure les champs volumineux
db.articles.find(
    {},
    { content: 0, rawHtml: 0 }
)
```

---

## Le Champ Spécial `_id`

Par défaut, le champ `_id` est **toujours inclus**, même si vous ne le spécifiez pas dans votre projection.

### Inclusion Automatique de `_id`

```javascript
// Sans mentionner _id
db.users.find({}, { name: 1, email: 1 })

// Résultat : _id est automatiquement inclus
{
    _id: ObjectId("..."),
    name: "Alice",
    email: "alice@example.com"
}
```

### Exclure `_id` Explicitement

Pour exclure `_id`, vous devez le spécifier explicitement :

```javascript
// Exclure _id
db.users.find(
    {},
    { name: 1, email: 1, _id: 0 }
)

// Résultat : _id n'est pas inclus
{
    name: "Alice",
    email: "alice@example.com"
}
```

### `_id` est l'Exception

`_id` est le **seul champ** que vous pouvez exclure dans une projection d'inclusion :

```javascript
// ✅ Valide : exclure _id dans une inclusion
db.users.find({}, { name: 1, email: 1, _id: 0 })

// ❌ Invalide : mélanger inclusion et exclusion (sauf pour _id)
db.users.find({}, { name: 1, password: 0 })
// Erreur : ne peut pas mélanger inclusion et exclusion
```

---

## Règle : Inclusion OU Exclusion (pas les deux)

**Important** : Vous ne pouvez pas mélanger inclusion et exclusion dans la même projection (sauf pour `_id`).

### Règles

1. **Soit inclusion** : spécifier les champs à inclure (`1`)
2. **Soit exclusion** : spécifier les champs à exclure (`0`)
3. **Exception** : `_id` peut être exclu dans une projection d'inclusion

### Exemples Valides

```javascript
// ✅ Inclusion uniquement
db.users.find({}, { name: 1, email: 1 })

// ✅ Inclusion avec exclusion de _id
db.users.find({}, { name: 1, email: 1, _id: 0 })

// ✅ Exclusion uniquement
db.users.find({}, { password: 0, ssn: 0 })

// ✅ Exclusion incluant _id
db.users.find({}, { password: 0, _id: 0 })
```

### Exemples Invalides

```javascript
// ❌ Mélange inclusion/exclusion (erreur)
db.users.find({}, { name: 1, password: 0 })

// ❌ Mélange inclusion/exclusion (erreur)
db.products.find({}, { name: 1, price: 1, internalNotes: 0 })
```

**Exception** : Seul `_id` peut être mélangé avec des inclusions.

---

## Projections sur Champs Imbriqués

Les projections fonctionnent également sur des champs imbriqués en utilisant la **notation pointée**.

### Inclusion de Champs Imbriqués

```javascript
// Document
{
    _id: 1,
    name: "Alice",
    address: {
        street: "123 Main St",
        city: "Paris",
        zipCode: "75001",
        country: "France"
    },
    contact: {
        email: "alice@example.com",
        phone: "555-0123"
    }
}

// Inclure seulement la ville
db.users.find(
    {},
    { name: 1, "address.city": 1 }
)

// Résultat
{
    _id: 1,
    name: "Alice",
    address: {
        city: "Paris"
    }
}
```

### Inclure un Sous-document Entier

```javascript
// Inclure tout le sous-document address
db.users.find(
    {},
    { name: 1, address: 1 }
)

// Résultat : tout le sous-document address est inclus
{
    _id: 1,
    name: "Alice",
    address: {
        street: "123 Main St",
        city: "Paris",
        zipCode: "75001",
        country: "France"
    }
}
```

### Exclure des Champs Imbriqués

```javascript
// Exclure un champ imbriqué
db.users.find(
    {},
    { "address.street": 0 }
)

// Résultat : tous les champs sauf address.street
{
    _id: 1,
    name: "Alice",
    address: {
        city: "Paris",
        zipCode: "75001",
        country: "France"
    },
    contact: {
        email: "alice@example.com",
        phone: "555-0123"
    }
}
```

### Projections Multiples sur Sous-documents

```javascript
// Inclure plusieurs champs imbriqués
db.users.find(
    {},
    {
        name: 1,
        "address.city": 1,
        "address.country": 1,
        "contact.email": 1
    }
)

// Résultat
{
    _id: 1,
    name: "Alice",
    address: {
        city: "Paris",
        country: "France"
    },
    contact: {
        email: "alice@example.com"
    }
}
```

---

## Projections sur Tableaux

Les projections sur les tableaux nécessitent une attention particulière.

### Inclusion Complète d'un Tableau

```javascript
// Document
{
    _id: 1,
    name: "Alice",
    hobbies: ["reading", "swimming", "coding"],
    scores: [85, 92, 78, 95]
}

// Inclure le tableau complet
db.users.find({}, { name: 1, hobbies: 1 })

// Résultat : tout le tableau est inclus
{
    _id: 1,
    name: "Alice",
    hobbies: ["reading", "swimming", "coding"]
}
```

### Tableaux d'Objets

```javascript
// Document
{
    _id: 1,
    productName: "Laptop",
    reviews: [
        { author: "John", rating: 5, comment: "Great!" },
        { author: "Jane", rating: 4, comment: "Good" },
        { author: "Bob", rating: 3, comment: "OK" }
    ]
}

// Inclure tout le tableau reviews
db.products.find({}, { productName: 1, reviews: 1 })

// Résultat : tous les reviews avec tous leurs champs
{
    _id: 1,
    productName: "Laptop",
    reviews: [
        { author: "John", rating: 5, comment: "Great!" },
        { author: "Jane", rating: 4, comment: "Good" },
        { author: "Bob", rating: 3, comment: "OK" }
    ]
}
```

### Projections sur Champs de Tableaux d'Objets

```javascript
// Inclure seulement certains champs des éléments du tableau
db.products.find(
    {},
    {
        productName: 1,
        "reviews.author": 1,
        "reviews.rating": 1
    }
)

// Résultat
{
    _id: 1,
    productName: "Laptop",
    reviews: [
        { author: "John", rating: 5 },
        { author: "Jane", rating: 4 },
        { author: "Bob", rating: 3 }
    ]
    // Les champs "comment" sont exclus
}
```

---

## Opérateurs de Projection

MongoDB fournit des opérateurs spéciaux pour manipuler les projections, notamment sur les tableaux.

### L'Opérateur `$elemMatch` (dans les projections)

L'opérateur `$elemMatch` dans une projection permet de ne retourner que les **premiers éléments** d'un tableau qui correspondent à une condition.

#### Syntaxe

```javascript
{
    champ: { $elemMatch: { condition } }
}
```

#### Exemples

```javascript
// Document
{
    _id: 1,
    productName: "Laptop",
    reviews: [
        { author: "John", rating: 5, verified: true },
        { author: "Jane", rating: 4, verified: false },
        { author: "Bob", rating: 3, verified: true }
    ]
}

// Ne retourner que les reviews vérifiés et bien notés
db.products.find(
    { _id: 1 },
    {
        productName: 1,
        reviews: {
            $elemMatch: {
                rating: { $gte: 4 },
                verified: true
            }
        }
    }
)

// Résultat : seulement le PREMIER review correspondant
{
    _id: 1,
    productName: "Laptop",
    reviews: [
        { author: "John", rating: 5, verified: true }
    ]
}
```

**Important** : `$elemMatch` dans une projection ne retourne que le **premier élément** correspondant, pas tous.

### L'Opérateur `$slice`

L'opérateur `$slice` permet de limiter le nombre d'éléments retournés dans un tableau.

#### Syntaxe

```javascript
{ champ: { $slice: nombre } }
// ou
{ champ: { $slice: [skip, limit] } }
```

#### Exemples avec Nombre Simple

```javascript
// Document
{
    _id: 1,
    title: "Article",
    comments: [
        { text: "Comment 1" },
        { text: "Comment 2" },
        { text: "Comment 3" },
        { text: "Comment 4" },
        { text: "Comment 5" }
    ]
}

// Retourner les 2 premiers commentaires
db.articles.find(
    { _id: 1 },
    { title: 1, comments: { $slice: 2 } }
)

// Résultat
{
    _id: 1,
    title: "Article",
    comments: [
        { text: "Comment 1" },
        { text: "Comment 2" }
    ]
}

// Retourner les 2 derniers commentaires (nombre négatif)
db.articles.find(
    { _id: 1 },
    { title: 1, comments: { $slice: -2 } }
)

// Résultat
{
    _id: 1,
    title: "Article",
    comments: [
        { text: "Comment 4" },
        { text: "Comment 5" }
    ]
}
```

#### Exemples avec Skip et Limit

```javascript
// Sauter 1 élément et prendre les 2 suivants
db.articles.find(
    { _id: 1 },
    { title: 1, comments: { $slice: [1, 2] } }
)

// Résultat
{
    _id: 1,
    title: "Article",
    comments: [
        { text: "Comment 2" },
        { text: "Comment 3" }
    ]
}

// Sauter 2 éléments et prendre les 3 suivants
db.articles.find(
    { _id: 1 },
    { title: 1, comments: { $slice: [2, 3] } }
)
```

### L'Opérateur `$` (Opérateur Positionnel)

L'opérateur `$` dans une projection retourne le **premier élément** du tableau qui a correspondu dans la requête.

#### Syntaxe

```javascript
{ "champ.$": 1 }
```

#### Exemple

```javascript
// Documents
{
    _id: 1,
    scores: [45, 78, 92, 88]
}

// Trouver et projeter seulement le premier score >= 80
db.students.find(
    { scores: { $gte: 80 } },
    { "scores.$": 1 }
)

// Résultat : seulement le premier score qui correspond
{
    _id: 1,
    scores: [92]
}
```

**Note** : Le `$` positionnel nécessite que le tableau soit utilisé dans la requête.

### L'Opérateur `$meta`

L'opérateur `$meta` est utilisé pour projeter des métadonnées, notamment le score de recherche textuelle.

```javascript
// Recherche textuelle avec score
db.articles.find(
    { $text: { $search: "mongodb tutorial" } },
    {
        title: 1,
        score: { $meta: "textScore" }
    }
).sort({ score: { $meta: "textScore" } })

// Résultat avec scores
{
    _id: 1,
    title: "MongoDB Tutorial for Beginners",
    score: 1.5
}
```

---

## Cas d'Usage Pratiques

### Cas 1 : API REST - Limiter les Données Exposées

```javascript
// Ne jamais exposer les données sensibles
db.users.find(
    { status: "active" },
    {
        name: 1,
        email: 1,
        profilePicture: 1,
        _id: 0,
        // Exclure password, ssn, creditCard, etc. par omission
    }
)
```

### Cas 2 : Liste de Produits - Données Minimales

```javascript
// Pour une liste de produits, ne récupérer que l'essentiel
db.products.find(
    { category: "Electronics" },
    {
        name: 1,
        price: 1,
        thumbnail: 1,
        rating: 1
    }
)
```

### Cas 3 : Détails Complets - Exclure Métadonnées

```javascript
// Page de détails : tout sauf les métadonnées internes
db.products.find(
    { _id: ObjectId("...") },
    {
        internalNotes: 0,
        supplierCost: 0,
        lastModifiedBy: 0
    }
)
```

### Cas 4 : Tableaux de Bord - Agrégations Simples

```javascript
// Statistiques : seulement les champs nécessaires
db.orders.find(
    {
        status: "completed",
        orderDate: { $gte: ISODate("2024-01-01") }
    },
    {
        orderNumber: 1,
        amount: 1,
        orderDate: 1,
        customerId: 1,
        _id: 0
    }
)
```

### Cas 5 : Commentaires - Pagination avec `$slice`

```javascript
// Afficher les 10 premiers commentaires
db.articles.find(
    { _id: ObjectId("...") },
    {
        title: 1,
        content: 1,
        comments: { $slice: 10 }
    }
)

// Page 2 : sauter 10, prendre 10
db.articles.find(
    { _id: ObjectId("...") },
    {
        title: 1,
        comments: { $slice: [10, 10] }
    }
)
```

### Cas 6 : Recherche - Premiers Résultats

```javascript
// Recherche avec reviews limitées
db.products.find(
    { $text: { $search: "laptop" } },
    {
        name: 1,
        price: 1,
        rating: 1,
        reviews: { $slice: 3 },  // Seulement 3 reviews
        score: { $meta: "textScore" }
    }
).sort({ score: { $meta: "textScore" } })
```

---

## Projections avec `findOne()`

Les projections fonctionnent de la même manière avec `findOne()` :

```javascript
// Récupérer un utilisateur avec seulement certains champs
db.users.findOne(
    { email: "alice@example.com" },
    { name: 1, email: 1, status: 1, _id: 0 }
)

// Résultat
{
    name: "Alice",
    email: "alice@example.com",
    status: "active"
}
```

---

## Projections dans les Pipelines d'Agrégation

Dans les pipelines d'agrégation, utilisez l'étape `$project` :

```javascript
db.products.aggregate([
    { $match: { category: "Electronics" } },
    {
        $project: {
            name: 1,
            price: 1,
            discount: 1,
            finalPrice: { $subtract: ["$price", "$discount"] }
        }
    }
])
```

Les projections dans les agrégations sont plus puissantes et seront couvertes en détail dans le chapitre 6.

---

## Comparaison avec SQL

Les projections MongoDB sont similaires à la clause `SELECT` en SQL :

| SQL | MongoDB |
|-----|---------|
| `SELECT name, email FROM users` | `db.users.find({}, { name: 1, email: 1 })` |
| `SELECT * FROM users` | `db.users.find({})` |
| `SELECT name, email FROM users WHERE age >= 18` | `db.users.find({ age: { $gte: 18 } }, { name: 1, email: 1 })` |
| `SELECT name FROM users` (sans id) | `db.users.find({}, { name: 1, _id: 0 })` |

---

## Bonnes Pratiques

### 1. Toujours Utiliser des Projections en Production

```javascript
// ❌ Éviter : récupérer tous les champs
db.users.find({ status: "active" })

// ✅ Bon : spécifier les champs nécessaires
db.users.find(
    { status: "active" },
    { name: 1, email: 1, profilePicture: 1 }
)
```

### 2. Exclure les Champs Sensibles

```javascript
// ✅ Toujours exclure les données sensibles pour les API
db.users.find(
    {},
    { password: 0, ssn: 0, creditCard: 0 }
)
```

### 3. Optimiser la Bande Passante

```javascript
// ✅ Pour les listes : seulement l'essentiel
db.products.find(
    { category: "Electronics" },
    { name: 1, price: 1, thumbnail: 1 }
)

// ✅ Pour les détails : tout sauf les métadonnées
db.products.find(
    { _id: ObjectId("...") },
    { internalNotes: 0, supplierCost: 0 }
)
```

### 4. Utiliser `$slice` pour les Grands Tableaux

```javascript
// ✅ Limiter les tableaux volumineux
db.articles.find(
    {},
    {
        title: 1,
        summary: 1,
        comments: { $slice: 5 }  // Seulement 5 commentaires
    }
)
```

### 5. Exclure `_id` Quand Non Nécessaire

```javascript
// ✅ Si _id n'est pas nécessaire, l'exclure
db.users.find(
    {},
    { name: 1, email: 1, _id: 0 }
)
```

### 6. Documenter les Projections Complexes

```javascript
// ✅ Ajouter des commentaires pour les projections complexes
db.products.find(
    { category: "Electronics" },
    {
        // Informations de base
        name: 1,
        price: 1,

        // Image principale seulement
        "images.main": 1,

        // Premiers 3 reviews
        reviews: { $slice: 3 },

        // Exclure les métadonnées
        _id: 0
    }
)
```

### 7. Projections et Index Couvrants

```javascript
// Créer un index couvrant
db.users.createIndex({ status: 1, name: 1, email: 1 })

// Requête couverte par l'index (très rapide)
db.users.find(
    { status: "active" },
    { name: 1, email: 1, _id: 0 }
)
// MongoDB peut répondre uniquement depuis l'index sans lire les documents
```

---

## Pièges Courants à Éviter

### 1. Mélanger Inclusion et Exclusion

```javascript
// ❌ Erreur : mélange inclusion et exclusion
db.users.find({}, { name: 1, password: 0 })

// ✅ Correct : inclusion uniquement
db.users.find({}, { name: 1, email: 1 })

// ✅ Correct : exclusion uniquement
db.users.find({}, { password: 0, ssn: 0 })
```

### 2. Oublier d'Exclure `_id`

```javascript
// ⚠️ _id est toujours inclus par défaut
db.users.find({}, { name: 1, email: 1 })
// Retourne _id, name, email

// ✅ Exclure _id explicitement si non désiré
db.users.find({}, { name: 1, email: 1, _id: 0 })
```

### 3. Projections sur Tableaux Imbriqués

```javascript
// ❌ Incomplet : ne projette pas correctement
db.products.find(
    {},
    { "reviews.rating": 1 }
)
// Retourne tout le tableau reviews, pas seulement les ratings

// ✅ Pour filtrer les éléments : utiliser $elemMatch
db.products.find(
    {},
    {
        name: 1,
        reviews: {
            $elemMatch: { rating: { $gte: 4 } }
        }
    }
)
```

### 4. Performances avec Grands Documents

```javascript
// ⚠️ Sans projection : documents complets (lent)
db.articles.find({ status: "published" })

// ✅ Avec projection : seulement les champs nécessaires
db.articles.find(
    { status: "published" },
    { title: 1, summary: 1, publishedDate: 1 }
)
```

### 5. Projections Inutiles sur `findOne()`

```javascript
// ⚠️ Si un seul document : l'impact est faible
const user = db.users.findOne({ _id: ObjectId("...") })

// ✅ Mais toujours une bonne pratique
const user = db.users.findOne(
    { _id: ObjectId("...") },
    { name: 1, email: 1, status: 1 }
)
```

---

## Performance et Optimisation

### Impact des Projections sur les Performances

| Aspect | Sans Projection | Avec Projection |
|--------|-----------------|-----------------|
| Données transférées | Toutes | Seulement nécessaires |
| Bande passante | Élevée | Réduite |
| Mémoire serveur | Élevée | Réduite |
| Temps de traitement | Plus long | Plus court |
| Utilisation index | Standard | Peut être couvrante |

### Requêtes Couvertes par Index

Une **requête couverte** est une requête où toutes les informations nécessaires sont dans l'index, MongoDB n'a pas besoin de lire les documents :

```javascript
// Créer un index couvrant
db.users.createIndex({ status: 1, name: 1, email: 1 })

// Requête couverte
db.users.find(
    { status: "active" },
    { name: 1, email: 1, _id: 0 }  // _id: 0 est important !
)
// Très rapide : MongoDB lit seulement l'index
```

**Important** : Pour qu'une requête soit couverte, vous devez exclure `_id` (sauf si `_id` fait partie de l'index).

### Vérification avec `explain()`

```javascript
// Analyser une requête avec projection
db.users.find(
    { status: "active" },
    { name: 1, email: 1, _id: 0 }
).explain("executionStats")

// Chercher "PROJECTION_COVERED" dans le plan d'exécution
// ou "totalDocsExamined": 0 (aucun document lu)
```

### Optimisation des Tableaux

```javascript
// ⚠️ Lent : récupère tous les commentaires
db.articles.find(
    { category: "Tech" },
    { title: 1, comments: 1 }
)

// ✅ Rapide : limite les commentaires
db.articles.find(
    { category: "Tech" },
    { title: 1, comments: { $slice: 10 } }
)
```

---

## Projections Dynamiques dans les Applications

Dans vos applications, vous pouvez construire des projections dynamiquement :

### Exemple en JavaScript (Node.js)

```javascript
// Construire une projection dynamique
function buildProjection(fields) {
    const projection = {};
    fields.forEach(field => {
        projection[field] = 1;
    });
    return projection;
}

// Utilisation
const fieldsToReturn = ['name', 'email', 'age'];
const projection = buildProjection(fieldsToReturn);

db.users.find({ status: 'active' }, projection);
// Équivalent à : { name: 1, email: 1, age: 1 }
```

### Exemple avec Paramètres API

```javascript
// API : GET /users?fields=name,email,age
app.get('/users', async (req, res) => {
    const fields = req.query.fields;

    let projection = {};
    if (fields) {
        projection = buildProjection(fields.split(','));
    }

    const users = await db.collection('users')
        .find({ status: 'active' }, projection)
        .toArray();

    res.json(users);
});
```

---

## Points Clés à Retenir

✅ Les **projections** contrôlent quels champs sont retournés dans les résultats

✅ **`1` ou `true`** : inclure le champ ; **`0` ou `false`** : exclure le champ

✅ Le champ **`_id` est toujours inclus** par défaut (sauf exclusion explicite)

✅ Vous ne pouvez pas **mélanger inclusion et exclusion** (sauf pour `_id`)

✅ Les projections fonctionnent sur **champs imbriqués** avec la notation pointée

✅ **`$slice`** limite le nombre d'éléments dans un tableau

✅ **`$elemMatch`** dans une projection retourne le premier élément correspondant

✅ Les projections **améliorent les performances** en réduisant les données transférées

✅ Les **requêtes couvertes** (covered queries) sont très rapides avec les bonnes projections

✅ Toujours **exclure les données sensibles** dans les API publiques

---

## Résumé

Dans ce chapitre, vous avez appris :

- Comment utiliser les projections pour sélectionner des champs spécifiques
- La différence entre inclusion et exclusion de champs
- Le comportement spécial du champ `_id`
- Comment projeter des champs imbriqués et des tableaux
- Les opérateurs de projection : `$elemMatch`, `$slice`, `$`, `$meta`
- Les cas d'usage pratiques dans des applications réelles
- Les bonnes pratiques pour optimiser les performances
- Comment créer des requêtes couvertes par index
- Les pièges courants à éviter

Les projections sont un outil essentiel pour optimiser vos requêtes MongoDB. Elles vous permettent de réduire la bande passante, d'améliorer les performances, et de contrôler précisément les données exposées par vos applications.

Dans le prochain chapitre, nous explorerons le **tri, la limitation et la pagination** des résultats pour un contrôle encore plus fin sur vos requêtes.

---


⏭️ [Tri, limite et pagination (sort, limit, skip)](/03-requetes-et-filtres/08-tri-limite-pagination.md)
