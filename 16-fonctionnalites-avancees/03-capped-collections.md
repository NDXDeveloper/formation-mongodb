🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.3 Capped Collections

## Introduction

Les **Capped Collections** sont des collections MongoDB de taille fixe qui maintiennent automatiquement l'ordre d'insertion et suppriment les documents les plus anciens lorsque la limite de taille est atteinte. Elles fonctionnent comme un **buffer circulaire** (ring buffer) avec des performances d'insertion exceptionnelles, idéales pour les logs, les caches éphémères, et les files de messages temporaires.

Contrairement aux collections standard, les capped collections garantissent l'ordre d'insertion et offrent des performances d'écriture optimisées au détriment de certaines fonctionnalités (pas de suppression individuelle, pas de croissance de documents).

---

## Caractéristiques fondamentales

### Architecture et comportement

```
┌────────────────────────────────────────────────────────────┐
│              Capped Collection (FIFO)                      │
│                                                            │
│  Taille fixe: 10 MB                                        │
│  ┌────────────────────────────────────────────────────┐    │
│  │ [Doc1] [Doc2] [Doc3] ... [Doc998] [Doc999]         │    │
│  │   ▲                                        ▲       │    │
│  │   │ Plus ancien                   Plus récent      │    │
│  │   │                                                │    │
│  └───┼────────────────────────────────────────────────┘    │
│      │                                                     │
│      │ Quand limite atteinte:                              │
│      │ Doc1 est supprimé automatiquement                   │
│      │ Nouveau document ajouté à la fin                    │
│      ▼                                                     │
│  ┌────────────────────────────────────────────────────┐    │
│  │ [Doc2] [Doc3] ... [Doc999] [Doc1000]               │    │
│  │   ▲                                        ▲       │    │
│  └───┼────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

### Propriétés clés

| Propriété | Comportement | Impact |
|-----------|--------------|--------|
| **Taille fixe** | Ne peut pas croître au-delà de la limite | Prévisibilité mémoire/disque |
| **FIFO automatique** | Suppression auto des plus anciens | Pas de gestion manuelle |
| **Ordre d'insertion** | Toujours maintenu | Lectures séquentielles rapides |
| **Pas de _id index** | Pas d'index par défaut (optionnel) | Performance insertion maximale |
| **Insertion rapide** | Pas de fragmentation | ~10-20% plus rapide |
| **Pas de suppression** | deleteOne/deleteMany interdits | Opérations limitées |
| **Pas de croissance docs** | Updates ne peuvent agrandir | Design spécifique requis |

---

## Création et configuration

### Syntaxe de création

```javascript
// Création avec taille en bytes
await db.createCollection('logs', {
  capped: true,
  size: 10485760  // 10 MB
});

// Création avec taille ET nombre max de documents
await db.createCollection('events', {
  capped: true,
  size: 5242880,   // 5 MB (prioritaire)
  max: 10000       // Max 10000 docs (secondaire)
});

// Note: La limite de taille (size) est toujours respectée en premier
// Si taille atteinte avant max documents, les plus anciens sont supprimés
```

### Conversion d'une collection existante

```javascript
// Convertir une collection standard en capped
await db.runCommand({
  convertToCapped: 'mylogs',
  size: 104857600  // 100 MB
});

// ⚠️ Attention: Opération coûteuse, bloque la collection
// Recommandé uniquement pendant maintenance window
```

### Vérification des propriétés

```javascript
// Obtenir les stats d'une capped collection
const stats = await db.collection('logs').stats();

console.log({
  capped: stats.capped,           // true
  size: stats.size,                // Taille actuelle utilisée
  maxSize: stats.maxSize,          // Taille maximale configurée
  count: stats.count,              // Nombre de documents actuels
  storageSize: stats.storageSize   // Espace disque utilisé
});

// Vérifier si une collection est capped
const isCapped = await db.collection('logs').isCapped();
console.log('Is capped:', isCapped);
```

---

## Opérations et limitations

### Opérations autorisées

```javascript
// ✅ INSERT - Performance optimale
await db.collection('logs').insertOne({
  level: 'info',
  message: 'Application started',
  timestamp: new Date()
});

await db.collection('logs').insertMany([
  { level: 'debug', message: 'Debug info 1', timestamp: new Date() },
  { level: 'debug', message: 'Debug info 2', timestamp: new Date() }
]);

// ✅ FIND - Ordre d'insertion maintenu
const recentLogs = await db.collection('logs')
  .find()
  .sort({ $natural: -1 })  // -1 = plus récent d'abord, 1 = plus ancien
  .limit(100)
  .toArray();

// ✅ UPDATE - Si ne change pas la taille
await db.collection('logs').updateOne(
  { _id: logId },
  { $set: { processed: true } }  // ✅ OK si ne grandit pas le document
);

// ✅ AGGREGATE
const errorCount = await db.collection('logs').aggregate([
  { $match: { level: 'error' } },
  { $count: 'total' }
]).toArray();
```

### Opérations interdites ou limitées

```javascript
// ❌ DELETE - Interdit
try {
  await db.collection('logs').deleteOne({ _id: logId });
} catch (error) {
  // Error: cannot remove from a capped collection
}

