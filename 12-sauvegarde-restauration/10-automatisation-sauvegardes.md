🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.10 Automatisation des Sauvegardes

## Introduction

L'automatisation des sauvegardes est un pilier essentiel de toute stratégie de continuité d'activité robuste. Les backups manuels sont sujets aux erreurs humaines, aux oublis et aux incohérences. Une automatisation bien conçue garantit des sauvegardes régulières, testées, validées et stockées de manière fiable, tout en minimisant l'intervention humaine et en maximisant la résilience.

Cette section explore les stratégies, outils et bonnes pratiques pour automatiser complètement l'écosystème de sauvegarde MongoDB, depuis la planification jusqu'au monitoring, en passant par la validation et la gestion du cycle de vie des backups.

## Architecture d'Automatisation

### Vue d'Ensemble

```
┌────────────────────────────────────────────────────────────┐
│            Architecture d'Automatisation Complète          │
│                                                            │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Orchestrateur (Scheduling)                 │    │
│  │  • Cron / Systemd Timers                           │    │
│  │  • Kubernetes CronJobs                             │    │
│  │  • Airflow / Luigi                                 │    │
│  │  • AWS EventBridge                                 │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │ Trigger                              │
│                     v                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Scripts de Backup                          │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐    │    │
│  │  │ Pre-checks  │→ │   Backup    │→ │Validation│    │    │
│  │  └─────────────┘  └─────────────┘  └──────────┘    │    │
│  │         │                │                │        │    │
│  │         └────────────────┴────────────────┘        │    │
│  │                          │                         │    │
│  └──────────────────────────┼─────────────────────────┘    │
│                             │                              │
│                             v                              │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Post-Processing                            │    │
│  │  • Compression                                     │    │
│  │  • Encryption                                      │    │
│  │  • Checksum                                        │    │
│  │  • Metadata                                        │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                      │
│                     v                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Stockage Distant                           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐          │    │
│  │  │   S3     │  │  Azure   │  │   GCS    │          │    │
│  │  └──────────┘  └──────────┘  └──────────┘          │    │
│  │  ┌──────────┐  ┌──────────┐                        │    │
│  │  │   NFS    │  │   SFTP   │                        │    │
│  │  └──────────┘  └──────────┘                        │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                      │
│                     v                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Lifecycle Management                       │    │
│  │  • Rotation (GFS)                                  │    │
│  │  • Archivage                                       │    │
│  │  • Cleanup                                         │    │
│  └──────────────────┬─────────────────────────────────┘    │
│                     │                                      │
│                     v                                      │
│  ┌────────────────────────────────────────────────────┐    │
│  │         Monitoring & Alerting                      │    │
│  │  • Success/Failure tracking                        │    │
│  │  • Duration metrics                                │    │
│  │  • Size metrics                                    │    │
│  │  • Alerts (Slack, PagerDuty, Email)                │    │
│  │  • Dashboards (Grafana)                            │    │
│  └────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────┘
```

## Script Maître d'Automatisation

### Script Complet avec Toutes les Fonctionnalités

