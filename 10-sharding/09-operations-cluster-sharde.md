🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.9 Opérations sur un Cluster Shardé

## Introduction

Les opérations quotidiennes sur un cluster shardé MongoDB diffèrent significativement de celles effectuées sur un simple Replica Set. La distribution des données à travers plusieurs shards introduit de la complexité mais aussi de nouvelles opportunités d'optimisation et de gestion. Cette section couvre l'ensemble des opérations courantes et avancées que vous effectuerez en production, depuis les opérations CRUD jusqu'aux maintenances planifiées, en passant par la gestion des index et le troubleshooting.

Comprendre le comportement de chaque opération dans un contexte distribué est essentiel pour maintenir un cluster performant, fiable et sécurisé.

---

## Architecture Opérationnelle : Vue d'Ensemble

### Flux des Opérations

```
┌──────────────────────────────────────────────────────────────┐
│                      APPLICATION                             │
│              (Drivers MongoDB)                               │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     │ Connection String
                     │
         ┌───────────┴──────────┬────────────────┐
         │                      │                │
    ┌────▼────┐           ┌─────▼───┐      ┌─────▼───┐
    │ mongos1 │           │ mongos2 │      │ mongos3 │
    │(Router) │           │(Router) │      │(Router) │
    └────┬────┘           └────┬────┘      └────┬────┘
         │                     │                │
         └─────────────────────┼────────────────┘
                               │
         ┌─────────────────────┴────────────────────┐
         │        Lecture Métadonnées               │
         │        (Config Servers)                  │
         └─────────────────────┬────────────────────┘
                               │
         ┌─────────────────────┴────────────────────┐
         │                                          │
    ┌────▼─────┐                               ┌────▼─────┐
    │ Shard A  │                               │ Shard B  │
    │  (RS)    │                               │  (RS)    │
    │  P S S   │                               │  P S S   │
    └──────────┘                               └──────────┘
         │                                           │
         │                                           │
    Données                                     Données
    Chunk 1-50                                  Chunk 51-100
```

### Points Clés Opérationnels

1. **Toutes les opérations passent par mongos** : Les clients ne se connectent jamais directement aux shards
2. **Mongos route intelligemment** : Utilise les métadonnées pour diriger vers les bons shards
3. **Les config servers sont critiques** : Sans eux, le cluster est en lecture seule
4. **Opérations targeted vs broadcast** : Impact majeur sur les performances

---

## Opérations CRUD dans un Cluster Shardé

### Insertions (Create)

#### Comportement Standard

```javascript
// Connexion via mongos
mongosh --host mongos1.example.com --port 27017

use mydb

// Insertion simple
db.users.insertOne({
  user_id: "user_12345",
  name: "Alice Dupont",
  email: "alice@example.com",
  created_at: new Date()
})

// Le mongos :
// 1. Détermine le chunk destination basé sur la shard key (user_id)
// 2. Route l'insertion vers le shard approprié
// 3. Retourne le résultat
```

**Processus interne** :

```javascript
// 1. Mongos consulte les métadonnées
// Config: Chunk [user_10000, user_20000) → shardA

// 2. Mongos envoie l'insertion au Primary de shardA
// write: { insert: "users", documents: [{ user_id: "user_12345", ... }] }

// 3. ShardA Primary réplique vers ses Secondaries
// 4. Mongos retourne le résultat avec write concern
```

#### Insertions en Masse (Bulk)

```javascript
// insertMany avec ordered: true (par défaut)
db.users.insertMany([
  { user_id: "user_10001", name: "User 1" },
  { user_id: "user_50001", name: "User 2" },  // Peut-être sur un autre shard
  { user_id: "user_10002", name: "User 3" }
], { ordered: true })

// Comportement :
// - Mongos route chaque document vers son shard
// - Si un shard différent : nouvelle connexion
// - Si ordered: true, arrêt à la première erreur
// - Si ordered: false, continue malgré les erreurs

// Résultat :
{
  "acknowledged": true,
  "insertedIds": {
    "0": ObjectId("..."),
    "1": ObjectId("..."),
    "2": ObjectId("...")
  }
}
```

**Optimisation pour insertions massives** :

```javascript
// Grouper par shard implicitement
// MongoDB 5.0+ optimise automatiquement

// Mais pour contrôle explicite :
var bulkA = db.users.initializeUnorderedBulkOp();
var bulkB = db.users.initializeUnorderedBulkOp();

// Documents pour shardA (user_id < 50000)
for (var i = 10000; i < 20000; i++) {
  bulkA.insert({ user_id: "user_" + i, name: "User " + i });
}

// Documents pour shardB (user_id >= 50000)
for (var i = 50000; i < 60000; i++) {
  bulkB.insert({ user_id: "user_" + i, name: "User " + i });
}

// Exécuter en parallèle (via promises dans un driver)
bulkA.execute();
bulkB.execute();
```

#### Write Concern dans un Cluster Shardé

```javascript
// Write concern s'applique au shard cible, pas au cluster entier
db.users.insertOne(
  { user_id: "user_12345", name: "Alice" },
  {
    writeConcern: {
      w: "majority",  // Majorité des membres du shard
      j: true,        // Journal committé
      wtimeout: 5000  // Timeout 5 secondes
    }
  }
)

// Si le document va sur shardA (replica set de 3 membres) :
// w: "majority" = 2 membres (Primary + 1 Secondary)

// ⚠️ Attention : w: "majority" ne garantit PAS
// la réplication sur tous les shards, uniquement sur le shard cible
```

