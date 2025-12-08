🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.3 Profiler de Requêtes MongoDB

## Introduction

Le **Database Profiler** de MongoDB est un outil d'analyse essentiel qui enregistre les opérations de base de données dans une collection système dédiée (`system.profile`). Il permet aux SRE et administrateurs d'identifier les requêtes lentes, d'analyser les patterns d'accès et d'optimiser les performances en production.

Contrairement aux logs MongoDB qui peuvent être verbeux et difficiles à parser, le profiler structure les données de performance dans un format interrogeable, facilitant l'analyse post-mortem et le monitoring continu.

---

## Architecture et Fonctionnement

### Collection system.profile

Le profiler stocke ses données dans une **capped collection** nommée `system.profile`, créée automatiquement à l'activation du profiling. Cette collection a les caractéristiques suivantes :

- **Taille par défaut** : 1 Mo (peut contenir environ 100-200 entrées selon leur complexité)
- **Type** : Capped collection (FIFO - First In, First Out)
- **Portée** : Une collection par base de données
- **Persistence** : Les données survivent au redémarrage de MongoDB

```javascript
// Vérifier la taille actuelle de la collection profiler
db.system.profile.stats()

// Résultat exemple
{
  "ns": "mydb.system.profile",
  "size": 1048576,  // 1 Mo
  "count": 156,
  "avgObjSize": 6721,
  "capped": true,
  "max": -1
}
```

### Redimensionnement de la Collection

Pour des environnements de production avec un volume élevé d'opérations, augmenter la taille de la collection profiler est souvent nécessaire :

```javascript
// 1. Désactiver le profiler
db.setProfilingLevel(0)

// 2. Supprimer l'ancienne collection
db.system.profile.drop()

// 3. Créer une nouvelle collection avec la taille souhaitée (128 Mo)
db.createCollection("system.profile", {
  capped: true,
  size: 134217728  // 128 Mo en octets
})

// 4. Réactiver le profiler
db.setProfilingLevel(1, { slowms: 100 })
```

**Recommandations de dimensionnement** :
- **Environnement de développement** : 1-10 Mo
- **Production à faible charge** : 50-100 Mo
- **Production à charge élevée** : 200-500 Mo
- **Analyse approfondie temporaire** : 1-2 Go

---

## Niveaux de Profiling

MongoDB offre trois niveaux de profiling, chacun avec son compromis entre visibilité et impact sur les performances.

### Niveau 0 : Désactivé (Défaut)

```javascript
db.setProfilingLevel(0)
```

**Caractéristiques** :
- Aucune opération n'est enregistrée dans system.profile
- Impact zéro sur les performances
- Les opérations lentes continuent d'apparaître dans les logs MongoDB

**Cas d'usage** : Mode par défaut en production pour minimiser l'overhead.

### Niveau 1 : Opérations Lentes Uniquement

```javascript
// Profiler les opérations dépassant 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Profiler les opérations dépassant 50ms
db.setProfilingLevel(1, { slowms: 50 })
```

**Caractéristiques** :
- Enregistre uniquement les opérations dépassant le seuil `slowms`
- Impact minimal sur les performances (< 5%)
- Permet de capturer les requêtes problématiques sans noyer les données

**Cas d'usage** :
- Monitoring continu en production
- Identification des requêtes nécessitant optimisation
- Diagnostic initial de problèmes de performance

**Recommandations de seuil** :
- **OLTP/Web applications** : 50-100ms
- **Analytics/Reporting** : 500-1000ms
- **Batch processing** : 2000-5000ms

### Niveau 2 : Toutes les Opérations

```javascript
db.setProfilingLevel(2)

// Avec filtres optionnels
db.setProfilingLevel(2, {
  slowms: 0,  // Toutes les opérations
  sampleRate: 0.1  // Échantillonner 10% des opérations
})
```

