🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 8.7 Bonnes pratiques transactionnelles

## Introduction : L'art de bien faire les transactions

Les bonnes pratiques transactionnelles ne sont pas une simple liste de règles à suivre aveuglément, mais un **ensemble de principes éprouvés** qui permettent de construire des systèmes robustes, performants et maintenables. Cette section distille l'expérience de systèmes de production à grande échelle pour offrir un guide pragmatique et actionnable.

### Philosophie des bonnes pratiques

```
Principes fondamentaux :

1. SIMPLICITÉ
   → La meilleure transaction est celle qu'on n'a pas à faire
   → Une transaction simple est une transaction maintenable

2. RAPIDITÉ
   → Minimiser le temps transactionnel
   → Libérer les verrous le plus tôt possible

3. RÉSILIENCE
   → Anticiper les échecs
   → Gérer les erreurs gracieusement

4. OBSERVABILITÉ
   → Mesurer constamment
   → Alerter proactivement

5. ÉVOLUTIVITÉ
   → Concevoir pour la croissance
   → Optimiser pour la scalabilité
```

## Pattern 1 : Wrapper transactionnel réutilisable

### Implémentation d'une classe Transaction Manager

```javascript
/**
 * TransactionManager - Gestionnaire de transactions avec retry automatique,
 * monitoring, et gestion des erreurs
 */
class TransactionManager {
  constructor(client, options = {}) {
    this.client = client;
    this.options = {
      maxRetries: options.maxRetries || 3,
      initialBackoffMs: options.initialBackoffMs || 100,
      maxBackoffMs: options.maxBackoffMs || 5000,
      enableMetrics: options.enableMetrics !== false,
      enableLogging: options.enableLogging !== false,
      ...options
    };

    this.metrics = {
      totalAttempts: 0,
      successCount: 0,
      failureCount: 0,
      retryCount: 0,
      totalDuration: 0,
      errors: new Map()
    };
  }

  /**
   * Exécute une fonction dans une transaction avec retry automatique
   */
  async execute(transactionFn, config = {}) {
    const startTime = Date.now();
    const txnConfig = this._buildConfig(config);
    let lastError = null;

    for (let attempt = 0; attempt < this.options.maxRetries; attempt++) {
      const session = this.client.startSession();

      try {
        this.metrics.totalAttempts++;

        if (this.options.enableLogging && attempt > 0) {
          console.log(`[TransactionManager] Retry attempt ${attempt + 1}/${this.options.maxRetries}`);
        }

        // Exécuter la transaction
        const result = await session.withTransaction(
          async () => await transactionFn(session),
          txnConfig
        );

        // Succès
        this.metrics.successCount++;
        this.metrics.totalDuration += Date.now() - startTime;

        if (attempt > 0) {
          this.metrics.retryCount++;
        }

        return {
          success: true,
          result,
          attempts: attempt + 1,
          duration: Date.now() - startTime
        };

      } catch (error) {
        lastError = error;
        this._recordError(error);

        // Déterminer si on doit retry
        if (this._shouldRetry(error, attempt)) {
          const backoff = this._calculateBackoff(attempt);

          if (this.options.enableLogging) {
            console.warn(
              `[TransactionManager] Transaction failed: ${error.message}. ` +
              `Retrying in ${backoff}ms...`
            );
          }

          await this._sleep(backoff);
          continue;
        }

        // Échec définitif
        break;

      } finally {
        await session.endSession();
      }
    }

    // Toutes les tentatives ont échoué
    this.metrics.failureCount++;
    this.metrics.totalDuration += Date.now() - startTime;

    return {
      success: false,
      error: lastError,
      attempts: this.options.maxRetries,
      duration: Date.now() - startTime
    };
  }

  /**
   * Version avec callback de succès/échec
   */
  async executeWithCallbacks(transactionFn, callbacks = {}) {
    const result = await this.execute(transactionFn, callbacks.config);

    if (result.success) {
      if (callbacks.onSuccess) {
        await callbacks.onSuccess(result.result, result);
      }
    } else {
      if (callbacks.onFailure) {
        await callbacks.onFailure(result.error, result);
      }
    }

    if (callbacks.onComplete) {
      await callbacks.onComplete(result);
    }

    return result;
  }

  /**
   * Construit la configuration de transaction
   */
  _buildConfig(config) {
    const defaults = {
      readConcern: { level: 'snapshot' },
      writeConcern: { w: 'majority' },
      maxCommitTimeMS: 10000
    };

    return {
      ...defaults,
      ...config,
      // Fusionner les concerns
      readConcern: { ...defaults.readConcern, ...config.readConcern },
      writeConcern: { ...defaults.writeConcern, ...config.writeConcern }
    };
  }

  /**
   * Détermine si on doit retry
   */
  _shouldRetry(error, attempt) {
    // Pas de retry si on a atteint le maximum
    if (attempt >= this.options.maxRetries - 1) {
      return false;
    }

    // Retry sur les erreurs transitoires
    if (error.hasErrorLabel('TransientTransactionError')) {
      return true;
    }

    // Retry sur les write conflicts
    if (error.code === 112 || error.codeName === 'WriteConflict') {
      return true;
    }

    // Retry sur les timeouts
    if (error.code === 50 || error.codeName === 'MaxTimeMSExpired') {
      return true;
    }

    // Pas de retry pour les autres erreurs
    return false;
  }

  /**
   * Calcule le backoff exponentiel avec jitter
   */
  _calculateBackoff(attempt) {
    const exponentialBackoff = Math.min(
      this.options.initialBackoffMs * Math.pow(2, attempt),
      this.options.maxBackoffMs
    );

    // Ajouter du jitter (±25%)
    const jitter = exponentialBackoff * 0.25 * (Math.random() - 0.5) * 2;

    return Math.floor(exponentialBackoff + jitter);
  }

  /**
   * Enregistre une erreur dans les métriques
   */
  _recordError(error) {
    const errorKey = error.codeName || error.code || 'Unknown';
    const count = this.metrics.errors.get(errorKey) || 0;
    this.metrics.errors.set(errorKey, count + 1);
  }

  /**
   * Sleep helper
   */
  _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  /**
   * Récupère les métriques
   */
  getMetrics() {
    const successRate = this.metrics.totalAttempts > 0
      ? (this.metrics.successCount / this.metrics.totalAttempts * 100).toFixed(2)
      : 0;

    const avgDuration = this.metrics.totalAttempts > 0
      ? Math.floor(this.metrics.totalDuration / this.metrics.totalAttempts)
      : 0;

    return {
      ...this.metrics,
      successRate: `${successRate}%`,
      avgDurationMs: avgDuration,
      errorBreakdown: Object.fromEntries(this.metrics.errors)
    };
  }

  /**
   * Réinitialise les métriques
   */
  resetMetrics() {
    this.metrics = {
      totalAttempts: 0,
      successCount: 0,
      failureCount: 0,
      retryCount: 0,
      totalDuration: 0,
      errors: new Map()
    };
  }
}

// ═══════════════════════════════════════════════════════════
// UTILISATION
// ═══════════════════════════════════════════════════════════

const txnManager = new TransactionManager(client, {
  maxRetries: 5,
  initialBackoffMs: 50,
  enableLogging: true
});

// Exemple 1 : Utilisation simple
const result = await txnManager.execute(async (session) => {
  await db.accounts.updateOne(
    { _id: 'ACC-123' },
    { $inc: { balance: -100 } },
    { session }
  );

  await db.transactions.insertOne(
    { accountId: 'ACC-123', amount: -100, timestamp: new Date() },
    { session }
  );

  return { success: true };
});

if (result.success) {
  console.log(`Transaction réussie en ${result.attempts} tentative(s)`);
} else {
  console.error('Transaction échouée:', result.error.message);
}

// Exemple 2 : Avec callbacks
await txnManager.executeWithCallbacks(
  async (session) => {
    // Logique transactionnelle
    return { orderId: 'ORD-123' };
  },
  {
    onSuccess: async (result) => {
      console.log('Commande créée:', result.orderId);
      await sendConfirmationEmail(result.orderId);
    },
    onFailure: async (error) => {
      console.error('Échec de la commande:', error);
      await notifyAdmins(error);
    },
    config: {
      writeConcern: { w: 'majority', j: true },
      maxCommitTimeMS: 30000
    }
  }
);

// Exemple 3 : Monitoring des métriques
setInterval(() => {
  const metrics = txnManager.getMetrics();
  console.log('Transaction Metrics:', metrics);

  if (parseFloat(metrics.successRate) < 90) {
    console.warn('⚠️  Taux de succès < 90%');
  }
}, 60000); // Toutes les minutes
```

