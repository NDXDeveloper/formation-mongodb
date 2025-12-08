🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.4 Sauvegarde de Clusters Shardés

## Introduction

Les clusters shardés (sharded clusters) constituent l'architecture la plus complexe de MongoDB, distribuant les données horizontalement sur plusieurs shards pour supporter des volumes massifs et des charges de travail élevées. Cette distribution introduit des défis significatifs pour les opérations de sauvegarde et de restauration, nécessitant une coordination précise entre tous les composants du cluster.

Cette section détaille les stratégies, procédures et bonnes pratiques pour sauvegarder et restaurer efficacement un cluster shardé tout en garantissant la cohérence des données et la continuité d'activité.

## Architecture et Composants

### Topologie d'un Cluster Shardé

```
┌────────────────────────────────────────────────────────────┐
│                   Sharded Cluster Architecture             │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Application Servers                     │  │
│  └────────────────────┬─────────────────────────────────┘  │
│                       │                                    │
│                       v                                    │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 mongos (Query Routers)               │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │  │
│  │  │ mongos-1 │  │ mongos-2 │  │ mongos-3 │            │  │
│  │  └──────────┘  └──────────┘  └──────────┘            │  │
│  └────┬──────────────────┬──────────────────┬───────────┘  │
│       │                  │                  │              │
│       v                  v                  v              │
│  ┌─────────────────────────────────────────────────┐       │
│  │         Config Servers (Replica Set)            │       │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐       │       │
│  │  │ config-1 │  │ config-2 │  │ config-3 │       │       │
│  │  │ PRIMARY  │  │SECONDARY │  │SECONDARY │       │       │
│  │  └──────────┘  └──────────┘  └──────────┘       │       │
│  │  • Metadata du cluster                          │       │
│  │  • Configuration du sharding                    │       │
│  │  • Mapping chunks → shards                      │       │
│  └─────────────────────────────────────────────────┘       │
│       │                  │                  │              │
│       v                  v                  v              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Shard 0    │  │   Shard 1    │  │   Shard 2    │      │
│  │ (Replica Set)│  │ (Replica Set)│  │ (Replica Set)│      │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │      │
│  │ │ Primary  │ │  │ │ Primary  │ │  │ │ Primary  │ │      │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │      │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │      │
│  │ │Secondary │ │  │ │Secondary │ │  │ │Secondary │ │      │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │      │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │      │
│  │ │Secondary │ │  │ │Secondary │ │  │ │Secondary │ │      │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │      │
│  │              │  │              │  │              │      │
│  │ Data Range:  │  │ Data Range:  │  │ Data Range:  │      │
│  │ {min: A}     │  │ {min: M}     │  │ {min: T}     │      │
│  │ {max: L}     │  │ {max: S}     │  │ {max: Z}     │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────────────────────────────────────────┘
```

### Composants Critiques pour les Backups

```yaml
Config Servers (CRITIQUE):
  rôle: "Métadonnées du cluster"
  contenu:
    - Configuration du sharding
    - Mapping chunks vers shards
    - Définition des zones
    - Historique des migrations
    - State du balancer
  importance: "Sans config servers, impossible de router les requêtes"
  backup_priority: "MAXIMUM"

Shards (Données):
  rôle: "Stockage des données partitionnées"
  contenu:
    - Données applicatives (range-based)
    - Index locaux
    - Collections non-shardées (selon routing)
  backup_priority: "HAUTE"

mongos (Routeurs):
  rôle: "Routage des requêtes"
  contenu:
    - Configuration en mémoire (reloadée depuis config)
    - Pas de données persistantes
  backup_priority: "AUCUNE (stateless)"
```

## Défis Spécifiques aux Clusters Shardés

### Problèmes de Cohérence

```javascript
// Problème: Chunks en migration pendant le backup
{
  "scenario": "Backup pendant migration de chunk",
  "problème": {
    "chunk_id": "users_chunk_45",
    "source_shard": "shard0",
    "target_shard": "shard1",
    "état_backup": {
      "shard0": "backup à T0 (chunk présent)",
      "shard1": "backup à T0+30min (chunk présent aussi)",
      "résultat": "DUPLICATION de données"
    }
  },
  "solution": "Arrêter le balancer avant backup"
}

// Problème: Incohérence temporelle entre shards
{
  "scenario": "Backups non-synchronisés",
  "problème": {
    "shard0": "backup à 02:00:00",
    "shard1": "backup à 02:15:00",
    "shard2": "backup à 02:30:00",
    "résultat": "État du cluster à différents moments"
  },
  "impact": "Possible perte de cohérence référentielle",
  "solution": "Backup coordonné ou snapshots simultanés"
}
```

### Impact du Balancer

Le balancer MongoDB redistribue automatiquement les chunks entre shards :

```javascript
// État du balancer
db.adminCommand({ balancerStatus: 1 })

{
  "mode": "full",                    // Actif
  "inBalancerRound": false,          // Pas de round en cours
  "numBalancerRounds": 1247,
  "ok": 1
}

// Pendant le backup, le balancer DOIT être arrêté pour éviter:
// 1. Migration de chunks (duplication/perte de données)
// 2. Incohérence des backups
// 3. Impact performance sur les shards
```

## Stratégies de Sauvegarde

### Stratégie 1 : Backup Coordonné avec Balancer Arrêté (Recommandé)

C'est l'approche standard pour garantir la cohérence :

