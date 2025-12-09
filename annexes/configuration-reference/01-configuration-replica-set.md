🔝 Retour au [Sommaire](/SOMMAIRE.md)

# D.1 - Configuration Replica Set (3 nœuds)

## Présentation

### Objectif
Configuration standard de haute disponibilité avec 3 membres MongoDB pour assurer la résilience et la continuité de service en cas de panne d'un nœud.

### Topologie

```
┌─────────────────────────────────────────────────────────────┐
│                    Replica Set: rs-prod                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐ │
│  │   PRIMARY    │     │  SECONDARY   │     │  SECONDARY   │ │
│  │              │────▶│              │────▶│              │ │
│  │ mongo-rs-01  │     │ mongo-rs-02  │     │ mongo-rs-03  │ │
│  │  27017       │◀────│  27017       │◀────│  27017       │ │
│  └──────────────┘     └──────────────┘     └──────────────┘ │
│         │                     │                     │       │
│         └─────────────────────┴─────────────────────┘       │
│                    Réplication bidirectionnelle             │
└─────────────────────────────────────────────────────────────┘
```

### Caractéristiques

- **Haute disponibilité** : Failover automatique en cas de panne du Primary
- **Lecture distribuée** : Possibilité de lire depuis les Secondary
- **Élection automatique** : Nouveau Primary élu en < 10 secondes
- **Cohérence des données** : Réplication asynchrone avec oplog
- **Tolérance aux pannes** : Survit à la perte de 1 nœud

---

## Prérequis matériels

### Configuration minimale (par nœud)

| Ressource | Minimum | Recommandé |
|-----------|---------|------------|
| **CPU** | 2 cores | 4 cores |
| **RAM** | 8 GB | 16 GB |
| **Stockage** | 100 GB SSD | 500 GB NVMe SSD |
| **Réseau** | 1 Gbps | 10 Gbps |
| **Latence réseau** | < 50ms | < 10ms |

### Configuration recommandée (production)

| Ressource | Spécification |
|-----------|---------------|
| **CPU** | 8 cores @ 3+ GHz |
| **RAM** | 32 GB |
| **Stockage OS** | 50 GB SSD |
| **Stockage données** | 1 TB NVMe SSD (RAID 10) |
| **Réseau** | 10 Gbps avec redondance |

### Système d'exploitation

- **Ubuntu** : 22.04 LTS ou 24.04 LTS
- **Rocky Linux** : 9.x
- **Debian** : 12
- **MongoDB** : 6.0+, 7.0+ ou 8.0+

---

## Architecture réseau

### Plan d'adressage

| Nœud | Hostname | IP Privée | Rôle Initial | Port |
|------|----------|-----------|--------------|------|
| Nœud 1 | mongo-rs-01 | 10.0.1.10 | PRIMARY | 27017 |
| Nœud 2 | mongo-rs-02 | 10.0.1.11 | SECONDARY | 27017 |
| Nœud 3 | mongo-rs-03 | 10.0.1.12 | SECONDARY | 27017 |

### Règles firewall

```bash
# Autoriser MongoDB entre les nœuds du Replica Set
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.10 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.11 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.12 -j ACCEPT

# Autoriser les applications
iptables -A INPUT -p tcp --dport 27017 -s 10.0.2.0/24 -j ACCEPT

# Bloquer tout le reste
iptables -A INPUT -p tcp --dport 27017 -j DROP
```

### Résolution DNS

```bash
# /etc/hosts sur chaque nœud
10.0.1.10   mongo-rs-01
10.0.1.11   mongo-rs-02
10.0.1.12   mongo-rs-03
```

---

## Configuration système

### Paramètres kernel Linux

```bash
# /etc/sysctl.conf - À appliquer sur tous les nœuds
vm.swappiness = 1
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
net.core.somaxconn = 4096
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 5
net.ipv4.tcp_keepalive_intvl = 15
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
mongod soft memlock unlimited
mongod hard memlock unlimited
```

### Désactivation de Transparent Huge Pages

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

