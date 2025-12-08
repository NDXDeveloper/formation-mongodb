🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.1 Vue d'Ensemble de la Sécurité MongoDB

## Introduction

La sécurité dans MongoDB est un système multicouche sophistiqué conçu pour protéger vos données contre un large éventail de menaces. Contrairement aux bases de données relationnelles traditionnelles, MongoDB nécessite une approche de sécurité adaptée à son architecture distribuée et à son modèle de données document. Cette section fournit une vue d'ensemble complète des mécanismes de sécurité disponibles et de leur mise en œuvre dans des environnements de production.

## Modèle de Menaces MongoDB

### Vecteurs d'Attaque Principaux

#### 1. Accès Non Autorisé

**Scénarios courants** :
- Instances MongoDB exposées sur Internet sans authentification
- Credentials par défaut ou faibles
- Absence de contrôle d'accès réseau
- Élévation de privilèges via exploitation de vulnérabilités

**Impact** :
- Lecture/modification/suppression de données
- Déploiement de ransomware
- Utilisation comme pivot pour d'autres attaques

**Exemple réel** : En 2017, plus de 26 000 instances MongoDB non sécurisées ont été compromises, avec des données effacées et des demandes de rançon.

#### 2. Interception de Données en Transit

**Scénarios** :
- Communications non chiffrées entre client et serveur
- Man-in-the-middle sur réseaux non sécurisés
- Sniffing de paquets sur le réseau interne

**Données exposées** :
- Credentials d'authentification
- Données métier sensibles
- Requêtes et résultats de requêtes

#### 3. Compromission des Données au Repos

**Scénarios** :
- Accès physique aux serveurs
- Vol de disques durs ou backups
- Accès non autorisé au système de fichiers
- Snapshots cloud mal protégés

#### 4. Attaques par Injection

**Types d'injection MongoDB** :
```javascript
// VULNÉRABLE : Injection NoSQL
db.users.find({ username: req.body.username, password: req.body.password });

// Si l'attaquant envoie : { username: "admin", password: { $ne: null } }
// La requête devient : find({ username: "admin", password: { $ne: null } })
// Qui retourne l'utilisateur admin sans vérifier le mot de passe
```

#### 5. Déni de Service (DoS)

**Vecteurs** :
- Requêtes mal optimisées consommant les ressources
- Connexions massives saturant le pool
- Opérations d'écriture massives saturant l'I/O
- Exploitation de bugs ou vulnérabilités

### Matrice de Risques

| Menace | Probabilité | Impact | Priorité | Mitigation Principale |
|--------|-------------|--------|----------|------------------------|
| Accès non autorisé | **Élevée** | **Critique** | P0 | Authentification + Autorisation |
| Interception données | Moyenne | Élevé | P1 | TLS/SSL |
| Données au repos | Faible | Critique | P1 | Encryption at Rest |
| Injection NoSQL | Moyenne | Élevé | P1 | Validation entrées + Paramétrage |
| DoS | Moyenne | Moyen | P2 | Rate limiting + Monitoring |
| Élévation privilèges | Faible | Critique | P1 | RBAC strict |
| Insider threat | Faible | Critique | P2 | Audit + Séparation des rôles |

## Architecture de Sécurité Multicouche

MongoDB implémente une approche **Defense in Depth** avec 7 couches de protection :

