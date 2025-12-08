🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.4 Configuration Réseau et Sécurité

## Introduction

La configuration réseau et sécurité d'Atlas est **fondamentale** pour les déploiements production. Cette section couvre les stratégies d'isolation réseau, les options de connectivité privée (VPC Peering, PrivateLink), le contrôle d'accès réseau (IP Whitelisting), et la configuration du chiffrement. Pour les architectes cloud et les équipes DevOps, maîtriser ces concepts est essentiel pour construire des infrastructures sécurisées, conformes et performantes.

### 🎯 Objectifs de cette Section

- Comprendre les modèles de connectivité réseau Atlas
- Configurer l'isolation réseau avec VPC Peering et Private Endpoints
- Implémenter des stratégies de contrôle d'accès IP
- Configurer le chiffrement en transit et au repos
- Gérer les utilisateurs et les droits d'accès (Database Access)
- Appliquer les best practices de sécurité réseau

---

## 🌐 Modèles de Connectivité Réseau

### Vue d'Ensemble des Options

```
┌────────────────────────────────────────────────────────────────────────┐
│                   ATLAS NETWORK CONNECTIVITY OPTIONS                   │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  OPTION                 SECURITY    LATENCY    COST      COMPLEXITY    │
│  ───────────────────────────────────────────────────────────────────── │
│  1. Public Internet     ⭐⭐☆☆☆    Medium     Free      Low            │
│     (IP Whitelist)      Basic      Variable                            │
│                                                                        │
│  2. VPC Peering         ⭐⭐⭐⭐☆   Low        Free*     Medium        │
│     (Private Network)   High       <2ms                                │
│                                                                        │
│  3. Private Endpoint    ⭐⭐⭐⭐⭐   Low        $$        High         │
│     (PrivateLink/PSC)   Highest    <1ms                                │
│                                                                        │
│  * VPC Peering: Free for same-region, data transfer charges apply      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Architecture Comparée

```
┌────────────────────────────────────────────────────────────────────────┐
│                    PUBLIC INTERNET CONNECTION                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   Your VPC (us-east-1)                 Internet                        │
│   ┌───────────────────┐                   │                            │
│   │                   │                   │                            │
│   │  App Server       │                   │                            │
│   │  (Private IP)     │                   │                            │
│   │       │           │                   │                            │
│   │       ▼           │                   │                            │
│   │  NAT Gateway ─────┼───────────────────┼─► Public IP                │
│   │    (Public IP)    │                   │                            │
│   └───────────────────┘                   │                            │
│                                           ▼                            │
│                              ┌─────────────────────────┐               │
│                              │   MongoDB Atlas         │               │
│                              │   (Public Endpoint)     │               │
│                              │   cluster0.xxxxx.       │               │
│                              │   mongodb.net           │               │
│                              └─────────────────────────┘               │
│                                                                        │
│  ⚠️ SECURITY CONCERNS:                                                 │
│  • Traffic exposed to Internet (even if encrypted)                     │
│  • Requires IP Whitelisting (can be brittle)                           │
│  • Vulnerable to DDoS on NAT Gateway                                   │
│  • Higher latency (routing through Internet)                           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────┐
│                         VPC PEERING CONNECTION                        │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   Your VPC (us-east-1)         VPC Peering      Atlas VPC (us-east-1) │
│   ┌───────────────────┐      Connection       ┌─────────────────────┐ │
│   │                   │    ◄──────────────►   │                     │ │
│   │  App Server       │    Private Network    │  MongoDB Atlas      │ │
│   │  10.0.1.50        │                       │  Cluster            │ │
│   │       │           │                       │  10.8.0.5           │ │
│   │       │           │                       │                     │ │
│   │       └───────────┼───────────────────────┼─► Direct Private    │ │
│   │     Route Table   │                       │     Connection      │ │
│   │  10.8.0.0/16 →    │                       │                     │ │
│   │  Peering          │                       │                     │ │
│   └───────────────────┘                       └─────────────────────┘ │
│                                                                       │
│  ✅ BENEFITS:                                                         │
│  • Traffic stays on private AWS/Azure/GCP backbone                    │
│  • No Internet exposure                                               │
│  • Lower latency (<2ms same region)                                   │
│  • Free data transfer (same region)                                   │
│  • No IP Whitelisting needed                                          │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────┐
│                    PRIVATE ENDPOINT (PrivateLink)                     │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   Your VPC (us-east-1)                        Atlas (us-east-1)       │
│   ┌───────────────────────────┐         ┌─────────────────────────┐   │
│   │                           │         │                         │   │
│   │  App Server               │         │  MongoDB Atlas          │   │
│   │  10.0.1.50                │         │  Cluster                │   │
│   │       │                   │         │                         │   │
│   │       ▼                   │         │                         │   │
│   │  Private Endpoint         │         │  Endpoint Service       │   │
│   │  (Interface Endpoint)     │◄────────┤  (PrivateLink)          │   │
│   │  10.0.1.100               │         │                         │   │
│   │       │                   │         │                         │   │
│   │       └───────────────────┼─────────┼─► cluster0-pl.xxxxx.    │   │
│   │     DNS: cluster0-pl...   │         │     mongodb.net         │   │
│   │     Resolves to:          │         │                         │   │
│   │     10.0.1.100            │         │                         │   │
│   └───────────────────────────┘         └─────────────────────────┘   │
│                                                                       │
│  ✅ BENEFITS:                                                         │
│  • Highest security (no VPC peering, no routes)                       │
│  • Traffic never leaves VPC                                           │
│  • DNS resolves to private IP                                         │
│  • Supports cross-account access                                      │
│  • Granular security groups                                           │
│                                                                       │
│  💰 COST: ~$0.01/GB + $0.01/hour per endpoint (~$7-10/month)          │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🔐 IP Access List (Whitelisting)

