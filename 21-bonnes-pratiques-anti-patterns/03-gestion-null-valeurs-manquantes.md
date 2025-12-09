🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.3 Gestion des null et Valeurs Manquantes

## Introduction

Dans MongoDB, la flexibilité du schéma permet aux documents d'une même collection d'avoir des structures différentes. Cette liberté soulève une question fondamentale : comment représenter l'absence d'information ? Contrairement aux bases de données relationnelles où `NULL` est la seule option, MongoDB offre plusieurs approches, chacune avec ses implications sur les requêtes, les performances, la validation et la sémantique métier.

Une mauvaise gestion des valeurs manquantes peut créer des bugs subtils, des incohérences de données, des problèmes de performance avec les index, et une complexité accrue dans le code applicatif. Cette section explore les meilleures pratiques pour gérer l'absence de données de manière cohérente, performante et maintenable.

---

## Comprendre les Options d'Absence

MongoDB offre trois façons principales de représenter l'absence d'une valeur :

### 1. Champ Absent (Recommandé par défaut)
```javascript
{
  _id: 1,
  name: "Alice",
  email: "alice@example.com"
  // middleName n'existe pas
}
```

### 2. Valeur null (Explicite)
```javascript
{
  _id: 1,
  name: "Alice",
  email: "alice@example.com",
  middleName: null  // Présence explicite de l'absence
}
```

### 3. Valeur undefined (À éviter)
```javascript
{
  _id: 1,
  name: "Alice",
  email: "alice@example.com",
  middleName: undefined  // Éviter : comportement incohérent
}
```

---

## ✅ DO : Privilégier les Champs Absents pour les Données Optionnelles

**Explication** : Ne stocker que les champs qui ont une valeur réelle réduit la taille des documents, simplifie la structure et est sémantiquement plus clair.

**Approche recommandée** :
```javascript
// ✅ Champs optionnels absents
const user = {
  _id: new ObjectId(),
  name: "Alice Smith",
  email: "alice@example.com",
  // middleName absent (Alice n'a pas de deuxième prénom)
  // phoneNumber absent (pas fourni)
  // company absent (pas employée)
};

await db.users.insertOne(user);
```

**Avantages mesurables** :

### 1. Réduction de la Taille
```javascript
// Avec null (tous les champs présents)
{
  name: "Alice",
  middleName: null,
  nickname: null,
  phoneNumber: null,
  company: null,
  department: null,
  bio: null
}
// Taille: ~120 bytes

// Sans les champs optionnels
{
  name: "Alice"
}
// Taille: ~40 bytes
= 66% de réduction
```

**Impact sur une collection** :
- 1 million d'utilisateurs
- 80 bytes économisés par document
- = 76 MB économisés en stockage
- = Moins de RAM nécessaire pour le cache
- = Moins de bande passante réseau

### 2. Clarté Sémantique
```javascript
// ✅ Clair : Alice n'a pas de numéro de téléphone
{ name: "Alice" }

// ⚠️ Ambigu : Alice a-t-elle un téléphone ou pas?
{ name: "Alice", phoneNumber: null }
```

### 3. Flexibilité du Schéma
- Nouveaux champs ajoutés sans migration
- Anciens documents restent valides
- Évolution naturelle du schéma

---

## ❌ DON'T : Remplir Tous les Champs Possibles avec null

**Explication** : Créer des documents avec tous les champs possibles définis à `null` reproduit le modèle relationnel et annule les avantages de MongoDB.

**Anti-pattern** :
```javascript
// ❌ "Schéma fixe" avec null partout
const user = {
  _id: new ObjectId(),
  firstName: "Alice",
  middleName: null,       // Optionnel mais forcé
  lastName: "Smith",
  nickname: null,         // Optionnel mais forcé
  email: "alice@example.com",
  phoneNumber: null,      // Optionnel mais forcé
  alternateEmail: null,   // Optionnel mais forcé
  company: null,          // Optionnel mais forcé
  department: null,       // Optionnel mais forcé
  manager: null,          // Optionnel mais forcé
  bio: null,              // Optionnel mais forcé
  website: null,          // Optionnel mais forcé
  twitter: null,          // Optionnel mais forcé
  linkedin: null,         // Optionnel mais forcé
  github: null            // Optionnel mais forcé
};
```

