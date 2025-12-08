🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.5 Snapshots du Système de Fichiers

## Introduction

Les snapshots du système de fichiers représentent une méthode de sauvegarde physique qui capture l'état complet du stockage MongoDB à un instant T. Contrairement aux backups logiques (mongodump), les snapshots opèrent au niveau du système de fichiers, offrant des avantages significatifs en termes de vitesse, d'impact minimal sur les performances et de fiabilité pour les très gros volumes de données.

Cette approche est particulièrement adaptée aux environnements de production à grande échelle où les backups logiques deviendraient trop lents ou trop coûteux en ressources.

## Concepts Fondamentaux

### Qu'est-ce qu'un Snapshot ?

Un snapshot est une **image figée** du système de fichiers à un moment précis, créée instantanément sans copier physiquement les données :

```
┌───────────────────────────────────────────────────────┐
│              Copy-on-Write (CoW) Snapshot             │
│                                                       │
│  État Initial (T0)          Snapshot (T0)             │
│  ┌──────────────┐           ┌──────────────┐          │
│  │ Block 1: A   │           │ Block 1: A   │          │
│  │ Block 2: B   │────snap───│ Block 2: B   │          │
│  │ Block 3: C   │           │ Block 3: C   │          │
│  └──────────────┘           └──────────────┘          │
│         │                         │                   │
│         │ Écriture (T1)           │                   │
│         v                         │                   │
│  ┌──────────────┐                 │                   │
│  │ Block 1: X   │←────nouveau     │                   │
│  │ Block 2: B   │                 │                   │
│  │ Block 3: C   │                 │                   │
│  └──────────────┘                 │                   │
│    (Production)        (Snapshot inchangé)            │
│                                                       │
│  Le snapshot conserve les blocs originaux             │
│  Les nouvelles écritures créent de nouveaux blocs     │
└───────────────────────────────────────────────────────┘
```

### Types de Snapshots

```yaml
LVM (Logical Volume Manager):
  plateforme: Linux
  performance: Rapide (CoW)
  overhead: 10-20% espace disque
  use_case: Serveurs on-premise

ZFS:
  plateforme: Linux, FreeBSD
  performance: Très rapide (CoW natif)
  overhead: Minimal (compression intégrée)
  use_case: Stockage haute performance

Btrfs:
  plateforme: Linux
  performance: Rapide (CoW)
  overhead: Minimal
  use_case: Systèmes modernes

AWS EBS Snapshots:
  plateforme: AWS
  performance: Incrémental automatique
  overhead: Stockage S3 optimisé
  use_case: Déploiements AWS

Azure Managed Disks:
  plateforme: Azure
  performance: Incrémental
  overhead: Minimal
  use_case: Déploiements Azure

GCP Persistent Disk Snapshots:
  plateforme: Google Cloud
  performance: Incrémental automatique
  overhead: Optimisé
  use_case: Déploiements GCP
```

### Avantages et Inconvénients

**Avantages des Snapshots** :
```yaml
Performance:
  - Création quasi-instantanée (< 1 seconde)
  - Impact minimal sur MongoDB (pause brève)
  - Pas de charge CPU/réseau continue
  - Adapté aux très gros volumes (> 1TB)

Fiabilité:
  - Capture exacte de l'état disque
  - Cohérence crash-consistent garantie
  - Inclut tous les fichiers (data, index, config)
  - Pas de risque d'export incomplet

Opérationnel:
  - Restauration rapide
  - Testing facile (mount readonly)
  - Clonage d'environnements
  - Space-efficient (CoW)
```

**Inconvénients** :
```yaml
Portabilité:
  - Dépendant de l'OS/plateforme
  - Pas de cross-platform (Linux → Windows)
  - Requiert même version MongoDB

Granularité:
  - Tout ou rien (pas de sélection)
  - Pas de filtrage par database/collection
  - Restauration complète nécessaire

Complexité:
  - Configuration initiale plus complexe
  - Nécessite privilèges système
  - Gestion du stockage sous-jacent
  - Coordination pour cohérence
```

## Garantie de Cohérence avec MongoDB

### Cohérence Crash-Consistent

MongoDB est conçu pour être **crash-consistent** grâce à son journal (WiredTiger) :

```javascript
// WiredTiger garantit la cohérence après crash
{
  "journal": {
    "enabled": true,
    "commitIntervalMs": 100,  // Flush toutes les 100ms
    "path": "/var/lib/mongodb/journal"
  },

  // Lors d'un snapshot:
  "snapshot_guarantees": {
    "consistency": "crash-consistent",
    "recovery": "automatic via journal replay",
    "data_loss": "max 100ms d'écritures (si journal sync)"
  }
}
```

### Procédure de Cohérence Stricte

Pour garantir une cohérence absolue, utiliser `fsyncLock()` :

```javascript
// 1. Verrou d'écriture (bloque les écritures)
db.fsyncLock()
// MongoDB flush toutes les écritures en mémoire vers le disque
// et bloque toutes les nouvelles écritures

// 2. Créer le snapshot (< 1 seconde)
// [Commande système de snapshot]

// 3. Débloquer immédiatement
db.fsyncUnlock()
// MongoDB reprend les écritures normalement

// Impact: 1-2 secondes de pause d'écriture
// Acceptable pour la plupart des cas
```