// ❌ UPDATE qui agrandit le document
try {
  await db.collection('logs').updateOne(
    { _id: logId },
    { $set: { veryLongField: 'x'.repeat(10000) } }
  );
} catch (error) {
  // Error: cannot change the size of a document in a capped collection
}

// ⚠️ REPLACE - Doit être exactement même taille
await db.collection('logs').replaceOne(
  { _id: logId },
  newDoc  // Doit faire exactement la même taille en bytes
);

// ✅ DROP - Vide la collection sans supprimer la structure
await db.collection('logs').drop();

// Puis recréer
await db.createCollection('logs', {
  capped: true,
  size: 10485760
});
```

---

## Cas d'usage avancés

### Cas 1 : Système de logs rotatifs haute performance

```javascript
class RotatingLogSystem {
  constructor(db, logSizes = {}) {
    this.db = db;
    this.logSizes = {
      error: 50 * 1024 * 1024,   // 50 MB pour errors
      warning: 30 * 1024 * 1024,  // 30 MB pour warnings
      info: 100 * 1024 * 1024,   // 100 MB pour info
      debug: 20 * 1024 * 1024,   // 20 MB pour debug (court terme)
      ...logSizes
    };
  }

  async initialize() {
    // Créer les collections capped pour chaque niveau
    for (const [level, size] of Object.entries(this.logSizes)) {
      const collectionName = `logs_${level}`;

      try {
        await this.db.createCollection(collectionName, {
          capped: true,
          size: size
        });
        console.log(`Created capped collection: ${collectionName} (${size} bytes)`);
      } catch (error) {
        if (error.code === 48) {
          // Collection déjà existe
          console.log(`Collection ${collectionName} already exists`);
        } else {
          throw error;
        }
      }

      // Index optionnel sur timestamp pour requêtes temporelles
      await this.db.collection(collectionName).createIndex(
        { timestamp: 1 }
      );
    }
  }

  async log(level, message, metadata = {}) {
    const collectionName = `logs_${level}`;

    const logEntry = {
      timestamp: new Date(),
      level,
      message,
      pid: process.pid,
      hostname: require('os').hostname(),
      ...metadata
    };

    try {
      await this.db.collection(collectionName).insertOne(logEntry);
    } catch (error) {
      // Fallback si insertion échoue (collection pleine impossible car capped)
      console.error('Log insertion failed:', error);
    }
  }

  async error(message, metadata = {}) {
    await this.log('error', message, metadata);
  }

  async warning(message, metadata = {}) {
    await this.log('warning', message, metadata);
  }

  async info(message, metadata = {}) {
    await this.log('info', message, metadata);
  }

  async debug(message, metadata = {}) {
    await this.log('debug', message, metadata);
  }

  async query(level, options = {}) {
    const {
      since,
      until,
      limit = 100,
      pattern
    } = options;

    const query = {};

    if (since || until) {
      query.timestamp = {};
      if (since) query.timestamp.$gte = since;
      if (until) query.timestamp.$lte = until;
    }

    if (pattern) {
      query.message = { $regex: pattern, $options: 'i' };
    }

    return await this.db.collection(`logs_${level}`)
      .find(query)
      .sort({ $natural: -1 })  // Plus récent d'abord
      .limit(limit)
      .toArray();
  }

  async getRecentErrors(limit = 50) {
    return await this.query('error', { limit });
  }

  async searchLogs(searchText, levels = ['error', 'warning', 'info']) {
    const results = await Promise.all(
      levels.map(level =>
        this.query(level, {
          pattern: searchText,
          limit: 20
        })
      )
    );

    // Fusionner et trier par timestamp
    return results
      .flat()
      .sort((a, b) => b.timestamp - a.timestamp);
  }

  async getStatistics() {
    const stats = {};

    for (const level of Object.keys(this.logSizes)) {
      const collStats = await this.db.collection(`logs_${level}`).stats();

      stats[level] = {
        count: collStats.count,
        size: collStats.size,
        maxSize: collStats.maxSize,
        usagePercent: ((collStats.size / collStats.maxSize) * 100).toFixed(2)
      };
    }

    return stats;
  }

  async tail(level, callback, options = {}) {
    // Tail -f like functionality avec tailable cursor
    const collection = this.db.collection(`logs_${level}`);

    // Créer un curseur tailable
    const cursor = collection.find(
      {},
      {
        tailable: true,
        awaitData: true,
        numberOfRetries: Number.MAX_VALUE,
        tailableRetryInterval: 100
      }
    ).sort({ $natural: 1 });  // Ordre d'insertion

    // Commencer après le dernier document actuel si demandé
    if (options.startFromEnd) {
      const lastDoc = await collection
        .find()
        .sort({ $natural: -1 })
        .limit(1)
        .toArray();

      if (lastDoc.length > 0) {
        cursor.filter({ _id: { $gt: lastDoc[0]._id } });
      }
    }

    // Stream les nouveaux logs
    cursor.stream().on('data', (log) => {
      callback(log);
    });

    return cursor;
  }
}

