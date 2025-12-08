🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.2 Architecture Shardée

## Introduction

L'architecture shardée de MongoDB est un système distribué complexe composé de plusieurs types de nœuds travaillant ensemble pour fournir scalabilité horizontale, haute disponibilité et performance. Comprendre cette architecture en profondeur est essentiel pour concevoir, déployer et maintenir un cluster en production.

Cette section détaille l'architecture complète d'un cluster shardé, les interactions entre composants, les flux de communication et les décisions architecturales critiques.

## Vue d'Ensemble de l'Architecture

Un cluster shardé MongoDB est constitué de **trois types de composants** qui collaborent pour former un système distribué cohérent :

```
┌───────────────────────────────────────────────────────────────┐
│                    ARCHITECTURE SHARDÉE MONGODB               │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────────────────────────────────┐     │
│  │              COUCHE APPLICATION                      │     │
│  │  (Drivers MongoDB, Applications, Services)           │     │
│  └────────────────────┬─────────────────────────────────┘     │
│                       │ Connection String                     │
│                       │ mongodb://mongos1,mongos2,mongos3     │
│                       ▼                                       │
│  ┌──────────────────────────────────────────────────────┐     │
│  │           COUCHE ROUTAGE (Query Routers)             │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │     │
│  │  │ Mongos 1 │  │ Mongos 2 │  │ Mongos N │            │     │
│  │  │ (stateless) │(stateless)  │(stateless)            │     │
│  │  └─────┬────┘  └─────┬────┘  └─────┬────┘            │     │
│  └────────┼─────────────┼─────────────┼─────────────────┘     │
│           │             │             │                       │
│  ┌────────┼─────────────┼─────────────┼────────────────┐      │
│  │        │   COUCHE MÉTADONNÉES      │                │      │
│  │        │  ┌──────────────────────┐ │                │      │
│  │        └─►│  Config Server RS    │◄┘                │      │
│  │           │  ┌────┐ ┌────┐ ┌────┐│                  │      │
│  │           │  │CS 1│ │CS 2│ │CS 3││                  │      │
│  │           │  │(P) │ │(S) │ │(S) ││                  │      │
│  │           │  └────┘ └────┘ └────┘│                  │      │
│  │           └──────────────────────┘                  │      │
│  └─────────────────────────────────────────────────────┘      │
│                       │                                       │
│  ┌────────────────────┼────────────────────────────────┐      │
│  │        COUCHE DONNÉES (Shards)                      │      │
│  │                    │                                │      │
│  │  ┌─────────────┐   │  ┌─────────────┐  ┌──────────┐ │      │
│  │  │  Shard A    │   │  │  Shard B    │  │ Shard N  │ │      │
│  │  │  (RS)       │   │  │  (RS)       │  │  (RS)    │ │      │
│  │  │ ┌──┐┌──┐┌──┐│   │  │ ┌──┐┌──┐┌──┐│  │┌──┐┌──┐┌──┐│      │
│  │  │ │P ││S ││S ││   │  │ │P ││S ││S ││  ││P ││S ││S ││      │
│  │  │ └──┘└──┘└──┘│   │  │ └──┘└──┘└──┘│  │└──┘└──┘└──┘│      │
│  │  │  Chunks:    │   │  │  Chunks:    │  │ Chunks:  │ │      │
│  │  │  1,2,3,7,8  │   │  │  4,5,6,9,10 │  │ 11,12,13 │ │      │
│  │  └─────────────┘   │  └─────────────┘  └──────────┘ │      │
│  └────────────────────┼────────────────────────────────┘      │
│                       │                                       │
└───────────────────────┼───────────────────────────────────────┘
                        │
             Communication Réseau (TCP/IP)
```

### Les Trois Piliers

**1. Query Routers (Mongos)**
- **Rôle** : Point d'entrée pour les applications
- **Nature** : Processus légers, stateless, multiples instances
- **Fonction** : Routage intelligent des requêtes vers les shards appropriés

**2. Config Servers (CSRS)**
- **Rôle** : Stockage des métadonnées du cluster
- **Nature** : Replica Set de 3+ membres
- **Fonction** : Source de vérité pour la topologie et le mapping chunks→shards

**3. Shards**
- **Rôle** : Stockage effectif des données
- **Nature** : Replica Sets indépendants
- **Fonction** : Hébergement des chunks, exécution des requêtes

## Composants Détaillés

### 1. Query Routers (Mongos)

Les instances **mongos** sont les processus qui reçoivent toutes les requêtes des applications et les routent vers les shards appropriés.

#### Caractéristiques Architecturales

