🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.1 Rappel : ACID (Atomicité, Cohérence, Isolation, Durabilité)

## Introduction

Les propriétés ACID constituent depuis plus de quatre décennies le socle des systèmes de bases de données transactionnelles. Formulées initialement par Theo Härder et Andreas Reuter en 1983, ces garanties ont été conçues dans le contexte des bases de données relationnelles centralisées. Comprendre ACID dans le contexte de MongoDB nécessite de revisiter ces concepts à travers le prisme d'une architecture distribuée, orientée document, et conçue pour la scalabilité horizontale.

Cette section ne se contente pas de définir ACID — elle explore comment ces propriétés fondamentales se manifestent, se transforment, et parfois se compromettent dans l'écosystème MongoDB moderne.

## ACID dans le Contexte Historique

### L'ère relationnelle (1970-2000)

Les bases de données relationnelles ont été conçues autour d'un paradigme fondamental : **la cohérence avant tout**. Dans un monde où les systèmes s'exécutaient sur des serveurs uniques avec des charges de travail prévisibles, ce choix était naturel :

```
Architecture typique des années 1990 :
┌─────────────────────────────────┐
│   Application Monolithique      │
│                                 │
│   ┌──────────────────────┐      │
│   │  Business Logic      │      │
│   └──────────────────────┘      │
└──────────┬──────────────────────┘
           │
           │ JDBC/ODBC
           ▼
┌─────────────────────────────────┐
│   Base de Données Relationnelle │
│   (Serveur Unique)              │
│                                 │
│   - ACID strict par défaut      │
│   - Verrous pessimistes         │
│   - 2-Phase Commit              │
│   - Journalisation synchrone    │
└─────────────────────────────────┘

Caractéristiques :
✓ Cohérence garantie
✓ Transactions longues acceptables
✓ Scaling vertical (CPU/RAM)
✗ Latence réseau négligeable
✗ Distribution non envisagée
```

### L'ère NoSQL et le théorème CAP (2000-2010)

L'explosion du web et des volumes de données a révélé les limites du modèle relationnel classique :

```
Défi du Web 2.0 :
- Volumes : Pétaoctets de données
- Vitesse : Millions de requêtes/seconde
- Variété : Données non structurées
- Distribution : Datacenters multi-régions

Constat du théorème CAP (Eric Brewer, 2000) :
Dans un système distribué, on ne peut garantir simultanément :
- Consistency (Cohérence)
- Availability (Disponibilité)
- Partition tolerance (Tolérance aux pannes réseau)

Choix : CP ou AP, mais pas CAP
```

MongoDB a été créé en 2009 dans ce contexte, avec un positionnement initial clair : **AP avec cohérence éventuelle**, sacrifiant les garanties ACID strictes pour la performance et la scalabilité.

### La convergence (2010-présent)

Un phénomène remarquable s'est produit depuis 2015 :

```
Convergence des paradigmes :

Bases relationnelles → NoSQL
PostgreSQL, MySQL :
- Ajout de types JSON
- Scaling horizontal (Citus, Vitess)
- Réplication asynchrone

Bases NoSQL → Relationnel
MongoDB, Cassandra :
- Transactions ACID
- Jointures (aggregation $lookup)
- Contraintes de schéma
```

MongoDB aujourd'hui offre un **spectre de garanties configurable**, permettant aux développeurs de choisir le niveau ACID approprié pour chaque opération.

## ACID dans MongoDB : Vue d'Ensemble

### Niveaux de Support ACID

MongoDB offre trois niveaux de garanties ACID avec des compromis différents :

