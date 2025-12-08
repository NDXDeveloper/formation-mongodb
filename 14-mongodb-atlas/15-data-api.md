🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.15 Data API

## Introduction

**Atlas Data API** expose votre base MongoDB via des **endpoints REST et GraphQL standards**, générés automatiquement. Accédez à vos données depuis n'importe quel client HTTP : applications web, mobiles, IoT, serverless functions, ou services externes. Pas besoin de driver MongoDB ni de gérer des connexions : un simple appel HTTPS suffit. C'est la solution idéale pour l'intégration rapide, les architectures microservices, et l'exposition contrôlée de données.

### 🎯 Objectifs de cette Section

- Maîtriser l'architecture REST et GraphQL de Data API
- Configurer l'authentication et les permissions
- Implémenter rate limiting et quotas
- Créer des custom endpoints avancés
- Optimiser performances avec caching
- Sécuriser l'exposition de données
- Monitorer et observer les appels API
- Intégrer avec divers clients et frameworks

---

## 🏗️ Architecture Data API

### Vue d'Ensemble

```
┌────────────────────────────────────────────────────────────────────────┐
│                     ATLAS DATA API ARCHITECTURE
├────────────────────────────────────────────────────────────────────────┤
│
│  CLIENTS (Anywhere with HTTP)
│  ┌──────────────────────────────────────────────────────────────────┐
│  │  Web Apps  │  Mobile Apps  │  IoT Devices  │  Serverless
│  │  React/Vue │  iOS/Android  │  Arduino/Pi   │  Lambda/Vercel
│  └────┬───────────┬──────────────┬────────────────┬─────────────────
│       │           │              │                │
│       │           │              │                │
│       └───────────┴──────────────┴────────────────┘
│                   │
│                   ▼ HTTPS Requests
│  ┌──────────────────────────────────────────────────────────────────┐
│  │              ATLAS DATA API GATEWAY
│  │  https://data.mongodb-api.com/app/{app-id}/endpoint/data/v1
│  ├──────────────────────────────────────────────────────────────────┤
│  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  │  │ AUTHENTICATION  │  │  AUTHORIZATION  │  │  RATE LIMITING  │
│  │  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│  │  │ • API Key       │  │ • Rules         │  │ • Per endpoint  │
│  │  │ • JWT Token     │  │ • Roles         │  │ • Per user      │
│  │  │ • Email/Pass    │  │ • Row-level     │  │ • Per IP        │
│  │  │ • Custom        │  │ • Field-level   │  │ • Global limits │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘
│  │
│  │  ┌──────────────────────────────────────────────────────────────┐
│  │  │                    ROUTING LAYER
│  │  │  • REST endpoints:   /action/{operation}                     │
│  │  │  • GraphQL endpoint: /graphql                                │
│  │  │  • Custom endpoints: /custom/{name}                          │
│  │  └──────────────────────────────────────────────────────────────┘
│  │
│  │  ┌──────────────────────────────────────────────────────────────┐
│  │  │                 QUERY OPTIMIZATION
│  │  │  • Query validation
│  │  │  • Index hints
│  │  │  • Projection optimization
│  │  │  • Result caching
│  │  └──────────────────────────────────────────────────────────────┘
│  └───────────────────────────────┬───────────────────────────────────
│                                  │
│                                  ▼
│  MONGODB ATLAS CLUSTER
│  ┌──────────────────────────────────────────────────────────────────┐
│  │  Production Data
│  │  • CRUD operations via API
│  │  • Aggregation pipelines
│  │  • Transactions support
│  └──────────────────────────────────────────────────────────────────┘
│
│  KEY FEATURES:
│  ✅ Auto-generated endpoints (no code)
│  ✅ HTTPS only (secure by default)
│  ✅ Rate limiting built-in
│  ✅ Global CDN (low latency)
│  ✅ OpenAPI specification
│  ✅ CORS configuration
│
└────────────────────────────────────────────────────────────────────────┘
```

### REST vs GraphQL

