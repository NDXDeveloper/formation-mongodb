🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Framework d'Agrégation

## La puissance des transformations de données ! 🔮

Vous maîtrisez maintenant les requêtes MongoDB, la modélisation et l'optimisation. Excellent ! Mais il est temps de découvrir l'un des outils les plus puissants de MongoDB : **le framework d'agrégation**. C'est avec lui que vous pourrez transformer, analyser, regrouper et calculer vos données de manières quasi infinies.

Le framework d'agrégation est à MongoDB ce que SQL avec ses `GROUP BY`, `JOIN`, et fonctions d'agrégation complexes est aux bases relationnelles. Mais en beaucoup plus puissant et flexible !

## Où en sommes-nous dans votre parcours ?

Vous avez complété les chapitres 1 à 5 et vous maîtrisez maintenant :
- ✅ Les opérations CRUD et requêtes complexes
- ✅ La modélisation des données (imbrication, références, patterns)
- ✅ Les index et l'optimisation des performances
- ✅ L'analyse avec `explain()`

**Parfait !** Vous êtes maintenant prêt à découvrir comment **transformer et analyser** vos données avec une puissance inégalée.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- ✅ **Comprendre** le concept de pipeline d'agrégation
- ✅ **Construire** des pipelines complexes avec plusieurs étapes
- ✅ **Utiliser** toutes les étapes fondamentales ($match, $group, $project, etc.)
- ✅ **Maîtriser** les étapes avancées ($lookup, $unwind, $facet, etc.)
- ✅ **Appliquer** les opérateurs d'agrégation (arithmétiques, dates, chaînes, etc.)
- ✅ **Optimiser** les pipelines pour la performance
- ✅ **Créer** des vues et vues matérialisées
- ✅ **Résoudre** des problèmes analytiques complexes

## De find() à aggregate() : l'évolution de vos compétences

### Ce que vous faites actuellement (find)

```javascript
// Requête simple avec find()
db.orders.find({
    status: "completed",
    total: { $gte: 100 }
})

// Limites de find() :
// - Filtrage basique
// - Pas de calculs complexes
// - Pas de regroupements
// - Pas de jointures (avant $lookup)
// - Projections limitées
```

**`find()` est parfait pour récupérer des documents, mais limité pour l'analyse.**

### Ce que vous ferez bientôt (aggregate)

```javascript
// Pipeline d'agrégation équivalent (et plus puissant)
db.orders.aggregate([
    // Étape 1 : Filtrer
    {
        $match: {
            status: "completed",
            total: { $gte: 100 }
        }
    },

    // Étape 2 : Regrouper par client
    {
        $group: {
            _id: "$customerId",
            totalSpent: { $sum: "$total" },
            orderCount: { $sum: 1 },
            avgOrderValue: { $avg: "$total" },
            maxOrder: { $max: "$total" }
        }
    },

    // Étape 3 : Filtrer les clients VIP (> 1000€)
    {
        $match: {
            totalSpent: { $gte: 1000 }
        }
    },

    // Étape 4 : Trier par montant dépensé
    {
        $sort: { totalSpent: -1 }
    },

    // Étape 5 : Joindre les infos clients
    {
        $lookup: {
            from: "customers",
            localField: "_id",
            foreignField: "_id",
            as: "customerInfo"
        }
    },

    // Étape 6 : Reformater le résultat
    {
        $project: {
            customerId: "$_id",
            customerName: { $arrayElemAt: ["$customerInfo.name", 0] },
            totalSpent: 1,
            orderCount: 1,
            avgOrderValue: { $round: ["$avgOrderValue", 2] },
            vipLevel: {
                $switch: {
                    branches: [
                        { case: { $gte: ["$totalSpent", 5000] }, then: "Platinum" },
                        { case: { $gte: ["$totalSpent", 2000] }, then: "Gold" },
                        { case: { $gte: ["$totalSpent", 1000] }, then: "Silver" }
                    ],
                    default: "Bronze"
                }
            }
        }
    }
])
```

**Résultat :**
```javascript
[
    {
        customerId: ObjectId("..."),
        customerName: "Alice Dupont",
        totalSpent: 5420.50,
        orderCount: 12,
        avgOrderValue: 451.71,
        vipLevel: "Platinum"
    },
    {
        customerId: ObjectId("..."),
        customerName: "Bob Martin",
        totalSpent: 3280.00,
        orderCount: 8,
        avgOrderValue: 410.00,
        vipLevel: "Gold"
    }
    // ...
]
```

