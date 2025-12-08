🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.10 Diagnostics avec FTDC

## Introduction

**FTDC (Full Time Diagnostic Data Capture)** est le système de diagnostic intégré à MongoDB qui collecte automatiquement et en continu des métriques détaillées sur l'état du serveur, les performances et le système d'exploitation. Introduit dans MongoDB 3.2, FTDC constitue la "boîte noire" de MongoDB, enregistrant tout ce qui se passe dans le serveur avec un impact minimal sur les performances (< 1% overhead).

Pour les SRE et administrateurs système, FTDC est **l'outil de diagnostic post-mortem par excellence**. Contrairement aux dashboards temps réel qui peuvent manquer des événements transitoires, FTDC capture tout avec une granularité fine (1 seconde par défaut), permettant une analyse forensique précise des incidents.

### Pourquoi FTDC est Crucial

```
┌───────────────────────────────────────────────────────────────┐
│              FTDC vs Autres Outils de Monitoring              │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Prometheus/Grafana          FTDC                             │
│  ├─ 15-60s granularity       ├─ 1s granularity                │
│  ├─ Peut manquer spikes      ├─ Capture tout                  │
│  ├─ Besoin infra externe     ├─ Intégré, toujours actif       │
│  ├─ Storage externe          ├─ Fichiers locaux (rotating)    │
│  └─ Vue globale cluster      └─ Vue détaillée par instance    │
│                                                               │
│  Logs MongoDB                FTDC                             │
│  ├─ Events discrets          ├─ Métriques continues           │
│  ├─ Verbosité configurable   ├─ Toujours verbeux              │
│  ├─ Texte non structuré      ├─ Format binaire structuré      │
│  └─ Analyse manuelle         └─ Analyse programmable          │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Cas d'usage FTDC** :
- **Post-mortem d'incidents** : "Qu'est-ce qui s'est passé exactement à 14h32:15 ?"
- **Analyse de performance** : Identifier les patterns de dégradation
- **Détection d'anomalies** : Comparer comportement normal vs problématique
- **Validation de changements** : Impact avant/après d'une modification
- **Support MongoDB** : Fichiers FTDC requis pour tickets support

---

## Architecture et Fonctionnement

### Composants FTDC

```
┌─────────────────────────────────────────────────────────────┐
│                     Architecture FTDC                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                      MongoDB Process                        │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │            FTDC Collector Thread                       │ │
│  │                                                        │ │
│  │  Every 1 second (default):                             │ │
│  │  1. Collect serverStatus()                             │ │
│  │  2. Collect replSetGetStatus()                         │ │
│  │  3. Collect system metrics (CPU, memory, disk)         │ │
│  │  4. Compress data (zlib)                               │ │
│  │  5. Append to current FTDC file                        │ │
│  └────────────────────┬───────────────────────────────────┘ │
│                       │                                     │
│                       ▼                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │         FTDC File Writer                               │ │
│  │                                                        │ │
│  │  • Binary format (BSON-like)                           │ │
│  │  • Compressed (zlib)                                   │ │
│  │  • Rotating files (300 MB max default)                 │ │
│  │  • Retention: Keep N files (default: retain all)       │ │
│  └────────────────────┬───────────────────────────────────┘ │
└───────────────────────┼─────────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────────┐
              │   Filesystem         │
              │                      │
              │  diagnostic.data/    │
              │  ├─ metrics.*        │
              │  ├─ metrics.*        │
              │  └─ metrics.*        │
              └──────────────────────┘
```

### Cycle de Vie des Fichiers FTDC

```
Timeline:
─────────────────────────────────────────────────────────────────

t=0s        t=300s      t=600s      t=900s
│           │           │           │
▼           ▼           ▼           ▼
metrics.001 metrics.002 metrics.003 metrics.004
[  NEW  ]   [WRITING]   [WRITING]   [WRITING]
            [ROTATE]    [ROTATE]    [ROTATE]

File Rotation Triggers:
1. File size exceeds 300 MB (default)
2. mongod restart
3. Manual rotation (not exposed)

File Naming:
metrics.<timestamp>
metrics.interim (current active file)
metrics.<timestamp>.gz (archived, if enabled)
```

### Paramètres de Configuration

```yaml
# mongod.conf
diagnosticDataCollectionEnabled: true  # Enable/disable (default: true)
diagnosticDataCollectionDirectorySizeMB: 200  # Max total size (default: 200 MB)
diagnosticDataCollectionPeriodMillis: 1000     # Collection interval (default: 1000ms)
diagnosticDataCollectionFileSizeMB: 300        # Max file size (default: 300 MB)
```

**Runtime configuration** :
```javascript
// Vérifier l'état
db.adminCommand({ getDiagnosticData: 1 })

// Désactiver temporairement (nécessite restart pour réactiver)
db.adminCommand({ setParameter: 1, diagnosticDataCollectionEnabled: false })

