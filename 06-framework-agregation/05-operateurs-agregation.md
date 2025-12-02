🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 6.5 Opérateurs d'Agrégation

## Introduction

Après avoir exploré les **étapes** (stages) du pipeline d'agrégation, il est temps de découvrir les **opérateurs d'agrégation**. Si les étapes sont les "verbes d'action" de votre pipeline (filtrer, grouper, trier), les opérateurs sont les "outils mathématiques et logiques" qui permettent de manipuler et transformer les valeurs au sein de ces étapes.

### Métaphore : La Cuisine

Imaginez que vous préparez un plat :
- Les **étapes** ($match, $group, $project) sont les **actions principales** : "Laver", "Couper", "Cuire"
- Les **opérateurs** sont les **techniques précises** : "Couper en dés de 1cm", "Cuire à 180°C pendant 20 minutes", "Mélanger énergiquement"

Sans les opérateurs, vos étapes ne pourraient faire que des transformations basiques. Avec les opérateurs, vous pouvez effectuer des calculs complexes, des transformations sophistiquées et des analyses approfondies.

## Différence entre Étapes et Opérateurs

### Étapes (Stages)

```javascript
{ $match: { ... } }      // UNE ÉTAPE du pipeline
{ $group: { ... } }      // UNE ÉTAPE du pipeline
{ $project: { ... } }    // UNE ÉTAPE du pipeline
```

- Organisent le flux du pipeline
- Apparaissent au niveau principal du tableau
- Commencent par `$` et sont suivies d'un objet de configuration

### Opérateurs

```javascript
{ $multiply: ["$prix", "$quantite"] }    // UN OPÉRATEUR
{ $sum: "$montant" }                     // UN OPÉRATEUR
{ $concat: ["$prenom", " ", "$nom"] }   // UN OPÉRATEUR
```

- Utilisés **à l'intérieur** des étapes
- Effectuent des calculs et transformations sur les valeurs
- Commencent aussi par `$` mais manipulent des données

### Exemple Visuel

```javascript
db.ventes.aggregate([
  // ÉTAPE 1: $match (une étape)
  {
    $match: {
      date: { $gte: ISODate("2024-01-01") }  // $gte est un opérateur de comparaison
    }
  },

  // ÉTAPE 2: $project (une étape)
  {
    $project: {
      produit: 1,
      // Les opérateurs sont utilisés ICI, dans l'étape
      montantTTC: { $multiply: ["$prixHT", 1.20] },          // $multiply = opérateur
      description: { $concat: ["Vente de ", "$produit"] }    // $concat = opérateur
    }
  },

  // ÉTAPE 3: $group (une étape)
  {
    $group: {
      _id: "$categorie",
      total: { $sum: "$montantTTC" },      // $sum = opérateur (accumulateur)
      moyenne: { $avg: "$montantTTC" }     // $avg = opérateur (accumulateur)
    }
  }
])
```

## Vue d'Ensemble des Opérateurs d'Agrégation

Les opérateurs d'agrégation MongoDB se divisent en **6 catégories principales** :

### 1. **Opérateurs Arithmétiques** 🔢
Effectuent des calculs mathématiques sur les nombres.

**Exemples :**
- `$add` - Addition
- `$subtract` - Soustraction
- `$multiply` - Multiplication
- `$divide` - Division
- `$mod` - Modulo (reste de division)
- `$pow` - Puissance
- `$sqrt` - Racine carrée
- `$abs` - Valeur absolue

### 2. **Opérateurs de Chaînes** 📝
Manipulent et transforment du texte.

**Exemples :**
- `$concat` - Concaténation de chaînes
- `$toUpper` - Conversion en majuscules
- `$toLower` - Conversion en minuscules
- `$substr` - Extraction de sous-chaîne
- `$split` - Division d'une chaîne
- `$trim` - Suppression des espaces

### 3. **Opérateurs de Dates** 📅
Extraient et manipulent des informations de date et heure.

**Exemples :**
- `$year` - Extraction de l'année
- `$month` - Extraction du mois
- `$dayOfMonth` - Extraction du jour
- `$hour` - Extraction de l'heure
- `$dateAdd` - Ajout de durée à une date
- `$dateDiff` - Différence entre deux dates

