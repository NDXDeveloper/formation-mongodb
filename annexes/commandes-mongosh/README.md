🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe B : Commandes mongosh Essentielles

## Vue d'ensemble

Cette annexe est un **guide de référence rapide** des commandes et méthodes les plus utilisées dans **mongosh** (MongoDB Shell). Elle est conçue pour être consultée pendant vos sessions de travail quotidiennes avec MongoDB.

---

## À propos de mongosh

**mongosh** (MongoDB Shell) est l'interface en ligne de commande interactive officielle de MongoDB qui a remplacé l'ancien `mongo` shell depuis MongoDB 5.0.

### Caractéristiques principales

| Fonctionnalité | Description |
|----------------|-------------|
| 🚀 JavaScript moderne | Support d'ES6+, async/await, promises |
| 🎨 Syntaxe colorée | Améliore la lisibilité des résultats |
| 📝 Auto-complétion | Tab pour compléter commandes et champs |
| 🔄 Historique | Flèches ↑/↓ pour naviguer dans l'historique |
| 📦 Support NPM | Import de packages Node.js |
| 🛠️ Édition avancée | Édition multi-lignes, copier-coller |

---

## Organisation de l'annexe

Cette référence est divisée en quatre sections pratiques :

### B.1 - Navigation
Commandes pour explorer et naviguer dans MongoDB :
- Lister les bases de données
- Basculer entre bases
- Lister les collections
- Obtenir des informations sur la base/collection courante

### B.2 - CRUD rapide
Opérations essentielles Create, Read, Update, Delete :
- Insertion de documents
- Recherche et filtrage
- Mise à jour de données
- Suppression de documents
- Méthodes utilitaires

### B.3 - Administration
Commandes d'administration et monitoring :
- Statuts (serveur, Replica Set, Sharding)
- Gestion des index
- Gestion des utilisateurs
- Statistiques et métriques
- Maintenance

### B.4 - Helpers et raccourcis
Astuces et fonctions helpers pour gagner du temps :
- Raccourcis clavier
- Commandes utilitaires
- Formatage des résultats
- Configuration du shell

---

## Comment utiliser cette référence

### 📖 Format des commandes

Chaque commande est présentée selon ce format :

```javascript
// Description de la commande
db.collection.method(paramètres)
```

**Conventions de notation :**
- `db` : base de données courante
- `collection` : nom de votre collection
- `[optionnel]` : paramètre optionnel
- `<requis>` : paramètre obligatoire
- `...` : paramètres répétables

---

## Connexion à MongoDB

### Connexion locale (défaut)

```bash
mongosh
# Équivalent à: mongosh "mongodb://localhost:27017"
```

### Connexion avec URI complète

```bash
mongosh "mongodb://username:password@host:port/database"
```

### Connexion à Atlas

```bash
mongosh "mongodb+srv://username:password@cluster.mongodb.net/database"
```

### Options de connexion courantes

```bash
# Spécifier base de données
mongosh --host localhost --port 27017 --username admin --password --authenticationDatabase admin

# Exécuter un script
mongosh script.js

# Mode quiet (sans banner)
mongosh --quiet

# Évaluer du code JavaScript
mongosh --eval "db.users.countDocuments()"
```

---

## Aide intégrée dans mongosh

### Commandes d'aide

| Commande | Description |
|----------|-------------|
| `help` | Aide générale sur mongosh |
| `db.help()` | Aide sur les méthodes de base de données |
| `db.collection.help()` | Aide sur les méthodes de collection |
| `show dbs` | Liste des bases disponibles |
| `show collections` | Collections de la base courante |
| `show users` | Utilisateurs de la base courante |
| `show roles` | Rôles de la base courante |
| `show profile` | Dernières opérations profilées |
| `show logs` | Types de logs disponibles |

### Auto-complétion

```javascript
// Appuyez sur Tab pour compléter
db.user<Tab>        // → db.users
db.users.find<Tab>  // → liste des méthodes find*
```

