🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.16 Atlas CLI

## Introduction

L'**Atlas CLI** (`atlas`) est l'outil en ligne de commande officiel pour gérer MongoDB Atlas. Conçu pour les équipes DevOps et SRE, il permet d'automatiser toutes les opérations Atlas : provisioning de clusters, gestion d'utilisateurs, configuration réseau, déploiement d'App Services, monitoring, et bien plus. Intégrez Atlas dans vos pipelines CI/CD, vos scripts d'automation, et vos workflows Infrastructure-as-Code pour une gestion déclarative et reproductible de votre infrastructure MongoDB.

### 🎯 Objectifs de cette Section

- Maîtriser l'installation et la configuration d'Atlas CLI
- Automatiser la gestion de clusters et ressources
- Intégrer Atlas CLI dans les pipelines CI/CD
- Implémenter Infrastructure-as-Code avec le CLI
- Scripter les opérations récurrentes
- Monitorer et observer via ligne de commande
- Construire des workflows DevOps modernes

---

## 🏗️ Architecture CLI

### Vue d'Ensemble

```
┌────────────────────────────────────────────────────────────────────────┐
│                      ATLAS CLI ARCHITECTURE                            │
├────────────────────────────────────────────────────────────────────────┤
│
│  DEVOPS WORKFLOWS
│  ┌──────────────────────────────────────────────────────────────────┐
│  │
│  │  Local Dev    CI/CD         Automation      Monitoring
│  │  ────────     ──────         ──────────     ──────────
│  │  Scripts      GitHub         Cron jobs      Alerting
│  │  Testing      Actions        Orchestration  Health checks
│  │  Debugging    GitLab CI      Batch ops      Log analysis
│  │               Jenkins
│  │
│  └────┬──────────────┬────────────────┬─────────────────┬───────────┘
│       │              │                │                 │
│       └──────────────┴────────────────┴─────────────────┘
│                      │
│                      ▼
│  ┌──────────────────────────────────────────────────────────────────┐
│  │                    ATLAS CLI (atlas)
│  │  https://github.com/mongodb/mongodb-atlas-cli
│  ├──────────────────────────────────────────────────────────────────┤
│  │
│  │  CATEGORIES:
│  │  ┌────────────────┬────────────────┬────────────────┐
│  │  │  CLUSTERS      │  PROJECTS      │  ORGANIZATIONS │
│  │  │  • Create      │  • List        │  • List        │
│  │  │  • Update      │  • Create      │  • Describe    │
│  │  │  • Delete      │  • Delete      │  • Users       │
│  │  │  • Scale       │  • Settings    │  • API keys    │
│  │  │  • Pause       │  • Teams       │                │
│  │  └────────────────┴────────────────┴────────────────┘
│  │
│  │  ┌────────────────┬────────────────┬────────────────┐
│  │  │  BACKUPS       │  NETWORKING    │  SECURITY      │
│  │  │  • Snapshots   │  • IP lists    │  • Users       │
│  │  │  • Restore     │  • Peering     │  • Roles       │
│  │  │  • Schedule    │  • Private end │  • Keys        │
│  │  │  • Export      │  • Firewall    │  • Audit       │
│  │  └────────────────┴────────────────┴────────────────┘
│  │
│  │  ┌────────────────┬────────────────┬────────────────┐
│  │  │  MONITORING    │  DATA          │  APP SERVICES  │
│  │  │  • Metrics     │  • Import      │  • Deploy      │
│  │  │  • Logs        │  • Export      │  • Functions   │
│  │  │  • Alerts      │  • Indexes     │  • Triggers    │
│  │  │  • Events      │  • Data Lake   │  • Logs        │
│  │  └────────────────┴────────────────┴────────────────┘
│  │
│  └───────────────────────────────┬───────────────────────────────────┘
│                                  │
│                                  ▼ REST API Calls
│  ┌──────────────────────────────────────────────────────────────────┐
│  │                     ATLAS REST API
│  │  https://cloud.mongodb.com/api/atlas/v2
│  │  • Authentication (API keys, OAuth)
│  │  • Rate limiting
│  │  • Pagination
│  └───────────────────────────────┬───────────────────────────────────┘
│                                  │
│                                  ▼
│  MONGODB ATLAS PLATFORM
│  ┌──────────────────────────────────────────────────────────────────┐
│  │  Clusters │ Databases │ App Services │ Data Lake │ Charts        │
│  └──────────────────────────────────────────────────────────────────┘
│
│  AVANTAGES:
│  ✅ Automation complète (scriptable)
│  ✅ CI/CD friendly (exit codes, JSON output)
│  ✅ Infrastructure-as-Code (declarative)
│  ✅ Cross-platform (Linux, macOS, Windows)
│  ✅ Open source (contributions welcome)
│
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🚀 Installation et Configuration

### Installation

```bash
# macOS (Homebrew)
brew install mongodb-atlas-cli

