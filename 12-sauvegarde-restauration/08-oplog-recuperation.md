🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.8 Oplog pour la Récupération

## Introduction

L'oplog (operations log) est le mécanisme fondamental qui permet à MongoDB de garantir la cohérence des données et de supporter la récupération point-in-time. Contrairement aux backups traditionnels qui capturent un état statique, l'oplog enregistre en continu toutes les opérations d'écriture, permettant de reconstruire l'état de la base à n'importe quel moment dans le passé.

Cette section explore en profondeur l'utilisation de l'oplog pour la récupération, les techniques d'extraction, de manipulation et de rejeu, ainsi que les stratégies avancées pour maximiser les capacités de récupération.

## Architecture et Fonctionnement de l'Oplog

### Structure Interne de l'Oplog

```
┌────────────────────────────────────────────────────────────┐
│                    Oplog Architecture                      │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │           local.oplog.rs (Capped Collection)         │  │
│  │                                                      │  │
│  │  Propriétés:                                         │  │
│  │  • Collection de taille fixe (capped)                │  │
│  │  • FIFO automatique (écrase anciennes entrées)       │  │
│  │  • Ordonnancement strict (timestamp)                 │  │
│  │  • Idempotent (rejeu safe)                           │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                 │
│                          v                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Oplog Entry Structure                   │  │
│  │                                                      │  │
│  │  {                                                   │  │
│  │    "ts": Timestamp(1701993600, 1),  ← Unique ID      │  │
│  │    "t": NumberLong(23),             ← Term           │  │
│  │    "h": NumberLong("123..."),       ← Hash           │  │
│  │    "v": 2,                          ← Version        │  │
│  │    "op": "i|u|d|c|n",              ← Operation       │  │
│  │    "ns": "db.collection",          ← Namespace       │  │
│  │    "ui": UUID("..."),               ← Collection ID  │  │
│  │    "o": {...},                      ← Operation doc  │  │
│  │    "o2": {...}                      ← Query (update) │  │
│  │  }                                                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                          │                                 │
│                          v                                 │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            Operation Types & Semantics               │  │
│  │                                                      │  │
│  │  "i" (insert):     o = document complet              │  │
│  │  "u" (update):     o = modification, o2 = query      │  │
│  │  "d" (delete):     o = {_id: ...}                    │  │
│  │  "c" (command):    o = {drop: "coll"}                │  │
│  │  "n" (no-op):      Heartbeat / placeholder           │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Caractéristiques Idempotentes

L'oplog est conçu pour être idempotent - rejouer une opération plusieurs fois produit le même résultat :

```javascript
// Exemple d'idempotence

// Opération NON-idempotente (application)
db.counters.update(
  { _id: "pageviews" },
  { $inc: { count: 1 } }  // Incrémente à chaque fois
)

// Opération idempotente (oplog)
{
  "op": "u",
  "ns": "mydb.counters",
  "o": {
    "$v": 2,
    "$set": { "count": 42 }  // SET absolu, pas INC
  },
  "o2": { "_id": "pageviews" }
}

// Rejouer 10 fois → count = 42 toujours
// C'est cette transformation qui garantit l'idempotence
```

### Cycle de Vie de l'Oplog

```
Timeline de l'Oplog (72 heures configuré):

T0 (maintenant)
│
├─ Oplog HEAD (dernière opération)
│  ts: Timestamp(1701993600, 999)
│
├─ T-24h: Encore dans l'oplog
│  ts: Timestamp(1701907200, ...)
│  ✓ Peut être utilisé pour PITR
│
├─ T-48h: Encore dans l'oplog
│  ts: Timestamp(1701820800, ...)
│  ✓ Peut être utilisé pour PITR
│
├─ T-72h: Bord de l'oplog
│  ts: Timestamp(1701734400, ...)
│  ⚠️  Près de l'expiration
│
└─ T-73h: ÉCRASÉ (oplog full)
   ✗ Ne peut plus être utilisé
   ✗ PITR impossible au-delà

Stratégies:
1. Dimensionner l'oplog adequatement
2. Exporter continuellement les anciennes entrées
3. Alerter quand fenêtre < seuil
```

## Extraction et Export de l'Oplog

### Export Basique avec mongodump

```bash
#!/bin/bash
# export_oplog_basic.sh

MONGO_HOST="mongodb://backup-secondary:27017"
OPLOG_DIR="/backup/oplog"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Oplog Export ==="

# Exporter l'oplog complet
mongodump \
  --uri="$MONGO_HOST" \
  --db=local \
  --collection=oplog.rs \
  --out="$OPLOG_DIR/export_$TIMESTAMP"

if [ $? -eq 0 ]; then
  # Compresser
  gzip "$OPLOG_DIR/export_$TIMESTAMP/local/oplog.rs.bson"

  # Stats
  OPLOG_SIZE=$(du -h "$OPLOG_DIR/export_$TIMESTAMP/local/oplog.rs.bson.gz" | cut -f1)
  OPLOG_ENTRIES=$(bsondump "$OPLOG_DIR/export_$TIMESTAMP/local/oplog.rs.bson.gz" | wc -l)

  log "✓ Oplog exported"
  log "  Size: $OPLOG_SIZE"
  log "  Entries: $OPLOG_ENTRIES"
