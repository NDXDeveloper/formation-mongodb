🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Fondamentaux de MongoDB

## Bienvenue dans le cœur technique de MongoDB ! 💾

Maintenant que vous avez compris **ce qu'est MongoDB** et **pourquoi l'utiliser**, il est temps de passer à la pratique ! Ce chapitre vous fera découvrir les fondamentaux techniques qui constituent le socle de votre expertise MongoDB. Vous allez enfin manipuler des données, créer vos premières bases et collections, et maîtriser les opérations CRUD essentielles.

## Où en sommes-nous dans votre parcours ?

Vous avez complété le chapitre 1 et vous comprenez maintenant :
- ✅ Ce qu'est MongoDB et son positionnement NoSQL
- ✅ Les différences conceptuelles avec les bases SQL
- ✅ Les cas d'usage appropriés
- ✅ L'architecture générale et la terminologie de base

**C'est parfait !** Vous êtes maintenant prêt à plonger dans la pratique et à manipuler MongoDB concrètement.

## Objectifs pédagogiques

À l'issue de ce chapitre, vous serez capable de :

- ✅ **Comprendre** la structure interne des documents BSON et les types de données disponibles
- ✅ **Créer** et gérer des bases de données et des collections
- ✅ **Effectuer** toutes les opérations CRUD (Create, Read, Update, Delete)
- ✅ **Utiliser** le shell MongoDB (mongosh) avec aisance
- ✅ **Naviguer** dans MongoDB Compass pour visualiser vos données
- ✅ **Manipuler** des documents de manière efficace et idiomatique

## De la théorie à la pratique

### Le passage crucial

Ce chapitre marque une transition importante dans votre apprentissage. Vous allez passer de la compréhension conceptuelle à la manipulation réelle. C'est ici que MongoDB va **prendre vie** sous vos doigts !

### Ce que vous allez faire concrètement

Dans ce chapitre, vous allez :
- 📝 Écrire vos premières requêtes MongoDB
- 🗄️ Créer vos premières structures de données
- 🔍 Interroger et filtrer des documents
- ✏️ Modifier et supprimer des données
- 🎯 Comprendre comment MongoDB stocke réellement vos informations

## Vue d'ensemble du chapitre

Ce chapitre est organisé en 7 sections progressives couvrant tous les fondamentaux pratiques :

### 🎯 Partie 1 : Structure et types (Sections 2.1 et 2.2)
Comprendre **BSON** (la représentation interne des données) et les **types de données** disponibles.

### 🎯 Partie 2 : Bases et collections (Sections 2.3 et 2.4)
Apprendre à **créer** et **gérer** des bases de données et des collections.

### 🎯 Partie 3 : Opérations CRUD (Section 2.5)
Maîtriser les opérations fondamentales : **Insert, Find, Update, Delete, Replace**.

### 🎯 Partie 4 : Outils (Sections 2.6 et 2.7)
Utiliser efficacement le **shell mongosh** et l'interface graphique **MongoDB Compass**.

## Premiers pas avec MongoDB : un exemple simple

Avant d'entrer dans les détails, voyons à quoi ressemble une interaction typique avec MongoDB. Ne vous inquiétez pas si tout n'est pas clair maintenant, chaque concept sera expliqué en profondeur dans les sections suivantes.

### Exemple : Gestion d'une collection de livres

```javascript
// Se connecter à MongoDB (via mongosh)
// mongosh "mongodb://localhost:27017"

// Créer/Sélectionner une base de données
use librairie

// Insérer un document (un livre)
db.livres.insertOne({
    titre: "Le Petit Prince",
    auteur: "Antoine de Saint-Exupéry",
    annee: 1943,
    genres: ["Fiction", "Philosophie", "Jeunesse"],
    prix: 8.50,
    disponible: true
})

// Résultat :
// {
//   acknowledged: true,
//   insertedId: ObjectId("507f1f77bcf86cd799439011")
// }

// Rechercher tous les livres
db.livres.find()

// Rechercher un livre spécifique
db.livres.findOne({ titre: "Le Petit Prince" })

// Mettre à jour le prix
db.livres.updateOne(
    { titre: "Le Petit Prince" },
    { $set: { prix: 9.50 } }
)

// Supprimer un livre
db.livres.deleteOne({ titre: "Le Petit Prince" })
```