```bash
#!/bin/bash
# coordinated_sharded_backup.sh

set -euo pipefail

# Configuration
MONGOS_HOST="mongos.example.com:27017"
CONFIG_RS="configReplSet/config1:27019,config2:27019,config3:27019"
BACKUP_ROOT="/backup/sharded"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="${BACKUP_ROOT}/${TIMESTAMP}"

# Credentials
MONGO_USER="backup-admin"
MONGO_PASS="SecureBackupPassword"
AUTH_DB="admin"

# Logging
LOG_FILE="/var/log/sharded_backup.log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# ============================================================================
# PHASE 1: PRÉPARATION
# ============================================================================

check_prerequisites() {
  log "=== Phase 1: Prerequisites Check ==="

  # Vérifier connexion au mongos
  if ! mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/${AUTH_DB}" \
    --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log "ERROR: Cannot connect to mongos"
    exit 1
  fi

  # Vérifier espace disque
  local required_space=$((500 * 1024 * 1024 * 1024))  # 500GB minimum
  local available_space=$(df --output=avail -B1 "$BACKUP_ROOT" | tail -1)

  if [ "$available_space" -lt "$required_space" ]; then
    log "ERROR: Insufficient disk space"
    log "  Required: $(numfmt --to=iec-i --suffix=B $required_space)"
    log "  Available: $(numfmt --to=iec-i --suffix=B $available_space)"
    exit 1
  fi

  log "✓ Prerequisites OK"
}

# ============================================================================
# PHASE 2: ARRÊT DU BALANCER
# ============================================================================

stop_balancer() {
  log "=== Phase 2: Stopping Balancer ==="

  # Arrêter le balancer
  mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/${AUTH_DB}" --eval "
    sh.stopBalancer();
    print('Balancer stop requested');
  "

  # Attendre que toutes les migrations soient terminées
  log "Waiting for active migrations to complete..."
  local max_wait=600  # 10 minutes max
  local elapsed=0

  while [ $elapsed -lt $max_wait ]; do
    local is_running=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/${AUTH_DB}" --quiet --eval "
      sh.isBalancerRunning();
    ")

    if [ "$is_running" == "false" ]; then
      log "✓ Balancer stopped and no active migrations"
      break
    fi

    log "  Waiting for balancer to stop... (${elapsed}s elapsed)"
    sleep 5
    elapsed=$((elapsed + 5))
  done

  if [ $elapsed -ge $max_wait ]; then
    log "ERROR: Balancer did not stop in time"
    exit 1
  fi

  # Vérifier qu'aucun chunk n'est en migration
  local active_migrations=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/config" --quiet --eval "
    db.locks.find({state: {\\$ne: 0}}).count();
  ")

  if [ "$active_migrations" -gt 0 ]; then
    log "WARNING: $active_migrations active locks found"
  fi
}

# ============================================================================
# PHASE 3: BACKUP DES CONFIG SERVERS
# ============================================================================

backup_config_servers() {
  log "=== Phase 3: Backing up Config Servers ==="

  local config_backup_dir="${BACKUP_DIR}/config_servers"
  mkdir -p "$config_backup_dir"

  log "Starting config servers backup..."
  local start_time=$(date +%s)

  if mongodump \
    --uri="mongodb://${MONGO_USER}:${MONGO_PASS}@${CONFIG_RS}/${AUTH_DB}" \
    --oplog \
    --gzip \
    --out="$config_backup_dir"; then

    local end_time=$(date +%s)
    local duration=$((end_time - start_time))

    log "✓ Config servers backup completed in ${duration}s"

    # Sauvegarder la configuration complète
    log "Saving cluster configuration..."
    mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/${AUTH_DB}" \
      > "${config_backup_dir}/cluster_config.json" <<'EOF'
      printjson({
        shardingVersion: db.adminCommand({ getShardMap: 1 }),
        shards: db.getSiblingDB('config').shards.find().toArray(),
        databases: db.getSiblingDB('config').databases.find().toArray(),
        collections: db.getSiblingDB('config').collections.find().toArray(),
        chunks: {
          count: db.getSiblingDB('config').chunks.count(),
          sample: db.getSiblingDB('config').chunks.find().limit(10).toArray()
        },
        settings: db.getSiblingDB('config').settings.find().toArray(),
        version: db.getSiblingDB('config').version.find().toArray(),
        mongos: db.getSiblingDB('config').mongos.find().toArray(),
        timestamp: new Date()
      });
EOF

  else
    log "ERROR: Config servers backup failed"
    restart_balancer  # Redémarrer le balancer en cas d'erreur
    exit 1
  fi
}

# ============================================================================
# PHASE 4: BACKUP DE CHAQUE SHARD
# ============================================================================

backup_all_shards() {
  log "=== Phase 4: Backing up All Shards ==="

  # Obtenir la liste des shards
  local shards_json=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/config" --quiet --eval "
    db.shards.find({}, {_id:1, host:1}).toArray();
  " | tail -1)

  # Extraire les shards
  local shard_count=$(echo "$shards_json" | jq '. | length')
  log "Found $shard_count shards to backup"

  # Backup de chaque shard en parallèle
  local pids=()

  for i in $(seq 0 $((shard_count - 1))); do
    local shard_id=$(echo "$shards_json" | jq -r ".[$i]._id")
    local shard_host=$(echo "$shards_json" | jq -r ".[$i].host")

    log "Starting backup of shard: $shard_id ($shard_host)"

    # Lancer le backup en arrière-plan
    (
      local shard_backup_dir="${BACKUP_DIR}/shard_${shard_id}"
      mkdir -p "$shard_backup_dir"

      local shard_start=$(date +%s)

      if mongodump \
        --uri="mongodb://${MONGO_USER}:${MONGO_PASS}@${shard_host}/${AUTH_DB}" \
        --oplog \
        --gzip \
        --numParallelCollections=8 \
        --out="$shard_backup_dir" \
        2>&1 | tee "${shard_backup_dir}/backup.log"; then

        local shard_end=$(date +%s)
        local shard_duration=$((shard_end - shard_start))

        echo "✓ Shard $shard_id backup completed in ${shard_duration}s" >> "$LOG_FILE"

        # Sauvegarder les métadonnées du shard
        mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${shard_host}/${AUTH_DB}" \
          > "${shard_backup_dir}/shard_metadata.json" <<'EOF'
          printjson({
            replSetStatus: rs.status(),
            replSetConfig: rs.conf(),
            serverStatus: db.serverStatus(),
            dbStats: db.adminCommand({ listDatabases: 1 }),
            timestamp: new Date()
          });
EOF

      else
        echo "ERROR: Shard $shard_id backup failed" >> "$LOG_FILE"
        exit 1
      fi
    ) &

    pids+=($!)
  done

  # Attendre tous les backups
  log "Waiting for all shard backups to complete..."
  local failed=0

  for pid in "${pids[@]}"; do
    if ! wait "$pid"; then
      failed=$((failed + 1))
    fi
  done

  if [ $failed -gt 0 ]; then
    log "ERROR: $failed shard backup(s) failed"
    restart_balancer
    exit 1
  fi

  log "✓ All shards backed up successfully"
}

# ============================================================================
# PHASE 5: CONSOLIDATION ET MÉTADONNÉES
# ============================================================================

consolidate_backup() {
  log "=== Phase 5: Consolidation ==="

  # Créer un manifeste du backup
  cat > "${BACKUP_DIR}/MANIFEST.json" <<EOF
{
  "backup_type": "sharded_cluster",
  "timestamp": "$(date -Iseconds)",
  "cluster_name": "$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/${AUTH_DB}" --quiet --eval "db.serverStatus().repl.setName || 'standalone'")",
  "mongodb_version": "$(mongo --quiet --eval 'db.version()')",
  "components": {
    "config_servers": "config_servers/",
    "shards": $(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/config" --quiet --eval "
      db.shards.find({}, {_id:1}).toArray();
    " | tail -1 | jq '[.[].id]'),
    "balancer_state": "stopped_for_backup"
  },
  "backup_size_bytes": $(du -sb "$BACKUP_DIR" | cut -f1),
  "backup_duration_seconds": $SECONDS
}
EOF

  # Créer une archive (optionnel pour très gros volumes)
  if [ "${CREATE_ARCHIVE:-true}" == "true" ] && [ $(du -sb "$BACKUP_DIR" | cut -f1) -lt $((100 * 1024 * 1024 * 1024)) ]; then
    log "Creating consolidated archive..."
    tar -czf "${BACKUP_DIR}.tar.gz" -C "$BACKUP_ROOT" "$(basename $BACKUP_DIR)"

    # Checksum
    sha256sum "${BACKUP_DIR}.tar.gz" > "${BACKUP_DIR}.tar.gz.sha256"

    log "Archive created: ${BACKUP_DIR}.tar.gz"
    log "  Size: $(du -h ${BACKUP_DIR}.tar.gz | cut -f1)"
  else
    log "Skipping archive creation (backup too large or disabled)"
  fi
}

# ============================================================================
# PHASE 6: REDÉMARRAGE DU BALANCER
# ============================================================================

restart_balancer() {
  log "=== Phase 6: Restarting Balancer ==="

  mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/${AUTH_DB}" --eval "
    sh.startBalancer();
    print('Balancer restart requested');
  "

  # Vérifier que le balancer a redémarré
  sleep 5
  local balancer_mode=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/${AUTH_DB}" --quiet --eval "
    db.getSiblingDB('config').settings.findOne({_id: 'balancer'}).mode;
  ")

  if [ "$balancer_mode" == "full" ]; then
    log "✓ Balancer restarted successfully"
  else
    log "WARNING: Balancer mode is $balancer_mode (expected: full)"
  fi
}

# ============================================================================
# PHASE 7: CLEANUP
# ============================================================================

cleanup_old_backups() {
  log "=== Phase 7: Cleanup ==="

  local retention_days=${RETENTION_DAYS:-30}

  log "Cleaning up backups older than $retention_days days..."
  find "$BACKUP_ROOT" -maxdepth 1 -type d -name "20*" -mtime +$retention_days -exec rm -rf {} \;
  find "$BACKUP_ROOT" -name "*.tar.gz" -mtime +$retention_days -delete

  log "✓ Cleanup completed"
}

# ============================================================================
# MAIN EXECUTION
# ============================================================================

main() {
  local overall_start=$(date +%s)

  log "========================================================"
  log "    MongoDB Sharded Cluster Backup"
  log "========================================================"
  log "Timestamp: $(date -Iseconds)"
  log "Backup Directory: $BACKUP_DIR"
  log ""

  # Créer le répertoire de backup
  mkdir -p "$BACKUP_DIR"

  # Exécution des phases
  check_prerequisites
  stop_balancer
  backup_config_servers
  backup_all_shards
  consolidate_backup
  restart_balancer
  cleanup_old_backups

  local overall_end=$(date +%s)
  local total_duration=$((overall_end - overall_start))

  log ""
  log "========================================================"
  log "✓ SHARDED CLUSTER BACKUP COMPLETED SUCCESSFULLY"
  log "========================================================"
  log "Total Duration: ${total_duration}s ($(date -u -d @${total_duration} +%H:%M:%S))"
  log "Backup Location: $BACKUP_DIR"
  log "Backup Size: $(du -sh $BACKUP_DIR | cut -f1)"
  log ""
  log "Next Steps:"
  log "  1. Upload to remote storage"
  log "  2. Verify backup integrity"
  log "  3. Test restoration procedure"
  log "========================================================"
}

# Gestion des erreurs
trap 'log "ERROR: Script failed at line $LINENO"; restart_balancer; exit 1' ERR

# Gestion des interruptions
trap 'log "Script interrupted"; restart_balancer; exit 130' INT TERM

main "$@"
```