### Lectures (Read)

#### Requêtes Ciblées (Targeted Queries)

```javascript
// Requête incluant la shard key
db.users.find({ user_id: "user_12345" })

// Mongos :
// 1. Identifie le chunk contenant user_12345
// 2. Route vers un seul shard (shardA)
// 3. Retourne les résultats

// explain() montre que c'est targeted
db.users.find({ user_id: "user_12345" }).explain("executionStats")

// Résultat :
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "SINGLE_SHARD",  // ✅ Requête ciblée !
      "shards": [
        {
          "shardName": "shardA",
          // ...
        }
      ]
    }
  }
}
```

#### Requêtes Broadcast (Scatter-Gather)

```javascript
// Requête sans shard key
db.users.find({ email: "alice@example.com" })

// Mongos :
// 1. Ne peut pas déterminer le shard → broadcast
// 2. Envoie la requête à TOUS les shards
// 3. Collecte les résultats (gather)
// 4. Merge et retourne

// explain() montre le broadcast
db.users.find({ email: "alice@example.com" }).explain("executionStats")

// Résultat :
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "SHARD_MERGE",  // ⚠️ Requête broadcast
      "shards": [
        { "shardName": "shardA", /* ... */ },
        { "shardName": "shardB", /* ... */ }
      ]
    }
  },
  "executionStats": {
    "totalDocsExamined": 10000000,  // Sur tous les shards !
    "executionTimeMillis": 2500
  }
}
```

**Impact performance** :

```javascript
// Comparaison : Targeted vs Broadcast (cluster 5 shards)

// Targeted (avec shard key)
db.orders.find({ customer_id: "CUST12345" })
// - 1 shard interrogé
// - Latence : ~10ms
// - Documents examinés : 1000

// Broadcast (sans shard key)
db.orders.find({ order_status: "pending" })
// - 5 shards interrogés
// - Latence : ~50ms (5x)
// - Documents examinés : 5000000 (5000x)
```

#### Read Preference dans un Cluster Shardé

```javascript
// Read preference s'applique aux replica sets des shards

// Lecture sur Primary uniquement (défaut)
db.users.find({ user_id: "user_12345" })
  .readPref("primary")

// Lecture sur Secondary préférée (pour reporting)
db.users.find({ user_id: "user_12345" })
  .readPref("secondaryPreferred")

// Lecture sur Secondary le plus proche
db.users.find({ user_id: "user_12345" })
  .readPref("nearest")

// ⚠️ Avec broadcast, chaque shard utilise la read preference
// Peut charger les secondaries si configured
```

**Cas d'usage read preferences** :

```javascript
// 1. Analytics/Reporting : Décharger les primaries
db.analytics.aggregate([
  { $match: { date: { $gte: ISODate("2024-01-01") } } },
  { $group: { _id: "$product", totalSales: { $sum: "$amount" } } }
])
.readPref("secondary")
.allowDiskUse(true)

// 2. Géolocalisation : Minimiser latence
// Application en Europe, secondary en Europe
db.users.find({ user_id: "user_EU_12345" })
  .readPref("nearest", [{ "datacenter": "europe" }])
```

### Mises à Jour (Update)

#### Update Ciblé (avec shard key)

```javascript
// Update incluant la shard key
db.users.updateOne(
  { user_id: "user_12345" },  // Shard key présente
  { $set: { last_login: new Date() } }
)

// Mongos :
// 1. Route vers le shard contenant user_12345
// 2. Update exécuté localement sur ce shard
// 3. Résultat retourné

// Résultat :
{
  "acknowledged": true,
  "matchedCount": 1,
  "modifiedCount": 1
}
```

#### Update Broadcast (sans shard key)

```javascript
// Update sans shard key (ou updateMany)
db.users.updateMany(
  { status: "inactive" },  // Pas de shard key
  { $set: { notification_sent: true } }
)

// Mongos :
// 1. Envoie l'update à TOUS les shards
// 2. Chaque shard exécute localement
// 3. Agrège les résultats

// Résultat :
{
  "acknowledged": true,
  "matchedCount": 1500,  // Somme de tous les shards
  "modifiedCount": 1500
}
```

#### ⚠️ Restriction : Modification de la Shard Key

```javascript
// MongoDB < 4.2 : Interdit
db.users.updateOne(
  { user_id: "user_12345" },
  { $set: { user_id: "user_99999" } }  // ❌ Erreur ImmutableField
)

// MongoDB >= 4.2 : Autorisé mais coûteux
db.users.updateOne(
  { user_id: "user_12345" },
  { $set: { user_id: "user_99999" } }  // ✅ OK mais migration interne
)

// Processus interne (MongoDB >= 4.2) :
// 1. Vérifier que user_99999 est sur un autre chunk
// 2. Supprimer le document sur shardA
// 3. Insérer le document sur shardB
// 4. Opération transactionnelle (atomique)
```

### Suppressions (Delete)

#### Delete Ciblé

```javascript
// Delete avec shard key
db.users.deleteOne({ user_id: "user_12345" })

// Mongos route vers le bon shard
// Suppression locale
```

#### Delete Broadcast

```javascript
// Delete sans shard key
db.users.deleteMany({ last_login: { $lt: ISODate("2022-01-01") } })

// Broadcast à tous les shards
// Chaque shard supprime localement
// Résultats agrégés

// Résultat :
{
  "acknowledged": true,
  "deletedCount": 5000  // Somme de tous les shards
}
```

#### Truncate d'une Collection Shardée