# Activer le service
sudo systemctl daemon-reload
sudo systemctl enable disable-thp.service
sudo systemctl start disable-thp.service
```

---

## Préparation des volumes

### Structure des répertoires

```bash
# Créer les répertoires sur chaque nœud
sudo mkdir -p /data/mongodb
sudo mkdir -p /var/log/mongodb
sudo mkdir -p /var/run/mongodb

# Définir les propriétaires
sudo chown -R mongod:mongod /data/mongodb
sudo chown -R mongod:mongod /var/log/mongodb
sudo chown -R mongod:mongod /var/run/mongodb

# Permissions
sudo chmod 755 /data/mongodb
sudo chmod 755 /var/log/mongodb
sudo chmod 755 /var/run/mongodb
```

### Formatage XFS (recommandé)

```bash
# Formatter le volume de données en XFS
sudo mkfs.xfs -f /dev/sdb

# Monter avec les options optimisées
sudo mount -t xfs -o noatime,nobarrier /dev/sdb /data/mongodb

# /etc/fstab - Montage automatique
/dev/sdb  /data/mongodb  xfs  noatime,nobarrier  0 0
```

---

## Configuration MongoDB

### Génération du Keyfile (sécurité)

```bash
# Générer sur le premier nœud
openssl rand -base64 756 > /etc/mongodb-keyfile
chmod 400 /etc/mongodb-keyfile
chown mongod:mongod /etc/mongodb-keyfile

# Copier sur les autres nœuds (même contenu exact)
scp /etc/mongodb-keyfile root@mongo-rs-02:/etc/
scp /etc/mongodb-keyfile root@mongo-rs-03:/etc/

# Permissions sur tous les nœuds
sudo chmod 400 /etc/mongodb-keyfile
sudo chown mongod:mongod /etc/mongodb-keyfile
```

### Fichier de configuration - Nœud 1 (PRIMARY)

```yaml
# /etc/mongod.conf - mongo-rs-01

# Stockage
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 12  # 50% RAM - 1 GB pour serveur 16 GB
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

# Journalisation
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc
  verbosity: 0
  component:
    replication:
      verbosity: 1

# Réseau
net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536
  compression:
    compressors: snappy,zstd,zlib

# Processus
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

# Sécurité
security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

# Réplication
replication:
  replSetName: rs-prod
  oplogSizeMB: 10240  # 10 GB

# Paramètres opérationnels
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

### Fichier de configuration - Nœud 2 (SECONDARY)

```yaml
# /etc/mongod.conf - mongo-rs-02
# Même configuration que mongo-rs-01
# Seul le bindIp pourrait différer si nécessaire

storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 12
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc
  verbosity: 0
  component:
    replication:
      verbosity: 1

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536
  compression:
    compressors: snappy,zstd,zlib

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

replication:
  replSetName: rs-prod
  oplogSizeMB: 10240

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

### Fichier de configuration - Nœud 3 (SECONDARY)

```yaml
# /etc/mongod.conf - mongo-rs-03
# Identique aux nœuds 1 et 2

storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 12
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc
  verbosity: 0
  component:
    replication:
      verbosity: 1

net:
  port: 27017
  bindIp: 0.0.0.0
  maxIncomingConnections: 65536
  compression:
    compressors: snappy,zstd,zlib

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

security:
  authorization: enabled
  keyFile: /etc/mongodb-keyfile

replication:
  replSetName: rs-prod
  oplogSizeMB: 10240

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

---

## Déploiement

### Étape 1 : Démarrage des instances

```bash
# Sur chaque nœud (mongo-rs-01, mongo-rs-02, mongo-rs-03)
sudo systemctl start mongod
sudo systemctl enable mongod

# Vérifier le démarrage
sudo systemctl status mongod
tail -f /var/log/mongodb/mongod.log
```

### Étape 2 : Initialisation du Replica Set

Se connecter au premier nœud **sans authentification** (la première fois) :

```bash
# Se connecter à mongo-rs-01
mongosh --host mongo-rs-01 --port 27017
```

Initialiser le Replica Set :

```javascript
// Configuration du Replica Set
rs.initiate({
  _id: "rs-prod",
  members: [
    { _id: 0, host: "mongo-rs-01:27017", priority: 2 },
    { _id: 1, host: "mongo-rs-02:27017", priority: 1 },
    { _id: 2, host: "mongo-rs-03:27017", priority: 1 }
  ]
})

// Attendre quelques secondes puis vérifier
rs.status()
```