// Utilisation
const logger = new RotatingLogSystem(db);
await logger.initialize();

// Logging simple
await logger.error('Database connection failed', {
  error: 'ECONNREFUSED',
  host: 'localhost',
  port: 27017
});

await logger.info('User logged in', {
  userId: 'user-123',
  ip: '192.168.1.100'
});

// Requêtes
const recentErrors = await logger.getRecentErrors(10);
const searchResults = await logger.searchLogs('connection');

// Statistiques
const stats = await logger.getStatistics();
console.log('Log statistics:', stats);

// Tail logs en temps réel
const tailCursor = await logger.tail('error', (log) => {
  console.log(`[${log.timestamp}] ${log.level}: ${log.message}`);
}, { startFromEnd: true });

// Exemple de sortie:
// [2024-12-08T15:30:45.123Z] error: Database connection failed
```

### Cas 2 : Cache éphémère haute performance

```javascript
class CappedCache {
  constructor(db, collectionName, options = {}) {
    this.db = db;
    this.collectionName = collectionName;
    this.maxSize = options.maxSize || 10 * 1024 * 1024; // 10 MB par défaut
    this.maxDocs = options.maxDocs || 10000;
    this.defaultTTL = options.defaultTTL || 3600; // 1h par défaut
  }

  async initialize() {
    // Créer la capped collection
    try {
      await this.db.createCollection(this.collectionName, {
        capped: true,
        size: this.maxSize,
        max: this.maxDocs
      });
    } catch (error) {
      if (error.code !== 48) { // Pas "collection already exists"
        throw error;
      }
    }

    // Index sur la clé pour recherche rapide
    await this.db.collection(this.collectionName).createIndex(
      { key: 1 },
      { unique: true }
    );

    // Index sur expiration
    await this.db.collection(this.collectionName).createIndex(
      { expiresAt: 1 }
    );
  }

  async set(key, value, ttl = null) {
    const expiresAt = new Date(
      Date.now() + (ttl || this.defaultTTL) * 1000
    );

    const cacheEntry = {
      key,
      value,
      createdAt: new Date(),
      expiresAt,
      accessCount: 0,
      lastAccessed: new Date()
    };

    try {
      // Insert ou update
      await this.db.collection(this.collectionName).updateOne(
        { key },
        { $set: cacheEntry },
        { upsert: true }
      );

      return true;
    } catch (error) {
      if (error.code === 11000) {
        // Duplicate key - rare car upsert, mais possible en race condition
        // Réessayer
        return await this.set(key, value, ttl);
      }
      throw error;
    }
  }

  async get(key) {
    const entry = await this.db.collection(this.collectionName).findOne({
      key,
      expiresAt: { $gt: new Date() }  // Pas expiré
    });

    if (!entry) {
      return null;
    }

    // Update access statistics (fire and forget)
    this.db.collection(this.collectionName).updateOne(
      { key },
      {
        $inc: { accessCount: 1 },
        $set: { lastAccessed: new Date() }
      }
    ).catch(() => {}); // Ignorer les erreurs (peut échouer si doc supprimé entre temps)

    return entry.value;
  }

  async has(key) {
    const count = await this.db.collection(this.collectionName).countDocuments({
      key,
      expiresAt: { $gt: new Date() }
    });

    return count > 0;
  }

  async delete(key) {
    // On ne peut pas delete dans une capped collection
    // Workaround: marquer comme expiré immédiatement
    await this.db.collection(this.collectionName).updateOne(
      { key },
      { $set: { expiresAt: new Date(0) } }  // Expiré dans le passé
    );
  }

  async getOrSet(key, factory, ttl = null) {
    // Essayer de récupérer du cache
    let value = await this.get(key);

    if (value !== null) {
      return { value, cached: true };
    }

    // Cache miss - générer la valeur
    value = await factory();

    // Stocker dans le cache
    await this.set(key, value, ttl);

    return { value, cached: false };
  }

  async clear() {
    // Drop et recréer
    await this.db.collection(this.collectionName).drop();
    await this.initialize();
  }

  async getStatistics() {
    const stats = await this.db.collection(this.collectionName).stats();

    const [totalEntries, expiredEntries, mostAccessed] = await Promise.all([
      this.db.collection(this.collectionName).countDocuments({}),

      this.db.collection(this.collectionName).countDocuments({
        expiresAt: { $lte: new Date() }
      }),

      this.db.collection(this.collectionName)
        .find()
        .sort({ accessCount: -1 })
        .limit(10)
        .project({ key: 1, accessCount: 1, lastAccessed: 1 })
        .toArray()
    ]);

    return {
      totalEntries,
      validEntries: totalEntries - expiredEntries,
      expiredEntries,
      size: stats.size,
      maxSize: stats.maxSize,
      usagePercent: ((stats.size / stats.maxSize) * 100).toFixed(2),
      mostAccessed
    };
  }

