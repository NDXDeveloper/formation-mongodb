🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.2 Concept de Pipeline

## Qu'est-ce qu'un Pipeline d'Agrégation ?

Un **pipeline d'agrégation** est au cœur du framework d'agrégation MongoDB. C'est une séquence ordonnée d'**étapes** (stages) que vos documents traversent, un peu comme une chaîne de production industrielle où chaque station effectue une opération spécifique.

### Métaphore : La Chaîne de Production

Imaginez une usine de jus de fruits :

```
🍎 Pommes brutes
     ↓
[Station 1: Lavage] - Nettoie les pommes
     ↓
[Station 2: Tri] - Ne garde que les bonnes pommes
     ↓
[Station 3: Découpe] - Coupe en morceaux
     ↓
[Station 4: Pressage] - Extrait le jus
     ↓
[Station 5: Mise en bouteille] - Emballe le produit final
     ↓
🧃 Jus de pommes prêt
```

Dans MongoDB, c'est la même chose :

```
📄 Documents bruts
     ↓
[$match] - Filtre les documents
     ↓
[$sort] - Trie les documents
     ↓
[$group] - Regroupe et calcule
     ↓
[$project] - Sélectionne les champs
     ↓
📊 Résultat final
```

## Anatomie d'un Pipeline

### Structure de Base

```javascript
db.collection.aggregate([
  // Le pipeline est un tableau d'étapes
  { étape1 },
  { étape2 },
  { étape3 }
])
```

### Structure Détaillée

```javascript
db.collection.aggregate([
  // Étape 1: Un objet avec un opérateur $
  {
    $operateur1: {
      // configuration
    }
  },

  // Étape 2: Un autre opérateur
  {
    $operateur2: {
      // configuration
    }
  },

  // Étape 3: Et ainsi de suite...
  {
    $operateur3: {
      // configuration
    }
  }
])
```

**Composants essentiels :**
1. **`aggregate()`** - La méthode qui lance le pipeline
2. **`[]`** - Un tableau qui contient les étapes
3. **`{}`** - Chaque étape est un objet
4. **`$operateur`** - Chaque étape commence par un opérateur spécial avec `$`

## Flux de Données dans un Pipeline

### Principe Fondamental

Chaque étape du pipeline :
1. **Reçoit** des documents de l'étape précédente (ou de la collection pour la première étape)
2. **Transforme** ces documents selon son opération
3. **Passe** le résultat à l'étape suivante

### Visualisation du Flux

```
Collection originale: 1000 documents
         ↓
    [$match] ────────> Filtre → 500 documents passent
         ↓
    [$sort] ─────────> Trie → 500 documents triés
         ↓
    [$limit] ────────> Limite → 10 documents conservés
         ↓
    [$project] ──────> Transforme → 10 documents formatés
         ↓
    Résultat final: 10 documents
```

### Exemple Concret

Collection de départ : **produits**
```javascript
[
  { "_id": 1, "nom": "Ordinateur", "prix": 1200, "stock": 5, "categorie": "Électronique" },
  { "_id": 2, "nom": "Souris", "prix": 25, "stock": 150, "categorie": "Accessoires" },
  { "_id": 3, "nom": "Écran", "prix": 300, "stock": 20, "categorie": "Électronique" },
  { "_id": 4, "nom": "Clavier", "prix": 75, "stock": 80, "categorie": "Accessoires" },
  { "_id": 5, "nom": "Webcam", "prix": 90, "stock": 0, "categorie": "Accessoires" }
]
```

**Pipeline :**
```javascript
db.produits.aggregate([
  // Étape 1: Garder seulement les produits en stock
  {
    $match: { stock: { $gt: 0 } }
  },

  // Étape 2: Trier par prix décroissant
  {
    $sort: { prix: -1 }
  },

  // Étape 3: Garder les 3 plus chers
  {
    $limit: 3
  },

  // Étape 4: Afficher seulement nom et prix
  {
    $project: {
      _id: 0,
      nom: 1,
      prix: 1
    }
  }
])
```

**Flux détaillé :**

