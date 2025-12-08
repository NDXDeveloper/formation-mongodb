🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 12.12 Bonnes Pratiques

## Introduction

Les bonnes pratiques de sauvegarde et restauration ne sont pas simplement une liste de recommandations techniques - elles constituent le **fondement même de la continuité d'activité** d'une organisation. Une stratégie de backup mal conçue ou mal exécutée peut conduire à des pertes de données catastrophiques, des interruptions prolongées et, dans les cas extrêmes, à la fermeture d'une entreprise.

Cette section synthétise les enseignements des chapitres précédents en un ensemble cohérent de principes, procédures et recommandations éprouvées pour garantir la résilience de vos données MongoDB dans tous les scénarios possibles.

## Principes Fondamentaux

### La Règle 3-2-1-1-0

```
┌──────────────────────────────────────────────────────────────┐
│              Règle 3-2-1-1-0 de Backup                       │
│                                                              │
│  3️⃣  TROIS copies de vos données                             │
│     • Original (production)                                  │
│     • Backup local                                           │
│     • Backup distant                                         │
│                                                              │
│  2️⃣  DEUX types de média différents                          │
│     • Disk (SSD/HDD local)                                   │
│     • Cloud/Tape/Remote storage                              │
│                                                              │
│  1️⃣  UNE copie off-site (hors site)                          │
│     • Protection contre sinistres locaux                     │
│     • Région cloud différente                                │
│     • Data center géographiquement distant                   │
│                                                              │
│  1️⃣  UNE copie offline/immutable                             │
│     • Protection contre ransomware                           │
│     • S3 Object Lock / Azure Immutable Blob                  │
│     • Air-gapped storage                                     │
│                                                              │
│  0️⃣  ZÉRO erreur de restauration                             │
│     • Tests réguliers obligatoires                           │
│     • Validation automatisée                                 │
│     • Procédures documentées et testées                      │
└──────────────────────────────────────────────────────────────┘
```

### Architecture de Référence

```
┌────────────────────────────────────────────────────────────────────┐
│                  Architecture de Backup Complète                   │
│                                                                    │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │              PRODUCTION (Site Primaire)                    │    │
│  │                                                            │    │
│  │  ┌──────────────────────────────────────────────────────┐  │    │
│  │  │  MongoDB Replica Set                                 │  │    │
│  │  │  • Primary + 2 Secondaries                           │  │    │
│  │  │  • Oplog: 72h minimum                                │  │    │
│  │  └──────────────────────────────────────────────────────┘  │    │
│  │                          │                                 │    │
│  │                          v                                 │    │
│  │  ┌──────────────────────────────────────────────────────┐  │    │
│  │  │  Backup Local (même DC)                              │  │    │
│  │  │  • Full backup: quotidien 2h                         │  │    │
│  │  │  • Snapshot: toutes les 6h                           │  │    │
│  │  │  • Oplog streaming: continu                          │  │    │
│  │  │  • Rétention: 7 jours                                │  │    │
│  │  └──────────────────────────────────────────────────────┘  │    │
│  └────────────────────────┬───────────────────────────────────┘    │
│                           │                                        │
│                           v                                        │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │           BACKUP DISTANT (Région différente)               │    │
│  │                                                            │    │
│  │  ┌──────────────────────────────────────────────────────┐  │    │
│  │  │  Object Storage (S3/Azure/GCS)                       │  │    │
│  │  │  • Réplication: quotidienne                          │  │    │
│  │  │  • Encryption: AES-256                               │  │    │
│  │  │  • Versioning: enabled                               │  │    │
│  │  │  • Rétention GFS: 7j/4w/6m/7y                        │  │    │
│  │  └──────────────────────────────────────────────────────┘  │    │
│  │                          │                                 │    │
│  │                          v                                 │    │
│  │  ┌──────────────────────────────────────────────────────┐  │    │
│  │  │  Immutable Storage                                   │  │    │
│  │  │  • S3 Object Lock / Glacier                          │  │    │
│  │  │  • Write-Once-Read-Many (WORM)                       │  │    │
│  │  │  • Protection ransomware                             │  │    │
│  │  │  • Rétention légale: selon compliance                │  │    │
│  │  └──────────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────┘    │
│                           │                                        │
│                           v                                        │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │              DISASTER RECOVERY SITE                        │    │
│  │                                                            │    │
│  │  • Standby cluster (warm/hot)                              │    │
│  │  • Continuous replication                                  │    │
│  │  • Automatic/Manual failover                               │    │
│  │  • RTO: < 1h, RPO: < 15min                                 │    │
│  └────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
```

## Stratégie par Environnement

### Development / Testing

```yaml
Environnement: Development / Testing
Criticité: Faible
RPO: 24 heures
RTO: 4-8 heures

Configuration Backup:
  méthode: mongodump
  fréquence: quotidien
  rétention: 7 jours
  stockage: local uniquement
  compression: gzip
  chiffrement: non requis

Oplog:
  taille: 5-10 GB
  fenêtre: 24 heures

Tests:
  fréquence: mensuel
  type: basique
  automatisation: optionnel

Coût estimé: $50-200/mois

Justification:
  - Données non critiques
  - Facilement recréables
  - Focus sur rapidité de setup
  - Coût minimal

Implémentation:
  ```bash
  # Cron simple
  0 2 * * * mongodump --uri=mongodb://localhost:27017 \
    --out=/backup/dev/$(date +\%Y\%m\%d) --gzip

  # Cleanup 7 jours
  find /backup/dev -type d -mtime +7 -exec rm -rf {} \;
  ```
```

### Staging / Pre-Production