  async cleanupExpired() {
    // Marquer tous les docs expirés pour rotation éventuelle
    // (ils seront naturellement supprimés quand la limite est atteinte)
    const result = await this.db.collection(this.collectionName).updateMany(
      { expiresAt: { $lte: new Date() } },
      { $set: { _expired: true } }
    );

    return result.modifiedCount;
  }
}

// Utilisation
const cache = new CappedCache(db, 'session_cache', {
  maxSize: 50 * 1024 * 1024,  // 50 MB
  maxDocs: 50000,
  defaultTTL: 1800  // 30 minutes
});

await cache.initialize();

// Set/Get basique
await cache.set('user:123:profile', {
  name: 'John Doe',
  email: 'john@example.com'
}, 3600);  // 1h TTL

const profile = await cache.get('user:123:profile');

// Pattern getOrSet
const userData = await cache.getOrSet(
  'user:456:data',
  async () => {
    // Fonction coûteuse à exécuter seulement si cache miss
    return await fetchUserFromDatabase('456');
  },
  1800  // 30 min TTL
);

console.log('Cached:', userData.cached);

// Statistiques
const stats = await cache.getStatistics();
console.log('Cache stats:', stats);

// Cleanup périodique
setInterval(async () => {
  const cleaned = await cache.cleanupExpired();
  console.log(`Cleaned ${cleaned} expired entries`);
}, 60000); // Toutes les minutes
```

### Cas 3 : Message Queue avec priorités

```javascript
class CappedMessageQueue {
  constructor(db, queueName) {
    this.db = db;
    this.queueName = queueName;
    this.priorities = ['critical', 'high', 'normal', 'low'];
  }

  async initialize() {
    // Créer une capped collection par priorité
    for (const priority of this.priorities) {
      const collectionName = `${this.queueName}_${priority}`;
      const size = this.getSizeForPriority(priority);

      try {
        await this.db.createCollection(collectionName, {
          capped: true,
          size: size
        });

        // Index sur status pour requêtes
        await this.db.collection(collectionName).createIndex({
          status: 1,
          createdAt: 1
        });

      } catch (error) {
        if (error.code !== 48) throw error;
      }
    }
  }

  getSizeForPriority(priority) {
    const sizes = {
      critical: 10 * 1024 * 1024,   // 10 MB
      high: 20 * 1024 * 1024,       // 20 MB
      normal: 50 * 1024 * 1024,     // 50 MB
      low: 30 * 1024 * 1024         // 30 MB
    };
    return sizes[priority] || sizes.normal;
  }

  async enqueue(message, priority = 'normal') {
    if (!this.priorities.includes(priority)) {
      throw new Error(`Invalid priority: ${priority}`);
    }

    const collectionName = `${this.queueName}_${priority}`;
    const queueMessage = {
      payload: message,
      status: 'pending',
      priority,
      createdAt: new Date(),
      attempts: 0,
      lockedUntil: null,
      error: null
    };

    const result = await this.db.collection(collectionName).insertOne(queueMessage);
    return result.insertedId;
  }

  async dequeue(options = {}) {
    const {
      priorities = this.priorities,
      lockDuration = 30000  // 30 secondes par défaut
    } = options;

    // Essayer chaque priorité dans l'ordre
    for (const priority of priorities) {
      const collectionName = `${this.queueName}_${priority}`;
      const now = new Date();
      const lockUntil = new Date(now.getTime() + lockDuration);

      // Chercher et locker un message
      const message = await this.db.collection(collectionName).findOneAndUpdate(
        {
          status: 'pending',
          $or: [
            { lockedUntil: null },
            { lockedUntil: { $lte: now } }  // Lock expiré
          ]
        },
        {
          $set: {
            status: 'processing',
            lockedUntil: lockUntil
          },
          $inc: { attempts: 1 }
        },
        {
          sort: { createdAt: 1 },  // FIFO
          returnDocument: 'after'
        }
      );

      if (message.value) {
        return {
          id: message.value._id,
          payload: message.value.payload,
          priority: message.value.priority,
          attempts: message.value.attempts,
          collectionName
        };
      }
    }

    return null;  // Queue vide
  }

  async complete(messageId, collectionName) {
    await this.db.collection(collectionName).updateOne(
      { _id: messageId },
      {
        $set: {
          status: 'completed',
          completedAt: new Date(),
          lockedUntil: null
        }
      }
    );
  }

  async fail(messageId, collectionName, error, retry = true) {
    const update = {
      $set: {
        status: retry ? 'pending' : 'failed',
        error: error.message || error,
        failedAt: new Date(),
        lockedUntil: null
      }
    };

    await this.db.collection(collectionName).updateOne(
      { _id: messageId },
      update
    );
  }

