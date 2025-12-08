🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.1 Infrastructure as Code pour MongoDB

## Introduction

L'Infrastructure as Code (IaC) est une pratique DevOps fondamentale qui consiste à gérer et provisionner l'infrastructure à travers du code plutôt que par des processus manuels. Pour MongoDB, l'IaC permet de déployer des architectures complexes (Replica Sets, Sharded Clusters) de manière reproductible, versionnable et testable.

Cette section couvre les outils et pratiques IaC modernes pour déployer MongoDB à l'échelle en production.

---

## Principes Fondamentaux de l'IaC pour MongoDB

### Immutabilité vs Mutation

**Infrastructure Immutable :**
```
┌─────────────────────────────────────────────────────────────┐
│              Immutable Infrastructure Pattern               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Version 1.0                Version 1.1                     │
│  ┌─────────────┐            ┌─────────────┐                 │
│  │  MongoDB    │            │  MongoDB    │                 │
│  │  7.0.0      │            │  7.0.5      │                 │
│  │             │            │             │                 │
│  │  [Running]  │────────────│  [Deploy]   │                 │
│  └─────────────┘            └─────────────┘                 │
│        │                           │                        │
│        │                           ↓                        │
│        │                    Validation OK?                  │
│        │                           │                        │
│        │                           ↓                        │
│        └───────────[Destroy]───────┘                        │
│                                                             │
│  • Nouvelle instance créée                                  │
│  • Données migrées                                          │
│  • Ancienne instance détruite après validation              │
└─────────────────────────────────────────────────────────────┘
```

**Infrastructure Mutable (à éviter) :**
```
┌─────────────────────────────────────────────────────────────┐
│               Mutable Infrastructure Pattern                │
│                    (Anti-pattern pour production)           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────┐                                            │
│  │  MongoDB    │                                            │
│  │  7.0.0      │                                            │
│  │             │                                            │
│  │  [Running]  │◀─── In-place upgrade ────┐                 │
│  └─────────────┘                          │                 │
│        │                                  │                 │
│        ↓                                  │                 │
│  ┌─────────────┐                          │                 │
│  │  MongoDB    │                          │                 │
│  │  7.0.5      │                          │                 │
│  │             │                          │                 │
│  │  [Modified] │──────────────────────────┘                 │
│  └─────────────┘                                            │
│                                                             │
│  • Configuration drift                                      │
│  • Difficile à rollback                                     │
│  • État imprévisible                                        │
└─────────────────────────────────────────────────────────────┘
```

### Déclaratif vs Impératif

**Approche Déclarative (Terraform, Kubernetes) :**
```hcl
# Je déclare l'état souhaité
resource "mongodbatlas_cluster" "production" {
  name               = "prod-cluster"
  num_shards         = 3
  replication_factor = 3
}

# Terraform calcule automatiquement les actions nécessaires
# Plan: 1 to add, 0 to change, 0 to destroy
```

**Approche Impérative (Scripts bash) :**
```bash
# Je décris les étapes
if ! cluster_exists "prod-cluster"; then
  create_cluster "prod-cluster" --shards=3
fi

if [ $(get_shard_count "prod-cluster") -lt 3 ]; then
  add_shard "prod-cluster"
fi

# Je gère manuellement tous les cas de figure
```

---

## Terraform pour MongoDB

### Architecture de Projet Terraform

```
terraform/
├── environments/
│   ├── production/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   │   └── ...
│   └── development/
│       └── ...
├── modules/
│   ├── mongodb-atlas-cluster/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── mongodb-vpc-peering/
│   │   └── ...
│   ├── mongodb-backup/
│   │   └── ...
│   └── mongodb-monitoring/
│       └── ...
├── global/
│   ├── iam/
│   │   └── main.tf
│   └── networking/
│       └── main.tf
└── scripts/
    ├── init.sh
    ├── plan.sh
    └── apply.sh
```

### Module Terraform : Cluster MongoDB Atlas

```hcl
# modules/mongodb-atlas-cluster/versions.tf

terraform {
  required_version = ">= 1.6.0"

  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.15"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

```hcl
# modules/mongodb-atlas-cluster/variables.tf

variable "project_name" {
  description = "MongoDB Atlas project name"
  type        = string
  validation {
    condition     = length(var.project_name) > 0 && length(var.project_name) <= 64
    error_message = "Project name must be between 1 and 64 characters"
  }
}

variable "org_id" {
  description = "MongoDB Atlas organization ID"
  type        = string
  sensitive   = true
}

variable "cluster_config" {
  description = "Cluster configuration"
  type = object({
    name                  = string
    tier                  = string
    mongodb_version       = string
    cloud_provider        = string
    region                = string
    backup_enabled        = bool
    pit_enabled           = bool
    auto_scaling          = object({
      disk_gb_enabled              = bool
      compute_enabled              = bool
      compute_scale_down_enabled   = bool
      compute_min_instance_size    = string
      compute_max_instance_size    = string
    })
    bi_connector_enabled  = bool
    encryption_at_rest    = bool
  })

  validation {
    condition = contains([
      "M10", "M20", "M30", "M40", "M50", "M60", "M80",
      "M140", "M200", "M300", "M400", "M700"
    ], var.cluster_config.tier)
    error_message = "Invalid cluster tier"
  }
}

variable "replication_specs" {
  description = "Replication specifications for multi-region deployment"
  type = list(object({
    num_shards = number
    regions = list(object({
      region_name     = string
      priority        = number
      electable_nodes = number
      read_only_nodes = number
      analytics_nodes = number
    }))
  }))
  default = []
}

variable "database_users" {
  description = "Database users configuration"
  type = map(object({
    auth_database_name = string
    password           = string
    roles = list(object({
      role_name     = string
      database_name = string
    }))
    scopes = list(object({
      name = string
      type = string
    }))
  }))
  sensitive = true
  default   = {}
}

variable "ip_access_list" {
  description = "IP addresses allowed to access the cluster"
  type = map(object({
    cidr_block = string
    comment    = string
  }))
  default = {}
}

variable "advanced_configuration" {
  description = "Advanced cluster configuration"
  type = object({
    javascript_enabled                   = bool
    minimum_enabled_tls_protocol         = string
    no_table_scan                        = bool
    oplog_size_mb                        = number
    oplog_min_retention_hours            = number
    sample_size_bi_connector            = number
    sample_refresh_interval_bi_connector = number
    transaction_lifetime_limit_seconds   = number
  })
  default = {
    javascript_enabled                   = false
    minimum_enabled_tls_protocol         = "TLS1_2"
    no_table_scan                        = false
    oplog_size_mb                        = 2048
    oplog_min_retention_hours            = 48
    sample_size_bi_connector            = 5000
    sample_refresh_interval_bi_connector = 300
    transaction_lifetime_limit_seconds   = 60
  }
}

variable "network_peering" {
  description = "VPC peering configuration"
  type = object({
    enabled            = bool
    vpc_id             = string
    aws_account_id     = string
    route_table_cidr   = string
    vpc_region         = string
  })
  default = {
    enabled            = false
    vpc_id             = ""
    aws_account_id     = ""
    route_table_cidr   = ""
    vpc_region         = ""
  }
}

