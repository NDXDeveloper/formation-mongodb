🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.7 Terraform et MongoDB Atlas

## Introduction

Terraform est l'outil d'Infrastructure as Code (IaC) de référence pour provisionner et gérer des ressources cloud de manière déclarative. MongoDB Atlas, la plateforme DBaaS (Database as a Service) de MongoDB, s'intègre parfaitement avec Terraform via un provider officiel, permettant de gérer l'ensemble du cycle de vie des clusters MongoDB dans le cloud.

```
┌───────────────────────────────────────────────────────────────────┐
│         Terraform + MongoDB Atlas Architecture                    │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              Terraform Configuration                        │  │
│  │                                                             │  │
│  │  ┌───────────────────────────────────────────────────────┐  │  │
│  │  │         HCL Configuration Files                       │  │  │
│  │  │                                                       │  │  │
│  │  │  • main.tf            - Main resources                │  │  │
│  │  │  • variables.tf       - Input variables               │  │  │
│  │  │  • outputs.tf         - Output values                 │  │  │
│  │  │  • providers.tf       - Provider config               │  │  │
│  │  │  • terraform.tfvars   - Variable values               │  │  │
│  │  │  • backend.tf         - State backend                 │  │  │
│  │  └───────────────────────────────────────────────────────┘  │  │
│  └────────────────────────┬────────────────────────────────────┘  │
│                           │                                       │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │           Terraform Core Engine                             │  │
│  │                                                             │  │
│  │  • Plan      - Calculate changes                            │  │
│  │  • Apply     - Execute changes                              │  │
│  │  • Destroy   - Remove resources                             │  │
│  │  • State     - Track resources                              │  │
│  └────────────────────────┬────────────────────────────────────┘  │
│                           │                                       │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │         MongoDB Atlas Provider                              │  │
│  │                                                             │  │
│  │  • Organizations                                            │  │
│  │  • Projects                                                 │  │
│  │  • Clusters (M0-M700)                                       │  │
│  │  • Database Users                                           │  │
│  │  • Network Access (IP, VPC Peering, PrivateLink)            │  │
│  │  • Backups & Snapshots                                      │  │
│  │  • Alerts & Monitoring                                      │  │
│  └────────────────────────┬────────────────────────────────────┘  │
│                           │ API Calls                             │
│                           ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │              MongoDB Atlas Cloud                            │  │
│  │                                                             │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │  │
│  │  │   AWS        │  │     GCP      │  │    Azure     │       │  │
│  │  │              │  │              │  │              │       │  │
│  │  │  • Clusters  │  │  • Clusters  │  │  • Clusters  │       │  │
│  │  │  • VPC Peer  │  │  • VPC Peer  │  │  • VNet Peer │       │  │
│  │  │  • Private   │  │  • Private   │  │  • Private   │       │  │
│  │  │    Endpoint  │  │    Service   │  │    Endpoint  │       │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘       │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Avantages de Terraform avec MongoDB Atlas

```yaml
# terraform-mongodb-atlas-benefits.yaml
---
infrastructure_as_code:
  description: "Infrastructure versionnée et reproductible"
  benefits:
    - "Git-based workflow"
    - "Code review pour les changements infra"
    - "Historique complet des modifications"
    - "Rollback facilité"

declarative:
  description: "Déclaration de l'état désiré"
  benefits:
    - "Abstraction de la complexité API"
    - "Idempotence garantie"
    - "Drift detection"
    - "Plan before apply"

multi_cloud:
  description: "Support multi-cloud natif"
  benefits:
    - "AWS, GCP, Azure support"
    - "Consistent interface"
    - "Multi-region deployments"
    - "Hybrid cloud strategies"

automation:
  description: "Automatisation complète du cycle de vie"
  benefits:
    - "CI/CD integration"
    - "GitOps workflows"
    - "Self-service portals"
    - "Reduced manual errors"

ecosystem:
  description: "Écosystème riche de providers"
  benefits:
    - "Integration avec AWS, GCP, Azure providers"
    - "Modules réutilisables"
    - "Terraform Cloud/Enterprise"
    - "Large community"
```

---

## Configuration de Base

### Installation de Terraform

```bash
# Installation sur Linux (Ubuntu/Debian)
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

# Via Homebrew (macOS)
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Via Chocolatey (Windows)
choco install terraform

# Vérifier l'installation
terraform version

# Auto-completion
terraform -install-autocomplete
```

### Structure du Projet

```bash
# Créer la structure du projet
mkdir -p terraform-mongodb-atlas/{environments,modules,policies}

# Structure complète
tree terraform-mongodb-atlas/
```

```
terraform-mongodb-atlas/
├── main.tf                          # Configuration principale
├── variables.tf                     # Variables d'entrée
├── outputs.tf                       # Outputs
├── providers.tf                     # Configuration providers
├── backend.tf                       # Configuration state backend
├── versions.tf                      # Version constraints
├── terraform.tfvars                 # Valeurs des variables (gitignored)
├── terraform.tfvars.example         # Exemple de variables
├── .terraform.lock.hcl             # Lock file des providers
├── environments/                    # Configurations par environnement
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── modules/                         # Modules réutilisables
│   ├── mongodb-cluster/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   ├── mongodb-network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── mongodb-users/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
├── policies/                        # OPA policies
│   └── mongodb-policies.rego
└── scripts/
    ├── init.sh
    ├── plan.sh
    └── apply.sh
```

### Configuration MongoDB Atlas API

```bash
# Obtenir les credentials Atlas
# 1. Se connecter à MongoDB Atlas Console
# 2. Organization Access Manager → API Keys
# 3. Create API Key

# Variables d'environnement
export MONGODB_ATLAS_PUBLIC_KEY="your-public-key"
export MONGODB_ATLAS_PRIVATE_KEY="your-private-key"

# Ou via fichier .env
cat > .env <<EOF
MONGODB_ATLAS_PUBLIC_KEY=your-public-key
MONGODB_ATLAS_PRIVATE_KEY=your-private-key
EOF

# Charger les variables
source .env
```

---

## Providers et Versions

### providers.tf - Configuration Provider

```hcl
# providers.tf
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.15.0"
    }

    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }

    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }

    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }

    time = {
      source  = "hashicorp/time"
      version = "~> 0.9"
    }
  }
}

# MongoDB Atlas Provider
provider "mongodbatlas" {
  public_key  = var.mongodb_atlas_public_key
  private_key = var.mongodb_atlas_private_key
}

