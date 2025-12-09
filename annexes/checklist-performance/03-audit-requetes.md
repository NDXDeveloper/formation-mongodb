🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.3 - Audit de Requêtes

## Introduction

L'audit de requêtes identifie et optimise les **patterns de requêtes inefficaces** qui dégradent les performances. Une requête mal optimisée peut impacter toute l'application, même avec une bonne modélisation et indexation.

### 🎯 Objectif

Détecter les requêtes lentes, comprendre leur comportement, et les optimiser pour réduire les temps de réponse et la charge serveur.

### ⏱️ Durée estimée
- Audit rapide : 30-45 minutes
- Audit complet : 2-4 heures

---

## Métriques Clés

### 📊 Seuils de Performance

| Type de requête | Acceptable | Warning | Critique |
|----------------|------------|---------|----------|
| **Find simple** | < 10ms | 10-50ms | > 50ms |
| **Find complexe** | < 50ms | 50-200ms | > 200ms |
| **Agrégation simple** | < 50ms | 50-300ms | > 300ms |
| **Agrégation complexe** | < 300ms | 300ms-1s | > 1s |
| **Count** | < 10ms | 10-100ms | > 100ms |

### 🎯 Ratios Optimaux

```markdown
✅ nReturned / totalDocsExamined > 0.8 (80%+)
✅ executionTimeMillis < 50ms (requêtes simples)
✅ keysExamined / nReturned ≈ 1 (index couvrant)
✅ totalKeysExamined / totalDocsExamined ≈ 1 (bonne sélectivité)
```

---

## Identification des Requêtes Lentes

### 🔍 Activation du Profiler

#### ✅ Niveaux de Profilage

| Niveau | Comportement | Usage |
|--------|--------------|-------|
| **0** | Désactivé | Production normale |
| **1** | Requêtes lentes uniquement | Production (recommandé) |
| **2** | Toutes les requêtes | Debug/analyse ponctuelle |

**Commandes** :
```javascript
// Vérifier le niveau actuel
db.getProfilingStatus()

// Activer niveau 1 (requêtes > 100ms)
db.setProfilingLevel(1, { slowms: 100 })

// Activer niveau 2 (TOUTES les requêtes - utiliser avec précaution)
db.setProfilingLevel(2)

// Niveau 1 avec filtre personnalisé
db.setProfilingLevel(1, {
  slowms: 100,
  sampleRate: 0.1  // 10% des requêtes seulement
})

// Désactiver
db.setProfilingLevel(0)
```

**⚠️ Attention** :
```markdown
- Niveau 2 génère BEAUCOUP de données
- Utiliser seulement en dev/staging ou ponctuellement
- Toujours désactiver après analyse
- system.profile est une capped collection (taille limitée)
```

---

### 🔍 Analyse du Profiler

#### Requêtes les plus lentes

```javascript
// Top 10 requêtes les plus lentes
db.system.profile.find()
  .sort({ millis: -1 })
  .limit(10)
  .pretty()

// Requêtes > 1 seconde
db.system.profile.find({
  millis: { $gt: 1000 }
}).sort({ ts: -1 })

// Grouper par pattern de requête
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  { $group: {
      _id: {
        ns: "$ns",
        op: "$op",
        planSummary: "$planSummary"
      },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" },
      maxMs: { $max: "$millis" },
      totalMs: { $sum: "$millis" }
    }
  },
  { $sort: { totalMs: -1 } },
  { $limit: 20 }
])
```

#### Requêtes avec COLLSCAN

```javascript
// Toutes les requêtes avec scan complet
db.system.profile.find({
  planSummary: "COLLSCAN"
}).sort({ millis: -1 })

// Par collection
db.system.profile.aggregate([
  { $match: { planSummary: "COLLSCAN" } },
  { $group: {
      _id: "$ns",
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" }
    }
  },
  { $sort: { count: -1 } }
])

// COLLSCAN sur grandes collections (> 100k docs)
db.system.profile.find({
  planSummary: "COLLSCAN",
  docsExamined: { $gt: 100000 }
})
```