### Stratégie 2 : Snapshots Simultanés (Pour Très Gros Volumes)

Pour clusters > 10TB, les snapshots filesystem sont préférables :

```bash
#!/bin/bash
# sharded_cluster_snapshots.sh

set -euo pipefail

MONGOS_HOST="mongos:27017"
CONFIG_SERVERS=("config1" "config2" "config3")
SHARDS=("shard0-primary" "shard1-primary" "shard2-primary")

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Arrêter le balancer
stop_balancer() {
  log "Stopping balancer..."
  mongo "mongodb://$MONGOS_HOST/admin" --eval "sh.stopBalancer()"

  # Attendre arrêt complet
  while true; do
    if [ "$(mongo mongodb://$MONGOS_HOST/admin --quiet --eval 'sh.isBalancerRunning()')" == "false" ]; then
      break
    fi
    sleep 2
  done

  log "✓ Balancer stopped"
}

# Créer snapshots LVM sur tous les serveurs
create_all_snapshots() {
  log "Creating snapshots on all nodes..."

  local snapshot_timestamp=$(date +%Y%m%d_%H%M%S)
  local pids=()

  # Config servers
  for config in "${CONFIG_SERVERS[@]}"; do
    (
      ssh "$config" "
        lvcreate --size 50G --snapshot --name snap_config_$snapshot_timestamp /dev/vg_mongodb/lv_data
      "
      log "  Snapshot created on config server: $config"
    ) &
    pids+=($!)
  done

  # Shards
  for shard in "${SHARDS[@]}"; do
    (
      ssh "$shard" "
        lvcreate --size 200G --snapshot --name snap_shard_$snapshot_timestamp /dev/vg_mongodb/lv_data
      "
      log "  Snapshot created on shard: $shard"
    ) &
    pids+=($!)
  done

  # Attendre tous les snapshots
  for pid in "${pids[@]}"; do
    wait "$pid"
  done

  log "✓ All snapshots created"
  echo "$snapshot_timestamp"
}

# Copier les snapshots vers stockage
copy_snapshots() {
  local snapshot_name=$1
  local backup_dest="/backup/sharded/$snapshot_name"

  log "Copying snapshots to backup storage..."

  mkdir -p "$backup_dest"

  # Copier depuis chaque serveur
  for config in "${CONFIG_SERVERS[@]}"; do
    ssh "$config" "
      mount -o ro /dev/vg_mongodb/snap_config_$snapshot_name /mnt/snapshot
      rsync -av /mnt/snapshot/ $backup_dest/config_$config/
      umount /mnt/snapshot
      lvremove -f /dev/vg_mongodb/snap_config_$snapshot_name
    " &
  done

  for shard in "${SHARDS[@]}"; do
    ssh "$shard" "
      mount -o ro /dev/vg_mongodb/snap_shard_$snapshot_name /mnt/snapshot
      rsync -av /mnt/snapshot/ $backup_dest/shard_$shard/
      umount /mnt/snapshot
      lvremove -f /dev/vg_mongodb/snap_shard_$snapshot_name
    " &
  done

  wait

  log "✓ All snapshots copied and cleaned up"
}

# Main
main() {
  log "=== Sharded Cluster Snapshot Backup ==="

  stop_balancer

  local snapshot_name=$(create_all_snapshots)

  # Redémarrer le balancer immédiatement
  mongo "mongodb://$MONGOS_HOST/admin" --eval "sh.startBalancer()"
  log "Balancer restarted (snapshot creation complete)"

  # Copier en arrière-plan
  copy_snapshots "$snapshot_name"

  log "✓ Snapshot backup completed: $snapshot_name"
}

main "$@"
```

