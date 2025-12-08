🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.1 Stratégies de Sauvegarde

## Introduction

Une stratégie de sauvegarde efficace pour MongoDB nécessite une analyse approfondie des besoins métier, des contraintes techniques et des objectifs de récupération. Il n'existe pas de solution universelle : chaque organisation doit définir sa stratégie en fonction de sa criticité métier, de son volume de données, de ses fenêtres de maintenance et de son budget.

Cette section présente les différentes approches stratégiques, leurs avantages, inconvénients et cas d'usage recommandés, ainsi que des exemples concrets d'implémentation.

## Types de Sauvegardes

### Sauvegarde Complète (Full Backup)

**Définition** : Copie intégrale de toutes les données de la base à un instant T.

**Caractéristiques** :
```yaml
Avantages:
  - Restauration simple et rapide (un seul fichier)
  - Indépendance totale (pas de chaîne de dépendances)
  - Facilité de vérification
  - Idéal pour audit et archivage

Inconvénients:
  - Consomme le plus d'espace disque
  - Temps de sauvegarde le plus long
  - Bande passante réseau importante
  - Fenêtre d'exécution potentiellement problématique

Cas d'usage:
  - Première sauvegarde d'un nouveau système
  - Base de référence hebdomadaire ou mensuelle
  - Systèmes avec peu de changements
  - Avant opérations majeures (migration, upgrade)
```

**Implémentation** :
```bash
#!/bin/bash
# full_backup.sh - Sauvegarde complète MongoDB

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/mongodb/full"
BACKUP_NAME="full_backup_${TIMESTAMP}"
MONGO_URI="mongodb://backup-user:password@localhost:27017/admin"

# Création du répertoire
mkdir -p ${BACKUP_DIR}/${BACKUP_NAME}

# Sauvegarde complète avec mongodump
mongodump \
  --uri="${MONGO_URI}" \
  --oplog \
  --gzip \
  --out="${BACKUP_DIR}/${BACKUP_NAME}" \
  2>&1 | tee ${BACKUP_DIR}/${BACKUP_NAME}/backup.log

# Vérification
if [ $? -eq 0 ]; then
  # Création d'une archive
  tar -czf ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz \
    -C ${BACKUP_DIR} ${BACKUP_NAME}

  # Checksum
  sha256sum ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz \
    > ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz.sha256

  # Métadonnées
  cat > ${BACKUP_DIR}/${BACKUP_NAME}.metadata <<EOF
{
  "backup_type": "full",
  "timestamp": "${TIMESTAMP}",
  "size_bytes": $(stat -f%z ${BACKUP_DIR}/${BACKUP_NAME}.tar.gz),
  "mongodb_version": "$(mongod --version | head -1)",
  "databases": $(mongo ${MONGO_URI} --quiet --eval "db.adminCommand('listDatabases').databases.length"),
  "status": "success"
}
EOF

  # Nettoyage du répertoire temporaire
  rm -rf ${BACKUP_DIR}/${BACKUP_NAME}

  echo "✓ Full backup completed: ${BACKUP_NAME}.tar.gz"
else
  echo "✗ Full backup failed"
  exit 1
fi
```

### Sauvegarde Incrémentale (Incremental Backup)

**Définition** : Sauvegarde uniquement des modifications depuis la dernière sauvegarde (complète ou incrémentale).

**Principe via Oplog** :
```javascript
// MongoDB utilise l'oplog pour les sauvegardes incrémentales
// L'oplog enregistre toutes les opérations d'écriture

// Structure de l'oplog
{
  "ts": Timestamp(1701993600, 1),     // Timestamp de l'opération
  "t": NumberLong(1),                  // Terme de réplication
  "h": NumberLong("1234567890"),       // Hash de l'opération
  "v": 2,                              // Version oplog
  "op": "i",                           // Type: i=insert, u=update, d=delete
  "ns": "mydb.users",                  // Namespace
  "o": { "_id": ObjectId("..."), ... } // Objet de l'opération
}
```

