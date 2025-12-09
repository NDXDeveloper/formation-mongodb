🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.11 Event Sourcing avec MongoDB

## Introduction

L'Event Sourcing est un pattern architectural qui stocke l'état d'une application comme une séquence d'événements immuables, plutôt que comme l'état courant. Chaque changement d'état est capturé comme un événement daté et ordonné.

**Principes fondamentaux :**
- **Events as source of truth** : Les événements sont la source de vérité
- **Immutabilité** : Les événements ne sont jamais modifiés ou supprimés
- **Ordering** : Séquence stricte des événements par aggregate
- **Audit trail complet** : Historique intégral de tous les changements
- **Time travel** : Reconstitution de l'état à n'importe quel moment
- **Event replay** : Reconstruction de l'état depuis les événements

**Avantages :**
- ✅ Audit trail parfait pour conformité
- ✅ Debugging facilité (replay pour reproduire bugs)
- ✅ Analytics sur données historiques
- ✅ Multiple read models depuis mêmes events
- ✅ Temporal queries (état à date T)
- ✅ Business insights depuis événements métier

**Challenges :**
- ❌ Complexité accrue
- ❌ Volume de données important
- ❌ Eventual consistency
- ❌ Event schema evolution
- ❌ Query complexes sur l'état actuel

MongoDB est excellent pour Event Sourcing grâce à :
- **Change Streams** : Notifications événements en temps réel
- **Transactions** : Atomicité append events + update projections
- **Flexible schema** : Evolution des événements
- **Aggregations** : Projections et analytics
- **Performance** : Append-only optimisé
- **TTL** : Archivage automatique anciens événements

## Architecture de référence

### Stack Event Sourcing

```
┌──────────────────────────────────────────────────────────┐
│                    Command Sources                       │
│           API • UI • External Systems • Jobs             │
└────────────────────────┬─────────────────────────────────┘
                         │
                         │ Commands
                         │
            ┌────────────▼──────────────┐
            │   Command Handler         │
            │  • Validate Command       │
            │  • Load Aggregate         │
            │  • Execute Business Logic │
            │  • Generate Events        │
            └────────────┬──────────────┘
                         │
                         │ Events
                         │
            ┌────────────▼─────────────┐
            │      Event Store         │
            │      (MongoDB)           │
            │                          │
            │  ┌──────────────────┐    │
            │  │  events          │    │
            │  │  • Immutable     │    │
            │  │  • Append-only   │    │
            │  │  • Versioned     │    │
            │  └──────────────────┘    │
            │                          │
            │  ┌──────────────────┐    │
            │  │  snapshots       │    │
            │  │  • Cached state  │    │
            │  │  • Performance   │    │
            │  └──────────────────┘    │
            └────────────┬─────────────┘
                         │
                         │ Change Streams
                         │
         ┌───────────────┼─────────────────┐
         │               │                 │
    ┌────▼─────┐   ┌────▼─────┐      ┌─────▼──────┐
    │Projection│   │Projection│      │ Projection │
    │ Handler  │   │ Handler  │      │  Handler   │
    │    #1    │   │    #2    │      │     #3     │
    └────┬─────┘   └────┬─────┘      └─────┬──────┘
         │              │                  │
    ┌────▼─────┐   ┌────▼─────┐      ┌─────▼──────┐
    │Read Model│   │Read Model│      │ Read Model │
    │  (Users) │   │ (Orders) │      │(Analytics) │
    └──────────┘   └──────────┘      └────────────┘
         │              │                   │
         └──────────────┴───────────────────┘
                         │
            ┌────────────▼──────────────┐
            │      Query Service        │
            │  • Read from Read Models  │
            │  • Optimized for Queries  │
            │  • Eventual Consistency   │
            └────────────┬──────────────┘
                         │
                         │ Queries
                         │
┌────────────────────────▼─────────────────────────────────┐
│                     Consumers                            │
│              API • UI • Reports • Dashboards             │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                  Supporting Services                     │
├──────────────────────────────────────────────────────────┤
│  Event Replay Service  •  Snapshot Service               │
│  Event Migration Service  •  Temporal Query Service      │
└──────────────────────────────────────────────────────────┘
```

### Composants architecturaux

#### 1. Event Store
**Technologies :** MongoDB collection optimisée append-only

**Justification :**
- Stockage immuable des événements
- Performance excellente pour writes séquentiels
- Indexes sur aggregate + version pour reconstruction
- Change Streams pour propagation temps réel

#### 2. Command Handler
**Responsabilités :**
- Valider commandes
- Charger aggregate depuis events
- Exécuter logique métier
- Générer nouveaux événements
- Persister dans event store

#### 3. Projection Handlers
**Responsabilités :**
- Écouter événements (Change Streams)
- Maintenir read models dénormalisés
- Optimiser pour queries spécifiques
- Eventual consistency acceptable

#### 4. Snapshot Service
**Responsabilités :**
- Créer snapshots périodiques
- Optimiser reconstruction d'aggregates
- Balance entre taille et fréquence

## Modélisation des événements

### 1. Event Store Schema

