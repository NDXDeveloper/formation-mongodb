🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.3 - Sharded Cluster avec Docker Compose

## Introduction

Configuration Docker Compose pour déployer un **Sharded Cluster MongoDB complet** avec plusieurs shards, config servers et mongos routers. Architecture de scalabilité horizontale pour grandes charges de données.

### 🎯 Cas d'usage

- Tests de scalabilité horizontale
- Prototypage d'architectures distribuées
- Formation sur le sharding MongoDB
- POC pour grandes volumétries
- Validation de shard key strategies
- Environnement de staging complexe

### ✅ Avantages

```markdown
✅ Scalabilité horizontale (ajout de shards)
✅ Distribution automatique des données
✅ Haute disponibilité (Replica Sets)
✅ Performance pour gros volumes
✅ Isolation des données (zone sharding)
✅ Architecture proche de la production
```

### ⚠️ Limitations

```markdown
⚠️ Très gourmand en ressources (8-12 GB RAM minimum)
⚠️ Complexité de gestion élevée
⚠️ Nombreux conteneurs (10+)
⚠️ Nécessite bonne compréhension du sharding
⚠️ Pas recommandé pour production (utiliser K8s)
⚠️ Latence réseau inter-conteneurs
```

---

## Architecture

### 🏗️ Architecture Complète

```
┌────────────────────────────────────────────────────────────┐
│                    Sharded Cluster                         │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Mongos (Query Routers)                  │  │
│  │  ┌─────────┐           ┌─────────┐                   │  │
│  │  │ mongos1 │           │ mongos2 │                   │  │
│  │  │ :27017  │           │ :27018  │                   │  │
│  │  └────┬────┘           └────┬────┘                   │  │
│  └───────┼─────────────────────┼────────────────────────┘  │
│          │                     │                           │
│          └──────────┬──────────┘                           │
│                     │                                      │
│  ┌──────────────────┴────────────────────────────────┐     │
│  │            Config Server (CSRS)                   │     │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │     │
│  │  │configsvr│  │configsvr│  │configsvr│            │     │
│  │  │   1     │  │   2     │  │   3     │            │     │
│  │  └─────────┘  └─────────┘  └─────────┘            │     │
│  └───────────────────────────────────────────────────┘     │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                     Shards                           │  │
│  │                                                      │  │
│  │  Shard 1 (RS)      Shard 2 (RS)      Shard 3 (RS)    │  │
│  │  ┌──────────┐      ┌──────────┐      ┌──────────┐    │  │
│  │  │ shard1-1 │      │ shard2-1 │      │ shard3-1 │    │  │
│  │  │ shard1-2 │      │ shard2-2 │      │ shard3-2 │    │  │
│  │  │ shard1-3 │      │ shard2-3 │      │ shard3-3 │    │  │
│  │  └──────────┘      └──────────┘      └──────────┘    │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**Composants** :
- **Mongos** (2) : Query routers, point d'entrée des clients
- **Config Servers** (3) : Métadonnées du cluster (CSRS)
- **Shards** (3) : Chaque shard est un Replica Set de 3 membres

**Total conteneurs** : 2 + 3 + 9 = **14 conteneurs**

---

## Configuration Minimale (2 Shards)

Pour réduire les ressources, démarrons avec une configuration plus légère :

### 📦 Architecture Simplifiée

```
- 2 Mongos
- 3 Config Servers (Replica Set)
- 2 Shards (chacun avec 2 membres)
= 9 conteneurs
```

### 🐳 docker-compose.yml (Version Simplifiée)

**Structure de projet** :
```
mongodb-sharded/
├── docker-compose.yml
├── .env
├── .gitignore
├── scripts/
│   ├── init-configsvr.sh
│   ├── init-shards.sh
│   ├── init-cluster.sh
│   └── setup-sharding.sh
└── data/
    ├── configsvr{1,2,3}/
    ├── shard1-{1,2}/
    └── shard2-{1,2}/