// Ajuster la période de collection
db.adminCommand({ setParameter: 1, diagnosticDataCollectionPeriodMillis: 5000 })
```

---

## Données Collectées par FTDC

### Catégories de Métriques

FTDC collecte **plusieurs centaines de métriques** regroupées en catégories :

#### 1. ServerStatus Metrics

```javascript
// Extrait des métriques serverStatus collectées
{
  "version": "7.0.4",
  "process": "mongod",
  "pid": 12345,
  "uptime": 3600,

  // Operations
  "opcounters": {
    "insert": 1234,
    "query": 5678,
    "update": 910,
    "delete": 123,
    "getmore": 456,
    "command": 7890
  },

  // Connections
  "connections": {
    "current": 87,
    "available": 51113,
    "totalCreated": 2345,
    "active": 23
  },

  // Memory
  "mem": {
    "bits": 64,
    "resident": 8192,    // MB
    "virtual": 16384,
    "supported": true
  },

  // WiredTiger Cache
  "wiredTiger": {
    "cache": {
      "bytes currently in the cache": 8589934592,
      "tracked dirty bytes in the cache": 858993459,
      "maximum bytes configured": 10737418240,
      "pages evicted by application threads": 12345,
      "pages read into cache": 234567,
      "pages written from cache": 123456
    },
    "concurrentTransactions": {
      "read": {
        "out": 12,
        "available": 116,
        "totalTickets": 128
      },
      "write": {
        "out": 8,
        "available": 120,
        "totalTickets": 128
      }
    }
  },

  // Global Lock
  "globalLock": {
    "totalTime": 3600000000,
    "currentQueue": {
      "total": 5,
      "readers": 3,
      "writers": 2
    },
    "activeClients": {
      "total": 23,
      "readers": 15,
      "writers": 8
    }
  },

  // Network
  "network": {
    "bytesIn": 123456789,
    "bytesOut": 987654321,
    "numRequests": 45678
  },

  // Opcounters repl
  "opcountersRepl": {
    "insert": 1234,
    "query": 0,
    "update": 910,
    "delete": 123,
    "getmore": 0,
    "command": 456
  }
}
```

#### 2. ReplSetGetStatus Metrics (Replica Sets)

```javascript
{
  "set": "production-rs0",
  "date": ISODate("2024-12-08T16:00:00Z"),
  "myState": 1,  // PRIMARY
  "term": 42,
  "syncSourceHost": "",
  "syncSourceId": -1,
  "heartbeatIntervalMillis": 2000,

  "members": [
    {
      "name": "mongo-01:27017",
      "health": 1,
      "state": 1,  // PRIMARY
      "stateStr": "PRIMARY",
      "uptime": 86400,
      "optime": {
        "ts": Timestamp(1702220400, 1),
        "t": 42
      },
      "optimeDate": ISODate("2024-12-08T16:00:00Z"),
      "syncSourceHost": "",
      "configVersion": 5,
      "electionTime": Timestamp(1701616000, 1),
      "electionDate": ISODate("2024-12-03T16:00:00Z"),
      "self": true
    },
    {
      "name": "mongo-02:27017",
      "health": 1,
      "state": 2,  // SECONDARY
      "stateStr": "SECONDARY",
      "uptime": 86400,
      "optime": {
        "ts": Timestamp(1702220400, 1),
        "t": 42
      },
      "optimeDurable": {
        "ts": Timestamp(1702220400, 1),
        "t": 42
      },
      "optimeDate": ISODate("2024-12-08T16:00:00Z"),
      "syncSourceHost": "mongo-01:27017",
      "syncSourceId": 0,
      "configVersion": 5,
      "lastHeartbeat": ISODate("2024-12-08T16:00:00Z"),
      "lastHeartbeatRecv": ISODate("2024-12-08T16:00:00Z"),
      "pingMs": 0
    }
  ],

  "ok": 1
}
```

#### 3. System Metrics (OS Level)

```javascript
{
  "start": ISODate("2024-12-08T16:00:00Z"),
  "end": ISODate("2024-12-08T16:00:01Z"),

  // CPU metrics
  "cpu": {
    "num_cpus": 16,
    "cpu_ms_user": 45000,
    "cpu_ms_nice": 0,
    "cpu_ms_system": 15000,
    "cpu_ms_idle": 3940000,
    "cpu_ms_iowait": 0,
    "cpu_ms_irq": 0,
    "cpu_ms_softirq": 0,
    "cpu_ms_steal": 0
  },

  // Memory metrics
  "memory": {
    "mem_total_kb": 33554432,
    "mem_free_kb": 5242880,
    "mem_available_kb": 20971520,
    "mem_buffers_kb": 1048576,
    "mem_cached_kb": 15728640,
    "swap_total_kb": 4194304,
    "swap_free_kb": 4194304,
    "swap_cached_kb": 0
  },

  // Disk I/O metrics (per device)
  "disks": {
    "sda": {
      "reads": 123456,
      "reads_merged": 1234,
      "sectors_read": 12345678,
      "read_time_ms": 123456,
      "writes": 234567,
      "writes_merged": 2345,
      "sectors_written": 23456789,
      "write_time_ms": 234567,
      "io_in_progress": 0,
      "io_time_ms": 123456,
      "io_queued_ms": 12345
    }
  },

  // Network I/O metrics (per interface)
  "network": {
    "eth0": {
      "rx_bytes": 123456789012,
      "rx_packets": 123456789,
      "rx_errors": 0,
      "rx_dropped": 0,
      "tx_bytes": 987654321098,
      "tx_packets": 987654321,
      "tx_errors": 0,
      "tx_dropped": 0
    }
  }
}
```

---

## Localisation et Format des Fichiers

### Structure des Répertoires

```bash
# Default location
/var/lib/mongodb/diagnostic.data/

# Structure
diagnostic.data/
├── metrics.2024-12-08T14-00-00Z-00000
├── metrics.2024-12-08T15-30-00Z-00000
├── metrics.2024-12-08T16-45-00Z-00000
└── metrics.interim                      # Current active file

# Taille typique
$ du -sh /var/lib/mongodb/diagnostic.data/
195M    /var/lib/mongodb/diagnostic.data/

# Nombre de fichiers
$ ls -1 /var/lib/mongodb/diagnostic.data/metrics.* | wc -l
8
```

### Format des Fichiers FTDC

Les fichiers FTDC sont **binaires et compressés** :

```
File Structure:
┌──────────────────────────────────────────┐
│          FTDC File Format                │
├──────────────────────────────────────────┤
│                                          │
│  Header (magic bytes)                    │
│  ├─ Format version                       │
│  └─ Compression type (zlib)              │
│                                          │
│  Compressed Chunks                       │
│  ├─ Chunk 1 (timestamp range)            │
│  │  ├─ Metadata                          │
│  │  ├─ Schema (field names)              │
│  │  └─ Data (compressed BSON)            │
│  │                                       │
│  ├─ Chunk 2                              │
│  ├─ Chunk 3                              │
│  └─ ...                                  │
│                                          │
└──────────────────────────────────────────┘

Compression:
- zlib compression per chunk
- ~90% compression ratio
- 1 hour of data ≈ 25 MB compressed
```

**Pourquoi format binaire ?**
- Haute densité (1s granularity × nombreuses métriques)
- Compression efficace
- Parsing rapide
- Pas de perte de précision

---

## Outils d'Analyse FTDC

### 1. MongoDB FTDC Decoder (Officiel)

Outil Python développé par MongoDB Inc. pour décoder les fichiers FTDC.

```bash
# Installation
pip install mongod-ftdc

# Usage basique
mongod-ftdc decode metrics.2024-12-08T14-00-00Z-00000

# Output: JSON par sample (un par seconde)
```

**Exemple de décodage** :

```bash
# Décoder et extraire métriques spécifiques
mongod-ftdc decode --fields "serverStatus.opcounters.query,serverStatus.connections.current" \
  metrics.2024-12-08T14-00-00Z-00000 > decoded.json

# Format JSON lisible
{
  "start": "2024-12-08T14:00:00Z",
  "serverStatus": {
    "opcounters": {
      "query": 5678
    },
    "connections": {
      "current": 87
    }
  }
}
```

### 2. Script Python d'Analyse

```python
#!/usr/bin/env python3
# ftdc_analyzer.py

