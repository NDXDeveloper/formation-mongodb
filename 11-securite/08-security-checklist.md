🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.8 Security Checklist

## Introduction

Cette checklist consolidée couvre tous les aspects de sécurité MongoDB abordés dans les sections précédentes. Elle est conçue pour être utilisée lors du déploiement initial, des audits de sécurité réguliers, et de la préparation aux certifications de conformité.

**Comment utiliser cette checklist** :

```
☐ Checklist item à compléter
✓ Checklist item complété
⚠️ Checklist item nécessitant attention
❌ Checklist item non applicable
N/A Non applicable à votre environnement
```

**Structure** :
1. Checklist par catégorie (authentification, réseau, chiffrement, etc.)
2. Checklist par environnement (développement, staging, production)
3. Checklist par conformité (PCI-DSS, HIPAA, SOX, RGPD)
4. Scripts d'audit automatisé
5. Matrice de risques et remediation

## Checklist par catégorie

### 1. Authentification et Autorisation

#### 1.1 Configuration de base

```
☐ L'authentification est activée (security.authorization: enabled)
☐ Utilisateur root/admin créé avec mot de passe fort (>16 caractères)
☐ Mécanisme SCRAM-SHA-256 activé (minimum, SCRAM-SHA-1 désactivé)
☐ Aucun utilisateur avec mot de passe par défaut ou vide
☐ Base de données admin protégée (accès restreint)
☐ Utilisateurs applicatifs créés avec privilèges minimaux
☐ Utilisateurs de monitoring créés (lecture seule sur admin/config)
☐ Utilisateurs de backup créés avec privilèges appropriés
```

**Commande de vérification** :

```bash
# Vérifier que l'authentification est activée
mongosh --eval "db.adminCommand({ getParameter: 1, authenticationMechanisms: 1 })"

# Lister tous les utilisateurs
mongosh admin --eval "db.system.users.find({}, {user:1, db:1, roles:1}).pretty()"

# Vérifier les mécanismes d'authentification
mongosh --eval "db.adminCommand({ getParameter: 1, authenticationMechanisms: 1 })"
```

#### 1.2 RBAC (Role-Based Access Control)

```
☐ Principe du moindre privilège appliqué à tous les utilisateurs
☐ Rôles personnalisés créés pour besoins spécifiques
☐ Aucun utilisateur avec rôle root en production (sauf admin d'urgence)
☐ Séparation des rôles : read, readWrite, dbAdmin, userAdmin
☐ Rôles de cluster (clusterAdmin, clusterManager) limités
☐ Documentation des rôles et privilèges maintenue
☐ Revue trimestrielle des accès utilisateurs
☐ Procédure d'offboarding (suppression utilisateurs partants)
```

**Script de validation** :

```javascript
// validate-rbac.js
const dangerousRoles = ['root', 'dbOwner', 'userAdminAnyDatabase', 'dbAdminAnyDatabase'];

db.getSiblingDB('admin').system.users.find().forEach(function(user) {
  user.roles.forEach(function(role) {
    if (dangerousRoles.includes(role.role)) {
      print("⚠️  WARNING: User '" + user.user + "' has dangerous role: " + role.role);
    }
  });
});

// Vérifier les utilisateurs sans authentification forte
db.getSiblingDB('admin').system.users.find({
  "credentials.SCRAM-SHA-256": { $exists: false }
}).forEach(function(user) {
  print("❌ ERROR: User '" + user.user + "' does not use SCRAM-SHA-256");
});
```

#### 1.3 Authentification externe (si applicable)

```
☐ LDAP/Active Directory configuré (si utilisé)
☐ Kerberos configuré (si utilisé)
☐ x.509 certificates pour authentification cluster configurés
☐ Mapping LDAP → MongoDB roles documenté
☐ Tests de failover LDAP réalisés
☐ Timeouts LDAP appropriés (10-30s)
☐ Pool de connexions LDAP optimisé
☐ Monitoring LDAP en place
```

### 2. Chiffrement

#### 2.1 Chiffrement en transit (TLS/SSL)

```
☐ TLS activé et requis (net.tls.mode: requireTLS)
☐ Certificats valides et non expirés (vérifier dates)
☐ Certificats signés par CA reconnue (pas self-signed en prod)
☐ Certificats avec SAN appropriés (tous les hostnames)
☐ TLS 1.2 minimum (TLS 1.0/1.1 désactivés)
☐ Cipher suites forts uniquement (AES-GCM, ECDHE)
☐ Client certificate authentication activée (mTLS, optionnel)
☐ Rotation des certificats planifiée (30 jours avant expiration)
☐ Procédure de renouvellement automatisée
☐ Monitoring expiration certificats en place
```

**Commandes de vérification** :

```bash
# Vérifier TLS
mongosh --tls --tlsCAFile /etc/ssl/mongodb/ca.pem \
  --eval "db.adminCommand({ getParameter: 1, tlsMode: 1 })"

# Vérifier les certificats
openssl x509 -in /etc/ssl/mongodb/server.pem -text -noout | grep -A 2 "Validity"
openssl x509 -in /etc/ssl/mongodb/server.pem -text -noout | grep -A 1 "Subject Alternative Name"

# Tester la connexion TLS
openssl s_client -connect mongodb.example.com:27017 -CAfile /etc/ssl/mongodb/ca.pem
```

