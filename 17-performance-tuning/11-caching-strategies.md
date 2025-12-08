🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.11 Stratégies de Caching

## Introduction

Le caching est l'un des leviers d'optimisation les plus puissants dans les architectures MongoDB, permettant d'atteindre des améliorations de performance de 10× à 1000× selon les patterns d'accès aux données. MongoDB dispose d'un système de cache multiniveau (WiredTiger, OS, application) qui, lorsqu'optimisé correctement, peut transformer radicalement les performances d'un système.

Cette section explore l'architecture complète du caching dans MongoDB, les stratégies d'optimisation du cache WiredTiger, les patterns de caching applicatif avec Redis/Memcached, et les méthodologies de monitoring et tuning pour maximiser le cache hit ratio en production.

Un cache optimisé peut réduire la latence de 50-95% et augmenter le throughput de 5-50× pour les workloads read-heavy.

## Architecture Multiniveau du Caching

### Vue d'Ensemble

```
┌─────────────────────────────────────────────────────────┐
│                    Application                          │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│          Application-Level Cache (L1)                   │
│   Redis / Memcached / In-Process Cache                  │
│   - Hot data (most accessed)                            │
│   - TTL-based expiration                                │
│   - Hit ratio: 80-95% (si bien dimensionné)             │
│   - Latency: <1ms (in-process) ou 1-3ms (Redis)         │
└────────────────┬────────────────────────────────────────┘
                 │ Cache miss
                 ▼
┌─────────────────────────────────────────────────────────┐
│             MongoDB Driver                              │
│   - Connection pooling                                  │
│   - Query routing                                       │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────────────────────┐
│          WiredTiger Cache (L2)                          │
│   - In-memory cache (configuré: 50-80% RAM)             │
│   - Pages décompressées (4-32KB)                        │
│   - LRU eviction policy                                 │
│   - Hit ratio: 95-99% (si bien dimensionné)             │
│   - Latency: <0.1ms                                     │
└────────────────┬────────────────────────────────────────┘
                 │ Cache miss
                 ▼
┌─────────────────────────────────────────────────────────┐
│            OS Page Cache (L3)                           │
│   - Kernel file cache                                   │
│   - Automatic management                                │
│   - Hit ratio: Variable                                 │
│   - Latency: 0.1-1ms                                    │
└────────────────┬────────────────────────────────────────┘
                 │ Cache miss
                 ▼
┌─────────────────────────────────────────────────────────┐
│               Disk Storage                              │
│   - SSD: 0.1-1ms latency                                │
│   - NVMe: 0.01-0.1ms latency                            │
│   - HDD: 5-20ms latency                                 │
└─────────────────────────────────────────────────────────┘

Latency cascade:
- L1 (App cache): <1ms
- L2 (WiredTiger): +0.1ms
- L3 (OS cache): +1ms
- Disk (SSD): +5-10ms
- Disk (HDD): +10-50ms

Exemple read path:
1. Check L1 (Redis) → Miss → 1ms wasted
2. Check L2 (WiredTiger) → Miss → +0.5ms
3. Check L3 (OS) → Miss → +2ms
4. Disk read (SSD) → Hit → +8ms
Total: ~11ms

vs L1 cache hit: <1ms (11× faster)
```

### Flow de Requête avec Caching

**Read operation** :
```
Query: db.users.findOne({ _id: 123 })

1. Application cache (Redis)
   └─ Key: "user:123"
   └─ Hit → Return (1ms) ✓
   └─ Miss → Continue

2. MongoDB WiredTiger cache
   └─ Check if page in cache
   └─ Hit → Return decompressed doc (0.2ms) ✓
   └─ Miss → Continue

3. OS page cache
   └─ Check if block in kernel cache
   └─ Hit → Return to WiredTiger (1ms) ✓
   └─ Miss → Continue

4. Disk read
   └─ Read compressed block
   └─ Return to WiredTiger (8ms)

5. WiredTiger processing
   └─ Decompress block
   └─ Cache in WiredTiger cache
   └─ Return to application

6. Application processing
   └─ Optionally cache in Redis
   └─ Return to client
```

**Write operation** :
```
Insert: db.users.insertOne({ _id: 123, name: "John" })

1. Write to Primary MongoDB
   └─ WiredTiger cache (dirty page)
   └─ Journal (WAL)
   └─ Acknowledge

2. Cache invalidation
   └─ Delete from Redis: "user:123"
   └─ Or update if write-through

3. Background flush
   └─ Checkpoint: Dirty → Disk
   └─ OS cache updated

4. Replication
   └─ Oplog to Secondaries
   └─ Cache invalidation on secondaries (if app cache)
```

## WiredTiger Cache Optimization

### Cache Internals

**Architecture** :
```javascript
WiredTiger Cache Structure:
{
  size: "20 GB",  // Configured cache size

  // Page types in cache
  pages: {
    clean: "12 GB",    // Read from disk, unchanged
    dirty: "4 GB",     // Modified, not yet flushed
    internal: "1 GB",  // B-tree internal nodes
    leaf: "15 GB",     // B-tree leaf nodes (data)
    index: "3 GB"      // Index pages
  },

  // Eviction management
  eviction: {
    trigger: "16 GB",     // 80% of cache
    target: "16 GB",      // Try to stay under this
    dirty_trigger: "5 GB", // 25% of cache
    dirty_target: "4 GB"   // Try to keep dirty under this
  },

  // Policies
  algorithm: "LRU",  // Least Recently Used
  threads: 4         // Eviction worker threads
}
```

