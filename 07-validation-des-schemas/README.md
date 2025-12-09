🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Validation des Schémas

## Garantir l'intégrité de vos données ! 🛡️

Vous avez découvert la **flexibilité** de MongoDB : pas de schéma rigide, documents variables, évolution facile. Mais avec cette grande liberté vient une question cruciale : **comment garantir la qualité et la cohérence de vos données ?** Comment s'assurer qu'un champ email contient bien un email valide ? Qu'un âge est un nombre positif ? Qu'un document respecte votre modèle de données ?

C'est là qu'intervient la **validation de schémas**. MongoDB vous permet de définir des règles de validation tout en conservant la flexibilité qui fait sa force. Ce chapitre va vous montrer comment établir un équilibre parfait entre souplesse et rigueur.

## Où en sommes-nous dans votre parcours ?

Vous avez complété les chapitres 1 à 6 et vous maîtrisez maintenant :
- ✅ Les fondamentaux de MongoDB et les opérations CRUD
- ✅ La modélisation des données et les patterns de conception
- ✅ L'optimisation avec les index
- ✅ Le framework d'agrégation pour l'analyse de données
- ✅ Les structures de documents flexibles

**Parfait !** Vous êtes maintenant prêt à apprendre comment **sécuriser et valider** vos structures de données.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- ✅ **Comprendre** le rôle de la validation dans MongoDB
- ✅ **Définir** des règles de validation avec JSON Schema
- ✅ **Créer** des validations pour tous les types de données
- ✅ **Valider** les champs obligatoires et optionnels
- ✅ **Configurer** les niveaux de validation (strict, moderate)
- ✅ **Choisir** les actions appropriées (error, warn)
- ✅ **Appliquer** des validations personnalisées avec $expr
- ✅ **Migrer** des collections existantes vers la validation
- ✅ **Suivre** les bonnes pratiques de validation

## Le paradoxe MongoDB : Flexibilité vs Contrôle

### La promesse de MongoDB : Schema-less

```javascript
// MongoDB accepte tout, par défaut
db.users.insertOne({
    name: "Alice",
    age: 28
})

db.users.insertOne({
    username: "bob",        // Champ différent
    age: "trente-cinq",     // Type différent
    email: "pas-un-email"   // Données invalides
})

db.users.insertOne({
    nom: "Charlie",         // Faute de frappe
    âge: -5                 // Valeur absurde
})

// Tous insérés avec succès ! 😱
```

**Problème :** Sans validation, les données peuvent devenir incohérentes et causer des bugs dans votre application.

### La solution : Validation de schémas

```javascript
// Définir des règles de validation
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "email", "age"],
            properties: {
                name: {
                    bsonType: "string",
                    description: "doit être une chaîne et est obligatoire"
                },
                email: {
                    bsonType: "string",
                    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
                    description: "doit être un email valide et est obligatoire"
                },
                age: {
                    bsonType: "int",
                    minimum: 0,
                    maximum: 150,
                    description: "doit être un entier entre 0 et 150"
                }
            }
        }
    }
})

// Maintenant, les données invalides sont rejetées
db.users.insertOne({
    name: "Bob",
    email: "pas-un-email",  // ❌ ERREUR : email invalide
    age: "trente-cinq"      // ❌ ERREUR : doit être un entier
})
// MongoServerError: Document failed validation

// Données valides acceptées
db.users.insertOne({
    name: "Bob",
    email: "bob@example.com",  // ✅ Email valide
    age: 35                     // ✅ Entier dans la plage
})
// Insertion réussie !
```

**Avantage :** Vous conservez la flexibilité de MongoDB tout en garantissant la qualité des données.

## Vue d'ensemble du chapitre

Ce chapitre est organisé en 10 sections qui couvrent tous les aspects de la validation :

### 🎯 Partie 1 : Fondamentaux (Sections 7.1 à 7.3)
- **7.1** : Introduction à la validation de schéma
- **7.2** : JSON Schema dans MongoDB
- **7.3** : Règles de validation avec $jsonSchema

### 🎯 Partie 2 : Configuration (Sections 7.4 et 7.5)
- **7.4** : Niveaux de validation (strict, moderate)
- **7.5** : Actions de validation (error, warn)

### 🎯 Partie 3 : Gestion (Section 7.6)
- **7.6** : Modification des règles de validation