#### 2.2 Chiffrement au repos (Encryption at Rest)

```
☐ Encryption at Rest activé (Enterprise uniquement)
☐ KMS externe configuré (AWS KMS, Azure Key Vault, KMIP)
☐ Master Key stockée de manière sécurisée
☐ Aucun keyfile local en production
☐ Rotation des Master Keys planifiée (annuelle minimum)
☐ Procédure de rotation testée
☐ Backup des Master Keys (séparé des données)
☐ Encryption validée sur toutes les collections
☐ Monitoring de l'état du chiffrement
```

**Script de validation** :

```bash
#!/bin/bash
# verify-encryption-at-rest.sh

# Vérifier que le chiffrement est activé
mongosh --eval "
  var status = db.serverStatus();
  if (status.hasOwnProperty('security') && status.security.hasOwnProperty('encryptionAtRest')) {
    print('✓ Encryption at Rest is enabled');
    printjson(status.security.encryptionAtRest);
  } else {
    print('❌ Encryption at Rest is NOT enabled');
  }
"

# Vérifier qu'il n'y a pas de données en clair sur le disque
echo "Checking for plaintext data on disk..."
strings /data/mongodb/*.wt | grep -i "password\|credit\|ssn" | head -5
if [ $? -eq 0 ]; then
  echo "⚠️  WARNING: Potential plaintext data found on disk"
else
  echo "✓ No obvious plaintext data found"
fi
```

#### 2.3 Chiffrement côté client (CSFLE / Queryable Encryption)

```
☐ CSFLE configuré pour données sensibles (si applicable)
☐ Key Vault collection protégée
☐ CMK stockée dans KMS externe
☐ DEKs créées pour chaque type de données sensibles
☐ Schéma de chiffrement documenté
☐ Algorithme approprié choisi (Deterministic vs Random)
☐ Queryable Encryption utilisé si recherche nécessaire
☐ Compaction des métadonnées planifiée (Queryable Encryption)
☐ Performance impact mesuré et acceptable
☐ Backup du Key Vault planifié
```

### 3. Sécurité réseau

#### 3.1 Configuration réseau de base

```
☐ bindIp configuré (jamais 0.0.0.0 en production)
☐ MongoDB non exposé à Internet directement
☐ Port par défaut (27017) changé (optionnel mais recommandé)
☐ Firewall activé sur tous les serveurs MongoDB
☐ Règles firewall restrictives (deny all par défaut)
☐ Seules les IPs applicatives autorisées
☐ Accès inter-serveurs Replica Set autorisé
☐ Rate limiting configuré (protection DDoS)
☐ Monitoring des connexions actives
```

**Commandes de vérification** :

```bash
# Vérifier bindIp
grep "bindIp:" /etc/mongod.conf

# Vérifier que MongoDB n'écoute pas sur 0.0.0.0
netstat -tuln | grep 27017

# Vérifier les règles firewall
iptables -L INPUT -n -v | grep 27017
firewall-cmd --list-all | grep 27017
ufw status | grep 27017

# Tester l'accessibilité publique
PUBLIC_IP=$(curl -s ifconfig.me)
timeout 5 bash -c "echo > /dev/tcp/$PUBLIC_IP/27017" && echo "❌ EXPOSED" || echo "✓ Protected"
```

#### 3.2 Segmentation réseau

```
☐ VPC/VNET configuré avec subnets séparés
☐ DMZ pour load balancers/API gateways
☐ Application tier isolé
☐ Database tier isolé (pas d'accès direct depuis Internet)
☐ Management network séparé
☐ Security Groups / NSGs configurés
☐ Network ACLs en place (couche supplémentaire)
☐ Pas de communication directe entre DMZ et Database tier
☐ VPC Peering configuré (si cloud multi-VPC)
☐ Private Endpoints utilisés (AWS PrivateLink, etc.)
```

#### 3.3 Accès administratif

```
☐ Bastion host / Jump server configuré
☐ VPN pour accès distant
☐ SSH keys uniquement (pas de passwords)
☐ fail2ban ou équivalent activé
☐ MFA activé pour accès administratif
☐ SSH hardening appliqué (algorithmes forts uniquement)
☐ Logs d'accès centralisés
☐ Session recording activé (optionnel)
☐ IP whitelist pour accès bastion
☐ Rotation des clés SSH planifiée
```

### 4. Audit et Logging

#### 4.1 Configuration audit

```
☐ Audit activé (auditLog configuré, Enterprise uniquement)
☐ Filtre d'audit approprié au cas d'usage
☐ Destination audit configurée (file, syslog, console)
☐ Format JSON (pas BSON)
☐ Rotation des logs configurée (logrotate)
☐ Rétention conforme aux exigences réglementaires
☐ Archivage vers stockage froid planifié
☐ Logs d'audit protégés (lecture seule pour non-admins)
☐ Intégration SIEM configurée (ELK, Splunk, etc.)
☐ Alertes sur événements critiques activées
```

