🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.5 Chunks et Balancing

## Introduction

Dans un cluster shardé MongoDB, les données ne sont pas distribuées de manière arbitraire entre les shards. Elles sont organisées en **chunks** (morceaux), qui constituent l'unité fondamentale de distribution des données. Le **balancer** est le composant responsable de maintenir une distribution équilibrée de ces chunks entre les différents shards du cluster.

Cette section explore en profondeur les mécanismes de chunking et de balancing, essentiels pour maintenir les performances et l'évolutivité d'un cluster shardé en production.

---

## Qu'est-ce qu'un Chunk ?

### Définition

Un **chunk** est un intervalle contigu de données de la shard key. Chaque chunk :
- Possède une **limite inférieure** (minKey) et une **limite supérieure** (maxKey)
- Est assigné à un **shard spécifique**
- A une **taille maximale** par défaut de **64 Mo** (configurable entre 1 Mo et 1024 Mo)
- Représente l'unité atomique de migration entre shards

### Exemple de structure de chunks

Pour une collection `orders` shardée sur `{ customer_id: 1 }` :

```javascript
// Chunk 1 : Shard A
{
  "_id": "customer_1_to_customer_5000",
  "min": { "customer_id": MinKey },
  "max": { "customer_id": "customer_5000" },
  "shard": "shard-a",
  "lastmod": Timestamp(1, 0)
}

// Chunk 2 : Shard B
{
  "_id": "customer_5000_to_customer_10000",
  "min": { "customer_id": "customer_5000" },
  "max": { "customer_id": "customer_10000" },
  "shard": "shard-b",
  "lastmod": Timestamp(1, 1)
}

// Chunk 3 : Shard A
{
  "_id": "customer_10000_to_maxkey",
  "min": { "customer_id": "customer_10000" },
  "max": { "customer_id": MaxKey },
  "shard": "shard-a",
  "lastmod": Timestamp(1, 2)
}
```

### Métadonnées des chunks

MongoDB stocke les métadonnées des chunks dans la collection `config.chunks` sur les config servers :

```javascript
// Visualiser tous les chunks d'une collection
db.getSiblingDB("config").chunks.find(
  { ns: "mydb.mycollection" }
).sort({ min: 1 }).pretty()

// Compter les chunks par shard
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "mydb.mycollection" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])
```

---

## Le Cycle de Vie d'un Chunk

### 1. Création initiale

Lors de l'activation du sharding sur une collection :

```javascript
sh.shardCollection("mydb.orders", { customer_id: 1 })
```

MongoDB crée **un seul chunk initial** contenant toutes les valeurs possibles :
- `min: MinKey`
- `max: MaxKey`
- Assigné au shard contenant le plus d'espace disponible

### 2. Splitting (Division)

À mesure que les données s'accumulent, les chunks dépassent la taille maximale et sont **divisés**.

#### Splitting automatique

Le splitting se produit automatiquement lorsqu'un chunk :
- Dépasse la taille configurée (par défaut 64 Mo)
- Pendant une opération d'insertion ou de mise à jour
- Sur le mongos qui effectue l'opération

```javascript
// Configuration de la taille des chunks (en Mo)
use config
db.settings.updateOne(
   { _id: "chunksize" },
   { $set: { value: 128 } },
   { upsert: true }
)
```

#### Splitting manuel

Pour les besoins spécifiques :

```javascript
// Diviser un chunk à une valeur précise
sh.splitAt("mydb.orders", { customer_id: "customer_5000" })

// Diviser un chunk en son point médian
sh.splitFind("mydb.orders", { customer_id: "customer_7500" })
```

### 3. Migration

Une fois divisés, les chunks peuvent être **migrés** vers d'autres shards par le balancer.

---

## Le Balancer : Principe et Fonctionnement

### Rôle du Balancer

