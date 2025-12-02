🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.6 Optimisation des Pipelines d'Agrégation

## Introduction

Créer un pipeline d'agrégation qui fonctionne est une chose. Créer un pipeline **rapide et efficace** en est une autre. L'optimisation des pipelines est essentielle pour garantir que vos agrégations s'exécutent rapidement, même avec des millions de documents.

### Pourquoi l'Optimisation est Importante ?

Imaginez deux itinéraires pour aller au travail :
- **Route A** : 30 km par l'autoroute → 20 minutes
- **Route B** : 15 km par les petites routes avec 20 feux rouges → 45 minutes

Les deux routes arrivent au même endroit, mais l'une est **beaucoup plus efficace**. C'est la même chose avec les pipelines d'agrégation : plusieurs pipelines peuvent produire le même résultat, mais leurs performances peuvent varier considérablement.

### Impact de la Performance

**Pipeline non optimisé :**
- ⏱️ Temps d'exécution : 30 secondes
- 💾 Consommation mémoire : 500 MB
- 🔥 Charge CPU : 80%
- 😞 Expérience utilisateur : Mauvaise

**Pipeline optimisé :**
- ⚡ Temps d'exécution : 2 secondes
- 💾 Consommation mémoire : 50 MB
- 🔥 Charge CPU : 15%
- 😊 Expérience utilisateur : Excellente

## Les Principes de l'Optimisation

### 1. Réduire le Volume de Données le Plus Tôt Possible

**Principe fondamental :** Moins vous avez de données à traiter, plus votre pipeline sera rapide.

```javascript
// ❌ MAUVAIS - Traite toutes les données
db.commandes.aggregate([
  { $lookup: { ... } },        // Joint TOUTES les commandes
  { $unwind: "$articles" },    // Déplie TOUTES les commandes
  { $match: { statut: "payé" } }  // Filtre à la fin
])

// ✅ BON - Réduit le volume immédiatement
db.commandes.aggregate([
  { $match: { statut: "payé" } },  // Filtre D'ABORD
  { $lookup: { ... } },             // Joint moins de données
  { $unwind: "$articles" }          // Déplie moins de données
])
```

**Comparaison visuelle :**

```
Pipeline non optimisé :
1 000 000 documents → $lookup → $unwind → $match → 10 000 résultats
   (traite tout)        (tout)    (tout)    (filtre)

Pipeline optimisé :
1 000 000 documents → $match → 10 000 documents → $lookup → $unwind
   (filtre d'abord)    (10 000)     (traite peu)
```

### 2. Utiliser les Index

Les index permettent à MongoDB de trouver rapidement les documents sans parcourir toute la collection.

**Sans index :**
```
🔍 MongoDB doit lire TOUS les documents pour trouver ceux qui correspondent
📚 1 000 000 documents lus → 50 000 correspondances trouvées
⏱️ Temps : 10 secondes
```

**Avec index :**
```
⚡ MongoDB utilise l'index pour aller directement aux bons documents
📑 Index utilisé → 50 000 documents lus directement
⏱️ Temps : 0.5 secondes
```

### 3. Projeter Uniquement les Champs Nécessaires

Ne gardez que les champs dont vous avez réellement besoin.

```javascript
// ❌ MAUVAIS - Garde tous les champs
db.produits.aggregate([
  { $match: { actif: true } },
  { $lookup: { from: "categories", ... } },  // Récupère tous les champs
  { $group: { _id: "$nom", count: { $sum: 1 } } }  // N'utilise que "nom"
])

// ✅ BON - Projette tôt
db.produits.aggregate([
  { $match: { actif: true } },
  { $project: { nom: 1, categorieId: 1 } },  // Ne garde que ce qui est nécessaire
  { $lookup: { from: "categories", ... } },
  { $group: { _id: "$nom", count: { $sum: 1 } } }
])
```

## Optimisations Automatiques de MongoDB

La bonne nouvelle : MongoDB optimise **automatiquement** certains pipelines !

### 1. Pipeline Coalescing (Fusion d'Étapes)

