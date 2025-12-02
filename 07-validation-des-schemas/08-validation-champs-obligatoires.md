🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.8 Validation des champs obligatoires

## 📚 Vue d'ensemble

La **validation des champs obligatoires** vous permet de garantir que certaines informations essentielles sont toujours présentes dans vos documents. C'est l'un des aspects les plus importants de la validation de schéma.

Dans cette section, nous verrons comment définir des champs obligatoires, gérer les dépendances entre champs, et mettre en place des validations conditionnelles.

---

## 🤔 Pourquoi des champs obligatoires ?

### Le problème des données manquantes

Sans validation, vous pouvez vous retrouver avec des documents incomplets :

```javascript
// Sans validation
db.utilisateurs.insertOne({ nom: "Dupont" })
// OK, mais pas d'email !

db.utilisateurs.insertOne({ email: "martin@example.com" })
// OK, mais pas de nom !

db.utilisateurs.insertOne({})
// OK, mais document vide !
```

**Conséquences** :
- ❌ Requêtes qui échouent à cause de données manquantes
- ❌ Bugs dans l'application
- ❌ Expérience utilisateur dégradée
- ❌ Difficultés à exploiter les données

### La solution : le mot-clé `required`

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom", "email"],  // Ces champs DOIVENT être présents
    properties: {
      nom: { bsonType: "string" },
      email: { bsonType: "string" }
    }
  }
}
```

### Analogie avec un formulaire

**Champs obligatoires** = Champs avec astérisque rouge dans un formulaire web
- Vous ne pouvez pas soumettre le formulaire sans les remplir
- L'application vous avertit immédiatement
- Garantit que les données essentielles sont collectées

---

## 📝 Syntaxe de base : `required`

### Structure

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    required: ["champ1", "champ2", "champ3"],  // Tableau de noms de champs
    properties: {
      champ1: { /* validation */ },
      champ2: { /* validation */ },
      champ3: { /* validation */ }
    }
  }
}
```

### Exemple simple

```javascript
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],
      properties: {
        nom: {
          bsonType: "string",
          minLength: 2
        },
        email: {
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        },
        telephone: {
          bsonType: "string"  // Facultatif (pas dans required)
        }
      }
    }
  }
})

// ✅ Valide : nom et email présents
db.utilisateurs.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})

// ✅ Valide : telephone facultatif
db.utilisateurs.insertOne({
  nom: "Martin",
  email: "martin@example.com",
  telephone: "0612345678"
})

// ❌ Invalide : email manquant
db.utilisateurs.insertOne({
  nom: "Bernard"
})
// Erreur : Document failed validation

// ❌ Invalide : nom manquant
db.utilisateurs.insertOne({
  email: "durand@example.com"
})
// Erreur : Document failed validation

// ❌ Invalide : les deux manquants
db.utilisateurs.insertOne({
  telephone: "0612345678"
})
// Erreur : Document failed validation
```

---

## 🎯 Champs obligatoires vs propriétés définies

### Distinction importante

Il est crucial de comprendre la différence entre :
- `required` : Le champ **doit être présent**
- `properties` : Le champ est **défini et validé s'il est présent**

### Exemple de différence

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    required: ["nom"],  // "nom" obligatoire
    properties: {
      nom: { bsonType: "string" },
      email: { bsonType: "string" }  // "email" défini mais pas obligatoire
    }
  }
}

// ✅ Valide : nom présent, email absent
{ nom: "Dupont" }

// ✅ Valide : nom et email présents
{ nom: "Dupont", email: "dupont@example.com" }

// ❌ Invalide : nom absent
{ email: "martin@example.com" }

// ❌ Invalide : nom présent mais de mauvais type
{ nom: 123, email: "test@example.com" }
```

### Champ obligatoire non défini dans properties

**Question** : Que se passe-t-il si un champ est dans `required` mais pas dans `properties` ?

**Réponse** : Le champ doit être présent, mais il n'est **pas validé** (n'importe quel type accepté).

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    required: ["id"],  // Obligatoire
    properties: {
      nom: { bsonType: "string" }
      // "id" pas défini dans properties
    }
  }
}

// ✅ Valide : "id" présent (n'importe quel type)
{ id: 123, nom: "Test" }
{ id: "ABC", nom: "Test" }
{ id: { code: "X" }, nom: "Test" }

// ❌ Invalide : "id" absent
{ nom: "Test" }
```