```
┌───────────────────────────────────────────────────────────────────────┐
│                    REST API vs GRAPHQL API                            │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  REST API                                                             │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Endpoints:                                                       │ │
│  │ POST /action/findOne       - Find single document                │ │
│  │ POST /action/find          - Find multiple documents             │ │
│  │ POST /action/insertOne     - Insert document                     │ │
│  │ POST /action/insertMany    - Insert multiple                     │ │
│  │ POST /action/updateOne     - Update document                     │ │
│  │ POST /action/updateMany    - Update multiple                     │ │
│  │ POST /action/deleteOne     - Delete document                     │ │
│  │ POST /action/deleteMany    - Delete multiple                     │ │
│  │ POST /action/aggregate     - Run aggregation                     │ │
│  │                                                                  │ │
│  │ Avantages:                                                       │ │
│  │ ✅ Simple à comprendre                                           │ │
│  │ ✅ Compatible avec tous les clients HTTP                         │ │
│  │ ✅ Facilité de debug (curl, Postman)                             │ │
│  │ ✅ Caching HTTP standard                                         │ │
│  │                                                                  │ │
│  │ Inconvénients:                                                   │ │
│  │ ⚠️ Multiple requests pour données liées                          │ │
│  │ ⚠️ Over-fetching (plus de données que nécessaire)                │ │
│  │ ⚠️ Under-fetching (pas assez, requêtes supplémentaires)          │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  GRAPHQL API                                                          │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Single endpoint: POST /graphql                                   │ │
│  │                                                                  │ │
│  │ Query: Flexible data fetching                                    │ │
│  │ query {                                                          │ │
│  │   users(query: { status: "active" }) {                           │ │
│  │     _id                                                          │ │
│  │     name                                                         │ │
│  │     orders {                                                     │ │
│  │       _id                                                        │ │
│  │       total                                                      │ │
│  │     }                                                            │ │
│  │   }                                                              │ │
│  │ }                                                                │ │
│  │                                                                  │ │
│  │ Avantages:                                                       │ │
│  │ ✅ Fetch exact data needed (no over/under fetching)              │ │
│  │ ✅ Single request for complex queries                            │ │
│  │ ✅ Strong typing (schema)                                        │ │
│  │ ✅ Introspection (auto-documentation)                            │ │
│  │                                                                  │ │
│  │ Inconvénients:                                                   │ │
│  │ ⚠️ Courbe d'apprentissage plus élevée                            │ │
│  │ ⚠️ Caching plus complexe                                         │ │
│  │ ⚠️ Peut exposer trop de schéma                                   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  RECOMMANDATION:                                                      │
│  • REST: Simple CRUD, compatibilité maximale, caching important       │
│  • GraphQL: Données complexes/liées, frontend moderne (React/Vue)     │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Authentication et Sécurité

### Méthodes d'Authentication

```javascript
// 1. API KEY (Recommandé pour services)
// Simple, stateless, révocable

const API_KEY = "your-api-key-here";

const response = await fetch(
  "https://data.mongodb-api.com/app/myapp-xxxxx/endpoint/data/v1/action/find",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "api-key": API_KEY
    },
    body: JSON.stringify({
      dataSource: "mongodb-atlas",
      database: "mydb",
      collection: "users",
      filter: { status: "active" }
    })
  }
);

const data = await response.json();

// 2. JWT TOKEN (Pour utilisateurs authentifiés)
// Custom JWT de votre système auth

const JWT_TOKEN = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...";

const response = await fetch(
  "https://data.mongodb-api.com/app/myapp-xxxxx/endpoint/data/v1/action/find",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "jwtTokenString": JWT_TOKEN
    },
    body: JSON.stringify({
      // ... query
    })
  }
);

// 3. EMAIL/PASSWORD (Login endpoint)
// Pour applications avec auth built-in

// Login first
const loginResponse = await fetch(
  "https://realm.mongodb.com/api/client/v2.0/app/myapp-xxxxx/auth/providers/local-userpass/login",
  {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      username: "user@example.com",
      password: "password123"
    })
  }
);

const { access_token } = await loginResponse.json();

