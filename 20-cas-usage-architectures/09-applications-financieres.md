🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.9 Applications Financières

## Introduction

Les applications financières imposent des exigences strictes en termes de :

- **Cohérence des données** : ACID transactions pour intégrité financière
- **Audit complet** : Traçabilité totale pour conformité réglementaire
- **Précision** : Calculs exacts sans erreurs d'arrondi
- **Sécurité** : Protection contre fraude et accès non autorisés
- **Disponibilité** : 99.99% uptime pour transactions critiques
- **Performance** : Latence minimale pour trading et paiements
- **Scalabilité** : Millions de transactions par jour
- **Conformité** : RGPD, PCI-DSS, SOX, Basel III, MiFID II
- **Réconciliation** : Balance checking et accounting

MongoDB répond à ces exigences grâce à :
- **Transactions multi-documents** : ACID garanties
- **Change Streams** : Event sourcing et audit trails
- **Replica Sets** : High availability avec failover automatique
- **Encryption** : At-rest et in-transit
- **Decimal128** : Précision exacte pour montants financiers
- **Agrégations** : Analytics et reporting complexes
- **Time Series** : Prix, métriques, historiques

## Architecture de référence

### Stack financière

```
┌──────────────────────────────────────────────────┐
│              Client Applications                 │
│  Web Banking • Mobile • Trading Platform • ATM   │
└────────────────────┬─────────────────────────────┘
                     │
                     │ HTTPS / TLS 1.3
                     │
        ┌────────────▼────────────┐
        │    API Gateway          │
        │  • Auth (OAuth2/JWT)    │
        │  • Rate Limiting        │
        │  • Request Validation   │
        └────────────┬────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼─────┐   ┌─────▼────┐   ┌──────▼───────┐
│ Account  │   │ Payment  │   │   Trading    │
│ Service  │   │ Service  │   │   Service    │
└────┬─────┘   └────┬─────┘   └──────┬───────┘
     │              │                │
     └──────────────┴────────────────┘
                     │
     ┌───────────────▼────────────┐
     │   MongoDB Replica Set      │
     │  ┌──────────────────┐      │
     │  │  Primary         │      │
     │  │  Write Concern   │      │
     │  │  w:majority      │      │
     │  │  j:true          │      │
     │  └──────────────────┘      │
     │  ┌──────────────────┐      │
     │  │  Secondary       │      │
     │  │  (Replica)       │      │
     │  └──────────────────┘      │
     │  ┌──────────────────┐      │
     │  │  Secondary       │      │
     │  │  (Replica)       │      │
     │  └──────────────────┘      │
     │                            │
     │  Collections:              │
     │  • accounts                │
     │  • transactions            │
     │  • audit_logs              │
     │  • balances                │
     │  • market_data (TS)        │
     └────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌─────▼────┐   ┌──────▼────┐
│ Fraud   │    │Reporting │   │Compliance │
│Detection│    │Analytics │   │  Engine   │
└─────────┘    └──────────┘   └───────────┘
     │               │               │
     └───────────────┴───────────────┘
                     │
        ┌────────────▼────────────┐
        │   External Services     │
        │  • KYC Verification     │
        │  • Payment Networks     │
        │  • Market Data Feeds    │
        │  • Regulatory Reporting │
        └─────────────────────────┘
```

### Composants architecturaux

#### 1. API Gateway
**Technologies :** Kong, AWS API Gateway, Apigee

**Justification :**
- Authentication et authorization centralisée
- Rate limiting par utilisateur/IP
- Request/response validation
- Logging et monitoring
- Protection DDoS

#### 2. Services métier
**Pattern :** Microservices avec séparation responsabilités

**Justification :**
- Isolation des domaines financiers
- Scalabilité indépendante
- Déploiements séparés
- Résilience et fault isolation

#### 3. MongoDB Replica Set
**Configuration :** 3+ nodes avec Write Concern majority

**Justification :**
- High availability (99.99%+)
- Automatic failover
- Read scaling via secondaries
- Data durability garantie

#### 4. Encryption
**Niveaux :**
- **At-rest** : Encrypted storage engine
- **In-transit** : TLS 1.3
- **Field-level** : Client-side field encryption

**Justification :**
- Conformité PCI-DSS
- Protection données sensibles
- Defense in depth

## Modélisation des données

### 1. Comptes bancaires

```javascript
// Collection: accounts
{
  _id: ObjectId("..."),
  accountId: "ACC_abc123def456",

  // Type de compte
  accountType: "checking",  // checking, savings, investment, credit_card

  // Titulaire
  holder: {
    customerId: "CUST_xyz789",
    name: "John Doe",
    email: "john.doe@example.com",

    // KYC (Know Your Customer)
    kyc: {
      status: "verified",  // pending, verified, rejected
      verifiedAt: ISODate("2024-01-15T10:00:00Z"),
      level: "full",  // basic, intermediate, full
      documents: [
        {
          type: "passport",
          number: "P123456789",
          expiryDate: ISODate("2030-01-15T00:00:00Z"),
          verifiedBy: "service_idverify"
        }
      ]
    },

    // AML (Anti-Money Laundering)
    aml: {
      riskLevel: "low",  // low, medium, high
      lastChecked: ISODate("2024-12-01T00:00:00Z"),
      watchlistStatus: "clear"  // clear, flagged, blocked
    }
  },

  // Balance (utiliser Decimal128 pour précision exacte)
  balance: {
    available: NumberDecimal("15432.50"),
    pending: NumberDecimal("250.00"),
    reserved: NumberDecimal("0.00"),

    // Balance = available + pending - reserved
    total: NumberDecimal("15682.50"),

    currency: "USD",

    // Pour multi-currency
    balances: [
      {
        currency: "USD",
        amount: NumberDecimal("15432.50")
      },
      {
        currency: "EUR",
        amount: NumberDecimal("3245.80")
      }
    ]
  },

  // Limites
  limits: {
    dailyWithdrawal: NumberDecimal("5000.00"),
    dailyTransfer: NumberDecimal("10000.00"),
    singleTransaction: NumberDecimal("2000.00"),

    // Utilisation actuelle (reset quotidien)
    usage: {
      withdrawal: NumberDecimal("0.00"),
      transfer: NumberDecimal("0.00"),
      resetAt: ISODate("2024-12-10T00:00:00Z")
    }
  },

  // Status
  status: "active",  // active, frozen, closed, suspended

  // Motif si frozen/closed
  statusReason: null,

  // Détails bancaires
  bankDetails: {
    iban: "FR7630006000011234567890189",
    bic: "BNPAFRPPXXX",
    accountNumber: "1234567890",
    sortCode: "12-34-56",
    routingNumber: "021000021"
  },

  // Intérêts (pour savings)
  interest: {
    rate: NumberDecimal("2.5"),  // %
    accrued: NumberDecimal("45.67"),
    lastCalculated: ISODate("2024-12-09T00:00:00Z"),
    nextPayment: ISODate("2025-01-01T00:00:00Z")
  },

  // Découvert autorisé
  overdraft: {
    enabled: true,
    limit: NumberDecimal("1000.00"),
    used: NumberDecimal("0.00"),
    interestRate: NumberDecimal("15.0")
  },

  // Carte associée
  cards: [
    {
      cardId: "CARD_abc123",
      type: "debit",  // debit, credit
      last4: "1234",
      expiryDate: "12/26",
      status: "active",  // active, blocked, expired, lost

      limits: {
        daily: NumberDecimal("3000.00"),
        monthly: NumberDecimal("20000.00")
      }
    }
  ],

  // Metadata
  createdAt: ISODate("2023-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z"),

  // Version pour optimistic locking
  version: 42,

  // Audit
  lastTransactionAt: ISODate("2024-12-09T14:30:00Z"),
  lastBalanceCheck: ISODate("2024-12-09T14:00:00Z")
}

// Index pour accounts
db.accounts.createIndex({ accountId: 1 }, { unique: true });
db.accounts.createIndex({ "holder.customerId": 1 });
db.accounts.createIndex({ "holder.email": 1 });
db.accounts.createIndex({ status: 1 });
db.accounts.createIndex({ accountType: 1, status: 1 });
db.accounts.createIndex({ "bankDetails.iban": 1 }, { unique: true, sparse: true });

// Index pour compliance
db.accounts.createIndex({ "holder.aml.riskLevel": 1, "holder.aml.lastChecked": 1 });
db.accounts.createIndex({ "holder.kyc.status": 1 });
```

