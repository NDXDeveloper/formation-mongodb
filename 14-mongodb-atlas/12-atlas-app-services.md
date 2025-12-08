🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.12 Atlas App Services

## Introduction

**Atlas App Services** (anciennement MongoDB Realm) est le **Backend-as-a-Service (BaaS)** de MongoDB qui permet de construire des applications complètes sans gérer de serveurs. Authentication, functions serverless, triggers automatiques, API REST/GraphQL générée, synchronisation mobile : tout est intégré et scalable automatiquement. C'est la solution idéale pour les applications mobiles, les prototypes rapides, et les architectures serverless.

### 🎯 Objectifs de cette Section

- Comprendre l'architecture App Services
- Configurer l'authentification multi-provider
- Créer des functions serverless
- Implémenter des triggers automatiques
- Utiliser le Data API (REST/GraphQL)
- Gérer la synchronisation offline (Device Sync)
- Sécuriser avec les règles d'accès

---

## 🏗️ Architecture App Services

### Vue d'Ensemble

```
┌────────────────────────────────────────────────────────────────────────┐
│                   ATLAS APP SERVICES ARCHITECTURE                      │
├────────────────────────────────────────────────────────────────────────┤
│
│   CLIENTS
│   ┌──────────────────────────────────────────────────────────────────┐
│   │  Web App  │  Mobile App  │  IoT Device  │  External Service      │
│   │  (React)  │  (iOS/Android)│  (Raspberry)│  (Webhook)             │
│   └────┬──────────┬──────────────┬────────────────┬──────────────────┘
│        │          │              │                │
│        │          │              │                │
│        ▼          ▼              ▼                ▼
│   ┌──────────────────────────────────────────────────────────────────┐
│   │            ATLAS APP SERVICES (Serverless Layer)                 │
│   ├──────────────────────────────────────────────────────────────────┤
│   │
│   │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   │  │ AUTHENTICATION  │  │   FUNCTIONS     │  │    TRIGGERS     │
│   │  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│   │  │ • Email/Pass    │  │ • Serverless    │  │ • Database      │
│   │  │ • Google OAuth  │  │ • JavaScript    │  │ • Scheduled     │
│   │  │ • Apple         │  │ • Node.js       │  │ • Auth events   │
│   │  │ • Anonymous     │  │ • HTTP invoke   │  │ • Webhooks      │
│   │  │ • API Key       │  │ • Auto-scale    │  │                 │
│   │  │ • Custom JWT    │  │                 │  │                 │
│   │  └─────────────────┘  └─────────────────┘  └─────────────────┘
│   │
│   │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   │  │   DATA API      │  │  DEVICE SYNC    │  │  ACCESS RULES   │
│   │  ├─────────────────┤  ├─────────────────┤  ├─────────────────┤
│   │  │ • REST          │  │ • Offline-first │  │ • Role-based    │
│   │  │ • GraphQL       │  │ • Real-time     │  │ • Field-level   │
│   │  │ • Auto-generated│  │ • Conflict res. │  │ • Dynamic       │
│   │  │ • OpenAPI       │  │ • Mobile SDK    │  │ • Per-document  │
│   │  └─────────────────┘  └─────────────────┘  └─────────────────┘
│   │
│   └───────────────────────────────┬───────────────────────────────────┘
│                                   │
│                                   ▼
│   MONGODB ATLAS CLUSTER
│   ┌──────────────────────────────────────────────────────────────────┐
│   │  Production Data
│   │  • Documents, Collections, Databases
│   │  • Indexes, Aggregations
│   │  • Standard MongoDB features
│   └──────────────────────────────────────────────────────────────────┘
│
│   AVANTAGES:
│   ✅ Zero server management (fully serverless)
│   ✅ Authentication built-in (8+ providers)
│   ✅ Auto-scaling (pay per execution)
│   ✅ Real-time sync (mobile/offline)
│   ✅ Security rules (row-level security)
│   ✅ Free tier généreux (1M requests/month)
│
└───────────────────────────────────────────────────────────────────────┘
```

### Cas d'Usage

