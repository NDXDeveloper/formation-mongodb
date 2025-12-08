🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.11 Gestion de la Mémoire et du Cache WiredTiger

## Introduction

Le **cache WiredTiger** est le composant le plus critique pour les performances de MongoDB. En tant que moteur de stockage par défaut depuis MongoDB 3.2, WiredTiger utilise un cache mémoire sophistiqué pour maintenir les données fréquemment accédées en RAM, réduisant drastiquement les accès disque et améliorant les temps de réponse. Pour les SRE et administrateurs système, une compréhension approfondie du fonctionnement du cache WiredTiger est **essentielle** pour optimiser les performances et résoudre les problèmes.

Un cache mal dimensionné ou saturé peut entraîner :
- **Dégradation des performances** (latence élevée, throughput réduit)
- **Eviction excessive** (thrashing du cache)
- **Pression I/O** (lectures/écritures disque intensives)
- **Instabilité du système** (OOM, swap)

### Architecture Globale de la Mémoire MongoDB

```
┌─────────────────────────────────────────────────────────────┐
│                    MongoDB Memory Layout                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    Physical RAM (Example: 64 GB)            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │      WiredTiger Cache (Configurable)                 │   │
│  │      Default: 50% RAM - 1 GB OR 256 MB               │   │
│  │      Example: (64 GB * 0.5) - 1 GB = 31 GB           │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │  Clean Pages (60-70% typical)                  │  │   │
│  │  │  • Index pages                                 │  │   │
│  │  │  • Data pages                                  │  │   │
│  │  │  • Unmodified, can be evicted immediately      │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  │  ┌────────────────────────────────────────────────┐  │   │
│  │  │  Dirty Pages (20-30% typical, <5% target)      │  │   │
│  │  │  • Modified pages not yet written to disk      │  │   │
│  │  │  • Must be checkpointed before eviction        │  │   │
│  │  └────────────────────────────────────────────────┘  │   │
│  │                                                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │      MongoDB Process Memory (Outside WT Cache)       │   │
│  │      • Connection overhead (~1 MB per connection)    │   │
│  │      • Query execution buffers                       │   │
│  │      • Aggregation pipeline memory                   │   │
│  │      • Replication buffers                           │   │
│  │      • Internal data structures                      │   │
│  │      Typical: 2-5 GB                                 │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │      OS File System Cache                            │   │
│  │      • WiredTiger relies on OS cache for files       │   │
│  │      • Remaining RAM after WT cache                  │   │
│  │      Example: 64 GB - 31 GB - 5 GB = 28 GB           │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘

Memory Allocation Formula:
WiredTiger Cache = max(
  (RAM * 0.5) - 1 GB,      # 50% - 1 GB
  256 MB                    # Minimum
)

Recommended RAM Allocation (Production):
- WiredTiger Cache: 50-60% RAM
- MongoDB Process: 5-10% RAM
- OS + File System Cache: 30-40% RAM
- Safety Buffer: 5-10% RAM (avoid OOM)
```

---

## Architecture du Cache WiredTiger

### Composants du Cache

```
┌─────────────────────────────────────────────────────────────────┐
│              WiredTiger Cache Architecture                      │
└─────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────┐
│                    Cache Memory Pool                          │
│                   (Configured Size: e.g., 30 GB)              │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Page Types:                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   Index     │  │    Data     │  │  Overflow   │            │
│  │   Pages     │  │   Pages     │  │   Pages     │            │
│  │  (B-Tree)   │  │  (Docs)     │  │  (Large)    │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                               │
│  Page States:                                                 │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  Clean Pages                                         │     │
│  │  • In sync with disk                                 │     │
│  │  • Can be evicted immediately                        │     │
│  │  • No checkpoint needed                              │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                               │
│  ┌──────────────────────────────────────────────────────┐     │
│  │  Dirty Pages                                         │     │
│  │  • Modified, not yet written to disk                 │     │
│  │  • Must be checkpointed                              │     │
│  │  • Tracked by dirty bytes metric                     │     │
│  └──────────────────────────────────────────────────────┘     │
│                                                               │
└───────────────────────────────────────────────────────────────┘
           │                      │                      │
           ▼                      ▼                      ▼
    ┌───────────┐        ┌────────────┐        ┌───────────┐
    │  Eviction │        │Checkpoint  │        │  Read     │
    │  Threads  │        │  Thread    │        │  Threads  │
    └───────────┘        └────────────┘        └───────────┘
           │                      │                      │
           ▼                      ▼                      ▼
    ┌────────────────────────────────────────────────────────┐
    │                  WiredTiger Files                      │
    │         (Data files, Journals, Metadata)               │
    └────────────────────────────────────────────────────────┘
```

### Flux de Données

```
Write Operation Flow:
═══════════════════════════════════════════════════════════════

Application Write
      │
      ▼
MongoDB mongod
      │
      ▼
WiredTiger API
      │
      ├─────────────────────────────┐
      │                             │
      ▼                             ▼
Update Cache Page              Write to Journal
(Mark Dirty)                   (WAL - Write Ahead Log)
      │                             │
      │                             ▼
      │                        Disk (immediate)
      │
      ├────────► Accumulate dirty pages
      │
      ▼
Checkpoint Trigger (60s default OR 2 GB journal)
      │
      ▼
Checkpoint Thread
      │
      ├─────► Write dirty pages to data files
      │
      └─────► Update metadata
              │
              ▼
        Clean Pages (can be evicted)

Read Operation Flow:
═══════════════════════════════════════════════════════════════

Application Read
      │
      ▼
MongoDB mongod
      │
      ▼
WiredTiger API
      │
      ▼
Check Cache
      │
      ├─────► Hit: Return from cache (fast)
      │
      └─────► Miss:
              │
              ▼
         Read from disk
              │
              ▼
         Load into cache
         (Evict old pages if needed)
              │
              ▼
         Return to application
```

---

## Métriques du Cache WiredTiger

### Métriques Essentielles

#### 1. Taille et Utilisation du Cache

```javascript
// Via serverStatus
db.serverStatus().wiredTiger.cache

{
  "maximum bytes configured": 10737418240,        // 10 GB configured
  "bytes currently in the cache": 8589934592,     // 8 GB used
  "tracked dirty bytes in the cache": 858993459,  // 819 MB dirty
  "bytes read into cache": 123456789012,          // Cumulative reads
  "bytes written from cache": 98765432109,        // Cumulative writes

  // Key ratios
  "percentage overhead": 8,                       // Cache metadata overhead

  // Important for monitoring
  "pages evicted by application threads": 12345,  // Application evictions (BAD)
  "pages read into cache": 234567,                // Cache misses
  "pages written from cache": 123456              // Cache writes to disk
}
```

**Métriques clés pour monitoring** :

```
Cache Usage Percentage:
= (bytes currently in cache / maximum bytes configured) * 100
Target: 70-90%
Alert: > 95% (saturation)

Dirty Percentage:
= (tracked dirty bytes / bytes currently in cache) * 100
Target: < 5-10%
Alert: > 20% (checkpoint pressure)
Critical: > 40% (eviction problems)

Cache Hit Ratio (approximate):
= pages in cache / (pages read + pages in cache)
Target: > 95%
Alert: < 90% (high miss rate)
```

#### 2. Eviction et Page Management

