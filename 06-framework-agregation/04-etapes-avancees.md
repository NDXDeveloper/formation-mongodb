🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.4 Étapes Avancées

## Introduction

Après avoir maîtrisé les **étapes de base** qui permettent de filtrer, trier, regrouper et projeter vos données, il est temps de découvrir les **étapes avancées** du framework d'agrégation. Ces étapes ouvrent la porte à des transformations beaucoup plus sophistiquées et puissantes.

Si les étapes de base sont comme les outils fondamentaux d'un menuisier (scie, marteau, rabot), les étapes avancées sont comme les machines électriques spécialisées qui permettent de réaliser des travaux complexes avec précision et efficacité.

## Différence entre Étapes de Base et Avancées

### Étapes de Base
- ✅ Operations simples et directes
- ✅ Utilisées dans 90% des pipelines
- ✅ Faciles à comprendre et à utiliser
- ✅ Performances prévisibles
- **Exemples :** $match, $group, $sort, $limit

### Étapes Avancées
- 🚀 Operations complexes et sophistiquées
- 🚀 Utilisées pour des cas spécifiques
- 🚀 Requièrent une compréhension plus approfondie
- 🚀 Peuvent impacter les performances si mal utilisées
- **Exemples :** $lookup, $facet, $graphLookup, $unwind

## Vue d'Ensemble des Étapes Avancées

Dans cette section, nous allons explorer **11 étapes avancées** qui vous permettront de :

### 1. **$lookup** - Jointures entre Collections
Combiner des données de plusieurs collections (équivalent du JOIN en SQL).

### 2. **$unwind** - Déplier les Tableaux
Transformer un document avec un tableau en plusieurs documents.

### 3. **$addFields / $set** - Ajouter des Champs
Enrichir vos documents avec de nouveaux champs calculés.

### 4. **$replaceRoot / $replaceWith** - Changer la Racine
Promouvoir un sous-document en document principal.

### 5. **$facet** - Analyses Multiples Parallèles
Exécuter plusieurs pipelines en parallèle sur les mêmes données.

### 6. **$bucket / $bucketAuto** - Catégorisation
Créer des catégories (buckets) pour regrouper les données.

### 7. **$graphLookup** - Jointures Récursives
Explorer des structures hiérarchiques ou graphes de données.

### 8. **$merge / $out** - Sauvegarder les Résultats
Écrire les résultats du pipeline dans une collection.

### 9. **$redact** - Filtrage Conditionnel Avancé
Filtrer ou transformer des documents selon des conditions complexes.

### 10. **$sample** - Échantillonnage Aléatoire
Obtenir un échantillon aléatoire de documents.

### 11. **$unionWith** - Union de Collections
Combiner les résultats de plusieurs collections.

## Tableau Récapitulatif

| Étape | Fonction Principale | Difficulté | Cas d'Usage Typique |
|-------|---------------------|------------|---------------------|
| **$lookup** | Jointure entre collections | ⭐⭐⭐ | Relations entre données |
| **$unwind** | Déplie les tableaux | ⭐⭐ | Normalisation de tableaux |
| **$addFields/$set** | Ajoute des champs | ⭐⭐ | Enrichissement de données |
| **$replaceRoot** | Change la racine | ⭐⭐ | Restructuration |
| **$facet** | Analyses parallèles | ⭐⭐⭐⭐ | Tableaux de bord complexes |
| **$bucket** | Catégorisation | ⭐⭐⭐ | Histogrammes, groupes |
| **$graphLookup** | Jointure récursive | ⭐⭐⭐⭐ | Hiérarchies, graphes |
| **$merge/$out** | Sauvegarde résultats | ⭐⭐ | Matérialisation de vues |
| **$redact** | Filtrage complexe | ⭐⭐⭐ | Contrôle d'accès |
| **$sample** | Échantillonnage | ⭐ | Tests, aperçus |
| **$unionWith** | Union de collections | ⭐⭐ | Agrégation multi-sources |

## Catégorisation des Étapes Avancées

### 📊 Transformation de Structure

#### **$unwind** - Déplier les Tableaux
Transforme chaque élément d'un tableau en un document séparé.

**Avant :**
```javascript
{ "_id": 1, "nom": "Alice", "hobbies": ["lecture", "sport", "musique"] }
```

**Après $unwind sur hobbies :**
```javascript
{ "_id": 1, "nom": "Alice", "hobbies": "lecture" }
{ "_id": 1, "nom": "Alice", "hobbies": "sport" }
{ "_id": 1, "nom": "Alice", "hobbies": "musique" }
```

