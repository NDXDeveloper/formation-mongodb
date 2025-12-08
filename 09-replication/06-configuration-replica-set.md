🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.6 Configuration d'un Replica Set

## Introduction

La configuration d'un Replica Set est une étape critique qui définit l'architecture de haute disponibilité de MongoDB. Une configuration appropriée garantit la résilience, les performances et la cohérence des données. Ce chapitre couvre l'ensemble du processus, des configurations de base aux architectures complexes multi-datacenter.

## Architecture de Référence

### Topologie Standard à 3 Nœuds

```
┌─────────────────┐
│    PRIMARY      │  ← Écritures
│  mongodb-01     │
│  Priority: 2    │
└────────┬────────┘
         │
    ┌────┴──────┐
    │           │
┌───▼──────┐ ┌──▼───────┐
│SECONDARY │ │SECONDARY │
│mongodb-02│ │mongodb-03│
│Priority:1│ │Priority:1│
└──────────┘ └──────────┘
```

### Topologie Multi-Datacenter (5 Nœuds)

```
Datacenter 1 (Principal)          Datacenter 2 (DR)
┌─────────────────┐              ┌─────────────────┐
│    PRIMARY      │              │   SECONDARY     │
│  mongodb-dc1-01 │              │  mongodb-dc2-01 │
│  Priority: 10   │              │  Priority: 1    │
└────────┬────────┘              └─────────────────┘
         │
    ┌────┴──────┐                   ┌─────────────────┐
    │           │                   │   SECONDARY     │
┌───▼──────┐ ┌──▼───────┐           │  mongodb-dc2-02 │
│SECONDARY │ │SECONDARY │           │  Priority: 1    │
│dc1-02    │ │dc1-03    │           │  votes: 0       │
│Priority:9│ │Priority:9│           └─────────────────┘
└──────────┘ └──────────┘
```

## Préparation de l'Infrastructure

### Prérequis Matériels

**Minimum pour Production** :

| Composant | Spécification Minimale | Recommandé |
|-----------|----------------------|------------|
| CPU | 4 cores | 8+ cores |
| RAM | 8 GB | 16-32 GB |
| Disque | 100 GB SSD | 500 GB+ NVMe SSD |
| Réseau | 1 Gbps | 10 Gbps |
| Latence réseau | < 50ms entre membres | < 10ms |

### Prérequis Système

```bash
# 1. Désactiver Transparent Huge Pages (THP)
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# Rendre permanent (/etc/rc.local)
cat >> /etc/rc.local <<EOF
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
EOF

# 2. Limites de fichiers (ulimit)
cat >> /etc/security/limits.conf <<EOF
mongodb soft nofile 64000
mongodb hard nofile 64000
mongodb soft nproc 64000
mongodb hard nproc 64000
EOF

# 3. Paramètres kernel
cat >> /etc/sysctl.conf <<EOF
# MongoDB recommandations
vm.swappiness = 1
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5
net.core.somaxconn = 4096
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 8
EOF

sysctl -p

# 4. Désactiver NUMA (si applicable)
numactl --interleave=all mongod --config /etc/mongod.conf

# 5. Synchronisation NTP
systemctl enable chronyd
systemctl start chronyd
```

### Résolution DNS et Hosts

```bash
# /etc/hosts sur chaque serveur
cat >> /etc/hosts <<EOF
# Replica Set members
10.0.1.10   mongodb-01  mongodb-01.internal.company.com
10.0.1.11   mongodb-02  mongodb-02.internal.company.com
10.0.1.12   mongodb-03  mongodb-03.internal.company.com
EOF

# Vérifier la résolution
ping -c 3 mongodb-01
ping -c 3 mongodb-02
ping -c 3 mongodb-03
```

## Configuration de Base

### Fichier mongod.conf (Format YAML)

#### Configuration Membre 1 (Primary)

