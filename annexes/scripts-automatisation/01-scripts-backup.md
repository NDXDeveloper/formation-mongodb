🔝 Retour au [Sommaire](/SOMMAIRE.md)

# G.1 Scripts de backup

## Introduction

Les **scripts de backup** sont essentiels pour garantir la **continuité de service** et la **récupération après incident**. Cette section fournit des scripts prêts à l'emploi pour différents scénarios de sauvegarde MongoDB.

## Vue d'ensemble des stratégies

| Stratégie | Méthode | Avantages | Inconvénients |
|-----------|---------|-----------|---------------|
| **mongodump** | Logique | Portable, flexible | Plus lent, charge CPU |
| **Snapshot filesystem** | Physique | Rapide, cohérent | Nécessite LVM/Cloud |
| **Oplog tailing** | Incrémental | Sauvegarde continue | Complexe à restaurer |
| **Cloud Backup** | Managed | Automatisé, fiable | Coût, dépendance |

## Configuration commune

### Fichier de configuration (.env)

```bash
# /opt/mongodb/scripts/.env
# Configuration des backups MongoDB

# Connexion MongoDB
MONGO_HOST="localhost"
MONGO_PORT="27017"
MONGO_USER="backup_user"
MONGO_PASSWORD="secure_password"
MONGO_AUTH_DB="admin"

# URI de connexion complète (alternative)
MONGO_URI="mongodb://backup_user:secure_password@localhost:27017/admin?authSource=admin"

# Replica Set (si applicable)
MONGO_REPLICA_SET="rs0"

# Chemins
BACKUP_ROOT="/backup/mongodb"
BACKUP_DAILY_DIR="$BACKUP_ROOT/daily"
BACKUP_WEEKLY_DIR="$BACKUP_ROOT/weekly"
BACKUP_MONTHLY_DIR="$BACKUP_ROOT/monthly"
LOG_DIR="/var/log/mongodb/backup"

# Rétention (en jours)
RETENTION_DAILY=7
RETENTION_WEEKLY=30
RETENTION_MONTHLY=365

# Compression
COMPRESSION="gzip"  # Options: gzip, none

# Notifications
EMAIL_ADMIN="admin@example.com"
SLACK_WEBHOOK=""
ENABLE_NOTIFICATIONS=true

# Parallélisme
NUM_PARALLEL_COLLECTIONS=4

# Options avancées
OPLOG_BACKUP=true
EXCLUDE_COLLECTIONS="logs,temp"
```

### Bibliothèque de fonctions communes

