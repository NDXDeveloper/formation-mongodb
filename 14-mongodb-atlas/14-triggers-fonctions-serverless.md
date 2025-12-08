🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.14 Triggers et Fonctions Serverless

## Introduction

Les **Triggers** et **Functions** d'Atlas App Services constituent une plateforme serverless complète pour l'automatisation et l'orchestration event-driven. Réagissez aux changements de données en temps réel, planifiez des tâches récurrentes, orchestrez des workflows complexes, et intégrez des services externes : tout en code JavaScript/Node.js sans gérer de serveurs. Cette section approfondit les patterns avancés et les architectures event-driven pour des systèmes production-grade.

### 🎯 Objectifs de cette Section

- Maîtriser les patterns avancés de triggers
- Orchestrer des workflows complexes multi-étapes
- Implémenter error handling et retry robuste
- Monitorer et observer les fonctions serverless
- Déployer via CI/CD et Infrastructure-as-Code
- Construire des architectures event-driven scalables
- Optimiser performances et coûts

---

## 🎯 Architecture Event-Driven

### Vue d'Ensemble

```
┌────────────────────────────────────────────────────────────────────────┐
│              EVENT-DRIVEN ARCHITECTURE WITH TRIGGERS                   │
├────────────────────────────────────────────────────────────────────────┤
│
│  ÉVÉNEMENTS
│  ┌──────────────────────────────────────────────────────────────────┐
│  │
│  │  Database     Scheduled      Authentication    HTTP
│  │  Changes      Events         Events            Webhooks
│  │  ────────     ────────       ────────          ────────
│  │  INSERT       Cron jobs      User signup       External API
│  │  UPDATE       Daily tasks    Login             Payment gateway
│  │  DELETE       Hourly checks  Logout            3rd party service
│  │  REPLACE      Maintenance    Delete account    IoT devices
│  │
│  └────┬──────────────┬────────────────┬─────────────────┬───────────┘
│       │              │                │                 │
│       ▼              ▼                ▼                 ▼
│  ┌──────────────────────────────────────────────────────────────────┐
│  │                      TRIGGER ROUTER
│  │  • Event matching
│  │  • Filter evaluation
│  │  • Function invocation
│  │  • Retry management
│  └────┬──────────────┬────────────────┬─────────────────┬───────────┘
│       │              │                │                 │
│       ▼              ▼                ▼                 ▼
│  ┌─────────┐  ┌─────────┐  ┌──────────┐  ┌──────────────────────┐
│  │Function │  │Function │  │Function  │  │Function Orchestrator │
│  │   A     │  │   B     │  │   C      │  │  (Complex Workflow)  │
│  └────┬────┘  └────┬────┘  └────┬─────┘  └──────────┬───────────┘
│       │            │            │                   │
│       └────────────┴────────────┴───────────────────┘
│                                 ▼
│  ┌──────────────────────────────────────────────────────────────────┐
│  │                    SIDE EFFECTS
│  │  • MongoDB operations (CRUD)
│  │  • External API calls (REST, GraphQL)
│  │  • Email/SMS notifications
│  │  • Message queues (SQS, Kafka)
│  │  • Other triggers invocation (chaining)
│  └──────────────────────────────────────────────────────────────────┘
│
│  AVANTAGES:
│  ✅ Découplage (loose coupling)
│  ✅ Scalabilité automatique
│  ✅ Résilience (retry, error handling)
│  ✅ Observabilité (logs, metrics)
│  ✅ Coût optimisé (pay per execution)
│
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Database Triggers Avancés

### Patterns de Triggers

```
┌───────────────────────────────────────────────────────────────────────┐
│                    DATABASE TRIGGER PATTERNS                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. AUDIT TRAIL (Traçabilité)                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Événement: INSERT, UPDATE, DELETE sur orders                     │ │
│  │ Action: Créer log d'audit avec before/after                      │ │
│  │                                                                  │ │
│  │ Use case: Compliance, forensics, debugging                       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  2. DATA ENRICHMENT (Enrichissement)                                  │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Événement: INSERT sur users                                      │ │
│  │ Action: Appel API externe pour géolocalisation, enrichir profil  │ │
│  │                                                                  │ │
│  │ Use case: Profile completion, data augmentation                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  3. CASCADE OPERATIONS (Opérations en cascade)                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Événement: DELETE sur users                                      │ │
│  │ Action: Supprimer orders, sessions, preferences associés         │ │
│  │                                                                  │ │
│  │ Use case: Data cleanup, referential integrity                    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  4. MATERIALIZED VIEWS (Vues matérialisées)                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Événement: INSERT/UPDATE sur orders                              │ │
│  │ Action: Mettre à jour aggregated_stats (totals, counts)          │ │
│  │                                                                  │ │
│  │ Use case: Performance optimization, real-time dashboards         │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  5. EVENT SOURCING (Source d'événements)                              │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Événement: Tout changement sur entities                          │ │
│  │ Action: Append to event_log (immutable, ordered)                 │ │
│  │                                                                  │ │
│  │ Use case: Event store, time-travel queries, replay               │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  6. NOTIFICATION DISPATCH (Distribution notifications)                │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Événement: UPDATE status="shipped" sur orders                    │ │
│  │ Action: Email + push notification au client                      │ │
│  │                                                                  │ │
│  │ Use case: User engagement, transactional messages                │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Implémentation Audit Trail Robuste

```javascript
// Trigger: Audit complet avec before/after states

exports = async function(changeEvent) {
  const {
    operationType,
    fullDocument,
    fullDocumentBeforeChange,
    documentKey,
    ns,
    clusterTime
  } = changeEvent;

  // Collection d'audit
  const auditCollection = context.services.get("mongodb-atlas")
    .db("audit")
    .collection("change_log");

  // Construire entrée d'audit
  const auditEntry = {
    // Métadonnées événement
    timestamp: new Date(clusterTime.getTime() * 1000),
    operation: operationType,
    collection: `${ns.db}.${ns.coll}`,
    documentId: documentKey._id,

    // User context (si authentifié)
    userId: context.user ? context.user.id : null,
    userEmail: context.user ? context.user.data.email : null,

    // Before/After states
    before: fullDocumentBeforeChange || null,
    after: fullDocument || null,

    // Changements détaillés (delta)
    changes: operationType === "update"
      ? calculateChanges(fullDocumentBeforeChange, fullDocument)
      : null,

    // Contexte applicatif
    source: context.values.get("appVersion") || "unknown",
    environment: context.environment.values.environment || "production"
  };

  try {
    await auditCollection.insertOne(auditEntry);
    console.log(`Audit log created for ${operationType} on ${documentKey._id}`);
  } catch (error) {
    // Logging critique - ne pas bloquer l'opération source
    console.error("Failed to create audit log:", error);

    // Optionnel: Dead letter queue pour retry
    await context.functions.execute("sendToDeadLetterQueue", {
      type: "audit_failure",
      data: auditEntry,
      error: error.message
    });
  }
};

// Helper: Calculer différences entre before/after
function calculateChanges(before, after) {
  if (!before || !after) return null;

  const changes = {};
  const allKeys = new Set([...Object.keys(before), ...Object.keys(after)]);

  for (const key of allKeys) {
    if (JSON.stringify(before[key]) !== JSON.stringify(after[key])) {
      changes[key] = {
        from: before[key],
        to: after[key]
      };
    }
  }

  return Object.keys(changes).length > 0 ? changes : null;
}
```

### Data Enrichment avec Rate Limiting

```javascript
// Trigger: Enrichir profil utilisateur avec API externe

exports = async function(changeEvent) {
  const { operationType, fullDocument } = changeEvent;

  // Seulement sur INSERT
  if (operationType !== "insert") return;

  const user = fullDocument;

  // Éviter re-enrichment si déjà fait
  if (user.enriched) return;

  try {
    // Rate limiting: Vérifier quota disponible
    const rateLimiter = await checkRateLimit("geocoding_api", 1000); // 1000/day
    if (!rateLimiter.allowed) {
      console.warn(`Rate limit exceeded for geocoding API`);
      return; // Skip enrichment, retry demain
    }

    // Appel API géolocalisation
    const geoData = await enrichWithGeolocation(user.email, user.ipAddress);

    // Appel API données démographiques
    const demoData = await enrichWithDemographics(user.email);

    // Mise à jour document utilisateur
    const users = context.services.get("mongodb-atlas")
      .db("mydb")
      .collection("users");

    await users.updateOne(
      { _id: user._id },
      {
        $set: {
          enriched: true,
          enrichedAt: new Date(),
          geolocation: geoData,
          demographics: demoData
        }
      }
    );

    console.log(`User ${user._id} enriched successfully`);

  } catch (error) {
    console.error(`Failed to enrich user ${user._id}:`, error);

    // Marquer pour retry ultérieur
    await users.updateOne(
      { _id: user._id },
      {
        $set: {
          enrichmentFailed: true,
          enrichmentError: error.message,
          enrichmentRetryAt: new Date(Date.now() + 3600000) // 1h
        }
      }
    );
  }
};

// Helper: Rate limiting simple
async function checkRateLimit(key, maxPerDay) {
  const rateLimits = context.services.get("mongodb-atlas")
    .db("system")
    .collection("rate_limits");

  const today = new Date().toISOString().split('T')[0];
  const limitDoc = await rateLimits.findOne({ key, date: today });

  if (!limitDoc) {
    await rateLimits.insertOne({ key, date: today, count: 1 });
    return { allowed: true, remaining: maxPerDay - 1 };
  }

  if (limitDoc.count >= maxPerDay) {
    return { allowed: false, remaining: 0 };
  }

  await rateLimits.updateOne(
    { key, date: today },
    { $inc: { count: 1 } }
  );

  return { allowed: true, remaining: maxPerDay - limitDoc.count - 1 };
}
```

---

## ⏰ Scheduled Triggers Avancés

### Patterns de Scheduling

```
┌───────────────────────────────────────────────────────────────────────┐
│                   SCHEDULED TRIGGER PATTERNS                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  CRON EXPRESSIONS COURANTES:                                          │
│  ─────────────────────────────────────────────────────────────────────│
│  "0 0 * * *"      → Daily at midnight                                 │
│  "0 */6 * * *"    → Every 6 hours                                     │
│  "*/15 * * * *"   → Every 15 minutes                                  │
│  "0 9 * * 1"      → Every Monday at 9 AM                              │
│  "0 0 1 * *"      → First day of month at midnight                    │
│                                                                       │
│  USE CASES:                                                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 1. Data Cleanup (Nettoyage)                                      │ │
│  │    • Delete old logs (> 30 days)                                 │ │
│  │    • Archive completed orders                                    │ │
│  │    • Purge temporary files                                       │ │
│  │    Schedule: Daily at 2 AM                                       │ │
│  ├──────────────────────────────────────────────────────────────────┤ │
│  │ 2. Report Generation (Rapports)                                  │ │
│  │    • Daily sales report                                          │ │
│  │    • Weekly analytics summary                                    │ │
│  │    • Monthly financial close                                     │ │
│  │    Schedule: Variable selon type                                 │ │
│  ├──────────────────────────────────────────────────────────────────┤ │
│  │ 3. Data Sync (Synchronisation)                                   │ │
│  │    • Sync to data warehouse                                      │ │
│  │    • Export to S3/Data Lake                                      │ │
│  │    • Update external systems                                     │ │
│  │    Schedule: Hourly or continuous                                │ │
│  ├──────────────────────────────────────────────────────────────────┤ │
│  │ 4. Health Checks (Surveillance)                                  │ │
│  │    • Monitor system metrics                                      │ │
│  │    • Check service availability                                  │ │
│  │    • Alert on anomalies                                          │ │
│  │    Schedule: Every 5 minutes                                     │ │
│  ├──────────────────────────────────────────────────────────────────┤ │
│  │ 5. Batch Processing (Traitement par lots)                        │ │
│  │    • Process pending jobs                                        │ │
│  │    • Send scheduled notifications                                │ │
│  │    • Update materialized views                                   │ │
│  │    Schedule: Every 15 minutes                                    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Scheduled Trigger avec Checkpointing

```javascript
// Trigger: Traitement batch incrémental avec checkpointing

exports = async function() {
  const startTime = Date.now();
  const maxExecutionTime = 80000; // 80 secondes (limite: 90s)

  const db = context.services.get("mongodb-atlas").db("mydb");
  const jobsCollection = db.collection("pending_jobs");
  const checkpointCollection = db.collection("job_checkpoints");

  // Récupérer dernier checkpoint
  let checkpoint = await checkpointCollection.findOne({
    jobType: "daily_processing"
  });

  if (!checkpoint) {
    checkpoint = {
      jobType: "daily_processing",
      lastProcessedId: null,
      lastProcessedAt: null,
      processedCount: 0
    };
  }

  let processedCount = 0;
  let lastProcessedId = checkpoint.lastProcessedId;

  try {
    // Query incrémentale: Traiter jobs après dernier checkpoint
    const query = lastProcessedId
      ? { _id: { $gt: lastProcessedId }, status: "pending" }
      : { status: "pending" };

    const jobs = await jobsCollection
      .find(query)
      .sort({ _id: 1 })
      .limit(100)
      .toArray();

    console.log(`Found ${jobs.length} pending jobs to process`);

    // Traiter jobs avec time limit
    for (const job of jobs) {
      // Check time limit
      if (Date.now() - startTime > maxExecutionTime) {
        console.log(`Time limit reached. Stopping at job ${job._id}`);
        break;
      }

      try {
        // Process job (votre logique métier)
        await processJob(job);

        // Marquer comme traité
        await jobsCollection.updateOne(
          { _id: job._id },
          {
            $set: {
              status: "completed",
              processedAt: new Date()
            }
          }
        );

        processedCount++;
        lastProcessedId = job._id;

      } catch (error) {
        console.error(`Failed to process job ${job._id}:`, error);

        // Marquer comme failed
        await jobsCollection.updateOne(
          { _id: job._id },
          {
            $set: {
              status: "failed",
              error: error.message,
              failedAt: new Date()
            },
            $inc: { retryCount: 1 }
          }
        );
      }
    }

    // Sauvegarder checkpoint
    await checkpointCollection.updateOne(
      { jobType: "daily_processing" },
      {
        $set: {
          lastProcessedId: lastProcessedId,
          lastProcessedAt: new Date(),
          processedCount: checkpoint.processedCount + processedCount
        }
      },
      { upsert: true }
    );

    console.log(`Processed ${processedCount} jobs. Checkpoint saved.`);

    return {
      success: true,
      processedCount,
      totalProcessed: checkpoint.processedCount + processedCount
    };

  } catch (error) {
    console.error("Batch processing failed:", error);
    throw error;
  }
};

async function processJob(job) {
  // Votre logique métier
  // Ex: Envoyer email, appeler API, transformer données, etc.

  // Simuler traitement
  await new Promise(resolve => setTimeout(resolve, 100));

  console.log(`Job ${job._id} processed`);
}
```

---

## 🔗 Orchestration de Workflows

### Pattern Chain of Responsibility

```javascript
// Workflow: Order Processing en plusieurs étapes

// ÉTAPE 1: Trigger sur INSERT order
exports = async function(changeEvent) {
  const order = changeEvent.fullDocument;

  if (changeEvent.operationType !== "insert") return;

  console.log(`New order received: ${order._id}`);

  // Démarrer workflow
  await context.functions.execute("processOrderWorkflow", order._id);
};

// FONCTION: Orchestrateur de workflow
exports = async function(orderId) {
  const db = context.services.get("mongodb-atlas").db("mydb");
  const orders = db.collection("orders");

  try {
    // Étape 1: Validation
    console.log(`Step 1: Validating order ${orderId}`);
    const validationResult = await context.functions.execute(
      "validateOrder",
      orderId
    );

    if (!validationResult.valid) {
      await orders.updateOne(
        { _id: orderId },
        {
          $set: {
            status: "validation_failed",
            validationError: validationResult.error
          }
        }
      );
      return { success: false, step: "validation" };
    }

    // Étape 2: Vérifier inventaire
    console.log(`Step 2: Checking inventory for order ${orderId}`);
    const inventoryCheck = await context.functions.execute(
      "checkInventory",
      orderId
    );

    if (!inventoryCheck.available) {
      await orders.updateOne(
        { _id: orderId },
        { $set: { status: "out_of_stock" } }
      );

      // Notifier client
      await context.functions.execute("notifyOutOfStock", orderId);
      return { success: false, step: "inventory" };
    }

    // Étape 3: Traiter paiement
    console.log(`Step 3: Processing payment for order ${orderId}`);
    const paymentResult = await context.functions.execute(
      "processPayment",
      orderId
    );

    if (!paymentResult.success) {
      await orders.updateOne(
        { _id: orderId },
        {
          $set: {
            status: "payment_failed",
            paymentError: paymentResult.error
          }
        }
      );
      return { success: false, step: "payment" };
    }

    // Étape 4: Réserver inventaire
    console.log(`Step 4: Reserving inventory for order ${orderId}`);
    await context.functions.execute("reserveInventory", orderId);

    // Étape 5: Créer commande d'expédition
    console.log(`Step 5: Creating shipment for order ${orderId}`);
    const shipmentId = await context.functions.execute(
      "createShipment",
      orderId
    );

    // Étape 6: Confirmer commande
    await orders.updateOne(
      { _id: orderId },
      {
        $set: {
          status: "confirmed",
          confirmedAt: new Date(),
          shipmentId: shipmentId,
          paymentTransactionId: paymentResult.transactionId
        }
      }
    );

    // Étape 7: Notification client
    await context.functions.execute("sendOrderConfirmation", orderId);

    console.log(`Order ${orderId} processed successfully`);
    return { success: true, shipmentId };

  } catch (error) {
    console.error(`Workflow failed for order ${orderId}:`, error);

    // Marquer comme erreur et programmer retry
    await orders.updateOne(
      { _id: orderId },
      {
        $set: {
          status: "processing_error",
          error: error.message,
          retryAt: new Date(Date.now() + 300000) // Retry dans 5 min
        }
      }
    );

    throw error;
  }
};
```

---

## 🛡️ Error Handling et Retry

### Pattern Circuit Breaker

```javascript
// Circuit Breaker pour appels API externes

class CircuitBreaker {
  constructor(functionName, threshold = 5, timeout = 60000) {
    this.functionName = functionName;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = "CLOSED"; // CLOSED, OPEN, HALF_OPEN
    this.failureCount = 0;
    this.lastFailureTime = null;
  }

  async execute(apiCall) {
    // État OPEN: Rejeter immédiatement
    if (this.state === "OPEN") {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        console.log(`Circuit breaker: Transitioning to HALF_OPEN`);
        this.state = "HALF_OPEN";
      } else {
        throw new Error(`Circuit breaker OPEN for ${this.functionName}`);
      }
    }

    try {
      const result = await apiCall();

      // Succès: Reset
      if (this.state === "HALF_OPEN") {
        console.log(`Circuit breaker: Transitioning to CLOSED`);
        this.state = "CLOSED";
        this.failureCount = 0;
      }

      return result;

    } catch (error) {
      this.failureCount++;
      this.lastFailureTime = Date.now();

      console.error(`Circuit breaker: Failure ${this.failureCount}/${this.threshold}`);

      // Trop d'échecs: Ouvrir circuit
      if (this.failureCount >= this.threshold) {
        console.error(`Circuit breaker: Opening circuit for ${this.functionName}`);
        this.state = "OPEN";
      }

      throw error;
    }
  }
}

