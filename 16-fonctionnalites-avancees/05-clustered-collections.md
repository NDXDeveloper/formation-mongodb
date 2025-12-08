🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.5 Clustered Collections

## Introduction

Les **Clustered Collections** sont une fonctionnalité avancée de MongoDB (introduite dans la version 5.3) qui organise physiquement les documents sur le disque selon une clé de clustering. Contrairement aux collections standard où les documents sont stockés dans un ordre arbitraire, les clustered collections maintiennent un ordre physique basé sur la clé de clustering (généralement `_id`).

Cette organisation améliore significativement les performances pour les requêtes par plages (range queries), les scans séquentiels, et réduit la fragmentation des données, particulièrement pour les workloads avec des accès temporels ou séquentiels fréquents.

---

## Architecture et concepts fondamentaux

### Organisation physique traditionnelle vs Clustered

```
┌────────────────────────────────────────────────────────────┐
│           Collection Standard (Non-Clustered)              │
│                                                            │
│  Documents stockés dans l'ordre d'insertion (WiredTiger):  │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Page 1: [Doc A] [Doc F] [Doc C] [Doc Z]            │    │
│  │ Page 2: [Doc B] [Doc M] [Doc D] [Doc K]            │    │
│  │ Page 3: [Doc E] [Doc L] [Doc G] [Doc H]            │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  Ordre physique ≠ Ordre logique (_id)                      │
│  → Range scan nécessite sauts aléatoires entre pages       │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│             Clustered Collection                           │
│                                                            │
│  Documents stockés ordonnés par clé de clustering (_id):   │
│  ┌────────────────────────────────────────────────────┐    │
│  │ Page 1: [Doc A] [Doc B] [Doc C] [Doc D]            │    │
│  │ Page 2: [Doc E] [Doc F] [Doc G] [Doc H]            │    │
│  │ Page 3: [Doc K] [Doc L] [Doc M] [Doc Z]            │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  Ordre physique = Ordre logique (_id)                      │
│  → Range scan = lecture séquentielle optimale              │
└────────────────────────────────────────────────────────────┘
```

### Caractéristiques clés

| Propriété | Clustered | Standard |
|-----------|-----------|----------|
| **Ordre physique** | Maintenu selon clé | Ordre d'insertion |
| **Index _id** | Intégré (pas séparé) | Séparé (overhead) |
| **Range queries** | Lecture séquentielle | Sauts aléatoires |
| **Fragmentation** | Réduite | Standard |
| **Insertions ordonnées** | Optimales | Standard |
| **Updates in-place** | Plus efficaces | Standard |
| **Stockage** | ~10-15% économie | Baseline |

### Avantages principaux

- ✅ **Performance range queries** : Jusqu'à 50% plus rapide pour requêtes par plages
- ✅ **Économie stockage** : Pas d'index _id séparé (~10-15% économie)
- ✅ **Réduction fragmentation** : Documents adjacents logiquement le sont physiquement
- ✅ **Scan séquentiel** : Lectures disk optimales (moins de I/O)
- ✅ **TTL efficace** : Suppression de plages temporelles optimisée

---

## Création et configuration

### Syntaxe de création

```javascript
// Création basique avec _id comme clé de clustering
await db.createCollection('events', {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true
  }
});

// Avec TTL pour auto-expiration
await db.createCollection('logs', {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true
  },
  expireAfterSeconds: 86400  // 24 heures
});

// Avec options avancées
await db.createCollection('timeseries_events', {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true,
    name: 'clustered_id_index'
  },
  expireAfterSeconds: 2592000,  // 30 jours
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['_id', 'timestamp', 'type'],
      properties: {
        _id: { bsonType: 'objectId' },
        timestamp: { bsonType: 'date' },
        type: { bsonType: 'string' }
      }
    }
  }
});
```

### Contraintes et limitations

```javascript
// ⚠️ La clé de clustering DOIT être _id
// ❌ INVALIDE
await db.createCollection('invalid', {
  clusteredIndex: {
    key: { timestamp: 1 },  // Doit être _id
    unique: true
  }
});

// ✅ VALIDE - utiliser ObjectId avec embedded timestamp
const customId = new ObjectId(); // Contient timestamp
await db.collection('events').insertOne({
  _id: customId,
  // ...
});

// ⚠️ Impossible de convertir collection existante
// Doit créer nouvelle collection et migrer
```

### Vérification

```javascript
// Vérifier si une collection est clustered
const collInfo = await db.listCollections({
  name: 'events'
}).toArray();

console.log('Is clustered:', collInfo[0].options?.clusteredIndex ? true : false);

// Obtenir les stats
const stats = await db.collection('events').stats();
console.log({
  clustered: stats.indexDetails?.clusteredIndex !== undefined,
  storageSize: stats.storageSize,
  totalIndexSize: stats.totalIndexSize
});
```

---

## Cas d'usage avancés