### Étape 3 : Création de l'administrateur

```javascript
// Sur le PRIMARY (mongo-rs-01 normalement)
use admin

db.createUser({
  user: "admin",
  pwd: "VotreMotDePasseSecurise123!",  // CHANGER EN PRODUCTION
  roles: [
    { role: "root", db: "admin" }
  ]
})

// Tester la connexion
exit
mongosh --host mongo-rs-01 --port 27017 -u admin -p --authenticationDatabase admin
```

### Étape 4 : Création des utilisateurs applicatifs

```javascript
// Se connecter en tant qu'admin
use admin
db.auth("admin", "VotreMotDePasseSecurise123!")

// Utilisateur en lecture/écriture pour l'application
use myapp

db.createUser({
  user: "myapp_rw",
  pwd: "AppPassword456!",  // CHANGER EN PRODUCTION
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// Utilisateur en lecture seule pour les analytics
db.createUser({
  user: "myapp_ro",
  pwd: "ReadOnlyPass789!",  // CHANGER EN PRODUCTION
  roles: [
    { role: "read", db: "myapp" }
  ]
})
```

---

## Vérification du déploiement

### Vérifier le statut du Replica Set

```javascript
// Se connecter au PRIMARY
mongosh "mongodb://admin:PASSWORD@mongo-rs-01:27017/admin"

// Statut complet
rs.status()

// Vue simplifiée
rs.isMaster()

// Configuration active
rs.conf()
```

### Résultat attendu de rs.status()

```javascript
{
  set: 'rs-prod',
  members: [
    {
      _id: 0,
      name: 'mongo-rs-01:27017',
      stateStr: 'PRIMARY',
      health: 1,
      // ...
    },
    {
      _id: 1,
      name: 'mongo-rs-02:27017',
      stateStr: 'SECONDARY',
      health: 1,
      syncSourceHost: 'mongo-rs-01:27017'
      // ...
    },
    {
      _id: 2,
      name: 'mongo-rs-03:27017',
      stateStr: 'SECONDARY',
      health: 1,
      syncSourceHost: 'mongo-rs-01:27017'
      // ...
    }
  ],
  ok: 1
}
```

### Tests de lecture/écriture

```javascript
// Sur le PRIMARY
use test

// Écriture
db.testCollection.insertOne({ test: "Hello Replica Set", date: new Date() })

// Lecture sur PRIMARY
db.testCollection.find()

// Se connecter à un SECONDARY
mongosh "mongodb://admin:PASSWORD@mongo-rs-02:27017/admin"

use test

// Activer les lectures sur SECONDARY
db.getMongo().setReadPref("secondary")

// Vérifier la réplication
db.testCollection.find()
```

### Vérifier l'oplog

```javascript
// Sur le PRIMARY
use local

// Taille de l'oplog
db.oplog.rs.stats()

// Premières et dernières entrées
db.oplog.rs.find().sort({$natural: 1}).limit(1).pretty()
db.oplog.rs.find().sort({$natural: -1}).limit(1).pretty()

// Calcul de la fenêtre de réplication (en heures)
var firstOp = db.oplog.rs.find().sort({$natural: 1}).limit(1).next()
var lastOp = db.oplog.rs.find().sort({$natural: -1}).limit(1).next()
var replicationWindow = (lastOp.ts.getTime() - firstOp.ts.getTime()) / 3600
print("Fenêtre de réplication : " + replicationWindow + " heures")
```

---

## Connexion depuis les applications

### Connection String standard

```javascript
// Connexion avec failover automatique
mongodb://myapp_rw:AppPassword456!@mongo-rs-01:27017,mongo-rs-02:27017,mongo-rs-03:27017/myapp?replicaSet=rs-prod&authSource=myapp

// Avec Read Preference
mongodb://myapp_ro:ReadOnlyPass789!@mongo-rs-01:27017,mongo-rs-02:27017,mongo-rs-03:27017/myapp?replicaSet=rs-prod&authSource=myapp&readPreference=secondary

// Avec Write Concern et Read Concern
mongodb://myapp_rw:AppPassword456!@mongo-rs-01:27017,mongo-rs-02:27017,mongo-rs-03:27017/myapp?replicaSet=rs-prod&authSource=myapp&w=majority&readConcernLevel=majority
```

