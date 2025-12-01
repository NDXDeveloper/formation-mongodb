🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.10 Requêtes sur Documents Imbriqués

## Introduction

L'une des grandes forces de MongoDB est sa capacité à stocker des **données structurées complexes** directement dans les documents. Contrairement aux bases de données relationnelles où vous devriez créer des tables séparées et des jointures, MongoDB vous permet d'imbriquer des objets et des tableaux directement dans vos documents.

Par exemple, un document utilisateur peut contenir :
- Des **objets imbriqués** : adresse, coordonnées de contact, préférences
- Des **tableaux d'objets** : historique des commandes, liste d'amis, compétences

Cette structure riche nécessite des techniques de requêtage spécifiques que nous allons explorer dans ce chapitre.

### Exemple de Document Complexe

```javascript
{
    _id: ObjectId("..."),
    name: "Alice Dupont",
    email: "alice@example.com",
    address: {                          // Objet imbriqué
        street: "123 Rue de la Paix",
        city: "Paris",
        zipCode: "75001",
        country: "France",
        coordinates: {                   // Objet imbriqué dans un objet
            latitude: 48.8566,
            longitude: 2.3522
        }
    },
    contact: {                          // Autre objet imbriqué
        phone: "+33612345678",
        emergencyContact: {
            name: "Bob Dupont",
            phone: "+33623456789"
        }
    },
    orders: [                           // Tableau d'objets
        {
            orderId: "ORD-001",
            date: ISODate("2024-01-15"),
            amount: 150.00,
            items: ["laptop", "mouse"]
        },
        {
            orderId: "ORD-002",
            date: ISODate("2024-02-20"),
            amount: 75.00,
            items: ["keyboard"]
        }
    ],
    preferences: {                      // Objet imbriqué
        newsletter: true,
        language: "fr",
        notifications: {
            email: true,
            sms: false
        }
    }
}
```

---

## La Notation Pointée (Dot Notation)

Pour accéder aux champs imbriqués dans MongoDB, nous utilisons la **notation pointée** : on sépare les niveaux par un point (`.`).

### Syntaxe

```javascript
"champ.souschamp.soussouschamp"
```

**Important** : Les chemins en notation pointée doivent être entre **guillemets**.

### Exemples de Chemins

```javascript
// Accéder à la ville dans l'adresse
"address.city"

// Accéder à la latitude dans les coordonnées
"address.coordinates.latitude"

// Accéder au téléphone d'urgence
"contact.emergencyContact.phone"

// Accéder à la préférence de newsletter
"preferences.newsletter"

// Accéder aux notifications par email
"preferences.notifications.email"
```

---

## Requêtes sur Objets Imbriqués Simples

### Recherche par Égalité

```javascript
// Trouver les utilisateurs à Paris
db.users.find({ "address.city": "Paris" })

// Trouver les utilisateurs en France
db.users.find({ "address.country": "France" })

// Trouver les utilisateurs avec newsletter activée
db.users.find({ "preferences.newsletter": true })

// Trouver les utilisateurs avec notifications email activées
db.users.find({ "preferences.notifications.email": true })
```

### Avec Opérateurs de Comparaison

```javascript
// Utilisateurs avec code postal commençant par 75 (Paris)
db.users.find({ "address.zipCode": { $regex: /^75/ } })

// Coordonnées dans une certaine latitude
db.users.find({ "address.coordinates.latitude": { $gte: 48, $lte: 49 } })

// Contact d'urgence existant
db.users.find({ "contact.emergencyContact.name": { $exists: true } })
```

### Requêtes sur Plusieurs Champs Imbriqués

```javascript
// Utilisateurs à Paris avec newsletter activée
db.users.find({
    "address.city": "Paris",
    "preferences.newsletter": true
})

// Utilisateurs en France avec notifications email
db.users.find({
    "address.country": "France",
    "preferences.notifications.email": true
})

// Coordonnées précises (latitude ET longitude)
db.users.find({
    "address.coordinates.latitude": { $gte: 48.8, $lte: 48.9 },
    "address.coordinates.longitude": { $gte: 2.3, $lte: 2.4 }
})
```

