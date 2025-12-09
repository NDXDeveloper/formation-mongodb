🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.8 Personnalisation et Recommandations

## Introduction

Les systèmes de personnalisation et recommandations sont essentiels pour les plateformes modernes, nécessitant :

- **Profils utilisateurs riches** : Préférences, comportements, historique
- **Analyse d'interactions** : Vues, clics, achats, ratings
- **Algorithmes de recommandation** : Collaborative filtering, content-based, hybrid
- **Temps réel** : Recommandations adaptatives selon contexte
- **Scalabilité** : Millions d'utilisateurs et items
- **Diversité** : Éviter les bulles de filtrage
- **Cold start** : Recommandations pour nouveaux utilisateurs/items
- **A/B testing** : Optimisation continue
- **Privacy** : Respect RGPD et confidentialité

MongoDB excelle dans ce contexte grâce à :
- **Schéma flexible** : Features diverses pour ML
- **Embeddings natifs** : Représentations vectorielles pour similarité
- **Agrégations** : Calculs complexes de recommandations
- **Change Streams** : Mise à jour temps réel
- **Indexes** : Recherche rapide par similarité
- **Transactions** : Atomicité pour interactions

## Architecture de référence

### Stack de recommandations

```
┌─────────────────────────────────────────────────┐
│           User Interactions                     │
│  Views • Clicks • Purchases • Ratings • Shares  │
└────────────────────┬────────────────────────────┘
                     │
                     │ Events Stream
                     │
        ┌────────────▼────────────┐
        │   Event Processor       │
        │  (Kafka / Stream)       │
        └────────────┬────────────┘
                     │
     ┌───────────────┼───────────────────┐
     │               │                   │
┌────▼─────┐   ┌─────▼────┐   ┌──────────▼───┐
│  Real-   │   │  Batch   │   │   Feature    │
│  time    │   │  Jobs    │   │  Engineering │
│  Update  │   │          │   │              │
└────┬─────┘   └────┬─────┘   └─────┬────────┘
     │              │               │
     └──────────────┴───────────────┘
                     │
     ┌───────────────▼────────────┐
     │   MongoDB Cluster          │
     │  ┌──────────────────┐      │
     │  │  Users           │      │
     │  │  Profiles +      │      │
     │  │  Preferences     │      │
     │  └──────────────────┘      │
     │  ┌──────────────────┐      │
     │  │  Items           │      │
     │  │  Features +      │      │
     │  │  Embeddings      │      │
     │  └──────────────────┘      │
     │  ┌──────────────────┐      │
     │  │  Interactions    │      │
     │  │  Time Series     │      │
     │  └──────────────────┘      │
     │  ┌──────────────────┐      │
     │  │  Recommendations │      │
     │  │  Pre-computed    │      │
     │  └──────────────────┘      │
     └────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌────▼─────┐   ┌────▼──────┐
│Content- │    │Collabor- │   │  Hybrid   │
│ Based   │    │  ative   │   │  Engine   │
│ Engine  │    │ Filtering│   │           │
└────┬────┘    └────┬─────┘   └────┬──────┘
     │              │              │
     └──────────────┴──────────────┘
                    │
        ┌───────────▼─────────────┐
        │   Ranking & Filtering   │
        │  • Diversity            │
        │  • Business Rules       │
        │  • Context              │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────┐
        │    Personalization      │
        │         API             │
        └─────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌─────▼────┐   ┌──────▼────┐
│   Web   │    │  Mobile  │   │   Email   │
│  Pages  │    │   App    │   │ Campaign  │
└─────────┘    └──────────┘   └───────────┘
```

### Composants architecturaux

#### 1. Event Processor
**Technologies :** Kafka, Kinesis, RabbitMQ

**Justification :**
- Capture interactions en temps réel
- Buffer pour résilience
- Multiple consumers (analytics, ML, real-time)
- Replay capability

#### 2. Feature Engineering
**Technologies :** Spark, Pandas, scikit-learn

**Justification :**
- Extraction features des interactions
- Calcul embeddings (Word2Vec, BERT)
- Feature normalization
- Dimensionality reduction

#### 3. Recommendation Engines
**Types :**
- **Collaborative Filtering** : User-user, item-item similarity
- **Content-Based** : Similarité features items
- **Hybrid** : Combinaison pondérée

**Justification :**
- Collaborative : Patterns sociaux, "users like you"
- Content-Based : Cold start, explainabilité
- Hybrid : Meilleure performance globale

#### 4. MongoDB pour Recommendations
**Justification :**
- Schéma flexible pour features ML
- Embeddings pour calcul similarité
- Agrégations pour collaborative filtering
- Change Streams pour temps réel
- Pre-computation pour latence

## Modélisation des données

### 1. Profils utilisateurs enrichis

