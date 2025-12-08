🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.5 Chiffrement

## Introduction

Le chiffrement constitue un pilier fondamental de la sécurité MongoDB en environnement de production. Dans un contexte où les données sont de plus en plus ciblées par les cyberattaques et où les réglementations sur la protection des données se renforcent (RGPD, HIPAA, PCI-DSS), la mise en œuvre d'une stratégie de chiffrement robuste n'est plus optionnelle.

MongoDB propose une approche multicouche du chiffrement, permettant de protéger les données à chaque étape de leur cycle de vie : en transit sur le réseau, au repos sur le disque, et même au niveau applicatif avec un chiffrement au niveau des champs.

## Les trois piliers du chiffrement MongoDB

### 1. Chiffrement en transit (Encryption in Transit)

Protège les données lors de leur transmission entre :
- Client et serveur MongoDB
- Membres d'un Replica Set
- Composants d'un cluster shardé (mongos, config servers, shards)
- Outils d'administration et base de données

**Technologies utilisées** : TLS/SSL (Transport Layer Security)

**Cas d'usage critiques** :
- Communication sur des réseaux non sécurisés
- Conformité réglementaire (PCI-DSS niveau 4)
- Prévention des attaques Man-in-the-Middle (MITM)
- Protection contre l'écoute passive du trafic réseau

### 2. Chiffrement au repos (Encryption at Rest)

Protège les données stockées sur le disque :
- Fichiers de données WiredTiger
- Journaux (journal logs)
- Snapshots de backup
- Fichiers temporaires

**Technologies utilisées** :
- Chiffrement natif MongoDB (WiredTiger Encryption)
- Chiffrement au niveau du système de fichiers (LUKS, dm-crypt)
- Chiffrement au niveau du stockage (AWS EBS, Azure Disk Encryption)

**Cas d'usage critiques** :
- Protection contre le vol physique de serveurs
- Conformité avec les réglementations de protection des données
- Sécurisation des backups
- Décommissionnement sécurisé du matériel

### 3. Chiffrement au niveau des champs (Field Level Encryption)

Protège des champs spécifiques au niveau applicatif :
- **Client-Side Field Level Encryption (CSFLE)** : Chiffrement côté client avant l'envoi à MongoDB
- **Queryable Encryption** : CSFLE avec capacité de requêtage sur données chiffrées

**Cas d'usage critiques** :
- Données hautement sensibles (numéros de carte bancaire, SSN, données médicales)
- Modèle de sécurité zero-trust
- Séparation des responsabilités (administrateurs DB ne peuvent pas voir les données sensibles)
- Conformité stricte (HIPAA, PCI-DSS niveau 1)

## Vue d'ensemble des solutions de chiffrement

### Matrice de couverture

| Type de chiffrement | Données protégées | Gestion des clés | Performance | Complexité | Édition MongoDB |
|---------------------|-------------------|------------------|-------------|------------|-----------------|
| **TLS/SSL** | Transit réseau | Certificats X.509 | Impact faible (~5-10%) | Moyenne | Community/Enterprise |
| **Encryption at Rest** | Fichiers disque | KMIP/Local key file | Impact modéré (~10-15%) | Moyenne | Enterprise uniquement |
| **CSFLE** | Champs spécifiques | CMK + DEK (AWS KMS, Azure Key Vault, etc.) | Impact variable | Élevée | Enterprise uniquement |
| **Queryable Encryption** | Champs avec requêtes | CMK + tokens | Impact élevé | Très élevée | Enterprise 6.0+ |

### Architecture de sécurité en profondeur (Defense in Depth)

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                        │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Queryable Encryption / CSFLE (Champs sensibles)       │ │
│  │  • Chiffrement avant envoi à MongoDB                   │ │
│  │  • Clés gérées par l'application                       │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Network Layer                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  TLS/SSL (En transit)                                  │ │
│  │  • Chiffrement de toutes les communications            │ │
│  │  • Authentification mutuelle (mTLS)                    │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                    Storage Layer                            │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Encryption at Rest (Au repos)                         │ │
│  │  • Chiffrement des fichiers WiredTiger                 │ │
│  │  • Protection des backups                              │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Considérations de performance

### Impact sur les performances

Le chiffrement introduit inévitablement une surcharge CPU et une latence supplémentaire. Voici les impacts typiques observés en production :

