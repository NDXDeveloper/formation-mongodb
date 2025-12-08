🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 12 : Sauvegarde et Restauration

## Vue d'ensemble

La sauvegarde et la restauration constituent un pilier fondamental de toute stratégie de continuité d'activité (Business Continuity) et de reprise après sinistre (Disaster Recovery) pour MongoDB. Dans un environnement de production, la perte de données peut avoir des conséquences catastrophiques : interruption de service, perte de revenus, atteinte à la réputation, sanctions réglementaires, et dans certains cas, la faillite de l'entreprise.

Ce chapitre traite des différentes approches et technologies disponibles pour protéger vos données MongoDB, depuis les sauvegardes logiques traditionnelles jusqu'aux solutions cloud natives avancées, en passant par les snapshots système et la réplication continue via l'oplog.

## Importance Critique de la Sauvegarde

### Objectifs de Récupération

Dans le contexte professionnel, deux métriques définissent vos exigences de sauvegarde :

**RTO (Recovery Time Objective)** : Le temps maximum acceptable pour restaurer le service après un incident.
- RTO de 4 heures : tolérance modérée aux interruptions
- RTO de 1 heure : applications critiques nécessitant haute disponibilité
- RTO de 15 minutes : systèmes mission-critiques (finance, santé)

**RPO (Recovery Point Objective)** : La quantité maximale de données qu'il est acceptable de perdre, mesurée en temps.
- RPO de 24 heures : sauvegardes quotidiennes suffisantes
- RPO de 1 heure : sauvegardes fréquentes ou réplication continue
- RPO proche de zéro : réplication synchrone et transactions distribuées

La relation entre RTO/RPO et le coût est exponentielle : réduire le RTO de 4h à 15 minutes peut multiplier les coûts d'infrastructure par 10 ou plus.

### Scénarios de Sinistre

Les sauvegardes protègent contre divers types de défaillances :

**Erreurs humaines** (60-70% des incidents) :
- Suppression accidentelle de documents ou collections (`db.collection.drop()`)
- Mise à jour incorrecte affectant des milliers de documents
- Exécution de commandes administratives sur la mauvaise base
- Scripts de migration défectueux

**Défaillances matérielles** :
- Corruption du disque ou défaillance du RAID
- Panne complète du datacenter
- Destruction physique (incendie, inondation)

**Attaques malveillantes** :
- Ransomware chiffrant les données
- Suppression intentionnelle par un utilisateur malveillant
- Compromission de la sécurité et corruption des données

**Bugs logiciels** :
- Corruption de données due à un bug dans l'application
- Problèmes de compatibilité lors des mises à jour
- Défaillances du système de fichiers

## Philosophie de Sauvegarde pour MongoDB

### Défense en Profondeur

Une stratégie de sauvegarde robuste repose sur plusieurs couches :

```
┌─────────────────────────────────────────────────┐
│  Niveau 1 : Réplication (Replica Set)           │
│  Protection : Pannes matérielles simples        │
│  RPO : Secondes  |  RTO : Minutes               │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│  Niveau 2 : Sauvegardes Quotidiennes            │
│  Protection : Erreurs humaines récentes         │
│  RPO : 24h  |  RTO : Heures                     │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│  Niveau 3 : Snapshots Fréquents                 │
│  Protection : Point-in-time recovery précis     │
│  RPO : 1-4h  |  RTO : 30min-2h                  │
└─────────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────┐
│  Niveau 4 : Archives Géo-Répliquées             │
│  Protection : Catastrophes datacenter           │
│  RPO : Variable  |  RTO : Heures-Jours          │
└─────────────────────────────────────────────────┘
```

**Principe fondamental** : La réplication n'est PAS une sauvegarde. Elle protège contre les pannes matérielles mais pas contre les erreurs logiques (suppression accidentelle, corruption, attaques).

### Règle du 3-2-1

La règle d'or de la sauvegarde :

- **3 copies** : Une copie primaire + 2 sauvegardes
- **2 supports** : Stockage sur des médias différents (disque, bande, cloud)
- **1 hors site** : Au moins une copie dans un emplacement géographique distant

Exemple d'implémentation pour MongoDB :
```bash
# Copie 1 : Cluster de production (3 nœuds)
# Copie 2 : Snapshots quotidiens sur SAN local
# Copie 3 : Archives dans S3 Glacier (région différente)
```