  async worker(handler, options = {}) {
    const {
      concurrency = 1,
      pollInterval = 1000,
      maxAttempts = 3
    } = options;

    const processingPromises = new Set();

    const processNext = async () => {
      const message = await this.dequeue();

      if (!message) {
        // Queue vide, attendre
        await new Promise(resolve => setTimeout(resolve, pollInterval));
        return;
      }

      // Vérifier le nombre de tentatives
      if (message.attempts > maxAttempts) {
        await this.fail(
          message.id,
          message.collectionName,
          `Max attempts (${maxAttempts}) exceeded`,
          false
        );
        return;
      }

      // Traiter le message
      try {
        await handler(message.payload);
        await this.complete(message.id, message.collectionName);
      } catch (error) {
        console.error('Message processing failed:', error);
        await this.fail(
          message.id,
          message.collectionName,
          error,
          message.attempts < maxAttempts
        );
      }
    };

    // Boucle de traitement
    const running = true;
    while (running) {
      // Maintenir le niveau de concurrence
      while (processingPromises.size < concurrency) {
        const promise = processNext()
          .catch(error => console.error('Worker error:', error))
          .finally(() => processingPromises.delete(promise));

        processingPromises.add(promise);
      }

      // Attendre qu'au moins un se termine
      await Promise.race(Array.from(processingPromises));
    }
  }

  async getStatistics() {
    const stats = {};

    for (const priority of this.priorities) {
      const collectionName = `${this.queueName}_${priority}`;

      const [pending, processing, completed, failed] = await Promise.all([
        this.db.collection(collectionName).countDocuments({ status: 'pending' }),
        this.db.collection(collectionName).countDocuments({ status: 'processing' }),
        this.db.collection(collectionName).countDocuments({ status: 'completed' }),
        this.db.collection(collectionName).countDocuments({ status: 'failed' })
      ]);

      stats[priority] = {
        pending,
        processing,
        completed,
        failed,
        total: pending + processing + completed + failed
      };
    }

    return stats;
  }

  async clearQueue(priority = null) {
    const prioritiesToClear = priority ? [priority] : this.priorities;

    for (const p of prioritiesToClear) {
      const collectionName = `${this.queueName}_${p}`;
      await this.db.collection(collectionName).drop();
    }

    await this.initialize();
  }
}

// Utilisation
const queue = new CappedMessageQueue(db, 'tasks');
await queue.initialize();

// Enqueue des messages
await queue.enqueue(
  { type: 'email', to: 'user@example.com', subject: 'Welcome' },
  'critical'
);

await queue.enqueue(
  { type: 'report', userId: '123' },
  'normal'
);

// Worker pour traiter les messages
queue.worker(async (payload) => {
  console.log('Processing:', payload);

  switch (payload.type) {
    case 'email':
      await sendEmail(payload);
      break;
    case 'report':
      await generateReport(payload);
      break;
  }
}, {
  concurrency: 5,
  pollInterval: 500,
  maxAttempts: 3
});

// Statistiques
const stats = await queue.getStatistics();
console.log('Queue stats:', stats);
```

### Cas 4 : Monitoring et métriques en temps réel

```javascript
class RealtimeMetricsCollector {
  constructor(db) {
    this.db = db;
    this.metricsTypes = ['system', 'application', 'business'];
  }

  async initialize() {
    for (const type of this.metricsTypes) {
      const collectionName = `metrics_${type}`;

      try {
        await this.db.createCollection(collectionName, {
          capped: true,
          size: 100 * 1024 * 1024,  // 100 MB
          max: 100000  // Max 100k points de données
        });

        // Index pour requêtes temporelles
        await this.db.collection(collectionName).createIndex({
          timestamp: 1,
          metric: 1
        });

      } catch (error) {
        if (error.code !== 48) throw error;
      }
    }
  }

  async record(type, metric, value, tags = {}) {
    const collectionName = `metrics_${type}`;

    const dataPoint = {
      timestamp: new Date(),
      metric,
      value,
      tags,
      host: require('os').hostname()
    };

    await this.db.collection(collectionName).insertOne(dataPoint);
  }

  async recordBatch(type, dataPoints) {
    const collectionName = `metrics_${type}`;

    const docs = dataPoints.map(dp => ({
      timestamp: new Date(),
      metric: dp.metric,
      value: dp.value,
      tags: dp.tags || {},
      host: require('os').hostname()
    }));

    await this.db.collection(collectionName).insertMany(docs);
  }

  async query(type, metric, options = {}) {
    const {
      since,
      until,
      tags = {},
      aggregation = 'raw'  // 'raw', 'avg', 'sum', 'min', 'max'
    } = options;

    const collectionName = `metrics_${type}`;
    const match = { metric };

    // Filtre temporel
    if (since || until) {
      match.timestamp = {};
      if (since) match.timestamp.$gte = since;
      if (until) match.timestamp.$lte = until;
    }

    // Filtre par tags
    for (const [key, value] of Object.entries(tags)) {
      match[`tags.${key}`] = value;
    }

    if (aggregation === 'raw') {
      return await this.db.collection(collectionName)
        .find(match)
        .sort({ timestamp: 1 })
        .toArray();
    }

    // Agrégation
    const pipeline = [
      { $match: match }
    ];

    switch (aggregation) {
      case 'avg':
        pipeline.push({
          $group: {
            _id: null,
            value: { $avg: '$value' },
            count: { $sum: 1 }
          }
        });
        break;

      case 'sum':
        pipeline.push({
          $group: {
            _id: null,
            value: { $sum: '$value' },
            count: { $sum: 1 }
          }
        });
        break;

      case 'min':
        pipeline.push({
          $group: {
            _id: null,
            value: { $min: '$value' },
            count: { $sum: 1 }
          }
        });
        break;

      case 'max':
        pipeline.push({
          $group: {
            _id: null,
            value: { $max: '$value' },
            count: { $sum: 1 }
          }
        });
        break;
    }

    const result = await this.db.collection(collectionName)
      .aggregate(pipeline)
      .toArray();

    return result[0] || { value: null, count: 0 };
  }

