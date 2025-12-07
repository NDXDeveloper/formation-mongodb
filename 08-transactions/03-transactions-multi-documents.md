🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.3 Transactions Multi-Documents

## Introduction

L'introduction des transactions multi-documents dans MongoDB 4.0 (juin 2018) a marqué un tournant majeur dans l'évolution de MongoDB, représentant l'une des fonctionnalités les plus attendues et les plus débattues de l'histoire de la base de données. Cette capacité a transformé MongoDB d'une base de données NoSQL privilégiant la performance et la scalabilité au détriment de certaines garanties ACID, en un système capable de rivaliser avec les bases relationnelles traditionnelles en termes de garanties transactionnelles, tout en conservant ses avantages natifs de flexibilité et de scalabilité.

Comprendre les transactions multi-documents nécessite d'appréhender non seulement leur utilisation pratique, mais aussi leur coût architectural, leurs limites intrinsèques, et surtout, de savoir quand leur utilisation est véritablement justifiée.

## Évolution Historique et Contexte

### La Genèse : Pourquoi MongoDB N'avait Pas de Transactions Multi-Documents

MongoDB a été conçu initialement (2009) avec une philosophie claire : **privilégier la scalabilité horizontale et la performance au détriment de certaines garanties transactionnelles**. Ce n'était pas un défaut de conception, mais un choix délibéré basé sur plusieurs constats :

```
Philosophie Originale MongoDB (2009-2018)
═══════════════════════════════════════════════════════════

Principes de Conception :
1. Document comme unité atomique
   → Modéliser pour que les opérations atomiques soient mono-document

2. Dénormalisation encouragée
   → Embarquer les données liées dans un seul document

3. Scalabilité horizontale native
   → Sharding sans coordination transactionnelle coûteuse

4. Performance avant cohérence stricte
   → Eventual consistency acceptable pour la plupart des cas

Rationale :
- 80% des applications web n'ont pas besoin de transactions distribuées
- L'atomicité document suffit si le schéma est bien conçu
- Les transactions distribuées sont intrinsèquement coûteuses (CAP theorem)
- Le marché cible (startups web, applications sociales) valorise
  la performance et la facilité de scaling plus que ACID strict

Résultat :
✓ Performance exceptionnelle (50,000+ ops/sec)
✓ Scaling linéaire jusqu'à des centaines de nœuds
✓ Simplicité opérationnelle (pas de deadlocks, pas de 2PC complexe)
⚠ Nécessite discipline de modélisation
⚠ Certains cas d'usage impossibles (systèmes financiers stricts)
```

### La Révolution : MongoDB 4.0 (Juin 2018)

L'introduction des transactions multi-documents était une réponse à trois facteurs majeurs :

1. **Pression du marché entreprise** : Les grandes entreprises voulaient migrer des charges de travail critiques vers MongoDB
2. **Compétition** : D'autres bases NoSQL (comme CockroachDB) offraient déjà des transactions distribuées
3. **Maturité technique** : L'architecture interne de MongoDB (WiredTiger, Oplog) était suffisamment mature

```
Timeline des Transactions MongoDB
═══════════════════════════════════════════════════════════

MongoDB 4.0 (Juin 2018)
│
├─ Transactions multi-documents sur Replica Sets
├─ Snapshot isolation
├─ API sessions et transactions
└─ Limites : Replica Set uniquement (pas de sharded clusters)

         ▼

MongoDB 4.2 (Août 2019)
│
├─ Transactions distribuées sur Sharded Clusters
├─ Distributed transactions avec two-phase commit
├─ Amélioration des performances (~30% plus rapide)
└─ Support des transactions dans Atlas

         ▼

MongoDB 5.0 (Juillet 2021)
│
├─ Performances transactionnelles considérablement améliorées
├─ Snapshot reads sans transaction
├─ Réduction de l'overhead (metadata plus léger)
└─ Meilleure gestion de la mémoire

         ▼

MongoDB 6.0+ (2022-2024)
│
├─ Optimisations continues
├─ Transactions plus longues supportées
├─ Meilleure gestion des conflits
└─ Réduction du write amplification

Impact :
────────────────────────────────────────────────────────────
MongoDB 4.0+ permet maintenant des cas d'usage impossibles
auparavant, tout en conservant ses avantages de scalabilité.

Le système offre un spectre complet :
- Performance pure : atomicité document
- Cohérence stricte : transactions multi-documents
- Compromis : configurable selon les besoins
```

## Architecture des Transactions Multi-Documents

### Composants Fondamentaux

