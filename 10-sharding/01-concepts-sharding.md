🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.1 Concepts du Sharding

## Introduction

Le **sharding** (partitionnement horizontal) est la méthode de distribution des données de MongoDB permettant de répartir un ensemble de données volumineux sur plusieurs machines. Contrairement à la réplication qui duplique les données pour la haute disponibilité, le sharding divise les données pour augmenter la capacité de stockage et le débit de traitement.

Cette section explore les concepts fondamentaux qui sous-tendent l'architecture shardée de MongoDB, nécessaires pour comprendre, déployer et gérer efficacement un cluster en production.

## Terminologie Essentielle

### Shard

Un **shard** est un sous-ensemble des données d'une collection shardée. Dans MongoDB, chaque shard est typiquement implémenté comme un **Replica Set** complet, offrant ainsi :
- Haute disponibilité au niveau du shard
- Tolérance aux pannes
- Possibilité de read preference par shard

```
┌─────────────────────────────────────┐
│         Shard A (Replica Set)       │
│  ┌─────────┐  ┌─────────┐  ┌─────┐  │
│  │ Primary │  │Secondary│  │Sec. │  │
│  └─────────┘  └─────────┘  └─────┘  │
│  Contient: user_id 0-999            │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Shard B (Replica Set)       │
│  ┌─────────┐  ┌─────────┐  ┌─────┐  │
│  │ Primary │  │Secondary│  │Sec. │  │
│  └─────────┘  └─────────┘  └─────┘  │
│  Contient: user_id 1000-1999        │
└─────────────────────────────────────┘
```

**Caractéristiques :**
- Chaque shard ne contient qu'une portion des données totales
- Les shards ne communiquent pas directement entre eux
- L'ajout de shards augmente la capacité globale du cluster

### Chunk

Un **chunk** est une unité logique de données contiguës selon la shard key. MongoDB divise automatiquement les données d'une collection shardée en chunks.

**Propriétés d'un chunk :**
- Taille par défaut : **128 MB** (configurable entre 1 MB et 1024 MB)
- Bornes : `[minKey, maxKey)` basées sur la shard key
- Indivisible lors du balancing (sauf splitting)
- Peut résider sur un seul shard à la fois

```javascript
// Exemple de structure d'un chunk dans config.chunks
{
  "_id": ObjectId("..."),
  "ns": "ecommerce.orders",
  "min": { "customer_id": 0 },
  "max": { "customer_id": 1000 },
  "shard": "shard-a",
  "lastmod": Timestamp(1, 0),
  "history": [...]
}
```

**Représentation visuelle :**
```
Collection: ecommerce.orders (shard key: customer_id)

┌──────────────────────────────────────────────────────┐
│                    Shard A                           │
├──────────────────┬──────────────────┬────────────────┤
│  Chunk 1         │  Chunk 2         │  Chunk 3       │
│  customer_id:    │  customer_id:    │  customer_id:  │
│  [0, 1000)       │  [1000, 2000)    │  [2000, 3000)  │
│  Size: 120 MB    │  Size: 115 MB    │  Size: 128 MB  │
└──────────────────┴──────────────────┴────────────────┘

┌──────────────────────────────────────────────────────┐
│                    Shard B                           │
├──────────────────┬──────────────────┬────────────────┤
│  Chunk 4         │  Chunk 5         │  Chunk 6       │
│  customer_id:    │  customer_id:    │  customer_id:  │
│  [3000, 4000)    │  [4000, 5000)    │  [5000, 6000)  │
│  Size: 125 MB    │  Size: 127 MB    │  Size: 118 MB  │
└──────────────────┴──────────────────┴────────────────┘
```

### Shard Key

La **shard key** est le champ indexé (ou la combinaison de champs) utilisé pour déterminer la distribution des documents entre les shards.

**Rôle critique :**
- Détermine le chunk qui contient chaque document
- Immuable après la création (avant MongoDB 5.0)
- Impact direct sur les performances et la distribution