### 🎯 Partie 4 : Cas d'usage (Sections 7.7 à 7.9)
- **7.7** : Validation des types de données
- **7.8** : Validation des champs obligatoires
- **7.9** : Validation personnalisée avec $expr

### 🎯 Partie 5 : Production (Section 7.10)
- **7.10** : Bonnes pratiques de validation

## Comparaison avec SQL : Contraintes vs Validation

### Contraintes SQL (rigides)

```sql
-- Schéma défini à la création
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    age INT CHECK (age >= 0 AND age <= 150),
    registration_date DATE NOT NULL DEFAULT CURRENT_DATE,
    CONSTRAINT email_format CHECK (email LIKE '%@%.%')
);

-- Impossible d'insérer sans tous les champs définis
-- Impossible d'ajouter un champ non déclaré
-- Modification du schéma = ALTER TABLE (coûteux)
```

**Caractéristiques SQL :**
- 🔒 Schéma strict défini à l'avance
- ⚠️ Modification coûteuse (ALTER TABLE)
- ❌ Pas de flexibilité
- ✅ Contraintes toujours appliquées

### Validation MongoDB (flexible)

```javascript
// Validation optionnelle, appliquée après création
db.users.insertOne({
    name: "Alice",
    age: 28
})  // ✅ OK même sans validation

// Ajouter la validation plus tard
db.runCommand({
    collMod: "users",
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "email"],
            properties: {
                name: { bsonType: "string" },
                email: {
                    bsonType: "string",
                    pattern: "^.+@.+\\..+$"
                },
                age: {
                    bsonType: "int",
                    minimum: 0,
                    maximum: 150
                },
                // Champs supplémentaires autorisés par défaut
                additionalProperties: true
            }
        }
    },
    validationLevel: "moderate"  // N'applique qu'aux nouveaux documents
})

// Ajouter des champs non définis = OK
db.users.insertOne({
    name: "Bob",
    email: "bob@example.com",
    age: 35,
    favoriteColor: "blue"  // ✅ Champ supplémentaire autorisé
})
```

**Caractéristiques MongoDB :**
- 🔓 Validation optionnelle
- 🔄 Modification facile (collMod)
- ✅ Flexibilité conservée
- ⚙️ Niveau de validation configurable

## JSON Schema : Le standard de validation

MongoDB utilise **JSON Schema**, un standard international pour valider les structures JSON/BSON.

### Structure de base

```javascript
{
    $jsonSchema: {
        bsonType: "object",           // Type du document racine
        required: ["field1", "field2"], // Champs obligatoires
        properties: {                 // Définition des propriétés
            field1: {
                bsonType: "string",
                description: "Description pour les erreurs"
            },
            field2: {
                bsonType: "int",
                minimum: 0
            }
        },
        additionalProperties: true    // Autoriser champs supplémentaires
    }
}
```

## Exemples progressifs de validation

### Exemple 1 : Validation simple - Utilisateurs

```javascript
// Collection users avec validation de base
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["username", "email", "createdAt"],
            properties: {
                username: {
                    bsonType: "string",
                    minLength: 3,
                    maxLength: 30,
                    pattern: "^[a-zA-Z0-9_]+$",
                    description: "Nom d'utilisateur : 3-30 caractères alphanumériques"
                },
                email: {
                    bsonType: "string",
                    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
                    description: "Adresse email valide"
                },
                age: {
                    bsonType: ["int", "null"],  // Optionnel mais si présent = int
                    minimum: 13,
                    maximum: 120,
                    description: "Âge entre 13 et 120 ans si fourni"
                },
                createdAt: {
                    bsonType: "date",
                    description: "Date de création obligatoire"
                },
                isActive: {
                    bsonType: "bool",
                    description: "Statut actif (booléen)"
                }
            }
        }
    }
})

// ✅ Document valide
db.users.insertOne({
    username: "alice_dev",
    email: "alice@example.com",
    age: 28,
    createdAt: new Date(),
    isActive: true
})

// ❌ Erreur : username trop court
db.users.insertOne({
    username: "ab",  // Moins de 3 caractères
    email: "alice@example.com",
    createdAt: new Date()
})
// Error: Document failed validation
// username: must be at least 3 characters

// ❌ Erreur : email invalide
db.users.insertOne({
    username: "alice",
    email: "pas-un-email",  // Format invalide
    createdAt: new Date()
})
// Error: email: must match pattern

// ❌ Erreur : champ obligatoire manquant
db.users.insertOne({
    username: "alice",
    email: "alice@example.com"
    // createdAt manquant
})
// Error: Missing required field: createdAt
```

