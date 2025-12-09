🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Index et Optimisation

## La clé des performances MongoDB ! ⚡

Vous savez modéliser vos données, écrire des requêtes complexes, et structurer vos documents efficacement. Excellent ! Mais voici la réalité : même le meilleur schéma du monde sera **lent** sans les bons index. Un index bien placé peut transformer une requête de 30 secondes en 10 millisecondes. Ce chapitre va vous apprendre à débloquer les véritables performances de MongoDB.

Les index sont **la différence entre une application lente et une application rapide**. C'est aussi simple que cela. Et la bonne nouvelle ? MongoDB offre des outils puissants pour analyser, optimiser et monitorer vos requêtes.

## Où en sommes-nous dans votre parcours ?

Vous avez complété les chapitres 1 à 4 et vous maîtrisez maintenant :
- ✅ Les fondamentaux de MongoDB (CRUD, types de données)
- ✅ Les requêtes complexes avec tous les opérateurs
- ✅ La modélisation des données (imbrication vs références, relations, patterns)
- ✅ Les structures de documents optimisées

**Parfait !** Vous êtes maintenant prêt à apprendre comment **rendre tout cela performant à grande échelle**.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- ✅ **Comprendre** comment fonctionnent les index en interne
- ✅ **Créer** tous les types d'index (simple, composé, multikey, texte, géospatial, etc.)
- ✅ **Analyser** les performances avec `explain()` et interpréter les résultats
- ✅ **Identifier** les requêtes lentes et les goulots d'étranglement
- ✅ **Optimiser** vos requêtes pour des performances maximales
- ✅ **Choisir** les bons index selon vos cas d'usage
- ✅ **Éviter** les pièges d'indexation courants
- ✅ **Monitorer** les performances en production
- ✅ **Gérer** les index dans un environnement de production

## Le problème de performance : un exemple concret

### Scénario : E-commerce avec 1 million de produits

Imaginons une collection `products` avec 1 million de documents :

```javascript
// Structure d'un produit
{
    _id: ObjectId("..."),
    name: "Ordinateur portable Dell XPS 15",
    category: "Informatique",
    brand: "Dell",
    price: 1299.99,
    stock: 15,
    rating: 4.5,
    tags: ["laptop", "dell", "gaming"],
    specs: {
        processor: "Intel i7",
        ram: 16,
        storage: 512
    },
    createdAt: ISODate("2024-01-15")
}
```

### Requête 1 : Recherche sans index

```javascript
// Chercher les produits Dell entre 1000€ et 1500€
db.products.find({
    brand: "Dell",
    price: { $gte: 1000, $lte: 1500 }
})
```

**Sans index, MongoDB doit scanner TOUS les documents :**

```javascript
db.products.find({
    brand: "Dell",
    price: { $gte: 1000, $lte: 1500 }
}).explain("executionStats")

// Résultat (simplifié)
{
    executionStats: {
        executionTimeMillis: 28453,        // 28 secondes ! 😱
        totalDocsExamined: 1000000,        // Tous les documents
        totalKeysExamined: 0,              // Aucun index utilisé
        nReturned: 150,                    // Seulement 150 résultats
        stage: "COLLSCAN"                  // Collection Scan (mauvais!)
    }
}
```

**Analyse :**
- ⏱️ **28 secondes** pour retourner 150 produits
- 🔍 **1 million de documents examinés** pour trouver 150 résultats
- 📊 **Ratio : 1,000,000 / 150 = 6,667 documents examinés par résultat !**
- ❌ **COLLSCAN** = Scan complet de la collection = Catastrophique

### Requête 2 : Même requête AVEC index

```javascript
// Créer un index composé sur brand et price
db.products.createIndex({ brand: 1, price: 1 })

// Même requête
db.products.find({
    brand: "Dell",
    price: { $gte: 1000, $lte: 1500 }
}).explain("executionStats")

// Résultat avec index
{
    executionStats: {
        executionTimeMillis: 12,           // 12 millisecondes ! 🚀
        totalDocsExamined: 150,            // Seulement les documents pertinents
        totalKeysExamined: 150,            // Utilisation de l'index
        nReturned: 150,
        stage: "IXSCAN",                   // Index Scan (excellent!)
        indexName: "brand_1_price_1"
    }
}
```

