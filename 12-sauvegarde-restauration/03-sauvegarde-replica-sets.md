🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.3 Sauvegarde de Replica Sets

## Introduction

Les Replica Sets constituent l'architecture de déploiement standard pour MongoDB en production, offrant haute disponibilité et redondance des données. La sauvegarde d'un Replica Set nécessite une approche spécifique qui tire parti de sa topologie distribuée tout en minimisant l'impact sur les opérations de production.

Cette section détaille les stratégies, procédures et bonnes pratiques pour sauvegarder efficacement un Replica Set sans compromettre la disponibilité du service ni les performances applicatives.

## Architecture et Considérations

### Topologie Standard d'un Replica Set

```
┌─────────────────────────────────────────────────────────────┐
│                     Replica Set (rs0)                       │
│                                                             │
│   ┌──────────────┐         ┌──────────────┐                 │
│   │   PRIMARY    │────────>│  SECONDARY   │                 │
│   │  mongo-1     │<────────│  mongo-2     │                 │
│   │  (Writes)    │  Oplog  │  (Reads)     │                 │
│   └──────────────┘  Replic └──────────────┘                 │
│         │                           │                       │
│         │         Oplog             │                       │
│         │       Replication         │                       │
│         v                           v                       │
│   ┌──────────────┐         ┌──────────────┐                 │
│   │  SECONDARY   │         │  SECONDARY   │                 │
│   │  mongo-3     │────────>│  mongo-4     │                 │
│   │  (Reads)     │         │ (BACKUP)     │                 │
│   └──────────────┘         └──────────────┘                 │
│                              hidden: true                   │
│                              priority: 0                    │
│                              tags: {backup: "true"}         │
└─────────────────────────────────────────────────────────────┘

Clients → Load Balancer → PRIMARY (writes) + SECONDARIES (reads)
                           ↓
                       (Backup node isolated)
```

### Principes Fondamentaux

**Règles d'or pour les backups de Replica Set** :

1. **Jamais sur le Primary** : Effectuer les backups sur un secondary pour ne pas impacter les écritures
2. **Cohérence point-in-time** : Utiliser `--oplog` pour garantir la cohérence transactionnelle
3. **Isolation du membre de backup** : Utiliser un membre caché (hidden) dédié aux backups
4. **Monitoring du lag** : Surveiller le replication lag pendant les opérations
5. **Validation post-backup** : Vérifier l'intégrité et la complétude du backup

### Impact sur la Réplication

Comprendre l'impact d'un backup sur la réplication :

```javascript
// Impact pendant le backup
{
  "backup_operation": {
    // Charge CPU
    "cpu_usage": "+30-50%",  // Sur le membre de backup
    "other_members": "minimal",

    // Charge I/O
    "disk_read": "élevée",    // Lecture de tous les fichiers
    "disk_write": "minimale",

    // Réseau
    "network_bandwidth": {
      "backup_node": "élevée (si upload distant)",
      "replication": "peut augmenter si lag"
    },

    // Réplication
    "oplog_lag": {
      "typical": "quelques secondes à minutes",
      "max_acceptable": "< 10% de la fenêtre oplog",
      "alert_threshold": "5 minutes"
    }
  }
}
```

## Configuration d'un Membre de Backup Dédié

### Ajout d'un Membre Caché

```javascript
// 1. Se connecter au Primary
mongo "mongodb://admin:password@mongo-primary:27017/admin?replicaSet=rs0"

// 2. Récupérer la configuration actuelle
cfg = rs.conf()

// 3. Ajouter un nouveau membre dédié aux backups
cfg.members.push({
  _id: 4,                    // ID unique dans le replica set
  host: "mongo-backup:27017",
  priority: 0,               // Ne peut JAMAIS devenir Primary
  votes: 0,                  // Ne participe pas aux élections
  hidden: true,              // Invisible pour les clients
  slaveDelay: 0,            // Pas de délai (sauf besoin spécifique)
  tags: {                    // Tags pour identification
    backup: "true",
    datacenter: "backup-dc",
    workload: "analytics"
  }
})

// 4. Appliquer la configuration
rs.reconfig(cfg)

// 5. Vérifier la configuration
rs.conf().members.forEach(function(member) {
  if (member.tags && member.tags.backup === "true") {
    printjson({
      id: member._id,
      host: member.host,
      priority: member.priority,
      hidden: member.hidden,
      tags: member.tags
    });
  }
});

// Résultat attendu:
{
  "id": 4,
  "host": "mongo-backup:27017",
  "priority": 0,
  "hidden": true,
  "tags": {
    "backup": "true",
    "datacenter": "backup-dc",
    "workload": "analytics"
  }
}
```

### Configuration avec Delayed Secondary (Alternative)

Pour protection contre erreurs humaines, un membre delayed peut être utile :

```javascript
// Membre delayed de 24h pour recovery d'erreurs
cfg = rs.conf()
cfg.members.push({
  _id: 5,
  host: "mongo-delayed:27017",
  priority: 0,
  votes: 0,
  hidden: true,
  slaveDelay: 86400,  // 24 heures de retard
  tags: {
    delayed: "true",
    recovery: "human-error"
  }
})
rs.reconfig(cfg)

// Usage: Backup sur le delayed member
// Avantage: Protection automatique contre suppressions accidentelles
// Inconvénient: Données de J-1
```

### Validation de la Configuration

