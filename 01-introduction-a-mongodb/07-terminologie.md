🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.7 Terminologie : Documents, Collections, Bases de données

## Introduction

Avant de commencer à utiliser MongoDB, il est essentiel de maîtriser sa terminologie. MongoDB utilise des concepts différents des bases de données relationnelles traditionnelles. Cette section vous présente les termes fondamentaux que vous rencontrerez tout au long de votre apprentissage.

Nous allons explorer la hiérarchie des données dans MongoDB, du niveau le plus bas (les champs) jusqu'au niveau le plus haut (l'instance serveur).

---

## Vue d'ensemble de la hiérarchie

MongoDB organise les données selon une structure hiérarchique à plusieurs niveaux :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Hiérarchie des données MongoDB                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Instance MongoDB (serveur mongod)                                 │
│   │                                                                 │
│   └─── Base de données (Database)                                   │
│        │                                                            │
│        └─── Collection                                              │
│             │                                                       │
│             └─── Document                                           │
│                  │                                                  │
│                  └─── Champ (Field)                                 │
│                       │                                             │
│                       └─── Valeur (Value)                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Correspondance avec le monde SQL

Si vous venez du monde des bases de données relationnelles, voici les équivalences :

| SQL (Relationnel) | MongoDB | Description |
|-------------------|---------|-------------|
| Database | Database | Conteneur de niveau supérieur |
| Table | Collection | Groupe d'enregistrements |
| Row (ligne) | Document | Un enregistrement individuel |
| Column (colonne) | Field (champ) | Un attribut de l'enregistrement |
| Primary Key | `_id` | Identifiant unique |
| Index | Index | Structure d'optimisation |
| JOIN | `$lookup` / Embedded | Association de données |
| Foreign Key | Reference / DBRef | Lien entre enregistrements |

---

## Le Document

### Définition

Le **document** est l'unité fondamentale de données dans MongoDB. C'est l'équivalent d'une ligne (row) dans une table SQL, mais avec beaucoup plus de flexibilité.

Un document est une structure de données composée de paires **clé-valeur**, similaire à un objet JSON.

### Format : BSON

MongoDB stocke les documents au format **BSON** (Binary JSON). BSON est une représentation binaire de JSON qui offre :

- Un encodage et décodage plus rapides
- Des types de données supplémentaires (Date, ObjectId, Binary, etc.)
- Une traversée efficace des documents

```
┌─────────────────────────────────────────────────────────────────────┐
│                        JSON vs BSON                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   JSON (ce que vous écrivez)         BSON (ce que MongoDB stocke)   │
│                                                                     │
│   {                                  ┌─────────────────────────┐    │
│     "nom": "Alice",        ───►      │ 0x16 0x00 0x00 0x00     │    │
│     "age": 25                        │ 0x02 "nom" 0x00 0x06    │    │
│   }                                  │ "Alice" 0x00            │    │
│                                      │ 0x10 "age" 0x00         │    │
│                                      │ 0x19 0x00 0x00 0x00     │    │
│                                      │ 0x00                    │    │
│                                      └─────────────────────────┘    │
│                                      (représentation binaire)       │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Structure d'un document

Un document MongoDB se compose de :

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "champ1": "valeur1",
  "champ2": 123,
  "champ3": {
    "sous_champ": "valeur imbriquée"
  },
  "champ4": ["élément1", "élément2", "élément3"]
}
```

### Le champ `_id`

Chaque document possède obligatoirement un champ **`_id`** qui sert d'identifiant unique :

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Le champ _id                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   • Obligatoire dans chaque document                                │
│   • Unique au sein de la collection                                 │
│   • Immutable (ne peut pas être modifié)                            │
│   • Créé automatiquement si non fourni                              │
│   • Par défaut : ObjectId (12 octets)                               │
│                                                                     │
│   Structure d'un ObjectId :                                         │
│                                                                     │
│   507f1f77 bcf86c d799 439011                                       │
│   ────────  ────── ──── ──────                                      │
│      │        │     │     │                                         │
│      │        │     │     └── Compteur (3 octets)                   │
│      │        │     └── ID processus (2 octets)                     │
│      │        └── Identifiant machine (3 octets)                    │
│      └── Timestamp Unix (4 octets)                                  │
│                                                                     │
│   Avantages de l'ObjectId :                                         │
│   • Génération distribuée sans collision                            │
│   • Triable chronologiquement                                       │
│   • Contient la date de création                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Exemples de documents

