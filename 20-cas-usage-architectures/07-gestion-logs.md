🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.7 Gestion des Logs

## Introduction

La gestion centralisée des logs est cruciale pour les systèmes modernes, nécessitant une infrastructure capable de gérer :

- **Volume massif** : Millions de logs par seconde
- **Variété** : Application logs, system logs, access logs, error logs
- **Ingestion haute performance** : Latence minimale pour ne pas impacter les applications
- **Recherche rapide** : Filtrage et full-text search pour troubleshooting
- **Rétention intelligente** : Hot/warm/cold storage selon ancienneté
- **Agrégations complexes** : Patterns, trends, anomalies
- **Alerting** : Détection d'erreurs et patterns anormaux
- **Conformité** : Audit trails et retention policies
- **Scalabilité** : De Go à Po de données

MongoDB excelle dans ce contexte grâce à :
- **Time Series Collections** : Optimisation pour données temporelles
- **Schéma flexible** : Différents formats de logs sans migration
- **Text Search** : Recherche full-text performante
- **Agrégations** : Analytics et pattern detection
- **TTL Index** : Cleanup automatique selon rétention
- **Sharding** : Distribution pour millions de logs/seconde

## Architecture de référence

### Stack de gestion de logs

```
┌─────────────────────────────────────────────────────┐
│              Log Sources                            │
│  Applications • Services • Servers • Containers     │
└────────────────────┬────────────────────────────────┘
                     │
                     │ Syslog / HTTP / Files
                     │
        ┌────────────▼────────────┐
        │    Log Collectors       │
        │  Fluentd • Logstash     │
        │  Filebeat • Vector      │
        └────────────┬────────────┘
                     │
                     │ JSON / Structured
                     │
        ┌────────────▼────────────┐
        │    Message Queue        │
        │   (Kafka / RabbitMQ)    │
        └────────────┬────────────┘
                     │
     ┌───────────────┼──────────────┐
     │               │              │
┌────▼─────┐   ┌─────▼────┐   ┌─────▼───────┐
│  Parser  │   │Enricher  │   │   Filter    │
│          │   │          │   │             │
└────┬─────┘   └────┬─────┘   └─────┬───────┘
     │              │               │
     └──────────────┴───────────────┘
                    │
     ┌──────────────▼─────────────┐
     │   MongoDB Cluster          │
     │  ┌──────────────────┐      │
     │  │  Logs (Hot)      │      │
     │  │  Time Series     │      │
     │  │  Last 7 days     │      │
     │  └──────────────────┘      │
     │  ┌──────────────────┐      │
     │  │  Logs (Warm)     │      │
     │  │  7-30 days       │      │
     │  └──────────────────┘      │
     │  ┌──────────────────┐      │
     │  │  Aggregated      │      │
     │  │  Long-term       │      │
     │  └──────────────────┘      │
     └────────────────────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼────┐    ┌─────▼────┐   ┌──────▼────┐
│ Search  │    │Analytics │   │ Alerting  │
│   UI    │    │Dashboard │   │  Engine   │
└─────────┘    └──────────┘   └───────────┘
     │               │               │
     └───────────────┴───────────────┘
                     │
        ┌────────────▼────────────┐
        │    Archive Storage      │
        │   (S3 / Object Store)   │
        └─────────────────────────┘
```

### Composants architecturaux

#### 1. Log Collectors
**Technologies :** Fluentd, Logstash, Filebeat, Vector

**Justification :**
- Collecte depuis multiples sources
- Parsing et transformation
- Buffer pour resilience
- Routing et filtrage

#### 2. Message Queue
**Technologies :** Kafka, RabbitMQ, AWS Kinesis

**Justification :**
- Découplage ingestion/storage
- Buffer pour pics de charge
- Replay capability
- Multiple consumers

#### 3. MongoDB pour Logs
**Justification :**
- Time Series Collections optimisées
- Schéma flexible pour formats variés
- Text search pour troubleshooting
- Agrégations pour analytics
- TTL automatique

#### 4. Archive Storage
**Technologies :** S3, MinIO, Azure Blob

**Justification :**
- Coût réduit pour long-term
- Compliance requirements
- Cold storage

## Modélisation des données

### 1. Logs structurés (Time Series)

