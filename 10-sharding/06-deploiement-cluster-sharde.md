🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.6 Déploiement d'un Cluster Shardé

## Introduction

Le déploiement d'un cluster shardé MongoDB est une opération complexe qui nécessite une planification minutieuse et une exécution rigoureuse. Contrairement à un déploiement de Replica Set, un cluster shardé implique plusieurs types de composants qui doivent être configurés et orchestrés correctement pour fonctionner ensemble.

Cette section fournit un guide complet pour déployer un cluster shardé en production, depuis la planification initiale jusqu'à la validation finale, en couvrant les stratégies, les anti-patterns et les bonnes pratiques éprouvées.

---

## Architecture Cible

### Topologie Standard de Production

Pour cet exemple, nous déployons un cluster shardé minimal mais robuste :

```
┌─────────────────────────────────────────────────────────┐
│                   APPLICATION TIER                      │
│                  (Load Balancer)                        │
└──────────────┬─────────────┬────────────────────────────┘
               │             │
     ┌─────────┴────┐  ┌─────┴──────┐   ┌─────────────┐
     │   mongos-1   │  │  mongos-2  │   │  mongos-3   │
     │  (Router)    │  │  (Router)  │   │  (Router)   │
     └──────┬───────┘  └──────┬─────┘   └──────┬──────┘
            │                 │                │
            └─────────────────┼────────────────┘
                              │
         ┌────────────────────┴────────────────────┐
         │                                         │
    ┌────▼─────┐         ┌──────────┐      ┌───────▼────┐
    │  Config  │         │  Config  │      │   Config   │
    │ Server 1 │◄───────►│ Server 2 │◄────►│  Server 3  │
    │ (Replica │         │ (Replica │      │  (Replica  │
    │   Set)   │         │   Set)   │      │    Set)    │
    └──────────┘         └──────────┘      └────────────┘
         │                     │                   │
         └─────────────────────┼───────────────────┘
                               │
         ┌─────────────────────┴──────────────────┐
         │                                        │
    ┌────▼────┐                            ┌──────▼───┐
    │ Shard A │                            │ Shard B  │
    │ Replica │                            │ Replica  │
    │   Set   │                            │   Set    │
    │         │                            │          │
    │ P S S   │                            │ P S S    │
    └─────────┘                            └──────────┘
```

**Composants** :
- **3 Config Servers** (Replica Set)
- **2 Shards** (chacun un Replica Set de 3 membres : 1 Primary + 2 Secondary)
- **3 Mongos** (Query Routers)

### Dimensionnement Matériel

#### Config Servers

| Ressource | Recommandation Production | Justification |
|-----------|---------------------------|---------------|
| CPU | 2-4 cores | Charge légère, métadonnées uniquement |
| RAM | 8-16 GB | Cache des métadonnées du cluster |
| Stockage | 50-100 GB SSD | Croissance lente des métadonnées |
| Réseau | 1 Gbps | Latence faible critique |

#### Shards (Replica Sets)

| Ressource | Recommandation Production | Justification |
|-----------|---------------------------|---------------|
| CPU | 8-16+ cores | Charge de travail principale |
| RAM | 32-128 GB | Proportionnel au working set |
| Stockage | 1-10+ TB SSD/NVMe | Selon volume de données |
| Réseau | 10 Gbps | Migrations de chunks intensives |

#### Mongos (Routers)

| Ressource | Recommandation Production | Justification |
|-----------|---------------------------|---------------|
| CPU | 4-8 cores | Routing et agrégation des résultats |
| RAM | 8-16 GB | Pas de stockage permanent |
| Stockage | 20 GB | Logs uniquement |
| Réseau | 10 Gbps | Point d'entrée des applications |

---

## Prérequis et Planification

### Checklist Pré-déploiement

#### 1. Infrastructure

- ✅ **Machines provisionnées** : Toutes les machines sont disponibles et accessibles
- ✅ **Réseau configuré** : Les machines peuvent communiquer entre elles
- ✅ **Pare-feu** : Ports ouverts (27017-27019 pour MongoDB)
- ✅ **DNS ou /etc/hosts** : Résolution de noms configurée
- ✅ **NTP synchronisé** : Horloges synchronisées sur tous les serveurs
- ✅ **Stockage** : Volumes montés avec les bonnes permissions

#### 2. Système d'Exploitation