#### Document simple

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "prenom": "Marie",
  "nom": "Dupont",
  "email": "marie.dupont@email.com",
  "age": 28
}
```

#### Document avec données imbriquées

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439012"),
  "prenom": "Jean",
  "nom": "Martin",
  "adresse": {
    "rue": "15 avenue des Champs-Élysées",
    "ville": "Paris",
    "codePostal": "75008",
    "pays": "France",
    "coordonnees": {
      "latitude": 48.8698,
      "longitude": 2.3075
    }
  },
  "telephones": {
    "domicile": "01 23 45 67 89",
    "mobile": "06 12 34 56 78"
  }
}
```

#### Document avec tableaux

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439013"),
  "titre": "Les Misérables",
  "auteur": "Victor Hugo",
  "annee": 1862,
  "genres": ["Roman", "Drame", "Historique"],
  "personnages": [
    { "nom": "Jean Valjean", "role": "Protagoniste" },
    { "nom": "Javert", "role": "Antagoniste" },
    { "nom": "Cosette", "role": "Personnage principal" }
  ],
  "editions": [
    { "editeur": "Gallimard", "annee": 1951, "pages": 1900 },
    { "editeur": "Le Livre de Poche", "annee": 1998, "pages": 2016 }
  ]
}
```

### Caractéristiques des documents

| Caractéristique | Description |
|-----------------|-------------|
| **Taille maximale** | 16 Mo par document |
| **Profondeur maximale** | 100 niveaux d'imbrication |
| **Schéma flexible** | Chaque document peut avoir une structure différente |
| **Types variés** | Peut contenir des valeurs de différents types |
| **Auto-contenu** | Peut inclure toutes les données liées |

### Types de données BSON

MongoDB supporte de nombreux types de données :

| Type | Description | Exemple |
|------|-------------|---------|
| **String** | Chaîne de caractères UTF-8 | `"Bonjour"` |
| **Integer** | Nombre entier (32 ou 64 bits) | `42`, `NumberLong(123456789)` |
| **Double** | Nombre à virgule flottante | `3.14159` |
| **Boolean** | Valeur booléenne | `true`, `false` |
| **ObjectId** | Identifiant unique 12 octets | `ObjectId("507f1f...")` |
| **Date** | Date/heure UTC | `ISODate("2024-11-28T14:30:00Z")` |
| **Array** | Tableau de valeurs | `[1, 2, 3]` |
| **Object** | Document imbriqué | `{ "a": 1, "b": 2 }` |
| **Null** | Valeur nulle | `null` |
| **Binary** | Données binaires | `BinData(0, "...")` |
| **Decimal128** | Décimal haute précision | `NumberDecimal("19.99")` |
| **Timestamp** | Horodatage interne MongoDB | `Timestamp(1234567890, 1)` |
| **RegExp** | Expression régulière | `/pattern/i` |

---

## La Collection

### Définition

Une **collection** est un groupe de documents MongoDB. C'est l'équivalent d'une table dans une base de données relationnelle, mais sans schéma fixe.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Collection                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Collection "utilisateurs"                                         │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                                                             │   │
│   │   ┌─────────────────────────────────────────────────────┐   │   │
│   │   │ Document 1                                          │   │   │
│   │   │ { "_id": ..., "nom": "Alice", "age": 25 }           │   │   │
│   │   └─────────────────────────────────────────────────────┘   │   │
│   │                                                             │   │
│   │   ┌─────────────────────────────────────────────────────┐   │   │
│   │   │ Document 2                                          │   │   │
│   │   │ { "_id": ..., "nom": "Bob", "email": "bob@..." }    │   │   │
│   │   └─────────────────────────────────────────────────────┘   │   │
│   │                                                             │   │
│   │   ┌─────────────────────────────────────────────────────┐   │   │
│   │   │ Document 3                                          │   │   │
│   │   │ { "_id": ..., "nom": "Charlie", "age": 35,          │   │   │
│   │   │   "adresse": { "ville": "Paris" } }                 │   │   │
│   │   └─────────────────────────────────────────────────────┘   │   │
│   │                                                             │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   Note : Chaque document peut avoir une structure différente !      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Création d'une collection

Les collections sont créées **automatiquement** lors de la première insertion de document :

```javascript
// La collection "produits" est créée automatiquement
db.produits.insertOne({ nom: "Laptop", prix: 999 })
```

Vous pouvez aussi créer une collection explicitement :

```javascript
// Création explicite avec options
db.createCollection("logs", {
  capped: true,           // Collection à taille fixe
  size: 10485760,         // Taille maximale en octets (10 Mo)
  max: 10000              // Nombre maximum de documents
})
```

### Nommage des collections

Les noms de collections doivent respecter certaines règles :

```
┌─────────────────────────────────────────────────────────────────────┐
│                  Règles de nommage des collections                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ✅ Autorisé :                                                     │
│   • Lettres, chiffres, underscores                                  │
│   • Longueur de 1 à 120 caractères                                  │
│   • Sensible à la casse (Users ≠ users)                             │
│                                                                     │
│   ❌ Interdit :                                                     │
│   • Commencer par "system." (réservé)                               │
│   • Contenir le caractère "$"                                       │
│   • Chaîne vide ""                                                  │
│   • Contenir le caractère nul (\0)                                  │
│                                                                     │
│   Exemples valides :                                                │
│   • users                                                           │
│   • user_profiles                                                   │
│   • orders2024                                                      │
│   • LOG_entries                                                     │
│                                                                     │
│   Conventions recommandées :                                        │
│   • Utiliser le pluriel (users, products, orders)                   │
│   • snake_case ou camelCase (être consistant)                       │
│   • Noms descriptifs et courts                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Types de collections

