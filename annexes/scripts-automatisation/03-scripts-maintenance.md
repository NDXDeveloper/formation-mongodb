🔝 Retour au [Sommaire](/SOMMAIRE.md)

# G.3 Scripts de maintenance

## Introduction

La **maintenance régulière** de MongoDB est cruciale pour maintenir des performances optimales, prévenir les problèmes et assurer la stabilité du système. Cette section fournit des scripts automatisables pour toutes les tâches de maintenance courantes.

## Planning de maintenance recommandé

| Tâche | Fréquence | Impact | Fenêtre maintenance |
|-------|-----------|--------|---------------------|
| **Rotation des logs** | Quotidienne | Faible | Non requise |
| **Nettoyage données expirées** | Quotidienne | Faible | Non requise |
| **Statistiques collections** | Hebdomadaire | Moyen | Non requise |
| **Compactage** | Mensuelle | Élevé | Requise |
| **Réindexation** | Trimestrielle | Élevé | Requise |
| **Validation données** | Trimestrielle | Faible | Non requise |
| **Mise à jour MongoDB** | Variable | Élevé | Requise |

## Configuration commune

### Fichier de configuration

```bash
# /opt/mongodb/scripts/.maintenance-env
# Configuration pour les scripts de maintenance

# Connexion MongoDB
MONGO_HOST="localhost"
MONGO_PORT="27017"
MONGO_USER="maintenance_user"
MONGO_PASSWORD="secure_password"
MONGO_AUTH_DB="admin"
MONGO_URI="mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGO_HOST}:${MONGO_PORT}/${MONGO_AUTH_DB}"

# Chemins
LOG_DIR="/var/log/mongodb/maintenance"
MAINTENANCE_LOG="$LOG_DIR/maintenance.log"
ARCHIVE_DIR="/var/lib/mongodb/archives"

# Paramètres de maintenance
COMPACT_THRESHOLD_GB=10
INDEX_FRAGMENTATION_THRESHOLD=30
LOG_RETENTION_DAYS=30
DATA_RETENTION_DAYS=90
ARCHIVE_RETENTION_DAYS=365

# Notifications
EMAIL_ADMIN="admin@example.com"
SLACK_WEBHOOK=""
ENABLE_NOTIFICATIONS=true

# Sécurité
BACKUP_BEFORE_MAINTENANCE=true
MAINTENANCE_WINDOW_START="02:00"
MAINTENANCE_WINDOW_END="06:00"
```

### Bibliothèque de fonctions

```bash
# /opt/mongodb/scripts/lib/maintenance-common.sh
# Fonctions communes pour la maintenance

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# Logging
log_maintenance() {
    local level=$1
    shift
    local message="$@"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    echo "[$timestamp] [$level] $message" | tee -a "$MAINTENANCE_LOG"
}

# Vérifier la fenêtre de maintenance
check_maintenance_window() {
    local current_time=$(date +%H:%M)
    local start="$MAINTENANCE_WINDOW_START"
    local end="$MAINTENANCE_WINDOW_END"

    if [[ "$current_time" < "$start" ]] || [[ "$current_time" > "$end" ]]; then
        log_maintenance "ERROR" "Hors fenêtre de maintenance ($start-$end)"
        return 1
    fi

    log_maintenance "INFO" "Dans la fenêtre de maintenance"
    return 0
}

# Créer un backup de sécurité
create_safety_backup() {
    local db_name=$1
    local backup_path="$ARCHIVE_DIR/safety_backup_${db_name}_$(date +%Y%m%d_%H%M%S)"

    log_maintenance "INFO" "Création d'un backup de sécurité: $db_name"

    mongodump --uri="$MONGO_URI" --db="$db_name" --out="$backup_path" --gzip \
        >> "$MAINTENANCE_LOG" 2>&1 || {
        log_maintenance "ERROR" "Échec du backup de sécurité"
        return 1
    }

    log_maintenance "INFO" "Backup créé: $backup_path"
    return 0
}

# Calculer la taille d'une collection
get_collection_size_gb() {
    local db_name=$1
    local coll_name=$2

    mongosh "$MONGO_URI" --quiet --eval "
        var stats = db.getSiblingDB('${db_name}').${coll_name}.stats();
        print((stats.size / 1024 / 1024 / 1024).toFixed(2));
    " 2>/dev/null
}

# Calculer la fragmentation d'un index
get_index_fragmentation() {
    local db_name=$1
    local coll_name=$2

    mongosh "$MONGO_URI" --quiet --eval "
        var stats = db.getSiblingDB('${db_name}').${coll_name}.stats();
        if (stats.indexSizes && stats.storageSize) {
            var totalIndexSize = Object.values(stats.indexSizes).reduce((a,b) => a+b, 0);
            var fragmentation = ((stats.storageSize - stats.size) / stats.storageSize * 100);
            print(fragmentation.toFixed(2));
        } else {
            print(0);
        }
    " 2>/dev/null
}

# Envoyer une notification
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

# Vérifier qu'on est sur un noeud approprié
check_node_suitability() {
    local is_primary=$(mongosh "$MONGO_URI" --quiet --eval "rs.isMaster().ismaster" 2>/dev/null || echo "true")

    if [ "$is_primary" == "true" ]; then
        log_maintenance "WARN" "Exécution sur le noeud PRIMARY - Impact possible"
        read -p "Continuer? (yes/no): " -r
        [[ ! $REPLY =~ ^[Yy][Ee][Ss]$ ]] && return 1
    else
        log_maintenance "INFO" "Exécution sur un noeud SECONDARY"
    fi

    return 0
}
```