### Cohérence sans Verrou (Recommandé)

Pour MongoDB 3.6+, WiredTiger assure la cohérence sans `fsyncLock()` :

```bash
# Snapshot direct sans pause
# MongoDB récupère automatiquement via journal replay
lvcreate --size 10G --snapshot --name mongo-snap /dev/vg0/mongo-lv

# Avantages:
# - Aucune pause de service
# - Snapshot instantané
# - Cohérence garantie par WiredTiger

# La récupération après restauration:
# 1. MongoDB détecte que le shutdown était non-clean
# 2. Replay automatique du journal
# 3. Retour à un état cohérent
```

## Snapshots avec LVM (Linux)

### Configuration Initiale LVM

```bash
#!/bin/bash
# setup_lvm_for_mongodb.sh

# 1. Créer un Physical Volume
pvcreate /dev/sdb

# 2. Créer un Volume Group
vgcreate vg_mongodb /dev/sdb

# 3. Créer un Logical Volume (80% de l'espace, 20% pour snapshots)
lvcreate -L 800G -n lv_mongodb_data vg_mongodb

# 4. Formater avec XFS (recommandé pour MongoDB)
mkfs.xfs -f /dev/vg_mongodb/lv_mongodb_data

# 5. Monter
mkdir -p /var/lib/mongodb
mount /dev/vg_mongodb/lv_mongodb_data /var/lib/mongodb

# 6. Configurer le montage automatique
echo "/dev/vg_mongodb/lv_mongodb_data /var/lib/mongodb xfs defaults,noatime 0 0" >> /etc/fstab

# 7. Ajuster les permissions
chown -R mongodb:mongodb /var/lib/mongodb

echo "✓ LVM configured for MongoDB"
```

### Script de Snapshot LVM Complet

