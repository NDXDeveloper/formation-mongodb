🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.6 Analyse en Temps Réel

## Introduction

L'analyse en temps réel (Real-Time Analytics) nécessite une infrastructure capable de traiter, agréger et visualiser des données avec une latence minimale :

- **Latence ultra-faible** : Métriques visibles < 1 seconde après l'événement
- **Throughput élevé** : Millions d'événements par seconde
- **Agrégations complexes** : Calculs multi-dimensionnels en temps réel
- **Fenêtres temporelles** : Tumbling, sliding, session windows
- **Matérialisation** : Pre-computed views pour dashboards
- **Visualisation** : Graphiques et tableaux de bord dynamiques
- **Alerting** : Détection d'anomalies et seuils franchis
- **Historique** : Drill-down sur données historiques
- **Scalabilité** : De milliers à milliards d'événements/jour

MongoDB excelle dans ce contexte grâce à :
- **Change Streams** : Réactivité instantanée aux modifications
- **Agrégation Pipeline** : Calculs complexes optimisés
- **$merge et $out** : Matérialisation de vues
- **Time Series Collections** : Optimisation pour données temporelles
- **Indexes performants** : Requêtes sub-milliseconde
- **Sharding** : Distribution de charge pour scale horizontal

## Architecture de référence

### Stack analytics temps réel

```
┌─────────────────────────────────────────────────────┐
│              Event Sources                          │
│   Applications • APIs • IoT • Logs • Transactions   │
└────────────────────┬────────────────────────────────┘
                     │
                     │ HTTP/MQTT/Kafka
                     │
        ┌────────────▼────────────┐
        │    Event Ingestion      │
        │   (Kafka/RabbitMQ)      │
        └────────────┬────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼─────┐   ┌────▼─────┐   ┌──────▼───────┐
│ Stream   │   │  Event   │   │   Raw Data   │
│Processor │   │Validator │   │   Storage    │
│(Flink)   │   │          │   │              │
└────┬─────┘   └────┬─────┘   └──────┬───────┘
     │              │                │
     │         ┌────▼────┐           │
     │         │  Redis  │           │
     │         │ (Cache) │           │
     │         └─────────┘           │
     │                               │
     └────────────┬──────────────────┘
                  │
     ┌────────────▼────────────┐
     │   MongoDB Cluster       │
     │  ┌──────────────────┐   │
     │  │  Events (Raw)    │   │
     │  │  Time Series     │   │
     │  └──────────────────┘   │
     │  ┌──────────────────┐   │
     │  │  Aggregates      │   │
     │  │  (Materialized)  │   │
     │  └──────────────────┘   │
     │  ┌──────────────────┐   │
     │  │  Metrics         │   │
     │  │  (Pre-computed)  │   │
     │  └──────────────────┘   │
     └─────────────────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
┌────▼────┐  ┌────▼────┐  ┌────▼─────┐
│Real-Time│  │Analytics│  │Alerting  │
│Dashboard│  │   API   │  │ Engine   │
│(React)  │  │         │  │          │
└─────────┘  └─────────┘  └──────────┘
     │            │            │
     └────────────┴────────────┘
                  │
        ┌─────────▼─────────┐
        │  Visualization    │
        │  (Grafana/Custom) │
        └───────────────────┘
```

### Composants architecturaux

#### 1. Event Ingestion Layer
**Technologie :** Kafka, RabbitMQ, AWS Kinesis

**Justification :**
- Buffer pour absorber pics de charge
- Replay capability pour retraitement
- Persistance pour durabilité
- Partitionnement pour parallélisme

#### 2. Stream Processing
**Technologie :** Apache Flink, Kafka Streams, Spark Streaming

**Responsabilités :**
- Agrégations sur fenêtres temporelles
- Enrichissement de données
- Filtrage et transformation
- Détection d'anomalies

#### 3. MongoDB pour Analytics
**Justification :**
- Agrégation Pipeline puissante
- Change Streams pour réactivité
- Matérialisation avec $merge
- Indexes optimisés pour queries complexes

#### 4. Cache Layer (Redis)
**Usage :**
- Métriques temps réel hot
- Compteurs atomiques
- Rate limiting
- Session state

## Modélisation des données

### 1. Événements bruts (Time Series)

```javascript
// Collection: events (Time Series Collection)
db.createCollection("events", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  },
  expireAfterSeconds: 604800  // 7 jours
});

// Structure d'événement
{
  timestamp: ISODate("2024-12-09T14:30:15.234Z"),

  // Métadonnées (indexées automatiquement)
  metadata: {
    eventType: "page_view",
    source: "web",
    version: "2.1.0",

    // Dimensions pour analyse
    dimensions: {
      userId: ObjectId("..."),
      sessionId: "session_abc123",
      deviceType: "mobile",
      browser: "Chrome",
      os: "iOS",
      country: "FR",
      city: "Paris",

      // Business dimensions
      productId: ObjectId("..."),
      categoryId: ObjectId("..."),
      campaignId: "black_friday_2024"
    }
  },

  // Métriques de l'événement
  metrics: {
    pageLoadTime: 1247,  // ms
    timeOnPage: 45,      // secondes
    scrollDepth: 0.78,   // 78%

    // Custom metrics
    customMetric1: 123.45,
    customMetric2: 67.89
  },

  // Attributs additionnels
  attributes: {
    referrer: "https://google.com",
    landingPage: "/products/winter-collection",
    exitPage: null,  // Si encore actif

    // UTM parameters
    utm_source: "google",
    utm_medium: "cpc",
    utm_campaign: "winter_sale"
  },

  // Géolocalisation
  geo: {
    country: "FR",
    city: "Paris",
    coordinates: {
      type: "Point",
      coordinates: [2.3522, 48.8566]
    },
    timezone: "Europe/Paris"
  }
}

// Index pour événements
db.events.createIndex({ "metadata.dimensions.userId": 1, timestamp: -1 });
db.events.createIndex({ "metadata.dimensions.sessionId": 1 });
db.events.createIndex({ "metadata.eventType": 1, timestamp: -1 });
db.events.createIndex({ "metadata.dimensions.country": 1, timestamp: -1 });
db.events.createIndex({ "metadata.dimensions.productId": 1, timestamp: -1 });
```

