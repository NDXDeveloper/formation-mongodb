🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.3 Catalogue Produits (E-commerce)

## Introduction

Les catalogues de produits e-commerce représentent un cas d'usage particulièrement complexe pour les bases de données, combinant :

- **Données hétérogènes** : Produits avec attributs variables selon les catégories
- **Hiérarchies multiples** : Catégories, marques, collections
- **Variations complexes** : Tailles, couleurs, options avec gestion de stock
- **Recherche avancée** : Filtres facettés, recherche full-text, tri multi-critères
- **Performance critique** : Temps de réponse < 100ms attendu
- **Scalabilité élevée** : De milliers à millions de produits
- **Personnalisation** : Prix dynamiques, recommandations, promotions
- **Multi-canal** : Web, mobile, marketplace, points de vente

MongoDB excelle dans ce contexte grâce à sa flexibilité de schéma (attributs variables), ses capacités d'agrégation (facettes), et sa performance en lecture.

## Architecture de référence

### Stack e-commerce moderne

```
┌─────────────────────────────────────────────────────┐
│              CDN + Edge Computing                   │
│        (Product Images, Static Assets)              │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────┴────────────┐
        │                         │
┌───────▼────────┐       ┌────────▼─────────┐
│  Storefront    │       │  Admin Panel     │
│  (Next.js)     │       │  (React)         │
└───────┬────────┘       └────────┬─────────┘
        │                         │
        └────────┬────────────────┘
                 │
        ┌────────▼────────┐
        │   API Gateway   │
        │  (Kong/Nginx)   │
        └────────┬────────┘
                 │
     ┌───────────┼─────────────────┐
     │           │                 │
┌────▼─────┐ ┌───▼───────┐ ┌───────▼────────┐
│ Catalog  │ │  Cart     │ │   Order        │
│ Service  │ │  Service  │ │   Service      │
└────┬─────┘ └──┬────────┘ └───────┬────────┘
     │          │                  │
     │    ┌─────┴─────┐            │
     │    │           │            │
┌────▼────▼──┐   ┌────▼────┐  ┌────▼──────┐
│   Redis    │   │RabbitMQ │  │PostgreSQL │
│  (Cache)   │   │(Events) │  │ (Orders)  │
└────────────┘   └─────────┘  └───────────┘
     │
     │    ┌──────────────┐
     └────┤Elasticsearch │
          │  (Search)    │
          └──────┬───────┘
                 │
     ┌───────────▼─────────────┐
     │   MongoDB Atlas         │
     │   Sharded Cluster       │
     │  ┌─────────────────┐    │
     │  │  Shard 1 (RS)   │    │
     │  │  Shard 2 (RS)   │    │
     │  │  Shard 3 (RS)   │    │
     │  └─────────────────┘    │
     │  ┌─────────────────┐    │
     │  │  Config Servers │    │
     │  └─────────────────┘    │
     └─────────────────────────┘
```

### Justifications architecturales

#### 1. MongoDB pour le catalogue
**Pourquoi MongoDB ?**
- Attributs produits variables (vêtements ≠ électronique ≠ alimentation)
- Scalabilité horizontale (millions de produits)
- Agrégations pour facettes de recherche
- Performance en lecture pour browsing

**Pourquoi PostgreSQL pour les commandes ?**
- Transactions ACID critiques
- Données relationnelles structurées
- Historique financier immuable
- Conformité comptable

#### 2. Elasticsearch pour la recherche
**Justification :**
- Recherche full-text avancée
- Filtres facettés performants
- Auto-complétion temps réel
- Pertinence configurable

**Alternative :** Atlas Search (MongoDB natif) pour cas simples

#### 3. Redis pour le cache
**Utilisation :**
- Session utilisateur et panier
- Prix calculés et promotions
- Compteurs (vues produit, stock)
- Rate limiting API

## Modélisation des données

### 1. Produits avec attributs variables (Pattern Attribute)