```bash
#!/bin/bash
# lvm_snapshot_backup.sh

set -euo pipefail

# Configuration
VG_NAME="vg_mongodb"
LV_NAME="lv_mongodb_data"
SNAPSHOT_SIZE="200G"  # 20% du volume pour CoW
MOUNT_POINT="/var/lib/mongodb"
SNAPSHOT_MOUNT="/mnt/mongodb_snapshot"
BACKUP_DEST="/backup/mongodb/lvm_snapshots"
MONGO_HOST="localhost:27017"
MONGO_USER="admin"
MONGO_PASS="SecurePass"

# Logging
LOG_FILE="/var/log/mongodb_lvm_backup.log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Vérifier les prérequis
check_prerequisites() {
  log "Checking prerequisites..."

  # Vérifier que le LV existe
  if ! lvdisplay "/dev/$VG_NAME/$LV_NAME" >/dev/null 2>&1; then
    log "ERROR: Logical Volume not found"
    exit 1
  fi

  # Vérifier l'espace disponible dans le VG
  VG_FREE=$(vgs --noheadings --units g --nosuffix -o vg_free "$VG_NAME" | tr -d ' ')
  SNAPSHOT_SIZE_NUM=$(echo "$SNAPSHOT_SIZE" | sed 's/G//')

  if (( $(echo "$VG_FREE < $SNAPSHOT_SIZE_NUM" | bc -l) )); then
    log "ERROR: Insufficient free space in VG (need ${SNAPSHOT_SIZE}, have ${VG_FREE}G)"
    exit 1
  fi

  # Vérifier MongoDB
  if ! mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGO_HOST}/admin" \
    --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log "ERROR: Cannot connect to MongoDB"
    exit 1
  fi

  log "✓ Prerequisites OK"
}

# Créer le snapshot
create_snapshot() {
  local timestamp=$(date +%Y%m%d_%H%M%S)
  local snapshot_name="snap_mongodb_${timestamp}"

  log "Creating LVM snapshot: $snapshot_name"

  # Option 1: Avec fsyncLock (pause < 2s)
  if [ "${USE_FSYNC_LOCK:-true}" = "true" ]; then
    log "Acquiring fsyncLock..."
    mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGO_HOST}/admin" \
      --eval "db.fsyncLock()"

    local lock_acquired=$?
    if [ $lock_acquired -ne 0 ]; then
      log "ERROR: Failed to acquire fsyncLock"
      exit 1
    fi
  fi

  # Créer le snapshot LVM
  local start_time=$(date +%s)

  if lvcreate --size "$SNAPSHOT_SIZE" \
    --snapshot \
    --name "$snapshot_name" \
    "/dev/$VG_NAME/$LV_NAME"; then

    local end_time=$(date +%s)
    local duration=$((end_time - start_time))

    log "✓ Snapshot created in ${duration}s"
  else
    log "ERROR: Failed to create snapshot"

    # Débloquer en cas d'erreur
    if [ "${USE_FSYNC_LOCK:-true}" = "true" ]; then
      mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGO_HOST}/admin" \
        --eval "db.fsyncUnlock()" || true
    fi

    exit 1
  fi

  # Option 1: Débloquer immédiatement
  if [ "${USE_FSYNC_LOCK:-true}" = "true" ]; then
    log "Releasing fsyncLock..."
    mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${MONGO_HOST}/admin" \
      --eval "db.fsyncUnlock()"

    log "MongoDB unlocked (total lock time: ${duration}s)"
  fi

  echo "$snapshot_name"
}

# Monter et copier le snapshot
backup_snapshot() {
  local snapshot_name=$1
  local timestamp=$(echo "$snapshot_name" | sed 's/snap_mongodb_//')
  local backup_path="$BACKUP_DEST/mongodb_snapshot_$timestamp"

  log "Backing up snapshot to: $backup_path"

  # Créer le point de montage
  mkdir -p "$SNAPSHOT_MOUNT"

  # Monter le snapshot en lecture seule
  log "Mounting snapshot..."
  mount -o ro,nouuid "/dev/$VG_NAME/$snapshot_name" "$SNAPSHOT_MOUNT"

  if [ $? -ne 0 ]; then
    log "ERROR: Failed to mount snapshot"
    lvremove -f "/dev/$VG_NAME/$snapshot_name"
    exit 1
  fi

  # Créer le répertoire de backup
  mkdir -p "$backup_path"

  # Copier les données (rsync pour efficacité)
  log "Copying data from snapshot..."
  local start_time=$(date +%s)

  rsync -a --info=progress2 \
    --exclude="diagnostic.data/*" \
    --exclude="*.tmp" \
    "$SNAPSHOT_MOUNT/" \
    "$backup_path/"

  local end_time=$(date +%s)
  local duration=$((end_time - start_time))
  local size=$(du -sh "$backup_path" | cut -f1)

  log "✓ Backup copied in ${duration}s (size: $size)"

  # Sauvegarder les métadonnées
  cat > "$backup_path/snapshot_metadata.json" <<EOF
{
  "snapshot_name": "$snapshot_name",
  "timestamp": "$(date -Iseconds)",
  "lv_source": "/dev/$VG_NAME/$LV_NAME",
  "backup_duration_seconds": $duration,
  "size": "$size",
  "mongodb_version": "$(mongo --quiet --eval 'db.version()')",
  "snapshot_type": "lvm",
  "consistency": "${USE_FSYNC_LOCK:-true}" == "true" ? "fsync-locked" : "crash-consistent"
}
EOF

  # Démonter le snapshot
  log "Unmounting snapshot..."
  umount "$SNAPSHOT_MOUNT"

  # Supprimer le snapshot LVM
  log "Removing LVM snapshot..."
  lvremove -f "/dev/$VG_NAME/$snapshot_name"

  # Compression (optionnelle)
  if [ "${COMPRESS_BACKUP:-true}" = "true" ]; then
    log "Compressing backup..."
    tar -czf "$backup_path.tar.gz" -C "$BACKUP_DEST" "$(basename $backup_path)"

    # Checksum
    sha256sum "$backup_path.tar.gz" > "$backup_path.tar.gz.sha256"

    # Supprimer le répertoire non compressé
    rm -rf "$backup_path"

    log "✓ Backup compressed: $backup_path.tar.gz"
  fi
}

# Cleanup des anciens backups
cleanup_old_backups() {
  local retention_days=${RETENTION_DAYS:-30}

  log "Cleaning up backups older than $retention_days days..."
  find "$BACKUP_DEST" -name "mongodb_snapshot_*" -mtime +$retention_days -delete

  log "✓ Cleanup completed"
}

# Main
main() {
  log "=== MongoDB LVM Snapshot Backup ==="

  check_prerequisites

  local snapshot_name=$(create_snapshot)

  backup_snapshot "$snapshot_name"

  cleanup_old_backups

  log "✓ Backup completed successfully"
}

# Gestion des erreurs
trap 'log "ERROR: Script failed"; exit 1' ERR

main "$@"
```

### Restauration depuis Snapshot LVM

```bash
#!/bin/bash
# restore_from_lvm_snapshot.sh

set -euo pipefail

BACKUP_FILE="/backup/mongodb/lvm_snapshots/mongodb_snapshot_20241208.tar.gz"
MONGO_DATA_DIR="/var/lib/mongodb"
MONGO_SERVICE="mongod"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== MongoDB LVM Snapshot Restoration ==="
log "Source: $BACKUP_FILE"

# 1. Arrêter MongoDB
log "Stopping MongoDB..."
systemctl stop "$MONGO_SERVICE"

# 2. Sauvegarder les données actuelles
log "Backing up current data..."
mv "$MONGO_DATA_DIR" "${MONGO_DATA_DIR}.bak_$(date +%s)"

# 3. Créer le répertoire et extraire
log "Extracting snapshot backup..."
mkdir -p "$MONGO_DATA_DIR"
tar -xzf "$BACKUP_FILE" -C "$MONGO_DATA_DIR" --strip-components=1

# 4. Ajuster les permissions
log "Setting permissions..."
chown -R mongodb:mongodb "$MONGO_DATA_DIR"

# 5. Démarrer MongoDB
log "Starting MongoDB..."
systemctl start "$MONGO_SERVICE"

# 6. Attendre que MongoDB soit prêt
log "Waiting for MongoDB to be ready..."
for i in {1..30}; do
  if mongo --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log "✓ MongoDB is ready"
    break
  fi
  sleep 2
done

# 7. Vérifier l'intégrité
log "Verifying data integrity..."
mongo --eval "
  dbs = db.adminCommand({ listDatabases: 1 }).databases;
  print('Databases: ' + dbs.length);

  dbs.forEach(function(database) {
    if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
      db.getSiblingDB(database.name).getCollectionNames().forEach(function(coll) {
        count = db.getSiblingDB(database.name)[coll].countDocuments();
        print('  ' + database.name + '.' + coll + ': ' + count + ' documents');
      });
    }
  });
"

log "✓ Restoration completed successfully"
```