```

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  # ============================================
  # Config Servers (Replica Set)
  # ============================================
  configsvr1:
    image: mongo:7.0
    container_name: configsvr1
    hostname: configsvr1
    command: mongod --configsvr --replSet configReplSet --port 27017 --bind_ip_all
    ports:
      - "27119:27017"
    volumes:
      - configsvr1_data:/data/db
      - configsvr1_config:/data/configdb
    networks:
      - sharded_network

  configsvr2:
    image: mongo:7.0
    container_name: configsvr2
    hostname: configsvr2
    command: mongod --configsvr --replSet configReplSet --port 27017 --bind_ip_all
    ports:
      - "27120:27017"
    volumes:
      - configsvr2_data:/data/db
      - configsvr2_config:/data/configdb
    networks:
      - sharded_network

  configsvr3:
    image: mongo:7.0
    container_name: configsvr3
    hostname: configsvr3
    command: mongod --configsvr --replSet configReplSet --port 27017 --bind_ip_all
    ports:
      - "27121:27017"
    volumes:
      - configsvr3_data:/data/db
      - configsvr3_config:/data/configdb
    networks:
      - sharded_network

  # ============================================
  # Shard 1 (Replica Set)
  # ============================================
  shard1-1:
    image: mongo:7.0
    container_name: shard1-1
    hostname: shard1-1
    command: mongod --shardsvr --replSet shard1ReplSet --port 27017 --bind_ip_all
    ports:
      - "27018:27017"
    volumes:
      - shard1_1_data:/data/db
      - shard1_1_config:/data/configdb
    networks:
      - sharded_network

  shard1-2:
    image: mongo:7.0
    container_name: shard1-2
    hostname: shard1-2
    command: mongod --shardsvr --replSet shard1ReplSet --port 27017 --bind_ip_all
    ports:
      - "27019:27017"
    volumes:
      - shard1_2_data:/data/db
      - shard1_2_config:/data/configdb
    networks:
      - sharded_network

  # ============================================
  # Shard 2 (Replica Set)
  # ============================================
  shard2-1:
    image: mongo:7.0
    container_name: shard2-1
    hostname: shard2-1
    command: mongod --shardsvr --replSet shard2ReplSet --port 27017 --bind_ip_all
    ports:
      - "27020:27017"
    volumes:
      - shard2_1_data:/data/db
      - shard2_1_config:/data/configdb
    networks:
      - sharded_network

  shard2-2:
    image: mongo:7.0
    container_name: shard2-2
    hostname: shard2-2
    command: mongod --shardsvr --replSet shard2ReplSet --port 27017 --bind_ip_all
    ports:
      - "27021:27017"
    volumes:
      - shard2_2_data:/data/db
      - shard2_2_config:/data/configdb
    networks:
      - sharded_network

  # ============================================
  # Mongos (Query Routers)
  # ============================================
  mongos1:
    image: mongo:7.0
    container_name: mongos1
    hostname: mongos1
    command: mongos --configdb configReplSet/configsvr1:27017,configsvr2:27017,configsvr3:27017 --bind_ip_all --port 27017
    ports:
      - "27017:27017"
    depends_on:
      - configsvr1
      - configsvr2
      - configsvr3
    networks:
      - sharded_network

  mongos2:
    image: mongo:7.0
    container_name: mongos2
    hostname: mongos2
    command: mongos --configdb configReplSet/configsvr1:27017,configsvr2:27017,configsvr3:27017 --bind_ip_all --port 27017
    ports:
      - "27117:27017"
    depends_on:
      - configsvr1
      - configsvr2
      - configsvr3
    networks:
      - sharded_network

volumes:
  configsvr1_data:
  configsvr1_config:
  configsvr2_data:
  configsvr2_config:
  configsvr3_data:
  configsvr3_config:
  shard1_1_data:
  shard1_1_config:
  shard1_2_data:
  shard1_2_config:
  shard2_1_data:
  shard2_1_config:
  shard2_2_data:
  shard2_2_config:

networks:
  sharded_network:
    driver: bridge
```

---

### 🚀 Scripts d'Initialisation