```javascript
// Collection: products
{
  _id: ObjectId("..."),
  sku: "TSHIRT-BLK-M-001",
  type: "configurable",  // simple, configurable, bundle

  // Informations de base
  name: "T-Shirt Premium Cotton",
  slug: "t-shirt-premium-cotton-black",
  description: "Comfortable 100% organic cotton t-shirt...",
  shortDescription: "Premium cotton t-shirt",

  // Brand et manufacturer
  brand: {
    id: ObjectId("..."),
    name: "EcoWear",
    slug: "ecowear",
    logo: "https://cdn.example.com/brands/ecowear.png"
  },

  manufacturer: {
    id: ObjectId("..."),
    name: "Green Textiles Co.",
    country: "Portugal"
  },

  // Catégories (hiérarchie dénormalisée)
  categories: [
    {
      id: ObjectId("..."),
      name: "Vêtements",
      slug: "vetements",
      path: ",vetements,"
    },
    {
      id: ObjectId("..."),
      name: "T-Shirts",
      slug: "t-shirts",
      path: ",vetements,t-shirts,"
    }
  ],

  // Attributs spécifiques (Pattern Attribute)
  attributes: {
    // Attributs communs
    material: "100% Organic Cotton",
    care: "Machine wash cold, tumble dry low",
    origin: "Made in Portugal",

    // Attributs spécifiques vêtements
    gender: "unisex",
    fit: "regular",
    neckline: "crew",
    sleeves: "short",

    // Attributs techniques
    weight: "180g",
    gsm: "180",
    certifications: ["GOTS", "Fair Trade"]
  },

  // Images
  media: {
    images: [
      {
        url: "https://cdn.example.com/products/tshirt-001-main.jpg",
        thumbnail: "https://cdn.example.com/products/tshirt-001-main-thumb.jpg",
        alt: "Black premium cotton t-shirt front view",
        position: 1,
        isMain: true
      },
      {
        url: "https://cdn.example.com/products/tshirt-001-back.jpg",
        thumbnail: "https://cdn.example.com/products/tshirt-001-back-thumb.jpg",
        alt: "Black premium cotton t-shirt back view",
        position: 2,
        isMain: false
      }
    ],
    videos: [
      {
        url: "https://cdn.example.com/videos/tshirt-001-360.mp4",
        thumbnail: "https://cdn.example.com/videos/tshirt-001-360-thumb.jpg",
        type: "360_view"
      }
    ]
  },

  // Prix et inventaire (dénormalisé pour performance)
  pricing: {
    currency: "EUR",
    basePrice: 29.99,
    salePrice: null,
    costPrice: 12.50,  // Pour calculs de marge
    msrp: 39.99,       // Prix conseillé

    // Pricing tiers (quantité)
    tiers: [
      { minQty: 1, price: 29.99 },
      { minQty: 5, price: 27.99 },
      { minQty: 10, price: 25.99 }
    ],

    // Historique de prix (pour tracking)
    priceHistory: [
      {
        price: 29.99,
        startDate: ISODate("2024-01-01T00:00:00Z"),
        endDate: null
      }
    ]
  },

  // Variantes (couleurs, tailles)
  variants: [
    {
      id: "var_001",
      sku: "TSHIRT-BLK-S",
      attributes: {
        color: { label: "Black", value: "black", hex: "#000000" },
        size: { label: "S", value: "s" }
      },
      pricing: {
        basePrice: 29.99,
        salePrice: null
      },
      inventory: {
        qty: 45,
        reserved: 3,
        available: 42,
        warehouses: [
          { id: "WH_EU1", qty: 30 },
          { id: "WH_EU2", qty: 15 }
        ]
      },
      images: ["https://cdn.example.com/products/tshirt-black-s.jpg"]
    },
    {
      id: "var_002",
      sku: "TSHIRT-BLK-M",
      attributes: {
        color: { label: "Black", value: "black", hex: "#000000" },
        size: { label: "M", value: "m" }
      },
      pricing: {
        basePrice: 29.99,
        salePrice: null
      },
      inventory: {
        qty: 67,
        reserved: 5,
        available: 62,
        warehouses: [
          { id: "WH_EU1", qty: 40 },
          { id: "WH_EU2", qty: 27 }
        ]
      },
      images: ["https://cdn.example.com/products/tshirt-black-m.jpg"]
    }
    // ... autres variantes (L, XL, autres couleurs)
  ],

  // Inventaire total (calculé)
  totalInventory: {
    qty: 234,
    reserved: 12,
    available: 222
  },

  // SEO
  seo: {
    title: "Premium Organic Cotton T-Shirt - Black | EcoWear",
    description: "Shop our premium 100% organic cotton t-shirt...",
    keywords: ["organic cotton", "t-shirt", "sustainable fashion"],
    canonicalUrl: "https://shop.example.com/t-shirt-premium-cotton-black"
  },

  // Statistiques (pour ranking et recommandations)
  stats: {
    views: 1547,
    purchases: 89,
    conversionRate: 0.0576,  // 5.76%
    averageRating: 4.6,
    reviewCount: 23,
    wishlistCount: 145,
    returnsCount: 2
  },

  // Informations produit
  details: {
    weight: 0.2,      // kg
    dimensions: {
      length: 72,     // cm
      width: 52,
      height: 2
    },
    shippingClass: "standard",
    taxClass: "clothing",

    // Shipping restrictions
    shippingRestrictions: [],

    // Availability
    availableFrom: ISODate("2024-01-15T00:00:00Z"),
    availableTo: null,

    // Flags
    isFeatured: true,
    isNew: false,
    isBestseller: true,
    isOnSale: false
  },

  // Related products (dénormalisés pour performance)
  related: {
    crossSell: [ObjectId("..."), ObjectId("...")],  // "Acheter avec"
    upSell: [ObjectId("..."), ObjectId("...")],     // "Vous aimerez aussi"
    accessories: [ObjectId("...")],                  // Accessoires
    alternatives: [ObjectId("...")]                  // Alternatives
  },

  // Status et dates
  status: "active",  // draft, active, inactive, discontinued
  visibility: "catalog_search",  // catalog, search, catalog_search, hidden

  createdAt: ISODate("2024-01-10T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z"),
  publishedAt: ISODate("2024-01-15T08:00:00Z"),

  // Version pour migrations
  schemaVersion: 3
}
```

### 2. Index essentiels pour catalogue

```javascript
// ===== Index de recherche et filtrage =====

// Index composé pour listing avec filtres
db.products.createIndex({
  status: 1,
  visibility: 1,
  "categories.id": 1,
  "details.isFeatured": -1,
  "stats.purchases": -1
});

// Index pour slug (unique)
db.products.createIndex({ slug: 1 }, { unique: true });
db.products.createIndex({ sku: 1 }, { unique: true });

// Index pour variantes
db.products.createIndex({ "variants.sku": 1 });

// Index pour marques
db.products.createIndex({
  "brand.id": 1,
  status: 1,
  "stats.purchases": -1
});

// Index pour prix (filtres de gamme)
db.products.createIndex({
  "pricing.basePrice": 1,
  status: 1
});

// Index pour recherche full-text
db.products.createIndex({
  name: "text",
  description: "text",
  "brand.name": "text",
  "attributes.material": "text"
}, {
  weights: {
    name: 10,
    "brand.name": 5,
    description: 3,
    "attributes.material": 2
  },
  default_language: "french"
});

// Index pour tri par popularité
db.products.createIndex({
  status: 1,
  "stats.purchases": -1,
  "stats.averageRating": -1
});

// Index pour nouveautés
db.products.createIndex({
  status: 1,
  publishedAt: -1
});

// Index pour promotions
db.products.createIndex({
  "details.isOnSale": 1,
  status: 1,
  publishedAt: -1
});

// Index géospatial (si availability par localisation)
db.products.createIndex({
  "availability.coordinates": "2dsphere"
});

// ===== Index pour inventory management =====
db.products.createIndex({
  "totalInventory.available": 1,
  status: 1
});

// Index partiel pour low stock alerts
db.products.createIndex(
  { "totalInventory.available": 1 },
  {
    partialFilterExpression: {
      "totalInventory.available": { $lte: 10 }
    }
  }
);
```

### 3. Pattern : Variantes de produits

#### Option A : Variantes embedded (recommandé < 100 variantes)

```javascript
// Collection: products (voir exemple ci-dessus)
// Variantes dans le document principal

// Avantages:
// - Une seule requête pour produit + variantes
// - Cohérence garantie
// - Performance optimale

// Inconvénients:
// - Limite de taille 16 MB
// - Mise à jour d'une variante = update du document entier
```

#### Option B : Variantes séparées (pour produits très variables)

