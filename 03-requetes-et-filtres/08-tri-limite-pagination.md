🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.8 Tri, Limite et Pagination

## Introduction

Lorsque vous récupérez des documents de MongoDB, vous avez souvent besoin de :
- **Trier** les résultats dans un ordre spécifique (alphabétique, numérique, par date, etc.)
- **Limiter** le nombre de résultats retournés
- **Paginer** les résultats pour afficher des pages successives de données

Par exemple :
- Afficher les 10 produits les plus récents
- Lister les utilisateurs par ordre alphabétique
- Afficher la page 2 d'une liste de 100 articles (éléments 11 à 20)
- Trouver les 5 commandes les plus chères

MongoDB fournit trois méthodes principales pour contrôler l'ordre et la quantité des résultats :
- **`sort()`** : trier les documents
- **`limit()`** : limiter le nombre de documents retournés
- **`skip()`** : sauter un certain nombre de documents

Dans ce chapitre, nous allons explorer ces trois méthodes et apprendre à les combiner efficacement.

---

## La Méthode `sort()`

La méthode `sort()` permet de trier les résultats d'une requête selon un ou plusieurs champs.

### Syntaxe

```javascript
db.collection.find(query).sort({ champ: ordre })
```

Où **`ordre`** peut être :
- **`1`** : ordre croissant (ascendant) - A→Z, 0→9, plus ancien→plus récent
- **`-1`** : ordre décroissant (descendant) - Z→A, 9→0, plus récent→plus ancien

### Tri Croissant

```javascript
// Trier les utilisateurs par nom (A → Z)
db.users.find().sort({ name: 1 })

// Trier les produits par prix (du moins cher au plus cher)
db.products.find().sort({ price: 1 })

// Trier les articles par date (du plus ancien au plus récent)
db.articles.find().sort({ publishedDate: 1 })
```

### Tri Décroissant

```javascript
// Trier les utilisateurs par nom (Z → A)
db.users.find().sort({ name: -1 })

// Trier les produits par prix (du plus cher au moins cher)
db.products.find().sort({ price: -1 })

// Trier les articles par date (du plus récent au plus ancien)
db.articles.find().sort({ publishedDate: -1 })
```

### Exemples Pratiques

```javascript
// Les 10 produits les plus chers
db.products.find().sort({ price: -1 }).limit(10)

// Les utilisateurs les plus récents
db.users.find().sort({ registrationDate: -1 })

// Articles les plus consultés
db.articles.find().sort({ views: -1 })

// Commandes les plus anciennes en attente
db.orders.find({ status: "pending" }).sort({ orderDate: 1 })
```

---

## Tri sur Plusieurs Champs

Vous pouvez trier sur plusieurs champs en spécifiant plusieurs clés dans le document de tri. MongoDB trie d'abord par le premier champ, puis en cas d'égalité, par le second, etc.

### Syntaxe

```javascript
db.collection.find().sort({ champ1: ordre1, champ2: ordre2, ... })
```

### Exemples

```javascript
// Trier par catégorie (A→Z), puis par prix (croissant)
db.products.find().sort({ category: 1, price: 1 })

// Trier par statut (A→Z), puis par date (décroissant)
db.orders.find().sort({ status: 1, orderDate: -1 })

// Trier par note (décroissant), puis par nom (A→Z)
db.products.find().sort({ rating: -1, name: 1 })

// Trier par priorité (décroissant), puis par date limite (croissant)
db.tasks.find().sort({ priority: -1, dueDate: 1 })
```

### Ordre de Priorité

L'ordre dans lequel vous spécifiez les champs est important :

```javascript
// Documents
{ name: "Alice", age: 30, city: "Paris" }
{ name: "Bob", age: 25, city: "Paris" }
{ name: "Charlie", age: 25, city: "Lyon" }
{ name: "David", age: 25, city: "Paris" }

// Trier d'abord par age, puis par name
db.users.find().sort({ age: 1, name: 1 })

// Résultat :
// 1. Bob (age: 25, name: B)
// 2. Charlie (age: 25, name: C)
// 3. David (age: 25, name: D)
// 4. Alice (age: 30)
```