### Exemple Node.js

```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb://myapp_rw:AppPassword456!@mongo-rs-01:27017,mongo-rs-02:27017,mongo-rs-03:27017/myapp?replicaSet=rs-prod&authSource=myapp";

const client = new MongoClient(uri, {
  maxPoolSize: 50,
  minPoolSize: 10,
  serverSelectionTimeoutMS: 5000,
  socketTimeoutMS: 45000,
});

async function run() {
  try {
    await client.connect();
    console.log("Connecté au Replica Set");

    const db = client.db("myapp");
    const result = await db.collection("test").findOne();
    console.log(result);
  } finally {
    await client.close();
  }
}

run().catch(console.dir);
```

### Exemple Python

```python
from pymongo import MongoClient, ReadPreference

# Connexion au Replica Set
client = MongoClient(
    "mongodb://myapp_rw:AppPassword456!@mongo-rs-01:27017,mongo-rs-02:27017,mongo-rs-03:27017/myapp?replicaSet=rs-prod&authSource=myapp",
    maxPoolSize=50,
    minPoolSize=10,
    serverSelectionTimeoutMS=5000
)

# Utiliser la base de données
db = client.myapp

# Écriture sur PRIMARY
db.test.insert_one({"message": "Hello from Python"})

# Lecture depuis SECONDARY
client_read = MongoClient(
    "mongodb://myapp_ro:ReadOnlyPass789!@mongo-rs-01:27017,mongo-rs-02:27017,mongo-rs-03:27017/myapp?replicaSet=rs-prod&authSource=myapp",
    read_preference=ReadPreference.SECONDARY
)

db_read = client_read.myapp
doc = db_read.test.find_one()
print(doc)
```

---

## Maintenance

### Redémarrage d'un membre SECONDARY

```bash
# 1. Se connecter au Replica Set
mongosh "mongodb://admin:PASSWORD@mongo-rs-01:27017/admin"

# 2. Vérifier que le nœud est bien SECONDARY
rs.status()

# 3. Sur le serveur à redémarrer (mongo-rs-02 par exemple)
sudo systemctl restart mongod

# 4. Vérifier le retour du nœud
rs.status()
```

### Redémarrage du PRIMARY (avec failover)

```bash
# 1. Forcer une élection (stepDown du PRIMARY)
mongosh "mongodb://admin:PASSWORD@mongo-rs-01:27017/admin"

rs.stepDown(60)  // Le nœud devient SECONDARY pendant 60 secondes

# 2. Attendre qu'un nouveau PRIMARY soit élu
rs.status()

# 3. Redémarrer l'ancien PRIMARY (maintenant SECONDARY)
sudo systemctl restart mongod

# 4. Le nœud peut redevenir PRIMARY automatiquement (selon priority)
```

### Rolling restart complet

```bash
# Script de rolling restart
#!/bin/bash

NODES=("mongo-rs-02" "mongo-rs-03" "mongo-rs-01")

for node in "${NODES[@]}"; do
  echo "Redémarrage de $node..."

  if [ "$node" == "mongo-rs-01" ]; then
    # Forcer stepDown si c'est le PRIMARY
    mongosh "mongodb://admin:PASSWORD@$node:27017/admin" --eval "rs.stepDown(60)"
    sleep 10
  fi

  ssh $node "sudo systemctl restart mongod"

  echo "Attente de la synchronisation de $node..."
  sleep 30

  # Vérifier que le nœud est de retour
  mongosh "mongodb://admin:PASSWORD@mongo-rs-01:27017/admin" --eval "rs.status()" | grep -A 5 $node

  echo "$node redémarré avec succès"
  echo "---"
done
```

### Ajout d'un nouveau membre (scale to 4 nodes)

```javascript
// Se connecter au PRIMARY
mongosh "mongodb://admin:PASSWORD@mongo-rs-01:27017/admin"

// Ajouter le 4e nœud
rs.add({
  host: "mongo-rs-04:27017",
  priority: 1,
  votes: 1
})

// Vérifier
rs.status()
rs.conf()
```

