🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.11 Tests de Restauration

## Introduction

Les tests de restauration constituent le maillon essentiel mais souvent négligé d'une stratégie de continuité d'activité robuste. Un backup non testé n'est qu'une **illusion de sécurité**. L'histoire des systèmes d'information est jalonnée d'incidents critiques où des organisations ont découvert trop tard que leurs backups étaient corrompus, incomplets ou impossibles à restaurer.

Les tests de restauration réguliers garantissent non seulement que les backups sont fonctionnels, mais aussi que l'équipe maîtrise les procédures, que les délais de récupération sont réalistes, et que la documentation est à jour. Cette section explore les stratégies, procédures et bonnes pratiques pour établir un programme de tests de restauration efficace et automatisé.

## Pourquoi Tester les Restaurations ?

### Statistiques Alarmantes

```yaml
Études de Cas Réelles:

GitLab (2017):
  incident: "Suppression accidentelle de base de données"
  problème: "5 méthodes de backup différentes - aucune fonctionnelle"
  impact: "Perte de 6 heures de données"
  cause: "Backups jamais testés"
  leçon: "Tests réguliers obligatoires depuis"

Knight Capital (2012):
  incident: "Bug de trading algorithmique"
  problème: "Restauration trop lente"
  impact: "$440 millions de pertes en 45 minutes"
  cause: "Procédures de récupération non testées"

Code Spaces (2014):
  incident: "Attaque ransomware"
  problème: "Backups dans même compte AWS - supprimés"
  impact: "Entreprise fermée définitivement"
  cause: "Stratégie de backup non testée end-to-end"

Statistiques Générales:
  backups_non_testés: "60% des organisations"
  backups_défaillants: "34% lors du premier test"
  restauration_partielle: "43% des cas"
  rto_non_respecté: "77% des premiers tests"
```

### Raisons de Tester

```
┌────────────────────────────────────────────────────────────┐
│           Pourquoi Tester les Restaurations ?              │
│                                                            │
│  1. VALIDATION TECHNIQUE                                   │
│     ✓ Backups non corrompus                                │
│     ✓ Procédures fonctionnelles                            │
│     ✓ Compatibilité versions                               │
│     ✓ Intégrité des données                                │
│                                                            │
│  2. VALIDATION OPÉRATIONNELLE                              │
│     ✓ RTO réaliste et atteignable                          │
│     ✓ RPO vérifié                                          │
│     ✓ Procédures à jour                                    │
│     ✓ Documentation complète                               │
│                                                            │
│  3. VALIDATION HUMAINE                                     │
│     ✓ Équipe formée                                        │
│     ✓ Compétences maintenues                               │
│     ✓ Stress management                                    │
│     ✓ Communication rodée                                  │
│                                                            │
│  4. VALIDATION BUSINESS                                    │
│     ✓ Données cohérentes                                   │
│     ✓ Fonctionnalités opérationnelles                      │
│     ✓ Conformité réglementaire                             │
│     ✓ Confiance stakeholders                               │
└────────────────────────────────────────────────────────────┘
```

## Types de Tests de Restauration

### Matrice des Tests

```yaml
Test Basique (Mensuel):
  scope: "Restauration d'une collection"
  durée: "30 minutes"
  automatisé: true
  objectif: "Vérifier que les backups sont exploitables"

Test Standard (Mensuel):
  scope: "Restauration d'une base complète"
  durée: "1-2 heures"
  automatisé: true
  objectif: "Valider intégrité et procédures"

Test Complet (Trimestriel):
  scope: "Restauration cluster entier"
  durée: "4-8 heures"
  automatisé: partiel
  objectif: "Valider RTO/RPO en conditions réelles"

Test PITR (Mensuel):
  scope: "Point-in-Time Recovery"
  durée: "2-3 heures"
  automatisé: true
  objectif: "Valider capacité de récupération précise"

Disaster Recovery Drill (Annuel):
  scope: "Failover complet vers DR site"
  durée: "1-2 jours"
  automatisé: non
  objectif: "Valider plan DR end-to-end"
```

## Infrastructure de Test

### Environnement de Test Isolé