```javascript
// NIVEAU 1 : Atomicité Document Unique
// ✓ ACID complet par défaut
// ✓ Performance maximale
// ✓ Aucune configuration requise

await users.updateOne(
    { userId: "U001" },
    {
        $set: { email: "newemail@example.com" },
        $inc: { version: 1 },
        $push: {
            emailHistory: {
                email: "newemail@example.com",
                changedAt: new Date()
            }
        }
    }
);
// Garantie : Soit tout réussit, soit rien
// Impact performance : ~2-5ms

// NIVEAU 2 : Transactions Multi-Documents (Replica Set)
// ✓ ACID complet sur plusieurs documents
// ✓ Même serveur logique
// ⚠ Performance réduite (2-5x plus lent)
// ⚠ Nécessite Replica Set (minimum 3 nœuds)

const session = client.startSession();
session.startTransaction();
try {
    await users.updateOne(
        { userId: "U001" },
        { $inc: { balance: -100 } },
        { session }
    );

    await transactions.insertOne(
        { userId: "U001", amount: -100, type: "debit" },
        { session }
    );

    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
    throw error;
} finally {
    await session.endSession();
}
// Garantie : ACID sur 2+ collections
// Impact performance : ~10-25ms

// NIVEAU 3 : Transactions Distribuées (Sharded Cluster)
// ✓ ACID complet cross-shard
// ⚠ Performance très réduite (5-10x plus lent)
// ⚠ Complexité opérationnelle élevée
// ⚠ Nécessite coordination distribuée

const session = client.startSession();
session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority", j: true },
    readPreference: "primary"
});
try {
    // Opérations sur plusieurs shards
    await ordersShardedCollection.insertOne(..., { session });
    await inventoryShardedCollection.updateOne(..., { session });
    await customersShardedCollection.updateOne(..., { session });

    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
    throw error;
} finally {
    await session.endSession();
}
// Garantie : ACID distribué
// Impact performance : ~50-200ms
// Risque de timeout et conflits élevé
```

### Matrice de Trade-offs ACID dans MongoDB

| Propriété | Sans Transaction | Transaction Replica Set | Transaction Sharded | Coût Réel |
|-----------|------------------|------------------------|---------------------|-----------|
| **Atomicité** | Document unique | Multi-documents, même RS | Multi-documents, cross-shard | 2x → 10x latence |
| **Cohérence** | Immédiate locale | Configurable (snapshot) | Configurable (snapshot) | Contention verrous |
| **Isolation** | Document-level | Snapshot isolation | Snapshot isolation | Retry logic requis |
| **Durabilité** | Configurable (journaling) | Configurable (write concern) | Configurable (write concern) | Latence vs fiabilité |

## Les Quatre Propriétés Revisitées pour MongoDB

### A comme Atomicité : "Tout ou Rien"

**Définition classique** : Une transaction est une unité indivisible d'opérations. Soit toutes les opérations réussissent, soit aucune.

**Réalité MongoDB** : L'atomicité a plusieurs visages selon le contexte :

```javascript
// CAS 1 : Atomicité native (documents imbriqués)
// Scénario : Mise à jour d'un profil utilisateur complexe

const userUpdate = {
    $set: {
        "profile.firstName": "Jean",
        "profile.lastName": "Dupont",
        "profile.lastModified": new Date()
    },
    $inc: {
        "profile.version": 1
    },
    $push: {
        "auditLog": {
            action: "profile_update",
            timestamp: new Date(),
            fields: ["firstName", "lastName"]
        }
    }
};

await users.updateOne({ userId: "U001" }, userUpdate);

// Garantie d'atomicité :
// ✓ Toutes les modifications sont appliquées atomiquement
// ✓ Aucune latence additionnelle
// ✓ Pas de verrous inter-documents
// ✓ Pas de risque de deadlock
// Performance : ~2-3ms
```

```javascript
// CAS 2 : Atomicité limitée sans transaction
// Scénario : Transfert d'argent mal implémenté

async function transferMoneyWRONG(fromId, toId, amount) {
    // ❌ DANGER : Pas atomique

    // Étape 1 : Débiter le compte source
    await accounts.updateOne(
        { accountId: fromId },
        { $inc: { balance: -amount } }
    );

    // PROBLÈME : Si le serveur crash ici, l'argent disparaît !
    // Ou si l'étape 2 échoue, incohérence permanente

    // Étape 2 : Créditer le compte destination
    await accounts.updateOne(
        { accountId: toId },
        { $inc: { balance: amount } }
    );

    // Entre les deux opérations :
    // - Le système est dans un état incohérent
    // - Les totaux ne correspondent pas
    // - Aucun moyen de rollback automatique
}

// Impact d'une panne entre les deux opérations :
// - 10:00:00.000 : Débit de 1000€ sur compte A (solde: 9000€)
// - 10:00:00.050 : CRASH SERVEUR
// - 10:00:10.000 : Redémarrage
// Résultat : Compte A débité, compte B jamais crédité
// 1000€ perdus dans la nature
```