## Snapshots Cloud

### AWS EBS Snapshots

```bash
#!/bin/bash
# aws_ebs_snapshot.sh

set -euo pipefail

# Configuration
INSTANCE_ID="i-1234567890abcdef0"
VOLUME_ID="vol-0123456789abcdef0"
REGION="us-east-1"
RETENTION_DAYS=30

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Créer le snapshot
create_ebs_snapshot() {
  log "Creating EBS snapshot for volume $VOLUME_ID"

  # Optionnel: fsyncLock via SSM
  if [ "${USE_FSYNC_LOCK:-false}" = "true" ]; then
    log "Acquiring fsyncLock via SSM..."
    aws ssm send-command \
      --instance-ids "$INSTANCE_ID" \
      --document-name "AWS-RunShellScript" \
      --parameters 'commands=["mongo admin --eval \"db.fsyncLock()\""]' \
      --region "$REGION"

    sleep 2
  fi

  # Créer le snapshot
  local timestamp=$(date +%Y%m%d_%H%M%S)
  local description="MongoDB backup $timestamp"

  local snapshot_id=$(aws ec2 create-snapshot \
    --volume-id "$VOLUME_ID" \
    --description "$description" \
    --tag-specifications "ResourceType=snapshot,Tags=[{Key=Name,Value=mongodb-backup-$timestamp},{Key=Type,Value=automated},{Key=Application,Value=mongodb}]" \
    --region "$REGION" \
    --query 'SnapshotId' \
    --output text)

  log "✓ Snapshot created: $snapshot_id"

  # Débloquer MongoDB
  if [ "${USE_FSYNC_LOCK:-false}" = "true" ]; then
    aws ssm send-command \
      --instance-ids "$INSTANCE_ID" \
      --document-name "AWS-RunShellScript" \
      --parameters 'commands=["mongo admin --eval \"db.fsyncUnlock()\""]' \
      --region "$REGION"

    log "MongoDB unlocked"
  fi

  echo "$snapshot_id"
}

# Attendre la complétion
wait_for_snapshot() {
  local snapshot_id=$1

  log "Waiting for snapshot completion..."

  while true; do
    local state=$(aws ec2 describe-snapshots \
      --snapshot-ids "$snapshot_id" \
      --region "$REGION" \
      --query 'Snapshots[0].State' \
      --output text)

    local progress=$(aws ec2 describe-snapshots \
      --snapshot-ids "$snapshot_id" \
      --region "$REGION" \
      --query 'Snapshots[0].Progress' \
      --output text)

    if [ "$state" = "completed" ]; then
      log "✓ Snapshot completed"
      break
    elif [ "$state" = "error" ]; then
      log "✗ Snapshot failed"
      exit 1
    fi

    log "  Progress: $progress (state: $state)"
    sleep 30
  done
}

# Cleanup anciens snapshots
cleanup_old_snapshots() {
  log "Cleaning up old snapshots..."

  local cutoff_date=$(date -d "$RETENTION_DAYS days ago" +%Y-%m-%d)

  local old_snapshots=$(aws ec2 describe-snapshots \
    --owner-ids self \
    --filters "Name=tag:Application,Values=mongodb" \
    --query "Snapshots[?StartTime<='$cutoff_date'].SnapshotId" \
    --region "$REGION" \
    --output text)

  for snapshot_id in $old_snapshots; do
    log "  Deleting snapshot: $snapshot_id"
    aws ec2 delete-snapshot \
      --snapshot-id "$snapshot_id" \
      --region "$REGION"
  done

  log "✓ Cleanup completed"
}

# Main
main() {
  log "=== AWS EBS Snapshot Backup ==="

  local snapshot_id=$(create_ebs_snapshot)

  wait_for_snapshot "$snapshot_id"

  # Copier vers une autre région (disaster recovery)
  if [ "${CROSS_REGION_COPY:-false}" = "true" ]; then
    log "Copying snapshot to secondary region..."
    aws ec2 copy-snapshot \
      --source-region "$REGION" \
      --source-snapshot-id "$snapshot_id" \
      --destination-region "${SECONDARY_REGION:-us-west-2}" \
      --description "Cross-region copy of $snapshot_id"
  fi

  cleanup_old_snapshots

  log "✓ EBS snapshot backup completed: $snapshot_id"
}

main "$@"
```

### Restauration EBS