```bash
#!/bin/bash
# setup_test_environment.sh

set -euo pipefail

TEST_ENV_ROOT="/opt/mongodb-restore-test"
TEST_PORT=27099
TEST_DBPATH="${TEST_ENV_ROOT}/data"
TEST_LOGPATH="${TEST_ENV_ROOT}/logs"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "=== Setting Up Restore Test Environment ==="

# Créer les répertoires
mkdir -p "$TEST_DBPATH"
mkdir -p "$TEST_LOGPATH"
mkdir -p "${TEST_ENV_ROOT}/backups"
mkdir -p "${TEST_ENV_ROOT}/scripts"

# Configuration MongoDB de test
cat > "${TEST_ENV_ROOT}/mongod.conf" <<EOF
# MongoDB Test Instance Configuration
systemLog:
  destination: file
  path: ${TEST_LOGPATH}/mongod.log
  logAppend: true

storage:
  dbPath: ${TEST_DBPATH}
  journal:
    enabled: true
  engine: wiredTiger

net:
  port: ${TEST_PORT}
  bindIp: 127.0.0.1

processManagement:
  fork: false

security:
  authorization: disabled

setParameter:
  enableTestCommands: 1
EOF

# Script de démarrage
cat > "${TEST_ENV_ROOT}/start_test_instance.sh" <<'SCRIPT'
#!/bin/bash
mongod --config /opt/mongodb-restore-test/mongod.conf &
echo $! > /opt/mongodb-restore-test/mongod.pid
sleep 5
mongo --port 27099 --eval "db.adminCommand('ping')" && echo "✓ Test instance started"
SCRIPT

chmod +x "${TEST_ENV_ROOT}/start_test_instance.sh"

# Script d'arrêt
cat > "${TEST_ENV_ROOT}/stop_test_instance.sh" <<'SCRIPT'
#!/bin/bash
if [ -f /opt/mongodb-restore-test/mongod.pid ]; then
  kill $(cat /opt/mongodb-restore-test/mongod.pid)
  rm -f /opt/mongodb-restore-test/mongod.pid
  echo "✓ Test instance stopped"
fi
SCRIPT

chmod +x "${TEST_ENV_ROOT}/stop_test_instance.sh"

# Script de cleanup
cat > "${TEST_ENV_ROOT}/cleanup_test_instance.sh" <<'SCRIPT'
#!/bin/bash
/opt/mongodb-restore-test/stop_test_instance.sh
rm -rf /opt/mongodb-restore-test/data/*
rm -rf /opt/mongodb-restore-test/logs/*
echo "✓ Test instance cleaned"
SCRIPT

chmod +x "${TEST_ENV_ROOT}/cleanup_test_instance.sh"

log "✓ Test environment configured"
log "  Data: $TEST_DBPATH"
log "  Logs: $TEST_LOGPATH"
log "  Port: $TEST_PORT"
log ""
log "Commands:"
log "  Start: ${TEST_ENV_ROOT}/start_test_instance.sh"
log "  Stop: ${TEST_ENV_ROOT}/stop_test_instance.sh"
log "  Cleanup: ${TEST_ENV_ROOT}/cleanup_test_instance.sh"
```

## Script Complet de Test de Restauration

### Test Automatisé avec Validation

