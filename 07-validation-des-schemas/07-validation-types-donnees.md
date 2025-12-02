🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.7 Validation des types de données

## 📚 Vue d'ensemble

La **validation des types de données** est fondamentale pour garantir la cohérence de vos données MongoDB. Dans cette section, nous allons explorer en détail comment valider chaque type BSON avec des exemples pratiques et des cas d'usage réels.

MongoDB utilise **BSON** (Binary JSON) qui propose plus de types que JSON standard, ce qui nécessite une attention particulière lors de la validation.

---

## 🎯 Rappel : BSON vs JSON

### Différences clés

| JSON Standard | BSON (MongoDB) |
|---------------|----------------|
| `number` (générique) | `int`, `long`, `double`, `decimal` |
| `string` | `string` |
| `boolean` | `bool` |
| `null` | `null` |
| `array` | `array` |
| `object` | `object` |
| ❌ Pas de date | `date` |
| ❌ Pas d'ID | `objectId` |
| ❌ Pas de binaire | `binData` |
| ❌ Pas de timestamp | `timestamp` |

### Pourquoi c'est important

```javascript
// En JSON, tout est "number"
{ prix: 19.99, quantite: 5 }

// En BSON, on distingue les types numériques
{
  prix: 19.99,      // double
  quantite: 5       // int
}

// La validation doit refléter cette distinction !
```

---

## 🔤 Type `string` - Chaînes de caractères

### Validation de base

```javascript
{
  nom: {
    bsonType: "string",
    description: "Doit être une chaîne de caractères"
  }
}
```

### Contraintes disponibles

**1. Longueur minimale et maximale**

```javascript
{
  pseudo: {
    bsonType: "string",
    minLength: 3,        // Au moins 3 caractères
    maxLength: 20,       // Maximum 20 caractères
    description: "Pseudo entre 3 et 20 caractères"
  }
}
```

**Exemples** :
- ✅ `"jean123"` → Valide (7 caractères)
- ❌ `"ab"` → Trop court (2 caractères)
- ❌ `"utilisateur_avec_un_tres_long_pseudo"` → Trop long (38 caractères)

**2. Format avec expression régulière**

```javascript
{
  email: {
    bsonType: "string",
    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
    description: "Email valide"
  }
}
```

**Patterns courants** :

```javascript
// Email
pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"

// Téléphone français (mobile)
pattern: "^(\\+33|0)[6-7][0-9]{8}$"

// Code postal français
pattern: "^[0-9]{5}$"

// URL
pattern: "^https?:\\/\\/(www\\.)?[-a-zA-Z0-9@:%._\\+~#=]{1,256}\\.[a-zA-Z0-9()]{1,6}\\b"

// Couleur hexadécimale
pattern: "^#[0-9A-Fa-f]{6}$"

// UUID
pattern: "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$"

// Nom (lettres, espaces, traits d'union)
pattern: "^[a-zA-ZÀ-ÿ\\s\\-']+$"

// Code ISBN-13
pattern: "^(978|979)[0-9]{10}$"
```

**3. Valeurs énumérées**

```javascript
{
  statut: {
    bsonType: "string",
    enum: ["actif", "inactif", "suspendu", "archive"],
    description: "Statut du compte"
  }
}
```

**Exemples** :
- ✅ `"actif"` → Valide
- ✅ `"suspendu"` → Valide
- ❌ `"bloque"` → Non autorisé
- ❌ `"Actif"` → Non autorisé (sensible à la casse !)

### Exemple complet : Validation d'un profil utilisateur

```javascript
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["username", "email", "statut"],
      properties: {
        username: {
          bsonType: "string",
          minLength: 3,
          maxLength: 30,
          pattern: "^[a-zA-Z0-9_-]+$",
          description: "Nom d'utilisateur (3-30 caractères, lettres, chiffres, _ et -)"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Adresse email valide"
        },
        telephone: {
          bsonType: "string",
          pattern: "^(\\+33|0)[1-9][0-9]{8}$",
          description: "Numéro de téléphone français"
        },
        statut: {
          enum: ["actif", "inactif", "suspendu"],
          description: "Statut du compte"
        }
      }
    }
  }
})

// Tests
// ✅ Valide
db.utilisateurs.insertOne({
  username: "jean_dupont",
  email: "jean.dupont@example.com",
  telephone: "+33612345678",
  statut: "actif"
})

// ❌ Invalide : username trop court
db.utilisateurs.insertOne({
  username: "ab",
  email: "test@example.com",
  statut: "actif"
})

// ❌ Invalide : email mal formaté
db.utilisateurs.insertOne({
  username: "martin",
  email: "mauvais-email",
  statut: "actif"
})
```