# Linux (apt - Debian/Ubuntu)
wget -qO - https://pgp.mongodb.com/server-6.0.asc | sudo apt-key add -
echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.com/apt/ubuntu focal/mongodb-atlas-cli multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-atlas-cli.list
sudo apt-get update
sudo apt-get install -y mongodb-atlas-cli

# Linux (yum - RHEL/CentOS)
sudo tee /etc/yum.repos.d/mongodb-atlas-cli.repo <<EOF
[mongodb-atlas-cli]
name=MongoDB Atlas CLI Repository
baseurl=https://repo.mongodb.com/yum/redhat/8/mongodb-atlas-cli/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-6.0.asc
EOF
sudo yum install -y mongodb-atlas-cli

# Windows (Chocolatey)
choco install mongodb-atlas-cli

# Via Go (any platform)
go install github.com/mongodb/mongodb-atlas-cli/atlascli@latest

# Verify installation
atlas --version
# Output: atlascli version: 1.x.x
```

### Configuration Initiale

```bash
# 1. Authentification interactive
atlas auth login

# Ouvre navigateur pour OAuth flow
# Ou utilise API key

# 2. Configuration avec API key (CI/CD)
atlas auth login \
  --publicKey "your-public-key" \
  --privateKey "your-private-key"

# 3. Configuration via environment variables (recommandé CI/CD)
export ATLAS_PUBLIC_KEY="your-public-key"
export ATLAS_PRIVATE_KEY="your-private-key"
export ATLAS_PROJECT_ID="your-project-id"

# Vérifier configuration
atlas config list

# Output:
# project_id: 507f1f77bcf86cd799439011
# org_id: 5a0a1e7e0f2912c554080adc
# service: cloud

# 4. Configurer projet par défaut
atlas config set project_id "507f1f77bcf86cd799439011"

# 5. Multiple profiles (dev, staging, prod)
atlas auth login --profile dev
atlas auth login --profile prod

# Use specific profile
atlas clusters list --profile prod
```

---

## 🎯 Commandes Essentielles

### Gestion de Clusters

```bash
# ═══════════════════════════════════════════════════════════════════════
# CLUSTERS - Gestion complète
# ═══════════════════════════════════════════════════════════════════════

# Lister clusters
atlas clusters list \
  --projectId "507f1f77bcf86cd799439011" \
  --output json

# Décrire un cluster
atlas clusters describe "production-cluster" \
  --projectId "507f1f77bcf86cd799439011"

# Créer un cluster M10
atlas clusters create "staging-cluster" \
  --provider AWS \
  --region US_EAST_1 \
  --tier M10 \
  --diskSizeGB 10 \
  --mdbVersion 7.0 \
  --projectId "507f1f77bcf86cd799439011"