// Usage dans fonction
const paymentApiBreaker = new CircuitBreaker("payment_api", 5, 60000);

exports = async function(orderId) {
  try {
    const result = await paymentApiBreaker.execute(async () => {
      // Appel API paiement
      return await context.http.post({
        url: "https://payment-gateway.com/charge",
        body: { orderId },
        headers: { "Authorization": "Bearer ..." }
      });
    });

    return { success: true, transactionId: result.body.id };

  } catch (error) {
    // Circuit breaker ouvert ou erreur API
    console.error("Payment failed:", error.message);
    return { success: false, error: error.message };
  }
};
```

### Exponential Backoff Retry

```javascript
// Helper: Retry avec exponential backoff

async function retryWithBackoff(
  operation,
  maxRetries = 3,
  initialDelay = 1000,
  maxDelay = 30000
) {
  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;

      if (attempt === maxRetries) {
        console.error(`Max retries (${maxRetries}) reached`);
        throw error;
      }

      // Calculer délai: 2^attempt * initialDelay
      const delay = Math.min(
        initialDelay * Math.pow(2, attempt),
        maxDelay
      );

      // Ajouter jitter (±20%)
      const jitter = delay * (0.8 + Math.random() * 0.4);

      console.log(`Retry ${attempt + 1}/${maxRetries} after ${Math.round(jitter)}ms`);
      await new Promise(resolve => setTimeout(resolve, jitter));
    }
  }

  throw lastError;
}

