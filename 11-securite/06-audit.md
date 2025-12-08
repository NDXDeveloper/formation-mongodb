🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 11.6 Audit

## Introduction

L'audit dans MongoDB permet d'enregistrer de manière détaillée toutes les activités sur le système de base de données : qui a accédé à quelles données, quand, depuis où, et quelle opération a été effectuée. Cette traçabilité est essentielle pour la sécurité, la conformité réglementaire, le troubleshooting, et la détection d'anomalies.

**L'audit MongoDB répond à plusieurs besoins critiques** :

```
Conformité réglementaire
├─ RGPD : Traçabilité des accès aux données personnelles
├─ HIPAA : Audit des accès aux PHI (Protected Health Information)
├─ PCI-DSS : Logging de tous les accès aux données de cartes
├─ SOX : Audit des modifications de données financières
└─ ISO 27001 : Exigences de logging et monitoring

Sécurité
├─ Détection d'intrusions et d'activités suspectes
├─ Investigation post-incident (forensics)
├─ Identification des tentatives d'accès non autorisées
└─ Alerting sur comportements anormaux

Opérationnel
├─ Troubleshooting de problèmes de permissions
├─ Analyse des performances d'accès
├─ Identification des requêtes problématiques
└─ Validation des changements de configuration
```

### Disponibilité

**⚠️ Important** : L'audit est une fonctionnalité **MongoDB Enterprise uniquement**.

```
MongoDB Community : ❌ Pas d'audit natif
MongoDB Enterprise : ✅ Audit complet
MongoDB Atlas : ✅ Audit inclus (interface graphique)
```

## Architecture de l'audit

### Flux d'audit

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Client envoie une commande                                  │
│     db.users.find({ ssn: "123-45-6789" })                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  2. MongoDB reçoit et authentifie                               │
│     • Vérifie les credentials                                   │
│     • Vérifie les permissions (authCheck)                       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  3. Audit Subsystem (si configuré)                              │
│     • Capture l'événement selon les filtres                     │
│     • Génère un document d'audit JSON                           │
│     • Timestamp, user, action, result                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  4. Écriture dans la destination configurée                     │
│     ├─ File : /var/log/mongodb/audit.json                       │
│     ├─ Syslog : rsyslog/syslog-ng                               │
│     └─ Console : stdout (conteneurs Docker)                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  5. SIEM / Log Analysis (optionnel)                             │
│     • Splunk, ELK Stack, Datadog                                │
│     • Alerting et dashboards                                    │
│     • Corrélation d'événements                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Structure d'un événement d'audit

```json
{
  "atype": "authCheck",
  "ts": { "$date": "2024-12-08T10:30:45.123Z" },
  "uuid": { "$binary": "aGVsbG8gd29ybGQ=", "$type": "04" },
  "local": {
    "ip": "10.0.1.50",
    "port": 27017
  },
  "remote": {
    "ip": "192.168.1.100",
    "port": 54321
  },
  "users": [
    {
      "user": "appUser",
      "db": "admin"
    }
  ],
  "roles": [
    {
      "role": "readWrite",
      "db": "hospital"
    }
  ],
  "param": {
    "command": "find",
    "ns": "hospital.patients",
    "args": {
      "find": "patients",
      "filter": { "ssn": "***REDACTED***" },
      "lsid": { "id": { "$binary": "..." } }
    }
  },
  "result": 0
}
```

**Champs importants** :

```
atype : Type d'action auditée
├─ authenticate : Tentative d'authentification
├─ authCheck : Vérification des permissions
├─ createUser : Création d'utilisateur
├─ dropUser : Suppression d'utilisateur
├─ createCollection : Création de collection
├─ dropCollection : Suppression de collection
└─ ... (voir liste complète plus bas)

ts : Timestamp de l'événement (ISO 8601)

users : Utilisateur(s) effectuant l'action

roles : Rôle(s) de l'utilisateur

param : Détails de l'opération
├─ command : Commande exécutée
├─ ns : Namespace (database.collection)
└─ args : Arguments de la commande

result : Résultat (0 = succès, >0 = erreur)

local : Serveur MongoDB (IP:port)

remote : Client (IP:port)
```

## Configuration de l'audit

### Configuration minimale (fichier)

```yaml
# mongod.conf - Configuration audit de base
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
```

**Démarrage** :

```bash
# Redémarrer MongoDB pour activer l'audit
systemctl restart mongod

# Vérifier que l'audit fonctionne
tail -f /var/log/mongodb/audit.json
```

### Configuration complète (production)

```yaml
# mongod.conf - Configuration production
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true

systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  verbosity: 0

# Configuration réseau et sécurité
net:
  port: 27017
  bindIp: 10.0.1.10
  tls:
    mode: requireTLS
    certificateKeyFile: /etc/ssl/mongodb/server.pem
    CAFile: /etc/ssl/mongodb/ca.pem

security:
  authorization: enabled
  clusterAuthMode: x509

# ============ AUDIT CONFIGURATION ============
auditLog:
  # Destination : file, syslog, ou console
  destination: file

  # Format : JSON (recommandé) ou BSON
  format: JSON

  # Chemin du fichier d'audit
  path: /var/log/mongodb/audit.json

  # Filtre d'audit (voir section suivante pour exemples détaillés)
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        { atype: "createUser" },
        { atype: "dropUser" },
        { atype: "createCollection" },
        { atype: "dropCollection" },
        { atype: "dropDatabase" },
        {
          atype: "authCheck",
          "param.command": {
            $in: ["find", "insert", "update", "delete", "aggregate"]
          }
        }
      ]
    }

replication:
  replSetName: rs1
```