```yaml
# /etc/mongod.conf - mongodb-01

# Réseau
net:
  port: 27017
  bindIp: 0.0.0.0  # Production: spécifier IPs spécifiques
  maxIncomingConnections: 65536

# Stockage
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
    commitIntervalMs: 100
  directoryPerDB: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # ~50% RAM disponible
      journalCompressor: snappy
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

# Processus
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen
  timeStampFormat: iso8601-utc
  component:
    replication:
      verbosity: 1

processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
  timeZoneInfo: /usr/share/zoneinfo

# Réplication
replication:
  replSetName: rs0
  oplogSizeMB: 10240  # 10 GB
  enableMajorityReadConcern: true

# Sécurité (à activer après configuration initiale)
# security:
#   authorization: enabled
#   keyFile: /etc/mongodb/keyfile
#   clusterAuthMode: keyFile

# Monitoring
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 1.0
```

#### Configuration Membres 2 et 3 (Secondary)

```yaml
# /etc/mongod.conf - mongodb-02 et mongodb-03
# Identique à mongodb-01, sauf :

net:
  port: 27017
  bindIp: 0.0.0.0

# Adapter dbPath et logs si sur même machine (dev)
storage:
  dbPath: /data/mongodb  # ou /data/mongodb-02, /data/mongodb-03

systemLog:
  path: /var/log/mongodb/mongod.log  # ou mongod-02.log, mongod-03.log

replication:
  replSetName: rs0  # DOIT être identique
```

### Création des Répertoires

```bash
# Sur chaque serveur
sudo mkdir -p /data/mongodb
sudo mkdir -p /var/log/mongodb
sudo mkdir -p /var/run/mongodb

sudo chown -R mongodb:mongodb /data/mongodb
sudo chown -R mongodb:mongodb /var/log/mongodb
sudo chown -R mongodb:mongodb /var/run/mongodb

sudo chmod 755 /data/mongodb
sudo chmod 755 /var/log/mongodb
```

## Démarrage des Instances

### Méthode 1 : Systemd (Recommandé)

```bash
# Sur chaque serveur

# 1. Démarrer MongoDB
sudo systemctl start mongod

# 2. Vérifier le statut
sudo systemctl status mongod

# 3. Activer au démarrage
sudo systemctl enable mongod

# 4. Vérifier les logs
sudo tail -f /var/log/mongodb/mongod.log

# 5. Vérifier la connexion
mongosh --host localhost:27017
```

### Méthode 2 : Démarrage Manuel

```bash
# Sur chaque serveur
mongod --config /etc/mongod.conf

# Ou avec numactl
numactl --interleave=all mongod --config /etc/mongod.conf
```

### Vérification Initiale

```bash
# Sur chaque serveur
mongosh --eval "db.serverStatus().ok"
# Devrait retourner : 1

# Vérifier que replication est configuré
mongosh --eval "db.serverStatus().repl"
# Devrait afficher : { setName: 'rs0', ... }
```

## Initialisation du Replica Set

### Connexion au Premier Membre

```bash
# Se connecter au serveur destiné à être Primary
mongosh --host mongodb-01:27017
```

### Configuration Initiale Basique

```javascript
// Configuration minimale
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-01:27017" },
    { _id: 1, host: "mongodb-02:27017" },
    { _id: 2, host: "mongodb-03:27017" }
  ]
})
```

**Réponse attendue** :
```javascript
{
  "ok": 1,
  "$clusterTime": {
    "clusterTime": Timestamp(1705320000, 1),
    "signature": { ... }
  },
  "operationTime": Timestamp(1705320000, 1)
}
```

### Configuration Avancée avec Options

```javascript
// Configuration complète avec toutes les options
rs.initiate({
  _id: "rs0",
  version: 1,

  // Configuration globale
  settings: {
    // Élection
    electionTimeoutMillis: 10000,
    catchUpTimeoutMillis: -1,  // Illimité
    catchUpTakeoverDelayMillis: 30000,

    // Heartbeat
    heartbeatIntervalMillis: 2000,
    heartbeatTimeoutSecs: 10,

    // Réplication
    chainingAllowed: true,

    // Write Concern par défaut (MongoDB 5.0+)
    getLastErrorDefaults: {
      w: "majority",
      wtimeout: 5000
    },

    // Mode de réplication
    getLastErrorModes: {
      datacenter: {
        dc: 2  // Au moins 2 datacenters
      }
    }
  },

  // Membres
  members: [
    {
      _id: 0,
      host: "mongodb-01:27017",
      priority: 2,       // Priorité élevée
      votes: 1,
      tags: {
        dc: "primary",
        rack: "A",
        nodeType: "data"
      }
    },
    {
      _id: 1,
      host: "mongodb-02:27017",
      priority: 1,
      votes: 1,
      tags: {
        dc: "primary",
        rack: "B",
        nodeType: "data"
      }
    },
    {
      _id: 2,
      host: "mongodb-03:27017",
      priority: 1,
      votes: 1,
      tags: {
        dc: "primary",
        rack: "C",
        nodeType: "data"
      }
    }
  ]
})
```