### Cas d'Usage : Classement avec Ex Aequo

```javascript
// Classement des joueurs par score, puis par temps
db.players.find().sort({ score: -1, completionTime: 1 })
// Plus haut score d'abord, et en cas d'égalité, le temps le plus rapide
```

---

## Tri sur Champs Imbriqués

Le tri fonctionne également sur les champs imbriqués en utilisant la **notation pointée**.

### Syntaxe

```javascript
db.collection.find().sort({ "champ.souschamp": ordre })
```

### Exemples

```javascript
// Trier par ville dans l'adresse
db.users.find().sort({ "address.city": 1 })

// Trier par code postal
db.stores.find().sort({ "location.zipCode": 1 })

// Trier par prix de vente dans les détails
db.products.find().sort({ "pricing.salePrice": 1 })
```

---

## Tri sur Tableaux

Lorsque vous triez sur un champ de type tableau, MongoDB utilise le plus petit ou le plus grand élément du tableau selon l'ordre de tri.

### Comportement

```javascript
// Documents
{ name: "Product A", ratings: [3, 5, 4] }
{ name: "Product B", ratings: [1, 2] }
{ name: "Product C", ratings: [5, 5, 5] }

// Tri croissant : utilise la plus petite valeur du tableau
db.products.find().sort({ ratings: 1 })
// Ordre : Product B (min: 1), Product A (min: 3), Product C (min: 5)

// Tri décroissant : utilise la plus grande valeur du tableau
db.products.find().sort({ ratings: -1 })
// Ordre : Product C (max: 5), Product A (max: 5), Product B (max: 2)
```

---

## La Méthode `limit()`

La méthode `limit()` restreint le nombre de documents retournés par une requête.

### Syntaxe

```javascript
db.collection.find().limit(nombre)
```

### Exemples de Base

```javascript
// Récupérer seulement 5 utilisateurs
db.users.find().limit(5)

// Les 10 premiers produits
db.products.find().limit(10)

// Un seul article
db.articles.find().limit(1)
```

### Combinaison avec Filtres

```javascript
// Les 20 premiers produits actifs
db.products.find({ status: "active" }).limit(20)

// Les 5 premières commandes complétées
db.orders.find({ status: "completed" }).limit(5)

// Les 10 premiers utilisateurs majeurs
db.users.find({ age: { $gte: 18 } }).limit(10)
```

### Cas d'Usage Pratiques

```javascript
// Top 10 des produits les plus vendus
db.products.find().sort({ salesCount: -1 }).limit(10)

// 5 derniers articles publiés
db.articles.find().sort({ publishedDate: -1 }).limit(5)

// 3 utilisateurs les plus actifs
db.users.find().sort({ activityScore: -1 }).limit(3)

// Page d'accueil : 8 produits vedettes
db.products.find({ featured: true }).limit(8)
```

---

## La Méthode `skip()`

La méthode `skip()` permet de sauter un certain nombre de documents dans les résultats.

### Syntaxe

```javascript
db.collection.find().skip(nombre)
```

### Exemples de Base

```javascript
// Sauter les 10 premiers documents
db.users.find().skip(10)

// Ignorer les 5 premières commandes
db.orders.find().skip(5)

// Commencer à partir du 21ème document
db.products.find().skip(20)
```

### Combinaison avec `limit()`

```javascript
// Sauter 10 documents, puis prendre 5
db.users.find().skip(10).limit(5)
// Retourne les documents 11 à 15

// Sauter 20, prendre 10
db.products.find().skip(20).limit(10)
// Retourne les documents 21 à 30
```

---

## Pagination

La pagination consiste à diviser un grand ensemble de résultats en plusieurs pages. C'est l'une des utilisations les plus courantes de `skip()` et `limit()`.

