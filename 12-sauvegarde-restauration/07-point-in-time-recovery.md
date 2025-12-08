🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.7 Point-in-Time Recovery (PITR)

## Introduction

Le Point-in-Time Recovery (PITR) représente la capacité de restaurer une base de données à un état exact à n'importe quel moment dans le passé, avec une précision à la seconde près. Cette fonctionnalité est cruciale pour les environnements de production où les erreurs humaines, les bugs applicatifs ou les corruptions de données peuvent nécessiter un retour en arrière précis sans perdre plus de données que nécessaire.

MongoDB supporte le PITR nativement grâce à son architecture de réplication basée sur l'oplog, permettant une récupération granulaire et précise après incidents.

## Concepts Fondamentaux

### Qu'est-ce que le Point-in-Time Recovery ?

```
Timeline de Données MongoDB:

├────────┬────────┬────────┬────────┬────────┬────────> Temps
│        │        │        │        │        │
│    Backup    Transaction  Bug      Détection
│    02:00     10:30:15   14:37:22  15:00:00
│        │        │        │        │
│        │        │        X (Incident)
│        │        │        │
│        │    Restauration possible à 14:37:21
│        │        │
│        └────────┴────────> Replay des opérations
│                           depuis le backup jusqu'à
│                           14:37:21 (juste avant l'incident)

Objectif: Minimiser la perte de données (RPO proche de zéro)
```

### Architecture du PITR avec MongoDB

```
┌────────────────────────────────────────────────────────────┐
│              Point-in-Time Recovery Architecture           │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          Base Backup (Snapshot)                      │  │
│  │  État de la base à T0 (ex: 02:00:00)                 │  │
│  │  • Toutes les collections                            │  │
│  │  • Tous les index                                    │  │
│  │  • État cohérent                                     │  │
│  └────────────────┬─────────────────────────────────────┘  │
│                   │                                        │
│                   v                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Oplog (Operations Log)                       │  │
│  │  Stream continu des opérations depuis T0             │  │
│  │                                                      │  │
│  │  T0+00:15  │ insert │ users │ {...}                  │  │
│  │  T0+00:32  │ update │ orders │ {...}                 │  │
│  │  T0+01:05  │ delete │ temp │ {...}                   │  │
│  │  ...                                                 │  │
│  │  T0+12:37:21 │ update │ products │ {...}             │  │
│  │  T0+12:37:22 │ DELETE │ orders │ {} ← ERREUR!        │  │
│  │  T0+12:37:23 │ insert │ logs │ {...}                 │  │
│  └────────────────┬─────────────────────────────────────┘  │
│                   │                                        │
│                   v                                        │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         PITR Process                                 │  │
│  │                                                      │  │
│  │  1. Restaurer le backup de T0                        │  │
│  │  2. Rejouer l'oplog jusqu'à T0+12:37:21              │  │
│  │  3. STOP avant l'opération erronée                   │  │
│  │                                                      │  │
│  │  Résultat: Base dans l'état exact à 14:37:21         │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### L'Oplog : Pilier du PITR

```javascript
// Structure de l'oplog
use local
db.oplog.rs.findOne()

{
  "ts": Timestamp(1701993600, 42),    // Timestamp unique (secondes, compteur)
  "t": NumberLong(23),                 // Terme de réplication
  "h": NumberLong("9876543210"),       // Hash de l'opération
  "v": 2,                              // Version du format oplog
  "op": "i",                           // Type: i=insert, u=update, d=delete, c=command
  "ns": "mydb.users",                  // Namespace (database.collection)
  "ui": UUID("..."),                   // UUID de la collection
  "o": {                               // Objet de l'opération
    "_id": ObjectId("..."),
    "name": "John Doe",
    "email": "john@example.com"
  },
  "wall": ISODate("2024-12-08T14:37:22.123Z")  // Wall clock time
}

// Types d'opérations dans l'oplog
{
  "i": "insert",              // Nouvelle insertion
  "u": "update",              // Mise à jour
  "d": "delete",              // Suppression
  "c": "command",             // Commande (create, drop, etc.)
  "n": "no-op"                // Opération vide (keepalive)
}
```

### Fenêtre de PITR

```javascript
// Calculer la fenêtre oplog disponible
use local

// Premier et dernier timestamp
var first = db.oplog.rs.find().sort({$natural: 1}).limit(1).toArray()[0].ts;
var last = db.oplog.rs.find().sort({$natural: -1}).limit(1).toArray()[0].ts;

// Durée en heures
var windowHours = (last.getTime() - first.getTime()) / 3600;

print("Oplog Window: " + windowHours.toFixed(2) + " hours");
print("First timestamp: " + first);
print("Last timestamp: " + last);

// Recommandations par criticité:
// Development:     12-24 heures
// Production:      48-72 heures
// Mission-critical: 168 heures (7 jours)
```

## Méthodes de Point-in-Time Recovery

### Méthode 1 : Backup + Oplog Replay (Recommandé)

C'est l'approche standard avec mongodump :

```bash
#!/bin/bash
# pitr_restore_from_backup.sh

set -euo pipefail

# Configuration
BACKUP_FILE="/backup/mongodb/full_backup_20241208_020000.gz"
TARGET_TIMESTAMP="2024-12-08T14:37:21Z"
RESTORE_HOST="mongodb://localhost:27017"
TEMP_DIR="/tmp/pitr_restore_$$"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  MongoDB Point-in-Time Recovery"
log "========================================================"
log "Backup: $BACKUP_FILE"
log "Target Time: $TARGET_TIMESTAMP"
log ""