**Caractéristiques** :
- Enregistre **toutes** les opérations de lecture et d'écriture
- Impact significatif sur les performances (10-30%)
- Volume de données très élevé
- La collection system.profile se remplit rapidement

**Cas d'usage** :
- Diagnostic approfondi de problèmes spécifiques (sessions temporaires)
- Audit de sécurité détaillé
- Analyse de patterns d'accès complets
- **Jamais en production continue sans échantillonnage**

---

## Configuration et Gestion

### Vérification de l'État Actuel

```javascript
// Obtenir le niveau et la configuration actuelle
db.getProfilingStatus()

// Résultat
{
  "was": 1,
  "slowms": 100,
  "sampleRate": 1.0,
  "ok": 1
}
```

### Échantillonnage (MongoDB 4.4+)

L'option `sampleRate` permet de réduire le volume de données collectées en mode niveau 2 :

```javascript
// Profiler 25% de toutes les opérations
db.setProfilingLevel(2, { sampleRate: 0.25 })

// Profiler 5% des opérations lentes (> 200ms)
db.setProfilingLevel(1, {
  slowms: 200,
  sampleRate: 0.05
})
```

**Calcul de l'échantillonnage** :
- `sampleRate: 1.0` → 100% des opérations (défaut)
- `sampleRate: 0.5` → 50% des opérations
- `sampleRate: 0.01` → 1% des opérations

### Configuration Multi-Bases

Le profiler est configuré **par base de données**. Pour activer sur plusieurs bases :

```javascript
// Script pour activer le profiling sur toutes les bases
db.adminCommand("listDatabases").databases.forEach(function(d) {
  if (d.name !== "admin" && d.name !== "local" && d.name !== "config") {
    db.getSiblingDB(d.name).setProfilingLevel(1, { slowms: 100 });
  }
});
```

### Configuration via Fichier de Configuration

Pour persister la configuration au redémarrage (MongoDB 4.4+) :

```yaml
# mongod.conf
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 1.0
```

---

## Structure des Entrées du Profiler

### Document Type Complet

Chaque opération profilée génère un document avec la structure suivante :

```javascript
{
  "op" : "query",              // Type d'opération
  "ns" : "mydb.users",         // Namespace (base.collection)
  "command" : {                // Commande complète exécutée
    "find" : "users",
    "filter" : { "age" : { "$gte" : 30 } },
    "sort" : { "lastName" : 1 },
    "projection" : { "email" : 1, "lastName" : 1 },
    "limit" : 100
  },
  "keysExamined" : 15234,      // Clés d'index examinées
  "docsExamined" : 15234,      // Documents examinés
  "nreturned" : 100,           // Documents retournés
  "responseLength" : 12456,    // Taille de la réponse en octets
  "millis" : 247,              // Durée totale en millisecondes
  "planSummary" : "IXSCAN { age: 1, lastName: 1 }",  // Plan d'exécution
  "execStats" : {              // Statistiques détaillées d'exécution
    "stage" : "FETCH",
    "nReturned" : 100,
    "executionTimeMillisEstimate" : 245,
    "works" : 15335,
    "advanced" : 100,
    "needTime" : 15234,
    "inputStage" : {
      "stage" : "IXSCAN",
      "keyPattern" : { "age" : 1, "lastName" : 1 },
      "indexName" : "age_1_lastName_1",
      "direction" : "forward"
    }
  },
  "ts" : ISODate("2024-12-08T14:23:45.123Z"),  // Timestamp
  "client" : "10.0.1.45",      // Adresse IP du client
  "appName" : "ProductionAPI", // Nom de l'application
  "user" : "apiUser@mydb",     // Utilisateur MongoDB
  "locks" : {                  // Locks acquis
    "Global" : { "r" : 252 },
    "Database" : { "r" : 126 },
    "Collection" : { "r" : 126 }
  },
  "flowControl" : {            // Contrôle de flux
    "acquireCount" : 1,
    "timeAcquiringMicros" : 5
  }
}
```