### Configuration avec Syslog

```yaml
# mongod.conf - Audit vers syslog
auditLog:
  destination: syslog
  format: JSON
  filter: '{ atype: { $in: ["authenticate", "authCheck"] } }'
```

**Configuration rsyslog** :

```bash
# /etc/rsyslog.d/30-mongodb-audit.conf
# Recevoir les logs MongoDB audit sur un fichier dédié

# Template pour formater les logs MongoDB
template(name="MongoAuditFormat" type="string"
  string="%TIMESTAMP:::date-rfc3339% %HOSTNAME% mongodb-audit: %msg%\n"
)

# Filtrer et rediriger les logs MongoDB audit
if $programname == 'mongod' and $msg contains '"atype"' then {
  action(type="omfile" file="/var/log/mongodb/audit-syslog.log" template="MongoAuditFormat")
  stop
}
```

**Redémarrage** :

```bash
# Redémarrer rsyslog
systemctl restart rsyslog

# Redémarrer MongoDB
systemctl restart mongod

# Vérifier
tail -f /var/log/mongodb/audit-syslog.log
```

### Configuration pour conteneurs (console)

```yaml
# mongod.conf - Pour Docker/Kubernetes
auditLog:
  destination: console
  format: JSON
  filter: '{ atype: "authenticate" }'

systemLog:
  destination: file
  path: /dev/stdout
  logAppend: true
```

**Docker Compose** :

```yaml
version: '3.8'

services:
  mongodb:
    image: mongo:7.0-enterprise
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD}
    volumes:
      - ./mongod.conf:/etc/mongod.conf:ro
      - mongodb_data:/data/db
    command: mongod --config /etc/mongod.conf
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  mongodb_data:
```

## Types d'événements auditables

### Catégories d'événements

```javascript
// 1. Authentification
atype: "authenticate"
├─ Tentatives de connexion (succès et échecs)
├─ Mécanisme utilisé (SCRAM, x509, LDAP, etc.)
└─ Source de la connexion (IP:port)

// 2. Autorisation
atype: "authCheck"
├─ Vérification de permissions avant chaque opération
├─ Commande tentée (find, insert, update, delete, etc.)
├─ Résultat (autorisé ou refusé)
└─ Collection/database ciblée

// 3. Gestion des utilisateurs
atype: "createUser", "updateUser", "dropUser"
├─ Création/modification/suppression d'utilisateurs
├─ Utilisateur effectuant l'action
└─ Utilisateur cible

// 4. Gestion des rôles
atype: "createRole", "updateRole", "dropRole", "grantRolesToUser", "revokeRolesFromUser"
├─ Création/modification/suppression de rôles
├─ Attribution/révocation de rôles
└─ Privilèges modifiés

// 5. Opérations sur les données (DDL)
atype: "createCollection", "dropCollection", "createIndex", "dropIndex", "dropDatabase"
├─ Modifications de structure
├─ Création/suppression de collections
└─ Gestion des index

// 6. Opérations sur les données (DML)
atype: "insert", "update", "remove"
├─ Modifications de données
├─ Documents affectés
└─ Critères de sélection

// 7. Configuration système
atype: "serverConfiguration"
├─ Modifications de configuration
├─ Paramètres changés
└─ Redémarrages

// 8. Replica Set / Sharding
atype: "replSetReconfig", "shardCollection", "addShard", "removeShard"
├─ Changements de topologie
├─ Configuration de réplication
└─ Opérations de sharding

// 9. Shutdown
atype: "shutdown"
├─ Arrêt du serveur
└─ Raison de l'arrêt
```

### Liste complète des atypes

```javascript
const auditTypes = {
  // Authentification et autorisation
  authenticate: "Tentative d'authentification",
  authCheck: "Vérification d'autorisation",
  logout: "Déconnexion utilisateur",

  // Gestion des utilisateurs
  createUser: "Création d'utilisateur",
  updateUser: "Modification d'utilisateur",
  dropUser: "Suppression d'utilisateur",
  dropAllUsersFromDatabase: "Suppression de tous les utilisateurs d'une DB",
  grantRolesToUser: "Attribution de rôles à un utilisateur",
  revokeRolesFromUser: "Révocation de rôles d'un utilisateur",

  // Gestion des rôles
  createRole: "Création de rôle",
  updateRole: "Modification de rôle",
  dropRole: "Suppression de rôle",
  dropAllRolesFromDatabase: "Suppression de tous les rôles d'une DB",
  grantPrivilegesToRole: "Attribution de privilèges à un rôle",
  revokePrivilegesFromRole: "Révocation de privilèges d'un rôle",
  grantRolesToRole: "Attribution de rôles à un rôle",
  revokeRolesFromRole: "Révocation de rôles d'un rôle",

  // DDL - Data Definition Language
  createCollection: "Création de collection",
  dropCollection: "Suppression de collection",
  renameCollection: "Renommage de collection",
  createIndex: "Création d'index",
  createIndexes: "Création d'index multiples",
  dropIndex: "Suppression d'index",
  createDatabase: "Création de base de données",
  dropDatabase: "Suppression de base de données",

  // DML - Data Manipulation Language (si activé)
  insert: "Insertion de documents",
  update: "Mise à jour de documents",
  remove: "Suppression de documents",

  // Replica Set
  replSetReconfig: "Reconfiguration du Replica Set",

  // Sharding
  shardCollection: "Activation du sharding sur une collection",
  addShard: "Ajout d'un shard",
  removeShard: "Suppression d'un shard",
  enableSharding: "Activation du sharding sur une DB",

  // Système
  shutdown: "Arrêt du serveur",
  serverConfiguration: "Modification de configuration serveur",
  applicationMessage: "Message applicatif personnalisé"
};
```