**scripts/init-configsvr.sh** :
```bash
#!/bin/bash
# Initialize Config Server Replica Set

echo "Initializing Config Server Replica Set..."
sleep 10

docker exec -it configsvr1 mongosh --port 27017 --eval "
rs.initiate({
  _id: 'configReplSet',
  configsvr: true,
  members: [
    { _id: 0, host: 'configsvr1:27017' },
    { _id: 1, host: 'configsvr2:27017' },
    { _id: 2, host: 'configsvr3:27017' }
  ]
})
"

echo "Waiting for Config Server Replica Set to stabilize..."
sleep 15

docker exec -it configsvr1 mongosh --port 27017 --eval "rs.status()"
```

**scripts/init-shards.sh** :
```bash
#!/bin/bash
# Initialize Shard Replica Sets

echo "Initializing Shard 1 Replica Set..."
docker exec -it shard1-1 mongosh --port 27017 --eval "
rs.initiate({
  _id: 'shard1ReplSet',
  members: [
    { _id: 0, host: 'shard1-1:27017' },
    { _id: 1, host: 'shard1-2:27017' }
  ]
})
"

sleep 10

echo "Initializing Shard 2 Replica Set..."
docker exec -it shard2-1 mongosh --port 27017 --eval "
rs.initiate({
  _id: 'shard2ReplSet',
  members: [
    { _id: 0, host: 'shard2-1:27017' },
    { _id: 1, host: 'shard2-2:27017' }
  ]
})
"

sleep 10

echo "Checking shard status..."
docker exec -it shard1-1 mongosh --port 27017 --eval "rs.status()"
docker exec -it shard2-1 mongosh --port 27017 --eval "rs.status()"
```

**scripts/init-cluster.sh** :
```bash
#!/bin/bash
# Add Shards to Cluster

echo "Adding Shards to Cluster..."
sleep 5

docker exec -it mongos1 mongosh --port 27017 --eval "
sh.addShard('shard1ReplSet/shard1-1:27017,shard1-2:27017')
"

sleep 5

docker exec -it mongos1 mongosh --port 27017 --eval "
sh.addShard('shard2ReplSet/shard2-1:27017,shard2-2:27017')
"

sleep 5

echo "Cluster status:"
docker exec -it mongos1 mongosh --port 27017 --eval "sh.status()"
```

**scripts/setup-sharding.sh** :
```bash
#!/bin/bash
# Enable sharding on database and collection

DB_NAME=${1:-testdb}
COLLECTION=${2:-users}
SHARD_KEY=${3:-userId}

echo "Enabling sharding on database: $DB_NAME"
docker exec -it mongos1 mongosh --port 27017 --eval "
sh.enableSharding('$DB_NAME')
"

echo "Creating index on shard key: $SHARD_KEY"
docker exec -it mongos1 mongosh --port 27017 --eval "
use $DB_NAME
db.$COLLECTION.createIndex({ $SHARD_KEY: 1 })
"

echo "Sharding collection: $DB_NAME.$COLLECTION on key: $SHARD_KEY"
docker exec -it mongos1 mongosh --port 27017 --eval "
sh.shardCollection('$DB_NAME.$COLLECTION', { $SHARD_KEY: 1 })
"

echo "Shard distribution:"
docker exec -it mongos1 mongosh --port 27017 --eval "
use $DB_NAME
db.$COLLECTION.getShardDistribution()
"
```

Rendre les scripts exécutables :
```bash
chmod +x scripts/*.sh
```

---

### 📋 Déploiement Complet

```bash
# 1. Créer le projet
mkdir mongodb-sharded && cd mongodb-sharded
mkdir -p scripts

# 2. Créer les fichiers
# (copier docker-compose.yml et les scripts)

# 3. Démarrer tous les conteneurs
docker-compose up -d

# 4. Attendre que tous démarrent
docker-compose ps
sleep 30

# 5. Initialiser Config Servers
./scripts/init-configsvr.sh

# 6. Initialiser les Shards
./scripts/init-shards.sh

# 7. Configurer le Cluster
./scripts/init-cluster.sh

# 8. Vérifier le statut
docker exec -it mongos1 mongosh --eval "sh.status()"

# 9. Activer le sharding sur une collection
./scripts/setup-sharding.sh testdb users userId

# 10. Se connecter via mongos
mongosh mongodb://localhost:27017
```

