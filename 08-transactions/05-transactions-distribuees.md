🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.5 Transactions distribuées

## Introduction : La complexité des transactions à travers les shards

Les transactions distribuées représentent l'un des défis les plus complexes des systèmes de bases de données modernes. Alors qu'une transaction dans un Replica Set simple ne concerne qu'un ensemble de nœuds géographiquement proches partageant les mêmes données, une transaction distribuée doit coordonner des opérations **à travers plusieurs shards**, potentiellement situés dans différents datacenters, gérant des partitions de données différentes.

MongoDB supporte les transactions multi-documents distribuées depuis la version 4.2, permettant des garanties ACID même dans un environnement shardé. Cependant, cette capacité vient avec des **coûts significatifs** en termes de performance, complexité et limitations opérationnelles qu'il est crucial de comprendre.

### Différence fondamentale : Replica Set vs Sharded Cluster

```
Transaction dans un Replica Set :
┌─────────────────────────────────────┐
│  Replica Set "Products"             │
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │Primary │→ │Second. │  │Second. │ │
│  └────────┘  └────────┘  └────────┘ │
│                                     │
│  Transaction coordonnée par         │
│  le Primary local                   │
└─────────────────────────────────────┘

Latence typique : 20-50ms
Coordination : Simple (un nœud coordinateur)
Points de défaillance : Réduits


Transaction distribuée (Sharded Cluster) :
┌──────────────────────────────────────────────────┐
│  Cluster Shardé                                  │
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
│  │  Shard A    │  │  Shard B    │  │  Shard C │  │
│  │  (users)    │  │  (orders)   │  │ (payment)│  │
│  │  RS:3 nodes │  │  RS:3 nodes │  │ RS:3nodes│  │
│  └─────────────┘  └─────────────┘  └──────────┘  │
│         ↑               ↑               ↑        │
│         └───────────────┼───────────────┘        │
│                         │                        │
│                  ┌──────┴──────┐                 │
│                  │   Mongos    │                 │
│                  │ (coordinateur)                │
│                  └─────────────┘                 │
└──────────────────────────────────────────────────┘

Latence typique : 100-500ms
Coordination : Complexe (protocole 2PC)
Points de défaillance : Multiples
```

## Protocole de commit distribué : Two-Phase Commit (2PC)

MongoDB utilise une variante du protocole de commit en deux phases pour garantir l'atomicité des transactions distribuées. Comprendre ce protocole est essentiel pour anticiper les comportements et les limitations.

### Phase 1 : Préparation (Prepare Phase)

Le coordinateur (mongos) demande à chaque shard participant si il peut committer la transaction.

```
Timeline détaillée d'une transaction distribuée :

T0 : Client démarre transaction
     session.startTransaction()

T1 : Client exécute opérations
     - INSERT dans Shard A (users)
     - UPDATE dans Shard B (orders)
     - INSERT dans Shard C (payments)

     Chaque shard maintient les modifications en mémoire
     avec des verrous (locks) sur les documents concernés

T2 : Client demande commit
     session.commitTransaction()

PHASE 1 - PREPARE :
─────────────────────
T3 : Mongos envoie "prepare" à tous les shards participants

T4 : Shard A vérifie :
     ✓ Pas de conflits ?
     ✓ Contraintes satisfaites ?
     ✓ Ressources disponibles ?
     → Répond "prepared" OU "abort"

T5 : Shard B vérifie (même chose)
     → Répond "prepared" OU "abort"

T6 : Shard C vérifie (même chose)
     → Répond "prepared" OU "abort"

PHASE 2 - COMMIT/ABORT :
────────────────────────
T7 : Mongos évalue les réponses

     SI tous ont répondu "prepared" :
       → Mongos envoie "commit" à tous

     SI au moins un a répondu "abort" :
       → Mongos envoie "abort" à tous

T8 : Chaque shard exécute commit/abort
     - Applique les modifications (commit)
       OU
     - Annule les modifications (abort)
     - Libère les verrous

T9 : Shards confirment à Mongos

T10: Mongos confirme au client
     → Succès ou échec

Durée totale typique : 100-500ms
(vs 20-50ms pour transaction Replica Set)
```

### Points critiques du protocole

**1. Période de blocage** :

Entre la phase prepare et la réception du commit/abort, chaque shard **maintient des verrous** sur les documents modifiés. Pendant cette période :

```javascript
// Shard A détient un verrou sur le document user:123
// Toute autre transaction tentant de modifier user:123 est BLOQUÉE

// Exemple de contention
Transaction T1:                    Transaction T2:
─────────────────────────────────────────────────────
UPDATE user:123
  → Verrou acquis
                                   UPDATE user:123
Prepare phase...                     → BLOQUÉ (en attente)
  (50ms)
                                     → BLOQUÉ (toujours)
Commit phase...
  (30ms)
                                     → BLOQUÉ (toujours)
Verrou libéré                        → Peut continuer

Temps de blocage pour T2 : ~80ms
```

**2. Window de vulnérabilité** :

