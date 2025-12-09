🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 22.6 Récupération après Panne

## Vue d'ensemble

La récupération après panne est une compétence critique pour assurer la disponibilité et la continuité de service MongoDB. Cette section couvre les procédures complètes de récupération pour tous les types de déploiements, des serveurs standalone aux clusters shardés complexes.

---

## Table des Matières

1. [Types de Pannes](#1-types-de-pannes)
2. [Procédures de Démarrage](#2-proc%C3%A9dures-de-d%C3%A9marrage)
3. [Récupération Serveur Standalone](#3-r%C3%A9cup%C3%A9ration-serveur-standalone)
4. [Récupération Replica Set](#4-r%C3%A9cup%C3%A9ration-replica-set)
5. [Récupération Cluster Shardé](#5-r%C3%A9cup%C3%A9ration-cluster-shard%C3%A9)
6. [Disaster Recovery](#6-disaster-recovery)
7. [Failover et Bascule](#7-failover-et-bascule)
8. [Tests et Validation](#8-tests-et-validation)

---

## 1. Types de Pannes

### Classification des Pannes

#### 1.1 Panne d'Arrêt Propre

**Caractéristiques :**
- MongoDB arrêté correctement (shutdown command)
- Journal complet et cohérent
- Données cohérentes sur disque
- Redémarrage simple

**Signes dans les logs :**
```
[timestamp] I CONTROL  [conn1] received shutdown command
[timestamp] I CONTROL  [conn1] shutting down
[timestamp] I STORAGE  [conn1] WiredTigerKVEngine shutting down
[timestamp] I STORAGE  [conn1] shutdown: closing all files...
[timestamp] I STORAGE  [conn1] shutdown: removing fs lock...
[timestamp] I CONTROL  [conn1] now exiting
[timestamp] I CONTROL  [conn1] shutdown: going to close listening sockets...
[timestamp] I CONTROL  [conn1] shutdown complete
```

#### 1.2 Panne d'Arrêt Brutal

**Causes :**
- Arrêt forcé (kill -9)
- Panne électrique
- Crash système
- Crash matériel

**Caractéristiques :**
- Journal potentiellement incomplet
- Nécessite récupération du journal
- Peut nécessiter réparation

**Signes dans les logs :**
```
[timestamp] W STORAGE  [initandlisten] Detected unclean shutdown
[timestamp] I STORAGE  [initandlisten] Recovering data from the last clean checkpoint
[timestamp] I STORAGE  [initandlisten] WiredTiger message: recovery checkpoint
[timestamp] I STORAGE  [initandlisten] WiredTiger recovery in progress
```

#### 1.3 Panne Matérielle

**Types :**
- Défaillance disque
- Défaillance mémoire
- Défaillance réseau
- Défaillance CPU/carte mère

**Impacts :**
- Perte potentielle de données
- Nécessite changement matériel
- Peut nécessiter restauration depuis backup

#### 1.4 Panne Réseau

**Types :**
- Partition réseau
- Perte de connectivité
- Latence élevée
- Saturation bande passante

**Impacts :**
- Élections dans replica set
- Problèmes de réplication
- Timeouts d'application

#### 1.5 Panne de Datacenter

**Scénarios :**
- Panne électrique totale
- Catastrophe naturelle
- Incendie
- Inondation

**Impacts :**
- Perte complète d'un site
- Nécessite disaster recovery
- Bascule vers site secondaire

---

## 2. Procédures de Démarrage

### Démarrage Standard

#### Étape 1 : Vérifications Pré-Démarrage

```bash
#!/bin/bash
# Script de vérification pré-démarrage

echo "=== MongoDB Pre-Start Checks ==="

# 1. Vérifier que mongod n'est pas déjà en cours d'exécution
if pgrep -x mongod > /dev/null; then
    echo "✗ MongoDB is already running"
    exit 1
else
    echo "✓ No mongod process found"
fi

# 2. Vérifier l'espace disque
DISK_USAGE=$(df -h /var/lib/mongodb | awk 'NR==2 {print $5}' | sed 's/%//')
if [ $DISK_USAGE -gt 90 ]; then
    echo "⚠  WARNING: Disk usage is ${DISK_USAGE}%"
else
    echo "✓ Disk usage OK: ${DISK_USAGE}%"
fi

# 3. Vérifier les permissions
if [ "$(stat -c '%U:%G' /var/lib/mongodb)" != "mongodb:mongodb" ]; then
    echo "✗ Incorrect ownership on /var/lib/mongodb"
    echo "  Running: chown -R mongodb:mongodb /var/lib/mongodb"
    sudo chown -R mongodb:mongodb /var/lib/mongodb
else
    echo "✓ Ownership correct"
fi

# 4. Vérifier l'existence du fichier de configuration
if [ ! -f /etc/mongod.conf ]; then
    echo "✗ Configuration file not found: /etc/mongod.conf"
    exit 1
else
    echo "✓ Configuration file exists"
fi

# 5. Vérifier la syntaxe de la configuration
mongod --config /etc/mongod.conf --validate > /dev/null 2>&1
if [ $? -eq 0 ]; then
    echo "✓ Configuration is valid"
else
    echo "✗ Configuration is invalid"
    exit 1
fi

# 6. Vérifier le lock file
if [ -f /var/lib/mongodb/mongod.lock ]; then
    if [ -s /var/lib/mongodb/mongod.lock ]; then
        echo "⚠  WARNING: Non-empty lock file found"
        echo "  This may indicate an unclean shutdown"
    else
        echo "✓ Empty lock file (normal)"
    fi
fi

# 7. Vérifier les logs précédents
if [ -f /var/log/mongodb/mongod.log ]; then
    LAST_SHUTDOWN=$(grep "shutdown complete" /var/log/mongodb/mongod.log | tail -1)
    if [ -n "$LAST_SHUTDOWN" ]; then
        echo "✓ Previous shutdown was clean"
    else
        echo "⚠  WARNING: Previous shutdown may have been unclean"
    fi
fi

echo ""
echo "Pre-start checks complete"
echo "Ready to start MongoDB"
```

#### Étape 2 : Démarrage Normal

```bash
# Démarrage via systemd (recommandé)
sudo systemctl start mongod

# Vérifier le démarrage
sudo systemctl status mongod

# Suivre les logs en temps réel
tail -f /var/log/mongodb/mongod.log

# Messages attendus :
# [timestamp] I CONTROL  [initandlisten] MongoDB starting
# [timestamp] I CONTROL  [initandlisten] db version v6.0.x
# [timestamp] I CONTROL  [initandlisten] OpenSSL version: ...
# [timestamp] I STORAGE  [initandlisten] WiredTiger opened
# [timestamp] I NETWORK  [listener] waiting for connections on port 27017
```

#### Étape 3 : Validation du Démarrage

```javascript
// Se connecter à MongoDB
mongosh

// Vérifier la connectivité
db.adminCommand({ping: 1})
// {ok: 1}

// Vérifier le statut du serveur
db.serverStatus()

// Vérifier les bases de données
show dbs

// Pour un replica set, vérifier l'état
rs.status()

// Vérifier les opérations en cours
db.currentOp()

// Vérifier les connexions
db.serverStatus().connections
```

---

## 3. Récupération Serveur Standalone

### Scénario 1 : Arrêt Propre - Redémarrage Simple

```bash
# Simple redémarrage
sudo systemctl start mongod

# Vérifier les logs
tail -50 /var/log/mongodb/mongod.log

# Tester la connexion
mongosh --eval "db.adminCommand({ping: 1})"
```

### Scénario 2 : Arrêt Brutal - Récupération du Journal

```bash
# 1. Tenter un démarrage normal
sudo systemctl start mongod

# 2. Surveiller les logs pour la récupération
tail -f /var/log/mongodb/mongod.log

# Messages de récupération :
# [timestamp] I STORAGE  [initandlisten] Detected unclean shutdown
# [timestamp] I STORAGE  [initandlisten] Recovering from the journal
# [timestamp] I STORAGE  [initandlisten] Journal playback complete

# 3. Si le démarrage échoue, vérifier les logs d'erreur
grep -i "error\|fatal\|exception" /var/log/mongodb/mongod.log | tail -20

# 4. Si corruption détectée, voir section Corruption de Données
```

### Scénario 3 : Échec de Démarrage - Mode Debug

```bash
# 1. Arrêter le service
sudo systemctl stop mongod

# 2. Démarrer manuellement en mode verbose
sudo -u mongodb mongod \
  --config /etc/mongod.conf \
  --verbose \
  --logpath /tmp/mongod-debug.log

# 3. Analyser les logs détaillés
tail -f /tmp/mongod-debug.log

# 4. Si problème identifié et résolu, redémarrer normalement
sudo systemctl start mongod
```

### Scénario 4 : Récupération avec --repair

**⚠️ Dernier recours - peut causer perte de données**

```bash
# 1. Arrêter MongoDB
sudo systemctl stop mongod

# 2. Sauvegarder l'état actuel
sudo cp -r /var/lib/mongodb /var/lib/mongodb.backup.$(date +%Y%m%d_%H%M%S)

# 3. Vérifier l'espace disque (besoin de 1.5x la taille des données)
df -h /var/lib/mongodb

# 4. Lancer la réparation
sudo -u mongodb mongod \
  --dbpath /var/lib/mongodb \
  --repair \
  --logpath /tmp/mongod-repair.log

# 5. Surveiller la progression
tail -f /tmp/mongod-repair.log

# 6. Une fois terminé, redémarrer normalement
sudo systemctl start mongod

# 7. Valider les données
mongosh --eval "
  db.adminCommand({listDatabases: 1}).databases.forEach(function(dbInfo) {
    if (!['admin', 'local', 'config'].includes(dbInfo.name)) {
      print('Validating database: ' + dbInfo.name);
      var db = db.getSiblingDB(dbInfo.name);
      db.getCollectionNames().forEach(function(coll) {
        var result = db[coll].validate();
        print('  ' + coll + ': ' + (result.valid ? 'OK' : 'INVALID'));
      });
    }
  });
"
```

---

## 4. Récupération Replica Set

### Architecture et Points de Défaillance

```
┌────────────────────────────────────────┐
│         REPLICA SET (rs0)              │
├────────────────────────────────────────┤
│                                        │
│   ┌──────────┐    ┌──────────┐         │
│   │ PRIMARY  │───▶│SECONDARY1│         │
│   │  Node1   │    │  Node2   │         │
│   └──────────┘    └──────────┘         │
│        │               │               │
│        │               │               │
│        ▼               ▼               │
│   ┌──────────┐    ┌──────────┐         │
│   │SECONDARY2│    │ ARBITER  │         │
│   │  Node3   │    │  Node4   │         │
│   └──────────┘    └──────────┘         │
│                                        │
└────────────────────────────────────────┘

Points de défaillance :
1. Primary down → Élection automatique
2. Secondary down → Réplication continue
3. Majority down → Pas de primary (read-only)
4. Tous down → Récupération complète
```

### Scénario 1 : Primary Down - Élection Automatique

```javascript
// Le replica set gère automatiquement le failover

// 1. Vérifier l'état du replica set
rs.status()

// Sortie attendue :
// Un secondary devient primary automatiquement
// {
//   "members": [
//     {"name": "node1:27017", "stateStr": "DOWN"},
//     {"name": "node2:27017", "stateStr": "PRIMARY"},  // ← Nouveau primary
//     {"name": "node3:27017", "stateStr": "SECONDARY"}
//   ]
// }

// 2. Récupérer le nœud down
// Sur node1 :
sudo systemctl start mongod

// 3. Le nœud rejoindra automatiquement comme secondary
// Vérifier
rs.status()
```

### Scénario 2 : Un Secondary Down - Récupération Simple

```bash
# 1. Sur le secondary down
sudo systemctl start mongod

# 2. Vérifier les logs
tail -f /var/log/mongodb/mongod.log

# Messages attendus :
# [timestamp] I REPL     [replication-0] Starting replication fetcher
# [timestamp] I REPL     [replication-0] Syncing from: node1:27017

# 3. Vérifier le catch-up
mongosh --eval "rs.printSecondaryReplicationInfo()"

# Sortie :
# source: node2:27017
#   syncedTo: ...
#   0 secs (0 hrs) behind the primary
```

### Scénario 3 : Majority Down - Pas de Primary

```javascript
// Situation : Seulement 1 nœud sur 3 disponible
// = Pas de majorité = Pas de primary élu

// 1. Vérifier l'état
rs.status()

// Sortie :
// {
//   "members": [
//     {"name": "node1:27017", "stateStr": "SECONDARY"},  // Pas PRIMARY !
//     {"name": "node2:27017", "stateStr": "UNKNOWN"},
//     {"name": "node3:27017", "stateStr": "UNKNOWN"}
//   ]
// }

// 2. Récupérer les nœuds down (priorité)
// Sur node2 et node3
sudo systemctl start mongod

// 3. Une fois la majorité revenue, élection automatique
// Vérifier
rs.status()

// ALTERNATIVE : Forcer une reconfiguration (DANGER)
// ⚠️ Seulement si impossible de récupérer les autres nœuds
// ⚠️ Peut causer perte de données

// Sur le nœud survivant
cfg = rs.conf()
cfg.members = [cfg.members[0]]  // Garder seulement ce nœud
cfg.version++
rs.reconfig(cfg, {force: true})

// Le nœud devient primary immédiatement
```

### Scénario 4 : Tous les Nœuds Down - Récupération Complète

```bash
#!/bin/bash
# Procédure de récupération complète d'un replica set

echo "=== Replica Set Full Recovery ==="

# 1. Identifier le nœud avec les données les plus récentes
echo "Step 1: Checking all nodes for latest data"

for node in node1 node2 node3; do
  echo "Checking $node..."
  ssh $node "ls -la /var/lib/mongodb/journal/"
  ssh $node "grep 'shutdown' /var/log/mongodb/mongod.log | tail -1"
done

# Le nœud avec le journal le plus récent = données les plus récentes

# 2. Démarrer le nœud avec les données les plus récentes
echo "Step 2: Starting node with latest data"
LATEST_NODE="node1"  # Remplacer par le nœud identifié

ssh $LATEST_NODE "sudo systemctl start mongod"

# Attendre le démarrage
sleep 10

# 3. Vérifier que le nœud démarre en standalone ou SECONDARY
ssh $LATEST_NODE "mongosh --eval 'db.adminCommand({isMaster: 1})'"

# 4. Si le nœud ne peut pas atteindre les autres membres,
#    forcer une reconfiguration temporaire
ssh $LATEST_NODE "mongosh" <<'EOF'
cfg = rs.conf()
cfg.members = cfg.members.filter(m => m.host === "node1:27017")
cfg.version++
rs.reconfig(cfg, {force: true})
EOF

# 5. Démarrer les autres nœuds
echo "Step 3: Starting remaining nodes"
for node in node2 node3; do
  ssh $node "sudo systemctl start mongod"
done

# 6. Ajouter les nœuds au replica set
ssh $LATEST_NODE "mongosh" <<'EOF'
// Attendre que le nœud soit PRIMARY
while(rs.status().myState !== 1) {
  print("Waiting for PRIMARY state...")
  sleep(2000)
}

// Ajouter les autres membres
rs.add("node2:27017")
rs.add("node3:27017")

// Vérifier
rs.status()
EOF

echo "Recovery complete"
```

### Scénario 5 : Oplog Trop Court - Resync Complet

```javascript
// Un secondary ne peut pas rattraper (oplog trop court)

// 1. Identifier le problème
rs.printSecondaryReplicationInfo()

// Sortie :
// source: node2:27017
//   syncedTo: Thu Dec 07 2024 10:00:00 GMT+0000
//   ERROR: too stale to catch up

// 2. Sur le secondary problématique
use admin
db.shutdownServer()

// 3. Supprimer les données
// bash:
sudo rm -rf /var/lib/mongodb/*
# GARDER mongod.conf !

// 4. Redémarrer
sudo systemctl start mongod

// 5. Initial sync automatique depuis le primary
// Surveiller les logs
tail -f /var/log/mongodb/mongod.log

// Messages attendus :
// [timestamp] I REPL     [replication-0] Starting initial sync
// [timestamp] I REPL     [replication-0] Cloning all databases
// [timestamp] I REPL     [replication-0] Initial sync done

// 6. Vérifier
rs.status()
```

---

## 5. Récupération Cluster Shardé

### Architecture et Points de Défaillance

```
┌────────────────────────────────────────────────┐
│              SHARDED CLUSTER                   │
├────────────────────────────────────────────────┤
│                                                │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│   │ MONGOS  │  │ MONGOS  │  │ MONGOS  │        │
│   │  App1   │  │  App2   │  │  App3   │        │
│   └────┬────┘  └────┬────┘  └────┬────┘        │
│        │            │            │             │
│        └────────────┼────────────┘             │
│                     │                          │
│              ┌──────┴──────┐                   │
│              │             │                   │
│         ┌────▼────┐   ┌────▼───┐               │
│         │ CONFIG  │   │ CONFIG │               │
│         │ SERVER  │   │ SERVER │               │
│         │  (RS)   │   │  (RS)  │               │
│         └────┬────┘   └───┬────┘               │
│              │            │                    │
│         ┌────▼────────────▼─────┐              │
│         │                       │              │
│    ┌────▼────┐  ┌────────┐  ┌───▼────┐         │
│    │ SHARD 1 │  │ SHARD 2│  │ SHARD 3│         │
│    │  (RS)   │  │  (RS)  │  │  (RS)  │         │
│    └─────────┘  └────────┘  └────────┘         │
│                                                │
└────────────────────────────────────────────────┘

Points de défaillance critiques :
1. Config Servers → Métadonnées inaccessibles
2. Tous les Mongos → Applications ne peuvent pas router
3. Un Shard → Données de ce shard inaccessibles
4. Balancer → Distribution bloquée (non critique)
```

### Scénario 1 : Mongos Down

```bash
# Mongos est stateless - redémarrage simple

# 1. Sur le serveur mongos
sudo systemctl start mongos

# 2. Vérifier les logs
tail -50 /var/log/mongodb/mongos.log

# Messages attendus :
# [timestamp] I SHARDING [Balancer] about to contact config servers
# [timestamp] I SHARDING [Balancer] config servers and shards contacted successfully
# [timestamp] I NETWORK  [listener] waiting for connections on port 27017

# 3. Tester la connexion
mongosh --host mongos-server:27017 --eval "sh.status()"

# Si plusieurs mongos, les autres prennent le relais automatiquement
```

### Scénario 2 : Config Server Replica Set - Récupération

```bash
# Config servers sont critiques - contiennent toutes les métadonnées

# 1. Vérifier l'état du config server replica set
mongosh --host config-server:27019 --eval "rs.status()"

# 2. Si un membre down, récupération standard replica set
ssh config-server-1 "sudo systemctl start mongod"

# 3. Si majority down, récupération comme replica set normal
# Voir section "Récupération Replica Set"

# 4. Vérifier que mongos peut reconnecter
mongosh --host mongos-server:27017 --eval "sh.status()"
```

### Scénario 3 : Un Shard Down

```javascript
// Un shard (replica set) est down

// 1. Identifier le shard down
sh.status()

// Sortie :
// shards:
//   { "_id" : "shard0000", "host" : "shard0000/node1:27017,node2:27017" }
//   { "_id" : "shard0001", "host" : "shard0001/node1:27017,node2:27017" }  // DOWN
//   { "_id" : "shard0002", "host" : "shard0002/node1:27017,node2:27017" }

// 2. Récupérer le shard (procédure replica set)
// Sur les nœuds du shard0001
// bash: sudo systemctl start mongod

// 3. Vérifier le statut depuis mongos
sh.status()

// 4. Les données du shard redeviennent accessibles
// Les applications peuvent continuer
```

### Scénario 4 : Cluster Complet Down - Récupération Ordonnée

```bash
#!/bin/bash
# Procédure de récupération complète d'un cluster shardé

echo "=== Sharded Cluster Full Recovery ==="

# ORDRE CRITIQUE :
# 1. Config Servers (PREMIER)
# 2. Shards
# 3. Mongos (DERNIER)

# ═══════════════════════════════════════
# Phase 1 : Config Servers
# ═══════════════════════════════════════
echo "Phase 1: Starting Config Servers..."

# Démarrer le premier config server
ssh config-server-1 "sudo systemctl start mongod"
sleep 10

# Vérifier qu'il démarre
ssh config-server-1 "mongosh --port 27019 --eval 'db.adminCommand({ping: 1})'"

# Démarrer les autres config servers
for server in config-server-2 config-server-3; do
  ssh $server "sudo systemctl start mongod"
done

# Attendre que le replica set forme une majorité
sleep 20

# Vérifier le config replica set
ssh config-server-1 "mongosh --port 27019 --eval 'rs.status()'" | grep "PRIMARY"

if [ $? -ne 0 ]; then
  echo "ERROR: Config servers did not elect a PRIMARY"
  exit 1
fi

echo "✓ Config Servers are up"

# ═══════════════════════════════════════
# Phase 2 : Shards
# ═══════════════════════════════════════
echo "Phase 2: Starting Shards..."

for shard in shard0000 shard0001 shard0002; do
  echo "Starting $shard..."

  # Démarrer tous les nœuds du shard
  for node in node1 node2 node3; do
    ssh ${shard}-${node} "sudo systemctl start mongod"
  done

  sleep 10

  # Vérifier le shard
  ssh ${shard}-node1 "mongosh --eval 'rs.status()'" | grep "PRIMARY"

  if [ $? -eq 0 ]; then
    echo "✓ $shard is up"
  else
    echo "⚠ $shard may have issues"
  fi
done

# ═══════════════════════════════════════
# Phase 3 : Mongos
# ═══════════════════════════════════════
echo "Phase 3: Starting Mongos routers..."

for mongos in mongos-1 mongos-2 mongos-3; do
  ssh $mongos "sudo systemctl start mongos"
done

sleep 10

# ═══════════════════════════════════════
# Phase 4 : Validation
# ═══════════════════════════════════════
echo "Phase 4: Validating cluster..."

# Tester depuis mongos
ssh mongos-1 "mongosh --eval 'sh.status()'" > /tmp/cluster-status.txt

# Vérifier que tous les shards sont présents
for shard in shard0000 shard0001 shard0002; do
  grep $shard /tmp/cluster-status.txt > /dev/null
  if [ $? -eq 0 ]; then
    echo "✓ $shard is visible from mongos"
  else
    echo "✗ $shard is NOT visible from mongos"
  fi
done

echo ""
echo "Cluster recovery complete"
echo "Full status:"
cat /tmp/cluster-status.txt
```

### Scénario 5 : Config Server Corruption - Restauration Metadata

```bash
# Si les config servers sont corrompus, restaurer depuis backup

# 1. Arrêter tous les mongos
for mongos in mongos-1 mongos-2 mongos-3; do
  ssh $mongos "sudo systemctl stop mongos"
done

# 2. Arrêter les config servers
for server in config-server-1 config-server-2 config-server-3; do
  ssh $server "sudo systemctl stop mongod"
done

# 3. Restaurer le backup sur tous les config servers
for server in config-server-1 config-server-2 config-server-3; do
  ssh $server "
    sudo rm -rf /var/lib/mongodb-config/*
    sudo rsync -av /backup/config-server-latest/ /var/lib/mongodb-config/
    sudo chown -R mongodb:mongodb /var/lib/mongodb-config/
  "
done

# 4. Redémarrer les config servers
for server in config-server-1 config-server-2 config-server-3; do
  ssh $server "sudo systemctl start mongod"
done

# 5. Vérifier le replica set config
ssh config-server-1 "mongosh --port 27019 --eval 'rs.status()'"

# 6. Redémarrer les mongos
for mongos in mongos-1 mongos-2 mongos-3; do
  ssh $mongos "sudo systemctl start mongos"
done

# 7. Valider
mongosh --host mongos-1:27017 --eval "sh.status()"
```

---

## 6. Disaster Recovery

### Plans de Disaster Recovery

#### RPO et RTO

**RPO (Recovery Point Objective) :**
- Quantité maximale de données pouvant être perdues
- Exemples :
  - RPO = 0 : Aucune perte acceptable (réplication synchrone)
  - RPO = 5 min : Perte max de 5 minutes de données
  - RPO = 24h : Backup quotidien

**RTO (Recovery Time Objective) :**
- Temps maximal pour restaurer le service
- Exemples :
  - RTO = 5 min : Service rétabli en 5 minutes
  - RTO = 1h : Service rétabli en 1 heure
  - RTO = 4h : Service rétabli en 4 heures

#### Stratégies par Niveau de Criticité

```
┌──────────────┬──────────┬──────────┬────────────────────────┐
│ Criticité    │   RPO    │   RTO    │ Solution               │
├──────────────┼──────────┼──────────┼────────────────────────┤
│ CRITIQUE     │   0      │  < 1 min │ Multi-region RS        │
│              │          │          │ + Auto-failover        │
├──────────────┼──────────┼──────────┼────────────────────────┤
│ HAUTE        │  < 5 min │  < 5 min │ Cross-DC RS            │
│              │          │          │ + Automated recovery   │
├──────────────┼──────────┼──────────┼────────────────────────┤
│ MOYENNE      │  < 1h    │  < 30min │ Regular snapshots      │
│              │          │          │ + Standby replica      │
├──────────────┼──────────┼──────────┼────────────────────────┤
│ BASSE        │  < 24h   │  < 4h    │ Daily backups          │
│              │          │          │ + Manual recovery      │
└──────────────┴──────────┴──────────┴────────────────────────┘
```

### Procédure DR Complète - Site Primaire Perdu

```bash
#!/bin/bash
# Disaster Recovery - Bascule vers site secondaire

echo "=== DISASTER RECOVERY PROCEDURE ==="
echo "Primary site lost - Failing over to secondary site"

# ═══════════════════════════════════════
# Phase 1 : Évaluation
# ═══════════════════════════════════════
echo "Phase 1: Assessment"

# Vérifier que le site primaire est vraiment inaccessible
ping -c 5 primary-site.example.com
if [ $? -eq 0 ]; then
  echo "WARNING: Primary site is responding!"
  echo "Confirm before proceeding with failover"
  read -p "Continue? (yes/no): " confirm
  if [ "$confirm" != "yes" ]; then
    exit 1
  fi
fi

# ═══════════════════════════════════════
# Phase 2 : Promotion du Site Secondaire
# ═══════════════════════════════════════
echo "Phase 2: Promoting Secondary Site"

# Pour Replica Set géo-distribué
# Les secondaries du site secondaire vont élire un nouveau primary

# Si pas d'élection automatique (pas de majorité), forcer
ssh secondary-site-node1 "mongosh" <<'EOF'
// Vérifier l'état actuel
var status = rs.status()
printjson(status)

// Si pas de primary, forcer une reconfiguration
var cfg = rs.conf()

// Retirer les membres du site primaire (inaccessibles)
cfg.members = cfg.members.filter(m =>
  m.host.includes("secondary-site")
)

cfg.version++

// Forcer la reconfiguration
rs.reconfig(cfg, {force: true})

// Attendre l'élection
sleep(5000)

// Vérifier
rs.status()
EOF

# ═══════════════════════════════════════
# Phase 3 : Mise à Jour DNS/Load Balancer
# ═══════════════════════════════════════
echo "Phase 3: Updating DNS and Load Balancers"

# Pointer les applications vers le site secondaire
# (Méthode dépend de votre infrastructure)

# Exemple avec DNS
# dig mongodb.example.com
# Mettre à jour pour pointer vers secondary-site

# Exemple avec HAProxy
ssh load-balancer "
  # Désactiver backend primaire
  echo 'disable server mongodb/primary-site-1' | socat stdio /var/run/haproxy.sock

  # Activer backend secondaire
  echo 'enable server mongodb/secondary-site-1' | socat stdio /var/run/haproxy.sock
"

# ═══════════════════════════════════════
# Phase 4 : Validation
# ═══════════════════════════════════════
echo "Phase 4: Validation"

# Tester la connectivité depuis l'application
mongosh --host secondary-site-node1:27017 --eval "
  db.adminCommand({ping: 1})
  db.test_dr.insertOne({test: true, timestamp: new Date()})
  print('✓ Write successful on secondary site')
"

# ═══════════════════════════════════════
# Phase 5 : Notification
# ═══════════════════════════════════════
echo "Phase 5: Notifications"

# Notifier les équipes
curl -X POST https://alerts.example.com/webhook \
  -d '{
    "event": "DR_FAILOVER",
    "site": "secondary",
    "timestamp": "'$(date -Iseconds)'",
    "status": "ACTIVE"
  }'

echo ""
echo "DISASTER RECOVERY COMPLETE"
echo "Site secondaire est maintenant le site actif"
echo ""
echo "TODO:"
echo "1. Investiguer la cause de la panne du site primaire"
echo "2. Planifier la restauration du site primaire"
echo "3. Communiquer avec les stakeholders"
echo "4. Documenter l'incident"
```

### Restauration du Site Primaire

```bash
#!/bin/bash
# Restauration du site primaire après DR

echo "=== PRIMARY SITE RESTORATION ==="

# ═══════════════════════════════════════
# Phase 1 : Préparation
# ═══════════════════════════════════════
echo "Phase 1: Preparation"

# Vérifier que le site primaire est fonctionnel
ping -c 10 primary-site.example.com
if [ $? -ne 0 ]; then
  echo "ERROR: Primary site still unreachable"
  exit 1
fi

# ═══════════════════════════════════════
# Phase 2 : Resynchronisation
# ═══════════════════════════════════════
echo "Phase 2: Resyncing Primary Site"

# Sur les nœuds du site primaire
for node in primary-site-node1 primary-site-node2 primary-site-node3; do
  ssh $node "
    # Nettoyer les anciennes données
    sudo systemctl stop mongod
    sudo rm -rf /var/lib/mongodb/*

    # Redémarrer pour initial sync
    sudo systemctl start mongod
  "
done

# Les nœuds vont faire une initial sync depuis le site secondaire (actif)

# Surveiller la progression
ssh primary-site-node1 "tail -f /var/log/mongodb/mongod.log" | grep "initial sync"

# ═══════════════════════════════════════
# Phase 3 : Validation
# ═══════════════════════════════════════
echo "Phase 3: Validation"

# Vérifier que les nœuds sont SECONDARY et à jour
ssh primary-site-node1 "mongosh --eval 'rs.status()'" | grep "SECONDARY"

# Vérifier le replication lag
ssh primary-site-node1 "mongosh --eval 'rs.printSecondaryReplicationInfo()'"

# ═══════════════════════════════════════
# Phase 4 : Failback (Optionnel)
# ═══════════════════════════════════════
echo "Phase 4: Failback to Primary Site (Optional)"

read -p "Failback to primary site now? (yes/no): " failback

if [ "$failback" == "yes" ]; then
  # Augmenter la priorité des nœuds du site primaire
  ssh secondary-site-node1 "mongosh" <<'EOF'
  cfg = rs.conf()

  // Augmenter priorité site primaire
  cfg.members.forEach(m => {
    if (m.host.includes("primary-site")) {
      m.priority = 2
    } else {
      m.priority = 1
    }
  })

  cfg.version++
  rs.reconfig(cfg)

  // Le primary va basculer automatiquement
EOF

  echo "Failback initiated. Monitor for new PRIMARY election..."

  # Attendre l'élection
  sleep 30

  # Vérifier
  mongosh --host primary-site-node1:27017 --eval "rs.status()" | grep "PRIMARY"

  echo "✓ Failback complete. Primary site is now PRIMARY again."
fi

echo ""
echo "PRIMARY SITE RESTORATION COMPLETE"
```

---

## 7. Failover et Bascule

### Failover Automatique (Replica Set)

```javascript
// MongoDB gère automatiquement le failover

// Configuration pour optimiser le failover
cfg = rs.conf()

// Réduire les timeouts pour failover plus rapide
cfg.settings = {
  electionTimeoutMillis: 10000,    // 10s (défaut)
  heartbeatIntervalMillis: 2000,   // 2s
  heartbeatTimeoutSecs: 10         // 10s
}

// Priorités pour contrôler quel membre devient primary
cfg.members[0].priority = 2  // Préféré
cfg.members[1].priority = 1  // Normal
cfg.members[2].priority = 0.5  // Dernier recours

rs.reconfig(cfg)

// Test de failover
// Sur le primary actuel
db.adminCommand({replSetStepDown: 60})  // Step down pour 60 secondes

// Un nouveau primary sera élu automatiquement
```

### Failover Manuel (Maintenance Planifiée)

```javascript
// Procédure pour maintenance sans downtime

// 1. Identifier le primary
rs.status().members.find(m => m.stateStr === "PRIMARY").name

// 2. Step down le primary (graceful)
db.adminCommand({
  replSetStepDown: 120,  // Reste secondary pendant 120 secondes
  force: false
})

// 3. Un secondary est élu primary automatiquement

// 4. Effectuer la maintenance sur l'ancien primary
// (Redémarrage, upgrade, etc.)

// 5. Le nœud rejoint automatiquement comme secondary

// 6. Si besoin, le faire redevenir primary
// Augmenter sa priorité et attendre
```

### Bascule Applicative

```javascript
// Configuration application pour gérer le failover

// Node.js avec retry automatique
const { MongoClient } = require('mongodb')

const client = new MongoClient(uri, {
  // Retry writes automatiquement sur failover
  retryWrites: true,

  // Retry reads
  retryReads: true,

  // Write concern pour assurer durabilité
  writeConcern: {
    w: 'majority',
    wtimeout: 5000
  },

  // Read preference
  readPreference: 'primaryPreferred',  // Fallback sur secondary

  // Timeouts
  serverSelectionTimeoutMS: 5000,
  connectTimeoutMS: 10000,
  socketTimeoutMS: 45000
})

// Gestion des erreurs de failover
client.on('serverDescriptionChanged', (event) => {
  console.log('Server state changed:', event)
})

client.on('topologyDescriptionChanged', (event) => {
  console.log('Topology changed:', event)
  // Le driver reconnecte automatiquement
})

// Wrapper avec retry custom
async function executeWithRetry(operation, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await operation()
    } catch (error) {
      // Erreurs liées au failover
      if (error.message.includes('not master') ||
          error.message.includes('connection') ||
          error.code === 11600) {  // InterruptedDueToReplStateChange

        if (i < maxRetries - 1) {
          console.log(`Retry ${i + 1}/${maxRetries} after failover...`)
          await new Promise(resolve => setTimeout(resolve, 1000 * (i + 1)))
          continue
        }
      }
      throw error
    }
  }
}

// Utilisation
await executeWithRetry(async () => {
  return await db.collection('users').insertOne({name: "Alice"})
})
```

---

## 8. Tests et Validation

### Tests de Récupération Réguliers

```bash
#!/bin/bash
# Script de test de récupération (à exécuter régulièrement)

echo "=== MongoDB Recovery Drill ==="
echo "Date: $(date)"

TEST_ENV="test"  # Environnement de test

# ═══════════════════════════════════════
# Test 1 : Restauration depuis Backup
# ═══════════════════════════════════════
echo "Test 1: Backup Restoration"

# Prendre un backup
mongodump --host $TEST_ENV-mongodb:27017 --out /tmp/test-backup/

# "Corrompre" les données
mongosh --host $TEST_ENV-mongodb:27017 --eval "
  db.test_data.drop()
  print('Dropped test collection')
"

# Restaurer
mongorestore --host $TEST_ENV-mongodb:27017 --drop /tmp/test-backup/

# Valider
RESTORED_COUNT=$(mongosh --host $TEST_ENV-mongodb:27017 --quiet --eval "
  db.test_data.countDocuments({})
")

if [ $RESTORED_COUNT -gt 0 ]; then
  echo "✓ Backup restoration successful"
else
  echo "✗ Backup restoration failed"
fi

# ═══════════════════════════════════════
# Test 2 : Failover Replica Set
# ═══════════════════════════════════════
echo "Test 2: Replica Set Failover"

# Step down le primary
PRIMARY=$(mongosh --host $TEST_ENV-mongodb:27017 --quiet --eval "
  rs.status().members.find(m => m.stateStr === 'PRIMARY').name
" | tail -1)

echo "Current primary: $PRIMARY"

# Forcer step down
mongosh --host $TEST_ENV-mongodb:27017 --eval "
  db.adminCommand({replSetStepDown: 60, force: true})
"

# Attendre nouvelle élection
sleep 15

# Vérifier nouveau primary
NEW_PRIMARY=$(mongosh --host $TEST_ENV-mongodb:27017 --quiet --eval "
  rs.status().members.find(m => m.stateStr === 'PRIMARY').name
" | tail -1)

echo "New primary: $NEW_PRIMARY"

if [ "$PRIMARY" != "$NEW_PRIMARY" ]; then
  echo "✓ Failover successful"
else
  echo "✗ Failover failed"
fi

# ═══════════════════════════════════════
# Test 3 : Recovery Time (RTO)
# ═══════════════════════════════════════
echo "Test 3: Measuring RTO"

START_TIME=$(date +%s)

# Arrêter MongoDB
ssh $TEST_ENV-mongodb-1 "sudo systemctl stop mongod"

# Attendre que le service soit down
sleep 5

# Redémarrer
ssh $TEST_ENV-mongodb-1 "sudo systemctl start mongod"

# Attendre que le service soit up
while ! mongosh --host $TEST_ENV-mongodb-1:27017 --eval "db.adminCommand({ping: 1})" > /dev/null 2>&1; do
  sleep 1
done

END_TIME=$(date +%s)
RTO=$((END_TIME - START_TIME))

echo "Recovery Time: ${RTO} seconds"

if [ $RTO -lt 60 ]; then
  echo "✓ RTO within acceptable range (< 60s)"
else
  echo "⚠ RTO exceeds target"
fi

# ═══════════════════════════════════════
# Rapport Final
# ═══════════════════════════════════════
echo ""
echo "=== Recovery Drill Complete ==="
echo "Report saved to: /var/log/mongodb-recovery-drill-$(date +%Y%m%d).log"
```

### Checklist de Validation Post-Récupération

```javascript
// Checklist complète après récupération

function postRecoveryValidation() {
  var results = {
    timestamp: new Date(),
    checks: []
  }

  // 1. Connectivité de base
  try {
    db.adminCommand({ping: 1})
    results.checks.push({name: "Connectivity", status: "OK"})
  } catch (e) {
    results.checks.push({name: "Connectivity", status: "FAIL", error: e.message})
  }

  // 2. Statut du serveur
  try {
    var status = db.serverStatus()
    results.checks.push({
      name: "Server Status",
      status: "OK",
      uptime: status.uptime
    })
  } catch (e) {
    results.checks.push({name: "Server Status", status: "FAIL", error: e.message})
  }

  // 3. Replica Set (si applicable)
  try {
    var rsStatus = rs.status()
    var primary = rsStatus.members.find(m => m.stateStr === "PRIMARY")

    results.checks.push({
      name: "Replica Set",
      status: primary ? "OK" : "NO PRIMARY",
      members: rsStatus.members.length,
      primary: primary ? primary.name : "none"
    })
  } catch (e) {
    results.checks.push({name: "Replica Set", status: "NOT A REPLICA SET"})
  }

  // 4. Validation des collections
  var databases = db.adminCommand({listDatabases: 1}).databases
  databases.forEach(dbInfo => {
    if (!['admin', 'local', 'config'].includes(dbInfo.name)) {
      var database = db.getSiblingDB(dbInfo.name)

      database.getCollectionNames().forEach(collName => {
        try {
          var validation = database[collName].validate()
          results.checks.push({
            name: `Validation: ${dbInfo.name}.${collName}`,
            status: validation.valid ? "OK" : "INVALID",
            errors: validation.errors || []
          })
        } catch (e) {
          results.checks.push({
            name: `Validation: ${dbInfo.name}.${collName}`,
            status: "ERROR",
            error: e.message
          })
        }
      })
    }
  })

  // 5. Test CRUD
  try {
    var testDoc = {test: true, timestamp: new Date()}
    db.test_recovery.insertOne(testDoc)
    db.test_recovery.findOne({test: true})
    db.test_recovery.updateOne({test: true}, {$set: {updated: true}})
    db.test_recovery.deleteOne({test: true})
    db.test_recovery.drop()

    results.checks.push({name: "CRUD Operations", status: "OK"})
  } catch (e) {
    results.checks.push({name: "CRUD Operations", status: "FAIL", error: e.message})
  }

  // 6. Compter les échecs
  var failures = results.checks.filter(c => c.status === "FAIL" || c.status === "INVALID")
  results.summary = {
    total: results.checks.length,
    passed: results.checks.length - failures.length,
    failed: failures.length
  }

  return results
}

// Exécuter
var validationResults = postRecoveryValidation()
printjson(validationResults)

if (validationResults.summary.failed === 0) {
  print("\n✓ All validation checks passed")
} else {
  print("\n✗ " + validationResults.summary.failed + " validation checks failed")
}
```

---

## Checklist Globale de Récupération

```
INCIDENT: Panne MongoDB détectée

☐ PHASE 1: ÉVALUATION IMMÉDIATE (2 minutes)
  ☐ Identifier le type de panne
  ☐ Évaluer l'impact (standalone/RS/sharded)
  ☐ Vérifier les logs
  ☐ Déterminer la criticité

☐ PHASE 2: DIAGNOSTIC (5 minutes)
  ☐ Hardware OK ? (disque, mémoire, réseau)
  ☐ Logs d'erreur ?
  ☐ Arrêt propre ou brutal ?
  ☐ Données accessibles ?
  ☐ Backup disponible ?

☐ PHASE 3: DÉCISION (2 minutes)
  ☐ Redémarrage simple ?
  ☐ Récupération du journal ?
  ☐ Failover RS ?
  ☐ Restauration backup ?
  ☐ DR complet ?

☐ PHASE 4: EXÉCUTION (15-120 minutes selon type)
  ☐ Suivre la procédure appropriée
  ☐ Documenter chaque action
  ☐ Monitorer la progression
  ☐ Communiquer status

☐ PHASE 5: VALIDATION (15 minutes)
  ☐ Service disponible ?
  ☐ Données intègres ? (validate)
  ☐ Réplication OK ? (si RS)
  ☐ Performance normale ?
  ☐ Applications fonctionnelles ?

☐ PHASE 6: POST-INCIDENT
  ☐ Analyse de cause racine
  ☐ Rapport d'incident
  ☐ Mise à jour procédures
  ☐ Actions préventives
  ☐ Retour d'expérience équipe
```

---

## Conclusion

La récupération après panne nécessite :

1. **Préparation rigoureuse** : Procédures documentées, testées régulièrement
2. **Architecture résiliente** : Replica sets, backups, multi-site
3. **Monitoring proactif** : Détecter avant la panne critique
4. **Équipe formée** : Connaître les procédures, exercices réguliers
5. **Automatisation** : Scripts testés, playbooks

**Principes clés :**
- ✅ **Toujours utiliser Replica Set** (haute disponibilité)
- ✅ **Backups testés régulièrement** (RPO/RTO connus)
- ✅ **Procédures documentées et à jour**
- ✅ **Drills de récupération mensuels**
- ✅ **Architecture multi-site** (DR)
- ✅ **Monitoring avec alertes**
- ✅ **Automatisation maximale**

**Temps de récupération typiques :**
- Redémarrage simple : 10-30 secondes
- Failover automatique RS : 10-30 secondes
- Récupération journal : 1-5 minutes
- Resync secondary : 30 minutes - 2 heures
- Restauration backup : 1-4 heures
- DR complet : 2-8 heures

**L'objectif est de minimiser le MTTR (Mean Time To Recovery).**

---


⏭️ [Analyse des logs d'erreurs](/22-depannage-resolution-problemes/07-analyse-logs-erreurs.md)