```
┌────────────────────────────────────────────────┐
│           Instance Mongos                      │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │     Connection Pool Manager          │      │
│  │  - Connexions vers Config Servers    │      │
│  │  - Connexions vers Shards            │      │
│  │  - Pool sizing dynamique             │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │     Metadata Cache                   │      │
│  │  - Mapping chunks → shards           │      │
│  │  - Shard keys des collections        │      │
│  │  - Database primary shards           │      │
│  │  - TTL: 30 secondes (configurable)   │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │     Query Router & Planner           │      │
│  │  - Analyse des requêtes              │      │
│  │  - Détermination des shards cibles   │      │
│  │  - Optimisation du plan d'exécution  │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │     Result Merger                    │      │
│  │  - Combine résultats multi-shards    │      │
│  │  - Apply sort/limit globaux          │      │
│  │  - Déduplication (si nécessaire)     │      │
│  └──────────────────────────────────────┘      │
│                                                │
│  ┌──────────────────────────────────────┐      │
│  │     Transaction Coordinator          │      │
│  │  - Gestion 2PC (Two-Phase Commit)    │      │
│  │  - Coordination multi-shard txns     │      │
│  └──────────────────────────────────────┘      │
│                                                │
└────────────────────────────────────────────────┘
```

#### Placement et Déploiement

**Option 1 : Mongos Co-localisés avec les Applications**
```
┌─────────────────────────────────┐
│     Serveur Application         │
│  ┌──────────────┐               │
│  │  App Process │               │
│  └───────┬──────┘               │
│          │ localhost:27017      │
│          ▼                      │
│  ┌──────────────┐               │
│  │    Mongos    │               │
│  └───────┬──────┘               │
│          │                      │
└──────────┼──────────────────────┘
           │ Network
           ▼ To Shards/Config
```

**Avantages :**
- ✅ Latence minimale (connexion locale)
- ✅ Isolation des ressources par application
- ✅ Scaling horizontal naturel (mongos scale avec les apps)

**Inconvénients :**
- ❌ Duplication des processus mongos
- ❌ Gestion décentralisée
- ❌ Consommation mémoire par serveur app

**Option 2 : Mongos Standalone Dédiés**
```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ App Server 1 │  │ App Server 2 │  │ App Server N │
└──────┬───────┘  └──────┬───────┘  └──────┬───────┘
       │                 │                 │
       └────────┬────────┴────────┬────────┘
                │                 │
       ┌────────▼─────┐  ┌────────▼─────┐
       │  Mongos 1    │  │  Mongos 2    │
       │  (Dedicated) │  │  (Dedicated) │
       └──────────────┘  └──────────────┘
```

**Avantages :**
- ✅ Gestion centralisée des mongos
- ✅ Ressources dédiées pour le routage
- ✅ Facilite le monitoring et le troubleshooting

**Inconvénients :**
- ❌ Latence réseau supplémentaire
- ❌ Infrastructure additionnelle à gérer
- ❌ Potentiel point de contention

**Recommandation Production :**
```
Petite échelle (< 10 app servers):
→ Mongos dédiés (2-3 instances)

Moyenne échelle (10-50 app servers):
→ Mix: Mongos dédiés + co-localisés

Grande échelle (> 50 app servers):
→ Mongos co-localisés avec load balancer devant
```

#### Dimensionnement Mongos

```javascript
// Formule empirique pour le nombre de mongos

// Facteurs à considérer:
connexions_totales = nb_app_servers × connexions_par_app
requêtes_par_sec = charge_totale / durée_moyenne_requête

// Règle de base:
mongos_instances = MAX(
  2,  // Minimum pour HA
  CEIL(connexions_totales / 1000),  // 1 mongos par 1000 connexions
  CEIL(requêtes_par_sec / 50000)    // 1 mongos par 50k req/sec
)

// Exemple:
// - 20 app servers
// - 100 connexions par app = 2000 connexions totales
// - 80,000 requêtes/sec
// → mongos_instances = MAX(2, CEIL(2000/1000), CEIL(80000/50000))
// → mongos_instances = MAX(2, 2, 2) = 2 mongos minimum
// → Recommandé: 3-4 mongos pour tolérance aux pannes
```

#### Ressources Mongos

```yaml
# Configuration typique d'un mongos
CPU: 2-4 cores (léger, I/O bound)
RAM: 512 MB - 2 GB
  - Metadata cache: 50-200 MB
  - Connection pools: ~100 MB par 1000 connexions
  - Buffers de merge: variable selon la charge

Réseau:
  - Bande passante critique (proxy de toutes les données)
  - 10 Gbps minimum en production haute charge

Disque:
  - Minimal (logs uniquement)
  - 10-20 GB suffisant
```

### 2. Config Servers (CSRS)

Le **Config Server Replica Set** (CSRS) est le cerveau du cluster, stockant toutes les métadonnées critiques.

