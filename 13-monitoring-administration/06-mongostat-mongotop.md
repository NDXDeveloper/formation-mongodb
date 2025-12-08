🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 13.6 mongostat et mongotop

## Introduction

**mongostat** et **mongotop** sont les deux utilitaires de monitoring en temps réel essentiels pour les SRE et administrateurs MongoDB. Contrairement aux outils de monitoring à long terme comme Prometheus ou aux outils d'analyse post-mortem comme le profiler, ces utilitaires fournissent une **vue immédiate et continue** de l'état du serveur, permettant un diagnostic rapide et une intervention en temps réel.

Ces outils sont particulièrement critiques pour :
- **Diagnostic immédiat** lors d'incidents de performance
- **Validation** des changements de configuration ou de code
- **Détection proactive** de patterns anormaux
- **Baseline** des performances en conditions normales
- **Investigation** de problèmes transitoires difficiles à capturer

Dans l'arsenal d'un SRE, mongostat et mongotop sont souvent les **premiers outils** invoqués lors d'une alerte de performance, avant même d'examiner les logs ou les dashboards.

---

## mongostat - Monitoring Global du Serveur

### Architecture et Fonctionnement

mongostat collecte et affiche les statistiques du serveur en exécutant périodiquement la commande `serverStatus()` et en calculant les deltas entre les snapshots successifs.

```
┌────────────────────────────────────────────────────────────────┐
│                        mongostat                               │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │   Snapshot   │────▶│   Calculate  │────▶│   Display    │    │
│  │ serverStatus │     │    Deltas    │     │   Metrics    │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│         │                                          │           │
│         │                                          │           │
│         ▼                                          ▼           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │            MongoDB Server (mongod/mongos)                │  │
│  │  • serverStatus()                                        │  │
│  │  • replSetGetStatus()                                    │  │
│  │  • shardingStatistics()                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

### Installation et Prérequis

mongostat est inclus dans le package MongoDB Database Tools :

```bash
# Vérifier l'installation
mongostat --version

# Output attendu
mongostat version: 100.9.4
git version: e22e7e2bc564646d65b2959c98fa01afd004f935
Go version: go1.20.12
os: linux
arch: amd64
compiler: gc
```

**Permissions requises** :
- Lecture sur `admin.system.version`
- Action `serverStatus` sur le cluster
- Pour replica sets : `replSetGetStatus`
- Pour sharding : `shardingState`

```javascript
// Créer un utilisateur dédié au monitoring
db.getSiblingDB('admin').createUser({
  user: "monitoring",
  pwd: "securePassword",
  roles: [
    { role: "clusterMonitor", db: "admin" },
    { role: "read", db: "local" }
  ]
})
```

### Syntaxe de Base

```bash
mongostat [options] [polling-interval]
```

**Exemple simple** :
```bash
# Rafraîchissement toutes les secondes (défaut)
mongostat --uri="mongodb://monitoring:password@localhost:27017/?authSource=admin"

# Rafraîchissement toutes les 5 secondes
mongostat --uri="mongodb://localhost:27017" 5

# Nombre limité d'itérations (10 snapshots)
mongostat --uri="mongodb://localhost:27017" --rowcount=10
```

---

## Métriques mongostat - Référence Complète

### Vue d'Ensemble des Colonnes

Sortie standard de mongostat :

```
insert query update delete getmore command dirty used flushes vsize  res qrw arw net_in net_out conn                time
   *45   *89    *23     *5       8    15|2  3.2% 42.1%       0 1.6G 1.1G 0|0 1|0  15.2k   38.4k   87 Dec 08 16:45:23.456
    52    95     28      7      12    18|3  3.5% 43.2%       0 1.6G 1.1G 0|0 1|0  16.8k   42.1k   89 Dec 08 16:45:24.456
```

### 1. Opérations par Seconde

#### insert, query, update, delete

**Description** : Nombre d'opérations par seconde pour chaque type.

**Signification** :
- `*` préfixant la valeur : au moins une opération a pris plus de 1ms
- Valeur simple : nombre d'opérations/seconde

**Interprétation** :

| Valeur | Niveau | Action |
|--------|--------|--------|
| < 100 | Normal | Aucune |
| 100-1000 | Modéré | Surveiller |
| 1000-5000 | Élevé | Vérifier indexes |
| > 5000 | Très élevé | Investigation requise |

**Patterns d'alerte** :

```bash
# Spike soudain d'insertions
insert: 45 → 5234 → 5678 → 4892
→ Possible batch insert ou import en cours

# Queries élevées continues avec *
query: *8234 *8456 *8123 *8345
→ Requêtes lentes, manque d'index probable

# Deletes élevés
delete: 2345 2456 2389 2412
→ Opération de purge, vérifier l'impact
```

**Exemple d'analyse** :
```bash
# Capturer 60 secondes de statistiques
mongostat --uri="$MONGO_URI" --rowcount=60 > mongostat_output.txt

