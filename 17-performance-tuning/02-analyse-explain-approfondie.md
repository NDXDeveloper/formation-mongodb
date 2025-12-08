🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 17.2 Analyse avec explain() Approfondie

## Introduction

La commande `explain()` est l'outil le plus puissant pour comprendre et optimiser l'exécution des requêtes MongoDB. Contrairement à une simple inspection des résultats, `explain()` révèle les mécanismes internes du Query Planner, les stratégies d'exécution choisies, et les métriques précises de performance. Maîtriser l'analyse approfondie d'`explain()` est essentiel pour toute optimisation de production.

Cette section détaille l'interprétation experte de chaque composant de la sortie `explain()`, les patterns problématiques à identifier, et les méthodologies d'optimisation basées sur cette analyse.

## Les Trois Modes d'Explain

MongoDB propose trois niveaux de verbosité pour `explain()`, chacun avec un coût d'exécution et un niveau de détail différent.

### queryPlanner : Analyse Statique

Le mode `queryPlanner` effectue uniquement la phase de planification sans exécuter la requête.

```javascript
db.collection.find({ field: value }).explain("queryPlanner")
```

**Caractéristiques :**
- **Performance** : Très rapide, pas d'accès aux données
- **Informations** : Plan d'exécution choisi, plans rejetés, index utilisés
- **Usage** : Vérification rapide de la stratégie sans impact sur le système

**Limites :**
- Pas de métriques réelles d'exécution
- Ne révèle pas les problèmes de cardinalité ou de distribution des données
- Estimations du Query Planner peuvent être inexactes

### executionStats : Métriques d'Exécution

Le mode `executionStats` exécute la requête et collecte des statistiques détaillées.

```javascript
db.collection.find({ field: value }).explain("executionStats")
```

