🔝 Retour au [Sommaire](/SOMMAIRE.md)

# G.2 Scripts de monitoring

## Introduction

Le **monitoring proactif** est essentiel pour maintenir la santé et les performances de MongoDB. Cette section fournit des scripts prêts à l'emploi pour surveiller tous les aspects critiques de votre infrastructure MongoDB.

## Métriques clés à surveiller

| Catégorie | Métriques | Seuils critiques |
|-----------|-----------|------------------|
| **Performance** | Latence, throughput, queue length | >100ms, <1000 ops/s |
| **Ressources** | CPU, RAM, Disk I/O | >80%, >85%, >80% |
| **Réplication** | Lag, oplog window | >10s, <24h |
| **Connexions** | Active, available | >80% max |
| **Locks** | Queue time, deadlocks | >100ms |
| **Disque** | Utilisation, IOPS | >85%, baseline |

## Configuration commune

### Fichier de configuration

```bash
# /opt/mongodb/scripts/.monitoring-env
# Configuration pour les scripts de monitoring

# Connexion MongoDB
MONGO_HOST="localhost"
MONGO_PORT="27017"
MONGO_USER="monitor_user"
MONGO_PASSWORD="secure_password"
MONGO_AUTH_DB="admin"
MONGO_URI="mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${MONGO_HOST}:${MONGO_PORT}/${MONGO_AUTH_DB}"

# Seuils d'alerte
CPU_THRESHOLD=80
MEMORY_THRESHOLD=85
DISK_THRESHOLD=85
REPLICATION_LAG_THRESHOLD=10
CONNECTION_THRESHOLD=80
QUEUE_LENGTH_THRESHOLD=100

# Chemins
LOG_DIR="/var/log/mongodb/monitoring"
METRICS_DIR="/var/lib/mongodb/metrics"
ALERT_LOG="$LOG_DIR/alerts.log"

# Notifications
ENABLE_ALERTS=true
EMAIL_ADMIN="admin@example.com"
SLACK_WEBHOOK=""
PAGERDUTY_KEY=""

# Intervalle de collecte (secondes)
COLLECTION_INTERVAL=60
RETENTION_DAYS=30
```

### Bibliothèque de fonctions de monitoring

```bash
# /opt/mongodb/scripts/lib/monitoring-common.sh
# Fonctions communes pour le monitoring MongoDB

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

# Logging
log_metric() {
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$timestamp] $@" | tee -a "$LOG_DIR/metrics.log"
}

log_alert() {
    local level=$1
    shift
    local message="$@"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    echo "[$timestamp] [$level] $message" | tee -a "$ALERT_LOG"

    if [ "$ENABLE_ALERTS" == "true" ]; then
        send_alert "$level" "$message"
    fi
}

# Alertes
send_alert() {
    local severity=$1
    local message=$2

    # Email
    if [ -n "$EMAIL_ADMIN" ]; then
        echo "$message" | mail -s "[$severity] MongoDB Alert" "$EMAIL_ADMIN" 2>/dev/null || true
    fi

    # Slack
    if [ -n "$SLACK_WEBHOOK" ]; then
        local emoji="⚠️"
        [ "$severity" == "CRITICAL" ] && emoji="🚨"
        [ "$severity" == "WARNING" ] && emoji="⚠️"
        [ "$severity" == "INFO" ] && emoji="ℹ️"

        curl -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"${emoji} [${severity}] ${message}\"}" \
            "$SLACK_WEBHOOK" 2>/dev/null || true
    fi

    # PagerDuty
    if [ -n "$PAGERDUTY_KEY" ] && [ "$severity" == "CRITICAL" ]; then
        trigger_pagerduty "$message"
    fi
}

# PagerDuty
trigger_pagerduty() {
    local description="$1"

    curl -X POST https://events.pagerduty.com/v2/enqueue \
        -H 'Content-Type: application/json' \
        -d "{
            \"routing_key\": \"$PAGERDUTY_KEY\",
            \"event_action\": \"trigger\",
            \"payload\": {
                \"summary\": \"$description\",
                \"severity\": \"critical\",
                \"source\": \"mongodb-monitor\"
            }
        }" 2>/dev/null || true
}

# Requêtes MongoDB utilitaires
mongo_query() {
    mongosh "$MONGO_URI" --quiet --eval "$1" 2>/dev/null
}

# Vérifier si valeur dépasse seuil
check_threshold() {
    local value=$1
    local threshold=$2
    local name=$3

    if (( $(echo "$value > $threshold" | bc -l) )); then
        log_alert "WARNING" "$name: ${value}% (seuil: ${threshold}%)"
        return 1
    fi
    return 0
}

# Formater des bytes en unités lisibles
format_bytes() {
    local bytes=$1

    if [ -z "$bytes" ] || [ "$bytes" -eq 0 ]; then
        echo "0 B"
        return
    fi

    local units=("B" "KB" "MB" "GB" "TB")
    local unit=0
    local size=$bytes

    while (( $(echo "$size >= 1024" | bc -l) )) && [ $unit -lt 4 ]; do
        size=$(echo "scale=2; $size / 1024" | bc)
        unit=$((unit + 1))
    done

    echo "$size ${units[$unit]}"
}

# Collecter timestamp
get_timestamp() {
    date +%s
}

# Sauvegarder métriques en JSON
save_metric() {
    local metric_name=$1
    local metric_value=$2
    local timestamp=$(get_timestamp)

    local metric_file="$METRICS_DIR/${metric_name}.jsonl"

    echo "{\"timestamp\":$timestamp,\"value\":$metric_value,\"metric\":\"$metric_name\"}" >> "$metric_file"
}
```

