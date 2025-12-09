🔝 Retour au [Sommaire](/SOMMAIRE.md)

# C.1 - Requêtes d'Administration

## Table des matières

1. [Gestion des Index](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#gestion-des-index)
2. [Statistiques de Collections](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#statistiques-de-collections)
3. [Statistiques de Base de Données](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#statistiques-de-base-de-donn%C3%A9es)
4. [Monitoring du Serveur](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#monitoring-du-serveur)
5. [Audit des Utilisateurs](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#audit-des-utilisateurs)
6. [Replica Set](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#replica-set)
7. [Sharding](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#sharding)
8. [Maintenance et Nettoyage](https://github.com/NDXDeveloper/testdoc/edit/main/README.md#maintenance-et-nettoyage)

---

## Gestion des Index

### 1.1 - Lister tous les index d'une collection

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 LISTER TOUS LES INDEX
// ============================================

// 💡 Objectif
// Obtenir la liste complète des index d'une collection avec leurs détails

// 🎯 Cas d'usage
// - Audit des index existants
// - Documentation de la structure
// - Vérification avant suppression

// ============================================
// REQUÊTE
// ============================================

db.users.getIndexes()

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    v: 2,
    key: { _id: 1 },
    name: '_id_'
  },
  {
    v: 2,
    key: { email: 1 },
    name: 'email_1',
    unique: true
  },
  {
    v: 2,
    key: { name: 1, age: -1 },
    name: 'name_1_age_-1'
  }
]

// ============================================
// 💡 VARIANTES
// ============================================

// Lister les index de toutes les collections
db.getCollectionNames().forEach(function(col) {
  print(`\n=== ${col} ===`);
  printjson(db[col].getIndexes());
});

// Compter les index par collection
db.getCollectionNames().forEach(function(col) {
  const count = db[col].getIndexes().length;
  print(`${col}: ${count} index`);
});
```

---

### 1.2 - Identifier les index inutilisés

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 INDEX INUTILISÉS
// ============================================

// 💡 Objectif
// Trouver les index qui n'ont jamais été utilisés depuis leur création
// ou depuis le dernier redémarrage du serveur

// 🎯 Cas d'usage
// - Optimisation de l'espace disque
// - Amélioration des performances d'écriture
// - Nettoyage de la base

// ⚠️ Important
// Les statistiques sont réinitialisées au redémarrage du serveur

// ============================================
// REQUÊTE
// ============================================

db.users.aggregate([
  { $indexStats: {} },
  {
    $match: {
      "accesses.ops": { $eq: 0 }
    }
  },
  {
    $project: {
      name: 1,
      key: 1,
      "accesses.ops": 1,
      "accesses.since": 1
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  {
    name: "old_index_name",
    key: { obsoleteField: 1 },
    accesses: {
      ops: 0,
      since: ISODate("2024-01-01T00:00:00Z")
    }
  }
]

// ============================================
// 💡 VARIANTES
// ============================================

// Index peu utilisés (< 100 utilisations)
db.users.aggregate([
  { $indexStats: {} },
  {
    $match: {
      "accesses.ops": { $lt: 100 }
    }
  },
  { $sort: { "accesses.ops": 1 } }
])

// Rapport complet d'utilisation des index
db.users.aggregate([
  { $indexStats: {} },
  {
    $project: {
      name: 1,
      usageCount: "$accesses.ops",
      since: "$accesses.since",
      daysSince: {
        $divide: [
          { $subtract: [new Date(), "$accesses.since"] },
          86400000  // ms par jour
        ]
      }
    }
  },
  { $sort: { usageCount: 1 } }
])

// ============================================
// 🔧 ACTION RECOMMANDÉE
// ============================================

// Après identification, supprimer les index inutilisés
// db.users.dropIndex("index_name")
```

---

### 1.3 - Analyser la taille des index

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 TAILLE DES INDEX
// ============================================

// 💡 Objectif
// Calculer l'espace disque utilisé par chaque index

// 🎯 Cas d'usage
// - Optimisation de l'espace disque
// - Planification de la capacité
// - Identification des index volumineux

// ============================================
// REQUÊTE
// ============================================

// Taille des index d'une collection
const stats = db.users.stats(1024 * 1024);  // En MB

print(`Collection: ${stats.ns}`);
print(`Total index size: ${stats.totalIndexSize.toFixed(2)} MB\n`);

Object.keys(stats.indexSizes).forEach(function(indexName) {
  const sizeMB = stats.indexSizes[indexName] / (1024 * 1024);
  print(`${indexName}: ${sizeMB.toFixed(2)} MB`);
});

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
Collection: myapp.users
Total index size: 125.50 MB

_id_: 45.20 MB
email_1: 38.15 MB
name_1_age_-1: 42.15 MB
*/

// ============================================
// 💡 VARIANTES
// ============================================

// Rapport complet toutes collections
db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024);

  if (stats.totalIndexSize > 0) {
    print(`\n=== ${col} ===`);
    print(`Data: ${stats.size.toFixed(2)} MB`);
    print(`Indexes: ${stats.totalIndexSize.toFixed(2)} MB`);

    const ratio = (stats.totalIndexSize / stats.size * 100).toFixed(2);
    print(`Ratio: ${ratio}%`);
  }
});

// Index les plus volumineux du serveur
const indexSizes = [];

db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024);

  Object.keys(stats.indexSizes).forEach(function(indexName) {
    const sizeMB = stats.indexSizes[indexName] / (1024 * 1024);
    indexSizes.push({
      collection: col,
      index: indexName,
      sizeMB: sizeMB
    });
  });
});

// Trier et afficher top 10
indexSizes
  .sort((a, b) => b.sizeMB - a.sizeMB)
  .slice(0, 10)
  .forEach(item => {
    print(`${item.collection}.${item.index}: ${item.sizeMB.toFixed(2)} MB`);
  });