#### Architecture CSRS

```
┌────────────────────────────────────────────────────┐
│   Config Server Replica Set (3 membres minimum)    │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────┐  ┌──────────────┐   ┌──────────┐ │
│  │   Primary    │  │  Secondary   │   │Secondary │ │
│  │   (CSR-1)    │  │   (CSR-2)    │   │ (CSR-3)  │ │
│  │              │  │              │   │          │ │
│  │  Port: 27019 │  │  Port: 27019 │   │Port:27019│ │
│  └──────┬───────┘  └──────┬───────┘   └─────┬────┘ │
│         │                 │                 │      │
│         └────────┬────────┴────────┬────────┘      │
│                  │ Replication     │               │
│                  ▼                 ▼               │
│         ┌────────────────────────────────┐         │
│         │    Database: config            │         │
│         │  ┌──────────────────────────┐  │         │
│         │  │ Collections:             │  │         │
│         │  │  - chunks                │  │         │
│         │  │  - databases             │  │         │
│         │  │  - collections           │  │         │
│         │  │  - shards                │  │         │
│         │  │  - locks                 │  │         │
│         │  │  - version               │  │         │
│         │  │  - mongos                │  │         │
│         │  │  - changelog             │  │         │
│         │  │  - settings              │  │         │
│         │  │  - tags                  │  │         │
│         │  └──────────────────────────┘  │         │
│         └────────────────────────────────┘         │
│                                                    │
└────────────────────────────────────────────────────┘
```

#### Collections Critiques dans config Database

**1. config.chunks**
```javascript
// Mapping définitif chunks → shards
{
  "_id": ObjectId("674f8e9a1234567890abcdef"),
  "uuid": UUID("a1b2c3d4-..."),  // UUID de la collection
  "min": { "customer_id": 1000, "order_date": ISODate("2024-01-01") },
  "max": { "customer_id": 2000, "order_date": ISODate("2024-02-01") },
  "shard": "shard-a-replica-set",
  "lastmod": Timestamp(1701456789, 5),
  "history": [
    {
      "validAfter": Timestamp(1701456789, 5),
      "shard": "shard-a-replica-set"
    },
    // Historique des migrations
    {
      "validAfter": Timestamp(1701356789, 2),
      "shard": "shard-b-replica-set"
    }
  ]
}
```

**2. config.collections**
```javascript
// Métadonnées des collections shardées
{
  "_id": "ecommerce.orders",
  "lastmodEpoch": ObjectId("674f8e9a1234567890abcdef"),
  "lastmod": ISODate("2024-12-08T10:30:00Z"),
  "dropped": false,
  "key": { "customer_id": 1, "order_date": 1 },  // Shard key
  "unique": false,
  "uuid": UUID("a1b2c3d4-e5f6-7890-abcd-ef1234567890"),
  "noBalance": false  // Si true, balancer désactivé pour cette collection
}
```

**3. config.databases**
```javascript
// Métadonnées des databases
{
  "_id": "ecommerce",
  "primary": "shard-a-replica-set",  // Primary shard (pour collections non-shardées)
  "partitioned": true,  // Database est shardée
  "version": {
    "uuid": UUID("..."),
    "lastMod": 1
  }
}
```

**4. config.shards**
```javascript
// Liste et état des shards
{
  "_id": "shard-a-replica-set",
  "host": "shard-a-replica-set/mongo-a1:27018,mongo-a2:27018,mongo-a3:27018",
  "state": 1,  // 1 = active, 0 = draining
  "tags": ["eu", "premium", "ssd"],  // Tags pour zone sharding
  "topologyTime": Timestamp(1701456789, 1)
}
```

**5. config.locks**
```javascript
// Verrous distribués pour opérations critiques
{
  "_id": "ecommerce.orders-moveChunk-chunk_123",
  "state": 2,  // 0 = unlocked, 1 = contended, 2 = locked
  "process": "ConfigServer",
  "ts": ObjectId("674f8e9a1234567890abcdef"),
  "when": ISODate("2024-12-08T10:30:00Z"),
  "who": "shard-a-replica-set:Balancer:12345:1701456789:0",
  "why": "Moving chunk [1000, 2000) from shard-a to shard-b"
}
```

**6. config.changelog**
```javascript
// Journal d'audit des opérations cluster
{
  "_id": "shard-a-replica-set-2024-12-08T10:30:00-674f8e9a1234567890abcdef",
  "server": "shard-a-replica-set",
  "clientAddr": "192.168.1.10:54321",
  "time": ISODate("2024-12-08T10:30:00Z"),
  "what": "moveChunk.commit",
  "ns": "ecommerce.orders",
  "details": {
    "min": { "customer_id": 1000 },
    "max": { "customer_id": 2000 },
    "from": "shard-a-replica-set",
    "to": "shard-b-replica-set",
    "cloneTime": 1500,  // ms
    "catchUpTime": 200,  // ms
    "deleteTime": 100    // ms
  }
}
```

