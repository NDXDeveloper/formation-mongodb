🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.1 Change Streams

## Introduction

Les **Change Streams** constituent l'une des fonctionnalités les plus puissantes de MongoDB pour construire des architectures modernes réactives et événementielles. Introduits dans MongoDB 3.6, les Change Streams permettent aux applications d'écouter en temps réel les modifications (insertions, mises à jour, suppressions) effectuées sur les collections, bases de données ou même l'ensemble d'un cluster.

Cette capacité transforme MongoDB d'une base de données traditionnelle en une source d'événements distribuée, ouvrant la voie à des patterns architecturaux comme Event Sourcing, CQRS, et Event-Driven Architecture.

---

## Qu'est-ce qu'un Change Stream ?

Un Change Stream est un **flux de notifications push** qui diffuse les événements de modification de données en temps quasi-réel. Contrairement au polling traditionnel (interrogation périodique de la base), les Change Streams utilisent une approche réactive où MongoDB pousse activement les changements vers les clients abonnés.

### Architecture conceptuelle

```
┌─────────────────────────────────────────────────────────────┐
│                    MongoDB Replica Set                      │
│                                                             │
│  ┌──────────────┐         ┌──────────────┐                  │
│  │   Primary    │────────▶│    Oplog     │                  │
│  │              │  writes │ (Operations  │                  │
│  │              │         │    Log)      │                  │
│  └──────────────┘         └───────┬──────┘                  │
│                                   │                         │
│                                   │ Tail & Transform        │
│                                   ▼                         │
│                          ┌────────────────┐                 │
│                          │ Change Streams │                 │
│                          │    Engine      │                 │
│                          └────────┬───────┘                 │
└───────────────────────────────────┼─────────────────────────┘
                                    │
                                    │ Push Events
                                    ▼
                    ┌───────────────────────────────┐
                    │   Subscribed Applications     │
                    │                               │
                    │  ┌─────────┐  ┌─────────┐     │
                    │  │  App 1  │  │  App 2  │     │
                    │  │(Node.js)│  │(Python) │     │
                    │  └─────────┘  └─────────┘     │
                    └───────────────────────────────┘
```

**Flux de données :**
1. Les opérations d'écriture sont enregistrées dans l'**Oplog** (Operations Log)
2. Le moteur Change Streams **surveille** l'Oplog en continu
3. Les événements sont **transformés** en documents normalisés
4. Les événements sont **poussés** vers les clients abonnés
5. Les clients **réagissent** aux changements en temps réel

---

## Caractéristiques fondamentales

### 1. Temps réel avec latence minimale

Les Change Streams offrent une latence typique de **50-200 millisecondes** entre l'écriture et la notification, selon la charge du système et la configuration réseau.

```javascript
// Exemple de latence mesurée
const startTime = Date.now();

// Insertion
await collection.insertOne({ status: "pending", timestamp: new Date() });

// Notification reçue dans le Change Stream
changeStream.on('change', (change) => {
  const latency = Date.now() - change.fullDocument.timestamp.getTime();
  console.log(`Latency: ${latency}ms`); // Typiquement 50-200ms
});
```

### 2. Garanties de livraison

Les Change Streams garantissent :
- **Au moins une fois (at-least-once delivery)** : Chaque événement est livré au moins une fois
- **Ordre total** : Les événements sont livrés dans l'ordre exact des opérations sur l'Oplog
- **Durabilité** : Les événements sont basés sur l'Oplog répliqué (pas de perte en cas de failover)

### 3. Reprise automatique (Resumability)

En cas de déconnexion, les Change Streams peuvent reprendre exactement où ils s'étaient arrêtés grâce aux **resume tokens**.

```javascript
let resumeToken;

const changeStream = collection.watch();

changeStream.on('change', (change) => {
  // Sauvegarder le token pour reprise ultérieure
  resumeToken = change._id;
  processChange(change);
});

// En cas de reconnexion
const resumedStream = collection.watch([], {
  resumeAfter: resumeToken
});
```

### 4. Filtrage côté serveur

Les Change Streams supportent des pipelines d'agrégation pour filtrer les événements directement sur le serveur, réduisant la bande passante et la charge client.

```javascript
// Filtrer uniquement les insertions de commandes > 1000€
const pipeline = [
  {
    $match: {
      operationType: 'insert',
      'fullDocument.amount': { $gt: 1000 }
    }
  }
];

const changeStream = collection.watch(pipeline);
```