```

---

### 1.4 - Vérifier les index manquants

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 INDEX MANQUANTS (SUGGESTIONS)
// ============================================

// 💡 Objectif
// Analyser les requêtes lentes pour identifier les index manquants

// 🎯 Cas d'usage
// - Optimisation des performances
// - Réduction du temps de réponse
// - Planification des index

// ⚠️ Important
// Nécessite le profiler activé (niveau 1 ou 2)

// ============================================
// PRÉREQUIS
// ============================================

// Activer le profiler (si pas déjà fait)
db.setProfilingLevel(1, 100);  // Requêtes > 100ms

// ============================================
// REQUÊTE
// ============================================

// Requêtes lentes avec COLLSCAN (scan de collection)
db.system.profile.aggregate([
  {
    $match: {
      ns: "myapp.users",
      planSummary: "COLLSCAN",
      millis: { $gt: 100 }
    }
  },
  {
    $group: {
      _id: "$command.filter",
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      maxMillis: { $max: "$millis" }
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
    _id: { status: "active", age: { $gte: 18 } },
    count: 1523,
    avgMillis: 250.5,
    maxMillis: 1200
  },
  {
    _id: { email: /gmail/ },
    count: 842,
    avgMillis: 180.3,
    maxMillis: 890
  }
]

// ============================================
// 💡 ANALYSE ET RECOMMANDATIONS
// ============================================

// Pour le premier cas : créer un index composé
// db.users.createIndex({ status: 1, age: 1 })

// Pour le second cas : créer un index texte
// db.users.createIndex({ email: "text" })
// Ou pour une recherche exacte : index simple
// db.users.createIndex({ email: 1 })

// ============================================
// 🔧 VARIANTE : Analyse automatique
// ============================================

// Script d'analyse automatique
function analyzeSlowQueries(collectionName, threshold = 100) {
  const results = db.system.profile.aggregate([
    {
      $match: {
        ns: `${db.getName()}.${collectionName}`,
        planSummary: { $in: ["COLLSCAN", "IXSCAN"] },
        millis: { $gt: threshold }
      }
    },
    {
      $group: {
        _id: {
          filter: "$command.filter",
          sort: "$command.sort",
          planSummary: "$planSummary"
        },
        count: { $sum: 1 },
        avgTime: { $avg: "$millis" }
      }
    },
    { $sort: { count: -1 } }
  ]).toArray();

  print(`\n=== Slow queries on ${collectionName} ===\n`);

  results.forEach(function(result) {
    print(`Occurrences: ${result.count}`);
    print(`Avg time: ${result.avgTime.toFixed(2)}ms`);
    print(`Plan: ${result._id.planSummary}`);
    print(`Filter: ${JSON.stringify(result._id.filter)}`);

    if (result._id.sort) {
      print(`Sort: ${JSON.stringify(result._id.sort)}`);
    }

    if (result._id.planSummary === "COLLSCAN") {
      print("⚠️ RECOMMENDATION: Consider creating an index");
    }

    print("---\n");
  });
}

// Usage
analyzeSlowQueries("users", 100);
```

---

## Statistiques de Collections

### 2.1 - Statistiques complètes d'une collection

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 STATISTIQUES DE COLLECTION
// ============================================

// 💡 Objectif
// Obtenir des métriques détaillées sur une collection

// 🎯 Cas d'usage
// - Monitoring de la croissance
// - Planification de capacité
// - Diagnostic de performance

// ============================================
// REQUÊTE
// ============================================

db.users.stats(1024 * 1024)  // Tailles en MB

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  ns: "myapp.users",
  size: 524.288,              // Taille des données (MB)
  count: 1000000,             // Nombre de documents
  avgObjSize: 549,            // Taille moyenne (octets)
  storageSize: 262.144,       // Espace disque alloué (MB)
  nindexes: 5,                // Nombre d'index
  totalIndexSize: 104.858,    // Taille totale des index (MB)
  indexSizes: {
    _id_: 41.943,
    email_1: 31.457,
    name_1_age_-1: 31.458
  },
  ok: 1
}

// ============================================
// 💡 VARIANTES
// ============================================

// Rapport formaté
function collectionReport(collectionName) {
  const stats = db[collectionName].stats(1024 * 1024);

  print(`\n=== ${collectionName} ===`);
  print(`Documents: ${stats.count.toLocaleString()}`);
  print(`Data size: ${stats.size.toFixed(2)} MB`);
  print(`Storage size: ${stats.storageSize.toFixed(2)} MB`);
  print(`Avg doc size: ${stats.avgObjSize} bytes`);
  print(`Indexes: ${stats.nindexes}`);
  print(`Index size: ${stats.totalIndexSize.toFixed(2)} MB`);

  const compressionRatio = ((1 - stats.storageSize / stats.size) * 100).toFixed(2);
  print(`Compression: ${compressionRatio}%`);

  const overhead = ((stats.totalIndexSize / stats.size) * 100).toFixed(2);
  print(`Index overhead: ${overhead}%`);
}

collectionReport("users");

// Statistiques de toutes les collections
db.getCollectionNames().forEach(function(col) {
  collectionReport(col);
});
```

---

### 2.2 - Collections les plus volumineuses

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 COLLECTIONS PAR TAILLE
// ============================================

// 💡 Objectif
// Identifier les collections qui consomment le plus d'espace

// 🎯 Cas d'usage
// - Planification du stockage
// - Identification des candidats à l'archivage
// - Optimisation de l'espace

// ============================================
// REQUÊTE
// ============================================

const collections = [];

db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024);

  collections.push({
    name: col,
    count: stats.count,
    sizeMB: stats.size,
    storageMB: stats.storageSize,
    indexesMB: stats.totalIndexSize,
    totalMB: stats.storageSize + stats.totalIndexSize
  });
});

// Trier par taille totale
collections.sort((a, b) => b.totalMB - a.totalMB);

// Afficher top 10
print("\n=== Top 10 collections by size ===\n");
print("Collection".padEnd(30) + "Docs".padEnd(15) + "Data".padEnd(12) + "Indexes".padEnd(12) + "Total");
print("-".repeat(80));

collections.slice(0, 10).forEach(function(col) {
  print(
    col.name.padEnd(30) +
    col.count.toLocaleString().padEnd(15) +
    `${col.sizeMB.toFixed(2)} MB`.padEnd(12) +
    `${col.indexesMB.toFixed(2)} MB`.padEnd(12) +
    `${col.totalMB.toFixed(2)} MB`
  );
});

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== Top 10 collections by size ===

Collection                    Docs           Data        Indexes     Total
--------------------------------------------------------------------------------
logs                          50000000       15360.00 MB 5120.00 MB  20480.00 MB
orders                        10000000       5120.00 MB  2048.00 MB  7168.00 MB
users                         1000000        524.29 MB   104.86 MB   629.15 MB
*/

// ============================================
// 💡 VARIANTES
// ============================================

// Collections par nombre de documents
collections.sort((a, b) => b.count - a.count);

// Collections par ratio index/data
collections.forEach(col => {
  col.indexRatio = (col.indexesMB / col.sizeMB * 100).toFixed(2);
});
collections.sort((a, b) => b.indexRatio - a.indexRatio);
```

---

### 2.3 - Croissance des collections

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 ANALYSE DE CROISSANCE
// ============================================

// 💡 Objectif
// Analyser la croissance des collections sur une période

