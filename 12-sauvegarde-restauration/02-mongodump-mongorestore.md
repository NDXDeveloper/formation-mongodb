🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.2 mongodump et mongorestore

## Introduction

`mongodump` et `mongorestore` constituent les outils de sauvegarde et restauration logiques de référence pour MongoDB. Contrairement aux snapshots physiques, ces outils opèrent au niveau des documents BSON, offrant portabilité, flexibilité et granularité fine dans les opérations de backup et recovery.

Ces utilitaires font partie de la suite **MongoDB Database Tools**, séparée du serveur MongoDB depuis la version 4.4, et sont essentiels pour toute stratégie de sauvegarde en production.

## Concepts Fondamentaux

### Architecture et Fonctionnement

```
┌───────────────────────────────────────────────────────────┐
│                      mongodump                            │
│                                                           │
│  ┌────────────┐    ┌──────────────┐    ┌───────────────┐  │
│  │  MongoDB   │ -> │ Extraction   │ -> │ Compression   │  │
│  │  Source    │    │ BSON + JSON  │    │ (gzip/zstd)   │  │
│  └────────────┘    └──────────────┘    └───────────────┘  │
│                            │                    │         │
│                            v                    v         │
│                    ┌──────────────────────────────┐       │
│                    │    Fichiers de Sortie        │       │
│                    │  - *.bson (données)          │       │
│                    │  - *.metadata.json (schéma)  │       │
│                    │  - oplog.bson (si --oplog)   │       │
│                    └──────────────────────────────┘       │
└───────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    mongorestore                             │
│                                                             │
│  ┌────────────────┐   ┌───────────────┐   ┌──────────────┐  │
│  │ Fichiers       │-> │ Décompression │-> │ Validation   │  │
│  │ Backup         │   │ & Parsing     │   │ & Transform  │  │
│  └────────────────┘   └───────────────┘   └──────────────┘  │
│                                │                            │
│                                v                            │
│                    ┌──────────────────────┐                 │
│                    │  Insertion MongoDB   │                 │
│                    │  - Bulk operations   │                 │
│                    │  - Index rebuild     │                 │
│                    │  - Oplog replay      │                 │
│                    └──────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

### Garanties et Cohérence

**mongodump** offre plusieurs niveaux de cohérence :

```javascript
// Sans --oplog: Cohérence par collection
// Chaque collection est cohérente, mais pas entre collections
// Acceptable pour: Dev, analytics, exports ponctuels

// Avec --oplog: Cohérence point-in-time
// Capture l'état à un instant T précis via l'oplog
// Requis pour: Production, backups critiques

// Exemple de timestamp capturé
{
  "ts": Timestamp(1701993600, 42),  // Instant exact du backup
  "t": NumberLong(23),               // Terme de réplication
  "h": NumberLong("987654321"),      // Hash
  "v": 2
}
```

## mongodump - Sauvegarde Complète

### Syntaxe et Options Principales

```bash
mongodump [options]

# Options de connexion
--uri="mongodb://user:pass@host:27017/dbname?authSource=admin"
--host=hostname:port
--username=user
--password=pass
--authenticationDatabase=admin

# Options de sélection
--db=database_name              # Base spécifique
--collection=collection_name    # Collection spécifique
--query='{"field": "value"}'    # Filtrage de documents
--queryFile=/path/to/query.json # Requête depuis fichier
--excludeCollection=col1        # Exclure une collection
--excludeCollectionsWithPrefix=temp_  # Exclure par préfixe

# Options de sortie
--out=/path/to/backup           # Répertoire de sortie
--archive=/path/to/file.gz      # Archive unique
--gzip                          # Compression gzip
--oplog                         # Capturer oplog pour cohérence

# Options de performance
--numParallelCollections=4      # Collections en parallèle
--readPreference=secondary      # Préférence de lecture

# Options avancées
--viewsAsCollections            # Exporter les vues comme collections
--dumpDbUsersAndRoles          # Inclure users/roles
```

### Exemples de Base

**Sauvegarde complète de toutes les bases** :
```bash
# Backup complet avec compression et oplog
mongodump \
  --uri="mongodb://backup-user:SecurePass123@mongodb-secondary:27017/?authSource=admin" \
  --oplog \
  --gzip \
  --out=/backup/mongodb/dump_$(date +%Y%m%d_%H%M%S)

# Vérification du résultat
ls -lh /backup/mongodb/dump_20241208_143025/
# admin/
# config/
# myapp_db/
# oplog.bson
```

**Sauvegarde d'une base spécifique** :
```bash
# Sauvegarde uniquement la base "production_db"
mongodump \
  --host=mongodb-primary:27017 \
  --db=production_db \
  --username=admin \
  --password=SecurePass \
  --authenticationDatabase=admin \
  --gzip \
  --out=/backup/production_db_$(date +%Y%m%d)