```javascript
// Collection: products (parent)
{
  _id: ObjectId("parent_001"),
  sku: "TSHIRT-PREMIUM",
  type: "configurable",
  name: "T-Shirt Premium Cotton",
  // ... autres infos communes

  // Résumé des variantes (pour filtres rapides)
  variantsSummary: {
    count: 24,  // 4 couleurs × 6 tailles

    // Options disponibles
    colors: [
      { value: "black", label: "Black", hex: "#000000", count: 6 },
      { value: "white", label: "White", hex: "#FFFFFF", count: 6 },
      { value: "navy", label: "Navy", hex: "#001F3F", count: 6 },
      { value: "gray", label: "Gray", hex: "#AAAAAA", count: 6 }
    ],

    sizes: [
      { value: "xs", label: "XS", count: 4 },
      { value: "s", label: "S", count: 4 },
      { value: "m", label: "M", count: 4 },
      { value: "l", label: "L", count: 4 },
      { value: "xl", label: "XL", count: 4 },
      { value: "xxl", label: "XXL", count: 4 }
    ],

    // Prix min/max
    priceRange: {
      min: 29.99,
      max: 34.99
    }
  }
}

// Collection: product_variants
{
  _id: ObjectId("..."),
  parentId: ObjectId("parent_001"),
  sku: "TSHIRT-BLK-M",

  // Attributs spécifiques
  attributes: {
    color: { label: "Black", value: "black", hex: "#000000" },
    size: { label: "M", value: "m" }
  },

  // Prix spécifique si différent
  pricing: {
    basePrice: 29.99,
    salePrice: null
  },

  // Inventaire
  inventory: {
    qty: 67,
    available: 62,
    warehouses: [
      { id: "WH_EU1", qty: 40 },
      { id: "WH_EU2", qty: 27 }
    ]
  },

  // Images spécifiques
  images: ["https://cdn.example.com/products/tshirt-black-m.jpg"],

  // Status
  status: "active",
  createdAt: ISODate("..."),
  updatedAt: ISODate("...")
}

// Index pour variantes
db.product_variants.createIndex({ parentId: 1, status: 1 });
db.product_variants.createIndex({ sku: 1 }, { unique: true });
db.product_variants.createIndex({
  parentId: 1,
  "attributes.color.value": 1,
  "attributes.size.value": 1
});
```

### 4. Gestion du stock et inventaire

#### Modèle avec réservations

```javascript
// Collection: inventory
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),
  variantId: "var_001",  // Si applicable
  sku: "TSHIRT-BLK-M",

  // Quantités
  quantities: {
    physical: 100,      // Stock physique total
    reserved: 8,        // Réservé (paniers actifs)
    committed: 5,       // Commandé mais pas encore expédié
    available: 87,      // Disponible = physical - reserved - committed
    incoming: 50,       // En transit vers entrepôt
    damaged: 2,         // Endommagé
    returned: 3         // Retour en cours de traitement
  },

  // Par entrepôt
  warehouses: [
    {
      id: "WH_EU1",
      name: "Paris Warehouse",
      location: {
        type: "Point",
        coordinates: [2.3522, 48.8566]
      },
      quantities: {
        physical: 60,
        reserved: 5,
        committed: 3,
        available: 52
      }
    },
    {
      id: "WH_EU2",
      name: "Berlin Warehouse",
      location: {
        type: "Point",
        coordinates: [13.4050, 52.5200]
      },
      quantities: {
        physical: 40,
        reserved: 3,
        committed: 2,
        available: 35
      }
    }
  ],

  // Seuils et alertes
  thresholds: {
    low: 20,
    critical: 10,
    reorderPoint: 30,
    reorderQty: 100
  },

  // Mouvements récents (pour audit)
  recentMovements: [
    {
      type: "sale",
      qty: -1,
      warehouse: "WH_EU1",
      orderId: ObjectId("..."),
      timestamp: ISODate("2024-12-09T14:30:00Z")
    },
    {
      type: "restock",
      qty: 50,
      warehouse: "WH_EU1",
      poNumber: "PO-2024-1234",
      timestamp: ISODate("2024-12-08T10:00:00Z")
    }
  ],

  // Dates
  lastRestockedAt: ISODate("2024-12-08T10:00:00Z"),
  lastSoldAt: ISODate("2024-12-09T14:30:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z")
}

// Index pour inventory
db.inventory.createIndex({ sku: 1 }, { unique: true });
db.inventory.createIndex({ productId: 1 });
db.inventory.createIndex({
  "quantities.available": 1,
  "thresholds.low": 1
});
db.inventory.createIndex({
  "warehouses.id": 1,
  "quantities.available": 1
});
```

#### Service de réservation de stock

```javascript
class InventoryService {
  async reserveStock(sku, quantity, cartId, ttl = 900) {
    const session = client.startSession();

    try {
      return await session.withTransaction(async () => {
        // 1. Vérifier disponibilité
        const inventory = await db.collection('inventory').findOne(
          { sku, "quantities.available": { $gte: quantity } },
          { session }
        );

        if (!inventory) {
          throw new Error('Insufficient stock');
        }

        // 2. Créer réservation
        const reservation = {
          _id: new ObjectId(),
          sku,
          quantity,
          cartId,
          status: "active",
          expiresAt: new Date(Date.now() + ttl * 1000),
          createdAt: new Date()
        };

        await db.collection('reservations').insertOne(
          reservation,
          { session }
        );

        // 3. Mettre à jour inventory
        await db.collection('inventory').updateOne(
          { sku },
          {
            $inc: {
              "quantities.reserved": quantity,
              "quantities.available": -quantity
            }
          },
          { session }
        );

        return { success: true, reservationId: reservation._id };
      });
    } finally {
      await session.endSession();
    }
  }

  async releaseReservation(reservationId) {
    const session = client.startSession();

    try {
      return await session.withTransaction(async () => {
        const reservation = await db.collection('reservations').findOne(
          { _id: ObjectId(reservationId) },
          { session }
        );

        if (!reservation || reservation.status !== 'active') {
          return { success: false, message: 'Reservation not found or inactive' };
        }

        // Libérer le stock
        await db.collection('inventory').updateOne(
          { sku: reservation.sku },
          {
            $inc: {
              "quantities.reserved": -reservation.quantity,
              "quantities.available": reservation.quantity
            }
          },
          { session }
        );

        // Marquer réservation comme libérée
        await db.collection('reservations').updateOne(
          { _id: ObjectId(reservationId) },
          {
            $set: {
              status: "released",
              releasedAt: new Date()
            }
          },
          { session }
        );

        return { success: true };
      });
    } finally {
      await session.endSession();
    }
  }

  // Job périodique pour nettoyer réservations expirées
  async cleanupExpiredReservations() {
    const expiredReservations = await db.collection('reservations').find({
      status: "active",
      expiresAt: { $lt: new Date() }
    }).toArray();

    for (const reservation of expiredReservations) {
      await this.releaseReservation(reservation._id);
    }

    console.log(`Cleaned up ${expiredReservations.length} expired reservations`);
  }
}

// Collection: reservations
{
  _id: ObjectId("..."),
  sku: "TSHIRT-BLK-M",
  quantity: 2,
  cartId: "cart_abc123",
  status: "active",  // active, committed, released, expired
  expiresAt: ISODate("2024-12-09T15:00:00Z"),
  createdAt: ISODate("2024-12-09T14:45:00Z")
}

// Index TTL pour auto-cleanup
db.reservations.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);

db.reservations.createIndex({ cartId: 1, status: 1 });
db.reservations.createIndex({ sku: 1, status: 1 });
```