```yaml
Environnement: Staging / Pre-Production
Criticité: Moyenne
RPO: 6 heures
RTO: 2-4 heures

Configuration Backup:
  méthode: mongodump + snapshots
  fréquence:
    - Full: quotidien
    - Snapshot: toutes les 6h
  rétention:
    - Daily: 14 jours
    - Weekly: 4 semaines
  stockage: local + cloud
  compression: gzip
  chiffrement: recommandé (AES-256)

Oplog:
  taille: 20-50 GB
  fenêtre: 48 heures
  export_continu: recommandé

Tests:
  fréquence: hebdomadaire
  type: standard
  automatisation: obligatoire

Monitoring:
  - Alertes sur échec
  - Dashboard basique
  - Métriques rétention

Coût estimé: $200-500/mois

Justification:
  - Environnement de validation
  - Tests pré-production
  - Données similaires à production
  - Besoin de restauration rapide

Implémentation:
  - Systemd timers
  - Upload S3 automatique
  - Rotation GFS basique
  - Notifications Slack
```

### Production Standard

```yaml
Environnement: Production Standard
Criticité: Élevée
RPO: 1 heure
RTO: 1-2 heures

Configuration Backup:
  méthode: mongodump + snapshots + oplog streaming
  fréquence:
    - Full: quotidien 2h
    - Snapshot: toutes les 4h
    - Oplog: continu
  rétention:
    - Hourly: 24 heures
    - Daily: 7 jours
    - Weekly: 4 semaines
    - Monthly: 6 mois
  stockage: local + cloud multi-région
  compression: zstd (meilleur ratio)
  chiffrement: AES-256 (obligatoire)

Oplog:
  taille: 100-200 GB
  fenêtre: 72 heures minimum
  export_continu: obligatoire

PITR:
  activé: true
  fenêtre: 48 heures

Tests:
  fréquence:
    - Basique: hebdomadaire (automatisé)
    - Standard: mensuel (automatisé)
    - Complet: trimestriel
    - PITR: mensuel
  validation: complète
  documentation: obligatoire

Monitoring:
  - Alertes temps réel
  - Dashboard Grafana
  - Métriques complètes
  - SLA tracking
  - PagerDuty intégration

DR:
  site_secondaire: warm standby
  réplication: continue
  failover: manuel (< 1h)
  tests: semestriels

Coût estimé: $1,000-3,000/mois

Justification:
  - Données business critiques
  - Impact financier modéré
  - SLA client standard
  - Compliance de base

Implémentation Clé:
  - Kubernetes CronJobs
  - Automation complète
  - Multi-région S3
  - Immutabilité partielle
  - Tests automatisés
```

### Production Critique / Mission-Critical

```yaml
Environnement: Production Mission-Critical
Criticité: Critique
RPO: 15 minutes ou moins
RTO: 30 minutes ou moins

Configuration Backup:
  méthode:
    - Continuous replication (DR site)
    - Snapshots (hourly)
    - mongodump (daily)
    - Oplog streaming (real-time)
  fréquence:
    - Snapshot: horaire
    - Full: quotidien
    - Oplog: temps réel
  rétention:
    - Hourly: 48 heures
    - Daily: 30 jours
    - Weekly: 12 semaines
    - Monthly: 12 mois
    - Yearly: 7 ans
  stockage:
    - Local: triple répliqué
    - Cloud: multi-région (3+)
    - Air-gapped: mensuel
  compression: zstd niveau 9
  chiffrement: AES-256 + HSM

Oplog:
  taille: 500+ GB
  fenêtre: 168 heures (7 jours)
  export_continu: obligatoire redondant

PITR:
  activé: true
  fenêtre: 7 jours
  précision: seconde

Tests:
  fréquence:
    - Basique: quotidien (automatisé)
    - Standard: hebdomadaire (automatisé)
    - Complet: mensuel
    - PITR: hebdomadaire
    - DR drill: trimestriel
  validation: exhaustive
  documentation: obligatoire + audit trail

Monitoring:
  - Alertes temps réel multi-canal
  - Dashboard 24/7
  - Métriques prédictives
  - Anomaly detection
  - War room protocol

DR:
  site_secondaire: hot standby (actif)
  réplication: synchrone/quasi-synchrone
  failover: automatique (< 30min)
  tests: mensuels
  multi_region: 3+ régions

Sécurité:
  - Chiffrement bout-en-bout
  - RBAC strict
  - Audit complet
  - Immutabilité obligatoire
  - Air-gap backups
  - Ransomware protection

Compliance:
  - SOC 2 Type II
  - ISO 27001
  - GDPR / HIPAA si applicable
  - Retention légale
  - Audit trail complet

Coût estimé: $5,000-20,000+/mois

Justification:
  - Données mission-critical
  - Impact financier majeur (>$10k/heure)
  - SLA client strict (99.99%+)
  - Compliance réglementaire
  - Réputation entreprise

Implémentation Clé:
  - MongoDB Atlas (recommandé) ou
  - Ops Manager + DR complet
  - GitOps pour IaC
  - Chaos engineering
  - Automation maximale
  - Équipe dédiée 24/7
```

## Sécurité des Backups

### Chiffrement