```bash
#!/bin/bash
# automated_restore_test.sh

set -euo pipefail

# ============================================================================
# CONFIGURATION
# ============================================================================

TEST_TYPE="${TEST_TYPE:-standard}"  # basic|standard|complete|pitr
BACKUP_SOURCE="${BACKUP_SOURCE:-/backup/mongodb/latest}"
TEST_ENV_ROOT="/opt/mongodb-restore-test"
TEST_PORT=27099

# Métriques
START_TIME=$(date +%s)
METRICS_FILE="/var/log/mongodb-restore-tests/test_$(date +%Y%m%d_%H%M%S).json"
TEST_ID="restore_test_$(date +%s)"

# Notifications
SLACK_WEBHOOK="${SLACK_WEBHOOK:-}"
EMAIL_RECIPIENTS="${EMAIL_RECIPIENTS:-}"

# Validation
EXPECTED_DBS=()
EXPECTED_COLLECTIONS=()
VALIDATION_QUERIES=()

# ============================================================================
# LOGGING
# ============================================================================

LOG_FILE="/var/log/mongodb-restore-tests/${TEST_ID}.log"
mkdir -p "$(dirname $LOG_FILE)"
mkdir -p "$(dirname $METRICS_FILE)"

log() {
  local level=$1
  shift
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $@" | tee -a "$LOG_FILE"
}

log_info() { log "INFO" "$@"; }
log_warn() { log "WARN" "$@"; }
log_error() { log "ERROR" "$@"; }
log_success() { log "SUCCESS" "$@"; }

# ============================================================================
# MÉTRIQUES
# ============================================================================

METRICS=()

add_metric() {
  local name=$1
  local value=$2
  METRICS+=("{\"name\":\"$name\",\"value\":$value,\"timestamp\":$(date +%s)}")
}

save_metrics() {
  cat > "$METRICS_FILE" <<EOF
{
  "test_id": "$TEST_ID",
  "test_type": "$TEST_TYPE",
  "timestamp": "$(date -Iseconds)",
  "duration_seconds": $(($(date +%s) - START_TIME)),
  "result": "$1",
  "metrics": [
    $(IFS=,; echo "${METRICS[*]}")
  ]
}
EOF
}

# ============================================================================
# PHASE 1: PRÉPARATION
# ============================================================================

prepare_test_environment() {
  log_info "=========================================="
  log_info "  MongoDB Restore Test"
  log_info "=========================================="
  log_info "Test ID: $TEST_ID"
  log_info "Test Type: $TEST_TYPE"
  log_info "Backup Source: $BACKUP_SOURCE"
  log_info ""

  # Vérifier que le backup existe
  if [ ! -d "$BACKUP_SOURCE" ] && [ ! -f "$BACKUP_SOURCE" ]; then
    log_error "Backup source not found: $BACKUP_SOURCE"
    exit 1
  fi

  log_info "✓ Backup source verified"

  # Nettoyer l'environnement de test
  log_info "Cleaning test environment..."
  "${TEST_ENV_ROOT}/cleanup_test_instance.sh"

  log_info "✓ Test environment ready"
}

# ============================================================================
# PHASE 2: DÉMARRAGE INSTANCE DE TEST
# ============================================================================

start_test_instance() {
  log_info "=== Starting Test MongoDB Instance ==="

  "${TEST_ENV_ROOT}/start_test_instance.sh"

  # Attendre que l'instance soit prête
  local max_wait=60
  local waited=0

  while [ $waited -lt $max_wait ]; do
    if mongo --port $TEST_PORT --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
      log_info "✓ Test instance is ready"
      return 0
    fi

    sleep 2
    waited=$((waited + 2))
  done

  log_error "Test instance failed to start"
  exit 1
}

# ============================================================================
# PHASE 3: RESTAURATION
# ============================================================================

perform_restore() {
  log_info "=== Performing Restoration ==="

  local restore_start=$(date +%s)

  case "$TEST_TYPE" in
    basic|standard|complete)
      restore_full
      ;;
    pitr)
      restore_point_in_time
      ;;
    *)
      log_error "Unknown test type: $TEST_TYPE"
      exit 1
      ;;
  esac

  local restore_end=$(date +%s)
  local restore_duration=$((restore_end - restore_start))

  log_info "✓ Restoration completed in ${restore_duration}s"
  add_metric "restore_duration_seconds" "$restore_duration"
}

restore_full() {
  log_info "Running full restore..."

  mongorestore \
    --port=$TEST_PORT \
    --drop \
    --oplogReplay \
    --gzip \
    --numParallelCollections=4 \
    --dir="$BACKUP_SOURCE" \
    2>&1 | tee -a "$LOG_FILE"

  if [ ${PIPESTATUS[0]} -ne 0 ]; then
    log_error "Restore failed"
    exit 1
  fi
}

restore_point_in_time() {
  log_info "Running PITR restore..."

  # Calculer un timestamp de test (2 heures avant le backup)
  local backup_time=$(jq -r '.timestamp' "${BACKUP_SOURCE}/MANIFEST.json" 2>/dev/null || echo "")

  if [ -z "$backup_time" ]; then
    log_warn "Cannot determine backup time, using full restore"
    restore_full
    return
  fi

  local target_time=$(date -d "$backup_time - 2 hours" +%s)

  log_info "Target PITR timestamp: $(date -d @$target_time -Iseconds)"

  mongorestore \
    --port=$TEST_PORT \
    --drop \
    --oplogReplay \
    --oplogLimit="${target_time}:0" \
    --gzip \
    --dir="$BACKUP_SOURCE" \
    2>&1 | tee -a "$LOG_FILE"
}

# ============================================================================
# PHASE 4: VALIDATION
# ============================================================================

validate_restored_data() {
  log_info "=== Validating Restored Data ==="

  local validation_errors=0

  # Test 1: Vérifier que MongoDB répond
  log_info "Test 1: MongoDB Connectivity"
  if mongo --port $TEST_PORT --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
    log_success "  ✓ MongoDB is responding"
  else
    log_error "  ✗ MongoDB is not responding"
    validation_errors=$((validation_errors + 1))
  fi

  # Test 2: Vérifier les bases de données
  log_info "Test 2: Database Presence"
  local dbs=$(mongo --port $TEST_PORT --quiet --eval "
    db.adminCommand({ listDatabases: 1 }).databases
      .filter(d => d.name !== 'admin' && d.name !== 'local' && d.name !== 'config')
      .map(d => d.name)
      .join(',');
  ")

  if [ -z "$dbs" ]; then
    log_error "  ✗ No application databases found"
    validation_errors=$((validation_errors + 1))
  else
    local db_count=$(echo "$dbs" | tr ',' '\n' | wc -l)
    log_success "  ✓ Found $db_count application database(s): $dbs"
    add_metric "databases_count" "$db_count"
  fi

  # Test 3: Vérifier les collections
  log_info "Test 3: Collections Check"
  mongo --port $TEST_PORT --quiet --eval "
    var totalCollections = 0;
    var totalDocuments = 0;

    db.adminCommand({ listDatabases: 1 }).databases.forEach(function(database) {
      if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
        var dbObj = db.getSiblingDB(database.name);
        var collections = dbObj.getCollectionNames();

        print('  Database: ' + database.name + ' (' + collections.length + ' collections)');

        collections.forEach(function(coll) {
          var count = dbObj[coll].countDocuments();
          totalDocuments += count;
          totalCollections += 1;
          print('    - ' + coll + ': ' + count + ' documents');
        });
      }
    });

    print('  Total: ' + totalCollections + ' collections, ' + totalDocuments + ' documents');
  " | tee -a "$LOG_FILE"

  # Test 4: Validation de l'intégrité
  log_info "Test 4: Data Integrity"

  local invalid_collections=$(mongo --port $TEST_PORT --quiet --eval "
    var invalid = [];

    db.adminCommand({ listDatabases: 1 }).databases.forEach(function(database) {
      if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
        var dbObj = db.getSiblingDB(database.name);

        dbObj.getCollectionNames().forEach(function(coll) {
          var validation = dbObj[coll].validate({ full: false });
          if (!validation.valid) {
            invalid.push(database.name + '.' + coll);
          }
        });
      }
    });

    print(invalid.join(','));
  ")

  if [ -z "$invalid_collections" ]; then
    log_success "  ✓ All collections are valid"
  else
    log_error "  ✗ Invalid collections: $invalid_collections"
    validation_errors=$((validation_errors + 1))
  fi

  # Test 5: Vérifier les index
  log_info "Test 5: Index Verification"

  local missing_indexes=$(mongo --port $TEST_PORT --quiet --eval "
    var missingIndexes = 0;

    db.adminCommand({ listDatabases: 1 }).databases.forEach(function(database) {
      if (database.name !== 'admin' && database.name !== 'local' && database.name !== 'config') {
        var dbObj = db.getSiblingDB(database.name);

        dbObj.getCollectionNames().forEach(function(coll) {
          var indexes = dbObj[coll].getIndexes();

          // Vérifier au moins l'index _id
          if (indexes.length === 0) {
            missingIndexes++;
          }
        });
      }
    });

    print(missingIndexes);
  ")

  if [ "$missing_indexes" = "0" ]; then
    log_success "  ✓ All collections have indexes"
  else
    log_warn "  ⚠ $missing_indexes collection(s) missing indexes"
  fi

  # Test 6: Requêtes fonctionnelles
  log_info "Test 6: Functional Queries"

  mongo --port $TEST_PORT --quiet --eval "
    // Exemple: tester quelques requêtes simples
    use mydb

    // Test de lecture
    try {
      var count = db.users.countDocuments();
      print('  ✓ Read query successful: ' + count + ' users');
    } catch(e) {
      print('  ✗ Read query failed: ' + e);
    }

    // Test d'agrégation
    try {
      var result = db.orders.aggregate([
        { \$group: { _id: '\$status', count: { \$sum: 1 } } },
        { \$limit: 5 }
      ]).toArray();
      print('  ✓ Aggregation query successful: ' + result.length + ' groups');
    } catch(e) {
      print('  ✗ Aggregation query failed: ' + e);
    }
  " | tee -a "$LOG_FILE"

  # Test 7: Vérification temporelle (PITR)
  if [ "$TEST_TYPE" = "pitr" ]; then
    log_info "Test 7: PITR Temporal Verification"

    # Vérifier qu'aucune donnée n'existe après le target timestamp
    # (nécessite des champs timestamp dans les documents)
    mongo --port $TEST_PORT --quiet --eval "
      use mydb

      // Exemple avec collection orders ayant un champ created_at
      if (db.orders.findOne({}) && db.orders.findOne({}).created_at) {
        var targetTime = new Date('$(date -d @$target_time -Iseconds)');
        var recentCount = db.orders.countDocuments({
          created_at: { \$gt: targetTime }
        });

        if (recentCount > 0) {
          print('  ✗ Found ' + recentCount + ' documents after target time');
        } else {
          print('  ✓ No documents found after target timestamp');
        }
      } else {
        print('  ⚠ Cannot verify temporal consistency (no timestamp field)');
      }
    " | tee -a "$LOG_FILE"
  fi

  # Résumé de la validation
  log_info ""
  log_info "Validation Summary:"
  log_info "  Errors: $validation_errors"

  add_metric "validation_errors" "$validation_errors"

  if [ $validation_errors -gt 0 ]; then
    log_error "✗ Validation FAILED with $validation_errors error(s)"
    return 1
  else
    log_success "✓ Validation PASSED"
    return 0
  fi
}

# ============================================================================
# PHASE 5: PERFORMANCE BENCHMARK
# ============================================================================

benchmark_performance() {
  log_info "=== Performance Benchmark ==="

  mongo --port $TEST_PORT --quiet --eval "
    use mydb

    // Test de lecture
    var startRead = new Date();
    var readCount = db.users.find().limit(1000).toArray().length;
    var readTime = new Date() - startRead;
    print('Read Performance: ' + readCount + ' docs in ' + readTime + 'ms');

    // Test d'écriture (sur instance de test, pas de souci)
    var startWrite = new Date();
    for (var i = 0; i < 100; i++) {
      db.test_collection.insertOne({ test: true, index: i });
    }
    var writeTime = new Date() - startWrite;
    print('Write Performance: 100 docs in ' + writeTime + 'ms');

    // Test d'agrégation
    var startAgg = new Date();
    var aggResult = db.orders.aggregate([
      { \$group: { _id: '\$status', count: { \$sum: 1 }, total: { \$sum: '\$amount' } } },
      { \$sort: { count: -1 } }
    ]).toArray();
    var aggTime = new Date() - startAgg;
    print('Aggregation Performance: ' + aggResult.length + ' groups in ' + aggTime + 'ms');
  " | tee -a "$LOG_FILE"
}

# ============================================================================
# PHASE 6: CLEANUP
# ============================================================================

cleanup_test_instance() {
  log_info "=== Cleaning Up ==="

  "${TEST_ENV_ROOT}/stop_test_instance.sh"

  # Optionnel: garder les données pour analyse
  if [ "${KEEP_TEST_DATA:-false}" = "false" ]; then
    "${TEST_ENV_ROOT}/cleanup_test_instance.sh"
  else
    log_info "Test data preserved for analysis"
  fi
}

# ============================================================================
# NOTIFICATIONS
# ============================================================================

send_notification() {
  local result=$1
  local duration=$2

  log_info "Sending notifications..."

  local color="good"
  local emoji="✅"

  if [ "$result" = "FAILED" ]; then
    color="danger"
    emoji="🚨"
  fi

  # Slack
  if [ -n "$SLACK_WEBHOOK" ]; then
    curl -X POST "$SLACK_WEBHOOK" \
      -H 'Content-Type: application/json' \
      -d "{
        \"text\": \"$emoji MongoDB Restore Test $result\",
        \"attachments\": [{
          \"color\": \"$color\",
          \"fields\": [
            {\"title\": \"Test ID\", \"value\": \"$TEST_ID\", \"short\": true},
            {\"title\": \"Test Type\", \"value\": \"$TEST_TYPE\", \"short\": true},
            {\"title\": \"Duration\", \"value\": \"${duration}s\", \"short\": true},
            {\"title\": \"Result\", \"value\": \"$result\", \"short\": true},
            {\"title\": \"Log\", \"value\": \"$LOG_FILE\", \"short\": false}
          ]
        }]
      }" >/dev/null 2>&1
  fi

  # Email
  if [ -n "$EMAIL_RECIPIENTS" ]; then
    (
      echo "MongoDB Restore Test $result"
      echo ""
      echo "Test ID: $TEST_ID"
      echo "Test Type: $TEST_TYPE"
      echo "Duration: ${duration}s"
      echo ""
      echo "Log file: $LOG_FILE"
      echo "Metrics file: $METRICS_FILE"
    ) | mail -s "$emoji MongoDB Restore Test $result - $TEST_ID" "$EMAIL_RECIPIENTS"
  fi
}

# ============================================================================
# MAIN
# ============================================================================

main() {
  local exit_code=0
  local result="SUCCESS"

  # Phase 1: Préparation
  prepare_test_environment

  # Phase 2: Démarrage
  start_test_instance

  # Phase 3: Restauration
  if ! perform_restore; then
    result="FAILED"
    exit_code=1
  fi

  # Phase 4: Validation (même si restauration échouée, pour diagnostics)
  if ! validate_restored_data; then
    result="FAILED"
    exit_code=1
  fi

  # Phase 5: Benchmark (seulement si succès)
  if [ $exit_code -eq 0 ]; then
    benchmark_performance
  fi

  # Phase 6: Cleanup
  cleanup_test_instance

  # Calcul final
  local end_time=$(date +%s)
  local total_duration=$((end_time - START_TIME))

  # Métriques
  add_metric "test_success" $([ $exit_code -eq 0 ] && echo "1" || echo "0")
  add_metric "test_duration_seconds" "$total_duration"
  save_metrics "$result"

  # Notifications
  send_notification "$result" "$total_duration"

  # Résumé
  log_info ""
  log_info "=========================================="
  log_info "  Test Completed: $result"
  log_info "=========================================="
  log_info "Test ID: $TEST_ID"
  log_info "Duration: ${total_duration}s"
  log_info "Log: $LOG_FILE"
  log_info "Metrics: $METRICS_FILE"
  log_info "=========================================="

  exit $exit_code
}

# Gestion des erreurs
trap 'log_error "Script failed at line $LINENO"; cleanup_test_instance; exit 1' ERR

# Exécuter
main "$@"
```

