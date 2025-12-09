🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.2 - Configuration Sharded Cluster

## Présentation

### Objectif
Architecture distribuée MongoDB pour la scalabilité horizontale massive, permettant de distribuer les données sur plusieurs serveurs (shards) tout en maintenant la haute disponibilité.

### Topologie

```
┌────────────────────────────────────────────────────────────────────┐
│                        Sharded Cluster: sh-prod                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Application Layer                         │  │
│  │              ┌───────────┬───────────┬───────────┐           │  │
│  │              │  App 1    │   App 2   │   App 3   │           │  │
│  └──────────────┴─────┬─────┴─────┬─────┴─────┬─────┴───────────┘  │
│                       │           │           │                    │
│  ┌────────────────────┴───────────┴───────────┴─────────────────┐  │
│  │                    Query Routers (mongos)                    │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
│  │  │  mongos-01   │  │  mongos-02   │  │  mongos-03   │        │  │
│  │  │   27017      │  │   27017      │  │   27017      │        │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘        │  │
│  └─────────┼─────────────────┼─────────────────┼────────────────┘  │
│            │                 │                 │                   │
│  ┌─────────┴─────────────────┴─────────────────┴────────────────┐  │
│  │              Config Server Replica Set (CSRS)                │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │  │
│  │  │  config-01   │  │  config-02   │  │  config-03   │        │  │
│  │  │  PRIMARY     │  │  SECONDARY   │  │  SECONDARY   │        │  │
│  │  │   27019      │  │   27019      │  │   27019      │        │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘        │  │
│  └────────────────────────────────────────────────────────────┬─┘  │
│                                                               │    │
│  ┌────────────────────────────────────────────────────────────┘    │
│  │                                                                 │
│  │  ┌───────────────────────────────────────────────────────────┐  │
│  │  │              Shard 1 - Replica Set (shard1-rs)            │  │
│  │  │  ┌──────────┐    ┌──────────┐    ┌──────────┐             │  │
│  │  │  │ shard1-1 │───▶│ shard1-2 │───▶│ shard1-3 │             │  │
│  │  │  │ PRIMARY  │◀───│ SECONDARY│◀───│ SECONDARY│             │  │
│  │  │  │  27018   │    │  27018   │    │  27018   │             │  │
│  │  │  └──────────┘    └──────────┘    └──────────┘             │  │
│  │  └───────────────────────────────────────────────────────────┘  │
│  │                                                                 │
│  │  ┌───────────────────────────────────────────────────────────┐  │
│  │  │              Shard 2 - Replica Set (shard2-rs)            │  │
│  │  │  ┌──────────┐    ┌──────────┐    ┌──────────┐             │  │
│  │  │  │ shard2-1 │───▶│ shard2-2 │───▶│ shard2-3 │             │  │
│  │  │  │ PRIMARY  │◀───│ SECONDARY│◀───│ SECONDARY│             │  │
│  │  │  │  27018   │    │  27018   │    │  27018   │             │  │
│  │  │  └──────────┘    └──────────┘    └──────────┘             │  │
│  │  └───────────────────────────────────────────────────────────┘  │
│  │                                                                 │
│  │  ┌───────────────────────────────────────────────────────────┐  │
│  │  │              Shard 3 - Replica Set (shard3-rs)            │  │
│  │  │  ┌──────────┐    ┌──────────┐    ┌──────────┐             │  │
│  │  │  │ shard3-1 │───▶│ shard3-2 │───▶│ shard3-3 │             │  │
│  │  │  │ PRIMARY  │◀───│ SECONDARY│◀───│ SECONDARY│             │  │
│  │  │  │  27018   │    │  27018   │    │  27018   │             │  │
│  │  │  └──────────┘    └──────────┘    └──────────┘             │  │
│  │  └───────────────────────────────────────────────────────────┘  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

### Composants

| Composant | Rôle | Nombre | Port |
|-----------|------|--------|------|
| **mongos** | Query Router | 3 | 27017 |
| **Config Servers** | Métadonnées | 3 (CSRS) | 27019 |
| **Shard 1** | Stockage données | 3 (Replica Set) | 27018 |
| **Shard 2** | Stockage données | 3 (Replica Set) | 27018 |
| **Shard 3** | Stockage données | 3 (Replica Set) | 27018 |

**Total : 15 serveurs**

### Caractéristiques

- **Scalabilité horizontale** : Ajout de shards sans interruption
- **Distribution des données** : Répartition automatique via shard key
- **Haute disponibilité** : Chaque shard est un Replica Set
- **Performance** : Parallélisation des requêtes sur les shards
- **Gestion des métadonnées** : Config Servers en Replica Set

---

## Prérequis matériels

### Configuration par type de serveur

#### mongos (Query Routers)

| Ressource | Minimum | Recommandé |
|-----------|---------|------------|
| **CPU** | 4 cores | 8 cores |
| **RAM** | 8 GB | 16 GB |
| **Stockage** | 50 GB | 100 GB SSD |
| **Réseau** | 1 Gbps | 10 Gbps |

#### Config Servers (par serveur)

| Ressource | Minimum | Recommandé |
|-----------|---------|------------|
| **CPU** | 2 cores | 4 cores |
| **RAM** | 4 GB | 8 GB |
| **Stockage** | 50 GB SSD | 100 GB SSD |
| **Réseau** | 1 Gbps | 10 Gbps |

#### Shard Servers (par serveur)

| Ressource | Minimum | Recommandé |
|-----------|---------|------------|
| **CPU** | 8 cores | 16 cores |
| **RAM** | 32 GB | 64 GB |
| **Stockage** | 1 TB SSD | 2 TB NVMe (RAID 10) |
| **Réseau** | 10 Gbps | 25 Gbps |

### Calcul du stockage total

Pour **3 shards** avec **replication factor 3** :

```
Données brutes : 1 TB
Avec réplication : 1 TB × 3 (replicas) × 3 (shards) = 9 TB
Avec overhead (20%) : 9 TB × 1.2 = ~11 TB minimum
```

---

## Architecture réseau

### Plan d'adressage IP

#### Query Routers (mongos)

| Serveur | Hostname | IP Privée | Rôle | Port |
|---------|----------|-----------|------|------|
| Router 1 | mongos-01 | 10.0.1.10 | mongos | 27017 |
| Router 2 | mongos-02 | 10.0.1.11 | mongos | 27017 |
| Router 3 | mongos-03 | 10.0.1.12 | mongos | 27017 |

#### Config Servers (CSRS)

| Serveur | Hostname | IP Privée | Rôle | Port |
|---------|----------|-----------|------|------|
| Config 1 | config-01 | 10.0.2.10 | Config Server | 27019 |
| Config 2 | config-02 | 10.0.2.11 | Config Server | 27019 |
| Config 3 | config-03 | 10.0.2.12 | Config Server | 27019 |

#### Shard 1 (shard1-rs)

| Serveur | Hostname | IP Privée | Rôle | Port |
|---------|----------|-----------|------|------|
| Shard 1.1 | shard1-01 | 10.0.3.10 | mongod | 27018 |
| Shard 1.2 | shard1-02 | 10.0.3.11 | mongod | 27018 |
| Shard 1.3 | shard1-03 | 10.0.3.12 | mongod | 27018 |

#### Shard 2 (shard2-rs)

| Serveur | Hostname | IP Privée | Rôle | Port |
|---------|----------|-----------|------|------|
| Shard 2.1 | shard2-01 | 10.0.4.10 | mongod | 27018 |
| Shard 2.2 | shard2-02 | 10.0.4.11 | mongod | 27018 |
| Shard 2.3 | shard2-03 | 10.0.4.12 | mongod | 27018 |

#### Shard 3 (shard3-rs)

| Serveur | Hostname | IP Privée | Rôle | Port |
|---------|----------|-----------|------|------|
| Shard 3.1 | shard3-01 | 10.0.5.10 | mongod | 27018 |
| Shard 3.2 | shard3-02 | 10.0.5.11 | mongod | 27018 |
| Shard 3.3 | shard3-03 | 10.0.5.12 | mongod | 27018 |

### Configuration DNS

```bash
# /etc/hosts sur TOUS les serveurs