```
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 1 : SÉCURITÉ RÉSEAU                                     │
│  • Firewall rules                                               │
│  • IP Whitelisting                                              │
│  • VPC / Subnet isolation                                       │
│  • Network ACLs                                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 2 : CHIFFREMENT EN TRANSIT (TLS/SSL)                    │
│  • TLS 1.2+ obligatoire                                         │
│  • Certificats x.509                                            │
│  • Perfect Forward Secrecy                                      │
│  • Validation des certificats                                   │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 3 : AUTHENTIFICATION                                    │
│  • SCRAM-SHA-256 (défaut)                                       │
│  • x.509 Certificates                                           │
│  • LDAP / Active Directory                                      │
│  • Kerberos                                                     │
│  • OIDC (MongoDB 7.0+)                                          │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 4 : AUTORISATION (RBAC)                                 │
│  • Rôles intégrés (built-in roles)                              │
│  • Rôles personnalisés (custom roles)                           │
│  • Privilèges granulaires                                       │
│  • Database-level / Collection-level permissions                │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 5 : CHIFFREMENT AU REPOS                                │
│  • WiredTiger Encryption                                        │
│  • Key Management (KMIP, local)                                 │
│  • Filesystem-level encryption (LUKS)                           │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 6 : CHIFFREMENT AU NIVEAU APPLICATIF                    │
│  • Client-Side Field Level Encryption (CSFLE)                   │
│  • Queryable Encryption                                         │
│  • Application-level encryption                                 │
└─────────────────────────────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  COUCHE 7 : AUDIT ET MONITORING                                 │
│  • Audit logs détaillés                                         │
│  • Monitoring temps réel                                        │
│  • Alerting automatisé                                          │
│  • SIEM integration                                             │
└─────────────────────────────────────────────────────────────────┘
```

### Détails par Couche

#### Couche 1 : Sécurité Réseau

La première ligne de défense empêche l'accès non autorisé au niveau réseau.

**Configuration recommandée** :

```yaml
# mongod.conf
net:
  # Bind uniquement sur interfaces privées
  bindIp: 127.0.0.1,10.0.1.100
  port: 27017

  # Utiliser un port non-standard (optionnel, security by obscurity)
  # port: 27118

  # Limite de connexions
  maxIncomingConnections: 1000
```

**Règles Firewall (iptables)** :

```bash
# Autoriser uniquement les serveurs applicatifs
iptables -A INPUT -p tcp --dport 27017 -s 10.0.2.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -j DROP

# Autoriser la réplication entre membres du replica set
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.101 -j ACCEPT
iptables -A INPUT -p tcp --dport 27017 -s 10.0.1.102 -j ACCEPT
```

**AWS Security Group** :

```hcl
# Terraform
resource "aws_security_group" "mongodb" {
  name        = "mongodb-sg"
  description = "Security group for MongoDB cluster"
  vpc_id      = aws_vpc.main.id

  # Autoriser MongoDB depuis le security group applicatif
  ingress {
    description     = "MongoDB from application tier"
    from_port       = 27017
    to_port         = 27017
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  # Autoriser réplication inter-replica set
  ingress {
    description = "MongoDB replication"
    from_port   = 27017
    to_port     = 27017
    protocol    = "tcp"
    self        = true
  }

  # Pas d'accès sortant nécessaire pour MongoDB
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "mongodb-security-group"
  }
}
```

#### Couche 2 : Chiffrement en Transit (TLS/SSL)

Protection des données circulant sur le réseau.

**Configuration TLS de Production** :

```yaml
# mongod.conf
net:
  tls:
    mode: requireTLS  # Options: disabled, allowTLS, preferTLS, requireTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    certificateKeyFilePassword: <mot_de_passe_chiffré>
    CAFile: /etc/ssl/mongodb/ca.pem
    allowConnectionsWithoutCertificates: false
    allowInvalidCertificates: false
    allowInvalidHostnames: false

    # Désactiver les protocoles et ciphers faibles
    disabledProtocols: TLS1_0,TLS1_1

    # Ciphers recommandés (TLS 1.2+)
    # Sera configuré automatiquement avec des valeurs sûres
```

**Génération de Certificats** :

```bash
# CA Root Certificate
openssl genrsa -out ca-key.pem 4096
openssl req -new -x509 -days 3650 -key ca-key.pem -out ca.pem \
  -subj "/C=FR/ST=IDF/L=Paris/O=MonEntreprise/CN=MongoDB CA"

# Server Certificate
openssl genrsa -out server-key.pem 4096
openssl req -new -key server-key.pem -out server.csr \
  -subj "/C=FR/ST=IDF/L=Paris/O=MonEntreprise/CN=mongodb01.internal"

# Signer avec la CA
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca-key.pem \
  -CAcreateserial -out server-cert.pem -days 365

# Combiner certificat et clé
cat server-cert.pem server-key.pem > server.pem

# Sécuriser les permissions
chmod 400 server.pem ca.pem
chown mongodb:mongodb server.pem ca.pem
```