  async getTimeSeries(type, metric, options = {}) {
    const {
      since = new Date(Date.now() - 3600000),  // 1h par défaut
      until = new Date(),
      interval = 60000,  // 1 minute par défaut
      tags = {}
    } = options;

    const collectionName = `metrics_${type}`;
    const match = {
      metric,
      timestamp: { $gte: since, $lte: until }
    };

    // Filtre par tags
    for (const [key, value] of Object.entries(tags)) {
      match[`tags.${key}`] = value;
    }

    // Agrégation par intervalles de temps
    const pipeline = [
      { $match: match },
      {
        $group: {
          _id: {
            $toDate: {
              $subtract: [
                { $toLong: '$timestamp' },
                { $mod: [{ $toLong: '$timestamp' }, interval] }
              ]
            }
          },
          avg: { $avg: '$value' },
          min: { $min: '$value' },
          max: { $max: '$value' },
          count: { $sum: 1 }
        }
      },
      { $sort: { _id: 1 } }
    ];

    return await this.db.collection(collectionName)
      .aggregate(pipeline)
      .toArray();
  }

  async tail(type, metric, callback, tags = {}) {
    const collectionName = `metrics_${type}`;
    const filter = { metric };

    for (const [key, value] of Object.entries(tags)) {
      filter[`tags.${key}`] = value;
    }

    const cursor = this.db.collection(collectionName)
      .find(filter, {
        tailable: true,
        awaitData: true
      })
      .sort({ $natural: 1 });

    cursor.stream().on('data', (dataPoint) => {
      callback(dataPoint);
    });

    return cursor;
  }

  async getDashboard() {
    const dashboard = {};

    for (const type of this.metricsTypes) {
      const collectionName = `metrics_${type}`;

      // Métriques des 5 dernières minutes
      const since = new Date(Date.now() - 5 * 60 * 1000);

      const metrics = await this.db.collection(collectionName)
        .aggregate([
          { $match: { timestamp: { $gte: since } } },
          {
            $group: {
              _id: '$metric',
              count: { $sum: 1 },
              avg: { $avg: '$value' },
              min: { $min: '$value' },
              max: { $max: '$value' },
              latest: { $last: '$value' }
            }
          }
        ])
        .toArray();

      dashboard[type] = metrics;
    }

    return dashboard;
  }
}

// Utilisation
const metrics = new RealtimeMetricsCollector(db);
await metrics.initialize();

// Enregistrement de métriques système
setInterval(async () => {
  const os = require('os');

  await metrics.recordBatch('system', [
    { metric: 'cpu_usage', value: os.loadavg()[0] },
    { metric: 'memory_usage', value: (1 - os.freemem() / os.totalmem()) * 100 },
    { metric: 'uptime', value: os.uptime() }
  ]);
}, 10000);  // Toutes les 10 secondes

// Enregistrement de métriques applicatives
await metrics.record('application', 'requests_per_second', 150, {
  endpoint: '/api/users',
  method: 'GET'
});

await metrics.record('business', 'orders_total', 1250.50, {
  currency: 'USD',
  country: 'US'
});

// Requêtes
const cpuStats = await metrics.query('system', 'cpu_usage', {
  since: new Date(Date.now() - 3600000),  // 1h
  aggregation: 'avg'
});

const timeSeries = await metrics.getTimeSeries('system', 'cpu_usage', {
  since: new Date(Date.now() - 3600000),
  interval: 60000  // Points par minute
});

// Dashboard temps réel
const dashboard = await metrics.getDashboard();
console.log('Dashboard:', dashboard);

// Tail métriques
await metrics.tail('system', 'cpu_usage', (dataPoint) => {
  console.log(`CPU: ${dataPoint.value}%`);
});
```

---

## Tailable Cursors

Une des fonctionnalités les plus puissantes des capped collections est le support des **tailable cursors** (curseurs qui restent ouverts et reçoivent de nouveaux documents).

### Comportement tail -f

```javascript
async function tailLogs(collection) {
  const cursor = collection.find(
    {},
    {
      tailable: true,           // Garder le curseur ouvert
      awaitData: true,          // Bloquer en attendant nouvelles données
      numberOfRetries: Number.MAX_VALUE,  // Retry infini
      tailableRetryInterval: 100  // 100ms entre retries
    }
  ).sort({ $natural: 1 });  // Ordre d'insertion

  // Stream les documents
  const stream = cursor.stream();

  stream.on('data', (doc) => {
    console.log('New log:', doc);
  });

  stream.on('error', (error) => {
    console.error('Tail error:', error);
  });

  return cursor;
}

