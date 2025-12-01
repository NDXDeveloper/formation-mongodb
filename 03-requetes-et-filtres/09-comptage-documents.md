🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.9 Comptage de Documents

## Introduction

Compter le nombre de documents dans une collection ou correspondant à certains critères est une opération très courante dans les applications MongoDB. Vous avez besoin de comptages pour :

- **Pagination** : savoir combien de pages afficher
- **Statistiques** : nombre d'utilisateurs actifs, de produits en stock, etc.
- **Tableaux de bord** : métriques et KPI
- **Validation** : vérifier qu'une opération a bien créé/supprimé des documents
- **Rapports** : générer des analyses de données

MongoDB offre plusieurs méthodes pour compter les documents, chacune avec ses avantages et ses cas d'usage spécifiques :

- **`countDocuments()`** : comptage précis avec possibilité de filtres
- **`estimatedDocumentCount()`** : comptage rapide estimé (sans filtres)
- **`count()`** : méthode dépréciée (à éviter)

Dans ce chapitre, nous allons explorer ces méthodes en détail et apprendre quand utiliser chacune d'elles.

---

## La Méthode `countDocuments()`

La méthode `countDocuments()` retourne le **nombre exact** de documents correspondant à une requête. C'est la méthode recommandée pour compter des documents avec des filtres.

### Syntaxe

```javascript
db.collection.countDocuments(query, options)
```

- **`query`** : critères de filtrage (peut être vide `{}` pour tout compter)
- **`options`** : options additionnelles (optionnel)

### Compter Tous les Documents

```javascript
// Compter tous les documents d'une collection
db.users.countDocuments({})

// Résultat : 1523
```

### Compter avec Filtres

```javascript
// Compter les utilisateurs actifs
db.users.countDocuments({ status: "active" })

// Compter les produits en stock
db.products.countDocuments({ stock: { $gt: 0 } })

// Compter les commandes complétées
db.orders.countDocuments({ status: "completed" })

// Compter les articles publiés en 2024
db.articles.countDocuments({
    status: "published",
    publishedDate: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2025-01-01T00:00:00Z")
    }
})
```

### Exemples avec Différents Opérateurs

```javascript
// Opérateurs de comparaison
db.products.countDocuments({ price: { $gte: 100 } })
db.users.countDocuments({ age: { $lt: 18 } })

// Opérateurs logiques
db.products.countDocuments({
    $or: [
        { category: "Electronics" },
        { category: "Books" }
    ]
})

// Opérateurs de tableaux
db.users.countDocuments({
    tags: { $all: ["developer", "javascript"] }
})

// Expressions régulières
db.users.countDocuments({
    email: { $regex: /@gmail\.com$/i }
})

// Opérateurs d'éléments
db.products.countDocuments({
    description: { $exists: true }
})
```

### Options de `countDocuments()`

```javascript
// Avec limite
db.users.countDocuments(
    { status: "active" },
    { limit: 1000 }
)
// S'arrête à 1000 même s'il y a plus de documents

// Avec skip
db.users.countDocuments(
    { status: "active" },
    { skip: 100 }
)
// Commence à compter après les 100 premiers

// Avec hint (forcer un index)
db.users.countDocuments(
    { status: "active" },
    { hint: { status: 1 } }
)
```

### Cas d'Usage Pratiques

#### Pagination

```javascript
// Informations pour la pagination
const query = { category: "Electronics", inStock: true };
const pageSize = 20;

const totalDocuments = await db.products.countDocuments(query);
const totalPages = Math.ceil(totalDocuments / pageSize);

console.log(`Total de produits : ${totalDocuments}`);
console.log(`Nombre de pages : ${totalPages}`);
```

#### Statistiques

```javascript
// Statistiques utilisateurs
const stats = {
    total: await db.users.countDocuments({}),
    active: await db.users.countDocuments({ status: "active" }),
    inactive: await db.users.countDocuments({ status: "inactive" }),
    banned: await db.users.countDocuments({ status: "banned" }),
    verified: await db.users.countDocuments({ verified: true })
};

console.log("Statistiques utilisateurs :", stats);
```

#### Validation d'Opérations