---

## Requêtes sur Objets Imbriqués Complets

Vous pouvez rechercher en spécifiant un **objet complet** ou des **champs individuels**.

### Correspondance d'Objet Exact

```javascript
// Document
{
    name: "Alice",
    address: {
        street: "123 Main St",
        city: "Paris",
        zipCode: "75001"
    }
}

// Recherche avec objet exact (ordre et champs doivent correspondre exactement)
db.users.find({
    address: {
        street: "123 Main St",
        city: "Paris",
        zipCode: "75001"
    }
})
// ✅ Correspond : objet identique dans le même ordre

// ❌ Ne correspond PAS : ordre différent
db.users.find({
    address: {
        city: "Paris",
        street: "123 Main St",
        zipCode: "75001"
    }
})

// ❌ Ne correspond PAS : champs manquants
db.users.find({
    address: {
        city: "Paris",
        zipCode: "75001"
    }
})
```

### Recherche par Champs Individuels (Recommandé)

```javascript
// ✅ Meilleure approche : utiliser la notation pointée
db.users.find({
    "address.city": "Paris",
    "address.zipCode": "75001"
})
// Correspond indépendamment de l'ordre ou des autres champs
```

**Conseil** : Privilégiez toujours la notation pointée pour plus de flexibilité.

---

## Requêtes sur Tableaux d'Objets

Les tableaux d'objets sont plus complexes car ils combinent deux niveaux de structures.

### Structure Exemple

```javascript
{
    name: "Alice",
    orders: [
        { orderId: "ORD-001", amount: 150, status: "completed" },
        { orderId: "ORD-002", amount: 75, status: "pending" },
        { orderId: "ORD-003", amount: 200, status: "completed" }
    ]
}
```

### Recherche Simple dans Tableau d'Objets

```javascript
// Trouver les utilisateurs ayant au moins une commande de 150
db.users.find({ "orders.amount": 150 })

// Trouver les utilisateurs avec au moins une commande complétée
db.users.find({ "orders.status": "completed" })

// Trouver les utilisateurs avec l'order ID spécifique
db.users.find({ "orders.orderId": "ORD-001" })
```

**Important** : Cette approche vérifie si **au moins un élément** du tableau correspond.

### Problème avec Conditions Multiples

```javascript
// Documents
{
    name: "Alice",
    orders: [
        { orderId: "ORD-001", amount: 150, status: "completed" },
        { orderId: "ORD-002", amount: 75, status: "pending" }
    ]
}

// ❌ Attention : cette requête peut donner des résultats inattendus
db.users.find({
    "orders.amount": { $gte: 100 },
    "orders.status": "completed"
})
// Correspond à Alice car :
// - Un ordre a amount >= 100 (ORD-001: 150)
// - Un ordre a status "completed" (ORD-001)
// Même si ce n'est pas nécessairement le MÊME ordre !
```

### Solution : L'Opérateur `$elemMatch`

`$elemMatch` garantit que **toutes les conditions** s'appliquent au **même élément** du tableau.

```javascript
// ✅ Correct : même élément doit satisfaire toutes les conditions
db.users.find({
    orders: {
        $elemMatch: {
            amount: { $gte: 100 },
            status: "completed"
        }
    }
})
// Ne correspond que si AU MOINS UN ordre a amount >= 100 ET status "completed"
```

### Exemples avec `$elemMatch`

```javascript
// Utilisateurs avec au moins une grosse commande complétée
db.users.find({
    orders: {
        $elemMatch: {
            amount: { $gte: 200 },
            status: "completed"
        }
    }
})

// Utilisateurs avec au moins une commande récente et chère
db.users.find({
    orders: {
        $elemMatch: {
            date: { $gte: ISODate("2024-01-01") },
            amount: { $gte: 100 }
        }
    }
})

// Commandes en attente avec montant spécifique
db.users.find({
    orders: {
        $elemMatch: {
            status: "pending",
            amount: { $lt: 100 }
        }
    }
})
```

