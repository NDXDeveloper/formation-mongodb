🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.2 JSON Schema dans MongoDB

## 📚 Vue d'ensemble

JSON Schema est un **langage de validation** qui permet de décrire la structure attendue de vos documents JSON (ou BSON dans MongoDB). C'est comme créer un "modèle" ou un "plan" que vos données doivent suivre.

MongoDB utilise une version adaptée de JSON Schema pour valider les documents dans vos collections.

---

## 🤔 Qu'est-ce que JSON Schema ?

### Définition simple

JSON Schema est un **standard universel** qui permet de définir :
- Quels champs doivent exister dans un document
- Quel type de données chaque champ doit contenir
- Quelles contraintes s'appliquent aux valeurs

### Analogie avec la construction

Imaginez la construction d'une maison :

📝 **JSON Schema** = Les plans d'architecture
- Définit où vont les murs, les portes, les fenêtres
- Spécifie les dimensions, les matériaux
- Décrit les contraintes à respecter

🏠 **Document MongoDB** = La maison construite
- Doit correspondre aux plans
- Est rejetée si elle ne respecte pas les spécifications

### Exemple visuel simple

Voici un document et son schéma :

**Document (données réelles)** :
```json
{
  "nom": "Dupont",
  "age": 30,
  "email": "dupont@example.com"
}
```

**Schéma (règles de validation)** :
```json
{
  "required": ["nom", "email"],
  "properties": {
    "nom": { "bsonType": "string" },
    "age": { "bsonType": "int", "minimum": 0 },
    "email": { "bsonType": "string" }
  }
}
```

---

## 🔧 JSON Schema dans MongoDB

### La différence : BSON vs JSON

MongoDB utilise **BSON** (Binary JSON) au lieu de JSON classique. Le schéma s'appelle donc `$jsonSchema`, mais il valide des documents **BSON**.

**Pourquoi c'est important ?**

BSON a plus de types de données que JSON standard :

| JSON | BSON (MongoDB) |
|------|----------------|
| `number` | `int`, `long`, `double`, `decimal` |
| `string` | `string` |
| `boolean` | `bool` |
| `null` | `null` |
| `array` | `array` |
| `object` | `object` |
| ❌ | `date`, `objectId`, `timestamp`, `binData`, etc. |

### Syntaxe de base

Pour utiliser JSON Schema dans MongoDB, vous l'enveloppez dans `$jsonSchema` :

```javascript
db.createCollection("maCollection", {
  validator: {
    $jsonSchema: {
      // Votre schéma ici
    }
  }
})
```

---

## 📖 Structure d'un schéma MongoDB

### Les éléments principaux

Un schéma MongoDB se compose de plusieurs parties :

```javascript
{
  $jsonSchema: {
    bsonType: "object",              // Type du document racine
    required: ["champ1", "champ2"],  // Champs obligatoires
    properties: {                     // Définition de chaque champ
      champ1: { /* règles */ },
      champ2: { /* règles */ }
    },
    additionalProperties: false      // Autoriser d'autres champs ?
  }
}
```

### 1. `bsonType` - Le type du document

Spécifie que le document racine est un objet :

```javascript
{
  $jsonSchema: {
    bsonType: "object"  // Le document est un objet JSON
  }
}
```

### 2. `required` - Champs obligatoires

Liste les champs qui **doivent** être présents :

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email", "dateInscription"]
    // Ces 3 champs DOIVENT être présents
  }
}
```

**Important** : Un champ dans `required` mais pas dans `properties` est accepté mais non validé.

### 3. `properties` - Définition des champs

Décrit les règles pour chaque champ :

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      nom: {
        bsonType: "string",
        description: "Le nom de l'utilisateur"
      },
      age: {
        bsonType: "int",
        minimum: 18,
        maximum: 120,
        description: "L'âge doit être entre 18 et 120"
      }
    }
  }
}
```

### 4. `additionalProperties` - Champs supplémentaires

Contrôle si des champs non définis sont autorisés :

```javascript
// Autoriser d'autres champs (par défaut)
additionalProperties: true

// Interdire d'autres champs (plus strict)
additionalProperties: false
```

**Exemple** :

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      nom: { bsonType: "string" },
      age: { bsonType: "int" }
    },
    additionalProperties: false
  }
}

// ✅ Accepté : nom et age seulement
{ nom: "Dupont", age: 30 }

