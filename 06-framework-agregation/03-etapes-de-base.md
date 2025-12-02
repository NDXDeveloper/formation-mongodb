🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.3 Étapes de Base

## Introduction

Maintenant que vous comprenez le concept de pipeline, il est temps de découvrir les **étapes de base** (stages) qui constituent les briques fondamentales de toute agrégation MongoDB. Ces étapes sont comme les outils essentiels d'une boîte à outils : une fois que vous les maîtrisez, vous pouvez construire des pipelines puissants et efficaces.

Dans cette section, nous allons explorer les six étapes de base les plus couramment utilisées :

1. **$match** - Filtrer les documents
2. **$project** - Sélectionner et transformer les champs
3. **$group** - Regrouper et calculer des agrégations
4. **$sort** - Trier les résultats
5. **$limit** et **$skip** - Contrôler la pagination
6. **$count** - Compter les documents

## Vue d'Ensemble des Étapes de Base

### Tableau Récapitulatif

| Étape | Rôle | Équivalent SQL | Usage typique |
|-------|------|----------------|---------------|
| **$match** | Filtre les documents | WHERE | Réduire le volume de données |
| **$project** | Sélectionne/transforme les champs | SELECT | Formater la sortie |
| **$group** | Regroupe et agrège | GROUP BY | Calculer des statistiques |
| **$sort** | Trie les documents | ORDER BY | Ordonner les résultats |
| **$limit** | Limite le nombre de résultats | LIMIT | Pagination, top N |
| **$skip** | Saute des documents | OFFSET | Pagination |
| **$count** | Compte les documents | COUNT(*) | Obtenir un total |

### Fréquence d'Utilisation

Les étapes de base sont présentes dans la majorité des pipelines MongoDB :

```
📊 Fréquence d'utilisation estimée :

$match   ████████████████████ 95%  (Presque toujours)
$group   ███████████████      75%  (Très fréquent)
$project █████████████        65%  (Fréquent)
$sort    ████████████         60%  (Fréquent)
$limit   ██████████           50%  (Moyen)
$skip    ████████             40%  (Moyen)
$count   ████                 20%  (Occasionnel)
```

## Catégorisation des Étapes de Base

### 1. Étapes de Filtrage

**But :** Réduire le nombre de documents dans le pipeline

#### **$match** - Le Filtre Universel
```javascript
{ $match: { critères } }
```
- Filtre les documents selon des conditions
- Fonctionne comme `find()`
- **Toujours placer en premier** pour optimiser les performances

**Exemple :**
```javascript
{ $match: { age: { $gte: 18 } } }
// Garde seulement les documents où age >= 18
```

#### **$limit** - Limite de Résultats
```javascript
{ $limit: nombre }
```
- Garde seulement les N premiers documents
- Utile pour le Top N ou la pagination

**Exemple :**
```javascript
{ $limit: 10 }
// Garde les 10 premiers documents
```

#### **$skip** - Sauter des Documents
```javascript
{ $skip: nombre }
```
- Saute les N premiers documents
- Utilisé avec `$limit` pour la pagination

**Exemple :**
```javascript
{ $skip: 20 }
// Saute les 20 premiers documents
```

### 2. Étapes de Transformation

**But :** Modifier la structure ou le contenu des documents

#### **$project** - Le Sculpteur de Documents
```javascript
{ $project: { champs } }
```
- Sélectionne les champs à afficher/masquer
- Crée de nouveaux champs calculés
- Renomme des champs

**Exemple :**
```javascript
{ $project: { nom: 1, age: 1, _id: 0 } }
// Affiche seulement nom et age, masque _id
```

### 3. Étapes d'Agrégation

**But :** Regrouper des données et calculer des statistiques

#### **$group** - Le Regroupeur Calculateur
```javascript
{ $group: { _id: expression, champs_calculés } }
```
- Regroupe les documents selon un critère
- Calcule des agrégations (somme, moyenne, etc.)
- Équivalent de GROUP BY en SQL

**Exemple :**
```javascript
{ $group: { _id: "$categorie", total: { $sum: "$prix" } } }
// Regroupe par catégorie et calcule la somme des prix
```

### 4. Étapes de Tri

**But :** Ordonner les résultats

#### **$sort** - L'Organisateur
```javascript
{ $sort: { champ: ordre } }
```
- Trie les documents
- 1 = ordre croissant, -1 = ordre décroissant

**Exemple :**
```javascript
{ $sort: { date: -1 } }
// Trie par date, du plus récent au plus ancien
```

### 5. Étapes de Comptage

