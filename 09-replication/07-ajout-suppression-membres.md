🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.7 Ajout et Suppression de Membres

## Introduction

L'ajout et la suppression de membres dans un Replica Set sont des opérations courantes en production, nécessaires pour le scaling, la maintenance ou la reconfiguration de l'architecture. Bien que ces opérations soient supportées en ligne (sans arrêt de service), elles nécessitent une planification minutieuse pour minimiser l'impact sur les performances et garantir la cohérence des données.

## Concepts Fondamentaux

### États de Synchronisation

Lorsqu'un nouveau membre rejoint un Replica Set, il traverse plusieurs états :

```
STARTUP → STARTUP2 → RECOVERING → SECONDARY → PRIMARY (si éligible)
```

| État | Description | Durée Typique |
|------|-------------|---------------|
| `STARTUP` | Initialisation, chargement de la configuration | Quelques secondes |
| `STARTUP2` | Chargement des métadonnées du Replica Set | Quelques secondes |
| `RECOVERING` | Synchronisation initiale des données | Minutes à heures |
| `SECONDARY` | Membre opérationnel en réplication | État stable |
| `PRIMARY` | Membre acceptant les écritures | État stable (un seul) |

### Méthodes de Synchronisation Initiale

#### 1. Initial Sync (Synchronisation Initiale Complète)

Le nouveau membre copie toutes les données depuis un membre existant :

```
Source Member (SECONDARY ou PRIMARY)
    ↓ (copie complète de toutes les collections)
New Member (RECOVERING)
    ↓ (application de l'oplog pendant la copie)
New Member (SECONDARY)
```

**Phases** :
1. **Clone** : Copie de toutes les collections
2. **Oplog Apply** : Application des opérations survenues pendant la copie
3. **Index Build** : Construction des index
4. **Catch-up** : Rattrapage final de l'oplog

#### 2. Logical Initial Sync

Utilisée par défaut depuis MongoDB 4.4 :

```javascript
// Configuration automatique
{
  "initialSyncMethod": "logical",  // Par défaut
  "initialSyncSourceReadPreference": "nearest"
}
```

**Avantages** :
- Plus efficace pour les grandes bases de données
- Meilleure gestion des échecs
- Reprise après interruption

#### 3. File Copy (Restauration depuis Backup)

Pour très grandes bases de données :

```bash
# 1. Arrêter le nouveau membre
systemctl stop mongod

# 2. Copier les données depuis un backup ou un membre existant
rsync -avz --progress mongodb-source:/data/mongodb/ /data/mongodb/

# 3. Démarrer et ajouter au Replica Set
systemctl start mongod
```

## Ajout de Membres

### Préparation

#### 1. Vérifications Préalables

```javascript
// Sur le Primary, vérifier :

// a) Nombre de membres actuels
rs.conf().members.length
// Maximum : 50 membres, 7 votants

// b) État de santé du Replica Set
rs.status().ok
// Doit retourner : 1

// c) Ressources disponibles
db.serverStatus().connections
db.serverStatus().mem

// d) Oplog window
rs.printReplicationInfo()
// Doit être > durée estimée de l'initial sync
```

#### 2. Préparation du Nouveau Serveur

```bash
# Installation de MongoDB
# (voir section 1.8 - Installation de MongoDB)

# Configuration mongod.conf
cat > /etc/mongod.conf <<EOF
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true

net:
  port: 27017
  bindIp: 0.0.0.0  # Sécuriser en production

replication:
  replSetName: rs0  # DOIT correspondre au Replica Set existant

# Si authentification activée
security:
  keyFile: /etc/mongodb/keyfile
  authorization: enabled
EOF

# Copier le keyfile depuis un membre existant
scp mongodb-01:/etc/mongodb/keyfile /etc/mongodb/keyfile
chmod 400 /etc/mongodb/keyfile
chown mongodb:mongodb /etc/mongodb/keyfile

# Créer les répertoires
mkdir -p /data/mongodb
chown -R mongodb:mongodb /data/mongodb

# Démarrer MongoDB
systemctl start mongod
systemctl enable mongod
```

#### 3. Vérifier la Connectivité

```bash
# Depuis le Primary
ping mongodb-new
telnet mongodb-new 27017

# Depuis le nouveau membre
mongosh --host mongodb-primary:27017 --eval "db.adminCommand('ping')"
```

### Méthode 1 : rs.add() - Ajout Simple

#### Ajout Basique

```javascript
// Se connecter au Primary
mongosh --host mongodb-primary:27017 -u admin -p password

// Ajouter le membre
rs.add("mongodb-04:27017")
```

**Réponse** :
```javascript
{
  "ok": 1,
  "$clusterTime": {
    "clusterTime": Timestamp(1705320000, 1),
    "signature": { ... }
  },
  "operationTime": Timestamp(1705320000, 1)
}
```