## Filtres d'audit

### Syntaxe des filtres

Les filtres d'audit utilisent la même syntaxe que les requêtes MongoDB :

```javascript
// Filtre simple : Auditer uniquement les authentifications
{ atype: "authenticate" }

// Filtre avec $or : Plusieurs types d'événements
{
  $or: [
    { atype: "authenticate" },
    { atype: "authCheck" }
  ]
}

// Filtre avec conditions : Événements spécifiques
{
  atype: "authCheck",
  "param.command": "delete",
  "param.ns": "hospital.patients"
}

// Filtre complexe : Combiner plusieurs critères
{
  $or: [
    { atype: "authenticate", result: { $ne: 0 } },  // Échecs d'auth
    {
      atype: "authCheck",
      "param.command": { $in: ["delete", "drop"] },
      result: 0  // Succès uniquement
    }
  ]
}
```

### Filtres par cas d'usage

#### 1. Conformité PCI-DSS

Auditer tous les accès aux données de cartes bancaires :

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-pci.json
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        { atype: "createUser" },
        { atype: "dropUser" },
        { atype: "updateUser" },
        { atype: "grantRolesToUser" },
        { atype: "revokeRolesFromUser" },
        {
          atype: "authCheck",
          "param.ns": { $regex: "^payment\\.(cards|transactions)" }
        },
        {
          atype: "authCheck",
          "param.command": { $in: ["find", "aggregate"] },
          "param.args.filter.creditCard": { $exists: true }
        }
      ]
    }
```

#### 2. Conformité HIPAA

Auditer tous les accès aux données de santé :

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-hipaa.json
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        {
          atype: "authCheck",
          "param.ns": { $regex: "^(hospital|clinic)\\." }
        },
        {
          atype: "authCheck",
          "param.command": { $in: ["find", "aggregate", "update", "delete"] },
          "param.ns": "hospital.patients"
        },
        { atype: "createUser" },
        { atype: "dropUser" },
        { atype: "grantRolesToUser" }
      ]
    }
```

#### 3. Sécurité - Détection d'intrusion

Auditer les activités suspectes :

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-security.json
  filter: |
    {
      $or: [
        { atype: "authenticate", result: { $ne: 0 } },
        { atype: "authCheck", result: { $ne: 0 } },
        {
          atype: "authCheck",
          "param.command": { $in: ["drop", "dropDatabase", "dropCollection"] }
        },
        {
          atype: "authCheck",
          "param.command": "delete",
          result: 0
        },
        { atype: "dropUser" },
        { atype: "grantRolesToUser", "param.roles.role": "root" },
        { atype: "shutdown" }
      ]
    }
```

#### 4. Opérations privilégiées uniquement

Auditer uniquement les opérations sensibles :

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-privileged.json
  filter: |
    {
      $or: [
        { atype: { $in: ["createUser", "dropUser", "updateUser"] } },
        { atype: { $in: ["createRole", "dropRole", "updateRole"] } },
        { atype: "grantRolesToUser" },
        { atype: "revokeRolesFromUser" },
        { atype: { $in: ["dropDatabase", "dropCollection"] } },
        {
          atype: "authCheck",
          "param.command": { $in: ["shutdown", "fsync"] }
        },
        { atype: "replSetReconfig" },
        { atype: { $in: ["addShard", "removeShard"] } }
      ]
    }
```

#### 5. Utilisateur spécifique

Auditer les actions d'un utilisateur particulier :

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-user.json
  filter: |
    {
      "users.user": "suspiciousUser"
    }
```

#### 6. Base de données spécifique

Auditer toutes les opérations sur une base critique :

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-production-db.json
  filter: |
    {
      atype: "authCheck",
      "param.ns": { $regex: "^production\\." }
    }
```

#### 7. Échecs d'authentification (brute force detection)

```yaml
# mongod.conf
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-auth-failures.json
  filter: |
    {
      atype: "authenticate",
      result: { $ne: 0 }
    }
```

#### 8. Modifications de données (DML)

```yaml
# mongod.conf - ⚠️ Volume élevé, utiliser avec précaution
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit-dml.json
  filter: |
    {
      atype: "authCheck",
      "param.command": { $in: ["insert", "update", "delete"] },
      "param.ns": { $regex: "^(production|financial)\\." }
    }
```

### Filtres à éviter (volume trop élevé)

