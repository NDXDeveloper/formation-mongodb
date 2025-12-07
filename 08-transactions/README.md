🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8. Transactions dans MongoDB

## Introduction

Les transactions représentent l'un des aspects les plus critiques des systèmes de bases de données modernes, garantissant l'intégrité et la cohérence des données lors d'opérations complexes. Dans MongoDB, l'évolution du support transactionnel illustre parfaitement la maturation d'une base de données NoSQL orientée document vers un système capable de gérer des charges de travail d'entreprise exigeantes.

## Contexte Historique et Évolution

### L'ère pré-transactions multi-documents

Avant MongoDB 4.0 (juin 2018), le modèle transactionnel de MongoDB se limitait aux opérations atomiques sur un seul document. Cette limitation n'était pas un défaut de conception, mais plutôt une conséquence directe de la philosophie orientée document :

- **Atomicité native** : Un document MongoDB peut contenir des structures imbriquées complexes, permettant de modéliser des relations entières au sein d'un seul document
- **Dénormalisation encouragée** : L'approche recommandée consistait à concevoir le schéma de manière à ce que les opérations nécessitant l'atomicité se fassent sur un seul document
- **Performance optimale** : L'absence de transactions distribuées permettait des performances exceptionnelles en lecture et écriture

Cette approche fonctionnait remarquablement bien pour de nombreux cas d'usage, particulièrement les applications web modernes où la dénormalisation est une pratique courante.

### La révolution MongoDB 4.0 et au-delà

L'introduction des transactions multi-documents a marqué un tournant majeur :

- **MongoDB 4.0 (2018)** : Transactions multi-documents sur les Replica Sets
- **MongoDB 4.2 (2019)** : Transactions distribuées sur les clusters shardés
- **MongoDB 5.0+ (2021)** : Améliorations significatives des performances transactionnelles

Cette évolution a permis à MongoDB de devenir une solution viable pour des applications nécessitant des garanties transactionnelles strictes, tout en conservant sa flexibilité de schéma et ses capacités de mise à l'échelle horizontale.

## Le Paradoxe des Transactions dans MongoDB

### Quand utiliser les transactions ?

Les transactions multi-documents sont essentielles dans des scénarios spécifiques :

**Transferts financiers**
```javascript
// Scénario : Transfert d'argent entre deux comptes
// Sans transaction, le risque est critique :
// - L'argent pourrait être débité sans être crédité
// - Ou crédité sans être débité
// - Créant une incohérence comptable catastrophique

session.startTransaction();
try {
    await accounts.updateOne(
        { accountId: "A001" },
        { $inc: { balance: -1000 } },
        { session }
    );

    await accounts.updateOne(
        { accountId: "B002" },
        { $inc: { balance: 1000 } },
        { session }
    );

    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
    throw error;
}
```

**Gestion d'inventaire e-commerce**
```javascript
// Scénario : Création d'une commande avec mise à jour de l'inventaire
// Problématique : Éviter la survente
// - Décrémentation du stock
// - Création de la commande
// - Mise à jour du statut client
// Ces opérations doivent être atomiques

session.startTransaction();
try {
    // Vérification et décrémentation du stock
    const product = await products.findOneAndUpdate(
        {
            sku: "LAPTOP-X1",
            stock: { $gte: 1 }
        },
        { $inc: { stock: -1 } },
        { session, returnDocument: 'after' }
    );

    if (!product) {
        throw new Error("Stock insuffisant");
    }

    // Création de la commande
    await orders.insertOne({
        orderId: generateOrderId(),
        customerId: "C12345",
        items: [{ sku: "LAPTOP-X1", quantity: 1 }],
        status: "confirmed",
        timestamp: new Date()
    }, { session });

    // Mise à jour du profil client
    await customers.updateOne(
        { customerId: "C12345" },
        {
            $inc: { totalOrders: 1 },
            $push: { recentOrders: { $each: [orderId], $slice: -10 } }
        },
        { session }
    );

    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
    throw error;
}
```

### Quand NE PAS utiliser les transactions ?

**Anti-pattern : Utilisation systématique des transactions**

Beaucoup de développeurs habitués aux bases relationnelles ont tendance à envelopper toutes leurs opérations dans des transactions. C'est une erreur coûteuse dans MongoDB :