#### Requêtes fréquentes

```javascript
// Requêtes les plus fréquentes
db.system.profile.aggregate([
  { $group: {
      _id: {
        ns: "$ns",
        command: "$command.filter"
      },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

---

### 🔍 Logs MongoDB

Pour analyse sans profiler actif :

```bash
# Grep requêtes lentes dans les logs
grep "Slow query" /var/log/mongodb/mongod.log

# Requêtes > 1000ms
grep -E "command.*([0-9]{4,}ms)" /var/log/mongodb/mongod.log

# Parser les logs avec mongoldap ou mtools
mloginfo /var/log/mongodb/mongod.log --queries
```

**Configuration du logging** :
```javascript
// Changer le seuil de log
db.setProfilingLevel(0)  // Désactiver profiler
db.adminCommand({
  setParameter: 1,
  slowms: 100  // Log requêtes > 100ms
})
```

---

## Analyse avec explain()

### 📊 Comprendre explain()

#### Modes d'explain()

```javascript
// queryPlanner : plan choisi uniquement
db.collection.find({...}).explain("queryPlanner")

// executionStats : statistiques d'exécution
db.collection.find({...}).explain("executionStats")

// allPlansExecution : tous les plans considérés
db.collection.find({...}).explain("allPlansExecution")
```

#### Points clés à analyser

```javascript
// Exemple de requête
db.orders.find({
  status: "pending",
  total: { $gte: 100 }
}).sort({ createdAt: -1 }).explain("executionStats")

