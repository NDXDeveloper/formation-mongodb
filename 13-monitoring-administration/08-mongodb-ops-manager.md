🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.8 MongoDB Ops Manager

## Introduction

**MongoDB Ops Manager** est la plateforme enterprise de monitoring, automatisation et gestion de MongoDB déployée on-premise. Contrairement aux solutions open-source (Prometheus, Grafana) ou cloud (Atlas), Ops Manager offre une solution complète et intégrée développée par MongoDB Inc., spécifiquement conçue pour gérer des déploiements MongoDB complexes à grande échelle dans des environnements privés.

### Positionnement dans l'Écosystème MongoDB

```
┌─────────────────────────────────────────────────────────────────┐
│                  MongoDB Management Solutions                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MongoDB Atlas          Cloud Manager        Ops Manager        │
│  (SaaS, Cloud)         (SaaS, Hybrid)       (On-Premise)        │
│  ┌──────────────┐      ┌──────────────┐    ┌──────────────┐     │
│  │ • Fully Mgd  │      │ • Monitoring │    │ • Full Stack │     │
│  │ • Auto-Scale │      │ • Backup     │    │ • On-Prem    │     │
│  │ • Multi-Cloud│      │ • Alerts     │    │ • Air-Gapped │     │
│  │ • Serverless │      │ • Self-Host  │    │ • Enterprise │     │
│  └──────────────┘      └──────────────┘    └──────────────┘     │
│         │                     │                    │            │
│         └─────────────────────┴────────────────────┘            │
│                    Same UI & Features                           │
└─────────────────────────────────────────────────────────────────┘
```

**Comparaison rapide** :

| Aspect | Ops Manager | Cloud Manager | MongoDB Atlas |
|--------|-------------|---------------|---------------|
| **Déploiement** | On-premise | SaaS | SaaS |
| **Contrôle infrastructure** | Total | Partiel | Aucun |
| **Compliance** | Data reste on-prem | Data metadata cloud | Data in cloud |
| **Cost model** | License + infrastructure | License per node | Pay-as-you-go |
| **Use case** | Enterprise, régulé, air-gapped | Hybrid cloud | Cloud-first |

### Fonctionnalités Principales

Ops Manager offre **trois piliers fonctionnels** :

#### 1. Monitoring
- Métriques en temps réel (100+ métriques)
- Alerting configurable multi-niveaux
- Dashboards personnalisables
- Historical trending et capacity planning
- Performance advisor

#### 2. Automation
- Déploiement automatisé de clusters
- Upgrades rolling sans downtime
- Configuration management centralisé
- Self-healing et auto-remediation
- Topology changes automatisés

#### 3. Backup
- Continuous backup (oplog-based)
- Point-in-time recovery
- Snapshot scheduling
- Queryable backups
- Automated testing de restauration

### Architecture Technique

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Ops Manager Architecture                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     Ops Manager Application                         │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│   │  Web UI      │  │  REST API    │  │  Query Engine│              │
│   │  (port 8080) │  │  (port 8080) │  │              │              │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│          │                 │                 │                      │
│          └─────────────────┴─────────────────┘                      │
│                            │                                        │
│  ┌─────────────────────────▼──────────────────────────┐             │
│  │     Application Database (MongoDB)                 │             │
│  │     - Monitoring data                              │             │
│  │     - Configuration                                │             │
│  │     - Automation state                             │             │
│  └────────────────────────────────────────────────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Monitoring      │ │ Automation      │ │ Backup          │
│ Agent           │ │ Agent           │ │ Daemon          │
│ (per mongod)    │ │ (per server)    │ │ (per RS/cluster)│
│                 │ │                 │ │                 │
│ • Collect stats │ │ • Deploy mongod │ │ • Oplog capture │
│ • Push to OM    │ │ • Config mgmt   │ │ • Snapshots     │
│ • Health checks │ │ • Upgrades      │ │ • S3/filesystem │
└────────┬────────┘ └────────┬────────┘ └────────┬────────┘
         │                   │                   │
         └───────────────────┼───────────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │   MongoDB Deployments        │
              │   • Standalone               │
              │   • Replica Sets             │
              │   • Sharded Clusters         │
              └──────────────────────────────┘
```

### Composants Clés

#### Application Server
- Interface Web et API REST
- Moteur d'alerting
- Planificateur d'automation
- Stocke la configuration et les métriques

#### Monitoring Agent
- Un agent par processus mongod/mongos
- Collecte serverStatus() toutes les 60s
- Push vers Ops Manager
- Lightweight (~50 MB RAM)

#### Automation Agent
- Un agent par serveur physique
- Gère le cycle de vie des processus MongoDB
- Applique la configuration désirée
- Effectue les upgrades/changements

#### Backup Daemon
- Un par replica set ou cluster shardé
- Lit l'oplog en continu
- Crée des snapshots périodiques
- Gère la retention

---

## Installation et Configuration

### Prérequis Système

**Ops Manager Application Server** :

| Ressource | Minimum | Recommandé Production | Large Scale |
|-----------|---------|----------------------|-------------|
| CPU | 4 cores | 8 cores | 16+ cores |
| RAM | 8 GB | 16 GB | 32+ GB |
| Disque | 50 GB | 200 GB SSD | 500+ GB SSD |
| OS | RHEL 7/8, Ubuntu 18.04+ | RHEL 8 | RHEL 8 |

**Application Database** (MongoDB backing Ops Manager) :
- Replica Set 3 membres minimum
- MongoDB 4.4+ (6.0+ recommandé)
- 16 GB RAM minimum par membre
- SSD storage

**Agents** (par serveur géré) :
- 512 MB RAM (Monitoring)
- 256 MB RAM (Automation)
- 1-4 GB RAM (Backup Daemon, selon charge)

### Installation de l'Application

#### 1. Préparation du Système

```bash
# RHEL 8
sudo dnf install -y cyrus-sasl cyrus-sasl-plain cyrus-sasl-gssapi \
  krb5-libs libcurl net-snmp net-snmp-agent-libs openldap openssl \
  tcp_wrappers-libs

# Créer l'utilisateur système
sudo useradd -r -d /opt/mongodb/mms -s /bin/false mongodb-mms

# Créer les répertoires
sudo mkdir -p /opt/mongodb/mms
sudo chown mongodb-mms:mongodb-mms /opt/mongodb/mms
```

#### 2. Téléchargement et Installation

```bash
# Télécharger Ops Manager (nécessite compte MongoDB Enterprise)
VERSION="7.0.0"
wget https://downloads.mongodb.com/on-prem-mms/rpm/mongodb-mms-${VERSION}.x86_64.rpm

# Installer
sudo rpm -ivh mongodb-mms-${VERSION}.x86_64.rpm

# Vérifier l'installation
ls -la /opt/mongodb/mms/
# conf/
# bin/
# jdk/
# logs/
```

#### 3. Configuration Application Database

Ops Manager nécessite sa propre base MongoDB (backing database) :

```javascript
// Déployer un replica set dédié
// rs-opsmgr-0, rs-opsmgr-1, rs-opsmgr-2

// Sur chaque membre
mongod --replSet opsmgr --port 27017 --dbpath /data/opsmgr

// Initialiser le replica set
rs.initiate({
  _id: "opsmgr",
  members: [
    { _id: 0, host: "rs-opsmgr-0:27017" },
    { _id: 1, host: "rs-opsmgr-1:27017" },
    { _id: 2, host: "rs-opsmgr-2:27017" }
  ]
})

// Créer l'utilisateur pour Ops Manager
use admin
db.createUser({
  user: "mmsAppUser",
  pwd: "strongPassword123!",
  roles: [
    { role: "readWriteAnyDatabase", db: "admin" },
    { role: "clusterAdmin", db: "admin" },
    { role: "userAdminAnyDatabase", db: "admin" }
  ]
})
```

#### 4. Configuration Ops Manager

```properties
# /opt/mongodb/mms/conf/conf-mms.properties

# MongoDB connection (backing database)
mongo.mongoUri=mongodb://mmsAppUser:strongPassword123!@rs-opsmgr-0:27017,rs-opsmgr-1:27017,rs-opsmgr-2:27017/?authSource=admin&replicaSet=opsmgr

# Base URL (accessible par agents et browsers)
mms.centralUrl=https://opsmgr.example.com:8080