**But :** Obtenir des totaux

#### **$count** - Le Compteur
```javascript
{ $count: "nomDuChamp" }
```
- Compte le nombre de documents
- Retourne un document avec le total

**Exemple :**
```javascript
{ $count: "total" }
// Retourne { "total": 42 }
```

## Patterns d'Utilisation Courants

### Pattern 1 : Filtrer et Compter

**Objectif :** Combien de documents correspondent à un critère ?

```javascript
db.collection.aggregate([
  { $match: { statut: "actif" } },
  { $count: "nombreActifs" }
])
```

**Ordre des étapes :**
```
Documents → [$match] → Documents filtrés → [$count] → { nombreActifs: X }
```

### Pattern 2 : Filtrer, Trier et Limiter (Top N)

**Objectif :** Les N meilleurs/derniers selon un critère

```javascript
db.collection.aggregate([
  { $match: { categorie: "sport" } },
  { $sort: { score: -1 } },
  { $limit: 5 }
])
```

**Ordre des étapes :**
```
Documents → [$match] → Filtrés → [$sort] → Triés → [$limit] → Top 5
```

### Pattern 3 : Regrouper et Calculer

**Objectif :** Statistiques par groupe

```javascript
db.collection.aggregate([
  { $match: { annee: 2024 } },
  { $group: {
      _id: "$mois",
      total: { $sum: "$montant" },
      moyenne: { $avg: "$montant" }
    }
  },
  { $sort: { _id: 1 } }
])
```

**Ordre des étapes :**
```
Documents → [$match] → Filtrés → [$group] → Agrégés → [$sort] → Résultat trié
```

### Pattern 4 : Pagination

**Objectif :** Afficher des résultats page par page

```javascript
// Page 1 (résultats 1-10)
db.collection.aggregate([
  { $match: { actif: true } },
  { $sort: { date: -1 } },
  { $skip: 0 },
  { $limit: 10 }
])

// Page 2 (résultats 11-20)
db.collection.aggregate([
  { $match: { actif: true } },
  { $sort: { date: -1 } },
  { $skip: 10 },
  { $limit: 10 }
])

// Page 3 (résultats 21-30)
db.collection.aggregate([
  { $match: { actif: true } },
  { $sort: { date: -1 } },
  { $skip: 20 },
  { $limit: 10 }
])
```

### Pattern 5 : Filtrer, Regrouper, Filtrer à Nouveau

**Objectif :** Filtrer les groupes selon leurs agrégations

```javascript
db.ventes.aggregate([
  // Étape 1: Filtre initial
  { $match: { annee: 2024 } },

  // Étape 2: Regroupement
  { $group: {
      _id: "$vendeur",
      totalVentes: { $sum: "$montant" }
    }
  },

  // Étape 3: Filtre sur les résultats agrégés
  { $match: { totalVentes: { $gte: 10000 } } },

  // Étape 4: Tri
  { $sort: { totalVentes: -1 } }
])
```

**Note importante :** Le deuxième `$match` filtre les **résultats du $group**, pas les documents originaux.

### Pattern 6 : Transformer et Afficher

**Objectif :** Créer une vue formatée des données

```javascript
db.produits.aggregate([
  { $match: { stock: { $gt: 0 } } },
  { $sort: { nom: 1 } },
  { $project: {
      _id: 0,
      nom: 1,
      prixTTC: { $multiply: ["$prixHT", 1.20] },
      disponible: true
    }
  }
])
```

## Ordre Recommandé des Étapes

Pour des performances optimales, suivez généralement cet ordre :

```javascript
db.collection.aggregate([
  // 1. FILTRER TÔT (réduit le volume)
  { $match: { ... } },

  // 2. TRIER (si nécessaire avant group)
  { $sort: { ... } },

  // 3. LIMITER (si possible avant des opérations coûteuses)
  { $limit: ... },

  // 4. REGROUPER (opération coûteuse)
  { $group: { ... } },

  // 5. FILTRER LES GROUPES (si nécessaire)
  { $match: { ... } },

  // 6. TRIER LES RÉSULTATS
  { $sort: { ... } },

  // 7. PAGINATION
  { $skip: ... },
  { $limit: ... },

  // 8. FORMATER (projection en dernier)
  { $project: { ... } }
])
```

### Pourquoi Cet Ordre ?

1. **$match en premier** → Réduit immédiatement le volume de données
2. **$sort avant $group** → Peut être utilisé par certaines opérations de groupe
3. **$limit tôt** → Traite moins de documents dans les étapes suivantes
4. **$group au milieu** → Opération coûteuse, mais sur moins de données
5. **$project en dernier** → Formate le résultat final

