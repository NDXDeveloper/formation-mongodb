🔝 Retour au [Sommaire](/SOMMAIRE.md)

# F.2 - Replica Set avec Docker Compose

## Introduction

Configuration Docker Compose pour déployer un **Replica Set MongoDB** complet avec 3 membres. Fournit haute disponibilité, redondance des données et failover automatique.

### 🎯 Cas d'usage

- Environnements de développement avec HA
- Tests de failover et réplication
- Staging proche de la production
- Formation sur la réplication MongoDB
- Prototypage d'architectures résilientes
- Tests de performance avec read preference

### ✅ Avantages

```markdown
✅ Haute disponibilité (failover automatique)
✅ Redondance des données (3 copies)
✅ Lectures distribuées (read preference)
✅ Pas de perte de données en cas de panne
✅ Élection automatique du Primary
✅ Simulation réaliste de production
```

### ⚠️ Limitations

```markdown
⚠️ Consomme 3x plus de ressources (3 conteneurs)
⚠️ Plus complexe à gérer qu'un standalone
⚠️ Nécessite 6-8 GB RAM minimum
⚠️ Latence réseau entre membres (conteneurs)
```

---

## Architecture

### 🏗️ Composants

```
┌─────────────────────────────────────────┐
│          Replica Set : rs0              │
├─────────────────────────────────────────┤
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  │
│  │ mongo1  │  │ mongo2  │  │ mongo3  │  │
│  │         │  │         │  │         │  │
│  │ PRIMARY │  │SECONDARY│  │SECONDARY│  │
│  │ :27017  │  │ :27018  │  │ :27019  │  │
│  └─────────┘  └─────────┘  └─────────┘  │
│       │            │            │       │
│       └────────────┴────────────┘       │
│          mongodb_network                │
└─────────────────────────────────────────┘
```

**Flux de données** :
- Écritures → Primary uniquement
- Lectures → Primary ou Secondaries (selon read preference)
- Réplication → Oplog synchronisé automatiquement
- Failover → Élection automatique si Primary down

---

## Configuration Standard (3 Membres)

### 📦 Structure de Projet

```
mongodb-replica-set/
├── docker-compose.yml
├── .env
├── .gitignore
├── scripts/
│   ├── rs-init.sh
│   └── healthcheck.sh
├── init-scripts/
│   └── 01-setup.js
├── data/
│   ├── mongo1/
│   ├── mongo2/
│   └── mongo3/
└── logs/
    ├── mongo1/
    ├── mongo2/
    └── mongo3/
```

---

### 🐳 docker-compose.yml

```yaml
version: '3.8'

services:
  mongo1:
    image: mongo:7.0
    container_name: mongo1
    hostname: mongo1
    restart: unless-stopped

    command: mongod --replSet rs0 --bind_ip_all

    ports:
      - "27017:27017"

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}

    volumes:
      - mongo1_data:/data/db
      - mongo1_config:/data/configdb
      - ./logs/mongo1:/var/log/mongodb

    networks:
      - mongodb_network

    healthcheck:
      test: |
        mongosh --quiet --eval "
          try {
            rs.status();
            print('Replica set initialized');
          } catch(e) {
            print('Replica set not initialized');
            exit(1);
          }
        " || exit 1
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  mongo2:
    image: mongo:7.0
    container_name: mongo2
    hostname: mongo2
    restart: unless-stopped

    command: mongod --replSet rs0 --bind_ip_all

    ports:
      - "27018:27017"

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}

    volumes:
      - mongo2_data:/data/db
      - mongo2_config:/data/configdb
      - ./logs/mongo2:/var/log/mongodb

    networks:
      - mongodb_network

    healthcheck:
      test: |
        mongosh --quiet --eval "db.adminCommand('ping')" || exit 1
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

  mongo3:
    image: mongo:7.0
    container_name: mongo3
    hostname: mongo3
    restart: unless-stopped

    command: mongod --replSet rs0 --bind_ip_all

    ports:
      - "27019:27017"

    environment:
      MONGO_INITDB_ROOT_USERNAME: ${MONGO_ROOT_USERNAME}
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}

    volumes:
      - mongo3_data:/data/db
      - mongo3_config:/data/configdb
      - ./logs/mongo3:/var/log/mongodb

    networks:
      - mongodb_network

    healthcheck:
      test: |
        mongosh --quiet --eval "db.adminCommand('ping')" || exit 1
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

volumes:
  mongo1_data:
  mongo1_config:
  mongo2_data:
  mongo2_config:
  mongo3_data:
  mongo3_config:

networks:
  mongodb_network:
    driver: bridge
```