```javascript
// CAS 3 : Atomicité garantie avec transaction
// Scénario : Transfert d'argent correctement implémenté

async function transferMoneyCORRECT(fromId, toId, amount) {
    const session = client.startSession();

    try {
        session.startTransaction({
            readConcern: { level: "snapshot" },
            writeConcern: { w: "majority" }
        });

        // Les deux opérations sont maintenant atomiques
        const debitResult = await accounts.updateOne(
            {
                accountId: fromId,
                balance: { $gte: amount } // Vérification du solde
            },
            {
                $inc: { balance: -amount },
                $push: {
                    transactions: {
                        type: "debit",
                        amount: amount,
                        timestamp: new Date(),
                        reference: toId
                    }
                }
            },
            { session }
        );

        if (debitResult.matchedCount === 0) {
            throw new Error("Solde insuffisant ou compte inexistant");
        }

        await accounts.updateOne(
            { accountId: toId },
            {
                $inc: { balance: amount },
                $push: {
                    transactions: {
                        type: "credit",
                        amount: amount,
                        timestamp: new Date(),
                        reference: fromId
                    }
                }
            },
            { session }
        );

        await session.commitTransaction();

        // Garanties :
        // ✓ Soit les deux comptes sont modifiés
        // ✓ Soit aucun n'est modifié
        // ✓ Pas d'état intermédiaire visible
        // ✓ Rollback automatique en cas d'erreur

    } catch (error) {
        await session.abortTransaction();
        throw error;
    } finally {
        await session.endSession();
    }
}

// Performance mesurée :
// - Sans transaction : 2-3ms par opération = 4-6ms total
// - Avec transaction : 15-30ms total (5x plus lent)
// Trade-off : 20ms de latence pour éliminer le risque de perte d'argent
```

**Implications pratiques de l'atomicité** :