### Champs Clés pour l'Analyse

| Champ | Description | Utilité SRE |
|-------|-------------|-------------|
| `millis` | Durée totale d'exécution | Identifier les requêtes lentes |
| `keysExamined` / `docsExamined` | Nombre d'éléments scannés | Détecter les scans inefficaces |
| `nreturned` | Documents retournés | Calculer le ratio d'efficacité |
| `planSummary` | Résumé du plan d'exécution | Vérifier l'utilisation des index |
| `responseLength` | Taille de la réponse | Identifier les réponses volumineuses |
| `ns` | Namespace | Analyser par collection |
| `client` | IP du client | Traçabilité et analyse par source |
| `locks` | Informations de verrouillage | Diagnostiquer les contentions |

---

## Analyse des Données du Profiler

### Requêtes d'Analyse Essentielles

#### 1. Top 10 des Requêtes les Plus Lentes

```javascript
db.system.profile.find()
  .sort({ millis: -1 })
  .limit(10)
  .pretty()
```

#### 2. Requêtes avec Ratio d'Efficacité Faible

Le ratio d'efficacité compare les documents examinés aux documents retournés. Un ratio élevé indique un scan inefficace.

```javascript
db.system.profile.aggregate([
  {
    $match: {
      op: { $in: ["query", "find"] },
      docsExamined: { $gt: 1000 }
    }
  },
  {
    $project: {
      ns: 1,
      millis: 1,
      docsExamined: 1,
      nreturned: 1,
      ratio: {
        $cond: {
          if: { $eq: ["$nreturned", 0] },
          then: "$docsExamined",
          else: { $divide: ["$docsExamined", "$nreturned"] }
        }
      }
    }
  },
  {
    $match: { ratio: { $gt: 100 } }  // Ratio > 100:1 est problématique
  },
  { $sort: { ratio: -1 } },
  { $limit: 20 }
])
```

**Interprétation** :
- **Ratio < 10** : Efficacité excellente
- **Ratio 10-100** : Acceptable, mais peut être optimisé
- **Ratio > 100** : Problème majeur, nécessite investigation
- **Ratio > 1000** : Scan de table complet probable

#### 3. Analyse par Collection

```javascript
db.system.profile.aggregate([
  {
    $group: {
      _id: "$ns",
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      maxMillis: { $max: "$millis" },
      totalDocsExamined: { $sum: "$docsExamined" }
    }
  },
  { $sort: { avgMillis: -1 } }
])
```

#### 4. Distribution Temporelle des Requêtes Lentes

```javascript
db.system.profile.aggregate([
  {
    $match: { millis: { $gt: 100 } }
  },
  {
    $group: {
      _id: {
        year: { $year: "$ts" },
        month: { $month: "$ts" },
        day: { $dayOfMonth: "$ts" },
        hour: { $hour: "$ts" }
      },
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1, "_id.day": 1, "_id.hour": 1 } }
])
```

#### 5. Requêtes Sans Index (Collection Scans)

```javascript
db.system.profile.find({
  "planSummary": { $regex: /^COLLSCAN/ }
}).sort({ millis: -1 }).limit(20)
```

#### 6. Analyse par Type d'Opération

```javascript
db.system.profile.aggregate([
  {
    $group: {
      _id: "$op",
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      maxMillis: { $max: "$millis" }
    }
  },
  { $sort: { count: -1 } }
])
```

#### 7. Requêtes par Application/Client

```javascript
db.system.profile.aggregate([
  {
    $group: {
      _id: {
        appName: "$appName",
        client: "$client"
      },
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      slowQueries: {
        $sum: { $cond: [{ $gt: ["$millis", 1000] }, 1, 0] }
      }
    }
  },
  { $sort: { avgMillis: -1 } }
])
```

---

## Métriques Clés et Seuils d'Alerte

