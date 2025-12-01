🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.6 Le Shell MongoDB (mongosh)

## Introduction

Le **MongoDB Shell** (mongosh) est votre interface en ligne de commande pour interagir avec MongoDB. C'est un outil puissant et interactif qui vous permet d'exécuter des requêtes, gérer vos bases de données, tester vos commandes et bien plus encore.

> **💡 Note :** mongosh est la version moderne du shell MongoDB. Il remplace l'ancien shell `mongo` (déprécié) et offre de nombreuses améliorations : meilleure syntaxe, coloration syntaxique, autocomplétion avancée, et support complet de JavaScript moderne.

Dans cette section, nous allons explorer :
- Installation et démarrage de mongosh
- Les commandes essentielles
- La navigation et l'exploration
- L'utilisation de JavaScript
- Les fonctionnalités avancées
- Les bonnes pratiques

---

## Qu'est-ce que mongosh ?

### Définition

**mongosh** (MongoDB Shell) est :
- 🖥️ Un **interpréteur JavaScript** interactif
- 🔧 Un **outil d'administration** puissant
- 📝 Un **environnement de test** pour vos requêtes
- 🚀 Une **interface en ligne de commande** moderne

### Comparaison avec l'ancien shell

| Caractéristique | mongo (ancien) | mongosh (nouveau) |
|----------------|----------------|-------------------|
| **JavaScript** | ES5 | ES6+ moderne |
| **Coloration** | Non | Oui ✅ |
| **Autocomplétion** | Basique | Avancée ✅ |
| **Snippets** | Non | Oui ✅ |
| **Performance** | Correcte | Améliorée ✅ |
| **Support** | Déprécié | Actif ✅ |

---

## Installation de mongosh

### Vérifier si mongosh est installé

```bash
# Vérifier la version
mongosh --version

# Sortie attendue (exemple)
# 2.0.0
```

### Installation selon votre OS

**Sur Windows :**
```bash
# Via MongoDB Installer (recommandé)
# Télécharger depuis : https://www.mongodb.com/try/download/shell

# Ou via Chocolatey
choco install mongodb-shell
```

**Sur macOS :**
```bash
# Via Homebrew
brew install mongosh

# Ou télécharger depuis le site officiel
```

**Sur Linux (Ubuntu/Debian) :**
```bash
# Ajouter le repository MongoDB
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -

# Installer mongosh
sudo apt-get install -y mongodb-mongosh
```

**Via npm (toutes plateformes) :**
```bash
npm install -g mongosh
```

---

## Démarrer mongosh

### Connexion Locale par Défaut

```bash
# Connexion simple (localhost:27017)
mongosh

# Sortie :
# Current Mongosh Log ID: 6565d7d8f2a3b4c5d6e7f8a9
# Connecting to: mongodb://127.0.0.1:27017/?directConnection=true
# Using MongoDB: 7.0.0
# Using Mongosh: 2.0.0
```

### Connexion avec Options

```bash
# Connexion à un hôte spécifique
mongosh "mongodb://localhost:27017"

# Connexion à une base spécifique
mongosh "mongodb://localhost:27017/mabase"

# Avec authentification
mongosh "mongodb://username:password@localhost:27017"

# Connexion à MongoDB Atlas
mongosh "mongodb+srv://cluster0.xxxxx.mongodb.net/mabase" --username monuser
```

### Options de Ligne de Commande

```bash
# Aide
mongosh --help

# Version
mongosh --version

# Mode silencieux (sans banner)
mongosh --quiet

# Évaluer une commande et quitter
mongosh --eval "db.stats()"

# Exécuter un script JavaScript
mongosh script.js

# Mode non-interactif (pour scripts)
mongosh --nodb
```

---

## Interface et Prompt

### Le Prompt

Lorsque vous lancez mongosh, vous voyez un prompt :

```javascript
test>
```

**Signification :**
- `test` : nom de la base de données actuelle
- `>` : invite de commande (prompt)

**Le prompt change selon le contexte :**

```javascript
test>           // Base 'test' sélectionnée
blog>           // Base 'blog' sélectionnée
admin>          // Base 'admin' sélectionnée
```

### Coloration Syntaxique

mongosh colore automatiquement votre code :