else
  log "✗ Oplog export failed"
  exit 1
fi
```

### Export Incrémental avec État Persistant

```bash
#!/bin/bash
# export_oplog_incremental.sh

set -euo pipefail

MONGO_URI="mongodb://backup-secondary:27017/local?replicaSet=rs0"
OPLOG_DIR="/backup/oplog/incremental"
STATE_FILE="/var/lib/mongodb_backup/oplog_state.json"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

mkdir -p "$OPLOG_DIR"
mkdir -p "$(dirname $STATE_FILE)"

# ============================================================================
# CHARGER LE DERNIER TIMESTAMP
# ============================================================================

if [ -f "$STATE_FILE" ]; then
  LAST_TS=$(jq -r '.last_timestamp.t' "$STATE_FILE")
  LAST_TS_INC=$(jq -r '.last_timestamp.i' "$STATE_FILE")

  log "Resuming from timestamp: Timestamp($LAST_TS, $LAST_TS_INC)"
else
  # Premier run: obtenir le timestamp actuel
  CURRENT_TS=$(mongo "$MONGO_URI" --quiet --eval "
    lastOp = db.oplog.rs.find().sort({ts: -1}).limit(1).toArray()[0];
    printjson({t: lastOp.ts.getTime(), i: lastOp.ts.getInc()});
  ")

  LAST_TS=$(echo "$CURRENT_TS" | jq -r '.t')
  LAST_TS_INC=$(echo "$CURRENT_TS" | jq -r '.i')

  log "Starting fresh from: Timestamp($LAST_TS, $LAST_TS_INC)"
fi

# ============================================================================
# EXPORTER LES NOUVELLES ENTRÉES
# ============================================================================

OUTPUT_FILE="$OPLOG_DIR/oplog_$(date +%Y%m%d_%H%M%S).bson"

log "Exporting new oplog entries..."

mongo "$MONGO_URI" --quiet --eval "
  // Construire le query
  lastTs = Timestamp($LAST_TS, $LAST_TS_INC);

  // Exporter vers fichier temporaire
  cursor = db.oplog.rs.find({
    ts: { \$gt: lastTs }
  }).sort({ ts: 1 });

  count = 0;
  cursor.forEach(function(doc) {
    count++;
  });

  print('Entries to export: ' + count);
"

# Export via mongodump avec query
mongodump \
  --uri="$MONGO_URI" \
  --db=local \
  --collection=oplog.rs \
  --query="{\"ts\": {\"\$gt\": {\"t\": $LAST_TS, \"i\": $LAST_TS_INC}}}" \
  --out="$OPLOG_DIR/temp"

if [ -f "$OPLOG_DIR/temp/local/oplog.rs.bson" ]; then
  mv "$OPLOG_DIR/temp/local/oplog.rs.bson" "$OUTPUT_FILE"

  # Obtenir le nouveau dernier timestamp
  NEW_LAST_TS=$(bsondump "$OUTPUT_FILE" | tail -1 | jq -r '.ts["$timestamp"]')
  NEW_LAST_TS_T=$(echo "$NEW_LAST_TS" | jq -r '.t')
  NEW_LAST_TS_I=$(echo "$NEW_LAST_TS" | jq -r '.i')

  # Sauvegarder l'état
  cat > "$STATE_FILE" <<EOF
{
  "last_timestamp": {
    "t": $NEW_LAST_TS_T,
    "i": $NEW_LAST_TS_I
  },
  "last_export": "$(date -Iseconds)",
  "last_file": "$OUTPUT_FILE",
  "entries_exported": $(bsondump "$OUTPUT_FILE" | wc -l)
}
EOF

  # Compresser
  gzip "$OUTPUT_FILE"

  log "✓ Incremental export completed"
  log "  File: $(basename $OUTPUT_FILE).gz"
  log "  Entries: $(jq -r '.entries_exported' $STATE_FILE)"
  log "  New timestamp: Timestamp($NEW_LAST_TS_T, $NEW_LAST_TS_I)"
else
  log "No new entries to export"
fi

# Cleanup
rm -rf "$OPLOG_DIR/temp"
```

### Export Streaming en Temps Réel (Node.js)

Pour un RPO minimal, streamer l'oplog continuellement :

```javascript
// oplog_streaming_export.js

const { MongoClient } = require('mongodb');
const fs = require('fs');
const zlib = require('zlib');

const MONGO_URI = 'mongodb://backup-secondary:27017/?replicaSet=rs0';
const OUTPUT_DIR = '/backup/oplog/streaming';
const STATE_FILE = '/var/lib/mongodb_backup/streaming_state.json';
const ROTATION_INTERVAL = 3600000; // 1 heure
const COMPRESSION_LEVEL = 6;

let currentStream = null;
let currentFile = null;
let entriesWritten = 0;
let lastTimestamp = null;

// Charger le dernier timestamp
function loadLastTimestamp() {
  try {
    if (fs.existsSync(STATE_FILE)) {
      const state = JSON.parse(fs.readFileSync(STATE_FILE));
      console.log(`[INFO] Resuming from timestamp: ${JSON.stringify(state.last_timestamp)}`);
      return state.last_timestamp;
    }
  } catch (err) {
    console.error(`[ERROR] Failed to load state: ${err.message}`);
  }
  return null;
}

// Sauvegarder l'état
function saveState() {
  if (lastTimestamp) {
    const state = {
      last_timestamp: lastTimestamp,
      last_update: new Date().toISOString(),
      current_file: currentFile,
      entries_written: entriesWritten
    };

    fs.writeFileSync(STATE_FILE, JSON.stringify(state, null, 2));
  }
}

// Rotation de fichier
function rotateFile() {
  if (currentStream) {
    currentStream.end();
  }

  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  currentFile = `${OUTPUT_DIR}/oplog_${timestamp}.bson.gz`;

  currentStream = fs.createWriteStream(currentFile)
    .pipe(zlib.createGzip({ level: COMPRESSION_LEVEL }));

  entriesWritten = 0;

  console.log(`[INFO] Rotated to new file: ${currentFile}`);
}

// Démarrer le streaming
async function startStreaming() {
  const client = await MongoClient.connect(MONGO_URI);
  const db = client.db('local');
  const oplog = db.collection('oplog.rs');

  // Charger le dernier timestamp
  const resumeTimestamp = loadLastTimestamp();

  // Construire le pipeline
  const pipeline = [
    {
      $match: {
        ts: resumeTimestamp ? { $gt: resumeTimestamp } : { $exists: true }
      }
    }
  ];

  // Créer le premier fichier
  rotateFile();

  // Configurer la rotation automatique
  setInterval(rotateFile, ROTATION_INTERVAL);

  // Sauvegarder l'état toutes les 30 secondes
  setInterval(saveState, 30000);

  // Watcher le change stream (MongoDB 3.6+)
  const changeStream = oplog.watch(pipeline, {
    fullDocument: 'updateLookup'
  });

  console.log('[INFO] Oplog streaming started');

  // Traiter chaque entrée
  changeStream.on('change', (change) => {
    try {
      // Écrire dans le fichier compressé
      const bson = require('bson').serialize(change);
      currentStream.write(bson);

      entriesWritten++;
      lastTimestamp = change.ts;

      // Log périodique
      if (entriesWritten % 1000 === 0) {
        console.log(`[INFO] Streamed ${entriesWritten} entries`);
      }
    } catch (err) {
      console.error(`[ERROR] Failed to write entry: ${err.message}`);
    }
  });

  changeStream.on('error', (err) => {
    console.error(`[ERROR] Change stream error: ${err.message}`);
    // Reconnection automatique géré par le driver
  });

  // Gestion des signaux
  process.on('SIGINT', () => {
    console.log('[INFO] Shutting down gracefully...');
    saveState();
    changeStream.close();
    client.close();
    process.exit(0);
  });
}

// Démarrer
startStreaming().catch(err => {
  console.error(`[FATAL] Failed to start streaming: ${err.message}`);
  process.exit(1);
});
```

## Rejeu Manuel de l'Oplog

### Rejeu Complet

```bash
#!/bin/bash
# replay_oplog.sh

set -euo pipefail

OPLOG_FILE="/backup/oplog/oplog_20241208_140000.bson.gz"
TARGET_HOST="mongodb://restore-target:27017"
TARGET_DB=""  # Vide = toutes les bases

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Oplog Replay"
log "========================================================"
log "Source: $OPLOG_FILE"
log "Target: $TARGET_HOST"
log ""

# Vérifier que le fichier existe
if [ ! -f "$OPLOG_FILE" ]; then
  log "ERROR: Oplog file not found"
  exit 1
fi

# Décompresser si nécessaire
TEMP_OPLOG="/tmp/oplog_replay_$$.bson"
if [[ "$OPLOG_FILE" == *.gz ]]; then
  log "Decompressing oplog..."
  gunzip -c "$OPLOG_FILE" > "$TEMP_OPLOG"
else
  cp "$OPLOG_FILE" "$TEMP_OPLOG"
fi

# Analyser l'oplog
log "Analyzing oplog..."
FIRST_TS=$(bsondump "$TEMP_OPLOG" | head -1 | jq -r '.ts["$timestamp"]')
LAST_TS=$(bsondump "$TEMP_OPLOG" | tail -1 | jq -r '.ts["$timestamp"]')
TOTAL_OPS=$(bsondump "$TEMP_OPLOG" | wc -l)

log "Oplog summary:"
log "  First timestamp: $(date -d @$(echo $FIRST_TS | jq -r '.t') -Iseconds)"
log "  Last timestamp: $(date -d @$(echo $LAST_TS | jq -r '.t') -Iseconds)"
log "  Total operations: $TOTAL_OPS"

read -p "Proceed with replay? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
  log "Replay cancelled"
  rm -f "$TEMP_OPLOG"
  exit 0
fi

# ============================================================================
# REPLAY
# ============================================================================

log "Starting oplog replay..."
START_TIME=$(date +%s)

# Option 1: Via mongorestore (recommandé)
mongorestore \
  --uri="$TARGET_HOST" \
  --oplogReplay \
  --oplogFile="$TEMP_OPLOG" \
  ${TARGET_DB:+--nsInclude="${TARGET_DB}.*"}

REPLAY_EXIT=$?
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

if [ $REPLAY_EXIT -eq 0 ]; then
  log "✓ Oplog replay completed in ${DURATION}s"
else
  log "✗ Oplog replay failed"
  rm -f "$TEMP_OPLOG"
  exit 1
fi

# Cleanup
rm -f "$TEMP_OPLOG"

log "✓ Replay completed successfully"
```

### Rejeu Filtré par Namespace

```bash
#!/bin/bash
# replay_oplog_filtered.sh

OPLOG_FILE="/backup/oplog/oplog_full.bson"
TARGET_HOST="mongodb://localhost:27017"
FILTER_NAMESPACE="mydb.important_collection"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Filtered Oplog Replay ==="
log "Namespace filter: $FILTER_NAMESPACE"

# Créer un oplog filtré
FILTERED_OPLOG="/tmp/oplog_filtered_$$.bson"

log "Filtering oplog entries..."
bsondump "$OPLOG_FILE" | \
  jq -c "select(.ns == \"$FILTER_NAMESPACE\")" | \
  while read -r line; do
    echo "$line" | mongoimport \
      --uri="mongodb://temp:27017/local" \
      --collection=temp_oplog \
      --jsonArray 2>/dev/null
  done

# Exporter le filtered oplog
mongodump \
  --uri="mongodb://temp:27017" \
  --db=local \
  --collection=temp_oplog \
  --out=/tmp/filtered_export

mv /tmp/filtered_export/local/temp_oplog.bson "$FILTERED_OPLOG"

# Rejouer
log "Replaying filtered oplog..."
mongorestore \
  --uri="$TARGET_HOST" \
  --oplogReplay \
  --oplogFile="$FILTERED_OPLOG"

# Cleanup
rm -f "$FILTERED_OPLOG"
rm -rf /tmp/filtered_export

log "✓ Filtered replay completed"
```

### Rejeu Jusqu'à un Timestamp Spécifique

```bash
#!/bin/bash
# replay_oplog_until_timestamp.sh

OPLOG_FILE="/backup/oplog/oplog_full.bson.gz"
TARGET_HOST="mongodb://localhost:27017"
STOP_TIMESTAMP="2024-12-08T14:37:21Z"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Oplog Replay Until Timestamp ==="
log "Stop at: $STOP_TIMESTAMP"

# Convertir en epoch
STOP_EPOCH=$(date -d "$STOP_TIMESTAMP" +%s)

# Créer oplog tronqué
TRUNCATED_OPLOG="/tmp/oplog_truncated_$$.bson"

log "Truncating oplog at timestamp..."
bsondump "$OPLOG_FILE" | \
  jq -c "select(.ts.\"\$timestamp\".t <= $STOP_EPOCH)" > "$TRUNCATED_OPLOG.json"

# Convertir en BSON
cat "$TRUNCATED_OPLOG.json" | \
  mongoimport \
    --uri="mongodb://temp:27017/local" \
    --collection=temp_oplog \
    --jsonArray

# Export en BSON
mongodump \
  --uri="mongodb://temp:27017" \
  --db=local \
  --collection=temp_oplog \
  --out=/tmp/truncated_export

mv /tmp/truncated_export/local/temp_oplog.bson "$TRUNCATED_OPLOG"

# Rejouer
log "Replaying truncated oplog..."
mongorestore \
  --uri="$TARGET_HOST" \
  --oplogReplay \
  --oplogFile="$TRUNCATED_OPLOG"

# Cleanup
rm -f "$TRUNCATED_OPLOG" "$TRUNCATED_OPLOG.json"
rm -rf /tmp/truncated_export

log "✓ Replay stopped at $STOP_TIMESTAMP"
```

## Gestion de la Taille de l'Oplog

### Calculer la Taille Optimale

```javascript
// calculate_optimal_oplog_size.js

use local

// Statistiques actuelles
var oplogStats = db.oplog.rs.stats();
var oplogSizeMB = oplogStats.maxSize / 1024 / 1024;
var usedMB = oplogStats.size / 1024 / 1024;

print("=== Current Oplog Statistics ===");
print("Max Size: " + oplogSizeMB.toFixed(2) + " MB");
print("Used: " + usedMB.toFixed(2) + " MB");
print("Used %: " + (usedMB / oplogSizeMB * 100).toFixed(2) + "%");

// Calculer la fenêtre temporelle
var firstEntry = db.oplog.rs.find().sort({$natural: 1}).limit(1).toArray()[0];
var lastEntry = db.oplog.rs.find().sort({$natural: -1}).limit(1).toArray()[0];

var firstTime = firstEntry.ts.getTime();
var lastTime = lastEntry.ts.getTime();
var windowHours = (lastTime - firstTime) / 3600;

print("\n=== Time Window ===");
print("Current window: " + windowHours.toFixed(2) + " hours");
print("First entry: " + new Date(firstTime * 1000).toISOString());
print("Last entry: " + new Date(lastTime * 1000).toISOString());

// Calculer le taux d'écriture
var writesPerHour = db.oplog.rs.count() / windowHours;
var mbPerHour = usedMB / windowHours;

print("\n=== Write Rate ===");
print("Operations per hour: " + writesPerHour.toFixed(0));
print("MB per hour: " + mbPerHour.toFixed(2));

// Recommandations
print("\n=== Recommendations ===");

var targetWindows = {
  "Development (24h)": 24,
  "Production (48h)": 48,
  "Production Critical (72h)": 72,
  "Mission Critical (168h)": 168
};

Object.keys(targetWindows).forEach(function(profile) {
  var hours = targetWindows[profile];
  var requiredMB = mbPerHour * hours;
  var requiredGB = requiredMB / 1024;

  print("\n" + profile + ":");
  print("  Required oplog size: " + requiredGB.toFixed(2) + " GB");

  if (requiredMB > oplogSizeMB) {
    print("  ⚠️  Current oplog is TOO SMALL");
    print("  Action: Increase to " + Math.ceil(requiredGB) + " GB");
  } else {
    print("  ✓ Current oplog is adequate");
  }
});

// Recommandation avec marge de sécurité (20%)
var recommendedMB = mbPerHour * 72 * 1.2; // 72h + 20% margin
var recommendedGB = recommendedMB / 1024;

print("\n=== Final Recommendation ===");
print("Recommended oplog size: " + Math.ceil(recommendedGB) + " GB");
print("(Based on 72h window + 20% safety margin)");
```

### Redimensionner l'Oplog

```bash
#!/bin/bash
# resize_oplog.sh

set -euo pipefail

MONGO_URI="mongodb://localhost:27017/admin"
NEW_SIZE_GB=50
NEW_SIZE_MB=$((NEW_SIZE_GB * 1024))

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Oplog Resize"
log "========================================================"
log "New size: ${NEW_SIZE_GB} GB"
log ""

# Vérifier taille actuelle
CURRENT_SIZE=$(mongo "$MONGO_URI" --quiet --eval "
  db.getSiblingDB('local').oplog.rs.stats().maxSize / 1024 / 1024;
")

log "Current oplog size: ${CURRENT_SIZE} MB"

if (( $(echo "$CURRENT_SIZE >= $NEW_SIZE_MB" | bc -l) )); then
  log "✓ Oplog is already larger than target size"
  exit 0
fi

read -p "⚠️  This will temporarily impact replication. Continue? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
  log "Resize cancelled"
  exit 0
fi

# ============================================================================
# MÉTHODE 1: Replica Set Member (recommandé)
# ============================================================================

log "=== Method 1: Rolling Resize (Replica Set) ==="

# Redimensionner membre par membre
MEMBERS=(
  "mongodb-1:27017"
  "mongodb-2:27017"
  "mongodb-3:27017"
)

for member in "${MEMBERS[@]}"; do
  log "Resizing oplog on: $member"

  # Se connecter au membre
  MEMBER_URI="mongodb://$member/admin"

  # Vérifier le rôle
  STATE=$(mongo "$MEMBER_URI" --quiet --eval "rs.status().myState;")

  if [ "$STATE" == "1" ]; then
    log "  This is PRIMARY - will resize last"
    continue
  fi

  # Redimensionner
  log "  Starting resize..."
  mongo "$MEMBER_URI" --eval "
    // Arrêter en mode maintenance
    db.adminCommand({ replSetMaintenance: 1 });

    // Redimensionner
    db.adminCommand({
      replSetResizeOplog: 1,
      size: $NEW_SIZE_MB
    });

    // Sortir du mode maintenance
    db.adminCommand({ replSetMaintenance: 0 });

    print('✓ Oplog resized on $member');
  "

  # Attendre la synchronisation
  log "  Waiting for member to catch up..."
  sleep 60
done

# Redimensionner le primary en dernier
log "Resizing PRIMARY..."
mongo "$MONGO_URI" --eval "
  // Step down primary
  rs.stepDown(60);
"

sleep 30

# Trouver le nouveau primary et redimensionner l'ancien
# [Code similaire pour l'ancien primary]

log "✓ Oplog resize completed on all members"

# ============================================================================
# MÉTHODE 2: Standalone (si applicable)
# ============================================================================

log "=== Method 2: Standalone Resize ==="

mongo "$MONGO_URI" --eval "
  // Pour standalone ou si urgent sur replica set
  db.adminCommand({
    replSetResizeOplog: 1,
    size: $NEW_SIZE_MB,
    minRetentionHours: 72
  });

  print('✓ Oplog resized to $NEW_SIZE_MB MB');
"

# Vérifier
NEW_ACTUAL_SIZE=$(mongo "$MONGO_URI" --quiet --eval "
  db.getSiblingDB('local').oplog.rs.stats().maxSize / 1024 / 1024;
")

log ""
log "========================================================"
log "✓ OPLOG RESIZE COMPLETED"
log "========================================================"
log "Previous size: ${CURRENT_SIZE} MB"
log "New size: ${NEW_ACTUAL_SIZE} MB"
log "========================================================"
```

## Monitoring de l'Oplog

### Script de Monitoring Complet

```bash
#!/bin/bash
# monitor_oplog.sh

MONGO_URI="mongodb://localhost:27017"
ALERT_THRESHOLD_HOURS=24  # Alerter si fenêtre < 24h
PROMETHEUS_PUSHGATEWAY="http://pushgateway:9091"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Collecter les métriques
collect_metrics() {
  mongo "$MONGO_URI/local" --quiet --eval "
    stats = db.oplog.rs.stats();
    first = db.oplog.rs.find().sort({ts: 1}).limit(1).toArray()[0];
    last = db.oplog.rs.find().sort({ts: -1}).limit(1).toArray()[0];

    windowSeconds = last.ts.getTime() - first.ts.getTime();
    windowHours = windowSeconds / 3600;

    printjson({
      maxSizeMB: stats.maxSize / 1024 / 1024,
      currentSizeMB: stats.size / 1024 / 1024,
      usedPercent: (stats.size / stats.maxSize) * 100,
      windowHours: windowHours,
      firstTimestamp: first.ts.getTime(),
      lastTimestamp: last.ts.getTime(),
      entryCount: db.oplog.rs.count()
    });
  "
}

# Générer rapport
generate_report() {
  local metrics=$1

  local max_size=$(echo "$metrics" | jq -r '.maxSizeMB')
  local current_size=$(echo "$metrics" | jq -r '.currentSizeMB')
  local used_percent=$(echo "$metrics" | jq -r '.usedPercent')
  local window_hours=$(echo "$metrics" | jq -r '.windowHours')

  echo "=== Oplog Status Report ==="
  echo "Timestamp: $(date -Iseconds)"
  echo ""
  echo "Size:"
  echo "  Max: ${max_size} MB"
  echo "  Used: ${current_size} MB (${used_percent}%)"
  echo ""
  echo "Time Window: ${window_hours} hours"

  # Alertes
  if (( $(echo "$window_hours < $ALERT_THRESHOLD_HOURS" | bc -l) )); then
    echo ""
    echo "⚠️  ALERT: Oplog window is below threshold!"
    echo "  Current: ${window_hours}h"
    echo "  Threshold: ${ALERT_THRESHOLD_HOURS}h"
    echo "  Action: Increase oplog size"
    return 1
  else
    echo ""
    echo "✓ Oplog window is healthy"
    return 0
  fi
}

# Exporter vers Prometheus
export_to_prometheus() {
  local metrics=$1

  if [ -n "$PROMETHEUS_PUSHGATEWAY" ]; then
    cat <<EOF | curl --data-binary @- "$PROMETHEUS_PUSHGATEWAY/metrics/job/mongodb_oplog"
# TYPE mongodb_oplog_max_size_mb gauge
mongodb_oplog_max_size_mb $(echo "$metrics" | jq -r '.maxSizeMB')

# TYPE mongodb_oplog_used_size_mb gauge
mongodb_oplog_used_size_mb $(echo "$metrics" | jq -r '.currentSizeMB')

# TYPE mongodb_oplog_used_percent gauge
mongodb_oplog_used_percent $(echo "$metrics" | jq -r '.usedPercent')

# TYPE mongodb_oplog_window_hours gauge
mongodb_oplog_window_hours $(echo "$metrics" | jq -r '.windowHours')

# TYPE mongodb_oplog_entry_count gauge
mongodb_oplog_entry_count $(echo "$metrics" | jq -r '.entryCount')
EOF
  fi
}

# Main
main() {
  log "=== Oplog Monitoring ==="

  metrics=$(collect_metrics)

  generate_report "$metrics"
  result=$?

  export_to_prometheus "$metrics"

  # Slack notification si problème
  if [ $result -ne 0 ]; then
    curl -X POST "$SLACK_WEBHOOK_URL" \
      -H 'Content-Type: application/json' \
      -d "{
        \"text\": \"🚨 MongoDB Oplog Alert\",
        \"attachments\": [{
          \"color\": \"danger\",
          \"text\": \"Oplog window is below threshold: $(echo "$metrics" | jq -r '.windowHours')h < ${ALERT_THRESHOLD_HOURS}h\"
        }]
      }"
  fi

  exit $result
}

