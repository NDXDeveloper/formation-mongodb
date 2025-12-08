🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.9 Restauration Complète vs Partielle

## Introduction

Les opérations de restauration MongoDB se déclinent en deux grandes catégories : la restauration complète qui reconstitue l'intégralité du cluster, et la restauration partielle qui cible spécifiquement certaines bases, collections ou documents. Le choix entre ces approches dépend de la nature de l'incident, de l'impact business, du RTO/RPO requis, et des contraintes opérationnelles.

Cette section explore en profondeur les stratégies, procédures et bonnes pratiques pour chaque type de restauration, avec un focus sur la minimisation de l'impact et la garantie de l'intégrité des données.

## Comparaison des Approches

### Vue d'Ensemble

```
┌─────────────────────────────────────────────────────────────┐
│            Restauration Complète vs Partielle               │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         RESTAURATION COMPLÈTE                        │   │
│  │                                                      │   │
│  │  Scope: Cluster entier ou instance complète          │   │
│  │                                                      │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │   │
│  │  │  admin   │  │   DB 1   │  │   DB 2   │            │   │
│  │  └──────────┘  └──────────┘  └──────────┘            │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │   │
│  │  │  config  │  │   DB 3   │  │   ...    │            │   │
│  │  └──────────┘  └──────────┘  └──────────┘            │   │
│  │                                                      │   │
│  │  Caractéristiques:                                   │   │
│  │  • Downtime total requis                             │   │
│  │  • Tout ou rien                                      │   │
│  │  • Cohérence garantie                                │   │
│  │  • Procédure standardisée                            │   │
│  │  • RTO: 30min - 4h selon taille                      │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         RESTAURATION PARTIELLE                       │   │
│  │                                                      │   │
│  │  Scope: Database, collection ou documents spécifiques│   │
│  │                                                      │   │
│  │  ┌──────────────────────────────────────────┐        │   │
│  │  │        Database Production               │        │   │
│  │  │  ┌──────────┐  ┌──────────┐  ┌────────┐  │        │   │
│  │  │  │  users   │  │  orders  │  │ config │  │        │   │
│  │  │  └──────────┘  └────┬─────┘  └────────┘  │        │   │
│  │  │                     │                    │        │   │
│  │  │                     │ ← Restauration     │        │   │
│  │  │                     │   ciblée           │        │   │
│  │  │  ┌──────────┐  ┌────▼─────┐  ┌────────┐  │        │   │
│  │  │  │  logs    │  │  orders  │  │ audit  │  │        │   │
│  │  │  └──────────┘  └──────────┘  └────────┘  │        │   │
│  │  └──────────────────────────────────────────┘        │   │
│  │                                                      │   │
│  │  Caractéristiques:                                   │   │
│  │  • Service peut rester en ligne                      │   │
│  │  • Granularité fine                                  │   │
│  │  • Risques de cohérence                              │   │
│  │  • Procédure complexe                                │   │
│  │  • RTO: minutes - 1h selon volume                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Matrice de Décision

```yaml
Restauration Complète - Utiliser quand:
  incidents:
    - Corruption totale de la base
    - Perte complète du cluster (disaster)
    - Compromission sécurité (ransomware)
    - Migration/upgrade raté
    - Besoin de cohérence absolue

  avantages:
    - Cohérence garantie
    - Procédure standardisée
    - Moins de risques d'erreurs
    - Validation simple
    - État connu et testé

  inconvénients:
    - Downtime total
    - Temps de restauration long
    - Impact business maximal
    - Perte de données récentes (depuis backup)

  rto_type: "Élevé (1-4 heures)"
  complexity: "Moyenne"

Restauration Partielle - Utiliser quand:
  incidents:
    - Erreur sur données spécifiques
    - Suppression accidentelle
    - Corruption limitée
    - Test/validation nécessaire
    - Données forensiques

  avantages:
    - Service reste disponible
    - Impact minimal
    - Rapide à exécuter
    - Ciblé et précis

  inconvénients:
    - Risques de cohérence
    - Procédure plus complexe
    - Validation difficile
    - Références croisées potentiellement cassées

  rto_type: "Faible (minutes - 1h)"
  complexity: "Élevée"
```

## Restauration Complète

### Procédure Standard - Instance Standalone

```bash
#!/bin/bash
# full_restore_standalone.sh

set -euo pipefail

BACKUP_FILE="/backup/mongodb/full_backup_20241208_020000.tar.gz"
MONGO_HOST="localhost"
MONGO_PORT=27017
DATA_DIR="/var/lib/mongodb"
SERVICE_NAME="mongod"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  MongoDB Full Restoration - Standalone"
log "========================================================"
log "Backup: $BACKUP_FILE"
log "Target: $MONGO_HOST:$MONGO_PORT"
log ""

# Vérifier le backup
if [ ! -f "$BACKUP_FILE" ]; then
  log "ERROR: Backup file not found"
  exit 1
fi

# ============================================================================
# PHASE 1: ARRÊT DU SERVICE
# ============================================================================

log "=== Phase 1: Stopping MongoDB ==="

systemctl stop "$SERVICE_NAME"

# Vérifier que le processus est bien arrêté
for i in {1..30}; do
  if ! pgrep -x mongod >/dev/null; then
    log "✓ MongoDB stopped"
    break
  fi

  if [ $i -eq 30 ]; then
    log "ERROR: MongoDB did not stop in time"
    exit 1
  fi

  sleep 1
done

# ============================================================================
# PHASE 2: SAUVEGARDE DES DONNÉES ACTUELLES
# ============================================================================

log "=== Phase 2: Backing up Current Data ==="

CURRENT_BACKUP_DIR="${DATA_DIR}.backup_$(date +%s)"

if [ -d "$DATA_DIR" ]; then
  log "Moving current data to: $CURRENT_BACKUP_DIR"
  mv "$DATA_DIR" "$CURRENT_BACKUP_DIR"
else
  log "No existing data directory"
fi

# ============================================================================
# PHASE 3: EXTRACTION DU BACKUP
# ============================================================================

log "=== Phase 3: Extracting Backup ==="

# Créer le répertoire de données
mkdir -p "$DATA_DIR"

# Extraire le backup
TEMP_EXTRACT="/tmp/mongodb_restore_$$"
mkdir -p "$TEMP_EXTRACT"

