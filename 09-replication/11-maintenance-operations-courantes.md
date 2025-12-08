🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.11 Maintenance et Opérations Courantes

## Introduction

La maintenance d'un Replica Set MongoDB est essentielle pour garantir les performances, la stabilité et la fiabilité du système en production. Ce chapitre couvre les opérations de maintenance courantes, les procédures recommandées, et les bonnes pratiques pour minimiser l'impact sur les applications tout en maintenant la haute disponibilité.

## Principes de Maintenance

### Rolling Maintenance

Le principe fondamental de maintenance d'un Replica Set est la **maintenance progressive (rolling)** qui permet de maintenir le service pendant les opérations.

```
Stratégie Rolling :
1. Maintenir toujours la majorité disponible
2. Un seul membre à la fois en maintenance
3. Commencer par les Secondary
4. Finir par le Primary
5. Valider après chaque étape
```

**Ordre recommandé** :
```
1. Secondary #1 (le moins prioritaire)
   ↓ Validation
2. Secondary #2
   ↓ Validation
3. Secondary #3 (si applicable)
   ↓ Validation
4. Primary (en dernier)
   ↓ Validation finale
```

### Fenêtres de Maintenance

```javascript
// Planification des fenêtres de maintenance
const maintenanceWindows = {
  // Maintenance régulière
  regular: {
    frequency: 'monthly',
    duration: '4 hours',
    preferredTime: '02:00-06:00 UTC',
    dayOfWeek: 'Sunday',
    impactLevel: 'low'
  },

  // Mises à jour critiques
  critical: {
    frequency: 'as-needed',
    duration: '2 hours',
    preferredTime: '02:00-04:00 UTC',
    dayOfWeek: 'any',
    impactLevel: 'medium'
  },

  // Upgrades majeurs
  major: {
    frequency: 'quarterly',
    duration: '8 hours',
    preferredTime: '00:00-08:00 UTC',
    dayOfWeek: 'Saturday',
    impactLevel: 'high'
  }
}
```

## Redémarrage des Membres

### Redémarrage d'un Secondary

#### Procédure Standard

```bash
#!/bin/bash
# restart-secondary.sh

MEMBER_HOST=$1
MEMBER_PORT=${2:-27017}

echo "=== Restarting Secondary: ${MEMBER_HOST}:${MEMBER_PORT} ==="

# 1. Vérifier que le membre est bien Secondary
echo "Step 1: Verifying member is Secondary..."
STATE=$(mongosh --host ${MEMBER_HOST}:${MEMBER_PORT} --quiet --eval "rs.isMaster().secondary")

if [ "$STATE" != "true" ]; then
  echo "ERROR: Member is not Secondary. Current state: $STATE"
  exit 1
fi

# 2. Vérifier la santé du Replica Set
echo "Step 2: Checking Replica Set health..."
PRIMARY=$(mongosh --host ${MEMBER_HOST}:${MEMBER_PORT} --quiet --eval "rs.isMaster().primary")
if [ -z "$PRIMARY" ]; then
  echo "ERROR: No Primary found in Replica Set"
  exit 1
fi

# 3. Arrêter MongoDB
echo "Step 3: Stopping MongoDB..."
ssh ${MEMBER_HOST} "sudo systemctl stop mongod"

if [ $? -ne 0 ]; then
  echo "ERROR: Failed to stop MongoDB"
  exit 1
fi

# 4. Attendre quelques secondes
echo "Step 4: Waiting for clean shutdown..."
sleep 5

# 5. Démarrer MongoDB
echo "Step 5: Starting MongoDB..."
ssh ${MEMBER_HOST} "sudo systemctl start mongod"

if [ $? -ne 0 ]; then
  echo "ERROR: Failed to start MongoDB"
  exit 1
fi

# 6. Attendre que le membre rejoigne le Replica Set
echo "Step 6: Waiting for member to rejoin..."
for i in {1..60}; do
  STATE=$(mongosh --host ${MEMBER_HOST}:${MEMBER_PORT} --quiet --eval "rs.status().ok" 2>/dev/null)
  if [ "$STATE" = "1" ]; then
    echo "Member rejoined successfully"
    break
  fi
  echo "  Waiting... ($i/60)"
  sleep 2
done

# 7. Vérifier l'état final
echo "Step 7: Verifying final state..."
mongosh --host ${MEMBER_HOST}:${MEMBER_PORT} --eval "
  var status = rs.status()
  var member = status.members.find(m => m.name === '${MEMBER_HOST}:${MEMBER_PORT}')
  print('State: ' + member.stateStr)
  print('Health: ' + member.health)
"

# 8. Vérifier le replication lag
echo "Step 8: Checking replication lag..."
sleep 10
mongosh --host ${PRIMARY} --eval "rs.printSecondaryReplicationInfo()"

echo "=== Restart completed ==="
```

#### Validation Post-Redémarrage

