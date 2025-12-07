🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.6 Limites et considérations de performance

## Introduction : Comprendre les contraintes

Les transactions multi-documents dans MongoDB, bien que puissantes, opèrent sous des **contraintes strictes** qui peuvent avoir un impact significatif sur les performances et la scalabilité d'une application. Ces limites ne sont pas des défauts arbitraires, mais plutôt des **garanties nécessaires** pour maintenir les propriétés ACID dans un système distribué.

Cette section explore en profondeur ces limites, leurs implications pratiques, et comment les gérer efficacement dans un environnement de production.

### Taxonomie des limites

Les limites des transactions MongoDB peuvent être classées en plusieurs catégories :

```
Limites des transactions MongoDB :

1. LIMITES HARD (Non contournables)
   - Taille maximale : 16 MB
   - Durée maximale : 60 secondes (par défaut)
   - Nombre d'opérations : ~1000 recommandé
   - Locks concurrents : WiredTiger limites

2. LIMITES SOFT (Performance)
   - Throughput réduit
   - Latence augmentée
   - Utilisation mémoire
   - Cache pressure

3. LIMITES OPÉRATIONNELLES
   - Retry automatique : 3 tentatives
   - Snapshot isolation overhead
   - Journal size impact
   - Replication lag amplification
```

## Limites techniques strictes (Hard Limits)

### Limite 1 : Taille maximale de transaction (16 MB)

MongoDB impose une limite stricte de **16 MB** sur la taille totale d'une transaction, mesurée par la somme de toutes les opérations BSON.

**Calcul de la taille** :

```javascript
// Exemple de calcul de taille de transaction

function estimateTransactionSize(operations) {
  let totalSize = 0;

  for (const op of operations) {
    // Taille de l'opération en BSON
    const bsonSize = Object.bsonsize(op.document);

    // Overhead de l'opération (commande, index keys, etc.)
    const overhead = 200; // bytes approximatifs

    totalSize += bsonSize + overhead;
  }

  return {
    bytes: totalSize,
    megabytes: (totalSize / 1024 / 1024).toFixed(2),
    percentOfLimit: ((totalSize / (16 * 1024 * 1024)) * 100).toFixed(2)
  };
}

// Exemple concret
const operations = [
  {
    type: 'insert',
    collection: 'orders',
    document: {
      _id: new ObjectId(),
      items: Array(100).fill({
        productId: new ObjectId(),
        name: 'Product',
        price: 99.99,
        quantity: 1
      }),
      customer: { /* données client */ }
    }
  },
  // ... plus d'opérations
];

const size = estimateTransactionSize(operations);
console.log('Taille estimée de la transaction:');
console.log(`  ${size.bytes} bytes`);
console.log(`  ${size.megabytes} MB`);
console.log(`  ${size.percentOfLimit}% de la limite`);

// Sortie exemple :
// Taille estimée de la transaction:
//   2,456,789 bytes
//   2.34 MB
//   14.65% de la limite
```

**Scénario problématique** :

```javascript
// ❌ DÉPASSEMENT DE LIMITE : Mise à jour massive

async function massiveUpdate() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // Tenter de mettre à jour 10,000 documents
      for (let i = 0; i < 10000; i++) {
        await db.products.updateOne(
          { _id: i },
          {
            $set: {
              description: 'Un texte très long...'.repeat(100),
              metadata: { /* beaucoup de données */ }
            }
          },
          { session }
        );
      }
      // PROBLÈME : Taille totale > 16 MB
      // → TransactionTooLarge error
    });
  } catch (error) {
    if (error.codeName === 'TransactionTooLarge') {
      console.error('❌ Transaction dépasse 16 MB');
    }
  } finally {
    await session.endSession();
  }
}
```

**Solution : Batching sans transaction** :

```javascript
// ✅ SOLUTION : Traitement par lots

async function batchedUpdate(updateData, batchSize = 100) {
  const batches = [];

  // Diviser en lots
  for (let i = 0; i < updateData.length; i += batchSize) {
    batches.push(updateData.slice(i, i + batchSize));
  }

  const results = {
    processed: 0,
    failed: 0,
    errors: []
  };

  // Traiter chaque lot SANS transaction
  for (const batch of batches) {
    try {
      const bulkOps = batch.map(item => ({
        updateOne: {
          filter: { _id: item._id },
          update: { $set: item.data },
          upsert: false
        }
      }));

      const result = await db.products.bulkWrite(bulkOps, {
        ordered: false  // Continuer même si certains échouent
      });

      results.processed += result.modifiedCount;

    } catch (error) {
      results.failed += batch.length;
      results.errors.push(error);
    }
  }

  return results;
}

// Alternative : Si atomicité nécessaire pour chaque lot
async function batchedUpdateWithTransactions(updateData, batchSize = 50) {
  for (let i = 0; i < updateData.length; i += batchSize) {
    const batch = updateData.slice(i, i + batchSize);
    const session = client.startSession();

    try {
      await session.withTransaction(async () => {
        // Transaction sur un lot (< 16 MB)
        for (const item of batch) {
          await db.products.updateOne(
            { _id: item._id },
            { $set: item.data },
            { session }
          );
        }
      });
    } finally {
      await session.endSession();
    }
  }
}
```