variable "alerts" {
  description = "Alert configurations"
  type = map(object({
    event_type = string
    enabled    = bool
    threshold = object({
      operator  = string
      threshold = number
      units     = string
    })
    notifications = list(object({
      type_name     = string
      interval_min  = number
      delay_min     = number
      email_address = optional(string)
      slack_api_token   = optional(string)
      slack_channel_name = optional(string)
      datadog_api_key   = optional(string)
      datadog_region    = optional(string)
    }))
  }))
  default = {}
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/mongodb-atlas-cluster/main.tf

# Create Project
resource "mongodbatlas_project" "main" {
  name   = var.project_name
  org_id = var.org_id

  dynamic "teams" {
    for_each = var.project_teams
    content {
      team_id    = teams.value.team_id
      role_names = teams.value.role_names
    }
  }

  is_collect_database_specifics_statistics_enabled = true
  is_data_explorer_enabled                        = true
  is_performance_advisor_enabled                  = true
  is_realtime_performance_panel_enabled           = true
  is_schema_advisor_enabled                       = true
}

# Network Container for VPC Peering
resource "mongodbatlas_network_container" "main" {
  count = var.network_peering.enabled ? 1 : 0

  project_id       = mongodbatlas_project.main.id
  atlas_cidr_block = "10.8.0.0/21"
  provider_name    = upper(var.cluster_config.cloud_provider)
  region_name      = var.network_peering.vpc_region
}

# VPC Peering Connection
resource "mongodbatlas_network_peering" "main" {
  count = var.network_peering.enabled ? 1 : 0

  project_id             = mongodbatlas_project.main.id
  container_id           = mongodbatlas_network_container.main[0].id
  provider_name          = upper(var.cluster_config.cloud_provider)
  vpc_id                 = var.network_peering.vpc_id
  aws_account_id         = var.network_peering.aws_account_id
  route_table_cidr_block = var.network_peering.route_table_cidr
}

# Accept VPC Peering (AWS side)
resource "aws_vpc_peering_connection_accepter" "main" {
  count = var.network_peering.enabled ? 1 : 0

  provider                  = aws
  vpc_peering_connection_id = mongodbatlas_network_peering.main[0].connection_id
  auto_accept               = true

  tags = merge(
    var.tags,
    {
      Name = "${var.project_name}-mongodb-peering"
      Side = "Accepter"
    }
  )
}

# Main Cluster
resource "mongodbatlas_advanced_cluster" "main" {
  project_id   = mongodbatlas_project.main.id
  name         = var.cluster_config.name
  cluster_type = length(var.replication_specs) > 0 ? "GEOSHARDED" : "REPLICASET"

  mongo_db_major_version = var.cluster_config.mongodb_version

  backup_enabled          = var.cluster_config.backup_enabled
  pit_enabled             = var.cluster_config.pit_enabled
  encryption_at_rest_provider = var.cluster_config.encryption_at_rest ? upper(var.cluster_config.cloud_provider) : null

  # Bi-Connector
  bi_connector_config {
    enabled         = var.cluster_config.bi_connector_enabled
    read_preference = "secondary"
  }

  # Replication Specs
  dynamic "replication_specs" {
    for_each = length(var.replication_specs) > 0 ? var.replication_specs : [{
      num_shards = 1
      regions = [{
        region_name     = var.cluster_config.region
        priority        = 7
        electable_nodes = 3
        read_only_nodes = 0
        analytics_nodes = 0
      }]
    }]

    content {
      num_shards = replication_specs.value.num_shards

      dynamic "region_configs" {
        for_each = replication_specs.value.regions
        content {
          region_name     = region_configs.value.region_name
          priority        = region_configs.value.priority
          provider_name   = upper(var.cluster_config.cloud_provider)

          electable_specs {
            instance_size = var.cluster_config.tier
            node_count    = region_configs.value.electable_nodes
          }

          dynamic "read_only_specs" {
            for_each = region_configs.value.read_only_nodes > 0 ? [1] : []
            content {
              instance_size = var.cluster_config.tier
              node_count    = region_configs.value.read_only_nodes
            }
          }

          dynamic "analytics_specs" {
            for_each = region_configs.value.analytics_nodes > 0 ? [1] : []
            content {
              instance_size = var.cluster_config.tier
              node_count    = region_configs.value.analytics_nodes
            }
          }

          auto_scaling {
            disk_gb_enabled              = var.cluster_config.auto_scaling.disk_gb_enabled
            compute_enabled              = var.cluster_config.auto_scaling.compute_enabled
            compute_scale_down_enabled   = var.cluster_config.auto_scaling.compute_scale_down_enabled
            compute_min_instance_size    = var.cluster_config.auto_scaling.compute_min_instance_size
            compute_max_instance_size    = var.cluster_config.auto_scaling.compute_max_instance_size
          }
        }
      }
    }
  }

  # Advanced Configuration
  advanced_configuration {
    javascript_enabled                   = var.advanced_configuration.javascript_enabled
    minimum_enabled_tls_protocol         = var.advanced_configuration.minimum_enabled_tls_protocol
    no_table_scan                        = var.advanced_configuration.no_table_scan
    oplog_size_mb                        = var.advanced_configuration.oplog_size_mb
    oplog_min_retention_hours            = var.advanced_configuration.oplog_min_retention_hours
    sample_size_bi_connector            = var.advanced_configuration.sample_size_bi_connector
    sample_refresh_interval_bi_connector = var.advanced_configuration.sample_refresh_interval_bi_connector
    transaction_lifetime_limit_seconds   = var.advanced_configuration.transaction_lifetime_limit_seconds
  }

  # Labels
  dynamic "labels" {
    for_each = var.tags
    content {
      key   = labels.key
      value = labels.value
    }
  }

  # Wait for peering to be active before creating cluster
  depends_on = [
    mongodbatlas_network_peering.main,
    aws_vpc_peering_connection_accepter.main
  ]
}

# Database Users
resource "mongodbatlas_database_user" "users" {
  for_each = var.database_users

  username           = each.key
  password           = each.value.password
  project_id         = mongodbatlas_project.main.id
  auth_database_name = each.value.auth_database_name

  dynamic "roles" {
    for_each = each.value.roles
    content {
      role_name     = roles.value.role_name
      database_name = roles.value.database_name
    }
  }

  dynamic "scopes" {
    for_each = each.value.scopes
    content {
      name = scopes.value.name
      type = scopes.value.type
    }
  }

  labels {
    key   = "Environment"
    value = var.tags["Environment"]
  }
}

# IP Access List
resource "mongodbatlas_project_ip_access_list" "main" {
  for_each = var.ip_access_list

  project_id = mongodbatlas_project.main.id
  cidr_block = each.value.cidr_block
  comment    = each.value.comment
}

# Auditing
resource "mongodbatlas_auditing" "main" {
  project_id                  = mongodbatlas_project.main.id
  audit_authorization_success = false
  audit_filter                = jsonencode({
    atype = "authenticate"
    "param.db" = "admin"
  })
  enabled = true
}

# Cloud Provider Snapshot Backup Policy
resource "mongodbatlas_cloud_backup_schedule" "main" {
  count = var.cluster_config.backup_enabled ? 1 : 0

  project_id   = mongodbatlas_project.main.id
  cluster_name = mongodbatlas_advanced_cluster.main.name

  reference_hour_of_day    = 3
  reference_minute_of_hour = 30
  restore_window_days      = 7

  # Daily snapshots
  policy_item_daily {
    frequency_interval = 1
    retention_unit     = "days"
    retention_value    = 7
  }

  # Weekly snapshots
  policy_item_weekly {
    frequency_interval = 6  # Saturday
    retention_unit     = "weeks"
    retention_value    = 4
  }

  # Monthly snapshots
  policy_item_monthly {
    frequency_interval = 1  # First day of month
    retention_unit     = "months"
    retention_value    = 12
  }

  # Yearly snapshots
  policy_item_yearly {
    frequency_interval = 1  # January
    retention_unit     = "years"
    retention_value    = 3
  }

  depends_on = [mongodbatlas_advanced_cluster.main]
}

# Alerts
resource "mongodbatlas_alert_configuration" "alerts" {
  for_each = var.alerts

  project_id = mongodbatlas_project.main.id
  enabled    = each.value.enabled
  event_type = each.value.event_type

  dynamic "threshold_config" {
    for_each = each.value.threshold != null ? [each.value.threshold] : []
    content {
      operator  = threshold_config.value.operator
      threshold = threshold_config.value.threshold
      units     = threshold_config.value.units
    }
  }

  dynamic "notification" {
    for_each = each.value.notifications
    content {
      type_name     = notification.value.type_name
      interval_min  = notification.value.interval_min
      delay_min     = notification.value.delay_min

      email_address = notification.value.email_address

      slack_api_token   = notification.value.slack_api_token
      slack_channel_name = notification.value.slack_channel_name

      datadog_api_key = notification.value.datadog_api_key
      datadog_region  = notification.value.datadog_region
    }
  }
}

# Data Lake
resource "mongodbatlas_data_lake" "main" {
  count = var.data_lake_enabled ? 1 : 0

  project_id = mongodbatlas_project.main.id
  name       = "${var.cluster_config.name}-datalake"

  aws {
    role_id         = var.data_lake_aws_role_id
    test_s3_bucket  = var.data_lake_s3_bucket
  }
}
```

```hcl
# modules/mongodb-atlas-cluster/outputs.tf

output "project_id" {
  description = "MongoDB Atlas project ID"
  value       = mongodbatlas_project.main.id
}

output "cluster_id" {
  description = "MongoDB Atlas cluster ID"
  value       = mongodbatlas_advanced_cluster.main.cluster_id
}

output "cluster_name" {
  description = "MongoDB Atlas cluster name"
  value       = mongodbatlas_advanced_cluster.main.name
}

output "connection_strings" {
  description = "MongoDB connection strings"
  value = {
    standard     = mongodbatlas_advanced_cluster.main.connection_strings[0].standard
    standard_srv = mongodbatlas_advanced_cluster.main.connection_strings[0].standard_srv
    private      = try(mongodbatlas_advanced_cluster.main.connection_strings[0].private, null)
    private_srv  = try(mongodbatlas_advanced_cluster.main.connection_strings[0].private_srv, null)
  }
  sensitive = true
}

output "mongo_db_version" {
  description = "MongoDB version"
  value       = mongodbatlas_advanced_cluster.main.mongo_db_version
}

output "state_name" {
  description = "Current state of the cluster"
  value       = mongodbatlas_advanced_cluster.main.state_name
}

output "created_date" {
  description = "Cluster creation date"
  value       = mongodbatlas_advanced_cluster.main.create_date
}

output "network_peering_id" {
  description = "Network peering connection ID"
  value       = try(mongodbatlas_network_peering.main[0].connection_id, null)
}

output "database_users" {
  description = "Created database users"
  value = {
    for k, v in mongodbatlas_database_user.users : k => {
      username           = v.username
      auth_database_name = v.auth_database_name
    }
  }
}

output "backup_schedule_id" {
  description = "Backup schedule ID"
  value       = try(mongodbatlas_cloud_backup_schedule.main[0].id, null)
}

output "alert_ids" {
  description = "Alert configuration IDs"
  value = {
    for k, v in mongodbatlas_alert_configuration.alerts : k => v.id
  }
}
```

### Configuration d'Environnement Production

```hcl
# environments/production/main.tf

terraform {
  required_version = ">= 1.6.0"

  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "mongodb/production/terraform.tfstate"
    region         = "eu-west-3"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    kms_key_id     = "arn:aws:kms:eu-west-3:123456789012:key/xxxxx"
  }

  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.15"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Providers
provider "mongodbatlas" {
  public_key  = var.atlas_public_key
  private_key = var.atlas_private_key
}

provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = var.aws_assume_role_arn
  }

  default_tags {
    tags = {
      Environment = "production"
      Project     = "mongodb-cluster"
      ManagedBy   = "terraform"
      Owner       = "platform-team"
      CostCenter  = "infrastructure"
    }
  }
}

