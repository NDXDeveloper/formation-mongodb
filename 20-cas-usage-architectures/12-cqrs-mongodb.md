🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.12 CQRS et MongoDB

## Introduction

CQRS (Command Query Responsibility Segregation) est un pattern architectural qui sépare les opérations de lecture (Queries) des opérations d'écriture (Commands). Au lieu d'utiliser un seul modèle pour lire et écrire, CQRS utilise des modèles distincts optimisés pour leurs usages spécifiques.

**Principes fondamentaux :**
- **Séparation Command/Query** : Modèles distincts pour write et read
- **Optimisation ciblée** : Chaque côté optimisé pour son usage
- **Eventual consistency** : Read models mis à jour de façon asynchrone
- **Scalabilité indépendante** : Scale reads et writes séparément
- **Multiple read models** : Plusieurs vues optimisées depuis même write model

**Différence avec approches traditionnelles :**

```
Traditional (CRUD):
Client → API → Single Model → Database
         (Read & Write mélangés)

CQRS:
Client → API → Command Handler → Write Model → Write DB
Client → API → Query Handler → Read Model → Read DB
                               ↑
                    Synchronisation asynchrone
```

**Avantages :**
- ✅ Performance optimale (read et write séparés)
- ✅ Scalabilité indépendante
- ✅ Read models dénormalisés pour queries complexes
- ✅ Security (séparation permissions read/write)
- ✅ Multiple projections optimisées
- ✅ Évolution indépendante des modèles
- ✅ Complexité métier isolée (write side)

**Challenges :**
- ❌ Complexité accrue
- ❌ Eventual consistency
- ❌ Duplication de données
- ❌ Synchronisation à gérer
- ❌ Testing plus complexe

MongoDB est excellent pour CQRS grâce à :
- **Flexible schema** : Modèles optimisés différemment
- **Change Streams** : Synchronisation temps réel
- **Aggregation Pipeline** : Transformations pour read models
- **Multiple databases** : Isolation write/read
- **Transactions** : Consistency côté write
- **Denormalization** : Read models optimisés

## Architecture de référence

### Stack CQRS complète

```
┌────────────────────────────────────────────────────────────┐
│                      Client Layer                          │
│              Web App • Mobile • Third-party API            │
└───────────────────┬────────────────────┬───────────────────┘
                    │                    │
         Commands   │                    │   Queries
                    │                    │
┌───────────────────▼──────┐  ┌──────────▼───────────────────┐
│   Command API            │  │    Query API                 │
│   • POST /orders         │  │    • GET /orders             │
│   • PUT /orders/:id      │  │    • GET /orders/:id         │
│   • DELETE /orders/:id   │  │    • GET /orders/search      │
└───────────────────┬──────┘  └─────────┬────────────────────┘
                    │                   │
┌───────────────────▼──────┐  ┌─────────▼────────────────────┐
│   Command Handler        │  │    Query Handler             │
│   • Validate             │  │    • No business logic       │
│   • Load Aggregate       │  │    • Direct DB queries       │
│   • Execute Logic        │  │    • Optimized reads         │
│   • Save Events/State    │  │    • Caching                 │
└───────────────────┬──────┘  └─────────┬────────────────────┘
                    │                   │
┌───────────────────▼──────┐  ┌─────────▼────────────────────┐
│      Write Model         │  │       Read Model             │
│   MongoDB (Normalized)   │  │   MongoDB (Denormalized)     │
│                          │  │                              │
│  ┌──────────────────┐    │  │  ┌──────────────────┐        │
│  │  orders          │    │  │  │  orders_read     │        │
│  │  • Normalized    │    │  │  │  • Denormalized  │        │
│  │  • Complex       │    │  │  │  • Embedded data │        │
│  │  • ACID          │    │  │  │  • Optimized     │        │
│  └──────────────────┘    │  │  └──────────────────┘        │
│                          │  │                              │
│  ┌──────────────────┐    │  │  ┌──────────────────┐        │
│  │  customers       │    │  │  │  orders_summary  │        │
│  └──────────────────┘    │  │  │  (Analytics)     │        │
│                          │  │  └──────────────────┘        │
│  ┌──────────────────┐    │  │                              │
│  │  products        │    │  │  ┌──────────────────┐        │
│  └──────────────────┘    │  │  │  product_catalog │        │
│                          │  │  │  (Search)        │        │
└───────────────────┬──────┘  │  └──────────────────┘        │
                    │         └──────────────────────────────┘
                    │
         Change     │
         Streams    │
                    │
┌───────────────────▼──────────────────────────────────────┐
│               Projection Builder                         │
│   • Listen to Write Model changes                        │
│   • Transform data                                       │
│   • Update Read Models                                   │
│   • Handle failures                                      │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                Supporting Infrastructure                 │
├──────────────────────────────────────────────────────────┤
│  Event Bus (optional): Kafka / RabbitMQ / Redis Streams  │
│  Cache: Redis for hot read data                          │
│  Search: Elasticsearch for full-text (optional)          │
│  Monitoring: Track lag between write and read models     │
└──────────────────────────────────────────────────────────┘
```

### Composants architecturaux

#### 1. Command Side (Write Model)
**Responsabilités :**
- Valider commandes
- Exécuter logique métier
- Maintenir consistency (ACID)
- Émettre événements de changement
- Optimisé pour writes

**Technologies :** MongoDB avec transactions, schema normalisé

#### 2. Query Side (Read Model)
**Responsabilités :**
- Servir les lectures
- Aucune logique métier
- Données dénormalisées
- Optimisé pour queries spécifiques
- Eventual consistency acceptable

**Technologies :** MongoDB avec schema dénormalisé, index optimisés

#### 3. Projection Builder
**Responsabilités :**
- Synchroniser write → read models
- Écouter changements (Change Streams)
- Transformer données
- Gérer idempotence
- Resilience et retry

#### 4. Event Bus (optionnel)
**Technologies :** Kafka, RabbitMQ, Redis Streams

**Justification :** Découplage, ordering garanti, replay capability