import json
import subprocess
from datetime import datetime
from collections import defaultdict
import statistics

class FTDCAnalyzer:
    def __init__(self, ftdc_file):
        self.ftdc_file = ftdc_file
        self.data = []
        self._decode()

    def _decode(self):
        """Decode FTDC file to JSON"""
        result = subprocess.run(
            ['mongod-ftdc', 'decode', self.ftdc_file],
            capture_output=True,
            text=True
        )

        # Parse line-delimited JSON
        for line in result.stdout.split('\n'):
            if line.strip():
                self.data.append(json.loads(line))

    def get_metric_timeseries(self, metric_path):
        """Extract time series for a specific metric

        Example: 'serverStatus.opcounters.query'
        """
        series = []

        for sample in self.data:
            timestamp = sample.get('start')

            # Navigate nested dict
            value = sample
            for key in metric_path.split('.'):
                value = value.get(key, {})

            if isinstance(value, (int, float)):
                series.append({
                    'timestamp': timestamp,
                    'value': value
                })

        return series

    def calculate_rate(self, metric_path):
        """Calculate per-second rate for counter metric"""
        series = self.get_metric_timeseries(metric_path)
        rates = []

        for i in range(1, len(series)):
            prev = series[i-1]
            curr = series[i]

            # Calculate delta
            delta_value = curr['value'] - prev['value']
            delta_time = 1  # FTDC samples every 1 second

            rates.append({
                'timestamp': curr['timestamp'],
                'rate': delta_value / delta_time
            })

        return rates

    def detect_spikes(self, metric_path, threshold_sigma=3):
        """Detect spikes using statistical analysis"""
        series = self.get_metric_timeseries(metric_path)
        values = [s['value'] for s in series]

        mean = statistics.mean(values)
        stdev = statistics.stdev(values)

        spikes = []
        for sample in series:
            z_score = (sample['value'] - mean) / stdev
            if abs(z_score) > threshold_sigma:
                spikes.append({
                    'timestamp': sample['timestamp'],
                    'value': sample['value'],
                    'z_score': z_score,
                    'deviation': sample['value'] - mean
                })

        return spikes

    def correlate_metrics(self, metric1, metric2):
        """Find correlation between two metrics"""
        series1 = self.get_metric_timeseries(metric1)
        series2 = self.get_metric_timeseries(metric2)

        # Ensure same length
        min_len = min(len(series1), len(series2))
        values1 = [s['value'] for s in series1[:min_len]]
        values2 = [s['value'] for s in series2[:min_len]]

        # Calculate Pearson correlation
        mean1 = statistics.mean(values1)
        mean2 = statistics.mean(values2)

        numerator = sum((v1 - mean1) * (v2 - mean2)
                       for v1, v2 in zip(values1, values2))

        denominator = (
            sum((v1 - mean1)**2 for v1 in values1) ** 0.5 *
            sum((v2 - mean2)**2 for v2 in values2) ** 0.5
        )

        correlation = numerator / denominator if denominator != 0 else 0

        return correlation

    def generate_report(self):
        """Generate analysis report"""
        report = []
        report.append("=" * 70)
        report.append("FTDC Analysis Report")
        report.append("=" * 70)
        report.append(f"File: {self.ftdc_file}")
        report.append(f"Samples: {len(self.data)}")
        report.append(f"Duration: {len(self.data)} seconds\n")

        # Analyze key metrics
        metrics = {
            'Queries/sec': 'serverStatus.opcounters.query',
            'Inserts/sec': 'serverStatus.opcounters.insert',
            'Connections': 'serverStatus.connections.current',
            'Memory (MB)': 'serverStatus.mem.resident',
            'Queue (readers)': 'serverStatus.globalLock.currentQueue.readers',
            'Queue (writers)': 'serverStatus.globalLock.currentQueue.writers'
        }

        for label, path in metrics.items():
            series = self.get_metric_timeseries(path)
            if series:
                values = [s['value'] for s in series]
                report.append(f"{label}:")
                report.append(f"  Min: {min(values)}")
                report.append(f"  Max: {max(values)}")
                report.append(f"  Avg: {statistics.mean(values):.2f}")
                report.append(f"  Stdev: {statistics.stdev(values):.2f}\n")

        return '\n'.join(report)

# Usage
analyzer = FTDCAnalyzer('metrics.2024-12-08T14-00-00Z-00000')
print(analyzer.generate_report())

# Detect query rate spikes
spikes = analyzer.detect_spikes('serverStatus.opcounters.query', threshold_sigma=3)
if spikes:
    print("\nQuery Rate Spikes Detected:")
    for spike in spikes[:5]:  # Top 5
        print(f"  {spike['timestamp']}: {spike['value']} (z-score: {spike['z_score']:.2f})")

# Correlation analysis
corr = analyzer.correlate_metrics(
    'serverStatus.opcounters.query',
    'serverStatus.globalLock.currentQueue.readers'
)
print(f"\nCorrelation (queries vs read queue): {corr:.2f}")
```

### 3. Visualisation avec Python/Matplotlib

```python
#!/usr/bin/env python3
# ftdc_visualizer.py

import matplotlib.pyplot as plt
import matplotlib.dates as mdates
from datetime import datetime
from ftdc_analyzer import FTDCAnalyzer