Les transactions multi-documents dans MongoDB reposent sur plusieurs mécanismes architecturaux sophistiqués :

```
Architecture Transactionnelle MongoDB
═══════════════════════════════════════════════════════════

┌─────────────────────────────────────────────────────────┐
│                     Client Application                  │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ Driver Connection
                 ▼
┌────────────────────────────────────────────────────────┐
│                  Session Management                    │
│  ┌────────────────────────────────────────────────┐    │
│  │  Session State                                 │    │
│  │  - Transaction ID (txnNumber)                  │    │
│  │  - Read timestamp (snapshot)                   │    │
│  │  - Write history                               │    │
│  │  - Transaction options (readConcern, etc.)     │    │
│  └────────────────────────────────────────────────┘    │
└────────────────┬───────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│              Transaction Coordinator                   │
│                (sur mongos ou primary)                 │
│  ┌────────────────────────────────────────────────┐    │
│  │  - Track active transactions                   │    │
│  │  - Manage transaction lifecycle                │    │
│  │  - Coordinate commit/abort                     │    │
│  │  - Handle two-phase commit (sharded)           │    │
│  └────────────────────────────────────────────────┘    │
└────────────────┬───────────────────────────────────────┘
                 │
                 ▼
┌────────────────────────────────────────────────────────┐
│                   Storage Layer                        │
│                   (WiredTiger)                         │
│  ┌────────────────────────────────────────────────┐    │
│  │  MVCC Engine                                   │    │
│  │  - Document versions                           │    │
│  │  - Snapshot management                         │    │
│  │  - Conflict detection                          │    │
│  └────────────────────────────────────────────────┘    │
│                                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │  Transaction Log (Oplog)                       │    │
│  │  - applyOps entries                            │    │
│  │  - Transaction boundaries (start/commit)       │    │
│  │  - Abort markers                               │    │
│  └────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────┘
```

### Sessions : Le Fondement des Transactions

Toute transaction multi-documents dans MongoDB nécessite une **session** :

```javascript
// Anatomie d'une session MongoDB

const session = client.startSession({
    // Options de la session
    causalConsistency: true,        // Garantit read-your-writes
    defaultTransactionOptions: {
        readConcern: { level: "snapshot" },
        writeConcern: { w: "majority" },
        readPreference: "primary"
    }
});

// Chaque session a un identifiant unique
console.log(session.id);
// Exemple: { id: Binary(...) }

// La session maintient l'état transactionnel :
// - Transaction number (txnNumber)
// - Cluster time (timestamp logique)
// - Transaction state (started, committed, aborted)
// - Read timestamp (pour snapshot isolation)

// Timeline d'une session :

// T=0ms : Création de la session
// ├─ Allocation d'un session ID unique
// ├─ Initialisation de l'état transactionnel
// └─ Enregistrement côté serveur (cache des sessions)

// T=10ms : Début de transaction
session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
});
// ├─ txnNumber incrémenté
// ├─ Snapshot timestamp capturé (lecture cohérente)
// ├─ Transaction state → "in progress"
// └─ Toutes les opérations suivantes utilisent ce snapshot

// T=20ms-100ms : Opérations transactionnelles
await collection1.updateOne({...}, {...}, { session });
await collection2.insertOne({...}, { session });
await collection3.deleteOne({...}, { session });
// ├─ Chaque opération porte le session ID et txnNumber
// ├─ Modifications buffered (pas encore visibles)
// ├─ Locks acquis (document-level)
// └─ Détection de conflits (WriteConflict si collision)

// T=150ms : Commit
await session.commitTransaction();
// ├─ Validation finale (pas de conflits)
// ├─ Écriture dans l'oplog (applyOps entry atomique)
// ├─ Application des modifications (visibles)
// ├─ Relâchement des locks
// └─ Transaction state → "committed"

// T=160ms : Fin de session
await session.endSession();
// ├─ Nettoyage de l'état transactionnel
// └─ Libération des ressources

// Propriétés clés de la session :
// 1. Causal Consistency : Garantit l'ordre des opérations
// 2. Snapshot Isolation : Vue cohérente des données
// 3. Server-side tracking : Le serveur sait quelles opérations
//    font partie de quelle transaction
```

### Oplog et applyOps : Le Cœur de l'Atomicité

Dans MongoDB, l'oplog (operations log) est le mécanisme de réplication. Pour les transactions multi-documents, un mécanisme spécial `applyOps` garantit l'atomicité :