### Cas 1 : Système de logs avec accès temporel optimisé

```javascript
class ClusteredLogSystem {
  constructor(db) {
    this.db = db;
    this.logs = null;
  }

  async initialize() {
    // Créer collection clustered pour logs
    try {
      await this.db.createCollection('application_logs', {
        clusteredIndex: {
          key: { _id: 1 },
          unique: true
        },
        expireAfterSeconds: 604800  // 7 jours TTL
      });

      console.log('Clustered log collection created');
    } catch (error) {
      if (error.code !== 48) throw error;
    }

    this.logs = this.db.collection('application_logs');

    // Index secondaires pour requêtes fréquentes
    await this.logs.createIndex({ level: 1, _id: 1 });
    await this.logs.createIndex({ service: 1, _id: 1 });
  }

  async log(level, message, metadata = {}) {
    // ObjectId contient timestamp embedded
    // Documents seront physiquement ordonnés chronologiquement
    const logEntry = {
      _id: new ObjectId(),  // Timestamp embedded
      level,
      message,
      service: metadata.service || 'unknown',
      host: metadata.host || require('os').hostname(),
      pid: process.pid,
      metadata,
      // Pas besoin de champ timestamp séparé car _id contient déjà
    };

    await this.logs.insertOne(logEntry);
  }

  async getLogsByTimeRange(startTime, endTime, options = {}) {
    const {
      level = null,
      service = null,
      limit = 1000
    } = options;

    // Convertir timestamps en ObjectId pour range query
    const startId = ObjectId.createFromTime(
      Math.floor(startTime.getTime() / 1000)
    );
    const endId = ObjectId.createFromTime(
      Math.floor(endTime.getTime() / 1000)
    );

    const query = {
      _id: { $gte: startId, $lte: endId }
    };

    if (level) query.level = level;
    if (service) query.service = service;

    // Range query optimale : lecture séquentielle grâce au clustering
    return await this.logs.find(query)
      .sort({ _id: 1 })  // Déjà physiquement ordonné
      .limit(limit)
      .toArray();
  }

  async getTailLogs(count = 100) {
    // Récupérer les N derniers logs
    // Très efficace car physiquement à la fin de la collection
    return await this.logs.find()
      .sort({ _id: -1 })
      .limit(count)
      .toArray();
  }

  async searchLogs(pattern, timeRange = null, options = {}) {
    const query = {
      $or: [
        { message: { $regex: pattern, $options: 'i' } },
        { 'metadata.error': { $regex: pattern, $options: 'i' } }
      ]
    };

    if (timeRange) {
      const { start, end } = timeRange;
      query._id = {
        $gte: ObjectId.createFromTime(Math.floor(start.getTime() / 1000)),
        $lte: ObjectId.createFromTime(Math.floor(end.getTime() / 1000))
      };
    }

    return await this.logs.find(query)
      .sort({ _id: -1 })
      .limit(options.limit || 100)
      .toArray();
  }

  async analyzeLogPatterns(hours = 24) {
    const sinceTime = new Date(Date.now() - hours * 3600 * 1000);
    const sinceId = ObjectId.createFromTime(
      Math.floor(sinceTime.getTime() / 1000)
    );

    // Agrégation sur période temporelle
    // Bénéficie du clustering pour scan séquentiel
    return await this.logs.aggregate([
      {
        $match: { _id: { $gte: sinceId } }
      },
      {
        $group: {
          _id: {
            level: '$level',
            service: '$service'
          },
          count: { $sum: 1 },
          firstSeen: { $min: '$_id' },
          lastSeen: { $max: '$_id' }
        }
      },
      {
        $project: {
          level: '$_id.level',
          service: '$_id.service',
          count: 1,
          firstSeen: 1,
          lastSeen: 1,
          _id: 0
        }
      },
      { $sort: { count: -1 } }
    ]).toArray();
  }

  async deleteOldLogs(olderThan) {
    // Suppression par plage très efficace avec clustered index
    const oldId = ObjectId.createFromTime(
      Math.floor(olderThan.getTime() / 1000)
    );

    const result = await this.logs.deleteMany({
      _id: { $lt: oldId }
    });

    console.log(`Deleted ${result.deletedCount} old logs`);
    return result.deletedCount;
  }

  async getHourlyStats(hours = 24) {
    const sinceTime = new Date(Date.now() - hours * 3600 * 1000);
    const sinceId = ObjectId.createFromTime(
      Math.floor(sinceTime.getTime() / 1000)
    );

    // Statistiques horaires
    return await this.logs.aggregate([
      { $match: { _id: { $gte: sinceId } } },
      {
        $project: {
          hour: {
            $dateToString: {
              format: '%Y-%m-%d %H:00',
              date: { $toDate: '$_id' }
            }
          },
          level: 1
        }
      },
      {
        $group: {
          _id: { hour: '$hour', level: '$level' },
          count: { $sum: 1 }
        }
      },
      {
        $group: {
          _id: '$_id.hour',
          levelCounts: {
            $push: {
              level: '$_id.level',
              count: '$count'
            }
          },
          total: { $sum: '$count' }
        }
      },
      { $sort: { _id: 1 } }
    ]).toArray();
  }

  async detectErrorSpikes(threshold = 100) {
    // Détecter pics d'erreurs dans dernière heure
    const lastHour = new Date(Date.now() - 3600000);
    const hourId = ObjectId.createFromTime(
      Math.floor(lastHour.getTime() / 1000)
    );

    const errorCount = await this.logs.countDocuments({
      _id: { $gte: hourId },
      level: 'error'
    });

    if (errorCount > threshold) {
      // Récupérer les erreurs pour analyse
      const errors = await this.logs.find({
        _id: { $gte: hourId },
        level: 'error'
      })
        .sort({ _id: -1 })
        .limit(50)
        .toArray();

      return {
        spike: true,
        errorCount,
        threshold,
        recentErrors: errors
      };
    }

    return { spike: false, errorCount };
  }

  getTimestampFromId(objectId) {
    // Extraire timestamp de l'ObjectId
    return objectId.getTimestamp();
  }

  async performanceBenchmark() {
    // Benchmark range query performance
    const now = new Date();
    const oneHourAgo = new Date(now - 3600000);

    const start = Date.now();
    const logs = await this.getLogsByTimeRange(oneHourAgo, now);
    const duration = Date.now() - start;

    return {
      documentsScanned: logs.length,
      durationMs: duration,
      docsPerSecond: Math.round(logs.length / (duration / 1000))
    };
  }
}

// Utilisation
const logSystem = new ClusteredLogSystem(db);
await logSystem.initialize();

// Logging
await logSystem.log('info', 'Application started', {
  service: 'api-gateway',
  version: '1.2.3'
});

await logSystem.log('error', 'Database connection failed', {
  service: 'user-service',
  error: 'ECONNREFUSED',
  host: 'db-primary'
});

// Requêtes temporelles optimisées
const lastHourLogs = await logSystem.getLogsByTimeRange(
  new Date(Date.now() - 3600000),
  new Date(),
  { level: 'error' }
);

// Tail logs (très rapide)
const recentLogs = await logSystem.getTailLogs(50);

// Analyse de patterns
const patterns = await logSystem.analyzeLogPatterns(24);
console.log('Log patterns:', patterns);

// Détection de spikes
const spike = await logSystem.detectErrorSpikes(100);
if (spike.spike) {
  console.warn('Error spike detected!', spike);
}

// Benchmark
const benchmark = await logSystem.performanceBenchmark();
console.log('Performance:', benchmark);
// Exemple: Scanned 50000 docs in 200ms (250k docs/sec)
```