# Analyser les pics
awk '{if ($2 > 1000) print $21, $2}' mongostat_output.txt
```

#### getmore

**Description** : Nombre d'opérations getMore par seconde (continuations de curseurs).

**Signification** : Résultats de requêtes retournés par batch. Valeur élevée = beaucoup de grands curseurs actifs.

**Seuils** :
- < 50 : Normal
- 50-200 : Surveillance
- > 200 : Attention - possibles curseurs non fermés ou scans lourds

**Diagnostic** :
```javascript
// Identifier les curseurs ouverts
db.getSiblingDB('admin').aggregate([
  { $currentOp: { idleCursors: true } },
  { $match: { type: "idleCursor" } },
  { $group: {
      _id: "$ns",
      count: { $sum: 1 },
      totalTime: { $sum: "$microsecs_running" }
  }},
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

#### command

**Description** : Nombre de commandes par seconde, format `read|write`.

**Exemples de commandes** :
- Read : find, aggregate, count, distinct
- Write : insert, update, delete, findAndModify
- Admin : replSetGetStatus, serverStatus, ping

**Analyse** :
```
command: 45|12
→ 45 commandes de lecture/sec, 12 d'écriture/sec

command: 2345|1234
→ Charge très élevée, vérifier les sources
```

### 2. Métriques Cache WiredTiger

#### dirty

**Description** : Pourcentage du cache WiredTiger contenant des données modifiées non encore écrites sur disque.

**Seuils critiques** :

| Valeur | État | Impact | Action |
|--------|------|--------|--------|
| < 5% | Normal | Aucun | - |
| 5-15% | Surveillé | Faible | Monitoring |
| 15-30% | Attention | Modéré | Vérifier I/O |
| 30-50% | Critique | Élevé | Action immédiate |
| > 50% | Urgence | Critique | Intervention |

**Signification** :
- Cache dirty élevé → Écritures plus rapides que la capacité du disque à absorber
- Peut causer des checkpoints lents
- Risque de ralentissement brutal quand limite atteinte

**Pattern problématique** :
```
dirty: 3.2% → 8.5% → 15.2% → 28.4% → 42.1% → 51.3%
→ Progression constante = surcharge d'écritures
```

**Actions correctives** :

```yaml
# 1. Vérifier les I/O disque
iostat -x 1 10

# 2. Analyser les écritures
db.serverStatus().wiredTiger.transaction

# 3. Configuration WiredTiger (mongod.conf)
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # Augmenter si possible
    collectionConfig:
      blockCompressor: snappy  # Vérifier compression
    indexConfig:
      prefixCompression: true

# 4. Ajuster le checkpoint interval
storage:
  syncPeriodSecs: 60  # Défaut: 60s, augmenter si nécessaire
```

#### used

**Description** : Pourcentage du cache WiredTiger actuellement utilisé.

**Seuils** :

| Valeur | État | Action |
|--------|------|--------|
| < 60% | Normal | Aucune |
| 60-80% | Optimal | Surveiller |
| 80-95% | Haute utilisation | Planifier augmentation |
| > 95% | Saturation | Action requise |

**Comportement normal** :
```
used: 42.1% → 45.3% → 48.7% → 52.1%
→ Croissance progressive = working set grandit

used: 78.2% → 79.1% → 78.5% → 79.3%
→ Stabilisation haute = working set correspond au cache
```

**Comportement problématique** :
```
used: 95.2% → 96.8% → 97.1% → 95.4% → 96.9%
→ Oscillation haute = cache trop petit, evictions fréquentes
```

**Calcul du cache optimal** :
```javascript
// Obtenir le working set
var stats = db.serverStatus();
var workingSet = stats.wiredTiger.cache["bytes currently in the cache"];
var cacheSize = stats.wiredTiger.cache["maximum bytes configured"];

print("Working Set: " + (workingSet / 1024 / 1024 / 1024).toFixed(2) + " GB");
print("Cache Size: " + (cacheSize / 1024 / 1024 / 1024).toFixed(2) + " GB");
print("Utilization: " + ((workingSet / cacheSize) * 100).toFixed(2) + "%");

// Recommandation
if (workingSet / cacheSize > 0.8) {
  var recommended = (workingSet / 0.7) / 1024 / 1024 / 1024;
  print("Recommended cache: " + recommended.toFixed(2) + " GB");
}
```

### 3. Métriques I/O et Mémoire

#### flushes

**Description** : Nombre de checkpoints WiredTiger par seconde.

**Valeurs normales** :
- 0 : Aucun checkpoint dans l'intervalle (normal si < 60s)
- 1 : Un checkpoint (attendu toutes les 60s par défaut)
- > 1 : Multiple checkpoints (anormal, investigation requise)

**Configuration checkpoint** :
```javascript
// Vérifier la configuration
db.serverStatus().wiredTiger.transaction

// Sortie exemple
{
  "transaction checkpoint currently running": false,
  "transaction checkpoint generation": 1234,
  "transaction checkpoint max time (msecs)": 127,
  "transaction checkpoint min time (msecs)": 45,
  "transaction checkpoint most recent time (msecs)": 87
}
```

**Checkpoint trop lent** :
```
Si "most recent time" > 30000 ms (30s) :
→ I/O disque saturé
→ Cache dirty trop élevé
→ Volume de données à écrire trop important
```

#### vsize (Virtual Memory Size)

**Description** : Taille totale de la mémoire virtuelle utilisée par MongoDB.

**Interprétation** :
- Inclut : mémoire mappée, cache, heap, stack
- Croissance normale : progressive avec le dataset
- Croissance anormale : fuite mémoire potentielle

**Monitoring** :
```bash
# Suivre l'évolution
mongostat --uri="$MONGO_URI" --rowcount=60 | awk '{print $21, $10}'

# Comparer avec la mémoire système
free -h
```

#### res (Resident Memory)

**Description** : Mémoire résidente (RAM physique utilisée).

**Calcul optimal** :
```
Optimal RES = Working Set + Overhead
- Working Set : données fréquemment accédées
- Overhead : ~1-2 GB (structures internes, connexions)
```

**Seuils d'alerte** :

| Condition | Signification | Action |
|-----------|--------------|--------|
| res > 90% RAM totale | Risque swap | Ajouter RAM ou réduire cache |
| res croît constamment | Fuite mémoire possible | Investigation |
| res >> cache configuré | Overhead élevé | Analyser connexions |

**Exemple d'analyse** :
```bash
#!/bin/bash
# memory_analysis.sh

TOTAL_RAM=$(free -g | awk '/Mem:/ {print $2}')
MONGO_RES=$(mongostat --uri="$MONGO_URI" --rowcount=1 | tail -1 | awk '{print $11}' | sed 's/G//')

USAGE_PERCENT=$(echo "scale=2; ($MONGO_RES / $TOTAL_RAM) * 100" | bc)

echo "Total RAM: ${TOTAL_RAM}GB"
echo "MongoDB RES: ${MONGO_RES}GB"
echo "Usage: ${USAGE_PERCENT}%"

if (( $(echo "$USAGE_PERCENT > 85" | bc -l) )); then
  echo "WARNING: High memory usage!"
fi
```

### 4. Métriques de Queue et Concurrence

#### qrw (Queue Read|Write)

**Description** : Nombre d'opérations en attente de traitement (queue), format `read|write`.

**Signification** : Indique la saturation du système - les requêtes arrivent plus vite qu'elles ne peuvent être traitées.

**Seuils critiques** :

| Valeur | État | Impact Utilisateur | Action |
|--------|------|-------------------|--------|
| 0\|0 | Normal | Aucun | - |
| < 10\|10 | Faible | Latence +50ms | Surveiller |
| 10-50\|10-50 | Modéré | Latence +200ms | Investigation |
| > 50\|> 50 | Critique | Latence +1s+ | Action immédiate |

**Patterns d'analyse** :

```bash
# Pattern 1 : Queue lectures élevée
qrw: 45|2 48|3 52|2 47|1
→ Goulot en lecture (queries lentes, index manquants)

# Pattern 2 : Queue écritures élevée
qrw: 2|34 1|38 3|42 2|39
→ Goulot en écriture (I/O disque lent, write concern élevé)

# Pattern 3 : Queue mixte élevée
qrw: 67|45 72|48 68|52 71|47
→ Surcharge générale du serveur
```

**Diagnostic des queues élevées** :
```javascript
// 1. Identifier les opérations en attente
db.currentOp({
  "$or": [
    { "waitingForLock": true },
    { "locks": { "$exists": true } }
  ]
}).inprog.forEach(function(op) {
  if (op.secs_running > 5) {
    printjson(op);
  }
})

// 2. Analyser les locks
db.serverStatus().locks

// 3. Vérifier les requêtes lentes en cours
db.currentOp({
  "active": true,
  "secs_running": { "$gt": 3 },
  "op": { "$in": ["query", "getmore"] }
})
```

#### arw (Active Read|Write)

**Description** : Nombre d'opérations actuellement en cours d'exécution, format `read|write`.

**Capacité théorique** :
- Nombre de CPU cores disponibles
- MongoDB utilise un pool de threads (défaut : 128)

**Interprétation** :

```bash
# Système 8 cores
arw: 4|2  → Utilisation normale (< cores)
arw: 12|8 → Utilisation élevée (> cores, mais normal si temporaire)
arw: 45|32 → Surcharge (nombreuses requêtes lentes concurrentes)

# Ratio queue/active élevé
qrw: 89|45, arw: 8|4
→ Problème : queue grandit plus vite que le traitement
→ Cause probable : requêtes lentes monopolisant les threads
```

**Configuration du pool de threads** :
```yaml
# mongod.conf
net:
  serviceExecutor: synchronous  # ou adaptive (défaut)

# Note : adaptive ajuste dynamiquement le pool
```

### 5. Métriques Réseau

#### net_in, net_out

**Description** : Trafic réseau entrant/sortant en octets par seconde.

**Valeurs typiques** :

| Type d'Application | net_in | net_out |
|-------------------|--------|---------|
| OLTP (transactionnel) | 10-100 KB/s | 50-500 KB/s |
| Analytics (reporting) | 1-10 KB/s | 1-50 MB/s |
| Bulk operations | 1-100 MB/s | 100 KB-10 MB/s |

**Patterns d'analyse** :

```bash
# Pattern 1 : net_out élevé
net_out: 45.2 MB/s
→ Requêtes retournant beaucoup de données
→ Vérifier projections et limites

# Pattern 2 : net_in élevé
net_in: 23.4 MB/s
→ Bulk inserts ou imports en cours
→ Normal si planifié

# Pattern 3 : Spike soudain
net_out: 125 KB/s → 45.2 MB/s → 47.8 MB/s
→ Requête retournant dataset complet (missing limit ?)
```

**Analyse du trafic** :
```javascript
// Identifier les opérations volumineuses
db.currentOp({
  "active": true,
  "responseLength": { "$gt": 10000000 }  // > 10 MB
}).inprog.forEach(function(op) {
  print("Client: " + op.client);
  print("Response size: " + (op.responseLength / 1024 / 1024).toFixed(2) + " MB");
  printjson(op.command);
  print("---");
})
```

### 6. Métriques de Connexion

#### conn

**Description** : Nombre total de connexions actives au serveur.

**Limites** :
```javascript
// Vérifier les limites
db.serverStatus().connections

// Sortie exemple
{
  "current": 87,
  "available": 51113,
  "totalCreated": 2345
}
```

**Seuils** :

| Condition | État | Action |
|-----------|------|--------|
| < 100 | Normal | - |
| 100-500 | Modéré | Surveiller |
| 500-1000 | Élevé | Vérifier connection pooling |
| > 1000 | Critique | Investigation requise |
| Proche de maxIncomingConnections | Urgence | Augmenter limite ou résoudre fuite |

**Diagnostic des connexions** :
```javascript
// Par IP source
db.getSiblingDB('admin').aggregate([
  { $currentOp: { allUsers: true } },
  { $group: {
      _id: "$client",
      count: { $sum: 1 }
  }},
  { $sort: { count: -1 } },
  { $limit: 10 }
])

// Par type d'opération
db.getSiblingDB('admin').aggregate([
  { $currentOp: { allUsers: true } },
  { $group: {
      _id: "$op",
      count: { $sum: 1 }
  }},
  { $sort: { count: -1 } }
])
```

**Configuration** :
```yaml
# mongod.conf
net:
  maxIncomingConnections: 2000  # Augmenter si nécessaire

# Côté application (exemple Node.js)
const client = new MongoClient(uri, {
  maxPoolSize: 100,
  minPoolSize: 10,
  maxIdleTimeMS: 30000
});
```

---

## Options Avancées de mongostat

### Output Personnalisé

```bash
# Colonnes spécifiques (-O)
mongostat --uri="$MONGO_URI" \
  -O='host,insert,query,update,delete,qrw,arw,dirty,used,conn,time'

# Output JSON pour parsing
mongostat --uri="$MONGO_URI" -o=json --rowcount=10

# Output exemple JSON
[{
  "host": "localhost:27017",
  "insert": 45,
  "query": 89,
  "update": 23,
  "delete": 5,
  "qr": 0,
  "qw": 0,
  "ar": 1,
  "aw": 0,
  "dirty": 3.2,
  "used": 42.1,
  "conn": 87,
  "time": "2024-12-08T16:45:23Z"
}]
```

### Monitoring de Replica Set

```bash
# Découverte automatique des membres
mongostat --uri="mongodb://rs0-primary:27017,rs0-secondary1:27017,rs0-secondary2:27017/?replicaSet=rs0" \
  --discover

# Output
               host insert query update delete getmore command dirty  used flushes vsize  res qrw arw net_in net_out conn repl                time
rs0-primary:27017     45    89     23      5       8    15|2  3.2% 42.1%       0 1.6G 1.1G 0|0 1|0  15.2k   38.4k   87  PRI Dec 08 16:45:23
rs0-sec1:27017         0    23      0      0      12     5|0  2.1% 38.7%       0 1.5G 1.0G 0|0 0|0   8.4k   12.1k   45  SEC Dec 08 16:45:23
rs0-sec2:27017         0    19      0      0       8     4|0  1.8% 37.2%       0 1.5G 1.0G 0|0 0|0   7.8k   11.3k   42  SEC Dec 08 16:45:23
```

**Colonne `repl`** :
- PRI : Primary
- SEC : Secondary
- REC : Recovering
- ARB : Arbiter
- UNK : Unknown

**Analyse comparative** :
```bash
# Script d'analyse des membres
#!/bin/bash

mongostat --uri="$RS_URI" --discover --rowcount=1 -o=json | \
  jq -r '.[] |
    [.host, .repl, .qr, .qw, .ar, .aw, .conn] |
    @tsv' | \
  column -t
```

### Monitoring de Cluster Sharded

```bash
# Connecter au mongos
mongostat --uri="mongodb://mongos:27017" --discover

# Output inclut les shards
               host insert query update delete getmore command dirty  used flushes vsize  res qrw arw net_in net_out conn                time
mongos1:27017        52    95     28      7      12    18|3     -     -        -    -    -  0|0 1|0  16.8k   42.1k   89 Dec 08 16:45:24
shard01:27017        23    42     12      3       5     8|1  3.1% 41.5%       0 1.6G 1.1G 0|0 1|0   7.2k   18.3k   34 Dec 08 16:45:24
shard02:27017        29    53     16      4       7    10|2  3.8% 43.2%       0 1.6G 1.2G 0|0 1|0   9.6k   23.9k   55 Dec 08 16:45:24
```

**Note** : mongos ne gère pas de cache, donc dirty/used sont vides.

### Mode Interactif

```bash
# Démarrer en mode interactif
mongostat --uri="$MONGO_URI" --interactive

# Commandes disponibles dans le mode interactif :
# - Espace : Pause/Resume
# - q : Quit
# - + : Augmenter refresh rate
# - - : Diminuer refresh rate
```

---

## mongotop - Monitoring par Collection

### Architecture et Fonctionnement

mongotop mesure le temps passé en lecture/écriture par namespace (base.collection), permettant d'identifier rapidement les "hot collections".

```
┌────────────────────────────────────────────────────────────┐
│                       mongotop                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  ┌──────────────┐     ┌──────────────┐     ┌────────────┐  │
│  │   Collect    │────▶│  Aggregate   │────▶│  Display   │  │
│  │   top()      │     │  by NS       │     │   Report   │  │
│  └──────────────┘     └──────────────┘     └────────────┘  │
│         │                                                  │
│         ▼                                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │            MongoDB Server                            │  │
│  │  • top command                                       │  │
│  │  • Collection-level stats                            │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
```

### Syntaxe et Options

```bash
# Syntaxe de base
mongotop [options] [polling-interval]

# Exemples
mongotop --uri="mongodb://localhost:27017"  # Défaut : 1 seconde
mongotop --uri="mongodb://localhost:27017" 5  # Toutes les 5 secondes
mongotop --uri="mongodb://localhost:27017" --rowcount=20  # 20 snapshots
```

### Interprétation de la Sortie

```bash
# Sortie standard
                      ns    total    read    write    2024-12-08T16:45:23Z
         mydb.orders    4567ms   1234ms    3333ms
          mydb.users     892ms    856ms      36ms
       mydb.products     234ms    234ms       0ms
         mydb.logs       89ms     12ms      77ms
  mydb.system.profile     45ms     45ms       0ms
```

**Colonnes** :
- **ns** : Namespace (database.collection)
- **total** : Temps total en millisecondes
- **read** : Temps passé en lectures
- **write** : Temps passé en écritures
- **timestamp** : Horodatage

**Signification** :
```
mydb.orders: total 4567ms, read 1234ms, write 3333ms
→ Sur l'intervalle (1s), MongoDB a passé 4.567s en opérations sur cette collection
→ 1.234s en lectures, 3.333s en écritures
→ Ratio write/read élevé = collection à forte écriture
```

### Analyse des Patterns

#### Pattern 1 : Collection Lecture-Intensive

```
mydb.analytics    12.3s    12.1s    0.2s
→ Presque exclusivement des lectures
→ Candidat pour :
  - Index de couverture
  - Read preference sur secondaries
  - Caching applicatif
```

**Actions** :
```javascript
// 1. Vérifier les requêtes
db.system.profile.find({ ns: "mydb.analytics" }).sort({ ts: -1 }).limit(5)

// 2. Analyser les index
db.analytics.getIndexes()

// 3. Vérifier si covered queries possible
db.analytics.find(
  { category: "electronics" },
  { _id: 0, name: 1, price: 1 }
).explain("executionStats")
```

#### Pattern 2 : Collection Écriture-Intensive

```
mydb.logs    8.7s    0.3s    8.4s
→ Presque exclusivement des écritures
→ Candidat pour :
  - Capped collection (si logs)
  - Bulk inserts
  - Write concern ajusté
  - Sharding si volume élevé
```

**Actions** :
```javascript
// 1. Convertir en capped collection (si applicable)
db.runCommand({
  convertToCapped: "logs",
  size: 1073741824  // 1 GB
})

// 2. Utiliser bulk operations
var bulk = db.logs.initializeUnorderedBulkOp();
for (var i = 0; i < 1000; i++) {
  bulk.insert({ timestamp: new Date(), message: "log" + i });
}
bulk.execute({ w: 0 });  // Pas d'attente de confirmation

// 3. Vérifier write concern
db.runCommand({ getLastError: 1 })
```

#### Pattern 3 : Collection Équilibrée mais Volumineuse

```
mydb.orders    15.2s    7.8s    7.4s
→ Volume très élevé (> 15s sur 1s d'intervalle)
→ Multiple operations concurrentes
→ Probable hot collection
```

**Actions** :
```javascript
// 1. Identifier les requêtes concurrentes
db.currentOp({
  "active": true,
  "ns": "mydb.orders"
}).inprog.length

// 2. Vérifier si sharding nécessaire
db.orders.stats().count / db.orders.stats().storageSize

// 3. Analyser la distribution de la charge
db.orders.aggregate([
  { $sample: { size: 10000 } },
  { $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
      count: { $sum: 1 }
  }},
  { $sort: { _id: 1 } }
])
```

### Options Avancées

#### Output JSON

```bash
mongotop --uri="mongodb://localhost:27017" -o=json

# Output
{
  "totals": [
    {
      "ns": "mydb.orders",
      "total": { "time": 4567, "count": 1234 },
      "read": { "time": 1234, "count": 456 },
      "write": { "time": 3333, "count": 778 }
    }
  ],
  "ok": 1
}
```

#### Filtrage et Tri

```bash
# Afficher uniquement certaines bases
mongotop --uri="mongodb://localhost:27017" --json | \
  jq '.totals[] | select(.ns | startswith("mydb.")) | {ns: .ns, total: .total.time}'

# Top 5 collections par temps total
mongotop --uri="mongodb://localhost:27017" --rowcount=1 --json | \
  jq '.totals | sort_by(.total.time) | reverse | .[0:5]'
```

#### Monitoring de Locks

```bash
# Inclure les informations de locks
mongotop --uri="mongodb://localhost:27017" --locks

# Output additionnel
Lock type    time
global read  123ms
global write  45ms
database      89ms
collection    234ms
```

---

## Cas d'Usage Avancés

### 1. Diagnostic d'Incident en Temps Réel

**Scénario** : Alerte de latence élevée en production.

```bash
#!/bin/bash
# incident_diagnosis.sh

MONGO_URI="mongodb://monitoring:password@prod.example.com:27017/?authSource=admin"
OUTPUT_DIR="/tmp/incident_$(date +%Y%m%d_%H%M%S)"
mkdir -p $OUTPUT_DIR

echo "=== INCIDENT DIAGNOSIS STARTED ==="

# 1. Capturer mongostat pendant 2 minutes
echo "Capturing mongostat..."
mongostat --uri="$MONGO_URI" --rowcount=120 > $OUTPUT_DIR/mongostat.txt &
MONGOSTAT_PID=$!

# 2. Capturer mongotop pendant 2 minutes
echo "Capturing mongotop..."
mongotop --uri="$MONGO_URI" --rowcount=120 > $OUTPUT_DIR/mongotop.txt &
MONGOTOP_PID=$!

# 3. Snapshot des opérations en cours
echo "Capturing currentOp..."
mongo "$MONGO_URI" --quiet --eval "
  printjson(db.currentOp({
    'active': true,
    'secs_running': { '\$gt': 1 }
  }))
" > $OUTPUT_DIR/currentop.json

# 4. Attendre la fin des captures
wait $MONGOSTAT_PID
wait $MONGOTOP_PID

# 5. Analyse rapide
echo ""
echo "=== QUICK ANALYSIS ==="

# Queues moyennes
echo "Average queues:"
awk 'NR>1 {
  split($12, qrw, "|");
  qr+=$qrw[0]; qw+=$qrw[1]; count++
}
END {
  print "  qr: " qr/count;
  print "  qw: " qw/count
}' $OUTPUT_DIR/mongostat.txt

# Cache dirty moyen
echo "Average dirty cache:"
awk 'NR>1 {
  gsub(/%/, "", $7);
  dirty+=$7; count++
}
END {
  print "  " dirty/count "%"
}' $OUTPUT_DIR/mongostat.txt

# Top 3 collections
echo "Top 3 hot collections:"
awk 'NR>1 {
  ns=$1; total=$2;
  if (ns != "") totals[ns]+=total
}
END {
  for (ns in totals) print totals[ns], ns
}' $OUTPUT_DIR/mongotop.txt | sort -rn | head -3

echo ""
echo "=== DATA SAVED TO $OUTPUT_DIR ==="
```

### 2. Baseline de Performance

**Objectif** : Établir les métriques normales pour comparaison future.

```bash
#!/bin/bash
# performance_baseline.sh

MONGO_URI="mongodb://monitoring:password@prod.example.com:27017/?authSource=admin"
BASELINE_FILE="/opt/monitoring/mongodb_baseline.json"

# Capturer 1 heure de métriques (échantillonnage toutes les 30s)
mongostat --uri="$MONGO_URI" --rowcount=120 30 -o=json > /tmp/baseline_raw.json

# Calculer les moyennes
jq '[.[] | {
  insert: .insert,
  query: .query,
  update: .update,
  delete: .delete,
  qr: .qr,
  qw: .qw,
  ar: .ar,
  aw: .aw,
  dirty: .dirty,
  used: .used,
  conn: .conn
}] | {
  insert_avg: ([.[].insert] | add / length),
  query_avg: ([.[].query] | add / length),
  update_avg: ([.[].update] | add / length),
  delete_avg: ([.[].delete] | add / length),
  qr_avg: ([.[].qr] | add / length),
  qw_avg: ([.[].qw] | add / length),
  dirty_avg: ([.[].dirty] | add / length),
  used_avg: ([.[].used] | add / length),
  conn_avg: ([.[].conn] | add / length),
  timestamp: now
}' /tmp/baseline_raw.json > $BASELINE_FILE

echo "Baseline saved to $BASELINE_FILE"
cat $BASELINE_FILE
```

### 3. Détection d'Anomalies

```python
#!/usr/bin/env python3
# anomaly_detector.py

import json
import subprocess
import statistics
from datetime import datetime

class MongoStatAnomalyDetector:
    def __init__(self, baseline_file):
        with open(baseline_file) as f:
            self.baseline = json.load(f)

    def get_current_stats(self, uri):
        """Capture current mongostat"""
        result = subprocess.run(
            ['mongostat', '--uri', uri, '--rowcount', '10', '-o', 'json'],
            capture_output=True,
            text=True
        )
        return json.loads(result.stdout)

    def detect_anomalies(self, current_stats):
        """Compare current with baseline"""
        anomalies = []

        # Calculer moyennes courantes
        current_avg = {
            'query': statistics.mean([s['query'] for s in current_stats]),
            'qr': statistics.mean([s['qr'] for s in current_stats]),
            'qw': statistics.mean([s['qw'] for s in current_stats]),
            'dirty': statistics.mean([s['dirty'] for s in current_stats]),
        }

        # Détecter anomalies (> 3x baseline)
        for metric, value in current_avg.items():
            baseline_value = self.baseline.get(f'{metric}_avg', 0)
            if value > baseline_value * 3:
                anomalies.append({
                    'metric': metric,
                    'current': value,
                    'baseline': baseline_value,
                    'ratio': value / baseline_value if baseline_value > 0 else float('inf'),
                    'severity': 'CRITICAL' if value > baseline_value * 5 else 'WARNING'
                })

        return anomalies

    def report(self, anomalies):
        """Generate report"""
        if not anomalies:
            print("✓ No anomalies detected")
            return

        print(f"\n⚠️  DETECTED {len(anomalies)} ANOMALIES\n")
        for a in sorted(anomalies, key=lambda x: x['ratio'], reverse=True):
            print(f"[{a['severity']}] {a['metric']}")
            print(f"  Current: {a['current']:.2f}")
            print(f"  Baseline: {a['baseline']:.2f}")
            print(f"  Ratio: {a['ratio']:.2f}x")
            print()

if __name__ == '__main__':
    import sys

    if len(sys.argv) < 3:
        print("Usage: anomaly_detector.py <baseline_file> <mongo_uri>")
        sys.exit(1)

    detector = MongoStatAnomalyDetector(sys.argv[1])
    stats = detector.get_current_stats(sys.argv[2])
    anomalies = detector.detect_anomalies(stats)
    detector.report(anomalies)
```

### 4. Monitoring Continu avec Alerting

```bash
#!/bin/bash
# continuous_monitor.sh

MONGO_URI="mongodb://monitoring:password@prod.example.com:27017/?authSource=admin"
ALERT_WEBHOOK="https://hooks.slack.com/services/XXX"

# Seuils d'alerte
THRESHOLD_QR=10
THRESHOLD_QW=10
THRESHOLD_DIRTY=20
THRESHOLD_CONN=500

while true; do
  # Capturer un snapshot
  STATS=$(mongostat --uri="$MONGO_URI" --rowcount=1 -o=json | jq '.[0]')

  # Extraire métriques
  QR=$(echo $STATS | jq -r '.qr')
  QW=$(echo $STATS | jq -r '.qw')
  DIRTY=$(echo $STATS | jq -r '.dirty')
  CONN=$(echo $STATS | jq -r '.conn')

  # Vérifier seuils
  ALERTS=""

  if (( $(echo "$QR > $THRESHOLD_QR" | bc -l) )); then
    ALERTS="${ALERTS}Read queue: $QR (threshold: $THRESHOLD_QR)\n"
  fi

  if (( $(echo "$QW > $THRESHOLD_QW" | bc -l) )); then
    ALERTS="${ALERTS}Write queue: $QW (threshold: $THRESHOLD_QW)\n"
  fi

  if (( $(echo "$DIRTY > $THRESHOLD_DIRTY" | bc -l) )); then
    ALERTS="${ALERTS}Dirty cache: $DIRTY% (threshold: $THRESHOLD_DIRTY%)\n"
  fi

  if [ "$CONN" -gt "$THRESHOLD_CONN" ]; then
    ALERTS="${ALERTS}Connections: $CONN (threshold: $THRESHOLD_CONN)\n"
  fi

  # Envoyer alertes si nécessaire
  if [ ! -z "$ALERTS" ]; then
    MESSAGE="MongoDB Alert - $(date)\n$ALERTS"
    curl -X POST $ALERT_WEBHOOK \
      -H 'Content-Type: application/json' \
      -d "{\"text\": \"$MESSAGE\"}"
  fi

  # Attendre 30 secondes
  sleep 30
done
```

### 5. Corrélation mongostat + mongotop

```bash
#!/bin/bash
# correlate_stats.sh

MONGO_URI="mongodb://monitoring:password@prod.example.com:27017/?authSource=admin"
DURATION=60  # 1 minute

echo "Collecting data for $DURATION seconds..."

# Collecter en parallèle
mongostat --uri="$MONGO_URI" --rowcount=$DURATION -o=json > /tmp/mongostat.json &
mongotop --uri="$MONGO_URI" --rowcount=$DURATION -o=json > /tmp/mongotop.json &

wait

echo ""
echo "=== CORRELATION ANALYSIS ==="

# Période de haute charge ?
HIGH_LOAD=$(jq '[.[] | select(.qr > 10 or .qw > 10)] | length' /tmp/mongostat.json)
echo "Periods of high queue: $HIGH_LOAD / $DURATION"

if [ "$HIGH_LOAD" -gt 10 ]; then
  echo ""
  echo "High load detected. Top collections during that period:"

  # Extraire les timestamps des périodes de haute charge
  TIMESTAMPS=$(jq -r '.[] | select(.qr > 10 or .qw > 10) | .time' /tmp/mongostat.json)

  # Trouver les collections correspondantes dans mongotop
  # (Note : nécessite correspondance temporelle précise)
  echo "(Analysis simplified - check /tmp/mongotop.json for details)"
fi
```

---

## Intégration avec Stack de Monitoring

### 1. Export vers Prometheus

```python
#!/usr/bin/env python3
# mongostat_exporter.py

from prometheus_client import start_http_server, Gauge
import subprocess
import json
import time

# Définir les métriques Prometheus
metrics = {
    'insert': Gauge('mongodb_operations_insert_total', 'Insert operations per second'),
    'query': Gauge('mongodb_operations_query_total', 'Query operations per second'),
    'update': Gauge('mongodb_operations_update_total', 'Update operations per second'),
    'delete': Gauge('mongodb_operations_delete_total', 'Delete operations per second'),
    'qr': Gauge('mongodb_queue_read', 'Read queue depth'),
    'qw': Gauge('mongodb_queue_write', 'Write queue depth'),
    'ar': Gauge('mongodb_active_read', 'Active read operations'),
    'aw': Gauge('mongodb_active_write', 'Active write operations'),
    'dirty': Gauge('mongodb_cache_dirty_percent', 'Dirty cache percentage'),
    'used': Gauge('mongodb_cache_used_percent', 'Used cache percentage'),
    'conn': Gauge('mongodb_connections_current', 'Current connections'),
}

def collect_stats(uri):
    """Collect stats from mongostat"""
    result = subprocess.run(
        ['mongostat', '--uri', uri, '--rowcount', '1', '-o', 'json'],
        capture_output=True,
        text=True
    )

    if result.returncode == 0:
        data = json.loads(result.stdout)
        return data[0] if data else {}
    return {}

def update_metrics(uri):
    """Update Prometheus metrics"""
    stats = collect_stats(uri)

    for metric_name, gauge in metrics.items():
        value = stats.get(metric_name, 0)
        gauge.set(value)

if __name__ == '__main__':
    import sys

    if len(sys.argv) < 2:
        print("Usage: mongostat_exporter.py <mongo_uri>")
        sys.exit(1)

    uri = sys.argv[1]

    # Démarrer le serveur HTTP pour Prometheus
    start_http_server(9216)
    print("Exporter started on port 9216")

    # Boucle de collecte
    while True:
        try:
            update_metrics(uri)
            time.sleep(15)  # Collecter toutes les 15 secondes
        except Exception as e:
            print(f"Error: {e}")
            time.sleep(15)
```

**Configuration Prometheus** :
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mongodb_mongostat'
    static_configs:
      - targets: ['localhost:9216']
        labels:
          cluster: 'prod'
          environment: 'production'
```

### 2. Dashboard Grafana

```json
{
  "dashboard": {
    "title": "MongoDB mongostat Dashboard",
    "panels": [
      {
        "title": "Operations per Second",
        "targets": [
          {
            "expr": "mongodb_operations_query_total",
            "legendFormat": "Queries"
          },
          {
            "expr": "mongodb_operations_insert_total",
            "legendFormat": "Inserts"
          },
          {
            "expr": "mongodb_operations_update_total",
            "legendFormat": "Updates"
          },
          {
            "expr": "mongodb_operations_delete_total",
            "legendFormat": "Deletes"
          }
        ]
      },
      {
        "title": "Queue Depth",
        "targets": [
          {
            "expr": "mongodb_queue_read",
            "legendFormat": "Read Queue"
          },
          {
            "expr": "mongodb_queue_write",
            "legendFormat": "Write Queue"
          }
        ],
        "alert": {
          "conditions": [
            {
              "evaluator": {
                "params": [10],
                "type": "gt"
              },
              "query": {
                "params": ["A", "5m", "now"]
              }
            }
          ]
        }
      },
      {
        "title": "Cache Usage",
        "targets": [
          {
            "expr": "mongodb_cache_dirty_percent",
            "legendFormat": "Dirty %"
          },
          {
            "expr": "mongodb_cache_used_percent",
            "legendFormat": "Used %"
          }
        ]
      }
    ]
  }
}
```

### 3. Logging Structuré

```bash
#!/bin/bash
# mongostat_logger.sh

MONGO_URI="mongodb://monitoring:password@prod.example.com:27017/?authSource=admin"
LOG_FILE="/var/log/mongodb/mongostat.log"

while true; do
  STATS=$(mongostat --uri="$MONGO_URI" --rowcount=1 -o=json | jq -c '.[0]')

  # Ajouter timestamp et hostname
  echo "{\"timestamp\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\", \"host\": \"$(hostname)\", \"stats\": $STATS}" >> $LOG_FILE

  sleep 60
done
```

---

## Comparaison et Complémentarité

### mongostat vs mongotop

| Aspect | mongostat | mongotop |
|--------|-----------|----------|
| **Granularité** | Serveur global | Par collection |
| **Métriques** | Opérations, cache, queues | Temps lecture/écriture |
| **Usage principal** | Santé générale | Hot collections |
| **Performance impact** | Très faible | Très faible |
| **Fréquence** | 1s typique | 1-5s typique |

**Utilisation combinée** :
```bash
# Terminal 1 : Vue globale
mongostat --uri="$MONGO_URI"

# Terminal 2 : Collections chaudes
mongotop --uri="$MONGO_URI" 5

# Analyse : Si mongostat montre qr/qw élevés, mongotop identifie quelle(s) collection(s)
```

### mongostat vs Prometheus Exporter

| Aspect | mongostat | Prometheus Exporter |
|--------|-----------|-------------------|
| **Temps réel** | Oui (1s) | Non (15-60s) |
| **Historique** | Non | Oui |
| **Alerting** | Manuel | Automatique |
| **Visualisation** | Terminal | Dashboard |
| **Export** | Possible | Natif |

**Stratégie recommandée** :
- **Production normale** : Prometheus + Grafana pour monitoring continu
- **Incident/debug** : mongostat pour diagnostic immédiat
- **Analyse** : Combiner les deux sources

### mongostat vs Profiler

| Aspect | mongostat | Profiler |
|--------|-----------|----------|
| **Niveau** | Serveur | Requête |
| **Detail** | Agrégé | Individuel |
| **Impact** | Négligeable | 1-5% |
| **Usage** | Temps réel | Post-mortem |

**Workflow typique** :
1. **Alerte** → mongostat identifie le problème global (ex: qr élevé)
2. **mongotop** → Identifie la collection problématique
3. **Profiler** → Analyse les requêtes spécifiques sur cette collection

---

## Bonnes Pratiques pour SRE

### 1. Checklist de Monitoring

```yaml
Setup Initial:
  ✓ Créer utilisateur dédié monitoring (clusterMonitor)
  ✓ Tester connexion mongostat/mongotop
  ✓ Établir baseline de performance
  ✓ Documenter valeurs normales
  ✓ Configurer alerting sur seuils

Monitoring Quotidien:
  ✓ Vérifier dashboard Grafana (vue long terme)
  ✓ Exécuter mongostat pendant 1-2 minutes si anomalie
  ✓ Comparer avec baseline
  ✓ Documenter tout écart significatif

Investigation Incident:
  ✓ Capturer mongostat + mongotop immédiatement
  ✓ Sauvegarder output pour analyse post-mortem
  ✓ Corréler avec logs application
  ✓ Documenter timeline

Post-Incident:
  ✓ Analyser données capturées
  ✓ Ajuster seuils d'alerte si nécessaire
  ✓ Mettre à jour baseline si changement légitime
  ✓ Créer/mettre à jour runbook
```

### 2. Seuils d'Alerte Recommandés

```yaml
Seuils Critiques (P0 - Page immédiat):
  qr OR qw: > 50
  dirty: > 40%
  used: > 95%
  conn: > 90% de maxIncomingConnections

Seuils Warning (P1 - Investigation 30min):
  qr OR qw: > 20
  dirty: > 20%
  used: > 85%
  operations (insert+query+update+delete): > 10000/s
  conn: > 75% de maxIncomingConnections

Seuils Information (P2 - Investigation 4h):
  qr OR qw: > 10
  dirty: > 10%
  operations: > 5000/s
  getmore: > 100/s
```

### 3. Scripts d'Automatisation

```bash
#!/bin/bash
# /opt/monitoring/mongodb_health_check.sh

MONGO_URI="mongodb://monitoring:password@prod.example.com:27017/?authSource=admin"
ALERT_EMAIL="ops@example.com"
SLACK_WEBHOOK="https://hooks.slack.com/services/XXX"

# Fonction d'alerte
send_alert() {
  local severity=$1
  local message=$2

  # Email
  echo "$message" | mail -s "MongoDB Alert [$severity]" $ALERT_EMAIL

  # Slack
  local color
  case $severity in
    CRITICAL) color="danger" ;;
    WARNING) color="warning" ;;
    *) color="good" ;;
  esac

  curl -X POST $SLACK_WEBHOOK \
    -H 'Content-Type: application/json' \
    -d "{\"attachments\":[{\"color\":\"$color\",\"text\":\"$message\"}]}"
}