### 4. **Opérateurs de Tableaux** 📋
Travaillent avec des tableaux (arrays).

**Exemples :**
- `$size` - Taille d'un tableau
- `$arrayElemAt` - Élément à une position
- `$first` - Premier élément
- `$last` - Dernier élément
- `$filter` - Filtrer un tableau
- `$map` - Transformer chaque élément

### 5. **Opérateurs Conditionnels** 🔀
Permettent la logique conditionnelle (if/else).

**Exemples :**
- `$cond` - Condition if/else
- `$ifNull` - Valeur par défaut si null
- `$switch` - Multiple conditions (comme switch/case)

### 6. **Accumulateurs** 📊
Calculent des valeurs agrégées (utilisés principalement dans $group).

**Exemples :**
- `$sum` - Somme
- `$avg` - Moyenne
- `$min` - Minimum
- `$max` - Maximum
- `$first` - Premier document du groupe
- `$last` - Dernier document du groupe
- `$push` - Créer un tableau avec toutes les valeurs

## Tableau Récapitulatif

| Catégorie | Nombre d'opérateurs | Usage principal | Fréquence |
|-----------|---------------------|-----------------|-----------|
| **Arithmétiques** | ~15 | Calculs numériques | ⭐⭐⭐⭐⭐ Très fréquent |
| **Chaînes** | ~20 | Manipulation de texte | ⭐⭐⭐⭐ Fréquent |
| **Dates** | ~20 | Extraction et calculs de dates | ⭐⭐⭐⭐ Fréquent |
| **Tableaux** | ~20 | Manipulation de tableaux | ⭐⭐⭐ Moyen |
| **Conditionnels** | ~5 | Logique if/else | ⭐⭐⭐⭐ Fréquent |
| **Accumulateurs** | ~15 | Agrégations dans $group | ⭐⭐⭐⭐⭐ Très fréquent |

## Où Utilise-t-on les Opérateurs ?

Les opérateurs d'agrégation peuvent être utilisés dans plusieurs étapes du pipeline :

### 1. Dans $project / $addFields / $set

**Pour :** Créer de nouveaux champs calculés

```javascript
{
  $project: {
    nom: 1,
    // Calcul du prix TTC
    prixTTC: { $multiply: ["$prixHT", 1.20] },
    // Nom complet
    nomComplet: { $concat: ["$prenom", " ", "$nom"] },
    // Année de naissance
    anneeNaissance: { $year: "$dateNaissance" }
  }
}
```

### 2. Dans $group

**Pour :** Calculer des agrégations

```javascript
{
  $group: {
    _id: "$categorie",
    totalVentes: { $sum: "$montant" },        // Accumulateur
    moyenneVentes: { $avg: "$montant" },      // Accumulateur
    maxVente: { $max: "$montant" },           // Accumulateur
    produitsVendus: { $push: "$produit" }     // Accumulateur
  }
}
```

### 3. Dans $match (avec $expr)

**Pour :** Filtrer avec des expressions calculées

```javascript
{
  $match: {
    $expr: {
      // Filtrer où le montant total > 1000
      $gt: [
        { $multiply: ["$prix", "$quantite"] },
        1000
      ]
    }
  }
}
```

### 4. Dans $sort (avec $meta)

**Pour :** Trier sur des valeurs calculées

```javascript
{
  $addFields: {
    scoreTotal: { $add: ["$scoreA", "$scoreB"] }
  }
},
{
  $sort: { scoreTotal: -1 }
}
```

### 5. Dans $bucket / $bucketAuto

**Pour :** Définir des limites de catégories

```javascript
{
  $bucket: {
    groupBy: { $year: "$dateCommande" },  // Opérateur dans groupBy
    boundaries: [2020, 2021, 2022, 2023, 2024],
    default: "Autre"
  }
}
```

## Exemples par Catégorie

### Opérateurs Arithmétiques 🔢

**Collection : produits**
```javascript
{
  "_id": 1,
  "nom": "Ordinateur",
  "prixHT": 1000,
  "quantiteStock": 50,
  "remise": 10  // en pourcentage
}
```