```bash
#!/bin/bash
# mongodb_automated_backup.sh
# Production-ready automated backup solution

set -euo pipefail

# ============================================================================
# CONFIGURATION
# ============================================================================

# Environnement
ENVIRONMENT="${BACKUP_ENV:-production}"
SCRIPT_VERSION="2.0.0"

# MongoDB
MONGO_URI="${MONGO_URI:-mongodb://localhost:27017}"
MONGO_USER="${MONGO_USER:-backup-user}"
MONGO_PASS="${MONGO_PASS}"  # Depuis variable d'environnement ou secrets
MONGO_AUTH_DB="${MONGO_AUTH_DB:-admin}"

# Backup
BACKUP_ROOT="${BACKUP_ROOT:-/backup/mongodb}"
BACKUP_TYPE="${BACKUP_TYPE:-full}"  # full|incremental|differential
COMPRESSION="${COMPRESSION:-gzip}"  # gzip|zstd|lz4|none
PARALLEL_COLLECTIONS="${PARALLEL_COLLECTIONS:-4}"

# Rétention (jours)
RETENTION_DAILY=7
RETENTION_WEEKLY=28
RETENTION_MONTHLY=180
RETENTION_YEARLY=2555  # 7 ans

# Stockage distant
REMOTE_STORAGE="${REMOTE_STORAGE:-s3}"  # s3|azure|gcs|sftp|none
S3_BUCKET="${S3_BUCKET:-}"
S3_PREFIX="${S3_PREFIX:-mongodb-backups}"
S3_STORAGE_CLASS="${S3_STORAGE_CLASS:-STANDARD_IA}"

# Notifications
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"
PAGERDUTY_KEY="${PAGERDUTY_KEY:-}"
EMAIL_RECIPIENTS="${EMAIL_RECIPIENTS:-}"

# Monitoring
PROMETHEUS_PUSHGATEWAY="${PROMETHEUS_PUSHGATEWAY:-}"
DATADOG_API_KEY="${DATADOG_API_KEY:-}"

# Limites
MAX_BACKUP_DURATION=14400  # 4 heures
MIN_FREE_SPACE_GB=100
MAX_RETRIES=3

# ============================================================================
# LOGGING ET UTILITIES
# ============================================================================

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_ID="${ENVIRONMENT}_${TIMESTAMP}"
LOG_FILE="${BACKUP_ROOT}/logs/backup_${BACKUP_ID}.log"
METRICS_FILE="${BACKUP_ROOT}/metrics/backup_${BACKUP_ID}.json"

mkdir -p "$(dirname $LOG_FILE)"
mkdir -p "$(dirname $METRICS_FILE)"

# Logging avec niveaux
log() {
  local level=$1
  shift
  local message="$@"
  local timestamp=$(date +'%Y-%m-%d %H:%M:%S')

  echo "[$timestamp] [$level] $message" | tee -a "$LOG_FILE"

  # Log vers syslog aussi
  logger -t mongodb-backup -p "user.$level" "$message"
}

log_info() { log "INFO" "$@"; }
log_warn() { log "WARN" "$@"; }
log_error() { log "ERROR" "$@"; }
log_debug() { [ "${DEBUG:-0}" = "1" ] && log "DEBUG" "$@" || true; }

# Métriques
START_TIME=$(date +%s)
METRICS=()

add_metric() {
  local name=$1
  local value=$2
  local type=${3:-gauge}

  METRICS+=("{\"name\":\"$name\",\"value\":$value,\"type\":\"$type\",\"timestamp\":$(date +%s)}")
}

# Gestion des erreurs
handle_error() {
  local exit_code=$?
  local line_number=$1

  log_error "Script failed at line $line_number with exit code $exit_code"

  # Métriques d'échec
  add_metric "mongodb_backup_failed" 1

  # Notifications
  send_failure_notification

  # Cleanup partiel
  cleanup_on_failure

  exit $exit_code
}

trap 'handle_error ${LINENO}' ERR

# Timeout pour éviter les backups bloqués
timeout_guard() {
  local timeout=$1
  shift

  timeout --signal=TERM --kill-after=60 "$timeout" "$@"
}

# ============================================================================
# PRÉ-VÉRIFICATIONS
# ============================================================================

pre_checks() {
  log_info "=========================================="
  log_info "  MongoDB Automated Backup"
  log_info "=========================================="
  log_info "Version: $SCRIPT_VERSION"
  log_info "Environment: $ENVIRONMENT"
  log_info "Backup ID: $BACKUP_ID"
  log_info "Backup Type: $BACKUP_TYPE"
  log_info ""

  # Vérifier les commandes requises
  local required_commands=("mongodump" "mongo" "jq" "date" "bc")

  for cmd in "${required_commands[@]}"; do
    if ! command -v "$cmd" >/dev/null 2>&1; then
      log_error "Required command not found: $cmd"
      exit 1
    fi
  done

  log_info "✓ Required commands available"

  # Vérifier la connectivité MongoDB
  log_info "Checking MongoDB connectivity..."

  if ! timeout_guard 30 mongo "$MONGO_URI/$MONGO_AUTH_DB" \
    --username "$MONGO_USER" \
    --password "$MONGO_PASS" \
    --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log_error "Cannot connect to MongoDB"
    exit 1
  fi

  log_info "✓ MongoDB connection successful"

  # Vérifier l'espace disque
  log_info "Checking disk space..."

  local available_gb=$(df --output=avail -B1G "$BACKUP_ROOT" | tail -1)

  if [ "$available_gb" -lt "$MIN_FREE_SPACE_GB" ]; then
    log_error "Insufficient disk space: ${available_gb}GB available, ${MIN_FREE_SPACE_GB}GB required"
    exit 1
  fi

  log_info "✓ Sufficient disk space: ${available_gb}GB available"

  # Vérifier qu'aucun backup n'est en cours
  if [ -f "${BACKUP_ROOT}/.backup_in_progress" ]; then
    local pid=$(cat "${BACKUP_ROOT}/.backup_in_progress")

    if ps -p "$pid" >/dev/null 2>&1; then
      log_error "Another backup is already running (PID: $pid)"
      exit 1
    else
      log_warn "Stale lock file found, removing..."
      rm -f "${BACKUP_ROOT}/.backup_in_progress"
    fi
  fi

  # Créer le lock file
  echo $$ > "${BACKUP_ROOT}/.backup_in_progress"

  log_info "✓ Pre-checks completed"
}

# ============================================================================
# BACKUP PRINCIPAL
# ============================================================================

perform_backup() {
  log_info "=== Starting Backup ==="

  local backup_dir="${BACKUP_ROOT}/backups/${BACKUP_ID}"
  mkdir -p "$backup_dir"

  local backup_start=$(date +%s)

  # Options de mongodump selon le type
  local dump_opts=(
    "--uri=$MONGO_URI/$MONGO_AUTH_DB"
    "--username=$MONGO_USER"
    "--password=$MONGO_PASS"
    "--oplog"
    "--numParallelCollections=$PARALLEL_COLLECTIONS"
    "--out=$backup_dir"
  )

  # Compression
  case "$COMPRESSION" in
    gzip)
      dump_opts+=("--gzip")
      ;;
    none)
      # Pas de compression
      ;;
    *)
      log_warn "Unsupported compression: $COMPRESSION, using gzip"
      dump_opts+=("--gzip")
      ;;
  esac

  # Exécuter le backup avec timeout
  log_info "Running mongodump..."
  log_info "Command: mongodump ${dump_opts[*]}"

  if timeout_guard "$MAX_BACKUP_DURATION" mongodump "${dump_opts[@]}" \
    2>&1 | tee -a "$LOG_FILE"; then

    local backup_end=$(date +%s)
    local duration=$((backup_end - backup_start))

    log_info "✓ Backup completed in ${duration}s"

    # Métriques
    add_metric "mongodb_backup_duration_seconds" "$duration"
    add_metric "mongodb_backup_success" 1

  else
    log_error "Backup failed"
    add_metric "mongodb_backup_failed" 1
    return 1
  fi

  # Taille du backup
  local backup_size=$(du -sb "$backup_dir" | cut -f1)
  local backup_size_mb=$(echo "scale=2; $backup_size / 1024 / 1024" | bc)

  log_info "Backup size: ${backup_size_mb} MB"
  add_metric "mongodb_backup_size_bytes" "$backup_size"

  echo "$backup_dir"
}

# ============================================================================
# POST-PROCESSING
# ============================================================================

post_process_backup() {
  local backup_dir=$1

  log_info "=== Post-Processing ==="

  # Générer checksums
  log_info "Generating checksums..."
  find "$backup_dir" -type f -exec sha256sum {} \; > "${backup_dir}/SHA256SUMS"

  # Créer le manifeste
  log_info "Creating manifest..."

  cat > "${backup_dir}/MANIFEST.json" <<EOF
{
  "backup_id": "$BACKUP_ID",
  "environment": "$ENVIRONMENT",
  "timestamp": "$(date -Iseconds)",
  "backup_type": "$BACKUP_TYPE",
  "mongodb_uri": "$MONGO_URI",
  "mongodb_version": "$(mongo --quiet --eval 'db.version()')",
  "compression": "$COMPRESSION",
  "backup_size_bytes": $(du -sb "$backup_dir" | cut -f1),
  "files": $(find "$backup_dir" -type f | wc -l),
  "script_version": "$SCRIPT_VERSION"
}
EOF

  # Compression supplémentaire si demandé
  if [ "${CREATE_ARCHIVE:-true}" = "true" ]; then
    log_info "Creating archive..."

    local archive_file="${backup_dir}.tar.gz"
    tar -czf "$archive_file" -C "$(dirname $backup_dir)" "$(basename $backup_dir)"

    # Checksum de l'archive
    sha256sum "$archive_file" > "${archive_file}.sha256"

    log_info "✓ Archive created: $(basename $archive_file)"
    log_info "  Size: $(du -h $archive_file | cut -f1)"
  fi

  log_info "✓ Post-processing completed"
}

# ============================================================================
# UPLOAD VERS STOCKAGE DISTANT
# ============================================================================

upload_to_remote() {
  local backup_dir=$1

  if [ "$REMOTE_STORAGE" = "none" ]; then
    log_info "Remote storage disabled, skipping upload"
    return 0
  fi

  log_info "=== Uploading to Remote Storage ==="
  log_info "Destination: $REMOTE_STORAGE"

  local upload_start=$(date +%s)

  case "$REMOTE_STORAGE" in
    s3)
      upload_to_s3 "$backup_dir"
      ;;
    azure)
      upload_to_azure "$backup_dir"
      ;;
    gcs)
      upload_to_gcs "$backup_dir"
      ;;
    sftp)
      upload_to_sftp "$backup_dir"
      ;;
    *)
      log_error "Unknown remote storage: $REMOTE_STORAGE"
      return 1
      ;;
  esac

  local upload_end=$(date +%s)
  local upload_duration=$((upload_end - upload_start))

  log_info "✓ Upload completed in ${upload_duration}s"
  add_metric "mongodb_backup_upload_duration_seconds" "$upload_duration"
}

upload_to_s3() {
  local backup_dir=$1

  if [ -z "$S3_BUCKET" ]; then
    log_error "S3_BUCKET not configured"
    return 1
  fi

  local s3_path="s3://${S3_BUCKET}/${S3_PREFIX}/${ENVIRONMENT}/$(basename $backup_dir)"

  log_info "Uploading to: $s3_path"

  # Upload avec aws cli
  aws s3 sync "$backup_dir" "$s3_path" \
    --storage-class "$S3_STORAGE_CLASS" \
    --only-show-errors

  # Upload de l'archive si elle existe
  if [ -f "${backup_dir}.tar.gz" ]; then
    aws s3 cp "${backup_dir}.tar.gz" "${s3_path}.tar.gz" \
      --storage-class "$S3_STORAGE_CLASS"

    aws s3 cp "${backup_dir}.tar.gz.sha256" "${s3_path}.tar.gz.sha256" \
      --storage-class "$S3_STORAGE_CLASS"
  fi

  log_info "✓ Uploaded to S3"
}

upload_to_azure() {
  local backup_dir=$1

  # Implémentation Azure Blob Storage
  log_info "Azure upload not implemented"
  # az storage blob upload-batch ...
}

upload_to_gcs() {
  local backup_dir=$1

  # Implémentation Google Cloud Storage
  log_info "GCS upload not implemented"
  # gsutil -m rsync -r ...
}

upload_to_sftp() {
  local backup_dir=$1

  # Implémentation SFTP
  log_info "SFTP upload not implemented"
  # rsync via sftp ...
}

# ============================================================================
# ROTATION ET CLEANUP
# ============================================================================

rotate_backups() {
  log_info "=== Rotating Backups (GFS) ==="

  local backups_dir="${BACKUP_ROOT}/backups"

  # Stratégie Grandfather-Father-Son (GFS)

  # Daily: garder les N derniers jours
  log_info "Rotating daily backups (keep last $RETENTION_DAILY days)..."
  find "$backups_dir" -maxdepth 1 -name "${ENVIRONMENT}_*" -type d -mtime +$RETENTION_DAILY \
    ! -name "*_weekly_*" ! -name "*_monthly_*" ! -name "*_yearly_*" \
    -exec rm -rf {} \;

  # Weekly: promouvoir un backup hebdomadaire
  local day_of_week=$(date +%u)  # 1=Monday, 7=Sunday

  if [ "$day_of_week" = "7" ]; then  # Dimanche
    log_info "Promoting to weekly backup..."

    local latest_backup=$(find "$backups_dir" -maxdepth 1 -name "${ENVIRONMENT}_*" -type d | sort | tail -1)
    if [ -n "$latest_backup" ]; then
      local weekly_name="${latest_backup}_weekly_$(date +%Y%W)"
      ln -sf "$(basename $latest_backup)" "$weekly_name"
      log_info "✓ Weekly backup created: $(basename $weekly_name)"
    fi
  fi

  # Cleanup weekly backups
  find "$backups_dir" -maxdepth 1 -name "*_weekly_*" -type l -mtime +$RETENTION_WEEKLY -delete

  # Monthly: promouvoir un backup mensuel
  local day_of_month=$(date +%d)

  if [ "$day_of_month" = "01" ]; then  # Premier du mois
    log_info "Promoting to monthly backup..."

    local latest_backup=$(find "$backups_dir" -maxdepth 1 -name "${ENVIRONMENT}_*" -type d | sort | tail -1)
    if [ -n "$latest_backup" ]; then
      local monthly_name="${latest_backup}_monthly_$(date +%Y%m)"
      ln -sf "$(basename $latest_backup)" "$monthly_name"
      log_info "✓ Monthly backup created: $(basename $monthly_name)"
    fi
  fi

  # Cleanup monthly backups
  find "$backups_dir" -maxdepth 1 -name "*_monthly_*" -type l -mtime +$RETENTION_MONTHLY -delete

  # Yearly: promouvoir un backup annuel
  local day_of_year=$(date +%j)

  if [ "$day_of_year" = "001" ]; then  # 1er janvier
    log_info "Promoting to yearly backup..."

    local latest_backup=$(find "$backups_dir" -maxdepth 1 -name "${ENVIRONMENT}_*" -type d | sort | tail -1)
    if [ -n "$latest_backup" ]; then
      local yearly_name="${latest_backup}_yearly_$(date +%Y)"
      ln -sf "$(basename $latest_backup)" "$yearly_name"
      log_info "✓ Yearly backup created: $(basename $yearly_name)"
    fi
  fi

  # Cleanup yearly backups
  find "$backups_dir" -maxdepth 1 -name "*_yearly_*" -type l -mtime +$RETENTION_YEARLY -delete

  # Afficher un résumé
  log_info "Backup retention summary:"
  log_info "  Daily: $(find $backups_dir -maxdepth 1 -name "${ENVIRONMENT}_*" -type d ! -name "*_weekly_*" ! -name "*_monthly_*" ! -name "*_yearly_*" | wc -l) backups"
  log_info "  Weekly: $(find $backups_dir -maxdepth 1 -name "*_weekly_*" -type l | wc -l) backups"
  log_info "  Monthly: $(find $backups_dir -maxdepth 1 -name "*_monthly_*" -type l | wc -l) backups"
  log_info "  Yearly: $(find $backups_dir -maxdepth 1 -name "*_yearly_*" -type l | wc -l) backups"
}

# ============================================================================
# VALIDATION
# ============================================================================

validate_backup() {
  local backup_dir=$1

  log_info "=== Validating Backup ==="

  # Vérifier les checksums
  log_info "Verifying checksums..."
  if [ -f "${backup_dir}/SHA256SUMS" ]; then
    cd "$backup_dir"
    if sha256sum -c SHA256SUMS --quiet; then
      log_info "✓ Checksums verified"
    else
      log_error "Checksum verification failed"
      return 1
    fi
    cd - >/dev/null
  fi

  # Vérifier le manifeste
  log_info "Verifying manifest..."
  if [ -f "${backup_dir}/MANIFEST.json" ]; then
    if jq empty "${backup_dir}/MANIFEST.json" 2>/dev/null; then
      log_info "✓ Manifest is valid JSON"
    else
      log_error "Manifest is invalid"
      return 1
    fi
  fi

  # Vérifier que les fichiers essentiels existent
  log_info "Verifying essential files..."

  local essential_found=0

  if [ -f "${backup_dir}/admin/system.version.bson.gz" ] || \
     [ -f "${backup_dir}/admin/system.version.bson" ]; then
    essential_found=1
  fi

  if [ $essential_found -eq 0 ]; then
    log_error "Essential backup files not found"
    return 1
  fi

  log_info "✓ Essential files present"

  # Test de restauration optionnel (coûteux)
  if [ "${TEST_RESTORE:-false}" = "true" ]; then
    log_info "Performing test restore..."
    test_restore "$backup_dir"
  fi

  log_info "✓ Backup validation completed"
}

test_restore() {
  local backup_dir=$1

  # Créer une instance temporaire pour tester la restauration
  local test_port=27099
  local test_dbpath="/tmp/mongodb_restore_test_$$"

  mkdir -p "$test_dbpath"

  # Démarrer MongoDB temporaire
  mongod --port $test_port --dbpath "$test_dbpath" --fork --logpath "${test_dbpath}/mongod.log"

  sleep 5

  # Tenter la restauration
  if mongorestore --port $test_port --drop --gzip --dir="$backup_dir" 2>&1 | tee -a "$LOG_FILE"; then
    log_info "✓ Test restore successful"

    # Vérifier quelques collections
    mongo --port $test_port --quiet --eval "
      dbs = db.adminCommand({ listDatabases: 1 }).databases;
      print('Databases restored: ' + dbs.length);
    "
  else
    log_error "Test restore failed"
  fi

  # Cleanup
  mongo --port $test_port admin --eval "db.shutdownServer()" || true
  sleep 2
  rm -rf "$test_dbpath"
}

# ============================================================================
# NOTIFICATIONS
# ============================================================================

send_success_notification() {
  local backup_dir=$1
  local duration=$2
  local size_mb=$3

  log_info "Sending success notifications..."

  # Slack
  if [ -n "$SLACK_WEBHOOK" ]; then
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-Type: application/json' \
      -d "{
        \"text\": \"✅ MongoDB Backup Successful\",
        \"attachments\": [{
          \"color\": \"good\",
          \"fields\": [
            {\"title\": \"Environment\", \"value\": \"$ENVIRONMENT\", \"short\": true},
            {\"title\": \"Backup ID\", \"value\": \"$BACKUP_ID\", \"short\": true},
            {\"title\": \"Duration\", \"value\": \"${duration}s\", \"short\": true},
            {\"title\": \"Size\", \"value\": \"${size_mb} MB\", \"short\": true},
            {\"title\": \"Type\", \"value\": \"$BACKUP_TYPE\", \"short\": true},
            {\"title\": \"Storage\", \"value\": \"$REMOTE_STORAGE\", \"short\": true}
          ]
        }]
      }" >/dev/null 2>&1
  fi

  # Email
  if [ -n "$EMAIL_RECIPIENTS" ]; then
    echo "MongoDB backup completed successfully for $ENVIRONMENT" | \
      mail -s "✅ MongoDB Backup Success - $BACKUP_ID" "$EMAIL_RECIPIENTS"
  fi
}

send_failure_notification() {
  log_info "Sending failure notifications..."

  # Slack
  if [ -n "$SLACK_WEBHOOK" ]; then
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-Type: application/json' \
      -d "{
        \"text\": \"🚨 MongoDB Backup Failed\",
        \"attachments\": [{
          \"color\": \"danger\",
          \"fields\": [
            {\"title\": \"Environment\", \"value\": \"$ENVIRONMENT\", \"short\": true},
            {\"title\": \"Backup ID\", \"value\": \"$BACKUP_ID\", \"short\": true},
            {\"title\": \"Log\", \"value\": \"$LOG_FILE\", \"short\": false}
          ]
        }]
      }" >/dev/null 2>&1
  fi

  # PagerDuty (pour environnements critiques)
  if [ -n "$PAGERDUTY_KEY" ] && [ "$ENVIRONMENT" = "production" ]; then
    curl -X POST "https://events.pagerduty.com/v2/enqueue" \
      -H 'Content-Type: application/json' \
      -d "{
        \"routing_key\": \"$PAGERDUTY_KEY\",
        \"event_action\": \"trigger\",
        \"payload\": {
          \"summary\": \"MongoDB Backup Failed - $ENVIRONMENT\",
          \"severity\": \"error\",
          \"source\": \"mongodb-backup\",
          \"custom_details\": {
            \"backup_id\": \"$BACKUP_ID\",
            \"environment\": \"$ENVIRONMENT\",
            \"log_file\": \"$LOG_FILE\"
          }
        }
      }" >/dev/null 2>&1
  fi

  # Email
  if [ -n "$EMAIL_RECIPIENTS" ]; then
    (
      echo "MongoDB backup failed for $ENVIRONMENT"
      echo ""
      echo "Backup ID: $BACKUP_ID"
      echo "Log file: $LOG_FILE"
      echo ""
      echo "Last 50 lines of log:"
      tail -50 "$LOG_FILE"
    ) | mail -s "🚨 MongoDB Backup Failed - $BACKUP_ID" "$EMAIL_RECIPIENTS"
  fi
}

# ============================================================================
# MÉTRIQUES ET MONITORING
# ============================================================================

export_metrics() {
  log_info "Exporting metrics..."

  # Sauvegarder localement
  cat > "$METRICS_FILE" <<EOF
{
  "backup_id": "$BACKUP_ID",
  "environment": "$ENVIRONMENT",
  "timestamp": $(date +%s),
  "metrics": [
    $(IFS=,; echo "${METRICS[*]}")
  ]
}
EOF

  # Prometheus Pushgateway
  if [ -n "$PROMETHEUS_PUSHGATEWAY" ]; then
    local job="mongodb_backup"
    local instance="$ENVIRONMENT"

    # Construire les métriques au format Prometheus
    for metric_json in "${METRICS[@]}"; do
      local name=$(echo "$metric_json" | jq -r '.name')
      local value=$(echo "$metric_json" | jq -r '.value')

      echo "$name{environment=\"$ENVIRONMENT\",backup_id=\"$BACKUP_ID\"} $value"
    done | curl --data-binary @- "$PROMETHEUS_PUSHGATEWAY/metrics/job/$job/instance/$instance"
  fi

  # Datadog (si configuré)
  if [ -n "$DATADOG_API_KEY" ]; then
    for metric_json in "${METRICS[@]}"; do
      local name=$(echo "$metric_json" | jq -r '.name')
      local value=$(echo "$metric_json" | jq -r '.value')
      local timestamp=$(echo "$metric_json" | jq -r '.timestamp')

      curl -X POST "https://api.datadoghq.com/api/v1/series" \
        -H "Content-Type: application/json" \
        -H "DD-API-KEY: $DATADOG_API_KEY" \
        -d "{
          \"series\": [{
            \"metric\": \"$name\",
            \"points\": [[$timestamp, $value]],
            \"type\": \"gauge\",
            \"tags\": [\"environment:$ENVIRONMENT\", \"backup_id:$BACKUP_ID\"]
          }]
        }" >/dev/null 2>&1
    done
  fi
}

# ============================================================================
# CLEANUP
# ============================================================================

cleanup_on_failure() {
  log_info "Cleaning up after failure..."

  # Supprimer le backup incomplet
  if [ -n "${backup_dir:-}" ] && [ -d "$backup_dir" ]; then
    log_info "Removing incomplete backup: $backup_dir"
    rm -rf "$backup_dir"
  fi

  # Supprimer le lock
  rm -f "${BACKUP_ROOT}/.backup_in_progress"
}

cleanup_on_success() {
  log_info "Final cleanup..."

  # Supprimer le lock
  rm -f "${BACKUP_ROOT}/.backup_in_progress"

  # Nettoyer les logs anciens
  find "${BACKUP_ROOT}/logs" -name "backup_*.log" -mtime +30 -delete

  # Nettoyer les métriques anciennes
  find "${BACKUP_ROOT}/metrics" -name "backup_*.json" -mtime +30 -delete
}

# ============================================================================
# MAIN
# ============================================================================

main() {
  local exit_code=0

  # Pré-vérifications
  pre_checks

  # Backup principal
  local backup_dir
  if ! backup_dir=$(perform_backup); then
    exit_code=1
    send_failure_notification
    cleanup_on_failure
    exit $exit_code
  fi

  # Post-processing
  post_process_backup "$backup_dir"

  # Validation
  if ! validate_backup "$backup_dir"; then
    log_error "Backup validation failed"
    exit_code=1
    send_failure_notification
    exit $exit_code
  fi

  # Upload vers stockage distant
  upload_to_remote "$backup_dir"

  # Rotation
  rotate_backups

  # Calcul des métriques finales
  local end_time=$(date +%s)
  local total_duration=$((end_time - START_TIME))
  local backup_size_mb=$(echo "scale=2; $(du -sb $backup_dir | cut -f1) / 1024 / 1024" | bc)

  add_metric "mongodb_backup_total_duration_seconds" "$total_duration"

  # Export des métriques
  export_metrics

  # Notifications
  send_success_notification "$backup_dir" "$total_duration" "$backup_size_mb"

  # Cleanup
  cleanup_on_success

  # Résumé
  log_info ""
  log_info "=========================================="
  log_info "  Backup Completed Successfully"
  log_info "=========================================="
  log_info "Backup ID: $BACKUP_ID"
  log_info "Duration: ${total_duration}s ($(date -u -d @${total_duration} +%H:%M:%S))"
  log_info "Size: ${backup_size_mb} MB"
  log_info "Location: $backup_dir"
  log_info "Log: $LOG_FILE"
  log_info "Metrics: $METRICS_FILE"
  log_info "=========================================="

  exit $exit_code
}

# Exécuter
main "$@"
```