---

## Requêtes Imbriquées à Plusieurs Niveaux

MongoDB supporte des niveaux d'imbrication arbitraires.

### Structure Profondément Imbriquée

```javascript
{
    name: "Alice",
    company: {
        name: "Tech Corp",
        address: {
            city: "Paris",
            coordinates: {
                latitude: 48.8566,
                longitude: 2.3522
            }
        },
        departments: [
            {
                name: "IT",
                manager: {
                    name: "Bob",
                    email: "bob@example.com"
                }
            }
        ]
    }
}
```

### Requêtes Multi-niveaux

```javascript
// Accéder à la ville de l'entreprise
db.employees.find({ "company.address.city": "Paris" })

// Accéder aux coordonnées
db.employees.find({
    "company.address.coordinates.latitude": { $gte: 48 }
})

// Recherche dans tableau imbriqué
db.employees.find({ "company.departments.name": "IT" })

// Avec $elemMatch pour garantir le même élément
db.employees.find({
    "company.departments": {
        $elemMatch: {
            name: "IT",
            "manager.name": "Bob"
        }
    }
})
```

---

## Opérateurs Spéciaux pour Documents Imbriqués

### L'Opérateur `$exists`

Vérifier l'existence de champs imbriqués :

```javascript
// Utilisateurs ayant renseigné leur adresse complète
db.users.find({
    "address.street": { $exists: true },
    "address.city": { $exists: true },
    "address.zipCode": { $exists: true }
})

// Utilisateurs avec contact d'urgence
db.users.find({
    "contact.emergencyContact": { $exists: true }
})

// Utilisateurs sans coordonnées GPS
db.users.find({
    "address.coordinates": { $exists: false }
})
```

### L'Opérateur `$type`

Vérifier le type de champs imbriqués :

```javascript
// Vérifier que le code postal est une chaîne
db.users.find({
    "address.zipCode": { $type: "string" }
})

// Vérifier que les coordonnées sont des nombres
db.users.find({
    "address.coordinates.latitude": { $type: "double" },
    "address.coordinates.longitude": { $type: "double" }
})

// Vérifier que orders est un tableau
db.users.find({
    orders: { $type: "array" }
})
```

### L'Opérateur `$ne` (Not Equal)

```javascript
// Utilisateurs pas à Paris
db.users.find({ "address.city": { $ne: "Paris" } })

// Newsletter désactivée
db.users.find({ "preferences.newsletter": { $ne: true } })

// Pas de contact d'urgence
db.users.find({
    "contact.emergencyContact": { $exists: false }
})
```

---

## Requêtes avec Expressions Régulières

Les regex fonctionnent aussi sur les champs imbriqués :

```javascript
// Emails professionnels (@company.com)
db.users.find({
    "contact.email": { $regex: /@company\.com$/i }
})

// Codes postaux parisiens (75xxx)
db.users.find({
    "address.zipCode": { $regex: /^75/ }
})

// Rues contenant "Main"
db.users.find({
    "address.street": { $regex: /Main/i }
})

// Téléphones français (+33...)
db.users.find({
    "contact.phone": { $regex: /^\+33/ }
})
```

---

## Projections sur Documents Imbriqués

Vous pouvez contrôler quels champs imbriqués retourner.

### Inclure des Champs Imbriqués Spécifiques

```javascript
// Seulement le nom et la ville
db.users.find(
    {},
    {
        name: 1,
        "address.city": 1
    }
)
// Résultat :
// {
//     _id: ObjectId("..."),
//     name: "Alice",
//     address: { city: "Paris" }
// }

// Plusieurs champs imbriqués
db.users.find(
    {},
    {
        name: 1,
        "address.city": 1,
        "address.country": 1,
        "contact.email": 1
    }
)
```