log "Extracting archive..."
tar -xzf "$BACKUP_FILE" -C "$TEMP_EXTRACT"

# Trouver le répertoire du backup
BACKUP_DIR=$(find "$TEMP_EXTRACT" -type d -name "backup_*" -o -name "dump" | head -1)

if [ -z "$BACKUP_DIR" ]; then
  log "ERROR: Cannot find backup directory in archive"
  exit 1
fi

log "Backup directory: $BACKUP_DIR"

# ============================================================================
# PHASE 4: RESTAURATION AVEC MONGORESTORE
# ============================================================================

log "=== Phase 4: Restoring Data ==="

# Démarrer MongoDB temporairement pour la restauration
log "Starting MongoDB temporarily..."
mongod --dbpath "$DATA_DIR" --port $MONGO_PORT --fork --logpath /tmp/mongod_restore.log

sleep 10

# Vérifier que MongoDB est démarré
if ! mongo --host $MONGO_HOST --port $MONGO_PORT --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
  log "ERROR: MongoDB failed to start"
  exit 1
fi

log "Restoring databases..."
START_TIME=$(date +%s)

mongorestore \
  --host="$MONGO_HOST" \
  --port=$MONGO_PORT \
  --drop \
  --oplogReplay \
  --gzip \
  --dir="$BACKUP_DIR" \
  --numParallelCollections=4

RESTORE_EXIT=$?
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

if [ $RESTORE_EXIT -ne 0 ]; then
  log "ERROR: Restore failed"

  # Restaurer les données originales
  systemctl stop mongod
  rm -rf "$DATA_DIR"
  mv "$CURRENT_BACKUP_DIR" "$DATA_DIR"
  systemctl start "$SERVICE_NAME"

  exit 1
fi

log "✓ Restore completed in ${DURATION}s"

# Arrêter l'instance temporaire
log "Stopping temporary MongoDB instance..."
mongo --host $MONGO_HOST --port $MONGO_PORT admin --eval "db.shutdownServer()" || true
sleep 5

# ============================================================================
# PHASE 5: AJUSTEMENT DES PERMISSIONS
# ============================================================================

log "=== Phase 5: Fixing Permissions ==="

chown -R mongodb:mongodb "$DATA_DIR"
chmod 750 "$DATA_DIR"

# ============================================================================
# PHASE 6: DÉMARRAGE DU SERVICE
# ============================================================================

log "=== Phase 6: Starting MongoDB Service ==="

systemctl start "$SERVICE_NAME"

# Attendre que MongoDB soit prêt
log "Waiting for MongoDB to be ready..."
for i in {1..60}; do
  if mongo --host $MONGO_HOST --port $MONGO_PORT --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log "✓ MongoDB is ready"
    break
  fi

  if [ $i -eq 60 ]; then
    log "ERROR: MongoDB did not start properly"
    exit 1
  fi

  sleep 2
done

# ============================================================================
# PHASE 7: VALIDATION
# ============================================================================

log "=== Phase 7: Validation ==="

# Vérifier les databases
mongo --host $MONGO_HOST --port $MONGO_PORT --quiet --eval "
  print('=== Databases ===');
  dbs = db.adminCommand({ listDatabases: 1 }).databases;
  dbs.forEach(function(database) {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      count = db.getSiblingDB(database.name).getCollectionNames().length;
      size = (database.sizeOnDisk / 1024 / 1024 / 1024).toFixed(2);
      print('✓ ' + database.name + ' (' + count + ' collections, ' + size + ' GB)');
    }
  });

  print('\n=== Sample Data Check ===');
  // Vérifier quelques collections importantes
  use mydb
  if (db.users) {
    count = db.users.countDocuments();
    print('Users: ' + count + ' documents');
  }
"

# Cleanup
rm -rf "$TEMP_EXTRACT"

log ""
log "========================================================"
log "✓ FULL RESTORATION COMPLETED SUCCESSFULLY"
log "========================================================"
log "Duration: ${DURATION}s"
log "Old data preserved in: $CURRENT_BACKUP_DIR"
log ""
log "Next Steps:"
log "  1. Verify application connectivity"
log "  2. Run application-level tests"
log "  3. Check data integrity"
log "  4. Monitor performance"
log "  5. Remove old backup when confident: $CURRENT_BACKUP_DIR"
log "========================================================"
```

### Restauration Complète - Replica Set

```bash
#!/bin/bash
# full_restore_replica_set.sh

set -euo pipefail

RS_NAME="rs0"
RS_MEMBERS=(
  "mongodb-1:27017"
  "mongodb-2:27017"
  "mongodb-3:27017"
)
BACKUP_DIR="/backup/mongodb/rs_backup_20241208"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  MongoDB Full Restoration - Replica Set"
log "========================================================"
log "Replica Set: $RS_NAME"
log "Members: ${RS_MEMBERS[@]}"
log ""

# ============================================================================
# PHASE 1: ARRÊTER LE REPLICA SET
# ============================================================================

log "=== Phase 1: Stopping Replica Set ==="

for member in "${RS_MEMBERS[@]}"; do
  host=$(echo $member | cut -d: -f1)
  port=$(echo $member | cut -d: -f2)

  log "Stopping member: $member"

  mongo "mongodb://$member/admin" --eval "
    db.shutdownServer({ force: true });
  " || true

  # Arrêter via systemd
  ssh "$host" "systemctl stop mongod"
done

sleep 10

# ============================================================================
# PHASE 2: RESTAURER SUR LE PREMIER MEMBRE (STANDALONE)
# ============================================================================

log "=== Phase 2: Restoring to Primary Member ==="

PRIMARY="${RS_MEMBERS[0]}"
PRIMARY_HOST=$(echo $PRIMARY | cut -d: -f1)
PRIMARY_PORT=$(echo $PRIMARY | cut -d: -f2)

log "Restoring to: $PRIMARY"

# Nettoyer le datadir
ssh "$PRIMARY_HOST" "rm -rf /var/lib/mongodb/*"

# Démarrer en mode standalone pour restauration
ssh "$PRIMARY_HOST" "
  mongod \
    --dbpath /var/lib/mongodb \
    --port $PRIMARY_PORT \
    --bind_ip 0.0.0.0 \
    --fork \
    --logpath /tmp/mongod_restore.log
