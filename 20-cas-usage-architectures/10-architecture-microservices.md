🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.10 Architecture Microservices

## Introduction

L'architecture microservices décompose une application en services indépendants, autonomes et faiblement couplés. Cette approche présente des défis spécifiques pour la gestion des données :

- **Database per service** : Chaque service a sa propre base de données
- **Consistency distribuée** : Eventual consistency vs strong consistency
- **Transactions distribuées** : Saga pattern pour orchestrer opérations
- **Communication** : Synchrone (REST/gRPC) vs asynchrone (events)
- **Data duplication** : Nécessaire pour autonomie
- **Service discovery** : Localisation dynamique des services
- **Resilience** : Circuit breakers, retries, timeouts
- **Observabilité** : Logging, metrics, tracing distribué
- **Déploiement** : Containers, orchestration (Kubernetes)

MongoDB s'intègre excellemment dans les microservices grâce à :
- **Schéma flexible** : Évolution indépendante par service
- **Change Streams** : Event-driven architecture
- **Transactions** : ACID dans un service
- **Replica Sets** : High availability par service
- **Atlas** : Database as a Service pour chaque microservice
- **Lightweight** : Déploiement facile en containers
- **Multi-tenancy** : Support natif pour isolation

## Architecture de référence

### Stack microservices complète

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Applications                      │
│              Web • Mobile • Desktop • Third-party APIs          │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS
                             │
                ┌────────────▼──────────────┐
                │      API Gateway          │
                │  • Kong / AWS API GW      │
                │  • Auth / Rate Limit      │
                │  • Request Routing        │
                │  • API Composition        │
                └────────────┬──────────────┘
                             │
         ┌───────────────────┼────────────────────┐
         │                   │                    │
    ┌────▼─────┐      ┌─────▼──────┐      ┌─────▼──────┐
    │  User    │      │   Order    │      │  Product   │
    │ Service  │      │  Service   │      │  Service   │
    └────┬─────┘      └─────┬──────┘      └─────┬──────┘
         │                  │                   │
    ┌────▼─────┐      ┌─────▼──────┐      ┌─────▼──────┐
    │ MongoDB  │      │  MongoDB   │      │  MongoDB   │
    │ users DB │      │ orders DB  │      │ products   │
    └──────────┘      └─────┬──────┘      └────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
    ┌────▼─────┐      ┌─────▼──────┐    ┌──────▼─────┐
    │Payment   │      │Inventory   │    │Shipping    │
    │Service   │      │ Service    │    │ Service    │
    └────┬─────┘      └─────┬──────┘    └─────┬──────┘
         │                  │                 │
    ┌────▼─────┐      ┌─────▼──────┐    ┌─────▼──────┐
    │MongoDB   │      │  MongoDB   │    │  MongoDB   │
    │payments  │      │ inventory  │    │ shipping   │
    └──────────┘      └────────────┘    └────────────┘
         │                  │                  │
         └──────────────────┴──────────────────┘
                            │
               ┌────────────▼──────────────┐
               │    Message Broker         │
               │  (Kafka / RabbitMQ)       │
               │  • Events                 │
               │  • Async Communication    │
               │  • Event Store            │
               └────────────┬──────────────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
    ┌────▼─────┐      ┌─────▼──────┐    ┌──────▼─────┐
    │  Email   │      │Analytics   │    │ Audit      │
    │ Service  │      │ Service    │    │ Service    │
    └──────────┘      └────────────┘    └────────────┘
         │                  │                  │
    ┌────▼─────┐      ┌─────▼──────┐    ┌──────▼─────┐
    │MongoDB   │      │  MongoDB   │    │  MongoDB   │
    │  email   │      │ analytics  │    │   audit    │
    └──────────┘      └────────────┘    └────────────┘

┌───────────────────────────────────────────────────────────────┐
│                   Supporting Infrastructure                   │
├───────────────────────────────────────────────────────────────┤
│  Service Discovery: Consul / Eureka / Kubernetes DNS          │
│  Config Management: Spring Cloud Config / Consul KV           │
│  Distributed Tracing: Jaeger / Zipkin                         │
│  Monitoring: Prometheus + Grafana                             │
│  Logging: ELK Stack (Elasticsearch, Logstash, Kibana)         │
│  Service Mesh: Istio / Linkerd (optional)                     │
└───────────────────────────────────────────────────────────────┘
```

### Principes architecturaux

#### 1. Database per Service
**Principe :** Chaque microservice possède sa propre base de données, inaccessible directement par les autres services.

**Justification :**
- **Autonomie** : Service peut évoluer indépendamment
- **Isolation** : Changements de schéma n'impactent qu'un service
- **Scalabilité** : Scaling indépendant par service
- **Technology diversity** : Choix optimal par service
- **Fault isolation** : Panne d'un service n'affecte pas les autres

**Trade-offs :**
- ❌ Pas de transactions cross-services
- ❌ Duplication de données nécessaire
- ❌ Queries cross-services complexes
- ✅ Loose coupling
- ✅ Independent deployment

#### 2. Communication Patterns

**Synchronous (REST/gRPC) :**
- Pour requêtes immédiates
- Request-response simple
- Strong consistency temporaire

**Asynchronous (Events) :**
- Pour operations non-bloquantes
- Eventual consistency
- Meilleure résilience
- Découplage temporel

**Justification :** Hybrid approach - synchrone pour queries critiques, asynchrone pour updates et notifications.

#### 3. Data Consistency

**Saga Pattern :** Transaction distribuée via événements/compensation
**CQRS :** Séparation read/write models
**Event Sourcing :** Store events, derive state

#### 4. Service Discovery

**Technologies :** Consul, Eureka, Kubernetes DNS

**Justification :** Services dynamiques, instances ephémères

## Patterns et implémentations

### 1. Database per Service Pattern

```javascript
// User Service - users database
// Collection: users
{
  _id: ObjectId("..."),
  userId: "user_abc123",
  email: "john.doe@example.com",
  name: "John Doe",
  passwordHash: "$2b$10$...",

  // Données propres au User Service
  profile: {
    avatar: "https://cdn.example.com/avatars/...",
    bio: "Software developer",
    location: "Paris, France"
  },

  preferences: {
    language: "fr",
    timezone: "Europe/Paris",
    emailNotifications: true
  },

  status: "active",
  createdAt: ISODate("2024-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z")
}