### Vérification de l'Initialisation

```javascript
// 1. Vérifier le statut
rs.status()

// 2. Vérifier la configuration
rs.conf()

// 3. Identifier le Primary
rs.isMaster()

// 4. Afficher les informations de réplication
rs.printReplicationInfo()
rs.printSecondaryReplicationInfo()
```

## Options de Configuration des Membres

### Structure Complète d'un Membre

```javascript
{
  _id: 0,                    // ID unique dans le Replica Set
  host: "mongodb-01:27017",  // Hostname:port

  // Votes et Priorité
  votes: 1,                  // 0 ou 1 (max 7 votants)
  priority: 1,               // 0-1000 (0 = ne peut être Primary)

  // Tags (pour Read Preference et Write Concern)
  tags: {
    dc: "east",
    rack: "A1",
    nodeType: "data",
    region: "us-east-1",
    usage: "production"
  },

  // Options spéciales
  arbiterOnly: false,        // Arbiter (ne stocke pas de données)
  hidden: false,             // Invisible pour les applications
  slaveDelay: 0,             // Délai de réplication (secondes)
  buildIndexes: true,        // Construire les index

  // Horizons (multi-datacenter, cloud)
  horizons: {
    external: "mongodb-01.external.com:27017",
    internal: "10.0.1.10:27017"
  }
}
```

### Priority (Priorité)

Contrôle la préférence pour devenir Primary :

```javascript
// Exemple : 3 nœuds avec priorités différentes
{
  members: [
    {
      _id: 0,
      host: "mongodb-ssd-01:27017",
      priority: 10  // Serveur haute performance - préféré
    },
    {
      _id: 1,
      host: "mongodb-standard-01:27017",
      priority: 5   // Serveur standard
    },
    {
      _id: 2,
      host: "mongodb-backup:27017",
      priority: 0   // Ne devient JAMAIS Primary
    }
  ]
}
```

**Règles** :
- Valeur par défaut : `1`
- Plage : `0` à `1000`
- `priority: 0` → Membre passif (ne peut pas être Primary)
- Plus haute priorité → plus de chances d'être élu

### Votes

Contrôle la participation aux élections :

```javascript
// Configuration avec membres non-votants
{
  members: [
    // Membres votants (max 7)
    { _id: 0, host: "mongodb-01:27017", votes: 1 },
    { _id: 1, host: "mongodb-02:27017", votes: 1 },
    { _id: 2, host: "mongodb-03:27017", votes: 1 },

    // Membres non-votants
    { _id: 3, host: "mongodb-analytics:27017", votes: 0, priority: 0 },
    { _id: 4, host: "mongodb-backup:27017", votes: 0, priority: 0 }
  ]
}
```

**Contraintes** :
- Maximum 7 membres votants par Replica Set
- Maximum 50 membres au total
- Un membre avec `votes: 0` ne participe pas aux élections

### Hidden Members (Membres Cachés)

Membres invisibles pour les applications :

```javascript
{
  _id: 3,
  host: "mongodb-analytics:27017",
  priority: 0,      // Obligatoire
  hidden: true,     // Caché des applications
  votes: 1,         // Peut voter
  tags: {
    usage: "analytics"
  }
}
```

**Cas d'usage** :
- Serveurs analytics/reporting
- Sauvegardes sans impact
- Traitement de données
- Tests de charge

**Note** : Les applications ne peuvent pas lire depuis un hidden member même avec `readPreference: secondary`.

### Delayed Members (Membres Retardés)

Réplication avec délai (protection contre erreurs) :

```javascript
{
  _id: 4,
  host: "mongodb-delayed:27017",
  priority: 0,        // Obligatoire
  hidden: true,       // Recommandé
  slaveDelay: 3600,   // 1 heure de délai
  votes: 1
}
```