```bash
# /opt/mongodb/scripts/lib/backup-common.sh
# Fonctions communes pour les scripts de backup

# Couleurs pour les logs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Fonction de logging
log() {
    local level=$1
    shift
    local message="$@"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    case $level in
        INFO)
            echo -e "[${timestamp}] ${GREEN}[INFO]${NC} ${message}" | tee -a "$LOG_FILE"
            ;;
        WARN)
            echo -e "[${timestamp}] ${YELLOW}[WARN]${NC} ${message}" | tee -a "$LOG_FILE"
            ;;
        ERROR)
            echo -e "[${timestamp}] ${RED}[ERROR]${NC} ${message}" | tee -a "$LOG_FILE"
            ;;
        *)
            echo -e "[${timestamp}] [${level}] ${message}" | tee -a "$LOG_FILE"
            ;;
    esac
}

# Gestion des erreurs
error_exit() {
    log ERROR "$1"
    send_notification "❌ Backup MongoDB échoué" "$1"
    exit 1
}

# Envoi de notifications
send_notification() {
    local subject="$1"
    local message="$2"

    [ "$ENABLE_NOTIFICATIONS" != "true" ] && return 0

    # Email
    if [ -n "$EMAIL_ADMIN" ]; then
        echo "$message" | mail -s "$subject" "$EMAIL_ADMIN" 2>/dev/null || true
    fi

    # Slack
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"${subject}\n${message}\"}" \
            "$SLACK_WEBHOOK" 2>/dev/null || true
    fi
}

# Créer les répertoires nécessaires
create_directories() {
    local dirs=("$@")
    for dir in "${dirs[@]}"; do
        if [ ! -d "$dir" ]; then
            mkdir -p "$dir" || error_exit "Impossible de créer le répertoire: $dir"
            log INFO "Répertoire créé: $dir"
        fi
    done
}

# Vérifier l'espace disque disponible
check_disk_space() {
    local path=$1
    local required_gb=${2:-10}  # 10 GB par défaut

    local available_kb=$(df "$path" | awk 'NR==2 {print $4}')
    local available_gb=$((available_kb / 1024 / 1024))

    if [ "$available_gb" -lt "$required_gb" ]; then
        error_exit "Espace disque insuffisant: ${available_gb}GB disponible, ${required_gb}GB requis"
    fi

    log INFO "Espace disque: ${available_gb}GB disponible"
}

# Vérifier la connexion MongoDB
check_mongo_connection() {
    log INFO "Vérification de la connexion MongoDB..."

    if mongosh "$MONGO_URI" --quiet --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
        log INFO "Connexion MongoDB réussie"
        return 0
    else
        error_exit "Impossible de se connecter à MongoDB"
    fi
}

# Obtenir la taille d'une base de données
get_db_size() {
    local db_name=$1
    mongosh "$MONGO_URI" --quiet --eval "print(db.getSiblingDB('${db_name}').stats().dataSize)" 2>/dev/null
}

# Nettoyer les anciens backups
cleanup_old_backups() {
    local backup_dir=$1
    local retention_days=$2

    log INFO "Nettoyage des backups de plus de ${retention_days} jours dans ${backup_dir}"

    find "$backup_dir" -type d -mtime +${retention_days} -exec rm -rf {} + 2>/dev/null || true

    local remaining=$(find "$backup_dir" -type d -maxdepth 1 | wc -l)
    log INFO "Backups restants: $((remaining - 1))"
}

# Calculer la durée d'exécution
calculate_duration() {
    local start_time=$1
    local end_time=$2
    local duration=$((end_time - start_time))

    local hours=$((duration / 3600))
    local minutes=$(((duration % 3600) / 60))
    local seconds=$((duration % 60))

    printf "%02d:%02d:%02d" $hours $minutes $seconds
}

# Valider un backup
validate_backup() {
    local backup_path=$1

    log INFO "Validation du backup: $backup_path"

    # Vérifier l'existence du répertoire
    if [ ! -d "$backup_path" ]; then
        error_exit "Répertoire de backup introuvable: $backup_path"
    fi

    # Vérifier la présence de fichiers BSON
    local bson_count=$(find "$backup_path" -name "*.bson" -o -name "*.bson.gz" | wc -l)
    if [ "$bson_count" -eq 0 ]; then
        error_exit "Aucun fichier BSON trouvé dans le backup"
    fi

    log INFO "Backup validé: ${bson_count} collections sauvegardées"
    return 0
}
```

## 1. Backup complet avec mongodump

### Script de backup quotidien