## Méthodologies de Sauvegarde MongoDB

### Approches Principales

MongoDB offre plusieurs méthodes de sauvegarde, chacune avec ses avantages et compromis :

| Méthode | Cohérence | Performance | Granularité | Flexibilité | Complexité |
|---------|-----------|-------------|-------------|-------------|------------|
| `mongodump` | ✓ Logique | Moyenne | Collection | Haute | Faible |
| Snapshots FS | ✓ Crash-consistent | Haute | Tout ou rien | Faible | Moyenne |
| Atlas Backup | ✓ Point-in-time | Haute | Cluster | Moyenne | Faible |
| Oplog Replay | ✓ Transaction-level | Haute | Précise | Haute | Élevée |
| MongoDB Ops Manager | ✓ Continuous | Haute | Flexible | Très haute | Moyenne |

### Sauvegarde Logique vs Physique

**Sauvegarde Logique** (`mongodump`, exports) :
```javascript
// Avantages
- Portable entre versions et systèmes d'exploitation
- Restauration sélective (collections spécifiques)
- Compression efficace
- Pas besoin d'arrêter le service

// Inconvénients
- Plus lente pour de grands volumes
- Impact sur les performances pendant l'extraction
- Ne capture pas les index (reconstruits lors de la restauration)
```

**Sauvegarde Physique** (snapshots, copies de fichiers) :
```javascript
// Avantages
- Très rapide pour de gros volumes
- Restauration complète et immédiate
- Capture l'état exact du disque
- Minimal overhead si utilisation de snapshots CoW

// Inconvénients
- Moins portable (dépendance OS/version)
- Tout ou rien (pas de granularité)
- Nécessite coordination pour cohérence
```

## Architecture de Sauvegarde

### Composants d'une Solution Complète

```
┌────────────────────────────────────────────────────────────┐
│                    MongoDB Cluster                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │ Primary  │  │Secondary │  │Secondary │                  │
│  │   (P)    │  │   (S1)   │  │   (S2)   │                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
└───────┼─────────────┼─────────────┼────────────────────────┘
        │             │             │
        │             │             └──────┐
        │             │                    │ (Backup Target)
        │             └───────────┐        │
        │                         │        │
        ↓                         ↓        ↓
┌─────────────┐         ┌──────────────────────┐
│  Hot Backup │         │  Backup Coordinator  │
│  (mongodump)│         │   - Orchestration    │
│  S3/Glacier │         │   - Scheduling       │
│             │         │   - Verification     │
└─────────────┘         │   - Retention        │
                        └──────────────────────┘
                                  │
                                  ↓
                        ┌──────────────────────┐
                        │  Backup Repository   │
                        │  - Local Storage     │
                        │  - Remote Storage    │
                        │  - Cloud Storage     │
                        └──────────────────────┘
```

### Planification des Sauvegardes

**Fréquence par Type d'Application** :

```yaml
# Applications critiques (finance, santé)
continuous_backup:
  method: oplog_continuous
  rpo: < 1 minute
  snapshots: every 15 minutes
  full_backup: daily
  retention: 90 days

# Applications standard
standard_backup:
  snapshots: every 4 hours
  full_backup: daily
  incremental: hourly
  retention: 30 days

# Applications développement
dev_backup:
  full_backup: daily
  retention: 7 days
```

## Considérations pour Replica Sets

### Choix du Membre pour la Sauvegarde

**Meilleure pratique** : Effectuer les sauvegardes sur un membre **Secondary** :

```javascript
// Vérifier le statut du replica set
rs.status()

// Identifier un secondary approprié
{
  members: [
    { _id: 0, name: "mongo-primary:27017", stateStr: "PRIMARY" },
    { _id: 1, name: "mongo-secondary-1:27017", stateStr: "SECONDARY" },
    { _id: 2, name: "mongo-secondary-2:27017", stateStr: "SECONDARY",
      priority: 0, hidden: true, tags: { backup: "true" } }  // ← Membre de backup
  ]
}
```

**Configuration d'un membre dédié aux sauvegardes** :
```javascript
// Ajouter un membre caché avec priorité 0
cfg = rs.conf()
cfg.members.push({
  _id: 3,
  host: "mongo-backup:27017",
  priority: 0,        // Ne peut jamais devenir Primary
  hidden: true,       // Invisible pour les clients
  votes: 0,           // Ne participe pas aux élections
  tags: { backup: "true", workload: "analytics" }
})
rs.reconfig(cfg)
```