```yaml
# ❌ MAUVAIS : Auditer TOUS les authCheck
# Génère des millions d'événements par jour
auditLog:
  filter: '{ atype: "authCheck" }'

# ❌ MAUVAIS : Auditer toutes les lectures
# Impact performance majeur
auditLog:
  filter: |
    {
      atype: "authCheck",
      "param.command": { $in: ["find", "aggregate"] }
    }

# ❌ MAUVAIS : Pas de filtre (tout auditer)
# Consommation disque énorme
auditLog:
  filter: '{}'
```

## Analyse des logs d'audit

### Parsing et requêtes avec jq

```bash
# Extraire les échecs d'authentification
cat /var/log/mongodb/audit.json | jq 'select(.atype == "authenticate" and .result != 0)'

# Compter les tentatives d'auth par utilisateur
cat /var/log/mongodb/audit.json | \
  jq -r 'select(.atype == "authenticate") | .users[0].user' | \
  sort | uniq -c | sort -rn

# Trouver les suppressions de collections
cat /var/log/mongodb/audit.json | \
  jq 'select(.atype == "dropCollection")'

# Identifier les sources d'échecs d'authentification
cat /var/log/mongodb/audit.json | \
  jq -r 'select(.atype == "authenticate" and .result != 0) | .remote.ip' | \
  sort | uniq -c | sort -rn

# Extraire les opérations d'un utilisateur spécifique
cat /var/log/mongodb/audit.json | \
  jq 'select(.users[0].user == "appUser")'

# Timeline des événements d'un jour spécifique
cat /var/log/mongodb/audit.json | \
  jq -r 'select(.ts."$date" | startswith("2024-12-08")) | "\(.ts."$date") - \(.atype) - \(.users[0].user)"'
```

### Analyse avec MongoDB

Importer les logs d'audit dans MongoDB pour analyse :

```bash
# Importer dans une collection
mongoimport --uri="mongodb://localhost:27017/audit" \
  --collection=events \
  --file=/var/log/mongodb/audit.json

# Créer des index pour les requêtes fréquentes
mongosh <<EOF
use audit

db.events.createIndex({ atype: 1, "ts.\$date": -1 })
db.events.createIndex({ "users.user": 1, "ts.\$date": -1 })
db.events.createIndex({ "param.ns": 1, "ts.\$date": -1 })
db.events.createIndex({ "remote.ip": 1, "ts.\$date": -1 })
EOF
```

**Requêtes d'analyse** :

```javascript
// 1. Top 10 des utilisateurs les plus actifs
db.events.aggregate([
  { $match: { atype: "authCheck" } },
  { $group: {
      _id: "$users.user",
      count: { $sum: 1 }
  }},
  { $sort: { count: -1 } },
  { $limit: 10 }
])

// 2. Échecs d'authentification par jour
db.events.aggregate([
  {
    $match: {
      atype: "authenticate",
      result: { $ne: 0 }
    }
  },
  {
    $group: {
      _id: {
        $dateToString: {
          format: "%Y-%m-%d",
          date: "$ts.$date"
        }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: -1 } }
])

// 3. Détection de brute force (> 10 échecs en 5 min)
db.events.aggregate([
  {
    $match: {
      atype: "authenticate",
      result: { $ne: 0 },
      "ts.$date": {
        $gte: ISODate("2024-12-08T10:00:00Z"),
        $lt: ISODate("2024-12-08T10:05:00Z")
      }
    }
  },
  {
    $group: {
      _id: "$remote.ip",
      attempts: { $sum: 1 }
    }
  },
  { $match: { attempts: { $gte: 10 } } },
  { $sort: { attempts: -1 } }
])

// 4. Suppressions de collections (audit trail)
db.events.find({
  atype: "dropCollection"
}).sort({ "ts.$date": -1 })

// 5. Modifications de privilèges
db.events.find({
  atype: { $in: ["grantRolesToUser", "revokeRolesFromUser"] }
}).sort({ "ts.$date": -1 })

// 6. Activité par heure de la journée
db.events.aggregate([
  {
    $group: {
      _id: { $hour: "$ts.$date" },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])

// 7. Commandes les plus fréquentes
db.events.aggregate([
  { $match: { atype: "authCheck" } },
  {
    $group: {
      _id: "$param.command",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } }
])
```

## Intégration SIEM

### ELK Stack (Elasticsearch, Logstash, Kibana)

**Logstash configuration** :

```ruby
# /etc/logstash/conf.d/mongodb-audit.conf
input {
  file {
    path => "/var/log/mongodb/audit.json"
    start_position => "beginning"
    codec => "json"
    type => "mongodb-audit"
  }
}

filter {
  if [type] == "mongodb-audit" {
    # Parser le timestamp MongoDB
    date {
      match => ["ts.$date", "ISO8601"]
      target => "@timestamp"
    }

    # Extraire les champs importants
    mutate {
      add_field => {
        "action_type" => "%{atype}"
        "user" => "%{[users][0][user]}"
        "database" => "%{[users][0][db]}"
        "remote_ip" => "%{[remote][ip]}"
        "command" => "%{[param][command]}"
        "namespace" => "%{[param][ns]}"
        "result_code" => "%{result}"
      }
    }

    # Enrichir avec géolocalisation IP
    geoip {
      source => "remote_ip"
      target => "geoip"
    }

    # Classifier les événements
    if [result] != 0 {
      mutate {
        add_tag => ["failure"]
      }
    }

    if [atype] == "authenticate" and [result] != 0 {
      mutate {
        add_tag => ["auth_failure", "security_alert"]
      }
    }

    if [atype] in ["dropCollection", "dropDatabase", "dropUser"] {
      mutate {
        add_tag => ["critical_operation"]
      }
    }
  }
}

output {
  if [type] == "mongodb-audit" {
    elasticsearch {
      hosts => ["localhost:9200"]
      index => "mongodb-audit-%{+YYYY.MM.dd}"
    }
  }
}
```