main "$@"
```

### Règles d'Alerting Prometheus

```yaml
# prometheus_oplog_alerts.yml

groups:
  - name: mongodb_oplog
    interval: 60s
    rules:
      # Fenêtre oplog trop courte
      - alert: MongoDBOplogWindowLow
        expr: mongodb_oplog_window_hours < 24
        for: 10m
        labels:
          severity: warning
          component: mongodb
        annotations:
          summary: "MongoDB oplog window is low"
          description: "Oplog window is {{ $value }}h (threshold: 24h) on {{ $labels.instance }}"

      # Fenêtre oplog critique
      - alert: MongoDBOplogWindowCritical
        expr: mongodb_oplog_window_hours < 12
        for: 5m
        labels:
          severity: critical
          component: mongodb
        annotations:
          summary: "MongoDB oplog window is critically low"
          description: "Oplog window is only {{ $value }}h on {{ $labels.instance }}. PITR capability at risk!"

      # Oplog presque plein
      - alert: MongoDBOplogNearFull
        expr: mongodb_oplog_used_percent > 90
        for: 15m
        labels:
          severity: warning
          component: mongodb
        annotations:
          summary: "MongoDB oplog is nearly full"
          description: "Oplog is {{ $value }}% full on {{ $labels.instance }}"

      # Taux d'écriture élevé
      - alert: MongoDBOplogHighWriteRate
        expr: rate(mongodb_oplog_entry_count[5m]) > 1000
        for: 15m
        labels:
          severity: info
          component: mongodb
        annotations:
          summary: "High oplog write rate detected"
          description: "Oplog write rate is {{ $value }} ops/sec on {{ $labels.instance }}"
