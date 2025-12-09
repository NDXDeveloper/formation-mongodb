🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.2 - Audit d'Indexation

## Introduction

L'audit d'indexation évalue la **stratégie d'index** de vos collections pour maximiser les performances de lecture tout en minimisant l'impact sur les écritures. Les index sont cruciaux pour les performances mais doivent être utilisés judicieusement.

### 🎯 Objectif

Identifier les index manquants, redondants ou inutilisés, et optimiser la stratégie globale d'indexation.

### ⏱️ Durée estimée
- Audit rapide : 30-45 minutes
- Audit complet : 2-3 heures

---

## Règles d'Or de l'Indexation

### 📌 Principes Fondamentaux

```markdown
✅ Un index par pattern de requête fréquent
✅ Index composés pour requêtes multi-champs
✅ Ordre des champs : Égalité → Tri → Plage
✅ Index couvrants pour requêtes critiques
✅ Supprimer les index inutilisés (< 1% utilisation)
✅ Maximum 10-15 index par collection (guideline)
✅ Surveiller le ratio index size / data size
```

### ⚖️ Balance Performance

```
📖 Lectures rapides ←→ 📝 Écritures ralenties
    Plus d'index           Moins d'index
```

---

## Checklist Générale

### 📊 Inventaire des Index

#### ✅ Vue d'Ensemble

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Liste complète des index disponible | 🟢 | Documentation |
| Index par collection < 15 | 🟡 | Limiter la prolifération |
| Index size < 50% RAM | 🟠 | Performances |
| Index fit entièrement en RAM | 🔴 | Critique pour performance |

**Commandes de base** :
```javascript
// Lister tous les index d'une collection
db.collection.getIndexes()

// Statistiques des index
db.collection.stats().indexSizes

// Taille totale des index
db.collection.stats().totalIndexSize

// RAM disponible vs index size
db.serverStatus().mem
```

**Analyse rapide** :
```javascript
// Script : audit complet des index
function auditIndexes(collectionName) {
  const coll = db.getCollection(collectionName);
  const stats = coll.stats();
  const indexes = coll.getIndexes();

  print("=== Audit Index : " + collectionName + " ===");
  print("Nombre d'index : " + indexes.length);
  print("Taille totale index : " + (stats.totalIndexSize / 1024 / 1024).toFixed(2) + " Mo");
  print("Taille données : " + (stats.size / 1024 / 1024).toFixed(2) + " Mo");
  print("Ratio index/data : " + ((stats.totalIndexSize / stats.size) * 100).toFixed(1) + "%");

  print("\n--- Liste des index ---");
  indexes.forEach((idx, i) => {
    print((i + 1) + ". " + idx.name);
    print("   Clés : " + JSON.stringify(idx.key));
    if (idx.unique) print("   [UNIQUE]");
    if (idx.sparse) print("   [SPARSE]");
    if (idx.partialFilterExpression) print("   [PARTIAL]");
  });
}

// Utilisation
auditIndexes("products");
```

---

### 🔍 Couverture des Requêtes

#### ✅ Requêtes Indexées

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Requêtes fréquentes utilisent un index | 🔴 | Obligatoire |
| Pas de COLLSCAN sur collections volumineuses | 🔴 | Critique |
| Requêtes de tri utilisent un index | 🟠 | Performance |
| Index couvrants pour requêtes critiques | 🟡 | Optimisation |

**Identifier les requêtes sans index** :
```javascript
// 1. Activer le profiler (niveau 2 = toutes requêtes, ou 1 = lentes uniquement)
db.setProfilingLevel(2, { slowms: 100 })

// 2. Après quelques minutes, analyser
db.system.profile.find({
  planSummary: "COLLSCAN",
  ns: /^mydb\./  // votre base
}).sort({ ts: -1 }).limit(20)

// 3. Requêtes lentes sans index
db.system.profile.find({
  millis: { $gt: 100 },
  planSummary: "COLLSCAN"
}).sort({ millis: -1 })

// 4. Désactiver après analyse
db.setProfilingLevel(0)
```

**Vérifier une requête spécifique** :
```javascript
// Analyse avec explain()
db.collection.find({ status: "active", category: "tech" })
  .sort({ createdAt: -1 })
  .explain("executionStats")

// Points à vérifier dans le résultat :
// - winningPlan.stage devrait être "IXSCAN" pas "COLLSCAN"
// - totalDocsExamined devrait être proche de nReturned
// - executionTimeMillis < 50ms pour requêtes simples
```