```javascript
db.serverStatus().wiredTiger.cache

{
  // Eviction statistics
  "eviction server candidate queue empty when topping up": 12,
  "eviction server candidate queue not empty when topping up": 345,
  "eviction server evicting pages": 1234,
  "eviction server slept, because we did not make progress": 56,

  "eviction worker thread active": 4,             // Current active threads
  "eviction worker thread created": 8,            // Max created
  "eviction worker thread evicting pages": 5678,
  "eviction worker thread stable number": 0,

  // Application thread eviction (WARNING SIGN)
  "pages evicted by application threads": 0,      // Should be 0!

  // Eviction walk statistics
  "pages walked for eviction": 123456,
  "pages selected for eviction unable to be evicted": 789,

  // Page read/write
  "pages read into cache": 234567,
  "pages read into cache requiring lookaside entries": 12,
  "pages written from cache": 123456,
  "pages written requiring in-memory restoration": 34
}
```

**Éviction saine** :
```
Eviction by Worker Threads:    99%+  ✓ Good
Eviction by App Threads:        0%   ✓ Good
Eviction Failed:               <1%   ✓ Good

Eviction by Worker Threads:    95%   ⚠ Warning
Eviction by App Threads:        5%   ⚠ Warning
Eviction Failed:               >5%   ⚠ Warning

Eviction by Worker Threads:    80%   ✗ Bad
Eviction by App Threads:       20%   ✗ Bad (performance impact)
Eviction Failed:              >10%   ✗ Bad (thrashing)
```

#### 3. Checkpoint Metrics

```javascript
db.serverStatus().wiredTiger.transaction

{
  "transaction checkpoint currently running": 0,
  "transaction checkpoint generation": 12345,
  "transaction checkpoint max time (msecs)": 5234,
  "transaction checkpoint min time (msecs)": 1234,
  "transaction checkpoint most recent time (msecs)": 2345,
  "transaction checkpoints": 2067,

  // Checkpoint sizing
  "transaction checkpoint scrub dirty target": 0,
  "transaction checkpoint scrub time (msecs)": 0,
  "transaction checkpoint total time (msecs)": 45678,

  // Important: time between checkpoints
  "transaction checkpoint most recent time (msecs)": 2345
}
```

**Checkpoint health** :
```
Checkpoint Duration:
  < 5 seconds:   ✓ Excellent
  5-15 seconds:  ✓ Good
  15-30 seconds: ⚠ Warning (high dirty pages)
  > 30 seconds:  ✗ Critical (I/O bottleneck or dirty cache)

Checkpoint Frequency:
  Every 60s:     ✓ Normal (default)
  < 60s:         ⚠ Triggered by journal size (high write load)
  > 60s:         ✗ Problem (checkpoint may be stuck)
```

#### 4. Ticket Metrics (Concurrency Control)

```javascript
db.serverStatus().wiredTiger.concurrentTransactions

{
  "write": {
    "out": 8,           // Active write tickets
    "available": 120,   // Available write tickets
    "totalTickets": 128 // Total write tickets
  },
  "read": {
    "out": 12,          // Active read tickets
    "available": 116,   // Available read tickets
    "totalTickets": 128 // Total read tickets
  }
}
```

**Ticket utilization** :
```
Write Ticket Usage = out / totalTickets * 100
Read Ticket Usage = out / totalTickets * 100

Healthy:  < 50%
Warning:  50-80%
Critical: > 80% (concurrency bottleneck)
```

---

## Configuration du Cache

### Dimensionnement du Cache

#### Calcul Recommandé

```
Working Set Analysis:
═══════════════════════════════════════════════════════════════

1. Identifier le Working Set (données fréquemment accédées)
   Working Set = Index Size + Hot Data Size

   Example:
   Index Size: 10 GB (db.collection.stats().totalIndexSize)
   Hot Data: 15 GB (80/20 rule: 20% of data = 80% of queries)
   Working Set = 25 GB

2. Cache Size Recommendation
   Cache Size = Working Set * 1.2-1.5 (buffer)
   Cache Size = 25 GB * 1.3 = 32.5 GB

3. Validate Against RAM
   Total RAM: 64 GB
   Cache Size: 32.5 GB (50.8% of RAM) ✓ Good
   MongoDB Process: ~5 GB
   OS + FS Cache: ~26.5 GB (41% of RAM) ✓ Good

4. Adjust if Needed
   If Working Set > 60% RAM:
     → Scale vertically (add RAM)
     → Scale horizontally (sharding)
     → Optimize queries (reduce working set)
```

#### Configuration Methods

**Méthode 1 : Configuration File**

```yaml
# /etc/mongod.conf

storage:
  wiredTiger:
    engineConfig:
      # Cache size in GB
      cacheSizeGB: 32

      # Alternative: percentage of RAM (minus 1 GB)
      # cacheSizeGB: 0.5  # 50% of RAM - 1 GB

      # Checkpoint interval (seconds)
      # Default: 60
      checkpointSizeMB: 2048

      # Journal compression (default: snappy)
      journalCompressor: snappy

      # Directory size for diagnostics
      directoryForIndexes: false
```

**Méthode 2 : Runtime avec setParameter**

```javascript
// Augmenter le cache (nécessite restart pour prendre effet)
// Note: wiredTigerCacheSizeGB n'est PAS modifiable à chaud

// Autres paramètres modifiables à chaud:
db.adminCommand({
  setParameter: 1,

  // Checkpoint interval (secondes)
  syncdelay: 60,

  // Journal commit interval (ms)
  journalCommitInterval: 100
})
```

**⚠️ Important** : Le cache WiredTiger **ne peut pas être redimensionné à chaud**. Un restart de mongod est nécessaire.

### Configuration Avancée

```yaml
# /etc/mongod.conf - Configuration Production

storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100  # Default: 100ms

  wiredTiger:
    engineConfig:
      # === CACHE CONFIGURATION ===
      cacheSizeGB: 32

      # === CHECKPOINT CONFIGURATION ===
      # Checkpoint every 60 seconds OR when journal reaches 2 GB
      checkpointSizeMB: 2048

      # === COMPRESSION ===
      journalCompressor: snappy    # Options: none, snappy, zlib, zstd

      # === EVICTION CONFIGURATION ===
      # These are internal, not directly configurable but good to know
      # evictionTarget: 80       # Start eviction at 80% cache full
      # evictionTrigger: 95      # Aggressive eviction at 95%
      # evictionDirtyTarget: 5   # Target 5% dirty pages
      # evictionDirtyTrigger: 20 # Force eviction at 20% dirty

    collectionConfig:
      blockCompressor: snappy      # Options: none, snappy, zlib, zstd

    indexConfig:
      prefixCompression: true      # Default: true (saves space)

# === SYSTEM TUNING ===
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

# === NETWORK ===
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

# === OPERATIONS ===
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

---

## Monitoring du Cache

### Script de Monitoring Complet

```python
#!/usr/bin/env python3
# wiredtiger_cache_monitor.py

from pymongo import MongoClient
from datetime import datetime
import time
import json