### Formule de Pagination

```javascript
const pageNumber = 1;  // Numéro de page (commence à 1)
const pageSize = 10;   // Nombre d'éléments par page

const skip = (pageNumber - 1) * pageSize;
const limit = pageSize;

db.collection.find().skip(skip).limit(limit)
```

### Exemples de Pagination

#### Page 1 (documents 1-10)

```javascript
db.products.find().skip(0).limit(10)
// skip = (1 - 1) * 10 = 0
// Documents 1 à 10
```

#### Page 2 (documents 11-20)

```javascript
db.products.find().skip(10).limit(10)
// skip = (2 - 1) * 10 = 10
// Documents 11 à 20
```

#### Page 3 (documents 21-30)

```javascript
db.products.find().skip(20).limit(10)
// skip = (3 - 1) * 10 = 20
// Documents 21 à 30
```

### Fonction de Pagination Réutilisable

```javascript
function paginate(collection, query, pageNumber, pageSize) {
    const skip = (pageNumber - 1) * pageSize;

    return collection
        .find(query)
        .skip(skip)
        .limit(pageSize);
}

// Utilisation
const page2 = paginate(db.products, { category: "Electronics" }, 2, 20);
```

### Pagination avec Tri

```javascript
// Page 2 des produits triés par prix
const page = 2;
const size = 10;

db.products
    .find({ category: "Electronics" })
    .sort({ price: 1 })
    .skip((page - 1) * size)
    .limit(size)
```

### Informations de Pagination

Pour une pagination complète, vous avez besoin du nombre total de documents :

```javascript
// Compter le total de documents
const totalDocuments = db.products.countDocuments({ category: "Electronics" });

// Calculer le nombre de pages
const pageSize = 10;
const totalPages = Math.ceil(totalDocuments / pageSize);

// Récupérer une page spécifique
const currentPage = 2;
const documents = db.products
    .find({ category: "Electronics" })
    .skip((currentPage - 1) * pageSize)
    .limit(pageSize);

// Informations de pagination
console.log(`Page ${currentPage} sur ${totalPages}`);
console.log(`Total de documents : ${totalDocuments}`);
```

---

## Combinaison de `sort()`, `limit()` et `skip()`

Les trois méthodes peuvent être chaînées dans n'importe quel ordre dans votre code, mais MongoDB les **exécute toujours dans cet ordre** :

1. **`sort()`** - trie d'abord
2. **`skip()`** - saute ensuite
3. **`limit()`** - limite enfin

### Ordre d'Écriture vs Ordre d'Exécution

```javascript
// Ces trois requêtes sont équivalentes et donnent le même résultat :

db.products.find().sort({ price: -1 }).skip(10).limit(5)
db.products.find().skip(10).sort({ price: -1 }).limit(5)
db.products.find().limit(5).skip(10).sort({ price: -1 })

// MongoDB exécute toujours dans l'ordre : sort → skip → limit
```

### Pourquoi cet Ordre ?

```javascript
// Imaginons 100 produits

// 1. sort({ price: -1 }) : trie les 100 produits par prix décroissant
// 2. skip(10) : saute les 10 premiers (les 10 plus chers)
// 3. limit(5) : prend les 5 suivants

// Résultat : les produits classés 11 à 15 en termes de prix
```

### Exemples Pratiques

```javascript
// Top 10 des produits les plus chers (classés 1 à 10)
db.products.find().sort({ price: -1 }).limit(10)

// Produits classés 11 à 20 par prix
db.products.find().sort({ price: -1 }).skip(10).limit(10)

// Page 3 des articles les plus récents (21 à 30)
db.articles.find().sort({ publishedDate: -1 }).skip(20).limit(10)

// Les 5 commandes les plus anciennes, en sautant les 10 premières
db.orders
    .find({ status: "pending" })
    .sort({ orderDate: 1 })
    .skip(10)
    .limit(5)
```