**Impressionnant, n'est-ce pas ?** Et ce n'est qu'un aperçu !

## Qu'est-ce qu'un pipeline d'agrégation ?

Un pipeline d'agrégation est comme une **chaîne de montage** pour vos données :

```
Documents    →  Étape 1  →  Étape 2  →  Étape 3  →  Résultat
d'entrée        (filtre)    (groupe)    (tri)       final

1000 docs   →   500 docs →   50 docs →  50 docs →  50 docs
                                          triés      formatés
```

### Concept clé : Le flux de transformation

Chaque étape (stage) :
1. **Reçoit** des documents de l'étape précédente (ou de la collection)
2. **Transforme** ces documents d'une manière spécifique
3. **Passe** le résultat à l'étape suivante

```javascript
db.collection.aggregate([
    stage1,  // Reçoit tous les documents
    stage2,  // Reçoit la sortie de stage1
    stage3,  // Reçoit la sortie de stage2
    stage4   // Reçoit la sortie de stage3
])
```

## Vue d'ensemble du chapitre

Ce chapitre est organisé en 7 sections qui couvrent tous les aspects du framework d'agrégation :

### 🎯 Partie 1 : Fondamentaux (Sections 6.1 et 6.2)
- **6.1** : Introduction au framework d'agrégation
- **6.2** : Concept de pipeline et mécanisme interne

### 🎯 Partie 2 : Étapes de base (Section 6.3)
Les 6 étapes fondamentales que vous utiliserez le plus :
- $match, $project, $group, $sort, $limit/$skip, $count

### 🎯 Partie 3 : Étapes avancées (Section 6.4)
11 étapes puissantes pour cas complexes :
- $lookup (jointures), $unwind, $addFields, $facet, $bucket, $graphLookup, etc.

### 🎯 Partie 4 : Opérateurs (Section 6.5)
Tous les opérateurs disponibles dans les pipelines :
- Arithmétiques, chaînes, dates, tableaux, conditionnels, accumulateurs

### 🎯 Partie 5 : Optimisation (Section 6.6)
Comment écrire des pipelines performants

### 🎯 Partie 6 : Vues (Section 6.7)
Créer des vues et vues matérialisées basées sur des agrégations

## Comparaison SQL : pour mieux comprendre

Si vous connaissez SQL, voici comment le framework d'agrégation se compare :

### Exemple 1 : GROUP BY simple

#### SQL
```sql
SELECT
    category,
    COUNT(*) as productCount,
    AVG(price) as avgPrice,
    SUM(stock) as totalStock
FROM products
WHERE active = true
GROUP BY category
HAVING COUNT(*) >= 10
ORDER BY avgPrice DESC;
```

#### MongoDB Aggregation
```javascript
db.products.aggregate([
    // WHERE
    {
        $match: {
            active: true
        }
    },

    // GROUP BY + fonctions d'agrégation
    {
        $group: {
            _id: "$category",
            productCount: { $sum: 1 },
            avgPrice: { $avg: "$price" },
            totalStock: { $sum: "$stock" }
        }
    },

    // HAVING
    {
        $match: {
            productCount: { $gte: 10 }
        }
    },

    // ORDER BY
    {
        $sort: { avgPrice: -1 }
    }
])
```

### Exemple 2 : JOIN

#### SQL
```sql
SELECT
    o.orderId,
    o.total,
    c.name as customerName,
    c.email
FROM orders o
JOIN customers c ON o.customerId = c.customerId
WHERE o.status = 'completed';
```

#### MongoDB Aggregation
```javascript
db.orders.aggregate([
    // WHERE
    {
        $match: {
            status: "completed"
        }
    },

    // JOIN
    {
        $lookup: {
            from: "customers",
            localField: "customerId",
            foreignField: "customerId",
            as: "customer"
        }
    },

    // SELECT (reshape)
    {
        $project: {
            orderId: "$_id",
            total: 1,
            customerName: { $arrayElemAt: ["$customer.name", 0] },
            email: { $arrayElemAt: ["$customer.email", 0] }
        }
    }
])
```

### Exemple 3 : UNION