```bash
#!/bin/bash
# restore_from_ebs_snapshot.sh

SNAPSHOT_ID="snap-0123456789abcdef0"
AVAILABILITY_ZONE="us-east-1a"
VOLUME_TYPE="gp3"
VOLUME_SIZE=1000  # GB
IOPS=16000
THROUGHPUT=1000  # MB/s

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== EBS Snapshot Restoration ==="

# 1. Créer un volume depuis le snapshot
log "Creating volume from snapshot $SNAPSHOT_ID..."

VOLUME_ID=$(aws ec2 create-volume \
  --snapshot-id "$SNAPSHOT_ID" \
  --availability-zone "$AVAILABILITY_ZONE" \
  --volume-type "$VOLUME_TYPE" \
  --size "$VOLUME_SIZE" \
  --iops "$IOPS" \
  --throughput "$THROUGHPUT" \
  --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=mongodb-restored}]" \
  --query 'VolumeId' \
  --output text)

log "Volume created: $VOLUME_ID"

# 2. Attendre que le volume soit disponible
log "Waiting for volume to be available..."
aws ec2 wait volume-available --volume-ids "$VOLUME_ID"

log "✓ Volume ready: $VOLUME_ID"
log "Attach this volume to your MongoDB instance and mount at /var/lib/mongodb"
```

### Azure Managed Disk Snapshots

```bash
#!/bin/bash
# azure_snapshot.sh

RESOURCE_GROUP="mongodb-prod"
DISK_NAME="mongodb-data-disk"
SNAPSHOT_NAME="mongodb-snapshot-$(date +%Y%m%d-%H%M%S)"
LOCATION="eastus"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Azure Managed Disk Snapshot ==="

# Créer le snapshot
log "Creating snapshot: $SNAPSHOT_NAME"

az snapshot create \
  --resource-group "$RESOURCE_GROUP" \
  --name "$SNAPSHOT_NAME" \
  --source "$DISK_NAME" \
  --location "$LOCATION" \
  --tags Application=mongodb Type=backup Date=$(date +%Y%m%d)

if [ $? -eq 0 ]; then
  log "✓ Snapshot created successfully"
else
  log "✗ Snapshot creation failed"
  exit 1
fi

# Copier vers une autre région (DR)
SECONDARY_LOCATION="westus2"
SECONDARY_RG="mongodb-prod-dr"

log "Copying snapshot to $SECONDARY_LOCATION..."

az snapshot create \
  --resource-group "$SECONDARY_RG" \
  --name "${SNAPSHOT_NAME}-dr" \
  --location "$SECONDARY_LOCATION" \
  --source $(az snapshot show -g "$RESOURCE_GROUP" -n "$SNAPSHOT_NAME" --query id -o tsv)

log "✓ Cross-region copy completed"
```

### GCP Persistent Disk Snapshots

```bash
#!/bin/bash
# gcp_snapshot.sh

PROJECT_ID="my-mongodb-project"
ZONE="us-central1-a"
DISK_NAME="mongodb-data-disk"
SNAPSHOT_NAME="mongodb-snapshot-$(date +%Y%m%d-%H%M%S)"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== GCP Persistent Disk Snapshot ==="

# Créer le snapshot
log "Creating snapshot: $SNAPSHOT_NAME"

gcloud compute disks snapshot "$DISK_NAME" \
  --project="$PROJECT_ID" \
  --snapshot-names="$SNAPSHOT_NAME" \
  --zone="$ZONE" \
  --labels="application=mongodb,type=backup,date=$(date +%Y%m%d)"

if [ $? -eq 0 ]; then
  log "✓ Snapshot created successfully"

  # Les snapshots GCP sont automatiquement multi-régionaux
  log "Snapshot is stored in multi-regional storage"
else
  log "✗ Snapshot creation failed"
  exit 1
fi

# Cleanup anciens snapshots
RETENTION_DAYS=30
CUTOFF_DATE=$(date -d "$RETENTION_DAYS days ago" +%Y-%m-%d)

log "Cleaning up snapshots older than $RETENTION_DAYS days..."

gcloud compute snapshots list \
  --filter="creationTimestamp<$CUTOFF_DATE AND labels.application=mongodb" \
  --format="value(name)" | while read snapshot; do

  log "  Deleting snapshot: $snapshot"
  gcloud compute snapshots delete "$snapshot" --quiet
done

log "✓ GCP snapshot backup completed"
```

## Snapshots pour Replica Sets

### Stratégie de Snapshot sur Secondary

```bash
#!/bin/bash
# replica_set_snapshot_strategy.sh

RS_NAME="rs0"
BACKUP_MEMBER="mongo-backup:27017"
MONGO_USER="admin"
MONGO_PASS="SecurePass"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Vérifier l'état du membre
check_member_state() {
  local state=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${BACKUP_MEMBER}/admin" --quiet --eval "
    rs.status().members.filter(m => m.name === '${BACKUP_MEMBER}')[0].stateStr;
  ")

  if [ "$state" != "SECONDARY" ]; then
    log "ERROR: Member is not SECONDARY (state: $state)"
    exit 1
  fi

  log "✓ Member is SECONDARY"
}

# Vérifier le replication lag
check_replication_lag() {
  local lag=$(mongo "mongodb://${MONGO_USER}:${MONGO_PASS}@${BACKUP_MEMBER}/admin" --quiet --eval "
    member = rs.status().members.filter(m => m.name === '${BACKUP_MEMBER}')[0];
    print((new Date() - member.optimeDate) / 1000);
  ")

  log "Replication lag: ${lag}s"

  if (( $(echo "$lag > 300" | bc -l) )); then
    log "WARNING: High replication lag, waiting..."
    sleep 60
    check_replication_lag
  fi
}

# Snapshot sur le secondary
perform_snapshot() {
  log "=== Replica Set Member Snapshot ==="
  log "Target member: $BACKUP_MEMBER"

  check_member_state
  check_replication_lag

  # Option 1: Sans fsyncLock (recommandé pour secondary)
  log "Creating snapshot without fsyncLock (crash-consistent)..."

  # Créer le snapshot LVM sur le serveur distant
  ssh mongodb-backup-server "
    lvcreate --size 200G \
      --snapshot \
      --name snap_mongodb_secondary_$(date +%Y%m%d_%H%M%S) \
      /dev/vg_mongodb/lv_mongodb_data
  "

  log "✓ Snapshot completed"

  # Vérifier que la réplication continue normalement
  check_replication_lag
}

perform_snapshot
```