**Implémentation** :
```bash
#!/bin/bash
# incremental_backup.sh - Sauvegarde incrémentale via oplog

BACKUP_DIR="/backup/mongodb/incremental"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
STATE_FILE="/var/lib/mongodb_backup/last_oplog_timestamp"

# Lire le dernier timestamp sauvegardé
if [ -f "$STATE_FILE" ]; then
  LAST_TS=$(cat $STATE_FILE)
  echo "Last backup timestamp: $LAST_TS"
else
  echo "No previous backup found. Run full backup first."
  exit 1
fi

# Sauvegarder les entrées oplog depuis le dernier backup
mongo --quiet <<EOF
  use local
  db.oplog.rs.find({
    ts: { \$gt: Timestamp($LAST_TS) }
  }).forEach(function(doc) {
    printjson(doc);
  })
EOF > ${BACKUP_DIR}/oplog_${TIMESTAMP}.json

# Compression
gzip ${BACKUP_DIR}/oplog_${TIMESTAMP}.json

# Sauvegarder le nouveau timestamp
CURRENT_TS=$(mongo --quiet --eval "
  db = db.getSiblingDB('local');
  printjson(db.oplog.rs.find().sort({ts:-1}).limit(1).toArray()[0].ts);
")

echo "$CURRENT_TS" > $STATE_FILE

# Métadonnées
cat > ${BACKUP_DIR}/oplog_${TIMESTAMP}.metadata <<EOF
{
  "backup_type": "incremental",
  "timestamp": "${TIMESTAMP}",
  "from_ts": $LAST_TS,
  "to_ts": $CURRENT_TS,
  "size_bytes": $(stat -c%s ${BACKUP_DIR}/oplog_${TIMESTAMP}.json.gz)
}
EOF

echo "✓ Incremental backup completed: oplog_${TIMESTAMP}.json.gz"
```

**Chaîne de restauration** :
```bash
# Pour restaurer, il faut :
# 1. La dernière sauvegarde complète
# 2. Toutes les sauvegardes incrémentales dans l'ordre

# Exemple de restauration
mongorestore --gzip --archive=/backup/full/full_backup_20241201.tar.gz
mongorestore --oplogReplay --oplogFile=/backup/incremental/oplog_20241202.json.gz
mongorestore --oplogReplay --oplogFile=/backup/incremental/oplog_20241203.json.gz
# ... continuer pour chaque fichier incrémental
```

### Sauvegarde Différentielle (Differential Backup)

**Définition** : Sauvegarde de toutes les modifications depuis la dernière sauvegarde complète (pas depuis la dernière différentielle).

**Comparaison** :
```
Chronologie des sauvegardes sur une semaine:

Full Backup:     [FULL]
                   ↓
Incrémentale:    [FULL]→[Inc1]→[Inc2]→[Inc3]→[Inc4]→[Inc5]→[Inc6]
Taille:          100%   5%     5%     5%     5%     5%     5%
Restauration:    1 + 6 fichiers nécessaires

Différentielle:  [FULL]→[Diff1]→[Diff2]→[Diff3]→[Diff4]→[Diff5]→[Diff6]
Taille:          100%   5%      10%     15%     20%     25%     30%
Restauration:    1 + 1 fichier nécessaire
```

**Avantages et inconvénients** :
```yaml
Incrémentale:
  avantages:
    - Sauvegardes plus rapides
    - Moins d'espace disque utilisé
  inconvénients:
    - Restauration complexe (chaîne complète)
    - Point de défaillance unique (perte d'une sauvegarde)

Différentielle:
  avantages:
    - Restauration plus simple (full + dernière diff)
    - Plus tolérant aux erreurs
  inconvénients:
    - Sauvegardes de plus en plus longues
    - Plus d'espace disque que incrémentale
```

## Stratégies par Environnement

### Environnement de Production Critique

**Caractéristiques** :
- RTO : < 1 heure
- RPO : < 15 minutes
- Disponibilité : 99.99% (4.38 min/mois)

**Stratégie recommandée** :
```yaml
Architecture:
  - Replica Set 5 membres minimum
  - 1 membre dédié backup (hidden, priority: 0)
  - Géo-distribution sur 3 datacenters

Sauvegarde Continue:
  méthode: MongoDB Ops Manager / Atlas
  snapshot_interval: 15 minutes
  retention:
    snapshots: 48 heures
    daily: 30 jours
    weekly: 3 mois
    monthly: 1 an

Sauvegarde Complémentaire:
  full_backup:
    fréquence: hebdomadaire
    fenêtre: dimanche 02h00
    destination:
      - Local: SAN avec réplication
      - Remote: S3 Glacier Deep Archive

  incremental:
    fréquence: toutes les 4 heures
    méthode: oplog export
    destination: S3 Standard-IA

Tests:
  restauration_partielle: hebdomadaire
  restauration_complète: mensuel
  disaster_recovery_drill: trimestriel
```

