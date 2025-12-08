🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.6 MongoDB Atlas Backup

## Introduction

MongoDB Atlas offre une solution de sauvegarde entièrement gérée et cloud-native qui élimine la complexité opérationnelle des backups traditionnels. Intégré nativement à la plateforme, Atlas Backup propose des snapshots automatiques, une restauration point-in-time, une réplication géographique et une conformité aux standards industriels, le tout sans nécessiter de gestion d'infrastructure.

Cette section détaille les capacités, la configuration et les stratégies d'utilisation d'Atlas Backup pour garantir une continuité d'activité optimale dans les environnements cloud.

## Architecture Atlas Backup

### Vue d'Ensemble du Système

```
┌────────────────────────────────────────────────────────────┐
│                   MongoDB Atlas Cluster                    │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │
│  │ Primary  │  │Secondary │  │Secondary │                  │
│  │  Node    │  │  Node    │  │  Node    │                  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘                  │
└───────┼─────────────┼─────────────┼────────────────────────┘
        │             │             │
        │  Continuous Oplog Streaming
        │             │             │
        v             v             v
┌────────────────────────────────────────────────────────────┐
│            Atlas Backup Infrastructure                     │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Snapshot Storage (S3-compatible)             │  │
│  │  • Snapshots toutes les 6-24h (configurable)         │  │
│  │  • Compression automatique                           │  │
│  │  • Chiffrement at-rest                               │  │
│  │  • Multi-région réplication                          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Oplog Store (Point-in-Time)                  │  │
│  │  • Streaming continu depuis le cluster               │  │
│  │  • Rétention 24h-7j (selon plan)                     │  │
│  │  • Permet PITR avec précision à la seconde           │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │         Queryable Backup (Snapshot Isolated)         │  │
│  │  • Accès read-only au backup                         │  │
│  │  • Validation sans restauration complète             │  │
│  │  • Queries directes sur les snapshots                │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                          │
                          v
┌─────────────────────────────────────────────────────────────┐
│              Cross-Region Backup Copies                     │
│  • Réplication automatique vers région secondaire           │
│  • Protection contre disaster datacenter                    │
│  • Compliance réglementaire géographique                    │
└─────────────────────────────────────────────────────────────┘
```

### Types de Backups Atlas

```yaml
Cloud Backups (Recommandé):
  disponibilité: Tous les tiers (M10+)
  snapshots:
    fréquence: Configurable (6h-24h)
    rétention: Flexible (7j-180j)
    type: Snapshots incrémentiaux
  point_in_time_recovery:
    disponible: Oui (24h-7j selon config)
    granularité: Seconde
  cross_region:
    disponible: Oui
    automatique: Optionnel
  encryption: AES-256 at-rest
  compliance: SOC 2, ISO 27001, HIPAA eligible
  pricing: Inclus dans le tier, ou based on storage

Legacy Backups (Déprécié):
  disponibilité: Anciens clusters uniquement
  migration: Vers Cloud Backups recommandée
  end_of_life: Progressive
```

## Configuration Atlas Backup

### Activation via Atlas UI

La configuration se fait via l'interface Atlas en quelques clics :

```yaml
# Navigation Atlas UI
Project → Clusters → [Nom du Cluster] → Backup Tab

Configuration de base:
  - Enable/Disable Backup
  - Snapshot Frequency (6h, 8h, 12h, 24h)
  - Retention Policies
  - Point-in-Time Windows
  - Cross-Region Copies

Politiques de rétention typiques:
  production_critical:
    snapshots: Toutes les 6 heures
    daily: 7 jours
    weekly: 4 semaines
    monthly: 6 mois
    yearly: 2 ans
    pit_window: 7 jours

  production_standard:
    snapshots: Quotidien
    daily: 7 jours
    weekly: 4 semaines
    monthly: 3 mois
    pit_window: 2 jours

  development:
    snapshots: Quotidien
    daily: 3 jours
    pit_window: 1 jour
```

### Configuration via Atlas CLI

```bash
# Installation Atlas CLI
curl -LO https://fastdl.mongodb.org/mongocli/mongodb-atlas-cli_latest_linux_x86_64.tar.gz
tar -xzf mongodb-atlas-cli_latest_linux_x86_64.tar.gz
sudo mv mongocli /usr/local/bin/atlas

# Authentification
atlas auth login

# Lister les clusters
atlas clusters list --projectId <project-id>

# Afficher la configuration backup d'un cluster
atlas backups schedule describe <cluster-name> \
  --projectId <project-id>

# Configurer la politique de backup
atlas backups schedule update <cluster-name> \
  --projectId <project-id> \
  --referenceHourOfDay 2 \
  --referenceMinuteOfHour 0 \
  --retentionUnit days \
  --retentionValue 7

# Exemple de configuration complète
cat > backup-policy.json <<EOF
{
  "autoExportEnabled": false,
  "referenceHourOfDay": 2,
  "referenceMinuteOfHour": 0,
  "restoreWindowDays": 7,
  "updateSnapshots": true,
  "policies": [
    {
      "id": "daily",
      "policyItems": [
        {
          "frequencyInterval": 1,
          "frequencyType": "daily",
          "retentionUnit": "days",
          "retentionValue": 7
        }
      ]
    },
    {
      "id": "weekly",
      "policyItems": [
        {
          "frequencyInterval": 1,
          "frequencyType": "weekly",
          "retentionUnit": "weeks",
          "retentionValue": 4
        }
      ]
    },
    {
      "id": "monthly",
      "policyItems": [
        {
          "frequencyInterval": 1,
          "frequencyType": "monthly",
          "retentionUnit": "months",
          "retentionValue": 6
        }
      ]
    }
  ]
}
EOF

# Appliquer la configuration
atlas backups schedule update <cluster-name> \
  --projectId <project-id> \
  --file backup-policy.json
```