## Tests Spécifiques

### Test de Disaster Recovery (DR Drill)

```bash
#!/bin/bash
# dr_drill_test.sh

set -euo pipefail

DR_SITE_REGION="us-west-2"
PRIMARY_SITE_REGION="us-east-1"
FAILOVER_TIME_LIMIT=3600  # 1 heure max

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

log "========================================================"
log "  Disaster Recovery Drill"
log "========================================================"
log "Primary Site: $PRIMARY_SITE_REGION"
log "DR Site: $DR_SITE_REGION"
log "Time Limit: ${FAILOVER_TIME_LIMIT}s"
log ""

START_TIME=$(date +%s)

# ============================================================================
# PHASE 1: SIMULATION DE DISASTER
# ============================================================================

log "=== Phase 1: Disaster Simulation ==="
log "⚠️  This is a DRILL - primary site will NOT be affected"

read -p "Proceed with DR drill? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
  log "DR drill cancelled"
  exit 0
fi

# ============================================================================
# PHASE 2: VÉRIFICATION BACKUPS DR SITE
# ============================================================================

log "=== Phase 2: Verifying DR Site Backups ==="

# Vérifier les backups sur le DR site
DR_BACKUP_COUNT=$(aws s3 ls s3://company-backups-dr/$DR_SITE_REGION/ | wc -l)

if [ "$DR_BACKUP_COUNT" -eq 0 ]; then
  log "✗ No backups found on DR site"
  exit 1
fi

log "✓ Found $DR_BACKUP_COUNT backup(s) on DR site"

# Identifier le backup le plus récent
LATEST_BACKUP=$(aws s3 ls s3://company-backups-dr/$DR_SITE_REGION/ --recursive | sort | tail -1 | awk '{print $4}')

log "Latest backup: $LATEST_BACKUP"

# ============================================================================
# PHASE 3: PROVISIONING INFRASTRUCTURE DR
# ============================================================================

log "=== Phase 3: Provisioning DR Infrastructure ==="

# Exemple avec Terraform
cd /infrastructure/mongodb-dr

terraform init
terraform plan -var="region=$DR_SITE_REGION"

read -p "Provision DR infrastructure? (yes/no): " confirm
if [ "$confirm" != "yes" ]; then
  log "DR provision cancelled"
  exit 0
fi

terraform apply -var="region=$DR_SITE_REGION" -auto-approve

DR_ENDPOINT=$(terraform output -raw mongodb_endpoint)

log "✓ DR infrastructure provisioned"
log "  Endpoint: $DR_ENDPOINT"

# ============================================================================
# PHASE 4: RESTAURATION SUR DR SITE
# ============================================================================

log "=== Phase 4: Restoring on DR Site ==="

# Télécharger le backup
aws s3 cp "s3://company-backups-dr/$LATEST_BACKUP" /tmp/dr_backup.tar.gz

# Restaurer
mongorestore \
  --uri="mongodb://$DR_ENDPOINT:27017" \
  --drop \
  --oplogReplay \
  --archive=/tmp/dr_backup.tar.gz \
  --gzip

log "✓ Data restored on DR site"

# ============================================================================
# PHASE 5: VALIDATION DR SITE
# ============================================================================

log "=== Phase 5: Validating DR Site ==="

# Vérifier connectivité
if mongo "mongodb://$DR_ENDPOINT:27017" --eval "db.adminCommand('ping')" >/dev/null 2>&1; then
  log "✓ DR site is accessible"
else
  log "✗ Cannot connect to DR site"
  exit 1
fi

# Vérifier les données
mongo "mongodb://$DR_ENDPOINT:27017" --quiet --eval "
  print('=== DR Site Data Verification ===');

  dbs = db.adminCommand({ listDatabases: 1 }).databases;
  print('Databases: ' + dbs.length);

  var totalDocs = 0;
  dbs.forEach(function(db) {
    if (db.name !== 'admin' && db.name !== 'local' && db.name !== 'config') {
      var dbObj = db.getSiblingDB(db.name);
      dbObj.getCollectionNames().forEach(function(coll) {
        totalDocs += dbObj[coll].countDocuments();
      });
    }
  });

  print('Total documents: ' + totalDocs);
"

# ============================================================================
# PHASE 6: FAILOVER TEST APPLICATIF
# ============================================================================

log "=== Phase 6: Application Failover Test ==="

# Mettre à jour la configuration applicative (simulé)
log "Updating application configuration to point to DR site..."
# kubectl set env deployment/myapp MONGO_URI="mongodb://$DR_ENDPOINT:27017"

log "⚠️  In real DR scenario:"
log "  1. Update DNS/load balancer to DR site"
log "  2. Update application configs"
log "  3. Validate application functionality"
log "  4. Communicate to stakeholders"

# ============================================================================
# PHASE 7: VALIDATION FONCTIONNELLE
# ============================================================================

log "=== Phase 7: Functional Validation ==="

# Simuler des opérations applicatives
mongo "mongodb://$DR_ENDPOINT:27017" --quiet --eval "
  use mydb

  // Test lecture
  var users = db.users.find().limit(10).toArray();
  print('✓ Read test: ' + users.length + ' users retrieved');

  // Test écriture
  var writeResult = db.test_failover.insertOne({
    test: 'dr_drill',
    timestamp: new Date()
  });
  print('✓ Write test: document inserted');

  // Test transaction
  var session = db.getMongo().startSession();
  session.startTransaction();
  try {
    db.orders.insertOne({ dr_test: true }, { session: session });
    session.commitTransaction();
    print('✓ Transaction test: successful');
  } catch(e) {
    session.abortTransaction();
    print('✗ Transaction test: failed');
  } finally {
    session.endSession();
  }
"

# ============================================================================
# PHASE 8: MÉTRIQUES ET RAPPORT
# ============================================================================

END_TIME=$(date +%s)
DRILL_DURATION=$((END_TIME - START_TIME))

log ""
log "========================================================"
log "  DR DRILL COMPLETED"
log "========================================================"
log "Duration: ${DRILL_DURATION}s ($(date -u -d @${DRILL_DURATION} +%H:%M:%S))"

if [ $DRILL_DURATION -lt $FAILOVER_TIME_LIMIT ]; then
  log "✓ RTO Target MET (< ${FAILOVER_TIME_LIMIT}s)"
else
  log "✗ RTO Target MISSED (> ${FAILOVER_TIME_LIMIT}s)"
fi

log ""
log "Lessons Learned:"
log "  - Document any issues encountered"
log "  - Update DR procedures if needed"
log "  - Review team coordination"
log "  - Validate communication plan"
log ""
log "⚠️  Remember to tear down DR infrastructure after drill"
log "========================================================"

# Générer rapport
cat > "/tmp/dr_drill_report_$(date +%Y%m%d).md" <<EOF
# Disaster Recovery Drill Report

**Date:** $(date -Iseconds)
**Duration:** ${DRILL_DURATION}s
**RTO Target:** ${FAILOVER_TIME_LIMIT}s
**Result:** $([ $DRILL_DURATION -lt $FAILOVER_TIME_LIMIT ] && echo "PASS" || echo "FAIL")

## Timeline

1. **Disaster Simulation:** Started
2. **Backup Verification:** $DR_BACKUP_COUNT backups found
3. **Infrastructure Provisioning:** Completed
4. **Data Restoration:** Completed
5. **Validation:** Passed
6. **Functional Testing:** Passed

## Metrics

- Total Duration: ${DRILL_DURATION}s
- Restoration Time: [to be filled]
- Validation Time: [to be filled]

## Issues Encountered

[Document any issues]

## Action Items

- [ ] Update DR documentation
- [ ] Address any issues found
- [ ] Schedule next drill
- [ ] Share results with team

## Participants

[List participants]

EOF

log "Report generated: /tmp/dr_drill_report_$(date +%Y%m%d).md"
```