```bash
#!/bin/bash
#
# Script: backup-daily.sh
# Description: Backup complet quotidien avec mongodump
# Usage: ./backup-daily.sh [database_name]
#

set -euo pipefail

# Configuration
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.env"
source "$SCRIPT_DIR/lib/backup-common.sh"

# Variables
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}"
BACKUP_PATH="$BACKUP_DAILY_DIR/$BACKUP_NAME"
LOG_FILE="$LOG_DIR/backup-daily-${TIMESTAMP}.log"
START_TIME=$(date +%s)

# Base de données spécifique ou toutes
TARGET_DB="${1:-all}"

# Fonction principale
main() {
    log INFO "=========================================="
    log INFO "Début du backup quotidien MongoDB"
    log INFO "=========================================="

    # Prérequis
    create_directories "$BACKUP_DAILY_DIR" "$LOG_DIR"
    check_disk_space "$BACKUP_ROOT" 20
    check_mongo_connection

    # Exécution du backup
    perform_backup

    # Validation
    validate_backup "$BACKUP_PATH"

    # Compression (optionnel)
    if [ "$COMPRESSION" == "gzip" ]; then
        compress_backup
    fi

    # Nettoyage
    cleanup_old_backups "$BACKUP_DAILY_DIR" "$RETENTION_DAILY"

    # Rapport final
    END_TIME=$(date +%s)
    DURATION=$(calculate_duration $START_TIME $END_TIME)
    BACKUP_SIZE=$(du -sh "$BACKUP_PATH" | cut -f1)

    log INFO "=========================================="
    log INFO "Backup terminé avec succès"
    log INFO "Durée: $DURATION"
    log INFO "Taille: $BACKUP_SIZE"
    log INFO "Chemin: $BACKUP_PATH"
    log INFO "=========================================="

    # Notification de succès
    send_notification \
        "✅ Backup MongoDB réussi" \
        "Backup: $BACKUP_NAME\nDurée: $DURATION\nTaille: $BACKUP_SIZE"
}

# Fonction de backup
perform_backup() {
    log INFO "Démarrage de mongodump..."

    local mongodump_cmd="mongodump --uri=\"$MONGO_URI\""
    mongodump_cmd="$mongodump_cmd --out=\"$BACKUP_PATH\""
    mongodump_cmd="$mongodump_cmd --numParallelCollections=$NUM_PARALLEL_COLLECTIONS"

    # Backup d'une base spécifique
    if [ "$TARGET_DB" != "all" ]; then
        log INFO "Backup de la base: $TARGET_DB"
        mongodump_cmd="$mongodump_cmd --db=$TARGET_DB"
    else
        log INFO "Backup de toutes les bases de données"
    fi

    # Inclure l'oplog (pour replica sets)
    if [ "$OPLOG_BACKUP" == "true" ] && [ -n "$MONGO_REPLICA_SET" ]; then
        log INFO "Backup avec oplog activé"
        mongodump_cmd="$mongodump_cmd --oplog"
    fi

    # Exclure certaines collections
    if [ -n "$EXCLUDE_COLLECTIONS" ]; then
        IFS=',' read -ra EXCLUDED <<< "$EXCLUDE_COLLECTIONS"
        for collection in "${EXCLUDED[@]}"; do
            mongodump_cmd="$mongodump_cmd --excludeCollection=$collection"
        done
        log INFO "Collections exclues: $EXCLUDE_COLLECTIONS"
    fi

    # Compression gzip
    if [ "$COMPRESSION" == "gzip" ]; then
        mongodump_cmd="$mongodump_cmd --gzip"
    fi

    # Exécution
    eval $mongodump_cmd >> "$LOG_FILE" 2>&1 || error_exit "mongodump a échoué"

    log INFO "mongodump terminé avec succès"
}

# Compression du backup (si non fait par mongodump)
compress_backup() {
    log INFO "Compression du backup..."

    local archive_name="${BACKUP_NAME}.tar.gz"
    local archive_path="$BACKUP_DAILY_DIR/$archive_name"

    tar -czf "$archive_path" -C "$BACKUP_DAILY_DIR" "$BACKUP_NAME" || error_exit "Compression échouée"

    # Supprimer le répertoire non compressé
    rm -rf "$BACKUP_PATH"
    BACKUP_PATH="$archive_path"

    log INFO "Backup compressé: $archive_name"
}

# Exécution
main "$@"
```

## 2. Backup de Replica Set

