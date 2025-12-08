🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.7 Intégration avec Prometheus et Grafana

## Introduction

L'intégration de MongoDB avec **Prometheus** et **Grafana** constitue la solution de monitoring moderne de référence pour les environnements de production. Cette stack offre aux SRE et administrateurs système une visibilité complète, des alertes intelligentes et des capacités d'analyse avancées essentielles pour maintenir la santé et les performances des clusters MongoDB.

**Prometheus** assure la collecte et le stockage des métriques time-series, tandis que **Grafana** fournit les dashboards de visualisation et l'interface d'alerting. Cette combinaison permet :

- **Monitoring continu** avec historique long-terme (contrairement à mongostat)
- **Alerting intelligent** basé sur des patterns et seuils multiples
- **Visualisation riche** avec corrélation de métriques
- **Analyse de tendances** pour la planification de capacité
- **Débogage efficace** via l'exploration interactive des données
- **Conformité** aux standards observability modernes

Dans un environnement Kubernetes ou cloud-native, cette stack s'intègre naturellement avec l'écosystème existant (service discovery, configuration as code, HA).

---

## Architecture d'Intégration

### Vue d'Ensemble

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Architecture de Monitoring                      │
└─────────────────────────────────────────────────────────────────────┘

┌──────────────────────┐         ┌──────────────────────┐
│   MongoDB Cluster    │         │   MongoDB Cluster    │
│   ┌──────────────┐   │         │   ┌──────────────┐   │
│   │  Primary     │   │         │   │  Primary     │   │
│   │  :27017      │   │         │   │  :27017      │   │
│   └──────┬───────┘   │         │   └──────┬───────┘   │
│          │           │         │          │           │
│   ┌──────▼───────┐   │         │   ┌──────▼───────┐   │
│   │ Secondary 1  │   │         │   │ Secondary 1  │   │
│   └──────────────┘   │         │   └──────────────┘   │
│   ┌──────────────┐   │         │   ┌──────────────┐   │
│   │ Secondary 2  │   │         │   │ Secondary 2  │   │
│   └──────────────┘   │         │   └──────────────┘   │
└──────────┬───────────┘         └──────────┬───────────┘
           │                                │
           │ metrics :9216                  │ metrics :9216
           │                                │
           ▼                                ▼
┌──────────────────┐              ┌──────────────────┐
│ MongoDB Exporter │              │ MongoDB Exporter │
│   (per cluster)  │              │   (per cluster)  │
│     :9216        │              │     :9216        │
└──────────┬───────┘              └──────────┬───────┘
           │                                 │
           │ /metrics (HTTP)                 │
           │                                 │
           └────────────┬────────────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │    Prometheus       │
              │  Time-Series DB     │
              │  + Alertmanager     │
              │      :9090          │
              └─────────┬───────────┘
                        │
                        │ PromQL queries
                        │
                        ▼
              ┌─────────────────────┐
              │      Grafana        │
              │   Visualization     │
              │      :3000          │
              └─────────────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │  Alerting Channels  │
              │ (Slack, PagerDuty,  │
              │   Email, etc.)      │
              └─────────────────────┘
```

### Composants Clés

#### 1. MongoDB Exporter

**Rôle** : Collecte les métriques depuis MongoDB et les expose au format Prometheus.

**Métriques collectées** :
- serverStatus : État global du serveur
- replSetGetStatus : État du replica set
- dbStats : Statistiques par base de données
- collStats : Statistiques par collection (optionnel)
- indexStats : Utilisation des index (optionnel)

**Versions disponibles** :
- **percona/mongodb_exporter** (recommandé) : Plus maintenu, plus de métriques
- **prometheus-community/mongodb_exporter** : Version communautaire officielle

#### 2. Prometheus

**Rôle** :
- Scraping périodique des exporters
- Stockage time-series
- Évaluation des règles d'alerte
- API de requête (PromQL)

**Caractéristiques** :
- Pull model (Prometheus scrape les targets)
- Retention configurable (15 jours par défaut)
- Service discovery (static, Kubernetes, Consul, etc.)

#### 3. Grafana

**Rôle** :
- Visualisation des métriques
- Dashboards interactifs
- Alerting (complémentaire à Alertmanager)
- Annotations et corrélations

---

## Installation et Configuration

### 1. MongoDB Exporter (Percona)

#### Installation via Binaire

```bash
# Téléchargement
VERSION="0.40.0"
wget https://github.com/percona/mongodb_exporter/releases/download/v${VERSION}/mongodb_exporter-${VERSION}.linux-amd64.tar.gz

# Extraction
tar -xzf mongodb_exporter-${VERSION}.linux-amd64.tar.gz
sudo mv mongodb_exporter /usr/local/bin/

# Vérification
mongodb_exporter --version
```

#### Configuration Utilisateur MongoDB

Créer un utilisateur dédié avec permissions minimales :

```javascript
// Se connecter à MongoDB
use admin

// Créer l'utilisateur monitoring
db.createUser({
  user: "mongodb_exporter",
  pwd: "strongPassword123!",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})

// Vérification
db.auth("mongodb_exporter", "strongPassword123!")
db.runCommand({ serverStatus: 1 })
```

**Permissions détaillées** :
- `clusterMonitor` : Lecture de serverStatus, replSetGetStatus, top
- `read` sur `local` : Accès à l'oplog pour les métriques de réplication

#### Service Systemd

```ini
# /etc/systemd/system/mongodb_exporter.service
[Unit]
Description=MongoDB Exporter
Documentation=https://github.com/percona/mongodb_exporter
After=network.target

[Service]
Type=simple
User=prometheus
Group=prometheus
Environment="MONGODB_URI=mongodb://mongodb_exporter:strongPassword123!@localhost:27017/?authSource=admin"
ExecStart=/usr/local/bin/mongodb_exporter \
  --mongodb.uri=${MONGODB_URI} \
  --web.listen-address=:9216 \
  --collect-all \
  --discovering-mode \
  --mongodb.collstats-colls=mydb.orders,mydb.users \
  --log.level=info

# Security
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=strict
ReadWritePaths=/var/log/mongodb_exporter

# Restart policy
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Options importantes** :