### Limite 2 : Durée maximale (Timeout)

Par défaut, une transaction ne peut pas durer plus de **60 secondes** (`transactionLifetimeLimitSeconds`).

**Configuration** :

```javascript
// Modifier la limite globale (sur mongod/mongos)
db.adminCommand({
  setParameter: 1,
  transactionLifetimeLimitSeconds: 120  // 2 minutes
});

// Maximum absolu : 86400 secondes (24 heures)
// Mais FORTEMENT DÉCONSEILLÉ - les transactions longues posent problème
```

**Problèmes des transactions longues** :

```javascript
// ❌ PROBLÈME : Transaction longue durée

async function longRunningTransaction() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {

      // Opération 1 : Lecture complexe (15s)
      const data = await db.large_collection.aggregate([
        { $match: { /* filtre complexe */ } },
        { $group: { /* agrégation lourde */ } },
        // ... pipeline complexe
      ], { session }).toArray();

      // Opération 2 : Traitement applicatif (20s)
      const processed = expensiveProcessing(data);

      // Opération 3 : Écritures multiples (10s)
      for (const item of processed) {
        await db.results.insertOne(item, { session });
      }

      // PROBLÈME 1 : Durée totale ~45s proche de la limite
      // PROBLÈME 2 : Verrous maintenus pendant 45s
      // PROBLÈME 3 : Autres transactions bloquées
      // PROBLÈME 4 : Risque de timeout

    }, {
      maxCommitTimeMS: 60000  // Timeout commit : 1 minute
    });

  } catch (error) {
    if (error.codeName === 'TransactionExceededLifetimeLimitSeconds') {
      console.error('❌ Transaction timeout après 60s');
    }
  } finally {
    await session.endSession();
  }
}
```

**Solution : Minimiser le temps transactionnel** :

```javascript
// ✅ SOLUTION : Séparer lecture/traitement/écriture

async function optimizedWorkflow() {

  // PHASE 1 : Lecture SANS transaction (lecture cohérente)
  const data = await db.large_collection.aggregate([
    { $match: { /* filtre */ } },
    { $group: { /* agrégation */ } }
  ], {
    readConcern: { level: 'majority' }  // Cohérent mais sans transaction
  }).toArray();

  // PHASE 2 : Traitement HORS transaction
  const processed = expensiveProcessing(data);

  // PHASE 3 : Écriture RAPIDE avec transaction
  const session = client.startSession();
  try {
    await session.withTransaction(async () => {
      // Transaction courte : seulement les écritures (~2s)
      await db.results.insertMany(processed, { session });
    });
  } finally {
    await session.endSession();
  }

  // Durée totale identique, mais transaction = 2s au lieu de 45s
  // → 95% moins de temps avec verrous
}
```

### Limite 3 : Nombre d'opérations recommandé (~1000)

Bien qu'il n'y ait pas de limite stricte sur le nombre d'opérations, MongoDB recommande de **limiter à ~1000 opérations** par transaction pour maintenir les performances.

**Impact du nombre d'opérations** :

```javascript
// Benchmark : Impact du nombre d'opérations

async function benchmarkTransactionSize(numOperations) {
  const session = client.startSession();
  const startTime = Date.now();

  try {
    await session.withTransaction(async () => {
      for (let i = 0; i < numOperations; i++) {
        await db.test.insertOne(
          { _id: i, data: 'test', timestamp: new Date() },
          { session }
        );
      }
    });

    const duration = Date.now() - startTime;
    return { numOperations, duration, avgPerOp: duration / numOperations };

  } finally {
    await session.endSession();
  }
}

// Résultats typiques :
const benchmarks = [
  { ops: 10,    duration: 45,   avgPerOp: 4.5   },  // Optimal
  { ops: 100,   duration: 180,  avgPerOp: 1.8   },  // Bon
  { ops: 500,   duration: 950,  avgPerOp: 1.9   },  // Acceptable
  { ops: 1000,  duration: 2100, avgPerOp: 2.1   },  // Limite recommandée
  { ops: 5000,  duration: 15000, avgPerOp: 3.0  },  // Dégradation
  { ops: 10000, duration: 45000, avgPerOp: 4.5  },  // Problématique
];

// Observation : Au-delà de 1000 ops, latence par opération augmente
```

**Stratégie de découpage** :