**LRU Algorithm** :
```
Cache organization (simplified):

Most Recently Used (MRU) ← New entries added here
├─ Page 1 (accessed 1s ago)
├─ Page 2 (accessed 5s ago)
├─ Page 3 (accessed 10s ago)
├─ ...
├─ Page N-2 (accessed 300s ago)
├─ Page N-1 (accessed 600s ago)
└─ Page N (accessed 1800s ago) ← Evicted first

When cache full and new page needed:
1. Check if page already in cache (hash lookup)
2. If yes: Move to MRU end, return
3. If no: Evict LRU page(s) to make space
4. Load new page from disk
5. Add to MRU end
```

### Cache Hit Ratio Analysis

**Calcul du hit ratio** :
```javascript
function analyzeCacheHitRatio() {
  const serverStatus = db.serverStatus();
  const cache = serverStatus.wiredTiger.cache;

  // Total requests vs cache misses
  const totalRequests = cache["pages requested from the cache"];
  const cacheMisses = cache["pages read into cache"];
  const cacheHits = totalRequests - cacheMisses;

  const hitRatio = (cacheHits / totalRequests * 100).toFixed(4);

  const analysis = {
    timestamp: new Date(),

    // Core metrics
    totalRequests: totalRequests,
    cacheHits: cacheHits,
    cacheMisses: cacheMisses,
    hitRatio: hitRatio + "%",

    // Cache size
    cacheSizeGB: (cache["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2),
    cacheUsedGB: (cache["bytes currently in the cache"] / 1024 / 1024 / 1024).toFixed(2),
    cacheUsagePercent: ((cache["bytes currently in the cache"] /
                         cache["maximum bytes configured"]) * 100).toFixed(2),

    // Dirty pages
    dirtyBytesGB: (cache["tracked dirty bytes in the cache"] / 1024 / 1024 / 1024).toFixed(2),
    dirtyPercent: ((cache["tracked dirty bytes in the cache"] /
                    cache["maximum bytes configured"]) * 100).toFixed(2),

    // Eviction stats
    evictionStats: {
      appThreadEvictions: cache["pages evicted by application threads"],
      serverEvictions: cache["pages evicted by the page eviction server"],
      modifiedEvicted: cache["modified pages evicted"],
      unmodifiedEvicted: cache["unmodified pages evicted"]
    },

    // Assessment
    assessment: ""
  };

  // Performance assessment
  const ratio = parseFloat(hitRatio);
  if (ratio >= 99) {
    analysis.assessment = "✅ Excellent cache performance";
  } else if (ratio >= 95) {
    analysis.assessment = "✅ Good cache performance";
  } else if (ratio >= 90) {
    analysis.assessment = "⚠️ Acceptable but could improve";
  } else if (ratio >= 80) {
    analysis.assessment = "⚠️ Poor cache performance - investigate";
  } else {
    analysis.assessment = "🔴 Critical: Very poor cache hit ratio";
  }

  // Specific recommendations
  const recommendations = [];

  if (parseFloat(analysis.cacheUsagePercent) > 95) {
    recommendations.push("Cache constantly full - consider increasing cache size");
  }

  if (analysis.evictionStats.appThreadEvictions > 10000) {
    recommendations.push("High application thread evictions - cache pressure");
  }

  if (parseFloat(analysis.dirtyPercent) > 25) {
    recommendations.push("High dirty page ratio - write pressure or slow checkpoints");
  }

  if (ratio < 95) {
    recommendations.push("Working set may exceed cache - analyze data access patterns");
  }

  analysis.recommendations = recommendations.length > 0 ?
    recommendations : ["Configuration appears optimal"];

  return analysis;
}

// Exécution
const cacheAnalysis = analyzeCacheHitRatio();
printjson(cacheAnalysis);

// Exemple output:
// {
//   hitRatio: "98.5234%",
//   assessment: "✅ Excellent cache performance",
//   recommendations: ["Configuration appears optimal"]
// }
```

**Target hit ratios par workload** :

| Workload | Target Hit Ratio | Acceptable | Poor |
|----------|------------------|------------|------|
| **Read-heavy** | >99% | 95-99% | <95% |
| **Mixed** | >97% | 93-97% | <93% |
| **Write-heavy** | >95% | 90-95% | <90% |
| **Analytics** | >90% | 85-90% | <85% |

### Working Set Analysis

