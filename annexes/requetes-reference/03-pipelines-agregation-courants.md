🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.3 - Pipelines d'Agrégation Courants

## Table des matières

1. [Statistiques de Base](#statistiques-de-base)
2. [Groupements et Agrégations](#groupements-et-agr%C3%A9gations)
3. [Jointures (Lookups)](#jointures-lookups)
4. [Transformations de Données](#transformations-de-donn%C3%A9es)
5. [Analyses Temporelles](#analyses-temporelles)
6. [Rapports et Tableaux de Bord](#rapports-et-tableaux-de-bord)
7. [Analyses Avancées](#analyses-avanc%C3%A9es)

---

## Statistiques de Base

### 1.1 - Compter, Somme, Moyenne

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 STATISTIQUES SIMPLES
// ============================================

// 💡 Objectif
// Calculer des statistiques de base sur une collection

// 🎯 Cas d'usage
// - Tableaux de bord
// - Rapports analytiques
// - KPIs

// ============================================
// PIPELINE
// ============================================

db.orders.aggregate([
  {
    $match: {
      status: "completed"
    }
  },
  {
    $group: {
      _id: null,
      totalOrders: { $sum: 1 },
      totalRevenue: { $sum: "$amount" },
      averageOrderValue: { $avg: "$amount" },
      minOrder: { $min: "$amount" },
      maxOrder: { $max: "$amount" }
    }
  },
  {
    $project: {
      _id: 0,
      totalOrders: 1,
      totalRevenue: { $round: ["$totalRevenue", 2] },
      averageOrderValue: { $round: ["$averageOrderValue", 2] },
      minOrder: 1,
      maxOrder: 1
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    totalOrders: 15234,
    totalRevenue: 1523400.50,
    averageOrderValue: 100.05,
    minOrder: 5.99,
    maxOrder: 2500.00
  }
]

// ============================================
// 💡 VARIANTES
// ============================================

// Avec filtrage par période
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2024-02-01")
      }
    }
  },
  {
    $group: {
      _id: null,
      count: { $sum: 1 },
      total: { $sum: "$amount" },
      avg: { $avg: "$amount" }
    }
  }
])

// Statistiques par catégorie
db.products.aggregate([
  {
    $group: {
      _id: "$category",
      count: { $sum: 1 },
      avgPrice: { $avg: "$price" },
      totalStock: { $sum: "$stock" }
    }
  },
  { $sort: { count: -1 } }
])
```

---

### 1.2 - Top N / Bottom N

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 TOP N ÉLÉMENTS
// ============================================

// 💡 Objectif
// Obtenir les N meilleurs ou pires éléments selon un critère

// 🎯 Cas d'usage
// - Top vendeurs
// - Produits les plus populaires
// - Clients les plus actifs

// 🔑 Index recommandé
// db.products.createIndex({ sales: -1 })

// ============================================
// PIPELINE - TOP 10 PRODUITS
// ============================================

db.products.aggregate([
  {
    $match: {
      active: true
    }
  },
  {
    $sort: { sales: -1 }
  },
  {
    $limit: 10
  },
  {
    $project: {
      name: 1,
      sales: 1,
      revenue: { $multiply: ["$sales", "$price"] },
      category: 1
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: ObjectId("..."),
    name: "Product A",
    sales: 5234,
    revenue: 104680.00,
    category: "Electronics"
  },
  {
    _id: ObjectId("..."),
    name: "Product B",
    sales: 4521,
    revenue: 90420.00,
    category: "Books"
  }
  // ... 8 more
]

// ============================================
// 💡 VARIANTES
// ============================================

// Bottom 10 (moins vendus)
db.products.aggregate([
  {
    $match: { active: true }
  },
  { $sort: { sales: 1 } },  // Tri croissant
  { $limit: 10 }
])

// Top 5 par catégorie
db.products.aggregate([
  {
    $sort: { sales: -1 }
  },
  {
    $group: {
      _id: "$category",
      topProducts: {
        $push: {
          name: "$name",
          sales: "$sales"
        }
      }
    }
  },
  {
    $project: {
      category: "$_id",
      topProducts: { $slice: ["$topProducts", 5] }
    }
  }
])

// Top 10 avec rang
db.products.aggregate([
  {
    $match: { active: true }
  },
  { $sort: { sales: -1 } },
  { $limit: 10 },
  {
    $group: {
      _id: null,
      products: { $push: "$$ROOT" }
    }
  },
  {
    $unwind: {
      path: "$products",
      includeArrayIndex: "rank"
    }
  },
  {
    $replaceRoot: {
      newRoot: {
        $mergeObjects: [
          "$products",
          { rank: { $add: ["$rank", 1] } }
        ]
      }
    }
  },
  {
    $project: {
      rank: 1,
      name: 1,
      sales: 1,
      category: 1
    }
  }
])
```

---

## Groupements et Agrégations

### 2.1 - Groupement par date (jour, mois, année)

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 AGRÉGATION PAR PÉRIODE
// ============================================

// 💡 Objectif
// Grouper et agréger des données par période temporelle

// 🎯 Cas d'usage
// - Ventes par mois
// - Trafic par jour
// - Évolution temporelle

// 🔑 Index recommandé
// db.orders.createIndex({ createdAt: -1 })

// ============================================
// PIPELINE - PAR MOIS
// ============================================

db.orders.aggregate([
  {
    $match: {
      createdAt: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2025-01-01")
      }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      totalOrders: { $sum: 1 },
      totalRevenue: { $sum: "$amount" },
      avgOrderValue: { $avg: "$amount" }
    }
  },
  {
    $sort: { "_id.year": 1, "_id.month": 1 }
  },
  {
    $project: {
      _id: 0,
      period: {
        $concat: [
          { $toString: "$_id.year" },
          "-",
          {
            $cond: [
              { $lt: ["$_id.month", 10] },
              { $concat: ["0", { $toString: "$_id.month" }] },
              { $toString: "$_id.month" }
            ]
          }
        ]
      },
      totalOrders: 1,
      totalRevenue: { $round: ["$totalRevenue", 2] },
      avgOrderValue: { $round: ["$avgOrderValue", 2] }
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    period: "2024-01",
    totalOrders: 1234,
    totalRevenue: 123400.50,
    avgOrderValue: 100.00
  },
  {
    period: "2024-02",
    totalOrders: 1456,
    totalRevenue: 145600.00,
    avgOrderValue: 100.00
  }
  // ...
]

// ============================================
// 💡 VARIANTES
// ============================================

// Par jour
db.orders.aggregate([
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 30 * 86400000) }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d", date: "$createdAt" }
      },
      count: { $sum: 1 },
      revenue: { $sum: "$amount" }
    }
  },
  { $sort: { _id: 1 } }
])