## Planification avec Systemd Timers

### Service Systemd

```ini
# /etc/systemd/system/mongodb-backup.service
[Unit]
Description=MongoDB Automated Backup
After=network.target mongod.service
Wants=mongod.service

[Service]
Type=oneshot
User=mongodb-backup
Group=mongodb-backup

# Variables d'environnement
Environment="BACKUP_ENV=production"
Environment="MONGO_URI=mongodb://localhost:27017"
Environment="BACKUP_ROOT=/backup/mongodb"
Environment="REMOTE_STORAGE=s3"
Environment="S3_BUCKET=company-mongodb-backups"

# Secrets depuis fichier
EnvironmentFile=/etc/mongodb-backup/credentials

# Script principal
ExecStart=/usr/local/bin/mongodb_automated_backup.sh

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=mongodb-backup

# Sécurité
PrivateTmp=yes
NoNewPrivileges=yes
ReadOnlyPaths=/
ReadWritePaths=/backup/mongodb
ReadWritePaths=/tmp

# Limites
TimeoutSec=14400
CPUQuota=80%
MemoryLimit=4G

[Install]
WantedBy=multi-user.target
```

### Timer Systemd

```ini
# /etc/systemd/system/mongodb-backup.timer
[Unit]
Description=MongoDB Backup Timer
Requires=mongodb-backup.service

[Timer]
# Exécuter quotidiennement à 2h du matin
OnCalendar=daily
OnCalendar=*-*-* 02:00:00

# Si le serveur était éteint, rattraper au démarrage
Persistent=true

# Randomiser légèrement l'heure de démarrage
RandomizedDelaySec=300

# Empêcher l'exécution si le précédent n'est pas terminé
Unit=mongodb-backup.service

[Install]
WantedBy=timers.target
```