### Tableau des Métriques Critiques

| Métrique | Seuil Normal | Seuil Attention | Seuil Critique | Action |
|----------|--------------|-----------------|----------------|--------|
| **millis** | < 50ms | 50-200ms | > 200ms | Optimiser la requête |
| **docsExamined/nreturned** | < 10 | 10-100 | > 100 | Ajouter/réviser index |
| **keysExamined/nreturned** | < 5 | 5-50 | > 50 | Réviser index composés |
| **responseLength** | < 1 MB | 1-10 MB | > 10 MB | Pagination/projection |
| **COLLSCAN** | 0% | < 5% | > 5% | Index manquants |
| **locks.timeAcquiringMicros** | < 1000 | 1000-10000 | > 10000 | Contention de locks |

### Exemples d'Alertes Prometheus/Grafana

```yaml
# Règle d'alerte pour requêtes lentes fréquentes
groups:
  - name: mongodb_profiler_alerts
    interval: 30s
    rules:
      - alert: HighSlowQueryRate
        expr: |
          rate(mongodb_profile_slow_queries_total[5m]) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Taux élevé de requêtes lentes sur {{ $labels.database }}"

      - alert: CriticalCollectionScan
        expr: |
          mongodb_profile_collscan_percentage > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Scans de collection excessifs sur {{ $labels.namespace }}"
```

---

## Patterns d'Optimisation Basés sur le Profiler

### Pattern 1 : Index Manquant

**Symptôme détecté** :
```javascript
{
  "planSummary": "COLLSCAN",
  "docsExamined": 45678,
  "nreturned": 10,
  "millis": 1245
}
```

**Diagnostic** : Scan de collection complet pour retourner seulement 10 documents.

**Solution** :
```javascript
// Analyser la requête
db.collection.find({ status: "active", country: "FR" })

// Créer l'index approprié
db.collection.createIndex({ status: 1, country: 1 })
```

### Pattern 2 : Index Non-Sélectif

**Symptôme détecté** :
```javascript
{
  "planSummary": "IXSCAN { status: 1 }",
  "keysExamined": 89234,
  "docsExamined": 89234,
  "nreturned": 50,
  "millis": 876
}
```

**Diagnostic** : L'index `status` est utilisé mais non sélectif (presque tous les documents sont scannés).

**Solution** :
```javascript
// Index composé plus sélectif
db.collection.createIndex({ country: 1, status: 1, createdAt: -1 })
```

### Pattern 3 : Projection Manquante

**Symptôme détecté** :
```javascript
{
  "responseLength": 15728640,  // ~15 MB
  "nreturned": 1000,
  "millis": 234
}
```

**Diagnostic** : Documents complets retournés alors que seulement quelques champs sont nécessaires.

**Solution** :
```javascript
// Ajouter une projection
db.collection.find(
  { status: "active" },
  { _id: 1, name: 1, email: 1 }  // Seulement les champs nécessaires
).limit(1000)
```

### Pattern 4 : Pas de Limite

**Symptôme détecté** :
```javascript
{
  "nreturned": 125000,
  "responseLength": 67108864,  // 64 MB
  "millis": 3456
}
```

**Diagnostic** : Requête retournant un nombre excessif de documents.

**Solution** :
```javascript
// Implémenter la pagination
db.collection.find({ status: "active" })
  .sort({ createdAt: -1 })
  .limit(100)
  .skip(pageNumber * 100)
```

---

## Impact sur les Performances

### Overhead du Profiler

| Niveau | Overhead CPU | Overhead I/O | Overhead Mémoire |
|--------|--------------|--------------|------------------|
| Niveau 0 | 0% | 0% | 0% |
| Niveau 1 (slowms: 100ms) | 1-3% | 2-5% | Minimal |
| Niveau 1 (slowms: 10ms) | 3-7% | 5-10% | Faible |
| Niveau 2 (sample: 0.1) | 5-10% | 10-15% | Modéré |
| Niveau 2 (sample: 1.0) | 15-30% | 20-40% | Élevé |