### 2. Métriques pré-calculées (agrégats)

```javascript
// Collection: metrics_minutely
{
  _id: ObjectId("..."),

  // Identifiants de l'agrégat
  metricType: "page_views",
  granularity: "minute",

  // Timestamp du bucket
  timestamp: ISODate("2024-12-09T14:30:00Z"),

  // Dimensions (pour group by)
  dimensions: {
    eventType: "page_view",
    country: "FR",
    deviceType: "mobile",
    browser: "Chrome"
  },

  // Métriques agrégées
  metrics: {
    count: 1547,
    uniqueUsers: 892,
    uniqueSessions: 734,

    // Statistiques
    avgPageLoadTime: 1247.5,
    p50PageLoadTime: 1150,
    p95PageLoadTime: 2340,
    p99PageLoadTime: 3456,

    avgTimeOnPage: 45.8,
    avgScrollDepth: 0.72,

    // Sommes
    totalPageLoadTime: 1930402.5,
    totalTimeOnPage: 70842.6
  },

  // Breakdown par sous-dimensions
  breakdown: {
    byPage: [
      { page: "/products", count: 456, avgLoadTime: 1123 },
      { page: "/home", count: 234, avgLoadTime: 987 },
      { page: "/cart", count: 189, avgLoadTime: 1456 }
    ],

    byOS: [
      { os: "iOS", count: 892, percentage: 57.7 },
      { os: "Android", count: 655, percentage: 42.3 }
    ]
  },

  // Métadonnées
  computedAt: ISODate("2024-12-09T14:31:05Z"),
  version: 1
}

// Index pour métriques
db.metrics_minutely.createIndex({
  metricType: 1,
  timestamp: -1
});

db.metrics_minutely.createIndex({
  metricType: 1,
  "dimensions.country": 1,
  timestamp: -1
});

db.metrics_minutely.createIndex({
  metricType: 1,
  "dimensions.deviceType": 1,
  timestamp: -1
});

// Index composé pour drill-down
db.metrics_minutely.createIndex({
  metricType: 1,
  "dimensions.country": 1,
  "dimensions.deviceType": 1,
  "dimensions.browser": 1,
  timestamp: -1
});

// TTL pour cleanup (garder 30 jours)
db.metrics_minutely.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 2592000 }
);
```

### 3. Métriques roulantes (rollups)

```javascript
// Collection: metrics_hourly
{
  _id: ObjectId("..."),
  metricType: "page_views",
  granularity: "hour",
  timestamp: ISODate("2024-12-09T14:00:00Z"),

  dimensions: {
    country: "FR",
    deviceType: "mobile"
  },

  metrics: {
    count: 92820,
    uniqueUsers: 53467,
    uniqueSessions: 44123,
    avgPageLoadTime: 1256.7,
    p95PageLoadTime: 2378,

    // Trend vs heure précédente
    trend: {
      countChange: +12.5,      // +12.5%
      usersChange: +8.3,       // +8.3%
      loadTimeChange: -3.2     // -3.2% (amélioration)
    }
  },

  // Top pages de l'heure
  topPages: [
    { page: "/products", count: 15678, percentage: 16.9 },
    { page: "/home", count: 12345, percentage: 13.3 },
    { page: "/sale", count: 9876, percentage: 10.6 }
  ],

  computedAt: ISODate("2024-12-09T15:00:05Z")
}

// Collection: metrics_daily (même structure)
// Collection: metrics_monthly (même structure)

// Hiérarchie de rollups:
// events (raw) → metrics_minutely → metrics_hourly → metrics_daily → metrics_monthly
```

### 4. Dashboard configuration

```javascript
// Collection: dashboards
{
  _id: ObjectId("..."),
  name: "Real-Time Analytics Dashboard",
  slug: "realtime-analytics",

  // Configuration
  config: {
    refreshInterval: 5000,  // 5 secondes
    timezone: "Europe/Paris",
    dateRange: {
      type: "relative",  // relative, absolute
      value: "last_24h"  // last_1h, last_24h, last_7d, custom
    }
  },

  // Widgets du dashboard
  widgets: [
    {
      id: "widget_1",
      type: "counter",
      title: "Active Users Now",
      position: { x: 0, y: 0, w: 3, h: 2 },

      // Query configuration
      query: {
        collection: "events",
        pipeline: [
          {
            $match: {
              timestamp: { $gte: "$$now - 5m" },
              "metadata.eventType": "page_view"
            }
          },
          {
            $group: {
              _id: null,
              activeUsers: { $addToSet: "$metadata.dimensions.userId" }
            }
          },
          {
            $project: {
              count: { $size: "$activeUsers" }
            }
          }
        ]
      },

      // Formatting
      format: {
        type: "number",
        decimals: 0,
        suffix: " users"
      }
    },

    {
      id: "widget_2",
      type: "timeseries",
      title: "Page Views (Last 24h)",
      position: { x: 3, y: 0, w: 9, h: 4 },

      query: {
        collection: "metrics_minutely",
        pipeline: [
          {
            $match: {
              metricType: "page_views",
              timestamp: { $gte: "$$now - 24h" }
            }
          },
          {
            $project: {
              timestamp: 1,
              count: "$metrics.count"
            }
          },
          {
            $sort: { timestamp: 1 }
          }
        ]
      },

      visualization: {
        chartType: "line",
        xAxis: "timestamp",
        yAxis: "count",
        interpolation: "smooth"
      }
    },

    {
      id: "widget_3",
      type: "breakdown",
      title: "Traffic by Country",
      position: { x: 0, y: 2, w: 4, h: 3 },

      query: {
        collection: "metrics_minutely",
        pipeline: [
          {
            $match: {
              metricType: "page_views",
              timestamp: { $gte: "$$now - 1h" }
            }
          },
          {
            $group: {
              _id: "$dimensions.country",
              count: { $sum: "$metrics.count" }
            }
          },
          {
            $sort: { count: -1 }
          },
          {
            $limit: 10
          }
        ]
      },

      visualization: {
        chartType: "pie",
        labelField: "_id",
        valueField: "count"
      }
    }
  ],

  // Access control
  access: {
    public: false,
    allowedUsers: [ObjectId("..."), ObjectId("...")],
    allowedRoles: ["admin", "analyst"]
  },

  // Métadonnées
  createdBy: ObjectId("..."),
  createdAt: ISODate("2024-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-12-05T14:30:00Z")
}

// Index pour dashboards
db.dashboards.createIndex({ slug: 1 }, { unique: true });
db.dashboards.createIndex({ "access.allowedUsers": 1 });
```