```javascript
// Avant suppression
const beforeCount = await db.products.countDocuments({ category: "Obsolete" });
console.log(`${beforeCount} produits à supprimer`);

// Suppression
await db.products.deleteMany({ category: "Obsolete" });

// Vérification après suppression
const afterCount = await db.products.countDocuments({ category: "Obsolete" });
console.log(`${afterCount} produits restants (devrait être 0)`);
```

#### Tableaux de Bord

```javascript
// Métriques pour un tableau de bord
const dashboard = {
    ordersToday: await db.orders.countDocuments({
        orderDate: { $gte: new Date().setHours(0, 0, 0, 0) }
    }),
    pendingOrders: await db.orders.countDocuments({
        status: "pending"
    }),
    lowStockProducts: await db.products.countDocuments({
        stock: { $lt: 10 }
    }),
    newUsersThisMonth: await db.users.countDocuments({
        registrationDate: {
            $gte: new Date(new Date().getFullYear(), new Date().getMonth(), 1)
        }
    })
};
```

---

## La Méthode `estimatedDocumentCount()`

La méthode `estimatedDocumentCount()` retourne une **estimation rapide** du nombre total de documents dans une collection. Elle utilise les métadonnées de la collection et est **beaucoup plus rapide** que `countDocuments()`.

### Syntaxe

```javascript
db.collection.estimatedDocumentCount(options)
```

**Important** : Cette méthode **ne prend pas de requête** en paramètre. Elle compte toujours tous les documents de la collection.

### Exemple de Base

```javascript
// Estimation rapide du nombre de documents
const count = await db.users.estimatedDocumentCount();
console.log(`Estimation : ${count} utilisateurs`);
```

### Différence Clé : Pas de Filtres

```javascript
// ✅ Correct : pas de paramètres ou options vides
const total = await db.products.estimatedDocumentCount();

// ❌ Incorrect : ne peut pas filtrer
const active = await db.products.estimatedDocumentCount({ status: "active" });
// Ceci ignore le filtre et compte TOUS les documents !
```

### Cas d'Usage de `estimatedDocumentCount()`

#### Affichage Rapide du Total

```javascript
// Afficher le nombre approximatif de documents
const approxTotal = await db.products.estimatedDocumentCount();
console.log(`Environ ${approxTotal} produits dans la base`);
```

#### Vérifications Rapides

```javascript
// Vérifier rapidement si la collection est vide
const count = await db.newCollection.estimatedDocumentCount();
if (count === 0) {
    console.log("Collection vide, initialisation nécessaire");
}
```

#### Métriques Générales

```javascript
// Statistiques globales rapides (sans besoin de précision)
const stats = {
    totalUsers: await db.users.estimatedDocumentCount(),
    totalProducts: await db.products.estimatedDocumentCount(),
    totalOrders: await db.orders.estimatedDocumentCount()
};

console.log("Statistiques globales (estimées) :", stats);
```

---

## Comparaison : `countDocuments()` vs `estimatedDocumentCount()`

### Tableau Comparatif

| Caractéristique | `countDocuments()` | `estimatedDocumentCount()` |
|-----------------|--------------------|-----------------------------|
| **Précision** | Exacte | Estimée (peut différer légèrement) |
| **Vitesse** | Plus lent | Très rapide |
| **Filtres** | ✅ Supporte les requêtes | ❌ Pas de filtres possibles |
| **Utilise** | Scan ou index | Métadonnées de collection |
| **Cas d'usage** | Comptages précis avec filtres | Total rapide sans filtres |
| **Impact performance** | Modéré à élevé | Très faible |

### Exemples Comparatifs

```javascript
// Scénario 1 : Compter tous les documents
// countDocuments : précis mais plus lent
const exact = await db.products.countDocuments({});

// estimatedDocumentCount : approximatif mais très rapide
const estimated = await db.products.estimatedDocumentCount();

// Différence généralement négligeable pour le total
console.log(`Exact: ${exact}, Estimé: ${estimated}`);


// Scénario 2 : Avec filtres
// countDocuments : seule option pour filtrer
const activeUsers = await db.users.countDocuments({ status: "active" });

// estimatedDocumentCount : NE PEUT PAS filtrer
// Cette approche est incorrecte pour compter les utilisateurs actifs
const wrong = await db.users.estimatedDocumentCount();
// Compte TOUS les utilisateurs, pas seulement les actifs !
```