```javascript
// ❌ MAUVAISE PRATIQUE - Transaction inutile
session.startTransaction();
await users.updateOne(
    { userId: "U001" },
    { $set: { lastLogin: new Date() } },
    { session }
);
await session.commitTransaction();

// ✅ BONNE PRATIQUE - Opération atomique naturelle
await users.updateOne(
    { userId: "U001" },
    { $set: { lastLogin: new Date() } }
);
```

**Impact sur les performances**

Les transactions multi-documents introduisent un coût significatif :

- **Latence accrue** : 2-5x plus lente qu'une opération non transactionnelle
- **Verrouillage** : Les transactions acquièrent des verrous, créant des points de contention
- **Pression mémoire** : Les sessions transactionnelles consomment plus de ressources
- **Complexité de retry** : Les conflits transactionnels nécessitent une logique de retry sophistiquée

## Compromis Fondamentaux

### Performance vs Cohérence

MongoDB vous offre un spectre de choix entre performance et garanties de cohérence :

**Cas 1 : Application de réseaux sociaux**
```javascript
// Scénario : Publication d'un post avec compteurs
// Compromis : Cohérence éventuelle acceptable

// Approche 1 : Sans transaction (RECOMMANDÉ)
// - Le post est créé immédiatement
// - Les compteurs sont mis à jour de manière asynchrone
// - Latence minimale pour l'utilisateur

await posts.insertOne({
    postId: generateId(),
    userId: "U001",
    content: "Mon nouveau post",
    timestamp: new Date(),
    likes: 0,
    comments: 0
});

// Mise à jour asynchrone du profil (peut être dans un job séparé)
await users.updateOne(
    { userId: "U001" },
    { $inc: { postCount: 1 } }
);

// Impact : Si le deuxième appel échoue, le compteur sera légèrement inexact
// mais cela n'affecte pas l'expérience utilisateur de manière critique
```

**Cas 2 : Système de facturation**
```javascript
// Scénario : Génération d'une facture avec mise à jour du statut client
// Compromis : Cohérence stricte requise

session.startTransaction();
try {
    const invoice = await invoices.insertOne({
        invoiceId: generateInvoiceId(),
        customerId: "C001",
        amount: 15000,
        status: "pending",
        dueDate: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000)
    }, { session });

    await customers.updateOne(
        { customerId: "C001" },
        {
            $inc: {
                outstandingBalance: 15000,
                invoiceCount: 1
            },
            $set: { lastInvoiceDate: new Date() }
        },
        { session }
    );

    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
    // Dans un contexte financier, l'échec doit être géré rigoureusement
    await auditLog.insertOne({
        event: "invoice_creation_failed",
        customerId: "C001",
        error: error.message,
        timestamp: new Date()
    });
    throw error;
}
```

### Scalabilité vs Garanties Transactionnelles

**Le défi de la distribution**

Plus votre système est distribué, plus les transactions deviennent coûteuses :

```javascript
// Scénario : Cluster shardé avec 10 shards
// Une transaction touchant plusieurs shards doit :
// 1. Coordonner les verrous sur tous les shards impliqués
// 2. Maintenir la cohérence entre les shards
// 3. Gérer les échecs partiels potentiels

// Impact réel mesuré sur un cluster production :
// - Opération simple sur 1 shard : 5ms
// - Transaction sur 2 shards : 50ms (10x plus lent)
// - Transaction sur 5 shards : 200ms (40x plus lent)
// - Risque de conflits et timeouts exponentiellement plus élevé
```

**Stratégie de conception pour la scalabilité**

```javascript
// Alternative 1 : Modélisation pour éviter les transactions
// Au lieu de séparer commandes et lignes de commande :

// ❌ DIFFICILE À SCALER
collections:
  - orders: { orderId, customerId, total, status }
  - orderLines: { orderLineId, orderId, productId, quantity, price }
// Nécessite des transactions pour maintenir la cohérence

// ✅ FACILE À SCALER
orders: {
    orderId: "ORD001",
    customerId: "C001",
    items: [
        { productId: "P001", quantity: 2, price: 50 },
        { productId: "P002", quantity: 1, price: 100 }
    ],
    total: 200,
    status: "confirmed",
    timestamp: ISODate("2024-01-15T10:30:00Z")
}
// Tout est atomique, pas de transaction nécessaire
```

### Complexité Opérationnelle

**Gestion des échecs transactionnels**

Les transactions introduisent une complexité significative dans la gestion des erreurs :