## Pipelines d'agrégation temps réel

### 1. Agrégation en fenêtre temporelle

```javascript
// Pipeline: Métriques des 5 dernières minutes
async function getRealTimeMetrics(metricType, minutes = 5) {
  const cutoffTime = new Date(Date.now() - minutes * 60000);

  const pipeline = [
    {
      $match: {
        "metadata.eventType": metricType,
        timestamp: { $gte: cutoffTime }
      }
    },

    // Fenêtre par minute
    {
      $group: {
        _id: {
          $dateTrunc: {
            date: "$timestamp",
            unit: "minute"
          }
        },

        // Compteurs
        count: { $sum: 1 },
        uniqueUsers: { $addToSet: "$metadata.dimensions.userId" },
        uniqueSessions: { $addToSet: "$metadata.dimensions.sessionId" },

        // Statistiques
        avgMetric: { $avg: "$metrics.pageLoadTime" },
        minMetric: { $min: "$metrics.pageLoadTime" },
        maxMetric: { $max: "$metrics.pageLoadTime" },

        // Percentiles (approximatifs)
        allMetrics: { $push: "$metrics.pageLoadTime" }
      }
    },

    // Calculer percentiles
    {
      $addFields: {
        uniqueUsersCount: { $size: "$uniqueUsers" },
        uniqueSessionsCount: { $size: "$uniqueSessions" },

        // Percentiles
        p50: {
          $arrayElemAt: [
            {
              $sortArray: {
                input: "$allMetrics",
                sortBy: 1
              }
            },
            { $floor: { $multiply: [{ $size: "$allMetrics" }, 0.5] } }
          ]
        },

        p95: {
          $arrayElemAt: [
            {
              $sortArray: {
                input: "$allMetrics",
                sortBy: 1
              }
            },
            { $floor: { $multiply: [{ $size: "$allMetrics" }, 0.95] } }
          ]
        }
      }
    },

    // Projection finale
    {
      $project: {
        _id: 0,
        minute: "$_id",
        count: 1,
        uniqueUsers: "$uniqueUsersCount",
        uniqueSessions: "$uniqueSessionsCount",

        metrics: {
          avg: { $round: ["$avgMetric", 2] },
          min: "$minMetric",
          max: "$maxMetric",
          p50: "$p50",
          p95: "$p95"
        }
      }
    },

    { $sort: { minute: 1 } }
  ];

  return db.collection('events').aggregate(pipeline).toArray();
}
```

### 2. Agrégation multi-dimensionnelle

```javascript
// Pipeline: Breakdown par multiples dimensions
async function getMultiDimensionalBreakdown(options = {}) {
  const {
    metricType = "page_view",
    timeRange = 3600000,  // 1 heure
    dimensions = ["country", "deviceType", "browser"]
  } = options;

  const cutoffTime = new Date(Date.now() - timeRange);

  const pipeline = [
    {
      $match: {
        "metadata.eventType": metricType,
        timestamp: { $gte: cutoffTime }
      }
    },

    // Facet pour chaque dimension
    {
      $facet: {
        // Breakdown par pays
        byCountry: [
          {
            $group: {
              _id: "$metadata.dimensions.country",
              count: { $sum: 1 },
              uniqueUsers: { $addToSet: "$metadata.dimensions.userId" }
            }
          },
          {
            $project: {
              _id: 0,
              country: "$_id",
              count: 1,
              uniqueUsers: { $size: "$uniqueUsers" }
            }
          },
          { $sort: { count: -1 } },
          { $limit: 10 }
        ],

        // Breakdown par type de device
        byDeviceType: [
          {
            $group: {
              _id: "$metadata.dimensions.deviceType",
              count: { $sum: 1 },
              avgLoadTime: { $avg: "$metrics.pageLoadTime" }
            }
          },
          {
            $project: {
              _id: 0,
              deviceType: "$_id",
              count: 1,
              avgLoadTime: { $round: ["$avgLoadTime", 2] }
            }
          },
          { $sort: { count: -1 } }
        ],

        // Breakdown par navigateur
        byBrowser: [
          {
            $group: {
              _id: "$metadata.dimensions.browser",
              count: { $sum: 1 },
              bounceRate: {
                $avg: {
                  $cond: [
                    { $lte: ["$metrics.timeOnPage", 10] },
                    1,
                    0
                  ]
                }
              }
            }
          },
          {
            $project: {
              _id: 0,
              browser: "$_id",
              count: 1,
              bounceRate: { $round: ["$bounceRate", 4] }
            }
          },
          { $sort: { count: -1 } },
          { $limit: 10 }
        ],

        // Métriques globales
        overall: [
          {
            $group: {
              _id: null,
              totalCount: { $sum: 1 },
              uniqueUsers: { $addToSet: "$metadata.dimensions.userId" },
              uniqueSessions: { $addToSet: "$metadata.dimensions.sessionId" },
              avgLoadTime: { $avg: "$metrics.pageLoadTime" },
              avgTimeOnPage: { $avg: "$metrics.timeOnPage" }
            }
          },
          {
            $project: {
              _id: 0,
              totalCount: 1,
              uniqueUsers: { $size: "$uniqueUsers" },
              uniqueSessions: { $size: "$uniqueSessions" },
              avgLoadTime: { $round: ["$avgLoadTime", 2] },
              avgTimeOnPage: { $round: ["$avgTimeOnPage", 2] }
            }
          }
        ]
      }
    }
  ];

  const result = await db.collection('events')
    .aggregate(pipeline)
    .toArray();

  return result[0];
}
```