| Option | Description | Valeur Recommandée |
|--------|-------------|-------------------|
| `--mongodb.uri` | URI de connexion | Variable d'environnement |
| `--web.listen-address` | Port d'écoute | :9216 (standard) |
| `--collect-all` | Toutes les métriques | Activé |
| `--discovering-mode` | Auto-découverte replica set | Activé pour RS |
| `--mongodb.collstats-colls` | Collections à monitorer | Collections critiques uniquement |
| `--mongodb.indexstats-colls` | Index à monitorer | Collections critiques uniquement |
| `--compatible-mode` | Mode compatibilité anciennes versions | Si MongoDB < 4.0 |

**Activation du service** :

```bash
# Créer l'utilisateur système
sudo useradd -rs /bin/false prometheus

# Activer et démarrer
sudo systemctl daemon-reload
sudo systemctl enable mongodb_exporter
sudo systemctl start mongodb_exporter

# Vérifier le statut
sudo systemctl status mongodb_exporter

# Vérifier les métriques
curl http://localhost:9216/metrics | head -20
```

#### Configuration pour Replica Set

Pour un replica set, déployer un exporter par membre **ou** utiliser le mode discovery :

**Option 1 : Exporter par membre** (recommandé pour granularité)
```bash
# mongodb-rs0-primary
MONGODB_URI="mongodb://mongodb_exporter:pass@localhost:27017/?authSource=admin&replicaSet=rs0"

# mongodb-rs0-secondary1
MONGODB_URI="mongodb://mongodb_exporter:pass@localhost:27017/?authSource=admin&replicaSet=rs0"

# mongodb-rs0-secondary2
MONGODB_URI="mongodb://mongodb_exporter:pass@localhost:27017/?authSource=admin&replicaSet=rs0"
```

**Option 2 : Mode Discovery** (un exporter découvre tous les membres)
```bash
MONGODB_URI="mongodb://mongodb_exporter:pass@rs0-primary:27017,rs0-sec1:27017,rs0-sec2:27017/?authSource=admin&replicaSet=rs0"
mongodb_exporter --mongodb.uri="${MONGODB_URI}" --discovering-mode
```

#### Configuration Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0
    container_name: mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: password
    volumes:
      - mongodb_data:/data/db
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    command: mongod --replSet rs0

  mongodb-exporter:
    image: percona/mongodb_exporter:0.40
    container_name: mongodb_exporter
    ports:
      - "9216:9216"
    environment:
      MONGODB_URI: "mongodb://mongodb_exporter:exporterPass@mongodb:27017/?authSource=admin"
    command:
      - '--collect-all'
      - '--compatible-mode'
    depends_on:
      - mongodb

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro

volumes:
  mongodb_data:
  prometheus_data:
  grafana_data:
```

### 2. Configuration Prometheus

#### Fichier de Configuration

```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
  external_labels:
    cluster: 'production'
    environment: 'prod'

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

# Load rules
rule_files:
  - '/etc/prometheus/rules/mongodb_alerts.yml'
  - '/etc/prometheus/rules/mongodb_recording.yml'

# Scrape configurations
scrape_configs:
  # Prometheus self-monitoring
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # MongoDB Replica Set - Primary
  - job_name: 'mongodb-rs0-primary'
    static_configs:
      - targets: ['mongodb-rs0-primary:9216']
        labels:
          cluster: 'rs0'
          role: 'primary'
          datacenter: 'dc1'

  # MongoDB Replica Set - Secondaries
  - job_name: 'mongodb-rs0-secondaries'
    static_configs:
      - targets:
          - 'mongodb-rs0-sec1:9216'
          - 'mongodb-rs0-sec2:9216'
        labels:
          cluster: 'rs0'
          role: 'secondary'
          datacenter: 'dc1'

  # MongoDB Sharded Cluster - Config Servers
  - job_name: 'mongodb-config-servers'
    static_configs:
      - targets:
          - 'mongodb-config1:9216'
          - 'mongodb-config2:9216'
          - 'mongodb-config3:9216'
        labels:
          cluster: 'sharded-prod'
          component: 'configsvr'

  # MongoDB Sharded Cluster - Shards
  - job_name: 'mongodb-shards'
    static_configs:
      - targets:
          - 'mongodb-shard01-rs0:9216'
          - 'mongodb-shard02-rs0:9216'
          - 'mongodb-shard03-rs0:9216'
        labels:
          cluster: 'sharded-prod'
          component: 'shard'

  # MongoDB Sharded Cluster - Mongos
  - job_name: 'mongodb-mongos'
    static_configs:
      - targets:
          - 'mongodb-mongos1:9216'
          - 'mongodb-mongos2:9216'
        labels:
          cluster: 'sharded-prod'
          component: 'mongos'
```

#### Configuration Kubernetes (Service Discovery)

```yaml
# mongodb-exporter-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: mongodb-exporter
  namespace: monitoring
  labels:
    app: mongodb-exporter
spec:
  selector:
    matchLabels:
      app: mongodb-exporter
  endpoints:
    - port: metrics
      interval: 30s
      scrapeTimeout: 10s
      path: /metrics
      relabelings:
        - sourceLabels: [__meta_kubernetes_pod_label_mongodb_cr_name]
          targetLabel: mongodb_cluster
        - sourceLabels: [__meta_kubernetes_pod_label_mongodb_cr_role]
          targetLabel: mongodb_role
```

#### Règles d'Enregistrement (Recording Rules)

Pré-calculer des métriques agrégées pour améliorer les performances des dashboards :

```yaml
# /etc/prometheus/rules/mongodb_recording.yml
groups:
  - name: mongodb_aggregations
    interval: 30s
    rules:
      # Taux d'opérations par seconde
      - record: mongodb:operations:rate5m
        expr: rate(mongodb_ss_opcounters[5m])

      # Utilisation mémoire résidente
      - record: mongodb:memory:resident_mb
        expr: mongodb_ss_mem_resident

      # Pourcentage cache utilisé
      - record: mongodb:cache:used_percent
        expr: |
          (
            mongodb_ss_wt_cache_bytes_currently_in_the_cache
            / mongodb_ss_wt_cache_maximum_bytes_configured
          ) * 100

      # Pourcentage cache dirty
      - record: mongodb:cache:dirty_percent
        expr: |
          (
            mongodb_ss_wt_cache_tracked_dirty_bytes_in_the_cache
            / mongodb_ss_wt_cache_maximum_bytes_configured
          ) * 100

      # Latence moyenne des opérations
      - record: mongodb:latency:ops_avg_ms
        expr: |
          rate(mongodb_ss_opLatencies_latency[5m])
          / rate(mongodb_ss_opLatencies_ops[5m])
          / 1000

      # Connexions actives
      - record: mongodb:connections:active
        expr: mongodb_ss_connections{conn_type="active"}

      # Ratio queue/connections
      - record: mongodb:queue:ratio
        expr: |
          (
            mongodb_ss_globalLock_currentQueue_readers
            + mongodb_ss_globalLock_currentQueue_writers
          ) / mongodb_ss_connections{conn_type="current"}

      # Lag de réplication max (par replica set)
      - record: mongodb:replication:lag_max_seconds
        expr: |
          max by (cluster) (
            mongodb_rs_members_optimeDate
            - on(cluster) group_left
            mongodb_rs_members_optimeDate{state="PRIMARY"}
          )