```bash
#!/bin/bash
#
# Script: backup-replicaset.sh
# Description: Backup d'un Replica Set avec oplog
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.env"
source "$SCRIPT_DIR/lib/backup-common.sh"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="rs_backup_${TIMESTAMP}"
BACKUP_PATH="$BACKUP_DAILY_DIR/$BACKUP_NAME"
LOG_FILE="$LOG_DIR/backup-replicaset-${TIMESTAMP}.log"

main() {
    log INFO "Début du backup Replica Set: $MONGO_REPLICA_SET"

    create_directories "$BACKUP_DAILY_DIR" "$LOG_DIR"
    check_disk_space "$BACKUP_ROOT" 30

    # Vérifier l'état du replica set
    check_replicaset_status

    # Backup depuis un membre SECONDARY (préférable)
    local secondary_host=$(get_secondary_host)

    if [ -n "$secondary_host" ]; then
        log INFO "Backup depuis le noeud SECONDARY: $secondary_host"
        BACKUP_URI="mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${secondary_host}/${MONGO_AUTH_DB}?replicaSet=${MONGO_REPLICA_SET}"
    else
        log WARN "Aucun SECONDARY disponible, utilisation du PRIMARY"
        BACKUP_URI="$MONGO_URI"
    fi

    # Exécution du backup avec oplog
    log INFO "Exécution de mongodump avec oplog..."

    mongodump \
        --uri="$BACKUP_URI" \
        --out="$BACKUP_PATH" \
        --oplog \
        --gzip \
        --numParallelCollections=$NUM_PARALLEL_COLLECTIONS \
        >> "$LOG_FILE" 2>&1 || error_exit "mongodump a échoué"

    # Validation
    validate_backup "$BACKUP_PATH"

    # Vérifier la présence de l'oplog
    if [ -f "$BACKUP_PATH/oplog.bson.gz" ] || [ -f "$BACKUP_PATH/oplog.bson" ]; then
        log INFO "Oplog sauvegardé avec succès"
    else
        log WARN "Fichier oplog non trouvé"
    fi

    cleanup_old_backups "$BACKUP_DAILY_DIR" "$RETENTION_DAILY"

    log INFO "Backup Replica Set terminé: $BACKUP_PATH"
}

# Vérifier l'état du replica set
check_replicaset_status() {
    log INFO "Vérification de l'état du Replica Set..."

    local rs_status=$(mongosh "$MONGO_URI" --quiet --eval "JSON.stringify(rs.status())" 2>/dev/null)

    if [ -z "$rs_status" ]; then
        error_exit "Impossible d'obtenir le statut du Replica Set"
    fi

    log INFO "Replica Set opérationnel"
}

# Obtenir l'adresse d'un noeud SECONDARY
get_secondary_host() {
    mongosh "$MONGO_URI" --quiet --eval "
        var status = rs.status();
        var secondary = status.members.find(m => m.stateStr === 'SECONDARY');
        if (secondary) print(secondary.name);
    " 2>/dev/null | tail -1
}

main "$@"
```

## 3. Backup de Cluster Shardé

```bash
#!/bin/bash
#
# Script: backup-sharded.sh
# Description: Backup d'un cluster shardé
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.env"
source "$SCRIPT_DIR/lib/backup-common.sh"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="sharded_backup_${TIMESTAMP}"
BACKUP_PATH="$BACKUP_DAILY_DIR/$BACKUP_NAME"
LOG_FILE="$LOG_DIR/backup-sharded-${TIMESTAMP}.log"

# Configuration spécifique au sharding
MONGOS_HOST="${MONGOS_HOST:-localhost:27017}"
CONFIG_SERVERS="${CONFIG_SERVERS:-localhost:27019}"

main() {
    log INFO "Début du backup du cluster shardé"

    create_directories "$BACKUP_DAILY_DIR" "$LOG_DIR"
    check_disk_space "$BACKUP_ROOT" 50

    # 1. Arrêter le balancer
    stop_balancer

    # 2. Backup des config servers
    backup_config_servers

    # 3. Backup de chaque shard
    backup_all_shards

    # 4. Redémarrer le balancer
    start_balancer

    # Validation finale
    validate_backup "$BACKUP_PATH"
    cleanup_old_backups "$BACKUP_DAILY_DIR" "$RETENTION_DAILY"

    log INFO "Backup du cluster shardé terminé: $BACKUP_PATH"
}

# Arrêter le balancer
stop_balancer() {
    log INFO "Arrêt du balancer..."

    mongosh "mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGOS_HOST}/admin" --quiet --eval "
        sh.stopBalancer();
        while(sh.isBalancerRunning()) {
            sleep(1000);
        }
        print('Balancer arrêté');
    " >> "$LOG_FILE" 2>&1

    log INFO "Balancer arrêté avec succès"
}

# Démarrer le balancer
start_balancer() {
    log INFO "Redémarrage du balancer..."

    mongosh "mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGOS_HOST}/admin" --quiet --eval "
        sh.startBalancer();
        print('Balancer démarré');
    " >> "$LOG_FILE" 2>&1

    log INFO "Balancer redémarré avec succès"
}

# Backup des config servers
backup_config_servers() {
    log INFO "Backup des config servers..."

    local config_backup_path="$BACKUP_PATH/config_servers"
    mkdir -p "$config_backup_path"

    mongodump \
        --uri="mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${CONFIG_SERVERS}/admin" \
        --out="$config_backup_path" \
        --oplog \
        --gzip \
        >> "$LOG_FILE" 2>&1 || error_exit "Backup des config servers échoué"

    log INFO "Config servers sauvegardés"
}

# Backup de tous les shards
backup_all_shards() {
    log INFO "Récupération de la liste des shards..."

    local shards=$(mongosh "mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGOS_HOST}/admin" --quiet --eval "
        db.getSiblingDB('config').shards.find({}, {_id:1, host:1}).forEach(function(s) {
            print(s._id + ':' + s.host);
        });
    ")

    while IFS=':' read -r shard_id shard_host; do
        [ -z "$shard_id" ] && continue

        log INFO "Backup du shard: $shard_id ($shard_host)"

        local shard_backup_path="$BACKUP_PATH/shard_${shard_id}"
        mkdir -p "$shard_backup_path"

        # Extraire le host principal du replica set
        local primary_host=$(echo "$shard_host" | sed 's|.*\/||' | cut -d',' -f1)

        mongodump \
            --uri="mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${primary_host}/admin" \
            --out="$shard_backup_path" \
            --oplog \
            --gzip \
            --numParallelCollections=$NUM_PARALLEL_COLLECTIONS \
            >> "$LOG_FILE" 2>&1 || log ERROR "Backup du shard $shard_id échoué"

        log INFO "Shard $shard_id sauvegardé"
    done <<< "$shards"
}

main "$@"
```