**Calcul du working set** :
```javascript
function estimateWorkingSet() {
  const collections = db.getCollectionNames();
  let totalWorkingSet = 0;

  const breakdown = collections.map(collName => {
    const stats = db.getCollection(collName).stats();

    // Estimation heuristique du working set
    // Généralement 20-40% du dataset pour workloads typiques
    const hotDataRatio = 0.3;  // 30% par défaut

    const dataSize = stats.size;
    const indexSize = stats.totalIndexSize;

    // Tous les index doivent être en cache
    // + portion des données chaudes
    const workingSet = (dataSize * hotDataRatio) + indexSize;

    totalWorkingSet += workingSet;

    return {
      collection: collName,
      dataSizeGB: (dataSize / 1024 / 1024 / 1024).toFixed(2),
      indexSizeGB: (indexSize / 1024 / 1024 / 1024).toFixed(2),
      estimatedWSGB: (workingSet / 1024 / 1024 / 1024).toFixed(2),
      hotDataRatio: (hotDataRatio * 100).toFixed(0) + "%"
    };
  });

  // Compare avec cache configuré
  const serverStatus = db.serverStatus();
  const cacheSizeGB = serverStatus.wiredTiger.cache["maximum bytes configured"] / 1024 / 1024 / 1024;
  const wsGB = totalWorkingSet / 1024 / 1024 / 1024;

  const coverage = (cacheSizeGB / wsGB * 100).toFixed(2);

  return {
    collections: breakdown,
    totalWorkingSetGB: wsGB.toFixed(2),
    configuredCacheGB: cacheSizeGB.toFixed(2),
    coveragePercent: coverage + "%",

    recommendation: coverage >= 100 ?
      "✅ Cache can hold entire working set" :
      coverage >= 80 ?
      "⚠️ Cache holds 80%+ of working set - acceptable" :
      "🔴 Cache too small for working set - increase cache size"
  };
}

printjson(estimateWorkingSet());
```

**Mesure empirique du working set** :
```javascript
function measureActualWorkingSet(durationSeconds = 3600) {
  print(`Starting working set measurement for ${durationSeconds}s...`);

  const initialStats = db.serverStatus().wiredTiger.cache;
  const initialTime = Date.now();

  // Attendre la période de mesure
  sleep(durationSeconds * 1000);

  const finalStats = db.serverStatus().wiredTiger.cache;
  const finalTime = Date.now();

  // Calculer les pages uniques accédées
  const pagesRequested = finalStats["pages requested from the cache"] -
                          initialStats["pages requested from the cache"];
  const pagesRead = finalStats["pages read into cache"] -
                     initialStats["pages read into cache"];

  // Estimation du working set basée sur les accès
  const avgPageSize = 16384;  // 16KB typical
  const uniquePagesAccessed = pagesRead + (pagesRequested * 0.1);  // Heuristic
  const workingSetBytes = uniquePagesAccessed * avgPageSize;

  return {
    measurementDuration: ((finalTime - initialTime) / 1000).toFixed(0) + "s",
    pagesRequested: pagesRequested,
    pagesReadFromDisk: pagesRead,
    estimatedWorkingSetGB: (workingSetBytes / 1024 / 1024 / 1024).toFixed(2),

    recommendation: "Run this during peak hours for accurate measurement"
  };
}

// Mesure pendant 1 heure de charge peak
const workingSet = measureActualWorkingSet(3600);
printjson(workingSet);
```

### Cache Warming

**Stratégies de warming** :

```javascript
// Stratégie 1 : Warm critical collections après restart
async function warmCriticalCollections() {
  const criticalCollections = ['users', 'products', 'orders'];

  print("Starting cache warming...");
  const startTime = Date.now();

  for (const collName of criticalCollections) {
    print(`Warming ${collName}...`);

    // Scan complet pour charger en cache
    const count = db.getCollection(collName).find({}, { _id: 1 })
      .hint({ _id: 1 })  // Force index scan
      .batchSize(1000)
      .toArray().length;

    print(`  ${collName}: ${count} documents loaded`);
  }

  const duration = ((Date.now() - startTime) / 1000).toFixed(2);
  print(`Cache warming completed in ${duration}s`);

  // Vérifier cache hit ratio après warming
  const analysis = analyzeCacheHitRatio();
  print(`Cache hit ratio: ${analysis.hitRatio}`);
}

// Stratégie 2 : Warm par query patterns
async function warmByQueryPatterns() {
  // Charger les query patterns les plus fréquents
  const commonQueries = [
    { collection: 'users', filter: { status: 'active' } },
    { collection: 'products', filter: { category: 'electronics' } },
    { collection: 'orders', filter: { status: 'pending' } }
  ];

  for (const query of commonQueries) {
    db.getCollection(query.collection)
      .find(query.filter)
      .limit(10000)
      .toArray();
  }
}

// Stratégie 3 : Warm en background (non-blocking)
function warmCacheBackground() {
  // Script externe exécuté après restart
  const script = `
    // warm-cache.js
    const collections = ['users', 'products', 'orders'];

    collections.forEach(collName => {
      print('Warming ' + collName);
      db.getCollection(collName).find().forEach(() => {});
    });
  `;

  // Exécution : mongo --quiet warm-cache.js &
  // Continue en background pendant que app démarre
}

// Exécution après restart
warmCriticalCollections();
```

**Automatic warming avec replica sets** :
```javascript
// Technique : Rotate members pour warming
// 1. Step down Primary
// 2. Nouveau Primary warm (déjà chaud car était Secondary)
// 3. Ancien Primary devient Secondary
// 4. Warm l'ancien Primary (maintenant Secondary)

function rollingWarm() {
  const status = rs.status();
  const primary = status.members.find(m => m.state === 1);

  print(`Current primary: ${primary.name}`);
  print("Stepping down primary...");

  // Step down for 60 seconds
  rs.stepDown(60);

  // Nouveau Primary prend le relai (déjà warm)
  // Ancien Primary peut être warmed en background

  print("New primary elected, warming old primary...");
  // Connect to old primary and warm
}
```