## Modélisation CQRS

### 1. Write Model (Command Side)

```javascript
// Write Model - Normalized, transactional

// Collection: orders (write)
{
  _id: ObjectId("..."),
  orderId: "order_abc123",

  // Références (normalized)
  customerId: "cust_xyz789",

  // Items avec références produits
  items: [
    {
      productId: "prod_def456",
      quantity: 2,
      priceAtOrder: NumberDecimal("299.99")
    }
  ],

  // Montants
  subtotal: NumberDecimal("599.98"),
  tax: NumberDecimal("59.99"),
  shipping: NumberDecimal("10.00"),
  total: NumberDecimal("669.97"),

  currency: "USD",

  // Status workflow
  status: "pending",  // pending, confirmed, shipped, delivered, cancelled

  // Addresses (références)
  shippingAddressId: "addr_123",
  billingAddressId: "addr_456",

  // Metadata
  createdAt: ISODate("2024-12-09T14:30:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z"),

  // Version pour optimistic locking
  version: 1
}

// Collection: customers (write)
{
  _id: ObjectId("..."),
  customerId: "cust_xyz789",

  name: "John Doe",
  email: "john.doe@example.com",

  status: "active",

  createdAt: ISODate("2024-01-15T10:00:00Z"),
  version: 1
}

// Collection: products (write)
{
  _id: ObjectId("..."),
  productId: "prod_def456",

  name: "Wireless Headphones",
  sku: "WH-1000XM5",

  price: NumberDecimal("299.99"),
  currency: "USD",

  inventory: {
    quantity: 150,
    reserved: 10
  },

  status: "active",
  version: 1
}

// Index pour write model (minimal, optimisé pour writes)
db.orders.createIndex({ orderId: 1 }, { unique: true });
db.orders.createIndex({ customerId: 1, createdAt: -1 });
db.orders.createIndex({ status: 1 });

db.customers.createIndex({ customerId: 1 }, { unique: true });
db.customers.createIndex({ email: 1 }, { unique: true });

db.products.createIndex({ productId: 1 }, { unique: true });
db.products.createIndex({ sku: 1 }, { unique: true });
```

### 2. Read Model (Query Side)

```javascript
// Read Model - Denormalized, optimized for queries

// Collection: orders_read (denormalized)
{
  _id: ObjectId("..."),
  orderId: "order_abc123",

  // Customer data embedded (dénormalisé)
  customer: {
    customerId: "cust_xyz789",
    name: "John Doe",
    email: "john.doe@example.com",

    // Informations additionnelles utiles pour affichage
    phone: "+1234567890",
    vipStatus: "gold"
  },

  // Items enrichis avec product details
  items: [
    {
      productId: "prod_def456",

      // Product details embedded
      product: {
        name: "Wireless Headphones",
        sku: "WH-1000XM5",
        imageUrl: "https://cdn.example.com/products/...",
        category: "Electronics",
        brand: "Sony"
      },

      quantity: 2,
      unitPrice: NumberDecimal("299.99"),
      subtotal: NumberDecimal("599.98")
    }
  ],

  // Montants
  subtotal: NumberDecimal("599.98"),
  tax: NumberDecimal("59.99"),
  shipping: NumberDecimal("10.00"),
  total: NumberDecimal("669.97"),
  currency: "USD",

  // Status
  status: "pending",
  statusHistory: [
    {
      status: "pending",
      timestamp: ISODate("2024-12-09T14:30:00Z")
    }
  ],

  // Shipping address embedded
  shippingAddress: {
    street: "123 Main St",
    city: "New York",
    state: "NY",
    zipCode: "10001",
    country: "US"
  },

  // Billing address embedded
  billingAddress: {
    street: "123 Main St",
    city: "New York",
    state: "NY",
    zipCode: "10001",
    country: "US"
  },

  // Metadata
  createdAt: ISODate("2024-12-09T14:30:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z"),

  // Tracking de synchronisation
  lastSyncedFrom: {
    writeModelVersion: 1,
    syncedAt: ISODate("2024-12-09T14:30:01Z")
  },

  // Pour recherche full-text
  searchText: "john doe wireless headphones sony wh-1000xm5"
}

// Collection: orders_summary (analytics read model)
{
  _id: ObjectId("..."),
  date: ISODate("2024-12-09T00:00:00Z"),

  // Agrégations par jour
  metrics: {
    orderCount: 142,
    totalRevenue: NumberDecimal("45678.90"),
    avgOrderValue: NumberDecimal("321.68"),

    // Par status
    byStatus: {
      pending: 23,
      confirmed: 56,
      shipped: 45,
      delivered: 18
    },

    // Top customers
    topCustomers: [
      {
        customerId: "cust_xyz789",
        name: "John Doe",
        orderCount: 5,
        totalSpent: NumberDecimal("2345.67")
      }
    ],

    // Top products
    topProducts: [
      {
        productId: "prod_def456",
        name: "Wireless Headphones",
        unitsSold: 45,
        revenue: NumberDecimal("13499.55")
      }
    ]
  },

  updatedAt: ISODate("2024-12-09T23:59:59Z")
}

// Collection: product_catalog (search read model)
{
  _id: ObjectId("..."),
  productId: "prod_def456",

  // Données optimisées pour recherche et affichage
  name: "Wireless Headphones",
  displayName: "Sony WH-1000XM5 Wireless Noise-Cancelling Headphones",
  description: "Premium over-ear headphones with industry-leading noise cancellation...",

  sku: "WH-1000XM5",
  brand: "Sony",
  category: "Electronics > Audio > Headphones",

  // Prix
  price: NumberDecimal("299.99"),
  currency: "USD",

  // Images multiples
  images: [
    {
      url: "https://cdn.example.com/products/wh1000xm5-main.jpg",
      type: "main"
    },
    {
      url: "https://cdn.example.com/products/wh1000xm5-side.jpg",
      type: "gallery"
    }
  ],

  // Features pour filtres
  features: {
    color: ["Black", "Silver"],
    connectivity: ["Bluetooth", "USB-C", "3.5mm"],
    batteryLife: "30 hours",
    weight: "250g"
  },

  // Métriques
  metrics: {
    averageRating: 4.7,
    reviewCount: 892,
    viewCount: 15420,
    purchaseCount: 234,

    // Popularité
    popularityScore: 0.87,
    trendingScore: 0.92
  },

  // Availability
  inStock: true,
  stockLevel: 140,

  // SEO
  slug: "sony-wh-1000xm5-wireless-headphones",
  metaDescription: "...",

  // Pour recherche
  searchKeywords: ["wireless", "noise-cancelling", "sony", "headphones", "bluetooth"],

  updatedAt: ISODate("2024-12-09T14:30:00Z")
}

// Index pour read models (nombreux, optimisés pour queries)
db.orders_read.createIndex({ orderId: 1 }, { unique: true });
db.orders_read.createIndex({ "customer.customerId": 1, createdAt: -1 });
db.orders_read.createIndex({ "customer.email": 1 });
db.orders_read.createIndex({ status: 1, createdAt: -1 });
db.orders_read.createIndex({ createdAt: -1 });
db.orders_read.createIndex({ total: 1 });

// Text index pour recherche
db.orders_read.createIndex({ searchText: "text" });

// Index pour analytics
db.orders_summary.createIndex({ date: -1 }, { unique: true });

// Index pour catalog
db.product_catalog.createIndex({ productId: 1 }, { unique: true });
db.product_catalog.createIndex({ category: 1, "metrics.popularityScore": -1 });
db.product_catalog.createIndex({ brand: 1 });
db.product_catalog.createIndex({ "metrics.trendingScore": -1 });
db.product_catalog.createIndex({ price: 1 });
db.product_catalog.createIndex({ inStock: 1 });

// Text index pour recherche full-text
db.product_catalog.createIndex({
  name: "text",
  description: "text",
  searchKeywords: "text"
}, {
  weights: {
    name: 10,
    searchKeywords: 5,
    description: 1
  }
});
```

