🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.9 Failover et Haute Disponibilité

## Introduction

Le **failover** (basculement automatique) est le mécanisme par lequel MongoDB garantit la haute disponibilité en cas de défaillance du Primary. Comprendre les subtilités du processus de failover, les différents types de défaillances et les stratégies d'optimisation est essentiel pour concevoir et opérer des systèmes résilients en production.

## Architecture de Haute Disponibilité

### Principes Fondamentaux

La haute disponibilité dans MongoDB repose sur trois piliers :

```
┌────────────────────────────────────────────┐
│     Haute Disponibilité MongoDB            │
├────────────────────────────────────────────┤
│                                            │
│  1. REDONDANCE                             │
│     ├─ Replica Set (≥3 membres)            │
│     └─ Données répliquées sur tous nœuds   │
│                                            │
│  2. DÉTECTION AUTOMATIQUE                  │
│     ├─ Heartbeats (2 secondes)             │
│     └─ Timeout (10 secondes)               │
│                                            │
│  3. FAILOVER AUTOMATIQUE                   │
│     ├─ Élection d'un nouveau Primary       │
│     └─ Reconfiguration automatique         │
│                                            │
└────────────────────────────────────────────┘
```

### Objectifs de Disponibilité

| Métrique | Calcul | Objectif Production |
|----------|--------|---------------------|
| **Availability (%)** | (Uptime / Total Time) × 100 | 99.9% - 99.99% |
| **RTO** (Recovery Time Objective) | Temps max pour restaurer | 30-120 secondes |
| **RPO** (Recovery Point Objective) | Perte de données max acceptable | 0 (w: majority) |
| **MTBF** (Mean Time Between Failures) | Temps moyen entre pannes | Maximiser |
| **MTTR** (Mean Time To Recovery) | Temps moyen de récupération | Minimiser |

**Calcul de disponibilité** :

```
Disponibilité 99.9%  = ~8.76 heures downtime/an
Disponibilité 99.95% = ~4.38 heures downtime/an
Disponibilité 99.99% = ~52.6 minutes downtime/an
```

## Mécanisme de Failover

### Séquence Complète de Failover

```
T0: État Normal
┌─────────────┐  Heartbeats  ┌──────────────┐
│   PRIMARY   │ ←----------→ │  SECONDARY-1 │
│ mongodb-01  │              │  mongodb-02  │
└─────────────┘              └──────────────┘
       ↕
    Heartbeats
       ↕
┌──────────────┐
│  SECONDARY-2 │
│  mongodb-03  │
└──────────────┘

───────────────────────────────────────────────

T1: Défaillance du Primary (crash, réseau, etc.)
┌─────────────┐              ┌──────────────┐
│    DOWN     │      ✗       │  SECONDARY-1 │
│ mongodb-01  │              │  mongodb-02  │
└─────────────┘              └──────────────┘


┌──────────────┐
│  SECONDARY-2 │
│  mongodb-03  │
└──────────────┘

───────────────────────────────────────────────

T2: Détection (après electionTimeoutMillis = 10s)
                             ┌────────────────┐
         Timeout             │  SECONDARY-1   │
         détecté             │ Pas de Primary!│
                             └────────────────┘
                                    ↓
                             Initie élection

┌──────────────┐                    ↓
│  SECONDARY-2 │ ←─── Demande vote ─┘
│  mongodb-03  │

───────────────────────────────────────────────

T3: Élection (1-2 secondes)
                             ┌──────────────┐
                             │  CANDIDATE   │
                             │  mongodb-02  │
                             │  Term: 43    │
                             └──────────────┘
                                    ↑
                        Vote accordé│
                                    │
┌──────────────┐                    │
│  SECONDARY-2 │ ───────────────────┘
│  mongodb-03  │
│  Vote: OUI   │
└──────────────┘

───────────────────────────────────────────────

T4: Nouveau Primary Élu
                             ┌──────────────┐
                             │   PRIMARY    │
                             │  mongodb-02  │
                             │  Term: 43    │
                             └──────────────┘
                                    ↕
                               Heartbeats
                                    ↕
┌──────────────┐
│  SECONDARY-2 │
│  mongodb-03  │
└──────────────┘

───────────────────────────────────────────────

T5: Ancien Primary Revient
┌─────────────┐              ┌──────────────┐
│  SECONDARY  │ Heartbeat    │   PRIMARY    │
│ mongodb-01  │ ←----------→ │  mongodb-02  │
│  Term: 42   │              │  Term: 43    │
└─────────────┘              └──────────────┘
     ↕                              ↕
  Rollback si                  Réplication
  nécessaire                    normale
     ↕                              ↕
┌──────────────┐
│  SECONDARY-2 │
│  mongodb-03  │
└──────────────┘
```

