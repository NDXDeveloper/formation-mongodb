🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.1 - Navigation

## Table des matières

1. [Bases de données](#bases-de-donn%C3%A9es)
2. [Collections](#collections)
3. [Informations sur la base courante](#informations-sur-la-base-courante)
4. [Informations sur les collections](#informations-sur-les-collections)
5. [Navigation dans les résultats](#navigation-dans-les-r%C3%A9sultats)
6. [Contexte et environnement](#contexte-et-environnement)
7. [Commandes système](#commandes-syst%C3%A8me)

---

## Bases de données

### Lister toutes les bases de données

```javascript
show dbs
// ou
show databases
```

**Résultat :**
```
admin      100 KB
config     80 KB
myapp      2.5 GB
test       1 MB
```

💡 **Note** : Seules les bases contenant des données sont affichées. Une base vide n'apparaît pas.

---

### Afficher la base de données courante

```javascript
db
// ou
db.getName()
```

**Résultat :**
```
myapp
```

---

### Basculer vers une base de données

```javascript
use <nom_database>
```

**Exemples :**

```javascript
// Basculer vers la base "myapp"
use myapp

// Basculer vers une base qui n'existe pas encore
use newDatabase  // La base sera créée lors de la première insertion
```

💡 **Note** : La commande `use` crée automatiquement la base si elle n'existe pas (création lors du premier document inséré).

---

### Obtenir l'objet base de données

```javascript
// Base courante
db

// Base spécifique (sans basculer)
db.getSiblingDB("autreDatabase")
```

**Exemple d'usage :**

```javascript
// Requête sur une autre base sans changer de contexte
const otherDb = db.getSiblingDB("logs");
otherDb.events.countDocuments();

// On reste dans la base courante
db.getName()  // Retourne toujours la base précédente
```

---

## Collections

### Lister toutes les collections

```javascript
show collections
// ou
show tables
// ou
db.getCollectionNames()
```

**Résultat :**

```javascript
// show collections affiche :
users
orders
products
logs

// db.getCollectionNames() retourne un tableau :
['users', 'orders', 'products', 'logs']
```

---

### Obtenir une collection

```javascript
// Syntaxe standard
db.nomCollection

// Syntaxe avec crochets (si nom spécial)
db["nom-avec-tirets"]
db.getCollection("nom-collection")
```

**Exemples :**

```javascript
// Collection "users"
db.users

// Collection avec caractères spéciaux
db["user-profiles"]
db.getCollection("user-profiles")

// Collection dont le nom est dans une variable
const collName = "orders";
db[collName]
db.getCollection(collName)
```

---

### Créer une collection

```javascript
// Création explicite
db.createCollection("<nom_collection>")

// Création avec options
db.createCollection("<nom_collection>", {
  capped: true,
  size: 100000,
  max: 5000
})
```

**Exemples :**

```javascript
// Création simple
db.createCollection("logs")

// Collection cappée (taille fixe, FIFO)
db.createCollection("recentLogs", {
  capped: true,
  size: 10485760,  // 10 MB
  max: 10000       // Maximum 10000 documents
})

// Collection avec validation de schéma
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      required: ["email", "name"],
      properties: {
        email: { bsonType: "string" },
        name: { bsonType: "string" }
      }
    }
  }
})
```

💡 **Note** : En général, pas besoin de créer explicitement les collections. MongoDB les crée automatiquement lors de la première insertion.

---

### Supprimer une collection

```javascript
db.<collection>.drop()
```

**Exemple :**

```javascript
// Supprimer la collection "logs"
db.logs.drop()

// Retourne true si succès, false si collection n'existe pas
```

⚠️ **Attention** : Opération destructive et irréversible !

---

### Renommer une collection

```javascript
db.<collection>.renameCollection("<nouveau_nom>")
```

**Exemple :**

```javascript
// Renommer "oldUsers" en "users"
db.oldUsers.renameCollection("users")
```

⚠️ **Important** : Ne fonctionne que dans la même base de données.

---

## Informations sur la base courante

### Statistiques de la base

```javascript
db.stats()
// ou
db.stats(1024 * 1024)  // Tailles en MB au lieu d'octets
```

**Résultat :**

```javascript
{
  db: 'myapp',
  collections: 5,
  views: 0,
  objects: 125000,        // Nombre total de documents
  avgObjSize: 512,        // Taille moyenne d'un document (octets)
  dataSize: 64000000,     // Taille totale des données
  storageSize: 32000000,  // Espace disque utilisé
  indexes: 10,
  indexSize: 5000000,     // Taille des index
  totalSize: 37000000,
  scaleFactor: 1,
  fsUsedSize: 150000000,
  fsTotalSize: 500000000,
  ok: 1
}
```

---

### Obtenir la version de MongoDB

```javascript
db.version()
```

**Résultat :**
```
7.0.5
```

---

### Obtenir les informations du serveur

```javascript
db.serverStatus()
```

**Résultat** : Document JSON détaillé avec :
- Informations système
- Métriques de performance
- Statistiques réseau
- Informations de réplication
- Etc.

💡 **Astuce** : Très verbeux, utilisez des projections :

```javascript
// Uniquement les connexions
db.serverStatus().connections

// Uniquement la mémoire
db.serverStatus().mem
```

---

### Informations sur le serveur (admin)

```javascript
db.adminCommand({ serverStatus: 1 })
// ou
db.serverBuildInfo()  // Informations de build
// ou
db.hostInfo()  // Informations sur l'hôte
```

---

## Informations sur les collections

### Statistiques d'une collection

```javascript
db.<collection>.stats()
// ou
db.<collection>.stats(1024 * 1024)  // Tailles en MB
```

**Exemple :**

```javascript
db.users.stats()
```

**Résultat :**

```javascript
{
  ns: 'myapp.users',
  size: 52428800,           // Taille des données (50 MB)
  count: 100000,            // Nombre de documents
  avgObjSize: 524,          // Taille moyenne (octets)
  storageSize: 26214400,    // Espace disque (25 MB)
  nindexes: 3,              // Nombre d'index
  totalIndexSize: 3145728,  // Taille totale des index (3 MB)
  indexSizes: {
    _id_: 1048576,
    email_1: 1048576,
    name_1_age_1: 1048576
  },
  ok: 1
}
```

---

### Lister les index d'une collection

```javascript
db.<collection>.getIndexes()
```

**Exemple :**

```javascript
db.users.getIndexes()
```

**Résultat :**

```javascript
[
  {
    v: 2,
    key: { _id: 1 },
    name: '_id_'
  },
  {
    v: 2,
    key: { email: 1 },
    name: 'email_1',
    unique: true
  },
  {
    v: 2,
    key: { name: 1, age: 1 },
    name: 'name_1_age_1'
  }
]
```

---

### Obtenir le schéma d'une collection (échantillon)

```javascript
// Pas de méthode native, mais on peut échantillonner :
db.<collection>.findOne()

// Ou analyser plusieurs documents
db.<collection>.find().limit(10)
```

💡 **Astuce** : Utilisez MongoDB Compass pour une analyse de schéma automatique.

---

### Compter les documents

```javascript
// Compte exact
db.<collection>.countDocuments()

// Compte exact avec filtre
db.<collection>.countDocuments({ status: "active" })

// Estimation rapide (utilise les métadonnées)
db.<collection>.estimatedDocumentCount()
```

**Exemples :**

```javascript
// Tous les documents
db.users.countDocuments()
// → 100000

// Documents actifs
db.users.countDocuments({ active: true })
// → 75000

// Estimation (plus rapide, pas de filtre possible)
db.users.estimatedDocumentCount()
// → ~100000
```

⚠️ **Différence** :
- `countDocuments()` : Compte exact, peut être lent sur grandes collections
- `estimatedDocumentCount()` : Très rapide, imprécis, pas de filtre

---

## Navigation dans les résultats

### Itérer sur un curseur

```javascript
// Obtenir un curseur
const cursor = db.<collection>.find()

// Parcourir manuellement
cursor.next()      // Document suivant
cursor.hasNext()   // Vérifie s'il reste des documents
```

**Exemple :**

```javascript
const cursor = db.users.find().limit(5);

while (cursor.hasNext()) {
  const doc = cursor.next();
  print(doc.name);
}
```

---

### Convertir en tableau

```javascript
db.<collection>.find().toArray()
```

**Exemple :**

```javascript
const users = db.users.find({ active: true }).limit(10).toArray();
print(`Trouvé ${users.length} utilisateurs actifs`);
```

⚠️ **Attention** : `toArray()` charge tous les résultats en mémoire. À utiliser avec modération sur de grandes collections.

---

### Itérer avec forEach

```javascript
db.<collection>.find().forEach(function(doc) {
  // Traitement de chaque document
  print(doc.name);
})
```

**Exemple :**

```javascript
// Afficher tous les emails
db.users.find({}, { email: 1, _id: 0 }).forEach(function(doc) {
  print(doc.email);
});

// Avec fonction fléchée (ES6)
db.users.find().forEach(doc => print(doc.name));
```

---

### Mapper les résultats

```javascript
db.<collection>.find().map(function(doc) {
  return doc.fieldName;
})
```

**Exemple :**

```javascript
// Extraire uniquement les noms
const names = db.users.find().map(doc => doc.name);
print(names);
// → ['Alice', 'Bob', 'Charlie', ...]

// Transformation complexe
const userSummaries = db.users.find().map(doc => ({
  fullName: `${doc.firstName} ${doc.lastName}`,
  email: doc.email
}));
```

---

### Taille du batch de résultats

```javascript
// Voir la taille actuelle
DBQuery.shellBatchSize

// Modifier (par défaut: 20)
DBQuery.shellBatchSize = 50
```

💡 **Note** : Affecte le nombre de documents affichés avant "Type 'it' for more".

---

### Continuer l'affichage des résultats

```javascript
// Après avoir lancé une requête qui affiche "Type 'it' for more"
it
```

**Exemple :**

```javascript
db.logs.find()
// Affiche 20 premiers résultats + "Type 'it' for more"

it
// Affiche les 20 suivants
```

---

## Contexte et environnement

### Obtenir la connexion courante

```javascript
db.getMongo()
```

**Résultat :**

```javascript
mongodb://localhost:27017/?directConnection=true&serverSelectionTimeoutMS=2000
```

---

### Lister les bases accessibles

```javascript
db.adminCommand('listDatabases')
```

**Résultat :**

```javascript
{
  databases: [
    { name: 'admin', sizeOnDisk: 102400, empty: false },
    { name: 'config', sizeOnDisk: 81920, empty: false },
    { name: 'myapp', sizeOnDisk: 2684354560, empty: false },
    { name: 'test', sizeOnDisk: 1048576, empty: false }
  ],
  totalSize: 2684539456,
  ok: 1
}
```

---

### Informations de réplication

```javascript
// Statut du Replica Set
rs.status()

// Configuration du Replica Set
rs.conf()

// Est-ce un Primary ?
db.isMaster()
```

💡 **Note** : Ces commandes nécessitent un Replica Set configuré.

---

### Informations de sharding

```javascript
// Statut du cluster shardé
sh.status()

// Version courte
sh.status(true)
```

💡 **Note** : Nécessite un cluster shardé.

---

## Commandes système

### Lister les utilisateurs

```javascript
show users
// ou
db.getUsers()
```

**Résultat :**

```javascript
[
  {
    _id: 'myapp.admin',
    userId: UUID("..."),
    user: 'admin',
    db: 'myapp',
    roles: [
      { role: 'readWrite', db: 'myapp' }
    ]
  }
]
```

---

### Lister les rôles

```javascript
show roles
// ou
db.getRoles()
```

---

### Afficher les logs

```javascript
show logs
// → Affiche les types de logs disponibles

show log global
// → Affiche les logs globaux récents

show log startupWarnings
// → Affiche les avertissements au démarrage
```

---

### Afficher le profiler

```javascript
show profile

// Équivalent à :
db.system.profile.find().limit(5).sort({ ts: -1 }).pretty()
```

💡 **Note** : Le profiler doit être activé (`db.setProfilingLevel(1)` ou `2`).

---

### Effacer l'écran

```javascript
cls
// ou utiliser Ctrl+L
```

---

### Quitter mongosh

```javascript
exit
// ou
quit()
// ou Ctrl+D (Linux/Mac) / Ctrl+C puis Y (Windows)
```

---

## Astuces de navigation

### Navigation rapide entre collections

```javascript
// Assigner une collection à une variable
const users = db.users;
const orders = db.orders;

// Utiliser directement
users.countDocuments()
orders.find({ status: "shipped" })
```

---

### Exploration rapide

```javascript
// Afficher un document exemple
db.<collection>.findOne()

// 5 premiers documents
db.<collection>.find().limit(5)

// Compter rapidement
db.<collection>.estimatedDocumentCount()

// Lister les champs d'un document
Object.keys(db.<collection>.findOne())
// → ['_id', 'name', 'email', 'createdAt', ...]
```

---

### Chaînage de contextes

```javascript
// Enchaîner plusieurs opérations
db.getSiblingDB("logs")
  .events
  .find({ level: "error" })
  .sort({ timestamp: -1 })
  .limit(10)
```

---

## Tableau récapitulatif des commandes

### Bases de données

| Commande | Description | Niveau |
|----------|-------------|--------|
| `show dbs` | Liste toutes les bases | 🟢 Débutant |
| `db` | Affiche la base courante | 🟢 Débutant |
| `use <db>` | Bascule vers une base | 🟢 Débutant |
| `db.stats()` | Statistiques de la base | 🟡 Intermédiaire |
| `db.getSiblingDB("<db>")` | Référence autre base | 🟡 Intermédiaire |

### Collections

| Commande | Description | Niveau |
|----------|-------------|--------|
| `show collections` | Liste les collections | 🟢 Débutant |
| `db.getCollectionNames()` | Liste (format array) | 🟢 Débutant |
| `db.<coll>.stats()` | Statistiques collection | 🟡 Intermédiaire |
| `db.<coll>.getIndexes()` | Liste les index | 🟡 Intermédiaire |
| `db.<coll>.countDocuments()` | Compte exact | 🟢 Débutant |
| `db.<coll>.estimatedDocumentCount()` | Estimation rapide | 🟡 Intermédiaire |

### Informations

| Commande | Description | Niveau |
|----------|-------------|--------|
| `db.version()` | Version MongoDB | 🟢 Débutant |
| `db.serverStatus()` | Statut du serveur | 🔴 Avancé |
| `db.hostInfo()` | Infos sur l'hôte | 🔴 Avancé |
| `rs.status()` | Statut Replica Set | 🟡 Intermédiaire |
| `sh.status()` | Statut Sharding | 🔴 Avancé |

### Navigation résultats

| Commande | Description | Niveau |
|----------|-------------|--------|
| `cursor.next()` | Document suivant | 🟢 Débutant |
| `cursor.hasNext()` | Vérifie si documents restants | 🟢 Débutant |
| `.toArray()` | Convertit en tableau | 🟢 Débutant |
| `.forEach()` | Itère sur résultats | 🟢 Débutant |
| `.map()` | Transforme résultats | 🟡 Intermédiaire |
| `it` | Continue l'affichage | 🟢 Débutant |

---

## Exemples de workflows complets

### Exploration d'une nouvelle base

```javascript
// 1. Se connecter et lister les bases
show dbs

// 2. Basculer vers une base
use myapp

// 3. Lister les collections
show collections

// 4. Explorer une collection
db.users.findOne()
db.users.countDocuments()

// 5. Voir la structure
Object.keys(db.users.findOne())

// 6. Vérifier les index
db.users.getIndexes()

// 7. Statistiques
db.users.stats(1024 * 1024)  // En MB
```

---

### Diagnostic rapide

```javascript
// Quelle base suis-je ?
db

// Quelle version ?
db.version()

// Combien de collections ?
db.getCollectionNames().length

// Taille totale de la base ?
db.stats(1024 * 1024)  // En MB

// Collections les plus volumineuses ?
db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024);
  print(`${col}: ${stats.size.toFixed(2)} MB`);
});
```

---

### Navigation multi-bases

```javascript
// Comparer des données entre bases
const prodUsers = db.getSiblingDB("production").users.countDocuments();
const testUsers = db.getSiblingDB("test").users.countDocuments();

print(`Production: ${prodUsers} utilisateurs`);
print(`Test: ${testUsers} utilisateurs`);

// Reste dans la base courante
db.getName()  // → base d'origine
```

---

**💡 Conseil** : Ajoutez cette page à vos favoris pour une consultation rapide pendant vos sessions mongosh !

⏭️ [CRUD rapide](/annexes/commandes-mongosh/02-crud-rapide.md)