### Inclure un Objet Imbriqué Complet

```javascript
// Tout l'objet address
db.users.find(
    {},
    {
        name: 1,
        address: 1
    }
)
// Résultat :
// {
//     _id: ObjectId("..."),
//     name: "Alice",
//     address: {
//         street: "123 Main St",
//         city: "Paris",
//         zipCode: "75001",
//         coordinates: { ... }
//     }
// }
```

### Exclure des Champs Imbriqués

```javascript
// Exclure les coordonnées
db.users.find(
    {},
    {
        "address.coordinates": 0
    }
)

// Exclure plusieurs champs imbriqués
db.users.find(
    {},
    {
        "contact.phone": 0,
        "preferences.notifications": 0
    }
)
```

### Projections sur Tableaux d'Objets

```javascript
// Limiter les éléments du tableau avec $slice
db.users.find(
    {},
    {
        name: 1,
        orders: { $slice: 3 }  // Seulement les 3 premiers orders
    }
)

// Projeter certains champs des éléments du tableau
db.users.find(
    {},
    {
        name: 1,
        "orders.orderId": 1,
        "orders.amount": 1
    }
)
// Résultat :
// {
//     _id: ObjectId("..."),
//     name: "Alice",
//     orders: [
//         { orderId: "ORD-001", amount: 150 },
//         { orderId: "ORD-002", amount: 75 }
//     ]
// }
```

---

## Cas d'Usage Pratiques

### Cas 1 : E-commerce - Recherche d'Utilisateurs

```javascript
// Structure
{
    name: "Alice",
    email: "alice@example.com",
    shippingAddress: {
        street: "123 Main St",
        city: "Paris",
        country: "France",
        zipCode: "75001"
    },
    billingAddress: {
        street: "456 Oak Ave",
        city: "Lyon",
        country: "France",
        zipCode: "69001"
    },
    orders: [
        {
            orderId: "ORD-001",
            date: ISODate("2024-01-15"),
            total: 150.00,
            status: "delivered"
        }
    ]
}

// Requêtes
// Utilisateurs avec livraison à Paris
db.users.find({ "shippingAddress.city": "Paris" })

// Utilisateurs avec adresses dans différentes villes
db.users.find({
    $expr: {
        $ne: ["$shippingAddress.city", "$billingAddress.city"]
    }
})

// Utilisateurs avec au moins une commande livrée récemment
db.users.find({
    orders: {
        $elemMatch: {
            status: "delivered",
            date: { $gte: ISODate("2024-01-01") }
        }
    }
})

// Utilisateurs français avec commandes de plus de 100€
db.users.find({
    "shippingAddress.country": "France",
    orders: {
        $elemMatch: {
            total: { $gte: 100 }
        }
    }
})
```

### Cas 2 : Gestion d'Entreprise - Employés

```javascript
// Structure
{
    name: "Alice Dupont",
    position: "Developer",
    department: {
        name: "IT",
        location: "Paris Office",
        budget: 500000
    },
    skills: [
        { name: "JavaScript", level: "expert", years: 5 },
        { name: "MongoDB", level: "advanced", years: 3 },
        { name: "Python", level: "intermediate", years: 2 }
    ],
    contact: {
        email: "alice@company.com",
        phone: "+33612345678",
        emergencyContact: {
            name: "Bob Dupont",
            relation: "spouse",
            phone: "+33623456789"
        }
    }
}

// Requêtes
// Employés du département IT à Paris
db.employees.find({
    "department.name": "IT",
    "department.location": "Paris Office"
})

// Employés avec compétence MongoDB niveau expert
db.employees.find({
    skills: {
        $elemMatch: {
            name: "MongoDB",
            level: "expert"
        }
    }
})

// Employés avec au moins une compétence expert et 5+ ans d'expérience
db.employees.find({
    skills: {
        $elemMatch: {
            level: "expert",
            years: { $gte: 5 }
        }
    }
})

// Employés sans contact d'urgence
db.employees.find({
    "contact.emergencyContact": { $exists: false }
})
```