### 2. Transactions financières

```javascript
// Collection: transactions
{
  _id: ObjectId("..."),
  transactionId: "TXN_abc123def456ghi789",

  // Type de transaction
  type: "transfer",  // transfer, withdrawal, deposit, payment, refund
  method: "wire_transfer",  // wire_transfer, card, ach, sepa, instant

  // Montant (TOUJOURS Decimal128)
  amount: NumberDecimal("1250.75"),
  currency: "USD",

  // Pour multi-currency
  originalAmount: NumberDecimal("1089.45"),
  originalCurrency: "EUR",
  exchangeRate: NumberDecimal("1.1481"),

  // Fees
  fees: {
    total: NumberDecimal("2.50"),
    breakdown: [
      {
        type: "transfer_fee",
        amount: NumberDecimal("1.50")
      },
      {
        type: "currency_conversion",
        amount: NumberDecimal("1.00")
      }
    ]
  },

  // Source (débiteur)
  source: {
    accountId: "ACC_abc123",
    customerId: "CUST_xyz789",

    // Balance avant transaction
    balanceBefore: NumberDecimal("16683.25"),
    balanceAfter: NumberDecimal("15430.00"),

    // Metadata
    name: "John Doe",
    iban: "FR7630006000011234567890189"
  },

  // Destination (créditeur)
  destination: {
    accountId: "ACC_def456",
    customerId: "CUST_abc123",

    balanceBefore: NumberDecimal("5234.20"),
    balanceAfter: NumberDecimal("6484.95"),

    name: "Jane Smith",
    iban: "FR7630004000051234567890143"
  },

  // Description
  description: "Rent payment - December 2024",
  reference: "RENT_DEC_2024",

  // Status et workflow
  status: "completed",  // pending, processing, completed, failed, reversed

  workflow: [
    {
      status: "initiated",
      timestamp: ISODate("2024-12-09T14:30:00.000Z"),
      by: "customer"
    },
    {
      status: "validated",
      timestamp: ISODate("2024-12-09T14:30:01.234Z"),
      by: "fraud_engine"
    },
    {
      status: "processing",
      timestamp: ISODate("2024-12-09T14:30:02.456Z"),
      by: "payment_service"
    },
    {
      status: "completed",
      timestamp: ISODate("2024-12-09T14:30:05.789Z"),
      by: "payment_service"
    }
  ],

  // Pour failed transactions
  failureReason: null,
  errorCode: null,

  // Fraud detection
  fraud: {
    checked: true,
    score: 0.15,  // 0-1, 0 = safe, 1 = fraudulent
    flags: [],  // ["unusual_amount", "velocity_check_failed"]
    verifiedBy: "fraud_engine_v2",
    verifiedAt: ISODate("2024-12-09T14:30:01.234Z")
  },

  // Metadata de la requête
  request: {
    ip: "192.168.1.100",
    userAgent: "Mozilla/5.0...",
    deviceId: "device_abc123",
    location: {
      country: "FR",
      city: "Paris",
      coordinates: {
        type: "Point",
        coordinates: [2.3522, 48.8566]
      }
    },

    // Pour détecter patterns suspects
    sessionId: "session_xyz789",
    timestamp: ISODate("2024-12-09T14:30:00.000Z")
  },

  // Idempotency (crucial pour éviter duplicates)
  idempotencyKey: "idem_abc123def456",

  // Double-entry bookkeeping
  entries: [
    {
      accountId: "ACC_abc123",
      type: "debit",
      amount: NumberDecimal("1250.75"),
      currency: "USD"
    },
    {
      accountId: "ACC_def456",
      type: "credit",
      amount: NumberDecimal("1250.75"),
      currency: "USD"
    }
  ],

  // Réconciliation
  reconciliation: {
    status: "reconciled",  // pending, reconciled, discrepancy
    reconciledAt: ISODate("2024-12-09T15:00:00Z"),
    batchId: "BATCH_20241209_001"
  },

  // Compliance et audit
  compliance: {
    // Pour reporting réglementaire
    reportable: false,
    category: "domestic_transfer",

    // AML checks
    amlChecked: true,
    amlStatus: "clear",

    // Pour transactions > seuil
    largeTransaction: false
  },

  // Timestamps
  createdAt: ISODate("2024-12-09T14:30:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:05.789Z"),
  completedAt: ISODate("2024-12-09T14:30:05.789Z"),

  // TTL pour archivage (optionnel)
  // Transactions gardées 7 ans pour conformité
  archiveAfter: ISODate("2031-12-09T00:00:00Z")
}

// Index critiques pour transactions
db.transactions.createIndex({ transactionId: 1 }, { unique: true });
db.transactions.createIndex({ "source.accountId": 1, createdAt: -1 });
db.transactions.createIndex({ "destination.accountId": 1, createdAt: -1 });
db.transactions.createIndex({ status: 1, createdAt: -1 });
db.transactions.createIndex({ idempotencyKey: 1 }, { unique: true, sparse: true });

// Pour fraud detection
db.transactions.createIndex({
  "source.customerId": 1,
  createdAt: -1,
  "fraud.score": 1
});

// Pour compliance reporting
db.transactions.createIndex({
  "compliance.reportable": 1,
  createdAt: -1
});

// Pour réconciliation
db.transactions.createIndex({
  "reconciliation.status": 1,
  createdAt: -1
});
```

### 3. Audit logs (Event Sourcing)