```javascript
// ✅ Découpage intelligent en micro-transactions

async function processLargeDataset(items) {
  const BATCH_SIZE = 500;  // Sous la limite recommandée
  const results = [];

  for (let i = 0; i < items.length; i += BATCH_SIZE) {
    const batch = items.slice(i, i + BATCH_SIZE);
    const session = client.startSession();

    try {
      const batchResult = await session.withTransaction(async () => {
        const insertResult = await db.collection.insertMany(
          batch,
          { session, ordered: false }
        );
        return insertResult;
      });

      results.push(batchResult);

    } catch (error) {
      console.error(`Échec du batch ${i / BATCH_SIZE}:`, error);
      // Continuer avec les autres batches

    } finally {
      await session.endSession();
    }

    // Pause entre batches pour éviter saturation
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  return results;
}
```

## Impact sur les performances

### Latence : Le coût de l'atomicité

Les transactions introduisent une **latence supplémentaire significative** par rapport aux opérations non transactionnelles.

**Décomposition de la latence** :

```javascript
// Analyse détaillée de la latence transactionnelle

class TransactionLatencyAnalyzer {
  constructor() {
    this.measurements = [];
  }

  async measureTransaction(operation) {
    const timings = {
      start: Date.now(),
      sessionCreation: 0,
      transactionStart: 0,
      operations: [],
      commit: 0,
      total: 0
    };

    // 1. Création de session
    const t1 = Date.now();
    const session = client.startSession();
    timings.sessionCreation = Date.now() - t1;

    try {
      // 2. Démarrage transaction
      const t2 = Date.now();
      session.startTransaction({
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority' }
      });
      timings.transactionStart = Date.now() - t2;

      // 3. Exécution des opérations
      const t3 = Date.now();
      await operation(session);
      timings.operations.push({
        name: 'all_operations',
        duration: Date.now() - t3
      });

      // 4. Commit
      const t4 = Date.now();
      await session.commitTransaction();
      timings.commit = Date.now() - t4;

    } catch (error) {
      await session.abortTransaction();
      throw error;
    } finally {
      await session.endSession();
      timings.total = Date.now() - timings.start;
    }

    this.measurements.push(timings);
    return timings;
  }

  getStats() {
    const stats = {
      count: this.measurements.length,
      avgSessionCreation: 0,
      avgTransactionStart: 0,
      avgOperations: 0,
      avgCommit: 0,
      avgTotal: 0,
      p95Total: 0,
      p99Total: 0
    };

    if (this.measurements.length === 0) return stats;

    // Calcul des moyennes
    this.measurements.forEach(m => {
      stats.avgSessionCreation += m.sessionCreation;
      stats.avgTransactionStart += m.transactionStart;
      stats.avgOperations += m.operations[0]?.duration || 0;
      stats.avgCommit += m.commit;
      stats.avgTotal += m.total;
    });

    const count = this.measurements.length;
    stats.avgSessionCreation /= count;
    stats.avgTransactionStart /= count;
    stats.avgOperations /= count;
    stats.avgCommit /= count;
    stats.avgTotal /= count;

    // Calcul P95 et P99
    const sorted = this.measurements.map(m => m.total).sort((a, b) => a - b);
    stats.p95Total = sorted[Math.floor(count * 0.95)];
    stats.p99Total = sorted[Math.floor(count * 0.99)];

    return stats;
  }
}

// Utilisation
const analyzer = new TransactionLatencyAnalyzer();

for (let i = 0; i < 100; i++) {
  await analyzer.measureTransaction(async (session) => {
    await db.collection.insertOne({ data: i }, { session });
    await db.collection.updateOne({ data: i }, { $set: { processed: true } }, { session });
  });
}

const stats = analyzer.getStats();
console.log('Statistiques de latence:');
console.log(`  Session création: ${stats.avgSessionCreation.toFixed(2)}ms`);
console.log(`  Transaction start: ${stats.avgTransactionStart.toFixed(2)}ms`);
console.log(`  Opérations: ${stats.avgOperations.toFixed(2)}ms`);
console.log(`  Commit: ${stats.avgCommit.toFixed(2)}ms`);
console.log(`  Total moyen: ${stats.avgTotal.toFixed(2)}ms`);
console.log(`  P95: ${stats.p95Total}ms`);
console.log(`  P99: ${stats.p99Total}ms`);

// Résultats typiques (Replica Set local) :
// Session création: 1.2ms
// Transaction start: 0.8ms
// Opérations: 12.5ms
// Commit: 18.3ms
// Total moyen: 32.8ms
// P95: 45ms
// P99: 67ms
```

**Comparaison : Avec vs Sans transaction** :

```
Opération simple : insertOne + updateOne

Sans transaction :
  insertOne : 2ms
  updateOne : 2ms
  Total     : 4ms

Avec transaction (snapshot + majority) :
  Session création   : 1ms
  Transaction start  : 1ms
  insertOne         : 3ms  (overhead snapshot)
  updateOne         : 3ms  (overhead snapshot)
  Commit (2PC)      : 20ms (synchronisation)
  Total             : 28ms

Overhead transactionnel : 7x plus lent
```