```javascript
// Collection: events (append-only, immutable)
{
  _id: ObjectId("..."),

  // Event identification
  eventId: "evt_abc123def456ghi789",
  eventType: "account.credited",
  eventVersion: "1.0",  // Schema version

  // Aggregate identification
  aggregateType: "account",
  aggregateId: "acc_xyz789",
  aggregateVersion: 42,  // Séquence d'événements pour cet aggregate

  // Event data (business data)
  data: {
    amount: NumberDecimal("1000.00"),
    currency: "USD",
    transactionId: "txn_abc123",
    reason: "salary_deposit",

    // Balance après événement (dénormalisation pour queries)
    balanceAfter: NumberDecimal("5432.50")
  },

  // Metadata
  metadata: {
    // Qui a déclenché
    userId: "user_123",
    userName: "John Doe",

    // Quand
    timestamp: ISODate("2024-12-09T14:30:15.234Z"),

    // Comment
    source: "web_app",
    ipAddress: "192.168.1.100",
    userAgent: "Mozilla/5.0...",

    // Contexte
    correlationId: "corr_abc123",  // Request ID
    causationId: "evt_previous_event",  // Event qui a causé celui-ci

    // Tracing
    traceId: "trace_xyz789",
    spanId: "span_abc123"
  },

  // Timestamp précis
  timestamp: ISODate("2024-12-09T14:30:15.234Z"),

  // Pour ordering strict
  sequence: NumberLong("12345678"),  // Global sequence

  // Flags
  isSnapshot: false,
  processed: false,  // Pour projections
  archived: false
}

// Index critiques pour Event Store
db.events.createIndex(
  { aggregateType: 1, aggregateId: 1, aggregateVersion: 1 },
  { unique: true, name: 'aggregate_ordering' }
);

db.events.createIndex(
  { eventId: 1 },
  { unique: true, name: 'event_id' }
);

db.events.createIndex(
  { sequence: 1 },
  { unique: true, name: 'global_sequence' }
);

db.events.createIndex(
  { timestamp: 1 },
  { name: 'timestamp' }
);

db.events.createIndex(
  { eventType: 1, timestamp: 1 },
  { name: 'event_type_time' }
);

// Pour projections
db.events.createIndex(
  { processed: 1, sequence: 1 },
  { name: 'projection_processing' }
);

// Pour correlation
db.events.createIndex(
  { 'metadata.correlationId': 1, timestamp: 1 },
  { name: 'correlation' }
);
```

### 2. Snapshots Schema

```javascript
// Collection: snapshots
// Optimisation: éviter replay complet pour aggregates avec beaucoup d'events
{
  _id: ObjectId("..."),

  // Aggregate identification
  aggregateType: "account",
  aggregateId: "acc_xyz789",

  // Version du snapshot
  version: 42,  // Correspond à aggregateVersion du dernier event inclus

  // État complet de l'aggregate à cette version
  state: {
    accountId: "acc_xyz789",
    balance: NumberDecimal("5432.50"),
    currency: "USD",
    status: "active",

    // Toutes les propriétés de l'aggregate
    owner: {
      userId: "user_123",
      name: "John Doe"
    },

    createdAt: ISODate("2024-01-15T10:00:00Z"),
    lastModified: ISODate("2024-12-09T14:30:15.234Z")
  },

  // Metadata
  createdAt: ISODate("2024-12-09T15:00:00Z"),
  createdBy: "snapshot_service",

  // Stats
  eventCount: 42,  // Nombre d'events agrégés

  // Pour cleanup
  expiresAt: ISODate("2025-01-09T00:00:00Z")  // Garder 1 mois
}

// Index pour snapshots
db.snapshots.createIndex(
  { aggregateType: 1, aggregateId: 1, version: -1 },
  { name: 'aggregate_latest_snapshot' }
);

db.snapshots.createIndex(
  { aggregateType: 1, aggregateId: 1 },
  { unique: true, name: 'aggregate_unique' }
);

// TTL pour cleanup automatique
db.snapshots.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0, name: 'ttl' }
);
```

### 3. Projections / Read Models

```javascript
// Collection: accounts_read (read model pour queries)
{
  _id: ObjectId("..."),
  accountId: "acc_xyz789",

  // État actuel (dénormalisé)
  balance: NumberDecimal("5432.50"),
  currency: "USD",
  status: "active",

  owner: {
    userId: "user_123",
    name: "John Doe",
    email: "john.doe@example.com"
  },

  // Metadata
  createdAt: ISODate("2024-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:15.234Z"),

  // Version tracking
  lastEventVersion: 42,  // Dernier event appliqué
  lastEventId: "evt_abc123def456ghi789",

  // Stats agrégées
  stats: {
    totalCredits: NumberDecimal("15000.00"),
    totalDebits: NumberDecimal("9567.50"),
    transactionCount: 42,
    lastTransactionAt: ISODate("2024-12-09T14:30:15.234Z")
  }
}

// Index pour read model
db.accounts_read.createIndex({ accountId: 1 }, { unique: true });
db.accounts_read.createIndex({ 'owner.userId': 1 });
db.accounts_read.createIndex({ status: 1 });
db.accounts_read.createIndex({ balance: 1 });
```

### 4. Event Types Catalog

```javascript
// Collection: event_schemas (event registry)
{
  _id: ObjectId("..."),

  eventType: "account.credited",
  version: "1.0",

  // JSON Schema pour validation
  schema: {
    type: "object",
    required: ["amount", "currency", "transactionId"],
    properties: {
      amount: {
        type: "number",
        minimum: 0
      },
      currency: {
        type: "string",
        enum: ["USD", "EUR", "GBP"]
      },
      transactionId: {
        type: "string"
      },
      reason: {
        type: "string"
      }
    }
  },

  // Description
  description: "Account was credited with funds",

  // Projections qui consomment cet event
  projections: ["accounts_read", "transactions_read", "analytics"],

  // Versioning
  previousVersion: null,  // "0.9" si upgrade
  deprecatedAt: null,

  createdAt: ISODate("2024-01-01T00:00:00Z")
}
```

## Event Store Service

### 1. Append Events