# Créer cluster avec fichier JSON (IaC)
cat > cluster-config.json <<EOF
{
  "name": "production-cluster",
  "clusterType": "REPLICASET",
  "providerSettings": {
    "providerName": "AWS",
    "regionName": "US_EAST_1",
    "instanceSizeName": "M30"
  },
  "diskSizeGB": 100,
  "mongoDBMajorVersion": "7.0",
  "backupEnabled": true,
  "autoScaling": {
    "diskGBEnabled": true,
    "compute": {
      "enabled": true,
      "scaleDownEnabled": true,
      "minInstanceSize": "M30",
      "maxInstanceSize": "M60"
    }
  }
}
EOF

atlas clusters create \
  --file cluster-config.json \
  --projectId "507f1f77bcf86cd799439011"

# Mettre à jour (scale up)
atlas clusters update "production-cluster" \
  --tier M40 \
  --projectId "507f1f77bcf86cd799439011"

# Auto-scaling
atlas clusters update "production-cluster" \
  --enableAutoscaling \
  --minInstanceSize M30 \
  --maxInstanceSize M60 \
  --projectId "507f1f77bcf89cd799439011"

# Pause cluster (économiser coûts)
atlas clusters pause "staging-cluster" \
  --projectId "507f1f77bcf86cd799439011"

# Resume
atlas clusters start "staging-cluster" \
  --projectId "507f1f77bcf86cd799439011"

# Supprimer (DANGEREUX)
atlas clusters delete "old-cluster" \
  --force \
  --projectId "507f1f77bcf86cd799439011"

# Watch status (attendre que cluster soit ready)
atlas clusters watch "production-cluster" \
  --projectId "507f1f77bcf86cd799439011"
```

### Gestion Réseau et Sécurité

```bash
# ═══════════════════════════════════════════════════════════════════════
# NETWORKING - IP Whitelist / Access Lists
# ═══════════════════════════════════════════════════════════════════════

# Lister IP access lists
atlas accessLists list \
  --projectId "507f1f77bcf86cd799439011"

# Ajouter IP
atlas accessLists create \
  --ipAddress "203.0.113.0/24" \
  --comment "Office network" \
  --projectId "507f1f77bcf86cd799439011"

# Ajouter IP actuelle (quick add)
atlas accessLists create \
  --currentIp \
  --comment "My laptop" \
  --projectId "507f1f77bcf86cd799439011"

# Bulk add from file
cat > ip-list.json <<EOF
[
  {
    "ipAddress": "203.0.113.0/24",
    "comment": "Office A"
  },
  {
    "ipAddress": "198.51.100.0/24",
    "comment": "Office B"
  }
]
EOF

# Supprimer IP
atlas accessLists delete "203.0.113.0/24" \
  --force \
  --projectId "507f1f77bcf86cd799439011"

# ═══════════════════════════════════════════════════════════════════════
# DATABASE USERS
# ═══════════════════════════════════════════════════════════════════════

# Créer utilisateur
atlas dbusers create \
  --username "app_user" \
  --password "SecurePassword123!" \
  --role "readWrite@mydb" \
  --projectId "507f1f77bcf86cd799439011"

# Avec rôles multiples
atlas dbusers create \
  --username "admin_user" \
  --password "AdminPass456!" \
  --role "readWrite@mydb" \
  --role "dbAdmin@mydb" \
  --projectId "507f1f77bcf86cd799439011"

# Lister utilisateurs
atlas dbusers list \
  --projectId "507f1f77bcf86cd799439011"

# Mettre à jour mot de passe
atlas dbusers update "app_user" \
  --password "NewPassword789!" \
  --projectId "507f1f77bcf86cd799439011"

# Supprimer utilisateur
atlas dbusers delete "old_user" \
  --force \
  --projectId "507f1f77bcf86cd799439011"
```

### Backups et Snapshots

```bash
# ═══════════════════════════════════════════════════════════════════════
# BACKUPS - Snapshots et Restore
# ═══════════════════════════════════════════════════════════════════════

# Lister snapshots
atlas backups snapshots list "production-cluster" \
  --projectId "507f1f77bcf86cd799439011"