### Retrait d'un membre

```javascript
// Se connecter au PRIMARY
mongosh "mongodb://admin:PASSWORD@mongo-rs-01:27017/admin"

// Retirer un nœud (par hostname)
rs.remove("mongo-rs-03:27017")

// Ou par ID
rs.remove(2)

// Vérifier
rs.conf()
```

### Changement de priorité

```javascript
// Modifier la priorité d'un membre
var cfg = rs.conf()
cfg.members[1].priority = 3  // Membre index 1 prioritaire
rs.reconfig(cfg)

// Vérifier
rs.conf()
```

---

## Sauvegarde

### Sauvegarde depuis un SECONDARY

```bash
# Se connecter à un SECONDARY (mongo-rs-02 par exemple)
mongosh "mongodb://admin:PASSWORD@mongo-rs-02:27017/admin"

# Vérifier que c'est bien un SECONDARY
rs.status()

# Créer la sauvegarde
mongodump \
  --host mongo-rs-02:27017 \
  --username admin \
  --password "VotreMotDePasseSecurise123!" \
  --authenticationDatabase admin \
  --out /backup/mongodb/$(date +%Y%m%d) \
  --oplog \
  --gzip

# Archive tar
cd /backup/mongodb
tar -czf backup-$(date +%Y%m%d-%H%M%S).tar.gz $(date +%Y%m%d)/
```

### Script de sauvegarde automatisé

```bash
#!/bin/bash
# /usr/local/bin/mongodb-backup.sh

BACKUP_DIR="/backup/mongodb"
RETENTION_DAYS=7
DATE=$(date +%Y%m%d-%H%M%S)

# Créer le répertoire
mkdir -p $BACKUP_DIR/$DATE

# Sauvegarde
mongodump \
  --host mongo-rs-02:27017 \
  --username admin \
  --password "VotreMotDePasseSecurise123!" \
  --authenticationDatabase admin \
  --out $BACKUP_DIR/$DATE \
  --oplog \
  --gzip \
  2>&1 | tee $BACKUP_DIR/$DATE/backup.log

# Archive
cd $BACKUP_DIR
tar -czf backup-$DATE.tar.gz $DATE/
rm -rf $DATE/

# Nettoyage des anciennes sauvegardes
find $BACKUP_DIR -name "backup-*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Sauvegarde terminée : backup-$DATE.tar.gz"
```

### Cron pour sauvegardes quotidiennes

```bash
# crontab -e
# Sauvegarde tous les jours à 2h du matin
0 2 * * * /usr/local/bin/mongodb-backup.sh >> /var/log/mongodb/backup.log 2>&1
```

---

## Monitoring

### Métriques clés à surveiller

```javascript
// Statut de réplication
rs.status()

// Lag de réplication
rs.printSlaveReplicationInfo()

// Oplog info
rs.printReplicationInfo()

// Statistiques serveur
db.serverStatus()

// Connexions actives
db.currentOp()

// Statistiques de base de données
db.stats()
```

### Script de monitoring simple

```bash
#!/bin/bash
# /usr/local/bin/mongodb-health-check.sh

PRIMARY=$(mongosh "mongodb://admin:PASSWORD@mongo-rs-01:27017/admin" --quiet --eval "rs.isMaster().primary")
echo "PRIMARY actuel : $PRIMARY"

# Vérifier tous les membres
for host in mongo-rs-01 mongo-rs-02 mongo-rs-03; do
  echo "---"
  echo "Vérification de $host:"

  STATE=$(mongosh "mongodb://admin:PASSWORD@$host:27017/admin" --quiet --eval "rs.status().myState")

  if [ $? -eq 0 ]; then
    echo "  État : $STATE"

    CONNECTIONS=$(mongosh "mongodb://admin:PASSWORD@$host:27017/admin" --quiet --eval "db.serverStatus().connections.current")
    echo "  Connexions : $CONNECTIONS"

    REPL_LAG=$(mongosh "mongodb://admin:PASSWORD@$host:27017/admin" --quiet --eval "rs.printSlaveReplicationInfo()" | grep "syncedTo" | awk '{print $4}')
    echo "  Replication lag : $REPL_LAG"
  else
    echo "  ERREUR : Impossible de se connecter"
  fi
done
```