```javascript
// Structure de l'Oplog pour une transaction

// Transaction simple (sans transaction)
// Chaque opération génère une entrée oplog séparée :
{
    ts: Timestamp(1701963000, 1),
    t: NumberLong(1),
    op: "u",  // update
    ns: "mydb.collection1",
    o: { $set: { status: "active" } },
    o2: { _id: ObjectId("...") }
}

{
    ts: Timestamp(1701963000, 2),
    t: NumberLong(1),
    op: "i",  // insert
    ns: "mydb.collection2",
    o: { _id: ObjectId("..."), data: "..." }
}

// ⚠ Problème : Entre ces deux entrées, un crash peut laisser
// la base dans un état incohérent (première op appliquée, pas la seconde)

// ═══════════════════════════════════════════════════════════════
// Transaction multi-documents
// TOUTES les opérations dans UNE SEULE entrée applyOps :
// ═══════════════════════════════════════════════════════════════

{
    ts: Timestamp(1701963000, 1),
    t: NumberLong(1),
    op: "c",  // command
    ns: "mydb.$cmd",
    o: {
        applyOps: [
            // ▼ Toutes les opérations de la transaction
            {
                op: "u",
                ns: "mydb.collection1",
                o: { $set: { status: "active" } },
                o2: { _id: ObjectId("...") }
            },
            {
                op: "i",
                ns: "mydb.collection2",
                o: { _id: ObjectId("..."), data: "..." }
            },
            {
                op: "d",
                ns: "mydb.collection3",
                o: { _id: ObjectId("...") }
            }
        ],
        // Métadonnées transactionnelles
        lsid: { id: UUID("...") },        // Session ID
        txnNumber: NumberLong(1),
        stmtIds: [0, 1, 2],               // Statement IDs
        prevOpTime: { ts: ..., t: ... }   // Previous op timestamp
    },
    wall: ISODate("2024-12-07T10:30:00.000Z")
}

// Garanties de applyOps :
// ✓ Atomicité : Toutes les ops appliquées ensemble ou aucune
// ✓ Ordre : Les ops sont appliquées dans l'ordre défini
// ✓ Réplication : L'entrée applyOps est répliquée atomiquement
// ✓ Durabilité : Une fois dans l'oplog, survit aux crashes
// ✓ Idempotence : Peut être rejouée sans effet de bord

// Mécanisme de réplication avec transactions :

// Primary (écriture)                Secondary (réplication)
// ══════════════════                ════════════════════════
//
// T=0 : BEGIN transaction
// T=10: operation 1 (buffered)
// T=20: operation 2 (buffered)
// T=30: operation 3 (buffered)
// T=40: COMMIT
//       ↓
//       Écriture applyOps dans oplog
//       ↓
//       ────────────────────────────→ T=45: Lecture oplog
//                                     T=46: Apply applyOps
//                                           (atomiquement)
//                                     T=47: Ops appliquées
//
// Si le secondary crash pendant l'apply :
// - Au redémarrage, rejoue le applyOps complet
// - Idempotence garantit le résultat correct

// Limite importante de applyOps :
// La taille de l'entrée applyOps est limitée à 16 MB
// (même limite qu'un document)
//
// Implication :
// Une transaction avec trop d'opérations ou trop de données
// peut dépasser cette limite et échouer
//
// Exemple : Transaction avec 10,000 insertions de documents de 2 KB
// → 20 MB de données → ÉCHEC
//
// Solution : Diviser en plusieurs transactions ou utiliser
// des opérations bulk moins nombreuses
```

### Snapshot Isolation : Le Mécanisme de Lecture Cohérente