**Indicateurs de problème** :
```markdown
⚠️ stage: "COLLSCAN" sur collections > 10k documents
⚠️ totalDocsExamined >> nReturned (ratio > 10)
⚠️ executionTimeMillis > 100ms pour requêtes simples
⚠️ SORT en mémoire pour tri volumineux
⚠️ rejectedPlans non vide (index concurrent non optimal)
```

---

#### ✅ Index Couvrants (Covered Queries)

Les index couvrants permettent de répondre à une requête **sans accéder aux documents**.

**Conditions** :
```markdown
1. Tous les champs de la requête sont dans l'index
2. Tous les champs de la projection sont dans l'index
3. Aucun champ du document n'est nécessaire
4. _id doit être explicitement exclu de la projection
```

**Exemple** :
```javascript
// Index
db.users.createIndex({ status: 1, email: 1, name: 1 })

// ✅ Requête couverte
db.users.find(
  { status: "active" },
  { _id: 0, email: 1, name: 1 }  // Projection explicite sans _id
)

// Vérifier avec explain()
// Dans le résultat : totalDocsExamined: 0 et stage: "IXSCAN"
```

**Checklist** :
```markdown
□ Requêtes les plus fréquentes analysées
□ Index couvrants créés pour top 5-10 requêtes
□ Projection explicite avec _id: 0
□ Vérification avec explain("executionStats")
```

---

### 🗑️ Index Inutilisés et Redondants

#### ✅ Détection des Index Inutilisés

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Index utilisés < 1 fois/jour | 🟡 | Candidats à suppression |
| Index jamais utilisés depuis création | 🟠 | Supprimer |
| Index redondants identifiés | 🟠 | Consolider |

**Commandes de détection** :
```javascript
// Statistiques d'utilisation des index (MongoDB 3.2+)
db.collection.aggregate([
  { $indexStats: {} }
])

// Trier par utilisation
db.collection.aggregate([
  { $indexStats: {} },
  { $sort: { "accesses.ops": 1 } }
])

// Index jamais utilisés
db.collection.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": 0 } }
])

// Index utilisés < 10 fois
db.collection.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 10 } } }
])
```

**Script d'analyse complet** :
```javascript
function analyzeIndexUsage(collectionName) {
  const coll = db.getCollection(collectionName);

  print("=== Analyse utilisation des index : " + collectionName + " ===\n");

  coll.aggregate([
    { $indexStats: {} },
    { $sort: { "accesses.ops": 1 } }
  ]).forEach(idx => {
    print("Index: " + idx.name);
    print("  Clés: " + JSON.stringify(idx.key));
    print("  Accès: " + idx.accesses.ops);
    print("  Depuis: " + idx.accesses.since);

    if (idx.accesses.ops === 0) {
      print("  ⚠️  JAMAIS UTILISÉ - Candidat à suppression");
    } else if (idx.accesses.ops < 10) {
      print("  ⚠️  Très peu utilisé - À vérifier");
    }
    print("");
  });
}

analyzeIndexUsage("products");
```

---

#### ✅ Détection des Index Redondants

**Définition** : Un index est redondant s'il est un **préfixe** d'un autre index.

**Exemples** :
```javascript
// ❌ Redondant
db.collection.createIndex({ status: 1 })
db.collection.createIndex({ status: 1, createdAt: 1 })
// Le premier est redondant (préfixe du second)

// ❌ Redondant
db.collection.createIndex({ email: 1 })
db.collection.createIndex({ email: 1 }, { unique: true })
// Le premier est redondant

// ✅ NON redondant (ordre différent)
db.collection.createIndex({ status: 1, createdAt: 1 })
db.collection.createIndex({ createdAt: 1, status: 1 })
// Différents, car ordre compte pour les requêtes
```