**.env** :
```bash
# MongoDB Version
MONGO_VERSION=7.0

# Authentication
MONGO_ROOT_USERNAME=admin
MONGO_ROOT_PASSWORD=SecureReplicaPassword123!

# Replica Set
REPLICA_SET_NAME=rs0
```

**.gitignore** :
```
.env
data/
logs/
*.log
```

---

### 🚀 Script d'Initialisation du Replica Set

**scripts/rs-init.sh** :
```bash
#!/bin/bash
# rs-init.sh - Initialize MongoDB Replica Set

echo "Waiting for MongoDB instances to be ready..."
sleep 10

echo "Initializing Replica Set..."

docker exec -it mongo1 mongosh --eval "
rs.initiate({
  _id: 'rs0',
  members: [
    { _id: 0, host: 'mongo1:27017', priority: 2 },
    { _id: 1, host: 'mongo2:27017', priority: 1 },
    { _id: 2, host: 'mongo3:27017', priority: 1 }
  ]
})
"

echo "Waiting for Replica Set to stabilize..."
sleep 15

echo "Checking Replica Set status..."
docker exec -it mongo1 mongosh --eval "rs.status()"

echo "Creating admin user..."
docker exec -it mongo1 mongosh --eval "
db.getSiblingDB('admin').createUser({
  user: '${MONGO_ROOT_USERNAME:-admin}',
  pwd: '${MONGO_ROOT_PASSWORD:-password}',
  roles: [
    { role: 'root', db: 'admin' }
  ]
})
"

echo "Replica Set initialized successfully!"
echo ""
echo "Connection string:"
echo "mongodb://${MONGO_ROOT_USERNAME:-admin}:${MONGO_ROOT_PASSWORD:-password}@localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&authSource=admin"
```

Rendre exécutable :
```bash
chmod +x scripts/rs-init.sh
```

---

### 📋 Déploiement Complet

```bash
# 1. Créer le projet
mkdir mongodb-replica-set && cd mongodb-replica-set
mkdir -p scripts logs/{mongo1,mongo2,mongo3}

# 2. Créer les fichiers
# (copier docker-compose.yml, .env, rs-init.sh)

# 3. Démarrer les conteneurs
docker-compose up -d

# 4. Attendre que tous les conteneurs soient healthy
docker-compose ps

# 5. Initialiser le Replica Set
./scripts/rs-init.sh

# 6. Vérifier le status
docker exec -it mongo1 mongosh --eval "rs.status()"

# 7. Se connecter
mongosh "mongodb://admin:SecureReplicaPassword123!@localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&authSource=admin"
```

---

## Configuration Avancée

### ⚙️ Avec Options de Priorité et Hidden Members

**docker-compose.yml (extrait)** :
```yaml
version: '3.8'

services:
  mongo1:
    # ... config de base ...
    environment:
      MONGO_REPLICA_PRIORITY: 2  # Primary préféré

  mongo2:
    # ... config de base ...
    environment:
      MONGO_REPLICA_PRIORITY: 1  # Secondary standard

  mongo3:
    # ... config de base ...
    environment:
      MONGO_REPLICA_PRIORITY: 0  # Hidden member
      MONGO_REPLICA_HIDDEN: true
    ports:
      - "127.0.0.1:27019:27017"  # Non exposé publiquement
```

