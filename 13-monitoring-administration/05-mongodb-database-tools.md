🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.5 MongoDB Database Tools

## Introduction

Les **MongoDB Database Tools** constituent une suite d'utilitaires en ligne de commande essentiels pour l'administration, la maintenance et les opérations de production des bases de données MongoDB. Ces outils, découplés du serveur MongoDB depuis la version 4.4, offrent aux SRE et administrateurs système un ensemble complet de fonctionnalités pour :

- **Sauvegarde et restauration** de données avec précision
- **Migration** de données entre environnements
- **Import/Export** dans divers formats
- **Monitoring** en temps réel des performances
- **Diagnostic** et analyse des problèmes
- **Automatisation** des tâches d'administration

Ces outils sont cruciaux dans les workflows DevOps modernes et constituent souvent le premier niveau d'intervention pour les opérations critiques en production.

---

## Architecture et Installation

### Vue d'Ensemble des Outils

```
┌─────────────────────────────────────────────────────────────┐
│           MongoDB Database Tools Suite                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Data Migration          Monitoring            Utilities    │
│  ┌──────────────┐       ┌──────────────┐      ┌─────────┐   │
│  │ mongodump    │       │ mongostat    │      │bsondump │   │
│  │ mongorestore │       │ mongotop     │      │mongofiles   │
│  │ mongoexport  │       └──────────────┘      └─────────┘   │
│  │ mongoimport  │                                           │
│  └──────────────┘                                           │
│                                                             │
│  Connection via:                                            │
│  • mongodb:// URI                                           │
│  • Authentication (SCRAM, x.509, LDAP, Kerberos)            │
│  • TLS/SSL encryption                                       │
│  • Direct connection or Replica Set                         │
└─────────────────────────────────────────────────────────────┘
```

### Installation

#### Via Gestionnaire de Paquets

**Ubuntu/Debian** :
```bash
# Import de la clé GPG
wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | sudo apt-key add -

# Ajout du repository
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu focal/mongodb-org/7.0 multiverse" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list

# Installation
sudo apt-get update
sudo apt-get install -y mongodb-database-tools

# Vérification
mongodump --version
```

**RHEL/CentOS** :
```bash
# Configuration du repository
cat <<EOF | sudo tee /etc/yum.repos.d/mongodb-org-7.0.repo
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/\$releasever/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-7.0.asc
EOF

# Installation
sudo yum install -y mongodb-database-tools

# Vérification
mongodump --version
```

**macOS** :
```bash
# Via Homebrew
brew tap mongodb/brew
brew install mongodb-database-tools

# Vérification
mongodump --version
```

#### Installation Manuelle (Binaires)

```bash
# Téléchargement
wget https://fastdl.mongodb.org/tools/db/mongodb-database-tools-ubuntu2004-x86_64-100.9.4.tgz

# Extraction
tar -zxvf mongodb-database-tools-ubuntu2004-x86_64-100.9.4.tgz

# Installation
sudo cp mongodb-database-tools-ubuntu2004-x86_64-100.9.4/bin/* /usr/local/bin/

# Vérification
mongodump --version
```

#### Conteneur Docker

```dockerfile
# Dockerfile pour outils MongoDB
FROM ubuntu:22.04

RUN apt-get update && \
    apt-get install -y wget gnupg && \
    wget -qO - https://www.mongodb.org/static/pgp/server-7.0.asc | apt-key add - && \
    echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | \
    tee /etc/apt/sources.list.d/mongodb-org-7.0.list && \
    apt-get update && \
    apt-get install -y mongodb-database-tools && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /backup
ENTRYPOINT ["/bin/bash"]
```

---

## mongodump - Sauvegarde de Données

### Principe de Fonctionnement

`mongodump` crée une sauvegarde logique en exportant les données au format BSON, préservant tous les types de données natifs MongoDB. Contrairement aux snapshots physiques, mongodump :

- **Lit les données via les drivers MongoDB** (non invasif)
- **Peut filtrer** les collections et documents
- **N'impacte pas** le service (lecture à faible priorité)
- **Capture un état cohérent** à un point dans le temps (avec --oplog)

### Syntaxe de Base

```bash
mongodump [options] --uri="mongodb://connection-string"
```

### Options Essentielles

#### Connexion et Authentification

```bash
# Connexion URI (recommandé)
mongodump --uri="mongodb://user:password@localhost:27017/mydb?authSource=admin"

# Connexion paramétrisée
mongodump \
  --host="localhost:27017" \
  --username="backupUser" \
  --password="securePassword" \
  --authenticationDatabase="admin" \
  --db="mydb"

# Connexion SSL/TLS
mongodump \
  --uri="mongodb://localhost:27017/mydb" \
  --ssl \
  --sslCAFile=/path/to/ca.pem \
  --sslPEMKeyFile=/path/to/client.pem

# Connexion à un Replica Set
mongodump \
  --uri="mongodb://node1:27017,node2:27017,node3:27017/mydb?replicaSet=rs0" \
  --readPreference=secondary
```

#### Sélection des Données

```bash
# Backup de toute l'instance (toutes les bases)
mongodump --uri="mongodb://localhost:27017" --out=/backup/full

# Backup d'une base spécifique
mongodump --uri="mongodb://localhost:27017" --db=mydb --out=/backup/mydb

# Backup d'une collection spécifique
mongodump --uri="mongodb://localhost:27017" --db=mydb --collection=users --out=/backup/users

# Backup avec filtre (query)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=orders \
  --query='{"status": "completed", "date": {"$gte": {"$date": "2024-01-01T00:00:00Z"}}}' \
  --out=/backup/orders_completed

# Backup de collections multiples (exclude)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --excludeCollection=temp_data \
  --excludeCollection=logs \
  --out=/backup/mydb_filtered
```

#### Options de Performance

```bash
# Parallélisation (4 collections simultanées)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --numParallelCollections=4 \
  --out=/backup/mydb

# Compression (gzip, très recommandé en production)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --gzip \
  --out=/backup/mydb_compressed

# Archive (single file pour transfert)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --archive=/backup/mydb.archive \
  --gzip

# Streaming vers stdout (pour pipe)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --archive \
  --gzip | \
  aws s3 cp - s3://my-bucket/backups/mydb-$(date +%Y%m%d).archive.gz
```