// 🎯 Cas d'usage
// - Planification de capacité
// - Prédiction de la croissance
// - Identification des tendances

// ⚠️ Prérequis
// Nécessite un champ de date (createdAt, timestamp, etc.)

// ============================================
// REQUÊTE
// ============================================

// Croissance par mois (derniers 12 mois)
db.users.aggregate([
  {
    $match: {
      createdAt: {
        $gte: new Date(new Date().setMonth(new Date().getMonth() - 12))
      }
    }
  },
  {
    $group: {
      _id: {
        year: { $year: "$createdAt" },
        month: { $month: "$createdAt" }
      },
      count: { $sum: 1 }
    }
  },
  {
    $sort: { "_id.year": 1, "_id.month": 1 }
  },
  {
    $project: {
      _id: 0,
      period: {
        $concat: [
          { $toString: "$_id.year" },
          "-",
          {
            $cond: [
              { $lt: ["$_id.month", 10] },
              { $concat: ["0", { $toString: "$_id.month" }] },
              { $toString: "$_id.month" }
            ]
          }
        ]
      },
      count: 1
    }
  }
])

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

[
  { period: "2024-01", count: 45234 },
  { period: "2024-02", count: 52100 },
  { period: "2024-03", count: 58345 },
  { period: "2024-04", count: 61250 },
  // ...
]

// ============================================
// 💡 VARIANTES
// ============================================

// Croissance par semaine
db.users.aggregate([
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 90 * 86400000) }  // 90 jours
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-W%V", date: "$createdAt" }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])

// Croissance par jour (dernier mois)
db.users.aggregate([
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 30 * 86400000) }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d", date: "$createdAt" }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])

// Statistiques de croissance globale
db.users.aggregate([
  {
    $facet: {
      total: [
        { $count: "count" }
      ],
      last30days: [
        {
          $match: {
            createdAt: { $gte: new Date(Date.now() - 30 * 86400000) }
          }
        },
        { $count: "count" }
      ],
      last7days: [
        {
          $match: {
            createdAt: { $gte: new Date(Date.now() - 7 * 86400000) }
          }
        },
        { $count: "count" }
      ],
      today: [
        {
          $match: {
            createdAt: {
              $gte: new Date(new Date().setHours(0, 0, 0, 0))
            }
          }
        },
        { $count: "count" }
      ]
    }
  },
  {
    $project: {
      total: { $arrayElemAt: ["$total.count", 0] },
      last30days: { $arrayElemAt: ["$last30days.count", 0] },
      last7days: { $arrayElemAt: ["$last7days.count", 0] },
      today: { $arrayElemAt: ["$today.count", 0] }
    }
  }
])
```

---

## Statistiques de Base de Données

### 3.1 - Vue d'ensemble de la base

**🟢 Niveau : Débutant** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 STATISTIQUES DE BASE DE DONNÉES
// ============================================

// 💡 Objectif
// Obtenir une vue d'ensemble des métriques de la base

// 🎯 Cas d'usage
// - Monitoring général
// - Rapports de capacité
// - Documentation

// ============================================
// REQUÊTE
// ============================================

db.stats(1024 * 1024)  // Tailles en MB

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  db: "myapp",
  collections: 12,
  views: 2,
  objects: 55234567,           // Total documents
  avgObjSize: 487,             // Taille moyenne
  dataSize: 26894.25,          // Taille des données (MB)
  storageSize: 13447.12,       // Espace disque (MB)
  indexes: 45,                 // Nombre total d'index
  indexSize: 5378.85,          // Taille totale des index (MB)
  totalSize: 18825.97,         // Total (storage + indexes)
  scaleFactor: 1048576,
  fsUsedSize: 450000,          // Espace disque utilisé (MB)
  fsTotalSize: 1000000,        // Espace disque total (MB)
  ok: 1
}

// ============================================
// 💡 RAPPORT FORMATÉ
// ============================================

function databaseReport() {
  const stats = db.stats(1024 * 1024);

  print("\n" + "=".repeat(60));
  print(`DATABASE: ${stats.db}`);
  print("=".repeat(60));

  print(`\nCollections: ${stats.collections}`);
  print(`Total documents: ${stats.objects.toLocaleString()}`);
  print(`Avg document size: ${stats.avgObjSize} bytes`);

  print(`\n--- Storage ---`);
  print(`Data size: ${stats.dataSize.toFixed(2)} MB`);
  print(`Storage size: ${stats.storageSize.toFixed(2)} MB`);
  print(`Index size: ${stats.indexSize.toFixed(2)} MB`);
  print(`Total size: ${stats.totalSize.toFixed(2)} MB`);

  const compression = ((1 - stats.storageSize / stats.dataSize) * 100).toFixed(2);
  print(`Compression ratio: ${compression}%`);

  const indexOverhead = ((stats.indexSize / stats.dataSize) * 100).toFixed(2);
  print(`Index overhead: ${indexOverhead}%`);

  print(`\n--- Disk Usage ---`);
  print(`Disk used: ${stats.fsUsedSize.toFixed(2)} MB`);
  print(`Disk total: ${stats.fsTotalSize.toFixed(2)} MB`);

  const diskUsage = ((stats.fsUsedSize / stats.fsTotalSize) * 100).toFixed(2);
  print(`Disk usage: ${diskUsage}%`);

  print("\n" + "=".repeat(60));
}

databaseReport();
```

---

### 3.2 - Comparaison entre bases de données

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 COMPARAISON DE BASES
// ============================================

// 💡 Objectif
// Comparer les métriques de plusieurs bases de données

// 🎯 Cas d'usage
// - Analyse multi-environnements
// - Planification de migration
// - Audit général

// ============================================
// REQUÊTE
// ============================================

const dbList = db.adminCommand('listDatabases');
const dbStats = [];

dbList.databases.forEach(function(database) {
  if (database.name !== 'admin' && database.name !== 'config' && database.name !== 'local') {
    const currentDb = db.getSiblingDB(database.name);
    const stats = currentDb.stats(1024 * 1024);

    dbStats.push({
      name: stats.db,
      collections: stats.collections,
      documents: stats.objects,
      dataMB: stats.dataSize,
      indexesMB: stats.indexSize,
      totalMB: stats.totalSize
    });
  }
});

// Trier par taille totale
dbStats.sort((a, b) => b.totalMB - a.totalMB);

// Afficher le rapport
print("\n=== Database Comparison ===\n");
print("Database".padEnd(20) + "Collections".padEnd(15) + "Documents".padEnd(15) + "Data (MB)".padEnd(12) + "Total (MB)");
print("-".repeat(75));

