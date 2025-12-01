🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.6 Analyse des requêtes avec explain()

## Introduction

La commande `explain()` est **l'outil le plus important** pour comprendre et optimiser les performances de vos requêtes MongoDB. Elle vous permet de voir "sous le capot" et de comprendre exactement comment MongoDB exécute vos requêtes.

Avec `explain()`, vous pouvez :
- 🔍 Voir si un **index est utilisé** ou non
- 📊 Mesurer le **nombre de documents examinés**
- ⏱️ Connaître le **temps d'exécution** réel
- 🎯 Identifier les **goulots d'étranglement**
- 🚀 Valider l'**impact des optimisations**

Maîtriser `explain()` est essentiel pour diagnostiquer les problèmes de performance et créer des applications MongoDB rapides et efficaces.

---

## Qu'est-ce que explain() ?

### Concept

`explain()` est une méthode qui vous montre le **plan d'exécution** d'une requête, c'est-à-dire la stratégie que MongoDB utilise pour récupérer les données.

### Analogie

```
Imaginez que vous demandez un itinéraire à un GPS :

Sans explain() :
Vous : "Emmène-moi à Paris"
GPS : *Vous conduit à Paris*
└─ Vous arrivez, mais vous ne savez pas quel chemin a été pris

Avec explain() :
Vous : "Emmène-moi à Paris + explique le trajet"
GPS : "Je vais prendre l'autoroute A1, puis la sortie 5,
      puis 3 ronds-points, durée estimée : 2h30"
└─ Vous comprenez la stratégie et pouvez l'optimiser
```

### Syntaxe de base

```javascript
// Ajouter .explain() à la fin de votre requête
db.collection.find({ ... }).explain()

// Avec différents niveaux de détail
db.collection.find({ ... }).explain("queryPlanner")      // Plan seulement
db.collection.find({ ... }).explain("executionStats")    // Plan + statistiques
db.collection.find({ ... }).explain("allPlansExecution") // Tous les plans testés
```

---

## Les trois modes d'explain()

MongoDB offre trois niveaux de verbosité pour `explain()` :

### 1. Mode "queryPlanner" (par défaut)

```javascript
db.users.find({ city: "Paris" }).explain()
// ou explicitement :
db.users.find({ city: "Paris" }).explain("queryPlanner")
```

**Ce qu'il retourne** :
- Le plan d'exécution choisi
- Les index disponibles
- Le plan de requête sélectionné
- **SANS exécuter réellement la requête**

**Quand l'utiliser** :
- Pour voir quel index sera utilisé
- Pour comprendre la stratégie globale
- Pas besoin des statistiques d'exécution

### 2. Mode "executionStats" (recommandé)

```javascript
db.users.find({ city: "Paris" }).explain("executionStats")
```

**Ce qu'il retourne** :
- Tout ce que retourne "queryPlanner"
- **+ Les statistiques d'exécution réelles** :
  - Nombre de documents examinés
  - Nombre de documents retournés
  - Temps d'exécution
  - Nombre de clés indexées examinées
- **Exécute réellement la requête**

**Quand l'utiliser** :
- Pour analyser les performances réelles
- Pour mesurer l'efficacité d'un index
- Pour comparer avant/après optimisation
- **C'est le mode le plus utilisé** ⭐

### 3. Mode "allPlansExecution"

```javascript
db.users.find({ city: "Paris" }).explain("allPlansExecution")
```

**Ce qu'il retourne** :
- Tout ce que retourne "executionStats"
- **+ Les détails de tous les plans testés** par le query planner
- Les raisons du choix du plan

**Quand l'utiliser** :
- Pour le debugging avancé
- Pour comprendre pourquoi un index est choisi plutôt qu'un autre
- Rarement nécessaire en pratique

### Comparaison visuelle

```
Niveau de détail et impact
═══════════════════════════

queryPlanner
├─ Détail : ⭐
├─ Exécution : ❌ Non
└─ Usage : Aperçu rapide

executionStats ⭐ RECOMMANDÉ
├─ Détail : ⭐⭐⭐
├─ Exécution : ✅ Oui
└─ Usage : Analyse de performance

allPlansExecution
├─ Détail : ⭐⭐⭐⭐⭐
├─ Exécution : ✅ Oui (tous les plans)
└─ Usage : Debugging avancé
```

---

