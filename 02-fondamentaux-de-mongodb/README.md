🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2. Fondamentaux de MongoDB

## Introduction au Chapitre

Bienvenue dans le chapitre consacré aux **fondamentaux de MongoDB** ! C'est ici que vous allez construire les bases solides nécessaires pour devenir un utilisateur efficace de MongoDB.

> **💡 Pourquoi ce chapitre est crucial :** Tout comme un architecte doit comprendre les matériaux avant de construire, vous devez comprendre comment MongoDB stocke et structure les données avant de pouvoir les manipuler efficacement.

Dans ce chapitre, nous allons explorer les concepts essentiels qui font de MongoDB ce qu'il est : une base de données orientée documents, flexible et performante.

---

## Objectifs du Chapitre

À la fin de ce chapitre, vous serez capable de :

- ✅ Comprendre la structure des documents BSON et leurs particularités
- ✅ Maîtriser les types de données disponibles dans MongoDB
- ✅ Créer et gérer des bases de données
- ✅ Créer et configurer des collections
- ✅ Effectuer toutes les opérations CRUD (Create, Read, Update, Delete)
- ✅ Utiliser le shell MongoDB (mongosh) avec confiance
- ✅ Naviguer dans MongoDB Compass (interface graphique)

---

## Qu'est-ce qu'un Fondamental ?

Un **fondamental** est un concept de base, essentiel, qui sert de socle pour tout le reste. Dans MongoDB, les fondamentaux incluent :

### 1. La Structure des Données
**Comment MongoDB organise l'information**
- Documents au format BSON (Binary JSON)
- Collections qui regroupent les documents
- Bases de données qui contiennent les collections

### 2. Les Types de Données
**Les différentes formes que peuvent prendre vos données**
- Texte, nombres, dates, booléens
- Tableaux et objets imbriqués
- Types spéciaux (ObjectId, Binary, etc.)

### 3. Les Opérations de Base
**Comment manipuler vos données**
- Créer (Insert)
- Lire (Find)
- Modifier (Update)
- Supprimer (Delete)

### 4. Les Outils
**Comment interagir avec MongoDB**
- Le shell en ligne de commande (mongosh)
- L'interface graphique (MongoDB Compass)

---

## Pourquoi Ces Fondamentaux Sont Importants

### Sans Fondamentaux Solides

```
❌ Structures de données inefficaces
❌ Erreurs fréquentes dans les requêtes
❌ Performance médiocre
❌ Difficultés à faire évoluer l'application
❌ Code difficile à maintenir
```

### Avec Fondamentaux Maîtrisés

```
✅ Modèles de données optimaux
✅ Requêtes précises et rapides
✅ Excellentes performances
✅ Application évolutive
✅ Code propre et maintenable
```

---

## Vue d'Ensemble du Chapitre

### Architecture de MongoDB

```
┌────────────────────────────────────┐
│      Serveur MongoDB               │
├────────────────────────────────────┤
│                                    │
│  ┌─────────────────────────────┐   │
│  │  Base de Données 1          │   │
│  │                             │   │
│  │  ┌─────────────────────┐    │   │
│  │  │  Collection A       │    │   │
│  │  │                     │    │   │
│  │  │  • Document 1       │    │   │
│  │  │  • Document 2       │    │   │
│  │  │  • Document 3       │    │   │
│  │  │  • ...              │    │   │
│  │  └─────────────────────┘    │   │
│  │                             │   │
│  │  ┌─────────────────────┐    │   │
│  │  │  Collection B       │    │   │
│  │  │  • Documents...     │    │   │
│  │  └─────────────────────┘    │   │
│  └─────────────────────────────┘   │
│                                    │
│  ┌─────────────────────────────┐   │
│  │  Base de Données 2          │   │
│  │  • Collections...           │   │
│  └─────────────────────────────┘   │
└────────────────────────────────────┘
```

### Hiérarchie des Concepts

```
Serveur MongoDB
    └── Base de données
            └── Collection
                    └── Document
                            └── Champs (avec types de données)
```

---

## Ce Que Vous Allez Apprendre

### Section 2.1 : Structure des Documents BSON

**Le format de stockage de MongoDB**

Vous découvrirez :
- Ce qu'est BSON (Binary JSON)
- Comment MongoDB structure les documents
- Les différences entre JSON et BSON
- L'importance du champ `_id`
- Les documents imbriqués et tableaux

**Exemple de document :**
```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  nom: "Dupont",
  age: 30,
  adresse: {
    rue: "123 Rue Example",
    ville: "Paris"
  },
  competences: ["JavaScript", "MongoDB"]
}
```

### Section 2.2 : Types de Données BSON

**La richesse des types MongoDB**

Vous apprendrez :
- Les types de base (String, Number, Boolean)
- Les types numériques (Int32, Int64, Double, Decimal128)
- Les types spéciaux (ObjectId, Date, Binary)
- Comment choisir le bon type
- La conversion entre types