```javascript
// Exemples de shard keys
sh.shardCollection("db.collection", { "_id": 1 })                    // Simple
sh.shardCollection("db.collection", { "user_id": 1, "date": 1 })     // Composée
sh.shardCollection("db.collection", { "location": "hashed" })        // Hashed
```

**Anatomie d'une shard key composée :**
```javascript
// Shard key: { "region": 1, "user_id": 1 }

Document 1: { region: "EU", user_id: 12345, name: "Alice" }
            ↓
            Shard Key Value: { region: "EU", user_id: 12345 }
            ↓
            Chunk: [{ region: "EU", user_id: 10000 }, { region: "EU", user_id: 20000 })
            ↓
            Shard: shard-eu-01
```

### Mongos (Query Router)

**Mongos** est le processus de routage qui dirige les requêtes des clients vers les shards appropriés.

**Fonctions principales :**
1. **Routage intelligent** : Analyse les requêtes et détermine les shards cibles
2. **Agrégation des résultats** : Combine les résultats de plusieurs shards
3. **Gestion des transactions** : Coordonne les transactions multi-shards
4. **Cache des métadonnées** : Maintient une copie locale de la configuration

```
┌─────────────────────────────────────────────────────┐
│                   Application                       │
└──────────────────────┬──────────────────────────────┘
                       │ Connection String
                       ▼
              ┌──────────────────┐
              │  Mongos Instance │
              └────────┬─────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
Query Analysis    Metadata Cache   Result Merge
        │              │              │
        ▼              ▼              ▼
   Shard A        Config Servers  Shard B
```

**Caractéristiques importantes :**
- Sans état (stateless) : peut être arrêté/redémarré sans perte de données
- Haute disponibilité : déployer plusieurs instances mongos
- Léger : faible empreinte mémoire (~256 MB typique)
- Scalable : ajouter des mongos pour gérer plus de connexions

### Config Servers

Les **config servers** stockent les métadonnées et la configuration du cluster shardé.

**Données stockées :**
- Mapping chunks → shards
- Shard key des collections
- Zones et tags
- Historique des migrations
- Configuration du balancer

```javascript
// Exemple de métadonnées dans config database
use config

// Collections critiques :
db.databases.find()       // Informations sur les databases shardées
db.collections.find()     // Collections shardées et leurs shard keys
db.chunks.find()          // Tous les chunks et leur localisation
db.shards.find()          // Liste des shards du cluster
db.version.find()         // Version du cluster metadata
```

**Architecture depuis MongoDB 3.4+ :**
- Implémentés comme un **Replica Set CSRS** (Config Server Replica Set)
- Minimum requis : **3 membres**
- Utilise le protocole de réplication standard
- Haute disponibilité critique : si config servers down → cluster en lecture seule

```
┌────────────────────────────────────────────────┐
│     Config Server Replica Set (CSRS)           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Primary  │  │Secondary │  │Secondary │      │
│  │  (CSR1)  │  │  (CSR2)  │  │  (CSR3)  │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│                                                │
│  Database: config                              │
│  Collections: chunks, databases, collections,  │
│               shards, version, locks...        │
└────────────────────────────────────────────────┘
```

## Mécanismes Fondamentaux

### 1. Chunk Splitting (Division des Chunks)

Lorsqu'un chunk atteint la taille configurée (par défaut 128 MB), MongoDB le divise automatiquement en deux chunks.

**Processus de splitting :**

```
État Initial:
┌────────────────────────────────┐
│         Chunk A                │
│  Range: [0, 10000)             │
│  Size: 140 MB (> 128 MB)       │
│  Shard: shard-a                │
└────────────────────────────────┘

Après Splitting:
┌──────────────────┬──────────────────┐
│    Chunk A-1     │    Chunk A-2     │
│ Range: [0, 5000) │Range: [5000,10000│
│ Size: 70 MB      │ Size: 70 MB      │
│ Shard: shard-a   │ Shard: shard-a   │
└──────────────────┴──────────────────┘
```