**Caractéristiques :**
- **Performance** : Exécution complète de la requête
- **Informations** : Métriques réelles (documents examinés, temps d'exécution, working memory)
- **Usage** : Analyse de performance réelle, identification des goulots

**Important :**
- La requête est exécutée mais les résultats ne sont pas retournés au client
- Les write operations sont annulées (rollback automatique)
- Impact sur le cache (données chargées en mémoire)

### allPlansExecution : Comparaison Complète

Le mode `allPlansExecution` exécute tous les plans candidats en parallèle pendant la phase de trial.

```javascript
db.collection.find({ field: value }).explain("allPlansExecution")
```

**Caractéristiques :**
- **Performance** : Coût élevé, exécution de multiples plans
- **Informations** : Comparaison de tous les plans candidats avec leurs scores
- **Usage** : Debug du Query Planner, compréhension des choix de plan

**Attention :**
- Très coûteux en production
- Peut masquer des problèmes de performance (trial period court)
- Réservé au diagnostic approfondi en environnement contrôlé

## Anatomie de la Sortie explain()

### Structure Globale

```javascript
{
  "queryPlanner": { /* Plan d'exécution */ },
  "executionStats": { /* Statistiques d'exécution */ },
  "serverInfo": { /* Info serveur */ },
  "ok": 1
}
```

### Section queryPlanner

#### Champs Essentiels

**plannerVersion** :
```javascript
"plannerVersion": 1
```
Version du Query Planner. MongoDB utilise le même planner depuis la version 2.6 (version 1).

**namespace** :
```javascript
"namespace": "mydb.users"
```
Base de données et collection concernées.

**indexFilterSet** :
```javascript
"indexFilterSet": false
```
Indique si un index filter est appliqué (manipulation manuelle de la sélection d'index).

**parsedQuery** :
```javascript
"parsedQuery": {
  "$and": [
    { "age": { "$gte": 25 } },
    { "status": { "$eq": "active" } }
  ]
}
```
Représentation canonique de la requête après parsing et normalisation.

**queryHash** :
```javascript
"queryHash": "8B3D4AB8"
```
Hash de la forme de la requête (query shape). Identique pour requêtes avec même structure mais valeurs différentes.
**Usage** : Regrouper les requêtes similaires dans le profiler ou les logs.

**planCacheKey** :
```javascript
"planCacheKey": "5C8B3D4A"
```
Clé du plan dans le cache. Combine le queryHash avec les index disponibles.
**Usage** : Identifier si le plan est mis en cache entre les exécutions.

#### winningPlan : Le Plan Sélectionné

**Structure hiérarchique des stages** :

```javascript
"winningPlan": {
  "stage": "FETCH",
  "inputStage": {
    "stage": "IXSCAN",
    "keyPattern": { "age": 1, "status": 1 },
    "indexName": "age_1_status_1",
    "isMultiKey": false,
    "multiKeyPaths": { "age": [], "status": [] },
    "isUnique": false,
    "isSparse": false,
    "isPartial": false,
    "indexVersion": 2,
    "direction": "forward",
    "indexBounds": {
      "age": ["[25.0, inf.0]"],
      "status": ["[\"active\", \"active\"]"]
    }
  }
}
```

**Analyse des stages** :

Les stages représentent les étapes d'exécution, organisées en arbre inversé (bottom-up execution).

**Stages courants et leur signification** :

| Stage | Signification | Performance |
|-------|---------------|-------------|
| **COLLSCAN** | Collection scan complète | ❌ Très mauvais (sauf petites collections) |
| **IXSCAN** | Index scan | ✅ Bon |
| **FETCH** | Récupération du document complet | ⚠️ Coût variable (dépend du nombre) |
| **SORT** | Tri en mémoire | ⚠️ Coûteux (limité à 32 MB) |
| **SORT_KEY_GENERATOR** | Génération des clés de tri | ⚠️ Overhead additionnel |
| **PROJECTION** | Projection des champs | ✅ Bon (réduit les transferts) |
| **LIMIT** | Limitation du nombre de résultats | ✅ Bon (early termination) |
| **SKIP** | Saut des premiers résultats | ⚠️ Coûteux (pas d'early termination) |
| **COUNT** | Comptage | ✅ Peut utiliser les index |
| **COUNT_SCAN** | Comptage via index scan | ✅ Très efficace |
| **TEXT** | Recherche full-text | ⚠️ Coût variable |
| **GEO_NEAR_2D** / **GEO_NEAR_2DSPHERE** | Recherche géospatiale | ⚠️ Coût variable |
| **CACHED_PLAN** | Plan récupéré du cache | ✅ Très bon |
| **SUBPLAN** | Sous-plan pour requêtes $or | ⚠️ Complexité accrue |
| **SHARD_MERGE** | Merge des résultats de shards | ⚠️ Coût réseau |

**Analyse de IXSCAN** :

```javascript
"stage": "IXSCAN",
"keyPattern": { "age": 1, "status": 1 },
"indexBounds": {
  "age": ["[25.0, inf.0]"],      // Range scan sur age
  "status": ["[\"active\", \"active\"]"]  // Point query sur status
}
```

**Points d'attention** :
- **indexBounds** : Vérifier l'utilisation effective des champs de l'index
- Bounds trop larges (`[MinKey, MaxKey]`) : Index partiellement utilisé
- Multi-range bounds : Peut indiquer une requête $in ou $or

**isMultiKey** :
```javascript
"isMultiKey": true,
"multiKeyPaths": {
  "tags": ["tags"],  // Le champ tags est un array
  "name": []         // Le champ name n'est pas un array
}
```

**Impact** :
- Index multikey ont des contraintes spécifiques
- Pas de compound index avec plusieurs champs array
- Performance légèrement réduite (plus de clés d'index par document)

**Direction de scan** :
```javascript
"direction": "backward"  // Scan inverse pour supporter un sort DESC
```

Peut indiquer un tri nécessitant un scan inverse de l'index.

#### rejectedPlans : Les Plans Non Retenus

```javascript
"rejectedPlans": [
  {
    "stage": "FETCH",
    "inputStage": {
      "stage": "IXSCAN",
      "keyPattern": { "status": 1 },
      "indexName": "status_1"
    }
  }
]
```

**Analyse** :
- Plans candidats testés mais non retenus
- Peut révéler des index sous-utilisés ou redondants
- Comprendre pourquoi rejetés aide à optimiser les index

**Scénarios d'analyse** :
- Si meilleur plan potentiel rejeté : Investiguer le score du trial
- Beaucoup de rejectedPlans : Trop d'index, confusion du planner
- Plan covering rejeté : Possible problème de statistiques

### Section executionStats

#### Métriques Globales

```javascript
"executionStats": {
  "executionSuccess": true,
  "nReturned": 145,
  "executionTimeMillis": 47,
  "totalKeysExamined": 145,
  "totalDocsExamined": 145,
  "executionStages": { /* ... */ }
}
```

**executionSuccess** :
- `true` : Exécution normale
- `false` : Erreur lors de l'exécution (vérifier `errorMessage`)

**nReturned** :
Nombre de documents retournés au client (après tous les filtres).

**executionTimeMillis** :
Temps total d'exécution en millisecondes.

**Analyse temporelle** :
- < 10ms : Excellent
- 10-100ms : Acceptable
- 100-1000ms : Préoccupant
- > 1000ms : Critique

**Attention** : Le temps inclut la planification + exécution. Pour les requêtes rapides, la planification peut représenter une portion significative.

**totalKeysExamined** :
Nombre de clés d'index examinées.

**totalDocsExamined** :
Nombre de documents examinés (après FETCH).

#### Ratios Critiques

**Ratio Index Efficiency** :
```javascript
indexEfficiency = totalKeysExamined / nReturned
```

**Interprétation** :
- = 1 : Parfait (index covering ou sélectivité parfaite)
- 1-2 : Excellent
- 2-10 : Bon à acceptable
- 10-100 : Problématique (index peu sélectif)
- > 100 : Critique (quasi collection scan)

**Ratio Fetch Overhead** :
```javascript
fetchOverhead = totalDocsExamined / nReturned
```

**Interprétation** :
- = 1 : Optimal (tous les documents récupérés sont retournés)
- > 1 : Documents récupérés mais filtrés ensuite (post-fetch filtering)
- >> 1 : Filtrage inefficace, index incomplet

**Exemple d'analyse** :
```javascript
// Requête : db.users.find({ age: 25, status: "active" })
// Index : { age: 1 }

executionStats: {
  nReturned: 10,
  totalKeysExamined: 150,    // Toutes les entrées avec age=25
  totalDocsExamined: 150     // Tous les documents récupérés
}

// Analyse :
// indexEfficiency = 150/10 = 15 (problématique)
// fetchOverhead = 150/10 = 15 (filtrage status après fetch)
// Solution : Compound index { age: 1, status: 1 }
```

#### executionStages : Détail par Stage

Structure récursive avec métriques par stage :

```javascript
"executionStages": {
  "stage": "FETCH",
  "nReturned": 145,
  "executionTimeMillisEstimate": 45,
  "works": 146,
  "advanced": 145,
  "needTime": 0,
  "needYield": 0,
  "saveState": 1,
  "restoreState": 1,
  "isEOF": 1,
  "docsExamined": 145,
  "inputStage": {
    "stage": "IXSCAN",
    "nReturned": 145,
    "executionTimeMillisEstimate": 2,
    "works": 146,
    "advanced": 145,
    "needTime": 0,
    "needYield": 0,
    "keysExamined": 145,
    /* ... */
  }
}
```

**Métriques par stage** :

**works** :
Nombre total d'unités de travail (cycles) effectuées par ce stage.
- `works = advanced + needTime + needYield`

**advanced** :
Nombre de fois où le stage a produit un résultat pour le stage parent.

**needTime** :
Nombre de cycles où le stage a travaillé mais n'a pas produit de résultat.

**needYield** :
Nombre de fois où le stage a yielded (libéré les locks pour d'autres opérations).

**Analyse avancée** :
```javascript
// Ratio d'efficacité du stage
stageEfficiency = advanced / works

// Proche de 1.0 : Très efficace (presque chaque cycle produit un résultat)
// Proche de 0.0 : Inefficace (beaucoup de travail pour peu de résultats)
```

**saveState / restoreState** :
Nombre de fois où le stage a sauvé/restauré son état (lors de yields).
- Élevé : Contention élevée ou requête longue avec beaucoup de yields

**executionTimeMillisEstimate** :
Temps estimé passé dans ce stage spécifique.
- Permet d'identifier quel stage consomme le plus de temps
- Somme des stages enfants ≤ temps du stage parent

**Exemple d'analyse temporelle** :
```javascript
FETCH: 45ms
  └─ IXSCAN: 2ms

// Analyse :
// - Index scan très rapide (2ms)
// - Fetch overhead de 43ms (45-2)
// - Possible : Documents larges ou cold cache
// - Action : Vérifier taille documents ou utiliser projection
```

### Stage SORT : Analyse Approfondie

Le stage SORT est particulièrement critique car limité en mémoire.

```javascript
"stage": "SORT",
"sortPattern": { "timestamp": -1 },
"memLimit": 33554432,  // 32 MB limit
"type": "simple",      // ou "default"
"totalDataSizeSorted": 12582912,  // ~12 MB
"usedDisk": false,
"inputStage": { /* ... */ }
```

**memLimit** :
Limite de mémoire pour le tri (défaut : 32 MB, configurable via `allowDiskUse`).

**totalDataSizeSorted** :
Quantité totale de données triées.

**Analyse critique** :
```javascript
if (totalDataSizeSorted > memLimit && !usedDisk) {
  // ERREUR : Sort limit exceeded
  // La requête échouera si allowDiskUse: false
}

if (usedDisk) {
  // Performance dégradée : tri sur disque
  // Action : Créer index supportant le sort
}
```

**type: "simple" vs "default"** :
- **simple** : Tri basique d'un seul champ
- **default** : Tri complexe (multiple champs, composé)

**Optimisation** :
Le sort peut être éliminé si un index supporte l'ordre de tri :

```javascript
// Requête avec sort
db.users.find({ status: "active" }).sort({ age: -1 })

// Index optimal (élimine le SORT stage)
db.users.createIndex({ status: 1, age: -1 })

// Résultat explain :
// Pas de stage SORT, directement IXSCAN avec direction: "backward"
```

### Stage COLLSCAN : Collection Scan

```javascript
"stage": "COLLSCAN",
"filter": { "age": { "$gte": 25 } },
"direction": "forward",
"docsExamined": 1000000
```

**Quand COLLSCAN est acceptable** :
1. Collection très petite (< 1000 documents)
2. Requête retourne > 30% de la collection (full scan peut être plus rapide)
3. Pas d'index approprié disponible et création impossible

**Quand COLLSCAN est critique** :
1. Collection volumineuse (> 100,000 documents)
2. Requête sélective (< 5% de résultats attendus)
3. Requête fréquente (impact cumulatif)

**Action immédiate** :
Créer un index approprié sur les champs du filtre.

## Analyse des Requêtes Complexes

### Requêtes avec $or

```javascript
db.users.find({
  $or: [
    { email: "user@example.com" },
    { username: "user123" }
  ]
}).explain("executionStats")
```

**Plan typique** :
```javascript
"stage": "SUBPLAN",
"inputStage": {
  "stage": "OR",
  "inputStages": [
    {
      "stage": "IXSCAN",
      "indexName": "email_1"
    },
    {
      "stage": "IXSCAN",
      "indexName": "username_1"
    }
  ]
}
```

**Analyse** :
- SUBPLAN : Le planner génère un plan pour chaque branche du $or
- OR : Merge des résultats (union)
- Chaque branche devrait utiliser un index

**Problème courant** :
Si une branche fait un COLLSCAN, toute la requête est pénalisée :

```javascript
"inputStages": [
  { "stage": "IXSCAN", "indexName": "email_1" },
  { "stage": "COLLSCAN" }  // ❌ Branche sans index
]

// Action : Créer index sur tous les champs du $or
```

**Performance** :
- $or avec N branches = N requêtes indépendantes
- Déduplication des résultats nécessaire
- Peut être moins efficace que deux requêtes séparées

### Requêtes avec Aggregation

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: { _id: "$customerId", total: { $sum: "$amount" } } },
  { $sort: { total: -1 } },
  { $limit: 10 }
]).explain("executionStats")
```

**Structure explain pour aggregation** :
```javascript
"stages": [
  {
    "$cursor": {
      "queryPlanner": { /* ... */ },
      "executionStats": { /* ... */ }
    }
  },
  { "$group": { /* ... */ } },
  { "$sort": { "sortKey": { "total": -1 } } },
  { "$limit": 10 }
]
```

**Points d'attention** :

**$cursor stage** :
La partie qui s'exécute via le query engine (peut utiliser les index).

**Pipeline stages** :
Exécutés après le $cursor, dans le framework d'agrégation.

**Optimisations automatiques** :

1. **$match pushdown** :
```javascript
// Pipeline original
[ { $project: {...} }, { $match: {...} } ]

// Optimisé automatiquement
[ { $match: {...} }, { $project: {...} } ]
```

2. **$sort + $limit fusion** :
```javascript
// Pipeline original
[ { $sort: {...} }, { $limit: N } ]

// Optimisé en top-K sort (plus efficace)
```

3. **Index utilization** :
```javascript
// $match peut utiliser un index
{ $match: { field: value } }

// $sort peut utiliser un index si premier stage ou après $match
{ $sort: { indexedField: 1 } }
```

**Analyse des métriques** :

```javascript
"executionStats": {
  "nReturned": 10,
  "executionTimeMillis": 1234,
  "totalKeysExamined": 50000,
  "totalDocsExamined": 50000,
  "executionStages": { /* ... */ }
}
```

Pour les agrégations, vérifier particulièrement :
- **totalDocsExamined** : Si trop élevé, le $match n'est pas assez sélectif
- **executionTimeMillis** : Temps dans le query engine vs temps total pipeline

### Requêtes Géospatiales

```javascript
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [2.3522, 48.8566] },
      $maxDistance: 1000
    }
  }
}).explain("executionStats")
```

**Plan typique** :
```javascript
"stage": "GEO_NEAR_2DSPHERE",
"keyPattern": { "location": "2dsphere" },
"indexName": "location_2dsphere",
"indexVersion": 3,
"searchStage": {
  "searchTimeMillis": 45,
  "numCandidates": 150,
  "numReturned": 12
}
```

**Métriques spécifiques** :

**numCandidates** :
Nombre de documents candidats identifiés par l'index géospatial.

**numReturned** :
Nombre de documents finalement retournés après calcul précis des distances.

**Ratio de précision** :
```javascript
precision = numReturned / numCandidates

