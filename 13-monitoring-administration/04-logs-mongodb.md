🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.4 Logs MongoDB

## Introduction

Les **logs MongoDB** constituent la source primaire d'informations pour le diagnostic, le monitoring et l'audit des instances MongoDB. Contrairement au profiler qui se concentre sur les performances des requêtes, les logs offrent une vue holistique de tous les événements système, des erreurs, des avertissements et des opérations d'administration.

Pour un SRE ou administrateur système, la maîtrise des logs MongoDB est cruciale pour :
- **Diagnostic rapide** des incidents en production
- **Détection proactive** des problèmes avant qu'ils n'impactent les utilisateurs
- **Audit de sécurité** et traçabilité des accès
- **Analyse des tendances** et planification de capacité
- **Validation** des changements de configuration

---

## Architecture du Système de Logging

### Vue d'Ensemble

MongoDB utilise un système de logging structuré basé sur plusieurs composants :

```
┌──────────────────────────────────────────────────────────┐
│                     MongoDB Process                      │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │   Network    │  │   Storage    │  │  Replication │    │
│  │  Component   │  │  Component   │  │  Component   │    │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘    │
│         │                  │                  │          │
│         └──────────────────┼──────────────────┘          │
│                            │                             │
│                    ┌───────▼────────┐                    │
│                    │  Log Subsystem │                    │
│                    │  (Structured)  │                    │
│                    └───────┬────────┘                    │
└────────────────────────────┼─────────────────────────────┘
                             │
                ┌────────────┼────────────┐
                │            │            │
         ┌──────▼─────┐ ┌────▼───┐ ┌──────▼──┐
         │  mongod.log│ │ syslog │ │  JSON   │
         │   (file)   │ │        │ │  file   │
         └────────────┘ └────────┘ └─────────┘
```

### Destinations des Logs

MongoDB peut écrire les logs vers plusieurs destinations :

| Destination | Usage | Avantages | Inconvénients |
|-------------|-------|-----------|---------------|
| **File** | Production (défaut) | Rotation native, parsing facile | Nécessite gestion disque |
| **Syslog** | Intégration système | Centralisation native | Format moins structuré |
| **stdout/stderr** | Conteneurs Docker | Compatible 12-factor apps | Pas de rotation native |
| **JSON** | Analyse automatisée | Parsing structuré | Plus verbeux |

---

## Configuration des Logs

### Fichier de Configuration (mongod.conf)

Configuration typique pour la production :

```yaml
systemLog:
  # Destination des logs
  destination: file
  path: /var/log/mongodb/mongod.log

  # Rotation automatique
  logRotate: reopen  # Options: rename, reopen
  logAppend: true

  # Format structuré (MongoDB 4.4+)
  component:
    accessControl:
      verbosity: 1
    command:
      verbosity: 0
    control:
      verbosity: 0
    ftdc:
      verbosity: 0
    geo:
      verbosity: 0
    index:
      verbosity: 0
    network:
      verbosity: 0
    query:
      verbosity: 1
    replication:
      verbosity: 1
    sharding:
      verbosity: 1
    storage:
      verbosity: 0
      journal:
        verbosity: 1
    write:
      verbosity: 0

  # Niveau global
  verbosity: 0

  # Timestamps précis (recommandé)
  timeStampFormat: iso8601-utc
```

### Configuration via Ligne de Commande

```bash
# Démarrage avec logs verbeux
mongod --logpath /var/log/mongodb/mongod.log \
       --logappend \
       --verbose \
       --setParameter diagnosticDataCollectionDirectoryPath=/var/log/mongodb/diagnostic.data

# Mode debug complet (seulement pour diagnostic temporaire)
mongod --logpath /var/log/mongodb/mongod-debug.log \
       --verbose vvvvv  # 5 niveaux de verbosité
```

### Modification Dynamique en Runtime

```javascript
// Augmenter la verbosité globalement
db.setLogLevel(1)

// Augmenter la verbosité pour un composant spécifique
db.setLogLevel(2, "query")
db.setLogLevel(1, "replication")

// Vérifier les niveaux actuels
db.getLogComponents()

// Résultat
{
  "verbosity": 0,
  "accessControl": { "verbosity": -1 },
  "command": { "verbosity": -1 },
  "control": { "verbosity": -1 },
  "query": { "verbosity": 2 },     // Modifié
  "replication": { "verbosity": 1 }, // Modifié
  "storage": {
    "verbosity": -1,
    "journal": { "verbosity": -1 },
    "recovery": { "verbosity": -1 }
  }
}
```

---

## Niveaux de Verbosité

### Hiérarchie des Niveaux

MongoDB utilise une échelle de verbosité de -1 à 5 :

| Niveau | Nom | Description | Usage | Impact Performance |
|--------|-----|-------------|-------|-------------------|
| **-1** | Inherit | Hérite du niveau parent | Défaut des composants | Aucun |
| **0** | Default | Messages standards (INFO, WARNING, ERROR) | Production standard | Minimal (< 1%) |
| **1** | Debug 1 | Informations de debug basiques | Diagnostic léger | Faible (1-3%) |
| **2** | Debug 2 | Informations de debug détaillées | Investigation | Modéré (3-7%) |
| **3** | Debug 3 | Informations très détaillées | Debug approfondi | Significatif (7-15%) |
| **4** | Debug 4 | Trace complète | Debug développeur | Élevé (15-25%) |
| **5** | Debug 5 | Trace exhaustive | Debug interne MongoDB | Très élevé (25-40%) |

### Recommandations par Environnement