## 1. Health Check complet

```bash
#!/bin/bash
#
# Script: health-check.sh
# Description: Vérification complète de la santé MongoDB
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.monitoring-env"
source "$SCRIPT_DIR/lib/monitoring-common.sh"

STATUS_OK=0
STATUS_WARN=1
STATUS_CRITICAL=2
EXIT_STATUS=$STATUS_OK

main() {
    echo "=========================================="
    echo "MongoDB Health Check - $(date)"
    echo "=========================================="

    # Vérifications
    check_mongodb_connection
    check_server_status
    check_replication_status
    check_disk_space
    check_connections
    check_locks

    # Résumé
    echo "=========================================="
    if [ $EXIT_STATUS -eq $STATUS_OK ]; then
        echo -e "${GREEN}✓ Tous les checks sont OK${NC}"
    elif [ $EXIT_STATUS -eq $STATUS_WARN ]; then
        echo -e "${YELLOW}⚠ Certains checks ont des warnings${NC}"
    else
        echo -e "${RED}✗ Checks critiques en échec${NC}"
    fi
    echo "=========================================="

    exit $EXIT_STATUS
}

# Vérifier la connexion MongoDB
check_mongodb_connection() {
    echo -n "Connexion MongoDB... "

    if mongosh "$MONGO_URI" --quiet --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
        echo -e "${GREEN}OK${NC}"
    else
        echo -e "${RED}FAILED${NC}"
        log_alert "CRITICAL" "Impossible de se connecter à MongoDB"
        EXIT_STATUS=$STATUS_CRITICAL
    fi
}

# Vérifier le statut du serveur
check_server_status() {
    echo -n "Server Status... "

    local uptime=$(mongo_query "db.serverStatus().uptime")
    local version=$(mongo_query "db.version()")

    if [ -n "$uptime" ]; then
        echo -e "${GREEN}OK${NC} (uptime: ${uptime}s, version: ${version})"
    else
        echo -e "${RED}FAILED${NC}"
        EXIT_STATUS=$STATUS_CRITICAL
    fi
}

# Vérifier l'état de la réplication
check_replication_status() {
    echo -n "Replication... "

    local is_replica=$(mongo_query "rs.status().ok" 2>/dev/null || echo "0")

    if [ "$is_replica" == "1" ]; then
        local primary=$(mongo_query "rs.isMaster().ismaster")
        local lag=$(mongo_query "
            var status = rs.status();
            var primary = status.members.find(m => m.state === 1);
            var self = status.members.find(m => m.self);
            if (primary && self && primary._id !== self._id) {
                var lag = (primary.optime.ts.getTime() - self.optime.ts.getTime());
                print(lag);
            } else {
                print(0);
            }
        " 2>/dev/null || echo "0")

        if [ "$primary" == "true" ]; then
            echo -e "${GREEN}OK${NC} (PRIMARY)"
        elif (( $(echo "$lag > $REPLICATION_LAG_THRESHOLD" | bc -l) )); then
            echo -e "${YELLOW}WARNING${NC} (lag: ${lag}s)"
            log_alert "WARNING" "Replication lag: ${lag}s"
            EXIT_STATUS=$STATUS_WARN
        else
            echo -e "${GREEN}OK${NC} (SECONDARY, lag: ${lag}s)"
        fi
    else
        echo -e "${BLUE}STANDALONE${NC}"
    fi
}

# Vérifier l'espace disque
check_disk_space() {
    echo -n "Disk Space... "

    local data_path=$(mongo_query "db.serverStatus().storageEngine.persistent ? db.serverStatus().storageEngine.persistent : '/var/lib/mongodb'")
    local disk_usage=$(df -h "$data_path" 2>/dev/null | awk 'NR==2 {print $5}' | sed 's/%//')

    if [ -z "$disk_usage" ]; then
        disk_usage=$(df -h /var/lib/mongodb | awk 'NR==2 {print $5}' | sed 's/%//')
    fi

    if [ -n "$disk_usage" ]; then
        if [ "$disk_usage" -gt "$DISK_THRESHOLD" ]; then
            echo -e "${RED}CRITICAL${NC} (${disk_usage}% used)"
            log_alert "CRITICAL" "Disk usage: ${disk_usage}%"
            EXIT_STATUS=$STATUS_CRITICAL
        elif [ "$disk_usage" -gt 70 ]; then
            echo -e "${YELLOW}WARNING${NC} (${disk_usage}% used)"
            EXIT_STATUS=$STATUS_WARN
        else
            echo -e "${GREEN}OK${NC} (${disk_usage}% used)"
        fi
    else
        echo -e "${YELLOW}UNKNOWN${NC}"
    fi
}

# Vérifier les connexions
check_connections() {
    echo -n "Connections... "

    local current=$(mongo_query "db.serverStatus().connections.current")
    local available=$(mongo_query "db.serverStatus().connections.available")
    local max=$((current + available))
    local usage_pct=$(echo "scale=2; ($current / $max) * 100" | bc)

    if (( $(echo "$usage_pct > $CONNECTION_THRESHOLD" | bc -l) )); then
        echo -e "${YELLOW}WARNING${NC} (${current}/${max} = ${usage_pct}%)"
        log_alert "WARNING" "Connection usage: ${usage_pct}%"
        EXIT_STATUS=$STATUS_WARN
    else
        echo -e "${GREEN}OK${NC} (${current}/${max})"
    fi
}

# Vérifier les locks
check_locks() {
    echo -n "Locks... "

    local queue_readers=$(mongo_query "db.serverStatus().globalLock.currentQueue.readers" 2>/dev/null || echo "0")
    local queue_writers=$(mongo_query "db.serverStatus().globalLock.currentQueue.writers" 2>/dev/null || echo "0")

    if [ "$queue_readers" -gt "$QUEUE_LENGTH_THRESHOLD" ] || [ "$queue_writers" -gt "$QUEUE_LENGTH_THRESHOLD" ]; then
        echo -e "${YELLOW}WARNING${NC} (readers: $queue_readers, writers: $queue_writers)"
        log_alert "WARNING" "Lock queue: readers=$queue_readers, writers=$queue_writers"
        EXIT_STATUS=$STATUS_WARN
    else
        echo -e "${GREEN}OK${NC} (readers: $queue_readers, writers: $queue_writers)"
    fi
}

main "$@"
```