```javascript
// Collection: audit_logs
{
  _id: ObjectId("..."),
  eventId: "EVT_abc123def456",

  // Type d'événement
  eventType: "account.balance.updated",

  // Aggregate
  aggregateType: "account",
  aggregateId: "ACC_abc123",

  // Version pour event ordering
  version: 42,

  // Données de l'événement
  data: {
    previousBalance: NumberDecimal("16683.25"),
    newBalance: NumberDecimal("15430.00"),
    change: NumberDecimal("-1253.25"),
    reason: "transaction",
    transactionId: "TXN_abc123def456ghi789"
  },

  // Metadata
  metadata: {
    // Qui a déclenché
    initiatedBy: {
      type: "customer",  // customer, admin, system, service
      id: "CUST_xyz789",
      name: "John Doe"
    },

    // Comment
    via: "web_app",  // web_app, mobile_app, api, admin_panel, system

    // Où
    ip: "192.168.1.100",
    location: "Paris, FR",

    // Contexte
    sessionId: "session_xyz789",
    requestId: "req_abc123",

    // Pour compliance
    purpose: "payment_execution"
  },

  // Timestamp précis
  timestamp: ISODate("2024-12-09T14:30:05.789Z"),

  // Pour correlation avec autres events
  correlationId: "corr_abc123",
  causationId: "EVT_previous_event",

  // Status
  processed: true,

  // Compliance
  retention: {
    required: true,
    retainUntil: ISODate("2031-12-09T00:00:00Z")  // 7 ans
  }
}

// Index pour audit_logs
db.audit_logs.createIndex({ eventId: 1 }, { unique: true });
db.audit_logs.createIndex({
  aggregateType: 1,
  aggregateId: 1,
  version: 1
}, { unique: true });
db.audit_logs.createIndex({ timestamp: -1 });
db.audit_logs.createIndex({ eventType: 1, timestamp: -1 });
db.audit_logs.createIndex({ correlationId: 1, timestamp: 1 });

// Pour compliance queries
db.audit_logs.createIndex({
  "metadata.initiatedBy.id": 1,
  timestamp: -1
});
```

### 4. Balances snapshot (pour performance)

```javascript
// Collection: balances
// Snapshot quotidien des balances pour reconciliation rapide
{
  _id: ObjectId("..."),
  accountId: "ACC_abc123",
  date: ISODate("2024-12-09T00:00:00Z"),

  // Balances à cette date
  openingBalance: NumberDecimal("16683.25"),
  closingBalance: NumberDecimal("15430.00"),

  // Mouvements du jour
  debits: {
    count: 3,
    total: NumberDecimal("1453.25")
  },
  credits: {
    count: 2,
    total: NumberDecimal("200.00")
  },

  // Réconciliation
  reconciled: true,
  reconciledAt: ISODate("2024-12-10T01:00:00Z"),

  // Discrepancies
  discrepancy: NumberDecimal("0.00"),

  // Metadata
  currency: "USD",
  timezone: "Europe/Paris"
}

// Index pour balances
db.balances.createIndex({ accountId: 1, date: -1 }, { unique: true });
db.balances.createIndex({ date: -1 });
db.balances.createIndex({ reconciled: 1, date: -1 });

// TTL pour archivage (garder 2 ans en ligne)
db.balances.createIndex(
  { date: 1 },
  { expireAfterSeconds: 63072000 }  // 2 ans
);
```

### 5. Market data (Time Series pour trading)

```javascript
// Collection: market_data (Time Series)
db.createCollection("market_data", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  }
});

// Structure
{
  timestamp: ISODate("2024-12-09T14:30:15.234Z"),

  metadata: {
    symbol: "AAPL",
    exchange: "NASDAQ",
    type: "stock"  // stock, forex, crypto, commodity
  },

  // Prix
  price: {
    open: NumberDecimal("178.50"),
    high: NumberDecimal("179.25"),
    low: NumberDecimal("178.30"),
    close: NumberDecimal("179.00"),

    // Volume
    volume: 15423567,

    // Bid/Ask
    bid: NumberDecimal("178.98"),
    ask: NumberDecimal("179.02"),
    spread: NumberDecimal("0.04")
  },

  // Indicateurs (pré-calculés)
  indicators: {
    sma_50: NumberDecimal("175.23"),
    sma_200: NumberDecimal("168.45"),
    rsi: NumberDecimal("65.4"),
    macd: NumberDecimal("2.34")
  }
}

// Index pour market_data
db.market_data.createIndex({
  "metadata.symbol": 1,
  timestamp: -1
});

db.market_data.createIndex({
  "metadata.exchange": 1,
  "metadata.type": 1,
  timestamp: -1
});
```

## Services financiers

### 1. Account Service