```
┌───────────────────────────────────────────────────────────────────────┐
│                 ATLAS APP SERVICES - CAS D'USAGE                      │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. APPLICATION MOBILE                                                │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Besoin:                                                          │ │
│  │ • Authentification utilisateurs                                  │ │
│  │ • Synchronisation offline/online                                 │ │
│  │ • Backend sans serveur à gérer                                   │ │
│  │                                                                  │ │
│  │ Solution:                                                        │ │
│  │ • Authentication (Google, Apple, Email)                          │ │
│  │ • Device Sync (données locales + sync cloud)                     │ │
│  │ • Functions (business logic serverless)                          │ │
│  │                                                                  │ │
│  │ Exemple: App de tâches, notes, fitness tracking                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  2. MVP / PROTOTYPE RAPIDE                                            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Besoin:                                                          │ │
│  │ • Lancer rapidement sans infrastructure                          │ │
│  │ • Backend API généré automatiquement                             │ │
│  │ • Coûts minimaux au démarrage                                    │ │
│  │                                                                  │ │
│  │ Solution:                                                        │ │
│  │ • Data API (REST/GraphQL auto-généré)                            │ │
│  │ • Free tier (1M requests/month)                                  │ │
│  │ • Deploy en minutes                                              │ │
│  │                                                                  │ │
│  │ Exemple: Startup testing product-market fit                      │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  3. AUTOMATION / WORKFLOWS                                            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Besoin:                                                          │ │
│  │ • Réagir aux changements de données                              │ │
│  │ • Tâches planifiées (cron jobs)                                  │ │
│  │ • Intégrations externes                                          │ │
│  │                                                                  │ │
│  │ Solution:                                                        │ │
│  │ • Database Triggers (onChange)                                   │ │
│  │ • Scheduled Triggers (cron)                                      │ │
│  │ • HTTP Functions (webhooks)                                      │ │
│  │                                                                  │ │
│  │ Exemple: Email notifications, data processing, sync externe      │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  4. IOT / EDGE COMPUTING                                              │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Besoin:                                                          │ │
│  │ • Collecte données capteurs                                      │ │
│  │ • Processing temps réel                                          │ │
│  │ • Gestion device fleet                                           │ │
│  │                                                                  │ │
│  │ Solution:                                                        │ │
│  │ • Device authentication (API keys)                               │ │
│  │ • Functions (data processing)                                    │ │
│  │ • Triggers (alerting)                                            │ │
│  │                                                                  │ │
│  │ Exemple: Smart home, industrial sensors, fleet management        │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 Authentication

### Providers Disponibles

```javascript
// Atlas App Services supporte 8+ authentication providers

// 1. EMAIL/PASSWORD (Built-in)
const credentials = Realm.Credentials.emailPassword(
  "user@example.com",
  "securePassword123"
);
const user = await app.logIn(credentials);

// 2. ANONYMOUS (No credentials)
const anonUser = await app.logIn(Realm.Credentials.anonymous());
// Use case: Guest access, convert to registered later

// 3. GOOGLE OAUTH
const googleUser = await app.logIn(
  Realm.Credentials.google({ idToken: googleIdToken })
);

// 4. APPLE OAUTH
const appleUser = await app.logIn(
  Realm.Credentials.apple(appleIdToken)
);

// 5. FACEBOOK
const fbUser = await app.logIn(
  Realm.Credentials.facebook(accessToken)
);

// 6. CUSTOM JWT (Your own auth system)
const jwtUser = await app.logIn(
  Realm.Credentials.jwt(yourJwtToken)
);
// Use case: Integration avec système auth existant

// 7. API KEY (For services/IoT)
const apiKeyUser = await app.logIn(
  Realm.Credentials.apiKey("xvDH2...")
);
// Use case: Server-to-server, IoT devices

// 8. CUSTOM FUNCTION (Advanced)
const customUser = await app.logIn(
  Realm.Credentials.function({ username: "user", specialToken: "xyz" })
);
```

### Configuration Authentication

```javascript
// Configuration via Atlas UI ou App Services CLI

// Email/Password Configuration
{
  "email_password": {
    "enabled": true,
    "autoConfirm": false,  // Require email confirmation
    "confirmEmailSubject": "Confirm your email",
    "resetPasswordSubject": "Reset your password",
    "resetPasswordUrl": "https://app.example.com/reset-password",
    "runConfirmationFunction": false,
    "runResetFunction": false
  }
}

// Google OAuth Configuration
{
  "google": {
    "enabled": true,
    "clientId": "xxxxx.apps.googleusercontent.com",
    "metadataFields": [
      {
        "required": true,
        "name": "email"
      },
      {
        "required": false,
        "name": "name"
      }
    ]
  }
}

// API Key Configuration
{
  "api_key": {
    "enabled": true,
    "autoConfirm": true
  }
}
```

### Exemple Complet (React)

```javascript
// React app with Atlas App Services authentication