**Conséquences** :

### 1. Gaspillage de Ressources
- Documents 2-3x plus volumineux
- Utilisation mémoire accrue
- Bande passante réseau gaspillée
- Coût de stockage augmenté

### 2. Code Plus Complexe
```javascript
// ❌ Doit vérifier chaque champ
if (user.phoneNumber !== null) {
  sendSMS(user.phoneNumber);
}
if (user.company !== null) {
  // ...
}
if (user.twitter !== null) {
  // ...
}
```

### 3. Maintenance Difficile
- Ajout de nouveaux champs = migration de toute la collection
- Suppression de champs obsolètes = migration nécessaire
- Perte de la flexibilité MongoDB

### 4. Sémantique Confuse
```javascript
// Quelle est la différence entre :
{ phoneNumber: null }        // Pas de téléphone?
{ phoneNumber: undefined }   // Non renseigné?
{ /* phoneNumber absent */ } // Optionnel?
```

**Solution** :
```javascript
// ✅ Ne stocker que ce qui existe
const user = {
  _id: new ObjectId(),
  firstName: "Alice",
  lastName: "Smith",
  email: "alice@example.com"
  // Autres champs ajoutés seulement s'ils existent
};

if (phoneNumber) {
  user.phoneNumber = phoneNumber;
}
if (company) {
  user.company = company;
}
```

---

## ✅ DO : Utiliser null pour Distinguer "Inconnu" vs "Non Applicable"

**Explication** : Quand la sémantique métier nécessite de distinguer entre "donnée non fournie" et "donnée explicitement absente", `null` est approprié.

**Cas d'usage légitimes** :
```javascript
// ✅ null pour valeur explicitement "aucun"
{
  _id: 1,
  employeeId: "EMP001",
  name: "Bob Johnson",
  manager: null,  // Bob n'a PAS de manager (CEO)
  department: "Executive"
}
// vs champ absent qui signifierait "information manquante"

// ✅ null pour "réponse négative explicite"
{
  _id: 2,
  productId: "PROD123",
  name: "Basic Widget",
  warrantyYears: null,  // Explicitement AUCUNE garantie
  price: 9.99
}
// vs champ absent = information pas encore renseignée

// ✅ null pour optionnel dans un schéma validé
{
  _id: 3,
  orderId: "ORD456",
  customerId: ObjectId("..."),
  discountCode: null,   // Pas de code promo utilisé
  total: 150.00
}
```

**Différences sémantiques** :
```javascript
// Champ absent : "Je ne sais pas"
{ name: "Alice" }
// middleName n'est pas renseigné

// null : "Je sais qu'il n'y a rien"
{ name: "Alice", middleName: null }
// Alice n'a explicitement pas de deuxième prénom

// Valeur vide : "Il y a quelque chose mais c'est vide"
{ name: "Alice", middleName: "" }
// Alice a un deuxième prénom enregistré mais il est vide (inhabituel)
```

---

## ❌ DON'T : Mélanger les Conventions dans la Même Collection

**Explication** : L'incohérence dans la représentation des valeurs manquantes crée de la confusion et des bugs.

**Anti-pattern** :
```javascript
// ❌ Incohérence dans la même collection
// Document 1
{
  _id: 1,
  name: "Alice",
  phoneNumber: null
}

// Document 2
{
  _id: 2,
  name: "Bob"
  // phoneNumber absent
}

// Document 3
{
  _id: 3,
  name: "Charlie",
  phoneNumber: undefined
}

// Document 4
{
  _id: 4,
  name: "David",
  phoneNumber: ""  // String vide
}
```

**Conséquences** :

### 1. Requêtes Complexes et Fragiles
```javascript
// ❌ Requête qui doit gérer toutes les variations
db.users.find({
  $or: [
    { phoneNumber: null },
    { phoneNumber: { $exists: false } },
    { phoneNumber: undefined },
    { phoneNumber: "" }
  ]
});
```