## Pattern 2 : Isolation des opérations transactionnelles

### Séparation des responsabilités

```javascript
/**
 * Repository Pattern - Encapsuler la logique transactionnelle
 */
class OrderRepository {
  constructor(db, txnManager) {
    this.db = db;
    this.txnManager = txnManager;
  }

  /**
   * Crée une commande dans une transaction
   */
  async createOrder(orderData) {
    // Validation AVANT la transaction
    this._validateOrderData(orderData);

    // Calculs AVANT la transaction
    const pricing = this._calculatePricing(orderData);
    const total = this._calculateTotal(pricing);

    // Transaction COURTE et FOCALISÉE
    return await this.txnManager.execute(async (session) => {
      // 1. Vérifier le stock (avec verrou)
      const stockCheck = await this._checkAndReserveStock(
        orderData.items,
        session
      );

      if (!stockCheck.success) {
        throw new Error(`Stock insuffisant: ${stockCheck.missing.join(', ')}`);
      }

      // 2. Créer la commande
      const order = await this.db.orders.insertOne(
        {
          userId: orderData.userId,
          items: orderData.items,
          pricing,
          total,
          status: 'pending',
          createdAt: new Date()
        },
        { session }
      );

      // 3. Créer l'entrée de paiement
      await this.db.payments.insertOne(
        {
          orderId: order.insertedId,
          amount: total,
          status: 'pending',
          createdAt: new Date()
        },
        { session }
      );

      return {
        orderId: order.insertedId,
        total,
        status: 'pending'
      };
    }, {
      writeConcern: { w: 'majority' },
      maxCommitTimeMS: 15000
    });
  }

  /**
   * Annule une commande
   */
  async cancelOrder(orderId, reason) {
    return await this.txnManager.execute(async (session) => {
      // 1. Récupérer la commande
      const order = await this.db.orders.findOne(
        { _id: orderId },
        { session }
      );

      if (!order) {
        throw new Error('Commande introuvable');
      }

      if (order.status === 'cancelled') {
        throw new Error('Commande déjà annulée');
      }

      // 2. Restaurer le stock
      await this._releaseStock(order.items, session);

      // 3. Marquer la commande comme annulée
      await this.db.orders.updateOne(
        { _id: orderId },
        {
          $set: {
            status: 'cancelled',
            cancelReason: reason,
            cancelledAt: new Date()
          }
        },
        { session }
      );

      // 4. Annuler le paiement
      await this.db.payments.updateOne(
        { orderId },
        {
          $set: {
            status: 'cancelled',
            cancelledAt: new Date()
          }
        },
        { session }
      );

      return { orderId, status: 'cancelled' };
    });
  }

  /**
   * Vérifie et réserve le stock (privé, appelé dans transaction)
   */
  async _checkAndReserveStock(items, session) {
    const missing = [];

    for (const item of items) {
      const product = await this.db.products.findOneAndUpdate(
        {
          _id: item.productId,
          stock: { $gte: item.quantity }
        },
        {
          $inc: { stock: -item.quantity }
        },
        {
          session,
          returnDocument: 'after'
        }
      );

      if (!product.value) {
        missing.push(item.productId);
      }
    }

    return {
      success: missing.length === 0,
      missing
    };
  }

  /**
   * Libère le stock (privé, appelé dans transaction)
   */
  async _releaseStock(items, session) {
    for (const item of items) {
      await this.db.products.updateOne(
        { _id: item.productId },
        { $inc: { stock: item.quantity } },
        { session }
      );
    }
  }

  /**
   * Validation (HORS transaction)
   */
  _validateOrderData(orderData) {
    if (!orderData.userId) {
      throw new Error('userId requis');
    }

    if (!orderData.items || orderData.items.length === 0) {
      throw new Error('Items requis');
    }

    for (const item of orderData.items) {
      if (!item.productId || !item.quantity || item.quantity <= 0) {
        throw new Error('Item invalide');
      }
    }
  }

  /**
   * Calculs (HORS transaction)
   */
  _calculatePricing(orderData) {
    // Logique de pricing complexe
    return orderData.items.map(item => ({
      productId: item.productId,
      quantity: item.quantity,
      unitPrice: item.price,
      subtotal: item.price * item.quantity
    }));
  }

  _calculateTotal(pricing) {
    return pricing.reduce((sum, item) => sum + item.subtotal, 0);
  }
}

// Utilisation
const orderRepo = new OrderRepository(db, txnManager);

try {
  const result = await orderRepo.createOrder({
    userId: 'USER-123',
    items: [
      { productId: 'PROD-1', quantity: 2, price: 29.99 },
      { productId: 'PROD-2', quantity: 1, price: 49.99 }
    ]
  });

  console.log('Commande créée:', result.orderId);
} catch (error) {
  console.error('Échec création commande:', error.message);
}
```