// Use access token for subsequent requests
const response = await fetch(
  "https://data.mongodb-api.com/app/myapp-xxxxx/endpoint/data/v1/action/find",
  {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${access_token}`,
      "Content-Type": "application/json"
    },
    body: JSON.stringify({
      // ... query
    })
  }
);
```

### Rules et Permissions

```javascript
// Configuration des règles d'accès (Atlas UI)

// Collection: orders
// Rule: Users can only access their own orders
{
  "name": "userCanReadOwnOrders",
  "when": {
    // Apply this rule when...
    "%%true": true
  },
  "read": {
    // Allow read if userId matches authenticated user
    "userId": "%%user.id"
  },
  "write": {
    // Allow write if userId matches authenticated user
    "userId": "%%user.id"
  }
}

// Collection: products
// Rule: Everyone can read, only admins can write
{
  "name": "publicReadAdminWrite",
  "when": {
    "%%true": true
  },
  "read": true,  // Everyone can read
  "write": {
    // Only users with role "admin" can write
    "%%user.custom_data.role": "admin"
  }
}

// Collection: analytics
// Rule: Field-level permissions
{
  "name": "restrictedFields",
  "when": {
    "%%true": true
  },
  "read": {
    // Public users see limited fields
    "%%user.custom_data.role": {
      "$ne": "admin"
    }
  },
  "fields": {
    // Hide sensitive fields for non-admins
    "internalNotes": {
      "read": {
        "%%user.custom_data.role": "admin"
      }
    },
    "costData": {
      "read": {
        "%%user.custom_data.role": "admin"
      }
    }
  }
}
```

---

## 📡 REST API Avancé

### Requêtes Complexes

```javascript
// 1. FIND avec projection et sorting
const response = await fetch(baseUrl + "/action/find", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "api-key": API_KEY
  },
  body: JSON.stringify({
    dataSource: "mongodb-atlas",
    database: "mydb",
    collection: "products",

    // Filter
    filter: {
      category: "electronics",
      price: { $gte: 100, $lte: 1000 },
      inStock: true
    },

    // Projection (only return these fields)
    projection: {
      _id: 1,
      name: 1,
      price: 1,
      rating: 1
    },

    // Sorting
    sort: { price: 1 },  // Ascending by price

    // Pagination
    limit: 20,
    skip: 0
  })
});

// 2. AGGREGATION (Complex analytics)
const response = await fetch(baseUrl + "/action/aggregate", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "api-key": API_KEY
  },
  body: JSON.stringify({
    dataSource: "mongodb-atlas",
    database: "mydb",
    collection: "orders",

    pipeline: [
      {
        $match: {
          createdAt: {
            $gte: { $date: "2024-01-01T00:00:00Z" },
            $lte: { $date: "2024-12-31T23:59:59Z" }
          },
          status: "completed"
        }
      },
      {
        $group: {
          _id: {
            year: { $year: "$createdAt" },
            month: { $month: "$createdAt" }
          },
          totalRevenue: { $sum: "$total" },
          orderCount: { $sum: 1 },
          avgOrderValue: { $avg: "$total" }
        }
      },
      {
        $sort: { "_id.year": 1, "_id.month": 1 }
      }
    ]
  })
});

// 3. UPDATE avec opérateurs MongoDB
const response = await fetch(baseUrl + "/action/updateOne", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "api-key": API_KEY
  },
  body: JSON.stringify({
    dataSource: "mongodb-atlas",
    database: "mydb",
    collection: "users",

    filter: {
      _id: { $oid: "507f1f77bcf86cd799439011" }
    },

    update: {
      $set: {
        lastLogin: { $date: new Date().toISOString() }
      },
      $inc: {
        loginCount: 1
      },
      $push: {
        loginHistory: {
          timestamp: { $date: new Date().toISOString() },
          ipAddress: "192.168.1.1"
        }
      }
    }
  })
});
```

### Custom Endpoints

```javascript
// Custom Endpoint: Plus flexible que les endpoints générés

// Configuration (Atlas UI)
{
  "route": "/products/search",
  "http_method": "POST",
  "function_name": "searchProducts",
  "validation_method": "NO_VALIDATION",
  "respond_result": true,
  "fetch_custom_user_data": false,
  "create_user_on_auth": false,
  "disabled": false
}

// Function: searchProducts
exports = async function({ query, body }) {
  const { searchTerm, category, minPrice, maxPrice } = body;

  const products = context.services.get("mongodb-atlas")
    .db("mydb")
    .collection("products");

  // Build dynamic filter
  const filter = {};

  if (searchTerm) {
    filter.$text = { $search: searchTerm };
  }

  if (category) {
    filter.category = category;
  }

  if (minPrice || maxPrice) {
    filter.price = {};
    if (minPrice) filter.price.$gte = minPrice;
    if (maxPrice) filter.price.$lte = maxPrice;
  }

  // Execute query
  const results = await products
    .find(filter)
    .sort({ _id: -1 })
    .limit(20)
    .toArray();

  // Add computed field
  const enrichedResults = results.map(product => ({
    ...product,
    discountedPrice: product.price * 0.9,  // 10% discount
    inStockLabel: product.stock > 0 ? "Available" : "Out of stock"
  }));

  return {
    success: true,
    count: enrichedResults.length,
    products: enrichedResults
  };
};

// Usage from client
const response = await fetch(
  "https://data.mongodb-api.com/app/myapp-xxxxx/endpoint/products/search",
  {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "api-key": API_KEY
    },
    body: JSON.stringify({
      searchTerm: "laptop",
      category: "electronics",
      minPrice: 500,
      maxPrice: 2000
    })
  }
);

const data = await response.json();
```

---

## 🌐 GraphQL API

### Queries et Mutations

```graphql
# Schema auto-généré par Atlas

# QUERY: Fetch data
query GetProducts {
  products(
    query: {
      category: "electronics",
      price_gte: 100,
      price_lte: 1000
    }
    limit: 10
    sortBy: PRICE_ASC
  ) {
    _id
    name
    price
    category
    inStock
    rating
  }
}

# QUERY: With relationships
query GetUserWithOrders($userId: String!) {
  user(query: { _id: $userId }) {
    _id
    name
    email
    orders(limit: 5, sortBy: CREATEDAT_DESC) {
      _id
      total
      status
      createdAt
      items {
        productId
        quantity
        price
      }
    }
  }
}

# MUTATION: Insert document
mutation CreateProduct($product: ProductInsertInput!) {
  insertOneProduct(data: $product) {
    _id
    name
    price
  }
}

# Variables
{
  "product": {
    "name": "Wireless Mouse",
    "price": 29.99,
    "category": "electronics",
    "inStock": true,
    "stock": 150
  }
}

# MUTATION: Update document
mutation UpdateProduct($id: ObjectId!, $update: ProductUpdateInput!) {
  updateOneProduct(
    query: { _id: $id }
    set: $update
  ) {
    _id
    name
    price
    updatedAt
  }
}

# Variables
{
  "id": "507f1f77bcf86cd799439011",
  "update": {
    "price": 24.99,
    "updatedAt": "2024-12-08T15:30:00Z"
  }
}
```

### Client GraphQL (JavaScript)

```javascript
// GraphQL client avec Apollo

import { ApolloClient, InMemoryCache, HttpLink, gql } from '@apollo/client';

// Setup client
const client = new ApolloClient({
  link: new HttpLink({
    uri: 'https://data.mongodb-api.com/app/myapp-xxxxx/endpoint/graphql',
    headers: {
      'api-key': API_KEY
    }
  }),
  cache: new InMemoryCache()
});

// Query
const GET_PRODUCTS = gql`
  query GetProducts($category: String!) {
    products(query: { category: $category }, limit: 10) {
      _id
      name
      price
      inStock
    }
  }
`;

const { data } = await client.query({
  query: GET_PRODUCTS,
  variables: { category: "electronics" }
});

console.log(data.products);

// Mutation
const CREATE_ORDER = gql`
  mutation CreateOrder($order: OrderInsertInput!) {
    insertOneOrder(data: $order) {
      _id
      total
      status
    }
  }
`;

const { data } = await client.mutate({
  mutation: CREATE_ORDER,
  variables: {
    order: {
      userId: "user123",
      items: [
        { productId: "prod456", quantity: 2, price: 29.99 }
      ],
      total: 59.98,
      status: "pending"
    }
  }
});

console.log("Order created:", data.insertOneOrder._id);
```

---

## ⚡ Performance et Optimisation

### Rate Limiting

```
┌───────────────────────────────────────────────────────────────────────┐
│                         RATE LIMITING                                 │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  DEFAULT LIMITS (Free Tier)                                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 5,000 requests/month                                             │ │
│  │ 100 requests/minute per endpoint                                 │ │
│  │ 10 requests/second per IP                                        │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  PAID TIER LIMITS                                                     │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Unlimited requests/month                                         │ │
│  │ 1,000 requests/minute per endpoint                               │ │
│  │ 100 requests/second per IP                                       │ │
│  │ Custom limits configurable                                       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  HEADERS RETOURNÉS:                                                   │
│  X-RateLimit-Limit: 100          (max requests per window)            │
│  X-RateLimit-Remaining: 87       (remaining in current window)        │
│  X-RateLimit-Reset: 1702037400   (window reset timestamp)             │
│                                                                       │
│  HANDLING (Client-side):                                              │
│  • Check X-RateLimit-Remaining before burst requests                  │
│  • Implement exponential backoff on 429 responses                     │
│  • Cache responses agressivement                                      │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Caching Strategy

```javascript
// Client-side caching avec rate limit awareness

class DataAPIClient {
  constructor(baseUrl, apiKey) {
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
    this.cache = new Map();
    this.cacheTTL = 60000; // 60 seconds
  }

  getCacheKey(endpoint, body) {
    return `${endpoint}:${JSON.stringify(body)}`;
  }

  async request(endpoint, body, options = {}) {
    const cacheKey = this.getCacheKey(endpoint, body);

    // Check cache
    if (options.useCache !== false) {
      const cached = this.cache.get(cacheKey);
      if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
        console.log('Cache hit:', cacheKey);
        return cached.data;
      }
    }

    // Make request
    const response = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'api-key': this.apiKey
      },
      body: JSON.stringify(body)
    });

    // Check rate limit
    const remaining = parseInt(response.headers.get('X-RateLimit-Remaining'));
    const reset = parseInt(response.headers.get('X-RateLimit-Reset'));

    if (remaining < 10) {
      console.warn(`Low rate limit: ${remaining} requests remaining`);
      console.warn(`Resets at: ${new Date(reset * 1000).toISOString()}`);
    }

    // Handle 429 Too Many Requests
    if (response.status === 429) {
      const retryAfter = parseInt(response.headers.get('Retry-After') || '60');
      console.error(`Rate limited. Retry after ${retryAfter}s`);
      throw new Error(`Rate limited. Retry after ${retryAfter} seconds`);
    }

    const data = await response.json();

    // Cache successful response
    if (response.ok) {
      this.cache.set(cacheKey, {
        data,
        timestamp: Date.now()
      });
    }

    return data;
  }

  async find(collection, filter, options = {}) {
    return this.request('/action/find', {
      dataSource: 'mongodb-atlas',
      database: 'mydb',
      collection,
      filter,
      ...options
    }, { useCache: true });
  }
}