#### **$replaceRoot** - Changer la Racine du Document
Permet de promouvoir un sous-document au niveau principal.

**Avant :**
```javascript
{
  "_id": 1,
  "utilisateur": {
    "nom": "Alice",
    "age": 30
  }
}
```

**Après $replaceRoot :**
```javascript
{ "nom": "Alice", "age": 30 }
```

### 🔗 Jointures et Relations

#### **$lookup** - Jointures entre Collections
L'équivalent du JOIN en SQL, permet de combiner des données de plusieurs collections.

**Collection 1 : commandes**
```javascript
{ "_id": 1, "clientId": 101, "montant": 250 }
```

**Collection 2 : clients**
```javascript
{ "_id": 101, "nom": "Alice", "ville": "Paris" }
```

**Après $lookup :**
```javascript
{
  "_id": 1,
  "clientId": 101,
  "montant": 250,
  "infoClient": { "_id": 101, "nom": "Alice", "ville": "Paris" }
}
```

#### **$graphLookup** - Jointures Récursives
Permet d'explorer des structures hiérarchiques (organigrammes, catégories, etc.).

**Exemple :** Trouver tous les employés sous un manager, incluant les sous-managers.

### ➕ Enrichissement de Données

#### **$addFields / $set** - Ajouter des Champs Calculés
Ajoute de nouveaux champs sans modifier les champs existants.

**Avant :**
```javascript
{ "prixHT": 100, "quantite": 5 }
```

**Avec $addFields :**
```javascript
{
  "prixHT": 100,
  "quantite": 5,
  "prixTTC": 120,
  "total": 600
}
```

### 📈 Analyses Complexes

#### **$facet** - Analyses Multiples en Parallèle
Exécute plusieurs pipelines simultanément sur les mêmes données.

**Exemple :** En une seule requête, obtenir :
- Les statistiques globales
- Le top 10 des produits
- La distribution par catégorie

#### **$bucket / $bucketAuto** - Catégorisation Automatique
Regroupe les documents dans des catégories (buckets) selon des valeurs.

**Exemple :** Répartir les produits par tranches de prix :
- 0-50€
- 50-100€
- 100-200€
- 200€+

### 💾 Sauvegarde de Résultats

#### **$merge / $out** - Écrire dans une Collection
Sauvegarde les résultats du pipeline dans une nouvelle collection ou met à jour une collection existante.

**Cas d'usage :**
- Créer des vues matérialisées
- Sauvegarder des rapports précalculés
- ETL (Extract, Transform, Load)

### 🎲 Échantillonnage et Union

#### **$sample** - Échantillon Aléatoire
Obtient un nombre aléatoire de documents.

**Exemple :** Obtenir 10 produits aléatoires pour afficher sur la page d'accueil.

#### **$unionWith** - Union de Collections
Combine les résultats de plusieurs collections dans un seul pipeline.

**Exemple :** Analyser simultanément les commandes de 2024 et 2023.

### 🔒 Filtrage Avancé

#### **$redact** - Filtrage Conditionnel Complexe
Filtre ou transforme des documents selon des règles complexes.

