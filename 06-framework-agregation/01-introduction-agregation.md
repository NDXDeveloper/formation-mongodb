🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.1 Introduction au Framework d'Agrégation

## Qu'est-ce que le Framework d'Agrégation ?

Le **framework d'agrégation** de MongoDB est un outil puissant qui permet de traiter, transformer et analyser des données directement au sein de la base de données. Imaginez-le comme une chaîne de montage où vos documents passent par différentes étapes de transformation pour obtenir le résultat souhaité.

### Analogie Simple

Pensez au framework d'agrégation comme à une **recette de cuisine** :
- Vous commencez avec des **ingrédients bruts** (vos documents)
- Vous appliquez différentes **étapes de préparation** (filtrage, découpage, mélange)
- Vous obtenez un **plat final** transformé (résultat agrégé)

## Pourquoi Utiliser le Framework d'Agrégation ?

### Limites des Requêtes Simples

Les méthodes CRUD de base (`find()`, `findOne()`) sont excellentes pour récupérer des documents, mais elles ont des limitations :

```javascript
// Requête simple - récupère les documents tels quels
db.commandes.find({ statut: "livré" })
```

**Ce que vous NE pouvez PAS faire facilement avec find() :**
- ❌ Calculer des totaux, moyennes, sommes
- ❌ Regrouper des données par catégorie
- ❌ Joindre des données de plusieurs collections
- ❌ Transformer la structure des documents
- ❌ Effectuer des calculs complexes

### Ce que Permet l'Agrégation

Avec le framework d'agrégation, vous pouvez :

✅ **Calculer des statistiques**
```javascript
// Calculer le montant total des ventes par mois
db.ventes.aggregate([
  { $group: { _id: "$mois", total: { $sum: "$montant" } } }
])
```

✅ **Transformer les données**
```javascript
// Ajouter un champ calculé
db.produits.aggregate([
  { $addFields: { prixTTC: { $multiply: ["$prixHT", 1.20] } } }
])
```

✅ **Filtrer et trier de manière avancée**
```javascript
// Combiner plusieurs opérations
db.utilisateurs.aggregate([
  { $match: { age: { $gte: 18 } } },
  { $sort: { score: -1 } },
  { $limit: 10 }
])
```

✅ **Joindre des collections**
```javascript
// Équivalent d'un JOIN SQL
db.commandes.aggregate([
  { $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "infoClient"
    }
  }
])
```

## Concept Fondamental : Le Pipeline

### Qu'est-ce qu'un Pipeline ?

Un **pipeline d'agrégation** est une séquence d'**étapes** (stages) que vos documents traversent. Chaque étape effectue une opération spécifique et passe le résultat à l'étape suivante.

### Visualisation du Pipeline

```
Documents originaux
        ↓
   [Étape 1: Filtrage]
        ↓
   [Étape 2: Tri]
        ↓
   [Étape 3: Regroupement]
        ↓
   [Étape 4: Calcul]
        ↓
   Résultat final
```

### Syntaxe de Base

```javascript
db.collection.aggregate([
  { étape1 },
  { étape2 },
  { étape3 },
  // ... autant d'étapes que nécessaire
])
```

**Points importants :**
- Le pipeline est un **tableau** `[]`
- Chaque étape est un **objet** `{}`
- Les étapes sont exécutées **dans l'ordre**
- Chaque étape reçoit les résultats de l'étape précédente

## Exemple Concret : Analyse de Ventes

Imaginons une collection de ventes avec cette structure :

```javascript
// Collection: ventes
{
  "_id": 1,
  "produit": "Ordinateur",
  "categorie": "Électronique",
  "montant": 1200,
  "quantite": 2,
  "date": ISODate("2024-01-15"),
  "vendeur": "Alice"
}
```

### Objectif : Trouver le Top 3 des Vendeurs

**Sans agrégation** (impossible ou très complexe)
```javascript
// ❌ Impossible de faire cela simplement avec find()
db.ventes.find() // Récupère tout, calculs côté application
```

