🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.4 Création et Gestion des Collections

## Introduction

Après avoir appris à créer des bases de données, découvrons maintenant les **collections** ! Si vous venez du monde SQL, pensez aux collections comme l'équivalent des **tables**, mais avec beaucoup plus de flexibilité.

> **💡 Définition :** Une collection est un regroupement de documents MongoDB. C'est l'unité d'organisation principale dans une base de données MongoDB.

Dans cette section, nous allons explorer :
- La création de collections
- Les différents types de collections
- Les options de configuration
- La gestion et la maintenance
- Les bonnes pratiques

---

## Qu'est-ce qu'une Collection ?

### Concept de Base

Une **collection** dans MongoDB est similaire à une **table** dans une base relationnelle, mais avec des différences importantes :

**SQL (Table) :**
```sql
-- Structure rigide, schéma fixe
CREATE TABLE utilisateurs (
    id INT PRIMARY KEY,
    nom VARCHAR(50),
    email VARCHAR(100)
);
```

**MongoDB (Collection) :**
```javascript
// Structure flexible, pas de schéma imposé
db.utilisateurs.insertOne({
  nom: "Dupont",
  email: "dupont@example.com",
  age: 30
})

db.utilisateurs.insertOne({
  nom: "Martin",
  email: "martin@example.com",
  ville: "Paris",
  competences: ["JavaScript", "MongoDB"]
})
// Deux documents avec des structures différentes ! ✅
```

### Caractéristiques des Collections

| Caractéristique | Description |
|----------------|-------------|
| **Schéma flexible** | Les documents peuvent avoir des structures différentes |
| **Documents JSON/BSON** | Stockage en format document |
| **Index** | Peuvent être indexées pour optimiser les performances |
| **Validation optionnelle** | Possibilité de définir des règles de validation |
| **Pas de jointures natives** | Pensé pour l'imbrication et la dénormalisation |

---

## Création Implicite de Collections

### Le Principe

Comme pour les bases de données, MongoDB utilise la **création implicite** (lazy creation) pour les collections.

**Principe :** Une collection est créée automatiquement dès que vous y insérez un document.

```javascript
// Sélectionner la base
use blog

// Insérer un document - la collection "articles" est créée automatiquement !
db.articles.insertOne({
  titre: "Introduction à MongoDB",
  auteur: "Jean"
})

// Vérifier que la collection existe
show collections
// Sortie : articles
```

**C'est aussi simple que ça ! 🎉**

### Exemple Complet

```javascript
// Base de données e-commerce
use boutique

// Créer plusieurs collections par insertion
db.produits.insertOne({
  nom: "Laptop",
  prix: 999.99
})

db.clients.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})

db.commandes.insertOne({
  clientId: ObjectId("..."),
  total: 999.99
})

// Vérifier les collections créées
show collections
```

**Sortie :**
```
clients
commandes
produits
```

---

## Création Explicite de Collections

### Pourquoi Créer Explicitement ?

Bien que la création implicite soit pratique, il existe des cas où vous voulez créer explicitement une collection :

1. **Définir des options spécifiques** (validation, capped, etc.)
2. **Préparer la structure** avant l'insertion
3. **Configurer des paramètres avancés**
4. **Documenter l'architecture**

### Commande `createCollection()`

**Syntaxe de base :**
```javascript
db.createCollection("nom_collection")
```

**Exemple simple :**
```javascript
use mon_projet

// Créer une collection vide
db.createCollection("utilisateurs")

// Vérifier
show collections
// Sortie : utilisateurs
```

**Résultat :**
```javascript
{ ok: 1 }
```

### Création avec Options

```javascript
db.createCollection("nom_collection", {
  // Options de configuration
  capped: false,
  size: 5242880,
  max: 5000,
  validator: {},
  validationLevel: "strict",
  validationAction: "error"
})
```

**Nous verrons ces options en détail plus bas.**

---

## Lister les Collections

### Commandes de Listage

```javascript
// Méthode 1 : Commande shell simple
show collections

// Méthode 2 : Obtenir un tableau de noms
db.getCollectionNames()

// Méthode 3 : Informations détaillées
db.getCollectionInfos()
```

### Exemples de Sortie

**`show collections` :**
```
articles
auteurs
categories
commentaires
```