```javascript
// Validation après redémarrage d'un Secondary
function validateSecondaryRestart(memberHost) {
  const status = rs.status()
  const member = status.members.find(m => m.name === memberHost)

  const checks = {
    timestamp: new Date(),
    member: memberHost,

    // Check 1: État
    state: {
      value: member.stateStr,
      expected: 'SECONDARY',
      passed: member.state === 2
    },

    // Check 2: Santé
    health: {
      value: member.health,
      expected: 1,
      passed: member.health === 1
    },

    // Check 3: Replication lag
    lag: {
      value: member.optimeDate ? (status.date - member.optimeDate) / 1000 : null,
      expected: '< 30s',
      passed: member.optimeDate ? (status.date - member.optimeDate) / 1000 < 30 : false
    },

    // Check 4: Uptime
    uptime: {
      value: member.uptime,
      expected: '> 0',
      passed: member.uptime > 0
    },

    // Check 5: Sync source
    syncSource: {
      value: member.syncSourceHost,
      expected: 'present',
      passed: member.syncSourceHost !== undefined
    }
  }

  const allPassed = Object.values(checks)
    .filter(v => typeof v === 'object' && 'passed' in v)
    .every(check => check.passed)

  return {
    success: allPassed,
    checks: checks,
    summary: allPassed ? 'All checks passed' : 'Some checks failed'
  }
}

// Utilisation
const validation = validateSecondaryRestart('mongodb-02:27017')
printjson(validation)
```

### Redémarrage du Primary

#### Procédure avec Step Down

```bash
#!/bin/bash
# restart-primary.sh

PRIMARY_HOST=$1
PRIMARY_PORT=${2:-27017}

echo "=== Restarting Primary: ${PRIMARY_HOST}:${PRIMARY_PORT} ==="

# 1. Vérifier que le membre est bien Primary
echo "Step 1: Verifying member is Primary..."
STATE=$(mongosh --host ${PRIMARY_HOST}:${PRIMARY_PORT} --quiet --eval "rs.isMaster().ismaster")

if [ "$STATE" != "true" ]; then
  echo "ERROR: Member is not Primary"
  exit 1
fi

# 2. Vérifier la santé du Replica Set
echo "Step 2: Checking Replica Set health..."
SECONDARY_COUNT=$(mongosh --host ${PRIMARY_HOST}:${PRIMARY_PORT} --quiet --eval "
  rs.status().members.filter(m => m.state === 2).length
")

if [ "$SECONDARY_COUNT" -lt 1 ]; then
  echo "ERROR: No healthy Secondary found"
  exit 1
fi

# 3. Step down du Primary
echo "Step 3: Stepping down Primary..."
mongosh --host ${PRIMARY_HOST}:${PRIMARY_PORT} --eval "rs.stepDown(60)"

if [ $? -ne 0 ]; then
  echo "ERROR: Failed to step down Primary"
  exit 1
fi

# 4. Attendre l'élection du nouveau Primary
echo "Step 4: Waiting for new Primary election..."
for i in {1..30}; do
  NEW_PRIMARY=$(mongosh --host ${PRIMARY_HOST}:${PRIMARY_PORT} --quiet --eval "rs.isMaster().primary" 2>/dev/null)
  if [ -n "$NEW_PRIMARY" ] && [ "$NEW_PRIMARY" != "${PRIMARY_HOST}:${PRIMARY_PORT}" ]; then
    echo "New Primary elected: $NEW_PRIMARY"
    break
  fi
  echo "  Waiting for election... ($i/30)"
  sleep 2
done

# 5. Vérifier que l'ancien Primary est maintenant Secondary
echo "Step 5: Verifying old Primary is now Secondary..."
IS_SECONDARY=$(mongosh --host ${PRIMARY_HOST}:${PRIMARY_PORT} --quiet --eval "rs.isMaster().secondary")

if [ "$IS_SECONDARY" != "true" ]; then
  echo "ERROR: Old Primary did not become Secondary"
  exit 1
fi

# 6. Arrêter MongoDB
echo "Step 6: Stopping MongoDB..."
ssh ${PRIMARY_HOST} "sudo systemctl stop mongod"

# 7. Effectuer la maintenance
echo "Step 7: Performing maintenance..."
# Insérer ici les opérations de maintenance spécifiques
sleep 5

# 8. Redémarrer MongoDB
echo "Step 8: Starting MongoDB..."
ssh ${PRIMARY_HOST} "sudo systemctl start mongod"

# 9. Attendre que le membre rejoigne
echo "Step 9: Waiting for member to rejoin as Secondary..."
for i in {1..60}; do
  STATE=$(mongosh --host ${PRIMARY_HOST}:${PRIMARY_PORT} --quiet --eval "rs.status().ok" 2>/dev/null)
  if [ "$STATE" = "1" ]; then
    echo "Member rejoined successfully"
    break
  fi
  echo "  Waiting... ($i/60)"
  sleep 2
done

# 10. Optionnel : Restaurer comme Primary
echo "Step 10: Optional - Restore as Primary? (y/N)"
read -t 30 RESTORE
if [ "$RESTORE" = "y" ]; then
  echo "Configuring member to become Primary again..."
  mongosh --host ${NEW_PRIMARY} --eval "
    cfg = rs.conf()
    cfg.members.find(m => m.host === '${PRIMARY_HOST}:${PRIMARY_PORT}').priority = 10
    rs.reconfig(cfg)
  "
  echo "Member will become Primary after priority takeover"
fi

echo "=== Primary restart completed ==="
```

### Redémarrage d'Urgence (Force)

