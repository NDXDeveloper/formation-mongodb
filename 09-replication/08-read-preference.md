🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.8 Read Preference

## Introduction

Le **Read Preference** est un mécanisme permettant de contrôler depuis quels membres d'un Replica Set les opérations de lecture doivent être exécutées. Ce concept est fondamental pour optimiser les performances, répartir la charge, garantir la disponibilité et gérer les compromis entre cohérence et latence dans les architectures distribuées.

## Concepts Fondamentaux

### Définition

Le Read Preference détermine :
1. **Quel type de membre** peut servir les lectures (Primary, Secondary, ou les deux)
2. **Quel membre spécifique** choisir parmi les candidats éligibles
3. **Quelles garanties de cohérence** sont acceptables

### Compromis CAP et Read Preference

Dans le contexte du théorème CAP, le Read Preference influence le positionnement :

```
┌─────────────────────────────────────────────┐
│         Théorème CAP                        │
├─────────────────────────────────────────────┤
│  C (Consistency)    ←→    A (Availability)  │
│                                             │
│  primary            →  primaryPreferred     │
│  (forte cohérence)  →  secondary            │
│                     →  secondaryPreferred   │
│                     →  nearest              │
│                        (haute disponibilité)│
└─────────────────────────────────────────────┘
```

### Architecture et Flux

```
Application
    ↓ (Connection String avec readPreference)
Driver MongoDB
    ↓ (Sélection du membre selon la stratégie)
Replica Set
    ├── PRIMARY (accepte toujours les écritures)
    ├── SECONDARY-1 (peut servir certaines lectures)
    ├── SECONDARY-2 (peut servir certaines lectures)
    └── SECONDARY-3 (peut servir certaines lectures)
```

## Modes de Read Preference

### 1. primary (Défaut)

Toutes les lectures sont dirigées vers le Primary.

```javascript
// mongosh
db.collection.find().readPref("primary")

// Node.js Driver
const collection = db.collection('users')
const docs = await collection.find({}).readPreference('primary').toArray()
```

**Caractéristiques** :

| Aspect | Comportement |
|--------|--------------|
| **Cohérence** | Forte - Lectures toujours à jour |
| **Disponibilité** | Réduite - Échec si Primary down |
| **Performance** | Charge concentrée sur le Primary |
| **Latence** | Dépend de la proximité du Primary |
| **Cas d'usage** | Données critiques, forte cohérence requise |

**Comportement en cas de défaillance** :
```javascript
// Si le Primary tombe
// → Erreur : "No primary available"
// → Attendre l'élection d'un nouveau Primary
```

**Exemple** :
```javascript
// Opérations financières critiques
db.transactions.insertOne({
  from: "account-A",
  to: "account-B",
  amount: 1000
}, { writeConcern: { w: "majority" } })

// Lecture immédiate pour vérification
db.transactions.findOne(
  { _id: transactionId },
  { readPreference: "primary" }
)
// Garantit que la lecture voit l'écriture précédente
```

### 2. primaryPreferred

Préfère le Primary, mais bascule sur un Secondary si le Primary est indisponible.

```javascript
db.collection.find().readPref("primaryPreferred")
```

**Caractéristiques** :

| Aspect | Comportement |
|--------|--------------|
| **Cohérence** | Forte en temps normal, eventual si Primary down |
| **Disponibilité** | Haute - Bascule automatique |
| **Performance** | Charge sur Primary par défaut |
| **Latence** | Variable selon disponibilité |
| **Cas d'usage** | Préférence pour cohérence avec failover automatique |

**Flux de décision** :
```
Primary disponible ?
  ├── Oui → Lire depuis Primary
  └── Non → Lire depuis Secondary
```

**Exemple** :
```javascript
// Dashboard administratif
// Préfère les données à jour, mais tolère un léger délai
db.stats.aggregate([
  { $match: { date: today } },
  { $group: { _id: "$category", total: { $sum: "$amount" } } }
], {
  readPreference: { mode: "primaryPreferred" }
})
```

### 3. secondary

Toutes les lectures sont dirigées vers les membres Secondary.

```javascript
db.collection.find().readPref("secondary")
```