### Recommandations de Production

**Environnements de Production Critiques** :
```javascript
// Configuration conservative
db.setProfilingLevel(1, {
  slowms: 200,      // Seuil élevé
  sampleRate: 0.5   // Échantillonner 50%
})

// Collection profiler dimensionnée
db.system.profile.drop()
db.createCollection("system.profile", {
  capped: true,
  size: 104857600  // 100 Mo
})
```

**Environnements de Développement/Staging** :
```javascript
// Configuration plus agressive pour capturer plus de données
db.setProfilingLevel(1, {
  slowms: 50,
  sampleRate: 1.0
})
```

**Diagnostic Temporaire** :
```javascript
// Activer niveau 2 temporairement (15-30 minutes max)
db.setProfilingLevel(2, { sampleRate: 0.1 })

// Désactiver après investigation
db.setProfilingLevel(0)
```

---

## Intégration avec les Outils de Monitoring

### Export vers Outils Externes

#### Script d'Export vers JSON

```javascript
// export_profiler.js
const fs = require('fs');
const { MongoClient } = require('mongodb');

async function exportProfiler() {
  const client = await MongoClient.connect('mongodb://localhost:27017');
  const db = client.db('mydb');

  const profiles = await db.collection('system.profile')
    .find({ millis: { $gt: 100 } })
    .sort({ ts: -1 })
    .limit(10000)
    .toArray();

  fs.writeFileSync('profiler_export.json', JSON.stringify(profiles, null, 2));
  await client.close();
}

exportProfiler();
```

#### Intégration Elasticsearch

```javascript
// Logstash pipeline pour indexer les données du profiler
input {
  mongodb {
    uri => "mongodb://localhost:27017/mydb"
    placeholder_db_dir => "/opt/logstash/data/"
    collection => "system.profile"
    batch_size => 5000
  }
}

filter {
  mutate {
    rename => { "millis" => "duration_ms" }
  }

  if [docsExamined] > 0 and [nreturned] > 0 {
    ruby {
      code => "event.set('efficiency_ratio', event.get('docsExamined').to_f / event.get('nreturned'))"
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "mongodb-profiler-%{+YYYY.MM.dd}"
  }
}
```

### Dashboard Grafana

Exemple de requêtes PromQL pour un exporter MongoDB :

```promql
# Taux de requêtes lentes par seconde
rate(mongodb_profile_slow_queries_total[5m])

# Durée moyenne des requêtes lentes
avg(mongodb_profile_query_duration_ms{slow="true"})

# Pourcentage de collection scans
(mongodb_profile_collscan_total / mongodb_profile_queries_total) * 100

# Top 5 collections avec le plus de requêtes lentes
topk(5, sum by (namespace) (rate(mongodb_profile_slow_queries_total[10m])))
```

---

## Bonnes Pratiques pour SRE

### 1. Stratégie de Profiling Progressive

```javascript
// Phase 1 : Baseline (1 semaine)
db.setProfilingLevel(1, { slowms: 500, sampleRate: 1.0 })

// Phase 2 : Réduction du seuil (2 semaines)
db.setProfilingLevel(1, { slowms: 200, sampleRate: 1.0 })

// Phase 3 : Monitoring continu
db.setProfilingLevel(1, { slowms: 100, sampleRate: 0.5 })
```

### 2. Rotation et Archivage

```javascript
// Script de rotation quotidien
use mydb

// Exporter les anciennes données
var yesterday = new Date();
yesterday.setDate(yesterday.getDate() - 1);

db.system.profile.find({ ts: { $lt: yesterday } }).forEach(function(doc) {
  db.profiler_archive.insert(doc);
});

// Nettoyer
db.system.profile.remove({ ts: { $lt: yesterday } });
```