class WiredTigerCacheMonitor:
    def __init__(self, uri='mongodb://localhost:27017'):
        self.client = MongoClient(uri)
        self.db = self.client.admin
        self.previous_stats = None

    def get_cache_stats(self):
        """Get current cache statistics"""
        status = self.db.command('serverStatus')
        wt = status['wiredTiger']

        return {
            'timestamp': datetime.now().isoformat(),

            # Cache size
            'cache_max_bytes': wt['cache']['maximum bytes configured'],
            'cache_used_bytes': wt['cache']['bytes currently in the cache'],
            'cache_dirty_bytes': wt['cache']['tracked dirty bytes in the cache'],

            # Eviction
            'pages_evicted_total': wt['cache']['pages evicted by application threads'] +
                                  wt['cache']['eviction worker thread evicting pages'],
            'pages_evicted_app': wt['cache']['pages evicted by application threads'],
            'pages_evicted_worker': wt['cache']['eviction worker thread evicting pages'],

            # Page I/O
            'pages_read': wt['cache']['pages read into cache'],
            'pages_written': wt['cache']['pages written from cache'],

            # Checkpoint
            'checkpoint_duration_ms': wt['transaction']['transaction checkpoint most recent time (msecs)'],
            'checkpoints_total': wt['transaction']['transaction checkpoints'],

            # Tickets
            'tickets_read_out': wt['concurrentTransactions']['read']['out'],
            'tickets_read_available': wt['concurrentTransactions']['read']['available'],
            'tickets_write_out': wt['concurrentTransactions']['write']['out'],
            'tickets_write_available': wt['concurrentTransactions']['write']['available']
        }

    def calculate_rates(self, current, previous):
        """Calculate rates between two samples"""
        if not previous:
            return {}

        time_delta = (datetime.fromisoformat(current['timestamp']) -
                     datetime.fromisoformat(previous['timestamp'])).total_seconds()

        return {
            'eviction_rate': (current['pages_evicted_total'] - previous['pages_evicted_total']) / time_delta,
            'read_rate': (current['pages_read'] - previous['pages_read']) / time_delta,
            'write_rate': (current['pages_written'] - previous['pages_written']) / time_delta
        }

    def analyze_cache_health(self, stats):
        """Analyze cache health and return status"""
        health = {
            'status': 'HEALTHY',
            'issues': [],
            'warnings': []
        }

        # Cache usage
        cache_usage_pct = (stats['cache_used_bytes'] / stats['cache_max_bytes']) * 100
        if cache_usage_pct > 95:
            health['issues'].append(f"Cache usage critical: {cache_usage_pct:.1f}%")
            health['status'] = 'CRITICAL'
        elif cache_usage_pct > 85:
            health['warnings'].append(f"Cache usage high: {cache_usage_pct:.1f}%")
            if health['status'] == 'HEALTHY':
                health['status'] = 'WARNING'

        # Dirty pages
        if stats['cache_used_bytes'] > 0:
            dirty_pct = (stats['cache_dirty_bytes'] / stats['cache_used_bytes']) * 100
            if dirty_pct > 40:
                health['issues'].append(f"Dirty cache critical: {dirty_pct:.1f}%")
                health['status'] = 'CRITICAL'
            elif dirty_pct > 20:
                health['warnings'].append(f"Dirty cache high: {dirty_pct:.1f}%")
                if health['status'] == 'HEALTHY':
                    health['status'] = 'WARNING'

        # Application eviction
        if self.previous_stats:
            app_evict_delta = stats['pages_evicted_app'] - self.previous_stats['pages_evicted_app']
            if app_evict_delta > 0:
                health['issues'].append(f"Application thread eviction detected: {app_evict_delta} pages")
                health['status'] = 'CRITICAL'

        # Ticket exhaustion
        tickets_read_pct = (stats['tickets_read_out'] /
                           (stats['tickets_read_out'] + stats['tickets_read_available'])) * 100
        tickets_write_pct = (stats['tickets_write_out'] /
                            (stats['tickets_write_out'] + stats['tickets_write_available'])) * 100

        if tickets_read_pct > 80 or tickets_write_pct > 80:
            health['warnings'].append(f"Ticket pressure: read {tickets_read_pct:.0f}%, write {tickets_write_pct:.0f}%")
            if health['status'] == 'HEALTHY':
                health['status'] = 'WARNING'

        return health

    def format_bytes(self, bytes_val):
        """Format bytes to human readable"""
        for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
            if bytes_val < 1024:
                return f"{bytes_val:.2f} {unit}"
            bytes_val /= 1024
        return f"{bytes_val:.2f} PB"

    def print_report(self, stats, health, rates=None):
        """Print formatted monitoring report"""
        print("\n" + "=" * 80)
        print(f"WiredTiger Cache Monitoring Report - {stats['timestamp']}")
        print("=" * 80)

        # Status
        status_symbol = {
            'HEALTHY': '✓',
            'WARNING': '⚠',
            'CRITICAL': '✗'
        }
        print(f"\nStatus: {status_symbol[health['status']]} {health['status']}\n")

        # Cache overview
        print("CACHE OVERVIEW")
        print("-" * 80)
        cache_usage_pct = (stats['cache_used_bytes'] / stats['cache_max_bytes']) * 100
        dirty_pct = (stats['cache_dirty_bytes'] / stats['cache_used_bytes'] * 100
                    if stats['cache_used_bytes'] > 0 else 0)

        print(f"Cache Size:       {self.format_bytes(stats['cache_max_bytes'])}")
        print(f"Cache Used:       {self.format_bytes(stats['cache_used_bytes'])} ({cache_usage_pct:.2f}%)")
        print(f"Cache Dirty:      {self.format_bytes(stats['cache_dirty_bytes'])} ({dirty_pct:.2f}%)")
        print(f"Cache Clean:      {self.format_bytes(stats['cache_used_bytes'] - stats['cache_dirty_bytes'])}")

        # Eviction
        print("\nEVICTION STATISTICS")
        print("-" * 80)
        total_evict = stats['pages_evicted_total']
        app_evict = stats['pages_evicted_app']
        worker_evict = stats['pages_evicted_worker']

        print(f"Total Evictions:  {total_evict}")
        print(f"  Worker Threads: {worker_evict} ({worker_evict/total_evict*100 if total_evict > 0 else 0:.1f}%)")
        print(f"  App Threads:    {app_evict} ({app_evict/total_evict*100 if total_evict > 0 else 0:.1f}%)")

        if rates:
            print(f"\nEviction Rate:    {rates['eviction_rate']:.2f} pages/sec")
            print(f"Read Rate:        {rates['read_rate']:.2f} pages/sec")
            print(f"Write Rate:       {rates['write_rate']:.2f} pages/sec")

        # Checkpoint
        print("\nCHECKPOINT")
        print("-" * 80)
        print(f"Last Duration:    {stats['checkpoint_duration_ms']} ms")
        print(f"Total Checkpoints: {stats['checkpoints_total']}")

        # Tickets
        print("\nTICKETS")
        print("-" * 80)
        tickets_read_pct = (stats['tickets_read_out'] /
                           (stats['tickets_read_out'] + stats['tickets_read_available'])) * 100
        tickets_write_pct = (stats['tickets_write_out'] /
                            (stats['tickets_write_out'] + stats['tickets_write_available'])) * 100

        print(f"Read Tickets:     {stats['tickets_read_out']} / {stats['tickets_read_out'] + stats['tickets_read_available']} ({tickets_read_pct:.1f}%)")
        print(f"Write Tickets:    {stats['tickets_write_out']} / {stats['tickets_write_out'] + stats['tickets_write_available']} ({tickets_write_pct:.1f}%)")

        # Issues and warnings
        if health['issues']:
            print("\nISSUES")
            print("-" * 80)
            for issue in health['issues']:
                print(f"✗ {issue}")

        if health['warnings']:
            print("\nWARNINGS")
            print("-" * 80)
            for warning in health['warnings']:
                print(f"⚠ {warning}")

        print("\n" + "=" * 80 + "\n")

    def monitor_continuous(self, interval=10):
        """Continuous monitoring loop"""
        print("Starting WiredTiger Cache Monitor...")
        print(f"Monitoring interval: {interval} seconds")
        print("Press Ctrl+C to stop\n")

        try:
            while True:
                current_stats = self.get_cache_stats()
                rates = self.calculate_rates(current_stats, self.previous_stats)
                health = self.analyze_cache_health(current_stats)

                self.print_report(current_stats, health, rates)

                self.previous_stats = current_stats
                time.sleep(interval)

        except KeyboardInterrupt:
            print("\nMonitoring stopped.")