**Avantages** :
- Protection contre suppressions accidentelles
- Point de restauration récent
- Détection précoce de corruption

**Attention** : Le delayed member ne doit jamais devenir Primary (`priority: 0`).

### Arbiter

Membre léger qui vote uniquement :

```javascript
{
  _id: 2,
  host: "arbiter:27017",
  arbiterOnly: true,
  votes: 1
}
```

**Caractéristiques** :
- Ne stocke aucune donnée
- Participe aux élections uniquement
- Ressources minimales
- Utile pour nombre pair de membres

**Ajout d'un arbiter** :
```javascript
rs.addArb("arbiter:27017")
```

**Déconseillé** : MongoDB recommande d'utiliser un vrai membre de données plutôt qu'un arbiter.

### Tags (Étiquettes)

Métadonnées pour routage avancé :

```javascript
{
  members: [
    {
      _id: 0,
      host: "mongodb-dc1-01:27017",
      tags: {
        dc: "east",
        region: "us-east-1",
        zone: "us-east-1a",
        nodeType: "ssd",
        workload: "oltp"
      }
    },
    {
      _id: 1,
      host: "mongodb-dc2-01:27017",
      tags: {
        dc: "west",
        region: "us-west-1",
        zone: "us-west-1a",
        nodeType: "ssd",
        workload: "oltp"
      }
    },
    {
      _id: 2,
      host: "mongodb-analytics:27017",
      tags: {
        dc: "east",
        region: "us-east-1",
        zone: "us-east-1b",
        nodeType: "standard",
        workload: "analytics"
      }
    }
  ]
}
```

**Utilisation** :
- Read Preference ciblée
- Write Concern multi-datacenter
- Isolation des workloads

### Horizons (Multi-Réseau)

Pour environnements avec plusieurs réseaux (VPN, cloud hybride) :

```javascript
{
  _id: 0,
  host: "mongodb-01.internal:27017",
  horizons: {
    // Réseau interne
    internal: "10.0.1.10:27017",

    // Réseau externe (VPN)
    external: "mongodb-01.vpn.company.com:27017",

    // Réseau public (avec TLS)
    public: "mongodb-01.cloud.provider.com:27017"
  }
}
```

**Connexion** :
```javascript
// Client interne
mongosh "mongodb://mongodb-01.internal:27017/?replicaSet=rs0"

// Client externe
mongosh "mongodb://mongodb-01.vpn.company.com:27017/?replicaSet=rs0&loadBalanced=false"
```

## Paramètres Globaux (Settings)

### Configuration des Settings

```javascript
cfg = rs.conf()

// Modifier les settings
cfg.settings = {
  // Timeouts d'élection
  electionTimeoutMillis: 10000,           // 10 secondes
  catchUpTimeoutMillis: -1,               // Illimité (ou 30000 pour 30s)
  catchUpTakeoverDelayMillis: 30000,     // 30 secondes

  // Heartbeat
  heartbeatIntervalMillis: 2000,          // Non modifiable en pratique
  heartbeatTimeoutSecs: 10,

  // Réplication
  chainingAllowed: true,                  // Réplication chaînée

  // Write Concern par défaut (MongoDB 4.4+)
  getLastErrorDefaults: {
    w: "majority",
    wtimeout: 5000
  },

  // Write Concern personnalisés
  getLastErrorModes: {
    // Au moins un membre par datacenter
    multiDC: { dc: 2 },

    // Au moins un membre par zone
    multiZone: { zone: 3 },

    // Tous les membres critiques
    critical: { critical: 3 }
  }
}

// Appliquer la configuration
rs.reconfig(cfg)
```

### electionTimeoutMillis

Temps avant qu'un Secondary initie une élection :

```javascript
cfg.settings.electionTimeoutMillis = 10000  // 10 secondes (défaut)

// Réseaux lents/WAN
cfg.settings.electionTimeoutMillis = 30000  // 30 secondes

// Réseaux rapides/LAN
cfg.settings.electionTimeoutMillis = 5000   // 5 secondes
```

### catchUpTimeoutMillis

Durée maximale de la catchup phase après élection :