# AWS Provider
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
      Owner       = var.owner
    }
  }
}

# Google Cloud Provider
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region
}

# Azure Provider
provider "azurerm" {
  features {}

  subscription_id = var.azure_subscription_id
}
```

### backend.tf - State Backend

```hcl
# backend.tf
# Configuration du backend S3 pour le state Terraform

terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "mongodb-atlas/production/terraform.tfstate"
    region         = "eu-west-3"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"

    # Tags sur le bucket S3
    tags = {
      Name        = "Terraform State"
      Environment = "Production"
      Project     = "MongoDB Atlas"
    }
  }
}

# Alternative: Terraform Cloud Backend
# terraform {
#   cloud {
#     organization = "company"
#
#     workspaces {
#       name = "mongodb-atlas-production"
#     }
#   }
# }

# Alternative: Azure Backend
# terraform {
#   backend "azurerm" {
#     resource_group_name  = "terraform-state-rg"
#     storage_account_name = "companytfstate"
#     container_name       = "tfstate"
#     key                  = "mongodb-atlas.tfstate"
#   }
# }
```

### variables.tf - Variables Globales

```hcl
# variables.tf
# MongoDB Atlas Credentials
variable "mongodb_atlas_public_key" {
  description = "MongoDB Atlas Public API Key"
  type        = string
  sensitive   = true
}

variable "mongodb_atlas_private_key" {
  description = "MongoDB Atlas Private API Key"
  type        = string
  sensitive   = true
}

variable "mongodb_atlas_org_id" {
  description = "MongoDB Atlas Organization ID"
  type        = string
}

# Project Configuration
variable "project_name" {
  description = "Name of the MongoDB Atlas project"
  type        = string
  default     = "production-mongodb"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be dev, staging, or production."
  }
}

variable "owner" {
  description = "Owner of the infrastructure"
  type        = string
  default     = "devops-team"
}

# Cloud Provider Configuration
variable "cloud_provider" {
  description = "Cloud provider for MongoDB Atlas"
  type        = string
  default     = "AWS"
  validation {
    condition     = contains(["AWS", "GCP", "AZURE"], var.cloud_provider)
    error_message = "Cloud provider must be AWS, GCP, or AZURE."
  }
}

variable "aws_region" {
  description = "AWS region for MongoDB Atlas cluster"
  type        = string
  default     = "EU_WEST_3"
}

variable "gcp_region" {
  description = "GCP region for MongoDB Atlas cluster"
  type        = string
  default     = "europe-west1"
}

variable "azure_region" {
  description = "Azure region for MongoDB Atlas cluster"
  type        = string
  default     = "westeurope"
}

# Cluster Configuration
variable "cluster_name" {
  description = "Name of the MongoDB Atlas cluster"
  type        = string
  default     = "production-cluster"
}

variable "cluster_tier" {
  description = "MongoDB Atlas cluster tier"
  type        = string
  default     = "M30"
  validation {
    condition = can(regex("^M(0|2|5|10|20|30|40|50|60|80|100|140|200|300|400|700)$", var.cluster_tier))
    error_message = "Invalid cluster tier. Must be M0, M2, M5, M10, M20, M30, etc."
  }
}

variable "mongodb_version" {
  description = "MongoDB version"
  type        = string
  default     = "7.0"
}

variable "disk_size_gb" {
  description = "Disk size in GB"
  type        = number
  default     = 100
}

variable "auto_scaling_enabled" {
  description = "Enable cluster auto-scaling"
  type        = bool
  default     = true
}

# Network Configuration
variable "ip_whitelist" {
  description = "List of IP addresses to whitelist"
  type = list(object({
    cidr_block = string
    comment    = string
  }))
  default = []
}

# Backup Configuration
variable "backup_enabled" {
  description = "Enable continuous backup"
  type        = bool
  default     = true
}

variable "pit_enabled" {
  description = "Enable point-in-time recovery"
  type        = bool
  default     = true
}

# Tags
variable "tags" {
  description = "Additional tags for resources"
  type        = map(string)
  default     = {}
}
```

---

## Déploiement d'un Cluster MongoDB Atlas

### main.tf - Configuration Principale

```hcl
# main.tf
# Configuration principale pour MongoDB Atlas

# Project MongoDB Atlas
resource "mongodbatlas_project" "main" {
  name   = var.project_name
  org_id = var.mongodb_atlas_org_id

  # Teams avec permissions
  teams {
    team_id    = mongodbatlas_team.admin.team_id
    role_names = ["GROUP_OWNER"]
  }

  teams {
    team_id    = mongodbatlas_team.developers.team_id
    role_names = ["GROUP_DATA_ACCESS_ADMIN"]
  }

  teams {
    team_id    = mongodbatlas_team.readonly.team_id
    role_names = ["GROUP_READ_ONLY"]
  }

  # Alertes au niveau projet
  is_collect_database_specifics_statistics_enabled = true
  is_data_explorer_enabled                         = true
  is_performance_advisor_enabled                   = true
  is_realtime_performance_panel_enabled           = true
  is_schema_advisor_enabled                        = true
}

# Teams
resource "mongodbatlas_team" "admin" {
  org_id    = var.mongodb_atlas_org_id
  name      = "${var.project_name}-admin"
  usernames = var.admin_users
}

resource "mongodbatlas_team" "developers" {
  org_id    = var.mongodb_atlas_org_id
  name      = "${var.project_name}-developers"
  usernames = var.developer_users
}

resource "mongodbatlas_team" "readonly" {
  org_id    = var.mongodb_atlas_org_id
  name      = "${var.project_name}-readonly"
  usernames = var.readonly_users
}