# Usage
if __name__ == '__main__':
    monitor = WiredTigerCacheMonitor('mongodb://localhost:27017')

    # Single snapshot
    # stats = monitor.get_cache_stats()
    # health = monitor.analyze_cache_health(stats)
    # monitor.print_report(stats, health)

    # Continuous monitoring
    monitor.monitor_continuous(interval=10)
```

### Métriques Prometheus

```yaml
# prometheus_wiredtiger_exporter.py
# Export WiredTiger cache metrics to Prometheus

from prometheus_client import Gauge, start_http_server
from pymongo import MongoClient
import time

# Define metrics
cache_max = Gauge('mongodb_wt_cache_max_bytes', 'Maximum cache size in bytes')
cache_used = Gauge('mongodb_wt_cache_used_bytes', 'Cache used in bytes')
cache_dirty = Gauge('mongodb_wt_cache_dirty_bytes', 'Dirty cache in bytes')

cache_usage_pct = Gauge('mongodb_wt_cache_usage_percent', 'Cache usage percentage')
cache_dirty_pct = Gauge('mongodb_wt_cache_dirty_percent', 'Dirty cache percentage')

pages_evicted_app = Gauge('mongodb_wt_pages_evicted_app_total', 'Pages evicted by application threads')
pages_evicted_worker = Gauge('mongodb_wt_pages_evicted_worker_total', 'Pages evicted by worker threads')
pages_read = Gauge('mongodb_wt_pages_read_total', 'Pages read into cache')
pages_written = Gauge('mongodb_wt_pages_written_total', 'Pages written from cache')

checkpoint_duration = Gauge('mongodb_wt_checkpoint_duration_ms', 'Last checkpoint duration in ms')

tickets_read_out = Gauge('mongodb_wt_tickets_read_out', 'Read tickets in use')
tickets_read_available = Gauge('mongodb_wt_tickets_read_available', 'Read tickets available')
tickets_write_out = Gauge('mongodb_wt_tickets_write_out', 'Write tickets in use')
tickets_write_available = Gauge('mongodb_wt_tickets_write_available', 'Write tickets available')

def collect_metrics(client):
    """Collect and export metrics"""
    status = client.admin.command('serverStatus')
    wt = status['wiredTiger']

    # Cache metrics
    max_bytes = wt['cache']['maximum bytes configured']
    used_bytes = wt['cache']['bytes currently in the cache']
    dirty_bytes = wt['cache']['tracked dirty bytes in the cache']

    cache_max.set(max_bytes)
    cache_used.set(used_bytes)
    cache_dirty.set(dirty_bytes)

    cache_usage_pct.set((used_bytes / max_bytes) * 100)
    if used_bytes > 0:
        cache_dirty_pct.set((dirty_bytes / used_bytes) * 100)

    # Eviction metrics
    pages_evicted_app.set(wt['cache']['pages evicted by application threads'])
    pages_evicted_worker.set(wt['cache']['eviction worker thread evicting pages'])
    pages_read.set(wt['cache']['pages read into cache'])
    pages_written.set(wt['cache']['pages written from cache'])

    # Checkpoint
    checkpoint_duration.set(wt['transaction']['transaction checkpoint most recent time (msecs)'])

    # Tickets
    tickets_read_out.set(wt['concurrentTransactions']['read']['out'])
    tickets_read_available.set(wt['concurrentTransactions']['read']['available'])
    tickets_write_out.set(wt['concurrentTransactions']['write']['out'])
    tickets_write_available.set(wt['concurrentTransactions']['write']['available'])

if __name__ == '__main__':
    # Start Prometheus metrics server
    start_http_server(9217)

    # Connect to MongoDB
    client = MongoClient('mongodb://localhost:27017')

    print("WiredTiger Cache Exporter started on port 9217")

    while True:
        try:
            collect_metrics(client)
        except Exception as e:
            print(f"Error collecting metrics: {e}")

        time.sleep(15)  # Collect every 15 seconds
```

---

## Tuning et Optimisation

### Stratégies de Tuning

#### 1. Identification du Working Set

```javascript
// Script pour analyser le working set
// working_set_analyzer.js

function analyzeWorkingSet() {
    print("Analyzing Working Set...\n");

    let totalIndexSize = 0;
    let totalDataSize = 0;
    let collections = [];

    // Iterate through all databases
    db.adminCommand('listDatabases').databases.forEach(function(database) {
        if (database.name === 'admin' || database.name === 'local' || database.name === 'config') {
            return;  // Skip system databases
        }

        let dbConn = db.getSiblingDB(database.name);

        dbConn.getCollectionNames().forEach(function(collName) {
            if (collName.startsWith('system.')) {
                return;  // Skip system collections
            }

            let stats = dbConn[collName].stats();

            collections.push({
                db: database.name,
                collection: collName,
                indexSize: stats.totalIndexSize,
                dataSize: stats.size,
                storageSize: stats.storageSize,
                count: stats.count
            });

            totalIndexSize += stats.totalIndexSize;
            totalDataSize += stats.size;
        });
    });

    // Sort by index size (most important for cache)
    collections.sort((a, b) => b.indexSize - a.indexSize);

    print("=" .repeat(80));
    print("WORKING SET ANALYSIS");
    print("=".repeat(80) + "\n");

    print("SUMMARY");
    print("-".repeat(80));
    print(`Total Index Size:  ${formatBytes(totalIndexSize)}`);
    print(`Total Data Size:   ${formatBytes(totalDataSize)}`);
    print(`Total:             ${formatBytes(totalIndexSize + totalDataSize)}\n`);

    // 80/20 rule estimation
    let hotDataEstimate = totalDataSize * 0.2;
    let estimatedWorkingSet = totalIndexSize + hotDataEstimate;

    print("WORKING SET ESTIMATE (80/20 rule)");
    print("-".repeat(80));
    print(`All Indexes:       ${formatBytes(totalIndexSize)}`);
    print(`Hot Data (20%):    ${formatBytes(hotDataEstimate)}`);
    print(`Working Set:       ${formatBytes(estimatedWorkingSet)}\n`);

    // Recommended cache size
    let recommendedCache = estimatedWorkingSet * 1.3;
    print("RECOMMENDATIONS");
    print("-".repeat(80));
    print(`Recommended Cache: ${formatBytes(recommendedCache)}`);
    print(`  (Working Set × 1.3 buffer)\n`);

    // Top collections
    print("TOP 10 COLLECTIONS BY INDEX SIZE");
    print("-".repeat(80));
    collections.slice(0, 10).forEach(function(coll, idx) {
        print(`${idx + 1}. ${coll.db}.${coll.collection}`);
        print(`   Index Size: ${formatBytes(coll.indexSize)}`);
        print(`   Data Size:  ${formatBytes(coll.dataSize)}`);
        print(`   Documents:  ${coll.count}`);
    });
}