### Cas 2 : Event Sourcing avec accès séquentiel

```javascript
class ClusteredEventStore {
  constructor(db) {
    this.db = db;
    this.events = null;
    this.snapshots = null;
  }

  async initialize() {
    // Collection clustered pour events
    try {
      await this.db.createCollection('event_store', {
        clusteredIndex: {
          key: { _id: 1 },
          unique: true
        }
      });

      // Collection standard pour snapshots
      await this.db.createCollection('event_snapshots');
    } catch (error) {
      if (error.code !== 48) throw error;
    }

    this.events = this.db.collection('event_store');
    this.snapshots = this.db.collection('event_snapshots');

    // Index sur aggregateId pour reconstruction
    await this.events.createIndex({ aggregateId: 1, _id: 1 });
    await this.snapshots.createIndex({ aggregateId: 1, version: -1 });
  }

  async appendEvent(aggregateId, eventType, eventData, metadata = {}) {
    // Obtenir la version actuelle
    const lastEvent = await this.events
      .find({ aggregateId })
      .sort({ _id: -1 })
      .limit(1)
      .toArray();

    const version = lastEvent.length > 0
      ? lastEvent[0].version + 1
      : 1;

    const event = {
      _id: new ObjectId(),  // Ordre chronologique garanti
      aggregateId,
      aggregateType: metadata.aggregateType || 'unknown',
      eventType,
      eventData,
      version,
      timestamp: new Date(),
      metadata: {
        userId: metadata.userId,
        correlationId: metadata.correlationId || new ObjectId().toString(),
        causationId: metadata.causationId
      }
    };

    await this.events.insertOne(event);

    // Créer snapshot périodiquement
    if (version % 100 === 0) {
      await this.createSnapshot(aggregateId, version);
    }

    return event;
  }

  async getAggregateEvents(aggregateId, fromVersion = 0) {
    // Range query très efficace grâce au clustering
    // Events ordonnés chronologiquement et physiquement adjacents
    return await this.events.find({
      aggregateId,
      version: { $gt: fromVersion }
    })
      .sort({ _id: 1 })
      .toArray();
  }

  async getEventsByTimeRange(startTime, endTime, options = {}) {
    const startId = ObjectId.createFromTime(
      Math.floor(startTime.getTime() / 1000)
    );
    const endId = ObjectId.createFromTime(
      Math.floor(endTime.getTime() / 1000)
    );

    const query = {
      _id: { $gte: startId, $lte: endId }
    };

    if (options.aggregateType) {
      query.aggregateType = options.aggregateType;
    }

    if (options.eventTypes) {
      query.eventType = { $in: options.eventTypes };
    }

    return await this.events.find(query)
      .sort({ _id: 1 })
      .limit(options.limit || 10000)
      .toArray();
  }

  async createSnapshot(aggregateId, version) {
    // Reconstruire l'état depuis les events
    const events = await this.getAggregateEvents(aggregateId);

    if (events.length === 0) return;

    // Replay events pour obtenir l'état
    const state = this.replayEvents(events);

    await this.snapshots.updateOne(
      { aggregateId },
      {
        $set: {
          aggregateId,
          version,
          state,
          timestamp: new Date()
        }
      },
      { upsert: true }
    );

    console.log(`Snapshot created for ${aggregateId} at version ${version}`);
  }

  replayEvents(events) {
    // Simplification - implémenter selon logique métier
    return events.reduce((state, event) => {
      switch (event.eventType) {
        case 'OrderCreated':
          return {
            ...state,
            orderId: event.aggregateId,
            status: 'created',
            items: event.eventData.items,
            total: event.eventData.total
          };

        case 'ItemAdded':
          return {
            ...state,
            items: [...(state.items || []), event.eventData.item],
            total: (state.total || 0) + event.eventData.item.price
          };

        case 'OrderConfirmed':
          return {
            ...state,
            status: 'confirmed',
            confirmedAt: event.timestamp
          };

        default:
          return state;
      }
    }, {});
  }

  async loadAggregate(aggregateId) {
    // Charger depuis snapshot si disponible
    const snapshot = await this.snapshots
      .find({ aggregateId })
      .sort({ version: -1 })
      .limit(1)
      .toArray();

    let state = {};
    let fromVersion = 0;

    if (snapshot.length > 0) {
      state = snapshot[0].state;
      fromVersion = snapshot[0].version;
    }

    // Charger events depuis snapshot
    const events = await this.getAggregateEvents(aggregateId, fromVersion);

    // Replay events depuis snapshot
    state = events.reduce((s, event) => this.replayEvent(s, event), state);

    return {
      aggregateId,
      state,
      version: fromVersion + events.length
    };
  }

  replayEvent(state, event) {
    // Appliquer un seul event
    switch (event.eventType) {
      case 'OrderCreated':
        return { ...state, status: 'created', ...event.eventData };
      case 'OrderConfirmed':
        return { ...state, status: 'confirmed' };
      default:
        return state;
    }
  }

  async getEventStatistics() {
    const [totalEvents, oldestEvent, newestEvent, aggregateCount] = await Promise.all([
      this.events.countDocuments(),

      this.events.find().sort({ _id: 1 }).limit(1).toArray(),

      this.events.find().sort({ _id: -1 }).limit(1).toArray(),

      this.events.distinct('aggregateId').then(ids => ids.length)
    ]);

    return {
      totalEvents,
      aggregateCount,
      oldestEvent: oldestEvent[0]?._id.getTimestamp(),
      newestEvent: newestEvent[0]?._id.getTimestamp(),
      timespan: oldestEvent.length > 0 && newestEvent.length > 0
        ? newestEvent[0]._id.getTimestamp() - oldestEvent[0]._id.getTimestamp()
        : 0
    };
  }

  async analyzeEventThroughput(intervalMinutes = 5) {
    const sinceTime = new Date(Date.now() - intervalMinutes * 60000);
    const sinceId = ObjectId.createFromTime(
      Math.floor(sinceTime.getTime() / 1000)
    );

    const throughput = await this.events.aggregate([
      { $match: { _id: { $gte: sinceId } } },
      {
        $group: {
          _id: {
            $dateToString: {
              format: '%Y-%m-%d %H:%M',
              date: { $toDate: '$_id' }
            }
          },
          eventCount: { $sum: 1 },
          eventTypes: { $addToSet: '$eventType' }
        }
      },
      { $sort: { _id: 1 } }
    ]).toArray();

    return throughput;
  }

  async archiveOldEvents(olderThanDays = 365) {
    const archiveDate = new Date(Date.now() - olderThanDays * 86400000);
    const archiveId = ObjectId.createFromTime(
      Math.floor(archiveDate.getTime() / 1000)
    );

    // Exporter vers archive
    const oldEvents = await this.events.find({
      _id: { $lt: archiveId }
    }).toArray();

    if (oldEvents.length > 0) {
      // Sauvegarder dans collection archive
      await this.db.collection('event_archive').insertMany(oldEvents, {
        ordered: false
      });

      // Supprimer de la collection principale
      const result = await this.events.deleteMany({
        _id: { $lt: archiveId }
      });

      return {
        archived: oldEvents.length,
        deleted: result.deletedCount
      };
    }

    return { archived: 0, deleted: 0 };
  }
}

// Utilisation
const eventStore = new ClusteredEventStore(db);
await eventStore.initialize();

// Append events
await eventStore.appendEvent(
  'order-123',
  'OrderCreated',
  {
    customerId: 'cust-456',
    items: [
      { sku: 'PROD-001', quantity: 2, price: 29.99 }
    ],
    total: 59.98
  },
  {
    aggregateType: 'Order',
    userId: 'user-789'
  }
);

await eventStore.appendEvent(
  'order-123',
  'OrderConfirmed',
  { confirmedAt: new Date() },
  { aggregateType: 'Order', userId: 'user-789' }
);

// Charger aggregate (avec snapshot optimization)
const order = await eventStore.loadAggregate('order-123');
console.log('Order state:', order.state);

// Requêtes temporelles
const todayEvents = await eventStore.getEventsByTimeRange(
  new Date(new Date().setHours(0, 0, 0, 0)),
  new Date()
);

// Statistiques
const stats = await eventStore.getEventStatistics();
console.log('Event store stats:', stats);

// Archivage
const archived = await eventStore.archiveOldEvents(365);
console.log('Archived:', archived);
```