```javascript
// Collection: users
{
  _id: ObjectId("..."),
  userId: "user_abc123",

  // Informations de base
  profile: {
    email: "john.doe@example.com",
    name: "John Doe",
    avatar: "https://cdn.example.com/avatars/...",

    // Démographiques
    age: 32,
    gender: "M",
    location: {
      type: "Point",
      coordinates: [-73.935242, 40.730610]  // NYC
    },
    country: "US",
    city: "New York",
    timezone: "America/New_York",

    // Langue et locale
    language: "en",
    locale: "en_US"
  },

  // Préférences explicites
  preferences: {
    // Catégories préférées
    favoriteCategories: [
      { category: "electronics", weight: 0.9 },
      { category: "books", weight: 0.7 },
      { category: "sports", weight: 0.5 }
    ],

    // Marques préférées
    favoriteBrands: ["Apple", "Nike", "Sony"],

    // Prix range
    priceRange: {
      min: 50,
      max: 500,
      preferred: 200
    },

    // Préférences de notification
    notifications: {
      email: true,
      push: true,
      sms: false,
      frequency: "daily"  // realtime, daily, weekly
    },

    // Privacy settings
    privacy: {
      allowPersonalization: true,
      allowDataCollection: true,
      allowThirdParty: false
    }
  },

  // Comportement agrégé
  behavior: {
    // Activité
    totalViews: 1547,
    totalClicks: 892,
    totalPurchases: 45,
    totalSpent: 4567.89,

    // Fréquence
    avgSessionDuration: 1847,  // secondes
    visitFrequency: 4.5,  // par semaine
    lastActive: ISODate("2024-12-09T14:30:00Z"),

    // Engagement
    engagementScore: 0.78,  // 0-1
    churnRisk: 0.15,  // 0-1
    lifetimeValue: 5432.10,

    // Patterns temporels
    activeHours: [8, 9, 12, 13, 19, 20, 21],  // heures préférées
    activeDays: [1, 2, 3, 4, 5],  // lundi-vendredi

    // Diversité
    categoryDiversity: 0.65,  // Shannon entropy
    brandLoyalty: 0.42
  },

  // Feature vector pour ML (embedding utilisateur)
  embedding: {
    vector: [0.234, -0.156, 0.789, /* ... 128 dimensions */],
    version: "v2.1",
    computedAt: ISODate("2024-12-08T00:00:00Z")
  },

  // Segments
  segments: [
    {
      id: "high_value",
      name: "High Value Customer",
      score: 0.92,
      enteredAt: ISODate("2024-06-15T00:00:00Z")
    },
    {
      id: "tech_enthusiast",
      name: "Technology Enthusiast",
      score: 0.85,
      enteredAt: ISODate("2024-01-10T00:00:00Z")
    }
  ],

  // Données de session courante (cache)
  currentSession: {
    sessionId: "session_xyz789",
    startedAt: ISODate("2024-12-09T14:00:00Z"),

    // Context temps réel
    context: {
      device: "mobile",
      browser: "Chrome",
      referrer: "google.com",
      campaign: "winter_sale_2024"
    },

    // Intent détecté
    intent: {
      type: "purchase",  // browse, research, purchase
      confidence: 0.78,
      category: "electronics"
    },

    // Viewed items dans session
    recentViews: [
      { itemId: "item_001", viewedAt: ISODate("2024-12-09T14:15:00Z") },
      { itemId: "item_042", viewedAt: ISODate("2024-12-09T14:20:00Z") }
    ]
  },

  // Métadonnées
  createdAt: ISODate("2023-03-15T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z")
}

// Index pour users
db.users.createIndex({ userId: 1 }, { unique: true });
db.users.createIndex({ "profile.email": 1 });
db.users.createIndex({ "profile.location": "2dsphere" });
db.users.createIndex({ segments: 1 });
db.users.createIndex({
  "behavior.lastActive": -1,
  "behavior.engagementScore": -1
});

// Index pour recherche par embedding (approximative)
db.users.createIndex({ "embedding.vector": 1 });
```

### 2. Catalogue items avec features

```javascript
// Collection: items
{
  _id: ObjectId("..."),
  itemId: "item_abc123",

  // Informations de base
  name: "Wireless Noise-Cancelling Headphones",
  description: "Premium over-ear headphones with active noise cancellation...",

  // Catégorisation
  category: {
    primary: "electronics",
    secondary: "audio",
    tertiary: "headphones",

    // Taxonomy complète
    path: ["electronics", "audio", "headphones", "wireless"],
    level: 3
  },

  // Attributs produit
  attributes: {
    brand: "Sony",
    model: "WH-1000XM5",
    color: "Black",

    // Specs techniques
    specs: {
      battery_life: "30 hours",
      connectivity: ["Bluetooth 5.2", "USB-C", "3.5mm"],
      weight: "250g",
      drivers: "40mm"
    },

    // Features
    features: [
      "Active Noise Cancellation",
      "Ambient Sound Mode",
      "Touch Controls",
      "Multipoint Connection",
      "LDAC Support"
    ],

    // Tags pour recherche
    tags: [
      "wireless",
      "noise-cancelling",
      "bluetooth",
      "premium",
      "travel",
      "audiophile"
    ]
  },

  // Prix et disponibilité
  pricing: {
    currency: "USD",
    price: 399.99,
    originalPrice: 449.99,
    discount: 0.11,

    // Prix historique
    priceHistory: [
      { price: 449.99, date: ISODate("2024-01-01T00:00:00Z") },
      { price: 399.99, date: ISODate("2024-11-15T00:00:00Z") }
    ]
  },

  availability: {
    inStock: true,
    quantity: 42,
    warehouse: "US-EAST-1"
  },

  // Métriques de performance
  metrics: {
    // Popularité
    views: 15420,
    clicks: 4567,
    purchases: 892,

    // Conversion
    conversionRate: 0.195,  // purchases / clicks

    // Engagement
    avgTimeOnPage: 145,  // secondes
    bounceRate: 0.23,

    // Social
    shares: 234,
    wishlistAdds: 567,

    // Ratings
    avgRating: 4.7,
    ratingCount: 892,
    ratingDistribution: {
      5: 678,
      4: 156,
      3: 34,
      2: 12,
      1: 12
    },

    // Reviews
    reviewCount: 234,

    // Trending
    trendingScore: 0.87,  // 0-1
    velocity: 1.45  // growth rate
  },

  // Content pour recommendations content-based
  content: {
    // Text features pour NLP
    text: "Premium wireless noise-cancelling headphones with 30-hour battery...",

    // TF-IDF weights (pré-calculés)
    tfidf: {
      "noise": 0.89,
      "wireless": 0.76,
      "premium": 0.65,
      "battery": 0.54
    },

    // Catégories similaires
    similarCategories: ["earbuds", "speakers", "amplifiers"],

    // Items similaires (pre-computed)
    similarItems: [
      { itemId: "item_def456", score: 0.92 },
      { itemId: "item_ghi789", score: 0.88 },
      { itemId: "item_jkl012", score: 0.85 }
    ]
  },

  // Feature vector pour ML (embedding item)
  embedding: {
    vector: [0.456, 0.234, -0.123, /* ... 128 dimensions */],
    version: "v2.1",
    method: "item2vec",  // item2vec, bert, custom
    computedAt: ISODate("2024-12-08T00:00:00Z")
  },

  // Images
  images: [
    {
      url: "https://cdn.example.com/items/abc123/main.jpg",
      type: "main",

      // Visual features (pour visual search)
      visualFeatures: {
        dominantColors: ["#000000", "#C0C0C0"],
        brightness: 0.45,
        saturation: 0.23
      }
    }
  ],

  // Seasonal patterns
  seasonality: {
    peakMonths: [11, 12],  // Novembre, Décembre
    seasonal: true,
    trend: "increasing"
  },

  // Métadonnées
  createdAt: ISODate("2023-06-01T00:00:00Z"),
  updatedAt: ISODate("2024-12-09T10:00:00Z"),
  status: "active"
}

// Index pour items
db.items.createIndex({ itemId: 1 }, { unique: true });
db.items.createIndex({ "category.primary": 1, "category.secondary": 1 });
db.items.createIndex({ "attributes.brand": 1 });
db.items.createIndex({ "attributes.tags": 1 });
db.items.createIndex({ status: 1, "availability.inStock": 1 });

// Index pour popularité
db.items.createIndex({
  "metrics.trendingScore": -1,
  "metrics.avgRating": -1
});

// Index pour prix
db.items.createIndex({ "pricing.price": 1 });

// Text index pour recherche
db.items.createIndex({
  name: "text",
  description: "text",
  "attributes.tags": "text"
}, {
  weights: {
    name: 10,
    "attributes.tags": 5,
    description: 1
  }
});
```