### Throughput : La réduction de capacité

Les transactions réduisent significativement le **throughput maximal** du système.

**Benchmark de throughput** :

```javascript
// Mesure du throughput avec et sans transactions

async function benchmarkThroughput(useTransaction, durationMs = 10000) {
  const startTime = Date.now();
  let operations = 0;
  let errors = 0;

  while (Date.now() - startTime < durationMs) {
    try {
      if (useTransaction) {
        // Avec transaction
        const session = client.startSession();
        try {
          await session.withTransaction(async () => {
            await db.test.insertOne(
              { _id: new ObjectId(), data: 'test' },
              { session }
            );
          });
        } finally {
          await session.endSession();
        }
      } else {
        // Sans transaction
        await db.test.insertOne({ _id: new ObjectId(), data: 'test' });
      }

      operations++;

    } catch (error) {
      errors++;
    }
  }

  const actualDuration = Date.now() - startTime;
  const opsPerSecond = (operations / actualDuration) * 1000;

  return {
    mode: useTransaction ? 'WITH transaction' : 'WITHOUT transaction',
    operations,
    errors,
    durationMs: actualDuration,
    opsPerSecond: opsPerSecond.toFixed(0),
    avgLatencyMs: (actualDuration / operations).toFixed(2)
  };
}

// Exécution des benchmarks
const withoutTxn = await benchmarkThroughput(false, 10000);
const withTxn = await benchmarkThroughput(true, 10000);

console.log('Résultats du benchmark:');
console.log('\nSans transaction:');
console.log(`  Operations/sec: ${withoutTxn.opsPerSecond}`);
console.log(`  Latence moyenne: ${withoutTxn.avgLatencyMs}ms`);

console.log('\nAvec transaction:');
console.log(`  Operations/sec: ${withTxn.opsPerSecond}`);
console.log(`  Latence moyenne: ${withTxn.avgLatencyMs}ms`);

console.log('\nImpact:');
const reduction = ((withoutTxn.opsPerSecond - withTxn.opsPerSecond) / withoutTxn.opsPerSecond * 100).toFixed(1);
console.log(`  Réduction throughput: ${reduction}%`);

// Résultats typiques :
// Sans transaction:
//   Operations/sec: 12500
//   Latence moyenne: 0.8ms
//
// Avec transaction:
//   Operations/sec: 1800
//   Latence moyenne: 5.6ms
//
// Impact:
//   Réduction throughput: 85.6%
```

### Utilisation mémoire et cache

Les transactions augmentent la **pression sur le cache WiredTiger** et la mémoire.

**Consommation mémoire par transaction** :

```javascript
// Estimation de l'utilisation mémoire

class TransactionMemoryEstimator {

  estimateMemoryUsage(transactionProfile) {
    const {
      numDocuments,
      avgDocumentSizeKB,
      numIndexes,
      transactionDurationMs
    } = transactionProfile;

    // 1. Snapshot isolation overhead
    // WiredTiger maintient plusieurs versions des documents
    const snapshotOverhead = numDocuments * avgDocumentSizeKB * 0.5; // 50% overhead

    // 2. Transaction metadata
    const metadataKB = 2; // ~2KB par transaction

    // 3. Lock table entries
    const lockTableKB = numDocuments * 0.1; // ~100 bytes par document

    // 4. Index versions (pour MVCC)
    const indexVersionsKB = numDocuments * numIndexes * 0.2;

    // 5. Oplog entries (en mémoire avant flush)
    const oplogKB = numDocuments * avgDocumentSizeKB * 0.3;

    const totalMemoryKB =
      snapshotOverhead +
      metadataKB +
      lockTableKB +
      indexVersionsKB +
      oplogKB;

    return {
      snapshotOverheadKB: snapshotOverhead.toFixed(2),
      metadataKB: metadataKB.toFixed(2),
      lockTableKB: lockTableKB.toFixed(2),
      indexVersionsKB: indexVersionsKB.toFixed(2),
      oplogKB: oplogKB.toFixed(2),
      totalMemoryKB: totalMemoryKB.toFixed(2),
      totalMemoryMB: (totalMemoryKB / 1024).toFixed(2)
    };
  }
}

const estimator = new TransactionMemoryEstimator();

// Scénario 1 : Petite transaction
const small = estimator.estimateMemoryUsage({
  numDocuments: 10,
  avgDocumentSizeKB: 2,
  numIndexes: 3,
  transactionDurationMs: 50
});

console.log('Petite transaction (10 docs):');
console.log(`  Mémoire totale: ${small.totalMemoryMB} MB`);

// Scénario 2 : Grande transaction
const large = estimator.estimateMemoryUsage({
  numDocuments: 1000,
  avgDocumentSizeKB: 5,
  numIndexes: 5,
  transactionDurationMs: 5000
});

console.log('\nGrande transaction (1000 docs):');
console.log(`  Mémoire totale: ${large.totalMemoryMB} MB`);
console.log(`  - Snapshot overhead: ${large.snapshotOverheadKB} KB`);
console.log(`  - Lock table: ${large.lockTableKB} KB`);
console.log(`  - Index versions: ${large.indexVersionsKB} KB`);

// Impact sur un système avec 100 transactions concurrentes :
console.log('\n100 grandes transactions concurrentes:');
console.log(`  Mémoire totale: ${(large.totalMemoryMB * 100).toFixed(2)} MB`);
console.log(`  → Pression significative sur cache WiredTiger`);
```