## 1. Rotation des logs

```bash
#!/bin/bash
#
# Script: rotate-logs.sh
# Description: Rotation des fichiers de logs MongoDB
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.maintenance-env"
source "$SCRIPT_DIR/lib/maintenance-common.sh"

MONGODB_LOG_DIR="/var/log/mongodb"
MONGODB_LOG_FILE="$MONGODB_LOG_DIR/mongod.log"

main() {
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Début de la rotation des logs"
    log_maintenance "INFO" "=========================================="

    # Vérifier que le fichier de log existe
    if [ ! -f "$MONGODB_LOG_FILE" ]; then
        log_maintenance "ERROR" "Fichier de log introuvable: $MONGODB_LOG_FILE"
        exit 1
    fi

    # Obtenir la taille du log avant rotation
    local log_size_before=$(du -h "$MONGODB_LOG_FILE" | cut -f1)
    log_maintenance "INFO" "Taille du log avant rotation: $log_size_before"

    # Rotation via commande MongoDB
    rotate_mongodb_log

    # Compression de l'ancien log
    compress_old_logs

    # Nettoyage des anciens logs
    cleanup_old_logs

    # Statistiques
    local log_size_after=$(du -h "$MONGODB_LOG_FILE" | cut -f1)
    log_maintenance "INFO" "Taille du log après rotation: $log_size_after"

    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Rotation des logs terminée"
    log_maintenance "INFO" "=========================================="
}

# Rotation via logRotate MongoDB
rotate_mongodb_log() {
    log_maintenance "INFO" "Exécution de logRotate..."

    if mongosh "$MONGO_URI" --quiet --eval "db.adminCommand({logRotate: 1})" >/dev/null 2>&1; then
        log_maintenance "INFO" "logRotate exécuté avec succès"
    else
        log_maintenance "ERROR" "Échec de logRotate"
        exit 1
    fi

    # Attendre que la rotation soit effective
    sleep 2
}

# Compresser les anciens logs
compress_old_logs() {
    log_maintenance "INFO" "Compression des anciens logs..."

    local compressed=0
    for log_file in "$MONGODB_LOG_DIR"/mongod.log.*; do
        [ ! -f "$log_file" ] && continue
        [ "${log_file##*.}" == "gz" ] && continue

        log_maintenance "INFO" "Compression de: $(basename $log_file)"
        gzip "$log_file" && compressed=$((compressed + 1))
    done

    log_maintenance "INFO" "$compressed fichiers compressés"
}

# Nettoyer les logs trop anciens
cleanup_old_logs() {
    log_maintenance "INFO" "Nettoyage des logs de plus de ${LOG_RETENTION_DAYS} jours..."

    local deleted=$(find "$MONGODB_LOG_DIR" -name "mongod.log.*.gz" -mtime +${LOG_RETENTION_DAYS} -delete -print | wc -l)

    log_maintenance "INFO" "$deleted fichiers supprimés"
}

main "$@"
```

## 2. Compactage des collections