### 3. Alerting Intelligent

**Règles d'alerte graduées** :

```javascript
// Niveau 1 : Avertissement
// > 5 requêtes/sec dépassant 200ms pendant 5 minutes

// Niveau 2 : Attention
// > 10 requêtes/sec dépassant 500ms pendant 2 minutes

// Niveau 3 : Critique
// > 20 requêtes/sec dépassant 1000ms pendant 1 minute
// OU > 50% des requêtes sont des COLLSCAN
```

### 4. Checklist de Review Hebdomadaire

1. **Analyser les nouvelles requêtes lentes** apparues cette semaine
2. **Vérifier les tendances** de durée moyenne par collection
3. **Identifier les patterns d'accès** changeants
4. **Corréler avec les déploiements** et incidents
5. **Documenter les optimisations** effectuées
6. **Planifier les maintenances d'index** si nécessaire

### 5. Documentation des Optimisations

```javascript
// Template de documentation
{
  "date": "2024-12-08",
  "collection": "users",
  "issue": "COLLSCAN sur recherche par email",
  "profilerData": {
    "avgMillis": 1234,
    "docsExamined": 450000,
    "nreturned": 1
  },
  "solution": "Ajout d'index unique sur email",
  "command": "db.users.createIndex({ email: 1 }, { unique: true })",
  "result": {
    "avgMillis": 2,
    "docsExamined": 1,
    "improvement": "99.8%"
  }
}
```

---

## Limitations et Considérations

### Limitations Techniques

1. **Pas d'historique long terme** : La collection capped a une taille limitée
2. **Overhead non négligeable** en niveau 2
3. **Pas de profiling des opérations système** (compact, repairDatabase, etc.)
4. **Granularité limitée** : pas de profiling au niveau des étapes de pipeline
5. **Format de stockage** : BSON uniquement, pas de streaming direct

### Opérations Non Profilées

- Commandes d'administration (listDatabases, serverStatus)
- Opérations de réplication interne
- Opérations de balancing (sharding)
- Heartbeats de Replica Set
- Lectures du cache WiredTiger

### Alternatives et Compléments

| Outil | Usage | Avantages | Inconvénients |
|-------|-------|-----------|---------------|
| **Profiler** | Analyse détaillée post-mortem | Données structurées, queryable | Overhead, pas de temps réel |
| **Logs MongoDB** | Monitoring continu | Overhead minimal | Format texte, parsing nécessaire |
| **mongostat** | Monitoring en temps réel | Léger, overview global | Pas de détails par requête |
| **explain()** | Analyse ciblée | Zéro overhead | Requête par requête uniquement |
| **PMM** | Monitoring complet | Dashboards, alerting | Infrastructure supplémentaire |

---

## Cas d'Usage Avancés

### Scénario 1 : Investigation Post-Incident

**Contexte** : Pic de latence détecté entre 14h00 et 14h15.

```javascript
// 1. Analyser les requêtes pendant la période
db.system.profile.aggregate([
  {
    $match: {
      ts: {
        $gte: ISODate("2024-12-08T14:00:00Z"),
        $lte: ISODate("2024-12-08T14:15:00Z")
      },
      millis: { $gt: 500 }
    }
  },
  {
    $group: {
      _id: {
        ns: "$ns",
        op: "$op",
        planSummary: "$planSummary"
      },
      count: { $sum: 1 },
      avgMillis: { $avg: "$millis" },
      maxMillis: { $max: "$millis" },
      exampleCommand: { $first: "$command" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 10 }
])

// 2. Identifier la requête problématique
// 3. Vérifier si un index est manquant ou non optimal
// 4. Documenter et créer un ticket
```

### Scénario 2 : Audit de Performance Mensuel