### Que s'est-il passé ?

1. **Pas de schéma préalable** : Nous n'avons pas eu à définir de structure avant d'insérer des données
2. **Format JSON/Document** : Les données ressemblent à des objets JavaScript natifs
3. **Flexibilité** : Chaque document peut avoir des champs différents
4. **Syntaxe intuitive** : Les opérations sont simples et lisibles

## Les concepts clés de ce chapitre

### 1. BSON : Le format de stockage

BSON (Binary JSON) est le format que MongoDB utilise en interne pour stocker les documents. Il ressemble à JSON mais offre des avantages supplémentaires :

```javascript
// Ce que vous écrivez (JSON-like)
{
    nom: "Alice",
    age: 30,
    dateInscription: new Date("2024-01-15")
}

// Comment MongoDB le stocke (BSON)
// - Types binaires optimisés
// - Support de types supplémentaires (Date, ObjectId, etc.)
// - Performance accrue pour les opérations
```

**Pourquoi c'est important ?** Comprendre BSON vous aidera à choisir les bons types de données et à optimiser vos requêtes.

### 2. Structure hiérarchique

MongoDB organise vos données en trois niveaux :

```
📊 Serveur MongoDB
  └─ 📂 Base de données (librairie)
      └─ 📁 Collection (livres)
          ├─ 📄 Document 1
          ├─ 📄 Document 2
          └─ 📄 Document 3
```

**Analogie SQL :**
- Base de données = Base de données
- Collection = Table
- Document = Ligne/Enregistrement

### 3. Les opérations CRUD

CRUD est l'acronyme des quatre opérations fondamentales sur les données :

| Opération | Signification | Méthode MongoDB | Exemple d'usage |
|-----------|---------------|-----------------|-----------------|
| **C**reate | Créer | `insertOne()`, `insertMany()` | Ajouter un nouveau client |
| **R**ead | Lire | `find()`, `findOne()` | Rechercher des produits |
| **U**pdate | Mettre à jour | `updateOne()`, `updateMany()` | Modifier un prix |
| **D**elete | Supprimer | `deleteOne()`, `deleteMany()` | Retirer un article |

### 4. Documents flexibles

Un des aspects les plus puissants de MongoDB est la flexibilité des documents :

```javascript
// Document 1 : Un livre imprimé
{
    _id: ObjectId("..."),
    titre: "1984",
    auteur: "George Orwell",
    pages: 328,
    format: "papier"
}

// Document 2 : Un ebook avec des champs différents
{
    _id: ObjectId("..."),
    titre: "Le Meilleur des mondes",
    auteur: "Aldous Huxley",
    tailleNumero: 2.4,  // En Mo
    format: "epub",
    drm: false,
    liseuses: ["Kindle", "Kobo"]  // Champ absent dans Document 1
}

// Les deux documents peuvent coexister dans la même collection !
```

**Avantage :** Cette flexibilité permet à votre schéma d'évoluer naturellement avec votre application.

## Exemple pratique : Du SQL au MongoDB

Pour ceux qui viennent du monde SQL, voyons comment traduire des opérations familières :

### Créer une base et une table/collection

```sql
-- SQL
CREATE DATABASE librairie;
USE librairie;
CREATE TABLE livres (
    id INT PRIMARY KEY AUTO_INCREMENT,
    titre VARCHAR(200),
    auteur VARCHAR(100),
    annee INT
);
```

```javascript
// MongoDB
use librairie  // Crée la base automatiquement
// Pas besoin de créer la collection explicitement !
// Elle sera créée au premier insert
```

### Insérer des données

```sql
-- SQL
INSERT INTO livres (titre, auteur, annee)
VALUES ('1984', 'George Orwell', 1949);
```

```javascript
// MongoDB
db.livres.insertOne({
    titre: "1984",
    auteur: "George Orwell",
    annee: 1949
})
```

### Rechercher des données

```sql
-- SQL
SELECT * FROM livres WHERE auteur = 'George Orwell';
```