```javascript
// Snapshot Isolation garantit que toutes les lectures dans une
// transaction voient un état cohérent de la base de données

// État initial de la base :
// Document A : { value: 100 }
// Document B : { value: 200 }

// ═══════════════════════════════════════════════════════════════
// Timeline avec Snapshot Isolation
// ═══════════════════════════════════════════════════════════════

// T1 : Transaction 1 (lecture)
const session1 = client.startSession();
session1.startTransaction({
    readConcern: { level: "snapshot" }
});

// T=0ms : Snapshot créé au timestamp T0
// session1 voit : A=100, B=200

const docA_t1_read1 = await collectionA.findOne(
    { _id: "A" },
    { session: session1 }
);
console.log(docA_t1_read1.value);  // 100

// ═══════════════════════════════════════════════════════════════

// T2 : Pendant ce temps, Transaction 2 modifie les documents
const session2 = client.startSession();
session2.startTransaction();

// T=50ms : T2 modifie A et B
await collectionA.updateOne(
    { _id: "A" },
    { $set: { value: 150 } },
    { session: session2 }
);

await collectionB.updateOne(
    { _id: "B" },
    { $set: { value: 250 } },
    { session: session2 }
);

// T=100ms : T2 committe
await session2.commitTransaction();
await session2.endSession();

// État après T2 commit :
// Document A : { value: 150 }
// Document B : { value: 250 }

// ═══════════════════════════════════════════════════════════════

// T1 continue (toujours sur son snapshot T0)

// T=150ms : T1 relit A
const docA_t1_read2 = await collectionA.findOne(
    { _id: "A" },
    { session: session1 }
);
console.log(docA_t1_read2.value);  // 100 (PAS 150!)

// T=200ms : T1 lit B
const docB_t1_read = await collectionB.findOne(
    { _id: "B" },
    { session: session1 }
);
console.log(docB_t1_read.value);  // 200 (PAS 250!)

// T=250ms : T1 committe
await session1.commitTransaction();
await session1.endSession();

// Garantie Snapshot Isolation :
// ✓ T1 a vu un état cohérent (snapshot à T0)
// ✓ Les modifications de T2 n'ont pas "pollué" la vue de T1
// ✓ Pas de lecture "sale" (dirty read)
// ✓ Pas de lecture non répétable (non-repeatable read)
// ✓ Pas de lecture fantôme (phantom read)

// Mécanisme interne (MVCC dans WiredTiger) :

// Cache WiredTiger maintient plusieurs versions :
// ┌────────────────────────────────────────┐
// │ Document A                             │
// │ ├─ Version @T0: { value: 100 }         │
// │ │  [Visible pour: T1]                  │
// │ └─ Version @T100: { value: 150 }       │
// │    [Visible pour: nouvelles lectures]  │
// └────────────────────────────────────────┘

// Quand T1 lit A à T=150ms :
// - WiredTiger cherche la version <= snapshot de T1 (T0)
// - Trouve version @T0 : { value: 100 }
// - Retourne cette version (pas la version @T100)

// Nettoyage (eviction) :
// Une fois T1 terminée, la version @T0 peut être supprimée
// si aucune autre transaction ne l'utilise
```

## Différences Fondamentales : Replica Set vs Sharded Cluster

Les transactions multi-documents se comportent différemment selon l'architecture :

### Transactions sur Replica Set

```javascript
// Architecture : 1 Primary + N Secondaries (même replica set)

// ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
// │   Primary    │  │  Secondary   │  │  Secondary   │
// │              │  │              │  │              │
// │  Collection  │  │  Réplication │  │  Réplication │
// │  A, B, C     │──┤  oplog       │──┤  oplog       │
// │              │  │              │  │              │
// └──────────────┘  └──────────────┘  └──────────────┘

const session = client.startSession();

try {
    session.startTransaction({
        readConcern: { level: "snapshot" },
        writeConcern: { w: "majority" },
        readPreference: "primary"
    });

    // Toutes les opérations sont sur le même replica set
    await collectionA.updateOne({...}, {...}, { session });
    await collectionB.insertOne({...}, { session });
    await collectionC.deleteOne({...}, { session });

    await session.commitTransaction();

    // Mécanisme interne simplifié :
    // ─────────────────────────────────────────────────────
    // 1. Opérations exécutées sur le Primary
    // 2. Modifications buffered en mémoire
    // 3. COMMIT :
    //    a) Écriture applyOps dans l'oplog du Primary
    //    b) Réplication vers Secondaries (selon writeConcern)
    //    c) Application des modifications
    // 4. Pas de coordination distribuée nécessaire

} catch (error) {
    await session.abortTransaction();
    throw error;
} finally {
    await session.endSession();
}

// Caractéristiques Replica Set :
// ✓ Relativement simple (pas de 2PC)
// ✓ Performance acceptable (10-30ms selon writeConcern)
// ✓ Coordination locale (même instance logique)
// ✓ Latence prédictible
// ⚠ Limité à un seul replica set
// ⚠ Pas de scaling horizontal des transactions
```

### Transactions sur Sharded Cluster