### Cas 3 : Data Archiving avec compression temporelle

```javascript
class ClusteredDataArchive {
  constructor(db) {
    this.db = db;
    this.active = null;
    this.archive = null;
  }

  async initialize() {
    // Collection active (hot data)
    try {
      await this.db.createCollection('transactions_active', {
        clusteredIndex: {
          key: { _id: 1 },
          unique: true
        },
        expireAfterSeconds: 2592000  // 30 jours
      });

      // Collection archive (cold data) - aussi clustered
      await this.db.createCollection('transactions_archive', {
        clusteredIndex: {
          key: { _id: 1 },
          unique: true
        }
      });

    } catch (error) {
      if (error.code !== 48) throw error;
    }

    this.active = this.db.collection('transactions_active');
    this.archive = this.db.collection('transactions_archive');

    // Index secondaires
    await this.active.createIndex({ userId: 1, _id: -1 });
    await this.archive.createIndex({ userId: 1, _id: -1 });
    await this.archive.createIndex({ archivedAt: 1 });
  }

  async recordTransaction(userId, transactionData) {
    const transaction = {
      _id: new ObjectId(),
      userId,
      type: transactionData.type,
      amount: transactionData.amount,
      currency: transactionData.currency || 'USD',
      description: transactionData.description,
      metadata: transactionData.metadata || {},
      status: 'completed'
    };

    await this.active.insertOne(transaction);
    return transaction._id;
  }

  async getUserTransactions(userId, options = {}) {
    const {
      startDate = null,
      endDate = null,
      includeArchive = false,
      limit = 100
    } = options;

    const query = { userId };

    if (startDate || endDate) {
      query._id = {};
      if (startDate) {
        query._id.$gte = ObjectId.createFromTime(
          Math.floor(startDate.getTime() / 1000)
        );
      }
      if (endDate) {
        query._id.$lte = ObjectId.createFromTime(
          Math.floor(endDate.getTime() / 1000)
        );
      }
    }

    // Requêter active
    const activeTransactions = await this.active.find(query)
      .sort({ _id: -1 })
      .limit(limit)
      .toArray();

    if (!includeArchive) {
      return activeTransactions;
    }

    // Si besoin, ajouter depuis archive
    const remaining = limit - activeTransactions.length;
    if (remaining > 0) {
      const archivedTransactions = await this.archive.find(query)
        .sort({ _id: -1 })
        .limit(remaining)
        .toArray();

      return [...activeTransactions, ...archivedTransactions];
    }

    return activeTransactions;
  }

  async archiveOldTransactions(olderThanDays = 30) {
    const archiveDate = new Date(Date.now() - olderThanDays * 86400000);
    const archiveId = ObjectId.createFromTime(
      Math.floor(archiveDate.getTime() / 1000)
    );

    console.log(`Archiving transactions older than ${archiveDate}`);

    // Utiliser cursor pour traiter par batch
    const cursor = this.active.find({
      _id: { $lt: archiveId }
    }).batchSize(1000);

    let archived = 0;
    let batch = [];

    for await (const doc of cursor) {
      // Enrichir avec info d'archivage
      batch.push({
        ...doc,
        archivedAt: new Date(),
        archivedFrom: 'transactions_active'
      });

      if (batch.length >= 1000) {
        await this.archive.insertMany(batch, { ordered: false });

        // Supprimer de active
        const idsToDelete = batch.map(d => d._id);
        await this.active.deleteMany({
          _id: { $in: idsToDelete }
        });

        archived += batch.length;
        batch = [];

        console.log(`Archived ${archived} transactions...`);
      }
    }

    // Batch final
    if (batch.length > 0) {
      await this.archive.insertMany(batch, { ordered: false });
      const idsToDelete = batch.map(d => d._id);
      await this.active.deleteMany({ _id: { $in: idsToDelete } });
      archived += batch.length;
    }

    console.log(`Archive complete: ${archived} transactions moved`);
    return archived;
  }

  async generateMonthlyReport(year, month) {
    // Générer rapport mensuel depuis active et archive
    const startDate = new Date(year, month - 1, 1);
    const endDate = new Date(year, month, 0, 23, 59, 59);

    const startId = ObjectId.createFromTime(
      Math.floor(startDate.getTime() / 1000)
    );
    const endId = ObjectId.createFromTime(
      Math.floor(endDate.getTime() / 1000)
    );

    const query = {
      _id: { $gte: startId, $lte: endId }
    };

    const pipeline = [
      { $match: query },
      {
        $group: {
          _id: {
            type: '$type',
            currency: '$currency'
          },
          count: { $sum: 1 },
          totalAmount: { $sum: '$amount' },
          avgAmount: { $avg: '$amount' }
        }
      },
      { $sort: { totalAmount: -1 } }
    ];

    // Exécuter sur les deux collections
    const [activeReport, archiveReport] = await Promise.all([
      this.active.aggregate(pipeline).toArray(),
      this.archive.aggregate(pipeline).toArray()
    ]);

    // Fusionner les résultats
    return this.mergeReports(activeReport, archiveReport);
  }

  mergeReports(report1, report2) {
    const merged = new Map();

    [...report1, ...report2].forEach(item => {
      const key = `${item._id.type}-${item._id.currency}`;

      if (merged.has(key)) {
        const existing = merged.get(key);
        merged.set(key, {
          _id: item._id,
          count: existing.count + item.count,
          totalAmount: existing.totalAmount + item.totalAmount,
          avgAmount: (existing.totalAmount + item.totalAmount) /
                     (existing.count + item.count)
        });
      } else {
        merged.set(key, item);
      }
    });

    return Array.from(merged.values());
  }

  async getStorageStatistics() {
    const [activeStats, archiveStats] = await Promise.all([
      this.active.stats(),
      this.archive.stats()
    ]);

    return {
      active: {
        count: activeStats.count,
        size: activeStats.size,
        storageSize: activeStats.storageSize,
        avgObjSize: activeStats.avgObjSize
      },
      archive: {
        count: archiveStats.count,
        size: archiveStats.size,
        storageSize: archiveStats.storageSize,
        avgObjSize: archiveStats.avgObjSize
      },
      total: {
        count: activeStats.count + archiveStats.count,
        storageSize: activeStats.storageSize + archiveStats.storageSize
      }
    };
  }

  async restoreFromArchive(transactionIds) {
    // Restaurer des transactions depuis l'archive
    const transactions = await this.archive.find({
      _id: { $in: transactionIds }
    }).toArray();

    if (transactions.length > 0) {
      // Retirer info d'archivage
      const restored = transactions.map(t => {
        const { archivedAt, archivedFrom, ...original } = t;
        return original;
      });

      await this.active.insertMany(restored, { ordered: false });
      await this.archive.deleteMany({ _id: { $in: transactionIds } });

      return restored.length;
    }

    return 0;
  }
}

// Utilisation
const archive = new ClusteredDataArchive(db);
await archive.initialize();

// Enregistrer transactions
await archive.recordTransaction('user-123', {
  type: 'payment',
  amount: 99.99,
  currency: 'USD',
  description: 'Subscription renewal'
});

// Récupérer transactions utilisateur
const transactions = await archive.getUserTransactions('user-123', {
  startDate: new Date('2024-01-01'),
  endDate: new Date('2024-12-31'),
  includeArchive: true
});

// Archivage périodique
setInterval(async () => {
  const archived = await archive.archiveOldTransactions(30);
  console.log(`Archived ${archived} transactions`);
}, 86400000);  // Quotidien

// Rapport mensuel
const report = await archive.generateMonthlyReport(2024, 12);
console.log('Monthly report:', report);

// Statistiques
const stats = await archive.getStorageStatistics();
console.log('Storage stats:', stats);
```