**Pourquoi c'est important :** Utiliser le mauvais type peut causer des bugs ou des pertes de précision (ex: prix en Double au lieu de Decimal128).

### Section 2.3 : Création d'une Base de Données

**Votre premier espace de travail**

Vous maîtriserez :
- La création de bases de données
- Les conventions de nommage
- Les commandes essentielles
- La gestion multi-environnements
- Les bases de données système

**Particularité MongoDB :** Une base de données est créée automatiquement dès qu'on y insère des données (création "lazy").

### Section 2.4 : Création et Gestion des Collections

**Organiser vos documents**

Vous découvrirez :
- La création de collections
- Les options de configuration
- Les collections spéciales (capped, time series)
- La validation de schéma
- Les opérations de maintenance

**Flexibilité :** Contrairement aux tables SQL, les collections MongoDB n'ont pas de schéma obligatoire.

### Section 2.5 : Opérations CRUD de Base

**Manipuler vos données au quotidien**

Vous maîtriserez les 4 opérations fondamentales :

**CREATE** : Ajouter des données
- `insertOne()` : Un document
- `insertMany()` : Plusieurs documents

**READ** : Lire des données
- `find()` : Plusieurs documents
- `findOne()` : Un document

**UPDATE** : Modifier des données
- `updateOne()` : Un document
- `updateMany()` : Plusieurs documents
- `replaceOne()` : Remplacer complètement

**DELETE** : Supprimer des données
- `deleteOne()` : Un document
- `deleteMany()` : Plusieurs documents

**Le plus important :** Ces 4 opérations représentent 90% de votre travail quotidien avec MongoDB.

### Section 2.6 : Le Shell MongoDB (mongosh)

**Votre outil en ligne de commande**

Vous apprendrez :
- Installation et configuration
- Connexion à MongoDB
- Commandes de base et helpers
- Scripts JavaScript
- Astuces et raccourcis

**Avantage :** mongosh est puissant, scriptable et disponible partout.

### Section 2.7 : Introduction à MongoDB Compass

**L'interface graphique officielle**

Vous découvrirez :
- Installation et connexion
- Explorer les données visuellement
- Analyse de schéma
- Construction de requêtes
- Gestion des index

**Complément parfait :** Compass pour explorer et déboguer, mongosh pour automatiser et scripter.

---

## Parcours d'Apprentissage Recommandé

### Pour les Débutants Complets

```
1. Section 2.1 → Comprendre BSON
2. Section 2.2 → Connaître les types
3. Section 2.7 → Installer Compass (interface visuelle)
4. Section 2.3 → Créer une base
5. Section 2.4 → Créer une collection
6. Section 2.5 → Pratiquer CRUD
7. Section 2.6 → Maîtriser mongosh
```

**Conseil :** Prenez votre temps sur les sections 2.5 (CRUD). C'est le cœur de tout le reste.

### Pour Ceux Qui Connaissent SQL

```
1. Section 2.1 → Documents vs Tables
2. Section 2.2 → Types MongoDB vs SQL
3. Section 2.5 → CRUD (comparer avec INSERT, SELECT, UPDATE, DELETE)
4. Section 2.6 → mongosh (similaire à psql ou mysql client)
5. Sections 2.3 et 2.4 → Créer structures
6. Section 2.7 → Compass (optionnel)
```

**Focus :** Concentrez-vous sur les différences conceptuelles plutôt que sur les similitudes.

---

## Comparaison : SQL vs MongoDB

### Terminologie

| Concept SQL | Concept MongoDB | Description |
|-------------|-----------------|-------------|
| Base de données | Base de données | Conteneur principal |
| Table | Collection | Groupe de documents |
| Ligne/Enregistrement | Document | Unité de données |
| Colonne | Champ | Attribut d'un document |
| Index | Index | Optimisation des requêtes |
| JOIN | Embedding / $lookup | Liaison de données |
| PRIMARY KEY | _id | Identifiant unique |

### Philosophie

**SQL (Relationnel)**
```
Structure rigide → Relations → Jointures
```

**MongoDB (Orienté Documents)**
```
Structure flexible → Embedding → Dénormalisation
```

### Exemple Comparatif

**SQL : Deux tables liées**
```sql
-- Table utilisateurs
CREATE TABLE utilisateurs (
    id INT PRIMARY KEY,
    nom VARCHAR(50),
    email VARCHAR(100)
);

-- Table adresses (relation 1-1)
CREATE TABLE adresses (
    id INT PRIMARY KEY,
    user_id INT,
    rue VARCHAR(100),
    ville VARCHAR(50),
    FOREIGN KEY (user_id) REFERENCES utilisateurs(id)
);
```