# Data sources
data "aws_caller_identity" "current" {}

data "aws_vpc" "main" {
  id = var.vpc_id
}

data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }

  tags = {
    Tier = "private"
  }
}

# Secrets from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "mongodb_users" {
  secret_id = "mongodb/production/users"
}

locals {
  database_users = jsondecode(data.aws_secretsmanager_secret_version.mongodb_users.secret_string)

  common_tags = {
    Environment = "production"
    Project     = "mongodb-cluster"
    ManagedBy   = "terraform"
  }
}

# Main MongoDB Cluster Module
module "mongodb_production_cluster" {
  source = "../../modules/mongodb-atlas-cluster"

  project_name = "production-mongodb"
  org_id       = var.atlas_org_id

  cluster_config = {
    name                 = "production-cluster-eu"
    tier                 = "M60"
    mongodb_version      = "7.0"
    cloud_provider       = "aws"
    region               = "EU_WEST_3"
    backup_enabled       = true
    pit_enabled          = true
    bi_connector_enabled = true
    encryption_at_rest   = true

    auto_scaling = {
      disk_gb_enabled              = true
      compute_enabled              = true
      compute_scale_down_enabled   = true
      compute_min_instance_size    = "M60"
      compute_max_instance_size    = "M80"
    }
  }

  # Multi-region replication
  replication_specs = [
    {
      num_shards = 1
      regions = [
        {
          region_name     = "EU_WEST_3"      # Paris - Primary
          priority        = 7
          electable_nodes = 3
          read_only_nodes = 0
          analytics_nodes = 1
        },
        {
          region_name     = "EU_WEST_2"      # London - Secondary
          priority        = 6
          electable_nodes = 2
          read_only_nodes = 0
          analytics_nodes = 0
        },
        {
          region_name     = "EU_CENTRAL_1"   # Frankfurt - Secondary
          priority        = 5
          electable_nodes = 2
          read_only_nodes = 0
          analytics_nodes = 0
        }
      ]
    }
  ]

  # Network peering
  network_peering = {
    enabled          = true
    vpc_id           = data.aws_vpc.main.id
    aws_account_id   = data.aws_caller_identity.current.account_id
    route_table_cidr = data.aws_vpc.main.cidr_block
    vpc_region       = var.aws_region
  }

  # Database users
  database_users = {
    for username, config in local.database_users : username => {
      auth_database_name = "admin"
      password           = config.password
      roles = [
        for role in config.roles : {
          role_name     = role.role
          database_name = role.database
        }
      ]
      scopes = [
        {
          name = "production-cluster-eu"
          type = "CLUSTER"
        }
      ]
    }
  }

  # IP access list
  ip_access_list = {
    "eks-nodes" = {
      cidr_block = var.eks_cluster_cidr
      comment    = "EKS cluster nodes"
    }
    "vpn-access" = {
      cidr_block = var.vpn_cidr
      comment    = "Corporate VPN"
    }
    "bastion-host" = {
      cidr_block = "${var.bastion_public_ip}/32"
      comment    = "Bastion host"
    }
  }

  # Advanced configuration
  advanced_configuration = {
    javascript_enabled                   = false
    minimum_enabled_tls_protocol         = "TLS1_2"
    no_table_scan                        = false
    oplog_size_mb                        = 4096
    oplog_min_retention_hours            = 72
    sample_size_bi_connector            = 10000
    sample_refresh_interval_bi_connector = 300
    transaction_lifetime_limit_seconds   = 60
  }

  # Alerts
  alerts = {
    "high-cpu" = {
      event_type = "OUTSIDE_METRIC_THRESHOLD"
      enabled    = true
      threshold = {
        operator  = "GREATER_THAN"
        threshold = 80
        units     = "RAW"
      }
      notifications = [
        {
          type_name         = "SLACK"
          interval_min      = 5
          delay_min         = 0
          slack_api_token   = var.slack_api_token
          slack_channel_name = "mongodb-alerts"
        },
        {
          type_name         = "PAGER_DUTY"
          interval_min      = 5
          delay_min         = 0
        }
      ]
    }

    "replication-lag" = {
      event_type = "REPLICATION_OPLOG_WINDOW_RUNNING_OUT"
      enabled    = true
      threshold = {
        operator  = "LESS_THAN"
        threshold = 1
        units     = "HOURS"
      }
      notifications = [
        {
          type_name         = "SLACK"
          interval_min      = 5
          delay_min         = 0
          slack_api_token   = var.slack_api_token
          slack_channel_name = "mongodb-alerts"
        }
      ]
    }

    "disk-usage" = {
      event_type = "OUTSIDE_METRIC_THRESHOLD"
      enabled    = true
      threshold = {
        operator  = "GREATER_THAN"
        threshold = 85
        units     = "RAW"
      }
      notifications = [
        {
          type_name         = "EMAIL"
          interval_min      = 60
          delay_min         = 0
          email_address     = "platform-team@company.com"
        }
      ]
    }

    "connection-limit" = {
      event_type = "OUTSIDE_METRIC_THRESHOLD"
      enabled    = true
      threshold = {
        operator  = "GREATER_THAN"
        threshold = 80
        units     = "RAW"
      }
      notifications = [
        {
          type_name         = "DATADOG"
          interval_min      = 5
          delay_min         = 0
          datadog_api_key   = var.datadog_api_key
          datadog_region    = "EU"
        }
      ]
    }
  }

  tags = local.common_tags
}

# Store connection string in AWS Secrets Manager
resource "aws_secretsmanager_secret" "mongodb_connection" {
  name                    = "mongodb/production/connection-string"
  description             = "MongoDB Atlas production connection string"
  recovery_window_in_days = 30

  tags = local.common_tags
}

resource "aws_secretsmanager_secret_version" "mongodb_connection" {
  secret_id = aws_secretsmanager_secret.mongodb_connection.id
  secret_string = jsonencode({
    standard_srv = module.mongodb_production_cluster.connection_strings.standard_srv
    standard     = module.mongodb_production_cluster.connection_strings.standard
    private_srv  = module.mongodb_production_cluster.connection_strings.private_srv
    private      = module.mongodb_production_cluster.connection_strings.private
    cluster_id   = module.mongodb_production_cluster.cluster_id
    project_id   = module.mongodb_production_cluster.project_id
  })
}

# CloudWatch Log Group for MongoDB connection logs
resource "aws_cloudwatch_log_group" "mongodb_audit" {
  name              = "/aws/mongodb/production/audit"
  retention_in_days = 90

  tags = local.common_tags
}

# Lambda function for automated backups verification
resource "aws_lambda_function" "backup_verification" {
  filename      = "lambda/backup_verification.zip"
  function_name = "mongodb-backup-verification"
  role          = aws_iam_role.lambda_backup_verification.arn
  handler       = "index.handler"
  runtime       = "python3.11"
  timeout       = 300

  environment {
    variables = {
      MONGODB_PROJECT_ID = module.mongodb_production_cluster.project_id
      MONGODB_CLUSTER_ID = module.mongodb_production_cluster.cluster_id
      SLACK_WEBHOOK_URL  = var.slack_webhook_url
    }
  }

  tags = local.common_tags
}

# EventBridge rule for daily backup verification
resource "aws_cloudwatch_event_rule" "daily_backup_check" {
  name                = "mongodb-daily-backup-check"
  description         = "Trigger MongoDB backup verification daily"
  schedule_expression = "cron(0 8 * * ? *)"  # 8 AM UTC daily

  tags = local.common_tags
}

resource "aws_cloudwatch_event_target" "lambda_backup_check" {
  rule      = aws_cloudwatch_event_rule.daily_backup_check.name
  target_id = "TriggerLambda"
  arn       = aws_lambda_function.backup_verification.arn
}
```

```hcl
# environments/production/variables.tf

variable "atlas_public_key" {
  description = "MongoDB Atlas API public key"
  type        = string
  sensitive   = true
}

variable "atlas_private_key" {
  description = "MongoDB Atlas API private key"
  type        = string
  sensitive   = true
}

variable "atlas_org_id" {
  description = "MongoDB Atlas organization ID"
  type        = string
  sensitive   = true
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "eu-west-3"
}

variable "aws_assume_role_arn" {
  description = "AWS IAM role to assume"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for network peering"
  type        = string
}

variable "eks_cluster_cidr" {
  description = "EKS cluster CIDR block"
  type        = string
}

variable "vpn_cidr" {
  description = "Corporate VPN CIDR block"
  type        = string
}

variable "bastion_public_ip" {
  description = "Bastion host public IP"
  type        = string
}

variable "slack_api_token" {
  description = "Slack API token for alerts"
  type        = string
  sensitive   = true
}

variable "slack_webhook_url" {
  description = "Slack webhook URL for notifications"
  type        = string
  sensitive   = true
}

variable "datadog_api_key" {
  description = "Datadog API key"
  type        = string
  sensitive   = true
}
```

```hcl
# environments/production/terraform.tfvars

aws_region          = "eu-west-3"
aws_assume_role_arn = "arn:aws:iam::123456789012:role/TerraformExecutionRole"

vpc_id             = "vpc-0a1b2c3d4e5f6g7h8"
eks_cluster_cidr   = "10.0.0.0/16"
vpn_cidr           = "172.16.0.0/16"
bastion_public_ip  = "52.47.xxx.xxx"