---

## Configuration Complète (3 Shards)

### 🐳 docker-compose-full.yml

Pour un environnement plus réaliste avec 3 shards de 3 membres chacun :

```yaml
version: '3.8'

services:
  # ============================================
  # Config Servers (3 membres)
  # ============================================
  configsvr1:
    image: mongo:7.0
    container_name: configsvr1
    hostname: configsvr1
    command: mongod --configsvr --replSet configReplSet --port 27017 --bind_ip_all
    volumes:
      - configsvr1:/data/db
    networks:
      - sharded_network

  configsvr2:
    image: mongo:7.0
    container_name: configsvr2
    hostname: configsvr2
    command: mongod --configsvr --replSet configReplSet --port 27017 --bind_ip_all
    volumes:
      - configsvr2:/data/db
    networks:
      - sharded_network

  configsvr3:
    image: mongo:7.0
    container_name: configsvr3
    hostname: configsvr3
    command: mongod --configsvr --replSet configReplSet --port 27017 --bind_ip_all
    volumes:
      - configsvr3:/data/db
    networks:
      - sharded_network

  # ============================================
  # Shard 1 (3 membres)
  # ============================================
  shard1-1:
    image: mongo:7.0
    container_name: shard1-1
    hostname: shard1-1
    command: mongod --shardsvr --replSet shard1 --port 27017 --bind_ip_all
    volumes:
      - shard1-1:/data/db
    networks:
      - sharded_network

  shard1-2:
    image: mongo:7.0
    container_name: shard1-2
    hostname: shard1-2
    command: mongod --shardsvr --replSet shard1 --port 27017 --bind_ip_all
    volumes:
      - shard1-2:/data/db
    networks:
      - sharded_network

  shard1-3:
    image: mongo:7.0
    container_name: shard1-3
    hostname: shard1-3
    command: mongod --shardsvr --replSet shard1 --port 27017 --bind_ip_all
    volumes:
      - shard1-3:/data/db
    networks:
      - sharded_network

  # ============================================
  # Shard 2 (3 membres)
  # ============================================
  shard2-1:
    image: mongo:7.0
    container_name: shard2-1
    hostname: shard2-1
    command: mongod --shardsvr --replSet shard2 --port 27017 --bind_ip_all
    volumes:
      - shard2-1:/data/db
    networks:
      - sharded_network

  shard2-2:
    image: mongo:7.0
    container_name: shard2-2
    hostname: shard2-2
    command: mongod --shardsvr --replSet shard2 --port 27017 --bind_ip_all
    volumes:
      - shard2-2:/data/db
    networks:
      - sharded_network

  shard2-3:
    image: mongo:7.0
    container_name: shard2-3
    hostname: shard2-3
    command: mongod --shardsvr --replSet shard2 --port 27017 --bind_ip_all
    volumes:
      - shard2-3:/data/db
    networks:
      - sharded_network

  # ============================================
  # Shard 3 (3 membres)
  # ============================================
  shard3-1:
    image: mongo:7.0
    container_name: shard3-1
    hostname: shard3-1
    command: mongod --shardsvr --replSet shard3 --port 27017 --bind_ip_all
    volumes:
      - shard3-1:/data/db
    networks:
      - sharded_network

  shard3-2:
    image: mongo:7.0
    container_name: shard3-2
    hostname: shard3-2
    command: mongod --shardsvr --replSet shard3 --port 27017 --bind_ip_all
    volumes:
      - shard3-2:/data/db
    networks:
      - sharded_network

  shard3-3:
    image: mongo:7.0
    container_name: shard3-3
    hostname: shard3-3
    command: mongod --shardsvr --replSet shard3 --port 27017 --bind_ip_all
    volumes:
      - shard3-3:/data/db
    networks:
      - sharded_network

  # ============================================
  # Mongos (2 routers)
  # ============================================
  mongos1:
    image: mongo:7.0
    container_name: mongos1
    hostname: mongos1
    command: mongos --configdb configReplSet/configsvr1:27017,configsvr2:27017,configsvr3:27017 --bind_ip_all
    ports:
      - "27017:27017"
    depends_on:
      - configsvr1
      - configsvr2
      - configsvr3
    networks:
      - sharded_network

  mongos2:
    image: mongo:7.0
    container_name: mongos2
    hostname: mongos2
    command: mongos --configdb configReplSet/configsvr1:27017,configsvr2:27017,configsvr3:27017 --bind_ip_all
    ports:
      - "27018:27017"
    depends_on:
      - configsvr1
      - configsvr2
      - configsvr3
    networks:
      - sharded_network

volumes:
  configsvr1:
  configsvr2:
  configsvr3:
  shard1-1:
  shard1-2:
  shard1-3:
  shard2-1:
  shard2-2:
  shard2-3:
  shard3-1:
  shard3-2:
  shard3-3:

networks:
  sharded_network:
    driver: bridge
```