# Query Routers
10.0.1.10   mongos-01
10.0.1.11   mongos-02
10.0.1.12   mongos-03

# Config Servers
10.0.2.10   config-01
10.0.2.11   config-02
10.0.2.12   config-03

# Shard 1
10.0.3.10   shard1-01
10.0.3.11   shard1-02
10.0.3.12   shard1-03

# Shard 2
10.0.4.10   shard2-01
10.0.4.11   shard2-02
10.0.4.12   shard2-03

# Shard 3
10.0.5.10   shard3-01
10.0.5.11   shard3-02
10.0.5.12   shard3-03
```

### Règles firewall

```bash
# Script firewall à appliquer sur tous les serveurs
#!/bin/bash

# Config Servers - Port 27019
iptables -A INPUT -p tcp --dport 27019 -s 10.0.2.0/24 -j ACCEPT  # Entre config servers
iptables -A INPUT -p tcp --dport 27019 -s 10.0.1.0/24 -j ACCEPT  # Depuis mongos

# Shards - Port 27018
iptables -A INPUT -p tcp --dport 27018 -s 10.0.3.0/24 -j ACCEPT  # Shard1 interne
iptables -A INPUT -p tcp --dport 27018 -s 10.0.4.0/24 -j ACCEPT  # Shard2 interne
iptables -A INPUT -p tcp --dport 27018 -s 10.0.5.0/24 -j ACCEPT  # Shard3 interne
iptables -A INPUT -p tcp --dport 27018 -s 10.0.1.0/24 -j ACCEPT  # Depuis mongos