```

---

## Métriques Essentielles

### Taxonomie des Métriques

Le MongoDB Exporter expose plusieurs milliers de métriques. Voici les plus critiques pour le monitoring :

#### 1. Métriques d'Opérations

```promql
# Opérations par seconde (par type)
mongodb_ss_opcounters
  - insert
  - query
  - update
  - delete
  - getmore
  - command

# Taux de croissance
rate(mongodb_ss_opcounters[5m])

# Exemples de requêtes
# Total des opérations/sec
sum(rate(mongodb_ss_opcounters[5m]))

# Par type
sum by (type) (rate(mongodb_ss_opcounters[5m]))

# Par instance
sum by (instance) (rate(mongodb_ss_opcounters[5m]))
```

#### 2. Métriques de Performance

```promql
# Latences (microseconds)
mongodb_ss_opLatencies_latency
mongodb_ss_opLatencies_ops

# Latence moyenne par opération
rate(mongodb_ss_opLatencies_latency{type="command"}[5m])
/ rate(mongodb_ss_opLatencies_ops{type="command"}[5m])
/ 1000  # Conversion en millisecondes

# Connexions
mongodb_ss_connections{conn_type="current"}
mongodb_ss_connections{conn_type="available"}
mongodb_ss_connections{conn_type="active"}

# Queues
mongodb_ss_globalLock_currentQueue_readers
mongodb_ss_globalLock_currentQueue_writers

# Active clients
mongodb_ss_globalLock_activeClients_readers
mongodb_ss_globalLock_activeClients_writers
```

#### 3. Métriques de Cache (WiredTiger)

```promql
# Cache size
mongodb_ss_wt_cache_maximum_bytes_configured
mongodb_ss_wt_cache_bytes_currently_in_the_cache

# Cache dirty
mongodb_ss_wt_cache_tracked_dirty_bytes_in_the_cache

# Cache metrics
mongodb_ss_wt_cache_pages_evicted_by_application_threads
mongodb_ss_wt_cache_pages_read_into_cache
mongodb_ss_wt_cache_pages_written_from_cache

# Pourcentages calculés
(mongodb_ss_wt_cache_bytes_currently_in_the_cache
 / mongodb_ss_wt_cache_maximum_bytes_configured) * 100

(mongodb_ss_wt_cache_tracked_dirty_bytes_in_the_cache
 / mongodb_ss_wt_cache_maximum_bytes_configured) * 100
```

#### 4. Métriques de Mémoire

```promql
# Mémoire résidente (MB)
mongodb_ss_mem_resident

# Mémoire virtuelle (MB)
mongodb_ss_mem_virtual

# Mémoire mappée
mongodb_ss_mem_mapped
```

#### 5. Métriques de Réplication

```promql
# État du membre
mongodb_rs_members_state
  # 0: Startup
  # 1: Primary
  # 2: Secondary
  # 3: Recovering
  # 5: Startup2
  # 6: Unknown
  # 7: Arbiter
  # 8: Down
  # 9: Rollback
  # 10: Removed

# Lag de réplication (secondes)
mongodb_rs_members_optimeDate{state="PRIMARY"}
- on(cluster) mongodb_rs_members_optimeDate{state="SECONDARY"}

# Santé du membre (0=down, 1=up)
mongodb_rs_members_health

# Elections
mongodb_rs_members_electionTime

# Oplog
mongodb_rs_oplog_logSizeMB
mongodb_rs_oplog_usedSizeMB
mongodb_rs_oplog_timeDiff
```

#### 6. Métriques de Stockage

```promql
# Taille base de données
mongodb_dbstats_dataSize
mongodb_dbstats_storageSize
mongodb_dbstats_indexSize

# Collections
mongodb_collstats_size
mongodb_collstats_storageSize
mongodb_collstats_count

# Index
mongodb_indexstats_accesses_ops
```

#### 7. Métriques Réseau

```promql
# Trafic réseau (bytes/sec)
rate(mongodb_ss_network_bytesIn[5m])
rate(mongodb_ss_network_bytesOut[5m])

# Nombre de requêtes
mongodb_ss_network_numRequests
```

#### 8. Métriques de Sharding

```promql
# Nombre de chunks
mongodb_sharding_chunks_total

# Migrations
mongodb_sharding_active_migrations
mongodb_sharding_failed_migrations_total

# Balancer
mongodb_sharding_balancer_enabled
```

---

## Configuration Grafana

### 1. Installation et Configuration Initiale

#### Installation

```bash
# Ubuntu/Debian
sudo apt-get install -y apt-transport-https software-properties-common
sudo wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana

# Démarrer
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

#### Ajout de Prometheus comme Data Source

**Via Interface Web** :
1. Se connecter à Grafana : http://localhost:3000 (admin/admin)
2. Configuration → Data Sources → Add data source
3. Sélectionner Prometheus
4. URL : http://localhost:9090
5. Save & Test

**Via Provisioning** (automatisé) :

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://localhost:9090
    isDefault: true
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
      httpMethod: POST
    editable: false
