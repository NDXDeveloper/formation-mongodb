🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.4 Opérateurs d'Éléments

## Introduction

Dans les chapitres précédents, nous avons appris à rechercher des documents en fonction des **valeurs** de leurs champs. Mais parfois, vous devez rechercher des documents en fonction de **l'existence même** d'un champ ou de son **type de données**.

Par exemple :
- Trouver tous les utilisateurs qui ont renseigné leur numéro de téléphone
- Identifier les documents où un champ est manquant
- Vérifier que certains champs sont du bon type (nombre, chaîne, date, etc.)
- Détecter les incohérences de données après une migration

MongoDB fournit deux **opérateurs d'éléments** pour ces besoins : `$exists` et `$type`.

Dans ce chapitre, nous allons explorer ces opérateurs en détail et apprendre comment les utiliser efficacement.

---

## Vue d'Ensemble des Opérateurs d'Éléments

MongoDB propose deux opérateurs d'éléments principaux :

| Opérateur | Description | Usage Principal |
|-----------|-------------|-----------------|
| `$exists` | Vérifie si un champ existe ou non dans un document | Identifier les documents avec ou sans un champ spécifique |
| `$type` | Vérifie le type BSON d'un champ | Valider le type de données d'un champ |

---

## L'Opérateur `$exists`

L'opérateur `$exists` permet de vérifier si un champ **existe** ou **n'existe pas** dans un document, indépendamment de sa valeur.

### Syntaxe

```javascript
{ champ: { $exists: <boolean> } }
```

- `$exists: true` : retourne les documents où le champ existe
- `$exists: false` : retourne les documents où le champ n'existe pas

### Concept Important

MongoDB est une base de données **sans schéma strict** (schemaless). Cela signifie que tous les documents d'une même collection n'ont pas nécessairement les mêmes champs. `$exists` vous permet de gérer cette flexibilité.

### Exemples de Base

#### Trouver les Documents avec un Champ Existant

```javascript
// Trouver les utilisateurs qui ont renseigné leur numéro de téléphone
db.users.find({ phone: { $exists: true } })

// Trouver les produits qui ont une description
db.products.find({ description: { $exists: true } })

// Trouver les commandes qui ont une date de livraison
db.orders.find({ deliveryDate: { $exists: true } })
```

#### Trouver les Documents Sans un Champ Spécifique

```javascript
// Trouver les utilisateurs qui n'ont PAS renseigné leur numéro de téléphone
db.users.find({ phone: { $exists: false } })

// Trouver les produits sans description
db.products.find({ description: { $exists: false } })

// Trouver les utilisateurs sans photo de profil
db.users.find({ profilePicture: { $exists: false } })
```

### Comportement avec `null`

**Important** : Un champ qui existe avec la valeur `null` est considéré comme **existant** par `$exists`.

```javascript
// Documents de la collection users :
{ name: "Alice", phone: "123456789" }        // phone existe
{ name: "Bob", phone: null }                  // phone existe (même si null)
{ name: "Charlie" }                           // phone n'existe pas

// Cette requête retourne Alice ET Bob (pas Charlie)
db.users.find({ phone: { $exists: true } })

// Cette requête retourne seulement Charlie
db.users.find({ phone: { $exists: false } })
```

### Différence entre `$exists` et Vérification de `null`

```javascript
// Trouver les documents où le champ existe (peut être null)
db.users.find({ phone: { $exists: true } })

// Trouver les documents où le champ est null (le champ doit exister)
db.users.find({ phone: null })
// Ceci retourne les documents où phone existe ET est null
// OU les documents où phone n'existe pas

// Pour être précis : trouver où phone existe ET n'est pas null
db.users.find({
    phone: { $exists: true, $ne: null }
})

// Pour trouver où phone n'existe pas OU est null
db.users.find({
    $or: [
        { phone: { $exists: false } },
        { phone: null }
    ]
})
```

### Exemples Pratiques

#### Validation de Données

```javascript
// Identifier les utilisateurs sans email (problème de données)
db.users.find({ email: { $exists: false } })

// Trouver les produits sans prix défini
db.products.find({ price: { $exists: false } })

// Identifier les commandes sans adresse de livraison
db.orders.find({ shippingAddress: { $exists: false } })
```

#### Champs Optionnels

```javascript
// Trouver les utilisateurs qui ont renseigné une biographie
db.users.find({ bio: { $exists: true } })

// Trouver les produits avec des images supplémentaires
db.products.find({ additionalImages: { $exists: true } })

// Trouver les articles avec des tags
db.articles.find({ tags: { $exists: true } })
```

