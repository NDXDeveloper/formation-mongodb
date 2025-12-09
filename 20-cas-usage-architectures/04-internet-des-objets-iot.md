🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 20.4 Internet des Objets (IoT)

## Introduction

L'Internet des Objets (IoT) génère des volumes massifs de données temporelles nécessitant une infrastructure capable de gérer :

- **Volume extrême** : Millions d'événements par seconde
- **Vélocité élevée** : Ingestion continue sans interruption
- **Variété de données** : Multiples types de capteurs et formats
- **Données temporelles** : Séries chronologiques avec agrégations
- **Latence critique** : Alertes en temps réel (< 1 seconde)
- **Rétention intelligente** : Données récentes détaillées, anciennes agrégées
- **Edge computing** : Traitement distribué près des sources
- **Scalabilité massive** : De dizaines à millions d'appareils

MongoDB excelle dans ce contexte grâce à :
- **Time Series Collections** : Optimisées pour données temporelles
- **Schéma flexible** : Différents types de capteurs sans migration
- **Agrégations puissantes** : Analytics en temps réel
- **Sharding automatique** : Scalabilité horizontale transparente
- **TTL Index** : Rétention automatique des données
- **Change Streams** : Réactivité temps réel pour alertes

## Architecture de référence

### Stack IoT moderne

```
┌─────────────────────────────────────────────────────┐
│              IoT Devices (Edge)                     │
│   Sensors • Actuators • Gateways • Controllers      │
└────────────────────┬────────────────────────────────┘
                     │
                     │ MQTT / CoAP / HTTP
                     │
        ┌────────────▼────────────┐
        │    Message Broker       │
        │  (MQTT Broker, Kafka)   │
        └────────────┬────────────┘
                     │
     ┌───────────────┼───────────────┐
     │               │               │
┌────▼─────┐   ┌────▼─────┐   ┌──────▼──────┐
│ Stream   │   │ Rules    │   │  Device     │
│Processor │   │ Engine   │   │  Registry   │
│(Flink)   │   │(Drools)  │   │             │
└────┬─────┘   └────┬─────┘   └──────┬──────┘
     │              │                │
     │         ┌────▼─────┐          │
     │         │  Redis   │          │
     │         │ (Cache)  │          │
     │         └──────────┘          │
     │                               │
     └────────────┬──────────────────┘
                  │
     ┌────────────▼────────────┐
     │  MongoDB Time Series    │
     │    Sharded Cluster      │
     │  ┌──────────────────┐   │
     │  │  Shard 1 (Hot)   │   │
     │  │  Recent data     │   │
     │  └──────────────────┘   │
     │  ┌──────────────────┐   │
     │  │  Shard 2 (Warm)  │   │
     │  │  Aggregated data │   │
     │  └──────────────────┘   │
     │  ┌──────────────────┐   │
     │  │  Shard 3 (Cold)  │   │
     │  │  Historical data │   │
     │  └──────────────────┘   │
     └─────────────────────────┘
                  │
     ┌────────────┼────────────┐
     │            │            │
┌────▼────┐  ┌────▼─────┐  ┌───▼──────┐
│Analytics│  │ Real-time│  │  Alerts  │
│Dashboard│  │   API    │  │  Service │
└─────────┘  └──────────┘  └──────────┘
```

### Composants architecturaux

#### 1. Edge Layer (Devices & Gateways)

**Types d'appareils IoT :**
- **Capteurs** : Température, humidité, pression, mouvement
- **Actuateurs** : Moteurs, valves, relais
- **Gateways** : Agrégation locale et pré-traitement
- **Edge Computing Nodes** : Traitement distribué

**Protocoles de communication :**
- **MQTT** : Léger, pub/sub, idéal pour IoT
- **CoAP** : REST-like pour appareils contraints
- **HTTP/HTTPS** : Pour appareils puissants
- **LoRaWAN** : Longue portée, faible consommation

#### 2. Message Broker

**Options :**
- **MQTT Broker** (Mosquitto, EMQX) : Protocole natif IoT
- **Apache Kafka** : Haute performance, persistance
- **RabbitMQ** : Flexibilité, multi-protocoles
- **AWS IoT Core** : Managed cloud solution

**Justification Kafka pour grande échelle :**
- Throughput élevé (millions de messages/s)
- Persistance pour replay
- Partitionnement pour parallélisme
- Écosystème riche (Kafka Streams, Connect)

#### 3. Stream Processing

**Apache Flink / Kafka Streams :**
- Agrégations temps réel (moyennes, compteurs)
- Détection d'anomalies
- Enrichissement de données
- Fenêtres temporelles (tumbling, sliding)

#### 4. MongoDB Time Series Collections

**Pourquoi Time Series Collections ?**
- Compression automatique (10x vs collections normales)
- Index optimisés pour queries temporelles
- Performance en écriture (bulk inserts)
- Agrégations optimisées
- Rétention automatique avec TTL

## Modélisation des données

### 1. Time Series Collections (MongoDB 5.0+)

#### Configuration de base