```

**Sauvegarde d'une collection avec filtrage** :
```bash
# Exporter seulement les commandes actives
mongodump \
  --db=ecommerce \
  --collection=orders \
  --query='{"status": "active", "created_at": {"$gte": {"$date": "2024-01-01T00:00:00Z"}}}' \
  --gzip \
  --out=/backup/active_orders
```

### Mode Archive

Le mode archive crée un fichier unique au lieu d'une structure de répertoires :

```bash
# Archive complète
mongodump \
  --uri="mongodb://localhost:27017" \
  --oplog \
  --gzip \
  --archive=/backup/mongo_full_$(date +%Y%m%d).gz

# Archive vers stdout (streaming vers remote)
mongodump \
  --uri="mongodb://localhost:27017" \
  --oplog \
  --archive \
  | ssh backup-server "cat > /remote/backup/mongo_$(date +%Y%m%d).archive"

# Archive avec encryption à la volée
mongodump \
  --uri="mongodb://localhost:27017" \
  --oplog \
  --archive \
  | gzip \
  | openssl enc -aes-256-cbc -salt -pbkdf2 \
    -out /backup/encrypted_$(date +%Y%m%d).gz.enc
```

### Backup de Replica Set

```bash
#!/bin/bash
# replica_set_backup.sh

RS_NAME="rs0"
RS_MEMBERS="mongodb-1:27017,mongodb-2:27017,mongodb-3:27017"
BACKUP_DIR="/backup/mongodb"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Identifier un secondary pour le backup
SECONDARY=$(mongo "mongodb://${RS_MEMBERS}/?replicaSet=${RS_NAME}" --quiet --eval "
  rs.status().members.forEach(function(member) {
    if (member.stateStr === 'SECONDARY' && !member.hidden) {
      print(member.name);
    }
  });
" | head -1)

if [ -z "$SECONDARY" ]; then
  echo "ERROR: No secondary member available"
  exit 1
fi

echo "Using secondary: $SECONDARY"

# Backup depuis le secondary
mongodump \
  --host="$SECONDARY" \
  --oplog \
  --gzip \
  --numParallelCollections=4 \
  --readPreference='{ mode: "secondary", tagSets: [ { "backup": "true" } ] }' \
  --out="${BACKUP_DIR}/rs_backup_${TIMESTAMP}"

# Vérifier la cohérence
if [ $? -eq 0 ]; then
  # Enregistrer les métadonnées du replica set
  mongo "mongodb://${RS_MEMBERS}/?replicaSet=${RS_NAME}" --quiet > \
    "${BACKUP_DIR}/rs_backup_${TIMESTAMP}/replica_set_status.json" <<EOF
    printjson({
      "replSetConfig": rs.conf(),
      "replSetStatus": rs.status(),
      "timestamp": new Date()
    });
EOF

  echo "✓ Replica Set backup completed successfully"
  echo "  Location: ${BACKUP_DIR}/rs_backup_${TIMESTAMP}"
else
  echo "✗ Backup failed"
  exit 1
fi
```

### Backup de Sharded Cluster

Pour un cluster shardé, une coordination spéciale est nécessaire :

```bash
#!/bin/bash
# sharded_backup_comprehensive.sh

MONGOS_URI="mongodb://admin:pass@mongos:27017/admin"
BACKUP_ROOT="/backup/sharded"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_ROOT}/${TIMESTAMP}"

mkdir -p "$BACKUP_DIR"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$BACKUP_DIR/backup.log"
}

log "Starting sharded cluster backup"

# 1. Arrêter le balancer
log "Stopping balancer..."
mongo "$MONGOS_URI" --eval "sh.stopBalancer()"

# Attendre arrêt complet
while true; do
  IS_RUNNING=$(mongo "$MONGOS_URI" --quiet --eval "sh.isBalancerRunning()")
  if [ "$IS_RUNNING" = "false" ]; then
    break
  fi
  sleep 2
done
log "Balancer stopped"