```javascript
// Collection: logs (Time Series Collection)
db.createCollection("logs", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  },
  expireAfterSeconds: 604800  // 7 jours
});

// Structure de log
{
  timestamp: ISODate("2024-12-09T14:30:15.234Z"),

  // Métadonnées (indexées automatiquement)
  metadata: {
    // Source
    source: {
      application: "api-gateway",
      service: "auth-service",
      version: "2.3.1",
      environment: "production",

      // Infrastructure
      host: "api-gateway-pod-42",
      hostname: "api-gateway-42.cluster.local",
      ip: "10.0.1.42",
      container_id: "abc123def456",

      // Kubernetes (si applicable)
      k8s: {
        namespace: "production",
        pod: "api-gateway-deployment-7d5f6c8b9-42xyz",
        node: "node-01",
        cluster: "prod-cluster-eu-west-1"
      }
    },

    // Niveau de log
    level: "error",  // debug, info, warn, error, fatal

    // Catégorie
    category: "authentication",  // authentication, database, api, system

    // Tags pour recherche
    tags: ["auth", "login", "failure"]
  },

  // Message du log
  message: "Authentication failed: Invalid credentials",

  // Données structurées du log
  data: {
    // Context applicatif
    user_id: "user_abc123",
    session_id: "session_xyz789",
    request_id: "req_def456ghi789",
    correlation_id: "corr_jkl012mno345",

    // Détails de l'erreur
    error: {
      type: "AuthenticationError",
      code: "AUTH_001",
      message: "Invalid credentials provided",

      // Stack trace (si applicable)
      stack: "Error: Invalid credentials\n  at authenticate (/app/auth.js:45:15)\n  ...",

      // Détails additionnels
      details: {
        username: "john.doe@example.com",
        attempts: 3,
        ip_address: "192.168.1.100",
        user_agent: "Mozilla/5.0..."
      }
    },

    // Request details (pour API logs)
    request: {
      method: "POST",
      path: "/api/v1/auth/login",
      query: {},
      headers: {
        "content-type": "application/json",
        "x-request-id": "req_def456ghi789",
        "user-agent": "Mozilla/5.0..."
      },
      body_size: 128,
      remote_addr: "192.168.1.100"
    },

    // Response details
    response: {
      status_code: 401,
      body_size: 64,
      duration_ms: 45
    },

    // Performance metrics
    metrics: {
      cpu_usage: 0.45,
      memory_mb: 512,
      duration_ms: 45,
      db_queries: 2,
      db_duration_ms: 23
    }
  },

  // Contexte additionnel
  context: {
    trace_id: "trace_abc123",  // Distributed tracing
    span_id: "span_def456",
    parent_span_id: "span_ghi789",

    // Business context
    tenant_id: "tenant_001",
    organization_id: "org_456"
  }
}

// Index pour logs
db.logs.createIndex({
  "metadata.level": 1,
  timestamp: -1
});

db.logs.createIndex({
  "metadata.source.application": 1,
  "metadata.source.service": 1,
  timestamp: -1
});

db.logs.createIndex({
  "metadata.level": 1,
  "metadata.source.application": 1,
  timestamp: -1
});

db.logs.createIndex({
  "data.request_id": 1
});

db.logs.createIndex({
  "data.correlation_id": 1,
  timestamp: 1
});

// Index text pour recherche full-text
db.logs.createIndex({
  message: "text",
  "data.error.message": "text",
  "data.error.stack": "text"
}, {
  weights: {
    message: 10,
    "data.error.message": 5,
    "data.error.stack": 1
  }
});

// Index pour tags
db.logs.createIndex({ "metadata.tags": 1, timestamp: -1 });
```

### 2. Logs agrégés (analytics)

```javascript
// Collection: log_aggregates
{
  _id: ObjectId("..."),

  // Période d'agrégation
  period: "hour",  // minute, hour, day
  timestamp: ISODate("2024-12-09T14:00:00Z"),

  // Dimensions
  dimensions: {
    application: "api-gateway",
    service: "auth-service",
    environment: "production",
    level: "error"
  },

  // Métriques agrégées
  metrics: {
    count: 1547,

    // Par niveau
    byLevel: {
      debug: 234,
      info: 892,
      warn: 345,
      error: 76,
      fatal: 0
    },

    // Erreurs les plus fréquentes
    topErrors: [
      {
        type: "AuthenticationError",
        code: "AUTH_001",
        count: 45,
        percentage: 59.2
      },
      {
        type: "DatabaseError",
        code: "DB_TIMEOUT",
        count: 23,
        percentage: 30.3
      },
      {
        type: "ValidationError",
        code: "VAL_001",
        count: 8,
        percentage: 10.5
      }
    ],

    // Performance
    performance: {
      avgDuration: 45.7,
      p50Duration: 42,
      p95Duration: 89,
      p99Duration: 156,

      avgCpu: 0.45,
      avgMemory: 512
    },

    // HTTP status codes
    statusCodes: {
      "200": 1234,
      "401": 145,
      "404": 89,
      "500": 79
    }
  },

  // Patterns détectés
  patterns: [
    {
      type: "error_spike",
      severity: "warning",
      description: "Error rate increased by 150%",
      detected_at: ISODate("2024-12-09T14:15:00Z")
    }
  ],

  computedAt: ISODate("2024-12-09T15:00:05Z")
}

// Index pour agrégats
db.log_aggregates.createIndex({
  period: 1,
  timestamp: -1,
  "dimensions.application": 1
});

db.log_aggregates.createIndex({
  period: 1,
  "dimensions.environment": 1,
  "dimensions.level": 1,
  timestamp: -1
});

// TTL pour cleanup (garder 90 jours)
db.log_aggregates.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 7776000 }
);
```

### 3. Règles d'alerting