## 2. Monitoring des performances

```bash
#!/bin/bash
#
# Script: monitor-performance.sh
# Description: Collecte des métriques de performance
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.monitoring-env"
source "$SCRIPT_DIR/lib/monitoring-common.sh"

main() {
    log_metric "=== Performance Monitoring - $(date) ==="

    # Collecter les métriques
    collect_operation_metrics
    collect_memory_metrics
    collect_disk_metrics
    collect_network_metrics
    collect_cache_metrics

    log_metric "=== Collecte terminée ==="
}

# Métriques d'opérations
collect_operation_metrics() {
    log_metric "--- Opérations ---"

    local ops=$(mongo_query "
        var status = db.serverStatus();
        print(JSON.stringify({
            insert: status.opcounters.insert,
            query: status.opcounters.query,
            update: status.opcounters.update,
            delete: status.opcounters.delete,
            command: status.opcounters.command
        }));
    ")

    log_metric "Operations: $ops"

    # Sauvegarder pour graphiques
    echo "$ops" | jq -r 'to_entries[] | "\(.key),\(.value)"' | while IFS=, read -r op_type op_count; do
        save_metric "ops_${op_type}" "$op_count"
    done
}

# Métriques mémoire
collect_memory_metrics() {
    log_metric "--- Mémoire ---"

    local mem_info=$(mongo_query "
        var status = db.serverStatus();
        print(JSON.stringify({
            resident: status.mem.resident,
            virtual: status.mem.virtual,
            mapped: status.mem.mapped || 0,
            mappedWithJournal: status.mem.mappedWithJournal || 0
        }));
    ")

    log_metric "Memory: $mem_info"

    # Vérifier utilisation système
    local mem_used=$(free | awk 'NR==2 {printf "%.0f", $3/$2*100}')
    log_metric "System Memory Usage: ${mem_used}%"

    if [ "$mem_used" -gt "$MEMORY_THRESHOLD" ]; then
        log_alert "WARNING" "Memory usage: ${mem_used}%"
    fi

    save_metric "memory_usage" "$mem_used"
}

# Métriques disque
collect_disk_metrics() {
    log_metric "--- Disque ---"

    # IOPS
    local reads=$(mongo_query "db.serverStatus().wiredTiger.data.\"block-manager\".\"blocks read\"")
    local writes=$(mongo_query "db.serverStatus().wiredTiger.data.\"block-manager\".\"blocks written\"")

    log_metric "Disk I/O - Reads: $reads, Writes: $writes"

    # Latence
    local fsync_time=$(mongo_query "db.serverStatus().backgroundFlushing ? db.serverStatus().backgroundFlushing.average_ms : 0")
    log_metric "Fsync Latency: ${fsync_time}ms"

    save_metric "disk_reads" "$reads"
    save_metric "disk_writes" "$writes"
    save_metric "fsync_latency" "$fsync_time"
}

# Métriques réseau
collect_network_metrics() {
    log_metric "--- Réseau ---"

    local net_info=$(mongo_query "
        var status = db.serverStatus();
        print(JSON.stringify({
            bytesIn: status.network.bytesIn,
            bytesOut: status.network.bytesOut,
            numRequests: status.network.numRequests
        }));
    ")

    log_metric "Network: $net_info"
}

# Métriques de cache WiredTiger
collect_cache_metrics() {
    log_metric "--- Cache WiredTiger ---"

    local cache_info=$(mongo_query "
        var wt = db.serverStatus().wiredTiger;
        if (wt && wt.cache) {
            var bytesCurrentlyInCache = wt.cache['bytes currently in the cache'];
            var maxBytesConfigured = wt.cache['maximum bytes configured'];
            var usage = (bytesCurrentlyInCache / maxBytesConfigured * 100).toFixed(2);
            print(JSON.stringify({
                currentMB: (bytesCurrentlyInCache / 1024 / 1024).toFixed(2),
                maxMB: (maxBytesConfigured / 1024 / 1024).toFixed(2),
                usagePct: usage,
                dirtyPct: (wt.cache['tracked dirty bytes in the cache'] / bytesCurrentlyInCache * 100).toFixed(2)
            }));
        }
    ")

    log_metric "Cache: $cache_info"

    # Extraire le pourcentage d'utilisation
    local cache_usage=$(echo "$cache_info" | jq -r '.usagePct' 2>/dev/null || echo "0")
    save_metric "cache_usage" "$cache_usage"
}

main "$@"
```

