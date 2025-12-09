🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.4 - Helpers et Raccourcis

## Table des matières

1. [Raccourcis Clavier](#raccourcis-clavier)
2. [Helpers de Navigation](#helpers-de-navigation)
3. [Helpers de Formatage](#helpers-de-formatage)
4. [Variables Globales Utiles](#variables-globales-utiles)
5. [Fonctions Utilitaires](#fonctions-utilitaires)
6. [Configuration du Shell](#configuration-du-shell)
7. [Snippets Réutilisables](#snippets-r%C3%A9utilisables)
8. [Commandes Système](#commandes-syst%C3%A8me)
9. [Import/Export Rapide](#importexport-rapide)
10. [Astuces de Productivité](#astuces-de-productivit%C3%A9)

---

## Raccourcis Clavier

### Édition de ligne

| Raccourci | Action | Équivalent |
|-----------|--------|------------|
| `Ctrl+A` | Début de ligne | `Home` |
| `Ctrl+E` | Fin de ligne | `End` |
| `Ctrl+K` | Supprimer jusqu'à la fin | - |
| `Ctrl+U` | Supprimer toute la ligne | - |
| `Ctrl+W` | Supprimer le mot précédent | - |
| `Alt+D` | Supprimer le mot suivant | - |
| `Ctrl+Y` | Coller le dernier texte supprimé | - |
| `Ctrl+T` | Inverser deux caractères | - |
| `Alt+T` | Inverser deux mots | - |

---

### Navigation

| Raccourci | Action |
|-----------|--------|
| `↑` / `↓` | Historique précédent/suivant |
| `Ctrl+R` | Recherche inverse dans l'historique |
| `Ctrl+S` | Recherche avant dans l'historique |
| `Ctrl+G` | Annuler la recherche |
| `Alt+B` | Reculer d'un mot |
| `Alt+F` | Avancer d'un mot |

---

### Contrôle

| Raccourci | Action |
|-----------|--------|
| `Ctrl+C` | Annuler la commande en cours |
| `Ctrl+D` | Quitter mongosh (ou EOF) |
| `Ctrl+L` | Effacer l'écran |
| `Ctrl+Z` | Suspendre mongosh (Linux/Mac) |
| `Tab` | Auto-complétion |
| `Tab Tab` | Liste des possibilités |

---

### Auto-complétion intelligente

```javascript
// Complétion des méthodes
db.users.find<Tab>
// → findOne, findOneAndDelete, findOneAndReplace, findOneAndUpdate

// Complétion des champs (après avoir fait une requête)
db.users.find({na<Tab>
// → name

// Complétion des collections
db.use<Tab>
// → users

// Complétion des bases de données
use my<Tab>
// → myapp
```

💡 **Astuce** : Tab Tab affiche toutes les possibilités si l'auto-complétion est ambiguë.

---

## Helpers de Navigation

### Raccourcis show

```javascript
// Bases de données
show dbs
show databases

// Collections
show collections
show tables

// Utilisateurs
show users

// Rôles
show roles

// Profiler
show profile

// Logs
show logs
show log global
show log startupWarnings
```

---

### Helpers de base de données

```javascript
// Base courante
db

// Nom de la base courante
db.getName()

// Obtenir une autre base sans basculer
db.getSiblingDB("autreBase")

// Basculer de base
use autreBase

// Lister toutes les bases
db.adminCommand('listDatabases')
```

---

### Helpers de collection

```javascript
// Obtenir une collection
db.users
db.getCollection("users")
db["users"]

// Lister les collections
db.getCollectionNames()

// Collection courante (dans le contexte)
db.getCollectionInfos()
```

---

### Navigation rapide

```javascript
// Chaînage de contextes
db.getSiblingDB("logs").events.find().limit(5)

// Assigner à des variables
const users = db.users
const orders = db.orders

users.findOne()
orders.countDocuments()
```

---

## Helpers de Formatage

### pretty()

```javascript
// Formater le JSON (moins nécessaire dans mongosh)
db.users.findOne().pretty()

// Équivalent moderne (mongosh formate automatiquement)
db.users.findOne()
```

💡 **Note** : mongosh formate automatiquement, `pretty()` est conservé pour compatibilité.

---

### printjson()

```javascript
// Afficher un objet JSON formaté
const user = db.users.findOne()
printjson(user)

// Équivalent
print(JSON.stringify(user, null, 2))
```

---

### print()

```javascript
// Afficher du texte
print("Hello MongoDB")

// Afficher une variable
const count = db.users.countDocuments()
print("Total users: " + count)

// Interpolation moderne
print(`Total users: ${count}`)
```

---

### tojson() et tojsonObject()

```javascript
// Convertir en chaîne JSON
const user = db.users.findOne()
const json = tojson(user)
print(json)

// Avec indentation
const jsonPretty = tojson(user, null, true)
```

---

### Formatage de curseur

```javascript
// Batch size personnalisé
DBQuery.shellBatchSize = 50

// Affichage compact
db.users.find().toArray()

// Comptage rapide
db.users.find().count()  // Déprécié
db.users.countDocuments()  // Recommandé
```

---

## Variables Globales Utiles

### Variables de configuration

```javascript
// Taille du batch d'affichage
DBQuery.shellBatchSize
DBQuery.shellBatchSize = 10

// Timeout de réseau (millisecondes)
// Note: à configurer dans la connection string

// Prompt personnalisé
prompt = function() {
  return db + "> ";
}

// Prompt avec timestamp
prompt = function() {
  return new Date().toISOString() + " " + db + "> ";
}
```

---

### Variables système

```javascript
// Version du shell
version()

// Connexion courante
db.getMongo()

// Nom d'hôte
hostname()

// Répertoire de données (si accessible)
// Nécessite privilèges admin
db.serverCmdLineOpts()
```

---

### Variables de session

```javascript
// Créer des variables pour un usage fréquent
const users = db.users
const orders = db.orders

// Sauvegarder des résultats
const activeUsers = users.find({ active: true }).toArray()
print(`Found ${activeUsers.length} active users`)

// Résultats de requêtes complexes
const stats = db.users.aggregate([
  { $group: { _id: "$country", count: { $sum: 1 } } }
]).toArray()
```

---

## Fonctions Utilitaires

### Fonctions de comptage rapide

```javascript
// Compter tous les documents de toutes les collections
function countAll() {
  db.getCollectionNames().forEach(function(col) {
    print(col + ": " + db[col].countDocuments());
  });
}

countAll()
```

---

### Fonctions de taille

```javascript
// Taille de toutes les collections
function sizeAll() {
  db.getCollectionNames().forEach(function(col) {
    const stats = db[col].stats(1024 * 1024);
    print(`${col}: ${stats.size.toFixed(2)} MB`);
  });
}

sizeAll()
```

---

### Fonction de recherche globale

```javascript
// Rechercher dans toutes les collections
function searchAll(field, value) {
  db.getCollectionNames().forEach(function(col) {
    const query = {};
    query[field] = value;
    const count = db[col].countDocuments(query);
    if (count > 0) {
      print(`${col}: ${count} documents`);
    }
  });
}

searchAll("email", "test@example.com")
```

---

### Fonction de backup rapide

```javascript
// Exporter une collection en JSON
function backupCollection(collectionName) {
  const docs = db[collectionName].find().toArray();
  print(JSON.stringify(docs, null, 2));
}

// Usage:
// backupCollection("users") > users_backup.json
```

---

### Fonction d'analyse de schéma

```javascript
// Analyser les types de champs d'une collection
function analyzeSchema(collectionName, sampleSize = 100) {
  const sample = db[collectionName].find().limit(sampleSize).toArray();
  const schema = {};

  sample.forEach(function(doc) {
    Object.keys(doc).forEach(function(key) {
      const type = typeof doc[key];
      if (!schema[key]) {
        schema[key] = {};
      }
      schema[key][type] = (schema[key][type] || 0) + 1;
    });
  });

  printjson(schema);
}

analyzeSchema("users")
```

---

### Fonction de nettoyage

```javascript
// Supprimer les champs null
function removeNullFields(collectionName) {
  const result = db[collectionName].updateMany(
    {},
    { $unset: { fieldName: "" } }
  );
  print(`Modified ${result.modifiedCount} documents`);
}
```

---

## Configuration du Shell

### Fichier .mongoshrc.js

**Emplacement** : `~/.mongoshrc.js` (Linux/Mac) ou `%HOME%\.mongoshrc.js` (Windows)

```javascript
// ~/.mongoshrc.js - Configuration personnalisée

// Prompt personnalisé
prompt = function() {
  const db = db.getName();
  const host = db.getMongo().host;
  return `[${host}] ${db}> `;
}

// Fonctions utilitaires
function ll() {
  db.getCollectionNames().forEach(function(col) {
    const count = db[col].estimatedDocumentCount();
    print(`${col}: ${count} documents`);
  });
}

function size() {
  const stats = db.stats(1024 * 1024);
  print(`Database: ${stats.db}`);
  print(`Size: ${stats.dataSize.toFixed(2)} MB`);
  print(`Collections: ${stats.collections}`);
}

function indexes() {
  db.getCollectionNames().forEach(function(col) {
    print(`\n=== ${col} ===`);
    db[col].getIndexes().forEach(function(idx) {
      print(`  ${idx.name}: ${JSON.stringify(idx.key)}`);
    });
  });
}

// Helpers de connexion
function useProd() {
  db = connect("mongodb://prod-server:27017/myapp");
  print("Connected to PRODUCTION");
}

function useDev() {
  db = connect("mongodb://localhost:27017/myapp");
  print("Connected to DEVELOPMENT");
}

// Message de bienvenue
print("\n=== Custom MongoDB Shell ===");
print("Available commands:");
print("  ll()      - List collections with counts");
print("  size()    - Database size info");
print("  indexes() - List all indexes");
print("  useProd() - Connect to production");
print("  useDev()  - Connect to development");
print("=============================\n");

// Batch size par défaut
DBQuery.shellBatchSize = 20;
```

---

### Variables d'environnement

```bash
# Désactiver la télémétrie
export MONGOSH_NO_TELEMETRY=1

# Définir l'éditeur
export EDITOR=vim
export VISUAL=code

# Fichier d'historique personnalisé
export MONGOSH_HISTORY_FILE=/path/to/custom/history

# Taille maximale de l'historique
export MONGOSH_HISTORY_SIZE=10000
```

---

## Snippets Réutilisables

### Snippet de connexion

```javascript
// connect.js
const config = {
  dev: "mongodb://localhost:27017",
  prod: "mongodb://prod-server:27017",
  atlas: "mongodb+srv://user:pass@cluster.mongodb.net"
};

function connectTo(env) {
  if (!config[env]) {
    print("Environment not found. Available: " + Object.keys(config));
    return;
  }
  db = connect(config[env]);
  print(`Connected to ${env.toUpperCase()}`);
}

// Usage: load("connect.js"); connectTo("dev");
```

---

### Snippet d'analyse

```javascript
// analyze.js
function analyzeDatabase() {
  print("=== Database Analysis ===\n");

  const dbStats = db.stats(1024 * 1024);
  print(`Database: ${dbStats.db}`);
  print(`Size: ${dbStats.dataSize.toFixed(2)} MB`);
  print(`Collections: ${dbStats.collections}`);
  print(`Indexes: ${dbStats.indexes}`);
  print(`Index Size: ${dbStats.indexSize.toFixed(2)} MB\n`);

  print("=== Collections ===");
  db.getCollectionNames().forEach(function(col) {
    const stats = db[col].stats(1024 * 1024);
    print(`${col}:`);
    print(`  Documents: ${stats.count}`);
    print(`  Size: ${stats.size.toFixed(2)} MB`);
    print(`  Indexes: ${stats.nindexes}`);
  });
}

// Usage: load("analyze.js"); analyzeDatabase();
```

---

### Snippet de migration

```javascript
// migrate.js
function addFieldToAll(collectionName, fieldName, defaultValue) {
  const result = db[collectionName].updateMany(
    { [fieldName]: { $exists: false } },
    { $set: { [fieldName]: defaultValue } }
  );
  print(`Updated ${result.modifiedCount} documents`);
}

function renameField(collectionName, oldName, newName) {
  const result = db[collectionName].updateMany(
    {},
    { $rename: { [oldName]: newName } }
  );
  print(`Renamed field in ${result.modifiedCount} documents`);
}

function removeField(collectionName, fieldName) {
  const result = db[collectionName].updateMany(
    {},
    { $unset: { [fieldName]: "" } }
  );
  print(`Removed field from ${result.modifiedCount} documents`);
}
```

---

### Snippet de monitoring

```javascript
// monitor.js
function monitor() {
  cls;
  print("=== MongoDB Monitor ===");
  print(new Date().toISOString());

  const status = db.serverStatus();
  print("\nConnections: " + status.connections.current);
  print("Memory (Resident): " + status.mem.resident + " MB");
  print("Operations:");
  printjson(status.opcounters);

  print("\n=== Active Operations ===");
  const ops = db.currentOp({ "$all": false });
  ops.inprog.forEach(function(op) {
    if (op.secs_running > 1) {
      print(`${op.opid}: ${op.op} on ${op.ns} (${op.secs_running}s)`);
    }
  });
}

// Usage: load("monitor.js"); setInterval(monitor, 5000);
```

---

## Commandes Système

### Exécuter des commandes shell

```javascript
// Lister des fichiers (Linux/Mac)
// Note: mongosh n'a pas accès direct au shell système

// Alternative: utiliser les commandes MongoDB natives
db.adminCommand({ listDatabases: 1 })
```

💡 **Note** : mongosh n'a pas de commandes shell intégrées comme l'ancien mongo shell.

---

### Charger des fichiers JavaScript

```javascript
// Charger un script
load("script.js")
load("/path/to/script.js")

// Charger un module
// Note: support limité, préférer les fonctions dans .mongoshrc.js
```

---

### Éditer dans l'éditeur externe

```javascript
// Éditer une variable ou fonction
edit functionName

// Définit la variable EDITOR
// Linux/Mac: export EDITOR=vim
// Windows: set EDITOR=notepad
```

---

## Import/Export Rapide

### Import JSON dans une collection

```javascript
// Lire un fichier et insérer (nécessite fs si disponible)
// Alternative: utiliser mongoimport en ligne de commande

// mongoimport --db myapp --collection users --file users.json --jsonArray
```

---

### Export depuis mongosh

```javascript
// Exporter en JSON (copier-coller la sortie)
const data = db.users.find().toArray();
printjson(data);

// Exporter avec formatage
db.users.find().forEach(function(doc) {
  print(JSON.stringify(doc));
});

// Alternative: utiliser mongoexport en ligne de commande
// mongoexport --db myapp --collection users --out users.json
```

---

### Copie rapide de collection

```javascript
// Copier une collection
db.users.find().forEach(function(doc) {
  db.users_backup.insertOne(doc);
});

// Copier avec filtre
db.users.find({ active: true }).forEach(function(doc) {
  db.active_users.insertOne(doc);
});

// Méthode agrégation (plus efficace)
db.users.aggregate([
  { $match: { active: true } },
  { $out: "active_users" }
]);
```

---

## Astuces de Productivité

### 1. Alias de commandes fréquentes

```javascript
// Dans .mongoshrc.js
const u = db.users;
const o = db.orders;
const p = db.products;

// Usage direct
u.findOne()
o.countDocuments()
```

---

### 2. Fonctions de debug

```javascript
// Fonction pour logger avec timestamp
function log(message) {
  print(`[${new Date().toISOString()}] ${message}`);
}

// Fonction pour mesurer le temps d'exécution
function time(fn) {
  const start = Date.now();
  const result = fn();
  const duration = Date.now() - start;
  print(`Execution time: ${duration}ms`);
  return result;
}

// Usage
time(() => db.users.find({ age: { $gt: 25 } }).count())
```

---

### 3. Requêtes réutilisables

```javascript
// Sauvegarder des requêtes complexes
const activeUsersQuery = { active: true, lastLogin: { $gte: new Date("2024-01-01") } };
const recentOrdersQuery = { createdAt: { $gte: new Date(Date.now() - 86400000) } };

// Réutiliser
db.users.find(activeUsersQuery)
db.orders.find(recentOrdersQuery)
```

---

### 4. Pipeline d'agrégation réutilisable

```javascript
// Définir des stages réutilisables
const matchActive = { $match: { active: true } };
const sortByDate = { $sort: { createdAt: -1 } };
const limitTen = { $limit: 10 };

const groupByCountry = {
  $group: {
    _id: "$country",
    count: { $sum: 1 }
  }
};

// Composer des pipelines
db.users.aggregate([matchActive, groupByCountry, sortByDate])
db.users.aggregate([matchActive, sortByDate, limitTen])
```

---

### 5. Helpers de validation

```javascript
// Vérifier si une collection existe
function collectionExists(name) {
  return db.getCollectionNames().includes(name);
}

// Vérifier si un index existe
function indexExists(collectionName, indexName) {
  const indexes = db[collectionName].getIndexes();
  return indexes.some(idx => idx.name === indexName);
}

// Vérifier si un document existe
function documentExists(collectionName, query) {
  return db[collectionName].countDocuments(query) > 0;
}
```

---

### 6. Batch operations helper

```javascript
// Traiter par lots
function processBatch(collectionName, query, batchSize, processFunc) {
  let skip = 0;
  let batch;

  do {
    batch = db[collectionName]
      .find(query)
      .skip(skip)
      .limit(batchSize)
      .toArray();

    batch.forEach(processFunc);
    skip += batchSize;

    print(`Processed ${skip} documents...`);
  } while (batch.length === batchSize);

  print("Done!");
}

// Usage
processBatch("users", { active: false }, 1000, function(user) {
  // Traitement de chaque utilisateur
  db.archived_users.insertOne(user);
});
```

---

### 7. Quick stats

```javascript
// Statistiques rapides multi-collections
function quickStats() {
  const collections = db.getCollectionNames();
  const stats = [];

  collections.forEach(function(col) {
    const count = db[col].estimatedDocumentCount();
    const size = db[col].stats(1024 * 1024).size;
    stats.push({ collection: col, count: count, sizeMB: size.toFixed(2) });
  });

  stats.sort((a, b) => b.count - a.count);
  printjson(stats);
}
```

---

### 8. Connection helper avec retry

```javascript
// Connexion avec retry
function connectWithRetry(uri, maxRetries = 3) {
  let attempts = 0;

  while (attempts < maxRetries) {
    try {
      db = connect(uri);
      print("Connected successfully");
      return db;
    } catch (e) {
      attempts++;
      print(`Connection failed (attempt ${attempts}/${maxRetries})`);
      if (attempts >= maxRetries) {
        throw e;
      }
      sleep(1000);
    }
  }
}
```

---

### 9. Pretty print pour objets complexes

```javascript
// Affichage amélioré
function pp(obj) {
  print(JSON.stringify(obj, null, 2));
}

// Usage
const result = db.users.aggregate([...]).toArray();
pp(result);
```

---

### 10. Historique de commandes

```javascript
// Sauvegarder une commande pour référence future
// L'historique est automatique, mais on peut l'exporter

// Linux/Mac: cat ~/.mongosh/.mongosh_repl_history
// Windows: type %USERPROFILE%\.mongosh\.mongosh_repl_history

// Recherche dans l'historique: Ctrl+R
```

---

## Raccourcis d'édition avancés

### Multi-ligne

```javascript
// Édition multi-ligne naturelle
db.users.aggregate([
  { $match: { active: true } },
  { $group: {
      _id: "$country",
      count: { $sum: 1 }
  } },
  { $sort: { count: -1 } }
])

// Continuer sur nouvelle ligne avec \
const longQuery = { \
  name: "Alice", \
  age: 30, \
  active: true \
}
```

---

### Copier-coller

```javascript
// Copier depuis l'éditeur
// Sélectionner le texte et Ctrl+C

// Coller dans mongosh
// Ctrl+V ou Shift+Insert (selon terminal)

// Coller du code formaté
// mongosh préserve l'indentation
```

---

## Templates de requêtes

### Template CRUD

```javascript
// CREATE template
const newDoc = {
  field1: "value1",
  field2: "value2",
  createdAt: new Date()
};
db.collection.insertOne(newDoc);

// READ template
const query = { field: "value" };
const projection = { field1: 1, field2: 1, _id: 0 };
db.collection.find(query, projection).limit(10);

// UPDATE template
const filter = { _id: ObjectId("...") };
const update = { $set: { field: "newValue" } };
db.collection.updateOne(filter, update);

// DELETE template
const deleteFilter = { field: "value" };
db.collection.deleteMany(deleteFilter);
```

---

### Template Aggregation

```javascript
// Template d'agrégation
db.collection.aggregate([
  // Stage 1: Filtrer
  { $match: { status: "active" } },

  // Stage 2: Projeter
  { $project: {
      name: 1,
      total: { $sum: "$amounts" }
  } },

  // Stage 3: Grouper
  { $group: {
      _id: "$category",
      count: { $sum: 1 },
      avg: { $avg: "$total" }
  } },

  // Stage 4: Trier
  { $sort: { count: -1 } },

  // Stage 5: Limiter
  { $limit: 10 }
]);
```

---

## Tableau récapitulatif des raccourcis

### Raccourcis essentiels

| Catégorie | Raccourci | Action |
|-----------|-----------|--------|
| **Édition** | `Ctrl+A` | Début de ligne |
| | `Ctrl+E` | Fin de ligne |
| | `Ctrl+K` | Supprimer fin de ligne |
| | `Ctrl+U` | Supprimer toute la ligne |
| **Navigation** | `↑` / `↓` | Historique |
| | `Ctrl+R` | Recherche historique |
| | `Tab` | Auto-complétion |
| **Contrôle** | `Ctrl+C` | Annuler |
| | `Ctrl+D` | Quitter |
| | `Ctrl+L` | Effacer écran |
| **Helpers** | `show dbs` | Lister bases |
| | `show collections` | Lister collections |
| | `db` | Base courante |
| | `it` | Continuer résultats |

---

## Checklist de productivité

### ✅ Configuration initiale

```javascript
// 1. Créer ~/.mongoshrc.js avec fonctions utiles
// 2. Définir prompt personnalisé
// 3. Configurer DBQuery.shellBatchSize
// 4. Ajouter helpers fréquents
// 5. Définir aliases de collections
```

---

### ✅ Bonnes pratiques

```javascript
// ✅ Utiliser des variables pour requêtes complexes
const query = { active: true, age: { $gte: 18 } };

// ✅ Sauvegarder résultats intermédiaires
const results = db.users.find(query).toArray();

// ✅ Commenter le code dans les scripts
// Recherche des utilisateurs actifs majeurs

// ✅ Utiliser des fonctions réutilisables
function findActiveUsers() {
  return db.users.find({ active: true });
}

// ✅ Mesurer les performances
const start = Date.now();
// ... opération
print(`Duration: ${Date.now() - start}ms`);
```

---

### ✅ À éviter

```javascript
// ❌ Requêtes non limitées
db.hugeLogs.find()  // Peut bloquer

// ❌ Variables mal nommées
const x = db.users.find()

// ❌ Modification sans backup
db.users.deleteMany({})  // DANGER

// ❌ Ignorer les erreurs
try {
  db.users.insertOne(doc)
} catch (e) {
  // Ne rien faire
}
```

---

## Exemples de .mongoshrc.js complet

```javascript
// ~/.mongoshrc.js - Configuration complète

// ============================================
// CONFIGURATION
// ============================================

// Prompt personnalisé avec couleurs
prompt = function() {
  const dbName = db.getName();
  const host = db.getMongo().host.split(':')[0];
  return `[\x1b[36m${host}\x1b[0m] \x1b[33m${dbName}\x1b[0m> `;
}

// Batch size
DBQuery.shellBatchSize = 20;

// ============================================
// HELPERS NAVIGATION
// ============================================

function ll() {
  db.getCollectionNames().forEach(function(col) {
    const count = db[col].estimatedDocumentCount();
    print(`${col.padEnd(30)} ${count.toLocaleString()} docs`);
  });
}

function size() {
  const stats = db.stats(1024 * 1024);
  print(`Database: ${stats.db}`);
  print(`Size: ${stats.dataSize.toFixed(2)} MB`);
  print(`Collections: ${stats.collections}`);
  print(`Objects: ${stats.objects.toLocaleString()}`);
}

// ============================================
// HELPERS ANALYSE
// ============================================

function indexes() {
  db.getCollectionNames().forEach(function(col) {
    const indexes = db[col].getIndexes();
    if (indexes.length > 1) {
      print(`\n${col}:`);
      indexes.forEach(function(idx) {
        print(`  ${idx.name}: ${JSON.stringify(idx.key)}`);
      });
    }
  });
}

function slow(threshold = 100) {
  const profile = db.system.profile
    .find({ millis: { $gte: threshold } })
    .sort({ millis: -1 })
    .limit(10)
    .toArray();

  profile.forEach(function(op) {
    print(`${op.millis}ms - ${op.ns} - ${op.op}`);
  });
}

// ============================================
// HELPERS UTILITAIRES
// ============================================

function backup(collectionName) {
  const timestamp = new Date().toISOString().replace(/:/g, '-');
  const backupName = `${collectionName}_backup_${timestamp}`;

  db[collectionName].aggregate([
    { $match: {} },
    { $out: backupName }
  ]);

  print(`Backup created: ${backupName}`);
}

function countAll() {
  let total = 0;
  db.getCollectionNames().forEach(function(col) {
    const count = db[col].estimatedDocumentCount();
    total += count;
  });
  print(`Total documents: ${total.toLocaleString()}`);
}

// ============================================
// HELPERS DEBUG
// ============================================

function time(fn) {
  const start = Date.now();
  const result = fn();
  print(`Execution time: ${Date.now() - start}ms`);
  return result;
}

function log(msg) {
  print(`[${new Date().toISOString()}] ${msg}`);
}

// ============================================
// MESSAGE DE BIENVENUE
// ============================================

print("\n" + "=".repeat(50));
print("  Custom MongoDB Shell Loaded");
print("=".repeat(50));
print("\nAvailable commands:");
print("  ll()          - List collections with counts");
print("  size()        - Database size info");
print("  indexes()     - List all indexes");
print("  slow(ms)      - Show slow queries");
print("  backup(col)   - Backup a collection");
print("  countAll()    - Total document count");
print("  time(fn)      - Time a function");
print("  log(msg)      - Log with timestamp");
print("\n" + "=".repeat(50) + "\n");
```

---

**💡 Conseil final** : Personnalisez votre `.mongoshrc.js` avec vos propres helpers pour maximiser votre productivité quotidienne !

**🚀 Productivité** : Maîtriser ces raccourcis et helpers peut vous faire gagner des heures chaque semaine.

⏭️ [Requêtes MongoDB de Référence](/annexes/requetes-reference/README.md)