---

## Gestion du Sharded Cluster

### 🔍 Commandes de Vérification

```javascript
// Se connecter via mongos
mongosh mongodb://localhost:27017

// Status général du cluster
sh.status()

// Liste des shards
db.adminCommand({ listShards: 1 })

// Config servers
sh.status().configsvr

// Databases shardées
use config
db.databases.find()

// Collections shardées
db.collections.find()

// Distribution des chunks
db.chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])
```

---

### ⚙️ Configuration du Sharding

**Activer le sharding sur une database** :
```javascript
sh.enableSharding("mydb")
```

**Sharder une collection** :
```javascript
// Avec hashed shard key
sh.shardCollection("mydb.users", { userId: "hashed" })

// Avec range shard key
sh.shardCollection("mydb.orders", { customerId: 1, orderDate: 1 })

// Avec zone sharding
sh.shardCollection("mydb.analytics", { region: 1, timestamp: 1 })
```

**Vérifier la distribution** :
```javascript
use mydb
db.users.getShardDistribution()
```

---

### 📊 Gestion des Chunks

**Voir les chunks** :
```javascript
use config
db.chunks.find({ ns: "mydb.users" }).pretty()

// Statistiques par shard
db.chunks.aggregate([
  { $match: { ns: "mydb.users" } },
  { $group: { _id: "$shard", chunks: { $sum: 1 } } }
])
```

**Split manuel** :
```javascript
sh.splitAt("mydb.users", { userId: 5000 })
sh.splitFind("mydb.users", { userId: 2500 })
```

**Déplacer un chunk** :
```javascript
sh.moveChunk("mydb.users", { userId: 1000 }, "shard2ReplSet")
```

**Balancer** :
```javascript
// Status du balancer
sh.getBalancerState()
sh.isBalancerRunning()

// Arrêter le balancer
sh.stopBalancer()

// Démarrer le balancer
sh.startBalancer()

// Définir fenêtre de balancing
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "23:00", stop: "06:00" } } },
  { upsert: true }
)
```

---

### 🎯 Zone Sharding

Associer des plages de données à des shards spécifiques :

```javascript
// Créer des zones
sh.addShardToZone("shard1ReplSet", "EU")
sh.addShardToZone("shard2ReplSet", "US")
sh.addShardToZone("shard3ReplSet", "ASIA")

// Définir les ranges
sh.updateZoneKeyRange(
  "mydb.users",
  { region: "EU", userId: MinKey },
  { region: "EU", userId: MaxKey },
  "EU"
)

sh.updateZoneKeyRange(
  "mydb.users",
  { region: "US", userId: MinKey },
  { region: "US", userId: MaxKey },
  "US"
)

sh.updateZoneKeyRange(
  "mydb.users",
  { region: "ASIA", userId: MinKey },
  { region: "ASIA", userId: MaxKey },
  "ASIA"
)

// Vérifier les zones
sh.status()
```

---