Si le coordinateur (mongos) crash entre les phases prepare et commit, les shards restent dans un état "prepared" indéfiniment jusqu'à récupération.

```
Scénario de panne du coordinateur :

T0  : Transaction démarre
T1-T5 : Opérations exécutées sur shards A, B, C
T6  : Prepare phase commence
T7  : Shards A, B, C répondent "prepared"
T8  : ⚠️  MONGOS CRASH avant d'envoyer commit

État du système :
- Shards A, B, C : En état "prepared"
- Verrous maintenus sur les documents
- Autres transactions bloquées
- Transaction en "limbo state"

T9  : Mongos de backup détecte la transaction en suspens
T10 : Récupération : Mongos lit le journal de transaction
T11 : Mongos envoie commit/abort selon l'état enregistré
T12 : Shards finalisent la transaction

Durée du blocage en cas de panne : 10-60 secondes
Impact : Toutes les transactions touchant ces documents sont bloquées
```

## Architecture et composants

### Rôle de mongos dans les transactions distribuées

Le mongos n'est plus un simple routeur de requêtes, il devient le **coordinateur de transaction** avec des responsabilités critiques :

```javascript
// Responsabilités de mongos en transaction distribuée

class TransactionCoordinator {
  constructor() {
    this.activeTransactions = new Map();
    this.transactionLog = []; // Journal pour récupération
  }

  // 1. Initialisation de la transaction
  startTransaction(session) {
    const txnId = generateTransactionId();
    const coordinatorState = {
      txnId,
      participants: new Set(), // Shards impliqués
      state: 'active',
      startTime: Date.now(),
      operations: []
    };

    this.activeTransactions.set(txnId, coordinatorState);
    return txnId;
  }

  // 2. Routage des opérations et tracking des participants
  routeOperation(txnId, operation) {
    const shard = this.determineTargetShard(operation);
    const txnState = this.activeTransactions.get(txnId);

    // Ajouter le shard aux participants
    txnState.participants.add(shard);

    // Router l'opération
    return this.executeOnShard(shard, operation, txnId);
  }

  // 3. Phase Prepare
  async preparePhase(txnId) {
    const txnState = this.activeTransactions.get(txnId);
    const preparePromises = [];

    // Envoyer prepare à tous les participants
    for (const shard of txnState.participants) {
      preparePromises.push(
        this.sendPrepareToShard(shard, txnId)
      );
    }

    // Attendre toutes les réponses (avec timeout)
    try {
      const results = await Promise.all(preparePromises);

      // Vérifier que tous ont répondu "prepared"
      const allPrepared = results.every(r => r.status === 'prepared');

      if (allPrepared) {
        txnState.state = 'prepared';
        this.logTransaction(txnId, 'prepared');
        return { decision: 'commit' };
      } else {
        return { decision: 'abort' };
      }

    } catch (error) {
      console.error('Prepare phase timeout:', error);
      return { decision: 'abort' };
    }
  }

  // 4. Phase Commit/Abort
  async commitPhase(txnId, decision) {
    const txnState = this.activeTransactions.get(txnId);
    const commitPromises = [];

    // Enregistrer la décision AVANT de l'envoyer
    this.logTransaction(txnId, decision);

    // Envoyer commit/abort à tous les participants
    for (const shard of txnState.participants) {
      if (decision === 'commit') {
        commitPromises.push(this.sendCommitToShard(shard, txnId));
      } else {
        commitPromises.push(this.sendAbortToShard(shard, txnId));
      }
    }

    // Attendre les confirmations
    await Promise.all(commitPromises);

    // Nettoyer
    this.activeTransactions.delete(txnId);
    txnState.state = decision === 'commit' ? 'committed' : 'aborted';
  }

  // 5. Récupération après panne
  async recoverTransactions() {
    // Lire le journal des transactions
    const pendingTransactions = this.readTransactionLog();

    for (const txn of pendingTransactions) {
      if (txn.state === 'prepared') {
        // Transaction en limbo - finaliser le commit
        console.log(`Récupération transaction ${txn.txnId}`);
        await this.commitPhase(txn.txnId, txn.decision);
      }
    }
  }
}
```

### Configuration des shards pour les transactions

Chaque shard doit être configuré pour supporter les transactions distribuées :

```javascript
// Configuration requise sur chaque Replica Set shard

// 1. Activer le journal (obligatoire)
db.adminCommand({
  setParameter: 1,
  wiredTigerEngineRuntimeConfig: "eviction=(threads_min=4,threads_max=8)"
});

// 2. Configurer les paramètres de transaction
db.adminCommand({
  setParameter: 1,
  transactionLifetimeLimitSeconds: 60  // Max 60 secondes
});

// 3. Augmenter le cache WiredTiger si beaucoup de transactions
db.adminCommand({
  setParameter: 1,
  wiredTigerCacheSizeGB: 4  // Selon la RAM disponible
});
```

## Cas d'usage et exemples réalistes

### Cas 1 : Plateforme e-commerce - Commande distribuée

**Architecture des données** :