## 4. Backup incrémental avec Oplog

```bash
#!/bin/bash
#
# Script: backup-incremental.sh
# Description: Backup incrémental basé sur l'oplog
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.env"
source "$SCRIPT_DIR/lib/backup-common.sh"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="incremental_${TIMESTAMP}"
BACKUP_PATH="$BACKUP_DAILY_DIR/$BACKUP_NAME"
LOG_FILE="$LOG_DIR/backup-incremental-${TIMESTAMP}.log"
LAST_BACKUP_TS_FILE="$BACKUP_ROOT/.last_oplog_timestamp"

main() {
    log INFO "Début du backup incrémental"

    create_directories "$BACKUP_DAILY_DIR" "$LOG_DIR"

    # Vérifier si c'est le premier backup incrémental
    if [ ! -f "$LAST_BACKUP_TS_FILE" ]; then
        log INFO "Premier backup incrémental, création d'un backup complet de référence"
        create_base_backup
    fi

    # Lire le dernier timestamp
    local last_ts=$(cat "$LAST_BACKUP_TS_FILE")
    log INFO "Dernier timestamp: $last_ts"

    # Extraire l'oplog depuis le dernier timestamp
    extract_oplog "$last_ts"

    # Sauvegarder le nouveau timestamp
    save_current_timestamp

    log INFO "Backup incrémental terminé: $BACKUP_PATH"
}

# Créer un backup complet de base
create_base_backup() {
    local base_backup="$BACKUP_ROOT/base_backup"

    log INFO "Création du backup de base..."

    mongodump \
        --uri="$MONGO_URI" \
        --out="$base_backup" \
        --oplog \
        --gzip \
        >> "$LOG_FILE" 2>&1 || error_exit "Backup de base échoué"

    # Sauvegarder le timestamp initial
    mongosh "$MONGO_URI" --quiet --eval "
        var oplog = db.getSiblingDB('local').oplog.rs.find().sort({ts: -1}).limit(1).toArray()[0];
        print(oplog.ts.toString());
    " > "$LAST_BACKUP_TS_FILE"

    log INFO "Backup de base créé: $base_backup"
}

# Extraire l'oplog depuis un timestamp
extract_oplog() {
    local start_ts=$1

    log INFO "Extraction de l'oplog depuis: $start_ts"

    mkdir -p "$BACKUP_PATH"

    # Requête pour extraire l'oplog
    mongodump \
        --uri="$MONGO_URI" \
        --db=local \
        --collection=oplog.rs \
        --query="{ts: {\$gt: Timestamp($start_ts)}}" \
        --out="$BACKUP_PATH" \
        --gzip \
        >> "$LOG_FILE" 2>&1 || error_exit "Extraction de l'oplog échouée"

    log INFO "Oplog extrait avec succès"
}

# Sauvegarder le timestamp actuel
save_current_timestamp() {
    local current_ts=$(mongosh "$MONGO_URI" --quiet --eval "
        var oplog = db.getSiblingDB('local').oplog.rs.find().sort({ts: -1}).limit(1).toArray()[0];
        print(oplog.ts.toString());
    ")

    echo "$current_ts" > "$LAST_BACKUP_TS_FILE"
    log INFO "Nouveau timestamp sauvegardé: $current_ts"
}

main "$@"
```