---

## Benchmark et comparaisons

### Performance des range queries

```javascript
class ClusteredVsStandardBenchmark {
  constructor(db) {
    this.db = db;
  }

  async setupCollections() {
    // Collection clustered
    await this.db.createCollection('data_clustered', {
      clusteredIndex: {
        key: { _id: 1 },
        unique: true
      }
    });

    // Collection standard
    await this.db.collection('data_standard').createIndex({ _id: 1 });
  }

  async populateData(count = 100000) {
    console.log(`Populating ${count} documents...`);

    const batchSize = 1000;
    for (let i = 0; i < count; i += batchSize) {
      const batch = Array.from({ length: Math.min(batchSize, count - i) }, (_, j) => {
        const timestamp = new Date(Date.now() - (count - i - j) * 1000);
        return {
          _id: ObjectId.createFromTime(Math.floor(timestamp.getTime() / 1000)),
          value: Math.random() * 1000,
          data: 'x'.repeat(500)  // Payload
        };
      });

      await Promise.all([
        this.db.collection('data_clustered').insertMany([...batch], { ordered: false }),
        this.db.collection('data_standard').insertMany([...batch], { ordered: false })
      ]);
    }

    console.log('Data populated');
  }

  async benchmarkRangeQuery(rangeSize = 10000) {
    const now = new Date();
    const startTime = new Date(now - rangeSize * 1000);

    const startId = ObjectId.createFromTime(Math.floor(startTime.getTime() / 1000));
    const endId = ObjectId.createFromTime(Math.floor(now.getTime() / 1000));

    // Benchmark clustered
    const clusteredStart = Date.now();
    const clusteredResults = await this.db.collection('data_clustered')
      .find({ _id: { $gte: startId, $lte: endId } })
      .toArray();
    const clusteredDuration = Date.now() - clusteredStart;

    // Benchmark standard
    const standardStart = Date.now();
    const standardResults = await this.db.collection('data_standard')
      .find({ _id: { $gte: startId, $lte: endId } })
      .toArray();
    const standardDuration = Date.now() - standardStart;

    return {
      clustered: {
        duration: clusteredDuration,
        resultsCount: clusteredResults.length,
        docsPerMs: (clusteredResults.length / clusteredDuration).toFixed(2)
      },
      standard: {
        duration: standardDuration,
        resultsCount: standardResults.length,
        docsPerMs: (standardResults.length / standardDuration).toFixed(2)
      },
      improvement: `${((1 - clusteredDuration / standardDuration) * 100).toFixed(2)}%`
    };
  }

  async benchmarkSequentialScan() {
    // Scan séquentiel complet
    const clusteredStart = Date.now();
    const clusteredCount = await this.db.collection('data_clustered')
      .find()
      .sort({ _id: 1 })
      .toArray()
      .then(r => r.length);
    const clusteredDuration = Date.now() - clusteredStart;

    const standardStart = Date.now();
    const standardCount = await this.db.collection('data_standard')
      .find()
      .sort({ _id: 1 })
      .toArray()
      .then(r => r.length);
    const standardDuration = Date.now() - standardStart;

    return {
      clustered: { duration: clusteredDuration, count: clusteredCount },
      standard: { duration: standardDuration, count: standardCount },
      improvement: `${((1 - clusteredDuration / standardDuration) * 100).toFixed(2)}%`
    };
  }

  async compareStorageSize() {
    const [clusteredStats, standardStats] = await Promise.all([
      this.db.collection('data_clustered').stats(),
      this.db.collection('data_standard').stats()
    ]);

    return {
      clustered: {
        storageSize: clusteredStats.storageSize,
        indexSize: clusteredStats.totalIndexSize,
        total: clusteredStats.storageSize + clusteredStats.totalIndexSize
      },
      standard: {
        storageSize: standardStats.storageSize,
        indexSize: standardStats.totalIndexSize,
        total: standardStats.storageSize + standardStats.totalIndexSize
      },
      savings: {
        bytes: (standardStats.storageSize + standardStats.totalIndexSize) -
               (clusteredStats.storageSize + clusteredStats.totalIndexSize),
        percent: ((1 - (clusteredStats.storageSize + clusteredStats.totalIndexSize) /
                      (standardStats.storageSize + standardStats.totalIndexSize)) * 100).toFixed(2)
      }
    };
  }
}

// Utilisation
const benchmark = new ClusteredVsStandardBenchmark(db);
await benchmark.setupCollections();
await benchmark.populateData(100000);

// Benchmark range query
const rangeResults = await benchmark.benchmarkRangeQuery(10000);
console.log('Range Query Benchmark:', rangeResults);
// Exemple: Clustered 30-50% plus rapide

// Benchmark scan séquentiel
const scanResults = await benchmark.benchmarkSequentialScan();
console.log('Sequential Scan Benchmark:', scanResults);
// Exemple: Clustered 40-60% plus rapide

// Comparaison stockage
const storageResults = await benchmark.compareStorageSize();
console.log('Storage Comparison:', storageResults);
// Exemple: Clustered 10-15% moins d'espace (pas d'index _id séparé)
```

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Utiliser pour workloads temporels/séquentiels
await db.createCollection('time_ordered_data', {
  clusteredIndex: { key: { _id: 1 }, unique: true }
});