# Capturer stats
STATS=$(mongostat --uri="$MONGO_URI" --rowcount=10 -o=json)

# Analyser
QR_AVG=$(echo $STATS | jq '[.[].qr] | add / length')
QW_AVG=$(echo $STATS | jq '[.[].qw] | add / length')
DIRTY_AVG=$(echo $STATS | jq '[.[].dirty] | add / length')
USED_AVG=$(echo $STATS | jq '[.[].used] | add / length')

# Vérifier seuils
if (( $(echo "$QR_AVG > 50 || $QW_AVG > 50" | bc -l) )); then
  send_alert "CRITICAL" "Queue depth critical: qr=$QR_AVG, qw=$QW_AVG"
elif (( $(echo "$QR_AVG > 20 || $QW_AVG > 20" | bc -l) )); then
  send_alert "WARNING" "Queue depth high: qr=$QR_AVG, qw=$QW_AVG"
fi

if (( $(echo "$DIRTY_AVG > 40" | bc -l) )); then
  send_alert "CRITICAL" "Dirty cache critical: $DIRTY_AVG%"
elif (( $(echo "$DIRTY_AVG > 20" | bc -l) )); then
  send_alert "WARNING" "Dirty cache high: $DIRTY_AVG%"
fi

# Log résultat
echo "[$(date)] Health check completed. qr=$QR_AVG, qw=$QW_AVG, dirty=$DIRTY_AVG%, used=$USED_AVG%" >> /var/log/mongodb_health.log
```

### 4. Runbook Template

```markdown
# Runbook: High Queue Depth (mongostat qr/qw > 20)