```javascript
// Collection: log_alert_rules
{
  _id: ObjectId("..."),
  name: "High Error Rate Alert",
  description: "Alert when error rate exceeds threshold",

  // Condition
  condition: {
    type: "threshold",  // threshold, pattern, anomaly

    // Query selector
    query: {
      "metadata.level": "error",
      "metadata.source.application": "api-gateway",
      "metadata.source.environment": "production"
    },

    // Seuil
    threshold: {
      metric: "count",
      operator: "greater_than",
      value: 100,
      window: 300  // 5 minutes
    },

    // Ou pattern matching
    pattern: {
      message: /database.*timeout/i,
      minOccurrences: 10,
      window: 60  // 1 minute
    }
  },

  // Severity
  severity: "critical",  // info, warning, critical

  // Actions
  actions: [
    {
      type: "email",
      recipients: ["oncall@example.com"],
      template: "high_error_rate"
    },
    {
      type: "pagerduty",
      service_key: "service_abc123",
      escalation_policy: "primary"
    },
    {
      type: "webhook",
      url: "https://api.example.com/alerts",
      method: "POST"
    },
    {
      type: "slack",
      channel: "#alerts-production",
      webhook_url: "https://hooks.slack.com/services/..."
    }
  ],

  // Throttling
  throttle: {
    enabled: true,
    cooldownPeriod: 900  // 15 minutes
  },

  // Status
  enabled: true,
  createdAt: ISODate("2024-01-01T00:00:00Z"),
  updatedAt: ISODate("2024-12-01T00:00:00Z")
}

// Index pour règles
db.log_alert_rules.createIndex({ enabled: 1 });
```

### 4. Traces distribuées

```javascript
// Collection: traces
{
  _id: ObjectId("..."),
  trace_id: "trace_abc123def456",

  // Root span
  root_span: {
    span_id: "span_root_001",
    operation: "POST /api/v1/orders",
    service: "api-gateway",

    start_time: ISODate("2024-12-09T14:30:00.000Z"),
    end_time: ISODate("2024-12-09T14:30:01.234Z"),
    duration_ms: 1234,

    status: "error",
    error: {
      type: "DatabaseError",
      message: "Connection timeout"
    }
  },

  // All spans dans la trace
  spans: [
    {
      span_id: "span_root_001",
      parent_span_id: null,
      service: "api-gateway",
      operation: "POST /api/v1/orders",
      start_time: ISODate("2024-12-09T14:30:00.000Z"),
      duration_ms: 1234,

      tags: {
        http_method: "POST",
        http_url: "/api/v1/orders",
        http_status_code: 500
      },

      logs: [
        {
          timestamp: ISODate("2024-12-09T14:30:00.050Z"),
          message: "Validating order",
          level: "info"
        },
        {
          timestamp: ISODate("2024-12-09T14:30:01.200Z"),
          message: "Database timeout",
          level: "error"
        }
      ]
    },
    {
      span_id: "span_auth_002",
      parent_span_id: "span_root_001",
      service: "auth-service",
      operation: "authenticate_user",
      start_time: ISODate("2024-12-09T14:30:00.010Z"),
      duration_ms: 45,

      tags: {
        user_id: "user_abc123"
      }
    },
    {
      span_id: "span_db_003",
      parent_span_id: "span_root_001",
      service: "order-service",
      operation: "insert_order",
      start_time: ISODate("2024-12-09T14:30:00.100Z"),
      duration_ms: 1100,  // Lent!

      tags: {
        db_type: "postgresql",
        db_statement: "INSERT INTO orders..."
      },

      error: {
        type: "TimeoutError",
        message: "Query execution timeout"
      }
    }
  ],

  // Metadata
  environment: "production",

  // Durée totale
  total_duration_ms: 1234,

  // Timestamps
  start_time: ISODate("2024-12-09T14:30:00.000Z"),
  end_time: ISODate("2024-12-09T14:30:01.234Z")
}

// Index pour traces
db.traces.createIndex({ trace_id: 1 });
db.traces.createIndex({ "root_span.service": 1, start_time: -1 });
db.traces.createIndex({ "root_span.status": 1, start_time: -1 });
db.traces.createIndex({ start_time: -1 });

// TTL pour cleanup (garder 30 jours)
db.traces.createIndex(
  { start_time: 1 },
  { expireAfterSeconds: 2592000 }
);
```

## Services de traitement

### 1. Service d'ingestion

