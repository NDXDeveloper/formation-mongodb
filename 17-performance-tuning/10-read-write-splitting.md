🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.10 Read/Write Splitting

## Introduction

Le read/write splitting est une stratégie d'optimisation fondamentale dans MongoDB qui exploite l'architecture des replica sets pour distribuer la charge entre les membres. En dirigeant intelligemment les lectures vers les secondaries et les écritures vers le primary, il est possible d'augmenter le throughput global de 2-5× tout en réduisant la latence sous charge.

Cependant, cette stratégie introduit des compromis critiques en termes de cohérence des données (replication lag) et nécessite une compréhension approfondie des read preferences, write concerns, et des patterns d'accès applicatifs.

Cette section explore les architectures de read/write splitting, leurs implémentations, impacts sur les performances, et méthodologies d'optimisation pour différents profils de charge en production.

## Architecture du Replica Set

### Topologie et Rôles

```
┌───────────────────────────────────────────────────────┐
│                    Replica Set                        │
│                                                       │
│  ┌──────────────┐         ┌──────────────┐            │
│  │   PRIMARY    │────────>│  SECONDARY 1 │            │
│  │              │ oplog   │              │            │
│  │ All Writes   │ repl    │ Read replica │            │
│  │ Some Reads   │────────>│              │            │
│  └──────┬───────┘         └──────────────┘            │
│         │ oplog                                       │
│         │ repl                                        │
│         v                                             │
│  ┌──────────────┐                                     │
│  │  SECONDARY 2 │                                     │
│  │              │                                     │
│  │ Read replica │                                     │
│  └──────────────┘                                     │
│                                                       │
└───────────────────────────────────────────────────────┘

Application Layer:
┌──────────────┐         ┌──────────────┐
│   WRITES     │────────>│   PRIMARY    │
│              │         │              │
└──────────────┘         └──────────────┘

┌──────────────┐         ┌──────────────┐
│   READS      │────────>│ SECONDARIES  │
│ (eventual    │         │              │
│  consistency)│         └──────────────┘
└──────────────┘
```

### Flow de Données

**Opération Write** :
```
1. Application → Primary (write)
   ↓
2. Primary → Oplog (local)
   ↓
3. Primary → Secondaries (oplog replication)
   ↓
4. Secondaries → Apply oplog entries
   ↓
5. Replication lag = temps entre (2) et (4)
```

**Opération Read** :
```
Read with readPreference: primary
├─ Application → Primary → Response
└─ Consistent, mais charge le Primary

Read with readPreference: secondary
├─ Application → Secondary → Response
└─ Éventuellement consistent, décharge le Primary
```

## Read Preferences

### Modes Disponibles

MongoDB offre 5 modes de read preference :

#### 1. primary (Défaut)

```javascript
// Configuration
db.collection.find().readPref("primary")

// Comportement
- Toutes les lectures sur le Primary
- Cohérence forte garantie
- Aucune lecture stale possible
```

**Caractéristiques** :
```yaml
Avantages:
  - Cohérence forte (linearizable reads)
  - Aucun replication lag concern
  - Données toujours à jour

Inconvénients:
  - Charge entière sur Primary
  - Pas de scaling horizontal des reads
  - Primary devient bottleneck

Use cases:
  - Writes fréquents après reads (read-your-writes)
  - Cohérence critique (transactions financières)
  - Workload write-heavy (peu de bénéfice splitting)
```

**Métriques typiques** :
```
Setup: 3-node replica set, 10K ops/sec (70% reads)

primary only:
- Primary CPU: 85%
- Secondary 1 CPU: 15% (oplog application)
- Secondary 2 CPU: 15%
- Read latency P99: 45ms

Conclusion: Primary saturé, secondaries sous-utilisés
```

#### 2. primaryPreferred

```javascript
// Configuration
db.collection.find().readPref("primaryPreferred")

// Comportement
- Préfère le Primary
- Fallback vers Secondary si Primary indisponible
```

**Caractéristiques** :
```yaml
Avantages:
  - Cohérence forte quand Primary disponible
  - Haute disponibilité (reads continuent en cas de failover)
  - Transition smooth pendant élection

Inconvénients:
  - Pas de distribution de charge en temps normal
  - Stale reads possibles pendant failover

Use cases:
  - Cohérence prioritaire mais HA requise
  - Protection contre downtime Primary
  - Transition progressive vers secondary reads
```

**Pattern typique** :
```javascript
// Application configuration
const client = new MongoClient(uri, {
  readPreference: 'primaryPreferred',
  maxStalenessSeconds: 120  // Accepter max 2 min de lag
});

// Scénario : Primary failover
// 1. Primary down détecté
// 2. Driver route vers Secondary immédiatement
// 3. Reads continuent (avec possible staleness)
// 4. Nouvelle élection terminée
// 5. Retour vers nouveau Primary
```

#### 3. secondary

```javascript
// Configuration
db.collection.find().readPref("secondary")

// Comportement
- Toutes les lectures sur Secondaries uniquement
- Erreur si aucun Secondary disponible
```