---

## Configuration du shell

### Variables d'environnement

```bash
# Désactiver la télémétrie
export MONGOSH_NO_TELEMETRY=1

# Définir l'éditeur
export EDITOR=vim

# Historique personnalisé
export MONGOSH_HISTORY_FILE=/path/to/custom/history
```

### Fichier de configuration

**Emplacement** : `~/.mongoshrc.js` (ou `%HOME%/.mongoshrc.js` sur Windows)

```javascript
// Exemple de configuration personnalisée
// Ce fichier est exécuté automatiquement au démarrage

// Prompt personnalisé
prompt = function() {
  return db + "> ";
}

// Helpers personnalisés
function countAll() {
  db.getCollectionNames().forEach(function(col) {
    print(col + ": " + db[col].countDocuments());
  });
}

// Connexion automatique à une base
use myDatabase;

print("Configuration personnalisée chargée !");
```

---

## Conventions et bonnes pratiques

### 🎯 Nommage

```javascript
// ✅ Bonnes pratiques
db.users           // camelCase pour collections
db.orders
db.productCatalog

// ❌ À éviter
db["my-collection"]  // traits d'union nécessitent des crochets
db.Users             // PascalCase moins courant
```

### 🎯 Requêtes

```javascript
// ✅ Bien : utiliser des projections
db.users.find({active: true}, {name: 1, email: 1, _id: 0})

// ❌ Éviter : récupérer tous les champs inutilement
db.users.find({active: true})

// ✅ Bien : limiter les résultats
db.logs.find().sort({date: -1}).limit(10)

// ❌ Éviter : curseur sans limite sur grosse collection
db.logs.find()
```

### 🎯 Sécurité

```javascript
// ✅ Bien : utiliser des variables
const email = "user@example.com";
db.users.findOne({email: email})

// ⚠️ Attention : risque d'injection (depuis input non validé)
const userInput = req.body.email; // Non sécurisé si pas validé
db.users.findOne({email: userInput})
```

---

## Formatage des résultats

### Pretty print

```javascript
// Affichage formaté JSON
db.users.find().pretty()

// Note: mongosh formate automatiquement par défaut
// pretty() est moins nécessaire qu'avec l'ancien mongo shell
```

### Limitation des résultats affichés

```javascript
// Par défaut: 20 documents affichés
db.collection.find()

// Modifier le batch size
DBQuery.shellBatchSize = 10

// Itérer manuellement sur le curseur
const cursor = db.collection.find()
cursor.next()  // Document suivant
cursor.hasNext()  // Vérifie s'il reste des documents
```

### Conversion en tableau

```javascript
// Récupérer tous les résultats dans un tableau
const results = db.users.find({active: true}).toArray()

// Compter sans récupérer
db.users.countDocuments({active: true})
```

---

## Gestion des erreurs

### Try-Catch

```javascript
try {
  db.users.insertOne({email: "test@example.com"})
  print("Insertion réussie")
} catch (e) {
  print("Erreur: " + e.message)
}
```

### Vérification d'existence

```javascript
// Vérifier si une base existe
show dbs  // ou
db.adminCommand('listDatabases')

// Vérifier si une collection existe
db.getCollectionNames().includes('users')

// Vérifier si un document existe
if (db.users.countDocuments({email: "test@example.com"}) > 0) {
  print("Utilisateur existe")
}
```

---

## Scripts et automatisation

### Exécuter un script JavaScript

```bash
# Depuis la ligne de commande
mongosh script.js

# Depuis mongosh
load("script.js")
```

### Exemple de script

```javascript
// backup-users.js
const db = connect("mongodb://localhost/myapp");

const users = db.users.find().toArray();
print(`${users.length} utilisateurs trouvés`);

// Écrire dans un fichier (depuis Node.js ou environnement approprié)
// fs.writeFileSync('users-backup.json', JSON.stringify(users, null, 2));

print("Backup terminé");
```