## Command Handler (Write Side)

### 1. Order Command Handler

```javascript
class OrderCommandHandler {
  constructor(db) {
    this.db = db;
  }

  async createOrder(command) {
    const session = this.db.client.startSession();

    try {
      let order;

      await session.withTransaction(async () => {
        // 1. Valider customer
        const customer = await this.db.collection('customers')
          .findOne(
            { customerId: command.customerId },
            { session }
          );

        if (!customer || customer.status !== 'active') {
          throw new Error('Invalid customer');
        }

        // 2. Valider et réserver products
        const items = [];
        let subtotal = 0;

        for (const item of command.items) {
          const product = await this.db.collection('products')
            .findOneAndUpdate(
              {
                productId: item.productId,
                status: 'active',
                'inventory.quantity': {
                  $gte: item.quantity
                }
              },
              {
                $inc: {
                  'inventory.quantity': -item.quantity,
                  'inventory.reserved': item.quantity
                }
              },
              {
                session,
                returnDocument: 'before'
              }
            );

          if (!product) {
            throw new Error(
              `Product ${item.productId} not available`
            );
          }

          const itemSubtotal = product.price * item.quantity;

          items.push({
            productId: item.productId,
            quantity: item.quantity,
            priceAtOrder: product.price
          });

          subtotal += itemSubtotal;
        }

        // 3. Calculer totaux
        const tax = subtotal * 0.1;  // 10% tax
        const shipping = NumberDecimal("10.00");
        const total = subtotal + tax + shipping;

        // 4. Créer commande
        order = {
          orderId: this.generateOrderId(),
          customerId: command.customerId,

          items,

          subtotal: NumberDecimal(subtotal.toString()),
          tax: NumberDecimal(tax.toString()),
          shipping,
          total: NumberDecimal(total.toString()),

          currency: "USD",
          status: "pending",

          shippingAddressId: command.shippingAddressId,
          billingAddressId: command.billingAddressId,

          createdAt: new Date(),
          updatedAt: new Date(),
          version: 1
        };

        const result = await this.db.collection('orders')
          .insertOne(order, { session });

        order._id = result.insertedId;

      }, {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority', j: true },
        readPreference: 'primary'
      });

      return {
        success: true,
        orderId: order.orderId
      };

    } catch (error) {
      console.error('Create order failed:', error);

      return {
        success: false,
        error: error.message
      };

    } finally {
      await session.endSession();
    }
  }

  async updateOrderStatus(command) {
    const session = this.db.client.startSession();

    try {
      let updated;

      await session.withTransaction(async () => {
        // Optimistic locking
        updated = await this.db.collection('orders')
          .findOneAndUpdate(
            {
              orderId: command.orderId,
              version: command.expectedVersion
            },
            {
              $set: {
                status: command.newStatus,
                updatedAt: new Date()
              },
              $inc: { version: 1 }
            },
            {
              session,
              returnDocument: 'after'
            }
          );

        if (!updated) {
          throw new Error('Order not found or version conflict');
        }

        // Business logic selon status
        if (command.newStatus === 'cancelled') {
          // Libérer inventory
          for (const item of updated.items) {
            await this.db.collection('products').updateOne(
              { productId: item.productId },
              {
                $inc: {
                  'inventory.quantity': item.quantity,
                  'inventory.reserved': -item.quantity
                }
              },
              { session }
            );
          }
        }

      }, {
        writeConcern: { w: 'majority', j: true }
      });

      return {
        success: true,
        orderId: updated.orderId,
        version: updated.version
      };

    } catch (error) {
      console.error('Update order status failed:', error);

      return {
        success: false,
        error: error.message
      };

    } finally {
      await session.endSession();
    }
  }

  generateOrderId() {
    return `order_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### 2. Command Validation