**Caractéristiques** :
```yaml
Avantages:
  - Décharge complète du Primary pour reads
  - Scaling horizontal des reads
  - Primary dédié aux writes

Inconvénients:
  - Eventual consistency (replication lag)
  - Possible stale reads
  - Erreur si tous Secondaries down

Use cases:
  - Read-heavy workload (>80% reads)
  - Eventual consistency acceptable
  - Analytics / Reporting séparés
  - Read scaling prioritaire
```

**Performance impact** :
```
Setup: 3-node replica set, 10K ops/sec (70% reads)

secondary reads:
- Primary CPU: 35% (3K writes + oplog)
- Secondary 1 CPU: 40% (3.5K reads + oplog)
- Secondary 2 CPU: 40% (3.5K reads + oplog)
- Read latency P99: 25ms (déchargement)

Amélioration: 2× distribution de charge
```

#### 4. secondaryPreferred (Recommandé pour Read Splitting)

```javascript
// Configuration
db.collection.find().readPref("secondaryPreferred")

// Comportement
- Préfère les Secondaries
- Fallback vers Primary si aucun Secondary disponible
```

**Caractéristiques** :
```yaml
Avantages:
  - Distribution de charge vers Secondaries
  - Haute disponibilité (fallback Primary)
  - Meilleur compromis pour read scaling

Inconvénients:
  - Eventual consistency (replication lag)
  - Stale reads possibles

Use cases:
  - Read-heavy avec HA requise
  - Scaling horizontal optimal
  - Production généraliste avec forte charge read
```

**Configuration optimale** :
```javascript
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred',
  maxStalenessSeconds: 90,  // Max 90s de lag acceptable

  // Tag sets pour routing intelligent
  readPreferenceTags: [
    { dc: 'local', usage: 'read' },  // Préférence 1
    { dc: 'local' },                 // Préférence 2
    {}                                // Fallback any
  ]
});
```

#### 5. nearest

```javascript
// Configuration
db.collection.find().readPref("nearest")

// Comportement
- Route vers le membre avec latence réseau la plus faible
- Peut inclure Primary ou Secondaries
```

**Caractéristiques** :
```yaml
Avantages:
  - Latence minimale (géographique)
  - Optimal pour multi-datacenter
  - Distribution automatique basée sur network latency

Inconvénients:
  - Peut surcharger le Primary si closest
  - Eventual consistency (si secondary)
  - Comportement variable selon topology

Use cases:
  - Déploiement multi-région
  - Latence critique (<10ms P99)
  - Applications distribuées géographiquement
```

**Mesure de latency** :
```javascript
function measureMemberLatency() {
  const status = rs.status();

  const latencies = status.members.map(member => ({
    name: member.name,
    state: member.stateStr,
    pingMs: member.pingMs || 0,
    health: member.health
  }));

  print("Member Latencies:");
  latencies.forEach(m => {
    print(`  ${m.name}: ${m.pingMs}ms (${m.state})`);
  });

  return latencies;
}

measureMemberLatency();

// Exemple output:
// mongo1.dc1: 1ms (PRIMARY)
// mongo2.dc1: 2ms (SECONDARY)
// mongo3.dc2: 45ms (SECONDARY)
//
// nearest → mongo1 ou mongo2 (selon load)
```

### Read Preference avec Tag Sets

**Configuration de tags** :
```javascript
// Configuration des tags sur les membres
cfg = rs.conf();

cfg.members[0].tags = { dc: "east", usage: "general" };
cfg.members[1].tags = { dc: "east", usage: "analytics" };
cfg.members[2].tags = { dc: "west", usage: "general" };

rs.reconfig(cfg);
```

**Utilisation dans application** :
```javascript
// Read general queries from east datacenter
db.collection.find().readPref(
  "secondaryPreferred",
  [
    { dc: "east", usage: "general" },  // Préférence 1
    { dc: "east" },                    // Préférence 2
    {}                                  // Fallback any
  ]
);

// Analytics queries vers secondary dédié
db.collection.aggregate([...]).readPref(
  "secondary",
  [
    { usage: "analytics" }  // Seulement vers analytics secondary
  ]
);
```

**Architecture avec tags** :
```
┌────────────────────────────────────────────────────┐
│ Replica Set avec Tag Sets                          │
│                                                    │
│  Primary (dc: east, usage: general)                │
│  ├─ Writes: Toutes                                 │
│  └─ Reads: Applications est + fallback             │
│                                                    │
│  Secondary 1 (dc: east, usage: analytics)          │
│  ├─ Reads: Queries analytics/reporting             │
│  └─ Isolated workload                              │
│                                                    │
│  Secondary 2 (dc: west, usage: general)            │
│  ├─ Reads: Applications ouest                      │
│  └─ Low-latency local reads                        │
└────────────────────────────────────────────────────┘

Benefits:
- Workload isolation (analytics vs transactional)
- Geographic locality (latency optimization)
- Resource segregation (CPU, I/O)
```

### maxStalenessSeconds

Contrôle l'acceptable replication lag pour les reads.

```javascript
// Configuration
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred',
  maxStalenessSeconds: 120  // Max 2 minutes de lag
});

// Comportement
- Driver mesure le replication lag de chaque Secondary
- Exclut les Secondaries avec lag > maxStalenessSeconds
- Route seulement vers Secondaries "fresh enough"
```