### Eviction Tuning

**Monitoring eviction pressure** :
```javascript
function analyzeEvictionPressure() {
  const cache = db.serverStatus().wiredTiger.cache;

  const eviction = {
    // Types d'éviction
    applicationThreadEvictions: cache["pages evicted by application threads"],
    evictionServerEvictions: cache["pages evicted by the page eviction server"],

    // Pages évincées par type
    modifiedEvicted: cache["modified pages evicted"],
    unmodifiedEvicted: cache["unmodified pages evicted"],

    // Fails
    evictionFailures: cache["pages selected for eviction unable to be evicted"],

    // Assessment
    pressure: ""
  };

  // Évaluation de la pression
  if (eviction.applicationThreadEvictions > 10000) {
    eviction.pressure = "🔴 CRITICAL: Application threads doing eviction";
    eviction.impact = "Latency spikes, performance degradation";
    eviction.action = "Increase cache size immediately";
  } else if (eviction.applicationThreadEvictions > 1000) {
    eviction.pressure = "⚠️ WARNING: Moderate eviction pressure";
    eviction.impact = "Occasional latency increases";
    eviction.action = "Consider increasing cache size";
  } else {
    eviction.pressure = "✅ OK: Normal eviction by server threads";
    eviction.impact = "No performance impact";
    eviction.action = "No action needed";
  }

  return eviction;
}

printjson(analyzeEvictionPressure());
```

**Tuning eviction threads** (MongoDB 4.0+) :
```javascript
// Augmenter threads d'éviction si cache pressure
db.adminCommand({
  setParameter: 1,
  wiredTigerEvictionThreadsMax: 8  // Défaut: 4
});

// Trade-off:
// + Plus de threads = éviction plus rapide
// - Plus de threads = overhead CPU
// Optimal: 4-8 threads pour la plupart des cas
```

## Application-Level Caching

### Architecture avec Cache Externe

**Redis/Memcached integration** :

```javascript
// Architecture pattern
const redis = require('redis');
const { MongoClient } = require('mongodb');

class CachedMongoClient {
  constructor(mongoUri, redisConfig) {
    this.mongo = new MongoClient(mongoUri);
    this.redis = redis.createClient(redisConfig);
    this.defaultTTL = 300;  // 5 minutes
  }

  async get(collection, query, options = {}) {
    // Générer cache key
    const cacheKey = this.generateCacheKey(collection, query);

    // Check cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return {
        data: JSON.parse(cached),
        source: 'cache',
        latency: '<1ms'
      };
    }

    // Cache miss : Query MongoDB
    const startTime = Date.now();
    const db = this.mongo.db();
    const result = await db.collection(collection).findOne(query);
    const latency = Date.now() - startTime;

    // Store in cache
    if (result) {
      const ttl = options.ttl || this.defaultTTL;
      await this.redis.setex(
        cacheKey,
        ttl,
        JSON.stringify(result)
      );
    }

    return {
      data: result,
      source: 'mongodb',
      latency: latency + 'ms'
    };
  }

  generateCacheKey(collection, query) {
    // Hash stable du query
    const queryString = JSON.stringify(query, Object.keys(query).sort());
    return `mongo:${collection}:${this.hash(queryString)}`;
  }

  hash(str) {
    // Simple hash pour demo (utiliser crypto.createHash en prod)
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) - hash) + str.charCodeAt(i);
      hash |= 0;
    }
    return hash.toString(36);
  }

  async invalidate(collection, query) {
    const cacheKey = this.generateCacheKey(collection, query);
    await this.redis.del(cacheKey);
  }
}

// Usage
const client = new CachedMongoClient(mongoUri, { host: 'localhost', port: 6379 });

// Read avec cache
const user = await client.get('users', { _id: 123 });
console.log(`Source: ${user.source}, Latency: ${user.latency}`);
// Premier call : Source: mongodb, Latency: 15ms
// Calls suivants : Source: cache, Latency: <1ms

// Write avec invalidation
await mongoClient.db().collection('users').updateOne(
  { _id: 123 },
  { $set: { name: 'John Updated' } }
);
await client.invalidate('users', { _id: 123 });
```

### Cache Patterns

#### Pattern 1 : Cache-Aside (Lazy Loading)

```javascript
async function cacheAsidePattern(userId) {
  const cacheKey = `user:${userId}`;

  // 1. Check cache
  let user = await redis.get(cacheKey);

  if (user) {
    // Cache hit
    return JSON.parse(user);
  }

  // 2. Cache miss : Load from MongoDB
  user = await db.users.findOne({ _id: userId });

  // 3. Store in cache
  if (user) {
    await redis.setex(cacheKey, 300, JSON.stringify(user));
  }

  return user;
}

// Caractéristiques:
// + Simple à implémenter
// + Cache seulement ce qui est demandé
// - First request toujours lent (cold cache)
// - Cache miss penalty
```

#### Pattern 2 : Read-Through Cache