```

## Cas d'Usage Avancés

### Récupération Sélective de Données

```bash
#!/bin/bash
# selective_data_recovery.sh

# Contexte: Récupérer uniquement les documents supprimés d'une collection
# entre deux timestamps

OPLOG_FILE="/backup/oplog/continuous/oplog_20241208_*.bson.gz"
TARGET_COLLECTION="mydb.orders"
START_TIME="2024-12-08T10:00:00Z"
END_TIME="2024-12-08T15:00:00Z"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Selective Data Recovery ==="
log "Collection: $TARGET_COLLECTION"
log "Time range: $START_TIME to $END_TIME"

START_EPOCH=$(date -d "$START_TIME" +%s)
END_EPOCH=$(date -d "$END_TIME" +%s)

# Extraire les opérations de suppression
RECOVERED_DOCS="/tmp/recovered_documents_$$.json"

log "Extracting deleted documents..."

for oplog in $OPLOG_FILE; do
  bsondump "$oplog" | \
    jq -c "
      select(
        .ns == \"$TARGET_COLLECTION\" and
        .op == \"d\" and
        .ts.\"\$timestamp\".t >= $START_EPOCH and
        .ts.\"\$timestamp\".t <= $END_EPOCH
      ) | .o
    " >> "$RECOVERED_DOCS"