"

sleep 10

# Restaurer le backup
log "Starting restoration..."
mongorestore \
  --host="$PRIMARY" \
  --drop \
  --oplogReplay \
  --gzip \
  --numParallelCollections=8 \
  --dir="$BACKUP_DIR"

if [ $? -ne 0 ]; then
  log "ERROR: Restore failed"
  exit 1
fi

log "✓ Data restored on primary member"

# Arrêter l'instance standalone
mongo "mongodb://$PRIMARY/admin" --eval "db.shutdownServer()" || true
sleep 5

# ============================================================================
# PHASE 3: RECONFIGURER LE REPLICA SET
# ============================================================================

log "=== Phase 3: Reconfiguring Replica Set ==="

# Démarrer le primary en mode replica set
ssh "$PRIMARY_HOST" "systemctl start mongod"
sleep 10

# Initialiser le replica set avec un seul membre
log "Initializing replica set..."
mongo "mongodb://$PRIMARY/admin" <<EOF
  rs.initiate({
    _id: "$RS_NAME",
    members: [
      { _id: 0, host: "$PRIMARY", priority: 2 }
    ]
  });
EOF

sleep 15

# Vérifier que le replica set est initialisé
STATUS=$(mongo "mongodb://$PRIMARY/admin" --quiet --eval "rs.status().ok;")
if [ "$STATUS" != "1" ]; then
  log "ERROR: Replica set initialization failed"
  exit 1
fi

log "✓ Replica set initialized"

# ============================================================================
# PHASE 4: AJOUTER LES MEMBRES SECONDAIRES
# ============================================================================

log "=== Phase 4: Adding Secondary Members ==="