### Impact sur la Réplication

```javascript
// Monitoring du lag de réplication pendant la sauvegarde
db.adminCommand({ replSetGetStatus: 1 }).members.forEach(member => {
  if (member.stateStr === "SECONDARY") {
    print(`${member.name}: lag = ${member.optimeDate - member.lastHeartbeat}ms`)
  }
})

// Ajuster les paramètres si nécessaire
db.adminCommand({
  setParameter: 1,
  replWriterThreadCount: 32  // Augmenter si lag excessif
})
```

## Cohérence et Intégrité

### Garanties de Cohérence

MongoDB garantit différents niveaux de cohérence selon la méthode :

**mongodump avec --oplog** :
```bash
# Capture un point de cohérence spécifique
mongodump --oplog --gzip --archive=/backup/mongo_$(date +%Y%m%d).gz

# Le flag --oplog enregistre l'état de l'oplog au début
# Permet de rejouer les opérations pour cohérence point-in-time
```

**Snapshots filesystem** :
```bash
# Nécessite coordination pour cohérence
# 1. Flusher les écritures en mémoire
mongo --eval "db.fsyncLock()"

# 2. Créer le snapshot
lvcreate --size 10G --snapshot --name mongo-snap /dev/vg0/mongo-lv

# 3. Débloquer
mongo --eval "db.fsyncUnlock()"

# 4. Monter et copier
mount /dev/vg0/mongo-snap /mnt/backup
rsync -av /mnt/backup/ /backup/mongo-snapshot-$(date +%Y%m%d)/
```

### Vérification d'Intégrité

**Validation post-sauvegarde** :
```bash
#!/bin/bash
# validate_backup.sh

BACKUP_PATH="/backup/mongo_20241208.gz"
TEMP_RESTORE="/tmp/validate_restore"

# Restauration test dans environnement isolé
mkdir -p $TEMP_RESTORE
mongorestore --gzip --archive=$BACKUP_PATH --nsInclude="testdb.*" \
  --port=27018 --drop

# Vérification
mongo --port=27018 --eval "
  db = db.getSiblingDB('testdb');
  stats = db.stats();
  print('Collections: ' + stats.collections);
  print('Objects: ' + stats.objects);
  print('Data Size: ' + stats.dataSize);

  // Validation de la cohérence
  db.getCollectionNames().forEach(coll => {
    result = db[coll].validate({ full: true });
    if (!result.valid) {
      print('ERROR: Collection ' + coll + ' is invalid');
      quit(1);
    }
  });
  print('✓ Backup validation successful');
"

# Checksum pour intégrité
sha256sum $BACKUP_PATH > ${BACKUP_PATH}.sha256
```

## Compression et Chiffrement

### Stratégies de Compression

```bash
# mongodump avec compression gzip (niveau 6 par défaut)
mongodump --gzip --archive=/backup/db.gz

# Compression custom avec meilleure ratio
mongodump --archive | zstd -19 > /backup/db.zst

# Comparaison des ratios (base 100 GB)
gzip -6:    25 GB (75% compression, 10 min)
gzip -9:    22 GB (78% compression, 18 min)
zstd -19:   18 GB (82% compression, 12 min)
lz4:        35 GB (65% compression, 3 min)
```

### Chiffrement des Sauvegardes

**Chiffrement côté application** :
```bash
# Avec GPG
mongodump --archive | gzip | gpg --encrypt --recipient backup@company.com \
  > /backup/mongo_encrypted_$(date +%Y%m%d).gz.gpg

# Avec OpenSSL
mongodump --archive | gzip | openssl enc -aes-256-cbc -salt \
  -out /backup/mongo_$(date +%Y%m%d).gz.enc -pass file:/secrets/backup.key
```

**Chiffrement côté stockage** :
```bash
# AWS S3 avec KMS
aws s3 cp /backup/mongo.gz s3://company-backups/ \
  --sse aws:kms \
  --sse-kms-key-id arn:aws:kms:us-east-1:123456:key/abc-def

# Azure Blob avec encryption
az storage blob upload \
  --account-name companybackups \
  --container backups \
  --name mongo_$(date +%Y%m%d).gz \
  --file /backup/mongo.gz \
  --encryption-scope company-backup-encryption
```