```javascript
// Les commandes sont colorées
db.users.find({ age: { $gte: 18 } })

// Sortie colorée :
// - db en bleu
// - users en cyan
// - find en vert
// - { } en blanc
// - Opérateurs $gte en magenta
// - Valeurs en jaune
```

### Formatage de la Sortie

```javascript
// Sortie brute
db.users.find()

// Sortie formatée (jolie)
db.users.find().pretty()  // Note : pretty() est maintenant par défaut dans mongosh

// Limite l'affichage
db.users.find().limit(5)

// Sortie en JSON
JSON.stringify(db.users.findOne())
```

---

## Commandes de Base

### Navigation dans les Bases de Données

```javascript
// Afficher toutes les bases
show dbs
show databases  // Identique

// Afficher la base courante
db

// Changer de base
use mabase

// Afficher les collections
show collections

// Afficher les utilisateurs
show users

// Afficher les rôles
show roles
```

### Commandes d'Aide

```javascript
// Aide générale
help

// Aide sur les méthodes de db
db.help()

// Aide sur une collection
db.users.help()

// Aide sur une méthode spécifique
db.users.find.help()

// Lister les méthodes disponibles
db.getCollectionNames()
```

**Exemple de sortie `help` :**
```
Shell Help:

  use                                        Set current database
  show                                       'show databases'/'show dbs': Print a list of all available databases.
                                            'show collections'/'show tables': Print a list of all collections for current database.

  db.help()                                 Help on db methods
  db.mycollection.help()                    Help on collection methods

  exit / quit()                             Exit the shell
  cls / clear                               Clear the console
```

### Commandes Système

```javascript
// Quitter mongosh
exit
quit()      // Alternative
Ctrl+D      // Raccourci Linux/Mac
Ctrl+C      // Raccourci Windows

// Effacer l'écran
cls         // Windows
clear       // Linux/Mac
Ctrl+L      // Raccourci universel

// Recharger le shell
.reload
```

---

## Variables et JavaScript

### Utilisation de JavaScript

mongosh est un interpréteur JavaScript complet :

```javascript
// Variables
let nom = "Dupont"
const age = 30
var ville = "Paris"

// Affichage
print(nom)
console.log(age)

// Calculs
let total = 10 + 20
print(total)  // 30

// Tableaux
let fruits = ["pomme", "banane", "orange"]
print(fruits[0])  // pomme

// Objets
let personne = {
  nom: "Martin",
  age: 25,
  ville: "Lyon"
}
print(personne.nom)  // Martin

// Fonctions
function bonjour(nom) {
  return `Bonjour ${nom} !`
}
print(bonjour("Alice"))  // Bonjour Alice !
```

### Variables Prédéfinies

```javascript
// db : base de données courante
db

// version() : version de MongoDB
version()

// hostname() : nom de l'hôte
hostname()

// _ : résultat de la dernière commande
db.users.findOne()
print(_)  // Affiche le document trouvé
```

### Stocker des Résultats

```javascript
// Stocker le résultat d'une requête
let utilisateurs = db.users.find().toArray()

// Utiliser le résultat
print(utilisateurs.length)
print(utilisateurs[0].nom)

// Boucler sur les résultats
utilisateurs.forEach(user => {
  print(user.nom)
})

// Stocker un curseur
let cursor = db.users.find()
cursor.forEach(doc => print(doc.nom))
```

### Fonctions ES6+

```javascript
// Arrow functions
db.users.find().toArray().map(u => u.nom)

// Template strings
let nom = "Alice"
print(`Bienvenue ${nom}`)

// Destructuring
let { nom, age } = db.users.findOne()
print(nom, age)

// Spread operator
let doc = { nom: "Bob", age: 30 }
let newDoc = { ...doc, ville: "Paris" }
db.users.insertOne(newDoc)

// Async/await (avec précaution)
async function getData() {
  let data = await db.users.find().toArray()
  return data
}
```

---

## Helpers (Raccourcis)

### Qu'est-ce qu'un Helper ?

Les **helpers** sont des raccourcis pour les commandes courantes.

### Helpers de Base de Données

```javascript
// Statistiques de la base
db.stats()

// Version du serveur
db.version()

// Nom de la base
db.getName()

// Informations du serveur
db.serverStatus()

// Supprimer la base
db.dropDatabase()

// Créer une collection
db.createCollection("users")

// Liste des collections
db.getCollectionNames()
```