### Backup Point-in-Time (Replica Set)

Pour capturer un état cohérent avec possibilité de restauration à un point précis :

```bash
# Backup avec oplog (capture les opérations pendant le dump)
mongodump \
  --uri="mongodb://localhost:27017" \
  --oplog \
  --gzip \
  --out=/backup/mydb_pit

# Structure créée :
# /backup/mydb_pit/
#   mydb/
#     collection1.bson.gz
#     collection1.metadata.json.gz
#   oplog.bson.gz  # <- Opérations capturées
```

**Important** : --oplog nécessite :
- Connexion à un replica set (ou standalone avec oplog activé)
- Permissions sur `local.oplog.rs`
- Ne fonctionne pas avec --db (backup full uniquement)

### Cas d'Usage Avancés

#### 1. Backup Incrémental (Simulation)

MongoDB ne supporte pas nativement le backup incrémental, mais on peut le simuler :

```bash
#!/bin/bash
# incremental_backup.sh

LAST_BACKUP_DATE=$(cat /backup/.last_backup_date 2>/dev/null || echo "1970-01-01T00:00:00Z")
CURRENT_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)

mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=audit_logs \
  --query="{\"timestamp\": {\"\$gte\": {\"\$date\": \"$LAST_BACKUP_DATE\"}}}" \
  --out=/backup/incremental/$(date +%Y%m%d_%H%M%S) \
  --gzip

echo "$CURRENT_DATE" > /backup/.last_backup_date
```

#### 2. Backup par Shards (Cluster Sharded)

```bash
#!/bin/bash
# backup_sharded_cluster.sh

# 1. Arrêter le balancer
mongo --eval 'sh.stopBalancer()'

# 2. Backup des config servers
mongodump \
  --uri="mongodb://config1:27019,config2:27019,config3:27019/?replicaSet=configRS" \
  --oplog \
  --out=/backup/config_servers \
  --gzip

# 3. Backup de chaque shard
for shard in shard01 shard02 shard03; do
  mongodump \
    --uri="mongodb://${shard}:27017/?replicaSet=${shard}RS" \
    --oplog \
    --out=/backup/${shard} \
    --gzip &
done

wait

# 4. Redémarrer le balancer
mongo --eval 'sh.startBalancer()'
```

#### 3. Backup avec Exclusion de Données Sensibles

```bash
# Backup sans champs sensibles (nécessite MongoDB 4.2+)
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --query='{}' \
  --excludeFields="password,ssn,creditCard" \
  --out=/backup/users_anonymized \
  --gzip
```

#### 4. Backup avec Vérification d'Intégrité

```bash
#!/bin/bash
# backup_with_verification.sh

BACKUP_DIR="/backup/$(date +%Y%m%d_%H%M%S)"
LOG_FILE="/var/log/mongodb_backup.log"

echo "[$(date)] Starting backup..." | tee -a $LOG_FILE

# Backup
mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --oplog \
  --gzip \
  --out=$BACKUP_DIR \
  2>&1 | tee -a $LOG_FILE

# Vérification
if [ $? -eq 0 ]; then
  # Calculer checksum
  find $BACKUP_DIR -type f -name "*.bson.gz" -exec sha256sum {} \; > $BACKUP_DIR/checksums.txt

  # Vérifier la taille
  BACKUP_SIZE=$(du -sh $BACKUP_DIR | cut -f1)
  echo "[$(date)] Backup completed successfully: $BACKUP_SIZE" | tee -a $LOG_FILE

  # Notifier succès
  curl -X POST "https://monitoring.example.com/webhooks/backup-success" \
    -d "backup_dir=$BACKUP_DIR&size=$BACKUP_SIZE"
else
  echo "[$(date)] Backup FAILED!" | tee -a $LOG_FILE

  # Notifier échec
  curl -X POST "https://monitoring.example.com/webhooks/backup-failure" \
    -d "backup_dir=$BACKUP_DIR"

  exit 1
fi
```

### Métriques et Monitoring

**Métriques clés à surveiller** :

```bash
# Script de monitoring du backup
#!/bin/bash

BACKUP_START=$(date +%s)

mongodump \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --out=/backup/mydb \
  --verbose 2>&1 | \
  tee /tmp/mongodump_output.log

BACKUP_END=$(date +%s)
BACKUP_DURATION=$((BACKUP_END - BACKUP_START))

# Extraire les métriques
COLLECTIONS_DUMPED=$(grep -c "writing" /tmp/mongodump_output.log)
DOCS_DUMPED=$(grep "documents" /tmp/mongodump_output.log | awk '{sum+=$1} END {print sum}')
BACKUP_SIZE=$(du -sh /backup/mydb | cut -f1)

# Envoyer à Prometheus/Grafana
cat <<EOF | curl -X POST --data-binary @- http://pushgateway:9091/metrics/job/mongodb_backup
# HELP mongodb_backup_duration_seconds Backup duration in seconds
# TYPE mongodb_backup_duration_seconds gauge
mongodb_backup_duration_seconds $BACKUP_DURATION

# HELP mongodb_backup_collections_total Number of collections backed up
# TYPE mongodb_backup_collections_total gauge
mongodb_backup_collections_total $COLLECTIONS_DUMPED

# HELP mongodb_backup_documents_total Number of documents backed up
# TYPE mongodb_backup_documents_total gauge
mongodb_backup_documents_total $DOCS_DUMPED
EOF
```

---

## mongorestore - Restauration de Données

### Principe de Fonctionnement

`mongorestore` restaure les données depuis un backup créé par mongodump. Il :

- **Insère les documents** via les drivers MongoDB
- **Reconstruit les index** automatiquement
- **Gère les conflits** via différentes stratégies
- **Supporte la restauration sélective** par base/collection

### Syntaxe de Base

```bash
mongorestore [options] <backup-directory>
```

### Stratégies de Restauration

#### 1. Restauration Complète (Drop and Restore)

```bash
# Restaure en écrasant les données existantes
mongorestore \
  --uri="mongodb://localhost:27017" \
  --drop \
  --gzip \
  /backup/mydb
```

**--drop** : Supprime les collections existantes avant restauration (recommandé pour restauration propre)

#### 2. Restauration Additive