## Structure d'un résultat explain()

### Exemple complet (mode executionStats)

```javascript
db.users.find({ city: "Paris" }).explain("executionStats")
```

**Résultat simplifié** :

```json
{
  "queryPlanner": {
    "namespace": "mydb.users",
    "indexFilterSet": false,
    "parsedQuery": {
      "city": { "$eq": "Paris" }
    },
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "keyPattern": { "city": 1 },
        "indexName": "city_1",
        "direction": "forward"
      }
    },
    "rejectedPlans": []
  },
  "executionStats": {
    "executionSuccess": true,
    "nReturned": 1500,
    "executionTimeMillis": 12,
    "totalKeysExamined": 1500,
    "totalDocsExamined": 1500,
    "executionStages": {
      "stage": "FETCH",
      "nReturned": 1500,
      "executionTimeMillisEstimate": 10,
      "works": 1501,
      "advanced": 1500,
      "inputStage": {
        "stage": "IXSCAN",
        "nReturned": 1500,
        "executionTimeMillisEstimate": 5,
        "indexName": "city_1",
        "keysExamined": 1500
      }
    }
  },
  "ok": 1
}
```

---

## Sections importantes du résultat

### 1. queryPlanner - Le plan choisi

#### a) winningPlan.stage

Le **stage** indique le type d'opération principal :

```
Stages courants et leur signification
══════════════════════════════════════

COLLSCAN (Collection Scan)
└─ ❌ MAUVAIS : Scan complet de la collection
└─ Aucun index utilisé
└─ Très lent sur grandes collections

IXSCAN (Index Scan)
└─ ✅ BON : Utilise un index
└─ Rapide et efficace
└─ C'est ce qu'on veut !

FETCH
└─ Récupération du document complet
└─ Souvent après un IXSCAN
└─ Normal et attendu

SORT
└─ Tri des résultats
└─ ⚠️ En mémoire si pas d'index
└─ Peut être coûteux

COUNT
└─ Comptage de documents
└─ Peut utiliser un index

TEXT
└─ Recherche full-text
└─ Utilise un index texte
```

#### b) indexName

Si un index est utilisé, vous verrez son nom :

```json
{
  "winningPlan": {
    "inputStage": {
      "stage": "IXSCAN",
      "indexName": "city_1",        // ← Index utilisé !
      "keyPattern": { "city": 1 }
    }
  }
}
```

### 2. executionStats - Les statistiques d'exécution

#### Métriques clés

##### a) nReturned

**Nombre de documents retournés** à l'utilisateur :

```json
{
  "executionStats": {
    "nReturned": 150    // 150 documents correspondent à la requête
  }
}
```

##### b) totalDocsExamined

**Nombre total de documents examinés** par MongoDB :

```json
{
  "executionStats": {
    "totalDocsExamined": 150    // MongoDB a examiné 150 documents
  }
}
```

##### c) totalKeysExamined

**Nombre de clés d'index examinées** :

```json
{
  "executionStats": {
    "totalKeysExamined": 150    // 150 clés d'index parcourues
  }
}
```

##### d) executionTimeMillis

**Temps d'exécution total** en millisecondes :

```json
{
  "executionStats": {
    "executionTimeMillis": 12   // 12 millisecondes
  }
}
```

### 3. Le ratio d'efficacité (le plus important !)

Le **ratio le plus important** à analyser :

```
Ratio d'efficacité = nReturned / totalDocsExamined

Interprétation :
════════════════

Ratio = 1.0 (100%)
└─ ✅ PARFAIT : Chaque document examiné correspond
└─ Index parfaitement ciblé

Ratio > 0.5 (>50%)
└─ ✅ BON : Efficacité acceptable
└─ Index utilisé correctement

Ratio < 0.1 (<10%)
└─ ⚠️ MOYEN : Beaucoup de documents inutiles examinés
└─ Index peut-être pas optimal

Ratio < 0.01 (<1%)
└─ ❌ MAUVAIS : Énormément de gaspillage
└─ Index manquant ou mal configuré
```

#### Exemples