## Tests et Validation

### 🧪 Test de Distribution

**Script de test** :
```javascript
// Insérer des données de test
use testdb

for (let i = 0; i < 10000; i++) {
  db.users.insertOne({
    userId: i,
    name: "User " + i,
    email: "user" + i + "@example.com",
    region: ["EU", "US", "ASIA"][i % 3],
    createdAt: new Date()
  });
}

// Vérifier la distribution
db.users.getShardDistribution()

// Résultat attendu :
// Shard shard1ReplSet at shard1-1:27017,shard1-2:27017
//   data : ~3.3KB docs : 3333 chunks : 2
// Shard shard2ReplSet at shard2-1:27017,shard2-2:27017
//   data : ~3.3KB docs : 3334 chunks : 2
```

---

### 🔍 Test de Queries

```javascript
// Query routée vers un shard spécifique
db.users.find({ userId: 1000 }).explain()
// Vérifier dans "winningPlan" → "shards" → 1 seul shard

// Query scatter-gather (tous les shards)
db.users.find({ name: /User/ }).explain()
// "shards" devrait montrer tous les shards

// Agrégation
db.users.aggregate([
  { $group: { _id: "$region", count: { $sum: 1 } } }
]).explain()
```

---

### 🚀 Test de Performance

**Script de benchmark** :
```bash
#!/bin/bash
# benchmark.sh

echo "=== Sharded Cluster Benchmark ==="

# Insertions
echo "Testing insertions..."
time docker exec mongos1 mongosh --quiet --eval "
  use testdb
  for (let i = 0; i < 10000; i++) {
    db.benchmark.insertOne({ value: i, data: 'x'.repeat(100) });
  }
  print('Inserted 10000 documents');
"

# Queries
echo "Testing queries..."
time docker exec mongos1 mongosh --quiet --eval "
  use testdb
  for (let i = 0; i < 1000; i++) {
    db.benchmark.findOne({ value: Math.floor(Math.random() * 10000) });
  }
  print('Executed 1000 queries');
"

# Distribution
echo "Checking distribution..."
docker exec mongos1 mongosh --quiet --eval "
  use testdb
  db.benchmark.getShardDistribution()
"
```

---

## Connexion depuis Applications

### 🔗 Connection Strings

**Format pour Sharded Cluster** :
```bash
# Via mongos uniquement
mongodb://mongos1:27017,mongos2:27017/

# Avec options
mongodb://mongos1:27017,mongos2:27017/?readPreference=nearest&retryWrites=true&w=majority

# ⚠️ Ne JAMAIS se connecter directement aux shards ou config servers
# Toujours passer par mongos
```

---

### 💻 Code Applicatif

**Node.js** :
```javascript
const { MongoClient } = require('mongodb');

const uri = 'mongodb://mongos1:27017,mongos2:27017/';

const client = new MongoClient(uri, {
  useNewUrlParser: true,
  useUnifiedTopology: true,

  // Options importantes pour sharded cluster
  readPreference: 'nearest',
  w: 'majority',
  retryWrites: true,

  maxPoolSize: 100
});

async function main() {
  try {
    await client.connect();

    const db = client.db('mydb');

    // Insert (routé automatiquement par mongos)
    await db.collection('users').insertOne({
      userId: 12345,
      name: 'John Doe'
    });

    // Query ciblée (1 seul shard)
    const user = await db.collection('users').findOne({ userId: 12345 });

    // Agrégation (potentiellement tous les shards)
    const stats = await db.collection('users').aggregate([
      { $group: { _id: "$region", count: { $sum: 1 } } }
    ]).toArray();

    console.log(stats);
  } finally {
    await client.close();
  }
}

main();
```

---

### 🐳 Application Dockerisée

```yaml
version: '3.8'

services:
  # ... (config servers, shards, mongos)

  app:
    build: ./app
    environment:
      MONGODB_URI: mongodb://mongos1:27017,mongos2:27017/mydb
    depends_on:
      - mongos1
      - mongos2
    networks:
      - sharded_network
```

---

## Monitoring et Maintenance