# mongos - Port 27017
iptables -A INPUT -p tcp --dport 27017 -s 10.0.10.0/24 -j ACCEPT # Applications

# SSH (administration)
iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/16 -j ACCEPT

# Bloquer le reste
iptables -A INPUT -p tcp --dport 27017 -j DROP
iptables -A INPUT -p tcp --dport 27018 -j DROP
iptables -A INPUT -p tcp --dport 27019 -j DROP
```

---

## Préparation système

### Paramètres kernel (tous les serveurs)

```bash
# /etc/sysctl.conf
vm.swappiness = 1
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
net.core.somaxconn = 4096
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_max_syn_backlog = 4096
fs.file-max = 98000

# Appliquer
sudo sysctl -p
```

### Limites système

```bash
# /etc/security/limits.conf
mongod soft nofile 64000
mongod hard nofile 64000
mongod soft nproc 32000
mongod hard nproc 32000
```

### Désactivation THP

```bash
# /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages (THP)
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=mongod.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target

# Activer
sudo systemctl enable disable-thp.service
sudo systemctl start disable-thp.service
```

### Génération du keyfile (sécurité)

```bash
# Générer une seule fois
openssl rand -base64 756 > /etc/mongodb-keyfile

# Copier sur TOUS les 15 serveurs avec le même contenu
chmod 400 /etc/mongodb-keyfile
chown mongod:mongod /etc/mongodb-keyfile
```

---

## Configuration des Config Servers

### Fichier mongod.conf - Config Server 1

```yaml
# /etc/mongod.conf - config-01

storage:
  dbPath: /data/configdb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2
      journalCompressor: snappy

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  timeStampFormat: iso8601-utc

net:
  port: 27019
  bindIp: 0.0.0.0

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

replication:
  replSetName: configReplSet

sharding:
  clusterRole: configsvr
```

### Configuration identique pour config-02 et config-03

Les fichiers `mongod.conf` pour `config-02` et `config-03` sont **identiques** à celui de `config-01`.

### Démarrage des Config Servers

```bash
# Sur config-01, config-02, config-03
sudo mkdir -p /data/configdb
sudo chown mongod:mongod /data/configdb
sudo systemctl start mongod
sudo systemctl enable mongod
```

### Initialisation du Config Server Replica Set

```bash
# Se connecter à config-01
mongosh --host config-01 --port 27019

# Initialiser le CSRS
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "config-01:27019" },
    { _id: 1, host: "config-02:27019" },
    { _id: 2, host: "config-03:27019" }
  ]
})

# Vérifier
rs.status()
```

---

## Configuration des Shards

### Fichier mongod.conf - Shard 1 Member 1

```yaml
# /etc/mongod.conf - shard1-01

storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 28  # 50% de 64 GB - 4 GB
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  timeStampFormat: iso8601-utc
  verbosity: 0

net:
  port: 27018
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

replication:
  replSetName: shard1-rs
  oplogSizeMB: 20480  # 20 GB

sharding:
  clusterRole: shardsvr

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

### Configuration pour tous les membres des shards

**Shard 1** : `shard1-01`, `shard1-02`, `shard1-03`
- replSetName: `shard1-rs`
- Fichier identique, ajuster cacheSizeGB selon RAM

**Shard 2** : `shard2-01`, `shard2-02`, `shard2-03`
- replSetName: `shard2-rs`
- Même configuration en changeant le replSetName