#### Couche 3 : Authentification

Vérification de l'identité des clients.

**Activation de l'Authentification** :

```yaml
# mongod.conf
security:
  authorization: enabled

  # Clé inter-membres du replica set
  keyFile: /etc/mongodb/keyfile
```

**Génération de Keyfile** :

```bash
# Générer une clé aléatoire de 1024 octets
openssl rand -base64 756 > /etc/mongodb/keyfile
chmod 400 /etc/mongodb/keyfile
chown mongodb:mongodb /etc/mongodb/keyfile
```

**Connexion avec Authentification** :

```bash
# Via mongosh
mongosh "mongodb://admin:SecurePassword123!@mongodb01.internal:27017/admin?authSource=admin&tls=true&tlsCAFile=/etc/ssl/ca.pem"

# Via URI complète
mongodb://username:password@host1:27017,host2:27017,host3:27017/database?replicaSet=rs0&authSource=admin&tls=true
```

#### Couche 4 : Autorisation (RBAC)

Contrôle des actions autorisées après authentification.

**Hiérarchie des Rôles** :

```
┌─────────────────────────────────────┐
│   Rôles Super Admin                 │
│   • root                            │
│   • __system                        │
└─────────────────────────────────────┘
                ▼
┌─────────────────────────────────────┐
│   Rôles Admin Base de Données       │
│   • dbOwner                         │
│   • userAdmin                       │
│   • dbAdmin                         │
└─────────────────────────────────────┘
                ▼
┌─────────────────────────────────────┐
│   Rôles Lecture/Écriture            │
│   • readWrite                       │
│   • read                            │
└─────────────────────────────────────┘
                ▼
┌─────────────────────────────────────┐
│   Rôles Personnalisés               │
│   • Privilèges granulaires          │
└─────────────────────────────────────┘
```

#### Couche 5 : Chiffrement au Repos

Protection des données stockées sur disque.

**Configuration WiredTiger Encryption** :

```yaml
# mongod.conf
security:
  enableEncryption: true
  encryptionCipherMode: AES256-CBC
  encryptionKeyFile: /etc/mongodb/encryption-key

  # Ou utilisation d'un KMS externe
  # kmip:
  #   serverName: kmip.example.com
  #   port: 5696
  #   clientCertificateFile: /etc/ssl/kmip-client.pem
  #   serverCAFile: /etc/ssl/kmip-ca.pem
```

**Génération de Master Key** :

```bash
# Générer une clé de chiffrement
openssl rand -base64 32 > /etc/mongodb/encryption-key
chmod 400 /etc/mongodb/encryption-key
chown mongodb:mongodb /etc/mongodb/encryption-key
```

#### Couche 6 : Chiffrement au Niveau Applicatif (CSFLE)

Chiffrement transparent côté client pour les données ultra-sensibles.

**Architecture CSFLE** :

```
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│              │        │              │        │              │
│  Application │───────▶│  MongoDB     │───────▶│   MongoDB    │
│              │ Chiffré│  Driver      │ Chiffré│   Server     │
│              │        │  (Auto)      │        │              │
└──────────────┘        └──────────────┘        └──────────────┘
       │                       │                        │
       │                       │                        │
       ▼                       ▼                        │
┌──────────────┐        ┌──────────────┐                │
│              │        │              │                │
│  Data Keys   │◀───────│    Master    │                │
│  (MongoDB)   │        │     Key      │                │
│              │        │   (KMS)      │                │
└──────────────┘        └──────────────┘                │
                                                        │
                        Les données restent chiffrées   │
                        sur le serveur MongoDB ◀────────┘
```

#### Couche 7 : Audit et Monitoring

Traçabilité complète des actions.