```javascript
class AccountService {
  constructor(db) {
    this.db = db;
  }

  async createAccount(accountData) {
    const session = this.db.client.startSession();

    try {
      let account;

      await session.withTransaction(async () => {
        // 1. Vérifier KYC du customer
        const customer = await this.db.collection('customers')
          .findOne({ customerId: accountData.customerId }, { session });

        if (!customer || customer.kyc.status !== 'verified') {
          throw new Error('Customer KYC not verified');
        }

        // 2. Créer compte
        account = {
          accountId: this.generateAccountId(),
          accountType: accountData.accountType,

          holder: {
            customerId: customer.customerId,
            name: customer.name,
            email: customer.email,
            kyc: customer.kyc,
            aml: customer.aml
          },

          balance: {
            available: NumberDecimal("0.00"),
            pending: NumberDecimal("0.00"),
            reserved: NumberDecimal("0.00"),
            total: NumberDecimal("0.00"),
            currency: accountData.currency || "USD"
          },

          limits: this.getDefaultLimits(accountData.accountType),

          status: "active",

          bankDetails: {
            iban: this.generateIBAN(accountData.country),
            accountNumber: this.generateAccountNumber()
          },

          createdAt: new Date(),
          updatedAt: new Date(),
          version: 1
        };

        const result = await this.db.collection('accounts')
          .insertOne(account, { session });

        account._id = result.insertedId;

        // 3. Créer audit log
        await this.createAuditLog({
          eventType: 'account.created',
          aggregateType: 'account',
          aggregateId: account.accountId,
          version: 1,
          data: {
            accountType: account.accountType,
            currency: account.balance.currency
          },
          metadata: {
            initiatedBy: {
              type: 'customer',
              id: customer.customerId
            }
          }
        }, session);

        // 4. Créer balance snapshot initial
        await this.db.collection('balances').insertOne({
          accountId: account.accountId,
          date: this.getStartOfDay(new Date()),
          openingBalance: NumberDecimal("0.00"),
          closingBalance: NumberDecimal("0.00"),
          debits: { count: 0, total: NumberDecimal("0.00") },
          credits: { count: 0, total: NumberDecimal("0.00") },
          reconciled: true,
          currency: account.balance.currency
        }, { session });

      }, {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority', j: true },
        readPreference: 'primary'
      });

      return account;

    } finally {
      await session.endSession();
    }
  }

  async getBalance(accountId) {
    const account = await this.db.collection('accounts')
      .findOne(
        { accountId },
        {
          projection: {
            balance: 1,
            status: 1
          }
        }
      );

    if (!account) {
      throw new Error('Account not found');
    }

    if (account.status !== 'active') {
      throw new Error(`Account is ${account.status}`);
    }

    return account.balance;
  }

  async freezeAccount(accountId, reason, initiatedBy) {
    const session = this.db.client.startSession();

    try {
      await session.withTransaction(async () => {
        // Update account
        const result = await this.db.collection('accounts')
          .findOneAndUpdate(
            {
              accountId,
              status: 'active'
            },
            {
              $set: {
                status: 'frozen',
                statusReason: reason,
                updatedAt: new Date()
              },
              $inc: { version: 1 }
            },
            {
              session,
              returnDocument: 'after'
            }
          );

        if (!result) {
          throw new Error('Account not found or already frozen');
        }

        // Audit log
        await this.createAuditLog({
          eventType: 'account.frozen',
          aggregateType: 'account',
          aggregateId: accountId,
          version: result.version,
          data: {
            reason,
            previousStatus: 'active'
          },
          metadata: {
            initiatedBy
          }
        }, session);

      }, {
        writeConcern: { w: 'majority', j: true }
      });

    } finally {
      await session.endSession();
    }
  }

  async checkLimits(accountId, transactionType, amount) {
    const account = await this.db.collection('accounts')
      .findOne({ accountId });

    if (!account) {
      throw new Error('Account not found');
    }

    const limits = account.limits;
    const usage = limits.usage;

    // Vérifier si usage doit être reset
    if (new Date() >= new Date(usage.resetAt)) {
      // Reset needed
      await this.resetDailyLimits(accountId);
      usage.withdrawal = NumberDecimal("0.00");
      usage.transfer = NumberDecimal("0.00");
    }

    // Vérifier limite de transaction unique
    if (amount > limits.singleTransaction) {
      return {
        allowed: false,
        reason: 'EXCEEDS_SINGLE_TRANSACTION_LIMIT',
        limit: limits.singleTransaction
      };
    }

    // Vérifier limites quotidiennes
    let dailyLimit, currentUsage;

    if (transactionType === 'withdrawal') {
      dailyLimit = limits.dailyWithdrawal;
      currentUsage = usage.withdrawal;
    } else if (transactionType === 'transfer') {
      dailyLimit = limits.dailyTransfer;
      currentUsage = usage.transfer;
    }

    if (currentUsage + amount > dailyLimit) {
      return {
        allowed: false,
        reason: 'EXCEEDS_DAILY_LIMIT',
        limit: dailyLimit,
        used: currentUsage,
        remaining: dailyLimit - currentUsage
      };
    }

    return {
      allowed: true,
      remaining: dailyLimit - currentUsage - amount
    };
  }

  generateAccountId() {
    return `ACC_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  getDefaultLimits(accountType) {
    const limits = {
      checking: {
        dailyWithdrawal: NumberDecimal("5000.00"),
        dailyTransfer: NumberDecimal("10000.00"),
        singleTransaction: NumberDecimal("2000.00")
      },
      savings: {
        dailyWithdrawal: NumberDecimal("2000.00"),
        dailyTransfer: NumberDecimal("5000.00"),
        singleTransaction: NumberDecimal("1000.00")
      }
    };

    const limit = limits[accountType] || limits.checking;

    return {
      ...limit,
      usage: {
        withdrawal: NumberDecimal("0.00"),
        transfer: NumberDecimal("0.00"),
        resetAt: this.getNextMidnight()
      }
    };
  }

  getNextMidnight() {
    const tomorrow = new Date();
    tomorrow.setDate(tomorrow.getDate() + 1);
    tomorrow.setHours(0, 0, 0, 0);
    return tomorrow;
  }

  getStartOfDay(date) {
    const start = new Date(date);
    start.setHours(0, 0, 0, 0);
    return start;
  }

  async createAuditLog(eventData, session) {
    await this.db.collection('audit_logs').insertOne({
      eventId: `EVT_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      ...eventData,
      timestamp: new Date(),
      processed: true
    }, { session });
  }
}
```

### 2. Payment Service (avec transactions ACID)

```javascript
class PaymentService {
  constructor(db) {
    this.db = db;
  }

  async transfer(transferRequest) {
    const {
      sourceAccountId,
      destinationAccountId,
      amount,
      currency,
      description,
      reference,
      idempotencyKey
    } = transferRequest;

    // Vérifier idempotency
    const existing = await this.checkIdempotency(idempotencyKey);
    if (existing) {
      return existing;
    }

    const session = this.db.client.startSession();

    try {
      let transaction;

      await session.withTransaction(async () => {
        // 1. Lock et récupérer comptes
        const [sourceAccount, destAccount] = await Promise.all([
          this.db.collection('accounts').findOne(
            { accountId: sourceAccountId },
            { session }
          ),
          this.db.collection('accounts').findOne(
            { accountId: destinationAccountId },
            { session }
          )
        ]);

        // 2. Validations
        this.validateTransfer(sourceAccount, destAccount, amount, currency);

        // 3. Vérifier balance suffisante
        const sourceBalance = sourceAccount.balance.available;
        if (sourceBalance < amount) {
          throw new Error('INSUFFICIENT_FUNDS');
        }

        // 4. Vérifier limites
        const limitsCheck = await this.checkTransactionLimits(
          sourceAccountId,
          'transfer',
          amount,
          session
        );

        if (!limitsCheck.allowed) {
          throw new Error(`LIMIT_EXCEEDED: ${limitsCheck.reason}`);
        }

        // 5. Fraud check
        const fraudCheck = await this.checkFraud({
          sourceAccountId,
          destinationAccountId,
          amount,
          currency
        });

        if (fraudCheck.score > 0.8) {
          throw new Error('FRAUD_DETECTED');
        }

        // 6. Créer transaction
        transaction = {
          transactionId: this.generateTransactionId(),
          type: 'transfer',
          method: 'internal',

          amount: NumberDecimal(amount.toString()),
          currency,

          fees: {
            total: NumberDecimal("0.00"),
            breakdown: []
          },

          source: {
            accountId: sourceAccountId,
            customerId: sourceAccount.holder.customerId,
            balanceBefore: sourceAccount.balance.available,
            balanceAfter: NumberDecimal(
              (sourceAccount.balance.available - amount).toString()
            ),
            name: sourceAccount.holder.name,
            iban: sourceAccount.bankDetails.iban
          },

          destination: {
            accountId: destinationAccountId,
            customerId: destAccount.holder.customerId,
            balanceBefore: destAccount.balance.available,
            balanceAfter: NumberDecimal(
              (destAccount.balance.available + amount).toString()
            ),
            name: destAccount.holder.name,
            iban: destAccount.bankDetails.iban
          },

          description,
          reference,

          status: 'processing',

          workflow: [
            {
              status: 'initiated',
              timestamp: new Date(),
              by: 'customer'
            },
            {
              status: 'validated',
              timestamp: new Date(),
              by: 'payment_service'
            }
          ],

          fraud: fraudCheck,

          idempotencyKey,

          entries: [
            {
              accountId: sourceAccountId,
              type: 'debit',
              amount: NumberDecimal(amount.toString()),
              currency
            },
            {
              accountId: destinationAccountId,
              type: 'credit',
              amount: NumberDecimal(amount.toString()),
              currency
            }
          ],

          reconciliation: {
            status: 'pending'
          },

          compliance: {
            reportable: amount >= 10000,
            category: 'domestic_transfer',
            amlChecked: true,
            amlStatus: 'clear',
            largeTransaction: amount >= 10000
          },

          createdAt: new Date(),
          updatedAt: new Date()
        };

        // 7. Insérer transaction
        const txResult = await this.db.collection('transactions')
          .insertOne(transaction, { session });

        transaction._id = txResult.insertedId;

        // 8. Mettre à jour balances (double-entry bookkeeping)
        // Débiter source
        const sourceUpdate = await this.db.collection('accounts')
          .findOneAndUpdate(
            {
              accountId: sourceAccountId,
              version: sourceAccount.version
            },
            {
              $inc: {
                'balance.available': -amount,
                'balance.total': -amount,
                version: 1,
                'limits.usage.transfer': amount
              },
              $set: {
                updatedAt: new Date(),
                lastTransactionAt: new Date()
              }
            },
            {
              session,
              returnDocument: 'after'
            }
          );

        if (!sourceUpdate) {
          throw new Error('CONCURRENT_MODIFICATION');
        }

        // Créditer destination
        const destUpdate = await this.db.collection('accounts')
          .findOneAndUpdate(
            {
              accountId: destinationAccountId,
              version: destAccount.version
            },
            {
              $inc: {
                'balance.available': amount,
                'balance.total': amount,
                version: 1
              },
              $set: {
                updatedAt: new Date(),
                lastTransactionAt: new Date()
              }
            },
            {
              session,
              returnDocument: 'after'
            }
          );

        if (!destUpdate) {
          throw new Error('CONCURRENT_MODIFICATION');
        }

        // 9. Mettre à jour transaction status
        transaction.status = 'completed';
        transaction.workflow.push({
          status: 'completed',
          timestamp: new Date(),
          by: 'payment_service'
        });
        transaction.completedAt = new Date();

        await this.db.collection('transactions').updateOne(
          { _id: transaction._id },
          {
            $set: {
              status: 'completed',
              workflow: transaction.workflow,
              completedAt: new Date(),
              updatedAt: new Date()
            }
          },
          { session }
        );

        // 10. Créer audit logs
        await Promise.all([
          this.createAuditLog({
            eventType: 'account.balance.debited',
            aggregateType: 'account',
            aggregateId: sourceAccountId,
            version: sourceUpdate.version,
            data: {
              amount: -amount,
              transactionId: transaction.transactionId,
              balanceBefore: sourceAccount.balance.available,
              balanceAfter: sourceUpdate.balance.available
            }
          }, session),

          this.createAuditLog({
            eventType: 'account.balance.credited',
            aggregateType: 'account',
            aggregateId: destinationAccountId,
            version: destUpdate.version,
            data: {
              amount: amount,
              transactionId: transaction.transactionId,
              balanceBefore: destAccount.balance.available,
              balanceAfter: destUpdate.balance.available
            }
          }, session),

          this.createAuditLog({
            eventType: 'transaction.completed',
            aggregateType: 'transaction',
            aggregateId: transaction.transactionId,
            version: 1,
            data: {
              type: 'transfer',
              amount,
              currency,
              sourceAccountId,
              destinationAccountId
            }
          }, session)
        ]);

      }, {
        readConcern: { level: 'snapshot' },
        writeConcern: { w: 'majority', j: true },
        readPreference: 'primary',
        maxCommitTimeMS: 10000
      });

      return {
        success: true,
        transaction
      };

    } catch (error) {
      // Log erreur
      console.error('Transfer failed:', error);

      // Créer transaction failed si elle existe
      if (transaction?._id) {
        await this.db.collection('transactions').updateOne(
          { _id: transaction._id },
          {
            $set: {
              status: 'failed',
              failureReason: error.message,
              updatedAt: new Date()
            }
          }
        );
      }

      throw error;

    } finally {
      await session.endSession();
    }
  }

  validateTransfer(sourceAccount, destAccount, amount, currency) {
    // Validations
    if (!sourceAccount) {
      throw new Error('SOURCE_ACCOUNT_NOT_FOUND');
    }

    if (!destAccount) {
      throw new Error('DESTINATION_ACCOUNT_NOT_FOUND');
    }

    if (sourceAccount.status !== 'active') {
      throw new Error(`SOURCE_ACCOUNT_${sourceAccount.status.toUpperCase()}`);
    }

    if (destAccount.status !== 'active') {
      throw new Error(`DESTINATION_ACCOUNT_${destAccount.status.toUpperCase()}`);
    }

    if (sourceAccount.balance.currency !== currency) {
      throw new Error('CURRENCY_MISMATCH');
    }

    if (amount <= 0) {
      throw new Error('INVALID_AMOUNT');
    }

    if (sourceAccount.accountId === destAccount.accountId) {
      throw new Error('SAME_ACCOUNT_TRANSFER');
    }
  }

  async checkIdempotency(idempotencyKey) {
    if (!idempotencyKey) return null;

    const existing = await this.db.collection('transactions')
      .findOne({ idempotencyKey });

    return existing;
  }

  async checkFraud(transferData) {
    // Implémentation simplifiée
    // En production: appeler service ML de détection de fraude

    const { sourceAccountId, amount } = transferData;

    // Vérifier vélocité (nombre de transactions récentes)
    const recentCount = await this.db.collection('transactions')
      .countDocuments({
        'source.accountId': sourceAccountId,
        createdAt: {
          $gte: new Date(Date.now() - 3600000)  // Dernière heure
        }
      });

    let score = 0;

    // Trop de transactions
    if (recentCount > 10) {
      score += 0.3;
    }

    // Montant inhabituel
    const avgAmount = await this.getAverageTransactionAmount(sourceAccountId);
    if (amount > avgAmount * 3) {
      score += 0.2;
    }

    return {
      checked: true,
      score: Math.min(score, 1.0),
      flags: recentCount > 10 ? ['high_velocity'] : [],
      verifiedBy: 'fraud_engine_v2',
      verifiedAt: new Date()
    };
  }

  async getAverageTransactionAmount(accountId) {
    const result = await this.db.collection('transactions')
      .aggregate([
        {
          $match: {
            'source.accountId': accountId,
            status: 'completed',
            createdAt: {
              $gte: new Date(Date.now() - 30 * 24 * 3600000)  // 30 jours
            }
          }
        },
        {
          $group: {
            _id: null,
            avgAmount: { $avg: { $toDouble: '$amount' } }
          }
        }
      ])
      .toArray();

    return result[0]?.avgAmount || 0;
  }

  generateTransactionId() {
    return `TXN_${Date.now()}_${Math.random().toString(36).substr(2, 15)}`;
  }

  async createAuditLog(eventData, session) {
    await this.db.collection('audit_logs').insertOne({
      eventId: `EVT_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      ...eventData,
      timestamp: new Date(),
      processed: true
    }, { session });
  }
}
```

### 3. Reconciliation Service

```javascript
class ReconciliationService {
  constructor(db) {
    this.db = db;
  }