dbStats.forEach(function(db) {
  print(
    db.name.padEnd(20) +
    db.collections.toString().padEnd(15) +
    db.documents.toLocaleString().padEnd(15) +
    db.dataMB.toFixed(2).padEnd(12) +
    db.totalMB.toFixed(2)
  );
});

// Totaux
const totalData = dbStats.reduce((sum, db) => sum + db.dataMB, 0);
const totalSize = dbStats.reduce((sum, db) => sum + db.totalMB, 0);
const totalDocs = dbStats.reduce((sum, db) => sum + db.documents, 0);

print("-".repeat(75));
print(`TOTAL:`.padEnd(20) + `${dbStats.length}`.padEnd(15) + totalDocs.toLocaleString().padEnd(15) + totalData.toFixed(2).padEnd(12) + totalSize.toFixed(2));

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== Database Comparison ===

Database            Collections    Documents      Data (MB)   Total (MB)
---------------------------------------------------------------------------
production          25             15234567       12589.45    15234.12
staging             18             2456789        2045.67     2489.34
development         12             345678         289.12      356.78
---------------------------------------------------------------------------
TOTAL:              3              18036034       14924.24    18080.24
*/
```

---

## Monitoring du Serveur

### 4.1 - État général du serveur

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 ÉTAT DU SERVEUR
// ============================================

// 💡 Objectif
// Obtenir les métriques essentielles du serveur MongoDB

// 🎯 Cas d'usage
// - Monitoring temps réel
// - Health check
// - Diagnostic rapide

// ============================================
// REQUÊTE
// ============================================

const status = db.serverStatus();

// Métriques essentielles
const metrics = {
  version: db.version(),
  uptime: (status.uptime / 3600).toFixed(2) + " hours",
  connections: {
    current: status.connections.current,
    available: status.connections.available,
    usage: ((status.connections.current / (status.connections.current + status.connections.available)) * 100).toFixed(2) + "%"
  },
  memory: {
    resident: status.mem.resident + " MB",
    virtual: status.mem.virtual + " MB",
    mapped: status.mem.mapped || "N/A"
  },
  operations: {
    insert: status.opcounters.insert,
    query: status.opcounters.query,
    update: status.opcounters.update,
    delete: status.opcounters.delete,
    command: status.opcounters.command
  },
  network: {
    bytesIn: (status.network.bytesIn / (1024 * 1024 * 1024)).toFixed(2) + " GB",
    bytesOut: (status.network.bytesOut / (1024 * 1024 * 1024)).toFixed(2) + " GB",
    requests: status.network.numRequests
  }
};

printjson(metrics);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  version: "7.0.5",
  uptime: "168.50 hours",
  connections: {
    current: 52,
    available: 838808,
    usage: "0.01%"
  },
  memory: {
    resident: "1250 MB",
    virtual: "2500 MB"
  },
  operations: {
    insert: 1234567,
    query: 5678901,
    update: 234567,
    delete: 12345,
    command: 8901234
  },
  network: {
    bytesIn: "125.50 GB",
    bytesOut: "89.25 GB",
    requests: 15234567
  }
}

// ============================================
// 💡 RAPPORT FORMATÉ
// ============================================

function serverHealthReport() {
  const status = db.serverStatus();

  print("\n" + "=".repeat(60));
  print("SERVER HEALTH REPORT");
  print("=".repeat(60));

  print(`\nVersion: ${db.version()}`);
  print(`Uptime: ${(status.uptime / 3600).toFixed(2)} hours`);
  print(`Host: ${status.host}`);

  print(`\n--- Connections ---`);
  print(`Current: ${status.connections.current}`);
  print(`Available: ${status.connections.available}`);
  const connUsage = (status.connections.current / (status.connections.current + status.connections.available) * 100).toFixed(2);
  print(`Usage: ${connUsage}%`);

  print(`\n--- Memory ---`);
  print(`Resident: ${status.mem.resident} MB`);
  print(`Virtual: ${status.mem.virtual} MB`);

  print(`\n--- Operations (total) ---`);
  print(`Insert: ${status.opcounters.insert.toLocaleString()}`);
  print(`Query: ${status.opcounters.query.toLocaleString()}`);
  print(`Update: ${status.opcounters.update.toLocaleString()}`);
  print(`Delete: ${status.opcounters.delete.toLocaleString()}`);

  print(`\n--- Network ---`);
  print(`Bytes In: ${(status.network.bytesIn / (1024 * 1024 * 1024)).toFixed(2)} GB`);
  print(`Bytes Out: ${(status.network.bytesOut / (1024 * 1024 * 1024)).toFixed(2)} GB`);
  print(`Requests: ${status.network.numRequests.toLocaleString()}`);

  if (status.repl) {
    print(`\n--- Replication ---`);
    print(`Set name: ${status.repl.setName}`);
    print(`Is master: ${status.repl.ismaster}`);
  }

  print("\n" + "=".repeat(60));
}

serverHealthReport();
```

---

### 4.2 - Utilisation de la mémoire WiredTiger

**🔴 Niveau : Avancé** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 CACHE WIREDTIGER
// ============================================

// 💡 Objectif
// Analyser l'utilisation du cache WiredTiger

// 🎯 Cas d'usage
// - Optimisation de la mémoire
// - Diagnostic de performance
// - Dimensionnement du cache

// ============================================
// REQUÊTE
// ============================================

const status = db.serverStatus();
const wt = status.wiredTiger.cache;

const cacheStats = {
  maxCacheSize: (wt["maximum bytes configured"] / (1024 * 1024 * 1024)).toFixed(2) + " GB",
  currentCacheSize: (wt["bytes currently in the cache"] / (1024 * 1024 * 1024)).toFixed(2) + " GB",
  dirtyData: (wt["tracked dirty bytes in the cache"] / (1024 * 1024)).toFixed(2) + " MB",
  pagesRead: wt["pages read into cache"],
  pagesWritten: wt["pages written from cache"],
  evictions: wt["unmodified pages evicted"],
  usage: ((wt["bytes currently in the cache"] / wt["maximum bytes configured"]) * 100).toFixed(2) + "%",
  hitRatio: ((wt["pages read into cache"] / (wt["pages read into cache"] + wt["pages requested from the cache"])) * 100).toFixed(2) + "%"
};

printjson(cacheStats);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  maxCacheSize: "2.00 GB",
  currentCacheSize: "1.75 GB",
  dirtyData: "128.50 MB",
  pagesRead: 1234567,
  pagesWritten: 234567,
  evictions: 12345,
  usage: "87.50%",
  hitRatio: "95.23%"
}

// ============================================
// 💡 INTERPRÉTATION
// ============================================