```bash
#!/bin/bash
#
# Script: compact-collections.sh
# Description: Compactage des collections fragmentées
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.maintenance-env"
source "$SCRIPT_DIR/lib/maintenance-common.sh"

TARGET_DB="${1:-}"
TARGET_COLLECTION="${2:-}"

main() {
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Début du compactage des collections"
    log_maintenance "INFO" "=========================================="

    # Vérifier la fenêtre de maintenance
    check_maintenance_window || exit 1

    # Vérifier le noeud
    check_node_suitability || exit 1

    # Analyser les collections
    if [ -n "$TARGET_DB" ] && [ -n "$TARGET_COLLECTION" ]; then
        compact_single_collection "$TARGET_DB" "$TARGET_COLLECTION"
    elif [ -n "$TARGET_DB" ]; then
        compact_database "$TARGET_DB"
    else
        compact_all_eligible_collections
    fi

    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Compactage terminé"
    log_maintenance "INFO" "=========================================="
}

# Compacter une seule collection
compact_single_collection() {
    local db_name=$1
    local coll_name=$2

    log_maintenance "INFO" "Compactage: ${db_name}.${coll_name}"

    # Backup de sécurité
    if [ "$BACKUP_BEFORE_MAINTENANCE" == "true" ]; then
        create_safety_backup "$db_name" || exit 1
    fi

    # Obtenir la taille avant
    local size_before=$(get_collection_size_gb "$db_name" "$coll_name")
    log_maintenance "INFO" "Taille avant: ${size_before} GB"

    # Exécuter compact
    local start_time=$(date +%s)

    if mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').runCommand({compact: '${coll_name}', force: true})
    " >> "$MAINTENANCE_LOG" 2>&1; then
        local end_time=$(date +%s)
        local duration=$((end_time - start_time))

        # Taille après
        local size_after=$(get_collection_size_gb "$db_name" "$coll_name")
        local saved=$(echo "$size_before - $size_after" | bc)

        log_maintenance "INFO" "Compactage réussi"
        log_maintenance "INFO" "Durée: ${duration}s"
        log_maintenance "INFO" "Taille après: ${size_after} GB"
        log_maintenance "INFO" "Espace récupéré: ${saved} GB"
    else
        log_maintenance "ERROR" "Échec du compactage"
        return 1
    fi
}

# Compacter toutes les collections d'une base
compact_database() {
    local db_name=$1

    log_maintenance "INFO" "Compactage de la base: $db_name"

    # Lister les collections
    local collections=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').getCollectionNames().forEach(function(c) {
            print(c);
        });
    ")

    local total=0
    local success=0

    while read -r coll_name; do
        [ -z "$coll_name" ] && continue
        total=$((total + 1))

        if compact_single_collection "$db_name" "$coll_name"; then
            success=$((success + 1))
        fi
    done <<< "$collections"

    log_maintenance "INFO" "Compactage terminé: $success/$total collections"
}

# Compacter toutes les collections éligibles
compact_all_eligible_collections() {
    log_maintenance "INFO" "Recherche des collections éligibles au compactage..."

    local eligible=$(mongosh "$MONGO_URI" --quiet --eval "
        var dbs = db.adminCommand('listDatabases').databases;

        dbs.forEach(function(database) {
            if (['admin', 'config', 'local'].indexOf(database.name) !== -1) return;

            var dbObj = db.getSiblingDB(database.name);
            var collections = dbObj.getCollectionNames();

            collections.forEach(function(collName) {
                var stats = dbObj[collName].stats();
                var sizeGB = stats.size / 1024 / 1024 / 1024;

                // Éligible si > seuil et fragmentation > 20%
                if (sizeGB > ${COMPACT_THRESHOLD_GB}) {
                    var fragmentation = ((stats.storageSize - stats.size) / stats.storageSize * 100);
                    if (fragmentation > 20) {
                        print(database.name + ':' + collName + ':' + sizeGB.toFixed(2) + ':' + fragmentation.toFixed(2));
                    }
                }
            });
        });
    ")

    if [ -z "$eligible" ]; then
        log_maintenance "INFO" "Aucune collection éligible au compactage"
        return
    fi

    log_maintenance "INFO" "Collections éligibles:"
    echo "$eligible" | while IFS=: read -r db_name coll_name size frag; do
        log_maintenance "INFO" "  ${db_name}.${coll_name} - ${size}GB - ${frag}% fragmentation"
    done

    # Compacter chaque collection éligible
    echo "$eligible" | while IFS=: read -r db_name coll_name size frag; do
        [ -z "$db_name" ] && continue
        compact_single_collection "$db_name" "$coll_name"
        sleep 5  # Pause entre chaque compactage
    done
}

main "$@"
```

## 3. Réindexation