```bash
#!/bin/bash
# validate_backup_member.sh

MONGO_URI="mongodb://admin:password@mongo-primary:27017/admin?replicaSet=rs0"

echo "=== Replica Set Backup Configuration Validation ==="

# Vérifier la présence d'un membre de backup
mongo "$MONGO_URI" --quiet --eval "
  members = rs.conf().members;
  backupMembers = members.filter(function(m) {
    return m.tags && m.tags.backup === 'true';
  });

  if (backupMembers.length === 0) {
    print('❌ ERROR: No backup member configured');
    quit(1);
  }

  backupMembers.forEach(function(m) {
    print('✓ Backup member found: ' + m.host);
    print('  Priority: ' + m.priority + (m.priority === 0 ? ' ✓' : ' ✗ WARNING'));
    print('  Hidden: ' + m.hidden + (m.hidden ? ' ✓' : ' ✗ WARNING'));
    print('  Votes: ' + m.votes);

    // Vérifier l'état
    status = rs.status().members.filter(function(s) {
      return s._id === m._id;
    })[0];

    print('  State: ' + status.stateStr);
    print('  Health: ' + status.health);

    if (status.stateStr !== 'SECONDARY') {
      print('  ⚠️  WARNING: Member is not in SECONDARY state');
    }

    // Vérifier le lag
    if (status.optimeDate && status.lastHeartbeat) {
      lag = (new Date() - status.optimeDate) / 1000;
      print('  Replication lag: ' + lag.toFixed(2) + 's' +
            (lag < 60 ? ' ✓' : ' ⚠️'));
    }
  });
"

if [ $? -eq 0 ]; then
  echo ""
  echo "✓ Backup member configuration is valid"
else
  echo ""
  echo "✗ Backup member configuration has issues"
  exit 1
fi
```

## Stratégies de Sauvegarde

### Stratégie 1 : Backup depuis Secondary Caché

**Approche recommandée pour production** :

```bash
#!/bin/bash
# backup_from_hidden_secondary.sh

set -euo pipefail

# Configuration
RS_NAME="rs0"
BACKUP_MEMBER="mongo-backup:27017"
BACKUP_DIR="/backup/mongodb/replica_set"
RETENTION_DAYS=30
MONGO_USER="backup-user"
MONGO_PASS="SecureBackupPass123"

# Logging
LOG_FILE="/var/log/mongodb_rs_backup.log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Fonction: Vérifier l'état du membre
check_member_health() {
  local member=$1

  log "Checking health of backup member: $member"

  local state=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${member}/admin" --quiet --eval "
    rs.status().members.filter(function(m) {
      return m.name === '${member}';
    })[0].stateStr;
  ")

  if [ "$state" != "SECONDARY" ]; then
    log "ERROR: Backup member is not in SECONDARY state (current: $state)"
    return 1
  fi

  # Vérifier le lag de réplication
  local lag=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${member}/admin" --quiet --eval "
    member = rs.status().members.filter(function(m) {
      return m.name === '${member}';
    })[0];
    print((new Date() - member.optimeDate) / 1000);
  ")

  log "Replication lag: ${lag}s"

  # Alert si lag > 5 minutes
  if (( $(echo "$lag > 300" | bc -l) )); then
    log "WARNING: Replication lag is high (${lag}s)"
  fi

  return 0
}

# Fonction: Backup principal
perform_backup() {
  local timestamp=$(date +%Y%m%d_%H%M%S)
  local backup_path="${BACKUP_DIR}/rs_backup_${timestamp}"
  local start_time=$(date +%s)

  log "=== Starting Replica Set Backup ==="
  log "Timestamp: $timestamp"
  log "Target: $BACKUP_MEMBER"

  # Vérifier la santé avant de commencer
  if ! check_member_health "$BACKUP_MEMBER"; then
    log "ERROR: Pre-backup health check failed"
    exit 1
  fi

  # Créer le répertoire de backup
  mkdir -p "$backup_path"

  # Effectuer le backup avec oplog
  log "Executing mongodump with oplog..."

  if mongodump \
    --host="$BACKUP_MEMBER" \
    --username="$MONGO_USER" \
    --password="$MONGO_PASS" \
    --authenticationDatabase=admin \
    --oplog \
    --gzip \
    --numParallelCollections=8 \
    --readPreference='{ mode: "secondary", tagSets: [ { "backup": "true" } ] }' \
    --out="$backup_path"; then

    local end_time=$(date +%s)
    local duration=$((end_time - start_time))

    log "Backup completed in ${duration}s"

    # Sauvegarder les métadonnées du Replica Set
    log "Saving replica set metadata..."
    mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${BACKUP_MEMBER}/admin" --quiet > \
      "$backup_path/replica_set_metadata.json" <<'EOF'
      printjson({
        replSetConfig: rs.conf(),
        replSetStatus: rs.status(),
        oplogInfo: db.getSiblingDB('local').oplog.rs.find().sort({$natural: -1}).limit(1).toArray()[0],
        timestamp: new Date(),
        serverVersion: db.version(),
        serverBuildInfo: db.serverBuildInfo()
      });
EOF

    # Créer archive compressée
    log "Creating archive..."
    tar -czf "${backup_path}.tar.gz" -C "$BACKUP_DIR" "$(basename $backup_path)"

    # Checksum
    sha256sum "${backup_path}.tar.gz" > "${backup_path}.tar.gz.sha256"

    # Métadonnées du backup
    local size=$(stat -c%s "${backup_path}.tar.gz")
    cat > "${backup_path}.json" <<EOF
{
  "backup_type": "replica_set",
  "replica_set_name": "$RS_NAME",
  "backup_member": "$BACKUP_MEMBER",
  "timestamp": "$(date -Iseconds)",
  "duration_seconds": $duration,
  "size_bytes": $size,
  "size_human": "$(numfmt --to=iec-i --suffix=B $size)",
  "mongodb_version": "$(mongo --quiet --eval 'db.version()')",
  "oplog_included": true
}
EOF

    # Nettoyage du répertoire temporaire
    rm -rf "$backup_path"

    # Cleanup anciens backups
    log "Cleaning up old backups..."
    find "$BACKUP_DIR" -name "rs_backup_*.tar.gz" -mtime +$RETENTION_DAYS -delete
    find "$BACKUP_DIR" -name "rs_backup_*.json" -mtime +$RETENTION_DAYS -delete

    log "✓ Replica Set backup completed successfully"
    log "  Archive: ${backup_path}.tar.gz"
    log "  Size: $(numfmt --to=iec-i --suffix=B $size)"

    return 0
  else
    log "✗ Backup failed"
    return 1
  fi
}

# Fonction: Vérifier post-backup
post_backup_check() {
  log "Performing post-backup checks..."

  # Vérifier que le membre est toujours en bonne santé
  check_member_health "$BACKUP_MEMBER"

  # Vérifier que la réplication continue
  local lag_after=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${BACKUP_MEMBER}/admin" --quiet --eval "
    member = rs.status().members.filter(function(m) {
      return m.name === '${BACKUP_MEMBER}';
    })[0];
    print((new Date() - member.optimeDate) / 1000);
  ")

  log "Post-backup replication lag: ${lag_after}s"

  if (( $(echo "$lag_after > 600" | bc -l) )); then
    log "⚠️  WARNING: Replication lag increased significantly after backup"
  fi
}

# Exécution principale
main() {
  if perform_backup; then
    post_backup_check
    exit 0
  else
    log "Backup failed"
    exit 1
  fi
}

main "$@"
```