```javascript
// Architecture : Multiple shards + Config servers + Mongos

// ┌─────────────────────────────────────────────────────┐
// │                     Mongos                          │
// │              (Query Router + Coordinator)           │
// └───────────────┬──────────────┬──────────────────────┘
//                 │              │
//       ┌─────────┴──────┐   ┌───┴──────────┐
//       │                │   │              │
//   ┌───▼──────┐    ┌───▼──────┐    ┌────▼─────┐
//   │ Shard 1  │    │ Shard 2  │    │ Shard 3  │
//   │ (RS)     │    │ (RS)     │    │ (RS)     │
//   │ Docs A-H │    │ Docs I-P │    │ Docs Q-Z │
//   └──────────┘    └──────────┘    └──────────┘

const session = client.startSession();

try {
    session.startTransaction({
        readConcern: { level: "snapshot" },
        writeConcern: { w: "majority" },
        readPreference: "primary"
    });

    // Ces opérations touchent différents shards
    await collection.updateOne(
        { userId: "Alice" },    // → Shard 1 (A-H)
        { $inc: { balance: -100 } },
        { session }
    );

    await collection.updateOne(
        { userId: "Zoe" },      // → Shard 3 (Q-Z)
        { $inc: { balance: 100 } },
        { session }
    );

    await session.commitTransaction();

    // Mécanisme interne (Two-Phase Commit) :
    // ───────────────────────────────────────────────────────
    // PHASE 1 : PREPARE
    // ───────────────────────────────────────────────────────
    // T=0ms : Mongos (coordinator) envoie PREPARE à tous les shards
    //
    // T=10ms : Shard 1 reçoit PREPARE
    //   ├─ Applique les modifications localement (buffer)
    //   ├─ Acquiert les locks
    //   ├─ Écrit prepare dans son oplog local
    //   └─ Répond "PREPARED" au coordinator
    //
    // T=15ms : Shard 3 reçoit PREPARE
    //   ├─ Applique les modifications localement (buffer)
    //   ├─ Acquiert les locks
    //   ├─ Écrit prepare dans son oplog local
    //   └─ Répond "PREPARED" au coordinator
    //
    // T=20ms : Coordinator reçoit tous les PREPARED
    //   └─ Décision : COMMIT (tous OK) ou ABORT (au moins un échec)
    //
    // ───────────────────────────────────────────────────────
    // PHASE 2 : COMMIT
    // ───────────────────────────────────────────────────────
    // T=25ms : Coordinator écrit la décision dans config servers
    //   └─ Garantit la durabilité de la décision
    //
    // T=30ms : Coordinator envoie COMMIT à tous les shards
    //
    // T=35ms : Shard 1 reçoit COMMIT
    //   ├─ Applique définitivement les modifications
    //   ├─ Écrit commit dans l'oplog
    //   ├─ Relâche les locks
    //   └─ Répond "COMMITTED"
    //
    // T=40ms : Shard 3 reçoit COMMIT
    //   ├─ Applique définitivement les modifications
    //   ├─ Écrit commit dans l'oplog
    //   ├─ Relâche les locks
    //   └─ Répond "COMMITTED"
    //
    // T=45ms : Transaction complète

} catch (error) {
    await session.abortTransaction();

    // En cas d'ABORT, même mécanisme 2PC :
    // - Coordinator envoie ABORT à tous les shards
    // - Chaque shard annule ses modifications buffered
    // - Locks relâchés

    throw error;
} finally {
    await session.endSession();
}

// Caractéristiques Sharded Cluster :
// ✓ Permet transactions cross-shard
// ✓ ACID complet même distribué
// ⚠ Complexité élevée (2PC)
// ⚠ Performance dégradée (50-200ms typique)
// ⚠ Plus de risques d'échec (network, timeouts)
// ⚠ Overhead de coordination significatif

// Points de défaillance additionnels :
// 1. Coordinator (mongos) peut crasher
//    → Récupération automatique mais latence accrue
// 2. Shard peut être inaccessible pendant PREPARE
//    → Timeout et abort de la transaction
// 3. Network partition entre prepare et commit
//    → Shards en état "prepared" (indécis)
//    → Résolution nécessaire au recovery
```

## Coûts et Compromis des Transactions Multi-Documents

### Impact sur la Performance

Les transactions multi-documents ont un coût mesurable et significatif :