# Cluster MongoDB Atlas
resource "mongodbatlas_cluster" "main" {
  project_id = mongodbatlas_project.main.id
  name       = var.cluster_name

  # Cloud Provider
  provider_name               = var.cloud_provider
  provider_instance_size_name = var.cluster_tier

  # MongoDB Version
  mongo_db_major_version = var.mongodb_version

  # Cluster Type
  cluster_type = "REPLICASET"

  # Replication Specs
  replication_specs {
    num_shards = 1

    regions_config {
      region_name     = var.aws_region
      electable_nodes = 3
      priority        = 7
      read_only_nodes = 0
    }
  }

  # Storage
  disk_size_gb = var.disk_size_gb

  # Auto-scaling
  auto_scaling_disk_gb_enabled = var.auto_scaling_enabled

  dynamic "auto_scaling_compute_enabled" {
    for_each = var.auto_scaling_enabled ? [1] : []
    content {
      enabled = true
    }
  }

  auto_scaling_compute_scale_down_enabled = var.auto_scaling_enabled

  # Backup
  backup_enabled                   = var.backup_enabled
  pit_enabled                      = var.pit_enabled
  cloud_backup                     = var.backup_enabled
  retain_backups_enabled          = true

  # Advanced Configuration
  advanced_configuration {
    javascript_enabled                   = false
    minimum_enabled_tls_protocol        = "TLS1_2"
    no_table_scan                       = false
    oplog_size_mb                       = 10240
    sample_size_bi_connector           = 5000
    sample_refresh_interval_bi_connector = 300
  }

  # Bi-Connector
  bi_connector_config {
    enabled         = false
    read_preference = "secondary"
  }

  # Labels
  labels {
    key   = "Environment"
    value = var.environment
  }

  labels {
    key   = "ManagedBy"
    value = "Terraform"
  }

  labels {
    key   = "Project"
    value = var.project_name
  }

  # Lifecycle
  lifecycle {
    prevent_destroy = true

    ignore_changes = [
      disk_size_gb,  # Ignoré car auto-scaling
    ]
  }

  depends_on = [
    mongodbatlas_project.main
  ]
}

# Cluster Multi-Region (optionnel)
resource "mongodbatlas_cluster" "multi_region" {
  count = var.multi_region_enabled ? 1 : 0

  project_id = mongodbatlas_project.main.id
  name       = "${var.cluster_name}-multi-region"

  provider_name               = var.cloud_provider
  provider_instance_size_name = var.cluster_tier
  mongo_db_major_version      = var.mongodb_version
  cluster_type                = "REPLICASET"

  # Multi-Region Configuration
  replication_specs {
    num_shards = 1

    # Region 1 (Primary)
    regions_config {
      region_name     = var.aws_region
      electable_nodes = 2
      priority        = 7
      read_only_nodes = 0
    }

    # Region 2 (Secondary)
    regions_config {
      region_name     = var.aws_region_secondary
      electable_nodes = 2
      priority        = 6
      read_only_nodes = 0
    }

    # Region 3 (DR)
    regions_config {
      region_name     = var.aws_region_dr
      electable_nodes = 1
      priority        = 5
      read_only_nodes = 0
    }
  }

  disk_size_gb = var.disk_size_gb

  backup_enabled = true
  pit_enabled    = true
  cloud_backup   = true

  advanced_configuration {
    javascript_enabled           = false
    minimum_enabled_tls_protocol = "TLS1_2"
    oplog_size_mb                = 20480
  }
}

# Advanced Cluster (Sharded)
resource "mongodbatlas_advanced_cluster" "sharded" {
  count = var.sharded_cluster_enabled ? 1 : 0

  project_id   = mongodbatlas_project.main.id
  name         = "${var.cluster_name}-sharded"
  cluster_type = "SHARDED"

  # MongoDB Version
  mongo_db_major_version = var.mongodb_version

  # Backup
  backup_enabled = true
  pit_enabled    = true

  # Replication Specs
  replication_specs {
    # Shard 1
    region_configs {
      electable_specs {
        instance_size = var.cluster_tier
        node_count    = 3
      }

      analytics_specs {
        instance_size = var.analytics_tier
        node_count    = 1
      }

      provider_name = var.cloud_provider
      priority      = 7
      region_name   = var.aws_region
    }
  }

  replication_specs {
    # Shard 2
    region_configs {
      electable_specs {
        instance_size = var.cluster_tier
        node_count    = 3
      }

      analytics_specs {
        instance_size = var.analytics_tier
        node_count    = 1
      }

      provider_name = var.cloud_provider
      priority      = 7
      region_name   = var.aws_region
    }
  }

  # Advanced Configuration
  advanced_configuration {
    javascript_enabled                   = false
    minimum_enabled_tls_protocol        = "TLS1_2"
    oplog_min_retention_hours           = 24
    sample_size_bi_connector            = 5000
    sample_refresh_interval_bi_connector = 300
  }

  # Bi-Connector
  bi_connector_config {
    enabled         = true
    read_preference = "analytics"
  }

  # Labels
  labels {
    key   = "ClusterType"
    value = "Sharded"
  }
}

# Database Users
resource "mongodbatlas_database_user" "admin" {
  username           = "admin"
  password           = random_password.admin_password.result
  project_id         = mongodbatlas_project.main.id
  auth_database_name = "admin"

  roles {
    role_name     = "atlasAdmin"
    database_name = "admin"
  }

  labels {
    key   = "Type"
    value = "Admin"
  }

  scopes {
    name = mongodbatlas_cluster.main.name
    type = "CLUSTER"
  }
}

resource "mongodbatlas_database_user" "app_user" {
  username           = var.app_username
  password           = random_password.app_password.result
  project_id         = mongodbatlas_project.main.id
  auth_database_name = "admin"

  roles {
    role_name     = "readWrite"
    database_name = var.app_database
  }

  roles {
    role_name     = "read"
    database_name = "config"
  }

  labels {
    key   = "Type"
    value = "Application"
  }

  scopes {
    name = mongodbatlas_cluster.main.name
    type = "CLUSTER"
  }
}

resource "mongodbatlas_database_user" "backup_user" {
  username           = "backup-user"
  password           = random_password.backup_password.result
  project_id         = mongodbatlas_project.main.id
  auth_database_name = "admin"

  roles {
    role_name     = "backup"
    database_name = "admin"
  }

  labels {
    key   = "Type"
    value = "Backup"
  }

  scopes {
    name = mongodbatlas_cluster.main.name
    type = "CLUSTER"
  }
}

# Passwords
resource "random_password" "admin_password" {
  length  = 32
  special = true
}

resource "random_password" "app_password" {
  length  = 32
  special = true
}

resource "random_password" "backup_password" {
  length  = 32
  special = true
}

# Store passwords in AWS Secrets Manager
resource "aws_secretsmanager_secret" "mongodb_credentials" {
  name = "${var.project_name}-mongodb-credentials"

  recovery_window_in_days = 7

  tags = merge(
    var.tags,
    {
      Name = "${var.project_name}-mongodb-credentials"
    }
  )
}