**Script de détection** :
```javascript
function findRedundantIndexes(collectionName) {
  const indexes = db.getCollection(collectionName).getIndexes();

  print("=== Détection index redondants : " + collectionName + " ===\n");

  for (let i = 0; i < indexes.length; i++) {
    for (let j = i + 1; j < indexes.length; j++) {
      const idx1Keys = Object.keys(indexes[i].key);
      const idx2Keys = Object.keys(indexes[j].key);

      // Vérifier si idx1 est préfixe de idx2
      if (idx1Keys.length < idx2Keys.length) {
        const isPrefix = idx1Keys.every((key, pos) => idx2Keys[pos] === key);
        if (isPrefix) {
          print("⚠️  Index redondant détecté:");
          print("   " + indexes[i].name + " : " + JSON.stringify(indexes[i].key));
          print("   est préfixe de");
          print("   " + indexes[j].name + " : " + JSON.stringify(indexes[j].key));
          print("");
        }
      }
    }
  }
}

findRedundantIndexes("users");
```

**Action** :
```javascript
// Avant de supprimer, cacher l'index pour tester
db.collection.hideIndex("redundant_index_name")

// Surveiller les performances pendant 24-48h

// Si OK, supprimer définitivement
db.collection.dropIndex("redundant_index_name")

// Si problème, réactiver
db.collection.unhideIndex("redundant_index_name")
```

---

### 📐 Stratégie d'Index Composés

#### ✅ Règle ESR (Equality, Sort, Range)

L'ordre des champs dans un index composé est **crucial** :

```
1. Equality (=, $in)  - Filtre d'égalité
2. Sort              - Tri
3. Range (>, <, !=)  - Filtre de plage
```

**Exemples** :
```javascript
// Requête
db.orders.find({
  status: "shipped",        // Equality
  total: { $gt: 100 }       // Range
}).sort({
  createdAt: -1             // Sort
})

// ✅ Index optimal (ESR)
db.orders.createIndex({
  status: 1,      // E - Equality
  createdAt: -1,  // S - Sort
  total: 1        // R - Range
})

// ❌ Mauvais ordre
db.orders.createIndex({
  total: 1,       // Range en premier = inefficace
  status: 1,
  createdAt: -1
})
```

**Validation** :
```javascript
// Vérifier l'ordre avec explain()
db.orders.find({
  status: "shipped",
  total: { $gt: 100 }
}).sort({ createdAt: -1 }).explain("executionStats")

// Vérifier :
// - stage: "IXSCAN" (pas COLLSCAN)
// - totalDocsExamined proche de nReturned
// - Pas de SORT stage (tri fait par l'index)
```

---

#### ✅ Direction des Index (1 vs -1)

**Règles** :
```markdown
✅ Pour tri uni-directionnel : direction peu importante
✅ Pour tri bi-directionnel : direction critique
✅ Pour index composé : inverser toutes les directions = équivalent
```

**Exemples** :
```javascript
// Index
db.events.createIndex({ userId: 1, timestamp: -1 })

// ✅ Peut servir ces requêtes :
db.events.find({ userId: 123 }).sort({ timestamp: -1 })  // ✅
db.events.find({ userId: 123 }).sort({ timestamp: 1 })   // ✅ (reverse scan)

// ❌ Ne peut PAS servir efficacement :
db.events.find().sort({ userId: 1, timestamp: 1 })       // ❌ (directions opposées)

// Pour supporter les deux cas :
db.events.createIndex({ userId: 1, timestamp: -1 })
db.events.createIndex({ userId: 1, timestamp: 1 })  // Ou inverse: -1, -1
```

---

### 🎯 Index Spécialisés

#### ✅ Index Texte (Full-Text Search)

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| 1 seul index texte par collection | 🔴 | Limite MongoDB |
| Champs prioritaires weightés | 🟡 | Optimisation |
| Language correctement défini | 🟡 | Pertinence recherche |

```javascript
// Créer index texte avec poids
db.articles.createIndex(
  { title: "text", content: "text", tags: "text" },
  {
    weights: { title: 10, content: 5, tags: 1 },
    default_language: "french",
    name: "text_search_index"
  }
)

// Utilisation
db.articles.find({ $text: { $search: "mongodb performance" } })

// ⚠️ Limitation : 1 seul index texte par collection
```

**Alternative recommandée** : **Atlas Search** pour recherche avancée

---

#### ✅ Index Géospatiaux

```javascript
// 2dsphere pour données géographiques modernes
db.locations.createIndex({ coordinates: "2dsphere" })

// Requête
db.locations.find({
  coordinates: {
    $near: {
      $geometry: { type: "Point", coordinates: [2.3522, 48.8566] },
      $maxDistance: 5000  // mètres
    }
  }
})
```