# Email configuration (SMTP)
mms.mail.transport=smtp
mms.mail.hostname=smtp.example.com
mms.mail.port=587
mms.mail.username=opsmgr@example.com
mms.mail.password=emailPassword
mms.mail.tls=true
mms.fromEmailAddr=opsmgr@example.com

# Backup storage
mms.backup.fsstore.path=/data/backup

# Security
mms.https.PEMKeyFile=/opt/mongodb/mms/conf/certs/server.pem
mms.https.ClientCertificateMode=optional

# Performance tuning
mms.maxActiveJobs=30
mms.queryableBackupMaxConcurrentRestores=4

# Logging
mms.log.path=/opt/mongodb/mms/logs
mms.log.level=INFO
```

#### 5. Démarrage et Initialisation

```bash
# Démarrer le service
sudo systemctl start mongodb-mms

# Vérifier les logs
sudo tail -f /opt/mongodb/mms/logs/mms0.log

# Output attendu
# [main] INFO  com.xgen.svc.mms.Application - Starting Ops Manager
# [main] INFO  com.mongodb.StartupService - All services started successfully

# Activer au démarrage
sudo systemctl enable mongodb-mms

# Vérifier le statut
sudo systemctl status mongodb-mms
```

#### 6. Configuration Initiale via Web UI

```bash
# Accéder à l'interface web
https://opsmgr.example.com:8080

# 1. Créer le premier utilisateur admin
#    Username: admin
#    Email: admin@example.com
#    Password: ComplexPassword123!

# 2. Configurer l'organisation
#    Name: Production
#    Company: Example Corp

# 3. Télécharger les agents
#    - Monitoring Agent
#    - Automation Agent
#    - Backup Agent

# 4. Générer les clés API
#    Admin → Settings → Public API Access → Generate
```

### Installation des Agents

#### Monitoring Agent

```bash
# Sur chaque serveur MongoDB à monitorer
VERSION="12.0.28.7750"

# Téléchargement
curl -OL https://downloads.mongodb.com/on-prem-mms/monitoring-agent/mongodb-mms-monitoring-agent-${VERSION}.x86_64.rpm

# Installation
sudo rpm -ivh mongodb-mms-monitoring-agent-${VERSION}.x86_64.rpm

# Configuration
sudo vi /etc/mongodb-mms/monitoring-agent.config

# Contenu
mmsBaseUrl=https://opsmgr.example.com:8080
mmsGroupId=YOUR_PROJECT_ID
mmsApiKey=YOUR_API_KEY

# Monitoring de tous les processus sur le serveur
# L'agent découvrira automatiquement les mongod/mongos

# Démarrage
sudo systemctl start mongodb-mms-monitoring-agent
sudo systemctl enable mongodb-mms-monitoring-agent

# Vérification
sudo systemctl status mongodb-mms-monitoring-agent
tail -f /var/log/mongodb-mms/monitoring-agent.log
```

#### Automation Agent

```bash
# Installation
curl -OL https://downloads.mongodb.com/on-prem-mms/automation-agent/mongodb-mms-automation-agent-${VERSION}.x86_64.rpm
sudo rpm -ivh mongodb-mms-automation-agent-${VERSION}.x86_64.rpm

# Configuration
sudo vi /etc/mongodb-mms/automation-agent.config

mmsBaseUrl=https://opsmgr.example.com:8080
mmsGroupId=YOUR_PROJECT_ID
mmsApiKey=YOUR_API_KEY

# User pour exécuter mongod
mmsConfigBackup=/var/lib/mongodb-mms-automation/mms-cluster-config-backup.json
logPath=/var/log/mongodb-mms/automation-agent.log

# Démarrage
sudo systemctl start mongodb-mms-automation-agent
sudo systemctl enable mongodb-mms-automation-agent
```

---

## Monitoring et Métriques

### Dashboard Overview

Ops Manager offre plusieurs niveaux de dashboards :

#### 1. Deployment View (Vue Cluster)

```
Deployment: Production Replica Set (rs0)
├─ Status: ✓ Healthy
├─ Version: MongoDB 7.0.4
├─ Members: 3 (1 Primary, 2 Secondaries)
├─ Oplog Window: 48 hours
├─ Data Size: 1.2 TB
└─ Active Alerts: 0

Real-time Metrics:
┌─────────────────────────────────────────────────┐
│ Operations/sec:     1,234                       │
│ Connections:        87 / 65,536                 │
│ Network In/Out:     15 MB/s / 45 MB/s           │
│ Memory (Resident):  8.2 GB / 16 GB              │
│ Replication Lag:    0.2s (max)                  │
└─────────────────────────────────────────────────┘
```

#### 2. Server View (Vue Serveur)

Métriques par serveur incluent :

**System Metrics** :
- CPU utilization (user, system, iowait)
- Memory (resident, virtual, mapped)
- Disk I/O (IOPS, throughput, latency)
- Network (bytes in/out, connections)

**MongoDB Metrics** :
- Operations (insert, query, update, delete, getmore, command)
- Queues (read, write)
- Active connections
- Page faults
- Opcounters

**Replication Metrics** :
- Replication lag (per secondary)
- Oplog GB/hour
- Replication headroom

### Métriques Critiques et Seuils

Ops Manager collecte **100+ métriques** automatiquement. Voici les plus critiques :

#### Operations Metrics

```
Metric: Operations per Second
Path: Performance > Operations
Query: opcounters (insert + query + update + delete + command)

Seuils SRE:
- Normal:    < 1,000 ops/sec
- Elevated:  1,000 - 5,000 ops/sec
- High:      5,000 - 10,000 ops/sec
- Critical:  > 10,000 ops/sec

Action si > 10k:
1. Vérifier si spike attendu (déploiement, batch job)
2. Analyser query patterns via Performance Advisor
3. Vérifier index usage
4. Considérer scale-out si sustained
```

#### Memory Metrics

```
Metric: Memory - Resident
Path: Hardware > Memory
Query: mem.resident (MB)

Analysis:
resident_mb / total_system_ram_mb = memory_utilization

Seuils:
- Healthy:   < 75% total RAM
- Warning:   75-85% total RAM
- Critical:  > 85% total RAM

Correlation:
Si resident élevé + high page faults:
  → Working set > RAM → Considérer upgrade RAM
Si resident élevé + low page faults:
  → Normal, cache est bien utilisé
```

#### Cache Metrics

```
Metric: WiredTiger Cache - Dirty Bytes
Path: Performance > WiredTiger Cache

dirty_percent = (dirty_bytes / max_configured_bytes) * 100

Seuils:
- Normal:    < 10%
- Attention: 10-20%
- Warning:   20-40%
- Critical:  > 40%

Pattern Analysis:
dirty% croissant:
  → Writes > disk flush capacity
  → Check: I/O wait, checkpoint duration
  → Action: Faster disks, adjust checkpoint interval
```

#### Replication Lag

```
Metric: Replication Lag
Path: Replica Set > Replication Lag
Query: (Primary optime - Secondary optime)

Seuils temporels:
- Normal:    < 10 seconds
- Elevated:  10-30 seconds
- Warning:   30-60 seconds
- Critical:  > 60 seconds

Seuils opérationnels:
- oplog_hours_remaining < 24 hours → Warning
- oplog_hours_remaining < 12 hours → Critical

Investigation:
1. Vérifier network latency entre membres
2. Analyser slow queries sur secondary
3. Vérifier I/O performance
4. Considérer write concern adjustment
```

### Performance Advisor

Ops Manager inclut un **Performance Advisor** qui analyse automatiquement :

#### Index Suggestions

```
Analysis Window: Last 24 hours
Slow Queries Analyzed: 1,247

Recommendations:
┌────────────────────────────────────────────────────────────┐
│ Collection: production.orders                              │
│ Impact: HIGH (487 slow queries)                            │
│ Suggestion: Create index { customerId: 1, orderDate: -1 }  │
│ Expected Improvement: 92% reduction in execution time      │
│                                                            │
│ Sample Query:                                              │
│ db.orders.find({ customerId: "C123" }).sort({ orderDate: -1 })
│                                                            │
│ Current Plan: COLLSCAN (450k docs examined)                │
│ Proposed Plan: IXSCAN (8 docs examined)                    │
│                                                            │
│ [Create Index] [Ignore] [Details]                          │
└────────────────────────────────────────────────────────────┘
```

**Création automatisée** :
```javascript
// Via Ops Manager UI ou API
// L'index sera créé en background sur tous les membres