---

## 🔢 Types numériques

MongoDB propose plusieurs types numériques selon la précision et la taille nécessaires.

### Type `int` - Entier 32 bits

**Plage** : -2,147,483,648 à 2,147,483,647

**Usage** : Compteurs, quantités, âges, etc.

```javascript
{
  age: {
    bsonType: "int",
    minimum: 0,
    maximum: 150,
    description: "Âge en années (entier)"
  },
  stock: {
    bsonType: "int",
    minimum: 0,
    description: "Quantité en stock"
  }
}
```

**Contraintes disponibles** :

```javascript
{
  note: {
    bsonType: "int",
    minimum: 0,              // Valeur minimale (inclusive)
    maximum: 20,             // Valeur maximale (inclusive)
    exclusiveMinimum: 0,     // > 0 (pas 0)
    exclusiveMaximum: 20,    // < 20 (pas 20)
    multipleOf: 5,           // Multiple de 5 (0, 5, 10, 15, 20)
    description: "Note sur 20"
  }
}
```

**Exemples** :
- ✅ `10` → int valide
- ❌ `10.5` → double, pas int
- ❌ `"10"` → string, pas int
- ❌ `NumberInt(10)` constructeur OK mais le type validé est int

### Type `long` - Entier 64 bits

**Plage** : -9,223,372,036,854,775,808 à 9,223,372,036,854,775,807

**Usage** : Grands nombres entiers, timestamps Unix, compteurs très élevés

```javascript
{
  population: {
    bsonType: "long",
    minimum: 0,
    description: "Population mondiale (long)"
  },
  timestamp_unix: {
    bsonType: "long",
    description: "Timestamp Unix en millisecondes"
  }
}
```

### Type `double` - Nombre à virgule flottante

**Usage** : Prix, mesures, pourcentages, calculs décimaux courants

```javascript
{
  prix: {
    bsonType: "double",
    minimum: 0,
    maximum: 999999.99,
    description: "Prix en euros"
  },
  temperature: {
    bsonType: "double",
    minimum: -273.15,  // Zéro absolu
    maximum: 5778,     // Température surface du soleil (K)
    description: "Température en Kelvin"
  },
  pourcentage: {
    bsonType: "double",
    minimum: 0,
    maximum: 100,
    description: "Pourcentage (0-100)"
  }
}
```

**Important** : Les doubles peuvent avoir des imprécisions dues à la représentation binaire.

```javascript
// Exemple d'imprécision
0.1 + 0.2 === 0.3  // false ! (0.30000000000000004)
```

### Type `decimal` - Décimal haute précision (Decimal128)

**Usage** : Valeurs monétaires, calculs financiers précis, mesures scientifiques

```javascript
{
  montant: {
    bsonType: "decimal",
    minimum: 0,
    description: "Montant financier (précision exacte)"
  },
  taux_interet: {
    bsonType: "decimal",
    minimum: 0,
    maximum: 100,
    description: "Taux d'intérêt en pourcentage"
  }
}
```

**Création de valeurs decimal** :

```javascript
// Dans l'application
db.transactions.insertOne({
  montant: NumberDecimal("1234.56"),  // Précision exacte
  commission: NumberDecimal("0.025")
})
```

### Comparaison des types numériques

```javascript
db.createCollection("demo_nombres", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        compteur: {
          bsonType: "int",
          description: "Petit entier (-2B à +2B)"
        },
        grand_nombre: {
          bsonType: "long",
          description: "Grand entier (-9 quintillions à +9 quintillions)"
        },
        mesure_approx: {
          bsonType: "double",
          description: "Nombre décimal avec imprécision acceptable"
        },
        montant_exact: {
          bsonType: "decimal",
          description: "Nombre décimal avec précision exacte"
        }
      }
    }
  }
})

// Tests
db.demo_nombres.insertOne({
  compteur: 42,                           // int
  grand_nombre: NumberLong("9999999999"), // long
  mesure_approx: 3.14159,                 // double
  montant_exact: NumberDecimal("99.99")   // decimal
})
```