### 3. Interactions utilisateur (Time Series)

```javascript
// Collection: interactions (Time Series)
db.createCollection("interactions", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  },
  expireAfterSeconds: 7776000  // 90 jours
});

// Structure d'interaction
{
  timestamp: ISODate("2024-12-09T14:30:15Z"),

  // Métadonnées
  metadata: {
    userId: "user_abc123",
    itemId: "item_def456",

    type: "purchase",  // view, click, add_to_cart, purchase, rating, wishlist

    // Context
    sessionId: "session_xyz789",
    device: "mobile",
    platform: "ios",

    // Source
    source: "homepage",  // homepage, search, recommendations, email
    campaign: "winter_sale_2024",

    // Position (si applicable)
    position: 3,  // position dans liste de recommandations
    list: "recommended_for_you"
  },

  // Données spécifiques
  data: {
    // Pour purchase
    price: 399.99,
    quantity: 1,
    discount: 0.11,

    // Pour rating
    rating: 5,
    review: "Excellent product, highly recommended!",

    // Pour view
    duration: 145,  // secondes passées
    scrollDepth: 0.78,

    // Pour click
    clickPosition: { x: 234, y: 567 }
  },

  // Feedback implicite
  implicit: {
    engagement: 0.85,  // calculé
    intent: "high",  // low, medium, high

    // Séquence (interaction précédente)
    previousAction: "view",
    timeSincePrevious: 120  // secondes
  }
}

// Index pour interactions
db.interactions.createIndex({
  "metadata.userId": 1,
  timestamp: -1
});

db.interactions.createIndex({
  "metadata.itemId": 1,
  timestamp: -1
});

db.interactions.createIndex({
  "metadata.type": 1,
  timestamp: -1
});

db.interactions.createIndex({
  "metadata.userId": 1,
  "metadata.type": 1,
  timestamp: -1
});
```

### 4. Recommandations pré-calculées

```javascript
// Collection: recommendations
{
  _id: ObjectId("..."),
  userId: "user_abc123",

  // Type de recommandation
  type: "personalized",  // personalized, trending, similar, category

  // Algorithme utilisé
  algorithm: "hybrid",  // collaborative, content_based, hybrid
  version: "v2.1",

  // Items recommandés
  items: [
    {
      itemId: "item_xyz789",
      score: 0.92,

      // Reasons pour explainabilité
      reasons: [
        {
          type: "purchased_similar",
          description: "Based on your purchase of Sony WH-1000XM4",
          weight: 0.5
        },
        {
          type: "trending",
          description: "Trending in your area",
          weight: 0.3
        },
        {
          type: "user_preference",
          description: "Matches your preference for electronics",
          weight: 0.2
        }
      ],

      // Composants du score
      scoreBreakdown: {
        collaborative: 0.85,
        contentBased: 0.78,
        popularity: 0.92,
        recency: 0.67,

        // Poids appliqués
        weights: {
          collaborative: 0.5,
          contentBased: 0.3,
          popularity: 0.1,
          recency: 0.1
        }
      },

      // Metadata pour ranking
      metadata: {
        category: "electronics",
        price: 349.99,
        inStock: true,
        avgRating: 4.6
      }
    },
    // ... top 50 items
  ],

  // Context de génération
  context: {
    // Filtres appliqués
    filters: {
      minRating: 4.0,
      maxPrice: 1000,
      inStock: true,
      excludeViewed: true  // dernières 24h
    },

    // Diversité
    diversity: {
      categorySpread: 0.75,
      brandSpread: 0.68,
      priceRangeSpread: 0.82
    },

    // Business rules
    businessRules: [
      "boost_margin_>30%",
      "boost_new_arrivals",
      "exclude_discontinued"
    ]
  },

  // Métriques de qualité
  quality: {
    confidence: 0.87,
    coverage: 0.92,  // % catalog couvert
    novelty: 0.65,  // items non évidents
    serendipity: 0.34  // surprenantes mais pertinentes
  },

  // Performance tracking
  performance: {
    impressions: 15,
    clicks: 3,
    conversions: 1,

    ctr: 0.20,  // click-through rate
    conversionRate: 0.067,

    // A/B test
    variant: "A",
    experimentId: "exp_winter_2024"
  },

  // Metadata
  computedAt: ISODate("2024-12-09T14:00:00Z"),
  expiresAt: ISODate("2024-12-09T20:00:00Z"),  // Refresh every 6h

  // Status
  status: "active"  // active, expired, testing
}

// Index pour recommendations
db.recommendations.createIndex({
  userId: 1,
  type: 1,
  status: 1
});

db.recommendations.createIndex({ expiresAt: 1 });

// TTL pour cleanup automatique
db.recommendations.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);
```

### 5. Matrice de similarité (pour collaborative filtering)