### 3. Matérialisation de vues avec $merge

```javascript
// Job périodique: Matérialiser métriques minutely
class MetricsMaterializationJob {
  async materializeMinuteMetrics() {
    const now = new Date();
    const oneMinuteAgo = new Date(now - 60000);
    const twoMinutesAgo = new Date(now - 120000);

    const pipeline = [
      {
        $match: {
          timestamp: {
            $gte: twoMinutesAgo,
            $lt: oneMinuteAgo
          }
        }
      },

      // Grouper par minute et dimensions
      {
        $group: {
          _id: {
            minute: {
              $dateTrunc: {
                date: "$timestamp",
                unit: "minute"
              }
            },
            eventType: "$metadata.eventType",
            country: "$metadata.dimensions.country",
            deviceType: "$metadata.dimensions.deviceType",
            browser: "$metadata.dimensions.browser"
          },

          // Métriques
          count: { $sum: 1 },
          uniqueUsers: { $addToSet: "$metadata.dimensions.userId" },
          uniqueSessions: { $addToSet: "$metadata.dimensions.sessionId" },

          avgPageLoadTime: { $avg: "$metrics.pageLoadTime" },
          avgTimeOnPage: { $avg: "$metrics.timeOnPage" },
          avgScrollDepth: { $avg: "$metrics.scrollDepth" },

          // Pour percentiles
          pageLoadTimes: { $push: "$metrics.pageLoadTime" },

          // Breakdown
          pageViews: {
            $push: {
              page: "$attributes.landingPage",
              loadTime: "$metrics.pageLoadTime"
            }
          }
        }
      },

      // Calculer statistiques avancées
      {
        $addFields: {
          metrics: {
            count: "$count",
            uniqueUsers: { $size: "$uniqueUsers" },
            uniqueSessions: { $size: "$uniqueSessions" },

            avgPageLoadTime: { $round: ["$avgPageLoadTime", 2] },
            avgTimeOnPage: { $round: ["$avgTimeOnPage", 2] },
            avgScrollDepth: { $round: ["$avgScrollDepth", 4] },

            // Percentiles
            p50PageLoadTime: {
              $arrayElemAt: [
                { $sortArray: { input: "$pageLoadTimes", sortBy: 1 } },
                { $floor: { $multiply: [{ $size: "$pageLoadTimes" }, 0.5] } }
              ]
            },

            p95PageLoadTime: {
              $arrayElemAt: [
                { $sortArray: { input: "$pageLoadTimes", sortBy: 1 } },
                { $floor: { $multiply: [{ $size: "$pageLoadTimes" }, 0.95] } }
              ]
            },

            p99PageLoadTime: {
              $arrayElemAt: [
                { $sortArray: { input: "$pageLoadTimes", sortBy: 1 } },
                { $floor: { $multiply: [{ $size: "$pageLoadTimes" }, 0.99] } }
              ]
            }
          },

          // Top pages
          breakdown: {
            byPage: {
              $slice: [
                {
                  $sortArray: {
                    input: {
                      $map: {
                        input: {
                          $setUnion: [
                            { $map: { input: "$pageViews", in: "$$this.page" } }
                          ]
                        },
                        as: "page",
                        in: {
                          page: "$$page",
                          count: {
                            $size: {
                              $filter: {
                                input: "$pageViews",
                                cond: { $eq: ["$$this.page", "$$page"] }
                              }
                            }
                          },
                          avgLoadTime: {
                            $avg: {
                              $map: {
                                input: {
                                  $filter: {
                                    input: "$pageViews",
                                    cond: { $eq: ["$$this.page", "$$page"] }
                                  }
                                },
                                in: "$$this.loadTime"
                              }
                            }
                          }
                        }
                      }
                    },
                    sortBy: { count: -1 }
                  }
                },
                10  // Top 10
              ]
            }
          }
        }
      },

      // Format final
      {
        $project: {
          _id: 0,
          metricType: "$_id.eventType",
          granularity: "minute",
          timestamp: "$_id.minute",

          dimensions: {
            eventType: "$_id.eventType",
            country: "$_id.country",
            deviceType: "$_id.deviceType",
            browser: "$_id.browser"
          },

          metrics: 1,
          breakdown: 1,

          computedAt: new Date(),
          version: 1
        }
      },

      // Merge dans collection de métriques
      {
        $merge: {
          into: "metrics_minutely",
          on: ["metricType", "timestamp", "dimensions"],
          whenMatched: "replace",
          whenNotMatched: "insert"
        }
      }
    ];

    await db.collection('events').aggregate(pipeline).toArray();

    console.log(`Materialized metrics for ${oneMinuteAgo.toISOString()}`);
  }

  async materializeHourlyMetrics() {
    // Similaire mais agrège depuis metrics_minutely
    const now = new Date();
    const oneHourAgo = new Date(now - 3600000);
    const twoHoursAgo = new Date(now - 7200000);

    const pipeline = [
      {
        $match: {
          granularity: "minute",
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
                date: "$timestamp",
                unit: "hour"
              }
            },
            metricType: "$metricType",
            country: "$dimensions.country",
            deviceType: "$dimensions.deviceType"
          },

          // Agréger métriques minutely
          count: { $sum: "$metrics.count" },
          uniqueUsers: { $sum: "$metrics.uniqueUsers" },
          uniqueSessions: { $sum: "$metrics.uniqueSessions" },

          avgPageLoadTime: { $avg: "$metrics.avgPageLoadTime" },
          p95PageLoadTime: { $avg: "$metrics.p95PageLoadTime" }
        }
      },

      {
        $project: {
          _id: 0,
          metricType: "$_id.metricType",
          granularity: "hour",
          timestamp: "$_id.hour",

          dimensions: {
            country: "$_id.country",
            deviceType: "$_id.deviceType"
          },

          metrics: {
            count: "$count",
            uniqueUsers: "$uniqueUsers",
            uniqueSessions: "$uniqueSessions",
            avgPageLoadTime: { $round: ["$avgPageLoadTime", 2] },
            p95PageLoadTime: { $round: ["$p95PageLoadTime", 2] }
          },

          computedAt: new Date()
        }
      },

      {
        $merge: {
          into: "metrics_hourly",
          on: ["metricType", "timestamp", "dimensions"],
          whenMatched: "replace",
          whenNotMatched: "insert"
        }
      }
    ];

    await db.collection('metrics_minutely')
      .aggregate(pipeline)
      .toArray();

    console.log(`Materialized hourly metrics for ${oneHourAgo.toISOString()}`);
  }
}

// Scheduler pour exécution périodique
const materializationJob = new MetricsMaterializationJob();

// Chaque minute
setInterval(() => {
  materializationJob.materializeMinuteMetrics();
}, 60000);

// Chaque heure
setInterval(() => {
  materializationJob.materializeHourlyMetrics();
}, 3600000);
```