**MongoDB : Un document auto-suffisant**
```javascript
// Collection utilisateurs
{
  _id: ObjectId("..."),
  nom: "Dupont",
  email: "dupont@example.com",
  adresse: {
    rue: "123 Rue Example",
    ville: "Paris"
  }
}
```

**Avantage MongoDB :** Pas de jointure nécessaire, tout est dans un document.

---

## Concepts Clés à Comprendre

### 1. Document-Oriented (Orienté Documents)

**Définition :** Les données sont stockées sous forme de documents JSON/BSON, pas de lignes dans des tables.

**Avantage :**
- Structure naturelle pour les applications modernes
- Mapping direct avec les objets JavaScript/Python/etc.
- Flexibilité du schéma

### 2. Schema Flexibility (Flexibilité du Schéma)

**Définition :** Les documents d'une même collection peuvent avoir des structures différentes.

**Exemple :**
```javascript
// Document 1
{ nom: "Alice", age: 25 }

// Document 2 (structure différente !)
{ nom: "Bob", ville: "Paris", competences: ["JS", "Python"] }
```

**Attention :** Flexibilité ≠ Anarchie. Une certaine cohérence reste recommandée.

### 3. Embedded Documents (Documents Imbriqués)

**Définition :** Un document peut contenir d'autres documents.

**Exemple :**
```javascript
{
  utilisateur: "Alice",
  commande: {
    produits: [
      { nom: "Laptop", prix: 999 },
      { nom: "Souris", prix: 29 }
    ],
    total: 1028
  }
}
```

**Avantage :** Données liées stockées ensemble, pas de jointure nécessaire.

### 4. Atomic Operations (Opérations Atomiques)

**Définition :** Chaque opération sur un document est atomique (tout ou rien).

**Exemple :**
```javascript
// Cette mise à jour est atomique
db.comptes.updateOne(
  { _id: 1 },
  {
    $inc: { solde: 100 },
    $push: { transactions: { montant: 100, date: new Date() } }
  }
)
// Soit tout réussit, soit rien ne change
```

---

## Prérequis et Outils

### Connaissances Requises

Pour suivre ce chapitre confortablement :

**Indispensables :**
- ✅ Notions de base en programmation
- ✅ Compréhension du JSON
- ✅ Utilisation de la ligne de commande

**Utiles mais pas obligatoires :**
- 📚 Expérience avec une autre base de données
- 📚 Connaissances JavaScript
- 📚 Concepts de modélisation de données

### Outils à Installer

**1. MongoDB Server** (si pas déjà fait)
```bash
# Voir chapitre 1 pour l'installation
```

**2. MongoDB Shell (mongosh)**
```bash
# Inclus avec MongoDB ou téléchargeable séparément
mongosh --version
```

**3. MongoDB Compass** (optionnel mais recommandé)
```bash
# Interface graphique officielle
# Téléchargement : https://www.mongodb.com/try/download/compass
```

### Environnement de Test

**Configuration minimale :**
```javascript
// Connexion locale
mongosh "mongodb://localhost:27017"

// Créer une base de test
use apprentissage_mongodb

// Vérifier la connexion
db.runCommand({ ping: 1 })
```

---

## Conseils pour Réussir ce Chapitre

### 🎯 Stratégies d'Apprentissage

**1. Pratiquez activement**
```javascript
// ❌ Ne pas se contenter de lire
// ✅ Taper chaque exemple vous-même
```

**2. Expérimentez**
```javascript
// ✅ Modifiez les exemples
// ✅ Testez vos propres idées
// ✅ Cassez des choses pour comprendre
```

**3. Prenez des notes**
```javascript
// ✅ Notez les concepts difficiles
// ✅ Créez vos propres aide-mémoires
// ✅ Documentez vos découvertes
```

**4. Construisez un projet**
```javascript
// ✅ Appliquez à un cas réel
// Exemple : Blog, Todo list, Catalogue produits
```

### 💡 Astuces Pratiques

**Créez une base de données de test**
```javascript
use test_formation
db.dropDatabase()  // Recommencer à zéro si besoin
```

**Gardez MongoDB Compass ouvert**
```javascript
// Visualisez vos données pendant que vous codez
```

**Utilisez des données réalistes**
```javascript
// ❌ { nom: "test", age: 1 }
// ✅ { nom: "Alice Martin", age: 28, ville: "Lyon" }
```

**Testez toujours vos requêtes avec find() avant delete()**
```javascript
// 1. Voir ce qui sera affecté
db.collection.find(filtre)

// 2. Si OK, exécuter l'opération
db.collection.deleteMany(filtre)
```

### ⚠️ Pièges à Éviter

**1. Confondre find() et findOne()**
```javascript
// find() retourne un curseur
let resultat = db.users.find()  // Curseur, pas les données !

// findOne() retourne un document
let user = db.users.findOne()   // Document directement
```