## 5. Script de restauration

```bash
#!/bin/bash
#
# Script: restore-backup.sh
# Description: Restauration d'un backup MongoDB
# Usage: ./restore-backup.sh <backup_path> [target_db]
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.env"
source "$SCRIPT_DIR/lib/backup-common.sh"

BACKUP_TO_RESTORE="${1:?Usage: $0 <backup_path> [target_db]}"
TARGET_DB="${2:-}"
LOG_FILE="$LOG_DIR/restore-$(date +%Y%m%d_%H%M%S).log"

main() {
    log INFO "=========================================="
    log INFO "Début de la restauration MongoDB"
    log INFO "=========================================="

    # Vérifications
    check_backup_exists
    confirm_restore

    # Décompression si nécessaire
    if [[ "$BACKUP_TO_RESTORE" == *.tar.gz ]]; then
        decompress_backup
    fi

    # Restauration
    perform_restore

    log INFO "=========================================="
    log INFO "Restauration terminée avec succès"
    log INFO "=========================================="
}

# Vérifier l'existence du backup
check_backup_exists() {
    if [ ! -e "$BACKUP_TO_RESTORE" ]; then
        error_exit "Backup introuvable: $BACKUP_TO_RESTORE"
    fi

    log INFO "Backup trouvé: $BACKUP_TO_RESTORE"
}

# Confirmation interactive
confirm_restore() {
    log WARN "ATTENTION: Cette opération va restaurer les données MongoDB"

    if [ -n "$TARGET_DB" ]; then
        log WARN "Base de données cible: $TARGET_DB"
    else
        log WARN "Restauration complète de toutes les bases de données"
    fi

    read -p "Voulez-vous continuer? (yes/no): " -r
    if [[ ! $REPLY =~ ^[Yy][Ee][Ss]$ ]]; then
        log INFO "Restauration annulée par l'utilisateur"
        exit 0
    fi
}

# Décompression
decompress_backup() {
    log INFO "Décompression du backup..."

    local temp_dir="/tmp/mongodb_restore_$$"
    mkdir -p "$temp_dir"

    tar -xzf "$BACKUP_TO_RESTORE" -C "$temp_dir" || error_exit "Décompression échouée"

    BACKUP_TO_RESTORE="$temp_dir/$(ls -1 $temp_dir | head -1)"
    log INFO "Backup décompressé dans: $BACKUP_TO_RESTORE"
}

# Restauration
perform_restore() {
    log INFO "Démarrage de mongorestore..."

    local restore_cmd="mongorestore --uri=\"$MONGO_URI\""

    # Options
    if [ -n "$TARGET_DB" ]; then
        restore_cmd="$restore_cmd --db=$TARGET_DB"
        log INFO "Restauration dans la base: $TARGET_DB"
    fi

    # Drop avant restauration (optionnel)
    # restore_cmd="$restore_cmd --drop"

    # Parallélisme
    restore_cmd="$restore_cmd --numParallelCollections=$NUM_PARALLEL_COLLECTIONS"

    # Gzip
    if ls "$BACKUP_TO_RESTORE"/*.bson.gz >/dev/null 2>&1; then
        restore_cmd="$restore_cmd --gzip"
    fi

    # Oplog replay (si présent)
    if [ -f "$BACKUP_TO_RESTORE/oplog.bson" ] || [ -f "$BACKUP_TO_RESTORE/oplog.bson.gz" ]; then
        log INFO "Application de l'oplog"
        restore_cmd="$restore_cmd --oplogReplay"
    fi

    # Chemin du backup
    restore_cmd="$restore_cmd \"$BACKUP_TO_RESTORE\""

    # Exécution
    eval $restore_cmd >> "$LOG_FILE" 2>&1 || error_exit "mongorestore a échoué"

    log INFO "mongorestore terminé avec succès"
}

main "$@"
```

## 6. Script de vérification de backup