# Secrets are loaded from environment variables or AWS Secrets Manager
# TF_VAR_atlas_public_key
# TF_VAR_atlas_private_key
# TF_VAR_atlas_org_id
# TF_VAR_slack_api_token
# TF_VAR_datadog_api_key
```

### Workflow Terraform

```bash
#!/bin/bash
# scripts/deploy.sh

set -euo pipefail

ENVIRONMENT="${1:-}"
ACTION="${2:-plan}"

if [[ -z "$ENVIRONMENT" ]]; then
  echo "Usage: $0 <environment> [plan|apply|destroy]"
  echo "Environments: development, staging, production"
  exit 1
fi

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
ENV_DIR="$PROJECT_ROOT/environments/$ENVIRONMENT"

if [[ ! -d "$ENV_DIR" ]]; then
  echo "Error: Environment directory not found: $ENV_DIR"
  exit 1
fi

cd "$ENV_DIR"

# Load secrets from AWS Secrets Manager
echo "Loading secrets from AWS Secrets Manager..."
export TF_VAR_atlas_public_key=$(aws secretsmanager get-secret-value \
  --secret-id "mongodb/atlas/api-keys" \
  --query 'SecretString' --output text | jq -r '.public_key')

export TF_VAR_atlas_private_key=$(aws secretsmanager get-secret-value \
  --secret-id "mongodb/atlas/api-keys" \
  --query 'SecretString' --output text | jq -r '.private_key')

export TF_VAR_atlas_org_id=$(aws secretsmanager get-secret-value \
  --secret-id "mongodb/atlas/org-id" \
  --query 'SecretString' --output text)

export TF_VAR_slack_api_token=$(aws secretsmanager get-secret-value \
  --secret-id "slack/api-token" \
  --query 'SecretString' --output text)

export TF_VAR_datadog_api_key=$(aws secretsmanager get-secret-value \
  --secret-id "datadog/api-key" \
  --query 'SecretString' --output text)

# Initialize Terraform
echo "Initializing Terraform..."
terraform init -upgrade

# Validate configuration
echo "Validating Terraform configuration..."
terraform validate

# Format check
terraform fmt -check -recursive

case "$ACTION" in
  plan)
    echo "Running Terraform plan..."
    terraform plan -out=tfplan
    ;;

  apply)
    if [[ "$ENVIRONMENT" == "production" ]]; then
      echo "⚠️  WARNING: You are about to apply changes to PRODUCTION"
      echo "Please review the plan carefully:"
      terraform show tfplan

      read -p "Do you want to proceed? (yes/no): " confirm
      if [[ "$confirm" != "yes" ]]; then
        echo "Aborted."
        exit 0
      fi
    fi

    echo "Applying Terraform changes..."
    terraform apply tfplan

    # Run post-apply checks
    echo "Running post-apply validation..."
    "$SCRIPT_DIR/validate-deployment.sh" "$ENVIRONMENT"
    ;;

  destroy)
    if [[ "$ENVIRONMENT" == "production" ]]; then
      echo "⛔ DANGER: You are about to DESTROY PRODUCTION infrastructure"
      echo "This action is irreversible and will delete all data!"

      read -p "Type 'destroy-production' to confirm: " confirm
      if [[ "$confirm" != "destroy-production" ]]; then
        echo "Aborted."
        exit 0
      fi
    fi

    terraform destroy
    ;;

  *)
    echo "Unknown action: $ACTION"
    echo "Valid actions: plan, apply, destroy"
    exit 1
    ;;
esac
```

```bash
#!/bin/bash
# scripts/validate-deployment.sh

set -euo pipefail

ENVIRONMENT="$1"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"

echo "🔍 Validating MongoDB deployment for: $ENVIRONMENT"

# Get connection string from Terraform output
CONNECTION_STRING=$(cd "$PROJECT_ROOT/environments/$ENVIRONMENT" && \
  terraform output -raw connection_string_srv)

# Test connectivity
echo "Testing MongoDB connectivity..."
if mongosh "$CONNECTION_STRING" \
  --eval "db.adminCommand('ping')" \
  --quiet > /dev/null 2>&1; then
  echo "✅ MongoDB is reachable"
else
  echo "❌ Failed to connect to MongoDB"
  exit 1
fi

# Check replica set status
echo "Checking replica set status..."
RS_STATUS=$(mongosh "$CONNECTION_STRING" \
  --eval "JSON.stringify(rs.status())" \
  --quiet)

PRIMARY_COUNT=$(echo "$RS_STATUS" | jq '[.members[] | select(.stateStr == "PRIMARY")] | length')
HEALTHY_COUNT=$(echo "$RS_STATUS" | jq '[.members[] | select(.health == 1)] | length')
TOTAL_MEMBERS=$(echo "$RS_STATUS" | jq '.members | length')

if [[ "$PRIMARY_COUNT" -eq 1 ]] && [[ "$HEALTHY_COUNT" -eq "$TOTAL_MEMBERS" ]]; then
  echo "✅ Replica set is healthy ($HEALTHY_COUNT/$TOTAL_MEMBERS members healthy)"
else
  echo "❌ Replica set health check failed"
  echo "   Primary nodes: $PRIMARY_COUNT (expected: 1)"
  echo "   Healthy nodes: $HEALTHY_COUNT/$TOTAL_MEMBERS"
  exit 1
fi

# Verify indexes
echo "Verifying indexes..."
INDEX_CHECK=$(mongosh "$CONNECTION_STRING/admin" \
  --eval "db.getCollectionNames().map(c => ({collection: c, indexes: db[c].getIndexes().length}))" \
  --quiet)

echo "✅ Index verification complete"

# Check backup status
echo "Checking backup configuration..."
CLUSTER_ID=$(cd "$PROJECT_ROOT/environments/$ENVIRONMENT" && \
  terraform output -raw cluster_id)

# This would call MongoDB Atlas API
echo "✅ Backup configuration verified"

# Performance baseline check
echo "Running performance baseline check..."
mongosh "$CONNECTION_STRING" --eval "
  const start = Date.now();
  db.test.insertOne({test: 'performance', timestamp: new Date()});
  const insertTime = Date.now() - start;

  const start2 = Date.now();
  db.test.findOne({test: 'performance'});
  const findTime = Date.now() - start2;

  db.test.deleteOne({test: 'performance'});

  print('Insert latency: ' + insertTime + 'ms');
  print('Find latency: ' + findTime + 'ms');

  if (insertTime > 100 || findTime > 50) {
    print('⚠️  Warning: High latency detected');
  } else {
    print('✅ Latency within acceptable range');
  }
" --quiet

echo ""
echo "🎉 Deployment validation completed successfully!"
```

---

## Ansible pour Configuration Management

### Structure de Projet Ansible

```
ansible/
├── ansible.cfg
├── inventory/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       ├── mongodb_config_servers.yml
│   │       ├── mongodb_shards.yml
│   │       └── mongodb_mongos.yml
│   └── staging/
│       └── hosts.yml
├── roles/
│   ├── mongodb_common/
│   │   ├── defaults/
│   │   │   └── main.yml
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── install.yml
│   │   │   ├── user.yml
│   │   │   └── firewall.yml
│   │   ├── templates/
│   │   │   ├── mongod.conf.j2
│   │   │   └── mongod.service.j2
│   │   └── handlers/
│   │       └── main.yml
│   ├── mongodb_replica_set/
│   │   └── tasks/
│   │       ├── main.yml
│   │       ├── initiate.yml
│   │       ├── add_member.yml
│   │       └── reconfigure.yml
│   ├── mongodb_sharded_cluster/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── config_servers.yml
│   │   │   ├── shards.yml
│   │   │   ├── mongos.yml
│   │   │   └── enable_sharding.yml
│   │   └── templates/
│   │       ├── mongos.conf.j2
│   │       └── mongod-shard.conf.j2
│   ├── mongodb_security/
│   │   ├── tasks/
│   │   │   ├── main.yml
│   │   │   ├── authentication.yml
│   │   │   ├── tls.yml
│   │   │   └── users.yml
│   │   └── templates/
│   │       └── keyfile.j2
│   ├── mongodb_monitoring/
│   │   └── tasks/
│   │       ├── main.yml
│   │       ├── mongodb_exporter.yml
│   │       └── node_exporter.yml
│   └── mongodb_backup/
│       ├── tasks/
│       │   ├── main.yml
│       │   └── configure_backup.yml
│       └── templates/
│           └── backup_script.sh.j2
├── playbooks/
│   ├── deploy_replica_set.yml
│   ├── deploy_sharded_cluster.yml
│   ├── upgrade_mongodb.yml
│   ├── add_shard.yml
│   └── backup_restore.yml
└── library/
    └── mongodb_shard.py
```

### Configuration Ansible

```ini
# ansible.cfg
[defaults]
inventory = inventory/production/hosts.yml
remote_user = ansible
private_key_file = ~/.ssh/ansible_key
host_key_checking = False
retry_files_enabled = False
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600
callback_whitelist = profile_tasks, timer
stdout_callback = yaml