**Code interne (simplifié) :**
```javascript
// MongoDB détermine le point de split (médiane de la shard key)
splitPoint = findMedian(chunk.minKey, chunk.maxKey)

// Crée deux nouveaux chunks
chunk1 = { min: chunk.minKey, max: splitPoint, shard: chunk.shard }
chunk2 = { min: splitPoint, max: chunk.maxKey, shard: chunk.shard }

// Les deux chunks restent sur le même shard initialement
```

**Points importants :**
- Le split est **local** au shard (pas de transfert de données)
- Le split point est calculé par la **médiane** de la shard key
- Opération rapide (~millisecondes)
- Peut être déclenché manuellement : `sh.splitAt()` ou `sh.splitFind()`

**Anti-pattern - Jumbo Chunks :**
```javascript
// ❌ MAUVAIS : Shard key avec peu de valeurs distinctes
sh.shardCollection("app.events", { "event_type": 1 })
// event_type a seulement 5 valeurs : "login", "logout", "purchase", "view", "click"

// Résultat : Chunks de 500 GB car impossible de diviser plus finement
// → Jumbo chunk marqué, balancing bloqué
```

### 2. Balancing (Équilibrage)

Le **balancer** est un processus automatique qui migre les chunks entre shards pour maintenir une distribution équilibrée.

**Déclencheurs du balancing :**
- Différence de nombre de chunks entre shards > seuil de migration
- Seuils configurables selon le nombre total de chunks

```
Seuils de migration par défaut :
< 20 chunks:     différence de 2 chunks
20-79 chunks:    différence de 4 chunks
≥ 80 chunks:     différence de 8 chunks
```

**Algorithme de balancing (simplifié) :**

```
Étape 1: Identifier les shards déséquilibrés
┌─────────┬─────────┬─────────┐
│ Shard A │ Shard B │ Shard C │
│ 45 chks │ 38 chks │ 25 chks │ → Différence: 20 chunks (A vs C)
└─────────┴─────────┴─────────┘

Étape 2: Sélectionner un chunk à migrer de A vers C
Chunk sélectionné: chunk_17 (critères: taille, activité récente)

Étape 3: Migration
┌─────────────────────────────────────────┐
│  1. Clone chunk_17 de Shard A → Shard C │
│  2. Sync incrementale (oplog)           │
│  3. Update metadata (config servers)    │
│  4. Delete chunk_17 from Shard A        │
└─────────────────────────────────────────┘

Résultat:
┌─────────┬─────────┬─────────┐
│ Shard A │ Shard B │ Shard C │
│ 44 chks │ 38 chks │ 26 chks │ → Plus équilibré
└─────────┴─────────┴─────────┘
```

**Fenêtre de balancing :**
```javascript
// Configurer une fenêtre pour limiter l'impact
use config
db.settings.updateOne(
  { _id: "balancer" },
  { $set: {
      activeWindow: {
        start: "01:00",  // 1h du matin
        stop: "05:00"    // 5h du matin
      }
    }
  },
  { upsert: true }
)
```

**Contrôle du balancer :**
```javascript
// Arrêter temporairement
sh.stopBalancer()

// Vérifier le statut
sh.getBalancerState()
// true = actif, false = inactif

// Redémarrer
sh.startBalancer()

// Vérifier si le balancer est en cours d'exécution
sh.isBalancerRunning()
```

### 3. Routage des Requêtes

Le routage détermine quels shards doivent être interrogés pour une requête donnée.

#### Types de Routage

**A. Targeted Query (Requête Ciblée)**

La requête contient la shard key → mongos interroge uniquement le(s) shard(s) pertinent(s).

```javascript
// Shard key: { "user_id": 1 }
db.orders.find({ "user_id": 12345, "status": "pending" })

// Mongos détermine:
// 1. Chunk contenant user_id=12345 → chunk_42
// 2. Shard hébergeant chunk_42 → shard-b
// 3. Route la requête uniquement vers shard-b

┌──────────┐
│  Mongos  │
└────┬─────┘
     │ Query: user_id=12345
     ▼
┌─────────┐     ┌─────────┐     ┌─────────┐
│Shard A  │     │Shard B  │     │Shard C  │
│         │     │  ✓✓✓✓   │     │         │
│         │     │ Queried │     │         │
└─────────┘     └─────────┘     └─────────┘
```