**Analyse :**
- ⚡ **12 ms** au lieu de 28 secondes → **2,371x plus rapide !**
- 🎯 **150 documents examinés** au lieu de 1,000,000 → **6,667x moins de travail**
- ✅ **IXSCAN** = Utilisation de l'index = Optimal
- 📈 **Ratio : 150 / 150 = 1 document examiné par résultat = Parfait !**

**Conclusion : Un simple index a divisé le temps d'exécution par 2,371 !**

## Vue d'ensemble du chapitre

Ce chapitre est organisé en 11 sections qui couvrent tous les aspects de l'indexation et de l'optimisation :

### 🎯 Partie 1 : Fondamentaux (Sections 5.1 à 5.2)
- **5.1** : Comprendre l'importance des index
- **5.2** : Types d'index fondamentaux (simple, composé, multikey)

### 🎯 Partie 2 : Index spécialisés (Section 5.3)
Les index pour cas d'usage spécifiques :
- Texte (recherche full-text)
- Géospatial (coordonnées, proximité)
- Haché (distribution uniforme pour sharding)
- Wildcard (schémas flexibles)
- TTL (expiration automatique)

### 🎯 Partie 3 : Options avancées (Section 5.4)
Modificateurs qui affinent le comportement des index :
- Unique, Partial, Sparse, Hidden
- Combinaisons d'options

### 🎯 Partie 4 : Analyse et optimisation (Sections 5.5 à 5.10)
- **5.5** : Création et suppression d'index
- **5.6** : Analyse avec `explain()`
- **5.7** : Le Query Planner
- **5.8** : Stratégies d'optimisation
- **5.9** : Index couvrants (Covered Queries)
- **5.10** : Gestion en production

### 🎯 Partie 5 : Monitoring (Section 5.11)
Outils et techniques pour surveiller les performances

## Comprendre explain() : votre meilleur ami

`explain()` est l'outil principal pour analyser les performances de vos requêtes. Il existe trois modes :

### 1. Mode "queryPlanner" (par défaut)

```javascript
db.products.find({ brand: "Dell" }).explain()
// ou
db.products.find({ brand: "Dell" }).explain("queryPlanner")
```

**Retourne :** Le plan d'exécution choisi par MongoDB (quel index, quelle stratégie).

**Utilisation :** Vérifier rapidement quel index est utilisé.

### 2. Mode "executionStats"

```javascript
db.products.find({ brand: "Dell" }).explain("executionStats")
```

**Retourne :** Statistiques d'exécution réelles (temps, documents examinés, etc.).

**Utilisation :** Analyser les performances réelles et identifier les problèmes.

### 3. Mode "allPlansExecution"

```javascript
db.products.find({ brand: "Dell" }).explain("allPlansExecution")
```

**Retourne :** Tous les plans évalués par le Query Planner.

**Utilisation :** Déboguer les choix d'index non optimaux.

## Anatomie d'un résultat explain()

Décortiquons un résultat `explain("executionStats")` complet :

```javascript
db.products.find({
    category: "Informatique",
    price: { $lte: 500 }
}).explain("executionStats")
```

### Résultat détaillé avec annotations

```javascript
{
    // 1. Informations sur le plan choisi
    "queryPlanner": {
        "plannerVersion": 1,
        "namespace": "ecommerce.products",
        "indexFilterSet": false,

        // Plan d'exécution gagnant
        "winningPlan": {
            "stage": "FETCH",              // Étape finale : récupérer documents
            "inputStage": {
                "stage": "IXSCAN",         // ✅ Index Scan (bon signe!)
                "keyPattern": {            // Index utilisé
                    "category": 1,
                    "price": 1
                },
                "indexName": "category_1_price_1",
                "isMultiKey": false,
                "direction": "forward",
                "indexBounds": {           // Limites de scan dans l'index
                    "category": ["Informatique", "Informatique"],
                    "price": ["-Infinity", 500.0]
                }
            }
        },

        // Plans rejetés (si plusieurs index disponibles)
        "rejectedPlans": [
            // ... autres plans testés
        ]
    },

    // 2. Statistiques d'exécution (le plus important!)
    "executionStats": {
        "executionSuccess": true,
        "nReturned": 847,                  // Nombre de documents retournés
        "executionTimeMillis": 15,         // ⏱️ Temps total (15ms = bon)
        "totalKeysExamined": 847,          // 🔑 Clés d'index examinées
        "totalDocsExamined": 847,          // 📄 Documents examinés

        // Détails par étape
        "executionStages": {
            "stage": "FETCH",
            "nReturned": 847,
            "executionTimeMillisEstimate": 10,
            "works": 848,
            "advanced": 847,
            "docsExamined": 847,           // Documents lus depuis le disque

            "inputStage": {
                "stage": "IXSCAN",
                "nReturned": 847,
                "executionTimeMillisEstimate": 5,
                "works": 848,
                "advanced": 847,
                "keyPattern": {
                    "category": 1,
                    "price": 1
                },
                "indexName": "category_1_price_1",
                "keysExamined": 847        // Entrées d'index lues
            }
        }
    },

    "serverInfo": { /* ... */ }
}
```