```javascript
// Création d'une Time Series Collection
db.createCollection("sensor_readings", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"  // seconds, minutes, hours
  },
  expireAfterSeconds: 2592000  // 30 jours
});

// Structure recommandée
{
  timestamp: ISODate("2024-12-09T14:30:00.000Z"),

  // Métadonnées (indexées automatiquement)
  metadata: {
    deviceId: "sensor_temp_001",
    deviceType: "temperature_sensor",
    location: {
      building: "Building_A",
      floor: 3,
      room: "302",
      coordinates: {
        type: "Point",
        coordinates: [2.3522, 48.8566]
      }
    },
    tags: ["critical", "hvac"],
    firmware: "v2.1.3"
  },

  // Mesures (données temporelles)
  temperature: 22.5,
  humidity: 45.2,
  pressure: 1013.25,

  // Métadonnées additionnelles
  unit: "celsius",
  accuracy: 0.1,
  batteryLevel: 87,
  signalStrength: -62,  // dBm

  // Qualité de la mesure
  quality: {
    valid: true,
    errorCode: null,
    calibrated: true,
    lastCalibration: ISODate("2024-11-01T00:00:00Z")
  }
}
```

#### Choix de granularité

```javascript
// Granularité selon fréquence de mesures

// "seconds" : < 1000 mesures/heure par device
db.createCollection("high_frequency_sensors", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  }
});

// "minutes" : 1000-100000 mesures/heure par device
db.createCollection("medium_frequency_sensors", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "minutes"
  }
});

// "hours" : > 100000 mesures/heure par device
db.createCollection("low_frequency_sensors", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "hours"
  }
});

// Impact sur performance:
// - Granularité appropriée = compression optimale
// - Granularité incorrecte = perte de performance
```

### 2. Device Registry

```javascript
// Collection: devices
{
  _id: "sensor_temp_001",

  // Informations du device
  deviceInfo: {
    type: "temperature_sensor",
    model: "TempPro-2000",
    manufacturer: "IoTSensors Inc.",
    serialNumber: "SN-2024-001234",
    firmwareVersion: "v2.1.3",
    hardwareVersion: "v1.0"
  },

  // Configuration
  config: {
    samplingInterval: 60,  // secondes
    transmissionInterval: 300,  // 5 minutes

    // Seuils d'alerte
    thresholds: {
      temperature: {
        min: 15,
        max: 30,
        critical_min: 10,
        critical_max: 35
      },
      humidity: {
        min: 30,
        max: 70
      }
    },

    // Mode de fonctionnement
    mode: "continuous",  // continuous, on_demand, scheduled
    powerSaving: true
  },

  // Localisation
  location: {
    building: "Building_A",
    floor: 3,
    room: "302",
    zone: "HVAC_Zone_A",
    coordinates: {
      type: "Point",
      coordinates: [2.3522, 48.8566]
    },
    installationDate: ISODate("2024-01-15T10:00:00Z")
  },

  // État actuel
  status: {
    state: "active",  // active, inactive, maintenance, error
    online: true,
    lastSeen: ISODate("2024-12-09T14:30:00Z"),
    lastMeasurement: ISODate("2024-12-09T14:30:00Z"),

    // Santé du device
    health: {
      batteryLevel: 87,
      signalStrength: -62,
      errorCount: 0,
      uptime: 2592000,  // secondes
      dataQuality: 0.99  // 99% de données valides
    }
  },

  // Statistiques (pré-calculées)
  stats: {
    totalReadings: 259200,  // 30 jours × 86400s / 10s
    lastHourReadings: 360,
    averageTemperature24h: 22.3,
    alertsTriggered: 5,
    lastMaintenanceDate: ISODate("2024-11-01T00:00:00Z")
  },

  // Métadonnées
  tags: ["critical", "hvac", "monitoring"],
  owner: {
    department: "Facilities",
    contact: "facilities@example.com"
  },

  // Audit
  createdAt: ISODate("2024-01-15T10:00:00Z"),
  updatedAt: ISODate("2024-12-09T14:30:00Z"),
  createdBy: ObjectId("..."),

  // Maintenance
  maintenance: {
    schedule: "monthly",
    lastMaintenance: ISODate("2024-11-01T00:00:00Z"),
    nextMaintenance: ISODate("2025-01-01T00:00:00Z"),
    maintenanceHistory: [
      {
        date: ISODate("2024-11-01T00:00:00Z"),
        type: "calibration",
        notes: "Sensor recalibrated",
        performedBy: "John Technician"
      }
    ]
  }
}

// Index pour devices
db.devices.createIndex({ "status.state": 1, "status.online": 1 });
db.devices.createIndex({ "location.building": 1, "location.floor": 1 });
db.devices.createIndex({ "location.coordinates": "2dsphere" });
db.devices.createIndex({ "deviceInfo.type": 1 });
db.devices.createIndex({ tags: 1 });
db.devices.createIndex({ "status.lastSeen": 1 });
```

### 3. Alertes et événements

