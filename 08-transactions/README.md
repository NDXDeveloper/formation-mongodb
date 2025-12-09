🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Transactions

## La garantie ACID dans un monde NoSQL ! 🔐

Vous maîtrisez maintenant MongoDB : modélisation, requêtes, agrégations, validation. Mais il reste une question fondamentale : **comment garantir la cohérence des données lors d'opérations complexes impliquant plusieurs documents ou collections ?** Comment s'assurer qu'un transfert bancaire débite ET crédite les comptes, sans risque d'incohérence ?

Les transactions MongoDB apportent les garanties ACID (Atomicité, Cohérence, Isolation, Durabilité) au monde NoSQL. Mais contrairement à SQL, leur utilisation nécessite une **compréhension approfondie des compromis** entre cohérence et performance. Ce chapitre va vous révéler quand, comment et surtout **pourquoi** utiliser (ou ne pas utiliser) les transactions.

## Où en sommes-nous dans votre parcours ?

Vous avez complété les chapitres 1 à 7 et vous maîtrisez maintenant :
- ✅ La modélisation des données et les patterns
- ✅ Les index et l'optimisation des performances
- ✅ Le framework d'agrégation
- ✅ La validation des schémas
- ✅ Les opérations CRUD et leur atomicité native

**Parfait !** Vous êtes maintenant prêt à comprendre les **garanties transactionnelles** et leurs implications sur l'architecture de vos applications.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- ✅ **Comprendre** les propriétés ACID dans le contexte NoSQL
- ✅ **Distinguer** atomicité mono-document et transactions multi-documents
- ✅ **Utiliser** les sessions et transactions correctement
- ✅ **Configurer** les niveaux de cohérence (Read/Write Concern)
- ✅ **Évaluer** les compromis performance vs cohérence
- ✅ **Identifier** les cas où les transactions sont nécessaires
- ✅ **Éviter** les pièges courants et anti-patterns
- ✅ **Optimiser** les transactions pour la performance
- ✅ **Appliquer** les bonnes pratiques transactionnelles

## Le paradoxe NoSQL : Performance vs Cohérence

### La promesse initiale du NoSQL

NoSQL (et MongoDB) est né avec une promesse : **sacrifier certaines garanties de cohérence pour obtenir des performances et une scalabilité exceptionnelles**.

```
Théorème CAP :
On ne peut avoir simultanément :
- Consistency (Cohérence)
- Availability (Disponibilité)  ← MongoDB privilégie ceci
- Partition tolerance (Tolérance au partitionnement)  ← Et ceci

→ MongoDB historiquement sacrifiait la cohérence stricte
```

### L'évolution : MongoDB 4.0+ avec transactions ACID

Depuis MongoDB 4.0 (2018), MongoDB offre des **transactions multi-documents** avec garanties ACID complètes :

```javascript
// Transaction ACID complète dans MongoDB
const session = db.getMongo().startSession()
session.startTransaction()

try {
    // Opération 1
    db.accounts.updateOne(
        { _id: "account1" },
        { $inc: { balance: -100 } },
        { session }
    )

    // Opération 2
    db.accounts.updateOne(
        { _id: "account2" },
        { $inc: { balance: 100 } },
        { session }
    )

    // Commit : TOUT réussit ou RIEN
    session.commitTransaction()
} catch (error) {
    // Rollback automatique en cas d'erreur
    session.abortTransaction()
    throw error
} finally {
    session.endSession()
}
```

**Mais attention :** Les transactions ont un **coût** en termes de performance et de complexité.

## Vue d'ensemble du chapitre

Ce chapitre est organisé en 7 sections qui couvrent tous les aspects des transactions :

### 🎯 Partie 1 : Fondements ACID (Sections 8.1 et 8.2)
- **8.1** : Rappel ACID et contexte NoSQL
- **8.2** : Atomicité native mono-document

### 🎯 Partie 2 : Transactions multi-documents (Section 8.3)
- **8.3.1** : Cas d'usage et nécessité
- **8.3.2** : Sessions et transactions
- **8.3.3** : Syntaxe et API
- **8.3.4** : Commit et rollback

### 🎯 Partie 3 : Niveaux de cohérence (Section 8.4)
- **8.4.1** : Read Concern (local, majority, linearizable, snapshot)
- **8.4.2** : Write Concern (w, j, wtimeout)
- **8.4.3** : Compromis performance vs cohérence

