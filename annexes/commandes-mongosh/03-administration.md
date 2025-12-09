🔝 Retour au [Sommaire](/SOMMAIRE.md)

# B.3 - Administration

## Table des matières

1. [Informations Serveur](#informations-serveur)
2. [Gestion des Index](#gestion-des-index)
3. [Gestion des Utilisateurs](#gestion-des-utilisateurs)
4. [Gestion des Rôles](#gestion-des-r%C3%B4les)
5. [Replica Set](#replica-set)
6. [Sharding](#sharding)
7. [Profiling et Performance](#profiling-et-performance)
8. [Maintenance](#maintenance)
9. [Monitoring](#monitoring)
10. [Diagnostics](#diagnostics)

---

## Informations Serveur

### Statut du serveur

```javascript
db.serverStatus()
```

**Résultat** : Document JSON complet avec métriques système, réseau, réplication, etc.

**Sections utiles :**

```javascript
// Connexions actives
db.serverStatus().connections
// → { current: 52, available: 838808, totalCreated: 1234 }

// Utilisation mémoire
db.serverStatus().mem
// → { resident: 150, virtual: 500, supported: true }

// Opérations réseau
db.serverStatus().network
// → { bytesIn: 123456789, bytesOut: 987654321, numRequests: 45678 }

// Statut réplication
db.serverStatus().repl
```

---

### Informations de build

```javascript
db.version()
// → "7.0.5"

db.serverBuildInfo()
```

**Résultat :**

```javascript
{
  version: "7.0.5",
  gitVersion: "...",
  modules: [],
  allocator: "tcmalloc",
  javascriptEngine: "mozjs",
  sysInfo: "...",
  versionArray: [7, 0, 5, 0]
}
```

---

### Informations sur l'hôte

```javascript
db.hostInfo()
```

**Résultat :**

```javascript
{
  system: {
    currentTime: ISODate("2024-01-15T10:30:00Z"),
    hostname: "mongodb-server",
    cpuAddrSize: 64,
    numCores: 8,
    cpuArch: "x86_64",
    numaEnabled: false
  },
  os: {
    type: "Linux",
    name: "Ubuntu",
    version: "22.04"
  },
  extra: {
    pageSize: 4096,
    numPages: 8388608,
    maxOpenFiles: 65536
  }
}
```

---

### Paramètres de configuration

```javascript
// Obtenir un paramètre
db.adminCommand({ getParameter: 1, maxIncomingConnections: 1 })

// Obtenir tous les paramètres
db.adminCommand({ getParameter: "*" })

// Définir un paramètre (sans redémarrage)
db.adminCommand({
  setParameter: 1,
  logLevel: 1
})
```

---

## Gestion des Index

### Lister les index

```javascript
// Index d'une collection
db.<collection>.getIndexes()

// Index de toutes les collections
db.getCollectionNames().forEach(function(col) {
  print(col);
  printjson(db[col].getIndexes());
});
```

---

### Créer un index

```javascript
// Index simple
db.<collection>.createIndex({ fieldName: 1 })

// Index composé
db.<collection>.createIndex({ field1: 1, field2: -1 })

// Index unique
db.<collection>.createIndex(
  { email: 1 },
  { unique: true }
)

// Index avec options multiples
db.<collection>.createIndex(
  { field: 1 },
  {
    unique: true,
    sparse: true,
    name: "custom_index_name",
    background: true
  }
)
```

**Options courantes :**

| Option | Description | Exemple |
|--------|-------------|---------|
| `unique` | Valeurs uniques | `{ unique: true }` |
| `sparse` | Ignore documents sans champ | `{ sparse: true }` |
| `name` | Nom personnalisé | `{ name: "idx_email" }` |
| `background` | Construction en arrière-plan | `{ background: true }` |
| `expireAfterSeconds` | TTL index | `{ expireAfterSeconds: 3600 }` |
| `partialFilterExpression` | Index partiel | `{ partialFilterExpression: {...} }` |

---

### Index spécialisés

```javascript
// Index texte (recherche full-text)
db.articles.createIndex(
  { title: "text", content: "text" },
  { weights: { title: 10, content: 1 } }
)

// Index géospatial (2dsphere)
db.places.createIndex({ location: "2dsphere" })

// Index haché
db.users.createIndex({ userId: "hashed" })

// Index TTL (expiration automatique)
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 3600 }  // 1 heure
)

// Index wildcard
db.products.createIndex({ "attributes.$**": 1 })

// Index partiel
db.orders.createIndex(
  { customerId: 1, orderDate: -1 },
  { partialFilterExpression: { status: "active" } }
)
```

---

### Supprimer un index

```javascript
// Par nom
db.<collection>.dropIndex("index_name")

// Par spécification
db.<collection>.dropIndex({ field: 1 })

// Tous les index (sauf _id)
db.<collection>.dropIndexes()
```

**Exemple :**

```javascript
// Supprimer l'index sur email
db.users.dropIndex("email_1")

// Supprimer tous les index sauf _id
db.users.dropIndexes()
```

⚠️ **Attention** : Suppression d'index peut dégrader les performances.

---

### Reconstruire les index

```javascript
// Reconstruire tous les index d'une collection
db.<collection>.reIndex()
```

💡 **Usage** : Défragmentation, récupération d'espace disque.

⚠️ **Impact** : Opération bloquante, à faire en maintenance.

---

### Masquer/Afficher un index

```javascript
// Masquer un index (test avant suppression)
db.<collection>.hideIndex("index_name")

// Afficher un index masqué
db.<collection>.unhideIndex("index_name")
```

💡 **Astuce** : Masquer un index permet de tester son impact sans le supprimer.

---

### Statistiques d'utilisation des index

```javascript
// Statistiques d'index
db.<collection>.aggregate([
  { $indexStats: {} }
])
```

**Résultat :**

```javascript
{
  name: "email_1",
  key: { email: 1 },
  host: "mongodb-server:27017",
  accesses: {
    ops: 12345,
    since: ISODate("2024-01-01T00:00:00Z")
  }
}
```

💡 **Usage** : Identifier les index inutilisés.

---

## Gestion des Utilisateurs

### Créer un utilisateur

```javascript
db.createUser({
  user: "<username>",
  pwd: "<password>",
  roles: [
    { role: "<role>", db: "<database>" }
  ]
})
```

**Exemples :**

```javascript
// Utilisateur avec lecture seule
db.createUser({
  user: "reader",
  pwd: "securePassword123",
  roles: [
    { role: "read", db: "myapp" }
  ]
})

// Utilisateur avec lecture/écriture
db.createUser({
  user: "appUser",
  pwd: "securePassword123",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// Utilisateur admin
db.createUser({
  user: "admin",
  pwd: "adminPassword123",
  roles: [
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})

// Utilisateur avec plusieurs rôles
db.createUser({
  user: "multiRole",
  pwd: "password123",
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "read", db: "logs" },
    { role: "dbAdmin", db: "myapp" }
  ]
})
```

---

### Lister les utilisateurs

```javascript
// Utilisateurs de la base courante
db.getUsers()

// Utilisateur spécifique
db.getUser("<username>")

// Tous les utilisateurs (admin)
db.getSiblingDB("admin").system.users.find()
```

---

### Modifier un utilisateur

```javascript
// Changer le mot de passe
db.changeUserPassword("<username>", "<newPassword>")

// Modifier les rôles
db.updateUser("<username>", {
  roles: [
    { role: "readWrite", db: "myapp" },
    { role: "dbAdmin", db: "myapp" }
  ]
})

// Ajouter un rôle
db.grantRolesToUser("<username>", [
  { role: "read", db: "logs" }
])

// Retirer un rôle
db.revokeRolesFromUser("<username>", [
  { role: "read", db: "logs" }
])
```

---

### Supprimer un utilisateur

```javascript
db.dropUser("<username>")
```

**Exemple :**

```javascript
// Supprimer l'utilisateur "oldUser"
db.dropUser("oldUser")
```

---

### Authentification

```javascript
// S'authentifier
db.auth("<username>", "<password>")

// Vérifier l'utilisateur courant
db.runCommand({ connectionStatus: 1 })
```

---

## Gestion des Rôles

### Rôles intégrés courants

| Rôle | Portée | Description |
|------|--------|-------------|
| `read` | Base de données | Lecture seule |
| `readWrite` | Base de données | Lecture et écriture |
| `dbAdmin` | Base de données | Administration de la base |
| `userAdmin` | Base de données | Gestion des utilisateurs |
| `clusterAdmin` | Cluster | Administration cluster |
| `readAnyDatabase` | Toutes bases | Lecture sur toutes les bases |
| `readWriteAnyDatabase` | Toutes bases | Lecture/écriture sur toutes |
| `dbAdminAnyDatabase` | Toutes bases | Admin sur toutes les bases |
| `userAdminAnyDatabase` | Toutes bases | Gestion users sur toutes |
| `root` | Toutes bases | Accès super-admin |

---

### Lister les rôles

```javascript
// Rôles de la base courante
db.getRoles()

// Rôles intégrés
db.getRoles({ showBuiltinRoles: true })

// Détails d'un rôle
db.getRole("<roleName>", { showPrivileges: true })
```

---

### Créer un rôle personnalisé

```javascript
db.createRole({
  role: "<roleName>",
  privileges: [
    {
      resource: { db: "<database>", collection: "<collection>" },
      actions: ["<action1>", "<action2>"]
    }
  ],
  roles: [
    { role: "<inheritedRole>", db: "<database>" }
  ]
})
```

**Exemple :**

```javascript
// Rôle pour lire users et écrire dans logs
db.createRole({
  role: "appMonitor",
  privileges: [
    {
      resource: { db: "myapp", collection: "users" },
      actions: ["find"]
    },
    {
      resource: { db: "myapp", collection: "logs" },
      actions: ["insert", "update"]
    }
  ],
  roles: []
})
```

**Actions courantes :**
- `find`, `insert`, `update`, `remove`
- `createCollection`, `dropCollection`
- `createIndex`, `dropIndex`
- `listCollections`, `listIndexes`

---

### Modifier un rôle

```javascript
// Ajouter des privilèges
db.grantPrivilegesToRole("<roleName>", [
  {
    resource: { db: "<database>", collection: "<collection>" },
    actions: ["<action>"]
  }
])

// Retirer des privilèges
db.revokePrivilegesFromRole("<roleName>", [
  {
    resource: { db: "<database>", collection: "<collection>" },
    actions: ["<action>"]
  }
])
```

---

### Supprimer un rôle

```javascript
db.dropRole("<roleName>")
```

---

## Replica Set

### Statut du Replica Set

```javascript
rs.status()
```

**Résultat :**

```javascript
{
  set: "myReplicaSet",
  date: ISODate("2024-01-15T10:30:00Z"),
  myState: 1,  // 1 = PRIMARY, 2 = SECONDARY
  members: [
    {
      _id: 0,
      name: "mongodb1:27017",
      health: 1,
      state: 1,  // PRIMARY
      stateStr: "PRIMARY",
      uptime: 86400,
      optime: { ts: Timestamp(...), t: 5 },
      optimeDate: ISODate("..."),
      electionTime: Timestamp(...),
      electionDate: ISODate("...")
    },
    {
      _id: 1,
      name: "mongodb2:27017",
      health: 1,
      state: 2,  // SECONDARY
      stateStr: "SECONDARY",
      uptime: 86400,
      optime: { ts: Timestamp(...), t: 5 },
      optimeDate: ISODate("..."),
      syncSourceHost: "mongodb1:27017",
      syncSourceId: 0
    }
  ],
  ok: 1
}
```

---

### Configuration du Replica Set

```javascript
// Obtenir la configuration
rs.conf()

// Version compacte
rs.config()
```

**Résultat :**

```javascript
{
  _id: "myReplicaSet",
  version: 1,
  members: [
    {
      _id: 0,
      host: "mongodb1:27017",
      priority: 1,
      votes: 1
    },
    {
      _id: 1,
      host: "mongodb2:27017",
      priority: 1,
      votes: 1
    },
    {
      _id: 2,
      host: "mongodb3:27017",
      priority: 1,
      votes: 1
    }
  ],
  settings: {
    heartbeatIntervalMillis: 2000,
    electionTimeoutMillis: 10000
  }
}
```

---

### Initialiser un Replica Set

```javascript
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "mongodb1:27017" },
    { _id: 1, host: "mongodb2:27017" },
    { _id: 2, host: "mongodb3:27017" }
  ]
})
```

---

### Ajouter un membre

```javascript
// Ajouter un Secondary
rs.add("mongodb4:27017")

// Ajouter avec options
rs.add({
  host: "mongodb4:27017",
  priority: 0,
  hidden: true
})

// Ajouter un Arbiter
rs.addArb("mongodb5:27017")
```

---

### Supprimer un membre

```javascript
rs.remove("mongodb4:27017")
```

---

### Reconfigurer le Replica Set

```javascript
// Obtenir la config
const config = rs.conf()

// Modifier (exemple: changer priorité)
config.members[1].priority = 2

// Réappliquer (incrémenter version)
config.version++
rs.reconfig(config)

// Force reconfig (dangereux)
rs.reconfig(config, { force: true })
```

⚠️ **Attention** : `force: true` peut causer une perte de données.

---

### Forcer une élection

```javascript
// Forcer le nœud courant à devenir Primary
rs.stepDown(60)  // Secondary actuel devient Primary, dure 60s

// Déclencher une élection
db.adminCommand({ replSetStepDown: 60 })
```

---

### Vérifier le replication lag

```javascript
// Statut détaillé
rs.printReplicationInfo()

// Lag depuis Primary
rs.printSecondaryReplicationInfo()
```

**Résultat :**

```
configured oplog size:   1024MB
log length start to end: 86164s (23.93hrs)
oplog first event time:  Mon Jan 15 2024 10:00:00 GMT+0000
oplog last event time:   Tue Jan 16 2024 09:56:04 GMT+0000

source: mongodb2:27017
    syncedTo: Tue Jan 16 2024 09:56:03 GMT+0000
    1 secs (0 hrs) behind the primary
```

---

### Vérifier si Primary

```javascript
db.isMaster()
// ou
db.hello()  // Nouveau nom depuis MongoDB 5.0
```

**Résultat :**

```javascript
{
  ismaster: true,  // ou isWritablePrimary
  topologyVersion: {...},
  maxBsonObjectSize: 16777216,
  maxMessageSizeBytes: 48000000,
  maxWriteBatchSize: 100000,
  localTime: ISODate("..."),
  setName: "myReplicaSet",
  primary: "mongodb1:27017",
  hosts: ["mongodb1:27017", "mongodb2:27017", "mongodb3:27017"],
  ok: 1
}
```

---

## Sharding

### Statut du cluster shardé

```javascript
sh.status()

// Version détaillée
sh.status(true)
```

**Résultat :**

```javascript
--- Sharding Status ---
  sharding version: {
    "_id": 1,
    "version": 6
  }
  shards:
    { "_id": "shard01", "host": "shard01/mongodb1:27017,mongodb2:27017" }
    { "_id": "shard02", "host": "shard02/mongodb3:27017,mongodb4:27017" }
  databases:
    { "_id": "myapp", "primary": "shard01", "partitioned": true }
    myapp.users
      shard key: { "userId": "hashed" }
      chunks:
        shard01: 50
        shard02: 50
      { "userId": MinKey } -->> { "userId": NumberLong(0) } on: shard01
      { "userId": NumberLong(0) } -->> { "userId": MaxKey } on: shard02
```

---

### Activer le sharding

```javascript
// Activer le sharding sur une base
sh.enableSharding("<database>")

// Sharder une collection
sh.shardCollection(
  "<database>.<collection>",
  { <shardKey>: 1 }  // ou "hashed"
)
```

**Exemples :**

```javascript
// Activer sharding sur la base "myapp"
sh.enableSharding("myapp")

// Sharding par range sur userId
sh.shardCollection("myapp.users", { userId: 1 })

// Sharding haché sur userId
sh.shardCollection("myapp.logs", { userId: "hashed" })

// Sharding composé
sh.shardCollection("myapp.orders", { customerId: 1, orderDate: 1 })
```

---

### Ajouter un shard

```javascript
sh.addShard("mongodb5:27017")

// Ajouter un Replica Set comme shard
sh.addShard("shard03/mongodb5:27017,mongodb6:27017,mongodb7:27017")
```

---

### Supprimer un shard

```javascript
// Commencer la suppression (migre les données)
db.adminCommand({ removeShard: "shard03" })

// Vérifier l'état de la migration
db.adminCommand({ removeShard: "shard03" })

// Compléter la suppression (après migration complète)
db.adminCommand({ removeShard: "shard03" })
```

💡 **Processus** : La suppression d'un shard est progressive et nécessite plusieurs appels.

---

### Gestion du balancer

```javascript
// Vérifier si le balancer est actif
sh.getBalancerState()

// Activer le balancer
sh.startBalancer()

// Désactiver le balancer
sh.stopBalancer()

// Vérifier si le balancer tourne actuellement
sh.isBalancerRunning()
```

💡 **Usage** : Désactiver le balancer pendant les opérations de maintenance.

---

### Créer des zones (zone sharding)

```javascript
// Définir une plage de zone
sh.addShardToZone("shard01", "EU")
sh.addShardToZone("shard02", "US")

// Associer une plage de clés à une zone
sh.updateZoneKeyRange(
  "myapp.users",
  { country: "FR" },
  { country: "GB" },
  "EU"
)

sh.updateZoneKeyRange(
  "myapp.users",
  { country: "US" },
  { country: "US" },
  "US"
)
```

---

### Répartition des chunks

```javascript
// Diviser un chunk manuellement
sh.splitAt("myapp.users", { userId: 5000 })

// Diviser en trouvant le milieu
sh.splitFind("myapp.users", { userId: 5000 })

// Déplacer un chunk
sh.moveChunk(
  "myapp.users",
  { userId: 5000 },
  "shard02"
)
```

⚠️ **Attention** : Opérations manuelles rarement nécessaires, le balancer gère automatiquement.

---

## Profiling et Performance

### Activer le profiler

```javascript
// Niveau 0: Désactivé
db.setProfilingLevel(0)

// Niveau 1: Requêtes lentes uniquement (> seuil)
db.setProfilingLevel(1, 100)  // Seuil: 100ms

// Niveau 2: Toutes les requêtes
db.setProfilingLevel(2)

// Obtenir le niveau actuel
db.getProfilingStatus()
```

**Résultat :**

```javascript
{
  was: 1,
  slowms: 100,
  sampleRate: 1.0,
  ok: 1
}
```

⚠️ **Impact** : Niveau 2 dégrade les performances, à utiliser temporairement.

---

### Consulter les données de profiling

```javascript
// 10 dernières opérations profilées
db.system.profile.find().sort({ ts: -1 }).limit(10)

// Requêtes > 1000ms
db.system.profile.find({ millis: { $gt: 1000 } })

// Requêtes sur une collection spécifique
db.system.profile.find({ ns: "myapp.users" })

// Avec détails de l'explain
db.system.profile.find({ "command.comment": "slowQuery" })
```

---

### Analyser une requête avec explain

```javascript
// Plan d'exécution simple
db.users.find({ age: 30 }).explain()

// Statistiques d'exécution
db.users.find({ age: 30 }).explain("executionStats")

// Tous les plans considérés
db.users.find({ age: 30 }).explain("allPlansExecution")
```

**Métriques importantes :**
- `executionTimeMillis` : Temps d'exécution
- `totalDocsExamined` : Documents scannés
- `totalKeysExamined` : Clés d'index scannées
- `stage` : Type de scan (IXSCAN = index, COLLSCAN = collection)

---

### Opérations en cours

```javascript
// Toutes les opérations en cours
db.currentOp()

// Opérations actives (excluant idle)
db.currentOp({ "$all": false })

// Opérations longues (> 5 secondes)
db.currentOp({ "secs_running": { "$gte": 5 } })

// Opérations d'écriture
db.currentOp({ "op": { "$in": ["insert", "update", "remove"] } })
```

---

### Tuer une opération

```javascript
// Par opId
db.killOp(<opId>)
```

**Exemple :**

```javascript
// 1. Trouver l'opération
const ops = db.currentOp({ "secs_running": { "$gte": 60 } })
print(ops.inprog[0].opid)

// 2. Tuer l'opération
db.killOp(12345)
```

---

## Maintenance

### Compactage d'une collection

```javascript
db.<collection>.compact()
```

💡 **Usage** : Récupère l'espace disque après suppressions massives.

⚠️ **Impact** : Opération bloquante, à faire en maintenance.

---

### Validation d'une collection

```javascript
// Validation basique
db.<collection>.validate()

// Validation complète (plus lent)
db.<collection>.validate({ full: true })
```

**Résultat :**

```javascript
{
  ns: "myapp.users",
  nInvalidDocuments: 0,
  nrecords: 100000,
  nIndexes: 3,
  keysPerIndex: {
    "_id_": 100000,
    "email_1": 100000,
    "age_1": 100000
  },
  valid: true,
  errors: [],
  ok: 1
}
```

---

### Réparer une base de données

```javascript
db.repairDatabase()
```

⚠️ **ATTENTION** :
- Opération destructive en cas de corruption
- Nécessite beaucoup d'espace disque
- Bloque toutes les opérations
- À utiliser en dernier recours

---

### Vider une collection

```javascript
// Supprimer tous les documents (mais garde les index)
db.<collection>.deleteMany({})

// Supprimer et recréer (efface aussi les index)
db.<collection>.drop()
db.createCollection("<collection>")
```

---

### Cloner une base de données

```javascript
// Cloner depuis un autre serveur
db.cloneDatabase("<host>")

// Copier une base locale
db.copyDatabase("<sourceDb>", "<targetDb>")
```

⚠️ **Déprécié** : Utiliser `mongodump`/`mongorestore` à la place.

---

## Monitoring

### Statistiques serveur en temps réel

```javascript
// Statistiques de la base
db.stats(1024 * 1024)  // En MB

// Statistiques d'une collection
db.<collection>.stats(1024 * 1024)

// Statistiques globales
db.serverStatus()
```

---

### Monitoring des connexions

```javascript
// Nombre de connexions
db.serverStatus().connections
// → { current: 52, available: 838808 }

// Détails des connexions
db.currentOp({ "$all": true })
```

---

### Monitoring de la mémoire

```javascript
db.serverStatus().mem
// → { resident: 150, virtual: 500 }

db.serverStatus().wiredTiger.cache
```

---

### Monitoring du réseau

```javascript
db.serverStatus().network
// → { bytesIn: 123456789, bytesOut: 987654321, numRequests: 45678 }
```

---

### Monitoring des opérations

```javascript
db.serverStatus().opcounters
// → { insert: 1000, query: 5000, update: 2000, delete: 500 }

db.serverStatus().opcountersRepl
// → Opérations de réplication
```

---

## Diagnostics

### Logs du serveur

```javascript
// Afficher les logs récents
db.adminCommand({ getLog: "global" })

// Types de logs disponibles
db.adminCommand({ getLog: "*" })

// Warnings au démarrage
db.adminCommand({ getLog: "startupWarnings" })
```

---

### Niveau de logging

```javascript
// Obtenir le niveau
db.getLogComponents()

// Définir le niveau global
db.setLogLevel(1)

// Définir le niveau par composant
db.setLogLevel(2, "query")
db.setLogLevel(3, "replication")
```

**Niveaux :** 0 (défaut) à 5 (debug verbose)

---

### Diagnostics du Replica Set

```javascript
// Santé du Replica Set
rs.status()

// Vérifier la synchronisation
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()

// Détails de l'Oplog
db.getSiblingDB("local").oplog.rs.stats()
```

---

### Diagnostics du Sharding

```javascript
// État du cluster
sh.status()

// État du balancer
sh.getBalancerState()
sh.isBalancerRunning()

// Chunks par shard
db.getSiblingDB("config").chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])
```

---

### Diagnostics de performance

```javascript
// Index inutilisés
db.<collection>.aggregate([
  { $indexStats: {} }
]).forEach(function(index) {
  if (index.accesses.ops === 0) {
    print("Index non utilisé: " + index.name);
  }
});

// Collections volumineuses
db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024);
  if (stats.size > 1000) {  // > 1 GB
    print(`${col}: ${stats.size.toFixed(2)} MB`);
  }
});

// Requêtes lentes dans le profiler
db.system.profile.find({ millis: { $gt: 1000 } }).sort({ millis: -1 })
```

---

## Commandes administratives avancées

### Shutdown du serveur

```javascript
// Arrêt propre
db.adminCommand({ shutdown: 1 })

// Arrêt forcé (non recommandé)
db.adminCommand({ shutdown: 1, force: true })
```

⚠️ **Attention** : Nécessite privilèges admin sur la base "admin".

---

### Rotation des logs

```javascript
db.adminCommand({ logRotate: 1 })
```

---

### Fsync et lock

```javascript
// Forcer l'écriture sur disque
db.fsyncLock()

// Libérer après backup
db.fsyncUnlock()

// Vérifier le statut
db.currentOp({ "fsyncLock": 1 })
```

💡 **Usage** : Backups cohérents (snapshots disque).

---

### Resynchroniser un Secondary

```javascript
// Sur le Secondary à resynchroniser
use local
db.dropDatabase()  // Supprime l'oplog

// Redémarrer mongod, resync automatique
```

⚠️ **Impact** : Resync complet depuis le Primary, peut être long.

---

## Tableau récapitulatif

### Commandes par catégorie

#### Informations

| Commande | Description | Niveau |
|----------|-------------|--------|
| `db.serverStatus()` | Métriques serveur | 🟡 Intermédiaire |
| `db.version()` | Version MongoDB | 🟢 Débutant |
| `db.hostInfo()` | Infos système | 🟡 Intermédiaire |
| `db.isMaster()` | Rôle dans Replica Set | 🟡 Intermédiaire |

#### Index

| Commande | Description | Niveau |
|----------|-------------|--------|
| `db.col.getIndexes()` | Liste index | 🟢 Débutant |
| `db.col.createIndex()` | Créer index | 🟢 Débutant |
| `db.col.dropIndex()` | Supprimer index | 🟢 Débutant |
| `db.col.reIndex()` | Reconstruire index | 🔴 Avancé |

#### Utilisateurs

| Commande | Description | Niveau |
|----------|-------------|--------|
| `db.createUser()` | Créer utilisateur | 🟡 Intermédiaire |
| `db.getUsers()` | Liste utilisateurs | 🟡 Intermédiaire |
| `db.changeUserPassword()` | Changer password | 🟡 Intermédiaire |
| `db.dropUser()` | Supprimer utilisateur | 🟡 Intermédiaire |

#### Replica Set

| Commande | Description | Niveau |
|----------|-------------|--------|
| `rs.status()` | Statut Replica Set | 🔴 Avancé |
| `rs.conf()` | Configuration | 🔴 Avancé |
| `rs.add()` | Ajouter membre | 🔴 Avancé |
| `rs.stepDown()` | Forcer élection | 🔴 Avancé |

#### Sharding

| Commande | Description | Niveau |
|----------|-------------|--------|
| `sh.status()` | Statut cluster | 🔴 Avancé |
| `sh.enableSharding()` | Activer sharding | 🔴 Avancé |
| `sh.shardCollection()` | Sharder collection | 🔴 Avancé |
| `sh.stopBalancer()` | Arrêter balancer | 🔴 Avancé |

#### Performance

| Commande | Description | Niveau |
|----------|-------------|--------|
| `db.setProfilingLevel()` | Config profiler | 🟡 Intermédiaire |
| `db.currentOp()` | Opérations en cours | 🟡 Intermédiaire |
| `db.killOp()` | Tuer opération | 🟡 Intermédiaire |
| `.explain()` | Analyser requête | 🟡 Intermédiaire |

---

## Scripts d'administration courants

### Script de monitoring général

```javascript
// monitoring.js
print("=== MongoDB Monitoring ===\n");

print("Version: " + db.version());
print("Uptime: " + db.serverStatus().uptime + " seconds\n");

print("Connections:");
printjson(db.serverStatus().connections);

print("\nMemory:");
printjson(db.serverStatus().mem);

print("\nOperations:");
printjson(db.serverStatus().opcounters);

if (rs.status().ok) {
  print("\nReplica Set: " + rs.status().set);
  print("State: " + rs.status().myState);
}

print("\n=== Collections ===");
db.getCollectionNames().forEach(function(col) {
  const stats = db[col].stats(1024 * 1024);
  print(`${col}: ${stats.count} docs, ${stats.size.toFixed(2)} MB`);
});
```

---

### Script de vérification des index

```javascript
// check-indexes.js
print("=== Unused Indexes ===\n");

db.getCollectionNames().forEach(function(col) {
  const indexes = db[col].aggregate([{ $indexStats: {} }]);

  indexes.forEach(function(index) {
    if (index.accesses.ops === 0) {
      print(`${col}.${index.name}: UNUSED`);
    }
  });
});
```

---

### Script de nettoyage

```javascript
// cleanup.js
print("=== Database Cleanup ===\n");

// Supprimer les index inutilisés (après analyse)
// db.collection.dropIndex("unused_index_name");

// Compacter les collections (en maintenance)
// db.collection.compact();

// Rotation des logs
db.adminCommand({ logRotate: 1 });
print("Logs rotated.");

// Nettoyer le profiler
db.system.profile.deleteMany({ ts: { $lt: new Date(Date.now() - 86400000) } });
print("Old profiler data cleaned.");
```

---

**⚠️ Rappel sécurité** : Les commandes d'administration nécessitent des privilèges appropriés. Testez toujours en environnement de développement avant la production.

⏭️ [Helpers et raccourcis](/annexes/commandes-mongosh/04-helpers-raccourcis.md)
