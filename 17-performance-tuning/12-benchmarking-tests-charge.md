🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.12 Benchmarking et Tests de Charge

## Introduction

Le benchmarking et les tests de charge sont des composantes essentielles de toute stratégie d'optimisation MongoDB. Ils permettent de valider les améliorations de performance, d'identifier les limites du système, et de garantir la stabilité sous charge avant le déploiement en production. Sans tests rigoureux, les optimisations restent théoriques et les surprises en production sont inévitables.

Cette section explore les méthodologies de benchmarking MongoDB, les outils de test de charge (YCSB, custom frameworks), les scénarios de test critiques, et les techniques d'analyse des résultats pour prendre des décisions d'optimisation basées sur des données quantifiables.

Un benchmarking bien mené permet d'identifier des gains de 2-10× sur les métriques critiques et d'éviter des incidents de production coûteux.

## Méthodologie de Benchmarking

### Principes Fondamentaux

**Reproductibilité** :
```yaml
Requirements:
  - Environnement identique (hardware, OS, versions)
  - Dataset représentatif (taille, distribution)
  - Workload réaliste (mix read/write, patterns)
  - Conditions stables (pas d'autres charges)
  - Mesures répétées (minimum 3 runs)
  - Warm-up period (éviter cold cache)
```

**Isolation** :
```
✅ BON : Environnement dédié
- Hardware dédié aux tests
- Pas d'autres applications
- Network isolé
- Monitoring minimal (overhead < 1%)

❌ MAUVAIS : Environnement partagé
- Serveur avec autres apps
- Network congestionné
- Résultats variables et non-fiables
```

### Phases de Benchmarking

```
┌─────────────────────────────────────────────────────┐
│              BENCHMARK LIFECYCLE                    │
│                                                     │
│  1. BASELINE                                        │
│     └─ Mesure performance actuelle                  │
│     └─ Identifier bottlenecks                       │
│                                                     │
│  2. HYPOTHESIS                                      │
│     └─ Optimisation à tester                        │
│     └─ Prédiction du gain                           │
│                                                     │
│  3. IMPLEMENTATION                                  │
│     └─ Appliquer changements                        │
│     └─ Documenter modifications                     │
│                                                     │
│  4. TEST                                            │
│     └─ Exécuter benchmark identique                 │
│     └─ Mesures multiples (n≥3)                      │
│                                                     │
│  5. ANALYSIS                                        │
│     └─ Comparer baseline vs optimisé                │
│     └─ Valider amélioration statistique             │
│                                                     │
│  6. DECISION                                        │
│     └─ Déployer si gain significatif (>10%)         │
│     └─ Rollback si dégradation                      │
│                                                     │
│  7. PRODUCTION VALIDATION                           │
│     └─ Monitor post-déploiement                     │
│     └─ Confirmer gains en production                │
└─────────────────────────────────────────────────────┘
```

### Métriques Clés

**Performance metrics** :
```javascript
const performanceMetrics = {
  // Latency (percentiles critiques)
  latency: {
    p50: "ms",   // Médiane
    p95: "ms",   // 95e percentile
    p99: "ms",   // 99e percentile
    p999: "ms",  // 99.9e percentile
    max: "ms"    // Pire cas
  },

  // Throughput
  throughput: {
    operations: "ops/sec",
    reads: "reads/sec",
    writes: "writes/sec",
    bandwidth: "MB/sec"
  },

  // Resources
  resources: {
    cpu: "percent",
    memory: "GB",
    diskIO: "IOPS",
    diskBandwidth: "MB/sec",
    network: "MB/sec"
  },

  // Errors
  errors: {
    timeouts: "count",
    connectionFailures: "count",
    writeErrors: "count",
    readErrors: "count"
  },

  // MongoDB-specific
  mongodb: {
    cacheHitRatio: "percent",
    replicationLag: "seconds",
    queueDepth: "count",
    connections: "count"
  }
};
```

**Cibles de performance** :

| Metric | Excellent | Good | Acceptable | Poor |
|--------|-----------|------|------------|------|
| **P99 Latency** | <10ms | <50ms | <100ms | >100ms |
| **P50 Latency** | <2ms | <10ms | <20ms | >20ms |
| **Throughput** | >10K ops/s | >5K ops/s | >1K ops/s | <1K ops/s |
| **Error Rate** | <0.01% | <0.1% | <1% | >1% |
| **Cache Hit** | >99% | >95% | >90% | <90% |

## Outils de Benchmarking

### YCSB (Yahoo Cloud Serving Benchmark)

**Installation et configuration** :
```bash
# Installation
git clone https://github.com/brianfrankcooper/YCSB.git
cd YCSB
mvn clean package

# Configuration MongoDB
cat > mongodb.properties <<EOF
mongodb.url=mongodb://localhost:27017/ycsb
mongodb.writeConcern=acknowledged
mongodb.readPreference=primaryPreferred
mongodb.maxconnections=100
EOF
```

