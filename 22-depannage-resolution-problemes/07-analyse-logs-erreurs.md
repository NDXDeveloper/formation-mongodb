🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.7 Analyse des Logs d'Erreurs

## Vue d'ensemble

Les logs MongoDB sont une source d'information essentielle pour le diagnostic et la résolution de problèmes. Cette section fournit une méthodologie complète pour analyser, interpréter et exploiter efficacement les logs MongoDB afin d'identifier rapidement les causes racines des incidents.

---

## Table des Matières

1. [Structure et Format des Logs](#1-structure-et-format-des-logs)
2. [Localisation et Configuration](#2-localisation-et-configuration)
3. [Niveaux de Sévérité](#3-niveaux-de-s%C3%A9v%C3%A9rit%C3%A9)
4. [Messages Critiques](#4-messages-critiques)
5. [Patterns d'Erreurs Courants](#5-patterns-derreurs-courants)
6. [Outils d'Analyse](#6-outils-danalyse)
7. [Corrélation d'Événements](#7-corr%C3%A9lation-d%C3%A9v%C3%A9nements)
8. [Automatisation et Alertes](#8-automatisation-et-alertes)

---

## 1. Structure et Format des Logs

### Format Standard des Logs MongoDB

```
[timestamp] [severity] [component] [context] message
```

**Exemple de ligne de log :**

```
2024-12-09T10:15:30.123+0000 I NETWORK  [listener] connection accepted from 192.168.1.100:54321 #12345 (10 connections now open)
```

**Décomposition :**

```
2024-12-09T10:15:30.123+0000    → Timestamp (ISO 8601)
I                                 → Severity (Informational)
NETWORK                           → Component
[listener]                        → Context/Thread
connection accepted from...       → Message
```

### Composants Principaux

```javascript
// Liste des composants MongoDB

COMPONENTS = {
  "ACCESS":       "Contrôle d'accès et authentification",
  "ASIO":         "Réseau asynchrone",
  "BRIDGE":       "Mongos routing",
  "COMMAND":      "Exécution de commandes",
  "CONTROL":      "Démarrage et arrêt",
  "DEFAULT":      "Messages généraux",
  "EXECUTOR":     "Pool de threads",
  "FTDC":         "Full Time Diagnostic Data Capture",
  "GEO":          "Requêtes géospatiales",
  "INDEX":        "Opérations sur les index",
  "INITSYNC":     "Synchronisation initiale",
  "JOURNAL":      "Journalisation",
  "NETWORK":      "Connexions réseau",
  "QUERY":        "Exécution de requêtes",
  "REPL":         "Réplication",
  "REPL_HB":      "Heartbeats réplication",
  "ROLLBACK":     "Rollback de réplication",
  "SHARDING":     "Sharding et balancing",
  "STORAGE":      "Moteur de stockage",
  "TXN":          "Transactions",
  "WRITE":        "Opérations d'écriture"
}
```

### Formats de Timestamp

```bash
# Format par défaut (iso8601-local)
2024-12-09T10:15:30.123+0000

# Format ctime (ancien)
Mon Dec  9 10:15:30.123

# Format epoch (millisecondes)
1702118130123
```

---

## 2. Localisation et Configuration

### Localisation par Défaut

```bash
# Linux (installation package)
/var/log/mongodb/mongod.log

# Linux (installation manuelle)
/var/lib/mongodb/logs/mongod.log

# macOS (Homebrew)
/usr/local/var/log/mongodb/mongod.log

# Windows
C:\Program Files\MongoDB\Server\<version>\log\mongod.log

# Docker
docker logs <container_id>
```

### Configuration des Logs

```yaml
# /etc/mongod.conf

systemLog:
  # Destination
  destination: file
  path: /var/log/mongodb/mongod.log

  # Append ou overwrite
  logAppend: true

  # Rotation
  logRotate: reopen  # ou "rename"

  # Verbosité globale (0-5)
  verbosity: 0

  # Verbosité par composant
  component:
    accessControl:
      verbosity: 0
    command:
      verbosity: 0
    control:
      verbosity: 0
    ftdc:
      verbosity: 0
    index:
      verbosity: 0
    network:
      verbosity: 0
    query:
      verbosity: 2      # Augmenté pour debug
    replication:
      verbosity: 0
      heartbeats:
        verbosity: 0
      rollback:
        verbosity: 0
    sharding:
      verbosity: 0
    storage:
      verbosity: 0
      journal:
        verbosity: 0
      recovery:
        verbosity: 0
    write:
      verbosity: 0

  # Timestamp format
  timeStampFormat: iso8601-local  # ou "ctime", "iso8601-utc"
```

### Rotation des Logs

```bash
# Méthode 1 : Via MongoDB (reopen)
mongosh --eval "db.adminCommand({logRotate: 1})"

# MongoDB renomme le fichier actuel et ouvre un nouveau fichier
# mongod.log → mongod.log.2024-12-09T10-15-30

# Méthode 2 : Via logrotate (Linux)
# /etc/logrotate.d/mongodb

/var/log/mongodb/*.log {
    daily
    rotate 7
    compress
    delaycompress
    notifempty
    missingok
    sharedscripts
    postrotate
        /bin/kill -SIGUSR1 $(cat /var/run/mongodb/mongod.pid 2>/dev/null) 2>/dev/null || true
    endscript
}
```

### Consulter les Logs en Temps Réel

```bash
# Suivre les logs
tail -f /var/log/mongodb/mongod.log

# Suivre avec highlighting
tail -f /var/log/mongodb/mongod.log | grep --color=always -E "ERROR|WARNING|FATAL|$"

# Suivre uniquement les erreurs
tail -f /var/log/mongodb/mongod.log | grep -E "E |F "

# Avec multitail (plusieurs fichiers)
multitail /var/log/mongodb/mongod.log /var/log/mongodb/mongos.log
```

---

## 3. Niveaux de Sévérité

### Classification

```
F - FATAL      : Erreur fatale, le processus va s'arrêter
E - ERROR      : Erreur qui affecte les opérations
W - WARNING    : Avertissement, problème potentiel
I - INFO       : Information générale
D - DEBUG      : Information de debug (verbosity > 0)
D1..D5         : Niveaux de debug détaillés
```

### Analyse par Sévérité

```bash
# Compter par niveau de sévérité
grep -E "^[0-9T:-]+ [FEWID]" /var/log/mongodb/mongod.log |
  cut -d' ' -f2 | sort | uniq -c

# Sortie exemple :
#   12345 I
#     234 W
#      45 E
#       2 F

# Extraire uniquement les FATAL
grep " F " /var/log/mongodb/mongod.log

# Extraire ERROR et FATAL
grep -E " [EF] " /var/log/mongodb/mongod.log

# Dernières 24 heures avec erreurs
grep -E " [EF] " /var/log/mongodb/mongod.log |
  grep "$(date +%Y-%m-%d)" | tail -50
```

---

## 4. Messages Critiques

### Messages FATAL

#### 1. Corruption WiredTiger

```
2024-12-09T10:15:30.123+0000 F STORAGE  [conn123] WiredTiger error (0) [1702118130:123456][12345:0x7f8a9c0b4700], file:collection-1--234567890.wt, WT_SESSION.checkpoint: __wt_block_read_off, 123: collection-1--234567890.wt: read checksum error
```

**Signification :** Corruption de fichier détectée (checksum invalide)

**Action :**
1. Arrêter MongoDB immédiatement
2. Vérifier l'intégrité du disque
3. Restaurer depuis backup ou réparer (voir section Corruption)

**Commandes de diagnostic :**

```bash
# Vérifier le disque
sudo smartctl -a /dev/sda | grep -i "error"

# Vérifier le filesystem
sudo fsck -n /dev/sda1

# Identifier les fichiers corrompus
cd /var/lib/mongodb
wt verify collection-1--234567890.wt
```

#### 2. Panique WiredTiger

```
2024-12-09T10:15:30.123+0000 F -        [conn456] Fatal Assertion 50853 at src/mongo/db/storage/wiredtiger/wiredtiger_util.cpp 123
2024-12-09T10:15:30.124+0000 F -        [conn456]
***aborting after fassert() failure
```

**Signification :** Erreur interne critique de WiredTiger

**Action :**
1. Analyser la stack trace complète dans les logs
2. Vérifier les bugs connus pour cette version
3. Contacter le support MongoDB avec les logs

#### 3. Out of Memory

```
2024-12-09T10:15:30.123+0000 F CONTROL  [initandlisten] Failed to allocate memory: Cannot allocate memory
```

**Signification :** Système à court de mémoire

**Action :**

```bash
# Vérifier la mémoire disponible
free -h

# Vérifier l'utilisation par MongoDB
ps aux | grep mongod | awk '{print $6}'

# Vérifier le swap
vmstat 1 5

# Solution temporaire : redimensionner le cache WiredTiger
# /etc/mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4  # Réduire
```

### Messages ERROR Critiques

#### 1. Cannot Connect to Replica Set

```
2024-12-09T10:15:30.123+0000 E REPL     [replication-0] Failed to connect to mongodb-secondary:27017, reason: Connection refused
```

**Diagnostic :**

```bash
# Vérifier que le membre est accessible
ping mongodb-secondary

# Tester le port MongoDB
telnet mongodb-secondary 27017

# Vérifier si MongoDB tourne sur le membre
ssh mongodb-secondary "systemctl status mongod"

# Vérifier les logs sur le membre distant
ssh mongodb-secondary "tail -50 /var/log/mongodb/mongod.log"
```

#### 2. Replication Lag

```
2024-12-09T10:15:30.123+0000 E REPL     [replication-0] Replica set member is more than 10 seconds behind the primary
```

**Diagnostic :**

```javascript
// Mesurer le lag
rs.printSecondaryReplicationInfo()

// Identifier la cause
db.currentOp({
  "active": true,
  "secs_running": {$gt: 10}
})

// Vérifier l'oplog
db.getReplicationInfo()
```

#### 3. Authentication Failed

```
2024-12-09T10:15:30.123+0000 E ACCESS   [conn789] SCRAM-SHA-256 authentication failed for user on database from client 192.168.1.100:54321; UserNotFound: Could not find user "username" for db "database"
```

**Diagnostic :**

```javascript
// Vérifier l'utilisateur
use admin
db.system.users.find({user: "username"})

// Lister tous les utilisateurs
db.system.users.find({}, {user: 1, db: 1, roles: 1})

// Vérifier les rôles
db.getUser("username")
```

#### 4. Index Build Failed

```
2024-12-09T10:15:30.123+0000 E INDEX    [conn123] Index build failed: DuplicateKey: duplicate key error collection: mydb.mycollection index: email_1 dup key: { email: "user@example.com" }
```

**Action :**

```javascript
// Identifier les doublons
db.mycollection.aggregate([
  {$group: {
    _id: "$email",
    count: {$sum: 1},
    ids: {$push: "$_id"}
  }},
  {$match: {count: {$gt: 1}}},
  {$sort: {count: -1}}
])

// Nettoyer les doublons avant de recréer l'index
```

### Messages WARNING Importants

#### 1. Connections Near Limit

```
2024-12-09T10:15:30.123+0000 W NETWORK  [listener] connection limit reached (950/1000), rejecting new connections
```

**Action :**

```javascript
// Vérifier les connexions actuelles
db.serverStatus().connections

// Identifier les clients
db.currentOp({"connectionId": {$exists: true}}).inprog.reduce((acc, op) => {
  var client = op.client || "unknown"
  acc[client] = (acc[client] || 0) + 1
  return acc
}, {})

// Augmenter la limite
db.adminCommand({setParameter: 1, maxIncomingConnections: 2000})
```

#### 2. Slow Query

```
2024-12-09T10:15:30.123+0000 W COMMAND  [conn123] command mydb.mycollection command: find { filter: { status: "active" } } planSummary: COLLSCAN keysExamined:0 docsExamined:1000000 cursorExhausted:1 numYields:7812 nreturned:50000 locks:{ Global: { acquireCount: { r: 15626 } }, Database: { acquireCount: { r: 7813 } }, Collection: { acquireCount: { r: 7813 } } } storage:{} 2543ms
```

**Diagnostic :**

```javascript
// Analyser la requête
db.mycollection.find({status: "active"}).explain("executionStats")

// COLLSCAN = index manquant
// Créer l'index approprié
db.mycollection.createIndex({status: 1})
```

---

## 5. Patterns d'Erreurs Courants

### Pattern 1 : Crash Loop

**Logs :**

```
2024-12-09T10:15:30.123+0000 I CONTROL  [initandlisten] MongoDB starting
2024-12-09T10:15:30.456+0000 F STORAGE  [initandlisten] Fatal error
2024-12-09T10:15:30.457+0000 I CONTROL  [initandlisten] now exiting

2024-12-09T10:15:35.123+0000 I CONTROL  [initandlisten] MongoDB starting
2024-12-09T10:15:35.456+0000 F STORAGE  [initandlisten] Fatal error
2024-12-09T10:15:35.457+0000 I CONTROL  [initandlisten] now exiting

# Répété...
```

**Diagnostic :**

```bash
# Identifier la fréquence
grep "MongoDB starting" /var/log/mongodb/mongod.log | tail -20

# Identifier l'erreur persistante
grep -A 5 "MongoDB starting" /var/log/mongodb/mongod.log |
  grep -E "ERROR|FATAL" | sort | uniq -c

# Empêcher le redémarrage automatique temporairement
sudo systemctl stop mongod
sudo systemctl disable mongod

# Tenter un démarrage manuel pour voir l'erreur complète
sudo -u mongodb mongod --config /etc/mongod.conf
```

### Pattern 2 : Élections Fréquentes

**Logs :**

```
2024-12-09T10:15:30.123+0000 I REPL     [replication-0] Starting election due to step down
2024-12-09T10:15:32.456+0000 I REPL     [replication-0] election succeeded, new PRIMARY elected
2024-12-09T10:16:15.789+0000 I REPL     [replication-0] Starting election due to step down
2024-12-09T10:16:17.012+0000 I REPL     [replication-0] election succeeded, new PRIMARY elected
```

**Analyse :**

```bash
# Compter les élections
grep "Starting election" /var/log/mongodb/mongod.log | wc -l

# Analyser les causes
grep -B 5 "Starting election" /var/log/mongodb/mongod.log |
  grep -E "heartbeat|timeout|unreachable"

# Timeline des élections
grep "Starting election\|election succeeded" /var/log/mongodb/mongod.log |
  tail -20
```

**Causes communes :**
- Problèmes réseau (heartbeat timeout)
- Primary surchargé
- Configuration des timeouts inadaptée

### Pattern 3 : Memory Pressure

**Logs :**

```
2024-12-09T10:15:30.123+0000 W STORAGE  [WTCheckpointThread] WiredTiger eviction server: cache pressure is high
2024-12-09T10:15:31.456+0000 W STORAGE  [WTCheckpointThread] WiredTiger eviction server: cache pressure is high
2024-12-09T10:15:32.789+0000 W STORAGE  [WTCheckpointThread] WiredTiger eviction server: cache pressure is high
```

**Analyse :**

```javascript
// Vérifier l'utilisation du cache
db.serverStatus().wiredTiger.cache

// Métriques importantes :
var cache = db.serverStatus().wiredTiger.cache
print("Cache max: " + cache["maximum bytes configured"])
print("Cache used: " + cache["bytes currently in the cache"])
print("Cache dirty: " + cache["tracked dirty bytes in the cache"])
print("Evictions: " + cache["pages evicted by application threads"])
```

**Action :**

```yaml
# Ajuster le cache WiredTiger
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # Augmenter
```

### Pattern 4 : Connection Storm

**Logs :**

```
2024-12-09T10:15:30.123+0000 I NETWORK  [listener] connection accepted from 192.168.1.100:54321 #12345
2024-12-09T10:15:30.125+0000 I NETWORK  [listener] connection accepted from 192.168.1.100:54322 #12346
2024-12-09T10:15:30.127+0000 I NETWORK  [listener] connection accepted from 192.168.1.100:54323 #12347
# ... 100+ connexions en quelques secondes
```

**Analyse :**

```bash
# Compter les connexions par seconde
grep "connection accepted" /var/log/mongodb/mongod.log |
  cut -d' ' -f1 | cut -d'.' -f1 | uniq -c

# Identifier les clients sources
grep "connection accepted" /var/log/mongodb/mongod.log |
  grep -oP 'from \K[0-9.]+' | sort | uniq -c | sort -rn

# Timeline des connexions
grep "connection accepted" /var/log/mongodb/mongod.log | tail -100
```

**Causes communes :**
- Pool de connexions mal configuré
- Application qui reconnecte en boucle
- Attaque DDoS

---

## 6. Outils d'Analyse

### Grep Avancé

```bash
# ═══════════════════════════════════════
# Patterns Utiles
# ═══════════════════════════════════════

# Toutes les erreurs des dernières 24h
grep -E " [EF] " /var/log/mongodb/mongod.log |
  grep "$(date +%Y-%m-%d)"

# Erreurs par composant
grep " E " /var/log/mongodb/mongod.log |
  awk '{print $3}' | sort | uniq -c | sort -rn

# Timeline des événements critiques
grep -E "FATAL|ERROR|election|shutdown|corruption" \
  /var/log/mongodb/mongod.log | tail -100

# Filtrer par plage horaire
awk '/2024-12-09T10:00/,/2024-12-09T11:00/' /var/log/mongodb/mongod.log |
  grep -E "ERROR|WARNING"

# Extraire les stack traces complètes
awk '/Fatal Assertion/,/^$/' /var/log/mongodb/mongod.log

# Connexions par IP
grep "connection accepted" /var/log/mongodb/mongod.log |
  grep -oP 'from \K[0-9.]+(?=:)' |
  sort | uniq -c | sort -rn | head -20
```

### AWK pour Analyse Avancée

```bash
# Analyser les temps de requête
awk '/command.*[0-9]+ms$/ {
  match($0, /([0-9]+)ms$/, time)
  if (time[1] > 1000) print $0
}' /var/log/mongodb/mongod.log

# Statistiques d'erreurs par heure
awk '/ E / {
  match($0, /^([0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2})/, hour)
  errors[hour[1]]++
}
END {
  for (h in errors) print h, errors[h]
}' /var/log/mongodb/mongod.log | sort

# Compter les opérations par type
awk '/command.*command:/ {
  match($0, /command: ([a-z]+)/, cmd)
  commands[cmd[1]]++
}
END {
  for (c in commands) print c, commands[c]
}' /var/log/mongodb/mongod.log | sort -k2 -rn
```

### Script Python d'Analyse

```python
#!/usr/bin/env python3
# mongodb_log_analyzer.py

import re
import sys
from collections import defaultdict, Counter
from datetime import datetime

class MongoLogAnalyzer:
    def __init__(self, logfile):
        self.logfile = logfile
        self.errors = []
        self.warnings = []
        self.slow_queries = []
        self.connections = []

    def parse_log(self):
        """Parse le fichier de log"""
        with open(self.logfile, 'r') as f:
            for line in f:
                self.analyze_line(line)

    def analyze_line(self, line):
        """Analyse une ligne de log"""
        # Extraire les composants
        match = re.match(
            r'(\S+)\s+([FEWID])\s+(\S+)\s+\[([^\]]+)\]\s+(.+)',
            line
        )

        if not match:
            return

        timestamp, severity, component, context, message = match.groups()

        # Catégoriser
        if severity == 'E':
            self.errors.append({
                'timestamp': timestamp,
                'component': component,
                'message': message
            })
        elif severity == 'W':
            self.warnings.append({
                'timestamp': timestamp,
                'component': component,
                'message': message
            })

        # Requêtes lentes
        if 'ms' in message and component == 'COMMAND':
            match = re.search(r'(\d+)ms$', message)
            if match and int(match.group(1)) > 1000:
                self.slow_queries.append({
                    'timestamp': timestamp,
                    'duration': int(match.group(1)),
                    'query': message
                })

        # Connexions
        if 'connection accepted' in message:
            match = re.search(r'from (\S+)', message)
            if match:
                self.connections.append(match.group(1))

    def generate_report(self):
        """Génère un rapport d'analyse"""
        print("=" * 80)
        print("MongoDB Log Analysis Report")
        print("=" * 80)
        print()

        # Résumé
        print("Summary:")
        print(f"  Total Errors: {len(self.errors)}")
        print(f"  Total Warnings: {len(self.warnings)}")
        print(f"  Slow Queries (>1s): {len(self.slow_queries)}")
        print(f"  Connections: {len(self.connections)}")
        print()

        # Top erreurs par composant
        if self.errors:
            print("Top Error Components:")
            components = Counter([e['component'] for e in self.errors])
            for comp, count in components.most_common(10):
                print(f"  {comp}: {count}")
            print()

        # Requêtes les plus lentes
        if self.slow_queries:
            print("Top 10 Slowest Queries:")
            sorted_queries = sorted(
                self.slow_queries,
                key=lambda x: x['duration'],
                reverse=True
            )
            for q in sorted_queries[:10]:
                print(f"  {q['duration']}ms: {q['query'][:80]}...")
            print()

        # Top connexions par IP
        if self.connections:
            print("Top Connection Sources:")
            ips = Counter(self.connections)
            for ip, count in ips.most_common(10):
                print(f"  {ip}: {count} connections")
            print()

        # Erreurs récentes
        if self.errors:
            print("Recent Errors (last 10):")
            for error in self.errors[-10:]:
                print(f"  [{error['timestamp']}] {error['component']}")
                print(f"    {error['message'][:120]}")
            print()

# Utilisation
if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python3 mongodb_log_analyzer.py <logfile>")
        sys.exit(1)

    analyzer = MongoLogAnalyzer(sys.argv[1])
    analyzer.parse_log()
    analyzer.generate_report()
```

**Utilisation :**

```bash
python3 mongodb_log_analyzer.py /var/log/mongodb/mongod.log
```

### mtools (Outils Communautaires)

```bash
# Installation
pip install mtools[mplotqueries]

# Analyser les logs
mloginfo /var/log/mongodb/mongod.log

# Sortie :
#      source: /var/log/mongodb/mongod.log
#        host: mongodb-server:27017
#       start: 2024 Dec 09 10:00:00.000
#         end: 2024 Dec 09 11:00:00.000
#   log length: 3600 lines
#      version: 6.0.x
#      storage: wiredTiger

# Filtrer par requêtes lentes
mlogfilter /var/log/mongodb/mongod.log --slow 1000 --json

# Visualiser les requêtes
mplotqueries /var/log/mongodb/mongod.log --type scatter

# Analyser les connexions
mlogfilter /var/log/mongodb/mongod.log --component NETWORK
```

---

## 7. Corrélation d'Événements

### Timeline d'Incident

```bash
#!/bin/bash
# Créer une timeline d'un incident

LOGFILE="/var/log/mongodb/mongod.log"
START_TIME="2024-12-09T10:00:00"
END_TIME="2024-12-09T11:00:00"

echo "=== Incident Timeline ==="
echo "Period: $START_TIME to $END_TIME"
echo ""

# Extraire tous les événements pertinents
awk -v start="$START_TIME" -v end="$END_TIME" '
    $1 >= start && $1 <= end {
        # Événements critiques
        if ($2 ~ /[EFW]/ ||
            /election/ ||
            /step down/ ||
            /slow query/ ||
            /connection/ ||
            /shutdown/ ||
            /starting/) {
            print $0
        }
    }
' "$LOGFILE" | sort

# Résumé par minute
echo ""
echo "=== Events per Minute ==="
awk -v start="$START_TIME" -v end="$END_TIME" '
    $1 >= start && $1 <= end {
        minute = substr($1, 1, 16)  # YYYY-MM-DDTHH:MM
        events[minute]++

        if ($2 == "E") errors[minute]++
        if ($2 == "W") warnings[minute]++
    }
    END {
        for (m in events) {
            printf "%s: %3d events", m, events[m]
            if (errors[m]) printf ", %d errors", errors[m]
            if (warnings[m]) printf ", %d warnings", warnings[m]
            print ""
        }
    }
' "$LOGFILE" | sort
```

### Corrélation avec Métriques Système

```bash
#!/bin/bash
# Corréler logs MongoDB avec métriques système

# Timestamp de l'incident
INCIDENT_TIME="2024-12-09T10:15:30"

# Convertir en epoch pour faciliter la recherche
INCIDENT_EPOCH=$(date -d "$INCIDENT_TIME" +%s)
START_EPOCH=$((INCIDENT_EPOCH - 300))  # 5 min avant
END_EPOCH=$((INCIDENT_EPOCH + 300))    # 5 min après

echo "=== System Metrics Around Incident ==="
echo "Incident time: $INCIDENT_TIME"
echo ""

# CPU usage (from sar if available)
if command -v sar &> /dev/null; then
    echo "CPU Usage:"
    sar -u -s $(date -d @$START_EPOCH +%H:%M:%S) \
        -e $(date -d @$END_EPOCH +%H:%M:%S)
fi

# Memory (from sar or vmstat logs)
if command -v sar &> /dev/null; then
    echo ""
    echo "Memory Usage:"
    sar -r -s $(date -d @$START_EPOCH +%H:%M:%S) \
        -e $(date -d @$END_EPOCH +%H:%M:%S)
fi

# Disk I/O (from sar)
if command -v sar &> /dev/null; then
    echo ""
    echo "Disk I/O:"
    sar -d -s $(date -d @$START_EPOCH +%H:%M:%S) \
        -e $(date -d @$END_EPOCH +%H:%M:%S)
fi

# Network (from sar)
if command -v sar &> /dev/null; then
    echo ""
    echo "Network:"
    sar -n DEV -s $(date -d @$START_EPOCH +%H:%M:%S) \
        -e $(date -d @$END_EPOCH +%H:%M:%S)
fi

# MongoDB logs pendant la même période
echo ""
echo "=== MongoDB Logs During Incident ==="
awk -v start="$(date -d @$START_EPOCH +%Y-%m-%dT%H:%M:%S)" \
    -v end="$(date -d @$END_EPOCH +%Y-%m-%dT%H:%M:%S)" '
    $1 >= start && $1 <= end
' /var/log/mongodb/mongod.log | grep -E "ERROR|WARNING|FATAL"
```

### Agrégation Multi-Serveurs

```bash
#!/bin/bash
# Agréger les logs de plusieurs serveurs MongoDB

SERVERS=(
    "mongodb-primary"
    "mongodb-secondary1"
    "mongodb-secondary2"
)

TIMEFRAME="2024-12-09T10:15"
OUTPUT="/tmp/mongodb-aggregate-logs.txt"

echo "=== Aggregating Logs from ${#SERVERS[@]} servers ===" > "$OUTPUT"
echo "Timeframe: $TIMEFRAME" >> "$OUTPUT"
echo "" >> "$OUTPUT"

for server in "${SERVERS[@]}"; do
    echo "=== $server ===" >> "$OUTPUT"

    ssh "$server" "grep '$TIMEFRAME' /var/log/mongodb/mongod.log | \
                   grep -E 'ERROR|WARNING|election|step down'" \
    >> "$OUTPUT" 2>&1

    echo "" >> "$OUTPUT"
done

# Trier par timestamp global
cat "$OUTPUT" | grep -E "^[0-9]{4}-" | sort

echo "Aggregated logs saved to: $OUTPUT"
```

---

## 8. Automatisation et Alertes

### Script de Monitoring Continu

```bash
#!/bin/bash
# mongodb_log_monitor.sh - Monitoring en temps réel

LOGFILE="/var/log/mongodb/mongod.log"
ALERT_EMAIL="admin@example.com"
STATE_FILE="/tmp/mongodb_log_monitor.state"

# Charger la dernière position
if [ -f "$STATE_FILE" ]; then
    LAST_LINE=$(cat "$STATE_FILE")
else
    LAST_LINE=0
fi

# Patterns critiques à surveiller
declare -A CRITICAL_PATTERNS=(
    ["FATAL"]="CRITICAL: Fatal error detected"
    ["WiredTiger error"]="CRITICAL: WiredTiger error"
    ["corruption"]="CRITICAL: Data corruption detected"
    ["out of memory"]="CRITICAL: Out of memory"
)

declare -A WARNING_PATTERNS=(
    ["Slow query"]="WARNING: Slow query detected"
    ["cache pressure"]="WARNING: Cache pressure high"
    ["too many open connections"]="WARNING: Connection limit near"
    ["election"]="WARNING: Replica set election"
)

# Analyser les nouvelles lignes
NEW_CONTENT=$(tail -n +$((LAST_LINE + 1)) "$LOGFILE")
CURRENT_LINE=$(wc -l < "$LOGFILE")

# Vérifier les patterns critiques
for pattern in "${!CRITICAL_PATTERNS[@]}"; do
    if echo "$NEW_CONTENT" | grep -q "$pattern"; then
        MESSAGE="${CRITICAL_PATTERNS[$pattern]}"
        DETAILS=$(echo "$NEW_CONTENT" | grep "$pattern" | tail -5)

        # Envoyer alerte
        echo "$MESSAGE" | mail -s "MongoDB CRITICAL Alert" "$ALERT_EMAIL"

        # Log
        logger -t mongodb_monitor "CRITICAL: $pattern detected in MongoDB logs"

        # Webhook (exemple)
        curl -X POST https://alerts.example.com/webhook \
            -H "Content-Type: application/json" \
            -d "{
                \"severity\": \"critical\",
                \"service\": \"mongodb\",
                \"message\": \"$MESSAGE\",
                \"details\": \"$DETAILS\"
            }" 2>/dev/null
    fi
done

# Vérifier les patterns warning
for pattern in "${!WARNING_PATTERNS[@]}"; do
    COUNT=$(echo "$NEW_CONTENT" | grep -c "$pattern")
    if [ $COUNT -gt 0 ]; then
        MESSAGE="${WARNING_PATTERNS[$pattern]} (occurred $COUNT times)"

        # Envoyer alerte (throttled)
        echo "$MESSAGE" | mail -s "MongoDB WARNING Alert" "$ALERT_EMAIL"
    fi
done

# Sauvegarder la position
echo "$CURRENT_LINE" > "$STATE_FILE"
```

**Déploiement :**

```bash
# Rendre exécutable
chmod +x /usr/local/bin/mongodb_log_monitor.sh

# Ajouter à cron (toutes les 5 minutes)
echo "*/5 * * * * /usr/local/bin/mongodb_log_monitor.sh" | crontab -
```

### Intégration avec ELK Stack

```yaml
# filebeat.yml - Envoyer les logs MongoDB vers Elasticsearch

filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/mongodb/mongod.log

  # Parser le format MongoDB
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}T'
  multiline.negate: true
  multiline.match: after

  # Tags
  tags: ["mongodb", "production"]

  # Fields additionnels
  fields:
    server: "mongodb-primary"
    datacenter: "dc1"

# Processors
processors:
  - dissect:
      tokenizer: "%{timestamp} %{severity} %{component} [%{context}] %{message}"
      field: "message"
      target_prefix: "mongodb"

  - timestamp:
      field: mongodb.timestamp
      layouts:
        - '2006-01-02T15:04:05.999Z0700'

# Output vers Logstash
output.logstash:
  hosts: ["logstash:5044"]
```

**Dashboard Kibana (queries) :**

```
# Erreurs par heure
mongodb.severity: "E" OR mongodb.severity: "F"

# Requêtes lentes
mongodb.component: "COMMAND" AND mongodb.message: *ms AND mongodb.duration > 1000

# Élections
mongodb.message: "election"

# Top composants avec erreurs
mongodb.severity: "E"
Aggregation: Terms on mongodb.component
```

### Alertes avec Prometheus

```yaml
# prometheus_alerts.yml

groups:
- name: mongodb_logs
  rules:

  # Taux d'erreur élevé
  - alert: HighErrorRate
    expr: rate(mongodb_log_errors_total[5m]) > 10
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High MongoDB error rate"
      description: "MongoDB error rate is {{ $value }} errors/sec"

  # Erreurs fatales
  - alert: FatalError
    expr: increase(mongodb_log_fatal_total[1m]) > 0
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: "MongoDB fatal error detected"
      description: "Fatal error in MongoDB logs"

  # Requêtes lentes fréquentes
  - alert: FrequentSlowQueries
    expr: rate(mongodb_slow_queries_total[5m]) > 5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Frequent slow queries"
      description: "{{ $value }} slow queries per second"
```

---

## Checklist d'Analyse de Logs

```
INCIDENT: Problème MongoDB détecté

☐ COLLECTE INITIALE (2 minutes)
  ☐ Capturer les dernières 1000 lignes de logs
  ☐ Identifier le timestamp de début
  ☐ Vérifier si logs rotatés récemment
  ☐ Collecter logs de tous les membres (RS/Sharded)

☐ FILTRAGE PAR SÉVÉRITÉ (3 minutes)
  ☐ Extraire tous les FATAL
  ☐ Extraire tous les ERROR
  ☐ Compter les WARNING
  ☐ Timeline des événements critiques

☐ ANALYSE PAR COMPOSANT (5 minutes)
  ☐ Identifier les composants affectés
  ☐ REPL → Problème de réplication ?
  ☐ STORAGE → Problème de stockage ?
  ☐ NETWORK → Problème réseau ?
  ☐ COMMAND → Problème de requêtes ?

☐ PATTERNS D'ERREURS (5 minutes)
  ☐ Erreurs répétées ?
  ☐ Séquence d'événements ?
  ☐ Corrélation temporelle ?
  ☐ Messages connus (bugs, known issues) ?

☐ CORRÉLATION SYSTÈME (5 minutes)
  ☐ CPU usage au même moment ?
  ☐ Memory pressure ?
  ☐ Disk I/O issues ?
  ☐ Network problems ?

☐ CONTEXTE APPLICATION (3 minutes)
  ☐ Déploiement récent ?
  ☐ Changement de charge ?
  ☐ Requêtes inhabituelles ?
  ☐ Erreurs applicatives ?

☐ DOCUMENTATION (2 minutes)
  ☐ Capturer les logs pertinents
  ☐ Noter les timestamps clés
  ☐ Identifier les actions entreprises
  ☐ Préparer pour escalade si nécessaire
```

---

## Conclusion

L'analyse efficace des logs MongoDB nécessite :

1. **Compréhension du format** : Structure, composants, sévérité
2. **Outils appropriés** : grep, awk, scripts, mtools
3. **Patterns reconnus** : Erreurs communes et leurs causes
4. **Corrélation** : Logs + métriques + événements
5. **Automatisation** : Monitoring continu, alertes proactives

**Bonnes pratiques :**
- ✅ **Centraliser les logs** (ELK, Splunk, etc.)
- ✅ **Rotation automatique** (éviter saturation disque)
- ✅ **Rétention appropriée** (30-90 jours minimum)
- ✅ **Monitoring proactif** (alertes sur patterns critiques)
- ✅ **Documentation des incidents** (logs + actions + résolution)
- ✅ **Verbosité adaptative** (augmenter temporairement pour debug)
- ✅ **Backup des logs** (incidents historiques)

**Ressources :**
- Documentation MongoDB : https://docs.mongodb.com/manual/reference/log-messages/
- mtools : https://github.com/rueckstiess/mtools
- Log format : https://docs.mongodb.com/manual/reference/log-format/

---


⏭️ [Support MongoDB et ressources](/22-depannage-resolution-problemes/08-support-mongodb-ressources.md)