---

## Types d'événements

Un Change Stream peut émettre plusieurs types d'événements, chacun correspondant à une opération différente :

| Type d'événement | Description | Disponibilité |
|------------------|-------------|---------------|
| `insert` | Nouveau document inséré | Toujours |
| `update` | Document modifié | Toujours |
| `replace` | Document remplacé entièrement | Toujours |
| `delete` | Document supprimé | Toujours |
| `drop` | Collection supprimée | Toujours |
| `rename` | Collection renommée | Toujours |
| `dropDatabase` | Base de données supprimée | Toujours |
| `invalidate` | Stream invalidé (ex: collection droppée) | Toujours |

### Structure d'un événement Change Stream

```javascript
{
  _id: {
    // Resume token (opaque, utilisé pour reprise)
    _data: "8264F3A123000000012B022C0100296E5A1004..."
  },
  operationType: "insert",
  clusterTime: Timestamp(1, 1643723456),
  wallTime: ISODate("2024-02-01T14:30:56.123Z"),
  fullDocument: {
    _id: ObjectId("6472a5b3c8f9e2d45a1b3c4d"),
    orderId: "ORD-2024-001",
    customerId: "CUST-5678",
    amount: 1250.00,
    status: "pending",
    items: [/* ... */]
  },
  ns: {
    db: "ecommerce",
    coll: "orders"
  },
  documentKey: {
    _id: ObjectId("6472a5b3c8f9e2d45a1b3c4d")
  }
}
```

**Champs clés :**
- `_id` : Token de reprise unique pour cet événement
- `operationType` : Type d'opération (insert, update, delete, etc.)
- `clusterTime` : Timestamp Oplog (ordre global)
- `wallTime` : Horodatage réel (wall-clock time)
- `fullDocument` : Document complet (selon configuration)
- `ns` : Namespace (base de données + collection)
- `documentKey` : Identifiant du document concerné

---

## Niveaux de granularité

Les Change Streams peuvent être ouverts à trois niveaux différents :

### 1. Collection unique

```javascript
// Surveiller une collection spécifique
const changeStream = db.collection('orders').watch();

changeStream.on('change', (change) => {
  console.log('Changement dans orders:', change.operationType);
});
```

**Cas d'usage :** Synchronisation cache, invalidation, notifications métier spécifiques

### 2. Base de données complète

```javascript
// Surveiller toutes les collections d'une base
const changeStream = db.watch();

changeStream.on('change', (change) => {
  console.log(`Changement dans ${change.ns.coll}:`, change.operationType);
});
```

**Cas d'usage :** Audit global, réplication vers data warehouse, CDC complet

### 3. Cluster entier (deployment)

```javascript
// Surveiller toutes les bases de données du cluster
const changeStream = client.watch();

changeStream.on('change', (change) => {
  console.log(`Changement dans ${change.ns.db}.${change.ns.coll}`);
});
```

**Cas d'usage :** Monitoring centralisé, réplication multi-tenant, observabilité

---

## Exemple complet : Système de notification en temps réel

Voici un exemple d'implémentation production-ready d'un système de notification basé sur Change Streams :