## Detection
Alert: High queue depth detected via mongostat monitoring

## Initial Response (< 2 minutes)
1. Confirm issue:
   ```bash
   mongostat --uri="$MONGO_URI" --rowcount=30
   ```

2. Check queue type:
   - High qr (read queue) → Go to Read Queue Section
   - High qw (write queue) → Go to Write Queue Section
   - Both high → Go to General Overload Section

## Read Queue (qr > 20)

### Diagnosis
```bash
# 1. Identify slow queries
mongo "$MONGO_URI" --eval "
  db.currentOp({
    'active': true,
    'op': { '\$in': ['query', 'getmore'] },
    'secs_running': { '\$gt': 3 }
  })
"

# 2. Check for missing indexes
mongotop --uri="$MONGO_URI" 5
# Note the hottest read-heavy collections

# 3. Check if collection scans
mongo "$MONGO_URI/mydb" --eval "
  db.system.profile.find({
    'planSummary': /COLLSCAN/
  }).sort({ts: -1}).limit(5)
"
```

### Actions
1. If missing indexes identified:
   - Create index in background
   - Monitor queue reduction

2. If slow queries identified:
   - Consider killing longest running (if safe)
   - Review application code

3. If legitimate load spike:
   - Scale horizontally (add secondaries)
   - Route reads to secondaries