class FTDCVisualizer:
    def __init__(self, ftdc_file):
        self.analyzer = FTDCAnalyzer(ftdc_file)

    def plot_metric(self, metric_path, title, ylabel):
        """Plot single metric over time"""
        series = self.analyzer.get_metric_timeseries(metric_path)

        timestamps = [datetime.fromisoformat(s['timestamp'].replace('Z', '+00:00'))
                     for s in series]
        values = [s['value'] for s in series]

        fig, ax = plt.subplots(figsize=(12, 6))
        ax.plot(timestamps, values, linewidth=1)
        ax.set_title(title)
        ax.set_xlabel('Time')
        ax.set_ylabel(ylabel)
        ax.grid(True, alpha=0.3)

        # Format x-axis
        ax.xaxis.set_major_formatter(mdates.DateFormatter('%H:%M:%S'))
        plt.xticks(rotation=45)
        plt.tight_layout()

        return fig

    def plot_multiple_metrics(self, metrics_config):
        """Plot multiple metrics in subplots"""
        n_metrics = len(metrics_config)
        fig, axes = plt.subplots(n_metrics, 1, figsize=(12, 4*n_metrics))

        if n_metrics == 1:
            axes = [axes]

        for ax, (label, path, ylabel) in zip(axes, metrics_config):
            series = self.analyzer.get_metric_timeseries(path)

            timestamps = [datetime.fromisoformat(s['timestamp'].replace('Z', '+00:00'))
                         for s in series]
            values = [s['value'] for s in series]

            ax.plot(timestamps, values, linewidth=1)
            ax.set_title(label)
            ax.set_ylabel(ylabel)
            ax.grid(True, alpha=0.3)
            ax.xaxis.set_major_formatter(mdates.DateFormatter('%H:%M:%S'))
            plt.setp(ax.xaxis.get_majorticklabels(), rotation=45)

        plt.tight_layout()
        return fig

    def plot_correlation_heatmap(self, metric_paths, labels):
        """Plot correlation heatmap between metrics"""
        import numpy as np

        n = len(metric_paths)
        correlations = np.zeros((n, n))

        for i, path1 in enumerate(metric_paths):
            for j, path2 in enumerate(metric_paths):
                if i == j:
                    correlations[i, j] = 1.0
                else:
                    corr = self.analyzer.correlate_metrics(path1, path2)
                    correlations[i, j] = corr

        fig, ax = plt.subplots(figsize=(10, 8))
        im = ax.imshow(correlations, cmap='coolwarm', aspect='auto', vmin=-1, vmax=1)

        ax.set_xticks(range(n))
        ax.set_yticks(range(n))
        ax.set_xticklabels(labels, rotation=45, ha='right')
        ax.set_yticklabels(labels)

        # Add correlation values
        for i in range(n):
            for j in range(n):
                text = ax.text(j, i, f'{correlations[i, j]:.2f}',
                             ha="center", va="center", color="black")

        ax.set_title("Metric Correlation Matrix")
        plt.colorbar(im, ax=ax)
        plt.tight_layout()

        return fig

# Usage
viz = FTDCVisualizer('metrics.2024-12-08T14-00-00Z-00000')

# Single metric plot
fig1 = viz.plot_metric(
    'serverStatus.opcounters.query',
    'Query Operations Over Time',
    'Queries/sec'
)
fig1.savefig('queries.png', dpi=150)

# Multiple metrics
metrics = [
    ('Operations', 'serverStatus.opcounters.query', 'ops/sec'),
    ('Connections', 'serverStatus.connections.current', 'count'),
    ('Queue Depth', 'serverStatus.globalLock.currentQueue.readers', 'count'),
    ('Memory', 'serverStatus.mem.resident', 'MB')
]
fig2 = viz.plot_multiple_metrics(metrics)
fig2.savefig('overview.png', dpi=150)

# Correlation heatmap
metric_paths = [
    'serverStatus.opcounters.query',
    'serverStatus.connections.current',
    'serverStatus.globalLock.currentQueue.readers',
    'serverStatus.mem.resident'
]
labels = ['Queries', 'Connections', 'Queue', 'Memory']
fig3 = viz.plot_correlation_heatmap(metric_paths, labels)
fig3.savefig('correlations.png', dpi=150)
```

---

## Cas d'Usage : Analyse Post-Mortem

### Scénario : Incident de Performance

**Contexte** : Alerte à 14h32 pour latence élevée, retour à la normale à 14h38.

#### Étape 1 : Identifier les Fichiers FTDC Pertinents

```bash
# Lister les fichiers couvrant la période d'incident
ls -lh /var/lib/mongodb/diagnostic.data/

# Output
-rw------- mongodb mongodb 25M Dec 8 14:00 metrics.2024-12-08T14-00-00Z-00000
-rw------- mongodb mongodb 28M Dec 8 14:30 metrics.2024-12-08T14-30-00Z-00000
-rw------- mongodb mongodb 12M Dec 8 15:00 metrics.2024-12-08T15-00-00Z-00000

# Le fichier 14:30 couvre notre incident (14:32-14:38)
```

#### Étape 2 : Extraire Métriques Clés

```python
#!/usr/bin/env python3
# incident_analysis.py

from ftdc_analyzer import FTDCAnalyzer
from datetime import datetime, timedelta