```javascript
const { MongoClient } = require('mongodb');
const WebSocket = require('ws');

class RealtimeNotificationService {
  constructor(mongoUri, wsPort = 8080) {
    this.mongoUri = mongoUri;
    this.wsPort = wsPort;
    this.wss = null;
    this.changeStream = null;
    this.resumeToken = null;
  }

  async start() {
    // Connexion MongoDB
    this.client = await MongoClient.connect(this.mongoUri, {
      useUnifiedTopology: true,
      retryWrites: true
    });

    const db = this.client.db('myapp');
    const collection = db.collection('notifications');

    // Configuration du Change Stream avec pipeline
    const pipeline = [
      {
        $match: {
          operationType: 'insert',
          'fullDocument.read': false  // Seulement notifications non lues
        }
      },
      {
        $project: {
          _id: 1,
          operationType: 1,
          fullDocument: 1,
          'fullDocument.userId': 1,
          'fullDocument.type': 1,
          'fullDocument.message': 1,
          'fullDocument.timestamp': 1
        }
      }
    ];

    // Démarrer le Change Stream
    this.changeStream = collection.watch(pipeline, {
      fullDocument: 'updateLookup',
      resumeAfter: this.resumeToken  // Reprise si disponible
    });

    // Gestion des événements
    this.changeStream.on('change', (change) => {
      this.handleChange(change);
    });

    this.changeStream.on('error', (error) => {
      console.error('Change Stream error:', error);
      this.reconnect();
    });

    // Serveur WebSocket
    this.wss = new WebSocket.Server({ port: this.wsPort });

    this.wss.on('connection', (ws, req) => {
      console.log('Client WebSocket connecté');

      // Authentification du client (userId depuis token JWT)
      const userId = this.authenticateClient(req);
      ws.userId = userId;

      ws.on('close', () => {
        console.log(`Client ${userId} déconnecté`);
      });
    });

    console.log(`Service démarré sur port ${this.wsPort}`);
  }

  handleChange(change) {
    // Sauvegarder le resume token
    this.resumeToken = change._id;

    const notification = change.fullDocument;
    const userId = notification.userId;

    // Diffuser vers le client WebSocket approprié
    this.wss.clients.forEach((client) => {
      if (client.userId === userId && client.readyState === WebSocket.OPEN) {
        client.send(JSON.stringify({
          type: 'notification',
          data: {
            id: notification._id,
            type: notification.type,
            message: notification.message,
            timestamp: notification.timestamp
          }
        }));
      }
    });

    // Logging pour analytics
    console.log(`Notification envoyée à user ${userId}: ${notification.type}`);
  }

  authenticateClient(req) {
    // Extraction et validation du JWT (simplifié)
    const token = req.headers['sec-websocket-protocol'];
    // Validation réelle ici...
    return 'user123'; // ID extrait du token
  }

  async reconnect() {
    console.log('Reconnexion au Change Stream...');

    // Attendre avant de reconnecter
    await new Promise(resolve => setTimeout(resolve, 5000));

    // Fermer proprement
    if (this.changeStream) {
      await this.changeStream.close();
    }

    // Redémarrer avec resume token
    await this.start();
  }

  async stop() {
    if (this.changeStream) {
      await this.changeStream.close();
    }
    if (this.client) {
      await this.client.close();
    }
    if (this.wss) {
      this.wss.close();
    }
  }
}

// Utilisation
const service = new RealtimeNotificationService(
  'mongodb://localhost:27017/?replicaSet=rs0'
);

service.start().catch(console.error);

// Gestion propre de l'arrêt
process.on('SIGINT', async () => {
  console.log('Arrêt du service...');
  await service.stop();
  process.exit(0);
});
```

### Côté client (JavaScript)

```javascript
// Connexion WebSocket côté frontend
class NotificationClient {
  constructor(wsUrl, jwtToken) {
    this.wsUrl = wsUrl;
    this.jwtToken = jwtToken;
    this.ws = null;
    this.reconnectDelay = 1000;
  }

  connect() {
    // Connexion avec authentification
    this.ws = new WebSocket(this.wsUrl, this.jwtToken);

    this.ws.onopen = () => {
      console.log('Connecté au service de notifications');
      this.reconnectDelay = 1000; // Reset du délai
    };

    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data);

      if (message.type === 'notification') {
        this.displayNotification(message.data);
      }
    };

    this.ws.onerror = (error) => {
      console.error('WebSocket error:', error);
    };

    this.ws.onclose = () => {
      console.log('Connexion fermée, reconnexion...');
      setTimeout(() => this.connect(), this.reconnectDelay);
      this.reconnectDelay = Math.min(this.reconnectDelay * 2, 30000);
    };
  }

  displayNotification(data) {
    // Affichage de la notification (browser notification API)
    if (Notification.permission === 'granted') {
      new Notification(data.type, {
        body: data.message,
        icon: '/icon.png',
        timestamp: new Date(data.timestamp)
      });
    }

    // Mise à jour de l'UI
    this.updateNotificationBadge();
  }

  updateNotificationBadge() {
    // Incrémenter le compteur de notifications
    const badge = document.getElementById('notification-badge');
    const count = parseInt(badge.textContent) || 0;
    badge.textContent = count + 1;
    badge.style.display = 'block';
  }
}

// Utilisation
const client = new NotificationClient(
  'ws://localhost:8080',
  localStorage.getItem('jwt_token')
);

client.connect();
```