## Rétention et Cycle de Vie

### Politique de Rétention Type

```yaml
# Schéma Grand-Père-Père-Fils (GFS)
retention_policy:
  hourly:
    frequency: every_hour
    keep: 24  # Dernières 24 heures

  daily:
    frequency: midnight
    keep: 30  # Dernier mois (30 jours)

  weekly:
    frequency: sunday_midnight
    keep: 12  # Dernier trimestre (12 semaines)

  monthly:
    frequency: first_of_month
    keep: 12  # Dernière année (12 mois)

  yearly:
    frequency: january_first
    keep: 7   # 7 ans (conformité réglementaire)
```

**Implémentation avec lifecycle policies** :
```bash
# AWS S3 Lifecycle
cat > lifecycle.json <<EOF
{
  "Rules": [
    {
      "Id": "MongoBackupLifecycle",
      "Status": "Enabled",
      "Filter": { "Prefix": "mongodb/" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER"
        }
      ],
      "Expiration": {
        "Days": 2555  // 7 ans
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket company-backups \
  --lifecycle-configuration file://lifecycle.json
```

## Monitoring et Alerting

### Métriques Clés à Surveiller

```javascript
// Script de monitoring des sauvegardes
// monitoring_backups.js

const fs = require('fs');
const path = require('path');

const BACKUP_DIR = '/backup/mongodb';
const MAX_AGE_HOURS = 26;  // Alert si > 26h sans backup
const MIN_SIZE_MB = 100;   // Alert si backup < 100 MB

function checkBackups() {
  const now = Date.now();
  const files = fs.readdirSync(BACKUP_DIR)
    .filter(f => f.endsWith('.gz'))
    .map(f => {
      const filepath = path.join(BACKUP_DIR, f);
      const stats = fs.statSync(filepath);
      return {
        name: f,
        age: (now - stats.mtimeMs) / 3600000,  // heures
        size: stats.size / (1024 * 1024)        // MB
      };
    });

  // Backup le plus récent
  const latest = files.sort((a, b) => a.age - b.age)[0];

  if (!latest) {
    console.error('CRITICAL: No backup found');
    process.exit(2);
  }

  if (latest.age > MAX_AGE_HOURS) {
    console.error(`CRITICAL: Latest backup is ${latest.age.toFixed(1)}h old`);
    process.exit(2);
  }

  if (latest.size < MIN_SIZE_MB) {
    console.error(`WARNING: Backup size ${latest.size.toFixed(1)}MB is suspiciously small`);
    process.exit(1);
  }

  console.log(`OK: Latest backup ${latest.name} (${latest.age.toFixed(1)}h old, ${latest.size.toFixed(1)}MB)`);
  process.exit(0);
}

checkBackups();
```

### Alertes Prometheus

```yaml
# prometheus_rules.yml
groups:
  - name: mongodb_backup
    interval: 5m
    rules:
      - alert: MongoBackupTooOld
        expr: (time() - mongodb_backup_last_success_timestamp) > 93600
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB backup is too old"
          description: "Last successful backup was {{ $value | humanizeDuration }} ago"

      - alert: MongoBackupFailed
        expr: mongodb_backup_last_status != 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "MongoDB backup failed"
          description: "Last backup failed with status {{ $value }}"

      - alert: MongoBackupSizeAnomaly
        expr: |
          abs(mongodb_backup_size_bytes -
              avg_over_time(mongodb_backup_size_bytes[7d])) /
          stddev_over_time(mongodb_backup_size_bytes[7d]) > 3
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "MongoDB backup size is anomalous"
          description: "Backup size deviates by more than 3 standard deviations"
```

## Considérations Légales et de Conformité

### Réglementations Communes

**RGPD (Europe)** :
- Droit à l'oubli : capacité de supprimer les données d'un individu dans les backups
- Notification de violation : 72h pour signaler une fuite de données
- Principe de minimisation : ne conserver que le nécessaire

**HIPAA (Santé US)** :
- Chiffrement obligatoire des backups
- Journaux d'audit d'accès aux backups
- Procédures documentées de récupération

**SOX (Finance)** :
- Rétention de 7 ans pour les données financières
- Immuabilité des backups (WORM - Write Once Read Many)
- Séparation des responsabilités