// Utilisation
const tailCursor = await tailLogs(db.collection('logs'));

// Le curseur reste ouvert et reçoit les nouveaux documents
// Fermer manuellement si nécessaire
// await tailCursor.close();
```

### Cas d'usage avec Change Streams alternative

```javascript
class CappedCollectionMonitor {
  constructor(collection) {
    this.collection = collection;
    this.cursor = null;
  }

  async start(handler) {
    // Démarrer à partir du dernier document actuel
    const lastDoc = await this.collection
      .find()
      .sort({ $natural: -1 })
      .limit(1)
      .toArray();

    const filter = {};
    if (lastDoc.length > 0) {
      filter._id = { $gt: lastDoc[0]._id };
    }

    this.cursor = this.collection.find(filter, {
      tailable: true,
      awaitData: true,
      numberOfRetries: Number.MAX_VALUE
    }).sort({ $natural: 1 });

    const stream = this.cursor.stream();

    stream.on('data', async (doc) => {
      try {
        await handler(doc);
      } catch (error) {
        console.error('Handler error:', error);
      }
    });

    stream.on('error', async (error) => {
      console.error('Stream error:', error);
      // Redémarrer automatiquement
      await this.restart(handler);
    });
  }

  async restart(handler) {
    if (this.cursor) {
      await this.cursor.close().catch(() => {});
    }

    await new Promise(resolve => setTimeout(resolve, 1000));
    await this.start(handler);
  }

  async stop() {
    if (this.cursor) {
      await this.cursor.close();
      this.cursor = null;
    }
  }
}

// Utilisation
const monitor = new CappedCollectionMonitor(db.collection('events'));

await monitor.start(async (event) => {
  console.log('New event:', event);

  // Traitement personnalisé
  if (event.type === 'error') {
    await sendAlert(event);
  }
});

// Arrêter proprement
process.on('SIGTERM', async () => {
  await monitor.stop();
  process.exit(0);
});
```

---

## Comparaison avec alternatives

### Capped Collections vs TTL Index

| Critère | Capped Collection | TTL Index |
|---------|-------------------|-----------|
| **Limite** | Par taille | Par temps |
| **Suppression** | FIFO automatique | Basé sur date |
| **Performance insert** | Excellente | Standard |
| **Ordre garanti** | ✅ Oui | ❌ Non |
| **Tailable cursors** | ✅ Supporté | ❌ Non |
| **Croissance docs** | ❌ Interdit | ✅ Autorisé |
| **Suppression manuelle** | ❌ Interdit | ✅ Autorisé |

```javascript
// TTL Index alternative pour expiration temporelle
await db.collection('sessions').createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // 1h
);

// Les docs sont supprimés automatiquement après 1h
// Mais pas de garantie d'ordre ni tailable cursors
```

### Capped Collections vs Change Streams

| Critère | Capped Collection + Tail | Change Streams |
|---------|-------------------------|----------------|
| **Latence** | Très faible (<10ms) | Faible (~50ms) |
| **Setup** | Simple | Nécessite Replica Set |
| **Filtrage** | Côté client | Côté serveur (pipeline) |
| **Historique** | Limité à taille capped | Limité à Oplog |
| **Overhead** | Minimal | Faible |

### Quand utiliser Capped Collections ?

✅ **Utiliser quand :**
- Logs applicatifs haute fréquence
- Cache éphémère avec rotation automatique
- Files de messages temporaires
- Métriques et monitoring en temps réel
- Besoin de tailable cursors
- Ordre d'insertion critique
- Performance d'écriture maximale requise

❌ **Éviter quand :**
- Besoin de suppression sélective
- Documents doivent pouvoir grandir
- Durée de rétention précise nécessaire (utiliser TTL)
- Besoin de toutes les fonctionnalités MongoDB standard

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Dimensionner correctement la taille
const avgDocSize = 1000; // bytes
const desiredRetention = 100000; // docs
const cappedSize = avgDocSize * desiredRetention * 1.2; // +20% marge

await db.createCollection('logs', {
  capped: true,
  size: cappedSize,
  max: desiredRetention
});

// 2. Utiliser $natural pour tri efficace
const recentLogs = await db.collection('logs')
  .find()
  .sort({ $natural: -1 })  // Pas d'index nécessaire
  .limit(100)
  .toArray();

// 3. Index optionnels pour requêtes fréquentes
await db.collection('logs').createIndex({ level: 1 });

// 4. Monitorer l'utilisation
const stats = await db.collection('logs').stats();
if (stats.size > stats.maxSize * 0.9) {
  console.warn('Capped collection nearly full');
}

// 5. Graceful shutdown des tailable cursors
process.on('SIGTERM', async () => {
  await tailCursor.close();
  process.exit(0);
});
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas essayer de delete
// ❌ ERREUR
await db.collection('logs').deleteOne({ _id: id });

// 2. Ne pas update avec croissance
// ❌ ERREUR si newField est volumineux
await db.collection('logs').updateOne(
  { _id: id },
  { $set: { newField: largeData } }
);

// 3. Ne pas créer trop petite
// ❌ Trop petit - rotation trop fréquente
await db.createCollection('logs', {
  capped: true,
  size: 1024 * 1024  // Seulement 1 MB
});

// 4. Ne pas oublier la marge de sécurité
// ❌ Dimensionnement exact sans marge
const size = avgDocSize * desiredCount;  // Pas de marge

// 5. Ne pas utiliser pour données permanentes
// ❌ Les données seront perdues
await db.collection('users').convertToCapped({ size: 10485760 });
```