### Chronologie Détaillée

**Phase 1 : Défaillance (T0)**
```
0ms : Primary devient inaccessible
     - Crash système
     - Panne réseau
     - Charge excessive
```

**Phase 2 : Détection (T0 + 10s)**
```
10000ms : Les Secondary détectent l'absence de heartbeat
         - Timeout = electionTimeoutMillis (défaut: 10s)
         - Chaque Secondary démarre un timer indépendant
```

**Phase 3 : Initiation Élection (T0 + 10s + randomDelay)**
```
10000-11000ms : Un Secondary initie l'élection
               - Incrémente son term (42 → 43)
               - Passe à l'état CANDIDATE
               - Vote pour lui-même
               - Envoie RequestVote aux autres
```

**Phase 4 : Vote (T0 + 10-12s)**
```
10500-12000ms : Collecte des votes
               - Chaque membre vote pour le premier CANDIDATE valide
               - Validation de l'OpTime (doit être à jour)
               - Vérification des priorités
```

**Phase 5 : Élection (T0 + 11-13s)**
```
11000-13000ms : CANDIDATE obtient la majorité
               - Devient PRIMARY
               - Envoie notification aux autres membres
```

**Phase 6 : Catchup (T0 + 13-43s)**
```
13000-43000ms : Nouveau Primary en catchup (optionnel)
               - Réplique les dernières opérations manquantes
               - Durée : catchUpTimeoutMillis (défaut: -1 = illimité)
               - Après catchup, accepte les écritures
```

**Temps Total de Failover** :
```
Temps minimal : ~12-15 secondes
Temps typique  : ~20-30 secondes
Temps maximal  : ~40-60 secondes (avec catchup)
```

## Types de Défaillances

### 1. Défaillance Matérielle

#### Crash Serveur

```javascript
// Symptômes
rs.status().members.find(m => m.name === "mongodb-01:27017")
// {
//   name: "mongodb-01:27017",
//   health: 0,
//   state: 8,        // DOWN
//   stateStr: "DOWN (not reachable/healthy)",
//   lastHeartbeat: ISODate("2024-01-15T10:15:23Z")
// }
```

**Causes** :
- Panne matérielle (CPU, RAM, disque)
- Kernel panic
- Alimentation électrique
- Surchauffe

**Détection** : Immédiate (heartbeat échoue)

**Récupération** :
```bash
# 1. Diagnostiquer le problème
journalctl -u mongod -n 100

# 2. Réparer/remplacer le hardware

# 3. Redémarrer MongoDB
systemctl start mongod

# 4. Le membre rejoindra automatiquement comme SECONDARY
```

#### Défaillance Disque

```bash
# Symptômes dans les logs
[ERROR] WiredTiger error: disk full
[FATAL] Exception in initAndListen: DBPathInUse

# Vérification
df -h /data/mongodb
# Filesystem      Size  Used Avail Use% Mounted on
# /dev/sda1       100G  100G    0G 100% /data
```

**Prévention** :
```javascript
// Monitoring de l'espace disque
function checkDiskSpace() {
  var dbStats = db.stats()
  var dataSize = dbStats.dataSize / 1024 / 1024 / 1024  // GB
  var storageSize = dbStats.storageSize / 1024 / 1024 / 1024  // GB

  // Alerter si utilisation > 80%
  if (storageSize > dbStats.fsUsedSize * 0.8) {
    print("WARNING: Disk usage > 80%")
  }
}
```

### 2. Défaillance Réseau

#### Partition Réseau Complète

```
Avant partition :
┌──────────┐     ┌──────────┐     ┌──────────┐
│ Primary  │──── │Secondary1│──── │Secondary2│
└──────────┘     └──────────┘     └──────────┘

Après partition (exemple split 2-1) :
┌──────────┐     ┌──────────┐     ║ ┌──────────┐
│ Primary  │──── │Secondary1│     ║ │Secondary2│
│ (reste)  │     │          │     ║ │ (isolé)  │
└──────────┘     └──────────┘     ║ └──────────┘
  Partition A (majorité)          ║  Partition B
  → Continue opérations           ║  → READ ONLY
```