Le **balancer** est un processus en arrière-plan qui :
- S'exécute sur l'un des membres **primary** des config servers
- Surveille en permanence la distribution des chunks
- Migre automatiquement les chunks pour équilibrer la charge
- Fonctionne par **rounds** (cycles d'équilibrage)

### Objectifs du Balancer

1. **Équilibrage de la charge** : Répartir uniformément les données
2. **Respect des zones** : Maintenir les contraintes de localisation (zone sharding)
3. **Performance** : Minimiser l'impact sur les opérations en cours

### Déclenchement du Balancing

Le balancer démarre une migration lorsque :

1. **Différence de chunks** : L'écart entre le shard le plus chargé et le moins chargé dépasse le seuil :

| Nombre total de chunks | Seuil de migration |
|------------------------|-------------------|
| < 20                   | 2 chunks          |
| 20 - 79                | 4 chunks          |
| ≥ 80                   | 8 chunks          |

2. **Zone sharding** : Un chunk viole les contraintes de zone définies

### Algorithme de Balancing

```
Pour chaque collection shardée :
  1. Calculer la distribution actuelle des chunks
  2. Identifier le shard le plus chargé (source)
  3. Identifier le shard le moins chargé (destination)
  4. Si différence > seuil :
     a. Sélectionner un chunk du shard source
     b. Initier la migration vers destination
     c. Attendre la fin de la migration
     d. Recommencer jusqu'à équilibrage
```

---

## Processus de Migration d'un Chunk

### Les Étapes Détaillées

1. **Initiation** (par le balancer)
   - Verrouillage des métadonnées du chunk
   - Notification au shard source

2. **Copie initiale**
   - Le shard destination copie tous les documents du chunk
   - Les opérations d'écriture continuent sur le shard source

3. **Synchronisation (tailing)**
   - Transfert incrémental des modifications survenues pendant la copie
   - Utilise l'oplog du shard source
   - Plusieurs passes si nécessaire

4. **Finalisation**
   - Blocage court des écritures sur le chunk
   - Transfert final des dernières modifications
   - Mise à jour des métadonnées sur les config servers

5. **Commit**
   - Les mongos redirigent les requêtes vers le nouveau shard
   - Le shard source supprime les données du chunk migré

### Durée et Impact

- **Durée moyenne** : De quelques secondes à plusieurs minutes selon la taille
- **Impact performance** :
  - Utilisation CPU et réseau accrue
  - Latence légèrement augmentée pendant le transfert
  - Blocage très court (< 1 seconde) lors du commit

### Exemple de monitoring d'une migration

```javascript
// État actuel du balancer
sh.getBalancerState()  // true si actif

// Voir si une migration est en cours
sh.isBalancerRunning()

// Détails sur les migrations
db.getSiblingDB("config").locks.find({ state: 2 })

// Logs de migration
db.getSiblingDB("config").changelog.find({
  what: "moveChunk.commit"
}).sort({ time: -1 }).limit(5).pretty()
```

---

## Gestion du Balancer

### Contrôle du Balancer

#### Vérifier l'état

```javascript
// État du balancer
sh.getBalancerState()

// Vérifier s'il est en cours d'exécution
sh.isBalancerRunning()

// Statistiques détaillées
db.getSiblingDB("config").settings.findOne({ _id: "balancer" })
```

#### Activer/Désactiver

```javascript
// Désactiver le balancer
sh.stopBalancer()

// Activer le balancer
sh.startBalancer()

// Attendre que le balancer s'arrête
sh.setBalancerState(false)
```

### Fenêtres de Balancing

Pour limiter le balancing à des plages horaires spécifiques :

```javascript
// Définir une fenêtre active (2h-6h du matin)
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

// Supprimer la restriction
db.getSiblingDB("config").settings.updateOne(
   { _id: "balancer" },
   { $unset: { activeWindow: "" } }
)
```

### Désactivation Temporaire (Maintenance)

```javascript
// Désactiver pour une maintenance
sh.stopBalancer()

// Effectuer les opérations de maintenance...
// - Backup
// - Upgrade
// - Ajout de shard

// Réactiver
sh.startBalancer()
```

---

## Stratégies d'Optimisation du Balancing

### 1. Dimensionnement des Chunks

#### Chunks trop petits (< 32 Mo)

**Problème** :
- Trop de chunks → surcharge métadonnées
- Migrations fréquentes → overhead réseau
- Latence accrue pour le balancer

**Solution** :
```javascript
// Augmenter la taille des chunks
use config
db.settings.updateOne(
   { _id: "chunksize" },
   { $set: { value: 128 } },
   { upsert: true }
)
```

#### Chunks trop gros (> 256 Mo)

**Problème** :
- Migrations lentes et coûteuses
- Difficulté à équilibrer finement
- Impact performance plus important

**Solution** :
- Utiliser une shard key plus granulaire
- Réduire la taille des chunks pour les nouvelles collections

### 2. Pré-splitting pour Charges Massives

Lors de l'import de volumes importants, le pré-splitting évite les goulots d'étranglement :

```javascript
// Exemple : Pré-split sur un range de dates (année 2024)
for (var month = 1; month <= 12; month++) {
  var date = new Date(2024, month, 1);
  sh.splitAt("mydb.events", { timestamp: date });
}

// Ou utiliser splitFind pour une distribution automatique
for (var i = 0; i < 100; i++) {
  sh.splitFind("mydb.users", { user_id: i * 1000 });
}
```

### 3. Distribution Initiale des Chunks

Après le pré-splitting, distribuer manuellement :

```javascript
// Obtenir la liste des shards
var shards = db.getSiblingDB("config").shards.find().toArray();

// Déplacer des chunks vers différents shards
sh.moveChunk(
  "mydb.orders",
  { customer_id: "customer_5000" },
  "shard-b"
)

sh.moveChunk(
  "mydb.orders",
  { customer_id: "customer_10000" },
  "shard-c"
)
```

### 4. Parallélisation des Migrations

MongoDB 4.2+ permet des migrations parallèles :

```javascript
// Configurer le nombre maximum de migrations parallèles
db.getSiblingDB("config").settings.updateOne(
   { _id: "balancer" },
   { $set: {
       _secondaryThrottle: false,
       _waitForDelete: false
     }
   },
   { upsert: true }
)
```

**Impact** :
- Plus rapide mais plus gourmand en ressources
- À utiliser lors de maintenance planifiée

---

## Jumbo Chunks : Détection et Résolution

### Qu'est-ce qu'un Jumbo Chunk ?

Un **jumbo chunk** est un chunk qui :
- Dépasse la taille maximale configurée
- **Ne peut pas être divisé** (toutes les valeurs de shard key sont identiques)
- Bloque le balancer (impossible à migrer)

### Détection

```javascript
// Identifier les jumbo chunks
db.getSiblingDB("config").chunks.find({
  ns: "mydb.orders",
  jumbo: true
})

// Ou via sh.status()
sh.status()  // Affiche "jumbo" à côté des chunks concernés
```

### Causes

1. **Shard key de faible cardinalité** : Trop de documents avec la même valeur
2. **Distribution non uniforme** : Quelques valeurs très populaires (hot keys)

### Exemple de Jumbo Chunk

```javascript
// Collection avec une mauvaise shard key
// Shard key : { status: 1 }
// 90% des documents ont status: "active"

// Résultat : Chunk gigantesque et impossible à diviser
{
  "min": { "status": "active" },
  "max": { "status": "inactive" },
  "size": 2048,  // 2 Go !
  "jumbo": true,
  "shard": "shard-a"
}
```

### Solutions

#### Solution 1 : Diviser manuellement (si possible)

```javascript
// Si des sous-valeurs différentes existent
sh.splitAt("mydb.orders", { status: "active", _id: ObjectId("...") })
```

#### Solution 2 : Modifier la shard key (MongoDB 5.0+)

```javascript
// Refactoriser vers une shard key composée
db.adminCommand({
  refineCollectionShardKey: "mydb.orders",
  key: { status: 1, customer_id: 1 }
})
```

#### Solution 3 : Migration forcée (cas extrême)

```javascript
// Désactiver la vérification de taille (attention : risqué)
db.getSiblingDB("config").settings.updateOne(
   { _id: "balancer" },
   { $set: { attemptToBalanceJumboChunks: true } },
   { upsert: true }
)

// Puis forcer la migration
sh.moveChunk("mydb.orders", { status: "active" }, "shard-b")
```

---

## Anti-Patterns et Erreurs Courantes

### ❌ Anti-Pattern 1 : Shard Key Monotone sans Hashed

**Problème** :
```javascript
// Mauvais : _id auto-incrémenté ou timestamp
sh.shardCollection("mydb.orders", { _id: 1 })
sh.shardCollection("mydb.logs", { timestamp: 1 })
```

**Conséquence** :
- Toutes les insertions vont sur le dernier chunk
- Un seul shard reçoit toute la charge d'écriture
- Le balancer migre constamment les chunks pleins

**Solution** :
```javascript
// Utiliser un hashed index
sh.shardCollection("mydb.orders", { _id: "hashed" })

// Ou une shard key composée
sh.shardCollection("mydb.logs", {
  application: 1,
  timestamp: 1
})
```

### ❌ Anti-Pattern 2 : Taille de Chunk Inadaptée

**Problème** :
```javascript
// Chunks trop petits (8 Mo)
db.settings.updateOne(
   { _id: "chunksize" },
   { $set: { value: 8 } }
)
```

**Conséquence** :
- Fragmentation excessive → des milliers de chunks
- Overhead de métadonnées important
- Performance du balancer dégradée

**Solution** :
```javascript
// Taille standard : 64-128 Mo
db.settings.updateOne(
   { _id: "chunksize" },
   { $set: { value: 64 } }
)
```

### ❌ Anti-Pattern 3 : Balancer Actif en Production Intensive

**Problème** :
- Balancer actif pendant les heures de pointe
- Migrations impactant les performances utilisateur

**Solution** :
```javascript
// Définir une fenêtre de maintenance
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

### ❌ Anti-Pattern 4 : Ignorer les Jumbo Chunks

**Problème** :
- Laisser des jumbo chunks s'accumuler
- Distribution déséquilibrée permanente

**Signes** :
```javascript
// Un shard contient 80% des données
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "mydb.orders" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])