#### TLS/SSL (Chiffrement en transit)
```
Impact CPU      : +5-10%
Impact latence  : +0.5-2ms par opération
Impact débit    : Négligeable avec matériel moderne (AES-NI)
Recommandation  : TOUJOURS activer en production
```

#### Encryption at Rest
```
Impact CPU      : +10-15%
Impact I/O      : +5-10%
Impact mémoire  : Minimal
Recommandation  : Activer pour toutes les données sensibles
```

#### CSFLE / Queryable Encryption
```
Impact CPU      : +15-40% (variable selon implémentation)
Impact latence  : +5-20ms par opération chiffrée
Impact réseau   : +10-30% (données chiffrées plus volumineuses)
Recommandation  : Utiliser sélectivement sur champs critiques
```

### Optimisation des performances avec chiffrement

**1. Utilisation d'instructions matérielles AES-NI**

Les processeurs modernes supportent AES-NI (Advanced Encryption Standard New Instructions), accélérant significativement les opérations cryptographiques.

```bash
# Vérifier le support AES-NI sur Linux
grep -o 'aes' /proc/cpuinfo | uniq
# Sortie attendue : aes

# Vérifier sur un système en production
lscpu | grep -i aes
```

**Gain de performance** : Jusqu'à 5x plus rapide pour les opérations de chiffrement/déchiffrement.

**2. Allocation mémoire pour le cache WiredTiger**

Avec Encryption at Rest, il est recommandé d'augmenter la taille du cache WiredTiger :

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # Au lieu de 4GB par défaut pour un serveur avec 16GB RAM
```

**Règle générale** :
- Sans chiffrement : 50% de la RAM disponible
- Avec chiffrement : 60-70% de la RAM disponible

**3. Choix des algorithmes de chiffrement**

Pour TLS/SSL, privilégier les cipher suites modernes et performantes :

```yaml
# Configuration recommandée pour MongoDB 6.0+
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem
    disabledProtocols: TLS1_0,TLS1_1
    # Cipher suites recommandées (ordre de préférence)
    allowedCiphers: >
      ECDHE-ECDSA-AES256-GCM-SHA384,
      ECDHE-RSA-AES256-GCM-SHA384,
      ECDHE-ECDSA-AES128-GCM-SHA256,
      ECDHE-RSA-AES128-GCM-SHA256
```

**Critères de sélection** :
- Forward Secrecy (ECDHE)
- Mode GCM (meilleure performance que CBC)
- AES-128 ou AES-256 selon exigences de sécurité

## Conformité et réglementation

### Exigences par standard

#### RGPD (Règlement Général sur la Protection des Données)

**Exigences** :
- Chiffrement des données personnelles recommandé (Article 32)
- Protection against accidental loss or destruction
- Pseudonymisation et chiffrement comme mesures techniques

**Implémentation MongoDB** :
```
✓ TLS/SSL pour toutes les communications
✓ Encryption at Rest pour les serveurs européens
✓ CSFLE pour les données hautement sensibles (données de santé)
✓ Audit logging activé
✓ Contrôle d'accès basé sur les rôles (RBAC)
```

#### PCI-DSS (Payment Card Industry Data Security Standard)

**Exigences critiques** :
- Requirement 3.4 : Chiffrement des PAN (Primary Account Number)
- Requirement 4.1 : Chiffrement en transit pour les transmissions de données de cartes

**Implémentation MongoDB** :
```
✓ TLS 1.2+ obligatoire (Requirement 4.1)
✓ CSFLE pour les numéros de carte (Requirement 3.4)
✓ Key rotation tous les 90 jours
✓ Séparation des environnements cardholder data (CDE)
✓ Logs d'accès complets
```

Configuration minimale PCI-DSS :

```javascript
// Création d'une collection avec CSFLE pour PCI compliance
db.createCollection("payments", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["customerId", "encryptedCardNumber", "timestamp"],
      properties: {
        encryptedCardNumber: {
          bsonType: "binData",
          description: "Encrypted PAN - PCI DSS Requirement 3.4"
        },
        encryptedCVV: {
          bsonType: "binData",
          description: "Encrypted CVV - Never store unencrypted"
        },
        last4Digits: {
          bsonType: "string",
          pattern: "^[0-9]{4}$",
          description: "Last 4 digits - Safe to store unencrypted"
        }
      }
    }
  }
})
```

#### HIPAA (Health Insurance Portability and Accountability Act)

**Exigences** :
- Chiffrement des PHI (Protected Health Information) at rest and in transit
- Audit complet des accès
- Contrôle d'accès strict

**Implémentation MongoDB** :
```
✓ TLS 1.3 recommandé
✓ Encryption at Rest obligatoire
✓ CSFLE pour SSN, diagnostic codes, prescriptions
✓ Queryable Encryption pour recherches sur données chiffrées
✓ Audit filters pour toutes les opérations PHI
✓ BAA (Business Associate Agreement) avec MongoDB Inc.
```

## Gestion des clés cryptographiques (Key Management)

### Hiérarchie des clés

MongoDB utilise une architecture de clés à plusieurs niveaux :

```
┌─────────────────────────────────────────────────────────┐
│  Customer Master Key (CMK)                              │
│  • Stockée dans un KMS externe (AWS KMS, Azure, etc.)   │
│  • Ne quitte jamais le KMS                              │
│  • Utilisée pour chiffrer les Data Encryption Keys      │
└─────────────────────────────────────────────────────────┘
                        ↓ chiffre