```javascript
// Collection: alerts
{
  _id: ObjectId("..."),

  // Identification
  alertId: "alert_20241209_143000_001",
  type: "threshold_exceeded",  // threshold, anomaly, offline, error
  severity: "warning",  // info, warning, critical, emergency

  // Device concerné
  deviceId: "sensor_temp_001",
  deviceType: "temperature_sensor",
  location: {
    building: "Building_A",
    floor: 3,
    room: "302"
  },

  // Détails de l'alerte
  trigger: {
    metric: "temperature",
    value: 32.5,
    threshold: 30,
    condition: "greater_than",
    duration: 300  // Durée de dépassement en secondes
  },

  // Contexte
  context: {
    previousValue: 29.8,
    averageLast24h: 22.3,
    deviation: 10.2,  // Écart par rapport à la moyenne

    // Mesures corrélées
    relatedMetrics: {
      humidity: 38.5,
      pressure: 1012.8
    }
  },

  // État de l'alerte
  status: "active",  // active, acknowledged, resolved, ignored

  // Actions
  actions: [
    {
      type: "notification",
      channel: "email",
      recipient: "facilities@example.com",
      sentAt: ISODate("2024-12-09T14:30:05Z"),
      status: "sent"
    },
    {
      type: "webhook",
      url: "https://api.example.com/alerts",
      sentAt: ISODate("2024-12-09T14:30:05Z"),
      status: "success"
    }
  ],

  // Résolution
  resolution: {
    acknowledgedBy: ObjectId("user_id"),
    acknowledgedAt: ISODate("2024-12-09T14:35:00Z"),
    resolvedBy: ObjectId("user_id"),
    resolvedAt: ISODate("2024-12-09T15:00:00Z"),
    notes: "HVAC system adjusted, temperature normalized",
    actions_taken: ["adjusted_hvac_settings"]
  },

  // Timestamps
  triggeredAt: ISODate("2024-12-09T14:30:00Z"),
  lastUpdatedAt: ISODate("2024-12-09T15:00:00Z"),

  // TTL pour cleanup automatique
  expiresAt: ISODate("2025-01-08T14:30:00Z")  // 30 jours
}

// Index pour alerts
db.alerts.createIndex({ deviceId: 1, triggeredAt: -1 });
db.alerts.createIndex({ status: 1, severity: 1, triggeredAt: -1 });
db.alerts.createIndex({ "location.building": 1, severity: 1 });
db.alerts.createIndex({ type: 1, triggeredAt: -1 });
db.alerts.createIndex(
  { expiresAt: 1 },
  { expireAfterSeconds: 0 }
);
```

## Ingestion haute performance

### 1. Batch Insert optimisé

```javascript
class IoTDataIngestionService {
  constructor(db, options = {}) {
    this.db = db;
    this.batchSize = options.batchSize || 1000;
    this.flushInterval = options.flushInterval || 1000;  // ms
    this.buffer = [];
    this.flushTimer = null;
  }

  async ingest(reading) {
    this.buffer.push(reading);

    if (this.buffer.length >= this.batchSize) {
      await this.flush();
    } else if (!this.flushTimer) {
      this.flushTimer = setTimeout(() => this.flush(), this.flushInterval);
    }
  }

  async flush() {
    if (this.buffer.length === 0) return;

    clearTimeout(this.flushTimer);
    this.flushTimer = null;

    const batch = this.buffer.splice(0);

    try {
      // Bulk insert avec ordered: false pour parallélisation
      await this.db.collection('sensor_readings').insertMany(
        batch,
        {
          ordered: false,
          writeConcern: { w: 1, j: false }  // Performance over durability
        }
      );

      console.log(`Flushed ${batch.length} readings`);
    } catch (error) {
      console.error('Batch insert failed:', error);

      // Retry logic ou dead letter queue
      await this.handleFailedBatch(batch, error);
    }
  }

  async handleFailedBatch(batch, error) {
    // Option 1: Retry avec exponential backoff
    // Option 2: Stocker dans dead letter queue
    // Option 3: Logger pour investigation manuelle

    await this.db.collection('failed_readings').insertMany(
      batch.map(reading => ({
        ...reading,
        error: error.message,
        failedAt: new Date()
      })),
      { ordered: false }
    );
  }
}

// Utilisation
const ingestionService = new IoTDataIngestionService(db, {
  batchSize: 1000,
  flushInterval: 1000
});

// Consumer Kafka
kafkaConsumer.on('message', async (message) => {
  const reading = JSON.parse(message.value);
  await ingestionService.ingest(reading);
});
```

### 2. Write Concern optimisé

```javascript
// Configuration selon criticité des données

// Données critiques (équipements médicaux, sécurité)
const criticalWriteConcern = {
  writeConcern: {
    w: "majority",
    j: true,
    wtimeout: 5000
  }
};

// Données importantes (monitoring industriel)
const standardWriteConcern = {
  writeConcern: {
    w: 1,
    j: true,
    wtimeout: 1000
  }
};

// Données volumineuses non-critiques (télémétrie générale)
const bulkWriteConcern = {
  writeConcern: {
    w: 1,
    j: false  // Performance over durability
  }
};

// Utilisation
async function insertReading(reading, criticality = 'standard') {
  const concerns = {
    critical: criticalWriteConcern,
    standard: standardWriteConcern,
    bulk: bulkWriteConcern
  };

  return db.collection('sensor_readings').insertOne(
    reading,
    concerns[criticality]
  );
}
```

### 3. Sharding pour scalabilité