```
📦 Entrée: 5 documents
     ↓
[$match: stock > 0]
     ↓
📦 4 documents (Webcam éliminée car stock = 0)
[
  { "_id": 1, "nom": "Ordinateur", "prix": 1200, "stock": 5, "categorie": "Électronique" },
  { "_id": 2, "nom": "Souris", "prix": 25, "stock": 150, "categorie": "Accessoires" },
  { "_id": 3, "nom": "Écran", "prix": 300, "stock": 20, "categorie": "Électronique" },
  { "_id": 4, "nom": "Clavier", "prix": 75, "stock": 80, "categorie": "Accessoires" }
]
     ↓
[$sort: prix décroissant]
     ↓
📦 4 documents triés
[
  { "_id": 1, "nom": "Ordinateur", "prix": 1200, ... },
  { "_id": 3, "nom": "Écran", "prix": 300, ... },
  { "_id": 4, "nom": "Clavier", "prix": 75, ... },
  { "_id": 2, "nom": "Souris", "prix": 25, ... }
]
     ↓
[$limit: 3]
     ↓
📦 3 documents conservés
[
  { "_id": 1, "nom": "Ordinateur", "prix": 1200, ... },
  { "_id": 3, "nom": "Écran", "prix": 300, ... },
  { "_id": 4, "nom": "Clavier", "prix": 75, ... }
]
     ↓
[$project: nom et prix seulement]
     ↓
📦 Résultat final: 3 documents formatés
[
  { "nom": "Ordinateur", "prix": 1200 },
  { "nom": "Écran", "prix": 300 },
  { "nom": "Clavier", "prix": 75 }
]
```

## L'Ordre des Étapes Est Important

### Exemple : Deux Ordres Différents

**Pipeline A : Filtrer puis limiter**
```javascript
db.produits.aggregate([
  { $match: { prix: { $gte: 100 } } },  // D'abord filtrer
  { $limit: 5 }                          // Puis limiter
])
// Résultat: 5 produits parmi ceux qui coûtent >= 100€
```

**Pipeline B : Limiter puis filtrer**
```javascript
db.produits.aggregate([
  { $limit: 5 },                         // D'abord limiter
  { $match: { prix: { $gte: 100 } } }   // Puis filtrer
])
// Résultat: Parmi les 5 premiers produits, ceux qui coûtent >= 100€
// (Peut donner 0, 1, 2... jusqu'à 5 résultats)
```

### Règle d'Or : Filtrer Tôt

Pour optimiser les performances, **filtrez le plus tôt possible** dans le pipeline :

```javascript
// ✅ BON - Filtre en premier
db.commandes.aggregate([
  { $match: { statut: "payé" } },      // Réduit immédiatement le volume
  { $sort: { date: -1 } },             // Trie moins de documents
  { $group: { ... } }                   // Groupe moins de documents
])

// ❌ MOINS BON - Filtre en dernier
db.commandes.aggregate([
  { $sort: { date: -1 } },             // Trie TOUS les documents
  { $group: { ... } },                  // Groupe TOUS les documents
  { $match: { statut: "payé" } }       // Filtre à la fin
])
```

## Types d'Opérations dans un Pipeline

Les étapes du pipeline peuvent être classées en plusieurs catégories :

### 1. **Filtrage et Sélection**
Réduire le nombre de documents ou sélectionner des champs.

```javascript
{ $match: { ... } }      // Filtre les documents
{ $limit: n }            // Garde les n premiers
{ $skip: n }             // Saute les n premiers
{ $project: { ... } }    // Sélectionne/transforme les champs
```

### 2. **Transformation**
Modifier la structure ou le contenu des documents.

```javascript
{ $addFields: { ... } }     // Ajoute des champs
{ $set: { ... } }           // Alias de $addFields
{ $unset: [...] }           // Supprime des champs
{ $replaceRoot: { ... } }   // Change la racine du document
```

### 3. **Regroupement et Calculs**
Agréger des données et calculer des statistiques.

```javascript
{ $group: { ... } }         // Regroupe et calcule
{ $count: "nom" }           // Compte les documents
{ $bucket: { ... } }        // Crée des buckets
```

### 4. **Tri et Réorganisation**
Ordonner les résultats.

```javascript
{ $sort: { ... } }          // Trie les documents
{ $sample: { size: n } }    // Échantillon aléatoire
```

### 5. **Jointures**
Combiner des données de plusieurs collections.

```javascript
{ $lookup: { ... } }        // Joint des collections
{ $graphLookup: { ... } }   // Jointure récursive
```

### 6. **Dépliage et Restructuration**
Travailler avec des tableaux.

```javascript
{ $unwind: "$champ" }       // Déplie un tableau
{ $facet: { ... } }         // Analyses multiples parallèles
```

## Exemple Progressif : Du Simple au Complexe

### Niveau 1 : Pipeline Simple (1 étape)

**Objectif :** Compter le nombre de commandes payées

```javascript
db.commandes.aggregate([
  { $match: { statut: "payé" } }
])
```

### Niveau 2 : Pipeline Basique (2-3 étapes)

**Objectif :** Top 5 des produits les plus chers en stock

```javascript
db.produits.aggregate([
  { $match: { stock: { $gt: 0 } } },
  { $sort: { prix: -1 } },
  { $limit: 5 }
])
```

### Niveau 3 : Pipeline Intermédiaire (4-5 étapes)

**Objectif :** Chiffre d'affaires par catégorie, trié par montant