## 3. Monitoring de la réplication

```bash
#!/bin/bash
#
# Script: monitor-replication.sh
# Description: Surveillance de l'état de réplication
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.monitoring-env"
source "$SCRIPT_DIR/lib/monitoring-common.sh"

main() {
    log_metric "=== Replication Monitoring - $(date) ==="

    # Vérifier si c'est un replica set
    local is_replica=$(mongo_query "rs.status().ok" 2>/dev/null || echo "0")

    if [ "$is_replica" != "1" ]; then
        log_metric "Instance standalone - pas de réplication"
        exit 0
    fi

    check_replica_set_health
    check_replication_lag
    check_oplog_window
    check_member_states

    log_metric "=== Monitoring réplication terminé ==="
}

# Santé du replica set
check_replica_set_health() {
    log_metric "--- Santé du Replica Set ---"

    local rs_name=$(mongo_query "rs.status().set")
    local rs_members=$(mongo_query "rs.status().members.length")

    log_metric "Replica Set: $rs_name"
    log_metric "Membres: $rs_members"

    # Vérifier qu'il y a un primary
    local has_primary=$(mongo_query "rs.status().members.filter(m => m.state === 1).length > 0")

    if [ "$has_primary" != "true" ]; then
        log_alert "CRITICAL" "Aucun PRIMARY dans le replica set!"
    fi
}

# Lag de réplication
check_replication_lag() {
    log_metric "--- Replication Lag ---"

    # Obtenir le lag pour chaque membre
    local lag_info=$(mongo_query "
        var status = rs.status();
        var primary = status.members.find(m => m.state === 1);

        if (!primary) {
            print('NO_PRIMARY');
        } else {
            status.members.forEach(function(member) {
                if (member.state === 2) { // SECONDARY
                    var lag = (primary.optime.ts.getTime() - member.optime.ts.getTime());
                    print(member.name + ':' + lag);
                }
            });
        }
    ")

    if [ "$lag_info" == "NO_PRIMARY" ]; then
        log_alert "CRITICAL" "Aucun PRIMARY détecté"
        return
    fi

    echo "$lag_info" | while IFS=: read -r member lag; do
        [ -z "$member" ] && continue

        log_metric "  $member: ${lag}s"

        if (( $(echo "$lag > $REPLICATION_LAG_THRESHOLD" | bc -l) )); then
            log_alert "WARNING" "Replication lag sur $member: ${lag}s"
        fi

        save_metric "replication_lag_$(echo $member | tr ':.' '_')" "$lag"
    done
}

# Fenêtre oplog
check_oplog_window() {
    log_metric "--- Oplog Window ---"

    local oplog_info=$(mongo_query "
        var oplog = db.getSiblingDB('local').oplog.rs;
        var first = oplog.find().sort({ts: 1}).limit(1).toArray()[0];
        var last = oplog.find().sort({ts: -1}).limit(1).toArray()[0];

        if (first && last) {
            var window = (last.ts.getTime() - first.ts.getTime());
            var windowHours = (window / 3600).toFixed(2);
            print(JSON.stringify({
                windowSeconds: window,
                windowHours: windowHours,
                firstTs: first.ts,
                lastTs: last.ts
            }));
        }
    ")

    local window_hours=$(echo "$oplog_info" | jq -r '.windowHours' 2>/dev/null || echo "0")

    log_metric "Oplog Window: ${window_hours} heures"

    if (( $(echo "$window_hours < 24" | bc -l) )); then
        log_alert "WARNING" "Fenêtre oplog courte: ${window_hours}h (recommandé: >24h)"
    fi

    save_metric "oplog_window_hours" "$window_hours"
}

# États des membres
check_member_states() {
    log_metric "--- États des Membres ---"

    local members=$(mongo_query "
        rs.status().members.forEach(function(m) {
            print(m.name + ':' + m.stateStr + ':' + m.health);
        });
    ")

    echo "$members" | while IFS=: read -r name state health; do
        [ -z "$name" ] && continue

        if [ "$health" != "1" ]; then
            log_alert "CRITICAL" "Membre $name non sain: état=$state, health=$health"
        else
            log_metric "  $name: $state (healthy)"
        fi
    done
}

main "$@"
```

## 4. Monitoring du sharding