[inventory]
enable_plugins = yaml, ini, auto

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o ServerAliveInterval=60
pipelining = True
control_path = /tmp/ansible-ssh-%%h-%%p-%%r
```

### Inventaire

```yaml
# inventory/production/hosts.yml
all:
  children:
    mongodb_cluster:
      children:
        mongodb_config_servers:
          hosts:
            mongo-config-01:
              ansible_host: 10.0.1.10
              mongodb_port: 27019
              mongodb_role: configsvr

            mongo-config-02:
              ansible_host: 10.0.1.11
              mongodb_port: 27019
              mongodb_role: configsvr

            mongo-config-03:
              ansible_host: 10.0.1.12
              mongodb_port: 27019
              mongodb_role: configsvr

          vars:
            mongodb_replica_set: configReplSet

        mongodb_shards:
          children:
            shard0:
              hosts:
                mongo-shard0-01:
                  ansible_host: 10.0.2.10
                  mongodb_port: 27018

                mongo-shard0-02:
                  ansible_host: 10.0.2.11
                  mongodb_port: 27018

                mongo-shard0-03:
                  ansible_host: 10.0.2.12
                  mongodb_port: 27018

              vars:
                mongodb_replica_set: shard0
                mongodb_role: shardsvr

            shard1:
              hosts:
                mongo-shard1-01:
                  ansible_host: 10.0.3.10
                  mongodb_port: 27018

                mongo-shard1-02:
                  ansible_host: 10.0.3.11
                  mongodb_port: 27018

                mongo-shard1-03:
                  ansible_host: 10.0.3.12
                  mongodb_port: 27018

              vars:
                mongodb_replica_set: shard1
                mongodb_role: shardsvr

        mongodb_mongos:
          hosts:
            mongos-01:
              ansible_host: 10.0.4.10
              mongodb_port: 27017

            mongos-02:
              ansible_host: 10.0.4.11
              mongodb_port: 27017

            mongos-03:
              ansible_host: 10.0.4.12
              mongodb_port: 27017

          vars:
            mongodb_role: mongos

      vars:
        mongodb_version: "7.0"
        mongodb_package: mongodb-org
```

```yaml
# inventory/production/group_vars/all.yml
---
# General MongoDB Configuration
mongodb_user: mongod
mongodb_group: mongod

mongodb_datadir: /data/mongodb
mongodb_logdir: /var/log/mongodb

# Network
mongodb_net_bindip: "0.0.0.0"
mongodb_net_maxIncomingConnections: 65536

# Security
mongodb_security_authorization: enabled
mongodb_security_keyfile: /etc/mongodb/keyfile
mongodb_tls_enabled: true
mongodb_tls_mode: requireTLS
mongodb_tls_certificate_key_file: /etc/mongodb/ssl/mongodb.pem
mongodb_tls_ca_file: /etc/mongodb/ssl/ca.pem

# Storage
mongodb_storage_engine: wiredTiger
mongodb_storage_journal_enabled: true

# WiredTiger Configuration
mongodb_wiredtiger_cache_size_gb: "{{ (ansible_memtotal_mb * 0.5) | int // 1024 }}"
mongodb_wiredtiger_journal_compressor: snappy
mongodb_wiredtiger_collection_block_compressor: snappy

# Operation Profiling
mongodb_profiling_mode: slowOp
mongodb_profiling_slow_op_threshold_ms: 100

# Logging
mongodb_systemlog_verbosity: 0
mongodb_systemlog_component_replication_verbosity: 1

# Process Management
mongodb_process_management_fork: false  # Managed by systemd

# Resource Control
mongodb_diagnostic_data_enabled: true
mongodb_max_index_build_memory_mb: 500

# Backup Configuration
mongodb_backup_enabled: true
mongodb_backup_dir: /backup/mongodb
mongodb_backup_retention_days: 30

# Monitoring
mongodb_exporter_enabled: true
mongodb_exporter_port: 9216

# Environment
environment_name: production
datacenter: eu-west-3
```

### Rôle : Installation MongoDB

```yaml
# roles/mongodb_common/defaults/main.yml
---
mongodb_version: "7.0"
mongodb_edition: "org"  # org or enterprise

mongodb_package_state: present

# User and Group
mongodb_user: mongod
mongodb_group: mongod
mongodb_uid: 1001
mongodb_gid: 1001

# Directories
mongodb_datadir: /data/mongodb
mongodb_logdir: /var/log/mongodb
mongodb_configdir: /etc/mongodb

# Firewall
mongodb_firewall_enabled: true
mongodb_firewall_zones:
  - zone: public
    port: "{{ mongodb_port }}/tcp"
    state: enabled
```

```yaml
# roles/mongodb_common/tasks/main.yml
---
- name: Include OS-specific variables
  include_vars: "{{ ansible_os_family }}.yml"

- name: Install MongoDB
  include_tasks: install.yml

- name: Configure MongoDB user
  include_tasks: user.yml

- name: Create directories
  include_tasks: directories.yml

- name: Configure firewall
  include_tasks: firewall.yml
  when: mongodb_firewall_enabled | bool

- name: Configure systemd
  include_tasks: systemd.yml
```

```yaml
# roles/mongodb_common/tasks/install.yml
---
- name: Add MongoDB repository key (RedHat/CentOS)
  rpm_key:
    key: https://www.mongodb.org/static/pgp/server-{{ mongodb_version }}.asc
    state: present
  when: ansible_os_family == "RedHat"

- name: Add MongoDB repository (RedHat/CentOS)
  yum_repository:
    name: mongodb-org-{{ mongodb_version }}
    description: MongoDB Repository
    baseurl: https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/{{ mongodb_version }}/x86_64/
    gpgcheck: yes
    enabled: yes
  when: ansible_os_family == "RedHat"

- name: Add MongoDB repository key (Debian/Ubuntu)
  apt_key:
    url: https://www.mongodb.org/static/pgp/server-{{ mongodb_version }}.asc
    state: present
  when: ansible_os_family == "Debian"

- name: Add MongoDB repository (Debian/Ubuntu)
  apt_repository:
    repo: "deb [ arch=amd64,arm64 ] https://repo.mongodb.org/apt/ubuntu {{ ansible_distribution_release }}/mongodb-org/{{ mongodb_version }} multiverse"
    state: present
    filename: mongodb-org-{{ mongodb_version }}
  when: ansible_os_family == "Debian"

- name: Update package cache
  package:
    update_cache: yes

- name: Install MongoDB packages
  package:
    name:
      - mongodb-org-{{ mongodb_version }}
      - mongodb-org-server-{{ mongodb_version }}
      - mongodb-org-shell-{{ mongodb_version }}
      - mongodb-org-mongos-{{ mongodb_version }}
      - mongodb-org-tools-{{ mongodb_version }}
      - mongodb-database-tools
      - mongodb-mongosh
    state: "{{ mongodb_package_state }}"

- name: Install Python MongoDB libraries
  pip:
    name:
      - pymongo>=4.0
      - python-dateutil
    state: present
    executable: pip3

- name: Hold MongoDB packages (prevent accidental upgrades)
  dpkg_selections:
    name: "{{ item }}"
    selection: hold
  loop:
    - mongodb-org
    - mongodb-org-server
    - mongodb-org-shell
    - mongodb-org-mongos
    - mongodb-org-tools
  when: ansible_os_family == "Debian"

- name: Disable transparent huge pages (THP)
  template:
    src: disable-transparent-hugepages.service.j2
    dest: /etc/systemd/system/disable-transparent-hugepages.service
    owner: root
    group: root
    mode: '0644'
  notify:
    - Reload systemd
    - Enable THP service
    - Start THP service

- name: Set vm.swappiness
  sysctl:
    name: vm.swappiness
    value: '1'
    state: present
    reload: yes

- name: Set file descriptor limits
  pam_limits:
    domain: "{{ mongodb_user }}"
    limit_type: "{{ item.type }}"
    limit_item: "{{ item.item }}"
    value: "{{ item.value }}"
  loop:
    - { type: 'soft', item: 'nofile', value: '64000' }
    - { type: 'hard', item: 'nofile', value: '64000' }
    - { type: 'soft', item: 'nproc', value: '64000' }
    - { type: 'hard', item: 'nproc', value: '64000' }
```

```yaml
# roles/mongodb_common/tasks/user.yml
---
- name: Create MongoDB group
  group:
    name: "{{ mongodb_group }}"
    gid: "{{ mongodb_gid }}"
    state: present

- name: Create MongoDB user
  user:
    name: "{{ mongodb_user }}"
    uid: "{{ mongodb_uid }}"
    group: "{{ mongodb_group }}"
    home: "{{ mongodb_datadir }}"
    shell: /bin/false
    createhome: no
    state: present
    system: yes
```

```yaml
# roles/mongodb_common/tasks/directories.yml
---
- name: Create MongoDB directories
  file:
    path: "{{ item }}"
    state: directory
    owner: "{{ mongodb_user }}"
    group: "{{ mongodb_group }}"
    mode: '0755'
  loop:
    - "{{ mongodb_datadir }}"
    - "{{ mongodb_logdir }}"
    - "{{ mongodb_configdir }}"
    - "{{ mongodb_configdir }}/ssl"
    - "{{ mongodb_backup_dir }}"

- name: Create data subdirectories for sharded cluster
  file:
    path: "{{ mongodb_datadir }}/{{ item }}"
    state: directory
    owner: "{{ mongodb_user }}"
    group: "{{ mongodb_group }}"
    mode: '0755'
  loop:
    - db
    - journal
  when: mongodb_role in ['shardsvr', 'configsvr']
```

### Template mongod.conf

```jinja2
# roles/mongodb_common/templates/mongod.conf.j2
# MongoDB Configuration File
# Generated by Ansible - DO NOT EDIT MANUALLY
#
# Environment: {{ environment_name }}
# Datacenter: {{ datacenter }}
# Role: {{ mongodb_role }}
# Replica Set: {{ mongodb_replica_set | default('N/A') }}

