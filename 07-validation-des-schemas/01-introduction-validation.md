🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 7.1 Introduction à la validation de schéma

## 📚 Vue d'ensemble

La validation de schéma dans MongoDB permet de définir des règles qui contrôlent la structure et le contenu des documents insérés ou modifiés dans une collection. C'est comme mettre en place des "garde-fous" pour s'assurer que vos données respectent certaines normes de qualité.

---

## 🤔 Pourquoi valider les schémas ?

### MongoDB et la flexibilité des schémas

L'une des grandes forces de MongoDB est sa **flexibilité de schéma** : contrairement aux bases de données relationnelles (SQL), vous n'êtes pas obligé de définir une structure rigide avant d'insérer des données.

**Par exemple**, dans une collection `utilisateurs`, vous pourriez avoir :

```json
// Utilisateur 1
{
  "_id": 1,
  "nom": "Dupont",
  "prenom": "Marie",
  "email": "marie.dupont@example.com"
}

// Utilisateur 2 - structure différente
{
  "_id": 2,
  "nom": "Martin",
  "age": 35,
  "ville": "Paris"
}
```

Ces deux documents ont des champs complètement différents, et MongoDB les accepte sans problème dans la même collection.

### Le problème de la trop grande flexibilité

Cette flexibilité est puissante, mais elle peut causer des problèmes :

- ❌ **Erreurs de saisie** : Un développeur écrit `emial` au lieu de `email`
- ❌ **Types incorrects** : L'âge est stocké comme texte `"35"` au lieu d'un nombre `35`
- ❌ **Champs manquants** : Des documents sans les informations essentielles
- ❌ **Incohérences** : Certains documents avec `telephone`, d'autres avec `tel` ou `phone`

### La solution : la validation de schéma

La validation de schéma vous permet de **définir des règles** que MongoDB vérifiera automatiquement :

- ✅ Quels champs sont **obligatoires**
- ✅ Quels **types de données** sont acceptés pour chaque champ
- ✅ Quelles **valeurs** sont autorisées (ex: âge entre 0 et 150)
- ✅ Quelle **structure** doivent avoir les documents imbriqués

---

## 🎯 Concept de base

### Analogie avec la vie réelle

Imaginez que vous gérez l'inscription à un événement :

**Sans validation** (trop permissif) :
- Certains participants donnent leur âge, d'autres non
- Certains écrivent leur email en majuscules, d'autres en minuscules
- Certains oublient leur nom
- Résultat : données incohérentes et exploitation difficile

**Avec validation** (règles claires) :
- Formulaire avec champs obligatoires (nom, email)
- Format email vérifié
- Âge doit être un nombre
- Résultat : données propres et cohérentes

### Comment ça fonctionne dans MongoDB

Vous définissez un **ensemble de règles** lors de la création d'une collection (ou en modifiant une collection existante). MongoDB vérifiera ensuite automatiquement chaque document avant de l'accepter.

```javascript
// Exemple simple de validation
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      required: ["nom", "email"],
      properties: {
        nom: {
          bsonType: "string",
          description: "Le nom est obligatoire et doit être une chaîne"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Email obligatoire avec format valide"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150,
          description: "L'âge doit être un entier entre 0 et 150"
        }
      }
    }
  }
})
```

---

## 📊 Que peut-on valider ?

### 1. **Types de données**

Vérifier que chaque champ contient le bon type de données :

| Type BSON | Description | Exemple |
|-----------|-------------|---------|
| `string` | Texte | `"Bonjour"` |
| `int` | Nombre entier | `42` |
| `double` | Nombre décimal | `3.14` |
| `bool` | Booléen | `true` ou `false` |
| `date` | Date | `ISODate("2025-01-01")` |
| `array` | Tableau | `[1, 2, 3]` |
| `object` | Objet imbriqué | `{rue: "...", ville: "..."}` |

### 2. **Champs obligatoires**

Spécifier quels champs **doivent** être présents dans chaque document :

```javascript
required: ["nom", "email", "dateInscription"]
```

### 3. **Valeurs autorisées**

Définir des contraintes sur les valeurs :

```javascript
// L'âge doit être entre 18 et 100
age: {
  bsonType: "int",
  minimum: 18,
  maximum: 100
}

// Le statut ne peut être que l'une de ces valeurs
statut: {
  enum: ["actif", "inactif", "en_attente"]
}
```

### 4. **Format des chaînes**

Vérifier le format des données textuelles avec des expressions régulières :

```javascript
// Email valide
email: {
  bsonType: "string",
  pattern: "^.+@.+\\..+$"
}

// Code postal français (5 chiffres)
codePostal: {
  bsonType: "string",
  pattern: "^[0-9]{5}$"
}
```

### 5. **Structure des documents imbriqués**

Valider les objets imbriqués :

```javascript
adresse: {
  bsonType: "object",
  required: ["rue", "ville", "codePostal"],
  properties: {
    rue: { bsonType: "string" },
    ville: { bsonType: "string" },
    codePostal: { bsonType: "string" }
  }
}
```

---

## ⚖️ Les deux approches de validation

### 1. **Validation stricte** (recommandée pour les nouvelles collections)