## Restauration d'un Cluster Shardé

### Restauration Complète

```bash
#!/bin/bash
# restore_sharded_cluster.sh

set -euo pipefail

BACKUP_DIR="/backup/sharded/20241208_020000"
MONGOS_HOST="mongos:27017"
MONGO_USER="admin"
MONGO_PASS="SecurePass"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  MongoDB Sharded Cluster Restoration"
log "========================================================"
log "Source: $BACKUP_DIR"
log ""

# Vérifier que le backup existe
if [ ! -d "$BACKUP_DIR" ]; then
  log "ERROR: Backup directory not found: $BACKUP_DIR"
  exit 1
fi

# Charger le manifeste
if [ ! -f "$BACKUP_DIR/MANIFEST.json" ]; then
  log "ERROR: MANIFEST.json not found in backup"
  exit 1
fi

log "Backup manifest:"
jq . "$BACKUP_DIR/MANIFEST.json"
log ""

read -p "⚠️  This will DESTROY current cluster data. Continue? (type 'yes'): " confirm
if [ "$confirm" != "yes" ]; then
  log "Restoration cancelled"
  exit 0
fi

# ============================================================================
# PHASE 1: ARRÊTER LE CLUSTER
# ============================================================================

log "=== Phase 1: Stopping Cluster ==="

# Arrêter le balancer
log "Stopping balancer..."
mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/admin" \
  --eval "sh.stopBalancer()" || true

# Arrêter tous les mongos
log "Stopping mongos instances..."
# [Commandes spécifiques à votre environnement]

# Arrêter tous les shards
log "Stopping all shards..."
# [Commandes spécifiques à votre environnement]

# Arrêter les config servers
log "Stopping config servers..."
# [Commandes spécifiques à votre environnement]

sleep 10

# ============================================================================
# PHASE 2: RESTAURER LES CONFIG SERVERS
# ============================================================================

log "=== Phase 2: Restoring Config Servers ==="

CONFIG_SERVERS=("config1:27019" "config2:27019" "config3:27019")

for config_server in "${CONFIG_SERVERS[@]}"; do
  host=$(echo "$config_server" | cut -d: -f1)
  port=$(echo "$config_server" | cut -d: -f2)

  log "Restoring config server: $host:$port"

  # Nettoyer le datadir
  ssh "$host" "rm -rf /var/lib/mongodb/*"

  # Restaurer depuis le backup
  mongorestore \
    --host="$host:$port" \
    --drop \
    --oplogReplay \
    --gzip \
    --dir="$BACKUP_DIR/config_servers" &
done

wait

log "✓ Config servers restored"

# Démarrer les config servers
log "Starting config servers..."
for config_server in "${CONFIG_SERVERS[@]}"; do
  host=$(echo "$config_server" | cut -d: -f1)
  ssh "$host" "systemctl start mongod"
done

# Attendre que le replica set soit prêt
log "Waiting for config replica set to be ready..."
sleep 30

CONFIG_RS="configReplSet/${CONFIG_SERVERS[0]},${CONFIG_SERVERS[1]},${CONFIG_SERVERS[2]}"

for i in {1..30}; do
  if mongo "mongodb://$CONFIG_RS/admin" --eval "rs.status()" >/dev/null 2>&1; then
    log "✓ Config servers replica set is ready"
    break
  fi
  sleep 5
done

# ============================================================================
# PHASE 3: RESTAURER CHAQUE SHARD
# ============================================================================

log "=== Phase 3: Restoring Shards ==="

# Extraire la liste des shards depuis le manifeste
SHARD_IDS=$(jq -r '.components.shards[]' "$BACKUP_DIR/MANIFEST.json")

for shard_id in $SHARD_IDS; do
  log "Restoring shard: $shard_id"

  shard_backup_dir="$BACKUP_DIR/shard_${shard_id}"

  if [ ! -d "$shard_backup_dir" ]; then
    log "ERROR: Shard backup not found: $shard_backup_dir"
    continue
  fi

  # Obtenir les membres du shard depuis la config
  shard_host=$(mongo "mongodb://$CONFIG_RS/config" --quiet --eval "
    db.shards.findOne({_id: '$shard_id'}).host;
  ")

  log "  Shard host: $shard_host"

  # Extraire le primary
  primary_host=$(echo "$shard_host" | cut -d/ -f2 | cut -d, -f1)

  # Restaurer sur le primary
  mongorestore \
    --host="$primary_host" \
    --drop \
    --oplogReplay \
    --gzip \
    --numParallelCollections=8 \
    --dir="$shard_backup_dir" &
done

# Attendre toutes les restaurations
wait

log "✓ All shards restored"

# ============================================================================
# PHASE 4: REDÉMARRER LE CLUSTER
# ============================================================================

log "=== Phase 4: Restarting Cluster ==="

# Démarrer les mongos
log "Starting mongos instances..."
# [Commandes spécifiques]

sleep 10

# Vérifier la connectivité
log "Verifying cluster connectivity..."
for i in {1..30}; do
  if mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/admin" \
    --eval "sh.status()" >/dev/null 2>&1; then
    log "✓ Cluster is accessible"
    break
  fi
  sleep 5
done

# ============================================================================
# PHASE 5: VALIDATION
# ============================================================================

log "=== Phase 5: Validation ==="

# Vérifier le statut du sharding
log "Checking sharding status..."
mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/admin" <<'EOF'
  print("\n=== Cluster Status ===");
  printjson(sh.status());

  print("\n=== Databases ===");
  db.adminCommand({ listDatabases: 1 }).databases.forEach(function(db) {
    print(db.name + " - " + (db.sizeOnDisk / 1024 / 1024 / 1024).toFixed(2) + " GB");
  });

  print("\n=== Shards ===");
  db.getSiblingDB('config').shards.find().forEach(function(shard) {
    print(shard._id + " - " + shard.host);
  });

  print("\n=== Balancer ===");
  printjson(sh.getBalancerState());
EOF

# Redémarrer le balancer
log "Restarting balancer..."
mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGOS_HOST}/admin" \
  --eval "sh.startBalancer()"

log ""
log "========================================================"
log "✓ SHARDED CLUSTER RESTORATION COMPLETED"
log "========================================================"
log ""
log "Important Post-Restoration Tasks:"
log "  1. Verify data integrity (run sample queries)"
log "  2. Check balancer is running: sh.getBalancerState()"
log "  3. Monitor chunk distribution: sh.status()"
log "  4. Verify application connectivity"
log "  5. Check replication lag on all shards"
log "  6. Review logs for any errors"
log "========================================================"
```