## Combinaisons Efficaces

### ✅ Bon : Filtrer puis Trier

```javascript
db.collection.aggregate([
  { $match: { categorie: "A" } },    // Filtre 1000 → 100 documents
  { $sort: { date: -1 } }             // Trie seulement 100 documents
])
```

### ❌ Moins Bon : Trier puis Filtrer

```javascript
db.collection.aggregate([
  { $sort: { date: -1 } },            // Trie 1000 documents
  { $match: { categorie: "A" } }      // Puis garde 100 documents
])
```

### ✅ Bon : Limiter Après le Tri

```javascript
db.collection.aggregate([
  { $sort: { score: -1 } },
  { $limit: 10 }                      // Top 10
])
```

### ❌ Erreur : Limiter Avant le Tri

```javascript
db.collection.aggregate([
  { $limit: 10 },                     // 10 premiers (pas forcément les meilleurs)
  { $sort: { score: -1 } }            // Trie seulement ces 10
])
```

## Cas d'Usage par Étape

### Quand Utiliser $match ?
- ✅ Filtrer par statut, date, catégorie
- ✅ Restreindre à une période
- ✅ Exclure des documents invalides
- ✅ En PREMIER dans le pipeline

### Quand Utiliser $project ?
- ✅ Masquer des champs sensibles
- ✅ Calculer de nouveaux champs
- ✅ Renommer des champs
- ✅ En DERNIER pour formater la sortie

### Quand Utiliser $group ?
- ✅ Calculer des totaux, moyennes, sommes
- ✅ Regrouper par catégorie, date, région
- ✅ Compter des occurrences
- ✅ Trouver des min/max par groupe

### Quand Utiliser $sort ?
- ✅ Afficher du plus récent au plus ancien
- ✅ Classer par pertinence, score
- ✅ Avant $limit pour un Top N
- ✅ Après $group pour trier les résultats agrégés

### Quand Utiliser $limit et $skip ?
- ✅ Pagination d'une liste
- ✅ Top N des résultats
- ✅ Échantillonnage (avec $sample souvent mieux)
- ✅ Limiter les résultats pour tests

### Quand Utiliser $count ?
- ✅ Nombre total de documents correspondants
- ✅ Statistiques simples
- ✅ En fin de pipeline pour compter les résultats

## Exemple Complet : Analyse de Blog

Imaginons une collection d'articles de blog :

```javascript
// Collection: articles
{
  "_id": ObjectId("..."),
  "titre": "Introduction à MongoDB",
  "auteur": "Alice",
  "categorie": "Tutoriels",
  "vues": 1250,
  "likes": 87,
  "publie": true,
  "datePublication": ISODate("2024-01-15")
}
```

### Objectif : Top 5 des Articles les Plus Populaires

```javascript
db.articles.aggregate([
  // 1. Garder seulement les articles publiés
  {
    $match: { publie: true }
  },

  // 2. Calculer un score de popularité
  {
    $project: {
      titre: 1,
      auteur: 1,
      categorie: 1,
      scorePopularite: { $add: ["$vues", { $multiply: ["$likes", 10] }] }
    }
  },

  // 3. Trier par score décroissant
  {
    $sort: { scorePopularite: -1 }
  },

  // 4. Garder les 5 premiers
  {
    $limit: 5
  },

  // 5. Formater la sortie finale
  {
    $project: {
      _id: 0,
      titre: 1,
      auteur: 1,
      categorie: 1,
      score: "$scorePopularite"
    }
  }
])
```

**Résultat :**
```javascript
[
  { "titre": "Guide complet MongoDB", "auteur": "Alice", "categorie": "Tutoriels", "score": 2120 },
  { "titre": "Performance MongoDB", "auteur": "Bob", "categorie": "Avancé", "score": 1980 },
  { "titre": "MongoDB et Node.js", "auteur": "Alice", "categorie": "Tutoriels", "score": 1750 },
  { "titre": "Agrégation avancée", "auteur": "Charlie", "categorie": "Avancé", "score": 1650 },
  { "titre": "Modélisation de données", "auteur": "Bob", "categorie": "Intermédiaire", "score": 1520 }
]
```

### Décomposition du Pipeline

| Étape | Entrée | Sortie | Opération |
|-------|--------|--------|-----------|
| **$match** | 1000 articles | 850 publiés | Filtre publie: true |
| **$project** | 850 articles | 850 avec score | Calcule scorePopularite |
| **$sort** | 850 articles | 850 triés | Trie par score |
| **$limit** | 850 articles | 5 articles | Garde top 5 |
| **$project** | 5 articles | 5 formatés | Formate sortie |