// Usage
const client = new DataAPIClient(BASE_URL, API_KEY);

// Cached for 60s
const products = await client.find('products', { category: 'electronics' });

// Subsequent call within 60s returns cached data
const cachedProducts = await client.find('products', { category: 'electronics' });
```

---

## 🔍 Monitoring et Observabilité

### Logging Requests

```javascript
// Middleware pour logging détaillé

class DataAPILogger {
  constructor(client) {
    this.client = client;
    this.logs = [];
  }

  async loggedRequest(endpoint, body) {
    const requestId = this.generateRequestId();
    const startTime = Date.now();

    const logEntry = {
      requestId,
      endpoint,
      body,
      timestamp: new Date().toISOString(),
      status: null,
      duration: null,
      error: null
    };

    console.log(`[${requestId}] Request started: ${endpoint}`);

    try {
      const response = await this.client.request(endpoint, body);

      logEntry.status = 'success';
      logEntry.duration = Date.now() - startTime;

      console.log(`[${requestId}] Success in ${logEntry.duration}ms`);

      this.logs.push(logEntry);
      return response;

    } catch (error) {
      logEntry.status = 'error';
      logEntry.duration = Date.now() - startTime;
      logEntry.error = error.message;

      console.error(`[${requestId}] Error in ${logEntry.duration}ms:`, error);

      this.logs.push(logEntry);
      throw error;
    }
  }