- ✅ **OS supporté** : Linux (RHEL/CentOS, Ubuntu, Debian) recommandé
- ✅ **Ulimits configurés** : Limites de fichiers et processus augmentées
- ✅ **Transparent Huge Pages désactivé** : Impact négatif sur MongoDB
- ✅ **NUMA désactivé ou configuré** : Si applicable
- ✅ **Filesystem** : XFS ou ext4 recommandé

#### 3. MongoDB

- ✅ **Version compatible** : Même version sur tous les composants
- ✅ **Binaires installés** : mongod et mongos disponibles
- ✅ **Utilisateur système** : Compte dédié (mongodb:mongodb)
- ✅ **Répertoires créés** : /data/db, /var/log/mongodb

#### 4. Sécurité

- ✅ **Certificats TLS/SSL** : Si chiffrement activé
- ✅ **Keyfile** : Pour l'authentification inter-cluster
- ✅ **Plan d'authentification** : SCRAM-SHA-256 configuré
- ✅ **Comptes administrateurs** : Stratégie définie

### Plan de Déploiement

**Ordre recommandé** :
1. Config Servers (Replica Set)
2. Shards (Replica Sets)
3. Mongos (Routers)
4. Initialisation du cluster shardé
5. Configuration de la sécurité
6. Tests de validation
7. Mise en production

**Durée estimée** : 4-8 heures pour un premier déploiement
**Personnel** : 2-3 administrateurs recommandés

---

## Phase 1 : Déploiement des Config Servers

### Étape 1.1 : Configuration des fichiers mongod.conf

Sur chaque serveur config (3 serveurs : cfg1, cfg2, cfg3) :

```yaml
# /etc/mongod-configsvr.conf

# Stockage
storage:
  dbPath: /data/configdb
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 2

# Réplication
replication:
  replSetName: configReplSet

# Sharding
sharding:
  clusterRole: configsvr

# Réseau
net:
  port: 27019
  bindIp: 0.0.0.0

# Système
systemLog:
  destination: file
  path: /var/log/mongodb/configsvr.log
  logAppend: true

# Processus
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/configsvr.pid

# Sécurité (optionnel pour l'instant, activé après)
#security:
#  keyFile: /etc/mongodb-keyfile
#  authorization: enabled
```

### Étape 1.2 : Création des répertoires

Sur chaque config server :

```bash
# Créer les répertoires
sudo mkdir -p /data/configdb
sudo mkdir -p /var/log/mongodb
sudo mkdir -p /var/run/mongodb

# Permissions
sudo chown -R mongodb:mongodb /data/configdb
sudo chown -R mongodb:mongodb /var/log/mongodb
sudo chown -R mongodb:mongodb /var/run/mongodb
```

### Étape 1.3 : Démarrage des instances

Sur **cfg1, cfg2, cfg3** :

```bash
# Démarrer mongod en mode config server
sudo mongod --config /etc/mongod-configsvr.conf

# Vérifier le démarrage
sudo tail -f /var/log/mongodb/configsvr.log

# Vérifier le processus
ps aux | grep mongod
```

### Étape 1.4 : Initialisation du Replica Set

Se connecter à **cfg1** :

```javascript
// Connexion au premier config server
mongosh --port 27019

// Initialiser le replica set
rs.initiate({
  _id: "configReplSet",
  configsvr: true,
  members: [
    { _id: 0, host: "cfg1.example.com:27019" },
    { _id: 1, host: "cfg2.example.com:27019" },
    { _id: 2, host: "cfg3.example.com:27019" }
  ]
})

// Attendre l'élection du Primary
// Vérifier le statut
rs.status()

// Résultat attendu : 1 PRIMARY, 2 SECONDARY
```

### Étape 1.5 : Validation

```javascript
// Vérifier la configuration
rs.conf()

// Vérifier que tous les membres sont sains
rs.status().members.forEach(function(member) {
  print(member.name + " : " + member.stateStr);
})

// Test d'écriture (sur le PRIMARY)
use config
db.test.insertOne({ test: "config servers ready" })
db.test.find()

// Nettoyage
db.test.drop()
```

---

## Phase 2 : Déploiement des Shards

### Étape 2.1 : Configuration pour le Shard A

Sur chaque membre du Shard A (shardA1, shardA2, shardA3) :

```yaml
# /etc/mongod-shardA.conf

# Stockage
storage:
  dbPath: /data/shardA
  journal:
    enabled: true
  engine: wiredTiger
  wiredTiger:
    engineConfig:
      cacheSizeGB: 16  # Ajuster selon la RAM disponible

# Réplication
replication:
  replSetName: shardA

# Sharding
sharding:
  clusterRole: shardsvr

# Réseau
net:
  port: 27018
  bindIp: 0.0.0.0

# Système
systemLog:
  destination: file
  path: /var/log/mongodb/shardA.log
  logAppend: true

# Processus
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/shardA.pid
```