// Équivalent MongoDB shell
db.orders.createIndex(
  { customerId: 1, orderDate: -1 },
  { background: true }
)
```

#### Query Insights

```
Top 10 Slowest Query Shapes (Last 7 Days)

1. Collection: users
   Pattern: find({ email: ? }).sort({ createdAt: -1 })
   Avg Duration: 1,234 ms
   Executions: 45,678
   Total Time: 15.6 hours
   → Missing index on email field

2. Collection: logs
   Pattern: aggregate([{ $match: { level: ? } }, { $group: ... }])
   Avg Duration: 892 ms
   Executions: 12,345
   Total Time: 3.1 hours
   → Consider index on level, use $match early in pipeline
```

### Custom Charts et Dashboards

Ops Manager permet de créer des **custom charts** pour métriques spécifiques :

#### Exemple : Custom Dashboard "Application Performance"

```
Chart 1: Application Operation Mix
Type: Stacked Area
Metrics:
  - opcounters.insert (rate)
  - opcounters.query (rate)
  - opcounters.update (rate)
  - opcounters.delete (rate)
Display: Last 24 hours, 5-minute granularity

Chart 2: P95 Query Latency by Collection
Type: Line
Metrics:
  - top.total.time (p95) for top 5 collections
Display: Last 7 days, 1-hour granularity

Chart 3: Cache Efficiency
Type: Multi-line
Metrics:
  - wiredTiger.cache.bytes currently in the cache / max (%)
  - wiredTiger.cache.tracked dirty bytes / max (%)
  - wiredTiger.cache.pages evicted (rate)

Chart 4: Connection Pool Health
Type: Gauge
Metrics:
  - connections.current
  - connections.available
Thresholds:
  - Green: < 70% used
  - Yellow: 70-85% used
  - Red: > 85% used
```

---

## Alerting

### Configuration d'Alertes

Ops Manager offre un système d'alerting multi-niveaux avec **50+ conditions prédéfinies**.

#### Types d'Alertes

```
┌─────────────────────────────────────────────────────────────┐
│                    Alert Categories                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Host Alerts          Process Alerts      Backup Alerts     │
│  ├─ CPU > 80%         ├─ Primary Down     ├─ Backup Failed  │
│  ├─ Memory > 90%      ├─ Too Many         ├─ Snapshot Old   │
│  ├─ Disk Space < 10%  │   Connections     └─ No Recent Snap │
│  └─ Network Issues    ├─ Replication Lag                    │
│                       ├─ Oplog < 1 hour                     │
│                       └─ Assert Warnings                    │
│                                                             │
│  User Alerts          Backup Job Alerts                     │
│  ├─ Failed Login      ├─ Snapshot Taking                    │
│  ├─ User Changes      │   Too Long                          │
│  └─ API Key Changes   └─ Queryable Backup                   │
│                           Stale                             │
└─────────────────────────────────────────────────────────────┘
```

#### Configuration d'Alerte Standard

**Exemple : Replication Lag Alert**

```json
{
  "typeName": "REPLICATION_LAG",
  "enabled": true,
  "eventTypeName": "REPLICATION_OPLOG_WINDOW_RUNNING_OUT",
  "threshold": {
    "value": 1,
    "units": "HOURS"
  },
  "notifications": [
    {
      "typeName": "EMAIL",
      "emailEnabled": true,
      "intervalMin": 5,
      "delayMin": 0,
      "emailAddress": "dba-oncall@example.com"
    },
    {
      "typeName": "PAGER_DUTY",
      "serviceKey": "YOUR_PAGERDUTY_KEY",
      "intervalMin": 0,
      "delayMin": 0
    },
    {
      "typeName": "SLACK",
      "channelName": "#mongodb-alerts",
      "apiToken": "YOUR_SLACK_TOKEN",
      "intervalMin": 5
    }
  ],
  "matchers": [
    {
      "fieldName": "REPLICA_SET_NAME",
      "operator": "EQUALS",
      "value": "production-rs0"
    }
  ]
}
```

#### Configuration via API

```bash
# Créer une alerte via REST API
curl -X POST \
  "https://opsmgr.example.com:8080/api/public/v1.0/groups/${PROJECT_ID}/alertConfigs" \
  -u "${PUBLIC_KEY}:${PRIVATE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "typeName": "HOST_METRIC",
    "metricName": "DISK_PARTITION_SPACE_USED_DATA",
    "mode": "AVERAGE",
    "operator": "GREATER_THAN",
    "threshold": 80,
    "units": "RAW",
    "enabled": true,
    "notifications": [
      {
        "typeName": "EMAIL",
        "emailAddress": "ops@example.com",
        "intervalMin": 15,
        "delayMin": 0
      }
    ]
  }'
```

#### Alert Escalation

Configuration d'escalade multi-niveaux :

```json
{
  "alertConfigId": "5f2c8b1234567890abcdef12",
  "typeName": "REPLICATION_LAG",
  "notifications": [
    {
      "typeName": "EMAIL",
      "emailAddress": "team-mongodb@example.com",
      "delayMin": 0,
      "intervalMin": 5
    },
    {
      "typeName": "PAGER_DUTY",
      "serviceKey": "PRIMARY_ONCALL_KEY",
      "delayMin": 5,
      "intervalMin": 0
    },
    {
      "typeName": "PAGER_DUTY",
      "serviceKey": "SECONDARY_ONCALL_KEY",
      "delayMin": 15,
      "intervalMin": 0
    }
  ]
}
```

**Logique d'escalade** :
1. t=0 : Email à l'équipe
2. t=5min : Page on-call primary (si non résolu)
3. t=15min : Page on-call secondary (si non résolu)
4. Répéter email toutes les 5 minutes jusqu'à résolution

### Suppression d'Alertes (Maintenance Windows)

```bash
# Créer une fenêtre de maintenance
curl -X POST \
  "https://opsmgr.example.com:8080/api/public/v1.0/groups/${PROJECT_ID}/maintenanceWindows" \
  -u "${PUBLIC_KEY}:${PRIVATE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2024-12-15T02:00:00Z",
    "endDate": "2024-12-15T04:00:00Z",
    "description": "Rolling upgrade to MongoDB 7.0.5",
    "alertTypeNames": [
      "HOST_DOWN",
      "REPLICA_SET_ELECTION",
      "PRIMARY_ELECTED"
    ]
  }'
```

---

## Automation et Déploiement

### Concept d'Automation

Ops Manager utilise un modèle **"desired state"** :

```
┌─────────────────────────────────────────────────────────────┐
│              Automation Desired State Model                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Administrator configures desired state in UI            │
│     ↓                                                       │
│  2. Ops Manager stores configuration                        │
│     ↓                                                       │
│  3. Automation Agents poll for configuration changes        │
│     ↓                                                       │
│  4. Agents compare current vs. desired state                │
│     ↓                                                       │
│  5. Agents execute changes to reach desired state           │
│     ↓                                                       │
│  6. Agents report back current state                        │
│     ↓                                                       │
│  7. Ops Manager validates goal state reached                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Déploiement Automatisé de Replica Set

#### Via UI

```
Deployment → Add New Deployment → Replica Set

Configuration:
├─ Name: production-rs0
├─ MongoDB Version: 7.0.4
├─ Members: 3
│  ├─ Host: mongo-prod-01:27017
│  ├─ Host: mongo-prod-02:27017
│  └─ Host: mongo-prod-03:27017
├─ Oplog Size: 10 GB
├─ Storage Engine: WiredTiger
├─ Authentication: SCRAM-SHA-256
└─ TLS: Enabled

[Review & Deploy]
```

#### Via API