**Comportement** :
```javascript
// Partition A (2 membres sur 3 - majorité)
// Primary reste PRIMARY
// Écritures continuent normalement

// Partition B (1 membre sur 3 - minorité)
// Secondary-2 devient SECONDARY en READ-ONLY
// Erreur sur tentatives d'écriture :
// "not master and slaveOk=false"
```

**Prévention du Split-Brain** :
```
Grâce à l'exigence de majorité :
- Seule la partition avec majorité peut avoir un Primary
- L'autre partition n'a QUE des Secondary
- Impossible d'avoir 2 Primary simultanément
```

#### Latence Réseau Élevée

```javascript
// Symptômes
rs.status().members.forEach(m => {
  if (m.pingMs && m.pingMs > 100) {
    print(`WARNING: High latency to ${m.name}: ${m.pingMs}ms`)
  }
})
```

**Impact** :
- Replication lag augmenté
- Élections plus fréquentes
- Timeout de heartbeat

**Solution** :
```javascript
// Augmenter electionTimeoutMillis pour réseaux WAN
cfg = rs.conf()
cfg.settings.electionTimeoutMillis = 30000  // 30 secondes
rs.reconfig(cfg)
```

#### Perte Intermittente de Paquets

```bash
# Diagnostic
ping -c 100 mongodb-secondary
# 100 packets transmitted, 85 received, 15% packet loss

# Impact sur réplication
# Logs MongoDB :
# [replication] sync source: mongodb-02:27017 dropped connection
# [replication] changing sync source from mongodb-02:27017 to mongodb-03:27017
```

### 3. Défaillance Logicielle

#### Out of Memory (OOM)

```bash
# Logs système
Jan 15 10:15:23 kernel: Out of memory: Kill process 12345 (mongod)

# Logs MongoDB
[ERROR] Fatal Assertion 28561
```

**Prévention** :
```javascript
// Configurer WiredTiger cache correctement
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  // ~50% de RAM disponible

// Monitoring
db.serverStatus().wiredTiger.cache["bytes currently in the cache"]
```

#### Deadlock / Blocage

```javascript
// Identifier les opérations bloquées
db.currentOp({
  "secs_running": { $gte: 30 },
  "op": { $in: ["query", "update", "remove"] }
})

// Tuer l'opération problématique
db.killOp(opid)
```

#### Corruption de Données WiredTiger

```bash
# Symptôme
[ERROR] WiredTiger error: WT_PANIC: WiredTiger library panic

# Récupération
mongod --repair --dbpath /data/mongodb

# Si échec, restaurer depuis backup
```

### 4. Défaillance Applicative

#### Surcharge de Connexions

```javascript
// Monitoring
db.serverStatus().connections
// {
//   current: 40000,
//   available: 819228,
//   totalCreated: 52341234
// }

// Si current approche de la limite
// Augmenter maxIncomingConnections
net:
  maxIncomingConnections: 65536
```

#### Requêtes Lentes Saturant le Système

```javascript
// Identifier les requêtes lentes en cours
db.currentOp({
  "active": true,
  "secs_running": { $gt: 10 },
  "op": "query"
}).inprog.forEach(op => {
  print(`Slow query: ${op.secs_running}s`)
  print(`Query: ${JSON.stringify(op.query)}`)
  print(`OpID: ${op.opid}`)
})

// Tuer si nécessaire
db.killOp(opid)
```

## Stratégies de Haute Disponibilité

### 1. Topologie N+2 (Recommandée)

```javascript
// 5 membres au lieu de 3
{
  members: [
    { _id: 0, host: "mongodb-01:27017", priority: 10 },  // Primary préféré
    { _id: 1, host: "mongodb-02:27017", priority: 9 },
    { _id: 2, host: "mongodb-03:27017", priority: 8 },

    // Membres supplémentaires pour résilience
    { _id: 3, host: "mongodb-04:27017", priority: 1 },
    { _id: 4, host: "mongodb-05:27017", priority: 1 }
  ]
}
```

**Avantages** :
- Tolérance à 2 défaillances simultanées
- Majorité : 3 sur 5
- Si 2 membres tombent : 3 restent (majorité maintenue)

### 2. Déploiement Multi-Datacenter