#### SQL
```sql
SELECT name, 'customer' as type FROM customers
UNION ALL
SELECT name, 'supplier' as type FROM suppliers;
```

#### MongoDB Aggregation
```javascript
db.customers.aggregate([
    {
        $addFields: {
            type: "customer"
        }
    },
    {
        $unionWith: {
            coll: "suppliers",
            pipeline: [
                {
                    $addFields: {
                        type: "supplier"
                    }
                }
            ]
        }
    }
])
```

## Exemples progressifs : construire un pipeline pas à pas

### Scénario : E-commerce - Analyse des ventes

Collection `orders` :
```javascript
{
    _id: ObjectId("..."),
    orderId: "ORD001",
    customerId: ObjectId("..."),
    orderDate: ISODate("2024-01-15T10:30:00Z"),
    status: "completed",
    items: [
        {
            productId: ObjectId("..."),
            productName: "Ordinateur Dell",
            category: "Informatique",
            quantity: 1,
            price: 1299.99
        },
        {
            productId: ObjectId("..."),
            productName: "Souris Logitech",
            category: "Accessoires",
            quantity: 2,
            price: 29.99
        }
    ],
    total: 1359.97,
    shippingAddress: {
        city: "Paris",
        country: "France"
    }
}
```

### Niveau 1 : Pipeline simple - Filtrage

**Objectif :** Trouver les commandes complétées > 1000€

```javascript
db.orders.aggregate([
    // Étape unique : Filtrer
    {
        $match: {
            status: "completed",
            total: { $gte: 1000 }
        }
    }
])

// Équivalent à :
// db.orders.find({ status: "completed", total: { $gte: 1000 } })
```

**Sortie :** Documents complets des commandes filtrées.

### Niveau 2 : Pipeline à 2 étapes - Filtrage + Projection

**Objectif :** Même chose mais retourner seulement certains champs

```javascript
db.orders.aggregate([
    // Étape 1 : Filtrer
    {
        $match: {
            status: "completed",
            total: { $gte: 1000 }
        }
    },

    // Étape 2 : Projeter (sélectionner champs)
    {
        $project: {
            orderId: 1,
            customerId: 1,
            total: 1,
            orderDate: 1,
            city: "$shippingAddress.city"  // Extraire champ imbriqué
        }
    }
])
```

**Sortie :**
```javascript
[
    {
        _id: ObjectId("..."),
        orderId: "ORD001",
        customerId: ObjectId("..."),
        total: 1359.97,
        orderDate: ISODate("2024-01-15T10:30:00Z"),
        city: "Paris"
    }
    // ...
]
```

### Niveau 3 : Pipeline à 3 étapes - Ajout de calculs

**Objectif :** Ajouter le mois de commande et catégoriser le montant

```javascript
db.orders.aggregate([
    // Étape 1 : Filtrer
    {
        $match: {
            status: "completed",
            total: { $gte: 1000 }
        }
    },

    // Étape 2 : Ajouter des champs calculés
    {
        $addFields: {
            orderMonth: { $month: "$orderDate" },     // Extraire le mois
            orderYear: { $year: "$orderDate" },       // Extraire l'année
            orderCategory: {                          // Catégoriser
                $switch: {
                    branches: [
                        { case: { $gte: ["$total", 5000] }, then: "Premium" },
                        { case: { $gte: ["$total", 2000] }, then: "High" },
                        { case: { $gte: ["$total", 1000] }, then: "Medium" }
                    ],
                    default: "Low"
                }
            }
        }
    },

    // Étape 3 : Projeter résultat final
    {
        $project: {
            orderId: 1,
            total: 1,
            orderYear: 1,
            orderMonth: 1,
            orderCategory: 1,
            city: "$shippingAddress.city"
        }
    }
])
```

**Sortie :**
```javascript
[
    {
        _id: ObjectId("..."),
        orderId: "ORD001",
        total: 1359.97,
        orderYear: 2024,
        orderMonth: 1,
        orderCategory: "Medium",
        city: "Paris"
    }
    // ...
]
```

### Niveau 4 : Regroupement - Statistiques par ville

**Objectif :** Calculer les ventes totales par ville