```javascript
// Benchmark comparatif : Même opération, différentes approches

// ═══════════════════════════════════════════════════════════════
// Scénario 1 : Sans transaction (atomicité document uniquement)
// ═══════════════════════════════════════════════════════════════

// Document bien modélisé (tout est imbriqué)
{
    orderId: "O001",
    customerId: "C001",
    items: [
        { sku: "P001", qty: 2, price: 50 },
        { sku: "P002", qty: 1, price: 100 }
    ],
    total: 200,
    status: "pending"
}

await orders.updateOne(
    { orderId: "O001" },
    {
        $set: {
            status: "confirmed",
            confirmedAt: new Date()
        },
        $inc: { version: 1 }
    }
);

// Métriques mesurées (Replica Set 3 nœuds, w:1) :
// - Latence P50 : 2.5ms
// - Latence P95 : 6ms
// - Latence P99 : 12ms
// - Throughput : 40,000 ops/sec
// - CPU : 25%
// - Échecs : ~0.01% (network temporaire)

// ═══════════════════════════════════════════════════════════════
// Scénario 2 : Avec transaction (modélisation relationnelle)
// ═══════════════════════════════════════════════════════════════

// Collections séparées
// orders: { orderId, customerId, total, status }
// order_items: { orderItemId, orderId, sku, qty, price }

const session = client.startSession();
session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
});

try {
    await orders.updateOne(
        { orderId: "O001" },
        { $set: { status: "confirmed", confirmedAt: new Date() } },
        { session }
    );

    await orderItems.updateMany(
        { orderId: "O001" },
        { $set: { confirmed: true } },
        { session }
    );

    await session.commitTransaction();
} finally {
    await session.endSession();
}

// Métriques mesurées (Replica Set 3 nœuds, w:majority) :
// - Latence P50 : 28ms
// - Latence P95 : 65ms
// - Latence P99 : 120ms
// - Throughput : 3,500 ops/sec
// - CPU : 45%
// - Échecs : ~2% (WriteConflict + timeouts)

// Analyse :
// ────────────────────────────────────────────────────────────
// Overhead transactionnel :
// - Latence P50 : 11x plus lent (2.5ms → 28ms)
// - Throughput : 11x plus faible (40k → 3.5k ops/sec)
// - CPU : 1.8x plus élevé
// - Taux d'échec : 200x plus élevé
//
// Causes :
// 1. Coordination de session (overhead de bookkeeping)
// 2. Snapshot isolation (MVCC overhead)
// 3. WriteConcern majority (attente réplication)
// 4. Conflict detection (WriteConflict possible)
// 5. Deux-phase commit si sharded (non applicable ici)

// ═══════════════════════════════════════════════════════════════
// Scénario 3 : Transaction sur Sharded Cluster (pire cas)
// ═══════════════════════════════════════════════════════════════

// Même transaction, mais sur un cluster shardé
// où les documents sont sur différents shards

// Métriques mesurées (Sharded cluster 3 shards, LAN) :
// - Latence P50 : 85ms
// - Latence P95 : 180ms
// - Latence P99 : 350ms
// - Throughput : 800 ops/sec
// - CPU : 60% (mongos)
// - Échecs : ~5% (coordination + network)

// Métriques mesurées (Sharded cluster multi-région) :
// - Latence P50 : 250ms
// - Latence P95 : 600ms
// - Latence P99 : 1200ms
// - Throughput : 200 ops/sec
// - Échecs : ~8%

// Analyse :
// ────────────────────────────────────────────────────────────
// Overhead additionnel sharded :
// - 2PC coordination (50-100ms)
// - Network round-trips multiples
// - Config servers writes
// - Plus de points de défaillance

// Comparaison finale :
// ────────────────────────────────────────────────────────────
// Sans transaction (document atomique) : 2.5ms baseline
// Transaction Replica Set : 28ms (11x overhead)
// Transaction Sharded LAN : 85ms (34x overhead)
// Transaction Sharded multi-région : 250ms (100x overhead)
```

### Consommation de Ressources

```javascript
// Impact mémoire et CPU des transactions

// Configuration de test :
// - 1000 transactions concurrentes
// - Chaque transaction : 10 opérations
// - Duration moyenne : 500ms par transaction

// ═══════════════════════════════════════════════════════════════
// Sans transactions
// ═══════════════════════════════════════════════════════════════

// Consommation mesurée :
// - RAM : 2 GB (WiredTiger cache)
// - CPU : 35% (4 cores)
// - Connections : 1000 (une par thread applicatif)
// - Disk I/O : 50 MB/s (writes)

// ═══════════════════════════════════════════════════════════════
// Avec transactions
// ═══════════════════════════════════════════════════════════════

// Consommation mesurée :
// - RAM : 3.8 GB (+90%)
//   ├─ Session state : +600 MB
//   ├─ Transaction metadata : +400 MB
//   ├─ MVCC versions : +800 MB
//   └─ WiredTiger cache : 2 GB (inchangé)
//
// - CPU : 58% (+65%)
//   ├─ Conflict detection : +10%
//   ├─ Snapshot management : +8%
//   └─ Transaction coordination : +7%
//
// - Connections : 1000 (inchangé)
//   └─ Mais chaque connection maintient une session active
//
// - Disk I/O : 75 MB/s (+50%)
//   ├─ Oplog entries plus grandes (applyOps)
//   └─ Transaction metadata writes

// Facteurs aggravants :
// ────────────────────────────────────────────────────────────
// 1. Transactions longues (> 1s)
//    → Accumulation de versions MVCC
//    → Pression mémoire accrue
//
// 2. Haute contention (même documents modifiés)
//    → WriteConflict fréquents
//    → Retries = CPU et latence
//
// 3. Large transactions (1000+ opérations)
//    → applyOps entry > 16 MB → ÉCHEC
//    → Ou très proche limite → performance dégradée
//
// 4. Transactions abandonnées (non endSession)
//    → Fuite de ressources
//    → Sessions zombies

// Recommandations :
// ────────────────────────────────────────────────────────────
// ✓ Garder les transactions courtes (< 100ms idéal)
// ✓ Limiter le nombre d'opérations (< 100 par transaction)
// ✓ Toujours appeler endSession() (try/finally)
// ✓ Configurer des timeouts appropriés
// ✓ Monitorer la consommation mémoire
```