### Stratégie 2 : Rolling Backup (Backup Rotatif)

Pour minimiser l'impact, effectuer des backups sur différents membres en rotation :

```bash
#!/bin/bash
# rolling_backup_strategy.sh

RS_MEMBERS=(
  "mongo-secondary-1:27017"
  "mongo-secondary-2:27017"
  "mongo-secondary-3:27017"
)

CURRENT_MEMBER_INDEX=0
STATE_FILE="/var/lib/mongodb_backup/rolling_backup_state"

# Lire l'index du dernier membre utilisé
if [ -f "$STATE_FILE" ]; then
  CURRENT_MEMBER_INDEX=$(cat "$STATE_FILE")
fi

# Sélectionner le prochain membre (rotation)
NEXT_MEMBER_INDEX=$(( (CURRENT_MEMBER_INDEX + 1) % ${#RS_MEMBERS[@]} ))
BACKUP_MEMBER="${RS_MEMBERS[$NEXT_MEMBER_INDEX]}"

echo "Selected member for backup: $BACKUP_MEMBER (index: $NEXT_MEMBER_INDEX)"

# Sauvegarder l'index pour la prochaine fois
echo "$NEXT_MEMBER_INDEX" > "$STATE_FILE"

# Effectuer le backup sur ce membre
mongodump \
  --host="$BACKUP_MEMBER" \
  --oplog \
  --gzip \
  --out="/backup/rolling_backup_$(date +%Y%m%d)"

echo "✓ Rolling backup completed on $BACKUP_MEMBER"
```

### Stratégie 3 : Backup avec Freeze du Membre

Pour garantir zéro impact sur la production, temporairement "geler" le membre :

```bash
#!/bin/bash
# backup_with_freeze.sh

BACKUP_MEMBER="mongo-backup:27017"
MONGO_URI="mongodb://admin:pass@${BACKUP_MEMBER}/admin"
FREEZE_DURATION=3600  # 1 heure

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# 1. Geler le membre (empêche élection pendant la période)
log "Freezing member for $FREEZE_DURATION seconds..."
mongo "$MONGO_URI" --eval "rs.freeze($FREEZE_DURATION)"

# 2. Optionnel: Passer en mode maintenance (pas de lectures)
log "Setting maintenance mode..."
mongo "$MONGO_URI" --eval "db.adminCommand({ replSetMaintenance: 1 })"

# 3. Effectuer le backup
log "Performing backup..."
mongodump \
  --host="$BACKUP_MEMBER" \
  --oplog \
  --gzip \
  --numParallelCollections=16 \
  --out="/backup/frozen_backup_$(date +%Y%m%d)"

BACKUP_EXIT_CODE=$?

# 4. Sortir du mode maintenance
log "Exiting maintenance mode..."
mongo "$MONGO_URI" --eval "db.adminCommand({ replSetMaintenance: 0 })"

# 5. Dégeler (automatique après timeout, mais bonne pratique)
mongo "$MONGO_URI" --eval "rs.freeze(0)"

if [ $BACKUP_EXIT_CODE -eq 0 ]; then
  log "✓ Backup completed successfully"
else
  log "✗ Backup failed"
  exit 1
fi
```

## Sauvegarde de l'Oplog

### Importance de l'Oplog dans les Replica Sets