```javascript
class LogIngestionService {
  constructor(db, kafka) {
    this.db = db;
    this.kafka = kafka;
    this.buffer = [];
    this.bufferSize = 1000;
    this.flushInterval = 1000;  // 1 seconde
    this.flushTimer = null;
  }

  async start() {
    // Consumer Kafka
    const consumer = this.kafka.consumer({ groupId: 'log-ingestion' });
    await consumer.connect();
    await consumer.subscribe({ topic: 'logs', fromBeginning: false });

    await consumer.run({
      eachMessage: async ({ message }) => {
        try {
          const log = JSON.parse(message.value.toString());

          // Parser et enrichir
          const enrichedLog = await this.parseAndEnrich(log);

          // Buffer pour batch insert
          this.buffer.push(enrichedLog);

          if (this.buffer.length >= this.bufferSize) {
            await this.flush();
          } else if (!this.flushTimer) {
            this.flushTimer = setTimeout(() => this.flush(), this.flushInterval);
          }
        } catch (error) {
          console.error('Failed to process log:', error);
          // Dead letter queue
          await this.sendToDeadLetter(message);
        }
      }
    });

    console.log('Log ingestion service started');
  }

  async parseAndEnrich(rawLog) {
    // Parser selon format
    let parsed;

    if (typeof rawLog === 'string') {
      // Parse formats communs
      parsed = this.parseLogString(rawLog);
    } else {
      parsed = rawLog;
    }

    // Enrichir avec métadonnées
    const enriched = {
      timestamp: new Date(parsed.timestamp || Date.now()),

      metadata: {
        source: {
          application: parsed.application || 'unknown',
          service: parsed.service || 'unknown',
          version: parsed.version || 'unknown',
          environment: parsed.environment || 'unknown',
          host: parsed.host || 'unknown',
          hostname: parsed.hostname || 'unknown',
          ip: parsed.ip || 'unknown'
        },

        level: this.normalizeLevel(parsed.level),
        category: parsed.category || this.inferCategory(parsed),
        tags: parsed.tags || []
      },

      message: parsed.message || parsed.msg || '',

      data: {
        ...parsed.data,
        request_id: parsed.request_id || parsed.requestId,
        correlation_id: parsed.correlation_id || parsed.correlationId
      }
    };

    // Géolocalisation si IP disponible
    if (parsed.ip) {
      enriched.geo = await this.geolocateIP(parsed.ip);
    }

    return enriched;
  }

  parseLogString(logString) {
    // Patterns communs de logs
    const patterns = [
      // Apache/Nginx combined log format
      /^(\S+) \S+ \S+ \[([^\]]+)\] "(\S+) (\S+) \S+" (\d+) (\d+) "([^"]*)" "([^"]*)"/,

      // Syslog format
      /^<(\d+)>(\w+\s+\d+\s+\d+:\d+:\d+) (\S+) (\S+): (.*)$/,

      // JSON log
      /^\{.*\}$/
    ];

    for (const pattern of patterns) {
      const match = logString.match(pattern);
      if (match) {
        return this.extractFromPattern(match, pattern);
      }
    }

    // Fallback: message brut
    return {
      message: logString,
      level: 'info',
      timestamp: new Date()
    };
  }

  normalizeLevel(level) {
    const normalized = {
      'TRACE': 'debug',
      'DEBUG': 'debug',
      'INFO': 'info',
      'WARN': 'warn',
      'WARNING': 'warn',
      'ERROR': 'error',
      'FATAL': 'fatal',
      'CRITICAL': 'fatal'
    };

    return normalized[level?.toUpperCase()] || 'info';
  }

  inferCategory(log) {
    // Inférer catégorie depuis le contenu
    const message = (log.message || '').toLowerCase();

    if (message.includes('auth') || message.includes('login')) {
      return 'authentication';
    }
    if (message.includes('database') || message.includes('sql')) {
      return 'database';
    }
    if (message.includes('api') || message.includes('http')) {
      return 'api';
    }

    return 'system';
  }

  async flush() {
    if (this.buffer.length === 0) return;

    clearTimeout(this.flushTimer);
    this.flushTimer = null;

    const batch = this.buffer.splice(0);

    try {
      // Bulk insert
      await this.db.collection('logs').insertMany(
        batch,
        {
          ordered: false,
          writeConcern: { w: 1, j: false }  // Performance over durability
        }
      );

      console.log(`Flushed ${batch.length} logs`);
    } catch (error) {
      console.error('Batch insert failed:', error);

      // Retry ou dead letter queue
      await this.handleFailedBatch(batch, error);
    }
  }

  async handleFailedBatch(batch, error) {
    // Stocker dans collection de failed logs
    await this.db.collection('failed_logs').insertMany(
      batch.map(log => ({
        ...log,
        error: error.message,
        failedAt: new Date()
      })),
      { ordered: false }
    );
  }

  async geolocateIP(ip) {
    // Cache ou service de géolocalisation
    // Exemple simplifié
    return {
      country: 'FR',
      city: 'Paris',
      latitude: 48.8566,
      longitude: 2.3522
    };
  }
}
```

### 2. Service de recherche