**Configuration de l'Audit** :

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log

  # Filtrer les événements audités
  filter: '{
    $or: [
      { "atype": "authenticate" },
      { "atype": "createUser" },
      { "atype": "dropUser" },
      { "atype": "createRole" },
      { "atype": "dropRole" },
      { "atype": "grantRolesToUser" },
      { "atype": "revokeRolesFromUser" },
      { "atype": "shutdown" },
      { "users": { "$elemMatch": { "db": "admin" } } }
    ]
  }'
```

## Comparaison des Mécanismes de Sécurité

### Authentification : Comparatif des Méthodes

| Méthode | Sécurité | Complexité | Cas d'Usage | Entreprise |
|---------|----------|------------|-------------|------------|
| **SCRAM-SHA-256** | ⭐⭐⭐⭐ | Faible | Défaut, applications simples | Non requis |
| **x.509** | ⭐⭐⭐⭐⭐ | Élevée | Forte sécurité, pas de mots de passe | Non requis |
| **LDAP** | ⭐⭐⭐⭐ | Moyenne | Intégration AD/LDAP existant | **Requis** |
| **Kerberos** | ⭐⭐⭐⭐⭐ | Très élevée | Environnements hautement sécurisés | **Requis** |
| **OIDC** | ⭐⭐⭐⭐ | Moyenne | SSO moderne, Atlas | Non requis |

### Chiffrement : Options et Performance

| Type | Protection | Impact Performance | Configuration | Recommandation |
|------|------------|--------------------|--------------|--------------  |
| **TLS en transit** | Man-in-the-middle | 2-5% | Moyenne | **Obligatoire en prod** |
| **Encryption at Rest** | Accès physique/disque | 5-10% | Moyenne | **Obligatoire si sensible** |
| **CSFLE** | Base compromise, admins | 10-30% | Élevée | Données ultra-sensibles |
| **Queryable Encryption** | Base compromise, recherche | 15-40% | Très élevée | Données sensibles + recherche |

## Configuration de Sécurité par Environnement

### Développement Local

```yaml
# mongod-dev.conf
# Sécurité minimale pour développement
net:
  bindIp: 127.0.0.1
  port: 27017

security:
  authorization: enabled  # Même en dev !

# Pas de TLS en développement local (optionnel)
# Pas d'encryption at rest
# Audit désactivé
```

```javascript
// Créer un utilisateur de dev
use admin
db.createUser({
  user: "dev",
  pwd: "dev123",
  roles: [
    { role: "readWrite", db: "myapp_dev" },
    { role: "dbAdmin", db: "myapp_dev" }
  ]
})
```

### Staging/Pré-Production

```yaml
# mongod-staging.conf
# Configuration proche de la production
net:
  bindIp: 0.0.0.0  # Ou IP privée spécifique
  port: 27017
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/staging-server.pem
    CAFile: /etc/ssl/mongodb/staging-ca.pem

security:
  authorization: enabled
  keyFile: /etc/mongodb/staging-keyfile

auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log
  # Audit sélectif pour ne pas surcharger
  filter: '{ "atype": { "$in": ["authenticate", "createUser", "dropUser"] } }'

# Encryption at rest recommandée
# security:
#   enableEncryption: true
#   encryptionKeyFile: /etc/mongodb/staging-encryption-key
```

### Production

```yaml
# mongod-prod.conf
# Configuration sécurité maximale
net:
  bindIp: 10.0.1.100  # IP privée uniquement
  port: 27017
  maxIncomingConnections: 2000

  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/prod-server.pem
    CAFile: /etc/ssl/mongodb/prod-ca.pem
    allowConnectionsWithoutCertificates: false
    allowInvalidCertificates: false
    allowInvalidHostnames: false
    disabledProtocols: TLS1_0,TLS1_1

security:
  authorization: enabled
  keyFile: /etc/mongodb/prod-keyfile

  # Chiffrement au repos OBLIGATOIRE
  enableEncryption: true

  # KMIP pour production (recommandé)
  kmip:
    serverName: kmip.prod.internal
    port: 5696
    clientCertificateFile: /etc/ssl/kmip-client.pem
    serverCAFile: /etc/ssl/kmip-ca.pem

auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.log

  # Audit complet en production
  filter: '{
    $or: [
      { "atype": { "$in": ["authenticate", "authCheck"] } },
      { "atype": { "$regex": "^(create|drop|update|grant|revoke)" } },
      { "param.ns": { "$regex": "^(admin|config)\\." } }
    ]
  }'

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen

  # Verbosité sécurité
  component:
    accessControl:
      verbosity: 2

# Paramètres de performance et stabilité
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 10
    collectionConfig:
      blockCompressor: snappy

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
```

## Gestion des Identités et Accès (IAM)

### Principes de Base

#### 1. Séparation des Responsabilités

```
┌────────────────────────────────────────────────────────────┐
│  ADMINISTRATEURS INFRASTRUCTURE                            │
│  • Gestion des serveurs                                    │
│  • Déploiement MongoDB                                     │
│  • Backups système                                         │
│  └──── Rôles: clusterAdmin, backup, restore                │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  ADMINISTRATEURS BASE DE DONNÉES (DBA)                     │
│  • Gestion des utilisateurs                                │
│  • Optimisation des index                                  │
│  • Monitoring des performances                             │
│  └──── Rôles: dbAdmin, userAdmin                           │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  DÉVELOPPEURS / APPLICATIONS                               │
│  • Lecture/écriture des données applicatives               │
│  • Aucun accès admin                                       │
│  └──── Rôles: readWrite, read (par base spécifique)        │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  LECTURE SEULE / ANALYTICS                                 │
│  • Reporting et analyses                                   │
│  • Aucune modification                                     │
│  └──── Rôles: read (bases spécifiques)                     │
└────────────────────────────────────────────────────────────┘
```

#### 2. Matrice de Contrôle d'Accès

| Rôle | Admin DB | Créer Index | Insert | Update | Delete | Drop Collection | Créer User |
|------|----------|-------------|--------|--------|--------|-----------------|------------|
| **root** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **dbOwner** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| **dbAdmin** | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |
| **userAdmin** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ✅ |
| **readWrite** | ❌ | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ |
| **read** | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

### Stratégie de Gestion des Utilisateurs

#### Pattern 1 : Utilisateurs Applicatifs

```javascript
// Créer un utilisateur pour une application
use admin
db.createUser({
  user: "app_myservice",
  pwd: passwordPrompt(),  // Ne jamais hardcoder les passwords
  roles: [
    { role: "readWrite", db: "myservice_prod" },
    { role: "read", db: "reference_data" }
  ],
  mechanisms: ["SCRAM-SHA-256"],
  authenticationRestrictions: [
    {
      clientSource: ["10.0.2.0/24"],  // Limiter aux IPs applicatives
      serverAddress: ["10.0.1.100"]
    }
  ]
})
```

#### Pattern 2 : Utilisateurs Administratifs

```javascript
// DBA avec accès limité
use admin
db.createUser({
  user: "dba_john",
  pwd: passwordPrompt(),
  roles: [
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readAnyDatabase", db: "admin" }
  ],
  mechanisms: ["SCRAM-SHA-256"],
  authenticationRestrictions: [
    {
      clientSource: ["10.0.10.0/24"]  // Réseau admin uniquement
    }
  ]
})
```

#### Pattern 3 : Service Accounts (CI/CD)

```javascript
// Compte pour les déploiements automatisés
use admin
db.createUser({
  user: "svc_cicd",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "myapp_prod" },
    { role: "dbAdmin", db: "myapp_prod" }  // Pour créer index, collections
  ],
  mechanisms: ["SCRAM-SHA-256"]
})
```

## Rotation des Credentials

### Politique de Rotation

| Type de Credential | Fréquence de Rotation | Méthode |
|-------------------|----------------------|---------|
| Utilisateurs humains | **90 jours** | Manuel avec notification |
| Service accounts | **180 jours** | Automatisé via secret manager |
| Keyfile réplication | **Annuel** | Rolling restart |
| Certificats TLS | **Avant expiration** | Automatisé (Let's Encrypt) |
| Master encryption keys | **Annuel** | Procédure dédiée |

### Procédure de Rotation des Passwords

```javascript
// Script de rotation automatisé
use admin