**Implémentation** :
```bash
#!/bin/bash
# production_critical_backup_strategy.sh

# Configuration
PRIMARY_BACKUP_DIR="/backup/primary"
REMOTE_BACKUP_BUCKET="s3://company-critical-backups"
BACKUP_MEMBER="mongodb-backup-node:27017"

# Fonction: Snapshot continu (via cron toutes les 15 min)
create_snapshot() {
  local timestamp=$(date +%Y%m%d_%H%M%S)
  local snapshot_name="snapshot_${timestamp}"

  # Utiliser le membre dédié aux backups
  mongodump \
    --host=${BACKUP_MEMBER} \
    --oplog \
    --gzip \
    --archive=${PRIMARY_BACKUP_DIR}/${snapshot_name}.gz \
    --readPreference=secondary

  # Upload immédiat vers S3
  aws s3 cp ${PRIMARY_BACKUP_DIR}/${snapshot_name}.gz \
    ${REMOTE_BACKUP_BUCKET}/snapshots/ \
    --storage-class STANDARD_IA

  # Garder local seulement 48h
  find ${PRIMARY_BACKUP_DIR} -name "snapshot_*.gz" -mmin +2880 -delete
}

# Fonction: Full backup hebdomadaire
weekly_full_backup() {
  local timestamp=$(date +%Y%m%d)
  local backup_name="full_${timestamp}"

  # Backup complet
  mongodump \
    --host=${BACKUP_MEMBER} \
    --oplog \
    --gzip \
    --archive=${PRIMARY_BACKUP_DIR}/${backup_name}.gz

  # Chiffrement avant upload
  gpg --encrypt --recipient backup@company.com \
    ${PRIMARY_BACKUP_DIR}/${backup_name}.gz

  # Upload vers Glacier
  aws s3 cp ${PRIMARY_BACKUP_DIR}/${backup_name}.gz.gpg \
    ${REMOTE_BACKUP_BUCKET}/weekly/ \
    --storage-class GLACIER

  # Validation
  aws s3api head-object \
    --bucket company-critical-backups \
    --key weekly/${backup_name}.gz.gpg

  if [ $? -eq 0 ]; then
    # Envoyer notification success
    send_notification "✓ Weekly backup completed: ${backup_name}"
  fi
}

# Exécution selon le type
case "$1" in
  snapshot)
    create_snapshot
    ;;
  weekly)
    weekly_full_backup
    ;;
  *)
    echo "Usage: $0 {snapshot|weekly}"
    exit 1
    ;;
esac
```

### Environnement de Production Standard

**Caractéristiques** :
- RTO : 2-4 heures
- RPO : 1-2 heures
- Disponibilité : 99.9% (43.8 min/mois)

**Stratégie recommandée** :
```yaml
Architecture:
  - Replica Set 3 membres
  - Sauvegarde sur secondary

Sauvegarde:
  full_backup:
    fréquence: quotidienne
    heure: 02h00 (hors pic)
    méthode: mongodump
    destination:
      - Local NAS: 7 jours
      - Cloud S3: 30 jours

  incremental:
    fréquence: toutes les 6 heures
    méthode: oplog export
    rétention: 48 heures

Archivage:
  - Backup hebdomadaire conservé 3 mois
  - Backup mensuel conservé 1 an

Tests:
  restauration: bimensuel
  dr_drill: semestriel
```

**Script d'orchestration** :
```bash
#!/bin/bash
# standard_production_backup.sh

BACKUP_ROOT="/backup/mongodb"
DAILY_DIR="${BACKUP_ROOT}/daily"
WEEKLY_DIR="${BACKUP_ROOT}/weekly"
MONTHLY_DIR="${BACKUP_ROOT}/monthly"
S3_BUCKET="s3://company-backups/mongodb"

# Déterminer le type de backup
DAY_OF_WEEK=$(date +%u)  # 1-7 (1=lundi, 7=dimanche)
DAY_OF_MONTH=$(date +%d)

if [ "$DAY_OF_MONTH" == "01" ]; then
  BACKUP_TYPE="monthly"
  TARGET_DIR="$MONTHLY_DIR"
  RETENTION_DAYS=365
elif [ "$DAY_OF_WEEK" == "7" ]; then
  BACKUP_TYPE="weekly"
  TARGET_DIR="$WEEKLY_DIR"
  RETENTION_DAYS=90
else
  BACKUP_TYPE="daily"
  TARGET_DIR="$DAILY_DIR"
  RETENTION_DAYS=7
fi

TIMESTAMP=$(date +%Y%m%d)
BACKUP_NAME="${BACKUP_TYPE}_${TIMESTAMP}"

echo "Executing ${BACKUP_TYPE} backup: ${BACKUP_NAME}"

# Exécution du backup
mkdir -p "$TARGET_DIR"

mongodump \
  --host=mongodb-secondary:27017 \
  --oplog \
  --gzip \
  --archive="${TARGET_DIR}/${BACKUP_NAME}.gz" \
  --numParallelCollections=4

if [ $? -eq 0 ]; then
  # Métadonnées
  SIZE=$(stat -c%s "${TARGET_DIR}/${BACKUP_NAME}.gz")
  cat > "${TARGET_DIR}/${BACKUP_NAME}.json" <<EOF
{
  "type": "${BACKUP_TYPE}",
  "timestamp": "$(date -Iseconds)",
  "size_bytes": ${SIZE},
  "size_human": "$(numfmt --to=iec-i --suffix=B ${SIZE})",
  "retention_days": ${RETENTION_DAYS}
}
EOF

  # Upload S3
  aws s3 cp "${TARGET_DIR}/${BACKUP_NAME}.gz" \
    "${S3_BUCKET}/${BACKUP_TYPE}/" \
    --storage-class STANDARD_IA \
    --metadata "backup-type=${BACKUP_TYPE},timestamp=${TIMESTAMP}"

  # Cleanup local selon rétention
  find "$TARGET_DIR" -name "*.gz" -mtime +${RETENTION_DAYS} -delete

  echo "✓ Backup completed successfully"
  exit 0
else
  echo "✗ Backup failed"
  exit 1
fi
```