**Événements critiques à auditer** :

```yaml
☐ Authentications (succès et échecs)
☐ Créations/suppressions d'utilisateurs
☐ Modifications de rôles/privilèges
☐ Suppressions de collections/databases
☐ Accès aux données sensibles (si identifiables)
☐ Shutdown du serveur
☐ Modifications de configuration
☐ Reconfigurations Replica Set
```

**Script de validation** :

```bash
#!/bin/bash
# validate-audit.sh

# Vérifier que l'audit est activé
if grep -q "auditLog:" /etc/mongod.conf; then
  echo "✓ Audit is configured"
  grep -A 5 "auditLog:" /etc/mongod.conf
else
  echo "❌ Audit is NOT configured"
  exit 1
fi

# Vérifier que le fichier d'audit existe et est récent
AUDIT_FILE="/var/log/mongodb/audit.json"
if [ -f "$AUDIT_FILE" ]; then
  LAST_MODIFIED=$(stat -c %Y "$AUDIT_FILE")
  NOW=$(date +%s)
  DIFF=$((NOW - LAST_MODIFIED))

  if [ $DIFF -lt 3600 ]; then
    echo "✓ Audit log is active (last modified ${DIFF}s ago)"
  else
    echo "⚠️  WARNING: Audit log last modified ${DIFF}s ago"
  fi
else
  echo "❌ Audit log file not found: $AUDIT_FILE"
fi

# Vérifier les événements récents
if [ -f "$AUDIT_FILE" ]; then
  echo "Recent audit events:"
  tail -5 "$AUDIT_FILE" | jq -r '.atype' | sort | uniq -c
fi
```

#### 4.2 Logging MongoDB

```
☐ Logging MongoDB activé (systemLog configuré)
☐ Verbosity appropriée (0 = info, 1 = debug)
☐ Logs rotationnés quotidiennement
☐ Rétention 30-90 jours minimum
☐ Logs centralisés (rsyslog, Fluentd, etc.)
☐ Monitoring des erreurs critiques
☐ Alertes sur crash/restart
☐ Slow query logging activé
☐ Profiler configuré (niveau 1 ou 2)
```

### 5. Configuration MongoDB

#### 5.1 Paramètres système

```
☐ Transparent Huge Pages (THP) désactivé
☐ NUMA désactivé (si applicable)
☐ ulimits appropriés (nofile: 64000, nproc: 64000)
☐ Swappiness réduit (1-10)
☐ readahead réduit (16 ou 32 pour SSD)
☐ Filesystem XFS recommandé (ou ext4)
☐ atime désactivé sur partition MongoDB (noatime)
☐ Kernel tuning appliqué (tcp_keepalive, somaxconn, etc.)
```

**Script de validation** :

```bash
#!/bin/bash
# validate-system-config.sh

echo "=== System Configuration Validation ==="

# THP
echo -n "Transparent Huge Pages: "
cat /sys/kernel/mm/transparent_hugepage/enabled
if grep -q "\[never\]" /sys/kernel/mm/transparent_hugepage/enabled; then
  echo "✓ THP is disabled"
else
  echo "❌ THP should be disabled"
fi

# ulimits
echo -n "ulimit -n (open files): "
ulimit -n
if [ $(ulimit -n) -ge 64000 ]; then
  echo "✓ ulimit is sufficient"
else
  echo "⚠️  ulimit should be at least 64000"
fi

# Swappiness
echo -n "Swappiness: "
cat /proc/sys/vm/swappiness
if [ $(cat /proc/sys/vm/swappiness) -le 10 ]; then
  echo "✓ Swappiness is optimal"
else
  echo "⚠️  Swappiness should be 1-10"
fi

# readahead
echo -n "readahead (should be 16-32 for SSD): "
blockdev --getra /dev/sda

# Filesystem
echo -n "Filesystem type: "
df -T /data/mongodb | tail -1 | awk '{print $2}'
```

#### 5.2 Configuration Replica Set

```
☐ Replica Set configuré (3+ membres minimum)
☐ Membres dans différents datacenters/AZs
☐ Arbiter évité (ou utilisé avec précaution)
☐ Priority configurée pour contrôler les élections
☐ Votes configurés correctement (nombre impair)
☐ Hidden members pour analytics/backup (si applicable)
☐ Delayed members pour protection erreurs humaines (optionnel)
☐ Heartbeat interval approprié (défaut 2s)
☐ Election timeout approprié (défaut 10s)
☐ Oplog size suffisant (5-10% de la taille des données)
```

#### 5.3 Configuration Sharded Cluster

```
☐ Config servers en Replica Set (3 membres)
☐ Mongos isolés (pas sur mêmes serveurs que shards)
☐ Shard key approprié choisi
☐ Balancer configuré (fenêtre de maintenance si nécessaire)
☐ Chunk size approprié (64 MB par défaut)
☐ Zones configurées (si applicable)
☐ TLS entre tous les composants
☐ Authentification cluster avec keyFile ou x509
☐ Monitoring des migrations de chunks
```

### 6. Backup et Disaster Recovery