for i in $(seq 1 $((${#RS_MEMBERS[@]} - 1))); do
  member="${RS_MEMBERS[$i]}"
  host=$(echo $member | cut -d: -f1)

  log "Adding member: $member"

  # Nettoyer le datadir
  ssh "$host" "rm -rf /var/lib/mongodb/*"

  # Démarrer le membre
  ssh "$host" "systemctl start mongod"
  sleep 5

  # Ajouter au replica set
  mongo "mongodb://$PRIMARY/admin" --eval "
    rs.add('$member');
  "

  log "✓ Member added, waiting for initial sync..."
  sleep 10
done

# ============================================================================
# PHASE 5: ATTENDRE LA SYNCHRONISATION
# ============================================================================

log "=== Phase 5: Waiting for Synchronization ==="

TIMEOUT=3600  # 1 heure
ELAPSED=0

while [ $ELAPSED -lt $TIMEOUT ]; do
  # Compter les membres healthy
  HEALTHY=$(mongo "mongodb://$PRIMARY/admin" --quiet --eval "
    rs.status().members.filter(m => m.health === 1 && (m.state === 1 || m.state === 2)).length;
  ")

  if [ "$HEALTHY" == "${#RS_MEMBERS[@]}" ]; then
    log "✓ All members are healthy and synchronized"
    break
  fi

  # Afficher la progression
  mongo "mongodb://$PRIMARY/admin" --quiet --eval "
    rs.status().members.forEach(function(m) {
      if (m.stateStr === 'SECONDARY') {
        print('  ' + m.name + ': ' + m.stateStr +
              ' (lag: ' + Math.round((Date.now() - m.optimeDate.getTime()) / 1000) + 's)');
      } else {
        print('  ' + m.name + ': ' + m.stateStr);
      }
    });
  "

  sleep 30
  ELAPSED=$((ELAPSED + 30))
done

if [ $ELAPSED -ge $TIMEOUT ]; then
  log "WARNING: Synchronization timeout - check members manually"
fi

# ============================================================================
# PHASE 6: VALIDATION
# ============================================================================

log "=== Phase 6: Validation ==="

mongo "mongodb://${RS_MEMBERS[0]}/${RS_NAME}" --quiet --eval "
  print('=== Replica Set Status ===');
  status = rs.status();

  print('Set: ' + status.set);
  print('Members:');
  status.members.forEach(function(m) {
    print('  ' + m.name + ': ' + m.stateStr + ' (health: ' + m.health + ')');
  });

  print('\n=== Data Verification ===');
  dbs = db.adminCommand({ listDatabases: 1 }).databases;
  total = 0;
  dbs.forEach(function(database) {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      count = db.getSiblingDB(database.name).getCollectionNames().length;
      total += count;
      print('✓ ' + database.name + ': ' + count + ' collections');
    }
  });
  print('Total collections: ' + total);
"

log ""
log "========================================================"
log "✓ REPLICA SET FULL RESTORATION COMPLETED"
log "========================================================"
log ""
log "Verify:"
log "  mongo \"mongodb://${RS_MEMBERS[@]}/?replicaSet=$RS_NAME\""
log "  rs.status()"
log "========================================================"
```

## Restauration Partielle

### Restauration au Niveau Base de Données

```bash
#!/bin/bash
# partial_restore_database.sh

set -euo pipefail

BACKUP_DIR="/backup/mongodb/full_backup_20241208"
TARGET_DB="orders_db"
MONGO_URI="mongodb://localhost:27017"
TEMP_RESTORE_DB="${TARGET_DB}_restore_temp"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Partial Restoration - Database Level"
log "========================================================"
log "Target Database: $TARGET_DB"
log "Source: $BACKUP_DIR"
log ""

# Vérifier que le backup contient la database
if [ ! -d "$BACKUP_DIR/$TARGET_DB" ]; then
  log "ERROR: Database not found in backup"
  exit 1
fi

# ============================================================================
# STRATÉGIE: RESTAURER DANS UNE BASE TEMPORAIRE
# ============================================================================

log "=== Restoring to Temporary Database ==="

# Restaurer dans une base temporaire pour validation
mongorestore \
  --uri="$MONGO_URI" \
  --db="$TEMP_RESTORE_DB" \
  --gzip \
  --dir="$BACKUP_DIR/$TARGET_DB"

if [ $? -ne 0 ]; then
  log "ERROR: Restore to temporary database failed"
  exit 1
fi

log "✓ Data restored to temporary database: $TEMP_RESTORE_DB"

# ============================================================================
# VALIDATION DES DONNÉES RESTAURÉES
# ============================================================================

log "=== Validating Restored Data ==="

mongo "$MONGO_URI/$TEMP_RESTORE_DB" --quiet --eval "
  print('=== Data Validation ===');

  // Compter les collections
  colls = db.getCollectionNames();
  print('Collections: ' + colls.length);

  // Vérifier chaque collection
  colls.forEach(function(coll) {
    count = db[coll].countDocuments();
    size = db[coll].stats().size;
    print('  ' + coll + ': ' + count + ' documents (' +
          (size / 1024 / 1024).toFixed(2) + ' MB)');

    // Valider l'intégrité
    validation = db[coll].validate({ full: false });
    if (!validation.valid) {
      print('  ⚠️  WARNING: Validation failed for ' + coll);
    }
  });

  print('\n✓ Validation completed');
"

# ============================================================================
# CONFIRMATION UTILISATEUR
# ============================================================================

log ""
log "Restored data is in temporary database: $TEMP_RESTORE_DB"
log "Please verify the data before proceeding."
log ""

read -p "Replace production database $TARGET_DB? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
  log "Restoration cancelled - temporary database preserved"
  log "You can:"
  log "  1. Inspect: mongo $MONGO_URI/$TEMP_RESTORE_DB"
  log "  2. Remove: mongo $MONGO_URI --eval 'db.getSiblingDB(\"$TEMP_RESTORE_DB\").dropDatabase()'"
  exit 0
fi

# ============================================================================
# REMPLACEMENT DE LA BASE DE PRODUCTION
# ============================================================================

log "=== Replacing Production Database ==="

# Backup de la base actuelle (si existe)
BACKUP_TIMESTAMP=$(date +%s)
SAFETY_BACKUP="${TARGET_DB}_backup_${BACKUP_TIMESTAMP}"

mongo "$MONGO_URI/admin" --quiet --eval "
  // Copier la base actuelle vers backup de sécurité
  if (db.getSiblingDB('$TARGET_DB').getCollectionNames().length > 0) {
    print('Creating safety backup: $SAFETY_BACKUP');

    db.getSiblingDB('$TARGET_DB').getCollectionNames().forEach(function(coll) {
      db.getSiblingDB('$TARGET_DB')[coll].aggregate([
        { \$out: { db: '$SAFETY_BACKUP', coll: coll } }
      ]);
    });

    print('✓ Safety backup created');
  }
"

# Supprimer la base de production
log "Dropping production database..."
mongo "$MONGO_URI/$TARGET_DB" --eval "db.dropDatabase()"

# Renommer la base temporaire
log "Renaming temporary database to production..."
mongo "$MONGO_URI/admin" --quiet --eval "
  db.getSiblingDB('$TEMP_RESTORE_DB').getCollectionNames().forEach(function(coll) {
    db.adminCommand({
      renameCollection: '$TEMP_RESTORE_DB.' + coll,
      to: '$TARGET_DB.' + coll,
      dropTarget: false
    });
  });

  print('✓ Database renamed');
"

# Supprimer la base temporaire (devrait être vide)
mongo "$MONGO_URI/$TEMP_RESTORE_DB" --eval "db.dropDatabase()"

# ============================================================================
# VALIDATION POST-RESTAURATION
# ============================================================================

log "=== Post-Restoration Validation ==="

mongo "$MONGO_URI/$TARGET_DB" --quiet --eval "
  print('=== Production Database Status ===');

  colls = db.getCollectionNames();
  print('Collections: ' + colls.length);

  totalDocs = 0;
  colls.forEach(function(coll) {
    count = db[coll].countDocuments();
    totalDocs += count;
    print('  ' + coll + ': ' + count + ' documents');
  });

  print('Total documents: ' + totalDocs);
  print('\n✓ Database is operational');
"

log ""
log "========================================================"
log "✓ DATABASE PARTIAL RESTORATION COMPLETED"
log "========================================================"
log "Database restored: $TARGET_DB"
log "Safety backup: $SAFETY_BACKUP"
log ""
log "Next Steps:"
log "  1. Test application functionality"
log "  2. Verify data integrity"
log "  3. Monitor for errors"
log "  4. Remove safety backup when confident:"
log "     mongo $MONGO_URI/$SAFETY_BACKUP --eval 'db.dropDatabase()'"
log "========================================================"
```

### Restauration au Niveau Collection

```bash
#!/bin/bash
# partial_restore_collection.sh

set -euo pipefail

BACKUP_DIR="/backup/mongodb/full_backup_20241208"
TARGET_DB="mydb"
TARGET_COLLECTION="orders"
MONGO_URI="mongodb://localhost:27017"
RESTORE_MODE="replace"  # replace|merge|test

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Partial Restoration - Collection Level"
log "========================================================"
log "Target: $TARGET_DB.$TARGET_COLLECTION"
log "Mode: $RESTORE_MODE"
log ""

# Vérifier que la collection existe dans le backup
COLLECTION_FILE="$BACKUP_DIR/$TARGET_DB/$TARGET_COLLECTION.bson.gz"
if [ ! -f "$COLLECTION_FILE" ]; then
  log "ERROR: Collection not found in backup: $COLLECTION_FILE"
  exit 1
fi

# ============================================================================
# STATISTIQUES PRÉ-RESTAURATION
# ============================================================================

log "=== Pre-Restoration Statistics ==="

BEFORE_STATS=$(mongo "$MONGO_URI/$TARGET_DB" --quiet --eval "
  stats = db.$TARGET_COLLECTION.stats();
  printjson({
    exists: db.$TARGET_COLLECTION.countDocuments() > 0,
    count: db.$TARGET_COLLECTION.countDocuments(),
    size: stats.size,
    avgObjSize: stats.avgObjSize
  });
" | tail -1)

log "Current state:"
echo "$BEFORE_STATS" | jq .

# ============================================================================
# RESTAURATION SELON LE MODE
# ============================================================================

case "$RESTORE_MODE" in

  # MODE 1: REMPLACEMENT COMPLET
  replace)
    log "=== Mode: Replace ==="
    log "This will DROP and recreate the collection"

    read -p "Continue? (yes/no): " confirm
    if [ "$confirm" != "yes" ]; then
      log "Restoration cancelled"
      exit 0
    fi

    # Backup de sécurité
    SAFETY_BACKUP="$TARGET_COLLECTION"_backup_$(date +%s)
    log "Creating safety backup: $SAFETY_BACKUP"

    mongo "$MONGO_URI/$TARGET_DB" --quiet --eval "
      db.$TARGET_COLLECTION.aggregate([
        { \$out: '$SAFETY_BACKUP' }
      ]);
      print('✓ Safety backup created');
    "

    # Restaurer avec drop
    mongorestore \
      --uri="$MONGO_URI" \
      --db="$TARGET_DB" \
      --collection="$TARGET_COLLECTION" \
      --drop \
      --gzip \
      "$COLLECTION_FILE"
    ;;

  # MODE 2: FUSION (UPSERT)
  merge)
    log "=== Mode: Merge ==="
    log "This will merge data using upsert"

    # Exporter vers JSON pour upsert
    TEMP_JSON="/tmp/restore_$$.json"

    log "Extracting backup data..."
    bsondump "$COLLECTION_FILE" > "$TEMP_JSON"

    log "Merging data..."
    mongoimport \
      --uri="$MONGO_URI" \
      --db="$TARGET_DB" \
      --collection="$TARGET_COLLECTION" \
      --file="$TEMP_JSON" \
      --mode=upsert \
      --upsertFields="_id"

    rm -f "$TEMP_JSON"
    ;;

  # MODE 3: TEST (COLLECTION TEMPORAIRE)
  test)
    log "=== Mode: Test ==="
    log "Restoring to temporary collection for validation"

    TEMP_COLLECTION="${TARGET_COLLECTION}_restore_test"

    mongorestore \
      --uri="$MONGO_URI" \
      --db="$TARGET_DB" \
      --collection="$TEMP_COLLECTION" \
      --gzip \
      "$COLLECTION_FILE"

    log "✓ Data restored to: $TARGET_DB.$TEMP_COLLECTION"
    log ""
    log "Compare:"
    log "  mongo $MONGO_URI/$TARGET_DB"
    log "  > db.$TARGET_COLLECTION.countDocuments()"
    log "  > db.$TEMP_COLLECTION.countDocuments()"
    log ""
    log "When ready:"
    log "  db.$TEMP_COLLECTION.renameCollection('$TARGET_COLLECTION', true)"

    exit 0
    ;;

  *)
    log "ERROR: Unknown restore mode: $RESTORE_MODE"
    exit 1
    ;;
esac

# ============================================================================
# VALIDATION POST-RESTAURATION
# ============================================================================

log "=== Post-Restoration Validation ==="

AFTER_STATS=$(mongo "$MONGO_URI/$TARGET_DB" --quiet --eval "
  stats = db.$TARGET_COLLECTION.stats();
  printjson({
    count: db.$TARGET_COLLECTION.countDocuments(),
    size: stats.size,
    avgObjSize: stats.avgObjSize
  });
" | tail -1)

log "After restoration:"
echo "$AFTER_STATS" | jq .

# Comparer
BEFORE_COUNT=$(echo "$BEFORE_STATS" | jq -r '.count // 0')
AFTER_COUNT=$(echo "$AFTER_STATS" | jq -r '.count')

log ""
log "Comparison:"
log "  Before: $BEFORE_COUNT documents"
log "  After: $AFTER_COUNT documents"
log "  Difference: $((AFTER_COUNT - BEFORE_COUNT))"

# Validation de l'intégrité
log ""
log "Validating collection integrity..."
mongo "$MONGO_URI/$TARGET_DB" --quiet --eval "
  validation = db.$TARGET_COLLECTION.validate({ full: false });
  if (validation.valid) {
    print('✓ Collection is valid');
  } else {
    print('✗ Collection validation failed');
    printjson(validation);
    quit(1);
  }
"

log ""
log "========================================================"
log "✓ COLLECTION PARTIAL RESTORATION COMPLETED"
log "========================================================"
log "Collection: $TARGET_DB.$TARGET_COLLECTION"
log "Documents: $AFTER_COUNT"
log "========================================================"
```

### Restauration de Documents Spécifiques

```bash
#!/bin/bash
# partial_restore_documents.sh

set -euo pipefail

BACKUP_DIR="/backup/mongodb/full_backup_20241208"
TARGET_DB="mydb"
TARGET_COLLECTION="users"
MONGO_URI="mongodb://localhost:27017"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Partial Restoration - Specific Documents"
log "========================================================"

# ============================================================================
# STRATÉGIE 1: PAR ID
# ============================================================================

restore_by_ids() {
  local ids_file=$1

  log "=== Restoring Documents by IDs ==="
  log "IDs file: $ids_file"

  if [ ! -f "$ids_file" ]; then
    log "ERROR: IDs file not found"
    return 1
  fi

  # Exporter le backup vers JSON
  BACKUP_JSON="/tmp/backup_export_$$.json"
  bsondump "$BACKUP_DIR/$TARGET_DB/$TARGET_COLLECTION.bson.gz" > "$BACKUP_JSON"

  # Filtrer les documents par ID
  FILTERED_DOCS="/tmp/filtered_docs_$$.json"

  while read -r doc_id; do
    jq -c "select(._id.\"\$oid\" == \"$doc_id\")" "$BACKUP_JSON" >> "$FILTERED_DOCS"
  done < "$ids_file"

  RESTORE_COUNT=$(wc -l < "$FILTERED_DOCS")
  log "Documents to restore: $RESTORE_COUNT"

  if [ $RESTORE_COUNT -eq 0 ]; then
    log "No documents found matching IDs"
    rm -f "$BACKUP_JSON" "$FILTERED_DOCS"
    return 0
  fi

  # Importer avec upsert
  log "Restoring documents..."
  mongoimport \
    --uri="$MONGO_URI" \
    --db="$TARGET_DB" \
    --collection="$TARGET_COLLECTION" \
    --file="$FILTERED_DOCS" \
    --mode=upsert \
    --upsertFields="_id"

  log "✓ Restored $RESTORE_COUNT documents"

  # Cleanup
  rm -f "$BACKUP_JSON" "$FILTERED_DOCS"
}

# ============================================================================
# STRATÉGIE 2: PAR CRITÈRES
# ============================================================================

restore_by_criteria() {
  local query=$1

  log "=== Restoring Documents by Criteria ==="
  log "Query: $query"

  # Exporter le backup
  BACKUP_JSON="/tmp/backup_export_$$.json"
  bsondump "$BACKUP_DIR/$TARGET_DB/$TARGET_COLLECTION.bson.gz" > "$BACKUP_JSON"

  # Filtrer avec jq
  FILTERED_DOCS="/tmp/filtered_docs_$$.json"
  jq -c "$query" "$BACKUP_JSON" > "$FILTERED_DOCS"

  RESTORE_COUNT=$(wc -l < "$FILTERED_DOCS")
  log "Documents matching criteria: $RESTORE_COUNT"

  if [ $RESTORE_COUNT -eq 0 ]; then
    log "No documents found matching criteria"
    rm -f "$BACKUP_JSON" "$FILTERED_DOCS"
    return 0
  fi

  # Afficher échantillon
  log "Sample documents:"
  head -3 "$FILTERED_DOCS" | jq .

  read -p "Restore these $RESTORE_COUNT documents? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    log "Restoration cancelled"
    rm -f "$BACKUP_JSON" "$FILTERED_DOCS"
    return 0
  fi

  # Importer
  mongoimport \
    --uri="$MONGO_URI" \
    --db="$TARGET_DB" \
    --collection="$TARGET_COLLECTION" \
    --file="$FILTERED_DOCS" \
    --mode=upsert \
    --upsertFields="_id"

  log "✓ Restored $RESTORE_COUNT documents"

  # Cleanup
  rm -f "$BACKUP_JSON" "$FILTERED_DOCS"
}

# ============================================================================
# STRATÉGIE 3: RESTAURATION AVEC TRANSFORMATION
# ============================================================================

restore_with_transformation() {
  log "=== Restoring Documents with Transformation ==="

  # Exporter le backup
  BACKUP_JSON="/tmp/backup_export_$$.json"
  bsondump "$BACKUP_DIR/$TARGET_DB/$TARGET_COLLECTION.bson.gz" > "$BACKUP_JSON"

  # Transformer les documents (exemple: corriger un champ)
  TRANSFORMED_DOCS="/tmp/transformed_docs_$$.json"

  jq -c '
    # Exemple: fixer les emails en minuscules
    .email = (.email | ascii_downcase) |
    # Ajouter un champ de tracking
    .restored_at = (now | todate) |
    .restored_from_backup = true
  ' "$BACKUP_JSON" > "$TRANSFORMED_DOCS"

  log "Documents transformed: $(wc -l < $TRANSFORMED_DOCS)"

  # Importer
  mongoimport \
    --uri="$MONGO_URI" \
    --db="$TARGET_DB" \
    --collection="$TARGET_COLLECTION" \
    --file="$TRANSFORMED_DOCS" \
    --mode=upsert \
    --upsertFields="_id"

  log "✓ Documents restored with transformations"

  # Cleanup
  rm -f "$BACKUP_JSON" "$TRANSFORMED_DOCS"
}

# ============================================================================
# MENU INTERACTIF
# ============================================================================

log ""
log "Select restoration strategy:"
log "  1) Restore by document IDs"
log "  2) Restore by criteria (jq query)"
log "  3) Restore with transformation"
log ""

read -p "Choice (1-3): " choice

case "$choice" in
  1)
    read -p "Path to IDs file (one ID per line): " ids_file
    restore_by_ids "$ids_file"
    ;;
  2)
    read -p "jq query (e.g., 'select(.status == \"active\")'): " query
    restore_by_criteria "$query"
    ;;
  3)
    restore_with_transformation
    ;;
  *)
    log "Invalid choice"
    exit 1
    ;;