```javascript
// Redémarrage forcé en cas de problème
// ATTENTION : À utiliser uniquement en dernier recours

function forceRestartMember(memberHost) {
  print(`WARNING: Force restarting ${memberHost}`)
  print('This may cause data inconsistencies')
  print('Press Ctrl+C within 10 seconds to abort...')

  sleep(10000)

  // 1. Kill le processus
  print('Killing mongod process...')
  // ssh memberHost "sudo pkill -9 mongod"

  // 2. Vérifier le filesystem
  print('Checking filesystem...')
  // ssh memberHost "sudo xfs_repair -n /data/mongodb" (pour XFS)

  // 3. Démarrer en mode standalone si nécessaire
  print('Starting in standalone mode for repair...')
  // ssh memberHost "mongod --repair --dbpath /data/mongodb"

  // 4. Redémarrer normalement
  print('Restarting in replica set mode...')
  // ssh memberHost "sudo systemctl start mongod"

  print('Force restart completed. Verify data integrity!')
}
```

## Mises à Jour et Upgrades

### Upgrade de Version MongoDB

#### Préparation

```bash
#!/bin/bash
# prepare-upgrade.sh

echo "=== MongoDB Upgrade Preparation ==="

# 1. Vérifier la version actuelle
echo "Current version:"
mongosh --eval "db.version()"

# 2. Vérifier la compatibilité
echo "Checking feature compatibility version:"
mongosh --eval "db.adminCommand({getParameter: 1, featureCompatibilityVersion: 1})"

# 3. Backup de la configuration
echo "Backing up configuration..."
mongosh --eval "printjson(rs.conf())" > rs-config-backup-$(date +%Y%m%d).json

# 4. Backup des données (optionnel mais recommandé)
echo "Creating backup..."
mongodump --out=/backup/pre-upgrade-$(date +%Y%m%d)

# 5. Vérifier l'espace disque
echo "Checking disk space..."
df -h /data/mongodb
df -h /backup

# 6. Documenter l'état actuel
echo "Documenting current state..."
mongosh --eval "
  printjson({
    version: db.version(),
    fcv: db.adminCommand({getParameter: 1, featureCompatibilityVersion: 1}),
    members: rs.status().members.map(m => ({name: m.name, state: m.stateStr})),
    databases: db.adminCommand('listDatabases').databases.map(d => d.name),
    timestamp: new Date()
  })
" > pre-upgrade-state-$(date +%Y%m%d).json

echo "=== Preparation completed ==="
```

#### Procédure d'Upgrade Rolling

```bash
#!/bin/bash
# rolling-upgrade.sh

OLD_VERSION=$1
NEW_VERSION=$2

echo "=== Rolling Upgrade: $OLD_VERSION -> $NEW_VERSION ==="

# Membres du Replica Set
SECONDARY_1="mongodb-02:27017"
SECONDARY_2="mongodb-03:27017"
PRIMARY="mongodb-01:27017"

# Fonction d'upgrade d'un membre
upgrade_member() {
  local MEMBER=$1
  local HOST=$(echo $MEMBER | cut -d: -f1)

  echo "=== Upgrading $MEMBER ==="

  # 1. Arrêter MongoDB
  echo "Stopping MongoDB on $HOST..."
  ssh $HOST "sudo systemctl stop mongod"

  # 2. Installer nouvelle version
  echo "Installing MongoDB $NEW_VERSION..."
  ssh $HOST "
    sudo apt-get update
    sudo apt-get install -y mongodb-org=$NEW_VERSION \
                            mongodb-org-server=$NEW_VERSION \
                            mongodb-org-shell=$NEW_VERSION \
                            mongodb-org-mongos=$NEW_VERSION \
                            mongodb-org-tools=$NEW_VERSION
  "

  # 3. Redémarrer
  echo "Starting MongoDB..."
  ssh $HOST "sudo systemctl start mongod"

  # 4. Attendre que le membre soit opérationnel
  echo "Waiting for member to be healthy..."
  for i in {1..60}; do
    STATE=$(mongosh --host $MEMBER --quiet --eval "rs.status().ok" 2>/dev/null)
    if [ "$STATE" = "1" ]; then
      echo "Member is healthy"
      break
    fi
    sleep 2
  done

  # 5. Vérifier la version
  echo "Verifying version..."
  mongosh --host $MEMBER --eval "db.version()"

  # 6. Attendre stabilisation
  echo "Waiting for replication to stabilize..."
  sleep 30

  echo "=== Upgrade of $MEMBER completed ==="
}

# Phase 1 : Upgrader les Secondary
echo "Phase 1: Upgrading Secondary members..."
upgrade_member $SECONDARY_1
upgrade_member $SECONDARY_2

# Phase 2 : Step down et upgrader le Primary
echo "Phase 2: Upgrading Primary..."
echo "Stepping down Primary..."
mongosh --host $PRIMARY --eval "rs.stepDown(60)"

sleep 15

# Trouver le nouveau Primary
NEW_PRIMARY=$(mongosh --host $SECONDARY_1 --quiet --eval "rs.isMaster().primary")
echo "New Primary: $NEW_PRIMARY"

# Upgrader l'ancien Primary (maintenant Secondary)
upgrade_member $PRIMARY

# Phase 3 : Mettre à jour le Feature Compatibility Version
echo "Phase 3: Updating Feature Compatibility Version..."
mongosh --host $NEW_PRIMARY --eval "
  db.adminCommand({setFeatureCompatibilityVersion: '$NEW_VERSION'})
"

# Phase 4 : Validation finale
echo "Phase 4: Final validation..."
mongosh --host $NEW_PRIMARY --eval "
  print('=== Upgrade Summary ===')
  rs.status().members.forEach(m => {
    print(m.name + ': ' + m.stateStr)
  })

  print('\nFeature Compatibility Version:')
  printjson(db.adminCommand({getParameter: 1, featureCompatibilityVersion: 1}))
"

echo "=== Rolling Upgrade completed ==="
```

