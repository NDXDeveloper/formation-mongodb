🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.9 Validation personnalisée avec $expr

## 📚 Vue d'ensemble

La **validation personnalisée avec `$expr`** vous permet de créer des règles de validation complexes qui vont au-delà des capacités de `$jsonSchema`. Avec `$expr`, vous pouvez comparer des champs entre eux, effectuer des calculs, et valider des règles métier sophistiquées.

---

## 🤔 Pourquoi utiliser `$expr` ?

### Les limites de `$jsonSchema`

`$jsonSchema` est excellent pour :
- ✅ Valider les types de données
- ✅ Définir des champs obligatoires
- ✅ Vérifier des formats (regex)
- ✅ Limiter les valeurs (min/max, enum)

Mais `$jsonSchema` **ne peut pas** :
- ❌ Comparer deux champs entre eux
- ❌ Effectuer des calculs
- ❌ Valider des conditions basées sur le temps actuel
- ❌ Créer des règles métier complexes

### C'est là qu'intervient `$expr`

`$expr` utilise le **langage d'agrégation** de MongoDB pour créer des validations avancées.

### Analogie

**$jsonSchema** = Formulaire avec champs prédéfinis
- Chaque champ a ses propres règles
- Pas de logique entre les champs

**$expr** = Validation avec logique personnalisée
- "Le prix de vente doit être supérieur au prix d'achat"
- "La date de fin doit être après la date de début"
- "La remise ne peut pas dépasser 50% du prix"

---

## 🎯 Syntaxe de base

### Structure

```javascript
{
  validator: {
    $expr: {
      // Expression de validation
      // Doit retourner true pour que le document soit valide
    }
  }
}
```

### Exemple simple

```javascript
db.createCollection("produits", {
  validator: {
    $expr: {
      $gte: ["$prix", 0]  // prix >= 0
    }
  }
})

// ✅ Valide
db.produits.insertOne({ nom: "Clavier", prix: 29.99 })

// ❌ Invalide
db.produits.insertOne({ nom: "Souris", prix: -10 })
```

**Comment ça marche** :
- `$gte` : opérateur "greater than or equal" (>=)
- `["$prix", 0]` : compare le champ `prix` avec 0
- `$prix` : référence au champ (notez le `$` devant)

---

## 🔗 Comparaison entre champs

### Cas d'usage classique

**Besoin** : Le prix de vente doit être supérieur au coût de fabrication.

```javascript
db.createCollection("produits", {
  validator: {
    $expr: {
      $gt: ["$prix_vente", "$cout_fabrication"]
      // prix_vente > cout_fabrication
    }
  }
})

// ✅ Valide : prix_vente (50) > cout_fabrication (30)
db.produits.insertOne({
  nom: "Chaise",
  cout_fabrication: 30,
  prix_vente: 50
})

// ❌ Invalide : prix_vente (25) < cout_fabrication (30)
db.produits.insertOne({
  nom: "Table",
  cout_fabrication: 30,
  prix_vente: 25
})
```

### Opérateurs de comparaison

| Opérateur | Signification | Exemple |
|-----------|---------------|---------|
| `$eq` | Égal | `$eq: ["$a", "$b"]` → a == b |
| `$ne` | Différent | `$ne: ["$a", "$b"]` → a != b |
| `$gt` | Supérieur strict | `$gt: ["$a", "$b"]` → a > b |
| `$gte` | Supérieur ou égal | `$gte: ["$a", "$b"]` → a >= b |
| `$lt` | Inférieur strict | `$lt: ["$a", "$b"]` → a < b |
| `$lte` | Inférieur ou égal | `$lte: ["$a", "$b"]` → a <= b |

### Exemple : Dates

```javascript
db.createCollection("evenements", {
  validator: {
    $expr: {
      $lt: ["$date_debut", "$date_fin"]
      // date_debut < date_fin
    }
  }
})

// ✅ Valide : début avant fin
db.evenements.insertOne({
  titre: "Conférence MongoDB",
  date_debut: new Date("2025-06-01"),
  date_fin: new Date("2025-06-03")
})

// ❌ Invalide : fin avant début
db.evenements.insertOne({
  titre: "Atelier",
  date_debut: new Date("2025-06-10"),
  date_fin: new Date("2025-06-08")
})
```