**Pipeline avec opérateurs arithmétiques :**
```javascript
db.produits.aggregate([
  {
    $project: {
      nom: 1,
      prixHT: 1,
      // Calcul du prix après remise
      prixAvecRemise: {
        $subtract: [
          "$prixHT",
          { $multiply: ["$prixHT", { $divide: ["$remise", 100] }] }
        ]
      },
      // Calcul du prix TTC (TVA 20%)
      prixTTC: { $multiply: ["$prixHT", 1.20] },
      // Valeur du stock
      valeurStock: { $multiply: ["$prixHT", "$quantiteStock"] }
    }
  }
])
```

**Résultat :**
```javascript
{
  "_id": 1,
  "nom": "Ordinateur",
  "prixHT": 1000,
  "prixAvecRemise": 900,        // 1000 - (1000 * 0.10)
  "prixTTC": 1200,              // 1000 * 1.20
  "valeurStock": 50000          // 1000 * 50
}
```

### Opérateurs de Chaînes 📝

**Collection : utilisateurs**
```javascript
{
  "_id": 1,
  "prenom": "alice",
  "nom": "DUPONT",
  "email": "alice.dupont@example.com"
}
```

**Pipeline avec opérateurs de chaînes :**
```javascript
db.utilisateurs.aggregate([
  {
    $project: {
      // Prénom avec première lettre en majuscule
      prenomFormate: {
        $concat: [
          { $toUpper: { $substr: ["$prenom", 0, 1] } },
          { $toLower: { $substr: ["$prenom", 1, -1] } }
        ]
      },
      // Nom en majuscules
      nomFormate: { $toUpper: "$nom" },
      // Nom complet
      nomComplet: {
        $concat: [
          { $toUpper: { $substr: ["$prenom", 0, 1] } },
          { $toLower: { $substr: ["$prenom", 1, -1] } },
          " ",
          { $toUpper: "$nom" }
        ]
      },
      // Domaine de l'email
      domaine: {
        $arrayElemAt: [
          { $split: ["$email", "@"] },
          1
        ]
      }
    }
  }
])
```

**Résultat :**
```javascript
{
  "_id": 1,
  "prenomFormate": "Alice",
  "nomFormate": "DUPONT",
  "nomComplet": "Alice DUPONT",
  "domaine": "example.com"
}
```

### Opérateurs de Dates 📅

**Collection : commandes**
```javascript
{
  "_id": 1,
  "numero": "CMD-001",
  "dateCommande": ISODate("2024-03-15T14:30:00Z"),
  "dateLivraison": ISODate("2024-03-20T10:00:00Z")
}
```

**Pipeline avec opérateurs de dates :**
```javascript
db.commandes.aggregate([
  {
    $project: {
      numero: 1,
      dateCommande: 1,
      // Extraction de composants de date
      annee: { $year: "$dateCommande" },
      mois: { $month: "$dateCommande" },
      jour: { $dayOfMonth: "$dateCommande" },
      jourSemaine: { $dayOfWeek: "$dateCommande" },
      // Délai de livraison en jours
      delaiLivraison: {
        $dateDiff: {
          startDate: "$dateCommande",
          endDate: "$dateLivraison",
          unit: "day"
        }
      },
      // Formatage de la date
      dateFormatee: {
        $dateToString: {
          format: "%d/%m/%Y",
          date: "$dateCommande"
        }
      }
    }
  }
])
```

**Résultat :**
```javascript
{
  "_id": 1,
  "numero": "CMD-001",
  "dateCommande": ISODate("2024-03-15T14:30:00Z"),
  "annee": 2024,
  "mois": 3,
  "jour": 15,
  "jourSemaine": 6,  // Vendredi
  "delaiLivraison": 5,
  "dateFormatee": "15/03/2024"
}
```

### Opérateurs de Tableaux 📋

**Collection : etudiants**
```javascript
{
  "_id": 1,
  "nom": "Alice",
  "notes": [15, 18, 12, 16, 14],
  "matieres": ["Math", "Physique", "Chimie"]
}
```