```javascript
// Stratégie de sharding pour Time Series

// Option 1: Shard key sur metaField + timeField
// Avantage: Distribution par device et temps
// Inconvénient: Queries cross-shard pour agrégations globales

sh.shardCollection(
  "iot.sensor_readings",
  { "metadata.deviceId": "hashed", timestamp: 1 }
);

// Option 2: Shard key sur location + timeField
// Avantage: Queries localisées efficaces
// Inconvénient: Hotspots si concentration géographique

sh.shardCollection(
  "iot.sensor_readings",
  { "metadata.location.building": 1, timestamp: 1 }
);

// Option 3: Ranged sharding par temps (avec pre-split)
// Avantage: Queries temporelles optimales
// Inconvénient: Hotspot sur shard le plus récent

// Pre-split pour éviter hotspot initial
for (let i = 0; i < 12; i++) {
  const date = new Date(2024, i, 1);
  sh.splitAt(
    "iot.sensor_readings",
    { timestamp: date }
  );
}

// Zone sharding pour archivage
sh.addShardToZone("shard01", "hot");   // Données récentes (SSD)
sh.addShardToZone("shard02", "warm");  // Données agrégées
sh.addShardToZone("shard03", "cold");  // Données archivées (HDD)

const now = new Date();
const weekAgo = new Date(now - 7 * 24 * 3600000);
const monthAgo = new Date(now - 30 * 24 * 3600000);

// Hot data: dernière semaine
sh.updateZoneKeyRange(
  "iot.sensor_readings",
  { timestamp: weekAgo },
  { timestamp: new Date(2099, 12, 31) },
  "hot"
);

// Warm data: dernières 4 semaines
sh.updateZoneKeyRange(
  "iot.sensor_readings",
  { timestamp: monthAgo },
  { timestamp: weekAgo },
  "warm"
);

// Cold data: > 1 mois
sh.updateZoneKeyRange(
  "iot.sensor_readings",
  { timestamp: new Date(2020, 1, 1) },
  { timestamp: monthAgo },
  "cold"
);
```

## Agrégations et analytics

### 1. Fenêtres temporelles

```javascript
// Agrégation par fenêtres de 5 minutes
async function getAveragesByTimeWindow(deviceId, startDate, endDate) {
  const pipeline = [
    {
      $match: {
        "metadata.deviceId": deviceId,
        timestamp: {
          $gte: startDate,
          $lte: endDate
        }
      }
    },

    // Grouper par fenêtres de 5 minutes
    {
      $group: {
        _id: {
          $dateTrunc: {
            date: "$timestamp",
            unit: "minute",
            binSize: 5
          }
        },

        avgTemperature: { $avg: "$temperature" },
        minTemperature: { $min: "$temperature" },
        maxTemperature: { $max: "$temperature" },

        avgHumidity: { $avg: "$humidity" },

        count: { $sum: 1 },

        // Calcul de déviation standard
        temperatureStdDev: { $stdDevPop: "$temperature" }
      }
    },

    { $sort: { _id: 1 } },

    // Renommer pour clarté
    {
      $project: {
        _id: 0,
        window: "$_id",
        temperature: {
          avg: { $round: ["$avgTemperature", 2] },
          min: { $round: ["$minTemperature", 2] },
          max: { $round: ["$maxTemperature", 2] },
          stdDev: { $round: ["$temperatureStdDev", 2] }
        },
        humidity: {
          avg: { $round: ["$avgHumidity", 2] }
        },
        sampleCount: "$count"
      }
    }
  ];

  return db.collection('sensor_readings')
    .aggregate(pipeline)
    .toArray();
}
```

### 2. Détection d'anomalies

```javascript
// Détection basée sur écart à la moyenne mobile
async function detectAnomalies(deviceId, hours = 24) {
  const cutoffDate = new Date(Date.now() - hours * 3600000);

  const pipeline = [
    {
      $match: {
        "metadata.deviceId": deviceId,
        timestamp: { $gte: cutoffDate }
      }
    },

    { $sort: { timestamp: 1 } },

    // Calcul de statistiques globales
    {
      $group: {
        _id: null,
        avgTemp: { $avg: "$temperature" },
        stdDevTemp: { $stdDevPop: "$temperature" },
        readings: { $push: "$$ROOT" }
      }
    },

    // Identifier anomalies (> 3 écarts-types)
    {
      $project: {
        avgTemp: 1,
        stdDevTemp: 1,
        anomalies: {
          $filter: {
            input: "$readings",
            as: "reading",
            cond: {
              $gt: [
                {
                  $abs: {
                    $subtract: [
                      "$$reading.temperature",
                      "$avgTemp"
                    ]
                  }
                },
                { $multiply: ["$stdDevTemp", 3] }
              ]
            }
          }
        }
      }
    },

    { $unwind: "$anomalies" },

    {
      $project: {
        _id: "$anomalies._id",
        timestamp: "$anomalies.timestamp",
        temperature: "$anomalies.temperature",
        deviation: {
          $abs: {
            $subtract: ["$anomalies.temperature", "$avgTemp"]
          }
        },
        expectedRange: {
          min: { $subtract: ["$avgTemp", { $multiply: ["$stdDevTemp", 3] }] },
          max: { $add: ["$avgTemp", { $multiply: ["$stdDevTemp", 3] }] }
        }
      }
    }
  ];

  return db.collection('sensor_readings')
    .aggregate(pipeline)
    .toArray();
}
```