### 🎯 Partie 4 : Avancé (Sections 8.5 à 8.7)
- **8.5** : Transactions distribuées (sharded clusters)
- **8.6** : Limites et considérations de performance
- **8.7** : Bonnes pratiques

## ACID dans SQL : le modèle de référence

### Transaction SQL classique

```sql
-- Transfert bancaire en SQL
BEGIN TRANSACTION;

-- Débiter compte source
UPDATE accounts
SET balance = balance - 100
WHERE account_id = 'ACC001';

-- Créditer compte destination
UPDATE accounts
SET balance = balance + 100
WHERE account_id = 'ACC002';

-- Si tout OK : valider
COMMIT;

-- Si erreur : annuler tout
-- ROLLBACK; (automatique en cas d'erreur)
```

**Garanties SQL :**
- ✅ **A**tomicité : Tout ou rien
- ✅ **C**ohérence : État valide avant et après
- ✅ **I**solation : Transactions concurrentes isolées
- ✅ **D**urabilité : Changements persistés

**Coût :** Acceptable car SQL est conçu pour les transactions.

## Les trois niveaux d'atomicité dans MongoDB

MongoDB offre **trois niveaux** d'atomicité, chacun avec ses compromis :

### Niveau 1 : Atomicité mono-document (native, gratuite)

```javascript
// Opération atomique sur UN document
db.accounts.updateOne(
    { _id: "account1" },
    {
        $inc: { balance: -100 },
        $push: {
            transactions: {
                amount: -100,
                date: new Date(),
                type: "withdrawal"
            }
        }
    }
)

// ✅ Atomique : balance ET transactions modifiés ensemble
// ✅ Gratuit en performance
// ✅ Toujours disponible
```

**Garanties :**
- ✅ Atomique au niveau document
- ✅ Pas de surcoût
- ❌ Limité à un seul document

**Usage :** 90% des cas d'usage si bien modélisé !

### Niveau 2 : Transactions multi-documents dans un Replica Set

```javascript
// Transaction sur plusieurs documents
const session = db.getMongo().startSession()
session.startTransaction()

try {
    db.accounts.updateOne(
        { _id: "account1" },
        { $inc: { balance: -100 } },
        { session }
    )

    db.accounts.updateOne(
        { _id: "account2" },
        { $inc: { balance: 100 } },
        { session }
    )

    session.commitTransaction()
} catch (error) {
    session.abortTransaction()
} finally {
    session.endSession()
}

// ✅ Atomique sur plusieurs documents
// ⚠️ Coût en performance (10-30% plus lent)
// ⚠️ Complexité accrue
```

**Garanties :**
- ✅ ACID complet
- ✅ Plusieurs documents/collections
- ⚠️ Coût en performance
- ⚠️ Latence accrue

**Usage :** Quand vraiment nécessaire (< 10% des cas).

### Niveau 3 : Transactions distribuées (sharded cluster)

```javascript
// Transaction distribuée sur plusieurs shards
const session = db.getMongo().startSession()
session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
})

try {
    // Documents potentiellement sur des shards différents
    db.accounts.updateOne(
        { _id: "account1" },  // Peut-être shard A
        { $inc: { balance: -100 } },
        { session }
    )

    db.accounts.updateOne(
        { _id: "account2" },  // Peut-être shard B
        { $inc: { balance: 100 } },
        { session }
    )

    session.commitTransaction()
} catch (error) {
    session.abortTransaction()
} finally {
    session.endSession()
}

// ✅ Atomique même sur shards différents
// ❌ Coût significatif (30-50% plus lent)
// ❌ Complexité élevée
// ❌ Protocole 2PC (Two-Phase Commit)
```

**Garanties :**
- ✅ ACID distribué
- ✅ Cohérence globale
- ❌ Coût très élevé
- ❌ Latence importante

**Usage :** Rare, à éviter si possible.

## Exemple réaliste 1 : Système bancaire

### Approche 1 : Sans transaction (risque d'incohérence)