## Automatisation des Tests

### Cron Job de Tests Réguliers

```bash
# /etc/cron.d/mongodb-restore-tests

# Test basique hebdomadaire (dimanche 3h)
0 3 * * 0 mongodb-backup /usr/local/bin/automated_restore_test.sh basic >> /var/log/mongodb-restore-tests/cron.log 2>&1

# Test standard mensuel (premier du mois, 4h)
0 4 1 * * mongodb-backup /usr/local/bin/automated_restore_test.sh standard >> /var/log/mongodb-restore-tests/cron.log 2>&1

# Test PITR mensuel (15 du mois, 5h)
0 5 15 * * mongodb-backup /usr/local/bin/automated_restore_test.sh pitr >> /var/log/mongodb-restore-tests/cron.log 2>&1
```

### Kubernetes CronJob

```yaml
# restore-test-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-restore-test
  namespace: database
spec:
  # Hebdomadaire le dimanche à 3h
  schedule: "0 3 * * 0"

  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 5

  jobTemplate:
    spec:
      activeDeadlineSeconds: 7200  # 2 heures max
      backoffLimit: 1

      template:
        metadata:
          labels:
            app: mongodb-restore-test
        spec:
          restartPolicy: Never

          serviceAccountName: mongodb-backup

          containers:
          - name: restore-test
            image: mongodb-test-tools:latest

            env:
            - name: TEST_TYPE
              value: "standard"
            - name: BACKUP_SOURCE
              value: "/backups/latest"
            - name: SLACK_WEBHOOK
              valueFrom:
                secretKeyRef:
                  name: mongodb-backup-credentials
                  key: slack_webhook

            volumeMounts:
            - name: backups
              mountPath: /backups
              readOnly: true
            - name: test-env
              mountPath: /opt/mongodb-restore-test
            - name: scripts
              mountPath: /scripts

            command:
            - /scripts/automated_restore_test.sh

            resources:
              requests:
                cpu: 2
                memory: 4Gi
              limits:
                cpu: 4
                memory: 8Gi

          volumes:
          - name: backups
            persistentVolumeClaim:
              claimName: mongodb-backup-pvc
          - name: test-env
            emptyDir:
              sizeLimit: 100Gi
          - name: scripts
            configMap:
              name: mongodb-backup-scripts
              defaultMode: 0755
```