  async reconcileDaily(date) {
    console.log(`Starting daily reconciliation for ${date.toISOString()}`);

    const startOfDay = this.getStartOfDay(date);
    const endOfDay = this.getEndOfDay(date);

    // Récupérer tous les comptes actifs
    const accounts = await this.db.collection('accounts')
      .find({ status: { $in: ['active', 'frozen'] } })
      .project({ accountId: 1, balance: 1 })
      .toArray();

    const results = {
      total: accounts.length,
      reconciled: 0,
      discrepancies: []
    };

    for (const account of accounts) {
      const reconciliation = await this.reconcileAccount(
        account.accountId,
        startOfDay,
        endOfDay
      );

      if (reconciliation.reconciled) {
        results.reconciled++;
      }

      if (reconciliation.discrepancy !== 0) {
        results.discrepancies.push({
          accountId: account.accountId,
          discrepancy: reconciliation.discrepancy
        });
      }
    }

    console.log(`Reconciliation complete: ${results.reconciled}/${results.total} accounts`);

    if (results.discrepancies.length > 0) {
      console.warn(`Found ${results.discrepancies.length} discrepancies`);
      await this.alertDiscrepancies(results.discrepancies, date);
    }

    return results;
  }

  async reconcileAccount(accountId, startDate, endDate) {
    // 1. Récupérer balance de début
    const previousSnapshot = await this.db.collection('balances')
      .findOne(
        {
          accountId,
          date: { $lt: startDate }
        },
        { sort: { date: -1 } }
      );

    const openingBalance = previousSnapshot
      ? previousSnapshot.closingBalance
      : NumberDecimal("0.00");

    // 2. Calculer mouvements du jour
    const pipeline = [
      {
        $match: {
          $or: [
            { 'source.accountId': accountId },
            { 'destination.accountId': accountId }
          ],
          status: 'completed',
          completedAt: {
            $gte: startDate,
            $lt: endDate
          }
        }
      },
      {
        $group: {
          _id: null,

          debits: {
            $sum: {
              $cond: [
                { $eq: ['$source.accountId', accountId] },
                { $toDecimal: '$amount' },
                0
              ]
            }
          },

          debitCount: {
            $sum: {
              $cond: [
                { $eq: ['$source.accountId', accountId] },
                1,
                0
              ]
            }
          },

          credits: {
            $sum: {
              $cond: [
                { $eq: ['$destination.accountId', accountId] },
                { $toDecimal: '$amount' },
                0
              ]
            }
          },

          creditCount: {
            $sum: {
              $cond: [
                { $eq: ['$destination.accountId', accountId] },
                1,
                0
              ]
            }
          }
        }
      }
    ];

    const movements = await this.db.collection('transactions')
      .aggregate(pipeline)
      .toArray();

    const debits = movements[0]?.debits || NumberDecimal("0.00");
    const credits = movements[0]?.credits || NumberDecimal("0.00");
    const debitCount = movements[0]?.debitCount || 0;
    const creditCount = movements[0]?.creditCount || 0;

    // 3. Calculer closing balance attendu
    const expectedClosing = NumberDecimal(
      (openingBalance + credits - debits).toString()
    );

    // 4. Récupérer balance réelle
    const account = await this.db.collection('accounts')
      .findOne({ accountId });

    const actualBalance = account.balance.total;

    // 5. Vérifier discrepancy
    const discrepancy = NumberDecimal(
      (actualBalance - expectedClosing).toString()
    );

    const reconciled = Math.abs(discrepancy) < 0.01;  // Tolérance 1 centime

    // 6. Sauvegarder snapshot
    await this.db.collection('balances').updateOne(
      {
        accountId,
        date: startDate
      },
      {
        $set: {
          accountId,
          date: startDate,
          openingBalance,
          closingBalance: actualBalance,

          debits: {
            count: debitCount,
            total: debits
          },
          credits: {
            count: creditCount,
            total: credits
          },

          reconciled,
          reconciledAt: new Date(),
          discrepancy,

          currency: account.balance.currency
        }
      },
      { upsert: true }
    );

    return {
      accountId,
      reconciled,
      discrepancy,
      openingBalance,
      closingBalance: actualBalance,
      expectedClosing
    };
  }