### Exemple 2 : Validation complexe - Produits e-commerce

```javascript
db.createCollection("products", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "sku", "price", "category", "stock"],
            properties: {
                name: {
                    bsonType: "string",
                    minLength: 1,
                    maxLength: 200,
                    description: "Nom du produit"
                },
                sku: {
                    bsonType: "string",
                    pattern: "^[A-Z]{3}-[0-9]{6}$",
                    description: "SKU format: ABC-123456"
                },
                price: {
                    bsonType: "double",
                    minimum: 0.01,
                    description: "Prix > 0"
                },
                category: {
                    enum: ["Électronique", "Informatique", "Accessoires", "Livres"],
                    description: "Catégorie parmi la liste définie"
                },
                stock: {
                    bsonType: "int",
                    minimum: 0,
                    description: "Stock >= 0"
                },
                tags: {
                    bsonType: "array",
                    items: {
                        bsonType: "string",
                        minLength: 2,
                        maxLength: 30
                    },
                    minItems: 1,
                    maxItems: 10,
                    uniqueItems: true,
                    description: "1-10 tags uniques"
                },
                specifications: {
                    bsonType: "object",
                    properties: {
                        weight: {
                            bsonType: "double",
                            minimum: 0
                        },
                        dimensions: {
                            bsonType: "object",
                            required: ["length", "width", "height"],
                            properties: {
                                length: { bsonType: "double", minimum: 0 },
                                width: { bsonType: "double", minimum: 0 },
                                height: { bsonType: "double", minimum: 0 }
                            }
                        }
                    },
                    additionalProperties: true  // Autres specs autorisées
                },
                discount: {
                    bsonType: "object",
                    required: ["percentage", "startDate", "endDate"],
                    properties: {
                        percentage: {
                            bsonType: "double",
                            minimum: 0,
                            maximum: 100
                        },
                        startDate: { bsonType: "date" },
                        endDate: { bsonType: "date" }
                    }
                },
                status: {
                    enum: ["draft", "active", "discontinued", "out_of_stock"],
                    description: "Statut du produit"
                }
            }
        }
    }
})

// ✅ Document valide complet
db.products.insertOne({
    name: "Ordinateur portable Dell XPS 15",
    sku: "DEL-123456",
    price: 1299.99,
    category: "Informatique",
    stock: 25,
    tags: ["laptop", "dell", "gaming", "portable"],
    specifications: {
        weight: 2.1,
        dimensions: {
            length: 35.7,
            width: 23.5,
            height: 1.8
        },
        processor: "Intel i7",  // Champ additionnel autorisé
        ram: 16
    },
    discount: {
        percentage: 10,
        startDate: new Date("2024-01-01"),
        endDate: new Date("2024-01-31")
    },
    status: "active"
})

// ❌ Erreur : SKU format invalide
db.products.insertOne({
    name: "Produit test",
    sku: "123-ABC",  // Mauvais format (doit être ABC-123456)
    price: 99.99,
    category: "Informatique",
    stock: 10
})

// ❌ Erreur : catégorie non valide
db.products.insertOne({
    name: "Produit test",
    sku: "PRO-123456",
    price: 99.99,
    category: "Cuisine",  // Pas dans l'enum
    stock: 10
})

// ❌ Erreur : prix négatif
db.products.insertOne({
    name: "Produit test",
    sku: "PRO-123456",
    price: -10.00,  // Prix négatif
    category: "Informatique",
    stock: 10
})

// ❌ Erreur : discount percentage > 100
db.products.insertOne({
    name: "Produit test",
    sku: "PRO-123456",
    price: 99.99,
    category: "Informatique",
    stock: 10,
    discount: {
        percentage: 150,  // > 100%
        startDate: new Date(),
        endDate: new Date()
    }
})
```

### Exemple 3 : Validation avec documents imbriqués - Commandes