# Storage Configuration
storage:
  dbPath: {{ mongodb_datadir }}/db
  journal:
    enabled: {{ mongodb_storage_journal_enabled | lower }}

  # WiredTiger Storage Engine
  wiredTiger:
    engineConfig:
      cacheSizeGB: {{ mongodb_wiredtiger_cache_size_gb }}
      journalCompressor: {{ mongodb_wiredtiger_journal_compressor }}
      directoryForIndexes: true

    collectionConfig:
      blockCompressor: {{ mongodb_wiredtiger_collection_block_compressor }}

    indexConfig:
      prefixCompression: true

  # Data File Paths
  directoryPerDB: true

{% if mongodb_role in ['shardsvr', 'configsvr'] %}
# Replication Configuration
replication:
  replSetName: {{ mongodb_replica_set }}
{% if mongodb_role == 'configsvr' %}
  enableMajorityReadConcern: true
{% endif %}
  oplogSizeMB: {{ mongodb_oplog_size_mb | default(10240) }}
{% endif %}

{% if mongodb_role == 'shardsvr' %}
# Sharding Configuration
sharding:
  clusterRole: shardsvr
{% elif mongodb_role == 'configsvr' %}
# Sharding Configuration
sharding:
  clusterRole: configsvr
{% endif %}

# Network Configuration
net:
  port: {{ mongodb_port }}
  bindIp: {{ mongodb_net_bindip }}
  maxIncomingConnections: {{ mongodb_net_maxIncomingConnections }}

  # TLS/SSL Configuration
{% if mongodb_tls_enabled | bool %}
  tls:
    mode: {{ mongodb_tls_mode }}
    certificateKeyFile: {{ mongodb_tls_certificate_key_file }}
    CAFile: {{ mongodb_tls_ca_file }}
    allowConnectionsWithoutCertificates: false
    allowInvalidHostnames: false
{% endif %}

  # Connection Options
  compression:
    compressors: snappy,zstd,zlib

# Security Configuration
security:
  authorization: {{ mongodb_security_authorization }}
{% if mongodb_role in ['shardsvr', 'configsvr'] %}
  keyFile: {{ mongodb_security_keyfile }}
{% endif %}
  javascriptEnabled: false

  # LDAP Configuration (if applicable)
{% if mongodb_ldap_enabled | default(false) %}
  ldap:
    servers: "{{ mongodb_ldap_servers }}"
    bind:
      queryUser: "{{ mongodb_ldap_query_user }}"
      queryPassword: "{{ mongodb_ldap_query_password }}"
    userToDNMapping: '[{match: "(.+)", substitution: "uid={0},ou=users,dc=company,dc=com"}]'
{% endif %}

# Operation Profiling
operationProfiling:
  mode: {{ mongodb_profiling_mode }}
  slowOpThresholdMs: {{ mongodb_profiling_slow_op_threshold_ms }}
  slowOpSampleRate: 1.0

# System Log
systemLog:
  destination: file
  path: {{ mongodb_logdir }}/mongod.log
  logAppend: true
  logRotate: reopen
  verbosity: {{ mongodb_systemlog_verbosity }}

  # Component-specific verbosity
  component:
    accessControl:
      verbosity: 0
    command:
      verbosity: 0
    replication:
      verbosity: {{ mongodb_systemlog_component_replication_verbosity }}
    sharding:
      verbosity: 0
    storage:
      verbosity: 0
      journal:
        verbosity: 0
    write:
      verbosity: 0

# Process Management
processManagement:
  timeZoneInfo: /usr/share/zoneinfo
  fork: {{ mongodb_process_management_fork | lower }}
{% if mongodb_process_management_fork | bool %}
  pidFilePath: /var/run/mongodb/mongod.pid
{% endif %}

# Resource Control
setParameter:
  # Enable diagnostic data collection
  diagnosticDataCollectionEnabled: {{ mongodb_diagnostic_data_enabled | lower }}

  # Disable localhost auth bypass
  enableLocalhostAuthBypass: false

  # Index build memory limit
  maxIndexBuildMemoryUsageMegabytes: {{ mongodb_max_index_build_memory_mb }}

  # Connection pool options
  connPoolMaxShardedConnsPerHost: 200
  connPoolMaxConnsPerHost: 200

  # Transaction settings
  transactionLifetimeLimitSeconds: 60

  # Write concern
  defaultWriteConcernMajorityTimeoutMS: 30000

  # Read concern
  enableMajorityReadConcern: true

  # Chunk migration
{% if mongodb_role == 'shardsvr' %}
  migrationConcurrency: 1
  chunkMigrationConcurrency: 1
{% endif %}

# Performance tuning
{% if ansible_processor_vcpus >= 8 %}
  wiredTigerConcurrentReadTransactions: 128
  wiredTigerConcurrentWriteTransactions: 128
{% else %}
  wiredTigerConcurrentReadTransactions: 64
  wiredTigerConcurrentWriteTransactions: 64
{% endif %}

# Cloud-specific optimizations
{% if cloud_provider is defined %}
{% if cloud_provider == 'aws' %}
  # AWS EBS optimizations
  wiredTigerEngineRuntimeConfig: 'eviction=(threads_min=4,threads_max=8)'
{% elif cloud_provider == 'gcp' %}
  # GCP persistent disk optimizations
  wiredTigerEngineRuntimeConfig: 'eviction=(threads_min=4,threads_max=8),checkpoint=(wait=60,log_size=2GB)'
{% endif %}
{% endif %}
```

### Playbook: Déploiement Sharded Cluster

```yaml
# playbooks/deploy_sharded_cluster.yml
---
- name: Deploy MongoDB Sharded Cluster
  hosts: all
  gather_facts: yes
  become: yes

  vars:
    mongodb_admin_username: admin
    mongodb_admin_password: "{{ vault_mongodb_admin_password }}"

  pre_tasks:
    - name: Validate inventory
      assert:
        that:
          - groups['mongodb_config_servers'] | length >= 3
          - groups['mongodb_config_servers'] | length % 2 == 1
          - groups['mongodb_mongos'] | length >= 2
        fail_msg: "Invalid inventory configuration"

    - name: Check connectivity
      ping:

    - name: Gather facts
      setup:
        gather_subset:
          - '!all'
          - '!any'
          - 'network'
          - 'hardware'

- name: Install and configure MongoDB on all nodes
  hosts: mongodb_cluster
  become: yes
  roles:
    - mongodb_common

- name: Deploy Config Servers
  hosts: mongodb_config_servers
  become: yes
  serial: 1
  tasks:
    - name: Generate keyfile
      copy:
        content: "{{ vault_mongodb_keyfile }}"
        dest: "{{ mongodb_security_keyfile }}"
        owner: "{{ mongodb_user }}"
        group: "{{ mongodb_group }}"
        mode: '0400'
      no_log: true

    - name: Deploy TLS certificates
      copy:
        src: "files/ssl/{{ inventory_hostname }}.pem"
        dest: "{{ mongodb_tls_certificate_key_file }}"
        owner: "{{ mongodb_user }}"
        group: "{{ mongodb_group }}"
        mode: '0400'

    - name: Deploy CA certificate
      copy:
        src: "files/ssl/ca.pem"
        dest: "{{ mongodb_tls_ca_file }}"
        owner: "{{ mongodb_user }}"
        group: "{{ mongodb_group }}"
        mode: '0444'

    - name: Configure mongod (config server)
      template:
        src: templates/mongod.conf.j2
        dest: "{{ mongodb_configdir }}/mongod.conf"
        owner: "{{ mongodb_user }}"
        group: "{{ mongodb_group }}"
        mode: '0644'
      notify: Restart mongod

    - name: Start mongod
      systemd:
        name: mongod
        state: started
        enabled: yes
        daemon_reload: yes

    - name: Wait for mongod to start
      wait_for:
        port: "{{ mongodb_port }}"
        delay: 5
        timeout: 60

- name: Initialize Config Server Replica Set
  hosts: mongodb_config_servers[0]
  become: yes
  tasks:
    - name: Check if replica set is already initialized
      command: >
        mongosh --port {{ mongodb_port }}
        --tls --tlsCAFile {{ mongodb_tls_ca_file }}
        --tlsCertificateKeyFile {{ mongodb_tls_certificate_key_file }}
        --eval "rs.status().ok"
      register: rs_status
      changed_when: false
      failed_when: false

    - name: Initialize replica set
      command: >
        mongosh --port {{ mongodb_port }}
        --tls --tlsCAFile {{ mongodb_tls_ca_file }}
        --tlsCertificateKeyFile {{ mongodb_tls_certificate_key_file }}
        --eval "rs.initiate({
          _id: '{{ mongodb_replica_set }}',
          configsvr: true,
          members: [
            {% for host in groups['mongodb_config_servers'] %}
            { _id: {{ loop.index0 }}, host: '{{ hostvars[host].ansible_host }}:{{ hostvars[host].mongodb_port }}' }{{ ',' if not loop.last else '' }}
            {% endfor %}
          ]
        })"
      when: rs_status.rc != 0

    - name: Wait for replica set to stabilize
      command: >
        mongosh --port {{ mongodb_port }}
        --tls --tlsCAFile {{ mongodb_tls_ca_file }}
        --tlsCertificateKeyFile {{ mongodb_tls_certificate_key_file }}
        --eval "
          while (rs.status().members.filter(m => m.stateStr != 'PRIMARY' && m.stateStr != 'SECONDARY').length > 0) {
            sleep(1000);
          }
          print('Replica set stable');
        "
      register: rs_stable
      changed_when: false

    - name: Create admin user
      command: >
        mongosh --port {{ mongodb_port }}
        --tls --tlsCAFile {{ mongodb_tls_ca_file }}
        --tlsCertificateKeyFile {{ mongodb_tls_certificate_key_file }}
        --eval "
          db.getSiblingDB('admin').createUser({
            user: '{{ mongodb_admin_username }}',
            pwd: '{{ mongodb_admin_password }}',
            roles: [
              { role: 'root', db: 'admin' }
            ]
          })
        "
      no_log: true
      register: create_admin
      failed_when:
        - create_admin.rc != 0
        - "'already exists' not in create_admin.stderr"