esac

log ""
log "========================================================"
log "✓ DOCUMENT-LEVEL RESTORATION COMPLETED"
log "========================================================"
```

## Gestion de la Cohérence

### Problèmes de Cohérence Référentielle

```javascript
// check_referential_integrity.js

use mydb

function checkReferentialIntegrity() {
  print("=== Referential Integrity Check ===");
  print("");

  // Exemple: Vérifier que tous les orders ont un user valide
  print("Checking orders -> users relationship...");

  var orphanedOrders = db.orders.aggregate([
    {
      $lookup: {
        from: "users",
        localField: "user_id",
        foreignField: "_id",
        as: "user"
      }
    },
    {
      $match: {
        user: { $eq: [] }  // Pas de user trouvé
      }
    },
    {
      $project: {
        _id: 1,
        user_id: 1,
        order_date: 1
      }
    }
  ]).toArray();

  if (orphanedOrders.length > 0) {
    print("⚠️  Found " + orphanedOrders.length + " orphaned orders");
    print("Sample:");
    orphanedOrders.slice(0, 5).forEach(printjson);

    print("\nOptions:");
    print("  1. Remove orphaned orders:");
    print("     db.orders.deleteMany({ _id: { $in: [" +
          orphanedOrders.map(o => "ObjectId('" + o._id + "')").join(", ") + "] } })");
    print("  2. Fix user references");
    print("  3. Restore missing users");
  } else {
    print("✓ All orders have valid user references");
  }

  print("");

  // Vérifier products -> categories
  print("Checking products -> categories relationship...");

  var orphanedProducts = db.products.aggregate([
    {
      $lookup: {
        from: "categories",
        localField: "category_id",
        foreignField: "_id",
        as: "category"
      }
    },
    {
      $match: {
        category: { $eq: [] }
      }
    },
    {
      $count: "orphaned"
    }
  ]).toArray();

  var orphanedCount = orphanedProducts.length > 0 ? orphanedProducts[0].orphaned : 0;

  if (orphanedCount > 0) {
    print("⚠️  Found " + orphanedCount + " products without valid category");
  } else {
    print("✓ All products have valid category references");
  }

  print("");
  print("=== Check Complete ===");
}