```
☐ Backups automatisés configurés
☐ Fréquence appropriée (quotidien minimum pour production)
☐ Méthode de backup testée (mongodump, snapshot, Atlas backup)
☐ Backups chiffrés (GPG, KMS)
☐ Backups stockés dans emplacement séparé (autre datacenter)
☐ Rétention conforme aux exigences (7-90 jours)
☐ Tests de restauration réguliers (mensuel minimum)
☐ RTO/RPO documentés et validés
☐ Procédure de disaster recovery documentée
☐ Backup du Key Vault (CSFLE) séparé des données
☐ Point-in-Time Recovery configuré (si disponible)
```

### 7. Monitoring et Alerting

```
☐ Monitoring activé (MongoDB Ops Manager, Cloud Manager, ou Prometheus)
☐ Métriques système surveillées (CPU, RAM, Disk, Network)
☐ Métriques MongoDB surveillées (connections, opcount, replication lag)
☐ Alertes sur métriques critiques configurées
☐ Alertes sécurité configurées (échecs auth, accès non autorisés)
☐ Dashboard Grafana/Kibana créé
☐ Logs centralisés et indexés
☐ On-call rotation définie
☐ Runbooks pour incidents créés
☐ Tests réguliers des alertes
```

### 8. Conformité et Documentation

```
☐ Politique de sécurité documentée
☐ Procédures d'accès documentées
☐ Architecture réseau documentée
☐ Liste des utilisateurs et rôles maintenue
☐ Inventaire des données sensibles créé
☐ Procédure de gestion des incidents définie
☐ Procédure de patch management définie
☐ Formation sécurité pour équipe réalisée
☐ Audits de sécurité réguliers planifiés (trimestriels)
☐ Tests de pénétration annuels planifiés
```

## Checklist par environnement

### Développement

```
☐ Authentification activée (même en dev)
☐ Utilisateurs dédiés (pas de partage de comptes)
☐ Données de production JAMAIS en développement
☐ Données anonymisées/masquées pour tests
☐ Backup minimal (optionnel)
☐ TLS optionnel (mais recommandé)
☐ Monitoring de base
☐ Logs de debug activés (verbosity: 1)
☐ Accès réseau restreint au réseau de développement
```

### Staging

```
☐ Configuration identique à production
☐ Authentification stricte (SCRAM-SHA-256)
☐ TLS activé et requis
☐ RBAC complet
☐ Données de test réalistes mais anonymisées
☐ Backup quotidien
☐ Monitoring complet
☐ Tests de sécurité réguliers
☐ Audit activé (filtre allégé acceptable)
☐ Réseau isolé (pas d'accès depuis Internet)
```

### Production

```
☐ TOUTES les mesures de sécurité activées
☐ Configuration auditée et validée
☐ Documentation complète et à jour
☐ Monitoring 24/7
☐ Alerting configuré avec on-call
☐ Backups testés mensuellement
☐ Disaster recovery plan validé
☐ Change management process en place
☐ Security hardening complet
☐ Tests de pénétration annuels
☐ Revue de sécurité trimestrielle
☐ Conformité réglementaire validée
```

## Checklist par conformité

### PCI-DSS (Payment Card Industry)

```
☐ Chiffrement en transit (TLS requis)
☐ Chiffrement au repos activé
☐ CSFLE pour numéros de cartes
☐ Audit complet activé (tous accès aux données de cartes)
☐ Rétention logs 1 an minimum (3 mois online)
☐ Authentification forte (MFA pour admins)
☐ Segmentation réseau (cardholder data isolé)
☐ Accès restreint par need-to-know
☐ Revue trimestrielle des accès
☐ Tests de pénétration annuels
☐ Vulnerability scanning trimestriel
☐ IDS/IPS en place
☐ Backup chiffré
☐ Procédure de réponse aux incidents documentée
```

**Script de validation PCI-DSS** :

```bash
#!/bin/bash
# validate-pci-compliance.sh

echo "=== PCI-DSS Compliance Check ==="

# 1. Vérifier chiffrement
echo "1. Encryption checks..."
mongosh --tls --eval "print('✓ TLS is enabled')" 2>/dev/null || echo "❌ TLS is NOT enabled"
mongosh --eval "if(db.serverStatus().security.encryptionAtRest) print('✓ Encryption at Rest is enabled'); else print('❌ Encryption at Rest is NOT enabled');"

# 2. Vérifier audit
echo "2. Audit checks..."
if [ -f /var/log/mongodb/audit.json ]; then
  AGE=$(($(date +%s) - $(stat -c %Y /var/log/mongodb/audit.json)))
  if [ $AGE -lt 3600 ]; then
    echo "✓ Audit is active"
  else
    echo "❌ Audit log is stale"
  fi
else
  echo "❌ Audit is NOT configured"
fi

# 3. Vérifier authentification
echo "3. Authentication checks..."
mongosh --eval "db.adminCommand({ getParameter: 1, authenticationMechanisms: 1 })" | grep -q "SCRAM-SHA-256" && echo "✓ Strong auth enabled" || echo "❌ Strong auth NOT enabled"

# 4. Vérifier segmentation réseau
echo "4. Network checks..."
grep -q "bindIp: 127.0.0.1\|bindIp: 10\." /etc/mongod.conf && echo "✓ bindIp is restricted" || echo "❌ bindIp is NOT restricted"

echo "=== Check complete ==="
```