// 1. Créer nouveau password
const newPassword = generateSecurePassword();

// 2. Mettre à jour l'utilisateur
db.updateUser("app_myservice", {
  pwd: newPassword
});

// 3. Stocker dans le secret manager
storeInVault("mongodb/app_myservice", newPassword);

// 4. Notifier l'équipe
sendNotification("Password rotated for app_myservice");

// 5. Logger l'opération
db.audit.insertOne({
  timestamp: new Date(),
  action: "password_rotation",
  user: "app_myservice",
  performedBy: "automation"
});
```

### Rotation de Keyfile (Replica Set)

```bash
#!/bin/bash
# rotate-keyfile.sh

# 1. Générer nouvelle keyfile
openssl rand -base64 756 > /etc/mongodb/keyfile-new

# 2. Sur chaque membre, un par un :
# Secondary 1
scp keyfile-new mongodb-secondary1:/etc/mongodb/keyfile-new
ssh mongodb-secondary1 << 'EOF'
  sudo systemctl stop mongod
  sudo mv /etc/mongodb/keyfile /etc/mongodb/keyfile-old
  sudo mv /etc/mongodb/keyfile-new /etc/mongodb/keyfile
  sudo chmod 400 /etc/mongodb/keyfile
  sudo chown mongodb:mongodb /etc/mongodb/keyfile
  sudo systemctl start mongod
EOF

# Attendre réplication
sleep 30

# 3. Répéter pour Secondary 2
# ...

# 4. Step down primary et faire la rotation
mongosh --eval "rs.stepDown()"
sleep 10
# Répéter la procédure sur l'ancien primary
```

## Checklist de Sécurité Initiale

### Avant le Déploiement

- [ ] **Architecture réseau définie**
  - [ ] VPC et sous-réseaux configurés
  - [ ] Security groups / Firewall rules en place
  - [ ] Pas d'exposition Internet directe

- [ ] **Certificats TLS générés**
  - [ ] Certificat serveur pour chaque nœud
  - [ ] CA certificate configurée
  - [ ] Certificats clients si x.509

- [ ] **Keyfile généré**
  - [ ] Keyfile sécurisé (400 permissions)
  - [ ] Identique sur tous les membres du replica set

- [ ] **Master encryption key préparée**
  - [ ] Clé générée ou KMIP configuré
  - [ ] Backup de la clé dans un coffre-fort sécurisé

### Après le Déploiement

- [ ] **Authentification activée**
  - [ ] `security.authorization: enabled`
  - [ ] Utilisateur admin créé
  - [ ] Pas de compte par défaut actif

- [ ] **TLS configuré**
  - [ ] `net.tls.mode: requireTLS`
  - [ ] Certificats validés
  - [ ] Protocoles faibles désactivés

- [ ] **Utilisateurs créés**
  - [ ] Utilisateurs applicatifs avec privilèges minimaux
  - [ ] Utilisateurs admin avec restrictions IP
  - [ ] Pas de `root` utilisé en production

- [ ] **Audit activé**
  - [ ] `auditLog` configuré
  - [ ] Filtres d'audit appropriés
  - [ ] Rotation des logs configurée

- [ ] **Monitoring configuré**
  - [ ] Métriques de sécurité collectées
  - [ ] Alertes sur événements sensibles
  - [ ] Dashboard de sécurité opérationnel

- [ ] **Documentation**
  - [ ] Procédures d'accès documentées
  - [ ] Contacts d'urgence définis
  - [ ] Runbooks de sécurité créés

## Outils de Vérification de Sécurité

### Script de Validation de Configuration

```javascript
// security-audit.js
// À exécuter régulièrement pour vérifier la configuration