/*
- Usage > 80% : Cache bien utilisé
- Hit ratio > 90% : Excellente performance
- Hit ratio < 80% : Considérer augmenter le cache
- Evictions élevées : Cache trop petit ou working set trop grand
*/

// ============================================
// 🔧 RECOMMANDATIONS
// ============================================

function analyzeCacheHealth() {
  const status = db.serverStatus();
  const cache = status.wiredTiger.cache;

  const usage = (cache["bytes currently in the cache"] / cache["maximum bytes configured"]) * 100;
  const hitRatio = (cache["pages read into cache"] / (cache["pages read into cache"] + cache["pages requested from the cache"])) * 100;

  print("\n=== Cache Health Analysis ===\n");

  if (usage > 90) {
    print("⚠️ Cache usage is very high (>90%)");
    print("   Consider increasing cache size");
  } else if (usage > 80) {
    print("✅ Cache usage is healthy (80-90%)");
  } else {
    print("ℹ️ Cache usage is low (<80%)");
    print("   Working set might be smaller than cache");
  }

  if (hitRatio > 95) {
    print("✅ Excellent cache hit ratio (>95%)");
  } else if (hitRatio > 85) {
    print("✅ Good cache hit ratio (85-95%)");
  } else {
    print("⚠️ Low cache hit ratio (<85%)");
    print("   Consider increasing cache size or reviewing queries");
  }

  const evictionRate = cache["unmodified pages evicted"] / cache["pages read into cache"];
  if (evictionRate > 0.1) {
    print("⚠️ High eviction rate detected");
    print("   Cache might be undersized for your working set");
  }
}

analyzeCacheHealth();
```

---

## Audit des Utilisateurs

### 5.1 - Lister tous les utilisateurs et leurs rôles

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 AUDIT DES UTILISATEURS
// ============================================

// 💡 Objectif
// Obtenir la liste complète des utilisateurs avec leurs permissions

// 🎯 Cas d'usage
// - Audit de sécurité
// - Documentation des accès
// - Conformité

// ⚠️ Prérequis
// Nécessite des privilèges d'admin

// ============================================
// REQUÊTE
// ============================================

// Tous les utilisateurs du serveur
const users = db.getSiblingDB("admin").system.users.find().toArray();

print("\n=== User Audit Report ===\n");
print("Username".padEnd(25) + "Database".padEnd(20) + "Roles");
print("-".repeat(80));

users.forEach(function(user) {
  const roles = user.roles.map(r => `${r.role}@${r.db}`).join(", ");
  print(user.user.padEnd(25) + user.db.padEnd(20) + roles);
});

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== User Audit Report ===

Username                 Database            Roles
--------------------------------------------------------------------------------
admin                    admin               root@admin
appUser                  myapp               readWrite@myapp
readOnlyUser             myapp               read@myapp
backupUser               admin               backup@admin
*/

// ============================================
// 💡 ANALYSE DÉTAILLÉE
// ============================================

// Rapport détaillé avec privilèges
function detailedUserAudit() {
  const adminDb = db.getSiblingDB("admin");
  const users = adminDb.system.users.find().toArray();

  users.forEach(function(user) {
    print(`\n${"=".repeat(60)}`);
    print(`User: ${user.user}`);
    print(`Database: ${user.db}`);
    print(`User ID: ${user.userId}`);
    print(`\nRoles:`);

    user.roles.forEach(function(role) {
      print(`  - ${role.role} on ${role.db}`);

      // Obtenir les détails du rôle
      const roleDetails = adminDb.getRole(role.role, {
        db: role.db,
        showPrivileges: true
      });

      if (roleDetails && roleDetails.privileges) {
        print(`    Privileges:`);
        roleDetails.privileges.forEach(function(priv) {
          const resource = priv.resource.db ?
            `${priv.resource.db}.${priv.resource.collection || '*'}` :
            'cluster';
          print(`      ${resource}: ${priv.actions.join(', ')}`);
        });
      }
    });
  });
}

// Usage (peut être verbeux)
// detailedUserAudit();

// ============================================
// 💡 STATISTIQUES
// ============================================

function userStatistics() {
  const users = db.getSiblingDB("admin").system.users.find().toArray();
  const stats = {
    totalUsers: users.length,
    byDatabase: {},
    byRole: {}
  };

  users.forEach(function(user) {
    // Par base de données
    if (!stats.byDatabase[user.db]) {
      stats.byDatabase[user.db] = 0;
    }
    stats.byDatabase[user.db]++;

    // Par rôle
    user.roles.forEach(function(role) {
      if (!stats.byRole[role.role]) {
        stats.byRole[role.role] = 0;
      }
      stats.byRole[role.role]++;
    });
  });

  print("\n=== User Statistics ===\n");
  print(`Total users: ${stats.totalUsers}\n`);

  print("Users by database:");
  Object.keys(stats.byDatabase).forEach(db => {
    print(`  ${db}: ${stats.byDatabase[db]}`);
  });

  print("\nUsers by role:");
  Object.keys(stats.byRole).forEach(role => {
    print(`  ${role}: ${stats.byRole[role]}`);
  });
}

userStatistics();
```

---

### 5.2 - Identifier les utilisateurs avec privilèges élevés

**🔴 Niveau : Avancé** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 UTILISATEURS PRIVILÉGIÉS
// ============================================

// 💡 Objectif
// Identifier les utilisateurs ayant des rôles d'administration

// 🎯 Cas d'usage
// - Audit de sécurité
// - Conformité SOC2/ISO27001
// - Revue des accès

// ============================================
// REQUÊTE
// ============================================

const privilegedRoles = [
  "root",
  "dbAdminAnyDatabase",
  "userAdminAnyDatabase",
  "readWriteAnyDatabase",
  "clusterAdmin",
  "dbOwner"
];

const users = db.getSiblingDB("admin").system.users.find().toArray();
const privilegedUsers = [];

users.forEach(function(user) {
  const userPrivileged = user.roles.some(role =>
    privilegedRoles.includes(role.role)
  );

  if (userPrivileged) {
    privilegedUsers.push({
      user: user.user,
      db: user.db,
      roles: user.roles.map(r => `${r.role}@${r.db}`)
    });
  }
});

print("\n=== Privileged Users Report ===\n");

if (privilegedUsers.length === 0) {
  print("No privileged users found (good!)");
} else {
  print(`⚠️ Found ${privilegedUsers.length} privileged users:\n`);

  privilegedUsers.forEach(function(user) {
    print(`User: ${user.user}@${user.db}`);
    print(`Roles: ${user.roles.join(", ")}`);
    print("---");
  });

  print("\n⚠️ Recommendation: Review and minimize privileged access");
}

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== Privileged Users Report ===