**Implémentation technique** :
```bash
# Backups immuables avec S3 Object Lock
aws s3api put-object-lock-configuration \
  --bucket compliance-backups \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Years": 7
      }
    }
  }'
```

## Tests de Restauration

### Importance des Tests Réguliers

> "Une sauvegarde non testée n'est pas une sauvegarde"

**Fréquence recommandée** :
- Tests complets : Trimestriellement
- Tests partiels : Mensuellement
- Drill de catastrophe : Annuellement

**Procédure de test** :
```bash
#!/bin/bash
# disaster_recovery_drill.sh

set -e

LOG_FILE="/var/log/dr_drill_$(date +%Y%m%d_%H%M%S).log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

echo "=== Disaster Recovery Drill Started ==="
echo "Timestamp: $(date)"

# 1. Identifier le backup à tester
BACKUP_FILE=$(ls -t /backup/mongo_*.gz | head -1)
echo "Testing backup: $BACKUP_FILE"

# 2. Environnement de test isolé
TEST_PORT=27099
TEST_DBPATH=/tmp/dr_test_$(date +%s)
mkdir -p $TEST_DBPATH

# 3. Démarrer mongod de test
mongod --port $TEST_PORT --dbpath $TEST_DBPATH --fork \
  --logpath $TEST_DBPATH/mongod.log

# 4. Restauration
echo "Restoring backup..."
START_TIME=$(date +%s)
mongorestore --port $TEST_PORT --gzip --archive=$BACKUP_FILE

END_TIME=$(date +%s)
RESTORE_DURATION=$((END_TIME - START_TIME))

# 5. Validation
echo "Validating restore..."
mongo --port $TEST_PORT --eval "
  dbs = db.adminCommand('listDatabases').databases;
  totalSize = 0;
  dbs.forEach(function(database) {
    dbName = database.name;
    if (dbName !== 'admin' && dbName !== 'local' && dbName !== 'config') {
      db = db.getSiblingDB(dbName);
      stats = db.stats();
      totalSize += stats.dataSize;
      print('✓ Database: ' + dbName +
            ' - Collections: ' + stats.collections +
            ' - Objects: ' + stats.objects);
    }
  });
  print('Total data size: ' + (totalSize / 1024 / 1024 / 1024).toFixed(2) + ' GB');
"

# 6. Tests fonctionnels
echo "Running functional tests..."
mongo --port $TEST_PORT <<EOF
  use production_db
  // Test 1: Vérifier une collection critique
  count = db.orders.countDocuments()
  print('Orders count: ' + count)
  assert(count > 0, 'No orders found')

  // Test 2: Vérifier l'intégrité des index
  indexes = db.orders.getIndexes()
  print('Indexes: ' + indexes.length)
  assert(indexes.length >= 3, 'Missing indexes')

  // Test 3: Requête de test
  result = db.orders.findOne({ status: 'completed' })
  assert(result !== null, 'No completed order found')

  print('✓ All functional tests passed')
EOF

# 7. Rapport
echo "=== Drill Results ==="
echo "Restore Duration: ${RESTORE_DURATION}s"
echo "RTO Achieved: $(date -u -d @${RESTORE_DURATION} +%H:%M:%S)"
echo "Status: SUCCESS"

# 8. Cleanup
mongod --port $TEST_PORT --dbpath $TEST_DBPATH --shutdown
rm -rf $TEST_DBPATH

echo "=== Disaster Recovery Drill Completed ==="
```

## Automatisation et Orchestration

### Script de Sauvegarde Complet