```bash
# Définition JSON du replica set
cat > replica-set-config.json <<'EOF'
{
  "name": "production-rs0",
  "processes": [
    {
      "hostname": "mongo-prod-01",
      "port": 27017,
      "processType": "mongod",
      "version": "7.0.4",
      "featureCompatibilityVersion": "7.0",
      "args2_6": {
        "net": {
          "port": 27017,
          "bindIp": "0.0.0.0"
        },
        "storage": {
          "dbPath": "/data/mongodb",
          "engine": "wiredTiger"
        },
        "systemLog": {
          "destination": "file",
          "path": "/var/log/mongodb/mongod.log"
        },
        "replication": {
          "replSetName": "production-rs0",
          "oplogSizeMB": 10240
        },
        "security": {
          "authorization": "enabled"
        }
      }
    },
    {
      "hostname": "mongo-prod-02",
      "port": 27017,
      "processType": "mongod",
      "version": "7.0.4",
      "featureCompatibilityVersion": "7.0",
      "args2_6": {
        // ... similar config
      }
    },
    {
      "hostname": "mongo-prod-03",
      "port": 27017,
      "processType": "mongod",
      "version": "7.0.4",
      "featureCompatibilityVersion": "7.0",
      "args2_6": {
        // ... similar config
      }
    }
  ],
  "replicaSets": [
    {
      "id": "production-rs0",
      "members": [
        {
          "id": 0,
          "hostname": "mongo-prod-01",
          "port": 27017,
          "priority": 1,
          "votes": 1
        },
        {
          "id": 1,
          "hostname": "mongo-prod-02",
          "port": 27017,
          "priority": 1,
          "votes": 1
        },
        {
          "id": 2,
          "hostname": "mongo-prod-03",
          "port": 27017,
          "priority": 1,
          "votes": 1
        }
      ]
    }
  ]
}
EOF

# Déployer via API
curl -X PUT \
  "https://opsmgr.example.com:8080/api/public/v1.0/groups/${PROJECT_ID}/automationConfig" \
  -u "${PUBLIC_KEY}:${PRIVATE_KEY}" \
  -H "Content-Type: application/json" \
  -d @replica-set-config.json
```

### Upgrades Rolling Automatisés

#### Upgrade MongoDB 6.0 → 7.0

**Processus automatique** :

```
Phase 1: Compatibility Version Upgrade
├─ Set FCV to 6.0 (if not already)
└─ Verify all nodes running 6.0

Phase 2: Binary Upgrade - Secondaries
├─ Stop secondary-1
├─ Replace binary 6.0 → 7.0
├─ Start secondary-1
├─ Wait for sync (< 1s lag)
├─ Stop secondary-2
├─ Replace binary 6.0 → 7.0
├─ Start secondary-2
├─ Wait for sync
└─ All secondaries upgraded

Phase 3: Primary Stepdown and Upgrade
├─ Step down primary → secondary-1 becomes primary
├─ Stop old primary (now secondary)
├─ Replace binary 6.0 → 7.0
├─ Start as secondary
└─ Wait for sync

Phase 4: FCV Upgrade
├─ Set FCV to 7.0
└─ New features now available

Estimated Downtime: 0 seconds (rolling)
Estimated Duration: 15-30 minutes (3 members)
```

**Configuration via UI** :

```
Deployment → Settings → Modify Deployment

1. Change MongoDB Version: 7.0.4
2. [Review Changes]
3. Confirmation:
   ┌─────────────────────────────────────────────────┐
   │ Changes to be applied:                          │
   │ • MongoDB version: 6.0.11 → 7.0.4               │
   │ • Rolling upgrade: Yes                          │
   │ • Estimated duration: 20 minutes                │
   │ • Downtime: None (rolling)                      │
   │                                                 │
   │ This will:                                      │
   │ 1. Upgrade secondaries first                    │
   │ 2. Step down primary                            │
   │ 3. Upgrade old primary                          │
   │                                                 │
   │ [Cancel] [Confirm & Deploy]                     │
   └─────────────────────────────────────────────────┘
```

### Configuration Management

#### Modification Centralisée

Changer un paramètre sur tout le replica set :

```
Example: Enable Profiling Level 1

Deployment → Configuration → Edit

Add to mongod arguments:
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

[Review & Deploy]

Result:
- Automation agents update mongod.conf on all members
- Rolling restart if needed
- Change tracked in audit log
```

#### Configuration Drift Detection

Ops Manager détecte les changements manuels (drift) :

```
Warning: Configuration Drift Detected

Server: mongo-prod-02
Process: mongod (port 27017)

Detected Changes:
┌────────────────────────────────────────────────────┐
│ Parameter            Expected    Actual            │
├────────────────────────────────────────────────────┤
│ oplogSize            10240 MB    5120 MB           │
│ cacheSizeGB          8 GB        4 GB              │
└────────────────────────────────────────────────────┘

Actions:
[Revert to Managed Config] [Update Managed Config] [Ignore]
```

---

## Backup et Restauration

### Architecture de Backup

```
┌───────────────────────────────────────────────────────────┐
│                  Backup Architecture                      │
└───────────────────────────────────────────────────────────┘

Replica Set (production-rs0)
├─ Primary
├─ Secondary-1
└─ Secondary-2
     │
     │ oplog tail (continuous)
     │
     ▼
┌────────────────────────┐
│  Backup Daemon         │
│  • Reads oplog         │
│  • Creates snapshots   │
│  • Manages retention   │
└────────┬───────────────┘
         │
         │ store snapshots
         │
         ▼
┌────────────────────────────────────────────────┐
│  Backup Storage                                │
│  ┌──────────────┐  ┌──────────────┐            │
│  │  Filesystem  │  │     S3       │            │
│  │   Blockstore │  │  Compatible  │            │
│  └──────────────┘  └──────────────┘            │
│                                                │
│  Snapshots:                                    │
│  ├─ 2024-12-08 00:00 (Base)                    │
│  ├─ 2024-12-09 00:00 (Incremental)             │
│  ├─ 2024-12-10 00:00 (Incremental)             │
│  └─ ... + continuous oplog                     │
└────────────────────────────────────────────────┘
         │
         │ queryable backup
         │
         ▼
┌────────────────────────┐
│  Queryable Snapshots   │
│  (mongod read-only)    │
└────────────────────────┘
```

### Configuration de Backup

#### 1. Activation du Backup

```
Deployment → Backup → Enable Backup

Configuration:
├─ Snapshot Schedule
│  ├─ Snapshot Interval: Daily at 02:00 UTC
│  ├─ Snapshot Retention:
│  │  ├─ Daily: 7 days
│  │  ├─ Weekly: 4 weeks
│  │  └─ Monthly: 12 months
│  └─ Continuous Oplog: Enabled
│
├─ Storage Configuration
│  ├─ Type: S3-Compatible
│  ├─ Endpoint: s3.amazonaws.com
│  ├─ Bucket: mongodb-backups-prod
│  └─ Encryption: AES-256
│
└─ Backup Daemon Allocation
   ├─ Assigned Server: backup-daemon-01
   └─ Backup Throughput: 100 MB/s

[Save & Enable]
```

#### 2. Configuration Avancée

```json
{
  "backupConfigs": [
    {
      "groupId": "${PROJECT_ID}",
      "clusterId": "production-rs0",
      "statusName": "STARTED",
      "syncSource": "SECONDARY",
      "snapshotIntervalHours": 24,
      "snapshotRetentionDays": 7,
      "pointInTimeWindowHours": 48,
      "provisioned": true,
      "blockstore": {
        "type": "S3",
        "s3BucketName": "mongodb-backups-prod",
        "s3BucketEndpoint": "s3.amazonaws.com",
        "sseEnabled": true,
        "maxCapacityGB": 5000
      }
    }
  ]
}
```

### Point-in-Time Recovery

#### Interface de Restauration

```
Backup → Restore

Source Snapshot:
├─ Replica Set: production-rs0
├─ Snapshot: 2024-12-10 00:00:00
└─ Point-in-Time: 2024-12-10 15:30:45

Restore Target:
├─ Create New Cluster: restore-rs0
├─ Hosts:
│  ├─ restore-host-01:27017
│  ├─ restore-host-02:27017
│  └─ restore-host-03:27017
└─ MongoDB Version: 7.0.4

Options:
├─ Download Snapshot Only: No
├─ Restore System Databases: No
└─ Oplog Replay: To 2024-12-10 15:30:45

[Start Restore]

Progress:
┌────────────────────────────────────────────────┐
│ ████████████████░░░░░░░░░░ 65%                 │
│                                                │
│ Phase: Applying Oplog                          │
│ Elapsed: 12m 34s                               │
│ Estimated Remaining: 6m 15s                    │
│                                                │
│ Operations Replayed: 1,234,567 / 1,900,000     │
└────────────────────────────────────────────────┘
```

#### Restauration via API