```javascript
// Illimité (attend que le nouveau Primary soit à jour)
cfg.settings.catchUpTimeoutMillis = -1

// Pas de catchup (risque de perte de données)
cfg.settings.catchUpTimeoutMillis = 0

// Timeout de 30 secondes
cfg.settings.catchUpTimeoutMillis = 30000
```

### chainingAllowed

Active/désactive la réplication chaînée :

```javascript
// Activer (défaut)
cfg.settings.chainingAllowed = true

// Désactiver (tous lisent depuis le Primary)
cfg.settings.chainingAllowed = false
```

**Impact** :
- `true` : Réduit la charge sur le Primary
- `false` : Garantit que tous les Secondary ont les mêmes données

### getLastErrorModes (Write Concern Personnalisé)

Définir des règles de write concern basées sur les tags :

```javascript
// Configuration des membres avec tags
cfg.members = [
  { _id: 0, host: "dc1-01:27017", tags: { dc: "dc1", critical: "yes" } },
  { _id: 1, host: "dc1-02:27017", tags: { dc: "dc1", critical: "yes" } },
  { _id: 2, host: "dc2-01:27017", tags: { dc: "dc2", critical: "no" } },
  { _id: 3, host: "dc2-02:27017", tags: { dc: "dc2", critical: "yes" } }
]

// Définir les modes
cfg.settings.getLastErrorModes = {
  // Écriture sur au moins 2 datacenters
  multiDC: { dc: 2 },

  // Écriture sur tous les membres critiques
  allCritical: { critical: 3 }
}

rs.reconfig(cfg)

// Utilisation
db.orders.insertOne(
  { orderId: 123 },
  { writeConcern: { w: "multiDC", wtimeout: 5000 } }
)
```

## Reconfiguration Dynamique

### Récupérer la Configuration

```javascript
// Méthode 1 : rs.conf()
cfg = rs.conf()

// Méthode 2 : Commande
cfg = db.adminCommand({ replSetGetConfig: 1 }).config
```

### Modifier la Configuration

```javascript
// 1. Récupérer
cfg = rs.conf()

// 2. Modifier (exemple : changer priorité)
cfg.members[0].priority = 10

// 3. Appliquer
rs.reconfig(cfg)
```

### Reconfig avec Force

Pour situations d'urgence (quorum perdu) :

```javascript
// ATTENTION : Peut causer des incohérences
rs.reconfig(cfg, { force: true })
```

**Utilisation** :
- Majorité des membres down
- Récupération d'urgence uniquement
- Risque de split-brain

### Ajouter un Membre

```javascript
// Méthode 1 : rs.add()
rs.add("mongodb-04:27017")

// Méthode 2 : Avec options
rs.add({
  host: "mongodb-04:27017",
  priority: 1,
  votes: 1,
  tags: { dc: "west" }
})

// Méthode 3 : Via rs.reconfig()
cfg = rs.conf()
cfg.members.push({
  _id: 3,
  host: "mongodb-04:27017",
  priority: 1,
  votes: 1
})
rs.reconfig(cfg)
```

### Supprimer un Membre

```javascript
// Méthode 1 : rs.remove()
rs.remove("mongodb-04:27017")

// Méthode 2 : Via rs.reconfig()
cfg = rs.conf()
cfg.members = cfg.members.filter(m => m.host !== "mongodb-04:27017")
rs.reconfig(cfg)
```

### Modifier un Membre Existant

```javascript
cfg = rs.conf()

// Trouver le membre
var member = cfg.members.find(m => m.host === "mongodb-02:27017")

// Modifier les propriétés
member.priority = 5
member.tags = { dc: "east", rack: "A" }

// Appliquer
rs.reconfig(cfg)
```

### Reconfiguration Non-Disruptive

Pour minimiser l'impact :

```javascript
// 1. Modifier un Secondary d'abord
cfg = rs.conf()
cfg.members[1].priority = 5
rs.reconfig(cfg)

// 2. Attendre la stabilisation
sleep(5000)

// 3. Modifier d'autres membres
cfg = rs.conf()
cfg.members[2].priority = 3
rs.reconfig(cfg)
```

## Configurations Avancées

### Configuration Multi-Datacenter