```javascript
db.orders.aggregate([
    // Étape 1 : Filtrer les commandes complétées
    {
        $match: {
            status: "completed"
        }
    },

    // Étape 2 : Regrouper par ville
    {
        $group: {
            _id: "$shippingAddress.city",              // Grouper par
            totalSales: { $sum: "$total" },            // Somme totale
            orderCount: { $sum: 1 },                   // Nombre de commandes
            avgOrderValue: { $avg: "$total" },         // Moyenne
            maxOrder: { $max: "$total" },              // Max
            minOrder: { $min: "$total" }               // Min
        }
    },

    // Étape 3 : Trier par ventes totales décroissantes
    {
        $sort: { totalSales: -1 }
    },

    // Étape 4 : Renommer les champs pour clarté
    {
        $project: {
            _id: 0,                                    // Masquer _id
            city: "$_id",                              // Renommer _id en city
            totalSales: { $round: ["$totalSales", 2] }, // Arrondir
            orderCount: 1,
            avgOrderValue: { $round: ["$avgOrderValue", 2] },
            maxOrder: 1,
            minOrder: 1
        }
    }
])
```

**Sortie :**
```javascript
[
    {
        city: "Paris",
        totalSales: 458920.50,
        orderCount: 1250,
        avgOrderValue: 367.14,
        maxOrder: 8999.99,
        minOrder: 15.50
    },
    {
        city: "Lyon",
        totalSales: 289340.20,
        orderCount: 842,
        avgOrderValue: 343.70,
        maxOrder: 7500.00,
        minOrder: 12.99
    }
    // ...
]
```

### Niveau 5 : Agrégations complexes - Analyse multidimensionnelle

**Objectif :** Analyser les ventes par ville ET par mois

```javascript
db.orders.aggregate([
    // Étape 1 : Filtrer (2024 uniquement)
    {
        $match: {
            status: "completed",
            orderDate: {
                $gte: ISODate("2024-01-01"),
                $lt: ISODate("2025-01-01")
            }
        }
    },

    // Étape 2 : Ajouter champs temporels
    {
        $addFields: {
            year: { $year: "$orderDate" },
            month: { $month: "$orderDate" },
            city: "$shippingAddress.city"
        }
    },

    // Étape 3 : Regrouper par ville et mois
    {
        $group: {
            _id: {
                city: "$city",
                year: "$year",
                month: "$month"
            },
            totalSales: { $sum: "$total" },
            orderCount: { $sum: 1 },
            avgOrderValue: { $avg: "$total" },
            customers: { $addToSet: "$customerId" }  // Liste unique de clients
        }
    },

    // Étape 4 : Calculer nombre de clients uniques
    {
        $addFields: {
            uniqueCustomers: { $size: "$customers" }
        }
    },

    // Étape 5 : Reformater
    {
        $project: {
            _id: 0,
            city: "$_id.city",
            year: "$_id.year",
            month: "$_id.month",
            totalSales: { $round: ["$totalSales", 2] },
            orderCount: 1,
            uniqueCustomers: 1,
            avgOrderValue: { $round: ["$avgOrderValue", 2] }
        }
    },

    // Étape 6 : Trier par ville puis mois
    {
        $sort: {
            city: 1,
            year: 1,
            month: 1
        }
    }
])
```

**Sortie :**
```javascript
[
    {
        city: "Lyon",
        year: 2024,
        month: 1,
        totalSales: 23450.80,
        orderCount: 65,
        uniqueCustomers: 48,
        avgOrderValue: 360.78
    },
    {
        city: "Lyon",
        year: 2024,
        month: 2,
        totalSales: 28920.50,
        orderCount: 78,
        uniqueCustomers: 55,
        avgOrderValue: 370.78
    },
    {
        city: "Paris",
        year: 2024,
        month: 1,
        totalSales: 45830.20,
        orderCount: 125,
        uniqueCustomers: 92,
        avgOrderValue: 366.64
    }
    // ...
]
```

### Niveau 6 : Dérouler les tableaux - Analyse par produit

**Objectif :** Analyser les ventes par catégorie de produit