```bash
# Initier une restauration
curl -X POST \
  "https://opsmgr.example.com:8080/api/public/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_ID}/restoreJobs" \
  -u "${PUBLIC_KEY}:${PRIVATE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "snapshotId": "5f2c8b1234567890abcdef12",
    "deliveryType": "AUTOMATED_RESTORE",
    "targetClusterId": "restore-cluster-id",
    "pointInTimeUTCMillis": 1702220445000,
    "oplogInc": 1,
    "oplogTs": "1702220445:1"
  }'

# Surveiller le statut
curl "https://opsmgr.example.com:8080/api/public/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_ID}/restoreJobs/${JOB_ID}" \
  -u "${PUBLIC_KEY}:${PRIVATE_KEY}"

# Response
{
  "id": "restore-job-123",
  "statusName": "IN_PROGRESS",
  "pointInTimeUTCMillis": 1702220445000,
  "created": "2024-12-10T16:00:00Z",
  "targetClusterId": "restore-cluster-id",
  "percentComplete": 65
}
```

### Queryable Backups

Fonctionnalité unique : **interroger directement les snapshots** sans restauration complète.

```
Backup → Snapshots → [Select Snapshot] → Query

Un mongod read-only temporaire est créé avec le snapshot:
mongodb://queryable-backup-host:27017/

Use case:
- Vérifier des données avant restauration complète
- Extraire des documents spécifiques
- Audit et compliance
- Analyse forensique

Example:
mongo mongodb://queryable-backup-host:27017/production

> db.orders.find({ orderId: "ORD-12345" })
> db.users.count({ deletedAt: { $exists: false } })
```

### Automated Restore Testing

```yaml
# Configuration test de restauration mensuel
backupTestConfig:
  enabled: true
  schedule: "0 2 1 * *"  # 1er de chaque mois à 2h
  targetCluster: "restore-test-cluster"
  validationQueries:
    - database: production
      collection: users
      query: { active: true }
      expectedMinCount: 100000
    - database: production
      collection: orders
      query: { status: "completed" }
      expectedMinCount: 50000
  notificationEmail: dba-team@example.com
  retentionDays: 1  # Cleanup après validation
```

---

## Sécurité et Authentification

### Authentication & Authorization

#### LDAP/Active Directory Integration

```properties
# conf-mms.properties

# LDAP configuration
mms.ldap.url=ldap://ldap.example.com:389
mms.ldap.bindDn=cn=opsmgr,ou=services,dc=example,dc=com
mms.ldap.bindPassword=ldapPassword

# User mapping
mms.ldap.userSearchBase=ou=users,dc=example,dc=com
mms.ldap.userSearchFilter=(&(objectClass=person)(uid={0}))

# Group mapping
mms.ldap.groupSearchBase=ou=groups,dc=example,dc=com
mms.ldap.groupSearchFilter=(member={0})
mms.ldap.groupRoleMapping=cn=DBAs,ou=groups,dc=example,dc=com:PROJECT_OWNER

# TLS
mms.ldap.tls=true
mms.ldap.trustStorePath=/opt/mongodb/mms/conf/truststore.jks
```

#### Role-Based Access Control

```
Organization → Access Manager

Roles disponibles:
├─ Organization Owner
│  └─ Full control (billing, users, all projects)
│
├─ Organization Member
│  └─ Access to assigned projects only
│
├─ Project Owner
│  └─ Full control within project
│
├─ Project Cluster Manager
│  └─ Manage deployments, no backup access
│
├─ Project Data Access Admin
│  └─ Create DB users, no cluster management
│
├─ Project Read Only
│  └─ View only, no modifications
│
└─ Project Backup Admin
   └─ Manage backups only
```

**Mapping d'équipe** :

```
Team: Database Administrators
├─ Users: alice@example.com, bob@example.com
├─ Projects: All
└─ Role: Project Owner

Team: Application Developers
├─ Users: dev-team@example.com (LDAP group)
├─ Projects: Development, Staging
└─ Role: Project Data Access Admin

Team: Operations
├─ Users: ops-team@example.com
├─ Projects: All
└─ Role: Project Cluster Manager

Team: Auditors
├─ Users: audit@example.com
├─ Projects: All
└─ Role: Project Read Only
```

### Audit Logging

```properties
# Enable audit logging
mms.audit.enabled=true
mms.audit.destination=file
mms.audit.filePath=/opt/mongodb/mms/logs/audit.log
mms.audit.format=JSON

# Audit filters
mms.audit.filter.authentication=true
mms.audit.filter.authorization=true
mms.audit.filter.schema=true
mms.audit.filter.backup=true
```

**Sample audit entry** :

```json
{
  "timestamp": "2024-12-10T15:30:45.123Z",
  "atype": "authCheck",
  "result": "success",
  "user": "alice@example.com",
  "roles": ["PROJECT_OWNER"],
  "param": {
    "resource": {
      "projectId": "5f2c8b1234567890abcdef12",
      "resourceType": "CLUSTER"
    },
    "action": "MODIFY_SETTINGS"
  },
  "remote": {
    "ip": "203.0.113.45",
    "userAgent": "Mozilla/5.0..."
  }
}
```

### TLS/SSL Configuration

```properties
# conf-mms.properties

# HTTPS for web interface
mms.https.enabled=true
mms.https.port=8443
mms.https.PEMKeyFile=/opt/mongodb/mms/conf/certs/opsmgr.pem
mms.https.ClientCertificateMode=optional

# Agent communication
mms.agentTLS.enabled=true
mms.agentTLS.ClientCertificateMode=required
mms.agentTLS.CAFile=/opt/mongodb/mms/conf/certs/ca.pem
```

**Certificate rotation automatisée** :

```bash
#!/bin/bash
# rotate_certs.sh

NEW_CERT="/path/to/new/cert.pem"
OPSMGR_CONF="/opt/mongodb/mms/conf/conf-mms.properties"

# Update configuration
sed -i "s|mms.https.PEMKeyFile=.*|mms.https.PEMKeyFile=$NEW_CERT|" $OPSMGR_CONF

# Restart Ops Manager (agents reconnect automatically)
systemctl restart mongodb-mms

# Verify
curl -k https://opsmgr.example.com:8443/api/public/v1.0/unauth/version
```

---

## API et Intégration

### REST API Overview

Ops Manager expose une **REST API complète** pour toutes les opérations.

**Base URL** : `https://opsmgr.example.com:8080/api/public/v1.0`

**Authentication** :
```bash
# Digest Authentication
PUBLIC_KEY="your-public-key"
PRIVATE_KEY="your-private-key"

# All requests use digest auth
curl -u "${PUBLIC_KEY}:${PRIVATE_KEY}" --digest \
  https://opsmgr.example.com:8080/api/public/v1.0/...
```

### Exemples d'Intégration

#### 1. Monitoring Integration (Prometheus Exporter)

```python
#!/usr/bin/env python3
# opsmgr_exporter.py

import requests
from requests.auth import HTTPDigestAuth
from prometheus_client import start_http_server, Gauge
import time

class OpsManagerExporter:
    def __init__(self, base_url, public_key, private_key, project_id):
        self.base_url = base_url
        self.auth = HTTPDigestAuth(public_key, private_key)
        self.project_id = project_id

        # Define Prometheus metrics
        self.metrics = {
            'ops': Gauge('opsmgr_operations_per_second', 'Operations per second', ['host']),
            'connections': Gauge('opsmgr_connections_current', 'Current connections', ['host']),
            'replication_lag': Gauge('opsmgr_replication_lag_seconds', 'Replication lag', ['host']),
            'memory': Gauge('opsmgr_memory_resident_mb', 'Resident memory MB', ['host'])
        }

    def get_processes(self):
        """Get all processes in project"""
        url = f"{self.base_url}/groups/{self.project_id}/processes"
        response = requests.get(url, auth=self.auth)
        return response.json()['results']

    def get_measurements(self, host, port, metrics):
        """Get measurements for specific host"""
        url = f"{self.base_url}/groups/{self.project_id}/hosts/{host}:{port}/measurements"
        params = {
            'granularity': 'PT1M',
            'period': 'PT1M',
            'metrics': ','.join(metrics)
        }
        response = requests.get(url, auth=self.auth, params=params)
        return response.json()

    def collect(self):
        """Collect metrics from Ops Manager"""
        processes = self.get_processes()

        for process in processes:
            if process['typeName'] != 'REPLICA_PRIMARY':
                continue

            host = process['hostname']
            port = process['port']
            host_label = f"{host}:{port}"

            # Get operations/sec
            data = self.get_measurements(host, port, ['OPCOUNTER_QUERY', 'OPCOUNTER_INSERT'])

            for measurement in data.get('measurements', []):
                if measurement['name'] == 'OPCOUNTER_QUERY':
                    value = measurement['dataPoints'][-1]['value'] if measurement['dataPoints'] else 0
                    self.metrics['ops'].labels(host=host_label).set(value)

    def run(self):
        """Main loop"""
        while True:
            try:
                self.collect()
            except Exception as e:
                print(f"Error collecting metrics: {e}")
            time.sleep(60)

if __name__ == '__main__':
    exporter = OpsManagerExporter(
        base_url='https://opsmgr.example.com:8080/api/public/v1.0',
        public_key='PUBLIC_KEY',
        private_key='PRIVATE_KEY',
        project_id='PROJECT_ID'
    )

    start_http_server(9217)
    print("Exporter started on port 9217")
    exporter.run()
```