function formatBytes(bytes) {
    if (bytes < 1024) return bytes + " B";
    if (bytes < 1024 * 1024) return (bytes / 1024).toFixed(2) + " KB";
    if (bytes < 1024 * 1024 * 1024) return (bytes / (1024 * 1024)).toFixed(2) + " MB";
    return (bytes / (1024 * 1024 * 1024)).toFixed(2) + " GB";
}

// Run analysis
analyzeWorkingSet();
```

#### 2. Cache Miss Analysis

```python
#!/usr/bin/env python3
# cache_miss_analyzer.py

from pymongo import MongoClient
import time

class CacheMissAnalyzer:
    def __init__(self, uri='mongodb://localhost:27017'):
        self.client = MongoClient(uri)
        self.db = self.client.admin

    def analyze_cache_misses(self, duration=60):
        """Analyze cache miss rate over time"""
        print(f"Analyzing cache misses over {duration} seconds...")

        # Initial snapshot
        initial = self.db.command('serverStatus')
        initial_reads = initial['wiredTiger']['cache']['pages read into cache']
        initial_in_cache = initial['wiredTiger']['cache']['bytes currently in the cache']

        time.sleep(duration)

        # Final snapshot
        final = self.db.command('serverStatus')
        final_reads = final['wiredTiger']['cache']['pages read into cache']
        final_in_cache = final['wiredTiger']['cache']['bytes currently in the cache']

        # Calculate metrics
        pages_read = final_reads - initial_reads
        pages_per_sec = pages_read / duration

        print("\n" + "=" * 80)
        print("CACHE MISS ANALYSIS")
        print("=" * 80 + "\n")

        print(f"Duration: {duration} seconds")
        print(f"Pages Read: {pages_read}")
        print(f"Pages/sec: {pages_per_sec:.2f}")

        # Estimate based on typical page size (4KB)
        bytes_read = pages_read * 4096
        mb_read = bytes_read / (1024 * 1024)
        print(f"Estimated Data Read: {mb_read:.2f} MB")

        # Calculate miss rate (approximate)
        # This is a simplified calculation
        cache_max = final['wiredTiger']['cache']['maximum bytes configured']
        miss_rate_estimate = (mb_read * 1024 * 1024) / cache_max * 100

        print(f"\nCache Miss Rate Estimate: {min(miss_rate_estimate, 100):.2f}%")

        # Recommendations
        print("\nRECOMMENDATIONS")
        print("-" * 80)

        if pages_per_sec > 1000:
            print("⚠ High cache miss rate detected")
            print("  → Consider increasing cache size")
            print("  → Check for full table scans")
            print("  → Review index coverage")
        elif pages_per_sec > 500:
            print("⚠ Moderate cache miss rate")
            print("  → Monitor during peak hours")
            print("  → Review query patterns")
        else:
            print("✓ Cache miss rate is acceptable")

# Usage
analyzer = CacheMissAnalyzer()
analyzer.analyze_cache_misses(duration=60)
```

#### 3. Dirty Page Management

```python
#!/usr/bin/env python3
# dirty_page_optimizer.py

from pymongo import MongoClient
import time

class DirtyPageOptimizer:
    def __init__(self, uri='mongodb://localhost:27017'):
        self.client = MongoClient(uri)
        self.db = self.client.admin

    def analyze_dirty_pages(self):
        """Analyze dirty page patterns"""
        print("Analyzing dirty page behavior...")

        samples = []
        duration = 300  # 5 minutes
        interval = 10   # 10 seconds

        start_time = time.time()

        while time.time() - start_time < duration:
            status = self.db.command('serverStatus')
            wt = status['wiredTiger']

            used_bytes = wt['cache']['bytes currently in the cache']
            dirty_bytes = wt['cache']['tracked dirty bytes in the cache']
            dirty_pct = (dirty_bytes / used_bytes * 100) if used_bytes > 0 else 0

            checkpoint_duration = wt['transaction']['transaction checkpoint most recent time (msecs)']

            samples.append({
                'timestamp': time.time(),
                'dirty_pct': dirty_pct,
                'dirty_bytes': dirty_bytes,
                'checkpoint_duration': checkpoint_duration
            })

            time.sleep(interval)

        # Analysis
        print("\n" + "=" * 80)
        print("DIRTY PAGE ANALYSIS")
        print("=" * 80 + "\n")

        avg_dirty = sum(s['dirty_pct'] for s in samples) / len(samples)
        max_dirty = max(s['dirty_pct'] for s in samples)
        avg_checkpoint = sum(s['checkpoint_duration'] for s in samples) / len(samples)
        max_checkpoint = max(s['checkpoint_duration'] for s in samples)

        print(f"Samples: {len(samples)}")
        print(f"Duration: {duration} seconds")
        print()

        print("DIRTY PAGES")
        print("-" * 80)
        print(f"Average Dirty: {avg_dirty:.2f}%")
        print(f"Max Dirty:     {max_dirty:.2f}%")
        print()

        print("CHECKPOINT PERFORMANCE")
        print("-" * 80)
        print(f"Average Duration: {avg_checkpoint:.0f} ms")
        print(f"Max Duration:     {max_checkpoint:.0f} ms")
        print()

        # Recommendations
        print("RECOMMENDATIONS")
        print("-" * 80)

        if avg_dirty > 20:
            print("⚠ High average dirty percentage")
            print("  → Checkpoint may be lagging behind writes")
            print("  → Consider:")
            print("     • Faster storage (SSD/NVMe)")
            print("     • Reduce write load")
            print("     • Check I/O wait times")

        if max_checkpoint > 10000:  # 10 seconds
            print("⚠ Checkpoint taking too long")
            print("  → I/O bottleneck likely")
            print("  → Check:")
            print("     • Disk I/O metrics (iostat)")
            print("     • Storage backend performance")
            print("     • Reduce dirty target if possible")

        if avg_dirty < 5 and avg_checkpoint < 2000:
            print("✓ Dirty page management is healthy")

# Usage
optimizer = DirtyPageOptimizer()
optimizer.analyze_dirty_pages()
```

---

## Troubleshooting

### Problème 1 : Cache Saturation

**Symptômes** :
```
Cache Usage: > 95%
Application Thread Eviction: > 0
Latency: Elevated
```

**Diagnostic** :

```python
#!/usr/bin/env python3
# diagnose_cache_saturation.py

from pymongo import MongoClient

def diagnose_cache_saturation():
    client = MongoClient('mongodb://localhost:27017')
    status = client.admin.command('serverStatus')
    wt = status['wiredTiger']

    cache_max = wt['cache']['maximum bytes configured']
    cache_used = wt['cache']['bytes currently in the cache']
    cache_usage_pct = (cache_used / cache_max) * 100

    app_evictions = wt['cache']['pages evicted by application threads']
    worker_evictions = wt['cache']['eviction worker thread evicting pages']

    print("CACHE SATURATION DIAGNOSIS")
    print("=" * 80)
    print(f"Cache Usage: {cache_usage_pct:.2f}%")
    print(f"App Thread Evictions: {app_evictions}")
    print(f"Worker Thread Evictions: {worker_evictions}")

    if cache_usage_pct > 95 and app_evictions > 0:
        print("\n✗ CRITICAL: Cache saturation detected")
        print("\nIMMEDIATE ACTIONS:")
        print("1. Check working set size")
        print("2. Identify queries causing cache pressure:")
        print("   db.currentOp({'active': true, 'secs_running': {'$gt': 5}})")
        print("3. Check for full collection scans:")
        print("   db.system.profile.find({'planSummary': /COLLSCAN/})")
        print("\nLONG-TERM SOLUTIONS:")
        print("• Increase cache size (add RAM)")
        print("• Optimize queries (add indexes)")
        print("• Shard to distribute working set")
        print("• Archive old data")
    elif cache_usage_pct > 85:
        print("\n⚠ WARNING: Cache usage high")
        print("Monitor closely and plan capacity expansion")
    else:
        print("\n✓ Cache usage healthy")