```javascript
// Exemple 1 : PARFAIT (ratio = 1.0)
{
  "nReturned": 150,
  "totalDocsExamined": 150
}
// → 150/150 = 100% d'efficacité ✅

// Exemple 2 : ACCEPTABLE (ratio = 0.75)
{
  "nReturned": 750,
  "totalDocsExamined": 1000
}
// → 750/1000 = 75% d'efficacité ✅

// Exemple 3 : MAUVAIS (ratio = 0.0015)
{
  "nReturned": 150,
  "totalDocsExamined": 100000
}
// → 150/100000 = 0.15% d'efficacité ❌
// 99.85% des documents examinés sont inutiles !
```

---

## Interpréter les résultats : Scénarios courants

### Scénario 1 : Index utilisé efficacement ✅

```javascript
db.users.find({ email: "alice@example.com" }).explain("executionStats")
```

**Résultat** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",              // ✅ Index utilisé
        "indexName": "email_1"
      }
    }
  },
  "executionStats": {
    "nReturned": 1,                     // 1 document retourné
    "executionTimeMillis": 3,           // 3 ms - Rapide !
    "totalKeysExamined": 1,             // 1 clé examinée
    "totalDocsExamined": 1              // 1 document examiné
  }
}
```

**Interprétation** :
```
✅ Stage : IXSCAN (index utilisé)
✅ Ratio : 1/1 = 100% (parfait)
✅ Temps : 3ms (excellent)
✅ Index optimal, rien à faire !
```

### Scénario 2 : Pas d'index (COLLSCAN) ❌

```javascript
db.users.find({ city: "Paris" }).explain("executionStats")
```

**Résultat** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "COLLSCAN",              // ❌ Pas d'index !
      "direction": "forward"
    }
  },
  "executionStats": {
    "nReturned": 1500,                  // 1500 documents retournés
    "executionTimeMillis": 4823,        // 4.8 secondes - LENT !
    "totalKeysExamined": 0,             // Aucune clé (pas d'index)
    "totalDocsExamined": 1000000        // 1 million examinés !
  }
}
```

**Interprétation** :
```
❌ Stage : COLLSCAN (scan complet)
❌ Ratio : 1500/1000000 = 0.15% (terrible)
❌ Temps : 4823ms (très lent)
🔧 Solution : Créer un index sur "city"
```

**Solution** :

```javascript
// Créer l'index
db.users.createIndex({ city: 1 })

// Ré-exécuter la requête
db.users.find({ city: "Paris" }).explain("executionStats")

// Nouveau résultat :
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",              // ✅ Index utilisé maintenant !
        "indexName": "city_1"
      }
    }
  },
  "executionStats": {
    "nReturned": 1500,
    "executionTimeMillis": 12,          // 12ms au lieu de 4823ms !
    "totalKeysExamined": 1500,
    "totalDocsExamined": 1500           // 1500 au lieu de 1M !
  }
}

// Amélioration : 4823ms → 12ms = 400x plus rapide ! 🚀
```

### Scénario 3 : Index utilisé mais inefficace ⚠️

```javascript
db.orders.find({
  status: "pending",
  amount: { $gt: 100 }
}).explain("executionStats")
```

**Résultat avec index simple sur "status"** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",
      "filter": { "amount": { "$gt": 100 } },  // Filtrage après
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "status_1"
      }
    }
  },
  "executionStats": {
    "nReturned": 150,                   // 150 documents retournés
    "executionTimeMillis": 85,          // 85ms - Moyen
    "totalKeysExamined": 5000,          // 5000 clés examinées
    "totalDocsExamined": 5000           // 5000 documents examinés
  }
}
```

**Interprétation** :
```
⚠️  Stage : IXSCAN (index utilisé, mais...)
⚠️  Ratio : 150/5000 = 3% (inefficace)
⚠️  Temps : 85ms (peut être amélioré)
⚠️  Problème : Filtre "amount" appliqué APRÈS l'index
🔧 Solution : Index composé (status, amount)
```

**Solution optimale** :

```javascript
// Créer un index composé
db.orders.createIndex({ status: 1, amount: 1 })

// Ré-exécuter
db.orders.find({
  status: "pending",
  amount: { $gt: 100 }
}).explain("executionStats")

// Nouveau résultat :
{
  "executionStats": {
    "nReturned": 150,
    "executionTimeMillis": 8,           // 8ms au lieu de 85ms !
    "totalKeysExamined": 150,           // 150 au lieu de 5000
    "totalDocsExamined": 150            // 150 au lieu de 5000
  }
}