```javascript
class ReadThroughCache {
  constructor(mongo, redis) {
    this.mongo = mongo;
    this.redis = redis;
  }

  async get(key, loaderFn) {
    // Cache abstraction : Loader appelé automatiquement
    let value = await this.redis.get(key);

    if (!value) {
      // Load via provided function
      value = await loaderFn();

      if (value) {
        await this.redis.setex(key, 300, JSON.stringify(value));
      }
    } else {
      value = JSON.parse(value);
    }

    return value;
  }
}

// Usage
const cache = new ReadThroughCache(mongoClient, redisClient);

const user = await cache.get(
  `user:${userId}`,
  () => db.users.findOne({ _id: userId })
);

// Caractéristiques:
// + Abstraction propre
// + Loader logic encapsulée
// - Toujours penalty sur cache miss
```

#### Pattern 3 : Write-Through Cache

```javascript
async function writeThroughPattern(userId, updates) {
  const cacheKey = `user:${userId}`;

  // 1. Write to MongoDB
  const result = await db.users.findOneAndUpdate(
    { _id: userId },
    { $set: updates },
    { returnDocument: 'after' }
  );

  // 2. Immediately update cache
  if (result.value) {
    await redis.setex(
      cacheKey,
      300,
      JSON.stringify(result.value)
    );
  }

  return result.value;
}

// Caractéristiques:
// + Cache toujours à jour après write
// + Pas d'invalidation nécessaire
// - Write latency augmentée (2 ops)
// - Complexité accrue
```

#### Pattern 4 : Write-Behind (Write-Back) Cache

```javascript
class WriteBehindCache {
  constructor(mongo, redis) {
    this.mongo = mongo;
    this.redis = redis;
    this.writeQueue = [];
    this.flushInterval = 5000;  // 5 seconds

    // Background flush
    this.startFlusher();
  }

  async write(collection, doc) {
    const cacheKey = `${collection}:${doc._id}`;

    // 1. Write to cache immediately
    await this.redis.setex(
      cacheKey,
      300,
      JSON.stringify(doc)
    );

    // 2. Queue write to MongoDB (async)
    this.writeQueue.push({ collection, doc });

    // 3. Return immediately (no wait)
    return doc;
  }

  startFlusher() {
    setInterval(async () => {
      if (this.writeQueue.length > 0) {
        const batch = this.writeQueue.splice(0, 100);

        // Batch write to MongoDB
        for (const item of batch) {
          try {
            await this.mongo.db()
              .collection(item.collection)
              .replaceOne(
                { _id: item.doc._id },
                item.doc,
                { upsert: true }
              );
          } catch (error) {
            console.error('Write-behind error:', error);
            // Re-queue ou dead letter queue
          }
        }
      }
    }, this.flushInterval);
  }
}

// Caractéristiques:
// + Write latency minimale
// + Throughput élevé
// - Risque de perte de données (si crash avant flush)
// - Complexité élevée
// Usage: Seulement si perte de données acceptable
```

### Cache Invalidation Strategies

**Problématique** : "There are only two hard things in Computer Science: cache invalidation and naming things." - Phil Karlton

#### Stratégie 1 : TTL-Based Expiration

```javascript
// Simple mais peut servir stale data
async function ttlInvalidation(userId) {
  const cacheKey = `user:${userId}`;
  const ttl = 300;  // 5 minutes

  // Cache avec TTL
  await redis.setex(cacheKey, ttl, JSON.stringify(user));

  // Après 5 minutes : Automatiquement expiré
  // Next read : Cache miss → Reload from MongoDB
}

// Trade-off:
// + Simple, automatique
// - Données potentiellement stale pendant TTL
// - Cache miss périodique même sans changement
```

#### Stratégie 2 : Explicit Invalidation

```javascript
async function explicitInvalidation(userId, updates) {
  // 1. Update MongoDB
  await db.users.updateOne(
    { _id: userId },
    { $set: updates }
  );

  // 2. Immediately invalidate cache
  await redis.del(`user:${userId}`);

  // Next read : Cache miss → Fresh data
}

// Trade-off:
// + Données toujours fresh après write
// + Pas de stale data
// - Nécessite invalidation à chaque write
// - Cache miss après chaque write
```

#### Stratégie 3 : Tag-Based Invalidation

```javascript
class TaggedCache {
  async set(key, value, tags = []) {
    // Store value
    await redis.setex(key, 300, JSON.stringify(value));

    // Store tags
    for (const tag of tags) {
      await redis.sadd(`tag:${tag}`, key);
    }
  }

  async invalidateByTag(tag) {
    // Get all keys with this tag
    const keys = await redis.smembers(`tag:${tag}`);

    // Delete all keys
    if (keys.length > 0) {
      await redis.del(...keys);
    }

    // Delete tag set
    await redis.del(`tag:${tag}`);
  }
}

// Usage
const cache = new TaggedCache();

// Cache user avec tags
await cache.set(
  `user:${userId}`,
  user,
  ['users', `company:${user.companyId}`]
);

// Invalidate tous les users d'une company
await cache.invalidateByTag(`company:${companyId}`);

// Trade-off:
// + Invalidation en masse facile
// + Flexible
// - Overhead de gestion des tags
// - Complexité accrue
```

#### Stratégie 4 : Version-Based Invalidation