```javascript
class EventStore {
  constructor(db) {
    this.db = db;
    this.sequenceCounter = null;
  }

  async initialize() {
    // Initialiser sequence counter
    await this.db.collection('counters').updateOne(
      { _id: 'event_sequence' },
      { $setOnInsert: { value: 0 } },
      { upsert: true }
    );
  }

  async appendEvent(event) {
    const session = this.db.client.startSession();

    try {
      let persistedEvent;

      await session.withTransaction(async () => {
        // 1. Vérifier version optimistic locking
        const latestEvent = await this.db.collection('events')
          .findOne(
            {
              aggregateType: event.aggregateType,
              aggregateId: event.aggregateId
            },
            {
              sort: { aggregateVersion: -1 },
              projection: { aggregateVersion: 1 },
              session
            }
          );

        const expectedVersion = event.expectedVersion;
        const currentVersion = latestEvent?.aggregateVersion || 0;

        if (expectedVersion !== currentVersion) {
          throw new Error(
            `Concurrency conflict: expected version ${expectedVersion}, ` +
            `but current version is ${currentVersion}`
          );
        }

        // 2. Générer sequence globale
        const sequenceDoc = await this.db.collection('counters')
          .findOneAndUpdate(
            { _id: 'event_sequence' },
            { $inc: { value: 1 } },
            {
              returnDocument: 'after',
              session
            }
          );

        const sequence = sequenceDoc.value;

        // 3. Créer event complet
        persistedEvent = {
          eventId: this.generateEventId(),

          eventType: event.eventType,
          eventVersion: event.eventVersion || '1.0',

          aggregateType: event.aggregateType,
          aggregateId: event.aggregateId,
          aggregateVersion: currentVersion + 1,

          data: event.data,

          metadata: {
            ...event.metadata,
            timestamp: new Date()
          },

          timestamp: new Date(),
          sequence,

          isSnapshot: false,
          processed: false,
          archived: false
        };

        // 4. Valider event schema
        await this.validateEvent(persistedEvent);

        // 5. Persister event
        await this.db.collection('events').insertOne(
          persistedEvent,
          { session }
        );

        // 6. Mettre à jour projection si synchrone
        if (event.projection) {
          await event.projection.apply(persistedEvent, session);
        }

      }, {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority', j: true },
        readPreference: 'primary'
      });

      return persistedEvent;

    } finally {
      await session.endSession();
    }
  }

  async appendEvents(events) {
    // Batch append pour performance
    const session = this.db.client.startSession();

    try {
      const persistedEvents = [];

      await session.withTransaction(async () => {
        // Vérifier tous même aggregate
        const aggregateType = events[0].aggregateType;
        const aggregateId = events[0].aggregateId;

        if (!events.every(e =>
          e.aggregateType === aggregateType &&
          e.aggregateId === aggregateId
        )) {
          throw new Error('All events must be for same aggregate');
        }

        // Vérifier version
        const latestEvent = await this.db.collection('events')
          .findOne(
            { aggregateType, aggregateId },
            {
              sort: { aggregateVersion: -1 },
              projection: { aggregateVersion: 1 },
              session
            }
          );

        let currentVersion = latestEvent?.aggregateVersion || 0;
        const expectedVersion = events[0].expectedVersion;

        if (expectedVersion !== currentVersion) {
          throw new Error('Concurrency conflict');
        }

        // Générer sequences
        const count = events.length;
        const sequenceDoc = await this.db.collection('counters')
          .findOneAndUpdate(
            { _id: 'event_sequence' },
            { $inc: { value: count } },
            { returnDocument: 'after', session }
          );

        const startSequence = sequenceDoc.value - count + 1;

        // Préparer events
        for (let i = 0; i < events.length; i++) {
          const event = events[i];

          const persistedEvent = {
            eventId: this.generateEventId(),
            eventType: event.eventType,
            eventVersion: event.eventVersion || '1.0',

            aggregateType,
            aggregateId,
            aggregateVersion: currentVersion + i + 1,

            data: event.data,
            metadata: {
              ...event.metadata,
              timestamp: new Date()
            },

            timestamp: new Date(),
            sequence: startSequence + i,

            isSnapshot: false,
            processed: false,
            archived: false
          };

          persistedEvents.push(persistedEvent);
        }

        // Bulk insert
        await this.db.collection('events').insertMany(
          persistedEvents,
          {
            ordered: true,  // Important: préserver ordre
            session
          }
        );

      }, {
        writeConcern: { w: 'majority', j: true }
      });

      return persistedEvents;

    } finally {
      await session.endSession();
    }
  }

  async validateEvent(event) {
    // Récupérer schema
    const schema = await this.db.collection('event_schemas')
      .findOne({
        eventType: event.eventType,
        version: event.eventVersion
      });

    if (!schema) {
      // Si pas de schema, accepter (opt-in validation)
      return;
    }

    // Valider avec JSON Schema
    const Ajv = require('ajv');
    const ajv = new Ajv();

    const validate = ajv.compile(schema.schema);
    const valid = validate(event.data);

    if (!valid) {
      throw new Error(
        `Event validation failed: ${ajv.errorsText(validate.errors)}`
      );
    }
  }

  generateEventId() {
    return `evt_${Date.now()}_${Math.random().toString(36).substr(2, 15)}`;
  }
}
```

### 2. Load Aggregate depuis Events

```javascript
class AggregateRepository {
  constructor(eventStore, snapshotStore) {
    this.eventStore = eventStore;
    this.snapshotStore = snapshotStore;
  }

  async load(aggregateType, aggregateId) {
    // 1. Essayer charger depuis snapshot
    const snapshot = await this.snapshotStore.getLatest(
      aggregateType,
      aggregateId
    );

    let aggregate;
    let fromVersion;

    if (snapshot) {
      // Commencer depuis snapshot
      aggregate = this.createAggregate(aggregateType, snapshot.state);
      fromVersion = snapshot.version;

      console.log(`Loaded snapshot at version ${fromVersion}`);
    } else {
      // Créer nouveau aggregate vide
      aggregate = this.createAggregate(aggregateType);
      fromVersion = 0;
    }

    // 2. Charger events depuis snapshot/début
    const events = await this.eventStore.getEvents(
      aggregateType,
      aggregateId,
      fromVersion
    );

    console.log(`Replaying ${events.length} events`);

    // 3. Appliquer events
    for (const event of events) {
      aggregate.applyEvent(event);
    }

    return aggregate;
  }

  async save(aggregate, expectedVersion) {
    // Récupérer uncommitted events
    const events = aggregate.getUncommittedEvents();

    if (events.length === 0) {
      return;  // Rien à sauvegarder
    }

    // Préparer events pour persistence
    const eventsToAppend = events.map(event => ({
      eventType: event.eventType,
      eventVersion: event.eventVersion || '1.0',

      aggregateType: aggregate.aggregateType,
      aggregateId: aggregate.aggregateId,
      expectedVersion,

      data: event.data,
      metadata: event.metadata
    }));

    // Append au event store
    const persistedEvents = await this.eventStore.appendEvents(
      eventsToAppend
    );

    // Clear uncommitted events
    aggregate.markEventsAsCommitted();

    // Vérifier si snapshot nécessaire
    const currentVersion = expectedVersion + events.length;

    if (this.shouldSnapshot(currentVersion)) {
      await this.snapshotStore.createSnapshot(
        aggregate.aggregateType,
        aggregate.aggregateId,
        currentVersion,
        aggregate.getState()
      );
    }

    return persistedEvents;
  }

  shouldSnapshot(version) {
    // Créer snapshot tous les 50 events
    return version % 50 === 0;
  }

  createAggregate(aggregateType, initialState = null) {
    // Factory pattern
    switch (aggregateType) {
      case 'account':
        return new AccountAggregate(initialState);
      case 'order':
        return new OrderAggregate(initialState);
      default:
        throw new Error(`Unknown aggregate type: ${aggregateType}`);
    }
  }
}

// Event Store queries
class EventStoreQueries {
  constructor(db) {
    this.db = db;
  }

  async getEvents(aggregateType, aggregateId, fromVersion = 0) {
    const events = await this.db.collection('events')
      .find({
        aggregateType,
        aggregateId,
        aggregateVersion: { $gt: fromVersion }
      })
      .sort({ aggregateVersion: 1 })
      .toArray();

    return events;
  }

  async getEventsByType(eventType, from, to) {
    const query = { eventType };

    if (from || to) {
      query.timestamp = {};
      if (from) query.timestamp.$gte = from;
      if (to) query.timestamp.$lte = to;
    }

    const events = await this.db.collection('events')
      .find(query)
      .sort({ timestamp: 1 })
      .toArray();

    return events;
  }

  async getEventsByCorrelation(correlationId) {
    const events = await this.db.collection('events')
      .find({ 'metadata.correlationId': correlationId })
      .sort({ timestamp: 1 })
      .toArray();

    return events;
  }

  async getEventStream(fromSequence = 0, limit = 1000) {
    // Pour projections qui rattrapent
    const events = await this.db.collection('events')
      .find({
        sequence: { $gt: fromSequence }
      })
      .sort({ sequence: 1 })
      .limit(limit)
      .toArray();

    return events;
  }
}
```