```javascript
// Collection: user_similarity
{
  _id: ObjectId("..."),
  userId: "user_abc123",

  // Top N utilisateurs similaires
  similarUsers: [
    {
      userId: "user_def456",
      similarity: 0.87,  // cosine similarity

      // Features communes
      commonFeatures: {
        categories: ["electronics", "books"],
        brands: ["Apple", "Sony"],
        priceRange: "mid-high"
      },

      // Items qu'ils ont aimé et pas nous
      recommendations: [
        { itemId: "item_ghi789", theirRating: 5 },
        { itemId: "item_jkl012", theirRating: 4 }
      ]
    },
    // ... top 50 similar users
  ],

  computedAt: ISODate("2024-12-09T00:00:00Z"),
  version: "v2.0"
}

// Index
db.user_similarity.createIndex({ userId: 1 }, { unique: true });

// Collection: item_similarity
{
  _id: ObjectId("..."),
  itemId: "item_abc123",

  // Items similaires
  similarItems: [
    {
      itemId: "item_def456",
      similarity: 0.92,

      method: "content_based",  // content_based, collaborative

      // Features communes
      sharedFeatures: {
        category: true,
        brand: false,
        priceRange: true,
        tags: ["wireless", "premium"]
      }
    },
    // ... top 100 similar items
  ],

  computedAt: ISODate("2024-12-09T00:00:00Z")
}

// Index
db.item_similarity.createIndex({ itemId: 1 }, { unique: true });
```

## Services de recommandation

### 1. Collaborative Filtering Engine

```javascript
class CollaborativeFilteringEngine {
  constructor(db) {
    this.db = db;
  }

  // User-based collaborative filtering
  async getUserBasedRecommendations(userId, limit = 20) {
    // 1. Trouver utilisateurs similaires
    const similarUsers = await this.db.collection('user_similarity')
      .findOne({ userId });

    if (!similarUsers || similarUsers.similarUsers.length === 0) {
      return this.getFallbackRecommendations(userId, limit);
    }

    // 2. Agréger items qu'ils ont aimé
    const similarUserIds = similarUsers.similarUsers
      .slice(0, 20)  // Top 20 similar users
      .map(u => u.userId);

    const pipeline = [
      {
        $match: {
          'metadata.userId': { $in: similarUserIds },
          'metadata.type': { $in: ['purchase', 'rating'] }
        }
      },

      // Filtrer ratings >= 4 et purchases
      {
        $match: {
          $or: [
            { 'metadata.type': 'purchase' },
            {
              'metadata.type': 'rating',
              'data.rating': { $gte: 4 }
            }
          ]
        }
      },

      // Grouper par item
      {
        $group: {
          _id: '$metadata.itemId',

          // Compter occurrences
          count: { $sum: 1 },

          // Somme des similarités des users qui l'ont aimé
          totalSimilarity: {
            $sum: {
              $let: {
                vars: {
                  user: {
                    $arrayElemAt: [
                      {
                        $filter: {
                          input: similarUsers.similarUsers,
                          cond: { $eq: ['$$this.userId', '$metadata.userId'] }
                        }
                      },
                      0
                    ]
                  }
                },
                in: { $ifNull: ['$$user.similarity', 0] }
              }
            }
          },

          // Moyenne rating si disponible
          avgRating: {
            $avg: {
              $cond: [
                { $eq: ['$metadata.type', 'rating'] },
                '$data.rating',
                5  // Default pour purchases
              ]
            }
          },

          users: { $addToSet: '$metadata.userId' }
        }
      },

      // Calculer score final
      {
        $addFields: {
          score: {
            $multiply: [
              { $divide: ['$totalSimilarity', '$count'] },  // Avg similarity
              { $divide: ['$avgRating', 5] },  // Normalized rating
              { $add: [1, { $multiply: ['$count', 0.1] }] }  // Popularity boost
            ]
          }
        }
      },

      // Exclure items déjà vus/achetés par l'utilisateur
      {
        $lookup: {
          from: 'interactions',
          let: { itemId: '$_id' },
          pipeline: [
            {
              $match: {
                $expr: {
                  $and: [
                    { $eq: ['$metadata.userId', userId] },
                    { $eq: ['$metadata.itemId', '$$itemId'] },
                    {
                      $in: [
                        '$metadata.type',
                        ['purchase', 'view']
                      ]
                    },
                    {
                      $gte: [
                        '$timestamp',
                        new Date(Date.now() - 7 * 24 * 3600000)  // 7 jours
                      ]
                    }
                  ]
                }
              }
            }
          ],
          as: 'userInteractions'
        }
      },

      {
        $match: {
          userInteractions: { $size: 0 }
        }
      },

      // Sort par score
      {
        $sort: { score: -1 }
      },

      {
        $limit: limit
      },

      // Lookup item details
      {
        $lookup: {
          from: 'items',
          localField: '_id',
          foreignField: 'itemId',
          as: 'item'
        }
      },

      {
        $unwind: '$item'
      },

      // Format final
      {
        $project: {
          itemId: '$_id',
          score: 1,

          reasons: {
            $map: {
              input: { $slice: ['$users', 3] },
              as: 'userId',
              in: {
                type: 'similar_user',
                userId: '$$userId',
                description: 'Liked by users similar to you'
              }
            }
          },

          metadata: {
            name: '$item.name',
            category: '$item.category.primary',
            price: '$item.pricing.price',
            avgRating: '$item.metrics.avgRating',
            inStock: '$item.availability.inStock'
          }
        }
      }
    ];

    const recommendations = await this.db.collection('interactions')
      .aggregate(pipeline)
      .toArray();

    return recommendations;
  }

  // Item-based collaborative filtering
  async getItemBasedRecommendations(itemId, userId, limit = 20) {
    // Récupérer items similaires
    const similarItems = await this.db.collection('item_similarity')
      .findOne({ itemId });

    if (!similarItems) {
      return [];
    }

    // Filtrer items déjà vus
    const userInteractions = await this.db.collection('interactions')
      .find({
        'metadata.userId': userId,
        'metadata.type': { $in: ['purchase', 'view'] },
        timestamp: { $gte: new Date(Date.now() - 7 * 24 * 3600000) }
      })
      .project({ 'metadata.itemId': 1 })
      .toArray();

    const viewedItems = new Set(userInteractions.map(i => i.metadata.itemId));

    // Filtrer et enrichir
    const recommendations = [];

    for (const similar of similarItems.similarItems.slice(0, limit * 2)) {
      if (viewedItems.has(similar.itemId)) continue;

      // Lookup item
      const item = await this.db.collection('items')
        .findOne({
          itemId: similar.itemId,
          status: 'active',
          'availability.inStock': true
        });

      if (item) {
        recommendations.push({
          itemId: similar.itemId,
          score: similar.similarity,

          reasons: [
            {
              type: 'similar_item',
              description: `Similar to ${itemId}`,
              weight: 1.0
            }
          ],

          metadata: {
            name: item.name,
            category: item.category.primary,
            price: item.pricing.price,
            avgRating: item.metrics.avgRating
          }
        });
      }

      if (recommendations.length >= limit) break;
    }

    return recommendations;
  }

  async getFallbackRecommendations(userId, limit) {
    // Pour cold start: recommander trending items
    const trending = await this.db.collection('items')
      .find({
        status: 'active',
        'availability.inStock': true
      })
      .sort({ 'metrics.trendingScore': -1 })
      .limit(limit)
      .toArray();

    return trending.map(item => ({
      itemId: item.itemId,
      score: item.metrics.trendingScore,

      reasons: [
        {
          type: 'trending',
          description: 'Trending now',
          weight: 1.0
        }
      ],

      metadata: {
        name: item.name,
        category: item.category.primary,
        price: item.pricing.price,
        avgRating: item.metrics.avgRating
      }
    }));
  }
}
```