### Exemple : Nombres

```javascript
db.createCollection("notes", {
  validator: {
    $expr: {
      $and: [
        { $gte: ["$note", 0] },      // note >= 0
        { $lte: ["$note", 20] },     // note <= 20
        { $lte: ["$note", "$maximum"] }  // note <= maximum
      ]
    }
  }
})

// ✅ Valide
db.notes.insertOne({
  etudiant: "Dupont",
  note: 15,
  maximum: 20
})

// ❌ Invalide : note > maximum
db.notes.insertOne({
  etudiant: "Martin",
  note: 18,
  maximum: 16
})
```

---

## 🧮 Calculs et expressions arithmétiques

### Opérateurs arithmétiques

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$add` | Addition | `$add: ["$a", "$b"]` → a + b |
| `$subtract` | Soustraction | `$subtract: ["$a", "$b"]` → a - b |
| `$multiply` | Multiplication | `$multiply: ["$a", "$b"]` → a × b |
| `$divide` | Division | `$divide: ["$a", "$b"]` → a ÷ b |
| `$mod` | Modulo | `$mod: ["$a", "$b"]` → a % b |

### Exemple : Total = quantité × prix

```javascript
db.createCollection("lignes_commande", {
  validator: {
    $expr: {
      $eq: [
        "$total",
        { $multiply: ["$quantite", "$prix_unitaire"] }
      ]
      // total == quantite * prix_unitaire
    }
  }
})

// ✅ Valide : 3 × 10 = 30
db.lignes_commande.insertOne({
  produit: "Stylo",
  quantite: 3,
  prix_unitaire: 10,
  total: 30
})

// ❌ Invalide : 3 × 10 ≠ 25
db.lignes_commande.insertOne({
  produit: "Cahier",
  quantite: 3,
  prix_unitaire: 10,
  total: 25
})
```

### Exemple : Remise maximale

```javascript
db.createCollection("promotions", {
  validator: {
    $expr: {
      $lte: [
        "$remise",
        { $multiply: ["$prix_original", 0.5] }
      ]
      // remise <= prix_original * 50%
    }
  }
})

// ✅ Valide : remise (20) <= 100 * 0.5 (50)
db.promotions.insertOne({
  article: "Livre",
  prix_original: 100,
  remise: 20
})

// ❌ Invalide : remise (60) > 100 * 0.5 (50)
db.promotions.insertOne({
  article: "DVD",
  prix_original: 100,
  remise: 60
})
```

### Exemple : Marge bénéficiaire

```javascript
db.createCollection("ventes", {
  validator: {
    $expr: {
      $gte: [
        { $subtract: ["$prix_vente", "$cout"] },  // Bénéfice
        { $multiply: ["$cout", 0.2] }              // 20% du coût
      ]
      // (prix_vente - cout) >= cout * 20%
      // Marge minimale de 20%
    }
  }
})

// ✅ Valide : (60 - 40) = 20 >= 40 * 0.2 = 8
db.ventes.insertOne({
  produit: "Montre",
  cout: 40,
  prix_vente: 60
})

// ❌ Invalide : (45 - 40) = 5 < 40 * 0.2 = 8
db.ventes.insertOne({
  produit: "Bracelet",
  cout: 40,
  prix_vente: 45
})
```

---

## 📅 Validations temporelles

### Comparer avec la date actuelle

MongoDB fournit la variable `$$NOW` qui contient la date/heure actuelle.

```javascript
db.createCollection("reservations", {
  validator: {
    $expr: {
      $gte: ["$date_reservation", "$$NOW"]
      // date_reservation >= maintenant
    }
  }
})

// ✅ Valide : date dans le futur
db.reservations.insertOne({
  client: "Dupont",
  date_reservation: new Date("2025-12-31")
})