```javascript
class VersionedCache {
  async get(key) {
    // Check version
    const currentVersion = await redis.get(`${key}:version`);
    const cachedData = await redis.get(key);

    if (cachedData) {
      const cached = JSON.parse(cachedData);

      if (cached.version === currentVersion) {
        return cached.data;
      }
    }

    // Version mismatch ou cache miss
    return null;
  }

  async set(key, data, version) {
    await redis.setex(
      key,
      300,
      JSON.stringify({ data, version })
    );
    await redis.set(`${key}:version`, version);
  }

  async invalidate(key) {
    // Increment version (invalide automatiquement old cache)
    await redis.incr(`${key}:version`);
  }
}

// Trade-off:
// + Pas de delete nécessaire
// + Cache peut rester (garbage collected later)
// - Overhead de gestion version
```

### Hot Key Problem

**Problématique** : Une clé très fréquemment accédée peut saturer le cache.

```javascript
// Problème : Hot key sur single Redis instance
// 100,000 req/sec sur "trending:products"
// → Single Redis instance bottleneck

// Solution 1 : Local in-process cache
class TwoLevelCache {
  constructor(redisClient) {
    this.redis = redisClient;
    this.localCache = new Map();
    this.localTTL = 10000;  // 10 seconds
  }

  async get(key) {
    // L1 : Check local cache
    const local = this.localCache.get(key);
    if (local && Date.now() < local.expiry) {
      return local.value;
    }

    // L2 : Check Redis
    const value = await this.redis.get(key);
    if (value) {
      // Store in local cache
      this.localCache.set(key, {
        value: value,
        expiry: Date.now() + this.localTTL
      });
    }

    return value;
  }
}

// Solution 2 : Cache key sharding
function getShardedKey(baseKey, numShards = 10) {
  const shard = Math.floor(Math.random() * numShards);
  return `${baseKey}:shard:${shard}`;
}

// Read avec sharding
const key = getShardedKey('trending:products', 10);
const data = await redis.get(key);

// Write à tous les shards
for (let i = 0; i < 10; i++) {
  await redis.setex(`trending:products:shard:${i}`, 60, data);
}

// Trade-off:
// + Distribution de charge
// - Inconsistency possible entre shards
// - Overhead writes
```

### Cache Stampede Prevention

**Problématique** : Quand un cache populaire expire, beaucoup de requests simultanées vont vers MongoDB.

```javascript
// Solution : Cache stampede lock
class StampedeProtectedCache {
  constructor(redis, mongo) {
    this.redis = redis;
    this.mongo = mongo;
    this.locks = new Map();
  }

  async get(key, loaderFn, ttl = 300) {
    // Check cache
    let value = await this.redis.get(key);
    if (value) {
      return JSON.parse(value);
    }

    // Cache miss : Acquire lock
    const lockKey = `lock:${key}`;
    const lockId = Math.random().toString(36);

    const acquired = await this.redis.set(
      lockKey,
      lockId,
      'NX',  // Only if not exists
      'EX',  // Expiration
      10     // 10 seconds
    );

    if (acquired) {
      // This thread loads data
      try {
        value = await loaderFn();

        if (value) {
          await this.redis.setex(key, ttl, JSON.stringify(value));
        }

        return value;
      } finally {
        // Release lock
        await this.redis.del(lockKey);
      }
    } else {
      // Another thread is loading : Wait and retry
      await new Promise(resolve => setTimeout(resolve, 100));
      return this.get(key, loaderFn, ttl);
    }
  }
}

// Usage
const cache = new StampedeProtectedCache(redisClient, mongoClient);

const products = await cache.get(
  'trending:products',
  () => db.products.find({ trending: true }).toArray(),
  300
);

// Résultat:
// - Cache miss : 1 thread charge, autres attendent
// - Pas de stampede vers MongoDB
// - Cache populated une seule fois
```

### Cache Sizing

**Formule de dimensionnement** :
```javascript
function calculateCacheSize(workload) {
  // Inputs
  const {
    avgDocumentSizeKB = 5,
    hotSetPercent = 20,  // 20% des données sont chaudes
    totalDocuments = 10000000,
    desiredHitRatio = 0.95
  } = workload;

  // Calculs
  const hotDocuments = totalDocuments * (hotSetPercent / 100);
  const hotSetSizeGB = (hotDocuments * avgDocumentSizeKB) / 1024 / 1024;

  // Cache size pour atteindre hit ratio désiré
  const requiredCacheSizeGB = hotSetSizeGB * (desiredHitRatio / 0.8);

  return {
    totalDocuments: totalDocuments,
    hotSetPercent: hotSetPercent + "%",
    hotDocuments: hotDocuments,
    hotSetSizeGB: hotSetSizeGB.toFixed(2),
    desiredHitRatio: (desiredHitRatio * 100).toFixed(0) + "%",
    recommendedCacheSizeGB: requiredCacheSizeGB.toFixed(2),

    notes: [
      "This is for application-level cache (Redis/Memcached)",
      "Add 20-30% buffer for overhead",
      "Monitor actual hit ratio and adjust"
    ]
  };
}

// Exemple
const sizing = calculateCacheSize({
  avgDocumentSizeKB: 5,
  hotSetPercent: 20,
  totalDocuments: 10000000,
  desiredHitRatio: 0.95
});

printjson(sizing);

// Output:
// {
//   hotSetSizeGB: "9.54",
//   recommendedCacheSizeGB: "11.31",
//   notes: [...]
// }
```