```bash
#!/bin/bash
#
# Script: reindex-collections.sh
# Description: Reconstruction des index fragmentés
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.maintenance-env"
source "$SCRIPT_DIR/lib/maintenance-common.sh"

TARGET_DB="${1:-}"

main() {
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Début de la réindexation"
    log_maintenance "INFO" "=========================================="

    check_maintenance_window || exit 1
    check_node_suitability || exit 1

    if [ -n "$TARGET_DB" ]; then
        reindex_database "$TARGET_DB"
    else
        reindex_all_databases
    fi

    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Réindexation terminée"
    log_maintenance "INFO" "=========================================="
}

# Réindexer une base de données
reindex_database() {
    local db_name=$1

    log_maintenance "INFO" "Réindexation de la base: $db_name"

    # Backup de sécurité
    if [ "$BACKUP_BEFORE_MAINTENANCE" == "true" ]; then
        create_safety_backup "$db_name" || exit 1
    fi

    # Lister les collections
    local collections=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').getCollectionNames().forEach(function(c) {
            print(c);
        });
    ")

    local total=0
    local success=0

    while read -r coll_name; do
        [ -z "$coll_name" ] && continue
        [ "$coll_name" == "system.profile" ] && continue

        total=$((total + 1))

        if reindex_collection "$db_name" "$coll_name"; then
            success=$((success + 1))
        fi
    done <<< "$collections"

    log_maintenance "INFO" "Réindexation: $success/$total collections"
}

# Réindexer une collection
reindex_collection() {
    local db_name=$1
    local coll_name=$2

    log_maintenance "INFO" "Réindexation: ${db_name}.${coll_name}"

    # Compter les index
    local index_count=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').${coll_name}.getIndexes().length
    ")

    log_maintenance "INFO" "Nombre d'index: $index_count"

    # Exécuter reIndex
    local start_time=$(date +%s)

    if mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').${coll_name}.reIndex()
    " >> "$MAINTENANCE_LOG" 2>&1; then
        local end_time=$(date +%s)
        local duration=$((end_time - start_time))

        log_maintenance "INFO" "Réindexation réussie (durée: ${duration}s)"
        return 0
    else
        log_maintenance "ERROR" "Échec de la réindexation"
        return 1
    fi
}

# Réindexer toutes les bases
reindex_all_databases() {
    log_maintenance "INFO" "Réindexation de toutes les bases de données..."

    local databases=$(mongosh "$MONGO_URI" --quiet --eval "
        db.adminCommand('listDatabases').databases.forEach(function(d) {
            if (['admin', 'config', 'local'].indexOf(d.name) === -1) {
                print(d.name);
            }
        });
    ")

    while read -r db_name; do
        [ -z "$db_name" ] && continue
        reindex_database "$db_name"
        sleep 5
    done <<< "$databases"
}

main "$@"
```

## 4. Nettoyage des données expirées

```bash
#!/bin/bash
#
# Script: cleanup-expired-data.sh
# Description: Suppression des données expirées
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.maintenance-env"
source "$SCRIPT_DIR/lib/maintenance-common.sh"

main() {
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Début du nettoyage des données expirées"
    log_maintenance "INFO" "=========================================="

    # Nettoyage basé sur TTL indexes
    cleanup_ttl_collections

    # Nettoyage manuel pour collections sans TTL
    cleanup_logs_collection
    cleanup_sessions_collection
    cleanup_temp_data

    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Nettoyage terminé"
    log_maintenance "INFO" "=========================================="
}

# Vérifier les collections avec index TTL
cleanup_ttl_collections() {
    log_maintenance "INFO" "--- Collections avec index TTL ---"

    local ttl_collections=$(mongosh "$MONGO_URI" --quiet --eval "
        var dbs = db.adminCommand('listDatabases').databases;

        dbs.forEach(function(database) {
            if (['admin', 'config', 'local'].indexOf(database.name) !== -1) return;

            var dbObj = db.getSiblingDB(database.name);
            var collections = dbObj.getCollectionNames();

            collections.forEach(function(collName) {
                var indexes = dbObj[collName].getIndexes();
                indexes.forEach(function(idx) {
                    if (idx.expireAfterSeconds !== undefined) {
                        print(database.name + ':' + collName + ':' + idx.name + ':' + idx.expireAfterSeconds);
                    }
                });
            });
        });
    ")

    if [ -z "$ttl_collections" ]; then
        log_maintenance "INFO" "Aucune collection avec index TTL"
        return
    fi

    echo "$ttl_collections" | while IFS=: read -r db_name coll_name idx_name ttl_seconds; do
        [ -z "$db_name" ] && continue

        local ttl_hours=$((ttl_seconds / 3600))
        log_maintenance "INFO" "${db_name}.${coll_name} - TTL: ${ttl_hours}h (index: $idx_name)"
    done
}

# Nettoyage de la collection de logs
cleanup_logs_collection() {
    log_maintenance "INFO" "--- Nettoyage logs ---"

    local db_name="myapp"
    local coll_name="logs"
    local cutoff_date=$(date -u -d "${DATA_RETENTION_DAYS} days ago" +%Y-%m-%dT%H:%M:%S)

    # Vérifier si la collection existe
    local exists=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').getCollectionNames().includes('${coll_name}')
    ")

    if [ "$exists" != "true" ]; then
        log_maintenance "INFO" "Collection ${db_name}.${coll_name} inexistante"
        return
    fi

    log_maintenance "INFO" "Suppression des logs avant: $cutoff_date"

    local deleted=$(mongosh "$MONGO_URI" --quiet --eval "
        var result = db.getSiblingDB('${db_name}').${coll_name}.deleteMany({
            timestamp: { \$lt: new Date('${cutoff_date}') }
        });
        print(result.deletedCount);
    ")

    log_maintenance "INFO" "Documents supprimés: $deleted"
}

# Nettoyage des sessions expirées
cleanup_sessions_collection() {
    log_maintenance "INFO" "--- Nettoyage sessions ---"

    local db_name="myapp"
    local coll_name="sessions"

    local exists=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').getCollectionNames().includes('${coll_name}')
    ")

    if [ "$exists" != "true" ]; then
        log_maintenance "INFO" "Collection ${db_name}.${coll_name} inexistante"
        return
    fi

    local now=$(date -u +%Y-%m-%dT%H:%M:%S)

    local deleted=$(mongosh "$MONGO_URI" --quiet --eval "
        var result = db.getSiblingDB('${db_name}').${coll_name}.deleteMany({
            expiresAt: { \$lt: new Date('${now}') }
        });
        print(result.deletedCount);
    ")

    log_maintenance "INFO" "Sessions expirées supprimées: $deleted"
}

# Nettoyage des données temporaires
cleanup_temp_data() {
    log_maintenance "INFO" "--- Nettoyage données temporaires ---"

    local db_name="myapp"
    local coll_name="temp"

    local exists=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').getCollectionNames().includes('${coll_name}')
    ")

    if [ "$exists" != "true" ]; then
        log_maintenance "INFO" "Collection ${db_name}.${coll_name} inexistante"
        return
    fi

    # Supprimer toute la collection temp si vide ou très ancienne
    local doc_count=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').${coll_name}.countDocuments()
    ")

    log_maintenance "INFO" "Documents dans temp: $doc_count"

    if [ "$doc_count" -gt 0 ]; then
        mongosh "$MONGO_URI" --quiet --eval "
            db.getSiblingDB('${db_name}').${coll_name}.deleteMany({})
        " >> "$MAINTENANCE_LOG" 2>&1

        log_maintenance "INFO" "Collection temp vidée"
    fi
}

main "$@"
```