#### Ajout avec Options

```javascript
rs.add({
  host: "mongodb-04:27017",
  priority: 1,
  votes: 1,
  tags: {
    dc: "east",
    rack: "A3",
    nodeType: "data"
  }
})
```

#### Ajout d'un Arbiter

```javascript
rs.addArb("arbiter:27017")

// Équivalent à :
rs.add({
  host: "arbiter:27017",
  arbiterOnly: true,
  votes: 1
})
```

### Méthode 2 : rs.reconfig() - Ajout avec Configuration Complète

```javascript
// 1. Récupérer la configuration actuelle
cfg = rs.conf()

// 2. Trouver le prochain _id disponible
var nextId = Math.max(...cfg.members.map(m => m._id)) + 1

// 3. Ajouter le nouveau membre
cfg.members.push({
  _id: nextId,
  host: "mongodb-04:27017",
  priority: 1,
  votes: 1,
  tags: {
    dc: "east",
    rack: "A3"
  }
})

// 4. Appliquer la configuration
rs.reconfig(cfg)
```

### Ajout de Membres Spéciaux

#### Hidden Member (Analytics/Reporting)

```javascript
rs.add({
  host: "mongodb-analytics:27017",
  priority: 0,      // Ne peut pas devenir Primary
  hidden: true,     // Invisible pour les applications
  votes: 1,         // Participe aux élections
  tags: {
    usage: "analytics",
    workload: "reporting"
  }
})
```

#### Delayed Member (Backup)

```javascript
rs.add({
  host: "mongodb-delayed:27017",
  priority: 0,
  hidden: true,
  slaveDelay: 3600,  // 1 heure de délai
  votes: 1,
  tags: {
    backup: "delayed"
  }
})
```

#### Membre Non-Votant (> 7 membres)

```javascript
// Pour dépasser 7 membres votants
rs.add({
  host: "mongodb-08:27017",
  priority: 0,
  votes: 0,  // Non-votant
  tags: {
    dc: "west",
    nodeType: "readonly"
  }
})
```

### Surveillance de la Synchronisation Initiale

#### Méthode 1 : rs.status()

```javascript
// Observer l'état du nouveau membre
rs.status().members.forEach(m => {
  if (m.name === "mongodb-04:27017") {
    print("State: " + m.stateStr)
    print("Sync source: " + m.syncSourceHost)
    if (m.syncingTo) {
      print("Syncing to: " + m.syncingTo)
    }
  }
})
```

**États observés** :
```
State: STARTUP2
State: RECOVERING
  Syncing to: mongodb-01:27017
State: SECONDARY
```

#### Méthode 2 : Logs du Nouveau Membre

```bash
# Sur le nouveau membre
tail -f /var/log/mongodb/mongod.log | grep -E "initial sync|InitialSyncer"
```

**Exemples de logs** :
```
[InitialSyncer] Starting initial sync
[InitialSyncer] Choosing sync source: mongodb-01:27017
[InitialSyncer] Cloning database: mydb
[InitialSyncer] Collection clone progress: mydb.users 45%
[InitialSyncer] Building indexes for database: mydb
[InitialSyncer] Initial sync completed
```

#### Méthode 3 : Monitoring Détaillé

```javascript
// Script de monitoring
function monitorInitialSync(memberHost) {
  var intervalId = setInterval(function() {
    var status = rs.status()
    var member = status.members.find(m => m.name === memberHost)

    if (!member) {
      print("Member not found")
      clearInterval(intervalId)
      return
    }

    print("\n=== " + new Date().toISOString() + " ===")
    print("State: " + member.stateStr)
    print("Health: " + member.health)

    if (member.syncSourceHost) {
      print("Sync source: " + member.syncSourceHost)
    }

    if (member.optimeDate) {
      var lag = (status.date - member.optimeDate) / 1000
      print("Replication lag: " + lag + " seconds")
    }

    // Arrêter si SECONDARY
    if (member.state === 2) {
      print("\n✓ Initial sync completed!")
      clearInterval(intervalId)
    }
  }, 5000)  // Vérifier toutes les 5 secondes
}

// Utilisation
monitorInitialSync("mongodb-04:27017")
```

### Estimation de la Durée

#### Facteurs Influençant la Durée