# Vérifier que le backup existe
if [ ! -f "$BACKUP_FILE" ]; then
  log "ERROR: Backup file not found"
  exit 1
fi

# Convertir le timestamp en format MongoDB
TARGET_TS=$(date -d "$TARGET_TIMESTAMP" +%s)
log "Target timestamp (epoch): $TARGET_TS"

# ============================================================================
# PHASE 1: RESTAURER LE BACKUP DE BASE
# ============================================================================

log "=== Phase 1: Restoring Base Backup ==="

mkdir -p "$TEMP_DIR"

# Extraire le backup
log "Extracting backup..."
tar -xzf "$BACKUP_FILE" -C "$TEMP_DIR"

BACKUP_DIR=$(find "$TEMP_DIR" -type d -name "*backup*" | head -1)

if [ -z "$BACKUP_DIR" ]; then
  log "ERROR: Cannot find backup directory"
  exit 1
fi

# Identifier le timestamp du backup (depuis l'oplog si présent)
if [ -f "$BACKUP_DIR/oplog.bson" ]; then
  BACKUP_TS=$(bsondump "$BACKUP_DIR/oplog.bson" | head -1 | jq -r '.ts["$timestamp"].t')
  log "Backup timestamp: $BACKUP_TS"

  # Vérifier que le target est après le backup
  if [ "$TARGET_TS" -lt "$BACKUP_TS" ]; then
    log "ERROR: Target timestamp is before backup timestamp"
    log "  Target: $TARGET_TS ($TARGET_TIMESTAMP)"
    log "  Backup: $BACKUP_TS"
    exit 1
  fi
else
  log "WARNING: No oplog found in backup"
  read -p "Continue without oplog verification? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    exit 1
  fi
fi

# Restaurer le backup de base
log "Restoring base backup..."
mongorestore \
  --uri="$RESTORE_HOST" \
  --drop \
  --gzip \
  --dir="$BACKUP_DIR"

if [ $? -ne 0 ]; then
  log "ERROR: Base backup restoration failed"
  exit 1
fi

log "✓ Base backup restored"

# ============================================================================
# PHASE 2: REJOUER L'OPLOG JUSQU'AU TARGET TIMESTAMP
# ============================================================================

log "=== Phase 2: Replaying Oplog to Target Time ==="

if [ -f "$BACKUP_DIR/oplog.bson" ]; then
  log "Replaying oplog up to $TARGET_TIMESTAMP..."

  # Créer un oplog filtré jusqu'au timestamp cible
  FILTERED_OPLOG="$TEMP_DIR/oplog_filtered.bson"

  # Filtrer l'oplog avec bsondump et mongoimport
  bsondump "$BACKUP_DIR/oplog.bson" | \
    jq -c "select(.ts.\"\$timestamp\".t <= $TARGET_TS)" | \
    while read line; do
      echo "$line" | mongoimport \
        --uri="$RESTORE_HOST" \
        --db=local \
        --collection=temp_oplog \
        --jsonArray
    done

  # Rejouer l'oplog filtré
  mongorestore \
    --uri="$RESTORE_HOST" \
    --oplogReplay \
    --oplogLimit="$TARGET_TS:0" \
    --oplogFile="$BACKUP_DIR/oplog.bson"

  if [ $? -eq 0 ]; then
    log "✓ Oplog replayed successfully up to target time"
  else
    log "ERROR: Oplog replay failed"
    exit 1
  fi
else
  log "WARNING: No oplog to replay"
fi

# ============================================================================
# PHASE 3: VALIDATION
# ============================================================================

log "=== Phase 3: Validation ==="

# Vérifier les bases de données
log "Verifying databases..."
mongo "$RESTORE_HOST" --quiet --eval "
  dbs = db.adminCommand({ listDatabases: 1 }).databases;
  dbs.forEach(function(database) {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      print('✓ Database: ' + database.name +
            ' - Size: ' + (database.sizeOnDisk / 1024 / 1024).toFixed(2) + ' MB');
    }
  });
"

# Vérifier quelques collections
log "Verifying sample data..."
mongo "$RESTORE_HOST" --quiet --eval "
  // Exemple: vérifier la collection users
  use mydb
  count = db.users.countDocuments();
  print('Users count: ' + count);

  // Vérifier qu'on ne voit pas les données après le target timestamp
  // (nécessite un champ timestamp dans vos documents)
  if (db.users.findOne({}) && db.users.findOne({}).created_at) {
    recentCount = db.users.countDocuments({
      created_at: { \$gt: new Date('$TARGET_TIMESTAMP') }
    });

    if (recentCount > 0) {
      print('⚠️  WARNING: Found ' + recentCount + ' documents after target time');
      print('    This might indicate incomplete PITR');
    } else {
      print('✓ No documents found after target time (as expected)');
    }
  }
"

# Cleanup
rm -rf "$TEMP_DIR"

log ""
log "========================================================"
log "✓ POINT-IN-TIME RECOVERY COMPLETED"
log "========================================================"
log "Database restored to: $TARGET_TIMESTAMP"
log ""
log "Important Post-Recovery Steps:"
log "  1. Verify data integrity thoroughly"
log "  2. Test critical application functionality"
log "  3. Check for missing expected data"
log "  4. Review application logs for the restored period"
log "  5. Coordinate with application team before going live"
log "========================================================"
```

### Méthode 2 : Oplog Continu (Export Streaming)

Pour un RPO minimal, exporter continuellement l'oplog :

```bash
#!/bin/bash
# continuous_oplog_export.sh