### HIPAA (Health Insurance Portability and Accountability Act)

```
☐ Chiffrement en transit (TLS requis)
☐ Chiffrement au repos activé
☐ CSFLE pour PHI (Protected Health Information)
☐ Audit complet (tous accès aux PHI)
☐ Rétention logs 6 ans
☐ Authentification unique par utilisateur (pas de comptes partagés)
☐ Timeout de session configuré
☐ Contrôle d'accès granulaire (RBAC)
☐ Logs d'audit non modifiables
☐ Backup chiffré avec rétention 6 ans
☐ Procédure de breach notification
☐ Business Associate Agreement (BAA) signé avec vendors
☐ Risk analysis annuelle
☐ Formation HIPAA pour personnel
```

### SOX (Sarbanes-Oxley Act)

```
☐ Séparation des rôles (développeurs ≠ production admins)
☐ Audit des modifications de données financières
☐ Traçabilité complète des changements
☐ Revue trimestrielle des accès
☐ Rétention logs 7 ans
☐ Change management formel
☐ Backup et recovery testés
☐ Contrôles IT généraux documentés
☐ Tests des contrôles annuels
```

### RGPD (Règlement Général sur la Protection des Données)

```
☐ Inventaire des données personnelles
☐ Base légale pour traitement documentée
☐ Consentement tracé (si applicable)
☐ Droit à l'oubli implémentable
☐ Droit à la portabilité implémentable
☐ Chiffrement des données sensibles
☐ Pseudonymisation/anonymisation quand possible
☐ Limitation de la rétention (durée appropriée)
☐ Sécurité by design et by default
☐ Data Protection Impact Assessment (si applicable)
☐ Registre des traitements maintenu
☐ Procédure de notification de breach (72h)
☐ DPO désigné (si requis)
☐ Transferts hors UE sécurisés (clauses contractuelles)
```

## Scripts d'audit automatisé

### Script d'audit complet