**Symptômes de pression mémoire** :

```javascript
// Monitorer la pression mémoire causée par les transactions

async function checkCachePressure() {
  const status = await db.serverStatus();
  const wiredTiger = status.wiredTiger;

  const metrics = {
    // Taille du cache
    cacheMaxBytes: wiredTiger.cache['maximum bytes configured'],
    cacheCurrentBytes: wiredTiger.cache['bytes currently in the cache'],
    cacheUsagePercent: (
      (wiredTiger.cache['bytes currently in the cache'] /
       wiredTiger.cache['maximum bytes configured']) * 100
    ).toFixed(2),

    // Évictions (mauvais signe si élevé)
    evictions: wiredTiger.cache['pages evicted by application threads'],

    // Transactions snapshot actives
    snapshotsActive: wiredTiger.snapshot['transaction range of IDs currently active'],

    // Transaction la plus ancienne
    oldestTxnTimestamp: wiredTiger.transaction['transaction range']
  };

  console.log('Pression sur le cache:');
  console.log(`  Utilisation: ${metrics.cacheUsagePercent}%`);
  console.log(`  Évictions: ${metrics.evictions}`);
  console.log(`  Snapshots actifs: ${metrics.snapshotsActive}`);

  // Alertes
  if (metrics.cacheUsagePercent > 95) {
    console.warn('⚠️  Cache saturé - risque de dégradation performance');
  }

  if (metrics.evictions > 1000) {
    console.warn('⚠️  Évictions élevées - augmenter cache ou réduire transactions');
  }

  return metrics;
}
```

## Contention et conflits

### Write Conflicts : La malédiction des transactions concurrentes

Les transactions utilisant `readConcern: snapshot` peuvent échouer avec des **WriteConflict** si elles tentent de modifier les mêmes documents.

**Scénario de conflit** :

```javascript
// Démonstration de write conflict

async function demonstrateWriteConflict() {
  const accountId = 'ACC-123';

  // Transaction 1
  const txn1 = (async () => {
    const session1 = client.startSession();
    try {
      await session1.withTransaction(async () => {
        // Lire le compte
        const account = await db.accounts.findOne(
          { _id: accountId },
          { session: session1 }
        );

        console.log('TXN1: Lu le compte, solde =', account.balance);

        // Simuler traitement
        await new Promise(resolve => setTimeout(resolve, 100));

        // Mettre à jour
        await db.accounts.updateOne(
          { _id: accountId },
          { $inc: { balance: -100 } },
          { session: session1 }
        );

        console.log('TXN1: Compte débité de 100');
      });
    } finally {
      await session1.endSession();
    }
  })();

  // Transaction 2 (commence presque en même temps)
  await new Promise(resolve => setTimeout(resolve, 10));

  const txn2 = (async () => {
    const session2 = client.startSession();
    try {
      await session2.withTransaction(async () => {
        // Lire le compte
        const account = await db.accounts.findOne(
          { _id: accountId },
          { session: session2 }
        );

        console.log('TXN2: Lu le compte, solde =', account.balance);

        // Simuler traitement
        await new Promise(resolve => setTimeout(resolve, 100));

        // Mettre à jour - CONFLIT !
        await db.accounts.updateOne(
          { _id: accountId },
          { $inc: { balance: -50 } },
          { session: session2 }
        );

        console.log('TXN2: Compte débité de 50');
      });
    } catch (error) {
      if (error.hasErrorLabel('TransientTransactionError')) {
        console.log('TXN2: ❌ WriteConflict détecté, abort automatique');
      }
      throw error;
    } finally {
      await session2.endSession();
    }
  })();

  try {
    await Promise.all([txn1, txn2]);
  } catch (error) {
    console.log('Une transaction a échoué (comportement normal)');
  }
}

// Sortie typique :
// TXN1: Lu le compte, solde = 1000
// TXN2: Lu le compte, solde = 1000
// TXN1: Compte débité de 100
// TXN2: ❌ WriteConflict détecté, abort automatique
```

**Stratégies de mitigation** :