---

## Architecture d'entreprise avec Change Streams

### Pattern 1 : Event Bus distribué

```javascript
// Hub central qui redistribue les événements
class EventBusHub {
  constructor() {
    this.subscribers = new Map();
  }

  async initialize(mongoUri) {
    this.client = await MongoClient.connect(mongoUri);
    const db = this.client.db('main');

    // Surveiller toutes les collections critiques
    const collections = ['orders', 'payments', 'inventory', 'users'];

    collections.forEach(collName => {
      const changeStream = db.collection(collName).watch();

      changeStream.on('change', (change) => {
        this.broadcast({
          source: collName,
          event: change.operationType,
          data: change.fullDocument,
          timestamp: change.wallTime
        });
      });
    });
  }

  subscribe(eventType, callback) {
    if (!this.subscribers.has(eventType)) {
      this.subscribers.set(eventType, []);
    }
    this.subscribers.get(eventType).push(callback);
  }

  broadcast(event) {
    const eventType = `${event.source}.${event.event}`;
    const handlers = this.subscribers.get(eventType) || [];

    handlers.forEach(handler => {
      try {
        handler(event);
      } catch (error) {
        console.error(`Error in handler for ${eventType}:`, error);
      }
    });
  }
}

// Services qui s'abonnent
const eventBus = new EventBusHub();
await eventBus.initialize('mongodb://localhost:27017/?replicaSet=rs0');

// Service de cache
eventBus.subscribe('orders.insert', (event) => {
  cache.set(`order:${event.data._id}`, event.data);
});

eventBus.subscribe('orders.update', (event) => {
  cache.invalidate(`order:${event.data._id}`);
});

// Service d'email
eventBus.subscribe('orders.insert', async (event) => {
  await emailService.sendOrderConfirmation(event.data);
});

// Service d'analytics
eventBus.subscribe('payments.insert', (event) => {
  analytics.track('payment_completed', {
    amount: event.data.amount,
    currency: event.data.currency
  });
});
```

### Pattern 2 : CDC (Change Data Capture) vers Data Warehouse

```javascript
// Réplication vers Snowflake/BigQuery
class CDCPipeline {
  constructor(mongoUri, warehouseConfig) {
    this.mongoUri = mongoUri;
    this.warehouse = warehouseConfig;
    this.buffer = [];
    this.flushInterval = 5000; // 5 secondes
    this.batchSize = 1000;
  }

  async start() {
    const client = await MongoClient.connect(this.mongoUri);
    const db = client.db('production');

    // Pipeline de transformation
    const pipeline = [
      {
        $match: {
          operationType: { $in: ['insert', 'update', 'delete'] }
        }
      },
      {
        $project: {
          operation: '$operationType',
          table: '$ns.coll',
          data: '$fullDocument',
          timestamp: '$wallTime',
          key: '$documentKey._id'
        }
      }
    ];

    const changeStream = db.watch(pipeline);

    changeStream.on('change', (change) => {
      this.buffer.push(change);

      if (this.buffer.length >= this.batchSize) {
        this.flush();
      }
    });

    // Flush périodique
    setInterval(() => this.flush(), this.flushInterval);
  }

  async flush() {
    if (this.buffer.length === 0) return;

    const batch = [...this.buffer];
    this.buffer = [];

    try {
      // Insertion batch vers data warehouse
      await this.warehouse.insertBatch(batch);
      console.log(`${batch.length} événements répliqués vers warehouse`);
    } catch (error) {
      console.error('Erreur réplication:', error);
      // Réinsérer dans le buffer pour retry
      this.buffer.unshift(...batch);
    }
  }
}
```

### Pattern 3 : Cache invalidation intelligente