// Par semaine
db.orders.aggregate([
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 90 * 86400000) }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-W%V", date: "$createdAt" }
      },
      count: { $sum: 1 },
      revenue: { $sum: "$amount" }
    }
  },
  { $sort: { _id: 1 } }
])

// Par heure (dernières 24h)
db.events.aggregate([
  {
    $match: {
      timestamp: { $gte: new Date(Date.now() - 86400000) }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d %H:00", date: "$timestamp" }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])

// Par trimestre
db.sales.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$date" },
        quarter: {
          $ceil: { $divide: [{ $month: "$date" }, 3] }
        }
      },
      revenue: { $sum: "$amount" }
    }
  },
  {
    $project: {
      _id: 0,
      period: {
        $concat: [
          { $toString: "$_id.year" },
          "-Q",
          { $toString: "$_id.quarter" }
        ]
      },
      revenue: 1
    }
  },
  { $sort: { period: 1 } }
])
```

---

### 2.2 - Groupement multi-niveaux

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 GROUPEMENT HIÉRARCHIQUE
// ============================================

// 💡 Objectif
// Créer des agrégations avec plusieurs niveaux de groupement

// 🎯 Cas d'usage
// - Ventes par pays > région > ville
// - Catégorie > Sous-catégorie > Produit
// - Département > Équipe > Membre

// ============================================
// PIPELINE - VENTES PAR PAYS ET RÉGION
// ============================================

db.orders.aggregate([
  {
    $match: {
      status: "completed"
    }
  },
  {
    $group: {
      _id: {
        country: "$customer.country",
        region: "$customer.region"
      },
      orders: { $sum: 1 },
      revenue: { $sum: "$amount" },
      customers: { $addToSet: "$customerId" }
    }
  },
  {
    $project: {
      _id: 0,
      country: "$_id.country",
      region: "$_id.region",
      orders: 1,
      revenue: { $round: ["$revenue", 2] },
      uniqueCustomers: { $size: "$customers" }
    }
  },
  {
    $sort: { country: 1, region: 1 }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    country: "France",
    region: "Île-de-France",
    orders: 5234,
    revenue: 523400.00,
    uniqueCustomers: 1234
  },
  {
    country: "France",
    region: "Provence",
    orders: 3421,
    revenue: 342100.00,
    uniqueCustomers: 892
  }
  // ...
]

// ============================================
// 💡 VARIANTE - AVEC SOUS-TOTAUX
// ============================================

db.orders.aggregate([
  {
    $match: { status: "completed" }
  },
  {
    $facet: {
      // Par pays et région
      byRegion: [
        {
          $group: {
            _id: {
              country: "$customer.country",
              region: "$customer.region"
            },
            orders: { $sum: 1 },
            revenue: { $sum: "$amount" }
          }
        },
        {
          $project: {
            _id: 0,
            level: "region",
            country: "$_id.country",
            region: "$_id.region",
            orders: 1,
            revenue: { $round: ["$revenue", 2] }
          }
        }
      ],
      // Totaux par pays
      byCountry: [
        {
          $group: {
            _id: "$customer.country",
            orders: { $sum: 1 },
            revenue: { $sum: "$amount" }
          }
        },
        {
          $project: {
            _id: 0,
            level: "country",
            country: "$_id",
            region: null,
            orders: 1,
            revenue: { $round: ["$revenue", 2] }
          }
        }
      ],
      // Total global
      total: [
        {
          $group: {
            _id: null,
            orders: { $sum: 1 },
            revenue: { $sum: "$amount" }
          }
        },
        {
          $project: {
            _id: 0,
            level: "total",
            country: null,
            region: null,
            orders: 1,
            revenue: { $round: ["$revenue", 2] }
          }
        }
      ]
    }
  },
  {
    $project: {
      results: {
        $concatArrays: ["$byRegion", "$byCountry", "$total"]
      }
    }
  },
  { $unwind: "$results" },
  { $replaceRoot: { newRoot: "$results" } }
])
```