**Workloads standards** :
```bash
# Workload A : Update heavy (50% read, 50% update)
# Simule cache/session store
./bin/ycsb run mongodb -s -P workloads/workloada \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  -p operationcount=1000000 \
  -p threadcount=32 \
  -p target=10000

# Workload B : Read mostly (95% read, 5% update)
# Simule tag/metadata server
./bin/ycsb run mongodb -s -P workloads/workloadb \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  -p operationcount=1000000 \
  -p threadcount=32

# Workload C : Read only (100% read)
# Simule cache read-only
./bin/ycsb run mongodb -s -P workloads/workloadc \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  -p operationcount=1000000 \
  -p threadcount=32

# Workload D : Read latest (95% read latest, 5% insert)
# Simule user timeline
./bin/ycsb run mongodb -s -P workloads/workloadd \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  -p operationcount=1000000 \
  -p threadcount=32

# Workload E : Short ranges (95% scan, 5% insert)
# Simule threaded conversations
./bin/ycsb run mongodb -s -P workloads/workloade \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  -p operationcount=1000000 \
  -p threadcount=32 \
  -p maxscanlength=100

# Workload F : Read-modify-write (50% read, 50% read-modify-write)
# Simule user database
./bin/ycsb run mongodb -s -P workloads/workloadf \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  -p operationcount=1000000 \
  -p threadcount=32
```

**Analyse des résultats YCSB** :
```bash
# Output example:
[OVERALL], RunTime(ms), 45231
[OVERALL], Throughput(ops/sec), 22106.89
[READ], Operations, 475123
[READ], AverageLatency(us), 891.2
[READ], MinLatency(us), 234
[READ], MaxLatency(us), 45678
[READ], 95thPercentileLatency(us), 2345
[READ], 99thPercentileLatency(us), 4567
[UPDATE], Operations, 524877
[UPDATE], AverageLatency(us), 1234.5
[UPDATE], 95thPercentileLatency(us), 3456
[UPDATE], 99thPercentileLatency(us), 6789
```

**Parser les résultats** :
```javascript
// Script Node.js pour parser YCSB output
const fs = require('fs');

function parseYCSBResults(filename) {
  const content = fs.readFileSync(filename, 'utf8');
  const lines = content.split('\n');

  const results = {
    overall: {},
    operations: {}
  };

  lines.forEach(line => {
    const match = line.match(/\[(.*?)\], (.*?), (.*)/);
    if (match) {
      const [, category, metric, value] = match;

      if (category === 'OVERALL') {
        results.overall[metric] = parseFloat(value);
      } else {
        if (!results.operations[category]) {
          results.operations[category] = {};
        }
        results.operations[category][metric] = parseFloat(value);
      }
    }
  });

  // Calculate summary
  results.summary = {
    throughput: results.overall['Throughput(ops/sec)'].toFixed(2),
    runtime: (results.overall['RunTime(ms)'] / 1000).toFixed(2) + 's',

    readLatencyP99: results.operations.READ ?
      (results.operations.READ['99thPercentileLatency(us)'] / 1000).toFixed(2) + 'ms' :
      'N/A',

    writeLatencyP99: results.operations.UPDATE ?
      (results.operations.UPDATE['99thPercentileLatency(us)'] / 1000).toFixed(2) + 'ms' :
      'N/A'
  };

  return results;
}

// Usage
const baseline = parseYCSBResults('baseline.txt');
const optimized = parseYCSBResults('optimized.txt');

console.log('COMPARISON:');
console.log('Throughput:',
  ((optimized.overall['Throughput(ops/sec)'] / baseline.overall['Throughput(ops/sec)'] - 1) * 100).toFixed(2) + '% improvement');
```

### Custom Benchmark Framework