### Étape 2.2 : Démarrage et initialisation du Shard A

```bash
# Sur shardA1, shardA2, shardA3
sudo mkdir -p /data/shardA
sudo chown mongodb:mongodb /data/shardA
sudo mongod --config /etc/mongod-shardA.conf
```

Connexion à **shardA1** :

```javascript
mongosh --port 27018

// Initialiser le replica set
rs.initiate({
  _id: "shardA",
  members: [
    { _id: 0, host: "shardA1.example.com:27018" },
    { _id: 1, host: "shardA2.example.com:27018" },
    { _id: 2, host: "shardA3.example.com:27018" }
  ]
})

// Vérifier
rs.status()
```

### Étape 2.3 : Configuration pour le Shard B

Répéter le processus pour le Shard B (shardB1, shardB2, shardB3) :

```yaml
# /etc/mongod-shardB.conf
# Identique au Shard A mais avec :
# - replSetName: shardB
# - dbPath: /data/shardB
# - logPath: /var/log/mongodb/shardB.log
```

Initialisation du Shard B :

```javascript
mongosh --host shardB1.example.com --port 27018

rs.initiate({
  _id: "shardB",
  members: [
    { _id: 0, host: "shardB1.example.com:27018" },
    { _id: 1, host: "shardB2.example.com:27018" },
    { _id: 2, host: "shardB3.example.com:27018" }
  ]
})
```

### Étape 2.4 : Validation des Shards

Pour chaque shard :

```javascript
// Vérifier le statut
rs.status()

// Test d'écriture
use testdb
db.test.insertOne({ shard: "A", test: "ready" })
db.test.find()

// Vérifier l'oplog
use local
db.oplog.rs.find().limit(5).sort({ $natural: -1 })
```

---

## Phase 3 : Déploiement des Mongos

### Étape 3.1 : Configuration des Mongos

Sur chaque serveur mongos (mongos1, mongos2, mongos3) :

```yaml
# /etc/mongos.conf

# Sharding
sharding:
  configDB: configReplSet/cfg1.example.com:27019,cfg2.example.com:27019,cfg3.example.com:27019

# Réseau
net:
  port: 27017
  bindIp: 0.0.0.0

# Système
systemLog:
  destination: file
  path: /var/log/mongodb/mongos.log
  logAppend: true

# Processus
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongos.pid
```

### Étape 3.2 : Démarrage des Mongos

Sur **mongos1, mongos2, mongos3** :

```bash
# Créer les répertoires de logs
sudo mkdir -p /var/log/mongodb
sudo mkdir -p /var/run/mongodb
sudo chown -R mongodb:mongodb /var/log/mongodb /var/run/mongodb

# Démarrer mongos
sudo mongos --config /etc/mongos.conf

# Vérifier
sudo tail -f /var/log/mongodb/mongos.log
ps aux | grep mongos
```

### Étape 3.3 : Validation

```bash
# Se connecter à un mongos
mongosh --host mongos1.example.com --port 27017

# Vérifier la connexion aux config servers
sh.status()  # Devrait montrer le cluster initialisé mais sans shards
```

---

## Phase 4 : Initialisation du Cluster Shardé

### Étape 4.1 : Ajout des Shards au Cluster

Se connecter à **n'importe quel mongos** :

```javascript
mongosh --host mongos1.example.com --port 27017

// Ajouter le Shard A
sh.addShard("shardA/shardA1.example.com:27018,shardA2.example.com:27018,shardA3.example.com:27018")

// Résultat attendu :
// {
//   "shardAdded" : "shardA",
//   "ok" : 1
// }

// Ajouter le Shard B
sh.addShard("shardB/shardB1.example.com:27018,shardB2.example.com:27018,shardB3.example.com:27018")

// Résultat attendu :
// {
//   "shardAdded" : "shardB",
//   "ok" : 1
// }
```

### Étape 4.2 : Vérification du Cluster

```javascript
// Vérifier que les shards sont ajoutés
sh.status()

// Output attendu :
// shards:
//   { "_id" : "shardA", "host" : "shardA/shardA1.example.com:27018,..." }
//   { "_id" : "shardB", "host" : "shardB/shardB1.example.com:27018,..." }

// Lister les shards
db.adminCommand({ listShards: 1 })

// Vérifier les config servers
db.getSiblingDB("config").shards.find().pretty()
```