```javascript
db.orders.aggregate([
    // Étape 1 : Filtrer commandes complétées
    {
        $match: {
            status: "completed"
        }
    },

    // Étape 2 : Dérouler le tableau items (un document par item)
    {
        $unwind: "$items"
    },
    /* Transformation :
       Document original avec 3 items → 3 documents (un par item)
       {
           orderId: "ORD001",
           items: [item1, item2, item3]
       }
       devient →
       { orderId: "ORD001", items: item1 }
       { orderId: "ORD001", items: item2 }
       { orderId: "ORD001", items: item3 }
    */

    // Étape 3 : Regrouper par catégorie de produit
    {
        $group: {
            _id: "$items.category",
            totalRevenue: {
                $sum: {
                    $multiply: ["$items.quantity", "$items.price"]
                }
            },
            totalQuantity: { $sum: "$items.quantity" },
            productsSold: { $sum: 1 },
            avgPrice: { $avg: "$items.price" }
        }
    },

    // Étape 4 : Calculer pourcentage de revenu
    {
        $group: {
            _id: null,  // Regrouper tout pour calculer le total
            categories: {
                $push: {  // Conserver les données de chaque catégorie
                    category: "$_id",
                    totalRevenue: "$totalRevenue",
                    totalQuantity: "$totalQuantity",
                    productsSold: "$productsSold",
                    avgPrice: "$avgPrice"
                }
            },
            grandTotal: { $sum: "$totalRevenue" }
        }
    },

    // Étape 5 : Reconstruire avec pourcentages
    {
        $unwind: "$categories"
    },

    {
        $project: {
            _id: 0,
            category: "$categories.category",
            totalRevenue: { $round: ["$categories.totalRevenue", 2] },
            totalQuantity: "$categories.totalQuantity",
            productsSold: "$categories.productsSold",
            avgPrice: { $round: ["$categories.avgPrice", 2] },
            percentOfTotal: {
                $round: [
                    {
                        $multiply: [
                            { $divide: ["$categories.totalRevenue", "$grandTotal"] },
                            100
                        ]
                    },
                    2
                ]
            }
        }
    },

    // Étape 6 : Trier par revenu
    {
        $sort: { totalRevenue: -1 }
    }
])
```

**Sortie :**
```javascript
[
    {
        category: "Informatique",
        totalRevenue: 458920.50,
        totalQuantity: 1285,
        productsSold: 1285,
        avgPrice: 357.14,
        percentOfTotal: 45.82
    },
    {
        category: "Électronique",
        totalRevenue: 298450.30,
        totalQuantity: 2140,
        productsSold: 2140,
        avgPrice: 139.46,
        percentOfTotal: 29.81
    },
    {
        category: "Accessoires",
        totalRevenue: 124680.20,
        totalQuantity: 4580,
        productsSold: 4580,
        avgPrice: 27.22,
        percentOfTotal: 12.45
    }
    // ...
]
```

### Niveau 7 : Jointures - Enrichir avec données clients

**Objectif :** Analyser les ventes avec informations clients complètes

```javascript
db.orders.aggregate([
    // Étape 1 : Filtrer commandes récentes
    {
        $match: {
            status: "completed",
            orderDate: {
                $gte: ISODate("2024-01-01")
            }
        }
    },

    // Étape 2 : Regrouper par client
    {
        $group: {
            _id: "$customerId",
            totalSpent: { $sum: "$total" },
            orderCount: { $sum: 1 },
            avgOrderValue: { $avg: "$total" },
            firstOrder: { $min: "$orderDate" },
            lastOrder: { $max: "$orderDate" }
        }
    },

    // Étape 3 : Joindre avec collection customers (LEFT JOIN)
    {
        $lookup: {
            from: "customers",
            localField: "_id",
            foreignField: "_id",
            as: "customerInfo"
        }
    },

    // Étape 4 : Extraire infos client (dérouler tableau)
    {
        $unwind: {
            path: "$customerInfo",
            preserveNullAndEmptyArrays: true  // Garder même si pas de match
        }
    },

    // Étape 5 : Calculer statut VIP
    {
        $addFields: {
            daysSinceFirstOrder: {
                $divide: [
                    { $subtract: [new Date(), "$firstOrder"] },
                    1000 * 60 * 60 * 24  // Convertir ms en jours
                ]
            },
            vipStatus: {
                $cond: {
                    if: { $gte: ["$totalSpent", 5000] },
                    then: "Platinum",
                    else: {
                        $cond: {
                            if: { $gte: ["$totalSpent", 2000] },
                            then: "Gold",
                            else: {
                                $cond: {
                                    if: { $gte: ["$totalSpent", 1000] },
                                    then: "Silver",
                                    else: "Bronze"
                                }
                            }
                        }
                    }
                }
            }
        }
    },

    // Étape 6 : Projeter résultat final
    {
        $project: {
            _id: 0,
            customerId: "$_id",
            name: "$customerInfo.name",
            email: "$customerInfo.email",
            phone: "$customerInfo.phone",
            totalSpent: { $round: ["$totalSpent", 2] },
            orderCount: 1,
            avgOrderValue: { $round: ["$avgOrderValue", 2] },
            vipStatus: 1,
            customerSince: {
                $dateToString: {
                    format: "%Y-%m-%d",
                    date: "$firstOrder"
                }
            },
            daysSinceFirstOrder: { $round: ["$daysSinceFirstOrder", 0] }
        }
    },

    // Étape 7 : Trier par montant dépensé
    {
        $sort: { totalSpent: -1 }
    },

    // Étape 8 : Limiter aux 100 meilleurs clients
    {
        $limit: 100
    }
])
```