```javascript
// Scénario réaliste : Système de réservation
// Problématique : Gérer les conflits de concurrence

async function createReservationWithRetry(reservationData, maxRetries = 3) {
    let attempt = 0;

    while (attempt < maxRetries) {
        const session = client.startSession();

        try {
            session.startTransaction({
                readConcern: { level: "snapshot" },
                writeConcern: { w: "majority" },
                readPreference: "primary"
            });

            // Vérifier la disponibilité
            const room = await rooms.findOne(
                {
                    roomId: reservationData.roomId,
                    status: "available"
                },
                { session }
            );

            if (!room) {
                throw new Error("Chambre non disponible");
            }

            // Créer la réservation
            await reservations.insertOne({
                ...reservationData,
                status: "confirmed",
                createdAt: new Date()
            }, { session });

            // Mettre à jour le statut de la chambre
            await rooms.updateOne(
                { roomId: reservationData.roomId },
                {
                    $set: {
                        status: "reserved",
                        reservedUntil: reservationData.checkoutDate
                    }
                },
                { session }
            );

            await session.commitTransaction();
            return { success: true };

        } catch (error) {
            await session.abortTransaction();

            // TransientTransactionError : peut être retenté
            if (error.hasErrorLabel('TransientTransactionError')) {
                attempt++;
                console.log(`Tentative ${attempt} échouée, retry...`);
                await sleep(Math.pow(2, attempt) * 100); // Backoff exponentiel
                continue;
            }

            // UnknownTransactionCommitResult : statut incertain
            if (error.hasErrorLabel('UnknownTransactionCommitResult')) {
                // Nécessite une vérification manuelle du résultat
                const existing = await reservations.findOne({
                    roomId: reservationData.roomId,
                    guestId: reservationData.guestId,
                    checkInDate: reservationData.checkInDate
                });

                if (existing) {
                    return { success: true, note: "Transaction réussie après incertitude" };
                }
            }

            throw error;
        } finally {
            await session.endSession();
        }
    }

    throw new Error(`Échec après ${maxRetries} tentatives`);
}
```

## Implications Architecturales

### Impact sur la Conception des Systèmes

**Pattern : Saga au lieu de transactions distribuées**

Pour les systèmes fortement distribués, les sagas offrent une alternative plus scalable :

```javascript
// Scénario : Plateforme de voyage (vols + hôtels + voitures)
// Au lieu d'une transaction distribuée massive :

// ❌ DIFFICILE : Transaction unique
session.startTransaction();
await flights.reserve(..., { session });
await hotels.reserve(..., { session });
await cars.reserve(..., { session });
await payments.charge(..., { session });
await session.commitTransaction();
// Problème : Timeout probable, rollback coûteux

// ✅ SCALABLE : Saga avec compensation
async function bookTravel(travelData) {
    const bookingId = generateId();
    const compensations = [];

    try {
        // Étape 1 : Réserver le vol
        const flightReservation = await flights.reserve({
            ...travelData.flight,
            bookingId
        });
        compensations.push(() => flights.cancel(flightReservation.id));

        // Étape 2 : Réserver l'hôtel
        const hotelReservation = await hotels.reserve({
            ...travelData.hotel,
            bookingId
        });
        compensations.push(() => hotels.cancel(hotelReservation.id));

        // Étape 3 : Réserver la voiture
        const carReservation = await cars.reserve({
            ...travelData.car,
            bookingId
        });
        compensations.push(() => cars.cancel(carReservation.id));

        // Étape 4 : Paiement
        await payments.charge({
            amount: calculateTotal(flightReservation, hotelReservation, carReservation),
            customerId: travelData.customerId,
            bookingId
        });

        // Succès : Confirmer toutes les réservations
        await confirmAllReservations(bookingId);

        return { success: true, bookingId };

    } catch (error) {
        // Échec : Exécuter les compensations dans l'ordre inverse
        console.log("Échec de la réservation, annulation en cours...");

        for (const compensate of compensations.reverse()) {
            try {
                await compensate();
            } catch (compensationError) {
                // Logger l'échec de compensation pour intervention manuelle
                await logCompensationFailure(bookingId, compensationError);
            }
        }

        throw error;
    }
}
```

### Monitoring et Observabilité

Les transactions nécessitent un monitoring spécifique :