```javascript
class LogSearchService {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
  }

  async search(query) {
    const {
      text,
      level,
      application,
      service,
      environment,
      startDate,
      endDate,
      tags,
      limit = 100,
      offset = 0
    } = query;

    // Construire cache key
    const cacheKey = `log_search:${JSON.stringify(query)}`;

    // Essayer cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // Construire query MongoDB
    const matchStage = {
      timestamp: {
        $gte: new Date(startDate),
        $lte: new Date(endDate)
      }
    };

    // Filtres
    if (level) {
      matchStage['metadata.level'] = level;
    }

    if (application) {
      matchStage['metadata.source.application'] = application;
    }

    if (service) {
      matchStage['metadata.source.service'] = service;
    }

    if (environment) {
      matchStage['metadata.source.environment'] = environment;
    }

    if (tags && tags.length > 0) {
      matchStage['metadata.tags'] = { $in: tags };
    }

    // Text search
    if (text) {
      matchStage.$text = { $search: text };
    }

    const pipeline = [
      { $match: matchStage },

      // Score pour text search
      ...(text ? [{
        $addFields: {
          score: { $meta: 'textScore' }
        }
      }] : []),

      // Sort
      {
        $sort: text
          ? { score: { $meta: 'textScore' }, timestamp: -1 }
          : { timestamp: -1 }
      },

      // Pagination
      { $skip: offset },
      { $limit: limit },

      // Projection
      {
        $project: {
          timestamp: 1,
          'metadata.level': 1,
          'metadata.source.application': 1,
          'metadata.source.service': 1,
          message: 1,
          'data.request_id': 1,
          'data.error.type': 1,
          ...(text ? { score: 1 } : {})
        }
      }
    ];

    // Exécuter query
    const results = await this.db.collection('logs')
      .aggregate(pipeline)
      .toArray();

    // Compter total
    const total = await this.db.collection('logs')
      .countDocuments(matchStage);

    const response = {
      results,
      total,
      limit,
      offset,
      pages: Math.ceil(total / limit)
    };

    // Cacher pour 30 secondes
    await this.redis.setex(cacheKey, 30, JSON.stringify(response));

    return response;
  }

  async getLogContext(logId, contextSize = 10) {
    // Récupérer log spécifique
    const log = await this.db.collection('logs')
      .findOne({ _id: ObjectId(logId) });

    if (!log) {
      return null;
    }

    // Récupérer logs avant et après
    const before = await this.db.collection('logs')
      .find({
        'metadata.source.application': log.metadata.source.application,
        'metadata.source.service': log.metadata.source.service,
        'data.request_id': log.data.request_id,
        timestamp: { $lt: log.timestamp }
      })
      .sort({ timestamp: -1 })
      .limit(contextSize)
      .toArray();

    const after = await this.db.collection('logs')
      .find({
        'metadata.source.application': log.metadata.source.application,
        'metadata.source.service': log.metadata.source.service,
        'data.request_id': log.data.request_id,
        timestamp: { $gt: log.timestamp }
      })
      .sort({ timestamp: 1 })
      .limit(contextSize)
      .toArray();

    return {
      log,
      context: {
        before: before.reverse(),
        after
      }
    };
  }

  async getErrorPatterns(timeRange, minOccurrences = 10) {
    const cutoffTime = new Date(Date.now() - timeRange);

    const pipeline = [
      {
        $match: {
          'metadata.level': { $in: ['error', 'fatal'] },
          timestamp: { $gte: cutoffTime }
        }
      },

      // Grouper par type d'erreur et message
      {
        $group: {
          _id: {
            type: '$data.error.type',
            message: '$data.error.message',
            application: '$metadata.source.application',
            service: '$metadata.source.service'
          },

          count: { $sum: 1 },
          firstSeen: { $min: '$timestamp' },
          lastSeen: { $max: '$timestamp' },

          // Exemples
          examples: {
            $push: {
              $cond: [
                { $lte: [{ $size: '$examples' }, 3] },
                {
                  id: '$_id',
                  timestamp: '$timestamp',
                  request_id: '$data.request_id'
                },
                '$$REMOVE'
              ]
            }
          }
        }
      },

      // Filtrer par occurrences minimales
      {
        $match: {
          count: { $gte: minOccurrences }
        }
      },

      // Sort par fréquence
      {
        $sort: { count: -1 }
      },

      {
        $limit: 50
      },

      // Format final
      {
        $project: {
          _id: 0,
          pattern: {
            type: '$_id.type',
            message: '$_id.message',
            application: '$_id.application',
            service: '$_id.service'
          },
          count: 1,
          firstSeen: 1,
          lastSeen: 1,
          duration_minutes: {
            $divide: [
              { $subtract: ['$lastSeen', '$firstSeen'] },
              60000
            ]
          },
          examples: { $slice: ['$examples', 3] }
        }
      }
    ];

    return this.db.collection('logs')
      .aggregate(pipeline)
      .toArray();
  }

  async getCorrelatedLogs(correlationId) {
    // Récupérer tous les logs d'une même correlation
    const logs = await this.db.collection('logs')
      .find({
        'data.correlation_id': correlationId
      })
      .sort({ timestamp: 1 })
      .toArray();

    // Construire timeline
    const timeline = {
      correlation_id: correlationId,
      logs: logs.map(log => ({
        timestamp: log.timestamp,
        service: log.metadata.source.service,
        level: log.metadata.level,
        message: log.message,
        duration_from_start: log.timestamp - logs[0].timestamp
      })),

      total_duration: logs[logs.length - 1].timestamp - logs[0].timestamp,
      services_involved: [...new Set(logs.map(l => l.metadata.source.service))],
      error_count: logs.filter(l => l.metadata.level === 'error').length
    };

    return timeline;
  }
}
```

### 3. Service d'alerting