#### Migration de Données

```javascript
// Après une migration, trouver les anciens documents sans nouveau champ
db.users.find({ newField: { $exists: false } })

// Compter combien de documents manquent un champ
db.products.countDocuments({ newPriceField: { $exists: false } })
```

---

## L'Opérateur `$type`

L'opérateur `$type` permet de vérifier le **type BSON** d'un champ. C'est utile pour valider les types de données, identifier les incohérences ou filtrer par type.

### Syntaxe

```javascript
{ champ: { $type: <type> } }
```

Le type peut être spécifié par :
- **Nom du type** (chaîne) : `"string"`, `"int"`, `"double"`, etc.
- **Numéro du type** (BSON type number) : `2` pour string, `16` pour int, etc.
- **Tableau de types** : pour rechercher plusieurs types possibles

### Types BSON Principaux

Voici les types BSON les plus couramment utilisés :

| Type | Alias | Numéro BSON | Description |
|------|-------|-------------|-------------|
| Double | `"double"` | 1 | Nombre à virgule flottante 64-bit |
| String | `"string"` | 2 | Chaîne de caractères UTF-8 |
| Object | `"object"` | 3 | Document imbriqué |
| Array | `"array"` | 4 | Tableau |
| Binary Data | `"binData"` | 5 | Données binaires |
| ObjectId | `"objectId"` | 7 | Identifiant ObjectId |
| Boolean | `"bool"` | 8 | Booléen (true/false) |
| Date | `"date"` | 9 | Date et heure |
| Null | `"null"` | 10 | Valeur null |
| Regular Expression | `"regex"` | 11 | Expression régulière |
| 32-bit Integer | `"int"` | 16 | Entier 32-bit |
| Timestamp | `"timestamp"` | 17 | Timestamp MongoDB |
| 64-bit Integer | `"long"` | 18 | Entier 64-bit |
| Decimal128 | `"decimal"` | 19 | Décimal haute précision |

### Exemples de Base

#### Vérifier le Type d'un Champ

```javascript
// Trouver les documents où age est un nombre (int ou long)
db.users.find({ age: { $type: "int" } })

// Trouver les documents où price est un double
db.products.find({ price: { $type: "double" } })

// Trouver les documents où createdAt est une date
db.orders.find({ createdAt: { $type: "date" } })

// Trouver les documents où name est une chaîne
db.users.find({ name: { $type: "string" } })

// Trouver les documents où tags est un tableau
db.articles.find({ tags: { $type: "array" } })
```

#### Utiliser les Numéros de Type

```javascript
// Équivalent avec numéros BSON
db.users.find({ age: { $type: 16 } })        // int
db.products.find({ price: { $type: 1 } })     // double
db.orders.find({ createdAt: { $type: 9 } })   // date
db.users.find({ name: { $type: 2 } })         // string
```

**Note** : L'utilisation des noms de types (chaînes) est généralement préférée pour la lisibilité.

### Rechercher Plusieurs Types

Vous pouvez rechercher des champs correspondant à **plusieurs types** en utilisant un tableau :

```javascript
// Trouver les documents où price est soit un int, soit un double
db.products.find({
    price: { $type: ["int", "double"] }
})

// Trouver les documents où quantity est un nombre (toutes variantes)
db.inventory.find({
    quantity: { $type: ["int", "long", "double", "decimal"] }
})

// Trouver les documents où value est soit null soit manquant
db.data.find({
    value: { $type: ["null"] }
})
```

### Type Spécial : `"number"`

MongoDB fournit un alias spécial `"number"` qui correspond à tous les types numériques :

```javascript
// Trouver tous les documents où price est un nombre (quelque soit le type numérique)
db.products.find({ price: { $type: "number" } })

// Équivalent à :
db.products.find({
    price: { $type: ["double", "int", "long", "decimal"] }
})
```

### Exemples Pratiques

#### Validation de Type de Données

```javascript
// Vérifier que tous les âges sont des entiers
db.users.find({ age: { $type: "int" } })

// Identifier les prix qui ne sont pas des nombres
db.products.find({
    price: { $not: { $type: "number" } }
})

// Trouver les documents où email n'est pas une chaîne
db.users.find({
    email: { $exists: true, $not: { $type: "string" } }
})
```

#### Détection d'Incohérences