const checks = [
  {
    name: "Authentification activée",
    check: () => {
      const serverStatus = db.serverStatus();
      return serverStatus.security.authentication.mechanisms.length > 0;
    }
  },
  {
    name: "TLS activé",
    check: () => {
      const connStats = db.serverStatus().connections;
      return connStats.tlsCurrent !== undefined && connStats.tlsCurrent > 0;
    }
  },
  {
    name: "Audit activé",
    check: () => {
      const params = db.adminCommand({ getCmdLineOpts: 1 });
      return params.parsed.auditLog !== undefined;
    }
  },
  {
    name: "Pas d'utilisateurs sans mot de passe",
    check: () => {
      const users = db.system.users.find({}).toArray();
      return users.every(u => u.credentials !== undefined);
    }
  },
  {
    name: "Bind IP configuré (pas 0.0.0.0 exposé)",
    check: () => {
      const params = db.adminCommand({ getCmdLineOpts: 1 });
      const bindIp = params.parsed.net.bindIp;
      return bindIp !== "0.0.0.0" || params.parsed.net.tls.mode === "requireTLS";
    }
  }
];

// Exécuter les vérifications
print("=== MongoDB Security Audit ===\n");
checks.forEach(check => {
  try {
    const result = check.check();
    print(`[${result ? "✓" : "✗"}] ${check.name}`);
  } catch (e) {
    print(`[!] ${check.name}: ERROR - ${e.message}`);
  }
});
```

### Scanner de Sécurité MongoDB

```python
#!/usr/bin/env python3
# mongodb-security-scanner.py

from pymongo import MongoClient
import sys

def check_security(uri):
    """Vérifie la configuration de sécurité"""

    try:
        client = MongoClient(uri)
        admin_db = client.admin

        # Test 1: Authentification requise
        try:
            admin_db.command("ping")
            print("❌ CRITICAL: Authentification non requise!")
            return False
        except:
            print("✅ Authentification requise")

        # Se connecter avec credentials
        client = MongoClient(uri)

        # Test 2: Version MongoDB
        server_info = client.server_info()
        version = server_info['version']
        print(f"✅ MongoDB version: {version}")

        # Test 3: Utilisateurs avec privilèges root
        admin_db = client.admin
        users = admin_db.command("usersInfo")
        root_users = [u for u in users['users']
                     if any(r['role'] == 'root' for r in u['roles'])]

        if len(root_users) > 1:
            print(f"⚠️  WARNING: {len(root_users)} utilisateurs avec rôle 'root'")

        # Test 4: Encryption at rest
        db_stats = admin_db.command("serverStatus")
        if 'encryptionAtRest' in db_stats:
            print("✅ Encryption at rest activé")
        else:
            print("⚠️  WARNING: Encryption at rest non détecté")

        # Test 5: Audit
        cmd_opts = admin_db.command("getCmdLineOpts")
        if 'auditLog' in cmd_opts.get('parsed', {}):
            print("✅ Audit logs configurés")
        else:
            print("⚠️  WARNING: Audit logs non configurés")

        return True

    except Exception as e:
        print(f"❌ ERROR: {e}")
        return False

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Usage: python mongodb-security-scanner.py <mongodb_uri>")
        sys.exit(1)

    uri = sys.argv[1]
    check_security(uri)
```

## Meilleures Pratiques de Production

### 1. Ne Jamais Exposer MongoDB sur Internet

```yaml
# ❌ MAUVAIS
net:
  bindIp: 0.0.0.0  # Écoute sur toutes les interfaces
  port: 27017

# ✅ BON
net:
  bindIp: 127.0.0.1,10.0.1.100  # Localhost + IP privée uniquement
  port: 27017
```

### 2. Utiliser des Connexions Chiffrées Uniquement

```javascript
// ❌ MAUVAIS : Connexion non chiffrée
mongodb://user:pass@mongodb.example.com:27017/mydb