## Reporting et Documentation

### Template de Rapport de Test

```markdown
# MongoDB Restore Test Report

**Test ID:** {TEST_ID}
**Date:** {DATE}
**Test Type:** {TEST_TYPE}
**Duration:** {DURATION}
**Result:** {RESULT}

## Executive Summary

[Brief overview of test results]

## Test Configuration

- **Backup Source:** {BACKUP_SOURCE}
- **Backup Date:** {BACKUP_DATE}
- **Backup Size:** {BACKUP_SIZE}
- **Test Environment:** {TEST_ENV}
- **MongoDB Version:** {MONGO_VERSION}

## Test Phases

### 1. Preparation
- [x] Environment cleaned
- [x] Backup verified
- [x] Test instance started

### 2. Restoration
- **Duration:** {RESTORE_DURATION}s
- **Method:** {RESTORE_METHOD}
- **Status:** {RESTORE_STATUS}

### 3. Validation
- [x] Connectivity verified
- [x] Databases present: {DB_COUNT}
- [x] Collections verified: {COLL_COUNT}
- [x] Data integrity checked
- [x] Indexes verified
- [x] Functional queries tested

### 4. Performance
- **Read Performance:** {READ_PERF}
- **Write Performance:** {WRITE_PERF}
- **Aggregation Performance:** {AGG_PERF}

## Validation Results

| Check | Status | Details |
|-------|--------|---------|
| Connectivity | ✓ | MongoDB responding |
| Databases | ✓ | {DB_COUNT} databases restored |
| Collections | ✓ | {COLL_COUNT} collections |
| Data Integrity | ✓ | All collections valid |
| Indexes | ✓ | All indexes present |
| Functional Queries | ✓ | All queries successful |

## Metrics

```json
{METRICS_JSON}
```

## Issues Encountered

{LIST_ISSUES}

## Recommendations

{RECOMMENDATIONS}

## Next Steps

- [ ] Address any issues found
- [ ] Update documentation
- [ ] Schedule next test
- [ ] Share results with team

---

**Prepared by:** {PREPARER}
**Reviewed by:** {REVIEWER}
**Approved by:** {APPROVER}
```