```

### 2. Dashboard MongoDB - Replica Set

Dashboard complet pour monitoring d'un replica set :

```json
{
  "dashboard": {
    "title": "MongoDB Replica Set Overview",
    "tags": ["mongodb", "replica-set"],
    "timezone": "browser",
    "refresh": "30s",
    "time": {
      "from": "now-6h",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Replica Set Health",
        "type": "stat",
        "targets": [
          {
            "expr": "mongodb_rs_members_health",
            "legendFormat": "{{member_name}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {
                  "value": 0,
                  "color": "red"
                },
                {
                  "value": 1,
                  "color": "green"
                }
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "Operations per Second",
        "type": "graph",
        "targets": [
          {
            "expr": "sum by (type) (rate(mongodb_ss_opcounters[5m]))",
            "legendFormat": "{{type}}"
          }
        ],
        "yaxes": [
          {
            "label": "ops/sec",
            "format": "short"
          }
        ]
      },
      {
        "id": 3,
        "title": "Replication Lag",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_rs_members_optimeDate{state=\"PRIMARY\"} - on(cluster) group_right mongodb_rs_members_optimeDate{state=\"SECONDARY\"}",
            "legendFormat": "{{member_name}}"
          }
        ],
        "yaxes": [
          {
            "label": "seconds",
            "format": "s"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [30],
                "type": "gt"
              },
              "operator": {
                "type": "and"
              },
              "query": {
                "params": ["A", "5m", "now"]
              },
              "reducer": {
                "params": [],
                "type": "avg"
              },
              "type": "query"
            }
          ],
          "frequency": "60s",
          "handler": 1,
          "name": "Replication Lag Alert",
          "noDataState": "no_data",
          "notifications": []
        }
      },
      {
        "id": 4,
        "title": "Cache Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "(mongodb_ss_wt_cache_bytes_currently_in_the_cache / mongodb_ss_wt_cache_maximum_bytes_configured) * 100",
            "legendFormat": "Used %"
          },
          {
            "expr": "(mongodb_ss_wt_cache_tracked_dirty_bytes_in_the_cache / mongodb_ss_wt_cache_maximum_bytes_configured) * 100",
            "legendFormat": "Dirty %"
          }
        ],
        "yaxes": [
          {
            "label": "percentage",
            "format": "percent",
            "max": 100
          }
        ]
      },
      {
        "id": 5,
        "title": "Queue Depth",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_ss_globalLock_currentQueue_readers",
            "legendFormat": "Read Queue"
          },
          {
            "expr": "mongodb_ss_globalLock_currentQueue_writers",
            "legendFormat": "Write Queue"
          }
        ]
      },
      {
        "id": 6,
        "title": "Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_ss_connections{conn_type=\"current\"}",
            "legendFormat": "Current"
          },
          {
            "expr": "mongodb_ss_connections{conn_type=\"active\"}",
            "legendFormat": "Active"
          }
        ]
      },
      {
        "id": 7,
        "title": "Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_ss_mem_resident",
            "legendFormat": "Resident (MB)"
          }
        ],
        "yaxes": [
          {
            "label": "MB",
            "format": "mbytes"
          }
        ]
      },
      {
        "id": 8,
        "title": "Network I/O",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mongodb_ss_network_bytesIn[5m])",
            "legendFormat": "In"
          },
          {
            "expr": "rate(mongodb_ss_network_bytesOut[5m])",
            "legendFormat": "Out"
          }
        ],
        "yaxes": [
          {
            "label": "bytes/sec",
            "format": "Bps"
          }
        ]
      },
      {
        "id": 9,
        "title": "Oplog Size",
        "type": "gauge",
        "targets": [
          {
            "expr": "(mongodb_rs_oplog_usedSizeMB / mongodb_rs_oplog_logSizeMB) * 100",
            "legendFormat": "Oplog Usage %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "mode": "absolute",
              "steps": [
                {"value": 0, "color": "green"},
                {"value": 70, "color": "yellow"},
                {"value": 90, "color": "red"}
              ]
            },
            "unit": "percent",
            "max": 100
          }
        }
      },
      {
        "id": 10,
        "title": "Average Operation Latency",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mongodb_ss_opLatencies_latency{type=\"reads\"}[5m]) / rate(mongodb_ss_opLatencies_ops{type=\"reads\"}[5m]) / 1000",
            "legendFormat": "Reads (ms)"
          },
          {
            "expr": "rate(mongodb_ss_opLatencies_latency{type=\"writes\"}[5m]) / rate(mongodb_ss_opLatencies_ops{type=\"writes\"}[5m]) / 1000",
            "legendFormat": "Writes (ms)"
          },
          {
            "expr": "rate(mongodb_ss_opLatencies_latency{type=\"commands\"}[5m]) / rate(mongodb_ss_opLatencies_ops{type=\"commands\"}[5m]) / 1000",
            "legendFormat": "Commands (ms)"
          }
        ]
      }
    ]
  }
}
```

### 3. Dashboard MongoDB - Sharded Cluster

```json
{
  "dashboard": {
    "title": "MongoDB Sharded Cluster Overview",
    "panels": [
      {
        "title": "Cluster Topology",
        "type": "table",
        "targets": [
          {
            "expr": "mongodb_rs_members_state",
            "format": "table",
            "instant": true
          }
        ]
      },
      {
        "title": "Chunks Distribution",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (shard) (mongodb_sharding_chunks_total)",
            "legendFormat": "{{shard}}"
          }
        ]
      },
      {
        "title": "Balancer Activity",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mongodb_sharding_active_migrations[5m])",
            "legendFormat": "Active Migrations"
          },
          {
            "expr": "rate(mongodb_sharding_failed_migrations_total[5m])",
            "legendFormat": "Failed Migrations"
          }
        ]
      },
      {
        "title": "Mongos Operations",
        "type": "graph",
        "targets": [
          {
            "expr": "sum by (instance) (rate(mongodb_ss_opcounters{component=\"mongos\"}[5m]))",
            "legendFormat": "{{instance}}"
          }
        ]
      }
    ]
  }
}
```

### 4. Dashboard de Diagnostic Avancé

```json
{
  "dashboard": {
    "title": "MongoDB Advanced Diagnostics",
    "panels": [
      {
        "title": "Cache Efficiency",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mongodb_ss_wt_cache_pages_read_into_cache[5m])",
            "legendFormat": "Pages Read"
          },
          {
            "expr": "rate(mongodb_ss_wt_cache_pages_written_from_cache[5m])",
            "legendFormat": "Pages Written"
          },
          {
            "expr": "rate(mongodb_ss_wt_cache_pages_evicted_by_application_threads[5m])",
            "legendFormat": "Pages Evicted"
          }
        ]
      },
      {
        "title": "Ticket Utilization",
        "type": "graph",
        "targets": [
          {
            "expr": "mongodb_ss_wt_concurrentTransactions_read_available",
            "legendFormat": "Read Tickets Available"
          },
          {
            "expr": "mongodb_ss_wt_concurrentTransactions_read_out",
            "legendFormat": "Read Tickets In Use"
          },
          {
            "expr": "mongodb_ss_wt_concurrentTransactions_write_available",
            "legendFormat": "Write Tickets Available"
          },
          {
            "expr": "mongodb_ss_wt_concurrentTransactions_write_out",
            "legendFormat": "Write Tickets In Use"
          }
        ]
      },
      {
        "title": "Document Scan Efficiency",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mongodb_ss_metrics_queryExecutor_scanned[5m])",
            "legendFormat": "Documents Scanned"
          },
          {
            "expr": "rate(mongodb_ss_metrics_queryExecutor_scannedObjects[5m])",
            "legendFormat": "Objects Returned"
          }
        ]
      },
      {
        "title": "Page Faults",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(mongodb_ss_extra_info_page_faults[5m])",
            "legendFormat": "Page Faults/sec"
          }
        ]
      }
    ]
  }
}
```

---

## Règles d'Alerting

### Alerting avec Prometheus Alertmanager

#### Configuration des Règles

```yaml
# /etc/prometheus/rules/mongodb_alerts.yml
groups:
  - name: mongodb_critical_alerts
    interval: 30s
    rules:
      # Instance Down
      - alert: MongoDBInstanceDown
        expr: up{job=~"mongodb.*"} == 0
        for: 2m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "MongoDB instance {{ $labels.instance }} is down"
          description: "MongoDB instance {{ $labels.instance }} has been down for more than 2 minutes."
          runbook_url: "https://runbooks.example.com/mongodb/instance-down"

      # Replication Lag Critical
      - alert: MongoDBReplicationLagCritical
        expr: |
          (
            mongodb_rs_members_optimeDate{state="PRIMARY"}
            - on(cluster) group_right
            mongodb_rs_members_optimeDate{state="SECONDARY"}
          ) > 60
        for: 5m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "MongoDB replication lag critical on {{ $labels.member_name }}"
          description: "Replication lag is {{ $value }}s (threshold: 60s)"
          runbook_url: "https://runbooks.example.com/mongodb/replication-lag"

      # High Queue Depth
      - alert: MongoDBHighQueueDepth
        expr: |
          (
            mongodb_ss_globalLock_currentQueue_readers
            + mongodb_ss_globalLock_currentQueue_writers
          ) > 50
        for: 5m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "MongoDB high queue depth on {{ $labels.instance }}"
          description: "Queue depth is {{ $value }} (threshold: 50)"
          action: "Check for slow queries and missing indexes"

      # Cache Dirty High
      - alert: MongoDBCacheDirtyHigh
        expr: |
          (
            mongodb_ss_wt_cache_tracked_dirty_bytes_in_the_cache
            / mongodb_ss_wt_cache_maximum_bytes_configured
          ) * 100 > 40
        for: 10m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "MongoDB dirty cache high on {{ $labels.instance }}"
          description: "Dirty cache is {{ $value | humanizePercentage }} (threshold: 40%)"
          action: "Check I/O performance and checkpoint duration"

      # Connections Near Limit
      - alert: MongoDBConnectionsNearLimit
        expr: |
          (
            mongodb_ss_connections{conn_type="current"}
            / (mongodb_ss_connections{conn_type="current"} + mongodb_ss_connections{conn_type="available"})
          ) * 100 > 80
        for: 5m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "MongoDB connections near limit on {{ $labels.instance }}"
          description: "Using {{ $value | humanizePercentage }} of available connections"
          action: "Check connection pooling and potential connection leaks"

      # Oplog Window Low
      - alert: MongoDBOplogWindowLow
        expr: mongodb_rs_oplog_timeDiff < 3600
        for: 10m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "MongoDB oplog window low on {{ $labels.instance }}"
          description: "Oplog window is {{ $value }}s (threshold: 3600s / 1h)"
          action: "Consider increasing oplog size"

      # High Memory Usage
      - alert: MongoDBHighMemoryUsage
        expr: |
          (mongodb_ss_mem_resident / 1024)
          / on(instance)
          node_memory_MemTotal_bytes{job="node"} * 100 > 85
        for: 10m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "MongoDB high memory usage on {{ $labels.instance }}"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # Replica Set Member Not Healthy
      - alert: MongoDBReplicaSetMemberUnhealthy
        expr: mongodb_rs_members_health == 0
        for: 2m
        labels:
          severity: critical
          team: database
        annotations:
          summary: "MongoDB replica set member {{ $labels.member_name }} is unhealthy"
          description: "Member has been unhealthy for more than 2 minutes"

      # High Operation Latency
      - alert: MongoDBHighLatency
        expr: |
          rate(mongodb_ss_opLatencies_latency{type="reads"}[5m])
          / rate(mongodb_ss_opLatencies_ops{type="reads"}[5m])
          / 1000 > 100
        for: 5m
        labels:
          severity: warning
          team: database
        annotations:
          summary: "MongoDB high read latency on {{ $labels.instance }}"
          description: "Average read latency is {{ $value }}ms (threshold: 100ms)"
          action: "Check for slow queries and ensure proper indexing"

  - name: mongodb_capacity_alerts
    interval: 1h
    rules:
      # Disk Usage High
      - alert: MongoDBDiskUsageHigh
        expr: |
          (
            mongodb_dbstats_storageSize{database!~"admin|config|local"}
            / (1024*1024*1024)
          ) > 100
        labels:
          severity: warning
          team: database
        annotations:
          summary: "MongoDB database {{ $labels.database }} is large"
          description: "Database size is {{ $value }}GB"

      # Index Size Large
      - alert: MongoDBLargeIndexSize
        expr: |
          (
            mongodb_dbstats_indexSize{database!~"admin|config|local"}
            / mongodb_dbstats_dataSize
          ) > 1
        labels:
          severity: info
          team: database
        annotations:
          summary: "MongoDB indexes larger than data on {{ $labels.database }}"
          description: "Index to data ratio is {{ $value }}"
          action: "Review index usage and consider removing unused indexes"