┌─────────────────────────────────────────────────────────┐
│  Data Encryption Key (DEK)                              │
│  • Générée par MongoDB ou l'application                 │
│  • Stockée chiffrée dans MongoDB                        │
│  • Utilisée pour chiffrer les données réelles           │
└─────────────────────────────────────────────────────────┘
                        ↓ chiffre
┌─────────────────────────────────────────────────────────┐
│  Données chiffrées                                      │
│  • Documents dans les collections                       │
│  • Champs spécifiques (CSFLE)                           │
└─────────────────────────────────────────────────────────┘
```

### Options de gestion des clés

#### 1. KMIP (Key Management Interoperability Protocol)

**Pour** : Encryption at Rest

**Compatibilité** :
- HashiCorp Vault
- AWS CloudHSM
- Thales CipherTrust Manager
- Fortanix DSM
- Autres serveurs KMIP compatibles

**Configuration exemple** :

```yaml
# mongod.conf
security:
  enableEncryption: true
  kmip:
    serverName: kmip.example.com
    port: 5696
    clientCertificateFile: /etc/ssl/mongodb-kmip-client.pem
    serverCAFile: /etc/ssl/kmip-ca.pem
    keyIdentifier: "mongodb-prod-master-key-2024"
```

**Avantages** :
- Standard industriel
- Rotation de clés simplifiée
- Audit centralisé
- Conformité enterprise

#### 2. Local Key File (pour test uniquement)

**⚠️ NE PAS utiliser en production**

```yaml
# mongod.conf - DEV ONLY
security:
  enableEncryption: true
  encryptionKeyFile: /secure/mongodb-keyfile
```

**Limitations** :
- Clé stockée localement (risque de compromission)
- Pas de rotation automatique
- Non conforme aux standards de sécurité
- Backup de la clé problématique

#### 3. Cloud Provider KMS

**Pour** : CSFLE, Queryable Encryption

**Providers supportés** :
- AWS KMS
- Azure Key Vault
- Google Cloud KMS
- MongoDB Atlas (automatique)

**Exemple AWS KMS** :

```javascript
// Configuration du KMS provider pour CSFLE
const kmsProviders = {
  aws: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY
  }
}

const keyVaultNamespace = "encryption.__keyVault"
const masterKey = {
  key: "arn:aws:kms:us-east-1:123456789012:key/abcd1234-a123-456a-a12b-a123b4cd56ef",
  region: "us-east-1"
}
```

### Stratégie de rotation des clés

**Fréquence recommandée** :

| Type de clé | Rotation standard | Rotation PCI-DSS | Rotation HIPAA |
|-------------|-------------------|------------------|----------------|
| CMK (Master Key) | Annuelle | Annuelle | Annuelle |
| DEK (Data Keys) | Trimestrielle | 90 jours | Trimestrielle |
| Certificats TLS | Annuelle | Annuelle | Annuelle |

**Process de rotation Encryption at Rest** :

```bash
# 1. Générer nouvelle clé dans KMIP
# (via interface du KMS)

# 2. Mettre à jour la configuration MongoDB
# mongod.conf
security:
  kmip:
    keyIdentifier: "mongodb-prod-master-key-2025"  # Nouvelle clé

# 3. Rolling restart du Replica Set (un membre à la fois)
# Les données seront re-chiffrées progressivement