```bash
#!/bin/bash
#
# Script: verify-backup.sh
# Description: Vérification approfondie d'un backup
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.env"
source "$SCRIPT_DIR/lib/backup-common.sh"

BACKUP_PATH="${1:?Usage: $0 <backup_path>}"
LOG_FILE="$LOG_DIR/verify-$(date +%Y%m%d_%H%M%S).log"

main() {
    log INFO "Vérification du backup: $BACKUP_PATH"

    # Vérifications de base
    check_backup_structure
    check_metadata_files
    check_bson_files
    check_oplog

    # Statistiques
    generate_statistics

    log INFO "Vérification terminée: Backup valide ✓"
}

# Vérifier la structure du backup
check_backup_structure() {
    if [ ! -d "$BACKUP_PATH" ]; then
        error_exit "Le chemin n'est pas un répertoire valide"
    fi

    log INFO "Structure du backup vérifiée"
}

# Vérifier les fichiers de métadonnées
check_metadata_files() {
    log INFO "Vérification des métadonnées..."

    local db_count=0
    for db_dir in "$BACKUP_PATH"/*/; do
        [ -d "$db_dir" ] || continue

        local db_name=$(basename "$db_dir")
        db_count=$((db_count + 1))

        # Vérifier les fichiers metadata.json
        local metadata_files=$(find "$db_dir" -name "*.metadata.json*" | wc -l)
        log INFO "  Base: $db_name - $metadata_files collections"
    done

    if [ $db_count -eq 0 ]; then
        error_exit "Aucune base de données trouvée dans le backup"
    fi

    log INFO "Bases de données trouvées: $db_count"
}

# Vérifier les fichiers BSON
check_bson_files() {
    log INFO "Vérification des fichiers BSON..."

    local bson_count=$(find "$BACKUP_PATH" -name "*.bson" -o -name "*.bson.gz" | wc -l)

    if [ $bson_count -eq 0 ]; then
        error_exit "Aucun fichier BSON trouvé"
    fi

    log INFO "Fichiers BSON trouvés: $bson_count"
}

# Vérifier l'oplog
check_oplog() {
    if [ -f "$BACKUP_PATH/oplog.bson" ] || [ -f "$BACKUP_PATH/oplog.bson.gz" ]; then
        log INFO "Oplog présent dans le backup ✓"
    else
        log WARN "Aucun oplog trouvé (normal pour backup standalone)"
    fi
}

# Générer des statistiques
generate_statistics() {
    log INFO "Génération des statistiques..."

    local total_size=$(du -sh "$BACKUP_PATH" | cut -f1)
    local file_count=$(find "$BACKUP_PATH" -type f | wc -l)
    local oldest_file=$(find "$BACKUP_PATH" -type f -printf '%T+ %p\n' | sort | head -1 | cut -d' ' -f1)

    log INFO "----------------------------------------"
    log INFO "Statistiques du backup:"
    log INFO "  Taille totale: $total_size"
    log INFO "  Nombre de fichiers: $file_count"
    log INFO "  Date de création: $oldest_file"
    log INFO "----------------------------------------"
}

main "$@"
```

## 7. Backup automatisé avec rotation