### 3. Aggregate Base Class

```javascript
class Aggregate {
  constructor(aggregateType, aggregateId, initialState = null) {
    this.aggregateType = aggregateType;
    this.aggregateId = aggregateId;
    this.version = 0;
    this.uncommittedEvents = [];

    if (initialState) {
      this.loadFromState(initialState);
    } else {
      this.initialize();
    }
  }

  initialize() {
    // À surcharger par classes dérivées
  }

  loadFromState(state) {
    // Restaurer depuis snapshot
    Object.assign(this, state);
  }

  applyEvent(event) {
    // Dispatcher vers handler spécifique
    const handler = this.getEventHandler(event.eventType);

    if (!handler) {
      console.warn(`No handler for event type: ${event.eventType}`);
      return;
    }

    handler.call(this, event.data, event.metadata);

    this.version = event.aggregateVersion;
  }

  raiseEvent(eventType, data, metadata = {}) {
    // Créer event
    const event = {
      eventType,
      eventVersion: '1.0',
      data,
      metadata: {
        ...metadata,
        timestamp: new Date()
      }
    };

    // Appliquer immédiatement (optimistic)
    this.applyEvent({
      ...event,
      aggregateVersion: this.version + 1
    });

    // Ajouter aux uncommitted
    this.uncommittedEvents.push(event);
  }

  getEventHandler(eventType) {
    // Convention: onEventName
    const handlerName = 'on' + eventType
      .split('.')
      .map(part => part.charAt(0).toUpperCase() + part.slice(1))
      .join('');

    return this[handlerName];
  }

  getUncommittedEvents() {
    return this.uncommittedEvents;
  }

  markEventsAsCommitted() {
    this.uncommittedEvents = [];
  }

  getState() {
    // Retourner état pour snapshot
    const { uncommittedEvents, ...state } = this;
    return state;
  }
}

// Exemple: Account Aggregate
class AccountAggregate extends Aggregate {
  initialize() {
    this.balance = NumberDecimal("0.00");
    this.currency = "USD";
    this.status = "pending";
    this.owner = null;
    this.createdAt = null;
  }

  // Commands (business logic)

  create(ownerId, ownerName, currency) {
    // Validation
    if (this.createdAt) {
      throw new Error('Account already created');
    }

    // Raise event
    this.raiseEvent('account.created', {
      ownerId,
      ownerName,
      currency
    });
  }

  credit(amount, transactionId, reason) {
    // Validation
    if (this.status !== 'active') {
      throw new Error('Account is not active');
    }

    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }

    // Raise event
    this.raiseEvent('account.credited', {
      amount,
      transactionId,
      reason,
      balanceAfter: this.balance + amount
    });
  }

  debit(amount, transactionId, reason) {
    // Validation
    if (this.status !== 'active') {
      throw new Error('Account is not active');
    }

    if (amount <= 0) {
      throw new Error('Amount must be positive');
    }

    if (this.balance < amount) {
      throw new Error('Insufficient funds');
    }

    // Raise event
    this.raiseEvent('account.debited', {
      amount,
      transactionId,
      reason,
      balanceAfter: this.balance - amount
    });
  }

  activate() {
    if (this.status === 'active') {
      throw new Error('Account already active');
    }

    this.raiseEvent('account.activated', {});
  }

  freeze(reason) {
    if (this.status !== 'active') {
      throw new Error('Can only freeze active account');
    }

    this.raiseEvent('account.frozen', { reason });
  }

  // Event handlers (state changes)

  onAccountCreated(data, metadata) {
    this.owner = {
      userId: data.ownerId,
      name: data.ownerName
    };
    this.currency = data.currency;
    this.status = 'pending';
    this.createdAt = metadata.timestamp;
  }

  onAccountCredited(data, metadata) {
    this.balance = NumberDecimal(
      (parseFloat(this.balance) + parseFloat(data.amount)).toString()
    );
    this.lastModified = metadata.timestamp;
  }

  onAccountDebited(data, metadata) {
    this.balance = NumberDecimal(
      (parseFloat(this.balance) - parseFloat(data.amount)).toString()
    );
    this.lastModified = metadata.timestamp;
  }

  onAccountActivated(data, metadata) {
    this.status = 'active';
    this.activatedAt = metadata.timestamp;
  }

  onAccountFrozen(data, metadata) {
    this.status = 'frozen';
    this.frozenAt = metadata.timestamp;
    this.frozenReason = data.reason;
  }
}
```