**Pipeline avec opérateurs de tableaux :**
```javascript
db.etudiants.aggregate([
  {
    $project: {
      nom: 1,
      notes: 1,
      // Nombre de notes
      nombreNotes: { $size: "$notes" },
      // Première et dernière note
      premiereNote: { $first: "$notes" },
      derniereNote: { $last: "$notes" },
      // Meilleure note
      meilleureNote: { $max: "$notes" },
      // Notes supérieures à 14
      bonnesNotes: {
        $filter: {
          input: "$notes",
          as: "note",
          cond: { $gte: ["$$note", 14] }
        }
      },
      // Nombre de matières
      nombreMatieres: { $size: "$matieres" }
    }
  }
])
```

**Résultat :**
```javascript
{
  "_id": 1,
  "nom": "Alice",
  "notes": [15, 18, 12, 16, 14],
  "nombreNotes": 5,
  "premiereNote": 15,
  "derniereNote": 14,
  "meilleureNote": 18,
  "bonnesNotes": [15, 18, 16, 14],
  "nombreMatieres": 3
}
```

### Opérateurs Conditionnels 🔀

**Collection : produits**
```javascript
{
  "_id": 1,
  "nom": "Ordinateur",
  "stock": 5,
  "prix": 1200
}
```

**Pipeline avec opérateurs conditionnels :**
```javascript
db.produits.aggregate([
  {
    $project: {
      nom: 1,
      stock: 1,
      prix: 1,
      // Statut basé sur le stock
      statut: {
        $cond: {
          if: { $eq: ["$stock", 0] },
          then: "Rupture",
          else: {
            $cond: {
              if: { $lte: ["$stock", 10] },
              then: "Stock faible",
              else: "Disponible"
            }
          }
        }
      },
      // Catégorie de prix
      categoriePrix: {
        $switch: {
          branches: [
            { case: { $lt: ["$prix", 100] }, then: "Économique" },
            { case: { $lt: ["$prix", 500] }, then: "Moyen" },
            { case: { $lt: ["$prix", 1000] }, then: "Premium" }
          ],
          default: "Luxe"
        }
      },
      // Message avec gestion du null
      message: {
        $ifNull: ["$promotion", "Pas de promotion en cours"]
      }
    }
  }
])
```

**Résultat :**
```javascript
{
  "_id": 1,
  "nom": "Ordinateur",
  "stock": 5,
  "prix": 1200,
  "statut": "Stock faible",
  "categoriePrix": "Luxe",
  "message": "Pas de promotion en cours"
}
```

### Accumulateurs 📊

**Collection : ventes**
```javascript
{ "_id": 1, "vendeur": "Alice", "produit": "A", "montant": 100, "date": "2024-01" }
{ "_id": 2, "vendeur": "Alice", "produit": "B", "montant": 150, "date": "2024-01" }
{ "_id": 3, "vendeur": "Bob", "produit": "A", "montant": 200, "date": "2024-01" }
{ "_id": 4, "vendeur": "Bob", "produit": "C", "montant": 120, "date": "2024-01" }
```

**Pipeline avec accumulateurs :**
```javascript
db.ventes.aggregate([
  {
    $group: {
      _id: "$vendeur",
      // Nombre de ventes
      nombreVentes: { $sum: 1 },
      // Total des montants
      totalVentes: { $sum: "$montant" },
      // Vente moyenne
      venteMoyenne: { $avg: "$montant" },
      // Vente minimale et maximale
      venteMin: { $min: "$montant" },
      venteMax: { $max: "$montant" },
      // Liste des produits vendus
      produitsVendus: { $push: "$produit" },
      // Produits uniques
      produitsUniques: { $addToSet: "$produit" },
      // Première et dernière vente
      premiereVente: { $first: "$montant" },
      derniereVente: { $last: "$montant" }
    }
  }
])
```

**Résultat :**
```javascript
[
  {
    "_id": "Alice",
    "nombreVentes": 2,
    "totalVentes": 250,
    "venteMoyenne": 125,
    "venteMin": 100,
    "venteMax": 150,
    "produitsVendus": ["A", "B"],
    "produitsUniques": ["A", "B"],
    "premiereVente": 100,
    "derniereVente": 150
  },
  {
    "_id": "Bob",
    "nombreVentes": 2,
    "totalVentes": 320,
    "venteMoyenne": 160,
    "venteMin": 120,
    "venteMax": 200,
    "produitsVendus": ["A", "C"],
    "produitsUniques": ["A", "C"],
    "premiereVente": 200,
    "derniereVente": 120
  }
]
```