**Framework complet en Node.js** :
```javascript
// benchmark-framework.js
const { MongoClient } = require('mongodb');
const { performance } = require('perf_hooks');

class MongoBenchmark {
  constructor(uri, options = {}) {
    this.uri = uri;
    this.client = null;
    this.db = null;

    // Configuration
    this.config = {
      warmupOperations: options.warmup || 1000,
      totalOperations: options.operations || 10000,
      concurrency: options.concurrency || 10,
      reportInterval: options.reportInterval || 1000
    };

    // Metrics
    this.metrics = {
      operations: 0,
      errors: 0,
      latencies: [],
      start: null,
      end: null
    };
  }

  async connect() {
    this.client = await MongoClient.connect(this.uri, {
      maxPoolSize: this.config.concurrency * 2
    });
    this.db = this.client.db();
  }

  async warmup() {
    console.log(`Warming up with ${this.config.warmupOperations} operations...`);

    const promises = [];
    for (let i = 0; i < this.config.warmupOperations; i++) {
      promises.push(this.executeOperation());

      if (promises.length >= this.config.concurrency) {
        await Promise.all(promises);
        promises.length = 0;
      }
    }

    if (promises.length > 0) {
      await Promise.all(promises);
    }

    console.log('Warmup complete');
  }

  async run() {
    console.log(`Starting benchmark: ${this.config.totalOperations} operations with concurrency ${this.config.concurrency}`);

    this.metrics.start = performance.now();

    const promises = [];
    let completed = 0;

    // Progress reporting
    const progressInterval = setInterval(() => {
      const elapsed = (performance.now() - this.metrics.start) / 1000;
      const opsPerSec = completed / elapsed;
      console.log(`Progress: ${completed}/${this.config.totalOperations} ops (${opsPerSec.toFixed(0)} ops/sec)`);
    }, this.config.reportInterval);

    // Execute operations
    for (let i = 0; i < this.config.totalOperations; i++) {
      const promise = this.executeOperation()
        .then(() => { completed++; });

      promises.push(promise);

      // Maintain concurrency level
      if (promises.length >= this.config.concurrency) {
        await Promise.race(promises);
        promises.splice(promises.findIndex(p => p.isFulfilled), 1);
      }
    }

    // Wait for remaining
    await Promise.all(promises);

    clearInterval(progressInterval);

    this.metrics.end = performance.now();

    return this.calculateResults();
  }

  async executeOperation() {
    const start = performance.now();

    try {
      // Override in subclass with specific workload
      await this.workload();

      const latency = performance.now() - start;
      this.metrics.latencies.push(latency);
      this.metrics.operations++;
    } catch (error) {
      this.metrics.errors++;
      console.error('Operation error:', error.message);
    }
  }

  async workload() {
    // Override this method in subclass
    throw new Error('workload() must be implemented');
  }

  calculateResults() {
    const duration = (this.metrics.end - this.metrics.start) / 1000;
    const throughput = this.metrics.operations / duration;

    // Sort latencies for percentiles
    const sorted = this.metrics.latencies.sort((a, b) => a - b);

    const percentile = (p) => {
      const index = Math.ceil((p / 100) * sorted.length) - 1;
      return sorted[index];
    };

    return {
      duration: duration.toFixed(2) + 's',
      operations: this.metrics.operations,
      errors: this.metrics.errors,
      errorRate: (this.metrics.errors / this.metrics.operations * 100).toFixed(4) + '%',
      throughput: throughput.toFixed(2) + ' ops/sec',

      latency: {
        min: sorted[0].toFixed(2) + 'ms',
        max: sorted[sorted.length - 1].toFixed(2) + 'ms',
        mean: (sorted.reduce((a, b) => a + b) / sorted.length).toFixed(2) + 'ms',
        p50: percentile(50).toFixed(2) + 'ms',
        p95: percentile(95).toFixed(2) + 'ms',
        p99: percentile(99).toFixed(2) + 'ms',
        p999: percentile(99.9).toFixed(2) + 'ms'
      }
    };
  }

  async cleanup() {
    if (this.client) {
      await this.client.close();
    }
  }
}

// Benchmark spécifique : Read-heavy workload
class ReadHeavyBenchmark extends MongoBenchmark {
  async workload() {
    const userId = Math.floor(Math.random() * 1000000);

    // 90% reads
    if (Math.random() < 0.9) {
      await this.db.collection('users').findOne({ _id: userId });
    } else {
      // 10% writes
      await this.db.collection('users').updateOne(
        { _id: userId },
        { $inc: { viewCount: 1 } },
        { upsert: true }
      );
    }
  }
}

// Benchmark spécifique : Write-heavy workload
class WriteHeavyBenchmark extends MongoBenchmark {
  async workload() {
    const userId = Math.floor(Math.random() * 1000000);

    // 70% writes
    if (Math.random() < 0.7) {
      await this.db.collection('events').insertOne({
        userId: userId,
        event: 'action',
        timestamp: new Date(),
        data: { value: Math.random() }
      });
    } else {
      // 30% reads
      await this.db.collection('users').findOne({ _id: userId });
    }
  }
}

// Usage
async function runBenchmark() {
  const benchmark = new ReadHeavyBenchmark('mongodb://localhost:27017/benchmark', {
    warmup: 1000,
    operations: 100000,
    concurrency: 50
  });

  try {
    await benchmark.connect();
    await benchmark.warmup();
    const results = await benchmark.run();

    console.log('\n=== BENCHMARK RESULTS ===');
    console.log(JSON.stringify(results, null, 2));
  } finally {
    await benchmark.cleanup();
  }
}

runBenchmark();
```

### mgodatagen - Realistic Dataset Generation

```bash
# Installation
go install github.com/feliixx/mgodatagen@latest

# Configuration JSON
cat > config.json <<EOF
[
  {
    "database": "benchmark",
    "collection": "users",
    "count": 1000000,
    "content": {
      "_id": {
        "type": "autoincrement",
        "startInt": 1
      },
      "username": {
        "type": "string",
        "minLength": 5,
        "maxLength": 15
      },
      "email": {
        "type": "string",
        "faker": "email"
      },
      "age": {
        "type": "int",
        "minInt": 18,
        "maxInt": 80
      },
      "country": {
        "type": "string",
        "faker": "country"
      },
      "registeredAt": {
        "type": "date",
        "startDate": "2020-01-01T00:00:00Z",
        "endDate": "2025-01-01T00:00:00Z"
      },
      "tags": {
        "type": "array",
        "size": 5,
        "arrayContent": {
          "type": "string",
          "minLength": 3,
          "maxLength": 10
        }
      },
      "profile": {
        "type": "object",
        "objectContent": {
          "bio": {
            "type": "string",
            "minLength": 50,
            "maxLength": 200
          },
          "avatar": {
            "type": "string",
            "faker": "imageURL"
          }
        }
      }
    }
  }
]
EOF

# Génération des données
mgodatagen -f config.json

# Résultat : 1M users avec distribution réaliste
```

## Scénarios de Test

### Baseline Test

**Objectif** : Établir la performance actuelle comme référence.