// Vérifier ces champs dans le résultat :
{
  executionStats: {
    executionTimeMillis: 15,      // ✅ < 50ms
    totalKeysExamined: 120,       // Clés d'index examinées
    totalDocsExamined: 120,       // Documents examinés
    nReturned: 100,               // Documents retournés
    // Ratio optimal : nReturned / totalDocsExamined > 0.8

    executionStages: {
      stage: "SORT",              // ⚠️ Tri en mémoire
      inputStage: {
        stage: "IXSCAN",          // ✅ Utilise un index
        indexName: "status_1_total_1"
      }
    }
  }
}
```

### ✅ Indicateurs de Bonne Performance

```markdown
✅ stage: "IXSCAN" (utilise un index)
✅ stage: "IDHACK" (recherche par _id, optimal)
✅ executionTimeMillis < 50ms
✅ totalDocsExamined ≈ nReturned (ratio > 0.8)
✅ Pas de stage "SORT" (tri fait par index)
✅ Pas de stage "FETCH" si query couverte
✅ keysExamined ≈ nReturned (index sélectif)
```

### ⚠️ Indicateurs de Problème

```markdown
⚠️ stage: "COLLSCAN" (scan complet)
⚠️ totalDocsExamined >> nReturned (ratio < 0.5)
⚠️ executionTimeMillis > 100ms
⚠️ stage: "SORT" avec memUsage élevé
⚠️ rejectedPlans non vide (compétition d'index)
⚠️ keysExamined >> totalDocsExamined (index peu sélectif)
```

---

### 📋 Checklist d'Analyse

```javascript
// Template d'analyse
function analyzeQuery(collection, query, options = {}) {
  print("=== Analyse de requête ===\n");

  const explain = db.getCollection(collection)
    .find(query, options.projection || {})
    .sort(options.sort || {})
    .limit(options.limit || 0)
    .explain("executionStats");

  const stats = explain.executionStats;

  print("Collection: " + collection);
  print("Temps d'exécution: " + stats.executionTimeMillis + "ms");
  print("Documents examinés: " + stats.totalDocsExamined);
  print("Documents retournés: " + stats.nReturned);
  print("Clés examinées: " + stats.totalKeysExamined);

  // Ratio d'efficacité
  const ratio = stats.totalDocsExamined > 0
    ? (stats.nReturned / stats.totalDocsExamined).toFixed(2)
    : 0;
  print("Ratio efficacité: " + ratio);

  // Analyse du stage
  const stage = stats.executionStages.stage;
  print("Stage principal: " + stage);

  // Alertes
  if (stage === "COLLSCAN") {
    print("❌ COLLSCAN détecté - Index manquant");
  }
  if (stats.executionTimeMillis > 100) {
    print("⚠️  Requête lente (> 100ms)");
  }
  if (ratio < 0.5) {
    print("⚠️  Faible ratio efficacité (< 50%)");
  }
  if (stats.executionStages.stage === "SORT") {
    print("⚠️  Tri en mémoire - Index de tri manquant");
  }

  return explain;
}

// Utilisation
analyzeQuery("orders",
  { status: "pending" },
  { sort: { createdAt: -1 }, limit: 100 }
);
```

---

## Anti-Patterns de Requêtes

### ❌ Anti-Pattern 1 : Requêtes N+1

**Problème** : Faire N requêtes au lieu d'une seule

```javascript
// ❌ Mauvais : N+1 queries
const orders = db.orders.find({ userId: 123 }).toArray();
orders.forEach(order => {
  const user = db.users.findOne({ _id: order.userId });  // N requêtes !
  print(user.name);
});

// ✅ Bon : Jointure avec $lookup
db.orders.aggregate([
  { $match: { userId: 123 } },
  { $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" }
])

// ✅ Ou : Extended Reference (duplication)
// Dupliquer name directement dans orders
{
  _id: 1,
  userId: 123,
  userName: "John Doe",  // Dupliqué pour performance
  items: [...]
}
```

---

### ❌ Anti-Pattern 2 : Projection Absente

**Problème** : Récupérer tous les champs alors qu'on n'en a besoin que de quelques-uns

```javascript
// ❌ Mauvais : récupère tout le document (potentiellement lourd)
db.users.find({ status: "active" })

// ✅ Bon : projection explicite
db.users.find(
  { status: "active" },
  { _id: 1, name: 1, email: 1 }  // Seulement les champs nécessaires
)

// Impact : réduction de 90% du volume de données transférées
```

---

### ❌ Anti-Pattern 3 : $regex Non Ancré

**Problème** : Regex sans ancrage au début désactive les index

```javascript
// ❌ Mauvais : regex non ancré, n'utilise pas l'index
db.products.find({ name: /smartphone/ })

// ⚠️ Acceptable : regex ancré au début
db.products.find({ name: /^smartphone/ })  // Peut utiliser l'index

// ✅ Meilleur : index texte pour recherche full-text
db.products.createIndex({ name: "text" })
db.products.find({ $text: { $search: "smartphone" } })

// ✅ Optimal : Atlas Search
```

---

### ❌ Anti-Pattern 4 : $where et $expr Inefficaces

**Problème** : Exécution JavaScript côté serveur, très lent

```javascript
// ❌ Très mauvais : $where avec JavaScript
db.users.find({
  $where: "this.age > 18 && this.status === 'active'"
})

// ✅ Bon : opérateurs natifs
db.users.find({
  age: { $gt: 18 },
  status: "active"
})

// ⚠️ $expr acceptable pour comparaisons inter-champs
db.sales.find({
  $expr: { $gt: ["$spent", "$budget"] }
})
```

---

### ❌ Anti-Pattern 5 : Tri Sans Index

**Problème** : Tri en mémoire avec limite de 32 Mo

```javascript
// ❌ Mauvais : tri en mémoire (stage: SORT)
db.posts.find({ published: true }).sort({ createdAt: -1 })
// Sans index sur { published: 1, createdAt: -1 }

// ✅ Bon : créer l'index approprié
db.posts.createIndex({ published: 1, createdAt: -1 })

// Vérifier avec explain() qu'il n'y a plus de stage SORT
```

---

### ❌ Anti-Pattern 6 : $in Avec Trop de Valeurs

**Problème** : $in avec des milliers de valeurs ralentit la requête

```javascript
// ❌ Mauvais : $in avec 10000 valeurs
db.products.find({
  _id: { $in: [/* 10000 IDs */] }
})

// ✅ Meilleur : découper en lots
const batchSize = 1000;
for (let i = 0; i < ids.length; i += batchSize) {
  const batch = ids.slice(i, i + batchSize);
  db.products.find({ _id: { $in: batch } });
}

// ✅ Ou : remodeler pour éviter cette requête
// Utiliser embedded ou autre pattern
```

---

### ❌ Anti-Pattern 7 : count() Sur Grandes Collections

**Problème** : count() peut être lent sans filtre approprié

```javascript
// ❌ Lent : count sans index
db.orders.count({ status: "pending" })

// ✅ Bon : countDocuments (plus précis)
db.orders.countDocuments({ status: "pending" })

// ✅ Rapide mais approximatif : estimatedDocumentCount
db.orders.estimatedDocumentCount()  // Utilise les métadonnées

// ✅ Meilleur : pré-calculer si besoin fréquent (pattern Computed)
{
  _id: "stats",
  pendingOrdersCount: 1234,
  lastUpdated: ISODate()
}
```

---

### ❌ Anti-Pattern 8 : limit() Sans sort()

**Problème** : Résultats non déterministes

```javascript
// ❌ Mauvais : limite sans tri
db.posts.find().limit(10)  // Résultats arbitraires

// ✅ Bon : toujours trier si on limite
db.posts.find().sort({ createdAt: -1 }).limit(10)
```

---

### ❌ Anti-Pattern 9 : skip() Pour Pagination

**Problème** : skip() devient très lent pour grandes offsets

```javascript
// ❌ Très lent : skip(10000) examine 10000 documents
db.posts.find().sort({ _id: 1 }).skip(10000).limit(20)

// ✅ Bon : pagination par curseur
// Page 1
const page1 = db.posts.find().sort({ _id: 1 }).limit(20);
const lastId = page1[page1.length - 1]._id;

// Page 2 (utilise le dernier _id)
db.posts.find({ _id: { $gt: lastId } }).sort({ _id: 1 }).limit(20)
```

---

## Optimisation des Agrégations

### 📊 Principes Généraux

```markdown
✅ $match le plus tôt possible dans le pipeline
✅ $project pour réduire la taille des documents
✅ Utiliser les index dès les premières étapes
✅ $limit après $match pour réduire le volume
✅ Éviter $unwind sur tableaux volumineux
✅ Utiliser allowDiskUse pour grandes agrégations
```

### Ordre Optimal des Stages

```javascript
// ❌ Mauvais ordre
db.orders.aggregate([
  { $unwind: "$items" },           // Explose les documents
  { $lookup: { ... } },            // Jointure sur tous
  { $match: { status: "paid" } },  // Filtre en dernier
  { $sort: { total: -1 } }
])

// ✅ Bon ordre
db.orders.aggregate([
  { $match: { status: "paid" } },    // 1. Filtre d'abord (utilise index)
  { $sort: { total: -1 } },          // 2. Tri (utilise index si disponible)
  { $limit: 100 },                   // 3. Limite tôt
  { $lookup: { ... } },              // 4. Jointure sur résultat réduit
  { $unwind: "$items" },             // 5. Unwind en dernier
  { $project: { ... } }              // 6. Projection finale
])
```

### ✅ $match Optimization

```javascript
// Splitter les $match complexes
// ❌ Un seul $match complexe
{ $match: {
    status: "active",
    createdAt: { $gte: date },
    category: { $in: ["tech", "books"] }
  }
}

// ✅ Séparer pour meilleure utilisation des index
{ $match: { status: "active" } },    // Utilise index
{ $match: { category: { $in: ["tech", "books"] } } },
{ $match: { createdAt: { $gte: date } } }
```

### ✅ $lookup Optimization

```javascript
// ❌ $lookup sans pipeline
{ $lookup: {
    from: "users",
    localField: "userId",
    foreignField: "_id",
    as: "user"
  }
}

// ✅ $lookup avec pipeline pour filtrer tôt
{ $lookup: {
    from: "users",
    let: { userId: "$userId" },
    pipeline: [
      { $match: {
          $expr: { $eq: ["$_id", "$$userId"] },
          status: "active"  // Filtre supplémentaire
        }
      },
      { $project: { name: 1, email: 1 } }  // Seulement champs nécessaires
    ],
    as: "user"
  }
}
```

### ✅ $group Optimization

```javascript
// Utiliser les accumulateurs efficacement
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: {
      _id: "$userId",
      totalSpent: { $sum: "$total" },      // ✅ Efficace
      avgOrder: { $avg: "$total" },        // ✅ Efficace
      orderCount: { $sum: 1 },             // ✅ Efficace
      orders: { $push: "$$ROOT" }          // ⚠️ Peut être lourd
    }
  }
])