### 5. Pricing et promotions

#### Modèle de promotions

```javascript
// Collection: promotions
{
  _id: ObjectId("..."),
  name: "Black Friday 2024",
  code: "BLACKFRIDAY24",  // Code promo optionnel
  type: "percentage",     // percentage, fixed, buy_x_get_y

  // Règles de réduction
  discount: {
    type: "percentage",
    value: 20,            // 20% de réduction
    maxAmount: 100        // Maximum 100€ de réduction
  },

  // Conditions d'application
  conditions: {
    // Montant minimum
    minOrderAmount: 50,

    // Produits éligibles
    applicableTo: "specific",  // all, category, brand, specific

    products: [
      ObjectId("product_1"),
      ObjectId("product_2")
    ],

    categories: [
      ObjectId("category_1")
    ],

    brands: [
      ObjectId("brand_1")
    ],

    // Exclure certains produits
    excludeProducts: [],

    // Client éligible
    customerSegments: ["premium", "vip"],  // Segments clients
    firstTimeCustomer: false,

    // Limite d'utilisation
    maxUsesTotal: 1000,      // Limite globale
    maxUsesPerCustomer: 1,   // Une fois par client

    // Combinable avec d'autres promos?
    stackable: false
  },

  // Période de validité
  validity: {
    startDate: ISODate("2024-11-29T00:00:00Z"),
    endDate: ISODate("2024-12-01T23:59:59Z"),
    timezone: "Europe/Paris"
  },

  // Statistiques
  stats: {
    timesUsed: 547,
    totalDiscount: 8432.50,
    revenue: 34567.89
  },

  // Status
  status: "active",  // draft, active, expired, disabled
  priority: 10,      // Pour ordre d'application

  createdAt: ISODate("2024-11-01T00:00:00Z"),
  updatedAt: ISODate("2024-11-28T10:00:00Z")
}

// Index pour promotions
db.promotions.createIndex({ code: 1 }, { unique: true, sparse: true });
db.promotions.createIndex({
  status: 1,
  "validity.startDate": 1,
  "validity.endDate": 1
});
db.promotions.createIndex({ "conditions.products": 1 });
db.promotions.createIndex({ "conditions.categories": 1 });
```

#### Service de calcul de prix

```javascript
class PricingService {
  async calculatePrice(productId, quantity, customerId, promoCode = null) {
    // 1. Récupérer produit
    const product = await db.collection('products')
      .findOne({ _id: ObjectId(productId) });

    let basePrice = product.pricing.basePrice;
    let finalPrice = basePrice;
    let appliedPromotions = [];

    // 2. Appliquer prix par palier de quantité
    if (product.pricing.tiers) {
      const tier = product.pricing.tiers
        .filter(t => t.minQty <= quantity)
        .sort((a, b) => b.minQty - a.minQty)[0];

      if (tier) {
        basePrice = tier.price;
        finalPrice = tier.price;
      }
    }

    // 3. Récupérer promotions applicables
    const promotions = await this.getApplicablePromotions(
      productId,
      customerId,
      promoCode
    );

    // 4. Appliquer promotions (par ordre de priorité)
    for (const promo of promotions.sort((a, b) => b.priority - a.priority)) {
      const discountAmount = this.calculateDiscount(
        finalPrice,
        quantity,
        promo
      );

      finalPrice -= discountAmount;
      appliedPromotions.push({
        id: promo._id,
        name: promo.name,
        discount: discountAmount
      });

      // Si non stackable, arrêter
      if (!promo.conditions.stackable) break;
    }

    // 5. Appliquer prix de vente si actif
    if (product.pricing.salePrice && product.pricing.salePrice < finalPrice) {
      finalPrice = product.pricing.salePrice;
    }

    return {
      basePrice,
      finalPrice: Math.max(0, finalPrice),
      totalPrice: Math.max(0, finalPrice) * quantity,
      discount: (basePrice - finalPrice) * quantity,
      appliedPromotions
    };
  }

  async getApplicablePromotions(productId, customerId, promoCode) {
    const now = new Date();
    const customer = customerId
      ? await db.collection('customers').findOne({ _id: ObjectId(customerId) })
      : null;

    const query = {
      status: "active",
      "validity.startDate": { $lte: now },
      "validity.endDate": { $gte: now },

      $or: [
        // Promo globale
        { "conditions.applicableTo": "all" },

        // Promo sur produit spécifique
        { "conditions.products": ObjectId(productId) },

        // Promo par code
        ...(promoCode ? [{ code: promoCode }] : [])
      ]
    };

    let promotions = await db.collection('promotions')
      .find(query)
      .toArray();

    // Filtrer par segment client si nécessaire
    if (customer) {
      promotions = promotions.filter(promo => {
        if (!promo.conditions.customerSegments?.length) return true;
        return promo.conditions.customerSegments.some(
          segment => customer.segments.includes(segment)
        );
      });
    }

    // Vérifier limites d'utilisation
    // ... (logique de vérification)

    return promotions;
  }

  calculateDiscount(price, quantity, promotion) {
    switch (promotion.discount.type) {
      case 'percentage':
        const discount = price * (promotion.discount.value / 100);
        return Math.min(discount, promotion.discount.maxAmount || Infinity);

      case 'fixed':
        return promotion.discount.value;

      case 'buy_x_get_y':
        // Logique "Achetez X obtenez Y gratuit"
        const eligibleForFree = Math.floor(
          quantity / promotion.discount.buyQuantity
        );
        return price * eligibleForFree * promotion.discount.getQuantity;

      default:
        return 0;
    }
  }
}
```

### 6. Recherche et filtres facettés

#### Agrégation pour facettes