import * as Realm from "realm-web";
import { useState, useEffect } from "react";

const APP_ID = "myapp-xxxxx";
const app = new Realm.App({ id: APP_ID });

function App() {
  const [user, setUser] = useState(app.currentUser);

  // Email/Password Login
  async function loginEmailPassword(email, password) {
    const credentials = Realm.Credentials.emailPassword(email, password);
    const user = await app.logIn(credentials);
    setUser(user);
  }

  // Google Login
  async function loginGoogle() {
    // Redirect to Google OAuth
    await app.logIn(Realm.Credentials.google({ redirectUrl: window.location.origin }));
  }

  // Anonymous Login
  async function loginAnonymous() {
    const user = await app.logIn(Realm.Credentials.anonymous());
    setUser(user);
  }

  // Logout
  async function logout() {
    await user.logOut();
    setUser(null);
  }

  // Register
  async function register(email, password) {
    await app.emailPasswordAuth.registerUser({ email, password });
    // Auto-login after registration
    await loginEmailPassword(email, password);
  }

  if (!user) {
    return (
      <div>
        <h1>Login</h1>
        <button onClick={() => loginEmailPassword("demo@example.com", "password")}>
          Email Login
        </button>
        <button onClick={loginGoogle}>
          Sign in with Google
        </button>
        <button onClick={loginAnonymous}>
          Continue as Guest
        </button>
      </div>
    );
  }

  return (
    <div>
      <h1>Welcome {user.profile.email || "Guest"}</h1>
      <button onClick={logout}>Logout</button>
      {/* Your app content */}
    </div>
  );
}
```

---

## ⚡ Functions (Serverless)

### Création de Functions

```javascript
// Functions = Serverless JavaScript/Node.js code

// FONCTION 1: HTTP Endpoint
// File: functions/getUser.js
exports = async function(request, response) {
  // Accessible via: https://data.mongodb-api.com/app/myapp/endpoint/getUser

  const { userId } = request.query;

  if (!userId) {
    return { error: "Missing userId parameter" };
  }

  // Access MongoDB
  const users = context.services.get("mongodb-atlas")
    .db("mydb")
    .collection("users");

  const user = await users.findOne({ _id: BSON.ObjectId(userId) });

  if (!user) {
    response.setStatusCode(404);
    return { error: "User not found" };
  }

  // Return data
  return {
    id: user._id,
    name: user.name,
    email: user.email
  };
};

// FONCTION 2: Create Order (Business Logic)
// File: functions/createOrder.js
exports = async function(orderData) {
  // Validate
  if (!orderData.items || orderData.items.length === 0) {
    throw new Error("Order must contain items");
  }

  // Calculate total
  let total = 0;
  for (const item of orderData.items) {
    total += item.price * item.quantity;
  }

  // Create order document
  const order = {
    userId: context.user.id,
    items: orderData.items,
    total: total,
    status: "pending",
    createdAt: new Date()
  };

  // Insert to database
  const orders = context.services.get("mongodb-atlas")
    .db("mydb")
    .collection("orders");

  const result = await orders.insertOne(order);

  // Send confirmation email (call another function)
  await context.functions.execute("sendOrderEmail", result.insertedId);

  return {
    orderId: result.insertedId,
    total: total,
    status: "success"
  };
};

// FONCTION 3: External API Call
// File: functions/sendEmail.js
exports = async function(to, subject, body) {
  const sgApiKey = context.values.get("sendgridApiKey");

  const response = await context.http.post({
    url: "https://api.sendgrid.com/v3/mail/send",
    headers: {
      "Authorization": [`Bearer ${sgApiKey}`],
      "Content-Type": ["application/json"]
    },
    body: JSON.stringify({
      personalizations: [{ to: [{ email: to }] }],
      from: { email: "noreply@myapp.com" },
      subject: subject,
      content: [{ type: "text/plain", value: body }]
    })
  });

  return response.statusCode === 202;
};
```

### Appel de Functions

```javascript
// Depuis client (React/Mobile)
const app = new Realm.App({ id: APP_ID });

// Call function
const result = await app.currentUser.functions.createOrder({
  items: [
    { productId: "123", quantity: 2, price: 29.99 },
    { productId: "456", quantity: 1, price: 49.99 }
  ]
});

console.log("Order created:", result.orderId);