done

RECOVERED_COUNT=$(wc -l < "$RECOVERED_DOCS")

log "Found $RECOVERED_COUNT deleted documents"

if [ $RECOVERED_COUNT -eq 0 ]; then
  log "No documents to recover"
  exit 0
fi

# Afficher échantillon
log "Sample of recovered documents:"
head -5 "$RECOVERED_DOCS" | jq .

read -p "Restore these documents to production? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
  log "Recovery cancelled"
  rm -f "$RECOVERED_DOCS"
  exit 0
fi

# Restaurer les documents
log "Restoring documents..."
mongoimport \
  --uri="mongodb://production:27017" \
  --db=$(echo $TARGET_COLLECTION | cut -d. -f1) \
  --collection=$(echo $TARGET_COLLECTION | cut -d. -f2) \
  --file="$RECOVERED_DOCS" \
  --mode=upsert \
  --upsertFields="_id"

log "✓ Recovered $RECOVERED_COUNT documents"

# Cleanup
rm -f "$RECOVERED_DOCS"
```

### Audit Trail depuis l'Oplog

```javascript
// oplog_audit_trail.js

// Générer un audit trail complet d'un document

use local

function generateAuditTrail(namespace, documentId) {
  print("=== Audit Trail ===");
  print("Document: " + documentId);
  print("Collection: " + namespace);
  print("");

  var operations = db.oplog.rs.find({
    ns: namespace,
    $or: [
      { "o._id": documentId },
      { "o2._id": documentId }
    ]
  }).sort({ ts: 1 }).toArray();

  if (operations.length === 0) {
    print("No operations found for this document");
    return;
  }

  print("Total operations: " + operations.length);
  print("");

  operations.forEach(function(op, index) {
    var timestamp = new Date(op.ts.getTime() * 1000);
    var opType = {
      'i': 'INSERT',
      'u': 'UPDATE',
      'd': 'DELETE',
      'c': 'COMMAND'
    }[op.op];

    print("[" + (index + 1) + "] " + opType + " at " + timestamp.toISOString());

    switch(op.op) {
      case 'i':
        print("  Created with:");
        printjson(op.o);
        break;

      case 'u':
        print("  Modified:");
        if (op.o.$set) {
          print("  Set:");
          printjson(op.o.$set);
        }
        if (op.o.$unset) {
          print("  Unset:");
          printjson(op.o.$unset);
        }
        break;

      case 'd':
        print("  Deleted");
        break;
    }

    print("");
  });

  // Générer état actuel
  print("=== Current State ===");
  var currentDoc = db.getSiblingDB(namespace.split('.')[0])[namespace.split('.')[1]].findOne({ _id: documentId });

  if (currentDoc) {
    printjson(currentDoc);
  } else {
    print("Document no longer exists (deleted)");
  }
}