// Proche de 1.0 : Index très précis
// Proche de 0.0 : Beaucoup de faux positifs, raffinement coûteux
```

**searchTimeMillis** :
Temps passé dans la recherche géospatiale spécifiquement.

### Requêtes avec Lookup (Jointures)

```javascript
db.orders.aggregate([
  {
    $lookup: {
      from: "customers",
      localField: "customerId",
      foreignField: "_id",
      as: "customer"
    }
  }
]).explain("executionStats")
```

**Analyse du $lookup** :
```javascript
"$lookup": {
  "from": "customers",
  "as": "customer",
  "localField": "customerId",
  "foreignField": "_id",
  "unwinding": { "preserveNullAndEmptyArrays": false },
  "matching": { /* ... */ }
}
```

**Performance critique** :

Le $lookup effectue une requête sur la collection "from" pour **chaque document** de la collection source.

```javascript
// Si orders contient 10,000 documents
// Et chaque lookup fait un IXSCAN
// = 10,000 requêtes individuelles

// Temps total = N_orders × temps_lookup
```

**Optimisation** :

1. **Index sur foreignField** :
```javascript
// ESSENTIEL : Index sur customers._id
db.customers.createIndex({ _id: 1 })  // Déjà présent par défaut

// Pour d'autres champs :
db.customers.createIndex({ email: 1 })
```

2. **Filtrer avant le $lookup** :
```javascript
// Mauvais : Lookup puis filter
[ { $lookup: {...} }, { $match: {...} } ]