## Pattern 3 : Transactions idempotentes

### Garantir l'idempotence pour la sécurité des retries

```javascript
/**
 * Service de paiement idempotent
 */
class IdempotentPaymentService {
  constructor(db, txnManager) {
    this.db = db;
    this.txnManager = txnManager;
  }

  /**
   * Traite un paiement de manière idempotente
   * @param {string} idempotencyKey - Clé d'idempotence (UUID fourni par le client)
   */
  async processPayment(paymentData, idempotencyKey) {
    // 1. Vérifier si ce paiement a déjà été traité
    const existing = await this.db.payments.findOne({
      idempotencyKey
    });

    if (existing) {
      console.log('Paiement déjà traité (idempotent)');
      return {
        status: 'already_processed',
        payment: existing,
        idempotent: true
      };
    }

    // 2. Traiter le paiement dans une transaction
    const result = await this.txnManager.execute(async (session) => {
      // Double-check dans la transaction (éviter race condition)
      const doubleCheck = await this.db.payments.findOne(
        { idempotencyKey },
        { session }
      );

      if (doubleCheck) {
        return {
          status: 'already_processed',
          payment: doubleCheck,
          idempotent: true
        };
      }

      // Créer le paiement avec la clé d'idempotence
      const payment = await this.db.payments.insertOne(
        {
          idempotencyKey,  // Index unique
          orderId: paymentData.orderId,
          amount: paymentData.amount,
          method: paymentData.method,
          status: 'processing',
          createdAt: new Date()
        },
        { session }
      );

      // Simuler traitement du paiement externe
      // (en réalité, appel à une API de paiement)
      const externalResult = {
        success: true,
        transactionId: 'EXT-' + Date.now()
      };

      // Mettre à jour avec le résultat
      await this.db.payments.updateOne(
        { _id: payment.insertedId },
        {
          $set: {
            status: externalResult.success ? 'completed' : 'failed',
            externalTransactionId: externalResult.transactionId,
            processedAt: new Date()
          }
        },
        { session }
      );

      // Mettre à jour la commande
      await this.db.orders.updateOne(
        { _id: paymentData.orderId },
        {
          $set: {
            paymentStatus: externalResult.success ? 'paid' : 'payment_failed'
          }
        },
        { session }
      );

      return {
        status: 'processed',
        payment: {
          id: payment.insertedId,
          status: externalResult.success ? 'completed' : 'failed'
        },
        idempotent: false
      };
    });

    return result;
  }
}

// Utilisation avec retry automatique
const paymentService = new IdempotentPaymentService(db, txnManager);

// Le client génère une clé d'idempotence unique
const idempotencyKey = uuidv4();

// Premier appel
const result1 = await paymentService.processPayment(
  {
    orderId: 'ORD-123',
    amount: 99.99,
    method: 'credit_card'
  },
  idempotencyKey
);
console.log('Résultat 1:', result1.status); // 'processed'

// Retry (même clé) - paiement non dupliqué
const result2 = await paymentService.processPayment(
  {
    orderId: 'ORD-123',
    amount: 99.99,
    method: 'credit_card'
  },
  idempotencyKey
);
console.log('Résultat 2:', result2.status); // 'already_processed'
console.log('Idempotent:', result2.idempotent); // true
```