MongoDB fusionne certaines étapes pour améliorer les performances.

**Exemple : Fusion de $limit et $limit**
```javascript
// Ce que vous écrivez :
[
  { $limit: 100 },
  { $limit: 10 }
]

// Ce que MongoDB exécute :
[
  { $limit: 10 }  // MongoDB garde la limite la plus restrictive
]
```

**Exemple : Fusion de $skip et $limit**
```javascript
// Ce que vous écrivez :
[
  { $skip: 10 },
  { $limit: 5 }
]

// Ce que MongoDB exécute :
[
  { $limit: 15 },  // Skip + Limit fusionnés
  { $skip: 10 }
]
```

### 2. Pipeline Reordering (Réordonnancement)

MongoDB peut réorganiser certaines étapes pour utiliser les index.

**Exemple : $match avant $project**
```javascript
// Ce que vous écrivez :
[
  { $project: { nom: 1, prix: 1 } },
  { $match: { prix: { $gt: 100 } } }
]

// Ce que MongoDB exécute :
[
  { $match: { prix: { $gt: 100 } } },  // Déplacé en premier !
  { $project: { nom: 1, prix: 1 } }
]
```

### 3. Index Utilization (Utilisation d'Index)

MongoDB essaie d'utiliser les index pour les premières étapes.

```javascript
// Si vous avez un index sur { statut: 1, date: -1 }
db.commandes.aggregate([
  { $match: { statut: "payé" } },  // ✅ Utilise l'index
  { $sort: { date: -1 } },         // ✅ Utilise aussi l'index
  { $limit: 10 }
])
```

## L'Ordre Optimal des Étapes

### Règle Générale (du premier au dernier)

```javascript
db.collection.aggregate([
  // 1️⃣ $match - FILTRER TÔT
  { $match: { critères_indexés } },

  // 2️⃣ $project/$addFields - RÉDUIRE LA TAILLE (si nécessaire)
  { $project: { champs_nécessaires: 1 } },

  // 3️⃣ $sort - TRIER (si un index peut être utilisé)
  { $sort: { champ_indexé: 1 } },

  // 4️⃣ $limit - LIMITER TÔT (si possible)
  { $limit: n },

  // 5️⃣ $lookup - JOINTURES (sur des données réduites)
  { $lookup: { ... } },

  // 6️⃣ $unwind - DÉPLIER (après avoir réduit le volume)
  { $unwind: "$tableau" },

  // 7️⃣ $group - REGROUPER
  { $group: { ... } },

  // 8️⃣ $match - FILTRER LES GROUPES (si nécessaire)
  { $match: { résultats_agrégés } },

  // 9️⃣ $sort - TRIER LES RÉSULTATS FINAUX
  { $sort: { champ_calculé: -1 } },

  // 🔟 $skip/$limit - PAGINATION FINALE
  { $skip: n },
  { $limit: m },

  // 1️⃣1️⃣ $project - FORMATAGE FINAL
  { $project: { format_final: 1 } }
])
```

### Exemple Concret : Avant et Après Optimisation

#### ❌ Version Non Optimisée (10 secondes)

```javascript
db.commandes.aggregate([
  // Étape 1: Jointure sur TOUTES les commandes (1M documents)
  {
    $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },

  // Étape 2: Déplie TOUTES les commandes
  { $unwind: "$client" },

  // Étape 3: Déplie tous les articles de toutes les commandes
  { $unwind: "$articles" },

  // Étape 4: Jointure avec produits pour TOUS les articles
  {
    $lookup: {
      from: "produits",
      localField: "articles.produitId",
      foreignField: "_id",
      as: "produit"
    }
  },

  // Étape 5: Déplie tous les produits
  { $unwind: "$produit" },

  // Étape 6: ENFIN, on filtre (mais on a déjà tout traité !)
  {
    $match: {
      date: { $gte: ISODate("2024-01-01") },
      statut: "payé",
      "client.ville": "Paris"
    }
  },

  // Étape 7: Regroupe
  {
    $group: {
      _id: "$produit.categorie",
      total: { $sum: "$articles.montant" }
    }
  },

  // Étape 8: Trie
  { $sort: { total: -1 } },

  // Étape 9: Limite
  { $limit: 10 }
])
```