// Exemple d'utilisation
generateAuditTrail("mydb.users", ObjectId("507f1f77bcf86cd799439011"));
```

### Détection d'Anomalies

```javascript
// oplog_anomaly_detection.js

use local

function detectAnomalies(namespace, timeRangeHours) {
  print("=== Anomaly Detection ===");
  print("Collection: " + namespace);
  print("Time range: last " + timeRangeHours + " hours");
  print("");

  var cutoffTime = Timestamp(Math.floor(Date.now() / 1000) - (timeRangeHours * 3600), 0);

  // Analyser les opérations
  var pipeline = [
    {
      $match: {
        ns: namespace,
        ts: { $gt: cutoffTime }
      }
    },
    {
      $group: {
        _id: "$op",
        count: { $sum: 1 }
      }
    },
    {
      $sort: { count: -1 }
    }
  ];

  var opCounts = db.oplog.rs.aggregate(pipeline).toArray();

  print("Operation distribution:");
  opCounts.forEach(function(op) {
    var opName = {
      'i': 'INSERT',
      'u': 'UPDATE',
      'd': 'DELETE',
      'c': 'COMMAND'
    }[op._id];

    print("  " + opName + ": " + op.count);
  });

  // Détecter les suppressions massives
  var deleteCount = opCounts.filter(function(op) { return op._id === 'd'; })[0];
  if (deleteCount && deleteCount.count > 1000) {
    print("");
    print("⚠️  ANOMALY: High delete count detected!");
    print("  Deletes in last " + timeRangeHours + "h: " + deleteCount.count);

    // Lister les timestamps des suppressions
    print("  Deletion timestamps:");
    db.oplog.rs.find({
      ns: namespace,
      op: 'd',
      ts: { $gt: cutoffTime }
    }).sort({ ts: 1 }).limit(10).forEach(function(op) {
      var time = new Date(op.ts.getTime() * 1000);
      print("    " + time.toISOString());
    });
  }

  // Détecter les updates massifs
  var updateCount = opCounts.filter(function(op) { return op._id === 'u'; })[0];
  if (updateCount && updateCount.count > 10000) {
    print("");
    print("⚠️  ANOMALY: High update count detected!");
    print("  Updates in last " + timeRangeHours + "h: " + updateCount.count);
  }
}