### Helpers de Collection

```javascript
// Statistiques de collection
db.users.stats()

// Nombre de documents
db.users.count()            // Déprécié
db.users.countDocuments()   // Recommandé

// Supprimer une collection
db.users.drop()

// Renommer une collection
db.users.renameCollection("utilisateurs")

// Créer un index
db.users.createIndex({ email: 1 })

// Lister les index
db.users.getIndexes()

// Supprimer un index
db.users.dropIndex("email_1")
```

### Helpers d'Administration

```javascript
// Informations système
db.hostInfo()

// État de réplication
rs.status()

// Configuration du replica set
rs.conf()

// État du sharding
sh.status()

// Opérations en cours
db.currentOp()

// Tuer une opération
db.killOp(opid)
```

---

## Autocomplétion et Historique

### Autocomplétion (Tab)

mongosh offre une autocomplétion intelligente :

```javascript
// Taper "db." puis Tab
db.[Tab]
// Affiche : find, insert, update, delete, stats, help, etc.

// Taper "db.users." puis Tab
db.users.[Tab]
// Affiche : find, findOne, insertOne, updateOne, etc.

// Complétion des méthodes
db.users.find[Tab]
// Affiche : find, findOne, findAndModify, etc.

// Complétion des noms de bases
use blo[Tab]
// Complète automatiquement : use blog
```

### Historique des Commandes

```javascript
// Flèche Haut/Bas : naviguer dans l'historique
↑  // Commande précédente
↓  // Commande suivante

// Ctrl+R : recherche dans l'historique (reverse search)
// Tapez quelques caractères et mongosh trouve les commandes correspondantes

// Visualiser l'historique
.history

// Effacer l'historique
.clearhistory
```

### Édition de Ligne

```javascript
// Raccourcis clavier
Ctrl+A  // Début de ligne
Ctrl+E  // Fin de ligne
Ctrl+K  // Couper jusqu'à la fin
Ctrl+U  // Couper jusqu'au début
Ctrl+W  // Supprimer le mot précédent
Ctrl+L  // Effacer l'écran
```

---

## Configuration de mongosh

### Fichier de Configuration

mongosh peut être configuré via `~/.mongoshrc.js` :

**Emplacement :**
- Linux/Mac : `~/.mongoshrc.js`
- Windows : `%USERPROFILE%\.mongoshrc.js`

**Exemple de configuration :**

```javascript
// ~/.mongoshrc.js

// Message de bienvenue personnalisé
print("🚀 Bienvenue dans MongoDB !")

// Prompt personnalisé
prompt = function() {
  return db + "(" + hostname() + ")> "
}

// Helpers personnalisés
function countAll() {
  let collections = db.getCollectionNames()
  collections.forEach(coll => {
    print(coll + ": " + db[coll].countDocuments())
  })
}

// Variables globales utiles
global.pretty = true  // Always pretty print

// Raccourcis
var cu = db.currentOp()
var ss = db.serverStatus()

print("Configuration chargée avec succès !")
```

### Options de Configuration

```javascript
// Dans mongosh ou .mongoshrc.js

// Désactiver les warnings
config.set("displayBatchSize", 50)

// Changer la taille du batch
config.set("maxTimeMS", 5000)

// Afficher les requêtes lentes
config.set("showLogs", true)
```

### Variables d'Environnement

```bash
# Désactiver la télémétrie
export MONGOSH_TELEMETRY_ANONYMOUS=false

# Désactiver les updates checks
export MONGOSH_SKIP_VERSION_CHECK=true

# Chemin de l'historique personnalisé
export MONGOSH_HISTORY=/chemin/vers/historique
```

---

## Scripts JavaScript

### Exécuter un Script

**Créer un fichier `script.js` :**

```javascript
// script.js
print("=== Rapport MongoDB ===")

// Lister les bases
print("\nBases de données :")
db.adminCommand('listDatabases').databases.forEach(db => {
  print("  - " + db.name + " (" + (db.sizeOnDisk / 1024 / 1024).toFixed(2) + " Mo)")
})

// Statistiques de la base courante
print("\nBase courante : " + db.getName())
print("Collections : " + db.getCollectionNames().length)

// Compter les documents
db.getCollectionNames().forEach(coll => {
  print("  " + coll + ": " + db[coll].countDocuments() + " documents")
})
```