// Usage
exports = async function(orderId) {
  const result = await retryWithBackoff(
    async () => {
      // Opération potentiellement instable
      return await context.http.post({
        url: "https://unstable-api.com/process",
        body: { orderId }
      });
    },
    3,    // Max 3 retries
    1000, // Initial delay 1s
    10000 // Max delay 10s
  );

  return result;
};
```

---

## 📊 Monitoring et Observabilité

### Structured Logging

```javascript
// Pattern: Structured logging pour observabilité

class Logger {
  constructor(context, functionName) {
    this.context = context;
    this.functionName = functionName;
    this.correlationId = this.generateCorrelationId();
  }

  generateCorrelationId() {
    return `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
  }

  log(level, message, metadata = {}) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      level: level,
      function: this.functionName,
      correlationId: this.correlationId,
      message: message,
      userId: this.context.user?.id || null,
      environment: this.context.environment?.values?.environment || "unknown",
      ...metadata
    };

    // Log to console (captured by Atlas)
    console.log(JSON.stringify(logEntry));

    // Optionnel: Log to collection pour analytics
    if (level === "ERROR" || level === "WARN") {
      this.persistLog(logEntry);
    }
  }

  async persistLog(logEntry) {
    try {
      const logs = this.context.services.get("mongodb-atlas")
        .db("logs")
        .collection("function_logs");

      await logs.insertOne(logEntry);
    } catch (error) {
      // Fail silently - ne pas bloquer fonction
      console.error("Failed to persist log:", error);
    }
  }

  info(message, metadata) {
    this.log("INFO", message, metadata);
  }

  warn(message, metadata) {
    this.log("WARN", message, metadata);
  }

  error(message, error, metadata) {
    this.log("ERROR", message, {
      ...metadata,
      error: {
        message: error.message,
        stack: error.stack,
        name: error.name
      }
    });
  }
}

// Usage dans fonction
exports = async function(orderId) {
  const logger = new Logger(context, "processOrder");

  logger.info("Processing order started", { orderId });

  try {
    const order = await fetchOrder(orderId);
    logger.info("Order fetched", { orderId, status: order.status });

    await validateOrder(order);
    logger.info("Order validated", { orderId });

    const result = await processPayment(order);
    logger.info("Payment processed", {
      orderId,
      transactionId: result.transactionId,
      amount: order.total
    });

    return { success: true };

  } catch (error) {
    logger.error("Order processing failed", error, { orderId });
    throw error;
  }
};
```

### Métriques Custom

```javascript
// Pattern: Collecter métriques custom pour monitoring

exports = async function() {
  const startTime = Date.now();

  const metrics = context.services.get("mongodb-atlas")
    .db("metrics")
    .collection("function_metrics");

  let processedCount = 0;
  let errorCount = 0;

  try {
    // Votre logique
    const jobs = await fetchPendingJobs();

    for (const job of jobs) {
      try {
        await processJob(job);
        processedCount++;
      } catch (error) {
        errorCount++;
      }
    }

  } finally {
    // Enregistrer métriques
    const duration = Date.now() - startTime;

    await metrics.insertOne({
      functionName: "batchProcessor",
      timestamp: new Date(),
      duration: duration,
      processedCount: processedCount,
      errorCount: errorCount,
      successRate: processedCount / (processedCount + errorCount) || 0,
      environment: context.environment.values.environment
    });

    console.log(JSON.stringify({
      event: "function_completed",
      duration,
      processedCount,
      errorCount
    }));
  }
};

// Query métriques pour dashboards
db.function_metrics.aggregate([
  {
    $match: {
      timestamp: { $gte: new Date(Date.now() - 86400000) } // Last 24h
    }
  },
  {
    $group: {
      _id: {
        function: "$functionName",
        hour: { $hour: "$timestamp" }
      },
      avgDuration: { $avg: "$duration" },
      totalProcessed: { $sum: "$processedCount" },
      totalErrors: { $sum: "$errorCount" },
      avgSuccessRate: { $avg: "$successRate" }
    }
  }
]);
```

---

## 🚀 CI/CD et Infrastructure-as-Code

### Déploiement avec Atlas CLI

```bash
#!/bin/bash
# deploy-functions.sh - Deploy functions via CI/CD

set -e

PROJECT_ID="your-project-id"
APP_ID="your-app-id"

echo "📦 Deploying Atlas Functions..."

# 1. Pull current configuration
atlas app describe $APP_ID \
  --project $PROJECT_ID \
  --output json > app-config.json

# 2. Update functions directory
cp -r ./functions/* ./atlas-app/functions/

# 3. Deploy (push changes)
atlas app deploy $APP_ID \
  --project $PROJECT_ID \
  --include-node-modules \
  --include-package-json

echo "✅ Deployment complete"

# 4. Smoke test
echo "🧪 Running smoke tests..."
atlas app logs tail $APP_ID \
  --project $PROJECT_ID \
  --type function \
  --limit 10

echo "✅ All done!"
```

### Configuration as Code

```json
// app-config.json - Atlas App Services configuration

{
  "name": "production-app",
  "config_version": 20210101,

  "functions": [
    {
      "name": "processOrder",
      "source": "functions/processOrder.js",
      "can_evaluate": {},
      "run_as_system": true
    }
  ],

  "triggers": [
    {
      "name": "onOrderInsert",
      "type": "DATABASE",
      "config": {
        "operation_types": ["INSERT"],
        "database": "mydb",
        "collection": "orders",
        "full_document": true,
        "full_document_before_change": false,
        "unordered": false
      },
      "function_name": "processOrder",
      "disabled": false
    },
    {
      "name": "dailyCleanup",
      "type": "SCHEDULED",
      "config": {
        "schedule": "0 2 * * *"
      },
      "function_name": "cleanupOldData",
      "disabled": false
    }
  ],

  "values": [
    {
      "name": "apiKey",
      "from_secret": true,
      "value": "secret-api-key"
    },
    {
      "name": "environment",
      "value": "production"
    }
  ]
}
```

---

## 📋 Best Practices

### Checklist Production

```
┌───────────────────────────────────────────────────────────────────────┐
│           TRIGGERS & FUNCTIONS PRODUCTION CHECKLIST                   │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  DESIGN                                                               │
│  ☐ Fonctions idempotentes (safe to retry)                             │
│  ☐ Single Responsibility Principle                                    │
│  ☐ Timeouts appropriés (< 90s execution)                              │
│  ☐ Découplage via event-driven patterns                               │
│  ☐ Éviter database triggers en cascade (boucles infinies)             │
│                                                                       │
│  ERROR HANDLING                                                       │
│  ☐ Try/catch sur toutes opérations externes                           │
│  ☐ Retry logic avec exponential backoff                               │
│  ☐ Circuit breaker pour APIs externes                                 │
│  ☐ Dead letter queue pour échecs critiques                            │
│  ☐ Graceful degradation (fallbacks)                                   │
│                                                                       │
│  PERFORMANCE                                                          │
│  ☐ Limiter itérations (batch size, time limits)                       │
│  ☐ Utiliser indexes pour queries                                      │
│  ☐ Checkpointing pour scheduled triggers                              │
│  ☐ Parallélisation si possible (Promise.all)                          │
│  ☐ Éviter N+1 queries                                                 │
│                                                                       │
│  OBSERVABILITY                                                        │
│  ☐ Structured logging (JSON format)                                   │
│  ☐ Correlation IDs pour traçabilité                                   │
│  ☐ Métriques custom (durée, counts, errors)                           │
│  ☐ Alerting sur erreurs critiques                                     │
│  ☐ Logs accessibles via Atlas UI                                      │
│                                                                       │
│  SECURITY                                                             │
│  ☐ Secrets dans Values (jamais hardcodés)                             │
│  ☐ Validation inputs                                                  │
│  ☐ Rate limiting sur fonctions exposées                               │
│  ☐ Permissions minimales (principle of least privilege)               │
│  ☐ Audit trail pour opérations sensibles                              │
│                                                                       │
│  CI/CD                                                                │
│  ☐ Version control (Git)                                              │
│  ☐ Automated testing (unit, integration)                              │
│  ☐ Deploy via Atlas CLI                                               │
│  ☐ Staging environment pour tests                                     │
│  ☐ Rollback procedure documentée                                      │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🏁 Résumé

### Points Clés

1. **Event-Driven Architecture**
   - Database triggers (onChange)
   - Scheduled triggers (cron)
   - Découplage et scalabilité
   - Résilience built-in

2. **Patterns Avancés**
   - Audit trail complet
   - Data enrichment
   - Workflow orchestration
   - Materialized views

3. **Error Handling**
   - Retry avec exponential backoff
   - Circuit breaker
   - Dead letter queue
   - Graceful degradation

4. **Observabilité**
   - Structured logging
   - Correlation IDs
   - Custom metrics
   - Monitoring dashboards

5. **DevOps**
   - Infrastructure-as-Code
   - CI/CD avec Atlas CLI
   - Automated testing
   - Rollback procedures

### Configuration Minimale Production

```javascript
// Function template avec best practices
exports = async function(changeEvent) {
  const logger = new Logger(context, "myFunction");
  const startTime = Date.now();

  try {
    logger.info("Function started", {
      operationType: changeEvent.operationType,
      documentId: changeEvent.documentKey._id
    });

    // Votre logique avec retry
    const result = await retryWithBackoff(
      () => processEvent(changeEvent),
      3,
      1000
    );

    logger.info("Function completed", {
      duration: Date.now() - startTime,
      result
    });

    return result;

  } catch (error) {
    logger.error("Function failed", error, {
      duration: Date.now() - startTime
    });

    // Dead letter queue
    await sendToDeadLetterQueue(changeEvent, error);

    throw error;
  }
};
```

### Ressources

- [App Services Triggers](https://www.mongodb.com/docs/atlas/app-services/triggers/)
- [Functions Reference](https://www.mongodb.com/docs/atlas/app-services/functions/)
- [Atlas CLI](https://www.mongodb.com/docs/atlas/cli/stable/)

---


⏭️ [Data API](/14-mongodb-atlas/15-data-api.md)