**Problèmes :**
- 🔴 Filtre à la fin : traite 1M de commandes inutilement
- 🔴 Jointures sur tout : jointures coûteuses sur toutes les données
- 🔴 Pas d'utilisation d'index
- 🔴 Consomme beaucoup de mémoire

#### ✅ Version Optimisée (1 seconde)

```javascript
db.commandes.aggregate([
  // Étape 1: FILTRER D'ABORD (avec index)
  {
    $match: {
      date: { $gte: ISODate("2024-01-01") },
      statut: "payé"
    }
  },
  // → Réduit de 1M à 50K documents

  // Étape 2: Projeter uniquement les champs nécessaires
  {
    $project: {
      clientId: 1,
      articles: 1
    }
  },

  // Étape 3: Jointure sur 50K documents seulement
  {
    $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },

  // Étape 4: Déplie les clients
  { $unwind: "$client" },

  // Étape 5: Filtre sur la ville
  {
    $match: {
      "client.ville": "Paris"
    }
  },
  // → Réduit de 50K à 10K documents

  // Étape 6: Déplie les articles (10K documents seulement)
  { $unwind: "$articles" },

  // Étape 7: Projeter avant la jointure produits
  {
    $project: {
      "articles.produitId": 1,
      "articles.montant": 1
    }
  },

  // Étape 8: Jointure produits
  {
    $lookup: {
      from: "produits",
      localField: "articles.produitId",
      foreignField: "_id",
      as: "produit"
    }
  },

  // Étape 9: Déplie produits
  { $unwind: "$produit" },

  // Étape 10: Regroupe
  {
    $group: {
      _id: "$produit.categorie",
      total: { $sum: "$articles.montant" }
    }
  },

  // Étape 11: Trie
  { $sort: { total: -1 } },

  // Étape 12: Limite
  { $limit: 10 }
])
```

**Améliorations :**
- ✅ Filtre avec index en premier : 1M → 50K documents
- ✅ Deuxième filtre : 50K → 10K documents
- ✅ Jointures sur données réduites
- ✅ Projection pour réduire la taille des documents
- ⚡ **10x plus rapide !**

## Utiliser les Index Efficacement

### Créer des Index Appropriés

Pour optimiser vos pipelines, créez des index sur les champs fréquemment utilisés dans $match et $sort.

```javascript
// Index pour $match sur statut
db.commandes.createIndex({ statut: 1 })

// Index composé pour $match + $sort
db.commandes.createIndex({ statut: 1, date: -1 })

// Index pour les jointures $lookup
db.commandes.createIndex({ clientId: 1 })
db.clients.createIndex({ _id: 1 })  // Déjà existant par défaut
```

### Vérifier l'Utilisation des Index

Les étapes qui **peuvent** utiliser des index :
- ✅ `$match` - Si placé en premier
- ✅ `$sort` - Si après un $match qui utilise un index
- ✅ `$lookup` - Sur les champs de jointure

Les étapes qui **ne peuvent pas** utiliser d'index :
- ❌ `$match` après `$unwind`, `$lookup`, ou `$group`
- ❌ `$sort` sur un champ calculé
- ❌ Toute étape qui transforme les données

### Exemple : Maximiser l'Usage des Index