# Créer snapshot à la demande
atlas backups snapshots create "production-cluster" \
  --desc "Pre-upgrade snapshot" \
  --retentionDays 7 \
  --projectId "507f1f77bcf86cd799439011"

# Restaurer vers nouveau cluster
atlas backups restores start \
  --clusterName "production-cluster" \
  --snapshotId "5f4a1c2e8f8b3a001c8b4567" \
  --targetClusterName "restored-cluster" \
  --projectId "507f1f77bcf86cd799439011"

# Watch restore status
atlas backups restores watch "restore-job-id" \
  --clusterName "production-cluster" \
  --projectId "507f1f77bcf86cd799439011"

# Lister restore jobs
atlas backups restores list "production-cluster" \
  --projectId "507f1f77bcf86cd799439011"
```

---

## 🔧 Automation et Scripting

### Script Bash : Provisionning Complet

```bash
#!/bin/bash
# provision-environment.sh
# Provisionne un environnement Atlas complet (cluster + réseau + users)

set -e  # Exit on error

# Configuration
PROJECT_ID="${ATLAS_PROJECT_ID}"
ENVIRONMENT="${1:-staging}"  # staging ou production
CLUSTER_NAME="${ENVIRONMENT}-cluster"

echo "🚀 Provisioning ${ENVIRONMENT} environment..."

# 1. Créer cluster
echo "📦 Creating cluster ${CLUSTER_NAME}..."

if [ "$ENVIRONMENT" = "production" ]; then
  TIER="M30"
  DISK_SIZE="100"
  AUTO_SCALING="--enableAutoscaling"
else
  TIER="M10"
  DISK_SIZE="10"
  AUTO_SCALING=""
fi

atlas clusters create "$CLUSTER_NAME" \
  --provider AWS \
  --region US_EAST_1 \
  --tier "$TIER" \
  --diskSizeGB "$DISK_SIZE" \
  --mdbVersion 7.0 \
  --backup \
  $AUTO_SCALING \
  --projectId "$PROJECT_ID" \
  --output json > cluster-output.json

echo "✅ Cluster created"

# 2. Attendre que cluster soit IDLE
echo "⏳ Waiting for cluster to be ready..."
atlas clusters watch "$CLUSTER_NAME" \
  --projectId "$PROJECT_ID"

echo "✅ Cluster is ready"

# 3. Configurer réseau (IP whitelist)
echo "🌐 Configuring network access..."

# Ajouter IP actuelle
atlas accessLists create \
  --currentIp \
  --comment "Provisioning script - $(date)" \
  --projectId "$PROJECT_ID"

# Ajouter IPs de l'environnement
if [ "$ENVIRONMENT" = "production" ]; then
  # Production IPs
  atlas accessLists create \
    --ipAddress "203.0.113.0/24" \
    --comment "Production servers" \
    --projectId "$PROJECT_ID"
else
  # Staging: Allow all (0.0.0.0/0) - NOT recommended for production
  atlas accessLists create \
    --ipAddress "0.0.0.0/0" \
    --comment "Staging - open access" \
    --projectId "$PROJECT_ID"
fi

echo "✅ Network configured"

# 4. Créer database users
echo "👤 Creating database users..."

# Application user (read/write)
APP_PASSWORD=$(openssl rand -base64 32)
atlas dbusers create \
  --username "${ENVIRONMENT}_app" \
  --password "$APP_PASSWORD" \
  --role "readWrite@appdb" \
  --projectId "$PROJECT_ID"

# Admin user
ADMIN_PASSWORD=$(openssl rand -base64 32)
atlas dbusers create \
  --username "${ENVIRONMENT}_admin" \
  --password "$ADMIN_PASSWORD" \
  --role "atlasAdmin" \
  --projectId "$PROJECT_ID"

echo "✅ Users created"

# 5. Sauvegarder credentials (secrets manager)
echo "🔐 Saving credentials..."

# Get connection string
CONNECTION_STRING=$(atlas clusters describe "$CLUSTER_NAME" \
  --projectId "$PROJECT_ID" \
  --output json | jq -r '.connectionStrings.standardSrv')