---

## Monitoring et maintenance

```javascript
class CappedCollectionMonitor {
  constructor(db) {
    this.db = db;
  }

  async getAllCappedCollections() {
    const collections = await this.db.listCollections().toArray();

    const capped = [];
    for (const col of collections) {
      if (col.options?.capped) {
        capped.push({
          name: col.name,
          maxSize: col.options.size,
          maxDocs: col.options.max
        });
      }
    }

    return capped;
  }

  async getDetailedStats() {
    const cappedCollections = await this.getAllCappedCollections();
    const stats = [];

    for (const col of cappedCollections) {
      const collStats = await this.db.collection(col.name).stats();

      stats.push({
        name: col.name,
        count: collStats.count,
        size: collStats.size,
        maxSize: collStats.maxSize,
        usagePercent: ((collStats.size / collStats.maxSize) * 100).toFixed(2),
        avgObjSize: collStats.avgObjSize,
        storageSize: collStats.storageSize
      });
    }

    return stats;
  }

  async checkHealth() {
    const stats = await this.getDetailedStats();
    const issues = [];

    for (const stat of stats) {
      const usage = parseFloat(stat.usagePercent);

      if (usage > 95) {
        issues.push({
          collection: stat.name,
          severity: 'critical',
          message: `Collection is ${usage}% full`
        });
      } else if (usage > 80) {
        issues.push({
          collection: stat.name,
          severity: 'warning',
          message: `Collection is ${usage}% full`
        });
      }

      if (stat.avgObjSize > 10000) {
        issues.push({
          collection: stat.name,
          severity: 'info',
          message: `Large average document size: ${stat.avgObjSize} bytes`
        });
      }
    }

    return {
      healthy: issues.filter(i => i.severity === 'critical').length === 0,
      issues
    };
  }

  async suggest Optimizations() {
    const stats = await this.getDetailedStats();
    const suggestions = [];

    for (const stat of stats) {
      const usage = parseFloat(stat.usagePercent);

      // Sous-utilisation
      if (usage < 50) {
        suggestions.push({
          collection: stat.name,
          type: 'size_reduction',
          message: `Consider reducing size from ${stat.maxSize} to ${Math.ceil(stat.size * 2)}`,
          savings: stat.maxSize - (stat.size * 2)
        });
      }

      // Sur-utilisation
      if (usage > 90) {
        suggestions.push({
          collection: stat.name,
          type: 'size_increase',
          message: `Consider increasing size from ${stat.maxSize} to ${Math.ceil(stat.maxSize * 1.5)}`
        });
      }
    }

    return suggestions;
  }
}

// Utilisation
const monitor = new CappedCollectionMonitor(db);

// Monitoring périodique
setInterval(async () => {
  const health = await monitor.checkHealth();

  if (!health.healthy) {
    console.error('Capped collections health issues:', health.issues);
  }

  const stats = await monitor.getDetailedStats();
  console.log('Capped collections stats:', stats);
}, 60000);  // Toutes les minutes

// Optimisations suggérées
const suggestions = await monitor.suggestOptimizations();
console.log('Optimization suggestions:', suggestions);
```

---

## Conclusion

Les Capped Collections sont une fonctionnalité spécialisée de MongoDB offrant :
- ✅ **Performances d'écriture exceptionnelles** (~10-20% plus rapides)
- ✅ **Rotation automatique FIFO** (pas de gestion manuelle)
- ✅ **Ordre d'insertion garanti** (important pour logs)
- ✅ **Tailable cursors** (tail -f like behavior)
- ✅ **Prévisibilité** (taille fixe, pas de croissance incontrôlée)

**Points clés à retenir :**
1. Dimensionner correctement (avgDocSize × retention + marge)
2. Comprendre les limitations (pas de delete, pas de croissance)
3. Utiliser pour cas d'usage appropriés (logs, cache, queues)
4. Monitorer l'utilisation et la santé
5. Considérer alternatives selon besoins (TTL, Change Streams)

**Cas d'usage idéaux :**
- Logs applicatifs rotatifs
- Cache éphémère haute performance
- Files de messages temporaires
- Métriques et monitoring temps réel
- Event streams avec tail

---


⏭️ [Time Series Collections](/16-fonctionnalites-avancees/04-time-series-collections.md)