MONGO_URI="mongodb://backup-node:27017/local?replicaSet=rs0"
OPLOG_DIR="/backup/oplog/continuous"
STATE_FILE="/var/lib/mongodb_pitr/oplog_state.json"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Créer les répertoires
mkdir -p "$OPLOG_DIR"
mkdir -p "$(dirname $STATE_FILE)"

# Charger le dernier timestamp
if [ -f "$STATE_FILE" ]; then
  LAST_TS=$(jq -r '.last_timestamp' "$STATE_FILE")
  log "Resuming from timestamp: $LAST_TS"
else
  # Premier run: prendre le timestamp actuel
  LAST_TS=$(mongo "$MONGO_URI" --quiet --eval "
    db.oplog.rs.find().sort({ts: -1}).limit(1).toArray()[0].ts.toJSON().\$timestamp;
  ")
  log "Starting fresh from timestamp: $LAST_TS"
fi

# Export continu
log "Starting continuous oplog export..."

while true; do
  CURRENT_FILE="$OPLOG_DIR/oplog_$(date +%Y%m%d_%H%M%S).bson"

  # Exporter les nouvelles entrées oplog
  mongodump \
    --uri="$MONGO_URI" \
    --db=local \
    --collection=oplog.rs \
    --query="{\"ts\": {\"\$gt\": $LAST_TS}}" \
    --out="$OPLOG_DIR/temp"

  # Si nouvelles données exportées
  if [ -f "$OPLOG_DIR/temp/local/oplog.rs.bson" ]; then
    mv "$OPLOG_DIR/temp/local/oplog.rs.bson" "$CURRENT_FILE"

    # Mettre à jour le timestamp
    LAST_TS=$(bsondump "$CURRENT_FILE" | tail -1 | jq -r '.ts["$timestamp"]')

    # Sauvegarder l'état
    cat > "$STATE_FILE" <<EOF
{
  "last_timestamp": $LAST_TS,
  "last_export": "$(date -Iseconds)",
  "last_file": "$CURRENT_FILE"
}
EOF

    # Compresser
    gzip "$CURRENT_FILE"

    log "Exported oplog segment: $(basename $CURRENT_FILE).gz"
  fi

  # Cleanup
  rm -rf "$OPLOG_DIR/temp"

  # Attendre avant la prochaine itération
  sleep 300  # 5 minutes
done
```

### Méthode 3 : MongoDB Atlas PITR (Cloud Native)

Atlas offre un PITR natif sans gestion manuelle :

```bash
#!/bin/bash
# atlas_pitr_restore.sh

ATLAS_PUBLIC_KEY="your-public-key"
ATLAS_PRIVATE_KEY="your-private-key"
PROJECT_ID="your-project-id"
CLUSTER_NAME="production-cluster"
TARGET_TIMESTAMP="2024-12-08T14:37:21Z"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== MongoDB Atlas Point-in-Time Recovery ==="
log "Cluster: $CLUSTER_NAME"
log "Target: $TARGET_TIMESTAMP"

# Convertir en epoch seconds
TARGET_SECONDS=$(date -d "$TARGET_TIMESTAMP" +%s)

# Créer le job de restauration PITR
log "Creating PITR restore job..."

RESTORE_CONFIG=$(cat <<EOF
{
  "deliveryType": "automated",
  "targetClusterName": "${CLUSTER_NAME}-pitr-$(date +%Y%m%d-%H%M%S)",
  "targetGroupId": "$PROJECT_ID",
  "pointInTimeUTCSeconds": $TARGET_SECONDS
}
EOF
)

RESPONSE=$(curl -s -X POST \
  --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
  -H "Content-Type: application/json" \
  -d "$RESTORE_CONFIG" \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/restoreJobs")

JOB_ID=$(echo "$RESPONSE" | jq -r '.id')

if [ "$JOB_ID" == "null" ]; then
  log "ERROR: Failed to create restore job"
  echo "$RESPONSE" | jq .
  exit 1
fi

log "✓ Restore job created: $JOB_ID"

# Monitorer le job
log "Monitoring restore progress..."

while true; do
  STATUS=$(curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/restoreJobs/${JOB_ID}" \
    | jq -r '.deliveryType')

  case "$STATUS" in
    "COMPLETED")
      log "✓ PITR restore completed successfully"
      break
      ;;
    "FAILED")
      log "✗ PITR restore failed"
      exit 1
      ;;
    *)
      log "  Status: $STATUS (waiting...)"
      sleep 60
      ;;
  esac
done

log "✓ Atlas PITR completed"
```

## PITR pour Replica Sets

### Procédure Détaillée

```bash
#!/bin/bash
# pitr_replica_set.sh

set -euo pipefail

# Configuration
RS_NAME="rs0"
RS_MEMBERS="mongodb-1:27017,mongodb-2:27017,mongodb-3:27017"
BACKUP_DIR="/backup/mongodb/rs_backup_20241208"
TARGET_TIMESTAMP="2024-12-08T14:37:21Z"
RESTORE_DIR="/data/mongodb_pitr_restore"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Replica Set Point-in-Time Recovery"
log "========================================================"
log "Replica Set: $RS_NAME"
log "Target Time: $TARGET_TIMESTAMP"
log ""