**Script d'initialisation avec hidden member** :
```javascript
// rs-init-advanced.js
rs.initiate({
  _id: 'rs0',
  members: [
    {
      _id: 0,
      host: 'mongo1:27017',
      priority: 2  // Primary préféré
    },
    {
      _id: 1,
      host: 'mongo2:27017',
      priority: 1
    },
    {
      _id: 2,
      host: 'mongo3:27017',
      priority: 0,      // Ne peut pas devenir Primary
      hidden: true,     // Invisible aux clients
      votes: 1          // Participe aux élections
    }
  ]
});
```

Utilisation du hidden member :
```markdown
✅ Backup sans impact sur les lectures
✅ Analytics sans charger le Primary
✅ Testing sans perturber la prod
```

---

### 🔐 Avec Authentification Keyfile

Pour sécuriser la communication inter-membres :

**Générer le keyfile** :
```bash
# Créer le keyfile
openssl rand -base64 756 > ./mongodb-keyfile
chmod 400 ./mongodb-keyfile
sudo chown 999:999 ./mongodb-keyfile
```

**docker-compose.yml (modifié)** :
```yaml
services:
  mongo1:
    command: |
      bash -c '
      chmod 400 /data/keyfile
      chown 999:999 /data/keyfile
      mongod --replSet rs0 --bind_ip_all --keyFile /data/keyfile
      '
    volumes:
      - mongo1_data:/data/db
      - ./mongodb-keyfile:/data/keyfile:ro

  # Répéter pour mongo2 et mongo3
```

---

### 📊 Avec Monitoring (Mongo Express)

```yaml
version: '3.8'

services:
  mongo1:
    # ... (config replica set) ...

  mongo2:
    # ... (config replica set) ...

  mongo3:
    # ... (config replica set) ...

  mongo-express:
    image: mongo-express:latest
    container_name: mongo-express
    restart: unless-stopped

    ports:
      - "8081:8081"

    environment:
      ME_CONFIG_MONGODB_URL: mongodb://admin:${MONGO_ROOT_PASSWORD}@mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0&authSource=admin
      ME_CONFIG_BASICAUTH_USERNAME: admin
      ME_CONFIG_BASICAUTH_PASSWORD: admin

    depends_on:
      - mongo1
      - mongo2
      - mongo3

    networks:
      - mongodb_network
```

---

## Gestion du Replica Set

### 🔍 Commandes de Vérification

```javascript
// Se connecter au Primary
mongosh "mongodb://admin:password@localhost:27017/?replicaSet=rs0&authSource=admin"

// Status du Replica Set
rs.status()

// Configuration
rs.conf()

// Qui est le Primary ?
rs.isMaster()

// Informations sur la réplication
db.printReplicationInfo()

// Lag des secondaries
rs.printSlaveReplicationInfo()  // Deprecated
rs.printSecondaryReplicationInfo()  // Nouveau

// Lister les membres
rs.status().members.forEach(m => {
  print(m.name + ' - ' + m.stateStr);
});
```

---

### ⚙️ Modification de Configuration

**Changer les priorités** :
```javascript
cfg = rs.conf();
cfg.members[0].priority = 3;  // mongo1 priorité haute
cfg.members[1].priority = 1;  // mongo2 priorité normale
cfg.members[2].priority = 0.5;  // mongo3 priorité basse
rs.reconfig(cfg);
```

**Ajouter un membre** :
```javascript
rs.add({
  host: "mongo4:27017",
  priority: 1
});
```

**Retirer un membre** :
```javascript
rs.remove("mongo3:27017");
```

**Forcer une élection** :
```javascript
rs.stepDown();  // Force le Primary actuel à devenir Secondary
```

---

## Tests de Haute Disponibilité

### 🧪 Test de Failover

**Scénario 1 : Arrêt du Primary** :
```bash
# 1. Identifier le Primary
docker exec mongo1 mongosh --quiet --eval "rs.isMaster().primary"

# 2. Simuler une panne (arrêter le conteneur Primary)
docker stop mongo1

# 3. Observer l'élection (10-12 secondes)
docker exec mongo2 mongosh --eval "rs.status()" | grep -A5 "stateStr"

# 4. Vérifier le nouveau Primary
docker exec mongo2 mongosh --quiet --eval "rs.isMaster().primary"

# 5. Tester les écritures sur le nouveau Primary
docker exec mongo2 mongosh --eval "
  db.getSiblingDB('test').items.insertOne({msg: 'Failover test', date: new Date()})
"

# 6. Redémarrer l'ancien Primary (devient Secondary)
docker start mongo1

# 7. Vérifier la resynchronisation
docker exec mongo1 mongosh --eval "rs.status()" | grep -A5 "stateStr"
```