```javascript
// Synchronisation automatique Redis <-> MongoDB
class SmartCacheSync {
  constructor(mongoUri, redisClient) {
    this.redisClient = redisClient;
    this.mongoUri = mongoUri;
  }

  async initialize() {
    const client = await MongoClient.connect(this.mongoUri);
    const db = client.db('myapp');

    // Surveiller uniquement les updates de documents mis en cache
    const pipeline = [
      {
        $match: {
          $or: [
            { operationType: 'update' },
            { operationType: 'replace' },
            { operationType: 'delete' }
          ]
        }
      }
    ];

    const changeStream = db.watch(pipeline);

    changeStream.on('change', async (change) => {
      const cacheKey = this.buildCacheKey(
        change.ns.coll,
        change.documentKey._id
      );

      if (change.operationType === 'delete') {
        // Supprimer du cache
        await this.redisClient.del(cacheKey);
        console.log(`Cache invalidated: ${cacheKey}`);
      } else {
        // Mettre à jour le cache avec nouvelles données
        const freshData = change.fullDocument;
        await this.redisClient.setex(
          cacheKey,
          3600, // TTL 1h
          JSON.stringify(freshData)
        );
        console.log(`Cache updated: ${cacheKey}`);
      }
    });
  }

  buildCacheKey(collection, id) {
    return `${collection}:${id}`;
  }
}

// Utilisation
const redis = require('redis').createClient();
const cacheSync = new SmartCacheSync(
  'mongodb://localhost:27017/?replicaSet=rs0',
  redis
);

await cacheSync.initialize();
```

---

## Considérations de performance

### Overhead et impact

Les Change Streams introduisent un overhead minimal :
- **CPU** : ~2-5% supplémentaire sur le Primary
- **Mémoire** : ~10-50 Mo par stream actif
- **Réseau** : Proportionnel au volume de changements
- **Oplog** : Aucun impact (lecture en lecture seule)

### Limites et quotas

| Paramètre | Limite | Notes |
|-----------|--------|-------|
| Change Streams simultanés | 1000 par mongos | Limite soft, ajustable |
| Taille d'événement | 16 Mo | Limite BSON |
| Durée de vie resume token | Taille de l'Oplog | Variable selon config |
| Latency P50 | < 100 ms | Réseau optimal |
| Latency P99 | < 500 ms | Charge normale |

### Optimisations recommandées

```javascript
// 1. Filtrer tôt dans le pipeline
const optimizedPipeline = [
  {
    $match: {
      operationType: 'insert',
      'fullDocument.status': 'pending'
    }
  },
  // Transformation après filtrage
  { $project: { /* ... */ } }
];

// 2. Utiliser fullDocument: 'updateLookup' uniquement si nécessaire
const changeStream = collection.watch(pipeline, {
  fullDocument: 'default'  // Plus performant si on n'a pas besoin du doc complet
});

// 3. Batch processing pour réduire la charge
const eventBuffer = [];
changeStream.on('change', (change) => {
  eventBuffer.push(change);

  if (eventBuffer.length >= 100) {
    processEventsBatch(eventBuffer);
    eventBuffer.length = 0;
  }
});
```

---

## Prérequis et limitations

### Prérequis techniques

✅ **Obligatoire :**
- MongoDB ≥ 3.6
- **Replica Set** ou **Sharded Cluster** (standalone non supporté)
- Driver compatible (Node.js ≥ 3.1, Python ≥ 3.6, Java ≥ 3.6, etc.)

⚠️ **Recommandé :**
- MongoDB ≥ 4.0 (meilleure gestion des transactions)
- Replica Set avec au moins 3 membres
- Oplog dimensionné (≥ 5% de la taille des données)

### Limitations importantes

❌ **Ne peut pas :**
- Fonctionner sur un MongoDB standalone (nécessite replica set minimum)
- Garantir l'ordre entre shards différents dans un cluster shardé
- Capturer les changements avant l'ouverture du stream (pas de replay historique)
- Capturer les opérations administratives (createCollection, dropCollection sauf via événements spéciaux)

✅ **Peut :**
- Reprendre après interruption (via resume tokens)
- Filtrer sur plusieurs critères complexes
- Transformer les événements via pipelines d'agrégation
- Fonctionner sur collection, database ou cluster entier

---

## Monitoring et observabilité

### Métriques clés à surveiller