## Projection Handlers

### 1. Projection Service avec Change Streams

```javascript
class ProjectionService {
  constructor(db, projectionHandlers) {
    this.db = db;
    this.projectionHandlers = projectionHandlers;
  }

  async start() {
    // Watch change stream sur events
    const changeStream = this.db.collection('events').watch([
      {
        $match: {
          operationType: 'insert'
        }
      }
    ]);

    changeStream.on('change', async (change) => {
      const event = change.fullDocument;

      try {
        await this.handleEvent(event);
      } catch (error) {
        console.error('Failed to project event:', error);
        // Dead letter queue
        await this.sendToDeadLetter(event, error);
      }
    });

    console.log('Projection service started');
  }

  async handleEvent(event) {
    // Dispatcher vers handlers appropriés
    for (const handler of this.projectionHandlers) {
      if (handler.canHandle(event)) {
        await handler.handle(event);
      }
    }

    // Marquer event comme processed
    await this.db.collection('events').updateOne(
      { _id: event._id },
      { $set: { processed: true } }
    );
  }

  async replay(projectionName, fromVersion = 0) {
    console.log(`Replaying projection: ${projectionName} from version ${fromVersion}`);

    const handler = this.projectionHandlers.find(
      h => h.name === projectionName
    );

    if (!handler) {
      throw new Error(`Projection not found: ${projectionName}`);
    }

    // Réinitialiser projection
    await handler.reset();

    // Streamer tous les events
    const cursor = this.db.collection('events')
      .find({
        sequence: { $gte: fromVersion }
      })
      .sort({ sequence: 1 });

    let count = 0;

    await cursor.forEach(async (event) => {
      if (handler.canHandle(event)) {
        await handler.handle(event);
        count++;

        if (count % 1000 === 0) {
          console.log(`Replayed ${count} events`);
        }
      }
    });

    console.log(`Replay completed: ${count} events`);
  }
}

// Account Read Model Projection
class AccountProjectionHandler {
  constructor(db) {
    this.db = db;
    this.name = 'accounts_read';
  }

  canHandle(event) {
    return event.eventType.startsWith('account.');
  }

  async handle(event) {
    switch (event.eventType) {
      case 'account.created':
        await this.onAccountCreated(event);
        break;

      case 'account.credited':
        await this.onAccountCredited(event);
        break;

      case 'account.debited':
        await this.onAccountDebited(event);
        break;

      case 'account.activated':
        await this.onAccountActivated(event);
        break;

      case 'account.frozen':
        await this.onAccountFrozen(event);
        break;
    }
  }

  async onAccountCreated(event) {
    const account = {
      accountId: event.aggregateId,

      balance: NumberDecimal("0.00"),
      currency: event.data.currency,
      status: 'pending',

      owner: {
        userId: event.data.ownerId,
        name: event.data.ownerName
      },

      createdAt: event.timestamp,
      updatedAt: event.timestamp,

      lastEventVersion: event.aggregateVersion,
      lastEventId: event.eventId,

      stats: {
        totalCredits: NumberDecimal("0.00"),
        totalDebits: NumberDecimal("0.00"),
        transactionCount: 0,
        lastTransactionAt: null
      }
    };

    await this.db.collection('accounts_read').insertOne(account);
  }

  async onAccountCredited(event) {
    await this.db.collection('accounts_read').updateOne(
      { accountId: event.aggregateId },
      {
        $set: {
          balance: event.data.balanceAfter,
          updatedAt: event.timestamp,
          lastEventVersion: event.aggregateVersion,
          lastEventId: event.eventId
        },
        $inc: {
          'stats.totalCredits': parseFloat(event.data.amount),
          'stats.transactionCount': 1
        },
        $max: {
          'stats.lastTransactionAt': event.timestamp
        }
      }
    );
  }

  async onAccountDebited(event) {
    await this.db.collection('accounts_read').updateOne(
      { accountId: event.aggregateId },
      {
        $set: {
          balance: event.data.balanceAfter,
          updatedAt: event.timestamp,
          lastEventVersion: event.aggregateVersion,
          lastEventId: event.eventId
        },
        $inc: {
          'stats.totalDebits': parseFloat(event.data.amount),
          'stats.transactionCount': 1
        },
        $max: {
          'stats.lastTransactionAt': event.timestamp
        }
      }
    );
  }

  async onAccountActivated(event) {
    await this.db.collection('accounts_read').updateOne(
      { accountId: event.aggregateId },
      {
        $set: {
          status: 'active',
          activatedAt: event.timestamp,
          updatedAt: event.timestamp,
          lastEventVersion: event.aggregateVersion,
          lastEventId: event.eventId
        }
      }
    );
  }

  async onAccountFrozen(event) {
    await this.db.collection('accounts_read').updateOne(
      { accountId: event.aggregateId },
      {
        $set: {
          status: 'frozen',
          frozenAt: event.timestamp,
          frozenReason: event.data.reason,
          updatedAt: event.timestamp,
          lastEventVersion: event.aggregateVersion,
          lastEventId: event.eventId
        }
      }
    );
  }

  async reset() {
    // Supprimer toutes les projections
    await this.db.collection('accounts_read').deleteMany({});
    console.log('Account projection reset');
  }
}
```

### 2. Analytics Projection

```javascript
class AnalyticsProjectionHandler {
  constructor(db) {
    this.db = db;
    this.name = 'analytics';
  }

  canHandle(event) {
    // Tous les événements financiers
    return event.eventType.includes('credited') ||
           event.eventType.includes('debited') ||
           event.eventType.includes('payment');
  }

  async handle(event) {
    // Agréger par heure
    const hour = new Date(event.timestamp);
    hour.setMinutes(0, 0, 0);

    const updates = {};

    if (event.eventType === 'account.credited') {
      updates.$inc = {
        'metrics.creditCount': 1,
        'metrics.creditTotal': parseFloat(event.data.amount)
      };
    }

    if (event.eventType === 'account.debited') {
      updates.$inc = {
        'metrics.debitCount': 1,
        'metrics.debitTotal': parseFloat(event.data.amount)
      };
    }

    await this.db.collection('analytics_hourly').updateOne(
      {
        hour,
        currency: event.data.currency || 'USD'
      },
      {
        ...updates,
        $set: {
          updatedAt: new Date()
        }
      },
      { upsert: true }
    );
  }

  async reset() {
    await this.db.collection('analytics_hourly').deleteMany({});
  }
}
```