```javascript
// Stratégie 1 : Retry avec backoff exponentiel

async function updateWithRetry(accountId, amount, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const session = client.startSession();

    try {
      await session.withTransaction(async () => {
        await db.accounts.updateOne(
          { _id: accountId },
          { $inc: { balance: -amount } },
          { session }
        );
      });

      // Succès
      return { success: true, attempts: attempt + 1 };

    } catch (error) {
      if (error.hasErrorLabel('TransientTransactionError') && attempt < maxRetries - 1) {
        // Backoff exponentiel avec jitter
        const baseDelay = Math.pow(2, attempt) * 50;
        const jitter = Math.random() * 50;
        const delay = baseDelay + jitter;

        console.log(`Attempt ${attempt + 1} failed, retrying in ${delay.toFixed(0)}ms`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }

      throw error;

    } finally {
      await session.endSession();
    }
  }

  throw new Error('Max retries exceeded');
}

// Stratégie 2 : Réduire la fenêtre de conflit

async function minimizeConflictWindow(accountId, amount) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {

      // ❌ MAUVAIS : Lecture puis traitement puis écriture
      // const account = await db.accounts.findOne({ _id: accountId }, { session });
      // const result = expensiveCalculation(account);  // Augmente window
      // await db.accounts.updateOne(..., { session });

      // ✅ BON : Tout le traitement HORS transaction
      // puis transaction ultra-courte pour l'écriture atomique
      await db.accounts.updateOne(
        {
          _id: accountId,
          balance: { $gte: amount }  // Vérification atomique
        },
        {
          $inc: { balance: -amount }
        },
        { session }
      );

    }, {
      maxCommitTimeMS: 5000  // Timeout court
    });
  } finally {
    await session.endSession();
  }
}
```

**Mesure du taux de conflits** :

```javascript
// Monitorer les write conflicts

class WriteConflictMonitor {
  constructor() {
    this.stats = {
      totalTransactions: 0,
      conflicts: 0,
      conflictRate: 0,
      retries: 0,
      failures: 0
    };
  }

  async executeWithMonitoring(transactionFn) {
    this.stats.totalTransactions++;
    let retryCount = 0;
    const maxRetries = 3;

    for (let attempt = 0; attempt < maxRetries; attempt++) {
      try {
        await transactionFn();

        if (retryCount > 0) {
          this.stats.retries += retryCount;
        }

        return { success: true, retries: retryCount };

      } catch (error) {
        if (error.hasErrorLabel('TransientTransactionError')) {
          this.stats.conflicts++;
          retryCount++;

          if (attempt < maxRetries - 1) {
            await new Promise(resolve => setTimeout(resolve, 50 * Math.pow(2, attempt)));
            continue;
          }
        }

        this.stats.failures++;
        throw error;
      }
    }
  }

  getReport() {
    this.stats.conflictRate = this.stats.totalTransactions > 0
      ? (this.stats.conflicts / this.stats.totalTransactions * 100).toFixed(2)
      : 0;

    return {
      ...this.stats,
      avgRetriesPerConflict: this.stats.conflicts > 0
        ? (this.stats.retries / this.stats.conflicts).toFixed(2)
        : 0
    };
  }
}

// Utilisation
const monitor = new WriteConflictMonitor();

// Simuler charge concurrente
const promises = [];
for (let i = 0; i < 100; i++) {
  promises.push(
    monitor.executeWithMonitoring(async () => {
      // Transaction qui peut causer des conflits
      const session = client.startSession();
      try {
        await session.withTransaction(async () => {
          await db.counter.updateOne(
            { _id: 'global' },
            { $inc: { count: 1 } },
            { session }
          );
        });
      } finally {
        await session.endSession();
      }
    })
  );
}

await Promise.allSettled(promises);

const report = monitor.getReport();
console.log('Rapport de conflits:');
console.log(`  Total transactions: ${report.totalTransactions}`);
console.log(`  Conflits: ${report.conflicts}`);
console.log(`  Taux de conflit: ${report.conflictRate}%`);
console.log(`  Retries moyens par conflit: ${report.avgRetriesPerConflict}`);
console.log(`  Échecs finaux: ${report.failures}`);
```

## Impact sur les opérations normales

### Blocage des lectures non transactionnelles

Les transactions peuvent **bloquer les lectures normales** sur les mêmes documents.