// HTTP Endpoint (accessible via REST)
fetch("https://data.mongodb-api.com/app/myapp/endpoint/getUser?userId=123")
  .then(r => r.json())
  .then(data => console.log(data));
```

---

## 🎯 Triggers

### Types de Triggers

```
┌───────────────────────────────────────────────────────────────────────┐
│                        TRIGGERS TYPES                                 │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. DATABASE TRIGGERS (onChange)                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Déclenché par: Insert, Update, Delete, Replace                   │ │
│  │                                                                  │ │
│  │ Configuration:                                                   │ │
│  │ • Collection: users                                              │ │
│  │ • Operations: INSERT, UPDATE                                     │ │
│  │ • Full Document: true (include complete doc)                     │ │
│  │ • Function: onUserChange                                         │ │
│  │                                                                  │ │
│  │ Use cases:                                                       │ │
│  │ • Audit logging                                                  │ │
│  │ • Data validation/enrichment                                     │ │
│  │ • Sync to external system                                        │ │
│  │ • Notifications                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  2. SCHEDULED TRIGGERS (cron)                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Déclenché par: Schedule (cron expression)                        │ │
│  │                                                                  │ │
│  │ Configuration:                                                   │ │
│  │ • Schedule: "0 2 * * *" (daily at 2 AM)                          │ │
│  │ • Function: dailyCleanup                                         │ │
│  │ • Skip if previous still running: true                           │ │
│  │                                                                  │ │
│  │ Use cases:                                                       │ │
│  │ • Batch processing                                               │ │
│  │ • Reports generation                                             │ │
│  │ • Data archiving                                                 │ │
│  │ • Scheduled notifications                                        │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  3. AUTHENTICATION TRIGGERS                                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Déclenché par: User events (create, login, delete)               │ │
│  │                                                                  │ │
│  │ Events:                                                          │ │
│  │ • CREATE: New user registered                                    │ │
│  │ • LOGIN: User logged in                                          │ │
│  │ • DELETE: User account deleted                                   │ │
│  │                                                                  │ │
│  │ Use cases:                                                       │ │
│  │ • Welcome email on signup                                        │ │
│  │ • Create user profile                                            │ │
│  │ • Analytics tracking                                             │ │
│  │ • Security alerts                                                │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Exemples de Triggers

```javascript
// TRIGGER 1: Database Trigger - Audit Log
// Triggered on: orders collection, INSERT operation

exports = async function(changeEvent) {
  const { fullDocument, operationType } = changeEvent;

  if (operationType !== "insert") return;

  // Log to audit collection
  const auditLogs = context.services.get("mongodb-atlas")
    .db("mydb")
    .collection("audit_logs");

  await auditLogs.insertOne({
    event: "order_created",
    orderId: fullDocument._id,
    userId: fullDocument.userId,
    amount: fullDocument.total,
    timestamp: new Date()
  });

  console.log(`Audit log created for order ${fullDocument._id}`);
};

// TRIGGER 2: Scheduled Trigger - Daily Cleanup
// Schedule: "0 2 * * *" (2 AM daily)

exports = async function() {
  const thirtyDaysAgo = new Date();
  thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);

  // Delete old logs
  const logs = context.services.get("mongodb-atlas")
    .db("mydb")
    .collection("logs");

  const result = await logs.deleteMany({
    timestamp: { $lt: thirtyDaysAgo }
  });

  console.log(`Deleted ${result.deletedCount} old logs`);

  // Archive old orders
  const orders = context.services.get("mongodb-atlas")
    .db("mydb")
    .collection("orders");

  const oldOrders = await orders.find({
    createdAt: { $lt: thirtyDaysAgo },
    status: "completed"
  }).toArray();

  // Move to archive (S3 via function)
  if (oldOrders.length > 0) {
    await context.functions.execute("archiveToS3", oldOrders);
    await orders.deleteMany({
      _id: { $in: oldOrders.map(o => o._id) }
    });
  }

  return {
    logsDeleted: result.deletedCount,
    ordersArchived: oldOrders.length
  };
};

// TRIGGER 3: Authentication Trigger - Welcome Email
// Event: CREATE (new user)

exports = async function(authEvent) {
  const { user } = authEvent;

  // Send welcome email
  await context.functions.execute("sendEmail", {
    to: user.data.email,
    subject: "Welcome to MyApp!",
    body: `Hi ${user.data.email}, welcome to our platform!`
  });

  // Create user profile
  const profiles = context.services.get("mongodb-atlas")
    .db("mydb")
    .collection("profiles");

  await profiles.insertOne({
    userId: user.id,
    email: user.data.email,
    createdAt: new Date(),
    preferences: {
      notifications: true,
      theme: "light"
    }
  });

  console.log(`Welcome email sent to ${user.data.email}`);
};
```