### 3. Corrélation multi-capteurs

```javascript
// Corréler température et humidité pour détection de patterns
async function correlateSensors(buildingId, startDate, endDate) {
  const pipeline = [
    {
      $match: {
        "metadata.location.building": buildingId,
        timestamp: { $gte: startDate, $lte: endDate }
      }
    },

    // Grouper par fenêtres horaires et floor
    {
      $group: {
        _id: {
          hour: {
            $dateTrunc: {
              date: "$timestamp",
              unit: "hour"
            }
          },
          floor: "$metadata.location.floor"
        },

        avgTemp: { $avg: "$temperature" },
        avgHumidity: { $avg: "$humidity" },

        devices: { $addToSet: "$metadata.deviceId" },
        sampleCount: { $sum: 1 }
      }
    },

    // Calcul du comfort index
    {
      $addFields: {
        comfortIndex: {
          $subtract: [
            100,
            {
              $add: [
                {
                  $abs: {
                    $subtract: ["$avgTemp", 22]  // Température idéale
                  }
                },
                {
                  $divide: [
                    {
                      $abs: {
                        $subtract: ["$avgHumidity", 50]  // Humidité idéale
                      }
                    },
                    2
                  ]
                }
              ]
            }
          ]
        }
      }
    },

    { $sort: { "_id.hour": 1, "_id.floor": 1 } },

    {
      $project: {
        _id: 0,
        timestamp: "$_id.hour",
        floor: "$_id.floor",
        temperature: { $round: ["$avgTemp", 2] },
        humidity: { $round: ["$avgHumidity", 2] },
        comfortIndex: { $round: ["$comfortIndex", 2] },
        devicesCount: { $size: "$devices" },
        sampleCount: 1
      }
    }
  ];

  return db.collection('sensor_readings')
    .aggregate(pipeline)
    .toArray();
}
```

### 4. Downsampling et agrégation hiérarchique

```javascript
// Pattern : Données granulaires → Agrégations pré-calculées
// Permet queries rapides sur historique

// Job périodique pour pré-agrégation
class DataAggregationJob {
  async runHourlyAggregation() {
    const oneHourAgo = new Date(Date.now() - 3600000);
    const twoHoursAgo = new Date(Date.now() - 7200000);

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
            deviceId: "$metadata.deviceId",
            hour: {
              $dateTrunc: {
                date: "$timestamp",
                unit: "hour"
              }
            }
          },

          // Statistiques complètes
          temperature: {
            avg: { $avg: "$temperature" },
            min: { $min: "$temperature" },
            max: { $max: "$temperature" },
            stdDev: { $stdDevPop: "$temperature" }
          },

          humidity: {
            avg: { $avg: "$humidity" },
            min: { $min: "$humidity" },
            max: { $max: "$humidity" }
          },

          pressure: {
            avg: { $avg: "$pressure" }
          },

          sampleCount: { $sum: 1 },

          // Première et dernière mesure
          firstReading: { $first: "$$ROOT" },
          lastReading: { $last: "$$ROOT" }
        }
      },

      {
        $project: {
          _id: 0,
          deviceId: "$_id.deviceId",
          timestamp: "$_id.hour",
          granularity: "hourly",

          temperature: 1,
          humidity: 1,
          pressure: 1,
          sampleCount: 1,

          metadata: "$firstReading.metadata",

          createdAt: new Date()
        }
      },

      // Écrire dans collection d'agrégats
      {
        $merge: {
          into: "sensor_readings_hourly",
          whenMatched: "replace",
          whenNotMatched: "insert"
        }
      }
    ];

    await db.collection('sensor_readings')
      .aggregate(pipeline)
      .toArray();

    console.log('Hourly aggregation completed');
  }

  async runDailyAggregation() {
    // Similaire mais agrège depuis sensor_readings_hourly
    const oneDayAgo = new Date(Date.now() - 86400000);
    const twoDaysAgo = new Date(Date.now() - 172800000);

    const pipeline = [
      {
        $match: {
          timestamp: {
            $gte: twoDaysAgo,
            $lt: oneDayAgo
          }
        }
      },

      {
        $group: {
          _id: {
            deviceId: "$deviceId",
            day: {
              $dateTrunc: {
                date: "$timestamp",
                unit: "day"
              }
            }
          },

          temperature: {
            avg: { $avg: "$temperature.avg" },
            min: { $min: "$temperature.min" },
            max: { $max: "$temperature.max" }
          },

          humidity: {
            avg: { $avg: "$humidity.avg" },
            min: { $min: "$humidity.min" },
            max: { $max: "$humidity.max" }
          },

          sampleCount: { $sum: "$sampleCount" }
        }
      },

      {
        $merge: {
          into: "sensor_readings_daily",
          whenMatched: "replace",
          whenNotMatched: "insert"
        }
      }
    ];

    await db.collection('sensor_readings_hourly')
      .aggregate(pipeline)
      .toArray();

    console.log('Daily aggregation completed');
  }
}

// Collections pour différentes granularités:
// - sensor_readings: Données brutes (TTL: 7 jours)
// - sensor_readings_hourly: Agrégats horaires (TTL: 90 jours)
// - sensor_readings_daily: Agrégats quotidiens (TTL: 2 ans)
// - sensor_readings_monthly: Agrégats mensuels (permanent)
```