---

## Phase 5 : Configuration de la Sécurité

### Étape 5.1 : Génération du Keyfile

Sur **un serveur quelconque**, générer le keyfile :

```bash
# Générer une clé aléatoire sécurisée
openssl rand -base64 756 > /tmp/mongodb-keyfile

# Définir les permissions strictes
chmod 400 /tmp/mongodb-keyfile
```

### Étape 5.2 : Distribution du Keyfile

Copier le keyfile sur **tous les serveurs** (config servers, shards, mongos) :

```bash
# Exemple avec scp
scp /tmp/mongodb-keyfile mongodb@cfg1.example.com:/etc/mongodb-keyfile
scp /tmp/mongodb-keyfile mongodb@cfg2.example.com:/etc/mongodb-keyfile
# ... répéter pour tous les serveurs

# Sur chaque serveur
sudo chown mongodb:mongodb /etc/mongodb-keyfile
sudo chmod 400 /etc/mongodb-keyfile
```

### Étape 5.3 : Création de l'Administrateur

**Avant d'activer l'authentification**, créer un utilisateur admin :

```javascript
// Se connecter à un mongos
mongosh --host mongos1.example.com --port 27017

// Créer l'administrateur du cluster
use admin
db.createUser({
  user: "clusterAdmin",
  pwd: "SecurePassword123!",  // Utiliser un mot de passe fort
  roles: [
    { role: "clusterAdmin", db: "admin" },
    { role: "userAdminAnyDatabase", db: "admin" },
    { role: "dbAdminAnyDatabase", db: "admin" },
    { role: "readWriteAnyDatabase", db: "admin" }
  ]
})

// Vérifier
db.getUsers()
```

### Étape 5.4 : Activation de l'Authentification

**Sur tous les composants**, mettre à jour les fichiers de configuration :

#### Config Servers

```yaml
# Ajouter dans /etc/mongod-configsvr.conf
security:
  keyFile: /etc/mongodb-keyfile
  authorization: enabled
```

#### Shards

```yaml
# Ajouter dans /etc/mongod-shardA.conf et /etc/mongod-shardB.conf
security:
  keyFile: /etc/mongodb-keyfile
  authorization: enabled
```

#### Mongos

```yaml
# Ajouter dans /etc/mongos.conf
security:
  keyFile: /etc/mongodb-keyfile
```

### Étape 5.5 : Redémarrage avec Authentification

**IMPORTANT** : Redémarrer dans l'ordre pour éviter les interruptions.

```bash
# 1. Redémarrer les config servers (un par un)
sudo systemctl restart mongod-configsvr

# 2. Redémarrer les shards (un replica set à la fois)
# Secondary d'abord, Primary en dernier
sudo systemctl restart mongod-shardA
sudo systemctl restart mongod-shardB

# 3. Redémarrer les mongos
sudo systemctl restart mongos
```

### Étape 5.6 : Test d'Authentification

```javascript
// Connexion avec authentification
mongosh --host mongos1.example.com --port 27017 \
  -u clusterAdmin \
  -p SecurePassword123! \
  --authenticationDatabase admin

// Vérifier l'accès
sh.status()
```

---

## Phase 6 : Tests de Validation

### Test 1 : Sharding d'une Collection de Test

```javascript
// Connexion authentifiée
mongosh --host mongos1.example.com -u clusterAdmin -p SecurePassword123! --authenticationDatabase admin

// Créer une base et activer le sharding
use testdb
sh.enableSharding("testdb")

// Créer une collection et la sharder
db.createCollection("users")
sh.shardCollection("testdb.users", { user_id: "hashed" })

// Insérer des données de test
for (let i = 0; i < 10000; i++) {
  db.users.insertOne({
    user_id: i,
    name: "User" + i,
    email: "user" + i + "@example.com",
    created_at: new Date()
  });
}

// Vérifier la distribution
db.users.getShardDistribution()
```

### Test 2 : Vérification du Balancer

```javascript
// Vérifier que le balancer fonctionne
sh.getBalancerState()  // Doit retourner true

// Vérifier les chunks
db.getSiblingDB("config").chunks.find({ ns: "testdb.users" }).count()

// Vérifier la distribution par shard
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "testdb.users" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])
```

### Test 3 : Haute Disponibilité