---

## 🌐 Data API (REST/GraphQL)

### Auto-Generated API

```
┌───────────────────────────────────────────────────────────────────────┐
│                      DATA API ARCHITECTURE                            │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Atlas App Services génère automatiquement:                           │
│  • REST API endpoints                                                 │
│  • GraphQL API                                                        │
│  • OpenAPI specification                                              │
│                                                                       │
│  Base URL:                                                            │
│  https://data.mongodb-api.com/app/<APP_ID>/endpoint/data/v1           │
│                                                                       │
│  ENDPOINTS GÉNÉRÉS:                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ POST   /action/findOne       Find single document                │ │
│  │ POST   /action/find          Find multiple documents             │ │
│  │ POST   /action/insertOne     Insert single document              │ │
│  │ POST   /action/insertMany    Insert multiple documents           │ │
│  │ POST   /action/updateOne     Update single document              │ │
│  │ POST   /action/updateMany    Update multiple documents           │ │
│  │ POST   /action/deleteOne     Delete single document              │ │
│  │ POST   /action/deleteMany    Delete multiple documents           │ │
│  │ POST   /action/aggregate     Run aggregation pipeline            │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  AUTHENTICATION:                                                      │
│  • API Key (header: api-key)                                          │
│  • JWT (header: jwtTokenString)                                       │
│  • Email/Password (login endpoint)                                    │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Exemples REST API

```bash
# Configuration
export API_KEY="your-api-key"
export APP_ID="myapp-xxxxx"
export BASE_URL="https://data.mongodb-api.com/app/$APP_ID/endpoint/data/v1"

# 1. Find documents
curl -X POST "$BASE_URL/action/find" \
  -H "Content-Type: application/json" \
  -H "api-key: $API_KEY" \
  -d '{
    "dataSource": "mongodb-atlas",
    "database": "mydb",
    "collection": "products",
    "filter": { "category": "electronics" },
    "limit": 10
  }'

# 2. Insert document
curl -X POST "$BASE_URL/action/insertOne" \
  -H "Content-Type: application/json" \
  -H "api-key: $API_KEY" \
  -d '{
    "dataSource": "mongodb-atlas",
    "database": "mydb",
    "collection": "orders",
    "document": {
      "userId": "user123",
      "items": [
        { "productId": "prod456", "quantity": 2 }
      ],
      "total": 59.98,
      "status": "pending"
    }
  }'

# 3. Update document
curl -X POST "$BASE_URL/action/updateOne" \
  -H "Content-Type: application/json" \
  -H "api-key: $API_KEY" \
  -d '{
    "dataSource": "mongodb-atlas",
    "database": "mydb",
    "collection": "orders",
    "filter": { "_id": { "$oid": "507f1f77bcf86cd799439011" } },
    "update": {
      "$set": { "status": "completed" }
    }
  }'

# 4. Aggregation
curl -X POST "$BASE_URL/action/aggregate" \
  -H "Content-Type: application/json" \
  -H "api-key: $API_KEY" \
  -d '{
    "dataSource": "mongodb-atlas",
    "database": "mydb",
    "collection": "orders",
    "pipeline": [
      { "$match": { "status": "completed" } },
      { "$group": {
          "_id": "$userId",
          "totalSpent": { "$sum": "$total" }
        }
      },
      { "$sort": { "totalSpent": -1 } },
      { "$limit": 10 }
    ]
  }'
```

### GraphQL API

```graphql
# Atlas génère un schema GraphQL automatiquement

# Query: Get products
query GetProducts {
  products(query: { category: "electronics" }, limit: 10) {
    _id
    name
    price
    category
    inStock
  }
}

# Query: Get user with orders
query GetUserWithOrders($userId: String!) {
  user(query: { _id: $userId }) {
    _id
    name
    email
    orders {
      _id
      total
      status
      createdAt
    }
  }
}

# Mutation: Create order
mutation CreateOrder($order: OrderInsertInput!) {
  insertOneOrder(data: $order) {
    _id
    total
    status
  }
}

# Variables
{
  "order": {
    "userId": "user123",
    "items": [
      { "productId": "prod456", "quantity": 2 }
    ],
    "total": 59.98,
    "status": "pending"
  }
}
```

---

## 📋 Best Practices

### Sécurité

```javascript
// Rules & Roles (Access Control)