```bash
# Ajoute les documents sans supprimer l'existant
mongorestore \
  --uri="mongodb://localhost:27017" \
  --gzip \
  /backup/mydb

# Gestion des duplicates : ignore les _id existants
mongorestore \
  --uri="mongodb://localhost:27017" \
  --gzip \
  --stopOnError=false \
  /backup/mydb
```

#### 3. Restauration Sélective

```bash
# Restaurer une seule base
mongorestore \
  --uri="mongodb://localhost:27017" \
  --nsInclude="mydb.*" \
  --drop \
  --gzip \
  /backup/full

# Restaurer une seule collection
mongorestore \
  --uri="mongodb://localhost:27017" \
  --nsInclude="mydb.users" \
  --drop \
  --gzip \
  /backup/mydb

# Restaurer avec renommage
mongorestore \
  --uri="mongodb://localhost:27017" \
  --nsFrom="mydb.users" \
  --nsTo="mydb_test.users" \
  --gzip \
  /backup/mydb
```

#### 4. Restauration Point-in-Time

```bash
# Restaurer avec replay de l'oplog jusqu'à un timestamp spécifique
mongorestore \
  --uri="mongodb://localhost:27017" \
  --oplogReplay \
  --oplogLimit="1702134000:1" \
  --drop \
  --gzip \
  /backup/mydb_pit

# Format de --oplogLimit : <timestamp_seconds>:<increment>
# Obtenir le timestamp actuel : date +%s
```

#### 5. Restauration avec Transformation

```bash
# Restaurer dans un namespace différent
mongorestore \
  --uri="mongodb://localhost:27017" \
  --db=mydb_restored \
  --gzip \
  /backup/mydb/mydb

# Restaurer depuis une archive
mongorestore \
  --uri="mongodb://localhost:27017" \
  --archive=/backup/mydb.archive \
  --gzip \
  --drop

# Restaurer depuis stdin (pipe depuis S3)
aws s3 cp s3://my-bucket/backups/mydb-20241208.archive.gz - | \
  mongorestore \
    --uri="mongodb://localhost:27017" \
    --archive \
    --gzip \
    --drop
```

### Options de Performance

```bash
# Parallélisation (4 collections simultanées)
mongorestore \
  --uri="mongodb://localhost:27017" \
  --numParallelCollections=4 \
  --numInsertionWorkersPerCollection=8 \
  --drop \
  --gzip \
  /backup/mydb

# Bypass document validation (plus rapide, attention !)
mongorestore \
  --uri="mongodb://localhost:27017" \
  --bypassDocumentValidation \
  --drop \
  /backup/mydb

# Mode maintainInsertionOrder (préserve l'ordre, plus lent)
mongorestore \
  --uri="mongodb://localhost:27017" \
  --maintainInsertionOrder \
  /backup/mydb
```

### Cas d'Usage Avancés

#### 1. Restauration avec Validation

```bash
#!/bin/bash
# restore_with_validation.sh

BACKUP_DIR="/backup/20241208_120000"
TARGET_DB="mydb"

echo "Starting restoration..."

# 1. Vérifier les checksums
if [ -f "$BACKUP_DIR/checksums.txt" ]; then
  cd $BACKUP_DIR
  sha256sum -c checksums.txt
  if [ $? -ne 0 ]; then
    echo "ERROR: Checksum verification failed!"
    exit 1
  fi
  cd -
fi

# 2. Restaurer
mongorestore \
  --uri="mongodb://localhost:27017" \
  --db=$TARGET_DB \
  --drop \
  --gzip \
  --verbose \
  $BACKUP_DIR/$TARGET_DB

# 3. Vérifier le nombre de documents
EXPECTED_DOCS=$(grep "documents" $BACKUP_DIR/restore.log | awk '{sum+=$1} END {print sum}')
ACTUAL_DOCS=$(mongo --quiet --eval "db.getSiblingDB('$TARGET_DB').stats().objects")

if [ "$EXPECTED_DOCS" != "$ACTUAL_DOCS" ]; then
  echo "WARNING: Document count mismatch! Expected: $EXPECTED_DOCS, Got: $ACTUAL_DOCS"
fi

# 4. Vérifier les index
mongo --eval "
  var db = db.getSiblingDB('$TARGET_DB');
  db.getCollectionNames().forEach(function(coll) {
    print(coll + ': ' + db[coll].getIndexes().length + ' indexes');
  });
"
```

#### 2. Restauration Blue/Green

```bash
#!/bin/bash
# blue_green_restore.sh

BLUE_URI="mongodb://blue.example.com:27017"
GREEN_URI="mongodb://green.example.com:27017"
BACKUP_DIR="/backup/latest"

# 1. Restaurer sur Green (environnement inactif)
mongorestore \
  --uri="$GREEN_URI" \
  --drop \
  --gzip \
  $BACKUP_DIR

# 2. Valider Green
if mongo "$GREEN_URI/mydb" --eval "db.users.count()" > /dev/null 2>&1; then
  echo "Green environment validated"

  # 3. Basculer le trafic (via load balancer)
  curl -X POST "http://loadbalancer:8080/switch" -d "target=green"

  echo "Traffic switched to Green"
else
  echo "ERROR: Green validation failed!"
  exit 1
fi
```

#### 3. Restauration Partielle avec Merge

```bash
#!/bin/bash
# partial_merge_restore.sh

# Restaurer uniquement les documents manquants
mongorestore \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=orders \
  --stopOnError=false \
  --writeConcern="{w: 'majority', j: true}" \
  /backup/mydb/orders.bson

# Les documents avec _id existants sont ignorés (pas d'erreur avec stopOnError=false)
```

---

## mongoexport / mongoimport - Export/Import JSON/CSV

### mongoexport - Export de Données

#### Export JSON (Format Standard)

```bash
# Export collection complète en JSON
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --out=/export/users.json

# Export avec filtre
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=orders \
  --query='{"status": "shipped", "total": {"$gt": 100}}' \
  --out=/export/orders_shipped.json

# Export avec projection (champs spécifiques)
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --fields="email,firstName,lastName,createdAt" \
  --out=/export/users_minimal.json

# Export avec tri
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=products \
  --sort='{"price": -1}' \
  --limit=100 \
  --out=/export/top_100_products.json
```

#### Export CSV