```javascript
class LogAlertingService {
  constructor(db) {
    this.db = db;
    this.alertState = new Map();
  }

  async start() {
    // Charger règles d'alerte
    await this.loadAlertRules();

    // Watch sur logs
    const changeStream = this.db.collection('logs').watch([
      {
        $match: {
          operationType: 'insert',
          'fullDocument.metadata.level': { $in: ['error', 'fatal'] }
        }
      }
    ]);

    changeStream.on('change', async (change) => {
      const log = change.fullDocument;
      await this.evaluateAlerts(log);
    });

    // Job périodique pour alertes sur agrégats
    setInterval(() => {
      this.evaluateAggregateAlerts();
    }, 60000);  // Chaque minute

    console.log('Log alerting service started');
  }

  async loadAlertRules() {
    const rules = await this.db.collection('log_alert_rules')
      .find({ enabled: true })
      .toArray();

    this.alertRules = rules;
    console.log(`Loaded ${rules.length} alert rules`);
  }

  async evaluateAlerts(log) {
    for (const rule of this.alertRules) {
      if (this.logMatchesRule(log, rule)) {
        await this.triggerAlert(rule, log);
      }
    }
  }

  logMatchesRule(log, rule) {
    const { query } = rule.condition;

    // Vérifier chaque condition
    for (const [key, value] of Object.entries(query)) {
      const logValue = this.getNestedValue(log, key);

      if (Array.isArray(value)) {
        // $in operator
        if (!value.includes(logValue)) {
          return false;
        }
      } else if (value instanceof RegExp) {
        if (!value.test(logValue)) {
          return false;
        }
      } else {
        if (logValue !== value) {
          return false;
        }
      }
    }

    // Vérifier pattern si défini
    if (rule.condition.pattern) {
      const pattern = rule.condition.pattern;
      if (!pattern.message.test(log.message)) {
        return false;
      }
    }

    return true;
  }

  async evaluateAggregateAlerts() {
    for (const rule of this.alertRules) {
      if (rule.condition.type === 'threshold') {
        await this.evaluateThresholdAlert(rule);
      }
    }
  }

  async evaluateThresholdAlert(rule) {
    const { query, threshold } = rule.condition;
    const { window } = threshold;

    const cutoffTime = new Date(Date.now() - window * 1000);

    // Compter logs matchant dans la fenêtre
    const count = await this.db.collection('logs').countDocuments({
      ...query,
      timestamp: { $gte: cutoffTime }
    });

    // Évaluer seuil
    let triggered = false;

    switch (threshold.operator) {
      case 'greater_than':
        triggered = count > threshold.value;
        break;
      case 'less_than':
        triggered = count < threshold.value;
        break;
      case 'equal':
        triggered = count === threshold.value;
        break;
    }

    if (triggered) {
      // Vérifier throttling
      const alertKey = `${rule._id}_${Math.floor(Date.now() / (rule.throttle.cooldownPeriod * 1000))}`;

      if (!this.alertState.has(alertKey)) {
        await this.sendAlert(rule, {
          metric: threshold.metric,
          value: count,
          threshold: threshold.value,
          window: window
        });

        this.alertState.set(alertKey, {
          triggeredAt: new Date(),
          count
        });
      }
    }
  }

  async triggerAlert(rule, log) {
    // Pattern matching alert (déclenché par log individuel)
    const alertKey = `${rule._id}_${rule.condition.pattern.message}`;

    // Vérifier throttling
    if (this.alertState.has(alertKey)) {
      const lastAlert = this.alertState.get(alertKey);
      const elapsed = Date.now() - lastAlert.triggeredAt.getTime();

      if (elapsed < rule.throttle.cooldownPeriod * 1000) {
        return;  // Encore en cooldown
      }
    }

    await this.sendAlert(rule, {
      log,
      pattern: rule.condition.pattern.message
    });

    this.alertState.set(alertKey, {
      triggeredAt: new Date()
    });
  }

  async sendAlert(rule, context) {
    // Créer alerte
    const alert = {
      ruleId: rule._id,
      ruleName: rule.name,
      severity: rule.severity,
      context,
      triggeredAt: new Date(),
      status: 'active'
    };

    // Sauvegarder
    await this.db.collection('log_alerts').insertOne(alert);

    // Envoyer notifications
    for (const action of rule.actions) {
      try {
        await this.executeAction(action, alert);
      } catch (error) {
        console.error(`Failed to execute action ${action.type}:`, error);
      }
    }

    console.log(`Alert triggered: ${rule.name}`);
  }

  async executeAction(action, alert) {
    switch (action.type) {
      case 'email':
        await this.sendEmail(action, alert);
        break;
      case 'slack':
        await this.sendSlack(action, alert);
        break;
      case 'pagerduty':
        await this.sendPagerDuty(action, alert);
        break;
      case 'webhook':
        await this.sendWebhook(action, alert);
        break;
    }
  }

  async sendSlack(action, alert) {
    const message = {
      channel: action.channel,
      text: `🚨 *${alert.severity.toUpperCase()}*: ${alert.ruleName}`,
      attachments: [
        {
          color: alert.severity === 'critical' ? 'danger' : 'warning',
          fields: [
            {
              title: 'Context',
              value: JSON.stringify(alert.context, null, 2),
              short: false
            },
            {
              title: 'Triggered At',
              value: alert.triggeredAt.toISOString(),
              short: true
            }
          ]
        }
      ]
    };

    await fetch(action.webhook_url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message)
    });
  }

  getNestedValue(obj, path) {
    return path.split('.').reduce((current, key) => current?.[key], obj);
  }
}
```

### 4. Service d'agrégation