### Configuration via API Atlas

```bash
#!/bin/bash
# configure_atlas_backup_api.sh

# Credentials Atlas API
ATLAS_PUBLIC_KEY="your-public-key"
ATLAS_PRIVATE_KEY="your-private-key"
PROJECT_ID="your-project-id"
CLUSTER_NAME="production-cluster"

# API Base URL
API_URL="https://cloud.mongodb.com/api/atlas/v1.0"

# Fonction pour appeler l'API
call_atlas_api() {
  local method=$1
  local endpoint=$2
  local data=$3

  curl -s -X "$method" \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    ${data:+-d "$data"} \
    "${API_URL}${endpoint}"
}

# Obtenir la configuration actuelle
get_current_config() {
  echo "=== Current Backup Configuration ==="
  call_atlas_api GET "/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/schedule" | jq .
}

# Configurer la politique de backup
configure_backup_policy() {
  local config=$(cat <<EOF
{
  "autoExportEnabled": false,
  "referenceHourOfDay": 2,
  "referenceMinuteOfHour": 0,
  "restoreWindowDays": 7,
  "updateSnapshots": true,
  "policies": [
    {
      "id": "hourly",
      "policyItems": [
        {
          "frequencyInterval": 6,
          "frequencyType": "hourly",
          "retentionUnit": "days",
          "retentionValue": 2
        }
      ]
    },
    {
      "id": "daily",
      "policyItems": [
        {
          "frequencyInterval": 1,
          "frequencyType": "daily",
          "retentionUnit": "days",
          "retentionValue": 7
        }
      ]
    },
    {
      "id": "weekly",
      "policyItems": [
        {
          "frequencyInterval": 1,
          "frequencyType": "weekly",
          "retentionUnit": "weeks",
          "retentionValue": 4
        }
      ]
    },
    {
      "id": "monthly",
      "policyItems": [
        {
          "frequencyInterval": 1,
          "frequencyType": "monthly",
          "retentionUnit": "months",
          "retentionValue": 12
        }
      ]
    }
  ]
}
EOF
)

  echo "=== Configuring Backup Policy ==="
  call_atlas_api PATCH "/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/schedule" "$config"
}

# Activer les copies cross-region
enable_cross_region_backup() {
  local copy_region="US_WEST_2"  # Région secondaire

  local config=$(cat <<EOF
{
  "copySettings": [
    {
      "cloudProvider": "AWS",
      "regionName": "$copy_region",
      "replicationSpecId": "auto",
      "shouldCopyOplogs": true,
      "frequencies": ["HOURLY", "DAILY", "WEEKLY", "MONTHLY"]
    }
  ]
}
EOF
)

  echo "=== Enabling Cross-Region Backup Copy to $copy_region ==="
  call_atlas_api PATCH "/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/schedule" "$config"
}

# Exécution
get_current_config
configure_backup_policy
enable_cross_region_backup

echo "✓ Atlas Backup configuration completed"
```

### Configuration via Terraform