```yaml
# PRODUCTION - Configuration conservative
systemLog:
  verbosity: 0
  component:
    query:
      verbosity: 0     # Queries lentes uniquement (> slowms)
    replication:
      verbosity: 0     # Events majeurs uniquement
    sharding:
      verbosity: 0
    storage:
      journal:
        verbosity: 0

# STAGING - Plus de détails pour validation
systemLog:
  verbosity: 0
  component:
    query:
      verbosity: 1     # Toutes les queries lentes + contexte
    replication:
      verbosity: 1     # Events + détails sync
    command:
      verbosity: 1     # Commandes d'admin

# DÉVELOPPEMENT - Maximum de visibilité
systemLog:
  verbosity: 1
  component:
    query:
      verbosity: 2     # Détails de parsing et plans
    replication:
      verbosity: 2
    network:
      verbosity: 1     # Connexions/déconnexions
```

---

## Format des Entrées de Log

### Format Legacy (Pré-4.4)

```
2024-12-08T15:23:45.678+0000 I NETWORK  [conn123] received client metadata from 10.0.1.45:54321
```

**Structure** :
- `2024-12-08T15:23:45.678+0000` - Timestamp ISO-8601
- `I` - Sévérité (I=Info, W=Warning, E=Error, F=Fatal)
- `NETWORK` - Composant
- `[conn123]` - Contexte
- Message texte

### Format Structuré (MongoDB 4.4+)

```json
{
  "t": {
    "$date": "2024-12-08T15:23:45.678Z"
  },
  "s": "I",
  "c": "NETWORK",
  "id": 51800,
  "ctx": "conn123",
  "msg": "client metadata",
  "attr": {
    "remote": "10.0.1.45:54321",
    "client": "conn123",
    "doc": {
      "application": {
        "name": "ProductionAPI"
      },
      "driver": {
        "name": "nodejs",
        "version": "4.12.0"
      },
      "os": {
        "type": "Linux",
        "name": "Ubuntu",
        "architecture": "x86_64",
        "version": "20.04"
      }
    }
  }
}
```

**Champs clés** :
- `t` - Timestamp précis avec timezone
- `s` - Severity (I/W/E/F/D1-D5)
- `c` - Component
- `id` - Message ID unique
- `ctx` - Context (thread/connection)
- `msg` - Message template
- `attr` - Attributs structurés (variables)

---

## Composants et Catégories de Logs

### Composants Principaux

#### 1. ACCESS (Contrôle d'Accès)

**Événements typiques** :
- Tentatives d'authentification
- Changements d'autorisation
- Échecs de sécurité

```json
// Authentification réussie
{
  "s": "I",
  "c": "ACCESS",
  "id": 20250,
  "msg": "Authentication succeeded",
  "attr": {
    "principalName": "apiUser",
    "authenticationDatabase": "admin",
    "client": "10.0.1.45:54321",
    "mechanism": "SCRAM-SHA-256"
  }
}

// Échec d'authentification
{
  "s": "I",
  "c": "ACCESS",
  "id": 20249,
  "msg": "Authentication failed",
  "attr": {
    "principalName": "unknownUser",
    "authenticationDatabase": "admin",
    "client": "192.168.1.100:45678",
    "mechanism": "SCRAM-SHA-256",
    "result": "UserNotFound"
  }
}
```

#### 2. COMMAND (Exécution de Commandes)

**Événements typiques** :
- Commandes d'administration
- Agrégations complexes
- Opérations longues

```json
{
  "s": "I",
  "c": "COMMAND",
  "id": 51803,
  "msg": "Slow query",
  "attr": {
    "type": "command",
    "ns": "mydb.users",
    "command": {
      "find": "users",
      "filter": {
        "status": "active",
        "lastLogin": { "$lt": { "$date": "2024-11-01T00:00:00Z" } }
      },
      "sort": { "createdAt": -1 },
      "limit": 1000
    },
    "planSummary": "IXSCAN { status: 1, lastLogin: 1 }",
    "keysExamined": 45678,
    "docsExamined": 45678,
    "nreturned": 1000,
    "durationMillis": 1234,
    "numYields": 0,
    "locks": {
      "Global": { "acquireCount": { "r": 1 } },
      "Database": { "acquireCount": { "r": 1 } },
      "Collection": { "acquireCount": { "r": 1 } }
    }
  }
}
```

#### 3. NETWORK (Réseau)

**Événements typiques** :
- Connexions/déconnexions
- Erreurs réseau
- Timeouts

```json
// Nouvelle connexion
{
  "s": "I",
  "c": "NETWORK",
  "id": 22943,
  "msg": "Connection accepted",
  "attr": {
    "remote": "10.0.1.45:54321",
    "connectionId": 123,
    "connectionCount": 87
  }
}

// Déconnexion anormale
{
  "s": "I",
  "c": "NETWORK",
  "id": 22944,
  "msg": "Connection ended",
  "attr": {
    "remote": "10.0.1.45:54321",
    "connectionId": 123,
    "error": "SocketException: Connection reset by peer"
  }
}
```

#### 4. REPL (Réplication)

**Événements typiques** :
- Élections de primary
- Synchronisation
- Changements de statut

```json
// Élection d'un nouveau primary
{
  "s": "I",
  "c": "REPL",
  "id": 21358,
  "msg": "Election succeeded",
  "attr": {
    "term": 5,
    "newPrimary": "mongodb-rs0-1:27017",
    "votesReceived": 2,
    "votesRequired": 2
  }
}

// Lag de réplication détecté
{
  "s": "W",
  "c": "REPL",
  "id": 21236,
  "msg": "Replication lag detected",
  "attr": {
    "member": "mongodb-rs0-2:27017",
    "lagSeconds": 45,
    "lastApplied": { "$date": "2024-12-08T15:22:00.000Z" },
    "primaryLastApplied": { "$date": "2024-12-08T15:22:45.000Z" }
  }
}
```

#### 5. SHARDING (Sharding)

**Événements typiques** :
- Migrations de chunks
- Balancing
- Opérations de configuration