```javascript
// Trouver les documents où un champ numérique est stocké comme chaîne
db.products.find({ price: { $type: "string" } })

// Identifier les dates stockées comme chaînes
db.orders.find({ orderDate: { $type: "string" } })

// Trouver les tableaux qui devraient être des objets
db.users.find({ address: { $type: "array" } })
```

#### Migration de Données

```javascript
// Après conversion, vérifier que tous les prix sont en decimal
db.products.find({ price: { $type: "decimal" } })

// Identifier les documents qui n'ont pas encore été migrés
db.users.find({
    birthDate: { $type: "string" }  // Ancien format
})

// Documents déjà migrés
db.users.find({
    birthDate: { $type: "date" }    // Nouveau format
})
```

---

## Combiner `$exists` et `$type`

La combinaison de `$exists` et `$type` est très puissante pour des validations précises.

### Syntaxe Combinée

```javascript
{
    champ: {
        $exists: true,
        $type: "type"
    }
}
```

### Exemples de Combinaison

```javascript
// Trouver les utilisateurs qui ont un email ET que c'est une chaîne
db.users.find({
    email: {
        $exists: true,
        $type: "string"
    }
})

// Trouver les produits avec un prix existant ET numérique
db.products.find({
    price: {
        $exists: true,
        $type: "number"
    }
})

// Trouver les commandes avec une date de livraison ET que c'est une date
db.orders.find({
    deliveryDate: {
        $exists: true,
        $type: "date"
    }
})
```

### Validation Stricte

```javascript
// Validation complète : le champ doit exister, être une chaîne ET non vide
db.users.find({
    email: {
        $exists: true,
        $type: "string",
        $ne: ""
    }
})

// Prix doit exister, être un nombre ET être positif
db.products.find({
    price: {
        $exists: true,
        $type: "number",
        $gt: 0
    }
})
```

---

## Cas d'Usage Pratiques

### Cas 1 : Audit de Qualité des Données

```javascript
// Collection: users
{
    _id: ObjectId("..."),
    name: "Alice",
    email: "alice@example.com",
    age: 30,
    phone: "123456789"
}
```

**Requêtes d'Audit** :

```javascript
// Trouver les utilisateurs sans email (données incomplètes)
db.users.find({ email: { $exists: false } })

// Trouver les utilisateurs avec un âge invalide (chaîne au lieu de nombre)
db.users.find({
    age: { $exists: true, $not: { $type: "number" } }
})

// Trouver les utilisateurs avec email mais pas du bon type
db.users.find({
    email: { $exists: true, $not: { $type: "string" } }
})

// Compter les utilisateurs avec données complètes
db.users.countDocuments({
    email: { $exists: true, $type: "string" },
    phone: { $exists: true, $type: "string" },
    age: { $exists: true, $type: "number" }
})

// Identifier les incohérences de type pour l'âge
db.users.find({
    age: { $type: "string" }  // Devrait être un nombre
})
```

### Cas 2 : Migration de Schéma

```javascript
// Collection: products (avant migration)
{
    _id: ObjectId("..."),
    name: "Laptop",
    price: "899.99",           // Ancien format : string
    stock: "15",               // Ancien format : string
    category: "Electronics"
}

// Collection: products (après migration)
{
    _id: ObjectId("..."),
    name: "Mouse",
    price: 29.99,              // Nouveau format : number
    stock: 50,                 // Nouveau format : number
    category: "Electronics",
    tags: ["wireless", "usb"]  // Nouveau champ
}
```

**Requêtes de Migration** :

```javascript
// Identifier les produits non encore migrés (prix en string)
db.products.find({ price: { $type: "string" } })

// Compter les produits déjà migrés
db.products.countDocuments({
    price: { $type: "number" },
    stock: { $type: "number" }
})

// Trouver les produits sans le nouveau champ tags
db.products.find({ tags: { $exists: false } })

// Vérifier que tous les produits ont le bon format
db.products.find({
    price: { $type: "number" },
    stock: { $type: "number" },
    tags: { $exists: true, $type: "array" }
})

// Produits partiellement migrés (besoin d'intervention)
db.products.find({
    $or: [
        { price: { $type: "string" } },
        { stock: { $type: "string" } }
    ]
})
```

### Cas 3 : Gestion de Champs Optionnels

```javascript
// Collection: profiles
{
    _id: ObjectId("..."),
    username: "johndoe",
    bio: "Developer and photographer",  // Optionnel
    website: "https://johndoe.com",     // Optionnel
    socialLinks: {                      // Optionnel
        twitter: "@johndoe",
        github: "johndoe"
    }
}
```

**Requêtes** :