```javascript
// MongoDB
db.livres.find({ auteur: "George Orwell" })
```

### Mettre à jour

```sql
-- SQL
UPDATE livres
SET annee = 1950
WHERE titre = '1984';
```

```javascript
// MongoDB
db.livres.updateOne(
    { titre: "1984" },
    { $set: { annee: 1950 } }
)
```

### Supprimer

```sql
-- SQL
DELETE FROM livres WHERE titre = '1984';
```

```javascript
// MongoDB
db.livres.deleteOne({ titre: "1984" })
```

## Les outils que vous allez utiliser

### mongosh : Le shell interactif

mongosh est votre interface en ligne de commande pour MongoDB. C'est un outil puissant pour :
- Tester rapidement des requêtes
- Administrer votre base
- Exécuter des scripts
- Déboguer des problèmes

```javascript
// Lancement de mongosh
$ mongosh

// Vous verrez :
Current Mongosh Log ID: 507f1f77bcf86cd799439011
Connecting to: mongodb://127.0.0.1:27017
Using MongoDB: 7.0.0

test>
```

### MongoDB Compass : L'interface graphique

Compass est l'outil graphique officiel qui vous permet de :
- ✅ Visualiser vos données de manière intuitive
- ✅ Construire des requêtes visuellement
- ✅ Analyser les performances
- ✅ Gérer les index
- ✅ Explorer votre schéma

**Avantage pour les débutants :** Compass vous aide à comprendre visuellement la structure de vos données.

## Philosophie MongoDB : quelques principes importants

### 1. Pas de schéma rigide (Schema-less)

```javascript
// Vous n'avez PAS besoin de définir ceci à l'avance :
// CREATE TABLE users (
//     nom VARCHAR(50),
//     email VARCHAR(100)
// );

// Vous insérez directement :
db.users.insertOne({
    nom: "Alice",
    email: "alice@example.com"
})

// Et plus tard, vous pouvez ajouter de nouveaux champs :
db.users.insertOne({
    nom: "Bob",
    email: "bob@example.com",
    telephone: "+33612345678",  // Nouveau champ
    preferences: {              // Structure imbriquée
        newsletter: true
    }
})
```

### 2. Documents = Objets naturels

MongoDB stocke les données comme vous les pensez dans votre code :

```javascript
// Votre objet JavaScript
const utilisateur = {
    nom: "Charlie",
    age: 28,
    adresse: {
        rue: "123 rue de la Paix",
        ville: "Paris",
        codePostal: "75001"
    },
    hobbies: ["lecture", "voyage", "photographie"]
}

// Vous l'insérez tel quel !
db.utilisateurs.insertOne(utilisateur)

// Pas besoin de le décomposer en plusieurs tables
```

### 3. L'_id automatique

Chaque document possède un identifiant unique automatiquement généré :

```javascript
// Vous insérez :
db.users.insertOne({ nom: "David" })

// MongoDB ajoute automatiquement :
{
    _id: ObjectId("507f1f77bcf86cd799439011"),  // Généré automatiquement
    nom: "David"
}

// Vous pouvez aussi fournir votre propre _id :
db.users.insertOne({
    _id: "user_001",  // _id personnalisé
    nom: "Eve"
})
```

## Structure des sections à venir

Voici un aperçu détaillé de ce que vous allez apprendre dans chaque section :

### Section 2.1 : Structure des documents BSON
- Comment MongoDB représente les données en interne
- Les avantages de BSON par rapport à JSON
- La structure d'un document MongoDB

### Section 2.2 : Types de données BSON
- Types primitifs : String, Number, Boolean, Date
- Types spéciaux : ObjectId, Binary, Decimal128
- Tableaux et documents imbriqués
- Null et valeurs manquantes

### Section 2.3 : Création d'une base de données
- Commande `use`
- Création implicite vs explicite
- Visualisation des bases existantes
- Suppression de bases

### Section 2.4 : Création et gestion des collections
- Création explicite avec `createCollection()`
- Création implicite au premier insert
- Options de collections
- Gestion et suppression