```javascript
class CommandValidator {
  async validateCreateOrder(command) {
    const errors = [];

    // Validation basique
    if (!command.customerId) {
      errors.push('customerId is required');
    }

    if (!command.items || command.items.length === 0) {
      errors.push('items are required');
    }

    // Validation items
    for (const item of command.items || []) {
      if (!item.productId) {
        errors.push('productId is required for all items');
      }

      if (!item.quantity || item.quantity <= 0) {
        errors.push('quantity must be positive');
      }
    }

    // Validation addresses
    if (!command.shippingAddressId) {
      errors.push('shippingAddressId is required');
    }

    if (!command.billingAddressId) {
      errors.push('billingAddressId is required');
    }

    if (errors.length > 0) {
      throw new ValidationError(errors);
    }
  }

  async validateUpdateOrderStatus(command) {
    const errors = [];

    if (!command.orderId) {
      errors.push('orderId is required');
    }

    if (!command.newStatus) {
      errors.push('newStatus is required');
    }

    const validStatuses = [
      'pending',
      'confirmed',
      'shipped',
      'delivered',
      'cancelled'
    ];

    if (!validStatuses.includes(command.newStatus)) {
      errors.push(`newStatus must be one of: ${validStatuses.join(', ')}`);
    }

    if (errors.length > 0) {
      throw new ValidationError(errors);
    }
  }
}
```

## Query Handler (Read Side)

### 1. Order Query Service

```javascript
class OrderQueryService {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }

  async getOrder(orderId) {
    // Essayer cache
    const cacheKey = `order:${orderId}`;
    const cached = await this.cache.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Query depuis read model (denormalized)
    const order = await this.db.collection('orders_read')
      .findOne({ orderId });

    if (order) {
      // Cache pour 5 minutes
      await this.cache.setex(cacheKey, 300, JSON.stringify(order));
    }

    return order;
  }

  async getCustomerOrders(customerId, options = {}) {
    const {
      status,
      fromDate,
      toDate,
      limit = 20,
      offset = 0,
      sortBy = 'createdAt',
      sortOrder = -1
    } = options;

    // Construire query
    const query = {
      'customer.customerId': customerId
    };

    if (status) {
      query.status = status;
    }

    if (fromDate || toDate) {
      query.createdAt = {};
      if (fromDate) query.createdAt.$gte = new Date(fromDate);
      if (toDate) query.createdAt.$lte = new Date(toDate);
    }

    // Query depuis read model optimisé
    const orders = await this.db.collection('orders_read')
      .find(query)
      .sort({ [sortBy]: sortOrder })
      .skip(offset)
      .limit(limit)
      .toArray();

    // Count total pour pagination
    const total = await this.db.collection('orders_read')
      .countDocuments(query);

    return {
      orders,
      pagination: {
        total,
        limit,
        offset,
        pages: Math.ceil(total / limit)
      }
    };
  }

  async searchOrders(searchText, filters = {}) {
    const query = {};

    // Text search
    if (searchText) {
      query.$text = { $search: searchText };
    }

    // Filters
    if (filters.status) {
      query.status = filters.status;
    }

    if (filters.minTotal) {
      query.total = { $gte: NumberDecimal(filters.minTotal.toString()) };
    }

    if (filters.maxTotal) {
      query.total = {
        ...query.total,
        $lte: NumberDecimal(filters.maxTotal.toString())
      };
    }

    const orders = await this.db.collection('orders_read')
      .find(query)
      .sort({ score: { $meta: 'textScore' } })
      .limit(50)
      .toArray();

    return orders;
  }

  async getOrdersSummary(date) {
    // Query depuis analytics read model
    const summary = await this.db.collection('orders_summary')
      .findOne({ date: this.getStartOfDay(date) });

    return summary;
  }

  getStartOfDay(date) {
    const start = new Date(date);
    start.setHours(0, 0, 0, 0);
    return start;
  }
}
```

### 2. Product Query Service

```javascript
class ProductQueryService {
  constructor(db, cache) {
    this.db = db;
    this.cache = cache;
  }

  async searchProducts(query, filters = {}, options = {}) {
    const {
      limit = 20,
      offset = 0
    } = options;

    // Construire query MongoDB
    const mongoQuery = {};

    // Text search
    if (query) {
      mongoQuery.$text = { $search: query };
    }

    // Filters
    if (filters.category) {
      mongoQuery.category = new RegExp(`^${filters.category}`);
    }

    if (filters.brand) {
      mongoQuery.brand = filters.brand;
    }

    if (filters.minPrice || filters.maxPrice) {
      mongoQuery.price = {};
      if (filters.minPrice) {
        mongoQuery.price.$gte = NumberDecimal(filters.minPrice.toString());
      }
      if (filters.maxPrice) {
        mongoQuery.price.$lte = NumberDecimal(filters.maxPrice.toString());
      }
    }

    if (filters.inStock) {
      mongoQuery.inStock = true;
    }

    // Features filters
    if (filters.features) {
      for (const [key, value] of Object.entries(filters.features)) {
        mongoQuery[`features.${key}`] = value;
      }
    }

    // Sort
    let sort = {};

    if (query) {
      sort = { score: { $meta: 'textScore' } };
    } else if (filters.sortBy === 'price') {
      sort = { price: filters.sortOrder || 1 };
    } else if (filters.sortBy === 'popularity') {
      sort = { 'metrics.popularityScore': -1 };
    } else if (filters.sortBy === 'rating') {
      sort = { 'metrics.averageRating': -1 };
    } else {
      sort = { 'metrics.trendingScore': -1 };
    }

    // Query depuis catalog read model
    const products = await this.db.collection('product_catalog')
      .find(mongoQuery)
      .sort(sort)
      .skip(offset)
      .limit(limit)
      .toArray();

    const total = await this.db.collection('product_catalog')
      .countDocuments(mongoQuery);

    return {
      products,
      pagination: {
        total,
        limit,
        offset,
        pages: Math.ceil(total / limit)
      }
    };
  }

  async getProduct(productId) {
    const cacheKey = `product:${productId}`;
    const cached = await this.cache.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    const product = await this.db.collection('product_catalog')
      .findOne({ productId });

    if (product) {
      await this.cache.setex(cacheKey, 600, JSON.stringify(product));
    }

    return product;
  }

  async getTrendingProducts(limit = 10) {
    const cacheKey = 'trending_products';
    const cached = await this.cache.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    const products = await this.db.collection('product_catalog')
      .find({ inStock: true })
      .sort({ 'metrics.trendingScore': -1 })
      .limit(limit)
      .toArray();

    // Cache pour 5 minutes
    await this.cache.setex(cacheKey, 300, JSON.stringify(products));

    return products;
  }
}
```