### 2. Bugs Subtils
```javascript
// Code qui compte les utilisateurs sans téléphone
const withoutPhone = await db.users.countDocuments({
  phoneNumber: null
});
// Oublie les documents où le champ est absent!

// Correction nécessaire
const withoutPhone = await db.users.countDocuments({
  $or: [
    { phoneNumber: null },
    { phoneNumber: { $exists: false } }
  ]
});
```

### 3. Maintenance Cauchemardesque
- Code défensif partout
- Tests plus complexes
- Impossible de raisonner sur les données
- Onboarding difficile

**Solution - Convention Stricte** :
```javascript
// ✅ Convention documentée et appliquée
/**
 * CONVENTION : Valeurs Optionnelles
 *
 * - Champs optionnels : ABSENTS si pas de valeur
 * - Champs obligatoires : Toujours présents, null si "aucun" explicite
 * - Jamais undefined
 * - String vide uniquement si sémantiquement différent de null
 */

// Application de la convention
{
  _id: 1,
  name: "Alice",              // Obligatoire
  email: "alice@example.com", // Obligatoire
  manager: null,              // Obligatoire, null = "pas de manager"
  // phoneNumber absent        // Optionnel, non fourni
}
```

---

## ✅ DO : Utiliser $exists pour Requêter les Champs Absents

**Explication** : L'opérateur `$exists` est l'outil approprié pour vérifier la présence ou l'absence d'un champ.

**Requêtes correctes** :
```javascript
// ✅ Trouver les documents sans numéro de téléphone
db.users.find({
  phoneNumber: { $exists: false }
});

// ✅ Trouver les documents avec numéro de téléphone
db.users.find({
  phoneNumber: { $exists: true }
});

// ✅ Trouver les documents où le champ est absent OU null
db.users.find({
  $or: [
    { phoneNumber: { $exists: false } },
    { phoneNumber: null }
  ]
});

// ✅ Alternative plus concise (attention à la sémantique)
db.users.find({
  phoneNumber: { $in: [null] }
  // Matche null ET champs absents
});
```

**Avec index** :
```javascript
// ✅ Créer un index sparse pour les champs optionnels
db.users.createIndex(
  { phoneNumber: 1 },
  { sparse: true }  // N'indexe que les documents avec phoneNumber
);

// Requête utilisant l'index
db.users.find({
  phoneNumber: { $exists: true, $regex: /^\\+33/ }
});
```

---

## ❌ DON'T : Oublier les Champs Absents dans les Requêtes

**Explication** : Une requête qui vérifie `null` oublie souvent les documents où le champ est complètement absent.

**Bug courant** :
```javascript
// ❌ Incomplète : oublie les champs absents
const usersWithoutPhone = await db.users.find({
  phoneNumber: null
}).toArray();

// Résultat partiel :
// - Inclut : { name: "Alice", phoneNumber: null }
// - Oublie : { name: "Bob" } (phoneNumber absent)
```

**Impact** :
```javascript
// Base de données :
// 1000 users sans phoneNumber (champ absent)
// 500 users avec phoneNumber: null
// 1500 users avec phoneNumber: "..."

// Requête incorrecte
db.users.countDocuments({ phoneNumber: null })
// Résultat : 500 (au lieu de 1500!)

// Requête correcte
db.users.countDocuments({
  $or: [
    { phoneNumber: null },
    { phoneNumber: { $exists: false } }
  ]
})
// Résultat : 1500 ✓
```

**Solution** :
```javascript
// ✅ Helper function pour simplifier
function isNullOrMissing(field) {
  return {
    $or: [
      { [field]: null },
      { [field]: { $exists: false } }
    ]
  };
}

// Usage
const query = isNullOrMissing('phoneNumber');
const usersWithoutPhone = await db.users.find(query).toArray();
```

---

## ✅ DO : Définir des Valeurs par Défaut au Niveau Application

**Explication** : Les valeurs par défaut doivent être gérées dans le code applicatif, pas dans la base de données.