### Section 2.5 : Opérations CRUD de base
Divisée en 5 sous-sections détaillées :
- **2.5.1** : `insertOne()` et `insertMany()` - Créer des documents
- **2.5.2** : `find()` et `findOne()` - Lire des documents
- **2.5.3** : `updateOne()` et `updateMany()` - Modifier des documents
- **2.5.4** : `deleteOne()` et `deleteMany()` - Supprimer des documents
- **2.5.5** : `replaceOne()` - Remplacer complètement un document

### Section 2.6 : Le shell MongoDB (mongosh)
- Navigation et commandes de base
- Helpers et raccourcis
- Scripting avec mongosh
- Configuration et personnalisation

### Section 2.7 : Introduction à MongoDB Compass
- Installation et connexion
- Navigation dans l'interface
- Opérations CRUD visuelles
- Analyse et exploration des données

## Votre premier exemple complet

Voici un exemple complet qui vous donne un aperçu de ce que vous saurez faire à la fin de ce chapitre :

```javascript
// 1. Sélectionner/Créer la base de données
use blogDB

// 2. Insérer plusieurs articles de blog
db.articles.insertMany([
    {
        titre: "Introduction à MongoDB",
        auteur: "Alice Dupont",
        contenu: "MongoDB est une base de données NoSQL...",
        tags: ["mongodb", "nosql", "database"],
        datePublication: new Date("2024-01-15"),
        vues: 0,
        commentaires: []
    },
    {
        titre: "Guide JavaScript ES6",
        auteur: "Bob Martin",
        contenu: "ES6 apporte de nombreuses nouveautés...",
        tags: ["javascript", "es6", "web"],
        datePublication: new Date("2024-01-20"),
        vues: 0,
        commentaires: []
    }
])

// 3. Rechercher tous les articles d'un auteur
db.articles.find({ auteur: "Alice Dupont" })

// 4. Incrémenter le nombre de vues
db.articles.updateOne(
    { titre: "Introduction à MongoDB" },
    { $inc: { vues: 1 } }  // $inc incrémente de 1
)

// 5. Ajouter un commentaire à un article
db.articles.updateOne(
    { titre: "Introduction à MongoDB" },
    {
        $push: {
            commentaires: {
                auteur: "Charlie",
                texte: "Excellent article !",
                date: new Date()
            }
        }
    }
)

// 6. Rechercher les articles publiés après une certaine date
db.articles.find({
    datePublication: {
        $gte: new Date("2024-01-18")
    }
})

// 7. Supprimer les articles avec 0 vues
db.articles.deleteMany({ vues: 0 })
```

### Analyse de l'exemple

Cet exemple illustre plusieurs concepts fondamentaux :
- ✅ Insertion multiple de documents
- ✅ Recherche avec critères
- ✅ Mise à jour avec opérateurs (`$inc`, `$push`)
- ✅ Documents imbriqués (tableau de commentaires)
- ✅ Types de données variés (String, Number, Date, Array)
- ✅ Filtres avec opérateurs de comparaison (`$gte`)

## Points d'attention pour les débutants

### 1. La création implicite

```javascript
// MongoDB crée automatiquement :
use nouvelleBase        // La base n'existe pas encore
db.nouvelleCollection.insertOne({ test: 1 })
// ✅ La base ET la collection sont créées automatiquement !
```

**Important :** Les bases et collections vides ne sont pas persistées. Elles n'apparaissent qu'après le premier insert.

### 2. L'_id est sacré

```javascript
// Chaque document a un _id unique
db.users.insertOne({ nom: "Alice" })  // _id généré automatiquement

// Erreur si vous essayez d'insérer deux fois le même _id
db.users.insertOne({ _id: 1, nom: "Bob" })   // ✅ OK
db.users.insertOne({ _id: 1, nom: "Charlie" }) // ❌ Erreur : duplicate key
```

### 3. Les opérations sont atomiques par document

```javascript
// Cette opération est atomique (tout ou rien)
db.users.updateOne(
    { nom: "Alice" },
    {
        $set: { age: 30 },
        $push: { hobbies: "lecture" }
    }
)
// Les deux modifications réussissent ensemble ou échouent ensemble
```

### 4. La syntaxe des opérateurs

MongoDB utilise le préfixe `$` pour ses opérateurs :