## Garanties et Limitations

### Ce que les Transactions Multi-Documents Garantissent

```javascript
// ✓ GARANTIES FOURNIES

// 1. Atomicité complète
const session = client.startSession();
session.startTransaction();

await collection1.insertOne({ data: "A" }, { session });
await collection2.insertOne({ data: "B" }, { session });
await collection3.insertOne({ data: "C" }, { session });

await session.commitTransaction();
// Garantie : Soit A, B, et C sont insérés
//            Soit aucun n'est inséré
//            Jamais A sans B, ou B sans C, etc.

// 2. Snapshot Isolation
session.startTransaction({ readConcern: { level: "snapshot" } });

const read1 = await collection.findOne({ _id: 1 }, { session });
// Snapshot capturé ici

// [D'autres transactions modifient le doc entre temps]

const read2 = await collection.findOne({ _id: 1 }, { session });
// Garantie : read1 === read2 (même snapshot)

// 3. Durabilité configurable
session.startTransaction({
    writeConcern: { w: "majority", j: true }
});

await collection.insertOne({ important: "data" }, { session });
await session.commitTransaction();
// Garantie : Données persistent même si majorité des nœuds crashent

// 4. Isolation entre transactions
// T1 et T2 concurrentes ne voient pas les modifications
// non committées l'une de l'autre

// 5. Ordre des opérations préservé
session.startTransaction();
await collection.insertOne({ _id: 1 }, { session });
await collection.updateOne({ _id: 1 }, { $set: { v: 2 } }, { session });
await session.commitTransaction();
// Garantie : Insert avant update, toujours
```

### Ce que les Transactions Multi-Documents NE Garantissent PAS

```javascript
// ✗ LIMITATIONS ET EXCEPTIONS

// 1. Pas de vrai Serializable (anomalies write skew possibles)
// Exemple : Deux médecins veulent partir, invariant = au moins un présent

const session1 = client.startSession();
session1.startTransaction({ readConcern: { level: "snapshot" } });

const session2 = client.startSession();
session2.startTransaction({ readConcern: { level: "snapshot" } });

// T1 : Médecin Smith vérifie
const count1 = await doctors.countDocuments(
    { onCall: true },
    { session: session1 }
);  // count = 2

// T2 : Médecin Jones vérifie (snapshot identique)
const count2 = await doctors.countDocuments(
    { onCall: true },
    { session: session2 }
);  // count = 2

// T1 : OK, je peux partir
if (count1 > 1) {
    await doctors.updateOne(
        { doctorId: "Smith" },
        { $set: { onCall: false } },
        { session: session1 }
    );
}
await session1.commitTransaction();

// T2 : OK, je peux partir aussi
if (count2 > 1) {
    await doctors.updateOne(
        { doctorId: "Jones" },
        { $set: { onCall: false } },
        { session: session2 }
    );
}
await session2.commitTransaction();

// Résultat : count = 0 (VIOLE L'INVARIANT!)
// ⚠ MongoDB ne détecte pas cette anomalie write skew

// 2. Timeout strict (60 secondes par défaut)
session.startTransaction();

await longRunningOperation();  // Prend 70 secondes

await session.commitTransaction();
// ✗ ERREUR : Transaction aborted (timeout dépassé)

// 3. Limite de taille (16 MB pour applyOps)
session.startTransaction();

for (let i = 0; i < 100000; i++) {
    await collection.insertOne({
        data: "x".repeat(1000)  // 1 KB par document
    }, { session });
}

await session.commitTransaction();
// ✗ ERREUR : Transaction too large
// (100,000 docs × 1 KB = 100 MB > 16 MB limit)

// 4. Pas de DDL dans les transactions
session.startTransaction();

await db.createCollection("newcollection", { session });
// ✗ ERREUR : Cannot create collection in transaction

await collection.createIndex({ field: 1 }, { session });
// ✗ ERREUR : Cannot create index in transaction

// 5. Certaines opérations non supportées
session.startTransaction();

await collection.distinct("field", {}, { session });
// ✗ ERREUR : distinct not supported in transactions

await collection.aggregate([
    { $out: "outputCollection" }
], { session });
// ✗ ERREUR : $out not supported in transactions

// 6. Pas de garantie de progression (livelock possible)
// Avec haute contention, des transactions peuvent infiniment
// retry sans jamais réussir (nécessite backoff applicatif)
```