// ❌ Invalide : date dans le passé
db.reservations.insertOne({
  client: "Martin",
  date_reservation: new Date("2024-01-01")
})
```

**Note** : `$$NOW` est évalué au moment de l'insertion/modification.

### Date de naissance dans le passé

```javascript
db.createCollection("utilisateurs", {
  validator: {
    $expr: {
      $lt: ["$date_naissance", "$$NOW"]
      // date_naissance < maintenant
    }
  }
})

// ✅ Valide : né dans le passé
db.utilisateurs.insertOne({
  nom: "Dupont",
  date_naissance: new Date("1990-05-15")
})

// ❌ Invalide : date dans le futur
db.utilisateurs.insertOne({
  nom: "Martin",
  date_naissance: new Date("2030-01-01")
})
```

### Âge minimum

```javascript
db.createCollection("comptes", {
  validator: {
    $expr: {
      $lte: [
        "$date_naissance",
        {
          $subtract: [
            "$$NOW",
            { $multiply: [18, 365, 24, 60, 60, 1000] }  // 18 ans en ms
          ]
        }
      ]
      // date_naissance <= (maintenant - 18 ans)
    }
  }
})
```

**Alternative plus lisible avec `$dateSubtract`** (MongoDB 5.0+) :

```javascript
db.createCollection("comptes_v2", {
  validator: {
    $expr: {
      $lte: [
        "$date_naissance",
        {
          $dateSubtract: {
            startDate: "$$NOW",
            unit: "year",
            amount: 18
          }
        }
      ]
    }
  }
})
```

### Durée maximale

```javascript
db.createCollection("locations", {
  validator: {
    $expr: {
      $lte: [
        {
          $subtract: ["$date_fin", "$date_debut"]  // Durée en ms
        },
        { $multiply: [30, 24, 60, 60, 1000] }  // 30 jours en ms
      ]
      // (date_fin - date_debut) <= 30 jours
    }
  }
})

// ✅ Valide : 7 jours
db.locations.insertOne({
  vehicule: "Peugeot 208",
  date_debut: new Date("2025-06-01"),
  date_fin: new Date("2025-06-08")
})

// ❌ Invalide : 45 jours (> 30)
db.locations.insertOne({
  vehicule: "Renault Clio",
  date_debut: new Date("2025-06-01"),
  date_fin: new Date("2025-07-16")
})
```

---

## 🔀 Logique conditionnelle

### Opérateurs logiques

| Opérateur | Description | Exemple |
|-----------|-------------|---------|
| `$and` | ET logique | `$and: [condition1, condition2]` |
| `$or` | OU logique | `$or: [condition1, condition2]` |
| `$not` | NON logique | `$not: condition` |

### Exemple : Conditions multiples

```javascript
db.createCollection("employes", {
  validator: {
    $expr: {
      $and: [
        { $gte: ["$age", 18] },           // age >= 18
        { $lte: ["$age", 65] },           // age <= 65
        { $gt: ["$salaire", 0] },         // salaire > 0
        { $lte: ["$salaire", 200000] }    // salaire <= 200000
      ]
    }
  }
})
```

### Exemple : Validation conditionnelle avec `$cond`

```javascript
db.createCollection("produits_promo", {
  validator: {
    $expr: {
      $cond: {
        if: { $eq: ["$en_promotion", true] },
        then: {
          $and: [
            { $lt: ["$prix_promo", "$prix_normal"] },
            { $gte: ["$prix_promo", { $multiply: ["$prix_normal", 0.5] }] }
          ]
          // Si en promo : prix_promo < prix_normal ET >= 50% prix_normal
        },
        else: true  // Sinon : toujours valide
      }
    }
  }
})

// ✅ Valide : pas en promo
db.produits_promo.insertOne({
  nom: "Livre",
  prix_normal: 20,
  en_promotion: false
})

// ✅ Valide : en promo avec bon prix
db.produits_promo.insertOne({
  nom: "DVD",
  prix_normal: 30,
  prix_promo: 20,      // 20 < 30 ET 20 >= 15
  en_promotion: true
})