### Environnement de Développement/Test

**Caractéristiques** :
- RTO : 24 heures acceptable
- RPO : 24-48 heures
- Focus : Coût réduit, simplicité

**Stratégie recommandée** :
```yaml
Architecture:
  - Instance standalone ou Replica Set 2 membres

Sauvegarde:
  full_backup:
    fréquence: quotidienne (nuit)
    rétention: 7 jours local

  archivage:
    hebdomadaire: 1 mois
    pas de backup incrémental

Tests:
  restauration: aucun requis
  mais restauration ad-hoc fréquente pour refresh
```

**Script simplifié** :
```bash
#!/bin/bash
# dev_backup_simple.sh

BACKUP_DIR="/backup/mongodb/dev"
RETENTION_DAYS=7

# Simple backup quotidien
mongodump \
  --host=localhost:27017 \
  --gzip \
  --archive="${BACKUP_DIR}/dev_$(date +%Y%m%d).gz"

# Cleanup simple
find "$BACKUP_DIR" -name "*.gz" -mtime +${RETENTION_DAYS} -delete
```

## Stratégies par Volume de Données

### Petits Volumes (< 100 GB)

**Approche** : Backups complets fréquents

```yaml
Stratégie:
  type: Full backup uniquement
  fréquence: Toutes les 4-6 heures
  méthode: mongodump
  durée_backup: 10-30 minutes

Avantages:
  - Simplicité maximale
  - Restauration rapide et fiable
  - Pas de gestion de chaîne

Configuration:
  compression: gzip -6 (balance vitesse/ratio)
  parallélisme: 4 collections simultanées
```

### Volumes Moyens (100 GB - 1 TB)

**Approche** : Full + incrémentale

```yaml
Stratégie:
  full_backup:
    fréquence: hebdomadaire
    durée_estimée: 2-6 heures
    fenêtre: weekend hors heures de pointe

  incremental:
    fréquence: toutes les 4 heures
    via: oplog export
    durée: 5-15 minutes

Optimisations:
  - Backup sur secondary dédié
  - Compression zstd (meilleur ratio)
  - Parallélisme élevé (8+ threads)
```

**Configuration optimisée** :
```bash
#!/bin/bash
# medium_volume_backup.sh

BACKUP_SIZE_ESTIMATE_GB=500
PARALLEL_COLLECTIONS=$((BACKUP_SIZE_ESTIMATE_GB / 50))  # 1 thread par 50GB

# Full backup optimisé
mongodump \
  --host=backup-secondary:27017 \
  --oplog \
  --archive \
  --numParallelCollections=${PARALLEL_COLLECTIONS} \
  --readPreference=secondary \
  --readPreference='{ mode: "secondary", tagSets: [ { "backup": "true" } ] }' \
  | zstd -19 -T0 > /backup/full_$(date +%Y%m%d).zst

# Statistiques
echo "Backup completed in $SECONDS seconds"
echo "Size: $(du -h /backup/full_$(date +%Y%m%d).zst)"
```

### Gros Volumes (> 1 TB)

**Approche** : Snapshots système + oplog

```yaml
Stratégie:
  snapshots:
    méthode: LVM/EBS snapshots
    fréquence: quotidienne
    durée: minutes (Copy-on-Write)
    avantage: Pas d'impact performance

  oplog_continuous:
    export: continu (streaming)
    rétention: 7 jours
    objectif: Point-in-time recovery

  full_logical:
    fréquence: mensuelle (archivage)
    méthode: mongodump avec streaming S3
```