## Combinaison d'Opérateurs

La véritable puissance vient de la **combinaison** d'opérateurs :

### Exemple 1 : Calcul Complexe

**Objectif :** Calculer une réduction progressive

```javascript
db.produits.aggregate([
  {
    $project: {
      nom: 1,
      prix: 1,
      quantite: 1,
      // Si quantité >= 10, réduction de 15%
      // Si quantité >= 5, réduction de 10%
      // Sinon, pas de réduction
      prixFinal: {
        $multiply: [
          "$prix",
          {
            $cond: {
              if: { $gte: ["$quantite", 10] },
              then: 0.85,  // -15%
              else: {
                $cond: {
                  if: { $gte: ["$quantite", 5] },
                  then: 0.90,  // -10%
                  else: 1.00   // Prix normal
                }
              }
            }
          }
        ]
      }
    }
  }
])
```

### Exemple 2 : Formatage Complexe

**Objectif :** Créer un libellé descriptif

```javascript
db.commandes.aggregate([
  {
    $project: {
      libelle: {
        $concat: [
          "Commande #",
          { $toString: "$numero" },
          " du ",
          { $dateToString: { format: "%d/%m/%Y", date: "$date" } },
          " - Montant: ",
          { $toString: { $round: ["$montant", 2] } },
          "€ (",
          {
            $cond: {
              if: { $eq: ["$statut", "payé"] },
              then: "PAYÉE",
              else: "EN ATTENTE"
            }
          },
          ")"
        ]
      }
    }
  }
])
```

**Résultat :**
```javascript
{
  "libelle": "Commande #1234 du 15/03/2024 - Montant: 250.50€ (PAYÉE)"
}
```

### Exemple 3 : Analyse Multi-critères

**Objectif :** Score de qualité d'un produit

```javascript
db.produits.aggregate([
  {
    $project: {
      nom: 1,
      // Score basé sur plusieurs critères
      scoreQualite: {
        $add: [
          // Note client (40% du score)
          { $multiply: [{ $ifNull: ["$noteClient", 0] }, 0.4] },
          // Disponibilité (30% du score)
          {
            $multiply: [
              {
                $cond: {
                  if: { $gt: ["$stock", 0] },
                  then: 10,
                  else: 0
                }
              },
              0.3
            ]
          },
          // Ancienneté du produit (30% du score)
          {
            $multiply: [
              {
                $subtract: [
                  10,
                  { $divide: [{ $dateDiff: { startDate: "$dateAjout", endDate: "$$NOW", unit: "day" } }, 36.5] }
                ]
              },
              0.3
            ]
          }
        ]
      }
    }
  }
])
```

## Syntaxe Générale des Opérateurs

### Format de Base

```javascript
{ $operateur: valeur }
```

### Opérateurs avec Arguments Multiples

```javascript
{ $operateur: [argument1, argument2, ...] }
```

### Opérateurs avec Options

```javascript
{
  $operateur: {
    option1: valeur1,
    option2: valeur2
  }
}
```

### Référence aux Champs

Pour référencer un champ du document, utilisez le préfixe `$` :

```javascript
"$nomDuChamp"           // Référence un champ
"$$variable"            // Référence une variable définie
{ $literal: "$texte" }  // Chaîne littérale contenant $
```

## Contexte d'Utilisation : $$ vs $

### $ - Référence aux Champs du Document

```javascript
{
  $project: {
    prix: "$prixUnitaire",        // Référence le champ prixUnitaire
    total: { $multiply: ["$prix", "$quantite"] }
  }
}
```

### $$ - Référence aux Variables

Utilisé avec `$map`, `$filter`, `$reduce`, etc.

```javascript
{
  $project: {
    notesAjustees: {
      $map: {
        input: "$notes",
        as: "note",
        in: { $add: ["$$note", 1] }  // $$note référence la variable "note"
      }
    }
  }
}
```