**Caractéristiques** :

| Aspect | Comportement |
|--------|--------------|
| **Cohérence** | Eventual - Possible replication lag |
| **Disponibilité** | Haute - Multiple Secondary |
| **Performance** | Décharge le Primary |
| **Latence** | Dépend de la proximité des Secondary |
| **Cas d'usage** | Analytics, reporting, lectures non-critiques |

**Comportement en cas de défaillance** :
```javascript
// Si tous les Secondary sont down
// → Erreur : "No secondary available"
// Le Primary N'EST PAS utilisé
```

**Exemple** :
```javascript
// Requêtes analytics lourdes
db.orders.aggregate([
  {
    $match: {
      orderDate: { $gte: ISODate("2024-01-01") }
    }
  },
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m", date: "$orderDate" } },
      revenue: { $sum: "$amount" },
      count: { $sum: 1 }
    }
  },
  {
    $sort: { _id: 1 }
  }
], {
  readPreference: { mode: "secondary" },
  allowDiskUse: true
})
```

### 4. secondaryPreferred

Préfère les Secondary, mais bascule sur le Primary si aucun Secondary n'est disponible.

```javascript
db.collection.find().readPref("secondaryPreferred")
```

**Caractéristiques** :

| Aspect | Comportement |
|--------|--------------|
| **Cohérence** | Eventual en temps normal, forte si tous Secondary down |
| **Disponibilité** | Très haute - Fallback vers Primary |
| **Performance** | Décharge maximale du Primary |
| **Latence** | Optimisée pour Secondary |
| **Cas d'usage** | Lectures fréquentes, tolérance au lag acceptable |

**Flux de décision** :
```
Secondary disponible ?
  ├── Oui → Lire depuis Secondary
  └── Non → Lire depuis Primary (fallback)
```

**Exemple** :
```javascript
// API publique de recherche de produits
// Tolère un léger lag, décharge le Primary
db.products.find(
  {
    $text: { $search: searchQuery },
    inStock: true
  },
  {
    readPreference: { mode: "secondaryPreferred" }
  }
).limit(20)
```

### 5. nearest

Sélectionne le membre avec la latence réseau la plus faible, qu'il soit Primary ou Secondary.

```javascript
db.collection.find().readPref("nearest")
```

**Caractéristiques** :

| Aspect | Comportement |
|--------|--------------|
| **Cohérence** | Variable - Dépend du membre sélectionné |
| **Disponibilité** | Très haute - Tous les membres éligibles |
| **Performance** | Optimale - Basée sur latence |
| **Latence** | Minimale - Membre le plus proche |
| **Cas d'usage** | Applications géo-distribuées, latence critique |

**Mécanisme de sélection** :
```
1. Mesurer la latence réseau vers tous les membres
2. Sélectionner le membre avec latency minimale
3. Considérer une fenêtre de tolérance (localThresholdMS)
```

**Exemple** :
```javascript
// Application mobile géo-distribuée
// Utilisateurs répartis globalement
const client = new MongoClient(uri, {
  readPreference: { mode: 'nearest' },
  localThresholdMS: 15  // Tolérance de 15ms
})

// Lecture depuis le membre le plus proche
db.users.findOne(
  { userId: currentUserId },
  { readPreference: "nearest" }
)
```

## Tags et Sélection Ciblée

### Configuration des Tags

Les tags permettent de cibler des sous-ensembles spécifiques de membres.

#### Configuration dans le Replica Set

```javascript
cfg = rs.conf()

cfg.members = [
  {
    _id: 0,
    host: "dc1-mongodb-01:27017",
    tags: {
      dc: "east",
      region: "us-east-1",
      zone: "us-east-1a",
      nodeType: "ssd",
      workload: "production"
    }
  },
  {
    _id: 1,
    host: "dc1-mongodb-02:27017",
    tags: {
      dc: "east",
      region: "us-east-1",
      zone: "us-east-1b",
      nodeType: "ssd",
      workload: "production"
    }
  },
  {
    _id: 2,
    host: "dc2-mongodb-01:27017",
    tags: {
      dc: "west",
      region: "us-west-1",
      zone: "us-west-1a",
      nodeType: "standard",
      workload: "production"
    }
  },
  {
    _id: 3,
    host: "analytics-mongodb:27017",
    tags: {
      dc: "east",
      region: "us-east-1",
      zone: "us-east-1c",
      nodeType: "standard",
      workload: "analytics"
    }
  }
]

rs.reconfig(cfg)
```