#### Dimensionnement Config Servers

```yaml
# Configuration typique Config Server
CPU: 2-4 cores
RAM: 4-8 GB
  - WiredTiger cache: 2-4 GB
  - Metadata: 100 MB - 2 GB (selon nombre de chunks)

Disque:
  - Type: SSD recommandé (metadata I/O critique)
  - Taille: 25-100 GB
  - IOPS: 1000+ recommandé

Réseau:
  - Latence critique (consulté fréquemment par mongos)
  - Colocalisé avec mongos si possible

Nombre de membres:
  - Production: 3 membres (minimum absolu)
  - Haute disponibilité critique: 5 membres
  - Jamais 2 (pas de majorité en cas de split-brain)
```

#### Calcul de la Taille des Métadonnées

```javascript
// Estimation de la taille de config database

// Formule:
taille_config_db = (
  nb_collections_shardées × 500 bytes +     // config.collections
  nb_chunks_total × 350 bytes +             // config.chunks
  nb_shards × 300 bytes +                   // config.shards
  nb_zones × 200 bytes +                    // config.tags
  changelog_entries × 400 bytes             // config.changelog (rolling)
)

// Exemple concret:
// - 100 collections shardées
// - 500,000 chunks (très gros cluster)
// - 20 shards
// - 50 zones
// - 100,000 changelog entries (dernière semaine)

taille_estimée = (
  100 × 500 +
  500000 × 350 +
  20 × 300 +
  50 × 200 +
  100000 × 400
) bytes

= 50 KB + 175 MB + 6 KB + 10 KB + 40 MB
≈ 215 MB

// Avec indexes et overhead WiredTiger: ~500 MB - 1 GB
```

### 3. Shards (Replica Sets de Données)

Chaque **shard** est un Replica Set complet qui héberge un sous-ensemble des données.

#### Architecture d'un Shard

```
┌────────────────────────────────────────────────────┐
│         Shard A (Replica Set)                      │
├────────────────────────────────────────────────────┤
│                                                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │
│  │   Primary    │  │  Secondary   │  │Secondary │  │
│  │   (mongo-a1) │  │  (mongo-a2)  │  │(mongo-a3)│  │
│  │              │  │              │  │          │  │
│  │  Port: 27018 │  │  Port: 27018 │  │Port:27018│  │
│  │              │  │              │  │          │  │
│  │  ┌────────┐  │  │  ┌────────┐  │  │┌────────┐│  │
│  │  │ Data   │  │  │  │ Data   │  │  ││ Data   ││  │
│  │  │ Chunks:│  │  │  │ Chunks:│  │  ││Chunks: ││  │
│  │  │ 1,2,3  │  │  │  │ 1,2,3  │  │  ││ 1,2,3  ││  │
│  │  │ 7,8,15 │  │  │  │ 7,8,15 │  │  ││7,8,15  ││  │
│  │  └────────┘  │  │  └────────┘  │  │└────────┘│  │
│  └──────┬───────┘  └──────┬───────┘  └─────┬────┘  │
│         │                 │                │       │
│         └────────┬────────┴────────┬───────┘       │
│                  │ Replication     │               │
│                  │ (Oplog)         │               │
│                  └─────────────────┘               │
│                                                    │
│  Caractéristiques:                                 │
│  - Stocke un sous-ensemble des données             │
│  - Exécute les requêtes routées par mongos         │
│  - Participe au balancing (source/destination)     │
│  - Indépendant des autres shards                   │
│                                                    │
└────────────────────────────────────────────────────┘
```

Les détails spécifiques des shards seront approfondis dans la section 10.2.1.

## Communication Inter-Composants

### Flux de Communication

```
┌────────────────────────────────────────────────────────┐
│                FLUX DE COMMUNICATION                   │
└────────────────────────────────────────────────────────┘

1. Application → Mongos
   Protocol: MongoDB Wire Protocol
   Port: 27017 (standard)
   Fréquence: Continue (toutes les requêtes)

2. Mongos → Config Servers
   Protocol: MongoDB Wire Protocol
   Port: 27019 (standard config)
   Fréquence:
     - Refresh metadata: toutes les 30s (ou on-demand)
     - Heartbeat: toutes les 10s

3. Mongos → Shards
   Protocol: MongoDB Wire Protocol
   Port: 27018 (standard shard)
   Fréquence: Continue (exécution des requêtes)

4. Config Servers ↔ Config Servers
   Protocol: MongoDB Replication Protocol
   Port: 27019
   Fréquence: Continue (replication)

5. Shard Members ↔ Shard Members
   Protocol: MongoDB Replication Protocol
   Port: 27018
   Fréquence: Continue (replication intra-shard)

6. Balancer (via mongos ou primary config) → Shards
   Protocol: MongoDB Wire Protocol
   Port: 27018
   Fréquence: Périodique (fenêtre de balancing)
```