**Calcul du replication lag** :
```javascript
function calculateReplicationLag() {
  const status = rs.status();
  const primary = status.members.find(m => m.state === 1);
  const secondaries = status.members.filter(m => m.state === 2);

  const lags = secondaries.map(sec => {
    const lagMs = primary.optimeDate - sec.optimeDate;
    const lagSeconds = Math.floor(lagMs / 1000);

    return {
      member: sec.name,
      lagSeconds: lagSeconds,
      acceptable: (maxStalenessSeconds) => lagSeconds <= maxStalenessSeconds
    };
  });

  return lags;
}

const lags = calculateReplicationLag();
lags.forEach(lag => {
  print(`${lag.member}: ${lag.lagSeconds}s lag`);
  print(`  Acceptable for maxStaleness=90s: ${lag.acceptable(90)}`);
});
```

**Trade-offs maxStalenessSeconds** :

| Valeur | Cohérence | Disponibilité | Use Case |
|--------|-----------|---------------|----------|
| **90s** | Bonne | Excellente | **Défaut recommandé** |
| 30s | Meilleure | Bonne | Staleness sensible |
| 300s | Acceptable | Maximale | Haute disponibilité prioritaire |
| Non défini | Variable | Maximale | Accepte tout lag |

## Write Concerns

### Impact sur Read/Write Splitting

Write concern détermine le niveau de garantie d'écriture.

#### w: 1 (Défaut)

```javascript
db.collection.insertOne(
  { data: "example" },
  { writeConcern: { w: 1 } }
);

// Comportement
- Acknowledge dès que Primary a écrit
- N'attend PAS la réplication vers Secondaries
- Latence minimale
```

**Implications** :
```yaml
Avantages:
  - Latence write minimale
  - Throughput maximal
  - Primary non bloqué par replication

Inconvénients:
  - Risque de perte de données (failover avant replication)
  - Reads sur Secondary peuvent être stale même après write ack
  - Pas de read-after-write consistency si read sur Secondary

Scénario problématique:
1. Write vers Primary avec w:1
2. Primary acknowledge immédiatement
3. Application read depuis Secondary (pas encore répliqué)
4. Read retourne ancienne valeur
   → Read-after-write violation
```

#### w: "majority" (Recommandé Production)

```javascript
db.collection.insertOne(
  { data: "example" },
  { writeConcern: { w: "majority" } }
);

// Comportement
- Acknowledge quand majorité des membres ont écrit
- Garantit durabilité en cas de failover
- Latence accrue
```

**Implications** :
```yaml
Avantages:
  - Durabilité garantie (majorité a données)
  - Safe en cas de failover Primary
  - Meilleure cohérence pour reads

Inconvénients:
  - Latence write +5-50ms selon network
  - Throughput réduit
  - Dépend de replication lag

Amélioration pour read splitting:
- Après write avec w:majority, données sur majorité
- Read depuis Secondary a meilleure chance d'être consistent
- Mais pas de garantie absolue (depend de quel Secondary)
```

**Mesure d'impact** :
```javascript
// Benchmark write concern
async function benchmarkWriteConcern(iterations = 1000) {
  const results = {
    w1: { total: 0, avg: 0 },
    majority: { total: 0, avg: 0 }
  };

  // Test w:1
  let start = Date.now();
  for (let i = 0; i < iterations; i++) {
    await db.test.insertOne(
      { data: i },
      { writeConcern: { w: 1 } }
    );
  }
  results.w1.total = Date.now() - start;
  results.w1.avg = results.w1.total / iterations;

  // Test w:majority
  start = Date.now();
  for (let i = 0; i < iterations; i++) {
    await db.test.insertOne(
      { data: i + iterations },
      { writeConcern: { w: "majority" } }
    );
  }
  results.majority.total = Date.now() - start;
  results.majority.avg = results.majority.total / iterations;

  results.overhead = {
    ms: results.majority.avg - results.w1.avg,
    percent: ((results.majority.avg / results.w1.avg - 1) * 100).toFixed(2)
  };

  return results;
}

// Résultats typiques:
// w:1 → 2ms avg
// w:majority → 7ms avg
// Overhead: +5ms (+250%)
```

#### j: true (Journal)

```javascript
db.collection.insertOne(
  { data: "example" },
  { writeConcern: { w: 1, j: true } }
);

// Comportement
- Attend que write soit dans journal (WAL)
- Garantit durabilité en cas de crash mongod
```

**Impact** :
```
Sans j:true:
├─ Write en cache WiredTiger
├─ Acknowledge immédiat
└─ Journal flush async (100ms)

Avec j:true:
├─ Write en cache WiredTiger
├─ Force flush vers journal
├─ Acknowledge après journal write
└─ +3-10ms latency

Overhead: +5-15% latency
Bénéfice: Durabilité crash
```

### Read Concern

Complément du write concern pour les reads.

#### available (Défaut)

```javascript
db.collection.find().readConcern("available")

// Comportement
- Retourne données immédiatement disponibles
- Peut retourner données non commitées (orphaned)
- Performance maximale
```

#### local

```javascript
db.collection.find().readConcern("local")

// Comportement
- Retourne dernières données locales au membre
- Pas de garantie de durabilité
- Standard pour la plupart des reads
```

#### majority