## Alerting en temps réel

### 1. Change Streams pour détection

```javascript
class RealtimeAlertingService {
  constructor(db) {
    this.db = db;
    this.alertRules = new Map();
  }

  async start() {
    // Watch sur Time Series Collection
    const changeStream = this.db.collection('sensor_readings').watch([
      {
        $match: {
          operationType: 'insert'
        }
      }
    ]);

    changeStream.on('change', async (change) => {
      const reading = change.fullDocument;
      await this.evaluateAlertRules(reading);
    });

    console.log('Real-time alerting service started');
  }

  async evaluateAlertRules(reading) {
    // Récupérer device config pour seuils
    const device = await this.db.collection('devices')
      .findOne({ _id: reading.metadata.deviceId });

    if (!device) return;

    const thresholds = device.config.thresholds;

    // Vérifier seuils de température
    if (reading.temperature) {
      if (reading.temperature > thresholds.temperature.critical_max) {
        await this.triggerAlert({
          deviceId: device._id,
          severity: 'critical',
          type: 'threshold_exceeded',
          metric: 'temperature',
          value: reading.temperature,
          threshold: thresholds.temperature.critical_max,
          reading
        });
      } else if (reading.temperature > thresholds.temperature.max) {
        await this.triggerAlert({
          deviceId: device._id,
          severity: 'warning',
          type: 'threshold_exceeded',
          metric: 'temperature',
          value: reading.temperature,
          threshold: thresholds.temperature.max,
          reading
        });
      }
    }

    // Vérifier anomalies (écart soudain)
    await this.checkForAnomalies(reading, device);
  }

  async checkForAnomalies(reading, device) {
    // Récupérer dernières mesures pour comparaison
    const recentReadings = await this.db.collection('sensor_readings')
      .find({
        "metadata.deviceId": device._id,
        timestamp: {
          $gte: new Date(Date.now() - 3600000)  // dernière heure
        }
      })
      .sort({ timestamp: -1 })
      .limit(10)
      .toArray();

    if (recentReadings.length < 3) return;

    const avgTemp = recentReadings.reduce((sum, r) => sum + r.temperature, 0)
      / recentReadings.length;

    const deviation = Math.abs(reading.temperature - avgTemp);

    // Alerte si écart > 5°C soudainement
    if (deviation > 5) {
      await this.triggerAlert({
        deviceId: device._id,
        severity: 'warning',
        type: 'anomaly',
        metric: 'temperature',
        value: reading.temperature,
        expectedValue: avgTemp,
        deviation,
        reading
      });
    }
  }

  async triggerAlert(alertData) {
    const alert = {
      alertId: `alert_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`,
      type: alertData.type,
      severity: alertData.severity,
      deviceId: alertData.deviceId,

      trigger: {
        metric: alertData.metric,
        value: alertData.value,
        threshold: alertData.threshold,
        condition: 'greater_than'
      },

      context: {
        previousValue: alertData.expectedValue,
        deviation: alertData.deviation
      },

      status: 'active',
      triggeredAt: new Date(),

      actions: []
    };

    // Insérer alerte
    await this.db.collection('alerts').insertOne(alert);

    // Déclencher notifications
    await this.sendNotifications(alert);

    console.log(`Alert triggered: ${alert.alertId}`);
  }

  async sendNotifications(alert) {
    // Email
    if (alert.severity === 'critical') {
      // Envoyer email urgent
      await this.sendEmail({
        to: 'ops@example.com',
        subject: `CRITICAL ALERT: ${alert.type}`,
        body: `Device ${alert.deviceId}: ${alert.trigger.metric} = ${alert.trigger.value}`
      });
    }

    // Webhook
    await this.sendWebhook({
      url: 'https://api.example.com/alerts',
      payload: alert
    });

    // Push notification
    // SMS pour alertes critiques
    // etc.
  }
}
```

### 2. Règles d'alerting complexes

```javascript
// Collection: alert_rules
{
  _id: ObjectId("..."),
  name: "HVAC Temperature Monitoring",
  description: "Alert when temperature exceeds threshold for sustained period",

  // Condition
  condition: {
    metric: "temperature",
    operator: "greater_than",
    threshold: 30,
    duration: 300,  // Doit persister 5 minutes
    aggregation: "avg"  // avg, min, max, count
  },

  // Scope
  scope: {
    deviceTypes: ["temperature_sensor"],
    locations: [
      { building: "Building_A", floors: [3, 4] }
    ],
    tags: ["critical", "hvac"]
  },

  // Severity
  severity: "warning",

  // Actions
  actions: [
    {
      type: "email",
      recipients: ["facilities@example.com"],
      template: "temperature_alert"
    },
    {
      type: "webhook",
      url: "https://api.example.com/alerts",
      method: "POST"
    }
  ],

  // Throttling (éviter spam)
  throttle: {
    enabled: true,
    cooldownPeriod: 1800  // 30 minutes entre alertes similaires
  },

  // Schedule
  schedule: {
    enabled: true,
    timezone: "Europe/Paris",
    activeHours: {
      start: "08:00",
      end: "18:00"
    },
    activeDays: ["monday", "tuesday", "wednesday", "thursday", "friday"]
  },

  // Status
  enabled: true,
  createdAt: ISODate("..."),
  updatedAt: ISODate("...")
}
```

