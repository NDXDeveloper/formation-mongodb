🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 16.4 Time Series Collections

## Introduction

Les **Time Series Collections** sont des collections MongoDB spécialement optimisées pour stocker des séquences de mesures au fil du temps. Introduites dans MongoDB 5.0, elles offrent une compression automatique (jusqu'à 90%), des performances d'écriture améliorées, et des requêtes optimisées pour les analyses temporelles.

Ces collections sont idéales pour les données horodatées telles que les métriques IoT, les logs applicatifs, les données financières, ou toute donnée avec un timestamp comme dimension principale.

---

## Architecture et concepts fondamentaux

### Structure interne : Bucketing

MongoDB organise automatiquement les documents des time series en **buckets** (seaux) selon le timestamp.

```
┌────────────────────────────────────────────────────────────┐
│              Time Series Collection                        │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Bucket 1: 2024-12-08 14:00:00 - 14:59:59          │    │
│  │  ┌──────────────────────────────────────────┐      │    │
│  │  │ metaField: { sensor: "temp-001" }        │      │    │
│  │  │ data: [                                  │      │    │
│  │  │   { t: 14:00:01, v: 22.5 },              │      │    │
│  │  │   { t: 14:00:11, v: 22.7 },              │      │    │
│  │  │   { t: 14:00:21, v: 22.6 },              │      │    │
│  │  │   ... (jusqu'à 1000 mesures)             │      │    │
│  │  │ ]                                        │      │    │
│  │  └──────────────────────────────────────────┘      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │  Bucket 2: 2024-12-08 15:00:00 - 15:59:59          │    │
│  │  ┌──────────────────────────────────────────┐      │    │
│  │  │ metaField: { sensor: "temp-001" }        │      │    │
│  │  │ data: [ ... ]                            │      │    │
│  │  └──────────────────────────────────────────┘      │    │
│  └────────────────────────────────────────────────────┘    │
│                                                            │
│  Compression automatique: 90-95% économie d'espace         │
└────────────────────────────────────────────────────────────┘
```

### Champs requis

| Champ | Description | Obligatoire |
|-------|-------------|-------------|
| **timeField** | Champ contenant le timestamp (Date) | ✅ Oui |
| **metaField** | Métadonnées identifiant la source (ex: sensor ID) | ⚠️ Recommandé |
| **granularity** | Granularité temporelle (seconds, minutes, hours) | ⚠️ Recommandé |

### Granularités disponibles

```javascript
// Granularités supportées
const granularities = {
  'seconds': 60 * 1000,        // Données chaque seconde
  'minutes': 60 * 60 * 1000,   // Données chaque minute
  'hours': 24 * 60 * 60 * 1000 // Données chaque heure
};

// MongoDB crée des buckets selon la granularité:
// - seconds: buckets d'1 heure
// - minutes: buckets d'1 jour
// - hours: buckets de 30 jours
```

---

## Création et configuration

### Syntaxe de création

```javascript
// Création basique
await db.createCollection('weather', {
  timeseries: {
    timeField: 'timestamp',
    metaField: 'metadata',
    granularity: 'minutes'
  }
});

// Avec options avancées
await db.createCollection('stock_prices', {
  timeseries: {
    timeField: 'time',
    metaField: 'ticker',
    granularity: 'seconds'
  },
  expireAfterSeconds: 2592000  // TTL: 30 jours
});

// Avec validation de schéma
await db.createCollection('iot_sensors', {
  timeseries: {
    timeField: 'timestamp',
    metaField: 'sensor',
    granularity: 'seconds'
  },
  validator: {
    $jsonSchema: {
      bsonType: 'object',
      required: ['timestamp', 'sensor', 'value'],
      properties: {
        timestamp: { bsonType: 'date' },
        sensor: { bsonType: 'object' },
        value: { bsonType: 'number' }
      }
    }
  }
});
```

### Collections système créées

```javascript
// MongoDB crée deux collections internes:
// - system.buckets.<collection>  (données compressées)
// - <collection>                 (vue sur les buckets)

// Vérifier les buckets
const buckets = await db.collection('system.buckets.weather')
  .find()
  .limit(1)
  .toArray();

console.log('Bucket structure:', buckets[0]);
/*
{
  _id: ObjectId("..."),
  control: {
    version: 1,
    min: { timestamp: ISODate("..."), temp: 15.2 },
    max: { timestamp: ISODate("..."), temp: 28.7 }
  },
  meta: { location: "Paris", sensor: "temp-001" },
  data: {
    timestamp: {
      0: ISODate("..."),
      1: ISODate("..."),
      // ... colonnes compressées
    },
    temp: {
      0: 22.5,
      1: 22.7,
      // ...
    }
  }
}
*/
```

---

## Opérations de base

### Insertions

```javascript
// Insertion simple
await db.collection('weather').insertOne({
  timestamp: new Date(),
  metadata: {
    location: 'Paris',
    sensor: 'temp-001'
  },
  temperature: 22.5,
  humidity: 65
});

// Insertions multiples (batch recommandé)
const measurements = [];
for (let i = 0; i < 1000; i++) {
  measurements.push({
    timestamp: new Date(Date.now() - i * 60000), // 1 mesure/minute
    metadata: {
      location: 'Paris',
      sensor: 'temp-001'
    },
    temperature: 20 + Math.random() * 10,
    humidity: 50 + Math.random() * 30
  });
}

await db.collection('weather').insertMany(measurements, {
  ordered: false  // Insertions parallèles pour performance
});

// Performance: Time series collections sont optimisées pour insertions
// en ordre chronologique
```

### Requêtes

```javascript
// Requête sur période temporelle
const recentData = await db.collection('weather').find({
  timestamp: {
    $gte: new Date('2024-12-08T00:00:00Z'),
    $lt: new Date('2024-12-09T00:00:00Z')
  },
  'metadata.location': 'Paris'
}).toArray();

// Agrégation temporelle
const hourlyAvg = await db.collection('weather').aggregate([
  {
    $match: {
      timestamp: {
        $gte: new Date('2024-12-08T00:00:00Z')
      }
    }
  },
  {
    $group: {
      _id: {
        $dateTrunc: {
          date: '$timestamp',
          unit: 'hour'
        }
      },
      avgTemp: { $avg: '$temperature' },
      avgHumidity: { $avg: '$humidity' },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
]).toArray();

// Requête avec window functions
const movingAvg = await db.collection('weather').aggregate([
  { $match: { 'metadata.sensor': 'temp-001' } },
  { $sort: { timestamp: 1 } },
  {
    $setWindowFields: {
      sortBy: { timestamp: 1 },
      output: {
        movingAvgTemp: {
          $avg: '$temperature',
          window: {
            documents: [-4, 0]  // 5 derniers documents
          }
        }
      }
    }
  }
]).toArray();
```

### Limitations

```javascript
// ❌ Pas de mises à jour individuelles
try {
  await db.collection('weather').updateOne(
    { _id: docId },
    { $set: { temperature: 25 } }
  );
} catch (error) {
  // Error: Cannot update documents in a time series collection
}

// ❌ Pas de suppressions individuelles
try {
  await db.collection('weather').deleteOne({ _id: docId });
} catch (error) {
  // Error: Cannot delete documents from a time series collection
}

// ✅ Suppression par plage temporelle (MongoDB 5.1+)
await db.collection('weather').deleteMany({
  timestamp: {
    $lt: new Date('2024-01-01T00:00:00Z')
  }
});

// ✅ TTL automatique via expireAfterSeconds
// Les documents sont supprimés automatiquement après expiration
```

---

## Cas d'usage avancés

### Cas 1 : Monitoring IoT avec capteurs multiples

```javascript
class IoTMonitoringSystem {
  constructor(db) {
    this.db = db;
    this.sensorData = null;
    this.alerts = null;
  }

  async initialize() {
    // Collection time series pour données capteurs
    try {
      await this.db.createCollection('sensor_data', {
        timeseries: {
          timeField: 'timestamp',
          metaField: 'metadata',
          granularity: 'seconds'
        },
        expireAfterSeconds: 7776000  // 90 jours
      });

      // Index pour requêtes fréquentes
      await this.db.collection('sensor_data').createIndex({
        'metadata.device_id': 1,
        'timestamp': -1
      });

      await this.db.collection('sensor_data').createIndex({
        'metadata.location': 1,
        'timestamp': -1
      });

    } catch (error) {
      if (error.code !== 48) throw error;
    }

    this.sensorData = this.db.collection('sensor_data');

    // Collection standard pour alertes
    this.alerts = this.db.collection('iot_alerts');
    await this.alerts.createIndex({ timestamp: 1, resolved: 1 });
  }

  async recordMeasurement(deviceId, measurements) {
    const document = {
      timestamp: new Date(),
      metadata: {
        device_id: deviceId,
        device_type: measurements.type || 'generic',
        location: measurements.location,
        firmware: measurements.firmware || 'unknown'
      },
      // Mesures
      temperature: measurements.temperature,
      humidity: measurements.humidity,
      pressure: measurements.pressure,
      battery: measurements.battery,
      signal_strength: measurements.signal_strength
    };

    await this.sensorData.insertOne(document);

    // Vérifier les seuils et créer des alertes
    await this.checkThresholds(deviceId, measurements);
  }

  async recordBatch(measurements) {
    const documents = measurements.map(m => ({
      timestamp: m.timestamp || new Date(),
      metadata: {
        device_id: m.device_id,
        device_type: m.type || 'generic',
        location: m.location
      },
      temperature: m.temperature,
      humidity: m.humidity,
      pressure: m.pressure,
      battery: m.battery
    }));

    // Insertion batch non ordonnée pour performance
    await this.sensorData.insertMany(documents, { ordered: false });
  }

  async checkThresholds(deviceId, measurements) {
    const alerts = [];

    // Vérifier température
    if (measurements.temperature > 35 || measurements.temperature < -10) {
      alerts.push({
        device_id: deviceId,
        type: 'temperature',
        severity: 'critical',
        value: measurements.temperature,
        threshold: measurements.temperature > 35 ? 35 : -10,
        message: `Temperature out of range: ${measurements.temperature}°C`
      });
    }

    // Vérifier batterie
    if (measurements.battery < 20) {
      alerts.push({
        device_id: deviceId,
        type: 'battery',
        severity: measurements.battery < 10 ? 'critical' : 'warning',
        value: measurements.battery,
        threshold: 20,
        message: `Low battery: ${measurements.battery}%`
      });
    }

    // Enregistrer les alertes
    for (const alert of alerts) {
      await this.alerts.insertOne({
        ...alert,
        timestamp: new Date(),
        resolved: false
      });
    }

    return alerts;
  }

  async getDeviceHistory(deviceId, options = {}) {
    const {
      since = new Date(Date.now() - 24 * 3600 * 1000),  // 24h par défaut
      until = new Date(),
      fields = ['temperature', 'humidity', 'pressure']
    } = options;

    const projection = {
      timestamp: 1,
      'metadata.device_id': 1
    };

    fields.forEach(field => {
      projection[field] = 1;
    });

    return await this.sensorData.find({
      'metadata.device_id': deviceId,
      timestamp: { $gte: since, $lte: until }
    })
      .project(projection)
      .sort({ timestamp: 1 })
      .toArray();
  }

  async getAggregatedMetrics(deviceId, interval = 'hour') {
    const intervalUnit = interval === 'hour' ? 'hour' :
                        interval === 'day' ? 'day' : 'minute';

    return await this.sensorData.aggregate([
      {
        $match: {
          'metadata.device_id': deviceId,
          timestamp: {
            $gte: new Date(Date.now() - 7 * 24 * 3600 * 1000)  // 7 jours
          }
        }
      },
      {
        $group: {
          _id: {
            $dateTrunc: {
              date: '$timestamp',
              unit: intervalUnit
            }
          },
          avgTemperature: { $avg: '$temperature' },
          minTemperature: { $min: '$temperature' },
          maxTemperature: { $max: '$temperature' },
          avgHumidity: { $avg: '$humidity' },
          avgPressure: { $avg: '$pressure' },
          avgBattery: { $avg: '$battery' },
          measurementCount: { $sum: 1 }
        }
      },
      { $sort: { _id: 1 } }
    ]).toArray();
  }

  async getLocationStatistics(location) {
    return await this.sensorData.aggregate([
      {
        $match: {
          'metadata.location': location,
          timestamp: {
            $gte: new Date(Date.now() - 24 * 3600 * 1000)  // 24h
          }
        }
      },
      {
        $group: {
          _id: '$metadata.device_id',
          deviceType: { $first: '$metadata.device_type' },
          avgTemperature: { $avg: '$temperature' },
          minTemperature: { $min: '$temperature' },
          maxTemperature: { $max: '$temperature' },
          lastSeen: { $max: '$timestamp' },
          measurementCount: { $sum: 1 }
        }
      },
      { $sort: { lastSeen: -1 } }
    ]).toArray();
  }

  async detectAnomalies(deviceId, threshold = 2) {
    // Détecter les anomalies avec écart-type
    const stats = await this.sensorData.aggregate([
      {
        $match: {
          'metadata.device_id': deviceId,
          timestamp: {
            $gte: new Date(Date.now() - 24 * 3600 * 1000)
          }
        }
      },
      {
        $group: {
          _id: null,
          avgTemp: { $avg: '$temperature' },
          stdDevTemp: { $stdDevPop: '$temperature' }
        }
      }
    ]).toArray();

    if (stats.length === 0) return [];

    const { avgTemp, stdDevTemp } = stats[0];

    // Trouver les valeurs hors norme
    return await this.sensorData.aggregate([
      {
        $match: {
          'metadata.device_id': deviceId,
          timestamp: {
            $gte: new Date(Date.now() - 24 * 3600 * 1000)
          }
        }
      },
      {
        $addFields: {
          deviation: {
            $abs: {
              $subtract: ['$temperature', avgTemp]
            }
          }
        }
      },
      {
        $match: {
          deviation: { $gt: stdDevTemp * threshold }
        }
      },
      { $sort: { timestamp: 1 } }
    ]).toArray();
  }

  async getRealtimeDashboard() {
    const lastMinute = new Date(Date.now() - 60000);

    const [activeDevices, recentAlerts, metrics] = await Promise.all([
      // Devices actifs
      this.sensorData.aggregate([
        { $match: { timestamp: { $gte: lastMinute } } },
        {
          $group: {
            _id: '$metadata.device_id',
            lastSeen: { $max: '$timestamp' },
            location: { $first: '$metadata.location' },
            currentTemp: { $last: '$temperature' },
            currentBattery: { $last: '$battery' }
          }
        }
      ]).toArray(),

      // Alertes récentes non résolues
      this.alerts.find({
        resolved: false,
        timestamp: { $gte: new Date(Date.now() - 3600000) }  // 1h
      })
        .sort({ timestamp: -1 })
        .limit(10)
        .toArray(),

      // Métriques globales
      this.sensorData.aggregate([
        { $match: { timestamp: { $gte: lastMinute } } },
        {
          $group: {
            _id: null,
            avgTemp: { $avg: '$temperature' },
            minTemp: { $min: '$temperature' },
            maxTemp: { $max: '$temperature' },
            totalMeasurements: { $sum: 1 }
          }
        }
      ]).toArray()
    ]);

    return {
      activeDevices: activeDevices.length,
      devices: activeDevices,
      alerts: recentAlerts,
      metrics: metrics[0] || {}
    };
  }

  async getCompressionStats() {
    const [dataSizeUncompressed, bucketsSize] = await Promise.all([
      // Taille estimée sans compression
      this.sensorData.stats().then(s => s.count * s.avgObjSize),

      // Taille réelle des buckets
      this.db.collection('system.buckets.sensor_data').stats().then(s => s.size)
    ]);

    return {
      uncompressedEstimate: dataSizeUncompressed,
      compressedSize: bucketsSize,
      compressionRatio: ((1 - bucketsSize / dataSizeUncompressed) * 100).toFixed(2) + '%',
      spaceSaved: dataSizeUncompressed - bucketsSize
    };
  }
}

// Utilisation
const iotSystem = new IoTMonitoringSystem(db);
await iotSystem.initialize();

// Enregistrer des mesures
await iotSystem.recordMeasurement('sensor-001', {
  type: 'temperature',
  location: 'warehouse-a',
  temperature: 23.5,
  humidity: 65,
  pressure: 1013,
  battery: 87,
  signal_strength: -65
});

// Batch insert pour haute performance
const batchData = Array.from({ length: 1000 }, (_, i) => ({
  device_id: `sensor-${String(i % 10).padStart(3, '0')}`,
  location: `zone-${Math.floor(i / 100)}`,
  temperature: 20 + Math.random() * 10,
  humidity: 50 + Math.random() * 30,
  pressure: 1000 + Math.random() * 50,
  battery: 50 + Math.random() * 50
}));

await iotSystem.recordBatch(batchData);

// Dashboard temps réel
const dashboard = await iotSystem.getRealtimeDashboard();
console.log('Dashboard:', dashboard);

// Statistiques de compression
const compressionStats = await iotSystem.getCompressionStats();
console.log('Compression:', compressionStats);
// Exemple: { compressionRatio: '93.5%', spaceSaved: 28.5 MB }
```

### Cas 2 : Données financières et trading

```javascript
class FinancialTimeSeriesManager {
  constructor(db) {
    this.db = db;
    this.tickData = null;
    this.ohlcData = null;
  }

  async initialize() {
    // Collection pour tick data (très haute fréquence)
    try {
      await this.db.createCollection('tick_data', {
        timeseries: {
          timeField: 'timestamp',
          metaField: 'symbol',
          granularity: 'seconds'
        },
        expireAfterSeconds: 2592000  // 30 jours
      });

      // Collection pour OHLC (agrégé)
      await this.db.createCollection('ohlc_data', {
        timeseries: {
          timeField: 'timestamp',
          metaField: 'symbol',
          granularity: 'minutes'
        }
      });

    } catch (error) {
      if (error.code !== 48) throw error;
    }

    this.tickData = this.db.collection('tick_data');
    this.ohlcData = this.db.collection('ohlc_data');

    // Index
    await this.tickData.createIndex({ 'symbol.ticker': 1, timestamp: -1 });
    await this.ohlcData.createIndex({ 'symbol.ticker': 1, timestamp: -1 });
  }

  async recordTick(ticker, price, volume, exchange = 'NYSE') {
    await this.tickData.insertOne({
      timestamp: new Date(),
      symbol: {
        ticker: ticker.toUpperCase(),
        exchange
      },
      price,
      volume,
      bid: price - 0.01,  // Simplifié
      ask: price + 0.01
    });
  }

  async recordTickBatch(ticks) {
    const documents = ticks.map(tick => ({
      timestamp: tick.timestamp || new Date(),
      symbol: {
        ticker: tick.ticker.toUpperCase(),
        exchange: tick.exchange || 'NYSE'
      },
      price: tick.price,
      volume: tick.volume,
      bid: tick.bid,
      ask: tick.ask
    }));

    await this.tickData.insertMany(documents, { ordered: false });
  }

  async generateOHLC(ticker, interval = 'minute') {
    const intervalUnit = interval === 'minute' ? 'minute' :
                        interval === 'hour' ? 'hour' : 'day';

    const ohlcData = await this.tickData.aggregate([
      {
        $match: {
          'symbol.ticker': ticker,
          timestamp: {
            $gte: new Date(Date.now() - 24 * 3600 * 1000)  // 24h
          }
        }
      },
      {
        $group: {
          _id: {
            $dateTrunc: {
              date: '$timestamp',
              unit: intervalUnit
            }
          },
          open: { $first: '$price' },
          high: { $max: '$price' },
          low: { $min: '$price' },
          close: { $last: '$price' },
          volume: { $sum: '$volume' },
          trades: { $sum: 1 }
        }
      },
      { $sort: { _id: 1 } }
    ]).toArray();

    // Sauvegarder dans ohlc_data
    const ohlcDocs = ohlcData.map(candle => ({
      timestamp: candle._id,
      symbol: { ticker, interval: intervalUnit },
      open: candle.open,
      high: candle.high,
      low: candle.low,
      close: candle.close,
      volume: candle.volume,
      trades: candle.trades
    }));

    if (ohlcDocs.length > 0) {
      await this.ohlcData.insertMany(ohlcDocs, { ordered: false });
    }

    return ohlcData;
  }

  async calculateTechnicalIndicators(ticker, period = 14) {
    // RSI (Relative Strength Index)
    const priceData = await this.ohlcData.find({
      'symbol.ticker': ticker,
      'symbol.interval': 'minute'
    })
      .sort({ timestamp: -1 })
      .limit(period + 1)
      .toArray();

    if (priceData.length < period + 1) {
      return null;
    }

    // Calculer les gains et pertes
    const changes = [];
    for (let i = 1; i < priceData.length; i++) {
      const change = priceData[i - 1].close - priceData[i].close;
      changes.push(change);
    }

    const gains = changes.map(c => c > 0 ? c : 0);
    const losses = changes.map(c => c < 0 ? Math.abs(c) : 0);

    const avgGain = gains.reduce((a, b) => a + b, 0) / period;
    const avgLoss = losses.reduce((a, b) => a + b, 0) / period;

    const rs = avgGain / avgLoss;
    const rsi = 100 - (100 / (1 + rs));

    // SMA (Simple Moving Average)
    const sma = priceData.slice(0, period).reduce((sum, d) => sum + d.close, 0) / period;

    return {
      ticker,
      timestamp: new Date(),
      rsi: rsi.toFixed(2),
      sma: sma.toFixed(2),
      currentPrice: priceData[0].close
    };
  }

  async detectPriceBreakout(ticker, threshold = 0.02) {
    // Détecter si prix actuel dépasse threshold% de la moyenne
    const [current, avg] = await Promise.all([
      this.tickData.find({ 'symbol.ticker': ticker })
        .sort({ timestamp: -1 })
        .limit(1)
        .toArray(),

      this.tickData.aggregate([
        {
          $match: {
            'symbol.ticker': ticker,
            timestamp: {
              $gte: new Date(Date.now() - 3600000)  // 1h
            }
          }
        },
        {
          $group: {
            _id: null,
            avgPrice: { $avg: '$price' }
          }
        }
      ]).toArray()
    ]);

    if (current.length === 0 || avg.length === 0) {
      return null;
    }

    const currentPrice = current[0].price;
    const avgPrice = avg[0].avgPrice;
    const change = (currentPrice - avgPrice) / avgPrice;

    return {
      ticker,
      currentPrice,
      avgPrice,
      change: (change * 100).toFixed(2) + '%',
      breakout: Math.abs(change) > threshold,
      direction: change > 0 ? 'up' : 'down'
    };
  }

  async getMarketSnapshot(tickers) {
    const snapshots = await Promise.all(
      tickers.map(async ticker => {
        const [latestTick, dayStats] = await Promise.all([
          this.tickData.find({ 'symbol.ticker': ticker })
            .sort({ timestamp: -1 })
            .limit(1)
            .toArray(),

          this.tickData.aggregate([
            {
              $match: {
                'symbol.ticker': ticker,
                timestamp: {
                  $gte: new Date(Date.now() - 24 * 3600 * 1000)
                }
              }
            },
            {
              $group: {
                _id: null,
                open: { $first: '$price' },
                high: { $max: '$price' },
                low: { $min: '$price' },
                close: { $last: '$price' },
                volume: { $sum: '$volume' }
              }
            }
          ]).toArray()
        ]);

        if (latestTick.length === 0) return null;

        const stats = dayStats[0] || {};
        const change = stats.open ? ((stats.close - stats.open) / stats.open * 100) : 0;

        return {
          ticker,
          price: latestTick[0].price,
          timestamp: latestTick[0].timestamp,
          dayOpen: stats.open,
          dayHigh: stats.high,
          dayLow: stats.low,
          dayClose: stats.close,
          dayVolume: stats.volume,
          dayChange: change.toFixed(2) + '%'
        };
      })
    );

    return snapshots.filter(s => s !== null);
  }

  async getVolumeProfile(ticker, bins = 20) {
    const data = await this.tickData.find({
      'symbol.ticker': ticker,
      timestamp: {
        $gte: new Date(Date.now() - 24 * 3600 * 1000)
      }
    }).toArray();

    if (data.length === 0) return [];

    const prices = data.map(d => d.price);
    const minPrice = Math.min(...prices);
    const maxPrice = Math.max(...prices);
    const binSize = (maxPrice - minPrice) / bins;

    // Créer les bins
    const profile = Array.from({ length: bins }, (_, i) => ({
      priceMin: minPrice + i * binSize,
      priceMax: minPrice + (i + 1) * binSize,
      volume: 0
    }));

    // Remplir les bins
    data.forEach(tick => {
      const binIndex = Math.min(
        Math.floor((tick.price - minPrice) / binSize),
        bins - 1
      );
      profile[binIndex].volume += tick.volume;
    });

    return profile;
  }
}

// Utilisation
const financeManager = new FinancialTimeSeriesManager(db);
await financeManager.initialize();

// Enregistrer des ticks
await financeManager.recordTick('AAPL', 178.50, 1000);
await financeManager.recordTick('GOOGL', 142.30, 500);

// Batch insert pour données en temps réel
const marketData = [
  { ticker: 'AAPL', price: 178.52, volume: 200, bid: 178.51, ask: 178.53 },
  { ticker: 'GOOGL', price: 142.32, volume: 150, bid: 142.31, ask: 142.33 },
  // ... milliers de ticks par seconde
];
await financeManager.recordTickBatch(marketData);

// Générer OHLC
const ohlc = await financeManager.generateOHLC('AAPL', 'minute');

// Indicateurs techniques
const indicators = await financeManager.calculateTechnicalIndicators('AAPL');
console.log('RSI:', indicators.rsi, 'SMA:', indicators.sma);

// Détecter breakouts
const breakout = await financeManager.detectPriceBreakout('AAPL', 0.02);
if (breakout.breakout) {
  console.log(`Breakout detected: ${breakout.ticker} ${breakout.direction}`);
}

// Market snapshot
const snapshot = await financeManager.getMarketSnapshot(['AAPL', 'GOOGL', 'MSFT']);
console.log('Market:', snapshot);
```

### Cas 3 : Logs applicatifs et APM (Application Performance Monitoring)

```javascript
class ApplicationPerformanceMonitor {
  constructor(db) {
    this.db = db;
    this.traces = null;
    this.metrics = null;
  }

  async initialize() {
    // Collection pour traces
    try {
      await this.db.createCollection('apm_traces', {
        timeseries: {
          timeField: 'timestamp',
          metaField: 'trace',
          granularity: 'seconds'
        },
        expireAfterSeconds: 604800  // 7 jours
      });

      // Collection pour métriques agrégées
      await this.db.createCollection('apm_metrics', {
        timeseries: {
          timeField: 'timestamp',
          metaField: 'service',
          granularity: 'minutes'
        }
      });

    } catch (error) {
      if (error.code !== 48) throw error;
    }

    this.traces = this.db.collection('apm_traces');
    this.metrics = this.db.collection('apm_metrics');

    // Index
    await this.traces.createIndex({
      'trace.service': 1,
      'trace.operation': 1,
      timestamp: -1
    });
  }

  async recordTrace(traceData) {
    await this.traces.insertOne({
      timestamp: traceData.timestamp || new Date(),
      trace: {
        trace_id: traceData.trace_id,
        span_id: traceData.span_id,
        parent_span_id: traceData.parent_span_id,
        service: traceData.service,
        operation: traceData.operation,
        resource: traceData.resource
      },
      duration_ms: traceData.duration_ms,
      status: traceData.status || 'ok',
      error: traceData.error || null,
      tags: traceData.tags || {},
      metadata: traceData.metadata || {}
    });

    // Créer alerte si lent
    if (traceData.duration_ms > 1000) {
      await this.createSlowQueryAlert(traceData);
    }
  }

  async createSlowQueryAlert(traceData) {
    await this.db.collection('apm_alerts').insertOne({
      timestamp: new Date(),
      type: 'slow_query',
      severity: traceData.duration_ms > 5000 ? 'critical' : 'warning',
      service: traceData.service,
      operation: traceData.operation,
      duration_ms: traceData.duration_ms,
      trace_id: traceData.trace_id
    });
  }

  async getServicePerformance(service, period = 'hour') {
    const since = period === 'hour' ?
      new Date(Date.now() - 3600000) :
      new Date(Date.now() - 24 * 3600000);

    return await this.traces.aggregate([
      {
        $match: {
          'trace.service': service,
          timestamp: { $gte: since }
        }
      },
      {
        $group: {
          _id: {
            $dateTrunc: {
              date: '$timestamp',
              unit: period === 'hour' ? 'minute' : 'hour'
            }
          },
          avgDuration: { $avg: '$duration_ms' },
          p50Duration: { $percentile: { input: '$duration_ms', p: [0.5], method: 'approximate' } },
          p95Duration: { $percentile: { input: '$duration_ms', p: [0.95], method: 'approximate' } },
          p99Duration: { $percentile: { input: '$duration_ms', p: [0.99], method: 'approximate' } },
          totalRequests: { $sum: 1 },
          errors: {
            $sum: { $cond: [{ $ne: ['$status', 'ok'] }, 1, 0] }
          }
        }
      },
      {
        $addFields: {
          errorRate: {
            $multiply: [
              { $divide: ['$errors', '$totalRequests'] },
              100
            ]
          }
        }
      },
      { $sort: { _id: 1 } }
    ]).toArray();
  }

  async getSlowestOperations(service, limit = 10) {
    return await this.traces.find({
      'trace.service': service,
      timestamp: {
        $gte: new Date(Date.now() - 3600000)  // 1h
      }
    })
      .sort({ duration_ms: -1 })
      .limit(limit)
      .toArray();
  }

  async getErrorTraces(service, limit = 20) {
    return await this.traces.find({
      'trace.service': service,
      status: { $ne: 'ok' },
      timestamp: {
        $gte: new Date(Date.now() - 3600000)
      }
    })
      .sort({ timestamp: -1 })
      .limit(limit)
      .toArray();
  }

  async aggregateMetrics() {
    // Exécuter périodiquement (ex: toutes les minutes)
    const services = await this.traces.distinct('trace.service');

    for (const service of services) {
      const stats = await this.traces.aggregate([
        {
          $match: {
            'trace.service': service,
            timestamp: {
              $gte: new Date(Date.now() - 60000)  // Dernière minute
            }
          }
        },
        {
          $group: {
            _id: null,
            avgDuration: { $avg: '$duration_ms' },
            maxDuration: { $max: '$duration_ms' },
            minDuration: { $min: '$duration_ms' },
            totalRequests: { $sum: 1 },
            errors: {
              $sum: { $cond: [{ $ne: ['$status', 'ok'] }, 1, 0] }
            }
          }
        }
      ]).toArray();

      if (stats.length > 0) {
        await this.metrics.insertOne({
          timestamp: new Date(),
          service: { name: service },
          ...stats[0]
        });
      }
    }
  }

  async getDashboard() {
    const lastMinute = new Date(Date.now() - 60000);

    const [serviceStats, recentErrors, slowQueries] = await Promise.all([
      // Stats par service
      this.traces.aggregate([
        { $match: { timestamp: { $gte: lastMinute } } },
        {
          $group: {
            _id: '$trace.service',
            requests: { $sum: 1 },
            avgDuration: { $avg: '$duration_ms' },
            errors: {
              $sum: { $cond: [{ $ne: ['$status', 'ok'] }, 1, 0] }
            }
          }
        }
      ]).toArray(),

      // Erreurs récentes
      this.traces.find({
        status: { $ne: 'ok' },
        timestamp: { $gte: lastMinute }
      })
        .sort({ timestamp: -1 })
        .limit(10)
        .toArray(),

      // Requêtes lentes
      this.traces.find({
        duration_ms: { $gt: 1000 },
        timestamp: { $gte: lastMinute }
      })
        .sort({ duration_ms: -1 })
        .limit(10)
        .toArray()
    ]);

    return {
      services: serviceStats,
      recentErrors,
      slowQueries,
      timestamp: new Date()
    };
  }
}

// Utilisation
const apm = new ApplicationPerformanceMonitor(db);
await apm.initialize();

// Enregistrer une trace
await apm.recordTrace({
  trace_id: 'abc123',
  span_id: 'span-001',
  service: 'api-gateway',
  operation: 'GET /users',
  resource: '/users',
  duration_ms: 45,
  status: 'ok',
  tags: {
    http_method: 'GET',
    http_status: 200
  }
});

// Performance d'un service
const performance = await apm.getServicePerformance('api-gateway', 'hour');
console.log('Performance:', performance);

// Dashboard APM
const dashboard = await apm.getDashboard();
console.log('APM Dashboard:', dashboard);

// Agrégation périodique des métriques
setInterval(async () => {
  await apm.aggregateMetrics();
}, 60000);  // Toutes les minutes
```

---

## Optimisations et performances

### Performances d'insertion

```javascript
// ✅ Insertion en ordre chronologique (optimal)
const measurements = [];
const startTime = Date.now();
for (let i = 0; i < 10000; i++) {
  measurements.push({
    timestamp: new Date(startTime + i * 1000),
    value: Math.random() * 100
  });
}
// Insertion rapide car suit l'ordre chronologique

// ⚠️ Insertion désordonnée (moins optimal)
const unorderedMeasurements = [];
for (let i = 0; i < 10000; i++) {
  unorderedMeasurements.push({
    timestamp: new Date(Date.now() - Math.random() * 86400000),
    value: Math.random() * 100
  });
}
// Plus lent car doit trouver les bons buckets
```

### Choix de la granularité

```javascript
// Granularité selon fréquence de données
const configs = {
  // Données chaque seconde ou moins
  highFrequency: {
    granularity: 'seconds',
    bucketSpan: '1 hour',
    exampleUseCase: 'Tick data financière, capteurs IoT haute fréquence'
  },

  // Données chaque minute
  mediumFrequency: {
    granularity: 'minutes',
    bucketSpan: '1 day',
    exampleUseCase: 'Métriques applicatives, monitoring standard'
  },

  // Données chaque heure ou moins fréquent
  lowFrequency: {
    granularity: 'hours',
    bucketSpan: '30 days',
    exampleUseCase: 'Données météo quotidiennes, analytics mensuels'
  }
};

// Choisir la granularité la plus proche de votre fréquence réelle
```

### Index secondaires

```javascript
// Index sur metaField pour requêtes fréquentes
await db.collection('sensor_data').createIndex({
  'metadata.device_id': 1,
  timestamp: -1
});

// Index composé
await db.collection('sensor_data').createIndex({
  'metadata.location': 1,
  'metadata.type': 1,
  timestamp: -1
});

// Index partiel pour données spécifiques
await db.collection('sensor_data').createIndex(
  { timestamp: -1 },
  {
    partialFilterExpression: {
      'metadata.critical': true
    }
  }
);
```

---

## Comparaison avec collections standard

### Benchmark de performance

```javascript
class TimeSeriesBenchmark {
  constructor(db) {
    this.db = db;
  }

  async setupCollections() {
    // Time series collection
    await this.db.createCollection('ts_data', {
      timeseries: {
        timeField: 'timestamp',
        metaField: 'metadata',
        granularity: 'seconds'
      }
    });

    // Collection standard
    await this.db.collection('standard_data').createIndex({
      metadata: 1,
      timestamp: -1
    });
  }

  async benchmarkInserts(count = 10000) {
    const tsData = [];
    const standardData = [];

    for (let i = 0; i < count; i++) {
      const doc = {
        timestamp: new Date(),
        metadata: { sensor: `sensor-${i % 100}` },
        value: Math.random() * 100
      };
      tsData.push(doc);
      standardData.push({ ...doc });
    }

    // Time series insert
    const tsStart = Date.now();
    await this.db.collection('ts_data').insertMany(tsData, { ordered: false });
    const tsDuration = Date.now() - tsStart;

    // Standard insert
    const stdStart = Date.now();
    await this.db.collection('standard_data').insertMany(standardData, { ordered: false });
    const stdDuration = Date.now() - stdStart;

    return {
      timeSeries: {
        duration: tsDuration,
        docsPerSec: Math.round(count / (tsDuration / 1000))
      },
      standard: {
        duration: stdDuration,
        docsPerSec: Math.round(count / (stdDuration / 1000))
      },
      improvement: `${Math.round((1 - tsDuration / stdDuration) * 100)}%`
    };
  }

  async benchmarkStorage() {
    const [tsStats, stdStats] = await Promise.all([
      this.db.collection('ts_data').stats(),
      this.db.collection('standard_data').stats()
    ]);

    return {
      timeSeries: {
        size: tsStats.size,
        storageSize: tsStats.storageSize,
        count: tsStats.count
      },
      standard: {
        size: stdStats.size,
        storageSize: stdStats.storageSize,
        count: stdStats.count
      },
      compression: `${Math.round((1 - tsStats.storageSize / stdStats.storageSize) * 100)}%`
    };
  }
}

// Utilisation
const benchmark = new TimeSeriesBenchmark(db);
await benchmark.setupCollections();

const insertResults = await benchmark.benchmarkInserts(100000);
console.log('Insert Performance:', insertResults);
// Exemple: Time series 15-20% plus rapide

const storageResults = await benchmark.benchmarkStorage();
console.log('Storage Comparison:', storageResults);
// Exemple: Time series 90-95% moins d'espace
```

### Tableau comparatif

| Critère | Time Series | Standard |
|---------|-------------|----------|
| **Compression** | 90-95% | Aucune |
| **Performance insert** | +15-20% | Baseline |
| **Requêtes temporelles** | Optimisées | Standard |
| **Update individuel** | ❌ Pas supporté | ✅ Oui |
| **Delete individuel** | ❌ Limité (plage) | ✅ Oui |
| **Stockage** | Très efficace | Standard |
| **Index secondaires** | ✅ Supportés | ✅ Supportés |

---

## Bonnes pratiques de production

### ✅ DO (À faire)

```javascript
// 1. Toujours définir metaField
await db.createCollection('sensors', {
  timeseries: {
    timeField: 'timestamp',
    metaField: 'metadata',  // ✅ Important pour bucketing efficace
    granularity: 'seconds'
  }
});

// 2. Choisir la bonne granularité
// Règle: granularité = intervalle moyen entre mesures
const avgInterval = 60; // secondes
const granularity = avgInterval <= 60 ? 'seconds' :
                   avgInterval <= 3600 ? 'minutes' : 'hours';

// 3. Utiliser batch inserts
const batch = [];
for (const measurement of measurements) {
  batch.push(measurement);
  if (batch.length >= 1000) {
    await collection.insertMany(batch, { ordered: false });
    batch.length = 0;
  }
}

// 4. Configurer TTL pour gestion automatique
await db.createCollection('temp_data', {
  timeseries: {
    timeField: 'timestamp',
    metaField: 'metadata',
    granularity: 'seconds'
  },
  expireAfterSeconds: 2592000  // 30 jours
});

// 5. Créer des index appropriés
await collection.createIndex({
  'metadata.device_id': 1,
  timestamp: -1
});
```

### ❌ DON'T (À éviter)

```javascript
// 1. Ne pas omettre metaField si données par source
// ❌ MAUVAIS
await db.createCollection('sensors', {
  timeseries: {
    timeField: 'timestamp'
    // Pas de metaField - bucketing moins efficace
  }
});

// 2. Ne pas choisir mauvaise granularité
// ❌ granularity: 'seconds' pour données horaires
// ❌ granularity: 'hours' pour données par seconde

// 3. Ne pas essayer d'update
// ❌ ERREUR
await collection.updateOne({ _id: id }, { $set: { value: 100 } });

// 4. Ne pas insérer en masse de manière désordonnée
// ⚠️ Moins performant
const randomTimestamps = generateRandomTimestamps();

// 5. Ne pas oublier la planification de la rétention
// Sans TTL, les données s'accumulent indéfiniment
```

---

## Migration vers Time Series

```javascript
class TimeSeriesMigration {
  constructor(db) {
    this.db = db;
  }

  async migrateFromStandard(sourceCollection, targetCollection, config) {
    console.log(`Starting migration from ${sourceCollection} to ${targetCollection}`);

    // Créer la time series collection
    await this.db.createCollection(targetCollection, {
      timeseries: {
        timeField: config.timeField,
        metaField: config.metaField,
        granularity: config.granularity
      }
    });

    // Compter les documents
    const totalDocs = await this.db.collection(sourceCollection).countDocuments();
    console.log(`Total documents to migrate: ${totalDocs}`);

    // Migration par batch
    const batchSize = 10000;
    let migrated = 0;

    while (migrated < totalDocs) {
      const batch = await this.db.collection(sourceCollection)
        .find()
        .skip(migrated)
        .limit(batchSize)
        .toArray();

      if (batch.length === 0) break;

      await this.db.collection(targetCollection).insertMany(batch, {
        ordered: false
      });

      migrated += batch.length;
      console.log(`Migrated: ${migrated}/${totalDocs} (${Math.round(migrated/totalDocs*100)}%)`);
    }

    console.log('Migration complete');

    // Vérifier
    const targetCount = await this.db.collection(targetCollection).countDocuments();
    console.log(`Verification: ${targetCount} documents in target`);

    return {
      source: totalDocs,
      target: targetCount,
      success: totalDocs === targetCount
    };
  }

  async compareStorageSize(standardCollection, tsCollection) {
    const [std, ts] = await Promise.all([
      this.db.collection(standardCollection).stats(),
      this.db.collection(tsCollection).stats()
    ]);

    return {
      standard: {
        size: std.size,
        storageSize: std.storageSize,
        count: std.count
      },
      timeSeries: {
        size: ts.size,
        storageSize: ts.storageSize,
        count: ts.count
      },
      savings: {
        bytes: std.storageSize - ts.storageSize,
        percent: Math.round((1 - ts.storageSize / std.storageSize) * 100)
      }
    };
  }
}

// Utilisation
const migration = new TimeSeriesMigration(db);

await migration.migrateFromStandard('old_sensor_data', 'sensor_data_ts', {
  timeField: 'timestamp',
  metaField: 'metadata',
  granularity: 'seconds'
});

const comparison = await migration.compareStorageSize('old_sensor_data', 'sensor_data_ts');
console.log('Storage comparison:', comparison);
// Exemple: { savings: { bytes: 450MB, percent: 92 } }
```

---

## Conclusion

Les Time Series Collections sont une fonctionnalité puissante de MongoDB pour :
- ✅ **Compression automatique** (90-95% d'économie d'espace)
- ✅ **Performances d'écriture améliorées** (+15-20%)
- ✅ **Requêtes temporelles optimisées**
- ✅ **TTL intégré** pour gestion automatique de la rétention
- ✅ **Bucketing intelligent** par timestamp et metadata

**Points clés à retenir :**
1. Toujours définir `metaField` pour bucketing optimal
2. Choisir la `granularity` selon la fréquence réelle des données
3. Privilégier insertions en ordre chronologique
4. Utiliser batch inserts pour haute performance
5. Configurer TTL pour gestion automatique de la rétention
6. Impossible d'update/delete documents individuels

**Cas d'usage idéaux :**
- Monitoring IoT et capteurs
- Données financières (ticks, OHLC)
- Métriques applicatives (APM)
- Logs avec timestamps
- Données météorologiques
- Analytics temporels

**Quand ne pas utiliser :**
- Besoin de mises à jour fréquentes de documents
- Suppressions sélectives nécessaires
- Données sans composante temporelle forte
- Fréquence d'échantillonnage très irrégulière

---


⏭️ [Clustered Collections](/16-fonctionnalites-avancees/05-clustered-collections.md)