## 5. Mise à jour des statistiques

```bash
#!/bin/bash
#
# Script: update-statistics.sh
# Description: Mise à jour des statistiques de collections
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.maintenance-env"
source "$SCRIPT_DIR/lib/maintenance-common.sh"

STATS_FILE="$ARCHIVE_DIR/collection_stats_$(date +%Y%m%d).json"

main() {
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Mise à jour des statistiques"
    log_maintenance "INFO" "=========================================="

    mkdir -p "$ARCHIVE_DIR"

    # Collecter les statistiques
    collect_database_stats
    collect_collection_stats
    collect_index_stats

    # Identifier les problèmes potentiels
    identify_issues

    log_maintenance "INFO" "Statistiques sauvegardées: $STATS_FILE"
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Mise à jour terminée"
    log_maintenance "INFO" "=========================================="
}

# Statistiques par base de données
collect_database_stats() {
    log_maintenance "INFO" "--- Statistiques bases de données ---"

    echo "{ \"databases\": [" > "$STATS_FILE"

    local databases=$(mongosh "$MONGO_URI" --quiet --eval "
        db.adminCommand('listDatabases').databases.forEach(function(d) {
            var stats = db.getSiblingDB(d.name).stats();
            print(JSON.stringify({
                name: d.name,
                sizeOnDisk: stats.dataSize,
                collections: stats.collections,
                indexes: stats.indexes,
                avgObjSize: stats.avgObjSize
            }));
        });
    ")

    local first=true
    while read -r db_stat; do
        [ -z "$db_stat" ] && continue

        [ "$first" = false ] && echo "," >> "$STATS_FILE"
        echo "$db_stat" >> "$STATS_FILE"
        first=false

        local db_name=$(echo "$db_stat" | jq -r '.name')
        local size=$(echo "$db_stat" | jq -r '.sizeOnDisk')
        local size_gb=$(echo "scale=2; $size / 1024 / 1024 / 1024" | bc)

        log_maintenance "INFO" "  $db_name: ${size_gb} GB"
    done <<< "$databases"

    echo "], \"collections\": [" >> "$STATS_FILE"
}

# Statistiques par collection
collect_collection_stats() {
    log_maintenance "INFO" "--- Statistiques collections (top 20) ---"

    local collections=$(mongosh "$MONGO_URI" --quiet --eval "
        var allStats = [];
        var dbs = db.adminCommand('listDatabases').databases;

        dbs.forEach(function(database) {
            if (['admin', 'config', 'local'].indexOf(database.name) !== -1) return;

            var dbObj = db.getSiblingDB(database.name);
            var colls = dbObj.getCollectionNames();

            colls.forEach(function(collName) {
                var stats = dbObj[collName].stats();
                allStats.push({
                    db: database.name,
                    collection: collName,
                    size: stats.size,
                    count: stats.count,
                    avgObjSize: stats.avgObjSize,
                    storageSize: stats.storageSize,
                    indexes: stats.nindexes
                });
            });
        });

        // Trier par taille
        allStats.sort(function(a, b) { return b.size - a.size; });

        // Top 20
        allStats.slice(0, 20).forEach(function(s) {
            print(JSON.stringify(s));
        });
    ")

    local first=true
    while read -r coll_stat; do
        [ -z "$coll_stat" ] && continue

        [ "$first" = false ] && echo "," >> "$STATS_FILE"
        echo "$coll_stat" >> "$STATS_FILE"
        first=false

        local ns=$(echo "$coll_stat" | jq -r '.db + "." + .collection')
        local size=$(echo "$coll_stat" | jq -r '.size')
        local count=$(echo "$coll_stat" | jq -r '.count')
        local size_mb=$(echo "scale=2; $size / 1024 / 1024" | bc)

        log_maintenance "INFO" "  $ns: ${size_mb} MB ($count docs)"
    done <<< "$collections"

    echo "], \"indexes\": [" >> "$STATS_FILE"
}

# Statistiques des index
collect_index_stats() {
    log_maintenance "INFO" "--- Statistiques index (top 20) ---"

    local indexes=$(mongosh "$MONGO_URI" --quiet --eval "
        var allIndexes = [];
        var dbs = db.adminCommand('listDatabases').databases;

        dbs.forEach(function(database) {
            if (['admin', 'config', 'local'].indexOf(database.name) !== -1) return;

            var dbObj = db.getSiblingDB(database.name);
            var colls = dbObj.getCollectionNames();

            colls.forEach(function(collName) {
                var stats = dbObj[collName].stats();
                if (stats.indexSizes) {
                    Object.keys(stats.indexSizes).forEach(function(idxName) {
                        allIndexes.push({
                            db: database.name,
                            collection: collName,
                            index: idxName,
                            size: stats.indexSizes[idxName]
                        });
                    });
                }
            });
        });

        // Trier par taille
        allIndexes.sort(function(a, b) { return b.size - a.size; });

        // Top 20
        allIndexes.slice(0, 20).forEach(function(i) {
            print(JSON.stringify(i));
        });
    ")

    local first=true
    while read -r idx_stat; do
        [ -z "$idx_stat" ] && continue

        [ "$first" = false ] && echo "," >> "$STATS_FILE"
        echo "$idx_stat" >> "$STATS_FILE"
        first=false

        local ns=$(echo "$idx_stat" | jq -r '.db + "." + .collection')
        local idx_name=$(echo "$idx_stat" | jq -r '.index')
        local size=$(echo "$idx_stat" | jq -r '.size')
        local size_mb=$(echo "scale=2; $size / 1024 / 1024" | bc)

        log_maintenance "INFO" "  $ns.$idx_name: ${size_mb} MB"
    done <<< "$indexes"

    echo "] }" >> "$STATS_FILE"
}

# Identifier les problèmes
identify_issues() {
    log_maintenance "INFO" "--- Problèmes potentiels ---"

    # Collections très volumineuses
    log_maintenance "INFO" "Collections > 50 GB:"
    jq -r '.collections[] | select(.size > 53687091200) | .db + "." + .collection + " (" + (.size/1073741824|tostring) + " GB)"' "$STATS_FILE" | while read -r line; do
        [ -z "$line" ] && continue
        log_maintenance "WARN" "  $line"
    done

    # Index très volumineux
    log_maintenance "INFO" "Index > 10 GB:"
    jq -r '.indexes[] | select(.size > 10737418240) | .db + "." + .collection + "." + .index + " (" + (.size/1073741824|tostring) + " GB)"' "$STATS_FILE" | while read -r line; do
        [ -z "$line" ] && continue
        log_maintenance "WARN" "  $line"
    done
}

main "$@"
```