```bash
#!/bin/bash
#
# Script: monitor-sharding.sh
# Description: Surveillance d'un cluster shardé
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.monitoring-env"
source "$SCRIPT_DIR/lib/monitoring-common.sh"

MONGOS_URI="${MONGOS_URI:-$MONGO_URI}"

main() {
    log_metric "=== Sharding Monitoring - $(date) ==="

    # Vérifier si c'est un cluster shardé
    local is_sharded=$(mongosh "$MONGOS_URI" --quiet --eval "sh.status().ok" 2>/dev/null || echo "0")

    if [ "$is_sharded" != "1" ]; then
        log_metric "Instance non shardée"
        exit 0
    fi

    check_balancer_status
    check_chunk_distribution
    check_shard_health
    check_jumbo_chunks

    log_metric "=== Monitoring sharding terminé ==="
}

# Statut du balancer
check_balancer_status() {
    log_metric "--- Balancer ---"

    local balancer_running=$(mongosh "$MONGOS_URI" --quiet --eval "sh.isBalancerRunning()")
    local balancer_enabled=$(mongosh "$MONGOS_URI" --quiet --eval "
        var config = db.getSiblingDB('config');
        var settings = config.settings.findOne({_id: 'balancer'});
        print(settings && settings.stopped ? false : true);
    ")

    log_metric "Balancer Running: $balancer_running"
    log_metric "Balancer Enabled: $balancer_enabled"

    if [ "$balancer_enabled" == "true" ] && [ "$balancer_running" != "true" ]; then
        log_alert "WARNING" "Balancer activé mais non en cours d'exécution"
    fi
}

# Distribution des chunks
check_chunk_distribution() {
    log_metric "--- Distribution des Chunks ---"

    local chunk_dist=$(mongosh "$MONGOS_URI" --quiet --eval "
        var config = db.getSiblingDB('config');
        var chunks = config.chunks.aggregate([
            {
                \$group: {
                    _id: '\$shard',
                    count: { \$sum: 1 }
                }
            },
            { \$sort: { count: -1 } }
        ]).toArray();

        chunks.forEach(function(c) {
            print(c._id + ':' + c.count);
        });
    ")

    # Analyser la distribution
    declare -A shard_chunks
    local total_chunks=0
    local max_chunks=0
    local min_chunks=999999

    while IFS=: read -r shard count; do
        [ -z "$shard" ] && continue

        shard_chunks[$shard]=$count
        total_chunks=$((total_chunks + count))
        [ $count -gt $max_chunks ] && max_chunks=$count
        [ $count -lt $min_chunks ] && min_chunks=$count

        log_metric "  $shard: $count chunks"
    done <<< "$chunk_dist"

    # Vérifier le déséquilibre
    if [ $total_chunks -gt 0 ]; then
        local imbalance=$(echo "scale=2; ($max_chunks - $min_chunks) / $total_chunks * 100" | bc)
        log_metric "Imbalance: ${imbalance}%"

        if (( $(echo "$imbalance > 20" | bc -l) )); then
            log_alert "WARNING" "Déséquilibre de chunks important: ${imbalance}%"
        fi

        save_metric "chunk_imbalance" "$imbalance"
    fi
}

# Santé des shards
check_shard_health() {
    log_metric "--- Santé des Shards ---"

    local shards=$(mongosh "$MONGOS_URI" --quiet --eval "
        var config = db.getSiblingDB('config');
        config.shards.find().forEach(function(s) {
            print(s._id + ':' + s.host);
        });
    ")

    while IFS=: read -r shard_id shard_host; do
        [ -z "$shard_id" ] && continue

        # Extraire le host principal
        local host=$(echo "$shard_host" | sed 's|.*\/||' | cut -d',' -f1)
        local shard_uri="mongodb://${MONGO_USER}:${MONGO_PASSWORD}@${host}/${MONGO_AUTH_DB}"

        # Tester la connexion
        if mongosh "$shard_uri" --quiet --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
            log_metric "  $shard_id: OK"
        else
            log_alert "CRITICAL" "Shard $shard_id injoignable"
        fi
    done <<< "$shards"
}

# Jumbo chunks
check_jumbo_chunks() {
    log_metric "--- Jumbo Chunks ---"

    local jumbo_count=$(mongosh "$MONGOS_URI" --quiet --eval "
        var config = db.getSiblingDB('config');
        print(config.chunks.countDocuments({jumbo: true}));
    ")

    log_metric "Jumbo Chunks: $jumbo_count"

    if [ "$jumbo_count" -gt 0 ]; then
        log_alert "WARNING" "$jumbo_count jumbo chunks détectés"

        # Lister les jumbo chunks
        local jumbo_list=$(mongosh "$MONGOS_URI" --quiet --eval "
            var config = db.getSiblingDB('config');
            config.chunks.find({jumbo: true}, {ns: 1, shard: 1}).forEach(function(c) {
                print(c.ns + ':' + c.shard);
            });
        ")

        echo "$jumbo_list" | while IFS=: read -r ns shard; do
            [ -z "$ns" ] && continue
            log_metric "    $ns on $shard"
        done
    fi

    save_metric "jumbo_chunks" "$jumbo_count"
}

main "$@"
```

## 5. Monitoring des slow queries