diagnose_cache_saturation()
```

**Solutions** :

```bash
# 1. Immediate: Kill long-running queries
mongo --eval "db.currentOp({'active': true, 'secs_running': {\$gt: 60}}).inprog.forEach(function(op) { db.killOp(op.opid); })"

# 2. Short-term: Identify problematic queries
mongo --eval "db.system.profile.find({'millis': {\$gt: 1000}}).sort({'ts': -1}).limit(10).pretty()"

# 3. Long-term: Increase cache size
# Edit /etc/mongod.conf
# storage.wiredTiger.engineConfig.cacheSizeGB: 48  # Increase from 32 to 48
# Then restart mongod
```

### Problème 2 : High Dirty Cache

**Symptômes** :
```
Dirty Cache: > 20%
Checkpoint Duration: > 10 seconds
Write Latency: Elevated
```

**Diagnostic** :

```python
#!/usr/bin/env python3
# diagnose_dirty_cache.py

from pymongo import MongoClient
import subprocess

def diagnose_dirty_cache():
    client = MongoClient('mongodb://localhost:27017')
    status = client.admin.command('serverStatus')
    wt = status['wiredTiger']

    used_bytes = wt['cache']['bytes currently in the cache']
    dirty_bytes = wt['cache']['tracked dirty bytes in the cache']
    dirty_pct = (dirty_bytes / used_bytes * 100) if used_bytes > 0 else 0

    checkpoint_duration = wt['transaction']['transaction checkpoint most recent time (msecs)']

    print("DIRTY CACHE DIAGNOSIS")
    print("=" * 80)
    print(f"Dirty Cache: {dirty_pct:.2f}%")
    print(f"Dirty Bytes: {dirty_bytes / (1024**3):.2f} GB")
    print(f"Last Checkpoint Duration: {checkpoint_duration} ms")

    if dirty_pct > 40:
        print("\n✗ CRITICAL: Dirty cache critical")
        print("\nCHECK I/O PERFORMANCE:")
        print("Run: iostat -x 1 10")

        # Try to get iostat
        try:
            result = subprocess.run(['iostat', '-x', '1', '2'],
                                  capture_output=True, text=True, timeout=5)
            print("\nI/O Statistics:")
            print(result.stdout)
        except:
            print("(iostat not available)")

        print("\nIMMEDIATE ACTIONS:")
        print("1. Check if writes can be throttled temporarily")
        print("2. Verify disk I/O is not saturated (await, %util)")
        print("3. Check checkpoint is not stuck:")
        print("   grep checkpoint /var/log/mongodb/mongod.log | tail -20")

        print("\nLONG-TERM SOLUTIONS:")
        print("• Upgrade to faster storage (SSD → NVMe)")
        print("• Increase I/O capacity")
        print("• Optimize write patterns (bulk writes)")
        print("• Consider write concern adjustments")

    elif dirty_pct > 20:
        print("\n⚠ WARNING: Dirty cache elevated")
        print("Monitor I/O and checkpoint duration")
    else:
        print("\n✓ Dirty cache healthy")

diagnose_dirty_cache()
```

### Problème 3 : Cache Thrashing

**Symptômes** :
```
Cache hit rate: < 90%
Page reads: Very high
Performance: Degraded significantly
```

**Diagnostic** :

```javascript
// diagnose_cache_thrashing.js

function diagnoseCacheThrashing() {
    print("Diagnosing cache thrashing...\n");

    // Get initial stats
    let initial = db.serverStatus().wiredTiger.cache;
    let initialReads = initial['pages read into cache'];

    print("Waiting 60 seconds...\n");
    sleep(60000);

    // Get final stats
    let final = db.serverStatus().wiredTiger.cache;
    let finalReads = final['pages read into cache'];

    let pagesRead = finalReads - initialReads;
    let pagesPerSec = pagesRead / 60;

    print("CACHE THRASHING DIAGNOSIS");
    print("=".repeat(80));
    print(`Pages read in 60s: ${pagesRead}`);
    print(`Pages/sec: ${pagesPerSec.toFixed(2)}`);

    if (pagesPerSec > 1000) {
        print("\n✗ CRITICAL: Cache thrashing detected");
        print("\nLIKELY CAUSES:");
        print("1. Working set larger than cache");
        print("2. Full collection scans");
        print("3. Missing indexes");

        print("\nACTIONS:");
        print("1. Check current operations for COLLSCAN:");
        db.currentOp().inprog.forEach(function(op) {
            if (op.planSummary && op.planSummary.includes('COLLSCAN')) {
                print(`   Found: ${op.ns} - ${op.planSummary}`);
            }
        });

        print("\n2. Check profiler for collection scans:");
        print("   db.system.profile.find({'planSummary': /COLLSCAN/}).limit(5).pretty()");

        print("\n3. Analyze working set:");
        print("   Run working_set_analyzer.js");

    } else if (pagesPerSec > 500) {
        print("\n⚠ WARNING: Elevated cache miss rate");
    } else {
        print("\n✓ Cache performance healthy");
    }
}

diagnoseCacheThrashing();
```

---

## Cas d'Usage Avancés

### 1. Cache Warmup après Restart

```python
#!/usr/bin/env python3
# cache_warmup.py

from pymongo import MongoClient
import time

class CacheWarmer:
    def __init__(self, uri='mongodb://localhost:27017'):
        self.client = MongoClient(uri)

    def warm_cache(self, db_name, collection_name, index_only=False):
        """Warm cache by reading data"""
        print(f"Warming cache for {db_name}.{collection_name}...")

        db = self.client[db_name]
        coll = db[collection_name]

        start_time = time.time()

        if index_only:
            # Read only index by using covered query
            # This is faster and loads indexes into cache
            print("Loading indexes into cache...")

            # Get index keys
            indexes = coll.index_information()
            for idx_name, idx_info in indexes.items():
                if idx_name == '_id_':
                    continue

                key = idx_info['key'][0][0]  # First field of index
                print(f"  Warming index: {idx_name} ({key})")

                # Scan using index
                count = 0
                for doc in coll.find({}, {key: 1, '_id': 0}).hint(idx_name):
                    count += 1
                    if count % 100000 == 0:
                        print(f"    Processed {count} documents...")

                print(f"    Completed: {count} documents")

        else:
            # Full collection scan to load everything
            print("Loading full collection into cache...")

            count = 0
            for doc in coll.find({}):
                count += 1
                if count % 100000 == 0:
                    print(f"  Processed {count} documents...")

            print(f"  Completed: {count} documents")

        duration = time.time() - start_time
        print(f"Cache warmup completed in {duration:.2f} seconds")

    def warm_working_set(self):
        """Warm cache for working set (all indexes + hot data)"""
        print("Warming working set...\n")

        # Get all databases and collections
        for db_name in self.client.list_database_names():
            if db_name in ['admin', 'local', 'config']:
                continue

            db = self.client[db_name]

            for coll_name in db.list_collection_names():
                if coll_name.startswith('system.'):
                    continue

                print(f"\nProcessing {db_name}.{coll_name}")

                # Warm indexes (fast)
                self.warm_cache(db_name, coll_name, index_only=True)

        print("\n✓ Working set warmup completed")