## Les métriques clés à surveiller

### 1. executionTimeMillis

```javascript
"executionTimeMillis": 15
```

**Signification :** Temps total d'exécution de la requête.

**Objectif :**
- ✅ < 100ms : Excellent
- ⚠️ 100-1000ms : Acceptable pour requêtes complexes
- ❌ > 1000ms : Problème à investiguer

### 2. totalDocsExamined vs nReturned

```javascript
"totalDocsExamined": 847,
"nReturned": 847
```

**Signification :** Combien de documents ont été examinés vs retournés.

**Ratio optimal :** `totalDocsExamined / nReturned` proche de 1

**Exemples :**
```javascript
// ✅ EXCELLENT : Ratio 1:1
totalDocsExamined: 100, nReturned: 100  // Ratio = 1.0

// ⚠️ ACCEPTABLE : Ratio 2:1
totalDocsExamined: 200, nReturned: 100  // Ratio = 2.0

// ❌ PROBLÈME : Ratio 100:1
totalDocsExamined: 10000, nReturned: 100  // Ratio = 100.0
// → 99% des documents examinés sont inutiles !
```

### 3. stage (Type de scan)

```javascript
"stage": "IXSCAN"  // ou "COLLSCAN", "FETCH", etc.
```

**Types principaux :**

| Stage | Signification | Performance |
|-------|---------------|-------------|
| `IXSCAN` | Index Scan | ✅ Excellent |
| `FETCH` | Récupération de documents | ✅ Normal après IXSCAN |
| `COLLSCAN` | Collection Scan (scan complet) | ❌ Problème |
| `SORT` | Tri en mémoire | ⚠️ Coûteux si gros volume |
| `TEXT` | Recherche full-text | ✅ Bon pour texte |
| `GEO_NEAR` | Recherche géospatiale | ✅ Bon pour géo |

**Règle d'or :** Si vous voyez `COLLSCAN`, vous avez probablement besoin d'un index !

### 4. totalKeysExamined

```javascript
"totalKeysExamined": 847
```

**Signification :** Nombre d'entrées d'index lues.

**Objectif :** Proche de `nReturned` (idéalement égal).

### 5. executionStages (Pipeline d'exécution)

```javascript
"executionStages": {
    "stage": "FETCH",           // Étape 3 : Récupérer documents complets
    "inputStage": {
        "stage": "IXSCAN",      // Étape 2 : Scanner l'index
        "inputStage": {
            "stage": "IXSCAN"   // Étape 1 : Scanner autre index (si intersection)
        }
    }
}
```

**Signification :** Pipeline d'étapes depuis la source jusqu'au résultat final.

**Lecture :** De l'intérieur vers l'extérieur (bottom-up).

## Exemples progressifs de performance

### Exemple 1 : Collection Scan vs Index Scan

```javascript
// Collection de 100,000 utilisateurs
db.users.insertMany([
    { _id: 1, username: "alice", age: 28, city: "Paris", active: true },
    { _id: 2, username: "bob", age: 35, city: "Lyon", active: true },
    // ... 99,998 autres utilisateurs
])
```

#### Sans index (Collection Scan)

```javascript
db.users.find({ city: "Paris" }).explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 142,         // 142ms
        totalDocsExamined: 100000,        // Scan complet
        totalKeysExamined: 0,             // Aucun index
        nReturned: 8500,                  // 8500 parisiens
        stage: "COLLSCAN"                 // ❌ Collection Scan
    }
}

// Ratio : 100,000 / 8,500 = 11.76
// → Pour chaque résultat, 11.76 documents sont examinés
```