// Résultat problématique :
// { "_id": "shard-a", "count": 400 }  // 80%
// { "_id": "shard-b", "count": 50 }   // 10%
// { "_id": "shard-c", "count": 50 }   // 10%
```

**Solution** :
- Diagnostiquer la cause (shard key)
- Refactoriser la shard key si nécessaire
- Monitorer régulièrement la distribution

### ❌ Anti-Pattern 5 : Splitting Manuel Excessif

**Problème** :
```javascript
// Sur-découpage préventif
for (var i = 0; i < 10000; i++) {
  sh.splitAt("mydb.orders", { order_id: i });
}
```

**Conséquence** :
- Overhead de métadonnées massif
- Le balancer passe son temps à gérer des chunks vides
- Performance globale dégradée

**Solution** :
- Pré-splitter raisonnablement (50-200 chunks max au démarrage)
- Laisser le splitting automatique gérer la croissance

---

## Monitoring et Diagnostics

### Commandes Essentielles

```javascript
// Vue globale du cluster
sh.status()

// Distribution des chunks par collection
db.getSiblingDB("config").chunks.aggregate([
  { $group: {
      _id: { ns: "$ns", shard: "$shard" },
      count: { $sum: 1 }
    }
  },
  { $sort: { "_id.ns": 1, "_id.shard": 1 } }
])