**Bonne pratique** : Définissez toujours les champs obligatoires dans `properties` pour les valider complètement.

```javascript
// ✅ RECOMMANDÉ
{
  $jsonSchema: {
    bsonType: "object",
    required: ["id", "nom"],
    properties: {
      id: { bsonType: "string" },     // Défini ET obligatoire
      nom: { bsonType: "string" }     // Défini ET obligatoire
    }
  }
}
```

---

## 🔗 Dépendances entre champs : `dependencies`

### Concept

Parfois, la présence d'un champ **implique** la présence d'autres champs.

**Exemple** : Si un utilisateur fournit une adresse de livraison, il doit fournir tous les éléments (rue, ville, code postal).

### Syntaxe

```javascript
{
  $jsonSchema: {
    bsonType: "object",
    properties: {
      champ1: { bsonType: "string" },
      champ2: { bsonType: "string" },
      champ3: { bsonType: "string" }
    },
    dependencies: {
      champ1: ["champ2", "champ3"]
      // Si champ1 présent, alors champ2 et champ3 obligatoires
    }
  }
}
```

### Exemple : Informations de paiement

```javascript
db.createCollection("paiements", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["montant"],
      properties: {
        montant: {
          bsonType: "decimal",
          minimum: 0
        },
        carte_credit: {
          bsonType: "string"
        },
        date_expiration: {
          bsonType: "string"
        },
        cvv: {
          bsonType: "string"
        }
      },
      dependencies: {
        carte_credit: ["date_expiration", "cvv"]
        // Si carte_credit présente, date_expiration et cvv obligatoires
      }
    }
  }
})

// ✅ Valide : pas de paiement par carte
db.paiements.insertOne({
  montant: NumberDecimal("50.00")
})

// ✅ Valide : paiement par carte avec tous les champs
db.paiements.insertOne({
  montant: NumberDecimal("100.00"),
  carte_credit: "1234567890123456",
  date_expiration: "12/25",
  cvv: "123"
})

// ❌ Invalide : carte sans CVV
db.paiements.insertOne({
  montant: NumberDecimal("75.00"),
  carte_credit: "1234567890123456",
  date_expiration: "12/25"
  // cvv manquant !
})

// ❌ Invalide : carte sans date d'expiration
db.paiements.insertOne({
  montant: NumberDecimal("75.00"),
  carte_credit: "1234567890123456",
  cvv: "123"
  // date_expiration manquante !
})
```

### Exemple : Adresse complète

```javascript
db.createCollection("clients", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string" },
        rue: { bsonType: "string" },
        ville: { bsonType: "string" },
        code_postal: { bsonType: "string" }
      },
      dependencies: {
        rue: ["ville", "code_postal"],
        ville: ["rue", "code_postal"],
        code_postal: ["rue", "ville"]
        // Si un élément d'adresse présent, tous doivent l'être
      }
    }
  }
})

// ✅ Valide : pas d'adresse
db.clients.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})

// ✅ Valide : adresse complète
db.clients.insertOne({
  nom: "Martin",
  email: "martin@example.com",
  rue: "10 rue de la Paix",
  ville: "Paris",
  code_postal: "75001"
})

// ❌ Invalide : adresse incomplète
db.clients.insertOne({
  nom: "Bernard",
  email: "bernard@example.com",
  rue: "20 avenue Victor Hugo"
  // ville et code_postal manquants
})
```

### Dépendances multiples

Un champ peut dépendre de plusieurs autres champs :

```javascript
{
  dependencies: {
    livraison_express: ["adresse_complete", "telephone", "date_souhaitee"]
    // Si livraison express, ces 3 champs obligatoires
  }
}
```

---

## 🔀 Champs obligatoires conditionnels avec `oneOf`

### Concept

Parfois, vous voulez que **au moins un** champ parmi plusieurs soit présent, ou que différentes combinaisons soient valides selon un contexte.

### Exemple : Contact (email OU téléphone)