#### Avec index (Index Scan)

```javascript
db.users.createIndex({ city: 1 })

db.users.find({ city: "Paris" }).explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 18,          // 18ms (8x plus rapide!)
        totalDocsExamined: 8500,          // Seulement les Parisiens
        totalKeysExamined: 8500,          // Entrées d'index utilisées
        nReturned: 8500,
        stage: "FETCH",
        inputStage: {
            stage: "IXSCAN",              // ✅ Index Scan
            indexName: "city_1"
        }
    }
}

// Ratio : 8,500 / 8,500 = 1.0 (parfait!)
```

**Gain : 142ms → 18ms = 7.9x plus rapide**

### Exemple 2 : Index simple vs Index composé

```javascript
// Requête avec deux critères
db.users.find({
    city: "Paris",
    age: { $gte: 25, $lte: 35 }
})
```

#### Option A : Index simple sur city

```javascript
db.users.createIndex({ city: 1 })

db.users.find({
    city: "Paris",
    age: { $gte: 25, $lte: 35 }
}).explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 25,
        totalDocsExamined: 8500,          // Tous les Parisiens
        totalKeysExamined: 8500,
        nReturned: 2100,                  // Seulement 2100 dans tranche d'âge
        stage: "FETCH"
    }
}

// Ratio : 8,500 / 2,100 = 4.05
// → Beaucoup de documents inutiles examinés
```

#### Option B : Index composé sur city + age

```javascript
db.users.createIndex({ city: 1, age: 1 })

db.users.find({
    city: "Paris",
    age: { $gte: 25, $lte: 35 }
}).explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 8,           // 3x plus rapide!
        totalDocsExamined: 2100,          // Seulement les correspondances
        totalKeysExamined: 2100,
        nReturned: 2100,
        stage: "FETCH",
        inputStage: {
            stage: "IXSCAN",
            indexName: "city_1_age_1"     // Index composé utilisé
        }
    }
}

// Ratio : 2,100 / 2,100 = 1.0 (parfait!)
```

**Gain : 25ms → 8ms = 3.1x plus rapide avec index composé**

### Exemple 3 : Tri sans et avec index

```javascript
// Récupérer les 10 utilisateurs les plus âgés de Paris
db.users.find({ city: "Paris" }).sort({ age: -1 }).limit(10)
```

#### Sans index sur age (tri en mémoire)

```javascript
db.users.createIndex({ city: 1 })  // Seulement sur city

db.users.find({ city: "Paris" })
    .sort({ age: -1 })
    .limit(10)
    .explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 35,
        totalDocsExamined: 8500,
        nReturned: 10,
        stage: "LIMIT",
        inputStage: {
            stage: "SORT",                // ⚠️ Tri en mémoire (coûteux)
            sortPattern: { age: -1 },
            memUsage: 425000,             // Utilise la mémoire
            inputStage: {
                stage: "FETCH",
                inputStage: {
                    stage: "IXSCAN",
                    indexName: "city_1"
                }
            }
        }
    }
}

// Doit récupérer tous les Parisiens, les trier, puis prendre les 10 premiers
```

#### Avec index composé city + age

```javascript
db.users.createIndex({ city: 1, age: -1 })  // age en ordre décroissant

db.users.find({ city: "Paris" })
    .sort({ age: -1 })
    .limit(10)
    .explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 2,           // 17x plus rapide!
        totalDocsExamined: 10,            // Seulement 10 documents
        totalKeysExamined: 10,
        nReturned: 10,
        stage: "LIMIT",
        inputStage: {
            stage: "FETCH",
            inputStage: {
                stage: "IXSCAN",          // Pas de SORT!
                indexName: "city_1_age_-1"
            }
        }
    }
}

// L'index est déjà trié → récupère directement les 10 premiers
```

**Gain : 35ms → 2ms = 17.5x plus rapide + économie de mémoire**

### Exemple 4 : Covered Query (requête couverte)

Une **Covered Query** est une requête dont tous les champs (filtre + projection) sont dans l'index.