```bash
#!/bin/bash
# encryption_best_practices.sh

# ============================================================================
# CHIFFREMENT AT-REST
# ============================================================================

# Méthode 1: Chiffrement avec GPG (simple)
backup_with_gpg() {
  local backup_file=$1
  local recipient="backup@company.com"

  # Compression puis chiffrement
  tar -czf - "$backup_file" | \
    gpg --encrypt --recipient "$recipient" \
    > "${backup_file}.tar.gz.gpg"

  # Vérification
  gpg --list-packets "${backup_file}.tar.gz.gpg" >/dev/null 2>&1

  echo "✓ Backup encrypted with GPG"
}

# Méthode 2: Chiffrement avec OpenSSL (sans clé publique)
backup_with_openssl() {
  local backup_file=$1
  local password_file="/etc/mongodb-backup/encryption.key"

  # Générer clé si n'existe pas
  if [ ! -f "$password_file" ]; then
    openssl rand -base64 32 > "$password_file"
    chmod 600 "$password_file"
  fi

  # Compression et chiffrement AES-256-CBC
  tar -czf - "$backup_file" | \
    openssl enc -aes-256-cbc -salt \
    -pass file:"$password_file" \
    -out "${backup_file}.tar.gz.enc"

  # Checksum
  sha256sum "${backup_file}.tar.gz.enc" > "${backup_file}.tar.gz.enc.sha256"

  echo "✓ Backup encrypted with OpenSSL AES-256"
}

# Méthode 3: AWS S3 avec KMS (recommandé cloud)
backup_to_s3_encrypted() {
  local backup_file=$1
  local s3_bucket="s3://company-backups"
  local kms_key="arn:aws:kms:us-east-1:123456789:key/abc-def-123"

  # Upload avec chiffrement côté serveur via KMS
  aws s3 cp "$backup_file" "$s3_bucket/" \
    --sse aws:kms \
    --sse-kms-key-id "$kms_key" \
    --storage-class STANDARD_IA

  echo "✓ Backup uploaded to S3 with KMS encryption"
}

# Méthode 4: Chiffrement MongoDB natif (WiredTiger)
enable_mongodb_encryption() {
  # Note: Nécessite MongoDB Enterprise

  cat > /etc/mongod-encryption.conf <<'EOF'
security:
  enableEncryption: true
  encryptionKeyFile: /etc/mongodb/encryption-key
  encryptionCipherMode: AES256-CBC

# Générer la clé (une seule fois)
# openssl rand -base64 32 > /etc/mongodb/encryption-key
# chmod 600 /etc/mongodb/encryption-key
# chown mongodb:mongodb /etc/mongodb/encryption-key
EOF

  echo "✓ MongoDB encryption-at-rest configured"
  echo "⚠️  Restart MongoDB to apply changes"
}
```

### Contrôle d'Accès

```yaml
# Matrice RBAC pour les Backups

Rôle: backup-operator (quotidien)
  permissions:
    - Exécuter backups automatisés
    - Lire configurations backup
    - Écrire dans répertoire backup local
    - Upload vers stockage distant
  restrictions:
    - Pas d'accès restauration
    - Pas de suppression backups
    - Pas de modification rétention

Rôle: backup-validator (tests)
  permissions:
    - backup-operator permissions +
    - Lire backups
    - Créer instances de test
    - Exécuter tests restauration
    - Écrire rapports de test
  restrictions:
    - Pas d'accès production
    - Pas de suppression backups

Rôle: restore-operator (incidents)
  permissions:
    - backup-validator permissions +
    - Restaurer vers production
    - Modifier configurations restauration
  restrictions:
    - Nécessite approbation manager
    - Actions auditées
    - MFA obligatoire

Rôle: backup-admin
  permissions:
    - Toutes permissions backup/restore
    - Modifier politiques rétention
    - Supprimer backups (avec justification)
    - Accès air-gap backups
  restrictions:
    - Actions auditées
    - MFA obligatoire
    - Require 2-person rule pour suppressions

Implémentation MongoDB:
  ```javascript
  // Créer les rôles
  use admin

  // Backup Operator
  db.createRole({
    role: "backupOperator",
    privileges: [
      { resource: { cluster: true }, actions: ["listDatabases"] },
      { resource: { db: "", collection: "" }, actions: ["find"] }
    ],
    roles: []
  });

  // Restore Operator
  db.createRole({
    role: "restoreOperator",
    privileges: [
      { resource: { cluster: true }, actions: ["listDatabases"] },
      { resource: { db: "", collection: "" }, actions: ["insert", "createIndex"] }
    ],
    roles: ["backupOperator"]
  });

  // Créer utilisateurs
  db.createUser({
    user: "backup-bot",
    pwd: "SecurePassword",
    roles: ["backupOperator"]
  });
  ```
```

### Protection Ransomware

```bash
#!/bin/bash
# ransomware_protection.sh

# ============================================================================
# STRATÉGIE DE PROTECTION CONTRE RANSOMWARE
# ============================================================================

# 1. Immutabilité S3 Object Lock
configure_s3_immutability() {
  local bucket="company-backups"

  # Activer versioning (prérequis)
  aws s3api put-bucket-versioning \
    --bucket "$bucket" \
    --versioning-configuration Status=Enabled

  # Activer Object Lock (compliance mode)
  aws s3api put-object-lock-configuration \
    --bucket "$bucket" \
    --object-lock-configuration '{
      "ObjectLockEnabled": "Enabled",
      "Rule": {
        "DefaultRetention": {
          "Mode": "COMPLIANCE",
          "Days": 90
        }
      }
    }'

  echo "✓ S3 Object Lock configured (90 days retention)"
  echo "⚠️  Objects cannot be deleted even by root for 90 days"
}

# 2. Azure Immutable Blob Storage
configure_azure_immutability() {
  local resource_group="backups-rg"
  local storage_account="companybackups"
  local container="mongodb-backups"

  # Créer policy d'immutabilité
  az storage container immutability-policy create \
    --account-name "$storage_account" \
    --container-name "$container" \
    --period 90 \
    --resource-group "$resource_group"

  # Verrouiller la policy
  az storage container immutability-policy lock \
    --account-name "$storage_account" \
    --container-name "$container" \
    --resource-group "$resource_group"

  echo "✓ Azure Immutable Blob configured"
}

# 3. Air-Gap Backups
setup_air_gap_backup() {
  local source_backup=$1
  local air_gap_mount="/mnt/air-gap"

  # Vérifier que le média air-gap est monté
  if ! mountpoint -q "$air_gap_mount"; then
    echo "✗ Air-gap storage not mounted"
    return 1
  fi

  # Copier avec vérification
  rsync -av --checksum "$source_backup" "$air_gap_mount/"

  # Créer manifeste
  find "$air_gap_mount" -type f -exec sha256sum {} \; > \
    "$air_gap_mount/CHECKSUMS_$(date +%Y%m%d).txt"

  # Créer read-only snapshot (si ZFS/Btrfs)
  # zfs snapshot tank/air-gap@$(date +%Y%m%d)

  echo "✓ Air-gap backup completed"
  echo "⚠️  Unmount and physically disconnect storage"
}

# 4. Détection d'anomalies
detect_backup_anomalies() {
  local backup_dir="/backup/mongodb"
  local alert_threshold=50  # % de variation

  # Obtenir taille actuelle
  local current_size=$(du -sb "$backup_dir/latest" | cut -f1)

  # Obtenir taille précédente
  local previous_size=$(cat "$backup_dir/.last_size" 2>/dev/null || echo "$current_size")

  # Calculer variation
  local variation=$(echo "scale=2; ($current_size - $previous_size) / $previous_size * 100" | bc)

  # Alerter si variation anormale
  if (( $(echo "$variation > $alert_threshold || $variation < -$alert_threshold" | bc -l) )); then
    echo "🚨 ANOMALY DETECTED: Backup size variation: ${variation}%"
    echo "  Previous: $(numfmt --to=iec-i --suffix=B $previous_size)"
    echo "  Current: $(numfmt --to=iec-i --suffix=B $current_size)"

    # Notifier équipe sécurité
    alert_security_team "Backup size anomaly detected: ${variation}%"

    return 1
  fi

  # Sauvegarder la taille pour prochaine comparaison
  echo "$current_size" > "$backup_dir/.last_size"

  echo "✓ Backup size within normal range (${variation}%)"
}

# 5. Backup Segregation (network isolation)
configure_backup_network_isolation() {
  # Réseau dédié aux backups avec règles strictes

  cat > /etc/nftables/backup-isolation.nft <<'EOF'
# Backup Network Isolation Rules

table inet backup_isolation {
  chain input {
    type filter hook input priority 0; policy drop;

    # Autoriser uniquement backup node
    ip saddr 10.0.backup.0/24 tcp dport 27017 accept

    # SSH admin uniquement
    ip saddr 10.0.admin.0/24 tcp dport 22 accept

    # Drop tout le reste
    log prefix "BACKUP_BLOCKED: " drop
  }

  chain output {
    type filter hook output priority 0; policy accept;

    # Autoriser seulement vers stockage backup
    ip daddr 10.0.storage.0/24 accept

    # Bloquer accès Internet (sauf backup cloud)
    # ip daddr backup-cloud.com accept
  }
}
EOF

  nft -f /etc/nftables/backup-isolation.nft

  echo "✓ Backup network isolation configured"
}
```