```javascript
// Configuration géo-distribuée
{
  members: [
    // DC1 (Principal) - 3 membres
    { _id: 0, host: "dc1-mongodb-01:27017", priority: 10, tags: {dc: "dc1"} },
    { _id: 1, host: "dc1-mongodb-02:27017", priority: 9, tags: {dc: "dc1"} },
    { _id: 2, host: "dc1-mongodb-03:27017", priority: 8, tags: {dc: "dc1"} },

    // DC2 (DR) - 2 membres
    { _id: 3, host: "dc2-mongodb-01:27017", priority: 1, tags: {dc: "dc2"} },
    { _id: 4, host: "dc2-mongodb-02:27017", priority: 1, votes: 0, tags: {dc: "dc2"} }
  ],

  settings: {
    electionTimeoutMillis: 30000,  // Latence WAN
    getLastErrorModes: {
      multiDC: { dc: 2 }  // Écriture sur au moins 2 DC
    }
  }
}
```

**Scénarios de défaillance** :

| Scénario | Membres UP | Majorité | Primary Possible | Écritures |
|----------|-----------|----------|------------------|-----------|
| Normal | DC1:3, DC2:2 | 3/4 | ✅ DC1 | ✅ |
| Perte DC2 | DC1:3, DC2:0 | 3/3 | ✅ DC1 | ✅ |
| Perte DC1 | DC1:0, DC2:2 | 2/4 | ❌ Pas de majorité | ❌ |
| 1 down DC1 | DC1:2, DC2:2 | 2/4 | ✅ DC1 | ✅ |
| 2 down DC1 | DC1:1, DC2:2 | 1/4 | ❌ Pas de majorité | ❌ |

### 3. Configuration avec Delayed Member

```javascript
{
  members: [
    // Membres normaux
    { _id: 0, host: "mongodb-01:27017", priority: 2 },
    { _id: 1, host: "mongodb-02:27017", priority: 1 },
    { _id: 2, host: "mongodb-03:27017", priority: 1 },

    // Delayed member (protection contre erreurs humaines)
    {
      _id: 3,
      host: "mongodb-delayed:27017",
      priority: 0,
      hidden: true,
      slaveDelay: 3600,  // 1 heure
      tags: { backup: "delayed" }
    }
  ]
}
```

**Récupération après suppression accidentelle** :
```javascript
// Données supprimées accidentellement à 14h00
// Le delayed member a encore les données jusqu'à 13h00

// 1. Se connecter au delayed member
mongosh --host mongodb-delayed:27017

// 2. Exporter les données
mongodump --db mydb --collection users --out /backup/recovery

// 3. Restaurer sur le Primary
mongorestore --host mongodb-01:27017 /backup/recovery
```

### 4. Monitoring et Alerting Proactif

```javascript
// Script de monitoring continu
function continuousHealthCheck() {
  setInterval(() => {
    const status = rs.status()
    const alerts = []

    // Check 1 : Présence du Primary
    const primary = status.members.filter(m => m.state === 1)
    if (primary.length !== 1) {
      alerts.push(`CRITICAL: ${primary.length} PRIMARY members`)
    }

    // Check 2 : Health de tous les membres
    status.members.forEach(m => {
      if (m.health !== 1) {
        alerts.push(`WARNING: ${m.name} health = ${m.health}`)
      }
    })

    // Check 3 : Replication lag
    const now = status.date
    status.members.forEach(m => {
      if (m.state === 2 && m.optimeDate) {  // SECONDARY
        const lag = (now - m.optimeDate) / 1000
        if (lag > 60) {
          alerts.push(`WARNING: ${m.name} lag = ${lag}s`)
        }
      }
    })

    // Check 4 : Oplog window
    const replInfo = db.getReplicationInfo()
    const oplogHours = replInfo.timeDiff / 3600
    if (oplogHours < 24) {
      alerts.push(`WARNING: Oplog window = ${oplogHours}h`)
    }

    // Check 5 : Nombre de membres votants
    const cfg = rs.conf()
    const voting = cfg.members.filter(m => m.votes === 1).length
    if (voting % 2 === 0) {
      alerts.push(`WARNING: Even number of voters (${voting})`)
    }

    if (alerts.length > 0) {
      print(`\n=== ALERTS at ${new Date().toISOString()} ===`)
      alerts.forEach(a => print(a))
      // Envoyer notifications (email, Slack, PagerDuty, etc.)
    }
  }, 30000)  // Vérifier toutes les 30 secondes
}

// Démarrer le monitoring
continuousHealthCheck()
```

## Tests de Failover

### Test 1 : Arrêt Propre du Primary