// Bon : Filter puis lookup
[ { $match: {...} }, { $lookup: {...} } ]
```

3. **Utiliser $match dans le pipeline du lookup** :
```javascript
{
  $lookup: {
    from: "customers",
    let: { custId: "$customerId" },
    pipeline: [
      { $match: { $expr: { $eq: ["$_id", "$$custId"] } } },
      { $match: { status: "active" } }  // Filter dans le lookup
    ],
    as: "customer"
  }
}
```

## Patterns Problématiques et Diagnostics

### Pattern 1 : "The Million Row Shuffle"

**Signature explain** :
```javascript
executionStats: {
  nReturned: 10,
  totalKeysExamined: 1000000,
  totalDocsExamined: 1000000,
  executionTimeMillis: 5234
}

// Ratio : 100,000:1 (catastrophique)
```

**Cause** :
Index non sélectif ou ordre des champs dans compound index incorrect.

**Diagnostic** :
```javascript
// Vérifier indexBounds
"indexBounds": {
  "status": ["[\"active\", \"active\"]"],  // Point query
  "timestamp": ["[MinKey, MaxKey]"]        // ❌ Range scan illimité
}
```

**Solution** :
Réordonner l'index : champs d'égalité avant les ranges.

```javascript
// Avant : { timestamp: 1, status: 1 }
// Après : { status: 1, timestamp: 1 }
```

### Pattern 2 : "The Hidden SORT"

**Signature explain** :
```javascript
"stage": "SORT",
"memLimit": 33554432,
"totalDataSizeSorted": 31457280,  // Proche de la limite
"usedDisk": false
```

**Risque** :
Proche de la limite mémoire. Peut échouer si le dataset grossit légèrement.

**Diagnostic** :
Requête performante aujourd'hui, mais bombe à retardement.

**Solution préventive** :
Créer un index supportant le tri même si actuellement "ça passe".

### Pattern 3 : "The Projection Trap"

**Signature explain** :
```javascript
executionStats: {
  nReturned: 100,
  totalKeysExamined: 100,      // ✅ Bon ratio
  totalDocsExamined: 100,      // ❌ Fetch inutile
  executionTimeMillis: 89
}