### Restauration d'un Shard Individuel

```bash
#!/bin/bash
# restore_single_shard.sh

SHARD_ID="shard0"
BACKUP_DIR="/backup/sharded/20241208_020000/shard_${SHARD_ID}"
MONGOS_HOST="mongos:27017"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Restoring Single Shard: $SHARD_ID ==="

# Obtenir les informations du shard
shard_info=$(mongo "mongodb://$MONGOS_HOST/config" --quiet --eval "
  db.shards.findOne({_id: '$SHARD_ID'});
")

shard_host=$(echo "$shard_info" | jq -r '.host')
log "Shard host: $shard_host"

# Retirer temporairement le shard du cluster
log "Removing shard from cluster..."
mongo "mongodb://$MONGOS_HOST/admin" --eval "
  db.adminCommand({ removeShard: '$SHARD_ID' });
"

# Attendre que le draining soit complet
log "Waiting for shard draining..."
while true; do
  status=$(mongo "mongodb://$MONGOS_HOST/admin" --quiet --eval "
    db.adminCommand({ removeShard: '$SHARD_ID' }).state;
  ")

  if [ "$status" == "completed" ]; then
    break
  fi

  log "  Draining in progress..."
  sleep 30
done

log "✓ Shard drained and removed"

# Restaurer le shard
log "Restoring shard data..."
primary_host=$(echo "$shard_host" | cut -d/ -f2 | cut -d, -f1)

mongorestore \
  --host="$primary_host" \
  --drop \
  --oplogReplay \
  --gzip \
  --dir="$BACKUP_DIR"

log "✓ Shard data restored"

# Ré-ajouter le shard au cluster
log "Re-adding shard to cluster..."
mongo "mongodb://$MONGOS_HOST/admin" --eval "
  sh.addShard('$shard_host');
"

log "✓ Shard $SHARD_ID restored and re-added to cluster"
```