```javascript
// drop() sur une collection shardée
db.users.drop()

// Processus :
// 1. Mongos envoie dropCollection à tous les shards
// 2. Chaque shard supprime sa partie de la collection
// 3. Métadonnées nettoyées sur config servers
// 4. Confirmation retournée

// Plus rapide que deleteMany({})
```

---

## Agrégations dans un Cluster Shardé

### Pipeline d'Agrégation : Exécution Distribuée

```javascript
// Agrégation complexe
db.orders.aggregate([
  { $match: { order_date: { $gte: ISODate("2024-01-01") } } },
  { $group: {
      _id: "$customer_id",
      totalAmount: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalAmount: -1 } },
  { $limit: 10 }
])

// Exécution optimisée par MongoDB :
//
// Phase 1 : SHARD (parallèle sur chaque shard)
// ┌─────────────────────────────────────┐
// │ ShardA : $match → $group (partial)  │
// │ ShardB : $match → $group (partial)  │
// │ ShardC : $match → $group (partial)  │
// └─────────────────────────────────────┘
//                  │
//                  ▼
// Phase 2 : MERGE (sur mongos ou shard primaire)
// ┌─────────────────────────────────────┐
// │ Mongos : $group (final merge)       │
// │          $sort                       │
// │          $limit                      │
// └─────────────────────────────────────┘
```

### Étapes qui Forcent le Merge

Certaines étapes nécessitent un merge centralisé :

```javascript
// 1. $lookup (jointures)
db.orders.aggregate([
  { $match: { customer_id: "CUST12345" } },  // Sur shard
  { $lookup: {  // ⚠️ Force merge sur mongos
      from: "customers",
      localField: "customer_id",
      foreignField: "_id",
      as: "customer_info"
    }
  }
])

// 2. $sort global sans limite
db.orders.aggregate([
  { $match: { order_date: { $gte: ISODate("2024-01-01") } } },  // Sur shard
  { $sort: { amount: -1 } }  // ⚠️ Merge nécessaire (sauf si $limit suit)
])

// 3. $facet
db.orders.aggregate([
  { $match: { status: "completed" } },  // Sur shard
  { $facet: {  // ⚠️ Force merge
      byCategory: [{ $group: { _id: "$category", count: { $sum: 1 } } }],
      byMonth: [{ $group: { _id: { $month: "$date" }, total: { $sum: "$amount" } } }]
    }
  }
])
```

### Optimisation : allowDiskUse

```javascript
// Pour agrégations volumineuses
db.orders.aggregate([
  { $match: { order_date: { $gte: ISODate("2023-01-01") } } },
  { $group: {
      _id: "$product_id",
      stats: { $push: "$$ROOT" }  // Peut dépasser 100 MB
    }
  }
], { allowDiskUse: true })

// allowDiskUse : true
// - Permet l'usage du disque temporaire si RAM insuffisante
// - S'applique sur chaque shard individuellement
// - Performance réduite mais évite les erreurs de mémoire
```

### Agrégations avec Shard Key : $merge et $out

```javascript
// $out : Créer une nouvelle collection shardée
db.daily_sales.aggregate([
  { $match: { date: ISODate("2024-01-15") } },
  { $group: { _id: "$product_id", totalSales: { $sum: "$amount" } } },
  { $out: "sales_summary" }  // Nouvelle collection
])

// Par défaut, la nouvelle collection est NON shardée
// Pour créer une collection shardée en sortie :

// 1. Pré-créer et sharder la collection de sortie
sh.shardCollection("mydb.sales_summary", { product_id: 1 })

// 2. Utiliser $merge au lieu de $out
db.daily_sales.aggregate([
  { $match: { date: ISODate("2024-01-15") } },
  { $group: { _id: "$product_id", totalSales: { $sum: "$amount" } } },
  { $merge: {
      into: "sales_summary",
      on: "_id",  // product_id (doit être la shard key ou partie de)
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
])
```

---

## Gestion des Index

### Création d'Index sur Collection Shardée

```javascript
// Créer un index via mongos
db.users.createIndex({ email: 1 })

// Processus :
// 1. Mongos envoie createIndex à TOUS les shards
// 2. Chaque shard construit l'index localement
// 3. Construction en parallèle sur tous les shards
// 4. Mongos attend que tous aient terminé
// 5. Retourne le succès global

// Durée = durée du shard le plus lent
```

#### Création d'Index en Arrière-Plan

```javascript
// Index background (moins de blocage)
db.users.createIndex(
  { last_login: 1 },
  { background: true }  // Déprecié depuis MongoDB 4.2
)

// MongoDB 4.2+ : Toujours en arrière-plan par défaut
// Mais pour foreground (plus rapide) :
db.users.createIndex(
  { last_login: 1 },
  { background: false }
)
```

#### Index avec Options Spécifiques

```javascript
// Index unique sur collection shardée
// ⚠️ L'unicité est garantie PAR SHARD, pas globalement
// SAUF si l'index inclut la shard key

// ❌ Pas d'unicité globale (sans shard key)
db.users.createIndex(
  { email: 1 },
  { unique: true }  // ⚠️ Unique par shard uniquement
)

// ✅ Unicité globale (inclut shard key)
db.users.createIndex(
  { user_id: 1, email: 1 },  // user_id est la shard key
  { unique: true }  // ✅ Unique globalement
)

// Ou shard key elle-même
db.users.createIndex(
  { user_id: 1 },
  { unique: true }  // ✅ Garantie par design du sharding
)
```

### Listing et Monitoring des Index