## Snapshot Service

```javascript
class SnapshotService {
  constructor(db) {
    this.db = db;
  }

  async createSnapshot(aggregateType, aggregateId, version, state) {
    const snapshot = {
      aggregateType,
      aggregateId,
      version,
      state,

      createdAt: new Date(),
      createdBy: 'snapshot_service',

      eventCount: version,

      // Expire après 30 jours
      expiresAt: new Date(Date.now() + 30 * 24 * 3600000)
    };

    // Upsert (remplacer ancien snapshot)
    await this.db.collection('snapshots').replaceOne(
      {
        aggregateType,
        aggregateId
      },
      snapshot,
      { upsert: true }
    );

    console.log(
      `Snapshot created for ${aggregateType}/${aggregateId} at version ${version}`
    );
  }

  async getLatest(aggregateType, aggregateId) {
    const snapshot = await this.db.collection('snapshots')
      .findOne({
        aggregateType,
        aggregateId
      });

    return snapshot;
  }

  async createSnapshotsForAll(aggregateType, minVersion = 50) {
    // Créer snapshots pour tous aggregates avec beaucoup d'events
    const pipeline = [
      {
        $match: {
          aggregateType
        }
      },
      {
        $group: {
          _id: '$aggregateId',
          maxVersion: { $max: '$aggregateVersion' },
          eventCount: { $sum: 1 }
        }
      },
      {
        $match: {
          eventCount: { $gte: minVersion }
        }
      }
    ];

    const aggregates = await this.db.collection('events')
      .aggregate(pipeline)
      .toArray();

    console.log(
      `Creating snapshots for ${aggregates.length} aggregates`
    );

    for (const agg of aggregates) {
      try {
        // Load aggregate
        const repository = new AggregateRepository(
          new EventStore(this.db),
          this
        );

        const aggregate = await repository.load(
          aggregateType,
          agg._id
        );

        // Create snapshot
        await this.createSnapshot(
          aggregateType,
          agg._id,
          agg.maxVersion,
          aggregate.getState()
        );

      } catch (error) {
        console.error(
          `Failed to create snapshot for ${agg._id}:`,
          error
        );
      }
    }

    console.log('Snapshot creation completed');
  }
}
```

## Temporal Queries (Time Travel)

```javascript
class TemporalQueryService {
  constructor(db) {
    this.db = db;
  }

  async getStateAt(aggregateType, aggregateId, timestamp) {
    // Récupérer tous les events jusqu'à timestamp
    const events = await this.db.collection('events')
      .find({
        aggregateType,
        aggregateId,
        timestamp: { $lte: timestamp }
      })
      .sort({ aggregateVersion: 1 })
      .toArray();

    if (events.length === 0) {
      return null;  // Aggregate n'existait pas encore
    }

    // Reconstruire état
    const repository = new AggregateRepository(
      new EventStore(this.db),
      new SnapshotService(this.db)
    );

    const AggregateClass = this.getAggregateClass(aggregateType);
    const aggregate = new AggregateClass();

    for (const event of events) {
      aggregate.applyEvent(event);
    }

    return aggregate.getState();
  }

  async getAggregatesAt(aggregateType, timestamp, filter = {}) {
    // Récupérer tous aggregates à une date donnée
    // Utile pour reporting historique

    const aggregateIds = await this.db.collection('events')
      .distinct('aggregateId', {
        aggregateType,
        timestamp: { $lte: timestamp }
      });

    const states = [];

    for (const aggregateId of aggregateIds) {
      const state = await this.getStateAt(
        aggregateType,
        aggregateId,
        timestamp
      );

      // Appliquer filtre
      if (this.matchesFilter(state, filter)) {
        states.push(state);
      }
    }

    return states;
  }

  async getBalanceHistory(accountId, fromDate, toDate) {
    // Historique de balance jour par jour

    const events = await this.db.collection('events')
      .find({
        aggregateType: 'account',
        aggregateId: accountId,
        timestamp: {
          $gte: fromDate,
          $lte: toDate
        },
        eventType: { $in: ['account.credited', 'account.debited'] }
      })
      .sort({ timestamp: 1 })
      .toArray();

    const history = [];
    let currentBalance = NumberDecimal("0.00");

    // Récupérer balance initiale
    const initialEvents = await this.db.collection('events')
      .find({
        aggregateType: 'account',
        aggregateId: accountId,
        timestamp: { $lt: fromDate }
      })
      .sort({ aggregateVersion: 1 })
      .toArray();

    for (const event of initialEvents) {
      if (event.eventType === 'account.credited') {
        currentBalance += parseFloat(event.data.amount);
      } else if (event.eventType === 'account.debited') {
        currentBalance -= parseFloat(event.data.amount);
      }
    }

    // Construire historique
    for (const event of events) {
      if (event.eventType === 'account.credited') {
        currentBalance += parseFloat(event.data.amount);
      } else if (event.eventType === 'account.debited') {
        currentBalance -= parseFloat(event.data.amount);
      }

      history.push({
        timestamp: event.timestamp,
        balance: NumberDecimal(currentBalance.toString()),
        eventType: event.eventType,
        amount: event.data.amount,
        transactionId: event.data.transactionId
      });
    }

    return history;
  }

  matchesFilter(state, filter) {
    for (const [key, value] of Object.entries(filter)) {
      if (state[key] !== value) {
        return false;
      }
    }
    return true;
  }

  getAggregateClass(aggregateType) {
    // Factory
    switch (aggregateType) {
      case 'account':
        return AccountAggregate;
      default:
        throw new Error(`Unknown aggregate: ${aggregateType}`);
    }
  }
}
```

## Event Migration et Versioning