**Kibana Dashboard Queries** :

```
# Échecs d'authentification dans les dernières 24h
action_type:authenticate AND result_code:(NOT 0) AND @timestamp:[now-24h TO now]

# Opérations sensibles
action_type:(dropCollection OR dropDatabase OR dropUser OR grantRolesToUser)

# Activité d'un utilisateur spécifique
user:"adminUser" AND @timestamp:[now-7d TO now]

# Sources d'échecs d'auth (top IPs)
action_type:authenticate AND result_code:(NOT 0)
Aggregation: Terms on remote_ip

# Volume d'événements par type
Aggregation: Terms on action_type
```

### Splunk

**Splunk inputs.conf** :

```ini
# /opt/splunk/etc/apps/mongodb_audit/local/inputs.conf
[monitor:///var/log/mongodb/audit.json]
disabled = false
index = mongodb_audit
sourcetype = mongodb:audit
```

**Splunk props.conf** :

```ini
# /opt/splunk/etc/apps/mongodb_audit/local/props.conf
[mongodb:audit]
SHOULD_LINEMERGE = false
LINE_BREAKER = ([\r\n]+)
TIME_PREFIX = "ts":\{"\\$date":"
TIME_FORMAT = %Y-%m-%dT%H:%M:%S.%3N%Z
MAX_TIMESTAMP_LOOKAHEAD = 32
TRUNCATE = 0

# Extractions de champs
EXTRACT-atype = "atype":"(?<action_type>[^"]+)"
EXTRACT-user = "user":"(?<user>[^"]+)"
EXTRACT-remote_ip = "remote":\{[^}]*"ip":"(?<remote_ip>[^"]+)"
EXTRACT-result = "result":(?<result_code>\d+)
EXTRACT-command = "command":"(?<command>[^"]+)"
EXTRACT-namespace = "ns":"(?<namespace>[^"]+)"
```

**Splunk Searches** :

```spl
# Échecs d'authentification
index=mongodb_audit action_type=authenticate result_code!=0
| stats count by user, remote_ip
| sort -count

# Timeline des événements
index=mongodb_audit
| timechart span=1h count by action_type

# Détection de brute force
index=mongodb_audit action_type=authenticate result_code!=0
| bin _time span=5m
| stats count by _time, remote_ip
| where count > 10

# Opérations administratives
index=mongodb_audit action_type IN (createUser, dropUser, grantRolesToUser, dropDatabase)
| table _time, action_type, user, namespace

# Alerting : Opération sensible détectée
index=mongodb_audit action_type IN (dropDatabase, dropCollection)
| eval alert_message="CRITICAL: " + action_type + " by " + user + " on " + namespace
| table _time, alert_message, user, remote_ip
```

### Datadog

**Datadog Agent configuration** :

```yaml
# /etc/datadog-agent/conf.d/logs.d/mongodb_audit.yaml
logs:
  - type: file
    path: /var/log/mongodb/audit.json
    service: mongodb
    source: mongodb-audit
    sourcecategory: database
    tags:
      - env:production
      - cluster:rs1
```

**Datadog Log Processing Pipeline** :

```json
{
  "type": "pipeline",
  "name": "MongoDB Audit Processing",
  "filter": {
    "query": "source:mongodb-audit"
  },
  "processors": [
    {
      "type": "date-remapper",
      "sources": ["ts.$date"],
      "target": "@timestamp"
    },
    {
      "type": "attribute-remapper",
      "sources": ["atype"],
      "target": "action.type"
    },
    {
      "type": "attribute-remapper",
      "sources": ["users[0].user"],
      "target": "usr.name"
    },
    {
      "type": "attribute-remapper",
      "sources": ["remote.ip"],
      "target": "network.client.ip"
    },
    {
      "type": "status-remapper",
      "sources": ["result"]
    }
  ]
}
```

## Rotation et rétention

### Rotation des logs avec logrotate

```bash
# /etc/logrotate.d/mongodb-audit
/var/log/mongodb/audit.json {
    daily
    rotate 90
    dateext
    dateformat -%Y%m%d
    compress
    delaycompress
    notifempty
    create 0640 mongodb mongodb
    sharedscripts
    postrotate
        # Envoyer SIGUSR1 pour que MongoDB reouvre le fichier
        /bin/kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

**Explication** :

```
daily : Rotation quotidienne
rotate 90 : Conserver 90 jours (requis pour PCI-DSS)
dateext : Ajouter la date au nom du fichier
compress : Compresser les anciens logs
delaycompress : Ne pas compresser le plus récent (pour debug)
create 0640 : Permissions restrictives
postrotate : Signaler à MongoDB de reouvrir le fichier
```

### Archivage vers stockage froid

```bash
#!/bin/bash
# /usr/local/bin/archive-mongodb-audit.sh