// 2. Combiner avec TTL pour auto-expiration
await db.createCollection('ephemeral_data', {
  clusteredIndex: { key: { _id: 1 }, unique: true },
  expireAfterSeconds: 86400  // 24h
});

// 3. Insertions en ordre chronologique (optimal)
const docs = [];
for (let i = 0; i < 1000; i++) {
  docs.push({
    _id: new ObjectId(),  // Timestamp embedded, ordre chronologique
    data: someData
  });
}
await collection.insertMany(docs);

// 4. Range queries avec _id
const results = await collection.find({
  _id: {
    $gte: ObjectId.createFromTime(startTimestamp),
    $lte: ObjectId.createFromTime(endTimestamp)
  }
}).toArray();

// 5. Index secondaires pour autres patterns
await collection.createIndex({ userId: 1, _id: -1 });
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas utiliser pour insertions aléatoires
// ❌ Perd l'avantage du clustering
const randomId = new ObjectId(); // Ordre aléatoire
randomId.setTimestamp(randomPastTimestamp);

// 2. Ne pas essayer de cluster sur autre champ que _id
// ❌ INVALIDE
await db.createCollection('invalid', {
  clusteredIndex: { key: { timestamp: 1 }, unique: true }
});

// 3. Ne pas utiliser si workload non séquentiel
// ❌ Pas d'avantage pour accès purement aléatoires
await collection.find({ randomField: value });