**Scénario 2 : Perte de 2 membres (perte de majorité)** :
```bash
# Arrêter 2 membres
docker stop mongo2 mongo3

# Le Primary devient READ-ONLY
docker exec mongo1 mongosh --eval "
  db.getSiblingDB('test').items.insertOne({msg: 'Should fail'})
"
# Erreur : "not master and slaveOk=false"

# Redémarrer les membres
docker start mongo2 mongo3
```

---

### 📖 Test de Read Preference

```javascript
// Se connecter avec read preference
mongosh "mongodb://admin:password@localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&authSource=admin&readPreference=secondaryPreferred"

// Tester différentes stratégies
db.getMongo().setReadPref("primary");           // Toujours Primary
db.getMongo().setReadPref("secondary");         // Toujours Secondary
db.getMongo().setReadPref("primaryPreferred");  // Primary si dispo
db.getMongo().setReadPref("secondaryPreferred");// Secondary si dispo
db.getMongo().setReadPref("nearest");           // Plus faible latence

// Vérifier sur quel membre on lit
db.runCommand({isMaster: 1}).me
```

**Depuis une application** :
```javascript
// Node.js
const client = new MongoClient(
  'mongodb://admin:password@localhost:27017,localhost:27018,localhost:27019',
  {
    replicaSet: 'rs0',
    readPreference: 'secondaryPreferred',
    w: 'majority',
    retryWrites: true
  }
);
```

---

## Write Concern et Read Concern

### ✍️ Write Concern

```javascript
// Attendre que l'écriture soit répliquée sur la majorité
db.collection.insertOne(
  { data: "value" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
);

// Configurations possibles :
w: 1            // Écrit sur le Primary uniquement (par défaut)
w: 2            // Primary + 1 Secondary
w: 3            // Tous les membres
w: "majority"   // Majorité des membres (recommandé)
w: 0            // Fire and forget (⚠️ dangereux)

j: true         // Attend la journalisation
wtimeout: 5000  // Timeout en ms
```

---

### 📖 Read Concern

```javascript
// Lire uniquement les données répliquées sur la majorité
db.collection.find().readConcern("majority");

// Niveaux disponibles :
"local"         // Données du membre local (par défaut)
"available"     // Données disponibles localement
"majority"      // Données confirmées par majorité
"linearizable"  // Lecture linéarisable (lente mais forte cohérence)
"snapshot"      // Snapshot dans une transaction
```

---

## Connexion depuis Applications

### 🔗 Connection Strings

**Format standard** :
```bash
mongodb://username:password@host1:port1,host2:port2,host3:port3/?options

# Exemple complet
mongodb://admin:SecurePassword123!@localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&authSource=admin&readPreference=secondaryPreferred&w=majority
```

**Options importantes** :
```bash
replicaSet=rs0              # OBLIGATOIRE pour Replica Set
authSource=admin            # Database d'authentification
readPreference=secondaryPreferred
w=majority                  # Write concern
retryWrites=true           # Retry automatique
maxPoolSize=100            # Pool de connexions
```

---

### 💻 Exemples de Code