---

## Cas d'Usage Pratiques

### Cas 1 : Tableau de Classement (Leaderboard)

```javascript
// Top 100 des meilleurs joueurs
db.players
    .find({ active: true })
    .sort({ score: -1, completionTime: 1 })
    .limit(100)

// Afficher la page 2 du classement (positions 11-20)
db.players
    .find({ active: true })
    .sort({ score: -1 })
    .skip(10)
    .limit(10)
```

### Cas 2 : Liste de Produits avec Filtres

```javascript
// Produits électroniques, triés par popularité, page 2
db.products
    .find({
        category: "Electronics",
        inStock: true
    })
    .sort({ salesCount: -1, rating: -1 })
    .skip(20)
    .limit(20)
```

### Cas 3 : Fil d'Actualité (Feed)

```javascript
// Les 50 derniers posts
db.posts
    .find({ status: "published" })
    .sort({ publishedDate: -1 })
    .limit(50)

// Charger plus : 50 posts suivants
db.posts
    .find({ status: "published" })
    .sort({ publishedDate: -1 })
    .skip(50)
    .limit(50)
```

### Cas 4 : Recommandations

```javascript
// Top 5 des produits recommandés (mais pas le premier)
db.products
    .find({
        category: "Books",
        rating: { $gte: 4.5 }
    })
    .sort({ rating: -1, reviewCount: -1 })
    .skip(1)
    .limit(5)
```

### Cas 5 : Historique des Commandes

```javascript
// 20 dernières commandes d'un client, page 1
db.orders
    .find({ customerId: ObjectId("...") })
    .sort({ orderDate: -1 })
    .limit(20)

// Page 2 (commandes 21-40)
db.orders
    .find({ customerId: ObjectId("...") })
    .sort({ orderDate: -1 })
    .skip(20)
    .limit(20)
```

### Cas 6 : Recherche avec Tri Pertinence

```javascript
// Recherche textuelle avec tri par score de pertinence
db.articles
    .find(
        { $text: { $search: "mongodb tutorial" } },
        { score: { $meta: "textScore" } }
    )
    .sort({ score: { $meta: "textScore" } })
    .limit(10)
```

---

## Pagination dans les Applications

### Structure Complète de Pagination

```javascript
async function getPage(query, page, pageSize, sortField, sortOrder) {
    // Valider les paramètres
    page = Math.max(1, parseInt(page) || 1);
    pageSize = Math.min(100, Math.max(1, parseInt(pageSize) || 10));

    // Calculer skip
    const skip = (page - 1) * pageSize;

    // Compter le total
    const totalDocuments = await db.products.countDocuments(query);
    const totalPages = Math.ceil(totalDocuments / pageSize);

    // Récupérer les documents
    const documents = await db.products
        .find(query)
        .sort({ [sortField]: sortOrder })
        .skip(skip)
        .limit(pageSize)
        .toArray();

    // Retourner les résultats avec métadonnées
    return {
        data: documents,
        pagination: {
            currentPage: page,
            pageSize: pageSize,
            totalDocuments: totalDocuments,
            totalPages: totalPages,
            hasNextPage: page < totalPages,
            hasPreviousPage: page > 1
        }
    };
}

// Utilisation
const result = await getPage(
    { category: "Electronics", inStock: true },
    2,      // page
    20,     // pageSize
    "price", // sortField
    -1      // sortOrder
);
```

### API REST avec Pagination