```hcl
# atlas_backup_configuration.tf

terraform {
  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.14"
    }
  }
}

provider "mongodbatlas" {
  public_key  = var.atlas_public_key
  private_key = var.atlas_private_key
}

# Configuration de la politique de backup
resource "mongodbatlas_cloud_backup_schedule" "production_backup" {
  project_id   = var.atlas_project_id
  cluster_name = var.cluster_name

  # Point-in-Time Recovery window
  restore_window_days = 7

  # Heure de référence pour les snapshots (UTC)
  reference_hour_of_day    = 2
  reference_minute_of_hour = 0

  # Politique de snapshot
  policy_item_hourly {
    frequency_interval = 6  # Toutes les 6 heures
    retention_unit     = "days"
    retention_value    = 2
  }

  policy_item_daily {
    frequency_interval = 1  # Quotidien
    retention_unit     = "days"
    retention_value    = 7
  }

  policy_item_weekly {
    frequency_interval = 1  # Hebdomadaire
    retention_unit     = "weeks"
    retention_value    = 4
  }

  policy_item_monthly {
    frequency_interval = 1  # Mensuel
    retention_unit     = "months"
    retention_value    = 12
  }

  # Export automatique vers S3 (optionnel)
  auto_export_enabled = false

  # Copie cross-region
  copy_settings {
    cloud_provider = "AWS"
    region_name    = "US_WEST_2"

    frequencies = [
      "HOURLY",
      "DAILY",
      "WEEKLY",
      "MONTHLY"
    ]
  }
}

# Configuration de l'export automatique vers S3
resource "mongodbatlas_cloud_backup_snapshot_export_bucket" "backup_export" {
  project_id   = var.atlas_project_id
  bucket_name  = "company-mongodb-backups"
  cloud_provider = "AWS"

  iam_role_id = aws_iam_role.atlas_backup_export.arn
}

# IAM Role AWS pour l'export
resource "aws_iam_role" "atlas_backup_export" {
  name = "MongoDBAtlasBackupExport"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${var.atlas_aws_account_id}:root"
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "sts:ExternalId" = var.atlas_external_id
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "atlas_backup_export" {
  name = "MongoDBAtlasBackupExportPolicy"
  role = aws_iam_role.atlas_backup_export.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:GetObject",
          "s3:ListBucket",
          "s3:GetBucketLocation"
        ]
        Resource = [
          "arn:aws:s3:::company-mongodb-backups",
          "arn:aws:s3:::company-mongodb-backups/*"
        ]
      }
    ]
  })
}

# Variables
variable "atlas_public_key" {
  description = "MongoDB Atlas Public Key"
  type        = string
  sensitive   = true
}

variable "atlas_private_key" {
  description = "MongoDB Atlas Private Key"
  type        = string
  sensitive   = true
}

variable "atlas_project_id" {
  description = "MongoDB Atlas Project ID"
  type        = string
}

variable "cluster_name" {
  description = "MongoDB Atlas Cluster Name"
  type        = string
}

variable "atlas_aws_account_id" {
  description = "MongoDB Atlas AWS Account ID for assume role"
  type        = string
  default     = "123456789012"  # ID compte Atlas
}

variable "atlas_external_id" {
  description = "External ID for Atlas assume role"
  type        = string
  sensitive   = true
}

# Outputs
output "backup_policy_id" {
  value       = mongodbatlas_cloud_backup_schedule.production_backup.id
  description = "Backup policy ID"
}

output "restore_window_days" {
  value       = mongodbatlas_cloud_backup_schedule.production_backup.restore_window_days
  description = "Point-in-time restore window in days"
}
```

## Point-in-Time Recovery (PITR)

### Comprendre PITR dans Atlas

```
Timeline de Point-in-Time Recovery:

├─────────┬─────────┬─────────┬─────────┬─────────┬──────> Temps
│         │         │         │         │         │
│    Snapshot   Snapshot   Snapshot   Snapshot    │
│    06:00      12:00      18:00      00:00       │
│         │         │         │         │         │
├─────────┴─────────┴─────────┴─────────┴─────────┤
│          Oplog Continu (PITR Window)            │
│          Restauration possible à TOUTE          │
│          seconde dans cette fenêtre             │
└─────────────────────────────────────────────────┘

Exemple: Incident à 14:37:22
→ Restauration possible à 14:37:21
→ Perte maximale: 1 seconde de données
```

### Cas d'Usage PITR

```yaml
Scénario 1 - Erreur Application:
  incident: "DELETE sans WHERE clause à 14:35:00"
  action: "Restore à 14:34:59 (juste avant)"
  rpo: "0 seconde de perte"
  impact: "Erreur annulée complètement"

Scénario 2 - Corruption Données:
  incident: "Bug applicatif corrompt données depuis 10:00"
  action: "Restore à 09:59:59"
  rpo: "Retour avant corruption"
  post_action: "Deploy fix, rejouer ops valides"

Scénario 3 - Ransomware:
  incident: "Chiffrement malveillant détecté 15:45"
  action: "Restore à 15:00 (dernière vérification clean)"
  rpo: "45 minutes de données"
  post_action: "Investigation sécurité, replay sélectif"

Scénario 4 - Test/Audit:
  objectif: "Vérifier état base à un moment précis"
  action: "PITR vers cluster temporaire"
  usage: "Analyse forensique, compliance check"
```

### Restauration PITR via Atlas UI

```yaml
# Navigation Atlas UI
Project → Clusters → [Cluster] → Backup Tab → Snapshots

Étapes:
  1. Cliquer sur "Restore" ou "Download"
  2. Sélectionner "Point in Time" (au lieu de "Snapshot")
  3. Choisir date et heure précise (à la seconde)
  4. Options de restauration:
     a) Automated Restore:
        - Nouveau cluster créé automatiquement
        - Configuration identique au source
        - Nom: [original]-restore-[timestamp]
     b) Download:
        - Archive tar.gz téléchargeable
        - Restauration manuelle ensuite

  5. Configuration post-restore:
     - Ajuster le sizing si nécessaire
     - Configurer les IP whitelists
     - Reconfigurer les connexions applicatives

Durée typique:
  - Cluster M10-M30: 15-30 minutes
  - Cluster M40-M60: 30-60 minutes
  - Cluster > M80: 1-2 heures
```

### Restauration PITR via API

