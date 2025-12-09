🔝 Retour au [Sommaire](/SOMMAIRE.md)

# G. Scripts et Automatisation

## Vue d'ensemble

Cette annexe fournit une collection de **scripts pratiques et directement utilisables** pour automatiser les tâches courantes d'administration MongoDB. Ces scripts sont conçus pour faciliter le quotidien des développeurs et des DevOps, quel que soit leur niveau d'expertise.

## Objectifs

- **Gagner du temps** : Automatiser les tâches répétitives
- **Réduire les erreurs** : Standardiser les opérations critiques
- **Améliorer la fiabilité** : Assurer la cohérence des opérations
- **Faciliter la maintenance** : Documenter les procédures opérationnelles

## Structure de l'annexe

| Section | Description | Niveau |
|---------|-------------|--------|
| **G.1 Scripts de backup** | Sauvegarde automatisée (mongodump, snapshots) | Tous |
| **G.2 Scripts de monitoring** | Surveillance et alertes | Intermédiaire |
| **G.3 Scripts de maintenance** | Nettoyage, optimisation, rotation des logs | Intermédiaire |
| **G.4 Playbooks Ansible** | Automatisation infrastructure MongoDB | Avancé |

## Prérequis techniques

### Outils nécessaires

```bash
# Vérifier les outils installés
mongo --version           # MongoDB Shell
mongodump --version       # Database Tools
mongorestore --version    # Database Tools
mongosh --version         # MongoDB Shell moderne
```

### Variables d'environnement recommandées

```bash
# Fichier .env ou /etc/mongodb/config.env
export MONGO_HOST="localhost"
export MONGO_PORT="27017"
export MONGO_USER="admin"
export MONGO_PASSWORD="votre_mot_de_passe"
export MONGO_AUTH_DB="admin"
export MONGO_BACKUP_DIR="/backup/mongodb"
export MONGO_LOG_DIR="/var/log/mongodb"
export RETENTION_DAYS=30
```

## Bonnes pratiques générales

### 1. Sécurité

```bash
# ✅ Protéger les credentials
chmod 600 /etc/mongodb/.env
chown mongodb:mongodb /etc/mongodb/.env

# ✅ Utiliser des fichiers de configuration sécurisés
# Éviter les mots de passe en clair dans les scripts

# ✅ Logs sensibles
# Rediriger les sorties vers des fichiers avec permissions restreintes
chmod 640 /var/log/mongodb/backup.log
```

### 2. Logging et traçabilité

```bash
# Format de log standardisé
LOG_FILE="/var/log/mongodb/script-$(date +%Y%m%d).log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

log "INFO: Démarrage du script"
log "ERROR: Échec de l'opération"
log "SUCCESS: Opération terminée avec succès"
```

### 3. Gestion des erreurs

```bash
# Arrêter le script en cas d'erreur
set -e          # Exit on error
set -u          # Exit on undefined variable
set -o pipefail # Exit on pipe failure

# Fonction de gestion d'erreur personnalisée
error_exit() {
    log "ERROR: $1"
    # Notification (email, Slack, etc.)
    exit 1
}

# Utilisation
mongodump --uri="$MONGO_URI" || error_exit "Échec du backup"
```

### 4. Notifications

```bash
# Email
send_email() {
    echo "$2" | mail -s "$1" admin@example.com
}

# Slack
send_slack() {
    curl -X POST -H 'Content-type: application/json' \
    --data "{\"text\":\"$1\"}" \
    "$SLACK_WEBHOOK_URL"
}

# Exemple d'utilisation
if [ $BACKUP_SUCCESS -eq 0 ]; then
    send_slack "✅ Backup MongoDB réussi"
else
    send_slack "❌ Échec du backup MongoDB - Action requise"
fi
```

### 5. Validation des sauvegardes

```bash
# Toujours vérifier l'intégrité après un backup
validate_backup() {
    local backup_dir=$1

    if [ ! -d "$backup_dir" ]; then
        error_exit "Répertoire de backup introuvable"
    fi

    # Vérifier la présence de fichiers
    file_count=$(find "$backup_dir" -type f | wc -l)
    if [ "$file_count" -eq 0 ]; then
        error_exit "Aucun fichier dans le backup"
    fi

    log "SUCCESS: Backup validé - $file_count fichiers"
}
```

## Configuration type pour scripts

### Script de base réutilisable

```bash
#!/bin/bash
#
# Script: nom_du_script.sh
# Description: Description du script
# Auteur: Votre nom
# Date: $(date +%Y-%m-%d)
#

# Configuration
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
LOG_DIR="/var/log/mongodb"
LOG_FILE="$LOG_DIR/$(basename "$0" .sh)-$(date +%Y%m%d).log"

# Sourcer les variables d'environnement
[ -f "$SCRIPT_DIR/.env" ] && source "$SCRIPT_DIR/.env"

# Fonctions utilitaires
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

error_exit() {
    log "ERROR: $1"
    exit 1
}

# Validation des prérequis
check_prerequisites() {
    command -v mongodump >/dev/null 2>&1 || error_exit "mongodump non installé"
    [ -z "${MONGO_URI:-}" ] && error_exit "MONGO_URI non défini"
}

# Fonction principale
main() {
    log "INFO: Démarrage du script"
    check_prerequisites

    # Votre logique ici

    log "SUCCESS: Script terminé avec succès"
}

# Exécution
main "$@"
```