```javascript
async function getProductsWithFacets(filters = {}) {
  const {
    category,
    brand,
    minPrice,
    maxPrice,
    color,
    size,
    search,
    page = 1,
    limit = 24,
    sortBy = 'popularity'
  } = filters;

  // Construire match stage
  const matchStage = {
    status: "active",
    visibility: { $in: ["catalog", "catalog_search"] }
  };

  if (category) {
    matchStage["categories.id"] = ObjectId(category);
  }

  if (brand) {
    matchStage["brand.id"] = ObjectId(brand);
  }

  if (minPrice || maxPrice) {
    matchStage["pricing.basePrice"] = {};
    if (minPrice) matchStage["pricing.basePrice"].$gte = minPrice;
    if (maxPrice) matchStage["pricing.basePrice"].$lte = maxPrice;
  }

  if (color) {
    matchStage["variants.attributes.color.value"] = color;
  }

  if (size) {
    matchStage["variants.attributes.size.value"] = size;
  }

  if (search) {
    matchStage.$text = { $search: search };
  }

  // Pipeline d'agrégation
  const pipeline = [
    { $match: matchStage },

    // Facette pour les résultats + métadonnées
    {
      $facet: {
        // Produits
        products: [
          // Tri
          {
            $sort: this.getSortCriteria(sortBy)
          },

          // Pagination
          { $skip: (page - 1) * limit },
          { $limit: limit },

          // Projection
          {
            $project: {
              name: 1,
              slug: 1,
              "brand.name": 1,
              "pricing.basePrice": 1,
              "pricing.salePrice": 1,
              "media.images": { $slice: ["$media.images", 1] },
              "stats.averageRating": 1,
              "stats.reviewCount": 1,
              "details.isFeatured": 1,
              "details.isNew": 1,
              "details.isOnSale": 1
            }
          }
        ],

        // Total count
        totalCount: [
          { $count: "count" }
        ],

        // Facettes de prix
        priceFacets: [
          {
            $bucket: {
              groupBy: "$pricing.basePrice",
              boundaries: [0, 25, 50, 100, 200, 500, 1000],
              default: "1000+",
              output: {
                count: { $sum: 1 }
              }
            }
          }
        ],

        // Facettes de marques
        brandFacets: [
          {
            $group: {
              _id: "$brand.id",
              name: { $first: "$brand.name" },
              count: { $sum: 1 }
            }
          },
          { $sort: { count: -1 } },
          { $limit: 20 }
        ],

        // Facettes de couleurs
        colorFacets: [
          { $unwind: "$variants" },
          {
            $group: {
              _id: "$variants.attributes.color.value",
              label: { $first: "$variants.attributes.color.label" },
              hex: { $first: "$variants.attributes.color.hex" },
              count: { $sum: 1 }
            }
          },
          { $sort: { count: -1 } }
        ],

        // Facettes de tailles
        sizeFacets: [
          { $unwind: "$variants" },
          {
            $group: {
              _id: "$variants.attributes.size.value",
              label: { $first: "$variants.attributes.size.label" },
              count: { $sum: 1 }
            }
          },
          {
            $sort: {
              _id: 1  // Tri par ordre de taille
            }
          }
        ],

        // Statistiques globales
        stats: [
          {
            $group: {
              _id: null,
              minPrice: { $min: "$pricing.basePrice" },
              maxPrice: { $max: "$pricing.basePrice" },
              avgPrice: { $avg: "$pricing.basePrice" },
              avgRating: { $avg: "$stats.averageRating" }
            }
          }
        ]
      }
    }
  ];

  const result = await db.collection('products')
    .aggregate(pipeline)
    .toArray();

  const data = result[0];

  return {
    products: data.products,
    pagination: {
      page,
      limit,
      total: data.totalCount[0]?.count || 0,
      pages: Math.ceil((data.totalCount[0]?.count || 0) / limit)
    },
    facets: {
      prices: data.priceFacets,
      brands: data.brandFacets,
      colors: data.colorFacets,
      sizes: data.sizeFacets
    },
    stats: data.stats[0] || {}
  };
}

getSortCriteria(sortBy) {
  const sortOptions = {
    popularity: { "stats.purchases": -1 },
    newest: { publishedAt: -1 },
    price_asc: { "pricing.basePrice": 1 },
    price_desc: { "pricing.basePrice": -1 },
    rating: { "stats.averageRating": -1 },
    name: { name: 1 }
  };

  return sortOptions[sortBy] || sortOptions.popularity;
}
```

### 7. Panier et wishlist

#### Modèle de panier

```javascript
// Collection: carts
{
  _id: "cart_abc123",  // Session ID ou user ID
  userId: ObjectId("..."),  // null si guest

  // Items du panier
  items: [
    {
      productId: ObjectId("..."),
      variantId: "var_001",
      sku: "TSHIRT-BLK-M",

      // Snapshot du produit (pour prix cohérent)
      productSnapshot: {
        name: "T-Shirt Premium Cotton",
        slug: "t-shirt-premium-cotton-black",
        image: "https://cdn.example.com/products/tshirt-001.jpg",

        // Attributs de la variante
        attributes: {
          color: { label: "Black", value: "black" },
          size: { label: "M", value: "m" }
        }
      },

      // Quantité et prix
      quantity: 2,

      pricing: {
        unitPrice: 29.99,
        originalPrice: 29.99,
        discount: 0,
        totalPrice: 59.98
      },

      // Réservation de stock
      reservationId: ObjectId("..."),
      reservationExpiresAt: ISODate("2024-12-09T15:00:00Z"),

      // Dates
      addedAt: ISODate("2024-12-09T14:30:00Z"),
      updatedAt: ISODate("2024-12-09T14:35:00Z")
    }
  ],

  // Totaux
  summary: {
    subtotal: 59.98,
    discount: 0,
    shipping: 0,        // Calculé au checkout
    tax: 0,             // Calculé au checkout
    total: 59.98,
    itemCount: 1,
    totalQuantity: 2
  },

  // Promotions appliquées
  appliedPromotions: [],
  promoCode: null,

  // Metadata
  currency: "EUR",
  locale: "fr",

  // Status
  status: "active",  // active, abandoned, converted

  // Dates
  createdAt: ISODate("2024-12-09T14:30:00Z"),
  updatedAt: ISODate("2024-12-09T14:35:00Z"),
  expiresAt: ISODate("2024-12-16T14:30:00Z"),  // 7 jours

  // Device info (pour abandoned cart recovery)
  device: {
    type: "mobile",
    os: "iOS",
    browser: "Safari"
  }
}

// Index pour carts
db.carts.createIndex({ userId: 1 });
db.carts.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);
db.carts.createIndex({
  status: 1,
  updatedAt: 1
});
```

#### Service de gestion du panier

