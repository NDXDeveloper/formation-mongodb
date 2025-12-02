🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.3 Règles de validation ($jsonSchema)

## 📚 Vue d'ensemble

Dans cette section, nous allons explorer en détail toutes les **règles de validation** que vous pouvez appliquer avec `$jsonSchema` dans MongoDB. Ces règles vous permettent de contrôler précisément la structure et le contenu de vos documents.

Nous allons voir les règles organisées par **type de données**, des plus simples aux plus complexes.

---

## 🎯 Organisation des règles

Les règles de validation sont organisées en plusieurs catégories :

| Catégorie | Description | Exemples |
|-----------|-------------|----------|
| **Règles générales** | S'appliquent à tous les types | `bsonType`, `enum`, `description` |
| **Règles pour strings** | Chaînes de caractères | `minLength`, `maxLength`, `pattern` |
| **Règles pour nombres** | Entiers et décimaux | `minimum`, `maximum`, `multipleOf` |
| **Règles pour tableaux** | Arrays | `minItems`, `maxItems`, `uniqueItems` |
| **Règles pour objets** | Documents imbriqués | `required`, `properties`, `minProperties` |
| **Règles logiques** | Combinaisons complexes | `oneOf`, `anyOf`, `allOf`, `not` |

---

## 🌐 Règles générales (tous types)

Ces règles s'appliquent à n'importe quel type de données.

### `bsonType` - Spécifier le type

La règle la plus importante : définir le type de données attendu.

```javascript
// Un seul type
{
  nom: {
    bsonType: "string"
  }
}

// Plusieurs types possibles
{
  identifiant: {
    bsonType: ["string", "int"]  // Peut être string OU int
  }
}
```

**Exemple pratique** :
```javascript
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        reference: {
          bsonType: ["string", "int"],  // Accepte "ABC123" ou 12345
          description: "Référence produit (texte ou numérique)"
        }
      }
    }
  }
})

// ✅ Valide
db.produits.insertOne({ reference: "ABC123" })
db.produits.insertOne({ reference: 12345 })

// ❌ Invalide
db.produits.insertOne({ reference: 123.45 })  // double non autorisé
```

### `enum` - Valeurs autorisées

Restreindre à une liste de valeurs précises.

```javascript
{
  statut: {
    enum: ["actif", "inactif", "suspendu", "archive"]
  }
}
```

**Exemple complet** :
```javascript
db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        statut: {
          enum: ["en_attente", "en_cours", "expedie", "livre", "annule"],
          description: "Statut de la commande"
        }
      }
    }
  }
})

// ✅ Valide
db.commandes.insertOne({ statut: "en_cours" })

// ❌ Invalide : valeur non autorisée
db.commandes.insertOne({ statut: "en_preparation" })
```

### `description` - Documentation

Ajouter une description explicative (très recommandé).

```javascript
{
  email: {
    bsonType: "string",
    description: "Adresse email de contact de l'utilisateur. Doit être unique dans la base."
  }
}
```

### `title` - Titre court

Donner un nom lisible au champ.

```javascript
{
  dateNaissance: {
    title: "Date de naissance",
    bsonType: "date",
    description: "Date de naissance de l'utilisateur au format ISO"
  }
}
```

---

## 📝 Règles pour les chaînes (string)

### `minLength` et `maxLength` - Longueur

Contrôler la longueur minimale et maximale d'une chaîne.

```javascript
{
  nom: {
    bsonType: "string",
    minLength: 2,       // Au moins 2 caractères
    maxLength: 50,      // Maximum 50 caractères
    description: "Nom entre 2 et 50 caractères"
  }
}
```

**Exemple pratique** :
```javascript
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        nom: {
          bsonType: "string",
          minLength: 2,
          maxLength: 50
        },
        pseudo: {
          bsonType: "string",
          minLength: 3,
          maxLength: 20
        }
      }
    }
  }
})

// ✅ Valide
db.utilisateurs.insertOne({ nom: "Dupont", pseudo: "marie_d" })

// ❌ Invalide : nom trop court
db.utilisateurs.insertOne({ nom: "D", pseudo: "user123" })

// ❌ Invalide : pseudo trop long
db.utilisateurs.insertOne({ nom: "Martin", pseudo: "utilisateur_avec_un_tres_long_pseudo" })
```

### `pattern` - Expression régulière

Valider le format avec une regex (expression régulière).

```javascript
{
  email: {
    bsonType: "string",
    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
    description: "Email au format valide"
  }
}
```