### Diagramme de Séquence : Requête Find

```
Application    Mongos         Config          Shard A       Shard B
    │             │           Servers            │             │
    │─find()─────►│             │                │             │
    │             │             │                │             │
    │             │─query       │                │             │
    │             │ metadata───►│                │             │
    │             │             │                │             │
    │             │◄─chunk      │                │             │
    │             │  locations──┘                │             │
    │             │                              │             │
    │             │  (Determine: Shard A only)   │             │
    │             │                              │             │
    │             │─find()──────────────────────►│             │
    │             │                              │             │
    │             │                              │─execute─┐   │
    │             │                              │         │   │
    │             │                              │◄────────┘   │
    │             │                              │             │
    │             │◄─results────────────────────┘              │
    │             │                                            │
    │             │  (No merge needed, single shard)           │
    │             │                                            │
    │◄─results────┘                                            │
    │             │                                            │
```

### Diagramme de Séquence : Requête Scatter-Gather

```
Application    Mongos         Config        Shard A    Shard B    Shard C
    │             │           Servers          │          │          │
    │─find()─────►│             │              │          │          │
    │  (no shard  │             │              │          │          │
    │   key)      │             │              │          │          │
    │             │─query       │              │          │          │
    │             │ metadata───►│              │          │          │
    │             │◄─all chunks─┘              │          │          │
    │             │                            │          │          │
    │             │  (Determine: ALL shards)   │          │          │
    │             │                            │          │          │
    │             │─find()────────────────────►│          │          │
    │             │─find()─────────────────────┼─────────►│          │
    │             │─find()─────────────────────┼──────────┼─────────►│
    │             │                            │          │          │
    │             │                            │─execute─┐│          │
    │             │                            │◄────────┘│          │
    │             │                            │          │─execute─┐│
    │             │                            │          │◄────────┘│
    │             │                            │          │          │─execute─┐
    │             │                            │          │          │◄────────┘
    │             │                            │          │          │
    │             │◄─results A─────────────────┘          │          │
    │             │◄─results B────────────────────────────┘          │
    │             │◄─results C───────────────────────────────────────┘
    │             │                                                  │
    │             │  ┌─────────────────────┐                         │
    │             │  │ MERGE RESULTS:      │                         │
    │             │  │ - Combine A+B+C     │                         │
    │             │  │ - Apply sort/limit  │                         │
    │             │  │ - Deduplicate       │                         │
    │             │  └─────────────────────┘                         │
    │             │                                                  │
    │◄─results────┘                                                  │
    │             │                                                  │
```

### Diagramme de Séquence : Migration de Chunk

```
Balancer       Mongos       Config        Shard A        Shard B
(mongos)                   Servers      (Source)    (Destination)
   │             │            │             │              │
   │─Check balance            │             │              │
   │  needed────►│            │             │              │
   │             │            │             │              │
   │             │─Query─────►│             │              │
   │             │  chunks    │             │              │
   │             │◄─Imbalanced┘             │              │
   │             │  detected                │              │
   │             │                          │              │
   │◄─Decision───┘                          │              │
   │  Move chunk_42: A→B                    │              │
   │                                        │              │
   │─Acquire lock──────────►│               │              │
   │                        │               │              │
   │◄─Lock granted──────────┘               │              │
   │                                        │              │
   │─moveChunk─────────────────────────────►│              │
   │  command                               │              │
   │                                        │              │
   │                                        │─Clone chunk─►│
   │                                        │  data        │
   │                                        │              │
   │                                        │◄─Clone done──┘
   │                                        │              │
   │                                        │─Sync oplog──►│
   │                                        │  (incremental)
   │                                        │              │
   │                                        │◄─Sync done───┘
   │                                        │              │
   │                                        │─Commit──────►│
   │                                        │              │
   │◄─Update metadata───────────────────────┼──────────────┘
   │                                        │              │
   │─Update config.chunks────►│             │              │
   │                          │             │              │
   │◄─Updated─────────────────┘             │              │
   │                                        │              │
   │─Delete old chunk──────────────────────►│              │
   │                                        │              │
   │◄─Deleted───────────────────────────────┘              │
   │                                                       │
   │─Release lock──────────►│                              │
   │                        │                              │
   │◄─Lock released─────────┘                              │
   │                                                       │
```

## Topologies de Déploiement

### Topologie Minimale (Dev/Test)