#### 2. Automated Deployment Pipeline

```python
#!/usr/bin/env python3
# deploy_mongodb_cluster.py

import requests
from requests.auth import HTTPDigestAuth
import time
import json

class MongoDBDeployer:
    def __init__(self, ops_manager_url, public_key, private_key, project_id):
        self.base_url = f"{ops_manager_url}/api/public/v1.0"
        self.auth = HTTPDigestAuth(public_key, private_key)
        self.project_id = project_id

    def deploy_replica_set(self, config):
        """Deploy replica set via automation"""

        # Get current automation config
        url = f"{self.base_url}/groups/{self.project_id}/automationConfig"
        current_config = requests.get(url, auth=self.auth).json()

        # Add new replica set
        current_config['processes'].extend(config['processes'])
        current_config['replicaSets'].append(config['replicaSet'])

        # Apply new config
        response = requests.put(
            url,
            auth=self.auth,
            headers={'Content-Type': 'application/json'},
            data=json.dumps(current_config)
        )

        if response.status_code != 200:
            raise Exception(f"Deployment failed: {response.text}")

        # Wait for goal state
        return self.wait_for_goal_state()

    def wait_for_goal_state(self, timeout=1800):
        """Wait for automation to reach goal state"""
        url = f"{self.base_url}/groups/{self.project_id}/automationStatus"
        start_time = time.time()

        while time.time() - start_time < timeout:
            status = requests.get(url, auth=self.auth).json()

            if status['goalVersion'] == status['lastGoalVersionAchieved']:
                print("Goal state reached!")
                return True

            print(f"Waiting... Goal: {status['goalVersion']}, Current: {status['lastGoalVersionAchieved']}")
            time.sleep(30)

        raise TimeoutError("Goal state not reached within timeout")

# Usage
deployer = MongoDBDeployer(
    'https://opsmgr.example.com:8080',
    'PUBLIC_KEY',
    'PRIVATE_KEY',
    'PROJECT_ID'
)

replica_set_config = {
    'processes': [
        {
            'hostname': 'new-mongo-01',
            'port': 27017,
            'processType': 'mongod',
            'version': '7.0.4',
            'args2_6': {
                'replication': {'replSetName': 'new-rs0'},
                'storage': {'dbPath': '/data/mongodb'}
            }
        },
        # ... additional members
    ],
    'replicaSet': {
        'id': 'new-rs0',
        'members': [
            {'id': 0, 'hostname': 'new-mongo-01', 'port': 27017, 'priority': 1}
            # ... additional members
        ]
    }
}

deployer.deploy_replica_set(replica_set_config)
```

#### 3. Alert Webhook Handler

```python
#!/usr/bin/env python3
# webhook_handler.py

from flask import Flask, request, jsonify
import requests

app = Flask(__name__)

@app.route('/webhook/opsmgr', methods=['POST'])
def handle_opsmgr_webhook():
    """Handle Ops Manager webhook alerts"""

    alert = request.json

    # Parse alert
    alert_type = alert.get('typeName')
    severity = alert.get('status')
    group = alert.get('groupName')
    host = alert.get('hostId')

    # Custom logic based on alert type
    if alert_type == 'HOST_DOWN':
        handle_host_down(alert)
    elif alert_type == 'REPLICATION_LAG':
        handle_replication_lag(alert)
    elif alert_type == 'BACKUP_FAILED':
        handle_backup_failure(alert)

    return jsonify({'status': 'processed'}), 200

def handle_host_down(alert):
    """Auto-remediation for host down"""

    # Send to PagerDuty with high severity
    requests.post('https://events.pagerduty.com/v2/enqueue', json={
        'routing_key': 'YOUR_ROUTING_KEY',
        'event_action': 'trigger',
        'payload': {
            'summary': f"MongoDB host down: {alert['hostId']}",
            'severity': 'critical',
            'source': 'ops-manager'
        }
    })

    # Create JIRA ticket
    requests.post('https://jira.example.com/rest/api/2/issue', auth=('user', 'pass'), json={
        'fields': {
            'project': {'key': 'INFRA'},
            'summary': f"MongoDB host down: {alert['hostId']}",
            'issuetype': {'name': 'Incident'},
            'priority': {'name': 'Critical'}
        }
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

---

## Cas d'Usage Avancés

### 1. Multi-Datacenter Deployment

Configuration active-passive multi-DC :

```
Organization: Global Corp
Project: Production MongoDB

Datacenter 1 (US-East - Active):
├─ Replica Set: prod-us-rs0
│  ├─ Primary: us-mongo-01 (Priority 2)
│  ├─ Secondary: us-mongo-02 (Priority 1)
│  └─ Secondary: us-mongo-03 (Priority 1)
│
└─ Backup Daemon: us-backup-01

Datacenter 2 (EU-West - Passive/DR):
├─ Replica Set Members:
│  ├─ Secondary: eu-mongo-01 (Priority 0, Hidden)
│  └─ Secondary: eu-mongo-02 (Priority 0, Hidden)
│
└─ Backup Daemon: eu-backup-01

Configuration Ops Manager:
- Monitoring: All members from single Ops Manager
- Alerts: Datacenter-aware (suppress DR site during planned failover)
- Automation: Coordinated configuration changes
- Backup: Dual backup daemons for redundancy
```

**Failover automatisé** :

```python
def failover_to_dr():
    """Promote DR site to active"""

    # 1. Update member priorities
    config = get_automation_config()

    # Reduce US members priority
    for member in config['replicaSets'][0]['members']:
        if member['hostname'].startswith('us-'):
            member['priority'] = 0

    # Increase EU members priority
    for member in config['replicaSets'][0]['members']:
        if member['hostname'].startswith('eu-'):
            member['priority'] = 1
            member['hidden'] = False

    # Apply configuration
    apply_automation_config(config)

    # 2. Wait for election
    wait_for_goal_state()

    # 3. Update DNS
    update_dns_to_eu()

    # 4. Send notifications
    notify_team("Failover to EU completed")
```

### 2. Capacity Planning Automation

```python
#!/usr/bin/env python3
# capacity_planner.py

import requests
from datetime import datetime, timedelta
from requests.auth import HTTPDigestAuth
import numpy as np

class CapacityPlanner:
    def __init__(self, ops_manager_url, public_key, private_key, project_id):
        self.base_url = f"{ops_manager_url}/api/public/v1.0"
        self.auth = HTTPDigestAuth(public_key, private_key)
        self.project_id = project_id

    def get_historical_metrics(self, host, port, metric, days=30):
        """Get historical data"""
        url = f"{self.base_url}/groups/{self.project_id}/hosts/{host}:{port}/measurements"

        end = datetime.utcnow()
        start = end - timedelta(days=days)

        params = {
            'granularity': 'PT1H',
            'period': f'P{days}D',
            'metrics': metric,
            'start': start.isoformat() + 'Z',
            'end': end.isoformat() + 'Z'
        }

        response = requests.get(url, auth=self.auth, params=params)
        data = response.json()

        if not data.get('measurements'):
            return []

        return [dp['value'] for dp in data['measurements'][0]['dataPoints'] if dp['value'] is not None]

    def predict_capacity(self, values, days_ahead=30):
        """Linear regression prediction"""
        if len(values) < 10:
            return None

        x = np.arange(len(values))
        y = np.array(values)

        # Linear regression
        coeffs = np.polyfit(x, y, 1)
        slope, intercept = coeffs

        # Predict future
        future_x = len(values) + (days_ahead * 24)  # hourly data
        prediction = slope * future_x + intercept

        return {
            'current': values[-1],
            'prediction': prediction,
            'growth_rate_per_day': slope * 24,
            'days_to_threshold': None
        }

    def analyze_project(self):
        """Analyze all clusters in project"""

        # Get all processes
        url = f"{self.base_url}/groups/{self.project_id}/processes"
        processes = requests.get(url, auth=self.auth).json()['results']

        report = []

        for process in processes:
            if process['typeName'] != 'REPLICA_PRIMARY':
                continue

            host = process['hostname']
            port = process['port']

            # Analyze storage
            storage_data = self.get_historical_metrics(host, port, 'DATABASE_STORAGE_SIZE', 30)
            storage_prediction = self.predict_capacity(storage_data)

            # Analyze memory
            memory_data = self.get_historical_metrics(host, port, 'SYSTEM_MEMORY_USED', 30)
            memory_prediction = self.predict_capacity(memory_data)

            report.append({
                'host': f"{host}:{port}",
                'storage': storage_prediction,
                'memory': memory_prediction
            })

        return report