## Write Queue (qw > 20)

### Diagnosis
```bash
# 1. Check I/O performance
iostat -x 1 10

# 2. Check dirty cache
mongostat --uri="$MONGO_URI" -O='dirty,used,flushes'

# 3. Identify write-heavy collections
mongotop --uri="$MONGO_URI" 5
```

### Actions
1. If I/O bottleneck:
   - Investigate disk saturation
   - Consider faster storage (SSD)

2. If dirty cache high (> 20%):
   - Checkpoints may be slow
   - Review storage configuration

3. If bulk operations:
   - Throttle application writes
   - Use bulk operations

## Escalation
If queue remains > 50 for > 5 minutes:
- Page secondary on-call
- Prepare for potential scale-out
```

### 5. Performance Regression Detection

```python
#!/usr/bin/env python3
# regression_detector.py

import json
import statistics
import subprocess
from datetime import datetime, timedelta

class PerformanceRegressionDetector:
    def __init__(self, baseline_file, mongo_uri):
        self.baseline_file = baseline_file
        self.mongo_uri = mongo_uri

        with open(baseline_file) as f:
            self.baseline = json.load(f)

    def collect_current_stats(self, duration_seconds=300):
        """Collect stats for specified duration"""
        rowcount = duration_seconds

        result = subprocess.run(
            ['mongostat', '--uri', self.mongo_uri, '--rowcount', str(rowcount), '-o', 'json'],
            capture_output=True,
            text=True
        )

        return json.loads(result.stdout)

    def calculate_stats(self, data):
        """Calculate statistical metrics"""
        metrics = {}

        for key in ['insert', 'query', 'update', 'delete', 'qr', 'qw', 'dirty', 'used']:
            values = [d[key] for d in data if key in d]
            if values:
                metrics[key] = {
                    'mean': statistics.mean(values),
                    'median': statistics.median(values),
                    'stdev': statistics.stdev(values) if len(values) > 1 else 0,
                    'p95': sorted(values)[int(len(values) * 0.95)] if len(values) > 0 else 0
                }

        return metrics

    def detect_regressions(self, current_metrics):
        """Detect performance regressions"""
        regressions = []

        for metric, stats in current_metrics.items():
            baseline_key = f'{metric}_avg'
            if baseline_key not in self.baseline:
                continue

            baseline_value = self.baseline[baseline_key]
            current_value = stats['mean']

            # Regression si > 50% plus élevé que baseline
            if current_value > baseline_value * 1.5:
                regressions.append({
                    'metric': metric,
                    'baseline': baseline_value,
                    'current_mean': current_value,
                    'current_p95': stats['p95'],
                    'degradation_percent': ((current_value - baseline_value) / baseline_value) * 100
                })

        return regressions

    def report(self, regressions):
        """Generate regression report"""
        if not regressions:
            print("✓ No performance regressions detected")
            return

        print(f"\n⚠️  DETECTED {len(regressions)} PERFORMANCE REGRESSIONS\n")

        for r in sorted(regressions, key=lambda x: x['degradation_percent'], reverse=True):
            print(f"Metric: {r['metric']}")
            print(f"  Baseline: {r['baseline']:.2f}")
            print(f"  Current (mean): {r['current_mean']:.2f}")
            print(f"  Current (p95): {r['current_p95']:.2f}")
            print(f"  Degradation: {r['degradation_percent']:.1f}%")
            print()