### Utilisation des Tags

#### Syntaxe avec Tags

```javascript
// mongosh
db.collection.find().readPref("secondary", [
  { dc: "east" },           // Préférence 1 : DC east
  { region: "us-west-1" }   // Préférence 2 : Region west si east indisponible
])
```

#### Node.js Driver

```javascript
const { ReadPreference } = require('mongodb')

// Lecture depuis datacenter spécifique
const readPref = new ReadPreference('secondary', [
  { dc: 'east' },
  { dc: 'west' }
])

const results = await collection.find({}).readPreference(readPref).toArray()
```

#### Python (PyMongo)

```python
from pymongo import ReadPreference
from pymongo.read_preferences import Secondary

# Tag sets pour sélection
tag_sets = [
    {'dc': 'east', 'nodeType': 'ssd'},
    {'dc': 'east'},
    {'dc': 'west'}
]

cursor = collection.find(
    {},
    read_preference=Secondary(tag_sets=tag_sets)
)
```

### Cas d'Usage avec Tags

#### 1. Isolation des Workloads

```javascript
// Production : Lire depuis nœuds production
db.orders.find({}).readPref("secondary", [
  { workload: "production" }
])

// Analytics : Lire depuis nœuds analytics
db.orders.aggregate(
  [ /* pipeline complexe */ ],
  {
    readPreference: {
      mode: "secondary",
      tags: [{ workload: "analytics" }]
    }
  }
)
```

#### 2. Localité Géographique

```javascript
// Application US East
const usEastReadPref = {
  mode: "nearest",
  tags: [
    { region: "us-east-1" },  // Préféré
    { dc: "east" },           // Fallback
    {}                        // Tout datacenter si nécessaire
  ]
}

db.products.find({}).readPreference(usEastReadPref)
```

#### 3. Performance Hardware

```javascript
// Requêtes lourdes sur hardware puissant
db.logs.aggregate(
  [ /* aggregation complexe */ ],
  {
    readPreference: {
      mode: "secondary",
      tags: [
        { nodeType: "ssd", workload: "analytics" }
      ]
    },
    allowDiskUse: true
  }
)
```

## maxStalenessSeconds

Paramètre limitant l'ancienneté acceptable des données sur un Secondary.

### Principe

```javascript
db.collection.find().readPref("secondary", [], {
  maxStalenessSeconds: 90  // Secondary avec lag ≤ 90 secondes
})
```

**Mécanisme** :
```
Pour chaque Secondary :
  staleness = (lastWriteDate - lastWriteDateSecondary) + heartbeatFrequency

Si staleness > maxStalenessSeconds :
  → Exclure ce Secondary de la sélection
```

### Configuration

```javascript
// mongosh
db.collection.find({}).readPref("secondaryPreferred", [], {
  maxStalenessSeconds: 120
})

// Node.js
const readPref = new ReadPreference('secondaryPreferred', [], {
  maxStalenessSeconds: 120
})

// Connection String
mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondaryPreferred&maxStalenessSeconds=120
```

### Contraintes

**Valeurs minimales** :

| Configuration | Valeur Minimale |
|---------------|-----------------|
| Avec heartbeat par défaut (10s) | 90 secondes |
| heartbeatFrequencyMS custom | (heartbeatFrequencyMS × 10) + 10000 ms |

**Exemple d'erreur** :
```javascript
// Erreur : maxStalenessSeconds trop faible
db.collection.find().readPref("secondary", [], {
  maxStalenessSeconds: 30  // < 90 secondes
})

// → Error: maxStalenessSeconds must be at least 90 seconds
```

### Cas d'Usage

```javascript
// Dashboard temps quasi-réel
// Tolère 2 minutes de lag maximum
db.metrics.find(
  { timestamp: { $gte: fiveMinutesAgo } },
  {
    readPreference: {
      mode: "secondaryPreferred",
      maxStalenessSeconds: 120
    }
  }
)
```