# Usage
warmer = CacheWarmer('mongodb://localhost:27017')

# Warm specific collection
# warmer.warm_cache('mydb', 'mycollection', index_only=True)

# Warm entire working set
warmer.warm_working_set()
```

### 2. Cache Performance Benchmarking

```python
#!/usr/bin/env python3
# cache_benchmark.py

from pymongo import MongoClient
import time
import statistics

class CacheBenchmark:
    def __init__(self, uri='mongodb://localhost:27017'):
        self.client = MongoClient(uri)

    def benchmark_cache_hit_rate(self, db_name, coll_name, query, iterations=1000):
        """Benchmark cache hit rate for specific query"""

        db = self.client[db_name]
        coll = db[coll_name]

        print(f"Benchmarking cache performance...")
        print(f"Query: {query}")
        print(f"Iterations: {iterations}\n")

        # First run to warm cache
        print("Warming cache...")
        coll.find_one(query)

        # Get initial stats
        initial_status = self.client.admin.command('serverStatus')
        initial_reads = initial_status['wiredTiger']['cache']['pages read into cache']

        # Benchmark
        latencies = []
        start_time = time.time()

        for i in range(iterations):
            query_start = time.time()
            coll.find_one(query)
            query_end = time.time()

            latencies.append((query_end - query_start) * 1000)  # ms

        total_time = time.time() - start_time

        # Get final stats
        final_status = self.client.admin.command('serverStatus')
        final_reads = final_status['wiredTiger']['cache']['pages read into cache']

        pages_read = final_reads - initial_reads

        # Calculate hit rate (approximate)
        # If pages_read is low, most queries hit cache
        hit_rate = (1 - (pages_read / iterations)) * 100
        hit_rate = max(0, min(100, hit_rate))  # Clamp to 0-100

        # Results
        print("=" * 80)
        print("BENCHMARK RESULTS")
        print("=" * 80)
        print(f"Total Time: {total_time:.2f} seconds")
        print(f"Throughput: {iterations / total_time:.2f} queries/sec")
        print()

        print("LATENCY")
        print("-" * 80)
        print(f"Min:    {min(latencies):.2f} ms")
        print(f"Max:    {max(latencies):.2f} ms")
        print(f"Mean:   {statistics.mean(latencies):.2f} ms")
        print(f"Median: {statistics.median(latencies):.2f} ms")
        print(f"P95:    {sorted(latencies)[int(len(latencies)*0.95)]:.2f} ms")
        print(f"P99:    {sorted(latencies)[int(len(latencies)*0.99)]:.2f} ms")
        print()

        print("CACHE PERFORMANCE")
        print("-" * 80)
        print(f"Pages Read: {pages_read}")
        print(f"Estimated Hit Rate: {hit_rate:.2f}%")

        if hit_rate > 99:
            print("✓ Excellent cache hit rate")
        elif hit_rate > 95:
            print("✓ Good cache hit rate")
        elif hit_rate > 90:
            print("⚠ Acceptable cache hit rate")
        else:
            print("✗ Poor cache hit rate - investigation needed")

# Usage
benchmark = CacheBenchmark('mongodb://localhost:27017')
benchmark.benchmark_cache_hit_rate(
    'mydb',
    'users',
    {'user_id': 12345},
    iterations=10000
)
```

### 3. Automated Cache Tuning Advisor

```python
#!/usr/bin/env python3
# cache_tuning_advisor.py

from pymongo import MongoClient
import psutil

class CacheTuningAdvisor:
    def __init__(self, uri='mongodb://localhost:27017'):
        self.client = MongoClient(uri)
        self.db = self.client.admin

    def analyze_and_recommend(self):
        """Analyze current configuration and recommend optimizations"""

        print("=" * 80)
        print("CACHE TUNING ADVISOR")
        print("=" * 80 + "\n")

        # Get system info
        total_ram = psutil.virtual_memory().total
        available_ram = psutil.virtual_memory().available

        # Get MongoDB stats
        status = self.db.command('serverStatus')
        wt = status['wiredTiger']

        cache_max = wt['cache']['maximum bytes configured']
        cache_used = wt['cache']['bytes currently in the cache']
        cache_dirty = wt['cache']['tracked dirty bytes in the cache']

        # Calculate working set
        total_index_size = 0
        total_data_size = 0

        for db_name in self.client.list_database_names():
            if db_name in ['admin', 'local', 'config']:
                continue

            db = self.client[db_name]
            for coll_name in db.list_collection_names():
                if coll_name.startswith('system.'):
                    continue

                stats = db[coll_name].stats()
                total_index_size += stats.get('totalIndexSize', 0)
                total_data_size += stats.get('size', 0)

        # Estimate working set (indexes + 20% of data)
        working_set_estimate = total_index_size + (total_data_size * 0.2)

        # Analysis
        print("CURRENT CONFIGURATION")
        print("-" * 80)
        print(f"Total RAM:        {self._format_bytes(total_ram)}")
        print(f"Cache Size:       {self._format_bytes(cache_max)} ({cache_max/total_ram*100:.1f}% of RAM)")
        print(f"Cache Used:       {self._format_bytes(cache_used)} ({cache_used/cache_max*100:.1f}%)")
        print(f"Cache Dirty:      {self._format_bytes(cache_dirty)} ({cache_dirty/cache_used*100:.1f}% of used)")
        print()

        print("WORKING SET ANALYSIS")
        print("-" * 80)
        print(f"Total Index Size: {self._format_bytes(total_index_size)}")
        print(f"Total Data Size:  {self._format_bytes(total_data_size)}")
        print(f"Estimated Working Set: {self._format_bytes(working_set_estimate)}")
        print()

        # Recommendations
        print("RECOMMENDATIONS")
        print("-" * 80)

        recommendations = []

        # Cache size recommendation
        recommended_cache = working_set_estimate * 1.3

        if recommended_cache > cache_max * 1.2:
            recommendations.append({
                'priority': 'HIGH',
                'issue': 'Cache undersized for working set',
                'current': f'Cache: {self._format_bytes(cache_max)}',
                'recommended': f'Increase to {self._format_bytes(recommended_cache)}',
                'action': f'Edit /etc/mongod.conf:\n'
                         f'  storage.wiredTiger.engineConfig.cacheSizeGB: {int(recommended_cache/(1024**3))}'
            })

        # RAM allocation
        if cache_max / total_ram > 0.65:
            recommendations.append({
                'priority': 'MEDIUM',
                'issue': 'Cache using > 65% of RAM',
                'current': f'{cache_max/total_ram*100:.1f}% of RAM',
                'recommended': 'Leave 30-40% RAM for OS and FS cache',
                'action': 'Consider adding more RAM or reducing cache size slightly'
            })

        # Dirty cache
        dirty_pct = (cache_dirty / cache_used * 100) if cache_used > 0 else 0
        if dirty_pct > 20:
            recommendations.append({
                'priority': 'HIGH',
                'issue': 'High dirty cache percentage',
                'current': f'{dirty_pct:.1f}% dirty',
                'recommended': 'Target < 10% dirty',
                'action': 'Check I/O performance:\n'
                         '  iostat -x 1 10\n'
                         '  Consider faster storage (SSD/NVMe)'
            })

        # Working set vs cache
        if working_set_estimate > cache_max * 0.9:
            recommendations.append({
                'priority': 'HIGH',
                'issue': 'Working set close to cache size',
                'current': f'Working set: {self._format_bytes(working_set_estimate)}',
                'recommended': 'Increase cache or reduce working set',
                'action': 'Options:\n'
                         '  1. Increase RAM and cache size\n'
                         '  2. Archive old data\n'
                         '  3. Implement sharding'
            })

        # Print recommendations
        if not recommendations:
            print("✓ Current configuration looks optimal")
        else:
            for i, rec in enumerate(recommendations, 1):
                print(f"\n{i}. [{rec['priority']}] {rec['issue']}")
                print(f"   Current:      {rec['current']}")
                print(f"   Recommended:  {rec['recommended']}")
                print(f"   Action:       {rec['action']}")

        print("\n" + "=" * 80)

    def _format_bytes(self, bytes_val):
        """Format bytes to human readable"""
        for unit in ['B', 'KB', 'MB', 'GB', 'TB']:
            if bytes_val < 1024:
                return f"{bytes_val:.2f} {unit}"
            bytes_val /= 1024
        return f"{bytes_val:.2f} PB"