```javascript
rs.initiate({
  _id: "rs0",

  settings: {
    electionTimeoutMillis: 30000,  // 30s pour WAN
    heartbeatTimeoutSecs: 20,
    getLastErrorModes: {
      multiDC: { dc: 2 }
    }
  },

  members: [
    // Datacenter 1 (Principal)
    {
      _id: 0,
      host: "dc1-mongodb-01:27017",
      priority: 10,
      tags: { dc: "dc1", region: "us-east-1" }
    },
    {
      _id: 1,
      host: "dc1-mongodb-02:27017",
      priority: 9,
      tags: { dc: "dc1", region: "us-east-1" }
    },
    {
      _id: 2,
      host: "dc1-mongodb-03:27017",
      priority: 8,
      tags: { dc: "dc1", region: "us-east-1" }
    },

    // Datacenter 2 (DR)
    {
      _id: 3,
      host: "dc2-mongodb-01:27017",
      priority: 1,
      tags: { dc: "dc2", region: "us-west-1" }
    },
    {
      _id: 4,
      host: "dc2-mongodb-02:27017",
      priority: 1,
      votes: 0,  // Non-votant pour éviter split-brain
      tags: { dc: "dc2", region: "us-west-1" }
    }
  ]
})
```

### Configuration Asymétrique (PSA)

Primary-Secondary-Arbiter (non recommandé pour production) :

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongodb-primary:27017" },
    { _id: 1, host: "mongodb-secondary:27017" },
    { _id: 2, host: "arbiter:27017", arbiterOnly: true }
  ]
})
```

**Problèmes** :
- Pas de haute disponibilité réelle
- Write concern "majority" = 2 nœuds (Primary + Arbiter ou Secondary)
- Pas de protection contre perte du Primary

**Alternative recommandée** : PSS (3 data members)

### Configuration avec Delayed Backup

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    // Membres principaux
    { _id: 0, host: "mongodb-01:27017", priority: 2 },
    { _id: 1, host: "mongodb-02:27017", priority: 1 },
    { _id: 2, host: "mongodb-03:27017", priority: 1 },

    // Membre retardé (protection)
    {
      _id: 3,
      host: "mongodb-delayed:27017",
      priority: 0,
      hidden: true,
      slaveDelay: 3600,  // 1 heure
      tags: { backup: "delayed" }
    },

    // Membre analytics
    {
      _id: 4,
      host: "mongodb-analytics:27017",
      priority: 0,
      hidden: true,
      votes: 0,
      tags: { usage: "analytics" }
    }
  ]
})
```

### Configuration Cloud Hybride

```javascript
rs.initiate({
  _id: "rs0",

  members: [
    // On-premise (priorité haute)
    {
      _id: 0,
      host: "onprem-01.internal:27017",
      priority: 10,
      tags: { environment: "onprem", dc: "datacenter1" },
      horizons: {
        internal: "10.0.1.10:27017",
        vpn: "onprem-01.vpn.company.com:27017"
      }
    },
    {
      _id: 1,
      host: "onprem-02.internal:27017",
      priority: 9,
      tags: { environment: "onprem", dc: "datacenter1" }
    },

    // Cloud (DR)
    {
      _id: 2,
      host: "cloud-01.aws.company.com:27017",
      priority: 1,
      tags: { environment: "cloud", provider: "aws", region: "us-east-1" },
      horizons: {
        internal: "172.31.10.10:27017",
        public: "cloud-01.aws.company.com:27017"
      }
    }
  ],

  settings: {
    electionTimeoutMillis: 30000,  // Latence WAN
    getLastErrorModes: {
      distributed: { environment: 2 }  // Un membre on-prem + un cloud
    }
  }
})
```

## Sécurisation du Replica Set

### Génération du KeyFile

```bash
# 1. Générer le keyfile sur le Primary
openssl rand -base64 756 > /etc/mongodb/keyfile

# 2. Définir les permissions
chmod 400 /etc/mongodb/keyfile
chown mongodb:mongodb /etc/mongodb/keyfile

# 3. Copier sur tous les membres
scp /etc/mongodb/keyfile mongodb-02:/etc/mongodb/keyfile
scp /etc/mongodb/keyfile mongodb-03:/etc/mongodb/keyfile

# 4. Permissions sur les autres membres
ssh mongodb-02 "chmod 400 /etc/mongodb/keyfile && chown mongodb:mongodb /etc/mongodb/keyfile"
ssh mongodb-03 "chmod 400 /etc/mongodb/keyfile && chown mongodb:mongodb /etc/mongodb/keyfile"
```