```bash
# Export CSV avec headers
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --type=csv \
  --fields="email,firstName,lastName,country" \
  --out=/export/users.csv

# Export CSV sans headers
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=orders \
  --type=csv \
  --fields="orderId,customerId,total,date" \
  --noHeaderLine \
  --out=/export/orders.csv
```

#### Export Avancé

```bash
# Export avec pretty printing
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --pretty \
  --out=/export/users_pretty.json

# Export JSON Array (toute la collection dans un array)
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --jsonArray \
  --out=/export/users_array.json

# Export vers stdout (pour traitement)
mongoexport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --query='{"country": "FR"}' | \
  jq '.email' | \
  sort -u > /tmp/french_emails.txt
```

### mongoimport - Import de Données

#### Import JSON

```bash
# Import JSON standard (one document per line)
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --file=/import/users.json

# Import JSON Array
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=products \
  --file=/import/products.json \
  --jsonArray

# Import avec upsert (insert ou update)
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --file=/import/users.json \
  --mode=upsert \
  --upsertFields="email"

# Import en écrasant les documents existants
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --drop \
  --file=/import/users.json
```

#### Import CSV

```bash
# Import CSV avec headers
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=sales \
  --type=csv \
  --headerline \
  --file=/import/sales.csv

# Import CSV sans headers (spécifier les champs)
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=sales \
  --type=csv \
  --fields="date,product,quantity,price" \
  --file=/import/sales.csv

# Import CSV avec transformation de types
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=measurements \
  --type=csv \
  --headerline \
  --columnsHaveTypes \
  --fields="timestamp.date(2006-01-02T15:04:05Z07:00),temperature.double(),humidity.int32()" \
  --file=/import/measurements.csv
```

#### Import Avancé

```bash
# Import avec validation ignorée
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=legacy_data \
  --file=/import/legacy.json \
  --bypassDocumentValidation

# Import depuis stdin
cat /import/users.json | \
  mongoimport \
    --uri="mongodb://localhost:27017" \
    --db=mydb \
    --collection=users

# Import avec write concern personnalisé
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=critical_data \
  --file=/import/data.json \
  --writeConcern="{w: 'majority', j: true, wtimeout: 5000}"

# Import parallèle (multiples fichiers)
for file in /import/users_part*.json; do
  mongoimport \
    --uri="mongodb://localhost:27017" \
    --db=mydb \
    --collection=users \
    --file=$file &
done
wait
```

### Cas d'Usage Pratiques

#### 1. Migration SQL → MongoDB

```bash
#!/bin/bash
# sql_to_mongodb.sh

# 1. Export depuis PostgreSQL vers CSV
psql -d mydb -c "COPY users TO STDOUT WITH CSV HEADER" > /tmp/users.csv

# 2. Import dans MongoDB avec transformation
mongoimport \
  --uri="mongodb://localhost:27017" \
  --db=mydb \
  --collection=users \
  --type=csv \
  --headerline \
  --columnsHaveTypes \
  --fields="id.int32(),email.string(),created_at.date(2006-01-02 15:04:05)" \
  --file=/tmp/users.csv

# 3. Créer les index
mongo mongodb://localhost:27017/mydb --eval '
  db.users.createIndex({ email: 1 }, { unique: true });
  db.users.createIndex({ created_at: -1 });
'
```

#### 2. Anonymisation de Données

```bash
#!/bin/bash
# anonymize_export.sh

# Export
mongoexport \
  --uri="mongodb://prod.example.com:27017" \
  --db=mydb \
  --collection=users \
  --out=/tmp/users_prod.json

# Anonymisation (via script Python ou jq)
cat /tmp/users_prod.json | jq '
  .email = (.email | split("@")[0] | . + "@example.com") |
  .phone = "000-000-0000" |
  .ssn = null
' > /tmp/users_anonymized.json

# Import dans environnement de test
mongoimport \
  --uri="mongodb://test.example.com:27017" \
  --db=mydb \
  --collection=users \
  --drop \
  --file=/tmp/users_anonymized.json
```

---

## mongostat - Monitoring en Temps Réel

### Principe de Fonctionnement

`mongostat` affiche les statistiques du serveur MongoDB en temps réel, similaire à `iostat` ou `vmstat` pour les systèmes Linux. C'est l'outil de première ligne pour diagnostiquer rapidement des problèmes de performance.

### Syntaxe et Utilisation

```bash
# Affichage standard (refresh toutes les secondes)
mongostat --uri="mongodb://localhost:27017"

# Output exemple :
insert query update delete getmore command dirty used flushes vsize   res qrw arw net_in net_out conn                time
    *0    *0     *0     *0       0     2|0  0.0% 0.0%       0 1.57G 80.0M 0|0 1|0   158b   38.4k    3 Dec  8 15:23:45.123
     0     5      2      0       0     8|0  0.1% 0.2%       0 1.57G 82.0M 0|0 1|0   2.1k   45.2k    3 Dec  8 15:23:46.123
```

### Colonnes Clés et Interprétation

| Colonne | Description | Normal | Attention | Critique |
|---------|-------------|--------|-----------|----------|
| **insert** | Insertions/sec | < 1000 | 1000-5000 | > 5000 |
| **query** | Queries/sec | < 1000 | 1000-10000 | > 10000 |
| **update** | Updates/sec | < 500 | 500-2000 | > 2000 |
| **delete** | Deletes/sec | < 100 | 100-500 | > 500 |
| **getmore** | Cursor iterations/sec | < 100 | 100-500 | > 500 |
| **command** | Commandes/sec (read\|write) | < 100 | 100-1000 | > 1000 |
| **dirty** | % cache WiredTiger modifié | < 5% | 5-20% | > 20% |
| **used** | % cache WiredTiger utilisé | < 60% | 60-80% | > 80% |
| **flushes** | Checkpoints/sec | 0-1 | 1-2 | > 2 |
| **vsize** | Mémoire virtuelle | - | - | Croissance rapide |
| **res** | Mémoire résidente | < 80% RAM | 80-90% | > 90% |
| **qrw** | Queue: read\|write | 0\|0 | < 10\|10 | > 10\|10 |
| **arw** | Active: read\|write | < 10\|10 | 10-50\|10-50 | > 50\|50 |
| **net_in** | Trafic réseau entrant | - | - | Spike anormal |
| **net_out** | Trafic réseau sortant | - | - | Spike anormal |
| **conn** | Connexions actives | < 100 | 100-500 | > 500 |