```bash
# Objectif : Vérifier le failover contrôlé
# RTO attendu : ~12-20 secondes

# 1. Identifier le Primary
mongosh --eval "rs.isMaster().primary"
# Output: mongodb-01:27017

# 2. Timer de début
START_TIME=$(date +%s)

# 3. Arrêter proprement le Primary
ssh mongodb-01 "sudo systemctl stop mongod"

# 4. Observer l'élection depuis un Secondary
mongosh --host mongodb-02:27017 --eval "
  while(true) {
    var status = rs.status()
    var primary = status.members.find(m => m.state === 1)
    print(new Date().toISOString() + ' - Primary: ' + (primary ? primary.name : 'NONE'))
    if (primary && primary.name !== 'mongodb-01:27017') {
      print('NEW PRIMARY ELECTED: ' + primary.name)
      break
    }
    sleep(1000)
  }
"

# 5. Calculer le RTO
END_TIME=$(date +%s)
RTO=$((END_TIME - START_TIME))
echo "Recovery Time: ${RTO} seconds"

# 6. Redémarrer l'ancien Primary
ssh mongodb-01 "sudo systemctl start mongod"

# 7. Vérifier qu'il rejoint comme SECONDARY
mongosh --host mongodb-01:27017 --eval "rs.isMaster().secondary"
```

### Test 2 : Crash Brutal (kill -9)

```bash
# Objectif : Simuler un crash système
# RTO attendu : ~20-30 secondes

# 1. Identifier le PID de mongod sur le Primary
ssh mongodb-01 "pgrep mongod"
# Output: 12345

# 2. Timer
START_TIME=$(date +%s)

# 3. Kill brutal
ssh mongodb-01 "sudo kill -9 12345"

# 4. Observer failover...
# (même procédure que Test 1)

# 5. Analyser les logs
ssh mongodb-01 "tail -100 /var/log/mongodb/mongod.log"
# Rechercher : rollback, recovery
```

### Test 3 : Partition Réseau

```bash
# Objectif : Tester split-brain prevention
# Utilise iptables pour simuler partition réseau

# Configuration : 3 membres
# mongodb-01 (Primary)
# mongodb-02 (Secondary)
# mongodb-03 (Secondary)

# Scénario : Isoler mongodb-01

# 1. Sur mongodb-01, bloquer trafic vers autres membres
ssh mongodb-01 "
  sudo iptables -A INPUT -s 10.0.1.11 -j DROP   # mongodb-02
  sudo iptables -A INPUT -s 10.0.1.12 -j DROP   # mongodb-03
  sudo iptables -A OUTPUT -d 10.0.1.11 -j DROP
  sudo iptables -A OUTPUT -d 10.0.1.12 -j DROP
"

# 2. Observer depuis mongodb-02
mongosh --host mongodb-02:27017 --eval "
  // Attendre élection
  sleep(15000)

  var status = rs.status()
  print('New Primary: ' + rs.isMaster().primary)

  // mongodb-01 devrait être vu comme DOWN
  var old = status.members.find(m => m.name === 'mongodb-01:27017')
  print('mongodb-01 state: ' + old.stateStr)
"

# 3. Observer depuis mongodb-01 (partition A - minorité)
mongosh --host mongodb-01:27017 --eval "
  var status = rs.status()
  var me = status.members.find(m => m.self)
  print('My state: ' + me.stateStr)
  // Devrait être SECONDARY (ne peut pas rester Primary sans majorité)
"

# 4. Réparer la partition
ssh mongodb-01 "
  sudo iptables -F  # Flush toutes les règles
"

# 5. Observer la réconciliation
# mongodb-01 rejoint comme SECONDARY
# Possible rollback si des écritures non répliquées
```

### Test 4 : Failover en Charge