// Order Service - orders database
// Collection: orders
{
  _id: ObjectId("..."),
  orderId: "order_xyz789",

  // Référence userId (pas de foreign key!)
  userId: "user_abc123",

  // Données dupliquées pour autonomie
  userSnapshot: {
    email: "john.doe@example.com",
    name: "John Doe",
    // Snapshot au moment de la commande
    capturedAt: ISODate("2024-12-09T14:30:00Z")
  },

  items: [
    {
      // Référence productId
      productId: "prod_def456",

      // Données dupliquées du Product Service
      productSnapshot: {
        name: "Wireless Headphones",
        price: NumberDecimal("299.99"),
        sku: "WH-1000XM5"
      },

      quantity: 1,
      price: NumberDecimal("299.99")
    }
  ],

  total: NumberDecimal("299.99"),
  currency: "USD",

  status: "pending",  // pending, paid, shipped, delivered, cancelled

  createdAt: ISODate("2024-12-09T14:30:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z")
}

// Product Service - products database
// Collection: products
{
  _id: ObjectId("..."),
  productId: "prod_def456",

  name: "Wireless Headphones",
  description: "Premium noise-cancelling headphones",

  price: NumberDecimal("299.99"),
  currency: "USD",

  sku: "WH-1000XM5",

  inventory: {
    quantity: 150,
    reserved: 10,
    available: 140
  },

  status: "active",
  createdAt: ISODate("2024-06-01T00:00:00Z"),
  updatedAt: ISODate("2024-12-09T10:00:00Z")
}
```

### 2. Event-Driven Communication avec Change Streams

```javascript
// User Service - Publier événements user
class UserService {
  constructor(db, eventBus) {
    this.db = db;
    this.eventBus = eventBus;
  }

  async start() {
    // Watch changes sur users collection
    const changeStream = this.db.collection('users').watch();

    changeStream.on('change', async (change) => {
      if (change.operationType === 'insert') {
        await this.publishUserCreatedEvent(change.fullDocument);
      }

      if (change.operationType === 'update') {
        await this.publishUserUpdatedEvent(
          change.documentKey._id,
          change.updateDescription
        );
      }

      if (change.operationType === 'delete') {
        await this.publishUserDeletedEvent(change.documentKey._id);
      }
    });

    console.log('User Service event publisher started');
  }

  async createUser(userData) {
    const user = {
      userId: this.generateUserId(),
      email: userData.email,
      name: userData.name,
      passwordHash: await this.hashPassword(userData.password),

      profile: userData.profile || {},
      preferences: this.getDefaultPreferences(),

      status: 'active',
      createdAt: new Date(),
      updatedAt: new Date()
    };

    const result = await this.db.collection('users').insertOne(user);
    user._id = result.insertedId;

    // Change Stream publiera l'événement automatiquement

    return user;
  }

  async publishUserCreatedEvent(user) {
    const event = {
      eventId: this.generateEventId(),
      eventType: 'user.created',
      version: '1.0',

      timestamp: new Date(),

      data: {
        userId: user.userId,
        email: user.email,
        name: user.name,
        status: user.status
      },

      metadata: {
        service: 'user-service',
        correlationId: user.userId
      }
    };

    // Publier sur message broker
    await this.eventBus.publish('user.events', event);

    console.log(`Published user.created event: ${user.userId}`);
  }

  async publishUserUpdatedEvent(userId, updateDescription) {
    const event = {
      eventId: this.generateEventId(),
      eventType: 'user.updated',
      version: '1.0',

      timestamp: new Date(),

      data: {
        userId,
        updatedFields: updateDescription.updatedFields,
        removedFields: updateDescription.removedFields
      },

      metadata: {
        service: 'user-service'
      }
    };

    await this.eventBus.publish('user.events', event);
  }