### 2. Content-Based Filtering Engine

```javascript
class ContentBasedFilteringEngine {
  constructor(db) {
    this.db = db;
  }

  async getContentBasedRecommendations(userId, limit = 20) {
    // 1. Récupérer profil utilisateur
    const user = await this.db.collection('users')
      .findOne({ userId });

    if (!user || !user.embedding) {
      return [];
    }

    // 2. Rechercher items similaires par embedding
    const pipeline = [
      {
        $match: {
          status: 'active',
          'availability.inStock': true
        }
      },

      // Calculer similarité cosine avec user embedding
      {
        $addFields: {
          similarity: {
            $let: {
              vars: {
                dotProduct: {
                  $reduce: {
                    input: { $range: [0, { $size: user.embedding.vector }] },
                    initialValue: 0,
                    in: {
                      $add: [
                        '$$value',
                        {
                          $multiply: [
                            { $arrayElemAt: [user.embedding.vector, '$$this'] },
                            { $arrayElemAt: ['$embedding.vector', '$$this'] }
                          ]
                        }
                      ]
                    }
                  }
                },

                userMagnitude: {
                  $sqrt: {
                    $reduce: {
                      input: user.embedding.vector,
                      initialValue: 0,
                      in: { $add: ['$$value', { $pow: ['$$this', 2] }] }
                    }
                  }
                },

                itemMagnitude: {
                  $sqrt: {
                    $reduce: {
                      input: '$embedding.vector',
                      initialValue: 0,
                      in: { $add: ['$$value', { $pow: ['$$this', 2] }] }
                    }
                  }
                }
              },
              in: {
                $divide: [
                  '$$dotProduct',
                  { $multiply: ['$$userMagnitude', '$$itemMagnitude'] }
                ]
              }
            }
          }
        }
      },

      // Boost si catégorie préférée
      {
        $addFields: {
          categoryBoost: {
            $cond: [
              {
                $in: [
                  '$category.primary',
                  user.preferences.favoriteCategories.map(c => c.category)
                ]
              },
              1.2,
              1.0
            ]
          },

          brandBoost: {
            $cond: [
              { $in: ['$attributes.brand', user.preferences.favoriteBrands] },
              1.15,
              1.0
            ]
          }
        }
      },

      {
        $addFields: {
          finalScore: {
            $multiply: ['$similarity', '$categoryBoost', '$brandBoost']
          }
        }
      },

      // Exclure items récemment vus
      {
        $lookup: {
          from: 'interactions',
          let: { itemId: '$itemId' },
          pipeline: [
            {
              $match: {
                $expr: {
                  $and: [
                    { $eq: ['$metadata.userId', userId] },
                    { $eq: ['$metadata.itemId', '$$itemId'] },
                    {
                      $gte: [
                        '$timestamp',
                        new Date(Date.now() - 7 * 24 * 3600000)
                      ]
                    }
                  ]
                }
              }
            }
          ],
          as: 'recentInteractions'
        }
      },

      {
        $match: {
          recentInteractions: { $size: 0 }
        }
      },

      {
        $sort: { finalScore: -1 }
      },

      {
        $limit: limit
      },

      {
        $project: {
          itemId: 1,
          score: '$finalScore',
          similarity: 1,

          reasons: [
            {
              type: 'content_match',
              description: 'Matches your interests',
              weight: 1.0
            }
          ],

          metadata: {
            name: 1,
            category: '$category.primary',
            price: '$pricing.price',
            avgRating: '$metrics.avgRating'
          }
        }
      }
    ];

    return this.db.collection('items')
      .aggregate(pipeline)
      .toArray();
  }

  async getContextualRecommendations(userId, context, limit = 20) {
    // Recommandations basées sur contexte (device, location, time, etc.)

    const user = await this.db.collection('users')
      .findOne({ userId });

    const pipeline = [
      {
        $match: {
          status: 'active',
          'availability.inStock': true
        }
      },

      // Score contextuel
      {
        $addFields: {
          contextScore: {
            $multiply: [
              // Device appropriateness
              {
                $cond: [
                  {
                    $and: [
                      { $eq: [context.device, 'mobile'] },
                      { $in: ['mobile', '$attributes.tags'] }
                    ]
                  },
                  1.2,
                  1.0
                ]
              },

              // Time appropriateness (example: electronics better in evening)
              {
                $cond: [
                  {
                    $and: [
                      { $eq: ['$category.primary', 'electronics'] },
                      { $gte: [new Date().getHours(), 18] }
                    ]
                  },
                  1.1,
                  1.0
                ]
              },

              // Price appropriateness
              {
                $cond: [
                  {
                    $and: [
                      { $gte: ['$pricing.price', user.preferences.priceRange.min] },
                      { $lte: ['$pricing.price', user.preferences.priceRange.max] }
                    ]
                  },
                  1.0,
                  0.8
                ]
              }
            ]
          }
        }
      },

      {
        $sort: { contextScore: -1 }
      },

      {
        $limit: limit
      }
    ];

    return this.db.collection('items')
      .aggregate(pipeline)
      .toArray();
  }
}
```

### 3. Hybrid Recommendation Engine