**Implémentation avec LVM** :
```bash
#!/bin/bash
# large_volume_snapshot_backup.sh

VOLUME_GROUP="vg_mongodb"
LOGICAL_VOLUME="lv_mongodb_data"
SNAPSHOT_NAME="snap_mongo_$(date +%Y%m%d_%H%M%S)"
SNAPSHOT_SIZE="50G"  # 10-20% de l'original
MOUNT_POINT="/mnt/mongo_snapshot"

# Phase 1: Préparer MongoDB
echo "Preparing MongoDB for snapshot..."
mongo admin --eval "
  db.fsyncLock();
  print('MongoDB locked for writes');
"

# Phase 2: Créer le snapshot LVM
echo "Creating LVM snapshot..."
lvcreate --size ${SNAPSHOT_SIZE} \
  --snapshot \
  --name ${SNAPSHOT_NAME} \
  /dev/${VOLUME_GROUP}/${LOGICAL_VOLUME}

# Phase 3: Débloquer MongoDB immédiatement
mongo admin --eval "
  db.fsyncUnlock();
  print('MongoDB unlocked');
"

echo "Snapshot created. MongoDB downtime: ~1 second"

# Phase 4: Monter et copier (en arrière-plan)
mkdir -p ${MOUNT_POINT}
mount -o ro /dev/${VOLUME_GROUP}/${SNAPSHOT_NAME} ${MOUNT_POINT}

# Copie vers stockage distant (pas de pression sur MongoDB)
rsync -av --progress ${MOUNT_POINT}/ /backup/snapshot_$(date +%Y%m%d)/ &

# Ou vers S3 avec streaming
tar -czf - -C ${MOUNT_POINT} . | \
  aws s3 cp - s3://backups/mongo_snapshot_$(date +%Y%m%d).tar.gz \
  --expected-size 1099511627776 &  # 1TB estimate

echo "Background copy initiated. Snapshot will be removed after completion."

# Cleanup automatique après copie
wait
umount ${MOUNT_POINT}
lvremove -f /dev/${VOLUME_GROUP}/${SNAPSHOT_NAME}
echo "✓ Snapshot backup completed"
```

**Export oplog continu** :
```javascript
// continuous_oplog_export.js
// À exécuter comme daemon

const { MongoClient } = require('mongodb');
const fs = require('fs');
const zlib = require('zlib');

const uri = 'mongodb://backup-node:27017/?replicaSet=rs0';
const client = new MongoClient(uri);

async function exportOplogContinuously() {
  await client.connect();

  const db = client.db('local');
  const oplog = db.collection('oplog.rs');

  // Dernier timestamp traité
  let lastTs = await getLastProcessedTimestamp();

  // Watch oplog en temps réel
  const changeStream = oplog.watch([
    { $match: { ts: { $gt: lastTs } } }
  ], { fullDocument: 'updateLookup' });

  const outputStream = fs.createWriteStream(
    `/backup/oplog/oplog_${new Date().toISOString()}.json.gz`
  );
  const gzipStream = zlib.createGzip();
  gzipStream.pipe(outputStream);

  changeStream.on('change', (change) => {
    gzipStream.write(JSON.stringify(change) + '\n');
    lastTs = change.ts;

    // Sauvegarder progression toutes les 1000 ops
    if (Math.random() < 0.001) {
      saveLastProcessedTimestamp(lastTs);
    }
  });

  // Rotation quotidienne
  setInterval(() => {
    gzipStream.end();
    // Créer nouveau fichier...
  }, 24 * 60 * 60 * 1000);
}

exportOplogContinuously().catch(console.error);
```

## Matrices de Décision

### Matrice de Sélection de Stratégie

```
┌─────────────┬──────────┬──────────┬──────────┬──────────┬────────────┐
│   Critère   │  < 10GB  │ 10-100GB │100GB-1TB │  > 1TB   │  > 10TB    │
├─────────────┼──────────┼──────────┼──────────┼──────────┼────────────┤
│ Méthode     │mongodump │mongodump │mongodump │ Snapshot │ Snapshot   │
│ principale  │          │+ oplog   │+ snapshot│+ oplog   │+ streaming │
├─────────────┼──────────┼──────────┼──────────┼──────────┼────────────┤
│ Fréquence   │ 4h       │ 6h       │ 12h      │ 24h      │ 24h        │
│ full        │          │          │          │          │            │
├─────────────┼──────────┼──────────┼──────────┼──────────┼────────────┤
│ Fréquence   │ N/A      │ 2h       │ 4h       │ Continu  │ Continu    │
│ incrémental │          │          │          │          │            │
├─────────────┼──────────┼──────────┼──────────┼──────────┼────────────┤
│ Impact      │ Faible   │ Moyen    │ Moyen-   │ Minimal  │ Minimal    │
│ performance │          │          │ Élevé    │          │            │
├─────────────┼──────────┼──────────┼──────────┼──────────┼────────────┤
│ Complexité  │ Simple   │ Moyenne  │ Élevée   │ Très     │ Expert     │
│             │          │          │          │ Élevée   │            │
└─────────────┴──────────┴──────────┴──────────┴──────────┴────────────┘
```

### Matrice RTO/RPO vs Coût