## Change Streams pour temps réel

### 1. Streaming de métriques live

```javascript
class RealTimeMetricsStreamer {
  constructor(db, io) {
    this.db = db;
    this.io = io;  // Socket.io instance
    this.counters = new Map();
  }

  async start() {
    // Watch sur collection events
    const changeStream = this.db.collection('events').watch([
      {
        $match: {
          operationType: 'insert',
          'fullDocument.metadata.eventType': {
            $in: ['page_view', 'click', 'conversion']
          }
        }
      }
    ]);

    changeStream.on('change', async (change) => {
      const event = change.fullDocument;

      // Incrémenter compteurs en mémoire
      await this.updateCounters(event);

      // Broadcast aux clients connectés
      this.broadcastMetrics();
    });

    // Broadcast périodique même sans événements
    setInterval(() => {
      this.broadcastMetrics();
    }, 1000);

    console.log('Real-time metrics streamer started');
  }

  async updateCounters(event) {
    const eventType = event.metadata.eventType;

    if (!this.counters.has(eventType)) {
      this.counters.set(eventType, {
        count: 0,
        uniqueUsers: new Set(),
        uniqueSessions: new Set()
      });
    }

    const counter = this.counters.get(eventType);
    counter.count++;
    counter.uniqueUsers.add(event.metadata.dimensions.userId.toString());
    counter.uniqueSessions.add(event.metadata.dimensions.sessionId);
  }

  broadcastMetrics() {
    const metrics = {};

    for (const [eventType, counter] of this.counters.entries()) {
      metrics[eventType] = {
        count: counter.count,
        uniqueUsers: counter.uniqueUsers.size,
        uniqueSessions: counter.uniqueSessions.size,
        timestamp: new Date()
      };
    }

    // Broadcast via WebSocket
    this.io.emit('metrics:update', metrics);

    // Reset compteurs toutes les minutes
    if (Date.now() % 60000 < 1000) {
      this.counters.clear();
    }
  }

  // API pour récupérer état actuel
  getCurrentMetrics() {
    const metrics = {};

    for (const [eventType, counter] of this.counters.entries()) {
      metrics[eventType] = {
        count: counter.count,
        uniqueUsers: counter.uniqueUsers.size,
        uniqueSessions: counter.uniqueSessions.size
      };
    }

    return metrics;
  }
}

// Utilisation avec Socket.io
const io = require('socket.io')(server);
const streamer = new RealTimeMetricsStreamer(db, io);
streamer.start();

// Client side (React/Vue)
/*
const socket = io('http://localhost:3000');

socket.on('metrics:update', (metrics) => {
  console.log('Real-time metrics:', metrics);
  // Update UI
  updateDashboard(metrics);
});
*/
```

### 2. Alerting en temps réel