```javascript
db.collection.find().readConcern("majority")

// Comportement
- Retourne seulement données répliquées sur majorité
- Garantit que données ne seront pas rollback
- Cohérence forte mais latence accrue
```

**Trade-off readConcern** :

```javascript
// Scénario : Read après Write
// 1. Write avec w:1
db.orders.insertOne({ id: 123 }, { writeConcern: { w: 1 } });

// 2. Read immédiat depuis Secondary
// Option A : readConcern: local
db.orders.find({ id: 123 }).readConcern("local");
// → Peut ne pas voir le document (pas encore répliqué)

// Option B : readConcern: majority
db.orders.find({ id: 123 }).readConcern("majority");
// → Attendra que document soit sur majorité
// → Garantit visibilité si write était w:majority

// Option C : Read depuis Primary
db.orders.find({ id: 123 }).readPref("primary");
// → Voit toujours le document
// → Mais charge le Primary
```

## Architectures de Read/Write Splitting

### Architecture 1 : Simple Read Splitting

**Configuration** :
```
Topology:
- 1 Primary
- 2 Secondaries (read replicas)

Strategy:
- Writes → Primary (w:majority)
- Reads → secondaryPreferred
- No dedicated secondaries
```

**Implémentation** :
```javascript
// Configuration driver
const client = new MongoClient(uri, {
  replicaSet: 'rs0',
  readPreference: 'secondaryPreferred',
  readConcern: { level: 'local' },
  writeConcern: { w: 'majority', j: false }
});

// Application code
// Writes (automatic routing to primary)
await db.orders.insertOne({
  customerId: 123,
  total: 99.99
});

// Reads (automatic routing to secondary)
const orders = await db.orders.find({
  customerId: 123
}).toArray();
```

**Performance** :
```
Before splitting (all on Primary):
- Primary CPU: 80%
- Read latency P99: 35ms
- Write latency P99: 25ms
- Max throughput: 8K ops/sec

After splitting:
- Primary CPU: 40% (writes only)
- Secondary 1 CPU: 30%
- Secondary 2 CPU: 30%
- Read latency P99: 25ms (improved)
- Write latency P99: 28ms (w:majority overhead)
- Max throughput: 15K ops/sec (+87%)
```

### Architecture 2 : Dedicated Analytics Secondary

**Configuration** :
```
Topology:
- 1 Primary
- 1 Secondary (transactional reads)
- 1 Secondary (analytics, hidden, priority=0)

Strategy:
- Writes → Primary
- Transactional reads → Secondary 1
- Analytics/Reporting → Secondary 2 (isolated)
```

**Configuration membre analytics** :
```javascript
cfg = rs.conf();

// Secondary 2 : Analytics
cfg.members[2].priority = 0;  // Ne peut pas devenir Primary
cfg.members[2].hidden = true;  // Caché du driver par défaut
cfg.members[2].tags = { usage: "analytics" };

rs.reconfig(cfg);
```

**Utilisation** :
```javascript
// Application transactionnelle (standard)
const transactionalClient = new MongoClient(uri, {
  readPreference: 'secondaryPreferred',
  // Exclut hidden members automatiquement
});

// Application analytics (explicit)
const analyticsClient = new MongoClient(uri, {
  readPreference: 'secondary',
  readPreferenceTags: [{ usage: 'analytics' }],
  directConnection: true  // Connect directement au member
});

// Analytics queries
const report = await analyticsDb.orders.aggregate([
  { $match: { date: { $gte: startDate } } },
  { $group: {
      _id: "$category",
      revenue: { $sum: "$total" }
  }},
  { $sort: { revenue: -1 } }
], { allowDiskUse: true });
```

**Bénéfices** :
```yaml
Isolation:
  - Analytics queries n'impactent pas transactional
  - Pas de contention CPU/RAM
  - allowDiskUse sans risque

Performance:
  - Primary: 100% writes
  - Secondary 1: 100% transactional reads
  - Secondary 2: 100% analytics (isolated)
  - No interference

Configuration:
  - Secondary 2 peut avoir config différente:
    * Plus de RAM (analytics cache)
    * Slower storage OK (batch OK)
    * Différent index strategy
```

### Architecture 3 : Geographic Distribution

**Configuration** :
```
Topology: Multi-datacenter
- Primary (DC1 - East)
- Secondary 1 (DC1 - East)
- Secondary 2 (DC2 - West)

Strategy:
- Writes → Primary (DC1)
- East reads → DC1 members (nearest)
- West reads → DC2 member (nearest)
```

**Configuration avec tags** :
```javascript
cfg = rs.conf();

cfg.members[0].tags = { dc: "east", region: "us-east-1" };
cfg.members[1].tags = { dc: "east", region: "us-east-1" };
cfg.members[2].tags = { dc: "west", region: "us-west-1" };

rs.reconfig(cfg);
```

**Application routing** :
```javascript
// East coast application
const eastClient = new MongoClient(uri, {
  readPreference: 'nearest',
  readPreferenceTags: [
    { dc: 'east' },  // Prefer local DC
    {}               // Fallback any
  ],
  maxStalenessSeconds: 90
});

// West coast application
const westClient = new MongoClient(uri, {
  readPreference: 'nearest',
  readPreferenceTags: [
    { dc: 'west' },  // Prefer local DC
    {}               // Fallback any (cross-DC)
  ],
  maxStalenessSeconds: 90
});
```