### Script avec arguments

```javascript
// import-data.js
const filename = process.argv[2];
print(`Import du fichier: ${filename}`);

// ... logique d'import
```

```bash
mongosh --file import-data.js data.json
```

---

## Performance et optimisation

### Analyser les requêtes

```javascript
// Plan d'exécution simple
db.users.find({email: "test@example.com"}).explain()

// Statistiques détaillées
db.users.find({email: "test@example.com"}).explain("executionStats")

// Tous les plans considérés
db.users.find({email: "test@example.com"}).explain("allPlansExecution")
```

### Profiler les requêtes lentes

```javascript
// Activer le profiler (niveau 1 = requêtes lentes)
db.setProfilingLevel(1, 100)  // Seuil: 100ms

// Consulter les requêtes profilées
db.system.profile.find().sort({ts: -1}).limit(5).pretty()

// Désactiver le profiler
db.setProfilingLevel(0)
```

---

## Raccourcis clavier essentiels

| Raccourci | Action |
|-----------|--------|
| `Ctrl+C` | Annuler la commande en cours |
| `Ctrl+D` | Quitter mongosh |
| `↑` / `↓` | Naviguer dans l'historique |
| `Tab` | Auto-complétion |
| `Ctrl+R` | Recherche dans l'historique (reverse search) |
| `Ctrl+L` | Effacer l'écran |
| `Ctrl+A` | Début de ligne |
| `Ctrl+E` | Fin de ligne |

---

## Différences avec l'ancien mongo shell

| Ancien (mongo) | Nouveau (mongosh) | Notes |
|----------------|-------------------|-------|
| `mongo` | `mongosh` | Nouvelle commande |
| API callback | API Promises/async-await | Syntaxe moderne |
| Pas de coloration | Coloration syntaxique | Meilleure lisibilité |
| Pretty print manuel | Auto-formatage | `.pretty()` optionnel |
| Pas de snippets | Support snippets NPM | Extensible |

---

## Ressources complémentaires

### Documentation officielle

- [Documentation mongosh](https://www.mongodb.com/docs/mongodb-shell/)
- [API Reference](https://www.mongodb.com/docs/manual/reference/method/)
- [mongosh sur GitHub](https://github.com/mongodb-js/mongosh)

### Outils complémentaires

| Outil | Usage |
|-------|-------|
| **MongoDB Compass** | GUI pour exploration visuelle |
| **MongoDB VSCode Extension** | Playgrounds dans VSCode |
| **Studio 3T** | IDE tiers avancé |
| **NoSQLBooster** | Alternative avec SQL |

---

## Structure de navigation

Cette annexe contient les sections suivantes :

- **[B.1 - Navigation](./01-navigation.md)** : Commandes d'exploration (show, use, etc.)
- **[B.2 - CRUD rapide](./02-crud-rapide.md)** : Opérations de base sur les données
- **[B.3 - Administration](./03-administration.md)** : Commandes d'admin et monitoring
- **[B.4 - Helpers et raccourcis](./04-helpers-raccourcis.md)** : Astuces et optimisations

---

## Notes importantes

> **⚠️ Environnements de production**
> - Toujours utiliser `--quiet` dans les scripts automatisés
> - Éviter les commandes sans limite sur de grosses collections
> - Utiliser Read Concern et Write Concern appropriés
> - Tester les commandes en dev/staging avant production

> **💡 Apprentissage progressif**
> - Commencez par B.1 (Navigation) si vous débutez
> - Pratiquez B.2 (CRUD) pour maîtriser les opérations de base
> - Explorez B.3 (Administration) quand vous gérez des environnements
> - Utilisez B.4 (Helpers) pour optimiser votre productivité

---

**Prêt à explorer mongosh ? Commencez par la section B.1 - Navigation ! 🚀**

⏭️ [Navigation (show dbs, use, show collections, etc.)](/annexes/commandes-mongosh/01-navigation.md)