```
                    RPO (Recovery Point Objective)
                 │
        24h      │  Dev/Test      Standard        Business
                 │  $              $$              $$$
                 │  Daily          Daily+6h        4h snapshots
                 │  mongodump      incremental     + oplog
        ─────────┼─────────────────────────────────────────
         4h      │  Standard       Critical        Mission
                 │  $$             $$$             $$$$
                 │  Daily+4h       Hourly          15min snapshots
                 │  incremental    snapshots       + streaming oplog
        ─────────┼─────────────────────────────────────────
         1h      │  Critical       Mission         Ultra-Critical
                 │  $$$            $$$$            $$$$$
                 │  Hourly         15min           Continuous
                 │  snapshots      snapshots       replication
        ─────────┼─────────────────────────────────────────
        < 15min  │  Mission        Ultra-          Financial/
                 │  $$$$           Critical        Healthcare
                 │  15min          $$$$$           $$$$$$
                 │  snapshots      Continuous      Multi-region
                 │                 + geo-replic    active-active
                 │
                 └────────────────────────────────────────
                      4h          1h         < 15min
                           RTO (Recovery Time Objective)
```

## Stratégies Hybrides

### Approche Multi-Niveaux (Recommandée pour Production)

```yaml
Niveau 1 - Protection Immédiate (RPO: minutes):
  - Replica Set avec 3+ membres
  - Oplog sizing: 72h minimum
  - Read concern: majority
  - Write concern: { w: "majority", j: true }

Niveau 2 - Snapshots Rapides (RPO: 15min-4h):
  méthode: Cloud provider snapshots ou LVM
  fréquence: toutes les 4 heures
  rétention: 48 heures
  restauration: 15-30 minutes

Niveau 3 - Backups Logiques (RPO: 24h):
  méthode: mongodump sur secondary
  fréquence: quotidienne
  rétention: 30 jours
  restauration: 2-4 heures

Niveau 4 - Archives Long Terme (RPO: 1 semaine):
  méthode: mongodump compressé + chiffré
  fréquence: hebdomadaire
  destination: Glacier / Archive tier
  rétention: 1-7 ans
  restauration: 12-48 heures
```

**Orchestrateur multi-niveaux** :
```bash
#!/bin/bash
# hybrid_backup_orchestrator.sh

LEVEL1_MONITOR="/usr/local/bin/monitor_replication.sh"
LEVEL2_SNAPSHOT="/usr/local/bin/create_snapshot.sh"
LEVEL3_LOGICAL="/usr/local/bin/logical_backup.sh"
LEVEL4_ARCHIVE="/usr/local/bin/archive_backup.sh"

# Fonction de monitoring et décision
check_and_execute() {
  local current_hour=$(date +%H)
  local day_of_week=$(date +%u)

  # Niveau 1: Monitoring continu
  $LEVEL1_MONITOR

  # Niveau 2: Snapshots toutes les 4h
  if [ $((current_hour % 4)) -eq 0 ]; then
    echo "Executing Level 2: Snapshot backup"
    $LEVEL2_SNAPSHOT
  fi

  # Niveau 3: Backup logique quotidien à 2h
  if [ "$current_hour" == "02" ]; then
    echo "Executing Level 3: Logical backup"
    $LEVEL3_LOGICAL
  fi

  # Niveau 4: Archive hebdomadaire dimanche à 3h
  if [ "$day_of_week" == "7" ] && [ "$current_hour" == "03" ]; then
    echo "Executing Level 4: Archive backup"
    $LEVEL4_ARCHIVE
  fi
}

# Exécution
check_and_execute

# Logger le résultat
echo "[$(date)] Backup orchestration completed" >> /var/log/backup_orchestrator.log
```

### Stratégie Geo-Distribuée

Pour les applications mondiales nécessitant résilience géographique :

```yaml
Architecture:
  primary_region: us-east-1
    - Replica Set 5 membres
    - Snapshots toutes les 15 min → S3 local

  secondary_region: eu-west-1
    - Replica Set 3 membres (delayed read replica)
    - Reçoit snapshots via S3 cross-region replication

  tertiary_region: ap-southeast-1
    - Archive storage uniquement
    - Reçoit backups hebdomadaires

Processus:
  1. Backup continu dans région primaire
  2. Réplication asynchrone vers région secondaire
  3. Archive mensuelle vers région tertiaire

Avantages:
  - Protection contre catastrophe datacenter
  - Conformité réglementaire (localisation données)
  - Réduction latence de restauration
```

**Configuration S3 cross-region** :
```bash
# Configurer la réplication cross-region
aws s3api put-bucket-replication \
  --bucket company-backups-us-east-1 \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456:role/s3-replication",
    "Rules": [
      {
        "Status": "Enabled",
        "Priority": 1,
        "Filter": { "Prefix": "mongodb/" },
        "Destination": {
          "Bucket": "arn:aws:s3:::company-backups-eu-west-1",
          "ReplicationTime": {
            "Status": "Enabled",
            "Time": { "Minutes": 15 }
          },
          "Metrics": {
            "Status": "Enabled"
          }
        }
      }
    ]
  }'
```

## Stratégies pour Sharded Clusters

Les clusters shardés nécessitent une coordination spéciale :

### Backup Cohérent de Sharded Cluster