```bash
# Script de baseline complet
#!/bin/bash

# baseline-test.sh

echo "=== MongoDB Baseline Performance Test ==="
echo "Date: $(date)"
echo "Host: $(hostname)"
echo ""

# 1. System info
echo "## System Information"
echo "CPU: $(lscpu | grep 'Model name' | cut -d: -f2 | xargs)"
echo "RAM: $(free -h | grep Mem | awk '{print $2}')"
echo "Disk: $(df -h /var/lib/mongodb | tail -1 | awk '{print $2}')"
echo ""

# 2. MongoDB info
echo "## MongoDB Configuration"
mongo --eval "
  const status = db.serverStatus();
  print('Version: ' + status.version);
  print('Storage Engine: ' + status.storageEngine.name);
  print('Cache Size: ' + (status.wiredTiger.cache['maximum bytes configured'] / 1024 / 1024 / 1024).toFixed(2) + ' GB');
"
echo ""

# 3. Run YCSB baseline
echo "## Running YCSB Workload A (Baseline)"
./ycsb load mongodb -s -P workloads/workloada \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  > baseline-load.txt

./ycsb run mongodb -s -P workloads/workloada \
  -p mongodb.url=mongodb://localhost:27017/ycsb \
  -p recordcount=1000000 \
  -p operationcount=1000000 \
  -p threadcount=32 \
  > baseline-run.txt

# 4. Extract key metrics
echo "## Key Metrics (Baseline)"
grep "Throughput" baseline-run.txt
grep "95thPercentile" baseline-run.txt | grep READ
grep "99thPercentile" baseline-run.txt | grep READ
grep "95thPercentile" baseline-run.txt | grep UPDATE
grep "99thPercentile" baseline-run.txt | grep UPDATE

echo ""
echo "=== Baseline Complete ==="
echo "Results saved to baseline-*.txt"
```

### Stress Test

**Objectif** : Identifier le point de rupture du système.

```javascript
// stress-test.js
const { MongoClient } = require('mongodb');

class StressTest {
  constructor(uri) {
    this.uri = uri;
    this.results = [];
  }

  async runAtLoad(opsPerSec) {
    console.log(`\nTesting at ${opsPerSec} ops/sec...`);

    const client = await MongoClient.connect(this.uri);
    const db = client.db('stress');

    const startTime = Date.now();
    const duration = 60000; // 1 minute
    const interval = 1000 / opsPerSec; // ms between ops

    let operations = 0;
    let errors = 0;
    const latencies = [];

    while (Date.now() - startTime < duration) {
      const opStart = Date.now();

      try {
        await db.collection('test').insertOne({
          timestamp: new Date(),
          data: Math.random()
        });

        operations++;
        latencies.push(Date.now() - opStart);
      } catch (error) {
        errors++;
      }

      // Throttle to target ops/sec
      const elapsed = Date.now() - opStart;
      if (elapsed < interval) {
        await new Promise(resolve => setTimeout(resolve, interval - elapsed));
      }
    }

    await client.close();

    // Calculate metrics
    const sorted = latencies.sort((a, b) => a - b);
    const p99 = sorted[Math.floor(sorted.length * 0.99)];

    const result = {
      targetOpsPerSec: opsPerSec,
      actualOpsPerSec: operations / (duration / 1000),
      errorRate: (errors / operations * 100).toFixed(2) + '%',
      p99Latency: p99 + 'ms',
      status: p99 < 100 && errors === 0 ? '✅ OK' : '⚠️ DEGRADED'
    };

    this.results.push(result);
    console.log(result);

    return result;
  }

  async findBreakingPoint() {
    console.log('=== STRESS TEST: Finding Breaking Point ===\n');

    // Start low and increase
    const loads = [1000, 2000, 5000, 10000, 15000, 20000, 25000, 30000];

    for (const load of loads) {
      const result = await this.runAtLoad(load);

      // Stop if system degraded
      if (result.status === '⚠️ DEGRADED') {
        console.log(`\n🔴 Breaking point found at ~${load} ops/sec`);
        break;
      }
    }

    console.log('\n=== STRESS TEST COMPLETE ===');
    console.log('Results:');
    console.table(this.results);
  }
}

// Run
const test = new StressTest('mongodb://localhost:27017');
test.findBreakingPoint();
```

### Endurance Test (Soak Test)

**Objectif** : Vérifier la stabilité sur longue durée.