### Concept et Fonctionnement

L'**IP Access List** est le mécanisme de contrôle d'accès réseau de base d'Atlas. Il définit quelles adresses IP ou plages CIDR peuvent se connecter au cluster.

```
┌────────────────────────────────────────────────────────────────────┐
│                      IP ACCESS LIST WORKFLOW                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  Incoming Connection                                               │
│  from 203.0.113.45                                                 │
│         │                                                          │
│         ▼                                                          │
│  ┌──────────────────┐                                              │
│  │ Atlas Firewall   │                                              │
│  │ (IP Access List) │                                              │
│  └──────────────────┘                                              │
│         │                                                          │
│         ├──► Check 1: Is IP in Access List?                        │
│         │    ├─ YES → Continue                                     │
│         │    └─ NO  → ❌ Reject (Connection Refused)               │
│         │                                                          │
│         ├──► Check 2: TLS Handshake                                │
│         │    └─ Valid TLS 1.2+ Certificate?                        │
│         │                                                          │
│         ├──► Check 3: Authentication                               │
│         │    └─ Valid username/password or certificate?            │
│         │                                                          │
│         └──► ✅ Connection Established                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Configuration et Best Practices

```yaml
# IP Access List Configuration Examples

# ❌ BAD PRACTICE: Allow all (0.0.0.0/0)
ip_access_list:
  - cidr_block: "0.0.0.0/0"
    comment: "Allow from anywhere"

# ⚠️ ACCEPTABLE FOR DEV: Temporary access
ip_access_list:
  - cidr_block: "0.0.0.0/0"
    comment: "TEMPORARY - Remove after testing"
    delete_after_date: "2025-12-31"

# ✅ GOOD PRACTICE: Specific IPs/ranges
ip_access_list:
  # Office IPs
  - cidr_block: "203.0.113.0/24"
    comment: "Company Office Network"

  # AWS NAT Gateway (Production)
  - cidr_block: "34.192.0.50/32"
    comment: "Production NAT Gateway us-east-1"

  # Azure NAT Gateway (Staging)
  - cidr_block: "52.226.0.100/32"
    comment: "Staging NAT Gateway eastus"

  # CI/CD Pipeline
  - cidr_block: "185.199.108.0/22"
    comment: "GitHub Actions IP Range"

  # VPN
  - cidr_block: "10.100.0.0/16"
    comment: "Corporate VPN Range"

# ✅ BEST PRACTICE: Use VPC Peering or Private Endpoint
# No IP whitelist needed when using private connectivity
```

### Stratégies par Environnement

```
┌──────────────────────────────────────────────────────────────────────┐
│              IP ACCESS LIST STRATEGIES BY ENVIRONMENT                │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ENVIRONMENT    STRATEGY                      EXAMPLE                │
│  ──────────────────────────────────────────────────────────────────  │
│  Development    Permissive (temporary)        • 0.0.0.0/0 with TTL   │
│                                               • Delete after 7 days  │
│                 OR Developer IPs              • Individual /32s      │
│                                                                      │
│  Staging        NAT Gateway + CI/CD           • NAT Gateway IP       │
│                                               • GitHub Actions IPs   │
│                                               • Office network       │
│                                                                      │
│  Production     ❌ NO Public Access           • VPC Peering          │
│                 Use Private Connectivity      • OR PrivateLink       │
│                                               • OR specific NAT IPs  │
│                                                                      │
│  DR/Backup      Backup tools only             • MongoDB Ops Mgr IP   │
│                                               • Backup service IPs   │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### Automation avec Terraform

```hcl
# Terraform: IP Access List Management
resource "mongodbatlas_project_ip_access_list" "production" {
  project_id = var.atlas_project_id

  # Production NAT Gateways (AWS)
  cidr_block = "34.192.0.50/32"
  comment    = "Production NAT Gateway us-east-1a"
}

resource "mongodbatlas_project_ip_access_list" "production_backup" {
  project_id = var.atlas_project_id
  cidr_block = "34.192.1.75/32"
  comment    = "Production NAT Gateway us-east-1b (backup)"
}

# GitHub Actions IP ranges
resource "mongodbatlas_project_ip_access_list" "github_actions" {
  for_each = toset([
    "185.199.108.0/22",
    "140.82.112.0/20",
    "143.55.64.0/20"
  ])

  project_id = var.atlas_project_id
  cidr_block = each.value
  comment    = "GitHub Actions CI/CD"
}

# Temporary developer access (expires in 24 hours)
resource "mongodbatlas_project_ip_access_list" "temp_developer" {
  project_id        = var.atlas_project_id
  cidr_block        = "203.0.113.100/32"
  comment           = "Temporary - John Doe debugging"
  delete_after_date = timeadd(timestamp(), "24h")
}
```