### Options Avancées

```bash
# Intervalle personnalisé (5 secondes)
mongostat --uri="mongodb://localhost:27017" 5

# Nombre limité de rafraîchissements
mongostat --uri="mongodb://localhost:27017" --rowcount=20

# Output JSON (pour parsing)
mongostat --uri="mongodb://localhost:27017" -o=json

# Monitoring de Replica Set
mongostat --uri="mongodb://node1:27017,node2:27017,node3:27017/?replicaSet=rs0" --discover

# Colonnes spécifiques
mongostat \
  --uri="mongodb://localhost:27017" \
  -O='host,insert,query,update,delete,dirty,used,conn'

# Mode interactif avec tri
mongostat --uri="mongodb://localhost:27017" --interactive
```

### Patterns de Diagnostic

#### 1. Utilisation Cache Élevée

```
dirty: 25%, used: 95%
```

**Diagnostic** : Cache WiredTiger saturé, risque de ralentissement.

**Actions** :
```bash
# Vérifier les stats du cache
mongo --eval "db.serverStatus().wiredTiger.cache" | grep -E "bytes currently|maximum bytes"

# Augmenter la taille du cache (mongod.conf)
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # Augmenter selon RAM disponible
```

#### 2. Queue de Lectures/Écritures Élevée

```
qrw: 45|23, arw: 128|64
```

**Diagnostic** : Saturation des threads, requêtes en attente.

**Actions** :
```bash
# Identifier les requêtes lentes
mongo --eval "db.currentOp({'active': true, 'secs_running': {'\$gt': 3}})"

# Vérifier les locks
mongo --eval "db.serverStatus().locks"
```

#### 3. Spike de Commandes

```
command: 5000|2000 (normal: 100|50)
```

**Diagnostic** : Pic d'activité, potentiellement anormal.

**Actions** :
```bash
# Analyser les opérations en cours
mongo --eval "db.currentOp().inprog.forEach(function(op) {
  if(op.secs_running > 5) printjson(op);
})"

# Vérifier le profiler
mongo --eval "db.system.profile.find().sort({ts:-1}).limit(10).pretty()"
```

### Script de Monitoring Automatisé

```bash
#!/bin/bash
# monitor_mongodb.sh

ALERT_THRESHOLD_DIRTY=20
ALERT_THRESHOLD_QUEUE=10
ALERT_EMAIL="ops@example.com"

mongostat --uri="mongodb://localhost:27017" --rowcount=1 -o=json | \
  jq -r '.[] |
    select(.dirty > '$ALERT_THRESHOLD_DIRTY' or .qr > '$ALERT_THRESHOLD_QUEUE') |
    "ALERT: dirty=\(.dirty)%, qr=\(.qr)"' | \
  while read alert; do
    echo "$alert" | mail -s "MongoDB Alert" $ALERT_EMAIL
  done
```

---

## mongotop - Monitoring des Opérations par Collection

### Principe de Fonctionnement

`mongotop` affiche le temps passé en lecture/écriture par collection, permettant d'identifier rapidement les collections les plus sollicitées.

### Syntaxe et Utilisation

```bash
# Affichage standard (refresh toutes les secondes)
mongotop --uri="mongodb://localhost:27017"

# Output exemple :
                    ns    total    read    write    2024-12-08T15:23:45Z
       mydb.users    234ms   180ms    54ms
     mydb.orders     89ms    67ms    22ms
   mydb.products     12ms    12ms     0ms
```

### Options

```bash
# Intervalle personnalisé (5 secondes)
mongotop --uri="mongodb://localhost:27017" 5

# Afficher uniquement les N premières collections
mongotop --uri="mongodb://localhost:27017" --rowcount=10

# Output JSON
mongotop --uri="mongodb://localhost:27017" -o=json

# Locks uniquement
mongotop --uri="mongodb://localhost:27017" --locks
```

### Cas d'Usage

#### Identifier les Hot Collections

```bash
# Script pour identifier les collections chaudes
#!/bin/bash

mongotop --uri="mongodb://localhost:27017" --rowcount=1 -o=json | \
  jq -r '.totals[] |
    select(.total.time > 1000) |
    "\(.ns): \(.total.time)ms (read: \(.read.time)ms, write: \(.write.time)ms)"' | \
  sort -t: -k2 -rn
```

---

## bsondump - Inspection de Fichiers BSON

### Utilisation

```bash
# Convertir BSON en JSON
bsondump /backup/mydb/users.bson > users.json

# Avec pretty printing
bsondump --pretty /backup/mydb/users.bson > users_pretty.json

# Format JSON Array
bsondump --array /backup/mydb/users.bson > users_array.json

# Diagnostic : compter les documents
bsondump /backup/mydb/users.bson | wc -l

# Inspecter la structure
bsondump /backup/mydb/users.bson | head -5 | jq 'keys'
```

---

## mongofiles - Gestion GridFS

### Principe

`mongofiles` permet de gérer les fichiers stockés dans GridFS via la ligne de commande.

### Opérations de Base

```bash
# Lister les fichiers
mongofiles --uri="mongodb://localhost:27017" --db=mydb list

# Upload fichier
mongofiles --uri="mongodb://localhost:27017" --db=mydb put /path/to/file.pdf

# Download fichier
mongofiles --uri="mongodb://localhost:27017" --db=mydb get file.pdf

# Supprimer fichier
mongofiles --uri="mongodb://localhost:27017" --db=mydb delete file.pdf

# Rechercher fichiers
mongofiles --uri="mongodb://localhost:27017" --db=mydb search "report"
```

---

## Intégration DevOps

### 1. Scripts Cron pour Backups Automatisés

```bash
# /etc/cron.d/mongodb-backup

# Backup daily à 2h du matin
0 2 * * * mongodb /opt/scripts/mongodb_backup_daily.sh

# Backup weekly (dimanche à 3h)
0 3 * * 0 mongodb /opt/scripts/mongodb_backup_weekly.sh

# Cleanup des anciens backups (tous les jours à 4h)
0 4 * * * mongodb /opt/scripts/mongodb_cleanup_old_backups.sh
```

**Script de backup complet** :