checkReferentialIntegrity();
```

### Script de Réconciliation

```bash
#!/bin/bash
# reconcile_partial_restore.sh

MONGO_URI="mongodb://localhost:27017"
TARGET_DB="mydb"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Post-Restoration Reconciliation ==="

# Vérifier la cohérence
mongo "$MONGO_URI/$TARGET_DB" <<'EOF'
  print("=== Data Reconciliation ===\n");

  // 1. Vérifier les compteurs
  print("1. Checking counters...");
  var userCount = db.users.countDocuments();
  var orderCount = db.orders.countDocuments();
  var productCount = db.products.countDocuments();

  print("  Users: " + userCount);
  print("  Orders: " + orderCount);
  print("  Products: " + productCount);

  // 2. Vérifier les dates
  print("\n2. Checking date ranges...");

  var dateStats = db.orders.aggregate([
    {
      $group: {
        _id: null,
        minDate: { $min: "$created_at" },
        maxDate: { $max: "$created_at" },
        count: { $sum: 1 }
      }
    }
  ]).toArray()[0];

  if (dateStats) {
    print("  Orders date range:");
    print("    Min: " + dateStats.minDate);
    print("    Max: " + dateStats.maxDate);
    print("    Count: " + dateStats.count);
  }

  // 3. Vérifier les index
  print("\n3. Checking indexes...");

  ["users", "orders", "products"].forEach(function(coll) {
    var indexes = db[coll].getIndexes();
    print("  " + coll + ": " + indexes.length + " indexes");

    indexes.forEach(function(idx) {
      if (idx.name !== "_id_") {
        print("    - " + idx.name);
      }
    });
  });

  // 4. Détecter les anomalies
  print("\n4. Detecting anomalies...");

  // Commandes sans produits
  var emptyOrders = db.orders.countDocuments({
    $or: [
      { items: { $exists: false } },
      { items: { $eq: [] } }
    ]
  });

  if (emptyOrders > 0) {
    print("  ⚠️  " + emptyOrders + " orders without items");
  }

  // Produits avec prix négatifs
  var invalidPrices = db.products.countDocuments({
    price: { $lte: 0 }
  });

  if (invalidPrices > 0) {
    print("  ⚠️  " + invalidPrices + " products with invalid prices");
  }

  print("\n=== Reconciliation Complete ===");