## Gestion du Cycle de Vie

### Politique de Rétention GFS

```python
#!/usr/bin/env python3
# gfs_retention_manager.py

"""
Grandfather-Father-Son (GFS) Retention Policy Manager
"""

import os
import datetime
import json
from pathlib import Path
from typing import List, Dict
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class GFSRetentionManager:
    """Gestionnaire de rétention GFS pour backups MongoDB"""

    def __init__(self, backup_root: str, config: Dict):
        self.backup_root = Path(backup_root)
        self.config = config

        # Politique par défaut (jours)
        self.retention = {
            'hourly': config.get('hourly', 24),
            'daily': config.get('daily', 7),
            'weekly': config.get('weekly', 28),
            'monthly': config.get('monthly', 180),
            'yearly': config.get('yearly', 2555)
        }

    def classify_backup(self, backup_path: Path) -> str:
        """Classifier un backup selon GFS"""

        # Extraire timestamp du nom
        # Format: backup_YYYYMMDD_HHMMSS
        try:
            timestamp_str = backup_path.name.split('_')[1] + backup_path.name.split('_')[2]
            timestamp = datetime.datetime.strptime(timestamp_str, '%Y%m%d%H%M%S')
        except (IndexError, ValueError):
            return 'unknown'

        now = datetime.datetime.now()
        age_hours = (now - timestamp).total_seconds() / 3600

        # Classification
        if age_hours <= self.retention['hourly']:
            return 'hourly'
        elif timestamp.hour == 2:  # Backup quotidien à 2h
            age_days = (now - timestamp).days

            if age_days <= self.retention['daily']:
                return 'daily'
            elif timestamp.weekday() == 6:  # Dimanche
                age_weeks = age_days / 7

                if age_weeks <= (self.retention['weekly'] / 7):
                    return 'weekly'
                elif timestamp.day == 1:  # Premier du mois
                    age_months = (now.year - timestamp.year) * 12 + (now.month - timestamp.month)

                    if age_months <= (self.retention['monthly'] / 30):
                        return 'monthly'
                    elif timestamp.month == 1 and timestamp.day == 1:  # 1er janvier
                        return 'yearly'

        return 'expired'

    def apply_retention(self, dry_run: bool = False) -> Dict:
        """Appliquer la politique de rétention"""

        stats = {
            'hourly': 0,
            'daily': 0,
            'weekly': 0,
            'monthly': 0,
            'yearly': 0,
            'expired': 0,
            'deleted': 0
        }

        logger.info("=== Applying GFS Retention Policy ===")
        logger.info(f"Backup root: {self.backup_root}")
        logger.info(f"Dry run: {dry_run}")

        # Scanner tous les backups
        for backup_path in sorted(self.backup_root.glob('backup_*')):
            if not backup_path.is_dir():
                continue

            classification = self.classify_backup(backup_path)
            stats[classification] += 1

            logger.debug(f"{backup_path.name}: {classification}")

            # Supprimer si expiré
            if classification == 'expired':
                if dry_run:
                    logger.info(f"[DRY-RUN] Would delete: {backup_path}")
                else:
                    logger.info(f"Deleting expired backup: {backup_path}")
                    # shutil.rmtree(backup_path)
                    stats['deleted'] += 1

        # Rapport
        logger.info("")
        logger.info("=== Retention Summary ===")
        for category, count in stats.items():
            if category != 'deleted':
                logger.info(f"  {category.capitalize()}: {count} backup(s)")

        if not dry_run:
            logger.info(f"  Deleted: {stats['deleted']} backup(s)")

        return stats

    def promote_backups(self):
        """Promouvoir les backups selon GFS"""

        logger.info("=== Promoting Backups ===")

        now = datetime.datetime.now()

        # Promouvoir vers weekly (dimanche)
        if now.weekday() == 6:
            latest = self._get_latest_backup('daily')
            if latest:
                weekly_link = latest.parent / f"{latest.name}_weekly_{now.strftime('%Y%W')}"
                weekly_link.symlink_to(latest.name)
                logger.info(f"✓ Promoted to weekly: {weekly_link.name}")

        # Promouvoir vers monthly (1er du mois)
        if now.day == 1:
            latest = self._get_latest_backup('weekly')
            if latest:
                monthly_link = latest.parent / f"{latest.name}_monthly_{now.strftime('%Y%m')}"
                monthly_link.symlink_to(latest.name)
                logger.info(f"✓ Promoted to monthly: {monthly_link.name}")

        # Promouvoir vers yearly (1er janvier)
        if now.month == 1 and now.day == 1:
            latest = self._get_latest_backup('monthly')
            if latest:
                yearly_link = latest.parent / f"{latest.name}_yearly_{now.year}"
                yearly_link.symlink_to(latest.name)
                logger.info(f"✓ Promoted to yearly: {yearly_link.name}")

    def _get_latest_backup(self, category: str) -> Path:
        """Obtenir le dernier backup d'une catégorie"""

        backups = [
            p for p in self.backup_root.glob('backup_*')
            if self.classify_backup(p) == category
        ]

        return max(backups, key=lambda p: p.stat().st_mtime) if backups else None

    def generate_report(self, output_file: str = None):
        """Générer rapport de rétention"""

        stats = {
            'timestamp': datetime.datetime.now().isoformat(),
            'policy': self.retention,
            'backups': {}
        }

        for backup_path in sorted(self.backup_root.glob('backup_*')):
            if not backup_path.is_dir():
                continue

            classification = self.classify_backup(backup_path)
            size = sum(f.stat().st_size for f in backup_path.rglob('*') if f.is_file())

            stats['backups'][backup_path.name] = {
                'classification': classification,
                'size_bytes': size,
                'modified': backup_path.stat().st_mtime
            }

        if output_file:
            with open(output_file, 'w') as f:
                json.dump(stats, f, indent=2)
            logger.info(f"✓ Report saved to: {output_file}")

        return stats


if __name__ == '__main__':
    # Configuration
    config = {
        'hourly': 24,      # 24 heures
        'daily': 7,        # 7 jours
        'weekly': 28,      # 4 semaines
        'monthly': 180,    # 6 mois
        'yearly': 2555     # 7 ans
    }

    manager = GFSRetentionManager('/backup/mongodb', config)

    # Dry run
    manager.apply_retention(dry_run=True)

    # Promouvoir
    manager.promote_backups()

    # Rapport
    manager.generate_report('/backup/mongodb/retention_report.json')
```