```javascript
// Objectif : Mesurer l'impact du failover sur les applications

// 1. Script de charge continue
const { MongoClient } = require('mongodb')

async function loadTest() {
  const client = new MongoClient('mongodb://mongodb-01,mongodb-02,mongodb-03/?replicaSet=rs0', {
    retryWrites: true,
    w: 'majority'
  })

  await client.connect()
  const db = client.db('test')
  const collection = db.collection('failover_test')

  let successCount = 0
  let errorCount = 0
  let errors = []

  const interval = setInterval(async () => {
    try {
      await collection.insertOne({
        timestamp: new Date(),
        data: Math.random()
      })
      successCount++
    } catch (error) {
      errorCount++
      errors.push({
        time: new Date(),
        error: error.message
      })
    }

    // Afficher stats toutes les 10 écritures
    if ((successCount + errorCount) % 10 === 0) {
      console.log(`Success: ${successCount}, Errors: ${errorCount}`)
    }
  }, 100)  // 10 écritures/seconde

  // Arrêter après 5 minutes
  setTimeout(() => {
    clearInterval(interval)
    console.log('\n=== Final Stats ===')
    console.log(`Total Success: ${successCount}`)
    console.log(`Total Errors: ${errorCount}`)
    console.log(`Error Rate: ${(errorCount / (successCount + errorCount) * 100).toFixed(2)}%`)

    if (errors.length > 0) {
      console.log('\n=== Error Timeline ===')
      errors.forEach(e => {
        console.log(`${e.time.toISOString()}: ${e.error}`)
      })
    }

    client.close()
  }, 300000)
}

// 2. Lancer le test
loadTest()

// 3. Pendant l'exécution, déclencher failover
// ssh mongodb-01 "sudo systemctl stop mongod"

// 4. Observer :
// - Période d'erreurs pendant failover (~10-30s)
// - Reprise automatique après élection
// - Aucune perte de données si w: majority
```

### Test 5 : Failover Multiple (Chaos Engineering)

```javascript
// Scénario : Défaillances en cascade

// Configuration : 5 membres
// Test sur 1 heure avec défaillances aléatoires

function chaosTest() {
  const members = [
    'mongodb-01:27017',
    'mongodb-02:27017',
    'mongodb-03:27017',
    'mongodb-04:27017',
    'mongodb-05:27017'
  ]

  const events = []

  // Toutes les 5-15 minutes, arrêter un membre aléatoire
  function randomFailure() {
    const randomDelay = (Math.random() * 600000) + 300000  // 5-15 min

    setTimeout(() => {
      const target = members[Math.floor(Math.random() * members.length)]
      const timestamp = new Date()

      console.log(`${timestamp.toISOString()}: Failing ${target}`)
      events.push({ time: timestamp, action: 'FAIL', target })

      // Arrêter le membre (via SSH ou API)
      // ssh <target> "sudo systemctl stop mongod"

      // Redémarrer après 2-5 minutes
      const recoveryDelay = (Math.random() * 180000) + 120000
      setTimeout(() => {
        const recoverTime = new Date()
        console.log(`${recoverTime.toISOString()}: Recovering ${target}`)
        events.push({ time: recoverTime, action: 'RECOVER', target })

        // ssh <target> "sudo systemctl start mongod"
      }, recoveryDelay)

      // Planifier prochaine défaillance
      randomFailure()
    }, randomDelay)
  }

  // Démarrer le chaos
  randomFailure()

  // Arrêter après 1 heure
  setTimeout(() => {
    console.log('\n=== Chaos Test Complete ===')
    console.log(`Total events: ${events.length}`)

    // Analyser la disponibilité
    // Vérifier que le système est resté opérationnel
  }, 3600000)
}
```

## Récupération et Rollback

### Processus de Rollback

Lorsqu'un ancien Primary redémarre avec des écritures non répliquées :

```
Scénario :
1. Primary (mongodb-01) écrit opération X (non répliquée)
2. Primary tombe avant réplication
3. Secondary (mongodb-02) élu Primary
4. Nouveau Primary écrit opération Y
5. Ancien Primary (mongodb-01) redémarre

Résultat :
mongodb-01 : [..., opX]
mongodb-02 : [..., opY]  (divergence)

Action : ROLLBACK
```

#### Détection du Rollback

```bash
# Logs MongoDB sur l'ancien Primary
[rollback] rollback started
[rollback] rollback common point: { ts: Timestamp(1705320000, 42), t: 42 }
[rollback] rollback end
[rollback] rollback files created in /data/db/rollback/
```

#### Fichiers de Rollback

```bash
# Emplacement
ls -lh /data/db/rollback/

# Exemple de contenu
2024-01-15T10-30-00.0.bson
2024-01-15T10-30-00.0.metadata.json
2024-01-15T10-30-00.0.removedDocs
```

#### Analyse des Rollbacks

```javascript
// Parser les fichiers rollback
const fs = require('fs')
const BSON = require('bson')

// Lire le fichier .bson
const rollbackData = fs.readFileSync('/data/db/rollback/2024-01-15T10-30-00.0.bson')
const documents = BSON.deserialize(rollbackData)

console.log('Rolled back documents:')
console.log(JSON.stringify(documents, null, 2))

// Décider si les données doivent être réappliquées manuellement
```