```yaml
Défi:
  - Multiples shards indépendants
  - Config servers critiques
  - Cohérence inter-shards

Approche:
  1. Arrêter le balancer
  2. Backup de chaque shard
  3. Backup des config servers
  4. Redémarrer le balancer
```

**Script coordonné** :
```bash
#!/bin/bash
# sharded_cluster_backup.sh

MONGOS_HOST="mongos:27017"
CONFIG_RS="configReplSet/config1:27019,config2:27019,config3:27019"
SHARDS=("shard1/shard1-a:27018" "shard2/shard2-a:27018" "shard3/shard3-a:27018")

BACKUP_DIR="/backup/sharded/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

echo "Starting sharded cluster backup..."

# 1. Arrêter le balancer
echo "Stopping balancer..."
mongo $MONGOS_HOST --eval "sh.stopBalancer()" admin

# Attendre que toutes migrations soient terminées
while true; do
  ACTIVE=$(mongo $MONGOS_HOST --quiet --eval "sh.isBalancerRunning()" admin)
  if [ "$ACTIVE" == "false" ]; then
    break
  fi
  echo "Waiting for balancer to stop..."
  sleep 5
done

echo "Balancer stopped"

# 2. Backup des config servers
echo "Backing up config servers..."
mongodump \
  --host="$CONFIG_RS" \
  --oplog \
  --gzip \
  --out="$BACKUP_DIR/configdb"

# 3. Backup de chaque shard en parallèle
echo "Backing up shards..."
for shard in "${SHARDS[@]}"; do
  shard_name=$(echo $shard | cut -d'/' -f1)
  echo "  Backing up $shard_name..."

  mongodump \
    --host="$shard" \
    --oplog \
    --gzip \
    --out="$BACKUP_DIR/$shard_name" &
done

# Attendre tous les backups
wait

# 4. Sauvegarder métadonnées du cluster
echo "Saving cluster metadata..."
mongo $MONGOS_HOST <<EOF > "$BACKUP_DIR/cluster_metadata.json"
  printjson({
    "config": db.getSiblingDB('config'),
    "shards": sh.status(),
    "databases": db.adminCommand({ listDatabases: 1 }),
    "timestamp": new Date()
  })
EOF

# 5. Redémarrer le balancer
echo "Restarting balancer..."
mongo $MONGOS_HOST --eval "sh.startBalancer()" admin

# 6. Créer archive complète
echo "Creating consolidated archive..."
tar -czf "$BACKUP_DIR.tar.gz" -C "$(dirname $BACKUP_DIR)" "$(basename $BACKUP_DIR)"

# 7. Checksum et métadonnées
sha256sum "$BACKUP_DIR.tar.gz" > "$BACKUP_DIR.tar.gz.sha256"

cat > "$BACKUP_DIR.json" <<EOF
{
  "cluster_type": "sharded",
  "timestamp": "$(date -Iseconds)",
  "shards": ${#SHARDS[@]},
  "size_bytes": $(stat -c%s "$BACKUP_DIR.tar.gz"),
  "balancer_downtime_seconds": $SECONDS
}
EOF

echo "✓ Sharded cluster backup completed"
echo "  Location: $BACKUP_DIR.tar.gz"
echo "  Size: $(du -h $BACKUP_DIR.tar.gz | cut -f1)"
echo "  Balancer downtime: ${SECONDS}s"
```

## Optimisations de Performance

### Parallélisation Intelligente

```bash
#!/bin/bash
# parallel_backup_optimized.sh

# Détecter le nombre de CPU
NCPU=$(nproc)

# Estimer le nombre optimal de threads
# Règle: 1 thread par 50GB de données, max 75% des CPU
DB_SIZE_GB=$(mongo --quiet --eval "
  db.adminCommand({ dbStats: 1, scale: 1024*1024*1024 }).dataSize
")

OPTIMAL_THREADS=$(echo "scale=0; $DB_SIZE_GB / 50" | bc)
MAX_THREADS=$(echo "scale=0; $NCPU * 0.75" | bc)

if [ $OPTIMAL_THREADS -gt $MAX_THREADS ]; then
  THREADS=$MAX_THREADS
else
  THREADS=$OPTIMAL_THREADS
fi

echo "Using $THREADS parallel threads for backup"

mongodump \
  --numParallelCollections=$THREADS \
  --gzip \
  --archive=/backup/optimized_$(date +%Y%m%d).gz
```

### Backup avec Limitation de Bande Passante

```bash
# Limiter l'impact réseau avec trickle ou tc
trickle -s -d 10000 -u 10000 \  # 10 MB/s max
  mongodump --gzip --archive | \
  ssh backup-server "cat > /backup/remote.gz"

# Ou avec rsync throttle
mongodump --gzip --out=/tmp/backup
rsync -avz --bwlimit=10000 /tmp/backup/ backup-server:/backup/
```