```javascript
// Profils complets (tous les champs optionnels remplis)
db.profiles.find({
    bio: { $exists: true },
    website: { $exists: true },
    socialLinks: { $exists: true }
})

// Profils incomplets (au moins un champ optionnel manquant)
db.profiles.find({
    $or: [
        { bio: { $exists: false } },
        { website: { $exists: false } },
        { socialLinks: { $exists: false } }
    ]
})

// Profils avec bio renseignée
db.profiles.find({
    bio: { $exists: true, $type: "string", $ne: "" }
})

// Profils avec liens sociaux (objet valide)
db.profiles.find({
    socialLinks: { $exists: true, $type: "object" }
})
```

### Cas 4 : Validation de Formulaires

```javascript
// Collection: submissions
{
    _id: ObjectId("..."),
    name: "Alice",
    email: "alice@example.com",
    age: 25,
    newsletter: true,
    comments: "Great service!"
}
```

**Requêtes de Validation** :

```javascript
// Soumissions valides (tous les champs obligatoires présents et du bon type)
db.submissions.find({
    name: { $exists: true, $type: "string", $ne: "" },
    email: { $exists: true, $type: "string" },
    age: { $exists: true, $type: "number", $gte: 0 }
})

// Soumissions invalides (champs manquants ou mauvais types)
db.submissions.find({
    $or: [
        { name: { $exists: false } },
        { name: { $not: { $type: "string" } } },
        { email: { $exists: false } },
        { email: { $not: { $type: "string" } } },
        { age: { $exists: false } },
        { age: { $not: { $type: "number" } } }
    ]
})

// Soumissions avec champs optionnels remplis
db.submissions.find({
    newsletter: { $exists: true, $type: "bool" },
    comments: { $exists: true, $type: "string" }
})
```

---

## Opérations sur Documents Imbriqués

Les opérateurs `$exists` et `$type` fonctionnent également sur des champs imbriqués.

### Avec `$exists` sur Champs Imbriqués

```javascript
// Structure de document
{
    name: "Alice",
    address: {
        street: "123 Main St",
        city: "Paris",
        zipCode: "75001"
    }
}

// Trouver les utilisateurs avec une adresse complète
db.users.find({
    "address.street": { $exists: true },
    "address.city": { $exists: true },
    "address.zipCode": { $exists: true }
})

// Trouver les utilisateurs sans code postal
db.users.find({ "address.zipCode": { $exists: false } })

// Vérifier qu'un sous-document existe
db.users.find({ address: { $exists: true, $type: "object" } })
```

### Avec `$type` sur Champs Imbriqués

```javascript
// Vérifier le type d'un champ imbriqué
db.users.find({
    "address.zipCode": { $type: "string" }
})

// Vérifier plusieurs types dans un sous-document
db.users.find({
    "address.street": { $type: "string" },
    "address.city": { $type: "string" },
    "address.zipCode": { $type: "string" }
})
```

---

## Opérations sur Tableaux

### `$exists` avec des Tableaux

```javascript
// Documents
{ name: "Alice", hobbies: ["reading", "swimming"] }
{ name: "Bob", hobbies: [] }
{ name: "Charlie" }

// Trouver les documents avec un tableau hobbies (même vide)
db.users.find({ hobbies: { $exists: true } })
// Retourne Alice ET Bob

// Trouver les documents sans tableau hobbies
db.users.find({ hobbies: { $exists: false } })
// Retourne Charlie
```

### `$type` avec des Tableaux

```javascript
// Vérifier qu'un champ est un tableau
db.users.find({ hobbies: { $type: "array" } })

// Vérifier qu'un champ n'est PAS un tableau
db.users.find({
    hobbies: { $exists: true, $not: { $type: "array" } }
})

// Identifier les erreurs de type (devrait être un tableau mais ne l'est pas)
db.products.find({
    tags: { $exists: true, $not: { $type: "array" } }
})
```

### Vérifier les Types d'Éléments dans un Tableau

Pour vérifier le type des éléments **à l'intérieur** d'un tableau, utilisez `$elemMatch` :

```javascript
// Trouver les documents où au moins un élément du tableau est une chaîne
db.data.find({
    values: {
        $elemMatch: { $type: "string" }
    }
})

// Tableau de nombres
db.stats.find({
    scores: {
        $elemMatch: { $type: "number" }
    }
})
```

---

## Comparaison avec SQL

Les opérateurs d'éléments n'ont pas d'équivalent direct en SQL traditionnel, car SQL impose un schéma strict. Voici néanmoins quelques approximations :