**`db.getCollectionNames()` :**
```javascript
[ 'articles', 'auteurs', 'categories', 'commentaires' ]
```

**`db.getCollectionInfos()` :**
```javascript
[
  {
    name: 'articles',
    type: 'collection',
    options: {},
    info: {
      readOnly: false,
      uuid: UUID("...")
    },
    idIndex: {
      v: 2,
      key: { _id: 1 },
      name: '_id_'
    }
  },
  // ... autres collections
]
```

### Filtrer les Collections

```javascript
// Lister les collections dont le nom commence par "log"
db.getCollectionNames().filter(name => name.startsWith('log'))

// Compter les collections
db.getCollectionNames().length
```

---

## Conventions de Nommage

### Règles Obligatoires

MongoDB impose certaines contraintes pour nommer les collections :

✅ **Autorisé :**
```javascript
db.createCollection("utilisateurs")
db.createCollection("articles_blog")
db.createCollection("produits2024")
db.createCollection("logs-application")
db.createCollection("_private")
```

❌ **Interdit :**
```javascript
db.createCollection("")              // ❌ Nom vide
db.createCollection("système.users") // ❌ Commence par "system."
db.createCollection("mon$collection") // ❌ Contient $
```

### Contraintes Techniques

| Contrainte | Limite |
|------------|--------|
| **Longueur max** | 120 caractères (avec le nom de base) |
| **Caractères interdits** | `$` (sauf dans les collections système) |
| **Préfixe réservé** | `system.` (réservé à MongoDB) |
| **Nom nul** | Interdit |

### Calcul de la Longueur Totale

```javascript
// Nom complet = nom_base + "." + nom_collection
// Exemple :
"ma_base_de_donnees.ma_collection_tres_longue_avec_beaucoup_de_caracteres"
// Doit faire moins de 120 caractères au total
```

### Recommandations de Nommage

**✅ Bonnes pratiques :**

```javascript
// Noms descriptifs et pluriels
db.createCollection("utilisateurs")
db.createCollection("articles")
db.createCollection("commandes")

// Snake case (recommandé)
db.createCollection("logs_application")
db.createCollection("sessions_utilisateur")
db.createCollection("historique_commandes")

// Noms cohérents
db.createCollection("produits")       // ✅
db.createCollection("categories")     // ✅
db.createCollection("sous_categories") // ✅
```

**❌ À éviter :**

```javascript
// Noms trop génériques
db.createCollection("data")      // ❌ Trop vague
db.createCollection("items")     // ❌ Peu descriptif
db.createCollection("temp")      // ❌ Ambigu

// Mélange de conventions
db.createCollection("Utilisateurs")  // ❌ PascalCase
db.createCollection("articles")      // ✅ minuscules
db.createCollection("Commandes")     // ❌ Incohérent

// Noms trop longs
db.createCollection("ma_super_collection_de_donnees_pour_les_utilisateurs_v2") // ❌
```

### Conseils Pratiques

1. **Utilisez le pluriel** : `users` plutôt que `user`
2. **Minuscules** : `articles` plutôt que `Articles`
3. **Descriptif** : `product_reviews` plutôt que `reviews`
4. **Cohérence** : Choisissez une convention et gardez-la
5. **Anglais vs Français** : Choisissez une langue et restez-y

```javascript
// ✅ Cohérent (anglais)
db.users
db.products
db.orders

// ✅ Cohérent (français)
db.utilisateurs
db.produits
db.commandes

// ❌ Incohérent (mélange)
db.users
db.produits
db.orders
```

---

## Options de Création

### Options Principales

Lors de la création explicite, vous pouvez spécifier plusieurs options :

```javascript
db.createCollection("ma_collection", {
  capped: <boolean>,           // Collection à taille limitée
  size: <number>,              // Taille max en octets (pour capped)
  max: <number>,               // Nombre max de documents (pour capped)
  validator: <document>,       // Règles de validation
  validationLevel: <string>,   // Niveau de validation
  validationAction: <string>,  // Action en cas d'échec
  collation: <document>,       // Règles de tri et comparaison
  timeseries: <document>,      // Options pour séries temporelles
  expireAfterSeconds: <number> // TTL pour séries temporelles
})
```