```javascript
// Scénario réel : Système de facturation e-commerce
// Question : Transaction nécessaire ou pas ?

// Contexte :
// - 100,000 commandes/jour
// - Latence p95 cible : <50ms
// - Taux d'erreur acceptable : <0.01%

// Option A : Sans transaction (risque calculé)
async function processOrderRISKY(orderData) {
    // Étape 1 : Créer la commande
    const order = await orders.insertOne({
        orderId: generateId(),
        customerId: orderData.customerId,
        items: orderData.items,
        total: orderData.total,
        status: "pending"
    });

    // Étape 2 : Décrémenter l'inventaire
    for (const item of orderData.items) {
        await inventory.updateOne(
            { sku: item.sku },
            { $inc: { stock: -item.quantity } }
        );
    }

    // Étape 3 : Créer la facture
    await invoices.insertOne({
        invoiceId: generateId(),
        orderId: order.insertedId,
        amount: orderData.total,
        status: "unpaid"
    });

    // Analyse de risque :
    // - Si crash entre les étapes : incohérence
    // - Mais : job de réconciliation quotidien
    // - Impact : <10 commandes/jour affectées
    // - Coût business : 100€/commande × 10 = 1000€/jour
    // - Performance : 8ms par commande
    // - Capacité : 12,500 commandes/seconde
}

// Option B : Avec transaction (zéro risque)
async function processOrderSAFE(orderData) {
    const session = client.startSession();

    try {
        session.startTransaction({
            readConcern: { level: "snapshot" },
            writeConcern: { w: "majority" }
        });

        const order = await orders.insertOne({
            orderId: generateId(),
            customerId: orderData.customerId,
            items: orderData.items,
            total: orderData.total,
            status: "pending"
        }, { session });

        for (const item of orderData.items) {
            await inventory.updateOne(
                { sku: item.sku },
                { $inc: { stock: -item.quantity } },
                { session }
            );
        }

        await invoices.insertOne({
            invoiceId: generateId(),
            orderId: order.insertedId,
            amount: orderData.total,
            status: "unpaid"
        }, { session });

        await session.commitTransaction();

    } catch (error) {
        await session.abortTransaction();
        throw error;
    } finally {
        await session.endSession();
    }

    // Analyse de coût :
    // - Risque d'incohérence : 0%
    // - Performance : 45ms par commande (5.6x plus lent)
    // - Capacité : 2,200 commandes/seconde
    // - Scaling requis : 6x plus de serveurs
    // - Coût infrastructure : +500k$/an
}

// Décision finale : Hybride intelligent
async function processOrderHYBRID(orderData) {
    // Modélisation qui élimine le besoin de transaction

    const orderDocument = {
        orderId: generateId(),
        customerId: orderData.customerId,
        items: orderData.items.map(item => ({
            sku: item.sku,
            quantity: item.quantity,
            price: item.price,
            // ⚠ Important : Snapshot du stock au moment de la commande
            stockAtPurchase: item.currentStock
        })),
        total: orderData.total,
        status: "pending",
        // Toutes les données nécessaires dans un document
        invoice: {
            invoiceId: generateId(),
            amount: orderData.total,
            status: "unpaid",
            createdAt: new Date()
        },
        createdAt: new Date()
    };

    // Une seule opération atomique
    await orders.insertOne(orderDocument);

    // Mise à jour de l'inventaire en asynchrone
    // (avec job de réconciliation si nécessaire)
    await updateInventoryAsync(orderData.items);

    // Résultat :
    // ✓ Atomicité de la commande : garantie
    // ✓ Performance : 5ms (meilleur des deux mondes)
    // ✓ Scalabilité : 20,000 commandes/seconde
    // ⚠ Cohérence inventaire : éventuelle (acceptable pour e-commerce)
}
```

### C comme Cohérence : "Respecter les Règles"

**Définition classique** : Une transaction fait passer la base de données d'un état cohérent à un autre état cohérent, respectant toutes les contraintes d'intégrité.

**Réalité MongoDB** : La cohérence a plusieurs dimensions et peut être relâchée stratégiquement :

```javascript
// DIMENSION 1 : Cohérence structurelle (via validation de schéma)

db.createCollection("accounts", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["accountId", "balance", "currency"],
            properties: {
                accountId: {
                    bsonType: "string",
                    pattern: "^ACC[0-9]{8}$"
                },
                balance: {
                    bsonType: "double",
                    minimum: 0, // Contrainte : pas de solde négatif
                    maximum: 1000000000
                },
                currency: {
                    enum: ["EUR", "USD", "GBP"]
                },
                status: {
                    enum: ["active", "frozen", "closed"]
                }
            }
        }
    },
    validationAction: "error" // Rejeter les documents non conformes
});

// Tentative de violation
try {
    await accounts.insertOne({
        accountId: "ACC12345678",
        balance: -500, // ❌ Violation : solde négatif
        currency: "EUR"
    });
} catch (error) {
    // WriteError: Document failed validation
    console.log("Contrainte de cohérence respectée");
}

// Garantie : Cohérence structurelle au niveau document
```