## Pattern 4 : Transactions avec compensation (Saga)

### Implémentation d'un Saga coordinator

```javascript
/**
 * Saga Coordinator - Gestion de processus long avec compensation
 */
class SagaCoordinator {
  constructor(db, txnManager) {
    this.db = db;
    this.txnManager = txnManager;
  }

  /**
   * Exécute un Saga avec compensation automatique en cas d'échec
   */
  async executeSaga(sagaDefinition) {
    const sagaId = new ObjectId();
    const completedSteps = [];

    // Créer l'enregistrement Saga
    await this.db.sagas.insertOne({
      _id: sagaId,
      name: sagaDefinition.name,
      status: 'running',
      steps: [],
      startedAt: new Date()
    });

    try {
      // Exécuter chaque étape
      for (const step of sagaDefinition.steps) {
        console.log(`[Saga] Exécution de ${step.name}...`);

        const stepResult = await this._executeStep(sagaId, step);
        completedSteps.push({ step, result: stepResult });

        // Enregistrer l'étape complétée
        await this.db.sagas.updateOne(
          { _id: sagaId },
          {
            $push: {
              steps: {
                name: step.name,
                status: 'completed',
                completedAt: new Date(),
                result: stepResult
              }
            }
          }
        );
      }

      // Saga complétée avec succès
      await this.db.sagas.updateOne(
        { _id: sagaId },
        {
          $set: {
            status: 'completed',
            completedAt: new Date()
          }
        }
      );

      console.log(`[Saga] ${sagaDefinition.name} complété avec succès`);

      return {
        success: true,
        sagaId,
        results: completedSteps.map(s => s.result)
      };

    } catch (error) {
      console.error(`[Saga] Échec à l'étape ${completedSteps.length + 1}:`, error.message);

      // Compenser les étapes déjà complétées (ordre inverse)
      await this._compensate(sagaId, completedSteps.reverse());

      // Marquer le Saga comme échoué
      await this.db.sagas.updateOne(
        { _id: sagaId },
        {
          $set: {
            status: 'compensated',
            error: error.message,
            compensatedAt: new Date()
          }
        }
      );

      return {
        success: false,
        sagaId,
        error: error.message
      };
    }
  }

  /**
   * Exécute une étape (dans une transaction locale)
   */
  async _executeStep(sagaId, step) {
    const result = await this.txnManager.execute(async (session) => {
      return await step.execute(session, { sagaId });
    });

    if (!result.success) {
      throw new Error(`Step ${step.name} failed: ${result.error?.message}`);
    }

    return result.result;
  }

  /**
   * Compense les étapes complétées
   */
  async _compensate(sagaId, completedSteps) {
    console.log(`[Saga] Début de la compensation (${completedSteps.length} étapes)`);

    for (const { step, result } of completedSteps) {
      if (step.compensate) {
        try {
          console.log(`[Saga] Compensation de ${step.name}...`);

          await this.txnManager.execute(async (session) => {
            return await step.compensate(session, result, { sagaId });
          });

          await this.db.sagas.updateOne(
            { _id: sagaId },
            {
              $push: {
                compensations: {
                  stepName: step.name,
                  status: 'compensated',
                  compensatedAt: new Date()
                }
              }
            }
          );

        } catch (compensationError) {
          console.error(
            `[Saga] Échec compensation de ${step.name}:`,
            compensationError.message
          );

          // Alerter les ops pour intervention manuelle
          await this._alertOps(sagaId, step.name, compensationError);
        }
      }
    }
  }

  /**
   * Alerte l'équipe ops pour intervention manuelle
   */
  async _alertOps(sagaId, stepName, error) {
    await this.db.manual_interventions.insertOne({
      sagaId,
      stepName,
      error: error.message,
      status: 'pending',
      createdAt: new Date()
    });

    // Envoyer notification (email, Slack, PagerDuty, etc.)
    console.error(`🚨 ALERTE OPS - Intervention manuelle requise pour Saga ${sagaId}`);
  }
}