### Accepter plusieurs types numériques

Parfois, vous voulez accepter n'importe quel type numérique :

```javascript
{
  valeur: {
    bsonType: ["int", "long", "double", "decimal"],
    minimum: 0,
    description: "Accepte tout type numérique positif"
  }
}
```

---

## ✅ Type `bool` - Booléen

**Valeurs** : `true` ou `false` uniquement

```javascript
{
  actif: {
    bsonType: "bool",
    description: "Compte actif ou non"
  },
  newsletter: {
    bsonType: "bool",
    description: "Inscription à la newsletter"
  }
}
```

**Important** : Les booléens sont stricts en BSON

```javascript
// ✅ Valide
{ actif: true }
{ actif: false }

// ❌ Invalide
{ actif: 1 }        // Number, pas bool
{ actif: 0 }        // Number, pas bool
{ actif: "true" }   // String, pas bool
{ actif: "false" }  // String, pas bool
{ actif: null }     // Null, pas bool
```

**Si vous voulez accepter plusieurs types** :

```javascript
{
  actif: {
    bsonType: ["bool", "null"],  // Accepte true, false ou null
    description: "Statut actif (ou inconnu si null)"
  }
}
```

---

## 🗓️ Type `date` - Date et heure

**Format** : Stocké en millisecondes depuis l'epoch Unix (1970-01-01)

```javascript
{
  dateCreation: {
    bsonType: "date",
    description: "Date de création du document"
  },
  dateNaissance: {
    bsonType: "date",
    description: "Date de naissance"
  }
}
```

**Création de dates** :

```javascript
// Différentes façons de créer des dates
db.evenements.insertOne({
  date1: new Date(),                          // Date actuelle
  date2: new Date("2025-01-15"),             // Date spécifique
  date3: new Date(2025, 0, 15),              // Année, mois (0-11), jour
  date4: ISODate("2025-01-15T10:30:00Z")     // Format ISO
})
```

**Validation avec contraintes temporelles** :

```javascript
{
  dateNaissance: {
    bsonType: "date",
    description: "Date de naissance (doit être dans le passé)"
    // Note : MongoDB ne peut pas valider "dans le passé" directement
    // Cela doit être fait dans l'application ou avec $expr
  }
}
```

**Validation avancée avec $expr** :

```javascript
db.createCollection("utilisateurs", {
  validator: {
    $expr: {
      $and: [
        // dateNaissance doit être dans le passé
        { $lt: ["$dateNaissance", "$$NOW"] },
        // dateNaissance doit être après 1900-01-01
        { $gt: ["$dateNaissance", new Date("1900-01-01")] }
      ]
    }
  }
})
```

**Exemple complet** :

```javascript
db.createCollection("reservations", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["dateDebut", "dateFin"],
      properties: {
        dateDebut: {
          bsonType: "date",
          description: "Date de début de réservation"
        },
        dateFin: {
          bsonType: "date",
          description: "Date de fin de réservation"
        }
      }
    }
  }
})

// Test
db.reservations.insertOne({
  dateDebut: new Date("2025-06-01"),
  dateFin: new Date("2025-06-07")
})
```

---

## 🆔 Type `objectId` - Identifiant unique

**Format** : 12 octets (24 caractères hexadécimaux)

```javascript
{
  _id: {
    bsonType: "objectId",
    description: "Identifiant unique du document"
  },
  auteur_id: {
    bsonType: "objectId",
    description: "Référence vers l'utilisateur auteur"
  }
}
```

**Structure d'un ObjectId** :
- 4 octets : timestamp (secondes depuis epoch)
- 5 octets : valeur aléatoire
- 3 octets : compteur incrémental

**Exemple** :

```javascript
db.createCollection("articles", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "titre", "auteur_id"],
      properties: {
        _id: {
          bsonType: "objectId"
        },
        titre: {
          bsonType: "string"
        },
        auteur_id: {
          bsonType: "objectId",
          description: "ID de l'auteur (référence vers collection utilisateurs)"
        }
      }
    }
  }
})

// Test
db.articles.insertOne({
  titre: "Introduction à MongoDB",
  auteur_id: ObjectId("507f1f77bcf86cd799439011")
})

// ❌ Invalide
db.articles.insertOne({
  titre: "Article test",
  auteur_id: "507f1f77bcf86cd799439011"  // String, pas ObjectId !
})
```