// Si besoin de tous les documents, limiter les champs
{ $group: {
    _id: "$userId",
    orders: {
      $push: {
        _id: "$_id",
        total: "$total"
      }  // Projection dans $push
    }
  }
}
```

### ✅ allowDiskUse

Pour agrégations volumineuses dépassant 100 Mo en RAM :

```javascript
db.orders.aggregate(
  [ /* pipeline */ ],
  { allowDiskUse: true }
)

// ⚠️ Plus lent mais permet de traiter de gros volumes
```

---

## Monitoring en Temps Réel

### 🔍 currentOp()

Voir les opérations en cours :

```javascript
// Toutes les opérations
db.currentOp()

// Seulement les opérations actives
db.currentOp({ "$all": true })

// Opérations longues (> 1 seconde)
db.currentOp({
  "active": true,
  "secs_running": { "$gt": 1 }
})

// Par collection
db.currentOp({
  "ns": "mydb.orders"
})
```

### 🛑 killOp()

Tuer une opération problématique :

```javascript
// Obtenir l'opId depuis currentOp()
const ops = db.currentOp({
  "active": true,
  "secs_running": { "$gt": 10 }
});

// Tuer l'opération
db.killOp(ops.inprog[0].opid)
```

---

## Scripts d'Audit Automatisés

### 📊 Script Complet d'Audit

```javascript
function auditQueries(dbName, minutes = 60) {
  const db = db.getSiblingDB(dbName);

  print("===================================");
  print("AUDIT DES REQUÊTES");
  print("Base : " + dbName);
  print("Période : " + minutes + " dernières minutes");
  print("===================================\n");

  const since = new Date(Date.now() - minutes * 60 * 1000);

  // 1. Requêtes les plus lentes
  print("--- TOP 10 REQUÊTES LES PLUS LENTES ---");
  db.system.profile.aggregate([
    { $match: { ts: { $gte: since } } },
    { $sort: { millis: -1 } },
    { $limit: 10 },
    { $project: {
        ns: 1,
        op: 1,
        millis: 1,
        planSummary: 1,
        ts: 1
      }
    }
  ]).forEach(doc => {
    print(doc.ns + " - " + doc.op + " - " + doc.millis + "ms - " + doc.planSummary);
  });

  print("\n--- COLLSCAN DÉTECTÉS ---");
  const collscans = db.system.profile.countDocuments({
    ts: { $gte: since },
    planSummary: "COLLSCAN"
  });
  print("Nombre de COLLSCAN : " + collscans);

  if (collscans > 0) {
    db.system.profile.aggregate([
      { $match: {
          ts: { $gte: since },
          planSummary: "COLLSCAN"
        }
      },
      { $group: {
          _id: "$ns",
          count: { $sum: 1 },
          avgMs: { $avg: "$millis" }
        }
      },
      { $sort: { count: -1 } }
    ]).forEach(doc => {
      print("  " + doc._id + " : " + doc.count + " fois (avg: " + doc.avgMs.toFixed(2) + "ms)");
    });
  }

  print("\n--- REQUÊTES LES PLUS FRÉQUENTES ---");
  db.system.profile.aggregate([
    { $match: { ts: { $gte: since } } },
    { $group: {
        _id: {
          ns: "$ns",
          op: "$op"
        },
        count: { $sum: 1 },
        avgMs: { $avg: "$millis" },
        maxMs: { $max: "$millis" }
      }
    },
    { $sort: { count: -1 } },
    { $limit: 10 }
  ]).forEach(doc => {
    print(doc._id.ns + " (" + doc._id.op + ") : " + doc.count + " fois");
    print("  Avg: " + doc.avgMs.toFixed(2) + "ms, Max: " + doc.maxMs + "ms");
  });

  print("\n===================================");
}