# 4. Monitoring de la progression
db.serverStatus().encryptionAtRest
```

**Process de rotation CSFLE** :

```javascript
// Rotation d'une Data Encryption Key
const clientEncryption = new ClientEncryption(client, {
  keyVaultNamespace,
  kmsProviders
})

// Créer une nouvelle DEK
const newKeyId = await clientEncryption.createDataKey('aws', {
  masterKey: {
    key: 'arn:aws:kms:...',  // CMK
    region: 'us-east-1'
  },
  keyAltNames: ['payment-card-key-2025']
})

// Re-chiffrer les données avec la nouvelle clé (opération batch)
await clientEncryption.rewrapManyDataKey({
  filter: { keyAltNames: 'payment-card-key-2024' }
})
```

## Recommandations de production

### Checklist de déploiement sécurisé

#### Niveau 1 : Obligatoire (Toute production)

```
☐ TLS/SSL activé pour toutes les connexions
☐ Certificats valides et non auto-signés
☐ TLS 1.2 minimum (TLS 1.3 recommandé)
☐ Désactivation de TLS 1.0 et 1.1
☐ Authentification activée (SCRAM-SHA-256 minimum)
☐ Pare-feu configuré (uniquement ports nécessaires)
☐ Bind IP restreint (pas de 0.0.0.0 en production)
☐ Logs d'audit activés
```

#### Niveau 2 : Recommandé (Données sensibles)

```
☐ Encryption at Rest activé
☐ KMIP pour la gestion des clés
☐ Rotation des clés planifiée
☐ Backups chiffrés
☐ mTLS (mutual TLS) pour l'authentification
☐ Network encryption entre tous les composants (replica set, sharding)
☐ Monitoring des accès aux clés
☐ Procédure de révocation de certificats
```

#### Niveau 3 : Avancé (Conformité stricte)

```
☐ CSFLE pour les champs hautement sensibles
☐ Queryable Encryption si recherche nécessaire
☐ HSM (Hardware Security Module) pour les clés
☐ Séparation des environnements (CDE pour PCI)
☐ Multi-région avec chiffrement
☐ Zero-trust architecture
☐ Audit SIEM intégré
☐ Tests d'intrusion réguliers
```

### Configuration de référence pour production

```yaml
# mongod.conf - Configuration production sécurisée
# MongoDB Enterprise 7.0+

# Network
net:
  port: 27017
  bindIp: 10.0.1.10  # IP interne, pas 0.0.0.0
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem
    # Désactivation des protocoles faibles
    disabledProtocols: TLS1_0,TLS1_1
    # Cipher suites modernes uniquement
    allowConnectionsWithoutCertificates: false
    # Mutual TLS pour authentification des clients
    allowInvalidCertificates: false
    allowInvalidHostnames: false

# Security
security:
  authorization: enabled
  # Encryption at Rest
  enableEncryption: true
  kmip:
    serverName: kmip.production.company.com
    port: 5696
    clientCertificateFile: /etc/ssl/mongodb/kmip-client.pem
    serverCAFile: /etc/ssl/kmip/ca.pem
    keyIdentifier: "mongodb-prod-2024-q4"
    rotateMasterKey: false
    # Timeout approprié pour environnements cloud
    serverConnectionTimeout: 60000

# Audit
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: '{ $or: [
    { "atype": "authenticate" },
    { "atype": "authCheck", "param.command": { $in: ["find", "insert", "update", "delete"] } },
    { "roles": { $in: ["dbOwner", "userAdmin"] } }
  ]}'

# Storage
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 12
      # Journaling pour durabilité avec chiffrement
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: snappy

# Replication
replication:
  replSetName: prod-rs-01

# System Log
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  verbosity: 0
  component:
    accessControl:
      verbosity: 2
    command:
      verbosity: 1

# Operation Profiling (désactivé en prod, activer temporairement pour debug)
operationProfiling:
  mode: off
  slowOpThresholdMs: 100
```

### Matrice de décision : Quel type de chiffrement ?

```
┌─────────────────────────────────────────────────────────────────────┐
│                     DECISION TREE                                   │
└─────────────────────────────────────────────────────────────────────┘

Q1: Avez-vous besoin de protéger les données en transit ?
    ├─ OUI → TLS/SSL obligatoire
    └─ NON → Réseau 100% isolé ? (rare)

Q2: Avez-vous besoin de protéger les données au repos ?
    ├─ OUI → Q3
    └─ NON → Données publiques uniquement ?