### Snapshot Coordonné du Replica Set

Pour sauvegarder tous les membres simultanément (rare, mais utile pour audit) :

```bash
#!/bin/bash
# coordinated_replica_set_snapshot.sh

RS_MEMBERS=(
  "mongo-1:27017:/dev/vg_mongo1/lv_data"
  "mongo-2:27017:/dev/vg_mongo2/lv_data"
  "mongo-3:27017:/dev/vg_mongo3/lv_data"
)

SNAPSHOT_TIMESTAMP=$(date +%Y%m%d_%H%M%S)

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Coordinated Replica Set Snapshot ==="

# 1. Arrêter le balancer (si sharded)
# mongo mongos:27017/admin --eval "sh.stopBalancer()"

# 2. Créer snapshots sur tous les membres en parallèle
log "Creating snapshots on all members..."

PIDS=()
for member_config in "${RS_MEMBERS[@]}"; do
  IFS=':' read -r host port lv_path <<< "$member_config"

  (
    log "  Snapshotting $host..."
    ssh "$host" "lvcreate --size 200G --snapshot --name snap_${SNAPSHOT_TIMESTAMP} $lv_path"
  ) &

  PIDS+=($!)
done

# Attendre tous les snapshots
for pid in "${PIDS[@]}"; do
  wait $pid
done

log "✓ All snapshots created"

# 3. Redémarrer le balancer
# mongo mongos:27017/admin --eval "sh.startBalancer()"
```

## Snapshots ZFS

ZFS offre des snapshots natifs très performants :

```bash
#!/bin/bash
# zfs_snapshot.sh

ZPOOL="mongodb"
DATASET="mongodb/data"
SNAPSHOT_NAME="backup-$(date +%Y%m%d-%H%M%S)"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== ZFS Snapshot Backup ==="

# Créer le snapshot (instantané)
log "Creating ZFS snapshot..."
zfs snapshot "${ZPOOL}/${DATASET}@${SNAPSHOT_NAME}"

if [ $? -eq 0 ]; then
  log "✓ Snapshot created: ${ZPOOL}/${DATASET}@${SNAPSHOT_NAME}"
else
  log "✗ Snapshot creation failed"
  exit 1
fi

# Exporter le snapshot vers un fichier
BACKUP_FILE="/backup/zfs/${SNAPSHOT_NAME}.zfs"
mkdir -p "$(dirname $BACKUP_FILE)"

log "Exporting snapshot to file..."
zfs send "${ZPOOL}/${DATASET}@${SNAPSHOT_NAME}" | gzip > "$BACKUP_FILE.gz"

log "✓ Snapshot exported: $BACKUP_FILE.gz"
log "  Size: $(du -h $BACKUP_FILE.gz | cut -f1)"

# Optionnel: Snapshot incrémental
# LAST_SNAPSHOT=$(zfs list -t snapshot -o name -s creation | grep "^${ZPOOL}/${DATASET}@" | tail -2 | head -1)
# zfs send -i "$LAST_SNAPSHOT" "${ZPOOL}/${DATASET}@${SNAPSHOT_NAME}" | gzip > "${BACKUP_FILE}_incremental.gz"

# Cleanup anciens snapshots
RETENTION_DAYS=30
CUTOFF_DATE=$(date -d "$RETENTION_DAYS days ago" +%s)

log "Cleaning up old snapshots..."
zfs list -t snapshot -o name,creation -H | grep "^${ZPOOL}/${DATASET}@" | while read name creation_date; do
  snapshot_date=$(date -d "$creation_date" +%s)

  if [ $snapshot_date -lt $CUTOFF_DATE ]; then
    log "  Destroying snapshot: $name"
    zfs destroy "$name"
  fi
done

log "✓ ZFS snapshot backup completed"
```

### Restauration ZFS

```bash
#!/bin/bash
# zfs_restore.sh

BACKUP_FILE="/backup/zfs/backup-20241208-020000.zfs.gz"
ZPOOL="mongodb"
DATASET="mongodb/data"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== ZFS Snapshot Restoration ==="

# 1. Arrêter MongoDB
systemctl stop mongod

# 2. Détruire le dataset actuel (ATTENTION: destructif!)
log "Destroying current dataset..."
zfs destroy -r "${ZPOOL}/${DATASET}"

# 3. Restaurer depuis le snapshot
log "Restoring from snapshot..."
gunzip -c "$BACKUP_FILE.gz" | zfs receive "${ZPOOL}/${DATASET}"

if [ $? -eq 0 ]; then
  log "✓ Dataset restored"
else
  log "✗ Restoration failed"
  exit 1
fi

# 4. Redémarrer MongoDB
log "Starting MongoDB..."
systemctl start mongod

log "✓ ZFS restoration completed"
```