```javascript
// Requête normale
db.users.find(
    { city: "Paris" },
    { city: 1, age: 1, _id: 0 }  // Projection : seulement city et age
)
```

#### Sans covered query

```javascript
db.users.createIndex({ city: 1 })

db.users.find(
    { city: "Paris" },
    { city: 1, age: 1, _id: 0 }
).explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 18,
        totalDocsExamined: 8500,          // Doit lire les documents
        totalKeysExamined: 8500,
        nReturned: 8500,
        stage: "PROJECTION_COVERED",
        inputStage: {
            stage: "FETCH",               // Récupère les documents
            inputStage: {
                stage: "IXSCAN"
            }
        }
    }
}
```

#### Avec covered query

```javascript
db.users.createIndex({ city: 1, age: 1 })  // Index contient city ET age

db.users.find(
    { city: "Paris" },
    { city: 1, age: 1, _id: 0 }  // Tous les champs sont dans l'index
).explain("executionStats")

{
    executionStats: {
        executionTimeMillis: 5,           // 3.6x plus rapide!
        totalDocsExamined: 0,             // ✅ Aucun document lu!
        totalKeysExamined: 8500,
        nReturned: 8500,
        stage: "PROJECTION_COVERED",
        inputStage: {
            stage: "IXSCAN",
            indexName: "city_1_age_1",
            isMultiKey: false,
            indexOnly: true               // ✅ Index-only query!
        }
    }
}

// Données lues uniquement depuis l'index → ultra rapide
```

**Gain : 18ms → 5ms = 3.6x plus rapide + aucun accès disque**

## Les types d'index : aperçu

MongoDB offre plusieurs types d'index adaptés à différents cas d'usage :

### 1. Index simple (Single Field)

```javascript
db.users.createIndex({ username: 1 })  // 1 = ordre croissant
```

**Usage :** Requêtes sur un seul champ.

**Performance :** O(log n) pour la recherche.

### 2. Index composé (Compound)

```javascript
db.users.createIndex({ city: 1, age: -1, username: 1 })
```

**Usage :** Requêtes sur plusieurs champs ensemble.

**Règle ESR :** Equality (=) → Sort (tri) → Range (plages)

### 3. Index multikey (sur tableaux)

```javascript
// Automatique si champ est un tableau
db.products.createIndex({ tags: 1 })

// Document
{ tags: ["laptop", "dell", "gaming"] }

// Index créé pour chaque élément du tableau
```

**Usage :** Recherche dans les tableaux.

### 4. Index texte (Text)

```javascript
db.articles.createIndex({ content: "text" })
```

**Usage :** Recherche full-text, mots-clés.

**Performance :** Optimisé pour la recherche textuelle.

### 5. Index géospatial (2dsphere)

```javascript
db.locations.createIndex({ coordinates: "2dsphere" })
```

**Usage :** Requêtes de proximité, recherche géographique.

### 6. Index haché (Hashed)

```javascript
db.users.createIndex({ email: "hashed" })
```

**Usage :** Distribution uniforme pour le sharding.

### 7. Index TTL (Time-To-Live)

```javascript
db.sessions.createIndex(
    { createdAt: 1 },
    { expireAfterSeconds: 3600 }  // Expire après 1h
)
```

**Usage :** Expiration automatique de documents.

### 8. Index Wildcard

```javascript
db.products.createIndex({ "$**": 1 })
```

**Usage :** Schémas flexibles avec champs variables.

## Index et coût de maintenance

### Impact sur les écritures

Les index améliorent les lectures mais **ralentissent les écritures** :

```javascript
// Collection SANS index
db.products.insertOne({ name: "Produit A", price: 99.99 })
// Temps : ~1ms

// Collection AVEC 5 index
db.products.insertOne({ name: "Produit A", price: 99.99 })
// Temps : ~3-5ms (chaque index doit être mis à jour)
```

**Règle :** N'indexez que ce qui est réellement utilisé dans vos requêtes fréquentes.

### Espace disque

Les index consomment de l'espace :

```javascript
db.users.stats()

{
    count: 100000,
    size: 15000000,              // 15 MB de données
    indexSizes: {
        "_id_": 2000000,         // 2 MB
        "username_1": 1500000,   // 1.5 MB
        "city_1": 1200000,       // 1.2 MB
        "city_1_age_1": 1800000  // 1.8 MB
    },
    totalIndexSize: 6500000      // 6.5 MB d'index (43% des données)
}
```