```javascript
// L'oplog est critique pour:
// 1. Cohérence point-in-time
// 2. Restauration avec replay
// 3. Synchronisation des membres

// Vérifier la fenêtre oplog
use local
db.oplog.rs.find().sort({$natural: 1}).limit(1).pretty()  // Premier
db.oplog.rs.find().sort({$natural: -1}).limit(1).pretty() // Dernier

// Calculer la durée de l'oplog
firstOplog = db.oplog.rs.find().sort({$natural: 1}).limit(1).toArray()[0]
lastOplog = db.oplog.rs.find().sort({$natural: -1}).limit(1).toArray()[0]
oplogWindowHours = (lastOplog.ts.getTime() - firstOplog.ts.getTime()) / 3600

print("Oplog window: " + oplogWindowHours.toFixed(2) + " hours")
// Production: Minimum 48-72h recommandé
```

### Export Continu de l'Oplog

Pour recovery point objectif minimal :

```javascript
// continuous_oplog_backup.js
const { MongoClient } = require('mongodb');
const fs = require('fs');
const zlib = require('zlib');

const config = {
  uri: 'mongodb://backup-node:27017/?replicaSet=rs0',
  database: 'local',
  collection: 'oplog.rs',
  outputDir: '/backup/oplog',
  rotationInterval: 3600000, // 1 heure
  compressionLevel: 6
};

class OplogBackup {
  constructor(config) {
    this.config = config;
    this.client = null;
    this.currentStream = null;
    this.lastTimestamp = null;
    this.stateFile = '/var/lib/mongodb_backup/oplog_state.json';
  }

  async connect() {
    this.client = await MongoClient.connect(this.config.uri, {
      readPreference: 'secondary',
      readPreferenceTags: [{ backup: 'true' }]
    });

    console.log('Connected to MongoDB for oplog streaming');
  }

  loadState() {
    try {
      const state = JSON.parse(fs.readFileSync(this.stateFile, 'utf8'));
      this.lastTimestamp = state.lastTimestamp;
      console.log(`Resuming from timestamp: ${this.lastTimestamp}`);
    } catch (err) {
      console.log('No previous state found, starting from current oplog');
      this.lastTimestamp = null;
    }
  }

  saveState(timestamp) {
    fs.writeFileSync(this.stateFile, JSON.stringify({
      lastTimestamp: timestamp,
      lastSaved: new Date().toISOString()
    }));
  }

  createOutputStream() {
    const filename = `oplog_${new Date().toISOString().replace(/[:.]/g, '-')}.jsonl.gz`;
    const filepath = `${this.config.outputDir}/${filename}`;

    const fileStream = fs.createWriteStream(filepath);
    const gzipStream = zlib.createGzip({ level: this.config.compressionLevel });

    gzipStream.pipe(fileStream);

    console.log(`Created new oplog file: ${filepath}`);
    return { gzipStream, filepath };
  }

  async streamOplog() {
    const db = this.client.db(this.config.database);
    const oplog = db.collection(this.config.collection);

    // Construire le filtre
    const filter = this.lastTimestamp
      ? { ts: { $gt: this.lastTimestamp } }
      : {};

    // Créer le change stream
    const changeStream = oplog.watch([
      { $match: filter }
    ], {
      fullDocument: 'updateLookup',
      startAtOperationTime: this.lastTimestamp
    });

    let { gzipStream, filepath } = this.createOutputStream();
    let rotationTimer = setTimeout(() => this.rotateStream(), this.config.rotationInterval);
    let operationCount = 0;

    console.log('Oplog streaming started...');

    changeStream.on('change', (change) => {
      // Écrire l'opération
      gzipStream.write(JSON.stringify(change) + '\n');

      // Mettre à jour le timestamp
      this.lastTimestamp = change.ts;
      operationCount++;

      // Sauvegarder l'état tous les 1000 ops
      if (operationCount % 1000 === 0) {
        this.saveState(this.lastTimestamp);
        console.log(`Processed ${operationCount} operations`);
      }
    });

    changeStream.on('error', (error) => {
      console.error('Error in change stream:', error);
      this.reconnect();
    });
  }

  async start() {
    await this.connect();
    this.loadState();
    await this.streamOplog();
  }
}

// Démarrage
const backup = new OplogBackup(config);
backup.start().catch(console.error);

// Gestion des signaux
process.on('SIGINT', () => {
  console.log('Shutting down gracefully...');
  backup.saveState(backup.lastTimestamp);
  process.exit(0);
});
```

### Backup de l'Oplog avec mongodump

```bash
#!/bin/bash
# oplog_backup_dedicated.sh

BACKUP_DIR="/backup/oplog"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
STATE_FILE="/var/lib/mongodb_backup/last_oplog_timestamp"

# Lire le dernier timestamp sauvegardé
if [ -f "$STATE_FILE" ]; then
  LAST_TS=$(cat "$STATE_FILE")
  echo "Last oplog timestamp: $LAST_TS"
else
  echo "No previous oplog backup found"
  LAST_TS="Timestamp(0, 0)"
fi

# Exporter l'oplog depuis le dernier timestamp
mongodump \
  --host="mongo-backup:27017" \
  --db=local \
  --collection=oplog.rs \
  --query="{\"ts\": {\"\$gt\": $LAST_TS}}" \
  --gzip \
  --out="$BACKUP_DIR/oplog_incremental_$TIMESTAMP"

# Sauvegarder le nouveau timestamp
CURRENT_TS=$(mongo "mongodb://mongo-backup:27017/local" --quiet --eval "
  db.oplog.rs.find().sort({ts: -1}).limit(1).toArray()[0].ts;
")

echo "$CURRENT_TS" > "$STATE_FILE"

echo "✓ Oplog backup completed: oplog_incremental_$TIMESTAMP"
echo "  Last timestamp: $CURRENT_TS"
```