```bash
#!/bin/bash
# /opt/scripts/mongodb_backup_daily.sh

set -euo pipefail

# Configuration
MONGO_URI="mongodb://backupUser:password@localhost:27017/?authSource=admin"
BACKUP_ROOT="/backups/mongodb"
RETENTION_DAYS=30
S3_BUCKET="s3://my-company-backups/mongodb"
SLACK_WEBHOOK="https://hooks.slack.com/services/XXX"

# Variables
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="$BACKUP_ROOT/$TIMESTAMP"
LOG_FILE="/var/log/mongodb_backup.log"

# Fonction de log
log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

# Fonction de notification Slack
notify_slack() {
  local message=$1
  local color=$2
  curl -X POST $SLACK_WEBHOOK \
    -H 'Content-Type: application/json' \
    -d "{\"attachments\":[{\"color\":\"$color\",\"text\":\"$message\"}]}" \
    > /dev/null 2>&1
}

log "=== Starting MongoDB Backup ==="

# 1. Créer le répertoire de backup
mkdir -p $BACKUP_DIR

# 2. Effectuer le backup
log "Dumping databases..."
if mongodump \
  --uri="$MONGO_URI" \
  --oplog \
  --gzip \
  --out=$BACKUP_DIR \
  --numParallelCollections=4 \
  2>&1 | tee -a $LOG_FILE; then

  log "Backup completed successfully"
else
  log "ERROR: Backup failed!"
  notify_slack "❌ MongoDB backup FAILED on $(hostname)" "danger"
  exit 1
fi

# 3. Calculer checksums
log "Calculating checksums..."
find $BACKUP_DIR -type f -name "*.bson.gz" -exec sha256sum {} \; > $BACKUP_DIR/checksums.txt

# 4. Créer une archive
log "Creating archive..."
tar -czf $BACKUP_DIR.tar.gz -C $BACKUP_ROOT $(basename $BACKUP_DIR)

# 5. Upload vers S3
log "Uploading to S3..."
if aws s3 cp $BACKUP_DIR.tar.gz $S3_BUCKET/ --storage-class STANDARD_IA; then
  log "S3 upload successful"
else
  log "WARNING: S3 upload failed"
  notify_slack "⚠️ MongoDB backup completed but S3 upload failed on $(hostname)" "warning"
fi

# 6. Nettoyer les anciens backups locaux
log "Cleaning up old backups..."
find $BACKUP_ROOT -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_ROOT -type d -mtime +7 -delete

# 7. Statistiques
BACKUP_SIZE=$(du -sh $BACKUP_DIR.tar.gz | cut -f1)
BACKUP_DURATION=$(($(date +%s) - $(date -d "$TIMESTAMP" +%s)))

log "=== Backup Summary ==="
log "Size: $BACKUP_SIZE"
log "Duration: ${BACKUP_DURATION}s"
log "Location: $BACKUP_DIR.tar.gz"

# 8. Notification de succès
notify_slack "✅ MongoDB backup completed successfully on $(hostname)\nSize: $BACKUP_SIZE\nDuration: ${BACKUP_DURATION}s" "good"

log "=== Backup Complete ==="
```

### 2. CI/CD - Restauration Automatisée

```yaml
# .gitlab-ci.yml
stages:
  - restore_db
  - test
  - deploy

restore_staging_db:
  stage: restore_db
  only:
    - develop
  script:
    # Download latest backup from S3
    - aws s3 cp s3://my-company-backups/mongodb/latest.tar.gz /tmp/
    - tar -xzf /tmp/latest.tar.gz -C /tmp/

    # Restore to staging
    - |
      mongorestore \
        --uri="mongodb://staging.example.com:27017" \
        --drop \
        --gzip \
        --numParallelCollections=4 \
        /tmp/backup

    # Anonymize sensitive data
    - mongo "mongodb://staging.example.com:27017/mydb" --eval "
        db.users.updateMany(
          {},
          { \$set: {
            email: { \$concat: ['user_', '\$_id', '@example.com'] },
            phone: '000-000-0000'
          }}
        )
      "

    # Verify restoration
    - |
      DOC_COUNT=$(mongo "mongodb://staging.example.com:27017/mydb" --quiet --eval "db.users.count()")
      echo "Restored $DOC_COUNT documents"
  artifacts:
    reports:
      dotenv: restore.env
```

### 3. Kubernetes CronJob

```yaml
# mongodb-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: database
spec:
  schedule: "0 2 * * *"  # 2h du matin chaque jour
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: mongodb-backup
            image: mongo:7.0
            command:
            - /bin/bash
            - -c
            - |
              set -e

              # Backup
              mongodump \
                --uri="mongodb://mongodb-0.mongodb-headless:27017,mongodb-1.mongodb-headless:27017,mongodb-2.mongodb-headless:27017/?replicaSet=rs0" \
                --username=$MONGO_USER \
                --password=$MONGO_PASSWORD \
                --oplog \
                --gzip \
                --archive=/backup/mongodb-$(date +%Y%m%d).archive.gz

              # Upload to S3
              aws s3 cp /backup/mongodb-$(date +%Y%m%d).archive.gz \
                s3://$S3_BUCKET/backups/

              # Cleanup local
              rm /backup/mongodb-$(date +%Y%m%d).archive.gz

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
              value: "my-company-backups"
            volumeMounts:
            - name: backup-volume
              mountPath: /backup
          volumes:
          - name: backup-volume
            persistentVolumeClaim:
              claimName: mongodb-backup-pvc
          restartPolicy: OnFailure
```

### 4. Terraform - Backup Automation

```hcl
# backup_automation.tf

resource "aws_s3_bucket" "mongodb_backups" {
  bucket = "company-mongodb-backups"

  lifecycle_rule {
    enabled = true

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }

  versioning {
    enabled = true
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_lambda_function" "mongodb_backup_monitor" {
  filename      = "lambda_backup_monitor.zip"
  function_name = "mongodb-backup-monitor"
  role          = aws_iam_role.lambda_exec.arn
  handler       = "index.handler"
  runtime       = "python3.11"
  timeout       = 300

  environment {
    variables = {
      S3_BUCKET = aws_s3_bucket.mongodb_backups.id
      SLACK_WEBHOOK = var.slack_webhook_url
    }
  }
}

resource "aws_cloudwatch_event_rule" "backup_check" {
  name                = "mongodb-backup-check"
  schedule_expression = "cron(0 3 * * ? *)"  # 3h UTC chaque jour
}

resource "aws_cloudwatch_event_target" "backup_check_target" {
  rule      = aws_cloudwatch_event_rule.backup_check.name
  target_id = "lambda"
  arn       = aws_lambda_function.mongodb_backup_monitor.arn
}
```