Q3: Niveau de sensibilité des données ?
    ├─ FAIBLE → Encryption at Rest (filesystem/volume)
    ├─ MOYEN → MongoDB Encryption at Rest (WiredTiger)
    └─ ÉLEVÉ → Q4

Q4: Les administrateurs DB doivent-ils pouvoir voir les données ?
    ├─ OUI → Encryption at Rest suffisant
    └─ NON → CSFLE obligatoire

Q5: Besoin de requêter sur les données chiffrées ?
    ├─ OUI → Queryable Encryption (MongoDB 6.0+)
    └─ NON → CSFLE classique

Q6: Conformité réglementaire stricte (PCI-DSS L1, HIPAA) ?
    ├─ OUI → CSFLE + Encryption at Rest + TLS + Audit + HSM
    └─ NON → Évaluation des risques personnalisée
```

### Tableau comparatif des solutions

| Critère | TLS/SSL seul | + Encryption at Rest | + CSFLE | + Queryable Encryption |
|---------|--------------|----------------------|---------|------------------------|
| Protection transit | ✅ | ✅ | ✅ | ✅ |
| Protection repos | ❌ | ✅ | ✅ | ✅ |
| Protection vs admin DB | ❌ | ❌ | ✅ | ✅ |
| Requêtes sur champs chiffrés | N/A | N/A | ❌ | ✅ (limité) |
| Complexité | Faible | Moyenne | Élevée | Très élevée |
| Impact performance | 5-10% | 15-20% | 20-35% | 30-50% |
| Coût licence | Community | Enterprise | Enterprise | Enterprise |
| PCI-DSS compliant | Partiel | ✅ | ✅✅ | ✅✅ |
| HIPAA compliant | Partiel | ✅ | ✅✅ | ✅✅ |

## Monitoring et maintenance du chiffrement

### Vérification de l'état du chiffrement

```javascript
// Vérifier que le chiffrement at rest est actif
db.serverStatus().encryptionAtRest
// Sortie attendue :
// {
//   "encryptionEnabled": true,
//   "keyStore": "KMIP",
//   "keyId": "mongodb-prod-2024-q4"
// }

// Vérifier les connexions TLS
db.serverStatus().connections
// {
//   "current": 52,
//   "available": 51148,
//   "totalCreated": 1829,
//   "active": 3,
//   "threaded": 52,
//   "exhaustIsMaster": 0,
//   "exhaustHello": 0,
//   "awaitingTopologyChanges": 0,
//   "loadBalanced": 0
// }

// Vérifier les clients TLS
db.runCommand({ connectionStatus: 1 })
// {
//   "authInfo": { ... },
//   "authenticatedUsers": [ ... ],
//   "authenticatedUserRoles": [ ... ],
//   "authenticatedUserPrivileges": [ ... ],
//   "ok": 1,
//   "operationTime": Timestamp(1, 1234567890),
//   "$clusterTime": { ... }
// }
```

### Alertes à configurer

**Alertes critiques** :

```javascript
// 1. Échec de connexion au KMIP
// À surveiller dans les logs :
// "Failed to connect to KMIP server"

// 2. Connexion non-TLS détectée
// À bloquer via configuration :
net.tls.mode: requireTLS

// 3. Clé de chiffrement inaccessible
// Arrêt automatique du mongod (comportement par défaut)

// 4. Certificat TLS expiré ou près d'expirer
// Check manuel ou via script :
```

```bash
#!/bin/bash
# check-cert-expiry.sh
CERT_FILE="/etc/ssl/mongodb/server.pem"
DAYS_WARNING=30