### Cas 3 : Réseaux Sociaux - Profils Utilisateurs

```javascript
// Structure
{
    username: "alice_dev",
    profile: {
        fullName: "Alice Dupont",
        bio: "Full-stack developer",
        location: {
            city: "Paris",
            country: "France",
            coordinates: {
                lat: 48.8566,
                lng: 2.3522
            }
        },
        website: "https://alice.dev"
    },
    settings: {
        privacy: {
            profileVisible: true,
            showEmail: false,
            showLocation: true
        },
        notifications: {
            email: true,
            push: true,
            sms: false
        }
    },
    posts: [
        {
            postId: "POST-001",
            content: "Learning MongoDB!",
            date: ISODate("2024-01-15"),
            likes: 42,
            tags: ["mongodb", "database"]
        }
    ]
}

// Requêtes
// Utilisateurs à Paris avec profil visible
db.users.find({
    "profile.location.city": "Paris",
    "settings.privacy.profileVisible": true
})

// Utilisateurs avec notifications email activées
db.users.find({
    "settings.notifications.email": true
})

// Utilisateurs avec au moins un post populaire (50+ likes)
db.users.find({
    posts: {
        $elemMatch: {
            likes: { $gte: 50 }
        }
    }
})

// Utilisateurs avec posts récents tagués "mongodb"
db.users.find({
    posts: {
        $elemMatch: {
            date: { $gte: ISODate("2024-01-01") },
            tags: "mongodb"
        }
    }
})
```

---

## Mise à Jour de Documents Imbriqués

Vous pouvez également mettre à jour des champs imbriqués avec la notation pointée.

### Mise à Jour de Champs Simples

```javascript
// Mettre à jour la ville
db.users.updateOne(
    { email: "alice@example.com" },
    { $set: { "address.city": "Lyon" } }
)

// Mettre à jour les coordonnées
db.users.updateOne(
    { email: "alice@example.com" },
    {
        $set: {
            "address.coordinates.latitude": 45.7640,
            "address.coordinates.longitude": 4.8357
        }
    }
)

// Mettre à jour les préférences
db.users.updateOne(
    { email: "alice@example.com" },
    { $set: { "preferences.newsletter": false } }
)
```

### Mise à Jour d'Objets Complets

```javascript
// Remplacer tout l'objet address
db.users.updateOne(
    { email: "alice@example.com" },
    {
        $set: {
            address: {
                street: "456 New St",
                city: "Lyon",
                zipCode: "69001",
                country: "France"
            }
        }
    }
)
```

### Mise à Jour dans Tableaux d'Objets

```javascript
// Mettre à jour le statut d'une commande spécifique
db.users.updateOne(
    {
        email: "alice@example.com",
        "orders.orderId": "ORD-001"
    },
    {
        $set: { "orders.$.status": "completed" }
    }
)
// L'opérateur $ positionnel met à jour le premier élément correspondant

// Ajouter une nouvelle commande
db.users.updateOne(
    { email: "alice@example.com" },
    {
        $push: {
            orders: {
                orderId: "ORD-003",
                date: new Date(),
                amount: 200.00,
                status: "pending"
            }
        }
    }
)
```

---

## Comparaison avec SQL

Dans SQL, les données imbriquées nécessiteraient plusieurs tables et jointures :

### Approche SQL (3 tables)