---

#### ✅ Index TTL (Time-To-Live)

Pour suppression automatique des documents expirés :

```javascript
// Créer index TTL (expire après 30 jours)
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 2592000 }  // 30 * 24 * 60 * 60
)

// Vérifier
db.sessions.getIndexes()

// ⚠️ Le champ doit être de type Date
// ⚠️ MongoDB vérifie toutes les 60 secondes
```

**Checklist** :
```markdown
□ Champ indexé est de type Date
□ expireAfterSeconds approprié
□ Pas d'index TTL sur collections critiques sans backup
```

---

#### ✅ Index Partiels (Partial)

Indexer seulement un **sous-ensemble** de documents :

```javascript
// Index uniquement pour documents actifs
db.users.createIndex(
  { email: 1 },
  {
    partialFilterExpression: { status: "active" },
    name: "active_users_email_idx"
  }
)

// ✅ Utilise l'index
db.users.find({ email: "user@example.com", status: "active" })

// ❌ N'utilise PAS l'index (pas de filtre status)
db.users.find({ email: "user@example.com" })

// ❌ N'utilise PAS l'index (filtre différent)
db.users.find({ email: "user@example.com", status: "inactive" })
```

**Avantages** :
- Index plus petit
- Mises à jour plus rapides
- Économie de RAM

---

#### ✅ Index Sparse

Indexer seulement les documents qui **ont le champ** :

```javascript
// Index sparse
db.users.createIndex(
  { optionalField: 1 },
  { sparse: true }
)

// Seuls les documents avec optionalField sont indexés
```

**Différence avec Partial** :
```markdown
Sparse   : filtre sur { champ: { $exists: true } }
Partial  : filtre personnalisé avec partialFilterExpression
```

---

#### ✅ Index Wildcard

Pour schémas très dynamiques ou polymorphes :

```javascript
// Index tous les champs sous attributes
db.products.createIndex({ "attributes.$**": 1 })

// Permet de requêter n'importe quel champ
db.products.find({ "attributes.color": "red" })
db.products.find({ "attributes.size": "L" })
db.products.find({ "attributes.brand": "Nike" })

// ⚠️ Moins efficace qu'un index spécifique
// Utiliser uniquement si vraiment nécessaire
```

---

### ⚡ Impact sur les Performances

#### ✅ Coût des Écritures

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Ratio lecture/écriture connu | 🟠 | Adapter stratégie |
| Impact mesuré sur insertions | 🟡 | Benchmark |
| Bulk operations utilisées | 🟡 | Performance |

**Impact estimé** :
```markdown
0 index    : 1x temps d'écriture
3 index    : ~1.3-1.5x temps d'écriture
5 index    : ~1.5-2x temps d'écriture
10 index   : ~2-3x temps d'écriture
```

**Mesurer l'impact** :
```javascript
// Sans index
var start = new Date();
for (let i = 0; i < 1000; i++) {
  db.test.insertOne({ field1: i, field2: "value" + i });
}
var withoutIndex = new Date() - start;
print("Sans index: " + withoutIndex + "ms");

// Avec index
db.test.createIndex({ field1: 1 });
db.test.createIndex({ field2: 1 });

var start = new Date();
for (let i = 1000; i < 2000; i++) {
  db.test.insertOne({ field1: i, field2: "value" + i });
}
var withIndex = new Date() - start;
print("Avec index: " + withIndex + "ms");

print("Ratio: " + (withIndex / withoutIndex).toFixed(2) + "x");
```

---

#### ✅ Mémoire (RAM)

**Règle critique** : Les index doivent tenir en RAM pour des performances optimales.

```javascript
// Vérifier RAM disponible
db.serverStatus().mem

// Taille index vs RAM
function checkIndexMemory(collectionName) {
  const stats = db.getCollection(collectionName).stats();
  const mem = db.serverStatus().mem;

  print("RAM totale : " + mem.resident + " Mo");
  print("Taille index : " + (stats.totalIndexSize / 1024 / 1024).toFixed(2) + " Mo");
  print("Ratio : " + ((stats.totalIndexSize / (mem.resident * 1024 * 1024)) * 100).toFixed(1) + "%");

  if (stats.totalIndexSize > mem.resident * 1024 * 1024 * 0.5) {
    print("⚠️  ATTENTION: Index utilisent > 50% de la RAM");
  }
}
```