```javascript
// endurance-test.js
class EnduranceTest {
  constructor(uri, durationHours = 24) {
    this.uri = uri;
    this.duration = durationHours * 3600 * 1000;
    this.snapshots = [];
  }

  async run() {
    console.log(`Starting ${this.duration / 3600000}h endurance test...`);

    const client = await MongoClient.connect(this.uri);
    const db = client.db('endurance');

    const startTime = Date.now();
    const snapshotInterval = 600000; // 10 minutes
    let lastSnapshot = startTime;

    while (Date.now() - startTime < this.duration) {
      // Mixed workload
      await this.executeWorkload(db);

      // Take snapshot every 10 minutes
      if (Date.now() - lastSnapshot > snapshotInterval) {
        await this.takeSnapshot(client);
        lastSnapshot = Date.now();
      }
    }

    await client.close();

    this.analyzeResults();
  }

  async executeWorkload(db) {
    const ops = Math.floor(Math.random() * 3);

    switch (ops) {
      case 0:
        await db.collection('data').insertOne({ ts: new Date() });
        break;
      case 1:
        await db.collection('data').findOne({ ts: { $lt: new Date() } });
        break;
      case 2:
        await db.collection('data').updateOne(
          { ts: { $lt: new Date() } },
          { $set: { updated: new Date() } }
        );
        break;
    }
  }

  async takeSnapshot(client) {
    const serverStatus = await client.db().admin().serverStatus();
    const cache = serverStatus.wiredTiger.cache;

    const snapshot = {
      timestamp: new Date(),
      cacheUsage: (cache["bytes currently in the cache"] /
                   cache["maximum bytes configured"] * 100).toFixed(2),
      connections: serverStatus.connections.current,
      operations: serverStatus.opcounters.insert +
                  serverStatus.opcounters.query +
                  serverStatus.opcounters.update
    };

    this.snapshots.push(snapshot);
    console.log('Snapshot:', snapshot);
  }

  analyzeResults() {
    console.log('\n=== ENDURANCE TEST ANALYSIS ===');

    // Check for memory leaks
    const cacheUsages = this.snapshots.map(s => parseFloat(s.cacheUsage));
    const trend = this.calculateTrend(cacheUsages);

    if (trend > 5) {
      console.log('⚠️ WARNING: Cache usage trending up (+' + trend.toFixed(2) + '%)');
      console.log('   Possible memory leak or growth pattern');
    } else {
      console.log('✅ Cache usage stable');
    }

    // Check for connection leaks
    const connections = this.snapshots.map(s => s.connections);
    const connTrend = this.calculateTrend(connections);

    if (connTrend > 10) {
      console.log('⚠️ WARNING: Connections trending up (+' + connTrend.toFixed(2) + '%)');
      console.log('   Possible connection leak');
    } else {
      console.log('✅ Connections stable');
    }

    console.log('\nSnapshots collected:', this.snapshots.length);
  }

  calculateTrend(values) {
    const first = values.slice(0, 5).reduce((a, b) => a + b) / 5;
    const last = values.slice(-5).reduce((a, b) => a + b) / 5;
    return (last - first) / first * 100;
  }
}

// Run 24h test
const test = new EnduranceTest('mongodb://localhost:27017', 24);
test.run();
```

### Spike Test

**Objectif** : Tester le comportement lors de pics soudains.

```javascript
// spike-test.js
class SpikeTest {
  async run() {
    console.log('=== SPIKE TEST ===\n');

    const client = await MongoClient.connect(this.uri);
    const db = client.db('spike');

    // Phase 1 : Normal load (5 minutes)
    console.log('Phase 1: Normal load (1000 ops/sec)');
    await this.runLoad(db, 1000, 300000);

    // Phase 2 : Spike (1 minute)
    console.log('\nPhase 2: SPIKE! (10000 ops/sec)');
    const spikeMetrics = await this.runLoad(db, 10000, 60000);

    // Phase 3 : Recovery (5 minutes)
    console.log('\nPhase 3: Recovery (1000 ops/sec)');
    const recoveryMetrics = await this.runLoad(db, 1000, 300000);

    await client.close();

    // Analysis
    console.log('\n=== SPIKE TEST ANALYSIS ===');
    console.log('Spike performance:', spikeMetrics);
    console.log('Recovery performance:', recoveryMetrics);

    if (recoveryMetrics.p99 < spikeMetrics.p99 * 1.2) {
      console.log('✅ System recovered well from spike');
    } else {
      console.log('⚠️ System showing degradation after spike');
    }
  }

  async runLoad(db, opsPerSec, duration) {
    const startTime = Date.now();
    const latencies = [];

    while (Date.now() - startTime < duration) {
      const opStart = Date.now();

      await db.collection('test').insertOne({
        timestamp: new Date(),
        data: Math.random()
      });

      latencies.push(Date.now() - opStart);

      // Throttle
      const interval = 1000 / opsPerSec;
      const elapsed = Date.now() - opStart;
      if (elapsed < interval) {
        await new Promise(resolve => setTimeout(resolve, interval - elapsed));
      }
    }

    const sorted = latencies.sort((a, b) => a - b);
    return {
      p50: sorted[Math.floor(sorted.length * 0.5)],
      p99: sorted[Math.floor(sorted.length * 0.99)]
    };
  }
}

const test = new SpikeTest('mongodb://localhost:27017');
test.run();
```

## Analyse Comparative

### A/B Testing Framework