class IncidentAnalyzer:
    def __init__(self, ftdc_file, incident_start, incident_end):
        self.analyzer = FTDCAnalyzer(ftdc_file)
        self.incident_start = datetime.fromisoformat(incident_start)
        self.incident_end = datetime.fromisoformat(incident_end)

    def filter_incident_window(self, series):
        """Filter time series to incident window"""
        return [
            s for s in series
            if self.incident_start <= datetime.fromisoformat(s['timestamp'].replace('Z', '+00:00')) <= self.incident_end
        ]

    def analyze_incident(self):
        """Comprehensive incident analysis"""

        print("=" * 70)
        print("INCIDENT ANALYSIS")
        print("=" * 70)
        print(f"Incident Window: {self.incident_start} to {self.incident_end}")
        print(f"Duration: {(self.incident_end - self.incident_start).total_seconds()} seconds\n")

        # 1. Operations Analysis
        print("1. OPERATIONS ANALYSIS")
        print("-" * 70)

        for op_type in ['query', 'insert', 'update', 'delete']:
            rates = self.analyzer.calculate_rate(f'serverStatus.opcounters.{op_type}')
            incident_rates = self.filter_incident_window(rates)

            if incident_rates:
                values = [r['rate'] for r in incident_rates]
                print(f"{op_type.capitalize()}s/sec:")
                print(f"  Peak: {max(values):.2f}")
                print(f"  Avg: {sum(values)/len(values):.2f}")

        print()

        # 2. Queue Analysis
        print("2. QUEUE DEPTH ANALYSIS")
        print("-" * 70)

        for queue_type in ['readers', 'writers']:
            series = self.analyzer.get_metric_timeseries(
                f'serverStatus.globalLock.currentQueue.{queue_type}'
            )
            incident_series = self.filter_incident_window(series)

            if incident_series:
                values = [s['value'] for s in incident_series]
                print(f"{queue_type.capitalize()} Queue:")
                print(f"  Peak: {max(values)}")
                print(f"  Avg: {sum(values)/len(values):.2f}")

        print()

        # 3. Connection Analysis
        print("3. CONNECTION ANALYSIS")
        print("-" * 70)

        conn_series = self.analyzer.get_metric_timeseries('serverStatus.connections.current')
        incident_conns = self.filter_incident_window(conn_series)

        if incident_conns:
            values = [s['value'] for s in incident_conns]
            print(f"Connections:")
            print(f"  Peak: {max(values)}")
            print(f"  Avg: {sum(values)/len(values):.2f}")

        print()

        # 4. Cache Analysis
        print("4. CACHE ANALYSIS")
        print("-" * 70)

        cache_used = self.analyzer.get_metric_timeseries(
            'serverStatus.wiredTiger.cache.bytes currently in the cache'
        )
        cache_max = self.analyzer.get_metric_timeseries(
            'serverStatus.wiredTiger.cache.maximum bytes configured'
        )

        if cache_used and cache_max:
            incident_used = self.filter_incident_window(cache_used)
            max_val = cache_max[0]['value']

            percentages = [(s['value'] / max_val) * 100 for s in incident_used]
            print(f"Cache Usage:")
            print(f"  Peak: {max(percentages):.2f}%")
            print(f"  Avg: {sum(percentages)/len(percentages):.2f}%")

        print()

        # 5. System Resource Analysis
        print("5. SYSTEM RESOURCES")
        print("-" * 70)

        # CPU (if available in FTDC)
        cpu_user = self.analyzer.get_metric_timeseries('systemMetrics.cpu.cpu_ms_user')
        if cpu_user:
            incident_cpu = self.filter_incident_window(cpu_user)
            # Calculate CPU percentage (assuming 1 second intervals)
            print("CPU usage data available (check detailed analysis)")

        print()

        # 6. Correlation Analysis
        print("6. CORRELATION ANALYSIS")
        print("-" * 70)

        correlations = {
            'Queries vs Queue': self.analyzer.correlate_metrics(
                'serverStatus.opcounters.query',
                'serverStatus.globalLock.currentQueue.readers'
            ),
            'Connections vs Queue': self.analyzer.correlate_metrics(
                'serverStatus.connections.current',
                'serverStatus.globalLock.currentQueue.readers'
            ),
            'Queue vs Memory': self.analyzer.correlate_metrics(
                'serverStatus.globalLock.currentQueue.readers',
                'serverStatus.mem.resident'
            )
        }

        for label, corr in correlations.items():
            print(f"{label}: {corr:.3f}")

        print()

        # 7. Conclusion
        print("7. CONCLUSIONS")
        print("-" * 70)

        # Detect likely causes based on patterns
        queue_peak = max([s['value'] for s in self.filter_incident_window(
            self.analyzer.get_metric_timeseries('serverStatus.globalLock.currentQueue.readers')
        )])

        if queue_peak > 50:
            print("⚠ High queue depth detected (> 50)")
            print("  → Likely cause: Slow queries or lock contention")

        if correlations['Queries vs Queue'] > 0.7:
            print("⚠ Strong correlation between queries and queue")
            print("  → Likely cause: Query performance issue")

        print("\n" + "=" * 70)

# Usage
analyzer = IncidentAnalyzer(
    'metrics.2024-12-08T14-30-00Z-00000',
    '2024-12-08T14:32:00Z',
    '2024-12-08T14:38:00Z'
)
analyzer.analyze_incident()
```

#### Étape 3 : Visualisation de l'Incident

```python
#!/usr/bin/env python3
# incident_visualization.py

from ftdc_visualizer import FTDCVisualizer
import matplotlib.pyplot as plt
import matplotlib.dates as mdates
from datetime import datetime

viz = FTDCVisualizer('metrics.2024-12-08T14-30-00Z-00000')

# Create incident visualization
fig, axes = plt.subplots(4, 1, figsize=(14, 12))

# Incident window
incident_start = datetime.fromisoformat('2024-12-08T14:32:00+00:00')
incident_end = datetime.fromisoformat('2024-12-08T14:38:00+00:00')

metrics_to_plot = [
    ('serverStatus.opcounters.query', 'Query Rate', 'ops/sec'),
    ('serverStatus.connections.current', 'Active Connections', 'count'),
    ('serverStatus.globalLock.currentQueue.readers', 'Read Queue Depth', 'count'),
    ('serverStatus.mem.resident', 'Memory Usage', 'MB')
]

for ax, (path, title, ylabel) in zip(axes, metrics_to_plot):
    series = viz.analyzer.get_metric_timeseries(path)

    timestamps = [datetime.fromisoformat(s['timestamp'].replace('Z', '+00:00'))
                 for s in series]
    values = [s['value'] for s in series]

    ax.plot(timestamps, values, linewidth=1, color='steelblue')
    ax.set_title(title, fontweight='bold')
    ax.set_ylabel(ylabel)
    ax.grid(True, alpha=0.3)

    # Highlight incident window
    ax.axvspan(incident_start, incident_end, alpha=0.3, color='red', label='Incident')

    ax.xaxis.set_major_formatter(mdates.DateFormatter('%H:%M'))
    ax.legend()

plt.suptitle('Incident Timeline Analysis - 2024-12-08',
             fontsize=16, fontweight='bold', y=0.995)
plt.tight_layout()
plt.savefig('incident_timeline.png', dpi=150, bbox_inches='tight')
print("Visualization saved to incident_timeline.png")
```

---

## Cas d'Usage Avancés

### 1. Détection Automatique d'Anomalies

```python
#!/usr/bin/env python3
# anomaly_detector.py

import numpy as np
from sklearn.ensemble import IsolationForest
from ftdc_analyzer import FTDCAnalyzer

class AnomalyDetector:
    def __init__(self, ftdc_file):
        self.analyzer = FTDCAnalyzer(ftdc_file)

    def detect_multivariate_anomalies(self, metric_paths):
        """Detect anomalies using multiple metrics"""

        # Extract time series for each metric
        data_matrix = []
        timestamps = []

        for path in metric_paths:
            series = self.analyzer.get_metric_timeseries(path)
            if not timestamps:
                timestamps = [s['timestamp'] for s in series]
            data_matrix.append([s['value'] for s in series])

        # Transpose to get samples × features
        X = np.array(data_matrix).T

        # Train Isolation Forest
        clf = IsolationForest(contamination=0.05, random_state=42)
        predictions = clf.fit_predict(X)

        # Extract anomalies
        anomalies = []
        for i, pred in enumerate(predictions):
            if pred == -1:  # Anomaly
                anomalies.append({
                    'timestamp': timestamps[i],
                    'values': {
                        path: X[i, j]
                        for j, path in enumerate(metric_paths)
                    }
                })

        return anomalies

    def report_anomalies(self, anomalies):
        """Generate anomaly report"""
        if not anomalies:
            print("No anomalies detected.")
            return

        print(f"Detected {len(anomalies)} anomalies:\n")

        for i, anomaly in enumerate(anomalies[:10], 1):  # Top 10
            print(f"Anomaly {i}:")
            print(f"  Timestamp: {anomaly['timestamp']}")
            print("  Values:")
            for metric, value in anomaly['values'].items():
                print(f"    {metric}: {value:.2f}")
            print()