EOF

log "✓ Reconciliation completed"
```

## Stratégies d'Impact Minimal

### Restauration Sans Downtime

```bash
#!/bin/bash
# zero_downtime_restore.sh

MONGO_PRIMARY="mongodb://primary:27017"
MONGO_SECONDARY="mongodb://secondary:27017"
BACKUP_DIR="/backup/mongodb/latest"
TARGET_DB="mydb"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Zero-Downtime Restoration Strategy ==="

# ============================================================================
# STRATÉGIE: RESTAURER SUR SECONDARY → PROMOTE → SWITCH
# ============================================================================

log "Step 1: Restore on secondary member (read-only)"

# Le secondary peut être pris hors service temporairement
mongo "$MONGO_SECONDARY/admin" --eval "
  db.adminCommand({ replSetMaintenance: 1 });
"

# Restaurer sur le secondary
mongorestore \
  --uri="$MONGO_SECONDARY" \
  --db="$TARGET_DB" \
  --drop \
  --gzip \
  --dir="$BACKUP_DIR/$TARGET_DB"

log "✓ Restored on secondary"

# Sortir du mode maintenance
mongo "$MONGO_SECONDARY/admin" --eval "
  db.adminCommand({ replSetMaintenance: 0 });
"

log "Step 2: Wait for replication to primary"
sleep 30

log "Step 3: Application switch to secondary for reads (optional)"
# L'application peut être configurée pour lire depuis le secondary

log "Step 4: Validation"
# Valider les données sur le secondary avant de continuer

log "✓ Zero-downtime restoration completed"
log "  Primary continues serving writes"
log "  Reads can be directed to secondary with restored data"
```

### Restauration Parallèle

```bash
#!/bin/bash
# parallel_partial_restore.sh

BACKUP_DIR="/backup/mongodb/latest"
MONGO_URI="mongodb://localhost:27017"
TARGET_DB="mydb"
COLLECTIONS=("users" "orders" "products" "inventory" "logs")

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Parallel Partial Restoration ==="
log "Collections: ${COLLECTIONS[@]}"

# Fonction de restauration d'une collection
restore_collection() {
  local collection=$1
  local log_file="/tmp/restore_${collection}.log"

  echo "[$(date +'%Y-%m-%d %H:%M:%S')] Starting: $collection" >> "$log_file"

  mongorestore \
    --uri="$MONGO_URI" \
    --db="$TARGET_DB" \
    --collection="$collection" \
    --drop \
    --gzip \
    "$BACKUP_DIR/$TARGET_DB/$collection.bson.gz" \
    >> "$log_file" 2>&1

  local exit_code=$?

  if [ $exit_code -eq 0 ]; then
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] ✓ Completed: $collection" >> "$log_file"
  else
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] ✗ Failed: $collection" >> "$log_file"
  fi

  return $exit_code
}

# Lancer les restaurations en parallèle
pids=()