## Monitoring et Validation

### Monitoring du Replication Lag

```bash
#!/bin/bash
# monitor_replication_lag.sh

MONGO_URI="mongodb://admin:pass@mongo-primary:27017/admin?replicaSet=rs0"
ALERT_THRESHOLD=300  # 5 minutes

while true; do
  echo "=== Replication Status - $(date) ==="

  mongo "$MONGO_URI" --quiet --eval "
    status = rs.status();
    primary = status.members.filter(m => m.stateStr === 'PRIMARY')[0];

    status.members.forEach(function(member) {
      if (member.stateStr === 'SECONDARY') {
        lag = (primary.optimeDate - member.optimeDate) / 1000;

        status_icon = '✓';
        if (lag > $ALERT_THRESHOLD) {
          status_icon = '⚠️';
        }
        if (member.health !== 1) {
          status_icon = '✗';
        }

        print(status_icon + ' ' + member.name +
              ' | State: ' + member.stateStr +
              ' | Health: ' + member.health +
              ' | Lag: ' + lag.toFixed(2) + 's' +
              (member.tags ? ' | Tags: ' + JSON.stringify(member.tags) : ''));
      }
    });
  "

  sleep 10
done
```

### Dashboard de Monitoring avec Prometheus

```yaml
# prometheus_rules.yml
groups:
  - name: mongodb_replica_set_backup
    interval: 30s
    rules:
      # Alerte sur lag de réplication élevé
      - alert: MongoDBReplicationLagHigh
        expr: |
          mongodb_mongod_replset_member_replication_lag_seconds{
            state="SECONDARY",
            tags_backup="true"
          } > 300
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB replication lag high on backup member"
          description: "Backup member {{ $labels.member }} has replication lag of {{ $value }}s"

      # Alerte si membre de backup n'est pas SECONDARY
      - alert: MongoDBBackupMemberNotHealthy
        expr: |
          mongodb_mongod_replset_member_state{
            tags_backup="true"
          } != 2
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB backup member is not in SECONDARY state"
          description: "Backup member {{ $labels.member }} is in state {{ $value }}"

      # Alerte sur échec de backup
      - alert: MongoDBBackupFailed
        expr: |
          time() - mongodb_backup_last_success_timestamp > 86400
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB backup has not succeeded in 24h"
          description: "Last successful backup was {{ $value | humanizeDuration }} ago"
```

### Script de Validation Complète

```bash
#!/bin/bash
# validate_replica_set_backup.sh

BACKUP_FILE="/backup/mongodb/rs_backup_20241208.tar.gz"
TEST_DIR="/tmp/backup_validation_$$"
TEST_PORT=27099

echo "=== Replica Set Backup Validation ==="
echo "Backup file: $BACKUP_FILE"

# 1. Vérifier l'intégrité du fichier
echo -e "\n1. Checking file integrity..."
if [ -f "${BACKUP_FILE}.sha256" ]; then
  if sha256sum -c "${BACKUP_FILE}.sha256"; then
    echo "✓ Checksum valid"
  else
    echo "✗ Checksum INVALID"
    exit 1
  fi
else
  echo "⚠️  No checksum file found"
fi

# 2. Extraire le backup
echo -e "\n2. Extracting backup..."
mkdir -p "$TEST_DIR"
tar -xzf "$BACKUP_FILE" -C "$TEST_DIR"

if [ $? -ne 0 ]; then
  echo "✗ Failed to extract backup"
  exit 1
fi

# 3. Vérifier la présence de l'oplog
echo -e "\n3. Checking oplog..."
if [ -f "$TEST_DIR"/*/oplog.bson ]; then
  OPLOG_SIZE=$(stat -c%s "$TEST_DIR"/*/oplog.bson)
  echo "✓ Oplog found ($(numfmt --to=iec-i --suffix=B $OPLOG_SIZE))"
else
  echo "✗ Oplog NOT found - backup may not be point-in-time consistent"
fi

# 4. Vérifier les métadonnées du replica set
echo -e "\n4. Checking replica set metadata..."
if [ -f "$TEST_DIR"/*/replica_set_metadata.json ]; then
  echo "✓ Replica set metadata found"

  RS_NAME=$(jq -r '.replSetConfig._id' "$TEST_DIR"/*/replica_set_metadata.json)
  MEMBER_COUNT=$(jq '.replSetStatus.members | length' "$TEST_DIR"/*/replica_set_metadata.json)

  echo "  Replica Set: $RS_NAME"
  echo "  Members: $MEMBER_COUNT"
else
  echo "⚠️  Replica set metadata not found"
fi

# 5. Test de restauration (optionnel, nécessite instance MongoDB test)
if command -v mongod &> /dev/null; then
  echo -e "\n5. Testing restoration (sample)..."

  # Démarrer instance MongoDB temporaire
  TEMP_DBPATH="$TEST_DIR/test_mongod"
  mkdir -p "$TEMP_DBPATH"

  mongod --port $TEST_PORT --dbpath "$TEMP_DBPATH" \
    --logpath "$TEST_DIR/mongod.log" --fork

  sleep 5

  # Restaurer le backup
  mongorestore --port $TEST_PORT --drop --gzip \
    --dir="$TEST_DIR"/*/ \
    --nsInclude="testdb.*" 2>&1 | grep -i "done"

  if [ $? -eq 0 ]; then
    echo "✓ Test restoration successful"

    # Vérifier quelques données
    mongo --port $TEST_PORT --quiet --eval "
      dbs = db.adminCommand({ listDatabases: 1 }).databases;
      print('Databases restored: ' + dbs.length);
    "
  else
    echo "✗ Test restoration failed"
  fi

  # Arrêter MongoDB test
  mongod --port $TEST_PORT --dbpath "$TEMP_DBPATH" --shutdown
else
  echo -e "\n5. Skipping restoration test (mongod not available)"
fi

# Cleanup
rm -rf "$TEST_DIR"

echo -e "\n✓ Validation completed"
```