  getStartOfDay(date) {
    const start = new Date(date);
    start.setHours(0, 0, 0, 0);
    return start;
  }

  getEndOfDay(date) {
    const end = new Date(date);
    end.setHours(23, 59, 59, 999);
    return end;
  }

  async alertDiscrepancies(discrepancies, date) {
    // Envoyer alerte aux ops
    console.error(`RECONCILIATION ALERT for ${date.toISOString()}:`);
    console.error(JSON.stringify(discrepancies, null, 2));

    // En production: envoyer email, Slack, PagerDuty, etc.
  }
}
```

### 4. Fraud Detection Service

```javascript
class FraudDetectionService {
  constructor(db) {
    this.db = db;
    this.rules = this.loadRules();
  }

  loadRules() {
    return [
      {
        id: 'velocity_check',
        name: 'High Transaction Velocity',
        check: this.checkVelocity.bind(this),
        weight: 0.3
      },
      {
        id: 'unusual_amount',
        name: 'Unusual Transaction Amount',
        check: this.checkUnusualAmount.bind(this),
        weight: 0.25
      },
      {
        id: 'location_change',
        name: 'Sudden Location Change',
        check: this.checkLocationChange.bind(this),
        weight: 0.2
      },
      {
        id: 'time_pattern',
        name: 'Unusual Time Pattern',
        check: this.checkTimePattern.bind(this),
        weight: 0.15
      },
      {
        id: 'blacklist',
        name: 'Blacklist Check',
        check: this.checkBlacklist.bind(this),
        weight: 0.5  // Si blacklist, score très élevé
      }
    ];
  }

  async analyzTransaction(transaction) {
    const scores = [];
    const flags = [];

    for (const rule of this.rules) {
      const result = await rule.check(transaction);

      if (result.triggered) {
        scores.push(result.score * rule.weight);
        flags.push(rule.id);
      }
    }

    const finalScore = Math.min(scores.reduce((a, b) => a + b, 0), 1.0);

    // Décision
    let decision = 'approve';

    if (finalScore > 0.8) {
      decision = 'block';
    } else if (finalScore > 0.5) {
      decision = 'review';
    }

    return {
      score: finalScore,
      decision,
      flags,
      details: {
        rules: this.rules.map(r => r.id),
        timestamp: new Date()
      }
    };
  }