```javascript
class HybridRecommendationEngine {
  constructor(db, collaborative, contentBased) {
    this.db = db;
    this.collaborative = collaborative;
    this.contentBased = contentBased;
  }

  async generateRecommendations(userId, options = {}) {
    const {
      limit = 20,
      context = {},
      algorithm = 'auto',  // auto, collaborative, content_based, hybrid
      diversify = true,
      explain = true
    } = options;

    // Déterminer algorithme optimal
    const strategy = await this.selectStrategy(userId, algorithm);

    let recommendations = [];

    if (strategy === 'hybrid' || strategy === 'auto') {
      // Combiner collaborative et content-based
      const [collaborative, contentBased] = await Promise.all([
        this.collaborative.getUserBasedRecommendations(userId, limit * 2),
        this.contentBased.getContentBasedRecommendations(userId, limit * 2)
      ]);

      // Merge avec pondération
      recommendations = this.mergeRecommendations(
        collaborative,
        contentBased,
        {
          collaborativeWeight: 0.6,
          contentWeight: 0.4
        }
      );

    } else if (strategy === 'collaborative') {
      recommendations = await this.collaborative
        .getUserBasedRecommendations(userId, limit * 2);

    } else if (strategy === 'content_based') {
      recommendations = await this.contentBased
        .getContentBasedRecommendations(userId, limit * 2);
    }

    // Appliquer business rules
    recommendations = await this.applyBusinessRules(
      recommendations,
      userId
    );

    // Diversifier
    if (diversify) {
      recommendations = this.diversifyRecommendations(
        recommendations,
        limit
      );
    } else {
      recommendations = recommendations.slice(0, limit);
    }

    // Enrichir avec explications
    if (explain) {
      recommendations = await this.enrichWithExplanations(
        recommendations,
        userId
      );
    }

    // Sauvegarder pour caching
    await this.saveRecommendations(userId, recommendations, {
      algorithm: strategy,
      context
    });

    return recommendations;
  }

  async selectStrategy(userId, requestedAlgorithm) {
    if (requestedAlgorithm !== 'auto') {
      return requestedAlgorithm;
    }

    // Déterminer meilleur algorithme basé sur données utilisateur
    const user = await this.db.collection('users')
      .findOne({ userId });

    if (!user) {
      return 'trending';  // Cold start
    }

    const interactionCount = user.behavior.totalPurchases +
                            user.behavior.totalClicks;

    // Cold start: pas assez d'interactions
    if (interactionCount < 10) {
      return 'content_based';  // Content-based meilleur pour cold start
    }

    // Warm: assez d'interactions
    if (interactionCount < 50) {
      return 'hybrid';  // Combinaison
    }

    // Hot: beaucoup d'interactions
    return 'collaborative';  // Collaborative meilleur avec beaucoup de données
  }

  mergeRecommendations(collaborative, contentBased, weights) {
    // Créer map par itemId
    const scoreMap = new Map();

    // Ajouter collaborative
    for (const rec of collaborative) {
      scoreMap.set(rec.itemId, {
        ...rec,
        collaborativeScore: rec.score,
        contentScore: 0,
        finalScore: rec.score * weights.collaborativeWeight
      });
    }

    // Ajouter/merger content-based
    for (const rec of contentBased) {
      if (scoreMap.has(rec.itemId)) {
        const existing = scoreMap.get(rec.itemId);
        existing.contentScore = rec.score;
        existing.finalScore += rec.score * weights.contentWeight;

        // Merger reasons
        existing.reasons = [
          ...existing.reasons,
          ...rec.reasons
        ];
      } else {
        scoreMap.set(rec.itemId, {
          ...rec,
          collaborativeScore: 0,
          contentScore: rec.score,
          finalScore: rec.score * weights.contentWeight
        });
      }
    }

    // Convertir en array et trier
    return Array.from(scoreMap.values())
      .sort((a, b) => b.finalScore - a.finalScore);
  }

  async applyBusinessRules(recommendations, userId) {
    // Appliquer règles business

    const user = await this.db.collection('users')
      .findOne({ userId });

    return recommendations.map(rec => {
      let boost = 1.0;

      // Boost items haute marge
      if (rec.metadata.margin && rec.metadata.margin > 0.3) {
        boost *= 1.1;
      }

      // Boost nouveautés
      const itemAge = Date.now() - new Date(rec.metadata.createdAt).getTime();
      if (itemAge < 30 * 24 * 3600000) {  // 30 jours
        boost *= 1.05;
      }

      // Penalty items trop chers pour utilisateur
      if (user && rec.metadata.price > user.preferences.priceRange.max) {
        boost *= 0.7;
      }

      return {
        ...rec,
        score: rec.score * boost
      };
    }).sort((a, b) => b.score - a.score);
  }

  diversifyRecommendations(recommendations, limit) {
    // Diversifier par catégorie et prix

    const selected = [];
    const categories = new Set();
    const priceRanges = new Set();

    // Définir price ranges
    const getPriceRange = (price) => {
      if (price < 50) return 'budget';
      if (price < 200) return 'mid';
      if (price < 500) return 'high';
      return 'premium';
    };

    // Sélection avec diversité
    for (const rec of recommendations) {
      if (selected.length >= limit) break;

      const category = rec.metadata.category;
      const priceRange = getPriceRange(rec.metadata.price);

      // Limiter items par catégorie (max 40% d'une catégorie)
      const categoryCount = selected.filter(
        r => r.metadata.category === category
      ).length;

      if (categoryCount >= Math.ceil(limit * 0.4)) {
        continue;
      }

      selected.push(rec);
      categories.add(category);
      priceRanges.add(priceRange);
    }

    // Remplir si pas assez
    if (selected.length < limit) {
      const remaining = recommendations.filter(
        r => !selected.find(s => s.itemId === r.itemId)
      );

      selected.push(...remaining.slice(0, limit - selected.length));
    }

    return selected;
  }

  async enrichWithExplanations(recommendations, userId) {
    // Enrichir avec explications détaillées

    const user = await this.db.collection('users')
      .findOne({ userId });

    return Promise.all(recommendations.map(async rec => {
      const explanations = [];

      // Analyser reasons
      for (const reason of rec.reasons) {
        switch (reason.type) {
          case 'similar_user':
            explanations.push({
              text: 'People with similar tastes loved this',
              icon: 'users',
              confidence: 'high'
            });
            break;

          case 'content_match':
            explanations.push({
              text: `Matches your interest in ${rec.metadata.category}`,
              icon: 'heart',
              confidence: 'medium'
            });
            break;

          case 'trending':
            explanations.push({
              text: 'Trending in your area',
              icon: 'trending-up',
              confidence: 'medium'
            });
            break;
        }
      }

      return {
        ...rec,
        explanations
      };
    }));
  }

  async saveRecommendations(userId, recommendations, metadata) {
    // Sauvegarder pour caching

    await this.db.collection('recommendations').updateOne(
      {
        userId,
        type: 'personalized',
        status: 'active'
      },
      {
        $set: {
          userId,
          type: 'personalized',
          algorithm: metadata.algorithm,

          items: recommendations.map(r => ({
            itemId: r.itemId,
            score: r.score,
            reasons: r.reasons,
            metadata: r.metadata
          })),

          context: metadata.context,

          computedAt: new Date(),
          expiresAt: new Date(Date.now() + 6 * 3600000),  // 6h
          status: 'active'
        }
      },
      { upsert: true }
    );
  }
}
```