```javascript
class CartService {
  async addToCart(cartId, productId, variantId, quantity) {
    // 1. Récupérer produit et variante
    const product = await db.collection('products')
      .findOne({ _id: ObjectId(productId) });

    const variant = product.variants.find(v => v.id === variantId);

    if (!variant || variant.inventory.available < quantity) {
      throw new Error('Product unavailable');
    }

    // 2. Calculer prix
    const pricingService = new PricingService();
    const pricing = await pricingService.calculatePrice(
      productId,
      quantity,
      null  // TODO: get customer ID
    );

    // 3. Réserver stock
    const inventoryService = new InventoryService();
    const reservation = await inventoryService.reserveStock(
      variant.sku,
      quantity,
      cartId,
      900  // 15 minutes
    );

    // 4. Créer item
    const item = {
      productId: ObjectId(productId),
      variantId: variant.id,
      sku: variant.sku,

      productSnapshot: {
        name: product.name,
        slug: product.slug,
        image: variant.images[0] || product.media.images[0].url,
        attributes: variant.attributes
      },

      quantity,

      pricing: {
        unitPrice: pricing.finalPrice,
        originalPrice: pricing.basePrice,
        discount: pricing.discount / quantity,
        totalPrice: pricing.totalPrice
      },

      reservationId: reservation.reservationId,
      reservationExpiresAt: new Date(Date.now() + 900000),

      addedAt: new Date(),
      updatedAt: new Date()
    };

    // 5. Mettre à jour panier
    let cart = await db.collection('carts').findOne({ _id: cartId });

    if (!cart) {
      // Créer nouveau panier
      cart = {
        _id: cartId,
        userId: null,
        items: [item],
        status: "active",
        currency: "EUR",
        locale: "fr",
        createdAt: new Date(),
        updatedAt: new Date(),
        expiresAt: new Date(Date.now() + 7 * 24 * 3600000)
      };

      await db.collection('carts').insertOne(cart);
    } else {
      // Vérifier si produit déjà dans panier
      const existingItemIndex = cart.items.findIndex(
        i => i.variantId === variantId
      );

      if (existingItemIndex >= 0) {
        // Augmenter quantité
        await db.collection('carts').updateOne(
          { _id: cartId, "items.variantId": variantId },
          {
            $inc: {
              "items.$.quantity": quantity,
              "items.$.pricing.totalPrice": pricing.totalPrice
            },
            $set: {
              "items.$.updatedAt": new Date(),
              updatedAt: new Date()
            }
          }
        );
      } else {
        // Ajouter nouvel item
        await db.collection('carts').updateOne(
          { _id: cartId },
          {
            $push: { items: item },
            $set: { updatedAt: new Date() }
          }
        );
      }
    }

    // 6. Recalculer totaux
    await this.recalculateCart(cartId);

    return { success: true, cartId };
  }

  async recalculateCart(cartId) {
    const cart = await db.collection('carts').findOne({ _id: cartId });

    const summary = {
      subtotal: 0,
      discount: 0,
      shipping: 0,
      tax: 0,
      total: 0,
      itemCount: cart.items.length,
      totalQuantity: 0
    };

    for (const item of cart.items) {
      summary.subtotal += item.pricing.totalPrice;
      summary.discount += item.pricing.discount * item.quantity;
      summary.totalQuantity += item.quantity;
    }

    summary.total = summary.subtotal - summary.discount;

    await db.collection('carts').updateOne(
      { _id: cartId },
      {
        $set: {
          summary,
          updatedAt: new Date()
        }
      }
    );

    return summary;
  }

  async removeFromCart(cartId, variantId) {
    const cart = await db.collection('carts').findOne({ _id: cartId });
    const item = cart.items.find(i => i.variantId === variantId);

    if (!item) {
      throw new Error('Item not found in cart');
    }

    // Libérer réservation
    const inventoryService = new InventoryService();
    await inventoryService.releaseReservation(item.reservationId);

    // Retirer du panier
    await db.collection('carts').updateOne(
      { _id: cartId },
      {
        $pull: { items: { variantId } },
        $set: { updatedAt: new Date() }
      }
    );

    // Recalculer totaux
    await this.recalculateCart(cartId);

    return { success: true };
  }
}
```

### 8. Reviews et ratings

```javascript
// Collection: reviews
{
  _id: ObjectId("..."),
  productId: ObjectId("..."),
  customerId: ObjectId("..."),
  orderId: ObjectId("..."),  // Référence à la commande

  // Review
  rating: 5,  // 1-5
  title: "Excellent quality!",
  comment: "This t-shirt is incredibly comfortable...",

  // Photos/vidéos du client
  media: [
    {
      type: "image",
      url: "https://cdn.example.com/reviews/photo-123.jpg",
      thumbnail: "https://cdn.example.com/reviews/photo-123-thumb.jpg"
    }
  ],

  // Attributs notés
  attributes: {
    quality: 5,
    fit: 4,
    comfort: 5,
    valueForMoney: 4
  },

  // Verification
  verified: true,  // Achat vérifié

  // Moderation
  status: "published",  // pending, published, rejected
  moderatedBy: ObjectId("..."),
  moderatedAt: ISODate("..."),

  // Helpfulness
  helpfulCount: 12,
  unhelpfulCount: 1,

  // Metadata
  locale: "fr",
  createdAt: ISODate("2024-11-20T10:00:00Z"),
  updatedAt: ISODate("2024-11-20T10:00:00Z"),

  // Réponse du vendeur
  response: {
    text: "Thank you for your review!",
    respondedBy: ObjectId("..."),
    respondedAt: ISODate("2024-11-21T09:00:00Z")
  }
}

// Index pour reviews
db.reviews.createIndex({ productId: 1, status: 1, createdAt: -1 });
db.reviews.createIndex({ customerId: 1, createdAt: -1 });
db.reviews.createIndex({
  productId: 1,
  rating: 1,
  status: 1
});
```

#### Agrégation des ratings

```javascript
// Pipeline pour calculer statistiques de reviews
async function aggregateProductRatings(productId) {
  const pipeline = [
    {
      $match: {
        productId: ObjectId(productId),
        status: "published"
      }
    },
    {
      $facet: {
        // Statistiques globales
        overall: [
          {
            $group: {
              _id: null,
              averageRating: { $avg: "$rating" },
              totalReviews: { $sum: 1 },

              // Moyennes par attribut
              avgQuality: { $avg: "$attributes.quality" },
              avgFit: { $avg: "$attributes.fit" },
              avgComfort: { $avg: "$attributes.comfort" },
              avgValue: { $avg: "$attributes.valueForMoney" }
            }
          }
        ],

        // Distribution des notes
        distribution: [
          {
            $group: {
              _id: "$rating",
              count: { $sum: 1 }
            }
          },
          { $sort: { _id: -1 } }
        ],

        // Reviews récents
        recent: [
          { $sort: { createdAt: -1 } },
          { $limit: 5 },
          {
            $project: {
              rating: 1,
              title: 1,
              comment: 1,
              "customer.name": 1,
              verified: 1,
              createdAt: 1,
              helpfulCount: 1
            }
          }
        ]
      }
    }
  ];

  const result = await db.collection('reviews')
    .aggregate(pipeline)
    .toArray();

  return result[0];
}

// Mettre à jour stats du produit (Change Stream ou job périodique)
async function updateProductRatingStats(productId) {
  const stats = await aggregateProductRatings(productId);

  await db.collection('products').updateOne(
    { _id: ObjectId(productId) },
    {
      $set: {
        "stats.averageRating": stats.overall[0].averageRating,
        "stats.reviewCount": stats.overall[0].totalReviews,
        "stats.ratingDistribution": stats.distribution,
        "stats.updatedAt": new Date()
      }
    }
  );
}
```