## Projection Builder (Synchronisation)

### 1. Change Streams Projections

```javascript
class ProjectionBuilder {
  constructor(writeDb, readDb) {
    this.writeDb = writeDb;
    this.readDb = readDb;
  }

  async start() {
    // Démarrer projections pour chaque collection
    await Promise.all([
      this.projectOrders(),
      this.projectProducts(),
      this.projectAnalytics()
    ]);

    console.log('All projections started');
  }

  async projectOrders() {
    // Watch changes sur orders (write model)
    const changeStream = this.writeDb.collection('orders').watch([
      {
        $match: {
          operationType: { $in: ['insert', 'update', 'replace'] }
        }
      }
    ], {
      fullDocument: 'updateLookup'
    });

    changeStream.on('change', async (change) => {
      try {
        await this.handleOrderChange(change);
      } catch (error) {
        console.error('Failed to project order:', error);
        // Dead letter queue
        await this.sendToDeadLetter('orders', change, error);
      }
    });

    console.log('Order projection started');
  }

  async handleOrderChange(change) {
    const order = change.fullDocument;

    if (!order) return;

    // Enrichir avec customer data
    const customer = await this.writeDb.collection('customers')
      .findOne({ customerId: order.customerId });

    if (!customer) {
      console.warn(`Customer ${order.customerId} not found`);
      return;
    }

    // Enrichir avec product data
    const enrichedItems = await this.enrichOrderItems(order.items);

    // Récupérer addresses
    const [shippingAddress, billingAddress] = await Promise.all([
      this.getAddress(order.shippingAddressId),
      this.getAddress(order.billingAddressId)
    ]);

    // Construire read model dénormalisé
    const orderRead = {
      orderId: order.orderId,

      customer: {
        customerId: customer.customerId,
        name: customer.name,
        email: customer.email,
        phone: customer.phone,
        vipStatus: customer.vipStatus || 'regular'
      },

      items: enrichedItems,

      subtotal: order.subtotal,
      tax: order.tax,
      shipping: order.shipping,
      total: order.total,
      currency: order.currency,

      status: order.status,
      statusHistory: this.buildStatusHistory(order),

      shippingAddress,
      billingAddress,

      createdAt: order.createdAt,
      updatedAt: order.updatedAt,

      lastSyncedFrom: {
        writeModelVersion: order.version,
        syncedAt: new Date()
      },

      searchText: this.buildSearchText(customer, enrichedItems)
    };

    // Upsert dans read model
    await this.readDb.collection('orders_read').replaceOne(
      { orderId: order.orderId },
      orderRead,
      { upsert: true }
    );

    console.log(`Order ${order.orderId} projected to read model`);
  }

  async enrichOrderItems(items) {
    const enriched = [];

    for (const item of items) {
      const product = await this.writeDb.collection('products')
        .findOne({ productId: item.productId });

      if (!product) {
        console.warn(`Product ${item.productId} not found`);
        continue;
      }

      enriched.push({
        productId: item.productId,

        product: {
          name: product.name,
          sku: product.sku,
          imageUrl: product.imageUrl,
          category: product.category,
          brand: product.brand
        },

        quantity: item.quantity,
        unitPrice: item.priceAtOrder,
        subtotal: NumberDecimal(
          (parseFloat(item.priceAtOrder) * item.quantity).toString()
        )
      });
    }

    return enriched;
  }

  async getAddress(addressId) {
    const address = await this.writeDb.collection('addresses')
      .findOne({ addressId });

    if (!address) return null;

    return {
      street: address.street,
      city: address.city,
      state: address.state,
      zipCode: address.zipCode,
      country: address.country
    };
  }

  buildStatusHistory(order) {
    // Construire historique depuis audit logs ou depuis ordre existant
    return [
      {
        status: order.status,
        timestamp: order.updatedAt
      }
    ];
  }

  buildSearchText(customer, items) {
    const parts = [
      customer.name,
      customer.email,
      ...items.map(i => i.product.name),
      ...items.map(i => i.product.sku),
      ...items.map(i => i.product.brand)
    ];

    return parts.join(' ').toLowerCase();
  }

  async projectProducts() {
    const changeStream = this.writeDb.collection('products').watch([
      {
        $match: {
          operationType: { $in: ['insert', 'update', 'replace'] }
        }
      }
    ], {
      fullDocument: 'updateLookup'
    });

    changeStream.on('change', async (change) => {
      try {
        await this.handleProductChange(change);
      } catch (error) {
        console.error('Failed to project product:', error);
        await this.sendToDeadLetter('products', change, error);
      }
    });

    console.log('Product projection started');
  }

  async handleProductChange(change) {
    const product = change.fullDocument;

    if (!product) return;

    // Enrichir avec métriques depuis orders
    const metrics = await this.calculateProductMetrics(product.productId);

    // Construire catalog read model
    const productCatalog = {
      productId: product.productId,

      name: product.name,
      displayName: product.displayName || product.name,
      description: product.description,

      sku: product.sku,
      brand: product.brand,
      category: product.category,

      price: product.price,
      currency: product.currency,

      images: product.images || [],
      features: product.features || {},

      metrics: {
        averageRating: metrics.averageRating,
        reviewCount: metrics.reviewCount,
        viewCount: metrics.viewCount,
        purchaseCount: metrics.purchaseCount,
        popularityScore: metrics.popularityScore,
        trendingScore: metrics.trendingScore
      },

      inStock: product.inventory.quantity > 0,
      stockLevel: product.inventory.quantity,

      slug: this.generateSlug(product.name),
      searchKeywords: this.extractKeywords(product),

      updatedAt: new Date()
    };

    await this.readDb.collection('product_catalog').replaceOne(
      { productId: product.productId },
      productCatalog,
      { upsert: true }
    );

    console.log(`Product ${product.productId} projected to catalog`);
  }

  async calculateProductMetrics(productId) {
    // Calculer métriques depuis write model
    const pipeline = [
      {
        $match: {
          'items.productId': productId,
          status: { $in: ['confirmed', 'shipped', 'delivered'] }
        }
      },
      {
        $unwind: '$items'
      },
      {
        $match: {
          'items.productId': productId
        }
      },
      {
        $group: {
          _id: null,
          purchaseCount: { $sum: 1 },
          totalQuantity: { $sum: '$items.quantity' }
        }
      }
    ];

    const result = await this.writeDb.collection('orders')
      .aggregate(pipeline)
      .toArray();

    const purchaseCount = result[0]?.purchaseCount || 0;

    // Autres métriques (simplifié)
    return {
      averageRating: 4.5,
      reviewCount: 100,
      viewCount: 1000,
      purchaseCount,
      popularityScore: Math.min(purchaseCount / 100, 1),
      trendingScore: Math.min(purchaseCount / 50, 1)
    };
  }

  generateSlug(name) {
    return name
      .toLowerCase()
      .replace(/[^a-z0-9]+/g, '-')
      .replace(/^-|-$/g, '');
  }

  extractKeywords(product) {
    const words = [
      product.name,
      product.brand,
      product.category,
      product.description
    ]
      .join(' ')
      .toLowerCase()
      .split(/\s+/)
      .filter(w => w.length > 3);

    return [...new Set(words)];
  }

  async projectAnalytics() {
    const changeStream = this.writeDb.collection('orders').watch([
      {
        $match: {
          operationType: 'insert'
        }
      }
    ], {
      fullDocument: 'updateLookup'
    });

    changeStream.on('change', async (change) => {
      try {
        await this.updateAnalytics(change.fullDocument);
      } catch (error) {
        console.error('Failed to update analytics:', error);
      }
    });

    console.log('Analytics projection started');
  }

  async updateAnalytics(order) {
    const date = this.getStartOfDay(order.createdAt);

    // Update daily summary
    await this.readDb.collection('orders_summary').updateOne(
      { date },
      {
        $inc: {
          'metrics.orderCount': 1,
          'metrics.totalRevenue': parseFloat(order.total)
        },
        $set: {
          updatedAt: new Date()
        }
      },
      { upsert: true }
    );
  }

  getStartOfDay(date) {
    const start = new Date(date);
    start.setHours(0, 0, 0, 0);
    return start;
  }

  async sendToDeadLetter(collection, change, error) {
    await this.readDb.collection('projection_failures').insertOne({
      collection,
      change,
      error: error.message,
      timestamp: new Date()
    });
  }
}
```