```javascript
// Données réparties sur 3 shards

// Shard A : Collection "users" (sharded by userId)
{
  _id: "user123",
  name: "Alice",
  email: "alice@example.com",
  balance: 1000.00  // Crédit magasin
}

// Shard B : Collection "products" (sharded by category)
{
  _id: "prod456",
  name: "Laptop",
  category: "electronics",
  price: 899.99,
  stock: 50
}

// Shard C : Collection "orders" (sharded by orderId)
{
  _id: "order789",
  userId: "user123",
  items: [...],
  total: 899.99,
  status: "pending"
}
```

**Implémentation de la transaction distribuée** :

```javascript
async function createOrder(userId, items) {
  const session = client.startSession();

  try {
    // Démarrer la transaction distribuée
    await session.withTransaction(async () => {

      // ─────────────────────────────────────────
      // OPÉRATION 1 : Vérifier le solde (Shard A)
      // ─────────────────────────────────────────
      const user = await db.users.findOne(
        { _id: userId },
        { session }
      );

      if (!user) {
        throw new Error('Utilisateur introuvable');
      }

      // Calculer le total
      const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

      if (user.balance < total) {
        throw new Error('Solde insuffisant');
      }

      // ─────────────────────────────────────────
      // OPÉRATION 2 : Vérifier et réserver stock (Shard B)
      // ─────────────────────────────────────────
      for (const item of items) {
        const product = await db.products.findOneAndUpdate(
          {
            _id: item.productId,
            stock: { $gte: item.quantity }  // Stock suffisant
          },
          {
            $inc: { stock: -item.quantity }  // Décrémenter
          },
          {
            session,
            returnDocument: 'after'
          }
        );

        if (!product.value) {
          throw new Error(`Stock insuffisant pour ${item.productId}`);
        }
      }

      // ─────────────────────────────────────────
      // OPÉRATION 3 : Créer la commande (Shard C)
      // ─────────────────────────────────────────
      const orderId = new ObjectId();
      await db.orders.insertOne(
        {
          _id: orderId,
          userId,
          items,
          total,
          status: 'confirmed',
          createdAt: new Date()
        },
        { session }
      );

      // ─────────────────────────────────────────
      // OPÉRATION 4 : Débiter le compte (Shard A)
      // ─────────────────────────────────────────
      await db.users.updateOne(
        { _id: userId },
        { $inc: { balance: -total } },
        { session }
      );

      // ─────────────────────────────────────────
      // Si on arrive ici, toutes les opérations ont réussi
      // MongoDB va maintenant exécuter le 2PC
      // ─────────────────────────────────────────

      return { orderId, total };

    }, {
      // Options de transaction
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority' },
      readPreference: 'primary',
      maxCommitTimeMS: 30000  // 30 secondes max
    });

    console.log('✅ Commande créée avec succès');
    return { success: true };

  } catch (error) {
    console.error('❌ Échec de la commande:', error);

    // MongoDB a automatiquement rollback toutes les opérations
    // - Stock restauré
    // - Solde non débité
    // - Commande non créée

    return { success: false, error: error.message };

  } finally {
    await session.endSession();
  }
}

// Utilisation
await createOrder('user123', [
  { productId: 'prod456', quantity: 1, price: 899.99 }
]);
```

**Analyse de performance** :

```
Timeline de la transaction distribuée :

T0     : startTransaction()                           [0ms]
T1-T10 : Opérations sur les shards                   [50ms]
         - Read user (Shard A)
         - Update products (Shard B)
         - Insert order (Shard C)
         - Update user (Shard A)

T11    : commitTransaction() appelé                   [50ms]

PHASE PREPARE :
T12    : Mongos → "prepare" vers Shards A, B, C      [55ms]
T13    : Shard A vérifie et répond "prepared"        [70ms]
T14    : Shard B vérifie et répond "prepared"        [75ms]
T15    : Shard C vérifie et répond "prepared"        [80ms]
T16    : Mongos reçoit tous les "prepared"           [85ms]

PHASE COMMIT :
T17    : Mongos enregistre décision "commit"         [90ms]
T18    : Mongos → "commit" vers Shards A, B, C       [95ms]
T19    : Shard A commit et libère verrous           [110ms]
T20    : Shard B commit et libère verrous           [115ms]
T21    : Shard C commit et libère verrous           [120ms]
T22    : Mongos confirme au client                  [125ms]

TOTAL : ~125ms

Comparaison :
- Transaction Replica Set simple : ~20-30ms
- Transaction distribuée : ~125ms (4-6x plus lent)
```

### Cas 2 : Système bancaire - Transfert inter-comptes multi-entités

**Scénario complexe** : Transfert entre comptes de banques différentes, chaque banque étant un shard distinct.