```json
// Début de migration de chunk
{
  "s": "I",
  "c": "SHARDING",
  "id": 22016,
  "msg": "Chunk migration started",
  "attr": {
    "namespace": "mydb.orders",
    "chunk": {
      "min": { "orderId": 1000000 },
      "max": { "orderId": 2000000 }
    },
    "from": "shard01",
    "to": "shard02"
  }
}

// Migration terminée
{
  "s": "I",
  "c": "SHARDING",
  "id": 22017,
  "msg": "Chunk migration completed",
  "attr": {
    "namespace": "mydb.orders",
    "durationMillis": 15678,
    "docsMovedCount": 123456,
    "bytesMoved": 524288000
  }
}
```

#### 6. STORAGE (Stockage)

**Événements typiques** :
- Checkpoints WiredTiger
- Opérations de journaling
- Problèmes d'espace disque

```json
// Checkpoint WiredTiger
{
  "s": "I",
  "c": "STORAGE",
  "id": 22430,
  "msg": "WiredTiger message",
  "attr": {
    "message": "Checkpoint completed. Written 1234 MB in 2.5 seconds"
  }
}

// Avertissement espace disque
{
  "s": "W",
  "c": "STORAGE",
  "id": 22297,
  "msg": "Low disk space warning",
  "attr": {
    "path": "/data/db",
    "freeSpaceMB": 512,
    "totalSpaceMB": 102400,
    "usagePercent": 99.5
  }
}
```

#### 7. QUERY (Requêtes)

**Événements typiques** :
- Plans de requête
- Requêtes lentes
- Analyses de performance

```json
// Requête lente avec détails
{
  "s": "I",
  "c": "QUERY",
  "id": 51803,
  "msg": "Slow query",
  "attr": {
    "ns": "mydb.products",
    "durationMillis": 2345,
    "planSummary": "COLLSCAN",
    "keysExamined": 0,
    "docsExamined": 567890,
    "nreturned": 10,
    "queryHash": "A1B2C3D4",
    "planCacheKey": "E5F6G7H8",
    "command": {
      "find": "products",
      "filter": {
        "category": "electronics",
        "price": { "$gte": 100, "$lte": 500 }
      }
    }
  }
}
```

---

## Patterns de Log et Leur Signification

### Patterns Critiques à Surveiller

#### 1. Connection Pool Exhaustion

```
Pattern: "Connection refused" ou "Too many connections"
```

```json
{
  "s": "W",
  "c": "NETWORK",
  "id": 22942,
  "msg": "Connection pool exhausted",
  "attr": {
    "maxPoolSize": 100,
    "currentConnections": 100,
    "queuedRequests": 45
  }
}
```

**Diagnostic** :
- Application ne libère pas les connexions
- Pool size insuffisant
- Pic de trafic inattendu

**Action** :
```javascript
// Vérifier les connexions actuelles
db.currentOp({ "active": true })

// Augmenter le pool côté driver ou serveur
// mongod.conf
net:
  maxIncomingConnections: 200
```

#### 2. Replication Lag Critique

```
Pattern: "Replication lag" avec lagSeconds élevé
```

**Critères d'alerte** :
- lagSeconds > 30s : Warning
- lagSeconds > 60s : Critical
- lagSeconds > 300s : Emergency

**Actions** :
```javascript
// 1. Vérifier l'état du replica set
rs.status()

// 2. Analyser l'oplog
db.getReplicationInfo()

// 3. Identifier la source du lag
db.printSlaveReplicationInfo()
```

#### 3. Checkpoint Lent (WiredTiger)

```
Pattern: "Checkpoint completed" avec durée > 60s
```

```json
{
  "s": "W",
  "c": "STORAGE",
  "id": 22430,
  "msg": "WiredTiger checkpoint slow",
  "attr": {
    "durationSeconds": 127,
    "dataSizeMB": 45678,
    "writeRateMBps": 359
  }
}
```

**Causes possibles** :
- I/O disque saturé
- Cache WiredTiger sous-dimensionné
- Volume d'écritures excessif

**Diagnostic** :
```bash
# Vérifier les I/O disque
iostat -x 1 10

# Vérifier le cache WiredTiger
mongo --eval 'db.serverStatus().wiredTiger.cache' | grep "bytes currently in the cache"
```

#### 4. Élections Fréquentes

```
Pattern: Multiples "Election succeeded" sur courte période
```

**Seuils d'alerte** :
- > 1 élection/heure : Investigation
- > 5 élections/heure : Critique

**Causes** :
- Latence réseau entre membres
- Heartbeat timeout trop court
- Charge CPU élevée sur primary

**Analyse** :
```javascript
// Historique des élections
db.getSiblingDB('local').system.replset.find().pretty()

// Latences réseau
rs.status().members.forEach(function(m) {
  print(m.name + ": " + m.pingMs + "ms");
});
```

#### 5. Opérations Échouées avec WriteConflict

```
Pattern: "WriteConflict" dans les logs
```

```json
{
  "s": "I",
  "c": "WRITE",
  "id": 51803,
  "msg": "Write operation encountered WriteConflict",
  "attr": {
    "namespace": "mydb.inventory",
    "retries": 3,
    "totalDurationMillis": 156
  }
}
```

**Signification** :
- Contention élevée sur documents
- Transactions concurrentes
- Normal en faible quantité, problématique si fréquent

---

## Analyse des Logs

### Commandes Essentielles

#### 1. Extraction des Erreurs

```bash
# Toutes les erreurs
grep '"s":"E"' /var/log/mongodb/mongod.log

# Erreurs des dernières 24h
find /var/log/mongodb -name "mongod.log*" -mtime -1 -exec grep '"s":"E"' {} \;

# Count par type d'erreur
grep '"s":"E"' /var/log/mongodb/mongod.log | \
  jq -r '.c' | sort | uniq -c | sort -rn
```