```javascript
// ❌ DANGER : Pas atomique entre les deux updates
async function transferMoney(fromAccount, toAccount, amount) {
    // Étape 1 : Débiter
    await db.accounts.updateOne(
        { _id: fromAccount, balance: { $gte: amount } },
        { $inc: { balance: -amount } }
    )

    // 💥 CRASH ICI = argent perdu !
    // 💥 ERREUR RÉSEAU = incohérence !

    // Étape 2 : Créditer
    await db.accounts.updateOne(
        { _id: toAccount },
        { $inc: { balance: amount } }
    )
}

// Problèmes possibles :
// 1. Crash entre les deux updates → argent débité mais pas crédité
// 2. Erreur réseau → incohérence
// 3. Pas de rollback possible
```

**Risque :** Perte de données, incohérence critique.

### Approche 2 : Avec transaction (cohérence garantie)

```javascript
// ✅ Transaction ACID : atomicité garantie
async function transferMoneySafe(fromAccount, toAccount, amount) {
    const session = db.getMongo().startSession()

    try {
        session.startTransaction({
            readConcern: { level: "snapshot" },
            writeConcern: { w: "majority" }
        })

        // Vérifier et débiter
        const debitResult = await db.accounts.updateOne(
            {
                _id: fromAccount,
                balance: { $gte: amount },
                status: "active"
            },
            {
                $inc: { balance: -amount },
                $push: {
                    transactions: {
                        type: "debit",
                        amount: -amount,
                        to: toAccount,
                        date: new Date()
                    }
                }
            },
            { session }
        )

        if (debitResult.matchedCount === 0) {
            throw new Error("Insufficient funds or inactive account")
        }

        // Créditer
        await db.accounts.updateOne(
            { _id: toAccount, status: "active" },
            {
                $inc: { balance: amount },
                $push: {
                    transactions: {
                        type: "credit",
                        amount: amount,
                        from: fromAccount,
                        date: new Date()
                    }
                }
            },
            { session }
        )

        // Enregistrer dans l'historique global
        await db.transferHistory.insertOne(
            {
                from: fromAccount,
                to: toAccount,
                amount: amount,
                date: new Date(),
                status: "completed"
            },
            { session }
        )

        // TOUT réussit
        await session.commitTransaction()
        return { success: true }

    } catch (error) {
        // RIEN ne réussit (rollback automatique)
        await session.abortTransaction()
        return { success: false, error: error.message }
    } finally {
        session.endSession()
    }
}

// Garanties :
// ✅ Soit tout réussit, soit rien
// ✅ Pas d'état intermédiaire visible
// ✅ Rollback automatique en cas d'erreur
// ✅ Cohérence garantie
```

**Coût :** ~20-30% plus lent qu'une opération simple, mais cohérence garantie.

### Approche 3 : Modélisation sans transaction (optimale)

```javascript
// 🎯 MEILLEURE SOLUTION : Tout dans un document
// Atomicité native = gratuite !

// Structure du document compte
{
    _id: "account1",
    owner: "Alice",
    balance: 1000,
    pendingTransfers: [
        // Transferts en cours
    ],
    history: [
        // Historique limité (100 dernières transactions)
    ]
}

// Transfert en 3 phases (pattern Two-Phase Commit applicatif)
async function transferMoneyOptimized(fromId, toId, amount) {
    const transferId = new ObjectId()
    const now = new Date()

    // Phase 1 : Marquer "pending" sur source
    const phase1 = await db.accounts.updateOne(
        {
            _id: fromId,
            balance: { $gte: amount },
            "pendingTransfers.transferId": { $ne: transferId }
        },
        {
            $inc: { balance: -amount },
            $push: {
                pendingTransfers: {
                    transferId,
                    to: toId,
                    amount,
                    state: "pending",
                    date: now
                }
            }
        }
    )

    if (phase1.matchedCount === 0) {
        throw new Error("Insufficient funds")
    }

    // Phase 2 : Appliquer sur destination
    await db.accounts.updateOne(
        { _id: toId },
        {
            $inc: { balance: amount },
            $push: {
                pendingTransfers: {
                    transferId,
                    from: fromId,
                    amount,
                    state: "applied",
                    date: now
                }
            }
        }
    )

    // Phase 3 : Nettoyer les pending
    await db.accounts.updateOne(
        { _id: fromId },
        {
            $pull: {
                pendingTransfers: { transferId }
            },
            $push: {
                history: {
                    $each: [{
                        transferId,
                        type: "transfer_out",
                        amount: -amount,
                        to: toId,
                        date: now
                    }],
                    $slice: -100  // Garder seulement 100 dernières
                }
            }
        }
    )

    await db.accounts.updateOne(
        { _id: toId },
        {
            $pull: {
                pendingTransfers: { transferId }
            },
            $push: {
                history: {
                    $each: [{
                        transferId,
                        type: "transfer_in",
                        amount: amount,
                        from: fromId,
                        date: now
                    }],
                    $slice: -100
                }
            }
        }
    )
}

// Avantages :
// ✅ Atomicité native (chaque update est atomique)
// ✅ Pas de transaction nécessaire
// ✅ Performance maximale
// ✅ État récupérable (pendingTransfers permet de nettoyer si crash)

// Process de récupération en cas de crash
async function cleanupPendingTransfers() {
    const oldPending = await db.accounts.find({
        "pendingTransfers.date": {
            $lt: new Date(Date.now() - 5 * 60 * 1000)  // > 5 minutes
        }
    })

    // Rollback ou compléter selon l'état
}
```