```javascript
// Index : { categorie: 1, prix: -1 }

// ✅ EXCELLENT - Utilise l'index complètement
db.produits.aggregate([
  { $match: { categorie: "Électronique" } },  // Utilise l'index
  { $sort: { prix: -1 } },                    // Utilise l'index aussi
  { $limit: 10 }
])

// ⚠️ PARTIEL - N'utilise l'index que pour $match
db.produits.aggregate([
  { $match: { categorie: "Électronique" } },  // Utilise l'index
  { $addFields: { prixTTC: { $multiply: ["$prix", 1.20] } } },
  { $sort: { prixTTC: -1 } },  // NE peut PAS utiliser l'index (champ calculé)
  { $limit: 10 }
])

// ❌ MAUVAIS - N'utilise pas l'index du tout
db.produits.aggregate([
  { $addFields: { prixTTC: { $multiply: ["$prix", 1.20] } } },
  { $match: { categorie: "Électronique" } },  // Trop tard pour l'index
  { $sort: { prixTTC: -1 } },
  { $limit: 10 }
])
```

## Utiliser allowDiskUse

Par défaut, chaque étape d'agrégation est limitée à **100 MB de RAM**. Pour les grandes agrégations, utilisez `allowDiskUse`.

### Quand Utiliser allowDiskUse ?

```javascript
// ❌ ERREUR si les données dépassent 100 MB
db.logs.aggregate([
  { $match: { date: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$userId", count: { $sum: 1 } } }
])
// Erreur : "Exceeded memory limit for $group"

// ✅ SOLUTION : allowDiskUse
db.logs.aggregate(
  [
    { $match: { date: { $gte: ISODate("2024-01-01") } } },
    { $group: { _id: "$userId", count: { $sum: 1 } } }
  ],
  { allowDiskUse: true }  // Utilise le disque si nécessaire
)
```

### Avantages et Inconvénients

**Avantages :**
- ✅ Permet de traiter de très gros volumes
- ✅ Évite les erreurs de mémoire

**Inconvénients :**
- ⚠️ Plus lent que la RAM
- ⚠️ Augmente l'utilisation disque

**Recommandation :**
- Optimisez d'abord votre pipeline (filtres, projections)
- N'utilisez `allowDiskUse` que si nécessaire
- C'est un dernier recours, pas une solution première

## Analyser les Performances avec explain()

La méthode `explain()` est votre meilleur ami pour comprendre ce que fait MongoDB.

### Syntaxe de Base

```javascript
db.collection.aggregate(
  [ /* pipeline */ ],
  { explain: true }
)
```

### Niveaux d'Explication

```javascript
// Niveau "queryPlanner" - Plan d'exécution
db.collection.explain("queryPlanner").aggregate([...])

// Niveau "executionStats" - Statistiques d'exécution
db.collection.explain("executionStats").aggregate([...])

// Niveau "allPlansExecution" - Tous les plans testés
db.collection.explain("allPlansExecution").aggregate([...])
```

### Interpréter les Résultats

#### Exemple de Résultat explain()

```javascript
{
  "stages": [
    {
      "$cursor": {
        "queryPlanner": {
          "winningPlan": {
            "stage": "COLLSCAN",  // ⚠️ Collection scan (mauvais)
            "filter": { "statut": { "$eq": "payé" } }
          }
        },
        "executionStats": {
          "executionTimeMillis": 1250,  // ⏱️ Temps d'exécution
          "totalDocsExamined": 1000000, // 📄 Documents examinés
          "totalKeysExamined": 0,       // 🔑 Clés d'index utilisées
          "nReturned": 50000            // ✅ Documents retournés
        }
      }
    }
  ]
}
```

#### Indicateurs Clés

| Indicateur | Signification | Bon ✅ | Mauvais ❌ |
|------------|---------------|---------|------------|
| **stage** | Type de scan | IXSCAN (index) | COLLSCAN (full scan) |
| **executionTimeMillis** | Temps d'exécution | < 100ms | > 1000ms |
| **totalDocsExamined** | Docs examinés | Proche de nReturned | >> nReturned |
| **totalKeysExamined** | Index utilisé | > 0 | 0 |
| **nReturned** | Docs retournés | Variable | - |

### Exemple d'Optimisation avec explain()

#### Avant Optimisation

```javascript
db.commandes.explain("executionStats").aggregate([
  { $match: { statut: "payé" } },
  { $sort: { date: -1 } },
  { $limit: 10 }
])

// Résultat :
// - stage: COLLSCAN ❌
// - totalDocsExamined: 1,000,000
// - executionTimeMillis: 2500ms
// - Index utilisé: NON
```