**Sortie :**
```javascript
[
    {
        customerId: ObjectId("..."),
        name: "Alice Dupont",
        email: "alice@example.com",
        phone: "+33612345678",
        totalSpent: 8920.50,
        orderCount: 18,
        avgOrderValue: 495.58,
        vipStatus: "Platinum",
        customerSince: "2023-03-15",
        daysSinceFirstOrder: 298
    },
    {
        customerId: ObjectId("..."),
        name: "Bob Martin",
        email: "bob@example.com",
        phone: "+33698765432",
        totalSpent: 6430.20,
        orderCount: 14,
        avgOrderValue: 459.30,
        vipStatus: "Platinum",
        customerSince: "2023-05-20",
        daysSinceFirstOrder: 232
    }
    // ...
]
```

## Les étapes (stages) principales : aperçu

### Étapes de base (Section 6.3)

| Stage | Rôle | Équivalent SQL |
|-------|------|----------------|
| `$match` | Filtrer documents | WHERE |
| `$project` | Sélectionner/transformer champs | SELECT |
| `$group` | Regrouper et agréger | GROUP BY |
| `$sort` | Trier | ORDER BY |
| `$limit` | Limiter nombre de résultats | LIMIT |
| `$skip` | Sauter des résultats | OFFSET |
| `$count` | Compter résultats | COUNT(*) |

### Étapes avancées (Section 6.4)

| Stage | Rôle | Usage |
|-------|------|-------|
| `$lookup` | Joindre collections | JOIN, sous-requêtes |
| `$unwind` | Dérouler tableaux | Un document par élément |
| `$addFields` | Ajouter/modifier champs | Calculs, transformations |
| `$replaceRoot` | Remplacer document racine | Élever sous-document |
| `$facet` | Pipelines parallèles | Multi-agrégations |
| `$bucket` | Regrouper en intervalles | Histogrammes |
| `$graphLookup` | Requêtes récursives | Hiérarchies, graphes |
| `$merge` | Fusionner dans collection | INSERT/UPDATE |
| `$out` | Écrire dans collection | CREATE TABLE AS |
| `$unionWith` | Union de collections | UNION |

## Les opérateurs d'agrégation : aperçu

### Arithmétiques
```javascript
$add, $subtract, $multiply, $divide, $mod
$abs, $ceil, $floor, $round, $sqrt, $pow
```

### Chaînes
```javascript
$concat, $substr, $toLower, $toUpper, $trim
$split, $strLenCP, $indexOfCP
```

### Dates
```javascript
$year, $month, $dayOfMonth, $hour, $minute
$dateToString, $dateToParts, $dateFromParts
$dateDiff, $dateAdd, $dateSubtract
```

### Tableaux
```javascript
$size, $arrayElemAt, $slice, $concatArrays
$filter, $map, $reduce, $zip
```

### Conditionnels
```javascript
$cond, $ifNull, $switch
```

### Accumulateurs (dans $group)
```javascript
$sum, $avg, $min, $max, $first, $last
$push, $addToSet, $stdDevPop, $stdDevSamp
```

## Cas d'usage réels

### 1. Dashboard analytique