### 4. Real-Time Personalization Service

```javascript
class RealTimePersonalizationService {
  constructor(db) {
    this.db = db;
    this.sessionCache = new Map();
  }

  async start() {
    // Watch sur interactions pour updates temps réel
    const changeStream = this.db.collection('interactions').watch();

    changeStream.on('change', async (change) => {
      if (change.operationType === 'insert') {
        await this.handleInteraction(change.fullDocument);
      }
    });

    console.log('Real-time personalization service started');
  }

  async handleInteraction(interaction) {
    const { userId, itemId, type } = interaction.metadata;

    // Update user profile en temps réel
    const updates = {};

    switch (type) {
      case 'view':
        updates.$inc = {
          'behavior.totalViews': 1
        };
        updates.$set = {
          'behavior.lastActive': new Date()
        };
        break;

      case 'click':
        updates.$inc = {
          'behavior.totalClicks': 1
        };
        break;

      case 'purchase':
        updates.$inc = {
          'behavior.totalPurchases': 1,
          'behavior.totalSpent': interaction.data.price
        };
        break;
    }

    if (Object.keys(updates).length > 0) {
      await this.db.collection('users').updateOne(
        { userId },
        updates
      );
    }

    // Update session cache
    this.updateSessionCache(userId, interaction);

    // Invalider recommendations cache si purchase
    if (type === 'purchase') {
      await this.invalidateRecommendations(userId);
    }
  }

  updateSessionCache(userId, interaction) {
    if (!this.sessionCache.has(userId)) {
      this.sessionCache.set(userId, {
        recentViews: [],
        recentSearches: [],
        intent: null
      });
    }

    const cache = this.sessionCache.get(userId);

    if (interaction.metadata.type === 'view') {
      cache.recentViews.push({
        itemId: interaction.metadata.itemId,
        timestamp: interaction.timestamp
      });

      // Garder seulement 10 dernières vues
      if (cache.recentViews.length > 10) {
        cache.recentViews.shift();
      }

      // Détecter intent
      cache.intent = this.detectIntent(cache.recentViews);
    }
  }

  detectIntent(recentViews) {
    if (recentViews.length < 3) return null;

    // Analyser séquence de vues
    const categories = recentViews.map(v => v.category);
    const uniqueCategories = new Set(categories);

    // Si focus sur une catégorie: purchase intent
    if (uniqueCategories.size === 1) {
      return {
        type: 'purchase',
        category: categories[0],
        confidence: 0.8
      };
    }

    // Si diverse: browsing
    return {
      type: 'browse',
      confidence: 0.6
    };
  }

  async invalidateRecommendations(userId) {
    await this.db.collection('recommendations').updateMany(
      { userId, status: 'active' },
      { $set: { status: 'expired' } }
    );
  }

  async getSessionRecommendations(userId, sessionId) {
    // Recommandations basées sur session en cours

    const cache = this.sessionCache.get(userId);
    if (!cache || cache.recentViews.length === 0) {
      return [];
    }

    // Derniers items vus
    const recentItemIds = cache.recentViews
      .slice(-3)
      .map(v => v.itemId);

    // Items similaires
    const similarItems = await this.db.collection('item_similarity')
      .find({
        itemId: { $in: recentItemIds }
      })
      .toArray();

    // Agréger scores
    const scoreMap = new Map();

    for (const doc of similarItems) {
      for (const similar of doc.similarItems) {
        const current = scoreMap.get(similar.itemId) || 0;
        scoreMap.set(similar.itemId, current + similar.similarity);
      }
    }

    // Convertir et trier
    const recommendations = Array.from(scoreMap.entries())
      .map(([itemId, score]) => ({ itemId, score }))
      .sort((a, b) => b.score - a.score)
      .slice(0, 10);

    return recommendations;
  }
}
```

## Performance et optimisation

### 1. Batch computation des similarités

```javascript
class SimilarityBatchJob {
  async computeUserSimilarities() {
    console.log('Computing user similarities...');

    // 1. Récupérer tous les utilisateurs avec assez d'interactions
    const users = await this.db.collection('users')
      .find({
        'behavior.totalPurchases': { $gte: 5 }
      })
      .project({ userId: 1, embedding: 1 })
      .toArray();

    console.log(`Processing ${users.length} users`);

    // 2. Calculer similarités par batch
    const batchSize = 100;

    for (let i = 0; i < users.length; i += batchSize) {
      const batch = users.slice(i, i + batchSize);

      await Promise.all(batch.map(user =>
        this.computeSimilarityForUser(user, users)
      ));

      console.log(`Processed ${Math.min(i + batchSize, users.length)} / ${users.length}`);
    }

    console.log('User similarities computed');
  }

  async computeSimilarityForUser(user, allUsers) {
    const similarities = [];

    for (const other of allUsers) {
      if (user.userId === other.userId) continue;

      // Cosine similarity
      const similarity = this.cosineSimilarity(
        user.embedding.vector,
        other.embedding.vector
      );

      if (similarity > 0.5) {  // Seuil
        similarities.push({
          userId: other.userId,
          similarity
        });
      }
    }

    // Garder top 50
    similarities.sort((a, b) => b.similarity - a.similarity);
    const topSimilar = similarities.slice(0, 50);

    // Sauvegarder
    await this.db.collection('user_similarity').updateOne(
      { userId: user.userId },
      {
        $set: {
          userId: user.userId,
          similarUsers: topSimilar,
          computedAt: new Date(),
          version: 'v2.0'
        }
      },
      { upsert: true }
    );
  }

  cosineSimilarity(vectorA, vectorB) {
    let dotProduct = 0;
    let magnitudeA = 0;
    let magnitudeB = 0;

    for (let i = 0; i < vectorA.length; i++) {
      dotProduct += vectorA[i] * vectorB[i];
      magnitudeA += vectorA[i] * vectorA[i];
      magnitudeB += vectorB[i] * vectorB[i];
    }

    magnitudeA = Math.sqrt(magnitudeA);
    magnitudeB = Math.sqrt(magnitudeB);

    if (magnitudeA === 0 || magnitudeB === 0) return 0;

    return dotProduct / (magnitudeA * magnitudeB);
  }
}
```