### Option : Capped Collections

**Qu'est-ce qu'une Capped Collection ?**

Une collection à **taille fixe** qui fonctionne comme un buffer circulaire :
- Taille maximum définie
- Les anciens documents sont automatiquement supprimés
- Ordre d'insertion préservé
- Performances optimales en écriture

**Création :**
```javascript
db.createCollection("logs", {
  capped: true,
  size: 5242880,    // 5 Mo
  max: 5000         // 5000 documents max
})
```

**Cas d'usage :**
- 📋 **Logs** : Garder uniquement les logs récents
- 📊 **Métriques** : Données de monitoring
- 💬 **Chat** : Messages récents
- 🔔 **Notifications** : Notifications temporaires

**Exemple complet :**
```javascript
// Collection de logs limitée
db.createCollection("application_logs", {
  capped: true,
  size: 10485760,  // 10 Mo
  max: 10000       // 10000 documents max
})

// Insérer des logs
db.application_logs.insertOne({
  niveau: "INFO",
  message: "Application démarrée",
  timestamp: new Date()
})

// Les anciens logs sont automatiquement supprimés
// quand la limite est atteinte
```

**⚠️ Limitations des Capped Collections :**
- ❌ Impossible de supprimer des documents individuellement
- ❌ Impossible d'augmenter la taille d'un document (update limité)
- ❌ Pas de sharding
- ✅ Mais très performantes pour l'insertion !

### Option : Validation de Schéma

**Définir des règles de validation :**
```javascript
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "email"],
      properties: {
        nom: {
          bsonType: "string",
          description: "Nom obligatoire en string"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
          description: "Email valide obligatoire"
        },
        age: {
          bsonType: "int",
          minimum: 0,
          maximum: 150,
          description: "Age entre 0 et 150"
        }
      }
    }
  },
  validationLevel: "strict",    // strict | moderate
  validationAction: "error"     // error | warn
})
```

**Niveaux de validation :**

| Niveau | Description |
|--------|-------------|
| **strict** | Valide tous les inserts et updates |
| **moderate** | Valide uniquement les documents qui respectent déjà le schéma |

**Actions de validation :**

| Action | Description |
|--------|-------------|
| **error** | Rejette les documents invalides |
| **warn** | Accepte mais enregistre un warning dans les logs |

**Exemple pratique :**
```javascript
// Création avec validation
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["nom", "prix"],
      properties: {
        nom: {
          bsonType: "string",
          minLength: 3
        },
        prix: {
          bsonType: ["double", "decimal"],
          minimum: 0
        },
        stock: {
          bsonType: "int",
          minimum: 0
        }
      }
    }
  }
})

// ✅ Insertion valide
db.produits.insertOne({
  nom: "Laptop",
  prix: 999.99,
  stock: 10
})

// ❌ Échoue (nom trop court)
db.produits.insertOne({
  nom: "PC",
  prix: 999.99
})
// Erreur : Document failed validation
```

### Option : Collation

Définit les règles de tri et de comparaison (locale, casse, etc.) :

```javascript
db.createCollection("articles", {
  collation: {
    locale: "fr",          // Langue française
    strength: 1,           // Insensible à la casse et aux accents
    caseLevel: false
  }
})
```

---

## Renommer une Collection

### Commande `renameCollection()`

```javascript
// Syntaxe
db.ancienNom.renameCollection("nouveauNom")

// Exemple
db.users.renameCollection("utilisateurs")
```

**Ou via la commande admin :**
```javascript
db.adminCommand({
  renameCollection: "mabase.users",
  to: "mabase.utilisateurs"
})
```

**⚠️ Points importants :**
- Les index sont conservés
- La collection ne doit pas avoir le même nom qu'une collection existante
- Opération atomique
- Peut nécessiter des privilèges spéciaux

### Exemple Pratique

```javascript
use blog

// Créer une collection
db.articles_temp.insertOne({ titre: "Test" })

// Vérifier
show collections
// articles_temp

// Renommer
db.articles_temp.renameCollection("articles")

// Vérifier
show collections
// articles
```

---

## Supprimer une Collection

### Commande `drop()`

**⚠️ ATTENTION : Cette action est IRRÉVERSIBLE !**