```sql
-- Table users
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

-- Table addresses
CREATE TABLE addresses (
    id INT PRIMARY KEY,
    user_id INT,
    street VARCHAR(100),
    city VARCHAR(50),
    zipCode VARCHAR(10),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Table orders
CREATE TABLE orders (
    id INT PRIMARY KEY,
    user_id INT,
    order_id VARCHAR(20),
    amount DECIMAL(10,2),
    status VARCHAR(20),
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Requête avec jointures
SELECT u.name, a.city, o.amount
FROM users u
JOIN addresses a ON u.id = a.user_id
JOIN orders o ON u.id = o.user_id
WHERE a.city = 'Paris'
  AND o.status = 'completed'
  AND o.amount >= 100;
```

### Approche MongoDB (1 collection)

```javascript
// Tout dans un seul document
db.users.find({
    "address.city": "Paris",
    orders: {
        $elemMatch: {
            status: "completed",
            amount: { $gte: 100 }
        }
    }
},
{
    name: 1,
    "address.city": 1,
    "orders.amount": 1
})
```

**Avantage MongoDB** : Pas de jointures, lecture en une seule opération.

---

## Bonnes Pratiques

### 1. Utiliser la Notation Pointée pour les Requêtes

```javascript
// ✅ Bon : notation pointée (flexible)
db.users.find({
    "address.city": "Paris",
    "address.country": "France"
})

// ❌ Éviter : objet complet (rigide)
db.users.find({
    address: {
        street: "...",
        city: "Paris",
        zipCode: "...",
        country: "France"
    }
})
```

### 2. Utiliser `$elemMatch` pour les Tableaux d'Objets

```javascript
// ✅ Bon : garantit le même élément
db.users.find({
    orders: {
        $elemMatch: {
            status: "completed",
            amount: { $gte: 100 }
        }
    }
})

// ⚠️ Peut donner des résultats inattendus
db.users.find({
    "orders.status": "completed",
    "orders.amount": { $gte: 100 }
})
```

### 3. Créer des Index sur Champs Imbriqués Fréquents

```javascript
// Index sur champs imbriqués souvent interrogés
db.users.createIndex({ "address.city": 1 })
db.users.createIndex({ "address.country": 1, "address.city": 1 })
db.users.createIndex({ "orders.status": 1 })
```

### 4. Limiter la Profondeur d'Imbrication

```javascript
// ✅ Raisonnable : 2-3 niveaux
{
    user: {
        profile: {
            name: "Alice"
        }
    }
}

// ⚠️ Trop profond : difficile à maintenir
{
    level1: {
        level2: {
            level3: {
                level4: {
                    level5: {
                        data: "..."
                    }
                }
            }
        }
    }
}
```

### 5. Documenter la Structure des Documents

```javascript
// Ajouter des commentaires ou documentation pour les structures complexes
/*
Structure du document User:
{
    name: string,
    email: string,
    address: {
        street: string,
        city: string,
        zipCode: string,
        country: string,
        coordinates: {
            latitude: number,
            longitude: number
        }
    },
    orders: [{
        orderId: string,
        date: Date,
        amount: number,
        status: string
    }]
}
*/
```

### 6. Considérer les Performances

```javascript
// Pour de très grands tableaux imbriqués, envisager une collection séparée
// ❌ Peut devenir problématique
{
    userId: 123,
    orders: [/* 10,000 commandes */]
}

// ✅ Meilleur pour grandes quantités
// Collection users
{ userId: 123, name: "Alice", ... }

// Collection orders
{ userId: 123, orderId: "ORD-001", ... }
```

---

## Pièges Courants à Éviter

### 1. Oublier les Guillemets en Notation Pointée

```javascript
// ❌ Erreur : syntaxe invalide
db.users.find({ address.city: "Paris" })

// ✅ Correct : avec guillemets
db.users.find({ "address.city": "Paris" })
```

### 2. Confusion avec `$elemMatch`

```javascript
// ❌ Conditions sur différents éléments
db.users.find({
    "orders.amount": { $gte: 100 },
    "orders.status": "completed"
})

// ✅ Conditions sur le même élément
db.users.find({
    orders: {
        $elemMatch: {
            amount: { $gte: 100 },
            status: "completed"
        }
    }
})
```