```javascript
class LogAggregationService {
  async aggregateHourlyMetrics() {
    const now = new Date();
    const oneHourAgo = new Date(now - 3600000);
    const twoHoursAgo = new Date(now - 7200000);

    const pipeline = [
      {
        $match: {
          timestamp: {
            $gte: twoHoursAgo,
            $lt: oneHourAgo
          }
        }
      },

      {
        $group: {
          _id: {
            hour: {
              $dateTrunc: {
                date: '$timestamp',
                unit: 'hour'
              }
            },
            application: '$metadata.source.application',
            service: '$metadata.source.service',
            environment: '$metadata.source.environment',
            level: '$metadata.level'
          },

          count: { $sum: 1 },

          // Par niveau
          debugCount: {
            $sum: {
              $cond: [{ $eq: ['$metadata.level', 'debug'] }, 1, 0]
            }
          },
          infoCount: {
            $sum: {
              $cond: [{ $eq: ['$metadata.level', 'info'] }, 1, 0]
            }
          },
          warnCount: {
            $sum: {
              $cond: [{ $eq: ['$metadata.level', 'warn'] }, 1, 0]
            }
          },
          errorCount: {
            $sum: {
              $cond: [{ $eq: ['$metadata.level', 'error'] }, 1, 0]
            }
          },
          fatalCount: {
            $sum: {
              $cond: [{ $eq: ['$metadata.level', 'fatal'] }, 1, 0]
            }
          },

          // Erreurs
          errors: {
            $push: {
              $cond: [
                { $in: ['$metadata.level', ['error', 'fatal']] },
                {
                  type: '$data.error.type',
                  code: '$data.error.code',
                  message: '$data.error.message'
                },
                '$$REMOVE'
              ]
            }
          },

          // Performance
          avgDuration: { $avg: '$data.response.duration_ms' },
          p95Duration: {
            $percentile: {
              input: '$data.response.duration_ms',
              p: [0.95],
              method: 'approximate'
            }
          }
        }
      },

      // Agréger erreurs
      {
        $addFields: {
          topErrors: {
            $slice: [
              {
                $sortArray: {
                  input: {
                    $map: {
                      input: {
                        $setUnion: [
                          { $map: { input: '$errors', in: '$$this.type' } }
                        ]
                      },
                      as: 'errorType',
                      in: {
                        type: '$$errorType',
                        count: {
                          $size: {
                            $filter: {
                              input: '$errors',
                              cond: { $eq: ['$$this.type', '$$errorType'] }
                            }
                          }
                        },
                        percentage: {
                          $multiply: [
                            {
                              $divide: [
                                {
                                  $size: {
                                    $filter: {
                                      input: '$errors',
                                      cond: { $eq: ['$$this.type', '$$errorType'] }
                                    }
                                  }
                                },
                                { $size: '$errors' }
                              ]
                            },
                            100
                          ]
                        }
                      }
                    }
                  },
                  sortBy: { count: -1 }
                }
              },
              10
            ]
          }
        }
      },

      // Format final
      {
        $project: {
          _id: 0,
          period: 'hour',
          timestamp: '$_id.hour',

          dimensions: {
            application: '$_id.application',
            service: '$_id.service',
            environment: '$_id.environment',
            level: '$_id.level'
          },

          metrics: {
            count: '$count',

            byLevel: {
              debug: '$debugCount',
              info: '$infoCount',
              warn: '$warnCount',
              error: '$errorCount',
              fatal: '$fatalCount'
            },

            topErrors: '$topErrors',

            performance: {
              avgDuration: { $round: ['$avgDuration', 2] },
              p95Duration: { $arrayElemAt: ['$p95Duration', 0] }
            }
          },

          computedAt: new Date()
        }
      },

      {
        $merge: {
          into: 'log_aggregates',
          on: ['period', 'timestamp', 'dimensions'],
          whenMatched: 'replace',
          whenNotMatched: 'insert'
        }
      }
    ];

    await this.db.collection('logs')
      .aggregate(pipeline)
      .toArray();

    console.log(`Aggregated hourly logs for ${oneHourAgo.toISOString()}`);
  }
}
```

## Performance et optimisation

### 1. Stratégies d'indexation avancées

```javascript
// Index pour queries fréquentes

// 1. Recherche par niveau et date
db.logs.createIndex({
  'metadata.level': 1,
  timestamp: -1
}, {
  name: 'level_time_idx'
});

// 2. Recherche par application/service
db.logs.createIndex({
  'metadata.source.application': 1,
  'metadata.source.service': 1,
  timestamp: -1
}, {
  name: 'app_service_time_idx'
});

// 3. Index partiel pour erreurs uniquement
db.logs.createIndex(
  {
    'metadata.level': 1,
    timestamp: -1,
    'data.error.type': 1
  },
  {
    name: 'errors_only_idx',
    partialFilterExpression: {
      'metadata.level': { $in: ['error', 'fatal'] }
    }
  }
);

// 4. Index pour correlation
db.logs.createIndex({
  'data.correlation_id': 1,
  timestamp: 1
}, {
  name: 'correlation_idx'
});

// 5. Index couvrant pour dashboard
db.logs.createIndex({
  'metadata.source.application': 1,
  'metadata.level': 1,
  timestamp: -1,
  message: 1
}, {
  name: 'dashboard_covering_idx'
});
```

### 2. Configuration MongoDB pour logs