#### Créer un Index

```javascript
db.commandes.createIndex({ statut: 1, date: -1 })
```

#### Après Optimisation

```javascript
db.commandes.explain("executionStats").aggregate([
  { $match: { statut: "payé" } },
  { $sort: { date: -1 } },
  { $limit: 10 }
])

// Résultat :
// - stage: IXSCAN ✅
// - totalDocsExamined: 10
// - executionTimeMillis: 15ms
// - Index utilisé: { statut: 1, date: -1 }
```

**Amélioration : 165x plus rapide !**

## Techniques d'Optimisation Avancées

### 1. Utiliser $sample au Lieu de $sort + $limit + Random

Pour obtenir des documents aléatoires :

```javascript
// ❌ LENT
db.produits.aggregate([
  { $addFields: { random: { $rand: {} } } },
  { $sort: { random: 1 } },
  { $limit: 10 }
])

// ✅ RAPIDE
db.produits.aggregate([
  { $sample: { size: 10 } }  // Optimisé par MongoDB
])
```

### 2. Déplacer $match Après $unwind (avec précaution)

Si le filtre ne peut être appliqué qu'après $unwind :

```javascript
// Si vous devez filtrer sur un champ du tableau
db.commandes.aggregate([
  { $unwind: "$articles" },
  { $match: { "articles.prix": { $gt: 100 } } }
])

// Mais si possible, filtrer avant :
db.commandes.aggregate([
  { $match: { "articles.prix": { $gt: 100 } } },  // Filtre sur le tableau
  { $unwind: "$articles" },
  { $match: { "articles.prix": { $gt: 100 } } }   // Re-filtre après unwind
])
```

### 3. Utiliser $addFields au Lieu de $project

`$project` peut empêcher l'utilisation d'index si mal placé.

```javascript
// ⚠️ Peut casser l'optimisation
[
  { $project: { nom: 1, prix: 1 } },
  { $match: { prix: { $gt: 100 } } }
]

// ✅ Meilleur
[
  { $match: { prix: { $gt: 100 } } },
  { $addFields: { /* champs calculés */ } }
]
```

### 4. Limiter les Jointures $lookup

Les jointures sont coûteuses. Minimisez-les :

```javascript
// ❌ Jointures multiples en cascade
[
  { $lookup: { from: "table1", ... } },
  { $unwind: "$table1" },
  { $lookup: { from: "table2", ... } },
  { $unwind: "$table2" },
  { $lookup: { from: "table3", ... } }
]

// ✅ Mieux : Dénormaliser si possible
// Stocker les infos essentielles directement dans le document
```

### 5. Batch Processing pour Très Gros Volumes

Pour traiter des millions de documents :

```javascript
// Traiter par lots
const batchSize = 10000
let skip = 0

while (true) {
  const results = db.collection.aggregate([
    { $skip: skip },
    { $limit: batchSize },
    { $match: { ... } },
    { $group: { ... } }
  ])

  if (results.length === 0) break

  // Traiter le lot
  processResults(results)

  skip += batchSize
}
```

## Métriques de Performance à Surveiller

### 1. Temps d'Exécution

```javascript
const start = Date.now()
const results = db.collection.aggregate([...])
const end = Date.now()
console.log(`Temps: ${end - start}ms`)
```

**Objectifs :**
- 🟢 < 100ms : Excellent
- 🟡 100-500ms : Acceptable
- 🟠 500-2000ms : À optimiser
- 🔴 > 2000ms : Problème

### 2. Documents Examinés vs Retournés

```javascript
// Ratio idéal : proche de 1
const ratio = totalDocsExamined / nReturned

// Ratios :
// 🟢 1-5 : Excellent (bonne sélectivité)
// 🟡 5-50 : Acceptable
// 🟠 50-500 : À optimiser
// 🔴 > 500 : Problème sérieux
```

### 3. Utilisation Mémoire