MongoDB propose plusieurs types de collections :

#### 1. Collection standard

La collection par défaut, sans limitation particulière.

```javascript
db.createCollection("articles")
```

#### 2. Collection cappée (Capped Collection)

Collection à taille fixe qui fonctionne comme un buffer circulaire :

```javascript
db.createCollection("logs", {
  capped: true,
  size: 5242880,    // 5 Mo
  max: 5000         // Maximum 5000 documents
})
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Capped Collection                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Comportement circulaire :                                         │
│                                                                     │
│   État initial (3 documents, max: 5)                                │
│   ┌─────┬─────┬─────┬─────┬─────┐                                   │
│   │ D1  │ D2  │ D3  │     │     │                                   │
│   └─────┴─────┴─────┴─────┴─────┘                                   │
│                                                                     │
│   Après 2 insertions                                                │
│   ┌─────┬─────┬─────┬─────┬─────┐                                   │
│   │ D1  │ D2  │ D3  │ D4  │ D5  │                                   │
│   └─────┴─────┴─────┴─────┴─────┘                                   │
│                                                                     │
│   Après 1 insertion supplémentaire (D1 est supprimé)                │
│   ┌─────┬─────┬─────┬─────┬─────┐                                   │
│   │ D2  │ D3  │ D4  │ D5  │ D6  │  ← D6 remplace D1                 │
│   └─────┴─────┴─────┴─────┴─────┘                                   │
│                                                                     │
│   Cas d'usage : logs, métriques, cache                              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3. Collection Time Series (MongoDB 5.0+)

Optimisée pour les données temporelles :

```javascript
db.createCollection("metriques", {
  timeseries: {
    timeField: "timestamp",       // Champ contenant la date
    metaField: "capteur_id",      // Champ de métadonnées (optionnel)
    granularity: "seconds"        // Granularité des données
  }
})
```

#### 4. Collection clusterisée (MongoDB 5.3+)

Stocke les documents ordonnés par clé de cluster :

```javascript
db.createCollection("events", {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true
  }
})
```

### Collections système

MongoDB crée automatiquement des collections système :

| Collection | Base | Description |
|------------|------|-------------|
| `system.users` | admin | Utilisateurs de la base |
| `system.roles` | admin | Rôles personnalisés |
| `system.indexes` | chaque base | Métadonnées des index |
| `system.profile` | chaque base | Données de profilage |
| `oplog.rs` | local | Journal de réplication |

---

## La Base de données (Database)

### Définition

Une **base de données** (database) est un conteneur de collections. Une instance MongoDB peut héberger plusieurs bases de données, chacune avec ses propres collections, index et permissions.

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Instance MongoDB                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │ Base de données : "ecommerce"                             │     │
│   │   ├── Collection : "products"                             │     │
│   │   ├── Collection : "users"                                │     │
│   │   ├── Collection : "orders"                               │     │
│   │   └── Collection : "reviews"                              │     │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                     │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │ Base de données : "blog"                                  │     │
│   │   ├── Collection : "posts"                                │     │
│   │   ├── Collection : "authors"                              │     │
│   │   └── Collection : "comments"                             │     │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                     │
│   ┌───────────────────────────────────────────────────────────┐     │
│   │ Base de données : "analytics"                             │     │
│   │   ├── Collection : "events"                               │     │
│   │   ├── Collection : "sessions"                             │     │
│   │   └── Collection : "metrics"                              │     │
│   └───────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Création d'une base de données

Comme pour les collections, les bases de données sont créées **automatiquement** lors de la première opération :

```javascript
// Bascule vers la base "maNouvelleBd" (créée si inexistante)
use maNouvelleBd