**Pattern recommandé** :
```javascript
// ✅ Valeurs par défaut dans le modèle
class User {
  constructor(data) {
    this._id = data._id || new ObjectId();
    this.name = data.name;
    this.email = data.email;
    this.role = data.role || 'user';        // Défaut: 'user'
    this.isActive = data.isActive ?? true;  // Défaut: true
    this.createdAt = data.createdAt || new Date();

    // Champs optionnels : seulement si fournis
    if (data.phoneNumber) {
      this.phoneNumber = data.phoneNumber;
    }
    if (data.company) {
      this.company = data.company;
    }
  }

  toJSON() {
    const obj = {
      _id: this._id,
      name: this.name,
      email: this.email,
      role: this.role,
      isActive: this.isActive,
      createdAt: this.createdAt
    };

    if (this.phoneNumber) obj.phoneNumber = this.phoneNumber;
    if (this.company) obj.company = this.company;

    return obj;
  }
}

// Usage
const user = new User({
  name: "Alice",
  email: "alice@example.com"
  // role et isActive utilisent les valeurs par défaut
});

await db.users.insertOne(user.toJSON());
```

**Avec Mongoose (ODM)** :
```javascript
// ✅ Schema avec valeurs par défaut
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  role: { type: String, default: 'user' },
  isActive: { type: Boolean, default: true },
  phoneNumber: { type: String },  // Optionnel, pas de défaut
  company: { type: String },      // Optionnel, pas de défaut
  createdAt: { type: Date, default: Date.now }
});
```

---

## ❌ DON'T : Stocker des Valeurs par Défaut Redondantes

**Explication** : Stocker explicitement des valeurs par défaut dans chaque document gaspille de l'espace et rend les changements de défaut impossibles.

**Anti-pattern** :
```javascript
// ❌ Valeur par défaut stockée dans chaque document
await db.users.insertMany([
  { name: "Alice", role: "user", status: "active" },
  { name: "Bob", role: "user", status: "active" },
  { name: "Charlie", role: "user", status: "active" },
  // "user" et "active" répétés 1 million de fois!
]);
```

**Problèmes** :

### 1. Gaspillage d'Espace
```javascript
// 1 million d'utilisateurs
// Champ "role": "user" (4 bytes) répété
= 4 MB de données redondantes

// Si omis avec valeur par défaut applicative
= 0 bytes stockés
```

### 2. Impossible de Changer la Valeur par Défaut
```javascript
// Si on veut changer le rôle par défaut de "user" à "member"
// Tous les documents existants ont "user" stocké
// Impossible de distinguer :
// - Ceux qui ont explicitement choisi "user"
// - Ceux qui ont utilisé la valeur par défaut
```

### 3. Migrations Nécessaires
```javascript
// ❌ Migration complexe pour changer la valeur par défaut
await db.users.updateMany(
  { role: "user" },  // Mais certains ont peut-être choisi "user"!
  { $unset: { role: "" } }
);
```

**Solution** :
```javascript
// ✅ Omit default values, handle in application
await db.users.insertOne({
  name: "Alice"
  // role omis, sera "user" au niveau application
  // status omis, sera "active" au niveau application
});

// Dans le code
function getUserWithDefaults(user) {
  return {
    ...user,
    role: user.role || 'user',
    status: user.status || 'active',
    isVerified: user.isVerified ?? false
  };
}

// Changer la valeur par défaut = changer une ligne de code
// Pas de migration nécessaire!
```

---

## ✅ DO : Utiliser des Index Sparse pour les Champs Optionnels

**Explication** : Les index sparse n'incluent que les documents où le champ indexé existe, économisant de l'espace et améliorant les performances.

**Usage approprié** :
```javascript
// ✅ Index sparse pour champ optionnel
db.users.createIndex(
  { phoneNumber: 1 },
  {
    sparse: true,
    unique: true  // Unicité uniquement pour les valeurs présentes
  }
);

// Bénéfices :
// - Index plus petit (seulement les docs avec phoneNumber)
// - Requêtes plus rapides sur phoneNumber
// - Unicité sans contraindre les champs absents
```