## Restauration de Replica Set

### Restauration Complète du Replica Set

```bash
#!/bin/bash
# restore_replica_set_complete.sh

set -euo pipefail

BACKUP_FILE="/backup/mongodb/rs_backup_20241208.tar.gz"
RS_NAME="rs0"
RS_MEMBERS=(
  "mongo-1:27017"
  "mongo-2:27017"
  "mongo-3:27017"
)

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Replica Set Complete Restoration ==="
log "Backup: $BACKUP_FILE"
log "Replica Set: $RS_NAME"
log ""

# 1. Extraire le backup
TEMP_DIR="/tmp/rs_restore_$$"
mkdir -p "$TEMP_DIR"

log "Extracting backup..."
tar -xzf "$BACKUP_FILE" -C "$TEMP_DIR"
BACKUP_DIR=$(find "$TEMP_DIR" -type d -name "rs_backup_*" | head -1)

# 2. Arrêter tous les membres du replica set
log "Stopping all replica set members..."
for member in "${RS_MEMBERS[@]}"; do
  host=$(echo $member | cut -d: -f1)
  port=$(echo $member | cut -d: -f2)

  log "  Stopping $member..."
  mongo "mongodb://$member/admin" --eval "db.shutdownServer()" || true
done

sleep 5

# 3. Restaurer sur chaque membre
log "Restoring data to each member..."

for i in "${!RS_MEMBERS[@]}"; do
  member="${RS_MEMBERS[$i]}"
  host=$(echo $member | cut -d: -f1)
  port=$(echo $member | cut -d: -f2)

  log "  Restoring to member $((i+1)): $member"

  # Nettoyer le datadir (ATTENTION: destructif!)
  ssh "$host" "sudo rm -rf /var/lib/mongodb/*"

  # Restaurer
  mongorestore \
    --host="$member" \
    --drop \
    --oplogReplay \
    --gzip \
    --dir="$BACKUP_DIR" &
done

# Attendre toutes les restaurations
wait

log "All members restored"

# 4. Démarrer le premier membre en mode standalone
log "Starting first member as standalone..."
PRIMARY="${RS_MEMBERS[0]}"
ssh "$(echo $PRIMARY | cut -d: -f1)" "sudo systemctl start mongod"

sleep 10

# 5. Réinitialiser le replica set
log "Re-initializing replica set..."
mongo "mongodb://$PRIMARY/admin" <<EOF
  // Initier le replica set
  rs.initiate({
    _id: "$RS_NAME",
    members: [
      { _id: 0, host: "${RS_MEMBERS[0]}" }
    ]
  });

  // Attendre que le member devienne PRIMARY
  sleep(5000);

  // Ajouter les autres membres
  $(for i in $(seq 1 $((${#RS_MEMBERS[@]} - 1))); do
    echo "rs.add({ host: '${RS_MEMBERS[$i]}', priority: 1 });"
    echo "sleep(2000);"
  done)

  // Afficher le statut
  rs.status();
EOF

# 6. Attendre la synchronisation
log "Waiting for replica set to stabilize..."
for attempt in {1..30}; do
  HEALTHY=$(mongo "mongodb://$PRIMARY/admin" --quiet --eval "
    status = rs.status();
    healthy = status.members.filter(m => m.health === 1).length;
    print(healthy);
  ")

  if [ "$HEALTHY" == "${#RS_MEMBERS[@]}" ]; then
    log "✓ All members are healthy"
    break
  fi

  log "  Waiting... ($HEALTHY/${#RS_MEMBERS[@]} healthy)"
  sleep 10
done

# 7. Vérifier l'intégrité
log "Verifying data integrity..."
mongo "mongodb://$PRIMARY/admin" --quiet --eval "
  db.adminCommand({ listDatabases: 1 }).databases.forEach(function(db) {
    if (db.name !== 'admin' && db.name !== 'local' && db.name !== 'config') {
      print('✓ Database: ' + db.name + ' - Size: ' +
            (db.sizeOnDisk / 1024 / 1024 / 1024).toFixed(2) + ' GB');
    }
  });
"

# Cleanup
rm -rf "$TEMP_DIR"

log ""
log "✓ Replica Set restoration completed successfully"
log "  Verify application connectivity before resuming production traffic"
```

### Restauration d'un Seul Membre

