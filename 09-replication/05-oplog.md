🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.5 Oplog (Operations Log)

## Introduction

L'**Oplog** (Operations Log) est le journal des opérations de réplication dans MongoDB. Il constitue le mécanisme central qui permet la synchronisation des données entre les membres d'un Replica Set. Comprendre son fonctionnement est essentiel pour maîtriser la réplication, diagnostiquer les problèmes de lag et optimiser les performances.

## Concepts Fondamentaux

### Définition et Rôle

L'oplog est une **capped collection** spéciale qui enregistre toutes les opérations modifiant les données sur le Primary. Les membres Secondary lisent et appliquent ces opérations pour maintenir leurs données synchronisées.

**Caractéristiques clés** :
- Collection fixe de type FIFO (First In, First Out)
- Stockée dans la base de données système `local`
- Nom de collection : `oplog.rs`
- Les anciennes entrées sont automatiquement supprimées lorsque la limite de taille est atteinte
- Accessibilité en lecture seule (sauf pour le système de réplication)

### Emplacement

```javascript
// Connexion à la base local
use local

// La collection oplog.rs
db.oplog.rs.find().limit(5).pretty()
```

L'oplog n'est **jamais répliqué** entre les membres - chaque membre maintient son propre oplog local des opérations qu'il a appliquées.

## Structure de l'Oplog

### Anatomie d'une Entrée Oplog

Chaque opération dans l'oplog est représentée par un document avec la structure suivante :

```javascript
{
  "ts": Timestamp(1705320000, 1),     // Timestamp unique
  "t": NumberLong(42),                 // Term (mandat électoral)
  "h": NumberLong("-1234567890"),     // Hash unique de l'opération
  "v": 2,                              // Version du format oplog
  "op": "i",                           // Type d'opération
  "ns": "mydb.users",                  // Namespace (base.collection)
  "ui": UUID("a1b2c3d4-..."),         // UUID de la collection
  "wall": ISODate("2024-01-15T10:00:00Z"), // Heure réelle
  "o": {                               // Objet opération
    "_id": ObjectId("..."),
    "name": "Alice",
    "email": "alice@example.com"
  },
  "o2": { "_id": ObjectId("...") }    // Critère (pour updates/deletes)
}
```

### Champs Principaux

#### ts (Timestamp)

Le timestamp est un identifiant unique et monotone :

```javascript
Timestamp(1705320000, 1)
//        └─ secondes Unix
//                     └─ ordinal (numéro d'ordre)
```

**Propriétés** :
- Combine un timestamp Unix (secondes depuis epoch) et un compteur ordinal
- Garantit l'unicité même pour plusieurs opérations dans la même seconde
- Utilisé pour synchroniser les Secondary
- Permet le **Point-in-Time Recovery**

#### t (Term)

Le term correspond au mandat électoral du Primary qui a créé l'entrée :

```javascript
"t": NumberLong(42)
```

- S'incrémente à chaque nouvelle élection
- Permet de détecter les divergences après un changement de Primary
- Utilisé par le protocole Raft pour garantir la cohérence

#### op (Type d'Opération)

Code indiquant le type d'opération :

| Code | Type | Description |
|------|------|-------------|
| `i` | Insert | Insertion d'un document |
| `u` | Update | Modification d'un document |
| `d` | Delete | Suppression d'un document |
| `c` | Command | Commande administrative (createIndex, dropCollection, etc.) |
| `n` | No-op | Opération vide (heartbeat, élections) |

#### ns (Namespace)

Identifie la base de données et la collection :

```javascript
"ns": "production.orders"
//     └─ database
//                └─ collection
```

Pour les commandes :
```javascript
"ns": "production.$cmd"  // Commande sur la base production
```

#### o et o2 (Objets)

- **o** : L'objet principal de l'opération
  - Pour `i` : le document inséré
  - Pour `u` : les modifications à appliquer
  - Pour `d` : (vide, car o2 contient le critère)
  - Pour `c` : la commande à exécuter

- **o2** : Critère de sélection (pour `u` et `d`)
  ```javascript
  "o2": { "_id": ObjectId("507f1f77bcf86cd799439011") }
  ```

### Exemples d'Entrées Oplog

#### Insertion