```javascript
db.createCollection("orders", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["orderId", "customerId", "items", "total", "status", "createdAt"],
            properties: {
                orderId: {
                    bsonType: "string",
                    pattern: "^ORD-[0-9]{8}$"
                },
                customerId: {
                    bsonType: "objectId"
                },
                items: {
                    bsonType: "array",
                    minItems: 1,
                    maxItems: 50,
                    items: {
                        bsonType: "object",
                        required: ["productId", "quantity", "price"],
                        properties: {
                            productId: {
                                bsonType: "objectId"
                            },
                            productName: {
                                bsonType: "string"
                            },
                            quantity: {
                                bsonType: "int",
                                minimum: 1,
                                maximum: 100
                            },
                            price: {
                                bsonType: "double",
                                minimum: 0
                            }
                        }
                    }
                },
                total: {
                    bsonType: "double",
                    minimum: 0
                },
                status: {
                    enum: ["pending", "processing", "shipped", "delivered", "cancelled"]
                },
                shippingAddress: {
                    bsonType: "object",
                    required: ["street", "city", "postalCode", "country"],
                    properties: {
                        street: { bsonType: "string", minLength: 5 },
                        city: { bsonType: "string", minLength: 2 },
                        postalCode: {
                            bsonType: "string",
                            pattern: "^[0-9]{5}$"  // Code postal français
                        },
                        country: {
                            bsonType: "string",
                            enum: ["France", "Belgique", "Suisse", "Luxembourg"]
                        }
                    }
                },
                createdAt: {
                    bsonType: "date"
                },
                deliveredAt: {
                    bsonType: ["date", "null"]
                }
            }
        }
    }
})

// ✅ Commande valide
db.orders.insertOne({
    orderId: "ORD-20240115",
    customerId: ObjectId("507f1f77bcf86cd799439011"),
    items: [
        {
            productId: ObjectId("507f1f77bcf86cd799439012"),
            productName: "Ordinateur",
            quantity: 1,
            price: 1299.99
        },
        {
            productId: ObjectId("507f1f77bcf86cd799439013"),
            productName: "Souris",
            quantity: 2,
            price: 29.99
        }
    ],
    total: 1359.97,
    status: "pending",
    shippingAddress: {
        street: "123 rue de la Paix",
        city: "Paris",
        postalCode: "75001",
        country: "France"
    },
    createdAt: new Date()
})

// ❌ Erreur : items vide
db.orders.insertOne({
    orderId: "ORD-20240115",
    customerId: ObjectId("507f1f77bcf86cd799439011"),
    items: [],  // Tableau vide (minItems: 1)
    total: 0,
    status: "pending",
    createdAt: new Date()
})

// ❌ Erreur : quantité trop élevée
db.orders.insertOne({
    orderId: "ORD-20240115",
    customerId: ObjectId("507f1f77bcf86cd799439011"),
    items: [
        {
            productId: ObjectId("507f1f77bcf86cd799439012"),
            quantity: 150,  // > maximum (100)
            price: 1299.99
        }
    ],
    total: 194998.50,
    status: "pending",
    createdAt: new Date()
})

// ❌ Erreur : code postal invalide
db.orders.insertOne({
    orderId: "ORD-20240115",
    customerId: ObjectId("507f1f77bcf86cd799439011"),
    items: [
        {
            productId: ObjectId("507f1f77bcf86cd799439012"),
            quantity: 1,
            price: 1299.99
        }
    ],
    total: 1299.99,
    status: "pending",
    shippingAddress: {
        street: "123 rue test",
        city: "Paris",
        postalCode: "ABCDE",  // Pas au format numérique
        country: "France"
    },
    createdAt: new Date()
})
```

## Niveaux de validation : strict vs moderate

MongoDB offre deux niveaux de validation :

### Niveau "strict" (par défaut)

```javascript
db.runCommand({
    collMod: "users",
    validator: { /* règles */ },
    validationLevel: "strict"
})
```

**Comportement :**
- ✅ Appliqué aux insertions (insertOne, insertMany)
- ✅ Appliqué aux mises à jour (updateOne, updateMany, replaceOne)
- ✅ Tous les documents doivent être valides

**Usage :** Nouvelles collections ou collections avec données déjà propres.

### Niveau "moderate"

```javascript
db.runCommand({
    collMod: "users",
    validator: { /* règles */ },
    validationLevel: "moderate"
})
```

**Comportement :**
- ✅ Appliqué aux insertions
- ⚠️ Pour les mises à jour : appliqué SEULEMENT si le document était déjà valide
- 🔄 Les documents invalides existants peuvent être mis à jour sans respecter les règles

**Usage :** Migration progressive, collections existantes avec données historiques.

### Exemple pratique