```javascript
db.commandes.aggregate([
  // 1. Garder seulement les commandes payées
  { $match: { statut: "payé" } },

  // 2. Déplier le tableau des articles
  { $unwind: "$articles" },

  // 3. Regrouper par catégorie et calculer le total
  { $group: {
      _id: "$articles.categorie",
      chiffreAffaires: { $sum: "$articles.montant" },
      nombreVentes: { $sum: 1 }
    }
  },

  // 4. Trier par CA décroissant
  { $sort: { chiffreAffaires: -1 } },

  // 5. Renommer _id en categorie
  { $project: {
      _id: 0,
      categorie: "$_id",
      chiffreAffaires: 1,
      nombreVentes: 1
    }
  }
])
```

### Niveau 4 : Pipeline Avancé (6+ étapes)

**Objectif :** Analyse complète des ventes avec jointure client

```javascript
db.commandes.aggregate([
  // 1. Période spécifique
  { $match: {
      date: {
        $gte: ISODate("2024-01-01"),
        $lt: ISODate("2024-02-01")
      }
    }
  },

  // 2. Jointure avec la collection clients
  { $lookup: {
      from: "clients",
      localField: "clientId",
      foreignField: "_id",
      as: "client"
    }
  },

  // 3. Déplier le tableau client (tableau à objet)
  { $unwind: "$client" },

  // 4. Ajouter le montant TTC
  { $addFields: {
      montantTTC: { $multiply: ["$montantHT", 1.20] }
    }
  },

  // 5. Regrouper par région du client
  { $group: {
      _id: "$client.region",
      totalVentes: { $sum: "$montantTTC" },
      nombreCommandes: { $sum: 1 },
      panierMoyen: { $avg: "$montantTTC" }
    }
  },

  // 6. Trier par total décroissant
  { $sort: { totalVentes: -1 } },

  // 7. Formater le résultat
  { $project: {
      _id: 0,
      region: "$_id",
      totalVentes: { $round: ["$totalVentes", 2] },
      nombreCommandes: 1,
      panierMoyen: { $round: ["$panierMoyen", 2] }
    }
  }
])
```

## Bonnes Pratiques pour Construire un Pipeline

### 1. **Commencer Simple**
Construisez votre pipeline étape par étape et testez après chaque ajout.

```javascript
// Étape 1: Tester le filtre seul
db.collection.aggregate([
  { $match: { ... } }
])

// Étape 2: Ajouter le tri et tester
db.collection.aggregate([
  { $match: { ... } },
  { $sort: { ... } }
])

// Étape 3: Continuer ainsi...
```

### 2. **Filtrer Tôt**
Placez `$match` au début pour réduire rapidement le volume de données.

```javascript
// ✅ BON
db.collection.aggregate([
  { $match: { ... } },        // Filtre d'abord
  { $sort: { ... } },
  { $group: { ... } }
])
```

### 3. **Projeter Tard**
Gardez tous les champs nécessaires jusqu'à la fin, puis projetez juste avant le résultat final.

```javascript
db.collection.aggregate([
  { $match: { ... } },
  { $group: { ... } },
  { $sort: { ... } },
  { $project: { ... } }      // Projection en dernier
])
```

### 4. **Limiter Tôt Si Possible**
Si vous savez que vous ne voulez qu'un nombre limité de résultats, ajoutez `$limit` tôt.

```javascript
db.collection.aggregate([
  { $match: { ... } },
  { $sort: { ... } },
  { $limit: 10 },           // Limite après le tri
  { $lookup: { ... } }      // Moins de documents à joindre
])
```

### 5. **Utiliser des Index**
Assurez-vous que vos `$match` et `$sort` peuvent utiliser des index.

```javascript
// Si vous avez un index sur { statut: 1, date: -1 }
db.commandes.aggregate([
  { $match: { statut: "payé" } },  // Utilise l'index
  { $sort: { date: -1 } }           // Utilise l'index aussi
])
```

### 6. **Commenter Votre Code**
Pour les pipelines complexes, ajoutez des commentaires.

```javascript
db.collection.aggregate([
  // Étape 1: Filtrer les commandes du mois
  { $match: {
      date: { $gte: ISODate("2024-01-01") }
    }
  },

  // Étape 2: Regrouper par client
  { $group: {
      _id: "$clientId",
      total: { $sum: "$montant" }
    }
  },

  // Étape 3: Garder les meilleurs clients (>1000€)
  { $match: { total: { $gt: 1000 } } }
])
```

## Debugger un Pipeline

### Technique 1 : Tester Étape par Étape

```javascript
// Test de l'étape 1 seulement
db.collection.aggregate([
  { $match: { ... } }
])

// Test des étapes 1-2
db.collection.aggregate([
  { $match: { ... } },
  { $group: { ... } }
])

// Test complet
db.collection.aggregate([
  { $match: { ... } },
  { $group: { ... } },
  { $sort: { ... } }
])
```