```javascript
class EventMigrationService {
  constructor(db) {
    this.db = db;
    this.migrations = [];
  }

  registerMigration(migration) {
    this.migrations.push(migration);
  }

  async migrate(fromVersion, toVersion) {
    console.log(
      `Migrating events from version ${fromVersion} to ${toVersion}`
    );

    const migration = this.migrations.find(
      m => m.fromVersion === fromVersion && m.toVersion === toVersion
    );

    if (!migration) {
      throw new Error(
        `No migration found from ${fromVersion} to ${toVersion}`
      );
    }

    // Trouver events à migrer
    const events = await this.db.collection('events')
      .find({
        eventType: migration.eventType,
        eventVersion: fromVersion
      })
      .toArray();

    console.log(`Found ${events.length} events to migrate`);

    let migrated = 0;

    for (const event of events) {
      try {
        // Appliquer transformation
        const transformedData = migration.transform(event.data);

        // Update event
        await this.db.collection('events').updateOne(
          { _id: event._id },
          {
            $set: {
              eventVersion: toVersion,
              data: transformedData,
              'metadata.migrated': true,
              'metadata.migratedAt': new Date(),
              'metadata.migratedFrom': fromVersion
            }
          }
        );

        migrated++;

        if (migrated % 100 === 0) {
          console.log(`Migrated ${migrated} events`);
        }

      } catch (error) {
        console.error(`Failed to migrate event ${event._id}:`, error);
      }
    }

    console.log(`Migration completed: ${migrated} events`);
  }
}

// Exemple de migration
const accountCreditedMigration = {
  eventType: 'account.credited',
  fromVersion: '1.0',
  toVersion: '2.0',

  transform: (oldData) => {
    // V1: { amount, transactionId }
    // V2: { amount, currency, transactionId, metadata }

    return {
      amount: oldData.amount,
      currency: oldData.currency || 'USD',  // Default
      transactionId: oldData.transactionId,
      metadata: {
        migrated: true,
        originalVersion: '1.0'
      }
    };
  }
};

// Event Upcaster pour lecture
class EventUpcaster {
  constructor() {
    this.upcasters = new Map();
  }

  registerUpcaster(eventType, fromVersion, toVersion, transformer) {
    const key = `${eventType}:${fromVersion}`;
    this.upcasters.set(key, { toVersion, transformer });
  }

  upcast(event) {
    const key = `${event.eventType}:${event.eventVersion}`;
    const upcaster = this.upcasters.get(key);

    if (!upcaster) {
      return event;  // Déjà à jour
    }

    // Transformer
    const upcastedData = upcaster.transformer(event.data);

    return {
      ...event,
      eventVersion: upcaster.toVersion,
      data: upcastedData,
      _upcast: true
    };
  }

  upcastStream(events) {
    return events.map(event => this.upcast(event));
  }
}
```

## Command Handler Integration