```

#### Configuration Alertmanager

```yaml
# /etc/alertmanager/alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true

    - match:
        severity: warning
      receiver: 'slack-warnings'

    - match:
        severity: info
      receiver: 'slack-info'

receivers:
  - name: 'default'
    slack_configs:
      - channel: '#mongodb-alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_PAGERDUTY_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}'

  - name: 'slack-warnings'
    slack_configs:
      - channel: '#mongodb-warnings'
        color: 'warning'
        title: '[WARNING] {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Details:* {{ .Annotations.description }}
          *Action:* {{ .Annotations.action }}
          {{ end }}

  - name: 'slack-info'
    slack_configs:
      - channel: '#mongodb-info'
        color: 'good'
        title: '[INFO] {{ .GroupLabels.alertname }}'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']
```

### Alerting avec Grafana

```json
{
  "alert": {
    "name": "MongoDB Replication Lag",
    "conditions": [
      {
        "evaluator": {
          "params": [30],
          "type": "gt"
        },
        "operator": {
          "type": "and"
        },
        "query": {
          "params": ["A", "5m", "now"]
        },
        "reducer": {
          "params": [],
          "type": "avg"
        },
        "type": "query"
      }
    ],
    "executionErrorState": "alerting",
    "for": "5m",
    "frequency": "1m",
    "handler": 1,
    "message": "Replication lag exceeds 30 seconds",
    "name": "MongoDB Replication Lag",
    "noDataState": "no_data",
    "notifications": [
      {
        "uid": "slack-notifications"
      }
    ]
  }
}
```

---

## Cas d'Usage Avancés

### 1. Détection de Régression de Performance

**Objectif** : Identifier automatiquement les dégradations de performance après un déploiement.

```yaml
# recording_rules_baseline.yml
groups:
  - name: mongodb_baseline
    interval: 5m
    rules:
      # Latence p95 sur les dernières 24h
      - record: mongodb:latency:p95_24h
        expr: |
          histogram_quantile(0.95,
            rate(mongodb_ss_opLatencies_latency_bucket[24h])
          ) / 1000

      # Latence p95 actuelle (5min)
      - record: mongodb:latency:p95_5m
        expr: |
          histogram_quantile(0.95,
            rate(mongodb_ss_opLatencies_latency_bucket[5m])
          ) / 1000

      # Alert si dégradation > 50%
      - alert: MongoDBPerformanceRegression
        expr: |
          (
            mongodb:latency:p95_5m
            / mongodb:latency:p95_24h
          ) > 1.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Performance regression detected"
          description: "p95 latency increased by {{ $value | humanizePercentage }}"