| Concept | SQL (approximation) | MongoDB |
|---------|---------------------|---------|
| Vérifier si une colonne n'est pas NULL | `WHERE phone IS NOT NULL` | `{ phone: { $exists: true } }` |
| Vérifier si une colonne est NULL | `WHERE phone IS NULL` | `{ phone: { $exists: false } }` ou `{ phone: null }` |
| Vérifier le type (peu courant en SQL) | `WHERE typeof(age) = 'integer'` | `{ age: { $type: "int" } }` |

**Note** : En SQL, toutes les lignes ont les mêmes colonnes (nulles ou non), tandis qu'en MongoDB, les documents peuvent avoir des structures différentes.

---

## Bonnes Pratiques

### 1. Utiliser `$exists` pour les Champs Obligatoires

```javascript
// Identifier les documents problématiques
db.users.find({
    email: { $exists: false }
})

// Créer un rapport des données manquantes
const missing = {
    withoutEmail: db.users.countDocuments({ email: { $exists: false } }),
    withoutPhone: db.users.countDocuments({ phone: { $exists: false } }),
    withoutAge: db.users.countDocuments({ age: { $exists: false } })
}
```

### 2. Valider les Types avec `$type`

```javascript
// Avant de traiter des données, vérifier les types
db.products.find({
    price: { $type: "number" },
    stock: { $type: "number" }
})

// Identifier les données à nettoyer
db.products.find({
    price: { $not: { $type: "number" } }
})
```

### 3. Combiner pour une Validation Stricte

```javascript
// Validation complète d'un document
db.users.find({
    email: { $exists: true, $type: "string", $ne: "" },
    age: { $exists: true, $type: "number", $gte: 0 },
    status: { $exists: true, $in: ["active", "inactive", "pending"] }
})
```

### 4. Documenter les Champs Optionnels

```javascript
// Clairement indiquer quels champs sont optionnels
// Champs obligatoires : name, email
// Champs optionnels : phone, bio, website

// Vérifier les documents complets
db.users.find({
    name: { $exists: true },
    email: { $exists: true },
    phone: { $exists: true },  // Optionnel mais présent
    bio: { $exists: true }     // Optionnel mais présent
})
```

### 5. Créer des Index pour `$exists`

Pour améliorer les performances sur les requêtes fréquentes avec `$exists` :

```javascript
// Index sparse pour les champs optionnels
db.users.createIndex({ phone: 1 }, { sparse: true })

// La requête utilisera l'index efficacement
db.users.find({ phone: { $exists: true } })
```

### 6. Utiliser dans les Validations de Schéma

```javascript
// Définir des règles de validation
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            required: ["name", "email"],
            properties: {
                name: { bsonType: "string" },
                email: { bsonType: "string" },
                age: { bsonType: "int" },
                phone: { bsonType: "string" }
            }
        }
    }
})
```

---

## Pièges Courants à Éviter

### 1. Confusion entre `null` et Champ Inexistant

```javascript
// Documents
{ name: "Alice", phone: "123456789" }
{ name: "Bob", phone: null }
{ name: "Charlie" }

// ⚠️ Attention : cette requête retourne Bob ET Charlie
db.users.find({ phone: null })

// ✅ Pour trouver seulement les documents avec phone = null
db.users.find({
    phone: { $exists: true, $eq: null }
})

// ✅ Pour trouver seulement les documents sans le champ phone
db.users.find({ phone: { $exists: false } })
```

### 2. Type `"number"` vs Types Numériques Spécifiques

```javascript
// ⚠️ Peut matcher plusieurs types numériques
db.products.find({ price: { $type: "number" } })
// Retourne int, long, double, decimal

// ✅ Pour un type spécifique
db.products.find({ price: { $type: "double" } })
```

### 3. Oublier que `$exists: true` Inclut `null`

```javascript
// ⚠️ Inclut les valeurs null
db.users.find({ bio: { $exists: true } })

// ✅ Exclure explicitement les null
db.users.find({
    bio: { $exists: true, $ne: null }
})

// ✅ Ou vérifier que c'est une chaîne non vide
db.users.find({
    bio: { $exists: true, $type: "string", $ne: "" }
})
```

### 4. Type Alias Incomplet

```javascript
// ⚠️ "number" n'existe pas pour tous les nombres
// Il existe "int", "long", "double", "decimal"

// ✅ Utiliser l'alias "number" pour tous les types numériques
db.products.find({ quantity: { $type: "number" } })

// ✅ Ou spécifier les types explicitement
db.products.find({
    quantity: { $type: ["int", "long", "double"] }
})
```