#### 2. Analyse des Requêtes Lentes

```bash
# Requêtes > 1 seconde
grep '"msg":"Slow query"' /var/log/mongodb/mongod.log | \
  jq 'select(.attr.durationMillis > 1000)'

# Top 10 des requêtes les plus lentes
grep '"msg":"Slow query"' /var/log/mongodb/mongod.log | \
  jq -r '[.attr.ns, .attr.durationMillis] | @csv' | \
  sort -t',' -k2 -rn | head -10

# Distribution des durées
grep '"msg":"Slow query"' /var/log/mongodb/mongod.log | \
  jq -r '.attr.durationMillis' | \
  awk '{
    if ($1 < 100) range["<100ms"]++
    else if ($1 < 500) range["100-500ms"]++
    else if ($1 < 1000) range["500ms-1s"]++
    else if ($1 < 5000) range["1-5s"]++
    else range[">5s"]++
  }
  END {
    for (r in range) print r ": " range[r]
  }'
```

#### 3. Analyse des Connexions

```bash
# Pics de connexions
grep '"msg":"Connection accepted"' /var/log/mongodb/mongod.log | \
  jq -r '.t.$date[0:13]' | uniq -c

# Connexions par IP
grep '"msg":"Connection accepted"' /var/log/mongodb/mongod.log | \
  jq -r '.attr.remote' | cut -d':' -f1 | sort | uniq -c | sort -rn

# Déconnexions anormales
grep '"msg":"Connection ended"' /var/log/mongodb/mongod.log | \
  jq 'select(.attr.error != null)'
```

#### 4. Monitoring de Réplication

```bash
# Événements de réplication
grep '"c":"REPL"' /var/log/mongodb/mongod.log | tail -50

# Détection de lag
grep '"msg":"Replication lag"' /var/log/mongodb/mongod.log | \
  jq '.attr | {member: .member, lagSeconds: .lagSeconds}'

# Historique des élections
grep '"msg":"Election succeeded"' /var/log/mongodb/mongod.log | \
  jq '{time: .t.$date, term: .attr.term, newPrimary: .attr.newPrimary}'
```

#### 5. Analyse de Sharding

```bash
# Migrations en cours
grep '"msg":"Chunk migration"' /var/log/mongodb/mongod.log | tail -20

# Durée des migrations
grep '"msg":"Chunk migration completed"' /var/log/mongodb/mongod.log | \
  jq '{namespace: .attr.namespace, durationMin: (.attr.durationMillis/60000), docsMoved: .attr.docsMovedCount}'

# Échecs de balancing
grep '"c":"SHARDING"' /var/log/mongodb/mongod.log | \
  jq 'select(.s == "E" or .s == "W")'
```

---

## Outils d'Analyse Avancés

### 1. mtools (MongoDB Log Analysis)

Installation :
```bash
pip install mtools[mloginfo]
```

**Analyse statistique des logs** :
```bash
# Vue d'ensemble
mloginfo /var/log/mongodb/mongod.log

# Output exemple
╔═══════════════════════════════════════════════════════════════╗
║                         GENERAL STATS                         ║
╠═══════════════════════════════════════════════════════════════╣
║  Log file:        /var/log/mongodb/mongod.log                 ║
║  Lines parsed:    1,234,567                                   ║
║  Date range:      Dec 07 00:00:01 - Dec 08 15:30:45 (1d 15h) ║
║  MongoDB version: 6.0.11                                      ║
╚═══════════════════════════════════════════════════════════════╝

╔═══════════════════════════════════════════════════════════════╗
║                        QUERY PATTERNS                         ║
╠═══════════════════════════════════════════════════════════════╣
║  namespace                 pattern              count   %     ║
╠═══════════════════════════════════════════════════════════════╣
║  mydb.users               find { status }       45678  37.0%  ║
║  mydb.orders              aggregate             23456  19.0%  ║
║  mydb.products            find { category }     12345  10.0%  ║
╚═══════════════════════════════════════════════════════════════╝

# Requêtes lentes uniquement
mloginfo /var/log/mongodb/mongod.log --queries --sort-queries duration

# Analyse des connexions
mloginfo /var/log/mongodb/mongod.log --connections

# Analyse de réplication
mloginfo /var/log/mongodb/mongod.log --rsstate
```

**Visualisation graphique** :
```bash
# Graphique des requêtes par heure
mplotqueries /var/log/mongodb/mongod.log --type duration --group hour

# Graphique de réplication
mplotqueries /var/log/mongodb/mongod.log --type replication
```

### 2. Parsing avec jq (JSON Query)