**Shard 3** : `shard3-01`, `shard3-02`, `shard3-03`
- replSetName: `shard3-rs`
- Même configuration en changeant le replSetName

### Démarrage des Shards

```bash
# Sur CHAQUE serveur de shard (9 serveurs au total)
sudo mkdir -p /data/mongodb
sudo chown -R mongod:mongod /data/mongodb
sudo systemctl start mongod
sudo systemctl enable mongod
```

### Initialisation des Replica Sets de shards

#### Shard 1

```bash
mongosh --host shard1-01 --port 27018

rs.initiate({
  _id: "shard1-rs",
  members: [
    { _id: 0, host: "shard1-01:27018" },
    { _id: 1, host: "shard1-02:27018" },
    { _id: 2, host: "shard1-03:27018" }
  ]
})

rs.status()
```

#### Shard 2

```bash
mongosh --host shard2-01 --port 27018

rs.initiate({
  _id: "shard2-rs",
  members: [
    { _id: 0, host: "shard2-01:27018" },
    { _id: 1, host: "shard2-02:27018" },
    { _id: 2, host: "shard2-03:27018" }
  ]
})

rs.status()
```

#### Shard 3

```bash
mongosh --host shard3-01 --port 27018

rs.initiate({
  _id: "shard3-rs",
  members: [
    { _id: 0, host: "shard3-01:27018" },
    { _id: 1, host: "shard3-02:27018" },
    { _id: 2, host: "shard3-03:27018" }
  ]
})

rs.status()
```

---

## Configuration des mongos (Query Routers)

### Fichier mongos.conf

```yaml
# /etc/mongos.conf - mongos-01, mongos-02, mongos-03

systemLog:
  destination: file
  path: /var/log/mongodb/mongos.log
  logAppend: true
  timeStampFormat: iso8601-utc

net:
  port: 27017
  bindIp: 0.0.0.0

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongos.pid

security:
  keyFile: /etc/mongodb-keyfile

sharding:
  configDB: configReplSet/config-01:27019,config-02:27019,config-03:27019
```

### Démarrage des mongos

```bash
# Sur mongos-01, mongos-02, mongos-03

# Créer le répertoire de logs
sudo mkdir -p /var/log/mongodb
sudo chown mongod:mongod /var/log/mongodb

# Démarrer mongos
sudo mongos --config /etc/mongos.conf

# OU via systemd (créer le service si nécessaire)
sudo systemctl start mongos
sudo systemctl enable mongos
```

### Service systemd pour mongos

```ini
# /etc/systemd/system/mongos.service
[Unit]
Description=MongoDB Router (mongos)
After=network.target

[Service]
User=mongod
Group=mongod
Type=forking
PIDFile=/var/run/mongodb/mongos.pid
ExecStart=/usr/bin/mongos --config /etc/mongos.conf
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

---

## Initialisation du Sharded Cluster

### Création de l'administrateur

```bash
# Se connecter à un mongos
mongosh --host mongos-01 --port 27017

# Créer l'administrateur
use admin

db.createUser({
  user: "admin",
  pwd: "SecureAdminPassword123!",  // CHANGER EN PRODUCTION
  roles: [
    { role: "root", db: "admin" }
  ]
})

exit
```

### Ajout des shards au cluster

```bash
# Se connecter en tant qu'admin
mongosh --host mongos-01 --port 27017 -u admin -p --authenticationDatabase admin

# Ajouter Shard 1
sh.addShard("shard1-rs/shard1-01:27018,shard1-02:27018,shard1-03:27018")

# Ajouter Shard 2
sh.addShard("shard2-rs/shard2-01:27018,shard2-02:27018,shard2-03:27018")

# Ajouter Shard 3
sh.addShard("shard3-rs/shard3-01:27018,shard3-02:27018,shard3-03:27018")

# Vérifier les shards
sh.status()
```

### Résultat attendu

```javascript
{
  shardingVersion: { ... },
  shards: [
    {
      _id: 'shard1-rs',
      host: 'shard1-rs/shard1-01:27018,shard1-02:27018,shard1-03:27018',
      state: 1
    },
    {
      _id: 'shard2-rs',
      host: 'shard2-rs/shard2-01:27018,shard2-02:27018,shard2-03:27018',
      state: 1
    },
    {
      _id: 'shard3-rs',
      host: 'shard3-rs/shard3-01:27018,shard3-02:27018,shard3-03:27018',
      state: 1
    }
  ]
}
```

---

## Activation du Sharding

### Activer le sharding sur une base de données

```javascript
// Se connecter à mongos
mongosh "mongodb://admin:SecureAdminPassword123!@mongos-01:27017/admin"