if __name__ == '__main__':
    import sys

    if len(sys.argv) < 3:
        print("Usage: regression_detector.py <baseline_file> <mongo_uri>")
        sys.exit(1)

    detector = PerformanceRegressionDetector(sys.argv[1], sys.argv[2])

    print("Collecting current stats (5 minutes)...")
    current_data = detector.collect_current_stats(300)

    print("Calculating metrics...")
    current_metrics = detector.calculate_stats(current_data)

    print("Detecting regressions...")
    regressions = detector.detect_regressions(current_metrics)

    detector.report(regressions)
```

---

## Troubleshooting

### Problème : mongostat Ne Se Connecte Pas

**Symptômes** :
```
Error: could not connect to server
```

**Solutions** :
```bash
# 1. Vérifier la connectivité réseau
telnet mongodb-host 27017

# 2. Vérifier les credentials
mongo --uri="$MONGO_URI" --eval "db.runCommand({ping: 1})"

# 3. Vérifier les permissions
mongo --uri="$MONGO_URI" --eval "db.runCommand({serverStatus: 1})"

# 4. Vérifier le firewall
sudo iptables -L -n | grep 27017
```

### Problème : Métriques Incohérentes

**Symptômes** :
```
mongostat montre des valeurs aberrantes
```

**Solutions** :
```bash
# 1. Augmenter l'intervalle de polling
mongostat --uri="$MONGO_URI" 5  # Au lieu de 1s