**Performance : O(1) shard** - Optimal

**B. Scatter-Gather Query**

La requête ne contient pas (ou partiellement) la shard key → mongos interroge tous les shards.

```javascript
// Shard key: { "user_id": 1 }
db.orders.find({ "product_id": "ABC123" })  // product_id n'est PAS dans la shard key

// Mongos doit interroger TOUS les shards:
┌──────────┐
│  Mongos  │
└─┬──┬──┬──┘
  │  │  │ Query: product_id=ABC123
  ▼  ▼  ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Shard A  │  │Shard B  │  │Shard C  │
│  ✓✓✓✓   │  │  ✓✓✓✓   │  │  ✓✓✓✓   │
│ Queried │  │ Queried │  │ Queried │
└─────────┘  └─────────┘  └─────────┘
     │            │            │
     └────────────┼────────────┘
                  ▼
         ┌────────────────┐
         │ Merge Results  │
         │   in Mongos    │
         └────────────────┘
```

**Performance : O(N) shards** - À éviter si possible

**C. Broadcast Query**

Requêtes d'administration ou sans filtre → tous les shards.

```javascript
db.orders.find({})  // Pas de filtre
db.orders.count()   // Count global
```

#### Optimisation du Routage avec Shard Key Composée

```javascript
// Shard key composée: { "region": 1, "user_id": 1 }

// ✅ Requête totalement ciblée (1 shard)
db.users.find({ "region": "EU", "user_id": 12345 })

// ⚠️ Requête partiellement ciblée (quelques shards)
db.users.find({ "region": "EU" })
// Mongos interroge uniquement les chunks avec region="EU"
// Mais peut être réparti sur plusieurs shards

// ❌ Requête scatter-gather (tous les shards)
db.users.find({ "user_id": 12345 })
// user_id est dans la shard key MAIS region manque (préfixe)
```

**Règle du préfixe de shard key :**
Pour une shard key composée `{ a: 1, b: 1, c: 1 }`, le routage ciblé nécessite :
- `{ a: ... }` → Partiellement ciblé
- `{ a: ..., b: ... }` → Plus ciblé
- `{ a: ..., b: ..., c: ... }` → Totalement ciblé
- `{ b: ... }` ou `{ c: ... }` → Scatter-gather

### 4. Metadata Management

Les métadonnées du cluster sont stockées dans la base **config** sur les config servers.

**Collections critiques :**

```javascript
// 1. config.shards - Liste des shards
{
  "_id": "shard-a",
  "host": "shard-a/mongo1:27018,mongo2:27018,mongo3:27018",
  "state": 1,  // 1 = active
  "tags": ["eu", "premium"]
}

// 2. config.databases - Databases et primary shard
{
  "_id": "ecommerce",
  "primary": "shard-a",  // Primary shard pour collections non-shardées
  "partitioned": true    // Database est shardée
}

// 3. config.collections - Collections shardées
{
  "_id": "ecommerce.orders",
  "key": { "customer_id": 1, "order_date": 1 },  // Shard key
  "unique": false,
  "lastmodEpoch": ObjectId("..."),
  "dropped": false
}

// 4. config.chunks - Mapping chunks → shards
{
  "_id": ObjectId("..."),
  "ns": "ecommerce.orders",
  "min": { "customer_id": 0, "order_date": ISODate("2024-01-01") },
  "max": { "customer_id": 1000, "order_date": ISODate("2024-02-01") },
  "shard": "shard-a",
  "lastmod": Timestamp(5, 1),
  "history": [
    { "validAfter": Timestamp(...), "shard": "shard-a" }
  ]
}

// 5. config.locks - Verrous distribués
{
  "_id": "ecommerce.orders",
  "state": 2,  // 2 = locked
  "process": "ConfigServer",
  "ts": ObjectId("..."),
  "when": ISODate("2024-12-08T10:30:00Z"),
  "who": "shard-a:Balancer:123456",
  "why": "Migration chunk_42"
}
```