```
┌────────────────────────────────────────────────┐
│     TOPOLOGIE MINIMALE (DEV/TEST ONLY)         │
├────────────────────────────────────────────────┤
│                                                │
│  ┌──────────┐                                  │
│  │  Mongos  │  (1 instance)                    │
│  └────┬─────┘                                  │
│       │                                        │
│  ┌────┴─────────────┐                          │
│  │                  │                          │
│  ▼                  ▼                          │
│ ┌─────────┐    ┌──────────┐                    │
│ │ Config  │    │  Shards  │                    │
│ │ Servers │    │ ┌──────┐ │                    │
│ │(1 node) │    │ │Shard1│ │                    │
│ │         │    │ │(1node) │                    │
│ │         │    │ ├──────┤ │                    │
│ │         │    │ │Shard2│ │                    │
│ │         │    │ │(1node) │                    │
│ │         │    │ └──────┘ │                    │
│ └─────────┘    └──────────┘                    │
│                                                │
│  Total: 4 processus MongoDB                    │
│  ⚠️  AUCUNE haute disponibilité                │
│  ⚠️  Uniquement pour développement             │
└────────────────────────────────────────────────┘
```

**Ressources :**
- 1-2 serveurs (ou Docker Compose)
- 8-16 GB RAM total
- Cas d'usage : Développement local, tests fonctionnels

### Topologie Production Minimale

```
┌─────────────────────────────────────────────────────────┐
│       TOPOLOGIE PRODUCTION MINIMALE                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────┐  ┌──────────┐                             │
│  │ Mongos 1 │  │ Mongos 2 │  (2+ instances)             │
│  └─────┬────┘  └─────┬────┘                             │
│        │             │                                  │
│  ┌─────┴─────────────┴───────┐                          │
│  │                           │                          │
│  ▼                           ▼                          │
│ ┌──────────────────────────────────┐                    │
│ │    Config Server Replica Set     │                    │
│ │  ┌────┐   ┌────┐   ┌────┐        │                    │
│ │  │CS-1│   │CS-2│   │CS-3│        │                    │
│ │  │(P) │   │(S) │   │(S) │        │                    │
│ │  └────┘   └────┘   └────┘        │                    │
│ └──────────────────────────────────┘                    │
│                                                         │
│ ┌──────────────────────────────────┐                    │
│ │         Shard A (RS)             │                    │
│ │  ┌────┐   ┌────┐   ┌────┐        │                    │
│ │  │ P  │   │ S  │   │ S  │        │                    │
│ │  └────┘   └────┘   └────┘        │                    │
│ └──────────────────────────────────┘                    │
│                                                         │
│ ┌──────────────────────────────────┐                    │
│ │         Shard B (RS)             │                    │
│ │  ┌────┐   ┌────┐   ┌────┐        │                    │
│ │  │ P  │   │ S  │   │ S  │        │                    │
│ │  └────┘   └────┘   └────┘        │                    │
│ └──────────────────────────────────┘                    │
│                                                         │
│  Total: 11 processus (2 mongos + 3 config + 6 shard)    │
│  ✅ Haute disponibilité complète                        │
│  ✅ Tolérance aux pannes: 1 nœud par RS                 │
└─────────────────────────────────────────────────────────┘
```

**Ressources :**
- 11 serveurs physiques/VMs
- 16-32 GB RAM par serveur shard
- 8-16 GB RAM par config server
- 2-4 GB RAM par mongos

### Topologie Production Multi-Région