---

## Bonnes Pratiques pour SRE

### 1. Checklist de Backup

```yaml
Backup Strategy:
  ✓ Backup quotidien automatisé
  ✓ Backup hebdomadaire avec rétention longue
  ✓ Point-in-time recovery configuré (--oplog)
  ✓ Compression activée (--gzip)
  ✓ Checksums calculés et vérifiés
  ✓ Upload vers stockage distant (S3/GCS)
  ✓ Chiffrement at-rest et in-transit
  ✓ Tests de restauration mensuels
  ✓ Monitoring et alerting configurés
  ✓ Documentation des procédures

Performance:
  ✓ Parallélisation activée (--numParallelCollections)
  ✓ Read preference: secondary (replica set)
  ✓ Backup pendant heures creuses
  ✓ Monitoring de l'impact sur production

Security:
  ✓ Utilisateur dédié avec rôle backup
  ✓ Authentification forte (certificats si possible)
  ✓ Accès restreint aux backups
  ✓ Audit trail des restaurations
```

### 2. Matrice de Décision : Outil Approprié

| Besoin | Outil Recommandé | Alternative |
|--------|------------------|-------------|
| **Backup complet production** | mongodump --oplog | Snapshot filesystem |
| **Backup sélectif (filtré)** | mongodump --query | mongoexport |
| **Export pour analytics** | mongoexport CSV | mongodump + ETL |
| **Migration vers autre DB** | mongoexport JSON | Application-level |
| **Monitoring temps réel** | mongostat | Prometheus exporter |
| **Debug performance** | mongostat + mongotop | Profiler + explain() |
| **Import bulk data** | mongoimport | Bulk API |
| **Restauration rapide** | mongorestore --archive | mongorestore parallel |

### 3. Monitoring des Database Tools

```bash
#!/bin/bash
# monitor_database_tools.sh

# Métriques Prometheus
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/mongodb_tools

# HELP mongodb_backup_duration_seconds Time taken for backup
# TYPE mongodb_backup_duration_seconds gauge
mongodb_backup_duration_seconds{type="full"} $BACKUP_DURATION

# HELP mongodb_backup_size_bytes Size of backup in bytes
# TYPE mongodb_backup_size_bytes gauge
mongodb_backup_size_bytes{type="full"} $BACKUP_SIZE_BYTES

# HELP mongodb_backup_last_success_timestamp Last successful backup timestamp
# TYPE mongodb_backup_last_success_timestamp gauge
mongodb_backup_last_success_timestamp $(date +%s)

# HELP mongodb_backup_collections_total Number of collections backed up
# TYPE mongodb_backup_collections_total gauge
mongodb_backup_collections_total $COLLECTIONS_COUNT

# HELP mongodb_backup_documents_total Number of documents backed up
# TYPE mongodb_backup_documents_total gauge
mongodb_backup_documents_total $DOCUMENTS_COUNT
EOF
```

### 4. Troubleshooting Guide

#### Problème : mongodump Lent

**Symptômes** :
```
mongodump prend plusieurs heures pour terminer
```

**Diagnostic** :
```bash
# Vérifier les index manquants pour les queries de scan
mongo --eval "db.currentOp({'op': 'query', 'active': true})"

# Vérifier la charge système
iostat -x 1 10
```

**Solutions** :
```bash
# 1. Augmenter la parallélisation
mongodump --numParallelCollections=8 ...

# 2. Utiliser read preference secondary
mongodump --uri="mongodb://.../?readPreference=secondary" ...

# 3. Backup par collections (parallèle manuel)
for coll in $(mongo --quiet --eval "db.getCollectionNames().join(' ')"); do
  mongodump --collection=$coll ... &
done
wait
```

#### Problème : mongorestore Échoue

**Symptômes** :
```
Error: E11000 duplicate key error
```

**Solutions** :
```bash
# 1. Avec --drop pour écraser
mongorestore --drop ...

# 2. Sans --drop, ignorer les duplicates
mongorestore --stopOnError=false ...

# 3. Restaurer dans nouveau namespace
mongorestore --nsFrom="mydb.*" --nsTo="mydb_new.*" ...
```

#### Problème : mongoexport Mémoire Insuffisante

**Symptômes** :
```
JavaScript execution failed: out of memory
```

**Solutions** :
```bash
# 1. Export par batch avec skip/limit
for i in {0..9}; do
  mongoexport \
    --skip=$((i * 100000)) \
    --limit=100000 \
    --out=/export/users_part_$i.json
done

# 2. Export avec filtre temporel
mongoexport \
  --query='{"createdAt": {"$gte": {"$date": "2024-01-01"}, "$lt": {"$date": "2024-02-01"}}}' \
  --out=/export/users_jan.json
```

### 5. Script de Validation Post-Restauration