**Latency impact** :
```
East application:
- Write latency: 5ms (primary local)
- Read latency: 2ms (secondary local)

West application:
- Write latency: 75ms (primary cross-DC)
- Read latency: 3ms (secondary local)

Improvement:
- Without splitting: All queries to East DC
  * West read latency: 70ms
- With splitting: Local reads
  * West read latency: 3ms (23× faster!)
```

### Architecture 4 : Priority-based Routing

**Configuration** :
```
Topology:
- Primary (high-end server)
- Secondary 1 (high-end, critical reads)
- Secondary 2 (standard, general reads)

Strategy:
- Critical reads → Secondary 1
- General reads → Secondary 2
- Writes → Primary
```

**Tag configuration** :
```javascript
cfg = rs.conf();

cfg.members[1].tags = { tier: "premium", priority: "high" };
cfg.members[2].tags = { tier: "standard", priority: "normal" };

rs.reconfig(cfg);
```

**Application routing** :
```javascript
// Critical path (user-facing)
async function getCriticalData(userId) {
  return await db.users.findOne(
    { _id: userId },
    {
      readPreference: 'secondary',
      readPreferenceTags: [
        { priority: 'high' },  // Premium secondary
        {}                     // Fallback
      ]
    }
  );
}

// Background processing (less critical)
async function getAnalyticsData() {
  return await db.events.aggregate([...], {
    readPreference: 'secondary',
    readPreferenceTags: [
      { priority: 'normal' }  // Standard secondary
    ],
    allowDiskUse: true
  });
}
```

## Cohérence et Replication Lag

### Problématique de la Cohérence

**Read-your-writes problem** :
```javascript
// Scénario problématique
async function problemScenario() {
  // 1. User crée un order
  const result = await db.orders.insertOne(
    { userId: 123, total: 99.99 },
    { writeConcern: { w: 1 } }  // Ack immédiat
  );

  const orderId = result.insertedId;

  // 2. Immédiatement, redirect vers page order
  // 3. Read depuis Secondary (pas encore répliqué)
  const order = await db.orders.findOne(
    { _id: orderId }
    // readPreference: secondaryPreferred (default)
  );

  // 4. order === null (Secondary lag = 100ms)
  // 5. User voit erreur "Order not found"

  return order;
}

// Solution 1 : Read from Primary pour read-after-write
async function solution1() {
  const result = await db.orders.insertOne({...});

  // Read from Primary explicitly
  const order = await db.orders.findOne(
    { _id: result.insertedId }
  ).readPref('primary');

  return order;
}

// Solution 2 : w:majority + maxStaleness
async function solution2() {
  const result = await db.orders.insertOne(
    {...},
    { writeConcern: { w: 'majority' } }  // Majorité a données
  );

  // Read avec maxStaleness court
  const order = await db.orders.findOne(
    { _id: result.insertedId }
  ).readPref('secondaryPreferred', null, { maxStalenessSeconds: 5 });

  return order;
}

// Solution 3 : Session causally consistent
async function solution3() {
  const session = client.startSession({ causalConsistency: true });

  try {
    const result = await db.orders.insertOne(
      {...},
      { session }
    );

    // Read voit le write précédent
    const order = await db.orders.findOne(
      { _id: result.insertedId },
      { session }
    );

    return order;
  } finally {
    await session.endSession();
  }
}
```

### Monitoring Replication Lag

**Script de monitoring** :
```javascript
function comprehensiveReplicationLagMonitoring() {
  const status = rs.status();
  const primary = status.members.find(m => m.state === 1);
  const secondaries = status.members.filter(m => m.state === 2);

  const monitoring = {
    timestamp: new Date(),
    primary: {
      name: primary.name,
      optime: primary.optimeDate
    },
    secondaries: [],
    metrics: {
      maxLagSeconds: 0,
      avgLagSeconds: 0,
      minLagSeconds: Infinity
    }
  };

  secondaries.forEach(sec => {
    const lagMs = primary.optimeDate - sec.optimeDate;
    const lagSeconds = lagMs / 1000;

    monitoring.secondaries.push({
      name: sec.name,
      lagSeconds: lagSeconds.toFixed(2),
      health: sec.health,
      state: sec.stateStr,
      pingMs: sec.pingMs || 0
    });

    // Update metrics
    if (lagSeconds > monitoring.metrics.maxLagSeconds) {
      monitoring.metrics.maxLagSeconds = lagSeconds;
    }
    if (lagSeconds < monitoring.metrics.minLagSeconds) {
      monitoring.metrics.minLagSeconds = lagSeconds;
    }
  });

  // Calculate average
  const totalLag = monitoring.secondaries.reduce((sum, s) =>
    sum + parseFloat(s.lagSeconds), 0);
  monitoring.metrics.avgLagSeconds =
    (totalLag / monitoring.secondaries.length).toFixed(2);

  // Assessment
  const maxLag = monitoring.metrics.maxLagSeconds;
  if (maxLag > 60) {
    monitoring.assessment = "⚠️ HIGH LAG (>60s) - Investigate replication";
  } else if (maxLag > 10) {
    monitoring.assessment = "WARNING: Elevated lag (>10s)";
  } else {
    monitoring.assessment = "✅ Replication lag healthy";
  }

  return monitoring;
}

// Exécution périodique (toutes les 30s)
setInterval(() => {
  const lag = comprehensiveReplicationLagMonitoring();
  print(JSON.stringify(lag, null, 2));
}, 30000);
```