```
┌──────────────────────────────────────────────────────────┐
│              TOPOLOGIE MULTI-RÉGION                      │
├──────────────────────────────────────────────────────────┤
│                                                          │
│   RÉGION EU (Paris)              RÉGION US (Virginia)    │
│  ┌──────────────────┐           ┌──────────────────┐     │
│  │ ┌──────┐ ┌──────┐│           │ ┌──────┐ ┌──────┐│     │
│  │ │Mongos│ │Mongos││           │ │Mongos│ │Mongos││     │
│  │ │ EU-1 │ │ EU-2 ││           │ │ US-1 │ │ US-2 ││     │
│  │ └───┬──┘ └───┬──┘│           │ └───┬──┘ └───┬──┘│     │
│  └─────┼────────┼───┘           └─────┼────────┼───┘     │
│        │        │                     │        │         │
│  ┌─────┴────────┴─────────────────────┴────────┴────┐    │
│  │             Config Servers (CSRS)                │    │
│  │  ┌────────┐     ┌────────┐     ┌─────────┐       │    │
│  │  │ CS-EU  │     │ CS-US  │     │ CS-ASIA │       │    │
│  │  │  (P)   │─────┤  (S)   │─────┤  (S)    │       │    │
│  │  │ Paris  │     │Virginia│     │Singapore│       │    │
│  │  └────────┘     └────────┘     └─────────┘       │    │
│  └──────────────────────────────────────────────────┘    │
│                                                          │
│  ┌──────────────────┐           ┌──────────────────┐     │
│  │  Shard EU (RS)   │           │  Shard US (RS)   │     │
│  │  ┌────┐ ┌────┐   │           │  ┌────┐ ┌────┐   │     │
│  │  │ P  │ │ S  │   │           │  │ P  │ │ S  │   │     │
│  │  │ EU │ │ EU │   │           │  │ US │ │ US │   │     │
│  │  └────┘ └────┘   │           │  └────┘ └────┘   │     │
│  │  ┌────┐          │           │  ┌────┐          │     │
│  │  │ S  │          │           │  │ S  │          │     │
│  │  │ US │          │           │  │ EU │          │     │
│  │  └────┘          │           │  └────┘          │     │
│  └──────────────────┘           └──────────────────┘     │
│                                                          │
│  Zone Sharding:                                          │
│  - EU users → Shard EU (RGPD compliance)                 │
│  - US users → Shard US (latency optimization)            │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**Avantages :**
- ✅ Latence optimisée par région
- ✅ Conformité réglementaire (résidence des données)
- ✅ Tolérance aux pannes régionales
- ✅ Scalabilité géographique

**Considérations :**
- ⚠️ Latence inter-région pour metadata (config servers)
- ⚠️ Complexité de déploiement et monitoring
- ⚠️ Coûts de transfert de données entre régions

## Anti-Patterns Architecturaux

### 🚫 Anti-Pattern 1 : Mongos Unique

```
┌─────────────────────────────────┐
│     ❌ ANTI-PATTERN              │
│                                 │
│  ┌──────────────────┐           │
│  │ 1 seul Mongos    │           │
│  └────────┬─────────┘           │
│           │ SPOF!               │
│           ▼                     │
│    [Config + Shards]            │
└─────────────────────────────────┘

Problèmes:
- ❌ Single Point of Failure
- ❌ Bottleneck de performance
- ❌ Impossibilité de rolling update
```

**Solution :**
```
┌─────────────────────────────────┐
│     ✅ PATTERN                  │
│                                 │
│  ┌────────┐  ┌────────┐         │
│  │Mongos 1│  │Mongos 2│         │
│  └───┬────┘  └───┬────┘         │
│      │  Load     │              │
│      │  Balancer │              │
│      ▼           ▼              │
│   [Config + Shards]             │
└─────────────────────────────────┘

Minimum: 2 mongos
Recommandé: 3+ mongos
```

### 🚫 Anti-Pattern 2 : Config Servers Non-Répliqués

```
┌─────────────────────────────────┐
│     ❌ ANTI-PATTERN              │
│                                 │
│  ┌──────────────────┐           │
│  │ Config: 1 node   │           │
│  └──────────────────┘           │
│         SPOF!                   │
│                                 │
│  Si config down → Cluster       │
│  passe en READ-ONLY             │
└─────────────────────────────────┘
```

**Solution :**
```
┌─────────────────────────────────┐
│     ✅ PATTERN                  │
│                                 │
│  ┌────────────────────────────┐ │
│  │  Config Server RS (CSRS)   │ │
│  │  ┌────┐ ┌────┐ ┌────┐      │ │
│  │  │CS-1│ │CS-2│ │CS-3│      │ │
│  │  └────┘ └────┘ └────┘      │ │
│  └────────────────────────────┘ │
│                                 │
│  Minimum ABSOLU: 3 members      │
│  Haute dispo: 5 members         │
└─────────────────────────────────┘
```

### 🚫 Anti-Pattern 3 : Shards Sans Replica Sets

```
┌─────────────────────────────────┐
│     ❌ ANTI-PATTERN              │
│                                 │
│  Shard A: [1 mongod]            │
│  Shard B: [1 mongod]            │
│                                 │
│  Aucune HA au niveau shard      │
└─────────────────────────────────┘