### Prévention des Rollbacks

#### Write Concern "majority"

```javascript
// ✅ Garantit que l'écriture est répliquée avant confirmation
db.orders.insertOne(
  { orderId: 12345, amount: 500 },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)

// Si le Primary tombe avant réplication sur majorité :
// → L'écriture échoue (erreur retournée)
// → Pas de rollback nécessaire
```

#### Configuration par Défaut

```javascript
// MongoDB 5.0+ : Définir w: majority par défaut
cfg = rs.conf()
cfg.settings.getLastErrorDefaults = {
  w: "majority",
  wtimeout: 5000
}
rs.reconfig(cfg)

// Toutes les écritures sans writeConcern explicite
// utiliseront w: majority
```

### Récupération après Rollback

```bash
# 1. Identifier les documents rollbackés
ls -lh /data/db/rollback/

# 2. Extraire et analyser
bsondump /data/db/rollback/2024-01-15T10-30-00.0.bson > rollback.json

# 3. Décision métier
# - Les données sont-elles critiques ?
# - Doivent-elles être réappliquées ?

# 4. Si réapplication nécessaire
mongoimport --host mongodb-primary \
  --db mydb \
  --collection orders \
  --file rollback.json

# 5. Vérifier l'intégrité
mongosh --eval "
  db.orders.find({ orderId: 12345 })
"

# 6. Archiver les fichiers rollback
mv /data/db/rollback/* /backup/rollback-archive/
```

## Optimisation du RTO/RPO

### Réduire le RTO (Recovery Time Objective)

#### 1. Optimiser electionTimeoutMillis

```javascript
cfg = rs.conf()

// Pour réseaux locaux/datacenter
cfg.settings.electionTimeoutMillis = 5000  // 5 secondes

// Pour réseaux WAN
cfg.settings.electionTimeoutMillis = 15000  // 15 secondes

rs.reconfig(cfg)
```

**Impact sur RTO** :
```
RTO ≈ electionTimeoutMillis + 2-3 secondes (élection)

electionTimeout = 5s  → RTO ≈ 7-8s
electionTimeout = 10s → RTO ≈ 12-13s
electionTimeout = 30s → RTO ≈ 32-33s
```

#### 2. Désactiver la Catchup Phase

```javascript
cfg = rs.conf()
cfg.settings.catchUpTimeoutMillis = 0  // Pas de catchup
rs.reconfig(cfg)
```

**Attention** : Risque de perte des dernières écritures non répliquées.

**Meilleure pratique** : Utiliser w: "majority" + catchupTimeout faible
```javascript
cfg.settings.catchUpTimeoutMillis = 5000  // 5 secondes max
```

#### 3. Priorités Optimisées

```javascript
// Membre haute performance comme Primary préféré
cfg.members[0].priority = 100  // SSD, plus de RAM, CPU
cfg.members[1].priority = 1
cfg.members[2].priority = 1

rs.reconfig(cfg)
```

#### 4. Retry Automatique dans les Drivers

```javascript
// Node.js
const client = new MongoClient(uri, {
  retryWrites: true,      // Retry écritures automatiquement
  retryReads: true,       // Retry lectures automatiquement
  serverSelectionTimeoutMS: 30000,  // 30s pour trouver un serveur
  socketTimeoutMS: 45000,
  connectTimeoutMS: 10000
})
```

### Réduire le RPO (Recovery Point Objective)

#### 1. Write Concern Majority

```javascript
// RPO = 0 (aucune perte de données)
db.collection.insertOne(doc, {
  writeConcern: { w: "majority", wtimeout: 5000 }
})
```

#### 2. Read Concern Majority

```javascript
// Lire uniquement les données répliquées sur majorité
db.collection.find({}, {
  readConcern: { level: "majority" }
})
```

#### 3. Journaling

```yaml
# mongod.conf
storage:
  journal:
    enabled: true
    commitIntervalMs: 100  # Commit journal toutes les 100ms
```

**RPO avec journaling** : ~100-300ms (durée du commit interval)

#### 4. Monitoring de la Réplication