```javascript
// Résumé des ventes en temps réel
db.orders.aggregate([
    {
        $match: {
            orderDate: {
                $gte: ISODate("2024-01-01")
            }
        }
    },
    {
        $facet: {
            // Statistiques globales
            totalStats: [
                {
                    $group: {
                        _id: null,
                        totalRevenue: { $sum: "$total" },
                        totalOrders: { $sum: 1 },
                        avgOrderValue: { $avg: "$total" }
                    }
                }
            ],

            // Top 5 villes
            topCities: [
                {
                    $group: {
                        _id: "$shippingAddress.city",
                        revenue: { $sum: "$total" }
                    }
                },
                { $sort: { revenue: -1 } },
                { $limit: 5 }
            ],

            // Ventes par mois
            monthlyTrend: [
                {
                    $group: {
                        _id: {
                            year: { $year: "$orderDate" },
                            month: { $month: "$orderDate" }
                        },
                        revenue: { $sum: "$total" },
                        orders: { $sum: 1 }
                    }
                },
                { $sort: { "_id.year": 1, "_id.month": 1 } }
            ]
        }
    }
])
```

### 2. Analyse de cohorte

```javascript
// Analyser la rétention client par mois d'inscription
db.customers.aggregate([
    {
        $lookup: {
            from: "orders",
            localField: "_id",
            foreignField: "customerId",
            as: "orders"
        }
    },
    {
        $addFields: {
            cohortMonth: {
                $dateToString: {
                    format: "%Y-%m",
                    date: "$registrationDate"
                }
            },
            firstPurchase: { $min: "$orders.orderDate" },
            totalOrders: { $size: "$orders" }
        }
    },
    {
        $group: {
            _id: "$cohortMonth",
            totalCustomers: { $sum: 1 },
            customersWithPurchase: {
                $sum: { $cond: [{ $gt: ["$totalOrders", 0] }, 1, 0] }
            },
            avgOrdersPerCustomer: { $avg: "$totalOrders" }
        }
    },
    {
        $project: {
            cohort: "$_id",
            totalCustomers: 1,
            customersWithPurchase: 1,
            conversionRate: {
                $multiply: [
                    { $divide: ["$customersWithPurchase", "$totalCustomers"] },
                    100
                ]
            },
            avgOrdersPerCustomer: { $round: ["$avgOrdersPerCustomer", 2] }
        }
    },
    { $sort: { cohort: -1 } }
])
```

### 3. Recommandations de produits

```javascript
// Produits fréquemment achetés ensemble
db.orders.aggregate([
    { $unwind: "$items" },
    {
        $group: {
            _id: "$orderId",
            products: { $addToSet: "$items.productId" }
        }
    },
    { $unwind: "$products" },
    {
        $lookup: {
            from: "orders",
            let: { productId: "$products", orderId: "$_id" },
            pipeline: [
                { $match: { $expr: { $eq: ["$_id", "$$orderId"] } } },
                { $unwind: "$items" },
                {
                    $match: {
                        $expr: { $ne: ["$items.productId", "$$productId"] }
                    }
                },
                {
                    $group: {
                        _id: "$items.productId",
                        count: { $sum: 1 }
                    }
                }
            ],
            as: "relatedProducts"
        }
    },
    { $unwind: "$relatedProducts" },
    {
        $group: {
            _id: {
                product: "$products",
                related: "$relatedProducts._id"
            },
            frequency: { $sum: "$relatedProducts.count" }
        }
    },
    { $sort: { frequency: -1 } },
    {
        $group: {
            _id: "$_id.product",
            recommendations: {
                $push: {
                    productId: "$_id.related",
                    frequency: "$frequency"
                }
            }
        }
    }
])
```

## Optimisation des pipelines

### Principe 1 : $match tôt dans le pipeline

```javascript
// ❌ Mauvais : $match après $unwind
db.orders.aggregate([
    { $unwind: "$items" },
    { $match: { status: "completed" } }  // Trop tard!
])

// ✅ Bon : $match au début
db.orders.aggregate([
    { $match: { status: "completed" } },  // Filtre d'abord
    { $unwind: "$items" }
])
```

### Principe 2 : $project pour réduire la taille des documents

```javascript
// ✅ Projeter tôt pour transporter moins de données
db.orders.aggregate([
    { $match: { status: "completed" } },
    {
        $project: {  // Garder seulement ce qui est nécessaire
            customerId: 1,
            total: 1,
            orderDate: 1
        }
    },
    // ... autres étapes
])
```