```javascript
// Route Express.js
app.get('/api/products', async (req, res) => {
    const {
        page = 1,
        limit = 20,
        sort = 'createdAt',
        order = 'desc',
        category,
        minPrice,
        maxPrice
    } = req.query;

    // Construire la requête
    const query = {};
    if (category) query.category = category;
    if (minPrice) query.price = { ...query.price, $gte: parseFloat(minPrice) };
    if (maxPrice) query.price = { ...query.price, $lte: parseFloat(maxPrice) };

    // Ordre de tri
    const sortOrder = order === 'desc' ? -1 : 1;

    // Pagination
    const skip = (parseInt(page) - 1) * parseInt(limit);

    // Récupérer les données
    const [products, total] = await Promise.all([
        db.collection('products')
            .find(query)
            .sort({ [sort]: sortOrder })
            .skip(skip)
            .limit(parseInt(limit))
            .toArray(),
        db.collection('products').countDocuments(query)
    ]);

    res.json({
        products,
        page: parseInt(page),
        limit: parseInt(limit),
        total,
        pages: Math.ceil(total / parseInt(limit))
    });
});
```

---

## Comparaison avec SQL

| SQL | MongoDB |
|-----|---------|
| `ORDER BY name ASC` | `.sort({ name: 1 })` |
| `ORDER BY price DESC` | `.sort({ price: -1 })` |
| `ORDER BY category ASC, price DESC` | `.sort({ category: 1, price: -1 })` |
| `LIMIT 10` | `.limit(10)` |
| `OFFSET 20` | `.skip(20)` |
| `LIMIT 10 OFFSET 20` | `.skip(20).limit(10)` |
| `ORDER BY price DESC LIMIT 5` | `.sort({ price: -1 }).limit(5)` |

### Exemple Complet

```sql
-- SQL
SELECT * FROM products
WHERE category = 'Electronics'
ORDER BY price DESC, name ASC
LIMIT 10 OFFSET 20;
```

```javascript
// MongoDB
db.products
    .find({ category: "Electronics" })
    .sort({ price: -1, name: 1 })
    .skip(20)
    .limit(10)
```

---

## Bonnes Pratiques

### 1. Toujours Créer des Index pour le Tri

```javascript
// ✅ Créer un index pour les champs de tri fréquents
db.products.createIndex({ price: 1 })
db.articles.createIndex({ publishedDate: -1 })
db.users.createIndex({ name: 1 })

// Les requêtes utilisent l'index efficacement
db.products.find().sort({ price: 1 })
```

### 2. Index Composés pour Tri Multi-champs

```javascript
// ✅ Index composé pour tri sur plusieurs champs
db.products.createIndex({ category: 1, price: -1 })

// Requête optimisée
db.products
    .find({ category: "Electronics" })
    .sort({ category: 1, price: -1 })
```

### 3. Limiter les Résultats

```javascript
// ✅ Toujours utiliser limit() pour éviter les grandes réponses
db.products.find().sort({ price: -1 }).limit(100)

// ❌ Éviter : récupérer tous les documents sans limite
db.products.find().sort({ price: -1 })
```

### 4. Pagination Efficace pour Petits Résultats

```javascript
// ✅ Pour des datasets modérés (< 10,000 documents)
db.products
    .find(query)
    .sort({ price: 1 })
    .skip(skip)
    .limit(limit)
```

### 5. Alternative à `skip()` pour Grandes Données

Pour de très grandes collections, `skip()` devient inefficace. Utilisez la **pagination basée sur curseur** :

```javascript
// ❌ Lent pour grandes pages (skip est coûteux)
db.products.find().sort({ _id: 1 }).skip(100000).limit(20)

// ✅ Meilleur : pagination basée sur le dernier ID
const lastId = ObjectId("...");  // Dernier _id de la page précédente
db.products
    .find({ _id: { $gt: lastId } })
    .sort({ _id: 1 })
    .limit(20)
```

### 6. Combiner Filtres et Tri avec Index

```javascript
// ✅ Index composé optimisé
db.orders.createIndex({ status: 1, orderDate: -1 })

// Requête efficace
db.orders
    .find({ status: "pending" })
    .sort({ orderDate: -1 })
    .limit(20)
```

### 7. Limiter la Taille des Pages