```javascript
// Métriques essentielles à surveiller

// 1. Durée des transactions
db.currentOp({
    active: true,
    op: "command",
    "command.commitTransaction": { $exists: true }
})

// 2. Transactions en attente (deadlock potentiel)
db.currentOp({
    active: true,
    secs_running: { $gt: 10 },
    "transaction": { $exists: true }
})

// 3. Taux d'échec transactionnel
db.serverStatus().transactions
// Analyse :
// - retriedCommandsCount : nombre de retry
// - retriedStatementsCount : instructions retry
// - totalAborted : transactions annulées
// - totalCommitted : transactions réussies
// Ratio totalAborted / totalCommitted > 5% = problème

// 4. Contention de verrous
db.serverStatus().locks
// GlobalLock.currentQueue indique la pression
```

## Considérations de Coût-Bénéfice

### Analyse Décisionnelle

**Matrice de décision pour l'utilisation des transactions**

| Critère | Poids | Sans Transaction | Avec Transaction |
|---------|-------|------------------|------------------|
| **Intégrité critique** | 10 | Si incohérence = catastrophe → Transaction requise | |
| **Fréquence d'opération** | 8 | >1000 ops/sec → Préférer design sans transaction | |
| **Distribution géographique** | 7 | Multi-région → Coût prohibitif des transactions | |
| **Complexité acceptable** | 6 | Équipe expérimentée → Peut gérer transactions | |
| **Budget latence** | 9 | <50ms requis → Éviter transactions | |

**Exemple de décision : Système de votes en ligne**

```javascript
// Contexte : Application de sondages en temps réel
// Volume : 10,000 votes/seconde durant les pics
// Exigence : Latence <20ms perçue par l'utilisateur

// Analyse :
// - Cohérence absolue non critique (vote dupliqué = biais négligeable)
// - Performance critique (expérience utilisateur)
// - Volume élevé (scalabilité prioritaire)

// Décision : SANS transaction

// Solution optimisée
await votes.updateOne(
    {
        pollId: "P001",
        optionId: "O1"
    },
    {
        $inc: { count: 1 },
        $addToSet: { voters: userId } // Prévention de vote dupliqué
    },
    { upsert: true }
);

// Résultat :
// - Latence : 5ms moyenne
// - Débit : 15,000 ops/sec
// - Cohérence éventuelle acceptable
// - Pas de verrous bloquants
```

## Évolution et Maturité

### MongoDB Transaction Maturity Model

**Niveau 1 : Atomicité document (pré-4.0)**
- Modélisation orientée document unique
- Performance maximale
- Adapté à 80% des cas d'usage web

**Niveau 2 : Transactions multi-documents Replica Set (4.0+)**
- Garanties ACID sur collections multiples
- Même serveur logique
- Latence acceptable (<100ms)

**Niveau 3 : Transactions distribuées Sharded (4.2+)**
- Garanties ACID cross-shard
- Coordination distribuée
- Coût de performance significatif

**Niveau 4 : Hybrid Approach (Recommandé)**
- 90% du code : opérations atomiques simples
- 10% du code : transactions pour cas critiques
- Architecture saga pour processus longs

## Perspectives et Recommandations

### Règles d'Or

1. **"Transaction as Last Resort"** : N'utilisez une transaction que si la conception sans transaction est impossible ou dangereuse

2. **"Model First, Transact Later"** : Investissez dans la modélisation des données pour minimiser le besoin de transactions

3. **"Measure Everything"** : Avant de déployer des transactions en production, benchmarquez leur impact réel sur votre charge de travail

4. **"Plan for Failure"** : Implémentez une logique de retry robuste et un monitoring transactionnel dès le premier jour

5. **"Consider Eventual Consistency"** : Pour beaucoup de cas d'usage, la cohérence éventuelle avec idempotence est suffisante et beaucoup plus performante

### Vision Pragmatique

MongoDB n'est pas né comme base de données transactionnelle, et ce n'est pas son point fort naturel. Son véritable avantage réside dans :
- La flexibilité du schéma
- L'excellente performance en lecture/écriture
- La scalabilité horizontale native
- La modélisation de domaine riche via les documents

Les transactions sont un outil puissant ajouté à cette boîte à outils, mais comme tout outil puissant, elles doivent être utilisées avec discernement et compréhension de leurs implications.

---


⏭️ [Rappel : ACID (Atomicité, Cohérence, Isolation, Durabilité)](/08-transactions/01-rappel-acid.md)