**Synchronisation des métadonnées :**

```
┌──────────────────────────────────────────────────┐
│  Config Servers (Source de vérité)               │
│  - Chunks mapping                                │
│  - Shard key definitions                         │
│  - Database/collection metadata                  │
└──────────────┬───────────────────────────────────┘
               │ Replication
               ▼
     ┌─────────────────────┐
     │  Mongos Instances   │
     │  (Cache des metadata│
     │   - TTL: 30s        │
     │   - Lazy refresh)   │
     └─────────────────────┘
```

**Impact du cache mongos :**
- Requêtes plus rapides (pas de round-trip vers config servers)
- Risque de stale metadata (30 secondes max)
- Rafraîchissement automatique en cas d'erreur de routage

## Flux de Données dans un Cluster Shardé

### Insertion de Document

```
┌─────────────────────────────────────────────────────────────┐
│  1. Application envoie insert                               │
│     db.orders.insertOne({                                   │
│       customer_id: 12345,                                   │
│       order_date: ISODate("2024-12-08"),                    │
│       total: 99.99                                          │
│     })                                                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
         ┌───────────────────────────────────────────────┐
         │  2. Mongos reçoit                             │
         │     - Extrait shard key:                      │
         │       { customer_id: 12345, order_date: ... } │
         │     - Consulte metadata:                      │
         │       Chunk range → Shard B                   │
         └──────────────┬────────────────────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │  3. Route vers   │
              │     Shard B      │
              │     (Primary)    │
              └────────┬─────────┘
                       │
                       ▼
            ┌────────────────────┐
            │ 4. Shard B insère  │
            │    - Write Concern │
            │    - Oplog entry   │
            │    - Replication   │
            └─────────┬──────────┘
                      │
                      ▼
            ┌──────────────────┐
            │ 5. Acknowledge   │
            │    à Mongos      │
            └────────┬─────────┘
                     │
                     ▼
          ┌────────────────────┐
          │ 6. Mongos retourne │
          │    à l'application │
          └────────────────────┘
```

### Requête Find avec Shard Key

```
┌────────────────────────────────────────────────────┐
│  1. Application envoie find                        │
│     db.orders.find({ customer_id: 12345 })         │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────┐
         │  2. Mongos analyse      │
         │     - Shard key présent │
         │     - Lookup metadata   │
         │     - Détermine: Shard B│
         └──────────┬──────────────┘
                    │
                    ▼
          ┌──────────────────┐
          │ 3. Query Shard B │
          │    (ciblée)      │
          └────────┬─────────┘
                   │
                   ▼
         ┌──────────────────┐
         │ 4. Shard B       │
         │    exécute query │
         │    retourne docs │
         └─────────┬────────┘
                   │
                   ▼
         ┌──────────────────┐
         │ 5. Mongos        │
         │    retourne docs │
         │    (pas de merge)│
         └─────────┬────────┘
                   │
                   ▼
         ┌────────────────────┐
         │ 6. Application     │
         │    reçoit résultat │
         └────────────────────┘
```

### Requête Find SANS Shard Key (Scatter-Gather)

```
┌────────────────────────────────────────────────────┐
│  1. Application envoie find                        │
│     db.orders.find({ product_id: "ABC123" })       │
└──────────────────────┬─────────────────────────────┘
                       │
                       ▼
         ┌─────────────────────────┐
         │  2. Mongos analyse      │
         │     - Shard key ABSENT  │
         │     - Décision: Scatter │
         └──────┬──────────────┬───┘
                │              │
       ┌────────┘              └────────┐
       │                                │
       ▼                                ▼
┌────────────┐   ┌────────────┐   ┌────────────┐
│3a. Query   │   │3b. Query   │   │3c. Query   │
│  Shard A   │   │  Shard B   │   │  Shard C   │
└──────┬─────┘   └──────┬─────┘   └──────┬─────┘
       │                │                │
       │   4. Results   │    Results     │
       └────────┬───────┴────────┬───────┘
                │                │
                ▼                ▼
         ┌──────────────────────────┐
         │ 5. Mongos MERGE          │
         │    - Combine results     │
         │    - Apply sort/limit    │
         │    - Deduplicate (si _id)│
         └──────────┬───────────────┘
                    │
                    ▼
         ┌────────────────────┐
         │ 6. Application     │
         │    reçoit résultat │
         └────────────────────┘
```