```javascript
// Lister les index via mongos
db.users.getIndexes()

// Retourne les index du premier shard trouvé
// ⚠️ Assume que tous les shards ont les mêmes index

// Vérifier les index sur chaque shard individuellement
var shards = db.getSiblingDB("config").shards.find().toArray();

shards.forEach(function(shard) {
  print("Indexes sur " + shard._id + " :");

  // Connexion directe au shard (pour inspection)
  var shardConn = new Mongo(shard.host.split("/")[1].split(",")[0]);
  var indexes = shardConn.getDB("mydb").users.getIndexes();

  printjson(indexes);
});
```

### Suppression d'Index

```javascript
// Supprimer un index via mongos
db.users.dropIndex("email_1")

// Processus :
// 1. Mongos envoie dropIndex à tous les shards
// 2. Chaque shard supprime l'index localement
// 3. Suppression en parallèle
// 4. Confirmation globale

// ⚠️ Attention : Suppression immédiate
// Vérifier d'abord que l'index n'est pas utilisé
db.users.aggregate([
  { $indexStats: {} },
  { $match: { name: "email_1" } }
])
```

---

## Opérations d'Administration

### Ajout d'un Shard

```javascript
// Contexte : Ajouter un nouveau shard au cluster pour scaling

// Étape 1 : Préparer le nouveau replica set (shardD)
// - Déployer les membres du replica set
// - Initialiser le replica set
// - Vérifier qu'il fonctionne correctement

// Étape 2 : Ajouter le shard via mongos
sh.addShard("shardD/shardD1.example.com:27018,shardD2.example.com:27018,shardD3.example.com:27018")

// Résultat :
{
  "shardAdded": "shardD",
  "ok": 1
}

// Étape 3 : Vérifier l'ajout
sh.status()

// Étape 4 : Le balancer redistribue automatiquement
// Monitoring de la redistribution
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.commit",
  "details.to": "shardD"
}).sort({ time: -1 })
```

### Retrait d'un Shard (Drain)

```javascript
// Contexte : Retirer shardA du cluster (décommission hardware)

// Étape 1 : Initier le drain
db.adminCommand({ removeShard: "shardA" })

// Résultat initial :
{
  "msg": "draining started successfully",
  "state": "started",
  "shard": "shardA",
  "remaining": {
    "chunks": 45,  // Chunks à migrer
    "dbs": 2       // Bases avec shardA comme primary shard
  }
}

// Étape 2 : Monitoring du drain
// Les chunks sont migrés automatiquement vers d'autres shards
db.adminCommand({ removeShard: "shardA" })

// Résultat intermédiaire :
{
  "msg": "draining ongoing",
  "state": "ongoing",
  "remaining": {
    "chunks": 12,  // Progression
    "dbs": 2
  },
  "note": "you need to drop or movePrimary these databases"
}

// Étape 3 : Gérer les bases dont shardA est primary
use config
db.databases.find({ primary: "shardA" })

// Déplacer le primary de chaque base
db.adminCommand({ movePrimary: "mydb", to: "shardB" })
db.adminCommand({ movePrimary: "analytics", to: "shardC" })

// Étape 4 : Finaliser le drain
db.adminCommand({ removeShard: "shardA" })

// Résultat final :
{
  "msg": "removeshard completed successfully",
  "state": "completed",
  "shard": "shardA"
}

// Étape 5 : Arrêter physiquement les serveurs de shardA
// Le replica set shardA peut maintenant être décommissionné
```

### Changement de Shard Primaire d'une Base

```javascript
// Voir le shard primaire actuel
use config
db.databases.findOne({ _id: "mydb" })

// Résultat :
{
  "_id": "mydb",
  "primary": "shardA",  // Shard primaire actuel
  "partitioned": true
}

// Changer le shard primaire
db.adminCommand({ movePrimary: "mydb", to: "shardB" })

// ⚠️ ATTENTION : Opération coûteuse
// - Déplace toutes les collections NON-shardées
// - Peut prendre des heures pour grandes bases
// - Impact performance pendant la migration

// Processus :
// 1. Copie des collections non-shardées de shardA → shardB
// 2. Mise à jour des métadonnées
// 3. Suppression des données sur shardA

// Validation
use config
db.databases.findOne({ _id: "mydb" })

// Résultat après :
{
  "_id": "mydb",
  "primary": "shardB",  // ✅ Changé
  "partitioned": true
}
```

---

## Maintenance et Mises à Jour

### Rolling Upgrade d'un Cluster Shardé

Ordre recommandé pour minimiser l'impact :