# Usage
detector = AnomalyDetector('metrics.2024-12-08T14-30-00Z-00000')

# Detect anomalies using multiple metrics
anomalies = detector.detect_multivariate_anomalies([
    'serverStatus.opcounters.query',
    'serverStatus.connections.current',
    'serverStatus.globalLock.currentQueue.readers',
    'serverStatus.mem.resident'
])

detector.report_anomalies(anomalies)
```

### 2. Comparaison Avant/Après Changement

```python
#!/usr/bin/env python3
# before_after_comparison.py

from ftdc_analyzer import FTDCAnalyzer
import statistics

class BeforeAfterComparison:
    def __init__(self, ftdc_before, ftdc_after):
        self.before = FTDCAnalyzer(ftdc_before)
        self.after = FTDCAnalyzer(ftdc_after)

    def compare_metric(self, metric_path):
        """Compare metric statistics before and after"""

        before_series = self.before.get_metric_timeseries(metric_path)
        after_series = self.after.get_metric_timeseries(metric_path)

        before_values = [s['value'] for s in before_series]
        after_values = [s['value'] for s in after_series]

        comparison = {
            'metric': metric_path,
            'before': {
                'mean': statistics.mean(before_values),
                'median': statistics.median(before_values),
                'stdev': statistics.stdev(before_values),
                'min': min(before_values),
                'max': max(before_values),
                'p95': sorted(before_values)[int(len(before_values) * 0.95)]
            },
            'after': {
                'mean': statistics.mean(after_values),
                'median': statistics.median(after_values),
                'stdev': statistics.stdev(after_values),
                'min': min(after_values),
                'max': max(after_values),
                'p95': sorted(after_values)[int(len(after_values) * 0.95)]
            }
        }

        # Calculate percentage change
        comparison['change'] = {
            'mean': ((comparison['after']['mean'] - comparison['before']['mean'])
                    / comparison['before']['mean'] * 100),
            'p95': ((comparison['after']['p95'] - comparison['before']['p95'])
                   / comparison['before']['p95'] * 100)
        }

        return comparison

    def generate_report(self, metrics):
        """Generate before/after comparison report"""

        print("=" * 80)
        print("BEFORE/AFTER COMPARISON REPORT")
        print("=" * 80)
        print()

        for metric_path in metrics:
            comp = self.compare_metric(metric_path)

            print(f"Metric: {metric_path}")
            print("-" * 80)
            print(f"{'Statistic':<15} {'Before':<15} {'After':<15} {'Change':<15}")
            print("-" * 80)

            for stat in ['mean', 'median', 'p95', 'max']:
                before_val = comp['before'][stat]
                after_val = comp['after'][stat]

                if stat in comp['change']:
                    change = comp['change'][stat]
                    change_str = f"{change:+.2f}%"
                else:
                    change_pct = ((after_val - before_val) / before_val * 100)
                    change_str = f"{change_pct:+.2f}%"

                print(f"{stat:<15} {before_val:<15.2f} {after_val:<15.2f} {change_str:<15}")

            print()

            # Verdict
            if abs(comp['change']['mean']) > 10:
                direction = "improved" if comp['change']['mean'] < 0 else "degraded"
                print(f"⚠ Significant change detected: {direction} by {abs(comp['change']['mean']):.1f}%")
            else:
                print("✓ No significant change")

            print("\n")

# Usage - Compare before and after index creation
comparator = BeforeAfterComparison(
    'metrics.before_index.ftdc',
    'metrics.after_index.ftdc'
)

metrics_to_compare = [
    'serverStatus.opcounters.query',
    'serverStatus.globalLock.currentQueue.readers',
    'serverStatus.wiredTiger.cache.pages read into cache'
]

comparator.generate_report(metrics_to_compare)
```

### 3. Génération de Rapport Automatique

```python
#!/usr/bin/env python3
# ftdc_report_generator.py

from ftdc_analyzer import FTDCAnalyzer
from ftdc_visualizer import FTDCVisualizer
import matplotlib.pyplot as plt
from datetime import datetime
import statistics

class FTDCReportGenerator:
    def __init__(self, ftdc_files):
        self.ftdc_files = ftdc_files
        self.analyzers = [FTDCAnalyzer(f) for f in ftdc_files]

    def generate_html_report(self, output_file='ftdc_report.html'):
        """Generate comprehensive HTML report"""

        html = []
        html.append("""
<!DOCTYPE html>
<html>
<head>
    <title>FTDC Analysis Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        h1 { color: #2c3e50; }
        h2 { color: #34495e; border-bottom: 2px solid #3498db; padding-bottom: 10px; }
        .metric-box {
            background: #f8f9fa;
            padding: 15px;
            margin: 10px 0;
            border-left: 4px solid #3498db;
        }
        .alert {
            background: #fee;
            border-left: 4px solid #e74c3c;
            padding: 10px;
            margin: 10px 0;
        }
        table {
            border-collapse: collapse;
            width: 100%;
            margin: 20px 0;
        }
        th, td {
            border: 1px solid #ddd;
            padding: 12px;
            text-align: left;
        }
        th { background-color: #3498db; color: white; }
        tr:nth-child(even) { background-color: #f2f2f2; }
        .chart { margin: 20px 0; }
    </style>
</head>
<body>
""")

        html.append("<h1>FTDC Analysis Report</h1>")
        html.append(f"<p>Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}</p>")
        html.append(f"<p>Files analyzed: {len(self.ftdc_files)}</p>")

        # Summary statistics for each analyzer
        html.append("<h2>Summary Statistics</h2>")

        for i, analyzer in enumerate(self.analyzers):
            html.append(f"<h3>File {i+1}: {self.ftdc_files[i]}</h3>")

            # Key metrics table
            html.append("<table>")
            html.append("<tr><th>Metric</th><th>Min</th><th>Max</th><th>Avg</th><th>P95</th></tr>")

            metrics = {
                'Queries/sec': 'serverStatus.opcounters.query',
                'Connections': 'serverStatus.connections.current',
                'Queue (read)': 'serverStatus.globalLock.currentQueue.readers',
                'Memory (MB)': 'serverStatus.mem.resident'
            }

            for label, path in metrics.items():
                series = analyzer.get_metric_timeseries(path)
                if series:
                    values = [s['value'] for s in series]
                    html.append(f"<tr>")
                    html.append(f"<td>{label}</td>")
                    html.append(f"<td>{min(values):.2f}</td>")
                    html.append(f"<td>{max(values):.2f}</td>")
                    html.append(f"<td>{statistics.mean(values):.2f}</td>")
                    html.append(f"<td>{sorted(values)[int(len(values)*0.95)]:.2f}</td>")
                    html.append(f"</tr>")

            html.append("</table>")

            # Alerts
            spikes = analyzer.detect_spikes('serverStatus.opcounters.query', threshold_sigma=3)
            if spikes:
                html.append("<div class='alert'>")
                html.append(f"<strong>⚠ {len(spikes)} query rate spikes detected</strong><br>")
                for spike in spikes[:3]:
                    html.append(f"• {spike['timestamp']}: {spike['value']:.0f} queries/sec (z-score: {spike['z_score']:.2f})<br>")
                html.append("</div>")

        html.append("</body></html>")

        with open(output_file, 'w') as f:
            f.write('\n'.join(html))

        print(f"Report generated: {output_file}")