#### Rollback d'Upgrade

```bash
#!/bin/bash
# rollback-upgrade.sh

echo "=== Rolling Back Upgrade ==="

# 1. Vérifier que le rollback est possible
echo "Checking if rollback is possible..."
mongosh --eval "
  var fcv = db.adminCommand({getParameter: 1, featureCompatibilityVersion: 1})
  if (fcv.featureCompatibilityVersion.version !== '6.0') {
    print('ERROR: Cannot rollback - FCV already upgraded')
    quit(1)
  }
"

# 2. Downgrader chaque membre
# (Même procédure que upgrade mais avec ancienne version)

# 3. NE PAS changer le FCV si déjà modifié
echo "WARNING: If FCV was already upgraded, full rollback is not possible"
echo "You may need to restore from backup"
```

### Patch de Sécurité

```bash
#!/bin/bash
# apply-security-patch.sh

PATCH_VERSION=$1

echo "=== Applying Security Patch: $PATCH_VERSION ==="

# 1. Vérifier que c'est un patch mineur
CURRENT=$(mongosh --quiet --eval "db.version()")
echo "Current version: $CURRENT"
echo "Patch version: $PATCH_VERSION"

# Vérifier que les 2 premiers numéros sont identiques (ex: 6.0.x -> 6.0.y)
CURRENT_MAJOR=$(echo $CURRENT | cut -d. -f1-2)
PATCH_MAJOR=$(echo $PATCH_VERSION | cut -d. -f1-2)

if [ "$CURRENT_MAJOR" != "$PATCH_MAJOR" ]; then
  echo "ERROR: This is not a patch - major/minor version change detected"
  exit 1
fi

# 2. Appliquer le patch (rolling)
# (Utiliser la même procédure que rolling-upgrade.sh)

echo "=== Security Patch applied ==="
```

## Maintenance du Système de Fichiers

### Compaction

La compaction récupère l'espace disque fragmenté.

#### Compaction sur Secondary

```javascript
// Compaction sur un Secondary
// Méthode 1 : Commande compact (peut être lente)
db.runCommand({ compact: 'collectionName' })

// Méthode 2 : Resync initial (plus rapide pour grandes collections)
// 1. Retirer le membre du Replica Set
// 2. Supprimer les données
// 3. Rajouter le membre (initial sync automatique)
```

**Procédure complète** :

```bash
#!/bin/bash
# compact-secondary.sh

SECONDARY_HOST=$1
SECONDARY_PORT=${2:-27017}

echo "=== Compacting Secondary: ${SECONDARY_HOST}:${SECONDARY_PORT} ==="

# Option 1 : Compact en place (conserve les données)
echo "Option 1: In-place compaction"
mongosh --host ${SECONDARY_HOST}:${SECONDARY_PORT} --eval "
  var dbs = db.adminCommand('listDatabases').databases
  dbs.forEach(database => {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      print('Compacting database: ' + database.name)
      db = db.getSiblingDB(database.name)
      db.getCollectionNames().forEach(collection => {
        print('  Compacting collection: ' + collection)
        var result = db.runCommand({ compact: collection, force: true })
        printjson(result)
      })
    }
  })
"

# Option 2 : Resync (plus rapide, nécessite espace pour initial sync)
echo "Option 2: Resync (use with caution)"
echo "This will remove the member, delete data, and resync"
echo "Press Ctrl+C to abort, or wait 10 seconds to continue..."
sleep 10

# Retirer du RS
mongosh --eval "rs.remove('${SECONDARY_HOST}:${SECONDARY_PORT}')"

# Arrêter MongoDB
ssh ${SECONDARY_HOST} "sudo systemctl stop mongod"

# Supprimer les données
ssh ${SECONDARY_HOST} "sudo rm -rf /data/mongodb/*"

# Redémarrer
ssh ${SECONDARY_HOST} "sudo systemctl start mongod"

# Rajouter au RS
mongosh --eval "rs.add('${SECONDARY_HOST}:${SECONDARY_PORT}')"

echo "Member will perform initial sync and rejoin the replica set"
```

### Défragmentation du Filesystem

```bash
#!/bin/bash
# defragment-filesystem.sh

MEMBER_HOST=$1

echo "=== Filesystem Defragmentation: $MEMBER_HOST ==="

# Pour XFS (recommandé pour MongoDB)
ssh $MEMBER_HOST "
  # Vérifier la fragmentation
  sudo xfs_db -c frag -r /dev/sda1

  # Défragmenter (à faire en dehors des heures de production)
  sudo xfs_fsr /data/mongodb
"

# Pour Ext4
ssh $MEMBER_HOST "
  # Vérifier la fragmentation
  sudo e4defrag -c /data/mongodb

  # Défragmenter
  sudo e4defrag /data/mongodb
"
```