## 6. Validation de l'intégrité

```bash
#!/bin/bash
#
# Script: validate-integrity.sh
# Description: Validation de l'intégrité des données
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.maintenance-env"
source "$SCRIPT_DIR/lib/maintenance-common.sh"

TARGET_DB="${1:-}"

main() {
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Validation de l'intégrité des données"
    log_maintenance "INFO" "=========================================="

    if [ -n "$TARGET_DB" ]; then
        validate_database "$TARGET_DB"
    else
        validate_all_databases
    fi

    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Validation terminée"
    log_maintenance "INFO" "=========================================="
}

# Valider une base de données
validate_database() {
    local db_name=$1

    log_maintenance "INFO" "Validation de la base: $db_name"

    local collections=$(mongosh "$MONGO_URI" --quiet --eval "
        db.getSiblingDB('${db_name}').getCollectionNames().forEach(function(c) {
            print(c);
        });
    ")

    local total=0
    local valid=0
    local errors=0

    while read -r coll_name; do
        [ -z "$coll_name" ] && continue
        [ "$coll_name" == "system.profile" ] && continue

        total=$((total + 1))

        if validate_collection "$db_name" "$coll_name"; then
            valid=$((valid + 1))
        else
            errors=$((errors + 1))
        fi
    done <<< "$collections"

    log_maintenance "INFO" "Résultat: $valid/$total valides, $errors erreurs"
}

# Valider une collection
validate_collection() {
    local db_name=$1
    local coll_name=$2

    log_maintenance "INFO" "Validation: ${db_name}.${coll_name}"

    local validation=$(mongosh "$MONGO_URI" --quiet --eval "
        var result = db.getSiblingDB('${db_name}').${coll_name}.validate({full: true});
        print(JSON.stringify({
            valid: result.valid,
            errors: result.errors || [],
            warnings: result.warnings || []
        }));
    " 2>&1)

    local is_valid=$(echo "$validation" | jq -r '.valid' 2>/dev/null)

    if [ "$is_valid" == "true" ]; then
        log_maintenance "INFO" "  ✓ Valide"
        return 0
    else
        log_maintenance "ERROR" "  ✗ Invalide"

        # Afficher les erreurs
        local errors=$(echo "$validation" | jq -r '.errors[]' 2>/dev/null)
        if [ -n "$errors" ]; then
            echo "$errors" | while read -r error; do
                log_maintenance "ERROR" "    $error"
            done
        fi

        return 1
    fi
}

# Valider toutes les bases
validate_all_databases() {
    log_maintenance "INFO" "Validation de toutes les bases de données..."

    local databases=$(mongosh "$MONGO_URI" --quiet --eval "
        db.adminCommand('listDatabases').databases.forEach(function(d) {
            if (['admin', 'config', 'local'].indexOf(d.name) === -1) {
                print(d.name);
            }
        });
    ")

    while read -r db_name; do
        [ -z "$db_name" ] && continue
        validate_database "$db_name"
    done <<< "$databases"
}

main "$@"
```