## Monitoring et Métriques

### Dashboard de Caching

```javascript
function comprehensiveCacheDashboard() {
  const serverStatus = db.serverStatus();
  const wtCache = serverStatus.wiredTiger.cache;

  const dashboard = {
    timestamp: new Date(),

    // WiredTiger cache
    wiredTigerCache: {
      sizeGB: (wtCache["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2),
      usedGB: (wtCache["bytes currently in the cache"] / 1024 / 1024 / 1024).toFixed(2),
      usagePercent: ((wtCache["bytes currently in the cache"] /
                      wtCache["maximum bytes configured"]) * 100).toFixed(2),

      hitRatio: (() => {
        const total = wtCache["pages requested from the cache"];
        const misses = wtCache["pages read into cache"];
        return ((total - misses) / total * 100).toFixed(4) + "%";
      })(),

      dirtyGB: (wtCache["tracked dirty bytes in the cache"] / 1024 / 1024 / 1024).toFixed(2),
      dirtyPercent: ((wtCache["tracked dirty bytes in the cache"] /
                      wtCache["maximum bytes configured"]) * 100).toFixed(2)
    },

    // Eviction metrics
    eviction: {
      appThreadEvictions: wtCache["pages evicted by application threads"],
      serverEvictions: wtCache["pages evicted by the page eviction server"],
      pressure: wtCache["pages evicted by application threads"] > 10000 ?
                "🔴 HIGH" : "✅ Normal"
    },

    // I/O metrics
    io: {
      pagesReadMillion: (wtCache["pages read into cache"] / 1000000).toFixed(2),
      pagesWrittenMillion: (wtCache["pages written from cache"] / 1000000).toFixed(2),
      bytesReadGB: (serverStatus.wiredTiger.block_manager["bytes read"] / 1024 / 1024 / 1024).toFixed(2),
      bytesWrittenGB: (serverStatus.wiredTiger.block_manager["bytes written"] / 1024 / 1024 / 1024).toFixed(2)
    },

    // Performance indicators
    performance: {
      ticketsAvailable: {
        read: wtCache["pages requested from the cache"] > 0 ? "active" : "idle",
        write: wtCache["pages written from cache"] > 0 ? "active" : "idle"
      }
    }
  };

  // Overall assessment
  const hitRatio = parseFloat(dashboard.wiredTigerCache.hitRatio);
  const usage = parseFloat(dashboard.wiredTigerCache.usagePercent);

  const issues = [];

  if (hitRatio < 95) {
    issues.push("⚠️ Cache hit ratio below 95%");
  }

  if (usage > 95) {
    issues.push("⚠️ Cache usage above 95%");
  }

  if (dashboard.eviction.pressure === "🔴 HIGH") {
    issues.push("🔴 High eviction pressure detected");
  }

  dashboard.assessment = issues.length === 0 ?
    "✅ Caching performance healthy" : issues;

  return dashboard;
}

// Monitoring périodique
setInterval(() => {
  const dashboard = comprehensiveCacheDashboard();
  // Export to monitoring system
  console.log(JSON.stringify(dashboard));
}, 60000);
```

### Alerting Rules

```yaml
# WiredTiger cache hit ratio
- alert: LowCacheHitRatio
  expr: |
    mongodb_wiredtiger_cache_hit_ratio < 0.95
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "Low WiredTiger cache hit ratio"
    description: "Hit ratio {{ $value }} below 95%"

# Cache pressure
- alert: HighEvictionPressure
  expr: |
    rate(mongodb_wiredtiger_cache_pages_evicted_by_app_threads[5m]) > 1000
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "High cache eviction pressure"
    description: "Application threads evicting pages"

# Cache full
- alert: CacheNearlyFull
  expr: |
    mongodb_wiredtiger_cache_usage_percent > 95
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "WiredTiger cache nearly full"
    description: "Cache at {{ $value }}% capacity"

# Application cache (Redis)
- alert: RedisHighMemory
  expr: |
    redis_memory_used_bytes / redis_memory_max_bytes > 0.9
  labels:
    severity: warning
  annotations:
    summary: "Redis memory usage high"
    description: "Redis using {{ $value }}% of max memory"

- alert: RedisEvictions
  expr: |
    rate(redis_evicted_keys_total[5m]) > 100
  labels:
    severity: warning
  annotations:
    summary: "Redis evicting keys"
    description: "{{ $value }} evictions/sec - consider increasing memory"
```

## Best Practices

### Checklist de Configuration