---

## 📦 Type `array` - Tableau

### Validation de base

```javascript
{
  tags: {
    bsonType: "array",
    description: "Liste de tags"
  }
}
```

### Contraintes sur le tableau

**1. Nombre d'éléments**

```javascript
{
  tags: {
    bsonType: "array",
    minItems: 1,      // Au moins 1 tag
    maxItems: 10,     // Maximum 10 tags
    description: "1 à 10 tags"
  }
}
```

**2. Éléments uniques**

```javascript
{
  categories: {
    bsonType: "array",
    uniqueItems: true,  // Pas de doublons
    description: "Liste de catégories (uniques)"
  }
}
```

**Exemples** :
- ✅ `["tech", "mongodb", "nosql"]` → Valide
- ❌ `["tech", "mongodb", "tech"]` → Doublon "tech"

**3. Type des éléments**

```javascript
{
  notes: {
    bsonType: "array",
    items: {
      bsonType: "int",
      minimum: 0,
      maximum: 20
    },
    description: "Liste de notes (entiers entre 0 et 20)"
  }
}
```

**Exemples** :
- ✅ `[15, 18, 12, 16]` → Valide
- ❌ `[15, 18.5, 12]` → 18.5 est un double, pas int
- ❌ `[15, 25, 12]` → 25 dépasse le maximum

### Tableaux d'objets

```javascript
{
  articles: {
    bsonType: "array",
    items: {
      bsonType: "object",
      required: ["nom", "quantite", "prix"],
      properties: {
        nom: {
          bsonType: "string"
        },
        quantite: {
          bsonType: "int",
          minimum: 1
        },
        prix: {
          bsonType: "double",
          minimum: 0
        }
      }
    },
    description: "Liste d'articles commandés"
  }
}
```

**Exemple** :

```javascript
db.commandes.insertOne({
  numero: "CMD-001",
  articles: [
    { nom: "Clavier", quantite: 2, prix: 29.99 },
    { nom: "Souris", quantite: 1, prix: 15.50 }
  ]
})
```

### Exemple complet : Panier e-commerce

```javascript
db.createCollection("paniers", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["client_id", "articles"],
      properties: {
        client_id: {
          bsonType: "objectId"
        },
        articles: {
          bsonType: "array",
          minItems: 1,        // Au moins 1 article
          maxItems: 50,       // Maximum 50 articles
          items: {
            bsonType: "object",
            required: ["produit_id", "nom", "quantite", "prix_unitaire"],
            properties: {
              produit_id: {
                bsonType: "objectId"
              },
              nom: {
                bsonType: "string",
                minLength: 1
              },
              quantite: {
                bsonType: "int",
                minimum: 1,
                maximum: 99
              },
              prix_unitaire: {
                bsonType: "decimal",
                minimum: 0
              }
            }
          }
        },
        codes_promo: {
          bsonType: "array",
          maxItems: 5,
          uniqueItems: true,
          items: {
            bsonType: "string",
            pattern: "^[A-Z0-9]{6,12}$"
          }
        }
      }
    }
  }
})
```

---

## 📄 Type `object` - Objet / Document imbriqué

### Validation de base

```javascript
{
  adresse: {
    bsonType: "object",
    description: "Adresse complète"
  }
}
```

### Structure d'objet complète

```javascript
{
  adresse: {
    bsonType: "object",
    required: ["rue", "ville", "codePostal"],
    properties: {
      rue: {
        bsonType: "string",
        minLength: 5
      },
      complement: {
        bsonType: "string"  // Facultatif
      },
      ville: {
        bsonType: "string",
        minLength: 2
      },
      codePostal: {
        bsonType: "string",
        pattern: "^[0-9]{5}$"
      },
      pays: {
        bsonType: "string",
        enum: ["France", "Belgique", "Suisse", "Luxembourg"]
      }
    },
    additionalProperties: false  // Pas d'autres champs
  }
}
```

### Objets imbriqués profonds