**Considération :** Les index prennent typiquement 10-50% de la taille des données.

## Stratégies d'optimisation : avant-goût

### Stratégie 1 : ESR (Equality, Sort, Range)

Ordre optimal des champs dans un index composé :

```javascript
// Requête
db.products.find({
    category: "Informatique",      // Equality
    brand: "Dell"                  // Equality
}).sort({ price: 1 })             // Sort

// Index optimal (ordre ESR)
db.products.createIndex({
    category: 1,    // E : Equality
    brand: 1,       // E : Equality
    price: 1        // S : Sort
})

// ❌ Mauvais ordre
db.products.createIndex({ price: 1, category: 1, brand: 1 })
```

### Stratégie 2 : Sélectivité

Indexer les champs les plus sélectifs en premier :

```javascript
// Champ très sélectif (beaucoup de valeurs uniques)
email: "alice@example.com"  // Presque unique → Haute sélectivité

// Champ peu sélectif (peu de valeurs uniques)
gender: "F"  // Seulement 2-3 valeurs → Basse sélectivité

// Index optimal : champ sélectif en premier
db.users.createIndex({ email: 1, gender: 1 })  // ✅ Bon
db.users.createIndex({ gender: 1, email: 1 })  // ❌ Moins efficace
```

### Stratégie 3 : Index partiels

N'indexer qu'un sous-ensemble de documents :

```javascript
// Index partiel : seulement les produits actifs
db.products.createIndex(
    { category: 1, price: 1 },
    { partialFilterExpression: { active: true } }
)

// Plus petit, plus rapide, utilise moins d'espace
```

## Outils de diagnostic

### 1. currentOp() : Opérations en cours

```javascript
db.currentOp()

// Voir les requêtes lentes
db.currentOp({ "secs_running": { $gte: 5 } })
```

### 2. Profiler : Historique des requêtes

```javascript
// Activer le profiler (niveau 1 = lentes, niveau 2 = toutes)
db.setProfilingLevel(1, { slowms: 100 })

// Analyser les requêtes lentes
db.system.profile.find().sort({ ts: -1 }).limit(10)
```

### 3. explain() : Analyse détaillée

```javascript
// Trois niveaux de détail
.explain()                       // Plan uniquement
.explain("executionStats")       // + Stats d'exécution
.explain("allPlansExecution")    // + Tous les plans testés
```

## Checklist de performance

Avant de mettre en production, vérifiez :

```javascript
// 1. Identifier les requêtes fréquentes
db.system.profile.find().sort({ ts: -1 })

// 2. Vérifier l'utilisation des index
db.collection.find({ /* requête */ }).explain("executionStats")

// 3. Ratio docs examined / returned
// Objectif : < 2

// 4. Temps d'exécution
// Objectif : < 100ms pour 95% des requêtes

// 5. Pas de COLLSCAN
// Sauf pour petites collections (< 1000 docs)

// 6. Covered queries quand possible
// totalDocsExamined = 0

// 7. Index couvrent les tris
// Pas de stage SORT en mémoire
```

## Ce qui vous attend dans ce chapitre

### Sections théoriques (5.1 à 5.4)

Vous apprendrez :
- Comment les index fonctionnent en interne (B-tree)
- Les différents types d'index et quand les utiliser
- Les options avancées (unique, sparse, partial, hidden)
- Comment choisir le bon type d'index

### Sections pratiques (5.5 à 5.11)

Vous pratiquerez :
- Créer, modifier, supprimer des index
- Analyser avec `explain()` en profondeur
- Comprendre le Query Planner
- Optimiser des requêtes réelles
- Gérer les index en production
- Monitorer les performances

## Exemples de gains réels

Voici des gains typiques avec une bonne stratégie d'indexation :

| Scénario | Sans index | Avec index | Gain |
|----------|------------|------------|------|
| Recherche dans 1M docs | 5000ms | 5ms | 1000x |
| Tri de 100K docs | 2000ms | 50ms | 40x |
| Jointure $lookup | 8000ms | 200ms | 40x |
| Covered query | 500ms | 20ms | 25x |
| Agrégation complexe | 15000ms | 800ms | 19x |