```
☐ WiredTiger Cache
  ☐ Cache size = Working Set × 1.3 minimum
  ☐ Target hit ratio: >95% (read-heavy), >90% (mixed)
  ☐ Monitor eviction pressure (app thread evictions)
  ☐ Cache warming après restart

☐ Application Cache (Redis/Memcached)
  ☐ Dimensionner pour 20-30% du hot set
  ☐ TTL approprié par type de données (60-600s)
  ☐ Invalidation strategy définie
  ☐ Stampede protection implémentée
  ☐ Monitoring hit ratio (target >80%)

☐ Cache Invalidation
  ☐ Strategy choisie (TTL, explicit, versioned)
  ☐ Invalidation cohérente avec writes
  ☐ Tag-based pour invalidation en masse si nécessaire

☐ Monitoring
  ☐ WiredTiger hit ratio alerting
  ☐ Application cache hit ratio tracking
  ☐ Eviction pressure monitoring
  ☐ Cache size vs working set validation

☐ Testing
  ☐ Cold cache scenario testé
  ☐ Cache stampede simulation
  ☐ Invalidation correctness validée
  ☐ Performance avec/sans cache comparé
```

### Configurations par Workload

**Read-heavy (>80% reads)** :
```yaml
WiredTiger:
  cacheSizeGB: 40  # Generous pour hot set
  target_hit_ratio: 99%

Application Cache:
  provider: Redis
  size: 16 GB  # 30% du hot set
  ttl: 300  # 5 minutes
  strategy: cache-aside
  stampede_protection: true
```

**Write-heavy (>50% writes)** :
```yaml
WiredTiger:
  cacheSizeGB: 30  # Focus sur working set
  target_hit_ratio: 95%

Application Cache:
  provider: Redis
  size: 8 GB  # Minimal pour hot queries
  ttl: 60  # 1 minute (courte pour freshness)
  strategy: explicit-invalidation
  # Avoid write-through (latency)
```

**Analytics** :
```yaml
WiredTiger:
  cacheSizeGB: 60  # Maximize pour scans
  target_hit_ratio: 90%  # Acceptable car scans

Application Cache:
  provider: Redis
  size: 32 GB  # Large pour résultats d'agrégations
  ttl: 3600  # 1 heure (rapports changent peu)
  strategy: ttl-based
```

### Erreurs Courantes

```javascript
// ❌ ERREUR 1 : Cache tous les documents
await redis.setex(
  `all:users`,
  300,
  JSON.stringify(await db.users.find().toArray())
);
// Problème : Large payload, lent, gaspille cache

// ✅ CORRECT : Cache seulement hot items
// Cache par ID, chargé on-demand

// ❌ ERREUR 2 : TTL trop long sur données fréquemment modifiées
await redis.setex(`user:${id}`, 3600, JSON.stringify(user));
// update user in MongoDB (cache stale pendant 1h!)

// ✅ CORRECT : TTL court ou invalidation explicite
await redis.setex(`user:${id}`, 60, JSON.stringify(user));

// ❌ ERREUR 3 : Pas de stampede protection
// 1000 concurrent requests sur expired cache key
// → 1000 queries simultanées vers MongoDB

// ✅ CORRECT : Lock-based loading

// ❌ ERREUR 4 : Cache sans monitoring
// Pas de visibilité sur hit ratio, taille, evictions

// ✅ CORRECT : Dashboard et alerting complets
```

## Conclusion

Le caching est un levier d'optimisation critique pour MongoDB :

**Gains mesurables** :
- Latence : 10-100× réduction (1-10ms → <1ms)
- Throughput : 5-50× augmentation
- Load MongoDB : 50-95% réduction
- Coûts : 30-70% économies (moins de compute nécessaire)

**Architecture optimale** :
1. **WiredTiger cache** : 50-80% RAM, hit ratio >95%
2. **Application cache** : Redis/Memcached pour hot data, hit ratio >80%
3. **Stratégie invalidation** : Explicit pour consistency critique, TTL sinon

**Recommandations clés** :
- WiredTiger cache = Working Set × 1.3 minimum
- Application cache = 20-30% du hot set
- TTL : 60-300s selon freshness requirements
- Monitor hit ratios continuellement
- Stampede protection sur hot keys
- Cache warming après restart

**Patterns recommandés** :
- **Cache-Aside** : Défaut, simple et efficace
- **Read-Through** : Si abstraction propre nécessaire
- **Write-Through** : Si consistency > latency
- **Write-Behind** : Seulement si perte données acceptable

**Quand cacher** :
- ✅ Read-heavy (>60% reads)
- ✅ Hot data identifiable (<30% du dataset)
- ✅ Eventual consistency acceptable
- ✅ Latence critique (<10ms P99)

**Quand éviter** :
- ❌ Write-heavy (>60% writes)
- ❌ Strong consistency absolue requise
- ❌ Données uniformément accédées (pas de hot set)
- ❌ Maintenance overhead > bénéfice

Le caching n'est pas gratuit (complexité, consistency trade-offs) mais correctement implémenté, c'est l'une des optimisations les plus rentables pour les systèmes MongoDB à forte charge.

---

**Points clés à retenir :**
- WiredTiger cache hit ratio >95% essentiel (working set doit tenir)
- Application cache (Redis) pour <1ms latency sur hot data
- Cache-Aside pattern recommandé par défaut
- Invalidation explicite pour consistency, TTL pour simplicité
- Stampede protection obligatoire sur hot keys
- Monitoring hit ratio continu pour ajustement
- Cache warming après restart pour éviter cold start
- Trade-off complexity vs performance toujours évaluer

⏭️ [Benchmarking et tests de charge](/17-performance-tuning/12-benchmarking-tests-charge.md)