### Activation

```bash
#!/bin/bash
# setup_systemd_backup.sh

# Copier les fichiers
sudo cp mongodb_automated_backup.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/mongodb_automated_backup.sh

sudo cp mongodb-backup.service /etc/systemd/system/
sudo cp mongodb-backup.timer /etc/systemd/system/

# Créer le fichier de credentials
sudo mkdir -p /etc/mongodb-backup
sudo cat > /etc/mongodb-backup/credentials <<EOF
MONGO_USER=backup-user
MONGO_PASS=SecurePassword123
SLACK_WEBHOOK=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
EOF

sudo chmod 600 /etc/mongodb-backup/credentials

# Créer l'utilisateur de backup
sudo useradd -r -s /bin/bash -d /backup/mongodb mongodb-backup

# Créer les répertoires
sudo mkdir -p /backup/mongodb/{backups,logs,metrics}
sudo chown -R mongodb-backup:mongodb-backup /backup/mongodb

# Recharger systemd
sudo systemctl daemon-reload

# Activer et démarrer le timer
sudo systemctl enable mongodb-backup.timer
sudo systemctl start mongodb-backup.timer

# Vérifier le statut
sudo systemctl status mongodb-backup.timer
sudo systemctl list-timers mongodb-backup.timer

echo "✓ Systemd backup timer configured"
echo ""
echo "Commands:"
echo "  View status: sudo systemctl status mongodb-backup.timer"
echo "  Run now: sudo systemctl start mongodb-backup.service"
echo "  View logs: sudo journalctl -u mongodb-backup -f"
```