```javascript
// Test de failover sur un shard
// 1. Se connecter directement au Primary du shardA
mongosh --host shardA1.example.com:27018 -u clusterAdmin -p SecurePassword123! --authenticationDatabase admin

// 2. Forcer une élection (step down du Primary)
rs.stepDown(60)

// 3. Vérifier depuis mongos que les requêtes continuent
mongosh --host mongos1.example.com -u clusterAdmin -p SecurePassword123! --authenticationDatabase admin
use testdb
db.users.find().limit(5)  // Doit fonctionner sans erreur

// 4. Vérifier le nouveau Primary
use admin
sh.status()
```

### Test 4 : Vérification des Performances

```javascript
// Test de lecture
use testdb
db.users.find({ user_id: 5000 }).explain("executionStats")

// Test d'écriture
db.users.insertOne({ user_id: 100000, name: "Performance Test" })

// Test d'agrégation
db.users.aggregate([
  { $group: { _id: null, total: { $sum: 1 } } }
])
```

---

## Stratégies de Déploiement Avancées

### Stratégie 1 : Déploiement Progressif (Rolling Deployment)

Pour minimiser les interruptions en production :

```
1. Déployer Config Servers
   ├─ cfg1 → cfg2 → cfg3
   └─ Valider chaque nœud avant le suivant

2. Déployer Shards (un à un)
   ├─ shardA : secondary1 → secondary2 → primary
   ├─ Valider shardA
   └─ shardB : secondary1 → secondary2 → primary

3. Déployer Mongos (un à un)
   ├─ mongos1 → tester
   ├─ mongos2 → tester
   └─ mongos3 → tester

4. Basculer le trafic progressivement
   ├─ 10% → mongos1
   ├─ 50% → mongos1 + mongos2
   └─ 100% → tous les mongos
```

### Stratégie 2 : Déploiement avec Environnement de Staging

```
1. Déployer cluster identique en staging
2. Tester toutes les opérations
3. Valider les performances
4. Documenter les procédures
5. Répliquer en production avec ajustements
```

### Stratégie 3 : Déploiement Multi-Datacenter

Pour la haute disponibilité géographique :

```yaml
# Exemple : 3 datacenters (DC1, DC2, DC3)

Config Servers:
  - cfg1 : DC1
  - cfg2 : DC2
  - cfg3 : DC3

Shard A:
  - shardA-primary : DC1
  - shardA-secondary1 : DC2
  - shardA-secondary2 : DC3

Shard B:
  - shardB-primary : DC2
  - shardB-secondary1 : DC1
  - shardB-secondary2 : DC3

Mongos:
  - mongos déployés dans chaque DC près des applications
```

Configuration avec priorités pour l'élection :

```javascript
// ShardA : Primary préféré dans DC1
cfg = rs.conf()
cfg.members[0].priority = 2  // DC1
cfg.members[1].priority = 1  // DC2
cfg.members[2].priority = 1  // DC3
rs.reconfig(cfg)

// ShardB : Primary préféré dans DC2
cfg = rs.conf()
cfg.members[0].priority = 1  // DC1
cfg.members[1].priority = 2  // DC2
cfg.members[2].priority = 1  // DC3
rs.reconfig(cfg)
```

---

## Anti-Patterns et Erreurs Courantes

### ❌ Anti-Pattern 1 : Config Servers non Redondants

**Problème** :
```bash
# Déployer seulement 1 ou 2 config servers
mongod --configsvr --replSet configReplSet --port 27019
```

**Conséquence** :
- Perte du seul config server → **cluster entier inaccessible**
- 2 config servers → pas de majorité pour élection
- **PERTE DE DONNÉES possible**

**Solution** :
```bash
# TOUJOURS déployer 3 config servers minimum
# 5 config servers pour criticité extrême
```

### ❌ Anti-Pattern 2 : Mongos sur les Shards

**Problème** :
```bash
# Installer mongos sur le même serveur que les shards
# serveur1: mongod (shard) + mongos
```

**Conséquence** :
- Contention des ressources (CPU, RAM, réseau)
- Couplage fort → panne d'un shard impacte le routing
- Difficulté de scaling indépendant

**Solution** :
```bash
# Mongos sur serveurs dédiés ou avec les applications
# Architecture découplée
```

### ❌ Anti-Pattern 3 : Shards en Standalone

**Problème** :
```bash
# Shards sans réplication
mongod --shardsvr --port 27018  # Standalone !
```

**Conséquence** :
- **Aucune tolérance aux pannes**
- Panne d'un shard → perte de données
- Maintenance impossible sans downtime