**Impact performance Scatter-Gather :**
- Latence = max(latence_shardA, latence_shardB, latence_shardC)
- Bande passante × N shards
- Charge CPU mongos pour le merge
- À éviter autant que possible

## Concepts Avancés

### Zones (Shard Tags)

Les **zones** permettent de contrôler la distribution des chunks sur des shards spécifiques selon des règles métier.

**Cas d'usage typiques :**
1. **Localisation géographique** : Données EU sur shards EU, US sur shards US
2. **Tiers de service** : Clients premium sur hardware premium
3. **Conformité** : Données sensibles sur shards dédiés

```javascript
// Configuration de zones géographiques
sh.addShardToZone("shard-eu-01", "EU")
sh.addShardToZone("shard-eu-02", "EU")
sh.addShardToZone("shard-us-01", "US")
sh.addShardToZone("shard-us-02", "US")

// Définir les plages de shard key pour chaque zone
sh.updateZoneKeyRange(
  "app.users",
  { country: "FR", user_id: MinKey },
  { country: "FR", user_id: MaxKey },
  "EU"
)

sh.updateZoneKeyRange(
  "app.users",
  { country: "DE", user_id: MinKey },
  { country: "DE", user_id: MaxKey },
  "EU"
)

sh.updateZoneKeyRange(
  "app.users",
  { country: "US", user_id: MinKey },
  { country: "US", user_id: MaxKey },
  "US"
)
```

**Visualisation :**
```
┌──────────────────────────────────────────────────────┐
│  Zone: EU                                            │
│  ┌────────────┐  ┌────────────┐                      │
│  │ Shard EU-1 │  │ Shard EU-2 │                      │
│  └────────────┘  └────────────┘                      │
│  Chunks:                                             │
│  - country="FR", user_id [0, 10000)                  │
│  - country="FR", user_id [10000, 20000)              │
│  - country="DE", user_id [0, 15000)                  │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│  Zone: US                                            │
│  ┌────────────┐  ┌────────────┐                      │
│  │ Shard US-1 │  │ Shard US-2 │                      │
│  └────────────┘  └────────────┘                      │
│  Chunks:                                             │
│  - country="US", user_id [0, 20000)                  │
│  - country="US", user_id [20000, 40000)              │
└──────────────────────────────────────────────────────┘
```

### Jumbo Chunks

Un **jumbo chunk** est un chunk qui dépasse la taille maximale configurée mais ne peut pas être divisé.

**Causes :**
1. **Faible cardinalité** de la shard key
2. **Distribution déséquilibrée** des valeurs
3. **Shard key avec trop de documents identiques**

```javascript
// Exemple causant jumbo chunks
sh.shardCollection("logs.events", { "severity": 1 })
// severity a 3 valeurs: "INFO", "WARNING", "ERROR"
// Si 90% des logs sont "INFO" → Chunk "INFO" devient jumbo

// Diagnostic
db.chunks.find({ ns: "logs.events", jumbo: true })

// Output:
{
  "_id": ObjectId("..."),
  "ns": "logs.events",
  "min": { "severity": "INFO" },
  "max": { "severity": "WARNING" },
  "shard": "shard-a",
  "jumbo": true,  // ⚠️ Marqué jumbo
  "lastmod": Timestamp(10, 0)
}
```

**Impact :**
- ❌ Le balancer ne peut pas migrer le chunk
- ❌ Distribution déséquilibrée permanente
- ❌ Hotspot sur le shard contenant le jumbo chunk