// ❌ Invalide : remise trop importante (> 50%)
db.produits_promo.insertOne({
  nom: "CD",
  prix_normal: 30,
  prix_promo: 10,      // 10 < 15 (50% de 30)
  en_promotion: true
})
```

### Exemple : Validation selon le type

```javascript
db.createCollection("utilisateurs_type", {
  validator: {
    $expr: {
      $cond: {
        if: { $eq: ["$type", "entreprise"] },
        then: {
          $and: [
            { $ne: ["$siret", null] },      // SIRET obligatoire
            { $ne: ["$raison_sociale", null] }
          ]
        },
        else: {
          $and: [
            { $ne: ["$nom", null] },        // Nom obligatoire
            { $ne: ["$prenom", null] }      // Prénom obligatoire
          ]
        }
      }
    }
  }
})

// ✅ Valide : entreprise avec SIRET
db.utilisateurs_type.insertOne({
  type: "entreprise",
  raison_sociale: "TechCorp SAS",
  siret: "12345678901234"
})

// ✅ Valide : particulier avec nom/prénom
db.utilisateurs_type.insertOne({
  type: "particulier",
  nom: "Dupont",
  prenom: "Jean"
})

// ❌ Invalide : entreprise sans SIRET
db.utilisateurs_type.insertOne({
  type: "entreprise",
  raison_sociale: "WebCorp"
})
```

---

## 🔍 Vérifier l'existence de champs

### Opérateur `$ifNull`

```javascript
db.createCollection("articles", {
  validator: {
    $expr: {
      $gt: [
        { $ifNull: ["$stock", 0] },  // Si stock n'existe pas, utiliser 0
        0
      ]
      // stock > 0 (ou si absent, considérer comme 0)
    }
  }
})
```

### Validation avec champs optionnels

```javascript
db.createCollection("contacts", {
  validator: {
    $expr: {
      $or: [
        { $ne: [{ $ifNull: ["$email", null] }, null] },
        { $ne: [{ $ifNull: ["$telephone", null] }, null] }
      ]
      // Au moins email OU telephone doit exister
    }
  }
})

// ✅ Valide : email présent
db.contacts.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})

// ✅ Valide : téléphone présent
db.contacts.insertOne({
  nom: "Martin",
  telephone: "0612345678"
})

// ✅ Valide : les deux présents
db.contacts.insertOne({
  nom: "Bernard",
  email: "bernard@example.com",
  telephone: "0698765432"
})

// ❌ Invalide : ni l'un ni l'autre
db.contacts.insertOne({
  nom: "Durand"
})
```

---

## 🔄 Combiner `$jsonSchema` et `$expr`

### Pourquoi combiner ?

- `$jsonSchema` : Validation des types, formats, structures
- `$expr` : Règles métier et logique complexe

**Meilleure pratique** : Utilisez les deux ensemble !

### Syntaxe de combinaison

```javascript
{
  validator: {
    $and: [
      {
        $jsonSchema: {
          // Validation de structure et types
        }
      },
      {
        $expr: {
          // Règles métier
        }
      }
    ]
  }
}
```

### Exemple complet

```javascript
db.createCollection("reservations_hotel", {
  validator: {
    $and: [
      // Validation de structure avec $jsonSchema
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["client", "date_arrivee", "date_depart", "nb_personnes", "prix_total"],
          properties: {
            client: {
              bsonType: "string",
              minLength: 2
            },
            date_arrivee: {
              bsonType: "date"
            },
            date_depart: {
              bsonType: "date"
            },
            nb_personnes: {
              bsonType: "int",
              minimum: 1,
              maximum: 10
            },
            prix_total: {
              bsonType: "decimal",
              minimum: 0
            },
            acompte: {
              bsonType: "decimal",
              minimum: 0
            }
          }
        }
      },
      // Règles métier avec $expr
      {
        $expr: {
          $and: [
            // Date d'arrivée < date de départ
            { $lt: ["$date_arrivee", "$date_depart"] },

            // Date d'arrivée >= aujourd'hui
            { $gte: ["$date_arrivee", "$$NOW"] },

            // Durée <= 30 jours
            {
              $lte: [
                { $subtract: ["$date_depart", "$date_arrivee"] },
                { $multiply: [30, 24, 60, 60, 1000] }
              ]
            },

            // Acompte <= 100% du prix total
            {
              $lte: [
                { $ifNull: ["$acompte", 0] },
                "$prix_total"
              ]
            },

            // Acompte >= 20% du prix total (si présent)
            {
              $or: [
                { $eq: [{ $ifNull: ["$acompte", null] }, null] },
                {
                  $gte: [
                    "$acompte",
                    { $multiply: ["$prix_total", 0.2] }
                  ]
                }
              ]
            }
          ]
        }
      }
    ]
  }
})