### 3. Correspondance d'Objet Exact Trop Stricte

```javascript
// ❌ Très fragile : ordre et tous les champs doivent correspondre
db.users.find({
    address: { city: "Paris", country: "France" }
})

// ✅ Flexible : notation pointée
db.users.find({
    "address.city": "Paris",
    "address.country": "France"
})
```

### 4. Performance avec Tableaux Volumineux

```javascript
// ⚠️ Peut être lent si orders contient des milliers d'éléments
db.users.find({
    orders: {
        $elemMatch: { status: "completed" }
    }
})

// ✅ Envisager une collection séparée pour de grandes quantités
```

### 5. Projections Incomplètes

```javascript
// ⚠️ Projette tout le sous-objet
db.users.find({}, { address: 1 })

// ✅ Spécifique : seulement les champs nécessaires
db.users.find({}, {
    "address.city": 1,
    "address.country": 1
})
```

---

## Performance et Optimisation

### Indexation de Champs Imbriqués

```javascript
// Créer des index sur champs fréquemment interrogés
db.users.createIndex({ "address.city": 1 })
db.users.createIndex({ "address.country": 1, "address.city": 1 })

// Index sur tableaux d'objets
db.users.createIndex({ "orders.status": 1 })
db.users.createIndex({ "orders.date": -1 })
```

### Vérification avec `explain()`

```javascript
// Analyser une requête sur champs imbriqués
db.users.find({
    "address.city": "Paris",
    "address.country": "France"
}).explain("executionStats")

// Vérifier si l'index est utilisé (IXSCAN vs COLLSCAN)
```

### Optimisation des Projections

```javascript
// ⚠️ Retourne tout le document
db.users.find({ "address.city": "Paris" })

// ✅ Ne retourne que les champs nécessaires
db.users.find(
    { "address.city": "Paris" },
    {
        name: 1,
        "address.city": 1,
        "address.zipCode": 1,
        _id: 0
    }
)
```

---

## Points Clés à Retenir

✅ Utilisez la **notation pointée** (`"champ.souschamp"`) pour accéder aux champs imbriqués

✅ Les chemins en notation pointée doivent être entre **guillemets**

✅ `$elemMatch` garantit que les conditions s'appliquent au **même élément** d'un tableau

✅ La **correspondance d'objet exact** nécessite ordre et champs identiques (éviter)

✅ Préférez la **notation pointée** pour plus de flexibilité

✅ Créez des **index** sur les champs imbriqués fréquemment interrogés

✅ Limitez la **profondeur d'imbrication** pour la maintenabilité

✅ Les **projections** fonctionnent aussi sur les champs imbriqués

✅ Pour de très grands tableaux, envisagez une **collection séparée**

✅ Utilisez `explain()` pour vérifier l'**utilisation des index**

---

## Résumé

Dans ce chapitre, vous avez appris :

- Comment utiliser la notation pointée pour accéder aux champs imbriqués
- La différence entre recherche par champs individuels et objets complets
- Comment interroger des tableaux d'objets avec `$elemMatch`
- Les requêtes sur structures profondément imbriquées
- Comment projeter des champs imbriqués spécifiques
- Les cas d'usage pratiques dans différents domaines
- Comment mettre à jour des documents imbriqués
- La comparaison avec l'approche SQL
- Les bonnes pratiques et pièges à éviter
- L'optimisation des performances avec les index

La maîtrise des requêtes sur documents imbriqués est essentielle pour exploiter pleinement la flexibilité de MongoDB. Cette capacité à stocker et interroger des structures complexes directement dans les documents est l'un des grands avantages de MongoDB par rapport aux bases de données relationnelles.

Dans le prochain chapitre, nous explorerons les **requêtes sur tableaux** pour approfondir davantage le travail avec les structures de données complexes.

---


⏭️ [Requêtes sur tableaux](/03-requetes-et-filtres/11-requetes-tableaux.md)