### Principe 3 : Utiliser les index

```javascript
// $match peut utiliser un index si c'est la première étape
db.orders.aggregate([
    {
        $match: {
            status: "completed",  // Si index sur status
            orderDate: { $gte: ISODate("2024-01-01") }
        }
    }
    // ...
])

// Vérifier avec explain
db.orders.explain("executionStats").aggregate([...])
```

## Performance : explain() sur les agrégations

```javascript
db.orders.explain("executionStats").aggregate([
    { $match: { status: "completed" } },
    {
        $group: {
            _id: "$customerId",
            total: { $sum: "$total" }
        }
    }
])

// Résultat inclut :
{
    stages: [
        {
            $cursor: {
                queryPlanner: { /* utilisation d'index */ },
                executionStats: { /* performances */ }
            }
        },
        { $group: { /* ... */ } }
    ],
    executionStats: {
        executionTimeMillis: 150,
        // ...
    }
}
```

## Limites et considérations

### 1. Limite de mémoire : 100 Mo par étape

```javascript
// ⚠️ Peut échouer si trop de données en mémoire
db.largeCollection.aggregate([
    {
        $group: {
            _id: "$field",
            data: { $push: "$$ROOT" }  // Accumule tout en mémoire
        }
    }
])

// ✅ Solution : allowDiskUse
db.largeCollection.aggregate(
    [/* pipeline */],
    { allowDiskUse: true }  // Utilise le disque si nécessaire
)
```

### 2. Limite de taille du document : 16 Mo

```javascript
// Le résultat final ne peut pas dépasser 16 Mo
```

### 3. Limite de pipeline : 1000 étapes

```javascript
// Rarement un problème en pratique
```

## Conseils d'apprentissage

### 🎯 Méthodologie recommandée

1. **Commencez simple** : 1-2 étapes
2. **Ajoutez progressivement** : une étape à la fois
3. **Testez à chaque étape** : vérifiez le résultat intermédiaire
4. **Utilisez $out temporaire** : pour déboguer
5. **Documentez vos pipelines** : commentez chaque étape

### 💡 Astuces de débogage

```javascript
// Voir le résultat intermédiaire après chaque étape
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $limit: 1 },  // Limiter pour voir la structure
    // { $group: { /* ... */ } },  // Commenter étapes suivantes
])

// Utiliser $out pour sauvegarder résultat intermédiaire
db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: { /* ... */ } },
    { $out: "temp_debug" }  // Sauvegarder pour inspection
])

db.temp_debug.find().limit(5)
```

### 🔗 Lien avec les autres chapitres

- **Chapitre 4** : La modélisation affecte la complexité des pipelines
- **Chapitre 5** : Les index accélèrent les agrégations
- **Chapitre 15** : Les drivers ont des APIs spécifiques pour l'agrégation
- **Chapitre 17** : Les agrégations complexes nécessitent optimisation

---

### 📌 Points clés à retenir de cette introduction

- Le framework d'agrégation est un pipeline de transformations successives
- Chaque étape reçoit des documents, les transforme, et passe le résultat
- Bien plus puissant que `find()` pour l'analyse et les calculs
- Étapes de base : $match, $project, $group, $sort, $limit
- Étapes avancées : $lookup, $unwind, $facet, $bucket, etc.
- Optimisation : $match tôt, projeter pour réduire, utiliser les index
- `explain()` fonctionne aussi sur les agrégations
- allowDiskUse pour les gros volumes
- Construire progressivement : tester étape par étape

---

**Durée estimée du chapitre** : 10-14 heures de lecture et pratique
**Niveau** : Intermédiaire nécessitant maîtrise des requêtes
**Prérequis** : Chapitres 1-5 complétés, excellente maîtrise des requêtes

🎯 **Prochaine étape** : Dans la section 6.1, nous allons approfondir le framework d'agrégation, comprendre son architecture interne, et établir les bases théoriques qui vous permettront de maîtriser cet outil puissant.

---

**Prochaine section** : 6.1 - Introduction au framework d'agrégation

Prêt à transformer vos données comme jamais ? Allons-y ! 🔮

⏭️ [Introduction au framework d'agrégation](/06-framework-agregation/01-introduction-agregation.md)