## Comparaison avec des Requêtes Simples

### Avec find() - Limité

```javascript
// ❌ On ne peut PAS faire ça avec find()
db.articles.find({
  publie: true
}).sort({
  // Impossible de trier par un champ calculé qui n'existe pas !
  scorePopularite: -1
}).limit(5)
```

### Avec aggregate() - Puissant

```javascript
// ✅ aggregate() permet des calculs et transformations
db.articles.aggregate([
  { $match: { publie: true } },
  { $addFields: {
      scorePopularite: { $add: ["$vues", { $multiply: ["$likes", 10] }] }
    }
  },
  { $sort: { scorePopularite: -1 } },
  { $limit: 5 }
])
```

## Erreurs Courantes à Éviter

### 1. Oublier les Accolades

```javascript
// ❌ ERREUR
db.collection.aggregate([
  $match: { age: 18 }  // Manque les accolades !
])

// ✅ CORRECT
db.collection.aggregate([
  { $match: { age: 18 } }
])
```

### 2. Mauvais Ordre des Étapes

```javascript
// ❌ INEFFICACE
db.collection.aggregate([
  { $sort: { ... } },    // Trie TOUT
  { $match: { ... } }    // Puis filtre
])

// ✅ EFFICACE
db.collection.aggregate([
  { $match: { ... } },   // Filtre d'abord
  { $sort: { ... } }     // Trie moins de données
])
```

### 3. Limiter Avant de Trier pour un Top N

```javascript
// ❌ ERREUR - Ne donne PAS le top 10
db.collection.aggregate([
  { $limit: 10 },        // 10 premiers (aléatoires)
  { $sort: { score: -1 } } // Trie ces 10
])

// ✅ CORRECT - Donne bien le top 10
db.collection.aggregate([
  { $sort: { score: -1 } }, // Trie tout
  { $limit: 10 }            // Garde les 10 meilleurs
])
```

### 4. Utiliser $count Trop Tôt

```javascript
// ❌ ERREUR - $count termine le pipeline
db.collection.aggregate([
  { $match: { ... } },
  { $count: "total" },
  { $sort: { ... } }     // Ne sera jamais exécuté !
])

// ✅ CORRECT - $count en dernier
db.collection.aggregate([
  { $match: { ... } },
  { $sort: { ... } },
  { $count: "total" }    // En dernier
])
```

## Mémo Rapide

```javascript
// Structure générale d'un pipeline avec étapes de base
db.collection.aggregate([

  // FILTRER (en premier !)
  { $match: { condition } },

  // TRIER
  { $sort: { champ: 1 ou -1 } },

  // LIMITER
  { $limit: nombre },
  { $skip: nombre },

  // REGROUPER
  { $group: {
      _id: "$champ",
      calcul: { $operateur: "$champ" }
    }
  },

  // PROJETER (en dernier !)
  { $project: {
      champ: 1 ou 0,
      nouveau: expression
    }
  },

  // COMPTER (tout à la fin)
  { $count: "nomChamp" }

])
```

## Récapitulatif

Les **étapes de base** sont les fondations de l'agrégation MongoDB :

| Étape | Action | Position typique |
|-------|--------|------------------|
| **$match** | Filtre | Début du pipeline |
| **$project** | Transforme | Fin du pipeline |
| **$group** | Agrège | Milieu du pipeline |
| **$sort** | Trie | Avant $limit ou après $group |
| **$limit/$skip** | Pagine | Après $sort |
| **$count** | Compte | Fin du pipeline |

**Règle d'or :**
> Filtrez tôt, transformez tard, et optimisez l'ordre des étapes !

## Prochaines Étapes

Dans les sections suivantes, nous explorerons en détail chacune de ces étapes de base :

- **6.3.1** - $match : Filtrage avancé
- **6.3.2** - $project : Projection et transformation
- **6.3.3** - $group : Regroupement et agrégation
- **6.3.4** - $sort : Tri des résultats
- **6.3.5** - $limit et $skip : Pagination
- **6.3.6** - $count : Comptage de documents

Chaque section détaillera la syntaxe, les options, les cas d'usage et les bonnes pratiques spécifiques à chaque étape.

---

**À retenir :**
> Les étapes de base sont comme les notes de musique : simples individuellement, mais capables de créer des symphonies complexes quand elles sont combinées intelligemment.

⏭️ [$match](/06-framework-agregation/03.1-match.md)