# Usage
planner = CapacityPlanner(
    'https://opsmgr.example.com:8080',
    'PUBLIC_KEY',
    'PRIVATE_KEY',
    'PROJECT_ID'
)

report = planner.analyze_project()
for item in report:
    print(f"\nHost: {item['host']}")
    print(f"Storage: {item['storage']['current']:.2f} GB")
    print(f"Predicted (30d): {item['storage']['prediction']:.2f} GB")
    print(f"Growth rate: {item['storage']['growth_rate_per_day']:.2f} GB/day")
```

### 3. Compliance et Audit Reporting

```python
#!/usr/bin/env python3
# compliance_reporter.py

import requests
from requests.auth import HTTPDigestAuth
from datetime import datetime, timedelta
import csv

class ComplianceReporter:
    def __init__(self, ops_manager_url, public_key, private_key, project_id):
        self.base_url = f"{ops_manager_url}/api/public/v1.0"
        self.auth = HTTPDigestAuth(public_key, private_key)
        self.project_id = project_id

    def generate_backup_report(self, days=30):
        """Verify all clusters have successful backups"""

        url = f"{self.base_url}/groups/{self.project_id}/clusters"
        clusters = requests.get(url, auth=self.auth).json()['results']

        report = []
        end_date = datetime.utcnow()
        start_date = end_date - timedelta(days=days)

        for cluster in clusters:
            cluster_id = cluster['id']

            # Get snapshots
            snapshots_url = f"{self.base_url}/groups/{self.project_id}/clusters/{cluster_id}/snapshots"
            params = {
                'minDate': start_date.isoformat() + 'Z',
                'maxDate': end_date.isoformat() + 'Z'
            }
            snapshots = requests.get(snapshots_url, auth=self.auth, params=params).json()

            successful_snapshots = [s for s in snapshots.get('results', []) if s['status'] == 'COMPLETED']
            failed_snapshots = [s for s in snapshots.get('results', []) if s['status'] == 'FAILED']

            report.append({
                'cluster_name': cluster['clusterName'],
                'total_snapshots': len(snapshots.get('results', [])),
                'successful': len(successful_snapshots),
                'failed': len(failed_snapshots),
                'compliance': len(successful_snapshots) >= days,  # At least 1 per day
                'last_successful': successful_snapshots[0]['created'] if successful_snapshots else 'N/A'
            })

        return report

    def generate_security_report(self):
        """Audit security configuration"""

        url = f"{self.base_url}/groups/{self.project_id}/automationConfig"
        config = requests.get(url, auth=self.auth).json()

        report = {
            'tls_enabled': False,
            'authentication_enabled': False,
            'authorization_enabled': False,
            'encryption_at_rest': False,
            'audit_logging': False
        }

        # Check TLS
        for process in config.get('processes', []):
            args = process.get('args2_6', {})
            if args.get('net', {}).get('tls', {}).get('mode') in ['requireTLS', 'preferTLS']:
                report['tls_enabled'] = True
            if args.get('security', {}).get('authorization') == 'enabled':
                report['authorization_enabled'] = True
            if args.get('auditLog', {}).get('destination'):
                report['audit_logging'] = True

        return report

    def export_to_csv(self, backup_report, security_report, filename):
        """Export compliance report to CSV"""

        with open(filename, 'w', newline='') as csvfile:
            writer = csv.writer(csvfile)

            # Backup compliance
            writer.writerow(['Backup Compliance Report'])
            writer.writerow(['Cluster', 'Total Snapshots', 'Successful', 'Failed', 'Compliant', 'Last Success'])

            for item in backup_report:
                writer.writerow([
                    item['cluster_name'],
                    item['total_snapshots'],
                    item['successful'],
                    item['failed'],
                    'YES' if item['compliance'] else 'NO',
                    item['last_successful']
                ])

            writer.writerow([])
            writer.writerow(['Security Configuration'])
            writer.writerow(['Setting', 'Status'])

            for key, value in security_report.items():
                writer.writerow([key, 'ENABLED' if value else 'DISABLED'])

# Usage
reporter = ComplianceReporter(
    'https://opsmgr.example.com:8080',
    'PUBLIC_KEY',
    'PRIVATE_KEY',
    'PROJECT_ID'
)

backup_report = reporter.generate_backup_report(30)
security_report = reporter.generate_security_report()
reporter.export_to_csv(backup_report, security_report, 'compliance_report.csv')
```

---

## Troubleshooting

### Problème 1 : Agents Non Connectés

**Symptômes** :
```
UI: Agent status "Not Connected" ou "Unavailable"
Monitoring data missing
Automation changes not applied
```

**Diagnostic** :
```bash
# 1. Vérifier le statut du service
sudo systemctl status mongodb-mms-monitoring-agent
sudo systemctl status mongodb-mms-automation-agent

# 2. Vérifier les logs
tail -f /var/log/mongodb-mms/monitoring-agent.log
tail -f /var/log/mongodb-mms/automation-agent.log