**Exécuter le script :**

```bash
# Depuis la ligne de commande
mongosh localhost:27017/mabase script.js

# Ou dans mongosh
load("script.js")

# Avec --eval
mongosh --eval "load('script.js')"
```

### Script avec Arguments

```javascript
// backup.js
// Utilisation : mongosh backup.js --eval "var dbName='mabase'"

if (typeof dbName === 'undefined') {
  print("Erreur : Veuillez spécifier dbName")
  quit(1)
}

use(dbName)

print("Backup de la base : " + dbName)
db.getCollectionNames().forEach(coll => {
  print("  Export de " + coll)
  // Logique de backup ici
})

print("Backup terminé !")
```

**Exécution :**
```bash
mongosh --eval "var dbName='blog'" backup.js
```

### Script d'Initialisation

```javascript
// init-db.js
use blog

// Supprimer les collections existantes
db.articles.drop()
db.auteurs.drop()
db.categories.drop()

// Créer les collections avec validation
db.createCollection("articles", {
  validator: {
    $jsonSchema: {
      required: ["titre", "contenu", "auteurId"],
      properties: {
        titre: { bsonType: "string" },
        contenu: { bsonType: "string" },
        auteurId: { bsonType: "objectId" }
      }
    }
  }
})

// Insérer des données de test
db.auteurs.insertMany([
  { nom: "Alice", email: "alice@example.com" },
  { nom: "Bob", email: "bob@example.com" }
])

let alice = db.auteurs.findOne({ nom: "Alice" })

db.articles.insertMany([
  {
    titre: "Introduction à MongoDB",
    contenu: "MongoDB est une base de données NoSQL...",
    auteurId: alice._id,
    publie: true
  },
  {
    titre: "Guide du débutant",
    contenu: "Commencez avec MongoDB...",
    auteurId: alice._id,
    publie: false
  }
])

// Créer les index
db.articles.createIndex({ titre: "text" })
db.articles.createIndex({ auteurId: 1 })
db.auteurs.createIndex({ email: 1 }, { unique: true })

print("Base de données initialisée avec succès !")
print("Auteurs : " + db.auteurs.countDocuments())
print("Articles : " + db.articles.countDocuments())
```

---

## Fonctionnalités Avancées

### Mode Snippet

mongosh supporte des snippets (extraits de code réutilisables) :

```javascript
// Installer un snippet
snippet install <package-name>

// Exemples de snippets utiles
snippet install mongosh-snippets-analyze  // Analyse de base
snippet install mongosh-snippets-export   // Export de données

// Utiliser un snippet
snippet <nom-du-snippet>
```

### Mode Verbose

```javascript
// Activer le mode verbose
db.setLogLevel(1)

// Voir les requêtes exécutées
db.system.profile.find().pretty()
```

### Mesure de Performance

```javascript
// Mesurer le temps d'exécution
let start = new Date()
db.users.find({ age: { $gte: 18 } }).toArray()
let end = new Date()
print("Temps : " + (end - start) + " ms")

// Ou utiliser explain()
db.users.find({ age: { $gte: 18 } }).explain("executionStats")
```

### Sessions et Transactions

```javascript
// Démarrer une session
const session = db.getMongo().startSession()

// Démarrer une transaction
session.startTransaction()

try {
  // Opérations
  db.comptes.updateOne(
    { _id: 1 },
    { $inc: { solde: -100 } },
    { session }
  )

  db.comptes.updateOne(
    { _id: 2 },
    { $inc: { solde: 100 } },
    { session }
  )

  // Commit
  session.commitTransaction()
  print("Transaction réussie")
} catch (error) {
  // Rollback
  session.abortTransaction()
  print("Transaction annulée : " + error)
} finally {
  session.endSession()
}
```

---

## Débogage et Dépannage

### Afficher les Logs

```javascript
// Voir les logs récents
db.adminCommand({ getLog: "global" }).log.forEach(print)

// Logs d'une catégorie spécifique
db.adminCommand({ getLog: "startupWarnings" })
```

### Profiling