```javascript
// Configuration optimale pour logs
const mongoConfig = {
  // WiredTiger
  storage: {
    wiredTiger: {
      engineConfig: {
        cacheSizeGB: 64,
        journalCompressor: 'snappy',
        directoryForIndexes: true
      },
      collectionConfig: {
        blockCompressor: 'zstd'  // Meilleure compression
      }
    }
  },

  // Oplog pour Change Streams
  replication: {
    oplogSizeMB: 100000  // 100 GB pour haute fréquence
  },

  // Write Concern pour logs
  writeConcern: {
    w: 1,
    j: false  // Performance over durability for logs
  }
};

// Sharding pour très gros volumes
sh.enableSharding('logs');

// Shard key: timestamp + application
// Permet range queries efficaces et distribution équitable
sh.shardCollection(
  'logs.logs',
  { timestamp: 1, 'metadata.source.application': 'hashed' }
);

// Zone sharding pour hot/warm/cold
sh.addShardToZone('shard01', 'hot');    // SSD
sh.addShardToZone('shard02', 'warm');   // SSD
sh.addShardToZone('shard03', 'cold');   // HDD

const now = new Date();
const sevenDaysAgo = new Date(now - 7 * 24 * 3600000);
const thirtyDaysAgo = new Date(now - 30 * 24 * 3600000);

// Hot: derniers 7 jours
sh.updateZoneKeyRange(
  'logs.logs',
  { timestamp: sevenDaysAgo },
  { timestamp: new Date(2099, 12, 31) },
  'hot'
);

// Warm: 7-30 jours
sh.updateZoneKeyRange(
  'logs.logs',
  { timestamp: thirtyDaysAgo },
  { timestamp: sevenDaysAgo },
  'warm'
);
```

### 3. Métriques de performance

```javascript
const logsMetrics = {
  // Ingestion
  'logs.ingestion_rate': {
    description: 'Logs ingested per second',
    target: 100000,
    alert_threshold: 50000
  },

  'logs.ingestion_lag': {
    description: 'Time between log generation and storage',
    target: 1000,  // ms
    alert_threshold: 5000
  },

  // Storage
  'logs.storage_growth': {
    description: 'Storage growth per day (GB)',
    monitor: true
  },

  'logs.compression_ratio': {
    description: 'Compression ratio (Time Series)',
    target: 10,
    alert_threshold: 5
  },

  // Search
  'logs.search_latency.p95': {
    description: 'Search query latency p95',
    target: 1000,  // ms
    alert_threshold: 5000
  },

  'logs.text_search_latency.p95': {
    description: 'Full-text search latency p95',
    target: 2000,  // ms
    alert_threshold: 10000
  },

  // Alerting
  'logs.alert_detection_latency': {
    description: 'Time to detect and trigger alert',
    target: 5000,  // ms
    alert_threshold: 30000
  }
};
```

## Checklist de déploiement

### ✅ Architecture

- [ ] Log collectors déployés (Fluentd/Filebeat)
- [ ] Message queue configuré (Kafka/RabbitMQ)
- [ ] Parser et enrichment pipeline
- [ ] Time Series Collections créées
- [ ] Index optimisés
- [ ] Sharding si > 1TB/jour

### ✅ Ingestion

- [ ] Batch inserts (1000+ logs)
- [ ] Parsing pour formats communs
- [ ] Enrichissement (geo, metadata)
- [ ] Dead letter queue
- [ ] Monitoring throughput

### ✅ Rétention

- [ ] TTL configuré (7j hot, 30j warm)
- [ ] Agrégation jobs schedulés
- [ ] Archive vers S3/cold storage
- [ ] Compression activée
- [ ] Zone sharding pour tiering

### ✅ Recherche

- [ ] Text indexes créés
- [ ] API de recherche
- [ ] Contexte logs (before/after)
- [ ] Correlation logs
- [ ] Cache pour queries fréquentes

### ✅ Alerting

- [ ] Règles d'alerte configurées
- [ ] Change Streams monitoring
- [ ] Pattern detection
- [ ] Notifications (Slack, PagerDuty)
- [ ] Throttling

### ✅ Performance

- [ ] Index couvrants
- [ ] Read replicas si nécessaire
- [ ] Write Concern optimisé (w:1, j:false)
- [ ] Compression zstd
- [ ] Monitoring latence

### ✅ Opérations

- [ ] Backup (agrégats uniquement)
- [ ] Monitoring infrastructure
- [ ] Alertes opérationnelles
- [ ] Runbooks troubleshooting
- [ ] Documentation

## Conclusion

MongoDB est particulièrement adapté à la gestion de logs grâce à :

**✅ Forces démontrées :**
- Time Series Collections optimisées avec compression 10x
- Schéma flexible pour formats de logs variés
- Text search performant pour troubleshooting
- Agrégations puissantes pour analytics
- TTL automatique pour rétention
- Change Streams pour alerting temps réel
- Sharding pour millions de logs/seconde

**⚠️ Considérations importantes :**
- Write Concern w:1, j:false recommandé (performance)
- Agrégation essentielle pour long-term analytics
- Archive vers object storage pour cold data
- Index text search peut être coûteux
- Monitoring de croissance storage critique

**🎯 Patterns essentiels logs :**
1. **Time Series Collections** pour logs bruts
2. **Batch inserts** pour ingestion haute performance
3. **Structured logging** pour parsing facilité
4. **TTL Index** pour cleanup automatique
5. **Agrégations hiérarchiques** pour analytics
6. **Zone Sharding** pour hot/warm/cold tiering
7. **Change Streams** pour alerting temps réel

Cette architecture supporte des systèmes de logs de quelques Go à plusieurs Po, avec ingestion de millions de logs par seconde et recherche sub-seconde.

---

**Références :**
- MongoDB Time Series Documentation
- "The Art of Monitoring" - James Turnbull
- Elasticsearch vs MongoDB for Logs
- Fluentd Documentation
- "Site Reliability Engineering" - Google

⏭️ [Personnalisation et recommandations](/20-cas-usage-architectures/08-personnalisation-recommandations.md)