## Gestion des Index

### Construction d'Index en Production

```javascript
// Construction d'index sans bloquer les écritures
// Méthode Rolling Index Build

// Étape 1 : Créer l'index sur tous les Secondary
// Se connecter à chaque Secondary
db.collection.createIndex(
  { field: 1 },
  { background: true }  // Deprecated depuis 4.2, mais maintenu pour compat
)

// Étape 2 : Attendre que l'index soit complet sur tous les Secondary
rs.printSecondaryReplicationInfo()

// Étape 3 : Step down du Primary
rs.stepDown(60)

// Étape 4 : Créer l'index sur l'ancien Primary (maintenant Secondary)
db.collection.createIndex({ field: 1 })
```

**Script automatisé** :

```javascript
// rolling-index-build.js

function rollingIndexBuild(collectionName, indexSpec, options = {}) {
  const status = rs.status()
  const config = rs.conf()

  print(`=== Rolling Index Build ===`)
  print(`Collection: ${collectionName}`)
  print(`Index: ${JSON.stringify(indexSpec)}`)

  // Phase 1 : Identifier les membres
  const primary = status.members.find(m => m.state === 1)
  const secondaries = status.members.filter(m => m.state === 2)

  print(`\nPrimary: ${primary.name}`)
  print(`Secondaries: ${secondaries.map(s => s.name).join(', ')}`)

  // Phase 2 : Créer l'index sur chaque Secondary
  print(`\nPhase 1: Building index on Secondaries...`)

  secondaries.forEach(secondary => {
    print(`\nBuilding on ${secondary.name}...`)

    const conn = new Mongo(secondary.name)
    const db = conn.getDB(db.getName())

    try {
      const result = db[collectionName].createIndex(indexSpec, options)
      print(`  Result: ${JSON.stringify(result)}`)
    } catch (e) {
      print(`  ERROR: ${e.message}`)
      return
    }
  })

  // Phase 3 : Attendre que les index soient complets
  print(`\nPhase 2: Waiting for index builds to complete...`)

  let allComplete = false
  while (!allComplete) {
    allComplete = true

    secondaries.forEach(secondary => {
      const conn = new Mongo(secondary.name)
      const db = conn.getDB(db.getName())
      const inprog = db.currentOp({
        $or: [
          { op: "command", "command.createIndexes": { $exists: true } },
          { op: "insert", ns: /\.system\.indexes$/ }
        ]
      })

      if (inprog.inprog.length > 0) {
        print(`  ${secondary.name}: Still building...`)
        allComplete = false
      }
    })

    if (!allComplete) {
      sleep(5000)
    }
  }

  print(`All Secondaries completed index build`)

  // Phase 4 : Step down du Primary
  print(`\nPhase 3: Stepping down Primary...`)
  rs.stepDown(60)

  sleep(15000)

  // Phase 5 : Construire sur l'ancien Primary
  print(`\nPhase 4: Building index on old Primary (now Secondary)...`)

  const conn = new Mongo(primary.name)
  const db = conn.getDB(db.getName())
  const result = db[collectionName].createIndex(indexSpec, options)

  print(`Result: ${JSON.stringify(result)}`)
  print(`\n=== Rolling Index Build Completed ===`)
}

// Utilisation
rollingIndexBuild('users', { email: 1 }, { unique: true })
```

### Suppression d'Index

```javascript
// Suppression d'index (safe - pas de rolling nécessaire)
db.collection.dropIndex({ field: 1 })

// Vérifier l'impact avant suppression
db.collection.explain().find({ field: value })

// Si l'index n'est plus utilisé, supprimer
db.collection.dropIndex('index_name')

// Supprimer tous les index sauf _id
db.collection.dropIndexes()
```

### Rebuild des Index

```javascript
// Rebuild tous les index d'une collection
db.collection.reIndex()

// ATTENTION : Bloque les lectures/écritures
// Préférer rolling rebuild :

// 1. Noter les index existants
const indexes = db.collection.getIndexes()

// 2. Pour chaque Secondary :
//    - Supprimer les index
//    - Recréer les index

// 3. Step down Primary et répéter
```

## Nettoyage et Optimisation

### Nettoyage de l'Oplog

```javascript
// L'oplog se nettoie automatiquement (capped collection)
// Mais on peut le redimensionner si nécessaire

// Agrandir l'oplog
db.adminCommand({ replSetResizeOplog: 1, size: 20480 })  // 20 GB

// Vérifier l'utilisation
rs.printReplicationInfo()
```

### Suppression des Anciennes Données

```javascript
// Supprimer les anciennes données par batch
function deleteOldDataByBatch(collection, dateField, cutoffDate, batchSize = 1000) {
  let totalDeleted = 0
  let batchCount = 0

  while (true) {
    const result = db[collection].deleteMany(
      { [dateField]: { $lt: cutoffDate } },
      {
        maxTimeMS: 5000,
        writeConcern: { w: 'majority' }
      }
    )

    totalDeleted += result.deletedCount
    batchCount++

    print(`Batch ${batchCount}: Deleted ${result.deletedCount} documents (Total: ${totalDeleted})`)

    if (result.deletedCount === 0) {
      break
    }

    // Pause entre les batches pour ne pas surcharger
    sleep(1000)
  }

  print(`\nTotal deleted: ${totalDeleted} documents in ${batchCount} batches`)
  return totalDeleted
}

// Utilisation
const cutoffDate = new Date('2023-01-01')
deleteOldDataByBatch('logs', 'timestamp', cutoffDate)
```