**Avec agrégation** (simple et efficace)
```javascript
// ✅ Pipeline d'agrégation
db.ventes.aggregate([
  // Étape 1: Regrouper par vendeur et calculer le total
  {
    $group: {
      _id: "$vendeur",
      totalVentes: { $sum: "$montant" },
      nombreVentes: { $sum: 1 }
    }
  },

  // Étape 2: Trier par total décroissant
  {
    $sort: { totalVentes: -1 }
  },

  // Étape 3: Garder seulement les 3 premiers
  {
    $limit: 3
  },

  // Étape 4: Renommer le champ _id en vendeur
  {
    $project: {
      _id: 0,
      vendeur: "$_id",
      totalVentes: 1,
      nombreVentes: 1
    }
  }
])
```

**Résultat :**
```javascript
[
  { vendeur: "Alice", totalVentes: 45000, nombreVentes: 25 },
  { vendeur: "Bob", totalVentes: 38000, nombreVentes: 20 },
  { vendeur: "Charlie", totalVentes: 32000, nombreVentes: 18 }
]
```

### Décomposition de l'Exemple

| Étape | Ce qui se passe | Résultat intermédiaire |
|-------|-----------------|------------------------|
| **$group** | Regroupe tous les documents par vendeur et calcule les totaux | Liste de vendeurs avec leurs totaux |
| **$sort** | Trie les vendeurs par totalVentes (du plus grand au plus petit) | Liste triée |
| **$limit** | Ne garde que les 3 premiers | Top 3 |
| **$project** | Renomme et sélectionne les champs à afficher | Format final propre |

## Avantages du Framework d'Agrégation

### 1. **Performance**
Les calculs sont effectués **directement dans MongoDB**, ce qui est beaucoup plus rapide que de récupérer tous les documents et calculer côté application.

**Comparaison :**
```javascript
// ❌ Approche inefficace (côté application)
const ventes = db.ventes.find().toArray()
let total = 0
ventes.forEach(v => total += v.montant) // Lent !

// ✅ Approche efficace (avec agrégation)
db.ventes.aggregate([
  { $group: { _id: null, total: { $sum: "$montant" } } }
]) // Rapide !
```

### 2. **Flexibilité**
Vous pouvez combiner des dizaines d'étapes différentes pour créer des analyses complexes.

### 3. **Lisibilité**
Le code en pipeline est structuré et facile à comprendre, chaque étape ayant un rôle précis.

### 4. **Optimisation Automatique**
MongoDB optimise automatiquement certains pipelines pour améliorer les performances.

## Différence avec SQL

Si vous connaissez SQL, voici un parallèle :

| SQL | MongoDB Aggregation |
|-----|---------------------|
| `SELECT` | `$project` |
| `WHERE` | `$match` |
| `GROUP BY` | `$group` |
| `ORDER BY` | `$sort` |
| `LIMIT` | `$limit` |
| `JOIN` | `$lookup` |
| `HAVING` | `$match` (après $group) |

**Exemple SQL :**
```sql
SELECT vendeur, SUM(montant) AS total
FROM ventes
WHERE categorie = 'Électronique'
GROUP BY vendeur
HAVING SUM(montant) > 10000
ORDER BY total DESC
LIMIT 5
```

**Équivalent MongoDB :**
```javascript
db.ventes.aggregate([
  { $match: { categorie: "Électronique" } },
  { $group: { _id: "$vendeur", total: { $sum: "$montant" } } },
  { $match: { total: { $gt: 10000 } } },
  { $sort: { total: -1 } },
  { $limit: 5 }
])
```

## Types d'Opérations Courantes

Le framework d'agrégation peut être utilisé pour :

### 📊 **Analyses Statistiques**
- Calcul de moyennes, médianes, sommes
- Comptage de documents
- Recherche de min/max

### 🔍 **Transformation de Données**
- Modification de structure des documents
- Ajout de champs calculés
- Conversion de types

### 🔗 **Jointures**
- Combiner des données de plusieurs collections
- Enrichissement de documents

### 📈 **Reporting**
- Rapports de ventes
- Tableaux de bord
- Analyses temporelles

### 🎯 **Filtrage Avancé**
- Conditions complexes
- Recherche dans des tableaux imbriqués
- Filtrage sur des champs calculés

## Cas d'Usage Réels