  generateRequestId() {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  getStats() {
    const total = this.logs.length;
    const successful = this.logs.filter(l => l.status === 'success').length;
    const failed = this.logs.filter(l => l.status === 'error').length;
    const avgDuration = this.logs.reduce((sum, l) => sum + l.duration, 0) / total;

    return {
      total,
      successful,
      failed,
      successRate: (successful / total * 100).toFixed(2) + '%',
      avgDuration: avgDuration.toFixed(2) + 'ms'
    };
  }
}

// Usage
const logger = new DataAPILogger(client);

await logger.loggedRequest('/action/find', {
  dataSource: 'mongodb-atlas',
  database: 'mydb',
  collection: 'users',
  filter: {}
});

console.log('Stats:', logger.getStats());
```

### Métriques Atlas

```javascript
// Collecter métriques pour monitoring

async function collectDataAPIMetrics() {
  const db = context.services.get("mongodb-atlas")
    .db("monitoring")
    .collection("api_metrics");

  // Métriques à tracker
  const metrics = {
    timestamp: new Date(),

    // Request metrics
    totalRequests: await getTotalRequests(),
    requestsByEndpoint: await getRequestsByEndpoint(),
    requestsByMethod: await getRequestsByMethod(),

    // Performance metrics
    avgResponseTime: await getAvgResponseTime(),
    p95ResponseTime: await getP95ResponseTime(),
    p99ResponseTime: await getP99ResponseTime(),

    // Error metrics
    errorRate: await getErrorRate(),
    errorsByType: await getErrorsByType(),

    // Rate limit metrics
    rateLimitHits: await getRateLimitHits(),

    // Authentication metrics
    authSuccessRate: await getAuthSuccessRate(),
    authMethodBreakdown: await getAuthMethodBreakdown()
  };

  await db.insertOne(metrics);

  return metrics;
}

// Schedule: Run every 5 minutes via scheduled trigger
```

---

## 📋 Best Practices

### Checklist Production

```
┌───────────────────────────────────────────────────────────────────────┐
│                DATA API PRODUCTION CHECKLIST                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  SÉCURITÉ                                                             │
│  ☐ HTTPS uniquement (no HTTP)                                         │
│  ☐ API keys sécurisées (environment variables, secrets manager)       │
│  ☐ Rules d'accès configurées (row-level, field-level)                 │
│  ☐ CORS configuré restrictif (pas de wildcard *)                      │
│  ☐ Rate limiting approprié                                            │
│  ☐ Input validation côté serveur (custom endpoints)                   │
│  ☐ Pas de données sensibles exposées                                  │
│                                                                       │
│  PERFORMANCE                                                          │
│  ☐ Indexes sur champs filtrés fréquemment                             │
│  ☐ Projection pour limiter champs retournés                           │
│  ☐ Pagination implémentée (limit + skip)                              │
│  ☐ Caching côté client (60s-300s selon use case)                      │
│  ☐ Batch operations quand possible                                    │
│  ☐ Éviter N+1 queries (utiliser aggregations)                         │
│                                                                       │
│  ERROR HANDLING                                                       │
│  ☐ Retry logic avec exponential backoff (429 errors)                  │
│  ☐ Graceful degradation (fallbacks)                                   │
│  ☐ Error messages user-friendly (pas de stack traces)                 │
│  ☐ Logging détaillé (request ID, timestamp, duration)                 │
│  ☐ Monitoring alertes (error rate > 5%)                               │
│                                                                       │
│  DÉVELOPPEMENT                                                        │
│  ☐ OpenAPI spec générée et documentée                                 │
│  ☐ Postman collection partagée avec équipe                            │
│  ☐ Environment variables (dev, staging, prod)                         │
│  ☐ Versioning API (v1, v2 endpoints)                                  │
│  ☐ Automated tests (integration tests)                                │
│                                                                       │
│  MONITORING                                                           │
│  ☐ Request metrics collectées                                         │
│  ☐ Error rate monitoré                                                │
│  ☐ Response time tracked (p50, p95, p99)                              │
│  ☐ Rate limit usage surveillé                                         │
│  ☐ Dashboards configurés (Grafana, DataDog)                           │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Patterns Anti-Patterns

```javascript
// ❌ ANTI-PATTERN 1: Exposer toute la collection sans filtre

// Bad: Returns ALL documents (security + performance issue)
const allUsers = await fetch(baseUrl + "/action/find", {
  method: "POST",
  body: JSON.stringify({
    dataSource: "mongodb-atlas",
    database: "mydb",
    collection: "users",
    filter: {}  // ❌ No filter!
  })
});

// ✅ GOOD: Always filter + paginate
const users = await fetch(baseUrl + "/action/find", {
  method: "POST",
  body: JSON.stringify({
    dataSource: "mongodb-atlas",
    database: "mydb",
    collection: "users",
    filter: { status: "active" },  // ✅ Filter
    limit: 20,                      // ✅ Limit
    skip: page * 20                 // ✅ Pagination
  })
});

// ❌ ANTI-PATTERN 2: Hardcoded API key in frontend

// Bad: API key in client-side code
const API_KEY = "xvDH2lLkjhg...";  // ❌ Exposed!

// ✅ GOOD: API key on backend, use JWT for frontend
// Backend exposes authenticated endpoint
app.post("/api/products", authenticateUser, async (req, res) => {
  const response = await fetch(dataApiUrl, {
    headers: { "api-key": process.env.DATA_API_KEY }  // ✅ Server-side
  });
  res.json(await response.json());
});

// ❌ ANTI-PATTERN 3: No error handling

// Bad: No retry, no fallback
const data = await fetch(url).then(r => r.json());

// ✅ GOOD: Comprehensive error handling
async function fetchWithRetry(url, options, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      const response = await fetch(url, options);

      if (response.status === 429) {
        const retryAfter = parseInt(response.headers.get('Retry-After') || '60');
        await sleep(retryAfter * 1000);
        continue;
      }

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}`);
      }

      return await response.json();

    } catch (error) {
      if (i === maxRetries - 1) throw error;
      await sleep(Math.pow(2, i) * 1000);  // Exponential backoff
    }
  }
}
```

---

## 🏁 Résumé

### Points Clés

1. **Architecture**
   - REST et GraphQL endpoints auto-générés
   - HTTPS only, secure by default
   - Global CDN, low latency
   - Rate limiting built-in

2. **Authentication**
   - API Key (services)
   - JWT Token (users)
   - Email/Password (built-in)
   - Row-level & field-level permissions

3. **REST API**
   - 9 operations (find, insert, update, delete, aggregate)
   - Custom endpoints (flexible)
   - Projection, sorting, pagination
   - Complex aggregations

4. **GraphQL API**
   - Single endpoint
   - Flexible queries
   - Strong typing
   - Auto-generated schema

5. **Performance**
   - Rate limiting awareness
   - Client-side caching
   - Index optimization
   - Batch operations

### Configuration Minimale

```javascript
// REST API client production-ready
class ProductionDataAPIClient {
  constructor(config) {
    this.baseUrl = config.baseUrl;
    this.apiKey = config.apiKey;
    this.cache = new Map();
    this.cacheTTL = config.cacheTTL || 60000;
  }