# ============================================================================
# PHASE 1: ARRÊTER LE REPLICA SET
# ============================================================================

log "=== Phase 1: Stopping Replica Set ==="

# Arrêter chaque membre
IFS=',' read -ra MEMBERS <<< "$RS_MEMBERS"
for member in "${MEMBERS[@]}"; do
  host=$(echo $member | cut -d: -f1)
  log "Stopping member: $member"

  mongo "mongodb://$member/admin" --eval "
    db.shutdownServer();
  " || true
done

sleep 10

# ============================================================================
# PHASE 2: NETTOYER LES DATADIRS
# ============================================================================

log "=== Phase 2: Cleaning Data Directories ==="

for member in "${MEMBERS[@]}"; do
  host=$(echo $member | cut -d: -f1)
  log "Cleaning datadir on: $host"

  ssh "$host" "rm -rf /var/lib/mongodb/*"
done

# ============================================================================
# PHASE 3: RESTAURER LE BACKUP SUR UN SEUL MEMBRE (STANDALONE)
# ============================================================================

log "=== Phase 3: Restoring to First Member as Standalone ==="

PRIMARY="${MEMBERS[0]}"
PRIMARY_HOST=$(echo $PRIMARY | cut -d: -f1)

log "Restoring to: $PRIMARY"

# Restaurer le backup
mongorestore \
  --host="$PRIMARY" \
  --drop \
  --oplogReplay \
  --oplogLimit="$(date -d "$TARGET_TIMESTAMP" +%s):0" \
  --gzip \
  --dir="$BACKUP_DIR"

if [ $? -ne 0 ]; then
  log "ERROR: Restore failed"
  exit 1
fi

log "✓ Backup restored to primary member"

# ============================================================================
# PHASE 4: RECONFIGURER LE REPLICA SET
# ============================================================================

log "=== Phase 4: Reconfiguring Replica Set ==="

# Démarrer le primary en mode replica set
ssh "$PRIMARY_HOST" "systemctl start mongod"

sleep 10

# Réinitialiser le replica set avec un seul membre
log "Initializing replica set with primary only..."
mongo "mongodb://$PRIMARY/admin" <<EOF
  rs.initiate({
    _id: "$RS_NAME",
    members: [
      { _id: 0, host: "$PRIMARY" }
    ]
  });
EOF

sleep 10