**Solution** :
```bash
# TOUJOURS des Replica Sets pour les shards
# Minimum 3 membres : 1 Primary + 2 Secondary
```

### ❌ Anti-Pattern 4 : Keyfile Faible ou Réutilisé

**Problème** :
```bash
# Keyfile trop simple
echo "password123" > /etc/mongodb-keyfile

# Ou réutiliser un keyfile entre environnements
cp /prod/mongodb-keyfile /staging/mongodb-keyfile
```

**Conséquence** :
- Sécurité compromise
- Accès non autorisé possible
- Conformité violée

**Solution** :
```bash
# Keyfile aléatoire et unique par environnement
openssl rand -base64 756 > /etc/mongodb-keyfile
chmod 400 /etc/mongodb-keyfile

# Différents keyfiles : production, staging, dev
```

### ❌ Anti-Pattern 5 : Pas de Test de Failover

**Problème** :
```bash
# Déployer et passer en production sans tester
# Ne jamais simuler de panne
```

**Conséquence** :
- Comportement inconnu en cas de vraie panne
- Panique et mauvaises décisions en incident
- RTO/RPO non respectés

**Solution** :
```bash
# Tests de failover obligatoires
# 1. Arrêter un config server
# 2. Arrêter le Primary d'un shard
# 3. Redémarrer un mongos
# 4. Simuler une panne réseau
# 5. Documenter les comportements observés
```

### ❌ Anti-Pattern 6 : Sharding Immédiat d'une Collection

**Problème** :
```javascript
// Sharder sans réflexion
sh.enableSharding("mydb")
sh.shardCollection("mydb.users", { _id: 1 })

// Puis insérer des millions de documents
```

**Conséquence** :
- Tous les documents sur un seul chunk/shard
- Distribution déséquilibrée durable
- Migrations massives ultérieures

**Solution** :
```javascript
// Pré-splitter avant insertion massive
sh.shardCollection("mydb.users", { user_id: "hashed" })

// Pré-splitter
for (var i = 0; i < 10; i++) {
  sh.splitAt("mydb.users", { user_id: i * 1000 });
}

// Puis insérer les données
```

### ❌ Anti-Pattern 7 : Ignorer les Métriques Réseau

**Problème** :
```bash
# Déployer dans différents datacenters
# Sans mesurer la latence réseau
```

**Conséquence** :
- Latence inter-datacenter élevée (>50ms)
- Réplication lente
- Élections fréquentes en cas de fluctuation réseau
- Write concern timeout

**Solution** :
```bash
# Mesurer la latence avant déploiement
ping -c 100 cfg2.example.com

# Latence acceptable : < 5ms (même DC), < 50ms (différents DC)
# Ajuster les timeouts en conséquence

# Configuration pour haute latence
replication:
  electionTimeoutMillis: 30000  # 30 secondes au lieu de 10
```

---

## Automatisation du Déploiement

### Script Bash de Déploiement Complet