// Utilisation
auditQueries("mydb", 60);  // Dernière heure
```

### 📊 Script de Détection d'Anti-Patterns

```javascript
function detectAntiPatterns(dbName) {
  const db = db.getSiblingDB(dbName);

  print("=== DÉTECTION D'ANTI-PATTERNS ===\n");

  // Regex non ancré
  print("--- $regex non ancré ---");
  const regexQueries = db.system.profile.countDocuments({
    "command.filter": {
      $exists: true
    },
    $where: function() {
      const filter = JSON.stringify(this.command.filter);
      return filter.includes("$regex") && !filter.includes("^");
    }
  });
  if (regexQueries > 0) {
    print("⚠️  " + regexQueries + " requêtes avec regex non ancré détectées");
  }

  // $where usage
  print("\n--- Utilisation de $where ---");
  const whereQueries = db.system.profile.countDocuments({
    "command.filter.$where": { $exists: true }
  });
  if (whereQueries > 0) {
    print("⚠️  " + whereQueries + " requêtes avec $where détectées");
  }

  // Tri sans index
  print("\n--- Tri en mémoire ---");
  db.system.profile.find({
    "execStats.stage": "SORT"
  }).limit(10).forEach(doc => {
    print("⚠️  " + doc.ns + " - Tri en mémoire détecté");
  });

  print("\n=== FIN ===");
}