```javascript
class AccountCommandHandler {
  constructor(repository, eventBus) {
    this.repository = repository;
    this.eventBus = eventBus;
  }

  async handle(command) {
    const { type, aggregateId, data, metadata } = command;

    try {
      // Load aggregate
      const account = await this.repository.load('account', aggregateId);

      // Execute command
      switch (type) {
        case 'CreateAccount':
          account.create(
            data.ownerId,
            data.ownerName,
            data.currency
          );
          break;

        case 'CreditAccount':
          account.credit(
            data.amount,
            data.transactionId,
            data.reason
          );
          break;

        case 'DebitAccount':
          account.debit(
            data.amount,
            data.transactionId,
            data.reason
          );
          break;

        case 'ActivateAccount':
          account.activate();
          break;

        case 'FreezeAccount':
          account.freeze(data.reason);
          break;

        default:
          throw new Error(`Unknown command: ${type}`);
      }

      // Save (persist events)
      const events = await this.repository.save(
        account,
        account.version - account.getUncommittedEvents().length
      );

      // Publish to event bus (optionnel, Change Streams peut suffire)
      for (const event of events) {
        await this.eventBus.publish(event.eventType, event);
      }

      return {
        success: true,
        aggregateId,
        version: account.version,
        events: events.map(e => e.eventId)
      };

    } catch (error) {
      console.error('Command failed:', error);

      return {
        success: false,
        error: error.message
      };
    }
  }
}

// API Controller
class AccountController {
  constructor(commandHandler, queryService) {
    this.commandHandler = commandHandler;
    this.queryService = queryService;
  }

  async createAccount(req, res) {
    const command = {
      type: 'CreateAccount',
      aggregateId: this.generateAccountId(),
      data: {
        ownerId: req.user.userId,
        ownerName: req.user.name,
        currency: req.body.currency || 'USD'
      },
      metadata: {
        userId: req.user.userId,
        ipAddress: req.ip,
        correlationId: req.headers['x-correlation-id']
      }
    };

    const result = await this.commandHandler.handle(command);

    if (result.success) {
      res.status(201).json({
        accountId: command.aggregateId,
        version: result.version
      });
    } else {
      res.status(400).json({ error: result.error });
    }
  }

  async creditAccount(req, res) {
    const command = {
      type: 'CreditAccount',
      aggregateId: req.params.accountId,
      data: {
        amount: req.body.amount,
        transactionId: this.generateTransactionId(),
        reason: req.body.reason
      },
      metadata: {
        userId: req.user.userId,
        ipAddress: req.ip
      }
    };

    const result = await this.commandHandler.handle(command);

    if (result.success) {
      res.json({ success: true, version: result.version });
    } else {
      res.status(400).json({ error: result.error });
    }
  }

  async getAccount(req, res) {
    // Query depuis read model
    const account = await this.queryService.getAccount(
      req.params.accountId
    );

    if (account) {
      res.json(account);
    } else {
      res.status(404).json({ error: 'Account not found' });
    }
  }

  async getAccountHistory(req, res) {
    // Query depuis event store
    const events = await this.queryService.getAccountEvents(
      req.params.accountId
    );

    res.json({
      accountId: req.params.accountId,
      eventCount: events.length,
      events: events.map(e => ({
        eventId: e.eventId,
        eventType: e.eventType,
        timestamp: e.timestamp,
        data: e.data
      }))
    });
  }

  generateAccountId() {
    return `acc_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  generateTransactionId() {
    return `txn_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}
```

## Performance et optimisation

### 1. Configuration MongoDB optimale

```javascript
const eventSourcingConfig = {
  // Event store collection optimisations
  eventStoreOptions: {
    // Write concern strict pour events
    writeConcern: { w: 'majority', j: true },

    // No reads on event store typically
    readConcern: { level: 'majority' },

    // Storage engine optimisé append-only
    storageEngine: {
      wiredTiger: {
        configString: 'log=(enabled=true,file_max=10MB)'
      }
    }
  },

  // Projections peuvent avoir write concern relaxé
  projectionOptions: {
    writeConcern: { w: 1, j: false },
    readConcern: { level: 'local' }
  },

  // Change Streams configuration
  changeStreamOptions: {
    fullDocument: 'updateLookup',
    maxAwaitTimeMS: 1000
  }
};

// Sharding pour très gros volume
sh.enableSharding('event_store');

// Shard key: aggregateType + aggregateId
// Permet isolation par aggregate
sh.shardCollection(
  'event_store.events',
  { aggregateType: 1, aggregateId: 'hashed' }
);

// Zone sharding pour archivage
sh.addShardToZone('shard01', 'recent');
sh.addShardToZone('shard02', 'archived');

const cutoff = new Date(Date.now() - 90 * 24 * 3600000);  // 90 jours

sh.updateZoneKeyRange(
  'event_store.events',
  { timestamp: new Date(0) },
  { timestamp: cutoff },
  'archived'
);

sh.updateZoneKeyRange(
  'event_store.events',
  { timestamp: cutoff },
  { timestamp: new Date(2099, 12, 31) },
  'recent'
);
```

### 2. Métriques de performance

```javascript
const eventSourcingMetrics = {
  // Event store
  'events.append.latency.p99': {
    description: 'Event append latency p99',
    target: 50,  // ms
    alert_threshold: 200
  },

  'events.replay.duration': {
    description: 'Full aggregate replay duration',
    target: 100,  // ms
    alert_threshold: 1000
  },

  // Projections
  'projections.lag': {
    description: 'Projection lag behind event store',
    target: 1000,  // ms
    alert_threshold: 10000
  },

  'projections.throughput': {
    description: 'Events processed per second',
    target: 1000,
    alert_threshold: 100
  },

  // Snapshots
  'snapshots.creation.duration': {
    description: 'Snapshot creation duration',
    target: 500,  // ms
    alert_threshold: 5000
  },

  'snapshots.hit_rate': {
    description: 'Snapshot hit rate',
    target: 0.90,
    alert_threshold: 0.50
  }
};
```

## Checklist de déploiement

### ✅ Event Store

- [ ] Events collection avec index appropriés
- [ ] Sequence counter initialisé
- [ ] Write concern strict (w:majority, j:true)
- [ ] Immutabilité garantie (pas de updates/deletes)
- [ ] Change Streams configuré
- [ ] Sharding si volume important

### ✅ Aggregates

- [ ] Base classes Aggregate implémentées
- [ ] Event handlers pour tous event types
- [ ] Command validation
- [ ] Optimistic concurrency control
- [ ] Business logic isolée

### ✅ Projections

- [ ] Read models définis
- [ ] Projection handlers implémentés
- [ ] Change Streams consumers
- [ ] Idempotence garantie
- [ ] Replay capability
- [ ] Dead letter queue

### ✅ Snapshots

- [ ] Snapshot service implémenté
- [ ] Stratégie snapshot (tous les N events)
- [ ] TTL pour cleanup
- [ ] Performance monitoring

### ✅ Event Schema

- [ ] Event type registry
- [ ] JSON Schema validation
- [ ] Versioning strategy
- [ ] Migration paths définis
- [ ] Upcasters pour backward compatibility

### ✅ Commands

- [ ] Command handlers implémentés
- [ ] Validation complète
- [ ] Authorization checks
- [ ] Audit logging
- [ ] Idempotency keys

### ✅ Queries

- [ ] Query service depuis read models
- [ ] Temporal queries si nécessaire
- [ ] Event history endpoints
- [ ] Performance optimisée

### ✅ Operations

- [ ] Monitoring métriques
- [ ] Alerting sur lag projections
- [ ] Replay procedures
- [ ] Event migration tools
- [ ] Backup strategy
- [ ] Disaster recovery plan

## Conclusion

Event Sourcing avec MongoDB offre :

**✅ Forces démontrées :**
- Audit trail complet et immuable
- Time travel et replay capabilities
- Multiple projections depuis mêmes events
- Change Streams pour propagation temps réel
- Flexible schema pour event evolution
- Excellent pour event-driven architectures
- Performance append-only optimisée
- ACID transactions pour consistency

**⚠️ Considérations importantes :**
- Complexité accrue vs CRUD traditionnel
- Eventual consistency pour projections
- Volume de données important
- Event schema evolution challenges
- Courbe d'apprentissage
- Testing plus complexe

**🎯 Patterns essentiels Event Sourcing :**
1. **Immutable events** comme source de vérité
2. **Snapshots** pour performance
3. **CQRS** pour séparation read/write
4. **Projections** pour read models optimisés
5. **Event versioning** pour évolution
6. **Replay** pour reconstruction et debugging
7. **Change Streams** pour propagation temps réel

**Quand utiliser Event Sourcing :**
- ✅ Audit trail critique (finance, santé, légal)
- ✅ Temporal queries nécessaires
- ✅ Event-driven architecture
- ✅ Complex domain logic
- ✅ Multiple read models requis
- ❌ Simple CRUD applications
- ❌ Équipe inexperimentée
- ❌ Queries complexes sur état actuel

Event Sourcing avec MongoDB est particulièrement puissant pour les domaines métier complexes nécessitant traçabilité complète, capacités d'audit avancées, et architectures event-driven.

---

**Références :**
- "Domain-Driven Design" - Eric Evans
- "Implementing Domain-Driven Design" - Vaughn Vernon
- "Versioning in an Event Sourced System" - Greg Young
- MongoDB Change Streams Documentation
- Event Sourcing Patterns - Martin Fowler
- CQRS Journey - Microsoft patterns & practices

⏭️ [CQRS et MongoDB](/20-cas-usage-architectures/12-cqrs-mongodb.md)