  async checkVelocity(transaction) {
    // Compter transactions récentes
    const recentCount = await this.db.collection('transactions')
      .countDocuments({
        'source.accountId': transaction.source.accountId,
        createdAt: {
          $gte: new Date(Date.now() - 3600000)  // 1 heure
        },
        status: { $in: ['completed', 'processing'] }
      });

    // Seuil: > 10 transactions/heure
    if (recentCount > 10) {
      return {
        triggered: true,
        score: Math.min(recentCount / 20, 1.0)
      };
    }

    return { triggered: false };
  }

  async checkUnusualAmount(transaction) {
    // Récupérer moyenne et stddev
    const stats = await this.db.collection('transactions')
      .aggregate([
        {
          $match: {
            'source.accountId': transaction.source.accountId,
            status: 'completed',
            createdAt: {
              $gte: new Date(Date.now() - 30 * 24 * 3600000)  // 30 jours
            }
          }
        },
        {
          $group: {
            _id: null,
            avg: { $avg: { $toDouble: '$amount' } },
            stdDev: { $stdDevPop: { $toDouble: '$amount' } }
          }
        }
      ])
      .toArray();

    if (!stats[0]) return { triggered: false };

    const { avg, stdDev } = stats[0];
    const amount = parseFloat(transaction.amount);

    // Si montant > avg + 3*stdDev: inhabituel
    if (amount > avg + 3 * stdDev) {
      const zScore = (amount - avg) / stdDev;
      return {
        triggered: true,
        score: Math.min(zScore / 10, 1.0)
      };
    }

    return { triggered: false };
  }

  async checkLocationChange(transaction) {
    // Récupérer dernière transaction
    const lastTransaction = await this.db.collection('transactions')
      .findOne(
        {
          'source.accountId': transaction.source.accountId,
          _id: { $ne: transaction._id },
          'request.location': { $exists: true }
        },
        { sort: { createdAt: -1 } }
      );

    if (!lastTransaction) return { triggered: false };

    const currentLocation = transaction.request.location;
    const lastLocation = lastTransaction.request.location;

    // Calculer distance (simplifié)
    const distance = this.calculateDistance(
      currentLocation.coordinates,
      lastLocation.coordinates
    );

    const timeDiff = (new Date(transaction.createdAt) - new Date(lastTransaction.createdAt)) / 1000;  // secondes

    // Si distance > 500km en < 1h: suspect
    if (distance > 500 && timeDiff < 3600) {
      return {
        triggered: true,
        score: 0.8
      };
    }

    return { triggered: false };
  }

  async checkTimePattern(transaction) {
    const hour = new Date(transaction.createdAt).getHours();

    // Transactions entre 2h et 6h du matin: inhabituel
    if (hour >= 2 && hour < 6) {
      return {
        triggered: true,
        score: 0.4
      };
    }

    return { triggered: false };
  }

  async checkBlacklist(transaction) {
    // Vérifier si compte/IP/device blacklisté
    const blacklisted = await this.db.collection('blacklist')
      .findOne({
        $or: [
          { accountId: transaction.source.accountId },
          { ip: transaction.request.ip },
          { deviceId: transaction.request.deviceId }
        ]
      });

    if (blacklisted) {
      return {
        triggered: true,
        score: 1.0  // Block immédiatement
      };
    }

    return { triggered: false };
  }

  calculateDistance(coords1, coords2) {
    // Haversine formula (simplifié)
    const [lon1, lat1] = coords1;
    const [lon2, lat2] = coords2;

    const R = 6371;  // Rayon terre en km
    const dLat = (lat2 - lat1) * Math.PI / 180;
    const dLon = (lon2 - lon1) * Math.PI / 180;

    const a =
      Math.sin(dLat / 2) * Math.sin(dLat / 2) +
      Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) *
      Math.sin(dLon / 2) * Math.sin(dLon / 2);

    const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));

    return R * c;
  }
}
```

## Sécurité et Compliance

### 1. Encryption at rest

```javascript
// Configuration MongoDB avec encryption
const mongoConfig = {
  security: {
    enableEncryption: true,
    encryptionKeyFile: '/path/to/keyfile',

    // Field-level encryption
    fieldLevelEncryption: {
      keyVaultNamespace: 'encryption.__keyVault',
      kmsProviders: {
        aws: {
          accessKeyId: process.env.AWS_ACCESS_KEY_ID,
          secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
        }
      }
    }
  }
};

// Champs à encrypter
const encryptedFields = {
  fields: [
    {
      path: 'holder.email',
      bsonType: 'string',
      keyId: dataKey
    },
    {
      path: 'bankDetails.iban',
      bsonType: 'string',
      keyId: dataKey
    },
    {
      path: 'bankDetails.accountNumber',
      bsonType: 'string',
      keyId: dataKey
    },
    {
      path: 'cards.*.last4',
      bsonType: 'string',
      keyId: dataKey
    }
  ]
};
```

### 2. Access Control (RBAC)

```javascript
// Rôles MongoDB
db.createRole({
  role: 'accountManager',
  privileges: [
    {
      resource: { db: 'banking', collection: 'accounts' },
      actions: ['find', 'insert', 'update']
    },
    {
      resource: { db: 'banking', collection: 'transactions' },
      actions: ['find', 'insert']
    },
    {
      resource: { db: 'banking', collection: 'audit_logs' },
      actions: ['insert']
    }
  ],
  roles: []
});

db.createRole({
  role: 'auditor',
  privileges: [
    {
      resource: { db: 'banking', collection: '' },
      actions: ['find']  // Read-only
    }
  ],
  roles: []
});

db.createRole({
  role: 'complianceOfficer',
  privileges: [
    {
      resource: { db: 'banking', collection: 'transactions' },
      actions: ['find']
    },
    {
      resource: { db: 'banking', collection: 'audit_logs' },
      actions: ['find']
    },
    {
      resource: { db: 'banking', collection: 'accounts' },
      actions: ['find', 'update']  // Pour freeze accounts
    }
  ],
  roles: []
});
```

### 3. Audit trail et compliance

```javascript
class ComplianceService {
  constructor(db) {
    this.db = db;
  }

  async generateAMLReport(startDate, endDate) {
    // Rapport AML pour transactions > 10,000
    const largeTransactions = await this.db.collection('transactions')
      .find({
        'compliance.largeTransaction': true,
        createdAt: { $gte: startDate, $lte: endDate },
        status: 'completed'
      })
      .toArray();

    return {
      period: { start: startDate, end: endDate },
      transactionCount: largeTransactions.length,
      totalAmount: largeTransactions.reduce(
        (sum, tx) => sum + parseFloat(tx.amount),
        0
      ),
      transactions: largeTransactions.map(tx => ({
        transactionId: tx.transactionId,
        date: tx.createdAt,
        amount: tx.amount,
        currency: tx.currency,
        source: tx.source.customerId,
        destination: tx.destination.customerId,
        amlStatus: tx.compliance.amlStatus
      }))
    };
  }