# Archiver les logs d'audit vers S3 après 90 jours
# Exécuter via cron : 0 2 * * * /usr/local/bin/archive-mongodb-audit.sh

RETENTION_DAYS=90
ARCHIVE_DIR="/var/log/mongodb/audit-archive"
LOG_DIR="/var/log/mongodb"
S3_BUCKET="s3://company-audit-logs/mongodb"

# Trouver les fichiers à archiver
find "$LOG_DIR" -name "audit.json-*" -mtime +$RETENTION_DAYS -type f | while read file; do
    BASENAME=$(basename "$file")
    YEAR=$(echo "$BASENAME" | grep -oP '\d{4}')
    MONTH=$(echo "$BASENAME" | grep -oP '\d{4}\d{2}' | tail -c 3)

    # Uploader vers S3 avec chiffrement
    aws s3 cp "$file" "${S3_BUCKET}/${YEAR}/${MONTH}/${BASENAME}" \
        --storage-class GLACIER \
        --server-side-encryption AES256

    # Vérifier l'upload
    if [ $? -eq 0 ]; then
        echo "Archived: $file to S3"
        # Supprimer le fichier local après upload réussi
        rm "$file"
    else
        echo "ERROR: Failed to archive $file" >&2
    fi
done
```

**Cron job** :

```bash
# Archivage quotidien à 2h du matin
0 2 * * * /usr/local/bin/archive-mongodb-audit.sh >> /var/log/audit-archive.log 2>&1
```

### Politique de rétention par conformité

```
┌─────────────────────────────────────────────────────────────────┐
│  Stockage local (SSD/NVMe rapide)                               │
│  • Durée : 7 jours                                              │
│  • Accès : Immédiat                                             │
│  • Coût : Élevé                                                 │
│  • Usage : Investigation temps réel                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Stockage local compressé                                       │
│  • Durée : 7-90 jours                                           │
│  • Accès : Rapide (quelques secondes)                           │
│  • Coût : Moyen                                                 │
│  • Usage : Audit régulier, conformité                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  S3 Standard                                                    │
│  • Durée : 90 jours - 1 an                                      │
│  • Accès : Quelques minutes                                     │
│  • Coût : Faible                                                │
│  • Usage : Conformité réglementaire (PCI, HIPAA)                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  S3 Glacier / Deep Archive                                      │
│  • Durée : 1-7 ans (selon réglementation)                       │
│  • Accès : Plusieurs heures                                     │
│  • Coût : Très faible                                           │
│  • Usage : Archivage légal, investigations forensiques          │
└─────────────────────────────────────────────────────────────────┘
```

**Exigences par standard** :

```
PCI-DSS : 1 an minimum (3 mois online, 9 mois archive)
HIPAA : 6 ans minimum
SOX : 7 ans minimum
RGPD : Variable selon le contexte (généralement 1-3 ans)
```

## Monitoring et alerting

### Script de monitoring

```python
#!/usr/bin/env python3
# /usr/local/bin/monitor-audit-logs.py

import json
import sys
from datetime import datetime, timedelta
from collections import Counter

AUDIT_LOG = '/var/log/mongodb/audit.json'
ALERT_THRESHOLD_AUTH_FAILURES = 10  # par IP en 5 minutes
ALERT_THRESHOLD_PRIVILEGED_OPS = 5   # opérations sensibles en 1 heure

def analyze_audit_log():
    """Analyse les logs d'audit et génère des alertes"""

    recent_events = []
    now = datetime.utcnow()
    five_min_ago = now - timedelta(minutes=5)
    one_hour_ago = now - timedelta(hours=1)

    # Lire les événements récents
    with open(AUDIT_LOG, 'r') as f:
        for line in f:
            try:
                event = json.loads(line)
                event_time = datetime.fromisoformat(
                    event['ts']['$date'].replace('Z', '+00:00')
                )

                if event_time > one_hour_ago:
                    recent_events.append(event)
            except (json.JSONDecodeError, KeyError):
                continue

    alerts = []

    # Détection 1 : Brute force (échecs d'auth répétés)
    auth_failures = [
        e for e in recent_events
        if e.get('atype') == 'authenticate'
        and e.get('result') != 0
        and datetime.fromisoformat(e['ts']['$date'].replace('Z', '+00:00')) > five_min_ago
    ]

    if auth_failures:
        ip_counts = Counter(e['remote']['ip'] for e in auth_failures)
        for ip, count in ip_counts.items():
            if count >= ALERT_THRESHOLD_AUTH_FAILURES:
                alerts.append({
                    'severity': 'HIGH',
                    'type': 'BRUTE_FORCE',
                    'message': f'Potential brute force attack from {ip}',
                    'details': f'{count} failed authentication attempts in 5 minutes',
                    'ip': ip
                })

    # Détection 2 : Opérations privilégiées suspectes
    privileged_ops = [
        e for e in recent_events
        if e.get('atype') in ['dropDatabase', 'dropCollection', 'dropUser', 'grantRolesToUser']
    ]

    if len(privileged_ops) >= ALERT_THRESHOLD_PRIVILEGED_OPS:
        alerts.append({
            'severity': 'MEDIUM',
            'type': 'PRIVILEGED_OPERATIONS',
            'message': f'{len(privileged_ops)} privileged operations in the last hour',
            'details': [e.get('atype') for e in privileged_ops]
        })

    # Détection 3 : Accès depuis IP inhabituelle
    # (nécessite une liste d'IPs autorisées)

    # Afficher les alertes
    if alerts:
        print(f"\n{'='*60}")
        print(f"SECURITY ALERTS - {now.isoformat()}")
        print(f"{'='*60}\n")

        for alert in alerts:
            print(f"[{alert['severity']}] {alert['type']}")
            print(f"  {alert['message']}")
            print(f"  Details: {alert['details']}")
            print()

        return 1
    else:
        print(f"No alerts - {now.isoformat()}")
        return 0