detectAntiPatterns("mydb");
```

---

## Checklist par Type de Requête

### 📖 Requêtes Find Simples

```markdown
□ Filtre utilise un index (IXSCAN)
□ Projection explicite des champs nécessaires
□ Ratio nReturned/totalDocsExamined > 0.8
□ executionTimeMillis < 50ms
□ Pas de COLLSCAN sur collections > 10k docs
```

### 📖 Requêtes Find avec Tri

```markdown
□ Index composé couvre filtre + tri
□ Pas de stage SORT (tri fait par index)
□ Direction de tri cohérente avec index
□ limit() utilisé pour limiter le résultat
```

### 📊 Agrégations

```markdown
□ $match en première position
□ $match utilise un index
□ $limit utilisé après $match
□ $project réduit la taille des docs
□ $unwind seulement si nécessaire
□ allowDiskUse: true si volume > 100 Mo
□ Pipeline optimisé (< 5 stages si possible)
```

### 🔢 Count

```markdown
□ countDocuments() pour précision
□ estimatedDocumentCount() si approximation OK
□ Éviter count() sans filtre
□ Pattern Computed si count() fréquent
```

### 🔍 Recherche Texte

```markdown
□ Index texte créé
□ $text utilisé plutôt que $regex
□ Projection des champs nécessaires
□ Considérer Atlas Search pour recherche avancée
```

---

## Actions Prioritaires

### 🔴 Critique - À corriger immédiatement

```markdown
□ COLLSCAN sur collections > 100k documents
□ Requêtes > 1 seconde
□ totalDocsExamined > 10x nReturned
□ Tri en mémoire sur collections volumineuses
□ Pattern N+1 détecté sur endpoints critiques
□ $where ou JavaScript côté serveur
```

### 🟠 Important - À planifier sous 2 semaines

```markdown
□ Requêtes 100-1000ms
□ COLLSCAN sur collections 10k-100k documents
□ Projections absentes sur documents volumineux
□ $in avec > 1000 valeurs
□ skip() > 1000 pour pagination
□ Agrégations > 500ms
□ Regex non ancré sur champs indexés
```

### 🟡 Modéré - À améliorer progressivement

```markdown
□ Requêtes 50-100ms
□ Ratio nReturned/totalDocsExamined < 0.8
□ Index non couvrants pour requêtes fréquentes
□ Agrégations non optimisées (ordre stages)
□ count() au lieu de countDocuments()
□ Absence de limit() sur requêtes
```

---

## Template de Rapport d'Audit

```markdown
# Rapport d'Audit de Requêtes
**Date** : [DATE]
**Base de données** : [NOM]
**Période analysée** : [DURÉE]
**Auditeur** : [NOM]