**Script d'analyse complet** :
```bash
#!/bin/bash
# analyze_mongodb_logs.sh

LOGFILE="/var/log/mongodb/mongod.log"
OUTPUT_DIR="./analysis_$(date +%Y%m%d_%H%M%S)"
mkdir -p "$OUTPUT_DIR"

echo "Analyzing MongoDB logs: $LOGFILE"

# 1. Summary statistics
echo "=== SUMMARY ===" > "$OUTPUT_DIR/summary.txt"
total=$(wc -l < "$LOGFILE")
errors=$(grep -c '"s":"E"' "$LOGFILE")
warnings=$(grep -c '"s":"W"' "$LOGFILE")
echo "Total lines: $total" >> "$OUTPUT_DIR/summary.txt"
echo "Errors: $errors" >> "$OUTPUT_DIR/summary.txt"
echo "Warnings: $warnings" >> "$OUTPUT_DIR/summary.txt"

# 2. Top errors
echo "=== TOP ERRORS ===" > "$OUTPUT_DIR/top_errors.txt"
grep '"s":"E"' "$LOGFILE" | \
  jq -r '.msg' | sort | uniq -c | sort -rn | head -20 >> "$OUTPUT_DIR/top_errors.txt"

# 3. Slow queries analysis
echo "=== SLOW QUERIES ===" > "$OUTPUT_DIR/slow_queries.json"
grep '"msg":"Slow query"' "$LOGFILE" | \
  jq -s 'sort_by(.attr.durationMillis) | reverse | .[0:50]' > "$OUTPUT_DIR/slow_queries.json"

# 4. Connection analysis
echo "=== CONNECTION STATS ===" > "$OUTPUT_DIR/connections.txt"
grep '"msg":"Connection accepted"' "$LOGFILE" | \
  jq -r '.attr.remote' | cut -d':' -f1 | sort | uniq -c | sort -rn >> "$OUTPUT_DIR/connections.txt"

# 5. Component breakdown
echo "=== COMPONENT ACTIVITY ===" > "$OUTPUT_DIR/components.txt"
jq -r '.c' "$LOGFILE" 2>/dev/null | sort | uniq -c | sort -rn >> "$OUTPUT_DIR/components.txt"

echo "Analysis complete. Results in: $OUTPUT_DIR"
```

### 3. Logrotate Configuration

Configuration optimale pour production :

```bash
# /etc/logrotate.d/mongodb
/var/log/mongodb/*.log {
    daily
    rotate 30
    missingok
    notifempty
    sharedscripts
    compress
    delaycompress
    postrotate
        /usr/bin/killall -SIGUSR1 mongod 2>/dev/null || true
    endscript
}
```

**Rotation manuelle** :
```javascript
// Via MongoDB
db.adminCommand({ logRotate: 1 })

// Vérification
db.adminCommand({ getLog: "global" }).log.slice(-10)
```

---

## Intégration avec Stack de Monitoring

### 1. ELK Stack (Elasticsearch, Logstash, Kibana)

**Logstash Pipeline** :
```ruby
# /etc/logstash/conf.d/mongodb.conf
input {
  file {
    path => "/var/log/mongodb/mongod.log"
    start_position => "beginning"
    codec => json
    type => "mongodb"
  }
}

filter {
  if [type] == "mongodb" {
    # Parse timestamp
    date {
      match => ["t.$date", "ISO8601"]
      target => "@timestamp"
    }

    # Severity mapping
    translate {
      field => "s"
      destination => "severity_name"
      dictionary => {
        "F" => "FATAL"
        "E" => "ERROR"
        "W" => "WARNING"
        "I" => "INFO"
        "D" => "DEBUG"
      }
    }

    # Extract slow query metrics
    if [msg] == "Slow query" {
      mutate {
        add_field => {
          "query_duration_ms" => "%{[attr][durationMillis]}"
          "query_namespace" => "%{[attr][ns]}"
          "docs_examined" => "%{[attr][docsExamined]}"
          "keys_examined" => "%{[attr][keysExamined]}"
          "docs_returned" => "%{[attr][nreturned]}"
        }
      }

      # Calculate efficiency ratio
      ruby {
        code => "
          docs_examined = event.get('docs_examined').to_f
          docs_returned = event.get('docs_returned').to_f
          if docs_returned > 0
            event.set('efficiency_ratio', docs_examined / docs_returned)
          end
        "
      }
    }

    # Geo-IP for client IPs
    if [attr][remote] {
      grok {
        match => { "[attr][remote]" => "%{IP:client_ip}:%{NUMBER:client_port}" }
      }

      if [client_ip] {
        geoip {
          source => "client_ip"
        }
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "mongodb-logs-%{+YYYY.MM.dd}"
  }
}
```

**Kibana Dashboard Queries** :
```json
// Requêtes lentes par namespace
{
  "size": 0,
  "query": {
    "bool": {
      "must": [
        { "match": { "msg": "Slow query" }},
        { "range": { "@timestamp": { "gte": "now-24h" }}}
      ]
    }
  },
  "aggs": {
    "by_namespace": {
      "terms": {
        "field": "query_namespace.keyword",
        "size": 10
      },
      "aggs": {
        "avg_duration": {
          "avg": { "field": "query_duration_ms" }
        }
      }
    }
  }
}
```

### 2. Prometheus + Loki

**Promtail Configuration** :
```yaml
# /etc/promtail/config.yml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: mongodb
    static_configs:
      - targets:
          - localhost
        labels:
          job: mongodb
          host: mongodb-prod-01
          __path__: /var/log/mongodb/*.log

    pipeline_stages:
      # Parse JSON
      - json:
          expressions:
            timestamp: t.$date
            severity: s
            component: c
            message: msg
            context: ctx

      # Extract timestamp
      - timestamp:
          source: timestamp
          format: RFC3339Nano

      # Add labels
      - labels:
          severity:
          component:

      # Extract metrics for slow queries
      - match:
          selector: '{job="mongodb"} |= "Slow query"'
          stages:
            - json:
                expressions:
                  duration: attr.durationMillis
                  namespace: attr.ns
            - labels:
                namespace:
            - metrics:
                slow_query_duration_ms:
                  type: Histogram
                  description: "MongoDB slow query duration"
                  source: duration
                  config:
                    buckets: [100, 500, 1000, 5000, 10000]
```

**LogQL Queries** :
```logql
# Taux d'erreurs par composant
sum by (component) (rate({job="mongodb", severity="E"}[5m]))

# Requêtes lentes > 1s
{job="mongodb"} |= "Slow query" | json | attr_durationMillis > 1000

# Connexions par IP
{job="mongodb"} |= "Connection accepted" | json | line_format "{{.attr_remote}}"

# Élections de replica set
{job="mongodb", component="REPL"} |= "Election"
```

### 3. Grafana Alerting