if __name__ == '__main__':
    sys.exit(analyze_audit_log())
```

**Exécution périodique** :

```bash
# Cron : Toutes les 5 minutes
*/5 * * * * /usr/local/bin/monitor-audit-logs.py | /usr/local/bin/send-to-slack.sh
```

### Intégration avec PagerDuty

```python
#!/usr/bin/env python3
# /usr/local/bin/audit-to-pagerduty.py

import json
import requests
import sys
from datetime import datetime

PAGERDUTY_API_KEY = 'your-api-key-here'
PAGERDUTY_SERVICE_ID = 'your-service-id'

def send_pagerduty_alert(severity, message, details):
    """Envoie une alerte PagerDuty"""

    payload = {
        'routing_key': PAGERDUTY_API_KEY,
        'event_action': 'trigger',
        'payload': {
            'summary': message,
            'severity': severity,
            'source': 'mongodb-audit-monitor',
            'timestamp': datetime.utcnow().isoformat(),
            'custom_details': details
        }
    }

    response = requests.post(
        'https://events.pagerduty.com/v2/enqueue',
        json=payload
    )

    if response.status_code == 202:
        print(f"Alert sent to PagerDuty: {message}")
    else:
        print(f"Failed to send alert: {response.text}", file=sys.stderr)

# Exemple d'utilisation
if __name__ == '__main__':
    send_pagerduty_alert(
        'critical',
        'MongoDB Audit: Multiple failed authentication attempts',
        {
            'source_ip': '192.168.1.100',
            'attempts': 15,
            'time_window': '5 minutes'
        }
    )
```

## Troubleshooting

### Audit log ne s'écrit pas

```bash
# 1. Vérifier que MongoDB Enterprise est installé
mongod --version | grep -i enterprise

# 2. Vérifier les permissions du fichier
ls -l /var/log/mongodb/audit.json
# Doit être : mongodb:mongodb avec permissions 640

# 3. Créer le fichier si nécessaire
sudo touch /var/log/mongodb/audit.json
sudo chown mongodb:mongodb /var/log/mongodb/audit.json
sudo chmod 640 /var/log/mongodb/audit.json

# 4. Vérifier la configuration
sudo grep -A 5 "auditLog" /etc/mongod.conf

# 5. Vérifier les logs MongoDB pour des erreurs
sudo tail -f /var/log/mongodb/mongod.log | grep -i audit

# 6. Redémarrer MongoDB
sudo systemctl restart mongod

# 7. Tester en générant un événement
mongosh --eval "db.auth('testuser', 'wrongpassword')"

# 8. Vérifier que l'événement est loggé
sudo tail -1 /var/log/mongodb/audit.json | jq .
```

### Audit log trop volumineux

```bash
# 1. Vérifier la taille actuelle
du -sh /var/log/mongodb/audit.json

# 2. Identifier les événements les plus fréquents
cat /var/log/mongodb/audit.json | jq -r .atype | sort | uniq -c | sort -rn

# 3. Affiner le filtre pour réduire le volume
# Exemple : Exclure les requêtes read-only
auditLog:
  filter: |
    {
      $and: [
        { atype: { $ne: "authCheck" } },
        {
          $or: [
            { atype: "authenticate" },
            { atype: { $in: ["createUser", "dropUser", "dropCollection"] } }
          ]
        }
      ]
    }

# 4. Augmenter la fréquence de rotation
# Dans /etc/logrotate.d/mongodb-audit : changer de daily à hourly

# 5. Compresser immédiatement
gzip /var/log/mongodb/audit.json.old
```

### Performance impactée par l'audit

```bash
# 1. Vérifier l'overhead I/O
iostat -x 5

# 2. Déplacer l'audit sur un disque dédié
# Éditer mongod.conf
auditLog:
  path: /audit-disk/mongodb/audit.json

# 3. Utiliser syslog pour déporter l'écriture
auditLog:
  destination: syslog

# 4. Simplifier le filtre d'audit
# Auditer uniquement les événements critiques

# 5. Augmenter la mémoire buffer de syslog
# /etc/rsyslog.conf
$MainMsgQueueSize 100000

# 6. Considérer l'audit asynchrone via queue
```

### Filtre d'audit ne fonctionne pas

```javascript
// Vérifier la syntaxe du filtre
// Le filtre doit être un document MongoDB valide