// Collection: orders
// Rule: Users can only see their own orders
{
  "name": "ownerRead",
  "apply_when": {},  // Always apply
  "read": {
    "userId": "%%user.id"  // Filter: only documents where userId matches
  },
  "write": {
    "userId": "%%user.id"
  }
}

// Collection: products
// Rule: Everyone can read, only admins can write
{
  "name": "publicRead",
  "apply_when": {},
  "read": true,  // All users can read
  "write": {
    "%%user.custom_data.role": "admin"  // Only admins can write
  }
}

// Function-level permissions
// Function: deleteUser (admin only)
{
  "name": "deleteUser",
  "private": false,
  "can_evaluate": {
    "%%user.custom_data.role": "admin"
  },
  "source": "..."
}
```

### Checklist Production

```
┌───────────────────────────────────────────────────────────────────────┐
│              APP SERVICES PRODUCTION CHECKLIST                        │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  AUTHENTICATION                                                       │
│  ☐ Enable email confirmation (no auto-confirm)                        │
│  ☐ Configure password reset flow                                      │
│  ☐ Set secure password policy (min 8 chars)                           │
│  ☐ Disable anonymous auth in production (if not needed)               │
│  ☐ Use custom JWT for existing auth systems                           │
│                                                                       │
│  SECURITY                                                             │
│  ☐ Configure granular access rules (per collection)                   │
│  ☐ Test rules thoroughly (positive & negative cases)                  │
│  ☐ Enable field-level permissions                                     │
│  ☐ Store secrets in Values (not hardcoded)                            │
│  ☐ Rotate API keys regularly                                          │
│                                                                       │
│  FUNCTIONS                                                            │
│  ☐ Handle errors gracefully (try/catch)                               │
│  ☐ Set appropriate execution timeout                                  │
│  ☐ Log important events                                               │
│  ☐ Avoid long-running functions (> 90s)                               │
│  ☐ Test with production-scale data                                    │
│                                                                       │
│  TRIGGERS                                                             │
│  ☐ Make triggers idempotent                                           │
│  ☐ Handle failures (retry logic)                                      │
│  ☐ Monitor trigger execution times                                    │
│  ☐ Limit database trigger frequency (avoid loops)                     │
│  ☐ Test scheduled triggers in staging                                 │
│                                                                       │
│  MONITORING                                                           │
│  ☐ Enable logs (review regularly)                                     │
│  ☐ Set up alerts for errors                                           │
│  ☐ Monitor function execution time                                    │
│  ☐ Track API usage (stay within limits)                               │
│  ☐ Review costs monthly                                               │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🏁 Résumé

### Points Clés

1. **Backend-as-a-Service**
   - Zero server management
   - Auto-scaling serverless
   - Free tier généreux (1M requests)
   - Pay per execution

2. **Authentication**
   - 8+ providers (Email, Google, Apple, etc.)
   - Anonymous auth
   - API keys for services
   - Custom JWT integration

3. **Functions**
   - JavaScript/Node.js serverless
   - HTTP endpoints
   - Call from client or triggers
   - Access MongoDB directly

4. **Triggers**
   - Database (onChange)
   - Scheduled (cron)
   - Authentication (user events)
   - Webhooks

5. **Data API**
   - REST auto-generated
   - GraphQL auto-generated
   - OpenAPI spec
   - Secure with rules

### Configuration Minimale

```javascript
// 1. Initialize App
const app = new Realm.App({ id: "myapp-xxxxx" });

// 2. Authenticate
const user = await app.logIn(
  Realm.Credentials.emailPassword(email, password)
);

// 3. Call Function
const result = await user.functions.myFunction(args);

// 4. Access Data API
fetch("https://data.mongodb-api.com/app/myapp/endpoint/data/v1/action/find", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    "api-key": API_KEY
  },
  body: JSON.stringify({
    dataSource: "mongodb-atlas",
    database: "mydb",
    collection: "items",
    filter: {}
  })
});
```

### Ressources

- [App Services Documentation](https://www.mongodb.com/docs/atlas/app-services/)
- [Realm SDK](https://www.mongodb.com/docs/realm/)
- [Data API Reference](https://www.mongodb.com/docs/atlas/api/data-api/)

---


⏭️ [Atlas Vector Search](/14-mongodb-atlas/13-atlas-vector-search.md)