```bash
#!/bin/bash
# deploy-sharded-cluster.sh

set -e  # Arrêt si erreur

# Variables
CONFIG_SERVERS=("cfg1.example.com" "cfg2.example.com" "cfg3.example.com")
SHARD_A_SERVERS=("shardA1.example.com" "shardA2.example.com" "shardA3.example.com")
SHARD_B_SERVERS=("shardB1.example.com" "shardB2.example.com" "shardB3.example.com")
MONGOS_SERVERS=("mongos1.example.com" "mongos2.example.com" "mongos3.example.com")

echo "=== Déploiement Cluster Shardé MongoDB ==="
echo ""

# Phase 1 : Config Servers
echo "Phase 1 : Déploiement Config Servers..."
for server in "${CONFIG_SERVERS[@]}"; do
  echo "  - Démarrage config server sur $server"
  ssh mongodb@$server "sudo mongod --config /etc/mongod-configsvr.conf"
  sleep 5
done

echo "  - Initialisation Replica Set Config Servers"
mongosh --host ${CONFIG_SERVERS[0]} --port 27019 --eval "
  rs.initiate({
    _id: 'configReplSet',
    configsvr: true,
    members: [
      { _id: 0, host: '${CONFIG_SERVERS[0]}:27019' },
      { _id: 1, host: '${CONFIG_SERVERS[1]}:27019' },
      { _id: 2, host: '${CONFIG_SERVERS[2]}:27019' }
    ]
  })
"
sleep 30  # Attendre élection

# Phase 2 : Shards
echo ""
echo "Phase 2 : Déploiement Shards..."

# Shard A
for server in "${SHARD_A_SERVERS[@]}"; do
  echo "  - Démarrage shard A sur $server"
  ssh mongodb@$server "sudo mongod --config /etc/mongod-shardA.conf"
  sleep 5
done

mongosh --host ${SHARD_A_SERVERS[0]} --port 27018 --eval "
  rs.initiate({
    _id: 'shardA',
    members: [
      { _id: 0, host: '${SHARD_A_SERVERS[0]}:27018' },
      { _id: 1, host: '${SHARD_A_SERVERS[1]}:27018' },
      { _id: 2, host: '${SHARD_A_SERVERS[2]}:27018' }
    ]
  })
"
sleep 30

# Shard B (similaire)
# ...

# Phase 3 : Mongos
echo ""
echo "Phase 3 : Déploiement Mongos..."
for server in "${MONGOS_SERVERS[@]}"; do
  echo "  - Démarrage mongos sur $server"
  ssh mongodb@$server "sudo mongos --config /etc/mongos.conf"
  sleep 5
done

# Phase 4 : Ajout des shards
echo ""
echo "Phase 4 : Ajout des Shards au cluster..."
mongosh --host ${MONGOS_SERVERS[0]} --port 27017 --eval "
  sh.addShard('shardA/${SHARD_A_SERVERS[0]}:27018,${SHARD_A_SERVERS[1]}:27018,${SHARD_A_SERVERS[2]}:27018')
  sh.addShard('shardB/${SHARD_B_SERVERS[0]}:27018,${SHARD_B_SERVERS[1]}:27018,${SHARD_B_SERVERS[2]}:27018')
"

# Validation
echo ""
echo "=== Validation du déploiement ==="
mongosh --host ${MONGOS_SERVERS[0]} --port 27017 --eval "sh.status()"

echo ""
echo "✅ Déploiement terminé avec succès !"
```

### Ansible Playbook

```yaml
# deploy-sharded-cluster.yml
---
- name: Deploy MongoDB Sharded Cluster
  hosts: all
  become: yes
  vars:
    mongodb_version: "7.0"
    config_servers:
      - cfg1.example.com
      - cfg2.example.com
      - cfg3.example.com

  tasks:
    - name: Install MongoDB
      include_role:
        name: mongodb-install

    - name: Deploy Config Servers
      include_role:
        name: mongodb-config-servers
      when: inventory_hostname in config_servers

    - name: Deploy Shards
      include_role:
        name: mongodb-shards
      when: "'shard' in group_names"

    - name: Deploy Mongos
      include_role:
        name: mongodb-mongos
      when: "'mongos' in group_names"

    - name: Initialize Sharded Cluster
      include_role:
        name: mongodb-init-cluster
      run_once: true
      delegate_to: "{{ groups['mongos'][0] }}"
```

---

## Monitoring Post-Déploiement

### Métriques Critiques à Surveiller

```javascript
// 1. État du cluster
sh.status()

// 2. Santé des replica sets
db.adminCommand({ replSetGetStatus: 1 })

// 3. Distribution des chunks
db.getSiblingDB("config").chunks.aggregate([
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])

// 4. État du balancer
sh.getBalancerState()
sh.isBalancerRunning()

// 5. Opérations en cours
db.currentOp({ active: true })

// 6. Logs récents de migration
db.getSiblingDB("config").changelog.find({
  time: { $gte: new Date(Date.now() - 3600000) }
}).sort({ time: -1 })
```

### Dashboard de Monitoring Recommandé

**Prometheus + Grafana** :

```yaml
# prometheus-mongodb-exporter.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'mongodb-sharded'
    static_configs:
      - targets:
        - 'mongos1.example.com:9216'
        - 'mongos2.example.com:9216'
        - 'mongos3.example.com:9216'
        labels:
          cluster: 'production'
          component: 'mongos'

  - job_name: 'mongodb-config'
    static_configs:
      - targets:
        - 'cfg1.example.com:9216'
        - 'cfg2.example.com:9216'
        - 'cfg3.example.com:9216'
        labels:
          cluster: 'production'
          component: 'config-server'

  - job_name: 'mongodb-shards'
    static_configs:
      - targets:
        - 'shardA1.example.com:9216'
        - 'shardB1.example.com:9216'
        labels:
          cluster: 'production'
          component: 'shard'
```

---

## Checklist de Validation Finale

### ✅ Infrastructure

- [ ] Tous les serveurs sont accessibles
- [ ] Résolution DNS fonctionne
- [ ] Ports réseau ouverts et testés
- [ ] NTP synchronisé sur tous les serveurs
- [ ] Stockage correctement monté avec les permissions