**Exemple de règles d'alerte** :
```yaml
# mongodb_alerts.yaml
groups:
  - name: mongodb_logs
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({job="mongodb", severity="E"}[5m])) > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Taux d'erreurs MongoDB élevé"
          description: "Plus de 10 erreurs/sec détectées"

      - alert: ReplicationLag
        expr: |
          mongodb_replication_lag_seconds > 30
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Lag de réplication détecté"
          description: "Lag: {{ $value }}s sur {{ $labels.member }}"

      - alert: SlowQueriesSpike
        expr: |
          rate({job="mongodb"} |= "Slow query"[5m]) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pic de requêtes lentes"
          description: "Taux de requêtes lentes élevé"

      - alert: FrequentElections
        expr: |
          sum(increase({job="mongodb", component="REPL"} |= "Election succeeded"[1h])) > 2
        labels:
          severity: critical
        annotations:
          summary: "Élections fréquentes du replica set"
          description: "Plus de 2 élections en 1 heure"
```

---

## Cas d'Usage Avancés

### Scénario 1 : Investigation Post-Mortem

**Contexte** : Incident de performance entre 14h00 et 14h30.

```bash
#!/bin/bash
# postmortem_analysis.sh

INCIDENT_START="2024-12-08T14:00:00"
INCIDENT_END="2024-12-08T14:30:00"
LOGFILE="/var/log/mongodb/mongod.log"
OUTPUT="incident_analysis_$(date +%Y%m%d).txt"

echo "=== POST-MORTEM ANALYSIS ===" > $OUTPUT
echo "Incident window: $INCIDENT_START to $INCIDENT_END" >> $OUTPUT
echo "" >> $OUTPUT

# 1. Timeline of events
echo "=== TIMELINE ===" >> $OUTPUT
jq -r 'select(.t."$date" >= "'"$INCIDENT_START"'" and .t."$date" <= "'"$INCIDENT_END"'") |
       [.t."$date", .s, .c, .msg] | @tsv' $LOGFILE | head -100 >> $OUTPUT

# 2. Errors during incident
echo -e "\n=== ERRORS ===" >> $OUTPUT
jq 'select(.t."$date" >= "'"$INCIDENT_START"'" and .t."$date" <= "'"$INCIDENT_END"'" and .s == "E")' \
   $LOGFILE >> $OUTPUT

# 3. Slow queries
echo -e "\n=== SLOW QUERIES ===" >> $OUTPUT
jq 'select(.t."$date" >= "'"$INCIDENT_START"'" and .t."$date" <= "'"$INCIDENT_END"'" and
           .msg == "Slow query")' $LOGFILE | \
   jq -s 'sort_by(.attr.durationMillis) | reverse | .[0:20]' >> $OUTPUT

# 4. Connection activity
echo -e "\n=== CONNECTION SPIKES ===" >> $OUTPUT
jq -r 'select(.t."$date" >= "'"$INCIDENT_START"'" and .t."$date" <= "'"$INCIDENT_END"'" and
              .msg == "Connection accepted") | .t."$date"[0:16]' $LOGFILE | \
   uniq -c | sort -rn >> $OUTPUT

# 5. Replication status
echo -e "\n=== REPLICATION EVENTS ===" >> $OUTPUT
jq 'select(.t."$date" >= "'"$INCIDENT_START"'" and .t."$date" <= "'"$INCIDENT_END"'" and
           .c == "REPL")' $LOGFILE >> $OUTPUT

echo "Analysis complete: $OUTPUT"
```

### Scénario 2 : Détection d'Anomalies

**Script de détection automatique** :
```python
#!/usr/bin/env python3
# anomaly_detector.py

import json
import sys
from collections import defaultdict, Counter
from datetime import datetime, timedelta

class MongoDBAnomalyDetector:
    def __init__(self, logfile):
        self.logfile = logfile
        self.anomalies = []

    def detect(self):
        # Baselines
        normal_error_rate = 5  # erreurs/minute
        normal_slow_query_rate = 10  # requêtes/minute
        normal_connection_count = 100

        errors_per_minute = defaultdict(int)
        slow_queries_per_minute = defaultdict(int)
        connections_per_minute = defaultdict(int)

        with open(self.logfile, 'r') as f:
            for line in f:
                try:
                    entry = json.loads(line)
                    timestamp = entry['t']['$date'][:16]  # Minute precision

                    # Count errors
                    if entry['s'] == 'E':
                        errors_per_minute[timestamp] += 1

                    # Count slow queries
                    if entry.get('msg') == 'Slow query':
                        slow_queries_per_minute[timestamp] += 1

                    # Count connections
                    if 'Connection accepted' in entry.get('msg', ''):
                        connections_per_minute[timestamp] += 1

                except (json.JSONDecodeError, KeyError):
                    continue

        # Detect anomalies
        for minute, count in errors_per_minute.items():
            if count > normal_error_rate * 3:  # 3x baseline
                self.anomalies.append({
                    'type': 'HIGH_ERROR_RATE',
                    'timestamp': minute,
                    'value': count,
                    'baseline': normal_error_rate,
                    'severity': 'CRITICAL' if count > normal_error_rate * 5 else 'WARNING'
                })

        for minute, count in slow_queries_per_minute.items():
            if count > normal_slow_query_rate * 3:
                self.anomalies.append({
                    'type': 'HIGH_SLOW_QUERY_RATE',
                    'timestamp': minute,
                    'value': count,
                    'baseline': normal_slow_query_rate,
                    'severity': 'WARNING'
                })

        return self.anomalies

    def report(self):
        if not self.anomalies:
            print("No anomalies detected.")
            return

        print(f"\n=== DETECTED {len(self.anomalies)} ANOMALIES ===\n")

        for anomaly in sorted(self.anomalies, key=lambda x: x['timestamp']):
            print(f"[{anomaly['severity']}] {anomaly['type']}")
            print(f"  Time: {anomaly['timestamp']}")
            print(f"  Value: {anomaly['value']} (baseline: {anomaly['baseline']})")
            print()

if __name__ == '__main__':
    detector = MongoDBAnomalyDetector('/var/log/mongodb/mongod.log')
    detector.detect()
    detector.report()
```