```javascript
db.createCollection("entreprises", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "siege_social"],
      properties: {
        nom: {
          bsonType: "string"
        },
        siege_social: {
          bsonType: "object",
          required: ["adresse", "contact"],
          properties: {
            adresse: {
              bsonType: "object",
              required: ["rue", "ville", "code_postal"],
              properties: {
                rue: { bsonType: "string" },
                ville: { bsonType: "string" },
                code_postal: { bsonType: "string" }
              }
            },
            contact: {
              bsonType: "object",
              properties: {
                telephone: { bsonType: "string" },
                email: { bsonType: "string" }
              }
            }
          }
        }
      }
    }
  }
})

// Test
db.entreprises.insertOne({
  nom: "TechCorp",
  siege_social: {
    adresse: {
      rue: "10 Avenue des Champs-Élysées",
      ville: "Paris",
      code_postal: "75008"
    },
    contact: {
      telephone: "+33142563789",
      email: "contact@techcorp.fr"
    }
  }
})
```

---

## ⭕ Type `null` - Valeur nulle

```javascript
{
  date_fin: {
    bsonType: ["date", "null"],
    description: "Date de fin (null si toujours en cours)"
  }
}
```

**Exemples** :
- ✅ `{ date_fin: new Date("2025-12-31") }` → Valide
- ✅ `{ date_fin: null }` → Valide
- ❌ `{ }` → Invalide si date_fin est requis
- ❌ `{ date_fin: undefined }` → undefined n'existe pas en BSON

---

## 🔧 Types spécialisés

### Type `binData` - Données binaires

**Usage** : Images, fichiers, données chiffrées

```javascript
{
  photo: {
    bsonType: "binData",
    description: "Photo de profil (données binaires)"
  }
}
```

### Type `timestamp` - Timestamp interne MongoDB

**Usage** : Principalement pour les opérations internes de MongoDB (réplication, oplog)

```javascript
{
  ts: {
    bsonType: "timestamp",
    description: "Timestamp MongoDB interne"
  }
}
```

**Note** : Pour des timestamps applicatifs, utilisez plutôt `date` ou `long`.

### Type `regex` - Expression régulière

```javascript
{
  pattern: {
    bsonType: "regex",
    description: "Pattern de recherche"
  }
}
```

---

## 🎨 Accepter plusieurs types

### Syntaxe

```javascript
{
  identifiant: {
    bsonType: ["string", "int"],
    description: "Peut être string OU int"
  }
}
```

### Cas d'usage courants

**1. Champ facultatif acceptant null**

```javascript
{
  telephone: {
    bsonType: ["string", "null"],
    pattern: "^0[1-9][0-9]{8}$",
    description: "Téléphone (ou null si non renseigné)"
  }
}
```

**2. Identifiant flexible**

```javascript
{
  reference: {
    bsonType: ["string", "int", "objectId"],
    description: "Référence produit (formats multiples acceptés)"
  }
}
```

**3. Valeur numérique flexible**

```javascript
{
  montant: {
    bsonType: ["int", "double", "decimal"],
    minimum: 0,
    description: "Montant (tout type numérique accepté)"
  }
}
```

---

## 🎯 Exemples complets par domaine

### E-commerce : Produit

```javascript
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "nom", "prix", "stock", "categories"],
      properties: {
        _id: {
          bsonType: "objectId"
        },
        nom: {
          bsonType: "string",
          minLength: 3,
          maxLength: 200
        },
        description: {
          bsonType: "string",
          maxLength: 2000
        },
        prix: {
          bsonType: "decimal",
          minimum: 0,
          exclusiveMinimum: 0  // Prix > 0 (pas 0)
        },
        prix_barre: {
          bsonType: ["decimal", "null"],
          minimum: 0
        },
        stock: {
          bsonType: "int",
          minimum: 0
        },
        categories: {
          bsonType: "array",
          minItems: 1,
          maxItems: 5,
          uniqueItems: true,
          items: {
            bsonType: "string"
          }
        },
        specifications: {
          bsonType: "object",
          properties: {
            poids: {
              bsonType: "double",
              minimum: 0
            },
            dimensions: {
              bsonType: "object",
              properties: {
                longueur: { bsonType: "double", minimum: 0 },
                largeur: { bsonType: "double", minimum: 0 },
                hauteur: { bsonType: "double", minimum: 0 }
              }
            }
          }
        },
        images: {
          bsonType: "array",
          maxItems: 10,
          items: {
            bsonType: "string",
            pattern: "^https?:\\/\\/.+\\.(jpg|jpeg|png|webp)$"
          }
        },
        actif: {
          bsonType: "bool"
        },
        date_creation: {
          bsonType: "date"
        },
        date_modification: {
          bsonType: "date"
        }
      }
    }
  }
})
```