resource "aws_secretsmanager_secret_version" "mongodb_credentials" {
  secret_id = aws_secretsmanager_secret.mongodb_credentials.id

  secret_string = jsonencode({
    admin_username    = mongodbatlas_database_user.admin.username
    admin_password    = random_password.admin_password.result
    app_username      = mongodbatlas_database_user.app_user.username
    app_password      = random_password.app_password.result
    backup_username   = mongodbatlas_database_user.backup_user.username
    backup_password   = random_password.backup_password.result
    connection_string = mongodbatlas_cluster.main.connection_strings[0].standard_srv
  })
}
```

---

## Configuration Réseau

### Network Access - IP Whitelist

```hcl
# network.tf
# Configuration réseau pour MongoDB Atlas

# IP Whitelist
resource "mongodbatlas_project_ip_access_list" "office" {
  project_id = mongodbatlas_project.main.id
  cidr_block = "203.0.113.0/24"
  comment    = "Office IP Range"
}

resource "mongodbatlas_project_ip_access_list" "vpn" {
  project_id = mongodbatlas_project.main.id
  cidr_block = "198.51.100.0/24"
  comment    = "VPN IP Range"
}

# Dynamic IP Whitelist
resource "mongodbatlas_project_ip_access_list" "dynamic" {
  for_each = { for idx, ip in var.ip_whitelist : idx => ip }

  project_id = mongodbatlas_project.main.id
  cidr_block = each.value.cidr_block
  comment    = each.value.comment
}

# Temporary Access (expires after 7 days)
resource "mongodbatlas_project_ip_access_list" "temporary" {
  project_id      = mongodbatlas_project.main.id
  cidr_block      = var.temporary_ip
  comment         = "Temporary Access"
  delete_after_date = timeadd(timestamp(), "168h")  # 7 days
}
```

### VPC Peering - AWS

```hcl
# vpc-peering-aws.tf
# Configuration VPC Peering entre AWS et MongoDB Atlas

# AWS VPC
data "aws_vpc" "main" {
  id = var.aws_vpc_id
}

data "aws_route_table" "main" {
  vpc_id = data.aws_vpc.main.id
}

# MongoDB Atlas Network Container
resource "mongodbatlas_network_container" "aws" {
  project_id       = mongodbatlas_project.main.id
  atlas_cidr_block = var.atlas_cidr_block
  provider_name    = "AWS"
  region_name      = var.aws_region
}

# MongoDB Atlas Network Peering
resource "mongodbatlas_network_peering" "aws" {
  accepter_region_name   = var.aws_region
  project_id             = mongodbatlas_project.main.id
  container_id           = mongodbatlas_network_container.aws.container_id
  provider_name          = "AWS"
  route_table_cidr_block = data.aws_vpc.main.cidr_block
  vpc_id                 = data.aws_vpc.main.id
  aws_account_id         = var.aws_account_id

  depends_on = [mongodbatlas_network_container.aws]
}

# AWS VPC Peering Connection Accepter
resource "aws_vpc_peering_connection_accepter" "atlas" {
  vpc_peering_connection_id = mongodbatlas_network_peering.aws.connection_id
  auto_accept               = true

  tags = {
    Name = "VPC Peering to MongoDB Atlas"
    Side = "Accepter"
  }
}

# AWS Route to MongoDB Atlas
resource "aws_route" "atlas" {
  route_table_id            = data.aws_route_table.main.id
  destination_cidr_block    = var.atlas_cidr_block
  vpc_peering_connection_id = mongodbatlas_network_peering.aws.connection_id

  depends_on = [aws_vpc_peering_connection_accepter.atlas]
}

# Security Group for MongoDB Access
resource "aws_security_group" "mongodb_access" {
  name        = "${var.project_name}-mongodb-access"
  description = "Allow access to MongoDB Atlas"
  vpc_id      = data.aws_vpc.main.id

  ingress {
    description = "MongoDB from VPC"
    from_port   = 27017
    to_port     = 27017
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
  }

  egress {
    description = "MongoDB Atlas"
    from_port   = 27017
    to_port     = 27017
    protocol    = "tcp"
    cidr_blocks = [var.atlas_cidr_block]
  }

  tags = {
    Name = "${var.project_name}-mongodb-access"
  }
}
```

### AWS PrivateLink

```hcl
# privatelink-aws.tf
# Configuration AWS PrivateLink pour MongoDB Atlas

# PrivateLink Endpoint
resource "mongodbatlas_privatelink_endpoint" "aws" {
  project_id    = mongodbatlas_project.main.id
  provider_name = "AWS"
  region        = var.aws_region
}

# AWS VPC Endpoint
resource "aws_vpc_endpoint" "mongodb" {
  vpc_id             = var.aws_vpc_id
  service_name       = mongodbatlas_privatelink_endpoint.aws.endpoint_service_name
  vpc_endpoint_type  = "Interface"
  subnet_ids         = var.aws_private_subnet_ids
  security_group_ids = [aws_security_group.privatelink.id]

  private_dns_enabled = false

  tags = {
    Name = "${var.project_name}-mongodb-privatelink"
  }
}

# Security Group for PrivateLink
resource "aws_security_group" "privatelink" {
  name        = "${var.project_name}-mongodb-privatelink"
  description = "Allow MongoDB through PrivateLink"
  vpc_id      = var.aws_vpc_id

  ingress {
    description = "MongoDB from VPC"
    from_port   = 1024
    to_port     = 1026
    protocol    = "tcp"
    cidr_blocks = [data.aws_vpc.main.cidr_block]
  }

  egress {
    description = "All outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project_name}-mongodb-privatelink"
  }
}

# PrivateLink Endpoint Service
resource "mongodbatlas_privatelink_endpoint_service" "aws" {
  project_id          = mongodbatlas_project.main.id
  private_link_id     = mongodbatlas_privatelink_endpoint.aws.private_link_id
  endpoint_service_id = aws_vpc_endpoint.mongodb.id
  provider_name       = "AWS"

  depends_on = [
    aws_vpc_endpoint.mongodb,
    mongodbatlas_privatelink_endpoint.aws
  ]
}

# Route53 Private Hosted Zone
resource "aws_route53_zone" "mongodb" {
  name = mongodbatlas_cluster.main.connection_strings[0].private_endpoint[0].srv_connection_string

  vpc {
    vpc_id = var.aws_vpc_id
  }

  tags = {
    Name = "${var.project_name}-mongodb-privatelink"
  }
}