## Planification avec Cron

### Exemples de crontab

```cron
# Éditer le crontab
crontab -e

# Backup quotidien à 2h du matin
0 2 * * * /opt/mongodb/scripts/backup-daily.sh >> /var/log/mongodb/cron-backup.log 2>&1

# Monitoring toutes les 5 minutes
*/5 * * * * /opt/mongodb/scripts/monitor-health.sh

# Maintenance hebdomadaire (dimanche à 3h)
0 3 * * 0 /opt/mongodb/scripts/maintenance-weekly.sh

# Nettoyage mensuel (1er du mois à 4h)
0 4 1 * * /opt/mongodb/scripts/cleanup-old-backups.sh
```

### Bonnes pratiques cron

```bash
# 1. Toujours spécifier le chemin complet
# ❌ Mauvais
0 2 * * * backup.sh

# ✅ Bon
0 2 * * * /opt/mongodb/scripts/backup.sh

# 2. Rediriger les sorties
0 2 * * * /opt/mongodb/scripts/backup.sh >> /var/log/mongodb/cron.log 2>&1

# 3. Définir l'environnement dans le script
# Ne pas se fier aux variables d'environnement du cron
```

## Outils d'automatisation recommandés

### 1. Systemd Timers (alternative à cron)

```ini
# /etc/systemd/system/mongodb-backup.timer
[Unit]
Description=MongoDB Backup Timer
Requires=mongodb-backup.service

[Timer]
OnCalendar=daily
OnCalendar=02:00
Persistent=true

[Install]
WantedBy=timers.target
```

### 2. Ansible (voir G.4)

```yaml
# Automatisation complète de l'infrastructure
- hosts: mongodb_servers
  roles:
    - mongodb-backup
    - mongodb-monitoring
```

### 3. Systemd Services

```ini
# /etc/systemd/system/mongodb-backup.service
[Unit]
Description=MongoDB Backup Service
After=mongod.service

[Service]
Type=oneshot
User=mongodb
ExecStart=/opt/mongodb/scripts/backup.sh
StandardOutput=journal
StandardError=journal
```

## Structure de fichiers recommandée

```
/opt/mongodb/
├── scripts/
│   ├── .env                    # Variables d'environnement
│   ├── backup/
│   │   ├── backup-daily.sh
│   │   ├── backup-weekly.sh
│   │   └── restore.sh
│   ├── monitoring/
│   │   ├── health-check.sh
│   │   ├── disk-usage.sh
│   │   └── replicaset-status.sh
│   ├── maintenance/
│   │   ├── compact-collections.sh
│   │   ├── rebuild-indexes.sh
│   │   └── cleanup-logs.sh
│   └── lib/
│       ├── common.sh           # Fonctions partagées
│       └── notifications.sh    # Système de notifications
│
├── config/
│   └── mongodb.conf
│
└── logs/
    ├── backup/
    ├── monitoring/
    └── maintenance/
```

## Checklist avant déploiement

- [ ] **Tests** : Tous les scripts testés en environnement de développement
- [ ] **Permissions** : Fichiers exécutables et droits appropriés
- [ ] **Credentials** : Variables d'environnement configurées et sécurisées
- [ ] **Logs** : Répertoires de logs créés avec permissions correctes
- [ ] **Notifications** : Système d'alertes configuré et testé
- [ ] **Documentation** : Instructions de maintenance documentées
- [ ] **Monitoring** : Supervision des jobs planifiés active
- [ ] **Backup** : Scripts de sauvegarde testés avec restauration
- [ ] **Rollback** : Procédure de retour arrière documentée

## Ressources et liens utiles

### Documentation officielle
- [MongoDB Database Tools](https://www.mongodb.com/docs/database-tools/)
- [mongosh Documentation](https://www.mongodb.com/docs/mongodb-shell/)
- [MongoDB Administration](https://www.mongodb.com/docs/manual/administration/)

### Outils tiers
- **Percona Backup for MongoDB** : Solution de backup enterprise
- **MongoDB Ops Manager** : Plateforme d'automatisation complète
- **Ansible MongoDB Role** : Collection officielle Ansible

### Communauté
- [MongoDB Community Forums](https://www.mongodb.com/community/forums/)
- [Stack Overflow - MongoDB](https://stackoverflow.com/questions/tagged/mongodb)

---

## Comment utiliser cette annexe

1. **Parcourez** les sections pour identifier vos besoins
2. **Adaptez** les scripts à votre environnement
3. **Testez** en développement avant la production
4. **Planifiez** l'exécution selon vos contraintes
5. **Surveillez** l'exécution et ajustez si nécessaire

> **💡 Astuce** : Commencez par les scripts de backup (G.1) - c'est la priorité absolue pour toute installation MongoDB.

---

**Prochaine section** : G.1 Scripts de backup

⏭️ [Scripts de backup](/annexes/scripts-automatisation/01-scripts-backup.md)