## Documentation Obligatoire

### Runbook de Restauration

```markdown
# MongoDB Restoration Runbook

## Document Information

**Version:** 2.1.0
**Last Updated:** 2024-12-08
**Owner:** Database Team
**Reviewers:** DevOps, Security, Management
**Next Review:** 2025-03-08

## Purpose

This runbook provides step-by-step procedures for restoring MongoDB databases
in various disaster scenarios.

## Prerequisites

- [ ] Access to backup storage (credentials in 1Password vault "DB-Backups")
- [ ] SSH access to production servers (bastion host required)
- [ ] MongoDB credentials (stored in AWS Secrets Manager)
- [ ] Approval from manager (for production restores)
- [ ] Communication channel open (#incident-response Slack)

## Scenarios

### Scenario 1: Complete Database Loss

**RTO:** 2 hours
**RPO:** 1 hour (latest backup)

**Detection:**
- MongoDB cluster completely unavailable
- All replica set members down
- Data directory corrupted/lost

**Procedure:**

1. **Assess & Communicate** (5 minutes)
   ```bash
   # Verify cluster is truly down
   mongo mongodb://primary:27017 --eval "db.adminCommand('ping')"

   # Alert stakeholders
   python3 /scripts/alert_incident.py --severity=critical \
     --title="MongoDB Complete Outage" \
     --channel="#incidents"
   ```

2. **Identify Latest Valid Backup** (10 minutes)
   ```bash
   # List recent backups
   aws s3 ls s3://company-backups/prod/mongodb/ --recursive | tail -20

   # Download manifest of latest
   aws s3 cp s3://company-backups/prod/mongodb/latest/MANIFEST.json /tmp/

   # Verify integrity
   jq . /tmp/MANIFEST.json
   ```

3. **Provision Infrastructure** (30 minutes)
   ```bash
   # If infrastructure lost, provision new cluster
   cd /infrastructure/mongodb-prod
   terraform apply -var="restore_mode=true"
   ```

4. **Download Backup** (20 minutes)
   ```bash
   # Download to restoration server
   aws s3 sync s3://company-backups/prod/mongodb/20241208_020000 \
     /restore/backup --no-progress

   # Verify checksums
   cd /restore/backup
   sha256sum -c SHA256SUMS
   ```

5. **Restore Data** (45 minutes)
   ```bash
   # Stop MongoDB if running
   systemctl stop mongod

   # Clean data directory
   rm -rf /var/lib/mongodb/*

   # Restore
   mongorestore \
     --uri="mongodb://localhost:27017" \
     --drop \
     --oplogReplay \
     --gzip \
     --numParallelCollections=8 \
     --dir=/restore/backup

   # Start MongoDB
   systemctl start mongod
   ```

6. **Validate** (15 minutes)
   ```bash
   # Run validation script
   /scripts/validate_restore.sh

   # Check key metrics
   mongo mongodb://localhost:27017 --eval "
     db.adminCommand({ listDatabases: 1 }).databases.forEach(printjson);
   "
   ```

7. **Reconnect Application** (5 minutes)
   ```bash
   # Update application configuration
   kubectl set env deployment/app MONGO_URI="mongodb://restored-cluster"

   # Monitor errors
   kubectl logs -f deployment/app
   ```

8. **Post-Restore** (ongoing)
   - Monitor database performance
   - Validate data integrity with business team
   - Document incident and lessons learned
   - Update runbook if procedures changed

### Scenario 2: Accidental Data Deletion

[Detailed procedure similar to above]

### Scenario 3: Ransomware Attack

[Detailed procedure similar to above]

## Contacts

**Primary On-Call:** +1-555-0100 (PagerDuty)
**Database Team Lead:** john.doe@company.com
**Security Team:** security@company.com
**Management (Critical):** cto@company.com

## Tools & Access

- Backup Storage: AWS S3 (us-east-1, us-west-2, eu-west-1)
- Credentials: 1Password vault "DB-Backups"
- Monitoring: https://grafana.company.com/mongodb
- Incident Channel: #incident-response
- Documentation: https://wiki.company.com/mongodb

## Post-Incident

- [ ] Complete incident report
- [ ] Update runbook with learnings
- [ ] Review backup strategy
- [ ] Test restore procedures
- [ ] Conduct post-mortem
```

## Checklists Opérationnelles

### Checklist Quotidienne

```markdown
# MongoDB Backup - Daily Checklist

Date: ___________
Operator: ___________

## Morning (9:00 AM)

- [ ] Verify last night's backup completed successfully
      Log: /var/log/mongodb-backup/latest.log
      Expected completion: 02:30 AM ± 15 min

- [ ] Check backup size is within expected range
      Previous: _______ GB
      Current: _______ GB
      Variation: _______ % (alert if > ±30%)

- [ ] Verify upload to S3 completed
      Command: aws s3 ls s3://company-backups/prod/mongodb/ | tail -1

- [ ] Review monitoring dashboard
      URL: https://grafana.company.com/mongodb-backups
      All metrics green: YES / NO

- [ ] Check disk space on backup servers
      Backup server 1: _______ % used (alert if > 80%)
      Backup server 2: _______ % used (alert if > 80%)

## Issues Found

_______________________________________________________________________
_______________________________________________________________________

## Actions Taken

_______________________________________________________________________
_______________________________________________________________________

## Escalations

_______________________________________________________________________
_______________________________________________________________________

Signature: ___________________
```

### Checklist Hebdomadaire

```markdown
# MongoDB Backup - Weekly Checklist

Week of: ___________
Operator: ___________

## Sunday Evening

- [ ] Weekly full backup completed
- [ ] Backup promoted to "weekly" tier (GFS)
- [ ] Test restore executed automatically
      Result: PASS / FAIL
      Log: /var/log/mongodb-restore-tests/latest.log

- [ ] Review test restore results
      Databases restored: _______ (expected: _______)
      Collections restored: _______ (expected: _______)
      Validation errors: _______ (expected: 0)

- [ ] Review backup retention
      Daily backups: _______ (expected: 7)
      Weekly backups: _______ (expected: 4)
      Monthly backups: _______ (expected: varies)

- [ ] Check oplog window
      Current window: _______ hours
      Target: ≥ 72 hours
      Status: OK / WARNING / CRITICAL

- [ ] Review backup failures log
      Failures this week: _______
      Root causes identified: YES / NO

- [ ] Update documentation if procedures changed
      Changes made: _______________________________________

## Actions Required

_______________________________________________________________________
_______________________________________________________________________
```

## Métriques et KPIs

### Dashboard de Métriques

```yaml
MongoDB Backup & Restore - Key Performance Indicators

Métriques de Disponibilité:
  backup_success_rate:
    calcul: "successful_backups / total_backups * 100"
    target: "> 99%"
    alerte: "< 95%"

  restore_test_success_rate:
    calcul: "successful_restore_tests / total_tests * 100"
    target: "100%"
    alerte: "< 100%"

  backup_window_compliance:
    description: "Backups complétés dans fenêtre planifiée"
    target: "> 95%"
    alerte: "< 90%"

Métriques de Performance:
  backup_duration:
    description: "Durée moyenne backup full"
    target: "< 2 heures"
    alerte: "> 3 heures"

  restore_duration:
    description: "Durée restauration (RTO réel)"
    target: "< objectif RTO"
    alerte: "> objectif RTO"

  backup_size_growth:
    description: "Croissance mensuelle taille backups"
    target: "< 20% par mois"
    alerte: "> 30% par mois"

Métriques de Couverture:
  oplog_window:
    description: "Fenêtre temporelle oplog"
    target: "> 72 heures"
    alerte: "< 48 heures"

  pitr_capability:
    description: "Fenêtre PITR disponible"
    target: "> 48 heures"
    alerte: "< 24 heures"

  backup_coverage:
    description: "% données couvertes par backup"
    target: "100%"
    alerte: "< 100%"

Métriques de Coût:
  cost_per_gb:
    description: "Coût par GB stocké"
    target: "< $0.03/GB/mois"
    monitoring: "mensuel"

  total_storage_cost:
    description: "Coût total stockage backups"
    budget: "[selon environment]"
    monitoring: "mensuel"

Métriques de Qualité:
  validation_errors:
    description: "Erreurs validation backups"
    target: "0"
    alerte: "> 0"

  data_loss_incidents:
    description: "Incidents perte de données"
    target: "0"
    monitoring: "continu"

  mttr_restore:
    description: "Mean Time To Restore"
    target: "< RTO"
    monitoring: "par incident"
```

## Erreurs Communes à Éviter

### Top 10 des Erreurs Critiques

```markdown
# Top 10 MongoDB Backup Mistakes (& Solutions)

## 1. ❌ Backups Jamais Testés
**Symptôme:** Backups réguliers mais aucun test de restauration
**Impact:** Découverte que les backups sont inutilisables lors d'un incident réel
**Solution:**
- Tests automatisés hebdomadaires minimum
- DR drills trimestriels
- Documentation des résultats

## 2. ❌ Oplog Dimensionné Insuffisamment
**Symptôme:** Fenêtre oplog < 24 heures
**Impact:** PITR impossible, pertes de données
**Solution:**
```bash
# Calculer taille nécessaire
use local
db.oplog.rs.stats().maxSize / 1024 / 1024 / 1024  # GB

# Redimensionner
db.adminCommand({ replSetResizeOplog: 1, size: 204800 })  # 200 GB
```

## 3. ❌ Backup Sans --oplog
**Symptôme:** mongodump sans option --oplog
**Impact:** Backups inconsistants, PITR impossible
**Solution:**
```bash
# Toujours inclure --oplog
mongodump --uri="..." --oplog --out=/backup
```

## 4. ❌ Tous les Backups Au Même Endroit
**Symptôme:** Backups uniquement locaux ou dans une seule région
**Impact:** Perte complète lors d'un sinistre du data center
**Solution:**
- Règle 3-2-1-1-0
- Multi-région obligatoire
- Air-gap pour données critiques

## 5. ❌ Pas de Chiffrement
**Symptôme:** Backups en clair
**Impact:** Violation GDPR, fuite de données
**Solution:**
```bash
# Chiffrement systématique
tar -czf - /backup | openssl enc -aes-256-cbc -out backup.enc
```

## 6. ❌ Credentials en Dur dans Scripts
**Symptôme:** Mots de passe en clair dans scripts
**Impact:** Sécurité compromise
**Solution:**
- Variables d'environnement
- Secrets managers (AWS Secrets Manager, Vault)
- Fichiers credentials avec permissions 600

## 7. ❌ Pas de Monitoring des Backups
**Symptôme:** Échecs de backup non détectés
**Impact:** Découverte tardive, perte de données
**Solution:**
- Alertes temps réel
- Dashboard dédié
- PagerDuty pour production

## 8. ❌ Ignorer les Sharded Clusters
**Symptôme:** Backup de shards sans coordination
**Impact:** Incohérence entre shards
**Solution:**
- Arrêter balancer
- Backup coordonné de config + tous shards
- Documenter topologie

## 9. ❌ Pas de Rotation/Retention
**Symptôme:** Backups s'accumulent infiniment
**Impact:** Coûts explosifs, disque plein
**Solution:**
- Politique GFS claire
- Automation de la rotation
- Monitoring de l'utilisation

## 10. ❌ Documentation Obsolète
**Symptôme:** Procédures non à jour
**Impact:** Confusion lors d'incident, RTO dépassé
**Solution:**
- Mise à jour après chaque changement
- Review mensuelle
- Versioning des runbooks
```

## Matrice de Décision

### Choisir sa Stratégie de Backup

```
┌───────────────────────────────────────────────────────────────┐
│         Arbre de Décision - Stratégie de Backup               │
│                                                               │
│  Quelle est la criticité de vos données?                      │
│                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐  │
│  │   Faible     │   │   Moyenne    │   │   Critique        │  │
│  │  (Dev/Test)  │   │  (Prod Std)  │   │ (Mission-Critical)│  │
│  └──────┬───────┘   └──────┬───────┘   └────────┬──────────┘  │
│         │                  │                    │             │
│         v                  v                    v             │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐  │
│  │ mongodump    │   │ mongodump +  │   │ Atlas / Ops Mgr + │  │
│  │ quotidien    │   │ snapshots +  │   │ Continuous Backup │  │
│  │ local        │   │ oplog stream │   │ + Multi-région    │  │
│  │              │   │ S3           │   │ + Immutable       │  │
│  └──────────────┘   └──────────────┘   └───────────────────┘  │
│                                                               │
│  Quel est votre RTO acceptable?                               │
│                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐  │
│  │   > 4h       │   │  1-4h        │   │   < 1h            │  │
│  └──────┬───────┘   └──────┬───────┘   └────────┬──────────┘  │
│         │                  │                    │             │
│         v                  v                    v             │
│  Daily backups      Hourly snapshots     Hot standby + PITR   │
│  Test restore       + PITR window        + Automated failover │
│  monthly            Test restore          Test restore weekly │
│                     weekly                                    │
│                                                               │
│  Quel est votre budget?                                       │
│                                                               │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐  │
│  │  < $500/m    │   │ $500-3k/m    │   │   > $3k/m         │  │
│  └──────┬───────┘   └──────┬───────┘   └────────┬──────────┘  │
│         │                  │                    │             │
│         v                  v                    v             │
│  Self-managed       Self-managed        MongoDB Atlas         │
│  Basic automation   Full automation     Fully managed         │
│  Local + S3         Multi-région        Enterprise support    │
│                     Monitoring                                │
└───────────────────────────────────────────────────────────────┘
```

## Conformité et Audit

### Checklist de Conformité

```yaml
GDPR Compliance (Europe):
  - [ ] Données personnelles chiffrées at-rest et in-transit
  - [ ] Accès aux backups audité et tracé
  - [ ] Capacité de suppression (droit à l'oubli)
  - [ ] Rétention max 7 ans (sauf obligation légale)
  - [ ] Transferts hors UE documentés et conformes
  - [ ] DPO informé de la stratégie backup
  - [ ] Incident response plan documenté

HIPAA Compliance (US Healthcare):
  - [ ] Chiffrement AES-256 minimum
  - [ ] Accès restreint (minimum necessary)
  - [ ] Audit trail complet (tous accès tracés)
  - [ ] Backups testés annuellement minimum
  - [ ] BAA (Business Associate Agreement) avec providers
  - [ ] Integrity controls (checksums)
  - [ ] Disaster recovery plan testé

SOC 2 Type II:
  - [ ] Politiques backup documentées
  - [ ] Tests de restauration réguliers
  - [ ] Monitoring et alerting actifs
  - [ ] Gestion des changements documentée
  - [ ] Revue des accès trimestrielle
  - [ ] Documentation des incidents
  - [ ] Audit trail complet

PCI-DSS (Payment Card Data):
  - [ ] Chiffrement fort (AES-256)
  - [ ] Backups isolés du réseau production
  - [ ] Tests de restauration trimestriels
  - [ ] Rétention max 3 mois (cardholder data)
  - [ ] Secure deletion (overwrite)
  - [ ] Accès strictement contrôlé
  - [ ] Logging de tous les accès

ISO 27001:
  - [ ] Politique de backup formelle
  - [ ] Risk assessment documenté
  - [ ] Procédures de restauration testées
  - [ ] Gestion des incidents
  - [ ] Formation du personnel
  - [ ] Revues management régulières
  - [ ] Amélioration continue documentée
```

### Audit Trail

```javascript
// audit_backup_access.js
// Script pour tracker tous les accès aux backups

use admin

// Activer l'audit
db.adminCommand({
  setParameter: 1,
  auditLog: {
    destination: "file",
    path: "/var/log/mongodb/audit.json",
    filter: {
      // Auditer tous les accès aux backups
      "atype": { "$in": ["authenticate", "authCheck"] },
      "param.db": { "$in": ["admin", "config"] }
    },
    format: "JSON"
  }
});

// Query pour analyser les accès backups
db.getSiblingDB("audit").log.aggregate([
  {
    $match: {
      "users.user": "backup-operator",
      "ts": {
        $gte: new Date(Date.now() - 30*24*60*60*1000)  // 30 derniers jours
      }
    }
  },
  {
    $group: {
      _id: {
        user: "$users.user",
        operation: "$param.command"
      },
      count: { $sum: 1 },
      lastAccess: { $max: "$ts" }
    }
  },
  {
    $sort: { count: -1 }
  }
]);
```

## Framework de Continuité d'Activité

### Business Continuity Plan (BCP)

```markdown
# MongoDB Business Continuity Plan

## 1. Business Impact Analysis

### Données Critiques
| Database | Criticité | RPO | RTO | Impact Financier |
|----------|-----------|-----|-----|------------------|
| Orders   | Critique  | 15min | 30min | $10k/heure |
| Users    | Haute     | 1h  | 1h  | $2k/heure |
| Logs     | Moyenne   | 24h | 4h  | $200/heure |

### Scénarios de Disaster

**Scenario 1: Panne Data Center**
- Probabilité: Faible (1/10 ans)
- Impact: Très élevé
- Mitigation: Multi-région actif

**Scenario 2: Corruption de Données**
- Probabilité: Moyenne (1/an)
- Impact: Moyen
- Mitigation: PITR + Backups fréquents

**Scenario 3: Ransomware**
- Probabilité: Élevée (secteur cible)
- Impact: Très élevé
- Mitigation: Immutabilité + Air-gap

## 2. Recovery Strategies

### Tier 1 (Critique)
- **Strategy:** Hot standby multi-région
- **Failover:** Automatique (< 5 min)
- **Testing:** Mensuel

### Tier 2 (Haute)
- **Strategy:** Warm standby + PITR
- **Failover:** Manuel (< 1h)
- **Testing:** Trimestriel

### Tier 3 (Moyenne)
- **Strategy:** Backups + Snapshots
- **Recovery:** Best effort (< 4h)
- **Testing:** Semestriel

## 3. Team Responsibilities

### Database Team (Primary)
- Monitoring 24/7
- Incident response
- Backup execution & validation
- Documentation

### DevOps Team (Support)
- Infrastructure provisioning
- Automation
- Deployment recovery

### Security Team
- Access control
- Encryption
- Audit compliance

## 4. Communication Plan

### Escalation Path
1. On-Call DBA (0-15 min)
2. Database Team Lead (15-30 min)
3. CTO (30-60 min)
4. CEO (if business impact > $50k)

### Stakeholder Notifications
- **Internal:** #incidents Slack (immediate)
- **Customers:** Status page (< 30 min)
- **Management:** Email summary (< 1h)

## 5. Post-Incident

- [ ] Root cause analysis (48h)
- [ ] Post-mortem (1 week)
- [ ] Action items tracking
- [ ] BCP update if needed
- [ ] Team debrief
```

## Conclusion

Les bonnes pratiques de sauvegarde et restauration ne sont pas statiques - elles évoluent avec votre organisation, vos technologies et les menaces. Un programme de backup mature repose sur trois piliers :

### 1. **Excellence Technique**
- Architecture résiliente (3-2-1-1-0)
- Automation maximale
- Monitoring proactif
- Tests réguliers obligatoires

### 2. **Excellence Opérationnelle**
- Procédures documentées et testées
- Équipe formée et prête
- Communication claire
- Amélioration continue

### 3. **Excellence Organisationnelle**
- Alignement business-IT
- Budget approprié
- Compliance garantie
- Culture de la résilience

### Minimum Viable pour Production

**Must-Have absolu** :
- ✅ Backups quotidiens automatisés
- ✅ Stockage multi-région
- ✅ Tests mensuels minimum
- ✅ Monitoring avec alertes
- ✅ Documentation à jour
- ✅ Chiffrement des backups
- ✅ Plan de restoration testé

**Nice-to-Have recommandé** :
- ⭐ PITR (48h minimum)
- ⭐ Immutabilité (ransomware protection)
- ⭐ DR site (warm/hot standby)
- ⭐ Automation complète
- ⭐ Tests hebdomadaires
- ⭐ Dashboards temps réel

**Excellence (Mission-Critical)** :
- 🏆 MongoDB Atlas / Ops Manager
- 🏆 Multi-région actif-actif
- 🏆 RTO < 30 minutes
- 🏆 RPO < 15 minutes
- 🏆 Tests quotidiens automatisés
- 🏆 Équipe dédiée 24/7

### Le Test Ultime

**Votre stratégie de backup est-elle vraiment solide ?**

Posez-vous ces questions :
1. ✅ Pouvez-vous restaurer en production **maintenant** avec confiance ?
2. ✅ Votre équipe peut-elle restaurer **sans vous** ?
3. ✅ Vos backups survivraient-ils à un **ransomware** ?
4. ✅ Respectez-vous votre **RTO/RPO promis** ?
5. ✅ Votre dernière restauration remonte à **quand** ? (doit être < 1 mois)

Si vous avez répondu **OUI** à tout : Bravo ! 🎉

Sinon : Ce chapitre vous donne tous les outils pour y arriver.

**Remember:** *Un backup non testé n'est qu'une illusion de sécurité.*

---


⏭️ [Monitoring et Administration](/13-monitoring-administration/README.md)