```javascript
// Supprimer une collection
db.nom_collection.drop()
```

**Exemple :**
```javascript
use blog

// Supprimer la collection articles
db.articles.drop()

// Vérifier
show collections
// articles n'apparaît plus
```

**Résultat :**
```javascript
true  // Si la suppression réussit
false // Si la collection n'existe pas
```

### Supprimer Plusieurs Collections

```javascript
// Supprimer plusieurs collections
db.collection1.drop()
db.collection2.drop()
db.collection3.drop()

// Ou via une boucle
const collections = ["temp1", "temp2", "temp3"]
collections.forEach(coll => {
  db[coll].drop()
})
```

### Supprimer Tous les Documents (sans supprimer la collection)

```javascript
// Vider une collection (garder la structure)
db.articles.deleteMany({})

// La collection existe toujours, mais elle est vide
db.articles.countDocuments()  // 0
```

**Différence importante :**

```javascript
// drop() : Supprime la collection ET les index
db.articles.drop()

// deleteMany({}) : Supprime les documents mais garde les index
db.articles.deleteMany({})
```

---

## Statistiques d'une Collection

### Commande `stats()`

```javascript
// Statistiques de base
db.articles.stats()

// Statistiques en Ko
db.articles.stats(1024)

// Statistiques en Mo
db.articles.stats(1024 * 1024)
```

**Exemple de sortie :**
```javascript
{
  ns: 'blog.articles',
  size: 45678,              // Taille des données
  count: 150,               // Nombre de documents
  avgObjSize: 304,          // Taille moyenne d'un document
  storageSize: 49152,       // Espace de stockage
  nindexes: 3,              // Nombre d'index
  totalIndexSize: 98304,    // Taille totale des index
  indexSizes: {
    _id_: 36864,
    titre_1: 32768,
    auteur_1: 28672
  },
  ok: 1
}
```

### Informations Utiles

```javascript
// Nombre de documents
db.articles.countDocuments()

// Taille estimée
db.articles.estimatedDocumentCount()

// Taille d'un document spécifique
Object.bsonsize(db.articles.findOne())
```

---

## Types de Collections Spéciaux

### 1. Collections Normales

Les collections standard que nous avons vues jusqu'ici.

```javascript
db.createCollection("produits")
```

### 2. Capped Collections

Collections à taille fixe (déjà vues plus haut).

```javascript
db.createCollection("logs", {
  capped: true,
  size: 10485760
})
```

### 3. Time Series Collections

Optimisées pour les séries temporelles (IoT, métriques, etc.).

```javascript
db.createCollection("weather", {
  timeseries: {
    timeField: "timestamp",
    metaField: "sensor",
    granularity: "hours"
  }
})
```

**Cas d'usage :**
- 🌡️ Données météorologiques
- 📈 Métriques de performance
- 💓 Données médicales
- 🏭 Données IoT

### 4. Clustered Collections

Collections où l'index `_id` détermine l'ordre physique de stockage.

```javascript
db.createCollection("events", {
  clusteredIndex: {
    key: { _id: 1 },
    unique: true
  }
})
```

### 5. Views (Vues)

Collections en lecture seule basées sur des pipelines d'agrégation.

```javascript
db.createView(
  "articles_publies",           // Nom de la vue
  "articles",                    // Collection source
  [                              // Pipeline d'agrégation
    { $match: { publie: true } },
    { $project: { titre: 1, auteur: 1, date: 1 } }
  ]
)
```

**Utilisation :**
```javascript
// La vue se comporte comme une collection en lecture
db.articles_publies.find()
```

---

## Exemples de Gestion par Secteur

### Blog

```javascript
use blog

// Articles
db.createCollection("articles", {
  validator: {
    $jsonSchema: {
      required: ["titre", "contenu", "auteur"],
      properties: {
        titre: { bsonType: "string", minLength: 5 },
        contenu: { bsonType: "string" },
        auteur: { bsonType: "objectId" },
        publie: { bsonType: "bool" }
      }
    }
  }
})

// Commentaires
db.createCollection("commentaires", {
  validator: {
    $jsonSchema: {
      required: ["articleId", "auteur", "texte"],
      properties: {
        articleId: { bsonType: "objectId" },
        auteur: { bsonType: "string" },
        texte: { bsonType: "string", maxLength: 1000 }
      }
    }
  }
})

// Catégories (simple)
db.createCollection("categories")

// Logs (capped)
db.createCollection("logs_acces", {
  capped: true,
  size: 5242880,
  max: 10000
})
```