```javascript
// Architecture shardée par institution bancaire

// Shard "BANK_A" : Comptes de la banque A
// Shard "BANK_B" : Comptes de la banque B
// Shard "CENTRAL" : Journal central des transactions

async function interBankTransfer(fromAccount, toAccount, amount) {
  const session = client.startSession();

  // Métadonnées pour audit
  const transferId = new ObjectId();
  const timestamp = new Date();

  try {
    const result = await session.withTransaction(async () => {

      // ─────────────────────────────────────────
      // 1. Vérifier compte source (Shard BANK_A)
      // ─────────────────────────────────────────
      const sourceAccount = await db.accounts.findOne(
        {
          accountNumber: fromAccount,
          bank: 'BANK_A'
        },
        { session }
      );

      if (!sourceAccount) {
        throw new Error('Compte source introuvable');
      }

      if (sourceAccount.balance < amount) {
        throw new Error('Solde insuffisant');
      }

      if (sourceAccount.status !== 'active') {
        throw new Error('Compte source non actif');
      }

      // ─────────────────────────────────────────
      // 2. Vérifier compte destination (Shard BANK_B)
      // ─────────────────────────────────────────
      const destAccount = await db.accounts.findOne(
        {
          accountNumber: toAccount,
          bank: 'BANK_B'
        },
        { session }
      );

      if (!destAccount) {
        throw new Error('Compte destination introuvable');
      }

      if (destAccount.status !== 'active') {
        throw new Error('Compte destination non actif');
      }

      // ─────────────────────────────────────────
      // 3. Enregistrer dans le journal central (Shard CENTRAL)
      // ─────────────────────────────────────────
      await db.transfer_journal.insertOne(
        {
          _id: transferId,
          from: {
            bank: 'BANK_A',
            account: fromAccount,
            prevBalance: sourceAccount.balance
          },
          to: {
            bank: 'BANK_B',
            account: toAccount,
            prevBalance: destAccount.balance
          },
          amount,
          status: 'processing',
          timestamp,
          metadata: {
            initiatedBy: sourceAccount.holder,
            ipAddress: '...',
            userAgent: '...'
          }
        },
        { session }
      );

      // ─────────────────────────────────────────
      // 4. Débiter le compte source (Shard BANK_A)
      // ─────────────────────────────────────────
      const debitResult = await db.accounts.findOneAndUpdate(
        {
          accountNumber: fromAccount,
          bank: 'BANK_A',
          balance: { $gte: amount }  // Vérification atomique
        },
        {
          $inc: { balance: -amount },
          $push: {
            transactions: {
              transferId,
              type: 'debit',
              amount,
              timestamp
            }
          }
        },
        {
          session,
          returnDocument: 'after'
        }
      );

      if (!debitResult.value) {
        throw new Error('Échec du débit (race condition)');
      }

      // ─────────────────────────────────────────
      // 5. Créditer le compte destination (Shard BANK_B)
      // ─────────────────────────────────────────
      await db.accounts.updateOne(
        {
          accountNumber: toAccount,
          bank: 'BANK_B'
        },
        {
          $inc: { balance: amount },
          $push: {
            transactions: {
              transferId,
              type: 'credit',
              amount,
              timestamp
            }
          }
        },
        { session }
      );

      // ─────────────────────────────────────────
      // 6. Marquer le transfert comme complété (Shard CENTRAL)
      // ─────────────────────────────────────────
      await db.transfer_journal.updateOne(
        { _id: transferId },
        {
          $set: {
            status: 'completed',
            completedAt: new Date()
          }
        },
        { session }
      );

      return {
        transferId,
        newBalanceSource: debitResult.value.balance,
        newBalanceDest: destAccount.balance + amount
      };

    }, {
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority', j: true },  // Journalisation obligatoire
      maxCommitTimeMS: 60000  // 1 minute max pour ce cas critique
    });

    console.log('✅ Transfert inter-bancaire réussi:', result);
    return { success: true, ...result };

  } catch (error) {
    console.error('❌ Échec du transfert:', error);

    // En cas d'échec, enregistrer dans les logs d'audit
    await db.failed_transfers.insertOne({
      fromAccount,
      toAccount,
      amount,
      error: error.message,
      timestamp: new Date()
    });

    return { success: false, error: error.message };

  } finally {
    await session.endSession();
  }
}
```

**Garanties ACID dans ce scénario** :

```
Atomicité :
- Soit TOUTES les opérations réussissent (6 opérations sur 3 shards)
- Soit AUCUNE n'est appliquée
- Pas d'état intermédiaire visible

Cohérence :
- Les invariants sont préservés :
  * Solde source ≥ montant
  * Comptes actifs
  * Balance totale du système inchangée
    (débit + crédit = 0)

Isolation :
- Niveau snapshot : Aucune autre transaction ne peut voir
  les modifications partielles
- Verrous empêchent les modifications concurrentes sur
  les mêmes comptes

Durabilité :
- writeConcern: { w: 'majority', j: true }
- Garantie de persistance sur disque (journal)
- Survit à un crash de n'importe quel composant
```

## Limitations et considérations de performance

### Limitation 1 : Nombre de documents modifiés

MongoDB impose des **limites strictes** sur les transactions distribuées :