### Scénario 3 : Audit de Sécurité

```bash
#!/bin/bash
# security_audit.sh

LOGFILE="/var/log/mongodb/mongod.log"
AUDIT_REPORT="security_audit_$(date +%Y%m%d).txt"

echo "=== MONGODB SECURITY AUDIT ===" > $AUDIT_REPORT
echo "Date: $(date)" >> $AUDIT_REPORT
echo "" >> $AUDIT_REPORT

# 1. Failed authentication attempts
echo "=== FAILED AUTHENTICATION ATTEMPTS ===" >> $AUDIT_REPORT
grep '"msg":"Authentication failed"' $LOGFILE | \
  jq -r '[.t.$date, .attr.client, .attr.principalName] | @tsv' | \
  column -t >> $AUDIT_REPORT

# 2. Successful authentications from unusual IPs
echo -e "\n=== AUTHENTICATIONS BY IP ===" >> $AUDIT_REPORT
grep '"msg":"Authentication succeeded"' $LOGFILE | \
  jq -r '.attr.client' | sort | uniq -c | sort -rn >> $AUDIT_REPORT

# 3. Admin command usage
echo -e "\n=== ADMIN COMMANDS ===" >> $AUDIT_REPORT
grep '"c":"COMMAND"' $LOGFILE | \
  jq 'select(.attr.command | keys[] |
     test("shutdown|dropDatabase|dropCollection|createUser|dropUser"))' | \
  jq -r '[.t.$date, .attr.command | keys[0], .ctx] | @tsv' | \
  column -t >> $AUDIT_REPORT

# 4. Connection attempts outside business hours
echo -e "\n=== OFF-HOURS CONNECTIONS ===" >> $AUDIT_REPORT
grep '"msg":"Connection accepted"' $LOGFILE | \
  jq -r 'select(.t.$date | test("T(0[0-6]|2[2-3]):")) |
         [.t.$date, .attr.remote] | @tsv' | \
  column -t >> $AUDIT_REPORT

echo "Audit report generated: $AUDIT_REPORT"
```

---

## Bonnes Pratiques pour SRE

### 1. Politique de Rétention des Logs

| Environnement | Rétention Locale | Rétention Centralisée | Archivage Froid |
|---------------|------------------|----------------------|-----------------|
| **Production** | 7 jours | 90 jours | 1 an (S3/Glacier) |
| **Staging** | 3 jours | 30 jours | 3 mois |
| **Développement** | 1 jour | 7 jours | Aucun |

### 2. Checklist de Configuration

```yaml
# Production-ready logging configuration
✓ Verbosité globale: 0
✓ Composant query: 0 ou 1 (selon SLA)
✓ Composant replication: 0 ou 1
✓ Format: JSON structuré (MongoDB 4.4+)
✓ Rotation: Activée (daily)
✓ Compression: Activée
✓ Destination centralisée: Configurée (ELK/Loki)
✓ Alerting: Configuré sur erreurs critiques
✓ Dashboards: Créés et testés
✓ Runbooks: Documentés pour patterns courants
```

### 3. Niveaux d'Alerte Recommandés

```javascript
// Alerting tiers
const alertingRules = {
  P0_CRITICAL: {
    // Nécessite action immédiate - Page on-call
    conditions: [
      'error_rate > 50/min',
      'replica_set_down',
      'primary_election_failed',
      'disk_space < 5%'
    ],
    response_time: '5 minutes'
  },

  P1_HIGH: {
    // Nécessite action rapide - Ticket urgent
    conditions: [
      'error_rate > 20/min for 5min',
      'replication_lag > 60s',
      'slow_queries > 100/min',
      'connection_pool_exhausted'
    ],
    response_time: '30 minutes'
  },

  P2_MEDIUM: {
    // Investigation nécessaire - Ticket normal
    conditions: [
      'warning_rate > 50/min',
      'replication_lag > 30s',
      'checkpoint_duration > 60s'
    ],
    response_time: '4 hours'
  },

  P3_LOW: {
    // Monitoring - Ticket planifié
    conditions: [
      'slow_query_pattern_detected',
      'unusual_connection_pattern',
      'deprecated_feature_usage'
    ],
    response_time: '1 business day'
  }
};
```

### 4. Runbook Template

```markdown
# Runbook: High Error Rate in MongoDB Logs

## Detection
Alert: "HighErrorRate" triggered
Threshold: > 10 errors/second for 5 minutes

## Triage (< 5 minutes)
1. Vérifier le dashboard MongoDB
2. Identifier le type d'erreur dominant
3. Vérifier le statut du replica set: `rs.status()`
4. Vérifier les connexions: `db.currentOp({"active": true})`

## Common Patterns

### Pattern 1: Connection Errors
Symptoms: "Connection refused", "Too many connections"
Action:
  - Check connection pool: `db.serverStatus().connections`
  - Check application logs for connection leaks
  - Temporary: Increase maxIncomingConnections

### Pattern 2: Write Errors
Symptoms: "WriteConflict", "DuplicateKey"
Action:
  - Identify conflicting documents
  - Review application logic for retries
  - Check for missing indexes

### Pattern 3: Replication Errors
Symptoms: "Replication failed", "OplogSlotTooOld"
Action:
  - Check replica set status
  - Verify network connectivity between members
  - Check oplog size: `db.getReplicationInfo()`

## Escalation
If unresolved after 15 minutes:
1. Page secondary on-call
2. Prepare for potential failover
3. Document all actions in incident ticket
```