```bash
#!/bin/bash
# restore_single_member.sh

BACKUP_FILE="/backup/mongodb/rs_backup_20241208.tar.gz"
TARGET_MEMBER="mongo-2:27017"
RS_NAME="rs0"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Single Member Restoration ==="
log "Target: $TARGET_MEMBER"

# 1. Retirer le membre du replica set
log "Removing member from replica set..."
PRIMARY=$(mongo "mongodb://$TARGET_MEMBER/admin?replicaSet=$RS_NAME" --quiet --eval "
  rs.isMaster().primary;
")

mongo "mongodb://$PRIMARY/admin" --eval "
  cfg = rs.conf();
  cfg.members = cfg.members.filter(m => m.host !== '$TARGET_MEMBER');
  rs.reconfig(cfg, {force: true});
"

sleep 5

# 2. Arrêter le membre
log "Stopping target member..."
mongo "mongodb://$TARGET_MEMBER/admin" --eval "db.shutdownServer()" || true

sleep 5

# 3. Nettoyer et restaurer
host=$(echo $TARGET_MEMBER | cut -d: -f1)

log "Cleaning data directory..."
ssh "$host" "sudo rm -rf /var/lib/mongodb/*"

log "Extracting and restoring backup..."
TEMP_DIR="/tmp/restore_$$"
mkdir -p "$TEMP_DIR"
tar -xzf "$BACKUP_FILE" -C "$TEMP_DIR"

mongorestore \
  --host="$TARGET_MEMBER" \
  --drop \
  --oplogReplay \
  --gzip \
  --dir="$TEMP_DIR"/*

# 4. Réajouter au replica set
log "Re-adding member to replica set..."
mongo "mongodb://$PRIMARY/admin" --eval "
  rs.add('$TARGET_MEMBER');
"

log "✓ Member restored and re-added to replica set"
log "  Waiting for initial sync to complete..."

# Attendre la synchronisation
for i in {1..60}; do
  STATE=$(mongo "mongodb://$PRIMARY/admin" --quiet --eval "
    rs.status().members.filter(m => m.name === '$TARGET_MEMBER')[0].stateStr;
  ")

  if [ "$STATE" == "SECONDARY" ]; then
    log "✓ Member is now SECONDARY and in sync"
    break
  fi

  log "  State: $STATE (waiting...)"
  sleep 10
done

rm -rf "$TEMP_DIR"
```

## Automatisation et Orchestration

### Cron pour Backups Automatiques

```bash
# /etc/cron.d/mongodb-replica-set-backup

# Backup quotidien à 2h du matin
0 2 * * * mongodb /usr/local/bin/backup_replica_set.sh >> /var/log/mongodb_backup_cron.log 2>&1

# Backup hebdomadaire complet le dimanche à 3h
0 3 * * 0 mongodb /usr/local/bin/backup_replica_set_full.sh >> /var/log/mongodb_backup_cron.log 2>&1

# Export oplog toutes les 4 heures
0 */4 * * * mongodb /usr/local/bin/backup_oplog_incremental.sh >> /var/log/mongodb_oplog_cron.log 2>&1

# Validation quotidienne à 4h
0 4 * * * mongodb /usr/local/bin/validate_last_backup.sh >> /var/log/mongodb_validation_cron.log 2>&1

# Cleanup hebdomadaire le lundi à 5h
0 5 * * 1 mongodb /usr/local/bin/cleanup_old_backups.sh >> /var/log/mongodb_cleanup_cron.log 2>&1
```

### Kubernetes CronJob pour Replica Set

```yaml
# mongodb-rs-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-replica-set-backup
  namespace: database
spec:
  schedule: "0 2 * * *"  # 2h du matin tous les jours
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: mongodb-backup
        spec:
          restartPolicy: OnFailure
          serviceAccountName: mongodb-backup

          # Affinité pour s'exécuter près du membre de backup
          affinity:
            podAffinity:
              preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchExpressions:
                    - key: app
                      operator: In
                      values:
                      - mongodb
                    - key: role
                      operator: In
                      values:
                      - backup
                  topologyKey: kubernetes.io/hostname

          containers:
          - name: backup
            image: mongo:7.0
            command:
            - /bin/bash
            - -c
            - |
              set -euo pipefail

              echo "=== MongoDB Replica Set Backup ==="
              echo "Timestamp: $(date -Iseconds)"

              # Identifier le membre de backup
              BACKUP_MEMBER=$(mongo "mongodb://mongo-0.mongo:27017,mongo-1.mongo:27017,mongo-2.mongo:27017/?replicaSet=rs0" \
                --username=$MONGO_USER \
                --password=$MONGO_PASSWORD \
                --authenticationDatabase=admin \
                --quiet --eval "
                  rs.status().members.filter(function(m) {
                    return m.tags && m.tags.backup === 'true';
                  })[0].name;
                ")

              echo "Backup member: $BACKUP_MEMBER"

              # Effectuer le backup
              BACKUP_NAME="rs_backup_$(date +%Y%m%d_%H%M%S).gz"

              mongodump \
                --host="$BACKUP_MEMBER" \
                --username=$MONGO_USER \
                --password=$MONGO_PASSWORD \
                --authenticationDatabase=admin \
                --oplog \
                --gzip \
                --numParallelCollections=8 \
                --readPreference='{ mode: "secondary", tagSets: [ { "backup": "true" } ] }' \
                --archive=/backup/$BACKUP_NAME

              # Upload vers S3
              echo "Uploading to S3..."
              aws s3 cp /backup/$BACKUP_NAME $S3_BUCKET/ \
                --storage-class STANDARD_IA \
                --metadata "backup-type=replica-set,timestamp=$(date +%s)"

              # Métadonnées
              cat > /backup/${BACKUP_NAME%.gz}.json <<EOF
              {
                "backup_type": "replica_set",
                "timestamp": "$(date -Iseconds)",
                "size_bytes": $(stat -c%s /backup/$BACKUP_NAME),
                "backup_member": "$BACKUP_MEMBER"
              }
              EOF

              aws s3 cp /backup/${BACKUP_NAME%.gz}.json $S3_BUCKET/

              echo "✓ Backup completed: $BACKUP_NAME"

            env:
            - name: MONGO_USER
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-credentials
                  key: username
            - name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-credentials
                  key: password
            - name: S3_BUCKET
              value: "s3://company-backups/mongodb-replica-sets"
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: access_key_id
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: aws-credentials
                  key: secret_access_key

            volumeMounts:
            - name: backup-storage
              mountPath: /backup

            resources:
              requests:
                memory: "4Gi"
                cpu: "2"
              limits:
                memory: "8Gi"
                cpu: "4"

          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: mongodb-backup-pvc
```