winningPlan: {
  "stage": "PROJECTION_COVERED",  // ❌ Nom trompeur
  "transformBy": { _id: 0, name: 1, email: 1 },
  "inputStage": {
    "stage": "FETCH",  // ❌ Fetch présent
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "name_1_email_1"
    }
  }
}
```

**Tromperie** :
Le nom "PROJECTION_COVERED" suggère une covered query, mais la présence de FETCH indique le contraire.

**Cause** :
Projection inclut un champ non présent dans l'index, forçant le fetch.

**Solution** :
Covering index complet :
```javascript
db.collection.createIndex({ name: 1, email: 1, _id: 1 })
// Note : _id est toujours récupéré sauf si explicitement exclu
```

### Pattern 4 : "The Scatter-Gather Explosion"

**Signature explain (sharded cluster)** :
```javascript
"shards": {
  "shard01": { /* plan */ },
  "shard02": { /* plan */ },
  "shard03": { /* plan */ },
  // ... tous les shards contactés
},
"executionTimeMillis": 2345,
"nReturned": 50
```

**Cause** :
Requête ne contient pas la shard key, forçant une interrogation de tous les shards.

**Diagnostic** :
```javascript
// Vérifier si requête contient la shard key
parsedQuery: {
  "status": "active"  // ❌ Pas la shard key
}