// ❌ Refusé : le champ "ville" n'est pas autorisé
{ nom: "Dupont", age: 30, ville: "Paris" }
```

---

## 🎯 Types BSON disponibles

### Types simples

| Type BSON | Description | Exemple |
|-----------|-------------|---------|
| `string` | Chaîne de caractères | `"Bonjour"` |
| `int` | Entier 32 bits | `42` |
| `long` | Entier 64 bits | `9223372036854775807` |
| `double` | Nombre décimal | `3.14159` |
| `decimal` | Décimal haute précision | `NumberDecimal("99.99")` |
| `bool` | Booléen | `true` ou `false` |
| `null` | Valeur nulle | `null` |

### Types complexes

| Type BSON | Description | Exemple |
|-----------|-------------|---------|
| `object` | Objet/document imbriqué | `{ rue: "...", ville: "..." }` |
| `array` | Tableau | `[1, 2, 3, 4]` |
| `objectId` | Identifiant MongoDB | `ObjectId("507f1f77...")` |
| `date` | Date/heure | `ISODate("2025-01-01")` |
| `timestamp` | Horodatage interne | `Timestamp(1, 0)` |
| `binData` | Données binaires | `BinData(0, "...")` |
| `regex` | Expression régulière | `/^[A-Z]/` |

### Utilisation des types BSON

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      _id: {
        bsonType: "objectId",
        description: "Identifiant unique"
      },
      nom: {
        bsonType: "string",
        description: "Nom de l'utilisateur"
      },
      age: {
        bsonType: "int",
        description: "Âge en années"
      },
      solde: {
        bsonType: "decimal",
        description: "Solde du compte"
      },
      actif: {
        bsonType: "bool",
        description: "Compte actif ou non"
      },
      dateCreation: {
        bsonType: "date",
        description: "Date de création du compte"
      }
    }
  }
}
```

---

## 📝 Exemples progressifs

### Exemple 1 : Schéma très simple

Validons uniquement le type d'un champ :

```javascript
db.createCollection("messages", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        texte: {
          bsonType: "string"
        }
      }
    }
  }
})

// ✅ Valide
db.messages.insertOne({ texte: "Bonjour" })

// ❌ Invalide : "texte" doit être une chaîne
db.messages.insertOne({ texte: 123 })
```

### Exemple 2 : Champs obligatoires

Ajoutons des champs requis :

```javascript
db.createCollection("articles", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["titre", "contenu", "auteur"],
      properties: {
        titre: {
          bsonType: "string",
          description: "Le titre est obligatoire"
        },
        contenu: {
          bsonType: "string",
          description: "Le contenu est obligatoire"
        },
        auteur: {
          bsonType: "string",
          description: "L'auteur est obligatoire"
        }
      }
    }
  }
})

// ✅ Valide : tous les champs requis présents
db.articles.insertOne({
  titre: "Introduction à MongoDB",
  contenu: "MongoDB est une base de données NoSQL...",
  auteur: "Marie Dupont"
})

// ❌ Invalide : champ "auteur" manquant
db.articles.insertOne({
  titre: "Un autre article",
  contenu: "Contenu de l'article..."
})
```

### Exemple 3 : Contraintes sur les valeurs

Ajoutons des limites et des formats :

```javascript
db.createCollection("employes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email", "age", "salaire"],
      properties: {
        nom: {
          bsonType: "string",
          minLength: 2,
          maxLength: 50,
          description: "Nom entre 2 et 50 caractères"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Email au format valide"
        },
        age: {
          bsonType: "int",
          minimum: 18,
          maximum: 65,
          description: "Âge entre 18 et 65 ans"
        },
        salaire: {
          bsonType: "double",
          minimum: 0,
          description: "Salaire positif"
        }
      }
    }
  }
})

// ✅ Valide : toutes les contraintes respectées
db.employes.insertOne({
  nom: "Martin",
  email: "martin@example.com",
  age: 35,
  salaire: 45000.00
})

// ❌ Invalide : âge hors limites
db.employes.insertOne({
  nom: "Jeune",
  email: "jeune@example.com",
  age: 16,
  salaire: 25000.00
})

// ❌ Invalide : email mal formaté
db.employes.insertOne({
  nom: "Durand",
  email: "mauvais-email",
  age: 30,
  salaire: 40000.00
})
```

