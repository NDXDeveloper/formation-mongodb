🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.8 Migration des Chunks

## Introduction

La migration des chunks est le mécanisme fondamental qui permet au cluster shardé MongoDB de maintenir un équilibre de charge entre les shards. Contrairement à d'autres systèmes de bases de données distribuées qui nécessitent des arrêts planifiés pour redistribuer les données, MongoDB effectue ces migrations **en arrière-plan**, de manière **transparente** pour les applications, tout en continuant à servir les requêtes de lecture et d'écriture.

Cette section explore en profondeur le processus de migration, ses phases techniques, les optimisations possibles, et les situations problématiques qui peuvent survenir en production.

---

## Concepts Fondamentaux

### Qu'est-ce qu'une Migration de Chunk ?

Une **migration de chunk** est le processus de transfert d'un chunk (et de tous ses documents) d'un shard source vers un shard destination. Ce processus :

- Est **initié par le balancer** (automatique) ou **manuellement** (opération d'administration)
- S'exécute de manière **asynchrone** en arrière-plan
- Maintient la **disponibilité** des données pendant la migration
- Garantit la **cohérence** des données après la migration
- Peut être **annulé** en cas de problème

### Acteurs de la Migration

```
┌─────────────────────────────────────────────────────────┐
│                    BALANCER                             │
│                (Config Server Primary)                  │
│  - Détecte le déséquilibre                              │
│  - Planifie les migrations                              │
│  - Coordonne l'exécution                                │
└───────────────┬─────────────────────────────────────────┘
                │
                │ Commande moveChunk
                │
                ▼
┌───────────────────────────────┐    Transfert      ┌──────────────────────────────┐
│     SHARD SOURCE              │    de données     │    SHARD DESTINATION         │
│   (Possède le chunk)          │ ───────────────►  │  (Reçoit le chunk)           │
│                               │                   │                              │
│  - Copie les documents        │                   │  - Reçoit les documents      │
│  - Synchronise modifications  │                   │  - Applique les mises à jour │
│  - Supprime après commit      │                   │  - Devient propriétaire      │
└───────────────────────────────┘                   └──────────────────────────────┘
                │                                                  │
                └──────────────────┬───────────────────────────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │     CONFIG SERVERS           │
                    │  - Mise à jour métadonnées   │
                    │  - Coordination globale      │
                    └──────────────────────────────┘
```

### Triggers de Migration

Les migrations sont déclenchées dans les situations suivantes :

1. **Balancer automatique** : Déséquilibre détecté entre shards
2. **Migration manuelle** : Commande `moveChunk` explicite
3. **Ajout de shard** : Distribution vers le nouveau shard
4. **Zone sharding** : Respect des contraintes de localisation
5. **Vidage d'un shard** : Avant suppression (`removeShard`)

---

## Processus Détaillé de Migration

### Phase 1 : Initiation (Balancer → Shard Source)

Le balancer identifie un déséquilibre et envoie une commande `moveChunk` au shard source.

```javascript
// Commande envoyée par le balancer (interne)
{
  moveChunk: "mydb.users",
  find: { user_id: "user_50000" },  // Point dans le chunk
  to: "shardB"                      // Shard destination
}
```

**Actions du shard source** :
1. Vérifie que le chunk existe et lui appartient
2. Acquiert un **verrou exclusif** sur les métadonnées du chunk
3. Notifie le shard destination pour préparer la réception
4. Initialise les structures de tracking des modifications

**Logs observables** :
```
[balancer] Starting chunk migration: mydb.users chunk [MinKey, user_50000) from shardA to shardB
[shardA] Acquired migration lock for chunk mydb.users [MinKey, user_50000)
```

### Phase 2 : Copie Initiale (Bulk Transfer)

Le shard destination **copie en masse** tous les documents du chunk depuis le shard source.

```javascript
// Processus interne de copie
// Le destination lit les documents du source via une connexion directe

// Shard Source : Itération sur les documents du chunk
db.users.find({
  $and: [
    { user_id: { $gte: MinKey } },
    { user_id: { $lt: "user_50000" } }
  ]
}).batchSize(10000)

// Shard Destination : Insertion en batch
db.users.insertMany([...documents...], { ordered: false })
```

**Caractéristiques** :
- Transfert par **batches** (typiquement 10 000 documents)
- Utilise une **connexion directe** entre les shards (pas via mongos)
- Les **écritures continuent** sur le shard source pendant la copie
- Durée proportionnelle à la **taille du chunk**

**Métriques observables** :
```javascript
// État de la migration en cours
db.currentOp({
  type: "op",
  op: { $in: ["command"] },
  "command.moveChunk": { $exists: true }
})

// Résultat exemple :
{
  "opid": "shardA:12345",
  "op": "command",
  "ns": "mydb.users",
  "command": { "moveChunk": "mydb.users", ... },
  "msg": "Cloning phase: copied 500000/800000 documents"
}
```

### Phase 3 : Synchronisation Incrémentale (Catch-up)

Pendant la copie initiale, des écritures ont eu lieu sur le shard source. Cette phase **rattrape les modifications**.

```javascript
// Le shard source maintient un log des opérations survenues pendant la copie
// Ces opérations sont rejouées sur le destination

// Modifications trackées :
// - Insertions (nouveau documents dans le chunk)
// - Mises à jour (modifications de documents existants)
// - Suppressions (documents supprimés)

// Exemple de rejeu :
// Source : updateOne({ user_id: "user_25000" }, { $set: { status: "active" } })
// Destination : Applique la même mise à jour
```

**Optimisations** :
- Utilise un **buffer en mémoire** pour tracker les modifications
- Plusieurs **passes de synchronisation** si nécessaire
- Timeout si trop de modifications (charge trop élevée)

**Cas problématique** :
```javascript
// Si le taux d'écriture est trop élevé :
// Le buffer de modifications déborde
// → La migration échoue et est retentée plus tard

// Log d'erreur :
"Migration failed: too much data in transfer mods queue (>500MB)"
```

### Phase 4 : Finalisation (Critical Section)

Phase critique où le shard source **bloque brièvement les écritures** pour transférer les dernières modifications.

```javascript
// Séquence critique (quelques centaines de millisecondes) :
// 1. Bloquer les écritures sur le chunk
// 2. Transférer les dernières modifications
// 3. Mettre à jour les métadonnées sur les config servers
// 4. Rediriger les futures écritures vers le destination
```

**Impact sur les applications** :
```javascript
// Les écritures pendant cette phase :
db.users.insertOne({ user_id: "user_30000", ... })

// Peuvent recevoir des erreurs transitoires :
// - WriteConcernError si timeout
// - OperationFailed si la migration échoue

// Les applications avec retry automatique gèrent cela sans problème
```

**Logs** :
```
[shardA] Entering critical section for chunk migration
[shardA] Critical section duration: 347ms
[configsvr] Updated chunk metadata: mydb.users [MinKey, user_50000) now on shardB
```

### Phase 5 : Commit et Nettoyage

Le chunk est désormais **officiellement** sur le shard destination.

```javascript
// 1. Config servers mettent à jour les métadonnées
db.getSiblingDB("config").chunks.updateOne(
  {
    ns: "mydb.users",
    min: { user_id: MinKey },
    max: { user_id: "user_50000" }
  },
  {
    $set: {
      shard: "shardB",
      lastmod: Timestamp(...)
    }
  }
)

// 2. Les mongos sont notifiés du changement
// 3. Ils mettent à jour leur cache de routing
// 4. Les futures requêtes sont dirigées vers shardB

// 5. Le shard source supprime les documents du chunk
db.users.deleteMany({
  $and: [
    { user_id: { $gte: MinKey } },
    { user_id: { $lt: "user_50000" } }
  ]
})
```

**Délai de suppression** :
```javascript
// Par défaut, suppression immédiate
// Peut être configuré pour attendre (prudence) :

db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  { $set: { _waitForDelete: true } }
)

// Avec _waitForDelete: true
// → Le balancer attend que la suppression soit terminée
// → Évite les surcharges sur le source
```

### Phase 6 : Validation (Post-migration)

```javascript
// Vérification automatique par MongoDB
// 1. Compter les documents sur le destination
db.users.countDocuments({
  $and: [
    { user_id: { $gte: MinKey } },
    { user_id: { $lt: "user_50000" } }
  ]
})

// 2. Vérifier les métadonnées
db.getSiblingDB("config").chunks.findOne({
  ns: "mydb.users",
  min: { user_id: MinKey }
})

// 3. Logger le succès
// [configsvr] Migration succeeded: mydb.users chunk [MinKey, user_50000)
//             from shardA to shardB in 45.2 seconds
```

---

## Migration Manuelle de Chunks

### Commande moveChunk

```javascript
// Syntaxe de base
sh.moveChunk(
  "mydb.users",                    // Collection
  { user_id: "user_50000" },       // Point dans le chunk à migrer
  "shardB"                         // Shard destination
)

// Ou via adminCommand
db.adminCommand({
  moveChunk: "mydb.users",
  find: { user_id: "user_50000" },
  to: "shardB",
  _waitForDelete: true             // Attendre suppression sur source
})
```

### Options Avancées

```javascript
// Migration avec options de performance
db.adminCommand({
  moveChunk: "mydb.users",
  find: { user_id: "user_50000" },
  to: "shardB",

  // Options avancées
  _secondaryThrottle: false,       // Désactiver throttling (plus rapide)
  writeConcern: { w: 1 },          // Write concern pendant migration
  maxTimeMS: 3600000               // Timeout : 1 heure
})
```

**Paramètres critiques** :

| Option | Valeur | Impact |
|--------|--------|--------|
| `_secondaryThrottle` | `false` | Migration plus rapide mais charge réseau accrue |
| `_waitForDelete` | `true` | Évite la surcharge du shard source |
| `maxTimeMS` | `3600000` | Timeout pour migrations très longues |
| `writeConcern` | `{ w: "majority" }` | Garanties de durabilité vs performance |

### Cas d'Usage : Migration Manuelle

#### 1. Préparation d'une Maintenance

```javascript
// Avant maintenance sur shardA
// Migrer tous les chunks vers d'autres shards

// Lister les chunks sur shardA
var chunks = db.getSiblingDB("config").chunks.find({
  ns: "mydb.users",
  shard: "shardA"
}).toArray();

// Désactiver le balancer
sh.stopBalancer();

// Migrer un par un vers shardB
chunks.forEach(function(chunk) {
  print("Migrating chunk: " + tojson(chunk.min) + " to " + tojson(chunk.max));

  sh.moveChunk(
    "mydb.users",
    chunk.min,
    "shardB"
  );
});

// Réactiver le balancer après
sh.startBalancer();
```

#### 2. Équilibrage Forcé

```javascript
// Distribution initiale déséquilibrée
// shardA : 80 chunks
// shardB : 20 chunks

// Forcer l'équilibrage manuel
var targetPerShard = 50;

// Migrer 30 chunks de A vers B
for (var i = 0; i < 30; i++) {
  var chunk = db.getSiblingDB("config").chunks.findOne({
    ns: "mydb.users",
    shard: "shardA"
  });

  if (chunk) {
    sh.moveChunk("mydb.users", chunk.min, "shardB");
    sleep(5000);  // Pause entre migrations
  }
}
```

#### 3. Zone Sharding : Migration vers Zones

```javascript
// Définir des zones géographiques
sh.addShardToZone("shardEU", "europe")
sh.addShardToZone("shardUS", "usa")

// Définir les plages
sh.updateZoneKeyRange(
  "mydb.users",
  { region: "EU", user_id: MinKey },
  { region: "EU", user_id: MaxKey },
  "europe"
)

sh.updateZoneKeyRange(
  "mydb.users",
  { region: "US", user_id: MinKey },
  { region: "US", user_id: MaxKey },
  "usa"
)

// Le balancer migrera automatiquement les chunks
// Ou forcer manuellement :
sh.moveChunk(
  "mydb.users",
  { region: "EU", user_id: "user_1000" },
  "shardEU"
)
```

---

## Performance et Optimisation

### Métriques de Performance

```javascript
// Durée typique d'une migration par taille de chunk
// (sur réseau 1 Gbps, charge normale)

// Chunk 10 MB   : 2-5 secondes
// Chunk 64 MB   : 10-30 secondes
// Chunk 128 MB  : 30-60 secondes
// Chunk 256 MB  : 1-3 minutes
// Chunk 1 GB    : 5-15 minutes

// Mesurer la durée d'une migration
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.commit",
  ns: "mydb.users"
}).sort({ time: -1 }).limit(10).forEach(function(entry) {
  var start = entry.details.step1of6;
  var end = entry.time;
  var durationMs = end - start;

  print(
    "Chunk: " + tojson(entry.details.min) + " → " + tojson(entry.details.max) +
    " | Duration: " + (durationMs / 1000).toFixed(2) + "s" +
    " | From: " + entry.details.from + " | To: " + entry.details.to
  );
});
```

### Facteurs Impactant la Performance

#### 1. Taille du Chunk

```javascript
// Impact direct sur la durée
// Optimiser la taille des chunks :

// Chunks trop petits (< 32 MB)
// - Migrations fréquentes → overhead
// - Fragmentation métadonnées

// Chunks trop gros (> 256 MB)
// - Migrations lentes
// - Impact performance prolongé
// - Risque de timeout

// Taille optimale : 64-128 MB
use config
db.settings.updateOne(
  { _id: "chunksize" },
  { $set: { value: 64 } },
  { upsert: true }
)
```

#### 2. Charge du Système

```javascript
// Migration pendant pic de charge
// → Ralentissement, timeouts possibles

// Stratégie : Fenêtre de migration
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "01:00",  // 1h du matin
        stop: "05:00"    // 5h du matin
      }
    }
  },
  { upsert: true }
)
```

#### 3. Bande Passante Réseau

```javascript
// Mesurer la bande passante entre shards
// Sur le shard source :

// Test avec iperf (hors MongoDB)
// shardA → shardB : iperf3 -c shardB_ip

// Résultats typiques :
// - 1 Gbps réseau : 100-120 MB/s théorique
// - Réel MongoDB : 60-80 MB/s (overhead protocole)

// Pour 1 chunk de 128 MB :
// Temps transfert = 128 MB / 80 MB/s ≈ 1.6 secondes
// + overhead copie + sync + commit ≈ 30-60 secondes total
```

#### 4. Type de Stockage

```javascript
// HDD vs SSD impact significatif

// HDD (7200 RPM)
// - Lecture séquentielle : 100-150 MB/s
// - IOPS random : 100-200
// → Migrations lentes (bottleneck disque)

// SSD SATA
// - Lecture séquentielle : 500-550 MB/s
// - IOPS random : 10 000+
// → Migrations 3-5x plus rapides

// NVMe
// - Lecture séquentielle : 3000-7000 MB/s
// - IOPS random : 100 000+
// → Migrations très rapides (bottleneck réseau)
```

### Optimisations Avancées

#### 1. Parallélisation des Migrations

```javascript
// MongoDB 4.2+ : Migrations parallèles
// Par défaut : 1 migration à la fois par collection

// Augmenter le parallélisme (avec précaution)
db.adminCommand({
  setParameter: 1,
  migrationMaxConcurrency: 2  // 2 migrations simultanées
})

// Attention : Charge CPU/réseau doublée
// Recommandé uniquement pour :
// - Maintenance planifiée
// - Hardware puissant
// - Réseau haute capacité (10 Gbps+)
```

#### 2. Désactivation du Throttling

```javascript
// Par défaut, MongoDB "throttle" les migrations
// pour limiter l'impact sur les réplications secondaires

// Désactiver pour migrations rapides (maintenance)
db.adminCommand({
  moveChunk: "mydb.users",
  find: { user_id: "user_50000" },
  to: "shardB",
  _secondaryThrottle: false  // Plus rapide
})

// Impact : Réplication vers secondaries peut prendre du retard
// Acceptable lors de fenêtre de maintenance
```

#### 3. Compression du Transfert

```javascript
// MongoDB 4.2+ : Compression automatique du transfert réseau
// (si activée dans la configuration)

// mongod.conf
net:
  compression:
    compressors: "snappy,zlib,zstd"

// Gains typiques :
// - Données texte : 60-80% réduction
// - Données binaires : 20-40% réduction
// - Trade-off CPU vs réseau
```

---

## Monitoring et Diagnostics

### Commandes de Monitoring

```javascript
// 1. État du balancer
sh.getBalancerState()  // true/false
sh.isBalancerRunning()  // true si migration en cours

// 2. Migrations en cours
db.currentOp({
  $or: [
    { op: "command", "command.moveChunk": { $exists: true } },
    { desc: /^migrateThread/ }
  ]
})

// Résultat détaillé :
{
  "opid": "shardA:67890",
  "op": "command",
  "ns": "mydb.users",
  "command": {
    "moveChunk": "mydb.users",
    "find": { "user_id": "user_50000" },
    "to": "shardB"
  },
  "msg": "Cloning phase: 700000/800000 documents",
  "progress": { "done": 700000, "total": 800000 },
  "microsecs_running": 34567890,  // ~34 secondes
  "numYields": 0
}

// 3. Historique des migrations
db.getSiblingDB("config").changelog.find({
  what: { $in: ["moveChunk.start", "moveChunk.commit", "moveChunk.error"] },
  time: { $gte: ISODate("2024-01-15T00:00:00Z") }
}).sort({ time: -1 }).limit(20)

// 4. Distribution actuelle
db.users.getShardDistribution()

// 5. Chunks par shard
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "mydb.users" } },
  { $group: {
      _id: "$shard",
      numChunks: { $sum: 1 },
      jumbo: { $sum: { $cond: ["$jumbo", 1, 0] } }
    }
  }
])
```

### Dashboard de Monitoring

```javascript
// Script de monitoring automatisé
function monitorMigrations() {
  print("=== État des Migrations ===");
  print("Date : " + new Date());
  print("");

  // 1. Balancer
  var balancerState = sh.getBalancerState();
  var balancerRunning = sh.isBalancerRunning();

  print("Balancer activé : " + balancerState);
  print("Migration en cours : " + balancerRunning);
  print("");

  // 2. Migrations actives
  var activeMigrations = db.currentOp({
    op: "command",
    "command.moveChunk": { $exists: true }
  }).inprog;

  if (activeMigrations.length > 0) {
    print("Migrations actives : " + activeMigrations.length);
    activeMigrations.forEach(function(op) {
      var progress = op.progress ?
        ((op.progress.done / op.progress.total) * 100).toFixed(2) + "%" :
        "N/A";

      print("  - Collection : " + op.ns);
      print("    Progression : " + progress);
      print("    Durée : " + (op.microsecs_running / 1000000).toFixed(2) + "s");
      print("    Message : " + op.msg);
      print("");
    });
  } else {
    print("Aucune migration active");
  }
  print("");

  // 3. Dernières migrations (24h)
  var recentMigrations = db.getSiblingDB("config").changelog.aggregate([
    {
      $match: {
        what: "moveChunk.commit",
        time: { $gte: new Date(Date.now() - 86400000) }
      }
    },
    {
      $group: {
        _id: "$ns",
        count: { $sum: 1 }
      }
    },
    { $sort: { count: -1 } }
  ]).toArray();

  if (recentMigrations.length > 0) {
    print("Migrations (24h) :");
    recentMigrations.forEach(function(stat) {
      print("  - " + stat._id + " : " + stat.count + " migrations");
    });
  }
}

// Exécuter toutes les 5 minutes
while (true) {
  monitorMigrations();
  sleep(300000);  // 5 minutes
}
```

### Alertes Critiques

```javascript
// Scénarios nécessitant des alertes

// 1. Migration échoue de manière répétée
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.error",
  time: { $gte: new Date(Date.now() - 3600000) }  // Dernière heure
}).count()
// Si count > 5 → Alerte

// 2. Migration dure trop longtemps
db.currentOp({
  op: "command",
  "command.moveChunk": { $exists: true },
  microsecs_running: { $gt: 3600000000 }  // > 1 heure
})
// Si résultat → Alerte

// 3. Jumbo chunks bloquent le balancer
db.getSiblingDB("config").chunks.find({
  ns: { $regex: /^mydb\./ },
  jumbo: true
}).count()
// Si count > 0 → Alerte

// 4. Déséquilibre important persiste
var distribution = db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "mydb.users" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } }
]).toArray();

var max = Math.max(...distribution.map(d => d.count));
var min = Math.min(...distribution.map(d => d.count));
var imbalance = ((max - min) / min) * 100;

if (imbalance > 20) {
  print("⚠️  Déséquilibre : " + imbalance.toFixed(2) + "%");
}
```

---

## Problèmes Courants et Résolutions

### Problème 1 : Migration Timeout

**Symptômes** :
```javascript
// Log sur le shard source
"Migration failed: chunk migration timed out after 3600 seconds"

// Ou dans db.currentOp() après longue durée
{
  "opid": "shardA:12345",
  "microsecs_running": 3600000000,  // 1 heure
  "msg": "Waiting for critical section"
}
```

**Causes** :
1. Chunk trop volumineux (>500 MB)
2. Charge d'écriture trop élevée sur le chunk
3. Réseau lent ou instable
4. Disque saturé (I/O bottleneck)

**Solutions** :

```javascript
// Solution 1 : Augmenter le timeout
db.adminCommand({
  moveChunk: "mydb.users",
  find: { user_id: "user_50000" },
  to: "shardB",
  maxTimeMS: 7200000  // 2 heures au lieu de 1
})

// Solution 2 : Splitter le chunk avant migration
sh.splitAt("mydb.users", { user_id: "user_75000" })
// Puis migrer les deux morceaux séparément

// Solution 3 : Réduire la charge pendant la migration
// - Activer la fenêtre de balancing (heures creuses)
// - Throttle des écritures applicatives temporairement

// Solution 4 : Désactiver throttling (si acceptable)
db.adminCommand({
  moveChunk: "mydb.users",
  find: { user_id: "user_50000" },
  to: "shardB",
  _secondaryThrottle: false
})
```

### Problème 2 : Jumbo Chunk Immobile

**Symptômes** :
```javascript
// Chunk marqué jumbo
db.getSiblingDB("config").chunks.findOne({
  ns: "mydb.orders",
  min: { status: "active" }
})

// Résultat :
{
  "_id": ObjectId("..."),
  "ns": "mydb.orders",
  "min": { "status": "active" },
  "max": { "status": "inactive" },
  "shard": "shardA",
  "jumbo": true  // ⚠️ Jumbo chunk
}

// Le balancer ne peut pas le migrer
// Log : "cannot move chunk: too large to migrate"
```

**Causes** :
- Shard key de faible cardinalité
- Tous les documents ont la même valeur de shard key

**Solutions** :

```javascript
// Solution 1 : Forcer la migration (MongoDB 4.4+)
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  { $set: { attemptToBalanceJumboChunks: true } },
  { upsert: true }
)

sh.moveChunk("mydb.orders", { status: "active" }, "shardB")

// Solution 2 : Refine la shard key (MongoDB 5.0+)
db.adminCommand({
  refineCollectionShardKey: "mydb.orders",
  key: { status: 1, customer_id: 1 }  // Ajouter customer_id
})

// Solution 3 : Resharding complet (MongoDB 5.0+)
db.adminCommand({
  reshardCollection: "mydb.orders",
  key: { customer_id: "hashed" }  // Nouvelle shard key
})

// Solution 4 : Recréer la collection (dernière option)
// - Exporter les données
// - Créer nouvelle collection avec bonne shard key
// - Importer les données
// - Renommer
```

### Problème 3 : Échec Répété de Migration

**Symptômes** :
```javascript
// Changelog montre des échecs répétés
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.error",
  ns: "mydb.users",
  time: { $gte: new Date(Date.now() - 3600000) }
}).count()
// Résultat : 10 échecs dans la dernière heure

// Logs détaillés
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.error",
  ns: "mydb.users"
}).sort({ time: -1 }).limit(1).pretty()

// Erreur typique :
{
  "what": "moveChunk.error",
  "ns": "mydb.users",
  "details": {
    "errmsg": "Migration abort requested",
    "from": "shardA",
    "to": "shardB"
  }
}
```

**Causes** :
1. Charge trop élevée (buffer overflow)
2. Problème réseau entre shards
3. Espace disque insuffisant sur destination
4. Config servers inaccessibles

**Diagnostic et Résolution** :

```javascript
// Diagnostic 1 : Vérifier la charge
db.currentOp({ secs_running: { $gte: 10 } }).inprog.length
// Si > 50 → surcharge

// Diagnostic 2 : Vérifier l'espace disque
db.adminCommand({ dbStats: 1, scale: 1024*1024*1024 })  // GB
// Vérifier dataSize vs diskSpace

// Diagnostic 3 : Tester la connectivité réseau
// Depuis shardA vers shardB
mongosh --host shardB_ip:27018 --eval "db.adminCommand({ ping: 1 })"

// Diagnostic 4 : Vérifier les config servers
db.adminCommand({ replSetGetStatus: 1 })  // Sur config server

// Résolution :
// - Réduire la charge (throttle applicatif)
// - Nettoyer l'espace disque
// - Résoudre les problèmes réseau
// - Redémarrer les config servers si nécessaire
```

### Problème 4 : Migration Bloquée en Critical Section

**Symptômes** :
```javascript
// Migration bloquée depuis longtemps
db.currentOp({
  op: "command",
  "command.moveChunk": { $exists: true },
  microsecs_running: { $gt: 300000000 }  // > 5 minutes
})

// msg: "Entering critical section"
// → Bloqué, les écritures sont bloquées
```

**Causes** :
- Deadlock avec autre opération
- Config servers injoignables
- Bug MongoDB (rare)

**Résolution d'urgence** :

```javascript
// 1. Identifier l'opération bloquée
var blockedOp = db.currentOp({
  op: "command",
  "command.moveChunk": { $exists: true }
}).inprog[0];

print("OpID : " + blockedOp.opid);

// 2. Tenter d'annuler (attention : risque)
db.killOp(blockedOp.opid)

// 3. Si pas de réponse, redémarrer le mongod (shard source)
// En dernier recours seulement

// 4. Vérifier après redémarrage
sh.status()
db.getSiblingDB("config").chunks.findOne({
  ns: "mydb.users",
  min: { user_id: "user_50000" }
})

// 5. Re-tenter la migration après investigation
```

---

## Stratégies de Migration en Production

### Stratégie 1 : Migration Progressive (Évolutive)

Pour ajout d'un nouveau shard sans impact :

```javascript
// Contexte : Ajouter shardC au cluster existant (shardA, shardB)

// Étape 1 : Ajouter le shard
sh.addShard("shardC/shardC1.example.com:27018,shardC2.example.com:27018,shardC3.example.com:27018")

// Étape 2 : Le balancer migrera automatiquement
// Mais pour contrôler le rythme :

// 2a. Désactiver le balancer
sh.stopBalancer()

// 2b. Migrer manuellement par petits lots
var targetChunksPerShard = 30;  // Objectif final

for (var i = 0; i < targetChunksPerShard; i++) {
  // Trouver un chunk sur shardA ou shardB
  var chunk = db.getSiblingDB("config").chunks.findOne({
    ns: "mydb.users",
    shard: { $in: ["shardA", "shardB"] }
  });

  if (chunk) {
    print("Migrating chunk " + (i+1) + "/" + targetChunksPerShard);
    sh.moveChunk("mydb.users", chunk.min, "shardC");

    // Pause entre migrations
    sleep(10000);  // 10 secondes
  }
}

// 2c. Réactiver le balancer pour finaliser
sh.startBalancer()
```

### Stratégie 2 : Vidage d'un Shard (Décommission)

Pour retirer un shard du cluster :

```javascript
// Contexte : Retirer shardA (hardware obsolète)

// Étape 1 : Marquer le shard pour removal
db.adminCommand({ removeShard: "shardA" })

// Résultat :
{
  "msg": "draining started successfully",
  "state": "started",
  "shard": "shardA",
  "remaining": {
    "chunks": 45,
    "dbs": 2
  }
}

// Étape 2 : MongoDB migre automatiquement tous les chunks
// Monitoring de la progression
db.adminCommand({ removeShard: "shardA" })

// Résultat intermédiaire :
{
  "msg": "draining ongoing",
  "state": "ongoing",
  "remaining": {
    "chunks": 12,  // Réduction progressive
    "dbs": 2
  }
}

// Étape 3 : Gérer les bases primaires
// Certaines bases ont shardA comme shard primaire
// Il faut les déplacer manuellement

db.adminCommand({ movePrimary: "mydb", to: "shardB" })

// Étape 4 : Finaliser la removal
db.adminCommand({ removeShard: "shardA" })

// Résultat final :
{
  "msg": "removeshard completed successfully",
  "state": "completed",
  "shard": "shardA"
}

// Étape 5 : Arrêter le shard physiquement
// Les membres du replica set shardA peuvent être arrêtés
```

### Stratégie 3 : Zone Sharding et Migration Ciblée

Pour conformité géographique (GDPR, etc.) :

```javascript
// Contexte : Données européennes doivent rester en Europe

// Étape 1 : Définir les zones
sh.addShardToZone("shardEU1", "europe")
sh.addShardToZone("shardEU2", "europe")
sh.addShardToZone("shardUS1", "usa")
sh.addShardToZone("shardUS2", "usa")

// Étape 2 : Associer les plages de données aux zones
sh.updateZoneKeyRange(
  "mydb.users",
  { country: "FR", user_id: MinKey },
  { country: "FR", user_id: MaxKey },
  "europe"
)

sh.updateZoneKeyRange(
  "mydb.users",
  { country: "DE", user_id: MinKey },
  { country: "DE", user_id: MaxKey },
  "europe"
)

sh.updateZoneKeyRange(
  "mydb.users",
  { country: "US", user_id: MinKey },
  { country: "US", user_id: MaxKey },
  "usa"
)

// Étape 3 : Le balancer migre automatiquement
// Vérifier que les migrations respectent les zones
db.getSiblingDB("config").tags.find().pretty()

// Étape 4 : Valider la conformité
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "mydb.users" } },
  {
    $lookup: {
      from: "shards",
      localField: "shard",
      foreignField: "_id",
      as: "shardInfo"
    }
  },
  {
    $group: {
      _id: {
        min: "$min",
        max: "$max",
        shard: "$shard"
      }
    }
  }
])
```

---

## Anti-Patterns et Erreurs Critiques

### ❌ Anti-Pattern 1 : Migrations Pendant Pic de Charge

**Problème** :
```javascript
// Le balancer est actif 24/7
// Migrations se produisent pendant les heures de pointe
// → Impact performance pour les utilisateurs
```

**Conséquence** :
- Latence accrue pour les requêtes
- Timeouts applicatifs
- Dégradation de l'expérience utilisateur

**Solution** :
```javascript
// Définir une fenêtre de maintenance
db.getSiblingDB("config").settings.updateOne(
  { _id: "balancer" },
  {
    $set: {
      activeWindow: {
        start: "02:00",  // 2h du matin
        stop: "06:00"    // 6h du matin
      }
    }
  },
  { upsert: true }
)

// Ou désactiver complètement pendant événements critiques
// (Black Friday, lancement produit)
sh.stopBalancer()
```

### ❌ Anti-Pattern 2 : Ignorer les Échecs de Migration

**Problème** :
```javascript
// Les migrations échouent régulièrement
// Mais aucune alerte ni investigation
// → Déséquilibre s'aggrave
```

**Conséquence** :
- Distribution déséquilibrée persistante
- Hot spots sur certains shards
- Performances dégradées à long terme

**Solution** :
```javascript
// Monitoring actif des échecs
// Script d'alerte (à intégrer dans votre monitoring)
function checkMigrationFailures() {
  var failures = db.getSiblingDB("config").changelog.find({
    what: "moveChunk.error",
    time: { $gte: new Date(Date.now() - 86400000) }
  }).count();

  if (failures > 5) {
    // Déclencher alerte
    print("⚠️  ALERTE : " + failures + " échecs de migration dans les 24h");

    // Logger les détails
    db.getSiblingDB("config").changelog.find({
      what: "moveChunk.error",
      time: { $gte: new Date(Date.now() - 86400000) }
    }).forEach(function(failure) {
      printjson(failure.details);
    });
  }
}

// Exécuter quotidiennement
checkMigrationFailures();
```

### ❌ Anti-Pattern 3 : Migration Manuelle Sans Arrêter le Balancer

**Problème** :
```javascript
// Migration manuelle pendant que le balancer est actif
sh.moveChunk("mydb.users", { user_id: "user_50000" }, "shardB")

// En parallèle, le balancer tente de migrer d'autres chunks
// → Conflit possible, comportement imprévisible
```

**Conséquence** :
- Conflits de verrouillage
- Migrations échouées
- Métadonnées incohérentes (rare mais possible)

**Solution** :
```javascript
// TOUJOURS arrêter le balancer avant migration manuelle
sh.stopBalancer()

// Attendre que toutes migrations en cours se terminent
while (sh.isBalancerRunning()) {
  print("Attente arrêt du balancer...");
  sleep(1000);
}

// Puis effectuer les migrations manuelles
sh.moveChunk("mydb.users", { user_id: "user_50000" }, "shardB")
// ...

// Réactiver après
sh.startBalancer()
```

### ❌ Anti-Pattern 4 : Pas de Test de Migration

**Problème** :
```bash
# Déployer un nouveau cluster en production
# Sans jamais avoir testé les migrations
# → Comportement inconnu en condition réelle
```

**Conséquence** :
- Surprises désagréables (timeouts, échecs)
- Pas de baseline de performance
- Panique lors de problèmes

**Solution** :
```javascript
// Phase de test pré-production
// 1. Créer un environnement de staging identique
// 2. Charger des données réalistes
// 3. Tester les migrations

// Test de migration
function testMigration() {
  var startTime = new Date();

  sh.moveChunk("mydb.users", { user_id: "user_50000" }, "shardB");

  // Attendre la fin
  while (sh.isBalancerRunning()) {
    sleep(100);
  }

  var endTime = new Date();
  var duration = (endTime - startTime) / 1000;

  print("Migration testée : " + duration + " secondes");

  // Vérifier la distribution
  db.users.getShardDistribution();
}

testMigration();

// Documenter les résultats
// Créer des runbooks basés sur l'expérience
```

### ❌ Anti-Pattern 5 : Sur-Parallélisation

**Problème** :
```javascript
// Augmenter excessivement le parallélisme
db.adminCommand({
  setParameter: 1,
  migrationMaxConcurrency: 10  // 10 migrations simultanées !
})
```

**Conséquence** :
- Saturation réseau
- Contention CPU/disque
- Toutes les migrations sont lentes
- Instabilité du cluster

**Solution** :
```javascript
// Parallélisme raisonnable
// Règle : max 2-3 migrations simultanées
db.adminCommand({
  setParameter: 1,
  migrationMaxConcurrency: 2
})

// Uniquement pendant maintenance planifiée
// Surveiller les métriques :
// - Utilisation réseau < 70%
// - Utilisation CPU < 80%
// - Latence réseau stable
```

---

## Outils et Scripts Utiles

### Script de Monitoring Complet

```javascript
// migration-monitor.js
// À exécuter via mongosh

function comprehensiveMigrationMonitor() {
  print("=".repeat(80));
  print("MONITORING DES MIGRATIONS - " + new Date().toISOString());
  print("=".repeat(80));
  print("");

  // 1. État du balancer
  print("1. État du Balancer");
  print("-".repeat(40));
  print("  Activé : " + sh.getBalancerState());
  print("  En cours : " + sh.isBalancerRunning());

  // Fenêtre active
  var balancerConfig = db.getSiblingDB("config").settings.findOne({ _id: "balancer" });
  if (balancerConfig && balancerConfig.activeWindow) {
    print("  Fenêtre active : " + balancerConfig.activeWindow.start + " - " + balancerConfig.activeWindow.stop);
  } else {
    print("  Fenêtre active : Aucune (actif 24/7)");
  }
  print("");

  // 2. Migrations en cours
  print("2. Migrations Actives");
  print("-".repeat(40));
  var activeMigrations = db.currentOp({
    $or: [
      { op: "command", "command.moveChunk": { $exists: true } },
      { desc: /^migrateThread/ }
    ]
  }).inprog;

  if (activeMigrations.length === 0) {
    print("  Aucune migration active");
  } else {
    activeMigrations.forEach(function(mig) {
      print("  Collection : " + mig.ns);
      print("  Shard source : " + (mig.command ? mig.command.find : "N/A"));
      print("  Shard destination : " + (mig.command ? mig.command.to : "N/A"));
      print("  Durée : " + (mig.microsecs_running / 1000000).toFixed(2) + "s");
      print("  État : " + mig.msg);
      print("");
    });
  }
  print("");

  // 3. Statistiques récentes (24h)
  print("3. Statistiques (24 heures)");
  print("-".repeat(40));

  var last24h = new Date(Date.now() - 86400000);

  var successCount = db.getSiblingDB("config").changelog.count({
    what: "moveChunk.commit",
    time: { $gte: last24h }
  });

  var errorCount = db.getSiblingDB("config").changelog.count({
    what: "moveChunk.error",
    time: { $gte: last24h }
  });

  print("  Migrations réussies : " + successCount);
  print("  Migrations échouées : " + errorCount);

  if (errorCount > 0) {
    print("  ⚠️  Taux d'échec : " + ((errorCount / (successCount + errorCount)) * 100).toFixed(2) + "%");
  }
  print("");

  // 4. Distribution par collection
  print("4. Distribution des Chunks");
  print("-".repeat(40));

  var collections = db.getSiblingDB("config").collections.find({ dropped: false }).toArray();

  collections.forEach(function(coll) {
    if (coll.key) {  // Collection shardée
      var distribution = db.getSiblingDB("config").chunks.aggregate([
        { $match: { ns: coll._id } },
        { $group: { _id: "$shard", count: { $sum: 1 } } },
        { $sort: { count: -1 } }
      ]).toArray();

      print("  " + coll._id);
      distribution.forEach(function(dist) {
        print("    - " + dist._id + " : " + dist.count + " chunks");
      });
    }
  });
  print("");

  // 5. Jumbo chunks
  print("5. Jumbo Chunks");
  print("-".repeat(40));

  var jumboChunks = db.getSiblingDB("config").chunks.aggregate([
    { $match: { jumbo: true } },
    { $group: { _id: "$ns", count: { $sum: 1 } } }
  ]).toArray();

  if (jumboChunks.length === 0) {
    print("  Aucun jumbo chunk");
  } else {
    print("  ⚠️  Collections avec jumbo chunks :");
    jumboChunks.forEach(function(jc) {
      print("    - " + jc._id + " : " + jc.count + " jumbo chunks");
    });
  }
  print("");

  // 6. Recommandations
  print("6. Recommandations");
  print("-".repeat(40));

  var recommendations = [];

  if (errorCount > successCount * 0.1) {
    recommendations.push("Taux d'échec élevé (>" + (errorCount / (successCount + errorCount) * 100).toFixed(0) + "%) - Investiguer les causes");
  }

  if (jumboChunks.length > 0) {
    recommendations.push("Jumbo chunks détectés - Revoir la shard key ou forcer la migration");
  }

  if (activeMigrations.some(m => m.microsecs_running > 1800000000)) {
    recommendations.push("Migration longue (>30min) - Vérifier la charge et la bande passante");
  }

  if (recommendations.length === 0) {
    print("  ✅ Tout semble normal");
  } else {
    recommendations.forEach(function(rec) {
      print("  ⚠️  " + rec);
    });
  }

  print("");
  print("=".repeat(80));
}

// Exécuter
comprehensiveMigrationMonitor();
```

### Script de Migration Contrôlée

```javascript
// controlled-migration.js
// Pour migrer un lot de chunks de manière contrôlée

function controlledMigration(namespace, sourceShard, destShard, numChunks, pauseMs) {
  print("Début migration contrôlée");
  print("Collection : " + namespace);
  print("Source : " + sourceShard);
  print("Destination : " + destShard);
  print("Nombre de chunks : " + numChunks);
  print("Pause entre migrations : " + (pauseMs / 1000) + "s");
  print("");

  // Arrêter le balancer
  print("Arrêt du balancer...");
  sh.stopBalancer();

  while (sh.isBalancerRunning()) {
    print("  Attente arrêt balancer...");
    sleep(1000);
  }
  print("  ✅ Balancer arrêté");
  print("");

  var migratedCount = 0;
  var failedCount = 0;

  for (var i = 0; i < numChunks; i++) {
    // Trouver un chunk sur le source shard
    var chunk = db.getSiblingDB("config").chunks.findOne({
      ns: namespace,
      shard: sourceShard
    });

    if (!chunk) {
      print("Plus de chunks à migrer sur " + sourceShard);
      break;
    }

    print("[" + (i+1) + "/" + numChunks + "] Migration chunk : " + tojson(chunk.min) + " → " + tojson(chunk.max));

    try {
      var startTime = new Date();

      sh.moveChunk(namespace, chunk.min, destShard);

      var endTime = new Date();
      var duration = (endTime - startTime) / 1000;

      print("  ✅ Succès en " + duration.toFixed(2) + "s");
      migratedCount++;

    } catch (e) {
      print("  ❌ Échec : " + e.message);
      failedCount++;
    }

    // Pause avant la prochaine migration
    if (i < numChunks - 1 && pauseMs > 0) {
      print("  Pause de " + (pauseMs / 1000) + "s...");
      sleep(pauseMs);
    }

    print("");
  }

  // Résumé
  print("=".repeat(40));
  print("RÉSUMÉ");
  print("Migrés avec succès : " + migratedCount);
  print("Échecs : " + failedCount);
  print("=".repeat(40));
  print("");

  // Réactiver le balancer
  print("Réactivation du balancer...");
  sh.startBalancer();
  print("  ✅ Balancer réactivé");
}

// Exemple d'utilisation :
// controlledMigration("mydb.users", "shardA", "shardB", 10, 5000);
```

---

## Conclusion

La migration de chunks est le cœur de l'élasticité et de l'équilibrage de charge dans MongoDB. Une compréhension approfondie de ce processus est essentielle pour :

- ✅ **Maintenir un cluster équilibré** : Distribution uniforme des données
- ✅ **Diagnostiquer les problèmes** : Timeouts, échecs, jumbo chunks
- ✅ **Optimiser les performances** : Fenêtres de migration, parallélisme, throttling
- ✅ **Gérer les opérations** : Ajout/retrait de shards, zone sharding
- ✅ **Anticiper les impacts** : Charge réseau, latence, disponibilité

**Points clés à retenir** :
- Les migrations sont **automatiques** mais **configurables**
- Elles s'exécutent **en arrière-plan** sans interruption de service
- Mais peuvent **impacter les performances** si mal gérées
- Le **monitoring proactif** est indispensable en production
- Les **fenêtres de maintenance** protègent les heures critiques

En production, privilégiez toujours :
- **Monitoring continu** des migrations
- **Alertes** sur échecs et durées anormales
- **Fenêtres de balancing** adaptées à votre charge
- **Tests** avant toute opération critique

---

## Ressources

- [MongoDB Documentation - Chunk Migration](https://docs.mongodb.com/manual/core/sharding-balancer-administration/)
- [MongoDB Documentation - moveChunk Command](https://docs.mongodb.com/manual/reference/command/moveChunk/)
- [MongoDB Blog - Chunk Migration Internals](https://www.mongodb.com/blog)
- [MongoDB University - M103: Basic Cluster Administration](https://university.mongodb.com/)

---


⏭️ [Opérations sur un cluster shardé](/10-sharding/09-operations-cluster-sharde.md)