### Nettoyage des Orphelins (Sharded Clusters)

```javascript
// Sur un cluster shardé, nettoyer les documents orphelins
db.adminCommand({ cleanupOrphaned: 'dbName.collectionName' })

// Automated cleanup
function cleanupAllOrphans() {
  const databases = db.adminCommand('listDatabases').databases

  databases.forEach(database => {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      const db = db.getSiblingDB(database.name)
      const collections = db.getCollectionNames()

      collections.forEach(collection => {
        print(`Cleaning orphans in ${database.name}.${collection}`)
        try {
          const result = db.adminCommand({
            cleanupOrphaned: `${database.name}.${collection}`
          })
          printjson(result)
        } catch (e) {
          print(`Error: ${e.message}`)
        }
      })
    }
  })
}
```

## Rotation des Logs

### Configuration de la Rotation

```yaml
# mongod.conf
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen  # ou 'rename'
```

### Rotation Manuelle

```bash
#!/bin/bash
# rotate-mongodb-logs.sh

echo "=== MongoDB Log Rotation ==="

# Méthode 1 : Avec logRotate reopen
mongosh --eval "db.adminCommand({ logRotate: 1 })"

# Méthode 2 : Script personnalisé
LOG_DIR="/var/log/mongodb"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

# Renommer le log actuel
mv ${LOG_DIR}/mongod.log ${LOG_DIR}/mongod.log.${TIMESTAMP}

# Signal MongoDB pour réouvrir le log
mongosh --eval "db.adminCommand({ logRotate: 1 })"

# Comprésser l'ancien log
gzip ${LOG_DIR}/mongod.log.${TIMESTAMP}

# Supprimer les logs de plus de 30 jours
find ${LOG_DIR} -name "mongod.log.*.gz" -mtime +30 -delete

echo "Log rotation completed"
```

### Logrotate System Integration

```bash
# /etc/logrotate.d/mongodb

/var/log/mongodb/mongod.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    sharedscripts
    postrotate
        /bin/kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid) 2>/dev/null || true
    endscript
}
```

## Monitoring de la Maintenance

### Checklist Pré-Maintenance

```javascript
// Pre-maintenance checklist
function preMaintenanceChecklist() {
  const checks = []

  // 1. Replica Set Health
  const status = rs.status()
  checks.push({
    name: 'Replica Set Health',
    status: status.ok === 1 ? 'PASS' : 'FAIL',
    details: `${status.members.filter(m => m.health === 1).length}/${status.members.length} members healthy`
  })

  // 2. Primary Present
  const primaryCount = status.members.filter(m => m.state === 1).length
  checks.push({
    name: 'Primary Present',
    status: primaryCount === 1 ? 'PASS' : 'FAIL',
    details: `${primaryCount} Primary member(s)`
  })

  // 3. Replication Lag
  const maxLag = Math.max(...status.members
    .filter(m => m.state === 2 && m.optimeDate)
    .map(m => (status.date - m.optimeDate) / 1000))

  checks.push({
    name: 'Replication Lag',
    status: maxLag < 60 ? 'PASS' : 'WARN',
    details: `Max lag: ${maxLag.toFixed(2)}s`
  })

  // 4. Oplog Window
  const replInfo = db.getReplicationInfo()
  const oplogHours = replInfo.timeDiff / 3600

  checks.push({
    name: 'Oplog Window',
    status: oplogHours >= 24 ? 'PASS' : 'WARN',
    details: `${oplogHours.toFixed(2)} hours`
  })

  // 5. Disk Space
  const dbStats = db.stats()
  const diskUsagePercent = (dbStats.dataSize / dbStats.fsUsedSize) * 100

  checks.push({
    name: 'Disk Space',
    status: diskUsagePercent < 80 ? 'PASS' : 'WARN',
    details: `${diskUsagePercent.toFixed(2)}% used`
  })

  // 6. Backup Status
  // Vérifier qu'un backup récent existe
  checks.push({
    name: 'Recent Backup',
    status: 'MANUAL',
    details: 'Verify backup exists < 24h old'
  })

  // Résumé
  const passed = checks.filter(c => c.status === 'PASS').length
  const warnings = checks.filter(c => c.status === 'WARN').length
  const failed = checks.filter(c => c.status === 'FAIL').length

  print('=== Pre-Maintenance Checklist ===')
  checks.forEach(check => {
    const icon = check.status === 'PASS' ? '✓' :
                 check.status === 'WARN' ? '⚠' :
                 check.status === 'FAIL' ? '✗' : '○'
    print(`${icon} ${check.name}: ${check.details}`)
  })

  print(`\nSummary: ${passed} passed, ${warnings} warnings, ${failed} failed`)

  const canProceed = failed === 0
  print(`\nCan proceed with maintenance: ${canProceed ? 'YES' : 'NO'}`)

  return {
    canProceed: canProceed,
    checks: checks,
    summary: { passed, warnings, failed }
  }
}

// Exécuter
preMaintenanceChecklist()
```