### Exemple 4 : Documents imbriqués

Validons des objets à l'intérieur de documents :

```javascript
db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["numero", "client", "adresseLivraison"],
      properties: {
        numero: {
          bsonType: "string",
          description: "Numéro de commande"
        },
        client: {
          bsonType: "string",
          description: "Nom du client"
        },
        adresseLivraison: {
          bsonType: "object",
          required: ["rue", "ville", "codePostal"],
          properties: {
            rue: {
              bsonType: "string",
              description: "Nom de la rue"
            },
            ville: {
              bsonType: "string",
              description: "Nom de la ville"
            },
            codePostal: {
              bsonType: "string",
              pattern: "^[0-9]{5}$",
              description: "Code postal à 5 chiffres"
            }
          }
        }
      }
    }
  }
})

// ✅ Valide : objet imbriqué correct
db.commandes.insertOne({
  numero: "CMD-2025-001",
  client: "Marie Dupont",
  adresseLivraison: {
    rue: "15 rue de la Paix",
    ville: "Paris",
    codePostal: "75001"
  }
})

// ❌ Invalide : code postal invalide (6 chiffres)
db.commandes.insertOne({
  numero: "CMD-2025-002",
  client: "Jean Martin",
  adresseLivraison: {
    rue: "20 avenue Victor Hugo",
    ville: "Lyon",
    codePostal: "690001"  // Erreur : doit être 5 chiffres
  }
})
```

### Exemple 5 : Tableaux

Validons des tableaux et leurs éléments :

```javascript
db.createCollection("etudiants", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "notes"],
      properties: {
        nom: {
          bsonType: "string",
          description: "Nom de l'étudiant"
        },
        notes: {
          bsonType: "array",
          minItems: 1,
          maxItems: 10,
          items: {
            bsonType: "double",
            minimum: 0,
            maximum: 20,
            description: "Note entre 0 et 20"
          },
          description: "Liste de notes (1 à 10 notes)"
        }
      }
    }
  }
})

// ✅ Valide : tableau avec notes valides
db.etudiants.insertOne({
  nom: "Sophie Bernard",
  notes: [15.5, 18.0, 12.5, 16.0]
})

// ❌ Invalide : une note dépasse 20
db.etudiants.insertOne({
  nom: "Paul Leroux",
  notes: [15.0, 22.0, 18.0]  // Erreur : 22 > 20
})

// ❌ Invalide : tableau vide
db.etudiants.insertOne({
  nom: "Alice Robert",
  notes: []  // Erreur : minItems = 1
})
```

---

## 🔍 Propriétés de validation courantes

### Pour les chaînes (`string`)

```javascript
{
  bsonType: "string",
  minLength: 5,           // Longueur minimale
  maxLength: 100,         // Longueur maximale
  pattern: "^[A-Z].*",    // Expression régulière
  enum: ["actif", "inactif", "suspendu"]  // Valeurs autorisées
}
```

### Pour les nombres (`int`, `double`, `decimal`)

```javascript
{
  bsonType: "int",
  minimum: 0,             // Valeur minimale (inclusive)
  maximum: 100,           // Valeur maximale (inclusive)
  exclusiveMinimum: 0,    // Valeur minimale (exclusive)
  exclusiveMaximum: 100,  // Valeur maximale (exclusive)
  multipleOf: 5           // Multiple de 5
}
```

### Pour les tableaux (`array`)

```javascript
{
  bsonType: "array",
  minItems: 1,            // Nombre minimum d'éléments
  maxItems: 10,           // Nombre maximum d'éléments
  uniqueItems: true,      // Tous les éléments doivent être uniques
  items: {                // Type des éléments du tableau
    bsonType: "string"
  }
}
```

### Pour les objets (`object`)

```javascript
{
  bsonType: "object",
  required: ["champ1"],           // Champs obligatoires
  properties: { /* ... */ },      // Définition des propriétés
  additionalProperties: false,    // Interdire champs supplémentaires
  minProperties: 1,               // Nombre minimum de propriétés
  maxProperties: 10               // Nombre maximum de propriétés
}
```

---

## 🎨 Description et documentation

### Le champ `description`