**Avantage :** Performance native, pas de transaction nécessaire !

## Exemple réaliste 2 : E-commerce - Commande et stock

### Scénario

Un client passe une commande. Il faut :
1. Créer la commande
2. Décrémenter le stock
3. Créer une notification
4. Tout doit réussir ou échouer ensemble

### Approche 1 : Avec transaction (cohérence maximale)

```javascript
async function createOrder(customerId, items) {
    const session = db.getMongo().startSession()

    try {
        session.startTransaction({
            readConcern: { level: "snapshot" },
            writeConcern: { w: "majority" }
        })

        // 1. Vérifier et réserver le stock
        for (const item of items) {
            const result = await db.products.updateOne(
                {
                    _id: item.productId,
                    stock: { $gte: item.quantity }
                },
                {
                    $inc: { stock: -item.quantity },
                    $inc: { reservedStock: item.quantity }
                },
                { session }
            )

            if (result.matchedCount === 0) {
                throw new Error(`Insufficient stock for product ${item.productId}`)
            }
        }

        // 2. Créer la commande
        const order = await db.orders.insertOne(
            {
                customerId,
                items,
                total: items.reduce((sum, item) => sum + item.price * item.quantity, 0),
                status: "pending",
                createdAt: new Date()
            },
            { session }
        )

        // 3. Créer notification
        await db.notifications.insertOne(
            {
                userId: customerId,
                type: "order_created",
                orderId: order.insertedId,
                message: "Your order has been created",
                createdAt: new Date()
            },
            { session }
        )

        await session.commitTransaction()
        return { success: true, orderId: order.insertedId }

    } catch (error) {
        await session.abortTransaction()
        return { success: false, error: error.message }
    } finally {
        session.endSession()
    }
}

// Garantie : Cohérence absolue
// Coût : ~30-40% plus lent
```

### Approche 2 : Sans transaction + compensation (performance)

```javascript
async function createOrderEventual(customerId, items) {
    try {
        // 1. Créer commande d'abord (état "pending")
        const order = await db.orders.insertOne({
            customerId,
            items,
            total: items.reduce((sum, item) => sum + item.price * item.quantity, 0),
            status: "pending_stock",  // État intermédiaire
            createdAt: new Date()
        })

        // 2. Réserver stock (opérations atomiques individuelles)
        const stockUpdates = await Promise.allSettled(
            items.map(item =>
                db.products.updateOne(
                    {
                        _id: item.productId,
                        stock: { $gte: item.quantity }
                    },
                    {
                        $inc: { stock: -item.quantity }
                    }
                )
            )
        )

        // Vérifier si tous les stocks ont été réservés
        const allStockReserved = stockUpdates.every(
            result => result.status === "fulfilled" && result.value.matchedCount > 0
        )

        if (!allStockReserved) {
            // Compensation : annuler la commande
            await db.orders.updateOne(
                { _id: order.insertedId },
                { $set: { status: "cancelled_insufficient_stock" } }
            )

            // Restaurer les stocks déjà réservés
            for (let i = 0; i < stockUpdates.length; i++) {
                if (stockUpdates[i].status === "fulfilled" &&
                    stockUpdates[i].value.matchedCount > 0) {
                    await db.products.updateOne(
                        { _id: items[i].productId },
                        { $inc: { stock: items[i].quantity } }
                    )
                }
            }

            throw new Error("Insufficient stock")
        }

        // 3. Finaliser la commande
        await db.orders.updateOne(
            { _id: order.insertedId },
            { $set: { status: "confirmed" } }
        )

        // 4. Notification (asynchrone, best-effort)
        db.notifications.insertOne({
            userId: customerId,
            type: "order_created",
            orderId: order.insertedId,
            createdAt: new Date()
        }).catch(err => console.error("Notification failed:", err))

        return { success: true, orderId: order.insertedId }

    } catch (error) {
        return { success: false, error: error.message }
    }
}

// Garantie : Eventual consistency + compensation
// Coût : Performance native (~2x plus rapide)
// Compromis : États intermédiaires visibles brièvement
```