```javascript
class RealTimeAlertingEngine {
  constructor(db) {
    this.db = db;
    this.alertRules = new Map();
    this.alertState = new Map();
  }

  async loadAlertRules() {
    const rules = await this.db.collection('alert_rules')
      .find({ enabled: true })
      .toArray();

    for (const rule of rules) {
      this.alertRules.set(rule._id.toString(), rule);
    }

    console.log(`Loaded ${rules.length} alert rules`);
  }

  async start() {
    await this.loadAlertRules();

    // Watch sur métriques matérialisées
    const changeStream = this.db.collection('metrics_minutely').watch([
      {
        $match: {
          operationType: { $in: ['insert', 'update'] }
        }
      }
    ]);

    changeStream.on('change', async (change) => {
      const metric = change.fullDocument;
      await this.evaluateAlerts(metric);
    });

    console.log('Real-time alerting engine started');
  }

  async evaluateAlerts(metric) {
    for (const [ruleId, rule] of this.alertRules.entries()) {
      // Vérifier si rule s'applique à cette métrique
      if (!this.ruleApplies(rule, metric)) {
        continue;
      }

      // Évaluer condition
      const triggered = this.evaluateCondition(rule, metric);

      if (triggered) {
        await this.triggerAlert(rule, metric);
      } else {
        await this.resolveAlert(rule, metric);
      }
    }
  }

  ruleApplies(rule, metric) {
    // Vérifier si métrique correspond aux critères de la règle
    if (rule.metricType !== metric.metricType) {
      return false;
    }

    if (rule.dimensions) {
      for (const [key, value] of Object.entries(rule.dimensions)) {
        if (metric.dimensions[key] !== value) {
          return false;
        }
      }
    }

    return true;
  }

  evaluateCondition(rule, metric) {
    const { field, operator, threshold } = rule.condition;

    // Extraire valeur de la métrique
    const value = this.getMetricValue(metric, field);

    if (value === null) return false;

    // Évaluer condition
    switch (operator) {
      case 'greater_than':
        return value > threshold;
      case 'less_than':
        return value < threshold;
      case 'equal':
        return value === threshold;
      case 'percent_change':
        return this.evaluatePercentChange(metric, field, threshold);
      default:
        return false;
    }
  }

  getMetricValue(metric, field) {
    const parts = field.split('.');
    let value = metric;

    for (const part of parts) {
      if (value && typeof value === 'object') {
        value = value[part];
      } else {
        return null;
      }
    }

    return value;
  }

  async evaluatePercentChange(metric, field, threshold) {
    // Comparer avec métrique précédente
    const previousMetric = await this.db.collection('metrics_minutely')
      .findOne({
        metricType: metric.metricType,
        timestamp: new Date(metric.timestamp - 60000),  // -1 minute
        dimensions: metric.dimensions
      });

    if (!previousMetric) return false;

    const currentValue = this.getMetricValue(metric, field);
    const previousValue = this.getMetricValue(previousMetric, field);

    if (!currentValue || !previousValue) return false;

    const percentChange = ((currentValue - previousValue) / previousValue) * 100;

    return Math.abs(percentChange) > threshold;
  }

  async triggerAlert(rule, metric) {
    const alertKey = `${rule._id}_${metric.timestamp}`;

    // Vérifier si alerte déjà déclenchée (throttling)
    if (this.alertState.has(alertKey)) {
      return;
    }

    // Créer alerte
    const alert = {
      ruleId: rule._id,
      ruleName: rule.name,
      severity: rule.severity,

      metric: {
        type: metric.metricType,
        timestamp: metric.timestamp,
        dimensions: metric.dimensions,
        value: this.getMetricValue(metric, rule.condition.field)
      },

      condition: rule.condition,

      status: 'active',
      triggeredAt: new Date(),

      notifications: []
    };

    // Sauvegarder alerte
    const result = await this.db.collection('alerts').insertOne(alert);

    // Envoyer notifications
    await this.sendNotifications(rule, alert);

    // Marquer comme déclenchée
    this.alertState.set(alertKey, {
      triggeredAt: new Date(),
      alertId: result.insertedId
    });

    console.log(`Alert triggered: ${rule.name}`);
  }

  async resolveAlert(rule, metric) {
    const alertKey = `${rule._id}_${metric.timestamp}`;

    if (!this.alertState.has(alertKey)) {
      return;
    }

    const state = this.alertState.get(alertKey);

    // Mettre à jour alerte comme résolue
    await this.db.collection('alerts').updateOne(
      { _id: state.alertId },
      {
        $set: {
          status: 'resolved',
          resolvedAt: new Date(),
          autoResolved: true
        }
      }
    );

    this.alertState.delete(alertKey);

    console.log(`Alert auto-resolved: ${rule.name}`);
  }

  async sendNotifications(rule, alert) {
    for (const action of rule.actions) {
      try {
        switch (action.type) {
          case 'email':
            await this.sendEmail(action, alert);
            break;
          case 'webhook':
            await this.sendWebhook(action, alert);
            break;
          case 'slack':
            await this.sendSlack(action, alert);
            break;
        }

        // Logger notification
        await this.db.collection('alerts').updateOne(
          { _id: alert._id },
          {
            $push: {
              notifications: {
                type: action.type,
                sentAt: new Date(),
                status: 'sent'
              }
            }
          }
        );
      } catch (error) {
        console.error(`Failed to send ${action.type} notification:`, error);
      }
    }
  }

  async sendEmail(action, alert) {
    // Implémentation email
    console.log(`Sending email to ${action.recipients}`);
  }

  async sendWebhook(action, alert) {
    const response = await fetch(action.url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(alert)
    });

    if (!response.ok) {
      throw new Error(`Webhook failed: ${response.statusText}`);
    }
  }

  async sendSlack(action, alert) {
    // Implémentation Slack webhook
    console.log(`Sending Slack message to ${action.channel}`);
  }
}
```

## API pour dashboards

### 1. Service d'analytics