**Cas d'usage :** Contrôle d'accès basé sur les rôles (afficher différents champs selon l'utilisateur).

## Quand Utiliser les Étapes Avancées ?

### ✅ Utilisez $lookup Quand :
- Vous devez combiner des données de plusieurs collections
- Vous avez des relations référencées (comme des foreign keys en SQL)
- Vous voulez enrichir vos documents avec des informations externes

### ✅ Utilisez $unwind Quand :
- Vos documents contiennent des tableaux
- Vous voulez analyser chaque élément d'un tableau individuellement
- Vous préparez des données pour un $group

### ✅ Utilisez $addFields Quand :
- Vous voulez ajouter des champs calculés
- Vous gardez tous les champs existants
- Vous enrichissez progressivement vos documents

### ✅ Utilisez $facet Quand :
- Vous avez besoin de plusieurs analyses en une seule requête
- Vous construisez un tableau de bord complexe
- Vous voulez optimiser les performances en évitant plusieurs requêtes

### ✅ Utilisez $bucket Quand :
- Vous créez des histogrammes
- Vous catégorisez des valeurs numériques (âges, prix, scores)
- Vous faites des analyses de distribution

### ✅ Utilisez $graphLookup Quand :
- Vous travaillez avec des structures hiérarchiques
- Vous explorez des relations récursives (manager → employés → sous-employés)
- Vous analysez des graphes de données

### ✅ Utilisez $merge/$out Quand :
- Vous voulez sauvegarder les résultats d'agrégation
- Vous créez des rapports précalculés
- Vous implémentez un processus ETL

## Exemple Progressif : E-commerce Complet

Prenons un exemple concret d'un système e-commerce avec plusieurs collections.

### Collections de Base

**Collection : commandes**
```javascript
{
  "_id": 1,
  "clientId": 101,
  "articles": [
    { "produitId": 501, "quantite": 2, "prixUnitaire": 50 },
    { "produitId": 502, "quantite": 1, "prixUnitaire": 100 }
  ],
  "date": ISODate("2024-01-15"),
  "statut": "livrée"
}
```

**Collection : clients**
```javascript
{
  "_id": 101,
  "nom": "Alice Dupont",
  "email": "alice@example.com",
  "ville": "Paris",
  "segment": "premium"
}
```

**Collection : produits**
```javascript
{
  "_id": 501,
  "nom": "Clavier",
  "categorie": "Informatique",
  "marque": "TechBrand"
}
```

### Niveau 1 : Utiliser $lookup (Jointure Simple)

**Objectif :** Enrichir les commandes avec les infos clients

```javascript
db.commandes.aggregate([
  {
    $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "infoClient"
    }
  }
])
```

### Niveau 2 : Combiner $unwind et $lookup

**Objectif :** Analyser chaque article avec les infos produit

```javascript
db.commandes.aggregate([
  // Déplier les articles
  { $unwind: "$articles" },

  // Joindre avec les produits
  {
    $lookup: {
      from: "produits",
      localField: "articles.produitId",
      foreignField: "_id",
      as: "infoProduit"
    }
  }
])
```

### Niveau 3 : Utiliser $addFields pour Calculer

**Objectif :** Ajouter le montant total par article

```javascript
db.commandes.aggregate([
  { $unwind: "$articles" },
  {
    $addFields: {
      "articles.montantTotal": {
        $multiply: ["$articles.quantite", "$articles.prixUnitaire"]
      }
    }
  }
])
```

### Niveau 4 : Utiliser $facet pour Analyses Multiples

**Objectif :** En une requête, obtenir statistiques globales ET top produits

```javascript
db.commandes.aggregate([
  { $unwind: "$articles" },
  {
    $facet: {
      // Pipeline 1: Statistiques globales
      "statistiques": [
        {
          $group: {
            _id: null,
            totalCommandes: { $sum: 1 },
            montantTotal: { $sum: "$articles.montantTotal" }
          }
        }
      ],

      // Pipeline 2: Top 5 produits
      "topProduits": [
        {
          $group: {
            _id: "$articles.produitId",
            quantiteVendue: { $sum: "$articles.quantite" }
          }
        },
        { $sort: { quantiteVendue: -1 } },
        { $limit: 5 }
      ]
    }
  }
])
```

### Niveau 5 : Utiliser $bucket pour Catégoriser

**Objectif :** Répartir les commandes par tranches de montant

```javascript
db.commandes.aggregate([
  { $unwind: "$articles" },
  {
    $group: {
      _id: "$_id",
      montantTotal: {
        $sum: { $multiply: ["$articles.quantite", "$articles.prixUnitaire"] }
      }
    }
  },
  {
    $bucket: {
      groupBy: "$montantTotal",
      boundaries: [0, 100, 250, 500, 1000],
      default: "1000+",
      output: {
        count: { $sum: 1 },
        commandes: { $push: "$_id" }
      }
    }
  }
])
```

## Combinaisons Puissantes d'Étapes Avancées

### Pattern 1 : $lookup + $unwind
**Pour :** Relations one-to-many avec analyse détaillée

```javascript
[
  // Jointure
  { $lookup: { from: "details", ... } },
  // Déplier le tableau résultant
  { $unwind: "$details" },
  // Analyse de chaque élément
  { $group: { ... } }
]
```

### Pattern 2 : $unwind + $group + $addFields
**Pour :** Analyse de tableaux avec calculs supplémentaires

```javascript
[
  // Déplier
  { $unwind: "$items" },
  // Regrouper
  { $group: { _id: "$category", total: { $sum: "$items.price" } } },
  // Enrichir
  { $addFields: { totalTTC: { $multiply: ["$total", 1.20] } } }
]
```

### Pattern 3 : $facet + $bucket + $lookup
**Pour :** Tableaux de bord complexes

```javascript
[
  {
    $facet: {
      "distribution": [
        { $bucket: { ... } }
      ],
      "details": [
        { $lookup: { ... } },
        { $unwind: "$details" }
      ]
    }
  }
]
```

### Pattern 4 : $graphLookup + $match
**Pour :** Navigation dans des hiérarchies avec filtres

```javascript
[
  // Explorer la hiérarchie
  { $graphLookup: { ... } },
  // Filtrer les résultats
  { $match: { "hierarchy.level": { $lte: 3 } } }
]
```

## Performance et Optimisation

### ⚠️ Attention aux Étapes Coûteuses

Certaines étapes avancées peuvent être très coûteuses en ressources :

| Étape | Coût | Raison |
|-------|------|--------|
| **$lookup** | 🔴 Élevé | Jointure = opération coûteuse |
| **$graphLookup** | 🔴🔴 Très élevé | Récursif = très coûteux |
| **$facet** | 🟡 Moyen | Plusieurs pipelines en parallèle |
| **$unwind** | 🟡 Moyen | Multiplie le nombre de documents |
| **$bucket** | 🟢 Faible | Opération simple |
| **$addFields** | 🟢 Faible | Ajout de champs uniquement |

### Conseils d'Optimisation

1. **Filtrer AVANT $lookup**
```javascript
// ✅ BON
[
  { $match: { actif: true } },      // Réduit le volume
  { $lookup: { ... } }               // Joint moins de documents
]

// ❌ MOINS BON
[
  { $lookup: { ... } },              // Joint tout
  { $match: { actif: true } }        // Filtre après
]
```

2. **Limiter AVANT $unwind si possible**
```javascript
// ✅ BON si vous voulez les 10 premiers documents
[
  { $limit: 10 },
  { $unwind: "$items" }
]

// ❌ Si vous voulez les 10 premiers items après unwind
[
  { $unwind: "$items" },
  { $limit: 10 }
]
```

3. **Indexer les champs de jointure**
```javascript
// Créer un index sur le champ utilisé dans $lookup
db.clients.createIndex({ "_id": 1 })  // Généralement déjà existant
db.commandes.createIndex({ "clientId": 1 })
```

4. **Utiliser $project pour réduire les données**
```javascript
[
  { $project: { champsNecessaires: 1 } },  // Réduit la taille
  { $lookup: { ... } }                     // Travaille avec moins de données
]
```

## Cas d'Usage Réels par Industrie

### 🛒 E-commerce
- **$lookup** : Enrichir commandes avec clients et produits
- **$facet** : Tableau de bord de ventes (stats + top produits + tendances)
- **$bucket** : Analyse de distribution des prix

### 📱 Réseau Social
- **$graphLookup** : Trouver tous les amis d'amis
- **$unwind** : Analyser les likes, commentaires
- **$sample** : Suggestions de contenu aléatoire

### 🏦 Finance
- **$lookup** : Joindre transactions avec comptes
- **$bucket** : Catégoriser transactions par montant
- **$redact** : Contrôle d'accès aux données sensibles

### 📊 Analytics / BI
- **$facet** : Rapports multi-dimensions
- **$merge** : Matérialiser des vues pour dashboards
- **$unionWith** : Combiner données de plusieurs périodes

### 🏥 Santé
- **$graphLookup** : Tracer propagation de maladies
- **$redact** : Respect de la confidentialité (RGPD/HIPAA)
- **$lookup** : Lier patients, consultations, prescriptions

## Erreurs Courantes avec les Étapes Avancées

### 1. Oublier $unwind après $lookup

```javascript
// ❌ ERREUR - Le résultat de $lookup est un tableau
db.commandes.aggregate([
  {
    $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },
  // client est un tableau [{ ... }], pas un objet !
  { $project: { "client.nom": 1 } }  // Syntaxe incorrecte
])

// ✅ CORRECT
db.commandes.aggregate([
  { $lookup: { ... } },
  { $unwind: "$client" },      // Transforme tableau en objet
  { $project: { "client.nom": 1 } }
])
```

### 2. Mauvais Ordre avec $unwind

```javascript
// ❌ ERREUR - Unwind avant de filtrer = inutile
[
  { $unwind: "$items" },           // Déplie TOUS les documents
  { $match: { "items.actif": true } }
]

// ✅ MIEUX - Filtrer d'abord si possible
[
  { $match: { "items.actif": true } },  // Si le match fonctionne
  { $unwind: "$items" }
]
```

### 3. Utiliser $out sans Comprendre

```javascript
// ⚠️ ATTENTION - $out REMPLACE la collection !
db.source.aggregate([
  { $match: { ... } },
  { $out: "destination" }  // Écrase complètement "destination"
])

// ✅ MIEUX - Utiliser $merge pour mise à jour
db.source.aggregate([
  { $match: { ... } },
  { $merge: { into: "destination", on: "_id" } }
])
```

### 4. $facet Trop Complexe

```javascript
// ❌ ÉVITER - Facet avec trop de sous-pipelines complexes
{
  $facet: {
    "pipeline1": [ /* 10 étapes */ ],
    "pipeline2": [ /* 15 étapes */ ],
    "pipeline3": [ /* 12 étapes */ ],
    // Difficile à maintenir et peut être lent
  }
}

// ✅ MIEUX - Séparer en plusieurs requêtes si trop complexe
```

## Progression d'Apprentissage Recommandée

### Niveau 1 : Commencer Simple
1. **$addFields** - Le plus simple
2. **$sample** - Pour tester et explorer
3. **$unwind** - Essentiel pour travailler avec tableaux

### Niveau 2 : Relations
4. **$lookup** - Jointures de base
5. **$replaceRoot** - Restructuration simple

### Niveau 3 : Analyses
6. **$bucket** - Catégorisation
7. **$facet** - Analyses multiples

### Niveau 4 : Avancé
8. **$merge/$out** - Sauvegarde de résultats
9. **$unionWith** - Combinaison de sources
10. **$redact** - Sécurité et filtrage complexe
11. **$graphLookup** - Le plus complexe

## Mémo Rapide : Choisir la Bonne Étape

```
Question : Que voulez-vous faire ?

├─ Combiner deux collections
│  └─> $lookup
│
├─ Travailler avec un tableau dans un document
│  └─> $unwind
│
├─ Ajouter des champs calculés
│  └─> $addFields ou $set
│
├─ Changer la structure du document
│  └─> $replaceRoot ou $replaceWith
│
├─ Faire plusieurs analyses en une fois
│  └─> $facet
│
├─ Créer des catégories/tranches
│  └─> $bucket ou $bucketAuto
│
├─ Explorer une hiérarchie
│  └─> $graphLookup
│
├─ Sauvegarder les résultats
│  └─> $merge ou $out
│
├─ Obtenir des documents aléatoires
│  └─> $sample
│
├─ Combiner plusieurs collections
│  └─> $unionWith
│
└─ Filtrage complexe conditionnel
   └─> $redact
```

## Récapitulatif

Les **étapes avancées** transforment MongoDB d'une simple base de données NoSQL en un puissant moteur d'analyse et de transformation de données.

### Points Clés à Retenir

1. **Les étapes avancées sont puissantes mais coûteuses**
   - Utilisez-les avec parcimonie
   - Optimisez toujours (filtres, index, ordre)

2. **Chaque étape a un cas d'usage spécifique**
   - $lookup pour les jointures
   - $facet pour les analyses multiples
   - $unwind pour les tableaux

3. **L'ordre est encore plus important**
   - Filtrez avant de joindre
   - Limitez avant de déplier
   - Projetez pour réduire les données

4. **Testez progressivement**
   - Construisez étape par étape
   - Vérifiez les résultats intermédiaires
   - Mesurez les performances

5. **Documentation et maintenabilité**
   - Commentez les pipelines complexes
   - Décomposez en étapes logiques
   - Documentez les décisions d'architecture

## Prochaines Étapes

Dans les sections suivantes, nous explorerons en détail chacune de ces étapes avancées :

- **6.4.1** - $lookup : Jointures entre collections
- **6.4.2** - $unwind : Déplier les tableaux
- **6.4.3** - $addFields et $set : Enrichissement
- **6.4.4** - $replaceRoot et $replaceWith : Restructuration
- **6.4.5** - $facet : Analyses parallèles
- **6.4.6** - $bucket et $bucketAuto : Catégorisation
- **6.4.7** - $graphLookup : Jointures récursives
- **6.4.8** - $merge et $out : Sauvegarde
- **6.4.9** - $redact : Filtrage conditionnel
- **6.4.10** - $sample : Échantillonnage
- **6.4.11** - $unionWith : Union de collections

Chaque section détaillera la syntaxe, les options, les cas d'usage avancés et les pièges à éviter.

---

**À retenir :**
> Les étapes avancées sont comme des pièces de puzzle sophistiquées : elles permettent de construire des systèmes d'analyse complexes et puissants, mais nécessitent une compréhension approfondie pour être utilisées efficacement.

Préparez-vous à découvrir des capacités que vous ne soupçonniez peut-être pas dans MongoDB !

⏭️ [$lookup (jointures)](/06-framework-agregation/04.01-lookup.md)