# Rechercher des erreurs
grep -i error /var/log/mongodb-mms/*-agent.log

# 3. Vérifier la connectivité réseau
curl -v https://opsmgr.example.com:8080/api/public/v1.0/unauth/version

# 4. Vérifier la configuration
cat /etc/mongodb-mms/monitoring-agent.config | grep -E "mmsBaseUrl|mmsGroupId|mmsApiKey"

# 5. Tester l'authentification API
curl -u "PUBLIC_KEY:PRIVATE_KEY" --digest \
  https://opsmgr.example.com:8080/api/public/v1.0/groups/PROJECT_ID
```

**Solutions** :
```bash
# 1. Vérifier mmsApiKey valide
# Regenerate in UI: Project Settings → Agents → Agent API Key

# 2. Vérifier les permissions firewall
sudo iptables -L -n | grep 8080
sudo firewall-cmd --list-all

# 3. Vérifier les certificats TLS
openssl s_client -connect opsmgr.example.com:8080 -showcerts

# 4. Restart agents
sudo systemctl restart mongodb-mms-monitoring-agent
sudo systemctl restart mongodb-mms-automation-agent
```

### Problème 2 : Automation Stuck "In Progress"

**Symptômes** :
```
UI: Deployment shows "Changes being deployed" indefinitely
Goal state never reached
```

**Diagnostic** :
```bash
# Via API
curl -u "PUBLIC_KEY:PRIVATE_KEY" --digest \
  "https://opsmgr.example.com:8080/api/public/v1.0/groups/PROJECT_ID/automationStatus" | jq .

# Output
{
  "goalVersion": 42,
  "lastGoalVersionAchieved": 40
}

# Check agent logs for errors
grep -A 5 -i "error\|exception" /var/log/mongodb-mms/automation-agent.log

# Check if processes are running
ps aux | grep mongod
```

**Solutions** :
```bash
# 1. Vérifier l'état des processus MongoDB
mongo --eval "db.adminCommand({ serverStatus: 1 }).process"

# 2. Vérifier les permissions filesystem
ls -la /data/mongodb
sudo chown -R mongodb:mongodb /data/mongodb

# 3. Vérifier le disk space
df -h /data/mongodb

# 4. En dernier recours : réinitialiser l'automation agent
sudo systemctl stop mongodb-mms-automation-agent
sudo rm -rf /var/lib/mongodb-mms-automation/automation-agent.status
sudo systemctl start mongodb-mms-automation-agent
```

### Problème 3 : Backup Failures

**Symptômes** :
```
Alert: "Backup snapshot failed"
Missing snapshots in UI
```

**Diagnostic** :
```bash
# Vérifier le statut du backup daemon
sudo systemctl status mongodb-mms-backup-daemon

# Logs
tail -f /var/log/mongodb-mms/backup-daemon.log

# Vérifier l'espace de stockage backup
df -h /data/backup  # ou bucket S3

# Vérifier la connectivité au replica set
mongo "mongodb://backup-user:pass@rs0-primary:27017,rs0-sec1:27017,rs0-sec2:27017/?replicaSet=rs0&readPreference=secondary"

# Vérifier l'oplog
db.getReplicationInfo()
```

**Solutions** :
```bash
# 1. Augmenter l'espace disque si plein

# 2. Vérifier les credentials backup user
mongo --eval "
  use admin
  db.getUser('backup-user')
"

# 3. Vérifier les permissions S3 (si applicable)
aws s3 ls s3://mongodb-backups-prod/ --profile backup

# 4. Restart backup daemon
sudo systemctl restart mongodb-mms-backup-daemon

# 5. Re-sync si corruption
# Via UI: Backup → Settings → Resync Backup
```

---

## Bonnes Pratiques

### 1. Architecture et Dimensionnement

```yaml
Ops Manager Application:
  Déploiement:
    ✓ HA avec load balancer (2+ instances)
    ✓ Application database en replica set (3+ membres)
    ✓ SSD pour application database
    ✓ Backup automatisé de l'application database

  Monitoring:
    ✓ Monitoring de l'Ops Manager lui-même (meta-monitoring)
    ✓ Alertes sur la santé de l'application
    ✓ Logs centralisés (Splunk/ELK)

Agents:
  ✓ Un automation agent par serveur physique/VM
  ✓ Un monitoring agent par processus MongoDB
  ✓ Backup daemon co-localisé ou dédié selon charge
  ✓ Agents sur version latest (auto-update recommandé)

Network:
  ✓ Faible latence entre Ops Manager et agents (< 100ms)
  ✓ Bandwidth suffisant pour metrics (~ 1 Mbps par 100 agents)
  ✓ Firewall rules pour ports : 8080 (HTTPS), 27017+ (MongoDB)
```

### 2. Sécurité

```yaml
Authentication:
  ✓ LDAP/AD integration pour SSO
  ✓ MFA obligatoire pour admin users
  ✓ API keys rotation tous les 90 jours
  ✓ Audit logging activé

Authorization:
  ✓ Principe du moindre privilège
  ✓ Séparation des rôles (cluster mgmt vs backup)
  ✓ Review trimestriel des accès

Encryption:
  ✓ TLS pour UI et agents
  ✓ Encrypted at rest pour backup storage
  ✓ Secrets management (Vault) pour credentials

Compliance:
  ✓ Audit logs exportés vers SIEM
  ✓ Backup retention conforme aux régulations
  ✓ Regular security scans
```

### 3. Opérations

```yaml
Monitoring:
  ✓ Baseline performance établi
  ✓ Alertes configurées pour tous clusters critiques
  ✓ Dashboard review hebdomadaire
  ✓ Performance advisor consulté régulièrement

Automation:
  ✓ Infrastructure as Code pour configurations
  ✓ Change control process pour modifications
  ✓ Maintenance windows planifiées
  ✓ Rollback plan documenté

Backup:
  ✓ Restore testing mensuel
  ✓ Point-in-time recovery validé
  ✓ Backup storage redundancy (multi-région)
  ✓ Retention policy conforme business needs

Documentation:
  ✓ Runbooks à jour pour alertes communes
  ✓ Architecture diagrams actualisés
  ✓ On-call escalation path défini
  ✓ Disaster recovery plan testé annuellement
```

### 4. Checklist Déploiement Production

```markdown
Phase 1: Préparation
□ Sizing calculé (application, database, agents)
□ Infrastructure provisionnée
□ Network configuration validée
□ Firewall rules appliquées
□ DNS entries créés
□ TLS certificates obtenus

Phase 2: Installation
□ Ops Manager application installé
□ Application database configuré en HA
□ Initial admin user créé
□ Organization/Project structure définie
□ LDAP integration configurée (si applicable)

Phase 3: Configuration
□ Agents déployés sur tous serveurs MongoDB
□ Tous clusters discovered et managed
□ Monitoring alerting configuré
□ Backup activé pour clusters production
□ API access configuré pour automation

Phase 4: Validation
□ Monitoring data visible pour tous clusters
□ Test alert envoyé et reçu
□ Test backup et restore effectué
□ Test automation change (config update)
□ Performance baseline établi

Phase 5: Documentation
□ Runbooks créés
□ Architecture documentée
□ Access list (qui a quelles permissions)
□ Escalation procedures
□ Disaster recovery plan

Phase 6: Go-Live
□ Communication équipe
□ Monitoring 24/7 pendant première semaine
□ Post-implementation review après 2 semaines
```

---

## Résumé pour SRE

### Avantages vs Autres Solutions

| Aspect | Ops Manager | Prometheus/Grafana | MongoDB Atlas |
|--------|-------------|-------------------|---------------|
| **Setup complexity** | Élevé | Modéré | Minimal |
| **Automation** | Complet (deploy, upgrade) | Aucune | Complet |
| **Backup** | Intégré (PITR) | Externe | Intégré |
| **On-premise** | ✓ | ✓ | ✗ |
| **Licensing** | Enterprise | Open-source | SaaS pricing |
| **Best for** | Enterprise on-prem | Cloud-native, multi-vendor | Cloud-first |

### Commandes Essentielles API

```bash
# List all clusters
curl -u "$KEY:$SECRET" --digest \
  "https://opsmgr/api/public/v1.0/groups/$PROJECT/clusters" | jq .

# Get automation status
curl -u "$KEY:$SECRET" --digest \
  "https://opsmgr/api/public/v1.0/groups/$PROJECT/automationStatus" | jq .

# List alerts
curl -u "$KEY:$SECRET" --digest \
  "https://opsmgr/api/public/v1.0/groups/$PROJECT/alerts" | jq .

# Get measurements
curl -u "$KEY:$SECRET" --digest \
  "https://opsmgr/api/public/v1.0/groups/$PROJECT/hosts/$HOST:$PORT/measurements?metrics=OPCOUNTER_QUERY&granularity=PT1M" | jq .

# Trigger backup snapshot
curl -X POST -u "$KEY:$SECRET" --digest \
  "https://opsmgr/api/public/v1.0/groups/$PROJECT/clusters/$CLUSTER/snapshots" \
  -H "Content-Type: application/json" \
  -d '{"description":"Manual snapshot"}'
```

### Métriques Critiques Top 10

```
1. Automation Goal State: goalVersion == lastGoalVersionAchieved
2. Agent Connectivity: all agents "connected"
3. Replication Lag: < 10 seconds
4. Backup Success Rate: > 99% snapshots successful
5. Oplog Window: > 24 hours
6. Operations/sec: baseline + 3σ
7. Connections: < 80% max
8. Cache Usage: 60-80% used, < 20% dirty
9. Disk Space: > 20% free
10. Alert Response Time: < 15 minutes moyenne
```

---

## Conclusion

MongoDB Ops Manager est la solution **enterprise-grade** pour gérer des déploiements MongoDB à grande échelle dans des environnements on-premise ou air-gapped. Sa combinaison unique de **monitoring**, **automation** et **backup** en fait un choix idéal pour :

**Cas d'usage idéaux** :
- Secteurs régulés (finance, santé, gouvernement)
- Exigences de data residency strictes
- Air-gapped environments
- Besoins de contrôle total sur l'infrastructure
- Déploiements multi-datacenters complexes

**Points clés à retenir** :
- Investissement initial élevé mais ROI fort à grande échelle
- Automatisation réduit les erreurs humaines
- Backup intégré simplifie la compliance
- Courbe d'apprentissage modérée
- Support MongoDB Enterprise inclus

**Prochaines étapes** :
- Évaluer le sizing pour votre environnement
- PoC sur environnement non-production
- Former l'équipe aux concepts d'automation
- Établir les runbooks et procédures
- Planifier migration progressive des clusters

---

**Références** :
- [Ops Manager Documentation](https://www.mongodb.com/docs/ops-manager/)
- [Ops Manager API Reference](https://www.mongodb.com/docs/ops-manager/current/reference/api/)
- [Ops Manager Sizing Calculator](https://www.mongodb.com/docs/ops-manager/current/installation/#prerequisites)
- [MongoDB Enterprise Advanced](https://www.mongodb.com/products/mongodb-enterprise-advanced)

⏭️ [Alerting et notifications](/13-monitoring-administration/09-alerting-notifications.md)