```javascript
// Démonstration du blocage

async function demonstrateReadBlocking() {
  const docId = 'doc-123';

  // Transaction longue qui modifie un document
  const longTransaction = (async () => {
    const session = client.startSession();
    try {
      await session.withTransaction(async () => {
        console.log('TXN: Début modification');

        await db.docs.updateOne(
          { _id: docId },
          { $set: { status: 'processing' } },
          { session }
        );

        // Simuler traitement long
        await new Promise(resolve => setTimeout(resolve, 5000));

        await db.docs.updateOne(
          { _id: docId },
          { $set: { status: 'completed' } },
          { session }
        );

        console.log('TXN: Fin modification');
      });
    } finally {
      await session.endSession();
    }
  })();

  // Lecture normale pendant la transaction
  await new Promise(resolve => setTimeout(resolve, 1000));

  const normalRead = (async () => {
    const start = Date.now();
    console.log('READ: Tentative de lecture...');

    // Lecture avec readConcern:majority peut être bloquée
    const doc = await db.docs.findOne(
      { _id: docId },
      { readConcern: { level: 'majority' } }
    );

    const duration = Date.now() - start;
    console.log(`READ: Document lu après ${duration}ms`);
    return doc;
  })();

  await Promise.all([longTransaction, normalRead]);
}

// Sortie :
// TXN: Début modification
// READ: Tentative de lecture...
// TXN: Fin modification
// READ: Document lu après 4050ms
// → La lecture a été bloquée 4 secondes !
```

### Amplification du Replication Lag

Les transactions **amplifient le replication lag**, surtout les transactions longues ou volumineuses.

```javascript
// Impact sur le replication lag

async function measureReplicationImpact() {

  // Mesurer lag avant
  const lagBefore = await getReplicationLag();
  console.log(`Replication lag avant: ${lagBefore}ms`);

  // Exécuter une transaction volumineuse
  const session = client.startSession();
  try {
    await session.withTransaction(async () => {
      // Insérer 1000 documents dans une transaction
      const docs = Array.from({ length: 1000 }, (_, i) => ({
        _id: new ObjectId(),
        data: `Document ${i}`,
        timestamp: new Date()
      }));

      await db.large_txn.insertMany(docs, { session });
    });
  } finally {
    await session.endSession();
  }

  // Mesurer lag après
  await new Promise(resolve => setTimeout(resolve, 1000));
  const lagAfter = await getReplicationLag();
  console.log(`Replication lag après: ${lagAfter}ms`);
  console.log(`Augmentation: ${lagAfter - lagBefore}ms`);
}

async function getReplicationLag() {
  const status = await db.adminCommand({ replSetGetStatus: 1 });
  const primary = status.members.find(m => m.stateStr === 'PRIMARY');
  const secondaries = status.members.filter(m => m.stateStr === 'SECONDARY');

  if (secondaries.length === 0) return 0;

  const primaryOptime = primary.optimeDate.getTime();
  const maxLag = Math.max(...secondaries.map(s =>
    primaryOptime - s.optimeDate.getTime()
  ));

  return maxLag;
}

// Résultat typique :
// Replication lag avant: 50ms
// Replication lag après: 850ms
// Augmentation: 800ms
// → Transaction volumineuse a multiplié le lag par 17 !
```

## Stratégies d'optimisation

### Optimisation 1 : Minimiser la portée transactionnelle

```javascript
// ❌ MAUVAIS : Tout dans une transaction

async function processOrderBad(orderData) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // 1. Calculs complexes (devrait être hors transaction)
      const pricing = calculateComplexPricing(orderData);

      // 2. Appel API externe (devrait être hors transaction)
      const validation = await externalPaymentValidation(pricing);

      // 3. Envoi email (devrait être hors transaction)
      await sendConfirmationEmail(orderData.email);

      // 4. Opérations DB
      await db.orders.insertOne({ ...orderData, pricing }, { session });
      await db.inventory.updateMany(
        { _id: { $in: orderData.items } },
        { $inc: { stock: -1 } },
        { session }
      );
    });
  } finally {
    await session.endSession();
  }
}

// ✅ BON : Seulement le strictement nécessaire

async function processOrderGood(orderData) {
  // 1. Calculs HORS transaction
  const pricing = calculateComplexPricing(orderData);

  // 2. Validation HORS transaction
  const validation = await externalPaymentValidation(pricing);

  if (!validation.success) {
    throw new Error('Payment validation failed');
  }

  // 3. Transaction COURTE et FOCALISÉE
  const session = client.startSession();
  let orderId;

  try {
    await session.withTransaction(async () => {
      // SEULEMENT les opérations atomiques critiques
      const orderResult = await db.orders.insertOne(
        { ...orderData, pricing, validationId: validation.id },
        { session }
      );
      orderId = orderResult.insertedId;

      await db.inventory.updateMany(
        { _id: { $in: orderData.items } },
        { $inc: { stock: -1 } },
        { session }
      );
    });
  } finally {
    await session.endSession();
  }

  // 4. Email HORS transaction (asynchrone)
  sendConfirmationEmail(orderData.email, orderId).catch(console.error);

  return orderId;
}
```

### Optimisation 2 : Utiliser bulkWrite quand possible