# Usage
generator = FTDCReportGenerator([
    'metrics.2024-12-08T14-00-00Z-00000',
    'metrics.2024-12-08T15-00-00Z-00000'
])
generator.generate_html_report()
```

---

## Intégration avec Support MongoDB

### Collecte pour Ticket Support

MongoDB Support requiert souvent les fichiers FTDC pour analyser les problèmes :

```bash
#!/bin/bash
# collect_ftdc_for_support.sh

# Configuration
MONGO_DATA_DIR="/var/lib/mongodb"
FTDC_DIR="$MONGO_DATA_DIR/diagnostic.data"
OUTPUT_DIR="/tmp/mongodb_support_$(date +%Y%m%d_%H%M%S)"
INCIDENT_START="2024-12-08T14:30:00Z"
INCIDENT_END="2024-12-08T15:00:00Z"

echo "Collecting FTDC files for MongoDB Support..."

# Create output directory
mkdir -p "$OUTPUT_DIR"

# Copy FTDC files covering incident period
echo "Copying FTDC files..."
find "$FTDC_DIR" -name "metrics.*" -type f | while read file; do
    # Extract timestamp from filename
    filename=$(basename "$file")

    # Copy file
    cp "$file" "$OUTPUT_DIR/"
done

# Collect MongoDB configuration
echo "Collecting configuration..."
cp /etc/mongod.conf "$OUTPUT_DIR/"

# Collect MongoDB logs for incident period
echo "Collecting logs..."
grep -A 1000 -B 100 "$INCIDENT_START" /var/log/mongodb/mongod.log > "$OUTPUT_DIR/mongod.log"

# Collect system information
echo "Collecting system information..."
cat > "$OUTPUT_DIR/system_info.txt" <<EOF
Hostname: $(hostname)
OS: $(cat /etc/os-release | grep PRETTY_NAME)
Kernel: $(uname -r)
MongoDB Version: $(mongod --version | head -1)
CPU: $(lscpu | grep "Model name" | cut -d: -f2 | xargs)
Memory: $(free -h | grep Mem | awk '{print $2}')
Disk: $(df -h / | tail -1 | awk '{print $2}')
EOF

# Collect replica set status (if applicable)
echo "Collecting replica set status..."
mongo --quiet --eval "rs.status()" > "$OUTPUT_DIR/rs_status.json" 2>/dev/null || echo "Not a replica set"

# Create tarball
echo "Creating archive..."
cd /tmp
tar -czf "mongodb_support_$(date +%Y%m%d_%H%M%S).tar.gz" "$(basename $OUTPUT_DIR)"

echo "Collection complete: mongodb_support_*.tar.gz"
echo "Upload this file to MongoDB Support"
```

---

## Troubleshooting

### Problème 1 : FTDC Non Activé

**Symptômes** :
```bash
ls /var/lib/mongodb/diagnostic.data/
# ls: cannot access: No such file or directory
```

**Diagnostic** :
```javascript
// Vérifier le statut
db.adminCommand({ getDiagnosticData: 1 })

// Output si désactivé
{
  "ok": 0,
  "errmsg": "Diagnostic data collection is disabled"
}
```

**Solution** :
```bash
# Vérifier la configuration
grep -i diagnostic /etc/mongod.conf

# Si absent, ajouter
echo "diagnosticDataCollectionEnabled: true" >> /etc/mongod.conf

# Restart requis
sudo systemctl restart mongod
```

### Problème 2 : Fichiers FTDC Trop Volumineux

**Symptômes** :
```bash
du -sh /var/lib/mongodb/diagnostic.data/
# 5.2G    /var/lib/mongodb/diagnostic.data/
```

**Solution** :
```yaml
# mongod.conf - Limiter la taille
diagnosticDataCollectionDirectorySizeMB: 500  # 500 MB max

# Restart pour appliquer
sudo systemctl restart mongod

# Ou nettoyer manuellement les anciens fichiers
cd /var/lib/mongodb/diagnostic.data
find . -name "metrics.*" -mtime +7 -delete  # Supprimer > 7 jours
```

### Problème 3 : Corruption de Fichier FTDC

**Symptômes** :
```bash
mongod-ftdc decode metrics.2024-12-08T14-00-00Z-00000
# Error: Failed to decompress chunk
```

**Solution** :
```bash
# Vérifier l'intégrité du fichier
file metrics.2024-12-08T14-00-00Z-00000

# Si corrompu, supprimer (nouvelles données continueront d'être collectées)
rm metrics.2024-12-08T14-00-00Z-00000

# Vérifier les logs MongoDB pour cause
grep -i "ftdc\|diagnostic" /var/log/mongodb/mongod.log
```

---

## Bonnes Pratiques

### 1. Gestion des Fichiers FTDC

```yaml
Configuration Recommandée:
  diagnosticDataCollectionEnabled: true
  diagnosticDataCollectionDirectorySizeMB: 500
  diagnosticDataCollectionPeriodMillis: 1000

Rétention:
  Production: 7 jours (rotation automatique)
  Archivage: 30 jours (pour analyse historique)

Stockage:
  ✓ Utiliser volume séparé si possible
  ✓ Monitoring du disk space
  ✓ Alertes si > 80% plein
```

### 2. Archivage Automatique

```bash
#!/bin/bash
# ftdc_archiver.sh

FTDC_DIR="/var/lib/mongodb/diagnostic.data"
ARCHIVE_DIR="/backup/mongodb/ftdc_archive"
RETENTION_DAYS=30

# Créer répertoire d'archive si inexistant
mkdir -p "$ARCHIVE_DIR"