```bash
# Ordre de mise à jour (du moins critique au plus critique)
# 1. Mongos (routers)
# 2. Config Servers
# 3. Shards (un à un)

# ========================================
# Phase 1 : Mise à jour des Mongos
# ========================================

# Pour chaque mongos (mongos1, mongos2, mongos3) :

# 1a. Retirer du load balancer (si applicable)
# → Diriger le trafic vers les autres mongos

# 1b. Arrêter le mongos
sudo systemctl stop mongos

# 1c. Mettre à jour les binaires
sudo apt-get update && sudo apt-get install mongodb-org=7.0.0

# 1d. Redémarrer
sudo systemctl start mongos

# 1e. Vérifier
mongosh --host mongos1.example.com --eval "db.version()"

# 1f. Réintégrer au load balancer

# Répéter pour mongos2, mongos3...

# ========================================
# Phase 2 : Mise à jour des Config Servers
# ========================================

# Pour chaque membre du replica set config (cfg1, cfg2, cfg3) :

# 2a. Sur les SECONDARIES d'abord
# Connexion au secondary
mongosh --host cfg2.example.com:27019

# Vérifier le statut
rs.status()

# Arrêter le mongod
use admin
db.shutdownServer()

# Mettre à jour et redémarrer
sudo apt-get install mongodb-org=7.0.0
sudo systemctl start mongod-configsvr

# Vérifier la réintégration au replica set
rs.status()

# Répéter pour cfg3

# 2b. PRIMARY en dernier
# Connexion au primary
mongosh --host cfg1.example.com:27019

# Forcer une élection (step down)
rs.stepDown(60)

# Attendre qu'un secondary devienne primary
sleep 10

# Puis même procédure : shutdown, upgrade, restart

# ========================================
# Phase 3 : Mise à jour des Shards
# ========================================

# Pour chaque shard (shardA, shardB, etc.) :
# Même procédure que pour config servers

# ⚠️ Mettre à jour UN SHARD À LA FOIS
# Ne pas mettre à jour plusieurs shards simultanément

# Pour shardA :
# 3a. Secondaries en premier (shardA2, shardA3)
# 3b. Primary en dernier (shardA1)

# Validation globale après chaque shard
sh.status()
```

### Sauvegarde d'un Cluster Shardé

#### Méthode 1 : mongodump via Mongos

```bash
# Sauvegarde complète du cluster via mongos
mongodump \
  --host mongos1.example.com \
  --port 27017 \
  --username admin \
  --password SecurePass123 \
  --authenticationDatabase admin \
  --out /backup/cluster-$(date +%Y%m%d) \
  --oplog

# Avantages :
# - Simple : une seule commande
# - Vue cohérente du cluster

# Inconvénients :
# - Lent pour gros volumes (tout passe par mongos)
# - Impact performance pendant la sauvegarde
```

#### Méthode 2 : Sauvegarde par Shard + Config Servers

```bash
# Sauvegarde optimisée : chaque shard en parallèle

# 1. Arrêter le balancer
mongosh --host mongos1.example.com --eval "sh.stopBalancer()"

# 2. Sauvegarder les config servers
mongodump \
  --host cfg1.example.com:27019 \
  --out /backup/configdb-$(date +%Y%m%d) \
  --oplog

# 3. Sauvegarder chaque shard en parallèle
# ShardA
mongodump \
  --host shardA1.example.com:27018 \
  --out /backup/shardA-$(date +%Y%m%d) \
  --oplog &

# ShardB
mongodump \
  --host shardB1.example.com:27018 \
  --out /backup/shardB-$(date +%Y%m%d) \
  --oplog &

# Attendre la fin de toutes les sauvegardes
wait

# 4. Réactiver le balancer
mongosh --host mongos1.example.com --eval "sh.startBalancer()"

# Avantages :
# - Plus rapide (parallèle)
# - Moins d'impact sur mongos

# Inconvénients :
# - Plus complexe à orchestrer
# - Restauration plus complexe
```

#### Méthode 3 : Snapshots Système

```bash
# Utiliser les snapshots du système de stockage (LVM, EBS, etc.)

# 1. Arrêter le balancer
sh.stopBalancer()

# 2. Pour chaque shard :
#    a. Flush des écritures
mongosh --host shardA1.example.com:27018
use admin
db.fsyncLock()  # Verrouiller les écritures

#    b. Créer snapshot (exemple AWS EBS)
aws ec2 create-snapshot \
  --volume-id vol-shardA-data \
  --description "ShardA backup $(date +%Y%m%d)"

#    c. Déverrouiller
db.fsyncUnlock()

# 3. Répéter pour tous les shards et config servers

# 4. Réactiver le balancer
sh.startBalancer()
```

### Restauration d'un Cluster Shardé

```bash
# Scénario : Restauration complète après disaster

# Étape 1 : Restaurer les config servers
mongorestore \
  --host cfg1.example.com:27019 \
  --drop \
  /backup/configdb-20240115 \
  --oplogReplay

# Répéter pour cfg2, cfg3

# Étape 2 : Restaurer chaque shard
mongorestore \
  --host shardA1.example.com:27018 \
  --drop \
  /backup/shardA-20240115 \
  --oplogReplay

mongorestore \
  --host shardB1.example.com:27018 \
  --drop \
  /backup/shardB-20240115 \
  --oplogReplay

# Étape 3 : Redémarrer les mongos
sudo systemctl restart mongos

# Étape 4 : Vérifier l'intégrité
mongosh --host mongos1.example.com
sh.status()

# Vérifier les collections
db.users.countDocuments({})
```

---

## Monitoring Opérationnel

### Métriques Critiques à Surveiller

```javascript
// 1. Distribution des chunks
db.getSiblingDB("config").chunks.aggregate([
  { $group: {
      _id: { ns: "$ns", shard: "$shard" },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id.ns": 1, "_id.shard": 1 } }
])

// 2. État du balancer
sh.getBalancerState()
sh.isBalancerRunning()

// 3. Migrations récentes
db.getSiblingDB("config").changelog.find({
  what: { $in: ["moveChunk.start", "moveChunk.commit", "moveChunk.error"] },
  time: { $gte: new Date(Date.now() - 3600000) }
}).count()

// 4. Jumbo chunks
db.getSiblingDB("config").chunks.find({ jumbo: true }).count()

// 5. Opérations lentes
db.currentOp({
  "secs_running": { $gte: 5 },
  "op": { $in: ["query", "update", "remove"] }
})

// 6. Connexions actives
db.serverStatus().connections
```