---

## Checklist par Type de Requête

### 🔍 Requêtes de Lecture Simple

```javascript
// Requête
db.users.find({ status: "active" })

// ✅ Index requis
db.users.createIndex({ status: 1 })
```

### 🔍 Requêtes avec Tri

```javascript
// Requête
db.posts.find({ published: true }).sort({ createdAt: -1 })

// ✅ Index composé
db.posts.createIndex({ published: 1, createdAt: -1 })
```

### 🔍 Requêtes avec Plage

```javascript
// Requête
db.orders.find({
  status: "pending",
  total: { $gte: 100, $lte: 1000 }
})

// ✅ Index composé (Equality puis Range)
db.orders.createIndex({ status: 1, total: 1 })
```

### 🔍 Requêtes avec $in

```javascript
// Requête
db.products.find({ category: { $in: ["tech", "books", "toys"] } })

// ✅ Index simple
db.products.createIndex({ category: 1 })

// $in se comporte comme Equality dans la règle ESR
```

### 🔍 Requêtes Multi-Critères

```javascript
// Requête complexe
db.events.find({
  type: "login",                    // Equality
  userId: { $in: [1, 2, 3] },      // Equality ($in)
  timestamp: { $gte: ISODate() }   // Range
}).sort({ timestamp: -1 })          // Sort

// ✅ Index optimal (ESR)
db.events.createIndex({
  type: 1,       // E
  userId: 1,     // E
  timestamp: -1  // S et R
})
```

---

## Outils d'Analyse Avancés

### 🔧 MongoDB Compass

**Index Tab** :
- Liste visuelle des index
- Taille et utilisation
- Suggestions d'index

**Performance Tab** :
- Requêtes lentes en temps réel
- Recommandations automatiques

---

### 🔧 Atlas Performance Advisor

```markdown
✅ Recommandations automatiques d'index
✅ Analyse des requêtes lentes
✅ Détection des index inutilisés
✅ Suggestions d'index composés
✅ Impact estimé des changements
```

---

### 🔧 Profiler Analysis

```javascript
// Script : analyser les requêtes du profiler pour suggestions d'index
db.system.profile.aggregate([
  { $match: {
      planSummary: "COLLSCAN",
      ns: "mydb.users"
    }
  },
  { $group: {
      _id: "$command.filter",
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 10 }
]).forEach(result => {
  print("Requête fréquente sans index:");
  print("Filtre: " + JSON.stringify(result._id));
  print("Occurrences: " + result.count);
  print("Temps moyen: " + result.avgTime.toFixed(2) + "ms");
  print("---");
});
```

---

## Matrice de Décision

### Type d'Index à Créer

```
Index Simple
├─ 1 seul champ
├─ Requêtes simples
└─ Pas de tri ou plage

Index Composé
├─ Multi-champs dans WHERE
├─ Requêtes avec tri
├─ Suivre règle ESR
└─ Ordre des champs crucial

Index Texte
├─ Recherche full-text
├─ 1 seul par collection
└─ Ou utiliser Atlas Search

Index Géospatial
├─ Coordonnées GPS
├─ Requêtes $near, $geoWithin
└─ Utiliser 2dsphere

Index TTL
├─ Expiration automatique
├─ Champ Date requis
└─ Sessions, logs, cache

Index Partiel
├─ Sous-ensemble de documents
├─ Économie RAM
└─ Requêtes avec condition

Index Wildcard
├─ Schéma très dynamique
├─ Dernière option
└─ Moins performant
```

---

## Actions Prioritaires

### 🔴 Critique - À corriger immédiatement

```markdown
□ COLLSCAN sur collections > 100k documents
□ Requêtes critiques > 1 seconde
□ Index size > RAM disponible
□ Pas d'index sur clés étrangères fréquemment jointes
□ Index texte manquant pour recherche full-text
```

### 🟠 Important - À planifier sous 2 semaines

```markdown
□ Index inutilisés depuis > 30 jours
□ Index redondants identifiés
□ Requêtes lentes 100-1000ms
□ Ratio index/data > 100%
□ Plus de 15 index par collection
□ Index non couvrants pour requêtes fréquentes
```

### 🟡 Modéré - À améliorer progressivement