## Résumé Exécutif
- Requêtes analysées : X
- Requêtes lentes (> 100ms) : X
- COLLSCAN détectés : X
- Anti-patterns identifiés : X
- Impact estimé des optimisations : X%

## Métriques Globales
| Métrique | Valeur | Statut |
|----------|--------|--------|
| P50 latence | Xms | 🟢/🟡/🔴 |
| P95 latence | Xms | 🟢/🟡/🔴 |
| P99 latence | Xms | 🟢/🟡/🔴 |
| % COLLSCAN | X% | 🟢/🟡/🔴 |
| Requêtes > 1s | X | 🟢/🟡/🔴 |

## Top 10 Requêtes Problématiques

### 1. Collection.operation
- **Fréquence** : X fois/heure
- **Latence moyenne** : Xms
- **Problème** : [COLLSCAN / Tri mémoire / etc.]
- **Impact** : [DESCRIPTION]
- **Solution** : [RECOMMANDATION]

[...]

## Anti-Patterns Détectés

### N+1 Queries
- Endpoint : [URL]
- Collection : [NOM]
- Impact : X requêtes au lieu de 1
- Solution : $lookup ou Extended Reference

### Pagination avec skip()
- Endpoint : [URL]
- Impact : Latence augmente avec page
- Solution : Cursor-based pagination

[...]

## Recommandations

### Immédiat (< 3 jours)
1. Créer index pour COLLSCAN critiques
2. Optimiser requête [X] (latence > 1s)
3. Corriger pattern N+1 sur [ENDPOINT]

### Court terme (< 2 semaines)
1. Refactorer agrégations complexes
2. Implémenter pagination par curseur
3. Ajouter projections sur requêtes volumineuses

### Moyen terme (1-2 mois)
1. Migrer vers Atlas Search
2. Implémenter pattern Computed
3. Audit complet des agrégations

## Actions Techniques
```javascript
// Index à créer
db.collection.createIndex({ field1: 1, field2: -1 });

// Requêtes à optimiser
db.collection.find({ ... })
  .project({ field1: 1, field2: 1 })
  .limit(100);
```

## Impact Estimé
- Réduction latence P95 : X%
- Économie CPU : X%
- Amélioration UX : [DESCRIPTION]

## Annexes
- Captures explain()
- Logs profiler
- Scripts utilisés
```

---

## Ressources Complémentaires

### Documentation Officielle
- [Analyze Query Performance](https://www.mongodb.com/docs/manual/tutorial/analyze-query-plan/)
- [Database Profiler](https://www.mongodb.com/docs/manual/tutorial/manage-the-database-profiler/)
- [Optimization Tips](https://www.mongodb.com/docs/manual/core/query-optimization/)

### Guides Avancés
- [Query Performance Troubleshooting](https://www.mongodb.com/docs/manual/tutorial/troubleshoot-query-performance/)
- [Aggregation Pipeline Optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/)

### Outils
- **MongoDB Compass** : Query Performance Tab
- **Atlas Performance Advisor** : Recommandations automatiques
- **explain()** : Analyse détaillée
- **mongostat/mongotop** : Monitoring temps réel

---


⏭️ [Audit d'infrastructure](/annexes/checklist-performance/04-audit-infrastructure.md)