// 4. Ne pas oublier que conversion impossible
// ❌ Doit créer nouvelle collection et migrer
// Pas de convertToClustered()

// 5. Ne pas utiliser pour collections avec nombreux index
// Le bénéfice principal est l'absence d'index _id séparé
// Si 10+ index, gain marginal
```

---

## Migration vers Clustered Collections

```javascript
async function migrateToClusteredCollection(db, sourceCollection, targetCollection) {
  console.log(`Migrating ${sourceCollection} to clustered collection ${targetCollection}`);

  // 1. Créer collection clustered
  await db.createCollection(targetCollection, {
    clusteredIndex: {
      key: { _id: 1 },
      unique: true
    }
  });

  // 2. Copier les index secondaires
  const indexes = await db.collection(sourceCollection).indexes();
  for (const index of indexes) {
    if (index.name === '_id_') continue;  // Skip index _id

    const keys = index.key;
    const options = {
      name: index.name,
      unique: index.unique || false,
      sparse: index.sparse || false
    };

    await db.collection(targetCollection).createIndex(keys, options);
  }

  // 3. Migration des données par batch
  const totalDocs = await db.collection(sourceCollection).countDocuments();
  const batchSize = 10000;
  let migrated = 0;

  console.log(`Total documents: ${totalDocs}`);

  while (migrated < totalDocs) {
    const batch = await db.collection(sourceCollection)
      .find()
      .skip(migrated)
      .limit(batchSize)
      .toArray();

    if (batch.length === 0) break;

    await db.collection(targetCollection).insertMany(batch, { ordered: false });

    migrated += batch.length;
    console.log(`Migrated: ${migrated}/${totalDocs} (${(migrated/totalDocs*100).toFixed(2)}%)`);
  }

  // 4. Vérification
  const targetCount = await db.collection(targetCollection).countDocuments();
  console.log(`Migration complete. Source: ${totalDocs}, Target: ${targetCount}`);

  // 5. (Optionnel) Renommer collections après validation
  // await db.collection(sourceCollection).rename(`${sourceCollection}_old`);
  // await db.collection(targetCollection).rename(sourceCollection);

  return { source: totalDocs, target: targetCount };
}