### Réservation : Événement

```javascript
db.createCollection("reservations", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["evenement_id", "client", "date_reservation", "statut"],
      properties: {
        evenement_id: {
          bsonType: "objectId"
        },
        client: {
          bsonType: "object",
          required: ["nom", "email"],
          properties: {
            nom: {
              bsonType: "string",
              minLength: 2
            },
            prenom: {
              bsonType: "string"
            },
            email: {
              bsonType: "string",
              pattern: "^.+@.+\\..+$"
            },
            telephone: {
              bsonType: "string",
              pattern: "^(\\+33|0)[1-9][0-9]{8}$"
            }
          }
        },
        date_reservation: {
          bsonType: "date"
        },
        nombre_places: {
          bsonType: "int",
          minimum: 1,
          maximum: 10
        },
        statut: {
          enum: ["en_attente", "confirmee", "annulee", "terminee"]
        },
        montant_total: {
          bsonType: "decimal",
          minimum: 0
        },
        notes: {
          bsonType: "string",
          maxLength: 500
        }
      }
    }
  }
})
```

### IoT : Mesure capteur

```javascript
db.createCollection("mesures", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["capteur_id", "timestamp", "valeurs"],
      properties: {
        capteur_id: {
          bsonType: "string",
          pattern: "^SENSOR-[A-Z0-9]{8}$"
        },
        timestamp: {
          bsonType: "date"
        },
        valeurs: {
          bsonType: "object",
          properties: {
            temperature: {
              bsonType: "double",
              minimum: -50,
              maximum: 150
            },
            humidite: {
              bsonType: "double",
              minimum: 0,
              maximum: 100
            },
            pression: {
              bsonType: "double",
              minimum: 800,
              maximum: 1100
            },
            luminosite: {
              bsonType: "int",
              minimum: 0,
              maximum: 100000
            }
          }
        },
        batterie: {
          bsonType: "int",
          minimum: 0,
          maximum: 100,
          description: "Niveau de batterie en pourcentage"
        },
        localisation: {
          bsonType: "object",
          properties: {
            latitude: {
              bsonType: "double",
              minimum: -90,
              maximum: 90
            },
            longitude: {
              bsonType: "double",
              minimum: -180,
              maximum: 180
            }
          }
        }
      }
    }
  }
})
```

---

## 💡 Bonnes pratiques

### 1. Choisir le bon type numérique

```javascript
// ✅ BON : Types appropriés
{
  age: { bsonType: "int" },                    // Petit entier
  population: { bsonType: "long" },            // Grand entier
  distance_km: { bsonType: "double" },         // Mesure approximative
  prix: { bsonType: "decimal" },               // Argent (précision exacte)
  pourcentage: { bsonType: "double" }          // 0-100
}

// ❌ ÉVITER : Types inadaptés
{
  prix: { bsonType: "double" },                // Imprécisions possibles !
  age: { bsonType: "double" }                  // Inutilement complexe
}
```

### 2. Toujours valider les formats critiques

```javascript
// ✅ BON : Validation stricte des formats
{
  email: {
    bsonType: "string",
    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
  },
  telephone: {
    bsonType: "string",
    pattern: "^(\\+33|0)[1-9][0-9]{8}$"
  },
  code_postal: {
    bsonType: "string",
    pattern: "^[0-9]{5}$"
  }
}

// ❌ ÉVITER : Pas de validation de format
{
  email: { bsonType: "string" },     // Accepte n'importe quoi !
  telephone: { bsonType: "string" }
}
```

### 3. Utiliser enum pour valeurs fixes

```javascript
// ✅ BON : Enum pour valeurs limitées
{
  statut: {
    enum: ["brouillon", "publie", "archive"]
  },
  langue: {
    enum: ["fr", "en", "es", "de"]
  }
}

// ❌ ÉVITER : String sans contrainte
{
  statut: { bsonType: "string" }  // Permet n'importe quoi
}
```

### 4. Documenter les types et contraintes