## Backup avec MongoDB Atlas

Pour les clusters shardés sur Atlas, la complexité est gérée automatiquement :

```bash
#!/bin/bash
# atlas_sharded_backup.sh

ATLAS_PUBLIC_KEY="your-public-key"
ATLAS_PRIVATE_KEY="your-private-key"
PROJECT_ID="your-project-id"
CLUSTER_NAME="sharded-prod-cluster"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Vérifier la configuration backup
check_backup_config() {
  log "Checking Atlas backup configuration..."

  curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/schedule" \
    | jq '{
      enabled: .policies != null,
      snapshot_frequency: .referenceHourOfDay,
      restore_window: .restoreWindowDays,
      policies: [.policies[] | {
        id: .id,
        items: [.policyItems[] | {
          frequency: .frequencyType,
          retention: .retentionValue
        }]
      }]
    }'
}

# Lister les snapshots disponibles
list_snapshots() {
  log "Listing available snapshots..."

  curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/snapshots" \
    | jq '.results[] | {
      id: .id,
      created: .createdAt,
      type: .type,
      status: .status,
      clusterInfo: .parts[0].typeName
    }'
}

# Atlas gère automatiquement:
# - Arrêt coordonné du balancer
# - Backup de tous les config servers
# - Backup de tous les shards en parallèle
# - Cohérence garantie point-in-time
# - Ré-activation du balancer

log "Atlas handles sharded cluster backups automatically:"
log "  ✓ Coordinated balancer stop/start"
log "  ✓ Config servers backup"
log "  ✓ All shards backup (parallel)"
log "  ✓ Point-in-time consistency"
log "  ✓ Cross-region copies (if configured)"

check_backup_config
list_snapshots
```

## Monitoring et Validation

### Script de Monitoring Complet