## Cas Particuliers et Troubleshooting

### Scénario 1 : Backup Pendant une Élection

```bash
# Gérer les élections pendant le backup
perform_backup_with_election_handling() {
  local max_attempts=3
  local attempt=0

  while [ $attempt -lt $max_attempts ]; do
    attempt=$((attempt + 1))

    echo "Backup attempt $attempt/$max_attempts"

    # Vérifier qu'il y a un primary
    PRIMARY=$(mongo "$MONGO_URI" --quiet --eval "rs.isMaster().primary")

    if [ -z "$PRIMARY" ]; then
      echo "No PRIMARY found, waiting for election..."
      sleep 30
      continue
    fi

    # Effectuer le backup
    if mongodump --host="$BACKUP_MEMBER" --oplog --gzip --out=/backup/attempt_$attempt; then
      echo "✓ Backup successful"
      return 0
    else
      echo "✗ Backup failed, retrying..."
      sleep 60
    fi
  done

  echo "✗ Backup failed after $max_attempts attempts"
  return 1
}
```

### Scénario 2 : Recovery après Split-Brain

```javascript
// Identifier et résoudre un split-brain
// Se connecter à chaque membre indépendamment

// Membre 1
mongo mongodb://mongo-1:27017/admin
rs.status()
// Si state: PRIMARY mais isolated

// Forcer une nouvelle configuration depuis le vrai PRIMARY
cfg = rs.conf()
cfg.version++
rs.reconfig(cfg, {force: true})

// Restaurer depuis le backup le plus récent si nécessaire
```

### Scénario 3 : Backup avec Oplog Épuisé

```bash
#!/bin/bash
# handle_oplog_exhaustion.sh

# Vérifier la taille de la fenêtre oplog avant backup
check_oplog_window() {
  WINDOW_HOURS=$(mongo "mongodb://backup-node:27017/local" --quiet --eval "
    first = db.oplog.rs.find().sort({ts: 1}).limit(1).toArray()[0].ts.getTime();
    last = db.oplog.rs.find().sort({ts: -1}).limit(1).toArray()[0].ts.getTime();
    print((last - first) / 3600);
  ")

  echo "Oplog window: ${WINDOW_HOURS}h"

  if (( $(echo "$WINDOW_HOURS < 24" | bc -l) )); then
    echo "⚠️  WARNING: Oplog window < 24h, backup may not capture full history"
    echo "Consider increasing oplogSizeMB or performing backup more frequently"
  fi
}

check_oplog_window

# Augmenter la taille de l'oplog si nécessaire
# Note: Nécessite MongoDB 4.0+
# mongo admin --eval "db.adminCommand({replSetResizeOplog: 1, size: 102400})"  # 100GB
```

## Checklist de Production

```markdown
### Avant le Backup

- [ ] Vérifier l'état du Replica Set (rs.status())
- [ ] Confirmer qu'un membre SECONDARY est disponible
- [ ] Vérifier le replication lag (< 5 min)
- [ ] Confirmer la fenêtre oplog (> 48h recommandé)
- [ ] Vérifier l'espace disque disponible
- [ ] Confirmer que le balancer est actif (si sharded)
- [ ] Vérifier les alertes de monitoring

### Pendant le Backup

- [ ] Monitorer le replication lag en temps réel
- [ ] Surveiller l'utilisation CPU/RAM/IO du membre
- [ ] Vérifier que les autres membres restent sains
- [ ] Confirmer progression du backup (taille croissante)
- [ ] Vérifier qu'aucune élection n'est en cours

### Après le Backup

- [ ] Vérifier l'intégrité du fichier (checksum)
- [ ] Valider la présence de l'oplog dans le backup
- [ ] Confirmer la taille (cohérence avec historique)
- [ ] Upload vers stockage distant réussi
- [ ] Métadonnées enregistrées
- [ ] Replication lag revenu à la normale
- [ ] Tester restauration (périodiquement)
- [ ] Documenter tout incident
```

## Conclusion

La sauvegarde de Replica Sets requiert une approche méthodique qui balance protection des données et continuité de service. Les points clés :

1. **Toujours backup sur un SECONDARY** - Jamais sur le PRIMARY
2. **Utiliser `--oplog`** - Pour cohérence point-in-time
3. **Membre dédié aux backups** - Configuration hidden/priority:0
4. **Monitoring continu** - Replication lag et santé du cluster
5. **Automatisation robuste** - Scripts avec gestion d'erreurs
6. **Tests réguliers** - Valider la recouvrabilité

Un Replica Set bien configuré pour les backups combine haute disponibilité et recouvrabilité fiable, deux piliers essentiels de toute infrastructure de production MongoDB.

---


⏭️ [Sauvegarde de clusters shardés](/12-sauvegarde-restauration/04-sauvegarde-clusters-shardes.md)