```bash
#!/bin/bash
# /usr/local/bin/mongodb-security-audit.sh
# Audit de sécurité MongoDB complet

OUTPUT_FILE="/var/log/mongodb-security-audit-$(date +%Y%m%d-%H%M%S).log"

exec > >(tee -a "$OUTPUT_FILE") 2>&1

echo "=========================================="
echo "MongoDB Security Audit"
echo "Date: $(date)"
echo "Hostname: $(hostname)"
echo "=========================================="
echo ""

# Fonction pour afficher le résultat
check_result() {
  if [ $1 -eq 0 ]; then
    echo "✓ PASS: $2"
  else
    echo "❌ FAIL: $2"
  fi
}

# 1. AUTHENTICATION
echo "=== 1. AUTHENTICATION ==="

# Vérifier que l'authentification est activée
mongosh --quiet --eval "db.adminCommand({ getParameter: 1, quiet: 1 }).quiet" > /dev/null 2>&1
if grep -q "authorization: enabled" /etc/mongod.conf; then
  check_result 0 "Authentication is enabled"
else
  check_result 1 "Authentication is NOT enabled"
fi

# Vérifier SCRAM-SHA-256
mongosh --quiet --eval "db.adminCommand({ getParameter: 1, authenticationMechanisms: 1 })" 2>/dev/null | grep -q "SCRAM-SHA-256"
check_result $? "SCRAM-SHA-256 is available"

# Compter les utilisateurs avec privilèges root
ROOT_USERS=$(mongosh admin --quiet --eval "db.system.users.countDocuments({ 'roles.role': 'root' })" 2>/dev/null)
if [ "$ROOT_USERS" -le 2 ]; then
  check_result 0 "Root users count is acceptable ($ROOT_USERS)"
else
  check_result 1 "Too many root users ($ROOT_USERS)"
fi

echo ""

# 2. ENCRYPTION
echo "=== 2. ENCRYPTION ==="

# Vérifier TLS
if grep -q "mode: requireTLS" /etc/mongod.conf; then
  check_result 0 "TLS is required"
else
  check_result 1 "TLS is NOT required"
fi

# Vérifier Encryption at Rest
mongosh --quiet --eval "
  var status = db.serverStatus();
  if (status.security && status.security.encryptionAtRest) {
    print('enabled');
  } else {
    print('disabled');
  }
" 2>/dev/null | grep -q "enabled"
check_result $? "Encryption at Rest"

# Vérifier les certificats
CERT_FILE="/etc/ssl/mongodb/server.pem"
if [ -f "$CERT_FILE" ]; then
  EXPIRY=$(openssl x509 -in "$CERT_FILE" -noout -enddate 2>/dev/null | cut -d= -f2)
  EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s 2>/dev/null)
  NOW_EPOCH=$(date +%s)
  DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

  if [ $DAYS_LEFT -gt 30 ]; then
    check_result 0 "Certificate expiry ($DAYS_LEFT days left)"
  else
    check_result 1 "Certificate expires soon ($DAYS_LEFT days left)"
  fi
else
  check_result 1 "Certificate file not found"
fi

echo ""

# 3. NETWORK SECURITY
echo "=== 3. NETWORK SECURITY ==="

# Vérifier bindIp
BIND_IP=$(grep "bindIp:" /etc/mongod.conf | awk '{print $2}')
if [ "$BIND_IP" != "0.0.0.0" ] && [ "$BIND_IP" != "" ]; then
  check_result 0 "bindIp is restricted ($BIND_IP)"
else
  check_result 1 "bindIp is not restricted ($BIND_IP)"
fi

# Vérifier que MongoDB n'écoute pas sur 0.0.0.0
netstat -tuln | grep 27017 | grep -q "0.0.0.0"
if [ $? -ne 0 ]; then
  check_result 0 "MongoDB is not listening on 0.0.0.0"
else
  check_result 1 "MongoDB is listening on 0.0.0.0 (DANGEROUS)"
fi

# Vérifier le firewall
if systemctl is-active --quiet firewalld || systemctl is-active --quiet ufw; then
  check_result 0 "Firewall is active"
else
  check_result 1 "Firewall is NOT active"
fi

# Tester l'exposition publique
PUBLIC_IP=$(curl -s ifconfig.me 2>/dev/null)
if [ ! -z "$PUBLIC_IP" ]; then
  timeout 5 bash -c "echo > /dev/tcp/$PUBLIC_IP/27017" 2>/dev/null
  if [ $? -ne 0 ]; then
    check_result 0 "MongoDB is NOT exposed to Internet"
  else
    check_result 1 "MongoDB IS exposed to Internet (CRITICAL)"
  fi
fi

echo ""

# 4. AUDIT
echo "=== 4. AUDIT ==="

# Vérifier que l'audit est configuré
if grep -q "auditLog:" /etc/mongod.conf; then
  check_result 0 "Audit is configured"

  # Vérifier que le fichier d'audit est récent
  AUDIT_FILE=$(grep "path:" /etc/mongod.conf | grep -A1 "auditLog:" | tail -1 | awk '{print $2}')
  if [ -f "$AUDIT_FILE" ]; then
    LAST_MODIFIED=$(stat -c %Y "$AUDIT_FILE" 2>/dev/null)
    NOW=$(date +%s)
    DIFF=$((NOW - LAST_MODIFIED))

    if [ $DIFF -lt 3600 ]; then
      check_result 0 "Audit log is active (last write ${DIFF}s ago)"
    else
      check_result 1 "Audit log is stale (last write ${DIFF}s ago)"
    fi
  fi
else
  check_result 1 "Audit is NOT configured"
fi

echo ""

# 5. SYSTEM CONFIGURATION
echo "=== 5. SYSTEM CONFIGURATION ==="

# THP
if grep -q "\[never\]" /sys/kernel/mm/transparent_hugepage/enabled 2>/dev/null; then
  check_result 0 "Transparent Huge Pages is disabled"
else
  check_result 1 "Transparent Huge Pages is NOT disabled"
fi

# ulimits
NOFILE=$(ulimit -n)
if [ $NOFILE -ge 64000 ]; then
  check_result 0 "ulimit nofile is sufficient ($NOFILE)"
else
  check_result 1 "ulimit nofile is too low ($NOFILE, should be >= 64000)"
fi

# Swappiness
SWAPPINESS=$(cat /proc/sys/vm/swappiness 2>/dev/null)
if [ $SWAPPINESS -le 10 ]; then
  check_result 0 "Swappiness is optimal ($SWAPPINESS)"
else
  check_result 1 "Swappiness is too high ($SWAPPINESS, should be <= 10)"
fi

echo ""

# 6. BACKUP
echo "=== 6. BACKUP ==="

# Vérifier la présence de backups récents
BACKUP_DIR="/backup/mongodb"
if [ -d "$BACKUP_DIR" ]; then
  LATEST_BACKUP=$(find "$BACKUP_DIR" -type f -name "*.gz" -mtime -1 | head -1)
  if [ ! -z "$LATEST_BACKUP" ]; then
    check_result 0 "Recent backup found (< 24h)"
  else
    check_result 1 "No recent backup found"
  fi
else
  check_result 1 "Backup directory not found"
fi

echo ""

# SUMMARY
echo "=========================================="
echo "Audit complete. Results saved to: $OUTPUT_FILE"
echo "=========================================="

# Compter les échecs
FAILURES=$(grep "❌ FAIL" "$OUTPUT_FILE" | wc -l)
if [ $FAILURES -eq 0 ]; then
  echo "✓ All checks passed!"
  exit 0
else
  echo "⚠️  $FAILURES check(s) failed. Review required."
  exit 1
fi
```

### Automatisation avec cron

```bash
# Audit hebdomadaire
0 2 * * 0 /usr/local/bin/mongodb-security-audit.sh && \
  mail -s "MongoDB Security Audit - $(hostname)" security@company.com < /var/log/mongodb-security-audit-latest.log

# Envoyer alerte si échecs
0 2 * * 0 /usr/local/bin/mongodb-security-audit.sh || \
  echo "ALERT: MongoDB security audit failed" | mail -s "SECURITY ALERT" security@company.com
```