```

### 2. Prédiction de Saturation (Trend Analysis)

```promql
# Prédire quand le disque sera plein (linear regression)
predict_linear(
  mongodb_dbstats_storageSize{database="production"}[7d],
  7*24*3600  # Prédiction à 7 jours
) > 500 * 1024 * 1024 * 1024  # 500 GB

# Prédire quand les connexions atteindront la limite
predict_linear(
  mongodb_ss_connections{conn_type="current"}[24h],
  24*3600  # Prédiction à 24h
) > (
  mongodb_ss_connections{conn_type="current"}
  + mongodb_ss_connections{conn_type="available"}
) * 0.9
```

### 3. Corrélation Multi-Métriques

**Identifier les patterns de problèmes complexes** :

```promql
# Charge élevée = haute latence + queues élevées + cache saturé
(
  (
    rate(mongodb_ss_opLatencies_latency{type="reads"}[5m])
    / rate(mongodb_ss_opLatencies_ops{type="reads"}[5m])
  ) > 100000
)
and
(
  mongodb_ss_globalLock_currentQueue_readers > 20
)
and
(
  (
    mongodb_ss_wt_cache_bytes_currently_in_the_cache
    / mongodb_ss_wt_cache_maximum_bytes_configured
  ) > 0.9
)
```

### 4. Analyse de Capacité

```python
#!/usr/bin/env python3
# capacity_analysis.py

import requests
from datetime import datetime, timedelta
import json

class MongoDBCapacityAnalyzer:
    def __init__(self, prometheus_url):
        self.prometheus_url = prometheus_url

    def query_prometheus(self, query, start, end, step='1h'):
        """Query Prometheus range"""
        url = f"{self.prometheus_url}/api/v1/query_range"
        params = {
            'query': query,
            'start': start.timestamp(),
            'end': end.timestamp(),
            'step': step
        }
        response = requests.get(url, params=params)
        return response.json()

    def analyze_growth(self, metric, days=30):
        """Analyze metric growth over time"""
        end = datetime.now()
        start = end - timedelta(days=days)

        data = self.query_prometheus(metric, start, end)

        if not data['data']['result']:
            return None

        values = data['data']['result'][0]['values']

        # Linear regression simple
        x = list(range(len(values)))
        y = [float(v[1]) for v in values]

        n = len(x)
        sum_x = sum(x)
        sum_y = sum(y)
        sum_xy = sum(x[i] * y[i] for i in range(n))
        sum_x2 = sum(x[i]**2 for i in range(n))

        # Slope (growth rate)
        slope = (n * sum_xy - sum_x * sum_y) / (n * sum_x2 - sum_x**2)

        # Intercept
        intercept = (sum_y - slope * sum_x) / n

        return {
            'current_value': y[-1],
            'growth_rate_per_hour': slope,
            'growth_rate_per_day': slope * 24,
            'projection_7d': intercept + slope * (len(x) + 7*24),
            'projection_30d': intercept + slope * (len(x) + 30*24)
        }

    def generate_report(self):
        """Generate capacity planning report"""
        metrics = {
            'Storage': 'mongodb_dbstats_storageSize{database="production"}',
            'Connections': 'mongodb_ss_connections{conn_type="current"}',
            'Memory': 'mongodb_ss_mem_resident'
        }

        report = {
            'generated_at': datetime.now().isoformat(),
            'metrics': {}
        }

        for name, query in metrics.items():
            analysis = self.analyze_growth(query)
            if analysis:
                report['metrics'][name] = analysis

        return report

if __name__ == '__main__':
    analyzer = MongoDBCapacityAnalyzer('http://localhost:9090')
    report = analyzer.generate_report()
    print(json.dumps(report, indent=2))
```

### 5. Monitoring Multi-Cluster

```yaml
# prometheus.yml pour multi-cluster
scrape_configs:
  # Cluster Production US-East
  - job_name: 'mongodb-prod-us-east'
    static_configs:
      - targets:
          - 'mongodb-prod-useast-1:9216'
          - 'mongodb-prod-useast-2:9216'
          - 'mongodb-prod-useast-3:9216'
        labels:
          cluster: 'prod'
          region: 'us-east-1'
          environment: 'production'

  # Cluster Production EU-West
  - job_name: 'mongodb-prod-eu-west'
    static_configs:
      - targets:
          - 'mongodb-prod-euwest-1:9216'
          - 'mongodb-prod-euwest-2:9216'
          - 'mongodb-prod-euwest-3:9216'
        labels:
          cluster: 'prod'
          region: 'eu-west-1'
          environment: 'production'

  # Cluster Staging
  - job_name: 'mongodb-staging'
    static_configs:
      - targets:
          - 'mongodb-staging-1:9216'
          - 'mongodb-staging-2:9216'
        labels:
          cluster: 'staging'
          region: 'us-east-1'
          environment: 'staging'