// Historique des migrations
db.getSiblingDB("config").changelog.find({
  what: { $in: ["moveChunk.start", "moveChunk.commit", "split"] }
}).sort({ time: -1 }).limit(20).pretty()

// Chunks problématiques (jumbo)
db.getSiblingDB("config").chunks.find({ jumbo: true }).count()

// Taille des données par shard
db.adminCommand({ listShards: 1 })
```

### Métriques Clés à Surveiller

1. **Distribution des chunks** : Écart entre shards < 10%
2. **Jumbo chunks** : 0 idéalement
3. **Fréquence des migrations** : Stable et prévisible
4. **Durée des migrations** : < 5 minutes par chunk
5. **Échecs de migration** : Proche de 0

### Alertes Recommandées

```javascript
// Exemple avec MongoDB Cloud Manager / Ops Manager
// - Alerte si jumbo chunks > 0
// - Alerte si distribution déséquilibrée > 20%
// - Alerte si migration échoue 3 fois de suite
// - Alerte si durée de migration > 10 minutes
```

---

## Bonnes Pratiques de Production

### 1. Planification du Balancing

- **Fenêtre de maintenance** : Définir des heures creuses
- **Désactivation temporaire** : Pendant les pics prévisibles (Black Friday, etc.)
- **Monitoring proactif** : Surveillance continue de la distribution

### 2. Dimensionnement Préventif

- **Taille de chunk adaptée** : 64-128 Mo selon le cas d'usage
- **Pré-splitting intelligent** : Pour les imports massifs
- **Shard key réfléchie** : Éviter les monotones et les faibles cardinalités

### 3. Réponse aux Incidents

```javascript
// Procédure d'urgence : Balancer impacte la prod