---

## 🔗 VPC Peering

VPC Peering crée une connexion réseau privée entre votre VPC et le VPC d'Atlas, permettant une communication sécurisée sans passer par Internet.

### Architecture VPC Peering

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      VPC PEERING ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   YOUR INFRASTRUCTURE                        MONGODB ATLAS              │
│   ────────────────────                       ─────────────              │
│                                                                         │
│   AWS Account: 123456789012                  AWS Account: MongoDB       │
│   Region: us-east-1                          Region: us-east-1          │
│   ┌─────────────────────────┐               ┌──────────────────────┐    │
│   │ Your VPC                │               │ Atlas VPC            │    │
│   │ CIDR: 10.0.0.0/16       │               │ CIDR: 10.8.0.0/16    │    │
│   │                         │               │                      │    │
│   │ ┌─────────────────────┐ │               │ ┌──────────────────┐ │    │
│   │ │ Subnet: 10.0.1.0/24 │ │               │ │ Atlas Cluster    │ │    │
│   │ │                     │ │               │ │ Primary:         │ │    │
│   │ │ ┌─────────────────┐ │ │   Peering     │ │  10.8.0.5:27017  │ │    │
│   │ │ │ App Server      │ │ │ Connection    │ │ Secondary:       │ │    │
│   │ │ │ 10.0.1.50       │ │ │◄──────────►   │ │  10.8.0.6:27017  │ │    │
│   │ │ │                 │ │ │   Private     │ │ Secondary:       │ │    │
│   │ │ │ Connects to:    │ │ │   Network     │ │  10.8.0.7:27017  │ │    │
│   │ │ │ 10.8.0.5:27017  │ │ │               │ └──────────────────┘ │    │
│   │ │ └─────────────────┘ │ │               │                      │    │
│   │ └─────────────────────┘ │               └──────────────────────┘    │
│   │                         │                                           │
│   │ Route Table:            │                                           │
│   │ • 10.8.0.0/16 → Peering │                                           │
│   │ • 0.0.0.0/0 → IGW       │                                           │
│   └─────────────────────────┘                                           │
│                                                                         │
│   Security Group Rules:                                                 │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │ Inbound:  None needed (we initiate)                             │   │
│   │ Outbound: TCP 27017 to 10.8.0.0/16 (Atlas VPC CIDR)             │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Configuration Steps Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                 VPC PEERING SETUP WORKFLOW                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  PHASE 1: PREREQUISITES                                            │
│  ───────────────────────                                           │
│  ☐ Atlas cluster on dedicated tier (M10+)                          │
│  ☐ VPC CIDR ranges don't overlap                                   │
│  ☐ Same cloud provider and region                                  │
│  ☐ AWS/Azure/GCP account credentials ready                         │
│                                                                    │
│  PHASE 2: ATLAS CONFIGURATION                                      │
│  ─────────────────────────────                                     │
│  1. Atlas UI → Project → Network Access → Peering                  │
│  2. Click "New Peering Connection"                                 │
│  3. Provide:                                                       │
│     • Your AWS Account ID                                          │
│     • Your VPC ID                                                  │
│     • Your VPC CIDR                                                │
│     • Region                                                       │
│  4. Atlas creates peering request                                  │
│  5. Atlas provides Peering Connection ID                           │
│                                                                    │
│  PHASE 3: CLOUD PROVIDER CONFIGURATION                             │
│  ──────────────────────────────────────                            │
│  AWS:                                                              │
│  1. AWS Console → VPC → Peering Connections                        │
│  2. Accept pending peering request                                 │
│  3. Add route to route table:                                      │
│     • Destination: Atlas VPC CIDR (e.g., 10.8.0.0/16)              │
│     • Target: Peering Connection ID                                │
│  4. Update Security Groups:                                        │
│     • Allow outbound TCP 27017 to Atlas CIDR                       │
│                                                                    │
│  PHASE 4: VALIDATION                                               │
│  ────────────────────                                              │
│  ☐ Peering status: "Available" in Atlas                            │
│  ☐ Routes configured in VPC route tables                           │
│  ☐ Security groups allow MongoDB traffic                           │
│  ☐ Test connection from app server:                                │
│    mongosh "mongodb://10.8.0.5:27017"                              │
│                                                                    │
│  ⏱️ SETUP TIME: ~15-30 minutes                                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Provider-Specific Configuration

#### AWS VPC Peering