**Comparaison** :
```javascript
// Collection : 1 million d'utilisateurs
// 200,000 ont un phoneNumber
// 800,000 n'ont pas de phoneNumber

// Index normal (non-sparse)
db.users.createIndex({ phoneNumber: 1 });
// Taille : ~1,000,000 entrées (incluant null/absent)
// Espace : ~40 MB

// Index sparse
db.users.createIndex({ phoneNumber: 1 }, { sparse: true });
// Taille : ~200,000 entrées (seulement les valeurs présentes)
// Espace : ~8 MB
// = 80% de réduction
```

**Attention aux requêtes** :
```javascript
// ⚠️ Index sparse n'est PAS utilisé pour cette requête
db.users.find({ phoneNumber: { $exists: false } });
// Car l'index ne contient pas les documents sans phoneNumber

// ✅ Index sparse est utilisé
db.users.find({ phoneNumber: { $regex: /^\\+33/ } });
```

---

## ❌ DON'T : Utiliser undefined dans MongoDB

**Explication** : `undefined` en JavaScript n'est pas un type BSON valide et crée un comportement incohérent.

**Problème** :
```javascript
// ❌ undefined n'est pas supporté par BSON
const user = {
  name: "Alice",
  phoneNumber: undefined  // Sera converti en null ou omis selon le driver
};

await db.users.insertOne(user);

// Résultat variable selon le driver et la version :
// - Certains drivers convertissent undefined → null
// - D'autres omettent le champ complètement
// - D'autres lancent une erreur

// = Comportement imprévisible
```

**Comparaison** :
```javascript
// JavaScript
const obj = { name: "Alice", phone: undefined };
console.log(obj);
// { name: "Alice", phone: undefined }

// JSON.stringify (utilisé par la plupart des drivers)
JSON.stringify({ name: "Alice", phone: undefined });
// {"name":"Alice"}
// undefined est omis!

// BSON n'a pas de type "undefined"
// Le comportement est non défini (jeu de mot intentionnel)
```

**Solution** :
```javascript
// ✅ Éviter undefined explicitement
const user = {
  name: "Alice",
  email: "alice@example.com"
};

// Si conditionnel
if (phoneNumber !== undefined) {
  user.phoneNumber = phoneNumber;
}

// Ou helper
function omitUndefined(obj) {
  return Object.fromEntries(
    Object.entries(obj).filter(([_, v]) => v !== undefined)
  );
}

const userData = omitUndefined({
  name: "Alice",
  email: "alice@example.com",
  phoneNumber: undefined,  // Sera omis
  company: undefined       // Sera omis
});
```

---

## ✅ DO : Documenter la Sémantique des Valeurs Manquantes

**Explication** : Une documentation claire de la convention choisie évite les erreurs et facilite la maintenance.

**Documentation recommandée** :
```javascript
/**
 * CONVENTION: Valeurs Manquantes
 *
 * === CHAMPS OBLIGATOIRES ===
 * Toujours présents dans tous les documents.
 * Peuvent être null si sémantiquement approprié.
 *
 * Exemples:
 * - name: string (jamais null)
 * - email: string (jamais null)
 * - manager: ObjectId | null ("null" = pas de manager, comme le CEO)
 *
 * === CHAMPS OPTIONNELS ===
 * Absents si pas de valeur.
 * Jamais null (utiliser absence).
 *
 * Exemples:
 * - phoneNumber?: string (absent si non fourni)
 * - company?: string (absent si non employé)
 * - bio?: string (absent si non renseigné)
 *
 * === INTERDICTIONS ===
 * - Jamais utiliser "undefined"
 * - Jamais mélanger null et absence pour le même champ
 * - Jamais utiliser "" (string vide) comme substitut à null/absence
 *
 * === REQUÊTES ===
 * Pour trouver les valeurs manquantes:
 * - Champs optionnels: { field: { $exists: false } }
 * - Champs obligatoires null: { field: null }
 * - Les deux: { $or: [{ field: null }, { field: { $exists: false } }] }
 */

// Exemple de schéma commenté
const userSchema = {
  // Obligatoires (toujours présents)
  _id: ObjectId,
  name: String,           // Jamais null
  email: String,          // Jamais null
  role: String,           // Default: "user" (application level)
  isActive: Boolean,      // Default: true (application level)
  createdAt: Date,        // Default: Date.now() (application level)

  // Obligatoires mais peuvent être null (sémantique spécifique)
  manager: ObjectId | null,      // null = pas de manager
  lastLoginAt: Date | null,      // null = jamais connecté

  // Optionnels (absents si pas de valeur)
  middleName: String,            // Présent uniquement si l'utilisateur a un deuxième prénom
  phoneNumber: String,           // Présent uniquement si fourni
  company: String,               // Présent uniquement si employé
  department: String,            // Présent uniquement si employé
  bio: String                    // Présent uniquement si renseigné
};
```