**Résolution (détaillée en section 10.11) :**
1. Raffiner la shard key (si possible depuis MongoDB 5.0+)
2. Split manuel avec valeur intermédiaire
3. Remodélisation et migration

### Hashed Shard Keys

Le **hashed sharding** applique une fonction de hachage sur la shard key pour garantir une distribution uniforme.

```javascript
// Activation du hashed sharding
sh.shardCollection("app.users", { "_id": "hashed" })

// MongoDB calcule automatiquement:
hash(user_id) → int64 distribué uniformément

// Exemple de distribution:
user_id: "user_12345" → hash → -4567891234567890123 → Chunk A → Shard 1
user_id: "user_67890" → hash →  8901234567890123456 → Chunk B → Shard 2
```

**Distribution garantie :**
```
Sans hashing (monotone):
Shard A: ████████████████ (100% insertions récentes)
Shard B: ░░░░░░░░░░░░░░░░ (idle)
Shard C: ░░░░░░░░░░░░░░░░ (idle)

Avec hashing:
Shard A: █████░░░░░░░░░░░ (33% insertions)
Shard B: ░░░░░█████░░░░░░ (33% insertions)
Shard C: ░░░░░░░░░░█████░ (34% insertions)
```

**Trade-off :**
- ✅ Distribution parfaite
- ✅ Pas de hotspots
- ❌ Impossibilité de requêtes par plage ciblées
- ❌ Toutes les requêtes par plage → scatter-gather

## Anti-Patterns Fondamentaux

### 1. Shard Key Non Incluse dans les Requêtes

```javascript
// ❌ ANTI-PATTERN
sh.shardCollection("products.catalog", { "category_id": 1 })

// Mais 90% des requêtes sont:
db.catalog.find({ "sku": "ABC123" })
db.catalog.find({ "price": { $lt: 50 } })

// Résultat: Scatter-gather systématique sur tous les shards
```

**Solution :**
```javascript
// ✅ PATTERN
// Analyser les patterns de requête AVANT de choisir la shard key
sh.shardCollection("products.catalog", { "sku": 1 })
```

### 2. Shard Key Unique Monotone

```javascript
// ❌ ANTI-PATTERN
sh.shardCollection("events.logs", { "timestamp": 1 })

// Problème: Toutes les insertions vont au chunk le plus récent
// Un seul shard actif en écriture → hotspot permanent
```

**Solution :**
```javascript
// ✅ PATTERN 1: Hashed
sh.shardCollection("events.logs", { "timestamp": "hashed" })

// ✅ PATTERN 2: Composé avec préfixe non-monotone
sh.shardCollection("events.logs", { "source_id": 1, "timestamp": 1 })
```

### 3. Cardinalité Insuffisante

```javascript
// ❌ ANTI-PATTERN
sh.shardCollection("orders.transactions", { "status": 1 })
// status: "pending", "completed", "cancelled" (3 valeurs)

// Résultat:
// - Maximum 3 chunks possibles
// - Impossible de distribuer sur plus de 3 shards
// - Jumbo chunks garantis à forte volumétrie
```

**Solution :**
```javascript
// ✅ PATTERN
sh.shardCollection("orders.transactions",
  { "status": 1, "customer_id": 1, "order_id": 1 }
)
// Cardinalité: 3 × millions × millions = suffisante
```

### 4. Shard Key Mutable

```javascript
// ❌ ANTI-PATTERN (avant MongoDB 5.0)
sh.shardCollection("users.profiles", { "email": 1 })

// Problème: Si un utilisateur change d'email
db.profiles.updateOne(
  { _id: userId },
  { $set: { email: "new_email@example.com" } }
)

// Avant MongoDB 5.0: Erreur - Impossible de changer shard key
// Depuis MongoDB 5.0: Possible mais overhead (migration de chunk)
```

**Solution :**
```javascript
// ✅ PATTERN
sh.shardCollection("users.profiles", { "_id": 1 })
// _id est toujours immuable
```

### 5. Sur-Sharding Prématuré