// Shard key : { userId: 1 }
```

**Solution** :
Inclure la shard key dans toutes les requêtes critiques :
```javascript
// Avant
db.orders.find({ status: "active" })

// Après
db.orders.find({ userId: currentUserId, status: "active" })
```

### Pattern 5 : "The Index Intersection Fallacy"

**Signature explain** :
```javascript
"stage": "AND_SORTED",  // ou "AND_HASH"
"inputStages": [
  { "stage": "IXSCAN", "indexName": "field1_1" },
  { "stage": "IXSCAN", "indexName": "field2_1" }
]
```

**Analyse** :
Index intersection (utilisation de deux index simples).

**Performance** :
Généralement **moins efficace** qu'un compound index approprié.

**Cause** :
Query planner choisit l'intersection faute de compound index.

**Solution** :
Créer un compound index :
```javascript
db.collection.createIndex({ field1: 1, field2: 1 })
```

## Méthodologies d'Optimisation Basées sur explain()

### Workflow d'Optimisation Systématique

```
1. IDENTIFIER
   └─> Exécuter explain("executionStats")
   └─> Calculer les ratios critiques

2. DIAGNOSTIQUER
   └─> Analyser le winningPlan
   └─> Identifier les stages problématiques
   └─> Vérifier indexBounds et selectivité

3. HYPOTHÈSE
   └─> Formuler une optimisation potentielle
   └─> Estimer l'impact attendu

4. VALIDER
   └─> Créer l'index ou modifier la requête en dev/staging
   └─> Réexécuter explain et comparer
   └─> Vérifier absence de régression

5. DEPLOYER
   └─> Online index build en production
   └─> Monitoring de l'impact réel
   └─> Rollback si nécessaire