# Route53 Records
resource "aws_route53_record" "mongodb" {
  for_each = toset(mongodbatlas_cluster.main.connection_strings[0].private_endpoint[0].endpoints)

  zone_id = aws_route53_zone.mongodb.zone_id
  name    = each.value.endpoint
  type    = "CNAME"
  ttl     = 300
  records = [aws_vpc_endpoint.mongodb.dns_entry[0].dns_name]
}
```

---

## Backup et Disaster Recovery

### Cloud Backup Configuration

```hcl
# backup.tf
# Configuration des backups MongoDB Atlas

# Cloud Backup Schedule
resource "mongodbatlas_cloud_backup_schedule" "main" {
  project_id   = mongodbatlas_project.main.id
  cluster_name = mongodbatlas_cluster.main.name

  # Continuous Backup
  auto_export_enabled = true

  # Policy Items
  policy_item_hourly {
    frequency_interval = 6  # Every 6 hours
    retention_unit     = "days"
    retention_value    = 7
  }

  policy_item_daily {
    frequency_interval = 1  # Every day
    retention_unit     = "days"
    retention_value    = 30
  }

  policy_item_weekly {
    frequency_interval = 1  # Every week
    retention_unit     = "weeks"
    retention_value    = 4
  }

  policy_item_monthly {
    frequency_interval = 1  # Every month
    retention_unit     = "months"
    retention_value    = 12
  }

  policy_item_yearly {
    frequency_interval = 1  # Every year
    retention_unit     = "years"
    retention_value    = 7
  }

  # Copy Settings
  copy_settings {
    cloud_provider     = "AWS"
    frequencies        = ["DAILY", "MONTHLY"]
    region_name        = var.backup_region
    replication_spec_id = mongodbatlas_cluster.main.replication_specs[0].id
    should_copy_oplogs = true
  }

  # Export Settings (to S3)
  export {
    export_bucket_id = mongodbatlas_cloud_backup_snapshot_export_bucket.aws.export_bucket_id
    frequency_type   = "monthly"
  }

  depends_on = [mongodbatlas_cluster.main]
}

# S3 Bucket for Backup Export
resource "aws_s3_bucket" "backup" {
  bucket = "${var.project_name}-mongodb-backup"

  tags = {
    Name = "${var.project_name}-mongodb-backup"
  }
}

resource "aws_s3_bucket_versioning" "backup" {
  bucket = aws_s3_bucket.backup.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_encryption" "backup" {
  bucket = aws_s3_bucket.backup.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "backup" {
  bucket = aws_s3_bucket.backup.id

  rule {
    id     = "transition-to-glacier"
    status = "Enabled"

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 2555  # 7 years
    }
  }
}

# IAM Role for MongoDB Atlas to access S3
resource "aws_iam_role" "mongodb_backup" {
  name = "${var.project_name}-mongodb-backup-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = var.mongodb_atlas_aws_account_arn
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "sts:ExternalId" = var.mongodb_atlas_org_id
          }
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "mongodb_backup" {
  name = "${var.project_name}-mongodb-backup-policy"
  role = aws_iam_role.mongodb_backup.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket",
          "s3:GetBucketLocation"
        ]
        Resource = [
          aws_s3_bucket.backup.arn,
          "${aws_s3_bucket.backup.arn}/*"
        ]
      }
    ]
  })
}

# MongoDB Atlas Backup Export Bucket
resource "mongodbatlas_cloud_backup_snapshot_export_bucket" "aws" {
  project_id     = mongodbatlas_project.main.id
  iam_role_id    = aws_iam_role.mongodb_backup.arn
  bucket_name    = aws_s3_bucket.backup.id
  cloud_provider = "AWS"
}

# Restore Job (example - run manually or via automation)
resource "mongodbatlas_cloud_backup_snapshot_restore_job" "example" {
  count = var.restore_enabled ? 1 : 0

  project_id      = mongodbatlas_project.main.id
  cluster_name    = mongodbatlas_cluster.main.name
  snapshot_id     = var.snapshot_id
  delivery_type_config {
    download = true
  }
}
```

---

## Monitoring et Alerting

### Alerts Configuration

```hcl
# alerts.tf
# Configuration des alertes MongoDB Atlas

# Alert - Replication Lag
resource "mongodbatlas_alert_configuration" "replication_lag" {
  project_id = mongodbatlas_project.main.id

  event_type = "REPLICATION_OPLOG_WINDOW_RUNNING_OUT"
  enabled    = true

  notification {
    type_name     = "GROUP"
    interval_min  = 5
    delay_min     = 0
    sms_enabled   = false
    email_enabled = true
    roles         = ["GROUP_OWNER"]
  }

  notification {
    type_name    = "SLACK"
    channel_name = "#mongodb-alerts"
    api_token    = var.slack_api_token
    interval_min = 5
    delay_min    = 0
  }

  notification {
    type_name    = "PAGER_DUTY"
    service_key  = var.pagerduty_service_key
    interval_min = 0
    delay_min    = 0
  }

  matcher {
    field_name = "CLUSTER_NAME"
    operator   = "EQUALS"
    value      = mongodbatlas_cluster.main.name
  }
}

# Alert - High CPU
resource "mongodbatlas_alert_configuration" "high_cpu" {
  project_id = mongodbatlas_project.main.id

  event_type = "OUTSIDE_METRIC_THRESHOLD"
  enabled    = true

  notification {
    type_name     = "GROUP"
    interval_min  = 5
    delay_min     = 0
    email_enabled = true
    roles         = ["GROUP_CLUSTER_MANAGER"]
  }

  metric_threshold_config {
    metric_name = "NORMALIZED_SYSTEM_CPU_USER"
    operator    = "GREATER_THAN"
    threshold   = 80.0
    units       = "RAW"
    mode        = "AVERAGE"
  }

  matcher {
    field_name = "CLUSTER_NAME"
    operator   = "EQUALS"
    value      = mongodbatlas_cluster.main.name
  }
}

# Alert - High Connections
resource "mongodbatlas_alert_configuration" "high_connections" {
  project_id = mongodbatlas_project.main.id

  event_type = "OUTSIDE_METRIC_THRESHOLD"
  enabled    = true

  notification {
    type_name     = "GROUP"
    interval_min  = 5
    delay_min     = 0
    email_enabled = true
  }

  metric_threshold_config {
    metric_name = "CONNECTIONS_PERCENT"
    operator    = "GREATER_THAN"
    threshold   = 80.0
    units       = "RAW"
    mode        = "AVERAGE"
  }

  matcher {
    field_name = "CLUSTER_NAME"
    operator   = "EQUALS"
    value      = mongodbatlas_cluster.main.name
  }
}