⚠️ Found 2 privileged users:

User: admin@admin
Roles: root@admin
---
User: dbadmin@admin
Roles: dbAdminAnyDatabase@admin, userAdminAnyDatabase@admin
---

⚠️ Recommendation: Review and minimize privileged access
*/
```

---

## Replica Set

### 6.1 - Santé du Replica Set

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 SANTÉ DU REPLICA SET
// ============================================

// 💡 Objectif
// Vérifier l'état de santé du Replica Set

// 🎯 Cas d'usage
// - Monitoring quotidien
// - Diagnostic de problèmes
// - Alerting

// ⚠️ Prérequis
// Nécessite un Replica Set configuré

// ============================================
// REQUÊTE
// ============================================

const status = rs.status();

// Résumé de santé
const health = {
  set: status.set,
  date: status.date,
  myState: status.myState,
  term: status.term,
  members: []
};

status.members.forEach(function(member) {
  health.members.push({
    name: member.name,
    state: member.stateStr,
    health: member.health === 1 ? "✅ Healthy" : "❌ Unhealthy",
    uptime: (member.uptime / 3600).toFixed(2) + "h",
    lag: member.optimeDate ?
      Math.round((status.date - member.optimeDate) / 1000) + "s" : "N/A",
    pingMs: member.pingMs || "N/A"
  });
});

printjson(health);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  set: "myReplicaSet",
  date: ISODate("2024-01-15T10:30:00Z"),
  myState: 1,
  term: 5,
  members: [
    {
      name: "mongodb1:27017",
      state: "PRIMARY",
      health: "✅ Healthy",
      uptime: "168.50h",
      lag: "0s",
      pingMs: "N/A"
    },
    {
      name: "mongodb2:27017",
      state: "SECONDARY",
      health: "✅ Healthy",
      uptime: "168.45h",
      lag: "1s",
      pingMs: 2
    },
    {
      name: "mongodb3:27017",
      state: "SECONDARY",
      health: "✅ Healthy",
      uptime: "168.40h",
      lag: "2s",
      pingMs: 3
    }
  ]
}

// ============================================
// 💡 ANALYSE AUTOMATIQUE
// ============================================

function analyzeReplicaSetHealth() {
  const status = rs.status();
  const issues = [];

  print("\n=== Replica Set Health Check ===\n");
  print(`Set: ${status.set}`);
  print(`Term: ${status.term}\n`);

  // Vérifier le Primary
  const primary = status.members.find(m => m.state === 1);
  if (!primary) {
    issues.push("❌ CRITICAL: No PRIMARY found!");
  } else {
    print(`✅ Primary: ${primary.name}`);
  }

  // Vérifier chaque membre
  status.members.forEach(function(member) {
    print(`\n${member.name}:`);
    print(`  State: ${member.stateStr}`);
    print(`  Health: ${member.health === 1 ? "✅" : "❌"}`);
    print(`  Uptime: ${(member.uptime / 3600).toFixed(2)}h`);

    if (member.health !== 1) {
      issues.push(`❌ ${member.name} is unhealthy`);
    }

    // Vérifier le lag
    if (member.state === 2 && member.optimeDate) {  // SECONDARY
      const lagSeconds = Math.round((status.date - member.optimeDate) / 1000);
      print(`  Lag: ${lagSeconds}s`);

      if (lagSeconds > 10) {
        issues.push(`⚠️ ${member.name} has high replication lag (${lagSeconds}s)`);
      }
    }
  });

  // Résumé
  print("\n" + "=".repeat(50));
  if (issues.length === 0) {
    print("✅ Replica Set is healthy");
  } else {
    print(`⚠️ ${issues.length} issue(s) detected:\n`);
    issues.forEach(issue => print(issue));
  }
  print("=".repeat(50));
}

analyzeReplicaSetHealth();
```

---

### 6.2 - Analyse du replication lag

**🔴 Niveau : Avancé** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 REPLICATION LAG
// ============================================

// 💡 Objectif
// Analyser le retard de réplication des Secondaries

// 🎯 Cas d'usage
// - Diagnostic de performance
// - Alerting sur le lag
// - Planification de maintenance

// ============================================
// REQUÊTE
// ============================================

const status = rs.status();
const primary = status.members.find(m => m.state === 1);

if (!primary) {
  print("❌ No PRIMARY found");
} else {
  const lagReport = [];

  status.members.forEach(function(member) {
    if (member.state === 2) {  // SECONDARY
      const lagSeconds = Math.round((status.date - member.optimeDate) / 1000);

      lagReport.push({
        member: member.name,
        state: member.stateStr,
        lagSeconds: lagSeconds,
        lagFormatted: lagSeconds < 60 ?
          `${lagSeconds}s` :
          `${Math.floor(lagSeconds / 60)}m ${lagSeconds % 60}s`,
        status: lagSeconds < 5 ? "✅ Excellent" :
                lagSeconds < 10 ? "✅ Good" :
                lagSeconds < 30 ? "⚠️ Warning" :
                "❌ Critical",
        primaryOptimeDate: primary.optimeDate,
        secondaryOptimeDate: member.optimeDate
      });
    }
  });

  print("\n=== Replication Lag Report ===\n");
  print(`Primary: ${primary.name}`);
  print(`Primary optime: ${primary.optimeDate}\n`);

  if (lagReport.length === 0) {
    print("No secondary members found");
  } else {
    lagReport.forEach(function(report) {
      print(`${report.member}:`);
      print(`  Lag: ${report.lagFormatted} (${report.lagSeconds}s)`);
      print(`  Status: ${report.status}`);
      print(`  Secondary optime: ${report.secondaryOptimeDate}`);
      print("");
    });
  }

  // Statistiques
  const lags = lagReport.map(r => r.lagSeconds);
  if (lags.length > 0) {
    const avgLag = (lags.reduce((a, b) => a + b, 0) / lags.length).toFixed(2);
    const maxLag = Math.max(...lags);

    print("=== Statistics ===");
    print(`Average lag: ${avgLag}s`);
    print(`Maximum lag: ${maxLag}s`);

    if (maxLag > 30) {
      print("\n⚠️ WARNING: High replication lag detected!");
      print("Possible causes:");
      print("  - Network issues");
      print("  - Secondary overloaded");
      print("  - Large write bursts");
      print("  - Slow disk I/O on secondary");
    }
  }
}

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== Replication Lag Report ===

Primary: mongodb1:27017
Primary optime: 2024-01-15T10:30:00.000Z