```javascript
// Script d'audit automatisé
function monthlyPerformanceAudit() {
  const startDate = new Date();
  startDate.setMonth(startDate.getMonth() - 1);

  print("=== Audit de Performance MongoDB ===\n");

  // 1. Top 10 collections les plus sollicitées
  print("1. Collections les plus actives:");
  db.system.profile.aggregate([
    { $match: { ts: { $gte: startDate } } },
    { $group: { _id: "$ns", count: { $sum: 1 } } },
    { $sort: { count: -1 } },
    { $limit: 10 }
  ]).forEach(printjson);

  // 2. Requêtes les plus lentes
  print("\n2. Top 10 requêtes les plus lentes:");
  db.system.profile.find({ ts: { $gte: startDate } })
    .sort({ millis: -1 })
    .limit(10)
    .forEach(function(doc) {
      print(`${doc.ns} - ${doc.millis}ms - ${doc.planSummary}`);
    });

  // 3. Taux de collection scans
  const collscans = db.system.profile.count({
    ts: { $gte: startDate },
    planSummary: /^COLLSCAN/
  });
  const total = db.system.profile.count({ ts: { $gte: startDate } });
  print(`\n3. Taux de COLLSCAN: ${(collscans/total*100).toFixed(2)}%`);

  // 4. Distributions de latence
  print("\n4. Distribution de latence:");
  db.system.profile.aggregate([
    { $match: { ts: { $gte: startDate } } },
    {
      $bucket: {
        groupBy: "$millis",
        boundaries: [0, 10, 50, 100, 500, 1000, 5000, 10000],
        default: "10000+",
        output: { count: { $sum: 1 } }
      }
    }
  ]).forEach(printjson);
}

monthlyPerformanceAudit();
```

---

## Résumé pour SRE

### Commandes Essentielles à Mémoriser

```javascript
// Activation/Désactivation rapide
db.setProfilingLevel(1, { slowms: 100 })  // Mode production
db.setProfilingLevel(0)                   // Désactivation

// Vérification état
db.getProfilingStatus()

// Analyse rapide
db.system.profile.find().sort({ millis: -1 }).limit(10)

// Nettoyage
db.setProfilingLevel(0)
db.system.profile.drop()
```

### Points Clés de Décision

| Situation | Configuration Recommandée |
|-----------|---------------------------|
| **Production stable** | Niveau 1, slowms: 200ms, sample: 0.5 |
| **Nouveau déploiement** | Niveau 1, slowms: 100ms, sample: 1.0 (2 semaines) |
| **Investigation active** | Niveau 2, sample: 0.1 (< 1 heure) |
| **Audit de sécurité** | Niveau 2, sample: 1.0 (session dédiée) |
| **Environnement dev** | Niveau 1, slowms: 50ms, sample: 1.0 |

---

## Conclusion

Le **Database Profiler** est un outil indispensable dans l'arsenal de tout SRE ou administrateur MongoDB. Sa capacité à capturer et structurer les informations de performance permet :

1. **Diagnostic précis** des problèmes de performance
2. **Identification proactive** des requêtes nécessitant optimisation
3. **Validation post-déploiement** des changements
4. **Documentation** des patterns d'accès

L'utilisation judicieuse du profiler, combinée avec une analyse régulière et une intégration dans les pipelines de monitoring, garantit la santé et les performances optimales des bases de données MongoDB en production.

**Prochaines étapes recommandées** :
- Configurer le profiler avec des seuils adaptés à votre SLA
- Automatiser l'analyse hebdomadaire des données
- Intégrer les métriques dans votre stack de monitoring
- Établir des runbooks pour les patterns d'optimisation courants

---

**Références** :
- [MongoDB Database Profiler Documentation](https://www.mongodb.com/docs/manual/tutorial/manage-the-database-profiler/)
- [Profiling MongoDB Operations](https://www.mongodb.com/docs/manual/reference/database-profiler/)
- [MongoDB Performance Best Practices](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)

⏭️ [Logs MongoDB](/13-monitoring-administration/04-logs-mongodb.md)