// ❌ Mauvais : Syntaxe JSON invalide
auditLog:
  filter: '{ atype: authenticate }'  // Manque les guillemets

// ✅ Bon : JSON valide
auditLog:
  filter: '{ "atype": "authenticate" }'

// ✅ Bon : YAML multi-lignes
auditLog:
  filter: |
    {
      "atype": "authenticate"
    }

// Tester le filtre
mongosh --eval 'db.adminCommand({ getParameter: 1, auditAuthorizationSuccess: 1 })'
```

## Bonnes pratiques de production

### Checklist de déploiement

```
☐ Configuration
  ☐ Audit activé sur tous les membres (Replica Set)
  ☐ Filtre approprié pour le cas d'usage
  ☐ Destination configurée (file, syslog, console)
  ☐ Format JSON (pas BSON)

☐ Stockage
  ☐ Disque dédié pour les logs (si volume élevé)
  ☐ Espace suffisant (estimer 1-10 GB/jour selon filtre)
  ☐ Rotation configurée (logrotate)
  ☐ Archivage vers stockage froid planifié

☐ Monitoring
  ☐ Alertes sur échecs d'authentification
  ☐ Alertes sur opérations sensibles
  ☐ Monitoring de l'espace disque
  ☐ Intégration SIEM configurée

☐ Conformité
  ☐ Rétention conforme aux exigences (PCI, HIPAA, etc.)
  ☐ Logs protégés en lecture/écriture
  ☐ Backup des logs d'audit
  ☐ Documentation des événements audités

☐ Performance
  ☐ Impact performance mesuré (< 5% recommandé)
  ☐ Filtre optimisé (pas de "tout auditer")
  ☐ Tests de charge avec audit activé

☐ Sécurité
  ☐ Accès aux logs restreint (chmod 640)
  ☐ Logs non modifiables par les utilisateurs DB
  ☐ Chiffrement des logs en transit (syslog TLS)
  ☐ Chiffrement des archives (S3 SSE)
```

### Configuration de référence par environnement

#### Développement

```yaml
# mongod.conf - Développement (audit minimal)
auditLog:
  destination: console
  format: JSON
  filter: '{ "atype": "authenticate" }'
```

#### Staging

```yaml
# mongod.conf - Staging
auditLog:
  destination: file
  format: JSON
  path: /var/log/mongodb/audit.json
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        { atype: { $in: ["createUser", "dropUser"] } },
        { atype: { $in: ["dropDatabase", "dropCollection"] } }
      ]
    }
```

#### Production (conformité PCI-DSS)

```yaml
# mongod.conf - Production PCI-DSS
auditLog:
  destination: file
  format: JSON
  path: /audit-disk/mongodb/audit-pci.json
  filter: |
    {
      $or: [
        { atype: "authenticate" },
        { atype: { $in: ["createUser", "updateUser", "dropUser"] } },
        { atype: { $in: ["createRole", "updateRole", "dropRole"] } },
        { atype: { $in: ["grantRolesToUser", "revokeRolesFromUser"] } },
        { atype: { $in: ["dropDatabase", "dropCollection"] } },
        {
          atype: "authCheck",
          "param.ns": { $regex: "^payment\\." }
        },
        { atype: "shutdown" }
      ]
    }
```

### Estimation des volumes

```
Facteurs influençant le volume :

1. Nombre d'opérations / seconde
2. Nombre de champs audités par événement
3. Filtre appliqué

Estimation :
├─ Audit léger (auth + admin ops) : 100-500 MB/jour
├─ Audit moyen (+ authCheck critiques) : 1-5 GB/jour
└─ Audit complet (tout authCheck) : 10-100+ GB/jour

Exemple pour 100K req/s :
├─ Sans filtre : ~50 GB/jour
├─ Filtre modéré : ~5 GB/jour
└─ Filtre strict : ~500 MB/jour
```

## Conclusion

L'audit MongoDB est un composant essentiel de toute stratégie de sécurité et de conformité en production. Il fournit la traçabilité nécessaire pour :

1. **Conformité réglementaire** : Répondre aux exigences PCI-DSS, HIPAA, SOX, RGPD
2. **Sécurité** : Détecter les intrusions, activités suspectes, et comportements anormaux
3. **Forensics** : Investiguer les incidents de sécurité
4. **Troubleshooting** : Résoudre les problèmes de permissions et d'accès

**Points clés** :

- Utiliser des **filtres appropriés** pour équilibrer volume et couverture
- **Rotation et archivage** réguliers pour gérer l'espace disque
- **Intégration SIEM** pour analyse et alerting en temps réel
- **Tests réguliers** de la chaîne d'audit (génération → stockage → analyse)
- **Documentation** des événements audités pour les auditeurs

Un audit bien configuré est transparent pour les performances (< 5% d'overhead) tout en fournissant une visibilité complète sur les activités de la base de données.

---

**Ressources complémentaires** :
- MongoDB Manual : Auditing
- PCI-DSS Logging Requirements
- HIPAA Audit Controls
- NIST SP 800-92 : Guide to Computer Security Log Management

⏭️ [Network Security et IP Whitelisting](/11-securite/07-network-security-ip-whitelisting.md)