  generateUserId() {
    return `user_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  generateEventId() {
    return `evt_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Order Service - Consommer événements user
class OrderEventConsumer {
  constructor(db, eventBus) {
    this.db = db;
    this.eventBus = eventBus;
  }

  async start() {
    // S'abonner aux événements user
    await this.eventBus.subscribe('user.events', async (event) => {
      try {
        await this.handleUserEvent(event);
      } catch (error) {
        console.error('Failed to handle user event:', error);
        // Dead letter queue
        await this.sendToDeadLetter(event, error);
      }
    });

    console.log('Order Service event consumer started');
  }

  async handleUserEvent(event) {
    switch (event.eventType) {
      case 'user.created':
        await this.onUserCreated(event.data);
        break;

      case 'user.updated':
        await this.onUserUpdated(event.data);
        break;

      case 'user.deleted':
        await this.onUserDeleted(event.data);
        break;
    }
  }

  async onUserUpdated(data) {
    // Mettre à jour userSnapshot dans toutes les commandes
    const { userId, updatedFields } = data;

    // Seulement si champs pertinents changés
    if (updatedFields.email || updatedFields.name) {
      await this.db.collection('orders').updateMany(
        { userId },
        {
          $set: {
            'userSnapshot.email': updatedFields.email,
            'userSnapshot.name': updatedFields.name,
            'userSnapshot.updatedAt': new Date()
          }
        }
      );

      console.log(`Updated user snapshot in orders for user ${userId}`);
    }
  }

  async onUserDeleted(data) {
    // Anonymiser commandes ou autre logique
    const { userId } = data;

    await this.db.collection('orders').updateMany(
      { userId },
      {
        $set: {
          'userSnapshot.email': 'deleted@example.com',
          'userSnapshot.name': 'Deleted User',
          userDeleted: true,
          deletedAt: new Date()
        }
      }
    );

    console.log(`Anonymized orders for deleted user ${userId}`);
  }
}
```

### 3. Saga Pattern pour transactions distribuées

```javascript
// Orchestration-based Saga pour création de commande
class OrderCreationSaga {
  constructor(services) {
    this.orderService = services.order;
    this.inventoryService = services.inventory;
    this.paymentService = services.payment;
    this.shippingService = services.shipping;
  }

  async execute(orderData) {
    const sagaId = this.generateSagaId();
    const compensations = [];

    try {
      // Step 1: Créer commande (status: pending)
      const order = await this.orderService.createOrder(orderData);
      compensations.push(() => this.orderService.cancelOrder(order.orderId));

      // Step 2: Réserver inventory
      const reservation = await this.inventoryService.reserveItems(
        order.items
      );
      compensations.push(() =>
        this.inventoryService.releaseReservation(reservation.reservationId)
      );

      // Step 3: Process payment
      const payment = await this.paymentService.processPayment({
        orderId: order.orderId,
        amount: order.total,
        currency: order.currency,
        userId: order.userId
      });
      compensations.push(() =>
        this.paymentService.refund(payment.paymentId)
      );

      // Step 4: Créer expédition
      const shipment = await this.shippingService.createShipment({
        orderId: order.orderId,
        userId: order.userId,
        items: order.items,
        address: order.shippingAddress
      });
      // Pas de compensation pour shipment (déjà payé)

      // Step 5: Finaliser commande
      await this.orderService.confirmOrder(order.orderId, {
        paymentId: payment.paymentId,
        reservationId: reservation.reservationId,
        shipmentId: shipment.shipmentId
      });

      // Success!
      await this.logSagaSuccess(sagaId, order.orderId);

      return {
        success: true,
        order
      };

    } catch (error) {
      console.error(`Saga ${sagaId} failed:`, error);

      // Exécuter compensations dans l'ordre inverse
      await this.compensate(compensations, sagaId);

      throw new Error(`Order creation failed: ${error.message}`);
    }
  }

  async compensate(compensations, sagaId) {
    console.log(`Starting compensation for saga ${sagaId}`);

    // Exécuter dans l'ordre inverse
    for (const compensation of compensations.reverse()) {
      try {
        await compensation();
      } catch (error) {
        console.error('Compensation failed:', error);
        // Log mais continue
        await this.logCompensationFailure(sagaId, error);
      }
    }

    console.log(`Compensation completed for saga ${sagaId}`);
  }

  generateSagaId() {
    return `saga_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Inventory Service
class InventoryService {
  constructor(db) {
    this.db = db;
  }

  async reserveItems(items) {
    const session = this.db.client.startSession();

    try {
      let reservation;

      await session.withTransaction(async () => {
        const reservationId = this.generateReservationId();

        // Vérifier et réserver chaque item
        for (const item of items) {
          const product = await this.db.collection('products')
            .findOneAndUpdate(
              {
                productId: item.productId,
                'inventory.available': { $gte: item.quantity }
              },
              {
                $inc: {
                  'inventory.available': -item.quantity,
                  'inventory.reserved': item.quantity
                }
              },
              { session, returnDocument: 'after' }
            );

          if (!product) {
            throw new Error(
              `Insufficient inventory for product ${item.productId}`
            );
          }
        }

        // Créer réservation
        reservation = {
          reservationId,
          items,
          status: 'active',
          expiresAt: new Date(Date.now() + 15 * 60 * 1000),  // 15 min
          createdAt: new Date()
        };

        await this.db.collection('reservations').insertOne(
          reservation,
          { session }
        );

      }, {
        writeConcern: { w: 'majority', j: true }
      });

      return reservation;

    } finally {
      await session.endSession();
    }
  }

  async releaseReservation(reservationId) {
    const session = this.db.client.startSession();

    try {
      await session.withTransaction(async () => {
        // Récupérer réservation
        const reservation = await this.db.collection('reservations')
          .findOne({ reservationId }, { session });

        if (!reservation) {
          throw new Error('Reservation not found');
        }

        // Libérer inventory
        for (const item of reservation.items) {
          await this.db.collection('products').updateOne(
            { productId: item.productId },
            {
              $inc: {
                'inventory.available': item.quantity,
                'inventory.reserved': -item.quantity
              }
            },
            { session }
          );
        }

        // Supprimer réservation
        await this.db.collection('reservations').deleteOne(
          { reservationId },
          { session }
        );

      });

    } finally {
      await session.endSession();
    }
  }

  generateReservationId() {
    return `rsv_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### 4. CQRS Pattern (Command Query Responsibility Segregation)

```javascript
// Write Model - Order Service
class OrderCommandService {
  constructor(db, eventBus) {
    this.db = db;
    this.eventBus = eventBus;
  }

  async createOrder(command) {
    const order = {
      orderId: this.generateOrderId(),
      userId: command.userId,

      items: command.items,
      total: this.calculateTotal(command.items),
      currency: command.currency,

      shippingAddress: command.shippingAddress,
      billingAddress: command.billingAddress,

      status: 'pending',

      createdAt: new Date(),
      updatedAt: new Date(),
      version: 1
    };

    await this.db.collection('orders').insertOne(order);

    // Publier événement
    await this.publishOrderCreatedEvent(order);

    return order;
  }

  async updateOrderStatus(orderId, newStatus) {
    const result = await this.db.collection('orders').findOneAndUpdate(
      { orderId },
      {
        $set: {
          status: newStatus,
          updatedAt: new Date()
        },
        $inc: { version: 1 }
      },
      { returnDocument: 'after' }
    );

    if (result) {
      await this.publishOrderStatusChangedEvent(result);
    }

    return result;
  }

  async publishOrderCreatedEvent(order) {
    const event = {
      eventType: 'order.created',
      data: {
        orderId: order.orderId,
        userId: order.userId,
        total: order.total,
        items: order.items
      },
      timestamp: new Date()
    };

    await this.eventBus.publish('order.events', event);
  }

  generateOrderId() {
    return `order_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Read Model - Order Query Service (denormalized, optimized for reads)
class OrderQueryService {
  constructor(db, eventBus) {
    this.db = db;
    this.eventBus = eventBus;
    this.startEventConsumer();
  }

  async startEventConsumer() {
    // Construire read model depuis events
    await this.eventBus.subscribe('order.events', async (event) => {
      await this.updateReadModel(event);
    });

    // Aussi subscribe user et product events pour enrichissement
    await this.eventBus.subscribe('user.events', async (event) => {
      if (event.eventType === 'user.updated') {
        await this.updateUserDataInOrders(event.data);
      }
    });
  }

  async updateReadModel(event) {
    switch (event.eventType) {
      case 'order.created':
        await this.onOrderCreated(event.data);
        break;

      case 'order.status.changed':
        await this.onOrderStatusChanged(event.data);
        break;
    }
  }

  async onOrderCreated(data) {
    // Enrichir avec user et product data
    const user = await this.fetchUserData(data.userId);
    const enrichedItems = await this.enrichItems(data.items);

    // Créer document read model dénormalisé
    const readModel = {
      orderId: data.orderId,

      // User data embedded
      user: {
        userId: user.userId,
        name: user.name,
        email: user.email
      },

      // Items enriched
      items: enrichedItems,

      total: data.total,
      status: 'pending',

      // Metadata
      createdAt: new Date(),
      updatedAt: new Date()
    };

    await this.db.collection('orders_read').insertOne(readModel);
  }

  async getUserOrders(userId, options = {}) {
    const {
      status,
      limit = 20,
      offset = 0,
      sortBy = 'createdAt',
      sortOrder = -1
    } = options;

    const query = { 'user.userId': userId };

    if (status) {
      query.status = status;
    }

    const orders = await this.db.collection('orders_read')
      .find(query)
      .sort({ [sortBy]: sortOrder })
      .skip(offset)
      .limit(limit)
      .toArray();

    return orders;
  }

  async getOrderDetails(orderId) {
    // Read model déjà dénormalisé, query simple
    return this.db.collection('orders_read')
      .findOne({ orderId });
  }

  async enrichItems(items) {
    // Fetch product details depuis Product Service
    const productIds = items.map(item => item.productId);

    // En production: appeler Product Service API
    // Pour démo: query directe (pas recommandé cross-service!)
    const products = await this.fetchProductsData(productIds);

    return items.map(item => ({
      ...item,
      product: products.find(p => p.productId === item.productId)
    }));
  }
}
```

### 5. API Gateway Pattern

```javascript
// API Gateway avec Express
class APIGateway {
  constructor(services, config) {
    this.services = services;
    this.config = config;
    this.app = express();
    this.setupMiddleware();
    this.setupRoutes();
  }

  setupMiddleware() {
    this.app.use(express.json());

    // CORS
    this.app.use(cors(this.config.cors));

    // Authentication
    this.app.use(this.authenticateRequest.bind(this));

    // Rate limiting
    this.app.use(rateLimit({
      windowMs: 15 * 60 * 1000,  // 15 minutes
      max: 100  // limit per IP
    }));

    // Request logging
    this.app.use(this.logRequest.bind(this));

    // Circuit breaker
    this.circuitBreakers = new Map();
  }

  setupRoutes() {
    // User routes
    this.app.post('/api/users', async (req, res) => {
      try {
        const user = await this.callService(
          'user-service',
          'POST',
          '/users',
          req.body
        );

        res.status(201).json(user);
      } catch (error) {
        this.handleError(res, error);
      }
    });

    this.app.get('/api/users/:userId', async (req, res) => {
      try {
        const user = await this.callService(
          'user-service',
          'GET',
          `/users/${req.params.userId}`
        );

        res.json(user);
      } catch (error) {
        this.handleError(res, error);
      }
    });

    // Order routes
    this.app.post('/api/orders', async (req, res) => {
      try {
        const order = await this.callService(
          'order-service',
          'POST',
          '/orders',
          {
            ...req.body,
            userId: req.user.userId  // From auth
          }
        );

        res.status(201).json(order);
      } catch (error) {
        this.handleError(res, error);
      }
    });

    // API Composition - Get order with user and product details
    this.app.get('/api/orders/:orderId/details', async (req, res) => {
      try {
        // Appeler plusieurs services en parallèle
        const [order, user, products] = await Promise.all([
          this.callService(
            'order-service',
            'GET',
            `/orders/${req.params.orderId}`
          ),
          this.callService(
            'user-service',
            'GET',
            `/users/${order.userId}`
          ),
          this.getOrderProducts(order.items)
        ]);

        // Composer réponse
        const enrichedOrder = {
          ...order,
          user: {
            name: user.name,
            email: user.email
          },
          items: order.items.map((item, i) => ({
            ...item,
            product: products[i]
          }))
        };

        res.json(enrichedOrder);
      } catch (error) {
        this.handleError(res, error);
      }
    });
  }

  async callService(serviceName, method, path, body = null) {
    // Service discovery
    const serviceUrl = await this.discoverService(serviceName);

    // Circuit breaker
    const breaker = this.getCircuitBreaker(serviceName);

    return breaker.execute(async () => {
      const response = await fetch(`${serviceUrl}${path}`, {
        method,
        headers: {
          'Content-Type': 'application/json',
          'X-Request-ID': this.generateRequestId(),
          'X-Correlation-ID': this.getCorrelationId()
        },
        body: body ? JSON.stringify(body) : undefined,
        timeout: 5000  // 5s timeout
      });

      if (!response.ok) {
        throw new Error(`Service ${serviceName} error: ${response.status}`);
      }

      return response.json();
    });
  }

  getCircuitBreaker(serviceName) {
    if (!this.circuitBreakers.has(serviceName)) {
      const breaker = new CircuitBreaker(
        async (fn) => fn(),
        {
          timeout: 5000,
          errorThresholdPercentage: 50,
          resetTimeout: 30000
        }
      );

      breaker.on('open', () => {
        console.error(`Circuit breaker OPEN for ${serviceName}`);
      });

      breaker.on('halfOpen', () => {
        console.log(`Circuit breaker HALF-OPEN for ${serviceName}`);
      });

      this.circuitBreakers.set(serviceName, breaker);
    }

    return this.circuitBreakers.get(serviceName);
  }

  async discoverService(serviceName) {
    // En production: utiliser Consul, Eureka, ou Kubernetes Service
    // Pour démo: configuration statique
    const services = {
      'user-service': 'http://user-service:3001',
      'order-service': 'http://order-service:3002',
      'product-service': 'http://product-service:3003'
    };

    return services[serviceName];
  }

  async authenticateRequest(req, res, next) {
    // Skip auth pour certaines routes
    if (req.path.startsWith('/api/health')) {
      return next();
    }

    const token = req.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    try {
      // Vérifier JWT
      const decoded = jwt.verify(token, this.config.jwtSecret);
      req.user = decoded;
      next();
    } catch (error) {
      res.status(401).json({ error: 'Invalid token' });
    }
  }

  logRequest(req, res, next) {
    const requestId = this.generateRequestId();
    req.requestId = requestId;

    console.log({
      requestId,
      method: req.method,
      path: req.path,
      userId: req.user?.userId,
      timestamp: new Date()
    });

    next();
  }

  handleError(res, error) {
    console.error('API Gateway error:', error);

    if (error.message.includes('Circuit breaker')) {
      return res.status(503).json({
        error: 'Service temporarily unavailable'
      });
    }

    res.status(500).json({
      error: 'Internal server error'
    });
  }

  generateRequestId() {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

### 6. Distributed Tracing avec OpenTelemetry

```javascript
// Instrumenter services avec OpenTelemetry
const { trace } = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');

class TracingService {
  constructor(serviceName) {
    this.serviceName = serviceName;
    this.setupTracing();
  }

  setupTracing() {
    const provider = new NodeTracerProvider({
      resource: new Resource({
        [SemanticResourceAttributes.SERVICE_NAME]: this.serviceName
      })
    });

    // Exporter vers Jaeger
    const exporter = new JaegerExporter({
      endpoint: 'http://jaeger:14268/api/traces'
    });

    provider.addSpanProcessor(
      new BatchSpanProcessor(exporter)
    );

    provider.register();

    this.tracer = trace.getTracer(this.serviceName);
  }

  startSpan(name, parentContext = null) {
    const span = this.tracer.startSpan(name, {
      parent: parentContext
    });

    return span;
  }

  async traced(name, fn, parentContext = null) {
    const span = this.startSpan(name, parentContext);

    try {
      const result = await fn(span);
      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: error.message
      });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  }
}

// Utilisation dans service
class OrderService {
  constructor(db, tracing) {
    this.db = db;
    this.tracing = tracing;
  }

  async createOrder(orderData, parentSpan = null) {
    return this.tracing.traced(
      'OrderService.createOrder',
      async (span) => {
        // Ajouter attributs au span
        span.setAttribute('order.userId', orderData.userId);
        span.setAttribute('order.itemCount', orderData.items.length);
        span.setAttribute('order.total', orderData.total);

        // Étape 1: Valider
        await this.tracing.traced(
          'validateOrder',
          async () => {
            await this.validateOrder(orderData);
          },
          span
        );

        // Étape 2: Créer
        const order = await this.tracing.traced(
          'insertOrder',
          async () => {
            return this.db.collection('orders').insertOne({
              orderId: this.generateOrderId(),
              ...orderData,
              status: 'pending',
              createdAt: new Date()
            });
          },
          span
        );

        // Étape 3: Publier événement
        await this.tracing.traced(
          'publishEvent',
          async () => {
            await this.eventBus.publish('order.created', order);
          },
          span
        );

        span.addEvent('order_created', {
          orderId: order.orderId
        });

        return order;
      },
      parentSpan
    );
  }
}
```

## Configuration et déploiement

### 1. Docker Compose pour développement

```yaml
version: '3.8'

services:
  # API Gateway
  api-gateway:
    build: ./api-gateway
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
      - JWT_SECRET=your-secret-key
    depends_on:
      - user-service
      - order-service
      - product-service
    networks:
      - microservices

  # User Service
  user-service:
    build: ./user-service
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=development
      - MONGODB_URI=mongodb://mongodb-users:27017/users
      - KAFKA_BROKER=kafka:9092
    depends_on:
      - mongodb-users
      - kafka
    networks:
      - microservices

  mongodb-users:
    image: mongo:7.0
    ports:
      - "27017:27017"
    volumes:
      - users-data:/data/db
    command: --replSet rs0
    networks:
      - microservices

  # Order Service
  order-service:
    build: ./order-service
    ports:
      - "3002:3002"
    environment:
      - NODE_ENV=development
      - MONGODB_URI=mongodb://mongodb-orders:27017/orders
      - KAFKA_BROKER=kafka:9092
    depends_on:
      - mongodb-orders
      - kafka
    networks:
      - microservices

  mongodb-orders:
    image: mongo:7.0
    ports:
      - "27018:27017"
    volumes:
      - orders-data:/data/db
    command: --replSet rs0
    networks:
      - microservices

  # Product Service
  product-service:
    build: ./product-service
    ports:
      - "3003:3003"
    environment:
      - NODE_ENV=development
      - MONGODB_URI=mongodb://mongodb-products:27017/products
      - KAFKA_BROKER=kafka:9092
    depends_on:
      - mongodb-products
      - kafka
    networks:
      - microservices

  mongodb-products:
    image: mongo:7.0
    ports:
      - "27019:27017"
    volumes:
      - products-data:/data/db
    command: --replSet rs0
    networks:
      - microservices

  # Message Broker
  kafka:
    image: confluentinc/cp-kafka:latest
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    depends_on:
      - zookeeper
    networks:
      - microservices

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    networks:
      - microservices

  # Monitoring
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "5775:5775/udp"
      - "6831:6831/udp"
      - "6832:6832/udp"
      - "5778:5778"
      - "16686:16686"
      - "14268:14268"
      - "14250:14250"
      - "9411:9411"
    networks:
      - microservices

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - microservices

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3500:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    depends_on:
      - prometheus
    networks:
      - microservices

networks:
  microservices:
    driver: bridge

volumes:
  users-data:
  orders-data:
  products-data:
```

### 2. Kubernetes Deployment

```yaml
# User Service Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: microservices
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
        version: v1
    spec:
      containers:
      - name: user-service
        image: myregistry/user-service:v1.0.0
        ports:
        - containerPort: 3001
        env:
        - name: NODE_ENV
          value: "production"
        - name: MONGODB_URI
          valueFrom:
            secretKeyRef:
              name: mongodb-credentials
              key: users-uri
        - name: KAFKA_BROKER
          value: "kafka-service:9092"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3001
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3001
          initialDelaySeconds: 5
          periodSeconds: 5

---
# User Service
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: microservices
spec:
  selector:
    app: user-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3001
  type: ClusterIP

---
# MongoDB StatefulSet for User Service
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb-users
  namespace: microservices
spec:
  serviceName: mongodb-users
  replicas: 3
  selector:
    matchLabels:
      app: mongodb-users
  template:
    metadata:
      labels:
        app: mongodb-users
    spec:
      containers:
      - name: mongodb
        image: mongo:7.0
        ports:
        - containerPort: 27017
        command:
        - mongod
        - --replSet
        - rs0
        - --bind_ip_all
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: mongodb-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 20Gi

---
# MongoDB Service
apiVersion: v1
kind: Service
metadata:
  name: mongodb-users
  namespace: microservices
spec:
  selector:
    app: mongodb-users
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
  clusterIP: None  # Headless service for StatefulSet

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: microservices
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 3. Configuration MongoDB pour microservices

```javascript
// Configuration optimale par service
const mongoConfig = {
  // Connection pooling
  poolSize: 20,
  maxIdleTimeMS: 60000,

  // Retry configuration
  retryWrites: true,
  retryReads: true,

  // Write concern
  writeConcern: {
    w: 'majority',
    j: true,
    wtimeout: 5000
  },

  // Read preference
  readPreference: 'primaryPreferred',

  // Timeouts
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,

  // Monitoring
  monitorCommands: true,

  // Compression
  compressors: ['snappy', 'zlib']
};

// Connexion avec retry logic
class MongoDBConnection {
  constructor(uri, options) {
    this.uri = uri;
    this.options = options;
    this.client = null;
    this.maxRetries = 5;
    this.retryDelay = 1000;
  }

  async connect() {
    for (let i = 0; i < this.maxRetries; i++) {
      try {
        this.client = await MongoClient.connect(this.uri, this.options);

        // Test connection
        await this.client.db('admin').admin().ping();

        console.log('MongoDB connected successfully');

        // Setup change streams monitoring
        this.setupHealthCheck();

        return this.client;

      } catch (error) {
        console.error(`MongoDB connection attempt ${i + 1} failed:`, error);

        if (i < this.maxRetries - 1) {
          await this.sleep(this.retryDelay * (i + 1));
        } else {
          throw new Error('Failed to connect to MongoDB after retries');
        }
      }
    }
  }

  setupHealthCheck() {
    setInterval(async () => {
      try {
        await this.client.db('admin').admin().ping();
      } catch (error) {
        console.error('MongoDB health check failed:', error);
        // Trigger alerting
        this.emitHealthCheckFailed(error);
      }
    }, 30000);  // Every 30s
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## Patterns de résilience

### 1. Circuit Breaker

```javascript
class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.resetTimeout = options.resetTimeout || 60000;
    this.timeout = options.timeout || 5000;

    this.state = 'CLOSED';  // CLOSED, OPEN, HALF_OPEN
    this.failures = 0;
    this.nextAttempt = Date.now();
  }

  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }

      // Try to recover
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await this.withTimeout(fn(), this.timeout);

      // Success
      this.onSuccess();

      return result;

    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failures = 0;

    if (this.state === 'HALF_OPEN') {
      this.state = 'CLOSED';
      console.log('Circuit breaker recovered to CLOSED');
    }
  }

  onFailure() {
    this.failures++;

    if (this.failures >= this.failureThreshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.resetTimeout;
      console.error('Circuit breaker opened after failures');
    }
  }

  withTimeout(promise, timeout) {
    return Promise.race([
      promise,
      new Promise((_, reject) =>
        setTimeout(() => reject(new Error('Timeout')), timeout)
      )
    ]);
  }
}
```

### 2. Retry with Exponential Backoff

```javascript
class RetryPolicy {
  constructor(options = {}) {
    this.maxRetries = options.maxRetries || 3;
    this.initialDelay = options.initialDelay || 1000;
    this.maxDelay = options.maxDelay || 30000;
    this.backoffMultiplier = options.backoffMultiplier || 2;
  }

  async execute(fn, retryableErrors = []) {
    let lastError;

    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error;

        // Check if error is retryable
        if (!this.isRetryable(error, retryableErrors)) {
          throw error;
        }

        if (attempt < this.maxRetries) {
          const delay = this.calculateDelay(attempt);
          console.log(`Retry attempt ${attempt + 1} after ${delay}ms`);
          await this.sleep(delay);
        }
      }
    }

    throw lastError;
  }

  isRetryable(error, retryableErrors) {
    if (retryableErrors.length === 0) {
      // Par défaut: retry sur erreurs réseau et timeout
      return error.message.includes('timeout') ||
             error.message.includes('ECONNREFUSED') ||
             error.message.includes('ETIMEDOUT');
    }

    return retryableErrors.some(e =>
      error.message.includes(e) || error.code === e
    );
  }

  calculateDelay(attempt) {
    const delay = this.initialDelay * Math.pow(this.backoffMultiplier, attempt);

    // Add jitter
    const jitter = Math.random() * 0.1 * delay;

    return Math.min(delay + jitter, this.maxDelay);
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 3. Bulkhead Pattern

```javascript
class Bulkhead {
  constructor(maxConcurrent = 10) {
    this.maxConcurrent = maxConcurrent;
    this.running = 0;
    this.queue = [];
  }

  async execute(fn) {
    if (this.running >= this.maxConcurrent) {
      // Wait in queue
      await this.enqueue();
    }

    this.running++;

    try {
      return await fn();
    } finally {
      this.running--;
      this.dequeue();
    }
  }

  enqueue() {
    return new Promise(resolve => {
      this.queue.push(resolve);
    });
  }

  dequeue() {
    if (this.queue.length > 0) {
      const resolve = this.queue.shift();
      resolve();
    }
  }
}
```

## Monitoring et observabilité

### 1. Métriques par service

```javascript
const metrics = {
  // Service health
  'service.health': {
    type: 'gauge',
    description: 'Service health status (0=down, 1=up)',
    labels: ['service_name', 'version']
  },

  // Request metrics
  'http.requests.total': {
    type: 'counter',
    description: 'Total HTTP requests',
    labels: ['service', 'method', 'endpoint', 'status']
  },

  'http.request.duration': {
    type: 'histogram',
    description: 'HTTP request duration',
    labels: ['service', 'method', 'endpoint']
  },

  // MongoDB metrics
  'mongodb.operations.total': {
    type: 'counter',
    description: 'Total MongoDB operations',
    labels: ['service', 'operation', 'collection']
  },

  'mongodb.operation.duration': {
    type: 'histogram',
    description: 'MongoDB operation duration',
    labels: ['service', 'operation', 'collection']
  },

  'mongodb.connection.pool.size': {
    type: 'gauge',
    description: 'Connection pool size',
    labels: ['service']
  },

  // Event bus metrics
  'events.published.total': {
    type: 'counter',
    description: 'Total events published',
    labels: ['service', 'event_type']
  },

  'events.consumed.total': {
    type: 'counter',
    description: 'Total events consumed',
    labels: ['service', 'event_type', 'status']
  },

  // Circuit breaker
  'circuit.breaker.state': {
    type: 'gauge',
    description: 'Circuit breaker state (0=closed, 1=open, 2=half-open)',
    labels: ['service', 'target']
  }
};
```

## Checklist de déploiement

### ✅ Architecture

- [ ] Database per service implémenté
- [ ] API Gateway configuré
- [ ] Service discovery setup
- [ ] Message broker (Kafka/RabbitMQ)
- [ ] Event-driven communication
- [ ] Service mesh (optionnel)

### ✅ Data Management

- [ ] MongoDB per service avec isolation
- [ ] Change Streams pour events
- [ ] Data duplication strategy
- [ ] Saga pattern pour transactions distribuées
- [ ] CQRS si nécessaire
- [ ] Event sourcing si nécessaire

### ✅ Résilience

- [ ] Circuit breakers configurés
- [ ] Retry policies avec backoff
- [ ] Timeouts sur toutes les requêtes
- [ ] Bulkhead pattern
- [ ] Graceful degradation
- [ ] Health checks

### ✅ Communication

- [ ] REST APIs documentées (OpenAPI)
- [ ] gRPC si haute performance
- [ ] Event schemas versionnés
- [ ] Idempotency keys
- [ ] Correlation IDs

### ✅ Sécurité

- [ ] Authentication (JWT/OAuth2)
- [ ] Authorization (RBAC)
- [ ] API rate limiting
- [ ] TLS/HTTPS partout
- [ ] Secrets management
- [ ] Network policies

### ✅ Observabilité

- [ ] Distributed tracing (Jaeger/Zipkin)
- [ ] Centralized logging (ELK)
- [ ] Metrics collection (Prometheus)
- [ ] Dashboards (Grafana)
- [ ] Alerting configuré
- [ ] APM tool (optionnel)

### ✅ Déploiement

- [ ] Containerization (Docker)
- [ ] Orchestration (Kubernetes)
- [ ] CI/CD pipelines
- [ ] Blue-green ou canary deployments
- [ ] Auto-scaling configuré
- [ ] Resource limits

### ✅ Testing

- [ ] Unit tests (>80% coverage)
- [ ] Integration tests
- [ ] Contract tests
- [ ] End-to-end tests
- [ ] Load testing
- [ ] Chaos engineering

### ✅ Documentation

- [ ] Architecture diagrams
- [ ] API documentation
- [ ] Event schemas
- [ ] Runbooks
- [ ] Incident response procedures

## Conclusion

MongoDB s'intègre excellemment dans les architectures microservices grâce à :

**✅ Forces démontrées :**
- Database per service avec isolation complète
- Change Streams pour event-driven architecture
- Transactions ACID dans chaque service
- Schéma flexible pour évolution indépendante
- Replica Sets pour high availability
- MongoDB Atlas pour database as a service
- Lightweight et facile à containeriser
- Performance élevée pour chaque microservice

**⚠️ Considérations importantes :**
- Duplication de données nécessaire (trade-off autonomie)
- Eventual consistency entre services
- Saga pattern complexe à implémenter
- Monitoring distribué essentiel
- Coût opérationnel plus élevé (N databases)
- Testing plus complexe

**🎯 Patterns essentiels microservices :**
1. **Database per service** pour autonomie
2. **Change Streams** pour events
3. **Saga pattern** pour transactions distribuées
4. **CQRS** pour séparation read/write
5. **Circuit breaker** pour résilience
6. **API Gateway** pour routing centralisé
7. **Distributed tracing** pour observabilité

Cette architecture supporte des systèmes à très grande échelle avec centaines de microservices, déploiements indépendants, et évolution continue.

**Trade-offs clés :**
- Complexité opérationnelle vs scalabilité
- Autonomie services vs consistency
- Performance vs résilience
- Duplication données vs coupling

MongoDB offre le meilleur équilibre entre performance, flexibilité et facilité d'opération pour les architectures microservices modernes.

---

**Références :**
- "Building Microservices" - Sam Newman
- "Microservices Patterns" - Chris Richardson
- "Release It!" - Michael Nygard
- MongoDB Microservices Best Practices
- Kubernetes Documentation
- Martin Fowler's Microservices Guide

⏭️ [Event Sourcing avec MongoDB](/20-cas-usage-architectures/11-event-sourcing-mongodb.md)