mongodb2:27017:
  Lag: 2s (2s)
  Status: ✅ Excellent
  Secondary optime: 2024-01-15T10:29:58.000Z

mongodb3:27017:
  Lag: 5s (5s)
  Status: ✅ Good
  Secondary optime: 2024-01-15T10:29:55.000Z

=== Statistics ===
Average lag: 3.50s
Maximum lag: 5s
*/
```

---

## Sharding

### 7.1 - État du cluster shardé

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 ÉTAT DU CLUSTER SHARDÉ
// ============================================

// 💡 Objectif
// Obtenir un résumé de l'état du cluster shardé

// 🎯 Cas d'usage
// - Monitoring du sharding
// - Diagnostic de distribution
// - Planification de scaling

// ⚠️ Prérequis
// Nécessite un cluster shardé et connexion via mongos

// ============================================
// REQUÊTE
// ============================================

const shardingStatus = db.getSiblingDB("config").shards.find().toArray();
const databases = db.getSiblingDB("config").databases.find().toArray();
const collections = db.getSiblingDB("config").collections.find().toArray();
const chunks = db.getSiblingDB("config").chunks.find().toArray();

const summary = {
  shards: shardingStatus.map(s => ({ id: s._id, host: s.host })),
  databases: databases.length,
  shardedCollections: collections.length,
  totalChunks: chunks.length,
  chunksByS hard: {}
};

// Compter les chunks par shard
chunks.forEach(function(chunk) {
  if (!summary.chunksByShard[chunk.shard]) {
    summary.chunksByShard[chunk.shard] = 0;
  }
  summary.chunksByShard[chunk.shard]++;
});

printjson(summary);

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

{
  shards: [
    { id: "shard01", host: "shard01/mongodb1:27017,mongodb2:27017" },
    { id: "shard02", host: "shard02/mongodb3:27017,mongodb4:27017" }
  ],
  databases: 3,
  shardedCollections: 5,
  totalChunks: 128,
  chunksByShard: {
    shard01: 64,
    shard02: 64
  }
}

// ============================================
// 💡 RAPPORT DÉTAILLÉ
// ============================================

function detailedShardingReport() {
  const configDb = db.getSiblingDB("config");

  print("\n" + "=".repeat(60));
  print("SHARDING STATUS REPORT");
  print("=".repeat(60));

  // Shards
  print("\n--- Shards ---");
  const shards = configDb.shards.find().toArray();
  shards.forEach(shard => {
    print(`${shard._id}: ${shard.host}`);
  });

  // Databases
  print("\n--- Databases ---");
  const databases = configDb.databases.find().toArray();
  databases.forEach(db => {
    print(`${db._id}: primary=${db.primary}, partitioned=${db.partitioned}`);
  });

  // Collections shardées
  print("\n--- Sharded Collections ---");
  const collections = configDb.collections.find().toArray();
  collections.forEach(coll => {
    print(`\n${coll._id}:`);
    print(`  Shard key: ${JSON.stringify(coll.key)}`);
    print(`  Unique: ${coll.unique || false}`);

    // Compter les chunks
    const chunkCount = configDb.chunks.countDocuments({ ns: coll._id });
    print(`  Chunks: ${chunkCount}`);

    // Distribution par shard
    const distribution = {};
    configDb.chunks.find({ ns: coll._id }).forEach(chunk => {
      distribution[chunk.shard] = (distribution[chunk.shard] || 0) + 1;
    });

    print(`  Distribution:`);
    Object.keys(distribution).forEach(shard => {
      const percentage = ((distribution[shard] / chunkCount) * 100).toFixed(2);
      print(`    ${shard}: ${distribution[shard]} chunks (${percentage}%)`);
    });
  });

  // Balancer status
  print("\n--- Balancer ---");
  const balancerStatus = sh.getBalancerState();
  print(`State: ${balancerStatus ? "✅ Enabled" : "❌ Disabled"}`);

  print("\n" + "=".repeat(60));
}

detailedShardingReport();
```

---

### 7.2 - Distribution des chunks

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 DISTRIBUTION DES CHUNKS
// ============================================

// 💡 Objectif
// Analyser la répartition des chunks entre les shards

// 🎯 Cas d'usage
// - Vérifier l'équilibrage
// - Identifier les déséquilibres
// - Planifier le rebalancing

// ============================================
// REQUÊTE
// ============================================

const configDb = db.getSiblingDB("config");
const collections = configDb.collections.find().toArray();

collections.forEach(function(coll) {
  print(`\n${"=".repeat(60)}`);
  print(`Collection: ${coll._id}`);
  print(`Shard Key: ${JSON.stringify(coll.key)}`);
  print("=".repeat(60));

  // Agrégation de distribution
  const distribution = configDb.chunks.aggregate([
    { $match: { ns: coll._id } },
    {
      $group: {
        _id: "$shard",
        count: { $sum: 1 },
        minKey: { $min: "$min" },
        maxKey: { $max: "$max" }
      }
    },
    { $sort: { count: -1 } }
  ]).toArray();

  const totalChunks = distribution.reduce((sum, d) => sum + d.count, 0);
  const avgChunksPerShard = totalChunks / distribution.length;

  print(`\nTotal chunks: ${totalChunks}`);
  print(`Average per shard: ${avgChunksPerShard.toFixed(2)}`);
  print("\nDistribution:");

  distribution.forEach(function(shard) {
    const percentage = ((shard.count / totalChunks) * 100).toFixed(2);
    const deviation = ((shard.count - avgChunksPerShard) / avgChunksPerShard * 100).toFixed(2);
    const status = Math.abs(deviation) < 10 ? "✅" :
                   Math.abs(deviation) < 20 ? "⚠️" : "❌";

    print(`\n  ${shard._id}:`);
    print(`    Chunks: ${shard.count} (${percentage}%)`);
    print(`    Deviation: ${deviation}% ${status}`);
  });

  // Alerte si déséquilibre
  const maxDeviation = Math.max(...distribution.map(s =>
    Math.abs((s.count - avgChunksPerShard) / avgChunksPerShard * 100)
  ));

  if (maxDeviation > 20) {
    print(`\n⚠️ WARNING: Significant imbalance detected (${maxDeviation.toFixed(2)}% max deviation)`);
    print("Consider running the balancer or manually splitting chunks");
  } else {
    print(`\n✅ Distribution is balanced (${maxDeviation.toFixed(2)}% max deviation)`);
  }
});

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
============================================================
Collection: myapp.users
Shard Key: {"userId":"hashed"}
============================================================

Total chunks: 128
Average per shard: 64.00