```bash
#!/bin/bash
# atlas_pitr_restore.sh

ATLAS_PUBLIC_KEY="your-public-key"
ATLAS_PRIVATE_KEY="your-private-key"
PROJECT_ID="your-project-id"
CLUSTER_NAME="production-cluster"

# Point de restauration (ISO 8601)
RESTORE_POINT="2024-12-08T14:37:21Z"

# Configuration du cluster de restauration
RESTORE_CLUSTER_NAME="${CLUSTER_NAME}-restore-$(date +%Y%m%d-%H%M%S)"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Créer une restauration PITR
create_pitr_restore() {
  log "Creating Point-in-Time Restore to $RESTORE_POINT"

  local restore_config=$(cat <<EOF
{
  "deliveryType": "automated",
  "targetClusterName": "$RESTORE_CLUSTER_NAME",
  "targetGroupId": "$PROJECT_ID",
  "pointInTimeUTCSeconds": $(date -d "$RESTORE_POINT" +%s)
}
EOF
)

  local response=$(curl -s -X POST \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    -d "$restore_config" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/restoreJobs")

  local job_id=$(echo "$response" | jq -r '.id')

  if [ "$job_id" != "null" ]; then
    log "✓ Restore job created: $job_id"
    echo "$job_id"
  else
    log "✗ Failed to create restore job"
    echo "$response" | jq .
    exit 1
  fi
}

# Monitorer le job de restauration
monitor_restore_job() {
  local job_id=$1

  log "Monitoring restore job: $job_id"

  while true; do
    local status=$(curl -s -X GET \
      --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
      "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/restoreJobs/${job_id}" \
      | jq -r '.deliveryType')

    case "$status" in
      "COMPLETED")
        log "✓ Restore completed successfully"
        break
        ;;
      "FAILED")
        log "✗ Restore failed"
        exit 1
        ;;
      "IN_PROGRESS"|"PENDING")
        log "  Restore in progress... ($status)"
        sleep 60
        ;;
      *)
        log "  Unknown status: $status"
        sleep 30
        ;;
    esac
  done
}

# Obtenir les détails du cluster restauré
get_restored_cluster_details() {
  log "Retrieving restored cluster details..."

  local cluster_info=$(curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${RESTORE_CLUSTER_NAME}")

  local connection_string=$(echo "$cluster_info" | jq -r '.connectionStrings.standardSrv')
  local state=$(echo "$cluster_info" | jq -r '.stateName')

  log "Cluster Name: $RESTORE_CLUSTER_NAME"
  log "State: $state"
  log "Connection String: $connection_string"

  echo "$connection_string"
}

# Main execution
main() {
  log "=== MongoDB Atlas Point-in-Time Restore ==="
  log "Source Cluster: $CLUSTER_NAME"
  log "Restore Point: $RESTORE_POINT"
  log "Target Cluster: $RESTORE_CLUSTER_NAME"

  local job_id=$(create_pitr_restore)

  monitor_restore_job "$job_id"

  local connection_string=$(get_restored_cluster_details)

  log ""
  log "✓ Point-in-Time Restore completed successfully"
  log "  Connect to restored cluster at:"
  log "  $connection_string"
  log ""
  log "⚠️  Remember to:"
  log "  1. Verify data integrity"
  log "  2. Update application configs if needed"
  log "  3. Delete original cluster when ready"
  log "  4. Rename restored cluster to original name"
}

main "$@"
```

## Queryable Backups

Atlas permet d'interroger directement les snapshots sans restauration complète :

```bash
#!/bin/bash
# query_atlas_backup.sh

# Créer un snapshot queryable (via API)
create_queryable_snapshot() {
  local snapshot_id=$1

  curl -s -X POST \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/snapshots/${snapshot_id}/queryable"
}

# Se connecter au snapshot pour requêtes
# Atlas fournit une connection string temporaire read-only
query_snapshot() {
  local queryable_connection_string=$1

  # Exemple: Vérifier des données spécifiques
  mongo "$queryable_connection_string" --eval "
    use production_db

    // Compter les documents à ce point dans le temps
    print('Total orders: ' + db.orders.countDocuments());

    // Vérifier l'existence d'un document spécifique
    doc = db.orders.findOne({ _id: ObjectId('...')});
    if (doc) {
      print('Document found: ' + JSON.stringify(doc));
    } else {
      print('Document not found at this point in time');
    }

    // Valider l'intégrité
    result = db.orders.validate({ full: true });
    print('Validation: ' + (result.valid ? 'OK' : 'FAILED'));
  "
}
```

### Cas d'Usage Queryable Backups

```yaml
Validation Pré-Restauration:
  objectif: "Vérifier que le backup contient les données attendues"
  action: "Query le snapshot avant restauration complète"
  avantage: "Économise temps et ressources"

Investigation Forensique:
  objectif: "Analyser l'état des données à un moment précis"
  action: "Requêtes complexes sur le snapshot"
  avantage: "Pas d'impact sur production"

Audit et Compliance:
  objectif: "Prouver l'état des données à une date donnée"
  action: "Extraire des rapports depuis snapshots historiques"
  avantage: "Immutabilité garantie"

Data Recovery Sélectif:
  objectif: "Restaurer seulement certains documents"
  action: "Query + export manuel des données nécessaires"
  avantage: "Évite restauration complète"
```