### Dashboard de Monitoring Temps Réel

```javascript
// monitoring-dashboard.js
// Script à exécuter en continu

function displayDashboard() {
  // Clear screen (pour terminal)
  print("\x1Bc");

  print("=".repeat(80));
  print("CLUSTER SHARDÉ - DASHBOARD OPÉRATIONNEL");
  print("Heure : " + new Date().toLocaleString());
  print("=".repeat(80));
  print("");

  // Section 1 : Santé du Cluster
  print("1. SANTÉ DU CLUSTER");
  print("-".repeat(40));

  var shards = db.getSiblingDB("config").shards.find().toArray();
  print("  Nombre de shards : " + shards.length);

  shards.forEach(function(shard) {
    // Ping chaque shard
    try {
      var shardConn = new Mongo(shard.host.split("/")[1].split(",")[0]);
      shardConn.getDB("admin").ping();
      print("  ✅ " + shard._id + " : Online");
    } catch (e) {
      print("  ❌ " + shard._id + " : OFFLINE !");
    }
  });
  print("");

  // Section 2 : Balancer
  print("2. BALANCER");
  print("-".repeat(40));
  print("  État : " + (sh.getBalancerState() ? "Activé" : "Désactivé"));
  print("  En cours : " + (sh.isBalancerRunning() ? "Oui" : "Non"));

  var activeMigrations = db.currentOp({
    op: "command",
    "command.moveChunk": { $exists: true }
  }).inprog.length;

  print("  Migrations actives : " + activeMigrations);
  print("");

  // Section 3 : Distribution
  print("3. DISTRIBUTION DES DONNÉES");
  print("-".repeat(40));

  var distribution = db.getSiblingDB("config").chunks.aggregate([
    { $group: { _id: "$shard", numChunks: { $sum: 1 } } },
    { $sort: { numChunks: -1 } }
  ]).toArray();

  var total = distribution.reduce((sum, d) => sum + d.numChunks, 0);

  distribution.forEach(function(dist) {
    var percent = ((dist.numChunks / total) * 100).toFixed(1);
    print("  " + dist._id + " : " + dist.numChunks + " chunks (" + percent + "%)");
  });
  print("");

  // Section 4 : Performance
  print("4. PERFORMANCE");
  print("-".repeat(40));

  var slowOps = db.currentOp({
    secs_running: { $gte: 5 }
  }).inprog.length;

  print("  Opérations lentes (>5s) : " + slowOps);

  var serverStatus = db.serverStatus();
  print("  Connexions actives : " + serverStatus.connections.current);
  print("  Opcounters (insert) : " + serverStatus.opcounters.insert);
  print("  Opcounters (query) : " + serverStatus.opcounters.query);
  print("");

  // Section 5 : Alertes
  print("5. ALERTES");
  print("-".repeat(40));

  var alerts = [];

  // Alerte : Jumbo chunks
  var jumboCount = db.getSiblingDB("config").chunks.find({ jumbo: true }).count();
  if (jumboCount > 0) {
    alerts.push("⚠️  " + jumboCount + " jumbo chunks détectés");
  }

  // Alerte : Déséquilibre
  if (distribution.length > 1) {
    var max = Math.max(...distribution.map(d => d.numChunks));
    var min = Math.min(...distribution.map(d => d.numChunks));
    var imbalance = ((max - min) / min) * 100;

    if (imbalance > 20) {
      alerts.push("⚠️  Déséquilibre significatif : " + imbalance.toFixed(1) + "%");
    }
  }

  // Alerte : Opérations lentes
  if (slowOps > 10) {
    alerts.push("⚠️  Trop d'opérations lentes : " + slowOps);
  }

  if (alerts.length === 0) {
    print("  ✅ Aucune alerte");
  } else {
    alerts.forEach(function(alert) {
      print("  " + alert);
    });
  }

  print("");
  print("=".repeat(80));
}

// Boucle de rafraîchissement
while (true) {
  displayDashboard();
  sleep(5000);  // Rafraîchir toutes les 5 secondes
}
```

---

## Troubleshooting Opérationnel

### Problème 1 : Performances Dégradées

**Diagnostic** :

```javascript
// 1. Identifier les requêtes lentes
db.currentOp({
  "secs_running": { $gte: 3 },
  "op": { $in: ["query", "update"] }
})

// 2. Analyser une requête lente
var slowOp = db.currentOp({ secs_running: { $gte: 3 } }).inprog[0];

print("Namespace : " + slowOp.ns);
print("Opération : " + slowOp.op);
print("Durée : " + slowOp.secs_running + "s");
printjson(slowOp.command);

// 3. Vérifier si c'est une requête broadcast
if (slowOp.command && !slowOp.command.$db) {
  // Analyser avec explain
  var explainResult = db[slowOp.ns.split(".")[1]].explain("executionStats").find(slowOp.command.filter || {});

  if (explainResult.queryPlanner.winningPlan.stage === "SHARD_MERGE") {
    print("⚠️  Requête BROADCAST détectée - considérer ajout shard key au filtre");
  }
}

// 4. Vérifier les index manquants
db[slowOp.ns.split(".")[1]].find(slowOp.command.filter || {}).explain("executionStats")
```

**Solutions** :

```javascript
// Solution 1 : Ajouter un index
db.users.createIndex({ email: 1, last_login: 1 })

// Solution 2 : Optimiser la requête pour inclure shard key
// Avant (broadcast) :
db.orders.find({ order_status: "pending" })

// Après (targeted si possible) :
db.orders.find({
  customer_id: "CUST12345",  // Shard key
  order_status: "pending"
})

// Solution 3 : Utiliser read preference pour décharger primaries
db.analytics.find({ date: ISODate("2024-01-15") })
  .readPref("secondaryPreferred")
```