### 1. **E-commerce : Analyse des Ventes**
```javascript
// Ventes par catégorie pour le mois en cours
db.commandes.aggregate([
  { $match: {
      date: {
        $gte: new Date("2024-01-01"),
        $lt: new Date("2024-02-01")
      }
    }
  },
  { $group: {
      _id: "$categorie",
      totalVentes: { $sum: "$montant" },
      commandesMoyennes: { $avg: "$montant" }
    }
  },
  { $sort: { totalVentes: -1 } }
])
```

### 2. **Réseau Social : Utilisateurs les Plus Actifs**
```javascript
// Top 10 des utilisateurs par nombre de posts
db.posts.aggregate([
  { $group: {
      _id: "$auteurId",
      nombrePosts: { $sum: 1 },
      likesTotal: { $sum: "$likes" }
    }
  },
  { $sort: { nombrePosts: -1 } },
  { $limit: 10 }
])
```

### 3. **IoT : Moyennes de Capteurs**
```javascript
// Température moyenne par heure
db.mesures.aggregate([
  { $group: {
      _id: {
        annee: { $year: "$timestamp" },
        mois: { $month: "$timestamp" },
        jour: { $dayOfMonth: "$timestamp" },
        heure: { $hour: "$timestamp" }
      },
      temperatureMoyenne: { $avg: "$temperature" },
      humiditemoyenne: { $avg: "$humidite" }
    }
  }
])
```

## Quand Utiliser l'Agrégation ?

### ✅ Utilisez l'Agrégation Quand :
- Vous devez calculer des statistiques (sommes, moyennes, comptages)
- Vous devez regrouper des données
- Vous devez transformer la structure des documents
- Vous devez joindre des collections
- Vous devez effectuer des analyses complexes

### ⚠️ Utilisez find() Quand :
- Vous récupérez des documents sans transformation
- Vous faites une simple recherche par critères
- Vous n'avez pas besoin de calculs ou de regroupements

## Structure d'un Document de Pipeline

Chaque étape du pipeline est un objet avec un **opérateur** spécial commençant par `$` :

```javascript
{
  $operateur: {
    // configuration de l'opérateur
  }
}
```

**Exemples d'opérateurs :**
- `$match` - Filtre les documents
- `$group` - Regroupe et calcule
- `$sort` - Trie les résultats
- `$project` - Sélectionne/transforme les champs
- `$limit` - Limite le nombre de résultats
- `$lookup` - Joint des collections

## Concepts Clés à Retenir

1. **Pipeline = Suite d'Étapes**
   - Chaque étape transforme les données
   - L'ordre des étapes est important

2. **Traitement Côté Serveur**
   - Les calculs se font dans MongoDB
   - Plus performant que côté application

3. **Expressivité**
   - Peut remplacer du code applicatif complexe
   - Syntaxe déclarative et lisible

4. **Optimisation**
   - MongoDB optimise automatiquement les pipelines
   - Utilise les index quand c'est possible

5. **Composabilité**
   - On peut combiner de nombreuses étapes
   - Création d'analyses très sophistiquées

## Prochaines Étapes

Dans les sections suivantes, nous explorerons :

- **Le concept de pipeline en détail** (6.2)
- **Les étapes de base** ($match, $project, $group, etc.) (6.3)
- **Les étapes avancées** ($lookup, $unwind, etc.) (6.4)
- **Les opérateurs d'agrégation** (arithmétiques, chaînes, dates) (6.5)
- **L'optimisation des pipelines** (6.6)

## Résumé

Le framework d'agrégation MongoDB est un **outil essentiel** pour :
- 📊 Analyser vos données
- 🔄 Transformer vos documents
- 📈 Créer des rapports
- 🎯 Effectuer des calculs complexes

Il fonctionne comme un **pipeline** où vos documents passent par différentes **étapes de transformation**, chacune effectuant une opération spécifique.

**Point clé :** Au lieu de récupérer tous les documents et de les traiter dans votre application, vous décrivez les transformations souhaitées et MongoDB les exécute efficacement côté serveur.

---

**À retenir :**
> Le framework d'agrégation transforme MongoDB d'une simple base de stockage en un puissant moteur d'analyse de données.

Dans la prochaine section, nous approfondirons le concept de pipeline et comment structurer efficacement vos agrégations.

⏭️ [Concept de pipeline](/06-framework-agregation/02-concept-pipeline.md)