```bash
#!/bin/bash
# monitor_sharded_backup.sh

MONGOS_HOST="mongos:27017"
BACKUP_DIR="/backup/sharded"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Vérifier l'état du balancer
check_balancer() {
  echo "=== Balancer Status ==="

  mongo "mongodb://$MONGOS_HOST/admin" --quiet --eval "
    status = sh.getBalancerState();
    print('Balancer Enabled: ' + status);

    if (status) {
      running = sh.isBalancerRunning();
      print('Currently Running: ' + running);

      lockInfo = db.getSiblingDB('config').locks.findOne({_id: 'balancer'});
      if (lockInfo) {
        print('Lock State: ' + lockInfo.state);
      }
    }
  "
}

# Vérifier les migrations actives
check_active_migrations() {
  echo ""
  echo "=== Active Migrations ==="

  local active=$(mongo "mongodb://$MONGOS_HOST/config" --quiet --eval "
    db.locks.find({state: {\$ne: 0}}).count();
  ")

  echo "Active locks: $active"

  if [ "$active" -gt 0 ]; then
    mongo "mongodb://$MONGOS_HOST/config" --quiet --eval "
      db.locks.find({state: {\$ne: 0}}).forEach(printjson);
    "
  fi
}

# Vérifier les chunks
check_chunk_distribution() {
  echo ""
  echo "=== Chunk Distribution ==="

  mongo "mongodb://$MONGOS_HOST/config" --quiet --eval "
    db.chunks.aggregate([
      {
        \$group: {
          _id: '\$shard',
          count: { \$sum: 1 }
        }
      },
      {
        \$sort: { count: -1 }
      }
    ]).forEach(function(result) {
      print(result._id + ': ' + result.count + ' chunks');
    });
  "
}

# Vérifier le dernier backup
check_last_backup() {
  echo ""
  echo "=== Last Backup ==="

  if [ ! -d "$BACKUP_DIR" ]; then
    echo "Backup directory not found: $BACKUP_DIR"
    return 1
  fi

  local latest=$(find "$BACKUP_DIR" -maxdepth 1 -type d -name "20*" | sort -r | head -1)

  if [ -z "$latest" ]; then
    echo "No backups found"
    return 1
  fi

  local backup_date=$(basename "$latest")
  local backup_age_hours=$(( ($(date +%s) - $(date -d "$backup_date" +%s 2>/dev/null || echo 0)) / 3600 ))

  echo "Latest backup: $backup_date"
  echo "Age: ${backup_age_hours}h"

  if [ -f "$latest/MANIFEST.json" ]; then
    echo ""
    echo "Manifest:"
    jq '{backup_type, timestamp, backup_size_bytes, backup_duration_seconds}' "$latest/MANIFEST.json"
  fi

  # Alert si > 26h
  if [ $backup_age_hours -gt 26 ]; then
    echo "⚠️  WARNING: Backup is older than 26 hours"
    return 1
  fi

  echo "✓ Backup is recent"
}

# Vérifier la santé du cluster
check_cluster_health() {
  echo ""
  echo "=== Cluster Health ==="

  # Vérifier chaque shard
  mongo "mongodb://$MONGOS_HOST/config" --quiet --eval "
    db.shards.find().forEach(function(shard) {
      print('\nShard: ' + shard._id);
      print('  Host: ' + shard.host);

      try {
        // Se connecter au shard
        conn = new Mongo(shard.host.split('/')[1].split(',')[0]);
        db = conn.getDB('admin');

        // Vérifier le replica set
        status = db.adminCommand({ replSetGetStatus: 1 });

        status.members.forEach(function(member) {
          print('  Member: ' + member.name);
          print('    State: ' + member.stateStr);
          print('    Health: ' + member.health);

          if (member.stateStr === 'SECONDARY') {
            lag = (status.date - member.optimeDate) / 1000;
            print('    Lag: ' + lag.toFixed(2) + 's');
          }
        });
      } catch (e) {
        print('  ERROR: Cannot connect - ' + e);
      }
    });
  "
}

# Main
main() {
  log "=== Sharded Cluster Backup Monitoring ==="

  check_balancer
  check_active_migrations
  check_chunk_distribution
  check_last_backup
  check_cluster_health

  log ""
  log "✓ Monitoring completed"
}

main "$@"
```

## Cas Particuliers et Troubleshooting

### Problème 1 : Backup pendant une Migration

```bash
# Symptôme: Migration en cours pendant le backup
# Solution: Attendre ou forcer l'arrêt

force_stop_migration() {
  mongo "mongodb://$MONGOS_HOST/config" --eval "
    // Identifier la migration en cours
    migration = db.locks.findOne({state: 1});

    if (migration) {
      print('Active migration found:');
      printjson(migration);

      // Forcer le déblocage (ATTENTION: utiliser avec précaution)
      db.locks.update(
        {_id: migration._id},
        {\$set: {state: 0}}
      );

      print('Migration forcefully stopped');
    }
  "
}
```

### Problème 2 : Config Server Corruption

```bash
# Solution: Restaurer uniquement les config servers
restore_config_only() {
  log "Restoring config servers only..."

  # Arrêter le cluster
  mongo "mongodb://$MONGOS_HOST/admin" --eval "sh.stopBalancer()"

  # Restaurer config servers
  mongorestore \
    --uri="mongodb://config1:27019,config2:27019,config3:27019/?replicaSet=configReplSet" \
    --drop \
    --oplogReplay \
    --gzip \
    --nsInclude="config.*" \
    --dir="/backup/sharded/latest/config_servers"

  # Redémarrer
  mongo "mongodb://$MONGOS_HOST/admin" --eval "sh.startBalancer()"

  log "✓ Config servers restored, cluster metadata intact"
}
```

### Problème 3 : Shard Manquant Après Restauration

```bash
# Solution: Ré-ajouter le shard manuellement
readd_missing_shard() {
  local shard_id="shard3"
  local shard_host="shard3rs/shard3-a:27018,shard3-b:27018,shard3-c:27018"

  log "Re-adding missing shard: $shard_id"

  # Vérifier s'il existe déjà
  exists=$(mongo "mongodb://$MONGOS_HOST/config" --quiet --eval "
    db.shards.find({_id: '$shard_id'}).count();
  ")

  if [ "$exists" -eq 0 ]; then
    mongo "mongodb://$MONGOS_HOST/admin" --eval "
      sh.addShard('$shard_host');
      print('Shard added: $shard_id');
    "
  else
    log "Shard already exists in config"
  fi
}
```

## Automatisation et Orchestration

### Kubernetes CronJob pour Cluster Shardé