**Comportement** :
```
Secondary-1 : lag = 60s  → Éligible
Secondary-2 : lag = 150s → Exclu
Secondary-3 : lag = 45s  → Éligible

Sélection entre Secondary-1 et Secondary-3
```

## Hedged Reads

Depuis MongoDB 4.4, optimisation automatique des lectures en mode `nearest`.

### Principe

```
Client envoie requête
    ↓
Driver envoie à 2 membres simultanément
    ├── Membre A (le plus proche mesuré)
    └── Membre B (2ème plus proche)

Premier à répondre → Résultat retourné
Second annulé
```

**Activation** :
```javascript
// Activé automatiquement avec mode "nearest"
db.collection.find().readPref("nearest")

// Ou explicitement
const readPref = new ReadPreference('nearest', [], {
  hedge: { enabled: true }
})
```

**Avantages** :
- Réduit la latence P99
- Compense les variations réseau
- Aucun impact sur la cohérence

## Read Preference dans Différents Contextes

### Connection String

```javascript
// Format général
mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=MODE&readPreferenceTags=TAG_SET

// Exemples
// 1. Primary (défaut)
mongodb://host1,host2,host3/?replicaSet=rs0

// 2. Secondary
mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondary

// 3. SecondaryPreferred avec tags
mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondaryPreferred&readPreferenceTags=dc:east,nodeType:ssd&readPreferenceTags=dc:east&readPreferenceTags=

// 4. Nearest avec maxStaleness
mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=nearest&maxStalenessSeconds=90
```

### Node.js Driver

```javascript
const { MongoClient, ReadPreference } = require('mongodb')

// Méthode 1 : Connection String
const client = new MongoClient(
  'mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondaryPreferred'
)

// Méthode 2 : Options de connexion
const client = new MongoClient(uri, {
  readPreference: 'secondaryPreferred'
})

// Méthode 3 : Niveau database
const db = client.db('mydb', {
  readPreference: new ReadPreference('secondary', [{ dc: 'east' }])
})

// Méthode 4 : Niveau collection
const collection = db.collection('users', {
  readPreference: 'nearest'
})

// Méthode 5 : Niveau requête
const results = await collection.find({}).readPreference('secondary').toArray()
```

### Python (PyMongo)

```python
from pymongo import MongoClient, ReadPreference

# Méthode 1 : Connection String
client = MongoClient('mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondary')

# Méthode 2 : Options
client = MongoClient(
    'mongodb://host1,host2,host3/',
    replicaset='rs0',
    read_preference=ReadPreference.SECONDARY
)

# Méthode 3 : Niveau collection
collection = db.users.with_options(
    read_preference=ReadPreference.SECONDARY_PREFERRED
)

# Méthode 4 : Avec tags
from pymongo.read_preferences import SecondaryPreferred

tag_sets = [{'dc': 'east'}, {'dc': 'west'}]
collection = db.users.with_options(
    read_preference=SecondaryPreferred(tag_sets=tag_sets)
)
```

### Java Driver

```java
import com.mongodb.ReadPreference;
import com.mongodb.TagSet;
import com.mongodb.Tag;

// Méthode 1 : Connection String
MongoClient client = MongoClients.create(
    "mongodb://host1,host2,host3/?replicaSet=rs0&readPreference=secondaryPreferred"
);

// Méthode 2 : ReadPreference API
MongoClientSettings settings = MongoClientSettings.builder()
    .applyConnectionString(new ConnectionString(uri))
    .readPreference(ReadPreference.secondaryPreferred())
    .build();

// Méthode 3 : Avec tags
List<TagSet> tagSets = Arrays.asList(
    new TagSet(new Tag("dc", "east")),
    new TagSet(new Tag("dc", "west"))
);

ReadPreference readPref = ReadPreference.secondary(tagSets);

MongoCollection<Document> collection = database
    .getCollection("users")
    .withReadPreference(readPref);
```

### Transactions Multi-Documents

**Important** : Les transactions DOIVENT utiliser le Primary pour toutes les opérations.