// ✅ BON : TLS obligatoire
mongodb://user:pass@mongodb.example.com:27017/mydb?tls=true&tlsCAFile=/path/to/ca.pem
```

### 3. Appliquer le Principe du Moindre Privilège

```javascript
// ❌ MAUVAIS : Trop de privilèges
db.createUser({
  user: "app_user",
  pwd: "password",
  roles: ["root"]  // Privilèges système complets !
});

// ✅ BON : Privilèges minimaux
db.createUser({
  user: "app_user",
  pwd: "password",
  roles: [
    { role: "readWrite", db: "myapp" }  // Uniquement la base nécessaire
  ]
});
```

### 4. Passwords Robustes et Stockage Sécurisé

```javascript
// ❌ MAUVAIS : Password faible hardcodé
const mongoUri = "mongodb://admin:admin123@localhost:27017";

// ✅ BON : Password fort depuis variable d'environnement
const mongoUri = `mongodb://${process.env.MONGO_USER}:${process.env.MONGO_PASS}@localhost:27017`;

// Password requirements:
// - Minimum 16 caractères
// - Majuscules, minuscules, chiffres, symboles
// - Pas de mots du dictionnaire
// - Rotation tous les 90 jours
```

### 5. Monitoring des Événements de Sécurité

```javascript
// Requête pour détecter les tentatives d'authentification échouées
db.adminCommand({
  getLog: "global"
}).log.filter(entry =>
  entry.includes("Authentication failed")
).forEach(print);

// Avec audit logs
db.audit.find({
  atype: "authenticate",
  result: { $ne: 0 }  // Authentifications échouées
}).sort({ ts: -1 }).limit(50);
```

## Intégration avec les Outils d'Entreprise

### SIEM (Security Information and Event Management)

**Forward des logs vers Splunk** :

```bash
# /etc/filebeat/filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/mongodb/audit.log
  json.keys_under_root: true
  json.add_error_key: true
  fields:
    service: mongodb
    environment: production

output.logstash:
  hosts: ["splunk-hec.internal:8088"]
  ssl.certificate_authorities: ["/etc/ssl/ca.pem"]
```

### Secret Management (HashiCorp Vault)

```bash
# Stocker les credentials MongoDB dans Vault
vault kv put secret/mongodb/prod \
  username=app_user \
  password=$(openssl rand -base64 32) \
  connection_string="mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=rs0"

# Lecture depuis l'application
MONGO_CREDS=$(vault kv get -field=connection_string secret/mongodb/prod)
```

### IAM Cloud (AWS IAM, Azure AD)

**MongoDB Atlas avec AWS IAM** :

```javascript
const { MongoClient } = require('mongodb');
const AWS = require('aws-sdk');

// Authentification via AWS IAM
const uri = `mongodb+srv://${AWS_ACCESS_KEY}:${AWS_SECRET_KEY}@cluster.mongodb.net/?authSource=$external&authMechanism=MONGODB-AWS`;

const client = new MongoClient(uri);
```

## Conclusion

La sécurité MongoDB repose sur une approche multicouche rigoureuse. Les points essentiels à retenir :

1. **Activez toujours l'authentification** - Même en développement
2. **Chiffrez en transit** - TLS 1.2+ obligatoire en production
3. **Chiffrez au repos** - Pour les données sensibles
4. **Principe du moindre privilège** - Permissions minimales nécessaires
5. **Auditez tout** - Traçabilité complète des accès
6. **Isolez le réseau** - Jamais d'exposition directe sur Internet
7. **Automatisez** - Rotation des credentials, scanning de sécurité
8. **Surveillez** - Monitoring temps réel et alerting

La sécurité n'est pas un état mais un **processus continu** qui doit évoluer avec les menaces et les besoins de l'organisation.

---

**Prochaines Sections** :
- **11.2** : Authentification (SCRAM, x.509, LDAP, Kerberos)
- **11.3** : Autorisation et RBAC
- **11.4** : Gestion des utilisateurs
- **11.5** : Chiffrement multi-niveaux

⏭️ [Authentification](/11-securite/02-authentification.md)