## Edge Computing et MongoDB

### 1. MongoDB on Edge Devices

```javascript
// Configuration MongoDB Realm pour edge
// Synchronisation bi-directionnelle edge ↔ cloud

// Sur le gateway edge
const app = new Realm.App({ id: "iot-app-xxxxx" });

// Authentification
const credentials = Realm.Credentials.apiKey(edgeApiKey);
const user = await app.logIn(credentials);

// Ouvrir Realm local avec sync
const realm = await Realm.open({
  schema: [SensorReadingSchema],
  sync: {
    user: user,
    partitionValue: "building_A_floor_3",

    // Configuration sync
    newRealmFileBehavior: {
      type: 'downloadBeforeOpen',
      timeOut: 30000
    },

    existingRealmFileBehavior: {
      type: 'openImmediately'
    }
  }
});

// Écriture locale (sync automatique vers cloud)
realm.write(() => {
  realm.create('SensorReading', {
    _id: new ObjectId(),
    timestamp: new Date(),
    metadata: {
      deviceId: 'sensor_temp_001',
      location: { building: 'Building_A', floor: 3 }
    },
    temperature: 22.5,
    humidity: 45.2
  });
});

// Le reading sera automatiquement sync vers MongoDB Atlas
// Résolution de conflits automatique (last-write-wins)
```

### 2. Edge Analytics (pré-traitement)

```javascript
// Traitement sur gateway edge avant envoi cloud
class EdgeAnalytics {
  constructor() {
    this.buffer = new Map();  // deviceId -> readings
    this.anomalyThreshold = 3;  // écarts-types
  }

  async processReading(reading) {
    const deviceId = reading.metadata.deviceId;

    // Maintenir buffer glissant (100 dernières mesures)
    if (!this.buffer.has(deviceId)) {
      this.buffer.set(deviceId, []);
    }

    const deviceBuffer = this.buffer.get(deviceId);
    deviceBuffer.push(reading);

    if (deviceBuffer.length > 100) {
      deviceBuffer.shift();
    }

    // Calculs locaux
    const stats = this.calculateStats(deviceBuffer);

    // Décision: envoyer au cloud?
    const shouldSendToCloud = this.decideTransmission(reading, stats);

    if (shouldSendToCloud) {
      await this.sendToCloud(reading, stats);
    } else {
      // Stocker localement seulement
      await this.storeLocally(reading);
    }

    // Vérifier anomalies localement (alerting rapide)
    await this.checkAnomalies(reading, stats);
  }

  calculateStats(readings) {
    const temps = readings.map(r => r.temperature);
    const avg = temps.reduce((a, b) => a + b) / temps.length;

    const variance = temps.reduce((sum, t) =>
      sum + Math.pow(t - avg, 2), 0) / temps.length;

    const stdDev = Math.sqrt(variance);

    return {
      avg,
      stdDev,
      min: Math.min(...temps),
      max: Math.max(...temps)
    };
  }

  decideTransmission(reading, stats) {
    // Stratégies pour réduire bande passante:

    // 1. Envoyer si écart significatif
    const deviation = Math.abs(reading.temperature - stats.avg);
    if (deviation > stats.stdDev * 2) {
      return true;
    }

    // 2. Envoyer périodiquement (heartbeat)
    const timeSinceLastSent = Date.now() - this.lastSentTime;
    if (timeSinceLastSent > 300000) {  // 5 minutes
      this.lastSentTime = Date.now();
      return true;
    }

    // 3. Sinon, agréger localement
    return false;
  }

  async sendToCloud(reading, stats) {
    // Enrichir avec statistiques edge
    const enrichedReading = {
      ...reading,
      edgeStats: stats,
      processedAt: new Date()
    };

    // Envoyer via MQTT/HTTP
    await mqttClient.publish(
      `iot/readings/${reading.metadata.deviceId}`,
      JSON.stringify(enrichedReading)
    );
  }
}
```

## Performance et optimisation

### 1. Métriques de performance IoT

```javascript
const iotPerformanceMetrics = {
  // Ingestion
  'ingestion.throughput': {
    description: 'Messages ingérés par seconde',
    target: 100000,  // 100K messages/s
    alert_threshold: 50000
  },

  'ingestion.latency.p99': {
    description: 'Latence p99 d\'ingestion',
    target: 100,  // ms
    alert_threshold: 500
  },

  'ingestion.batch_size': {
    description: 'Taille moyenne des batches',
    target: 1000,
    alert_threshold: 100
  },

  // Storage
  'storage.compression_ratio': {
    description: 'Ratio de compression Time Series',
    target: 10,  // 10x compression
    alert_threshold: 5
  },

  'storage.growth_rate': {
    description: 'Croissance du stockage (GB/jour)',
    monitor: true
  },

  // Queries
  'query.aggregation_time.p95': {
    description: 'Temps d\'agrégation p95',
    target: 200,  // ms
    alert_threshold: 1000
  },

  // Devices
  'devices.online_ratio': {
    description: 'Pourcentage de devices online',
    target: 0.98,  // 98%
    alert_threshold: 0.90
  },

  'devices.data_quality': {
    description: 'Qualité moyenne des données',
    target: 0.99,  // 99%
    alert_threshold: 0.95
  },

  // Alerting
  'alerts.detection_latency.p99': {
    description: 'Latence détection alerte p99',
    target: 1000,  // 1s
    alert_threshold: 5000
  }
};
```