### Activation de l'Authentification

```yaml
# /etc/mongod.conf sur tous les membres
security:
  authorization: enabled
  keyFile: /etc/mongodb/keyfile
  clusterAuthMode: keyFile
```

### Création des Utilisateurs

```javascript
// 1. Se connecter au Primary AVANT d'activer l'auth
mongosh --host mongodb-01:27017

// 2. Créer l'admin
use admin
db.createUser({
  user: "admin",
  pwd: "SecurePassword123!",
  roles: [
    { role: "root", db: "admin" }
  ]
})

// 3. Créer un utilisateur application
use myapp
db.createUser({
  user: "appuser",
  pwd: "AppPassword456!",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})

// 4. Créer un utilisateur monitoring
use admin
db.createUser({
  user: "monitoring",
  pwd: "MonitorPass789!",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})
```

### Redémarrage avec Authentification

```bash
# Sur chaque membre
sudo systemctl restart mongod

# Vérifier
mongosh --host mongodb-01:27017 -u admin -p SecurePassword123! --authenticationDatabase admin
```

### Connexion String avec Authentification

```javascript
// Node.js driver
const uri = "mongodb://appuser:AppPassword456!@mongodb-01:27017,mongodb-02:27017,mongodb-03:27017/myapp?replicaSet=rs0&authSource=myapp"

// mongosh
mongosh "mongodb://admin:SecurePassword123!@mongodb-01:27017,mongodb-02:27017,mongodb-03:27017/?replicaSet=rs0&authSource=admin"
```

## Validation et Tests

### Vérifications Post-Configuration

```javascript
// 1. Statut global
rs.status()

// 2. Configuration
rs.conf()

// 3. Membres et états
rs.status().members.forEach(m => {
  print(m.name + " - " + m.stateStr + " - Health: " + m.health)
})

// 4. Replication lag
rs.printSecondaryReplicationInfo()

// 5. Oplog
rs.printReplicationInfo()

// 6. Connexions
db.serverStatus().connections

// 7. Réplication
db.serverStatus().repl
```

### Test de Failover

```javascript
// 1. Identifier le Primary
var primary = rs.status().members.find(m => m.state === 1)
print("Primary: " + primary.name)

// 2. Simuler la défaillance (sur le Primary)
db.adminCommand({ shutdown: 1 })

// 3. Observer l'élection (sur un Secondary)
while(true) {
  var status = rs.status()
  var newPrimary = status.members.find(m => m.state === 1)
  if (newPrimary) {
    print("New Primary: " + newPrimary.name)
    break
  }
  sleep(1000)
}

// 4. Redémarrer l'ancien Primary
// Il rejoindra comme Secondary
```

### Test de Write Concern

```javascript
// Test majority
db.test.insertOne(
  { test: "majority" },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)

// Test custom mode (si configuré)
db.test.insertOne(
  { test: "multiDC" },
  { writeConcern: { w: "multiDC", wtimeout: 5000 } }
)

// Test tous les membres
db.test.insertOne(
  { test: "all" },
  { writeConcern: { w: 3, wtimeout: 5000 } }
)
```

### Test de Read Preference

```javascript
// Primary
db.test.find().readPref("primary")

// Secondary
db.test.find().readPref("secondary")

// PrimaryPreferred
db.test.find().readPref("primaryPreferred")

// SecondaryPreferred
db.test.find().readPref("secondaryPreferred")

// Nearest
db.test.find().readPref("nearest")

// Avec tags
db.test.find().readPref("secondary", [{ dc: "east" }])
```

## Bonnes Pratiques

### 1. Topologie

✅ **Recommandations** :
- Toujours un nombre impair de membres votants (3, 5, 7)
- PSS (3 data members) minimum pour production
- Éviter PSA (Primary-Secondary-Arbiter)
- Maximum 7 votants, jusqu'à 50 membres totaux

❌ **À éviter** :
- 2 membres seulement (pas de majorité possible)
- Trop de membres (overhead de réplication)
- Arbiter si possible (préférer un vrai membre)

### 2. Priorités