```javascript
// DIMENSION 2 : Cohérence référentielle (manuelle ou via transactions)

// Scénario : Système de blogging
// Relations : Users → Posts → Comments

// Approche 1 : Sans garantie de cohérence référentielle
async function deleteUserNAIVE(userId) {
    // ❌ PROBLÈME : Orphelins possibles
    await users.deleteOne({ userId });

    // Si crash ici, les posts et comments existent toujours
    // Références cassées (orphans)

    await posts.deleteMany({ authorId: userId });
    await comments.deleteMany({ authorId: userId });
}

// Impact réel :
// - Interface affiche "Auteur inconnu"
// - Impossibilité de contacter l'auteur
// - Pollution de la base avec données orphelines

// Approche 2 : Avec transaction (cohérence garantie)
async function deleteUserCONSISTENT(userId) {
    const session = client.startSession();

    try {
        session.startTransaction();

        // Suppression atomique de tout le graphe
        await users.deleteOne({ userId }, { session });
        await posts.deleteMany({ authorId: userId }, { session });
        await comments.deleteMany({ authorId: userId }, { session });

        await session.commitTransaction();

    } catch (error) {
        await session.abortTransaction();
        throw error;
    } finally {
        await session.endSession();
    }
}

// Approche 3 : Dénormalisation (élimine le problème)
// Post document contient les infos essentielles de l'auteur
{
    postId: "P001",
    title: "Mon article",
    content: "...",
    author: {
        userId: "U001",
        name: "Jean Dupont",
        avatar: "https://..."
        // Snapshot au moment de la publication
    },
    publishedAt: ISODate("2024-01-15")
}

// Si l'utilisateur est supprimé :
// ✓ Le post conserve les infos de l'auteur
// ✓ Pas d'orphelins
// ✓ Pas de transaction nécessaire
// ⚠ Données historiques (par design)
```

```javascript
// DIMENSION 3 : Cohérence causale (read-after-write)

// Scénario : Publication d'un article et notification
async function publishArticlePROBLEM(article) {
    // Écriture sur le Primary
    const result = await articles.insertOne({
        articleId: generateId(),
        title: article.title,
        content: article.content,
        publishedAt: new Date(),
        status: "published"
    });

    // Lecture immédiate (peut frapper un Secondary)
    const published = await articles.findOne(
        { articleId: result.insertedId },
        { readPreference: "secondaryPreferred" }
    );

    // ❌ PROBLÈME : L'article peut ne pas être trouvé !
    // La réplication vers les secondaries prend 10-100ms
    if (!published) {
        console.log("Article introuvable"); // Incohérence perçue
    }

    // Envoi de notification basée sur lecture incohérente
    await sendNotification(published); // Peut échouer
}

// Solution : Read Concern "majority" + Write Concern "majority"
async function publishArticleCONSISTENT(article) {
    const session = client.startSession();

    // Write Concern : attendre la majorité du replica set
    const result = await articles.insertOne({
        articleId: generateId(),
        title: article.title,
        content: article.content,
        publishedAt: new Date(),
        status: "published"
    }, {
        writeConcern: { w: "majority", wtimeout: 5000 }
    });

    // Read Concern : lire seulement les données majoritaires
    const published = await articles.findOne(
        { articleId: result.insertedId },
        {
            readConcern: { level: "majority" },
            session
        }
    );

    // ✓ Garantie : Si l'article est trouvé, il est durable
    // ⚠ Trade-off : Latence augmentée de 50-100ms

    await sendNotification(published);
}
```

### I comme Isolation : "Pas d'Interférence"

**Définition classique** : Les transactions concurrentes s'exécutent comme si elles étaient séquentielles. Les modifications d'une transaction ne sont pas visibles aux autres tant qu'elle n'est pas committée.

**Réalité MongoDB** : L'isolation est configurable avec des niveaux différents selon les besoins :

```javascript
// NIVEAU 1 : Isolation document-level (par défaut, sans transaction)

// Scénario : Compteur de vues sur un article
// 1000 utilisateurs lisent l'article simultanément

// Opération atomique garantie au niveau document
await articles.updateOne(
    { articleId: "ART001" },
    { $inc: { views: 1 } }
);

// Garantie d'isolation :
// ✓ Les 1000 incréments seront tous appliqués
// ✓ Aucune lecture ne verra un état intermédiaire
// ✓ Performance maximale (pas de verrouillage inter-documents)
// Résultat final : views: 1000 (correct)
```