**Patterns courants** :

```javascript
// Email
pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"

// Code postal français (5 chiffres)
pattern: "^[0-9]{5}$"

// Téléphone français (10 chiffres)
pattern: "^0[1-9][0-9]{8}$"

// URL
pattern: "^https?:\\/\\/.+"

// Code hexadécimal couleur (#RRGGBB)
pattern: "^#[0-9A-Fa-f]{6}$"

// Date au format YYYY-MM-DD
pattern: "^[0-9]{4}-[0-9]{2}-[0-9]{2}$"

// Commence par une majuscule
pattern: "^[A-Z].*"

// Uniquement lettres et espaces
pattern: "^[a-zA-Z ]+$"

// Code produit (3 lettres + 4 chiffres)
pattern: "^[A-Z]{3}[0-9]{4}$"
```

**Exemple avec plusieurs patterns** :
```javascript
db.createCollection("contacts", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        telephone: {
          bsonType: "string",
          pattern: "^0[1-9][0-9]{8}$"
        },
        codePostal: {
          bsonType: "string",
          pattern: "^[0-9]{5}$"
        }
      }
    }
  }
})

// ✅ Valide
db.contacts.insertOne({
  email: "contact@example.com",
  telephone: "0612345678",
  codePostal: "75001"
})

// ❌ Invalide : téléphone mal formaté
db.contacts.insertOne({
  email: "test@example.com",
  telephone: "06 12 34 56 78",  // Espaces non autorisés
  codePostal: "75001"
})
```

**⚠️ Important** : N'oubliez pas d'échapper les backslashes (`\\`) dans les regex !

---

## 🔢 Règles pour les nombres (int, long, double, decimal)

### `minimum` et `maximum` - Valeurs min/max (inclusive)

Définir des bornes pour les valeurs numériques.

```javascript
{
  age: {
    bsonType: "int",
    minimum: 0,         // >= 0
    maximum: 150,       // <= 150
    description: "Âge entre 0 et 150 ans"
  }
}
```

**Exemple pratique** :
```javascript
db.createCollection("employes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        age: {
          bsonType: "int",
          minimum: 18,      // Minimum 18 ans
          maximum: 65       // Maximum 65 ans
        },
        salaire: {
          bsonType: "double",
          minimum: 0,       // Salaire positif
          maximum: 1000000  // Plafond
        },
        experience: {
          bsonType: "int",
          minimum: 0,       // Années d'expérience positives
          maximum: 50
        }
      }
    }
  }
})

// ✅ Valide
db.employes.insertOne({ age: 30, salaire: 45000.00, experience: 5 })

// ❌ Invalide : âge trop jeune
db.employes.insertOne({ age: 16, salaire: 25000.00, experience: 0 })

// ❌ Invalide : salaire négatif
db.employes.insertOne({ age: 25, salaire: -1000.00, experience: 2 })
```

### `exclusiveMinimum` et `exclusiveMaximum` - Valeurs min/max (exclusive)