```javascript
// Limites par transaction (MongoDB 7.0)

const TRANSACTION_LIMITS = {
  // Taille totale des opérations
  maxSizeBytes: 16 * 1024 * 1024,  // 16 MB

  // Durée maximale
  maxDurationSeconds: 60,  // 1 minute par défaut (configurable)

  // Nombre d'opérations
  maxOperations: 1000,  // Recommandation (pas de limite stricte)

  // Documents affectés
  maxDocumentsAffected: 'Pas de limite stricte, mais impact performance'
};

// ❌ ANTIPATTERN : Transaction trop volumineuse
async function massUpdate() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // Modifier 10,000 documents dans une transaction
      await db.products.updateMany(
        { category: 'electronics' },
        { $inc: { price: 10 } },
        { session }
      );
      // PROBLÈME :
      // - Transaction peut dépasser 16MB
      // - Timeout probable
      // - Verrous maintenus trop longtemps
      // - Impact sur toutes les autres transactions
    });
  } finally {
    await session.endSession();
  }
}

// ✅ SOLUTION : Batching sans transaction
async function massUpdateBatched() {
  const batchSize = 100;
  let processed = 0;

  while (true) {
    // Traiter par lots SANS transaction
    const result = await db.products.updateMany(
      {
        category: 'electronics',
        _processed: { $ne: true }  // Idempotence
      },
      {
        $inc: { price: 10 },
        $set: { _processed: true }
      },
      { limit: batchSize }
    );

    processed += result.modifiedCount;

    if (result.modifiedCount < batchSize) {
      break;  // Terminé
    }

    // Pause pour éviter de saturer
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  console.log(`${processed} documents mis à jour par lots`);
}
```

### Limitation 2 : Performance et latence

Les transactions distribuées sont **significativement plus lentes** que les transactions locales :

```
Benchmark : Latence des transactions (P99)

Type de transaction                    Latence P99    Throughput max
────────────────────────────────────────────────────────────────────
Mono-document (pas de transaction)     5-10ms         50,000 ops/s
Multi-documents (Replica Set)          40-80ms        5,000 ops/s
Multi-documents (2 shards)             150-300ms      1,000 ops/s
Multi-documents (3+ shards)            300-800ms      500 ops/s
Multi-documents (géo-distribué)        1-3 secondes   100 ops/s

Facteurs d'impact :
- Nombre de shards participants : Linéaire
- Latence réseau inter-shards : Critique
- Nombre d'opérations : Modéré
- Contention sur les documents : Quadratique
```

**Visualisation de l'impact du nombre de shards** :

```javascript
// Impact du nombre de shards participants

function estimateTransactionLatency(config) {
  const {
    numShards,
    networkLatencyMs,
    operationsPerShard,
    baseOperationMs
  } = config;

  // Latence des opérations
  const operationTime = numShards * operationsPerShard * baseOperationMs;

  // Phase prepare : Round-trip vers tous les shards
  const preparePhase = networkLatencyMs * 2; // Aller-retour

  // Phase commit : Round-trip vers tous les shards
  const commitPhase = networkLatencyMs * 2;

  // Overhead de coordination
  const coordinationOverhead = 10 * numShards;

  const totalLatency =
    operationTime +
    preparePhase +
    commitPhase +
    coordinationOverhead;

  return {
    operationTime,
    preparePhase,
    commitPhase,
    coordinationOverhead,
    totalLatency
  };
}

// Exemples
const scenarios = [
  { name: '2 shards (même DC)', numShards: 2, networkLatencyMs: 1, operationsPerShard: 2, baseOperationMs: 2 },
  { name: '3 shards (même DC)', numShards: 3, networkLatencyMs: 1, operationsPerShard: 2, baseOperationMs: 2 },
  { name: '2 shards (inter-DC)', numShards: 2, networkLatencyMs: 50, operationsPerShard: 2, baseOperationMs: 2 },
  { name: '5 shards (géo-distribué)', numShards: 5, networkLatencyMs: 150, operationsPerShard: 3, baseOperationMs: 2 }
];

scenarios.forEach(scenario => {
  const result = estimateTransactionLatency(scenario);
  console.log(`\n${scenario.name}:`);
  console.log(`  Latence totale: ${result.totalLatency}ms`);
  console.log(`  - Opérations: ${result.operationTime}ms`);
  console.log(`  - Prepare: ${result.preparePhase}ms`);
  console.log(`  - Commit: ${result.commitPhase}ms`);
});

// Sortie :
// 2 shards (même DC):
//   Latence totale: 32ms
//
// 3 shards (même DC):
//   Latence totale: 46ms
//
// 2 shards (inter-DC):
//   Latence totale: 228ms
//
// 5 shards (géo-distribué):
//   Latence totale: 680ms
```

### Limitation 3 : Contention et deadlocks

Les transactions distribuées augmentent le risque de contention et de deadlocks :