**Choix :**
- Stock critique + faible volumétrie → Transaction
- High throughput + compensation acceptable → Sans transaction

## Read Concern et Write Concern : Le réglage fin

### Read Concern : Quel niveau de lecture ?

```javascript
// Niveau "local" (par défaut, plus rapide)
db.orders.find().readConcern("local")
// ✅ Lit depuis le nœud local
// ⚠️ Peut lire des données non répliquées (risque de perte)
// 🚀 Performance maximale

// Niveau "majority" (recommandé en production)
db.orders.find().readConcern("majority")
// ✅ Lit seulement les données répliquées sur la majorité
// ✅ Pas de risque de lecture de données perdues après crash
// ⚠️ Légèrement plus lent (~5-10%)

// Niveau "snapshot" (dans les transactions)
session.startTransaction({
    readConcern: { level: "snapshot" }
})
// ✅ Isolation complète (lectures cohérentes dans le temps)
// ✅ Pas de lectures sales (dirty reads)
// ⚠️ Seulement dans les transactions

// Niveau "linearizable" (cohérence la plus forte)
db.criticalData.findOne(
    { _id: "config" },
    { readConcern: { level: "linearizable" } }
)
// ✅ Garantit la lecture la plus récente
// ✅ Lecture linéarisable (ordre global garanti)
// ❌ Impact performance significatif
// ❌ Seulement pour lecture d'un seul document
```

### Write Concern : Quel niveau d'écriture ?

```javascript
// Niveau par défaut (w: 1)
db.orders.insertOne(
    { /* data */ },
    { writeConcern: { w: 1 } }
)
// ✅ ACK dès que primary écrit
// ⚠️ Risque de perte si primary crash avant réplication
// 🚀 Très rapide

// Niveau "majority" (recommandé en production)
db.orders.insertOne(
    { /* data */ },
    { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
// ✅ ACK après réplication sur majorité des nœuds
// ✅ Données durables (pas de perte après crash)
// ✅ j: true = écrit dans le journal (fsync)
// ⚠️ Plus lent (~10-20%)
// ⚠️ wtimeout : timeout si réplication trop lente

// Niveau maximum (tous les nœuds)
db.criticalData.insertOne(
    { /* data */ },
    { writeConcern: { w: "all", j: true, wtimeout: 10000 } }
)
// ✅ ACK après écriture sur TOUS les nœuds
// ❌ Très lent
// ❌ Bloqué si un nœud est down
```

### Tableau des compromis

| Read/Write Concern | Performance | Durabilité | Usage |
|-------------------|-------------|------------|-------|
| local / w:1 | ⚡⚡⚡ Excellente | ⚠️ Risque perte | Logs, analytics, cache |
| majority / w:1 | ⚡⚡ Bonne | ⚠️ Risque perte write | Lectures critiques |
| local / w:majority | ⚡⚡ Bonne | ✅ Durable | Écritures critiques |
| majority / w:majority | ⚡ Acceptable | ✅ Très durable | **Production standard** |
| snapshot / w:majority | 🐌 Lent | ✅ Transaction ACID | Transactions critiques |
| linearizable / w:all | 🐌🐌 Très lent | ✅ Maximum | Config système uniquement |

## Les coûts réels des transactions

### Benchmark : Insertion simple