```javascript
// Optimiser les opérations multiples avec bulkWrite

// ❌ Moins performant : Opérations individuelles
async function updateManyIndividual(updates) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      for (const update of updates) {
        await db.collection.updateOne(
          { _id: update.id },
          { $set: update.data },
          { session }
        );
      }
    });
  } finally {
    await session.endSession();
  }
}

// ✅ Plus performant : bulkWrite
async function updateManyBulk(updates) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      const bulkOps = updates.map(update => ({
        updateOne: {
          filter: { _id: update.id },
          update: { $set: update.data }
        }
      }));

      await db.collection.bulkWrite(bulkOps, {
        session,
        ordered: false  // Paralléliser quand possible
      });
    });
  } finally {
    await session.endSession();
  }
}

// Benchmark : 100 updates
// Individuel : ~250ms
// BulkWrite : ~45ms
// Amélioration : 5.5x plus rapide
```

### Optimisation 3 : Configuration adaptée

```javascript
// Configuration optimale selon le cas d'usage

const transactionConfigs = {

  // Configuration ULTRA-RAPIDE : Lectures non critiques
  fast: {
    readConcern: { level: 'local' },
    writeConcern: { w: 1 },
    maxCommitTimeMS: 5000
  },

  // Configuration ÉQUILIBRÉE : Usage général
  balanced: {
    readConcern: { level: 'majority' },
    writeConcern: { w: 'majority' },
    maxCommitTimeMS: 10000
  },

  // Configuration STRICTE : Opérations critiques
  strict: {
    readConcern: { level: 'snapshot' },
    writeConcern: { w: 'majority', j: true },
    maxCommitTimeMS: 30000
  }
};

// Usage
async function executeTransaction(operations, priority = 'balanced') {
  const config = transactionConfigs[priority];
  const session = client.startSession();

  try {
    await session.withTransaction(operations, config);
  } finally {
    await session.endSession();
  }
}
```

## Recommandations finales

### Checklist de performance

```markdown
## Checklist : Optimisation des transactions

### Conception
- [ ] Transaction durée < 5 secondes (idéal < 1s)
- [ ] Nombre d'opérations < 1000
- [ ] Taille totale < 8 MB (50% de la limite)
- [ ] Traitement métier HORS transaction
- [ ] Appels externes HORS transaction

### Configuration
- [ ] ReadConcern adapté au cas d'usage
- [ ] WriteConcern optimal (pas de sur-garantie)
- [ ] maxCommitTimeMS configuré (ne pas laisser par défaut)
- [ ] Retry logic implémenté

### Monitoring
- [ ] Latence P99 < 200ms
- [ ] Taux de write conflict < 5%
- [ ] Cache WiredTiger < 90%
- [ ] Replication lag < 1s

### Tests de charge
- [ ] Testé avec charge concurrente réaliste
- [ ] Testé scénarios de contention
- [ ] Testé avec replication lag simulé
- [ ] Benchmark vs alternative sans transaction
```

### Arbre de décision

```
Ai-je besoin d'une transaction ?
├─ NON → Utiliser opérations atomiques simples
│
└─ OUI → Est-ce que < 100 documents ?
    ├─ NON → Considérer batching ou Saga pattern
    │
    └─ OUI → Est-ce que < 5 secondes ?
        ├─ NON → Refactorer pour réduire la portée
        │
        └─ OUI → Est-ce critique pour le business ?
            ├─ NON → Configuration 'balanced'
            │
            └─ OUI → Configuration 'strict' + monitoring renforcé
```

## Conclusion

Les limites des transactions MongoDB ne sont pas des obstacles, mais des **garde-fous** qui orientent vers des designs robustes et performants. Les systèmes les plus réussis ne sont pas ceux qui utilisent massivement les transactions, mais ceux qui les utilisent **judicieusement** :

**Principes directeurs** :

1. **Minimiser** : Transactions les plus courtes et étroites possibles
2. **Différencier** : Configurations adaptées à chaque cas d'usage
3. **Mesurer** : Monitoring constant et benchmarking régulier
4. **Optimiser** : Amélioration continue basée sur les métriques
5. **Alternatives** : Considérer Saga, versioning optimiste, eventual consistency

Une transaction optimale est une transaction que vous **n'avez pas eu à faire**. Quand elle est inévitable, elle doit être **rapide, focalisée, et monitored**.

---

**Points clés à retenir** :

- Limite stricte : 16 MB, 60 secondes, ~1000 opérations recommandé
- Overhead latence : 4-7x plus lent que sans transaction
- Overhead throughput : Réduction de 80-90% de la capacité
- Write conflicts : Inévitables sous charge concurrente
- Optimisation : Minimiser portée, utiliser bulkWrite, configuration adaptée
- Monitoring essentiel : Latence, conflits, cache, replication lag
- Alternatives souvent meilleures : Saga, versioning, eventual consistency

⏭️ [Bonnes pratiques transactionnelles](/08-transactions/07-bonnes-pratiques-transactionnelles.md)