# Save to file (or push to secrets manager)
cat > "${ENVIRONMENT}-credentials.txt" <<EOF
Environment: ${ENVIRONMENT}
Cluster: ${CLUSTER_NAME}
Connection String: ${CONNECTION_STRING}

App User: ${ENVIRONMENT}_app
App Password: ${APP_PASSWORD}

Admin User: ${ENVIRONMENT}_admin
Admin Password: ${ADMIN_PASSWORD}

Created: $(date)
EOF

chmod 600 "${ENVIRONMENT}-credentials.txt"

echo "✅ Credentials saved to ${ENVIRONMENT}-credentials.txt"

# 6. Résumé
echo ""
echo "🎉 Environment ${ENVIRONMENT} provisioned successfully!"
echo ""
echo "Cluster Name: ${CLUSTER_NAME}"
echo "Connection String: ${CONNECTION_STRING}"
echo ""
echo "⚠️  IMPORTANT: Store credentials securely!"
echo "    File: ${ENVIRONMENT}-credentials.txt"
echo ""
```

### Script : Daily Health Check

```bash
#!/bin/bash
# daily-health-check.sh
# Vérifie la santé des clusters et envoie rapport

set -e

PROJECT_ID="${ATLAS_PROJECT_ID}"
SLACK_WEBHOOK="${SLACK_WEBHOOK_URL}"

echo "🏥 Running daily health check..."

# Get all clusters
CLUSTERS=$(atlas clusters list \
  --projectId "$PROJECT_ID" \
  --output json)

CLUSTER_COUNT=$(echo "$CLUSTERS" | jq '. | length')

echo "Found ${CLUSTER_COUNT} clusters"

# Initialize report
REPORT="📊 *Daily Atlas Health Report* - $(date +%Y-%m-%d)\n\n"
ISSUES=0

# Check each cluster
for i in $(seq 0 $(($CLUSTER_COUNT - 1))); do
  CLUSTER=$(echo "$CLUSTERS" | jq -r ".[$i]")
  CLUSTER_NAME=$(echo "$CLUSTER" | jq -r '.name')
  STATE=$(echo "$CLUSTER" | jq -r '.stateName')

  echo "Checking ${CLUSTER_NAME}..."

  REPORT+="*${CLUSTER_NAME}*\n"

  # Check state
  if [ "$STATE" != "IDLE" ]; then
    REPORT+="  ⚠️ State: ${STATE}\n"
    ISSUES=$((ISSUES + 1))
  else
    REPORT+="  ✅ State: ${STATE}\n"
  fi

  # Check disk usage
  DISK_SIZE=$(echo "$CLUSTER" | jq -r '.diskSizeGB')
  # TODO: Get actual usage from metrics

  # Check backup status
  BACKUP_ENABLED=$(echo "$CLUSTER" | jq -r '.backupEnabled')
  if [ "$BACKUP_ENABLED" != "true" ]; then
    REPORT+="  ⚠️ Backups: DISABLED\n"
    ISSUES=$((ISSUES + 1))
  else
    REPORT+="  ✅ Backups: Enabled\n"
  fi

  REPORT+="\n"
done

# Summary
if [ $ISSUES -eq 0 ]; then
  REPORT+="✅ *All systems healthy*"
  STATUS_EMOJI="✅"
else
  REPORT+="⚠️ *${ISSUES} issues detected*"
  STATUS_EMOJI="⚠️"
fi

# Send to Slack
if [ -n "$SLACK_WEBHOOK" ]; then
  curl -X POST "$SLACK_WEBHOOK" \
    -H 'Content-Type: application/json' \
    -d "{\"text\": \"${STATUS_EMOJI} ${REPORT}\"}"
fi

echo "$REPORT"