Tous les nouveaux documents ET les modifications doivent respecter les règles :

```javascript
validationLevel: "strict"  // Par défaut
```

**Quand l'utiliser ?**
- Nouvelles collections
- Lorsque vous voulez garantir la qualité des données
- Applications en développement actif

### 2. **Validation modérée** (utile pour les migrations)

Seuls les nouveaux documents sont validés, les documents existants peuvent rester "non conformes" :

```javascript
validationLevel: "moderate"
```

**Quand l'utiliser ?**
- Collections existantes avec données non conformes
- Migrations progressives
- Transition vers un schéma validé

---

## 🚦 Les deux actions possibles

Que faire quand un document ne respecte pas les règles ?

### 1. **Erreur** (recommandé)

MongoDB **refuse** l'insertion ou la modification :

```javascript
validationAction: "error"  // Par défaut
```

```javascript
// Tentative d'insertion invalide
db.utilisateurs.insertOne({
  nom: "Dupont"
  // email manquant -> ERREUR !
})

// Résultat : MongoServerError: Document failed validation
```

### 2. **Avertissement** (mode surveillance)

MongoDB **accepte** le document mais **enregistre** un avertissement dans les logs :

```javascript
validationAction: "warn"
```

**Quand l'utiliser ?**
- Phase de test de vos règles de validation
- Surveillance sans bloquer les opérations
- Analyse de l'impact avant activation stricte

---

## 🔍 Exemple complet débutant

Créons une collection `produits` avec validation simple :

```javascript
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prix", "stock"],
      properties: {
        nom: {
          bsonType: "string",
          description: "Le nom du produit est obligatoire"
        },
        prix: {
          bsonType: "double",
          minimum: 0,
          description: "Le prix doit être un nombre positif"
        },
        stock: {
          bsonType: "int",
          minimum: 0,
          description: "Le stock doit être un entier positif"
        },
        description: {
          bsonType: "string",
          description: "Description facultative du produit"
        }
      }
    }
  },
  validationLevel: "strict",
  validationAction: "error"
})
```

**Test avec des insertions** :

```javascript
// ✅ VALIDE - Tous les champs requis présents et corrects
db.produits.insertOne({
  nom: "Ordinateur portable",
  prix: 899.99,
  stock: 15,
  description: "Intel Core i7, 16Go RAM"
})

// ❌ INVALIDE - Prix négatif
db.produits.insertOne({
  nom: "Souris",
  prix: -10.00,
  stock: 50
})
// Erreur : Le prix doit être positif

// ❌ INVALIDE - Champ "nom" manquant
db.produits.insertOne({
  prix: 25.00,
  stock: 100
})
// Erreur : Le champ "nom" est requis

// ✅ VALIDE - Champ "description" facultatif peut être omis
db.produits.insertOne({
  nom: "Clavier",
  prix: 45.00,
  stock: 30
})
```

---

## 💡 Avantages de la validation de schéma

| Avantage | Description |
|----------|-------------|
| 🛡️ **Intégrité des données** | Empêche les erreurs de saisie et les incohérences |
| 📝 **Documentation** | Le schéma documente la structure attendue |
| 🐛 **Détection précoce** | Les erreurs sont détectées à l'insertion, pas à l'utilisation |
| 🤝 **Collaboration** | Tous les développeurs comprennent la structure |
| 🔄 **Migration facilitée** | Transition progressive vers un schéma structuré |
| ⚡ **Performance** | Données cohérentes = requêtes optimisées |

---

## ⚠️ Limitations et considérations

### Ce que la validation NE fait PAS

- ❌ **Ne remplace pas** la validation côté application
- ❌ **Ne vérifie pas** les relations entre collections
- ❌ **Ne garantit pas** la cohérence transactionnelle complexe
- ❌ **Ne protège pas** contre toutes les erreurs logiques

### Bonnes pratiques

- ✅ **Commencez simple** : Validez d'abord les champs essentiels
- ✅ **Testez en "warn"** : Avant d'activer en mode "error"
- ✅ **Documentez** : Expliquez pourquoi chaque règle existe
- ✅ **Validez aussi côté application** : Double protection
- ✅ **Évoluez progressivement** : Ajoutez des règles au fur et à mesure

---

## 🎓 Résumé

La validation de schéma dans MongoDB vous permet de :

1. **Définir des règles** sur la structure et le contenu de vos documents
2. **Garantir la qualité** des données stockées
3. **Prévenir les erreurs** dès l'insertion
4. **Documenter** la structure attendue de vos collections

C'est un outil puissant qui combine la **flexibilité de MongoDB** avec la **rigueur nécessaire** pour des applications de production.

---

## 📚 Dans les prochaines sections

Dans les sections suivantes, nous verrons :

- **7.2** : Comment utiliser JSON Schema dans MongoDB
- **7.3** : Règles de validation détaillées avec `$jsonSchema`
- **7.4** : Les différents niveaux de validation
- **7.5** : Les actions de validation (erreur vs avertissement)
- **7.6** : Comment modifier les règles de validation

---


⏭️ [JSON Schema dans MongoDB](/07-validation-des-schemas/02-json-schema-mongodb.md)