### 📊 Monitoring

**Script de monitoring** :
```bash
#!/bin/bash
# monitor-cluster.sh

echo "========================================="
echo "Sharded Cluster Monitoring"
echo "========================================="

# Cluster status
echo -e "\n### Cluster Status ###"
docker exec mongos1 mongosh --quiet --eval "
  var status = sh.status();
  printjson({
    shards: db.getSiblingDB('config').shards.count(),
    databases: db.getSiblingDB('config').databases.count(),
    collections: db.getSiblingDB('config').collections.count()
  });
"

# Shards health
echo -e "\n### Shards Health ###"
docker exec mongos1 mongosh --quiet --eval "
  db.adminCommand({ listShards: 1 }).shards.forEach(shard => {
    print(shard._id + ': ' + shard.host + ' - ' + shard.state);
  });
"

# Balancer status
echo -e "\n### Balancer ###"
docker exec mongos1 mongosh --quiet --eval "
  print('Running: ' + sh.isBalancerRunning());
  print('State: ' + sh.getBalancerState());
"

# Chunks distribution
echo -e "\n### Chunks Distribution ###"
docker exec mongos1 mongosh --quiet --eval "
  use config
  db.chunks.aggregate([
    { \$group: { _id: '\$shard', chunks: { \$sum: 1 } } }
  ]).forEach(printjson);
"

# Connections
echo -e "\n### Connections ###"
for shard in shard1-1 shard2-1; do
  echo "$shard:"
  docker exec $shard mongosh --quiet --eval "
    db.serverStatus().connections
  "
done
```

---

### 🔧 Maintenance

**Ajouter un shard** :
```bash
# 1. Démarrer de nouveaux membres dans docker-compose.yml
# shard4-1, shard4-2, shard4-3

# 2. Redémarrer
docker-compose up -d

# 3. Initialiser le nouveau shard
docker exec shard4-1 mongosh --eval "
rs.initiate({
  _id: 'shard4',
  members: [
    { _id: 0, host: 'shard4-1:27017' },
    { _id: 1, host: 'shard4-2:27017' },
    { _id: 2, host: 'shard4-3:27017' }
  ]
})
"

# 4. Ajouter au cluster
docker exec mongos1 mongosh --eval "
sh.addShard('shard4/shard4-1:27017,shard4-2:27017,shard4-3:27017')
"

# 5. Le balancer redistribue automatiquement
```

**Retirer un shard** :
```bash
# 1. Drainer le shard
docker exec mongos1 mongosh --eval "
db.adminCommand({ removeShard: 'shard3ReplSet' })
"

# 2. Vérifier le progrès
docker exec mongos1 mongosh --eval "
db.adminCommand({ removeShard: 'shard3ReplSet' })
"

# 3. Quand state: 'completed', arrêter les conteneurs
docker stop shard3-1 shard3-2 shard3-3
```

---

## Backup et Restauration

### 💾 Backup du Cluster

**Stratégie recommandée** :
```bash
# 1. Arrêter le balancer
docker exec mongos1 mongosh --eval "sh.stopBalancer()"

# 2. Attendre que les migrations se terminent
docker exec mongos1 mongosh --eval "
  while (sh.isBalancerRunning()) {
    print('Waiting for balancer to stop...');
    sleep(1000);
  }
"

# 3. Backup des config servers
docker exec configsvr1 mongodump \
  --port 27017 \
  --out /backup/config-$(date +%Y%m%d)

# 4. Backup de chaque shard
docker exec shard1-1 mongodump \
  --port 27017 \
  --out /backup/shard1-$(date +%Y%m%d)

docker exec shard2-1 mongodump \
  --port 27017 \
  --out /backup/shard2-$(date +%Y%m%d)

# 5. Redémarrer le balancer
docker exec mongos1 mongosh --eval "sh.startBalancer()"
```

---

## Dépannage

### 🔧 Problèmes Courants

#### Cluster ne s'initialise pas