```javascript
// ✅ Bon : Priorités distinctes
{ priority: 10 }  // Serveur haute performance
{ priority: 5 }   // Serveur standard
{ priority: 1 }   // Serveur backup/DR

// ❌ Mauvais : Toutes les priorités identiques
{ priority: 1 }
{ priority: 1 }
{ priority: 1 }
// Élections indéterministes
```

### 3. Géo-Distribution

```javascript
// ✅ Bon : Majorité dans le DC principal
DC1: 3 membres votants (majorité)
DC2: 2 membres (1 votant, 1 non-votant)

// ❌ Mauvais : Répartition égale
DC1: 2 membres
DC2: 2 membres
// Perte de DC = perte de majorité
```

### 4. Write Concern par Défaut

```javascript
// ✅ Recommandé pour production
cfg.settings.getLastErrorDefaults = {
  w: "majority",
  wtimeout: 5000
}

// ❌ À éviter
w: 1  // Risque de perte de données
```

### 5. Monitoring

```javascript
// Script de monitoring quotidien
function dailyReplicaSetCheck() {
  var checks = []

  // Check 1: Tous les membres sont UP
  var status = rs.status()
  status.members.forEach(m => {
    if (m.health !== 1 || (m.state !== 1 && m.state !== 2)) {
      checks.push("WARN: " + m.name + " is " + m.stateStr)
    }
  })

  // Check 2: Replication lag < 60s
  status.members.forEach(m => {
    if (m.state === 2) {  // SECONDARY
      var lag = (status.date - m.optimeDate) / 1000
      if (lag > 60) {
        checks.push("WARN: " + m.name + " lag is " + lag + "s")
      }
    }
  })

  // Check 3: Oplog window > 24h
  var replInfo = db.getReplicationInfo()
  var oplogHours = replInfo.timeDiff / 3600
  if (oplogHours < 24) {
    checks.push("WARN: Oplog window is " + oplogHours + "h")
  }

  return checks.length === 0 ? "All checks passed" : checks
}

dailyReplicaSetCheck()
```

## Dépannage

### Membre Ne Rejoint Pas le Replica Set

**Symptômes** :
```
Member state: STARTUP or STARTUP2
```

**Solutions** :
```javascript
// 1. Vérifier la connectivité réseau
rs.status().members

// 2. Vérifier le keyfile (si auth activée)
// Doit être identique sur tous les membres

// 3. Vérifier les logs
db.adminCommand({ getLog: "global" })

// 4. Forcer une resynchronisation
// Sur le membre problématique
db.adminCommand({ resync: 1 })
```

### Élections Fréquentes

**Diagnostic** :
```javascript
// Vérifier les élections récentes
rs.status().members.forEach(m => {
  if (m.electionTime) {
    print(m.name + " - Last election: " + m.electionDate)
  }
})
```

**Solutions** :
- Augmenter `electionTimeoutMillis`
- Vérifier la latence réseau
- Vérifier les ressources (CPU, mémoire)
- Stabiliser les priorités

### Configuration Corrompue

**Récupération** :
```javascript
// Sur le Primary
var cfg = {
  _id: "rs0",
  version: 1,
  members: [
    { _id: 0, host: "mongodb-01:27017" },
    { _id: 1, host: "mongodb-02:27017" },
    { _id: 2, host: "mongodb-03:27017" }
  ]
}

// Force reconfig (DANGER)
rs.reconfig(cfg, { force: true })
```

## Conclusion

La configuration d'un Replica Set MongoDB est un processus qui nécessite :

- ✅ **Planification minutieuse** de la topologie
- ✅ **Compréhension des options** de configuration
- ✅ **Tests rigoureux** avant la production
- ✅ **Monitoring continu** après déploiement
- ✅ **Documentation** des choix architecturaux

**Points clés** :
1. Toujours un nombre impair de membres votants
2. Configuration des priorités selon l'infrastructure
3. Write Concern "majority" par défaut
4. Sécurisation avec keyFile/x.509
5. Tests de failover réguliers
6. Monitoring proactif des métriques clés

Une configuration correcte du Replica Set garantit la haute disponibilité, la cohérence des données et la résilience de votre infrastructure MongoDB.

⏭️ [Ajout et suppression de membres](/09-replication/07-ajout-suppression-membres.md)