// Activer le sharding sur la base de données
sh.enableSharding("myapp")
```

### Créer un index sur la shard key

```javascript
use myapp

// Index sur le champ qui servira de shard key
db.users.createIndex({ user_id: 1 })
```

### Sharder une collection

#### Option 1 : Hashed Sharding

```javascript
// Distribution uniforme automatique
sh.shardCollection("myapp.users", { user_id: "hashed" })
```

#### Option 2 : Range Sharding

```javascript
// Distribution basée sur les valeurs
sh.shardCollection("myapp.orders", { created_at: 1, order_id: 1 })
```

### Vérifier le sharding

```javascript
// Statut du cluster
sh.status()

// Distribution des chunks
use config
db.chunks.find({ ns: "myapp.users" }).pretty()

// Statistiques par collection
db.users.getShardDistribution()
```

---

## Création des utilisateurs applicatifs

```javascript
// Se connecter à mongos en tant qu'admin
mongosh "mongodb://admin:SecureAdminPassword123!@mongos-01:27017/admin"

// Utilisateur read/write
use myapp

db.createUser({
  user: "myapp_rw",
  pwd: "AppReadWritePass456!",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// Utilisateur read only
db.createUser({
  user: "myapp_ro",
  pwd: "AppReadOnlyPass789!",
  roles: [
    { role: "read", db: "myapp" }
  ]
})

// Vérifier
db.getUsers()
```

---

## Connexion applicative

### Connection String

```javascript
// Connexion au sharded cluster via les mongos
mongodb://myapp_rw:AppReadWritePass456!@mongos-01:27017,mongos-02:27017,mongos-03:27017/myapp?authSource=myapp&w=majority

// Avec options de retry et timeout
mongodb://myapp_rw:AppReadWritePass456!@mongos-01:27017,mongos-02:27017,mongos-03:27017/myapp?authSource=myapp&retryWrites=true&w=majority&wtimeoutMS=5000
```

### Exemple Node.js

```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb://myapp_rw:AppReadWritePass456!@mongos-01:27017,mongos-02:27017,mongos-03:27017/myapp?authSource=myapp";

const client = new MongoClient(uri, {
  maxPoolSize: 100,
  minPoolSize: 20,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
});

async function main() {
  try {
    await client.connect();
    console.log("Connecté au Sharded Cluster");

    const db = client.db("myapp");
    const collection = db.collection("users");

    // Insert
    await collection.insertOne({
      user_id: 12345,
      name: "John Doe",
      email: "john@example.com"
    });

    // Find
    const user = await collection.findOne({ user_id: 12345 });
    console.log(user);

  } finally {
    await client.close();
  }
}

main().catch(console.error);
```

---

## Monitoring

### Commandes de monitoring essentielles

```javascript
// Statut global du cluster
sh.status()

// État des shards
db.adminCommand({ listShards: 1 })

// Statistiques de balancing
sh.getBalancerState()
sh.isBalancerRunning()

// Distribution des données
use config
db.databases.find()
db.collections.find()
db.chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])

// Opérations en cours sur les mongos
db.currentOp()
```

### Métriques par shard

```javascript
// Se connecter directement à un shard
mongosh "mongodb://admin:SecureAdminPassword123!@shard1-01:27018/admin"

// Statistiques du shard
db.serverStatus()

// Statistiques de stockage
db.stats()

// Collections sur ce shard
show collections
```

### Script de monitoring automatisé

```bash
#!/bin/bash
# /usr/local/bin/sharded-cluster-health.sh

echo "=== Sharded Cluster Health Check ==="
echo "Date: $(date)"
echo ""

# Vérifier les mongos
echo "--- mongos Status ---"
for host in mongos-01 mongos-02 mongos-03; do
  nc -zv $host 27017 2>&1 | grep -q succeeded && echo "$host: UP" || echo "$host: DOWN"
done
echo ""

# Vérifier les config servers
echo "--- Config Servers Status ---"
mongosh "mongodb://admin:PASSWORD@config-01:27019/admin" --quiet --eval "rs.status().members.forEach(m => print(m.name + ': ' + m.stateStr))"
echo ""