```javascript
db.createCollection("contacts", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom"],
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string" },
        telephone: { bsonType: "string" }
      },
      oneOf: [
        // Option 1 : email présent
        {
          required: ["email"]
        },
        // Option 2 : téléphone présent
        {
          required: ["telephone"]
        },
        // Option 3 : les deux présents
        {
          required: ["email", "telephone"]
        }
      ]
    }
  }
})

// ✅ Valide : email seulement
db.contacts.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})

// ✅ Valide : téléphone seulement
db.contacts.insertOne({
  nom: "Martin",
  telephone: "0612345678"
})

// ✅ Valide : les deux
db.contacts.insertOne({
  nom: "Bernard",
  email: "bernard@example.com",
  telephone: "0698765432"
})

// ❌ Invalide : ni email ni téléphone
db.contacts.insertOne({
  nom: "Durand"
})
```

**Note** : Pour une validation plus simple "email OU téléphone", utilisez plutôt `anyOf` :

```javascript
{
  anyOf: [
    { required: ["email"] },
    { required: ["telephone"] }
  ]
}
```

### Exemple : Type de document polymorphe

```javascript
db.createCollection("vehicules", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["type"],
      properties: {
        type: { enum: ["voiture", "moto"] }
      },
      oneOf: [
        // Si type = voiture
        {
          properties: {
            type: { const: "voiture" }
          },
          required: ["type", "nombre_portes", "nombre_places"]
        },
        // Si type = moto
        {
          properties: {
            type: { const: "moto" }
          },
          required: ["type", "cylindree", "type_permis"]
        }
      ]
    }
  }
})

// ✅ Valide : voiture avec champs requis
db.vehicules.insertOne({
  type: "voiture",
  nombre_portes: 4,
  nombre_places: 5
})

// ✅ Valide : moto avec champs requis
db.vehicules.insertOne({
  type: "moto",
  cylindree: 650,
  type_permis: "A2"
})

// ❌ Invalide : voiture sans nombre_places
db.vehicules.insertOne({
  type: "voiture",
  nombre_portes: 4
})

// ❌ Invalide : moto sans cylindree
db.vehicules.insertOne({
  type: "moto",
  type_permis: "A"
})
```

---

## 📦 Champs obligatoires dans les objets imbriqués

### Objets de premier niveau

Pour des objets imbriqués, définissez `required` **à l'intérieur** de la définition de l'objet.

```javascript
db.createCollection("entreprises", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "adresse"],  // adresse obligatoire
      properties: {
        nom: {
          bsonType: "string"
        },
        adresse: {
          bsonType: "object",
          required: ["rue", "ville", "code_postal"],  // Champs obligatoires dans adresse
          properties: {
            rue: { bsonType: "string" },
            complement: { bsonType: "string" },  // Facultatif
            ville: { bsonType: "string" },
            code_postal: { bsonType: "string" }
          }
        }
      }
    }
  }
})

// ✅ Valide : adresse complète
db.entreprises.insertOne({
  nom: "TechCorp",
  adresse: {
    rue: "10 avenue des Champs",
    ville: "Paris",
    code_postal: "75008"
  }
})

// ✅ Valide : avec complément facultatif
db.entreprises.insertOne({
  nom: "WebCorp",
  adresse: {
    rue: "15 rue Victor Hugo",
    complement: "Bâtiment B",
    ville: "Lyon",
    code_postal: "69001"
  }
})

// ❌ Invalide : ville manquante dans adresse
db.entreprises.insertOne({
  nom: "DataCorp",
  adresse: {
    rue: "20 boulevard Haussmann",
    code_postal: "75009"
  }
})

// ❌ Invalide : adresse complètement absente
db.entreprises.insertOne({
  nom: "CloudCorp"
})
```

### Objets imbriqués profonds