```javascript
// ❌ Incorrect - Erreur
session.startTransaction({
  readPreference: { mode: 'secondary' }  // Non autorisé
})

// ✅ Correct - Primary uniquement
session.startTransaction({
  readConcern: { level: 'snapshot' },
  writeConcern: { w: 'majority' }
  // readPreference implicitement "primary"
})

db.accounts.findOne({ _id: accountA }, { session })
// Lecture depuis le Primary même si readPreference global est "secondary"
```

## Impact sur la Cohérence

### Modèles de Cohérence

```
┌─────────────────────────────────────────────┐
│    Read Preference et Cohérence             │
├─────────────────────────────────────────────┤
│  primary            → Linearizable          │
│                       (cohérence forte)     │
│                                             │
│  primaryPreferred   → Session Causal        │
│                       (si Primary up)       │
│                                             │
│  secondary          → Eventual              │
│  secondaryPreferred   (cohérence faible)    │
│  nearest                                    │
└─────────────────────────────────────────────┘
```

### Read-Your-Writes

Garantir qu'une lecture voit les écritures précédentes de la même session :

```javascript
// ❌ Problème : Lecture peut ne pas voir l'écriture
db.users.updateOne({ _id: userId }, { $set: { status: "active" } })
// Écriture sur Primary

const user = db.users.findOne(
  { _id: userId },
  { readPreference: "secondary" }
)
// Lecture depuis Secondary - peut ne pas voir l'update (lag)
```

**Solutions** :

#### Solution 1 : Read Concern "majority" + Write Concern "majority"

```javascript
// Écriture avec majority
db.users.updateOne(
  { _id: userId },
  { $set: { status: "active" } },
  { writeConcern: { w: "majority" } }
)

// Lecture avec majority depuis Secondary
const user = db.users.findOne(
  { _id: userId },
  {
    readPreference: "secondary",
    readConcern: { level: "majority" }
  }
)
```

#### Solution 2 : Causal Consistency (Sessions)

```javascript
const session = client.startSession({ causalConsistency: true })

// Écriture
db.users.updateOne(
  { _id: userId },
  { $set: { status: "active" } },
  { session }
)

// Lecture - Voit automatiquement l'écriture précédente
const user = db.users.findOne(
  { _id: userId },
  {
    session,
    readPreference: "secondaryPreferred"
  }
)
```

#### Solution 3 : Lire depuis Primary après Écriture

```javascript
// Écriture
db.users.updateOne({ _id: userId }, { $set: { status: "active" } })

// Lecture immédiate depuis Primary
const user = db.users.findOne(
  { _id: userId },
  { readPreference: "primary" }
)

// Lectures suivantes peuvent utiliser Secondary
```

### Monotonic Reads

Garantir que les lectures successives voient un état qui progresse (jamais de régression) :

```javascript
// Problème potentiel
const session = client.startSession()

// Lecture 1 : Secondary-1 (lag = 10s)
const data1 = db.collection.findOne({ _id: id }, {
  session,
  readPreference: "secondary"
})

// Lecture 2 : Secondary-2 (lag = 30s - plus ancien!)
const data2 = db.collection.findOne({ _id: id }, {
  session,
  readPreference: "secondary"
})
// data2 peut être plus ancien que data1 !
```

**Solution : Causal Consistency**

```javascript
const session = client.startSession({ causalConsistency: true })

// Les lectures suivantes verront toujours un état ≥ lecture précédente
```

## Cas d'Usage Avancés

### 1. Architecture Multi-Région

```javascript
// Configuration Replica Set
{
  members: [
    // Region US
    { _id: 0, host: "us-01:27017", tags: { region: "us", dc: "us-east" } },
    { _id: 1, host: "us-02:27017", tags: { region: "us", dc: "us-west" } },

    // Region EU
    { _id: 2, host: "eu-01:27017", tags: { region: "eu", dc: "eu-central" } },
    { _id: 3, host: "eu-02:27017", tags: { region: "eu", dc: "eu-west" } },

    // Region APAC
    { _id: 4, host: "apac-01:27017", tags: { region: "apac", dc: "ap-southeast" } }
  ]
}

// Application US
const usReadPref = {
  mode: "nearest",
  tags: [
    { region: "us" },      // Préféré
    { region: "eu" },      // Fallback 1
    { region: "apac" }     // Fallback 2
  ],
  maxStalenessSeconds: 90
}

// Application EU
const euReadPref = {
  mode: "nearest",
  tags: [
    { region: "eu" },
    { region: "us" },
    { region: "apac" }
  ],
  maxStalenessSeconds: 90
}
```

