🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.3 Problèmes de Réplication

## Vue d'ensemble

La réplication est au cœur de la haute disponibilité MongoDB. Les problèmes de réplication peuvent compromettre la disponibilité, la cohérence des données et les performances. Cette section fournit des méthodologies complètes pour diagnostiquer et résoudre tous les problèmes liés à la réplication.

---

## Table des Matières

1. [Replication Lag](#1-replication-lag)
2. [Élections Fréquentes](#2-%C3%A9lections-fr%C3%A9quentes)
3. [Synchronisation Échouée](#3-synchronisation-%C3%A9chou%C3%A9e)
4. [Oplog Insuffisant](#4-oplog-insuffisant)
5. [Problèmes de Heartbeat](#5-probl%C3%A8mes-de-heartbeat)
6. [Rollback](#6-rollback)
7. [Split-Brain](#7-split-brain)
8. [Configuration et Monitoring](#8-configuration-et-monitoring)

---

## 1. Replication Lag

### Symptômes

```
Secondaries falling behind primary
Read operations returning stale data
Increasing lag over time
Applications experiencing inconsistencies
```

### Causes Possibles

- Charge d'écriture élevée sur le primary
- Secondaries sous-dimensionnés (CPU, RAM, I/O)
- Réseau lent ou instable entre membres
- Opérations longues sur le primary
- Secondary occupé par des lectures
- Index manquants sur secondary
- Oplog trop petit

---

### Diagnostic Pas à Pas

#### Étape 1 : Mesurer le Replication Lag

```javascript
// Depuis le primary
rs.printReplicationInfo()

// Sortie exemple :
// configured oplog size:   990MB
// log length start to end: 2483secs (0.69hrs)
// oplog first event time:  Mon Jan 15 2024 10:15:20
// oplog last event time:   Mon Jan 15 2024 10:56:43
// now:                     Mon Jan 15 2024 11:00:00

// Depuis un secondary
rs.printSecondaryReplicationInfo()

// Sortie exemple :
// source: mongodb-secondary1:27017
//   syncedTo: Mon Jan 15 2024 10:55:30
//   120 secs (0.03 hrs) behind the primary
// source: mongodb-secondary2:27017
//   syncedTo: Mon Jan 15 2024 10:56:40
//   20 secs (0.01 hrs) behind the primary
```

**Interpréter les résultats :**

```javascript
// Analyse détaillée
function analyzeLag() {
  const status = rs.status()

  const primary = status.members.find(m => m.stateStr === "PRIMARY")
  const primaryOptime = primary.optimeDate

  status.members.forEach(member => {
    if (member.stateStr === "SECONDARY") {
      const lag = (primaryOptime - member.optimeDate) / 1000  // secondes

      console.log(`${member.name}:`)
      console.log(`  Lag: ${lag} seconds`)
      console.log(`  State: ${member.stateStr}`)
      console.log(`  Health: ${member.health}`)

      if (lag > 60) {
        console.log(`  ⚠️ WARNING: High lag detected!`)
      }
    }
  })
}

analyzeLag()
```

**Seuils d'alerte :**
- < 10 secondes : Normal
- 10-60 secondes : Surveiller
- 60-300 secondes : Attention
- > 300 secondes : Critique

#### Étape 2 : Identifier la Cause du Lag

```javascript
// 1. Vérifier l'état général du replica set
rs.status()

// 2. Vérifier les opérations en cours sur le primary
db.currentOp({
  "active": true,
  "secs_running": {$gt: 10}
})

// 3. Vérifier la charge du secondary
// Se connecter au secondary
db.getMongo().setReadPref("secondary")

db.currentOp({
  "active": true
})

// 4. Vérifier les métriques de réplication
db.serverStatus().metrics.repl
```

#### Étape 3 : Analyser les Métriques Réseau

```bash
# Latence entre primary et secondary
ping -c 10 mongodb-secondary1

# Bande passante
iperf3 -c mongodb-secondary1

# Paquets perdus
mtr mongodb-secondary1 --report

# Connexions réseau
netstat -an | grep :27017
```

```javascript
// Métriques réseau MongoDB
db.serverStatus().network

// Vérifier le buffer de réplication
db.serverStatus().metrics.repl.buffer
```

#### Étape 4 : Analyser l'Oplog

```javascript
// Taille et utilisation de l'oplog
db.getReplicationInfo()

// Sortie :
// {
//   logSizeMB: 990,
//   usedMB: 543.2,
//   timeDiff: 2483,              // secondes couvertes
//   timeDiffHours: 0.69,
//   tFirst: "Mon Jan 15 2024...",
//   tLast: "Mon Jan 15 2024...",
//   now: "Mon Jan 15 2024..."
// }

// Analyser le contenu de l'oplog
use local
db.oplog.rs.find().sort({$natural: -1}).limit(10)

// Voir les plus grosses opérations
db.oplog.rs.aggregate([
  {$project: {
    ts: 1,
    op: 1,
    ns: 1,
    size: {$bsonSize: "$$ROOT"}
  }},
  {$sort: {size: -1}},
  {$limit: 20}
])
```

#### Étape 5 : Vérifier les Ressources du Secondary

```bash
# CPU
top -p $(pgrep mongod)

# Mémoire
free -h
cat /proc/$(pgrep mongod)/status | grep VmRSS

# I/O
iostat -x 1 5

# Depuis MongoDB
mongosh --host secondary-host --eval "db.serverStatus().mem"
mongosh --host secondary-host --eval "db.serverStatus().wiredTiger.cache"
```

---

### Résolution Pas à Pas

#### Solution 1 : Réduire la Charge sur le Primary

```javascript
// 1. Identifier les opérations lourdes
db.currentOp({
  "secs_running": {$gt: 5},
  "op": {$in: ["insert", "update", "delete"]}
})

// 2. Optimiser les écritures avec bulk operations
// ❌ MAUVAIS
for (let i = 0; i < 10000; i++) {
  db.logs.insertOne({timestamp: new Date(), data: i})
}

// ✅ BON
const docs = []
for (let i = 0; i < 10000; i++) {
  docs.push({timestamp: new Date(), data: i})
}
db.logs.insertMany(docs, {ordered: false})

// 3. Utiliser un write concern moins strict pour les données non critiques
db.logs.insertMany(docs, {
  writeConcern: {w: 1}  // Ne pas attendre les secondaries
})
```

#### Solution 2 : Augmenter les Ressources du Secondary

**Scaling vertical :**

```bash
# 1. Identifier le goulot d'étranglement
# CPU > 80% → Augmenter CPU
# RAM < 20% libre → Augmenter RAM
# I/O await > 10ms → Disques plus rapides

# 2. Pour un secondary d'un replica set :
# a. Le mettre en mode RECOVERING
rs.stepDown(120)  # Sur le primary si c'est lui

# Depuis le secondary à upgrader
db.adminCommand({replSetMaintenance: 1})

# b. Arrêter MongoDB
sudo systemctl stop mongod

# c. Upgrader les ressources (cloud, hardware, etc.)

# d. Redémarrer
sudo systemctl start mongod

# e. Sortir du mode maintenance
db.adminCommand({replSetMaintenance: 0})

# f. Vérifier la synchronisation
rs.status()
```

**Configuration WiredTiger optimisée :**

```yaml
# /etc/mongod.conf sur le secondary
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 16  # Augmenter le cache
      journalCompressor: snappy

    collectionConfig:
      blockCompressor: snappy
```

#### Solution 3 : Optimiser la Réplication

**Créer les mêmes index sur les secondaries :**

```javascript
// Vérifier les index sur le primary
db.collection.getIndexes()

// Se connecter au secondary
db.getMongo().setReadPref("secondary")

// Vérifier les index manquants
// Créer les index manquants en background
db.collection.createIndex({field: 1}, {background: true})

// Note : MongoDB 4.2+ build les index automatiquement
// sur tous les membres du replica set
```

**Ajuster les paramètres de réplication :**

```javascript
// Augmenter le batch size de réplication
db.adminCommand({
  setParameter: 1,
  replBatchLimitBytes: 104857600  // 100MB (défaut: 100MB)
})

// Augmenter le nombre de threads de réplication
db.adminCommand({
  setParameter: 1,
  replWriterThreadCount: 16  // Défaut: varie selon CPU
})
```

#### Solution 4 : Augmenter la Taille de l'Oplog

```javascript
// 1. Vérifier la taille actuelle
db.getReplicationInfo()

// 2. Calculer la nouvelle taille nécessaire
// Formule : Taille = (Taux d'écriture MB/s) × (Temps de récupération voulu en secondes)
// Exemple : 10 MB/s × 3600s (1h) = 36000 MB = 36 GB

// 3. Redimensionner l'oplog (MongoDB 4.0+)
db.adminCommand({
  replSetResizeOplog: 1,
  size: 36000  // MB
})

// 4. Vérifier
db.getReplicationInfo()
```

**Pour versions < 4.0 (procédure complexe) :**

```javascript
// 1. Arrêter le membre en mode standalone
sudo systemctl stop mongod

// 2. Démarrer sans réplication
mongod --dbpath /var/lib/mongodb --port 37017

// 3. Se connecter
mongosh --port 37017

// 4. Créer un nouveau oplog
use local
db.temp.drop()
db.temp.save({size: 36000})  // MB
db.oplog.rs.drop()
db.runCommand({
  create: "oplog.rs",
  capped: true,
  size: 36000 * 1024 * 1024
})

// 5. Redémarrer en mode replica set
sudo systemctl start mongod
```

#### Solution 5 : Dédier un Secondary aux Lectures

```javascript
// Configuration d'un secondary pour analytics
cfg = rs.conf()

// Trouver l'index du membre
var memberIndex = cfg.members.findIndex(m => m.host === "analytics-secondary:27017")

// Augmenter la priorité à 0 (ne peut pas devenir primary)
cfg.members[memberIndex].priority = 0

// Marquer comme hidden (invisible pour les lectures normales)
cfg.members[memberIndex].hidden = true

// Appliquer
rs.reconfig(cfg)

// Maintenant diriger les analytics vers ce secondary
const client = new MongoClient(uri, {
  readPreference: {
    mode: 'secondary',
    tags: [{analytics: 'true'}]
  }
})
```

---

## 2. Élections Fréquentes

### Symptômes

```
Primary changes frequently
Applications experiencing connection resets
"Not master" errors
Replica set instability
Multiple elections in short period
```

### Causes Possibles

- Problèmes réseau (timeouts, paquets perdus)
- Heartbeat timeout trop court
- Primary surchargé
- Configuration de priorité inadéquate
- Hardware instable
- Conflit de votes (membres pairs)

---

### Diagnostic Pas à Pas

#### Étape 1 : Analyser l'Historique des Élections

```javascript
// Voir les changements récents de primary
rs.status().members.forEach(member => {
  console.log(`${member.name}: ${member.stateStr}`)
  if (member.electionTime) {
    console.log(`  Last election: ${member.electionTime}`)
  }
})

// Analyser les logs
// grep "election" /var/log/mongodb/mongod.log | tail -50

// Compter les élections récentes
db.getSiblingDB("local").replset.election.find().sort({ts: -1}).limit(20)
```

#### Étape 2 : Vérifier la Configuration du Replica Set

```javascript
// Configuration actuelle
var cfg = rs.conf()
printjson(cfg)

// Points à vérifier :
// 1. Nombre de membres (impair recommandé)
console.log("Member count:", cfg.members.length)

// 2. Settings d'élection
console.log("Election timeout:", cfg.settings.electionTimeoutMillis, "ms")
console.log("Heartbeat interval:", cfg.settings.heartbeatIntervalMillis, "ms")
console.log("Heartbeat timeout:", cfg.settings.heartbeatTimeoutSecs, "s")

// 3. Priorités des membres
cfg.members.forEach(m => {
  console.log(`${m.host}: priority ${m.priority}`)
})

// 4. Votes
cfg.members.forEach(m => {
  console.log(`${m.host}: votes ${m.votes}`)
})
```

#### Étape 3 : Analyser la Connectivité Réseau

```bash
# Tester la latence entre tous les membres
for host in mongodb1 mongodb2 mongodb3; do
  echo "Testing $host"
  ping -c 10 $host | grep avg
  echo "---"
done

# Vérifier les timeouts
mtr --report --report-cycles 100 mongodb-primary

# Statistiques réseau MongoDB
mongosh --eval "db.serverStatus().network"
```

```javascript
// Depuis MongoDB, vérifier les heartbeats
rs.status().members.forEach(member => {
  console.log(`${member.name}:`)
  console.log(`  Last heartbeat: ${member.lastHeartbeat}`)
  console.log(`  Last heartbeat recv: ${member.lastHeartbeatRecv}`)
  console.log(`  Ping: ${member.pingMs}ms`)
  console.log(`  State: ${member.stateStr}`)
})
```

#### Étape 4 : Vérifier les Ressources Système

```bash
# Sur chaque membre du replica set
# CPU
top -bn1 | grep mongod

# Mémoire
free -h

# Swap (doit être 0)
vmstat 1 5

# I/O
iostat -x 1 5

# Charge système
uptime
```

---

### Résolution Pas à Pas

#### Solution 1 : Ajuster les Timeouts

```javascript
// Configuration actuelle
var cfg = rs.conf()

// Paramètres par défaut :
// electionTimeoutMillis: 10000 (10 secondes)
// heartbeatIntervalMillis: 2000 (2 secondes)
// heartbeatTimeoutSecs: 10

// Pour réseau instable, augmenter les timeouts
cfg.settings = cfg.settings || {}
cfg.settings.electionTimeoutMillis = 20000    // 20 secondes
cfg.settings.heartbeatTimeoutSecs = 20        // 20 secondes
cfg.settings.heartbeatIntervalMillis = 4000   // 4 secondes

// Appliquer
rs.reconfig(cfg)

// Vérifier
rs.conf().settings
```

**⚠️ Attention :**
- Timeouts trop longs = Failover plus lent
- Timeouts trop courts = Élections inutiles
- Trouver le bon équilibre selon votre réseau

#### Solution 2 : Ajuster les Priorités

```javascript
// Définir un primary préféré
var cfg = rs.conf()

// Primary préféré (priorité haute)
cfg.members[0].priority = 2
cfg.members[0].host  // "mongodb-primary:27017"

// Secondaries (priorité normale)
cfg.members[1].priority = 1
cfg.members[2].priority = 1

// Secondary de backup (priorité basse, ne devient primary qu'en dernier recours)
cfg.members[3].priority = 0.5

// Arbiter (priorité 0, ne peut jamais être primary)
cfg.members[4].priority = 0

// Appliquer
rs.reconfig(cfg)
```

**Configuration recommandée :**

```javascript
// Datacenter principal (priorité haute)
cfg.members[0].priority = 2  // Primary préféré
cfg.members[1].priority = 1  // Secondary local

// Datacenter secondaire (priorité basse)
cfg.members[2].priority = 0.5  // Ne devient primary qu'en cas de désastre

// Arbiter (datacenter tiers)
cfg.members[3].priority = 0
cfg.members[3].arbiterOnly = true
```

#### Solution 3 : Assurer un Nombre Impair de Membres

```javascript
// Problème : Nombre pair de membres votants
rs.conf().members.length  // 4 membres

// Solution 1 : Ajouter un arbiter (léger, pas de données)
rs.addArb("mongodb-arbiter:27017")

// Solution 2 : Retirer un membre (si possible)
rs.remove("mongodb-secondary3:27017")

// Solution 3 : Ajuster les votes
var cfg = rs.conf()
cfg.members[3].votes = 0  // Membre non-votant
rs.reconfig(cfg)

// Vérification
rs.conf().members.filter(m => m.votes === 1).length  // Doit être impair
```

#### Solution 4 : Isoler les Problèmes Réseau

**Configuration avec zones :**

```javascript
var cfg = rs.conf()

// Définir les zones géographiques
cfg.members[0].tags = {datacenter: "dc1", rack: "r1"}
cfg.members[1].tags = {datacenter: "dc1", rack: "r2"}
cfg.members[2].tags = {datacenter: "dc2", rack: "r1"}

// Configuration de write concern pour éviter les élections
cfg.settings = cfg.settings || {}
cfg.settings.getLastErrorModes = {
  multiDatacenter: {datacenter: 2}  // Au moins 2 datacenters
}

rs.reconfig(cfg)

// Utiliser dans les écritures
db.collection.insertOne(doc, {
  writeConcern: {w: "multiDatacenter"}
})
```

**Priorité de lecture par localité :**

```javascript
// Node.js - Préférer les membres locaux
const client = new MongoClient(uri, {
  readPreference: {
    mode: 'nearest',
    maxStalenessSeconds: 90
  }
})
```

#### Solution 5 : Monitorer et Alerter

```javascript
// Script de monitoring des élections
function monitorElections() {
  const status = rs.status()

  // Vérifier la stabilité du primary
  const primary = status.members.find(m => m.stateStr === "PRIMARY")

  if (!primary) {
    console.error("ALERT: No primary elected!")
    return
  }

  const uptimeSeconds = status.members.reduce((max, m) => {
    if (m.stateStr === "PRIMARY") {
      return m.uptime
    }
    return max
  }, 0)

  if (uptimeSeconds < 300) {  // Moins de 5 minutes
    console.warn(`WARNING: Recent election, primary uptime: ${uptimeSeconds}s`)
  }

  // Vérifier les heartbeats
  status.members.forEach(member => {
    if (member.pingMs > 1000) {  // > 1 seconde
      console.warn(`WARNING: High ping to ${member.name}: ${member.pingMs}ms`)
    }
  })
}

// Exécuter toutes les minutes
setInterval(monitorElections, 60000)
```

---

## 3. Synchronisation Échouée

### Symptômes

```
Secondary stuck in STARTUP2 or RECOVERING
"Initial sync failed" errors
Secondary not catching up
Continuous resync attempts
```

### Causes Possibles

- Oplog trop court (données expirées avant sync complète)
- Problèmes réseau pendant la synchronisation
- Espace disque insuffisant
- Collections très volumineuses
- Index builds longs
- Permissions insuffisantes

---

### Diagnostic Pas à Pas

#### Étape 1 : Identifier l'État du Membre

```javascript
// État du replica set
rs.status()

// Vérifier l'état spécifique
rs.status().members.forEach(member => {
  if (member.stateStr !== "PRIMARY" && member.stateStr !== "SECONDARY") {
    console.log(`${member.name}: ${member.stateStr}`)
    console.log(`  Error: ${member.lastHeartbeatMessage}`)
    console.log(`  Uptime: ${member.uptime}s`)
  }
})
```

#### Étape 2 : Analyser les Logs

```bash
# Rechercher les erreurs de synchronisation
grep -i "sync\|initial sync\|replication" /var/log/mongodb/mongod.log | tail -100

# Messages clés à rechercher :
# - "initial sync failed"
# - "replSet error"
# - "oplog rolled over"
# - "too stale to catch up"
```

```javascript
// Voir les détails depuis MongoDB
db.adminCommand({getLog: "global"}).log.filter(line =>
  line.includes("sync") || line.includes("replication")
).slice(-50)
```

#### Étape 3 : Vérifier l'Oplog

```javascript
// Sur le primary
db.getReplicationInfo()

// Sortie importante :
// timeDiff: durée couverte par l'oplog en secondes
// Si timeDiff est court et que la sync initiale prend plus longtemps
// → Problème !

// Calculer si l'oplog est suffisant
// Temps de sync estimé (en heures) > timeDiff en heures
// → Augmenter l'oplog
```

#### Étape 4 : Vérifier les Ressources

```bash
# Espace disque
df -h /var/lib/mongodb

# Vérifier les permissions
ls -la /var/lib/mongodb
# Doit appartenir à mongodb:mongodb

# Mémoire disponible
free -h

# Vérifier si le processus mongod tourne
ps aux | grep mongod
```

---

### Résolution Pas à Pas

#### Solution 1 : Resynchronisation Forcée

```javascript
// ⚠️ ATTENTION : Efface toutes les données du membre

// 1. Se connecter au secondary problématique
mongo --host secondary-problem:27017

// 2. Arrêter MongoDB
use admin
db.shutdownServer()

// 3. Supprimer les données (shell système)
sudo rm -rf /var/lib/mongodb/*
# Garder la configuration !

// 4. Redémarrer MongoDB
sudo systemctl start mongod

// 5. Le secondary va automatiquement commencer une initial sync
// Surveiller les logs
tail -f /var/log/mongodb/mongod.log

// 6. Vérifier la progression
rs.status()
```

#### Solution 2 : Utiliser une Copie Physique (Faster Resync)

```bash
# Méthode plus rapide pour grandes bases de données

# 1. Sur le secondary qui fonctionne bien
sudo systemctl stop mongod

# 2. Copier les données vers le nouveau secondary
sudo rsync -av --progress \
  /var/lib/mongodb/ \
  new-secondary:/var/lib/mongodb/

# 3. Sur le nouveau secondary
sudo chown -R mongodb:mongodb /var/lib/mongodb

# 4. Démarrer MongoDB
sudo systemctl start mongod

# 5. Le secondary va rattraper uniquement le delta de l'oplog
# Beaucoup plus rapide qu'une initial sync complète
```

#### Solution 3 : Augmenter l'Oplog Avant Sync

```javascript
// Si l'oplog est trop court

// 1. Sur le primary, augmenter l'oplog
db.adminCommand({
  replSetResizeOplog: 1,
  size: 51200  // 50 GB (ajuster selon vos besoins)
})

// 2. Vérifier
db.getReplicationInfo()
// timeDiff devrait être plus grand maintenant

// 3. Relancer la sync sur le secondary
// Voir Solution 1
```

#### Solution 4 : Synchronisation par Étapes

```javascript
// Pour bases très volumineuses

// 1. Créer un backup depuis le primary
mongodump --host primary:27017 --out /backup/mongodb

// 2. Restaurer sur le secondary
mongorestore --host secondary:27017 /backup/mongodb

// 3. Le secondary va maintenant seulement rattraper l'oplog
// Beaucoup plus rapide

// 4. Vérifier
rs.status()
```

#### Solution 5 : Configurer le Build Index en Arrière-Plan

```javascript
// Si la sync échoue à cause des index builds

// Sur le secondary, temporairement
db.adminCommand({
  setParameter: 1,
  maxIndexBuildMemoryUsageMegabytes: 1024  // 1GB pour index builds
})

// Créer les index en arrière-plan
db.collection.createIndex({field: 1}, {background: true})
```

---

## 4. Oplog Insuffisant

### Symptômes

```
"Oplog too small" warnings
Members frequently resyncing
Unable to recover from downtime
Rapid oplog rollover
```

### Causes Possibles

- Taux d'écriture élevé
- Oplog dimensionné trop petit
- Opérations volumineuses
- Pas de monitoring de l'oplog

---

### Diagnostic Pas à Pas

#### Étape 1 : Analyser l'Oplog

```javascript
// Informations sur l'oplog
db.getReplicationInfo()

// Sortie :
// {
//   logSizeMB: 990,           // Taille configurée
//   usedMB: 850,              // Utilisé actuellement
//   timeDiff: 3600,           // Secondes couvertes (1h)
//   timeDiffHours: 1,
//   tFirst: "...",            // Premier événement
//   tLast: "...",             // Dernier événement
//   now: "..."
// }

// RÈGLE : timeDiffHours doit être > temps de récupération maximum prévu
// Recommandation : Au moins 24-72 heures

// Calculer le taux d'écriture
var oplogSize = db.getReplicationInfo().logSizeMB
var oplogHours = db.getReplicationInfo().timeDiffHours
var writeRateMBPerHour = oplogSize / oplogHours

print("Write rate: " + writeRateMBPerHour.toFixed(2) + " MB/hour")
```

#### Étape 2 : Analyser le Contenu de l'Oplog

```javascript
// Voir les plus grosses opérations
use local
db.oplog.rs.aggregate([
  {$project: {
    ts: 1,
    op: 1,
    ns: 1,
    o: 1,
    size: {$bsonSize: "$$ROOT"}
  }},
  {$sort: {size: -1}},
  {$limit: 20}
]).forEach(doc => {
  print(`${doc.ns} - ${doc.op} - ${(doc.size / 1024).toFixed(2)} KB`)
})

// Analyser la distribution des opérations
db.oplog.rs.aggregate([
  {$group: {
    _id: "$op",
    count: {$sum: 1},
    avgSize: {$avg: {$bsonSize: "$$ROOT"}}
  }},
  {$sort: {count: -1}}
])
```

#### Étape 3 : Monitoring Continu

```javascript
// Script de monitoring de l'oplog
function monitorOplog() {
  var info = db.getReplicationInfo()

  var utilizationPercent = (info.usedMB / info.logSizeMB) * 100
  var hoursRemaining = info.timeDiffHours

  console.log(`Oplog utilization: ${utilizationPercent.toFixed(1)}%`)
  console.log(`Time coverage: ${hoursRemaining.toFixed(1)} hours`)

  if (hoursRemaining < 24) {
    console.error(`CRITICAL: Oplog covers less than 24 hours!`)
  } else if (hoursRemaining < 48) {
    console.warn(`WARNING: Oplog covers less than 48 hours`)
  }

  return {
    utilizationPercent,
    hoursRemaining,
    sizeMB: info.logSizeMB,
    usedMB: info.usedMB
  }
}

monitorOplog()
```

---

### Résolution Pas à Pas

#### Solution 1 : Redimensionner l'Oplog (MongoDB 4.0+)

```javascript
// Calcul de la nouvelle taille
// Formule : Taille = Taux d'écriture (MB/h) × Heures souhaitées

// Exemple :
// Taux actuel : 500 MB/heure
// Couverture souhaitée : 72 heures
// Nouvelle taille : 500 × 72 = 36,000 MB = 36 GB

// Redimensionner (peut se faire en ligne !)
db.adminCommand({
  replSetResizeOplog: 1,
  size: 36000  // MB
})

// Vérifier
db.getReplicationInfo()

// Note : Faire sur tous les membres
// Primary
db.adminCommand({replSetResizeOplog: 1, size: 36000})

// Chaque secondary
db.getMongo().setReadPref("secondary")
db.adminCommand({replSetResizeOplog: 1, size: 36000})
```

#### Solution 2 : Optimiser les Écritures

```javascript
// Réduire les opérations volumineuses dans l'oplog

// ❌ PROBLÈME : Updates sans $set créent des opérations énormes
db.users.updateOne(
  {_id: 123},
  {
    name: "John",
    email: "john@example.com",
    profile: {/* gros objet */},
    history: [/* gros array */]
  }
)
// L'oplog contient TOUT le document !

// ✅ SOLUTION : Utiliser $set pour updates partiels
db.users.updateOne(
  {_id: 123},
  {$set: {
    "profile.city": "Paris",
    "lastLogin": new Date()
  }}
)
// L'oplog contient seulement les champs modifiés

// ❌ PROBLÈME : Deletes puis re-inserts
db.users.deleteOne({_id: 123})
db.users.insertOne({_id: 123, /* nouveau document */})
// 2 opérations dans l'oplog

// ✅ SOLUTION : replaceOne
db.users.replaceOne(
  {_id: 123},
  {/* nouveau document */}
)
// 1 seule opération
```

#### Solution 3 : Partitionnement des Données

```javascript
// Pour bases très actives

// Au lieu d'une grosse collection :
db.logs.insertOne({/* entrée log */})

// Créer des collections par période
db[`logs_${year}_${month}`].insertOne({/* entrée log */})

// Avec TTL sur les anciennes
db.logs_2023_01.createIndex(
  {createdAt: 1},
  {expireAfterSeconds: 7776000}  // 90 jours
)

// Avantage : Moins de pression sur l'oplog
```

#### Solution 4 : Monitoring et Alertes

```javascript
// Créer une alerte sur la couverture de l'oplog
function checkOplogHealth() {
  var info = db.getReplicationInfo()
  var hoursRemaining = info.timeDiffHours

  if (hoursRemaining < 24) {
    // Alerte critique
    sendAlert({
      level: "critical",
      message: `Oplog covers only ${hoursRemaining.toFixed(1)} hours`,
      action: "Increase oplog size immediately"
    })
  } else if (hoursRemaining < 48) {
    // Alerte warning
    sendAlert({
      level: "warning",
      message: `Oplog covers ${hoursRemaining.toFixed(1)} hours`,
      action: "Consider increasing oplog size"
    })
  }

  return hoursRemaining
}

// Vérifier toutes les heures
setInterval(checkOplogHealth, 3600000)
```

---

## 5. Problèmes de Heartbeat

### Symptômes

```
Members marked as unreachable
"Failed to send heartbeat" errors
Frequent state changes
Network timeout errors
```

### Causes Possibles

- Latence réseau élevée
- Paquets perdus
- Firewall bloquant les heartbeats
- Membres surchargés
- Configuration DNS

---

### Diagnostic Pas à Pas

#### Étape 1 : Vérifier les Heartbeats

```javascript
// État des heartbeats
rs.status().members.forEach(member => {
  console.log(`${member.name}:`)
  console.log(`  State: ${member.stateStr}`)
  console.log(`  Health: ${member.health}`)
  console.log(`  Ping: ${member.pingMs}ms`)
  console.log(`  Last heartbeat: ${member.lastHeartbeat}`)
  console.log(`  Last heartbeat message: ${member.lastHeartbeatMessage}`)
  console.log(`---`)
})

// Configuration des heartbeats
rs.conf().settings
// {
//   heartbeatIntervalMillis: 2000,    // Fréquence
//   heartbeatTimeoutSecs: 10,         // Timeout
//   electionTimeoutMillis: 10000      // Timeout élection
// }
```

#### Étape 2 : Tester la Connectivité

```bash
# Entre chaque paire de membres
# Depuis mongodb1
ping -c 100 mongodb2 | grep avg
ping -c 100 mongodb3 | grep avg

# Latence TCP sur port MongoDB
hping3 -S -p 27017 -c 100 mongodb2

# Test de bande passante
iperf3 -c mongodb2

# MTU path discovery
ping -M do -s 1472 -c 10 mongodb2
```

---

### Résolution Pas à Pas

#### Solution 1 : Ajuster les Timeouts

```javascript
var cfg = rs.conf()

// Pour réseau stable (datacenter local)
cfg.settings.heartbeatIntervalMillis = 2000   // 2s
cfg.settings.heartbeatTimeoutSecs = 10        // 10s

// Pour réseau instable (géo-distribué)
cfg.settings.heartbeatIntervalMillis = 5000   // 5s
cfg.settings.heartbeatTimeoutSecs = 20        // 20s

rs.reconfig(cfg)
```

#### Solution 2 : Optimiser le Réseau

```bash
# Augmenter les buffers TCP
sudo sysctl -w net.core.rmem_max=134217728
sudo sysctl -w net.core.wmem_max=134217728
sudo sysctl -w net.ipv4.tcp_rmem="4096 87380 134217728"
sudo sysctl -w net.ipv4.tcp_wmem="4096 65536 134217728"

# Rendre permanent
echo "net.core.rmem_max=134217728" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

---

## 6. Rollback

### Symptômes

```
"Rollback required" messages
Data inconsistencies after recovery
Rollback files created
Recent writes lost
```

### Causes Possibles

- Primary failure avant réplication
- Network partition
- Write concern insuffisant
- Rollback après élection

---

### Diagnostic Pas à Pas

```javascript
// Vérifier les rollbacks
db.serverStatus().metrics.repl.rollback

// Fichiers de rollback
// ls -lh /var/lib/mongodb/rollback/
```

---

### Résolution Pas à Pas

#### Solution 1 : Utiliser Write Concern Majority

```javascript
// Pour données critiques
db.orders.insertOne(order, {
  writeConcern: {w: "majority"}
})

// Empêche les rollbacks car confirme la réplication
```

#### Solution 2 : Récupérer les Données du Rollback

```bash
# Les fichiers de rollback sont dans /var/lib/mongodb/rollback/
cd /var/lib/mongodb/rollback/

# Examiner le contenu
mongorestore --dir=./rollback_data/ --dryRun

# Restaurer si nécessaire
mongorestore --dir=./rollback_data/
```

---

## 7. Split-Brain

### Symptômes

```
Multiple primaries
Data divergence
Conflicting writes
Network partition
```

### Diagnostic

```javascript
// Vérifier le nombre de primaries
db.adminCommand({isMaster: 1})

// Sur chaque membre
rs.status().members.filter(m => m.stateStr === "PRIMARY")
```

### Résolution

```javascript
// Forcer une reconfiguration
var cfg = rs.conf()
cfg.version++
rs.reconfig(cfg, {force: true})
```

---

## 8. Configuration et Monitoring

### Configuration Optimale

```javascript
var cfg = rs.conf()

// Settings recommandés
cfg.settings = {
  heartbeatIntervalMillis: 2000,
  heartbeatTimeoutSecs: 10,
  electionTimeoutMillis: 10000,
  catchUpTimeoutMillis: -1,  // Pas de limite pour le catchup
  getLastErrorModes: {
    majority: {datacenter: 2}
  }
}

// Priorités
cfg.members[0].priority = 2  // Primary préféré
cfg.members[1].priority = 1  // Secondary normal
cfg.members[2].priority = 0.5  // Backup
cfg.members[3].priority = 0  // Arbiter
cfg.members[3].arbiterOnly = true

rs.reconfig(cfg)
```

### Script de Monitoring Complet

```javascript
function replicaSetHealthCheck() {
  var status = rs.status()
  var health = {
    timestamp: new Date(),
    replSetName: status.set,
    members: [],
    alerts: []
  }

  // Analyser chaque membre
  status.members.forEach(member => {
    var memberHealth = {
      name: member.name,
      state: member.stateStr,
      health: member.health,
      uptime: member.uptime,
      pingMs: member.pingMs
    }

    // Lag pour les secondaries
    if (member.stateStr === "SECONDARY") {
      var primary = status.members.find(m => m.stateStr === "PRIMARY")
      if (primary) {
        var lag = (primary.optimeDate - member.optimeDate) / 1000
        memberHealth.lagSeconds = lag

        if (lag > 60) {
          health.alerts.push(`${member.name}: High replication lag (${lag}s)`)
        }
      }
    }

    // Ping élevé
    if (member.pingMs > 100) {
      health.alerts.push(`${member.name}: High ping (${member.pingMs}ms)`)
    }

    // Health
    if (member.health !== 1) {
      health.alerts.push(`${member.name}: Unhealthy (health: ${member.health})`)
    }

    health.members.push(memberHealth)
  })

  // Vérifier qu'il y a un primary
  var primaries = health.members.filter(m => m.state === "PRIMARY")
  if (primaries.length === 0) {
    health.alerts.push("CRITICAL: No primary elected")
  } else if (primaries.length > 1) {
    health.alerts.push("CRITICAL: Multiple primaries detected")
  }

  return health
}

// Exécuter et afficher
var health = replicaSetHealthCheck()
printjson(health)

if (health.alerts.length > 0) {
  print("\n⚠️  ALERTS:")
  health.alerts.forEach(alert => print("  - " + alert))
}
```

---

## Checklist de Dépannage Réplication

### Diagnostic Rapide (5 minutes)

```bash
# 1. État général
mongosh --eval "rs.status()"

# 2. Configuration
mongosh --eval "rs.conf()"

# 3. Lag de réplication
mongosh --eval "rs.printSecondaryReplicationInfo()"

# 4. Oplog
mongosh --eval "db.getReplicationInfo()"

# 5. Heartbeats
mongosh --eval "rs.status().members.forEach(m => print(m.name + ': ' + m.pingMs + 'ms'))"
```

### Actions Correctives par Problème

```
PROBLÈME: Replication Lag > 60s
→ Vérifier charge du primary
→ Vérifier ressources des secondaries
→ Augmenter oplog si nécessaire
→ Optimiser les requêtes

PROBLÈME: Élections fréquentes
→ Augmenter timeouts
→ Vérifier réseau (ping, paquets perdus)
→ Ajuster priorités
→ Assurer nombre impair de votants

PROBLÈME: Synchronisation échouée
→ Vérifier oplog (taille suffisante)
→ Vérifier espace disque
→ Forcer resync si nécessaire
→ Utiliser copie physique pour grandes bases

PROBLÈME: Oplog insuffisant
→ Calculer besoin (taux × heures)
→ Redimensionner avec replSetResizeOplog
→ Optimiser les écritures
→ Monitorer en continu

PROBLÈME: Heartbeat failures
→ Tester latence réseau
→ Augmenter timeouts
→ Vérifier firewall
→ Optimiser configuration réseau
```

---

## Conclusion

La réplication MongoDB est robuste mais nécessite :

1. **Configuration appropriée** (timeouts, priorités, oplog)
2. **Infrastructure stable** (réseau, ressources)
3. **Monitoring proactif** (lag, oplog, élections)
4. **Procédures claires** pour chaque type de problème

**Points critiques :**
- ✅ Oplog dimensionné pour 48-72h minimum
- ✅ Nombre impair de membres votants
- ✅ Timeouts adaptés à votre réseau
- ✅ Write concern majority pour données critiques
- ✅ Monitoring continu avec alertes

---


⏭️ [Problèmes de sharding](/22-depannage-resolution-problemes/04-problemes-sharding.md)