**Causes de replication lag** :

```javascript
function diagnoseReplicationLag() {
  const serverStatus = db.serverStatus();
  const replStatus = rs.status();

  const diagnosis = {
    possibleCauses: []
  };

  // 1. Network issues
  replStatus.members.forEach(member => {
    if (member.pingMs > 50) {
      diagnosis.possibleCauses.push({
        cause: "High network latency",
        member: member.name,
        pingMs: member.pingMs,
        recommendation: "Check network between members"
      });
    }
  });

  // 2. Secondary overloaded (read queries)
  const ops = serverStatus.opcounters;
  const totalOps = ops.query + ops.insert + ops.update + ops.delete;
  if (totalOps > 10000) {  // Threshold
    diagnosis.possibleCauses.push({
      cause: "Secondary overloaded with read queries",
      opsPerSec: totalOps,
      recommendation: "Reduce read load or add more secondaries"
    });
  }

  // 3. Slow oplog application
  const replMetrics = serverStatus.metrics?.repl;
  if (replMetrics && replMetrics.apply) {
    const applyBatchMs = replMetrics.apply.batchSize || 0;
    if (applyBatchMs > 1000) {
      diagnosis.possibleCauses.push({
        cause: "Slow oplog batch application",
        avgBatchMs: applyBatchMs,
        recommendation: "Check secondary performance (CPU, I/O)"
      });
    }
  }

  // 4. Disk I/O saturation
  const wtStats = serverStatus.wiredTiger;
  if (wtStats) {
    const checkpointMs = wtStats.transaction["transaction checkpoint most recent time (msecs)"];
    if (checkpointMs > 30000) {
      diagnosis.possibleCauses.push({
        cause: "Slow checkpoints (I/O bottleneck)",
        checkpointMs: checkpointMs,
        recommendation: "Upgrade storage or reduce write load"
      });
    }
  }

  // 5. Initial sync in progress
  replStatus.members.forEach(member => {
    if (member.stateStr === "STARTUP2") {
      diagnosis.possibleCauses.push({
        cause: "Initial sync in progress",
        member: member.name,
        recommendation: "Wait for initial sync to complete"
      });
    }
  });

  return diagnosis;
}

printjson(diagnoseReplicationLag());
```

### Stratégies de Mitigation

**1. Read from Primary pour critical paths** :
```javascript
// Router pattern
class SmartRouter {
  constructor(db) {
    this.db = db;
  }

  // Critical reads : Always from Primary
  async getCriticalData(query) {
    return await this.db.collection.find(query)
      .readPref('primary')
      .toArray();
  }

  // Non-critical reads : Secondary preferred
  async getNonCriticalData(query) {
    return await this.db.collection.find(query)
      .readPref('secondaryPreferred')
      .maxStaleness(90)
      .toArray();
  }

  // Analytics : Dedicated secondary
  async getAnalyticsData(pipeline) {
    return await this.db.collection.aggregate(pipeline, {
      readPreference: 'secondary',
      readPreferenceTags: [{ usage: 'analytics' }],
      allowDiskUse: true
    }).toArray();
  }
}
```

**2. Causal Consistency Sessions** :
```javascript
async function causallyConsistentWorkflow() {
  const session = client.startSession({
    causalConsistency: true
  });

  try {
    // Write
    await db.orders.insertOne(
      { userId: 123, total: 99.99 },
      { session }
    );

    // Read verra le write (même si sur Secondary)
    const orders = await db.orders.find(
      { userId: 123 },
      { session }
    ).readPref('secondaryPreferred').toArray();

    // Update basé sur read précédent
    await db.users.updateOne(
      { _id: 123 },
      { $inc: { orderCount: 1 } },
      { session }
    );

    return orders;
  } finally {
    await session.endSession();
  }
}
```

**3. Application-level retry** :
```javascript
async function readWithRetry(query, maxRetries = 3) {
  let attempt = 0;

  while (attempt < maxRetries) {
    try {
      // Try Secondary first
      const result = await db.collection.findOne(query)
        .readPref('secondaryPreferred')
        .maxStaleness(30);

      if (result) {
        return result;
      }

      // Not found, might be replication lag
      attempt++;
      if (attempt < maxRetries) {
        await new Promise(resolve => setTimeout(resolve, 100));
      }
    } catch (error) {
      throw error;
    }
  }

  // Final attempt: Read from Primary
  return await db.collection.findOne(query).readPref('primary');
}
```

## Performance Monitoring

### Métriques Clés