```javascript
// ✅ Limiter la taille maximale des pages
const MAX_PAGE_SIZE = 100;
const pageSize = Math.min(requestedPageSize, MAX_PAGE_SIZE);

db.products.find().limit(pageSize)
```

### 8. Mettre en Cache les Comptages

```javascript
// ⚠️ countDocuments() peut être lent sur grandes collections
const total = await db.products.countDocuments(query);

// ✅ Mettre en cache ou utiliser estimatedDocumentCount()
// Pour le total (sans filtre)
const total = await db.products.estimatedDocumentCount();
```

---

## Pièges Courants à Éviter

### 1. `skip()` sur de Grandes Valeurs

```javascript
// ❌ Très inefficace : MongoDB doit parcourir 1,000,000 documents
db.products.find().skip(1000000).limit(10)

// ✅ Solution : pagination basée sur curseur
db.products
    .find({ _id: { $gt: lastSeenId } })
    .sort({ _id: 1 })
    .limit(10)
```

### 2. Tri Sans Index

```javascript
// ❌ Lent : tri en mémoire sans index
db.products.find().sort({ price: 1 })

// ✅ Créer un index d'abord
db.products.createIndex({ price: 1 })
db.products.find().sort({ price: 1 })
```

### 3. Oublier `limit()` avec `skip()`

```javascript
// ❌ Retourne tous les documents après le skip
db.products.find().skip(100)

// ✅ Toujours utiliser limit() avec skip()
db.products.find().skip(100).limit(10)
```

### 4. Ordre de Tri Incohérent

```javascript
// ❌ Résultats non déterministes
db.products.find().skip(10).limit(10)
// Si aucun tri, l'ordre n'est pas garanti

// ✅ Toujours spécifier un tri explicite
db.products.find().sort({ _id: 1 }).skip(10).limit(10)
```

### 5. Comptage Inefficace

```javascript
// ❌ Lent : compte tous les documents à chaque requête
const total = await db.products.countDocuments(query);

// ✅ Mieux : mettre en cache ou calculer périodiquement
// Ou utiliser estimatedDocumentCount() si pas de filtre
```

### 6. Tri sur Champs Non Indexés

```javascript
// ❌ Tri en mémoire (limité à 32 Mo)
db.products.find().sort({ customField: 1 })

// ✅ Créer un index
db.products.createIndex({ customField: 1 })
```

### 7. Tri sur Tableaux Sans Comprendre le Comportement

```javascript
// ⚠️ Attention : tri utilise min ou max du tableau
db.products.find().sort({ ratings: -1 })
// Utilise la note la plus élevée de chaque produit

// Peut ne pas correspondre à l'intention
// Envisager de stocker la note moyenne séparément
```

---

## Performance et Optimisation

### Impact sur les Performances

| Opération | Sans Index | Avec Index | Notes |
|-----------|------------|------------|-------|
| `sort()` | Très lent (tri en mémoire) | Rapide | Index crucial pour sort |
| `limit()` | Rapide | Rapide | Peu d'impact |
| `skip()` | Lent (parcourt documents) | Moyen | Éviter grandes valeurs |
| `skip() + limit()` | Lent | Moyen | Utiliser curseur pour grandes pages |

### Optimisation du Tri

```javascript
// ✅ Index dans le bon ordre
db.products.createIndex({ category: 1, price: -1 })

// Requête optimisée
db.products
    .find({ category: "Electronics" })
    .sort({ category: 1, price: -1 })
```

### Limites de Tri en Mémoire

MongoDB limite le tri en mémoire à **32 Mo**. Si vous triez sans index et que les données dépassent cette limite, vous obtiendrez une erreur.

```javascript
// ❌ Erreur si trop de données
db.products.find().sort({ customField: 1 })
// Erreur : "Executor error during find command: OperationFailed: Sort operation used more than the maximum 33554432 bytes of RAM."

// ✅ Solution : créer un index
db.products.createIndex({ customField: 1 })
```

### Vérification avec `explain()`