```javascript
// NIVEAU 2 : Isolation faible (Read Uncommitted équivalent)

// Scénario : Dashboard temps réel
// Trade-off accepté : Précision vs performance

const stats = await db.collection('orders').aggregate([
    { $match: { status: "completed", date: today } },
    { $group: {
        _id: null,
        totalRevenue: { $sum: "$amount" },
        orderCount: { $sum: 1 }
    }}
], {
    // Lecture sur n'importe quel nœud, données potentiellement stale
    readPreference: "nearest",
    readConcern: { level: "local" }
});

// Caractéristiques :
// ✓ Latence minimale : 5-10ms
// ✓ Charge distribuée sur tous les nœuds
// ⚠ Peut lire des données non durables
// ⚠ Peut voir des mises à jour partielles
// ✓ Acceptable pour un dashboard (valeurs indicatives)
```

```javascript
// NIVEAU 3 : Snapshot Isolation (transactions multi-documents)

// Scénario : Rapport financier consistant
// Exigence : Vue cohérente à un instant T

const session = client.startSession();
session.startTransaction({
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" }
});

try {
    // Toutes les lectures voient la même version (snapshot)
    const accountsData = await accounts.find({}, { session }).toArray();
    const transactionsData = await transactions.find({
        date: { $gte: startDate, $lte: endDate }
    }, { session }).toArray();
    const balancesData = await balances.find({}, { session }).toArray();

    // Garantie : Vue cohérente à l'instant du début de transaction
    // Les modifications concurrentes ne sont PAS visibles

    const report = generateFinancialReport(
        accountsData,
        transactionsData,
        balancesData
    );

    await session.commitTransaction();
    return report;

} catch (error) {
    await session.abortTransaction();
    throw error;
} finally {
    await session.endSession();
}

// Caractéristiques :
// ✓ Cohérence stricte des données du rapport
// ✓ Isolation complète des transactions concurrentes
// ⚠ Latence : 100-500ms
// ⚠ Peut échouer sur conflit (WriteConflict)
```

```javascript
// Problème réel : Write Conflicts avec Snapshot Isolation

// Scénario : Deux transactions concurrentes sur le même document
// Transaction A et Transaction B tentent de modifier le même compte

// Transaction A (démarrée à t=0)
const sessionA = client.startSession();
sessionA.startTransaction({ readConcern: { level: "snapshot" } });

const accountA = await accounts.findOne(
    { accountId: "ACC001" },
    { session: sessionA }
);
// accountA.balance = 1000

// ... traitement long (500ms) ...

await accounts.updateOne(
    { accountId: "ACC001" },
    { $set: { balance: accountA.balance + 500 } },
    { session: sessionA }
);

// Transaction B (démarrée à t=100ms)
const sessionB = client.startSession();
sessionB.startTransaction({ readConcern: { level: "snapshot" } });

const accountB = await accounts.findOne(
    { accountId: "ACC001" },
    { session: sessionB }
);
// accountB.balance = 1000 (même snapshot qu'A)

// ... traitement court (50ms) ...

await accounts.updateOne(
    { accountId: "ACC001" },
    { $set: { balance: accountB.balance + 200 } },
    { session: sessionB }
);
// ✓ B commit en premier (à t=150ms)

await sessionB.commitTransaction();

// Maintenant A tente de committer (à t=500ms)
try {
    await sessionA.commitTransaction();
} catch (error) {
    // ❌ WriteConflict : B a modifié le document entre temps
    // error.hasErrorLabel('TransientTransactionError') === true

    // Solution : Retry avec backoff exponentiel
    await sessionA.abortTransaction();
}

// Implication : Logique de retry obligatoire
async function executeWithRetry(transactionFn, maxRetries = 3) {
    let attempt = 0;

    while (attempt < maxRetries) {
        try {
            return await transactionFn();
        } catch (error) {
            if (error.hasErrorLabel('TransientTransactionError') &&
                attempt < maxRetries - 1) {
                attempt++;
                // Backoff exponentiel : 100ms, 200ms, 400ms
                await sleep(Math.pow(2, attempt) * 100);
                continue;
            }
            throw error;
        }
    }
}
```

### D comme Durabilité : "Pas de Perte de Données"