Distribution:

  shard01:
    Chunks: 65 (50.78%)
    Deviation: 1.56% ✅

  shard02:
    Chunks: 63 (49.22%)
    Deviation: -1.56% ✅

✅ Distribution is balanced (1.56% max deviation)
*/
```

---

## Maintenance et Nettoyage

### 8.1 - Identifier les collections à archiver

**🟡 Niveau : Intermédiaire** | **⚡ Performance : Rapide**

```javascript
// ============================================
// 📌 COLLECTIONS À ARCHIVER
// ============================================

// 💡 Objectif
// Identifier les collections anciennes ou volumineuses
// candidates à l'archivage

// 🎯 Cas d'usage
// - Gestion de la croissance
// - Optimisation de l'espace
// - Stratégie d'archivage

// ⚠️ Nécessite un champ de date

// ============================================
// REQUÊTE
// ============================================

// Seuils d'archivage
const OLD_DATA_DAYS = 365;  // 1 an
const LARGE_COLLECTION_GB = 10;  // 10 GB

const candidates = [];
const cutoffDate = new Date(Date.now() - OLD_DATA_DAYS * 86400000);

db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024 * 1024);

  // Vérifier si la collection a un champ de date
  const sample = db[col].findOne();
  let hasOldData = false;
  let oldDataCount = 0;

  if (sample && (sample.createdAt || sample.timestamp || sample.date)) {
    const dateField = sample.createdAt ? "createdAt" :
                     sample.timestamp ? "timestamp" : "date";

    oldDataCount = db[col].countDocuments({
      [dateField]: { $lt: cutoffDate }
    });

    hasOldData = oldDataCount > 0;
  }

  // Vérifier si collection volumineuse
  const isLarge = stats.size > LARGE_COLLECTION_GB;

  if (hasOldData || isLarge) {
    candidates.push({
      collection: col,
      totalDocs: stats.count,
      oldDocs: oldDataCount,
      sizeGB: stats.size.toFixed(2),
      reason: hasOldData && isLarge ? "Old data + Large size" :
              hasOldData ? "Old data" : "Large size"
    });
  }
});

// Trier par taille
candidates.sort((a, b) => b.sizeGB - a.sizeGB);

print("\n=== Archive Candidates ===\n");

if (candidates.length === 0) {
  print("No collections need archiving at this time");
} else {
  print("Collection".padEnd(30) + "Size".padEnd(12) + "Old Docs".padEnd(15) + "Reason");
  print("-".repeat(80));

  candidates.forEach(function(c) {
    print(
      c.collection.padEnd(30) +
      `${c.sizeGB} GB`.padEnd(12) +
      `${c.oldDocs.toLocaleString()}`.padEnd(15) +
      c.reason
    );
  });

  print("\n💡 Recommendations:");
  print("1. Review data retention policies");
  print("2. Consider implementing TTL indexes");
  print("3. Archive old data to cold storage");
  print("4. Set up automated archiving processes");
}

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== Archive Candidates ===

Collection                    Size        Old Docs       Reason
--------------------------------------------------------------------------------
logs                          15.50 GB    45000000       Old data + Large size
orders                        8.25 GB     1234567        Old data
events                        12.00 GB    0              Large size

💡 Recommendations:
1. Review data retention policies
2. Consider implementing TTL indexes
3. Archive old data to cold storage
4. Set up automated archiving processes
*/
```

---

### 8.2 - Espace disque récupérable

**🔴 Niveau : Avancé** | **⏱️ Performance : Modérée**

```javascript
// ============================================
// 📌 ESPACE RÉCUPÉRABLE
// ============================================

// 💡 Objectif
// Estimer l'espace disque qui peut être récupéré
// après suppressions et compact

// 🎯 Cas d'usage
// - Planification de maintenance
// - Optimisation du stockage
// - Gestion de capacité

// ============================================
// REQUÊTE
// ============================================

const reclaimable = [];

db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024);

  // Fragmentation = (StorageSize - Size) / StorageSize
  const fragmentation = ((stats.storageSize - stats.size) / stats.storageSize * 100).toFixed(2);
  const reclaimableMB = (stats.storageSize - stats.size).toFixed(2);

  if (fragmentation > 10) {  // > 10% fragmentation
    reclaimable.push({
      collection: col,
      sizeMB: stats.size.toFixed(2),
      storageMB: stats.storageSize.toFixed(2),
      reclaimableMB: reclaimableMB,
      fragmentation: fragmentation + "%",
      documentsCount: stats.count
    });
  }
});

// Trier par espace récupérable
reclaimable.sort((a, b) => b.reclaimableMB - a.reclaimableMB);

print("\n=== Reclaimable Space Report ===\n");

if (reclaimable.length === 0) {
  print("✅ No significant fragmentation detected");
} else {
  print("Collection".padEnd(25) + "Data".padEnd(12) + "Storage".padEnd(12) + "Reclaimable".padEnd(15) + "Fragmentation");
  print("-".repeat(85));

  let totalReclaimable = 0;

  reclaimable.forEach(function(c) {
    print(
      c.collection.padEnd(25) +
      `${c.sizeMB} MB`.padEnd(12) +
      `${c.storageMB} MB`.padEnd(12) +
      `${c.reclaimableMB} MB`.padEnd(15) +
      c.fragmentation
    );
    totalReclaimable += parseFloat(c.reclaimableMB);
  });

  print("-".repeat(85));
  print(`TOTAL RECLAIMABLE: ${totalReclaimable.toFixed(2)} MB (${(totalReclaimable / 1024).toFixed(2)} GB)`);

  print("\n⚠️ To reclaim space, run:");
  print("db.<collection>.compact()");
  print("\n💡 Note: compact() blocks writes. Schedule during maintenance window.");
}

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

/*
=== Reclaimable Space Report ===

Collection               Data        Storage     Reclaimable    Fragmentation
-------------------------------------------------------------------------------------
logs                     12000.00 MB 15000.00 MB 3000.00 MB     20.00%
old_data                 5000.00 MB  6500.00 MB  1500.00 MB     23.08%
temp_collection          1000.00 MB  1200.00 MB  200.00 MB      16.67%
-------------------------------------------------------------------------------------
TOTAL RECLAIMABLE: 4700.00 MB (4.59 GB)

⚠️ To reclaim space, run:
db.<collection>.compact()

💡 Note: compact() blocks writes. Schedule during maintenance window.
*/
```

---

**💡 Rappel** : Testez toujours ces requêtes en environnement de développement avant de les utiliser en production. Utilisez `explain()` pour vérifier les performances sur de grandes collections.

⏭️ [Requêtes de monitoring (currentOp, profiler)](/annexes/requetes-reference/02-requetes-monitoring.md)