```hcl
# Terraform: AWS VPC Peering with Atlas
resource "mongodbatlas_network_peering" "aws_peer" {
  project_id     = var.atlas_project_id
  container_id   = mongodbatlas_network_container.aws.id

  provider_name  = "AWS"

  # Your AWS VPC details
  accepter_region_name   = "us-east-1"
  aws_account_id         = "123456789012"
  vpc_id                 = "vpc-0a1b2c3d4e5f6g7h8"
  route_table_cidr_block = "10.0.0.0/16"
}

# Accept the peering connection in AWS
resource "aws_vpc_peering_connection_accepter" "atlas" {
  vpc_peering_connection_id = mongodbatlas_network_peering.aws_peer.connection_id
  auto_accept               = true

  tags = {
    Name = "Atlas-VPC-Peering"
    Environment = "Production"
  }
}

# Add route to Atlas VPC
resource "aws_route" "atlas_route" {
  route_table_id            = aws_route_table.main.id
  destination_cidr_block    = "10.8.0.0/16"  # Atlas VPC CIDR
  vpc_peering_connection_id = mongodbatlas_network_peering.aws_peer.connection_id
}

# Security Group Rule
resource "aws_security_group_rule" "allow_mongodb" {
  type              = "egress"
  from_port         = 27017
  to_port           = 27017
  protocol          = "tcp"
  cidr_blocks       = ["10.8.0.0/16"]  # Atlas VPC CIDR
  security_group_id = aws_security_group.app_servers.id
  description       = "Allow MongoDB to Atlas via VPC Peering"
}
```

#### Azure VNet Peering

```hcl
# Terraform: Azure VNet Peering with Atlas
resource "mongodbatlas_network_peering" "azure_peer" {
  project_id     = var.atlas_project_id
  container_id   = mongodbatlas_network_container.azure.id

  provider_name  = "AZURE"

  # Your Azure VNet details
  azure_directory_id   = "a1b2c3d4-e5f6-7890-a1b2-c3d4e5f67890"
  azure_subscription_id = "b2c3d4e5-f678-90a1-b2c3-d4e5f6789012"
  resource_group_name   = "production-rg"
  vnet_name             = "production-vnet"

  atlas_cidr_block      = "10.8.0.0/16"
}

# Accept peering in Azure
resource "azurerm_virtual_network_peering" "atlas_to_azure" {
  name                      = "atlas-to-production"
  resource_group_name       = "production-rg"
  virtual_network_name      = "production-vnet"
  remote_virtual_network_id = mongodbatlas_network_peering.azure_peer.atlas_id

  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
}
```

#### GCP VPC Peering

```hcl
# Terraform: GCP VPC Peering with Atlas
resource "mongodbatlas_network_peering" "gcp_peer" {
  project_id     = var.atlas_project_id
  container_id   = mongodbatlas_network_container.gcp.id

  provider_name  = "GCP"

  # Your GCP VPC details
  gcp_project_id = "my-gcp-project-123456"
  network_name   = "production-vpc"

  atlas_cidr_block = "10.8.0.0/16"
}

# Accept peering in GCP
resource "google_compute_network_peering" "atlas_to_gcp" {
  name         = "atlas-to-production"
  network      = "projects/my-gcp-project-123456/global/networks/production-vpc"
  peer_network = mongodbatlas_network_peering.gcp_peer.atlas_gcp_project_id

  export_custom_routes = false
  import_custom_routes = false
}
```

### Troubleshooting VPC Peering