// Exécuter pour toutes les collections importantes
["mydb.users", "mydb.orders", "mydb.products"].forEach(function(ns) {
  detectAnomalies(ns, 24);
  print("\n" + "=".repeat(60) + "\n");
});
```

## Bonnes Pratiques

### Checklist Oplog

```markdown
### Configuration Initiale

- [ ] Dimensionner l'oplog adequatement (72h min prod)
- [ ] Activer --oplog dans les backups mongodump
- [ ] Configurer export continu de l'oplog
- [ ] Mettre en place monitoring fenêtre oplog
- [ ] Configurer alertes (< 24h)
- [ ] Documenter procédures de récupération

### Maintenance Régulière

- [ ] Vérifier fenêtre oplog hebdomadairement
- [ ] Monitorer taux de croissance
- [ ] Ajuster taille si nécessaire
- [ ] Valider exports continus fonctionnent
- [ ] Tester rejeu oplog mensuellement
- [ ] Archiver anciens oplogs (compression)

### En Cas d'Incident

- [ ] Identifier timestamp exact du problème
- [ ] Vérifier disponibilité oplog pour ce timestamp
- [ ] Extraire oplog pertinent
- [ ] Valider intégrité de l'oplog
- [ ] Rejouer dans environnement test
- [ ] Valider résultat avant production
- [ ] Documenter incident et procédure