// 1. Arrêter immédiatement le balancer
sh.stopBalancer()

// 2. Attendre la fin des migrations en cours
while (sh.isBalancerRunning()) {
  sleep(1000);
  print("Attente arrêt balancer...");
}

// 3. Investiguer la cause
db.getSiblingDB("config").changelog.find({
  time: { $gte: new Date(Date.now() - 3600000) }  // Dernière heure
}).pretty()

// 4. Réactiver après correction
sh.startBalancer()
```

### 4. Documentation et Runbooks

Maintenir une documentation à jour :
- Topologie du cluster (nombre de shards, capacités)
- Shard keys de chaque collection
- Fenêtres de balancing configurées
- Procédures d'escalade en cas de problème

---

## Évolutions et Nouveautés

### MongoDB 4.4+
- **Amélioration du balancer** : Migrations plus rapides et efficaces
- **Jumbo chunks intelligents** : Meilleure détection et gestion

### MongoDB 5.0+
- **Resharding** : Modification de la shard key en ligne
- **Refine shard key** : Ajout de champs à une shard key existante

### MongoDB 6.0+
- **Balancer parallèle amélioré** : Jusqu'à 10 migrations simultanées
- **Métriques étendues** : Monitoring plus fin des migrations

### MongoDB 7.0+
- **Balancer auto-tuning** : Ajustement dynamique selon la charge
- **Zone sharding amélioré** : Configuration simplifiée

---

## Conclusion

Les chunks et le balancer sont au cœur du fonctionnement d'un cluster shardé MongoDB. Une compréhension approfondie de ces mécanismes est essentielle pour :

- ✅ **Maintenir des performances optimales** en production
- ✅ **Anticiper et résoudre** les problèmes de distribution
- ✅ **Dimensionner correctement** les ressources du cluster
- ✅ **Éviter les pièges** classiques (jumbo chunks, shard keys monotones)

Le balancing est un processus majoritairement automatique, mais nécessite une **supervision active** et des **interventions occasionnelles** pour garantir le bon fonctionnement du cluster à long terme.

---

## Références

- [MongoDB Documentation - Sharding](https://docs.mongodb.com/manual/sharding/)
- [MongoDB Documentation - Balancer](https://docs.mongodb.com/manual/core/sharding-balancer-administration/)
- [MongoDB Best Practices - Chunks](https://docs.mongodb.com/manual/core/sharding-data-partitioning/)
- [MongoDB University - M103: Basic Cluster Administration](https://university.mongodb.com/)

---


⏭️ [Déploiement d'un cluster shardé](/10-sharding/06-deploiement-cluster-sharde.md)