// ✅ Valide : tous les critères respectés
db.reservations_hotel.insertOne({
  client: "Jean Dupont",
  date_arrivee: new Date("2025-07-01"),
  date_depart: new Date("2025-07-08"),
  nb_personnes: 2,
  prix_total: NumberDecimal("700.00"),
  acompte: NumberDecimal("150.00")  // 21.4% du total
})

// ❌ Invalide : acompte trop faible (< 20%)
db.reservations_hotel.insertOne({
  client: "Marie Martin",
  date_arrivee: new Date("2025-07-01"),
  date_depart: new Date("2025-07-08"),
  nb_personnes: 2,
  prix_total: NumberDecimal("700.00"),
  acompte: NumberDecimal("100.00")  // 14.3% seulement
})
```

---

## 🎯 Exemples pratiques par domaine

### E-commerce : Panier

```javascript
db.createCollection("paniers", {
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["client_id", "articles", "total_calcule"],
          properties: {
            client_id: { bsonType: "objectId" },
            articles: {
              bsonType: "array",
              minItems: 1,
              items: {
                bsonType: "object",
                required: ["prix_unitaire", "quantite"],
                properties: {
                  prix_unitaire: { bsonType: "decimal", minimum: 0 },
                  quantite: { bsonType: "int", minimum: 1 }
                }
              }
            },
            total_calcule: { bsonType: "decimal", minimum: 0 },
            code_promo_reduction: { bsonType: "decimal", minimum: 0 }
          }
        }
      },
      {
        $expr: {
          $and: [
            // Le total calculé doit correspondre à la somme des articles
            {
              $eq: [
                "$total_calcule",
                {
                  $reduce: {
                    input: "$articles",
                    initialValue: 0,
                    in: {
                      $add: [
                        "$$value",
                        { $multiply: ["$$this.prix_unitaire", "$$this.quantite"] }
                      ]
                    }
                  }
                }
              ]
            },
            // La réduction ne peut pas dépasser 50% du total
            {
              $lte: [
                { $ifNull: ["$code_promo_reduction", 0] },
                { $multiply: ["$total_calcule", 0.5] }
              ]
            }
          ]
        }
      }
    ]
  }
})
```

### Finance : Transaction

```javascript
db.createCollection("transactions", {
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["type", "montant", "date", "compte_source"],
          properties: {
            type: { enum: ["debit", "credit", "virement"] },
            montant: { bsonType: "decimal", minimum: 0 },
            date: { bsonType: "date" },
            compte_source: { bsonType: "string" },
            compte_destination: { bsonType: "string" },
            frais: { bsonType: "decimal", minimum: 0 }
          }
        }
      },
      {
        $expr: {
          $and: [
            // Date <= maintenant
            { $lte: ["$date", "$$NOW"] },

            // Si virement, compte_destination obligatoire
            {
              $or: [
                { $ne: ["$type", "virement"] },
                { $ne: [{ $ifNull: ["$compte_destination", null] }, null] }
              ]
            },

            // Frais <= 10% du montant
            {
              $lte: [
                { $ifNull: ["$frais", 0] },
                { $multiply: ["$montant", 0.1] }
              ]
            },

            // Montant minimum de 1 centime
            { $gte: ["$montant", 0.01] }
          ]
        }
      }
    ]
  }
})
```

### RH : Congés

```javascript
db.createCollection("demandes_conges", {
  validator: {
    $and: [
      {
        $jsonSchema: {
          bsonType: "object",
          required: ["employe_id", "date_debut", "date_fin", "nb_jours"],
          properties: {
            employe_id: { bsonType: "objectId" },
            date_debut: { bsonType: "date" },
            date_fin: { bsonType: "date" },
            nb_jours: { bsonType: "int", minimum: 1 },
            solde_conges: { bsonType: "int", minimum: 0 }
          }
        }
      },
      {
        $expr: {
          $and: [
            // Date début < date fin
            { $lt: ["$date_debut", "$date_fin"] },

            // Date début >= aujourd'hui (ou 7 jours avant pour modifications)
            {
              $gte: [
                "$date_debut",
                {
                  $subtract: [
                    "$$NOW",
                    { $multiply: [7, 24, 60, 60, 1000] }
                  ]
                }
              ]
            },

            // Durée <= 30 jours
            {
              $lte: [
                { $subtract: ["$date_fin", "$date_debut"] },
                { $multiply: [30, 24, 60, 60, 1000] }
              ]
            },

            // nb_jours <= solde disponible (si renseigné)
            {
              $or: [
                { $eq: [{ $ifNull: ["$solde_conges", null] }, null] },
                { $lte: ["$nb_jours", "$solde_conges"] }
              ]
            }
          ]
        }
      }
    ]
  }
})
```

---

## 💡 Bonnes pratiques

### 1. Combiner $jsonSchema et $expr

```javascript
// ✅ BON : Types + règles métier
{
  $and: [
    { $jsonSchema: { /* types et structure */ } },
    { $expr: { /* règles métier */ } }
  ]
}