```javascript
{
  "ts": Timestamp(1705320000, 1),
  "t": NumberLong(42),
  "h": NumberLong("-8765432109876543210"),
  "v": 2,
  "op": "i",
  "ns": "ecommerce.products",
  "ui": UUID("550e8400-e29b-41d4-a716-446655440000"),
  "wall": ISODate("2024-01-15T10:00:00.123Z"),
  "o": {
    "_id": ObjectId("65a5f1234567890abcdef012"),
    "name": "Laptop Pro 15",
    "price": 1299.99,
    "category": "Electronics",
    "stock": 50
  }
}
```

#### Update

```javascript
{
  "ts": Timestamp(1705320001, 1),
  "t": NumberLong(42),
  "h": NumberLong("-8765432109876543211"),
  "v": 2,
  "op": "u",
  "ns": "ecommerce.products",
  "ui": UUID("550e8400-e29b-41d4-a716-446655440000"),
  "wall": ISODate("2024-01-15T10:00:01.456Z"),
  "o": {
    "$v": 2,
    "diff": {
      "u": {
        "stock": 45,
        "lastModified": ISODate("2024-01-15T10:00:01.456Z")
      }
    }
  },
  "o2": {
    "_id": ObjectId("65a5f1234567890abcdef012")
  }
}
```

**Note** : Depuis MongoDB 4.4, les updates utilisent le format "diff" pour optimiser la taille des entrées oplog.

#### Delete

```javascript
{
  "ts": Timestamp(1705320002, 1),
  "t": NumberLong(42),
  "h": NumberLong("-8765432109876543212"),
  "v": 2,
  "op": "d",
  "ns": "ecommerce.products",
  "ui": UUID("550e8400-e29b-41d4-a716-446655440000"),
  "wall": ISODate("2024-01-15T10:00:02.789Z"),
  "o": {
    "_id": ObjectId("65a5f1234567890abcdef099")
  }
}
```

#### Command (Création d'Index)

```javascript
{
  "ts": Timestamp(1705320003, 1),
  "t": NumberLong(42),
  "h": NumberLong("-8765432109876543213"),
  "v": 2,
  "op": "c",
  "ns": "ecommerce.$cmd",
  "ui": UUID("550e8400-e29b-41d4-a716-446655440000"),
  "wall": ISODate("2024-01-15T10:00:03.012Z"),
  "o": {
    "createIndexes": "products",
    "v": 2,
    "key": { "category": 1 },
    "name": "category_1"
  }
}
```

#### No-op (Heartbeat)

```javascript
{
  "ts": Timestamp(1705320004, 1),
  "t": NumberLong(42),
  "h": NumberLong("0"),
  "v": 2,
  "op": "n",
  "ns": "",
  "wall": ISODate("2024-01-15T10:00:04.000Z"),
  "o": {
    "msg": "periodic noop"
  }
}
```

Les entrées no-op sont utilisées pour :
- Maintenir l'activité du heartbeat
- Forcer l'avancement du timestamp oplog
- Marquer des événements système (élections, etc.)

## Idempotence des Opérations

### Principe d'Idempotence

**Définition** : Une opération est idempotente si son application multiple produit le même résultat que son application unique.

L'oplog MongoDB garantit l'idempotence, ce qui permet aux Secondary de réappliquer des opérations sans risque de corruption.

### Transformation pour l'Idempotence

MongoDB transforme certaines opérations pour garantir l'idempotence :

#### Exemple 1 : Incrémentation

**Opération originale** (non idempotente) :
```javascript
db.counters.updateOne(
  { _id: "pageviews" },
  { $inc: { count: 1 } }
)
```

**Oplog enregistré** (idempotent) :
```javascript
{
  "op": "u",
  "ns": "stats.counters",
  "o": {
    "$v": 2,
    "diff": {
      "u": { "count": 1001 }  // Valeur absolue, pas incrémentation
    }
  },
  "o2": { "_id": "pageviews" }
}
```

Le Secondary applique la valeur absolue (1001), pas l'incrémentation (+1).

#### Exemple 2 : Push dans un Tableau

**Opération originale** :
```javascript
db.users.updateOne(
  { _id: ObjectId("...") },
  { $push: { tags: "vip" } }
)
```