## Téléchargement et Export de Backups

### Download Snapshot pour Utilisation Locale

```bash
#!/bin/bash
# download_atlas_snapshot.sh

ATLAS_PUBLIC_KEY="your-public-key"
ATLAS_PRIVATE_KEY="your-private-key"
PROJECT_ID="your-project-id"
CLUSTER_NAME="production-cluster"
DOWNLOAD_DIR="/backup/atlas-downloads"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Lister les snapshots disponibles
list_snapshots() {
  log "Listing available snapshots..."

  curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/snapshots" \
    | jq -r '.results[] | "\(.id) | \(.createdAt) | \(.type)"'
}

# Demander le téléchargement d'un snapshot
request_download() {
  local snapshot_id=$1

  log "Requesting snapshot download: $snapshot_id"

  local download_config=$(cat <<EOF
{
  "deliveryType": "download"
}
EOF
)

  local response=$(curl -s -X POST \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    -d "$download_config" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/snapshots/${snapshot_id}/restoreJobs")

  local job_id=$(echo "$response" | jq -r '.id')
  echo "$job_id"
}

# Obtenir l'URL de téléchargement
get_download_url() {
  local job_id=$1

  log "Waiting for download URL..."

  while true; do
    local response=$(curl -s -X GET \
      --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
      "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/restoreJobs/${job_id}")

    local url=$(echo "$response" | jq -r '.results[0].url // empty')

    if [ -n "$url" ]; then
      echo "$url"
      break
    fi

    sleep 30
  done
}

# Télécharger le snapshot
download_snapshot() {
  local url=$1
  local output_file="${DOWNLOAD_DIR}/atlas-snapshot-$(date +%Y%m%d-%H%M%S).tar.gz"

  log "Downloading snapshot to: $output_file"

  mkdir -p "$DOWNLOAD_DIR"

  wget --progress=bar:force \
    --auth-no-challenge \
    --user="$ATLAS_PUBLIC_KEY" \
    --password="$ATLAS_PRIVATE_KEY" \
    -O "$output_file" \
    "$url"

  if [ $? -eq 0 ]; then
    log "✓ Download completed"
    log "  Size: $(du -h $output_file | cut -f1)"

    # Checksum
    sha256sum "$output_file" > "${output_file}.sha256"

    echo "$output_file"
  else
    log "✗ Download failed"
    exit 1
  fi
}

# Main
main() {
  log "=== Atlas Snapshot Download ==="

  # Lister les snapshots
  list_snapshots

  # Sélectionner le snapshot le plus récent
  local latest_snapshot=$(curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/snapshots" \
    | jq -r '.results[0].id')

  log "Latest snapshot: $latest_snapshot"

  # Demander le téléchargement
  local job_id=$(request_download "$latest_snapshot")

  # Obtenir l'URL
  local download_url=$(get_download_url "$job_id")

  # Télécharger
  local local_file=$(download_snapshot "$download_url")

  log "✓ Snapshot downloaded successfully: $local_file"
}

main "$@"
```

### Export Automatique vers S3

Configuration de l'export automatique :

```yaml
# Via Atlas UI
Backup → Export → Configure Export

Configuration:
  export_bucket:
    provider: AWS|Azure|GCP
    bucket_name: "company-mongodb-exports"
    region: "us-east-1"
    iam_role: "arn:aws:iam::123456:role/MongoDBAtlasExport"

  export_schedule:
    frequency: "DAILY|WEEKLY|MONTHLY"
    retention: 90  # jours

  export_format:
    compression: true
    encryption: true
```

Script pour automatiser via API :

```bash
#!/bin/bash
# configure_automatic_export.sh

# Configurer le bucket d'export
configure_export_bucket() {
  local config=$(cat <<EOF
{
  "bucketName": "company-mongodb-exports",
  "cloudProvider": "AWS",
  "iamRoleId": "arn:aws:iam::123456789012:role/MongoDBAtlasExport"
}
EOF
)

  curl -s -X POST \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    -d "$config" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/backup/exportBuckets"
}

# Activer l'export automatique sur le cluster
enable_auto_export() {
  curl -s -X PATCH \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    -d '{"autoExportEnabled": true}' \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/schedule"
}
```

## Monitoring et Alertes

### Configuration des Alertes Atlas

```yaml
# Alertes recommandées pour Backups

Backup Failure:
  condition: "Snapshot creation failed"
  action: "Email + PagerDuty"
  severity: CRITICAL

Backup Delay:
  condition: "No successful backup in 26 hours"
  action: "Email + Slack"
  severity: WARNING

Storage Threshold:
  condition: "Backup storage > 80% of quota"
  action: "Email"
  severity: WARNING

PITR Window Expiring:
  condition: "PITR window < 24 hours remaining"
  action: "Email"
  severity: INFO

Cross-Region Copy Failure:
  condition: "Cross-region backup copy failed"
  action: "Email + PagerDuty"
  severity: HIGH
```