### 2. Séparation des Charges de Travail

```javascript
// Configuration
{
  members: [
    // Nœuds OLTP (production)
    { _id: 0, host: "oltp-01:27017", tags: { workload: "oltp", tier: "primary" } },
    { _id: 1, host: "oltp-02:27017", tags: { workload: "oltp", tier: "primary" } },

    // Nœuds OLAP (analytics)
    { _id: 2, host: "olap-01:27017", tags: { workload: "olap", tier: "analytics" }, priority: 0 },
    { _id: 3, host: "olap-02:27017", tags: { workload: "olap", tier: "analytics" }, priority: 0 }
  ]
}

// Application OLTP (transactionnelle)
const oltp = db.orders.find(
  { customerId: id },
  {
    readPreference: {
      mode: "primaryPreferred",
      tags: [{ workload: "oltp" }]
    }
  }
)

// Application OLAP (analytics)
const analytics = db.orders.aggregate(
  [
    { $group: { _id: "$category", total: { $sum: "$amount" } } }
  ],
  {
    readPreference: {
      mode: "secondary",
      tags: [{ workload: "olap" }]
    },
    allowDiskUse: true
  }
)
```

### 3. Lecture Locale avec Écriture Globale

```javascript
// Stratégie pour application globale
class GlobalDataAccess {
  constructor(client, userRegion) {
    this.client = client
    this.db = client.db('myapp')
    this.userRegion = userRegion
  }

  // Lectures : Depuis région locale
  async read(collection, query) {
    return this.db.collection(collection).find(query).readPreference({
      mode: 'nearest',
      tags: [
        { region: this.userRegion },
        { region: 'us' },  // Fallback
        {}
      ]
    }).toArray()
  }

  // Écritures : Vers Primary avec majority
  async write(collection, doc) {
    return this.db.collection(collection).insertOne(doc, {
      writeConcern: { w: 'majority', wtimeout: 5000 }
    })
  }
}

// Utilisateur EU
const euUser = new GlobalDataAccess(client, 'eu')
const data = await euUser.read('products', { category: 'electronics' })
// Lira depuis membre EU (latence minimale)
```

### 4. Optimisation des Requêtes Lourdes

```javascript
// Système de reporting avec cache
class ReportingService {
  constructor(db) {
    this.db = db
    this.analyticsReadPref = {
      mode: 'secondary',
      tags: [
        { nodeType: 'high-memory', workload: 'analytics' }
      ],
      maxStalenessSeconds: 300  // 5 minutes acceptable
    }
  }

  async generateHeavyReport(startDate, endDate) {
    return this.db.collection('transactions').aggregate([
      {
        $match: {
          date: { $gte: startDate, $lte: endDate }
        }
      },
      {
        $group: {
          _id: {
            year: { $year: "$date" },
            month: { $month: "$date" },
            category: "$category"
          },
          revenue: { $sum: "$amount" },
          count: { $sum: 1 }
        }
      },
      {
        $sort: { "_id.year": 1, "_id.month": 1 }
      }
    ], {
      readPreference: this.analyticsReadPref,
      allowDiskUse: true,
      maxTimeMS: 60000  // Timeout 1 minute
    }).toArray()
  }
}
```

## Performance et Monitoring

### Mesurer la Latence par Membre

```javascript
function measureMemberLatencies() {
  const status = rs.status()
  const latencies = []

  status.members.forEach(member => {
    if (member.health === 1 && member.state !== 8) {  // Pas DOWN
      latencies.push({
        member: member.name,
        state: member.stateStr,
        pingMs: member.pingMs || 'N/A',
        lastHeartbeat: member.lastHeartbeat
      })
    }
  })

  latencies.sort((a, b) => (a.pingMs || 999999) - (b.pingMs || 999999))

  print("=== Member Latencies ===")
  latencies.forEach(m => {
    print(`${m.member} (${m.state}): ${m.pingMs} ms`)
  })

  return latencies
}

measureMemberLatencies()
```