## Automatisation Kubernetes

### CronJob Kubernetes

```yaml
# mongodb-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: database
  labels:
    app: mongodb-backup
spec:
  # Quotidien à 2h UTC
  schedule: "0 2 * * *"

  # Politique de concurrence
  concurrencyPolicy: Forbid

  # Historique
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3

  # Template du Job
  jobTemplate:
    spec:
      # Timeout du job
      activeDeadlineSeconds: 14400  # 4 heures

      # Nombre de tentatives
      backoffLimit: 2

      template:
        metadata:
          labels:
            app: mongodb-backup
            cronjob: mongodb-backup
        spec:
          restartPolicy: OnFailure

          # Service Account avec permissions
          serviceAccountName: mongodb-backup

          # Affinité (exécuter sur node avec backup storage)
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: backup-node
                    operator: In
                    values:
                    - "true"

          # Volumes
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: mongodb-backup-pvc
          - name: backup-scripts
            configMap:
              name: mongodb-backup-scripts
              defaultMode: 0755
          - name: credentials
            secret:
              secretName: mongodb-backup-credentials

          # Conteneur principal
          containers:
          - name: backup
            image: mongodb-backup-tools:latest
            imagePullPolicy: IfNotPresent

            command:
            - /scripts/mongodb_automated_backup.sh

            # Variables d'environnement
            env:
            - name: BACKUP_ENV
              value: "production"
            - name: MONGO_URI
              value: "mongodb://mongodb-svc:27017"
            - name: BACKUP_ROOT
              value: "/backup"
            - name: REMOTE_STORAGE
              value: "s3"
            - name: S3_BUCKET
              valueFrom:
                configMapKeyRef:
                  name: mongodb-backup-config
                  key: s3_bucket
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
            - name: SLACK_WEBHOOK
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-credentials
                  key: slack_webhook
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

            # Montages
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            - name: backup-scripts
              mountPath: /scripts
            - name: credentials
              mountPath: /credentials
              readOnly: true

            # Ressources
            resources:
              requests:
                cpu: 2
                memory: 4Gi
              limits:
                cpu: 4
                memory: 8Gi

            # Probes (pas de liveness, juste pour monitoring)
            livenessProbe:
              exec:
                command:
                - test
                - -f
                - /backup/.backup_in_progress
              initialDelaySeconds: 60
              periodSeconds: 60
              timeoutSeconds: 5
              failureThreshold: 60  # 1 heure max

---
# ConfigMap pour les scripts
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-backup-scripts
  namespace: database
data:
  mongodb_automated_backup.sh: |
    #!/bin/bash
    # Script principal (version complète ci-dessus)
    # ...

---
# ConfigMap pour la configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-backup-config
  namespace: database
data:
  s3_bucket: "company-mongodb-backups"
  s3_prefix: "mongodb-backups"
  retention_daily: "7"
  retention_weekly: "28"
  retention_monthly: "180"

---
# Secret pour les credentials
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-backup-credentials
  namespace: database
type: Opaque
stringData:
  username: backup-user
  password: SecurePassword123
  slack_webhook: https://hooks.slack.com/services/YOUR/WEBHOOK/URL

---
# ServiceAccount
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mongodb-backup
  namespace: database

---
# PVC pour le stockage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-backup-pvc
  namespace: database
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Gi
  storageClassName: fast-ssd
```