**Oplog enregistré** :
```javascript
{
  "op": "u",
  "ns": "app.users",
  "o": {
    "$v": 2,
    "diff": {
      "u": {
        "tags": ["premium", "active", "vip"]  // Tableau complet
      }
    }
  },
  "o2": { "_id": ObjectId("...") }
}
```

#### Exemple 3 : UpdateMany

**Opération originale** :
```javascript
db.products.updateMany(
  { category: "Electronics" },
  { $set: { onSale: true } }
)
```

**Oplog enregistré** : Une entrée **par document modifié** :
```javascript
// Entrée 1
{
  "op": "u",
  "ns": "shop.products",
  "o": { "$v": 2, "diff": { "u": { "onSale": true } } },
  "o2": { "_id": ObjectId("65a5f1...001") }
}

// Entrée 2
{
  "op": "u",
  "ns": "shop.products",
  "o": { "$v": 2, "diff": { "u": { "onSale": true } } },
  "o2": { "_id": ObjectId("65a5f1...002") }
}
// ... une entrée par document
```

**Conséquence** : Les opérations multi-documents peuvent générer de nombreuses entrées oplog.

## Taille et Gestion de l'Oplog

### Dimensionnement de l'Oplog

La taille de l'oplog détermine combien de temps d'historique peut être conservé.

#### Taille par Défaut