```javascript
// Scénario de contention

// Transaction 1 :
await session1.withTransaction(async () => {
  await db.accounts.updateOne({ _id: 'A' }, { $inc: { balance: -100 } }, { session: session1 });
  // VERROU sur document A

  await sleep(10); // Simule du traitement

  await db.accounts.updateOne({ _id: 'B' }, { $inc: { balance: 100 } }, { session: session1 });
  // Attend VERROU sur document B
});

// Transaction 2 (concurrente) :
await session2.withTransaction(async () => {
  await db.accounts.updateOne({ _id: 'B' }, { $inc: { balance: -50 } }, { session: session2 });
  // VERROU sur document B

  await sleep(10);

  await db.accounts.updateOne({ _id: 'A' }, { $inc: { balance: 50 } }, { session: session2 });
  // Attend VERROU sur document A
});

// DEADLOCK :
// Transaction 1 détient A, attend B
// Transaction 2 détient B, attend A
// → MongoDB détecte et abort une des transactions
```

**Détection et résolution automatique** :

```javascript
// MongoDB détecte les deadlocks et abort automatiquement

async function transferWithRetry(fromId, toId, amount, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const session = client.startSession();

    try {
      await session.withTransaction(async () => {
        // Opérations de transfert
        await db.accounts.updateOne(
          { _id: fromId },
          { $inc: { balance: -amount } },
          { session }
        );

        await db.accounts.updateOne(
          { _id: toId },
          { $inc: { balance: amount } },
          { session }
        );
      });

      // Succès
      return { success: true };

    } catch (error) {
      if (error.hasErrorLabel('TransientTransactionError')) {
        // Erreur transitoire (deadlock, timeout, etc.)
        console.log(`Tentative ${attempt + 1} échouée, retry...`);

        // Backoff exponentiel
        const backoffMs = Math.pow(2, attempt) * 100;
        await new Promise(resolve => setTimeout(resolve, backoffMs));

        continue;  // Retry

      } else {
        // Erreur permanente (validation, etc.)
        throw error;
      }

    } finally {
      await session.endSession();
    }
  }

  throw new Error('Max retries atteint');
}
```

## Patterns et anti-patterns

### Pattern 1 : Minimiser les shards participants

**Principe** : Concevoir le schéma de sharding pour que les transactions typiques touchent le **minimum de shards possible**.

```javascript
// ❌ MAUVAIS : Sharding qui force les transactions multi-shards

// Shard par type de document
{
  users: { shardKey: { type: 1 } },      // Shard A
  orders: { shardKey: { type: 1 } },     // Shard B
  payments: { shardKey: { type: 1 } }    // Shard C
}

// Problème : Créer une commande nécessite TOUJOURS 3 shards

// ✅ BON : Sharding par tenant/customer

// Shard par userId
{
  users: { shardKey: { userId: 1 } },
  orders: { shardKey: { userId: 1 } },
  payments: { shardKey: { userId: 1 } }
}

// Avantage : Opérations d'un utilisateur sont sur le MÊME shard
// → Transactions locales au shard (pas de 2PC)
```

**Exemple concret : Application multi-tenant** :

```javascript
// Conception optimale pour minimiser les transactions distribuées

// Toutes les collections shardées par tenantId
const collections = {
  customers: {
    shardKey: { tenantId: 1, customerId: 1 }
  },
  invoices: {
    shardKey: { tenantId: 1, invoiceId: 1 }
  },
  payments: {
    shardKey: { tenantId: 1, paymentId: 1 }
  }
};

// Résultat : 95% des transactions sont locales à un shard
// Seulement les opérations inter-tenant nécessitent 2PC

async function createInvoiceAndPayment(tenantId, customerId, invoiceData) {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // Toutes ces opérations sont sur le MÊME shard
      // → Transaction locale, pas de 2PC nécessaire

      const invoice = await db.invoices.insertOne(
        { tenantId, customerId, ...invoiceData },
        { session }
      );

      await db.payments.insertOne(
        {
          tenantId,
          invoiceId: invoice.insertedId,
          amount: invoiceData.total,
          status: 'pending'
        },
        { session }
      );

      await db.customers.updateOne(
        { tenantId, customerId },
        { $inc: { outstandingBalance: invoiceData.total } },
        { session }
      );
    });

    // Latence : ~20-30ms (transaction locale)
    // vs ~200-300ms pour transaction distribuée

  } finally {
    await session.endSession();
  }
}
```

### Pattern 2 : Saga pattern pour orchestration longue durée

Pour les processus métier complexes nécessitant plusieurs étapes, le **Saga pattern** est souvent préférable aux transactions distribuées.