// ❌ ÉVITER : Tout avec $expr
{
  $expr: {
    // Validation des types avec $expr = complexe et moins performant
  }
}
```

### 2. Tester les cas limites

```javascript
// Tester :
// - Valeurs nulles
// - Champs manquants
// - Valeurs à zéro
// - Dates limites (passé/futur)
// - Calculs avec division par zéro
```

### 3. Documenter les expressions complexes

```javascript
// ✅ BON : Avec commentaires
{
  $expr: {
    $and: [
      // La remise ne peut pas dépasser 50% du prix
      {
        $lte: [
          "$remise",
          { $multiply: ["$prix", 0.5] }
        ]
      },
      // Le prix après remise doit être positif
      {
        $gt: [
          { $subtract: ["$prix", "$remise"] },
          0
        ]
      }
    ]
  }
}
```

### 4. Gérer les champs optionnels avec $ifNull

```javascript
// ✅ BON : Gestion des nulls
{
  $expr: {
    $lte: [
      { $ifNull: ["$remise", 0] },  // Si absent, 0
      { $multiply: ["$prix", 0.5] }
    ]
  }
}

// ❌ PROBLÈME : Erreur si remise absent
{
  $expr: {
    $lte: [
      "$remise",
      { $multiply: ["$prix", 0.5] }
    ]
  }
}
```

### 5. Attention aux performances

```javascript
// $expr peut être plus lent que $jsonSchema
// Utilisez $expr uniquement pour ce qui ne peut pas être fait avec $jsonSchema

// ✅ Préférer (plus rapide)
{
  $jsonSchema: {
    properties: {
      prix: { bsonType: "decimal", minimum: 0 }
    }
  }
}

// ❌ Éviter si possible
{
  $expr: {
    $gte: ["$prix", 0]
  }
}
```

---

## ⚠️ Pièges courants

### 1. Oublier le `$` devant les noms de champs

```javascript
// ❌ ERREUR : Pas de $
{
  $expr: {
    $gt: ["prix", 0]  // Compare la string "prix" avec 0 !
  }
}

// ✅ CORRECT
{
  $expr: {
    $gt: ["$prix", 0]  // Compare le champ prix avec 0
  }
}
```

### 2. Confusion entre `$$NOW` et `$NOW`

```javascript
// ✅ CORRECT : $$NOW (variable système)
{
  $expr: {
    $lt: ["$date", "$$NOW"]
  }
}

// ❌ ERREUR : $NOW (n'existe pas)
{
  $expr: {
    $lt: ["$date", "$NOW"]
  }
}
```

### 3. Division par zéro non gérée

```javascript
// ❌ DANGER : Division par zéro possible
{
  $expr: {
    $lt: [
      { $divide: ["$montant", "$nb_personnes"] },
      100
    ]
  }
}