# 2. Obtenir la liste des shards
SHARDS=$(mongo "$MONGOS_URI" --quiet --eval "
  db.getSiblingDB('config').shards.find({}, {_id:1, host:1}).forEach(function(doc) {
    print(doc._id + '|' + doc.host);
  });
")

# 3. Backup de chaque shard en parallèle
log "Backing up shards..."
PIDS=()
while IFS='|' read -r shard_id shard_host; do
  log "  Starting backup of shard: $shard_id"

  (
    mongodump \
      --uri="mongodb://$shard_host" \
      --oplog \
      --gzip \
      --out="$BACKUP_DIR/shard_${shard_id}" \
      2>&1 | tee "$BACKUP_DIR/shard_${shard_id}.log"
  ) &

  PIDS+=($!)
done <<< "$SHARDS"

# Attendre tous les backups de shards
for pid in "${PIDS[@]}"; do
  wait $pid
  if [ $? -ne 0 ]; then
    log "ERROR: Shard backup failed (PID: $pid)"
  fi
done

# 4. Backup des config servers
log "Backing up config servers..."
CONFIG_SERVERS=$(mongo "$MONGOS_URI" --quiet --eval "
  db.adminCommand({ getShardMap: 1 }).map.config;
")

mongodump \
  --uri="mongodb://$CONFIG_SERVERS" \
  --oplog \
  --gzip \
  --out="$BACKUP_DIR/config_servers"

# 5. Sauvegarder les métadonnées complètes
log "Saving cluster metadata..."
mongo "$MONGOS_URI" > "$BACKUP_DIR/cluster_metadata.json" <<'EOF'
printjson({
  "shardingStatus": sh.status(),
  "databases": db.getSiblingDB('config').databases.find().toArray(),
  "collections": db.getSiblingDB('config').collections.find().toArray(),
  "chunks": db.getSiblingDB('config').chunks.count(),
  "timestamp": new Date()
});
EOF

# 6. Redémarrer le balancer
log "Restarting balancer..."
mongo "$MONGOS_URI" --eval "sh.startBalancer()"

# 7. Créer archive complète
log "Creating consolidated archive..."
tar -czf "${BACKUP_DIR}.tar.gz" -C "$BACKUP_ROOT" "$(basename $BACKUP_DIR)"

# Checksum
sha256sum "${BACKUP_DIR}.tar.gz" > "${BACKUP_DIR}.tar.gz.sha256"

log "✓ Sharded cluster backup completed"
log "  Archive: ${BACKUP_DIR}.tar.gz"
log "  Size: $(du -h ${BACKUP_DIR}.tar.gz | cut -f1)"
```

### Options Avancées de Performance

**Parallélisation optimale** :
```bash
# Calculer le nombre optimal de threads
COLLECTION_COUNT=$(mongo --quiet --eval "db.getCollectionNames().length")
CPU_COUNT=$(nproc)
OPTIMAL_THREADS=$(( COLLECTION_COUNT < CPU_COUNT ? COLLECTION_COUNT : CPU_COUNT ))

mongodump \
  --uri="mongodb://localhost:27017" \
  --numParallelCollections=$OPTIMAL_THREADS \
  --gzip \
  --out=/backup/parallel_optimized
```

**Backup avec limitation de ressources** :
```bash
# Limiter l'impact CPU avec nice et ionice
nice -n 19 ionice -c 3 mongodump \
  --uri="mongodb://localhost:27017" \
  --gzip \
  --out=/backup/low_priority

# Limiter la bande passante réseau
trickle -d 50000 -u 50000 mongodump \  # 50 MB/s max
  --uri="mongodb://remote:27017" \
  --archive | gzip > /backup/throttled.gz
```

**Exclure des collections temporaires** :
```bash
#!/bin/bash
# Backup excluant les collections temporaires et de cache

mongodump \
  --uri="mongodb://localhost:27017" \
  --db=production_db \
  --excludeCollection=cache_* \
  --excludeCollection=temp_* \
  --excludeCollection=sessions \
  --gzip \
  --out=/backup/production_clean
```

## mongorestore - Restauration

### Syntaxe et Options Principales

```bash
mongorestore [options] [directory|file]

# Options de connexion
--uri="mongodb://user:pass@host:27017/dbname?authSource=admin"
--host=hostname:port
--username=user
--password=pass

# Options de sélection
--db=target_database           # Base cible
--collection=target_collection # Collection cible
--nsInclude=pattern           # Inclure namespace
--nsExclude=pattern           # Exclure namespace
--nsFrom=source_ns            # Renommer depuis
--nsTo=target_ns              # Renommer vers

# Options de restauration
--drop                        # Supprimer collections avant restauration
--oplogReplay                 # Rejouer l'oplog
--oplogLimit="timestamp"      # Limite temporelle oplog
--archive=/path/to/file       # Restaurer depuis archive
--gzip                        # Décompresser gzip

# Options de performance
--numParallelCollections=4    # Collections en parallèle
--numInsertionWorkers=1       # Workers par collection
--bypassDocumentValidation    # Ignorer validation schéma

# Options avancées
--maintainInsertionOrder      # Préserver l'ordre d'insertion
--stopOnError                 # Arrêter sur erreur
--writeConcern="{w: 'majority'}"  # Niveau d'écriture
```

### Scénarios de Restauration

#### Restauration Complète

```bash
# Restauration complète depuis répertoire
mongorestore \
  --uri="mongodb://localhost:27017" \
  --drop \
  --gzip \
  /backup/mongodb/dump_20241208_143025/

# Depuis archive
mongorestore \
  --uri="mongodb://localhost:27017" \
  --drop \
  --gzip \
  --archive=/backup/mongo_full_20241208.gz

# Depuis archive distante
ssh backup-server "cat /backup/mongo.archive" | \
  mongorestore \
    --uri="mongodb://localhost:27017" \
    --archive \
    --drop
```

#### Restauration Sélective

```bash
# Restaurer une seule base
mongorestore \
  --uri="mongodb://localhost:27017" \
  --db=production_db \
  --drop \
  --gzip \
  /backup/mongodb/dump_20241208/production_db/

# Restaurer une seule collection
mongorestore \
  --uri="mongodb://localhost:27017" \
  --db=production_db \
  --collection=users \
  --drop \
  /backup/mongodb/dump_20241208/production_db/users.bson.gz

# Restaurer avec renommage
mongorestore \
  --uri="mongodb://localhost:27017" \
  --nsFrom="prod.orders" \
  --nsTo="archive.orders_2024" \
  /backup/mongodb/dump/prod/orders.bson
```

#### Restauration Point-in-Time

La restauration point-in-time utilise l'oplog pour rejouer les opérations jusqu'à un timestamp spécifique :

```bash
#!/bin/bash
# point_in_time_restore.sh

TARGET_TIMESTAMP="2024-12-08T14:30:00Z"
BACKUP_DIR="/backup/mongodb/dump_20241208_120000"
RESTORE_HOST="mongodb://localhost:27017"

# Convertir le timestamp en format MongoDB
TARGET_TS=$(date -d "$TARGET_TIMESTAMP" +%s)

echo "Restoring to point-in-time: $TARGET_TIMESTAMP"

# 1. Restauration de base depuis le backup complet
echo "Step 1: Restoring base backup..."
mongorestore \
  --uri="$RESTORE_HOST" \
  --drop \
  --gzip \
  --oplogReplay \
  --oplogLimit="$TARGET_TS:0" \
  "$BACKUP_DIR"

# 2. Vérification
echo "Step 2: Verifying restoration..."
mongo "$RESTORE_HOST" --eval "
  db.adminCommand({
    listDatabases: 1
  }).databases.forEach(function(db) {
    print('Database: ' + db.name + ', Size: ' + db.sizeOnDisk);
  });
"

echo "✓ Point-in-time restore completed to $TARGET_TIMESTAMP"
```

#### Restauration avec Transformation

```bash
# Restaurer en changeant le namespace (renommer)
mongorestore \
  --uri="mongodb://localhost:27017" \
  --nsFrom="production.*" \
  --nsTo="staging.*" \
  --drop \
  /backup/production/

# Restaurer dans une base différente
mongorestore \
  --uri="mongodb://localhost:27017" \
  --db=test_db \
  --drop \
  /backup/dump/production_db/

# Exclure certaines collections lors de la restauration
mongorestore \
  --uri="mongodb://localhost:27017" \
  --nsExclude="production.temp_*" \
  --nsExclude="production.cache_*" \
  --drop \
  /backup/production/
```

### Restauration de Replica Set

```bash
#!/bin/bash
# restore_replica_set.sh

BACKUP_DIR="/backup/mongodb/rs_backup_20241208"
PRIMARY_URI="mongodb://mongodb-1:27017,mongodb-2:27017,mongodb-3:27017/?replicaSet=rs0"

# Vérifier que nous sommes sur un secondary ou standalone pour la restauration
echo "Checking replica set status..."

CURRENT_ROLE=$(mongo "$PRIMARY_URI" --quiet --eval "
  rs.isMaster().ismaster ? 'primary' : 'secondary'
")

if [ "$CURRENT_ROLE" = "primary" ]; then
  echo "WARNING: Restoring on PRIMARY. Consider using a secondary or isolated instance."
  read -p "Continue? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    exit 1
  fi
fi

# Restauration
echo "Starting restoration..."
mongorestore \
  --uri="$PRIMARY_URI" \
  --drop \
  --gzip \
  --oplogReplay \
  --numParallelCollections=4 \
  --writeConcern='{w: "majority", j: true}' \
  "$BACKUP_DIR"

if [ $? -eq 0 ]; then
  echo "✓ Replica Set restored successfully"

  # Vérifier la synchronisation
  echo "Checking replication status..."
  mongo "$PRIMARY_URI" --eval "
    rs.status().members.forEach(function(m) {
      print(m.name + ': ' + m.stateStr + ', lag: ' +
            (m.optimeDate ? (new Date() - m.optimeDate) : 'N/A') + 'ms');
    });
  "
else
  echo "✗ Restoration failed"
  exit 1
fi
```

### Restauration de Sharded Cluster

La restauration d'un cluster shardé est complexe et nécessite une procédure spécifique :

```bash
#!/bin/bash
# restore_sharded_cluster.sh

BACKUP_DIR="/backup/sharded/20241208_120000"
MONGOS_URI="mongodb://mongos:27017"

echo "=== Sharded Cluster Restoration ==="
echo "Backup: $BACKUP_DIR"
echo ""

# 1. Vérifier que le balancer est arrêté
echo "Step 1: Ensuring balancer is stopped..."
mongo "$MONGOS_URI" --eval "sh.stopBalancer()" admin

# 2. Restaurer les config servers en premier
echo "Step 2: Restoring config servers..."
CONFIG_SERVERS=$(mongo "$MONGOS_URI" --quiet --eval "
  db.adminCommand({ getShardMap: 1 }).map.config;
")

mongorestore \
  --uri="mongodb://$CONFIG_SERVERS" \
  --drop \
  --oplogReplay \
  --gzip \
  "$BACKUP_DIR/config_servers"

# 3. Restaurer chaque shard
echo "Step 3: Restoring shards..."
SHARDS=$(ls -1 "$BACKUP_DIR" | grep "^shard_")

for shard_dir in $SHARDS; do
  shard_name=$(basename "$shard_dir" | sed 's/^shard_//')

  # Obtenir l'URI du shard depuis la config
  shard_uri=$(mongo "$MONGOS_URI" --quiet --eval "
    db.getSiblingDB('config').shards.findOne({_id: '$shard_name'}).host;
  ")

  echo "  Restoring shard: $shard_name ($shard_uri)"

  mongorestore \
    --uri="mongodb://$shard_uri" \
    --drop \
    --oplogReplay \
    --gzip \
    "$BACKUP_DIR/$shard_dir" &
done

# Attendre toutes les restaurations
wait

# 4. Vérifier l'intégrité
echo "Step 4: Verifying cluster integrity..."
mongo "$MONGOS_URI" <<'EOF'
  use config
  print("Databases: " + db.databases.count());
  print("Collections: " + db.collections.count());
  print("Chunks: " + db.chunks.count());

  // Vérifier que tous les shards sont présents
  db.shards.find().forEach(function(shard) {
    print("Shard: " + shard._id + " - " + shard.host);
  });
EOF

# 5. Optionnel: Redémarrer le balancer
echo "Step 5: Restarting balancer..."
mongo "$MONGOS_URI" --eval "sh.startBalancer()" admin

echo ""
echo "✓ Sharded cluster restoration completed"
echo "  Verify data integrity before resuming production traffic"
```

## Gestion des Erreurs et Recovery

### Erreurs Communes et Solutions

**Erreur: Espace disque insuffisant**
```bash
# Vérifier l'espace avant restauration
BACKUP_SIZE=$(du -sb /backup/mongo.gz | cut -f1)
AVAILABLE_SPACE=$(df --output=avail -B1 /var/lib/mongodb | tail -1)

if [ $BACKUP_SIZE -gt $AVAILABLE_SPACE ]; then
  echo "ERROR: Insufficient disk space"
  echo "Required: $(numfmt --to=iec-i $BACKUP_SIZE)"
  echo "Available: $(numfmt --to=iec-i $AVAILABLE_SPACE)"
  exit 1
fi

# Restaurer avec monitoring d'espace
mongorestore --uri="mongodb://localhost:27017" /backup/dump/ &
PID=$!

while kill -0 $PID 2>/dev/null; do
  AVAIL=$(df --output=avail -B1 /var/lib/mongodb | tail -1)
  if [ $AVAIL -lt 1073741824 ]; then  # < 1GB
    echo "WARNING: Low disk space"
    kill $PID
    exit 1
  fi
  sleep 10
done
```

**Erreur: Conflit de données (duplicate key)**
```bash
# Option 1: Supprimer avant restauration
mongorestore --drop --uri="mongodb://localhost:27017" /backup/

# Option 2: Ignorer les erreurs de duplication (dangereux)
mongorestore \
  --stopOnError=false \
  --uri="mongodb://localhost:27017" \
  /backup/ 2>&1 | tee restore_errors.log

# Analyser les erreurs
grep "duplicate key error" restore_errors.log | wc -l
```

**Erreur: Version incompatible**
```bash
# Vérifier la version du backup
BACKUP_VERSION=$(cat /backup/dump/admin/system.version.bson | strings | grep -oP '\d+\.\d+\.\d+' | head -1)
SERVER_VERSION=$(mongo --quiet --eval "db.version()")

echo "Backup version: $BACKUP_VERSION"
echo "Server version: $SERVER_VERSION"

# La restauration cross-version est généralement supportée
# MongoDB 4.x backup -> MongoDB 5.x/6.x/7.x server: OK
# MongoDB 7.x backup -> MongoDB 4.x server: NOT OK
```

### Validation Post-Restauration

```bash
#!/bin/bash
# validate_restore.sh

MONGO_URI="mongodb://localhost:27017"

echo "=== Post-Restore Validation ==="

# 1. Vérifier les bases de données
echo "1. Checking databases..."
mongo "$MONGO_URI" --quiet --eval "
  db.adminCommand({ listDatabases: 1 }).databases.forEach(function(db) {
    if (db.name !== 'admin' && db.name !== 'local' && db.name !== 'config') {
      print('✓ Database: ' + db.name + ' (' +
            (db.sizeOnDisk / 1024 / 1024 / 1024).toFixed(2) + ' GB)');
    }
  });
"

# 2. Compter les documents
echo -e "\n2. Counting documents..."
mongo "$MONGO_URI" --quiet --eval "
  db.adminCommand({ listDatabases: 1 }).databases.forEach(function(database) {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      db.getSiblingDB(database.name).getCollectionNames().forEach(function(collection) {
        count = db.getSiblingDB(database.name)[collection].countDocuments();
        if (count > 0) {
          print('  ' + database.name + '.' + collection + ': ' + count + ' documents');
        }
      });
    }
  });
"

# 3. Vérifier les index
echo -e "\n3. Checking indexes..."
mongo "$MONGO_URI" --quiet --eval "
  db.adminCommand({ listDatabases: 1 }).databases.forEach(function(database) {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      db.getSiblingDB(database.name).getCollectionNames().forEach(function(collection) {
        indexes = db.getSiblingDB(database.name)[collection].getIndexes().length;
        print('  ' + database.name + '.' + collection + ': ' + indexes + ' indexes');
      });
    }
  });
"

# 4. Valider l'intégrité des collections
echo -e "\n4. Validating collections (sample)..."
mongo "$MONGO_URI" --quiet --eval "
  // Valider quelques collections importantes
  ['production_db.users', 'production_db.orders', 'production_db.products'].forEach(function(ns) {
    parts = ns.split('.');
    result = db.getSiblingDB(parts[0])[parts[1]].validate({ full: true });
    if (result.valid) {
      print('✓ ' + ns + ' is valid');
    } else {
      print('✗ ' + ns + ' has errors: ' + JSON.stringify(result.errors));
    }
  });
"

# 5. Test de requête fonctionnelle
echo -e "\n5. Running functional tests..."
mongo "$MONGO_URI" --quiet --eval "
  use production_db

  // Test 1: Lecture simple
  user = db.users.findOne();
  if (user) {
    print('✓ Test 1 passed: Can read documents');
  } else {
    print('✗ Test 1 failed: Cannot read documents');
  }

  // Test 2: Agrégation
  count = db.orders.aggregate([
    { \$match: { status: 'active' } },
    { \$count: 'total' }
  ]).toArray()[0];

  if (count) {
    print('✓ Test 2 passed: Aggregation works (' + count.total + ' active orders)');
  }
"

echo -e "\n✓ Validation completed"
```

## Optimisations et Bonnes Pratiques

### Performance de Backup

**Optimisation par ordre de magnitude** :

```bash
# Petit (<10GB): Backup simple rapide
mongodump --gzip --archive=/backup/small.gz

# Moyen (10-100GB): Parallélisation
mongodump \
  --numParallelCollections=8 \
  --gzip \
  --archive=/backup/medium.gz

# Grand (>100GB): Parallélisation + compression rapide
mongodump \
  --numParallelCollections=16 \
  --archive \
  | lz4 > /backup/large.lz4

# Très grand (>1TB): Streaming vers stockage distant
mongodump \
  --archive \
  | pigz -p 16 \
  | aws s3 cp - s3://backups/mongo_$(date +%Y%m%d).gz \
    --expected-size 1099511627776
```

### Performance de Restauration

```bash
# Optimisation de la restauration
mongorestore \
  --numParallelCollections=8 \
  --numInsertionWorkers=4 \
  --bypassDocumentValidation \
  --writeConcern='{w: 1}' \  # Plus rapide (moins sûr)
  --gzip \
  /backup/dump/

# Pour environnements critiques (plus lent mais plus sûr)
mongorestore \
  --numParallelCollections=4 \
  --maintainInsertionOrder \
  --writeConcern='{w: "majority", j: true}' \
  --gzip \
  /backup/dump/
```

### Backup Incrémental Simulé

mongodump ne supporte pas nativement les backups incrémentaux, mais on peut les simuler :

```bash
#!/bin/bash
# incremental_backup_simulation.sh

STATE_FILE="/var/lib/mongo_backup/last_backup_time"
BACKUP_DIR="/backup/incremental"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Lire le dernier timestamp de backup
if [ -f "$STATE_FILE" ]; then
  LAST_BACKUP=$(cat "$STATE_FILE")
  LAST_BACKUP_DATE=$(date -d "@$LAST_BACKUP" -Iseconds)
else
  # Premier backup
  LAST_BACKUP_DATE="2000-01-01T00:00:00Z"
fi

echo "Last backup: $LAST_BACKUP_DATE"
echo "Creating incremental backup..."

# Exporter seulement les documents modifiés
# (nécessite un champ updated_at ou _id avec ObjectId)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=production \
  --query="{\"updated_at\": {\"\$gte\": {\"\$date\": \"$LAST_BACKUP_DATE\"}}}" \
  --gzip \
  --out="$BACKUP_DIR/incremental_$TIMESTAMP"

# Sauvegarder le nouveau timestamp
date +%s > "$STATE_FILE"

echo "✓ Incremental backup completed"
```

### Backup Différentiel avec Oplog

```bash
#!/bin/bash
# differential_oplog_backup.sh

OPLOG_FILE="/backup/oplog/oplog_$(date +%Y%m%d_%H%M%S).bson"
STATE_FILE="/var/lib/mongo_backup/last_oplog_ts"

# Lire le dernier timestamp oplog sauvegardé
if [ -f "$STATE_FILE" ]; then
  LAST_TS=$(cat "$STATE_FILE")
else
  # Première sauvegarde: prendre le timestamp actuel
  LAST_TS=$(mongo --quiet --eval "
    db.getSiblingDB('local').oplog.rs.find().sort({ts:-1}).limit(1).toArray()[0].ts;
  ")
fi

echo "Exporting oplog from timestamp: $LAST_TS"

# Exporter les entrées oplog depuis le dernier backup
mongodump \
  --db=local \
  --collection=oplog.rs \
  --query="{\"ts\": {\"\$gt\": $LAST_TS}}" \
  --out=/backup/oplog/temp

# Compression
gzip /backup/oplog/temp/local/oplog.rs.bson
mv /backup/oplog/temp/local/oplog.rs.bson.gz "$OPLOG_FILE.gz"
rm -rf /backup/oplog/temp

# Sauvegarder le nouveau timestamp
CURRENT_TS=$(mongo --quiet --eval "
  db.getSiblingDB('local').oplog.rs.find().sort({ts:-1}).limit(1).toArray()[0].ts;
")
echo "$CURRENT_TS" > "$STATE_FILE"

echo "✓ Oplog differential backup completed"
echo "  File: $OPLOG_FILE.gz"
```

## Sécurité

### Chiffrement des Backups

```bash
#!/bin/bash
# encrypted_backup.sh

BACKUP_NAME="mongo_secure_$(date +%Y%m%d).gz.gpg"
GPG_RECIPIENT="backup@company.com"

# Backup avec chiffrement GPG
mongodump \
  --uri="mongodb://localhost:27017" \
  --oplog \
  --archive \
  | gzip \
  | gpg --encrypt --recipient "$GPG_RECIPIENT" \
  > "/backup/$BACKUP_NAME"

echo "✓ Encrypted backup created: $BACKUP_NAME"

# Pour restaurer
gpg --decrypt "/backup/$BACKUP_NAME" \
  | gunzip \
  | mongorestore --archive
```

### Validation des Backups Chiffrés

```bash
#!/bin/bash
# validate_encrypted_backup.sh

BACKUP_FILE="/backup/mongo_secure_20241208.gz.gpg"

echo "Validating encrypted backup..."

# Déchiffrer et vérifier sans restaurer
gpg --decrypt "$BACKUP_FILE" | gunzip | mongodump --archive --dryRun 2>&1 | grep -q "done dumping"

if [ $? -eq 0 ]; then
  echo "✓ Backup is valid and readable"
else
  echo "✗ Backup validation failed"
  exit 1
fi
```

### Contrôle d'Accès aux Backups

```bash
# Permissions strictes sur les backups
chmod 600 /backup/mongo_*.gz
chown mongodb-backup:mongodb-backup /backup/mongo_*.gz

# Audit des accès
auditctl -w /backup/mongodb/ -p wa -k mongodb_backup_access

# Vérifier les accès
ausearch -k mongodb_backup_access
```

## Automatisation Complète

### Script de Backup Production-Ready

```bash
#!/bin/bash
# production_backup_complete.sh

set -euo pipefail

# Configuration
BACKUP_ROOT="/backup/mongodb"
RETENTION_DAYS=30
MONGO_URI="mongodb://backup-user:SecurePass@secondary:27017/?authSource=admin"
S3_BUCKET="s3://company-backups/mongodb"
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK"
LOG_FILE="/var/log/mongodb_backup.log"

# Fonctions
log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

notify() {
  local status=$1
  local message=$2
  local color=$3

  curl -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"attachments\":[{\"color\":\"$color\",\"text\":\"$message\",\"footer\":\"MongoDB Backup\",\"ts\":$(date +%s)}]}"
}

cleanup_old_backups() {
  log "Cleaning up backups older than $RETENTION_DAYS days"
  find "$BACKUP_ROOT" -name "*.gz" -mtime +$RETENTION_DAYS -delete
  find "$BACKUP_ROOT" -name "*.json" -mtime +$RETENTION_DAYS -delete
}

# Main
main() {
  local timestamp=$(date +%Y%m%d_%H%M%S)
  local backup_file="$BACKUP_ROOT/mongo_${timestamp}.gz"
  local start_time=$(date +%s)

  log "=== Starting MongoDB Backup ==="

  # Pré-vérifications
  if ! mongo "$MONGO_URI" --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log "ERROR: Cannot connect to MongoDB"
    notify "failed" "❌ Backup failed: Cannot connect to MongoDB" "danger"
    exit 1
  fi

  # Espace disque
  required_space=$((100 * 1024 * 1024 * 1024))  # 100GB minimum
  available_space=$(df --output=avail -B1 "$BACKUP_ROOT" | tail -1)

  if [ "$available_space" -lt "$required_space" ]; then
    log "ERROR: Insufficient disk space"
    notify "failed" "❌ Backup failed: Insufficient disk space" "danger"
    exit 1
  fi

  # Exécution du backup
  log "Executing mongodump..."
  if mongodump \
    --uri="$MONGO_URI" \
    --oplog \
    --gzip \
    --numParallelCollections=8 \
    --archive="$backup_file" \
    2>&1 | tee -a "$LOG_FILE"; then

    local end_time=$(date +%s)
    local duration=$((end_time - start_time))
    local size=$(du -h "$backup_file" | cut -f1)

    log "Backup completed successfully in ${duration}s, size: $size"

    # Checksum
    sha256sum "$backup_file" > "${backup_file}.sha256"

    # Métadonnées
    cat > "${backup_file%.gz}.json" <<EOF
{
  "timestamp": "$(date -Iseconds)",
  "duration_seconds": $duration,
  "size_bytes": $(stat -c%s "$backup_file"),
  "size_human": "$size",
  "mongodb_version": "$(mongo --quiet --eval 'db.version()')",
  "checksum": "$(cut -d' ' -f1 ${backup_file}.sha256)"
}
EOF

    # Upload S3
    log "Uploading to S3..."
    if aws s3 cp "$backup_file" "$S3_BUCKET/" \
      --storage-class STANDARD_IA \
      --metadata "backup-date=${timestamp},size=${size}"; then

      log "S3 upload successful"

      # Cleanup
      cleanup_old_backups

      # Notification succès
      notify "success" "✅ Backup successful: $size in ${duration}s\nFile: mongo_${timestamp}.gz" "good"

      exit 0
    else
      log "ERROR: S3 upload failed"
      notify "warning" "⚠️ Backup created but S3 upload failed" "warning"
      exit 1
    fi
  else
    log "ERROR: mongodump failed"
    notify "failed" "❌ Backup failed during mongodump" "danger"
    exit 1
  fi
}

# Exécution avec gestion d'erreur
trap 'log "ERROR: Script terminated unexpectedly"; notify "failed" "❌ Backup script crashed" "danger"' ERR

main "$@"
```

## Limitations et Considérations

### Limitations Connues

```yaml
mongodump:
  - Ne capture pas les index en cours de construction
  - Performances dégradées sur très gros volumes (>10TB)
  - Pas de compression différentielle native
  - Impact possible sur le secondary pendant l'export
  - Ne sauvegarde pas certains types de collections spéciales

mongorestore:
  - Les index sont reconstruits (peut être long)
  - Pas de restauration online (nécessite arrêt ou mode maintenance)
  - Ordre d'insertion peut différer de l'original
  - Validation de schéma peut ralentir la restauration
```

### Alternatives à Considérer

```yaml
Pour volumes > 1TB:
  - Snapshots filesystem (LVM, EBS, etc.)
  - MongoDB Ops Manager Continuous Backup
  - MongoDB Atlas Backup (cloud)

Pour besoins spécifiques:
  - Change Streams pour backup continu
  - Oplog tailing pour réplication custom
  - Export CSV/JSON pour analytics
```

## Checklist de Production

```markdown
Avant chaque backup:
- [ ] Vérifier connectivité MongoDB
- [ ] Vérifier espace disque disponible
- [ ] Vérifier que le secondary est à jour
- [ ] Confirmer les fenêtres de maintenance
- [ ] Vérifier les alertes de monitoring

Pendant le backup:
- [ ] Monitorer l'utilisation CPU/RAM/IO
- [ ] Vérifier le replication lag
- [ ] Surveiller la taille du backup en cours

Après le backup:
- [ ] Vérifier l'intégrité (checksum)
- [ ] Valider la taille (cohérence historique)
- [ ] Uploader vers stockage distant
- [ ] Confirmer la réplication S3
- [ ] Logger les métadonnées
- [ ] Tester la restauration (périodiquement)
```

## Conclusion

`mongodump` et `mongorestore` restent des outils essentiels pour :
- Backups logiques portables
- Migrations entre environnements
- Exports sélectifs de données
- Disaster recovery

Pour les environnements de production critiques, ils doivent être combinés avec d'autres stratégies (snapshots, réplication, archivage) dans une approche de défense en profondeur.

---


⏭️ [Sauvegarde de Replica Sets](/12-sauvegarde-restauration/03-sauvegarde-replica-sets.md)