```javascript
// Sans transaction
const start1 = Date.now()
for (let i = 0; i < 1000; i++) {
    await db.test.insertOne({ value: i })
}
console.log("Sans transaction:", Date.now() - start1, "ms")
// Résultat : ~500ms

// Avec transaction
const start2 = Date.now()
for (let i = 0; i < 1000; i++) {
    const session = db.getMongo().startSession()
    session.startTransaction()
    await db.test.insertOne({ value: i }, { session })
    await session.commitTransaction()
    session.endSession()
}
console.log("Avec transaction:", Date.now() - start2, "ms")
// Résultat : ~1500ms (3x plus lent)
```

### Impact sur le throughput

```
Opérations/seconde :

Sans transaction :        10,000 ops/s  ████████████████████
Transaction mono-shard :   7,000 ops/s  ██████████████
Transaction multi-shard :  3,000 ops/s  ██████
```

### Consommation mémoire

```javascript
// Transactions accumulent les opérations en mémoire
session.startTransaction()

for (let i = 0; i < 100000; i++) {
    await db.large.insertOne({ data: "x".repeat(1000) }, { session })
    // ⚠️ Tout est gardé en mémoire jusqu'au commit
}

await session.commitTransaction()
// 💥 Risque : Out of memory si transaction trop grosse
```

**Limite :** Transaction > 16 Mo en mémoire → Erreur

## Quand utiliser les transactions ?

### ✅ Utiliser les transactions quand :

1. **Cohérence critique**
```javascript
// Système financier : transferts d'argent
// → Transaction obligatoire
```

2. **Opérations multi-collections interdépendantes**
```javascript
// Commande + Stock + Payment
// Si cohérence stricte requise
```

3. **Rollback automatique essentiel**
```javascript
// Processus complexe où annulation est critique
```

4. **Volumétrie faible à moyenne**
```javascript
// < 1000 transactions/seconde
```

### ❌ Éviter les transactions quand :

1. **Performance critique**
```javascript
// Analytics en temps réel
// Logs à haute fréquence
// → Eventual consistency acceptable
```

2. **Données indépendantes**
```javascript
// Insertion de metrics
// Logs d'activité
// → Pas de relation critique
```

3. **Opération mono-document possible**
```javascript
// Modélisation embedded
// → Atomicité native gratuite !
```

4. **High throughput requis**
```javascript
// > 10,000 ops/seconde
// → Transaction = goulot d'étranglement
```

## Anti-patterns et pièges

### ❌ Anti-pattern 1 : Transactions longues

```javascript
// MAUVAIS : Transaction qui dure longtemps
session.startTransaction()

// Lecture de 10,000 documents
const docs = await db.large.find({}, { session }).toArray()

// Traitement long (5 secondes)
await processHeavyComputation(docs)

// Écriture
await db.result.insertMany(processed, { session })

await session.commitTransaction()

// Problèmes :
// 1. Locks maintenus longtemps
// 2. Risque de timeout
// 3. Bloque autres transactions
```

**Solution :** Transactions courtes, traitement hors transaction.

### ❌ Anti-pattern 2 : Trop d'opérations

```javascript
// MAUVAIS : 10,000 opérations dans une transaction
session.startTransaction()

for (let i = 0; i < 10000; i++) {
    await db.test.insertOne({ value: i }, { session })
}

await session.commitTransaction()
// 💥 Timeout, out of memory, performance désastreuse
```

**Solution :** Limiter à ~100-1000 opérations par transaction.

### ❌ Anti-pattern 3 : Transactions imbriquées

```javascript
// MAUVAIS : Essayer d'imbriquer les transactions
session1.startTransaction()
    // ...
    session2.startTransaction()  // ❌ Pas supporté !
        // ...
    session2.commitTransaction()
    // ...
session1.commitTransaction()
```

**Solution :** MongoDB ne supporte pas les transactions imbriquées.

### ❌ Anti-pattern 4 : Ne pas gérer les erreurs de retry

```javascript
// MAUVAIS : Pas de retry logic
try {
    session.startTransaction()
    // ... opérations
    await session.commitTransaction()
} catch (error) {
    await session.abortTransaction()
    throw error  // ❌ Pas de retry
}

// Problème : TransientTransactionError non géré
```

**Solution :** Implémenter retry logic.

## Bonnes pratiques : aperçu

### ✅ Bonne pratique 1 : Transactions courtes