## 7. Script de maintenance complète

```bash
#!/bin/bash
#
# Script: full-maintenance.sh
# Description: Maintenance complète (orchestrateur)
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.maintenance-env"
source "$SCRIPT_DIR/lib/maintenance-common.sh"

START_TIME=$(date +%s)
REPORT_FILE="$LOG_DIR/maintenance_report_$(date +%Y%m%d_%H%M%S).txt"

main() {
    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "DÉBUT DE LA MAINTENANCE COMPLÈTE"
    log_maintenance "INFO" "=========================================="

    # Vérifications préliminaires
    check_maintenance_window || exit 1
    check_prerequisites

    # Notification de début
    send_notification "🔧 Maintenance MongoDB - Début" "La maintenance complète démarre"

    # Exécution des tâches
    run_task "Rotation des logs" "$SCRIPT_DIR/rotate-logs.sh"
    run_task "Nettoyage données expirées" "$SCRIPT_DIR/cleanup-expired-data.sh"
    run_task "Mise à jour statistiques" "$SCRIPT_DIR/update-statistics.sh"
    run_task "Validation intégrité" "$SCRIPT_DIR/validate-integrity.sh"
    run_task "Compactage collections" "$SCRIPT_DIR/compact-collections.sh"

    # Rapport final
    generate_final_report

    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "MAINTENANCE COMPLÈTE TERMINÉE"
    log_maintenance "INFO" "=========================================="

    # Notification de fin
    send_notification "✅ Maintenance MongoDB - Terminée" "$(cat $REPORT_FILE)"
}

# Vérifier les prérequis
check_prerequisites() {
    log_maintenance "INFO" "Vérification des prérequis..."

    # Connexion MongoDB
    if ! mongosh "$MONGO_URI" --quiet --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
        log_maintenance "ERROR" "Impossible de se connecter à MongoDB"
        exit 1
    fi

    # Espace disque
    local disk_free=$(df -h /var/lib/mongodb | awk 'NR==2 {print $4}')
    log_maintenance "INFO" "Espace disque disponible: $disk_free"

    log_maintenance "INFO" "Prérequis OK"
}

# Exécuter une tâche
run_task() {
    local task_name="$1"
    local task_script="$2"

    log_maintenance "INFO" "=========================================="
    log_maintenance "INFO" "Tâche: $task_name"
    log_maintenance "INFO" "=========================================="

    local task_start=$(date +%s)

    if [ -f "$task_script" ]; then
        if bash "$task_script"; then
            local task_end=$(date +%s)
            local task_duration=$((task_end - task_start))
            log_maintenance "INFO" "✓ Tâche terminée (durée: ${task_duration}s)"
            echo "✓ $task_name: ${task_duration}s" >> "$REPORT_FILE"
        else
            log_maintenance "ERROR" "✗ Échec de la tâche"
            echo "✗ $task_name: ÉCHEC" >> "$REPORT_FILE"
        fi
    else
        log_maintenance "WARN" "Script introuvable: $task_script"
        echo "⚠ $task_name: IGNORÉ (script absent)" >> "$REPORT_FILE"
    fi
}

# Générer le rapport final
generate_final_report() {
    local end_time=$(date +%s)
    local total_duration=$((end_time - START_TIME))
    local hours=$((total_duration / 3600))
    local minutes=$(((total_duration % 3600) / 60))
    local seconds=$((total_duration % 60))

    {
        echo ""
        echo "=========================================="
        echo "RAPPORT DE MAINTENANCE"
        echo "=========================================="
        echo "Date: $(date)"
        echo "Durée totale: ${hours}h ${minutes}m ${seconds}s"
        echo "=========================================="
    } >> "$REPORT_FILE"

    log_maintenance "INFO" "Rapport sauvegardé: $REPORT_FILE"
}

main "$@"
```