**2. Oublier $set dans updateOne()**
```javascript
// ❌ Remplace tout le document
db.users.updateOne({ _id: 1 }, { age: 31 })

// ✅ Modifie seulement le champ age
db.users.updateOne({ _id: 1 }, { $set: { age: 31 } })
```

**3. Utiliser le mauvais type numérique**
```javascript
// ❌ Pour des montants financiers
{ prix: 19.99 }  // Double, imprécis !

// ✅ Pour des montants financiers
{ prix: NumberDecimal("19.99") }  // Précis
```

---

## Structure du Chapitre

Ce chapitre est organisé en 7 sections progressives :

### 📋 Plan Détaillé

```
2. Fondamentaux de MongoDB
│
├── 2.1 Structure des Documents BSON
│   ├── Format BSON
│   ├── Champ _id
│   ├── Documents imbriqués
│   └── Limitations
│
├── 2.2 Types de Données BSON
│   ├── Types de base
│   ├── Types numériques
│   ├── Types spéciaux
│   └── Conversions
│
├── 2.3 Création d'une Base de Données
│   ├── Commandes de création
│   ├── Conventions de nommage
│   ├── Gestion multi-environnements
│   └── Bases système
│
├── 2.4 Création et Gestion des Collections
│   ├── Création de collections
│   ├── Options et configuration
│   ├── Types de collections
│   └── Maintenance
│
├── 2.5 Opérations CRUD de Base
│   ├── 2.5.1 insertOne() et insertMany()
│   ├── 2.5.2 find() et findOne()
│   ├── 2.5.3 updateOne() et updateMany()
│   ├── 2.5.4 deleteOne() et deleteMany()
│   └── 2.5.5 replaceOne()
│
├── 2.6 Le Shell MongoDB (mongosh)
│   ├── Installation
│   ├── Commandes de base
│   ├── Scripts JavaScript
│   └── Configuration
│
└── 2.7 Introduction à MongoDB Compass
    ├── Interface graphique
    ├── Exploration de données
    ├── Construction de requêtes
    └── Analyse de schéma
```

---

## Temps Estimé

### Par Section

| Section | Lecture | Pratique | Total |
|---------|---------|----------|-------|
| 2.1 BSON | 20 min | 20 min | 40 min |
| 2.2 Types | 30 min | 30 min | 1h |
| 2.3 Bases de données | 15 min | 15 min | 30 min |
| 2.4 Collections | 20 min | 20 min | 40 min |
| 2.5 CRUD | 2h | 3h | 5h |
| 2.6 mongosh | 30 min | 45 min | 1h15 |
| 2.7 Compass | 20 min | 30 min | 50 min |
| **TOTAL** | **~4h** | **~5h** | **~9h** |

**Recommandation :** Étalez sur plusieurs sessions pour mieux assimiler.

---

## Prochaines Étapes

Une fois ce chapitre maîtrisé, vous serez prêt pour :

- ✅ **Chapitre 3 : Requêtes Avancées** - Filtres complexes, agrégations
- ✅ **Chapitre 4 : Modélisation de Données** - Design patterns, optimisation
- ✅ **Chapitre 5 : Index et Performance** - Optimiser vos requêtes
- ✅ **Projets Réels** - Construire des applications complètes

---

## Ressources et Support

### Documentation Officielle

- 📚 [MongoDB Manual](https://docs.mongodb.com/manual/)
- 📚 [BSON Specification](http://bsonspec.org/)
- 📚 [MongoDB University](https://university.mongodb.com/) (cours gratuits)

### Communauté

- 💬 [MongoDB Community Forums](https://www.mongodb.com/community/forums/)
- 💬 [Stack Overflow - MongoDB Tag](https://stackoverflow.com/questions/tagged/mongodb)
- 💬 [MongoDB User Groups](https://www.mongodb.com/community/user-groups)

### Outils

- 🛠️ [MongoDB Compass](https://www.mongodb.com/products/compass)
- 🛠️ [MongoDB Shell (mongosh)](https://www.mongodb.com/docs/mongodb-shell/)
- 🛠️ [MongoDB VSCode Extension](https://marketplace.visualstudio.com/items?itemName=mongodb.mongodb-vscode)

---

## Prêt à Commencer ?

Vous avez maintenant une vue d'ensemble complète de ce qui vous attend dans ce chapitre. Les fondamentaux sont la clé de votre réussite avec MongoDB.

**Conseil final :** Ne vous précipitez pas. Prenez le temps de bien comprendre chaque concept, pratiquez régulièrement, et n'hésitez pas à revenir sur les sections précédentes si nécessaire.

➡️ **Commencez maintenant : 2.1 Structure des Documents BSON**

Bonne chance dans votre apprentissage ! 🚀

---


⏭️ [Structure des documents BSON](/02-fondamentaux-de-mongodb/01-structure-documents-bson.md)