### E-commerce

```javascript
use boutique

// Produits avec validation
db.createCollection("produits", {
  validator: {
    $jsonSchema: {
      required: ["nom", "prix", "stock"],
      properties: {
        nom: { bsonType: "string" },
        prix: { bsonType: "decimal" },
        stock: { bsonType: "int", minimum: 0 }
      }
    }
  }
})

// Clients
db.createCollection("clients", {
  validator: {
    $jsonSchema: {
      required: ["email", "nom"],
      properties: {
        email: {
          bsonType: "string",
          pattern: "^.+@.+\\..+$"
        }
      }
    }
  }
})

// Commandes
db.createCollection("commandes")

// Panier (sessions temporaires)
db.createCollection("paniers")

// Historique des prix (time series)
db.createCollection("prix_historique", {
  timeseries: {
    timeField: "date",
    metaField: "produitId",
    granularity: "hours"
  }
})
```

### Application de Monitoring

```javascript
use monitoring

// Métriques système (time series)
db.createCollection("metriques_systeme", {
  timeseries: {
    timeField: "timestamp",
    metaField: "serveur",
    granularity: "minutes"
  },
  expireAfterSeconds: 2592000  // 30 jours
})

// Logs d'erreur (capped)
db.createCollection("logs_erreurs", {
  capped: true,
  size: 52428800,  // 50 Mo
  max: 50000
})

// Alertes
db.createCollection("alertes")

// Statistiques agrégées (vue)
db.createView(
  "stats_journalieres",
  "metriques_systeme",
  [
    {
      $group: {
        _id: {
          $dateToString: { format: "%Y-%m-%d", date: "$timestamp" }
        },
        cpuMoyen: { $avg: "$cpu" },
        memoireMoyenne: { $avg: "$memoire" }
      }
    }
  ]
)
```

---

## Opérations Avancées

### Compacter une Collection

Libère l'espace disque inutilisé :

```javascript
db.runCommand({ compact: "nom_collection" })
```

**⚠️ Attention :**
- Opération bloquante
- À faire en maintenance
- Pas disponible sur toutes les versions

### Convertir en Capped Collection

```javascript
db.runCommand({
  convertToCapped: "ma_collection",
  size: 10485760  // 10 Mo
})
```

### Cloner une Collection

```javascript
// Dans la même base
db.source.aggregate([
  { $match: {} },
  { $out: "destination" }
])

// Vers une autre base
db.source.aggregate([
  { $match: {} },
  { $out: {
    db: "autre_base",
    coll: "destination"
  }}
])
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Noms significatifs**
   ```javascript
   db.createCollection("utilisateurs")       // ✅ Clair
   db.createCollection("commandes_2024")     // ✅ Explicite
   ```

2. **Validation pour les données critiques**
   ```javascript
   // Pour les transactions financières
   db.createCollection("transactions", {
     validator: { /* règles strictes */ }
   })
   ```

3. **Capped collections pour les logs**
   ```javascript
   db.createCollection("logs", {
     capped: true,
     size: 10485760
   })
   ```

4. **Index appropriés dès la création**
   ```javascript
   db.createCollection("produits")
   db.produits.createIndex({ nom: 1 })
   db.produits.createIndex({ categorie: 1, prix: 1 })
   ```

5. **Documentation dans la base**
   ```javascript
   db.metadata.insertOne({
     collection: "utilisateurs",
     description: "Données des utilisateurs de l'application",
     dateCreation: new Date(),
     responsable: "Équipe Backend"
   })
   ```

### ❌ À Éviter

1. **Trop de collections**
   ```javascript
   // ❌ Évitez de créer une collection par entité mineure
   db.createCollection("user_1_preferences")
   db.createCollection("user_2_preferences")
   // ...

   // ✅ Préférez une collection avec un champ discriminant
   db.user_preferences.insertOne({
     userId: 1,
     preferences: { ... }
   })
   ```

2. **Collections sans structure**
   ```javascript
   // ❌ Collection "fourre-tout"
   db.data.insertOne({ type: "user", ... })
   db.data.insertOne({ type: "product", ... })

   // ✅ Collections séparées et typées
   db.users.insertOne({ ... })
   db.products.insertOne({ ... })
   ```

3. **Noms incohérents**
   ```javascript
   // ❌ Mélange de conventions
   db.Users
   db.products
   db.Order_Items

   // ✅ Convention uniforme
   db.users
   db.products
   db.order_items
   ```

4. **Absence de validation pour les données sensibles**
   ```javascript
   // ❌ Pas de validation
   db.createCollection("transactions_bancaires")

   // ✅ Validation stricte
   db.createCollection("transactions_bancaires", {
     validator: { /* validation complète */ }
   })
   ```

---

## Collections Système

MongoDB crée automatiquement des collections système :

| Collection | Description |
|------------|-------------|
| **system.indexes** | (Obsolète) Informations sur les index |
| **system.namespaces** | (Obsolète) Liste des collections |
| **system.profile** | Données du profiler de requêtes |
| **system.users** | Utilisateurs de la base |
| **system.roles** | Rôles personnalisés |
| **system.views** | Définitions des vues |

**⚠️ Ne modifiez jamais ces collections directement !**

```javascript
// ❌ Ne faites pas ça
db.system.users.deleteMany({})