### Problème 2 : Cluster en Lecture Seule

**Symptômes** :

```javascript
// Tentative d'écriture échoue
db.users.insertOne({ user_id: "user_new", name: "Test" })

// Erreur :
MongoServerError: not master and slaveOk=false
// Ou
MongoServerError: can't connect to config server
```

**Diagnostic** :

```javascript
// 1. Vérifier les config servers
sh.status()

// Si erreur : "can't connect to config server"
// Les config servers sont inaccessibles

// 2. Vérifier chaque config server
var configServers = ["cfg1.example.com:27019", "cfg2.example.com:27019", "cfg3.example.com:27019"];

configServers.forEach(function(cs) {
  try {
    var conn = new Mongo(cs);
    var status = conn.getDB("admin").adminCommand({ replSetGetStatus: 1 });
    print(cs + " : " + status.myState + " (" + status.stateStr + ")");
  } catch (e) {
    print(cs + " : ❌ INJOIGNABLE - " + e.message);
  }
});
```

**Solutions** :

```bash
# Si majorité des config servers down :
# - Le cluster passe en lecture seule (sécurité)

# Solution : Restaurer les config servers
# 1. Redémarrer les config servers arrêtés
sudo systemctl start mongod-configsvr

# 2. Vérifier le replica set
mongosh --host cfg1.example.com:27019
rs.status()

# 3. Si problème d'élection, forcer une reconfig
cfg = rs.conf()
rs.reconfig(cfg, { force: true })

# 4. Vérifier depuis mongos
mongosh --host mongos1.example.com
sh.status()  # Devrait fonctionner à nouveau
```

### Problème 3 : Migration Bloquée

**Diagnostic** :

```javascript
// Migration bloquée depuis longtemps
db.currentOp({
  op: "command",
  "command.moveChunk": { $exists: true },
  microsecs_running: { $gt: 1800000000 }  // > 30 minutes
})

// Vérifier les logs de migration
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.error",
  time: { $gte: new Date(Date.now() - 3600000) }
}).sort({ time: -1 })
```

**Solutions** :

```javascript
// Solution 1 : Annuler la migration
var blockedMigration = db.currentOp({
  "command.moveChunk": { $exists: true }
}).inprog[0];

if (blockedMigration) {
  db.killOp(blockedMigration.opid);
  print("Migration annulée : " + blockedMigration.opid);
}

// Solution 2 : Identifier la cause
// - Réseau saturé ?
// - Chunk trop volumineux ?
// - Charge trop élevée ?

// Vérifier la taille du chunk
var chunk = db.getSiblingDB("config").chunks.findOne({
  ns: blockedMigration.ns,
  min: blockedMigration.command.find
});

print("Taille estimée du chunk : " + chunk.estimatedSize);

// Si jumbo chunk, le splitter avant re-tentative
if (chunk.jumbo) {
  sh.splitAt(blockedMigration.ns, chunk.max);
}

// Solution 3 : Réessayer pendant une fenêtre de faible charge
sh.stopBalancer();
// Attendre une heure creuse
sh.startBalancer();
```

---

## Anti-Patterns Opérationnels

### ❌ Anti-Pattern 1 : Connexion Directe aux Shards

**Problème** :

```javascript
// ❌ Se connecter directement à un shard
mongosh --host shardA1.example.com:27018

// Effectuer des opérations CRUD
db.users.find({ user_id: "user_12345" })
```

**Conséquence** :
- Vue partielle des données (un seul shard)
- Métadonnées non mises à jour
- Risque de corruption du cluster

**Solution** :

```javascript
// ✅ TOUJOURS passer par mongos
mongosh --host mongos1.example.com:27017

// Pour inspection/debugging uniquement, connexion directe acceptable
// Mais JAMAIS pour opérations CRUD en production
```

### ❌ Anti-Pattern 2 : Write Concern Insuffisant

**Problème** :

```javascript
// ❌ Write concern par défaut (w: 1)
db.critical_data.insertOne({
  transaction_id: "TXN123",
  amount: 50000
})

// Si le primary du shard crash juste après :
// - Donnée pas encore répliquée
// - Perte de données possible
```

**Conséquence** :
- Risque de perte de données critiques
- Non-conformité aux exigences métier

**Solution** :

```javascript
// ✅ Write concern "majority" pour données critiques
db.critical_data.insertOne(
  {
    transaction_id: "TXN123",
    amount: 50000
  },
  {
    writeConcern: {
      w: "majority",
      j: true,
      wtimeout: 5000
    }
  }
)

// Ou configurer au niveau de la collection
db.runCommand({
  collMod: "critical_data",
  writeConcern: { w: "majority", j: true }
})
```

### ❌ Anti-Pattern 3 : Ignorer les Requêtes Broadcast

**Problème** :

```javascript
// ❌ Requêtes systématiques sans shard key
db.orders.find({ status: "pending" })  // Broadcast !
db.users.find({ email: "alice@example.com" })  // Broadcast !
```

**Conséquence** :
- Latence multipliée par le nombre de shards
- Charge CPU/réseau élevée
- Scalabilité limitée

**Solution** :