```javascript
// ❌ ANTI-PATTERN
// Déployer 20 shards pour une database de 100 GB

// Overhead inutile:
// - 20 Replica Sets à gérer (60+ serveurs)
// - Balancer actif en permanence
// - Metadata overhead
// - Latence de routage accrue
```

**Solution :**
```javascript
// ✅ PATTERN
// Commencer modeste, scaler progressivement
// Règle empirique:
// - Démarrer avec 2-3 shards
// - Ajouter un shard lorsque:
//   * Stockage par shard > 2-3 TB
//   * Débit par shard > 100k ops/sec
//   * Latence > SLA défini
```

## Considérations de Performance

### Latence de Routage

```
Requête non-shardée (Replica Set):
Application → Primary → Résultat
Latence typique: 1-5 ms

Requête shardée ciblée:
Application → Mongos → Shard Primary → Résultat
Latence typique: 2-8 ms (+overhead mongos ~1-3 ms)

Requête scatter-gather (4 shards):
Application → Mongos → [4 Shards en parallèle] → Merge → Résultat
Latence typique: 10-50 ms (max des 4 + merge)
```

### Throughput d'Écriture

```
Replica Set:
Max throughput ≈ Capacité du Primary
Exemple: ~50,000 writes/sec sur hardware standard

Cluster Shardé (4 shards):
Max throughput ≈ 4 × Capacité d'un Primary
Exemple: ~200,000 writes/sec (scaling quasi-linéaire)

Condition: Distribution uniforme des écritures
```

### Consommation Mémoire

```
Mongos:
- Cache metadata: 50-200 MB
- Connexion pools: 100 MB par 1000 connexions
- Total typique: 256-512 MB par instance

Config Servers:
- Metadata database: 1-5 GB (pour 1M chunks)
- WiredTiger cache: 1-2 GB minimum
- Total: 2-4 GB par config server

Shards:
- Working set + Indexes (comme un Replica Set standard)
- WiredTiger cache: 50% de la RAM par défaut
```

## Checklist de Conception

Avant de déployer un cluster shardé, validez :

- [ ] **Volumétrie** : Données actuelles et projection 2-5 ans
- [ ] **Patterns de requête** : 80/20 rule - quelles sont les 20% de requêtes critiques ?
- [ ] **Shard key** : Satisfait CWT (Cardinalité, Write distribution, Targetability)
- [ ] **Indexes** : Shard key indexée + indexes secondaires planifiés
- [ ] **Read/Write ratio** : Majoritairement lecture, écriture, ou mixte ?
- [ ] **Latence acceptable** : SLA défini pour P50, P95, P99
- [ ] **Backup strategy** : Compatible avec sharding (backup par shard ou global)
- [ ] **Monitoring** : Métriques spécifiques sharding définies
- [ ] **Expertise** : Équipe formée à l'administration d'un cluster shardé
- [ ] **Rollback plan** : Stratégie de retour arrière si problème

## Résumé

Le sharding MongoDB repose sur des concepts fondamentaux interconnectés :

1. **Chunks** : Unités logiques de données distribuées
2. **Shard Key** : Détermine la distribution (choix le plus critique)
3. **Balancer** : Maintient l'équilibre automatiquement
4. **Mongos** : Route intelligemment les requêtes
5. **Config Servers** : Source de vérité pour les métadonnées

**Principes de conception :**
- La shard key est **immuable** (ou difficile à changer) → réflexion approfondie nécessaire
- Optimiser pour les **requêtes ciblées** (éviter scatter-gather)
- Démarrer **petit** et scaler progressivement
- **Monitorer** constamment la distribution et les performances

**Anti-patterns critiques à éviter :**
- ❌ Shard key à faible cardinalité
- ❌ Shard key monotone sans distribution
- ❌ Shard key absente des requêtes principales
- ❌ Shard key mutable
- ❌ Sur-sharding prématuré

Le sharding est un outil puissant mais complexe. Une conception soigneuse basée sur ces concepts fondamentaux est essentielle pour un cluster performant et maintenable en production.

---


⏭️ [Architecture shardée](/10-sharding/02-architecture-shardee.md)