```javascript
// ab-test.js
class ABTestFramework {
  constructor() {
    this.results = {
      baseline: null,
      optimized: null
    };
  }

  async runBenchmark(uri, label) {
    console.log(`\n=== Running benchmark: ${label} ===`);

    const benchmark = new MongoBenchmark(uri, {
      warmup: 1000,
      operations: 50000,
      concurrency: 20
    });

    await benchmark.connect();
    await benchmark.warmup();
    const results = await benchmark.run();
    await benchmark.cleanup();

    return results;
  }

  async compare(baselineUri, optimizedUri) {
    // Run baseline
    this.results.baseline = await this.runBenchmark(baselineUri, 'BASELINE');

    // Run optimized
    this.results.optimized = await this.runBenchmark(optimizedUri, 'OPTIMIZED');

    // Compare
    this.generateReport();
  }

  generateReport() {
    console.log('\n=== A/B TEST COMPARISON REPORT ===\n');

    const baseline = this.results.baseline;
    const optimized = this.results.optimized;

    // Throughput comparison
    const baselineThroughput = parseFloat(baseline.throughput);
    const optimizedThroughput = parseFloat(optimized.throughput);
    const throughputImprovement = ((optimizedThroughput - baselineThroughput) / baselineThroughput * 100);

    console.log('## Throughput');
    console.log(`Baseline:  ${baseline.throughput}`);
    console.log(`Optimized: ${optimized.throughput}`);
    console.log(`Change:    ${throughputImprovement > 0 ? '+' : ''}${throughputImprovement.toFixed(2)}%`);
    console.log(this.getVerdict(throughputImprovement, 'higher'));

    // Latency comparison
    console.log('\n## Latency (P99)');
    const baselineP99 = parseFloat(baseline.latency.p99);
    const optimizedP99 = parseFloat(optimized.latency.p99);
    const latencyImprovement = ((baselineP99 - optimizedP99) / baselineP99 * 100);

    console.log(`Baseline:  ${baseline.latency.p99}`);
    console.log(`Optimized: ${optimized.latency.p99}`);
    console.log(`Change:    ${latencyImprovement > 0 ? '-' : '+'}${Math.abs(latencyImprovement).toFixed(2)}%`);
    console.log(this.getVerdict(latencyImprovement, 'lower'));

    // Error rate comparison
    console.log('\n## Error Rate');
    console.log(`Baseline:  ${baseline.errorRate}`);
    console.log(`Optimized: ${optimized.errorRate}`);

    // Overall verdict
    console.log('\n## OVERALL VERDICT');
    if (throughputImprovement > 10 && latencyImprovement > 10) {
      console.log('✅ DEPLOY: Significant improvements on all metrics');
    } else if (throughputImprovement > 0 && latencyImprovement > 0) {
      console.log('✅ DEPLOY: Improvements detected');
    } else if (throughputImprovement < -5 || latencyImprovement < -5) {
      console.log('🔴 ROLLBACK: Performance degradation detected');
    } else {
      console.log('⚠️ INCONCLUSIVE: Marginal differences, more testing needed');
    }
  }

  getVerdict(improvement, preferredDirection) {
    const threshold = 10; // 10% minimum for significance

    if (preferredDirection === 'higher') {
      if (improvement > threshold) return '✅ Significant improvement';
      if (improvement > 0) return '↗️ Minor improvement';
      if (improvement > -threshold) return '➡️ Negligible change';
      return '🔴 Degradation';
    } else {
      if (improvement > threshold) return '✅ Significant improvement';
      if (improvement > 0) return '↗️ Minor improvement';
      if (improvement > -threshold) return '➡️ Negligible change';
      return '🔴 Degradation';
    }
  }
}

// Usage
const abTest = new ABTestFramework();
abTest.compare(
  'mongodb://baseline-server:27017',
  'mongodb://optimized-server:27017'
);
```

### Statistical Significance

```javascript
// statistical-analysis.js
class StatisticalAnalysis {
  // T-test pour comparer deux échantillons
  tTest(sample1, sample2) {
    const mean1 = sample1.reduce((a, b) => a + b) / sample1.length;
    const mean2 = sample2.reduce((a, b) => a + b) / sample2.length;

    const variance1 = sample1.reduce((sum, val) =>
      sum + Math.pow(val - mean1, 2), 0) / (sample1.length - 1);
    const variance2 = sample2.reduce((sum, val) =>
      sum + Math.pow(val - mean2, 2), 0) / (sample2.length - 1);

    const pooledStdDev = Math.sqrt(
      (variance1 / sample1.length) + (variance2 / sample2.length)
    );

    const tStatistic = (mean1 - mean2) / pooledStdDev;

    return {
      tStatistic: tStatistic,
      mean1: mean1,
      mean2: mean2,
      difference: mean2 - mean1,
      percentChange: ((mean2 - mean1) / mean1 * 100).toFixed(2) + '%',

      // Simplified p-value interpretation
      isSignificant: Math.abs(tStatistic) > 2, // p < 0.05 approximation
      confidence: Math.abs(tStatistic) > 2 ? '95%+' : '<95%'
    };
  }

  analyzePerformanceChange(baselineResults, optimizedResults) {
    console.log('=== STATISTICAL ANALYSIS ===\n');

    // Extract latency arrays
    const baselineLatencies = baselineResults.map(r => r.latency);
    const optimizedLatencies = optimizedResults.map(r => r.latency);

    const analysis = this.tTest(baselineLatencies, optimizedLatencies);

    console.log('T-Test Results:');
    console.log(`  Baseline mean: ${analysis.mean1.toFixed(2)}ms`);
    console.log(`  Optimized mean: ${analysis.mean2.toFixed(2)}ms`);
    console.log(`  Difference: ${analysis.difference.toFixed(2)}ms (${analysis.percentChange})`);
    console.log(`  T-statistic: ${analysis.tStatistic.toFixed(4)}`);
    console.log(`  Statistically significant: ${analysis.isSignificant ? 'YES' : 'NO'}`);
    console.log(`  Confidence level: ${analysis.confidence}`);

    if (analysis.isSignificant && analysis.difference < 0) {
      console.log('\n✅ Performance improvement is statistically significant');
    } else if (analysis.isSignificant && analysis.difference > 0) {
      console.log('\n🔴 Performance degradation is statistically significant');
    } else {
      console.log('\n⚠️ Difference not statistically significant - more data needed');
    }

    return analysis;
  }
}

// Usage
const stats = new StatisticalAnalysis();
const analysis = stats.analyzePerformanceChange(baselineRuns, optimizedRuns);
```