| Système de Stockage | Taille par Défaut |
|---------------------|-------------------|
| Disque < 64 Go | ~990 Mo (5% de l'espace libre) |
| Disque ≥ 64 Go | 5% de l'espace libre |
| Windows | 5% de l'espace libre |
| Minimum | 990 Mo |
| Maximum (sans configuration) | 50 Go |

#### Vérifier la Taille Actuelle

```javascript
use local
db.oplog.rs.stats()
```

Résultat :
```javascript
{
  "ns": "local.oplog.rs",
  "size": 5368709120,        // Taille en bytes (~5 GB)
  "count": 1234567,          // Nombre d'entrées
  "avgObjSize": 435,         // Taille moyenne d'une entrée
  "storageSize": 5368709120,
  "capped": true,
  "max": -1,                 // Pas de limite sur le nombre d'entrées
  "maxSize": 5368709120,     // Limite de taille
  ...
}
```

#### Calculer la Fenêtre Oplog

```javascript
rs.printReplicationInfo()
```

Résultat :
```
configured oplog size:   5120MB
log length start to end: 86394secs (23.99hrs)
oplog first event time:  Mon Jan 15 2024 10:00:00 GMT+0000 (UTC)
oplog last event time:   Tue Jan 16 2024 10:59:54 GMT+0000 (UTC)
now:                     Tue Jan 16 2024 11:00:00 GMT+0000 (UTC)
```

**Interprétation** :
- L'oplog couvre ~24 heures d'opérations
- Si un Secondary est déconnecté plus de 24h, il ne pourra pas se resynchroniser via l'oplog

### Redimensionner l'Oplog

#### Augmenter la Taille (MongoDB 4.0+)

Sur un membre à la fois (en commençant par les Secondary) :

```javascript
// 1. Se connecter au membre
mongosh --host mongodb-secondary-01:27017

// 2. Vérifier l'état actuel
use local
db.oplog.rs.stats().maxSize / 1024 / 1024 / 1024  // Taille en GB

// 3. Redimensionner (exemple : 10 GB)
db.adminCommand({
  replSetResizeOplog: 1,
  size: 10240  // Taille en MB
})

// 4. Vérifier le changement
db.oplog.rs.stats().maxSize / 1024 / 1024 / 1024
```

**Important** :
- Opération en ligne (pas d'arrêt nécessaire)
- Effectuer sur chaque membre individuellement
- Commencer par les Secondary, finir par le Primary
- Le changement est permanent

#### Réduire la Taille

La réduction nécessite une approche différente :

```javascript
// Option 1 : Via replSetResizeOplog (MongoDB 5.0+)
db.adminCommand({
  replSetResizeOplog: 1,
  size: 2048,        // Nouvelle taille en MB
  minRetentionHours: 1  // Durée minimale de rétention
})

// Option 2 : Méthode manuelle (versions antérieures)
// Nécessite de supprimer et recréer l'oplog (procédure complexe)
```

### Calcul de la Taille Optimale

**Formule de base** :
```
Taille oplog (GB) = Taux d'écriture (GB/h) × Fenêtre souhaitée (h) × Facteur de sécurité
```

**Exemple** :

```javascript
// Mesurer le taux d'écriture pendant 1 heure
var startSize = db.oplog.rs.stats().size;
var startTime = new Date();

// Attendre 1 heure...
// sleep(3600000);

var endSize = db.oplog.rs.stats().size;
var endTime = new Date();

var duration = (endTime - startTime) / 1000 / 3600;  // heures
var growthRate = (endSize - startSize) / 1024 / 1024 / 1024 / duration;  // GB/h

print("Taux de croissance : " + growthRate + " GB/h");

// Pour une fenêtre de 48h avec facteur 1.5
var recommendedSize = growthRate * 48 * 1.5;
print("Taille recommandée : " + recommendedSize + " GB");
```

**Facteurs à considérer** :
- **Charge d'écriture** : Plus d'écritures = oplog plus grand
- **Fenêtre de maintenance** : Durée maximale de déconnexion acceptable
- **Opérations volumineuses** : Updates multi-documents, migrations
- **Marge de sécurité** : Prévoir 50-100% de marge

### Compression de l'Oplog

Depuis MongoDB 4.4, l'oplog peut être compressé avec WiredTiger :

```javascript
// Dans mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: snappy  // ou zlib, zstd

// Vérifier la compression
use local
db.oplog.rs.stats().wiredTiger.block-manager["blocks compressed"]
```

**Algorithmes** :
- `snappy` : Rapide, compression modérée (par défaut)
- `zlib` : Plus lent, meilleure compression
- `zstd` : Bon compromis vitesse/taille (MongoDB 4.2+)

## Processus de Réplication

### Mécanisme de Synchronisation

```
┌─────────────┐                    ┌──────────────┐
│   PRIMARY   │                    │  SECONDARY   │
│             │                    │              │
│  1. Write   │                    │              │
│     ↓       │                    │              │
│  2. Oplog   │                    │              │
│     Entry   │                    │              │
└──────┬──────┘                    └──────┬───────┘
       │                                  │
       │  3. Fetch oplog entries          │
       │  ←────────────────────────────── │
       │                                  │
       │  4. Return entries               │
       │  ─────────────────────────────→  │
       │                                  │
       │                                  ↓
       │                           5. Apply operations
       │                                  ↓
       │                           6. Update local oplog
```

### Étapes Détaillées

#### 1. Écriture sur le Primary

```javascript
// Application exécute
db.orders.insertOne({
  orderId: 12345,
  customer: "Alice",
  amount: 99.99,
  timestamp: new Date()
})
```

#### 2. Enregistrement dans l'Oplog du Primary

Le Primary crée une entrée oplog :
```javascript
{
  "ts": Timestamp(1705320100, 1),
  "t": NumberLong(42),
  "op": "i",
  "ns": "shop.orders",
  "o": {
    "_id": ObjectId("..."),
    "orderId": 12345,
    "customer": "Alice",
    "amount": 99.99,
    "timestamp": ISODate("2024-01-15T10:01:40Z")
  }
}
```

#### 3. Lecture par les Secondary (Tailing)

Chaque Secondary maintient un curseur tailable sur `oplog.rs` du Primary :

```javascript
// Simplifié (géré par MongoDB)
var cursor = db.getSiblingDB('local').oplog.rs.find({
  ts: { $gt: lastAppliedTimestamp }
}).tailable().awaitData();

cursor.forEach(entry => {
  applyOperation(entry);
});
```

#### 4. Application des Opérations

Le Secondary applique l'opération de manière idempotente :
```javascript
// Même opération que sur le Primary
db.orders.insertOne({
  "_id": ObjectId("..."),  // Même _id
  "orderId": 12345,
  "customer": "Alice",
  "amount": 99.99,
  "timestamp": ISODate("2024-01-15T10:01:40Z")
})
```

#### 5. Enregistrement dans l'Oplog Local

Le Secondary enregistre l'opération dans son propre oplog local.

### Réplication Chaînée (Chained Replication)

Par défaut, les Secondary peuvent lire l'oplog d'autres Secondary :

```
PRIMARY (A)
    ↓
SECONDARY (B) ←─── lit depuis A
    ↓
SECONDARY (C) ←─── lit depuis B (chaîné)
```

**Avantages** :
- Réduit la charge sur le Primary
- Efficace pour les déploiements géographiques

**Configuration** :
```javascript
// Désactiver la réplication chaînée
cfg = rs.conf()
cfg.settings.chainingAllowed = false
rs.reconfig(cfg)
```

## Monitoring de l'Oplog

### Vérifier le Lag de Réplication

#### Méthode 1 : rs.printReplicationInfo()

Sur le Primary :
```javascript
rs.printReplicationInfo()
```

Résultat :
```
configured oplog size:   5120MB
log length start to end: 86394secs (23.99hrs)
oplog first event time:  Mon Jan 15 2024 10:00:00 GMT+0000
oplog last event time:   Tue Jan 16 2024 10:59:54 GMT+0000
now:                     Tue Jan 16 2024 11:00:00 GMT+0000
```

#### Méthode 2 : rs.printSecondaryReplicationInfo()

Sur le Primary :
```javascript
rs.printSecondaryReplicationInfo()
```

Résultat :
```
source: mongodb-secondary-01:27017
  syncedTo: Tue Jan 16 2024 10:59:50 GMT+0000
  0 secs (0 hrs) behind the primary

source: mongodb-secondary-02:27017
  syncedTo: Tue Jan 16 2024 10:59:45 GMT+0000
  5 secs (0 hrs) behind the primary
```

#### Méthode 3 : rs.status()

```javascript
rs.status().members.forEach(member => {
  print(member.name + " - Lag: " +
    (member.optimeDate ?
      ((new Date() - member.optimeDate) / 1000) + " sec" :
      "N/A")
  );
});
```

### Requêtes Oplog Personnalisées

#### Analyser les Types d'Opérations

```javascript
use local

// Compter par type d'opération
db.oplog.rs.aggregate([
  {
    $group: {
      _id: "$op",
      count: { $sum: 1 }
    }
  },
  {
    $sort: { count: -1 }
  }
])
```

Résultat :
```javascript
[
  { "_id": "u", "count": 450000 },  // Updates
  { "_id": "i", "count": 150000 },  // Inserts
  { "_id": "n", "count": 43200 },   // No-ops (heartbeats)
  { "_id": "d", "count": 25000 },   // Deletes
  { "_id": "c", "count": 150 }      // Commands
]
```

#### Opérations par Namespace

```javascript
db.oplog.rs.aggregate([
  { $match: { op: { $in: ["i", "u", "d"] } } },
  {
    $group: {
      _id: "$ns",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

#### Opérations Récentes sur une Collection

```javascript
db.oplog.rs.find({
  ns: "mydb.users",
  op: { $in: ["i", "u", "d"] }
}).sort({ ts: -1 }).limit(10).pretty()
```

#### Taille Moyenne des Entrées par Type

```javascript
db.oplog.rs.aggregate([
  {
    $group: {
      _id: "$op",
      avgSize: { $avg: { $bsonSize: "$$ROOT" } },
      count: { $sum: 1 }
    }
  }
])
```

### Métriques à Surveiller

| Métrique | Commande | Seuil d'Alerte |
|----------|----------|----------------|
| **Replication Lag** | `rs.status()` | > 60 secondes |
| **Oplog Window** | `rs.printReplicationInfo()` | < 24 heures |
| **Oplog Growth Rate** | Stats sur 1h | Croissance anormale |
| **Oplog Size** | `db.oplog.rs.stats().size` | > 80% de maxSize |

## Cas Particuliers et Problèmes

### Oplog Trop Petit

**Symptômes** :
- Secondary ne peut pas rattraper le Primary
- Erreur : "too stale to catch up"
- Nécessité de resynchronisation complète

**Diagnostic** :
```javascript
rs.printReplicationInfo()
// Si "log length" < "replication lag" → oplog trop petit
```

**Solution** :
```javascript
// 1. Augmenter la taille de l'oplog
db.adminCommand({ replSetResizeOplog: 1, size: 10240 })

// 2. Si trop tard, resynchronisation nécessaire
rs.remove("mongodb-secondary-02:27017")
// Supprimer les données et relancer
rs.add("mongodb-secondary-02:27017")
```

### Opérations Volumineuses

Les grandes opérations peuvent saturer l'oplog rapidement :

```javascript
// Mauvais : updateMany sur des millions de documents
db.products.updateMany(
  {},
  { $set: { reviewed: true } }
)
// Génère des millions d'entrées oplog
```

**Solution** : Traitement par lots
```javascript
// Bon : Traitement par lots
var batch = 1000;
var query = { reviewed: { $ne: true } };

while (db.products.countDocuments(query) > 0) {
  var ids = db.products.find(query, { _id: 1 })
    .limit(batch)
    .toArray()
    .map(doc => doc._id);

  db.products.updateMany(
    { _id: { $in: ids } },
    { $set: { reviewed: true } }
  );

  sleep(100);  // Pause pour ne pas saturer
}
```

### Rollback et Oplog

Lors d'un rollback (voir section 9.4), MongoDB :

1. Identifie les opérations divergentes dans l'oplog
2. Annule ces opérations
3. Sauvegarde les documents affectés dans `/data/rollback/`
4. Applique les opérations du nouveau Primary

**Visualiser les opérations rollbackées** :
```bash
ls -lh /data/db/rollback/
# Contient des fichiers .bson avec les documents annulés
```

### Oplog et Transactions Multi-Documents

Les transactions génèrent plusieurs entrées oplog :

```javascript
// Transaction
session.startTransaction();
db.accounts.updateOne({ _id: "A" }, { $inc: { balance: -100 } });
db.accounts.updateOne({ _id: "B" }, { $inc: { balance: 100 } });
session.commitTransaction();
```

**Oplog généré** :
```javascript
// Entrée 1 : applyOps (transaction commit)
{
  "op": "c",
  "ns": "admin.$cmd",
  "o": {
    "applyOps": [
      {
        "op": "u",
        "ns": "banking.accounts",
        "o": { /* update A */ },
        "o2": { "_id": "A" }
      },
      {
        "op": "u",
        "ns": "banking.accounts",
        "o": { /* update B */ },
        "o2": { "_id": "B" }
      }
    ]
  },
  "lsid": { /* session id */ },
  "txnNumber": NumberLong(1)
}
```

Les transactions sont atomiques dans l'oplog via la commande `applyOps`.

## Oplog et Outils Externes

### Change Streams

Les Change Streams utilisent l'oplog en interne :

```javascript
const changeStream = db.orders.watch();

changeStream.on('change', (change) => {
  console.log('Detected change:', change);
});
```

Change Stream lit l'oplog mais fournit une API de plus haut niveau.

### MongoDB Connector for Apache Kafka

Le connecteur lit l'oplog pour diffuser les changements vers Kafka :

```json
{
  "connector.class": "com.mongodb.kafka.connect.MongoSourceConnector",
  "connection.uri": "mongodb://localhost:27017",
  "database": "mydb",
  "collection": "users",
  "publish.full.document.only": true
}
```

### Outils de Backup Incrémental

Certains outils utilisent l'oplog pour les sauvegardes incrémentielles :

```bash
# mongodump avec oplog
mongodump --oplog --out=/backup/full

# Sauvegarde incrémentielle via oplog
mongodump --oplog --query='{"ts":{$gt:Timestamp(1705320000,1)}}'
```

## Bonnes Pratiques

### 1. Dimensionnement Approprié

```javascript
// Règle générale : fenêtre oplog ≥ 3× durée de maintenance
// Exemple : maintenance de 4h → oplog de 12h minimum

// Vérification mensuelle
rs.printReplicationInfo()
// Si oplog window < seuil → redimensionner
```

### 2. Monitoring Proactif

```javascript
// Script de monitoring (à exécuter régulièrement)
function checkOplogHealth() {
  var status = rs.status();
  var info = db.getReplicationInfo();

  var warnings = [];

  // Vérifier la fenêtre oplog
  var oplogHours = info.timeDiff / 3600;
  if (oplogHours < 24) {
    warnings.push("Oplog window < 24h: " + oplogHours + "h");
  }

  // Vérifier le lag
  status.members.forEach(member => {
    if (member.state === 2) {  // SECONDARY
      var lag = (status.date - member.optimeDate) / 1000;
      if (lag > 60) {
        warnings.push(member.name + " lag: " + lag + "s");
      }
    }
  });

  return warnings.length === 0 ? "OK" : warnings;
}

checkOplogHealth();
```

### 3. Éviter les Opérations Massives

```javascript
// ❌ Mauvais
db.huge_collection.updateMany({}, { $set: { migrated: true } });

// ✅ Bon
var cursor = db.huge_collection.find({ migrated: { $ne: true } });
var batch = [];
var batchSize = 1000;

cursor.forEach(doc => {
  batch.push(doc._id);

  if (batch.length >= batchSize) {
    db.huge_collection.updateMany(
      { _id: { $in: batch } },
      { $set: { migrated: true } }
    );
    batch = [];
    sleep(100);
  }
});

// Derniers documents
if (batch.length > 0) {
  db.huge_collection.updateMany(
    { _id: { $in: batch } },
    { $set: { migrated: true } }
  );
}
```

### 4. Compression pour Réduire l'Espace

```yaml
# mongod.conf
storage:
  wiredTiger:
    collectionConfig:
      blockCompressor: zstd  # Meilleure compression
```

### 5. Retention Minimale

Depuis MongoDB 5.0, définir une rétention minimale :

```javascript
db.adminCommand({
  replSetResizeOplog: 1,
  size: 10240,
  minRetentionHours: 24  // Garder au moins 24h même si l'oplog est plein
})
```

### 6. Documentation des Fenêtres de Maintenance

```javascript
// Documenter les besoins métier
var maintenanceWindows = {
  "weekly_backup": { duration: 4, frequency: "weekly" },
  "monthly_migration": { duration: 8, frequency: "monthly" },
  "emergency_restore": { duration: 12, frequency: "rare" }
};

// Calculer l'oplog requis
var maxDuration = 12;  // heures
var safetyFactor = 3;
var requiredOplogWindow = maxDuration * safetyFactor;

print("Oplog window required: " + requiredOplogWindow + " hours");
```

## Analyse Forensique avec l'Oplog

### Retrouver une Opération Spécifique

```javascript
// Trouver quand un document a été supprimé
use local
db.oplog.rs.find({
  ns: "mydb.users",
  op: "d",
  "o._id": ObjectId("65a5f1234567890abcdef012")
}).pretty()
```

### Reconstruire l'Historique d'un Document

```javascript
// Toutes les opérations sur un document
db.oplog.rs.find({
  ns: "mydb.users",
  $or: [
    { "o._id": ObjectId("65a5f1234567890abcdef012") },
    { "o2._id": ObjectId("65a5f1234567890abcdef012") }
  ]
}).sort({ ts: 1 }).pretty()
```

### Audit des Modifications

```javascript
// Modifications dans une plage de temps
var startTime = Timestamp(1705320000, 0);
var endTime = Timestamp(1705406400, 0);

db.oplog.rs.find({
  ns: "sensitive.data",
  op: { $in: ["u", "d"] },
  ts: { $gte: startTime, $lte: endTime }
}).sort({ ts: 1 })
```

## Conclusion

L'oplog est le cœur battant de la réplication MongoDB. Sa maîtrise est essentielle pour :

- ✅ **Garantir la haute disponibilité** via une réplication fiable
- ✅ **Diagnostiquer les problèmes** de lag et de synchronisation
- ✅ **Optimiser les performances** en dimensionnant correctement
- ✅ **Implémenter des sauvegardes** incrémentielles et PITR
- ✅ **Intégrer avec des systèmes externes** (Kafka, CDC, etc.)
- ✅ **Effectuer des audits** et analyses forensiques

**Points clés à retenir** :

1. L'oplog est une capped collection FIFO dans `local.oplog.rs`
2. Toutes les opérations sont transformées pour être idempotentes
3. La taille de l'oplog détermine la fenêtre de récupération
4. Le monitoring du lag et de la fenêtre est critique
5. Les opérations volumineuses doivent être traitées par lots
6. La compression peut réduire significativement l'espace utilisé

Une bonne gestion de l'oplog est un pilier de la fiabilité et de la résilience des déploiements MongoDB en production.

⏭️ [Configuration d'un Replica Set](/09-replication/06-configuration-replica-set.md)