```javascript
function readWriteSplittingDashboard() {
  const serverStatus = db.serverStatus();
  const replStatus = rs.status();

  const dashboard = {
    timestamp: new Date(),

    // Member status
    members: replStatus.members.map(m => ({
      name: m.name,
      state: m.stateStr,
      health: m.health,
      uptime: m.uptime,
      pingMs: m.pingMs || 0
    })),

    // Replication lag
    replicationLag: (() => {
      const primary = replStatus.members.find(m => m.state === 1);
      const secondaries = replStatus.members.filter(m => m.state === 2);

      return secondaries.map(sec => ({
        member: sec.name,
        lagSeconds: ((primary.optimeDate - sec.optimeDate) / 1000).toFixed(2)
      }));
    })(),

    // Operations distribution
    operations: {
      total: serverStatus.opcounters.query +
             serverStatus.opcounters.insert +
             serverStatus.opcounters.update +
             serverStatus.opcounters.delete,

      breakdown: {
        queries: serverStatus.opcounters.query,
        inserts: serverStatus.opcounters.insert,
        updates: serverStatus.opcounters.update,
        deletes: serverStatus.opcounters.delete
      },

      readWriteRatio: (
        serverStatus.opcounters.query /
        (serverStatus.opcounters.insert +
         serverStatus.opcounters.update +
         serverStatus.opcounters.delete)
      ).toFixed(2)
    },

    // Connections
    connections: {
      current: serverStatus.connections.current,
      available: serverStatus.connections.available,
      active: serverStatus.connections.active || 0
    },

    // Performance
    performance: {
      cachePressure: ((serverStatus.wiredTiger.cache["bytes currently in the cache"] /
                      serverStatus.wiredTiger.cache["maximum bytes configured"]) * 100).toFixed(2) + "%",

      ticketsAvailable: {
        read: serverStatus.wiredTiger.concurrentTransactions.read.available,
        write: serverStatus.wiredTiger.concurrentTransactions.write.available
      }
    }
  };

  // Assessment
  const issues = [];

  dashboard.replicationLag.forEach(lag => {
    if (parseFloat(lag.lagSeconds) > 60) {
      issues.push(`⚠️ High lag on ${lag.member}: ${lag.lagSeconds}s`);
    }
  });

  if (dashboard.connections.available < 100) {
    issues.push("⚠️ Low available connections");
  }

  if (parseFloat(dashboard.performance.cachePressure) > 95) {
    issues.push("⚠️ High cache pressure");
  }

  dashboard.assessment = issues.length === 0 ?
    "✅ Read/Write splitting healthy" : issues;

  return dashboard;
}

// Monitoring périodique
setInterval(() => {
  const dashboard = readWriteSplittingDashboard();
  // Export vers monitoring system (Prometheus, Datadog, etc.)
  print(JSON.stringify(dashboard));
}, 60000);  // Toutes les minutes
```

### Alerting Rules

**Prometheus-style alerts** :
```yaml
# High replication lag
- alert: HighReplicationLag
  expr: mongodb_replication_lag_seconds > 60
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High replication lag detected"
    description: "Replication lag > 60s on {{ $labels.member }}"

# Secondary unhealthy
- alert: SecondaryUnhealthy
  expr: mongodb_member_health == 0
  for: 2m
  labels:
    severity: critical
  annotations:
    summary: "Secondary member unhealthy"
    description: "Member {{ $labels.member }} is unhealthy"

# Unbalanced load
- alert: UnbalancedReadLoad
  expr: |
    stddev(mongodb_operations_reads_per_second) /
    avg(mongodb_operations_reads_per_second) > 0.5
  for: 10m
  labels:
    severity: info
  annotations:
    summary: "Unbalanced read load distribution"
    description: "Read load not evenly distributed across secondaries"

# High secondary CPU
- alert: HighSecondaryCPU
  expr: |
    mongodb_system_cpu_usage{state="secondary"} > 85
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "High CPU on secondary"
    description: "Secondary CPU usage > 85% - consider adding capacity"
```

## Best Practices et Recommandations

### Configuration Recommandée par Workload

**Read-heavy (>70% reads)** :
```javascript
const client = new MongoClient(uri, {
  replicaSet: 'rs0',

  // Read splitting agressif
  readPreference: 'secondaryPreferred',
  maxStalenessSeconds: 90,

  // Write concern balanced
  writeConcern: { w: 'majority', j: false },

  // Read concern standard
  readConcern: { level: 'local' }
});

// Expected improvement: 2-3× read throughput
```

**Write-heavy (>50% writes)** :
```javascript
const client = new MongoClient(uri, {
  replicaSet: 'rs0',

  // Minimal read splitting (Primary peut gérer)
  readPreference: 'primaryPreferred',

  // Write concern performance
  writeConcern: { w: 1, j: false },

  readConcern: { level: 'local' }
});

// Focus: Optimize write path, not read splitting
```

**Mixed with critical consistency** :
```javascript
const client = new MongoClient(uri, {
  replicaSet: 'rs0',

  // Moderate splitting
  readPreference: 'secondaryPreferred',
  maxStalenessSeconds: 30,  // Tighter staleness

  // Strong durability
  writeConcern: { w: 'majority', j: true },

  // Stronger consistency
  readConcern: { level: 'majority' }
});

// Use causal consistency sessions for workflows
```

### Checklist de Déploiement