// Amélioration : 85ms → 8ms = 10x plus rapide ! 🚀
// Ratio : 150/150 = 100% (parfait) ✅
```

### Scénario 4 : Tri sans index ⚠️

```javascript
db.posts.find().sort({ publishedAt: -1 }).limit(10)
  .explain("executionStats")
```

**Résultat sans index** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "LIMIT",
      "inputStage": {
        "stage": "SORT",                // ⚠️ Tri en mémoire !
        "sortPattern": { "publishedAt": -1 },
        "memLimit": 104857600,          // Limite mémoire : 100 Mo
        "inputStage": {
          "stage": "COLLSCAN"
        }
      }
    }
  },
  "executionStats": {
    "nReturned": 10,
    "executionTimeMillis": 1523,        // 1.5 secondes - LENT
    "totalDocsExamined": 500000         // 500K documents chargés !
  }
}
```

**Interprétation** :
```
❌ Stage : SORT (tri en mémoire)
❌ Tous les documents chargés puis triés
❌ Temps : 1523ms (très lent)
❌ Risque : Erreur si > 100 Mo de données
🔧 Solution : Index sur "publishedAt"
```

**Solution** :

```javascript
// Créer l'index
db.posts.createIndex({ publishedAt: -1 })

// Ré-exécuter
db.posts.find().sort({ publishedAt: -1 }).limit(10)
  .explain("executionStats")

// Nouveau résultat :
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "LIMIT",
      "inputStage": {
        "stage": "IXSCAN",              // ✅ Utilise l'index pour le tri
        "indexName": "publishedAt_-1",
        "direction": "forward"          // Déjà trié !
      }
    }
  },
  "executionStats": {
    "nReturned": 10,
    "executionTimeMillis": 2,           // 2ms au lieu de 1523ms !
    "totalDocsExamined": 10             // Seulement 10 documents !
  }
}

// Amélioration : 1523ms → 2ms = 760x plus rapide ! 🚀
```

---

## Stages d'exécution détaillés

### Diagramme des stages courants

```
Pipeline d'exécution MongoDB
════════════════════════════

Requête simple avec index :
┌─────────────┐
│   IXSCAN    │  ← Parcourt l'index
└─────┬───────┘
      │
      ▼
┌─────────────┐
│    FETCH    │  ← Récupère les documents
└─────┬───────┘
      │
      ▼
   Résultats


Requête sans index :
┌─────────────┐
│  COLLSCAN   │  ← Parcourt toute la collection
└─────┬───────┘
      │
      ▼
   Résultats


Requête avec tri sans index :
┌─────────────┐
│  COLLSCAN   │  ← Charge tous les documents
└─────┬───────┘
      │
      ▼
┌─────────────┐
│    SORT     │  ← Tri en mémoire (coûteux)
└─────┬───────┘
      │
      ▼
┌─────────────┐
│   LIMIT     │  ← Limite les résultats
└─────┬───────┘
      │
      ▼
   Résultats


Requête avec tri et index :
┌─────────────┐
│   IXSCAN    │  ← Parcourt l'index (déjà trié)
└─────┬───────┘
      │
      ▼
┌─────────────┐
│    FETCH    │  ← Récupère les documents
└─────┬───────┘
      │
      ▼
┌─────────────┐
│   LIMIT     │  ← Limite les résultats
└─────┬───────┘
      │
      ▼
   Résultats (Pas de SORT !)
```

### Descriptions des stages

#### COLLSCAN - Collection Scan

```
❌ Scan complet de la collection

Signification :
├─ MongoDB lit TOUS les documents un par un
├─ Aucun index n'est utilisé
├─ Très lent sur grandes collections
└─ À éviter en production

Quand c'est acceptable :
├─ Collection < 1000 documents
├─ Requête retourne la majorité des documents
└─ Pas d'index disponible et création impossible

Action :
└─ Créer un index approprié
```

#### IXSCAN - Index Scan

```
✅ Utilise un index

Signification :
├─ MongoDB parcourt l'index
├─ Localise rapidement les documents pertinents
├─ Rapide et efficace
└─ C'est ce qu'on veut voir !

Détails utiles :
├─ indexName : Nom de l'index utilisé
├─ keyPattern : Structure de l'index
├─ direction : forward (croissant) ou backward (décroissant)
└─ indexBounds : Plage de valeurs parcourues
```