```bash
#!/bin/bash
# validate_restore.sh

MONGO_URI="mongodb://localhost:27017"
DATABASE="mydb"
REPORT_FILE="restore_validation_$(date +%Y%m%d_%H%M%S).txt"

echo "=== MongoDB Restore Validation ===" > $REPORT_FILE
echo "Date: $(date)" >> $REPORT_FILE
echo "" >> $REPORT_FILE

# 1. Vérifier la connectivité
if mongo "$MONGO_URI" --eval "db.runCommand({ping: 1})" > /dev/null 2>&1; then
  echo "✓ Connectivity: OK" >> $REPORT_FILE
else
  echo "✗ Connectivity: FAILED" >> $REPORT_FILE
  exit 1
fi

# 2. Vérifier les collections
echo "" >> $REPORT_FILE
echo "Collections:" >> $REPORT_FILE
mongo "$MONGO_URI/$DATABASE" --quiet --eval "
  db.getCollectionNames().forEach(function(coll) {
    var count = db[coll].count();
    print('  ' + coll + ': ' + count + ' documents');
  })
" >> $REPORT_FILE

# 3. Vérifier les index
echo "" >> $REPORT_FILE
echo "Indexes:" >> $REPORT_FILE
mongo "$MONGO_URI/$DATABASE" --quiet --eval "
  db.getCollectionNames().forEach(function(coll) {
    var indexes = db[coll].getIndexes().length;
    print('  ' + coll + ': ' + indexes + ' indexes');
  })
" >> $REPORT_FILE

# 4. Vérifier l'intégrité référentielle (exemple)
echo "" >> $REPORT_FILE
echo "Integrity Checks:" >> $REPORT_FILE

# Exemple: vérifier que tous les orderId existent
ORPHAN_COUNT=$(mongo "$MONGO_URI/$DATABASE" --quiet --eval "
  db.orderItems.aggregate([
    {
      \$lookup: {
        from: 'orders',
        localField: 'orderId',
        foreignField: '_id',
        as: 'order'
      }
    },
    { \$match: { order: { \$size: 0 } } },
    { \$count: 'orphans' }
  ]).toArray()[0]?.orphans || 0
")

if [ "$ORPHAN_COUNT" -eq 0 ]; then
  echo "✓ Referential integrity: OK" >> $REPORT_FILE
else
  echo "✗ Referential integrity: $ORPHAN_COUNT orphaned documents" >> $REPORT_FILE
fi

# 5. Vérifier les utilisateurs et rôles
echo "" >> $REPORT_FILE
echo "Users and Roles:" >> $REPORT_FILE
mongo "$MONGO_URI/admin" --quiet --eval "
  db.system.users.find({}, {user: 1, roles: 1}).forEach(function(u) {
    print('  ' + u.user + ': ' + u.roles.map(r => r.role).join(', '));
  })
" >> $REPORT_FILE

echo "" >> $REPORT_FILE
echo "=== Validation Complete ===" >> $REPORT_FILE
echo "Report saved to: $REPORT_FILE"

# Envoyer le rapport par email
mail -s "MongoDB Restore Validation" ops@example.com < $REPORT_FILE
```

---

## Alternatives et Outils Complémentaires

### Comparaison avec d'Autres Solutions

| Outil | Avantages MongoDB Tools | Avantages Alternative |
|-------|------------------------|----------------------|
| **Percona Backup for MongoDB** | Simplicité, officiel | Point-in-time incremental, compression meilleure |
| **mongodump vs Filesystem Snapshot** | Portable, sélectif | Plus rapide, cohérence garantie |
| **mongoexport vs Application Export** | Rapide, natif | Transformation flexible, business logic |
| **mongostat vs PMM** | Léger, immédiat | Historique, dashboards, alerting |
| **mongosh vs Compass** | Scriptable, CI/CD | GUI, visualisation, facilité |

### Intégration avec Percona Backup

```bash
# Installation
curl -s https://repo.percona.com/apt/percona-release_latest.generic_all.deb -o percona-release.deb
sudo dpkg -i percona-release.deb
sudo apt-get update
sudo apt-get install percona-backup-mongodb

# Configuration
pbm config --set storage.type=s3
pbm config --set storage.s3.region=us-east-1
pbm config --set storage.s3.bucket=my-backups
pbm config --set storage.s3.prefix=mongodb/

# Backup
pbm backup --type=logical

# Restauration point-in-time
pbm restore --time="2024-12-08T15:00:00Z"
```

---

## Résumé pour SRE

### Commandes Essentielles

```bash
# Backup production
mongodump --uri="$URI" --oplog --gzip --numParallelCollections=4 --out=/backup

# Restauration
mongorestore --uri="$URI" --drop --gzip --oplogReplay --dir=/backup

# Export pour analytics
mongoexport --uri="$URI" --collection=orders --type=csv --fields="id,date,total" --out=orders.csv

# Import bulk
mongoimport --uri="$URI" --collection=users --drop --jsonArray --file=users.json

# Monitoring
mongostat --uri="$URI" --discover
mongotop --uri="$URI" 5

# Debug BSON
bsondump --pretty backup.bson | jq .
```

### Checklist Pré-Production

```yaml
Testing:
  ✓ Test backup complet sur env staging
  ✓ Test restauration complète
  ✓ Test restauration sélective (une collection)
  ✓ Test point-in-time recovery
  ✓ Mesurer temps de backup/restore
  ✓ Valider impact sur performance

Automation:
  ✓ Cron jobs configurés
  ✓ Scripts de monitoring en place
  ✓ Alerting configuré (backup failed, long duration)
  ✓ Documentation à jour
  ✓ Runbooks créés

Security:
  ✓ Credentials sécurisés (Vault/Secrets Manager)
  ✓ Chiffrement activé
  ✓ Accès restreints (RBAC)
  ✓ Audit trail configuré

Compliance:
  ✓ Rétention conforme aux régulations
  ✓ Anonymisation pour non-prod
  ✓ Tests de restauration documentés
  ✓ DR plan validé
```

---

## Conclusion

Les **MongoDB Database Tools** sont des utilitaires essentiels dans l'arsenal de tout SRE et administrateur MongoDB. Leur maîtrise permet :

1. **Sauvegardes fiables** avec stratégies point-in-time
2. **Restaurations rapides** en cas de besoin
3. **Migrations facilitées** entre environnements
4. **Monitoring proactif** des performances
5. **Automatisation** des tâches répétitives

**Points clés à retenir** :
- Toujours utiliser `--oplog` pour backups de replica sets
- Activer compression (`--gzip`) par défaut
- Paralléliser les opérations (`--numParallelCollections`)
- Valider systématiquement les restaurations
- Monitorer et alerter sur les échecs de backup
- Tester régulièrement les procédures de DR

**Prochaines étapes** :
- Configurer l'automatisation des backups
- Établir les tests de restauration mensuels
- Intégrer dans le pipeline CI/CD
- Former l'équipe aux procédures
- Documenter les runbooks

---

**Références** :
- [MongoDB Database Tools Documentation](https://www.mongodb.com/docs/database-tools/)
- [mongodump Reference](https://www.mongodb.com/docs/database-tools/mongodump/)
- [mongorestore Reference](https://www.mongodb.com/docs/database-tools/mongorestore/)
- [MongoDB Backup Methods](https://www.mongodb.com/docs/manual/core/backups/)

⏭️ [mongostat et mongotop](/13-monitoring-administration/06-mongostat-mongotop.md)