# Archiver les fichiers > 1 jour
find "$FTDC_DIR" -name "metrics.2*" -type f -mtime +1 | while read file; do
    filename=$(basename "$file")

    # Compresser et archiver
    gzip -c "$file" > "$ARCHIVE_DIR/${filename}.gz"

    # Supprimer l'original
    rm "$file"
done

# Nettoyer les archives > 30 jours
find "$ARCHIVE_DIR" -name "metrics.*.gz" -mtime +$RETENTION_DAYS -delete

# Cron: Exécuter quotidiennement
# 0 2 * * * /opt/scripts/ftdc_archiver.sh
```

### 3. Monitoring de FTDC

```python
#!/usr/bin/env python3
# ftdc_health_check.py

import os
import sys
from datetime import datetime, timedelta

class FTDCHealthCheck:
    def __init__(self, ftdc_dir):
        self.ftdc_dir = ftdc_dir

    def check_collection_active(self):
        """Verify FTDC is actively collecting"""

        # Check for recent files
        recent_threshold = datetime.now() - timedelta(minutes=5)

        files = [
            f for f in os.listdir(self.ftdc_dir)
            if f.startswith('metrics.')
        ]

        if not files:
            return False, "No FTDC files found"

        # Check most recent file
        latest_file = max(
            [os.path.join(self.ftdc_dir, f) for f in files],
            key=os.path.getmtime
        )

        mtime = datetime.fromtimestamp(os.path.getmtime(latest_file))

        if mtime < recent_threshold:
            return False, f"No recent FTDC activity (last: {mtime})"

        return True, "FTDC collecting normally"

    def check_disk_usage(self):
        """Check FTDC disk usage"""

        total_size = 0
        for f in os.listdir(self.ftdc_dir):
            if f.startswith('metrics.'):
                total_size += os.path.getsize(os.path.join(self.ftdc_dir, f))

        total_size_mb = total_size / (1024 * 1024)

        # Thresholds
        if total_size_mb > 1000:  # 1 GB
            return False, f"High disk usage: {total_size_mb:.2f} MB"

        return True, f"Disk usage OK: {total_size_mb:.2f} MB"

    def run_checks(self):
        """Run all health checks"""

        print("FTDC Health Check")
        print("=" * 50)

        checks = [
            ("Collection Active", self.check_collection_active),
            ("Disk Usage", self.check_disk_usage)
        ]

        all_passed = True

        for name, check_func in checks:
            passed, message = check_func()
            status = "✓ PASS" if passed else "✗ FAIL"
            print(f"{name}: {status}")
            print(f"  {message}")

            if not passed:
                all_passed = False

        return 0 if all_passed else 1

# Usage
checker = FTDCHealthCheck('/var/lib/mongodb/diagnostic.data')
sys.exit(checker.run_checks())
```

---

## Résumé pour SRE

### Commandes Essentielles

```bash
# Vérifier FTDC actif
mongo --eval "db.adminCommand({ getDiagnosticData: 1 })"

# Lister fichiers FTDC
ls -lh /var/lib/mongodb/diagnostic.data/

# Décoder un fichier
mongod-ftdc decode metrics.2024-12-08T14-00-00Z-00000 > decoded.json

# Extraire métriques spécifiques
mongod-ftdc decode --fields "serverStatus.opcounters,serverStatus.connections" \
  metrics.*.ftdc

# Taille totale FTDC
du -sh /var/lib/mongodb/diagnostic.data/

# Archiver et compresser
tar -czf ftdc_archive.tar.gz /var/lib/mongodb/diagnostic.data/
```

### Checklist Analyse Post-Mortem

```yaml
1. Identifier la période d'incident:
   ✓ Timestamp exact de début/fin
   ✓ Source: logs, alerting, monitoring

2. Localiser fichiers FTDC pertinents:
   ✓ Lister fichiers couvrant la période
   ✓ Vérifier intégrité (pas de corruption)

3. Extraire et décoder:
   ✓ Décoder les fichiers FTDC
   ✓ Extraire fenêtre d'incident

4. Analyser métriques clés:
   ✓ Operations (queries, inserts, etc.)
   ✓ Queue depth
   ✓ Connections
   ✓ Memory/Cache
   ✓ Replication (si applicable)

5. Identifier patterns:
   ✓ Spikes/anomalies
   ✓ Corrélations entre métriques
   ✓ Changements système

6. Visualiser:
   ✓ Créer graphiques timeline
   ✓ Heatmap corrélations
   ✓ Identifier causalité

7. Documenter:
   ✓ Écrire rapport d'incident
   ✓ Root cause analysis
   ✓ Actions préventives
```

### Métriques Critiques dans FTDC

```
Top 10 Metrics to Monitor:
1. serverStatus.opcounters.* (operations rate)
2. serverStatus.globalLock.currentQueue.* (queue depth)
3. serverStatus.connections.current (connections)
4. serverStatus.mem.resident (memory)
5. serverStatus.wiredTiger.cache.* (cache metrics)
6. serverStatus.network.* (network I/O)
7. replSetGetStatus.members.*.optimeDate (replication lag)
8. systemMetrics.cpu.* (CPU usage)
9. systemMetrics.disks.*.io_time_ms (disk I/O)
10. serverStatus.asserts.* (errors/warnings)
```

---

## Conclusion

**FTDC (Full Time Diagnostic Data Capture)** est un outil indispensable pour les SRE MongoDB, offrant une visibilité granulaire (1 seconde) sur l'état du serveur avec un overhead minimal. Les points clés à retenir :

1. **Toujours actif** en production (enabled par défaut)
2. **Première source** pour analyse post-mortem
3. **Complète** les outils temps réel (Prometheus, Grafana)
4. **Requis** pour tickets MongoDB Support
5. **Programmable** via Python pour analyses avancées

**Prochaines étapes** :
- Vérifier que FTDC est activé sur tous les serveurs
- Mettre en place l'archivage automatique
- Développer scripts d'analyse personnalisés
- Intégrer dans workflow d'incident response
- Former l'équipe à l'utilisation des outils FTDC

---

**Références** :
- [MongoDB FTDC Documentation](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/#full-time-diagnostic-data-capture)
- [mongod-ftdc Python Package](https://pypi.org/project/mongod-ftdc/)
- [MongoDB Diagnostic Tools](https://github.com/mongodb/support-tools)
- [FTDC Format Specification](https://github.com/mongodb/mongo/blob/master/src/mongo/db/ftdc/README.md)

⏭️ [Gestion de la mémoire et du cache WiredTiger](/13-monitoring-administration/11-memoire-cache-wiredtiger.md)