```bash
#!/bin/bash
#
# Script: monitor-slow-queries.sh
# Description: Détection et analyse des requêtes lentes
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.monitoring-env"
source "$SCRIPT_DIR/lib/monitoring-common.sh"

SLOW_QUERY_THRESHOLD="${SLOW_QUERY_THRESHOLD:-100}"  # ms
REPORT_FILE="$LOG_DIR/slow-queries-$(date +%Y%m%d).log"

main() {
    log_metric "=== Slow Queries Monitoring - $(date) ==="

    # Activer le profiler si nécessaire
    enable_profiler

    # Analyser les slow queries
    analyze_slow_queries

    # Générer un rapport
    generate_report

    log_metric "=== Monitoring terminé ==="
}

# Activer le profiler
enable_profiler() {
    log_metric "Vérification du profiler..."

    # Vérifier le niveau actuel
    local profiler_level=$(mongo_query "db.getProfilingLevel()")

    if [ "$profiler_level" == "0" ]; then
        log_metric "Activation du profiler (niveau 1, seuil: ${SLOW_QUERY_THRESHOLD}ms)"
        mongo_query "db.setProfilingLevel(1, { slowms: $SLOW_QUERY_THRESHOLD })" >/dev/null
    else
        log_metric "Profiler déjà actif (niveau: $profiler_level)"
    fi
}

# Analyser les requêtes lentes
analyze_slow_queries() {
    log_metric "--- Requêtes Lentes (dernière heure) ---"

    local one_hour_ago=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S)

    local slow_queries=$(mongo_query "
        db.getSiblingDB('admin').system.profile.aggregate([
            {
                \$match: {
                    ts: { \$gte: new Date('$one_hour_ago') },
                    millis: { \$gte: $SLOW_QUERY_THRESHOLD }
                }
            },
            {
                \$group: {
                    _id: {
                        ns: '\$ns',
                        op: '\$op'
                    },
                    count: { \$sum: 1 },
                    avgMs: { \$avg: '\$millis' },
                    maxMs: { \$max: '\$millis' }
                }
            },
            { \$sort: { count: -1 } },
            { \$limit: 10 }
        ]).toArray();

        slow_queries.forEach(function(q) {
            print(q._id.ns + ':' + q._id.op + ':' + q.count + ':' + q.avgMs.toFixed(2) + ':' + q.maxMs);
        });
    ")

    if [ -z "$slow_queries" ]; then
        log_metric "Aucune requête lente détectée"
        return
    fi

    echo "Namespace:Operation:Count:AvgMs:MaxMs" > "$REPORT_FILE"
    echo "$slow_queries" | while IFS=: read -r ns op count avg_ms max_ms; do
        [ -z "$ns" ] && continue

        log_metric "  $ns ($op): count=$count, avg=${avg_ms}ms, max=${max_ms}ms"
        echo "$ns:$op:$count:$avg_ms:$max_ms" >> "$REPORT_FILE"

        # Alerte si trop de slow queries
        if [ "$count" -gt 100 ]; then
            log_alert "WARNING" "Slow queries nombreuses: $ns ($op) - $count occurrences"
        fi
    done
}

# Générer un rapport
generate_report() {
    log_metric "--- Rapport généré ---"
    log_metric "Fichier: $REPORT_FILE"

    # Top 5 des requêtes les plus lentes
    local top_slow=$(mongo_query "
        db.getSiblingDB('admin').system.profile.find(
            { millis: { \$gte: $SLOW_QUERY_THRESHOLD } }
        ).sort({ millis: -1 }).limit(5).toArray();

        top_slow.forEach(function(q) {
            print('Duration: ' + q.millis + 'ms, NS: ' + q.ns + ', Op: ' + q.op);
        });
    ")

    if [ -n "$top_slow" ]; then
        log_metric "Top 5 des requêtes les plus lentes:"
        echo "$top_slow" | while read -r line; do
            log_metric "  $line"
        done
    fi
}

main "$@"
```

## 6. Collecteur de métriques pour Prometheus

```bash
#!/bin/bash
#
# Script: prometheus-exporter.sh
# Description: Export des métriques au format Prometheus
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.monitoring-env"
source "$SCRIPT_DIR/lib/monitoring-common.sh"

METRICS_FILE="/var/lib/node_exporter/textfile_collector/mongodb.prom"

main() {
    # Créer le fichier de métriques temporaire
    local temp_file=$(mktemp)

    # Collecter les métriques
    collect_server_metrics "$temp_file"
    collect_connection_metrics "$temp_file"
    collect_operation_metrics "$temp_file"
    collect_replication_metrics "$temp_file"

    # Atomiquement remplacer le fichier
    mv "$temp_file" "$METRICS_FILE"
}

# Métriques serveur
collect_server_metrics() {
    local file=$1

    cat >> "$file" << EOF
# HELP mongodb_up MongoDB instance is up
# TYPE mongodb_up gauge
mongodb_up 1

# HELP mongodb_uptime_seconds MongoDB uptime in seconds
# TYPE mongodb_uptime_seconds counter
EOF

    local uptime=$(mongo_query "db.serverStatus().uptime")
    echo "mongodb_uptime_seconds $uptime" >> "$file"
}

# Métriques de connexions
collect_connection_metrics() {
    local file=$1

    local current=$(mongo_query "db.serverStatus().connections.current")
    local available=$(mongo_query "db.serverStatus().connections.available")

    cat >> "$file" << EOF

# HELP mongodb_connections_current Current connections
# TYPE mongodb_connections_current gauge
mongodb_connections_current $current

# HELP mongodb_connections_available Available connections
# TYPE mongodb_connections_available gauge
mongodb_connections_available $available
EOF
}

# Métriques d'opérations
collect_operation_metrics() {
    local file=$1

    cat >> "$file" << EOF

# HELP mongodb_opcounters_total Operation counters
# TYPE mongodb_opcounters_total counter
EOF

    mongo_query "
        var ops = db.serverStatus().opcounters;
        print('mongodb_opcounters_total{type=\"insert\"} ' + ops.insert);
        print('mongodb_opcounters_total{type=\"query\"} ' + ops.query);
        print('mongodb_opcounters_total{type=\"update\"} ' + ops.update);
        print('mongodb_opcounters_total{type=\"delete\"} ' + ops.delete);
        print('mongodb_opcounters_total{type=\"command\"} ' + ops.command);
    " >> "$file"
}

# Métriques de réplication
collect_replication_metrics() {
    local file=$1

    local is_replica=$(mongo_query "rs.status().ok" 2>/dev/null || echo "0")

    if [ "$is_replica" == "1" ]; then
        cat >> "$file" << EOF

# HELP mongodb_replication_lag_seconds Replication lag in seconds
# TYPE mongodb_replication_lag_seconds gauge
EOF

        local lag=$(mongo_query "
            var status = rs.status();
            var primary = status.members.find(m => m.state === 1);
            var self = status.members.find(m => m.self);
            if (primary && self && primary._id !== self._id) {
                print((primary.optime.ts.getTime() - self.optime.ts.getTime()));
            } else {
                print(0);
            }
        " 2>/dev/null || echo "0")

        echo "mongodb_replication_lag_seconds $lag" >> "$file"
    fi
}

main "$@"
```