### $$NOW, $$ROOT, etc. - Variables Système

```javascript
{
  $addFields: {
    dateTraitement: "$$NOW",      // Date actuelle
    documentComplet: "$$ROOT"     // Document complet
  }
}
```

## Opérateurs les Plus Utilisés

### Top 10 des Opérateurs par Catégorie

#### Arithmétiques
1. `$multiply` - Multiplication
2. `$add` - Addition
3. `$subtract` - Soustraction
4. `$divide` - Division

#### Chaînes
1. `$concat` - Concaténation
2. `$toUpper` / `$toLower` - Conversion casse
3. `$substr` - Extraction

#### Dates
1. `$dateToString` - Formatage
2. `$year`, `$month`, `$dayOfMonth` - Extraction
3. `$dateDiff` - Différence

#### Tableaux
1. `$size` - Taille
2. `$first` / `$last` - Premier/Dernier
3. `$filter` - Filtrage

#### Conditionnels
1. `$cond` - If/Else
2. `$ifNull` - Gestion du null
3. `$switch` - Multi-conditions

#### Accumulateurs
1. `$sum` - Somme
2. `$avg` - Moyenne
3. `$max` / `$min` - Max/Min
4. `$push` - Créer tableau

## Erreurs Courantes avec les Opérateurs

### 1. Oublier les Crochets pour les Arguments

```javascript
// ❌ ERREUR
{ $multiply: "$prix", "$quantite" }  // Syntaxe incorrecte

// ✅ CORRECT
{ $multiply: ["$prix", "$quantite"] }  // Arguments dans un tableau
```

### 2. Oublier le $ pour Référencer un Champ

```javascript
// ❌ ERREUR
{ $sum: "montant" }  // "montant" est une chaîne littérale

// ✅ CORRECT
{ $sum: "$montant" }  // Référence le champ montant
```

### 3. Mauvais Type de Données

```javascript
// ❌ ERREUR - Multiplier une chaîne
{
  $multiply: ["$prixTexte", 2]  // Si prixTexte est "100€"
}

// ✅ CORRECT - Convertir d'abord
{
  $multiply: [
    { $toDouble: { $trim: { input: { $substr: ["$prixTexte", 0, -1] } } } },
    2
  ]
}
```

### 4. Division par Zéro

```javascript
// ❌ RISQUE - Division par zéro
{ $divide: ["$total", "$quantite"] }  // Si quantite = 0

// ✅ CORRECT - Gérer le cas
{
  $cond: {
    if: { $eq: ["$quantite", 0] },
    then: 0,
    else: { $divide: ["$total", "$quantite"] }
  }
}
```

### 5. Null dans les Calculs

```javascript
// ❌ PROBLÈME - Null dans calcul
{ $add: ["$valeur1", "$valeur2"] }  // Si valeur1 ou valeur2 est null

// ✅ CORRECT - Gérer les null
{
  $add: [
    { $ifNull: ["$valeur1", 0] },
    { $ifNull: ["$valeur2", 0] }
  ]
}
```

## Cas d'Usage Réels

### 1. E-commerce : Calcul de Panier

```javascript
db.paniers.aggregate([
  { $unwind: "$articles" },
  {
    $addFields: {
      "articles.sousTotal": {
        $multiply: [
          "$articles.prix",
          "$articles.quantite"
        ]
      },
      "articles.remiseAppliquee": {
        $multiply: [
          "$articles.prix",
          "$articles.quantite",
          { $divide: ["$articles.pourcentageRemise", 100] }
        ]
      }
    }
  },
  {
    $group: {
      _id: "$_id",
      totalHT: { $sum: "$articles.sousTotal" },
      totalRemises: { $sum: "$articles.remiseAppliquee" },
      nbArticles: { $sum: "$articles.quantite" }
    }
  },
  {
    $addFields: {
      totalAvecRemise: { $subtract: ["$totalHT", "$totalRemises"] },
      totalTTC: {
        $multiply: [
          { $subtract: ["$totalHT", "$totalRemises"] },
          1.20
        ]
      }
    }
  }
])
```