---

## ✅ DO : Utiliser la Validation de Schéma pour Enforcer la Convention

**Explication** : JSON Schema validation dans MongoDB peut enforcer vos conventions sur les valeurs manquantes.

**Schema de validation** :
```javascript
// ✅ Validation qui enforce la convention
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["name", "email", "role", "isActive", "createdAt"],
      properties: {
        _id: { bsonType: "objectId" },

        // Obligatoires (pas de null autorisé)
        name: {
          bsonType: "string",
          minLength: 1
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        },
        role: {
          enum: ["user", "admin", "moderator"]
        },
        isActive: {
          bsonType: "bool"
        },
        createdAt: {
          bsonType: "date"
        },

        // Obligatoires mais null accepté
        manager: {
          bsonType: ["objectId", "null"]
        },
        lastLoginAt: {
          bsonType: ["date", "null"]
        },

        // Optionnels (peuvent être absents)
        phoneNumber: {
          bsonType: "string",
          pattern: "^\\+?[1-9]\\d{1,14}$"
        },
        company: {
          bsonType: "string"
        },
        bio: {
          bsonType: "string",
          maxLength: 500
        }
      },
      additionalProperties: false
    }
  }
});
```

**Validation qui rejette undefined** :
```javascript
// ✅ Empêcher undefined au niveau du code
function validateUserData(data) {
  // Vérifier qu'aucun champ n'est undefined
  for (const [key, value] of Object.entries(data)) {
    if (value === undefined) {
      throw new Error(`Field ${key} cannot be undefined. Use null or omit the field.`);
    }
  }

  // Vérifier les champs requis
  const required = ['name', 'email', 'role', 'isActive'];
  for (const field of required) {
    if (!(field in data) || data[field] === null) {
      throw new Error(`Field ${field} is required and cannot be null`);
    }
  }

  return data;
}

// Usage
try {
  const userData = validateUserData({
    name: "Alice",
    email: "alice@example.com",
    phoneNumber: undefined  // Erreur lancée
  });
} catch (error) {
  console.error(error.message);
  // "Field phoneNumber cannot be undefined. Use null or omit the field."
}
```

---

## ❌ DON'T : Ignorer les Valeurs null dans les Agrégations

**Explication** : Les opérateurs d'agrégation traitent `null` et les champs absents différemment, ce qui peut causer des résultats inattendus.

**Problème** :
```javascript
// Collection
db.orders.insertMany([
  { customerId: 1, total: 100, discount: 10 },
  { customerId: 2, total: 200, discount: null },
  { customerId: 3, total: 150 }  // discount absent
]);

// ❌ Agrégation naïve
db.orders.aggregate([
  {
    $group: {
      _id: null,
      avgDiscount: { $avg: "$discount" }
    }
  }
]);
// Résultat : { avgDiscount: 10 }
// N'inclut que le premier document!
// Les null et absents sont ignorés par $avg
```

**Résultats variables par opérateur** :
```javascript
// $sum ignore null et absent (traite comme 0)
{ $sum: "$discount" }  // = 10

// $avg ignore null et absent
{ $avg: "$discount" }  // = 10 (pas 10/3 = 3.33)

// $min et $max ignorent null et absent
{ $min: "$discount" }  // = 10
{ $max: "$discount" }  // = 10
```