# Alert - Disk Space
resource "mongodbatlas_alert_configuration" "disk_space" {
  project_id = mongodbatlas_project.main.id

  event_type = "OUTSIDE_METRIC_THRESHOLD"
  enabled    = true

  notification {
    type_name     = "GROUP"
    interval_min  = 5
    delay_min     = 0
    email_enabled = true
    roles         = ["GROUP_OWNER"]
  }

  metric_threshold_config {
    metric_name = "DISK_PARTITION_SPACE_PERCENT_USED"
    operator    = "GREATER_THAN"
    threshold   = 85.0
    units       = "RAW"
    mode        = "AVERAGE"
  }

  matcher {
    field_name = "CLUSTER_NAME"
    operator   = "EQUALS"
    value      = mongodbatlas_cluster.main.name
  }
}

# Alert - Cluster Down
resource "mongodbatlas_alert_configuration" "cluster_down" {
  project_id = mongodbatlas_project.main.id

  event_type = "CLUSTER_MONGOS_IS_MISSING"
  enabled    = true

  notification {
    type_name     = "GROUP"
    interval_min  = 0
    delay_min     = 0
    sms_enabled   = true
    email_enabled = true
    roles         = ["GROUP_OWNER"]
  }

  notification {
    type_name    = "PAGER_DUTY"
    service_key  = var.pagerduty_service_key
    interval_min = 0
    delay_min    = 0
  }

  matcher {
    field_name = "CLUSTER_NAME"
    operator   = "EQUALS"
    value      = mongodbatlas_cluster.main.name
  }
}

# Alert - Backup Failed
resource "mongodbatlas_alert_configuration" "backup_failed" {
  project_id = mongodbatlas_project.main.id

  event_type = "BACKUP_JOB_FAILED"
  enabled    = true

  notification {
    type_name     = "GROUP"
    interval_min  = 0
    delay_min     = 0
    email_enabled = true
    roles         = ["GROUP_OWNER"]
  }

  matcher {
    field_name = "CLUSTER_NAME"
    operator   = "EQUALS"
    value      = mongodbatlas_cluster.main.name
  }
}

# Third-party Monitoring Integration
resource "mongodbatlas_third_party_integration" "datadog" {
  count = var.datadog_enabled ? 1 : 0

  project_id = mongodbatlas_project.main.id
  type       = "DATADOG"
  api_key    = var.datadog_api_key
  region     = var.datadog_region
}

resource "mongodbatlas_third_party_integration" "prometheus" {
  count = var.prometheus_enabled ? 1 : 0

  project_id  = mongodbatlas_project.main.id
  type        = "PROMETHEUS"
  user_name   = var.prometheus_user
  password    = var.prometheus_password
  service_discovery = "http"
  enabled     = true
}
```

---

## Outputs

### outputs.tf - Valeurs de Sortie

```hcl
# outputs.tf
# Outputs pour MongoDB Atlas

# Project Information
output "project_id" {
  description = "MongoDB Atlas Project ID"
  value       = mongodbatlas_project.main.id
}

output "project_name" {
  description = "MongoDB Atlas Project Name"
  value       = mongodbatlas_project.main.name
}

# Cluster Information
output "cluster_id" {
  description = "MongoDB Atlas Cluster ID"
  value       = mongodbatlas_cluster.main.cluster_id
}

output "cluster_name" {
  description = "MongoDB Atlas Cluster Name"
  value       = mongodbatlas_cluster.main.name
}

output "cluster_state" {
  description = "MongoDB Atlas Cluster State"
  value       = mongodbatlas_cluster.main.state_name
}

output "mongodb_version" {
  description = "MongoDB Version"
  value       = mongodbatlas_cluster.main.mongo_db_version
}

# Connection Strings
output "connection_string_standard" {
  description = "Standard MongoDB connection string"
  value       = mongodbatlas_cluster.main.connection_strings[0].standard
  sensitive   = true
}

output "connection_string_standard_srv" {
  description = "Standard SRV MongoDB connection string"
  value       = mongodbatlas_cluster.main.connection_strings[0].standard_srv
  sensitive   = true
}

output "connection_string_private" {
  description = "Private MongoDB connection string"
  value       = try(mongodbatlas_cluster.main.connection_strings[0].private, null)
  sensitive   = true
}

output "connection_string_private_srv" {
  description = "Private SRV MongoDB connection string"
  value       = try(mongodbatlas_cluster.main.connection_strings[0].private_srv, null)
  sensitive   = true
}

# Database Users
output "admin_username" {
  description = "Admin username"
  value       = mongodbatlas_database_user.admin.username
}

output "app_username" {
  description = "Application username"
  value       = mongodbatlas_database_user.app_user.username
}

# Network
output "network_container_id" {
  description = "MongoDB Atlas Network Container ID"
  value       = try(mongodbatlas_network_container.aws.container_id, null)
}

output "vpc_peering_connection_id" {
  description = "VPC Peering Connection ID"
  value       = try(mongodbatlas_network_peering.aws.connection_id, null)
}

output "privatelink_endpoint_id" {
  description = "PrivateLink Endpoint ID"
  value       = try(mongodbatlas_privatelink_endpoint.aws.private_link_id, null)
}

# Backup
output "backup_enabled" {
  description = "Backup enabled status"
  value       = mongodbatlas_cluster.main.backup_enabled
}

output "pit_enabled" {
  description = "Point-in-time recovery enabled status"
  value       = mongodbatlas_cluster.main.pit_enabled
}

# AWS Secrets Manager
output "secrets_manager_arn" {
  description = "AWS Secrets Manager ARN for MongoDB credentials"
  value       = aws_secretsmanager_secret.mongodb_credentials.arn
}

# For use in applications
output "connection_details" {
  description = "Connection details for applications (non-sensitive)"
  value = {
    cluster_name = mongodbatlas_cluster.main.name
    mongodb_version = mongodbatlas_cluster.main.mongo_db_version
    region = var.aws_region
    project_id = mongodbatlas_project.main.id
  }
}
```

---

## Modules Réutilisables

### Module MongoDB Cluster

```hcl
# modules/mongodb-cluster/main.tf
# Module réutilisable pour créer un cluster MongoDB Atlas