**Ces gains sont réels et reproductibles avec les bonnes techniques !**

## Points d'attention

### ⚠️ Plus d'index ≠ Plus de performance

```javascript
// ❌ Trop d'index
db.users.createIndex({ username: 1 })
db.users.createIndex({ email: 1 })
db.users.createIndex({ age: 1 })
db.users.createIndex({ city: 1 })
db.users.createIndex({ active: 1 })
db.users.createIndex({ createdAt: 1 })
// → Écritures lentes, mémoire gaspillée

// ✅ Index ciblés sur requêtes fréquentes
db.users.createIndex({ username: 1 })
db.users.createIndex({ city: 1, age: 1 })
db.users.createIndex({ active: 1, createdAt: -1 })
```

### ⚠️ Index inutilisés = Gaspillage

```javascript
// Vérifier l'utilisation des index
db.users.aggregate([
    { $indexStats: {} }
])

// Supprimer les index inutilisés
db.users.dropIndex("unused_index_name")
```

### ⚠️ Ordre des champs important

```javascript
// Index composé
db.users.createIndex({ city: 1, age: 1, username: 1 })

// ✅ Utilisera l'index
db.users.find({ city: "Paris" })
db.users.find({ city: "Paris", age: 30 })
db.users.find({ city: "Paris", age: 30, username: "alice" })

// ❌ N'utilisera PAS l'index
db.users.find({ age: 30 })
db.users.find({ username: "alice" })

// Règle : Le préfixe de l'index doit correspondre
```

## Conseils d'apprentissage pour ce chapitre

### 🎯 Méthodologie recommandée

1. **Créez un jeu de données test significatif** (10K-100K documents minimum)
2. **Mesurez AVANT** d'optimiser (baseline)
3. **Testez chaque type d'index** sur vos requêtes réelles
4. **Utilisez explain() systématiquement**
5. **Documentez** vos décisions d'indexation
6. **Surveillez** l'impact en production

### 💡 Environnement de test

```javascript
// Script pour générer des données de test
use test_performance

// Générer 100,000 utilisateurs
for (let i = 0; i < 100000; i++) {
    db.users.insertOne({
        username: `user${i}`,
        email: `user${i}@example.com`,
        age: Math.floor(Math.random() * 60) + 18,
        city: ["Paris", "Lyon", "Marseille", "Toulouse", "Nice"][Math.floor(Math.random() * 5)],
        active: Math.random() > 0.3,
        createdAt: new Date(Date.now() - Math.random() * 365 * 24 * 60 * 60 * 1000)
    })
}
```

### 🔗 Connexion avec les autres chapitres

- **Chapitre 4** : La modélisation influence la stratégie d'indexation
- **Chapitre 6** : Les agrégations utilisent les index
- **Chapitre 9** : Les index sont répliqués dans le Replica Set
- **Chapitre 10** : Le sharding nécessite un index sur la shard key

---

### 📌 Points clés à retenir de cette introduction

- Les index sont **LA** clé des performances MongoDB
- Un bon index peut améliorer les performances de 10x à 1000x
- `explain()` est votre outil principal pour analyser les performances
- Métriques clés : executionTimeMillis, totalDocsExamined/nReturned, stage
- Objectif : IXSCAN plutôt que COLLSCAN
- Ratio optimal : totalDocsExamined/nReturned proche de 1
- Les index ont un coût : mémoire, disque, écritures plus lentes
- Plus d'index ≠ automatiquement plus de performance
- Testez, mesurez, optimisez dans cet ordre

---

**Durée estimée du chapitre** : 10-12 heures de lecture et pratique
**Niveau** : Intermédiaire avec notions de performance
**Prérequis** : Chapitres 1-4 complétés, compréhension des requêtes et de la modélisation

🎯 **Prochaine étape** : Dans la section 5.1, nous allons plonger en profondeur dans le fonctionnement interne des index et comprendre pourquoi ils sont si essentiels pour les performances de vos applications MongoDB.

---

**Prochaine section** : 5.1 - Comprendre l'importance des index

Prêt à transformer vos requêtes lentes en requêtes ultra-rapides ? Allons-y ! ⚡

⏭️ [Comprendre l'importance des index](/05-index-et-optimisation/01-importance-des-index.md)