### 2. Configuration MongoDB optimale

```javascript
// Index pour performance
const recommendationIndexes = {
  // Users
  users: [
    { userId: 1 },
    { "embedding.vector": 1 },
    { "behavior.lastActive": -1, "behavior.engagementScore": -1 },
    { segments: 1 }
  ],

  // Items
  items: [
    { itemId: 1 },
    { "category.primary": 1, "metrics.trendingScore": -1 },
    { "embedding.vector": 1 },
    { status: 1, "availability.inStock": 1 }
  ],

  // Interactions (Time Series optimized)
  interactions: [
    { "metadata.userId": 1, timestamp: -1 },
    { "metadata.itemId": 1, timestamp: -1 },
    { "metadata.type": 1, timestamp: -1 }
  ],

  // Recommendations
  recommendations: [
    { userId: 1, type: 1, status: 1 },
    { expiresAt: 1 }  // TTL
  ],

  // Similarity
  user_similarity: [
    { userId: 1 }
  ],
  item_similarity: [
    { itemId: 1 }
  ]
};

// Configuration sharding pour scale
sh.enableSharding('recommendations');

// Shard users par userId
sh.shardCollection(
  'recommendations.users',
  { userId: 'hashed' }
);

// Shard interactions par userId + timestamp
sh.shardCollection(
  'recommendations.interactions',
  { 'metadata.userId': 'hashed', timestamp: 1 }
);
```

### 3. Métriques de performance

```javascript
const recommendationMetrics = {
  // Latence
  'recommendations.compute_latency.p95': {
    description: 'Time to compute recommendations',
    target: 500,  // ms
    alert_threshold: 2000
  },

  'recommendations.api_latency.p95': {
    description: 'API response time',
    target: 100,  // ms
    alert_threshold: 500
  },

  // Qualité
  'recommendations.click_through_rate': {
    description: 'CTR on recommendations',
    target: 0.15,
    alert_threshold: 0.05
  },

  'recommendations.conversion_rate': {
    description: 'Conversion rate on recommendations',
    target: 0.08,
    alert_threshold: 0.02
  },

  // Coverage
  'recommendations.user_coverage': {
    description: '% users with recommendations',
    target: 0.95,
    alert_threshold: 0.80
  },

  'recommendations.catalog_coverage': {
    description: '% catalog in recommendations',
    target: 0.70,
    alert_threshold: 0.40
  }
};
```

## Checklist de déploiement

### ✅ Modélisation

- [ ] User profiles avec embeddings
- [ ] Items avec features et embeddings
- [ ] Interactions Time Series
- [ ] Similarity matrices pré-calculées
- [ ] Recommendations cache

### ✅ Algorithmes

- [ ] Collaborative filtering (user-based et item-based)
- [ ] Content-based filtering
- [ ] Hybrid engine
- [ ] Cold start strategy
- [ ] Diversification

### ✅ Real-Time

- [ ] Change Streams pour updates
- [ ] Session tracking
- [ ] Intent detection
- [ ] Recommendation invalidation
- [ ] Cache strategy

### ✅ Qualité

- [ ] A/B testing framework
- [ ] Metrics tracking (CTR, conversion)
- [ ] Explainability
- [ ] Diversity enforcement
- [ ] Business rules

### ✅ Performance

- [ ] Index optimisés
- [ ] Pre-computation jobs
- [ ] Caching strategy
- [ ] Sharding si nécessaire
- [ ] Batch processing

### ✅ Privacy

- [ ] RGPD compliance
- [ ] User consent
- [ ] Data minimization
- [ ] Right to deletion
- [ ] Transparency

### ✅ Operations

- [ ] Monitoring metrics
- [ ] Alerting sur qualité
- [ ] Batch jobs schedulés
- [ ] Model versioning
- [ ] Documentation

## Conclusion

MongoDB est excellent pour les systèmes de recommandations grâce à :

**✅ Forces démontrées :**
- Schéma flexible pour features ML diverses
- Embeddings natifs pour similarité
- Agrégations puissantes pour collaborative filtering
- Change Streams pour temps réel
- Time Series pour interactions
- Performance pour millions d'utilisateurs/items

**⚠️ Considérations importantes :**
- Pre-computation essentielle pour latence
- Cold start nécessite fallback strategies
- Balance freshness vs computation cost
- Privacy et RGPD critiques
- A/B testing continu requis

**🎯 Patterns essentiels recommandations :**
1. **Embeddings** pour représentations vectorielles
2. **Hybrid approach** pour meilleure qualité
3. **Pre-computation** pour latence
4. **Change Streams** pour temps réel
5. **Diversification** pour éviter bulles
6. **Explainability** pour confiance utilisateur
7. **A/B testing** pour optimisation continue

Cette architecture supporte des plateformes de millions d'utilisateurs avec recommandations temps réel et personnalisation contextuelle avancée.

---

**Références :**
- "Recommender Systems Handbook" - Ricci et al.
- Netflix Prize Competition
- Amazon Recommendation System
- "Deep Learning for Recommender Systems" - Google
- Matrix Factorization Techniques

⏭️ [Applications financières](/20-cas-usage-architectures/09-applications-financieres.md)