## Bonnes Pratiques

### Checklist de Tests de Restauration

```markdown
### Fréquence Recommandée

Development:
  - [ ] Tests basiques: Mensuel
  - [ ] Tests complets: Trimestriel
  - [ ] DR drills: Annuel

Production Standard:
  - [ ] Tests basiques: Hebdomadaire
  - [ ] Tests standard: Mensuel
  - [ ] Tests complets: Trimestriel
  - [ ] Tests PITR: Mensuel
  - [ ] DR drills: Semestriel

Production Critique:
  - [ ] Tests basiques: Quotidien (automatisé)
  - [ ] Tests standard: Hebdomadaire
  - [ ] Tests complets: Mensuel
  - [ ] Tests PITR: Hebdomadaire
  - [ ] DR drills: Trimestriel

### Configuration des Tests

- [ ] Environnement de test isolé configuré
- [ ] Scripts de test automatisés
- [ ] Validation complète implémentée
- [ ] Métriques collectées
- [ ] Notifications configurées
- [ ] Documentation maintenue
- [ ] Rotation des tests programmée

### Après Chaque Test

- [ ] Résultats documentés
- [ ] Métriques analysées
- [ ] Issues identifiées et trackées
- [ ] RTO/RPO validés ou révisés
- [ ] Procédures mises à jour si nécessaire
- [ ] Équipe informée des résultats
- [ ] Actions correctives planifiées

### Amélioration Continue

- [ ] Analyser les tendances (durée, succès)
- [ ] Identifier les points de friction
- [ ] Automatiser davantage
- [ ] Former l'équipe régulièrement
- [ ] Réviser les procédures DR
- [ ] Optimiser les performances de restauration
```