terraform {
  required_providers {
    mongodbatlas = {
      source  = "mongodb/mongodbatlas"
      version = "~> 1.15.0"
    }
  }
}

variable "project_id" {
  description = "MongoDB Atlas Project ID"
  type        = string
}

variable "cluster_name" {
  description = "Cluster name"
  type        = string
}

variable "cluster_tier" {
  description = "Cluster tier"
  type        = string
  default     = "M30"
}

variable "cloud_provider" {
  description = "Cloud provider"
  type        = string
  default     = "AWS"
}

variable "region" {
  description = "Cloud region"
  type        = string
}

variable "mongodb_version" {
  description = "MongoDB version"
  type        = string
  default     = "7.0"
}

variable "disk_size_gb" {
  description = "Disk size in GB"
  type        = number
  default     = 100
}

variable "backup_enabled" {
  description = "Enable backups"
  type        = bool
  default     = true
}

variable "auto_scaling_enabled" {
  description = "Enable auto-scaling"
  type        = bool
  default     = true
}

variable "labels" {
  description = "Labels for the cluster"
  type        = map(string)
  default     = {}
}

# Cluster
resource "mongodbatlas_cluster" "this" {
  project_id   = var.project_id
  name         = var.cluster_name
  cluster_type = "REPLICASET"

  provider_name               = var.cloud_provider
  provider_instance_size_name = var.cluster_tier
  mongo_db_major_version      = var.mongodb_version

  replication_specs {
    num_shards = 1

    regions_config {
      region_name     = var.region
      electable_nodes = 3
      priority        = 7
      read_only_nodes = 0
    }
  }

  disk_size_gb                    = var.disk_size_gb
  auto_scaling_disk_gb_enabled    = var.auto_scaling_enabled
  auto_scaling_compute_enabled    = var.auto_scaling_enabled
  auto_scaling_compute_scale_down_enabled = var.auto_scaling_enabled

  backup_enabled = var.backup_enabled
  pit_enabled    = var.backup_enabled
  cloud_backup   = var.backup_enabled

  advanced_configuration {
    javascript_enabled           = false
    minimum_enabled_tls_protocol = "TLS1_2"
  }

  dynamic "labels" {
    for_each = var.labels
    content {
      key   = labels.key
      value = labels.value
    }
  }
}

output "cluster_id" {
  value = mongodbatlas_cluster.this.cluster_id
}

output "connection_strings" {
  value     = mongodbatlas_cluster.this.connection_strings
  sensitive = true
}

output "state" {
  value = mongodbatlas_cluster.this.state_name
}
```

### Module Usage Example

```hcl
# environments/production/main.tf
# Utilisation du module

module "production_cluster" {
  source = "../../modules/mongodb-cluster"

  project_id      = mongodbatlas_project.main.id
  cluster_name    = "production-cluster"
  cluster_tier    = "M50"
  cloud_provider  = "AWS"
  region          = "EU_WEST_3"
  mongodb_version = "7.0"
  disk_size_gb    = 500

  backup_enabled       = true
  auto_scaling_enabled = true

  labels = {
    Environment = "Production"
    Team        = "Platform"
    CostCenter  = "Engineering"
  }
}

module "analytics_cluster" {
  source = "../../modules/mongodb-cluster"

  project_id      = mongodbatlas_project.main.id
  cluster_name    = "analytics-cluster"
  cluster_tier    = "M30"
  cloud_provider  = "AWS"
  region          = "EU_WEST_3"
  mongodb_version = "7.0"
  disk_size_gb    = 200

  backup_enabled       = true
  auto_scaling_enabled = true

  labels = {
    Environment = "Production"
    Team        = "Analytics"
    CostCenter  = "DataScience"
  }
}
```

---

## CI/CD Integration

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
# CI/CD Pipeline pour Terraform MongoDB Atlas

stages:
  - validate
  - plan
  - apply
  - destroy

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  TF_STATE_NAME: ${CI_ENVIRONMENT_NAME}
  TF_CACHE_KEY: ${CI_COMMIT_REF_SLUG}

cache:
  key: "${TF_CACHE_KEY}"
  paths:
    - ${TF_ROOT}/.terraform
    - ${TF_ROOT}/.terraform.lock.hcl

before_script:
  - cd ${TF_ROOT}
  - terraform --version
  - terraform init -backend-config="key=mongodb-atlas/${CI_ENVIRONMENT_NAME}/terraform.tfstate"

# Validation
validate:
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate
  only:
    - merge_requests
    - main

# Security Scan
security-scan:
  stage: validate
  image: aquasec/tfsec:latest
  script:
    - tfsec . --format json --out tfsec-report.json
  artifacts:
    reports:
      sast: tfsec-report.json
  allow_failure: true
  only:
    - merge_requests
    - main

# Plan - Development
plan:dev:
  stage: plan
  environment:
    name: development
  script:
    - terraform plan -var-file="environments/dev/terraform.tfvars" -out=plan.tfplan
    - terraform show -json plan.tfplan > plan.json
  artifacts:
    paths:
      - plan.tfplan
      - plan.json
    expire_in: 1 week
  only:
    - merge_requests

# Plan - Staging
plan:staging:
  stage: plan
  environment:
    name: staging
  script:
    - terraform plan -var-file="environments/staging/terraform.tfvars" -out=plan.tfplan
  artifacts:
    paths:
      - plan.tfplan
    expire_in: 1 week
  only:
    - main

# Plan - Production
plan:prod:
  stage: plan
  environment:
    name: production
  script:
    - terraform plan -var-file="environments/production/terraform.tfvars" -out=plan.tfplan
  artifacts:
    paths:
      - plan.tfplan
    expire_in: 1 week
  only:
    - main

# Apply - Development (automatic)
apply:dev:
  stage: apply
  environment:
    name: development
  script:
    - terraform apply -var-file="environments/dev/terraform.tfvars" -auto-approve
  dependencies:
    - plan:dev
  only:
    - merge_requests
  when: manual

# Apply - Staging (manual approval)
apply:staging:
  stage: apply
  environment:
    name: staging
  script:
    - terraform apply plan.tfplan
  dependencies:
    - plan:staging
  only:
    - main
  when: manual

# Apply - Production (manual approval + protection)
apply:prod:
  stage: apply
  environment:
    name: production
  script:
    - terraform apply plan.tfplan
  dependencies:
    - plan:prod
  only:
    - main
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual

# Destroy - Development
destroy:dev:
  stage: destroy
  environment:
    name: development
  script:
    - terraform destroy -var-file="environments/dev/terraform.tfvars" -auto-approve
  only:
    - main
  when: manual

# Cost Estimation (with Infracost)
cost-estimate:
  stage: plan
  image: infracost/infracost:latest
  script:
    - infracost breakdown --path plan.json --format table
  dependencies:
    - plan:dev
  artifacts:
    reports:
      dotenv: infracost.env
  only:
    - merge_requests
```

### GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform MongoDB Atlas

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  workflow_dispatch:

env:
  TF_VERSION: 1.6.0
  AWS_REGION: eu-west-3

jobs:
  terraform-validate:
    name: Terraform Validate
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: terraform init -backend=false

      - name: Terraform Validate
        run: terraform validate

  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    needs: terraform-validate

    strategy:
      matrix:
        environment: [dev, staging, production]

    env:
      TF_VAR_mongodb_atlas_public_key: ${{ secrets.MONGODB_ATLAS_PUBLIC_KEY }}
      TF_VAR_mongodb_atlas_private_key: ${{ secrets.MONGODB_ATLAS_PRIVATE_KEY }}
      TF_VAR_mongodb_atlas_org_id: ${{ secrets.MONGODB_ATLAS_ORG_ID }}

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="key=mongodb-atlas/${{ matrix.environment }}/terraform.tfstate"

      - name: Terraform Plan
        run: |
          terraform plan \
            -var-file="environments/${{ matrix.environment }}/terraform.tfvars" \
            -out=plan.tfplan

      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan-${{ matrix.environment }}
          path: plan.tfplan
          retention-days: 7

  terraform-apply:
    name: Terraform Apply
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    strategy:
      matrix:
        environment: [dev, staging, production]
      max-parallel: 1  # Apply sequentially

    environment:
      name: ${{ matrix.environment }}
      url: https://cloud.mongodb.com

    env:
      TF_VAR_mongodb_atlas_public_key: ${{ secrets.MONGODB_ATLAS_PUBLIC_KEY }}
      TF_VAR_mongodb_atlas_private_key: ${{ secrets.MONGODB_ATLAS_PRIVATE_KEY }}
      TF_VAR_mongodb_atlas_org_id: ${{ secrets.MONGODB_ATLAS_ORG_ID }}

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan-${{ matrix.environment }}

      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="key=mongodb-atlas/${{ matrix.environment }}/terraform.tfstate"

      - name: Terraform Apply
        run: terraform apply plan.tfplan
```

---

## Best Practices

### Checklist Terraform MongoDB Atlas

```yaml
# terraform-best-practices.yaml
---
code_organization:
  - ✅ Structure modulaire avec modules réutilisables
  - ✅ Séparation par environnement (dev/staging/prod)
  - ✅ Variables centralisées dans variables.tf
  - ✅ Outputs documentés
  - ✅ README.md à jour

state_management:
  - ✅ Remote state (S3, Terraform Cloud, Azure)
  - ✅ State locking avec DynamoDB
  - ✅ Encryption du state activée
  - ✅ Backup du state configuré
  - ✅ Workspaces pour multi-environnements

security:
  - ✅ Variables sensibles marquées sensitive = true
  - ✅ Credentials via variables d'environnement
  - ✅ Secrets dans AWS Secrets Manager/Vault
  - ✅ .gitignore pour terraform.tfvars
  - ✅ TLS 1.2 minimum requis
  - ✅ JavaScript disabled dans MongoDB

networking:
  - ✅ VPC Peering ou PrivateLink pour production
  - ✅ IP Whitelist restrictive
  - ✅ Security groups configurés
  - ✅ Routes configurées correctement

backup_and_dr:
  - ✅ Continuous backup activé
  - ✅ Point-in-time recovery enabled
  - ✅ Multiple retention policies
  - ✅ Cross-region backup copies
  - ✅ Export to S3 configuré
  - ✅ Restore testé régulièrement

monitoring:
  - ✅ Alertes configurées (CPU, disk, connections)
  - ✅ Notifications multi-canal (email, Slack, PagerDuty)
  - ✅ Integration avec outils tiers (Datadog, Prometheus)
  - ✅ Dashboards monitoring disponibles

high_availability:
  - ✅ Minimum 3 nodes en production
  - ✅ Multi-region pour DR
  - ✅ Auto-scaling activé
  - ✅ Appropriate cluster tier pour la charge

cicd:
  - ✅ Terraform validate dans CI
  - ✅ Terraform plan sur pull requests
  - ✅ Manual approval pour production
  - ✅ Cost estimation intégrée
  - ✅ Security scanning (tfsec, Checkov)

lifecycle:
  - ✅ prevent_destroy sur ressources critiques
  - ✅ ignore_changes pour auto-scaling
  - ✅ Depends_on explicites
  - ✅ Count/for_each pour resources conditionnelles

documentation:
  - ✅ README par module
  - ✅ Variables documentées
  - ✅ Outputs documentés
  - ✅ Exemples d'utilisation
  - ✅ Architecture diagrams
```

---

## Conclusion

Terraform avec MongoDB Atlas offre une solution puissante pour gérer l'infrastructure DBaaS de manière déclarative :

**Avantages clés :**
- **Infrastructure as Code** : Versioning, code review, reproductibilité
- **Multi-cloud** : Support AWS, GCP, Azure avec API unifiée
- **Automation** : CI/CD integration, GitOps workflows
- **State Management** : Tracking précis des ressources
- **Modules** : Réutilisabilité et standardisation

**Fonctionnalités couvertes :**
- Création et configuration de clusters
- VPC Peering et PrivateLink
- Database users et sécurité
- Backup et disaster recovery
- Monitoring et alerting
- Multi-region deployments

**Best Practices :**
- Remote state avec locking
- Modules réutilisables
- CI/CD avec validation automatique
- Secrets management approprié
- Documentation complète

Terraform simplifie considérablement la gestion de MongoDB Atlas en permettant de provisionner et maintenir des clusters de manière reproductible et automatisée, tout en maintenant une haute disponibilité et sécurité.

---


⏭️ [CI/CD et migrations de schéma](/18-devops-deploiement/08-cicd-migrations-schema.md)