```bash
# Vérifier tous les conteneurs
docker-compose ps

# Vérifier la résolution DNS
docker exec mongos1 ping configsvr1
docker exec mongos1 ping shard1-1

# Réinitialiser si nécessaire
docker-compose down -v
docker-compose up -d
# Réexécuter les scripts d'init
```

#### Shards déséquilibrés

```bash
# Vérifier la distribution
docker exec mongos1 mongosh --eval "
  use config
  db.chunks.aggregate([
    { \$group: { _id: '\$shard', count: { \$sum: 1 } } }
  ])
"

# Forcer le balancer
docker exec mongos1 mongosh --eval "
  sh.startBalancer()
  sh.setBalancerState(true)
"

# Vérifier jumbo chunks
docker exec mongos1 mongosh --eval "
  use config
  db.chunks.find({ jumbo: true }).count()
"
```

#### Performances dégradées

```bash
# Vérifier les mongos
docker stats mongos1 mongos2

# Vérifier les slow queries
docker exec mongos1 mongosh --eval "
  db.setProfilingLevel(1, { slowms: 100 })
"

# Analyser les queries scatter-gather
docker exec mongos1 mongosh --eval "
  use testdb
  db.collection.find().explain('executionStats')
"
# Vérifier le nombre de shards contactés
```

---

## Checklist de Déploiement

### ✅ Avant le Démarrage

```markdown
□ Docker 20.10+ et Docker Compose 2.0+ installés
□ Ressources suffisantes (8-12 GB RAM, 8 cores minimum)
□ Ports disponibles (27017-27121)
□ Scripts d'initialisation préparés
□ Architecture validée (2 ou 3 shards)
□ Shard key strategy définie
□ Backup strategy planifiée
```

### ✅ Après le Démarrage

```markdown
□ Tous les conteneurs UP : docker-compose ps
□ Config Servers initialisés
□ Tous les Shards initialisés
□ Shards ajoutés au cluster : sh.status()
□ Balancer actif
□ Database shardée : sh.enableSharding()
□ Collection shardée avec shard key
□ Distribution vérifiée : getShardDistribution()
□ Connexion mongos fonctionne
□ Tests de distribution réussis
□ Monitoring en place
```

---

## Optimisations

### ⚡ Performance

**Ressources par conteneur** :
```yaml
services:
  shard1-1:
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M
```

**Cache WiredTiger** :
```yaml
command: |
  mongod --shardsvr --replSet shard1
  --wiredTigerCacheSizeGB 0.5
  --bind_ip_all
```

---

## Ressources

### 📚 Documentation
- [Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [Deploy Sharded Cluster](https://www.mongodb.com/docs/manual/tutorial/deploy-shard-cluster/)
- [Shard Keys](https://www.mongodb.com/docs/manual/core/sharding-shard-key/)
- [Zone Sharding](https://www.mongodb.com/docs/manual/core/zone-sharding/)

### 🔗 Liens Utiles
- [Balancer](https://www.mongodb.com/docs/manual/core/sharding-balancer-administration/)
- [Chunk Migrations](https://www.mongodb.com/docs/manual/core/sharding-data-partitioning/)

---

## Conclusion

Le Sharded Cluster avec Docker Compose est **idéal pour** :
- ✅ Tests de scalabilité
- ✅ Validation de shard key strategy
- ✅ Formation sur le sharding
- ✅ Prototypage d'architectures distribuées
- ✅ Environnements de staging complexes

**Ne pas utiliser pour** :
- ❌ Production réelle (utiliser K8s ou Atlas)
- ❌ Ressources limitées (< 8 GB RAM)
- ❌ Applications simples (overhead inutile)

**Pour la production** :
- Utiliser Kubernetes avec MongoDB Operator
- MongoDB Atlas (solution managée)
- Infrastructure dédiée avec monitoring avancé

**Prochaine étape** : F.4 - Stack Complète

---

**Version** : 1.0
**Testé avec** : MongoDB 7.0, Docker 24.0, Docker Compose 2.23

⏭️ [Stack complète (MongoDB + Mongo Express + Application)](/annexes/docker-compose/04-stack-complete.md)