// ═══════════════════════════════════════════════════════════
// DÉFINITION D'UN SAGA
// ═══════════════════════════════════════════════════════════

const orderSaga = {
  name: 'CreateOrderSaga',
  steps: [
    {
      name: 'reserve_stock',
      execute: async (session, context) => {
        const items = [
          { productId: 'PROD-1', quantity: 2 },
          { productId: 'PROD-2', quantity: 1 }
        ];

        for (const item of items) {
          const result = await db.products.findOneAndUpdate(
            {
              _id: item.productId,
              stock: { $gte: item.quantity }
            },
            {
              $inc: { stock: -item.quantity }
            },
            { session, returnDocument: 'after' }
          );

          if (!result.value) {
            throw new Error(`Stock insuffisant: ${item.productId}`);
          }
        }

        return { items };
      },
      compensate: async (session, stepResult) => {
        // Libérer le stock réservé
        for (const item of stepResult.items) {
          await db.products.updateOne(
            { _id: item.productId },
            { $inc: { stock: item.quantity } },
            { session }
          );
        }
      }
    },
    {
      name: 'create_order',
      execute: async (session, context) => {
        const order = await db.orders.insertOne(
          {
            userId: 'USER-123',
            items: context.previousResult?.items || [],
            status: 'pending',
            createdAt: new Date()
          },
          { session }
        );

        return { orderId: order.insertedId };
      },
      compensate: async (session, stepResult) => {
        // Annuler la commande
        await db.orders.updateOne(
          { _id: stepResult.orderId },
          { $set: { status: 'cancelled', cancelledAt: new Date() } },
          { session }
        );
      }
    },
    {
      name: 'process_payment',
      execute: async (session, context) => {
        // Simuler traitement paiement
        const payment = await db.payments.insertOne(
          {
            orderId: context.previousResult?.orderId,
            amount: 99.99,
            status: 'completed',
            createdAt: new Date()
          },
          { session }
        );

        return { paymentId: payment.insertedId };
      },
      compensate: async (session, stepResult) => {
        // Rembourser
        await db.payments.updateOne(
          { _id: stepResult.paymentId },
          { $set: { status: 'refunded', refundedAt: new Date() } },
          { session }
        );
      }
    }
  ]
};

// Exécution du Saga
const sagaCoordinator = new SagaCoordinator(db, txnManager);
const result = await sagaCoordinator.executeSaga(orderSaga);

if (result.success) {
  console.log('✅ Saga complété');
} else {
  console.log('❌ Saga échoué et compensé');
}
```

## Pattern 5 : Testing des transactions

### Framework de test pour transactions

```javascript
/**
 * Test helpers pour transactions
 */
class TransactionTestHelper {
  constructor(db) {
    this.db = db;
  }

  /**
   * Setup - Prépare un environnement de test propre
   */
  async setup() {
    // Nettoyer les collections de test
    await this.db.test_accounts.deleteMany({});
    await this.db.test_transactions.deleteMany({});

    // Insérer des données de test
    await this.db.test_accounts.insertMany([
      { _id: 'ACC-1', balance: 1000 },
      { _id: 'ACC-2', balance: 500 }
    ]);
  }

  /**
   * Teardown - Nettoie après les tests
   */
  async teardown() {
    await this.db.test_accounts.deleteMany({});
    await this.db.test_transactions.deleteMany({});
  }

  /**
   * Simule une contention pour tester les write conflicts
   */
  async simulateContention(documentId, numConcurrent = 5) {
    const promises = [];

    for (let i = 0; i < numConcurrent; i++) {
      promises.push(
        (async () => {
          const session = client.startSession();
          try {
            await session.withTransaction(async () => {
              await this.db.test_accounts.updateOne(
                { _id: documentId },
                { $inc: { counter: 1 } },
                { session }
              );
            });
            return { success: true, id: i };
          } catch (error) {
            return { success: false, id: i, error: error.message };
          } finally {
            await session.endSession();
          }
        })()
      );
    }

    const results = await Promise.allSettled(promises);

    const succeeded = results.filter(r => r.value?.success).length;
    const failed = results.filter(r => !r.value?.success).length;

    return { succeeded, failed, total: numConcurrent };
  }

  /**
   * Vérifie l'isolation snapshot
   */
  async verifySnapshotIsolation() {
    const session = client.startSession();

    try {
      await session.withTransaction(async () => {
        // Lire la valeur initiale
        const initial = await this.db.test_accounts.findOne(
          { _id: 'ACC-1' },
          { session }
        );

        console.log('Valeur initiale dans la transaction:', initial.balance);

        // Modifier depuis HORS transaction
        await this.db.test_accounts.updateOne(
          { _id: 'ACC-1' },
          { $set: { balance: 9999 } }
        );

        console.log('Valeur modifiée HORS transaction: 9999');

        // Re-lire depuis la transaction
        const second = await this.db.test_accounts.findOne(
          { _id: 'ACC-1' },
          { session }
        );

        console.log('Valeur dans la transaction (après modif externe):', second.balance);

        // Vérifier l'isolation
        if (initial.balance === second.balance) {
          console.log('✅ Snapshot isolation vérifiée');
          return true;
        } else {
          console.log('❌ Snapshot isolation échouée');
          return false;
        }
      }, {
        readConcern: { level: 'snapshot' }
      });
    } finally {
      await session.endSession();
    }
  }