```javascript
// Activer le profiler (niveau 2 = toutes les opérations)
db.setProfilingLevel(2)

// Voir les opérations profilées
db.system.profile.find().limit(5).pretty()

// Désactiver le profiler
db.setProfilingLevel(0)

// Profiler uniquement les opérations lentes (> 100ms)
db.setProfilingLevel(1, { slowms: 100 })
```

### Opérations en Cours

```javascript
// Voir les opérations actives
db.currentOp()

// Filtrer les opérations longues
db.currentOp({ "secs_running": { $gte: 5 } })

// Tuer une opération
db.killOp(12345)  // Remplacer par l'opid réel
```

### Messages d'Erreur

```javascript
// Capturer les erreurs
try {
  db.users.insertOne({ email: "duplicate@example.com" })
} catch (e) {
  print("Erreur : " + e.message)
  print("Code : " + e.code)
}

// Vérifier la dernière erreur
db.getLastError()
```

---

## Astuces et Raccourcis

### Raccourcis Pratiques

```javascript
// it : iterate through cursor (afficher plus de résultats)
db.users.find()
it  // Affiche les 20 prochains résultats

// _ : dernier résultat
db.users.findOne()
print(_._id)  // Affiche l'_id du document

// DBQuery.shellBatchSize : changer le nombre de docs affichés
DBQuery.shellBatchSize = 50  // Affiche 50 docs au lieu de 20
```

### Commandes Courtes

```javascript
// Compter rapidement
db.users.count()

// Trouver et afficher proprement
db.users.find().pretty()

// Projection rapide
db.users.find({}, { nom: 1, email: 1 })

// Tri rapide
db.users.find().sort({ age: -1 })

// Limite et tri
db.users.find().sort({ age: -1 }).limit(5)
```

### Helpers Personnalisés

```javascript
// Dans .mongoshrc.js

// Helper pour compter toutes les collections
function countAllCollections() {
  let result = {}
  db.getCollectionNames().forEach(coll => {
    result[coll] = db[coll].countDocuments()
  })
  return result
}

// Helper pour trouver les plus gros documents
function findLargestDocs(collection, limit = 5) {
  return db[collection].find()
    .sort({ $natural: -1 })
    .limit(limit)
    .toArray()
    .map(doc => ({
      id: doc._id,
      size: Object.bsonsize(doc)
    }))
    .sort((a, b) => b.size - a.size)
}

// Utilisation
countAllCollections()
findLargestDocs("users")
```

---

## Différences avec les Drivers

### mongosh vs Driver Node.js

**Dans mongosh :**
```javascript
db.users.find({ age: { $gte: 18 } })
```

**Dans Node.js :**
```javascript
const users = await db.collection('users').find({ age: { $gte: 18 } }).toArray()
```

**Différences principales :**
- mongosh : synchrone et interactif
- Drivers : asynchrones (Promises/async-await)
- mongosh : pour tests et administration
- Drivers : pour applications en production

---

## Bonnes Pratiques

### ✅ À Faire

1. **Utiliser des variables pour les requêtes complexes**
   ```javascript
   // ✅ Bon : lisible et réutilisable
   let filtre = { age: { $gte: 18, $lte: 65 } }
   let projection = { nom: 1, email: 1 }
   db.users.find(filtre, projection)
   ```

2. **Toujours tester avant de modifier**
   ```javascript
   // ✅ D'abord compter
   db.users.countDocuments({ actif: false })

   // Puis supprimer
   db.users.deleteMany({ actif: false })
   ```

3. **Utiliser .explain() pour optimiser**
   ```javascript
   db.users.find({ email: "test@example.com" }).explain("executionStats")
   ```

4. **Sauvegarder vos scripts utiles**
   ```javascript
   // Créer un dossier de scripts
   // ~/mongodb-scripts/
   ```

5. **Utiliser l'historique**
   ```javascript
   // Profitez de Ctrl+R pour rechercher dans l'historique
   ```

### ❌ À Éviter

1. **Requêtes sans filtre sur de grosses collections**
   ```javascript
   // ❌ Dangereux sur une grosse collection
   db.users.find()  // Peut retourner des millions de docs

   // ✅ Bon : limiter
   db.users.find().limit(20)
   ```