```

**Dashboard Multi-Cluster** :
```promql
# Operations totales par cluster
sum by (cluster) (rate(mongodb_ss_opcounters[5m]))

# Latence moyenne par région
avg by (region) (
  rate(mongodb_ss_opLatencies_latency[5m])
  / rate(mongodb_ss_opLatencies_ops[5m])
)

# Comparaison de santé
sum by (cluster, region) (mongodb_rs_members_health)
```

---

## Optimisation et Performance

### 1. Réduction de la Cardinalité

**Problème** : Trop de séries temporelles créent une charge excessive sur Prometheus.

**Solutions** :

```yaml
# Configuration exporter - Limiter les métriques collStats
mongodb_exporter:
  --mongodb.collstats-colls: "mydb.critical_collection1,mydb.critical_collection2"
  --no-mongodb.indexstats  # Désactiver si non nécessaire

# Prometheus - Dropping de métriques non utilisées
scrape_configs:
  - job_name: 'mongodb'
    metric_relabel_configs:
      # Supprimer les métriques de debug
      - source_labels: [__name__]
        regex: 'mongodb_ss_asserts_.*'
        action: drop

      # Garder uniquement certains labels
      - source_labels: [database]
        regex: 'admin|config|local'
        action: drop
```

### 2. Aggregation et Downsampling

```yaml
# Recording rules pour downsampling
groups:
  - name: mongodb_downsampling
    interval: 5m
    rules:
      # Agréger les opérations toutes les 5 minutes
      - record: mongodb:ops:rate5m:sum
        expr: sum(rate(mongodb_ss_opcounters[5m]))

      # Moyenner la latence
      - record: mongodb:latency:avg5m
        expr: |
          avg(
            rate(mongodb_ss_opLatencies_latency[5m])
            / rate(mongodb_ss_opLatencies_ops[5m])
          )
```

### 3. Configuration Retention

```yaml
# prometheus.yml
storage:
  tsdb:
    path: /prometheus
    retention.time: 30d  # 30 jours de rétention
    retention.size: 50GB  # OU limite en taille

    # Compaction
    min-block-duration: 2h
    max-block-duration: 36h
```

**Stratégie multi-tiers** :
- **Prometheus** : 30 jours, haute résolution (15s)
- **Thanos/Cortex** : 1 an, résolution moyenne (5m)
- **S3 Cold Storage** : Illimité, résolution faible (1h)

---

## Troubleshooting

### Problème 1 : Exporter Ne Se Connecte Pas

**Symptômes** :
```
Error: unable to connect to MongoDB
```

**Diagnostic** :
```bash
# Tester la connexion manuellement
mongo "mongodb://mongodb_exporter:password@localhost:27017/?authSource=admin"

# Vérifier les logs de l'exporter
journalctl -u mongodb_exporter -f

# Vérifier les permissions
mongo --eval "
  db.getSiblingDB('admin').auth('mongodb_exporter', 'password');
  db.runCommand({ serverStatus: 1 });
"
```

**Solutions** :
- Vérifier le mot de passe et l'URI
- Vérifier que l'utilisateur a le rôle `clusterMonitor`
- Vérifier la connectivité réseau
- Vérifier le firewall (port 27017)

### Problème 2 : Métriques Manquantes

**Symptômes** :
```
Certaines métriques n'apparaissent pas dans Grafana
```

**Diagnostic** :
```bash
# Vérifier les métriques exposées
curl -s http://localhost:9216/metrics | grep mongodb_rs

# Vérifier que Prometheus scrape correctement
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.job=="mongodb")'

# Vérifier les erreurs de scrape
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | select(.health=="down")'
```

**Solutions** :
- Activer `--collect-all` sur l'exporter
- Vérifier les options de collecte spécifiques
- Augmenter le timeout de scrape dans Prometheus
- Vérifier les logs Prometheus pour les erreurs

### Problème 3 : Performance Prometheus Dégradée

**Symptômes** :
```
Queries lentes, dashboard timeout
```

**Diagnostic** :
```promql
# Nombre de séries temporelles
count(up)

# Taux d'ingestion
rate(prometheus_tsdb_head_samples_appended_total[5m])

# Utilisation mémoire
process_resident_memory_bytes
```

**Solutions** :
```yaml
# 1. Augmenter les ressources
storage:
  tsdb:
    retention.size: 100GB  # Augmenter

# 2. Réduire la cardinalité
scrape_configs:
  - job_name: mongodb
    metric_relabel_configs:
      - action: drop
        regex: 'mongodb_collstats_.*'
        source_labels: [__name__]

# 3. Utiliser recording rules
# Pré-calculer les requêtes complexes

# 4. Federation ou Thanos
# Pour scale horizontal
```

### Problème 4 : Alertes Non Reçues

**Diagnostic** :
```bash
# Vérifier l'état des alertes
curl http://localhost:9090/api/v1/alerts | jq .

# Vérifier Alertmanager
curl http://localhost:9093/api/v1/alerts | jq .

# Tester le webhook
curl -X POST http://localhost:9093/api/v1/alerts \
  -d '[{"labels":{"alertname":"test","severity":"warning"}}]'
```

**Solutions** :
- Vérifier la configuration des routes dans Alertmanager
- Vérifier les credentials (Slack, PagerDuty)
- Vérifier les inhibit rules
- Tester manuellement les receivers

---

## Bonnes Pratiques

### 1. Checklist de Déploiement

```yaml
Infrastructure:
  ✓ Exporter déployé sur chaque instance MongoDB
  ✓ Service systemd configuré avec auto-restart
  ✓ Utilisateur monitoring créé avec permissions minimales
  ✓ Firewall configuré (port 9216)
  ✓ TLS activé si réseau non sécurisé

Prometheus:
  ✓ Scrape interval approprié (15-30s)
  ✓ Retention configurée selon besoin (30d min)
  ✓ Recording rules créées pour requêtes fréquentes
  ✓ Alert rules configurées
  ✓ Service discovery configuré (si Kubernetes)
  ✓ Backup de configuration versionnée (Git)