## 7. Dashboard de synthèse

```bash
#!/bin/bash
#
# Script: monitoring-dashboard.sh
# Description: Dashboard ASCII de monitoring en temps réel
#

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "$SCRIPT_DIR/.monitoring-env"
source "$SCRIPT_DIR/lib/monitoring-common.sh"

REFRESH_INTERVAL=5

main() {
    while true; do
        clear
        display_dashboard
        sleep $REFRESH_INTERVAL
    done
}

display_dashboard() {
    echo "╔═══════════════════════════════════════════════════════════════════╗"
    echo "║              MongoDB Monitoring Dashboard                         ║"
    echo "║                  $(date '+%Y-%m-%d %H:%M:%S')                            ║"
    echo "╚═══════════════════════════════════════════════════════════════════╝"
    echo ""

    # Statut général
    display_general_status
    echo ""

    # Connexions
    display_connections
    echo ""

    # Opérations
    display_operations
    echo ""

    # Réplication
    display_replication
    echo ""

    # Ressources
    display_resources
    echo ""

    echo "Actualisation toutes les ${REFRESH_INTERVAL}s - Ctrl+C pour quitter"
}

display_general_status() {
    echo "┌─ Statut Général ──────────────────────────────────────────────────┐"

    local version=$(mongo_query "db.version()")
    local uptime=$(mongo_query "db.serverStatus().uptime")
    local uptime_human=$(printf '%dd %dh %dm' $((uptime/86400)) $((uptime%86400/3600)) $((uptime%3600/60)))

    echo "│ Version: $version"
    echo "│ Uptime: $uptime_human"
    echo "└────────────────────────────────────────────────────────────────────┘"
}

display_connections() {
    echo "┌─ Connexions ──────────────────────────────────────────────────────┐"

    local current=$(mongo_query "db.serverStatus().connections.current")
    local available=$(mongo_query "db.serverStatus().connections.available")
    local max=$((current + available))
    local usage_pct=$(echo "scale=0; ($current / $max) * 100" | bc)

    # Barre de progression
    local bar_length=50
    local filled=$((usage_pct * bar_length / 100))
    local empty=$((bar_length - filled))

    local color=$GREEN
    [ $usage_pct -gt 70 ] && color=$YELLOW
    [ $usage_pct -gt 85 ] && color=$RED

    printf "│ Actives: %d / %d (%d%%)\n" $current $max $usage_pct
    printf "│ ${color}["
    printf "%${filled}s" | tr ' ' '█'
    printf "%${empty}s" | tr ' ' '░'
    printf "]${NC}\n"
    echo "└────────────────────────────────────────────────────────────────────┘"
}

display_operations() {
    echo "┌─ Opérations (total) ──────────────────────────────────────────────┐"

    local ops=$(mongo_query "
        var ops = db.serverStatus().opcounters;
        print(ops.insert + ',' + ops.query + ',' + ops.update + ',' + ops.delete);
    ")

    IFS=',' read -r insert query update delete <<< "$ops"

    printf "│ Insert: %-15s Query: %-15s\n" "$(format_number $insert)" "$(format_number $query)"
    printf "│ Update: %-15s Delete: %-15s\n" "$(format_number $update)" "$(format_number $delete)"
    echo "└────────────────────────────────────────────────────────────────────┘"
}

display_replication() {
    echo "┌─ Réplication ─────────────────────────────────────────────────────┐"

    local is_replica=$(mongo_query "rs.status().ok" 2>/dev/null || echo "0")

    if [ "$is_replica" == "1" ]; then
        local is_primary=$(mongo_query "rs.isMaster().ismaster")
        local rs_name=$(mongo_query "rs.status().set")

        if [ "$is_primary" == "true" ]; then
            echo "│ Rôle: ${GREEN}PRIMARY${NC}"
        else
            local lag=$(mongo_query "
                var status = rs.status();
                var primary = status.members.find(m => m.state === 1);
                var self = status.members.find(m => m.self);
                if (primary && self) {
                    print((primary.optime.ts.getTime() - self.optime.ts.getTime()));
                } else {
                    print(0);
                }
            " 2>/dev/null || echo "0")

            local lag_color=$GREEN
            [ "$lag" -gt 5 ] && lag_color=$YELLOW
            [ "$lag" -gt 10 ] && lag_color=$RED

            printf "│ Rôle: ${BLUE}SECONDARY${NC}  Lag: ${lag_color}${lag}s${NC}\n"
        fi
        echo "│ Replica Set: $rs_name"
    else
        echo "│ Mode: ${BLUE}STANDALONE${NC}"
    fi

    echo "└────────────────────────────────────────────────────────────────────┘"
}

display_resources() {
    echo "┌─ Ressources ──────────────────────────────────────────────────────┐"

    # CPU
    local cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
    printf "│ CPU: %.1f%%\n" $cpu_usage

    # Mémoire
    local mem_used=$(free | awk 'NR==2 {printf "%.1f", $3/$2*100}')
    printf "│ Mémoire: %.1f%%\n" $mem_used

    # Disque
    local disk_used=$(df -h /var/lib/mongodb 2>/dev/null | awk 'NR==2 {print $5}' | sed 's/%//')
    [ -z "$disk_used" ] && disk_used=0
    printf "│ Disque: %d%%\n" $disk_used

    echo "└────────────────────────────────────────────────────────────────────┘"
}

format_number() {
    printf "%'d" $1 2>/dev/null || echo $1
}

main "$@"
```