  /**
   * Teste le rollback automatique
   */
  async testRollback() {
    const balanceBefore = await this.db.test_accounts.findOne({ _id: 'ACC-1' });
    console.log('Balance avant:', balanceBefore.balance);

    const session = client.startSession();

    try {
      await session.withTransaction(async () => {
        // Modifier le compte
        await this.db.test_accounts.updateOne(
          { _id: 'ACC-1' },
          { $inc: { balance: -500 } },
          { session }
        );

        // Vérifier dans la transaction
        const inTxn = await this.db.test_accounts.findOne(
          { _id: 'ACC-1' },
          { session }
        );
        console.log('Balance dans la transaction:', inTxn.balance);

        // Forcer une erreur pour déclencher rollback
        throw new Error('Erreur simulée pour tester rollback');
      });
    } catch (error) {
      console.log('Transaction échouée (attendu):', error.message);
    } finally {
      await session.endSession();
    }

    // Vérifier que la balance est inchangée
    const balanceAfter = await this.db.test_accounts.findOne({ _id: 'ACC-1' });
    console.log('Balance après rollback:', balanceAfter.balance);

    if (balanceBefore.balance === balanceAfter.balance) {
      console.log('✅ Rollback automatique vérifié');
      return true;
    } else {
      console.log('❌ Rollback a échoué');
      return false;
    }
  }
}

// Exécution des tests
const testHelper = new TransactionTestHelper(db);

console.log('=== Test 1: Snapshot Isolation ===');
await testHelper.setup();
await testHelper.verifySnapshotIsolation();

console.log('\n=== Test 2: Rollback Automatique ===');
await testHelper.setup();
await testHelper.testRollback();

console.log('\n=== Test 3: Contention ===');
await testHelper.setup();
const contentionResult = await testHelper.simulateContention('ACC-1', 10);
console.log('Résultat contention:', contentionResult);

await testHelper.teardown();
```

## Bonnes pratiques de monitoring

### Dashboard de santé transactionnelle

```javascript
/**
 * TransactionHealthMonitor - Surveillance de la santé des transactions
 */
class TransactionHealthMonitor {
  constructor(db) {
    this.db = db;
    this.metrics = {
      lastCheck: null,
      alerts: []
    };
  }

  /**
   * Exécute un check complet de santé
   */
  async performHealthCheck() {
    this.metrics.lastCheck = new Date();
    this.metrics.alerts = [];

    const checks = [
      this.checkActiveTransactions(),
      this.checkPreparedTransactions(),
      this.checkAbortRate(),
      this.checkReplicationLag(),
      this.checkCachePressure(),
      this.checkLockWaits()
    ];

    const results = await Promise.all(checks);

    const report = {
      timestamp: this.metrics.lastCheck,
      healthy: this.metrics.alerts.length === 0,
      alerts: this.metrics.alerts,
      checks: {
        activeTransactions: results[0],
        preparedTransactions: results[1],
        abortRate: results[2],
        replicationLag: results[3],
        cachePressure: results[4],
        lockWaits: results[5]
      }
    };

    return report;
  }

  async checkActiveTransactions() {
    const status = await this.db.serverStatus();
    const active = status.transactions?.currentActive || 0;

    if (active > 100) {
      this._addAlert('HIGH', `Trop de transactions actives: ${active}`);
    }

    return { active, threshold: 100, ok: active <= 100 };
  }

  async checkPreparedTransactions() {
    const status = await this.db.serverStatus();
    const prepared = status.transactions?.currentPrepared || 0;

    if (prepared > 10) {
      this._addAlert('CRITICAL', `Transactions bloquées en prepare: ${prepared}`);
    }

    return { prepared, threshold: 10, ok: prepared <= 10 };
  }

  async checkAbortRate() {
    const status = await this.db.serverStatus();
    const total = status.transactions?.totalStarted || 1;
    const aborted = status.transactions?.totalAborted || 0;
    const abortRate = (aborted / total * 100).toFixed(2);

    if (abortRate > 10) {
      this._addAlert('MEDIUM', `Taux d'abort élevé: ${abortRate}%`);
    }

    return { abortRate: `${abortRate}%`, threshold: '10%', ok: abortRate <= 10 };
  }

  async checkReplicationLag() {
    const status = await this.db.adminCommand({ replSetGetStatus: 1 });
    const primary = status.members?.find(m => m.stateStr === 'PRIMARY');
    const secondaries = status.members?.filter(m => m.stateStr === 'SECONDARY') || [];

    if (!primary || secondaries.length === 0) {
      return { lag: 0, ok: true };
    }

    const primaryOptime = primary.optimeDate.getTime();
    const maxLag = Math.max(
      ...secondaries.map(s => primaryOptime - s.optimeDate.getTime())
    );

    if (maxLag > 5000) {
      this._addAlert('MEDIUM', `Replication lag élevé: ${maxLag}ms`);
    }

    return { lagMs: maxLag, threshold: 5000, ok: maxLag <= 5000 };
  }