### Déploiement Kubernetes

```bash
#!/bin/bash
# deploy_k8s_backup.sh

NAMESPACE="database"

# Créer le namespace
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

# Appliquer les manifestes
kubectl apply -f mongodb-backup-cronjob.yaml

# Vérifier
kubectl -n "$NAMESPACE" get cronjob mongodb-backup
kubectl -n "$NAMESPACE" get jobs

# Tester manuellement
kubectl -n "$NAMESPACE" create job --from=cronjob/mongodb-backup mongodb-backup-manual

# Suivre les logs
kubectl -n "$NAMESPACE" logs -f job/mongodb-backup-manual

echo "✓ Kubernetes backup CronJob deployed"
```

## Orchestration avec Ansible

### Playbook Ansible

```yaml
# mongodb_backup_automation.yml
---
- name: Configure MongoDB Automated Backup
  hosts: mongodb_servers
  become: yes
  vars:
    backup_user: mongodb-backup
    backup_root: /backup/mongodb
    mongodb_uri: "mongodb://{{ inventory_hostname }}:27017"

  tasks:
    - name: Install required packages
      apt:
        name:
          - mongodb-clients
          - awscli
          - jq
          - bc
        state: present
        update_cache: yes

    - name: Create backup user
      user:
        name: "{{ backup_user }}"
        system: yes
        shell: /bin/bash
        home: "{{ backup_root }}"
        createhome: yes

    - name: Create backup directories
      file:
        path: "{{ item }}"
        state: directory
        owner: "{{ backup_user }}"
        group: "{{ backup_user }}"
        mode: '0755'
      loop:
        - "{{ backup_root }}/backups"
        - "{{ backup_root }}/logs"
        - "{{ backup_root }}/metrics"
        - "{{ backup_root }}/scripts"

    - name: Copy backup script
      copy:
        src: mongodb_automated_backup.sh
        dest: "{{ backup_root }}/scripts/mongodb_automated_backup.sh"
        owner: "{{ backup_user }}"
        group: "{{ backup_user }}"
        mode: '0750'

    - name: Create credentials file
      template:
        src: backup_credentials.j2
        dest: /etc/mongodb-backup/credentials
        owner: "{{ backup_user }}"
        group: "{{ backup_user }}"
        mode: '0600'

    - name: Configure systemd service
      template:
        src: mongodb-backup.service.j2
        dest: /etc/systemd/system/mongodb-backup.service
        mode: '0644'
      notify: reload systemd

    - name: Configure systemd timer
      template:
        src: mongodb-backup.timer.j2
        dest: /etc/systemd/system/mongodb-backup.timer
        mode: '0644'
      notify: reload systemd

    - name: Enable and start backup timer
      systemd:
        name: mongodb-backup.timer
        enabled: yes
        state: started
        daemon_reload: yes

    - name: Setup logrotate
      copy:
        dest: /etc/logrotate.d/mongodb-backup
        content: |
          {{ backup_root }}/logs/*.log {
            daily
            rotate 30
            compress
            delaycompress
            missingok
            notifempty
            create 0644 {{ backup_user }} {{ backup_user }}
          }

    - name: Configure monitoring (Prometheus node exporter textfile)
      cron:
        name: "Export backup metrics to Prometheus"
        minute: "*/5"
        user: "{{ backup_user }}"
        job: "{{ backup_root }}/scripts/export_backup_metrics.sh > /var/lib/node_exporter/textfile_collector/mongodb_backup.prom"

  handlers:
    - name: reload systemd
      systemd:
        daemon_reload: yes
```