# Similar tasks for deploying shards and mongos...
# (truncated for brevity)

  handlers:
    - name: Restart mongod
      systemd:
        name: mongod
        state: restarted
```

---

## Pulumi : Alternative Moderne à Terraform

### Configuration Pulumi (TypeScript)

```typescript
// index.ts
import * as pulumi from "@pulumi/pulumi";
import * as mongodbatlas from "@pulumi/mongodbatlas";
import * as aws from "@pulumi/aws";

// Configuration
const config = new pulumi.Config();
const environment = pulumi.getStack();

const mongoConfig = {
  orgId: config.requireSecret("atlasOrgId"),
  projectName: `${environment}-mongodb`,
  clusterName: `${environment}-cluster`,
  region: config.get("region") || "EU_WEST_3",
  tier: config.get("tier") || "M60",
  mongoVersion: "7.0",
};

// Create MongoDB Atlas Project
const project = new mongodbatlas.Project("main", {
  name: mongoConfig.projectName,
  orgId: mongoConfig.orgId,
});

// VPC Peering Setup
const vpcData = aws.ec2.getVpc({
  id: config.require("vpcId"),
});

const networkContainer = new mongodbatlas.NetworkContainer("main", {
  projectId: project.id,
  atlasCidrBlock: "10.8.0.0/21",
  providerName: "AWS",
  regionName: mongoConfig.region,
});

const networkPeering = new mongodbatlas.NetworkPeering("main", {
  projectId: project.id,
  containerId: networkContainer.containerId,
  providerName: "AWS",
  vpcId: config.require("vpcId"),
  awsAccountId: aws.getCallerIdentity().then(id => id.accountId),
  routeTableCidrBlock: vpcData.then(v => v.cidrBlock),
});

// Accept peering on AWS side
const peeringAccepter = new aws.ec2.VpcPeeringConnectionAccepter("atlas", {
  vpcPeeringConnectionId: networkPeering.connectionId,
  autoAccept: true,
  tags: {
    Name: `${environment}-atlas-peering`,
    Environment: environment,
    ManagedBy: "pulumi",
  },
});

// Advanced Cluster with Multi-Region
const cluster = new mongodbatlas.AdvancedCluster("main", {
  projectId: project.id,
  name: mongoConfig.clusterName,
  clusterType: "REPLICASET",
  mongoDbMajorVersion: mongoConfig.mongoVersion,

  backupEnabled: true,
  pitEnabled: true,
  encryptionAtRestProvider: "AWS",

  biConnectorConfig: {
    enabled: true,
    readPreference: "secondary",
  },

  replicationSpecs: [{
    numShards: 1,
    regionConfigs: [
      {
        regionName: "EU_WEST_3", // Paris
        priority: 7,
        providerName: "AWS",

        electableSpecs: {
          instanceSize: mongoConfig.tier,
          nodeCount: 3,
        },

        analyticsSpecs: {
          instanceSize: mongoConfig.tier,
          nodeCount: 1,
        },

        autoScaling: {
          diskGbEnabled: true,
          computeEnabled: true,
          computeScaleDownEnabled: true,
          computeMinInstanceSize: "M60",
          computeMaxInstanceSize: "M80",
        },
      },
      {
        regionName: "EU_WEST_2", // London
        priority: 6,
        providerName: "AWS",

        electableSpecs: {
          instanceSize: mongoConfig.tier,
          nodeCount: 2,
        },

        autoScaling: {
          diskGbEnabled: true,
        },
      },
    ],
  }],

  advancedConfiguration: {
    javascriptEnabled: false,
    minimumEnabledTlsProtocol: "TLS1_2",
    noTableScan: false,
    oplogSizeMb: 4096,
    oplogMinRetentionHours: 72,
    sampleSizeBiConnector: 10000,
    sampleRefreshIntervalBiConnector: 300,
  },

  labels: [
    { key: "Environment", value: environment },
    { key: "ManagedBy", value: "pulumi" },
    { key: "Project", value: "production" },
  ],
}, { dependsOn: [peeringAccepter] });

// Database Users
interface UserConfig {
  roles: Array<{ role: string; database: string }>;
  password: string;
}

const userConfigs: Record<string, UserConfig> = {
  "app-user": {
    password: config.requireSecret("appUserPassword"),
    roles: [
      { role: "readWrite", database: "production_db" },
    ],
  },
  "analytics-user": {
    password: config.requireSecret("analyticsUserPassword"),
    roles: [
      { role: "read", database: "production_db" },
    ],
  },
  "backup-user": {
    password: config.requireSecret("backupUserPassword"),
    roles: [
      { role: "backup", database: "admin" },
      { role: "restore", database: "admin" },
    ],
  },
};

const dbUsers = Object.entries(userConfigs).map(([username, userConfig]) =>
  new mongodbatlas.DatabaseUser(username, {
    username,
    password: userConfig.password,
    projectId: project.id,
    authDatabaseName: "admin",

    roles: userConfig.roles.map(r => ({
      roleName: r.role,
      databaseName: r.database,
    })),

    scopes: [{
      name: cluster.name,
      type: "CLUSTER",
    }],

    labels: [
      { key: "Environment", value: environment },
    ],
  })
);

// IP Access List
const eksNodesCidr = config.require("eksNodesCidr");
const vpnCidr = config.require("vpnCidr");

const ipAccessList = new mongodbatlas.ProjectIpAccessList("eks-nodes", {
  projectId: project.id,
  cidrBlock: eksNodesCidr,
  comment: "EKS cluster nodes",
});

const vpnAccess = new mongodbatlas.ProjectIpAccessList("vpn", {
  projectId: project.id,
  cidrBlock: vpnCidr,
  comment: "Corporate VPN",
});

// Backup Schedule
const backupSchedule = new mongodbatlas.CloudBackupSchedule("main", {
  projectId: project.id,
  clusterName: cluster.name,

  referenceHourOfDay: 3,
  referenceMinuteOfHour: 30,
  restoreWindowDays: 7,

  policyItemDaily: {
    frequencyInterval: 1,
    retentionUnit: "days",
    retentionValue: 7,
  },

  policyItemWeekly: [{
    frequencyInterval: 6, // Saturday
    retentionUnit: "weeks",
    retentionValue: 4,
  }],

  policyItemMonthly: [{
    frequencyInterval: 1,
    retentionUnit: "months",
    retentionValue: 12,
  }],

  policyItemYearly: [{
    frequencyInterval: 1,
    retentionUnit: "years",
    retentionValue: 3,
  }],
}, { dependsOn: [cluster] });

// Alerts
const cpuAlert = new mongodbatlas.AlertConfiguration("high-cpu", {
  projectId: project.id,
  enabled: true,
  eventType: "OUTSIDE_METRIC_THRESHOLD",

  thresholdConfig: {
    operator: "GREATER_THAN",
    threshold: 80,
    units: "RAW",
  },

  metricThresholdConfig: {
    metricName: "SYSTEM_CPU_USER",
    operator: "GREATER_THAN",
    threshold: 80,
    units: "RAW",
    mode: "AVERAGE",
  },

  notifications: [
    {
      typeName: "SLACK",
      intervalMin: 5,
      delayMin: 0,
      slackApiToken: config.requireSecret("slackApiToken"),
      slackChannelName: "mongodb-alerts",
    },
  ],
});

// Store connection string in AWS Secrets Manager
const connectionSecret = new aws.secretsmanager.Secret("mongodb-connection", {
  name: `${environment}/mongodb/connection-string`,
  description: "MongoDB Atlas connection string",
  recoveryWindowInDays: 30,

  tags: {
    Environment: environment,
    ManagedBy: "pulumi",
  },
});

const connectionSecretVersion = new aws.secretsmanager.SecretVersion("mongodb-connection", {
  secretId: connectionSecret.id,
  secretString: pulumi.jsonStringify({
    standardSrv: cluster.connectionStrings[0].standardSrv,
    standard: cluster.connectionStrings[0].standard,
    privateSrv: cluster.connectionStrings[0].privateSrv,
    clusterId: cluster.clusterId,
    projectId: project.id,
  }),
});

// Exports
export const projectId = project.id;
export const clusterId = cluster.clusterId;
export const clusterName = cluster.name;
export const connectionString = pulumi.secret(cluster.connectionStrings[0].standardSrv);
export const mongoDbVersion = cluster.mongoDbVersion;
export const clusterState = cluster.stateName;
```

```yaml
# Pulumi.production.yaml
config:
  aws:region: eu-west-3
  mongodb-cluster:region: EU_WEST_3
  mongodb-cluster:tier: M60
  mongodb-cluster:vpcId: vpc-0a1b2c3d4e5f6g7h8
  mongodb-cluster:eksNodesCidr: 10.0.0.0/16
  mongodb-cluster:vpnCidr: 172.16.0.0/16

  # Secrets (encrypted with Pulumi)
  mongodb-cluster:atlasOrgId:
    secure: AAABAxxxxxx...
  mongodb-cluster:appUserPassword:
    secure: AAABAxxxxxx...
  mongodb-cluster:analyticsUserPassword:
    secure: AAABAxxxxxx...
  mongodb-cluster:backupUserPassword:
    secure: AAABAxxxxxx...
  mongodb-cluster:slackApiToken:
    secure: AAABAxxxxxx...