```javascript
// Alerter si lag > seuil
function monitorReplicationLag(maxLagSeconds) {
  const status = rs.status()
  const alerts = []

  status.members.forEach(m => {
    if (m.state === 2 && m.optimeDate) {  // SECONDARY
      const lag = (status.date - m.optimeDate) / 1000
      if (lag > maxLagSeconds) {
        alerts.push({
          member: m.name,
          lag: lag,
          severity: lag > maxLagSeconds * 2 ? 'CRITICAL' : 'WARNING'
        })
      }
    }
  })

  return alerts
}

// Vérifier toutes les 30 secondes
setInterval(() => {
  const alerts = monitorReplicationLag(10)  // Seuil: 10s
  if (alerts.length > 0) {
    console.log('REPLICATION LAG ALERTS:', alerts)
    // Envoyer notification
  }
}, 30000)
```

## Bonnes Pratiques

### 1. Architecture

```javascript
// ✅ Nombre impair de votants
// ✅ Minimum 3 membres (idéalement 5)
// ✅ Distribution géographique si possible
{
  members: [
    { _id: 0, host: "dc1-01:27017", priority: 10, tags: {dc: "dc1"} },
    { _id: 1, host: "dc1-02:27017", priority: 9, tags: {dc: "dc1"} },
    { _id: 2, host: "dc1-03:27017", priority: 8, tags: {dc: "dc1"} },
    { _id: 3, host: "dc2-01:27017", priority: 1, tags: {dc: "dc2"} },
    { _id: 4, host: "dc2-02:27017", priority: 1, tags: {dc: "dc2"} }
  ]
}
```

### 2. Configuration

```javascript
// ✅ Write Concern par défaut
cfg.settings.getLastErrorDefaults = {
  w: "majority",
  wtimeout: 5000
}

// ✅ Election timeout adapté au réseau
cfg.settings.electionTimeoutMillis = 10000  // LAN
cfg.settings.electionTimeoutMillis = 30000  // WAN

// ✅ Oplog suffisant
db.adminCommand({ replSetResizeOplog: 1, size: 10240 })  // 10 GB
```

### 3. Monitoring

```javascript
// Métriques clés à surveiller
const metrics = {
  // Disponibilité
  'Primary présent': () => rs.status().members.filter(m => m.state === 1).length === 1,

  // Performance
  'Replication lag': () => {
    const status = rs.status()
    const lags = status.members
      .filter(m => m.state === 2)
      .map(m => (status.date - m.optimeDate) / 1000)
    return Math.max(...lags)
  },

  // Résilience
  'Oplog window (hours)': () => db.getReplicationInfo().timeDiff / 3600,

  // Santé
  'Membres UP': () => rs.status().members.filter(m => m.health === 1).length
}
```

### 4. Tests Réguliers

```bash
# Planifier des tests de failover trimestriels
# Documenter les résultats
# Mesurer RTO/RPO
# Valider les runbooks

# Exemple de calendrier
# T1 : Test de failover contrôlé
# T2 : Test de partition réseau
# T3 : Test de crash brutal
# T4 : Test de failover en charge
```

### 5. Documentation

```yaml
# Runbook Failover
version: 2.1
last_updated: 2024-01-15

scenarios:
  - name: "Primary Down"
    detection: "No primary in rs.status()"
    expected_rto: "15-30 seconds"
    expected_rpo: "0 (with w:majority)"
    actions:
      - "Verify automatic election occurred"
      - "Check application connectivity"
      - "Investigate cause of failure"
      - "Repair/replace failed member"

  - name: "Network Partition"
    detection: "Members showing as DOWN but actually running"
    expected_behavior: "Partition with majority continues"
    actions:
      - "Identify network issue"
      - "Verify no split-brain"
      - "Restore network connectivity"
      - "Verify data consistency"
```

## Conclusion

La haute disponibilité dans MongoDB repose sur :

- ✅ **Architecture résiliente** : Minimum 3 membres, idéalement 5+
- ✅ **Configuration optimisée** : Write concern majority, timeout adaptés
- ✅ **Monitoring proactif** : Détection rapide des anomalies
- ✅ **Tests réguliers** : Validation des procédures de failover
- ✅ **Documentation** : Runbooks à jour et testés

**Métriques clés** :
- **RTO** : 15-30 secondes (failover automatique)
- **RPO** : 0 (avec w: "majority")
- **Disponibilité** : 99.9% - 99.99%

Le failover automatique de MongoDB garantit la continuité de service en cas de défaillance, mais nécessite une conception architecturale appropriée et une validation régulière pour assurer la résilience en production.

⏭️ [Monitoring d'un Replica Set](/09-replication/10-monitoring-replica-set.md)