---

## Troubleshooting

### Problème : Membre bloqué en STARTUP ou RECOVERING

```javascript
// Vérifier les logs
// Sur le serveur problématique
tail -f /var/log/mongodb/mongod.log

// Solutions possibles :

// 1. Vérifier la connectivité réseau
ping mongo-rs-01
telnet mongo-rs-01 27017

// 2. Vérifier l'oplog
use local
db.oplog.rs.stats()

// 3. Resync complet si nécessaire
rs.syncFrom("mongo-rs-01:27017")

// 4. En dernier recours : resynchronisation initiale
// ATTENTION : Efface toutes les données du nœud
mongosh "mongodb://admin:PASSWORD@mongo-rs-02:27017/admin"
db.shutdownServer()

// Sur le serveur
sudo rm -rf /data/mongodb/*
sudo systemctl start mongod

// Le nœud se resynchronisera automatiquement
```

### Problème : Lag de réplication élevé

```javascript
// Identifier le lag
rs.printSlaveReplicationInfo()

// Solutions :

// 1. Vérifier les performances réseau
// Sur le serveur
iftop -i eth0

// 2. Vérifier les write operations
db.serverStatus().opcounters

// 3. Vérifier l'I/O disk
iostat -x 1

// 4. Augmenter l'oplog si nécessaire
// ATTENTION : Opération impactante, à faire en maintenance
db.adminCommand({replSetResizeOplog: 1, size: 20480}) // 20 GB
```

### Problème : Elections fréquentes

```javascript
// Vérifier l'historique des élections
use local
db.system.replset.find()

// Causes possibles :
// - Problèmes réseau intermittents
// - Charge CPU/mémoire excessive
// - Latence réseau élevée

// Solution : Ajuster les timeouts
cfg = rs.conf()
cfg.settings = {
  electionTimeoutMillis: 10000,  // Défaut : 10000ms
  heartbeatTimeoutSecs: 10        // Défaut : 10s
}
rs.reconfig(cfg)
```

### Problème : Impossibilité d'élire un PRIMARY

```javascript
// Vérifier la majorité
// Il faut au moins 2 nœuds sur 3 disponibles

// Si un seul nœud disponible, forcer la reconfiguration
cfg = rs.conf()
cfg.members = [cfg.members[0]]  // Garder seulement le nœud actuel
rs.reconfig(cfg, {force: true})

// ATTENTION : À faire uniquement en cas d'urgence
// Reconfigurer correctement dès que les autres nœuds sont disponibles
```

---

## Checklist de mise en production

- [ ] Tous les nœuds démarrés et joignables
- [ ] Replica Set initialisé et état = OK
- [ ] PRIMARY élu et stable
- [ ] Keyfile configuré sur tous les nœuds
- [ ] Utilisateur admin créé
- [ ] Utilisateurs applicatifs créés
- [ ] Firewall configuré
- [ ] THP désactivé sur tous les nœuds
- [ ] Paramètres kernel optimisés
- [ ] Monitoring en place
- [ ] Sauvegarde automatisée configurée
- [ ] Test de failover réussi
- [ ] Connection strings testées depuis l'application
- [ ] Documentation architecture à jour
- [ ] Contacts d'escalade définis

---

## Ressources complémentaires

### Documentation officielle
- [Deploy a Replica Set](https://docs.mongodb.com/manual/tutorial/deploy-replica-set/)
- [Replica Set Configuration](https://docs.mongodb.com/manual/reference/replica-configuration/)
- [Replica Set Maintenance](https://docs.mongodb.com/manual/administration/replica-set-maintenance/)

### Commandes utiles

```javascript
// Cheat sheet Replica Set
rs.help()              // Aide sur les commandes
rs.status()            // Statut complet
rs.isMaster()          // Qui est le PRIMARY
rs.conf()              // Configuration
rs.printReplicationInfo()       // Info oplog
rs.printSlaveReplicationInfo()  // Lag de réplication
```

---


⏭️ [Configuration Sharded Cluster](/annexes/configuration-reference/02-configuration-sharded-cluster.md)