### 2. Optimisations spécifiques IoT

```javascript
// Configuration MongoDB optimisée pour IoT
const mongoConfig = {
  // WiredTiger cache
  storage: {
    wiredTiger: {
      engineConfig: {
        cacheSizeGB: 32,  // 50-60% RAM
        journalCompressor: "snappy",
        directoryForIndexes: true
      },

      collectionConfig: {
        blockCompressor: "zstd"  // Meilleure compression
      },

      indexConfig: {
        prefixCompression: true
      }
    }
  },

  // Opérations
  operationProfiling: {
    mode: 1,  // Log slow operations
    slowms: 100
  },

  // Network
  net: {
    maxIncomingConnections: 10000,
    compression: {
      compressors: ["snappy", "zstd"]
    }
  },

  // Replication
  replication: {
    oplogSizeMB: 50000  // 50 GB pour IoT haute fréquence
  }
};

// Index optimisés pour Time Series
db.sensor_readings.createIndex(
  { "metadata.deviceId": 1, timestamp: -1 },
  {
    name: "device_time_idx",
    background: true,
    // Partial index: seulement dernières 24h en mémoire
    partialFilterExpression: {
      timestamp: {
        $gte: new Date(Date.now() - 86400000)
      }
    }
  }
);
```

## Checklist de déploiement IoT

### ✅ Architecture

- [ ] Message broker configuré (MQTT/Kafka)
- [ ] Stream processing déployé (Flink/Kafka Streams)
- [ ] Time Series Collections créées
- [ ] Sharding configuré selon charge
- [ ] Edge gateways provisionnés
- [ ] Device registry initialisé

### ✅ Ingestion

- [ ] Batch inserts implémentés (1000+ par batch)
- [ ] Write Concern adapté à la criticité
- [ ] Dead letter queue pour échecs
- [ ] Monitoring de throughput
- [ ] Rate limiting configuré

### ✅ Rétention

- [ ] TTL configuré sur données brutes (7-30 jours)
- [ ] Agrégations horaires automatisées
- [ ] Agrégations quotidiennes automatisées
- [ ] Archive vers cold storage (S3/Glacier)
- [ ] Stratégie de downsampling définie

### ✅ Alerting

- [ ] Change Streams pour temps réel
- [ ] Règles d'alerting configurées
- [ ] Throttling anti-spam activé
- [ ] Canaux de notification (email, SMS, webhook)
- [ ] Escalation automatique pour alertes critiques

### ✅ Performance

- [ ] Index optimisés pour queries temporelles
- [ ] Compression activée (zstd)
- [ ] Oplog surdimensionné (50 GB+)
- [ ] Cache WiredTiger optimisé
- [ ] Métriques de performance suivies

### ✅ Sécurité

- [ ] Authentification MQTT (TLS client certificates)
- [ ] Chiffrement en transit activé
- [ ] Device credentials rotés régulièrement
- [ ] Access control par device/location
- [ ] Audit logging activé

### ✅ Opérations

- [ ] Backup automatique quotidien
- [ ] Monitoring devices online/offline
- [ ] Alertes sur qualité de données
- [ ] Runbooks pour incidents
- [ ] Plan de disaster recovery

## Conclusion

MongoDB est particulièrement adapté aux workloads IoT grâce à :

**✅ Forces démontrées :**
- Time Series Collections avec compression 10x
- Ingestion haute performance (millions de messages/s)
- Schéma flexible pour types de capteurs hétérogènes
- Agrégations puissantes pour analytics temps réel
- TTL automatique pour rétention intelligente
- Sharding transparent pour scalabilité illimitée
- Change Streams pour alerting réactif

**⚠️ Considérations importantes :**
- Write Concern doit balancer durabilité et performance
- Stratégie de rétention essentielle (coûts storage)
- Agrégations pré-calculées pour historique
- Monitoring de qualité de données critique
- Edge processing réduit bande passante et latence

**🎯 Patterns essentiels IoT :**
1. **Time Series Collections** pour données temporelles
2. **Batch Inserts** pour ingestion haute performance
3. **Downsampling hiérarchique** pour historique
4. **Zone Sharding** pour hot/warm/cold data
5. **Change Streams** pour alerting temps réel
6. **Edge Analytics** pour pré-traitement local

Cette architecture supporte des déploiements IoT de quelques centaines à plusieurs millions d'appareils, avec ingestion de milliards de points de données par jour.

---

**Références :**
- MongoDB Time Series Documentation
- "Designing Data-Intensive Applications" - Martin Kleppmann
- AWS IoT Core Best Practices
- "Building the Internet of Things" - Maciej Kranz
- Apache Kafka for IoT Use Cases

⏭️ [Gaming et leaderboards](/20-cas-usage-architectures/05-gaming-leaderboards.md)