2. **Modifier en production sans vérification**
   ```javascript
   // ❌ Dangereux
   db.users.deleteMany({})  // Supprime TOUT !

   // ✅ Bon : vérifier d'abord
   db.users.countDocuments({})
   db.users.find({}).limit(5)
   // Puis confirmer avant de supprimer
   ```

3. **Oublier de fermer les curseurs**
   ```javascript
   // ❌ Mauvais
   let cursor = db.users.find()
   // ... faire autre chose et oublier le curseur

   // ✅ Bon : utiliser toArray() ou forEach()
   db.users.find().toArray()
   ```

4. **Exécuter des opérations coûteuses aux heures de pointe**
   ```javascript
   // ❌ Évitez les analyses complètes de collection en prod
   db.users.find().forEach(doc => {
     // Traitement lourd
   })
   ```

---

## Commandes Récapitulatives

### Navigation et Exploration

| Commande | Description |
|----------|-------------|
| `show dbs` | Lister les bases |
| `use <db>` | Sélectionner une base |
| `show collections` | Lister les collections |
| `db` | Base courante |
| `help` | Aide générale |

### Manipulation de Données

| Commande | Description |
|----------|-------------|
| `db.collection.find()` | Rechercher |
| `db.collection.insertOne()` | Insérer un document |
| `db.collection.updateOne()` | Modifier un document |
| `db.collection.deleteOne()` | Supprimer un document |
| `db.collection.countDocuments()` | Compter |

### Administration

| Commande | Description |
|----------|-------------|
| `db.stats()` | Stats de la base |
| `db.collection.stats()` | Stats d'une collection |
| `db.serverStatus()` | État du serveur |
| `db.currentOp()` | Opérations en cours |
| `rs.status()` | État réplication |

### Utilitaires

| Commande | Description |
|----------|-------------|
| `exit` | Quitter mongosh |
| `cls` / `clear` | Effacer l'écran |
| `load("script.js")` | Charger un script |
| `it` | Afficher plus de résultats |
| `version()` | Version MongoDB |

---

## Points Clés à Retenir

### ✅ Essentiel

1. **mongosh = JavaScript** : Vous pouvez utiliser tout JavaScript moderne
2. **Interactif** : Testez vos requêtes avant de les mettre en prod
3. **Helpers** : Utilisez les raccourcis (show, db, it, etc.)
4. **Autocomplétion** : Tab est votre ami
5. **Scripts** : Automatisez avec des fichiers .js
6. **Configuration** : Personnalisez avec .mongoshrc.js

### 🎯 Workflow Standard

```javascript
// 1. Se connecter
mongosh

// 2. Sélectionner la base
use mabase

// 3. Explorer
show collections
db.users.countDocuments()

// 4. Tester une requête
db.users.find({ age: { $gte: 18 } }).limit(5)

// 5. Exécuter si OK
db.users.find({ age: { $gte: 18 } })
```

### ⚠️ Précautions

- Toujours vérifier la base courante avant de modifier
- Utiliser `.limit()` lors de l'exploration
- Sauvegarder avant les opérations destructives
- Tester en dev avant prod

---

## Prochaines Étapes

Maintenant que vous maîtrisez mongosh, explorez l'interface graphique :

➡️ **2.7 Introduction à MongoDB Compass** : Interface graphique pour MongoDB

Le shell MongoDB est votre compagnon quotidien pour gérer et interroger vos bases de données !

---

## Ressources Complémentaires

### Documentation Officielle

- [mongosh Documentation](https://docs.mongodb.com/mongodb-shell/)
- [mongosh API Reference](https://docs.mongodb.com/manual/reference/method/)
- [JavaScript in MongoDB](https://docs.mongodb.com/manual/tutorial/write-scripts-for-the-mongo-shell/)

### Tutoriels et Guides

- [mongosh Quick Reference](https://docs.mongodb.com/mongodb-shell/reference/)
- [Shell Scripting](https://docs.mongodb.com/manual/tutorial/write-scripts-for-the-mongo-shell/)

### Outils Complémentaires

- MongoDB Compass (interface graphique)
- VS Code avec extension MongoDB
- Studio 3T (client tiers)

---


⏭️ [Introduction à MongoDB Compass](/02-fondamentaux-de-mongodb/07-introduction-compass.md)