## Production-Like Testing

### Shadow Traffic

**Concept** : Dupliquer le traffic production vers environnement de test.

```javascript
// shadow-proxy.js
const { MongoClient } = require('mongodb');
const express = require('express');

class ShadowProxy {
  constructor(productionUri, shadowUri) {
    this.prodClient = new MongoClient(productionUri);
    this.shadowClient = new MongoClient(shadowUri);
    this.metrics = {
      requests: 0,
      shadowErrors: 0,
      shadowSlower: 0
    };
  }

  async proxyOperation(operation) {
    this.metrics.requests++;

    // Execute on production (primary)
    const prodStart = Date.now();
    const prodResult = await operation(this.prodClient.db());
    const prodLatency = Date.now() - prodStart;

    // Execute on shadow (secondary, non-blocking)
    this.executeShadow(operation, prodLatency).catch(err => {
      this.metrics.shadowErrors++;
    });

    // Return production result immediately
    return prodResult;
  }

  async executeShadow(operation, prodLatency) {
    const shadowStart = Date.now();

    try {
      await operation(this.shadowClient.db());
      const shadowLatency = Date.now() - shadowStart;

      if (shadowLatency > prodLatency * 1.2) {
        this.metrics.shadowSlower++;
      }

      // Log comparison
      console.log(`Prod: ${prodLatency}ms, Shadow: ${shadowLatency}ms`);
    } catch (error) {
      console.error('Shadow error:', error.message);
      throw error;
    }
  }

  getMetrics() {
    return {
      totalRequests: this.metrics.requests,
      shadowErrors: this.metrics.shadowErrors,
      shadowErrorRate: (this.metrics.shadowErrors / this.metrics.requests * 100).toFixed(2) + '%',
      shadowSlowerCount: this.metrics.shadowSlower,
      shadowSlowerRate: (this.metrics.shadowSlower / this.metrics.requests * 100).toFixed(2) + '%'
    };
  }
}

// Express middleware pour shadow testing
const proxy = new ShadowProxy(
  'mongodb://production:27017',
  'mongodb://shadow:27017'
);

app.use(async (req, res, next) => {
  // Wrap database operations
  req.db = {
    findOne: async (collection, query) => {
      return proxy.proxyOperation(db =>
        db.collection(collection).findOne(query)
      );
    }
  };

  next();
});

// Report endpoint
app.get('/shadow/metrics', (req, res) => {
  res.json(proxy.getMetrics());
});
```

### Canary Deployment Testing

```javascript
// canary-test.js
class CanaryTest {
  constructor(productionUri, canaryUri, canaryPercent = 5) {
    this.prodClient = new MongoClient(productionUri);
    this.canaryClient = new MongoClient(canaryUri);
    this.canaryPercent = canaryPercent;

    this.metrics = {
      production: { requests: 0, errors: 0, latencies: [] },
      canary: { requests: 0, errors: 0, latencies: [] }
    };
  }

  async routeRequest(operation) {
    // Route X% to canary
    const useCanary = Math.random() * 100 < this.canaryPercent;
    const client = useCanary ? this.canaryClient : this.prodClient;
    const metrics = useCanary ? this.metrics.canary : this.metrics.production;

    metrics.requests++;
    const start = Date.now();

    try {
      const result = await operation(client.db());
      metrics.latencies.push(Date.now() - start);
      return result;
    } catch (error) {
      metrics.errors++;
      throw error;
    }
  }

  analyzeCanary() {
    const prodErrorRate = this.metrics.production.errors /
                          this.metrics.production.requests;
    const canaryErrorRate = this.metrics.canary.errors /
                            this.metrics.canary.requests;

    const prodP99 = this.percentile(this.metrics.production.latencies, 99);
    const canaryP99 = this.percentile(this.metrics.canary.latencies, 99);

    console.log('=== CANARY ANALYSIS ===');
    console.log('Production:');
    console.log(`  Requests: ${this.metrics.production.requests}`);
    console.log(`  Error rate: ${(prodErrorRate * 100).toFixed(2)}%`);
    console.log(`  P99 latency: ${prodP99.toFixed(2)}ms`);

    console.log('\nCanary:');
    console.log(`  Requests: ${this.metrics.canary.requests}`);
    console.log(`  Error rate: ${(canaryErrorRate * 100).toFixed(2)}%`);
    console.log(`  P99 latency: ${canaryP99.toFixed(2)}ms`);

    // Decision
    if (canaryErrorRate > prodErrorRate * 1.5) {
      console.log('\n🔴 ABORT: Canary showing elevated error rate');
      return 'ABORT';
    }

    if (canaryP99 > prodP99 * 1.3) {
      console.log('\n🔴 ABORT: Canary showing degraded latency');
      return 'ABORT';
    }

    console.log('\n✅ PROMOTE: Canary performing well');
    return 'PROMOTE';
  }

  percentile(arr, p) {
    const sorted = arr.sort((a, b) => a - b);
    const index = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[index] || 0;
  }
}

// Gradual rollout
async function canaryDeployment() {
  // Phase 1 : 5% traffic
  let canary = new CanaryTest(prodUri, canaryUri, 5);
  // ... run for 1 hour ...
  if (canary.analyzeCanary() === 'ABORT') return;

  // Phase 2 : 25% traffic
  canary = new CanaryTest(prodUri, canaryUri, 25);
  // ... run for 1 hour ...
  if (canary.analyzeCanary() === 'ABORT') return;

  // Phase 3 : 50% traffic
  canary = new CanaryTest(prodUri, canaryUri, 50);
  // ... run for 1 hour ...
  if (canary.analyzeCanary() === 'ABORT') return;

  // Full rollout
  console.log('✅ Full rollout approved');
}
```