**Définition classique** : Une fois une transaction committée, ses modifications sont permanentes, même en cas de panne système.

**Réalité MongoDB** : La durabilité est un spectre configurable avec des trade-offs explicites :

```javascript
// NIVEAU 1 : Durabilité minimale (performance maximale)

await orders.insertOne({
    orderId: "ORD001",
    amount: 1500,
    status: "pending"
}, {
    writeConcern: { w: 0 } // "Fire and forget"
});

// Caractéristiques :
// ✓ Latence : <1ms (pas d'attente)
// ✓ Débit maximum
// ❌ Aucune garantie de persistance
// ❌ Les erreurs d'écriture sont ignorées
// ❌ En cas de crash immédiat : données perdues

// Cas d'usage légitime : Logging non critique, analytics
```

```javascript
// NIVEAU 2 : Durabilité locale (défaut MongoDB)

await orders.insertOne({
    orderId: "ORD002",
    amount: 2000,
    status: "pending"
}, {
    writeConcern: { w: 1, j: false } // Défaut implicite
});

// Caractéristiques :
// ✓ Latence : 2-5ms
// ✓ Confirmé par le primary
// ⚠ Pas encore sur disque (en mémoire WiredTiger)
// ⚠ Crash du serveur avant flush : perte possible (rare)
// ✓ Acceptable pour la plupart des applications web
```

```javascript
// NIVEAU 3 : Durabilité journal (crash-safe)

await orders.insertOne({
    orderId: "ORD003",
    amount: 3000,
    status: "confirmed"
}, {
    writeConcern: { w: 1, j: true }
});

// Caractéristiques :
// ✓ Latence : 10-20ms
// ✓ Écrit dans le journal (Write-Ahead Log)
// ✓ Survit à un crash du serveur
// ⚠ Pas de réplication (perte si disque défaillant)
// ✓ Bon compromis pour opérations critiques solo
```

```javascript
// NIVEAU 4 : Durabilité répliquée (haute disponibilité)

await orders.insertOne({
    orderId: "ORD004",
    amount: 5000,
    status: "confirmed"
}, {
    writeConcern: {
        w: "majority",        // Majorité du replica set
        j: true,              // + journal
        wtimeout: 5000        // Timeout après 5s
    }
});

// Caractéristiques :
// ✓ Latence : 50-150ms (dépend de la géographie)
// ✓ Répliqué sur majorité des nœuds
// ✓ Survit à la panne de n/2-1 nœuds
// ✓ Durabilité maximale
// ⚠ Latence élevée en multi-région
// ✓ Recommandé pour données financières

// Exemple concret : Replica Set à 3 nœuds
// - Primary (Paris)
// - Secondary 1 (Paris)
// - Secondary 2 (Londres)
//
// w: "majority" = attendre 2 nœuds (Paris + Londres)
// Latence réseau Paris-Londres : ~15ms
// Total : 50-80ms
```

```javascript
// Scénario réel : Échec de durabilité

// Contexte : E-commerce haute fréquence
// Configuration initiale : writeConcern { w: 1 }

await orders.insertOne({
    orderId: "ORD12345",
    amount: 150000, // Commande importante
    items: [...],
    status: "confirmed"
}, {
    writeConcern: { w: 1, j: false } // ⚠ Durabilité faible
});

// Timeline d'un incident réel :
// 14:32:45.123 - Commande insérée (confirmée à l'utilisateur)
// 14:32:45.125 - Écriture dans le cache WiredTiger (RAM)
// 14:32:45.XXX - Flush vers disque prévu dans ~60s
// 14:32:47.000 - ⚡ PANNE ÉLECTRIQUE DATACENTER
// 14:35:00.000 - Redémarrage du serveur
// 14:35:30.000 - Récupération du journal
// 14:35:35.000 - Serveur opérationnel

// Résultat :
// ❌ Commande perdue (jamais sur disque)
// ❌ Client débité mais pas de commande enregistrée
// ❌ Réclamation client : "Votre système a perdu ma commande"
// ❌ Coût : Investigation + remboursement + perte de confiance

// Solution : writeConcern adapté aux enjeux
await orders.insertOne({
    orderId: "ORD12346",
    amount: 150000,
    items: [...],
    status: "confirmed"
}, {
    writeConcern: {
        w: "majority",  // Réplication
        j: true,        // Journal
        wtimeout: 5000
    }
});

// Trade-off accepté :
// ✓ +40ms de latence
// ✓ Zéro perte de données
// ✓ Confiance client maintenue
// ROI : 40ms pour éviter des milliers d'euros de litiges
```