```javascript
// Collection existante avec données incohérentes
db.users.insertMany([
    { name: "Alice", age: 28 },           // Valide
    { name: "Bob", age: "trente-cinq" },  // Age incorrect
    { username: "charlie", age: 30 }      // Champ name manquant
])

// Ajouter validation en mode moderate
db.runCommand({
    collMod: "users",
    validator: {
        $jsonSchema: {
            required: ["name"],
            properties: {
                name: { bsonType: "string" },
                age: { bsonType: "int" }
            }
        }
    },
    validationLevel: "moderate"
})

// ✅ Nouvelle insertion : validation stricte
db.users.insertOne({
    name: "David",
    age: 35
})  // OK

db.users.insertOne({
    username: "david",
    age: 35
})  // ❌ ERREUR : name manquant

// ⚠️ Mise à jour document invalide (Bob) : validation ignorée
db.users.updateOne(
    { name: "Bob" },
    { $set: { city: "Lyon" } }
)  // ✅ OK car document déjà invalide

// ✅ Mise à jour document valide (Alice) : validation appliquée
db.users.updateOne(
    { name: "Alice" },
    { $set: { age: "vingt-huit" } }
)  // ❌ ERREUR : age doit être int
```

## Actions de validation : error vs warn

MongoDB permet de choisir l'action en cas de non-validation :

### Action "error" (par défaut)

```javascript
db.runCommand({
    collMod: "users",
    validator: { /* règles */ },
    validationAction: "error"
})
```

**Comportement :**
- ❌ Rejette l'opération
- 🚫 Retourne une erreur
- 📝 Log dans les logs MongoDB

**Usage :** Production, données critiques.

### Action "warn"

```javascript
db.runCommand({
    collMod: "users",
    validator: { /* règles */ },
    validationAction: "warn"
})
```

**Comportement :**
- ✅ Accepte l'opération
- ⚠️ Log un avertissement
- 📊 Permet de surveiller sans bloquer

**Usage :** Phase de test, monitoring, transition progressive.

### Exemple pratique

```javascript
// Mode warn pour surveiller
db.createCollection("products", {
    validator: {
        $jsonSchema: {
            required: ["name", "price"],
            properties: {
                name: { bsonType: "string" },
                price: { bsonType: "double", minimum: 0 }
            }
        }
    },
    validationAction: "warn"  // ⚠️ Avertir seulement
})

// Ces insertions réussissent mais génèrent des warnings
db.products.insertOne({
    name: "Produit A",
    price: -10  // ❌ Prix négatif mais insertion OK
})
// ✅ Insertion réussie
// ⚠️ Warning dans les logs

db.products.insertOne({
    description: "Produit sans nom",
    price: 99.99
})  // ❌ name manquant mais insertion OK
// ✅ Insertion réussie
// ⚠️ Warning dans les logs

// Analyser les warnings dans les logs
db.adminCommand({ getLog: "global" })
```

## Validation avec expressions personnalisées ($expr)

Pour des règles plus complexes, utilisez `$expr` :

### Exemple 1 : Comparer deux champs

```javascript
db.createCollection("events", {
    validator: {
        $expr: {
            $and: [
                // endDate doit être après startDate
                { $gte: ["$endDate", "$startDate"] },
                // duration doit correspondre à la différence
                {
                    $eq: [
                        "$duration",
                        {
                            $dateDiff: {
                                startDate: "$startDate",
                                endDate: "$endDate",
                                unit: "hour"
                            }
                        }
                    ]
                }
            ]
        }
    }
})

// ✅ Valide
db.events.insertOne({
    name: "Conférence MongoDB",
    startDate: ISODate("2024-01-15T09:00:00Z"),
    endDate: ISODate("2024-01-15T17:00:00Z"),
    duration: 8  // 8 heures
})

// ❌ Erreur : endDate avant startDate
db.events.insertOne({
    name: "Event invalide",
    startDate: ISODate("2024-01-15T17:00:00Z"),
    endDate: ISODate("2024-01-15T09:00:00Z"),
    duration: -8
})

// ❌ Erreur : duration ne correspond pas
db.events.insertOne({
    name: "Event",
    startDate: ISODate("2024-01-15T09:00:00Z"),
    endDate: ISODate("2024-01-15T17:00:00Z"),
    duration: 10  // Devrait être 8
})
```

### Exemple 2 : Validation conditionnelle