**Solution explicite** :
```javascript
// ✅ Gérer explicitement les null/absents
db.orders.aggregate([
  {
    $group: {
      _id: null,
      avgDiscount: {
        $avg: {
          $ifNull: ["$discount", 0]  // Traiter null/absent comme 0
        }
      },
      totalWithDiscount: {
        $sum: {
          $cond: [
            { $gt: ["$discount", 0] },
            1,
            0
          ]
        }
      }
    }
  }
]);
```

---

## ✅ DO : Utiliser $ifNull pour Gérer les Valeurs Manquantes

**Explication** : L'opérateur `$ifNull` permet de fournir une valeur par défaut lors du traitement de champs potentiellement absents.

**Usage dans les agrégations** :
```javascript
// ✅ Valeurs par défaut avec $ifNull
db.products.aggregate([
  {
    $project: {
      name: 1,
      price: 1,
      // Si discount absent ou null, utiliser 0
      finalPrice: {
        $subtract: [
          "$price",
          { $ifNull: ["$discount", 0] }
        ]
      },
      // Si rating absent, afficher "Not rated"
      displayRating: {
        $ifNull: ["$rating", "Not rated"]
      }
    }
  }
]);
```

**Chaînage de valeurs de repli** :
```javascript
// ✅ Cascade de valeurs par défaut
db.users.aggregate([
  {
    $project: {
      // Utiliser phoneNumber, sinon alternatePhone, sinon "No phone"
      contactPhone: {
        $ifNull: [
          "$phoneNumber",
          { $ifNull: ["$alternatePhone", "No phone"] }
        ]
      },
      // Utiliser displayName, sinon username, sinon email
      displayedName: {
        $ifNull: [
          "$displayName",
          { $ifNull: ["$username", "$email"] }
        ]
      }
    }
  }
]);
```

---

## ✅ DO : Gérer les Valeurs Manquantes dans les Updates

**Explication** : Les opérations d'update doivent être conscientes de la différence entre null et champs absents.

**Updates corrects** :
```javascript
// ✅ Ajouter un champ optionnel seulement s'il a une valeur
const updates = {
  name: "Alice Smith"
};

if (phoneNumber) {
  updates.phoneNumber = phoneNumber;
}
if (company) {
  updates.company = company;
}

db.users.updateOne(
  { _id: userId },
  { $set: updates }
);

// ✅ Supprimer un champ (le rendre absent)
db.users.updateOne(
  { _id: userId },
  { $unset: { phoneNumber: "" } }
);

// ✅ Définir explicitement à null (si sémantique appropriée)
db.users.updateOne(
  { _id: userId },
  { $set: { manager: null } }  // Plus de manager
);
```

**Anti-pattern** :
```javascript
// ❌ Update qui met null pour "supprimer"
db.users.updateOne(
  { _id: userId },
  { $set: { phoneNumber: null } }
  // phoneNumber existe maintenant avec valeur null
  // au lieu d'être absent
);

// ❌ Update qui crée des champs vides
db.users.updateOne(
  { _id: userId },
  { $set: {
    phoneNumber: phoneNumber || null,
    company: company || null
  }}
  // Crée des champs null même si pas nécessaire
);
```

---

## Patterns Avancés

### ✅ DO : Utiliser des Types Union pour la Clarté TypeScript

**Explication** : Avec TypeScript, définir explicitement les types union aide à gérer les valeurs manquantes.

```typescript
// ✅ Types explicites
interface User {
  _id: ObjectId;
  name: string;
  email: string;

  // Obligatoires mais peuvent être null (sémantique)
  manager: ObjectId | null;
  lastLoginAt: Date | null;

  // Optionnels (peuvent être absents)
  phoneNumber?: string;
  company?: string;
  middleName?: string;
}

// Usage
function createUser(data: Partial<User>): User {
  return {
    _id: new ObjectId(),
    name: data.name!,
    email: data.email!,
    manager: data.manager ?? null,
    lastLoginAt: data.lastLoginAt ?? null,
    // phoneNumber omis si undefined
    ...(data.phoneNumber && { phoneNumber: data.phoneNumber }),
    ...(data.company && { company: data.company })
  };
}
```

---