Grafana:
  ✓ Data source configurée
  ✓ Dashboards importés et testés
  ✓ Permissions configurées (RBAC)
  ✓ Alerting configuré avec notifications
  ✓ Dashboards sauvegardés (provisioning)

Alerting:
  ✓ Alertmanager configuré
  ✓ Routes et receivers testés
  ✓ Runbooks documentés
  ✓ Escalation paths définis
  ✓ Test mensuel de l'alerting

Documentation:
  ✓ Architecture documentée
  ✓ Dashboards documentés
  ✓ Runbooks créés pour alertes critiques
  ✓ Procédures de troubleshooting
  ✓ Contact on-call défini
```

### 2. Sécurité

```yaml
# TLS pour MongoDB Exporter
--web.tls-cert=/path/to/cert.pem
--web.tls-key=/path/to/key.pem

# Basic Auth
--web.config.file=/path/to/web-config.yml

# web-config.yml
basic_auth_users:
  prometheus: $2y$10$...  # bcrypt hash
```

```yaml
# Prometheus - mTLS
scrape_configs:
  - job_name: 'mongodb'
    scheme: https
    tls_config:
      ca_file: /path/to/ca.pem
      cert_file: /path/to/client-cert.pem
      key_file: /path/to/client-key.pem
```

### 3. High Availability

**Prometheus HA** :
```yaml
# Déployer 2+ instances Prometheus identiques
# Utiliser Thanos ou Cortex pour déduplication

# prometheus-1.yml
external_labels:
  replica: A

# prometheus-2.yml
external_labels:
  replica: B
```

**Grafana HA** :
```yaml
# Configuration base de données externe
database:
  type: postgres
  host: postgres.example.com:5432
  name: grafana
  user: grafana
  password: ${GRAFANA_DB_PASSWORD}

# Session storage
session:
  provider: redis
  provider_config: "addr=redis.example.com:6379,pool_size=100,db=0"
```

### 4. Automatisation

```bash
#!/bin/bash
# deploy_monitoring.sh

# Déployer exporter sur tous les membres MongoDB
MONGO_HOSTS=("mongo1" "mongo2" "mongo3")

for host in "${MONGO_HOSTS[@]}"; do
  ssh $host "sudo systemctl stop mongodb_exporter"
  scp mongodb_exporter $host:/tmp/
  ssh $host "sudo mv /tmp/mongodb_exporter /usr/local/bin/ && sudo chmod +x /usr/local/bin/mongodb_exporter"
  ssh $host "sudo systemctl start mongodb_exporter"

  # Vérifier
  curl -s http://$host:9216/metrics | grep -q mongodb_up && echo "$host: OK" || echo "$host: FAILED"
done

# Reload Prometheus
curl -X POST http://prometheus:9090/-/reload
```

### 5. Coûts et Optimisation

```yaml
Optimisation des Coûts:
  # 1. Réduire la fréquence de scrape pour métriques non critiques
  scrape_interval: 60s  # Au lieu de 15s

  # 2. Utiliser le downsampling agressif
  retention: 15d  # Court terme
  # + Thanos avec downsampling pour long terme

  # 3. Limiter collstats/indexstats
  --mongodb.collstats-colls: "top10_critical_collections"

  # 4. Drop de métriques inutilisées
  metric_relabel_configs:
    - action: drop
      regex: 'go_.*|process_.*'
      source_labels: [__name__]
```

---

## Résumé pour SRE

### Stack Minimale

```yaml
Production Minimale Viable:
  - MongoDB Exporter: 1 par instance (ou discovery mode)
  - Prometheus: 1 instance (2 pour HA)
  - Grafana: 1 instance
  - Dashboards: Replica Set Overview + Alerting
  - Alerts: Critical uniquement (down, lag, cache)
  - Retention: 30 jours
```

### Métriques Critiques Top 10

```promql
1. up{job=~"mongodb.*"}  # Instance health
2. mongodb_rs_members_health  # Replica set health
3. mongodb:replication:lag_max_seconds  # Replication lag
4. mongodb_ss_globalLock_currentQueue_*  # Queue depth
5. mongodb:cache:dirty_percent  # Cache dirty
6. mongodb:cache:used_percent  # Cache utilization
7. mongodb_ss_connections{conn_type="current"}  # Connections
8. mongodb:latency:ops_avg_ms  # Operation latency
9. rate(mongodb_ss_opcounters[5m])  # Operation rate
10. mongodb_ss_mem_resident  # Memory usage
```

### Commandes Utiles

```bash
# Vérifier santé exporter
curl -s http://localhost:9216/metrics | grep mongodb_up

# Tester query PromQL
curl -s 'http://localhost:9090/api/v1/query?query=up' | jq .

# Lister targets
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job:.job, health:.health}'

# Lister alertes actives
curl -s http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name:.labels.alertname, state:.state}'

# Reload Prometheus config
curl -X POST http://localhost:9090/-/reload

# Export dashboard Grafana
curl -H "Authorization: Bearer YOUR_API_KEY" \
  http://localhost:3000/api/dashboards/uid/mongodb-replica-set
```

---

## Conclusion

L'intégration MongoDB avec Prometheus et Grafana offre une solution de monitoring **moderne, scalable et flexible** pour les environnements de production. Cette stack permet :

1. **Visibilité complète** : Historique, trends, corrélations
2. **Alerting intelligent** : Multi-seuils, inhibition, escalation
3. **Analyse avancée** : PromQL pour requêtes complexes
4. **Visualisation riche** : Dashboards interactifs et customisables
5. **Intégration cloud-native** : Kubernetes, service discovery, HA

**Points clés à retenir** :
- Déployer un exporter par instance ou utiliser discovery mode
- Commencer avec les dashboards de base, enrichir progressivement
- Configurer les alertes critiques dès le début
- Documenter les runbooks pour chaque alerte
- Optimiser la cardinalité pour la performance
- Planifier la rétention et le stockage long terme

**Prochaines étapes** :
- Déployer l'infrastructure de monitoring
- Importer les dashboards de référence
- Configurer l'alerting critique
- Former l'équipe aux dashboards et requêtes PromQL
- Établir les runbooks d'intervention
- Mettre en place les tests réguliers

---

**Références** :
- [Percona MongoDB Exporter](https://github.com/percona/mongodb_exporter)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [PromQL Basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [MongoDB Monitoring Best Practices](https://www.mongodb.com/docs/manual/administration/monitoring/)

⏭️ [MongoDB Ops Manager](/13-monitoring-administration/08-mongodb-ops-manager.md)