// ✅ Utilisez les commandes appropriées
db.dropUser("nom_utilisateur")
```

---

## Commandes Récapitulatives

| Commande | Description | Exemple |
|----------|-------------|---------|
| `db.createCollection()` | Créer une collection | `db.createCollection("users")` |
| `show collections` | Lister les collections | `show collections` |
| `db.getCollectionNames()` | Obtenir noms (array) | `db.getCollectionNames()` |
| `db.collection.renameCollection()` | Renommer | `db.old.renameCollection("new")` |
| `db.collection.drop()` | Supprimer | `db.users.drop()` |
| `db.collection.stats()` | Statistiques | `db.users.stats()` |
| `db.collection.countDocuments()` | Compter | `db.users.countDocuments()` |

---

## Points Clés à Retenir

### ✅ Essentiel

1. **Création implicite** : Collections créées automatiquement à l'insertion
2. **Création explicite** : Utile pour définir options et validation
3. **Schéma flexible** : Documents différents dans la même collection
4. **Types spéciaux** : Capped, Time Series, Views
5. **Nommage** : Cohérence et descriptivité essentielles
6. **Validation** : Optionnelle mais recommandée pour données critiques

### 🎯 Workflow Standard

```javascript
// 1. Sélectionner la base
use mon_projet

// 2. Créer la collection (implicite ou explicite)
db.createCollection("utilisateurs")

// 3. Configurer les index
db.utilisateurs.createIndex({ email: 1 }, { unique: true })

// 4. Insérer des données
db.utilisateurs.insertOne({ nom: "Dupont", email: "dupont@example.com" })
```

### ⚠️ Pièges Courants

- ❌ Oublier de créer des index sur les champs fréquemment recherchés
- ❌ Utiliser des capped collections pour des données qu'on veut garder
- ❌ Créer trop de collections au lieu d'utiliser l'imbrication
- ❌ Négliger la validation pour les données critiques

---

## Prochaines Étapes

Maintenant que vous maîtrisez les collections, découvrons les opérations fondamentales :

➡️ **2.5 Opérations CRUD de base** : Créer, lire, modifier et supprimer des documents

Vous avez maintenant les fondations pour organiser vos données dans MongoDB !

---

## Ressources Complémentaires

### Documentation Officielle

- [Collections - MongoDB Manual](https://docs.mongodb.com/manual/core/databases-and-collections/#collections)
- [createCollection()](https://docs.mongodb.com/manual/reference/method/db.createCollection/)
- [Capped Collections](https://docs.mongodb.com/manual/core/capped-collections/)
- [Schema Validation](https://docs.mongodb.com/manual/core/schema-validation/)

### Pour Aller Plus Loin

- Optimisation des performances des collections
- Stratégies de partitionnement
- Collections distribuées (sharding)

---


⏭️ [Opérations CRUD de base](/02-fondamentaux-de-mongodb/05-operations-crud.md)