## Best Practices

### Checklist de Benchmarking

```
☐ Preparation
  ☐ Environnement dédié et isolé
  ☐ Hardware représentatif de production
  ☐ Dataset réaliste (taille, distribution)
  ☐ Workload représentatif (read/write ratio)
  ☐ Warm-up period configuré

☐ Execution
  ☐ Multiple runs (minimum 3)
  ☐ Durée suffisante (minimum 10 minutes)
  ☐ Monitoring système actif
  ☐ Pas d'autres charges sur le système
  ☐ Logs détaillés activés

☐ Metrics Collection
  ☐ Latency (P50, P95, P99, P999)
  ☐ Throughput (ops/sec)
  ☐ Error rate
  ☐ Resource utilization (CPU, RAM, I/O)
  ☐ MongoDB-specific (cache hit, replication lag)

☐ Analysis
  ☐ Baseline établie
  ☐ Comparaison avant/après
  ☐ Significance statistique vérifiée
  ☐ Bottlenecks identifiés
  ☐ Trade-offs documentés

☐ Decision
  ☐ Amélioration >10% pour déployer
  ☐ Pas de dégradation sur métriques critiques
  ☐ Validated en production-like
  ☐ Rollback plan défini
```

### Erreurs Courantes

```javascript
// ❌ ERREUR 1 : Pas de warm-up
// Cold cache fausse les résultats
await benchmark.run(); // Première opération très lente

// ✅ CORRECT : Warm-up avant mesure
await benchmark.warmup(1000);
await benchmark.run();

// ❌ ERREUR 2 : Dataset non-représentatif
// Test avec 1000 documents alors que prod a 100M
const testData = 1000;

// ✅ CORRECT : Dataset similaire à production
const testData = 100000000; // Ou proportionnel

// ❌ ERREUR 3 : Un seul run
// Variance élevée, pas de confiance statistique
const result = runBenchmark();

// ✅ CORRECT : Multiple runs avec analyse
const results = [];
for (let i = 0; i < 5; i++) {
  results.push(await runBenchmark());
}
const analysis = statisticalAnalysis(results);

// ❌ ERREUR 4 : Ignorer le monitoring système
// Focus uniquement sur MongoDB

// ✅ CORRECT : Monitoring holistique
// CPU, RAM, Disk I/O, Network, MongoDB metrics

// ❌ ERREUR 5 : Benchmark sur laptop
// Performance non-représentative

// ✅ CORRECT : Environnement production-like
// Même hardware, même configuration
```

## Conclusion

Le benchmarking et les tests de charge sont essentiels pour :

**Validation** :
- Confirmer les améliorations (2-10× typique)
- Éviter les régressions
- Quantifier les gains

**Dimensionnement** :
- Identifier les limites du système
- Planifier la capacité
- Optimiser les coûts

**Confiance** :
- Déployer en production sereinement
- Prédire le comportement sous charge
- Anticiper les problèmes

**Méthodologie clé** :
1. **Baseline** : Mesurer l'état actuel
2. **Optimize** : Appliquer changements
3. **Test** : Benchmarker avec même protocole
4. **Compare** : Analyser différences (>10% significatif)
5. **Validate** : Tester production-like (shadow, canary)
6. **Deploy** : Rollout progressif avec monitoring

**Outils recommandés** :
- **YCSB** : Standard industry pour comparaisons
- **Custom framework** : Workload spécifique
- **mgodatagen** : Génération datasets réalistes
- **Statistical analysis** : Validation significance

**Métriques critiques** :
- Latency P99 < 50ms (target)
- Throughput selon requirements
- Error rate < 0.1%
- Cache hit ratio > 95%
- Resource utilization < 80%

**Tests obligatoires** :
- ✅ Baseline (référence)
- ✅ Stress (limites)
- ✅ Endurance (stabilité)
- ✅ Spike (résilience)
- ✅ A/B comparison (validation)

Le benchmarking n'est pas une option mais une nécessité pour toute optimisation sérieuse. "You can't improve what you don't measure" - sans benchmark, pas d'optimisation validée.

---

**Points clés à retenir :**
- Baseline obligatoire avant toute optimisation
- Multiple runs (≥3) pour significance statistique
- Warm-up period essentiel (éviter cold cache bias)
- Dataset et workload doivent être représentatifs
- Amélioration >10% pour justifier déploiement
- Production-like testing (shadow, canary) avant rollout
- Monitoring holistique (MongoDB + système)
- Documentation complète des résultats et décisions

⏭️ [DevOps et Déploiement](/18-devops-deploiement/README.md)