```
┌────────────────────────────────────────────────────────────────────┐
│                VPC PEERING COMMON ISSUES                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ISSUE                          SOLUTION                           │
│  ───────────────────────────────────────────────────────────────   │
│  Peering status "Pending"       → Accept peering in cloud console  │
│                                                                    │
│  Connection timeouts            → Check security groups/NACLs      │
│                                 → Verify routes in route tables    │
│                                 → Confirm CIDR doesn't overlap     │
│                                                                    │
│  DNS resolution fails           → Use private IPs directly         │
│                                 → Atlas connection string has IPs  │
│                                                                    │
│  "No route to host"             → Add route to Atlas CIDR          │
│                                 → Destination: 10.8.0.0/16         │
│                                 → Target: pcx-xxxxx (peering ID)   │
│                                                                    │
│  Works from one subnet,         → Route table not associated       │
│  not others                     → Add route to ALL relevant tables │
│                                                                    │
│  High latency despite peering   → Check if same region             │
│                                 → Cross-region peering adds latency│
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Private Endpoints (AWS PrivateLink / Azure Private Link / GCP PSC)

Private Endpoints offrent le **plus haut niveau de sécurité** en créant une interface réseau privée dans votre VPC qui route le trafic vers Atlas sans jamais quitter le réseau du cloud provider.

### Architecture Private Endpoint

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  AWS PRIVATELINK ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   YOUR VPC (us-east-1)                                                  │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                                                                  │  │
│   │   Availability Zone 1a          Availability Zone 1b             │  │
│   │   ┌────────────────────┐        ┌────────────────────┐           │  │
│   │   │ Private Subnet     │        │ Private Subnet     │           │  │
│   │   │ 10.0.1.0/24        │        │ 10.0.2.0/24        │           │  │
│   │   │                    │        │                    │           │  │
│   │   │ ┌────────────────┐ │        │ ┌────────────────┐ │           │  │
│   │   │ │ App Server     │ │        │ │ App Server     │ │           │  │
│   │   │ │ 10.0.1.50      │ │        │ │ 10.0.2.50      │ │           │  │
│   │   │ └────────┬───────┘ │        │ └────────┬───────┘ │           │  │
│   │   │          │         │        │          │         │           │  │
│   │   │          ▼         │        │          ▼         │           │  │
│   │   │ ┌────────────────┐ │        │ ┌────────────────┐ │           │  │
│   │   │ │ VPC Endpoint   │ │        │ │ VPC Endpoint   │ │           │  │
│   │   │ │ Interface (ENI)│ │        │ │ Interface (ENI)│ │           │  │
│   │   │ │ 10.0.1.100     │ │        │ │ 10.0.2.100     │ │           │  │
│   │   │ └────────┬───────┘ │        │ └────────┬───────┘ │           │  │
│   │   └──────────┼─────────┘        └──────────┼─────────┘           │  │
│   │              │                             │                     │  │
│   └──────────────┼─────────────────────────────┼─────────────────────┘  │
│                  │                             │                        │
│                  │        AWS PrivateLink      │                        │
│                  │        (Private Network)    │                        │
│                  ▼                             ▼                        │
│   ┌──────────────────────────────────────────────────────────────────┐  │
│   │                    ATLAS ENDPOINT SERVICE                        │  │
│   │                    (MongoDB-managed)                             │  │
│   │   ┌───────────────────────────────────────────────────────────┐  │  │
│   │   │  MongoDB Atlas Cluster                                    │  │  │
│   │   │  • Primary:   Internal endpoint                           │  │  │
│   │   │  • Secondary: Internal endpoint                           │  │  │
│   │   │  • Secondary: Internal endpoint                           │  │  │
│   │   └───────────────────────────────────────────────────────────┘  │  │
│   └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│   DNS RESOLUTION:                                                       │
│   cluster0-pl.xxxxx.mongodb.net → 10.0.1.100, 10.0.2.100                │
│   (Private IPs in YOUR VPC)                                             │
│                                                                         │
│   ✅ Traffic NEVER leaves your VPC                                      │
│   ✅ No VPC Peering required (simpler)                                  │
│   ✅ Works across AWS accounts                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Comparaison VPC Peering vs Private Endpoint

```
┌────────────────────────────────────────────────────────────────────────┐
│             VPC PEERING vs PRIVATE ENDPOINT COMPARISON                 │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  CRITÈRE              VPC PEERING          PRIVATE ENDPOINT            │
│  ──────────────────────────────────────────────────────────────────────│
│  Setup Complexity     Medium               High                        │
│  Configuration        Both sides           Mostly Atlas side           │
│  Routing              Explicit routes      Automatic (DNS)             │
│  IP Management        Must track CIDRs     Managed by cloud provider   │
│  Cross-Account        Complex              Simple                      │
│  DNS Resolution       Manual (IPs)         Automatic (private DNS)     │
│  Security Groups      Needed               Needed (more granular)      │
│  Scalability          Limited by CIDR      Highly scalable             │
│  Cost                 Free*               $0.01/GB + $7-10/month       │
│  Latency              <2ms                 <1ms                        │
│  Multi-Region         Requires multiple    One per region              │
│                       peerings                                         │
│  Maintenance          Manual updates       Auto-managed                │
│                                                                        │
│  RECOMMENDATION:                                                       │
│  • Simple setup, same account    → VPC Peering                         │
│  • Multi-account, complex arch   → Private Endpoint                    │
│  • Highest security requirements → Private Endpoint                    │
│  • Budget-conscious              → VPC Peering                         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Configuration Private Endpoint (AWS)

```hcl
# Terraform: AWS PrivateLink Configuration
resource "mongodbatlas_privatelink_endpoint" "atlas" {
  project_id    = var.atlas_project_id
  provider_name = "AWS"
  region        = "us-east-1"
}

# Create VPC Endpoint in your AWS account
resource "aws_vpc_endpoint" "atlas" {
  vpc_id             = aws_vpc.main.id
  service_name       = mongodbatlas_privatelink_endpoint.atlas.endpoint_service_name
  vpc_endpoint_type  = "Interface"

  subnet_ids = [
    aws_subnet.private_1a.id,
    aws_subnet.private_1b.id,
  ]

  security_group_ids = [
    aws_security_group.atlas_endpoint.id
  ]

  private_dns_enabled = true

  tags = {
    Name = "Atlas-PrivateLink-Endpoint"
  }
}

# Security Group for the endpoint
resource "aws_security_group" "atlas_endpoint" {
  name        = "atlas-privatelink-endpoint"
  description = "Security group for Atlas PrivateLink endpoint"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "MongoDB from app servers"
    from_port   = 27017
    to_port     = 27017
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Complete the Private Endpoint setup in Atlas
resource "mongodbatlas_privatelink_endpoint_service" "atlas" {
  project_id          = var.atlas_project_id
  private_link_id     = mongodbatlas_privatelink_endpoint.atlas.id
  endpoint_service_id = aws_vpc_endpoint.atlas.id
  provider_name       = "AWS"
}

# Output the private connection string
output "atlas_private_connection_string" {
  value = "mongodb://cluster0-pl-0.xxxxx.mongodb.net:27017,cluster0-pl-1.xxxxx.mongodb.net:27017,cluster0-pl-2.xxxxx.mongodb.net:27017/"
  sensitive = true
}
```