---

## Jointures (Lookups)

### 3.1 - Jointure simple (One-to-Many)

**🟡 Niveau : Intermédiaire** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 JOINTURE SIMPLE
// ============================================

// 💡 Objectif
// Combiner des documents de collections liées

// 🎯 Cas d'usage
// - Commandes avec détails clients
// - Produits avec catégories
// - Posts avec auteurs

// 🔑 Index recommandés
// db.customers.createIndex({ _id: 1 })
// db.orders.createIndex({ customerId: 1 })

// ============================================
// PIPELINE - COMMANDES AVEC CLIENTS
// ============================================

db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: { $gte: new Date("2024-01-01") }
    }
  },
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  {
    $unwind: "$customer"
  },
  {
    $project: {
      orderNumber: 1,
      amount: 1,
      createdAt: 1,
      customerName: "$customer.name",
      customerEmail: "$customer.email",
      customerCountry: "$customer.country"
    }
  },
  { $limit: 100 }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: ObjectId("..."),
    orderNumber: "ORD-2024-001",
    amount: 150.50,
    createdAt: ISODate("2024-01-15T10:30:00Z"),
    customerName: "Alice Dupont",
    customerEmail: "alice@example.com",
    customerCountry: "France"
  }
  // ...
]

// ============================================
// 💡 VARIANTES
// ============================================

// Avec agrégation après jointure
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },
  {
    $group: {
      _id: "$customer.country",
      totalOrders: { $sum: 1 },
      totalRevenue: { $sum: "$amount" },
      customers: { $addToSet: "$customerId" }
    }
  },
  {
    $project: {
      country: "$_id",
      totalOrders: 1,
      totalRevenue: { $round: ["$totalRevenue", 2] },
      uniqueCustomers: { $size: "$customers" }
    }
  },
  { $sort: { totalRevenue: -1 } }
])

// Lookup avec pipeline (plus de contrôle)
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      let: { customerId: "$customerId" },
      pipeline: [
        {
          $match: {
            $expr: { $eq: ["$_id", "$$customerId"] },
            active: true  // Filtre supplémentaire
          }
        },
        {
          $project: {
            name: 1,
            email: 1,
            tier: 1
          }
        }
      ],
      as: "customer"
    }
  },
  {
    $match: {
      customer: { $ne: [] }  // Seulement si client trouvé
    }
  },
  { $unwind: "$customer" }
])
```

---

### 3.2 - Jointures multiples

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 JOINTURES MULTIPLES
// ============================================

// 💡 Objectif
// Combiner des données de plusieurs collections

// 🎯 Cas d'usage
// - Vue complète d'une entité
// - Rapports complexes
// - Données dénormalisées

// 🔑 Index recommandés
// db.customers.createIndex({ _id: 1 })
// db.products.createIndex({ _id: 1 })
// db.orders.createIndex({ customerId: 1, productId: 1 })

// ============================================
// PIPELINE - COMMANDES COMPLÈTES
// ============================================

db.orders.aggregate([
  // Jointure avec customers
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  },
  { $unwind: "$customer" },

  // Jointure avec products
  {
    $lookup: {
      from: "products",
      localField: "items.productId",
      foreignField: "_id",
      as: "products"
    }
  },

  // Jointure avec shipping
  {
    $lookup: {
      from: "shipments",
      localField: "_id",
      foreignField: "orderId",
      as: "shipment"
    }
  },
  { $unwind: { path: "$shipment", preserveNullAndEmptyArrays: true } },

  // Projection finale
  {
    $project: {
      orderNumber: 1,
      orderDate: "$createdAt",
      status: 1,
      customer: {
        name: "$customer.name",
        email: "$customer.email"
      },
      products: {
        $map: {
          input: "$items",
          as: "item",
          in: {
            $mergeObjects: [
              "$$item",
              {
                productInfo: {
                  $arrayElemAt: [
                    {
                      $filter: {
                        input: "$products",
                        cond: { $eq: ["$$this._id", "$$item.productId"] }
                      }
                    },
                    0
                  ]
                }
              }
            ]
          }
        }
      },
      shipping: {
        carrier: "$shipment.carrier",
        trackingNumber: "$shipment.trackingNumber",
        status: "$shipment.status"
      },
      totalAmount: 1
    }
  },
  { $limit: 10 }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: ObjectId("..."),
    orderNumber: "ORD-2024-001",
    orderDate: ISODate("2024-01-15T10:30:00Z"),
    status: "shipped",
    customer: {
      name: "Alice Dupont",
      email: "alice@example.com"
    },
    products: [
      {
        productId: ObjectId("..."),
        quantity: 2,
        price: 25.00,
        productInfo: {
          _id: ObjectId("..."),
          name: "Product A",
          category: "Electronics"
        }
      }
    ],
    shipping: {
      carrier: "UPS",
      trackingNumber: "1Z999AA10123456784",
      status: "in_transit"
    },
    totalAmount: 50.00
  }
]
```