```javascript
// ✅ BON : Documentation claire
{
  temperature: {
    bsonType: "double",
    minimum: -273.15,
    maximum: 5778,
    description: "Température en Kelvin. Min = zéro absolu, Max = surface du soleil"
  }
}

// ❌ ÉVITER : Pas de documentation
{
  temperature: {
    bsonType: "double",
    minimum: -273.15,
    maximum: 5778
  }
}
```

### 5. Gérer les valeurs nulles explicitement

```javascript
// ✅ BON : Autoriser null explicitement
{
  date_fin: {
    bsonType: ["date", "null"],
    description: "Date de fin (null si en cours)"
  }
}

// ❌ AMBIGU : Type unique sans gestion de null
{
  date_fin: {
    bsonType: "date"  // Requis ou facultatif ?
  }
}
```

---

## ⚠️ Pièges courants

### 1. Confusion entre types numériques

```javascript
// ❌ ERREUR : Type strict ne correspond pas
db.produits.insertOne({
  quantite: 5.0  // Double
})
// Si le schéma attend un int, ça échoue !

// ✅ SOLUTION : Types cohérents
db.produits.insertOne({
  quantite: 5  // Int
})

// OU accepter plusieurs types
{
  quantite: {
    bsonType: ["int", "double"]
  }
}
```

### 2. String vs Number

```javascript
// ❌ ERREUR FRÉQUENTE
db.utilisateurs.insertOne({
  age: "30"  // String !
})
// Si le schéma attend un int

// ✅ CORRECT
db.utilisateurs.insertOne({
  age: 30  // Number
})
```

### 3. Oublier l'échappement dans les regex

```javascript
// ❌ INCORRECT
pattern: "\d{5}"  // Ne marche pas !

// ✅ CORRECT
pattern: "\\d{5}"  // Backslash échappé
```

### 4. ObjectId en string

```javascript
// ❌ ERREUR
db.articles.insertOne({
  auteur_id: "507f1f77bcf86cd799439011"  // String !
})

// ✅ CORRECT
db.articles.insertOne({
  auteur_id: ObjectId("507f1f77bcf86cd799439011")  // ObjectId
})
```

### 5. Date vs String

```javascript
// ❌ ERREUR
db.evenements.insertOne({
  date: "2025-01-15"  // String !
})

// ✅ CORRECT
db.evenements.insertOne({
  date: new Date("2025-01-15")  // Date
})
```

---

## 🎓 Résumé

### Types BSON principaux

| Type | Usage | Validation clé |
|------|-------|----------------|
| `string` | Texte | `minLength`, `maxLength`, `pattern`, `enum` |
| `int` | Petit entier | `minimum`, `maximum`, `multipleOf` |
| `long` | Grand entier | `minimum`, `maximum` |
| `double` | Décimal approx. | `minimum`, `maximum` |
| `decimal` | Décimal exact | `minimum`, `maximum` |
| `bool` | Vrai/Faux | Pas de contrainte |
| `date` | Date/Heure | Pas de contrainte directe (utiliser `$expr`) |
| `objectId` | ID unique | Pas de contrainte |
| `array` | Tableau | `minItems`, `maxItems`, `uniqueItems`, `items` |
| `object` | Document | `required`, `properties` |
| `null` | Valeur nulle | Combiner avec autre type |

### Checklist de validation

✅ **Choisir le bon type** :
- [ ] int pour compteurs et petits entiers
- [ ] long pour grands entiers
- [ ] double pour mesures approximatives
- [ ] decimal pour argent et précision exacte
- [ ] string avec pattern pour formats spécifiques

✅ **Ajouter des contraintes** :
- [ ] min/max pour les nombres
- [ ] minLength/maxLength pour les strings
- [ ] pattern (regex) pour les formats
- [ ] enum pour les valeurs fixes
- [ ] required pour les champs obligatoires

✅ **Documenter** :
- [ ] description pour chaque champ
- [ ] Expliquer les contraintes
- [ ] Donner des exemples de valeurs valides

✅ **Tester** :
- [ ] Valeurs valides
- [ ] Valeurs invalides
- [ ] Cas limites
- [ ] Types incorrects

---

## 📚 Dans la prochaine section

Dans la section suivante (7.8), nous verrons la **validation des champs obligatoires** avec des stratégies pour gérer les dépendances entre champs et les champs conditionnels.

---

⏭️ [Validation des champs obligatoires](/07-validation-des-schemas/08-validation-champs-obligatoires.md)