### 9. Recommandations et personnalisation

#### Système de recommandations simple

```javascript
// Collection: user_behavior
{
  _id: ObjectId("..."),
  userId: ObjectId("..."),
  sessionId: "session_abc123",

  // Historique des vues
  viewedProducts: [
    {
      productId: ObjectId("..."),
      timestamp: ISODate("..."),
      duration: 45  // secondes
    }
  ],

  // Historique des achats
  purchasedProducts: [
    {
      productId: ObjectId("..."),
      timestamp: ISODate("..."),
      quantity: 2,
      price: 59.98
    }
  ],

  // Recherches
  searches: [
    {
      query: "t-shirt coton bio",
      timestamp: ISODate("..."),
      resultsClicked: [ObjectId("...")]
    }
  ],

  // Préférences inférées
  preferences: {
    categories: [
      { id: ObjectId("..."), name: "T-Shirts", affinity: 0.85 }
    ],
    brands: [
      { id: ObjectId("..."), name: "EcoWear", affinity: 0.72 }
    ],
    priceRange: { min: 20, max: 50 },
    colors: ["black", "navy", "white"],
    sizes: ["M", "L"]
  },

  updatedAt: ISODate("...")
}
```

#### Service de recommandations

```javascript
class RecommendationService {
  async getRecommendations(userId, productId, type = 'similar') {
    switch (type) {
      case 'similar':
        return this.getSimilarProducts(productId);

      case 'personalized':
        return this.getPersonalizedRecommendations(userId);

      case 'trending':
        return this.getTrendingProducts();

      case 'bought_together':
        return this.getFrequentlyBoughtTogether(productId);

      default:
        return [];
    }
  }

  async getSimilarProducts(productId, limit = 6) {
    const product = await db.collection('products')
      .findOne({ _id: ObjectId(productId) });

    // Produits similaires basés sur:
    // - Même catégorie
    // - Même gamme de prix
    // - Attributs similaires

    const pipeline = [
      {
        $match: {
          _id: { $ne: ObjectId(productId) },
          status: "active",
          "categories.id": { $in: product.categories.map(c => c.id) }
        }
      },

      // Score de similarité
      {
        $addFields: {
          similarityScore: {
            $add: [
              // Score catégorie (0-1)
              {
                $cond: [
                  {
                    $setIsSubset: [
                      product.categories.map(c => c.id),
                      "$categories.id"
                    ]
                  },
                  0.4,
                  0.2
                ]
              },

              // Score prix (0-1)
              {
                $subtract: [
                  1,
                  {
                    $abs: {
                      $divide: [
                        {
                          $subtract: [
                            "$pricing.basePrice",
                            product.pricing.basePrice
                          ]
                        },
                        product.pricing.basePrice
                      ]
                    }
                  }
                ]
              }
            ]
          }
        }
      },

      { $sort: { similarityScore: -1, "stats.purchases": -1 } },
      { $limit: limit },

      {
        $project: {
          name: 1,
          slug: 1,
          "pricing.basePrice": 1,
          "media.images": { $slice: ["$media.images", 1] },
          "stats.averageRating": 1,
          similarityScore: 1
        }
      }
    ];

    return db.collection('products').aggregate(pipeline).toArray();
  }

  async getPersonalizedRecommendations(userId, limit = 12) {
    // Récupérer comportement utilisateur
    const behavior = await db.collection('user_behavior')
      .findOne({ userId: ObjectId(userId) });

    if (!behavior) {
      // Fallback sur trending
      return this.getTrendingProducts(limit);
    }

    // Construire requête basée sur préférences
    const preferredCategories = behavior.preferences.categories
      .map(c => c.id);

    const preferredBrands = behavior.preferences.brands
      .map(b => b.id);

    const viewedProductIds = behavior.viewedProducts
      .slice(-20)  // 20 derniers
      .map(v => v.productId);

    const pipeline = [
      {
        $match: {
          status: "active",
          _id: { $nin: viewedProductIds },  // Exclure déjà vus

          $or: [
            { "categories.id": { $in: preferredCategories } },
            { "brand.id": { $in: preferredBrands } }
          ],

          // Gamme de prix préférée
          "pricing.basePrice": {
            $gte: behavior.preferences.priceRange.min,
            $lte: behavior.preferences.priceRange.max
          }
        }
      },

      // Score de pertinence
      {
        $addFields: {
          relevanceScore: {
            $add: [
              // Score catégorie
              {
                $cond: [
                  { $in: ["$categories.id", preferredCategories] },
                  0.4,
                  0
                ]
              },

              // Score marque
              {
                $cond: [
                  { $in: ["$brand.id", preferredBrands] },
                  0.3,
                  0
                ]
              },

              // Score popularité normalisé
              {
                $multiply: [
                  { $divide: ["$stats.purchases", 1000] },
                  0.2
                ]
              },

              // Score rating
              {
                $multiply: [
                  { $divide: ["$stats.averageRating", 5] },
                  0.1
                ]
              }
            ]
          }
        }
      },

      { $sort: { relevanceScore: -1 } },
      { $limit: limit },

      {
        $project: {
          name: 1,
          slug: 1,
          "brand.name": 1,
          "pricing.basePrice": 1,
          "media.images": { $slice: ["$media.images", 1] },
          "stats.averageRating": 1,
          relevanceScore: 1
        }
      }
    ];

    return db.collection('products').aggregate(pipeline).toArray();
  }

  async getFrequentlyBoughtTogether(productId, limit = 4) {
    // Analyser les commandes contenant ce produit
    // et identifier produits fréquemment achetés ensemble

    const pipeline = [
      // Depuis collection orders (PostgreSQL ou MongoDB)
      {
        $match: {
          "items.productId": ObjectId(productId),
          status: "completed"
        }
      },

      { $unwind: "$items" },

      {
        $match: {
          "items.productId": { $ne: ObjectId(productId) }
        }
      },

      {
        $group: {
          _id: "$items.productId",
          frequency: { $sum: 1 },
          totalRevenue: { $sum: "$items.price" }
        }
      },

      { $sort: { frequency: -1 } },
      { $limit: limit },

      // Lookup pour récupérer détails produits
      {
        $lookup: {
          from: "products",
          localField: "_id",
          foreignField: "_id",
          as: "product"
        }
      },

      { $unwind: "$product" },

      {
        $project: {
          "product.name": 1,
          "product.slug": 1,
          "product.pricing.basePrice": 1,
          "product.media.images": { $slice: ["$product.media.images", 1] },
          frequency: 1
        }
      }
    ];

    // Note: Cette requête serait plus efficace sur une collection
    // orders_analytics pré-calculée
    return db.collection('orders').aggregate(pipeline).toArray();
  }
}
```