### Surveiller la Distribution des Lectures

```javascript
// Monitoring de la distribution
function monitorReadDistribution() {
  const members = rs.status().members

  members.forEach(member => {
    // Se connecter à chaque membre
    const conn = new Mongo(member.name)
    const stats = conn.getDB('admin').serverStatus()

    print(`\n=== ${member.name} (${member.stateStr}) ===`)
    print(`Connections: ${stats.connections.current}`)
    print(`Operations:`)
    print(`  Queries: ${stats.opcounters.query}`)
    print(`  Inserts: ${stats.opcounters.insert}`)
    print(`  Updates: ${stats.opcounters.update}`)
    print(`Network:`)
    print(`  Bytes In: ${(stats.network.bytesIn / 1024 / 1024).toFixed(2)} MB`)
    print(`  Bytes Out: ${(stats.network.bytesOut / 1024 / 1024).toFixed(2)} MB`)
  })
}
```

### Impact du Read Preference sur les Métriques

```javascript
// Script de comparaison performance
async function compareReadPreferences() {
  const testQuery = { status: "active" }
  const iterations = 1000

  const modes = ['primary', 'primaryPreferred', 'secondary', 'secondaryPreferred', 'nearest']
  const results = {}

  for (const mode of modes) {
    const start = Date.now()

    for (let i = 0; i < iterations; i++) {
      await db.users.findOne(testQuery, { readPreference: mode })
    }

    const duration = Date.now() - start
    results[mode] = {
      totalMs: duration,
      avgMs: duration / iterations
    }
  }

  print("\n=== Read Preference Performance ===")
  Object.entries(results).forEach(([mode, stats]) => {
    print(`${mode}: ${stats.avgMs.toFixed(2)} ms avg (${stats.totalMs} ms total)`)
  })

  return results
}
```

## Bonnes Pratiques

### 1. Choix du Mode selon le Cas d'Usage

```javascript
// ✅ Recommandations

// Transactions financières, données critiques
{ readPreference: 'primary' }

// Tableau de bord admin (préfère fraîcheur, tolère failover)
{ readPreference: 'primaryPreferred' }

// Analytics, reporting (peut tolérer du lag)
{
  readPreference: 'secondary',
  tags: [{ workload: 'analytics' }]
}

// API publique (haute dispo, décharge Primary)
{
  readPreference: 'secondaryPreferred',
  maxStalenessSeconds: 90
}

// Application géo-distribuée (latence critique)
{
  readPreference: 'nearest',
  tags: [{ region: userRegion }]
}
```

### 2. Utilisation des Tags

```javascript
// ✅ Structure de tags cohérente
tags: {
  // Géographie
  region: "us-east-1",
  dc: "datacenter-1",
  zone: "zone-a",

  // Infrastructure
  nodeType: "ssd",
  tier: "premium",

  // Fonctionnel
  workload: "production",
  purpose: "oltp"
}

// ❌ Tags incohérents
tags: {
  location: "east",        // Un membre
  region: "us-west",       // Autre membre - nommage différent!
  datacenter: "dc1"        // Encore différent
}
```

### 3. Gestion du Lag

```javascript
// ✅ Définir maxStalenessSeconds approprié
{
  readPreference: 'secondaryPreferred',
  maxStalenessSeconds: 120  // Basé sur SLA métier
}

// ❌ Lag illimité pour données sensibles
{
  readPreference: 'secondary'
  // Pas de maxStalenessSeconds - risque de données très anciennes
}
```

### 4. Cohérence avec Sessions

```javascript
// ✅ Causal consistency pour read-your-writes
const session = client.startSession({ causalConsistency: true })

await db.orders.insertOne({ orderId: 123 }, { session })
const order = await db.orders.findOne(
  { orderId: 123 },
  { session, readPreference: 'secondaryPreferred' }
)
// Garantit de voir l'insertion

await session.endSession()

// ❌ Sans session - risque de ne pas voir l'écriture
await db.orders.insertOne({ orderId: 123 })
const order = await db.orders.findOne(
  { orderId: 123 },
  { readPreference: 'secondary' }
)
// Peut ne pas voir l'insertion si lag
```