---

## 🔐 Database Access (Users and Roles)

### Architecture Database Access

```
┌───────────────────────────────────────────────────────────────────┐
│                   DATABASE ACCESS ARCHITECTURE                    │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  AUTHENTICATION LAYER                                             │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                                                              │ │
│  │  Methods:                                                    │ │
│  │  • SCRAM-SHA-256 (default, password-based)                   │ │
│  │  • X.509 Certificate                                         │ │
│  │  • AWS IAM (for Atlas on AWS)                                │ │
│  │  • LDAP (Enterprise)                                         │ │
│  │                                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  AUTHORIZATION LAYER                                              │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                                                              │ │
│  │  Built-in Roles:                                             │ │
│  │  • atlasAdmin        (full admin)                            │ │
│  │  • readWriteAnyDatabase                                      │ │
│  │  • readAnyDatabase                                           │ │
│  │  • dbAdmin                                                   │ │
│  │  • read, readWrite (per database)                            │ │
│  │                                                              │ │
│  │  Custom Roles:                                               │ │
│  │  • Fine-grained permissions                                  │ │
│  │  • Action-level control                                      │ │
│  │                                                              │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  DATABASE ACCESS                                                  │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │  Collections, Documents, Indexes                             │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Built-in Roles Reference

```
┌────────────────────────────────────────────────────────────────────────┐
│                    MONGODB ATLAS BUILT-IN ROLES                        │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ROLE                      SCOPE           PERMISSIONS                 │
│  ───────────────────────────────────────────────────────────────────── │
│  atlasAdmin                Cluster         • Full administrative       │
│                                            • User management           │
│                                            • All databases             │
│                                            ❌ USE WITH CAUTION         │
│                                                                        │
│  readWriteAnyDatabase      Cluster         • Read/Write all databases  │
│                                            • Create collections        │
│                                            • Create indexes            │
│                                                                        │
│  readAnyDatabase           Cluster         • Read-only all databases   │
│                                            • Good for analytics        │
│                                                                        │
│  dbAdminAnyDatabase        Cluster         • Manage indexes            │
│                                            • Manage collections        │
│                                            • View stats                │
│                                                                        │
│  readWrite                 Database        • Read/Write specific DB    │
│  (Recommended for apps)                    • Create collections        │
│                                            • CANNOT create users       │
│                                                                        │
│  read                      Database        • Read-only specific DB     │
│  (Recommended for BI)                      • Good for reporting        │
│                                                                        │
│  dbAdmin                   Database        • Manage DB structure       │
│                                            • Indexes, collections      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Best Practice User Management

```yaml
# Database Users Configuration (Principle of Least Privilege)

# ❌ BAD: Admin user for application
users:
  - username: "app_user"
    password: "xxxxxxxxxx"
    roles:
      - role: "atlasAdmin"
        database: "admin"
    # TOO MUCH PRIVILEGE!

# ✅ GOOD: Specific permissions per use case
users:
  # Application user (production)
  - username: "prod_app_user"
    password: "secure_password_here"
    auth_database: "admin"
    roles:
      - role: "readWrite"
        database: "production_db"
    scopes:
      - type: "CLUSTER"
        name: "production-cluster"
    labels:
      - key: "environment"
        value: "production"
      - key: "application"
        value: "api-backend"

  # Read-only user for analytics/BI
  - username: "analytics_reader"
    password: "secure_password_here"
    auth_database: "admin"
    roles:
      - role: "read"
        database: "production_db"
    scopes:
      - type: "CLUSTER"
        name: "production-cluster"

  # DBA user (administrative tasks)
  - username: "dba_admin"
    password: "secure_password_here"
    auth_database: "admin"
    roles:
      - role: "dbAdmin"
        database: "production_db"
      - role: "clusterMonitor"
        database: "admin"
    scopes:
      - type: "CLUSTER"
        name: "production-cluster"

  # Backup user (for external backup tools)
  - username: "backup_user"
    password: "secure_password_here"
    auth_database: "admin"
    roles:
      - role: "backup"
        database: "admin"
    scopes:
      - type: "CLUSTER"
        name: "production-cluster"
```

### Terraform Database Users

```hcl
# Terraform: Database User Management
resource "mongodbatlas_database_user" "app_user" {
  username           = "prod_app_user"
  password           = var.app_user_password  # From secrets manager
  project_id         = var.atlas_project_id
  auth_database_name = "admin"

  roles {
    role_name     = "readWrite"
    database_name = "production_db"
  }

  scopes {
    name = "production-cluster"
    type = "CLUSTER"
  }

  labels {
    key   = "environment"
    value = "production"
  }
}

# Read-only user for analytics
resource "mongodbatlas_database_user" "analytics_user" {
  username           = "analytics_reader"
  password           = var.analytics_password
  project_id         = var.atlas_project_id
  auth_database_name = "admin"

  roles {
    role_name     = "read"
    database_name = "production_db"
  }

  scopes {
    name = "production-cluster"
    type = "CLUSTER"
  }
}

# X.509 Certificate authentication
resource "mongodbatlas_database_user" "cert_user" {
  username           = "CN=app-service,OU=Engineering,O=MyCompany"
  project_id         = var.atlas_project_id
  auth_database_name = "$external"
  x509_type          = "CUSTOMER"

  roles {
    role_name     = "readWrite"
    database_name = "production_db"
  }
}
```