### Configuration via API

```bash
#!/bin/bash
# configure_backup_alerts.sh

# Créer une alerte pour échec de backup
create_backup_failure_alert() {
  local alert_config=$(cat <<EOF
{
  "eventTypeName": "BACKUP_FAILURE",
  "enabled": true,
  "notifications": [
    {
      "typeName": "EMAIL",
      "emailEnabled": true,
      "emailAddress": "ops@company.com",
      "delayMin": 0
    },
    {
      "typeName": "PAGER_DUTY",
      "serviceKey": "your-pagerduty-key"
    }
  ]
}
EOF
)

  curl -s -X POST \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    -d "$alert_config" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/alertConfigs"
}
```

### Script de Monitoring Personnalisé

```bash
#!/bin/bash
# monitor_atlas_backups.sh

ATLAS_PUBLIC_KEY="your-public-key"
ATLAS_PRIVATE_KEY="your-private-key"
PROJECT_ID="your-project-id"
CLUSTER_NAME="production-cluster"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Vérifier le dernier snapshot
check_latest_snapshot() {
  log "Checking latest snapshot..."

  local snapshot_info=$(curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/snapshots?pageNum=1&itemsPerPage=1")

  local created_at=$(echo "$snapshot_info" | jq -r '.results[0].createdAt')
  local status=$(echo "$snapshot_info" | jq -r '.results[0].status')
  local type=$(echo "$snapshot_info" | jq -r '.results[0].type')

  # Calculer l'âge
  local created_ts=$(date -d "$created_at" +%s)
  local current_ts=$(date +%s)
  local age_hours=$(( (current_ts - created_ts) / 3600 ))

  log "Latest snapshot:"
  log "  Created: $created_at"
  log "  Age: ${age_hours}h"
  log "  Status: $status"
  log "  Type: $type"

  # Alertes
  if [ $age_hours -gt 26 ]; then
    log "⚠️  WARNING: Latest snapshot is ${age_hours}h old (expected < 26h)"
    send_alert "Atlas backup is stale"
    return 1
  fi

  if [ "$status" != "completed" ]; then
    log "⚠️  WARNING: Latest snapshot status is $status"
    send_alert "Atlas backup status issue"
    return 1
  fi

  log "✓ Latest snapshot is OK"
  return 0
}

# Vérifier la fenêtre PITR
check_pitr_window() {
  log "Checking PITR window..."

  local schedule_info=$(curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/schedule")

  local restore_window=$(echo "$schedule_info" | jq -r '.restoreWindowDays')

  log "PITR Window: ${restore_window} days"

  if [ "$restore_window" -lt 2 ]; then
    log "⚠️  WARNING: PITR window is only ${restore_window} days"
    return 1
  fi

  log "✓ PITR window is adequate"
  return 0
}

# Vérifier l'utilisation du stockage
check_storage_usage() {
  log "Checking backup storage usage..."

  local storage_info=$(curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/backup/usage")

  local total_gb=$(echo "$storage_info" | jq -r '.totalGigabytes')

  log "Total backup storage: ${total_gb} GB"

  # Note: Ajouter logique de quota selon votre plan
}

# Fonction d'alerte
send_alert() {
  local message=$1

  # Slack webhook
  curl -X POST "$SLACK_WEBHOOK_URL" \
    -H 'Content-Type: application/json' \
    -d "{\"text\": \"🚨 Atlas Backup Alert: $message\"}"
}

# Main
main() {
  log "=== Atlas Backup Monitoring ==="

  check_latest_snapshot
  check_pitr_window
  check_storage_usage

  log "✓ Monitoring completed"
}

main "$@"
```

## Disaster Recovery avec Atlas

### Stratégie Multi-Région

```yaml
Architecture DR Complète:
  primary_region:
    region: "us-east-1"
    cluster: "production-primary"
    backup:
      snapshots: Toutes les 6h
      pitr_window: 7 jours

  secondary_region:
    region: "us-west-2"
    backup_copies: true
    purpose: "DR site"

  tertiary_region:
    region: "eu-west-1"
    backup_copies: true
    purpose: "Compliance EU"

RTO/RPO:
  scenario_1_snapshot_restore:
    rto: 30-60 minutes
    rpo: 6 heures max

  scenario_2_pitr_restore:
    rto: 30-60 minutes
    rpo: < 1 minute

  scenario_3_cross_region:
    rto: 60-120 minutes (délai réplication)
    rpo: 6 heures (fréquence snapshot)
```

### Procédure de Failover DR