```javascript
// Opérateurs de mise à jour
$set     // Définir une valeur
$inc     // Incrémenter
$push    // Ajouter à un tableau
$pull    // Retirer d'un tableau

// Opérateurs de requête
$eq      // Égal à
$gt      // Plus grand que
$in      // Dans un tableau de valeurs
```

## Ressources et environnement

### Configuration recommandée

Pour suivre ce chapitre efficacement, assurez-vous d'avoir :
- ✅ MongoDB installé localement (ou accès à un cluster Atlas)
- ✅ mongosh installé et fonctionnel
- ✅ MongoDB Compass installé (optionnel mais recommandé)
- ✅ Un éditeur de texte pour noter vos requêtes

### Environnement de test

Créez un environnement de test dédié :

```javascript
// Base de test dédiée à l'apprentissage
use formation_mongodb

// Vous pouvez expérimenter librement !
// Suppression complète si besoin :
// db.dropDatabase()
```

## Conseils d'apprentissage

### 🎯 Approche recommandée

1. **Lisez d'abord la théorie** de chaque section
2. **Testez chaque exemple** dans mongosh
3. **Modifiez les exemples** pour expérimenter
4. **Explorez avec Compass** pour visualiser
5. **Passez à la section suivante** une fois à l'aise

### 💡 Astuces pratiques

- **Utilisez la complétion automatique** : TAB dans mongosh
- **Consultez l'aide** : `db.collection.help()`
- **Affichez les données joliment** : `.pretty()` après `find()`
- **Gardez la documentation ouverte** : docs.mongodb.com

### ⚠️ Erreurs courantes à éviter

```javascript
// ❌ Oublier les guillemets pour les chaînes
db.users.find({ nom: Alice })  // Erreur !

// ✅ Correct
db.users.find({ nom: "Alice" })

// ❌ Utiliser = au lieu de :
db.users.insertOne({ nom = "Bob" })  // Erreur !

// ✅ Correct
db.users.insertOne({ nom: "Bob" })

// ❌ Oublier les accolades pour les filtres
db.users.find("nom", "Alice")  // Erreur !

// ✅ Correct
db.users.find({ nom: "Alice" })
```

## Transition depuis le chapitre précédent

Dans le chapitre 1, vous avez appris la **théorie** :
- Les concepts NoSQL
- L'architecture MongoDB
- Les cas d'usage

Dans ce chapitre 2, vous apprenez la **pratique** :
- Comment manipuler réellement les données
- Les commandes concrètes
- Les outils pour travailler efficacement

## Ce qui vous attend ensuite

Après avoir maîtrisé ce chapitre, vous serez prêt pour :
- **Chapitre 3 : Requêtes et Filtres** - Recherches avancées, opérateurs complexes
- **Chapitre 4 : Modélisation des Données** - Conception optimale de vos structures
- **Chapitre 5 : Index et Optimisation** - Performances et scalabilité

Mais avant d'y arriver, vous devez d'abord maîtriser les fondamentaux !

---

### 📌 Points clés à retenir de cette introduction

- Ce chapitre vous fait passer de la théorie à la pratique
- BSON est le format interne optimisé de MongoDB (JSON-like mais binaire)
- Les opérations CRUD sont au cœur de toute interaction avec MongoDB
- MongoDB crée automatiquement les bases et collections au besoin
- Chaque document possède un `_id` unique (auto-généré ou personnalisé)
- Deux outils principaux : mongosh (CLI) et Compass (GUI)
- Les documents peuvent avoir des structures flexibles et évolutives

---

**Durée estimée du chapitre** : 5-7 heures de lecture et pratique
**Niveau** : Débutant ayant compris les bases conceptuelles
**Prérequis** : Chapitre 1 complété, MongoDB installé

🎯 **Prochaine étape** : Dans la section 2.1, nous allons plonger dans la structure des documents BSON et comprendre comment MongoDB stocke réellement vos données.

---

**Prochaine section** : 2.1 - Structure des documents BSON

Allons manipuler vos premières données ! 🚀

⏭️ [Structure des documents BSON](/02-fondamentaux-de-mongodb/01-structure-documents-bson.md)