## Configuration systemd

```ini
# /etc/systemd/system/mongodb-maintenance.service
[Unit]
Description=MongoDB Maintenance Service
After=mongod.service
Requires=mongod.service

[Service]
Type=oneshot
User=mongodb
ExecStart=/opt/mongodb/scripts/full-maintenance.sh
StandardOutput=journal
StandardError=journal
```

```ini
# /etc/systemd/system/mongodb-maintenance.timer
[Unit]
Description=MongoDB Maintenance Timer
Requires=mongodb-maintenance.service

[Timer]
OnCalendar=Sun 02:00
Persistent=true

[Install]
WantedBy=timers.target
```

## Configuration Cron

```cron
# Rotation des logs (quotidien)
0 3 * * * /opt/mongodb/scripts/rotate-logs.sh >> /var/log/mongodb/cron-rotate.log 2>&1

# Nettoyage données (quotidien)
30 3 * * * /opt/mongodb/scripts/cleanup-expired-data.sh >> /var/log/mongodb/cron-cleanup.log 2>&1

# Statistiques (hebdomadaire)
0 4 * * 0 /opt/mongodb/scripts/update-statistics.sh >> /var/log/mongodb/cron-stats.log 2>&1

# Maintenance complète (mensuel - 1er dimanche)
0 2 1-7 * 0 /opt/mongodb/scripts/full-maintenance.sh >> /var/log/mongodb/cron-maintenance.log 2>&1
```

## Checklist de maintenance

### Quotidienne
- [ ] Rotation des logs
- [ ] Nettoyage des données expirées
- [ ] Vérification de l'espace disque
- [ ] Monitoring des performances

### Hebdomadaire
- [ ] Mise à jour des statistiques
- [ ] Analyse des slow queries
- [ ] Vérification de la fragmentation
- [ ] Revue des alertes

### Mensuelle
- [ ] Compactage des collections fragmentées
- [ ] Validation de l'intégrité
- [ ] Révision des index
- [ ] Test de restauration backup

### Trimestrielle
- [ ] Réindexation complète
- [ ] Audit de sécurité
- [ ] Revue de la modélisation
- [ ] Planification de capacité

## Bonnes pratiques

### Planification
- ✅ Exécuter pendant les fenêtres de maintenance
- ✅ Éviter les heures de pointe
- ✅ Planifier sur des noeuds SECONDARY quand possible
- ✅ Prévoir du temps tampon

### Sécurité
- ✅ Toujours faire un backup avant maintenance lourde
- ✅ Tester en développement d'abord
- ✅ Avoir un plan de rollback
- ✅ Documenter toutes les opérations

### Monitoring
- ✅ Surveiller l'impact sur les performances
- ✅ Logger toutes les opérations
- ✅ Envoyer des notifications de début/fin
- ✅ Conserver l'historique des maintenances

### Documentation
- ✅ Maintenir un changelog des maintenances
- ✅ Documenter les problèmes rencontrés
- ✅ Partager les leçons apprises
- ✅ Mettre à jour les runbooks

---


⏭️ [Playbooks Ansible](/annexes/scripts-automatisation/04-playbooks-ansible.md)