```javascript
db.createCollection("projets", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "responsable"],
      properties: {
        nom: { bsonType: "string" },
        responsable: {
          bsonType: "object",
          required: ["nom", "contact"],
          properties: {
            nom: { bsonType: "string" },
            contact: {
              bsonType: "object",
              required: ["email"],  // Au moins email obligatoire
              properties: {
                email: { bsonType: "string" },
                telephone: { bsonType: "string" }  // Facultatif
              }
            }
          }
        }
      }
    }
  }
})

// ✅ Valide : structure complète
db.projets.insertOne({
  nom: "Site Web",
  responsable: {
    nom: "Dupont",
    contact: {
      email: "dupont@example.com",
      telephone: "0612345678"
    }
  }
})

// ✅ Valide : téléphone facultatif
db.projets.insertOne({
  nom: "Application Mobile",
  responsable: {
    nom: "Martin",
    contact: {
      email: "martin@example.com"
    }
  }
})

// ❌ Invalide : email manquant dans contact
db.projets.insertOne({
  nom: "API REST",
  responsable: {
    nom: "Bernard",
    contact: {
      telephone: "0698765432"
    }
  }
})
```

---

## 📊 Champs obligatoires dans les tableaux

### Tableaux d'objets

Chaque objet du tableau peut avoir ses propres champs obligatoires.

```javascript
db.createCollection("commandes", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["numero", "articles"],
      properties: {
        numero: { bsonType: "string" },
        articles: {
          bsonType: "array",
          minItems: 1,
          items: {
            bsonType: "object",
            required: ["produit_id", "quantite", "prix"],  // Obligatoires dans chaque article
            properties: {
              produit_id: { bsonType: "objectId" },
              nom: { bsonType: "string" },  // Facultatif
              quantite: { bsonType: "int", minimum: 1 },
              prix: { bsonType: "decimal", minimum: 0 }
            }
          }
        }
      }
    }
  }
})

// ✅ Valide : tous les articles conformes
db.commandes.insertOne({
  numero: "CMD-001",
  articles: [
    {
      produit_id: ObjectId("507f1f77bcf86cd799439011"),
      nom: "Clavier",
      quantite: 2,
      prix: NumberDecimal("29.99")
    },
    {
      produit_id: ObjectId("507f1f77bcf86cd799439012"),
      quantite: 1,
      prix: NumberDecimal("15.50")
    }
  ]
})

// ❌ Invalide : quantite manquante dans un article
db.commandes.insertOne({
  numero: "CMD-002",
  articles: [
    {
      produit_id: ObjectId("507f1f77bcf86cd799439013"),
      prix: NumberDecimal("45.00")
      // quantite manquante !
    }
  ]
})
```

---

## 🎯 Stratégies de gestion des champs obligatoires

### Stratégie 1 : Minimum vital au début

Commencez avec très peu de champs obligatoires, puis ajoutez progressivement.

```javascript
// Phase 1 : Strict minimum
{
  required: ["nom", "email"]
}

// Phase 2 : Ajout progressif (quelques mois plus tard)
{
  required: ["nom", "email", "date_inscription"]
}

// Phase 3 : Plus complet (après migration des données)
{
  required: ["nom", "email", "date_inscription", "statut"]
}
```

### Stratégie 2 : Champs obligatoires par contexte

Différentes collections pour différents contextes :

```javascript
// Collection "brouillons" : peu de champs obligatoires
db.createCollection("articles_brouillons", {
  validator: {
    $jsonSchema: {
      required: ["titre"]  // Juste le titre
    }
  }
})

// Collection "publies" : beaucoup de champs obligatoires
db.createCollection("articles_publies", {
  validator: {
    $jsonSchema: {
      required: ["titre", "contenu", "auteur", "date_publication", "categories"]
    }
  }
})
```

### Stratégie 3 : Validation en mode `moderate` pendant la migration

Lors de l'ajout de nouveaux champs obligatoires :

```javascript
// Étape 1 : Ajouter le champ comme facultatif
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      required: ["nom", "email"],
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string" },
        telephone: { bsonType: "string" }  // Nouveau, facultatif
      }
    }
  }
})

// Étape 2 : Migration des données (ajouter valeur par défaut)
db.utilisateurs.updateMany(
  { telephone: { $exists: false } },
  { $set: { telephone: "non_renseigne" } }
)

// Étape 3 : Rendre obligatoire
db.runCommand({
  collMod: "utilisateurs",
  validator: {
    $jsonSchema: {
      required: ["nom", "email", "telephone"],  // Maintenant obligatoire
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string" },
        telephone: { bsonType: "string" }
      }
    }
  }
})
```