# Usage
advisor = CacheTuningAdvisor('mongodb://localhost:27017')
advisor.analyze_and_recommend()
```

---

## Bonnes Pratiques

### Checklist Configuration Optimale

```yaml
Production Configuration Checklist:
═══════════════════════════════════════════════════════════════

Hardware:
  ✓ RAM: Sufficient for working set + 30-40% buffer
  ✓ Storage: SSD or NVMe (not HDD)
  ✓ CPU: Adequate cores for workload
  ✓ Network: Low latency (< 10ms intra-DC)

Cache Configuration:
  ✓ Cache size: 50-60% of RAM (not < 256 MB, not > 80% RAM)
  ✓ Working set < 70% cache size
  ✓ OS + FS cache: 30-40% RAM remaining

Monitoring:
  ✓ Cache usage percentage (target: 70-90%)
  ✓ Dirty cache percentage (target: < 5-10%)
  ✓ Application thread evictions (target: 0)
  ✓ Checkpoint duration (target: < 5 seconds)
  ✓ Ticket utilization (target: < 50%)

Operations:
  ✓ Regular working set analysis (monthly)
  ✓ Cache warmup after restarts
  ✓ Capacity planning (quarterly)
  ✓ Performance benchmarking (after changes)

Query Optimization:
  ✓ Proper indexes (eliminate COLLSCAN)
  ✓ Covered queries where possible
  ✓ Avoid full collection scans
  ✓ Use projection to limit data transfer
```

### Configuration par Type de Workload

```yaml
Read-Heavy Workload:
  Cache Size: 60% RAM (maximize cache for reads)
  Dirty Target: Not critical (< 10%)
  Focus: Cache hit rate > 95%
  Optimization: Ensure indexes fit in cache

Write-Heavy Workload:
  Cache Size: 50% RAM (leave room for FS cache)
  Dirty Target: < 5% (aggressive checkpointing)
  Focus: Checkpoint duration < 3 seconds
  Optimization: Fast storage, monitor I/O

Balanced Workload:
  Cache Size: 55% RAM
  Dirty Target: < 10%
  Focus: Overall performance balance
  Optimization: Regular monitoring and tuning

Analytics/Reporting:
  Cache Size: 40-50% RAM
  Dirty Target: Not critical
  Focus: Allow large sequential scans
  Optimization: Dedicated analytics nodes
```

---

## Résumé pour SRE

### Commandes Essentielles

```javascript
// Vérifier cache usage
db.serverStatus().wiredTiger.cache["bytes currently in the cache"] /
db.serverStatus().wiredTiger.cache["maximum bytes configured"] * 100

// Vérifier dirty cache
db.serverStatus().wiredTiger.cache["tracked dirty bytes in the cache"] /
db.serverStatus().wiredTiger.cache["bytes currently in the cache"] * 100

// Vérifier application evictions (should be 0)
db.serverStatus().wiredTiger.cache["pages evicted by application threads"]

// Vérifier checkpoint duration
db.serverStatus().wiredTiger.transaction["transaction checkpoint most recent time (msecs)"]

// Vérifier ticket utilization
db.serverStatus().wiredTiger.concurrentTransactions
```

### Seuils Critiques

```
Cache Usage:
  < 70%:     Underutilized
  70-90%:    ✓ Optimal
  90-95%:    ⚠ Warning
  > 95%:     ✗ Critical (add RAM or reduce working set)

Dirty Cache:
  < 5%:      ✓ Excellent
  5-10%:     ✓ Good
  10-20%:    ⚠ Warning (check I/O)
  20-40%:    ⚠ High (checkpoint pressure)
  > 40%:     ✗ Critical (I/O bottleneck)

App Thread Evictions:
  0:         ✓ Perfect
  > 0:       ✗ Cache saturation (immediate action)

Checkpoint Duration:
  < 3s:      ✓ Excellent
  3-5s:      ✓ Good
  5-15s:     ⚠ Warning
  > 15s:     ✗ Critical (I/O issue)
```

### Troubleshooting Quick Reference

```
Symptom: High latency, cache > 95%
→ Cache saturation
→ Actions: Check working set, add RAM, optimize queries

Symptom: Dirty cache > 20%, slow checkpoints
→ I/O bottleneck
→ Actions: Check iostat, upgrade storage, reduce write load

Symptom: Application evictions > 0
→ Critical cache pressure
→ Actions: Immediate - kill long queries, Long-term - increase cache

Symptom: Cache < 50% used, poor performance
→ Not cache-related, likely query/index issue
→ Actions: Check profiler, review explain plans

Symptom: High cache, low dirty, good performance
→ ✓ Healthy cache state
→ Actions: None, continue monitoring
```

---

## Conclusion

La **gestion optimale du cache WiredTiger** est fondamentale pour maintenir des performances MongoDB élevées et prévisibles. Les points clés à retenir pour les SRE :

1. **Dimensionnement** : Cache = Working Set × 1.3, avec 30-40% RAM pour OS/FS cache
2. **Monitoring** : Cache usage, dirty %, application evictions, checkpoint duration
3. **Tuning** : Basé sur métriques réelles, pas sur des suppositions
4. **Prévention** : Analyse régulière du working set, capacity planning proactif
5. **Réactivité** : Scripts automatisés pour détection et diagnostic rapides

**Actions recommandées** :
- Implémenter le monitoring continu du cache
- Établir les baselines de performance
- Créer des runbooks pour incidents cache
- Planifier les revues trimestrielles de capacité
- Automatiser l'analyse et les alertes

Le cache WiredTiger bien géré = MongoDB performant = Utilisateurs satisfaits = SRE heureux ! 🚀

---

**Références** :
- [WiredTiger Documentation](http://source.wiredtiger.com/develop/index.html)
- [MongoDB WiredTiger Internals](https://www.mongodb.com/docs/manual/core/wiredtiger/)
- [Tuning WiredTiger Cache](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)
- [WiredTiger: A Deep Dive](https://www.mongodb.com/presentations/deep-dive-into-the-wiredtiger-storage-engine)

⏭️ [MongoDB Atlas](/14-mongodb-atlas/README.md)