```yaml
# sharded-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-sharded-backup
  namespace: database
spec:
  schedule: "0 2 * * *"  # 2h du matin
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: mongodb-sharded-backup
        spec:
          restartPolicy: OnFailure
          serviceAccountName: mongodb-backup

          containers:
          - name: backup
            image: mongodb-backup-tools:latest
            command:
            - /bin/bash
            - -c
            - |
              #!/bin/bash
              set -euo pipefail

              # Script de backup intégré
              /scripts/coordinated_sharded_backup.sh

              # Upload vers S3
              aws s3 sync /backup/sharded s3://company-backups/mongodb/sharded/ \
                --storage-class STANDARD_IA \
                --exclude "*" \
                --include "$(date +%Y%m%d)*"

              echo "✓ Backup completed and uploaded to S3"

            env:
            - name: MONGOS_HOST
              value: "mongodb-mongos-svc:27017"
            - name: MONGO_USER
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-credentials
                  key: username
            - name: MONGO_PASS
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-credentials
                  key: password
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
            - name: backup-scripts
              mountPath: /scripts

            resources:
              requests:
                memory: "8Gi"
                cpu: "4"
              limits:
                memory: "16Gi"
                cpu: "8"

          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: mongodb-backup-pvc
          - name: backup-scripts
            configMap:
              name: mongodb-backup-scripts
              defaultMode: 0755
```

## Checklist de Production

```markdown
### Avant le Backup

- [ ] Vérifier connectivité au mongos
- [ ] Confirmer état de tous les shards (healthy)
- [ ] Vérifier replication lag < 5 min sur tous les shards
- [ ] Confirmer aucune migration de chunk en cours
- [ ] Vérifier fenêtre de maintenance appropriée
- [ ] Confirmer espace disque suffisant (tous les nœuds)
- [ ] Valider que le balancer peut être arrêté

### Pendant le Backup

- [ ] Balancer arrêté et confirmé
- [ ] Aucune migration active
- [ ] Monitorer progression sur chaque shard
- [ ] Vérifier replication lag reste acceptable
- [ ] Confirmer aucune erreur dans les logs
- [ ] Durée totale dans limites acceptables

### Après le Backup

- [ ] Balancer redémarré
- [ ] Vérifier intégrité config servers backup
- [ ] Vérifier intégrité backup de chaque shard
- [ ] Valider taille totale cohérente
- [ ] Checksums générés
- [ ] MANIFEST.json créé et valide
- [ ] Upload vers stockage distant réussi
- [ ] Métadonnées enregistrées
- [ ] Cleanup anciens backups effectué

### Validation Périodique

- [ ] Test de restauration des config servers (mensuel)
- [ ] Test de restauration d'un shard (mensuel)
- [ ] Test de restauration complète (trimestriel)
- [ ] DR drill complet (annuel)
- [ ] Documentation à jour
- [ ] Procédures testées et validées
```

## Comparaison des Stratégies

```
┌────────────────────┬──────────────┬──────────────┬──────────────┐
│    Critère         │ mongodump    │  Snapshots   │ Atlas Backup │
│                    │ Coordonné    │  Simultanés  │              │
├────────────────────┼──────────────┼──────────────┼──────────────┤
│ Complexité Setup   │ Moyenne      │ Élevée       │ Très faible  │
│ Temps Backup       │ 1-4 heures   │ < 2 minutes  │ < 1 heure    │
│ Impact Production  │ Moyen        │ Minimal      │ Minimal      │
│ Balancer Downtime  │ 1-4 heures   │ < 5 minutes  │ Auto-géré    │
│ Portabilité        │ ✓ Excellente │ ✗ OS-dépend  │ ✓ Bonne      │
│ Volume > 10TB      │ ✗ Lent       │ ✓ Optimal    │ ✓ Géré       │
│ Granularité        │ ✓ Fine       │ ✗ Globale    │ ✓ Fine       │
│ PITR               │ ✗ Limité     │ ✗ Non        │ ✓ Natif      │
│ Cross-Region       │ ✗ Manuel     │ ✗ Manuel     │ ✓ Auto       │
│ Coût Opérationnel  │ Élevé        │ Très élevé   │ Faible       │
│ Expertise Requise  │ Élevée       │ Expert       │ Faible       │
└────────────────────┴──────────────┴──────────────┴──────────────┘
```

## Conclusion

La sauvegarde de clusters shardés représente le défi le plus complexe en matière de backup MongoDB, nécessitant :

**Éléments critiques** :
1. **Coordination** - Arrêt balancer, backups synchronisés
2. **Config servers** - Priorité absolue (métadonnées)
3. **Cohérence** - État consistant entre tous les shards
4. **Automatisation** - Scripts robustes essentiels
5. **Tests** - Validation régulière de la procédure

**Recommandations par taille** :
- **< 1TB** : mongodump coordonné (simple, fiable)
- **1-10TB** : Combinaison mongodump + snapshots
- **> 10TB** : Snapshots simultanés ou Atlas Backup
- **Cloud** : MongoDB Atlas (complexité gérée)

**Points clés à retenir** :
- Toujours arrêter le balancer
- Config servers = composant le plus critique
- Tester restauration régulièrement
- Documenter la topologie complète
- Prévoir fenêtre de maintenance adéquate

Une stratégie de backup robuste pour clusters shardés est essentielle pour toute infrastructure MongoDB à grande échelle, et investir dans l'automatisation et les tests réguliers est crucial pour garantir la recouvrabilité.

---


⏭️ [Snapshots du système de fichiers](/12-sauvegarde-restauration/05-snapshots-systeme-fichiers.md)