### 2. Analytics : Rapport de Performance

```javascript
db.ventes.aggregate([
  {
    $group: {
      _id: {
        annee: { $year: "$date" },
        mois: { $month: "$date" }
      },
      ventesTotal: { $sum: "$montant" },
      nbTransactions: { $sum: 1 },
      clientsUniques: { $addToSet: "$clientId" }
    }
  },
  {
    $addFields: {
      venteMoyenne: { $divide: ["$ventesTotal", "$nbTransactions"] },
      nbClientsUniques: { $size: "$clientsUniques" },
      moisFormate: {
        $concat: [
          { $toString: "$_id.annee" },
          "-",
          {
            $cond: {
              if: { $lt: ["$_id.mois", 10] },
              then: { $concat: ["0", { $toString: "$_id.mois" }] },
              else: { $toString: "$_id.mois" }
            }
          }
        ]
      }
    }
  }
])
```

### 3. RH : Analyse de Présence

```javascript
db.pointages.aggregate([
  {
    $addFields: {
      heuresTravaillees: {
        $divide: [
          {
            $subtract: [
              { $toDate: "$heureSortie" },
              { $toDate: "$heureEntree" }
            ]
          },
          3600000  // Convertir millisecondes en heures
        ]
      },
      retard: {
        $cond: {
          if: {
            $gt: [
              { $hour: "$heureEntree" },
              9  // Heure d'arrivée normale: 9h
            ]
          },
          then: {
            $subtract: [
              { $hour: "$heureEntree" },
              9
            ]
          },
          else: 0
        }
      }
    }
  },
  {
    $group: {
      _id: "$employeId",
      totalHeures: { $sum: "$heuresTravaillees" },
      joursPresents: { $sum: 1 },
      totalRetards: { $sum: "$retard" },
      moyenneHeuresParJour: { $avg: "$heuresTravaillees" }
    }
  }
])
```

## Récapitulatif

Les **opérateurs d'agrégation** sont les outils qui donnent toute sa puissance au framework d'agrégation MongoDB :

### Points Clés

1. **Les opérateurs sont utilisés DANS les étapes**
   - Pas au même niveau que $match, $group, etc.
   - Effectuent des calculs et transformations

2. **6 catégories principales**
   - Arithmétiques : calculs numériques
   - Chaînes : manipulation de texte
   - Dates : travail avec les dates
   - Tableaux : manipulation de tableaux
   - Conditionnels : logique if/else
   - Accumulateurs : agrégations statistiques

3. **Référencement des champs**
   - `$` pour les champs du document
   - `$$` pour les variables
   - Crochets `[]` pour les arguments multiples

4. **Combinaison d'opérateurs**
   - Les opérateurs peuvent être imbriqués
   - Créer des expressions complexes
   - Attention à la lisibilité

5. **Gestion des erreurs**
   - Vérifier les null avec $ifNull
   - Éviter les divisions par zéro
   - Valider les types de données

## Prochaines Étapes

Dans les sections suivantes, nous explorerons en détail chaque catégorie d'opérateurs :

- **6.5.1** - Opérateurs arithmétiques : Tous les calculs mathématiques
- **6.5.2** - Opérateurs de chaînes : Manipulation de texte avancée
- **6.5.3** - Opérateurs de dates : Extraction et calculs temporels
- **6.5.4** - Opérateurs de tableaux : Transformation et analyse de tableaux
- **6.5.5** - Opérateurs conditionnels : Logique if/else sophistiquée
- **6.5.6** - Accumulateurs : Agrégations statistiques complètes

Chaque section détaillera tous les opérateurs disponibles avec leurs syntaxes, options, cas d'usage et exemples pratiques.

---

**À retenir :**
> Les opérateurs d'agrégation sont comme les fonctions d'un tableur Excel : individuellement simples, mais combinés ils permettent de réaliser des analyses et transformations extrêmement sophistiquées.

Préparez-vous à découvrir plus de 90 opérateurs différents qui transformeront votre façon d'analyser les données !

⏭️ [Opérateurs arithmétiques](/06-framework-agregation/05.1-operateurs-arithmetiques.md)