EXPIRY=$(openssl x509 -enddate -noout -in "$CERT_FILE" | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

if [ $DAYS_LEFT -lt $DAYS_WARNING ]; then
  echo "WARNING: Certificate expires in $DAYS_LEFT days"
  # Envoyer alerte (email, Slack, PagerDuty, etc.)
fi
```

### Tests de validation post-déploiement

```bash
# 1. Vérifier que TLS est obligatoire
mongosh "mongodb://user:pass@server:27017/?tls=false"
# Doit échouer avec : "connection requires TLS"

# 2. Vérifier la cipher suite négociée
openssl s_client -connect mongodb.prod.company.com:27017 -tls1_2
# Chercher : "Cipher    : ECDHE-RSA-AES256-GCM-SHA384"

# 3. Vérifier que les certificats auto-signés sont rejetés
mongosh "mongodb://server:27017/?tls=true&tlsAllowInvalidCertificates=false"
# Doit échouer si certificat invalide

# 4. Test de rotation de clé (environnement de test)
# Changer keyIdentifier dans mongod.conf
# Redémarrer mongod
# Vérifier que les données sont toujours accessibles

# 5. Test de backup chiffré
mongodump --uri="mongodb://..." --out=/backup/test
file /backup/test/db_name/collection_name.bson
# Vérifier que les données ne sont pas en clair
strings /backup/test/db_name/collection_name.bson | head
# Ne doit pas révéler de données sensibles
```

## Considérations pour architectures distribuées

### Replica Set

**Configuration requise** :

```yaml
# Tous les membres doivent avoir la même configuration de chiffrement
# mongod.conf (identique sur Primary, Secondary, Arbiter)

security:
  enableEncryption: true
  kmip:
    keyIdentifier: "mongodb-rs-prod-2024"  # MÊME CLÉ pour tous les membres

net:
  tls:
    mode: requireTLS
    # Chaque membre a son propre certificat
    certificateKeyFile: /etc/ssl/mongodb/node1.pem  # Spécifique au nœud
    CAFile: /etc/ssl/mongodb/ca.pem  # CA commune
```

**Points d'attention** :
- La clé de chiffrement doit être accessible à tous les membres avant l'ajout au Replica Set
- Lors de l'ajout d'un nouveau membre, synchroniser d'abord les clés
- Les données répliquées via l'oplog sont chiffrées en transit (TLS) mais l'oplog lui-même n'est pas chiffré at rest par défaut

### Sharded Cluster

**Architecture complexe** :

```
Config Servers (Replica Set)
├─ Encryption at Rest : OUI
├─ TLS : OUI (inter-cluster)
└─ Clé KMIP : mongodb-config-key

Shards (Replica Sets)
├─ Shard 1
│  ├─ Encryption at Rest : OUI
│  ├─ Clé KMIP : mongodb-shard1-key
│  └─ TLS : OUI
├─ Shard 2
│  └─ ...

Mongos (Query Routers)
├─ Encryption at Rest : NON (pas de stockage)
└─ TLS : OUI (clients et shards)
```

**Configuration mongos** :

```yaml
# mongos.conf
net:
  port: 27017
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/mongos.pem
    CAFile: /etc/ssl/mongodb/ca.pem

sharding:
  configDB: configRS/cfg1:27019,cfg2:27019,cfg3:27019

security:
  clusterAuthMode: x509  # Authentification inter-composants
  keyFile: /etc/mongodb/cluster-keyfile  # Fallback
```

**Recommandations** :
- Utiliser des clés différentes par shard pour limiter le blast radius en cas de compromission
- Centraliser la gestion des certificats (HashiCorp Vault, cert-manager)
- Automatiser le renouvellement des certificats (Let's Encrypt pour dev, CA enterprise pour prod)

## Conclusion

Le chiffrement dans MongoDB est une discipline à plusieurs facettes qui nécessite une approche méthodique et adaptée aux besoins spécifiques de chaque organisation. Les trois piliers — chiffrement en transit, au repos, et au niveau des champs — offrent une protection en profondeur contre différents vecteurs d'attaque.

**Points clés à retenir** :

1. **TLS/SSL est non-négociable** : Toute installation de production doit avoir TLS activé, sans exception.

2. **Encryption at Rest pour les données sensibles** : Obligatoire pour la conformité réglementaire et la protection contre le vol physique.

3. **CSFLE pour le modèle zero-trust** : Quand les administrateurs de base de données ne doivent pas avoir accès aux données sensibles.

4. **La performance a un coût** : Évaluer l'impact et dimensionner l'infrastructure en conséquence.

5. **La gestion des clés est critique** : Utiliser un KMS professionnel, ne jamais stocker les clés avec les données.

6. **Tester régulièrement** : Les procédures de rotation de clés, de restauration, et de révocation doivent être testées en conditions réelles.

Les sections suivantes détailleront l'implémentation technique de chaque type de chiffrement avec des exemples concrets et des cas d'usage spécifiques.

---

**Prochaines sections** :
- 11.5.1 Chiffrement en transit (TLS/SSL)
- 11.5.2 Chiffrement au repos (Encryption at Rest)
- 11.5.3 Client-Side Field Level Encryption (CSFLE)
- 11.5.4 Queryable Encryption

⏭️ [Chiffrement en transit (TLS/SSL)](/11-securite/05.1-chiffrement-transit.md)