### Intégration CI/CD

```yaml
# .gitlab-ci.yml
mongodb-security-audit:
  stage: security
  script:
    - ./scripts/mongodb-security-audit.sh
  only:
    - schedules
  allow_failure: false
  artifacts:
    reports:
      junit: security-audit-report.xml
    paths:
      - security-audit-*.log
    expire_in: 30 days
```

## Outils de scanning

### MongoDB Security Scanner

```python
#!/usr/bin/env python3
# mongodb-security-scanner.py
# Scanner de sécurité automatisé

import pymongo
import ssl
import socket
import sys
from datetime import datetime

def scan_mongodb(host, port=27017):
    """Scan MongoDB pour problèmes de sécurité"""

    print(f"\n{'='*60}")
    print(f"MongoDB Security Scanner")
    print(f"Target: {host}:{port}")
    print(f"Date: {datetime.now()}")
    print(f"{'='*60}\n")

    issues = []

    # Test 1: Connexion sans authentification
    print("[1] Testing unauthenticated access...")
    try:
        client = pymongo.MongoClient(
            host, port,
            serverSelectionTimeoutMS=5000
        )
        client.admin.command('ping')
        issues.append("CRITICAL: Unauthenticated access allowed!")
        print("    ❌ FAIL: No authentication required")
    except pymongo.errors.OperationFailure:
        print("    ✓ PASS: Authentication required")
    except Exception as e:
        print(f"    ⚠️  WARNING: Could not connect ({e})")

    # Test 2: TLS/SSL
    print("[2] Testing TLS/SSL...")
    try:
        client = pymongo.MongoClient(
            host, port,
            serverSelectionTimeoutMS=5000,
            tls=False
        )
        client.admin.command('ping')
        issues.append("HIGH: TLS not enforced")
        print("    ❌ FAIL: TLS not enforced")
    except:
        print("    ✓ PASS: TLS appears to be enforced")

    # Test 3: Default port
    if port == 27017:
        issues.append("MEDIUM: Using default port (27017)")
        print("[3] ⚠️  WARNING: Using default port")
    else:
        print("[3] ✓ INFO: Using non-default port")

    # Test 4: Banner grabbing
    print("[4] Checking version disclosure...")
    try:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.settimeout(5)
        s.connect((host, port))
        s.close()
        print("    ℹ️  INFO: Port is open")
    except:
        print("    ✓ PASS: Port is filtered/closed")

    # Test 5: Weak credentials (example list)
    print("[5] Testing common credentials...")
    weak_creds = [
        ('admin', 'admin'),
        ('root', 'root'),
        ('mongodb', 'mongodb')
    ]

    for username, password in weak_creds:
        try:
            client = pymongo.MongoClient(
                host, port,
                username=username,
                password=password,
                authSource='admin',
                serverSelectionTimeoutMS=2000
            )
            client.admin.command('ping')
            issues.append(f"CRITICAL: Weak credentials found ({username}:{password})")
            print(f"    ❌ FAIL: Weak credentials work ({username})")
        except:
            pass

    if not any('Weak credentials' in i for i in issues):
        print("    ✓ PASS: No common weak credentials found")

    # Summary
    print(f"\n{'='*60}")
    print("SCAN SUMMARY")
    print(f"{'='*60}")

    if not issues:
        print("✓ No security issues found")
        return 0
    else:
        print(f"⚠️  {len(issues)} issue(s) found:\n")
        for issue in issues:
            print(f"  - {issue}")
        return 1

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("Usage: python3 mongodb-security-scanner.py <host> [port]")
        sys.exit(1)

    host = sys.argv[1]
    port = int(sys.argv[2]) if len(sys.argv) > 2 else 27017

    sys.exit(scan_mongodb(host, port))
```

### Nmap NSE scripts

```bash
# Scanner MongoDB avec nmap
nmap -p 27017 --script mongodb-info,mongodb-databases <target>

# Scanner vulnérabilités connues
nmap -p 27017 --script vuln <target>

# Scanner configuration
nmap -p 27017 --script mongodb-brute <target>
```

## Matrice de risques

| Risque | Probabilité | Impact | Criticité | Mitigation |
|--------|-------------|--------|-----------|------------|
| **Pas d'authentification** | Faible (si configuré) | Critique | **CRITIQUE** | Activer `security.authorization: enabled` |
| **Mot de passe faible** | Moyenne | Élevé | **ÉLEVÉ** | Politique de mots de passe forts, rotation |
| **TLS désactivé** | Faible | Élevé | **ÉLEVÉ** | Activer `net.tls.mode: requireTLS` |
| **MongoDB exposé à Internet** | Moyenne | Critique | **CRITIQUE** | Firewall, bindIp restrictif, VPC |
| **Pas de chiffrement au repos** | Faible | Élevé | **ÉLEVÉ** | Activer Encryption at Rest (Enterprise) |
| **Pas d'audit** | Moyenne | Moyen | **MOYEN** | Activer audit logging (Enterprise) |
| **Certificats expirés** | Moyenne | Élevé | **ÉLEVÉ** | Monitoring, rotation automatisée |
| **Backup manquant/ancien** | Moyenne | Critique | **CRITIQUE** | Automatiser backups, tester restauration |
| **Pas de monitoring** | Moyenne | Moyen | **MOYEN** | Déployer monitoring (Ops Manager, Prometheus) |
| **Privilèges excessifs** | Élevée | Moyen | **MOYEN** | Revue RBAC trimestrielle, principe moindre privilège |
| **Patches manquants** | Moyenne | Élevé | **ÉLEVÉ** | Patch management process, tests en staging |
| **Logs non protégés** | Faible | Moyen | **FAIBLE** | Permissions restrictives, archivage sécurisé |