```javascript
// Monitoring d'un Change Stream
class MonitoredChangeStream {
  constructor(collection) {
    this.collection = collection;
    this.metrics = {
      eventsReceived: 0,
      eventsProcessed: 0,
      errors: 0,
      latencySum: 0,
      latencyCount: 0
    };
  }

  async start() {
    const changeStream = this.collection.watch();

    changeStream.on('change', async (change) => {
      const startTime = Date.now();
      this.metrics.eventsReceived++;

      try {
        await this.processChange(change);
        this.metrics.eventsProcessed++;

        // Latence
        const latency = Date.now() - startTime;
        this.metrics.latencySum += latency;
        this.metrics.latencyCount++;
      } catch (error) {
        this.metrics.errors++;
        console.error('Processing error:', error);
      }
    });

    // Métriques périodiques
    setInterval(() => this.logMetrics(), 60000);
  }

  logMetrics() {
    const avgLatency = this.metrics.latencyCount > 0
      ? this.metrics.latencySum / this.metrics.latencyCount
      : 0;

    console.log({
      eventsReceived: this.metrics.eventsReceived,
      eventsProcessed: this.metrics.eventsProcessed,
      errors: this.metrics.errors,
      avgLatency: `${avgLatency.toFixed(2)}ms`,
      successRate: `${((this.metrics.eventsProcessed / this.metrics.eventsReceived) * 100).toFixed(2)}%`
    });
  }
}
```

---

## Sécurité et bonnes pratiques

### 1. Contrôle d'accès

```javascript
// L'utilisateur doit avoir les privilèges appropriés
{
  role: "customChangeStreamRole",
  privileges: [
    {
      resource: { db: "mydb", collection: "mycollection" },
      actions: ["find", "changeStream"]
    }
  ]
}
```

### 2. Gestion des secrets

```javascript
// Ne jamais hardcoder les credentials
const mongoUri = process.env.MONGODB_URI;
const changeStream = client.watch();

// Chiffrement TLS obligatoire en production
const secureClient = new MongoClient(mongoUri, {
  tls: true,
  tlsCAFile: '/path/to/ca.pem',
  tlsCertificateKeyFile: '/path/to/client.pem'
});
```

### 3. Idempotence

```javascript
// Traiter les événements de manière idempotente
changeStream.on('change', async (change) => {
  const eventId = change._id._data;

  // Vérifier si déjà traité
  const processed = await redis.get(`processed:${eventId}`);
  if (processed) {
    console.log(`Event ${eventId} already processed, skipping`);
    return;
  }

  // Traiter
  await processEvent(change);

  // Marquer comme traité
  await redis.setex(`processed:${eventId}`, 86400, '1'); // 24h TTL
});
```

---

## Comparaison avec d'autres approches

| Approche | Latence | Overhead | Fiabilité | Complexité | Cas d'usage |
|----------|---------|----------|-----------|------------|-------------|
| **Change Streams** | 50-200ms | Faible | Élevée | Moyenne | Production, temps réel |
| **Polling** | 1-60s | Élevé | Moyenne | Faible | Dev, non critique |
| **Triggers** | < 100ms | Moyen | Élevée | Faible | Atlas uniquement |
| **Oplog Tailing** | < 50ms | Très faible | Moyenne | Élevée | Cas avancés spécifiques |
| **Message Queue** | Variable | Élevé | Élevée | Élevée | Architecture distribuée |

### Quand utiliser Change Streams ?

✅ **Utiliser quand :**
- Besoin de réactivité temps réel (< 1s)
- Architecture événementielle / Event-Driven
- Synchronisation de systèmes externes
- Invalidation de cache automatique
- Notifications push utilisateur
- CDC (Change Data Capture) vers data warehouse

❌ **Ne pas utiliser quand :**
- MongoDB standalone (pas de replica set)
- Latence > 5 secondes acceptable (polling suffit)
- Transformation complexe requise (préférer ETL batch)
- Oplog trop petit ou instable

---

## Prochaines sections

Cette introduction aux Change Streams a couvert les concepts fondamentaux, l'architecture, et des exemples d'implémentation avancés. Les sections suivantes approfondiront :

- **16.1.1 Principes et cas d'usage** : Patterns architecturaux détaillés, Event Sourcing, CQRS
- **16.1.2 Configuration et filtres** : Pipelines avancés, optimisations, options
- **16.1.3 Resume tokens** : Gestion de la résilience, reprise après panne, persistance

---

## Ressources complémentaires

- [Documentation officielle MongoDB Change Streams](https://www.mongodb.com/docs/manual/changeStreams/)
- [Blog MongoDB : Building with Change Streams](https://www.mongodb.com/blog)
- [Driver Specifications : Change Streams](https://github.com/mongodb/specifications/blob/master/source/change-streams)

---


⏭️ [Principes et cas d'usage](/16-fonctionnalites-avancees/01.1-principes-cas-usage.md)