```javascript
// ✅ Option 1 : Remodeler pour inclure shard key
db.orders.find({
  customer_id: "CUST12345",  // Shard key
  status: "pending"
})

// ✅ Option 2 : Index secondaire + accepter broadcast
db.users.createIndex({ email: 1 })
db.users.find({ email: "alice@example.com" })  // Broadcast mais indexé

// ✅ Option 3 : Refactorer la shard key si nécessaire
sh.refineCollectionShardKey("mydb.users", { email: 1, _id: 1 })
```

### ❌ Anti-Pattern 4 : Pas de Monitoring du Balancer

**Problème** :

```bash
# Balancer actif 24/7 sans surveillance
# Migrations pendant les pics de charge
# Personne ne monitore les échecs
```

**Conséquence** :
- Impact performance imprévisible
- Déséquilibre s'aggrave sans détection
- Jumbo chunks non traités

**Solution** :

```javascript
// ✅ Fenêtre de balancing
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "02:00",
        stop: "06:00"
      }
    }
  },
  { upsert: true }
)

// ✅ Alerting automatisé
// Script cron quotidien
function checkBalancerHealth() {
  var errors = db.getSiblingDB("config").changelog.count({
    what: "moveChunk.error",
    time: { $gte: new Date(Date.now() - 86400000) }
  });

  if (errors > 5) {
    // Déclencher alerte PagerDuty/Slack
    print("ALERT: " + errors + " migration errors in last 24h");
  }
}
```

### ❌ Anti-Pattern 5 : Modifications Non Testées

**Problème** :

```bash
# Modifier la configuration du cluster en production
# Sans test préalable en staging
# Exemple : Changer la taille des chunks, ajouter un index unique, etc.
```

**Conséquence** :
- Comportements inattendus
- Downtime imprévu
- Rollback complexe

**Solution** :

```bash
# ✅ Pipeline de test rigoureux
# 1. Tester en environnement de dev
# 2. Tester en staging avec données réalistes
# 3. Documenter les résultats
# 4. Planifier le déploiement en production
# 5. Avoir un plan de rollback

# Exemple : Ajout d'index unique
# Dev → OK
# Staging → OK, durée: 15 min
# Prod → Planifier fenêtre maintenance, informer équipes
```

---

## Checklist Opérationnelle Quotidienne

```yaml
daily_operations_checklist:

  morning_checks:
    - title: "Vérifier la santé du cluster"
      command: "sh.status()"
      expected: "Tous les shards et config servers online"

    - title: "État du balancer"
      command: "sh.getBalancerState() && sh.isBalancerRunning()"
      expected: "Activé mais pas de migration en cours (hors fenêtre)"

    - title: "Jumbo chunks"
      command: "db.getSiblingDB('config').chunks.find({ jumbo: true }).count()"
      expected: "0 jumbo chunks"

    - title: "Distribution des chunks"
      action: "Vérifier écart entre shards < 10%"

    - title: "Échecs de migration (24h)"
      command: "db.getSiblingDB('config').changelog.count({ what: 'moveChunk.error', time: { $gte: ... } })"
      expected: "< 5 échecs"

  afternoon_checks:
    - title: "Performance des requêtes"
      command: "db.currentOp({ secs_running: { $gte: 5 } })"
      expected: "< 10 opérations lentes"

    - title: "Utilisation disque"
      action: "Vérifier < 70% sur tous les shards"

    - title: "Connexions actives"
      command: "db.serverStatus().connections"
      expected: "< 80% du max"

  evening_checks:
    - title: "Sauvegarde quotidienne"
      action: "Vérifier que la sauvegarde s'est bien exécutée"

    - title: "Logs d'erreurs"
      action: "Analyser les logs pour erreurs critiques"

  weekly_checks:
    - title: "Index inutilisés"
      command: "db.collection.aggregate([{ $indexStats: {} }])"
      action: "Supprimer les index non utilisés"

    - title: "Croissance des données"
      action: "Projeter la croissance et planifier scaling si nécessaire"

    - title: "Test de restauration"
      action: "Tester la restauration d'une sauvegarde en staging"
```

---

## Conclusion

Les opérations sur un cluster shardé MongoDB nécessitent une compréhension approfondie de l'architecture distribuée et de ses implications. Les points clés à retenir :

- ✅ **Toujours passer par mongos** : Jamais de connexion directe aux shards
- ✅ **Comprendre targeted vs broadcast** : Impact majeur sur les performances
- ✅ **Monitorer activement** : Balancer, migrations, distribution, jumbo chunks
- ✅ **Write concern adapté** : Selon la criticité des données
- ✅ **Planifier les maintenances** : Rolling upgrades, fenêtres de balancing
- ✅ **Tester avant production** : Toute modification doit être testée
- ✅ **Automatiser le monitoring** : Alertes proactives sur anomalies

Les opérations quotidiennes doivent être **routinières** et **automatisées** autant que possible, permettant aux équipes de se concentrer sur l'optimisation et l'évolution du cluster plutôt que sur la maintenance réactive.

---

## Ressources

- [MongoDB Documentation - Sharded Cluster Administration](https://docs.mongodb.com/manual/administration/sharded-cluster-administration/)
- [MongoDB Documentation - Backup and Restore Sharded Clusters](https://docs.mongodb.com/manual/tutorial/backup-sharded-cluster-with-database-dumps/)
- [MongoDB Operations Checklist](https://docs.mongodb.com/manual/administration/production-checklist-operations/)
- [MongoDB University - M103: Basic Cluster Administration](https://university.mongodb.com/)

---


⏭️ [Monitoring et maintenance](/10-sharding/10-monitoring-maintenance.md)