## Métriques Clés

### KPIs à Suivre

```yaml
Métriques Techniques:
  restore_success_rate:
    description: "Taux de succès des tests de restauration"
    target: "> 95%"
    measurement: "Tests réussis / Tests total"

  restore_duration:
    description: "Durée moyenne de restauration"
    target: "< RTO défini"
    measurement: "Secondes"

  validation_errors:
    description: "Erreurs de validation"
    target: "= 0"
    measurement: "Nombre"

Métriques Opérationnelles:
  test_frequency:
    description: "Fréquence des tests"
    target: "Selon policy définie"
    measurement: "Tests / mois"

  documentation_updates:
    description: "Mises à jour documentation"
    target: "Après chaque test avec issues"
    measurement: "Updates / trimestre"

  team_readiness:
    description: "Préparation de l'équipe"
    target: "100%"
    measurement: "% équipe formée"

Métriques Business:
  rto_compliance:
    description: "Respect du RTO"
    target: "100%"
    measurement: "Tests respectant RTO / Tests total"

  rpo_validation:
    description: "Validation du RPO"
    target: "100%"
    measurement: "Tests validant RPO / Tests total"

  stakeholder_confidence:
    description: "Confiance des stakeholders"
    target: "Élevée"
    measurement: "Survey qualitative"
```

## Conclusion

Les tests de restauration ne sont pas optionnels - ils sont **essentiels** à toute stratégie de continuité d'activité crédible. Sans tests réguliers, vos backups ne sont qu'une illusion de sécurité.

**Principes clés** :

1. **Tester régulièrement** - Selon la criticité (quotidien à annuel)
2. **Automatiser** - Tests automatisés = consistance et couverture
3. **Valider complètement** - Ne pas se contenter de "ça démarre"
4. **Documenter** - Chaque test alimente l'amélioration continue
5. **Former l'équipe** - Les tests sont aussi des exercices de formation

**Minimum viable** :
- Tests mensuels automatisés
- Validation de l'intégrité des données
- Mesure du RTO réel
- DR drill annuel

**Excellence** :
- Tests quotidiens automatisés pour production critique
- Validation exhaustive (intégrité, performance, fonctionnelle)
- DR drills trimestriels
- Amélioration continue basée sur les métriques

Les organisations qui testent régulièrement leurs restaurations ne découvrent pas de mauvaises surprises lors d'un incident réel - elles ont confiance dans leur capacité à récupérer.

---


⏭️ [Bonnes pratiques](/12-sauvegarde-restauration/12-bonnes-pratiques.md)