  async request(endpoint, body, options = {}) {
    // Cache check
    const cacheKey = `${endpoint}:${JSON.stringify(body)}`;
    if (options.useCache) {
      const cached = this.cache.get(cacheKey);
      if (cached && Date.now() - cached.timestamp < this.cacheTTL) {
        return cached.data;
      }
    }

    // Request with retry
    const response = await this.fetchWithRetry(
      `${this.baseUrl}${endpoint}`,
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'api-key': this.apiKey
        },
        body: JSON.stringify(body)
      }
    );

    const data = await response.json();

    // Cache
    if (options.useCache) {
      this.cache.set(cacheKey, { data, timestamp: Date.now() });
    }

    return data;
  }

  async fetchWithRetry(url, options, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
      try {
        const response = await fetch(url, options);

        if (response.status === 429) {
          const retryAfter = parseInt(response.headers.get('Retry-After') || '60');
          await this.sleep(retryAfter * 1000);
          continue;
        }

        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        return response;

      } catch (error) {
        if (i === maxRetries - 1) throw error;
        await this.sleep(Math.pow(2, i) * 1000);
      }
    }
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// Usage
const client = new ProductionDataAPIClient({
  baseUrl: 'https://data.mongodb-api.com/app/myapp-xxxxx/endpoint/data/v1',
  apiKey: process.env.DATA_API_KEY,
  cacheTTL: 60000
});
```

### Ressources

- [Data API Documentation](https://www.mongodb.com/docs/atlas/app-services/data-api/)
- [OpenAPI Specification](https://www.mongodb.com/docs/atlas/app-services/data-api/openapi/)
- [GraphQL Reference](https://www.mongodb.com/docs/atlas/app-services/graphql/)

---


⏭️ [Atlas CLI](/14-mongodb-atlas/16-atlas-cli.md)