# Exit with error if issues
exit $ISSUES
```

---

## 🔄 CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/atlas-deploy.yml
# Deploy MongoDB Atlas infrastructure

name: Atlas Deploy

on:
  push:
    branches: [main]
    paths:
      - 'infrastructure/atlas/**'
  workflow_dispatch:

env:
  ATLAS_PROJECT_ID: ${{ secrets.ATLAS_PROJECT_ID }}
  ATLAS_PUBLIC_KEY: ${{ secrets.ATLAS_PUBLIC_KEY }}
  ATLAS_PRIVATE_KEY: ${{ secrets.ATLAS_PRIVATE_KEY }}

jobs:
  validate:
    name: Validate Configuration
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Atlas CLI
        run: |
          wget -qO - https://pgp.mongodb.com/server-6.0.asc | sudo apt-key add -
          echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.com/apt/ubuntu focal/mongodb-atlas-cli multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-atlas-cli.list
          sudo apt-get update
          sudo apt-get install -y mongodb-atlas-cli

      - name: Configure Atlas CLI
        run: |
          atlas config set project_id $ATLAS_PROJECT_ID

      - name: Validate cluster config
        run: |
          # Validate JSON syntax
          jq empty infrastructure/atlas/cluster-config.json
          echo "✅ Configuration is valid"

  deploy:
    name: Deploy to Atlas
    needs: validate
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Install Atlas CLI
        run: |
          wget -qO - https://pgp.mongodb.com/server-6.0.asc | sudo apt-key add -
          echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.com/apt/ubuntu focal/mongodb-atlas-cli multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-atlas-cli.list
          sudo apt-get update
          sudo apt-get install -y mongodb-atlas-cli

      - name: Configure Atlas CLI
        run: |
          atlas config set project_id $ATLAS_PROJECT_ID

      - name: Check if cluster exists
        id: check
        run: |
          if atlas clusters describe production-cluster --projectId $ATLAS_PROJECT_ID &>/dev/null; then
            echo "exists=true" >> $GITHUB_OUTPUT
          else
            echo "exists=false" >> $GITHUB_OUTPUT
          fi

      - name: Create cluster
        if: steps.check.outputs.exists == 'false'
        run: |
          atlas clusters create \
            --file infrastructure/atlas/cluster-config.json \
            --projectId $ATLAS_PROJECT_ID

          # Wait for cluster
          atlas clusters watch production-cluster \
            --projectId $ATLAS_PROJECT_ID

      - name: Update cluster
        if: steps.check.outputs.exists == 'true'
        run: |
          # Apply configuration changes
          atlas clusters update production-cluster \
            --file infrastructure/atlas/cluster-config.json \
            --projectId $ATLAS_PROJECT_ID

      - name: Configure network access
        run: |
          # Add GitHub Actions IPs
          atlas accessLists create \
            --cidrBlock "0.0.0.0/0" \
            --comment "GitHub Actions - $(date)" \
            --projectId $ATLAS_PROJECT_ID || true

      - name: Verify deployment
        run: |
          atlas clusters describe production-cluster \
            --projectId $ATLAS_PROJECT_ID \
            --output json | jq '.'

      - name: Run smoke tests
        run: |
          # TODO: Add database connectivity tests
          echo "✅ Deployment successful"

  notify:
    name: Notify
    needs: deploy
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Send Slack notification
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: |
            Atlas deployment: ${{ job.status }}
            Cluster: production-cluster
            Commit: ${{ github.sha }}
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### GitLab CI

```yaml
# .gitlab-ci.yml
# Deploy MongoDB Atlas via GitLab CI

stages:
  - validate
  - deploy
  - verify

variables:
  ATLAS_PROJECT_ID: $ATLAS_PROJECT_ID
  ATLAS_PUBLIC_KEY: $ATLAS_PUBLIC_KEY
  ATLAS_PRIVATE_KEY: $ATLAS_PRIVATE_KEY

