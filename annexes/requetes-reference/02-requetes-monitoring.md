🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.2 - Requêtes de Monitoring

## Table des matières

1. [Opérations en Cours](#op%C3%A9rations-en-cours)
2. [Profiler de Requêtes](#profiler-de-requ%C3%AAtes)
3. [Métriques de Performance](#m%C3%A9triques-de-performance)
4. [Connexions](#connexions)
5. [Réplication et Lag](#r%C3%A9plication-et-lag)
6. [Métriques Temps Réel](#m%C3%A9triques-temps-r%C3%A9el)
7. [Alerting et Seuils](#alerting-et-seuils)

---

## Opérations en Cours

### 1.1 - Toutes les opérations actives

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 OPÉRATIONS ACTIVES
// ============================================

// 💡 Objectif
// Afficher toutes les opérations actuellement en cours

// 🎯 Cas d'usage
// - Monitoring en temps réel
// - Diagnostic de performance
// - Identification de requêtes bloquantes

// ============================================
// REQUÊTE
// ============================================

db.currentOp({ "$all": false })

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  inprog: [
    {
      type: "op",
      host: "mongodb1:27017",
      desc: "conn12345",
      connectionId: 12345,
      client: "192.168.1.100:52345",
      appName: "myapp",
      clientMetadata: {...},
      active: true,
      currentOpTime: "2024-01-15T10:30:00.000Z",
      opid: 1234567,
      secs_running: 5,
      microsecs_running: 5000000,
      op: "query",
      ns: "myapp.users",
      command: {
        find: "users",
        filter: { status: "active" }
      },
      planSummary: "IXSCAN { status: 1 }",
      numYields: 10,
      locks: {...}
    }
  ],
  ok: 1
}

// ============================================
// 💡 VARIANTES
// ============================================

// Opérations actives uniquement (sans idle)
db.currentOp({ "$all": false, "active": true })

// Avec toutes les opérations (y compris idle)
db.currentOp({ "$all": true })

// Opérations sur une collection spécifique
db.currentOp({
  "$all": false,
  "ns": "myapp.users"
})

// Rapport formaté
function activeOperationsReport() {
  const ops = db.currentOp({ "$all": false, "active": true });

  if (ops.inprog.length === 0) {
    print("✅ No active operations");
    return;
  }

  print(`\n=== Active Operations (${ops.inprog.length}) ===\n`);

  ops.inprog.forEach(function(op) {
    print(`OpID: ${op.opid}`);
    print(`Operation: ${op.op}`);
    print(`Namespace: ${op.ns || "N/A"}`);
    print(`Running: ${op.secs_running}s`);
    print(`Client: ${op.client || "N/A"}`);
    if (op.planSummary) {
      print(`Plan: ${op.planSummary}`);
    }
    print("---");
  });
}

activeOperationsReport();
```

---

### 1.2 - Opérations longues (slow queries)

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 OPÉRATIONS LONGUES
// ============================================

// 💡 Objectif
// Identifier les opérations qui s'exécutent depuis trop longtemps

// 🎯 Cas d'usage
// - Détection de requêtes problématiques
// - Alerting sur les performances
// - Diagnostic de blocages

// ============================================
// REQUÊTE
// ============================================

// Opérations > 5 secondes
db.currentOp({
  "$all": false,
  "active": true,
  "secs_running": { "$gte": 5 }
})

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  inprog: [
    {
      opid: 1234567,
      op: "query",
      ns: "myapp.orders",
      secs_running: 12,
      microsecs_running: 12000000,
      command: {
        find: "orders",
        filter: { date: { $gte: ISODate("2024-01-01") } }
      },
      planSummary: "COLLSCAN",
      client: "192.168.1.100:52345"
    }
  ],
  ok: 1
}

// ============================================
// 💡 ANALYSE AVANCÉE
// ============================================

function slowOperationsAnalysis(thresholdSeconds = 5) {
  const ops = db.currentOp({
    "$all": false,
    "active": true,
    "secs_running": { "$gte": thresholdSeconds }
  });

  if (ops.inprog.length === 0) {
    print(`✅ No operations running longer than ${thresholdSeconds}s`);
    return;
  }

  print(`\n⚠️ ${ops.inprog.length} slow operation(s) detected\n`);
  print("=".repeat(80));

  ops.inprog.forEach(function(op) {
    print(`\nOpID: ${op.opid}`);
    print(`Type: ${op.op}`);
    print(`Namespace: ${op.ns || "N/A"}`);
    print(`Duration: ${op.secs_running}s`);
    print(`Client: ${op.client || "internal"}`);

    if (op.planSummary) {
      print(`Execution plan: ${op.planSummary}`);

      if (op.planSummary.includes("COLLSCAN")) {
        print("⚠️ WARNING: Collection scan detected - consider adding an index");
      }
    }

    if (op.command) {
      print(`Command: ${JSON.stringify(op.command, null, 2)}`);
    }

    if (op.secs_running > 30) {
      print(`❌ CRITICAL: Operation running for ${op.secs_running}s`);
      print(`   Consider killing with: db.killOp(${op.opid})`);
    }

    print("=".repeat(80));
  });

  // Statistiques
  const durations = ops.inprog.map(op => op.secs_running);
  const avgDuration = (durations.reduce((a, b) => a + b, 0) / durations.length).toFixed(2);
  const maxDuration = Math.max(...durations);

  print(`\nStatistics:`);
  print(`  Count: ${ops.inprog.length}`);
  print(`  Average duration: ${avgDuration}s`);
  print(`  Maximum duration: ${maxDuration}s`);
}

slowOperationsAnalysis(5);

// ============================================
// 💡 VARIANTES PAR SEUIL
// ============================================

// Opérations très longues (> 30s)
db.currentOp({
  "$all": false,
  "secs_running": { "$gte": 30 }
})

// Opérations bloquantes
db.currentOp({
  "$all": false,
  "waitingForLock": true
})

// Opérations d'écriture longues
db.currentOp({
  "$all": false,
  "op": { "$in": ["insert", "update", "remove"] },
  "secs_running": { "$gte": 5 }
})
```

---

### 1.3 - Opérations par type

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 OPÉRATIONS PAR TYPE
// ============================================

// 💡 Objectif
// Grouper et compter les opérations par type

// 🎯 Cas d'usage
// - Vue d'ensemble de l'activité
// - Identification des patterns
// - Monitoring de charge

// ============================================
// REQUÊTE
// ============================================

const ops = db.currentOp({ "$all": false });
const stats = {
  total: ops.inprog.length,
  byType: {},
  byNamespace: {},
  active: 0,
  idle: 0
};

ops.inprog.forEach(function(op) {
  // Par type
  const opType = op.op || "unknown";
  stats.byType[opType] = (stats.byType[opType] || 0) + 1;

  // Par namespace
  if (op.ns) {
    stats.byNamespace[op.ns] = (stats.byNamespace[op.ns] || 0) + 1;
  }

  // Active vs Idle
  if (op.active) {
    stats.active++;
  } else {
    stats.idle++;
  }
});

printjson(stats);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  total: 52,
  byType: {
    query: 25,
    insert: 10,
    update: 8,
    command: 7,
    getmore: 2
  },
  byNamespace: {
    "myapp.users": 15,
    "myapp.orders": 12,
    "myapp.products": 8
  },
  active: 12,
  idle: 40
}

// ============================================
// 💡 RAPPORT DÉTAILLÉ
// ============================================

function operationsBreakdown() {
  const ops = db.currentOp({ "$all": true });

  const breakdown = {
    total: ops.inprog.length,
    operations: {},
    clients: new Set(),
    databases: new Set(),
    longRunning: 0
  };

  ops.inprog.forEach(function(op) {
    // Types d'opérations
    const opType = op.op || "none";
    if (!breakdown.operations[opType]) {
      breakdown.operations[opType] = {
        count: 0,
        active: 0,
        totalTime: 0,
        maxTime: 0
      };
    }

    breakdown.operations[opType].count++;

    if (op.active) {
      breakdown.operations[opType].active++;
    }

    if (op.secs_running) {
      breakdown.operations[opType].totalTime += op.secs_running;
      breakdown.operations[opType].maxTime = Math.max(
        breakdown.operations[opType].maxTime,
        op.secs_running
      );

      if (op.secs_running > 5) {
        breakdown.longRunning++;
      }
    }

    // Clients uniques
    if (op.client) {
      breakdown.clients.add(op.client);
    }

    // Bases de données
    if (op.ns) {
      const db = op.ns.split('.')[0];
      breakdown.databases.add(db);
    }
  });

  print("\n=== Operations Breakdown ===\n");
  print(`Total operations: ${breakdown.total}`);
  print(`Unique clients: ${breakdown.clients.size}`);
  print(`Databases accessed: ${breakdown.databases.size}`);
  print(`Long running (>5s): ${breakdown.longRunning}`);

  print("\n--- By Operation Type ---");
  Object.keys(breakdown.operations).forEach(function(opType) {
    const op = breakdown.operations[opType];
    const avgTime = op.count > 0 ? (op.totalTime / op.count).toFixed(2) : 0;

    print(`\n${opType}:`);
    print(`  Total: ${op.count}`);
    print(`  Active: ${op.active}`);
    print(`  Avg time: ${avgTime}s`);
    print(`  Max time: ${op.maxTime}s`);
  });
}

operationsBreakdown();
```

---

### 1.4 - Tuer une opération

**🔴 Niveau : Avancé** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 TUER UNE OPÉRATION
// ============================================

// 💡 Objectif
// Terminer une opération problématique

// 🎯 Cas d'usage
// - Arrêter une requête bloquante
// - Libérer des ressources
// - Intervention d'urgence

// ⚠️ ATTENTION
// À utiliser avec précaution - peut impacter les clients

// ============================================
// PROCÉDURE
// ============================================

// 1. Identifier l'opération à tuer
const slowOps = db.currentOp({
  "$all": false,
  "secs_running": { "$gte": 30 }
});

if (slowOps.inprog.length > 0) {
  print("Slow operations found:");
  slowOps.inprog.forEach(function(op) {
    print(`OpID: ${op.opid} - ${op.op} on ${op.ns} - ${op.secs_running}s`);
  });

  // 2. Tuer l'opération (remplacer <opid> par le vrai ID)
  // db.killOp(<opid>)

  // Exemple:
  // db.killOp(1234567)
} else {
  print("No slow operations to kill");
}

// ============================================
// 💡 FONCTION HELPER
// ============================================

function killSlowOperations(thresholdSeconds = 60, dryRun = true) {
  const ops = db.currentOp({
    "$all": false,
    "active": true,
    "secs_running": { "$gte": thresholdSeconds }
  });

  if (ops.inprog.length === 0) {
    print(`✅ No operations running longer than ${thresholdSeconds}s`);
    return;
  }

  print(`\nFound ${ops.inprog.length} operation(s) to kill:\n`);

  ops.inprog.forEach(function(op) {
    print(`OpID: ${op.opid}`);
    print(`  Type: ${op.op}`);
    print(`  Namespace: ${op.ns || "N/A"}`);
    print(`  Duration: ${op.secs_running}s`);
    print(`  Client: ${op.client || "internal"}`);

    if (!dryRun) {
      try {
        db.killOp(op.opid);
        print(`  ✅ Killed`);
      } catch (e) {
        print(`  ❌ Failed to kill: ${e.message}`);
      }
    } else {
      print(`  [DRY RUN] Would kill this operation`);
    }

    print("---");
  });

  if (dryRun) {
    print("\n⚠️ This was a dry run. Set dryRun=false to actually kill operations.");
  }
}

// Dry run (ne tue pas réellement)
killSlowOperations(60, true);

// Pour vraiment tuer les opérations (ATTENTION!)
// killSlowOperations(60, false);
```

---

## Profiler de Requêtes

### 2.1 - Activer et configurer le profiler

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 CONFIGURATION DU PROFILER
// ============================================

// 💡 Objectif
// Activer le profiler pour enregistrer les requêtes lentes

// 🎯 Cas d'usage
// - Analyse de performance
// - Identification de requêtes à optimiser
// - Audit des requêtes

// ============================================
// NIVEAUX DU PROFILER
// ============================================

/*
Niveau 0: Désactivé
Niveau 1: Requêtes lentes uniquement (> seuil)
Niveau 2: Toutes les requêtes (⚠️ impact performance)
*/

// ============================================
// REQUÊTES
// ============================================

// Activer le profiler (niveau 1, seuil 100ms)
db.setProfilingLevel(1, 100)

// Vérifier le statut
db.getProfilingStatus()

// Résultat:
{
  was: 1,
  slowms: 100,
  sampleRate: 1.0,
  ok: 1
}

// ============================================
// 💡 CONFIGURATIONS COURANTES
// ============================================

// Requêtes > 50ms
db.setProfilingLevel(1, 50)

// Requêtes > 200ms
db.setProfilingLevel(1, 200)

// Toutes les requêtes (debug uniquement)
db.setProfilingLevel(2)

// Désactiver
db.setProfilingLevel(0)

// ============================================
// 💡 PROFILER AVEC SAMPLING
// ============================================

// Profiler 50% des requêtes lentes (réduit l'overhead)
db.setProfilingLevel(1, { slowms: 100, sampleRate: 0.5 })

// ============================================
// 🔧 BONNES PRATIQUES
// ============================================

function setupProfiler(slowMs = 100, level = 1) {
  const currentStatus = db.getProfilingStatus();

  print(`\nCurrent profiler status:`);
  print(`  Level: ${currentStatus.was}`);
  print(`  Slow threshold: ${currentStatus.slowms}ms`);
  print(`  Sample rate: ${currentStatus.sampleRate * 100}%`);

  if (currentStatus.was === level && currentStatus.slowms === slowMs) {
    print(`\n✅ Profiler already configured as requested`);
    return;
  }

  db.setProfilingLevel(level, slowMs);

  print(`\n✅ Profiler configured:`);
  print(`  Level: ${level}`);
  print(`  Slow threshold: ${slowMs}ms`);

  if (level === 2) {
    print(`\n⚠️ WARNING: Level 2 profiles ALL queries`);
    print(`   This can impact performance significantly`);
    print(`   Use only for short debugging sessions`);
  }
}

setupProfiler(100, 1);
```

---

### 2.2 - Requêtes les plus lentes

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 REQUÊTES LES PLUS LENTES
// ============================================

// 💡 Objectif
// Identifier les requêtes ayant les temps d'exécution les plus longs

// 🎯 Cas d'usage
// - Priorisation des optimisations
// - Analyse de performance
// - Reporting

// 🔑 Prérequis
// Profiler activé (niveau 1 ou 2)

// ============================================
// REQUÊTE
// ============================================

db.system.profile.find({
  millis: { $gt: 100 }
}).sort({ millis: -1 }).limit(10)

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    op: "query",
    ns: "myapp.users",
    command: {
      find: "users",
      filter: { status: "active", age: { $gte: 18 } }
    },
    planSummary: "COLLSCAN",
    execStats: {...},
    millis: 1250,
    ts: ISODate("2024-01-15T10:30:00Z"),
    client: "192.168.1.100:52345",
    appName: "myapp"
  }
]

// ============================================
// 💡 ANALYSE DÉTAILLÉE
// ============================================

function slowQueriesReport(thresholdMs = 100, limit = 10) {
  const slowQueries = db.system.profile.find({
    millis: { $gt: thresholdMs }
  }).sort({ millis: -1 }).limit(limit).toArray();

  if (slowQueries.length === 0) {
    print(`✅ No queries slower than ${thresholdMs}ms found`);
    return;
  }

  print(`\n=== Top ${limit} Slowest Queries ===\n`);

  slowQueries.forEach(function(query, index) {
    print(`${index + 1}. Duration: ${query.millis}ms`);
    print(`   Operation: ${query.op}`);
    print(`   Namespace: ${query.ns}`);
    print(`   Timestamp: ${query.ts}`);

    if (query.planSummary) {
      print(`   Plan: ${query.planSummary}`);

      if (query.planSummary.includes("COLLSCAN")) {
        print(`   ⚠️ Collection scan - consider adding index`);
      }
    }

    if (query.command) {
      print(`   Command: ${JSON.stringify(query.command, null, 2)}`);
    }

    if (query.execStats) {
      print(`   Docs examined: ${query.execStats.totalDocsExamined || "N/A"}`);
      print(`   Docs returned: ${query.execStats.nReturned || "N/A"}`);
    }

    print("   ---");
  });

  // Statistiques
  const durations = slowQueries.map(q => q.millis);
  const avgDuration = (durations.reduce((a, b) => a + b, 0) / durations.length).toFixed(2);

  print(`\nStatistics:`);
  print(`  Total slow queries: ${slowQueries.length}`);
  print(`  Average duration: ${avgDuration}ms`);
  print(`  Slowest: ${Math.max(...durations)}ms`);
}

slowQueriesReport(100, 10);

// ============================================
// 💡 VARIANTES
// ============================================

// Requêtes avec COLLSCAN
db.system.profile.find({
  planSummary: "COLLSCAN",
  millis: { $gt: 50 }
}).sort({ millis: -1 }).limit(10)

// Requêtes sur une collection spécifique
db.system.profile.find({
  ns: "myapp.users",
  millis: { $gt: 100 }
}).sort({ millis: -1 }).limit(10)

// Requêtes d'écriture lentes
db.system.profile.find({
  op: { $in: ["insert", "update", "remove"] },
  millis: { $gt: 100 }
}).sort({ millis: -1 }).limit(10)
```

---

### 2.3 - Requêtes les plus fréquentes

**🟡 Niveau : Intermédiaire** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 REQUÊTES LES PLUS FRÉQUENTES
// ============================================

// 💡 Objectif
// Identifier les patterns de requêtes les plus courants

// 🎯 Cas d'usage
// - Optimisation des index
// - Analyse des patterns d'accès
// - Planification du cache

// ============================================
// REQUÊTE
// ============================================

db.system.profile.aggregate([
  {
    $match: {
      ts: { $gte: new Date(Date.now() - 3600000) }  // Dernière heure
    }
  },
  {
    $group: {
      _id: {
        op: "$op",
        ns: "$ns",
        filter: "$command.filter",
        planSummary: "$planSummary"
      },
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      maxMillis: { $max: "$millis" },
      minMillis: { $min: "$millis" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 10 }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    _id: {
      op: "query",
      ns: "myapp.users",
      filter: { status: "active" },
      planSummary: "IXSCAN { status: 1 }"
    },
    count: 1523,
    avgMillis: 12.5,
    maxMillis: 45,
    minMillis: 3
  },
  {
    _id: {
      op: "query",
      ns: "myapp.orders",
      filter: { customerId: { $exists: true } },
      planSummary: "COLLSCAN"
    },
    count: 842,
    avgMillis: 180.3,
    maxMillis: 890,
    minMillis: 120
  }
]

// ============================================
// 💡 RAPPORT FORMATÉ
// ============================================

function frequentQueriesReport(hours = 1, limit = 10) {
  const cutoff = new Date(Date.now() - hours * 3600000);

  const queries = db.system.profile.aggregate([
    {
      $match: {
        ts: { $gte: cutoff }
      }
    },
    {
      $group: {
        _id: {
          op: "$op",
          ns: "$ns",
          filter: "$command.filter",
          planSummary: "$planSummary"
        },
        count: { $sum: 1 },
        avgMillis: { $avg: "$millis" },
        maxMillis: { $max: "$millis" },
        totalMillis: { $sum: "$millis" }
      }
    },
    { $sort: { count: -1 } },
    { $limit: limit }
  ]).toArray();

  print(`\n=== Most Frequent Queries (Last ${hours}h) ===\n`);

  queries.forEach(function(query, index) {
    print(`${index + 1}. Executed ${query.count} times`);
    print(`   Operation: ${query._id.op}`);
    print(`   Namespace: ${query._id.ns}`);
    print(`   Average time: ${query.avgMillis.toFixed(2)}ms`);
    print(`   Max time: ${query.maxMillis}ms`);
    print(`   Total time: ${(query.totalMillis / 1000).toFixed(2)}s`);

    if (query._id.planSummary) {
      print(`   Plan: ${query._id.planSummary}`);
    }

    if (query._id.filter) {
      print(`   Filter: ${JSON.stringify(query._id.filter)}`);
    }

    // Recommandations
    const impact = query.count * query.avgMillis;
    if (impact > 10000) {  // > 10 secondes total
      print(`   ⚠️ HIGH IMPACT: Total time ${(impact / 1000).toFixed(2)}s`);
    }

    if (query._id.planSummary && query._id.planSummary.includes("COLLSCAN")) {
      print(`   ⚠️ Collection scan - high priority for indexing`);
    }

    print("   ---");
  });
}

frequentQueriesReport(1, 10);
```

---

### 2.4 - Nettoyer le profiler

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 NETTOYER LE PROFILER
// ============================================

// 💡 Objectif
// Vider la collection system.profile pour libérer de l'espace

// 🎯 Cas d'usage
// - Maintenance régulière
// - Libération d'espace disque
// - Reset des statistiques

// ⚠️ Note
// La collection system.profile est capped (taille fixe)
// Elle se nettoie automatiquement (FIFO)

// ============================================
// REQUÊTES
// ============================================

// Vérifier la taille actuelle
db.system.profile.stats(1024 * 1024)

// Résultat:
{
  ns: "myapp.system.profile",
  size: 15.5,  // MB
  count: 50000,
  avgObjSize: 324
}

// Supprimer les anciennes entrées (> 24h)
db.system.profile.deleteMany({
  ts: { $lt: new Date(Date.now() - 86400000) }
})

// Vider complètement (nécessite de désactiver le profiler)
db.setProfilingLevel(0)
db.system.profile.drop()
db.createCollection("system.profile", { capped: true, size: 16777216 })  // 16MB
db.setProfilingLevel(1, 100)

// ============================================
// 💡 AUTOMATISATION
// ============================================

function cleanProfiler(daysToKeep = 7) {
  const cutoff = new Date(Date.now() - daysToKeep * 86400000);

  print(`Cleaning profiler data older than ${daysToKeep} days...`);

  const stats = db.system.profile.stats();
  print(`Current entries: ${stats.count}`);

  const result = db.system.profile.deleteMany({
    ts: { $lt: cutoff }
  });

  print(`Deleted: ${result.deletedCount} entries`);

  const newStats = db.system.profile.stats();
  print(`Remaining entries: ${newStats.count}`);
}

cleanProfiler(7);
```

---

## Métriques de Performance

### 3.1 - Taux d'opérations par seconde

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 OPS PAR SECONDE
// ============================================

// 💡 Objectif
// Calculer le nombre d'opérations par seconde

// 🎯 Cas d'usage
// - Monitoring de charge
// - Planification de capacité
// - Détection de pics

// ============================================
// REQUÊTE
// ============================================

function getOpsPerSecond(interval = 10) {
  const status1 = db.serverStatus();
  const ops1 = status1.opcounters;
  const time1 = status1.uptime;

  print(`Collecting metrics... (waiting ${interval}s)`);
  sleep(interval * 1000);

  const status2 = db.serverStatus();
  const ops2 = status2.opcounters;
  const time2 = status2.uptime;

  const timeDiff = time2 - time1;

  const opsPerSec = {
    insert: ((ops2.insert - ops1.insert) / timeDiff).toFixed(2),
    query: ((ops2.query - ops1.query) / timeDiff).toFixed(2),
    update: ((ops2.update - ops1.update) / timeDiff).toFixed(2),
    delete: ((ops2.delete - ops1.delete) / timeDiff).toFixed(2),
    getmore: ((ops2.getmore - ops1.getmore) / timeDiff).toFixed(2),
    command: ((ops2.command - ops1.command) / timeDiff).toFixed(2)
  };

  const totalOps = Object.values(opsPerSec).reduce((a, b) => parseFloat(a) + parseFloat(b), 0);

  print("\n=== Operations Per Second ===\n");
  print(`Insert:  ${opsPerSec.insert}/s`);
  print(`Query:   ${opsPerSec.query}/s`);
  print(`Update:  ${opsPerSec.update}/s`);
  print(`Delete:  ${opsPerSec.delete}/s`);
  print(`Getmore: ${opsPerSec.getmore}/s`);
  print(`Command: ${opsPerSec.command}/s`);
  print(`\nTotal:   ${totalOps.toFixed(2)}/s`);

  return opsPerSec;
}

getOpsPerSecond(10);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
Collecting metrics... (waiting 10s)

=== Operations Per Second ===

Insert:  125.50/s
Query:   450.30/s
Update:  89.20/s
Delete:  12.50/s
Getmore: 45.60/s
Command: 320.10/s

Total:   1043.20/s
*/

// ============================================
// 💡 MONITORING CONTINU
// ============================================

function monitorOpsPerSecond(intervalSeconds = 10, iterations = 6) {
  print(`Monitoring ops/sec every ${intervalSeconds}s (${iterations} iterations)\n`);
  print("Time".padEnd(12) + "Insert/s".padEnd(12) + "Query/s".padEnd(12) + "Update/s".padEnd(12) + "Total/s");
  print("-".repeat(60));

  for (let i = 0; i < iterations; i++) {
    const status1 = db.serverStatus();
    const ops1 = status1.opcounters;
    const time1 = status1.uptime;

    sleep(intervalSeconds * 1000);

    const status2 = db.serverStatus();
    const ops2 = status2.opcounters;
    const time2 = status2.uptime;

    const timeDiff = time2 - time1;

    const insertOps = ((ops2.insert - ops1.insert) / timeDiff).toFixed(2);
    const queryOps = ((ops2.query - ops1.query) / timeDiff).toFixed(2);
    const updateOps = ((ops2.update - ops1.update) / timeDiff).toFixed(2);
    const totalOps = (parseFloat(insertOps) + parseFloat(queryOps) + parseFloat(updateOps)).toFixed(2);

    const timestamp = new Date().toISOString().substr(11, 8);

    print(
      timestamp.padEnd(12) +
      insertOps.padEnd(12) +
      queryOps.padEnd(12) +
      updateOps.padEnd(12) +
      totalOps
    );
  }
}

// Usage: monitorOpsPerSecond(5, 12);  // 5s interval, 12 times = 1 minute total
```

---

### 3.2 - Utilisation CPU et mémoire

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 CPU ET MÉMOIRE
// ============================================

// 💡 Objectif
// Surveiller l'utilisation des ressources système

// 🎯 Cas d'usage
// - Monitoring de santé
// - Détection de problèmes
// - Alerting

// ============================================
// REQUÊTE
// ============================================

function systemResourcesReport() {
  const status = db.serverStatus();

  print("\n=== System Resources ===\n");

  // Mémoire
  print("--- Memory ---");
  print(`Resident: ${status.mem.resident} MB`);
  print(`Virtual: ${status.mem.virtual} MB`);

  if (status.mem.mapped) {
    print(`Mapped: ${status.mem.mapped} MB`);
  }

  // WiredTiger Cache
  if (status.wiredTiger && status.wiredTiger.cache) {
    const cache = status.wiredTiger.cache;
    const maxCache = cache["maximum bytes configured"] / (1024 * 1024);
    const currentCache = cache["bytes currently in the cache"] / (1024 * 1024);
    const usage = (currentCache / maxCache * 100).toFixed(2);

    print("\n--- WiredTiger Cache ---");
    print(`Max size: ${maxCache.toFixed(2)} MB`);
    print(`Current: ${currentCache.toFixed(2)} MB`);
    print(`Usage: ${usage}%`);

    if (usage > 90) {
      print(`⚠️ WARNING: Cache usage > 90%`);
    }
  }

  // Connexions
  print("\n--- Connections ---");
  print(`Current: ${status.connections.current}`);
  print(`Available: ${status.connections.available}`);

  const connUsage = (status.connections.current / (status.connections.current + status.connections.available) * 100).toFixed(2);
  print(`Usage: ${connUsage}%`);

  if (connUsage > 80) {
    print(`⚠️ WARNING: Connection usage > 80%`);
  }

  // Réseau
  print("\n--- Network ---");
  const bytesIn = (status.network.bytesIn / (1024 * 1024 * 1024)).toFixed(2);
  const bytesOut = (status.network.bytesOut / (1024 * 1024 * 1024)).toFixed(2);
  print(`Bytes in: ${bytesIn} GB`);
  print(`Bytes out: ${bytesOut} GB`);
  print(`Requests: ${status.network.numRequests.toLocaleString()}`);

  // Globale
  print("\n--- General ---");
  print(`Uptime: ${(status.uptime / 3600).toFixed(2)} hours`);
  print(`Version: ${db.version()}`);
}

systemResourcesReport();

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== System Resources ===

--- Memory ---
Resident: 1250 MB
Virtual: 2500 MB

--- WiredTiger Cache ---
Max size: 2048.00 MB
Current: 1792.50 MB
Usage: 87.52%

--- Connections ---
Current: 52
Available: 838808
Usage: 0.01%

--- Network ---
Bytes in: 125.50 GB
Bytes out: 89.25 GB
Requests: 15,234,567

--- General ---
Uptime: 168.50 hours
Version: 7.0.5
*/

// ============================================
// 💡 ALERTING
// ============================================

function checkResourceThresholds() {
  const status = db.serverStatus();
  const alerts = [];

  // Cache usage
  if (status.wiredTiger && status.wiredTiger.cache) {
    const cache = status.wiredTiger.cache;
    const usage = (cache["bytes currently in the cache"] / cache["maximum bytes configured"]) * 100;

    if (usage > 90) {
      alerts.push(`❌ CRITICAL: Cache usage ${usage.toFixed(2)}% (>90%)`);
    } else if (usage > 80) {
      alerts.push(`⚠️ WARNING: Cache usage ${usage.toFixed(2)}% (>80%)`);
    }
  }

  // Connections
  const connUsage = (status.connections.current / (status.connections.current + status.connections.available)) * 100;

  if (connUsage > 80) {
    alerts.push(`❌ CRITICAL: Connection usage ${connUsage.toFixed(2)}% (>80%)`);
  } else if (connUsage > 60) {
    alerts.push(`⚠️ WARNING: Connection usage ${connUsage.toFixed(2)}% (>60%)`);
  }

  // Memory
  const memoryMB = status.mem.resident;
  if (memoryMB > 4096) {  // > 4GB
    alerts.push(`⚠️ INFO: High memory usage: ${memoryMB} MB`);
  }

  // Affichage
  if (alerts.length === 0) {
    print("✅ All resource thresholds are healthy");
  } else {
    print("\n=== Resource Alerts ===\n");
    alerts.forEach(alert => print(alert));
  }

  return alerts.length === 0;
}

checkResourceThresholds();
```

---

## Connexions

### 4.1 - Connexions actives par client

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 CONNEXIONS PAR CLIENT
// ============================================

// 💡 Objectif
// Identifier les clients avec le plus de connexions

// 🎯 Cas d'usage
// - Détection de fuites de connexions
// - Monitoring des applications
// - Planification de pools

// ============================================
// REQUÊTE
// ============================================

const ops = db.currentOp({ "$all": true });
const connectionsByClient = {};
const connectionsByApp = {};

ops.inprog.forEach(function(op) {
  if (op.client) {
    connectionsByClient[op.client] = (connectionsByClient[op.client] || 0) + 1;
  }

  if (op.appName) {
    connectionsByApp[op.appName] = (connectionsByApp[op.appName] || 0) + 1;
  }
});

// Convertir en tableau et trier
const clientStats = Object.keys(connectionsByClient).map(client => ({
  client: client,
  connections: connectionsByClient[client]
})).sort((a, b) => b.connections - a.connections);

const appStats = Object.keys(connectionsByApp).map(app => ({
  app: app,
  connections: connectionsByApp[app]
})).sort((a, b) => b.connections - a.connections);

print("\n=== Connections by Client ===\n");
clientStats.slice(0, 10).forEach(function(stat) {
  print(`${stat.client}: ${stat.connections} connections`);
});

print("\n=== Connections by Application ===\n");
appStats.forEach(function(stat) {
  print(`${stat.app}: ${stat.connections} connections`);
});

print(`\nTotal connections: ${ops.inprog.length}`);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== Connections by Client ===

192.168.1.100:52345: 25 connections
192.168.1.101:52346: 15 connections
192.168.1.102:52347: 8 connections
192.168.1.103:52348: 4 connections

=== Connections by Application ===

myapp: 35 connections
background-worker: 12 connections
monitoring-service: 5 connections

Total connections: 52
*/

// ============================================
// 💡 DÉTECTION DE FUITES
// ============================================

function detectConnectionLeaks(threshold = 50) {
  const ops = db.currentOp({ "$all": true });
  const stats = {};

  ops.inprog.forEach(function(op) {
    const key = op.client || "unknown";

    if (!stats[key]) {
      stats[key] = {
        count: 0,
        idle: 0,
        active: 0,
        oldestIdle: 0
      };
    }

    stats[key].count++;

    if (op.active) {
      stats[key].active++;
    } else {
      stats[key].idle++;

      if (op.secs_running) {
        stats[key].oldestIdle = Math.max(stats[key].oldestIdle, op.secs_running);
      }
    }
  });

  const leaks = [];

  Object.keys(stats).forEach(function(client) {
    const stat = stats[client];

    if (stat.count > threshold) {
      leaks.push({
        client: client,
        connections: stat.count,
        idle: stat.idle,
        active: stat.active,
        oldestIdleSeconds: stat.oldestIdle
      });
    }
  });

  if (leaks.length === 0) {
    print(`✅ No connection leaks detected (threshold: ${threshold})`);
  } else {
    print(`\n⚠️ Potential connection leaks detected:\n`);

    leaks.forEach(function(leak) {
      print(`Client: ${leak.client}`);
      print(`  Total connections: ${leak.connections}`);
      print(`  Idle: ${leak.idle}`);
      print(`  Active: ${leak.active}`);
      print(`  Oldest idle: ${leak.oldestIdleSeconds}s`);
      print("  ---");
    });
  }

  return leaks;
}

detectConnectionLeaks(50);
```

---

## Réplication et Lag

### 5.1 - Monitoring du replication lag

**🔴 Niveau : Avancé** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 REPLICATION LAG MONITORING
// ============================================

// 💡 Objectif
// Surveiller continuellement le retard de réplication

// 🎯 Cas d'usage
// - Alerting en temps réel
// - Monitoring de santé
// - Diagnostics

// ⚠️ Prérequis
// Replica Set configuré

// ============================================
// REQUÊTE
// ============================================

function monitorReplicationLag(intervalSeconds = 10, iterations = 6) {
  print(`Monitoring replication lag every ${intervalSeconds}s\n`);
  print("Time".padEnd(12) + "Member".padEnd(30) + "Lag (s)".padEnd(12) + "Status");
  print("-".repeat(70));

  for (let i = 0; i < iterations; i++) {
    try {
      const status = rs.status();
      const primary = status.members.find(m => m.state === 1);

      if (!primary) {
        print(`${new Date().toISOString().substr(11, 8).padEnd(12)}NO PRIMARY FOUND`);
      } else {
        status.members.forEach(function(member) {
          if (member.state === 2) {  // SECONDARY
            const lagSeconds = Math.round((status.date - member.optimeDate) / 1000);
            const timestamp = new Date().toISOString().substr(11, 8);

            let statusIcon;
            if (lagSeconds < 5) {
              statusIcon = "✅ Excellent";
            } else if (lagSeconds < 10) {
              statusIcon = "✅ Good";
            } else if (lagSeconds < 30) {
              statusIcon = "⚠️ Warning";
            } else {
              statusIcon = "❌ Critical";
            }

            print(
              timestamp.padEnd(12) +
              member.name.padEnd(30) +
              lagSeconds.toString().padEnd(12) +
              statusIcon
            );
          }
        });
      }
    } catch (e) {
      print(`Error: ${e.message}`);
    }

    if (i < iterations - 1) {
      sleep(intervalSeconds * 1000);
    }
  }
}

monitorReplicationLag(5, 12);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
Monitoring replication lag every 5s

Time        Member                        Lag (s)     Status
----------------------------------------------------------------------
10:30:00    mongodb2:27017                2           ✅ Excellent
10:30:00    mongodb3:27017                3           ✅ Excellent
10:30:05    mongodb2:27017                2           ✅ Excellent
10:30:05    mongodb3:27017                3           ✅ Excellent
10:30:10    mongodb2:27017                15          ⚠️ Warning
10:30:10    mongodb3:27017                3           ✅ Excellent
*/
```

---

## Métriques Temps Réel

### 6.1 - Dashboard temps réel

**🟡 Niveau : Intermédiaire** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 DASHBOARD TEMPS RÉEL
// ============================================

// 💡 Objectif
// Afficher un tableau de bord avec mise à jour continue

// 🎯 Cas d'usage
// - Monitoring en direct
// - Démonstration
// - Surveillance d'incidents

// ============================================
// SCRIPT
// ============================================

function realTimeDashboard(refreshSeconds = 5, iterations = 20) {
  print("Starting real-time monitoring dashboard...");
  print("Press Ctrl+C to stop\n");

  for (let i = 0; i < iterations; i++) {
    // Clear screen effect
    print("\n".repeat(50));
    print("=".repeat(80));
    print(`MONGODB REAL-TIME DASHBOARD - ${new Date().toISOString()}`);
    print("=".repeat(80));

    try {
      const status = db.serverStatus();
      const currentOps = db.currentOp({ "$all": false, "active": true });

      // 1. Général
      print("\n--- GENERAL ---");
      print(`Uptime: ${(status.uptime / 3600).toFixed(2)} hours`);
      print(`Version: ${db.version()}`);
      print(`Host: ${status.host}`);

      // 2. Connexions
      print("\n--- CONNECTIONS ---");
      const connUsage = (status.connections.current / (status.connections.current + status.connections.available)) * 100;
      print(`Current: ${status.connections.current}`);
      print(`Available: ${status.connections.available}`);
      print(`Usage: ${connUsage.toFixed(2)}%`);

      // Barre de progression
      const connBar = Math.round(connUsage / 5);
      print(`[${"|".repeat(connBar)}${".".repeat(20 - connBar)}] ${connUsage.toFixed(0)}%`);

      // 3. Mémoire
      print("\n--- MEMORY ---");
      print(`Resident: ${status.mem.resident} MB`);
      print(`Virtual: ${status.mem.virtual} MB`);

      if (status.wiredTiger && status.wiredTiger.cache) {
        const cache = status.wiredTiger.cache;
        const cacheUsage = (cache["bytes currently in the cache"] / cache["maximum bytes configured"]) * 100;
        const cacheMB = (cache["bytes currently in the cache"] / (1024 * 1024)).toFixed(2);
        const maxCacheMB = (cache["maximum bytes configured"] / (1024 * 1024)).toFixed(2);

        print(`\nWiredTiger Cache: ${cacheMB} / ${maxCacheMB} MB`);
        const cacheBar = Math.round(cacheUsage / 5);
        print(`[${"|".repeat(cacheBar)}${".".repeat(20 - cacheBar)}] ${cacheUsage.toFixed(0)}%`);
      }

      // 4. Opérations actives
      print("\n--- ACTIVE OPERATIONS ---");
      print(`Total: ${currentOps.inprog.length}`);

      if (currentOps.inprog.length > 0) {
        const opTypes = {};
        let slowOps = 0;

        currentOps.inprog.forEach(function(op) {
          const type = op.op || "unknown";
          opTypes[type] = (opTypes[type] || 0) + 1;

          if (op.secs_running && op.secs_running >= 5) {
            slowOps++;
          }
        });

        print("\nBy type:");
        Object.keys(opTypes).forEach(type => {
          print(`  ${type}: ${opTypes[type]}`);
        });

        if (slowOps > 0) {
          print(`\n⚠️ Slow operations (>5s): ${slowOps}`);
        }
      }

      // 5. Opérations/seconde (estimation)
      print("\n--- OPERATIONS COUNTERS ---");
      print(`Insert: ${status.opcounters.insert.toLocaleString()}`);
      print(`Query: ${status.opcounters.query.toLocaleString()}`);
      print(`Update: ${status.opcounters.update.toLocaleString()}`);
      print(`Delete: ${status.opcounters.delete.toLocaleString()}`);

      // 6. Réplication (si applicable)
      try {
        const rsStatus = rs.status();
        print("\n--- REPLICATION ---");

        rsStatus.members.forEach(function(member) {
          const state = member.stateStr;
          const health = member.health === 1 ? "✅" : "❌";

          print(`${health} ${member.name} - ${state}`);

          if (member.state === 2) {  // SECONDARY
            const lagSeconds = Math.round((rsStatus.date - member.optimeDate) / 1000);
            print(`   Lag: ${lagSeconds}s`);
          }
        });
      } catch (e) {
        print("\n--- REPLICATION ---");
        print("Not running in replica set mode");
      }

      // 7. Alertes
      print("\n--- ALERTS ---");
      const alerts = [];

      if (connUsage > 80) {
        alerts.push("❌ Connection usage > 80%");
      }

      if (status.wiredTiger && status.wiredTiger.cache) {
        const cacheUsage = (status.wiredTiger.cache["bytes currently in the cache"] / status.wiredTiger.cache["maximum bytes configured"]) * 100;
        if (cacheUsage > 90) {
          alerts.push("❌ Cache usage > 90%");
        }
      }

      if (currentOps.inprog.some(op => op.secs_running && op.secs_running >= 30)) {
        alerts.push("❌ Operations running > 30s");
      }

      if (alerts.length === 0) {
        print("✅ No alerts");
      } else {
        alerts.forEach(alert => print(alert));
      }

      print("\n" + "=".repeat(80));
      print(`Iteration ${i + 1}/${iterations} - Refreshing in ${refreshSeconds}s...`);

    } catch (e) {
      print(`\n❌ Error: ${e.message}`);
    }

    if (i < iterations - 1) {
      sleep(refreshSeconds * 1000);
    }
  }

  print("\n✅ Monitoring completed");
}

// Usage
realTimeDashboard(5, 20);

// ============================================
// 📊 EXEMPLE DE SORTIE
// ============================================

/*
================================================================================
MONGODB REAL-TIME DASHBOARD - 2024-01-15T10:30:00.000Z
================================================================================

--- GENERAL ---
Uptime: 168.50 hours
Version: 7.0.5
Host: mongodb1:27017

--- CONNECTIONS ---
Current: 52
Available: 838808
Usage: 0.01%
[|..................] 0%

--- MEMORY ---
Resident: 1250 MB
Virtual: 2500 MB

WiredTiger Cache: 1792.50 / 2048.00 MB
[|||||||||||||||||...] 88%

--- ACTIVE OPERATIONS ---
Total: 8

By type:
  query: 5
  update: 2
  command: 1

--- OPERATIONS COUNTERS ---
Insert: 1,523,400
Query: 5,234,567
Update: 892,345
Delete: 123,456

--- REPLICATION ---
✅ mongodb1:27017 - PRIMARY
✅ mongodb2:27017 - SECONDARY
   Lag: 2s
✅ mongodb3:27017 - SECONDARY
   Lag: 3s

--- ALERTS ---
✅ No alerts

================================================================================
Iteration 1/20 - Refreshing in 5s...
*/
```

---

### 6.2 - Monitoring des métriques réseau

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 MÉTRIQUES RÉSEAU TEMPS RÉEL
// ============================================

// 💡 Objectif
// Surveiller le trafic réseau en temps réel

// 🎯 Cas d'usage
// - Analyse de la charge réseau
// - Détection de pics de trafic
// - Planification de bande passante

// ============================================
// SCRIPT
// ============================================

function monitorNetworkMetrics(intervalSeconds = 10, iterations = 12) {
  print("Monitoring network metrics...\n");
  print("Time".padEnd(12) + "Bytes In/s".padEnd(15) + "Bytes Out/s".padEnd(15) + "Requests/s".padEnd(15) + "Status");
  print("-".repeat(70));

  let prevStatus = db.serverStatus();
  let prevTime = Date.now();

  sleep(intervalSeconds * 1000);

  for (let i = 0; i < iterations; i++) {
    const currentStatus = db.serverStatus();
    const currentTime = Date.now();
    const timeDiff = (currentTime - prevTime) / 1000;

    // Calcul des différences
    const bytesInDiff = currentStatus.network.bytesIn - prevStatus.network.bytesIn;
    const bytesOutDiff = currentStatus.network.bytesOut - prevStatus.network.bytesOut;
    const requestsDiff = currentStatus.network.numRequests - prevStatus.network.numRequests;

    // Calcul des taux par seconde
    const bytesInPerSec = (bytesInDiff / timeDiff / 1024).toFixed(2);  // KB/s
    const bytesOutPerSec = (bytesOutDiff / timeDiff / 1024).toFixed(2);  // KB/s
    const requestsPerSec = (requestsDiff / timeDiff).toFixed(2);

    // Statut
    let status = "✅";
    if (bytesInPerSec > 10000 || bytesOutPerSec > 10000) {  // > 10 MB/s
      status = "⚠️ High traffic";
    }

    const timestamp = new Date().toISOString().substr(11, 8);

    print(
      timestamp.padEnd(12) +
      `${bytesInPerSec} KB/s`.padEnd(15) +
      `${bytesOutPerSec} KB/s`.padEnd(15) +
      requestsPerSec.padEnd(15) +
      status
    );

    prevStatus = currentStatus;
    prevTime = currentTime;

    if (i < iterations - 1) {
      sleep(intervalSeconds * 1000);
    }
  }

  print("\n✅ Network monitoring completed");
}

monitorNetworkMetrics(5, 12);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
Monitoring network metrics...

Time        Bytes In/s     Bytes Out/s    Requests/s     Status
----------------------------------------------------------------------
10:30:00    1250.50 KB/s   890.25 KB/s    125.50         ✅
10:30:05    1350.75 KB/s   920.30 KB/s    130.25         ✅
10:30:10    1450.20 KB/s   980.50 KB/s    145.60         ✅
10:30:15    12500.00 KB/s  8900.00 KB/s   1250.00        ⚠️ High traffic

✅ Network monitoring completed
*/

// ============================================
// 💡 VARIANTE - AVEC STATISTIQUES
// ============================================

function networkMetricsWithStats(intervalSeconds = 10, iterations = 6) {
  const metrics = [];

  let prevStatus = db.serverStatus();
  sleep(intervalSeconds * 1000);

  for (let i = 0; i < iterations; i++) {
    const currentStatus = db.serverStatus();

    const bytesInDiff = currentStatus.network.bytesIn - prevStatus.network.bytesIn;
    const bytesOutDiff = currentStatus.network.bytesOut - prevStatus.network.bytesOut;

    metrics.push({
      bytesInPerSec: bytesInDiff / intervalSeconds,
      bytesOutPerSec: bytesOutDiff / intervalSeconds,
      timestamp: new Date()
    });

    prevStatus = currentStatus;

    if (i < iterations - 1) {
      sleep(intervalSeconds * 1000);
    }
  }

  // Calcul des statistiques
  const bytesInValues = metrics.map(m => m.bytesInPerSec);
  const bytesOutValues = metrics.map(m => m.bytesOutPerSec);

  const avgBytesIn = (bytesInValues.reduce((a, b) => a + b, 0) / bytesInValues.length / 1024).toFixed(2);
  const maxBytesIn = (Math.max(...bytesInValues) / 1024).toFixed(2);
  const minBytesIn = (Math.min(...bytesInValues) / 1024).toFixed(2);

  const avgBytesOut = (bytesOutValues.reduce((a, b) => a + b, 0) / bytesOutValues.length / 1024).toFixed(2);
  const maxBytesOut = (Math.max(...bytesOutValues) / 1024).toFixed(2);
  const minBytesOut = (Math.min(...bytesOutValues) / 1024).toFixed(2);

  print("\n=== Network Metrics Statistics ===\n");
  print("Bytes In:");
  print(`  Average: ${avgBytesIn} KB/s`);
  print(`  Maximum: ${maxBytesIn} KB/s`);
  print(`  Minimum: ${minBytesIn} KB/s`);

  print("\nBytes Out:");
  print(`  Average: ${avgBytesOut} KB/s`);
  print(`  Maximum: ${maxBytesOut} KB/s`);
  print(`  Minimum: ${minBytesOut} KB/s`);
}

networkMetricsWithStats(5, 6);
```

---

### 6.3 - Monitoring compact des opérations

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 VUE COMPACT DES OPÉRATIONS
// ============================================

// 💡 Objectif
// Vue synthétique et mise à jour automatique des opérations

// 🎯 Cas d'usage
// - Monitoring rapide
// - Terminal secondaire
// - Vue d'ensemble

// ============================================
// SCRIPT
// ============================================

function compactOpsMonitor(intervalSeconds = 3, iterations = 20) {
  print("Compact operations monitor - Refreshing every " + intervalSeconds + "s\n");

  for (let i = 0; i < iterations; i++) {
    const timestamp = new Date().toISOString().substr(11, 8);
    const ops = db.currentOp({ "$all": false, "active": true });
    const status = db.serverStatus();

    // Compter par type
    const counts = {
      query: 0,
      insert: 0,
      update: 0,
      delete: 0,
      command: 0,
      getmore: 0,
      other: 0,
      slow: 0
    };

    ops.inprog.forEach(function(op) {
      const type = op.op || "other";
      if (counts.hasOwnProperty(type)) {
        counts[type]++;
      } else {
        counts.other++;
      }

      if (op.secs_running && op.secs_running >= 5) {
        counts.slow++;
      }
    });

    // Format compact
    const connUsage = (status.connections.current / (status.connections.current + status.connections.available) * 100).toFixed(0);

    let line = `${timestamp} | `;
    line += `Active:${ops.inprog.length.toString().padStart(3)} `;
    line += `(Q:${counts.query} I:${counts.insert} U:${counts.update} D:${counts.delete}) `;
    line += `| Conn:${connUsage}% `;
    line += `| Mem:${status.mem.resident}MB`;

    if (counts.slow > 0) {
      line += ` | ⚠️ ${counts.slow} SLOW`;
    }

    print(line);

    if (i < iterations - 1) {
      sleep(intervalSeconds * 1000);
    }
  }
}

compactOpsMonitor(3, 20);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
Compact operations monitor - Refreshing every 3s

10:30:00 | Active:  8 (Q:5 I:0 U:2 D:0) | Conn:0% | Mem:1250MB
10:30:03 | Active: 12 (Q:8 I:2 U:1 D:1) | Conn:0% | Mem:1252MB
10:30:06 | Active:  6 (Q:4 I:0 U:2 D:0) | Conn:0% | Mem:1248MB
10:30:09 | Active: 25 (Q:20 I:3 U:2 D:0) | Conn:1% | Mem:1280MB | ⚠️ 3 SLOW
*/
```

---

## Alerting et Seuils

### 7.1 - Script d'alerting complet

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 SYSTÈME D'ALERTING COMPLET
// ============================================

// 💡 Objectif
// Vérifier tous les seuils critiques et générer des alertes

// 🎯 Cas d'usage
// - Monitoring proactif
// - Prévention de pannes
// - Automatisation des alertes

// ============================================
// CONFIGURATION DES SEUILS
// ============================================

const thresholds = {
  cache: {
    warning: 80,
    critical: 90
  },
  connections: {
    warning: 60,
    critical: 80
  },
  replicationLag: {
    warning: 10,
    critical: 30
  },
  slowOperations: {
    warning: 5,
    critical: 30
  },
  diskUsage: {
    warning: 70,
    critical: 85
  }
};

// ============================================
// SCRIPT D'ALERTING
// ============================================

function healthCheck() {
  const alerts = {
    critical: [],
    warning: [],
    info: []
  };

  print("\n" + "=".repeat(60));
  print("MONGODB HEALTH CHECK");
  print(new Date().toISOString());
  print("=".repeat(60));

  try {
    const status = db.serverStatus();

    // 1. Cache WiredTiger
    if (status.wiredTiger && status.wiredTiger.cache) {
      const cache = status.wiredTiger.cache;
      const cacheUsage = (cache["bytes currently in the cache"] / cache["maximum bytes configured"]) * 100;

      if (cacheUsage >= thresholds.cache.critical) {
        alerts.critical.push(`Cache usage: ${cacheUsage.toFixed(2)}% (>=${thresholds.cache.critical}%)`);
      } else if (cacheUsage >= thresholds.cache.warning) {
        alerts.warning.push(`Cache usage: ${cacheUsage.toFixed(2)}% (>=${thresholds.cache.warning}%)`);
      } else {
        alerts.info.push(`Cache usage: ${cacheUsage.toFixed(2)}% - OK`);
      }
    }

    // 2. Connexions
    const connUsage = (status.connections.current / (status.connections.current + status.connections.available)) * 100;

    if (connUsage >= thresholds.connections.critical) {
      alerts.critical.push(`Connection usage: ${connUsage.toFixed(2)}% (>=${thresholds.connections.critical}%)`);
    } else if (connUsage >= thresholds.connections.warning) {
      alerts.warning.push(`Connection usage: ${connUsage.toFixed(2)}% (>=${thresholds.connections.warning}%)`);
    } else {
      alerts.info.push(`Connection usage: ${connUsage.toFixed(2)}% - OK`);
    }

    // 3. Opérations lentes
    const slowOps = db.currentOp({
      "$all": false,
      "active": true,
      "secs_running": { "$gte": thresholds.slowOperations.warning }
    });

    if (slowOps.inprog.length > 0) {
      const maxDuration = Math.max(...slowOps.inprog.map(op => op.secs_running));

      if (maxDuration >= thresholds.slowOperations.critical) {
        alerts.critical.push(`${slowOps.inprog.length} slow operation(s), max ${maxDuration}s`);
      } else {
        alerts.warning.push(`${slowOps.inprog.length} slow operation(s), max ${maxDuration}s`);
      }
    } else {
      alerts.info.push("No slow operations - OK");
    }

    // 4. Replication Lag (si Replica Set)
    try {
      const rsStatus = rs.status();
      const primary = rsStatus.members.find(m => m.state === 1);

      if (primary) {
        rsStatus.members.forEach(function(member) {
          if (member.state === 2) {  // SECONDARY
            const lagSeconds = Math.round((rsStatus.date - member.optimeDate) / 1000);

            if (lagSeconds >= thresholds.replicationLag.critical) {
              alerts.critical.push(`${member.name}: lag ${lagSeconds}s (>=${thresholds.replicationLag.critical}s)`);
            } else if (lagSeconds >= thresholds.replicationLag.warning) {
              alerts.warning.push(`${member.name}: lag ${lagSeconds}s (>=${thresholds.replicationLag.warning}s)`);
            } else {
              alerts.info.push(`${member.name}: lag ${lagSeconds}s - OK`);
            }
          }
        });
      }
    } catch (e) {
      // Not a replica set or not connected to one
      alerts.info.push("Not running in replica set mode");
    }

    // 5. Profiler - Requêtes lentes récentes
    const slowQueries = db.system.profile.find({
      ts: { $gte: new Date(Date.now() - 300000) },  // Last 5 minutes
      millis: { $gte: 1000 }
    }).count();

    if (slowQueries > 10) {
      alerts.warning.push(`${slowQueries} slow queries in last 5 minutes`);
    } else if (slowQueries > 0) {
      alerts.info.push(`${slowQueries} slow queries in last 5 minutes`);
    }

  } catch (e) {
    alerts.critical.push(`Health check failed: ${e.message}`);
  }

  // Affichage des résultats
  print("\n--- CRITICAL ALERTS ---");
  if (alerts.critical.length === 0) {
    print("None");
  } else {
    alerts.critical.forEach(alert => print(`❌ ${alert}`));
  }

  print("\n--- WARNINGS ---");
  if (alerts.warning.length === 0) {
    print("None");
  } else {
    alerts.warning.forEach(alert => print(`⚠️ ${alert}`));
  }

  print("\n--- INFO ---");
  alerts.info.forEach(alert => print(`ℹ️ ${alert}`));

  print("\n" + "=".repeat(60));

  // Statut global
  if (alerts.critical.length > 0) {
    print("❌ HEALTH STATUS: CRITICAL");
    return "CRITICAL";
  } else if (alerts.warning.length > 0) {
    print("⚠️ HEALTH STATUS: WARNING");
    return "WARNING";
  } else {
    print("✅ HEALTH STATUS: HEALTHY");
    return "HEALTHY";
  }
}

healthCheck();

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
============================================================
MONGODB HEALTH CHECK
2024-01-15T10:30:00.000Z
============================================================

--- CRITICAL ALERTS ---
None

--- WARNINGS ---
⚠️ Cache usage: 85.50% (>=80%)
⚠️ 2 slow operation(s), max 15s

--- INFO ---
ℹ️ Connection usage: 5.23% - OK
ℹ️ mongodb2:27017: lag 2s - OK
ℹ️ mongodb3:27017: lag 3s - OK
ℹ️ 3 slow queries in last 5 minutes

============================================================
⚠️ HEALTH STATUS: WARNING
*/
```

---

**💡 Conseil** : Automatisez ces requêtes de monitoring dans des scripts cron ou des outils de monitoring comme Prometheus, Grafana, ou Nagios pour un suivi continu de votre infrastructure MongoDB.

⏭️ [Pipelines d'agrégation courants](/annexes/requetes-reference/03-pipelines-agregation-courants.md)