#### FETCH

```
✅ Récupération des documents complets

Signification :
├─ Après avoir trouvé les _id via l'index
├─ MongoDB récupère les documents complets
├─ Normal après un IXSCAN
└─ Nécessaire si la requête demande tous les champs

Quand il peut être évité :
└─ Covered Query (projection sur champs indexés uniquement)
```

#### SORT

```
⚠️  Tri en mémoire

Signification :
├─ MongoDB charge les documents en RAM
├─ Les trie selon le critère demandé
├─ Coûteux en temps et mémoire
└─ Limite : 100 Mo par défaut

Action :
└─ Créer un index sur le champ de tri
```

#### COUNT

```
✅ Comptage de documents

Signification :
├─ Compte le nombre de documents
├─ Peut utiliser un index
└─ Plus rapide si index approprié existe

Variantes :
├─ COUNT_SCAN : Utilise un index
└─ COLLSCAN : Compte en parcourant tout
```

---

## Covered Queries (requêtes couvertes)

### Concept

Une **covered query** est une requête dont tous les champs demandés sont dans l'index. MongoDB n'a donc pas besoin de récupérer les documents complets (pas de FETCH).

### Exemple

```javascript
// Index créé
db.users.createIndex({ email: 1, name: 1 })

// Requête couverte (tous les champs sont dans l'index)
db.users.find(
  { email: "alice@example.com" },
  { _id: 0, email: 1, name: 1 }    // Projection sur champs indexés
).explain("executionStats")
```

**Résultat** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "PROJECTION_COVERED",    // ✅ Covered query !
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "email_1_name_1"
      }
    }
  },
  "executionStats": {
    "nReturned": 1,
    "totalDocsExamined": 0,             // ✅ 0 documents récupérés !
    "totalKeysExamined": 1              // Seulement l'index
  }
}
```

**Avantages** :
```
✅ Pas de FETCH (pas de lecture disque)
✅ Encore plus rapide
✅ Moins d'I/O
✅ totalDocsExamined = 0
```

**Conditions pour une covered query** :
```
1. Tous les champs retournés doivent être dans l'index
2. Le filtre doit utiliser l'index
3. La projection doit exclure _id (sauf si _id est dans l'index)
4. Aucun champ tableau dans l'index
```

---

## Analyser les performances : Méthodologie

### Processus en 5 étapes

```
1. IDENTIFIER la requête lente
   └─ Logs, monitoring, retours utilisateurs

2. EXÉCUTER avec explain("executionStats")
   └─ Observer les métriques clés

3. DIAGNOSTIQUER le problème
   └─ COLLSCAN ? Mauvais ratio ? Tri en mémoire ?

4. OPTIMISER
   └─ Créer/modifier un index

5. VALIDER l'amélioration
   └─ Ré-exécuter explain() et comparer
```

### Exemple complet d'analyse

#### Étape 1 : Requête lente identifiée

```javascript
// Requête utilisée dans l'application
db.orders.find({
  userId: 12345,
  status: "pending",
  createdAt: { $gte: new Date("2024-01-01") }
}).sort({ createdAt: -1 })
```

#### Étape 2 : Exécuter explain()

```javascript
db.orders.find({
  userId: 12345,
  status: "pending",
  createdAt: { $gte: new Date("2024-01-01") }
}).sort({ createdAt: -1 })
  .explain("executionStats")
```

**Résultat** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "SORT",                  // ⚠️ Tri en mémoire
      "inputStage": {
        "stage": "FETCH",
        "inputStage": {
          "stage": "IXSCAN",
          "indexName": "userId_1"
        }
      }
    }
  },
  "executionStats": {
    "nReturned": 50,
    "executionTimeMillis": 234,         // 234ms - Lent
    "totalKeysExamined": 2500,
    "totalDocsExamined": 2500
  }
}
```

#### Étape 3 : Diagnostic

```
Analyse :
─────────
✅ Index utilisé : userId_1
⚠️  Problème 1 : Ratio = 50/2500 = 2% (inefficace)
⚠️  Problème 2 : SORT en mémoire (coûteux)
⚠️  Cause : L'index ne couvre pas les autres filtres

Problèmes identifiés :
1. Index simple sur userId seulement
2. Filtres sur status et createdAt appliqués après
3. Tri sur createdAt nécessite chargement en mémoire
```

#### Étape 4 : Optimisation

```javascript
// Créer un index composé optimal
// Ordre selon règle ESR (Equality, Sort, Range)
db.orders.createIndex({
  userId: 1,        // E - Equality (filtre exact)
  status: 1,        // E - Equality (filtre exact)
  createdAt: -1     // S/R - Sort ET Range
})
```

#### Étape 5 : Validation

```javascript
// Ré-exécuter la même requête
db.orders.find({
  userId: 12345,
  status: "pending",
  createdAt: { $gte: new Date("2024-01-01") }
}).sort({ createdAt: -1 })
  .explain("executionStats")
```

**Nouveau résultat** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",                 // Pas de SORT !
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "userId_1_status_1_createdAt_-1"
      }
    }
  },
  "executionStats": {
    "nReturned": 50,
    "executionTimeMillis": 8,           // 8ms au lieu de 234ms !
    "totalKeysExamined": 50,            // 50 au lieu de 2500
    "totalDocsExamined": 50             // 50 au lieu de 2500
  }
}
```

**Résumé de l'amélioration** :

```
Avant optimisation :
├─ Temps : 234ms
├─ Ratio : 50/2500 = 2%
├─ Index : userId_1 (simple)
└─ Tri : En mémoire (SORT)