---

## 💡 Bonnes pratiques

### 1. Commencer minimal, étendre progressivement

```javascript
// ✅ BON : Commencer simple
{
  required: ["nom", "email"]
}

// ❌ ÉVITER : Trop de champs obligatoires au début
{
  required: ["nom", "prenom", "email", "telephone", "adresse", "ville",
             "code_postal", "pays", "date_naissance", "profession"]
}
```

### 2. Documenter pourquoi un champ est obligatoire

```javascript
{
  required: ["email"],
  properties: {
    email: {
      bsonType: "string",
      description: "Email obligatoire - utilisé pour la connexion et la récupération de mot de passe"
    }
  }
}
```

### 3. Différencier données essentielles vs utiles

**Essentielles** (obligatoires) :
- Données sans lesquelles le document n'a pas de sens
- Données nécessaires au fonctionnement de l'application
- Identifiants, clés étrangères

**Utiles** (facultatives) :
- Informations complémentaires
- Métadonnées
- Optimisations

```javascript
// Produit e-commerce
{
  required: ["nom", "prix", "stock"],  // Essentiels
  properties: {
    nom: { bsonType: "string" },
    prix: { bsonType: "decimal" },
    stock: { bsonType: "int" },
    description: { bsonType: "string" },  // Utile mais pas essentiel
    couleur: { bsonType: "string" },      // Utile mais pas essentiel
    poids: { bsonType: "double" }         // Utile mais pas essentiel
  }
}
```

### 4. Utiliser dependencies pour les groupes logiques

```javascript
// ✅ BON : Dependencies pour cohérence
{
  dependencies: {
    rue: ["ville", "code_postal"],
    ville: ["rue", "code_postal"],
    code_postal: ["rue", "ville"]
  }
}

// ❌ ÉVITER : Rendre tous obligatoires alors que c'est un groupe optionnel
{
  required: ["rue", "ville", "code_postal"]
  // Forcé même si l'utilisateur ne veut pas donner d'adresse
}
```

### 5. Valider également les champs obligatoires

```javascript
// ✅ BON : Obligatoire ET validé
{
  required: ["email"],
  properties: {
    email: {
      bsonType: "string",
      pattern: "^.+@.+\\..+$"
    }
  }
}

// ❌ INSUFFISANT : Obligatoire mais pas validé
{
  required: ["email"]
  // Pas de définition dans properties = n'importe quel type accepté !
}
```

### 6. Tester les cas limites

```javascript
// Tests à effectuer :
// - Document sans aucun champ obligatoire
// - Document avec un seul champ obligatoire
// - Document avec tous les champs obligatoires
// - Document avec champs obligatoires mais mauvais type
// - Document avec champs obligatoires null
```

---

## ⚠️ Pièges courants

### 1. Oublier de mettre à jour `required` lors de modifications

```javascript
// ❌ ERREUR : Champ retiré de properties mais pas de required
{
  required: ["nom", "ancien_champ"],  // ancien_champ n'existe plus !
  properties: {
    nom: { bsonType: "string" }
    // ancien_champ supprimé
  }
}

// ✅ CORRECT : Cohérence entre required et properties
{
  required: ["nom"],
  properties: {
    nom: { bsonType: "string" }
  }
}
```

### 2. Confondre présence et valeur non nulle

```javascript
// Attention : null satisfait "required" !
{
  required: ["statut"]
}

// ✅ Accepté car "statut" est présent (même si null)
{ statut: null }

// Pour interdire null :
{
  required: ["statut"],
  properties: {
    statut: {
      bsonType: "string",  // Pas "null" dans les types acceptés
      enum: ["actif", "inactif"]
    }
  }
}
```

### 3. Champs obligatoires dans objets imbriqués au mauvais niveau