### Technique 2 : Afficher des Résultats Intermédiaires

Ajoutez temporairement un `$limit` pour voir ce qui se passe :

```javascript
db.collection.aggregate([
  { $match: { ... } },
  { $limit: 5 },           // Voir 5 documents après le match
  // { $group: { ... } },  // Commenté temporairement
])
```

### Technique 3 : Utiliser $out pour Sauvegarder

Sauvegardez les résultats intermédiaires dans une collection temporaire :

```javascript
db.collection.aggregate([
  { $match: { ... } },
  { $group: { ... } },
  { $out: "resultats_temp" }  // Sauvegarde dans une collection
])

// Puis examiner
db.resultats_temp.find()
```

## Limitations des Pipelines

### 1. **Taille des Documents**
Chaque document dans le pipeline ne peut pas dépasser **16 Mo**.

### 2. **Mémoire**
Par défaut, chaque étape est limitée à **100 Mo** de RAM.

**Solution :** Utiliser l'option `allowDiskUse` pour les grandes agrégations :

```javascript
db.collection.aggregate(
  [ /* pipeline */ ],
  { allowDiskUse: true }
)
```

### 3. **Temps d'Exécution**
Les pipelines complexes peuvent être longs à exécuter.

**Solution :**
- Optimiser l'ordre des étapes
- Utiliser des index
- Filtrer tôt dans le pipeline

## Composition et Réutilisation

### Variables de Pipeline

Vous pouvez stocker des parties de pipeline dans des variables :

```javascript
// Étapes communes
const filtreActif = { $match: { actif: true } }
const triDate = { $sort: { date: -1 } }
const limit10 = { $limit: 10 }

// Réutilisation
db.collection1.aggregate([
  filtreActif,
  triDate,
  limit10
])

db.collection2.aggregate([
  filtreActif,
  triDate,
  limit10
])
```

### Fonctions Réutilisables

```javascript
function topNParCategorie(categorie, n) {
  return db.produits.aggregate([
    { $match: { categorie: categorie } },
    { $sort: { ventes: -1 } },
    { $limit: n }
  ])
}

// Utilisation
topNParCategorie("Électronique", 5)
topNParCategorie("Vêtements", 10)
```

## Résumé Visual

```
┌─────────────────────────────────────────────────────┐
│           PIPELINE D'AGRÉGATION                     │
│                                                     │
│  Collection initiale                                │
│         ↓                                           │
│  ┌──────────────────┐                               │
│  │  Étape 1: $match │  ← Filtre les documents       │
│  └──────────────────┘                               │
│         ↓                                           │
│  ┌──────────────────┐                               │
│  │  Étape 2: $sort  │  ← Trie les résultats         │
│  └──────────────────┘                               │
│         ↓                                           │
│  ┌──────────────────┐                               │
│  │  Étape 3: $group │  ← Regroupe et calcule        │
│  └──────────────────┘                               │
│         ↓                                           │
│  ┌──────────────────┐                               │
│  │ Étape 4: $project│  ← Formate le résultat        │
│  └──────────────────┘                               │
│         ↓                                           │
│    Résultat final                                   │
│                                                     │
│  Chaque étape transforme les données et les passe   │
│  à l'étape suivante dans l'ordre défini.            │
└─────────────────────────────────────────────────────┘
```

## Points Clés à Retenir

1. **Pipeline = Séquence d'Étapes**
   - Chaque étape transforme les données
   - Les données passent d'une étape à l'autre

2. **L'Ordre Est Crucial**
   - Changer l'ordre change le résultat
   - Optimiser en filtrant tôt

3. **Flux Unidirectionnel**
   - Les données vont toujours vers l'avant
   - Pas de retour en arrière

4. **Construction Progressive**
   - Construire étape par étape
   - Tester régulièrement

5. **Optimisation**
   - Filtrer tôt
   - Projeter tard
   - Utiliser les index

## Prochaines Étapes

Maintenant que vous comprenez le concept de pipeline, nous allons explorer dans les sections suivantes :

- **Les étapes de base** (6.3) : $match, $project, $group, $sort, etc.
- **Les étapes avancées** (6.4) : $lookup, $unwind, $facet, etc.
- **Les opérateurs d'agrégation** (6.5) : Fonctions de calcul, transformation, etc.

---

**Concept fondamental :**
> Un pipeline d'agrégation est comme une chaîne de montage où chaque station (étape) effectue une transformation spécifique, et où l'ordre des stations détermine le résultat final.

Dans la section suivante, nous découvrirons les étapes de base du pipeline et comment les utiliser concrètement.

⏭️ [Étapes de base](/06-framework-agregation/03-etapes-de-base.md)