```javascript
class AnalyticsAPIService {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
  }

  async getMetrics(query) {
    const {
      metricType,
      granularity = 'minute',
      timeRange,
      dimensions = {},
      aggregation = 'sum'
    } = query;

    // Construire cache key
    const cacheKey = this.buildCacheKey(query);

    // Essayer cache
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // Déterminer collection selon granularité
    const collection = this.getCollectionForGranularity(granularity);

    // Construire pipeline
    const pipeline = this.buildMetricsPipeline(query);

    // Exécuter query
    const results = await this.db.collection(collection)
      .aggregate(pipeline)
      .toArray();

    // Formater résultats
    const formatted = this.formatResults(results, query);

    // Cacher résultats
    await this.redis.setex(
      cacheKey,
      this.getCacheTTL(granularity),
      JSON.stringify(formatted)
    );

    return formatted;
  }

  buildMetricsPipeline(query) {
    const {
      metricType,
      timeRange,
      dimensions,
      groupBy,
      orderBy = 'timestamp',
      limit
    } = query;

    const pipeline = [];

    // Match stage
    const matchStage = {
      metricType,
      timestamp: {
        $gte: new Date(timeRange.start),
        $lte: new Date(timeRange.end)
      }
    };

    // Ajouter filtres de dimensions
    for (const [key, value] of Object.entries(dimensions)) {
      matchStage[`dimensions.${key}`] = value;
    }

    pipeline.push({ $match: matchStage });

    // Group by si demandé
    if (groupBy && groupBy.length > 0) {
      const groupStage = {
        _id: {}
      };

      for (const field of groupBy) {
        groupStage._id[field] = `$dimensions.${field}`;
      }

      // Agréger métriques
      groupStage.count = { $sum: "$metrics.count" };
      groupStage.uniqueUsers = { $sum: "$metrics.uniqueUsers" };
      groupStage.avgMetric = { $avg: "$metrics.avgPageLoadTime" };

      pipeline.push({ $group: groupStage });

      // Reformater
      pipeline.push({
        $project: {
          _id: 0,
          dimensions: "$_id",
          metrics: {
            count: "$count",
            uniqueUsers: "$uniqueUsers",
            avgMetric: { $round: ["$avgMetric", 2] }
          }
        }
      });
    } else {
      // Projection simple
      pipeline.push({
        $project: {
          _id: 0,
          timestamp: 1,
          dimensions: 1,
          metrics: 1
        }
      });
    }

    // Sort
    pipeline.push({
      $sort: { [orderBy]: orderBy === 'timestamp' ? 1 : -1 }
    });

    // Limit
    if (limit) {
      pipeline.push({ $limit: limit });
    }

    return pipeline;
  }

  getCollectionForGranularity(granularity) {
    const collections = {
      'minute': 'metrics_minutely',
      'hour': 'metrics_hourly',
      'day': 'metrics_daily',
      'month': 'metrics_monthly'
    };

    return collections[granularity] || 'metrics_minutely';
  }

  getCacheTTL(granularity) {
    const ttls = {
      'minute': 60,      // 1 minute
      'hour': 300,       // 5 minutes
      'day': 1800,       // 30 minutes
      'month': 3600      // 1 heure
    };

    return ttls[granularity] || 60;
  }

  buildCacheKey(query) {
    return `analytics:${JSON.stringify(query)}`;
  }

  formatResults(results, query) {
    // Formater selon type de visualisation attendu
    return {
      data: results,
      metadata: {
        count: results.length,
        query,
        generatedAt: new Date()
      }
    };
  }

  async getTopN(metricType, dimension, timeRange, n = 10) {
    const collection = 'metrics_minutely';

    const pipeline = [
      {
        $match: {
          metricType,
          timestamp: {
            $gte: new Date(timeRange.start),
            $lte: new Date(timeRange.end)
          }
        }
      },

      {
        $group: {
          _id: `$dimensions.${dimension}`,
          totalCount: { $sum: "$metrics.count" },
          totalUsers: { $sum: "$metrics.uniqueUsers" }
        }
      },

      {
        $sort: { totalCount: -1 }
      },

      {
        $limit: n
      },

      {
        $project: {
          _id: 0,
          [dimension]: "$_id",
          count: "$totalCount",
          users: "$totalUsers"
        }
      }
    ];

    return this.db.collection(collection)
      .aggregate(pipeline)
      .toArray();
  }

  async getTimeSeries(metricType, field, timeRange, granularity = 'minute') {
    const collection = this.getCollectionForGranularity(granularity);

    const pipeline = [
      {
        $match: {
          metricType,
          timestamp: {
            $gte: new Date(timeRange.start),
            $lte: new Date(timeRange.end)
          }
        }
      },

      {
        $project: {
          _id: 0,
          timestamp: 1,
          value: `$metrics.${field}`
        }
      },

      {
        $sort: { timestamp: 1 }
      }
    ];

    return this.db.collection(collection)
      .aggregate(pipeline)
      .toArray();
  }
}

// Routes Express
app.get('/api/analytics/metrics', async (req, res) => {
  const analyticsService = new AnalyticsAPIService(db, redis);

  const query = {
    metricType: req.query.metricType,
    granularity: req.query.granularity,
    timeRange: {
      start: req.query.startDate,
      end: req.query.endDate
    },
    dimensions: JSON.parse(req.query.dimensions || '{}'),
    groupBy: req.query.groupBy?.split(',') || []
  };

  try {
    const results = await analyticsService.getMetrics(query);
    res.json(results);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/api/analytics/top/:dimension', async (req, res) => {
  const analyticsService = new AnalyticsAPIService(db, redis);

  const results = await analyticsService.getTopN(
    req.query.metricType,
    req.params.dimension,
    {
      start: req.query.startDate,
      end: req.query.endDate
    },
    parseInt(req.query.n || 10)
  );

  res.json(results);
});
```

## Performance et optimisation

### 1. Stratégies d'indexation

```javascript
// Index pour queries temps réel fréquentes

// 1. Time series avec métadonnées
db.events.createIndex({
  "metadata.eventType": 1,
  timestamp: -1
});

// 2. Dimensions multiples
db.events.createIndex({
  "metadata.eventType": 1,
  "metadata.dimensions.country": 1,
  "metadata.dimensions.deviceType": 1,
  timestamp: -1
});

// 3. Métriques matérialisées
db.metrics_minutely.createIndex({
  metricType: 1,
  timestamp: -1,
  "dimensions.country": 1
});

// 4. Index partiel pour données chaudes
db.metrics_minutely.createIndex(
  {
    metricType: 1,
    timestamp: -1
  },
  {
    partialFilterExpression: {
      timestamp: { $gte: new Date(Date.now() - 86400000) }
    }
  }
);

// 5. Index couvrant pour queries fréquentes
db.metrics_minutely.createIndex({
  metricType: 1,
  "dimensions.country": 1,
  timestamp: -1,
  "metrics.count": 1,
  "metrics.uniqueUsers": 1
});
```