**Node.js (MongoDB Driver)** :
```javascript
const { MongoClient } = require('mongodb');

const uri = 'mongodb://admin:password@localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&authSource=admin';

const client = new MongoClient(uri, {
  useNewUrlParser: true,
  useUnifiedTopology: true,

  // Options Replica Set
  replicaSet: 'rs0',
  readPreference: 'secondaryPreferred',
  w: 'majority',
  retryWrites: true,

  // Connection pooling
  maxPoolSize: 50,
  minPoolSize: 10,

  // Timeouts
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000
});

async function main() {
  try {
    await client.connect();
    console.log('Connected to Replica Set');

    const db = client.db('mydb');

    // Écriture avec write concern
    const result = await db.collection('users').insertOne(
      { name: 'John', email: 'john@example.com' },
      { writeConcern: { w: 'majority', wtimeout: 5000 } }
    );

    // Lecture avec read preference
    const users = await db.collection('users')
      .find()
      .readPreference('secondaryPreferred')
      .toArray();

    console.log(users);
  } finally {
    await client.close();
  }
}

main();
```

**Python (PyMongo)** :
```python
from pymongo import MongoClient, ReadPreference

uri = 'mongodb://admin:password@localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs0&authSource=admin'

client = MongoClient(
    uri,
    replicaSet='rs0',
    readPreference=ReadPreference.SECONDARY_PREFERRED,
    w='majority',
    retryWrites=True
)

try:
    db = client.mydb

    # Écriture
    result = db.users.insert_one({'name': 'John', 'email': 'john@example.com'})

    # Lecture depuis Secondary
    users = list(db.users.find())
    print(users)

finally:
    client.close()
```

---

### 🐳 Avec Application Dockerisée

**docker-compose.yml complet** :
```yaml
version: '3.8'

services:
  mongo1:
    # ... (config replica set)

  mongo2:
    # ... (config replica set)

  mongo3:
    # ... (config replica set)

  app:
    build: ./app
    container_name: myapp

    environment:
      MONGODB_URI: mongodb://admin:${MONGO_ROOT_PASSWORD}@mongo1:27017,mongo2:27017,mongo3:27017/?replicaSet=rs0&authSource=admin&readPreference=secondaryPreferred&w=majority
      NODE_ENV: development

    ports:
      - "3000:3000"

    depends_on:
      - mongo1
      - mongo2
      - mongo3

    networks:
      - mongodb_network

    restart: unless-stopped

networks:
  mongodb_network:
    driver: bridge
```

---

## Backup et Restauration

### 💾 Backup du Replica Set

**Option 1 : mongodump depuis un Secondary** :
```bash
# Backup depuis mongo3 (hidden member idéal)
docker exec mongo3 mongodump \
  --host=mongo3:27017 \
  --username=admin \
  --password=SecurePassword123! \
  --authenticationDatabase=admin \
  --readPreference=secondary \
  --out=/backup

# Copier sur l'hôte
docker cp mongo3:/backup ./backup-$(date +%Y%m%d)
```

**Option 2 : Backup des volumes** :
```bash
# Script de backup
#!/bin/bash

DATE=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="./backups/$DATE"

mkdir -p $BACKUP_DIR

# Backup de chaque volume
for member in mongo1 mongo2 mongo3; do
  echo "Backing up $member..."
  docker run --rm \
    -v mongodb-replica-set_${member}_data:/data:ro \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/$DATE/${member}-data.tar.gz /data
done

echo "Backup completed: $BACKUP_DIR"
```

---

### 🔄 Restauration

**mongorestore** :
```bash
# Copier le backup dans un conteneur
docker cp ./backup-20240101 mongo1:/backup

# Restaurer
docker exec mongo1 mongorestore \
  --host=mongo1:27017 \
  --username=admin \
  --password=SecurePassword123! \
  --authenticationDatabase=admin \
  /backup

# La réplication propagera automatiquement aux secondaries
```

---

### 🤖 Backup Automatisé avec Cron

**scripts/backup-replica-set.sh** :
```bash
#!/bin/bash

BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d-%H%M%S)
RETENTION_DAYS=7

# Backup depuis le hidden member (mongo3)
docker exec mongo3 mongodump \
  --host=mongo3:27017 \
  --username=admin \
  --password=${MONGO_ROOT_PASSWORD} \
  --authenticationDatabase=admin \
  --readPreference=secondary \
  --gzip \
  --archive=/backup/replica-set-$DATE.gz

# Copier sur l'hôte
docker cp mongo3:/backup/replica-set-$DATE.gz $BACKUP_DIR/

# Nettoyer dans le conteneur
docker exec mongo3 rm /backup/replica-set-$DATE.gz

# Supprimer les backups > RETENTION_DAYS
find $BACKUP_DIR -name "replica-set-*.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completed: $BACKUP_DIR/replica-set-$DATE.gz"
```