```javascript
// Analyser une requête avec tri
db.products
    .find({ category: "Electronics" })
    .sort({ price: -1 })
    .limit(20)
    .explain("executionStats")

// Vérifier :
// - "IXSCAN" : utilise un index
// - "SORT" : tri en mémoire (pas optimal)
// - "executionTimeMillis" : temps d'exécution
```

### Pagination Basée sur Curseur (Alternative)

Pour de très grandes collections, utilisez la pagination basée sur curseur :

```javascript
// Approche traditionnelle (lente pour grandes pages)
db.products.find().sort({ _id: 1 }).skip(10000).limit(20)

// Approche par curseur (plus rapide)
// Page 1
const page1 = await db.products
    .find()
    .sort({ _id: 1 })
    .limit(20)
    .toArray();

const lastId = page1[page1.length - 1]._id;

// Page suivante
const page2 = await db.products
    .find({ _id: { $gt: lastId } })
    .sort({ _id: 1 })
    .limit(20)
    .toArray();
```

### Exemple Complet avec Performance Optimale

```javascript
// Configuration optimale
db.products.createIndex({ category: 1, price: -1, _id: 1 });

// Fonction de pagination optimisée
async function paginateProducts(category, page, pageSize) {
    // Valider
    page = Math.max(1, page);
    pageSize = Math.min(100, Math.max(1, pageSize));

    const query = { category };

    // Pour les premières pages : skip/limit classique
    if (page <= 10) {
        const skip = (page - 1) * pageSize;
        const [products, total] = await Promise.all([
            db.products
                .find(query)
                .sort({ price: -1 })
                .skip(skip)
                .limit(pageSize)
                .toArray(),
            db.products.countDocuments(query)
        ]);

        return { products, total, page, pageSize };
    }

    // Pour les pages lointaines : approche par curseur
    // (nécessite de stocker lastId dans l'état côté client)
    // ...
}
```

---

## Points Clés à Retenir

✅ **`sort()`** trie les résultats : `1` = croissant, `-1` = décroissant

✅ **`limit()`** restreint le nombre de documents retournés

✅ **`skip()`** saute un nombre spécifié de documents

✅ MongoDB exécute toujours dans l'ordre : **sort → skip → limit**

✅ Vous pouvez chaîner les méthodes dans **n'importe quel ordre** dans votre code

✅ Les **index sont cruciaux** pour les performances de tri

✅ **Pagination** = `skip((page - 1) * pageSize).limit(pageSize)`

✅ `skip()` est **inefficace** pour de grandes valeurs (> 10,000)

✅ Pour grandes données, utilisez la **pagination basée sur curseur**

✅ Le tri sans index est **limité à 32 Mo** en mémoire

---

## Résumé

Dans ce chapitre, vous avez appris :

- Comment utiliser `sort()` pour trier les résultats (croissant et décroissant)
- Comment trier sur plusieurs champs avec ordre de priorité
- Comment utiliser `limit()` pour restreindre le nombre de résultats
- Comment utiliser `skip()` pour sauter des documents
- Comment implémenter la pagination avec skip et limit
- L'ordre d'exécution de MongoDB (sort → skip → limit)
- Les cas d'usage pratiques (leaderboards, feeds, recherche, etc.)
- Les bonnes pratiques d'indexation et d'optimisation
- Les pièges courants et comment les éviter
- Les alternatives performantes pour les grandes collections

Le tri, la limitation et la pagination sont des opérations fondamentales pour toute application utilisant MongoDB. Une bonne maîtrise de ces techniques, combinée avec des index appropriés, vous permettra de créer des applications rapides et efficaces même avec de grandes quantités de données.

Dans le prochain chapitre, nous explorerons le **comptage de documents** avec les différentes méthodes disponibles et leurs cas d'usage spécifiques.

---


⏭️ [Comptage de documents (countDocuments, estimatedDocumentCount)](/03-requetes-et-filtres/09-comptage-documents.md)