### Optimisation

- [ ] Analyser patterns d'écriture
- [ ] Identifier collections à fort taux d'écriture
- [ ] Optimiser schéma pour réduire updates
- [ ] Utiliser $set plutôt que remplacer documents
- [ ] Batch operations quand possible
- [ ] Monitorer impact sur oplog
```

### Recommandations par Environnement

```yaml
Development:
  oplog_size: 2-5 GB
  window_target: 24 heures
  export_continu: Non
  monitoring: Basique

Staging:
  oplog_size: 10-20 GB
  window_target: 48 heures
  export_continu: Optionnel
  monitoring: Standard

Production Standard:
  oplog_size: 50-100 GB
  window_target: 72 heures
  export_continu: Recommandé
  monitoring: Complet + alertes
  backup_frequency: Toutes les 6h

Production Critical:
  oplog_size: 200+ GB
  window_target: 168 heures (7 jours)
  export_continu: Obligatoire
  monitoring: Temps réel + alertes multi-niveaux
  backup_frequency: Toutes les 4h
  redundancy: Export vers multiple destinations

Mission-Critical:
  oplog_size: 500+ GB
  window_target: 720 heures (30 jours)
  export_continu: Obligatoire avec réplication
  monitoring: Temps réel + prédictif
  backup_frequency: Horaire
  redundancy: Multi-région
  solution: Considérer MongoDB Atlas
```

## Conclusion

L'oplog est le mécanisme central qui permet à MongoDB de garantir la cohérence, la réplication et la récupération des données. Une compréhension approfondie de son fonctionnement et une gestion proactive sont essentielles pour toute stratégie de continuité d'activité robuste.

**Points clés à retenir** :

1. **Dimensionnement critique** - L'oplog doit couvrir au minimum 72h pour production
2. **Export continu** - Pour RPO minimal, exporter l'oplog en temps réel
3. **Monitoring essentiel** - Surveiller la fenêtre oplog et alerter proactivement
4. **Tests réguliers** - Valider le rejeu oplog mensuellement
5. **Idempotence** - Comprendre que l'oplog est safe à rejouer

**Stratégies selon criticité** :
- **Standard** : Backups avec --oplog + monitoring basique
- **Critique** : Export continu + monitoring avancé + alertes
- **Mission-critical** : Streaming temps réel + redondance + MongoDB Atlas

L'oplog n'est pas qu'un journal de réplication - c'est votre filet de sécurité pour la récupération granulaire, l'audit trail et la garantie de continuité d'activité.

---


⏭️ [Restauration complète vs partielle](/12-sauvegarde-restauration/09-restauration-complete-partielle.md)