## Performance et scalabilité

### Stratégies de cache

```javascript
class ProductCacheService {
  constructor(redis, db) {
    this.redis = redis;
    this.db = db;
    this.TTL = {
      product: 3600,           // 1 heure
      productList: 300,        // 5 minutes
      facets: 600,             // 10 minutes
      inventory: 60,           // 1 minute
      pricing: 300             // 5 minutes
    };
  }

  async getProduct(slug) {
    const cacheKey = `product:${slug}`;

    // Try cache
    let product = await this.redis.get(cacheKey);

    if (product) {
      return JSON.parse(product);
    }

    // Fetch from MongoDB
    product = await this.db.collection('products').findOne({
      slug,
      status: "active"
    });

    if (product) {
      // Cache for 1 hour
      await this.redis.setex(
        cacheKey,
        this.TTL.product,
        JSON.stringify(product)
      );
    }

    return product;
  }

  async invalidateProduct(productId) {
    const product = await this.db.collection('products')
      .findOne({ _id: ObjectId(productId) });

    if (product) {
      // Invalider cache produit
      await this.redis.del(`product:${product.slug}`);

      // Invalider listes contenant ce produit
      const categories = product.categories.map(c => c.slug);
      for (const category of categories) {
        await this.redis.del(`products:category:${category}:*`);
      }

      // Invalider CDN
      await this.purgeCDN([
        `/products/${product.slug}`,
        `/api/products/${product.slug}`
      ]);
    }
  }
}
```

### Sharding pour grande échelle

```javascript
// Stratégie de sharding pour catalogues très larges (> 10M produits)

// Choix de shard key: categories.id + _id
// Justification:
// - Queries souvent filtrées par catégorie
// - Distribution équitable si catégories balancées
// - Évite hotspots sur produits populaires

// Configuration sharding
sh.enableSharding("ecommerce");

sh.shardCollection(
  "ecommerce.products",
  { "categories.id": 1, _id: 1 }
);

// Zones de sharding pour isolation géographique
sh.addShardToZone("shard01", "EU");
sh.addShardToZone("shard02", "US");
sh.addShardToZone("shard03", "ASIA");

// Tag ranges par catégorie principale
sh.updateZoneKeyRange(
  "ecommerce.products",
  { "categories.id": ObjectId("eu_category_1"), _id: MinKey },
  { "categories.id": ObjectId("eu_category_1"), _id: MaxKey },
  "EU"
);
```

## Monitoring et métriques

### Métriques clés pour e-commerce

```javascript
const ecommerceMetrics = {
  // Performance catalogue
  'catalog.response_time.p95': {
    description: 'Product page load time',
    threshold: 200,  // ms
    alert: 'slow_catalog_response'
  },

  'catalog.search_time.p95': {
    description: 'Search results response time',
    threshold: 300,  // ms
    alert: 'slow_search'
  },

  // Inventory
  'inventory.low_stock_products': {
    description: 'Products with stock < threshold',
    query: 'db.inventory.count({ "quantities.available": { $lte: 10 } })',
    threshold: 50,
    alert: 'low_stock_alert'
  },

  'inventory.out_of_stock': {
    description: 'Products completely out of stock',
    query: 'db.inventory.count({ "quantities.available": 0 })',
    threshold: 10,
    alert: 'out_of_stock_alert'
  },

  // Business metrics
  'cart.abandonment_rate': {
    description: 'Cart abandonment rate',
    formula: '(abandoned_carts / total_carts) * 100',
    threshold: 70,  // %
    alert: 'high_abandonment'
  },

  'products.conversion_rate': {
    description: 'Product view to purchase conversion',
    formula: '(purchases / views) * 100',
    target: 3  // %
  }
};
```

## Checklist de déploiement

### ✅ Modélisation et schéma

- [ ] Attributs produits flexibles (Pattern Attribute)
- [ ] Variantes optimisées (embedded < 100, sinon séparé)
- [ ] Hiérarchies catégories avec Materialized Path
- [ ] Inventory avec réservations
- [ ] Reviews et ratings intégrés

### ✅ Performance

- [ ] Index pour toutes requêtes fréquentes (listing, filtres)
- [ ] Index text pour recherche ou Elasticsearch déployé
- [ ] Cache multi-niveaux (CDN + Redis + MongoDB)
- [ ] Agrégations optimisées pour facettes
- [ ] Images optimisées et CDN configuré
- [ ] Pagination cursor-based

### ✅ Scalabilité

- [ ] Sharding configuré si > 10M produits
- [ ] Read replicas pour analytics
- [ ] Connection pooling optimisé
- [ ] Rate limiting API
- [ ] Queue pour opérations asynchrones

### ✅ Business

- [ ] Système de réservation stock
- [ ] Pricing dynamique et promotions
- [ ] Panier avec expiration
- [ ] Recommandations configurées
- [ ] Analytics produits

### ✅ Monitoring

- [ ] Métriques performance catalogue
- [ ] Alertes stock bas/rupture
- [ ] Tracking abandons panier
- [ ] Monitoring taux conversion
- [ ] Logs centralisés

## Conclusion

MongoDB est particulièrement adapté aux catalogues e-commerce grâce à :

**✅ Forces démontrées :**
- Flexibilité de schéma pour produits hétérogènes
- Performance en lecture pour browsing intensif
- Agrégations pour filtres facettés complexes
- Scalabilité horizontale pour croissance
- Dénormalisation naturelle pour performance

**⚠️ Considérations importantes :**
- Inventory management nécessite réservations rigoureuses
- Synchronisation avec système de commandes (PostgreSQL)
- Cache agressif essentiel pour performance
- Elasticsearch recommandé pour recherche avancée
- Monitoring stock critique

**🎯 Patterns essentiels e-commerce :**
1. **Attribute Pattern** pour attributs variables
2. **Extended Reference** pour dénormalisation
3. **Computed Pattern** pour statistiques
4. **Bucket Pattern** pour historique prix
5. **Subset Pattern** pour variantes nombreuses

Cette architecture supporte des catalogues de quelques milliers à plusieurs millions de produits, avec possibilité de scaler horizontalement selon la croissance.

---

**Références :**
- MongoDB Blog: "E-commerce Architecture Patterns"
- Shopify Engineering Blog
- Amazon DynamoDB Best Practices (patterns transposables)
- "Building Microservices E-commerce" - Sam Newman

⏭️ [Internet des objets (IoT)](/20-cas-usage-architectures/04-internet-des-objets-iot.md)