## Plan de remediation

### Criticité CRITIQUE (à corriger immédiatement)

```
1. Authentification manquante
   Impact: Accès total sans credentials
   Action:
   ├─ Activer authorization dans mongod.conf
   ├─ Créer utilisateur admin
   ├─ Redémarrer MongoDB
   └─ Tester l'accès
   Timeline: < 1 heure

2. MongoDB exposé à Internet
   Impact: Attaque directe possible
   Action:
   ├─ Vérifier bindIp (pas 0.0.0.0)
   ├─ Activer firewall
   ├─ Configurer IP whitelist
   └─ Tester depuis Internet
   Timeline: < 2 heures

3. Backup manquant
   Impact: Perte de données en cas d'incident
   Action:
   ├─ Configurer backup automatique
   ├─ Tester restauration
   └─ Documenter procédure
   Timeline: < 4 heures
```

### Criticité ÉLEVÉE (à corriger sous 1 semaine)

```
1. TLS désactivé
   Impact: Données en clair sur réseau
   Action:
   ├─ Générer certificats
   ├─ Configurer TLS
   ├─ Tester connexions
   └─ Déployer progressivement
   Timeline: 2-5 jours

2. Pas de chiffrement au repos
   Impact: Données lisibles sur disque
   Action:
   ├─ Configurer KMS
   ├─ Activer encryption
   ├─ Valider chiffrement
   └─ Documenter procédure
   Timeline: 3-7 jours
```

### Criticité MOYENNE (à corriger sous 1 mois)

```
1. Audit désactivé
   Action: Activer et configurer audit logging
   Timeline: 1-2 semaines

2. Monitoring insuffisant
   Action: Déployer solution de monitoring complète
   Timeline: 2-4 semaines

3. Documentation manquante
   Action: Documenter architecture et procédures
   Timeline: 2-4 semaines
```

## Timeline de déploiement sécurisé

### Phase 1 : Fondations (Semaine 1-2)

```
Jour 1-2 : Configuration de base
├─ Activer authentification
├─ Créer utilisateurs avec RBAC
├─ Configurer bindIp
└─ Activer firewall

Jour 3-5 : Chiffrement
├─ Générer certificats TLS
├─ Activer TLS
├─ Tester connexions
└─ Documenter

Jour 6-10 : Backup
├─ Configurer backup automatique
├─ Tester restauration
├─ Documenter RTO/RPO
└─ Planifier rétention
```

### Phase 2 : Renforcement (Semaine 3-4)

```
Jour 11-15 : Network security
├─ Configurer VPC/Subnets
├─ Security Groups
├─ Bastion host
└─ Tests de pénétration internes

Jour 16-20 : Encryption at Rest
├─ Configurer KMS
├─ Activer encryption
├─ Valider
└─ Rotation planning
```

### Phase 3 : Observabilité (Semaine 5-6)

```
Jour 21-25 : Monitoring
├─ Déployer Prometheus/Grafana
├─ Configurer alertes
├─ Dashboards
└─ Tests

Jour 26-30 : Audit & Logging
├─ Activer audit (Enterprise)
├─ Configurer SIEM
├─ Alertes sécurité
└─ Tests
```

### Phase 4 : Conformité (Semaine 7-8)

```
Jour 31-40 : Documentation & Conformité
├─ Documenter architecture
├─ Procédures opérationnelles
├─ Runbooks
├─ Audit de conformité
├─ Tests finaux
└─ Formation équipe
```

## Conclusion

Cette checklist est un outil vivant qui doit être :
- **Révisée régulièrement** (trimestriellement minimum)
- **Adaptée à votre contexte** (conformité, risques spécifiques)
- **Automatisée quand possible** (scripts, CI/CD)
- **Intégrée au processus** (onboarding, changements, audits)

**Ressources complémentaires** :
- [MongoDB Security Checklist officielle](https://docs.mongodb.com/manual/administration/security-checklist/)
- [CIS MongoDB Benchmark](https://www.cisecurity.org/benchmark/mongodb)
- [OWASP Database Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html)
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)

**Prochaines étapes recommandées** :
1. Exécuter l'audit automatisé
2. Prioriser les remediations selon criticité
3. Planifier le déploiement progressif
4. Former l'équipe
5. Établir un cycle de revue régulier

La sécurité est un processus continu, pas un état final. Restez vigilants ! 🔒

⏭️ [Conformité et certifications](/11-securite/09-conformite-certifications.md)