### 2. Batch Projection (Rebuild)

```javascript
class ProjectionRebuilder {
  constructor(writeDb, readDb) {
    this.writeDb = writeDb;
    this.readDb = readDb;
  }

  async rebuildOrdersReadModel() {
    console.log('Rebuilding orders read model...');

    // Supprimer ancien read model
    await this.readDb.collection('orders_read').deleteMany({});

    // Streamer tous les orders depuis write model
    const cursor = this.writeDb.collection('orders')
      .find({})
      .sort({ createdAt: 1 });

    let count = 0;
    const batchSize = 100;
    let batch = [];

    await cursor.forEach(async (order) => {
      // Enrichir
      const customer = await this.writeDb.collection('customers')
        .findOne({ customerId: order.customerId });

      if (!customer) return;

      const enrichedItems = await this.enrichOrderItems(order.items);
      const [shippingAddress, billingAddress] = await Promise.all([
        this.getAddress(order.shippingAddressId),
        this.getAddress(order.billingAddressId)
      ]);

      const orderRead = {
        orderId: order.orderId,
        customer: {
          customerId: customer.customerId,
          name: customer.name,
          email: customer.email
        },
        items: enrichedItems,
        subtotal: order.subtotal,
        tax: order.tax,
        shipping: order.shipping,
        total: order.total,
        currency: order.currency,
        status: order.status,
        shippingAddress,
        billingAddress,
        createdAt: order.createdAt,
        updatedAt: order.updatedAt,
        lastSyncedFrom: {
          writeModelVersion: order.version,
          syncedAt: new Date()
        }
      };

      batch.push(orderRead);

      if (batch.length >= batchSize) {
        await this.readDb.collection('orders_read')
          .insertMany(batch, { ordered: false });

        count += batch.length;
        batch = [];

        console.log(`Rebuilt ${count} orders`);
      }
    });

    // Flush dernier batch
    if (batch.length > 0) {
      await this.readDb.collection('orders_read')
        .insertMany(batch, { ordered: false });
      count += batch.length;
    }

    console.log(`Rebuild completed: ${count} orders`);
  }

  async enrichOrderItems(items) {
    // Même logique que ProjectionBuilder
    const enriched = [];

    for (const item of items) {
      const product = await this.writeDb.collection('products')
        .findOne({ productId: item.productId });

      if (product) {
        enriched.push({
          productId: item.productId,
          product: {
            name: product.name,
            sku: product.sku
          },
          quantity: item.quantity,
          unitPrice: item.priceAtOrder,
          subtotal: NumberDecimal(
            (parseFloat(item.priceAtOrder) * item.quantity).toString()
          )
        });
      }
    }

    return enriched;
  }

  async getAddress(addressId) {
    const address = await this.writeDb.collection('addresses')
      .findOne({ addressId });

    return address ? {
      street: address.street,
      city: address.city,
      state: address.state,
      zipCode: address.zipCode,
      country: address.country
    } : null;
  }
}
```

## API Layer (REST)