```bash
#!/bin/bash
# atlas_dr_failover.sh

PRIMARY_REGION="US_EAST_1"
DR_REGION="US_WEST_2"
SOURCE_CLUSTER="production-primary"
DR_CLUSTER="production-dr"

log() {
  echo "[$(date +'%Y-%m-%d %H:%M:%S')] $1"
}

# Étape 1: Identifier le dernier backup disponible dans la région DR
identify_dr_backup() {
  log "Identifying latest backup in DR region..."

  local latest_backup=$(curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${SOURCE_CLUSTER}/backup/snapshots?regionName=${DR_REGION}" \
    | jq -r '.results[0].id')

  log "Latest DR backup: $latest_backup"
  echo "$latest_backup"
}

# Étape 2: Restaurer dans la région DR
restore_to_dr_region() {
  local snapshot_id=$1

  log "Restoring to DR region..."

  # Créer un nouveau cluster depuis le backup DR
  local restore_config=$(cat <<EOF
{
  "deliveryType": "automated",
  "targetClusterName": "$DR_CLUSTER",
  "targetGroupId": "$PROJECT_ID"
}
EOF
)

  curl -s -X POST \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    -H "Content-Type: application/json" \
    -d "$restore_config" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${SOURCE_CLUSTER}/backup/snapshots/${snapshot_id}/restoreJobs"
}

# Étape 3: Mettre à jour les DNS/connexions
update_connections() {
  log "Updating application connection strings..."

  # Cette partie dépend de votre infrastructure
  # Exemples:
  # - Mettre à jour Route53 / CloudFlare
  # - Mettre à jour variables d'environnement Kubernetes
  # - Notifier les équipes

  log "⚠️  Manual step: Update application configs to point to DR cluster"
}

# Main DR failover
main() {
  log "=== DISASTER RECOVERY FAILOVER ==="
  log "⚠️  This will initiate DR procedures"

  read -p "Confirm DR failover? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    log "DR failover cancelled"
    exit 0
  fi

  local dr_backup=$(identify_dr_backup)

  restore_to_dr_region "$dr_backup"

  log "Monitoring restore progress..."
  # Ajouter monitoring du job de restauration

  update_connections

  log "✓ DR failover initiated"
  log "  Monitor cluster readiness before switching traffic"
}

main "$@"
```

## Coûts et Optimisation

### Structure Tarifaire Atlas Backup

```yaml
Coûts de Stockage:
  snapshots:
    pricing: "$2.50 per GB/month"
    compression: "Automatique (ratio ~5:1)"
    deduplication: "Incrémental efficace"

  pitr_oplog:
    pricing: "$2.00 per GB/month"
    size: "Variable selon workload write"

  cross_region_copies:
    pricing: "+$1.50 per GB/month"
    transfer: "Gratuit entre régions AWS"

Exemple de Coût:
  cluster_data_size: 1000 GB
  compressed_snapshot: 200 GB (après compression)
  monthly_cost:
    snapshots: "$500 (200GB × $2.50)"
    oplog_7days: "$70 (10GB daily × 7 × $1.00)"
    cross_region: "$300 (200GB × $1.50)"
    total: "$870/month"
```

### Stratégies d'Optimisation

```yaml
Optimisation 1 - Ajuster Rétention:
  avant:
    daily: 30 jours
    monthly: 12 mois
    coût: "$1,200/month"

  après:
    daily: 7 jours
    weekly: 4 semaines
    monthly: 6 mois
    coût: "$650/month"
    économie: "46%"

Optimisation 2 - PITR Window:
  production_critical:
    pitr_window: 7 jours
    justifié: "Compliance requis"

  production_standard:
    pitr_window: 2 jours
    économie: "60% sur oplog storage"

  development:
    pitr_window: 0 (désactivé)
    économie: "100% sur oplog"

Optimisation 3 - Cross-Region:
  critique:
    regions: 3 (primary + 2 DR)

  standard:
    regions: 2 (primary + 1 DR)

  dev:
    regions: 1 (aucune copie DR)
    économie: "Significative"
```

### Monitoring des Coûts

```bash
#!/bin/bash
# monitor_atlas_backup_costs.sh

# Obtenir l'utilisation du stockage backup
get_backup_storage_usage() {
  curl -s -X GET \
    --digest -u "${ATLAS_PUBLIC_KEY}:${ATLAS_PRIVATE_KEY}" \
    "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/backup/usage" \
    | jq '{
      totalGigabytes: .totalGigabytes,
      estimatedMonthlyCost: (.totalGigabytes * 2.5),
      clusters: [.clusters[] | {
        name: .clusterName,
        size: .sizeBytes,
        sizeGB: (.sizeBytes / 1024 / 1024 / 1024)
      }]
    }'
}

# Calculer les coûts prévisionnels
calculate_projected_costs() {
  local total_gb=$1

  local snapshot_cost=$(echo "$total_gb * 2.5" | bc)
  local oplog_cost=$(echo "$total_gb * 0.15 * 2.0" | bc)  # ~15% pour oplog
  local total=$(echo "$snapshot_cost + $oplog_cost" | bc)

  echo "Projected Monthly Costs:"
  echo "  Snapshots: \$${snapshot_cost}"
  echo "  PITR Oplog: \$${oplog_cost}"
  echo "  Total: \$${total}"
}
```

## Conformité et Certifications