// La base n'existe réellement qu'après une première insertion
db.maCollection.insertOne({ message: "Hello MongoDB!" })
```

### Nommage des bases de données

```
┌─────────────────────────────────────────────────────────────────────┐
│               Règles de nommage des bases de données                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ✅ Autorisé :                                                     │
│   • Lettres minuscules (recommandé)                                 │
│   • Chiffres                                                        │
│   • Longueur de 1 à 64 caractères                                   │
│                                                                     │
│   ❌ Interdit :                                                     │
│   • Espaces                                                         │
│   • Caractères : / \ . " $ * < > : | ?                              │
│   • Chaîne vide                                                     │
│   • Noms réservés : admin, local, config                            │
│                                                                     │
│   ⚠️ Attention :                                                    │
│   • Sensible à la casse sur certains systèmes                       │
│   • Éviter les majuscules pour la portabilité                       │
│                                                                     │
│   Exemples valides :                                                │
│   • myapp                                                           │
│   • ecommerce_prod                                                  │
│   • analytics2024                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Bases de données système

MongoDB utilise trois bases de données réservées :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Bases de données système                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ admin                                                       │   │
│   │                                                             │   │
│   │ • Base d'administration centrale                            │   │
│   │ • Stocke les utilisateurs et rôles globaux                  │   │
│   │ • Commandes administratives (shutdown, etc.)                │   │
│   │ • Authentification des utilisateurs admin                   │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ config                                                      │   │
│   │                                                             │   │
│   │ • Utilisée par les clusters shardés                         │   │
│   │ • Métadonnées sur les shards et chunks                      │   │
│   │ • Configuration du routage                                  │   │
│   │ • Ne pas modifier manuellement !                            │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ local                                                       │   │
│   │                                                             │   │
│   │ • Données locales à chaque instance                         │   │
│   │ • oplog.rs pour la réplication                              │   │
│   │ • N'est jamais répliquée                                    │   │
│   │ • Informations spécifiques au serveur                       │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Commandes de base

```javascript
// Afficher la base de données courante
db

// Lister toutes les bases de données
show dbs

// Basculer vers une base (ou la créer)
use maBase

// Afficher les collections de la base courante
show collections

// Supprimer la base de données courante
db.dropDatabase()

// Statistiques de la base
db.stats()
```

---

## Le Namespace

### Définition

Le **namespace** est l'identifiant complet d'une collection, combinant le nom de la base de données et le nom de la collection :

```
namespace = <database>.<collection>
```

### Exemples

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Namespaces                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Base de données    Collection         Namespace                   │
│   ───────────────    ──────────         ─────────────────────────   │
│   ecommerce          products           ecommerce.products          │
│   ecommerce          users              ecommerce.users             │
│   blog               posts              blog.posts                  │
│   analytics          events             analytics.events            │
│                                                                     │
│   Utilisation dans les requêtes :                                   │
│                                                                     │
│   // Requête standard                                               │
│   use ecommerce                                                     │
│   db.products.find()                                                │
│                                                                     │
│   // Équivalent avec namespace complet                              │
│   db.getSiblingDB("ecommerce").getCollection("products").find()     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