```javascript
// ❌ ERREUR : required au mauvais endroit
{
  required: ["adresse", "rue", "ville"],  // "rue" et "ville" ne sont pas au niveau racine !
  properties: {
    adresse: {
      bsonType: "object",
      properties: {
        rue: { bsonType: "string" },
        ville: { bsonType: "string" }
      }
    }
  }
}

// ✅ CORRECT : required dans l'objet imbriqué
{
  required: ["adresse"],
  properties: {
    adresse: {
      bsonType: "object",
      required: ["rue", "ville"],  // Ici !
      properties: {
        rue: { bsonType: "string" },
        ville: { bsonType: "string" }
      }
    }
  }
}
```

### 4. Rendre trop de champs obligatoires trop tôt

```javascript
// ❌ PROBLÈME : Bloque les utilisateurs
{
  required: [
    "nom", "prenom", "email", "telephone",
    "adresse_complete", "date_naissance", "profession",
    "entreprise", "numero_tva"
  ]
}
// Trop de contraintes = utilisateurs abandonnent

// ✅ MIEUX : Progressif
// Phase 1 : Minimum
{
  required: ["email"]
}

// Phase 2 : Ajout progressif selon besoins
{
  required: ["email", "nom"]
}
```

### 5. Ne pas tester les dependencies

```javascript
// ❌ Dependencies non testées
{
  dependencies: {
    carte_credit: ["cvv", "date_expiration"]
  }
}

// Tester tous les cas :
// ✅ Ni carte ni dépendances
// ✅ Carte avec toutes les dépendances
// ❌ Carte sans une dépendance
// ❌ Carte sans aucune dépendance
```

---

## 📚 Exemples complets

### Exemple 1 : Inscription utilisateur progressive

```javascript
// Étape 1 : Inscription minimale
db.createCollection("utilisateurs_inscription", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "mot_de_passe"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Email pour la connexion"
        },
        mot_de_passe: {
          bsonType: "string",
          minLength: 8,
          description: "Mot de passe (min 8 caractères)"
        },
        nom: {
          bsonType: "string",
          description: "Nom (facultatif à l'inscription)"
        },
        prenom: {
          bsonType: "string",
          description: "Prénom (facultatif à l'inscription)"
        },
        date_inscription: {
          bsonType: "date",
          description: "Date d'inscription (auto)"
        }
      }
    }
  }
})

// Étape 2 : Profil complet (après validation email)
db.createCollection("utilisateurs_complets", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "mot_de_passe", "nom", "prenom", "date_naissance"],
      properties: {
        email: { bsonType: "string" },
        mot_de_passe: { bsonType: "string" },
        nom: { bsonType: "string", minLength: 2 },
        prenom: { bsonType: "string", minLength: 2 },
        date_naissance: { bsonType: "date" },
        telephone: { bsonType: "string" },
        adresse: {
          bsonType: "object",
          required: ["rue", "ville", "code_postal"],
          properties: {
            rue: { bsonType: "string" },
            ville: { bsonType: "string" },
            code_postal: { bsonType: "string" }
          }
        }
      }
    }
  }
})
```

### Exemple 2 : Commande e-commerce avec dépendances

```javascript
db.createCollection("commandes_ecommerce", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["numero", "client_id", "articles", "statut"],
      properties: {
        numero: {
          bsonType: "string",
          pattern: "^CMD-[0-9]{8}$"
        },
        client_id: {
          bsonType: "objectId"
        },
        articles: {
          bsonType: "array",
          minItems: 1,
          items: {
            bsonType: "object",
            required: ["produit_id", "nom", "quantite", "prix_unitaire"],
            properties: {
              produit_id: { bsonType: "objectId" },
              nom: { bsonType: "string" },
              quantite: { bsonType: "int", minimum: 1 },
              prix_unitaire: { bsonType: "decimal", minimum: 0 }
            }
          }
        },
        statut: {
          enum: ["en_attente", "validee", "expediee", "livree", "annulee"]
        },
        adresse_livraison: {
          bsonType: "object",
          required: ["nom_complet", "rue", "ville", "code_postal", "pays"],
          properties: {
            nom_complet: { bsonType: "string" },
            rue: { bsonType: "string" },
            complement: { bsonType: "string" },
            ville: { bsonType: "string" },
            code_postal: { bsonType: "string" },
            pays: { bsonType: "string" }
          }
        },
        numero_suivi: { bsonType: "string" },
        date_expedition: { bsonType: "date" }
      },
      dependencies: {
        numero_suivi: ["date_expedition"],
        date_expedition: ["numero_suivi"]
        // Si l'un est présent, l'autre doit l'être
      }
    }
  }
})
```