### Password Security Best Practices

```
┌────────────────────────────────────────────────────────────────────┐
│                  DATABASE USER SECURITY BEST PRACTICES             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. PASSWORD COMPLEXITY                                            │
│     ✅ Minimum 12 characters                                       │
│     ✅ Mix of uppercase, lowercase, numbers, symbols               │
│     ✅ Use password generator                                      │
│     ❌ Avoid dictionary words                                      │
│                                                                    │
│  2. PASSWORD STORAGE                                               │
│     ✅ Store in secrets manager (AWS Secrets Manager, Vault)       │
│     ✅ Rotate regularly (90 days)                                  │
│     ✅ Never commit to version control                             │
│     ❌ Don't hardcode in applications                              │
│                                                                    │
│  3. PASSWORD ROTATION                                              │
│     ✅ Automated rotation via CI/CD                                │
│     ✅ Zero-downtime rotation (dual credentials temporarily)       │
│     ✅ Audit log of all changes                                    │
│                                                                    │
│  4. ALTERNATIVE: CERTIFICATE AUTHENTICATION                        │
│     ✅ X.509 certificates (no passwords)                           │
│     ✅ AWS IAM authentication (for AWS deployments)                │
│     ✅ LDAP integration (Enterprise)                               │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🔒 Encryption

### Encryption in Transit (TLS/SSL)

```
┌────────────────────────────────────────────────────────────────────┐
│                     ENCRYPTION IN TRANSIT                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  DEFAULT CONFIGURATION:                                            │
│  • TLS 1.2+ required (TLS 1.0/1.1 disabled)                        │
│  • Perfect Forward Secrecy (PFS) enabled                           │
│  • Strong cipher suites only                                       │
│  • Certificate validation enforced                                 │
│                                                                    │
│  CONNECTION STRING OPTIONS:                                        │
│  mongodb+srv://cluster0.xxxxx.mongodb.net/?tls=true&               │
│    tlsAllowInvalidCertificates=false&                              │
│    tlsAllowInvalidHostnames=false                                  │
│                                                                    │
│  ✅ RECOMMENDED (default, secure):                                 │
│  • tls=true                                                        │
│  • Verify certificates                                             │
│  • Verify hostnames                                                │
│                                                                    │
│  ⚠️ DEVELOPMENT ONLY (insecure):                                   │
│  • tlsAllowInvalidCertificates=true                                │
│  • Only for local/test environments                                │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Encryption at Rest

```
┌────────────────────────────────────────────────────────────────────┐
│                     ENCRYPTION AT REST                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ATLAS DEFAULT:                                                    │
│  • Enabled automatically on all clusters (M10+)                    │
│  • MongoDB-managed keys (no additional cost)                       │
│  • AES-256 encryption                                              │
│  • Encrypts:                                                       │
│    - Data files                                                    │
│    - Indexes                                                       │
│    - Backups                                                       │
│    - Snapshots                                                     │
│                                                                    │
│  BRING YOUR OWN KEY (BYOK):                                        │
│  • Use your own KMS keys                                           │
│  • Supported providers:                                            │
│    - AWS KMS                                                       │
│    - Azure Key Vault                                               │
│    - Google Cloud KMS                                              │
│  • You control key lifecycle                                       │
│  • Additional compliance (HIPAA, PCI-DSS)                          │
│  • Additional cost: Key management overhead                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### BYOK Configuration (AWS KMS)

```hcl
# Terraform: Configure Encryption at Rest with AWS KMS
resource "mongodbatlas_encryption_at_rest" "aws_kms" {
  project_id = var.atlas_project_id

  aws_kms_config {
    enabled                = true
    customer_master_key_id = aws_kms_key.atlas.id
    region                 = "us-east-1"
    role_id                = mongodbatlas_cloud_provider_access_setup.aws.aws_config[0].atlas_aws_account_arn
  }
}