Le namespace est particulièrement visible dans :
- Les messages d'erreur
- Les logs
- L'oplog de réplication
- Les résultats de `explain()`

---

## Les Index

### Définition

Un **index** est une structure de données qui améliore la vitesse des opérations de recherche. Sans index, MongoDB doit scanner tous les documents d'une collection (collection scan).

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Recherche avec et sans index                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Sans index (Collection Scan) :                                    │
│                                                                     │
│   Recherche : { email: "alice@example.com" }                        │
│                                                                     │
│   ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐           │
│   │ D1 │→│ D2 │→│ D3 │→│ D4 │→│ D5 │→│ D6 │→│ D7 │→│ D8 │           │
│   └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘           │
│     ↓      ↓      ↓      ↓      ↓      ↓      ↓      ↓              │
│   Non    Non    Non    Non     ✓     Non    Non    Non              │
│                                                                     │
│   → Tous les documents sont examinés : O(n)                         │
│                                                                     │
│   ─────────────────────────────────────────────────────────────     │
│                                                                     │
│   Avec index sur "email" :                                          │
│                                                                     │
│                    Index B-tree                                     │
│                        │                                            │
│              ┌─────────┼─────────┐                                  │
│              ▼         ▼         ▼                                  │
│         ┌───────┐ ┌───────┐ ┌───────┐                               │
│         │ A-F   │ │ G-M   │ │ N-Z   │                               │
│         └───┬───┘ └───────┘ └───────┘                               │
│             │                                                       │
│             ▼                                                       │
│         ┌───────┐                                                   │
│         │alice@ │ → Document D5                                     │
│         └───────┘                                                   │
│                                                                     │
│   → Accès direct via l'index : O(log n)                             │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### L'index `_id`

Chaque collection possède automatiquement un index sur le champ `_id` :

```javascript
// Cet index existe toujours
{ "_id": 1 }
```

### Types d'index courants

| Type | Description | Exemple |
|------|-------------|---------|
| **Simple** | Un seul champ | `{ email: 1 }` |
| **Composé** | Plusieurs champs | `{ nom: 1, prenom: 1 }` |
| **Multiclé** | Champ tableau | `{ tags: 1 }` |
| **Texte** | Recherche full-text | `{ contenu: "text" }` |
| **Géospatial** | Coordonnées | `{ location: "2dsphere" }` |
| **TTL** | Expiration automatique | `{ createdAt: 1 }, { expireAfterSeconds: 3600 }` |

---

## Les Champs (Fields)

### Définition

Un **champ** (field) est une paire clé-valeur dans un document. C'est l'équivalent d'une colonne dans une table SQL.