  async checkCachePressure() {
    const status = await this.db.serverStatus();
    const cache = status.wiredTiger?.cache;

    if (!cache) {
      return { ok: true };
    }

    const maxBytes = cache['maximum bytes configured'];
    const currentBytes = cache['bytes currently in the cache'];
    const usage = (currentBytes / maxBytes * 100).toFixed(2);

    if (usage > 95) {
      this._addAlert('HIGH', `Cache saturé: ${usage}%`);
    }

    return { usage: `${usage}%`, threshold: '95%', ok: usage <= 95 };
  }

  async checkLockWaits() {
    const currentOp = await this.db.currentOp({ waitingForLock: true });
    const waiting = currentOp.inprog?.length || 0;

    if (waiting > 20) {
      this._addAlert('HIGH', `Trop d'opérations en attente de verrou: ${waiting}`);
    }

    return { waiting, threshold: 20, ok: waiting <= 20 };
  }

  _addAlert(severity, message) {
    this.metrics.alerts.push({
      severity,
      message,
      timestamp: new Date()
    });
  }

  /**
   * Génère un rapport formaté
   */
  formatReport(report) {
    let output = '\n';
    output += '═══════════════════════════════════════════════\n';
    output += '    TRANSACTION HEALTH REPORT\n';
    output += '═══════════════════════════════════════════════\n';
    output += `Timestamp: ${report.timestamp.toISOString()}\n`;
    output += `Status: ${report.healthy ? '✅ HEALTHY' : '⚠️  ISSUES DETECTED'}\n`;
    output += '\n';

    if (report.alerts.length > 0) {
      output += 'ALERTS:\n';
      report.alerts.forEach(alert => {
        const icon = alert.severity === 'CRITICAL' ? '🚨' :
                     alert.severity === 'HIGH' ? '⚠️ ' : '⚡';
        output += `  ${icon} [${alert.severity}] ${alert.message}\n`;
      });
      output += '\n';
    }

    output += 'CHECKS:\n';
    Object.entries(report.checks).forEach(([name, check]) => {
      const status = check.ok ? '✅' : '❌';
      output += `  ${status} ${name}: ${JSON.stringify(check)}\n`;
    });

    output += '═══════════════════════════════════════════════\n';

    return output;
  }
}

// Utilisation
const healthMonitor = new TransactionHealthMonitor(db);

// Check périodique (toutes les minutes)
setInterval(async () => {
  const report = await healthMonitor.performHealthCheck();

  if (!report.healthy) {
    console.log(healthMonitor.formatReport(report));

    // Envoyer notification si nécessaire
    if (report.alerts.some(a => a.severity === 'CRITICAL')) {
      await sendPageDutyAlert(report);
    }
  }
}, 60000);
```

## Checklist de production complète

### Liste de vérification avant déploiement

```markdown
## Checklist : Transactions en production

### Conception
- [ ] Transactions durée < 5 secondes (idéal < 1s)
- [ ] Nombre d'opérations < 500 par transaction
- [ ] Taille totale < 8 MB (50% de la limite)
- [ ] Logique métier HORS transaction
- [ ] Appels externes HORS transaction
- [ ] Schéma optimisé pour minimiser transactions multi-shards

### Configuration
- [ ] ReadConcern approprié défini
- [ ] WriteConcern approprié défini
- [ ] maxCommitTimeMS configuré (ne pas laisser défaut)
- [ ] transactionLifetimeLimitSeconds ajusté si nécessaire
- [ ] WiredTiger cache dimensionné correctement

### Gestion des erreurs
- [ ] Retry logic implémenté (3-5 tentatives)
- [ ] Backoff exponentiel avec jitter
- [ ] Gestion TransientTransactionError
- [ ] Gestion WriteConflict
- [ ] Logging approprié des échecs

### Idempotence
- [ ] Clés d'idempotence pour opérations critiques
- [ ] Index unique sur clé d'idempotence
- [ ] Vérification de duplication avant et dans transaction
- [ ] Documentation des garanties d'idempotence

### Monitoring
- [ ] Métriques de latence (P50, P95, P99)
- [ ] Métriques de throughput
- [ ] Taux de succès/échec
- [ ] Taux de write conflict
- [ ] Replication lag monitoring
- [ ] Cache utilization monitoring
- [ ] Alertes configurées (PagerDuty, Slack, etc.)

### Tests
- [ ] Tests unitaires de la logique transactionnelle
- [ ] Tests d'intégration avec vraie DB
- [ ] Tests de charge avec transactions concurrentes
- [ ] Tests de contention (write conflicts)
- [ ] Tests de rollback
- [ ] Tests de timeout
- [ ] Tests de panne (failover)