```

### Checklist d'Analyse Expert

**Niveau 1 : Rapide (30 secondes)**
```
☐ nReturned vs totalDocsExamined (ratio < 10 ?)
☐ Présence de COLLSCAN ? (mauvais sauf petite collection)
☐ Présence de SORT ? (index manquant pour le tri ?)
☐ executionTimeMillis acceptable ? (< 100ms idéalement)
```

**Niveau 2 : Détaillé (5 minutes)**
```
☐ Analyse des indexBounds (tous les champs utilisés ?)
☐ Index multikey ? (impact performance ?)
☐ RejectedPlans intéressants ? (meilleur plan possible ?)
☐ Works vs Advanced ratio par stage (efficacité ?)
☐ executionTimeMillisEstimate par stage (goulot ?)
```

**Niveau 3 : Expert (30 minutes)**
```
☐ Comparaison avec allPlansExecution (planner optimal ?)
☐ Analyse des selectivité des index bounds
☐ Review du parsedQuery (optimisations possibles ?)
☐ Vérification cache warming (cold vs warm cache ?)
☐ Impact sur write operations (overhead des index ?)
☐ Projection optimization possible ?
☐ Query shape variation (parameterized queries ?)
```

### Comparaison Avant/Après

Méthodologie pour mesurer l'impact d'une optimisation :

```javascript
// Capturer les métriques AVANT
const before = db.collection.find(query).explain("executionStats");
const beforeMetrics = {
  time: before.executionStats.executionTimeMillis,
  keysExamined: before.executionStats.totalKeysExamined,
  docsExamined: before.executionStats.totalDocsExamined,
  returned: before.executionStats.nReturned
};

// Créer l'index optimisé
db.collection.createIndex({ field1: 1, field2: 1 });

// Capturer les métriques APRÈS
const after = db.collection.find(query).explain("executionStats");
const afterMetrics = {
  time: after.executionStats.executionTimeMillis,
  keysExamined: after.executionStats.totalKeysExamined,
  docsExamined: after.executionStats.totalDocsExamined,
  returned: after.executionStats.nReturned
};

// Calcul des améliorations
const improvement = {
  timeReduction: ((beforeMetrics.time - afterMetrics.time) / beforeMetrics.time * 100).toFixed(2) + '%',
  keysReduction: ((beforeMetrics.keysExamined - afterMetrics.keysExamined) / beforeMetrics.keysExamined * 100).toFixed(2) + '%',
  docsReduction: ((beforeMetrics.docsExamined - afterMetrics.docsExamined) / beforeMetrics.docsExamined * 100).toFixed(2) + '%'
};

printjson(improvement);
```

## Limitations et Pièges de explain()

### Limitation 1 : Cache Effects

**Problème** :
`explain()` peut donner des résultats très différents selon l'état du cache.

```javascript
// Première exécution (cold cache)
db.collection.find(query).explain("executionStats")
// executionTimeMillis: 1250ms

// Deuxième exécution (warm cache)
db.collection.find(query).explain("executionStats")
// executionTimeMillis: 45ms
```

**Mitigation** :
1. Exécuter explain() plusieurs fois et prendre la médiane
2. Vider le cache entre les tests (restart mongod) pour tests reproductibles
3. Considérer les deux scénarios dans l'analyse

### Limitation 2 : Estimations du Query Planner

Les `nReturned` estimés dans `queryPlanner` peuvent être très inexacts :

```javascript
"estimatedNReturned": 5000  // Estimation du planner
// vs
"nReturned": 12  // Résultat réel dans executionStats
```

**Cause** :
Statistiques de distribution des données imparfaites ou obsolètes.

**Impact** :
Le planner peut choisir un plan sous-optimal basé sur de mauvaises estimations.

### Limitation 3 : Plan Cache Staleness

Le plan mis en cache peut devenir obsolète si les données évoluent :

```javascript
// Plan initial choisi avec 1000 documents
// Après 6 mois : 10,000,000 documents
// Le plan initial peut ne plus être optimal
```

**Mitigation** :
```javascript
// Vider le plan cache périodiquement
db.collection.getPlanCache().clear();