# Ajouter les autres membres
log "Adding secondary members..."
for i in $(seq 1 $((${#MEMBERS[@]} - 1))); do
  member="${MEMBERS[$i]}"

  log "  Adding member: $member"

  # Démarrer le membre
  host=$(echo $member | cut -d: -f1)
  ssh "$host" "systemctl start mongod"

  sleep 5

  # Ajouter au replica set
  mongo "mongodb://$PRIMARY/admin" --eval "
    rs.add('$member');
  "

  sleep 5
done

# ============================================================================
# PHASE 5: ATTENDRE LA SYNCHRONISATION
# ============================================================================

log "=== Phase 5: Waiting for Synchronization ==="

for attempt in {1..60}; do
  HEALTHY=$(mongo "mongodb://$PRIMARY/admin" --quiet --eval "
    rs.status().members.filter(m => m.health === 1).length;
  ")

  if [ "$HEALTHY" == "${#MEMBERS[@]}" ]; then
    log "✓ All members are healthy"
    break
  fi

  log "  Syncing... ($HEALTHY/${#MEMBERS[@]} healthy)"
  sleep 10
done

# ============================================================================
# PHASE 6: VALIDATION
# ============================================================================

log "=== Phase 6: Validation ==="

mongo "mongodb://${RS_MEMBERS}/?replicaSet=${RS_NAME}" --quiet --eval "
  print('=== Replica Set Status ===');
  status = rs.status();

  status.members.forEach(function(member) {
    print('Member: ' + member.name);
    print('  State: ' + member.stateStr);
    print('  Health: ' + member.health);

    if (member.stateStr === 'SECONDARY') {
      primary = status.members.filter(m => m.stateStr === 'PRIMARY')[0];
      lag = (primary.optimeDate - member.optimeDate) / 1000;
      print('  Lag: ' + lag.toFixed(2) + 's');
    }
  });

  print('\n=== Data Verification ===');
  dbs = db.adminCommand({ listDatabases: 1 }).databases;
  dbs.forEach(function(database) {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      print('Database: ' + database.name);
      db.getSiblingDB(database.name).getCollectionNames().forEach(function(coll) {
        count = db.getSiblingDB(database.name)[coll].countDocuments();
        print('  ' + coll + ': ' + count + ' documents');
      });
    }
  });
"

log ""
log "========================================================"
log "✓ REPLICA SET PITR COMPLETED"
log "========================================================"
log "Restored to: $TARGET_TIMESTAMP"
log ""
log "Verify:"
log "  mongo \"mongodb://${RS_MEMBERS}/?replicaSet=${RS_NAME}\""
log "========================================================"
```

## PITR pour Clusters Shardés

### Complexité et Coordination

```bash
#!/bin/bash
# pitr_sharded_cluster.sh

set -euo pipefail

MONGOS_HOST="mongos:27017"
CONFIG_RS="configReplSet/config1:27019,config2:27019,config3:27019"
BACKUP_ROOT="/backup/sharded/20241208_020000"
TARGET_TIMESTAMP="2024-12-08T14:37:21Z"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Sharded Cluster Point-in-Time Recovery"
log "========================================================"
log "Target Time: $TARGET_TIMESTAMP"
log ""

TARGET_SECONDS=$(date -d "$TARGET_TIMESTAMP" +%s)

# ============================================================================
# PHASE 1: ARRÊTER LE CLUSTER
# ============================================================================

log "=== Phase 1: Stopping Sharded Cluster ==="

# Arrêter le balancer
mongo "mongodb://$MONGOS_HOST/admin" --eval "sh.stopBalancer()" || true

# Arrêter mongos
log "Stopping mongos instances..."
# [Commandes spécifiques]

# Arrêter tous les shards
log "Stopping all shards..."
# [Commandes spécifiques]

# Arrêter config servers
log "Stopping config servers..."
# [Commandes spécifiques]

sleep 10

# ============================================================================
# PHASE 2: RESTAURER CONFIG SERVERS AVEC PITR
# ============================================================================

log "=== Phase 2: Restoring Config Servers with PITR ==="

log "Restoring config servers to target timestamp..."
mongorestore \
  --uri="mongodb://$CONFIG_RS" \
  --drop \
  --oplogReplay \
  --oplogLimit="$TARGET_SECONDS:0" \
  --gzip \
  --dir="$BACKUP_ROOT/config_servers"

if [ $? -ne 0 ]; then
  log "ERROR: Config servers PITR failed"
  exit 1
fi

log "✓ Config servers restored with PITR"

# ============================================================================
# PHASE 3: RESTAURER CHAQUE SHARD AVEC PITR
# ============================================================================

log "=== Phase 3: Restoring Shards with PITR ==="

# Lister les shards depuis le backup manifest
SHARDS=$(jq -r '.components.shards[]' "$BACKUP_ROOT/MANIFEST.json")

for shard_id in $SHARDS; do
  log "Restoring shard: $shard_id with PITR"

  shard_backup_dir="$BACKUP_ROOT/shard_${shard_id}"

  # Obtenir le host du shard
  shard_host=$(mongo "mongodb://$CONFIG_RS/config" --quiet --eval "
    db.shards.findOne({_id: '$shard_id'}).host;
  ")

  primary_host=$(echo "$shard_host" | cut -d/ -f2 | cut -d, -f1)

  log "  Host: $primary_host"

  # Restaurer avec PITR
  mongorestore \
    --host="$primary_host" \
    --drop \
    --oplogReplay \
    --oplogLimit="$TARGET_SECONDS:0" \
    --gzip \
    --dir="$shard_backup_dir" &
done

# Attendre toutes les restaurations
wait

log "✓ All shards restored with PITR"

# ============================================================================
# PHASE 4: REDÉMARRER LE CLUSTER
# ============================================================================

log "=== Phase 4: Restarting Sharded Cluster ==="

# Démarrer config servers
# Démarrer shards
# Démarrer mongos
# Vérifier connectivité

log "✓ Cluster restarted"

# ============================================================================
# PHASE 5: VALIDATION
# ============================================================================

log "=== Phase 5: Validation ==="

mongo "mongodb://$MONGOS_HOST/admin" <<'EOF'
  print("=== Cluster Status ===");
  sh.status();

  print("\n=== Verifying Target Timestamp ===");
  // Vérifier qu'aucune donnée n'existe après le target timestamp
  // (nécessite champs timestamp dans vos collections)

  print("\n=== Data Verification ===");
  db.adminCommand({ listDatabases: 1 }).databases.forEach(function(db) {
    if (db.name !== 'admin' && db.name !== 'local' && db.name !== 'config') {
      print("Database: " + db.name);
    }
  });
EOF

log ""
log "========================================================"
log "✓ SHARDED CLUSTER PITR COMPLETED"
log "========================================================"
```

## Cas d'Usage Pratiques

### Scénario 1 : Suppression Accidentelle de Documents

```bash
#!/bin/bash
# recover_deleted_documents.sh

# Contexte: DELETE sans WHERE clause exécuté à 14:37:22
# Objectif: Restaurer à 14:37:21 (juste avant)

TARGET_TIME="2024-12-08T14:37:21Z"
INCIDENT_TIME="2024-12-08T14:37:22Z"

log "=== Recovery: Accidental Document Deletion ==="
log "Incident occurred at: $INCIDENT_TIME"
log "Restoring to: $TARGET_TIME (1 second before)"

# Option 1: Restauration complète dans environnement isolé
# puis export des données manquantes

RESTORE_PORT=27099
TEMP_DBPATH="/tmp/pitr_recovery"

# Démarrer MongoDB temporaire
mongod --port $RESTORE_PORT --dbpath "$TEMP_DBPATH" --fork --logpath "$TEMP_DBPATH/mongod.log"

# Restaurer avec PITR
mongorestore \
  --port $RESTORE_PORT \
  --drop \
  --oplogReplay \
  --oplogLimit="$(date -d "$TARGET_TIME" +%s):0" \
  --gzip \
  --archive=/backup/latest.gz

# Exporter les documents qui ont été supprimés
mongoexport \
  --port $RESTORE_PORT \
  --db=mydb \
  --collection=orders \
  --out=/tmp/recovered_orders.json

# Ré-importer dans la production
mongoimport \
  --uri="mongodb://production:27017" \
  --db=mydb \
  --collection=orders \
  --file=/tmp/recovered_orders.json \
  --mode=upsert

# Cleanup
mongod --port $RESTORE_PORT --shutdown
rm -rf "$TEMP_DBPATH"

log "✓ Deleted documents recovered and restored to production"
```

### Scénario 2 : Corruption de Données par Bug Applicatif

```bash
#!/bin/bash
# recover_from_data_corruption.sh

# Contexte: Bug applicatif a corrompu les prix depuis 10:00
# Objectif: Identifier la dernière version propre et restaurer

SUSPECTED_BUG_TIME="2024-12-08T10:00:00Z"
SAFE_TIME="2024-12-08T09:59:59Z"

log "=== Recovery: Data Corruption from Application Bug ==="

# Stratégie: Restaurer dans environnement test, valider, puis migrer

# 1. Créer cluster de test
log "Creating test cluster for validation..."

# 2. Restaurer au point sûr
log "Restoring to safe point: $SAFE_TIME"
# [Restauration PITR]

# 3. Valider les données
log "Validating restored data..."
mongo "mongodb://test-cluster:27017" <<'EOF'
  use mydb

  // Vérifier les prix sont cohérents
  invalidPrices = db.products.find({
    price: { $lte: 0 }
  }).count();

  if (invalidPrices > 0) {
    print("ERROR: Found " + invalidPrices + " invalid prices");
    quit(1);
  }

  // Vérifier la plage de prix attendue
  stats = db.products.aggregate([
    {
      $group: {
        _id: null,
        min: { $min: "$price" },
        max: { $max: "$price" },
        avg: { $avg: "$price" }
      }
    }
  ]).toArray()[0];

  print("Price statistics:");
  print("  Min: $" + stats.min);
  print("  Max: $" + stats.max);
  print("  Avg: $" + stats.avg.toFixed(2));

  if (stats.min < 0.01 || stats.max > 100000) {
    print("WARNING: Price range seems unusual");
  } else {
    print("✓ Price data looks valid");
  }
EOF

# 4. Si validation OK, synchroniser vers production
log "Syncing validated data to production..."

# Export des données corrigées
mongoexport \
  --uri="mongodb://test-cluster:27017" \
  --db=mydb \
  --collection=products \
  --out=/tmp/clean_products.json

# Import dans production avec upsert
mongoimport \
  --uri="mongodb://production:27017" \
  --db=mydb \
  --collection=products \
  --file=/tmp/clean_products.json \
  --mode=upsert \
  --upsertFields="_id"

log "✓ Data corruption recovered"
```

### Scénario 3 : Investigation Forensique

```bash
#!/bin/bash
# forensic_investigation.sh

# Contexte: Activité suspecte détectée, besoin d'analyser l'état à différents moments

INVESTIGATION_TIMES=(
  "2024-12-08T12:00:00Z"
  "2024-12-08T13:00:00Z"
  "2024-12-08T14:00:00Z"
  "2024-12-08T15:00:00Z"
)

log "=== Forensic Investigation: Multiple Point-in-Time Analysis ==="

for target_time in "${INVESTIGATION_TIMES[@]}"; do
  log ""
  log "Analyzing state at: $target_time"

  # Créer un snapshot PITR pour ce moment
  snapshot_dir="/forensics/snapshot_$(echo $target_time | tr -d ':-' | cut -d'T' -f1-2)"

  # Restaurer dans environnement isolé
  mongorestore \
    --port=27100 \
    --dbpath="$snapshot_dir" \
    --oplogReplay \
    --oplogLimit="$(date -d "$target_time" +%s):0" \
    --gzip \
    --archive=/backup/investigation_base.gz

  # Analyser les anomalies
  mongo --port=27100 <<EOF
    use audit

    // Analyser les connexions
    connections = db.connection_logs.find({
      timestamp: {
        \$gte: new Date('${target_time}'),
        \$lt: new Date('${target_time}').setHours(new Date('${target_time}').getHours() + 1)
      }
    }).toArray();

    print("Connections at ${target_time}: " + connections.length);

    // Analyser les opérations suspectes
    suspicious = db.operations.find({
      timestamp: { \$gte: new Date('${target_time}') },
      type: { \$in: ['DELETE', 'DROP'] },
      user: { \$ne: 'admin' }
    }).toArray();

    print("Suspicious operations: " + suspicious.length);

    if (suspicious.length > 0) {
      print("Details:");
      suspicious.forEach(printjson);
    }
EOF

  # Générer rapport
  echo "=== Report for $target_time ===" >> /forensics/investigation_report.txt
  # [Ajouter résultats de l'analyse]
done

log "✓ Forensic investigation completed"
log "Report: /forensics/investigation_report.txt"
```

## Automatisation et Tests

### Script de Test PITR Régulier

```bash
#!/bin/bash
# test_pitr_capability.sh

set -euo pipefail

TEST_CLUSTER="mongodb://test-pitr:27017"
BACKUP_FILE="/backup/latest_prod_backup.gz"
LOG_FILE="/var/log/pitr_test_$(date +%Y%m%d).log"

exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Automated PITR Capability Test"
log "========================================================"

# Choisir un timestamp aléatoire dans les dernières 24h
HOURS_AGO=$((RANDOM % 24 + 1))
TARGET_TIME=$(date -d "$HOURS_AGO hours ago" -Iseconds)

log "Testing PITR to: $TARGET_TIME"

# ============================================================================
# PHASE 1: RESTAURATION PITR
# ============================================================================

log "=== Phase 1: PITR Restoration ==="

START_TIME=$(date +%s)

mongorestore \
  --uri="$TEST_CLUSTER" \
  --drop \
  --oplogReplay \
  --oplogLimit="$(date -d "$TARGET_TIME" +%s):0" \
  --gzip \
  --archive="$BACKUP_FILE"

RESTORE_EXIT_CODE=$?
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

if [ $RESTORE_EXIT_CODE -ne 0 ]; then
  log "✗ PITR restoration FAILED"
  exit 1
fi

log "✓ PITR restoration completed in ${DURATION}s"

# ============================================================================
# PHASE 2: VALIDATION DES DONNÉES
# ============================================================================

log "=== Phase 2: Data Validation ==="

# Test 1: Vérifier les bases de données
DB_COUNT=$(mongo "$TEST_CLUSTER" --quiet --eval "
  db.adminCommand({ listDatabases: 1 }).databases.length;
")

log "Databases found: $DB_COUNT"

if [ "$DB_COUNT" -lt 1 ]; then
  log "✗ No databases found after restore"
  exit 1
fi

# Test 2: Vérifier quelques collections
mongo "$TEST_CLUSTER" --quiet --eval "
  use production_db

  // Compter les documents
  usersCount = db.users.countDocuments();
  ordersCount = db.orders.countDocuments();

  print('Users: ' + usersCount);
  print('Orders: ' + ordersCount);

  if (usersCount === 0 || ordersCount === 0) {
    print('ERROR: Expected collections are empty');
    quit(1);
  }

  // Vérifier l'intégrité
  validation = db.orders.validate({ full: true });
  if (!validation.valid) {
    print('ERROR: Collection validation failed');
    quit(1);
  }

  print('✓ Data validation passed');
"

VALIDATION_EXIT_CODE=$?

if [ $VALIDATION_EXIT_CODE -ne 0 ]; then
  log "✗ Data validation FAILED"
  exit 1
fi

log "✓ Data validation passed"

# ============================================================================
# PHASE 3: VÉRIFICATION TEMPORELLE
# ============================================================================

log "=== Phase 3: Temporal Verification ==="

# Vérifier qu'aucune donnée n'existe après le target timestamp
mongo "$TEST_CLUSTER" --quiet --eval "
  use production_db

  // Vérifier les documents avec timestamp
  if (db.orders.findOne({}) && db.orders.findOne({}).created_at) {
    recentCount = db.orders.countDocuments({
      created_at: { \$gt: new Date('$TARGET_TIME') }
    });

    if (recentCount > 0) {
      print('WARNING: Found ' + recentCount + ' documents after target time');
      print('This indicates PITR did not work correctly');
      quit(1);
    } else {
      print('✓ No documents found after target timestamp');
    }
  } else {
    print('SKIP: Collections do not have timestamp field');
  }
"

TEMPORAL_EXIT_CODE=$?

if [ $TEMPORAL_EXIT_CODE -ne 0 ]; then
  log "⚠️  Temporal verification had issues"
fi

# ============================================================================
# RAPPORT
# ============================================================================

log ""
log "========================================================"
log "  PITR TEST RESULTS"
log "========================================================"
log "Target Time: $TARGET_TIME"
log "Restore Duration: ${DURATION}s"
log "Databases Restored: $DB_COUNT"
log "Status: SUCCESS"
log "========================================================"

# Envoyer notification
curl -X POST "$SLACK_WEBHOOK" -H 'Content-Type: application/json' -d "{
  \"text\": \"✓ PITR Test Passed\",
  \"attachments\": [{
    \"color\": \"good\",
    \"fields\": [
      {\"title\": \"Target Time\", \"value\": \"$TARGET_TIME\", \"short\": true},
      {\"title\": \"Duration\", \"value\": \"${DURATION}s\", \"short\": true},
      {\"title\": \"Status\", \"value\": \"SUCCESS\", \"short\": false}
    ]
  }]
}"

log "✓ PITR capability test completed successfully"
```

### Cron pour Tests Réguliers

```bash
# /etc/cron.d/mongodb-pitr-test

# Test PITR hebdomadaire le dimanche à 4h
0 4 * * 0 mongodb /usr/local/bin/test_pitr_capability.sh >> /var/log/pitr_test_cron.log 2>&1

# Notification si échec
5 4 * * 0 mongodb /usr/local/bin/check_pitr_test_result.sh
```

## Limitations et Considérations

### Limitations Techniques

```yaml
Limitation 1 - Fenêtre Oplog:
  problème: "PITR impossible au-delà de la fenêtre oplog"
  solution:
    - Dimensionner l'oplog adequatement (72h min prod)
    - Exporter continuellement l'oplog
    - Utiliser Atlas avec rétention longue

  commande_verification: |
    use local
    first = db.oplog.rs.find().sort({$natural: 1}).limit(1).toArray()[0].ts;
    last = db.oplog.rs.find().sort({$natural: -1}).limit(1).toArray()[0].ts;
    hours = (last.getTime() - first.getTime()) / 3600;
    print("Oplog window: " + hours + " hours");

Limitation 2 - Performance sur Gros Volumes:
  problème: "Replay oplog peut être très long (> 1TB)"
  impact: "RTO augmenté significativement"
  solution:
    - Snapshots plus fréquents (réduire oplog à rejouer)
    - Utiliser mongorestore parallèle
    - Considérer Atlas pour très gros volumes

Limitation 3 - Opérations Non-Rejouables:
  problème: "Certaines opérations ne sont pas dans l'oplog"
  exemples:
    - Opérations administratives directes
    - Modifications de configuration
    - Certaines commandes système
  solution: "Documenter et re-appliquer manuellement"

Limitation 4 - Cohérence Cross-Database:
  problème: "PITR par database peut casser cohérence"
  exemple: "Transaction multi-db ou collections référencées"
  solution: "PITR toujours global au niveau cluster"
```

### Considérations de Performance

```bash
# Mesurer le temps de replay oplog
measure_oplog_replay_time() {
  local oplog_size=$(mongo --quiet --eval "
    db.getSiblingDB('local').oplog.rs.stats().size;
  ")

  # Estimation: ~1GB par minute de replay
  local estimated_minutes=$((oplog_size / 1024 / 1024 / 1024))

  echo "Oplog size: $(numfmt --to=iec-i --suffix=B $oplog_size)"
  echo "Estimated replay time: ~${estimated_minutes} minutes"

  # Si > 60 min, recommander stratégie alternative
  if [ $estimated_minutes -gt 60 ]; then
    echo "⚠️  WARNING: Estimated replay time exceeds 1 hour"
    echo "Consider:"
    echo "  - More frequent backups"
    echo "  - Snapshot-based PITR"
    echo "  - MongoDB Atlas"
  fi
}
```

## Bonnes Pratiques

### Checklist PITR

```markdown
### Préparation Infrastructure

- [ ] Oplog dimensionné adequatement (72h min pour prod)
- [ ] Export continu de l'oplog configuré (pour RPO minimal)
- [ ] Backups réguliers avec --oplog activé
- [ ] Documentation de la topologie complète
- [ ] Procédures PITR testées et validées

### Documentation Requise

- [ ] Topologie complète du cluster
- [ ] Mapping des hostnames et ports
- [ ] Credentials de restauration
- [ ] Dépendances applicatives
- [ ] Points de contact (oncall, équipes)

### Tests Réguliers

- [ ] Test PITR mensuel sur environnement isolé
- [ ] Validation temporelle (pas de données après target)
- [ ] Mesure du RTO réel
- [ ] Documentation des résultats
- [ ] Amélioration continue des procédures

### Pendant un Incident

- [ ] Identifier le timestamp exact du problème
- [ ] Déterminer le timestamp cible (juste avant)
- [ ] Vérifier la disponibilité du backup et oplog
- [ ] Coordonner avec les équipes applicatives
- [ ] Communiquer l'ETA de récupération
- [ ] Documenter l'incident et la récupération

### Post-Récupération

- [ ] Validation exhaustive des données
- [ ] Tests fonctionnels applicatifs
- [ ] Vérification absence de données corrompues
- [ ] Communication aux stakeholders
- [ ] Post-mortem et lessons learned
- [ ] Mise à jour des procédures si nécessaire
```

### Recommandations par Criticité

```yaml
Development:
  oplog_size: 24 heures minimum
  backup_frequency: quotidien
  pitr_window: 24 heures
  test_frequency: jamais requis

Production Standard:
  oplog_size: 48 heures minimum
  backup_frequency: toutes les 6 heures
  pitr_window: 48 heures
  test_frequency: mensuel
  export_continu: recommandé

Production Critique:
  oplog_size: 72 heures minimum
  backup_frequency: toutes les 4 heures
  pitr_window: 7 jours
  test_frequency: hebdomadaire
  export_continu: obligatoire
  solution: MongoDB Atlas (PITR natif)

Mission-Critical:
  oplog_size: 168 heures (7 jours)
  backup_frequency: toutes les heures
  pitr_window: 30 jours
  test_frequency: hebdomadaire
  export_continu: obligatoire
  solution: MongoDB Atlas + export additionnel
  dr_site: requis avec PITR indépendant
```

## Conclusion

Le Point-in-Time Recovery est une capacité essentielle pour tout environnement MongoDB de production, offrant la granularité nécessaire pour minimiser la perte de données lors d'incidents.

**Points clés à retenir** :

1. **L'oplog est central** - Dimensionner adequatement et exporter continuellement
2. **Tests réguliers** - Valider la capacité PITR mensuellement minimum
3. **Documentation précise** - Timestamps exacts et procédures détaillées
4. **Automatisation** - Scripts robustes pour réduire erreurs humaines
5. **Atlas simplifie** - PITR natif sans gestion complexe

**Quand utiliser PITR** :
- Erreurs humaines (suppression, modification incorrecte)
- Bugs applicatifs corrupteurs
- Besoin forensique (analyser état passé)
- Compliance (prouver état données à date donnée)

**Alternatives si PITR insuffisant** :
- Change Streams pour capture temps réel
- Réplication vers site DR avec delay
- Snapshots plus fréquents
- Solutions SaaS comme MongoDB Atlas

Le PITR n'est pas une solution miracle mais un outil puissant dans l'arsenal de continuité d'activité, à combiner avec d'autres stratégies pour une protection complète des données.

---


⏭️ [Oplog pour la récupération](/12-sauvegarde-restauration/08-oplog-recuperation.md)