```javascript
// BON : Transaction rapide et focalisée
session.startTransaction()
try {
    // Seulement les opérations critiques
    await db.accounts.updateOne({ _id: from }, { $inc: { balance: -100 } }, { session })
    await db.accounts.updateOne({ _id: to }, { $inc: { balance: 100 } }, { session })
    await session.commitTransaction()
} catch (error) {
    await session.abortTransaction()
}
```

### ✅ Bonne pratique 2 : Read/Write Concern appropriés

```javascript
// BON : Niveau adapté au cas d'usage
session.startTransaction({
    readConcern: { level: "snapshot" },      // Isolation
    writeConcern: { w: "majority", j: true } // Durabilité
})
```

### ✅ Bonne pratique 3 : Timeout et retry

```javascript
// BON : Gestion complète des erreurs
async function withRetry(operation, maxRetries = 3) {
    for (let i = 0; i < maxRetries; i++) {
        try {
            return await operation()
        } catch (error) {
            if (error.hasErrorLabel("TransientTransactionError") && i < maxRetries - 1) {
                console.log(`Retry ${i + 1}/${maxRetries}`)
                await new Promise(resolve => setTimeout(resolve, 100 * (i + 1)))
                continue
            }
            throw error
        }
    }
}
```

### ✅ Bonne pratique 4 : Préférer la modélisation

```javascript
// MIEUX : Éviter la transaction via la modélisation
// Au lieu de transaction sur 2 documents :
{
    _id: "order1",
    items: [/* ... */],
    payment: {      // ← Embedded
        amount: 150,
        status: "pending",
        transactionId: "..."
    }
}

// Mise à jour atomique
db.orders.updateOne(
    { _id: "order1" },
    { $set: { "payment.status": "completed" } }
)
// ✅ Atomique sans transaction !
```

## Le théorème CAP appliqué

MongoDB vous permet de **choisir** votre position sur le spectre CAP :

```
High Consistency                          High Availability
(Transactions, majority)                  (local, w:1)
        |                                        |
        |                                        |
    [Finance]                              [Analytics]
    [Inventory]                            [Logs]
        |                                        |
        ↓                                        ↓
    Slow, Safe                            Fast, Eventual
```

**Configuration :**
```javascript
// Consistency > Availability
{
    readConcern: "majority",
    writeConcern: { w: "majority", j: true }
}

// Availability > Consistency
{
    readConcern: "local",
    writeConcern: { w: 1 }
}
```

## Conseils d'apprentissage

### 🎯 Méthodologie

1. **Questionner d'abord :** Ai-je vraiment besoin d'une transaction ?
2. **Modéliser pour éviter :** La meilleure transaction est celle qu'on n'a pas besoin de faire
3. **Commencer sans :** Puis ajouter si nécessaire
4. **Mesurer l'impact :** Benchmark avant/après
5. **Documenter le choix :** Pourquoi transaction ici ?

### 🔗 Lien avec les autres chapitres

- **Chapitre 4** : La modélisation peut éliminer 90% des besoins de transactions
- **Chapitre 5** : Les index impactent les performances des transactions
- **Chapitre 9** : Les transactions nécessitent un Replica Set
- **Chapitre 10** : Impact majeur sur les transactions distribuées
- **Chapitre 17** : Optimisation cruciale pour les transactions

---

### 📌 Points clés à retenir

- MongoDB offre ACID depuis la version 4.0
- Trois niveaux : mono-document (gratuit), multi-documents (coût), distribué (coût élevé)
- Transactions = compromis performance vs cohérence
- Read/Write Concern configurent le niveau de garantie
- Coût réel : 10-50% de performance en moins
- La plupart des cas n'ont PAS besoin de transactions
- Modélisation > Transactions
- Transactions courtes, opérations limitées
- Gérer les erreurs TransientTransactionError avec retry
- Choisir le bon niveau selon le cas d'usage

---

**Durée estimée du chapitre** : 6-8 heures
**Niveau** : Avancé nécessitant compréhension ACID
**Prérequis** : Chapitres 1-7, concepts de cohérence

🎯 **Prochaine étape** : Section 8.1 pour approfondir ACID dans le contexte NoSQL.

---

**Prochaine section** : 8.1 - Rappel : ACID

Prêt à maîtriser les transactions MongoDB ? Allons-y ! 🔐

⏭️ [Rappel : ACID (Atomicité, Cohérence, Isolation, Durabilité)](/08-transactions/01-rappel-acid.md)