```bash
#!/bin/bash
#
# Script: backup-with-rotation.sh
# Description: Backup avec rotation quotidien/hebdomadaire/mensuel
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.env"
source "$SCRIPT_DIR/lib/backup-common.sh"

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DAY_OF_WEEK=$(date +%u)     # 1-7 (Lundi-Dimanche)
DAY_OF_MONTH=$(date +%d)    # 01-31
LOG_FILE="$LOG_DIR/backup-rotation-${TIMESTAMP}.log"

main() {
    log INFO "Début du backup avec rotation"

    create_directories "$BACKUP_DAILY_DIR" "$BACKUP_WEEKLY_DIR" "$BACKUP_MONTHLY_DIR" "$LOG_DIR"
    check_disk_space "$BACKUP_ROOT" 50

    # Déterminer le type de backup
    if [ "$DAY_OF_MONTH" == "01" ]; then
        # Backup mensuel (1er du mois)
        perform_monthly_backup
    elif [ "$DAY_OF_WEEK" == "7" ]; then
        # Backup hebdomadaire (dimanche)
        perform_weekly_backup
    else
        # Backup quotidien
        perform_daily_backup
    fi

    log INFO "Backup avec rotation terminé"
}

# Backup quotidien
perform_daily_backup() {
    log INFO "Exécution du backup quotidien"

    local backup_name="daily_$(date +%Y%m%d)"
    local backup_path="$BACKUP_DAILY_DIR/$backup_name"

    mongodump \
        --uri="$MONGO_URI" \
        --out="$backup_path" \
        --gzip \
        --numParallelCollections=$NUM_PARALLEL_COLLECTIONS \
        >> "$LOG_FILE" 2>&1 || error_exit "Backup quotidien échoué"

    cleanup_old_backups "$BACKUP_DAILY_DIR" "$RETENTION_DAILY"

    send_notification "📅 Backup quotidien MongoDB" "Backup: $backup_name"
}

# Backup hebdomadaire
perform_weekly_backup() {
    log INFO "Exécution du backup hebdomadaire"

    local backup_name="weekly_$(date +%Y_W%V)"
    local backup_path="$BACKUP_WEEKLY_DIR/$backup_name"

    mongodump \
        --uri="$MONGO_URI" \
        --out="$backup_path" \
        --oplog \
        --gzip \
        --numParallelCollections=$NUM_PARALLEL_COLLECTIONS \
        >> "$LOG_FILE" 2>&1 || error_exit "Backup hebdomadaire échoué"

    cleanup_old_backups "$BACKUP_WEEKLY_DIR" "$RETENTION_WEEKLY"

    send_notification "📅 Backup hebdomadaire MongoDB" "Backup: $backup_name"
}

# Backup mensuel
perform_monthly_backup() {
    log INFO "Exécution du backup mensuel"

    local backup_name="monthly_$(date +%Y%m)"
    local backup_path="$BACKUP_MONTHLY_DIR/$backup_name"

    mongodump \
        --uri="$MONGO_URI" \
        --out="$backup_path" \
        --oplog \
        --gzip \
        --numParallelCollections=$NUM_PARALLEL_COLLECTIONS \
        >> "$LOG_FILE" 2>&1 || error_exit "Backup mensuel échoué"

    cleanup_old_backups "$BACKUP_MONTHLY_DIR" "$RETENTION_MONTHLY"

    send_notification "📅 Backup mensuel MongoDB" "Backup: $backup_name"
}

main "$@"
```

## Configuration Cron

```cron
# /etc/crontab ou crontab -e

# Backup quotidien à 2h du matin
0 2 * * * /opt/mongodb/scripts/backup-daily.sh >> /var/log/mongodb/cron-backup.log 2>&1

# Backup avec rotation
0 2 * * * /opt/mongodb/scripts/backup-with-rotation.sh >> /var/log/mongodb/cron-rotation.log 2>&1

# Vérification des backups (tous les jours à 3h)
0 3 * * * /opt/mongodb/scripts/verify-last-backup.sh >> /var/log/mongodb/cron-verify.log 2>&1

# Nettoyage hebdomadaire (dimanche à 4h)
0 4 * * 0 /opt/mongodb/scripts/cleanup-old-backups.sh >> /var/log/mongodb/cron-cleanup.log 2>&1
```

## Checklist de déploiement

- [ ] Variables d'environnement configurées dans `.env`
- [ ] Utilisateur MongoDB avec droits `backup` créé
- [ ] Répertoires de backup créés avec permissions appropriées
- [ ] Scripts testés en environnement de développement
- [ ] Restauration testée à partir d'un backup
- [ ] Notifications configurées et testées
- [ ] Tâches cron planifiées
- [ ] Monitoring des jobs de backup actif
- [ ] Documentation de restauration à jour
- [ ] Plan de disaster recovery documenté

## Bonnes pratiques

### Sécurité
- ✅ Stocker les credentials dans des fichiers protégés (chmod 600)
- ✅ Utiliser un utilisateur dédié avec privilèges minimaux
- ✅ Chiffrer les backups sensibles
- ✅ Stocker les backups hors site (S3, NAS distant)

### Performance
- ✅ Faire les backups depuis un noeud SECONDARY
- ✅ Utiliser `--numParallelCollections` pour accélérer
- ✅ Planifier pendant les heures creuses
- ✅ Compresser avec `--gzip`

### Fiabilité
- ✅ Toujours valider les backups après création
- ✅ Tester régulièrement les restaurations
- ✅ Conserver plusieurs générations (3-2-1 rule)
- ✅ Monitorer l'espace disque disponible

### Automatisation
- ✅ Utiliser des scripts standardisés
- ✅ Logger toutes les opérations
- ✅ Envoyer des notifications en cas d'échec
- ✅ Documenter les procédures

---


⏭️ [Scripts de monitoring](/annexes/scripts-automatisation/02-scripts-monitoring.md)