```javascript
db.adminCommand({
  setParameter: 1,
  internalQueryExecMaxBlockingSortBytes: 104857600  // 100 MB par défaut
})
```

Surveiller les erreurs :
```
"Exceeded memory limit for $sort"
"Exceeded memory limit for $group"
```

### 4. Index Hit Rate

Pourcentage de requêtes utilisant un index :
```javascript
db.collection.aggregate([
  { $match: { ... } }
]).explain("executionStats")

// Vérifier : totalKeysExamined > 0 ?
```

## Checklist d'Optimisation

Avant de mettre en production un pipeline, vérifiez :

### ✅ Structure du Pipeline

- [ ] Les $match sont placés le plus tôt possible
- [ ] Les filtres les plus sélectifs sont en premier
- [ ] $limit est utilisé après $sort pour Top N
- [ ] $project réduit les champs avant les opérations coûteuses
- [ ] Les jointures $lookup sont sur des données filtrées
- [ ] $unwind est utilisé après avoir réduit le volume

### ✅ Index

- [ ] Index créés sur les champs de $match
- [ ] Index composé pour $match + $sort
- [ ] Index sur les champs de jointure ($lookup)
- [ ] Pas d'index inutilisés
- [ ] explain() confirme l'utilisation d'index (IXSCAN)

### ✅ Performance

- [ ] executionTimeMillis < 500ms (idéalement < 100ms)
- [ ] Ratio docsExamined/nReturned < 10
- [ ] Pas d'erreurs de mémoire
- [ ] allowDiskUse uniquement si nécessaire
- [ ] Tests avec des volumes de production

### ✅ Maintenabilité

- [ ] Pipeline commenté et documenté
- [ ] Variables extraites pour réutilisation
- [ ] Décomposition en étapes logiques claires
- [ ] Tests unitaires des transformations

## Exemples de Patterns Optimisés

### Pattern 1 : Top N avec Groupement

```javascript
// Objectif : Top 10 des catégories par ventes
db.ventes.aggregate([
  // Filtrer la période
  { $match: {
      date: { $gte: ISODate("2024-01-01") }
    }
  },

  // Regrouper par catégorie
  { $group: {
      _id: "$categorie",
      total: { $sum: "$montant" }
    }
  },

  // Trier par total décroissant
  { $sort: { total: -1 } },

  // Garder top 10
  { $limit: 10 },

  // Formater
  { $project: {
      _id: 0,
      categorie: "$_id",
      total: 1
    }
  }
])
```

### Pattern 2 : Pagination Optimisée

```javascript
// Page N (20 résultats par page)
const page = 3
const pageSize = 20

db.produits.aggregate([
  // Filtrer
  { $match: { actif: true } },

  // Trier (avec index)
  { $sort: { nom: 1 } },

  // Calculer le total (pour l'UI)
  { $facet: {
      metadata: [{ $count: "total" }],
      data: [
        { $skip: (page - 1) * pageSize },
        { $limit: pageSize }
      ]
    }
  }
])
```

### Pattern 3 : Jointure Optimisée

```javascript
// Objectif : Enrichir commandes avec client (seulement pour Paris)
db.commandes.aggregate([
  // Pré-filtrer les commandes
  { $match: {
      date: { $gte: ISODate("2024-01-01") },
      statut: "payé"
    }
  },

  // Ne garder que les champs nécessaires
  { $project: {
      clientId: 1,
      montant: 1,
      date: 1
    }
  },

  // Jointure
  { $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },

  // Déplie
  { $unwind: "$client" },

  // Filtre sur la ville (après jointure)
  { $match: {
      "client.ville": "Paris"
    }
  },

  // Projection finale minimale
  { $project: {
      montant: 1,
      date: 1,
      "client.nom": 1,
      "client.ville": 1
    }
  }
])
```

## Erreurs d'Optimisation Courantes

### 1. Trier Avant de Filtrer