// Ou pour un query shape spécifique
db.collection.getPlanCache().clear("query shape hash");
```

### Limitation 4 : explain() sur Write Operations

Les write operations avec `explain()` sont **toujours rollback** :

```javascript
db.collection.update(query, update).explain("executionStats")
// L'update EST exécuté pour les métriques
// MAIS est rollback automatiquement
```

**Attention** :
- Locks acquis quand même (peut impacter la prod)
- Triggers et validations sont exécutés
- Coût comparable à l'opération réelle

## Automatisation de l'Analyse explain()

### Script d'Audit Automatisé

```javascript
// Fonction d'analyse automatique des requêtes profilées
function analyzeSlowQueries(thresholdMs = 100) {
  const slowQueries = db.system.profile.find({
    millis: { $gt: thresholdMs },
    ts: { $gte: new ISODate(new Date().getTime() - 3600000) }  // Dernière heure
  }).toArray();

  const results = [];

  slowQueries.forEach(function(query) {
    if (query.command && query.command.find) {
      try {
        const collection = db.getSiblingDB(query.ns.split('.')[0])[query.ns.split('.')[1]];
        const explained = collection.find(query.command.filter)
                                    .sort(query.command.sort || {})
                                    .explain("executionStats");

        const stats = explained.executionStats;
        const ratio = stats.totalDocsExamined / (stats.nReturned || 1);

        if (ratio > 10) {
          results.push({
            namespace: query.ns,
            filter: query.command.filter,
            ratio: ratio,
            executionTime: stats.executionTimeMillis,
            recommendation: ratio > 100 ? "CRITICAL: Add index" : "WARNING: Optimize index"
          });
        }
      } catch (e) {
        print(`Error analyzing query: ${e}`);
      }
    }
  });

  return results.sort((a, b) => b.ratio - a.ratio);
}

// Exécution
const inefficientQueries = analyzeSlowQueries(100);
printjson(inefficientQueries);
```

### Monitoring Continu avec Aggregation

```javascript
// Pipeline d'analyse du profiler
db.system.profile.aggregate([
  // Dernières 24 heures
  {
    $match: {
      ts: { $gte: new ISODate(new Date().getTime() - 86400000) },
      "command.find": { $exists: true }
    }
  },
  // Grouper par query shape
  {
    $group: {
      _id: {
        ns: "$ns",
        operation: "$op"
      },
      count: { $sum: 1 },
      avgMs: { $avg: "$millis" },
      maxMs: { $max: "$millis" },
      p95Ms: { $percentile: { input: "$millis", p: [0.95], method: 'approximate' } },
      avgDocsExamined: { $avg: "$docsExamined" },
      avgNReturned: { $avg: "$nreturned" }
    }
  },
  // Calculer le ratio
  {
    $addFields: {
      efficiency: {
        $cond: [
          { $gt: ["$avgNReturned", 0] },
          { $divide: ["$avgDocsExamined", "$avgNReturned"] },
          999999
        ]
      }
    }
  },
  // Filtrer les inefficaces
  {
    $match: {
      efficiency: { $gt: 10 }
    }
  },
  // Trier par impact (fréquence × ratio)
  {
    $addFields: {
      impact: { $multiply: ["$count", "$efficiency"] }
    }
  },
  { $sort: { impact: -1 } },
  { $limit: 20 }
])
```

## Conclusion

L'analyse approfondie avec `explain()` est un art autant qu'une science. La maîtrise de cet outil requiert :

1. **Compréhension profonde** des structures de données explain()
2. **Calcul et interprétation** des ratios critiques
3. **Identification** des patterns problématiques
4. **Méthodologie rigoureuse** de diagnostic et optimisation
5. **Conscience des limitations** et pièges de l'outil

Les optimisations basées sur `explain()` doivent toujours être :
- **Mesurées** : Comparaison avant/après avec métriques objectives
- **Testées** : Validation en environnement non-production d'abord
- **Monitorées** : Suivi de l'impact réel en production
- **Documentées** : Traçabilité des décisions et résultats

La section suivante abordera l'optimisation concrète de la modélisation des données basée sur les insights obtenus via `explain()`.

---

**Points clés à retenir :**
- Les trois modes d'explain() ont des usages et coûts différents
- Le ratio totalDocsExamined/nReturned est l'indicateur le plus critique
- Analyser les stages hiérarchiquement pour identifier les goulots
- COLLSCAN n'est pas toujours mauvais, IXSCAN n'est pas toujours bon
- L'automatisation de l'analyse permet un monitoring continu
- Toujours valider les optimisations avant déploiement production

⏭️ [Optimisation de la modélisation](/17-performance-tuning/03-optimisation-modelisation.md)