## Interactions et Dépendances entre Propriétés ACID

### Le Triangle Impossible

Il existe des tensions naturelles entre les propriétés ACID dans un système distribué :

```
        Cohérence Forte
              ▲
             /|\
            / | \
           /  |  \
          /   |   \
    Durabilité─┼─Disponibilité
               |
               |
          Performance

Théorème : On ne peut maximiser
les 4 dimensions simultanément
```

**Exemple : Système bancaire distribué**

```javascript
// Configuration ultra-stricte (priorité : cohérence + durabilité)
session.startTransaction({
    readConcern: { level: "linearizable" },    // Cohérence la plus forte
    writeConcern: { w: "majority", j: true },  // Durabilité maximale
    readPreference: "primary"                   // Isolation stricte
});

// Conséquences mesurées :
// - Latence : 200-500ms par opération
// - Disponibilité : Indisponible si majority non accessible
// - Performance : 50-100 transactions/seconde max
// - Cohérence : Garantie absolue
// - Durabilité : Garantie absolue

// Scénario acceptable :
// - Transactions bancaires (volume modéré, enjeux élevés)
// - Quelques milliers de transactions/jour
// - Budget de latence : plusieurs centaines de ms tolérables
```

```javascript
// Configuration performance (priorité : disponibilité + vitesse)
await metrics.insertOne({
    timestamp: new Date(),
    userId: "U001",
    action: "page_view",
    page: "/products"
}, {
    writeConcern: { w: 1, j: false },
    readConcern: { level: "local" }
});

// Conséquences mesurées :
// - Latence : 2-5ms par opération
// - Disponibilité : Haute (tolère pannes)
// - Performance : 50,000+ opérations/seconde
// - Cohérence : Éventuelle (acceptable)
// - Durabilité : Différée (acceptable)

// Scénario acceptable :
// - Analytics en temps réel
// - Millions d'événements/jour
// - Budget de latence : <10ms critique
// - Perte de quelques événements : acceptable
```

## Conclusion : ACID Pragmatique dans MongoDB

MongoDB moderne offre un **spectre complet de garanties ACID**, permettant aux architectes de faire des choix éclairés basés sur les véritables besoins métier plutôt que sur des absolus théoriques.

### Principes Directeurs

1. **Comprendre avant de configurer** : Chaque niveau ACID a un coût mesurable en latence, disponibilité, et complexité

2. **Différencier par criticité** : Toutes les données ne nécessitent pas les mêmes garanties
   - Données financières : ACID strict
   - Analytics : Cohérence éventuelle
   - Session utilisateur : Compromis

3. **Mesurer, ne pas supposer** : Les trade-offs théoriques se manifestent différemment selon l'infrastructure
   - Latence réseau
   - Configuration disque
   - Charge du système

4. **Modéliser pour minimiser** : La meilleure transaction est celle qu'on n'a pas besoin d'exécuter
   - Documents imbriqués
   - Dénormalisation stratégique
   - Idempotence

### L'Évolution Continue

MongoDB continue d'améliorer son support ACID :
- Performances transactionnelles accrues
- Nouveaux niveaux d'isolation
- Optimisations des conflits
- Durabilité configurable plus fine

Mais la leçon fondamentale reste : **ACID n'est pas binaire**. C'est un outil à utiliser avec discernement, en comprenant ses implications sur l'ensemble du système.

---


⏭️ [Définition des propriétés ACID](/08-transactions/01.1-definition-proprietes-acid.md)