Problèmes:
- ❌ Perte de données si shard down
- ❌ Pas de failover automatique
- ❌ Maintenance = downtime
```

**Solution :**
```
┌─────────────────────────────────┐
│     ✅ PATTERN                  │
│                                 │
│  Shard A RS: [P, S, S]          │
│  Shard B RS: [P, S, S]          │
│                                 │
│  Chaque shard = Replica Set 3+  │
└─────────────────────────────────┘
```

### 🚫 Anti-Pattern 4 : Config Servers et Shards Co-Localisés

```
┌─────────────────────────────────┐
│     ❌ ANTI-PATTERN              │
│                                 │
│  Serveur 1:                     │
│  ├─ Config Server (27019)       │
│  └─ Shard A Primary (27018)     │
│                                 │
│  Contention ressources +        │
│  Perte double si serveur down   │
└─────────────────────────────────┘
```

**Solution :**
```
┌─────────────────────────────────┐
│     ✅ PATTERN                  │
│                                 │
│  Serveurs dédiés:               │
│  - 3 serveurs Config            │
│  - N serveurs Shards            │
│  - Isolation complète           │
└─────────────────────────────────┘
```

### 🚫 Anti-Pattern 5 : Tous les Mongos sur Même Serveur

```
┌─────────────────────────────────┐
│     ❌ ANTI-PATTERN              │
│                                 │
│  Serveur Load Balancer:         │
│  ├─ Mongos 1 (27017)            │
│  ├─ Mongos 2 (27018)            │
│  └─ Mongos 3 (27019)            │
│                                 │
│  SPOF au niveau infrastructure  │
└─────────────────────────────────┘
```

**Solution :**
```
┌─────────────────────────────────┐
│     ✅ PATTERN                  │
│                                 │
│  Mongos distribués:             │
│  - Mongos 1: Serveur A          │
│  - Mongos 2: Serveur B          │
│  - Mongos 3: Serveur C          │
│                                 │
│  Ou: Co-localisés avec apps     │
└─────────────────────────────────┘
```

## Considérations de Production

### Réseau et Latence

```
Latences typiques acceptables:
┌────────────────────────────────────┐
│ Mongos ↔ App:         < 1 ms       │
│ Mongos ↔ Config:      < 5 ms       │
│ Mongos ↔ Shards:      < 10 ms      │
│ Config ↔ Config:      < 20 ms      │
│ Shard members ↔:      < 10 ms      │
└────────────────────────────────────┘

⚠️  Si latences > 50 ms entre composants:
   - Reconsidérer la topologie
   - Utiliser zones géographiques
   - Évaluer multi-région
```

### Bande Passante

```
Estimations de bande passante:
┌────────────────────────────────────────┐
│ Mongos → Shards:                       │
│   = Throughput applicatif              │
│   Exemple: 10k writes/sec × 1 KB       │
│           = 10 MB/sec                  │
│                                        │
│ Balancer (migrations):                 │
│   = Chunks/jour × Taille chunk         │
│   Exemple: 100 chunks × 128 MB         │
│           = 12.8 GB/jour               │
│           = ~1.5 MB/sec moyen          │
│                                        │
│ Replication (intra-shard):             │
│   = Oplog size                         │
│   Variable selon write load            │
└────────────────────────────────────────┘

Recommandation: 10 Gbps minimum en production
```

### Haute Disponibilité

```
Tolérance aux pannes par composant:
┌────────────────────────────────────────┐
│ Mongos (3 instances):                  │
│   ✅ Peut perdre 2 instances           │
│   (1 suffit pour routage)              │
│                                        │
│ Config Servers CSRS (3 membres):       │
│   ✅ Peut perdre 1 membre              │
│   (Majorité = 2/3)                     │
│                                        │
│ Shard RS (3 membres):                  │
│   ✅ Peut perdre 1 membre par shard    │
│   (Majorité = 2/3)                     │
│                                        │
│ Recommandation 5 membres:              │
│   ✅ Peut perdre 2 membres             │
│   (Majorité = 3/5)                     │
└────────────────────────────────────────┘
```

## Résumé

L'architecture shardée MongoDB est un système distribué sophistiqué composé de trois types de composants travaillant en harmonie :

**Mongos (Query Routers) :**
- Stateless, multiples instances
- Routage intelligent des requêtes
- Cache local des métadonnées
- Déploiement flexible (co-localisés ou dédiés)

**Config Servers (CSRS) :**
- Replica Set de 3+ membres
- Source de vérité pour métadonnées
- Critique pour haute disponibilité
- Dimensionnement basé sur nombre de chunks

**Shards (Data Replica Sets) :**
- Replica Sets indépendants
- Hébergement des chunks de données
- Détails dans section 10.2.1

**Principes architecturaux clés :**
1. ✅ Toujours déployer en Replica Sets (config + shards)
2. ✅ Multiples mongos pour HA et performance
3. ✅ Séparer physiquement les composants
4. ✅ Optimiser latence réseau entre composants
5. ✅ Planifier bande passante pour balancing

**Anti-patterns à éviter :**
- ❌ SPOF sur n'importe quel composant
- ❌ Co-localisation config servers + shards
- ❌ Sous-dimensionnement réseau
- ❌ Mongos unique
- ❌ Ignorer la latence inter-composants

La compréhension approfondie de cette architecture est essentielle avant de procéder au déploiement pratique d'un cluster shardé en production.

---


⏭️ [Shards](/10-sharding/02.1-shards.md)