Chaque propriété peut avoir une `description` :

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    required: ["email"],
    properties: {
      email: {
        bsonType: "string",
        pattern: "^.+@.+\\..+$",
        description: "Adresse email valide au format : utilisateur@domaine.com"
      }
    }
  }
}
```

**Avantages** :
- 📖 Documente le schéma pour les autres développeurs
- 🐛 Aide au débogage (apparaît dans les messages d'erreur)
- 📝 Facilite la maintenance du code

### Le champ `title`

Donne un titre court à une propriété :

```javascript
{
  title: "Adresse Email",
  bsonType: "string",
  description: "Email de contact principal de l'utilisateur"
}
```

---

## 💡 Conseils pratiques

### 1. Commencez simple

Ne créez pas un schéma trop complexe dès le début :

```javascript
// ✅ BON : Commencer avec l'essentiel
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

// ❌ ÉVITER : Trop complexe au début
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "prenom", "email", "telephone", "adresse"],
    properties: {
      // 20 propriétés avec contraintes complexes...
    }
  }
}
```

### 2. Documentez vos schémas

Ajoutez toujours des descriptions :

```javascript
{
  age: {
    bsonType: "int",
    minimum: 18,
    maximum: 120,
    description: "Âge de l'utilisateur. Doit être entre 18 et 120 ans."
  }
}
```

### 3. Utilisez `enum` pour les valeurs fixes

Quand un champ a des valeurs limitées :

```javascript
{
  statut: {
    bsonType: "string",
    enum: ["brouillon", "publie", "archive"],
    description: "Statut de l'article : brouillon, publié ou archivé"
  }
}
```

### 4. Validez les formats critiques

Utilisez `pattern` pour les formats importants :

```javascript
{
  telephone: {
    bsonType: "string",
    pattern: "^\\+?[1-9]\\d{1,14}$",
    description: "Numéro de téléphone au format international"
  }
}
```

---

## ⚠️ Pièges courants à éviter

### 1. Oublier d'échapper les backslashes

❌ **Incorrect** :
```javascript
pattern: "^\d{5}$"  // Ne fonctionne pas !
```

✅ **Correct** :
```javascript
pattern: "^\\d{5}$"  // Backslash échappé
```

### 2. Confondre `required` et `properties`

```javascript
// ❌ Erreur : "age" requis mais pas défini dans properties
{
  $jsonSchema: {
    required: ["nom", "age"],
    properties: {
      nom: { bsonType: "string" }
      // "age" manquant !
    }
  }
}

// ✅ Correct : tous les champs requis sont définis
{
  $jsonSchema: {
    required: ["nom", "age"],
    properties: {
      nom: { bsonType: "string" },
      age: { bsonType: "int" }
    }
  }
}
```

### 3. Utiliser le mauvais type BSON

```javascript
// ❌ Incorrect : JSON standard
{
  price: {
    type: "number"  // N'existe pas en BSON !
  }
}

// ✅ Correct : Type BSON
{
  price: {
    bsonType: "double"  // ou "int", "decimal"
  }
}
```

### 4. Oublier `bsonType: "object"` pour la racine

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
    bsonType: "object",  // Important !
    required: ["nom"],
    properties: { /* ... */ }
  }
}
```

---

## 🎓 Résumé

JSON Schema dans MongoDB vous permet de :

| Concept | Fonction |
|---------|----------|
| `$jsonSchema` | Enveloppe principale pour la validation |
| `bsonType` | Définit le type de données BSON |
| `required` | Liste les champs obligatoires |
| `properties` | Décrit les règles pour chaque champ |
| `description` | Documente le schéma |
| `pattern` | Valide les formats avec regex |
| `minimum`/`maximum` | Limite les valeurs numériques |
| `minLength`/`maxLength` | Limite la longueur des chaînes |
| `enum` | Restreint à des valeurs spécifiques |

### Points clés à retenir

- ✅ MongoDB utilise `$jsonSchema` avec des types **BSON** (pas JSON standard)
- ✅ Combinez `required` et `properties` pour une validation complète
- ✅ Utilisez `description` pour documenter vos schémas
- ✅ Commencez simple et ajoutez des contraintes progressivement
- ✅ Testez votre schéma avant de le mettre en production

---

## 📚 Dans la prochaine section

Dans la section suivante, nous approfondirons les **règles de validation** avec `$jsonSchema` et verrons des cas d'usage plus complexes.

---


⏭️ [Règles de validation ($jsonSchema)](/07-validation-des-schemas/03-regles-validation-jsonschema.md)