## Configuration systemd pour monitoring continu

```ini
# /etc/systemd/system/mongodb-monitor.service
[Unit]
Description=MongoDB Monitoring Service
After=mongod.service
Requires=mongod.service

[Service]
Type=simple
User=mongodb
ExecStart=/opt/mongodb/scripts/monitor-performance.sh
Restart=always
RestartSec=60

[Install]
WantedBy=multi-user.target
```

```ini
# /etc/systemd/system/mongodb-monitor.timer
[Unit]
Description=MongoDB Monitoring Timer
Requires=mongodb-monitor.service

[Timer]
OnBootSec=5min
OnUnitActiveSec=5min
Persistent=true

[Install]
WantedBy=timers.target
```

## Configuration Cron

```cron
# Monitoring toutes les 5 minutes
*/5 * * * * /opt/mongodb/scripts/health-check.sh >> /var/log/mongodb/health-check.log 2>&1

# Performance toutes les minutes
* * * * * /opt/mongodb/scripts/monitor-performance.sh >> /var/log/mongodb/performance.log 2>&1

# Réplication toutes les 2 minutes
*/2 * * * * /opt/mongodb/scripts/monitor-replication.sh >> /var/log/mongodb/replication.log 2>&1

# Slow queries toutes les heures
0 * * * * /opt/mongodb/scripts/monitor-slow-queries.sh >> /var/log/mongodb/slow-queries.log 2>&1

# Export Prometheus toutes les 30 secondes (via systemd)
# Voir fichier .timer ci-dessus
```

## Checklist de déploiement

- [ ] Utilisateur MongoDB avec droits de lecture créé
- [ ] Répertoires de logs et métriques créés
- [ ] Variables d'environnement configurées
- [ ] Scripts testés manuellement
- [ ] Notifications configurées et testées
- [ ] Tâches cron/systemd planifiées
- [ ] Dashboards Grafana créés (si applicable)
- [ ] Documentation des seuils d'alerte
- [ ] Procédure d'escalade définie
- [ ] Rétention des logs configurée

## Intégration avec des outils externes

### Prometheus + Grafana

1. Installer le MongoDB Exporter
2. Configurer Prometheus pour scraper les métriques
3. Importer les dashboards Grafana officiels

### Datadog

```bash
# Installation de l'agent Datadog pour MongoDB
DD_AGENT_MAJOR_VERSION=7 DD_API_KEY=<YOUR_KEY> bash -c "$(curl -L https://s3.amazonaws.com/dd-agent/scripts/install_script.sh)"

# Configuration
sudo vi /etc/datadog-agent/conf.d/mongo.d/conf.yaml
```

### New Relic

```bash
# Installation du plugin MongoDB
newrelic-infra-ctl install integration com.newrelic.mongodb
```

## Bonnes pratiques

### Surveillance
- ✅ Monitorer 24/7 avec alertes automatiques
- ✅ Définir des seuils adaptés à votre charge
- ✅ Archiver les métriques historiques
- ✅ Corréler avec les métriques applicatives

### Alertes
- ✅ Éviter la fatigue d'alerte (alert fatigue)
- ✅ Définir des niveaux de sévérité clairs
- ✅ Documenter les procédures de réponse
- ✅ Tester régulièrement les notifications

### Performance
- ✅ Limiter l'impact du monitoring sur la production
- ✅ Utiliser des membres SECONDARY pour les requêtes lourdes
- ✅ Optimiser les scripts pour minimiser les requêtes
- ✅ Mettre en cache les métriques peu volatiles

---


⏭️ [Scripts de maintenance](/annexes/scripts-automatisation/03-scripts-maintenance.md)