// ✅ MIEUX : Vérifier avant
{
  $expr: {
    $and: [
      { $gt: ["$nb_personnes", 0] },  // Vérifier d'abord
      {
        $lt: [
          { $divide: ["$montant", "$nb_personnes"] },
          100
        ]
      }
    ]
  }
}
```

### 4. Comparaison de types différents

```javascript
// ❌ PROBLÈME : Comparer string avec number
// Si prix est stocké comme "100" (string)
{
  $expr: {
    $gt: ["$prix", 0]  // Ne fonctionne pas correctement
  }
}

// ✅ SOLUTION : Assurer les types avec $jsonSchema
{
  $and: [
    {
      $jsonSchema: {
        properties: {
          prix: { bsonType: "double" }  // Forcer le type
        }
      }
    },
    {
      $expr: {
        $gt: ["$prix", 0]
      }
    }
  ]
}
```

### 5. Expressions trop complexes

```javascript
// ❌ TROP COMPLEXE : Difficile à maintenir
{
  $expr: {
    $and: [
      {
        $or: [
          {
            $and: [
              { $cond: { if: ..., then: ..., else: ... } },
              { $cond: { if: ..., then: ..., else: ... } }
            ]
          },
          {
            $not: {
              $and: [ /* ... */ ]
            }
          }
        ]
      }
    ]
  }
}

// ✅ MIEUX : Simplifier ou diviser
// Ou valider côté application pour logique trop complexe
```

---

## 🎓 Résumé

### Quand utiliser `$expr` ?

| Besoin | Utiliser `$expr` ? |
|--------|-------------------|
| Valider types de données | ❌ Utiliser `$jsonSchema` |
| Format de string (regex) | ❌ Utiliser `$jsonSchema` |
| Champs obligatoires | ❌ Utiliser `$jsonSchema` |
| Comparer deux champs | ✅ Utiliser `$expr` |
| Calculs arithmétiques | ✅ Utiliser `$expr` |
| Validation temporelle | ✅ Utiliser `$expr` |
| Règles métier complexes | ✅ Utiliser `$expr` |

### Opérateurs $expr principaux

**Comparaison** : `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`
**Arithmétique** : `$add`, `$subtract`, `$multiply`, `$divide`, `$mod`
**Logique** : `$and`, `$or`, `$not`, `$cond`
**Utilitaires** : `$ifNull`, `$reduce`
**Temporel** : `$$NOW`, `$dateSubtract`, `$dateAdd`

### Checklist

✅ **Conception** :
- [ ] Identifier ce qui nécessite vraiment `$expr`
- [ ] Privilégier `$jsonSchema` quand possible
- [ ] Combiner les deux pour validation complète

✅ **Implémentation** :
- [ ] Utiliser `$` devant les champs
- [ ] Utiliser `$$NOW` pour la date actuelle
- [ ] Gérer les champs optionnels avec `$ifNull`
- [ ] Vérifier la division par zéro

✅ **Tests** :
- [ ] Cas valides
- [ ] Cas invalides
- [ ] Cas limites (null, 0, dates extrêmes)
- [ ] Performance sur gros volumes

### Points clés

- ✅ `$expr` utilise le **langage d'agrégation** MongoDB
- ✅ Permet de **comparer des champs** entre eux
- ✅ Supporte les **calculs** et la **logique conditionnelle**
- ✅ Utiliser `$$NOW` pour la **date actuelle**
- ✅ **Combiner** avec `$jsonSchema` pour validation complète
- ✅ `$expr` peut être **plus lent** que `$jsonSchema`
- ✅ Référencer les champs avec **`$nomChamp`**
- ✅ Gérer les champs optionnels avec **`$ifNull`**

---

## 📚 Dans la prochaine section

Dans la section suivante (7.10), nous verrons les **bonnes pratiques de validation** qui rassemblent tous les concepts vus précédemment pour créer des schémas de validation robustes et maintenables.

---


⏭️ [Bonnes pratiques de validation](/07-validation-des-schemas/10-bonnes-pratiques-validation.md)