### Compression Adaptative

```bash
#!/bin/bash
# adaptive_compression.sh

BACKUP_SIZE_ESTIMATE=$(mongo --quiet --eval "
  db.adminCommand({ dbStats: 1 }).dataSize
")

# Choisir compression selon taille
if [ $BACKUP_SIZE_ESTIMATE -lt $((10 * 1024 * 1024 * 1024)) ]; then
  # < 10GB: compression maximale
  COMPRESSION="zstd -19"
  echo "Small dataset: using maximum compression"
elif [ $BACKUP_SIZE_ESTIMATE -lt $((100 * 1024 * 1024 * 1024)) ]; then
  # 10-100GB: balance
  COMPRESSION="gzip -6"
  echo "Medium dataset: using balanced compression"
else
  # > 100GB: compression rapide
  COMPRESSION="lz4"
  echo "Large dataset: using fast compression"
fi

mongodump --archive | $COMPRESSION > /backup/adaptive_$(date +%Y%m%d).compressed
```

## Checklist de Sélection de Stratégie

Utilisez cette checklist pour définir votre stratégie :

```markdown
### Analyse des Besoins

- [ ] Criticité métier définie (dev/standard/critique/mission-critique)
- [ ] RTO déterminé (4h/1h/15min/<15min)
- [ ] RPO déterminé (24h/4h/1h/<15min)
- [ ] Volume de données estimé (avec croissance 2 ans)
- [ ] Fenêtres de maintenance identifiées
- [ ] Budget alloué (infrastructure + stockage + bande passante)

### Contraintes Techniques

- [ ] Architecture MongoDB documentée (standalone/replica/sharded)
- [ ] Capacité réseau évaluée (bande passante disponible)
- [ ] Stockage disponible calculé (local + remote)
- [ ] Charge système analysée (CPU/RAM/IO pendant backup)
- [ ] Conformité réglementaire vérifiée (RGPD/HIPAA/SOX)

### Sélection de Méthode

- [ ] Méthode principale choisie (mongodump/snapshot/ops manager)
- [ ] Fréquence définie (continue/4h/12h/24h)
- [ ] Rétention planifiée (court/moyen/long terme)
- [ ] Compression sélectionnée (none/gzip/zstd/lz4)
- [ ] Chiffrement configuré (transit + repos)
- [ ] Destination validée (local/S3/Glacier/multi-region)

### Opérations

- [ ] Scripts de backup créés et testés
- [ ] Monitoring configuré (métriques + alertes)
- [ ] Automatisation déployée (cron/kubernetes/ops manager)
- [ ] Procédures de restauration documentées
- [ ] Tests de restauration planifiés
- [ ] DR drill organisé
```

## Recommandations par Profil

### Startup / PME

```yaml
Recommandation: MongoDB Atlas (M10+)
Raison: Backup automatisé intégré, pas de gestion
Coût: 60-200€/mois tout inclus
RTO/RPO: 1h/15min (snapshots point-in-time)
```

### Entreprise Standard

```yaml
Recommandation: Self-hosted + S3
Architecture: Replica Set 3 membres
Backup: mongodump quotidien + oplog 6h
Stockage: 30j S3 Standard-IA + 1an Glacier
Coût: 500-2000€/mois (infra + stockage)
RTO/RPO: 4h/6h
```

### Entreprise Critique

```yaml
Recommandation: Ops Manager ou Atlas Dedicated
Architecture: Replica Set 5+ membres multi-AZ
Backup: Snapshots 15min + oplog streaming
Stockage: Multi-région avec réplication
Coût: 5000-20000€/mois
RTO/RPO: 30min/15min
```

### Enterprise Financial/Healthcare

```yaml
Recommandation: Atlas + Ops Manager hybrid
Architecture: Sharded cluster géo-distribué
Backup: Continuous + snapshots every 15min
Conformité: Chiffrement E2E + audit complet
Stockage: Immutable backups multi-region
Coût: 20000-100000€/mois
RTO/RPO: 15min/1min
```

## Conclusion

Le choix d'une stratégie de sauvegarde MongoDB doit être guidé par :

1. **Les objectifs métier** (RTO/RPO) avant tout
2. **Le volume de données** et sa croissance anticipée
3. **Les contraintes techniques** de l'infrastructure
4. **Le budget disponible** (capex + opex)
5. **Les compétences de l'équipe** pour opérer la solution

Il n'existe pas de solution universelle. Les organisations matures adoptent généralement une **approche multi-niveaux** combinant plusieurs méthodes pour équilibrer coût, performance et fiabilité.

**Principe directeur** : Commencer simple, mesurer, puis optimiser progressivement. Une sauvegarde quotidienne testée vaut mieux qu'un système complexe non maîtrisé.

---


⏭️ [mongodump et mongorestore](/12-sauvegarde-restauration/02-mongodump-mongorestore.md)