# AWS KMS Key
resource "aws_kms_key" "atlas" {
  description             = "MongoDB Atlas Encryption Key"
  deletion_window_in_days = 30
  enable_key_rotation     = true

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow Atlas to use the key"
        Effect = "Allow"
        Principal = {
          AWS = mongodbatlas_cloud_provider_access_setup.aws.aws_config[0].atlas_aws_account_arn
        }
        Action = [
          "kms:Decrypt",
          "kms:Encrypt",
          "kms:DescribeKey"
        ]
        Resource = "*"
      }
    ]
  })
}
```

---

## 🛡️ Network Security Best Practices

### Security Checklist

```
┌────────────────────────────────────────────────────────────────────┐
│              NETWORK SECURITY CHECKLIST (PRODUCTION)               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  NETWORK ISOLATION                                                 │
│  ☐ VPC Peering OR Private Endpoint configured                      │
│  ☐ NO public Internet connectivity (0.0.0.0/0 removed)             │
│  ☐ IP Access List minimal and specific                             │
│  ☐ Security groups properly configured                             │
│  ☐ Network ACLs reviewed                                           │
│                                                                    │
│  ENCRYPTION                                                        │
│  ☐ TLS 1.2+ enforced                                               │
│  ☐ Certificate validation enabled                                  │
│  ☐ Encryption at rest enabled (default)                            │
│  ☐ BYOK configured (if compliance requires)                        │
│                                                                    │
│  ACCESS CONTROL                                                    │
│  ☐ Least privilege principle applied                               │
│  ☐ Separate users per application/service                          │
│  ☐ No shared credentials                                           │
│  ☐ No admin users for applications                                 │
│  ☐ Password rotation policy (90 days)                              │
│  ☐ Consider X.509 certificate auth                                 │
│                                                                    │
│  MONITORING                                                        │
│  ☐ Access logs enabled                                             │
│  ☐ Alerts on failed login attempts                                 │
│  ☐ Alerts on new IP connections                                    │
│  ☐ Regular audit log review                                        │
│                                                                     │
│  COMPLIANCE                                                        │
│  ☐ Data residency requirements met                                 │
│  ☐ Backup encryption verified                                      │
│  ☐ Disaster recovery tested                                        │
│  ☐ Security documentation updated                                  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Defense in Depth Strategy

```
┌───────────────────────────────────────────────────────────────────┐
│                   DEFENSE IN DEPTH LAYERS                         │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  LAYER 7: APPLICATION                                             │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • Input validation                                           │ │
│  │ • SQL injection prevention (MongoDB = NoSQL but still...)    │ │
│  │ • Rate limiting                                              │ │
│  │ • API authentication (JWT, OAuth)                            │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  LAYER 6: DATABASE ACCESS                                         │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • Strong authentication (X.509, LDAP)                        │ │
│  │ • Least privilege RBAC                                       │ │
│  │ • Password rotation                                          │ │
│  │ • Failed login monitoring                                    │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  LAYER 5: ENCRYPTION                                              │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • TLS 1.2+ in transit                                        │ │
│  │ • AES-256 at rest                                            │ │
│  │ • BYOK (optional)                                            │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  LAYER 4: NETWORK ISOLATION                                       │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • VPC Peering or Private Endpoint                            │ │
│  │ • No public Internet access                                  │ │
│  │ • Private DNS resolution                                     │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  LAYER 3: FIREWALL                                                │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • IP Access List (minimal)                                   │ │
│  │ • Security Groups (port 27017 only)                          │ │
│  │ • Network ACLs                                               │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  LAYER 2: MONITORING & AUDITING                                   │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • Connection logs                                            │ │
│  │ • Query profiling                                            │ │
│  │ • Audit logs (Enterprise)                                    │ │
│  │ • Alerting on anomalies                                      │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                            ▼                                      │
│  LAYER 1: PHYSICAL & CLOUD SECURITY                               │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ • SOC 2 Type II certified datacenters                        │ │
│  │ • Physical access control                                    │ │
│  │ • Hardware encryption                                        │ │
│  │ • Multi-tenant isolation                                     │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

---

## 🏁 Résumé

### Points Clés

1. **Connectivité Réseau** : 3 options
   - Public Internet + IP Whitelist (simple, moins sécurisé)
   - VPC Peering (sécurisé, gratuit, setup moyen)
   - Private Endpoint (plus sécurisé, payant, setup complexe)

2. **IP Access List**
   - Toujours restreindre au maximum
   - ❌ Éviter 0.0.0.0/0 en production
   - ✅ Utiliser VPC Peering/Private Endpoint pour prod

3. **VPC Peering**
   - Connexion réseau privée entre VPCs
   - Pas d'exposition Internet
   - Gratuit (same-region), faible latence
   - Configuration bidirectionnelle requise

4. **Private Endpoint**
   - Sécurité maximale (PrivateLink/PSC)
   - Traffic ne quitte jamais le VPC
   - DNS automatique vers IPs privées
   - Coût supplémentaire mais recommandé pour enterprise

5. **Database Access**
   - Principe du moindre privilège
   - Un utilisateur par application/service
   - Rotation des mots de passe
   - Considérer X.509 pour production

6. **Encryption**
   - TLS 1.2+ par défaut (transit)
   - AES-256 par défaut (repos)
   - BYOK pour compliance stricte

### Recommandations par Environnement

| Environment | Connectivity | IP Whitelist | Encryption |
|-------------|-------------|--------------|------------|
| **Development** | Public Internet | Developer IPs, 0.0.0.0/0 temp | TLS default |
| **Staging** | VPC Peering | NAT Gateway IPs | TLS + Encryption at rest |
| **Production** | Private Endpoint | ❌ None (private only) | TLS + BYOK |

### Coûts Réseau

- **IP Whitelist** : Gratuit
- **VPC Peering** : Gratuit (same-region), data transfer charges cross-region
- **Private Endpoint** : ~$7-10/mois + $0.01/GB

---


⏭️ [Connexion à Atlas](/14-mongodb-atlas/05-connexion-atlas.md)