### Monitoring Durant la Maintenance

```javascript
// Monitoring continu durant maintenance
function monitorDuringMaintenance(intervalSeconds = 30) {
  print('=== Starting Maintenance Monitoring ===')
  print('Press Ctrl+C to stop\n')

  const startTime = new Date()

  const monitorInterval = setInterval(() => {
    const status = rs.status()
    const serverStatus = db.serverStatus()

    const elapsed = (new Date() - startTime) / 1000 / 60  // minutes

    print(`\n=== ${new Date().toISOString()} (${elapsed.toFixed(1)} min) ===`)

    // État des membres
    print('Members:')
    status.members.forEach(m => {
      const lag = m.optimeDate ? ((status.date - m.optimeDate) / 1000).toFixed(1) : 'N/A'
      print(`  ${m.name}: ${m.stateStr} (lag: ${lag}s, health: ${m.health})`)
    })

    // Connexions
    print(`\nConnections: ${serverStatus.connections.current} / ${serverStatus.connections.available + serverStatus.connections.current}`)

    // Opérations
    print(`Operations: ${serverStatus.opcounters.insert + serverStatus.opcounters.query + serverStatus.opcounters.update + serverStatus.opcounters.delete}/sec`)

  }, intervalSeconds * 1000)

  return monitorInterval
}

// Utilisation
const monitor = monitorDuringMaintenance(30)

// Pour arrêter : clearInterval(monitor)
```

## Procédures d'Urgence

### Récupération après Crash

```bash
#!/bin/bash
# emergency-recovery.sh

MEMBER_HOST=$1

echo "=== Emergency Recovery: $MEMBER_HOST ==="

# 1. Vérifier l'état du processus
echo "Step 1: Checking process status..."
ssh $MEMBER_HOST "ps aux | grep mongod"

# 2. Vérifier les logs
echo "Step 2: Checking logs for errors..."
ssh $MEMBER_HOST "tail -100 /var/log/mongodb/mongod.log"

# 3. Vérifier l'intégrité du filesystem
echo "Step 3: Checking filesystem..."
ssh $MEMBER_HOST "sudo xfs_repair -n /dev/sda1" # ou ext4

# 4. Tenter un démarrage normal
echo "Step 4: Attempting normal start..."
ssh $MEMBER_HOST "sudo systemctl start mongod"

sleep 10

# 5. Si échec, tenter avec --repair
if ! ssh $MEMBER_HOST "systemctl is-active mongod"; then
  echo "Step 5: Normal start failed. Attempting repair..."

  ssh $MEMBER_HOST "
    sudo systemctl stop mongod
    sudo -u mongodb mongod --repair --dbpath /data/mongodb
    sudo systemctl start mongod
  "
fi

# 6. Si toujours échec, resync depuis backup ou autre membre
if ! ssh $MEMBER_HOST "systemctl is-active mongod"; then
  echo "Step 6: Repair failed. Manual intervention required."
  echo "Options:"
  echo "  1. Restore from backup"
  echo "  2. Remove from RS and resync"
  echo "  3. Contact support"
  exit 1
fi

echo "=== Recovery completed ==="
```

### Procédure de Rollback Manuel

```javascript
// Rollback manuel en cas de problème
// ATTENTION : Procédure d'urgence uniquement

function manualRollback(memberHost) {
  print(`WARNING: Manual rollback on ${memberHost}`)
  print('This is an emergency procedure')

  // 1. Identifier les fichiers rollback
  print('Step 1: Identifying rollback files...')
  // ls /data/db/rollback/

  // 2. Analyser les documents rollbackés
  print('Step 2: Analyzing rollback documents...')
  // bsondump /data/db/rollback/*.bson

  // 3. Décision : restaurer ou ignorer
  print('Step 3: Decision required')
  print('  - Review rollback documents')
  print('  - Determine if data should be restored')
  print('  - If restore needed, manually insert into database')

  // 4. Nettoyer les fichiers rollback
  print('Step 4: After review, archive rollback files')
  // mv /data/db/rollback/* /backup/rollback-archive/
}
```

## Documentation et Runbooks

### Template de Runbook

```markdown
# Runbook: [Nom de l'Opération]

## Informations Générales
- **Opération**: [Description]
- **Fréquence**: [Quotidien/Hebdomadaire/Mensuel/Ad-hoc]
- **Durée estimée**: [X minutes/heures]
- **Impact**: [Aucun/Faible/Moyen/Élevé]
- **Fenêtre recommandée**: [HH:MM - HH:MM UTC]

## Prérequis
- [ ] Backup récent < 24h
- [ ] Replica Set healthy
- [ ] Oplog window >= 24h
- [ ] Replication lag < 30s
- [ ] Notification équipe envoyée

## Procédure

### Phase 1: Préparation
1. Vérifier l'état du Replica Set
   ```bash
   mongosh --eval "rs.status()"
   ```

2. Exécuter checklist pré-maintenance
   ```bash
   ./pre-maintenance-checklist.sh
   ```

3. [Autres étapes de préparation]

### Phase 2: Exécution
1. [Étape 1]
   ```bash
   [commande]
   ```
   **Résultat attendu**: [description]
   **Si erreur**: [action]

2. [Étape 2]
   ...

### Phase 3: Validation
1. Vérifier l'état final
2. Vérifier les métriques
3. Valider avec l'application

### Phase 4: Nettoyage
1. Archiver les logs
2. Documenter les incidents
3. Mettre à jour la documentation

## Rollback
En cas de problème :
1. [Étape de rollback 1]
2. [Étape de rollback 2]

## Contacts
- **Équipe DBA**: [email/slack]
- **On-call**: [numéro]
- **Escalade**: [manager]

## Historique
| Date | Opérateur | Résultat | Notes |
|------|-----------|----------|-------|
| YYYY-MM-DD | [nom] | [OK/KO] | [commentaires] |
```