// Utilisation
await migrateToClusteredCollection(db, 'events', 'events_clustered');
```

---

## Conclusion

Les Clustered Collections sont une fonctionnalité avancée optimisant les performances pour :
- ✅ **Range queries temporelles** (30-50% plus rapide)
- ✅ **Scans séquentiels** (40-60% plus rapide)
- ✅ **Économie stockage** (10-15% grâce à l'absence d'index _id séparé)
- ✅ **Réduction fragmentation** (documents adjacents logiquement et physiquement)
- ✅ **TTL efficace** (suppression de plages optimisée)

**Points clés à retenir :**
1. Clé de clustering DOIT être `_id`
2. Optimal pour données ordonnées chronologiquement
3. ObjectId contient timestamp embedded (parfait pour clustering)
4. Impossible de convertir collection existante (migration nécessaire)
5. Meilleur pour workloads avec accès séquentiels/temporels

**Cas d'usage idéaux :**
- Logs avec accès par plages temporelles
- Event sourcing avec replay chronologique
- Time series (si pas besoin fonctionnalités time series collections)
- Data archiving avec accès séquentiel
- Audit trails ordonnés temporellement

**Quand ne pas utiliser :**
- Accès purement aléatoires
- Insertions désordonnées fréquentes
- Nombreux index secondaires (> 10)
- Besoin de clé de clustering autre que _id

---


⏭️ [Requêtes géospatiales avancées](/16-fonctionnalites-avancees/06-requetes-geospatiales-avancees.md)