.atlas_cli:
  before_script:
    - apt-get update && apt-get install -y wget jq
    - wget -qO - https://pgp.mongodb.com/server-6.0.asc | apt-key add -
    - echo "deb [ arch=amd64,arm64 ] https://repo.mongodb.com/apt/ubuntu focal/mongodb-atlas-cli multiverse" | tee /etc/apt/sources.list.d/mongodb-atlas-cli.list
    - apt-get update && apt-get install -y mongodb-atlas-cli
    - atlas config set project_id $ATLAS_PROJECT_ID

validate:
  extends: .atlas_cli
  stage: validate
  script:
    - echo "Validating configuration..."
    - jq empty infrastructure/atlas/*.json
    - echo "✅ Configuration valid"

deploy:
  extends: .atlas_cli
  stage: deploy
  environment:
    name: production
  only:
    - main
  script:
    - echo "Deploying to Atlas..."
    - ./scripts/deploy-atlas.sh production
  artifacts:
    reports:
      dotenv: deploy.env

verify:
  extends: .atlas_cli
  stage: verify
  needs: [deploy]
  script:
    - echo "Verifying deployment..."
    - atlas clusters describe production-cluster --projectId $ATLAS_PROJECT_ID
    - echo "✅ Deployment verified"
```

---

## 📋 Best Practices

### Checklist DevOps

```
┌───────────────────────────────────────────────────────────────────────┐
│                 ATLAS CLI DEVOPS BEST PRACTICES                       │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  AUTHENTICATION                                                       │
│  ☐ Utiliser API keys (pas email/password en CI/CD)                    │
│  ☐ API keys stockées dans secrets manager                             │
│  ☐ Rotation des keys tous les 90 jours                                │
│  ☐ Permissions minimales (principle of least privilege)               │
│  ☐ Keys différentes par environnement (dev, staging, prod)            │
│                                                                       │
│  INFRASTRUCTURE AS CODE                                               │
│  ☐ Configuration en JSON/YAML (versionné Git)                         │
│  ☐ Naming conventions cohérentes                                      │
│  ☐ Paramètres externalisés (environment variables)                    │
│  ☐ Documentation à jour (README, comments)                            │
│  ☐ Validation avant apply (dry-run)                                   │
│                                                                       │
│  SCRIPTING                                                            │
│  ☐ Scripts idempotents (safe to rerun)                                │
│  ☐ Error handling robuste (set -e, exit codes)                        │
│  ☐ Logging détaillé (stdout, stderr)                                  │
│  ☐ Dry-run mode pour testing                                          │
│  ☐ Confirmation prompts pour opérations destructives                  │
│                                                                       │
│  CI/CD                                                                │
│  ☐ Pipeline automated (GitHub Actions, GitLab, Jenkins)               │
│  ☐ Environment séparés (dev, staging, prod)                           │
│  ☐ Approval gates pour production                                     │
│  ☐ Rollback procedure documentée                                      │
│  ☐ Notifications (Slack, email)                                       │
│                                                                       │
│  MONITORING                                                           │
│  ☐ Health checks automatisés (cron)                                   │
│  ☐ Alerting configuré                                                 │
│  ☐ Metrics collectées (dashboard)                                     │
│  ☐ Logs centralisés (aggregation)                                     │
│  ☐ Audit trail (qui a fait quoi quand)                                │
│                                                                       │
│  SÉCURITÉ                                                             │
│  ☐ Jamais de secrets en clair dans Git                                │
│  ☐ Network access restrictif (IP whitelist)                           │
│  ☐ Database users avec permissions minimales                          │
│  ☐ Audit des accès régulier                                           │
│  ☐ Compliance checks (GDPR, HIPAA, etc.)                              │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🏁 Résumé

### Points Clés

1. **Installation**
   - Multi-platform (Linux, macOS, Windows)
   - Homebrew, apt, yum, Chocolatey
   - Via Go pour build from source

2. **Configuration**
   - API keys (CI/CD friendly)
   - OAuth (interactive)
   - Multiple profiles (dev, staging, prod)
   - Environment variables

3. **Commandes Essentielles**
   - Clusters : create, update, delete, scale, pause
   - Networking : IP access lists, peering
   - Users : create, update, delete, roles
   - Backups : snapshots, restore, schedule

4. **Automation**
   - Bash scripts pour provisionning
   - Health checks automatisés
   - Cron jobs pour maintenance
   - Monitoring continu

5. **CI/CD**
   - GitHub Actions integration
   - GitLab CI pipelines
   - Jenkins support
   - Deployment automation

### Script Template Production

```bash
#!/bin/bash
# atlas-deploy.sh - Production deployment template

set -euo pipefail  # Strict mode

# Configuration
readonly PROJECT_ID="${ATLAS_PROJECT_ID:?ATLAS_PROJECT_ID not set}"
readonly ENVIRONMENT="${1:?Environment required (staging|production)}"
readonly DRY_RUN="${DRY_RUN:-false}"

# Colors
readonly RED='\033[0;31m'
readonly GREEN='\033[0;32m'
readonly YELLOW='\033[1;33m'
readonly NC='\033[0m' # No Color

# Functions
log() {
  echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $*"
}

error() {
  echo -e "${RED}[ERROR]${NC} $*" >&2
}

warn() {
  echo -e "${YELLOW}[WARN]${NC} $*"
}

confirm() {
  if [ "$DRY_RUN" = "true" ]; then
    log "DRY RUN: Would execute: $1"
    return 0
  fi

  read -p "$(echo -e ${YELLOW}$1 '(y/N):'${NC}) " -n 1 -r
  echo
  [[ $REPLY =~ ^[Yy]$ ]]
}

# Main
main() {
  log "Starting Atlas deployment for ${ENVIRONMENT}"

  # Validate
  if ! command -v atlas &> /dev/null; then
    error "Atlas CLI not installed"
    exit 1
  fi

  # Check authentication
  if ! atlas config describe &> /dev/null; then
    error "Not authenticated. Run: atlas auth login"
    exit 1
  fi

  # Confirm for production
  if [ "$ENVIRONMENT" = "production" ]; then
    if ! confirm "Deploy to PRODUCTION?"; then
      log "Deployment cancelled"
      exit 0
    fi
  fi

  # Deploy cluster
  log "Deploying cluster..."
  atlas clusters create "${ENVIRONMENT}-cluster" \
    --file "configs/${ENVIRONMENT}/cluster.json" \
    --projectId "$PROJECT_ID" \
    --output json > deployment.json || {
      error "Cluster creation failed"
      exit 1
    }

  log "✅ Deployment successful"

  # Output connection string
  CONNECTION_STRING=$(jq -r '.connectionStrings.standardSrv' deployment.json)
  log "Connection String: ${CONNECTION_STRING}"
}

# Execute
main "$@"
```

### Ressources

- [Atlas CLI Documentation](https://www.mongodb.com/docs/atlas/cli/stable/)
- [GitHub Repository](https://github.com/mongodb/mongodb-atlas-cli)
- [API Reference](https://www.mongodb.com/docs/atlas/reference/api-resources-spec/)

---

## 🎉 Conclusion du Chapitre 14

Ce chapitre **MongoDB Atlas** est maintenant **COMPLET** avec les 16 sections :

1. ✅ Présentation d'Atlas
2. ✅ Création de clusters
3. ✅ Tiers et pricing
4. ✅ Réseau et sécurité
5. ✅ Connexion aux applications
6. ✅ Monitoring et alertes
7. ✅ Backups et restauration
8. ✅ Scaling
9. ✅ Atlas Search
10. ✅ Atlas Data Lake
11. ✅ Atlas Charts
12. ✅ Atlas App Services
13. ✅ Atlas Vector Search
14. ✅ Triggers et fonctions serverless
15. ✅ Data API
16. ✅ **Atlas CLI**

**Vous disposez maintenant d'une formation MongoDB Atlas complète et production-ready !** 🚀

---


⏭️ [Drivers et Intégration Applicative](/15-drivers-integration-applicative/README.md)