for collection in "${COLLECTIONS[@]}"; do
  restore_collection "$collection" &
  pids+=($!)

  log "Launched restoration for: $collection (PID: ${pids[-1]})"

  # Limiter la concurrence (ex: max 3 en parallèle)
  if [ ${#pids[@]} -ge 3 ]; then
    wait ${pids[0]}
    pids=("${pids[@]:1}")
  fi
done

# Attendre tous les processus
log "Waiting for all restorations to complete..."
for pid in "${pids[@]}"; do
  wait $pid
done

log "✓ All restorations completed"

# Afficher les résultats
log ""
log "=== Restoration Results ==="
for collection in "${COLLECTIONS[@]}"; do
  if grep -q "✓ Completed" "/tmp/restore_${collection}.log"; then
    log "  ✓ $collection"
  else
    log "  ✗ $collection (check /tmp/restore_${collection}.log)"
  fi
done
```

## Cas d'Usage Pratiques

### Scénario 1 : Restauration après Suppression Accidentelle

```bash
#!/bin/bash
# scenario_accidental_deletion.sh

# Contexte: Un administrateur a accidentellement supprimé
# tous les orders de décembre 2024

BACKUP_DIR="/backup/mongodb/backup_20241207"
MONGO_URI="mongodb://production:27017"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Scenario: Accidental Deletion Recovery ==="
log "Recovering December 2024 orders"

# Extraire les documents supprimés
TEMP_JSON="/tmp/deleted_orders.json"

bsondump "$BACKUP_DIR/mydb/orders.bson.gz" | \
  jq -c 'select(.order_date >= "2024-12-01" and .order_date < "2025-01-01")' \
  > "$TEMP_JSON"

RECOVERED_COUNT=$(wc -l < "$TEMP_JSON")

log "Found $RECOVERED_COUNT orders to recover"

if [ $RECOVERED_COUNT -eq 0 ]; then
  log "No orders found in the date range"
  exit 0
fi

# Afficher échantillon
log "Sample orders:"
head -3 "$TEMP_JSON" | jq .

read -p "Restore these $RECOVERED_COUNT orders? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
  log "Recovery cancelled"
  exit 0
fi

# Restaurer avec upsert (au cas où certains existent encore)
mongoimport \
  --uri="$MONGO_URI" \
  --db=mydb \
  --collection=orders \
  --file="$TEMP_JSON" \
  --mode=upsert \
  --upsertFields="_id"

log "✓ Recovered $RECOVERED_COUNT orders"

# Cleanup
rm -f "$TEMP_JSON"
```

### Scénario 2 : Migration de Données Entre Environnements

```bash
#!/bin/bash
# scenario_env_migration.sh

# Contexte: Copier des données de production vers staging
# pour debugging, tout en anonymisant les données sensibles

SOURCE_URI="mongodb://production:27017"
TARGET_URI="mongodb://staging:27017"
TARGET_DB="mydb_staging"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Scenario: Production to Staging Migration ==="

# Exporter depuis production
TEMP_EXPORT="/tmp/prod_export_$$"
mkdir -p "$TEMP_EXPORT"

log "Exporting from production..."
mongodump \
  --uri="$SOURCE_URI" \
  --db=mydb \
  --gzip \
  --out="$TEMP_EXPORT"

# Anonymiser les données sensibles
log "Anonymizing sensitive data..."

# Convertir en JSON, anonymiser, reconvertir
for collection in users orders; do
  BSON_FILE="$TEMP_EXPORT/mydb/$collection.bson.gz"
  JSON_FILE="$TEMP_EXPORT/$collection.json"

  # Extraire
  bsondump "$BSON_FILE" > "$JSON_FILE"

  # Anonymiser
  jq -c '
    # Masquer emails
    if .email then
      .email = ("user" + (._id."$oid" | .[0:8]) + "@example.com")
    else . end |

    # Masquer noms
    if .name then
      .name = ("User " + (._id."$oid" | .[0:8]))
    else . end |

    # Masquer téléphones
    if .phone then
      .phone = "+33600000000"
    else . end |

    # Masquer adresses
    if .address then
      .address = "123 Anonymized Street"
    else . end
  ' "$JSON_FILE" > "${JSON_FILE}.anonymized"

  # Importer dans staging
  mongoimport \
    --uri="$TARGET_URI" \
    --db="$TARGET_DB" \
    --collection="$collection" \
    --drop \
    --file="${JSON_FILE}.anonymized"
done

log "✓ Data migrated and anonymized"

# Cleanup
rm -rf "$TEMP_EXPORT"
```

## Checklist et Bonnes Pratiques

### Checklist Restauration Complète

```markdown
### Avant la Restauration

- [ ] Identifier la cause racine de l'incident
- [ ] Confirmer que restauration complète est nécessaire
- [ ] Valider que le backup est complet et récent
- [ ] Vérifier l'intégrité du backup (checksums)
- [ ] Documenter l'état actuel (screenshots, exports)
- [ ] Coordonner avec les équipes (dev, ops, business)
- [ ] Communiquer le downtime estimé
- [ ] Planifier fenêtre de maintenance

### Pendant la Restauration

- [ ] Arrêter toutes les connexions applicatives
- [ ] Sauvegarder l'état actuel (safety backup)
- [ ] Exécuter la restauration selon procédure
- [ ] Monitorer les logs
- [ ] Vérifier l'espace disque
- [ ] Ne pas interrompre le processus

### Après la Restauration

- [ ] Valider l'intégrité des données
- [ ] Vérifier les compteurs (documents, collections)
- [ ] Tester les requêtes critiques
- [ ] Vérifier les index
- [ ] Tester connectivité applicative
- [ ] Validation fonctionnelle par équipe métier
- [ ] Monitorer performances
- [ ] Documenter l'incident et la procédure
- [ ] Post-mortem
```

### Checklist Restauration Partielle

```markdown
### Avant la Restauration

- [ ] Identifier précisément les données à restaurer
- [ ] Vérifier impact sur cohérence référentielle
- [ ] Évaluer si restauration partielle est possible
- [ ] Documenter les dépendances
- [ ] Préparer plan de rollback
- [ ] Tester en environnement non-prod d'abord

### Pendant la Restauration

- [ ] Restaurer d'abord dans collection/db temporaire
- [ ] Valider les données restaurées
- [ ] Vérifier la cohérence
- [ ] Créer backup de sécurité avant remplacement
- [ ] Procéder au remplacement progressivement si possible

### Après la Restauration

- [ ] Vérifier cohérence référentielle
- [ ] Exécuter script de réconciliation
- [ ] Valider avec échantillons de données
- [ ] Tester fonctionnalités impactées
- [ ] Monitorer logs applicatifs
- [ ] Garder backup de sécurité 24-48h
```

## Conclusion

Le choix entre restauration complète et partielle dépend fortement du contexte de l'incident et des contraintes opérationnelles. Une restauration complète offre des garanties de cohérence maximales mais impose un downtime significatif, tandis qu'une restauration partielle minimise l'impact mais requiert une expertise plus poussée et comporte des risques de cohérence.

**Principes directeurs** :

1. **Évaluer d'abord** - Analyser l'incident avant de choisir l'approche
2. **Tester en isolation** - Toujours valider en environnement test d'abord
3. **Backup de sécurité** - Créer un safety backup avant toute opération
4. **Validation rigoureuse** - Ne pas se fier aux compteurs seuls
5. **Documentation** - Tracer chaque étape pour post-mortem

**Quand choisir quoi** :

- **Restauration complète** : Corruption généralisée, disaster, besoin de cohérence absolue, environnement peut être arrêté
- **Restauration partielle** : Erreur ciblée, données spécifiques corrompues, service doit rester en ligne, expertise disponible

La maîtrise de ces deux approches et de leurs variantes est essentielle pour toute stratégie de continuité d'activité MongoDB robuste.

---


⏭️ [Automatisation des sauvegardes](/12-sauvegarde-restauration/10-automatisation-sauvegardes.md)