### Nommage des champs

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Règles de nommage des champs                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ✅ Autorisé :                                                     │
│   • Peut contenir n'importe quel caractère UTF-8                    │
│   • Sensible à la casse (Name ≠ name)                               │
│                                                                     │
│   ❌ Restrictions :                                                 │
│   • Ne peut pas commencer par "$" (réservé aux opérateurs)          │
│   • Ne peut pas contenir "." (utilisé pour l'accès imbriqué)        │
│   • "_id" est réservé comme clé primaire                            │
│   • Ne peut pas contenir le caractère nul                           │
│                                                                     │
│   Conventions recommandées :                                        │
│   • camelCase : firstName, lastName, createdAt                      │
│   • ou snake_case : first_name, last_name, created_at               │
│   • Être consistant dans tout le projet                             │
│   • Noms courts mais descriptifs                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Accès aux champs imbriqués

La notation **pointée** (dot notation) permet d'accéder aux champs imbriqués :

```javascript
// Document
{
  "_id": 1,
  "personne": {
    "nom": "Dupont",
    "adresse": {
      "ville": "Paris",
      "codePostal": "75001"
    }
  },
  "scores": [85, 90, 78]
}

// Accès aux champs imbriqués
db.collection.find({ "personne.nom": "Dupont" })
db.collection.find({ "personne.adresse.ville": "Paris" })

// Accès aux éléments de tableau par index
db.collection.find({ "scores.0": 85 })  // Premier élément
db.collection.find({ "scores.2": 78 })  // Troisième élément
```

---

## Schéma comparatif complet

```
┌─────────────────────────────────────────────────────────────────────┐
│                 SQL vs MongoDB : Vue complète                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Base de données SQL              Base de données MongoDB          │
│                                                                     │
│   ┌─────────────────────┐         ┌──────────────────────┐          │
│   │ Database            │         │ Database             │          │
│   │ ┌─────────────────┐ │         │ ┌──────────────────┐ │          │
│   │ │ Table "users"   │ │         │ │Collection "users"│ │          │
│   │ │ ┌─────────────┐ │ │         │ │ ┌─────────────┐  │ │          │
│   │ │ │id│name│email│ │ │         │ │ │  Document   │  │ │          │
│   │ │ ├──┼────┼─────┤ │ │         │ │ │{            │  │ │          │
│   │ │ │1 │Ali │a@...│ │ │   ═══   │ │ │ _id: 1,     │  │ │          │
│   │ │ │2 │Bob │b@...│ │ │         │ │ │ name:"Ali", │  │ │          │
│   │ │ └──┴────┴─────┘ │ │         │ │ │ email:"a@"  │  │ │          │
│   │ │    Row (ligne)  │ │         │ │ │}            │  │ │          │
│   │ │   Column (col.) │ │         │ │ └─────────────┘  │ │          │
│   │ └─────────────────┘ │         │ │    Document      │ │          │
│   │                     │         │ │    Field         │ │          │
│   └─────────────────────┘         │ └──────────────────┘ │          │
│                                   └──────────────────────┘          │
│                                                                     │
│   Schéma fixe                     Schéma flexible                   │
│   Jointures natives               Documents imbriqués               │
│   SQL                             API orientée objet                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Glossaire rapide

| Terme | Définition |
|-------|------------|
| **Document** | Unité de données (équivalent d'une ligne SQL) |
| **Collection** | Groupe de documents (équivalent d'une table SQL) |
| **Database** | Conteneur de collections |
| **Field** | Paire clé-valeur dans un document |
| **`_id`** | Identifiant unique obligatoire |
| **ObjectId** | Type par défaut pour `_id` (12 octets) |
| **BSON** | Format binaire de stockage |
| **Namespace** | Identifiant complet : database.collection |
| **Index** | Structure d'optimisation des requêtes |
| **Embedded document** | Document imbriqué dans un autre |
| **Array** | Tableau de valeurs dans un document |
| **Dot notation** | Notation pointée pour accéder aux champs imbriqués |

---

## Conclusion

La terminologie MongoDB est différente de celle des bases SQL, mais les concepts se correspondent. L'essentiel à retenir :

- Les **documents** remplacent les lignes et offrent plus de flexibilité
- Les **collections** remplacent les tables mais sans schéma fixe
- Les **bases de données** fonctionnent de manière similaire
- Le champ **`_id`** est obligatoire et sert de clé primaire
- La **notation pointée** permet d'accéder aux données imbriquées

Cette terminologie vous accompagnera tout au long de votre apprentissage de MongoDB. Dans les prochaines sections, nous passerons à la pratique avec l'installation de MongoDB.

---

## Points clés à retenir

- **Document** = objet JSON/BSON avec des paires clé-valeur
- **Collection** = groupe de documents (sans schéma fixe)
- **Database** = conteneur de collections
- **`_id`** = identifiant unique obligatoire (ObjectId par défaut)
- **BSON** = format binaire optimisé basé sur JSON
- **Namespace** = `database.collection` (ex: `ecommerce.products`)
- Les documents peuvent contenir des **objets imbriqués** et des **tableaux**
- La **notation pointée** (`champ.souschamp`) accède aux données imbriquées
- Taille maximale d'un document : **16 Mo**

---


⏭️ [Installation de MongoDB (Windows, Linux, macOS)](/01-introduction-a-mongodb/08-installation-mongodb.md)