### 5. Monitoring Hygiene

```bash
# Weekly log maintenance script
#!/bin/bash
# weekly_log_maintenance.sh

MONGO_LOGS="/var/log/mongodb"
BACKUP_DIR="/backup/mongodb-logs"
RETENTION_DAYS=30

# 1. Compress old logs
find $MONGO_LOGS -name "mongod.log.*" -type f -mtime +7 ! -name "*.gz" \
  -exec gzip {} \;

# 2. Move to backup location
find $MONGO_LOGS -name "*.gz" -mtime +7 \
  -exec mv {} $BACKUP_DIR/ \;

# 3. Delete old backups
find $BACKUP_DIR -name "*.gz" -mtime +$RETENTION_DAYS \
  -exec rm {} \;

# 4. Verify log rotation is working
if [ ! -f "$MONGO_LOGS/mongod.log" ]; then
  echo "ERROR: Current log file missing!" | mail -s "MongoDB Log Alert" ops@example.com
fi

# 5. Check log size
current_size=$(du -m "$MONGO_LOGS/mongod.log" | cut -f1)
if [ $current_size -gt 1000 ]; then
  echo "WARNING: Current log file > 1GB" | mail -s "MongoDB Log Size Warning" ops@example.com
fi

# 6. Generate weekly report
./analyze_mongodb_logs.sh > weekly_report_$(date +%Y%m%d).txt
```

---

## Limitations et Considérations

### Limitations des Logs MongoDB

1. **Volume** : Logs en mode verbose peuvent générer plusieurs GB/jour
2. **Performance** : Verbosité élevée impacte les performances (jusqu'à 20-30%)
3. **Parsing** : Format legacy difficile à parser automatiquement
4. **Contexte limité** : Pas de correlation ID natif entre requêtes
5. **Sampling** : Pas de sampling natif (tout ou rien par niveau)

### Alternatives Complémentaires

| Besoin | Outil Principal | Complément |
|--------|----------------|------------|
| **Performance queries** | Profiler | Logs niveau 1 |
| **Monitoring temps réel** | mongostat/mongotop | Logs niveau 0 |
| **Audit détaillé** | MongoDB Audit Log | Logs sécurité |
| **Métriques système** | Prometheus exporter | Logs + FTDC |
| **Diagnostic approfondi** | explain() + Profiler | Logs niveau 2 |

---

## Résumé pour SRE

### Commandes Essentielles

```bash
# Vérifier configuration actuelle
mongo --eval 'db.getLogComponents()'

# Modification temporaire (verbose queries)
mongo --eval 'db.setLogLevel(1, "query")'

# Rotation manuelle
mongo --eval 'db.adminCommand({ logRotate: 1 })'

# Analyse rapide
tail -f /var/log/mongodb/mongod.log | grep -E '"s":"(E|W)"'

# Extraction erreurs dernière heure
grep '"s":"E"' /var/log/mongodb/mongod.log | \
  jq 'select(.t.$date > "'$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S)'")'
```

### Matrice de Décision : Niveau de Verbosité

| Situation | Configuration | Durée Max |
|-----------|---------------|-----------|
| **Production normale** | Global: 0, Query: 0 | Permanent |
| **Post-déploiement** | Global: 0, Query: 1, Repl: 1 | 48h |
| **Investigation performance** | Query: 2, Command: 1 | 4h |
| **Debug réplication** | Repl: 2, Network: 1 | 2h |
| **Debug sharding** | Sharding: 2, Network: 1 | 2h |
| **Audit sécurité** | AccessControl: 2 | Session |

### KPIs à Surveiller

```yaml
Critical KPIs:
  - Error rate: < 1/min normal, > 10/min critical
  - Warning rate: < 10/min normal, > 50/min warning
  - Slow query rate: < 5/min normal, > 20/min warning
  - Connection churn: < 100 new/min normal
  - Log growth: < 100 MB/day normal

Trends to Monitor:
  - Increasing slow query count: Review indexes
  - Growing error rate: Check application health
  - Connection spikes: Investigate connection pooling
  - Frequent elections: Check network/heartbeat
  - Checkpoint duration increase: I/O or cache issues
```

---

## Conclusion

Les **logs MongoDB** sont un outil fondamental pour tout SRE ou administrateur système. Une configuration appropriée, combinée avec des outils d'analyse modernes et une intégration dans un stack de monitoring centralisé, permet :

1. **Détection proactive** des problèmes avant impact utilisateur
2. **Diagnostic rapide** lors d'incidents
3. **Analyse de tendances** pour la planification
4. **Audit de sécurité** et conformité
5. **Documentation** des patterns d'utilisation

**Points clés à retenir** :
- Verbosité 0 en production, augmentation temporaire pour diagnostic
- JSON structuré pour faciliter le parsing automatisé
- Centralisation obligatoire (ELK/Loki) pour corrélation
- Alerting sur les patterns critiques, pas sur les événements isolés
- Rotation et archivage planifiés pour gestion d'espace

**Prochaines étapes** :
- Configurer la centralisation des logs
- Créer les dashboards de monitoring essentiels
- Établir les runbooks pour les patterns courants
- Automatiser l'analyse et l'alerting
- Former l'équipe aux outils d'analyse

---

**Références** :
- [MongoDB Log Messages Documentation](https://www.mongodb.com/docs/manual/reference/log-messages/)
- [MongoDB Logging and Monitoring](https://www.mongodb.com/docs/manual/administration/monitoring/)
- [mtools Documentation](https://github.com/rueckstiess/mtools)
- [MongoDB Production Notes](https://www.mongodb.com/docs/manual/administration/production-notes/)

⏭️ [MongoDB Database Tools](/13-monitoring-administration/05-mongodb-database-tools.md)