```markdown
□ Index sans statistiques d'utilisation
□ Ordre des champs non optimal (ESR)
□ Absence d'index partiels où applicable
□ Direction des index (-1/1) non optimale
□ Index composés trop larges (> 5 champs)
```

---

## Template de Rapport d'Audit

```markdown
# Rapport d'Audit d'Indexation
**Date** : [DATE]
**Collection(s)** : [NOMS]
**Auditeur** : [NOM]

## Résumé Exécutif
- Collections auditées : X
- Index totaux : X
- Index inutilisés : X
- Index redondants : X
- Recommandations prioritaires : X

## Métriques Globales
| Métrique | Valeur | Statut |
|----------|--------|--------|
| Index par collection | X | 🟢/🟡/🔴 |
| Index size total | X Mo | 🟢/🟡/🔴 |
| Ratio index/RAM | X% | 🟢/🟡/🔴 |
| COLLSCAN détectés | X | 🟢/🟡/🔴 |

## Index Problématiques

### Index Inutilisés
1. [NOM_INDEX] - Collection: [X]
   - Dernière utilisation : [DATE]
   - Action : Supprimer après test

### Index Redondants
1. [INDEX_1] redondant avec [INDEX_2]
   - Action : Conserver [INDEX_2], supprimer [INDEX_1]

### Index Manquants
1. Collection [X] - Requête : [PATTERN]
   - Impact : [DESCRIPTION]
   - Index suggéré : [SPEC]

## Recommandations

### Court terme (< 1 semaine)
1. Créer index pour COLLSCAN critiques
2. Supprimer index jamais utilisés

### Moyen terme (1-4 semaines)
1. Optimiser index composés (ESR)
2. Implémenter index couvrants
3. Supprimer index redondants

### Long terme (> 1 mois)
1. Migration vers index partiels
2. Réévaluation stratégie globale

## Actions Immédiates
```javascript
// Index à créer
db.collection.createIndex({ field1: 1, field2: -1 });

// Index à supprimer
db.collection.dropIndex("redundant_index");
```

## Annexes
- Captures explain()
- Logs profiler
- Scripts utilisés
```

---

## Scripts Utilitaires

### Script Complet d'Audit

```javascript
function completeIndexAudit(dbName) {
  const db = db.getSiblingDB(dbName);
  const collections = db.getCollectionNames();

  print("===================================");
  print("AUDIT COMPLET DES INDEX");
  print("Base de données : " + dbName);
  print("===================================\n");

  let totalIndexes = 0;
  let unusedIndexes = 0;

  collections.forEach(collName => {
    if (collName.startsWith("system.")) return;

    const coll = db.getCollection(collName);
    const stats = coll.stats();
    const indexes = coll.getIndexes();

    print("Collection : " + collName);
    print("  Documents : " + stats.count);
    print("  Index : " + indexes.length);

    totalIndexes += indexes.length;

    // Statistiques d'utilisation
    try {
      const usage = coll.aggregate([{ $indexStats: {} }]).toArray();

      usage.forEach(idx => {
        if (idx.accesses.ops === 0) {
          print("  ⚠️  Index inutilisé : " + idx.name);
          unusedIndexes++;
        }
      });
    } catch(e) {
      print("  ℹ️  $indexStats non disponible");
    }

    print("");
  });

  print("===================================");
  print("RÉSUMÉ");
  print("===================================");
  print("Total index : " + totalIndexes);
  print("Index inutilisés : " + unusedIndexes);
  print("Collections : " + collections.length);
}

// Utilisation
completeIndexAudit("mydb");
```

---

## Ressources Complémentaires

### Documentation Officielle
- [Indexes](https://www.mongodb.com/docs/manual/indexes/)
- [Create Indexes](https://www.mongodb.com/docs/manual/reference/method/db.collection.createIndex/)
- [Index Types](https://www.mongodb.com/docs/manual/indexes/#index-types)
- [Analyze Query Performance](https://www.mongodb.com/docs/manual/tutorial/analyze-query-plan/)

### Guides Avancés
- [ESR Rule](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-rule/)
- [Index Strategies](https://www.mongodb.com/blog/post/performance-best-practices-indexing)

### Outils
- **MongoDB Compass** : Analyse visuelle
- **Atlas Performance Advisor** : Recommandations automatiques
- **explain()** : Analyse de plans d'exécution

---


⏭️ [Audit de requêtes](/annexes/checklist-performance/03-audit-requetes.md)