```bash
#!/bin/bash
# mongodb_backup_orchestrator.sh

# Configuration
BACKUP_ROOT="/backup/mongodb"
RETENTION_DAYS=30
S3_BUCKET="s3://company-backups/mongodb"
MONGO_URI="mongodb://backup-user:password@mongo-secondary:27017/?authSource=admin"
SLACK_WEBHOOK="https://hooks.slack.com/services/YOUR/WEBHOOK"

# Functions
log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a /var/log/mongo_backup.log
}

notify_slack() {
  local message=$1
  local color=$2
  curl -X POST $SLACK_WEBHOOK -H 'Content-Type: application/json' -d '{
    "attachments": [{
      "color": "'"$color"'",
      "text": "'"$message"'",
      "footer": "MongoDB Backup",
      "ts": '$(date +%s)'
    }]
  }'
}

cleanup_old_backups() {
  log "Cleaning up backups older than $RETENTION_DAYS days"
  find $BACKUP_ROOT -name "mongo_*.gz" -mtime +$RETENTION_DAYS -delete
}

# Main backup process
main() {
  local timestamp=$(date +%Y%m%d_%H%M%S)
  local backup_file="$BACKUP_ROOT/mongo_${timestamp}.gz"
  local start_time=$(date +%s)

  log "Starting MongoDB backup"

  # Pre-backup checks
  if ! mongo $MONGO_URI --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log "ERROR: Cannot connect to MongoDB"
    notify_slack "❌ Backup failed: Cannot connect to MongoDB" "danger"
    exit 1
  fi

  # Execute backup
  if mongodump --uri="$MONGO_URI" --oplog --gzip --archive="$backup_file"; then
    local end_time=$(date +%s)
    local duration=$((end_time - start_time))
    local size=$(du -h "$backup_file" | cut -f1)

    log "Backup completed successfully in ${duration}s, size: $size"

    # Upload to S3
    log "Uploading to S3"
    if aws s3 cp "$backup_file" "$S3_BUCKET/" --storage-class STANDARD_IA; then
      log "S3 upload successful"

      # Generate checksum
      sha256sum "$backup_file" > "${backup_file}.sha256"

      # Cleanup old backups
      cleanup_old_backups

      notify_slack "✅ Backup successful: $size in ${duration}s" "good"
      exit 0
    else
      log "ERROR: S3 upload failed"
      notify_slack "⚠️ Backup created but S3 upload failed" "warning"
      exit 1
    fi
  else
    log "ERROR: Backup failed"
    notify_slack "❌ Backup failed" "danger"
    exit 1
  fi
}

# Execute
main
```

### Intégration avec Kubernetes CronJob

```yaml
# mongodb-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: database
spec:
  schedule: "0 2 * * *"  # 2h du matin tous les jours
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: mongodb-backup
          containers:
          - name: backup
            image: mongo:7.0
            command:
            - /bin/bash
            - -c
            - |
              set -e
              BACKUP_NAME="mongo_$(date +%Y%m%d_%H%M%S).gz"

              mongodump \
                --uri="mongodb://mongo-0.mongo:27017,mongo-1.mongo:27017,mongo-2.mongo:27017/?replicaSet=rs0" \
                --username=$MONGO_USER \
                --password=$MONGO_PASSWORD \
                --authenticationDatabase=admin \
                --oplog \
                --gzip \
                --archive=/backup/$BACKUP_NAME

              # Upload vers S3
              aws s3 cp /backup/$BACKUP_NAME s3://$S3_BUCKET/

              echo "Backup completed: $BACKUP_NAME"
            env:
            - name: MONGO_USER
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-secret
                  key: username
            - name: MONGO_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-secret
                  key: password
            - name: S3_BUCKET
              value: "company-backups/mongodb"
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
                memory: "2Gi"
                cpu: "1"
              limits:
                memory: "4Gi"
                cpu: "2"
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: mongodb-backup-pvc
```

## Conclusion du Chapitre

La sauvegarde et la restauration ne sont pas simplement des tâches techniques, mais des piliers stratégiques de la continuité d'activité. Une stratégie de sauvegarde efficace pour MongoDB nécessite :

1. **Compréhension des besoins** : RTO/RPO adaptés à la criticité
2. **Défense en profondeur** : Multiples couches de protection
3. **Automatisation complète** : Scripts robustes et monitoring
4. **Tests réguliers** : Validation continue de la recouvrabilité
5. **Conformité** : Respect des obligations légales et réglementaires

Les sections suivantes de ce chapitre détailleront chaque méthode de sauvegarde, leurs implémentations spécifiques pour les différentes topologies MongoDB (standalone, replica sets, sharded clusters), et les stratégies avancées comme le Point-in-Time Recovery.

---

**Points clés à retenir** :
- La réplication ≠ sauvegarde (protection différente)
- Règle 3-2-1 pour la redondance géographique
- Tests de restauration obligatoires et réguliers
- Chiffrement et monitoring sont critiques
- Automatisation pour fiabilité et réduction des erreurs humaines


⏭️ [Stratégies de sauvegarde](/12-sauvegarde-restauration/01-strategies-sauvegarde.md)