```javascript
// SAGA Pattern : Alternative aux transactions distribuées longues

class OrderSaga {
  constructor(db) {
    this.db = db;
  }

  async executeOrderSaga(orderData) {
    const sagaId = new ObjectId();

    // Journal de la saga
    await this.db.sagas.insertOne({
      _id: sagaId,
      type: 'order_creation',
      status: 'started',
      steps: [],
      data: orderData,
      startedAt: new Date()
    });

    try {
      // ── Étape 1 : Réserver le stock
      const stockReservation = await this.reserveStock(orderData.items);
      await this.recordStep(sagaId, 'stock_reserved', stockReservation);

      // ── Étape 2 : Créer la commande
      const order = await this.createOrder(orderData);
      await this.recordStep(sagaId, 'order_created', order);

      // ── Étape 3 : Traiter le paiement
      const payment = await this.processPayment(order);
      await this.recordStep(sagaId, 'payment_processed', payment);

      // ── Étape 4 : Confirmer la commande
      await this.confirmOrder(order._id);
      await this.recordStep(sagaId, 'order_confirmed');

      // Saga complétée
      await this.db.sagas.updateOne(
        { _id: sagaId },
        { $set: { status: 'completed', completedAt: new Date() } }
      );

      return { success: true, orderId: order._id };

    } catch (error) {
      console.error('Saga échouée:', error);

      // Compensation : Annuler les étapes précédentes
      await this.compensate(sagaId);

      return { success: false, error: error.message };
    }
  }

  async compensate(sagaId) {
    const saga = await this.db.sagas.findOne({ _id: sagaId });

    // Inverser les étapes dans l'ordre inverse
    for (let i = saga.steps.length - 1; i >= 0; i--) {
      const step = saga.steps[i];

      try {
        switch (step.name) {
          case 'stock_reserved':
            await this.releaseStock(step.data);
            break;
          case 'order_created':
            await this.cancelOrder(step.data._id);
            break;
          case 'payment_processed':
            await this.refundPayment(step.data._id);
            break;
        }

        await this.recordStep(sagaId, `${step.name}_compensated`);

      } catch (compensationError) {
        console.error('Échec de compensation:', compensationError);
        // Alerter l'équipe ops pour intervention manuelle
        await this.alertOps(sagaId, step, compensationError);
      }
    }

    await this.db.sagas.updateOne(
      { _id: sagaId },
      { $set: { status: 'compensated', compensatedAt: new Date() } }
    );
  }

  async recordStep(sagaId, stepName, data = null) {
    await this.db.sagas.updateOne(
      { _id: sagaId },
      {
        $push: {
          steps: {
            name: stepName,
            data,
            timestamp: new Date()
          }
        }
      }
    );
  }

  // Méthodes d'étapes individuelles (transactions locales)

  async reserveStock(items) {
    // Transaction locale sur le shard "products"
    const session = this.db.client.startSession();
    try {
      return await session.withTransaction(async () => {
        const reservations = [];
        for (const item of items) {
          const result = await this.db.products.findOneAndUpdate(
            { _id: item.productId, stock: { $gte: item.quantity } },
            { $inc: { stock: -item.quantity, reserved: item.quantity } },
            { session, returnDocument: 'after' }
          );
          if (!result.value) {
            throw new Error(`Stock insuffisant: ${item.productId}`);
          }
          reservations.push(result.value);
        }
        return reservations;
      });
    } finally {
      await session.endSession();
    }
  }

  async createOrder(orderData) {
    // Transaction locale sur le shard "orders"
    return await this.db.orders.insertOne({
      ...orderData,
      status: 'pending',
      createdAt: new Date()
    });
  }

  async processPayment(order) {
    // Transaction locale sur le shard "payments"
    const session = this.db.client.startSession();
    try {
      return await session.withTransaction(async () => {
        // Logique de paiement
        return { paymentId: new ObjectId(), status: 'completed' };
      });
    } finally {
      await session.endSession();
    }
  }

  // Méthodes de compensation

  async releaseStock(reservations) {
    for (const reservation of reservations) {
      await this.db.products.updateOne(
        { _id: reservation._id },
        { $inc: { stock: reservation.quantity, reserved: -reservation.quantity } }
      );
    }
  }

  async cancelOrder(orderId) {
    await this.db.orders.updateOne(
      { _id: orderId },
      { $set: { status: 'cancelled', cancelledAt: new Date() } }
    );
  }
}

// Avantages du Saga :
// - Chaque étape est une transaction locale (rapide)
// - Pas de verrous longue durée
// - Résilience aux pannes (compensation)
// - Traçabilité complète
//
// Inconvénients :
// - Cohérence éventuelle entre les étapes
// - Complexité de la logique de compensation
// - États intermédiaires visibles
```

### Pattern 3 : Versioning optimiste pour éviter les transactions

```javascript
// Éviter les transactions distribuées avec versioning optimiste

async function updateProductWithOptimisticLocking(productId, updates) {
  const maxRetries = 5;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    // Lire la version actuelle
    const product = await db.products.findOne({ _id: productId });

    if (!product) {
      throw new Error('Produit introuvable');
    }

    const currentVersion = product.version || 0;

    // Tenter la mise à jour SEULEMENT si la version n'a pas changé
    const result = await db.products.updateOne(
      {
        _id: productId,
        version: currentVersion  // Condition de version
      },
      {
        $set: updates,
        $inc: { version: 1 }  // Incrémenter la version
      }
    );

    if (result.modifiedCount === 1) {
      // Succès
      return { success: true, newVersion: currentVersion + 1 };
    }

    // Conflit de version - retry
    console.log(`Conflit de version (tentative ${attempt + 1}), retry...`);
    await new Promise(resolve => setTimeout(resolve, 10 * Math.pow(2, attempt)));
  }

  throw new Error('Trop de conflits de version');
}

// Pas de transaction nécessaire
// Pas de verrous
// Performance optimale
// Trade-off : Risque de retry en cas de contention élevée
```