Après optimisation :
├─ Temps : 8ms  ✅ (29x plus rapide)
├─ Ratio : 50/50 = 100%  ✅ (parfait)
├─ Index : userId_1_status_1_createdAt_-1 (composé)
└─ Tri : Via index (pas de SORT)  ✅

Amélioration globale : 234ms → 8ms = 2900% plus rapide ! 🚀
```

---

## Comprendre le Query Planner

### Comment MongoDB choisit un index ?

Quand plusieurs index sont disponibles, MongoDB utilise le **query planner** pour choisir le meilleur :

```
Processus de sélection d'index
═══════════════════════════════

Étape 1 : IDENTIFICATION
├─ MongoDB identifie les index candidats
└─ Index qui peuvent répondre à la requête

Étape 2 : COMPÉTITION
├─ Teste plusieurs plans en parallèle
├─ Exécute partiellement chaque plan
└─ Mesure les performances

Étape 3 : SÉLECTION
├─ Choisit le plan le plus rapide
└─ Ce plan devient le "winning plan"

Étape 4 : CACHE
├─ Le plan gagnant est mis en cache
└─ Réutilisé pour requêtes similaires
```

### Voir les plans rejetés

```javascript
db.users.find({ city: "Paris", age: 30 })
  .explain("allPlansExecution")
```

**Résultat (simplifié)** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "city_1_age_1"     // ✅ Plan gagnant
      }
    },
    "rejectedPlans": [                  // Plans testés et rejetés
      {
        "stage": "FETCH",
        "inputStage": {
          "stage": "IXSCAN",
          "indexName": "city_1"         // ❌ Moins bon
        }
      },
      {
        "stage": "FETCH",
        "inputStage": {
          "stage": "IXSCAN",
          "indexName": "age_1"          // ❌ Moins bon
        }
      }
    ]
  },
  "allPlansExecution": [
    // Détails de tous les plans testés...
  ]
}
```

---

## Conseils pratiques pour utiliser explain()

### 1. Toujours utiliser executionStats

```javascript
// ❌ Pas assez d'information
db.collection.find({ ... }).explain()

// ✅ Recommandé
db.collection.find({ ... }).explain("executionStats")
```

### 2. Comparer avant/après

```javascript
// Avant création d'index
const before = db.collection.find({ ... }).explain("executionStats")
print(`Avant : ${before.executionStats.executionTimeMillis}ms`)

// Créer l'index
db.collection.createIndex({ ... })

// Après création d'index
const after = db.collection.find({ ... }).explain("executionStats")
print(`Après : ${after.executionStats.executionTimeMillis}ms`)
print(`Amélioration : ${before.executionStats.executionTimeMillis / after.executionStats.executionTimeMillis}x`)
```

### 3. Surveiller les métriques clés