```javascript
db.createCollection("products", {
    validator: {
        $expr: {
            // Si discount existe, startDate et endDate doivent exister
            $or: [
                { $not: { $gt: ["$discount", null] } },  // Pas de discount
                {
                    $and: [
                        { $gt: ["$discount.startDate", null] },
                        { $gt: ["$discount.endDate", null] },
                        { $gte: ["$discount.endDate", "$discount.startDate"] }
                    ]
                }
            ]
        }
    }
})

// ✅ Produit sans discount
db.products.insertOne({
    name: "Produit A",
    price: 99.99
})

// ✅ Produit avec discount complet
db.products.insertOne({
    name: "Produit B",
    price: 99.99,
    discount: {
        percentage: 10,
        startDate: ISODate("2024-01-01"),
        endDate: ISODate("2024-01-31")
    }
})

// ❌ Erreur : discount incomplet
db.products.insertOne({
    name: "Produit C",
    price: 99.99,
    discount: {
        percentage: 10
        // startDate et endDate manquants
    }
})
```

### Exemple 3 : Validation basée sur calculs

```javascript
db.createCollection("orders", {
    validator: {
        $expr: {
            // Le total doit correspondre à la somme des items
            $eq: [
                "$total",
                {
                    $reduce: {
                        input: "$items",
                        initialValue: 0,
                        in: {
                            $add: [
                                "$$value",
                                { $multiply: ["$$this.quantity", "$$this.price"] }
                            ]
                        }
                    }
                }
            ]
        }
    }
})

// ✅ Total correct
db.orders.insertOne({
    orderId: "ORD-001",
    items: [
        { quantity: 2, price: 10.00 },  // 20
        { quantity: 1, price: 30.00 }   // 30
    ],
    total: 50.00  // 20 + 30 = 50 ✅
})

// ❌ Erreur : total incorrect
db.orders.insertOne({
    orderId: "ORD-002",
    items: [
        { quantity: 2, price: 10.00 },
        { quantity: 1, price: 30.00 }
    ],
    total: 100.00  // Incorrect (devrait être 50)
})
```

## Combiner JSON Schema et $expr

Vous pouvez combiner les deux approches :

```javascript
db.createCollection("bookings", {
    validator: {
        $and: [
            // JSON Schema pour structure de base
            {
                $jsonSchema: {
                    required: ["roomId", "checkIn", "checkOut", "guestCount"],
                    properties: {
                        roomId: { bsonType: "objectId" },
                        checkIn: { bsonType: "date" },
                        checkOut: { bsonType: "date" },
                        guestCount: { bsonType: "int", minimum: 1, maximum: 10 },
                        totalPrice: { bsonType: "double", minimum: 0 }
                    }
                }
            },
            // $expr pour logique complexe
            {
                $expr: {
                    $and: [
                        // checkOut après checkIn
                        { $gt: ["$checkOut", "$checkIn"] },
                        // Minimum 1 nuit
                        {
                            $gte: [
                                {
                                    $dateDiff: {
                                        startDate: "$checkIn",
                                        endDate: "$checkOut",
                                        unit: "day"
                                    }
                                },
                                1
                            ]
                        },
                        // checkIn dans le futur (ou aujourd'hui)
                        { $gte: ["$checkIn", "$$NOW"] }
                    ]
                }
            }
        ]
    }
})
```

## Modifier la validation d'une collection existante

```javascript
// Voir la validation actuelle
db.getCollectionInfos({ name: "users" })

// Modifier la validation
db.runCommand({
    collMod: "users",
    validator: {
        $jsonSchema: {
            // Nouvelles règles
        }
    },
    validationLevel: "strict",  // ou "moderate"
    validationAction: "error"   // ou "warn"
})

// Supprimer complètement la validation
db.runCommand({
    collMod: "users",
    validator: {},
    validationLevel: "off"
})
```

## Migration progressive vers la validation

### Stratégie recommandée

```javascript
// Phase 1 : Analyser les données existantes
db.users.aggregate([
    {
        $group: {
            _id: null,
            missingEmail: {
                $sum: { $cond: [{ $eq: [{ $type: "$email" }, "missing"] }, 1, 0] }
            },
            invalidEmailType: {
                $sum: { $cond: [{ $ne: [{ $type: "$email" }, "string"] }, 1, 0] }
            },
            invalidAge: {
                $sum: { $cond: [
                    { $or: [
                        { $lt: ["$age", 0] },
                        { $gt: ["$age", 150] }
                    ]},
                    1,
                    0
                ]}
            }
        }
    }
])

// Phase 2 : Nettoyer les données invalides
db.users.updateMany(
    { email: { $type: "string", $not: { $regex: /@/ } } },
    { $set: { email: "invalid@needsupdate.com" } }
)

// Phase 3 : Activer validation en mode warn
db.runCommand({
    collMod: "users",
    validator: { /* règles */ },
    validationAction: "warn"
})

// Phase 4 : Surveiller les warnings pendant 1-2 semaines

// Phase 5 : Passer en mode error
db.runCommand({
    collMod: "users",
    validationAction: "error"
})
```