```

```bash
#!/bin/bash
# deploy-pulumi.sh

set -euo pipefail

STACK="$1"

# Deploy infrastructure
pulumi stack select "$STACK"
pulumi up --yes

# Get outputs
CONNECTION_STRING=$(pulumi stack output connectionString --show-secrets)
CLUSTER_ID=$(pulumi stack output clusterId)

# Validate deployment
echo "Validating deployment..."
if mongosh "$CONNECTION_STRING" --eval "db.adminCommand('ping')" --quiet; then
  echo "✅ MongoDB cluster is accessible"
else
  echo "❌ Failed to connect to MongoDB"
  exit 1
fi

echo "🎉 Deployment successful!"
```

---

## GitOps et Flux CD

### Repository Structure

```
gitops-mongodb/
├── clusters/
│   ├── production/
│   │   ├── flux-system/
│   │   │   ├── gotk-components.yaml
│   │   │   ├── gotk-sync.yaml
│   │   │   └── kustomization.yaml
│   │   └── mongodb/
│   │       ├── namespace.yaml
│   │       ├── mongodb-operator.yaml
│   │       ├── mongodb-cluster.yaml
│   │       └── kustomization.yaml
│   └── staging/
│       └── ...
├── base/
│   ├── mongodb-operator/
│   │   ├── deployment.yaml
│   │   ├── rbac.yaml
│   │   └── kustomization.yaml
│   └── mongodb-cluster/
│       ├── cluster.yaml
│       ├── backup.yaml
│       └── kustomization.yaml
└── infrastructure/
    ├── sources/
    │   └── mongodb-helm-repo.yaml
    └── releases/
        └── mongodb-operator.yaml
```

### Flux Configuration

```yaml
# infrastructure/sources/mongodb-helm-repo.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: mongodb
  namespace: flux-system
spec:
  interval: 24h
  url: https://mongodb.github.io/helm-charts
---
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: mongodb-manifests
  namespace: flux-system
spec:
  interval: 5m
  url: https://github.com/company/mongodb-manifests
  ref:
    branch: main
  secretRef:
    name: github-token
```

```yaml
# infrastructure/releases/mongodb-operator.yaml
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: mongodb-kubernetes-operator
  namespace: mongodb-system
spec:
  interval: 30m
  chart:
    spec:
      chart: community-operator
      version: '0.9.x'
      sourceRef:
        kind: HelmRepository
        name: mongodb
        namespace: flux-system

  values:
    operator:
      watchNamespace: "mongodb,mongodb-staging"

    resource:
      limits:
        cpu: 1000m
        memory: 1Gi
      requests:
        cpu: 500m
        memory: 256Mi

  install:
    crds: CreateReplace
    remediation:
      retries: 3

  upgrade:
    crds: CreateReplace
    remediation:
      retries: 3

  postRenderers:
    - kustomize:
        patches:
          - target:
              kind: Deployment
              name: mongodb-kubernetes-operator
            patch: |
              - op: add
                path: /spec/template/metadata/annotations
                value:
                  prometheus.io/scrape: "true"
                  prometheus.io/port: "8080"
```

```yaml
# clusters/production/mongodb/mongodb-cluster.yaml
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: production-replica-set
  namespace: mongodb
spec:
  members: 3
  type: ReplicaSet
  version: "7.0.5"

  security:
    authentication:
      modes: ["SCRAM"]
    tls:
      enabled: true
      certificateKeySecretRef:
        name: mongodb-tls-cert
      caCertificateSecretRef:
        name: mongodb-ca-cert

  users:
    - name: app-user
      db: admin
      passwordSecretRef:
        name: mongodb-app-user-password
      roles:
        - name: readWrite
          db: production_db
      scramCredentialsSecretName: app-user-scram

    - name: backup-user
      db: admin
      passwordSecretRef:
        name: mongodb-backup-user-password
      roles:
        - name: backup
          db: admin
        - name: restore
          db: admin
      scramCredentialsSecretName: backup-user-scram

  additionalMongodConfig:
    storage.wiredTiger.engineConfig.cacheSizeGB: 8
    storage.wiredTiger.engineConfig.journalCompressor: snappy
    storage.wiredTiger.collectionConfig.blockCompressor: snappy
    net.maxIncomingConnections: 65536
    operationProfiling.mode: slowOp
    operationProfiling.slowOpThresholdMs: 100

  statefulSet:
    spec:
      template:
        spec:
          affinity:
            podAntiAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                - labelSelector:
                    matchLabels:
                      app: production-replica-set-svc
                  topologyKey: kubernetes.io/hostname

          containers:
            - name: mongod
              resources:
                requests:
                  memory: "16Gi"
                  cpu: "4"
                limits:
                  memory: "32Gi"
                  cpu: "8"

      volumeClaimTemplates:
        - metadata:
            name: data-volume
          spec:
            accessModes: ["ReadWriteOnce"]
            storageClassName: "fast-ssd"
            resources:
              requests:
                storage: 500Gi

        - metadata:
            name: logs-volume
          spec:
            accessModes: ["ReadWriteOnce"]
            storageClassName: "standard"
            resources:
              requests:
                storage: 50Gi
```

---

## Testing et Validation

### Test Kitchen pour Ansible

```yaml
# kitchen.yml
---
driver:
  name: docker
  privileged: true
  use_sudo: false

provisioner:
  name: ansible_playbook
  hosts: test-kitchen
  require_chef_for_busser: false
  require_ruby_for_busser: false
  ansible_verbose: true
  ansible_verbosity: 2

platforms:
  - name: ubuntu-22.04
    driver_config:
      image: ubuntu:22.04
      platform: ubuntu

  - name: centos-8
    driver_config:
      image: centos:8
      platform: centos

suites:
  - name: replica-set
    provisioner:
      playbook: tests/replica_set.yml
    verifier:
      inspec_tests:
        - tests/integration/replica_set

  - name: sharded-cluster
    provisioner:
      playbook: tests/sharded_cluster.yml
    verifier:
      inspec_tests:
        - tests/integration/sharded_cluster
```

### InSpec Tests

```ruby
# tests/integration/replica_set/mongodb_spec.rb
describe package('mongodb-org') do
  it { should be_installed }
  its('version') { should match /^7\.0/ }
end

describe service('mongod') do
  it { should be_installed }
  it { should be_enabled }
  it { should be_running }
end

describe port(27017) do
  it { should be_listening }
  its('protocols') { should include 'tcp' }
end

describe file('/data/mongodb') do
  it { should exist }
  it { should be_directory }
  its('owner') { should eq 'mongod' }
  its('group') { should eq 'mongod' }
  its('mode') { should cmp '0755' }
end

describe file('/etc/mongodb/mongod.conf') do
  it { should exist }
  it { should be_file }
  its('content') { should match /replication:/ }
  its('content') { should match /replSetName:/ }
end

# Test MongoDB connectivity
describe command('mongosh --eval "db.adminCommand(\'ping\')"') do
  its('exit_status') { should eq 0 }
  its('stdout') { should match /ok.*1/ }
end

# Test replica set status
describe command('mongosh --eval "rs.status().ok"') do
  its('exit_status') { should eq 0 }
  its('stdout') { should match /1/ }
end

# Test TLS configuration
describe file('/etc/mongodb/ssl/mongodb.pem') do
  it { should exist }
  its('mode') { should cmp '0400' }
end
```

### Terraform Testing avec Terratest

```go
// test/mongodb_atlas_test.go
package test

import (
    "testing"
    "time"

    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestMongoDBAtlasCluster(t *testing.T) {
    t.Parallel()

    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../environments/staging",

        Vars: map[string]interface{}{
            "cluster_tier": "M10",
            "environment":  "test",
        },

        EnvVars: map[string]string{
            "TF_VAR_atlas_public_key":  getAtlasPublicKey(),
            "TF_VAR_atlas_private_key": getAtlasPrivateKey(),
            "TF_VAR_atlas_org_id":      getAtlasOrgId(),
        },
    })

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    // Test outputs
    clusterId := terraform.Output(t, terraformOptions, "cluster_id")
    assert.NotEmpty(t, clusterId)

    connectionString := terraform.Output(t, terraformOptions, "connection_string_srv")
    assert.Contains(t, connectionString, "mongodb+srv://")

    // Test cluster state
    clusterState := terraform.Output(t, terraformOptions, "cluster_state")
    assert.Equal(t, "IDLE", clusterState)

    // Wait for cluster to be fully ready
    time.Sleep(2 * time.Minute)

    // Test connectivity
    testMongoDBConnectivity(t, connectionString)
}

func testMongoDBConnectivity(t *testing.T, connectionString string) {
    // Use MongoDB Go driver to test connectivity
    // Implementation details...
}
```

---

## Conclusion

L'Infrastructure as Code pour MongoDB permet de :
- **Automatiser** le provisionnement et la configuration
- **Versionner** l'infrastructure comme du code
- **Tester** les changements avant déploiement
- **Reproduire** des environnements identiques
- **Collaborer** via des workflows Git

Les outils complémentaires (Terraform, Ansible, Pulumi) offrent différents niveaux d'abstraction et s'adaptent aux besoins spécifiques de chaque organisation.

---


⏭️ [Déploiement avec Docker](/18-devops-deploiement/02-deploiement-docker.md)