### ✅ DO : Normaliser les Données en Entrée

**Explication** : Convertir les entrées utilisateur en format cohérent avant stockage.

```javascript
// ✅ Normalisation des entrées
function normalizeUserInput(input) {
  const normalized = {
    name: input.name?.trim(),
    email: input.email?.toLowerCase().trim()
  };

  // Convertir string vides en absence
  if (input.phoneNumber && input.phoneNumber.trim()) {
    normalized.phoneNumber = input.phoneNumber.trim();
  }
  // Si phoneNumber est "", "", "   " → omis

  // Convertir null/undefined/"" en absence pour optionnels
  if (input.company && input.company.trim() !== '') {
    normalized.company = input.company.trim();
  }

  return normalized;
}

// Usage
const userData = normalizeUserInput({
  name: "  Alice  ",
  email: "ALICE@EXAMPLE.COM  ",
  phoneNumber: "",        // Sera omis
  company: "   "          // Sera omis
});

// Résultat :
// {
//   name: "Alice",
//   email: "alice@example.com"
//   // phoneNumber et company absents
// }
```

---

## Checklist Gestion des Valeurs Manquantes

### Convention
- [ ] Convention documentée (champs absents vs null)
- [ ] Équipe alignée sur la convention
- [ ] undefined jamais utilisé
- [ ] Cohérence dans toute la collection

### Champs Optionnels
- [ ] Absents par défaut (pas de null)
- [ ] Ajoutés seulement si valeur présente
- [ ] Pas de valeurs par défaut redondantes stockées
- [ ] Index sparse utilisés si approprié

### Champs Obligatoires
- [ ] Toujours présents dans tous les documents
- [ ] null utilisé seulement si sémantique spécifique
- [ ] Validation schéma pour enforcer
- [ ] Valeurs par défaut au niveau application

### Requêtes
- [ ] $exists utilisé pour champs absents
- [ ] null vs absent considérés dans les requêtes
- [ ] Agrégations avec $ifNull si nécessaire
- [ ] Index appropriés (sparse si optionnel)

### Code
- [ ] Validation des entrées
- [ ] Normalisation des données
- [ ] Gestion cohérente dans updates
- [ ] Tests couvrant null et absent

---

## Tableau Comparatif

| Représentation | Cas d'Usage | Sémantique | Requête | Taille |
|----------------|-------------|------------|---------|--------|
| **Champ absent** | ✅ Optionnel standard | "Non applicable" | `$exists: false` | 0 bytes |
| **null** | ✅ "Explicitement aucun" | "Vide connu" | `field: null` | ~1 byte |
| **"" (string vide)** | ⚠️ Rare | "Texte vide" | `field: ""` | ~1 byte |
| **undefined** | ❌ Jamais | Comportement inconsistant | Imprévisible | Variable |
| **0 (nombre)** | ✅ Valeur légitime | Zéro | `field: 0` | ~1-8 bytes |
| **false (bool)** | ✅ Valeur légitime | Faux | `field: false` | ~1 byte |
| **[] (array vide)** | ⚠️ Contextuel | "Liste vide" | `field: []` | ~5 bytes |
| **{} (objet vide)** | ❌ Éviter | Ambigu | `field: {}` | ~5 bytes |

---

## Conclusion

La gestion cohérente des valeurs manquantes est essentielle pour :

- **Performance** : Documents plus compacts, index plus efficaces
- **Clarté** : Sémantique claire du domaine métier
- **Maintenabilité** : Code prévisible et requêtes simples
- **Évolutivité** : Schéma flexible et évolutif

**Règles d'or** :
1. **Champs optionnels** : Absents par défaut
2. **null** : Uniquement pour "explicitement aucun"
3. **undefined** : Jamais dans MongoDB
4. **Convention** : Documentée et appliquée
5. **Validation** : Enforcer au niveau schéma et application

Une gestion rigoureuse des valeurs manquantes dès le début évite des migrations coûteuses et des bugs subtils en production.

---


⏭️ [Éviter les documents trop volumineux](/21-bonnes-pratiques-anti-patterns/04-eviter-documents-volumineux.md)