# Vérifier les shards
echo "--- Shards Status ---"
mongosh "mongodb://admin:PASSWORD@mongos-01:27017/admin" --quiet --eval "sh.status()" | grep -A 2 "shards:"
echo ""

# Balancer status
echo "--- Balancer Status ---"
mongosh "mongodb://admin:PASSWORD@mongos-01:27017/admin" --quiet --eval "sh.getBalancerState()"
echo ""
```

---

## Maintenance

### Ajout d'un nouveau shard

```javascript
// Se connecter à mongos
mongosh "mongodb://admin:PASSWORD@mongos-01:27017/admin"

// Initialiser le nouveau Replica Set (shard4-rs)
// (effectuer d'abord rs.initiate sur les serveurs du nouveau shard)

// Ajouter le shard au cluster
sh.addShard("shard4-rs/shard4-01:27018,shard4-02:27018,shard4-03:27018")

// Vérifier
sh.status()

// Le balancer redistribuera automatiquement les chunks
```

### Retrait d'un shard

```javascript
// Drainer un shard (déplacer toutes ses données)
use admin
db.adminCommand({ removeShard: "shard3-rs" })

// Vérifier la progression
db.adminCommand({ removeShard: "shard3-rs" })

// Une fois drainé (state: "completed"), le shard est retiré
```

### Gestion du balancer

```javascript
// Arrêter le balancer (pendant la maintenance)
sh.stopBalancer()
sh.getBalancerState()

// Redémarrer le balancer
sh.startBalancer()

// Programmer une fenêtre de balancing
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $set: { activeWindow: { start: "23:00", stop: "06:00" } } },
  { upsert: true }
)
```

### Backup d'un sharded cluster

```bash
#!/bin/bash
# Sauvegarde complète d'un sharded cluster

BACKUP_DIR="/backup/sharded-cluster"
DATE=$(date +%Y%m%d-%H%M%S)

# 1. Arrêter le balancer
mongosh "mongodb://admin:PASSWORD@mongos-01:27017/admin" --eval "sh.stopBalancer()"

# 2. Sauvegarder les config servers
mongodump --host config-01:27019 -u admin -p PASSWORD --authenticationDatabase admin \
  --out $BACKUP_DIR/$DATE/configdb --oplog

# 3. Sauvegarder chaque shard
for shard in shard1-02 shard2-02 shard3-02; do
  echo "Backup de $shard..."
  mongodump --host $shard:27018 -u admin -p PASSWORD --authenticationDatabase admin \
    --out $BACKUP_DIR/$DATE/$shard --oplog --gzip
done

# 4. Redémarrer le balancer
mongosh "mongodb://admin:PASSWORD@mongos-01:27017/admin" --eval "sh.startBalancer()"

# 5. Archiver
cd $BACKUP_DIR
tar -czf backup-$DATE.tar.gz $DATE/
rm -rf $DATE/

echo "Backup terminé : backup-$DATE.tar.gz"
```

---

## Optimisation des performances

### Choix de la shard key

#### Bonnes pratiques

```javascript
// ✅ BON : Haute cardinalité + bonne distribution
sh.shardCollection("myapp.orders", { customer_id: "hashed" })

// ✅ BON : Compound key avec granularité temporelle
sh.shardCollection("myapp.events", { event_date: 1, event_id: 1 })

// ❌ MAUVAIS : Monotone croissant (hotspot d'écriture)
sh.shardCollection("myapp.logs", { _id: 1 })

// ❌ MAUVAIS : Faible cardinalité
sh.shardCollection("myapp.users", { country: 1 })
```

#### Évaluation d'une shard key

```javascript
// Analyser la distribution
use myapp
db.users.aggregate([
  { $group: { _id: "$user_id", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 10 }
])

// Cardinalité
db.users.distinct("user_id").length
```

### Zones de sharding (Zone Sharding)

```javascript
// Définir des zones géographiques
sh.addShardTag("shard1-rs", "EU")
sh.addShardTag("shard2-rs", "US")
sh.addShardTag("shard3-rs", "ASIA")

// Assigner des ranges aux zones
sh.addTagRange(
  "myapp.users",
  { region: "EU", user_id: MinKey },
  { region: "EU", user_id: MaxKey },
  "EU"
)

sh.addTagRange(
  "myapp.users",
  { region: "US", user_id: MinKey },
  { region: "US", user_id: MaxKey },
  "US"
)

// Vérifier
use config
db.tags.find()
```

### Chunk size optimization

```javascript
// Modifier la taille des chunks (défaut: 128 MB)
use config
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 64 } },  // 64 MB
  { upsert: true }
)
```

---

## Troubleshooting

### Problème : Chunks déséquilibrés

```javascript
// Vérifier la distribution
use config
db.chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])