**Cron** :
```bash
# Backup quotidien à 2h du matin
0 2 * * * /path/to/scripts/backup-replica-set.sh >> /var/log/mongodb-backup.log 2>&1
```

---

## Monitoring et Maintenance

### 📊 Monitoring du Replica Set

**Commandes essentielles** :
```bash
# Status général
docker exec mongo1 mongosh --eval "rs.status()" | grep -E "name|stateStr|health"

# Lag de réplication
docker exec mongo1 mongosh --eval "rs.printSecondaryReplicationInfo()"

# Oplog
docker exec mongo1 mongosh --eval "db.getReplicationInfo()"

# Connexions actives par membre
for member in mongo1 mongo2 mongo3; do
  echo "=== $member ==="
  docker exec $member mongosh --quiet --eval "db.serverStatus().connections"
done
```

---

### 📈 Script de Monitoring

**scripts/monitor-replica-set.sh** :
```bash
#!/bin/bash

echo "========================================="
echo "MongoDB Replica Set Monitoring"
echo "Date: $(date)"
echo "========================================="

# Status des conteneurs
echo -e "\n### Container Status ###"
docker-compose ps

# Replica Set Status
echo -e "\n### Replica Set Status ###"
docker exec mongo1 mongosh --quiet --eval "
  var status = rs.status();
  print('Set: ' + status.set);
  print('Members:');
  status.members.forEach(m => {
    print('  ' + m.name + ' - ' + m.stateStr + ' (health: ' + m.health + ')');
  });
"

# Replication Lag
echo -e "\n### Replication Lag ###"
docker exec mongo1 mongosh --quiet --eval "rs.printSecondaryReplicationInfo()"

# Oplog Window
echo -e "\n### Oplog Window ###"
docker exec mongo1 mongosh --quiet --eval "
  var info = db.getReplicationInfo();
  print('Oplog Size: ' + (info.logSizeMB / 1024).toFixed(2) + ' GB');
  print('Time Diff: ' + (info.timeDiff / 3600).toFixed(2) + ' hours');
"

# Connexions
echo -e "\n### Connections ###"
for member in mongo1 mongo2 mongo3; do
  echo "$member:"
  docker exec $member mongosh --quiet --eval "
    var conn = db.serverStatus().connections;
    print('  Current: ' + conn.current + ' / Available: ' + conn.available);
  "
done

echo -e "\n========================================="
```

---

### 🔧 Maintenance

**Redémarrage rolling (sans downtime)** :
```bash
# 1. Redémarrer les secondaries un par un
docker-compose restart mongo3
sleep 30
docker-compose restart mongo2
sleep 30

# 2. Forcer step down du primary
docker exec mongo1 mongosh --eval "rs.stepDown()"
sleep 15

# 3. Redémarrer l'ancien primary (maintenant secondary)
docker-compose restart mongo1
```

**Mise à jour de version** :
```bash
# 1. Backup complet
./scripts/backup-replica-set.sh

# 2. Modifier docker-compose.yml
# image: mongo:7.0 → image: mongo:8.0

# 3. Rolling update
docker-compose up -d mongo3
# Attendre synchronisation
docker exec mongo1 mongosh --eval "rs.printSecondaryReplicationInfo()"

docker-compose up -d mongo2
# Attendre synchronisation

docker exec mongo1 mongosh --eval "rs.stepDown()"
docker-compose up -d mongo1

# 4. Vérifier la version
docker exec mongo1 mongosh --eval "db.version()"
```

---

## Dépannage

### 🔧 Problèmes Courants

#### Replica Set ne s'initialise pas