# 2. Vérifier la charge serveur
uptime
top

# 3. Vérifier l'horloge système
date
ntpq -p
```

### Problème : mongotop Montre Temps > Intervalle

**Symptômes** :
```
total: 15.2s (sur intervalle de 1s)
```

**Explication** :
Ceci est normal ! Le temps est cumulé sur toutes les opérations concurrentes.

**Exemple** :
```
Si 15 opérations de 1s chacune s'exécutent en parallèle,
mongotop montre: total = 15s (sur 1s d'intervalle)
```

---

## Résumé pour SRE

### Commandes Essentielles

```bash
# Monitoring de base
mongostat --uri="$MONGO_URI"
mongotop --uri="$MONGO_URI" 5

# Diagnostic incident
mongostat --uri="$MONGO_URI" --rowcount=120 > incident_mongostat.txt
mongotop --uri="$MONGO_URI" --rowcount=120 > incident_mongotop.txt

# Replica set
mongostat --uri="$RS_URI" --discover

# Export JSON
mongostat --uri="$MONGO_URI" -o=json --rowcount=60

# Colonnes personnalisées
mongostat --uri="$MONGO_URI" -O='host,qrw,arw,dirty,used,conn'
```

### Métriques Critiques à Surveiller

```yaml
Tier 1 (Critique):
  - qr/qw: Seuil > 20 = WARNING, > 50 = CRITICAL
  - dirty: Seuil > 20% = WARNING, > 40% = CRITICAL
  - used: Seuil > 85% = WARNING, > 95% = CRITICAL

Tier 2 (Important):
  - ar/aw: Ratio avec CPU cores
  - conn: Proximité avec maxIncomingConnections
  - operations totales: Tendance croissante

Tier 3 (Informatif):
  - net_in/net_out: Patterns anormaux
  - getmore: Curseurs non fermés
  - flushes: Checkpoints multiples
```

### Workflow de Diagnostic Standard

```
1. Alerte reçue
   ↓
2. mongostat (30s) → Vue globale
   ↓
3. Problème identifié ?
   ├─ Cache (dirty/used) → Vérifier I/O, ajuster config
   ├─ Queue (qr/qw) → mongotop pour identifier collections
   │   ↓
   │   4. mongotop (60s) → Collections chaudes
   │   ↓
   │   5. Profiler/currentOp → Requêtes spécifiques
   │   ↓
   │   6. Actions correctives (index, optimisation, scale)
   │
   └─ Connexions → Analyser sources, vérifier pooling
```

---

## Conclusion

**mongostat** et **mongotop** sont des outils essentiels et complémentaires pour tout SRE MongoDB :

- **mongostat** fournit une **vue d'ensemble** instantanée de la santé du serveur
- **mongotop** permet d'**identifier rapidement** les collections problématiques
- Combinés, ils permettent un **diagnostic en < 5 minutes**
- Leur **faible impact** permet un usage continu sans affecter la production

**Points clés à retenir** :
- Établir une baseline des valeurs normales
- Configurer l'alerting sur les métriques critiques
- Les utiliser comme première ligne de défense
- Compléter avec profiler et logs pour analyse approfondie
- Intégrer dans le stack de monitoring global

**Prochaines étapes** :
- Créer les scripts d'automatisation pour votre environnement
- Établir la baseline de performance
- Configurer l'export vers Prometheus/Grafana
- Former l'équipe aux patterns de diagnostic
- Documenter les runbooks spécifiques

---

**Références** :
- [mongostat Documentation](https://www.mongodb.com/docs/database-tools/mongostat/)
- [mongotop Documentation](https://www.mongodb.com/docs/database-tools/mongotop/)
- [MongoDB Performance Best Practices](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)
- [serverStatus Command Reference](https://www.mongodb.com/docs/manual/reference/command/serverStatus/)

⏭️ [Intégration avec Prometheus et Grafana](/13-monitoring-administration/07-prometheus-grafana.md)