```javascript
class CQRSAPIController {
  constructor(commandHandler, queryService, validator) {
    this.commandHandler = commandHandler;
    this.queryService = queryService;
    this.validator = validator;
  }

  // Command endpoints (write)

  async createOrder(req, res) {
    try {
      // Validate command
      await this.validator.validateCreateOrder(req.body);

      // Execute command
      const result = await this.commandHandler.createOrder({
        customerId: req.user.customerId,
        items: req.body.items,
        shippingAddressId: req.body.shippingAddressId,
        billingAddressId: req.body.billingAddressId
      });

      if (result.success) {
        // 202 Accepted (eventual consistency)
        res.status(202).json({
          orderId: result.orderId,
          message: 'Order created successfully'
        });
      } else {
        res.status(400).json({ error: result.error });
      }

    } catch (error) {
      this.handleError(res, error);
    }
  }

  async updateOrderStatus(req, res) {
    try {
      await this.validator.validateUpdateOrderStatus(req.body);

      const result = await this.commandHandler.updateOrderStatus({
        orderId: req.params.orderId,
        newStatus: req.body.status,
        expectedVersion: req.body.version
      });

      if (result.success) {
        res.status(202).json({
          orderId: result.orderId,
          version: result.version,
          message: 'Order status updated'
        });
      } else {
        res.status(400).json({ error: result.error });
      }

    } catch (error) {
      this.handleError(res, error);
    }
  }

  // Query endpoints (read)

  async getOrder(req, res) {
    try {
      const order = await this.queryService.getOrder(
        req.params.orderId
      );

      if (order) {
        res.json(order);
      } else {
        res.status(404).json({ error: 'Order not found' });
      }

    } catch (error) {
      this.handleError(res, error);
    }
  }

  async getCustomerOrders(req, res) {
    try {
      const result = await this.queryService.getCustomerOrders(
        req.params.customerId,
        {
          status: req.query.status,
          fromDate: req.query.fromDate,
          toDate: req.query.toDate,
          limit: parseInt(req.query.limit) || 20,
          offset: parseInt(req.query.offset) || 0
        }
      );

      res.json(result);

    } catch (error) {
      this.handleError(res, error);
    }
  }

  async searchOrders(req, res) {
    try {
      const orders = await this.queryService.searchOrders(
        req.query.q,
        {
          status: req.query.status,
          minTotal: req.query.minTotal,
          maxTotal: req.query.maxTotal
        }
      );

      res.json({ orders });

    } catch (error) {
      this.handleError(res, error);
    }
  }

  handleError(res, error) {
    console.error('API error:', error);

    if (error instanceof ValidationError) {
      res.status(400).json({ errors: error.errors });
    } else {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
}

// Routes Express
function setupRoutes(app, controller) {
  // Command routes (POST, PUT, DELETE)
  app.post('/api/orders',
    authenticate,
    controller.createOrder.bind(controller)
  );

  app.put('/api/orders/:orderId/status',
    authenticate,
    authorize('admin'),
    controller.updateOrderStatus.bind(controller)
  );

  // Query routes (GET)
  app.get('/api/orders/:orderId',
    authenticate,
    controller.getOrder.bind(controller)
  );

  app.get('/api/customers/:customerId/orders',
    authenticate,
    controller.getCustomerOrders.bind(controller)
  );

  app.get('/api/orders/search',
    authenticate,
    controller.searchOrders.bind(controller)
  );
}
```

## Performance et optimisation

### 1. Configuration MongoDB pour CQRS

```javascript
// Write Database (optimisé pour writes)
const writeDbConfig = {
  writeConcern: {
    w: 'majority',
    j: true,
    wtimeout: 5000
  },

  readConcern: {
    level: 'majority'
  },

  // Moins d'index (juste essentiels)
  // Transactions activées
};

// Read Database (optimisé pour reads)
const readDbConfig = {
  writeConcern: {
    w: 1,  // Peut être relaxé
    j: false
  },

  readConcern: {
    level: 'local'  // Eventual consistency ok
  },

  readPreference: 'secondary',  // Offload reads

  // Nombreux index pour queries optimisées
  // Pas de transactions nécessaires
};

// Séparation physique possible
const writeConnection = await MongoClient.connect(
  'mongodb://write-cluster:27017/app',
  writeDbConfig
);

const readConnection = await MongoClient.connect(
  'mongodb://read-cluster:27017/app_read',
  readDbConfig
);
```

### 2. Caching Strategy

```javascript
class CQRSCacheStrategy {
  constructor(redis) {
    this.redis = redis;
  }

  // Cache uniquement read side
  async getCachedOrder(orderId) {
    const key = `order:${orderId}`;
    const cached = await this.redis.get(key);

    if (cached) {
      return JSON.parse(cached);
    }

    return null;
  }

  async cacheOrder(order) {
    const key = `order:${order.orderId}`;

    // TTL basé sur status
    let ttl = 300;  // 5 minutes par défaut

    if (order.status === 'delivered') {
      ttl = 3600;  // 1 heure (moins volatile)
    } else if (order.status === 'pending') {
      ttl = 60;  // 1 minute (très volatile)
    }

    await this.redis.setex(key, ttl, JSON.stringify(order));
  }

  async invalidateOrder(orderId) {
    await this.redis.del(`order:${orderId}`);
  }

  // Invalidation sur write
  async onOrderUpdated(orderId) {
    await this.invalidateOrder(orderId);

    // Invalider aussi listes associées
    const order = await this.getCachedOrder(orderId);
    if (order) {
      await this.redis.del(`customer:${order.customer.customerId}:orders`);
    }
  }
}
```

### 3. Monitoring Projection Lag