### 5. Testing des Scénarios de Failover

```javascript
// Script de test de failover
async function testFailoverBehavior() {
  const modes = ['primary', 'primaryPreferred', 'secondary', 'secondaryPreferred']

  for (const mode of modes) {
    console.log(`\n=== Testing ${mode} ===`)

    try {
      // Simuler la lecture pendant failover
      const result = await db.users.findOne(
        {},
        { readPreference: mode }
      )

      console.log(`✓ Success with ${mode}`)
      console.log(`  Read from: ${result ? 'available member' : 'no data'}`)
    } catch (error) {
      console.log(`✗ Failed with ${mode}`)
      console.log(`  Error: ${error.message}`)
    }
  }
}

// Exécuter avant/après stepDown du Primary
```

## Pièges Courants et Solutions

### 1. Read Preference Ignoré en Transaction

```javascript
// ❌ Problème : readPreference ignoré
const session = client.startSession()
session.startTransaction()

const data = await db.collection.findOne(
  {},
  { session, readPreference: 'secondary' }  // IGNORÉ!
)
// Lira toujours depuis Primary

await session.commitTransaction()

// ✅ Solution : Accepter que transactions = Primary only
// Ou éviter les transactions si read preference requis
```

### 2. Lag Invisible sans maxStaleness

```javascript
// ❌ Problème : Secondary avec 5 minutes de lag
const data = await db.collection.find({}).readPreference('secondary').toArray()
// Peut retourner des données très anciennes

// ✅ Solution : Toujours définir maxStalenessSeconds
const data = await db.collection.find({}).readPreference({
  mode: 'secondary',
  maxStalenessSeconds: 120
}).toArray()
```

### 3. Tags Manquants

```javascript
// ❌ Problème : Tag spécifié n'existe pas
const data = await db.collection.find({}).readPreference({
  mode: 'secondary',
  tags: [{ nonExistentTag: 'value' }]
}).toArray()
// Erreur: No secondary with matching tags

// ✅ Solution : Toujours avoir un fallback
const data = await db.collection.find({}).readPreference({
  mode: 'secondary',
  tags: [
    { region: 'us-east' },  // Préféré
    { region: 'us-west' },  // Fallback 1
    {}                      // Fallback 2: n'importe quel secondary
  ]
}).toArray()
```

### 4. Conflit read/write dans Même Opération

```javascript
// ❌ Problème : findAndModify avec secondary
db.collection.findOneAndUpdate(
  { _id: id },
  { $set: { status: 'updated' } },
  { readPreference: 'secondary' }  // ERREUR!
)
// findAndModify DOIT être sur Primary (c'est une écriture)

// ✅ Solution : Utiliser primary ou ne pas spécifier
db.collection.findOneAndUpdate(
  { _id: id },
  { $set: { status: 'updated' } }
  // readPreference implicitement "primary"
)
```

## Conclusion

Le Read Preference est un outil puissant pour :

- ✅ **Optimiser les performances** en répartissant la charge de lecture
- ✅ **Réduire la latence** en lisant depuis des membres géographiquement proches
- ✅ **Augmenter la disponibilité** avec des stratégies de fallback
- ✅ **Isoler les workloads** (OLTP vs OLAP)
- ✅ **Supporter les architectures multi-régions**

**Points clés** :

1. **primary** : Cohérence forte, disponibilité réduite
2. **secondary** : Haute performance, eventual consistency
3. **nearest** : Latence minimale, géo-distribution
4. **Tags** : Ciblage précis des membres
5. **maxStalenessSeconds** : Contrôle de la fraîcheur des données
6. **Causal Consistency** : Read-your-writes avec sessions

Le choix du read preference doit être guidé par les besoins métier en termes de cohérence, latence, disponibilité et charge système. Une stratégie bien pensée peut significativement améliorer les performances tout en maintenant les garanties de cohérence requises.

⏭️ [Failover et haute disponibilité](/09-replication/09-failover-haute-disponibilite.md)