## Automatisation et Orchestration

### Systemd Timer pour Snapshots

```ini
# /etc/systemd/system/mongodb-snapshot.service
[Unit]
Description=MongoDB LVM Snapshot Backup
After=network.target mongod.service

[Service]
Type=oneshot
User=root
ExecStart=/usr/local/bin/lvm_snapshot_backup.sh
StandardOutput=journal
StandardError=journal
```

```ini
# /etc/systemd/system/mongodb-snapshot.timer
[Unit]
Description=MongoDB Snapshot Backup Timer
Requires=mongodb-snapshot.service

[Timer]
# Exécuter tous les jours à 2h du matin
OnCalendar=daily
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
# Activer le timer
systemctl daemon-reload
systemctl enable mongodb-snapshot.timer
systemctl start mongodb-snapshot.timer

# Vérifier le statut
systemctl list-timers mongodb-snapshot.timer
```

### Terraform pour Automatisation AWS

```hcl
# mongodb_ebs_snapshot_automation.tf

resource "aws_dlm_lifecycle_policy" "mongodb_backup" {
  description        = "MongoDB EBS Snapshot Lifecycle"
  execution_role_arn = aws_iam_role.dlm_lifecycle_role.arn
  state              = "ENABLED"

  policy_details {
    resource_types = ["VOLUME"]

    schedule {
      name = "Daily MongoDB Backup"

      create_rule {
        interval      = 24
        interval_unit = "HOURS"
        times         = ["02:00"]
      }

      retain_rule {
        count = 30  # Conserver 30 jours
      }

      tags_to_add = {
        SnapshotType = "automated"
        Application  = "mongodb"
      }

      copy_tags = true
    }

    # Copie cross-region
    schedule {
      name = "Weekly Cross-Region Copy"

      create_rule {
        interval      = 7
        interval_unit = "DAYS"
        times         = ["03:00"]
      }

      cross_region_copy_rule {
        target    = "us-west-2"
        encrypted = true
        cmk_arn   = aws_kms_key.backup.arn

        retain_rule {
          interval      = 90
          interval_unit = "DAYS"
        }
      }
    }

    target_tags = {
      Backup = "mongodb"
    }
  }
}

# IAM Role pour DLM
resource "aws_iam_role" "dlm_lifecycle_role" {
  name = "dlm-lifecycle-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "dlm.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "dlm_lifecycle" {
  name = "dlm-lifecycle-policy"
  role = aws_iam_role.dlm_lifecycle_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:CreateSnapshot",
          "ec2:CreateSnapshots",
          "ec2:DeleteSnapshot",
          "ec2:DescribeVolumes",
          "ec2:DescribeSnapshots",
          "ec2:CopySnapshot"
        ]
        Resource = "*"
      }
    ]
  })
}
```

## Monitoring et Validation

### Script de Monitoring des Snapshots

```bash
#!/bin/bash
# monitor_snapshots.sh

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Vérifier les snapshots LVM
check_lvm_snapshots() {
  echo "=== LVM Snapshots ==="

  lvs --noheadings -o lv_name,origin,lv_size,data_percent | grep -v "^  $" | while read lv origin size percent; do
    if [ -n "$origin" ]; then
      echo "Snapshot: $lv"
      echo "  Origin: $origin"
      echo "  Size: $size"
      echo "  Used: ${percent}%"

      # Alert si > 80%
      if (( $(echo "$percent > 80" | bc -l) )); then
        echo "  ⚠️  WARNING: Snapshot is ${percent}% full"
      fi
    fi
  done
}

# Vérifier les backups récents
check_recent_backups() {
  echo ""
  echo "=== Recent Backups ==="

  BACKUP_DIR="/backup/mongodb/lvm_snapshots"
  HOURS_THRESHOLD=26  # Alert si pas de backup depuis 26h

  LATEST_BACKUP=$(find "$BACKUP_DIR" -name "mongodb_snapshot_*" -type f -printf '%T@ %p\n' | sort -n | tail -1)

  if [ -z "$LATEST_BACKUP" ]; then
    echo "✗ No backups found"
    exit 1
  fi

  LATEST_TIME=$(echo "$LATEST_BACKUP" | cut -d' ' -f1)
  LATEST_FILE=$(echo "$LATEST_BACKUP" | cut -d' ' -f2)
  CURRENT_TIME=$(date +%s)
  AGE_HOURS=$(echo "($CURRENT_TIME - $LATEST_TIME) / 3600" | bc)

  echo "Latest backup: $(basename $LATEST_FILE)"
  echo "Age: ${AGE_HOURS}h"
  echo "Size: $(du -h $LATEST_FILE | cut -f1)"

  if [ $AGE_HOURS -gt $HOURS_THRESHOLD ]; then
    echo "✗ Latest backup is too old (${AGE_HOURS}h > ${HOURS_THRESHOLD}h)"
    exit 1
  else
    echo "✓ Backup is recent"
  fi
}

check_lvm_snapshots
check_recent_backups
```

### Métriques Prometheus