```javascript
class ProjectionLagMonitor {
  constructor(writeDb, readDb) {
    this.writeDb = writeDb;
    this.readDb = readDb;
  }

  async measureLag() {
    // Récupérer dernière modification write model
    const latestWrite = await this.writeDb.collection('orders')
      .findOne(
        {},
        {
          sort: { updatedAt: -1 },
          projection: { updatedAt: 1, orderId: 1, version: 1 }
        }
      );

    if (!latestWrite) return null;

    // Récupérer projection correspondante
    const projection = await this.readDb.collection('orders_read')
      .findOne(
        { orderId: latestWrite.orderId },
        { projection: { lastSyncedFrom: 1 } }
      );

    if (!projection) {
      return {
        orderId: latestWrite.orderId,
        lag: null,
        status: 'not_projected'
      };
    }

    // Calculer lag
    const lag = projection.lastSyncedFrom.syncedAt - latestWrite.updatedAt;

    return {
      orderId: latestWrite.orderId,
      lag: Math.abs(lag),  // ms
      writeVersion: latestWrite.version,
      projectedVersion: projection.lastSyncedFrom.writeModelVersion,
      status: latestWrite.version === projection.lastSyncedFrom.writeModelVersion
        ? 'synced'
        : 'lagging'
    };
  }

  async startMonitoring() {
    setInterval(async () => {
      const lag = await this.measureLag();

      if (lag) {
        console.log(`Projection lag: ${lag.lag}ms (${lag.status})`);

        // Alert si lag trop important
        if (lag.lag > 10000) {  // 10 secondes
          console.error('ALERT: High projection lag detected!');
          await this.sendAlert(lag);
        }
      }
    }, 30000);  // Toutes les 30 secondes
  }

  async sendAlert(lag) {
    // Envoyer alerte (PagerDuty, Slack, etc.)
    console.error('Projection lag alert:', lag);
  }
}
```

### 4. Métriques de performance

```javascript
const cqrsMetrics = {
  // Write side
  'commands.execution.latency.p99': {
    description: 'Command execution latency p99',
    target: 100,  // ms
    alert_threshold: 500
  },

  'commands.throughput': {
    description: 'Commands per second',
    target: 500,
    alert_threshold: 50
  },

  'commands.failure_rate': {
    description: 'Command failure rate',
    target: 0.01,
    alert_threshold: 0.05
  },

  // Read side
  'queries.latency.p99': {
    description: 'Query latency p99',
    target: 50,  // ms
    alert_threshold: 200
  },

  'queries.cache_hit_rate': {
    description: 'Cache hit rate',
    target: 0.90,
    alert_threshold: 0.70
  },

  // Projection
  'projection.lag': {
    description: 'Projection lag (ms)',
    target: 1000,
    alert_threshold: 10000
  },

  'projection.throughput': {
    description: 'Events projected per second',
    target: 1000,
    alert_threshold: 100
  },

  'projection.failure_rate': {
    description: 'Projection failure rate',
    target: 0,
    alert_threshold: 0.01
  }
};
```

## Checklist de déploiement

### ✅ Architecture

- [ ] Write et Read models définis
- [ ] Séparation command/query APIs
- [ ] Projection builder implémenté
- [ ] Event bus (si utilisé)
- [ ] Cache layer configuré

### ✅ Write Model

- [ ] Schema normalisé
- [ ] Transactions ACID
- [ ] Optimistic locking
- [ ] Business logic isolation
- [ ] Validation complète
- [ ] Write concern strict

### ✅ Read Model

- [ ] Schema dénormalisé
- [ ] Index optimisés pour queries
- [ ] Multiple projections si nécessaire
- [ ] TTL pour données temporaires
- [ ] Text indexes pour recherche

### ✅ Synchronisation

- [ ] Change Streams configuré
- [ ] Projection handlers robustes
- [ ] Idempotence garantie
- [ ] Error handling et retry
- [ ] Dead letter queue
- [ ] Rebuild capability

### ✅ Performance

- [ ] Caching strategy
- [ ] Read replicas si nécessaire
- [ ] Séparation physique write/read clusters
- [ ] Index covering queries
- [ ] Query optimization

### ✅ Monitoring

- [ ] Projection lag tracking
- [ ] Command/query latency
- [ ] Failure rates
- [ ] Cache hit rates
- [ ] Alerting configuré

### ✅ Operations

- [ ] Projection rebuild procedures
- [ ] Data migration strategy
- [ ] Backup strategy
- [ ] Disaster recovery
- [ ] Documentation complète

## Conclusion

CQRS avec MongoDB offre :

**✅ Forces démontrées :**
- Performance optimale (read et write séparés)
- Scalabilité indépendante
- Schemas optimisés pour chaque usage
- Change Streams pour synchronisation temps réel
- Multiple read models depuis même write model
- Flexibility pour évolution
- Queries complexes facilitées
- Security (permissions séparées)

**⚠️ Considérations importantes :**
- Complexité accrue vs CRUD simple
- Eventual consistency à gérer
- Duplication de données
- Synchronisation à maintenir
- Testing plus complexe
- Courbe d'apprentissage

**🎯 Patterns essentiels CQRS :**
1. **Séparation Command/Query** stricte
2. **Write model normalisé** pour consistency
3. **Read models dénormalisés** pour performance
4. **Change Streams** pour synchronisation
5. **Projections idempotentes** pour resilience
6. **Caching** côté read
7. **Monitoring lag** pour health

**Quand utiliser CQRS :**
- ✅ Reads >> Writes (ratio important)
- ✅ Queries complexes nécessaires
- ✅ Multiple vues des données
- ✅ Performance critique
- ✅ Scalabilité indépendante requise
- ✅ Domaine complexe
- ❌ Simple CRUD applications
- ❌ Équipe petite/inexperimentée
- ❌ Strong consistency obligatoire partout

**Combinaison avec Event Sourcing :**
CQRS se combine excellemment avec Event Sourcing (section 20.11) :
- Write side = Event Store
- Read side = Projections depuis events
- Audit trail complet
- Time travel capability
- Maximum flexibility

CQRS avec MongoDB est idéal pour applications nécessitant haute performance, scalabilité indépendante, et queries complexes, tout en maintenant consistency côté write.

---

**Références :**
- "Domain-Driven Design" - Eric Evans
- "Implementing Domain-Driven Design" - Vaughn Vernon
- "CQRS Journey" - Microsoft patterns & practices
- Martin Fowler's CQRS Article
- MongoDB Change Streams Documentation
- Greg Young's CQRS Documents

⏭️ [Bonnes Pratiques et Anti-patterns](/21-bonnes-pratiques-anti-patterns/README.md)