// Résultat attendu : répartition équilibrée
// Si déséquilibré, vérifier le balancer
sh.getBalancerState()
sh.startBalancer()

// Forcer un balancing
sh.startBalancer()
sh.setBalancerState(true)
```

### Problème : Jumbo chunks

```javascript
// Identifier les jumbo chunks
use config
db.chunks.find({ jumbo: true })

// Solutions :
// 1. Augmenter chunkSize si trop petit
// 2. Changer la shard key (nécessite re-sharding)
// 3. Split manuel si possible
sh.splitAt("myapp.users", { user_id: 50000 })
```

### Problème : Hotspot sur un shard

```javascript
// Identifier le shard surchargé
db.currentOp({ active: true })

// Vérifier les stats d'écriture par shard
db.serverStatus().opcounters

// Solutions :
// 1. Améliorer la shard key
// 2. Pre-split les chunks
// 3. Utiliser hashed sharding
```

### Problème : mongos indisponible

```bash
# Vérifier les logs
tail -f /var/log/mongodb/mongos.log

# Vérifier la connexion aux config servers
nc -zv config-01 27019
nc -zv config-02 27019
nc -zv config-03 27019

# Redémarrer mongos
sudo systemctl restart mongos
```

---

## Checklist de mise en production

### Infrastructure

- [ ] 15 serveurs déployés et accessibles
- [ ] DNS/hosts configurés sur tous les serveurs
- [ ] Firewall rules appliquées
- [ ] NTP synchronisé sur tous les serveurs
- [ ] THP désactivé partout
- [ ] Volumes XFS montés avec noatime

### Config Servers

- [ ] 3 config servers démarrés
- [ ] CSRS initialisé et stable
- [ ] Keyfile présent et identique

### Shards

- [ ] 3 shards × 3 membres = 9 serveurs démarrés
- [ ] 3 Replica Sets initialisés
- [ ] Tous les membres en état SECONDARY ou PRIMARY

### mongos

- [ ] 3 mongos démarrés
- [ ] Connectés aux config servers
- [ ] Load balancer configuré devant les mongos

### Sharded Cluster

- [ ] Utilisateur admin créé
- [ ] 3 shards ajoutés au cluster
- [ ] sh.status() affiche correctement les shards
- [ ] Base de données shardée
- [ ] Collections shardées avec shard key appropriée

### Sécurité

- [ ] Authentication activée partout
- [ ] Utilisateurs applicatifs créés
- [ ] Keyfile configuré
- [ ] TLS/SSL activé (production)
- [ ] Firewall rules validées

### Monitoring

- [ ] Monitoring configuré
- [ ] Alertes définies
- [ ] Scripts de health check en place
- [ ] Dashboard opérationnel

### Sauvegarde

- [ ] Stratégie de backup définie
- [ ] Scripts de backup testés
- [ ] Restoration testée
- [ ] Retention policy configurée

### Tests

- [ ] Test d'écriture/lecture réussi
- [ ] Test de failover shard réussi
- [ ] Test de failover mongos réussi
- [ ] Test de balancing validé
- [ ] Load test effectué

---

## Ressources complémentaires

### Documentation officielle

- [Deploy a Sharded Cluster](https://docs.mongodb.com/manual/tutorial/deploy-shard-cluster/)
- [Shard a Collection](https://docs.mongodb.com/manual/tutorial/shard-collection/)
- [Choose a Shard Key](https://docs.mongodb.com/manual/core/sharding-shard-key/)
- [Zones](https://docs.mongodb.com/manual/core/zone-sharding/)

### Commandes utiles

```javascript
// Aide sharding
sh.help()

// Statut complet
sh.status()

// Balancer
sh.getBalancerState()
sh.isBalancerRunning()

// Shards
db.adminCommand({ listShards: 1 })

// Chunks
use config
db.chunks.count()
db.chunks.find({ ns: "myapp.users" }).count()
```

---

**Configuration testée avec** : MongoDB 6.0+, 7.0+, 8.0+
**Nombre de serveurs** : 15 (production standard)

⏭️ [Configuration développement local](/annexes/configuration-reference/03-configuration-developpement-local.md)