```
☐ Architecture Design
  ☐ Définir read/write ratio du workload
  ☐ Identifier queries critiques (consistency required)
  ☐ Décider si dedicated analytics secondary nécessaire
  ☐ Planifier topology (datacenter, tags)

☐ Configuration
  ☐ Read preference adapté au workload
  ☐ maxStalenessSeconds approprié (30-90s)
  ☐ Write concern pour durabilité vs performance
  ☐ Tag sets si architecture complexe

☐ Application Code
  ☐ Utiliser sessions causal consistency si nécessaire
  ☐ Différencier critical vs non-critical reads
  ☐ Implémenter retry logic pour replication lag
  ☐ Tester read-after-write scenarios

☐ Monitoring
  ☐ Replication lag alerting (<60s)
  ☐ Load distribution metrics
  ☐ Secondary health checks
  ☐ Connection distribution tracking

☐ Testing
  ☐ Load testing avec read splitting activé
  ☐ Failover scenarios (Primary down)
  ☐ Replication lag simulation
  ☐ Consistency validation (stale reads acceptables?)

☐ Documentation
  ☐ Read preference strategy documentée
  ☐ Consistency guarantees clairement définis
  ☐ Failover behavior compris par équipe
  ☐ Monitoring runbook créé
```

### Erreurs Courantes

```javascript
// ❌ ERREUR 1 : Read splitting sans considérer consistency
// Code assume strong consistency mais lit depuis Secondary
async function badPattern1() {
  const result = await db.orders.insertOne({ total: 99.99 });

  // Immédiatement après, lit depuis Secondary
  const order = await db.orders.findOne({ _id: result.insertedId });
  // readPreference: secondaryPreferred (default)

  // order peut être null si lag > 0
  return order;
}

// ✅ CORRECT : Use Primary ou causal consistency
async function goodPattern1() {
  const session = client.startSession({ causalConsistency: true });

  const result = await db.orders.insertOne(
    { total: 99.99 },
    { session }
  );

  const order = await db.orders.findOne(
    { _id: result.insertedId },
    { session }  // Verra le write
  );

  await session.endSession();
  return order;
}

// ❌ ERREUR 2 : Ignorer maxStalenessSeconds
const client = new MongoClient(uri, {
  readPreference: 'secondary'
  // Pas de maxStalenessSeconds → Accepte lag infini
});

// ✅ CORRECT : Définir limite acceptable
const client = new MongoClient(uri, {
  readPreference: 'secondary',
  maxStalenessSeconds: 90  // Max 90s de lag
});

// ❌ ERREUR 3 : Read splitting sur workload write-heavy
// 70% writes, 30% reads
const client = new MongoClient(uri, {
  readPreference: 'secondary'  // Peu de bénéfice, overhead réplication
});

// ✅ CORRECT : Garder Primary pour reads aussi
const client = new MongoClient(uri, {
  readPreference: 'primary'  // Plus simple et efficace
});

// ❌ ERREUR 4 : Pas de fallback strategy
const client = new MongoClient(uri, {
  readPreference: 'secondary'  // Erreur si tous Secondaries down
});

// ✅ CORRECT : Use secondaryPreferred pour HA
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred'  // Fallback Primary
});
```

## Conclusion

Le read/write splitting est une technique puissante pour améliorer les performances MongoDB :

**Bénéfices mesurables** :
- Throughput : +50-300% selon ratio read/write
- Latence reads : -20-40% (déchargement Primary)
- Scalabilité : Horizontale pour reads
- Isolation : Workloads séparés (analytics vs transactional)

**Trade-offs critiques** :
- Cohérence : Eventual consistency (replication lag)
- Complexité : Configuration et monitoring accrus
- Risques : Read-after-write violations si mal configuré

**Recommandations clés** :
1. **Analyser workload** : Read/write ratio, consistency requirements
2. **secondaryPreferred défaut** : Meilleur compromis HA + performance
3. **maxStalenessSeconds : 90s** : Balance cohérence/disponibilité
4. **Causal consistency sessions** : Pour workflows critiques
5. **Monitoring continu** : Replication lag < 10s optimal
6. **Tag sets** : Pour architectures avancées (analytics, geo)

**Quand utiliser** :
- ✅ Read-heavy (>60% reads)
- ✅ High traffic nécessitant scaling
- ✅ Analytics/Reporting séparés
- ✅ Multi-datacenter deployment

**Quand éviter** :
- ❌ Strong consistency absolue requise partout
- ❌ Write-heavy (>60% writes)
- ❌ Très faible traffic (overhead > bénéfice)
- ❌ Replication lag chroniquement élevé

Le read/write splitting n'est pas une solution universelle mais un outil puissant qui, correctement configuré et monitoré, peut transformer les performances d'une application MongoDB à forte charge de lecture.

---

**Points clés à retenir :**
- secondaryPreferred est le meilleur compromis production (HA + performance)
- maxStalenessSeconds : 90s recommandé (cohérence vs disponibilité)
- Causal consistency sessions pour read-after-write scenarios
- Monitoring replication lag essentiel (<10s optimal, <60s acceptable)
- Tag sets permettent architectures avancées (analytics, geo-distribution)
- w:majority + readConcern:majority pour cohérence forte si nécessaire
- Test de failover obligatoire avant production
- Read splitting bénéfique seulement si >60% reads

⏭️ [Caching strategies](/17-performance-tuning/11-caching-strategies.md)