### Exemple 3 : Événement avec participants

```javascript
db.createCollection("evenements", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["titre", "date_debut", "date_fin", "type", "organisateur"],
      properties: {
        titre: {
          bsonType: "string",
          minLength: 5,
          maxLength: 200
        },
        description: {
          bsonType: "string",
          maxLength: 2000
        },
        date_debut: {
          bsonType: "date"
        },
        date_fin: {
          bsonType: "date"
        },
        type: {
          enum: ["conference", "atelier", "webinaire", "networking"]
        },
        organisateur: {
          bsonType: "object",
          required: ["nom", "email"],
          properties: {
            nom: { bsonType: "string" },
            email: { bsonType: "string" },
            telephone: { bsonType: "string" }
          }
        },
        lieu: {
          bsonType: "object",
          required: ["nom", "adresse"],
          properties: {
            nom: { bsonType: "string" },
            adresse: {
              bsonType: "object",
              required: ["rue", "ville", "code_postal"],
              properties: {
                rue: { bsonType: "string" },
                ville: { bsonType: "string" },
                code_postal: { bsonType: "string" }
              }
            }
          }
        },
        lien_visio: {
          bsonType: "string",
          pattern: "^https?:\\/\\/.+"
        },
        participants: {
          bsonType: "array",
          items: {
            bsonType: "object",
            required: ["nom", "email"],
            properties: {
              nom: { bsonType: "string" },
              email: { bsonType: "string" },
              statut: { enum: ["inscrit", "confirme", "present", "absent"] }
            }
          }
        }
      },
      // Si événement physique, lieu obligatoire
      // Si événement en ligne, lien_visio obligatoire
      oneOf: [
        {
          properties: { type: { enum: ["conference", "atelier", "networking"] } },
          required: ["lieu"]
        },
        {
          properties: { type: { const: "webinaire" } },
          required: ["lien_visio"]
        }
      ]
    }
  }
})
```

---

## 🎓 Résumé

### Concepts clés

| Concept | Fonction | Syntaxe |
|---------|----------|---------|
| `required` | Champs obligatoires | `required: ["champ1", "champ2"]` |
| `dependencies` | Dépendances entre champs | `dependencies: { champ1: ["champ2"] }` |
| `oneOf` | Plusieurs combinaisons valides | `oneOf: [{ required: [...] }]` |
| `anyOf` | Au moins une condition | `anyOf: [{ required: [...] }]` |

### Checklist

✅ **Définition** :
- [ ] Champs essentiels identifiés
- [ ] `required` défini au bon niveau (racine ou objet imbriqué)
- [ ] Champs obligatoires aussi définis dans `properties`
- [ ] Validation complète de chaque champ obligatoire

✅ **Dépendances** :
- [ ] Dependencies définies pour groupes logiques
- [ ] Toutes les combinaisons testées

✅ **Documentation** :
- [ ] Raison de chaque champ obligatoire documentée
- [ ] Impact sur utilisateurs évalué

✅ **Migration** :
- [ ] Plan de migration pour ajout de champs obligatoires
- [ ] Mode `moderate` ou `warn` pendant transition
- [ ] Tests sur environnement de staging

### Points clés

- ✅ `required` liste les champs qui **doivent être présents**
- ✅ Définir les champs obligatoires dans `properties` pour validation complète
- ✅ `dependencies` pour champs interdépendants
- ✅ `oneOf` / `anyOf` pour validations conditionnelles
- ✅ Champs obligatoires dans objets imbriqués : définir au bon niveau
- ✅ Commencer minimal, étendre progressivement
- ✅ `null` satisfait `required` (présent mais nul)

---

## 📚 Dans la prochaine section

Dans la section suivante (7.9), nous verrons la **validation personnalisée avec `$expr`** qui permet des validations complexes et des règles métier avancées.

---

⏭️ [Validation personnalisée avec $expr](/07-validation-des-schemas/09-validation-personnalisee-expr.md)