## Monitoring et debugging

### Métriques essentielles

```javascript
// Monitoring des transactions distribuées

async function getTransactionMetrics() {
  const serverStatus = await db.adminCommand({ serverStatus: 1 });

  const txnMetrics = {
    // Transactions actives
    active: serverStatus.transactions.currentActive || 0,

    // Transactions en préparation
    inPrepare: serverStatus.transactions.currentPrepared || 0,

    // Total démarré
    totalStarted: serverStatus.transactions.totalStarted || 0,

    // Total commité
    totalCommitted: serverStatus.transactions.totalCommitted || 0,

    // Total aborté
    totalAborted: serverStatus.transactions.totalAborted || 0,

    // Taux d'échec
    failureRate: serverStatus.transactions.totalAborted /
                 serverStatus.transactions.totalStarted,

    // Transactions en préparation trop longtemps
    oldestPreparedTxn: serverStatus.transactions.oldestActiveOplogEntryTimestamp
  };

  return txnMetrics;
}

// Alertes à configurer
async function checkTransactionHealth() {
  const metrics = await getTransactionMetrics();

  // Alerte 1 : Trop de transactions actives
  if (metrics.active > 100) {
    console.error('⚠️  Trop de transactions actives:', metrics.active);
  }

  // Alerte 2 : Taux d'échec élevé
  if (metrics.failureRate > 0.1) {
    console.error('⚠️  Taux d'échec élevé:', (metrics.failureRate * 100).toFixed(2) + '%');
  }

  // Alerte 3 : Transactions bloquées en préparation
  if (metrics.inPrepare > 10) {
    console.error('⚠️  Transactions bloquées en prepare:', metrics.inPrepare);
  }
}
```

## Recommandations finales

### Quand utiliser les transactions distribuées

**OUI** - Utilisez les transactions distribuées quand :

```
✅ Cohérence stricte OBLIGATOIRE (finance, santé, légal)
✅ Volume faible à modéré (< 1000 transactions/sec)
✅ Latence acceptable (100-500ms)
✅ Opérations critiques peu fréquentes
✅ Alternative (Saga, eventual consistency) trop complexe
```

**NON** - Évitez les transactions distribuées quand :

```
❌ Volume élevé (> 5000 ops/sec)
❌ Latence critique (< 50ms requise)
❌ Opérations longues (> 10 secondes)
❌ Beaucoup de shards participants (> 3)
❌ Alternative viable existe (Saga, 2PC applicatif)
```

### Checklist de validation

```markdown
## Checklist : Transaction distribuée

### Avant implémentation
- [ ] Les garanties ACID sont-elles vraiment nécessaires ?
- [ ] L'alternative Saga a-t-elle été évaluée ?
- [ ] Le schéma de sharding est-il optimisé pour réduire les transactions multi-shards ?
- [ ] La latence de 100-500ms est-elle acceptable ?

### Configuration
- [ ] Write Concern: w:majority configuré
- [ ] Read Concern: snapshot configuré
- [ ] maxCommitTimeMS défini (30-60 secondes)
- [ ] Retry logic implémenté

### Monitoring
- [ ] Métriques de transaction configurées
- [ ] Alertes sur taux d'échec > 5%
- [ ] Alertes sur latence P99 > seuil
- [ ] Dashboard de santé transactionnelle

### Tests
- [ ] Test de charge avec transactions concurrentes
- [ ] Test de panne d'un shard pendant transaction
- [ ] Test de timeout et compensation
- [ ] Test de rollback automatique
```

## Conclusion

Les transactions distribuées dans MongoDB sont une fonctionnalité puissante qui apporte les garanties ACID dans un environnement shardé. Cependant, elles viennent avec des **coûts significatifs** :

- **Performance** : 4-6x plus lent qu'une transaction locale
- **Complexité** : Protocole 2PC, gestion des pannes
- **Limitations** : Durée, taille, nombre d'opérations

**Principe directeur** : Utilisez les transactions distribuées avec parcimonie, pour les opérations véritablement critiques où l'atomicité est non négociable. Pour le reste, privilégiez les patterns alternatifs comme Saga ou l'eventual consistency.

L'architecture optimale combine judicieusement :
- **Transactions locales** (même shard) pour les opérations courantes
- **Transactions distribuées** pour les opérations critiques rares
- **Saga pattern** pour les orchestrations complexes
- **Eventual consistency** pour les cas non critiques

---

**Points clés à retenir** :

- Transactions distribuées utilisent le protocole 2PC (Two-Phase Commit)
- Latence typique : 100-500ms (4-6x plus lent que local)
- Throughput réduit : ~500-1000 tx/sec maximum
- Optimiser le sharding pour minimiser les transactions multi-shards
- Saga pattern souvent préférable pour processus longue durée
- Toujours implémenter retry logic avec backoff exponentiel
- Monitorer activement : taux d'échec, latence, transactions bloquées

⏭️ [Limites et considérations de performance](/08-transactions/06-limites-considerations-performance.md)