## Monitoring Avancé

### Dashboard Grafana

```json
{
  "dashboard": {
    "title": "MongoDB Backup Automation",
    "panels": [
      {
        "title": "Backup Success Rate (24h)",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(increase(mongodb_backup_success[24h])) / (sum(increase(mongodb_backup_success[24h])) + sum(increase(mongodb_backup_failed[24h]))) * 100"
          }
        ]
      },
      {
        "title": "Backup Duration",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_backup_duration_seconds",
            "legendFormat": "{{environment}}"
          }
        ]
      },
      {
        "title": "Backup Size",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_backup_size_bytes / 1024 / 1024 / 1024",
            "legendFormat": "{{environment}} (GB)"
          }
        ]
      },
      {
        "title": "Last Successful Backup",
        "type": "stat",
        "targets": [
          {
            "expr": "time() - mongodb_backup_last_success_timestamp"
          }
        ]
      },
      {
        "title": "Upload Duration",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_backup_upload_duration_seconds",
            "legendFormat": "{{environment}}"
          }
        ]
      },
      {
        "title": "Backup Failures",
        "type": "table",
        "targets": [
          {
            "expr": "mongodb_backup_failed > 0"
          }
        ]
      }
    ]
  }
}
```

## Bonnes Pratiques

### Checklist d'Automatisation