```yaml
# prometheus_snapshot_exporter.sh
#!/bin/bash
# Exporter des métriques pour Prometheus

METRICS_FILE="/var/lib/node_exporter/textfile_collector/mongodb_snapshots.prom"

# Dernière sauvegarde réussie
LAST_BACKUP=$(find /backup/mongodb -name "*.tar.gz" -printf '%T@\n' | sort -n | tail -1)
echo "mongodb_snapshot_last_success_timestamp $LAST_BACKUP" > "$METRICS_FILE"

# Taille du dernier backup
LAST_SIZE=$(find /backup/mongodb -name "*.tar.gz" -printf '%s\n' | sort -n | tail -1)
echo "mongodb_snapshot_last_size_bytes $LAST_SIZE" >> "$METRICS_FILE"

# Nombre de snapshots LVM actifs
SNAPSHOT_COUNT=$(lvs --noheadings -o lv_name,origin | grep -v "^  $" | grep -c -v "^$")
echo "mongodb_lvm_snapshots_active $SNAPSHOT_COUNT" >> "$METRICS_FILE"
```

## Bonnes Pratiques

### Checklist de Production

```markdown
### Configuration Initiale
- [ ] Stockage configuré avec snapshot support (LVM/ZFS/Cloud)
- [ ] Espace suffisant pour snapshots (20-30% du volume)
- [ ] Permissions et credentials configurés
- [ ] Scripts de backup testés
- [ ] Automatisation mise en place
- [ ] Monitoring et alerting configurés

### Avant Chaque Snapshot
- [ ] Vérifier l'espace disque disponible
- [ ] Confirmer l'état de MongoDB (healthy)
- [ ] Si Replica Set: vérifier replication lag < 5 min
- [ ] Vérifier fenêtre de maintenance si fsyncLock
- [ ] Confirmer stockage de destination accessible

### Pendant le Snapshot
- [ ] Monitorer durée de création (< 2s normal)
- [ ] Si fsyncLock: vérifier déblocage automatique
- [ ] Surveiller espace CoW du snapshot
- [ ] Confirmer pas d'impact sur production

### Après le Snapshot
- [ ] Vérifier que le snapshot existe
- [ ] Valider la taille (cohérence avec attendu)
- [ ] Copie vers stockage distant réussie
- [ ] Checksum généré et sauvegardé
- [ ] Métadonnées enregistrées
- [ ] Cleanup des anciens snapshots effectué
- [ ] Test de restauration (périodique)
```

### Stratégie de Rétention

```yaml
Snapshots Locaux (LVM/ZFS):
  fréquence: quotidien
  rétention: 7 jours
  objectif: Recovery rapide incidents récents

Backups Cloud (EBS/Azure/GCP):
  fréquence: quotidien
  rétention: 30 jours (standard tier)
  objectif: Recovery moyen terme

Archives Long Terme:
  fréquence: mensuel
  rétention: 1-7 ans (cold storage)
  objectif: Compliance et audit
```

## Comparaison : Snapshots vs mongodump

```
┌──────────────────┬─────────────────┬──────────────────┐
│    Critère       │   Snapshots     │   mongodump      │
├──────────────────┼─────────────────┼──────────────────┤
│ Vitesse backup   │ < 2 secondes    │ Minutes/heures   │
│ Impact prod      │ Minimal         │ CPU/réseau       │
│ Taille > 1TB     │ ✓ Optimal       │ ✗ Lent           │
│ Portabilité      │ ✗ Dépendant OS  │ ✓ Universel      │
│ Granularité      │ ✗ Tout ou rien  │ ✓ Par collection │
│ Cross-platform   │ ✗ Limité        │ ✓ Oui            │
│ Compression      │ Via système     │ ✓ Intégrée       │
│ Restauration     │ Rapide (mount)  │ Rebuild index    │
│ Complexité       │ Moyenne-Élevée  │ Faible           │
│ Use case         │ Prod > 100GB    │ Toutes tailles   │
└──────────────────┴─────────────────┴──────────────────┘
```

### Quand Utiliser les Snapshots

**Utiliser les snapshots quand** :
- Volume de données > 500 GB
- RTO agressif (< 1 heure)
- Backup pendant heures de pointe acceptable
- Infrastructure compatible (LVM/ZFS/Cloud)
- Même OS/version pour restauration

**Utiliser mongodump quand** :
- Volume < 100 GB
- Besoin de portabilité cross-platform
- Migration entre versions MongoDB
- Export sélectif de données
- Environnement sans snapshot support

## Conclusion

Les snapshots du système de fichiers constituent une méthode de sauvegarde puissante et performante pour MongoDB, particulièrement adaptée aux environnements de production à grande échelle. Leur création quasi-instantanée et leur impact minimal font d'eux le choix préféré pour les bases > 500 GB.

**Points clés à retenir** :
1. **Crash-consistent suffit** - WiredTiger assure la récupération
2. **Snapshots sur secondary** - Pour Replica Sets
3. **Cloud-native** - Utiliser les snapshots cloud quand disponible
4. **Test régulier** - Valider la recouvrabilité
5. **Automatisation** - Snapshots programmés et monitoring

Une stratégie optimale combine souvent snapshots (backup rapide) et mongodump occasionnel (portabilité et archivage).

---


⏭️ [MongoDB Atlas Backup](/12-sauvegarde-restauration/06-mongodb-atlas-backup.md)