---

## Transformations de Données

### 4.1 - Restructuration de documents

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 RESTRUCTURATION
// ============================================

// 💡 Objectif
// Transformer la structure des documents

// 🎯 Cas d'usage
// - Mise en forme pour API
// - Export vers autre système
// - Normalisation/Dénormalisation

// ============================================
// PIPELINE - APLATIR UN DOCUMENT
// ============================================

// Document source:
// {
//   _id: ObjectId("..."),
//   customer: {
//     name: "Alice",
//     address: {
//       street: "5 rue de la Paix",
//       city: "Paris",
//       country: "France"
//     }
//   },
//   items: [...]
// }

db.orders.aggregate([
  {
    $project: {
      _id: 1,
      orderNumber: 1,
      customerName: "$customer.name",
      customerEmail: "$customer.email",
      street: "$customer.address.street",
      city: "$customer.address.city",
      country: "$customer.address.country",
      itemCount: { $size: "$items" },
      totalAmount: {
        $reduce: {
          input: "$items",
          initialValue: 0,
          in: {
            $add: [
              "$$value",
              { $multiply: ["$$this.quantity", "$$this.price"] }
            ]
          }
        }
      }
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: ObjectId("..."),
    orderNumber: "ORD-2024-001",
    customerName: "Alice",
    customerEmail: "alice@example.com",
    street: "5 rue de la Paix",
    city: "Paris",
    country: "France",
    itemCount: 3,
    totalAmount: 150.50
  }
]

// ============================================
// 💡 VARIANTES
// ============================================

// Transformer tableau en objet
db.users.aggregate([
  {
    $project: {
      name: 1,
      preferences: {
        $arrayToObject: {
          $map: {
            input: "$settings",
            as: "setting",
            in: {
              k: "$$setting.key",
              v: "$$setting.value"
            }
          }
        }
      }
    }
  }
])

// Créer des champs calculés
db.products.aggregate([
  {
    $addFields: {
      discountedPrice: {
        $cond: {
          if: { $gt: ["$discount", 0] },
          then: {
            $subtract: [
              "$price",
              { $multiply: ["$price", { $divide: ["$discount", 100] }] }
            ]
          },
          else: "$price"
        }
      },
      inStock: { $gt: ["$stock", 0] },
      priceCategory: {
        $switch: {
          branches: [
            { case: { $lt: ["$price", 20] }, then: "budget" },
            { case: { $lt: ["$price", 100] }, then: "standard" },
            { case: { $gte: ["$price", 100] }, then: "premium" }
          ],
          default: "unknown"
        }
      }
    }
  }
])
```

---

### 4.2 - Dépivotage (Unwind)

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 DÉPIVOTAGE DE TABLEAUX
// ============================================

// 💡 Objectif
// Transformer un tableau en documents séparés

// 🎯 Cas d'usage
// - Analyse d'items individuels
// - Statistiques détaillées
// - Export ligne par ligne

// ============================================
// PIPELINE - EXPLOSER LES ITEMS
// ============================================

// Document source:
// {
//   _id: ObjectId("..."),
//   orderNumber: "ORD-001",
//   items: [
//     { productId: 1, quantity: 2, price: 25 },
//     { productId: 2, quantity: 1, price: 50 }
//   ]
// }

db.orders.aggregate([
  { $unwind: "$items" },
  {
    $project: {
      orderNumber: 1,
      orderDate: "$createdAt",
      productId: "$items.productId",
      quantity: "$items.quantity",
      price: "$items.price",
      lineTotal: {
        $multiply: ["$items.quantity", "$items.price"]
      }
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: ObjectId("..."),
    orderNumber: "ORD-001",
    orderDate: ISODate("2024-01-15"),
    productId: 1,
    quantity: 2,
    price: 25,
    lineTotal: 50
  },
  {
    _id: ObjectId("..."),
    orderNumber: "ORD-001",
    orderDate: ISODate("2024-01-15"),
    productId: 2,
    quantity: 1,
    price: 50,
    lineTotal: 50
  }
]

// ============================================
// 💡 VARIANTES
// ============================================

// Avec préservation si tableau vide
db.orders.aggregate([
  {
    $unwind: {
      path: "$items",
      preserveNullAndEmptyArrays: true
    }
  }
])

// Avec index de position
db.orders.aggregate([
  {
    $unwind: {
      path: "$items",
      includeArrayIndex: "itemIndex"
    }
  },
  {
    $addFields: {
      itemNumber: { $add: ["$itemIndex", 1] }
    }
  }
])

// Analyse agrégée après unwind
db.orders.aggregate([
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.productId",
      totalQuantity: { $sum: "$items.quantity" },
      totalRevenue: {
        $sum: { $multiply: ["$items.quantity", "$items.price"] }
      },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalRevenue: -1 } }
])
```

---

## Analyses Temporelles

### 5.1 - Séries temporelles

**🟡 Niveau : Intermédiaire** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 ANALYSE DE SÉRIES TEMPORELLES
// ============================================

// 💡 Objectif
// Analyser l'évolution temporelle avec calculs de tendances

// 🎯 Cas d'usage
// - Croissance MoM / YoY
// - Détection de tendances
// - Prévisions

// 🔑 Index recommandé
// db.sales.createIndex({ date: 1 })

// ============================================
// PIPELINE - CROISSANCE MENSUELLE
// ============================================

db.sales.aggregate([
  {
    $match: {
      date: { $gte: new Date("2023-01-01") }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m", date: "$date" }
      },
      revenue: { $sum: "$amount" },
      orders: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } },
  {
    $group: {
      _id: null,
      months: {
        $push: {
          period: "$_id",
          revenue: "$revenue",
          orders: "$orders"
        }
      }
    }
  },
  {
    $project: {
      _id: 0,
      results: {
        $map: {
          input: { $range: [0, { $size: "$months" }] },
          as: "idx",
          in: {
            $let: {
              vars: {
                current: { $arrayElemAt: ["$months", "$$idx"] },
                previous: {
                  $arrayElemAt: ["$months", { $subtract: ["$$idx", 1] }]
                }
              },
              in: {
                period: "$$current.period",
                revenue: "$$current.revenue",
                orders: "$$current.orders",
                revenueGrowth: {
                  $cond: {
                    if: { $gt: ["$$idx", 0] },
                    then: {
                      $multiply: [
                        {
                          $divide: [
                            {
                              $subtract: [
                                "$$current.revenue",
                                "$$previous.revenue"
                              ]
                            },
                            "$$previous.revenue"
                          ]
                        },
                        100
                      ]
                    },
                    else: null
                  }
                },
                orderGrowth: {
                  $cond: {
                    if: { $gt: ["$$idx", 0] },
                    then: {
                      $multiply: [
                        {
                          $divide: [
                            {
                              $subtract: [
                                "$$current.orders",
                                "$$previous.orders"
                              ]
                            },
                            "$$previous.orders"
                          ]
                        },
                        100
                      ]
                    },
                    else: null
                  }
                }
              }
            }
          }
        }
      }
    }
  },
  { $unwind: "$results" },
  { $replaceRoot: { newRoot: "$results" } },
  {
    $project: {
      period: 1,
      revenue: { $round: ["$revenue", 2] },
      orders: 1,
      revenueGrowth: {
        $cond: {
          if: { $ne: ["$revenueGrowth", null] },
          then: { $round: ["$revenueGrowth", 2] },
          else: null
        }
      },
      orderGrowth: {
        $cond: {
          if: { $ne: ["$orderGrowth", null] },
          then: { $round: ["$orderGrowth", 2] },
          else: null
        }
      }
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    period: "2023-01",
    revenue: 100000.00,
    orders: 1000,
    revenueGrowth: null,
    orderGrowth: null
  },
  {
    period: "2023-02",
    revenue: 110000.00,
    orders: 1100,
    revenueGrowth: 10.00,
    orderGrowth: 10.00
  },
  {
    period: "2023-03",
    revenue: 121000.00,
    orders: 1210,
    revenueGrowth: 10.00,
    orderGrowth: 10.00
  }
  // ...
]

// ============================================
// 💡 VARIANTE SIMPLIFIÉE - COMPARAISON YoY
// ============================================

db.sales.aggregate([
  {
    $group: {
      _id: {
        year: { $year: "$date" },
        month: { $month: "$date" }
      },
      revenue: { $sum: "$amount" }
    }
  },
  {
    $sort: { "_id.year": 1, "_id.month": 1 }
  },
  {
    $project: {
      _id: 0,
      period: {
        $concat: [
          { $toString: "$_id.year" },
          "-",
          {
            $cond: [
              { $lt: ["$_id.month", 10] },
              { $concat: ["0", { $toString: "$_id.month" }] },
              { $toString: "$_id.month" }
            ]
          }
        ]
      },
      year: "$_id.year",
      month: "$_id.month",
      revenue: { $round: ["$revenue", 2] }
    }
  }
])
```

---

### 5.2 - Moyennes mobiles

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 MOYENNES MOBILES
// ============================================

// 💡 Objectif
// Calculer des moyennes mobiles pour lisser les tendances

// 🎯 Cas d'usage
// - Analyse de tendances
// - Détection d'anomalies
// - Prévisions

// ============================================
// PIPELINE - MOYENNE MOBILE 7 JOURS
// ============================================

db.sales.aggregate([
  {
    $match: {
      date: { $gte: new Date("2024-01-01") }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d", date: "$date" }
      },
      dailyRevenue: { $sum: "$amount" }
    }
  },
  { $sort: { _id: 1 } },
  {
    $group: {
      _id: null,
      days: {
        $push: {
          date: "$_id",
          revenue: "$dailyRevenue"
        }
      }
    }
  },
  {
    $project: {
      results: {
        $map: {
          input: { $range: [0, { $size: "$days" }] },
          as: "idx",
          in: {
            $let: {
              vars: {
                current: { $arrayElemAt: ["$days", "$$idx"] },
                window: {
                  $slice: [
                    "$days",
                    { $max: [0, { $subtract: ["$$idx", 6] }] },
                    7
                  ]
                }
              },
              in: {
                date: "$$current.date",
                revenue: "$$current.revenue",
                movingAvg7d: {
                  $avg: "$$window.revenue"
                }
              }
            }
          }
        }
      }
    }
  },
  { $unwind: "$results" },
  { $replaceRoot: { newRoot: "$results" } },
  {
    $project: {
      date: 1,
      revenue: { $round: ["$revenue", 2] },
      movingAvg7d: { $round: ["$movingAvg7d", 2] }
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    date: "2024-01-01",
    revenue: 10000.00,
    movingAvg7d: 10000.00
  },
  {
    date: "2024-01-02",
    revenue: 12000.00,
    movingAvg7d: 11000.00
  },
  {
    date: "2024-01-08",
    revenue: 11500.00,
    movingAvg7d: 11250.00  // Moyenne des 7 derniers jours
  }
  // ...
]
```

---

## Rapports et Tableaux de Bord

### 6.1 - Dashboard KPIs

**🟡 Niveau : Intermédiaire** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 TABLEAU DE BORD AVEC KPIS
// ============================================

// 💡 Objectif
// Générer plusieurs KPIs en une seule requête

// 🎯 Cas d'usage
// - Dashboards temps réel
// - Rapports exécutifs
// - Monitoring business

// ============================================
// PIPELINE - KPIs MULTIPLES
// ============================================

db.orders.aggregate([
  {
    $facet: {
      // KPI 1: Revenus du jour
      today: [
        {
          $match: {
            createdAt: {
              $gte: new Date(new Date().setHours(0, 0, 0, 0))
            },
            status: "completed"
          }
        },
        {
          $group: {
            _id: null,
            revenue: { $sum: "$amount" },
            orders: { $sum: 1 }
          }
        }
      ],

      // KPI 2: Revenus du mois
      thisMonth: [
        {
          $match: {
            createdAt: {
              $gte: new Date(new Date().getFullYear(), new Date().getMonth(), 1)
            },
            status: "completed"
          }
        },
        {
          $group: {
            _id: null,
            revenue: { $sum: "$amount" },
            orders: { $sum: 1 }
          }
        }
      ],

      // KPI 3: Top 5 clients
      topCustomers: [
        {
          $match: { status: "completed" }
        },
        {
          $group: {
            _id: "$customerId",
            totalSpent: { $sum: "$amount" },
            orderCount: { $sum: 1 }
          }
        },
        { $sort: { totalSpent: -1 } },
        { $limit: 5 },
        {
          $lookup: {
            from: "customers",
            localField: "_id",
            foreignField: "_id",
            as: "customer"
          }
        },
        { $unwind: "$customer" },
        {
          $project: {
            customerId: "$_id",
            customerName: "$customer.name",
            totalSpent: { $round: ["$totalSpent", 2] },
            orderCount: 1
          }
        }
      ],

      // KPI 4: Produits populaires
      topProducts: [
        { $unwind: "$items" },
        {
          $group: {
            _id: "$items.productId",
            quantity: { $sum: "$items.quantity" },
            revenue: {
              $sum: { $multiply: ["$items.quantity", "$items.price"] }
            }
          }
        },
        { $sort: { quantity: -1 } },
        { $limit: 5 }
      ],

      // KPI 5: Statistiques globales
      stats: [
        {
          $group: {
            _id: null,
            totalRevenue: { $sum: "$amount" },
            totalOrders: { $sum: 1 },
            avgOrderValue: { $avg: "$amount" },
            uniqueCustomers: { $addToSet: "$customerId" }
          }
        },
        {
          $project: {
            _id: 0,
            totalRevenue: { $round: ["$totalRevenue", 2] },
            totalOrders: 1,
            avgOrderValue: { $round: ["$avgOrderValue", 2] },
            uniqueCustomers: { $size: "$uniqueCustomers" }
          }
        }
      ]
    }
  },
  {
    $project: {
      dashboard: {
        today: { $arrayElemAt: ["$today", 0] },
        thisMonth: { $arrayElemAt: ["$thisMonth", 0] },
        topCustomers: "$topCustomers",
        topProducts: "$topProducts",
        globalStats: { $arrayElemAt: ["$stats", 0] }
      }
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    dashboard: {
      today: {
        revenue: 15234.50,
        orders: 152
      },
      thisMonth: {
        revenue: 523400.00,
        orders: 5234
      },
      topCustomers: [
        {
          customerId: ObjectId("..."),
          customerName: "Alice Dupont",
          totalSpent: 15234.00,
          orderCount: 45
        }
        // ... 4 more
      ],
      topProducts: [
        {
          _id: ObjectId("..."),
          quantity: 523,
          revenue: 26150.00
        }
        // ... 4 more
      ],
      globalStats: {
        totalRevenue: 5234000.00,
        totalOrders: 52340,
        avgOrderValue: 100.00,
        uniqueCustomers: 12345
      }
    }
  }
]
```

---

## Analyses Avancées

### 7.1 - Analyse de cohortes

**🔴 Niveau : Avancé** | **🐌 Performance : Lente**

```javascript
// ============================================
// 📌 ANALYSE DE COHORTES
// ============================================

// 💡 Objectif
// Analyser le comportement des utilisateurs par cohorte d'inscription

// 🎯 Cas d'usage
// - Rétention utilisateurs
// - Analyse de churn
// - Lifetime value

// ============================================
// PIPELINE - RÉTENTION PAR COHORTE
// ============================================

db.users.aggregate([
  {
    $lookup: {
      from: "orders",
      localField: "_id",
      foreignField: "customerId",
      as: "orders"
    }
  },
  {
    $project: {
      _id: 1,
      signupDate: "$createdAt",
      signupMonth: {
        $dateToString: { format: "%Y-%m", date: "$createdAt" }
      },
      orders: {
        $map: {
          input: "$orders",
          as: "order",
          in: {
            date: "$$order.createdAt",
            month: {
              $dateToString: { format: "%Y-%m", date: "$$order.createdAt" }
            },
            amount: "$$order.amount"
          }
        }
      }
    }
  },
  { $unwind: { path: "$orders", preserveNullAndEmptyArrays: true } },
  {
    $group: {
      _id: {
        cohort: "$signupMonth",
        orderMonth: "$orders.month"
      },
      users: { $addToSet: "$_id" },
      revenue: { $sum: "$orders.amount" }
    }
  },
  {
    $group: {
      _id: "$_id.cohort",
      months: {
        $push: {
          month: "$_id.orderMonth",
          userCount: { $size: "$users" },
          revenue: "$revenue"
        }
      }
    }
  },
  { $sort: { _id: 1 } }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: "2024-01",  // Cohorte de janvier
    months: [
      {
        month: "2024-01",
        userCount: 100,  // 100 users ont commandé en janvier
        revenue: 10000
      },
      {
        month: "2024-02",
        userCount: 75,   // 75 sont revenus en février (75% rétention)
        revenue: 8500
      },
      {
        month: "2024-03",
        userCount: 60,   // 60 sont revenus en mars (60% rétention)
        revenue: 7200
      }
    ]
  }
]
```

---

### 7.2 - Analyse RFM (Recency, Frequency, Monetary)

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 ANALYSE RFM
// ============================================

// 💡 Objectif
// Segmenter les clients selon Récence, Fréquence, Montant

// 🎯 Cas d'usage
// - Segmentation client
// - Ciblage marketing
// - Priorisation commerciale

// ============================================
// PIPELINE - CALCUL RFM
// ============================================

db.orders.aggregate([
  {
    $match: {
      status: "completed"
    }
  },
  {
    $group: {
      _id: "$customerId",
      lastOrderDate: { $max: "$createdAt" },
      orderCount: { $sum: 1 },
      totalSpent: { $sum: "$amount" }
    }
  },
  {
    $addFields: {
      recency: {
        $divide: [
          { $subtract: [new Date(), "$lastOrderDate"] },
          86400000  // Jours depuis dernière commande
        ]
      }
    }
  },
  {
    $project: {
      customerId: "$_id",
      recencyDays: { $round: ["$recency", 0] },
      frequency: "$orderCount",
      monetary: { $round: ["$totalSpent", 2] }
    }
  },
  // Calculer les scores RFM (1-5)
  {
    $addFields: {
      recencyScore: {
        $switch: {
          branches: [
            { case: { $lte: ["$recencyDays", 30] }, then: 5 },
            { case: { $lte: ["$recencyDays", 60] }, then: 4 },
            { case: { $lte: ["$recencyDays", 90] }, then: 3 },
            { case: { $lte: ["$recencyDays", 180] }, then: 2 }
          ],
          default: 1
        }
      },
      frequencyScore: {
        $switch: {
          branches: [
            { case: { $gte: ["$frequency", 20] }, then: 5 },
            { case: { $gte: ["$frequency", 10] }, then: 4 },
            { case: { $gte: ["$frequency", 5] }, then: 3 },
            { case: { $gte: ["$frequency", 2] }, then: 2 }
          ],
          default: 1
        }
      },
      monetaryScore: {
        $switch: {
          branches: [
            { case: { $gte: ["$monetary", 5000] }, then: 5 },
            { case: { $gte: ["$monetary", 2000] }, then: 4 },
            { case: { $gte: ["$monetary", 1000] }, then: 3 },
            { case: { $gte: ["$monetary", 500] }, then: 2 }
          ],
          default: 1
        }
      }
    }
  },
  {
    $addFields: {
      rfmScore: {
        $concat: [
          { $toString: "$recencyScore" },
          { $toString: "$frequencyScore" },
          { $toString: "$monetaryScore" }
        ]
      },
      rfmTotal: {
        $add: ["$recencyScore", "$frequencyScore", "$monetaryScore"]
      }
    }
  },
  {
    $addFields: {
      segment: {
        $switch: {
          branches: [
            {
              case: { $gte: ["$rfmTotal", 13] },
              then: "Champions"
            },
            {
              case: {
                $and: [
                  { $gte: ["$rfmTotal", 10] },
                  { $gte: ["$recencyScore", 4] }
                ]
              },
              then: "Loyal Customers"
            },
            {
              case: {
                $and: [
                  { $gte: ["$rfmTotal", 9] },
                  { $gte: ["$recencyScore", 3] }
                ]
              },
              then: "Potential Loyalists"
            },
            {
              case: { $lte: ["$recencyScore", 2] },
              then: "At Risk"
            },
            {
              case: { $eq: ["$recencyScore", 1] },
              then: "Lost"
            }
          ],
          default: "Regular"
        }
      }
    }
  },
  { $sort: { rfmTotal: -1 } }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: ObjectId("..."),
    customerId: ObjectId("..."),
    recencyDays: 15,
    frequency: 25,
    monetary: 7500.00,
    recencyScore: 5,
    frequencyScore: 5,
    monetaryScore: 5,
    rfmScore: "555",
    rfmTotal: 15,
    segment: "Champions"
  },
  {
    _id: ObjectId("..."),
    customerId: ObjectId("..."),
    recencyDays: 45,
    frequency: 12,
    monetary: 3500.00,
    recencyScore: 4,
    frequencyScore: 4,
    monetaryScore: 4,
    rfmScore: "444",
    rfmTotal: 12,
    segment: "Loyal Customers"
  }
  // ...
]

// ============================================
// 💡 DISTRIBUTION DES SEGMENTS
// ============================================

db.orders.aggregate([
  // ... (pipeline RFM ci-dessus jusqu'au calcul du segment)
  {
    $group: {
      _id: "$segment",
      customerCount: { $sum: 1 },
      totalRevenue: { $sum: "$monetary" },
      avgRecency: { $avg: "$recencyDays" },
      avgFrequency: { $avg: "$frequency" }
    }
  },
  { $sort: { customerCount: -1 } }
])
```

---

**💡 Conseil final** : Ces pipelines peuvent être combinés et adaptés selon vos besoins spécifiques. Testez toujours avec `explain()` pour optimiser les performances, et créez les index appropriés sur les champs utilisés dans les `$match` et `$sort`.

**🎯 Optimisation** : Pour de grandes collections, envisagez de créer des vues matérialisées avec `$merge` ou `$out` pour les rapports récurrents.

⏭️ [Configuration de Référence par Cas d'Usage](/annexes/configuration-reference/README.md)