### 5. Performance avec `$exists: false`

```javascript
// ⚠️ Peut être lent sur de grandes collections
db.users.find({ optionalField: { $exists: false } })

// ✅ Créer un index sparse si les requêtes sont fréquentes
db.users.createIndex({ optionalField: 1 }, { sparse: true })
```

---

## Performance et Optimisation

### Impact de `$exists`

#### `$exists: true`
- Peut utiliser un index sparse si disponible
- Plus performant avec un index approprié
- Scan complet possible sans index

#### `$exists: false`
- Ne peut pas utiliser d'index classique
- Nécessite généralement un scan complet
- Peut être lent sur de grandes collections

### Index Sparse pour `$exists`

Un **index sparse** n'indexe que les documents où le champ existe :

```javascript
// Créer un index sparse
db.users.createIndex({ phone: 1 }, { sparse: true })

// Cette requête utilisera l'index efficacement
db.users.find({ phone: { $exists: true } })

// Mais pas celle-ci (nécessite un scan complet)
db.users.find({ phone: { $exists: false } })
```

### Impact de `$type`

L'opérateur `$type` nécessite généralement un scan des documents car MongoDB doit vérifier le type BSON de chaque valeur.

```javascript
// Peut nécessiter un scan complet
db.products.find({ price: { $type: "string" } })
```

### Optimisation avec `explain()`

```javascript
// Analyser une requête avec $exists
db.users.find({
    phone: { $exists: true }
}).explain("executionStats")

// Analyser une requête avec $type
db.products.find({
    price: { $type: "number" }
}).explain("executionStats")

// Vérifier :
// - "executionTimeMillis" : temps d'exécution
// - "totalDocsExamined" : documents examinés
// - "stage" : IXSCAN (index) vs COLLSCAN (scan complet)
```

### Stratégies d'Optimisation

```javascript
// 1. Combiner avec des filtres sélectifs
db.products.find({
    category: "Electronics",  // Index sur category (très sélectif)
    price: { $type: "number" }
})

// 2. Utiliser des index partiels pour des cas spécifiques
db.users.createIndex(
    { phone: 1 },
    {
        partialFilterExpression: {
            phone: { $exists: true }
        }
    }
)

// 3. Limiter les résultats
db.users.find({
    email: { $exists: false }
}).limit(100)
```

---

## Points Clés à Retenir

✅ **`$exists`** vérifie si un champ existe (`true`) ou n'existe pas (`false`) dans un document

✅ Un champ avec la valeur **`null` est considéré comme existant** par `$exists`

✅ **`$type`** vérifie le type BSON d'un champ (string, number, date, array, etc.)

✅ Vous pouvez rechercher **plusieurs types** avec `$type` en utilisant un tableau

✅ L'alias **`"number"`** correspond à tous les types numériques (int, long, double, decimal)

✅ **Combiner** `$exists` et `$type` permet une validation précise des données

✅ Les deux opérateurs fonctionnent avec **la notation pointée** pour les champs imbriqués

✅ Les **index sparse** améliorent les performances de `$exists: true`

✅ `$exists: false` nécessite généralement un **scan complet** de la collection

✅ Ces opérateurs sont essentiels pour **l'audit**, la **validation** et la **migration** de données

---

## Résumé

Dans ce chapitre, vous avez appris :

- Comment utiliser `$exists` pour vérifier la présence ou l'absence de champs
- Comment utiliser `$type` pour valider les types BSON des champs
- Les différents types BSON disponibles et leurs alias
- Comment combiner ces opérateurs pour des validations strictes
- Les cas d'usage pratiques : audit, migration, validation de formulaires
- Les différences importantes entre `null` et champs inexistants
- Les bonnes pratiques et pièges à éviter
- Les considérations de performance et d'indexation

Ces opérateurs sont particulièrement utiles dans un contexte MongoDB où le schéma est flexible. Ils vous permettent de maintenir la qualité des données et de gérer les migrations de schéma efficacement.

Dans le prochain chapitre, nous explorerons les **opérateurs d'évaluation** qui vous permettront d'effectuer des requêtes encore plus sophistiquées avec des expressions régulières, des évaluations d'expressions et d'autres techniques avancées.

---


⏭️ [Opérateurs d'évaluation ($regex, $expr, $mod, $text, $where)](/03-requetes-et-filtres/05-operateurs-evaluation.md)