```javascript
// Calcul approximatif
function estimateInitialSyncDuration() {
  var stats = db.stats()
  var dataSizeGB = stats.dataSize / 1024 / 1024 / 1024
  var indexSizeGB = stats.indexSize / 1024 / 1024 / 1024
  var totalGB = dataSizeGB + indexSizeGB

  // Hypothèses :
  var networkSpeedGBps = 0.1  // 100 MB/s = 0.1 GB/s
  var indexBuildFactor = 1.5  // Index build ajoute 50%

  var copyTimeHours = (totalGB / networkSpeedGBps / 3600)
  var totalTimeHours = copyTimeHours * indexBuildFactor

  print("Data size: " + dataSizeGB.toFixed(2) + " GB")
  print("Index size: " + indexSizeGB.toFixed(2) + " GB")
  print("Total: " + totalGB.toFixed(2) + " GB")
  print("Estimated time: " + totalTimeHours.toFixed(2) + " hours")

  return totalTimeHours
}

estimateInitialSyncDuration()
```

**Formule générale** :
```
Durée (heures) = (Taille données + Taille index) / Vitesse réseau × Facteur overhead
```

Où :
- Facteur overhead = 1.3 - 2.0 (construction d'index, oplog apply)

### Optimisation de la Synchronisation

#### 1. Choix de la Source

```javascript
// Configurer la préférence de source
cfg = rs.conf()

cfg.settings = cfg.settings || {}
cfg.settings.chainingAllowed = true  // Permet la réplication chaînée

rs.reconfig(cfg)

// Le nouveau membre choisira automatiquement
// la source la plus proche/la moins chargée
```

#### 2. Augmenter la Fenêtre Oplog

```javascript
// Avant l'ajout, sur tous les membres existants
db.adminCommand({
  replSetResizeOplog: 1,
  size: 20480  // 20 GB (temporaire)
})

// Après l'initial sync, réduire si nécessaire
```

#### 3. Utiliser un Backup Récent

Pour très grandes bases (> 1 TB) :

```bash
# Méthode 1 : mongorestore depuis backup
mongorestore --host mongodb-04:27017 /backup/latest/

# Méthode 2 : Copie physique
# (MongoDB arrêté sur la destination)
rsync -avz mongodb-01:/data/mongodb/ /data/mongodb/

# Puis démarrer et ajouter au Replica Set
systemctl start mongod
```

#### 4. Désactiver le Balancer (si Sharded Cluster)

```javascript
// Sur le mongos
sh.stopBalancer()

// Ajouter le membre...

// Réactiver
sh.startBalancer()
```

## Suppression de Membres

### Préparation

#### Vérifications Avant Suppression

```javascript
// 1. Identifier le membre à supprimer
rs.status().members.forEach(m => {
  print(m.name + " - " + m.stateStr + " - Priority: " +
        rs.conf().members.find(cm => cm.host === m.name).priority)
})

// 2. Vérifier qu'il ne s'agit PAS du Primary
var primary = rs.status().members.find(m => m.state === 1)
print("Current Primary: " + primary.name)

// 3. Vérifier que la majorité sera maintenue
var votingMembers = rs.conf().members.filter(m => m.votes === 1).length
print("Voting members: " + votingMembers)
print("After removal: " + (votingMembers - 1))
print("Majority needed: " + Math.floor(votingMembers / 2) + 1)

// 4. Vérifier les read preference
// Si des applications lisent spécifiquement depuis ce membre
```

### Méthode 1 : rs.remove() - Suppression Simple

```javascript
// Se connecter au Primary
mongosh --host mongodb-primary:27017

// Supprimer le membre
rs.remove("mongodb-04:27017")
```

**Réponse** :
```javascript
{
  "ok": 1,
  "$clusterTime": { ... },
  "operationTime": Timestamp(...)
}
```

### Méthode 2 : rs.reconfig() - Suppression avec Configuration

```javascript
// 1. Récupérer la configuration
cfg = rs.conf()

// 2. Filtrer le membre à supprimer
cfg.members = cfg.members.filter(m => m.host !== "mongodb-04:27017")

// 3. Appliquer
rs.reconfig(cfg)
```

### Suppression du Primary

**Important** : Ne JAMAIS supprimer directement le Primary.

#### Procédure Sécurisée

```javascript
// 1. Identifier le Primary
var primary = rs.status().members.find(m => m.state === 1)
print("Current Primary: " + primary.name)

// 2. Forcer un stepDown (démission)
rs.stepDown(60)  // Le Primary devient Secondary pour 60 secondes minimum

// 3. Attendre la nouvelle élection
sleep(10000)

// 4. Vérifier le nouveau Primary
rs.isMaster().primary

// 5. Maintenant, supprimer l'ancien Primary (devenu Secondary)
rs.remove(primary.name)
```

### Suppression Multiple de Membres

```javascript
// Supprimer plusieurs membres en une seule reconfiguration
cfg = rs.conf()

var membersToRemove = ["mongodb-04:27017", "mongodb-05:27017"]

cfg.members = cfg.members.filter(m => !membersToRemove.includes(m.host))

rs.reconfig(cfg)
```

**Attention** : S'assurer de maintenir la majorité !

### Nettoyage Post-Suppression

#### Sur le Membre Supprimé

```bash
# 1. Arrêter MongoDB
systemctl stop mongod

# 2. Optionnel : Sauvegarder les données
tar -czf /backup/mongodb-04-$(date +%Y%m%d).tar.gz /data/mongodb/

# 3. Optionnel : Supprimer les données
rm -rf /data/mongodb/*

# 4. Désactiver le service
systemctl disable mongod
```

#### Nettoyage de la Configuration

```bash
# Retirer de /etc/hosts
sed -i '/mongodb-04/d' /etc/hosts

# Retirer du monitoring
# (dépend de votre système de monitoring)
```

## Remplacement de Membres

### Scénario 1 : Remplacement avec Même Hostname

```javascript
// 1. Supprimer le membre défaillant
rs.remove("mongodb-02:27017")

// 2. Sur le nouveau serveur (avec le même hostname/IP)
// - Installer MongoDB
// - Vider /data/mongodb si nécessaire
// - Copier le keyfile
// - Configurer mongod.conf avec le même replSetName

// 3. Démarrer le nouveau serveur
systemctl start mongod

// 4. Rajouter au Replica Set
rs.add({
  host: "mongodb-02:27017",
  priority: 1,
  votes: 1,
  tags: { /* mêmes tags */ }
})
```

### Scénario 2 : Remplacement avec Nouveau Hostname

```javascript
// 1. Ajouter le nouveau membre
rs.add({
  host: "mongodb-new:27017",
  priority: 1,
  votes: 1
})

// 2. Attendre la synchronisation complète
// (voir "Surveillance de la Synchronisation Initiale")

// 3. Supprimer l'ancien membre
rs.remove("mongodb-old:27017")

// 4. Optionnel : Ajuster les priorités
cfg = rs.conf()
var member = cfg.members.find(m => m.host === "mongodb-new:27017")
member.priority = 2  // Même priorité que l'ancien
rs.reconfig(cfg)
```

### Remplacement d'un Primary

```javascript
// 1. Ajouter le nouveau membre (il sera Secondary)
rs.add("mongodb-new:27017")

// 2. Attendre la synchronisation
// ...

// 3. Augmenter la priorité du nouveau membre
cfg = rs.conf()
cfg.members.find(m => m.host === "mongodb-new:27017").priority = 10
rs.reconfig(cfg)

// 4. Le nouveau membre déclenchera une élection et deviendra Primary

// 5. Supprimer l'ancien Primary (maintenant Secondary)
rs.remove("mongodb-old:27017")
```

## Migration de Replica Set

### Migration Complète (Rolling Migration)

#### Stratégie : Ajout Progressif

```javascript
// État initial : rs0 avec mongodb-01, mongodb-02, mongodb-03

// 1. Ajouter les nouveaux membres
rs.add("mongodb-new-01:27017")
rs.add("mongodb-new-02:27017")
rs.add("mongodb-new-03:27017")

// Attendre synchronisation...

// 2. Augmenter les priorités des nouveaux membres
cfg = rs.conf()
cfg.members.find(m => m.host === "mongodb-new-01:27017").priority = 10
cfg.members.find(m => m.host === "mongodb-new-02:27017").priority = 9
cfg.members.find(m => m.host === "mongodb-new-03:27017").priority = 8

// Réduire les priorités des anciens
cfg.members.find(m => m.host === "mongodb-01:27017").priority = 1
cfg.members.find(m => m.host === "mongodb-02:27017").priority = 1
cfg.members.find(m => m.host === "mongodb-03:27017").priority = 1

rs.reconfig(cfg)

// 3. Le nouveau membre avec priority 10 deviendra Primary

// 4. Supprimer les anciens membres un par un
rs.remove("mongodb-01:27017")
sleep(30000)
rs.remove("mongodb-02:27017")
sleep(30000)
rs.remove("mongodb-03:27017")
```

### Migration avec Changement de Réseau

#### Utilisation des Horizons

```javascript
// Configuration avec horizons pour transition
cfg = rs.conf()

cfg.members.forEach(m => {
  m.horizons = {
    old: m.host,                           // Ancien réseau
    new: m.host.replace("old.net", "new.net")  // Nouveau réseau
  }
})

rs.reconfig(cfg)

// Les clients peuvent maintenant utiliser les deux réseaux
// Connection string :
// mongodb://mongodb-01.old.net:27017/?replicaSet=rs0
// ou
// mongodb://mongodb-01.new.net:27017/?replicaSet=rs0
```

### Migration Datacenter

```javascript
// Situation : Migration DC1 → DC2

// Phase 1 : Ajouter membres dans DC2
rs.add({
  host: "dc2-mongodb-01:27017",
  priority: 0,  // Initialement passif
  votes: 1,
  tags: { dc: "dc2" }
})

rs.add({
  host: "dc2-mongodb-02:27017",
  priority: 0,
  votes: 1,
  tags: { dc: "dc2" }
})

// Attendre synchronisation complète (peut prendre des heures)...

// Phase 2 : Basculer les priorités
cfg = rs.conf()

// DC2 devient prioritaire
cfg.members.find(m => m.host === "dc2-mongodb-01:27017").priority = 10
cfg.members.find(m => m.host === "dc2-mongodb-02:27017").priority = 9

// DC1 devient secondaire
cfg.members.find(m => m.host === "dc1-mongodb-01:27017").priority = 1
cfg.members.find(m => m.host === "dc1-mongodb-02:27017").priority = 1

rs.reconfig(cfg)

// Phase 3 : Vérifier que le Primary est dans DC2
rs.isMaster().primary  // Devrait être dc2-mongodb-01

// Phase 4 : Supprimer les membres DC1
rs.remove("dc1-mongodb-01:27017")
rs.remove("dc1-mongodb-02:27017")
```

## Impact sur les Performances

### Pendant l'Ajout

#### Impact sur le Replica Set

```
┌────────────────────────────────────────┐
│         Impact Performance             │
├────────────────────────────────────────┤
│ CPU Source     : +10-30%               │
│ Réseau Source  : +50-100 MB/s          │
│ I/O Source     : +20-40% lectures      │
│ Replication Lag: +10-60 secondes       │
└────────────────────────────────────────┘
```

**Phases critiques** :
1. **Clone initial** : Forte charge I/O sur la source
2. **Index build** : Forte charge CPU sur le nouveau membre
3. **Catch-up** : Forte charge réseau

#### Monitoring Durant l'Ajout

```javascript
// Script de monitoring d'impact
function monitorAddMemberImpact(syncSource) {
  var stats = db.getSiblingDB("admin").serverStatus()

  print("=== " + new Date().toISOString() + " ===")
  print("Connections: " + stats.connections.current)
  print("Network in: " + (stats.network.bytesIn / 1024 / 1024).toFixed(2) + " MB")
  print("Network out: " + (stats.network.bytesOut / 1024 / 1024).toFixed(2) + " MB")
  print("Ops/sec: " + stats.opcounters.query)

  // Replication lag
  var replStatus = rs.status()
  var source = replStatus.members.find(m => m.name === syncSource)
  if (source && source.optimeDate) {
    var lag = (replStatus.date - source.optimeDate) / 1000
    print("Replication lag: " + lag + " sec")
  }

  // Oplog window
  var replInfo = db.getSiblingDB("local").oplog.rs.stats()
  print("Oplog size: " + (replInfo.size / 1024 / 1024 / 1024).toFixed(2) + " GB")
}

// Exécuter régulièrement
var monitorInterval = setInterval(function() {
  monitorAddMemberImpact("mongodb-01:27017")
}, 30000)  // Toutes les 30 secondes

// Arrêter avec : clearInterval(monitorInterval)
```

### Mitigation de l'Impact

#### 1. Limiter le Trafic de Synchronisation

```javascript
// Sur le nouveau membre, limiter la bande passante
// (nécessite configuration au niveau OS avec tc ou équivalent)

// Linux : Utiliser tc (traffic control)
# Limiter à 50 MB/s
tc qdisc add dev eth0 root tbf rate 50mbit burst 32kbit latency 400ms
```

#### 2. Planification Hors-Heures

```javascript
// Script de planification
function scheduleAddMember(memberHost, scheduledTime) {
  var now = new Date()
  var delay = scheduledTime - now

  if (delay <= 0) {
    print("Scheduled time is in the past!")
    return
  }

  print("Member will be added in " + (delay / 1000 / 60) + " minutes")

  setTimeout(function() {
    print("Adding member: " + memberHost)
    rs.add(memberHost)
  }, delay)
}

// Ajouter à 2h du matin
var scheduledTime = new Date()
scheduledTime.setHours(2, 0, 0, 0)
if (scheduledTime < new Date()) {
  scheduledTime.setDate(scheduledTime.getDate() + 1)
}

scheduleAddMember("mongodb-04:27017", scheduledTime)
```

#### 3. Utiliser un Backup pour l'Initial Sync

```bash
# Éviter l'initial sync complet via réseau

# 1. Créer un backup récent
mongodump --host mongodb-01:27017 --out /backup/$(date +%Y%m%d)

# 2. Restaurer sur le nouveau membre
mongorestore --host mongodb-04:27017 /backup/20240115

# 3. Ajouter au Replica Set
# L'initial sync sera minimal (seulement le delta)
```

## Gestion des Votes

### Modifier le Nombre de Votants

#### Augmenter le Nombre de Votants

```javascript
// Maximum 7 votants
cfg = rs.conf()

// Donner le droit de vote à un membre non-votant
var member = cfg.members.find(m => m.host === "mongodb-08:27017")
if (member.votes === 0) {
  member.votes = 1
  rs.reconfig(cfg)
}
```

#### Retirer le Droit de Vote

```javascript
cfg = rs.conf()

// Retirer le vote (par exemple, membre distant)
var member = cfg.members.find(m => m.host === "mongodb-remote:27017")
member.votes = 0
member.priority = 0  // Recommandé avec votes: 0

rs.reconfig(cfg)
```

### Stratégie de Votes par Datacenter

```javascript
// Configuration multi-DC avec votes asymétriques
cfg = rs.conf()

cfg.members = [
  // DC Principal : 3 votants
  { _id: 0, host: "dc1-01:27017", votes: 1, priority: 10, tags: {dc: "dc1"} },
  { _id: 1, host: "dc1-02:27017", votes: 1, priority: 9, tags: {dc: "dc1"} },
  { _id: 2, host: "dc1-03:27017", votes: 1, priority: 8, tags: {dc: "dc1"} },

  // DC Secondaire : 2 membres (1 votant, 1 non-votant)
  { _id: 3, host: "dc2-01:27017", votes: 1, priority: 1, tags: {dc: "dc2"} },
  { _id: 4, host: "dc2-02:27017", votes: 0, priority: 0, tags: {dc: "dc2"} }
]

rs.reconfig(cfg)
```

**Avantages** :
- Majorité dans DC1 : 3/4 votes
- DC1 peut fonctionner seul si DC2 tombe
- DC2 ne peut pas élire un Primary seul

## Cas Particuliers

### Ajout pendant une Opération de Maintenance

```javascript
// Scénario : Un membre est en maintenance

// 1. Vérifier l'état
rs.status().members.forEach(m => {
  print(m.name + " : " + m.stateStr)
})
// Output :
// mongodb-01:27017 : PRIMARY
// mongodb-02:27017 : SECONDARY
// mongodb-03:27017 : DOWN (maintenance)

// 2. On peut ajouter, mais vérifier la majorité
// Membres actuels votants : 3
// Membres UP votants : 2
// Après ajout : 4 votants, besoin de 3 pour majorité

// 3. Ajouter le nouveau membre
rs.add("mongodb-04:27017")

// 4. Nouvelle majorité : 3 (sur 4)
// Le Replica Set reste opérationnel
```

### Récupération après Perte de Majorité

```javascript
// Scénario : 3 membres, 2 tombent
// mongodb-01 : UP (seul survivant)
// mongodb-02 : DOWN
// mongodb-03 : DOWN

// IMPOSSIBLE d'ajouter normalement (pas de majorité)

// Solution : Reconfiguration forcée (DANGER)

// 1. Se connecter au seul membre UP
mongosh --host mongodb-01:27017

// 2. Reconfigurer avec force
cfg = rs.conf()
cfg.members = [
  { _id: 0, host: "mongodb-01:27017" },
  { _id: 3, host: "mongodb-new-01:27017" },
  { _id: 4, host: "mongodb-new-02:27017" }
]

rs.reconfig(cfg, {force: true})

// ATTENTION : Risque de perte de données si les membres DOWN
// avaient des écritures non répliquées
```

### Ajout d'un Membre après Split-Brain

**Situation** : Partition réseau ayant causé deux Primary

```javascript
// Phase 1 : Identifier le "vrai" Primary
// (celui avec le term le plus élevé et l'optime le plus récent)

rs.status().members.forEach(m => {
  if (m.state === 1) {
    print("PRIMARY: " + m.name)
    print("Term: " + db.serverStatus().repl.term)
    print("Optime: " + m.optimeDate)
  }
})

// Phase 2 : Se connecter au vrai Primary
mongosh --host <true-primary>:27017

// Phase 3 : Forcer la reconfiguration
cfg = rs.conf()
// Retirer les membres problématiques si nécessaire
cfg.members = cfg.members.filter(/* ... */)
rs.reconfig(cfg, {force: true})

// Phase 4 : Ajouter les nouveaux membres
rs.add("mongodb-new:27017")
```

## Automatisation

### Script d'Ajout Automatisé

```javascript
// Script complet d'ajout avec vérifications
function addMemberSafely(memberConfig) {
  // 1. Vérifications préalables
  var status = rs.status()

  if (!status.ok) {
    return {error: "Replica Set unhealthy"}
  }

  var cfg = rs.conf()
  var votingMembers = cfg.members.filter(m => m.votes === 1).length

  if (votingMembers >= 7 && memberConfig.votes === 1) {
    return {error: "Already have 7 voting members"}
  }

  if (cfg.members.length >= 50) {
    return {error: "Maximum 50 members reached"}
  }

  // 2. Vérifier la connectivité
  try {
    var conn = new Mongo(memberConfig.host)
    var pingResult = conn.getDB("admin").runCommand({ping: 1})
    if (!pingResult.ok) {
      return {error: "Cannot connect to " + memberConfig.host}
    }
  } catch (e) {
    return {error: "Connection failed: " + e}
  }

  // 3. Vérifier l'oplog window
  var replInfo = db.getSiblingDB("local").oplog.rs.stats()
  var oplogHours = (replInfo.maxSize / replInfo.size) * 24  // Estimation

  if (oplogHours < 12) {
    print("WARNING: Oplog window < 12 hours. Consider resizing.")
    print("Current window: ~" + oplogHours.toFixed(1) + " hours")
  }

  // 4. Ajouter le membre
  try {
    var result = rs.add(memberConfig)
    return {
      success: true,
      message: "Member added successfully",
      result: result
    }
  } catch (e) {
    return {error: "Add failed: " + e}
  }
}

// Utilisation
var newMember = {
  host: "mongodb-04:27017",
  priority: 1,
  votes: 1,
  tags: {dc: "east", rack: "A4"}
}

var result = addMemberSafely(newMember)
printjson(result)
```

### Script de Suppression Sécurisé

```javascript
function removeMemberSafely(memberHost) {
  var status = rs.status()
  var cfg = rs.conf()

  // 1. Vérifier que ce n'est pas le Primary
  var primary = status.members.find(m => m.state === 1)
  if (primary.name === memberHost) {
    return {error: "Cannot remove Primary. Step down first."}
  }

  // 2. Vérifier la majorité
  var votingMembers = cfg.members.filter(m => m.votes === 1).length
  var memberToRemove = cfg.members.find(m => m.host === memberHost)

  if (memberToRemove.votes === 1) {
    var afterRemovalVoting = votingMembers - 1
    var majorityNeeded = Math.floor(afterRemovalVoting / 2) + 1

    if (majorityNeeded > afterRemovalVoting) {
      return {error: "Removing this member would break majority"}
    }
  }

  // 3. Supprimer
  try {
    var result = rs.remove(memberHost)
    return {
      success: true,
      message: "Member removed successfully",
      result: result
    }
  } catch (e) {
    return {error: "Removal failed: " + e}
  }
}

// Utilisation
var result = removeMemberSafely("mongodb-04:27017")
printjson(result)
```

## Bonnes Pratiques

### 1. Planification

✅ **Toujours** :
- Vérifier l'oplog window avant ajout
- Planifier les ajouts en heures creuses
- Documenter les changements
- Avoir un plan de rollback

❌ **Jamais** :
- Ajouter/supprimer pendant les pics de charge
- Supprimer le Primary sans stepDown
- Modifier plusieurs membres simultanément (sauf reconfiguration planifiée)

### 2. Synchronisation

```javascript
// Attendre la synchronisation complète avant autre opération
function waitForMemberSync(memberHost, maxWaitSeconds) {
  var waited = 0
  var interval = 5

  while (waited < maxWaitSeconds) {
    var status = rs.status()
    var member = status.members.find(m => m.name === memberHost)

    if (member.state === 2) {  // SECONDARY
      var lag = (status.date - member.optimeDate) / 1000
      if (lag < 10) {  // Lag < 10 secondes
        print("Member synchronized (lag: " + lag + "s)")
        return true
      }
    }

    print("Waiting... (state: " + member.stateStr + ")")
    sleep(interval * 1000)
    waited += interval
  }

  return false
}

// Utilisation
if (waitForMemberSync("mongodb-04:27017", 3600)) {
  print("Ready for next operation")
} else {
  print("Timeout - member not synchronized")
}
```

### 3. Validation Post-Opération

```javascript
function validateReplicaSet() {
  var status = rs.status()
  var cfg = rs.conf()
  var issues = []

  // Check 1 : Un seul Primary
  var primaries = status.members.filter(m => m.state === 1)
  if (primaries.length !== 1) {
    issues.push("ERROR: " + primaries.length + " PRIMARY members")
  }

  // Check 2 : Tous les membres sont UP ou SECONDARY
  status.members.forEach(m => {
    if (![1, 2, 7].includes(m.state)) {  // PRIMARY, SECONDARY, ARBITER
      issues.push("WARN: " + m.name + " is " + m.stateStr)
    }
  })

  // Check 3 : Replication lag < 60s
  status.members.forEach(m => {
    if (m.state === 2 && m.optimeDate) {
      var lag = (status.date - m.optimeDate) / 1000
      if (lag > 60) {
        issues.push("WARN: " + m.name + " lag: " + lag + "s")
      }
    }
  })

  // Check 4 : Nombre impair de votants
  var votingMembers = cfg.members.filter(m => m.votes === 1).length
  if (votingMembers % 2 === 0) {
    issues.push("WARN: Even number of voting members (" + votingMembers + ")")
  }

  return issues.length === 0 ? "✓ All checks passed" : issues
}

// Après toute opération
printjson(validateReplicaSet())
```

### 4. Documentation

```javascript
// Template de documentation pour changements
var changeLog = {
  date: new Date(),
  operator: "admin@company.com",
  operation: "add_member",
  details: {
    member: "mongodb-04:27017",
    reason: "Scaling for increased load",
    config: {
      priority: 1,
      votes: 1,
      tags: {dc: "east", rack: "A4"}
    }
  },
  duration: {
    start: new Date("2024-01-15T02:00:00Z"),
    end: new Date("2024-01-15T03:15:00Z"),
    initialSyncDuration: "75 minutes"
  },
  validation: "All checks passed"
}

// Sauvegarder dans une collection d'audit
db.getSiblingDB("admin").replicaset_changes.insertOne(changeLog)
```

## Dépannage

### Problème : Membre Bloqué en STARTUP ou STARTUP2

**Symptômes** :
```javascript
rs.status().members.find(m => m.name === "mongodb-04:27017").stateStr
// "STARTUP" ou "STARTUP2"
```

**Causes et Solutions** :

1. **Keyfile incorrect**
```bash
# Vérifier le keyfile
md5sum /etc/mongodb/keyfile
# Doit correspondre aux autres membres

# Copier depuis un membre existant
scp mongodb-01:/etc/mongodb/keyfile /etc/mongodb/
chmod 400 /etc/mongodb/keyfile
systemctl restart mongod
```

2. **replSetName incorrect**
```yaml
# mongod.conf
replication:
  replSetName: rs0  # DOIT correspondre
```

3. **Connectivité réseau**
```bash
# Depuis le nouveau membre
telnet mongodb-01 27017

# Vérifier le firewall
iptables -L | grep 27017
```

### Problème : Initial Sync Échoue

**Symptômes** :
```
[InitialSyncer] Initial sync failed: NetworkTimeout
```

**Solutions** :

```javascript
// 1. Vérifier l'oplog window
rs.printReplicationInfo()

// 2. Augmenter l'oplog si nécessaire
db.adminCommand({replSetResizeOplog: 1, size: 20480})

// 3. Redémarrer l'initial sync
// Sur le membre problématique
db.adminCommand({resync: 1})
```

### Problème : Lag Important Après Ajout

**Diagnostic** :
```javascript
rs.status().members.forEach(m => {
  if (m.state === 2) {
    var lag = (rs.status().date - m.optimeDate) / 1000
    print(m.name + " : " + lag + "s")
  }
})
```

**Solutions** :

1. **Identifier la source du problème**
```javascript
// Vérifier les slow queries
db.currentOp({"secs_running": {$gt: 5}})

// Profiler
db.setProfilingLevel(1, {slowms: 100})
db.system.profile.find().sort({ts: -1}).limit(5)
```

2. **Augmenter les ressources temporairement**
```javascript
// Réduire la charge de lecture
db.adminCommand({setParameter: 1, internalQueryExecMaxBlockingSortBytes: 100000000})
```

### Problème : Cannot Remove Member

**Erreur** :
```
replSetReconfig should only be run on PRIMARY
```

**Solution** :
```javascript
// 1. Identifier le Primary
rs.isMaster().primary

// 2. Se connecter au Primary
mongosh --host <primary>:27017

// 3. Retry
rs.remove("mongodb-04:27017")
```

## Conclusion

L'ajout et la suppression de membres dans un Replica Set sont des opérations critiques qui nécessitent :

- ✅ **Planification** : Vérifier les prérequis, estimer les impacts
- ✅ **Monitoring** : Surveiller la synchronisation et les performances
- ✅ **Validation** : Vérifier l'état du Replica Set après chaque opération
- ✅ **Documentation** : Tracer tous les changements

**Points clés** :
1. Maintenir toujours un nombre impair de votants
2. Ne jamais supprimer directement le Primary
3. Vérifier l'oplog window avant les ajouts
4. Attendre la synchronisation complète entre opérations
5. Planifier les changements en heures creuses
6. Avoir un plan de rollback

Une gestion rigoureuse des membres garantit la stabilité et la haute disponibilité du Replica Set en production.

⏭️ [Read Preference](/09-replication/08-read-preference.md)