```yaml
Certifications Atlas:
  - SOC 2 Type II
  - ISO 27001
  - PCI DSS
  - HIPAA (BAA disponible)
  - FedRAMP (Government cloud)
  - GDPR compliant

Fonctionnalités Compliance:
  encryption_at_rest:
    algorithm: "AES-256"
    key_management: "AWS KMS / Azure Key Vault / GCP KMS"
    customer_managed: true

  encryption_in_transit:
    tls_version: "TLS 1.2+"
    certificates: "Automatique"

  audit_logs:
    available: true
    retention: "Configurable"
    export: "Vers SIEM"

  access_control:
    rbac: "Granulaire"
    mfa: "Obligatoire (option)"
    ip_whitelist: "Oui"

  data_residency:
    region_selection: true
    no_cross_border: "Option disponible"
```

## Comparaison : Atlas vs Self-Hosted

```
┌────────────────────┬──────────────────┬──────────────────┐
│     Critère        │  Atlas Backup    │  Self-Hosted     │
├────────────────────┼──────────────────┼──────────────────┤
│ Setup Time         │ < 5 minutes      │ Jours/Semaines   │
│ Maintenance        │ Zéro             │ Continue         │
│ PITR               │ ✓ Natif          │ ✗ Custom needed  │
│ Cross-Region       │ ✓ Automatique    │ ✗ Manual         │
│ Compliance         │ ✓ Certifié       │ ✗ DIY            │
│ Encryption         │ ✓ Automatique    │ ✗ Configure      │
│ Queryable Backups  │ ✓ Oui            │ ✗ Non            │
│ Cost (1TB)         │ ~$500-800/month  │ Variable         │
│ Expertise Required │ Minimal          │ Élevée           │
│ Recovery Testing   │ ✓ Facile         │ ✗ Complexe       │
│ Automation         │ ✓ Intégré        │ ✗ Build custom   │
└────────────────────┴──────────────────┴──────────────────┘
```

## Bonnes Pratiques

### Checklist de Configuration

```markdown
### Configuration Initiale
- [ ] Activer Cloud Backups sur tous les clusters prod
- [ ] Configurer snapshot frequency (6h recommandé)
- [ ] Activer PITR window (7 jours pour prod)
- [ ] Configurer cross-region copies vers DR site
- [ ] Définir politiques de rétention appropriées
- [ ] Configurer alertes (échecs, delays)
- [ ] Documenter procédures de restauration
- [ ] Établir RTO/RPO par environnement

### Maintenance Régulière
- [ ] Tester restauration PITR mensuellement
- [ ] Valider snapshots avec queryable backup
- [ ] Réviser politiques de rétention trimestriellement
- [ ] Auditer coûts de stockage backup
- [ ] Vérifier alertes fonctionnent (test)
- [ ] Documenter toute restauration effectuée
- [ ] Former équipe sur procédures DR

### DR Readiness
- [ ] Procédures DR documentées et à jour
- [ ] DR drill exécuté annuellement
- [ ] Temps de restauration mesurés et validés
- [ ] Contacts DR à jour (oncall, management)
- [ ] Accès backup vérifié (credentials, permissions)
- [ ] Communication plan établi
```

### Recommandations par Environnement

```yaml
Production Critique:
  snapshots: Toutes les 6 heures
  pitr_window: 7 jours
  daily_retention: 7 jours
  weekly_retention: 4 semaines
  monthly_retention: 12 mois
  yearly_retention: 3 ans
  cross_region: 2+ régions
  testing: Mensuel

Production Standard:
  snapshots: Quotidien
  pitr_window: 2 jours
  daily_retention: 7 jours
  weekly_retention: 4 semaines
  monthly_retention: 6 mois
  cross_region: 1 région
  testing: Trimestriel

Staging/QA:
  snapshots: Quotidien
  pitr_window: 1 jour
  daily_retention: 3 jours
  cross_region: Non
  testing: Ad-hoc

Development:
  snapshots: Quotidien (optionnel)
  pitr_window: 0 (désactivé)
  daily_retention: 1-2 jours
  cross_region: Non
  testing: Jamais nécessaire
```

## Conclusion

MongoDB Atlas Backup représente une solution de sauvegarde cloud-native mature qui élimine la complexité opérationnelle tout en offrant des capacités avancées comme PITR, queryable backups et réplication géographique automatique.

**Avantages clés** :
1. **Zéro maintenance** - Entièrement géré par Atlas
2. **PITR natif** - Restauration à la seconde près
3. **Compliance** - Certifications incluses
4. **Rapidité** - Setup en minutes, pas en jours
5. **Fiabilité** - SLA garanti par MongoDB

**Quand choisir Atlas Backup** :
- Applications cloud-native
- Besoin de PITR sans complexité
- Équipe limitée en expertise backup
- Exigences de compliance
- Budget prévisible préféré

**Quand considérer self-hosted** :
- On-premise requis (réglementaire)
- Contrôle total nécessaire
- Très grands volumes (>10TB) avec budget serré
- Architecture hybride complexe

Pour la majorité des cas d'usage cloud, Atlas Backup offre le meilleur équilibre entre fonctionnalités, fiabilité et coût total de possession.

---


⏭️ [Point-in-Time Recovery](/12-sauvegarde-restauration/07-point-in-time-recovery.md)