### Documentation
- [ ] Architecture transactionnelle documentée
- [ ] Diagrammes de flux transactionnels
- [ ] Runbook pour incidents
- [ ] Procédures de rollback manuel
- [ ] SLA et objectifs de performance définis

### Sécurité
- [ ] Validation des entrées AVANT transaction
- [ ] Gestion des timeouts pour éviter DoS
- [ ] Rate limiting si nécessaire
- [ ] Audit logging des transactions critiques

### Performance
- [ ] Benchmark effectué en conditions réalistes
- [ ] Index appropriés sur tous les champs de requête
- [ ] bulkWrite utilisé quand possible
- [ ] Projections pour limiter données lues
- [ ] Connection pooling optimisé

### Disaster Recovery
- [ ] Procédure de compensation manuelle documentée
- [ ] Backups testés
- [ ] Procédure de récupération après incident
- [ ] Plan de rollback des changements
```

## Anti-patterns à éviter absolument

### Anti-pattern 1 : Transactions imbriquées

```javascript
// ❌ TRÈS MAUVAIS : Transactions imbriquées
async function nestedTransactionAntiPattern() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // Opération 1
      await db.collection.insertOne({ data: 1 }, { session });

      // ❌ ERREUR : Créer une nouvelle session dans une transaction
      const session2 = client.startSession();
      try {
        await session2.withTransaction(async () => {
          await db.collection.insertOne({ data: 2 }, { session: session2 });
        });
      } finally {
        await session2.endSession();
      }
    });
  } finally {
    await session.endSession();
  }
}

// ✅ BON : Une seule transaction, opérations séquentielles
async function singleTransactionGoodPattern() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      await db.collection.insertOne({ data: 1 }, { session });
      await db.collection.insertOne({ data: 2 }, { session });
    });
  } finally {
    await session.endSession();
  }
}
```

### Anti-pattern 2 : Oublier de passer la session

```javascript
// ❌ MAUVAIS : Oublier de passer la session
async function forgottenSessionAntiPattern() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // ❌ ERREUR : Pas de session passée
      await db.accounts.updateOne(
        { _id: 'ACC-1' },
        { $inc: { balance: -100 } }
        // Manque : { session }
      );

      // Cette opération n'est PAS dans la transaction !
      // Elle sera commitée immédiatement, pas atomiquement
    });
  } finally {
    await session.endSession();
  }
}

// ✅ BON : Toujours passer la session
async function correctSessionUsage() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      await db.accounts.updateOne(
        { _id: 'ACC-1' },
        { $inc: { balance: -100 } },
        { session }  // ✅ Session passée
      );
    });
  } finally {
    await session.endSession();
  }
}
```

### Anti-pattern 3 : Transactions pour opérations simples

```javascript
// ❌ INUTILE : Transaction pour une seule opération atomique
async function unnecessaryTransaction() {
  const session = client.startSession();

  try {
    await session.withTransaction(async () => {
      // Une seule opération - transaction inutile !
      await db.counter.updateOne(
        { _id: 'global' },
        { $inc: { count: 1 } },
        { session }
      );
    });
  } finally {
    await session.endSession();
  }
}

// ✅ BON : Opération atomique simple sans transaction
async function atomicOperationWithoutTransaction() {
  // MongoDB garantit l'atomicité au niveau document
  await db.counter.updateOne(
    { _id: 'global' },
    { $inc: { count: 1 } }
  );
  // Plus rapide, plus simple, même garantie pour cette opération
}
```

## Conclusion : Les principes d'or

Les bonnes pratiques transactionnelles peuvent être résumées en **10 principes d'or** :

```
LES 10 COMMANDEMENTS DES TRANSACTIONS MONGODB

1. MINIMALISME
   → La transaction la plus performante est celle qu'on n'a pas à faire

2. RAPIDITÉ
   → Garder les transactions < 1 seconde
   → Libérer les verrous au plus vite

3. IDEMPOTENCE
   → Rendre les opérations critiques idempotentes
   → Permettre les retries sans duplication

4. ISOLATION
   → Séparer logique métier et logique transactionnelle
   → Calculs HORS transaction

5. RETRY
   → Implémenter retry avec backoff exponentiel
   → Gérer les erreurs transitoires gracieusement

6. MONITORING
   → Mesurer constamment
   → Alerter proactivement

7. TESTING
   → Tester sous charge
   → Simuler les échecs

8. DOCUMENTATION
   → Documenter les choix de cohérence
   → Maintenir des runbooks

9. SIMPLICITÉ
   → Code simple > Code clever
   → Patterns éprouvés > Innovations hasardeuses

10. ÉVOLUTION
    → Mesurer et améliorer continuellement
    → Adapter selon les besoins réels
```

---

**Points clés à retenir** :

- Utiliser un TransactionManager réutilisable pour standardiser
- Implémenter le Repository Pattern pour isoler la logique
- Rendre les opérations critiques idempotentes
- Saga pattern pour processus long avec compensation
- Testing rigoureux : isolation, rollback, contention
- Monitoring actif avec alertes
- Checklist de production complète
- Éviter les anti-patterns courants
- Documentation et runbooks essentiels
- Amélioration continue basée sur les métriques

⏭️ [Réplication](/09-replication/README.md)