  async getCustomerActivityReport(customerId, days = 90) {
    // Activité client pour compliance
    const cutoff = new Date(Date.now() - days * 24 * 3600000);

    const pipeline = [
      {
        $match: {
          $or: [
            { 'source.customerId': customerId },
            { 'destination.customerId': customerId }
          ],
          createdAt: { $gte: cutoff },
          status: 'completed'
        }
      },
      {
        $group: {
          _id: null,

          transactionCount: { $sum: 1 },

          totalSent: {
            $sum: {
              $cond: [
                { $eq: ['$source.customerId', customerId] },
                { $toDouble: '$amount' },
                0
              ]
            }
          },

          totalReceived: {
            $sum: {
              $cond: [
                { $eq: ['$destination.customerId', customerId] },
                { $toDouble: '$amount' },
                0
              ]
            }
          },

          largeTransactions: {
            $sum: {
              $cond: ['$compliance.largeTransaction', 1, 0]
            }
          }
        }
      }
    ];

    const result = await this.db.collection('transactions')
      .aggregate(pipeline)
      .toArray();

    return result[0] || {
      transactionCount: 0,
      totalSent: 0,
      totalReceived: 0,
      largeTransactions: 0
    };
  }
}
```

## Performance et optimisation

### 1. Configuration MongoDB pour finance

```javascript
const financialDbConfig = {
  // Replica Set avec 3+ nodes
  replication: {
    replSetName: 'banking-rs',
    oplogSizeMB: 50000  // 50GB pour long replay window
  },

  // Write Concern strict pour financial data
  writeConcern: {
    w: 'majority',
    j: true,  // Journal sync obligatoire
    wtimeout: 10000
  },

  // Read Concern pour consistency
  readConcern: {
    level: 'majority'  // Ou 'snapshot' pour transactions
  },

  // Storage engine
  storage: {
    wiredTiger: {
      engineConfig: {
        cacheSizeGB: 64,
        journalCompressor: 'snappy',
        directoryForIndexes: true
      },
      collectionConfig: {
        blockCompressor: 'snappy'
      }
    },

    // Encryption at rest
    encryption: {
      enableEncryption: true
    }
  },

  // Security
  security: {
    authorization: 'enabled',
    clusterAuthMode: 'x509'
  },

  // Network encryption
  net: {
    tls: {
      mode: 'requireTLS',
      certificateKeyFile: '/path/to/cert.pem',
      CAFile: '/path/to/ca.pem'
    }
  }
};
```

### 2. Métriques de performance

```javascript
const financialMetrics = {
  // Transactions
  'transactions.latency.p50': {
    description: 'Transaction processing latency p50',
    target: 100,  // ms
    alert_threshold: 500
  },

  'transactions.latency.p99': {
    description: 'Transaction processing latency p99',
    target: 500,  // ms
    alert_threshold: 2000
  },

  'transactions.throughput': {
    description: 'Transactions per second',
    target: 1000,
    alert_threshold: 100
  },

  'transactions.failure_rate': {
    description: 'Transaction failure rate',
    target: 0.001,  // 0.1%
    alert_threshold: 0.01  // 1%
  },

  // Reconciliation
  'reconciliation.discrepancy_rate': {
    description: 'Accounts with discrepancies',
    target: 0,
    alert_threshold: 0.001  // 0.1%
  },

  // Fraud
  'fraud.detection_latency': {
    description: 'Fraud check latency',
    target: 50,  // ms
    alert_threshold: 200
  },

  'fraud.false_positive_rate': {
    description: 'False positive rate',
    target: 0.05,  // 5%
    alert_threshold: 0.15  // 15%
  }
};
```

## Checklist de déploiement

### ✅ Architecture

- [ ] Replica Set 3+ nodes
- [ ] Write Concern w:majority, j:true
- [ ] Read Concern snapshot pour transactions
- [ ] Encryption at-rest activée
- [ ] TLS/SSL pour connexions
- [ ] Field-level encryption pour PII

### ✅ Modélisation

- [ ] Decimal128 pour tous les montants
- [ ] Versioning optimistic locking
- [ ] Idempotency keys
- [ ] Double-entry bookkeeping
- [ ] Audit logs pour tous les events
- [ ] Balance snapshots quotidiens

### ✅ Transactions ACID

- [ ] Multi-document transactions
- [ ] Proper error handling et rollback
- [ ] Timeout configuration
- [ ] Retry logic
- [ ] Idempotency garantie

### ✅ Sécurité

- [ ] RBAC configuré
- [ ] Authentication forte
- [ ] Encryption at-rest et in-transit
- [ ] Field-level encryption pour données sensibles
- [ ] Audit logging activé
- [ ] Network isolation

### ✅ Compliance

- [ ] KYC/AML checks
- [ ] Transaction limits
- [ ] Large transaction reporting
- [ ] Audit trails complets
- [ ] Data retention policies (7 ans)
- [ ] RGPD compliance
- [ ] PCI-DSS si cartes

### ✅ Fraud Detection

- [ ] Velocity checks
- [ ] Amount anomaly detection
- [ ] Location verification
- [ ] Blacklist checking
- [ ] Real-time scoring
- [ ] Manual review workflow

### ✅ Operations

- [ ] Automated backups (point-in-time)
- [ ] Disaster recovery plan
- [ ] Daily reconciliation jobs
- [ ] Monitoring et alerting
- [ ] Performance metrics
- [ ] Incident response procedures

### ✅ Testing

- [ ] Transaction testing (success/failure)
- [ ] Concurrency testing
- [ ] Failover testing
- [ ] Load testing
- [ ] Security testing
- [ ] Compliance testing

## Conclusion

MongoDB est adapté aux applications financières grâce à :

**✅ Forces démontrées :**
- Transactions ACID multi-documents pour intégrité
- Decimal128 pour précision exacte des montants
- Replica Sets pour haute disponibilité (99.99%+)
- Change Streams pour audit trails
- Encryption complète (at-rest, in-transit, field-level)
- Performance élevée avec consistency garantie
- Scalabilité horizontale via sharding

**⚠️ Considérations critiques :**
- Write Concern w:majority, j:true OBLIGATOIRE
- Idempotency keys pour éviter duplicates
- Reconciliation quotidienne essentielle
- Audit trails complets pour compliance
- Fraud detection temps réel
- Disaster recovery plan robuste
- Testing exhaustif avant production

**🎯 Patterns essentiels finance :**
1. **ACID transactions** pour cohérence
2. **Decimal128** pour précision monétaire
3. **Idempotency** pour sécurité
4. **Double-entry bookkeeping** pour audit
5. **Event sourcing** pour traçabilité
6. **Optimistic locking** pour concurrence
7. **Reconciliation** quotidienne

Cette architecture supporte des systèmes bancaires et financiers avec millions de transactions quotidiennes, cohérence stricte, et conformité réglementaire totale.

---

**Références :**
- MongoDB Transactions Documentation
- PCI-DSS Compliance Guide
- Basel III Requirements
- "Building Microservices" - Sam Newman
- "Release It!" - Michael Nygard (patterns résilience)

⏭️ [Architecture microservices](/20-cas-usage-architectures/10-architecture-microservices.md)