```javascript
function analyzeQuery(explainResult) {
  const stats = explainResult.executionStats

  console.log("=== Analyse de la requête ===")
  console.log(`Stage : ${explainResult.queryPlanner.winningPlan.stage}`)
  console.log(`Temps : ${stats.executionTimeMillis}ms`)
  console.log(`Retournés : ${stats.nReturned}`)
  console.log(`Examinés : ${stats.totalDocsExamined}`)
  console.log(`Ratio : ${(stats.nReturned / stats.totalDocsExamined * 100).toFixed(2)}%`)

  // Évaluation
  const ratio = stats.nReturned / stats.totalDocsExamined
  if (ratio === 1) {
    console.log("✅ PARFAIT - Efficacité optimale")
  } else if (ratio > 0.5) {
    console.log("✅ BON - Efficacité acceptable")
  } else if (ratio > 0.1) {
    console.log("⚠️  MOYEN - Peut être optimisé")
  } else {
    console.log("❌ MAUVAIS - Optimisation nécessaire")
  }
}

// Usage
const result = db.users.find({ city: "Paris" }).explain("executionStats")
analyzeQuery(result)
```

### 4. Tester avec des données réalistes

```javascript
// ❌ Mauvais : Tester sur collection vide ou très petite
db.testCollection.find({ ... }).explain("executionStats")
// Les résultats ne sont pas représentatifs

// ✅ Bon : Tester sur données de volume similaire à la production
// Utiliser un environnement de staging avec volume réaliste
```

### 5. Utiliser explain() sur toutes les opérations

```javascript
// find()
db.collection.find({ ... }).explain("executionStats")

// update()
db.collection.explain("executionStats").update({ ... }, { ... })

// delete()
db.collection.explain("executionStats").deleteMany({ ... })

// aggregate()
db.collection.explain("executionStats").aggregate([...])

// count()
db.collection.explain("executionStats").count({ ... })
```

---

## Checklist d'analyse avec explain()

### ✅ Checklist : Analyser une requête

```
□ J'ai exécuté explain("executionStats") sur la requête
□ J'ai vérifié le stage principal (IXSCAN vs COLLSCAN)
□ J'ai calculé le ratio (nReturned / totalDocsExamined)
□ J'ai mesuré le temps d'exécution (executionTimeMillis)
□ J'ai identifié l'index utilisé (ou son absence)
□ J'ai vérifié s'il y a un tri en mémoire (SORT)
□ J'ai comparé totalKeysExamined et nReturned
□ J'ai documenté les résultats pour référence future
```

### ✅ Checklist : Optimisation nécessaire ?

```
Optimisation URGENTE si :
□ Stage = COLLSCAN sur collection > 10 000 docs
□ executionTimeMillis > 1000ms (1 seconde)
□ Ratio < 0.01 (< 1%)
□ Stage = SORT avec memLimit atteinte

Optimisation RECOMMANDÉE si :
□ Stage = COLLSCAN sur collection > 1 000 docs
□ executionTimeMillis > 100ms
□ Ratio < 0.1 (< 10%)
□ totalDocsExamined >> nReturned (beaucoup de gaspillage)

Optimisation POSSIBLE si :
□ Ratio entre 0.1 et 0.5 (10-50%)
□ executionTimeMillis > 50ms
□ Index utilisé mais pas optimal
```

---

## Erreurs courantes d'interprétation

### Erreur 1 : Confondre queryPlanner et executionStats

```javascript
// ❌ Mauvais
const result = db.collection.find({ ... }).explain()
// Regarde executionTimeMillis
// → N'existe pas en mode queryPlanner !

// ✅ Correct
const result = db.collection.find({ ... }).explain("executionStats")
// Maintenant executionTimeMillis est disponible
```

### Erreur 2 : Ignorer le ratio d'efficacité

```javascript
// Résultat
{
  "executionStats": {
    "nReturned": 10,
    "executionTimeMillis": 50,          // 50ms semble "OK"
    "totalDocsExamined": 100000         // Mais 100K docs examinés !
  }
}

// ❌ Se dire : "50ms c'est acceptable"
// ✅ Calculer : 10/100000 = 0.01% → Terrible !
```

### Erreur 3 : Tester sur petites collections

```javascript
// Collection de 100 documents
db.smallCollection.find({ ... }).explain("executionStats")
// executionTimeMillis: 2ms

// ❌ Conclure : "Pas besoin d'index, c'est rapide"
// ✅ Réaliser : Sur 10M documents, ce sera 200x plus lent !
```

### Erreur 4 : Ne pas vérifier après création d'index

