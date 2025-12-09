🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 5 : Sécurité et Administration (Avancé)

## 🎯 Opérations et sécurité en production

Vous maîtrisez maintenant la modélisation, les transactions et l'architecture distribuée de MongoDB. Vos applications fonctionnent, vos données sont répliquées, et votre cluster peut scaler. Mais une question cruciale se pose : **comment sécuriser vos données contre les menaces, garantir leur disponibilité continue, et opérer un système MongoDB en production 24/7 avec confiance ?**

La Partie 5 est dédiée aux **opérations critiques en production** : sécurité, sauvegarde/restauration, et monitoring. Ce sont les compétences qui différencient un système MongoDB qui "fonctionne" d'un système MongoDB qui est **production-ready, sécurisé et résilient**.

## 🔒 La sécurité n'est pas optionnelle

### Le coût des violations de sécurité

Les violations de données sont une réalité et leurs conséquences sont dévastatrices :

**Impact financier direct :**
- Amendes réglementaires (RGPD : jusqu'à 4% du CA mondial ou 20M€)
- Coûts de notification et réponse aux incidents
- Pertes de revenus pendant l'incident
- **Coût moyen d'une violation (2023) : 4.45M$ par incident**

**Impact business long terme :**
- Perte de confiance des clients (churn)
- Dommages à la réputation (difficiles à quantifier mais réels)
- Perte de partenaires commerciaux
- Impact sur la valorisation de l'entreprise

**Impact légal et réglementaire :**
- Poursuites judiciaires (class actions)
- Investigations réglementaires
- Perte de certifications (ISO 27001, SOC 2, etc.)
- Restrictions d'activité

**Exemples réels (bases de données non sécurisées) :**
- MongoDB exposés sans authentification : Des millions de documents exfiltrés
- Ransomware ciblant les bases de données : Perte de données ou rançons
- Injection NoSQL : Contournement de l'authentification

> **Réalité brutale** : Une base MongoDB non sécurisée est découverte et compromise en quelques heures après son exposition sur Internet.

### Le principe de défense en profondeur

La sécurité n'est pas un point de contrôle unique, c'est une **série de couches défensives** :

```
┌────────────────────────────────────────────────────────┐
│ Couche 1 : Réseau (Firewall, VPN, IP Whitelisting)     │
├────────────────────────────────────────────────────────┤
│ Couche 2 : Authentification (Qui êtes-vous ?)          │
├────────────────────────────────────────────────────────┤
│ Couche 3 : Autorisation (Que pouvez-vous faire ?)      │
├────────────────────────────────────────────────────────┤
│ Couche 4 : Chiffrement en transit (TLS/SSL)            │
├────────────────────────────────────────────────────────┤
│ Couche 5 : Chiffrement au repos (Storage encryption)   │
├────────────────────────────────────────────────────────┤
│ Couche 6 : Audit et monitoring (Détection)             │
├────────────────────────────────────────────────────────┤
│ Couche 7 : Validation des données (Intégrité)          │
├────────────────────────────────────────────────────────┤
│ Couche 8 : Sauvegardes (Récupération)                  │
└────────────────────────────────────────────────────────┘
```

**Principe fondamental** : Si une couche échoue, les autres doivent encore protéger vos données.

### Conformité réglementaire

Les bases de données doivent souvent se conformer à des régulations strictes :

**Régulations majeures :**
- **RGPD** (EU) : Protection des données personnelles
- **HIPAA** (US) : Données de santé
- **PCI DSS** : Données de cartes de crédit
- **SOX** : Données financières
- **SOC 2** : Contrôles organisationnels

**Exigences communes :**
- Chiffrement des données sensibles (transit et repos)
- Contrôle d'accès granulaire (principe du moindre privilège)
- Audit trail complet (qui a fait quoi, quand)
- Sauvegarde et capacité de restauration
- Gestion des identités et accès
- Tests de sécurité réguliers

**Impact pour MongoDB :**
- Configuration de l'authentification forte
- Mise en place de rôles granulaires
- Activation de l'audit logging
- Chiffrement TLS obligatoire
- Chiffrement des données au repos
- Sauvegarde chiffrée et testée

## 🛡️ Les trois piliers opérationnels

### 1. Sécurité : Protéger contre les menaces

**Objectif** : Garantir que seules les personnes/systèmes autorisés accèdent aux données, et que ces données sont protégées contre l'exfiltration ou la modification.

**Composants clés :**
- Authentification multi-facteur
- Autorisation basée sur les rôles (RBAC)
- Chiffrement de bout en bout
- Segmentation réseau
- Audit et détection des anomalies

**Menaces à contrer :**
- ❌ Accès non autorisés (credential theft, brute force)
- ❌ Injection NoSQL
- ❌ Privilege escalation
- ❌ Data exfiltration
- ❌ Ransomware
- ❌ Insider threats (menaces internes)

---

### 2. Sauvegarde/Restauration : Résilience face aux catastrophes

**Objectif** : Garantir que les données peuvent être restaurées après une perte, corruption, erreur humaine ou attaque.

**Principe du 3-2-1 :**
- **3** copies de vos données
- Sur **2** types de media différents
- **1** copie hors site (off-site)

**Types de sinistres à gérer :**
- 🔥 **Panne matérielle** : Disque dur, RAID, serveur entier
- 🔥 **Corruption de données** : Bug logiciel, corruption filesystem
- 🔥 **Erreur humaine** : DROP collection/database accidentel
- 🔥 **Ransomware** : Chiffrement malveillant des données
- 🔥 **Catastrophe naturelle** : Incendie, inondation, tremblement de terre
- 🔥 **Cyberattaque** : Effacement intentionnel des données

**Métriques critiques :**
- **RPO (Recovery Point Objective)** : Combien de données pouvez-vous vous permettre de perdre ?
  - RPO = 0 : Aucune perte (réplication synchrone)
  - RPO = 1 heure : Perte max d'1h de données
  - RPO = 24 heures : Backup quotidien
  
- **RTO (Recovery Time Objective)** : Combien de temps pour restaurer ?
  - RTO = 5 minutes : Failover automatique
  - RTO = 1 heure : Restauration manuelle rapide
  - RTO = 8 heures : Restauration complexe

**Stratégies de backup :**
- **Continuous backup** (oplog-based) : RPO quasi-nul
- **Snapshot** (point-in-time) : RPO = fréquence des snapshots
- **Export logique** (mongodump) : RPO = fréquence des dumps

---

### 3. Monitoring : Visibilité et proactivité

**Objectif** : Détecter les problèmes avant qu'ils ne deviennent critiques, et diagnostiquer rapidement quand ils surviennent.

**Philosophie du monitoring :**
> "You can't manage what you don't measure." - Peter Drucker

> "Hope is not a strategy." - Proverbe DevOps

**Monitoring proactif vs réactif :**

**Réactif** (à éviter) :
```
❌ Incident → Clients se plaignent → Investigation → Résolution
   Impact : Downtime, mécontentement, perte de revenus
```

**Proactif** (objectif) :
```
✅ Alerte → Investigation → Résolution préventive → Aucun impact client
   Impact : Disponibilité maintenue, confiance préservée
```

**Métriques essentielles :**
- **Disponibilité** : Uptime, member states, élections
- **Performance** : Latence des requêtes, throughput, queue lengths
- **Ressources** : CPU, RAM, disque (IOPS, space), réseau
- **Réplication** : Replication lag, oplog window
- **Sécurité** : Tentatives d'authentification, anomalies d'accès
- **Intégrité** : Erreurs, crashes, corruptions

**Niveaux d'alerting :**
- 🟢 **Info** : Événements normaux (rolling restart)
- 🟡 **Warning** : Attention requise (replication lag > 10s)
- 🟠 **Error** : Problème sérieux (secondary down)
- 🔴 **Critical** : Urgence immédiate (primary down, disk full)

## 📋 Prérequis

Cette partie s'adresse à des **administrateurs système, DBA et SRE** ayant :

### Connaissances MongoDB requises
- ✅ **Maîtrise complète des Parties 1-4**
- ✅ Architecture distribuée (Replica Sets, Sharding)
- ✅ Transactions et cohérence
- ✅ Expérience de déploiement et configuration

### Compétences en administration système
- ✅ **Linux/Unix administration** : Avancé (fichiers, processus, services, logs)
- ✅ **Réseau** : Firewalls, VPN, DNS, load balancers, proxies
- ✅ **Sécurité système** : SSH keys, certificats SSL/TLS, PKI
- ✅ **Scripting** : Bash, Python pour l'automatisation
- ✅ **Gestion de stockage** : Filesystems, RAID, snapshots, backup
- ✅ **Monitoring** : Prometheus, Grafana, ELK, ou équivalents

### Connaissances en sécurité
- 🔐 Principes de sécurité (CIA : Confidentiality, Integrity, Availability)
- 🔐 Authentification et autorisation (RBAC, LDAP, Kerberos)
- 🔐 Cryptographie de base (symétrique, asymétrique, hashing)
- 🔐 TLS/SSL (certificats, CA, handshake)
- 🔐 Audit et compliance

### Expérience opérationnelle
- 🛠️ Gestion d'incidents en production
- 🛠️ On-call / astreintes
- 🛠️ Procédures de changement (change management)
- 🛠️ Postmortem et amélioration continue

### État d'esprit
- 🧠 **Paranoïa constructive** : Anticiper les pires scénarios
- 🧠 **Rigueur** : Zéro tolérance pour les raccourcis en sécurité
- 🧠 **Proactivité** : Prévenir plutôt que réagir
- 🧠 **Documentation obsessive** : Tout doit être documenté
- 🧠 **Culture blameless** : Apprendre des erreurs sans blâmer

**Si vous ne maîtrisez pas ces prérequis**, cette partie sera difficile. Prenez le temps de vous former sur les bases de l'administration système et de la sécurité.

## 🎓 Objectifs d'apprentissage

À la fin de cette partie, vous serez capable de :

### Compétences en sécurité

**Authentification et autorisation :**
- ✅ **Configurer** l'authentification SCRAM-SHA-256 (défaut)
- ✅ **Déployer** l'authentification x.509 avec certificats
- ✅ **Intégrer** LDAP pour l'authentification centralisée
- ✅ **Configurer** Kerberos pour les environnements entreprise
- ✅ **Créer** et gérer les utilisateurs et rôles
- ✅ **Appliquer** le principe du moindre privilège
- ✅ **Définir** des rôles personnalisés granulaires

**Chiffrement :**
- ✅ **Activer** TLS/SSL pour le chiffrement en transit
- ✅ **Gérer** les certificats (génération, renouvellement)
- ✅ **Configurer** le chiffrement au repos (storage encryption)
- ✅ **Implémenter** Client-Side Field Level Encryption (CSFLE)
- ✅ **Utiliser** Queryable Encryption pour les données sensibles
- ✅ **Gérer** les clés de chiffrement (rotation, KMS)

**Sécurité réseau :**
- ✅ **Configurer** les firewalls et IP whitelisting
- ✅ **Segmenter** le réseau (VLANs, subnets)
- ✅ **Utiliser** les VPN pour l'accès distant
- ✅ **Configurer** les bind IPs correctement
- ✅ **Sécuriser** les communications inter-nœuds

**Audit et compliance :**
- ✅ **Activer** l'audit logging
- ✅ **Configurer** les événements à auditer
- ✅ **Analyser** les logs d'audit
- ✅ **Implémenter** des alertes de sécurité
- ✅ **Préparer** les rapports de compliance

**Sécurité opérationnelle :**
- ✅ **Effectuer** des security assessments
- ✅ **Appliquer** les patches de sécurité
- ✅ **Gérer** les vulnérabilités
- ✅ **Répondre** aux incidents de sécurité
- ✅ **Former** les équipes aux bonnes pratiques

### Compétences en sauvegarde/restauration

**Stratégies et planification :**
- ✅ **Définir** RPO et RTO adaptés à l'activité
- ✅ **Concevoir** une stratégie de backup complète
- ✅ **Implémenter** la règle 3-2-1
- ✅ **Planifier** la fréquence et rétention des backups
- ✅ **Chiffrer** les backups

**Méthodes de backup :**
- ✅ **Utiliser** mongodump/mongorestore (logical backup)
- ✅ **Créer** des snapshots filesystem
- ✅ **Configurer** les backups de Replica Sets
- ✅ **Sauvegarder** des clusters shardés (complexe)
- ✅ **Utiliser** MongoDB Atlas Backup (cloud)
- ✅ **Implémenter** l'oplog-based backup

**Restauration :**
- ✅ **Restaurer** une collection ou une base spécifique
- ✅ **Effectuer** un Point-in-Time Recovery (PITR)
- ✅ **Restaurer** un Replica Set complet
- ✅ **Restaurer** un cluster shardé
- ✅ **Gérer** les dépendances et l'ordre de restauration
- ✅ **Valider** l'intégrité après restauration

**Tests et automatisation :**
- ✅ **Tester** régulièrement les restaurations (drill)
- ✅ **Automatiser** les backups (cron, orchestrateurs)
- ✅ **Monitorer** le succès/échec des backups
- ✅ **Alerter** en cas d'échec de backup
- ✅ **Documenter** les procédures de recovery

### Compétences en monitoring et administration

**Monitoring proactif :**
- ✅ **Identifier** les métriques critiques à surveiller
- ✅ **Configurer** Prometheus + Grafana pour MongoDB
- ✅ **Créer** des dashboards opérationnels
- ✅ **Définir** des seuils d'alerte intelligents
- ✅ **Réduire** le bruit (false positives)
- ✅ **Intégrer** avec PagerDuty, OpsGenie, etc.

**Outils MongoDB :**
- ✅ **Maîtriser** serverStatus, dbStats, collStats
- ✅ **Analyser** currentOp pour les requêtes en cours
- ✅ **Utiliser** le profiler pour les slow queries
- ✅ **Interpréter** les logs MongoDB
- ✅ **Exploiter** mongostat et mongotop
- ✅ **Utiliser** MongoDB Ops Manager / Cloud Manager

**Diagnostics et troubleshooting :**
- ✅ **Diagnostiquer** les problèmes de performance
- ✅ **Identifier** les goulots d'étranglement
- ✅ **Résoudre** les problèmes de réplication
- ✅ **Gérer** les problèmes de connexion
- ✅ **Analyser** les crashes et corruptions
- ✅ **Utiliser** FTDC (Full-Time Diagnostic Data Capture)

**Administration quotidienne :**
- ✅ **Effectuer** les maintenances préventives
- ✅ **Gérer** les upgrades de version
- ✅ **Optimiser** la configuration (WiredTiger, etc.)
- ✅ **Gérer** la croissance du stockage
- ✅ **Nettoyer** les anciennes données (TTL, archivage)
- ✅ **Documenter** les changements et incidents

**Gestion de la mémoire et du cache :**
- ✅ **Comprendre** l'architecture WiredTiger
- ✅ **Dimensionner** le cache correctement
- ✅ **Monitorer** l'utilisation de la RAM
- ✅ **Optimiser** pour le working set
- ✅ **Gérer** les situations de memory pressure

## 📚 Vue d'ensemble des modules

Cette partie contient **3 modules interdépendants** qui forment le triptyque opérationnel :

### Module 11 : Sécurité
**Durée estimée : 18-22 heures**

La sécurité de bout en bout de MongoDB.

#### 11.1 Vue d'ensemble de la sécurité MongoDB
**Durée : 1-2 heures**

Introduction au modèle de sécurité de MongoDB.

**Ce que vous maîtriserez :**
- Les vecteurs d'attaque sur MongoDB
- Le modèle de sécurité en couches
- La security checklist officielle MongoDB
- Les certifications et compliance

**Principe clé :** MongoDB secure by default depuis la version 3.6, mais nécessite une configuration appropriée.

---

#### 11.2 Authentification
**Durée : 4-5 heures**

Vérifier l'identité des clients.

**Mécanismes supportés :**

**SCRAM-SHA-256** (défaut) :
- Challenge-response sans envoyer le mot de passe
- Hash salé et itéré
- Résistant aux attaques par replay
- Recommandé pour la plupart des déploiements

**x.509** (certificats) :
- Authentification mutuelle (client et serveur)
- Pas de mots de passe à gérer
- Idéal pour les communications machine-to-machine
- Requiert une PKI (Public Key Infrastructure)

**LDAP** (centralisé) :
- Intégration avec Active Directory
- Gestion centralisée des utilisateurs
- Single Sign-On (SSO) possible
- Requiert MongoDB Enterprise

**Kerberos** :
- Authentification forte entreprise
- Intégration avec l'infrastructure existante
- Complexe à configurer
- Requiert MongoDB Enterprise

**Choix selon le contexte :**
```
Startup/PME : SCRAM-SHA-256
Services distribués : x.509
Grande entreprise : LDAP ou Kerberos
Cloud public : SCRAM-SHA-256 + MFA
```

---

#### 11.3 Autorisation et rôles
**Durée : 4-5 heures**

Contrôler ce que les utilisateurs peuvent faire (RBAC).

**Ce que vous maîtriserez :**

**Rôles intégrés :**
- `read` : Lecture seule sur une base
- `readWrite` : Lecture/écriture sur une base
- `dbAdmin` : Administration d'une base (pas les données)
- `userAdmin` : Gestion des utilisateurs
- `clusterAdmin` : Administration du cluster
- `root` : Super-admin (tous les privilèges)

**Principe du moindre privilège :**
```javascript
// ❌ MAUVAIS : Donner root à tout le monde
db.createUser({ user: "app", pwd: "...", roles: ["root"] })

// ✅ BON : Rôle minimal nécessaire
db.createUser({
  user: "app",
  pwd: "...",
  roles: [
    { role: "readWrite", db: "production" },
    { role: "read", db: "analytics" }
  ]
})
```

**Rôles personnalisés :**
- Définition granulaire des privilèges
- Par collection si nécessaire
- Actions spécifiques (find, insert, update, delete, etc.)
- Héritage de rôles

**Cas d'usage typiques :**
- Application web : `readWrite` sur sa base
- Service analytics : `read` sur toutes les bases
- DBA : `dbAdmin` + `clusterAdmin`
- Backup service : Privilèges de lecture + backup
- Utilisateur métier : `read` sur collections spécifiques

---

#### 11.4 Gestion des utilisateurs
**Durée : 2 heures**

Opérations quotidiennes sur les utilisateurs.

**Ce que vous maîtriserez :**
- Création, modification, suppression d'utilisateurs
- Rotation des mots de passe
- Gestion des rôles
- Audit des permissions
- Désactivation temporaire d'accès

**Bonnes pratiques :**
- Pas d'utilisateur "admin" partagé
- Un utilisateur par application/service
- Rotation régulière des credentials
- Suppression immédiate des comptes inutilisés
- Audit trail complet

---

#### 11.5 Chiffrement
**Durée : 5-6 heures**

Protection cryptographique des données.

**Chiffrement en transit (TLS/SSL) :**
- Chiffrement de toutes les communications réseau
- Authentification mutuelle avec certificats
- Prévention du MITM (Man-in-the-Middle)
- **OBLIGATOIRE en production**

**Configuration :**
```yaml
# mongod.conf
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem
```

**Chiffrement au repos (Storage Encryption) :**
- Chiffrement des fichiers de données sur disque
- Protection si les disques sont volés
- Transparent pour les applications
- MongoDB Enterprise ou Atlas

**Client-Side Field Level Encryption (CSFLE) :**
- Chiffrement des champs sensibles côté application
- Les données sont chiffrées avant d'atteindre MongoDB
- MongoDB ne voit jamais les données en clair
- Clés gérées par l'application ou KMS (AWS, Azure, GCP)

**Queryable Encryption :**
- Évolution de CSFLE
- Permet de faire des requêtes d'égalité sur des champs chiffrés
- MongoDB 6.0+
- Idéal pour les données ultra-sensibles (SSN, numéros de carte, etc.)

**Gestion des clés :**
- Rotation régulière (ex: tous les 90 jours)
- Sauvegarde sécurisée des clés
- Séparation des clés de données et de sauvegarde
- Utilisation de KMS (Key Management Service) en production

---

#### 11.6 Audit
**Durée : 2-3 heures**

Traçabilité des opérations pour la sécurité et la compliance.

**Ce que vous maîtriserez :**
- Activation de l'audit logging (Enterprise/Atlas)
- Configuration des événements à auditer
- Filtrage des événements (trop de logs = bruit)
- Analyse des logs d'audit
- Intégration avec SIEM (Splunk, ELK, etc.)

**Événements à auditer :**
- Authentification (succès et échecs)
- Opérations d'administration (createUser, dropDatabase, etc.)
- Opérations DDL (createCollection, createIndex, etc.)
- Accès aux données sensibles (optionnel, très verbeux)

**Format d'audit :**
```json
{
  "atype" : "authenticate",
  "ts" : { "$date": "2024-12-09T..." },
  "local" : { "ip": "127.0.0.1", "port": 27017 },
  "remote" : { "ip": "192.168.1.100", "port": 51789 },
  "users" : [ { "user": "appUser", "db": "admin" } ],
  "result" : 0  // 0 = succès
}
```

**Compliance :**
- GDPR : Audit des accès aux données personnelles
- HIPAA : Audit complet pour les données de santé
- SOX : Audit des modifications de données financières

---

#### 11.7 Network Security et IP Whitelisting
**Durée : 1-2 heures**

Contrôle de l'accès réseau.

**Ce que vous maîtriserez :**
- Configuration de bind_ip (limiter les interfaces d'écoute)
- IP whitelisting (autoriser uniquement des IPs connues)
- Firewalls (iptables, AWS Security Groups, etc.)
- VPN pour l'accès distant
- Bastion hosts / Jump servers

**Défense en profondeur réseau :**
```
Internet
   ↓
Firewall (WAF)
   ↓
Load Balancer (VIP whitelisting)
   ↓
Application Servers (dans subnet privé)
   ↓
MongoDB (dans subnet privé isolé, bind_ip limité)
```

---

#### 11.8-11.9 Security Checklist et Conformité
**Durée : 2 heures**

Validation et maintenance de la posture de sécurité.

**Security Checklist MongoDB :**
- [ ] Authentification activée
- [ ] Autorisation basée sur les rôles
- [ ] TLS/SSL activé
- [ ] bind_ip configuré (pas 0.0.0.0 en production)
- [ ] Firewall configuré
- [ ] Audit logging activé
- [ ] Chiffrement au repos activé
- [ ] Rotation régulière des credentials
- [ ] Patches de sécurité appliqués
- [ ] Backups chiffrés et testés

**Tests de sécurité :**
- Scan de vulnérabilités (Nessus, OpenVAS)
- Penetration testing
- Code review des configurations
- Audit régulier des accès

---

**Pourquoi ce module est critique :** Une violation de données peut détruire une entreprise. La sécurité n'est pas optionnelle.

---

### Module 12 : Sauvegarde et Restauration
**Durée estimée : 14-18 heures**

Garantir la résilience des données.

#### 12.1 Stratégies de sauvegarde
**Durée : 2-3 heures**

Conception d'une stratégie de backup adaptée.

**Ce que vous maîtriserez :**
- Définition de RPO et RTO
- Règle 3-2-1
- Full vs Incremental vs Differential backups
- Hot backup vs Cold backup
- Backup scheduling (fréquence, fenêtres)

**Matrice de décision :**
```
Application critique (banque) :
  RPO = 0 (pas de perte)
  RTO = 5 min
  → Réplication + Continuous backup

Application standard (e-commerce) :
  RPO = 1 heure
  RTO = 30 minutes
  → Snapshots horaires + Replica Set

Application non-critique (blog) :
  RPO = 24 heures
  RTO = 2 heures
  → Backup quotidien
```

---

#### 12.2 mongodump et mongorestore
**Durée : 2-3 heures**

Backup logique avec les outils officiels.

**Ce que vous maîtriserez :**
- Utilisation de mongodump (export BSON)
- Options de mongodump (--db, --collection, --query, --gzip)
- mongorestore (import)
- Avantages et limitations
- Performance et impact sur le système

**Avantages :**
- ✅ Portable (peut restaurer sur différentes versions)
- ✅ Granulaire (backup par collection)
- ✅ Peut filtrer avec --query

**Limitations :**
- ❌ Lent sur de gros volumes (gigabytes/heure)
- ❌ Impact sur les performances (lecture intensive)
- ❌ Pas de PITR natif

**Cas d'usage :**
- Petites bases (< 100 GB)
- Backup sélectif (certaines collections)
- Migration entre versions
- Export pour archivage

---

#### 12.3-12.4 Backup de Replica Sets et Clusters Shardés
**Durée : 4-5 heures**

Backup cohérent d'architectures distribuées.

**Replica Set :**
- Backup depuis un Secondary (pas d'impact sur le Primary)
- Arrêt temporaire du secondary (cold backup) ou snapshot
- Considérations de cohérence

**Cluster Shardé :**
- **Beaucoup plus complexe**
- Backup de tous les shards + config servers
- Cohérence entre les shards (point-in-time identique)
- Procédure d'arrêt du balancer
- Utilisation d'Atlas Backup recommandée

**Processus pour cluster shardé :**
```
1. Arrêter le balancer (sh.stopBalancer())
2. Flush toutes les écritures (fsync)
3. Backup de tous les shards simultanément
4. Backup des config servers
5. Redémarrer le balancer
```

**Complexité :** Le backup d'un cluster shardé à la main est **error-prone**. Privilégiez MongoDB Atlas Backup ou Ops Manager.

---

#### 12.5 Snapshots du système de fichiers
**Durée : 2-3 heures**

Backup au niveau du filesystem.

**Ce que vous maîtriserez :**
- LVM snapshots (Linux)
- EBS snapshots (AWS)
- Snapshots cloud (Azure, GCP)
- Cohérence des snapshots (fsync + lock)
- Performance et rapidité

**Avantages :**
- ✅ Très rapide (copy-on-write)
- ✅ Minimal downtime
- ✅ Idéal pour gros volumes

**Prérequis :**
- Arrêt temporaire des écritures (db.fsyncLock())
- Vérification de la cohérence après snapshot

---

#### 12.6-12.7 Atlas Backup et Point-in-Time Recovery
**Durée : 3-4 heures**

Solutions cloud et PITR.

**MongoDB Atlas Backup :**
- Continuous backup basé sur l'oplog
- RPO quasi-nul (quelques secondes)
- PITR : Restauration à n'importe quel point des dernières 24h
- Complètement managé (pas de gestion manuelle)
- Snapshots quotidiens conservés (selon rétention)

**Point-in-Time Recovery (PITR) :**
- Restauration à un timestamp précis
- Essentiel pour récupérer d'erreurs humaines
- Utilise l'oplog pour "rejouer" jusqu'au point désiré

**Exemple :**
```
10:00 : Snapshot quotidien
14:30 : DELETE accidentel de données
14:35 : Détection de l'erreur

Action : PITR à 14:29 (juste avant le DELETE)
```

---

#### 12.8-12.11 Oplog, Restauration, Automatisation, Tests
**Durée : 4-5 heures**

Opérations avancées et bonnes pratiques.

**Ce que vous maîtriserez :**
- Utilisation de l'oplog pour le backup continu
- Restauration complète vs partielle
- Automatisation avec cron, Ansible, etc.
- **Tests réguliers de restauration (crucial !)**

**Principe d'or :**
> "Un backup non testé est un backup inexistant."

**Planification des drills :**
- Drill de restauration mensuel minimum
- Drill de disaster recovery trimestriel
- Documentation des procédures
- Mesure des RTO réels (vs théoriques)

---

#### 12.12 Bonnes pratiques
**Durée : 1-2 heures**

Synthèse et recommandations.

**Checklist de backup :**
- [ ] Stratégie documentée (RPO, RTO, fréquence)
- [ ] Backups automatisés
- [ ] Backups chiffrés
- [ ] Backups stockés off-site
- [ ] Règle 3-2-1 respectée
- [ ] Monitoring du succès des backups
- [ ] Alertes en cas d'échec
- [ ] Tests de restauration réguliers
- [ ] Documentation à jour
- [ ] Runbooks pour les restaurations

---

**Pourquoi ce module est critique :** Les données sont votre actif le plus précieux. Une perte de données peut être irréversible.

---

### Module 13 : Monitoring et Administration
**Durée estimée : 16-20 heures**

Visibilité opérationnelle et gestion quotidienne.

#### 13.1 Métriques clés à surveiller
**Durée : 2-3 heures**

Identifier ce qui compte vraiment.

**Ce que vous maîtriserez :**
- Métriques de disponibilité
- Métriques de performance
- Métriques de ressources
- Métriques de réplication
- Métriques de sécurité

**Les 4 Golden Signals (Google SRE) :**
1. **Latency** : Temps de réponse des requêtes
2. **Traffic** : Nombre de requêtes par seconde
3. **Errors** : Taux d'erreurs
4. **Saturation** : Utilisation des ressources (CPU, RAM, disque)

**Métriques MongoDB spécifiques :**
```
Disponibilité :
- Member state (PRIMARY, SECONDARY, DOWN)
- Elections (nombre, durée)
- Heartbeat failures

Performance :
- Query execution time (p50, p95, p99)
- Throughput (ops/sec)
- Queue lengths (read, write)
- Lock contention

Ressources :
- CPU utilization
- RAM (resident, virtual, mapped)
- Disk (IOPS, latency, space)
- Network (bytes in/out)

Réplication :
- Replication lag (secondes)
- Oplog window (heures)
- Oplog GB/hour rate

Sharding (si applicable) :
- Chunk distribution
- Balancer activity
- Jumbo chunks
```

---

#### 13.2 Commandes d'administration
**Durée : 3-4 heures**

Outils en ligne de commande pour diagnostiquer.

**Ce que vous maîtriserez :**

**serverStatus** : Vue d'ensemble du serveur
```javascript
db.serverStatus()
// Retourne : connexions, opérations, mémoire, réplication, etc.
```

**dbStats** : Statistiques d'une base
```javascript
db.stats()
// Retourne : taille, nombre de collections, indexes, etc.
```

**collStats** : Statistiques d'une collection
```javascript
db.collection.stats()
// Retourne : taille, nombre de docs, indexes, etc.
```

**currentOp** : Opérations en cours
```javascript
db.currentOp()
// Retourne : requêtes actives, durée, locks
```

**killOp** : Tuer une opération bloquante
```javascript
db.killOp(opId)
```

**Cas d'usage :**
- currentOp : Identifier les slow queries en temps réel
- killOp : Tuer une requête qui bloque tout
- collStats : Vérifier la taille avant une migration

---

#### 13.3 Profiler de requêtes
**Durée : 2-3 heures**

Enregistrement des slow queries.

**Ce que vous maîtriserez :**
- Activation du profiler (niveau 0, 1, 2)
- Analyse de system.profile
- Identification des requêtes lentes
- Optimisation basée sur les résultats

**Niveaux du profiler :**
- **0** : Désactivé
- **1** : Log uniquement les slow queries (> threshold, ex: 100ms)
- **2** : Log toutes les requêtes (très verbeux, dev/debug seulement)

**Activation :**
```javascript
// Niveau 1 : slow queries > 50ms
db.setProfilingLevel(1, { slowms: 50 })

// Analyse
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()
```

**Impact :** Le profiler niveau 2 a un impact performance significatif. Utilisez-le uniquement pour le debugging.

---

#### 13.4-13.6 Logs, Tools, mongostat/mongotop
**Durée : 3-4 heures**

Outils de diagnostic et monitoring temps réel.

**Logs MongoDB :**
- Format et verbosité
- Log rotation
- Filtrage et analyse
- Intégration avec Splunk, ELK

**mongostat** :
- Vue en temps réel des opérations
- Affichage : inserts, queries, updates, deletes, getmore, etc.
- Utilisation : `mongostat --host <host> -u <user> -p <password> 1`

**mongotop** :
- Temps passé en lecture/écriture par collection
- Identification des hotspots

---

#### 13.7-13.9 Prometheus, Ops Manager, Alerting
**Durée : 5-6 heures**

Monitoring avancé et alerting.

**Prometheus + Grafana :**
- Scraping des métriques via MongoDB Exporter
- Stockage en time-series database
- Visualisation avec Grafana
- Alerting avec Alertmanager

**MongoDB Ops Manager / Cloud Manager :**
- Solution officielle MongoDB
- Monitoring, backup, automation
- Interface unifiée
- Requiert licence Enterprise

**Stratégie d'alerting :**
- Définir des seuils intelligents (basés sur l'historique)
- Éviter le bruit (false positives)
- Escalation (warning → critical)
- Intégration avec PagerDuty, Slack, etc.

**Exemples d'alertes :**
```
Critical :
- Primary down
- Disk usage > 90%
- Replication lag > 60s

Warning :
- CPU > 80% sustained
- Replication lag > 10s
- Slow queries > 100/min
```

---

#### 13.10-13.11 FTDC et Gestion mémoire
**Durée : 2-3 heures**

Diagnostics avancés et optimisation mémoire.

**FTDC (Full-Time Diagnostic Data Capture) :**
- Collecte automatique de métriques
- Stocké dans `diagnostic.data/`
- Utilisé par le support MongoDB
- Peut être analysé pour le debugging post-mortem

**WiredTiger Cache :**
- Cache interne de MongoDB
- Taille par défaut : 50% de (RAM - 1 GB)
- Ajustable avec `storage.wiredTiger.engineConfig.cacheSizeGB`
- Working set doit tenir dans le cache pour performance optimale

**Monitoring mémoire :**
- Resident memory (RAM utilisée)
- Virtual memory (mapped files)
- Page faults (indicateur de mémoire insuffisante)

---

**Pourquoi ce module est critique :** On ne peut pas gérer ce qu'on ne mesure pas. Le monitoring est la base de toute opération fiable.

## 🎯 Progression pédagogique

Cette partie suit une logique **sécuriser → protéger → surveiller** :

```
Sécurité (protéger l'accès) → Backup (protéger les données) → Monitoring (détecter les problèmes)
```

### Semaines 1-3 : Sécurité de bout en bout
**Focus : Construire des défenses en profondeur**

**Semaine 1 : Authentification et autorisation**
- Jours 1-2 : Concepts et architecture de sécurité
- Jours 3-4 : Configuration SCRAM et x.509
- Jours 5-7 : RBAC, rôles personnalisés, gestion utilisateurs

**Semaine 2 : Chiffrement**
- Jours 1-3 : TLS/SSL (setup, certificats, renouvellement)
- Jours 4-5 : Chiffrement au repos
- Jours 6-7 : CSFLE et Queryable Encryption

**Semaine 3 : Audit et conformité**
- Jours 1-3 : Configuration audit logging, analyse
- Jours 4-5 : Network security, firewalls, IP whitelisting
- Jours 6-7 : Security checklist, hardening, testing

**Livrables :**
- MongoDB sécurisé avec authentification + TLS
- Rôles RBAC documentés
- Audit logging configuré
- Security assessment report

---

### Semaines 4-5 : Sauvegarde et restauration
**Focus : Garantir la résilience**

**Semaine 4 : Stratégies et outils**
- Jours 1-2 : Définition RPO/RTO, stratégies de backup
- Jours 3-4 : mongodump/mongorestore, snapshots
- Jours 5-7 : Backup de Replica Sets et clusters shardés

**Semaine 5 : Automatisation et tests**
- Jours 1-2 : Atlas Backup, PITR
- Jours 3-4 : Automatisation (scripts, scheduling)
- Jours 5-7 : Tests de restauration, disaster recovery drills

**Livrables :**
- Stratégie de backup documentée et implémentée
- Backups automatisés et monitorés
- Tests de restauration réussis
- Runbooks de recovery

---

### Semaines 6-7 : Monitoring et administration
**Focus : Visibilité et proactivité**

**Semaine 6 : Métriques et outils**
- Jours 1-2 : Métriques clés, commandes d'administration
- Jours 3-4 : Profiler, logs, mongostat/mongotop
- Jours 5-7 : Setup Prometheus + Grafana

**Semaine 7 : Alerting et optimisation**
- Jours 1-3 : Configuration alerting, intégrations
- Jours 4-5 : Ops Manager, FTDC
- Jours 6-7 : Gestion mémoire, optimisations

**Livrables :**
- Dashboards Grafana opérationnels
- Alerting configuré et testé
- Documentation de troubleshooting
- SLIs et SLOs définis

---

**Rythme recommandé :** 3-4 heures par jour, avec des sessions pratiques intensives pour la configuration.

## 🧠 Principes opérationnels fondamentaux

### 1. La sécurité est un processus, pas un produit

> La sécurité n'est jamais "terminée". C'est une amélioration continue.

**Application :**
- Revues de sécurité trimestrielles
- Veille sur les CVE MongoDB
- Tests d'intrusion annuels
- Formation continue des équipes

### 2. Assume breach (présumer la compromise)

> Concevez en supposant qu'un attaquant a déjà pénétré une couche.

**Application :**
- Chiffrement même en interne (TLS)
- Segmentation réseau (DMZ, subnets privés)
- Principe du moindre privilège
- Audit logging activé

### 3. Les backups non testés sont une fausse sécurité

> 50% des restaurations échouent lors du premier essai réel.

**Application :**
- Drill de restauration mensuel minimum
- Mesure du RTO réel (pas théorique)
- Documentation mise à jour après chaque drill
- Formation des équipes

### 4. Monitoring : Signal vs Noise

> Trop d'alertes = alertes ignorées = incidents manqués.

**Application :**
- Définir des seuils intelligents (basés sur l'historique)
- Réduire les false positives agressivement
- Escalation progressive (warning → error → critical)
- Postmortem après chaque incident

### 5. Automatisation impitoyable

> Tout processus manuel sera oublié ou mal exécuté.

**Application :**
- Infrastructure as Code (Terraform, Ansible)
- Backups automatisés avec vérification
- Alerting automatisé
- Documentation as Code (versionnée)

### 6. Documentation is destiny

> Dans 6 mois, vous aurez tout oublié. Documentez maintenant.

**Application :**
- Runbooks pour chaque incident type
- Architecture decision records (ADRs)
- Postmortems après chaque incident
- Onboarding documentation

## 🚦 Validation des acquis

Avant de passer à la Partie 6, vous devez maîtriser :

### Checklist Sécurité
- [ ] J'ai configuré l'authentification SCRAM correctement
- [ ] Je sais générer et gérer des certificats x.509
- [ ] J'ai créé des rôles RBAC suivant le principe du moindre privilège
- [ ] J'ai activé TLS/SSL sur tous les nœuds
- [ ] Je comprends le chiffrement au repos et CSFLE
- [ ] J'ai configuré l'audit logging
- [ ] J'ai sécurisé le réseau (firewalls, bind_ip, VPN)
- [ ] J'ai effectué un security assessment complet

### Checklist Backup/Restore
- [ ] J'ai défini RPO et RTO pour mon système
- [ ] J'ai implémenté une stratégie de backup complète
- [ ] Je maîtrise mongodump/mongorestore
- [ ] Je peux créer des snapshots cohérents
- [ ] J'ai testé une restauration complète avec succès
- [ ] J'ai automatisé les backups avec monitoring
- [ ] Je peux effectuer un PITR (si applicable)
- [ ] J'ai un plan de disaster recovery documenté

### Checklist Monitoring
- [ ] J'ai identifié les métriques critiques pour mon système
- [ ] J'ai configuré un monitoring complet (Prometheus/Grafana ou équivalent)
- [ ] J'ai des dashboards opérationnels
- [ ] J'ai configuré des alertes intelligentes (pas de bruit)
- [ ] Je peux diagnostiquer rapidement les problèmes de performance
- [ ] Je maîtrise les commandes d'administration (serverStatus, currentOp, etc.)
- [ ] J'utilise le profiler pour identifier les slow queries
- [ ] J'ai documenté les procédures de troubleshooting

### Checklist Opérationnelle
- [ ] J'ai des runbooks pour tous les incidents courants
- [ ] Je peux effectuer une maintenance sans downtime
- [ ] J'ai un processus de change management
- [ ] J'ai mis en place des postmortems blameless
- [ ] Je peux effectuer un rolling upgrade
- [ ] J'ai des SLIs et SLOs définis
- [ ] Je surveille mes SLOs avec des dashboards

**Objectif :** Cocher 95%+ de ces cases. En production, la rigueur opérationnelle est non négociable.

## 🎯 Projet pratique : MongoDB Production-Ready

### Projet intégré : Système complet production-ready
**Durée : 40-50 heures**

**Objectif :** Déployer un système MongoDB sécurisé, sauvegardé et monitoré, prêt pour la production.

**Contexte :**
Application SaaS multi-tenant avec des exigences strictes :
- Données sensibles (PII - Personally Identifiable Information)
- SLA 99.9% uptime
- RPO = 1 heure, RTO = 15 minutes
- Conformité GDPR

**Tâches :**

**Phase 1 : Sécurité (15h)**
1. Déployer un Replica Set à 3 membres
2. Configurer l'authentification SCRAM
3. Créer des rôles RBAC pour : application, backup, monitoring, admin
4. Activer TLS/SSL avec certificats
5. Configurer firewalls et IP whitelisting
6. Activer l'audit logging
7. Implémenter le chiffrement au repos
8. Tester avec un security assessment tool

**Phase 2 : Backup/Restore (10h)**
9. Définir et documenter RPO/RTO
10. Implémenter des backups automatisés (snapshots + mongodump)
11. Configurer la rétention et la règle 3-2-1
12. Chiffrer les backups
13. Automatiser avec scripts + cron
14. Monitorer le succès des backups
15. Effectuer un drill de restauration complet

**Phase 3 : Monitoring (15h)**
16. Identifier les métriques critiques
17. Déployer Prometheus + MongoDB Exporter
18. Créer des dashboards Grafana (infra, perf, sécurité)
19. Configurer des alertes (critical, warning)
20. Intégrer avec un système de notification (Slack, PagerDuty)
21. Tester les alertes (simuler des pannes)
22. Documenter les procédures de troubleshooting

**Livrables :**
- Infrastructure complète (code Terraform/Ansible)
- Documentation d'architecture
- Security assessment report
- Stratégie de backup documentée
- Dashboards et alertes opérationnels
- Runbooks complets (incidents, maintenance)
- Preuve de tests (restauration, failover, alerting)

**Critères de validation :**
- ✅ Authentification et autorisation configurées
- ✅ TLS activé sur toutes les communications
- ✅ Audit logging opérationnel
- ✅ Backups automatisés et testés
- ✅ Monitoring complet avec alertes
- ✅ SLA 99.9% réalisable (testé avec failover)
- ✅ RPO/RTO respectés (testé avec restauration)
- ✅ Documentation complète

**Compétences validées :**
- Sécurité de bout en bout
- Résilience et recovery
- Observabilité et proactivité
- Rigueur opérationnelle

Ce projet constitue un excellent portfolio pour un poste de DBA ou SRE MongoDB.

## 📊 Tableau de bord opérationnel

### Métriques clés à surveiller en permanence

| Catégorie | Métrique | Seuil Warning | Seuil Critical | Action |
|-----------|----------|---------------|----------------|--------|
| **Disponibilité** | Primary disponible | N/A | Down | Escalade immédiate |
| | Member state changes | > 2/jour | > 5/jour | Investigate |
| **Performance** | Query latency p95 | > 100ms | > 500ms | Analyze slow queries |
| | Ops queue length | > 100 | > 1000 | Check locks/indexes |
| **Ressources** | CPU usage | > 80% | > 95% | Scale or optimize |
| | RAM usage | > 80% | > 95% | Add RAM or optimize |
| | Disk usage | > 80% | > 90% | Add storage |
| | Disk IOPS | > 80% | > 95% | Upgrade disk |
| **Réplication** | Replication lag | > 10s | > 60s | Check network/load |
| | Oplog window | < 24h | < 12h | Increase oplog size |
| **Sécurité** | Failed auth attempts | > 10/min | > 100/min | Possible attack |
| | Unusual access pattern | Detected | Detected | Investigate |
| **Backup** | Backup failure | 1 failure | 2 consecutive | Urgent fix |
| | Last successful backup | > 25h | > 36h | Check backup job |

## 🌟 Conseils d'administrateur MongoDB

### 1. La paranoïa est une vertu
Anticipez toujours le pire : hardware failure, data corruption, ransomware, insider threats. Préparez-vous pour tout.

### 2. Test, test, test
Ne présumez jamais. Testez vos backups, vos failovers, vos alertes, vos procédures. Régulièrement.

### 3. Document everything, version everything
Infrastructure as Code, documentation as Code. Tout doit être versionné et reproductible.

### 4. Automate the boring stuff
Tout ce qui est répétitif doit être automatisé. Focus sur ce qui nécessite vraiment l'intelligence humaine.

### 5. Learn from incidents
Chaque incident est une opportunité d'apprentissage. Postmortem blameless systématique.

### 6. Keep it simple
La complexité est l'ennemie de la sécurité et de la fiabilité. Le système le plus simple qui fonctionne est le meilleur.

### 7. Security is everyone's job
La sécurité n'est pas que l'affaire des admins. Formez les développeurs, les ops, tout le monde.

### 8. Stay updated
Veille technologique constante : CVEs, nouvelles versions, bonnes pratiques émergentes.

## 📚 Ressources complémentaires

### Documentation officielle
- [MongoDB Security Checklist](https://www.mongodb.com/docs/manual/administration/security-checklist/)
- [MongoDB Backup Methods](https://www.mongodb.com/docs/manual/core/backups/)
- [MongoDB Monitoring](https://www.mongodb.com/docs/manual/administration/monitoring/)
- [Production Notes](https://www.mongodb.com/docs/manual/administration/production-notes/)

### Certifications
- **MongoDB Certified DBA Associate** (indispensable)
- **MongoDB Certified DBA Professional** (avancé)

### Livres
- *MongoDB: The Definitive Guide* (3rd ed.) - Administration chapters
- *Site Reliability Engineering* (Google) - Principes SRE
- *The Phoenix Project* - DevOps culture

### Outils recommandés
- **Prometheus + Grafana** : Monitoring et alerting
- **MongoDB Ops Manager** : Solution officielle
- **Terraform** : Infrastructure as Code
- **Ansible** : Configuration management
- **Vault (HashiCorp)** : Gestion des secrets

## 🚀 Et après ?

Une fois cette partie maîtrisée, vous serez un **administrateur MongoDB production-ready**. Vous saurez :

- Sécuriser MongoDB contre toutes les menaces connues
- Garantir la disponibilité de vos données avec des backups testés
- Opérer un système MongoDB 24/7 avec confiance
- Détecter et résoudre les problèmes de façon proactive
- Respecter les exigences de conformité réglementaire

La **Partie 6** vous enseignera MongoDB Atlas et le Cloud, vous permettant de déléguer une grande partie de ces opérations à un service managé, tout en conservant la maîtrise des concepts.

La **Partie 7** couvrira le développement et l'intégration, vous permettant de mieux collaborer avec les équipes de développement.

Mais d'abord, **maîtrisez cette Partie 5**. Les opérations en production ne tolèrent pas l'à-peu-près. Une mauvaise configuration de sécurité, un backup non testé, ou un monitoring défaillant peut avoir des conséquences catastrophiques.

**La rigueur opérationnelle est ce qui différencie un système qui fonctionne d'un système fiable.**

---

**Prêt à devenir un administrateur MongoDB de classe mondiale ? Allons-y ! 🔐**

---

**Prochaine étape :** [Module 11 - Sécurité →](/11-securite/README.md)

---

*💡 Citation du jour : "Security is not a product, but a process." - Bruce Schneier (cryptographe et expert en sécurité informatique)*

⏭️ [Module 11 - Sécurité →](/11-securite/README.md)