```bash
# Vérifier les logs
docker-compose logs mongo1 | grep -i replica

# Vérifier la résolution DNS
docker exec mongo1 ping mongo2
docker exec mongo1 ping mongo3

# Réinitialiser (⚠️ perte de config RS)
docker exec mongo1 mongosh --eval "
  use local;
  db.system.replset.deleteOne({});
"
docker-compose restart mongo1
./scripts/rs-init.sh
```

#### Membre en état STARTUP ou RECOVERING

```bash
# Vérifier le status
docker exec mongo2 mongosh --eval "rs.status()"

# Vérifier les logs
docker logs mongo2 --tail 100

# Resync complet si nécessaire
docker exec mongo2 mongosh --eval "
  db.adminCommand({resync: 1})
"
```

#### Lag de réplication élevé

```bash
# Identifier le lag
docker exec mongo1 mongosh --eval "rs.printSecondaryReplicationInfo()"

# Causes possibles :
# 1. Charge élevée sur secondary
docker stats mongo2

# 2. Oplog trop petit
docker exec mongo1 mongosh --eval "db.getReplicationInfo()"

# 3. Requêtes lentes sur secondary
docker exec mongo2 mongosh --eval "db.currentOp()" | grep secs_running

# Solutions :
# - Augmenter oplog
# - Optimiser les index
# - Limiter les lectures sur secondary
```

#### Split Brain (rare avec Docker)

```bash
# Vérifier qu'il n'y a qu'un seul Primary
for member in mongo1 mongo2 mongo3; do
  docker exec $member mongosh --quiet --eval "rs.isMaster().ismaster"
done

# Si 2 Primary (split brain) :
# 1. Arrêter un membre
docker stop mongo3

# 2. Forcer reconfiguration
docker exec mongo1 mongosh --eval "rs.reconfig(rs.conf(), {force: true})"

# 3. Redémarrer le membre
docker start mongo3
```

---

## Checklist de Déploiement

### ✅ Avant le Démarrage

```markdown
□ Docker et Docker Compose installés (20.10+, 2.0+)
□ Ressources suffisantes (6-8 GB RAM, 4 cores)
□ Ports libres : 27017, 27018, 27019
□ .env créé avec mots de passe forts
□ .env dans .gitignore
□ Scripts rs-init.sh exécutable
□ Réseau Docker configuré
□ Volumes définis dans docker-compose.yml
```

### ✅ Après le Démarrage

```markdown
□ 3 conteneurs démarrés : docker-compose ps
□ Healthchecks PASSING
□ Replica Set initialisé : rs.status()
□ 1 Primary, 2 Secondaries identifiés
□ Authentification fonctionne
□ Connection string validée
□ Failover testé (optionnel)
□ Backup strategy en place
□ Monitoring configuré
□ Documentation à jour
```

---

## Ressources

### 📚 Documentation
- [Replication](https://www.mongodb.com/docs/manual/replication/)
- [Deploy a Replica Set](https://www.mongodb.com/docs/manual/tutorial/deploy-replica-set/)
- [Replica Set Configuration](https://www.mongodb.com/docs/manual/reference/replica-configuration/)

### 🔗 Liens Utiles
- [Read Preference](https://www.mongodb.com/docs/manual/core/read-preference/)
- [Write Concern](https://www.mongodb.com/docs/manual/reference/write-concern/)
- [Replica Set Elections](https://www.mongodb.com/docs/manual/core/replica-set-elections/)

---

## Conclusion

Le Replica Set avec Docker Compose est **idéal pour** :
- ✅ Développement avec HA
- ✅ Tests de failover
- ✅ Staging proche production
- ✅ Formation réplication MongoDB
- ✅ Prototypage architectures résilientes

**Passer à la production** :
- Utiliser Kubernetes pour orchestration
- MongoDB Atlas pour solution managée
- Déployer sur infrastructure dédiée

**Prochaine étape** : F.3 - Sharded Cluster avec Docker Compose

---

**Version** : 1.0
**Testé avec** : MongoDB 7.0, Docker 24.0, Docker Compose 2.23

⏭️ [Sharded Cluster avec Docker Compose](/annexes/docker-compose/03-sharded-cluster-docker-compose.md)