```javascript
// Créer l'index
db.collection.createIndex({ field: 1 })

// ❌ Supposer que l'index sera utilisé
// ✅ TOUJOURS vérifier avec explain()
db.collection.find({ field: "value" }).explain("executionStats")
```

---

## Concepts clés à retenir

### 🎯 Points essentiels

1. **explain()** montre comment MongoDB exécute une requête

2. **Trois modes** :
   - queryPlanner : Plan seulement
   - **executionStats** : ⭐ Mode recommandé (plan + stats)
   - allPlansExecution : Tous les plans testés

3. **Métriques cruciales** :
   - **Stage** : IXSCAN (✅) vs COLLSCAN (❌)
   - **nReturned** : Documents retournés
   - **totalDocsExamined** : Documents examinés
   - **Ratio** : nReturned / totalDocsExamined (viser 100%)
   - **executionTimeMillis** : Temps d'exécution

4. **Objectifs d'optimisation** :
   - Stage = IXSCAN
   - Ratio proche de 100%
   - Pas de SORT en mémoire
   - executionTimeMillis < 100ms

5. **Processus d'analyse** :
   - Exécuter explain("executionStats")
   - Identifier le problème
   - Créer/modifier l'index
   - Valider avec explain()

6. **Toujours vérifier** après création d'index !

---

## Ressources et commandes utiles

### Commandes explain() essentielles

```javascript
// Requête basique
db.collection.find({ field: value }).explain("executionStats")

// Avec tri
db.collection.find({ ... }).sort({ field: 1 }).explain("executionStats")

// Avec limite
db.collection.find({ ... }).limit(10).explain("executionStats")

// Update
db.collection.explain("executionStats").update({ ... }, { $set: { ... } })

// Delete
db.collection.explain("executionStats").deleteMany({ ... })

// Aggregate
db.collection.explain("executionStats").aggregate([{ $match: { ... } }])
```

### Script d'analyse rapide

```javascript
function quickAnalysis(collection, query) {
  const result = db[collection].find(query).explain("executionStats")
  const stats = result.executionStats
  const plan = result.queryPlanner.winningPlan

  console.log("\n========== ANALYSE RAPIDE ==========")
  console.log(`Collection : ${collection}`)
  console.log(`Stage : ${plan.stage}`)

  if (plan.inputStage && plan.inputStage.stage === "IXSCAN") {
    console.log(`✅ Index utilisé : ${plan.inputStage.indexName}`)
  } else {
    console.log(`❌ Pas d'index ou COLLSCAN`)
  }

  console.log(`Retournés : ${stats.nReturned}`)
  console.log(`Examinés : ${stats.totalDocsExamined}`)
  console.log(`Temps : ${stats.executionTimeMillis}ms`)

  const ratio = stats.totalDocsExamined > 0
    ? (stats.nReturned / stats.totalDocsExamined * 100).toFixed(2)
    : 0
  console.log(`Ratio : ${ratio}%`)

  if (ratio >= 80) {
    console.log("🌟 EXCELLENT")
  } else if (ratio >= 50) {
    console.log("✅ BON")
  } else if (ratio >= 10) {
    console.log("⚠️  MOYEN - À optimiser")
  } else {
    console.log("❌ MAUVAIS - Optimisation urgente")
  }
  console.log("===================================\n")
}

// Usage
quickAnalysis("users", { city: "Paris" })
```

---

## Analogie finale

> **explain() est comme le diagnostic médical d'une voiture :**
>
> **Sans explain()** = "Ma voiture est lente"
> → Vous savez qu'il y a un problème, mais pas pourquoi
>
> **Avec explain()** = Diagnostic complet :
> - Moteur OK ✅ (Stage: IXSCAN)
> - Filtre à air encrassé ⚠️ (Ratio: 20%)
> - Temps 0-100 km/h : 15s ❌ (executionTimeMillis)
> - Carburant gaspillé : 80% ❌ (totalDocsExamined >> nReturned)
>
> → Vous savez exactement quoi réparer (créer un index)
>
> **explain() transforme l'intuition en certitude et la supposition en optimisation mesurable !** 🔧

---

**Vous maîtrisez maintenant l'analyse des requêtes avec explain() !** 🚀

---


⏭️ [Le Query Planner](/05-index-et-optimisation/07-query-planner.md)