Bornes **exclusives** (la valeur exacte n'est pas incluse).

```javascript
{
  note: {
    bsonType: "double",
    exclusiveMinimum: 0,    // > 0 (mais pas 0)
    exclusiveMaximum: 20,   // < 20 (mais pas 20)
    description: "Note strictement entre 0 et 20"
  }
}
```

**Comparaison inclusive vs exclusive** :

```javascript
// Inclusive (défaut)
{
  age: {
    bsonType: "int",
    minimum: 18,      // Accepte 18, 19, 20, ...
    maximum: 65       // Accepte ..., 63, 64, 65
  }
}

// Exclusive
{
  pourcentage: {
    bsonType: "double",
    exclusiveMinimum: 0,     // Accepte 0.01, 0.1, ... mais PAS 0
    exclusiveMaximum: 100    // Accepte ..., 99.9, 99.99 mais PAS 100
  }
}
```

### `multipleOf` - Multiple de

La valeur doit être un multiple d'un nombre donné.

```javascript
{
  quantite: {
    bsonType: "int",
    multipleOf: 5,      // 0, 5, 10, 15, 20, ...
    description: "Quantité par lots de 5"
  }
}
```

**Exemples pratiques** :

```javascript
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        quantite: {
          bsonType: "int",
          multipleOf: 10,     // Lots de 10
          minimum: 10
        },
        prix: {
          bsonType: "double",
          multipleOf: 0.01,   // Prix avec 2 décimales max
          minimum: 0
        }
      }
    }
  }
})

// ✅ Valide
db.produits.insertOne({ quantite: 50, prix: 19.99 })

// ❌ Invalide : quantité pas un multiple de 10
db.produits.insertOne({ quantite: 35, prix: 25.00 })

// ❌ Invalide : prix avec trop de décimales
db.produits.insertOne({ quantite: 40, prix: 12.999 })
```

---

## 📊 Règles pour les tableaux (array)

### `minItems` et `maxItems` - Nombre d'éléments

Contrôler la taille du tableau.

```javascript
{
  tags: {
    bsonType: "array",
    minItems: 1,        // Au moins 1 élément
    maxItems: 10,       // Maximum 10 éléments
    description: "Liste de 1 à 10 tags"
  }
}
```

**Exemple** :
```javascript
db.createCollection("articles", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        tags: {
          bsonType: "array",
          minItems: 1,
          maxItems: 5
        }
      }
    }
  }
})

// ✅ Valide
db.articles.insertOne({ tags: ["mongodb", "nosql", "database"] })

// ❌ Invalide : tableau vide
db.articles.insertOne({ tags: [] })

// ❌ Invalide : trop d'éléments
db.articles.insertOne({ tags: ["tag1", "tag2", "tag3", "tag4", "tag5", "tag6"] })
```

### `uniqueItems` - Éléments uniques

Tous les éléments du tableau doivent être différents.

```javascript
{
  categories: {
    bsonType: "array",
    uniqueItems: true,  // Pas de doublons
    description: "Liste de catégories uniques"
  }
}
```

**Exemple** :
```javascript
db.createCollection("projets", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        competences: {
          bsonType: "array",
          uniqueItems: true
        }
      }
    }
  }
})

// ✅ Valide : tous différents
db.projets.insertOne({ competences: ["JavaScript", "Python", "MongoDB"] })

// ❌ Invalide : "Python" en double
db.projets.insertOne({ competences: ["JavaScript", "Python", "Python", "MongoDB"] })
```

### `items` - Type des éléments

Définir le type et les contraintes des éléments du tableau.

```javascript
{
  notes: {
    bsonType: "array",
    items: {
      bsonType: "double",
      minimum: 0,
      maximum: 20
    },
    description: "Notes entre 0 et 20"
  }
}
```

**Exemple avec objet** :
```javascript
db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        articles: {
          bsonType: "array",
          items: {
            bsonType: "object",
            required: ["nom", "quantite", "prix"],
            properties: {
              nom: { bsonType: "string" },
              quantite: {
                bsonType: "int",
                minimum: 1
              },
              prix: {
                bsonType: "double",
                minimum: 0
              }
            }
          }
        }
      }
    }
  }
})

// ✅ Valide : tableau d'objets conformes
db.commandes.insertOne({
  articles: [
    { nom: "Livre", quantite: 2, prix: 15.99 },
    { nom: "Stylo", quantite: 5, prix: 1.50 }
  ]
})

// ❌ Invalide : quantité négative dans un article
db.commandes.insertOne({
  articles: [
    { nom: "Cahier", quantite: -1, prix: 3.00 }
  ]
})
```

---

## 📦 Règles pour les objets (object)

### `required` - Propriétés obligatoires

Liste les champs qui doivent être présents dans l'objet.

```javascript
{
  adresse: {
    bsonType: "object",
    required: ["rue", "ville", "codePostal"],
    properties: {
      rue: { bsonType: "string" },
      ville: { bsonType: "string" },
      codePostal: { bsonType: "string" },
      pays: { bsonType: "string" }  // Facultatif
    }
  }
}
```

### `properties` - Définition des propriétés

Décrire chaque propriété de l'objet.

```javascript
{
  utilisateur: {
    bsonType: "object",
    properties: {
      nom: {
        bsonType: "string",
        minLength: 2
      },
      email: {
        bsonType: "string",
        pattern: "^.+@.+\\..+$"
      }
    }
  }
}
```

### `additionalProperties` - Propriétés supplémentaires

Autoriser ou non des propriétés non définies.

```javascript
// Interdire les propriétés supplémentaires
{
  bsonType: "object",
  properties: {
    nom: { bsonType: "string" },
    age: { bsonType: "int" }
  },
  additionalProperties: false  // Seuls "nom" et "age" autorisés
}

// Autoriser les propriétés supplémentaires (défaut)
{
  bsonType: "object",
  properties: {
    nom: { bsonType: "string" }
  },
  additionalProperties: true  // Autres champs acceptés
}
```

**Exemple complet** :
```javascript
db.createCollection("parametres", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["type"],
      properties: {
        type: { bsonType: "string" },
        valeur: { bsonType: "string" }
      },
      additionalProperties: false  // Strict
    }
  }
})

// ✅ Valide
db.parametres.insertOne({ type: "couleur", valeur: "rouge" })

// ❌ Invalide : champ "description" non autorisé
db.parametres.insertOne({
  type: "taille",
  valeur: "grand",
  description: "Taille du produit"  // Erreur !
})
```

### `minProperties` et `maxProperties` - Nombre de propriétés

Limiter le nombre de propriétés dans l'objet.

```javascript
{
  metadonnees: {
    bsonType: "object",
    minProperties: 1,    // Au moins 1 propriété
    maxProperties: 10,   // Maximum 10 propriétés
    description: "Métadonnées avec 1 à 10 propriétés"
  }
}
```

### `dependencies` - Dépendances entre champs

Si un champ est présent, d'autres doivent l'être aussi.

```javascript
{
  bsonType: "object",
  properties: {
    carteCredit: { bsonType: "string" },
    dateExpiration: { bsonType: "string" },
    cvv: { bsonType: "string" }
  },
  dependencies: {
    carteCredit: ["dateExpiration", "cvv"]
    // Si "carteCredit" existe, "dateExpiration" et "cvv" obligatoires
  }
}
```

**Exemple pratique** :
```javascript
db.createCollection("paiements", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        montant: { bsonType: "double" },
        carteCredit: { bsonType: "string" },
        dateExpiration: { bsonType: "string" },
        cvv: { bsonType: "string" }
      },
      dependencies: {
        carteCredit: ["dateExpiration", "cvv"]
      }
    }
  }
})

// ✅ Valide : paiement sans carte
db.paiements.insertOne({ montant: 50.00 })

// ✅ Valide : carte avec tous les champs requis
db.paiements.insertOne({
  montant: 100.00,
  carteCredit: "1234567890123456",
  dateExpiration: "12/25",
  cvv: "123"
})

// ❌ Invalide : carte sans CVV
db.paiements.insertOne({
  montant: 75.00,
  carteCredit: "1234567890123456",
  dateExpiration: "12/25"
  // CVV manquant !
})
```

---

## 🔀 Règles logiques avancées

Ces règles permettent de créer des validations complexes en combinant plusieurs conditions.

### `anyOf` - Au moins une condition vraie

Le document doit satisfaire **au moins une** des conditions.

```javascript
{
  contact: {
    anyOf: [
      { bsonType: "string" },  // Peut être un string
      { bsonType: "int" }      // OU un int
    ]
  }
}
```

**Exemple pratique** :
```javascript
db.createCollection("notifications", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        destinataire: {
          anyOf: [
            // Option 1 : email
            {
              bsonType: "string",
              pattern: "^.+@.+\\..+$"
            },
            // Option 2 : numéro de téléphone
            {
              bsonType: "string",
              pattern: "^0[1-9][0-9]{8}$"
            }
          ]
        }
      }
    }
  }
})

// ✅ Valide : email
db.notifications.insertOne({ destinataire: "user@example.com" })

// ✅ Valide : téléphone
db.notifications.insertOne({ destinataire: "0612345678" })

// ❌ Invalide : ni email ni téléphone valide
db.notifications.insertOne({ destinataire: "invalid" })
```

### `allOf` - Toutes les conditions vraies

Le document doit satisfaire **toutes** les conditions.

```javascript
{
  prix: {
    allOf: [
      { bsonType: "double" },      // Doit être un double
      { minimum: 0 },              // ET >= 0
      { maximum: 1000 },           // ET <= 1000
      { multipleOf: 0.01 }         // ET multiple de 0.01
    ]
  }
}
```

**Cas d'usage** : Combiner plusieurs contraintes complexes.

### `oneOf` - Exactement une condition vraie

Le document doit satisfaire **exactement une seule** condition.

```javascript
{
  moyen Paiement: {
    oneOf: [
      // Soit carte bancaire
      {
        properties: {
          type: { enum: ["carte"] },
          numero: { bsonType: "string" }
        },
        required: ["type", "numero"]
      },
      // Soit virement
      {
        properties: {
          type: { enum: ["virement"] },
          iban: { bsonType: "string" }
        },
        required: ["type", "iban"]
      }
    ]
  }
}
```

### `not` - Condition inverse

Le document **ne doit pas** satisfaire la condition.

```javascript
{
  age: {
    not: {
      bsonType: "string"  // NE DOIT PAS être un string
    }
  }
}
```

**Exemple** : Interdire certaines valeurs

```javascript
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        pseudo: {
          bsonType: "string",
          not: {
            enum: ["admin", "root", "system"]  // Interdire ces pseudos
          }
        }
      }
    }
  }
})

// ✅ Valide
db.utilisateurs.insertOne({ pseudo: "jean_dupont" })

// ❌ Invalide : pseudo interdit
db.utilisateurs.insertOne({ pseudo: "admin" })
```

---

## 🎯 Exemples complexes

### Exemple 1 : Validation d'une fiche produit e-commerce

```javascript
db.createCollection("produits_ecommerce", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "reference", "prix", "stock", "statut"],
      properties: {
        nom: {
          bsonType: "string",
          minLength: 3,
          maxLength: 200,
          description: "Nom du produit"
        },
        reference: {
          bsonType: "string",
          pattern: "^[A-Z]{3}[0-9]{6}$",
          description: "Référence : 3 lettres + 6 chiffres (ex: PRD123456)"
        },
        prix: {
          bsonType: "decimal",
          minimum: 0,
          description: "Prix en euros"
        },
        stock: {
          bsonType: "int",
          minimum: 0,
          description: "Quantité en stock"
        },
        statut: {
          enum: ["disponible", "rupture", "sur_commande", "archive"],
          description: "Statut du produit"
        },
        dimensions: {
          bsonType: "object",
          properties: {
            longueur: { bsonType: "double", minimum: 0 },
            largeur: { bsonType: "double", minimum: 0 },
            hauteur: { bsonType: "double", minimum: 0 },
            unite: { enum: ["cm", "m", "mm"] }
          }
        },
        categories: {
          bsonType: "array",
          minItems: 1,
          maxItems: 5,
          uniqueItems: true,
          items: { bsonType: "string" }
        },
        tags: {
          bsonType: "array",
          maxItems: 10,
          items: {
            bsonType: "string",
            minLength: 2,
            maxLength: 30
          }
        }
      }
    }
  }
})
```

### Exemple 2 : Validation d'un document utilisateur avec adresse

```javascript
db.createCollection("profils_utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "nom", "prenom", "dateInscription"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        nom: {
          bsonType: "string",
          minLength: 2,
          maxLength: 50
        },
        prenom: {
          bsonType: "string",
          minLength: 2,
          maxLength: 50
        },
        dateNaissance: {
          bsonType: "date"
        },
        telephone: {
          anyOf: [
            {
              bsonType: "string",
              pattern: "^0[1-9][0-9]{8}$"  // Format français
            },
            {
              bsonType: "string",
              pattern: "^\\+[1-9][0-9]{1,14}$"  // Format international
            }
          ]
        },
        adresse: {
          bsonType: "object",
          required: ["rue", "ville", "codePostal", "pays"],
          properties: {
            rue: { bsonType: "string" },
            complement: { bsonType: "string" },
            ville: { bsonType: "string" },
            codePostal: {
              bsonType: "string",
              pattern: "^[0-9]{5}$"
            },
            pays: {
              bsonType: "string",
              enum: ["France", "Belgique", "Suisse", "Luxembourg"]
            }
          },
          additionalProperties: false
        },
        dateInscription: {
          bsonType: "date"
        },
        preferences: {
          bsonType: "object",
          properties: {
            newsletter: { bsonType: "bool" },
            notifications: { bsonType: "bool" }
          }
        }
      }
    }
  }
})
```

### Exemple 3 : Validation polymorphique (documents différents selon un type)

```javascript
db.createCollection("evenements", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["type", "date"],
      properties: {
        date: { bsonType: "date" }
      },
      oneOf: [
        // Type : conférence
        {
          properties: {
            type: { enum: ["conference"] },
            titre: { bsonType: "string" },
            orateurs: {
              bsonType: "array",
              items: { bsonType: "string" }
            },
            duree: { bsonType: "int", minimum: 30 }
          },
          required: ["type", "titre", "orateurs"]
        },
        // Type : webinaire
        {
          properties: {
            type: { enum: ["webinaire"] },
            titre: { bsonType: "string" },
            lienInscription: { bsonType: "string" },
            capaciteMax: { bsonType: "int", minimum: 1 }
          },
          required: ["type", "titre", "lienInscription"]
        },
        // Type : atelier
        {
          properties: {
            type: { enum: ["atelier"] },
            nom: { bsonType: "string" },
            materiel: {
              bsonType: "array",
              items: { bsonType: "string" }
            },
            niveauRequis: { enum: ["debutant", "intermediaire", "avance"] }
          },
          required: ["type", "nom", "niveauRequis"]
        }
      ]
    }
  }
})
```

---

## 💡 Bonnes pratiques

### 1. Commencer simple, complexifier progressivement

```javascript
// ✅ Étape 1 : Validation de base
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email"],
    properties: {
      nom: { bsonType: "string" },
      email: { bsonType: "string" }
    }
  }
}

// ✅ Étape 2 : Ajouter des contraintes
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email"],
    properties: {
      nom: {
        bsonType: "string",
        minLength: 2,
        maxLength: 50
      },
      email: {
        bsonType: "string",
        pattern: "^.+@.+\\..+$"
      }
    }
  }
}
```

### 2. Utiliser `description` partout

```javascript
{
  prix: {
    bsonType: "decimal",
    minimum: 0,
    description: "Prix en euros TTC. Doit être positif et avoir au maximum 2 décimales."
  }
}
```

### 3. Tester le schéma avant déploiement

Utilisez `validationAction: "warn"` pendant les tests :

```javascript
db.runCommand({
  collMod: "maCollection",
  validator: { $jsonSchema: { /* ... */ } },
  validationAction: "warn"  // Accepte mais enregistre les erreurs
})
```

### 4. Documenter les regex complexes

```javascript
{
  telephone: {
    bsonType: "string",
    pattern: "^\\+?[1-9]\\d{1,14}$",
    description: "Numéro de téléphone au format E.164 (international). Exemples : +33612345678, +14155552671"
  }
}
```

### 5. Préférer `enum` aux regex quand possible

```javascript
// ✅ Meilleur : clair et performant
{
  statut: {
    enum: ["actif", "inactif", "suspendu"]
  }
}

// ❌ Moins bon : regex inutilement complexe
{
  statut: {
    bsonType: "string",
    pattern: "^(actif|inactif|suspendu)$"
  }
}
```

---

## ⚠️ Pièges à éviter

### 1. Regex sans échappement

```javascript
// ❌ Incorrect
pattern: "\d{5}"

// ✅ Correct
pattern: "\\d{5}"
```

### 2. Oublier `bsonType: "object"` à la racine

```javascript
// ❌ Incomplet
{
  $jsonSchema: {
    required: ["nom"],
    properties: { /* ... */ }
  }
}

// ✅ Complet
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom"],
    properties: { /* ... */ }
  }
}
```

### 3. Confondre `anyOf` et `oneOf`

- `anyOf` : au moins une condition (peut être plusieurs)
- `oneOf` : exactement une condition (pas plus, pas moins)

### 4. Utiliser `additionalProperties: false` trop tôt

Peut bloquer l'évolution du schéma. Commencez avec `true` puis passez à `false` quand le schéma est stabilisé.

---

## 🎓 Résumé

| Règle | Applicable à | Fonction |
|-------|--------------|----------|
| `bsonType` | Tous | Définir le type |
| `enum` | Tous | Lister les valeurs autorisées |
| `minLength` / `maxLength` | String | Longueur min/max |
| `pattern` | String | Valider avec regex |
| `minimum` / `maximum` | Nombres | Valeurs min/max inclusive |
| `exclusiveMinimum` / `exclusiveMaximum` | Nombres | Valeurs min/max exclusive |
| `multipleOf` | Nombres | Multiple de N |
| `minItems` / `maxItems` | Array | Nombre d'éléments |
| `uniqueItems` | Array | Éléments uniques |
| `items` | Array | Type des éléments |
| `required` | Object | Champs obligatoires |
| `properties` | Object | Définition des champs |
| `additionalProperties` | Object | Autoriser champs supplémentaires |
| `minProperties` / `maxProperties` | Object | Nombre de propriétés |
| `dependencies` | Object | Dépendances entre champs |
| `anyOf` | Tous | Au moins une condition |
| `allOf` | Tous | Toutes les conditions |
| `oneOf` | Tous | Exactement une condition |
| `not` | Tous | Condition inverse |

---

## 📚 Dans la prochaine section

Dans la section suivante (7.4), nous verrons les **niveaux de validation** (`strict` vs `moderate`) et comment les utiliser selon vos besoins.

---


⏭️ [Niveaux de validation (strict, moderate)](/07-validation-des-schemas/04-niveaux-validation.md)