## Bonnes pratiques de validation

### ✅ À faire

1. **Commencer simple** : validez les champs critiques d'abord
2. **Utiliser des descriptions** : pour des messages d'erreur clairs
3. **Tester abondamment** : avant d'activer en production
4. **Mode moderate pour migration** : collections existantes
5. **Mode warn en phase de test** : surveiller sans bloquer
6. **Documenter les règles** : pourquoi chaque validation existe

### ❌ À éviter

1. **Validation trop stricte** : perte de flexibilité MongoDB
2. **Règles non testées** : peut bloquer opérations légitimes
3. **Ignorer les performances** : $expr peut être coûteux
4. **Validation côté application ET base** : duplication inutile
5. **Changer fréquemment** : stabilité importante

## Performance de la validation

```javascript
// ⚠️ Validation complexe = impact performance
db.createCollection("expensive", {
    validator: {
        $expr: {
            // Calcul coûteux sur chaque insertion
            $eq: [
                "$complexCalculation",
                {
                    $reduce: {
                        input: { $range: [0, 1000] },
                        initialValue: 0,
                        in: { $add: ["$$value", "$$this"] }
                    }
                }
            ]
        }
    }
})

// ✅ Validation simple = impact minimal
db.createCollection("efficient", {
    validator: {
        $jsonSchema: {
            required: ["name", "age"],
            properties: {
                name: { bsonType: "string" },
                age: { bsonType: "int", minimum: 0 }
            }
        }
    }
})
```

## Exemples de validations courantes

### Email valide

```javascript
{
    email: {
        bsonType: "string",
        pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
    }
}
```

### Téléphone français

```javascript
{
    phone: {
        bsonType: "string",
        pattern: "^(\\+33|0)[1-9][0-9]{8}$"
    }
}
```

### URL

```javascript
{
    website: {
        bsonType: "string",
        pattern: "^https?://[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}(/.*)?$"
    }
}
```

### Code postal français

```javascript
{
    postalCode: {
        bsonType: "string",
        pattern: "^[0-9]{5}$"
    }
}
```

### Date dans le futur

```javascript
{
    $expr: {
        $gte: ["$eventDate", "$$NOW"]
    }
}
```

## Conseils d'apprentissage

### 🎯 Méthodologie

1. **Identifiez les champs critiques** : quelles données doivent absolument être valides ?
2. **Commencez par JSON Schema** : plus simple et performant
3. **Ajoutez $expr si nécessaire** : pour logique complexe
4. **Testez en mode warn** : avant d'activer error
5. **Documentez vos règles** : maintenabilité

### 🔗 Lien avec les autres chapitres

- **Chapitre 4** : La modélisation guide les règles de validation
- **Chapitre 15** : Les drivers gèrent les erreurs de validation
- **Chapitre 21** : Validation = bonne pratique essentielle

---

### 📌 Points clés à retenir

- La validation MongoDB combine flexibilité et contrôle qualité
- JSON Schema pour validation structurelle
- $expr pour logique complexe et inter-champs
- Deux niveaux : strict (toujours) et moderate (nouveaux docs)
- Deux actions : error (rejeter) et warn (logger)
- Mode moderate + warn pour migration progressive
- Validation = équilibre entre rigueur et flexibilité
- Commencer simple, complexifier progressivement

---

**Durée estimée du chapitre** : 4-6 heures
**Niveau** : Intermédiaire avec compréhension de la modélisation
**Prérequis** : Chapitres 1-4 complétés

🎯 **Prochaine étape** : Section 7.1 pour approfondir les concepts de validation.

---

**Prochaine section** : 7.1 - Introduction à la validation de schéma

Prêt à sécuriser vos données ? Allons-y ! 🛡️

⏭️ [Introduction à la validation de schéma](/07-validation-des-schemas/01-introduction-validation.md)