### Quand Utiliser Chaque Méthode

#### Utilisez `countDocuments()` quand :

- ✅ Vous avez besoin d'un **comptage exact**
- ✅ Vous devez appliquer des **filtres** ou des **critères**
- ✅ Vous comptez pour la **pagination** (besoin de précision)
- ✅ Vous comptez pour des **validations** ou des **vérifications**
- ✅ Le comptage fait partie d'une **logique métier** critique

```javascript
// Exemples appropriés pour countDocuments()
db.orders.countDocuments({ status: "pending" })
db.products.countDocuments({ category: "Electronics", price: { $lt: 100 } })
db.users.countDocuments({ lastLogin: { $gte: lastWeek } })
```

#### Utilisez `estimatedDocumentCount()` quand :

- ✅ Vous avez besoin du **total général** de la collection
- ✅ Une **estimation** suffit (pas besoin d'exactitude absolue)
- ✅ La **performance** est critique
- ✅ Vous affichez des **statistiques générales** non critiques

```javascript
// Exemples appropriés pour estimatedDocumentCount()
const totalUsers = await db.users.estimatedDocumentCount();
const totalProducts = await db.products.estimatedDocumentCount();

// Affichage : "Environ 15,000 produits disponibles"
```

---

## La Méthode `count()` (Dépréciée)

**Attention** : La méthode `count()` est **dépréciée** depuis MongoDB 4.0 et ne devrait plus être utilisée.

### Pourquoi `count()` est Déprécié

```javascript
// ❌ DÉPRÉCIÉ : Ne plus utiliser
db.users.count({ status: "active" })

// ✅ UTILISER À LA PLACE
db.users.countDocuments({ status: "active" })
```

### Problèmes avec `count()`

1. **Comportement incohérent** dans les clusters shardés
2. **Comptages incorrects** après certaines opérations
3. **Résultats non fiables** avec des filtres complexes

### Migration de `count()` vers `countDocuments()`

```javascript
// Ancien code (déprécié)
const total = db.users.count();
const active = db.users.count({ status: "active" });

// Nouveau code (recommandé)
const total = await db.users.countDocuments({});
const active = await db.users.countDocuments({ status: "active" });

// Ou pour le total sans filtre (plus rapide)
const total = await db.users.estimatedDocumentCount();
```

---

## Comptage avec le Framework d'Agrégation

Pour des comptages plus complexes, utilisez le framework d'agrégation avec `$count`.

### Syntaxe avec `$count`

```javascript
db.collection.aggregate([
    { $match: { critères } },
    { $count: "nomDuComptage" }
])
```

### Exemples

```javascript
// Compter les produits actifs
db.products.aggregate([
    { $match: { status: "active" } },
    { $count: "totalActive" }
])
// Résultat : [{ totalActive: 523 }]

// Compter après plusieurs étapes
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $unwind: "$items" },
    { $count: "totalItems" }
])
// Compte le nombre total d'articles dans les commandes complétées
```

### Comptages Multiples avec `$facet`

```javascript
// Plusieurs comptages en une seule requête
db.users.aggregate([
    {
        $facet: {
            "total": [
                { $count: "count" }
            ],
            "active": [
                { $match: { status: "active" } },
                { $count: "count" }
            ],
            "inactive": [
                { $match: { status: "inactive" } },
                { $count: "count" }
            ],
            "verified": [
                { $match: { verified: true } },
                { $count: "count" }
            ]
        }
    }
])

// Résultat :
// {
//     total: [{ count: 1523 }],
//     active: [{ count: 1204 }],
//     inactive: [{ count: 289 }],
//     verified: [{ count: 1450 }]
// }
```

### Comptage par Groupe avec `$group`

```javascript
// Compter le nombre de produits par catégorie
db.products.aggregate([
    {
        $group: {
            _id: "$category",
            count: { $sum: 1 }
        }
    },
    { $sort: { count: -1 } }
])

// Résultat :
// [
//     { _id: "Electronics", count: 523 },
//     { _id: "Books", count: 412 },
//     { _id: "Clothing", count: 356 }
// ]


// Compter les commandes par statut
db.orders.aggregate([
    {
        $group: {
            _id: "$status",
            count: { $sum: 1 }
        }
    }
])

// Résultat :
// [
//     { _id: "completed", count: 5234 },
//     { _id: "pending", count: 156 },
//     { _id: "cancelled", count: 89 }
// ]
```

### Comptages Conditionnels

```javascript
// Compter selon plusieurs conditions
db.products.aggregate([
    {
        $group: {
            _id: null,
            totalProducts: { $sum: 1 },
            inStock: {
                $sum: { $cond: [{ $gt: ["$stock", 0] }, 1, 0] }
            },
            outOfStock: {
                $sum: { $cond: [{ $eq: ["$stock", 0] }, 1, 0] }
            },
            onSale: {
                $sum: { $cond: ["$onSale", 1, 0] }
            }
        }
    }
])

// Résultat :
// [{
//     _id: null,
//     totalProducts: 1523,
//     inStock: 1340,
//     outOfStock: 183,
//     onSale: 256
// }]
```

---

## Cas d'Usage Avancés

### Cas 1 : Pagination Complète

```javascript
async function getPaginatedProducts(query, page, pageSize) {
    // Valider les paramètres
    page = Math.max(1, parseInt(page));
    pageSize = Math.min(100, Math.max(1, parseInt(pageSize)));

    const skip = (page - 1) * pageSize;

    // Exécuter en parallèle pour optimiser
    const [products, totalCount] = await Promise.all([
        db.products
            .find(query)
            .skip(skip)
            .limit(pageSize)
            .toArray(),
        db.products.countDocuments(query)
    ]);

    const totalPages = Math.ceil(totalCount / pageSize);

    return {
        data: products,
        pagination: {
            currentPage: page,
            pageSize: pageSize,
            totalDocuments: totalCount,
            totalPages: totalPages,
            hasNextPage: page < totalPages,
            hasPreviousPage: page > 1
        }
    };
}
```

### Cas 2 : Statistiques de Tableau de Bord

```javascript
async function getDashboardStats() {
    // Utiliser Promise.all pour paralléliser
    const [
        totalUsers,
        activeUsers,
        totalProducts,
        lowStockProducts,
        pendingOrders,
        todayOrders
    ] = await Promise.all([
        // Estimation rapide pour le total
        db.users.estimatedDocumentCount(),

        // Comptages précis pour les filtres
        db.users.countDocuments({ status: "active" }),
        db.products.estimatedDocumentCount(),
        db.products.countDocuments({ stock: { $lt: 10 } }),
        db.orders.countDocuments({ status: "pending" }),
        db.orders.countDocuments({
            orderDate: { $gte: new Date().setHours(0, 0, 0, 0) }
        })
    ]);

    return {
        users: {
            total: totalUsers,
            active: activeUsers,
            activePercentage: ((activeUsers / totalUsers) * 100).toFixed(2)
        },
        products: {
            total: totalProducts,
            lowStock: lowStockProducts
        },
        orders: {
            pending: pendingOrders,
            today: todayOrders
        }
    };
}
```

### Cas 3 : Vérification de Doublons

```javascript
// Vérifier s'il existe déjà un utilisateur avec cet email
const existingCount = await db.users.countDocuments({
    email: "new.user@example.com"
});

if (existingCount > 0) {
    throw new Error("Cet email est déjà utilisé");
}

// Continuer l'inscription...
```

### Cas 4 : Validation de Suppression en Masse

```javascript
async function safeBulkDelete(query) {
    // Compter combien de documents seront supprimés
    const count = await db.products.countDocuments(query);

    console.log(`${count} documents seront supprimés`);

    // Demander confirmation si plus de 100
    if (count > 100) {
        const confirmed = await askUserConfirmation(
            `Êtes-vous sûr de vouloir supprimer ${count} documents ?`
        );

        if (!confirmed) {
            console.log("Suppression annulée");
            return;
        }
    }

    // Procéder à la suppression
    const result = await db.products.deleteMany(query);
    console.log(`${result.deletedCount} documents supprimés`);
}
```

### Cas 5 : Rapports Périodiques

```javascript
async function generateMonthlyReport(year, month) {
    const startDate = new Date(year, month - 1, 1);
    const endDate = new Date(year, month, 1);

    const dateQuery = {
        createdAt: { $gte: startDate, $lt: endDate }
    };

    const report = {
        period: `${year}-${month.toString().padStart(2, '0')}`,
        newUsers: await db.users.countDocuments(dateQuery),
        newOrders: await db.orders.countDocuments(dateQuery),
        completedOrders: await db.orders.countDocuments({
            ...dateQuery,
            status: "completed"
        }),
        revenue: await calculateRevenue(startDate, endDate)
    };

    return report;
}
```

---

## Comparaison avec SQL

| SQL | MongoDB |
|-----|---------|
| `SELECT COUNT(*) FROM users` | `db.users.countDocuments({})` ou `db.users.estimatedDocumentCount()` |
| `SELECT COUNT(*) FROM users WHERE status = 'active'` | `db.users.countDocuments({ status: "active" })` |
| `SELECT COUNT(DISTINCT category) FROM products` | Nécessite agrégation (voir ci-dessous) |
| `SELECT category, COUNT(*) FROM products GROUP BY category` | `db.products.aggregate([{ $group: { _id: "$category", count: { $sum: 1 } } }])` |

### Comptage Distinct

En SQL, vous utiliseriez `COUNT(DISTINCT field)`. En MongoDB :

```javascript
// Compter les catégories distinctes
db.products.distinct("category").length

// Ou avec agrégation
db.products.aggregate([
    { $group: { _id: "$category" } },
    { $count: "uniqueCategories" }
])
```

---

## Bonnes Pratiques

### 1. Choisir la Bonne Méthode

```javascript
// ✅ Pour le total sans filtre : estimatedDocumentCount (rapide)
const total = await db.users.estimatedDocumentCount();

// ✅ Pour compter avec filtres : countDocuments (précis)
const active = await db.users.countDocuments({ status: "active" });

// ❌ Éviter : count() (déprécié)
const deprecated = await db.users.count({ status: "active" });
```

### 2. Paralléliser les Comptages Multiples

```javascript
// ✅ Bon : exécution en parallèle
const [total, active, inactive] = await Promise.all([
    db.users.countDocuments({}),
    db.users.countDocuments({ status: "active" }),
    db.users.countDocuments({ status: "inactive" })
]);

// ❌ Lent : exécution séquentielle
const total = await db.users.countDocuments({});
const active = await db.users.countDocuments({ status: "active" });
const inactive = await db.users.countDocuments({ status: "inactive" });
```

### 3. Mettre en Cache les Comptages Fréquents

```javascript
// Pour des comptages très fréquents, envisager le cache
let cachedCount = null;
let cacheTime = null;
const CACHE_DURATION = 60000; // 1 minute

async function getCachedUserCount() {
    const now = Date.now();

    if (!cachedCount || !cacheTime || (now - cacheTime) > CACHE_DURATION) {
        cachedCount = await db.users.estimatedDocumentCount();
        cacheTime = now;
    }

    return cachedCount;
}
```

### 4. Utiliser les Index pour Améliorer les Performances

```javascript
// Créer un index pour les comptages fréquents
db.products.createIndex({ status: 1 });
db.orders.createIndex({ orderDate: 1 });

// Les comptages utilisent les index
db.products.countDocuments({ status: "active" }); // Utilise l'index
```

### 5. Limiter les Comptages sur Grandes Collections

```javascript
// Pour de très grandes collections, envisager un comptage approximatif
const MAX_COUNT = 10000;

const count = await db.products.countDocuments(
    { category: "Electronics" },
    { limit: MAX_COUNT }
);

if (count === MAX_COUNT) {
    console.log(`Plus de ${MAX_COUNT} résultats`);
} else {
    console.log(`${count} résultats`);
}
```

### 6. Stocker les Comptages dans des Documents Séparés

Pour des compteurs très sollicités :

```javascript
// Collection de compteurs
{
    _id: "user_count",
    value: 15234,
    lastUpdated: ISODate("2024-12-01T10:30:00Z")
}

// Incrémenter lors de l'ajout
await db.counters.updateOne(
    { _id: "user_count" },
    {
        $inc: { value: 1 },
        $set: { lastUpdated: new Date() }
    }
);

// Lecture rapide
const userCount = await db.counters.findOne({ _id: "user_count" });
console.log(`Utilisateurs : ${userCount.value}`);
```

### 7. Utiliser l'Agrégation pour Comptages Complexes

```javascript
// ✅ Pour des statistiques multiples en une seule requête
db.products.aggregate([
    {
        $facet: {
            "total": [{ $count: "count" }],
            "byCategory": [
                { $group: { _id: "$category", count: { $sum: 1 } } }
            ],
            "inStock": [
                { $match: { stock: { $gt: 0 } } },
                { $count: "count" }
            ]
        }
    }
])
```

---

## Pièges Courants à Éviter

### 1. Utiliser `estimatedDocumentCount()` avec des Filtres

```javascript
// ❌ Incorrect : ignore complètement le filtre
const active = await db.users.estimatedDocumentCount({ status: "active" });
// Compte TOUS les utilisateurs, pas seulement les actifs !

// ✅ Correct : utiliser countDocuments pour filtrer
const active = await db.users.countDocuments({ status: "active" });
```

### 2. Utiliser `count()` (Déprécié)

```javascript
// ❌ Déprécié
const count = await db.users.count({ status: "active" });

// ✅ Recommandé
const count = await db.users.countDocuments({ status: "active" });
```

### 3. Compter Sans Index sur Grandes Collections

```javascript
// ⚠️ Lent sans index
const count = await db.products.countDocuments({
    customField: "value"
});

// ✅ Créer un index d'abord
db.products.createIndex({ customField: 1 });
const count = await db.products.countDocuments({
    customField: "value"
});
```

### 4. Comptages Séquentiels au Lieu de Parallèles

```javascript
// ❌ Lent : 3 requêtes séquentielles
const total = await db.users.countDocuments({});
const active = await db.users.countDocuments({ status: "active" });
const verified = await db.users.countDocuments({ verified: true });

// ✅ Rapide : 3 requêtes en parallèle
const [total, active, verified] = await Promise.all([
    db.users.countDocuments({}),
    db.users.countDocuments({ status: "active" }),
    db.users.countDocuments({ verified: true })
]);
```

### 5. Compter pour Vérifier l'Existence

```javascript
// ❌ Inefficace : compte tous les documents
const count = await db.users.countDocuments({ email: "test@example.com" });
if (count > 0) {
    // L'email existe
}

// ✅ Plus efficace : limiter à 1
const count = await db.users.countDocuments(
    { email: "test@example.com" },
    { limit: 1 }
);
if (count > 0) {
    // L'email existe
}

// ✅ Encore mieux : utiliser findOne
const user = await db.users.findOne(
    { email: "test@example.com" },
    { projection: { _id: 1 } }
);
if (user) {
    // L'email existe
}
```

### 6. Oublier la Gestion d'Erreurs

```javascript
// ❌ Pas de gestion d'erreurs
const count = await db.users.countDocuments({ status: "active" });

// ✅ Avec gestion d'erreurs
try {
    const count = await db.users.countDocuments({ status: "active" });
    console.log(`${count} utilisateurs actifs`);
} catch (error) {
    console.error("Erreur lors du comptage :", error);
    // Gérer l'erreur appropriatement
}
```

---

## Performance et Optimisation

### Impact sur les Performances

| Méthode | Performance | Cas d'Usage |
|---------|-------------|-------------|
| `estimatedDocumentCount()` | ⚡ Très rapide | Total sans filtres |
| `countDocuments()` avec index | 🟢 Rapide | Avec filtres et index |
| `countDocuments()` sans index | 🔴 Lent | Scan complet (éviter) |
| `count()` (déprécié) | 🟡 Variable | Ne plus utiliser |

### Optimisation avec Index

```javascript
// Créer des index pour les comptages fréquents
db.users.createIndex({ status: 1 });
db.products.createIndex({ category: 1, inStock: 1 });
db.orders.createIndex({ orderDate: 1, status: 1 });

// Les comptages utilisent les index efficacement
db.users.countDocuments({ status: "active" });
db.products.countDocuments({ category: "Electronics", inStock: true });
db.orders.countDocuments({
    orderDate: { $gte: startDate },
    status: "completed"
});
```

### Vérification avec `explain()`

```javascript
// Analyser comment le comptage est exécuté
db.users.explain("executionStats").countDocuments({ status: "active" });

// Vérifier :
// - "IXSCAN" : utilise un index (bon)
// - "COLLSCAN" : scan complet (à optimiser)
// - "executionTimeMillis" : temps d'exécution
```

### Comparaison de Performance

```javascript
// Test de performance
console.time("estimatedDocumentCount");
await db.users.estimatedDocumentCount();
console.timeEnd("estimatedDocumentCount");
// estimatedDocumentCount: 2ms

console.time("countDocuments all");
await db.users.countDocuments({});
console.timeEnd("countDocuments all");
// countDocuments all: 150ms

console.time("countDocuments filtered");
await db.users.countDocuments({ status: "active" });
console.timeEnd("countDocuments filtered");
// countDocuments filtered: 45ms (avec index)
```

### Stratégies d'Optimisation

```javascript
// 1. Utiliser estimatedDocumentCount pour le total
const total = await db.users.estimatedDocumentCount(); // Rapide

// 2. Créer des index pour les filtres
db.users.createIndex({ status: 1 });

// 3. Mettre en cache les résultats fréquents
const cacheKey = "active_users_count";
let count = cache.get(cacheKey);
if (!count) {
    count = await db.users.countDocuments({ status: "active" });
    cache.set(cacheKey, count, 60); // Cache 60 secondes
}

// 4. Paralléliser les comptages multiples
const [count1, count2, count3] = await Promise.all([...]);

// 5. Limiter pour les vérifications d'existence
const exists = await db.users.countDocuments(
    { email: "test@example.com" },
    { limit: 1 }
) > 0;
```

---

## Points Clés à Retenir

✅ **`countDocuments()`** : comptage **exact** avec support des **filtres**

✅ **`estimatedDocumentCount()`** : estimation **rapide** du total (sans filtres)

✅ `estimatedDocumentCount()` utilise les **métadonnées** (très rapide)

✅ `countDocuments()` peut utiliser des **index** pour optimiser

✅ **`count()`** est **déprécié** depuis MongoDB 4.0 (ne plus utiliser)

✅ Pour le **total** sans filtre : préférer `estimatedDocumentCount()`

✅ Pour compter avec **critères** : utiliser `countDocuments()`

✅ **Paralléliser** les comptages multiples avec `Promise.all()`

✅ Créer des **index** sur les champs fréquemment utilisés pour compter

✅ Le framework d'**agrégation** offre des comptages plus complexes

---

## Résumé

Dans ce chapitre, vous avez appris :

- Les deux méthodes principales pour compter : `countDocuments()` et `estimatedDocumentCount()`
- Les différences entre ces méthodes et quand utiliser chacune
- Comment compter avec des filtres et des critères complexes
- Pourquoi `count()` est déprécié et comment migrer
- Comment utiliser l'agrégation pour des comptages avancés
- Les cas d'usage pratiques : pagination, statistiques, validation
- Les bonnes pratiques d'optimisation et de performance
- Les pièges courants à éviter
- Comment utiliser les index pour améliorer les performances

Le comptage de documents est une opération fondamentale dans MongoDB. En choisissant la bonne méthode selon vos besoins (précision vs performance) et en optimisant avec des index, vous pouvez créer des applications rapides et efficaces même avec de grandes quantités de données.

Dans le prochain chapitre, nous explorerons les **requêtes sur documents imbriqués** pour maîtriser l'interrogation de structures de données complexes.

---


⏭️ [Requêtes sur documents imbriqués](/03-requetes-et-filtres/10-requetes-documents-imbriques.md)