```javascript
// ❌ MAUVAIS
[
  { $sort: { date: -1 } },     // Trie 1M de docs
  { $match: { actif: true } }  // Puis filtre
]

// ✅ BON
[
  { $match: { actif: true } }, // Filtre → 100K docs
  { $sort: { date: -1 } }      // Trie 100K docs
]
```

### 2. Projeter Trop Tard

```javascript
// ❌ MAUVAIS
[
  { $lookup: { ... } },         // Joint avec tous les champs
  { $unwind: "$data" },
  { $project: { nom: 1, prix: 1 } }  // Projection à la fin
]

// ✅ BON
[
  { $project: { nom: 1, prix: 1, refId: 1 } },  // Réduit la taille
  { $lookup: { ... } },         // Joint avec moins de données
  { $unwind: "$data" }
]
```

### 3. Ne Pas Utiliser $limit Avec $sort

```javascript
// ❌ MAUVAIS - Trie tout
[
  { $sort: { score: -1 } }
]

// ✅ BON - MongoDB optimise
[
  { $sort: { score: -1 } },
  { $limit: 10 }  // MongoDB ne garde que top 10 en mémoire
]
```

### 4. Jointures en Cascade Non Filtrées

```javascript
// ❌ TRÈS LENT
[
  { $lookup: { from: "A", ... } },
  { $unwind: "$A" },
  { $lookup: { from: "B", ... } },
  { $unwind: "$B" },
  { $lookup: { from: "C", ... } }
]

// ✅ MIEUX - Filtrer entre les jointures
[
  { $lookup: { from: "A", ... } },
  { $unwind: "$A" },
  { $match: { "A.important": true } },  // Filtre !
  { $lookup: { from: "B", ... } },
  { $unwind: "$B" },
  { $match: { "B.valide": true } },     // Filtre !
  { $lookup: { from: "C", ... } }
]
```

## Résumé

L'optimisation des pipelines d'agrégation repose sur quelques principes clés :

### Principes Fondamentaux

1. **🎯 Filtrer tôt et souvent**
   - $match en premier
   - Utiliser les index

2. **📦 Réduire la taille des données**
   - $project pour ne garder que le nécessaire
   - $limit dès que possible

3. **🔍 Utiliser les index**
   - Créer des index appropriés
   - Vérifier avec explain()

4. **⚡ Optimiser l'ordre**
   - Filter → Sort → Limit → Join → Group
   - Laisser MongoDB réorganiser quand possible

5. **📊 Mesurer et analyser**
   - explain() pour comprendre
   - Surveiller les métriques
   - Itérer et améliorer

### Mémo Rapide

```javascript
// Template de pipeline optimisé
db.collection.aggregate([
  // ✅ 1. Filtrer avec index
  { $match: { champ_indexé: valeur } },

  // ✅ 2. Projeter si nécessaire
  { $project: { champs_utiles: 1 } },

  // ✅ 3. Trier avec index si possible
  { $sort: { champ_indexé: 1 } },

  // ✅ 4. Limiter tôt
  { $limit: n },

  // ✅ 5. Jointures sur données réduites
  { $lookup: { ... } },

  // ✅ 6. Déplier après filtrage
  { $unwind: "$tableau" },

  // ✅ 7. Regrouper
  { $group: { ... } },

  // ✅ 8. Formater en dernier
  { $project: { format_final: 1 } }
])
```

### Points Clés à Retenir

> **L'ordre des étapes peut faire la différence entre 30 secondes et 1 seconde d'exécution.**

> **Un bon index peut améliorer les performances de 100x ou plus.**

> **explain() est votre meilleur ami pour comprendre et optimiser.**

> **Optimisez d'abord la logique, utilisez allowDiskUse en dernier recours.**

---

**Règle d'or de l'optimisation :**
> Mesurez d'abord, optimisez ensuite, vérifiez toujours.

Dans la section suivante (6.7), nous explorerons les vues et les vues matérialisées, qui permettent de sauvegarder des pipelines optimisés pour une réutilisation facile.

⏭️ [Vues (Views) et vues matérialisées](/06-framework-agregation/07-vues-materialisees.md)