## Quand NE PAS Utiliser les Transactions Multi-Documents

### Anti-Patterns Courants

```javascript
// ❌ ANTI-PATTERN 1 : Transaction pour opération unique

const session = client.startSession();
session.startTransaction();

await collection.updateOne(
    { _id: docId },
    { $set: { status: "active" } },
    { session }
);

await session.commitTransaction();
await session.endSession();

// Problème : Overhead transactionnel inutile
// L'atomicité document suffit largement
//
// ✓ Solution : Opération simple
await collection.updateOne(
    { _id: docId },
    { $set: { status: "active" } }
);

// ❌ ANTI-PATTERN 2 : Transaction pour éviter la modélisation

// Collections séparées par "réflexe relationnel"
// users: { userId, username }
// profiles: { userId, bio }
// settings: { userId, preferences }

const session = client.startSession();
session.startTransaction();
await users.updateOne({...}, {...}, { session });
await profiles.updateOne({...}, {...}, { session });
await settings.updateOne({...}, {...}, { session });
await session.commitTransaction();

// Problème : Complexité et coût élevés
//
// ✓ Solution : Document unique bien modélisé
{
    userId: "U001",
    username: "jdupont",
    profile: { bio: "..." },
    settings: { preferences: {...} }
}

await users.updateOne(
    { userId: "U001" },
    {
        $set: {
            "profile.bio": "New bio",
            "settings.preferences.theme": "dark"
        }
    }
);

// ❌ ANTI-PATTERN 3 : Transactions longues (> 1 minute)

const session = client.startSession();
session.startTransaction();

for (let i = 0; i < 1000000; i++) {
    await collection.insertOne({ data: i }, { session });

    if (i % 10000 === 0) {
        // Traitement lourd
        await heavyComputation();
    }
}

await session.commitTransaction();
// Problème :
// - Timeout (60s)
// - Mémoire (accumulation MVCC)
// - Locks maintenus trop longtemps
//
// ✓ Solution : Batch processing sans transaction
for (let batch = 0; batch < 100; batch++) {
    const docs = [];
    for (let i = 0; i < 10000; i++) {
        docs.push({ data: batch * 10000 + i });
    }
    await collection.insertMany(docs);
}
```

## Conclusion de l'Introduction

Les transactions multi-documents représentent un ajout majeur aux capacités de MongoDB, mais elles ne doivent pas être utilisées par défaut. Leur introduction était nécessaire pour certains cas d'usage critiques, mais leur coût en termes de performance, complexité et ressources est significatif.

### Principes Directeurs

1. **L'atomicité document devrait être votre première option** : 80-90% des cas d'usage peuvent être satisfaits avec une bonne modélisation
2. **Les transactions multi-documents sont pour les 10-20% restants** : Opérations cross-document véritablement critiques
3. **Mesurer avant de déployer** : Benchmarker l'impact réel sur votre charge de travail
4. **Garder les transactions courtes** : < 100ms idéal, < 1s acceptable, > 1s problématique
5. **Préférer Replica Set à Sharded pour les transactions** : Si possible, éviter les transactions cross-shard

### Questions à se Poser

Avant d'utiliser une transaction multi-documents :

- Puis-je modéliser différemment pour utiliser l'atomicité document ?
- L'incohérence temporaire est-elle vraiment inacceptable ?
- Les bénéfices ACID justifient-ils le coût de performance ?
- Puis-je utiliser des patterns alternatifs (saga, idempotence) ?
- Mon équipe est-elle prête à gérer la complexité additionnelle ?

Dans les sections suivantes, nous explorerons les cas d'usage légitimes des transactions multi-documents, leur implémentation pratique, et les stratégies d'optimisation.

---


⏭️ [Cas d'usage et nécessité](/08-transactions/03.1-cas-usage-necessite.md)