### ✅ Config Servers

- [ ] 3 config servers démarrés
- [ ] Replica Set initialisé (1 PRIMARY, 2 SECONDARY)
- [ ] Pas d'erreurs dans les logs
- [ ] Test d'écriture/lecture réussi

### ✅ Shards

- [ ] Tous les shards démarrés (Replica Sets)
- [ ] Chaque shard : 1 PRIMARY, 2 SECONDARY
- [ ] Pas d'erreurs dans les logs
- [ ] Oplogs sains et répliquant

### ✅ Mongos

- [ ] Tous les mongos démarrés
- [ ] Connexion aux config servers réussie
- [ ] sh.status() affiche tous les shards
- [ ] Requêtes fonctionnent via mongos

### ✅ Sécurité

- [ ] Keyfile déployé sur tous les composants
- [ ] Authentification activée
- [ ] Utilisateur admin créé
- [ ] Connexion authentifiée fonctionne
- [ ] TLS/SSL activé (si applicable)

### ✅ Fonctionnalités

- [ ] Sharding d'une collection test réussi
- [ ] Insertion de données test réussie
- [ ] Distribution des chunks visible
- [ ] Balancer actif
- [ ] Test de failover réussi

### ✅ Monitoring

- [ ] Logs centralisés configurés
- [ ] Métriques collectées (Prometheus/Grafana)
- [ ] Alertes configurées
- [ ] Dashboards opérationnels

### ✅ Documentation

- [ ] Architecture documentée
- [ ] Procédures de maintenance écrites
- [ ] Runbooks d'incidents créés
- [ ] Contacts d'escalade définis

---

## Troubleshooting : Problèmes Courants

### Problème 1 : Config Server ne démarre pas

**Symptômes** :
```bash
# Logs
[error] Failed to start config server: Address already in use
```

**Solutions** :
```bash
# Vérifier qu'aucun processus n'utilise le port
sudo lsof -i :27019
sudo netstat -tuln | grep 27019

# Tuer le processus si nécessaire
sudo kill -9 <PID>

# Ou changer le port dans la configuration
```

### Problème 2 : Impossible d'ajouter un Shard

**Symptômes** :
```javascript
sh.addShard("shardA/...")
// Error: Connection refused
```

**Solutions** :
```bash
# 1. Vérifier que le shard est accessible
mongosh --host shardA1.example.com:27018

# 2. Vérifier le replica set
rs.status()

# 3. S'assurer qu'il y a un PRIMARY
# 4. Vérifier les règles firewall
sudo iptables -L -n | grep 27018

# 5. Vérifier la configuration réseau
ping shardA1.example.com
```

### Problème 3 : Élection du Primary échoue

**Symptômes** :
```javascript
rs.status()
// Tous les membres en SECONDARY ou STARTUP
```

**Solutions** :
```javascript
// 1. Forcer une reconfiguration
cfg = rs.conf()
cfg.members[0].priority = 2
rs.reconfig(cfg, { force: true })

// 2. Vérifier les priorités
rs.conf().members.forEach(m => print(m.host + ": " + m.priority))

// 3. Vérifier la connectivité réseau entre membres
```

---

## Conclusion

Le déploiement d'un cluster shardé MongoDB est une opération complexe mais maîtrisable avec :

- ✅ **Une planification rigoureuse** : Dimensionnement, architecture, checklist
- ✅ **Un déploiement méthodique** : Phase par phase, validation continue
- ✅ **La sécurité dès le départ** : Keyfile, authentification, TLS
- ✅ **Des tests exhaustifs** : Failover, performance, distribution
- ✅ **Un monitoring proactif** : Métriques, logs, alertes

En suivant ce guide et en évitant les anti-patterns documentés, vous disposez d'une base solide pour déployer et maintenir un cluster shardé MongoDB performant et résilient en production.

---

## Ressources

- [MongoDB Documentation - Deploy a Sharded Cluster](https://docs.mongodb.com/manual/tutorial/deploy-shard-cluster/)
- [MongoDB Production Notes](https://docs.mongodb.com/manual/administration/production-notes/)
- [MongoDB Security Checklist](https://docs.mongodb.com/manual/administration/security-checklist/)
- [MongoDB Ops Manager](https://www.mongodb.com/products/ops-manager)

---


⏭️ [Activer le sharding sur une base et une collection](/10-sharding/07-activer-sharding.md)