```markdown
### Configuration Initiale

- [ ] Scripts testés manuellement d'abord
- [ ] Credentials stockés de manière sécurisée
- [ ] Utilisateur dédié avec permissions minimales
- [ ] Répertoires de backup avec permissions correctes
- [ ] Planification configurée (cron/systemd/k8s)
- [ ] Stockage distant configuré et testé
- [ ] Notifications configurées (Slack/Email/PagerDuty)
- [ ] Monitoring configuré (Prometheus/Datadog)

### Validation

- [ ] Test de backup manuel réussi
- [ ] Test de restauration réussi
- [ ] Vérification des checksums
- [ ] Validation du manifeste
- [ ] Upload vers stockage distant vérifié
- [ ] Notifications reçues
- [ ] Métriques exportées

### Opérationnel

- [ ] Rotation des backups fonctionnelle
- [ ] Cleanup automatique des anciens backups
- [ ] Alertes sur échec configurées
- [ ] Logs rotatés automatiquement
- [ ] Documentation à jour
- [ ] Procédures de restauration documentées
- [ ] Tests de restauration réguliers planifiés

### Sécurité

- [ ] Credentials jamais en clair dans les scripts
- [ ] Backups chiffrés (at-rest et in-transit)
- [ ] Accès restreint aux backups
- [ ] Audit trail des accès aux backups
- [ ] Immutabilité des backups (S3 Object Lock)
- [ ] Séparation des rôles (backup vs restore)
```

## Conclusion

L'automatisation des sauvegardes MongoDB est essentielle pour garantir la fiabilité et la consistance de votre stratégie de continuité d'activité. Un système d'automatisation bien conçu doit être :

**Caractéristiques clés** :

1. **Robuste** - Gestion des erreurs, retry, timeouts
2. **Monitoré** - Métriques, alertes, dashboards
3. **Validé** - Tests automatiques de restauration
4. **Notifié** - Alertes immédiates sur échec
5. **Documenté** - Procédures claires et maintenues

**Technologies recommandées** :
- **Petite échelle** : Cron + scripts bash
- **Moyenne échelle** : Systemd timers + Ansible
- **Grande échelle** : Kubernetes CronJobs + GitOps
- **Enterprise** : Orchestrateurs dédiés (Airflow) + MongoDB Ops Manager

L'investissement dans une automatisation solide se traduit par une réduction drastique des risques de perte de données et une amélioration significative du RTO/RPO.

---


⏭️ [Tests de restauration](/12-sauvegarde-restauration/11-tests-restauration.md)