### 2. Configuration MongoDB pour analytics

```javascript
// Configuration optimale
const mongoConfig = {
  // WiredTiger cache
  storage: {
    wiredTiger: {
      engineConfig: {
        cacheSizeGB: 64,  // 50-60% RAM disponible
        journalCompressor: "snappy"
      },
      collectionConfig: {
        blockCompressor: "zstd"  // Meilleure compression
      }
    }
  },

  // Oplog surdimensionné pour Change Streams
  replication: {
    oplogSizeMB: 50000  // 50 GB
  },

  // Network optimization
  net: {
    maxIncomingConnections: 5000,
    compression: {
      compressors: ["snappy", "zstd"]
    }
  }
};

// Read Preference pour analytics
const readPreference = {
  // Queries temps réel: primaryPreferred
  realtime: 'primaryPreferred',

  // Queries analytics lourdes: secondary
  analytics: 'secondary',

  // Dashboards: secondaryPreferred
  dashboards: 'secondaryPreferred'
};
```

### 3. Métriques de performance

```javascript
const analyticsMetrics = {
  // Ingestion
  'ingestion.events_per_second': {
    description: 'Events ingested per second',
    target: 100000,
    alert_threshold: 50000
  },

  'ingestion.lag': {
    description: 'Lag between event time and ingestion',
    target: 1000,  // ms
    alert_threshold: 5000
  },

  // Queries
  'query.dashboard_load_time.p95': {
    description: 'Dashboard load time p95',
    target: 2000,  // ms
    alert_threshold: 5000
  },

  'query.realtime_metric.p95': {
    description: 'Real-time metric query p95',
    target: 100,  // ms
    alert_threshold: 500
  },

  // Matérialisation
  'materialization.lag': {
    description: 'Lag for metrics materialization',
    target: 60000,  // 1 minute
    alert_threshold: 300000  // 5 minutes
  },

  // Cache
  'cache.hit_rate': {
    description: 'Cache hit rate for analytics',
    target: 0.90,
    alert_threshold: 0.70
  }
};
```

## Checklist de déploiement

### ✅ Architecture

- [ ] Event ingestion configuré (Kafka/RabbitMQ)
- [ ] Stream processing déployé (Flink/Spark)
- [ ] Time Series Collections créées
- [ ] Matérialisation jobs schedulés
- [ ] Change Streams configurés
- [ ] Cache Redis déployé

### ✅ Modélisation

- [ ] Événements bruts avec Time Series
- [ ] Hiérarchie de métriques (minute/hour/day/month)
- [ ] Dimensions et breakdown définis
- [ ] TTL configurés par granularité
- [ ] Index optimisés pour queries fréquentes

### ✅ Matérialisation

- [ ] Jobs minutely automatisés
- [ ] Jobs hourly automatisés
- [ ] Jobs daily automatisés
- [ ] $merge pipelines optimisés
- [ ] Monitoring de lag

### ✅ API et Dashboards

- [ ] API REST pour métriques
- [ ] WebSocket pour temps réel
- [ ] Dashboard configuration
- [ ] Cache multi-niveaux
- [ ] Rate limiting

### ✅ Alerting

- [ ] Règles d'alerte configurées
- [ ] Change Streams monitoring
- [ ] Notifications (email, Slack, webhook)
- [ ] Throttling anti-spam
- [ ] Dashboard d'alertes

### ✅ Performance

- [ ] Sharding si > 1TB données/jour
- [ ] Read replicas pour analytics
- [ ] Index couvrants
- [ ] Compression activée
- [ ] Monitoring latence

### ✅ Opérations

- [ ] Backup automatique
- [ ] Retention policies
- [ ] Monitoring infrastructure
- [ ] Alertes opérationnelles
- [ ] Runbooks

## Conclusion

MongoDB est particulièrement adapté à l'analyse temps réel grâce à :

**✅ Forces démontrées :**
- Change Streams pour réactivité instantanée
- Agrégation Pipeline puissante et flexible
- $merge pour matérialisation de vues
- Time Series Collections optimisées
- Indexes performants pour queries complexes
- Sharding pour scalabilité horizontale
- Schéma flexible pour dimensions variables

**⚠️ Considérations importantes :**
- Matérialisation essentielle pour performance dashboards
- Cache multi-niveaux requis pour latence < 100ms
- Stream processing externe pour agrégations complexes
- Monitoring de lag critique
- Rétention intelligente pour maîtriser coûts

**🎯 Patterns essentiels analytics :**
1. **Time Series Collections** pour événements bruts
2. **Matérialisation hiérarchique** (minute → hour → day)
3. **Change Streams** pour réactivité temps réel
4. **$merge** pour vues pré-calculées
5. **Cache Redis** pour métriques chaudes
6. **Index couvrants** pour queries fréquentes

Cette architecture supporte des systèmes d'analytics allant de milliers à milliards d'événements par jour, avec dashboards temps réel et alerting intelligent.

---

**Références :**
- MongoDB Time Series Documentation
- "Streaming Systems" - Tyler Akidau et al.
- Apache Flink Documentation
- Grafana Real-Time Analytics
- "Designing Data-Intensive Applications" - Martin Kleppmann

⏭️ [Gestion des logs](/20-cas-usage-architectures/07-gestion-logs.md)