### Checklist de Maintenance Mensuelle

```yaml
# monthly-maintenance-checklist.yml

monthly_maintenance:
  date: "First Sunday of month"
  time: "02:00-06:00 UTC"

  pre_tasks:
    - verify_backup_exists:
        age_max_hours: 24
        action: "Verify backup < 24h old"

    - check_replica_set:
        action: "Run rs.status() and verify health"

    - check_disk_space:
        threshold: 80
        action: "Verify disk usage < 80%"

    - notify_team:
        channels: ["#ops", "#mongodb"]
        message: "Monthly maintenance starting"

  maintenance_tasks:
    - rotate_logs:
        action: "Rotate and compress logs"
        script: "./rotate-logs.sh"

    - analyze_slow_queries:
        action: "Review slow query logs"
        threshold_ms: 100

    - review_index_usage:
        action: "Identify unused indexes"
        script: "./analyze-indexes.js"

    - update_statistics:
        action: "Run db.stats() on all databases"

    - check_oplog_size:
        action: "Verify oplog window >= 48h"
        min_hours: 48

    - compact_if_needed:
        action: "Compact collections with >20% fragmentation"
        threshold: 0.2

  post_tasks:
    - verify_health:
        action: "Run post-maintenance checklist"

    - update_documentation:
        action: "Document any issues encountered"

    - notify_completion:
        channels: ["#ops", "#mongodb"]
        message: "Monthly maintenance completed"
```

## Bonnes Pratiques

### 1. Planification

```javascript
const maintenanceBestPractices = {
  planning: [
    'Planifier pendant les heures creuses',
    'Notification 48-72h à l\'avance',
    'Avoir un plan de rollback',
    'Tester d\'abord en staging',
    'Documenter chaque étape'
  ],

  execution: [
    'Toujours commencer par les Secondary',
    'Valider après chaque étape',
    'Maintenir la majorité disponible',
    'Monitorer en continu',
    'Garder une trace de toutes les commandes'
  ],

  communication: [
    'Notifier avant/pendant/après',
    'Canal de communication dédié',
    'Escalation path claire',
    'Post-mortem si incident'
  ]
}
```

### 2. Sécurité

```bash
# Toujours faire un backup avant maintenance critique
mongodump --out=/backup/pre-maintenance-$(date +%Y%m%d)

# Vérifier l'intégrité du backup
mongorestore --dryRun --dir=/backup/pre-maintenance-20240115

# Documenter l'état actuel
mongosh --eval "printjson(rs.conf())" > rs-config-$(date +%Y%m%d).json
```

### 3. Automatisation

```javascript
// Framework de maintenance automatisée
class MaintenanceFramework {
  constructor(config) {
    this.config = config
    this.log = []
  }

  async runTask(task) {
    this.log.push({
      timestamp: new Date(),
      task: task.name,
      status: 'started'
    })

    try {
      const result = await task.execute()

      this.log.push({
        timestamp: new Date(),
        task: task.name,
        status: 'completed',
        result: result
      })

      return { success: true, result }
    } catch (error) {
      this.log.push({
        timestamp: new Date(),
        task: task.name,
        status: 'failed',
        error: error.message
      })

      if (task.rollback) {
        await task.rollback()
      }

      return { success: false, error }
    }
  }

  async runMaintenance(tasks) {
    const results = []

    for (const task of tasks) {
      const result = await this.runTask(task)
      results.push(result)

      if (!result.success && task.critical) {
        print(`Critical task failed: ${task.name}`)
        break
      }
    }

    return {
      completed: results.filter(r => r.success).length,
      failed: results.filter(r => !r.success).length,
      log: this.log,
      results: results
    }
  }
}
```

## Conclusion

La maintenance d'un Replica Set MongoDB nécessite :

- ✅ **Planification rigoureuse** : Fenêtres définies, procédures documentées
- ✅ **Approche progressive** : Rolling maintenance pour haute disponibilité
- ✅ **Validation continue** : Vérifier après chaque étape
- ✅ **Documentation complète** : Runbooks, checklists, historique
- ✅ **Automatisation** : Scripts testés et réutilisables
- ✅ **Plan de secours** : Procédures de rollback et d'urgence

**Opérations clés** :
1. Redémarrages (Secondary puis Primary avec stepDown)
2. Upgrades (rolling avec FCV)
3. Compaction et optimisation
4. Gestion des index (rolling build)
5. Rotation des logs et nettoyage

Une maintenance bien planifiée et exécutée garantit la stabilité, les performances et la disponibilité du Replica Set en production.

⏭️ [Réplication chaînée](/09-replication/12-replication-chainee.md)
