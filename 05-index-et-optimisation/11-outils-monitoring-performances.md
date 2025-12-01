🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.11 Outils de monitoring des performances

## Introduction

Le monitoring des performances est essentiel pour maintenir une base MongoDB saine et rapide. Sans outils de surveillance, vous naviguez à l'aveugle : vous ne voyez les problèmes que lorsqu'ils deviennent critiques et affectent vos utilisateurs.

Dans ce chapitre, nous allons explorer **l'écosystème complet** des outils de monitoring MongoDB :
- 🔧 **Outils natifs MongoDB** intégrés
- 🖥️ **Outils d'administration** visuels
- 📊 **Solutions tierces** spécialisées
- 🚨 **Systèmes d'alerting**
- 📈 **Tableaux de bord** personnalisés

Chaque outil a son rôle : certains pour le diagnostic ponctuel, d'autres pour la surveillance continue. Maîtriser cet écosystème vous permettra de détecter et résoudre les problèmes avant qu'ils n'impactent vos utilisateurs.

---

## Vue d'ensemble des outils

### Catégories d'outils

```
Écosystème de Monitoring MongoDB
════════════════════════════════

1. OUTILS NATIFS MONGODB (intégrés)
   ├─ explain()           : Analyse de requêtes
   ├─ Profiler            : Enregistrement des requêtes
   ├─ currentOp()         : Opérations en cours
   ├─ serverStatus()      : Métriques serveur
   ├─ dbStats() / stats() : Statistiques DB/Collection
   └─ $indexStats         : Utilisation des index

2. OUTILS D'ADMINISTRATION (UI)
   ├─ MongoDB Compass     : Client graphique officiel
   ├─ MongoDB Atlas       : Platform as a Service
   └─ Ops Manager         : Gestion enterprise

3. OUTILS TIERCES (Monitoring externe)
   ├─ Prometheus + Grafana
   ├─ Datadog
   ├─ New Relic
   ├─ AppDynamics
   └─ Elastic Stack (ELK)

4. OUTILS DE LIGNE DE COMMANDE
   ├─ mongostat           : Statistiques temps réel
   ├─ mongotop            : Temps par collection
   └─ mongodump/restore   : Backup/Restore
```

### Quand utiliser quel outil ?

```
Situation → Outil recommandé
════════════════════════════

Requête lente ponctuelle
└─ explain("executionStats")

Surveiller en temps réel
└─ mongostat, MongoDB Compass

Analyser l'historique des requêtes
└─ Profiler MongoDB

Détecter un problème en cours
└─ currentOp()

Métriques système globales
└─ serverStatus()

Monitoring production 24/7
└─ Atlas, Prometheus+Grafana, Datadog

Analyse visuelle des données
└─ MongoDB Compass

Dashboard personnalisé
└─ Grafana + mongodb_exporter
```

---

## Outils natifs MongoDB

### 1. explain() - Analyse de requêtes

**Objectif** : Comprendre comment MongoDB exécute une requête spécifique.

#### Utilisation basique

```javascript
// Analyser une requête
db.users.find({ email: "user@example.com" })
  .explain("executionStats")
```

#### Ce que explain() vous dit

```
Information fournie par explain() :
═══════════════════════════════════

Plan d'exécution :
├─ Quel index est utilisé (ou COLLSCAN)
├─ Ordre des opérations (IXSCAN → FETCH)
└─ Plans alternatifs testés

Statistiques d'exécution :
├─ Temps d'exécution (executionTimeMillis)
├─ Documents examinés (totalDocsExamined)
├─ Documents retournés (nReturned)
├─ Clés index examinées (totalKeysExamined)
└─ Ratio d'efficacité

Détails du Query Planner :
├─ Index disponibles
├─ Raison du choix
└─ Optimisations appliquées
```

#### Exemple d'utilisation

```javascript
// Avant optimisation
const before = db.orders.find({
  userId: 12345,
  status: "pending"
}).explain("executionStats")

console.log(`Stage : ${before.queryPlanner.winningPlan.stage}`)
console.log(`Temps : ${before.executionStats.executionTimeMillis}ms`)
console.log(`Docs examinés : ${before.executionStats.totalDocsExamined}`)

// Créer l'index
db.orders.createIndex({ userId: 1, status: 1 })

// Après optimisation
const after = db.orders.find({
  userId: 12345,
  status: "pending"
}).explain("executionStats")

console.log(`\nAmélioration : ${before.executionStats.executionTimeMillis / after.executionStats.executionTimeMillis}x`)
```

**Avantages** :
- ✅ Gratuit, intégré
- ✅ Très détaillé
- ✅ Indispensable pour optimisation

**Limitations** :
- ⚠️ Une requête à la fois
- ⚠️ Manuel (pas d'automatisation)
- ⚠️ Ne montre pas les tendances

### 2. Profiler MongoDB - Enregistrement des requêtes

**Objectif** : Enregistrer automatiquement toutes les requêtes (ou seulement les lentes) pour analyse ultérieure.

#### Activation du profiler

```javascript
// Niveau 0 : Désactivé (défaut)
db.setProfilingLevel(0)

// Niveau 1 : Seulement les requêtes lentes (> threshold)
db.setProfilingLevel(1, { slowms: 100 })  // > 100ms

// Niveau 2 : TOUTES les requêtes (⚠️ impact performance)
db.setProfilingLevel(2)

// Vérifier le niveau actuel
db.getProfilingStatus()
```

**Sortie** :

```json
{
  "was": 1,
  "slowms": 100,
  "sampleRate": 1.0,
  "ok": 1
}
```

#### Analyser les données du profiler

```javascript
// Voir les 10 requêtes les plus lentes
db.system.profile.find()
  .sort({ millis: -1 })
  .limit(10)
  .pretty()

// Requêtes par collection
db.system.profile.aggregate([
  { $group: {
      _id: "$ns",
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" },
      maxTime: { $max: "$millis" }
  }},
  { $sort: { count: -1 } }
])

// Requêtes avec COLLSCAN
db.system.profile.find({
  "planSummary": "COLLSCAN",
  "millis": { $gt: 100 }
}).sort({ millis: -1 })
```

#### Exemple : Identifier les requêtes problématiques

```javascript
// Top 10 des requêtes par fréquence ET lenteur
db.system.profile.aggregate([
  { $match: {
      op: "query",
      millis: { $gt: 50 }
  }},
  { $group: {
      _id: {
        ns: "$ns",
        command: "$command.filter"
      },
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" },
      maxTime: { $max: "$millis" },
      totalTime: { $sum: "$millis" }
  }},
  { $sort: { totalTime: -1 } },
  { $limit: 10 }
])
```

**Avantages** :
- ✅ Capture automatique
- ✅ Historique des requêtes
- ✅ Identifie les patterns

**Limitations** :
- ⚠️ Impact performance (niveau 2)
- ⚠️ Collection system.profile limitée en taille
- ⚠️ Doit être activé par database

**Bonnes pratiques** :

```
Recommandations Profiler :
═══════════════════════════

Production :
├─ Niveau 1 avec slowms: 100
├─ Activer seulement pendant le diagnostic
└─ Désactiver après analyse

Développement :
├─ Niveau 1 ou 2
└─ Analyser régulièrement

Taille collection :
├─ Par défaut : 1 Mo
├─ Augmenter si besoin : db.setProfilingLevel(1, { slowms: 100 })
└─ Surveiller la croissance
```

### 3. currentOp() - Opérations en cours

**Objectif** : Voir ce qui se passe **en ce moment** sur le serveur MongoDB.

#### Utilisation basique

```javascript
// Toutes les opérations en cours
db.currentOp()

// Seulement les opérations actives
db.currentOp({ active: true })

// Opérations longues (> 5 secondes)
db.currentOp({
  "active": true,
  "secs_running": { $gt: 5 }
})

// Requêtes avec COLLSCAN
db.currentOp({
  "active": true,
  "planSummary": "COLLSCAN"
})
```

#### Exemple de sortie

```json
{
  "inprog": [
    {
      "opid": 123456,
      "active": true,
      "secs_running": 12,
      "op": "query",
      "ns": "mydb.users",
      "command": {
        "find": "users",
        "filter": { "status": "active" }
      },
      "planSummary": "COLLSCAN",          // ⚠️ Pas d'index !
      "numYields": 450,
      "locks": { ... },
      "waitingForLock": false,
      "client": "192.168.1.100:54321"
    }
  ]
}
```

#### Interprétation

```
Champs importants de currentOp() :
═══════════════════════════════════

opid : Identifiant unique de l'opération
secs_running : Temps d'exécution (⚠️ si > 10s)
op : Type d'opération (query, insert, update, delete)
ns : Namespace (database.collection)
planSummary : Plan d'exécution (IXSCAN vs COLLSCAN)
numYields : Nombre de pauses (élevé = lent)
waitingForLock : Bloqué par un verrou
client : Client d'origine
```

#### Tuer une opération problématique

```javascript
// Identifier l'opération problématique
const ops = db.currentOp({
  "active": true,
  "secs_running": { $gt: 30 },
  "ns": "mydb.users"
})

// Tuer l'opération (ATTENTION : à utiliser avec précaution)
db.killOp(123456)  // opid de l'opération

// Ou via admin
db.adminCommand({ killOp: 1, op: 123456 })
```

**⚠️ ATTENTION** : Ne tuez une opération que si vous êtes certain qu'elle pose problème !

**Avantages** :
- ✅ Temps réel
- ✅ Identifier blocages
- ✅ Diagnostiquer problèmes en cours

**Limitations** :
- ⚠️ Snapshot instantané (pas d'historique)
- ⚠️ Ne montre pas les tendances

### 4. serverStatus() - Métriques serveur

**Objectif** : Obtenir les métriques globales du serveur MongoDB.

#### Utilisation

```javascript
// Toutes les métriques
db.serverStatus()

// Sections spécifiques
db.serverStatus().connections
db.serverStatus().opcounters
db.serverStatus().wiredTiger.cache
db.serverStatus().network
```

#### Métriques clés

```javascript
// Connexions
const conn = db.serverStatus().connections
console.log(`Connexions actives : ${conn.current}`)
console.log(`Connexions disponibles : ${conn.available}`)

// Opérations par seconde
const ops = db.serverStatus().opcounters
console.log(`Inserts : ${ops.insert}`)
console.log(`Queries : ${ops.query}`)
console.log(`Updates : ${ops.update}`)
console.log(`Deletes : ${ops.delete}`)

// Cache WiredTiger
const cache = db.serverStatus().wiredTiger.cache
const used = cache["bytes currently in the cache"]
const max = cache["maximum bytes configured"]
console.log(`Cache utilisé : ${(used / max * 100).toFixed(2)}%`)

// Network
const net = db.serverStatus().network
console.log(`Bytes in : ${(net.bytesIn / 1024 / 1024).toFixed(2)} Mo`)
console.log(`Bytes out : ${(net.bytesOut / 1024 / 1024).toFixed(2)} Mo`)
```

#### Script de monitoring

```javascript
// Script de surveillance système
function monitorSystem() {
  const status = db.serverStatus()

  console.log("\n=== MONGODB SERVER STATUS ===")
  console.log(`Uptime : ${(status.uptime / 3600).toFixed(2)} heures`)

  // Connexions
  const conn = status.connections
  console.log(`\nConnexions : ${conn.current} / ${conn.current + conn.available}`)
  if (conn.current / (conn.current + conn.available) > 0.8) {
    console.log("⚠️  ALERTE : Connexions élevées")
  }

  // Cache
  const cache = status.wiredTiger.cache
  const cacheUsage = cache["bytes currently in the cache"] / cache["maximum bytes configured"]
  console.log(`\nCache : ${(cacheUsage * 100).toFixed(2)}%`)
  if (cacheUsage > 0.9) {
    console.log("⚠️  ALERTE : Cache presque plein")
  }

  // Page faults
  const mem = status.extra_info
  console.log(`\nPage faults : ${mem.page_faults}`)

  console.log("============================\n")
}

// Exécuter toutes les 5 secondes
setInterval(monitorSystem, 5000)
```

**Avantages** :
- ✅ Vue d'ensemble complète
- ✅ Métriques système et moteur
- ✅ Détecte les problèmes globaux

**Limitations** :
- ⚠️ Snapshot (pas d'historique)
- ⚠️ Beaucoup de données (filtrer)

### 5. dbStats() et collStats() - Statistiques

#### dbStats() - Statistiques database

```javascript
// Stats complètes d'une database
db.stats()

// Version plus lisible
const stats = db.stats()
console.log(`Collections : ${stats.collections}`)
console.log(`Documents : ${stats.objects}`)
console.log(`Taille données : ${(stats.dataSize / 1024 / 1024).toFixed(2)} Mo`)
console.log(`Taille index : ${(stats.indexSize / 1024 / 1024).toFixed(2)} Mo`)
console.log(`Taille totale : ${(stats.storageSize / 1024 / 1024).toFixed(2)} Mo`)
```

#### collStats() - Statistiques collection

```javascript
// Stats d'une collection spécifique
db.collection.stats()

// Informations clés
const stats = db.users.stats()
console.log(`Documents : ${stats.count}`)
console.log(`Taille moyenne doc : ${stats.avgObjSize} octets`)
console.log(`Taille données : ${(stats.size / 1024 / 1024).toFixed(2)} Mo`)
console.log(`Taille index : ${(stats.totalIndexSize / 1024 / 1024).toFixed(2)} Mo`)
console.log(`Nombre d'index : ${stats.nindexes}`)

// Détail par index
console.log("\nTaille par index :")
for (const [name, size] of Object.entries(stats.indexSizes)) {
  console.log(`  ${name}: ${(size / 1024 / 1024).toFixed(2)} Mo`)
}
```

#### $indexStats - Utilisation des index

```javascript
// Statistiques d'utilisation des index
db.collection.aggregate([{ $indexStats: {} }])

// Analyser l'utilisation
db.users.aggregate([
  { $indexStats: {} },
  { $project: {
      name: 1,
      ops: "$accesses.ops",
      since: "$accesses.since"
  }},
  { $sort: { ops: -1 } }
])
```

**Exemple de sortie** :

```json
[
  {
    "name": "email_1",
    "ops": 125634,
    "since": ISODate("2024-11-01T00:00:00Z")
  },
  {
    "name": "city_1_age_1",
    "ops": 15423,
    "since": ISODate("2024-11-01T00:00:00Z")
  },
  {
    "name": "oldIndex_1",
    "ops": 0,                           // ⚠️ Jamais utilisé !
    "since": ISODate("2024-11-01T00:00:00Z")
  }
]
```

---

## Outils de ligne de commande

### 1. mongostat - Statistiques temps réel

**Objectif** : Voir les métriques en temps réel (comme `top` pour MongoDB).

#### Utilisation

```bash
# Statistiques par seconde
mongostat

# Avec intervalle personnalisé (toutes les 5 secondes)
mongostat --rowcount=0 5

# Connexion à un serveur distant
mongostat --host=mongodb.example.com:27017 --username=admin --password=xxx
```

#### Sortie type

```
insert query update delete getmore command dirty used flushes vsize  res qrw arw net_in net_out conn      time
*0    *250   *10    *0       0     12|0  3.4% 60.2%       0  1.3G  800M 0|0 1|0  2.5m   80.3k   92  10:23:45
*0    *248   *12    *0       0     15|0  3.6% 60.5%       0  1.3G  805M 0|0 1|0  2.4m   79.8k   92  10:23:46
*0    *252   *9     *0       0     11|0  3.3% 60.3%       0  1.3G  802M 0|0 1|0  2.5m   81.2k   92  10:23:47
```

#### Interprétation des colonnes

```
Colonnes mongostat importantes :
════════════════════════════════

insert/query/update/delete : Opérations par seconde
command : Commandes administratives
dirty : % du cache modifié (pas encore écrit)
used : % du cache utilisé
vsize : Mémoire virtuelle
res : Mémoire résidente (RAM)
qrw : Queue de lecture|écriture
net_in/net_out : Trafic réseau
conn : Nombre de connexions actives
```

**Avantages** :
- ✅ Temps réel
- ✅ Vue d'ensemble rapide
- ✅ Détecte les pics d'activité

**Limitations** :
- ⚠️ Pas de détails par collection
- ⚠️ Pas d'historique

### 2. mongotop - Temps par collection

**Objectif** : Voir quelle collection consomme le plus de temps.

#### Utilisation

```bash
# Temps par collection (rafraîchi chaque seconde)
mongotop

# Intervalle personnalisé (toutes les 5 secondes)
mongotop 5

# Connexion à un serveur distant
mongotop --host=mongodb.example.com:27017
```

#### Sortie type

```
                    ns    total    read    write
    mydb.orders     1250ms   450ms    800ms
    mydb.users       850ms   820ms     30ms
    mydb.products    320ms   280ms     40ms
```

**Interprétation** :

```
Collections les plus actives :
══════════════════════════════

total : Temps total passé
read : Temps en lecture
write : Temps en écriture

Si une collection domine :
└─ Vérifier les index
└─ Analyser les requêtes
└─ Optimiser si nécessaire
```

**Avantages** :
- ✅ Identifie les collections problématiques
- ✅ Temps réel
- ✅ Simple à interpréter

**Limitations** :
- ⚠️ Pas de détails sur les requêtes
- ⚠️ Ne montre pas les index utilisés

---

## Outils d'administration graphiques

### 1. MongoDB Compass - Client graphique officiel

**Objectif** : Interface graphique pour explorer, analyser et optimiser MongoDB.

#### Fonctionnalités clés

```
MongoDB Compass - Fonctionnalités
═════════════════════════════════

Exploration de données :
├─ Navigation visuelle des collections
├─ Schéma automatiquement détecté
├─ Requêtes visuelles (Query Builder)
└─ Export/Import de données

Analyse de performance :
├─ Explain Plan visuel
├─ Index recommandés
├─ Analyse du schéma
└─ Validation de documents

Gestion des index :
├─ Création d'index graphique
├─ Visualisation des index existants
├─ Statistiques d'utilisation
└─ Suppression d'index

Agrégations :
├─ Pipeline Builder visuel
├─ Prévisualisation des étapes
├─ Export du code
└─ Optimisation automatique
```

#### Exemple : Analyse de requête dans Compass

```
1. Onglet "Documents"
2. Filtrer : { email: "user@example.com" }
3. Clic sur "Explain"
4. Visualisation graphique :

   ┌──────────────────┐
   │   IXSCAN         │  2ms
   │   email_1        │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │   FETCH          │  1ms
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │   1 document     │
   └──────────────────┘

5. Statistiques :
   - Temps total : 3ms
   - Documents examinés : 1
   - Index utilisé : email_1 ✅
```

#### Recommandations d'index

Compass analyse vos requêtes et suggère des index :

```
💡 Index Recommendation

Requête fréquente détectée :
{ city: "Paris", age: { $gte: 25 } }

Index suggéré :
db.users.createIndex({ city: 1, age: 1 })

Impact estimé :
- Temps de requête : 450ms → 15ms (30x)
- Requêtes affectées : ~1,200/jour
- Espace disque : +120 Mo
```

**Avantages** :
- ✅ Très visuel et intuitif
- ✅ Idéal pour débutants
- ✅ Recommandations automatiques
- ✅ Gratuit

**Limitations** :
- ⚠️ Client lourd (doit être installé)
- ⚠️ Pas de monitoring continu
- ⚠️ Une connexion à la fois

### 2. MongoDB Atlas - Platform as a Service

**Objectif** : MongoDB hébergé avec monitoring intégré.

#### Dashboard Atlas

```
MongoDB Atlas Dashboard
═══════════════════════

CLUSTER OVERVIEW
├─ Utilisation CPU, RAM, Disk
├─ Connexions actives
├─ Opérations par seconde
└─ Network traffic

PERFORMANCE ADVISOR
├─ Index suggestions automatiques
├─ Slow queries détectées
├─ Schéma anti-patterns
└─ Recommandations d'optimisation

REAL-TIME PERFORMANCE
├─ Graphiques temps réel
├─ Top operations
├─ Hottest collections
└─ Query patterns

ALERTS
├─ CPU > 80%
├─ Disk > 90%
├─ Connexions > seuil
└─ Requêtes lentes
```

#### Performance Advisor (exclusif Atlas)

```javascript
// Atlas analyse automatiquement et suggère :

Suggestion 1 : CREATE INDEX
Collection : orders
Index : { userId: 1, createdAt: -1 }
Impact : 3,450 requêtes/jour (avg 250ms → 8ms)
Estimation gain : 96.8%

Suggestion 2 : DROP INDEX
Collection : users
Index : tempIndex_1
Raison : Jamais utilisé depuis 90 jours
Gain : -75 Mo espace disque

Suggestion 3 : SCHEMA CHANGE
Collection : products
Problème : Tableau imbriqué profond
Impact : Performance lectures
Recommandation : Dénormalisation
```

#### Alertes configurables

```
Exemples d'alertes Atlas :
══════════════════════════

Performance :
├─ Requêtes > 1000ms détectées
├─ CPU > 80% pendant 5 min
├─ Disk utilization > 90%
└─ Connections > 90% du max

Index :
├─ Index inutilisés détectés
├─ COLLSCAN fréquents
└─ Ratio index/données > 60%

Disponibilité :
├─ Primary election
├─ Oplog window < 2 heures
└─ Replication lag > 10s
```

**Avantages** :
- ✅ Monitoring 24/7 automatique
- ✅ Performance Advisor intelligent
- ✅ Alertes configurables
- ✅ Backups automatiques
- ✅ Scaling facile

**Limitations** :
- ⚠️ Payant (gratuit tier limité)
- ⚠️ Verrouillage cloud provider

### 3. MongoDB Ops Manager - Enterprise

**Objectif** : Gestion et monitoring pour déploiements on-premise.

```
Ops Manager Features
════════════════════

Monitoring :
├─ Real-time metrics
├─ Custom dashboards
├─ Alerting
└─ Performance trends

Backup :
├─ Automated backups
├─ Point-in-time restore
├─ Encryption
└─ Retention policies

Automation :
├─ Deployment automation
├─ Rolling upgrades
├─ Configuration management
└─ Index management

Query Optimization :
├─ Query profiling
├─ Index recommendations
└─ Performance insights
```

**Avantages** :
- ✅ Complet et professionnel
- ✅ On-premise (contrôle total)
- ✅ Multi-cluster management

**Limitations** :
- ⚠️ Payant (MongoDB Enterprise)
- ⚠️ Complexe à configurer

---

## Solutions tierces

### 1. Prometheus + Grafana

**Objectif** : Stack open-source populaire pour monitoring.

#### Architecture

```
Architecture Prometheus + Grafana
═════════════════════════════════

MongoDB Server
      ↓ (export métriques)
mongodb_exporter
      ↓ (scrape)
Prometheus (stockage time-series)
      ↓ (visualisation)
Grafana (dashboards)
```

#### Installation mongodb_exporter

```bash
# Télécharger mongodb_exporter
wget https://github.com/percona/mongodb_exporter/releases/download/vX.X.X/mongodb_exporter-X.X.X.linux-amd64.tar.gz

# Extraire
tar xvzf mongodb_exporter-X.X.X.linux-amd64.tar.gz

# Lancer l'exporter
./mongodb_exporter --mongodb.uri=mongodb://localhost:27017

# L'exporter expose les métriques sur :9216/metrics
```

#### Configuration Prometheus

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets: ['localhost:9216']
    scrape_interval: 15s
```

#### Métriques disponibles

```
Métriques MongoDB dans Prometheus
══════════════════════════════════

Opérations :
├─ mongodb_op_counters_total{type="insert"}
├─ mongodb_op_counters_total{type="query"}
├─ mongodb_op_counters_total{type="update"}
└─ mongodb_op_counters_total{type="delete"}

Connexions :
├─ mongodb_connections{state="current"}
└─ mongodb_connections{state="available"}

Cache :
├─ mongodb_wiredtiger_cache_bytes{type="total"}
├─ mongodb_wiredtiger_cache_bytes{type="dirty"}
└─ mongodb_wiredtiger_cache_bytes_percentage

Index :
├─ mongodb_index_size_bytes
└─ mongodb_index_accesses_ops

Collections :
├─ mongodb_collection_size_bytes
└─ mongodb_collection_document_count
```

#### Dashboard Grafana

```
Dashboard MongoDB Grafana
═════════════════════════

[OVERVIEW]
┌────────────────────────────────────┐
│ ● MongoDB Status : UP              │
│ ● Uptime : 15d 3h 42m              │
│ ● Version : 7.0.5                  │
└────────────────────────────────────┘

[OPERATIONS] (Graphique temps réel)
Operations/sec ▲
    1000 ┤     ╭──╮
     750 ┤   ╭─╯  ╰──╮
     500 ┤ ╭─╯       ╰─╮
     250 ┤─╯           ╰─
       0 └─────────────────
           12:00  12:15  12:30

[CONNECTIONS]
Current : 85 / 1000 (8.5%)
[████░░░░░░░░░░░░░░░░]

[CACHE]
Used : 6.5 GB / 8 GB (81.2%)
[████████████████░░░░]

[TOP COLLECTIONS BY SIZE]
1. orders    : 2.5 GB
2. users     : 1.8 GB
3. products  : 0.9 GB
```

**Avantages** :
- ✅ Open-source gratuit
- ✅ Très flexible
- ✅ Dashboards personnalisables
- ✅ Historique long terme
- ✅ Alerting puissant

**Limitations** :
- ⚠️ Configuration manuelle
- ⚠️ Courbe d'apprentissage

### 2. Datadog

**Objectif** : Solution SaaS complète de monitoring.

```
Datadog MongoDB Integration
════════════════════════════

Métriques automatiques :
├─ Performance queries
├─ Replica set health
├─ Sharding metrics
└─ Resource utilization

APM Integration :
├─ Trace des requêtes
├─ Corrélation app ↔ DB
└─ Slow query detection

Dashboards prédéfinis :
├─ MongoDB Overview
├─ Replica Set
├─ Sharded Cluster
└─ WiredTiger Engine

Alertes intelligentes :
├─ Anomaly detection
├─ Forecasting
└─ Multi-alert conditions
```

**Avantages** :
- ✅ Installation rapide
- ✅ Dashboards prêts à l'emploi
- ✅ APM intégré
- ✅ Corrélation avec autres services

**Limitations** :
- ⚠️ Payant (coût élevé)
- ⚠️ Vendor lock-in

### 3. Elastic Stack (ELK)

**Objectif** : Centraliser les logs MongoDB.

```
Stack ELK pour MongoDB
══════════════════════

MongoDB Logs
      ↓
Filebeat (collecte)
      ↓
Logstash (parse & enrich)
      ↓
Elasticsearch (indexation)
      ↓
Kibana (visualisation)
```

#### Configuration Filebeat

```yaml
# filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/mongodb/mongodb.log

output.logstash:
  hosts: ["localhost:5044"]
```

#### Analyse dans Kibana

```
Recherches utiles dans Kibana
══════════════════════════════

Slow queries :
└─ "COMMAND" AND "millis" > 1000

COLLSCAN détectés :
└─ "planSummary:COLLSCAN"

Erreurs :
└─ severity:E

Connexions :
└─ "connection accepted"
```

**Avantages** :
- ✅ Centralisation des logs
- ✅ Recherche puissante
- ✅ Corrélation avec autres services
- ✅ Open-source

**Limitations** :
- ⚠️ Complexe à configurer
- ⚠️ Ressources importantes
- ⚠️ Focus sur logs (pas métriques)

---

## Construire un dashboard personnalisé

### Métriques essentielles à surveiller

```
Dashboard MongoDB - Métriques clés
═══════════════════════════════════

1. PERFORMANCE QUERIES
   ├─ P50 latency
   ├─ P95 latency
   ├─ P99 latency
   └─ Slow queries count (> 100ms)

2. THROUGHPUT
   ├─ Operations/sec (insert, query, update, delete)
   ├─ Documents scanned/sec
   └─ Bytes in/out

3. RESSOURCES
   ├─ CPU utilization (%)
   ├─ RAM usage (%)
   ├─ Disk usage (%)
   └─ Network I/O

4. CACHE WIREDTIGER
   ├─ Cache hit ratio (%)
   ├─ Cache used (%)
   ├─ Cache dirty (%)
   └─ Page faults/sec

5. CONNEXIONS
   ├─ Current connections
   ├─ Available connections
   └─ Connection rate (new/sec)

6. REPLICATION (si applicable)
   ├─ Replication lag
   ├─ Oplog window
   └─ Member health

7. INDEX
   ├─ Index size vs data size ratio
   ├─ Index misses
   └─ COLLSCAN count/hour

8. ERRORS & ALERTS
   ├─ Error rate
   ├─ Failed operations
   └─ Active alerts
```

### Script de collecte custom

```javascript
// collect_metrics.js
// À exécuter périodiquement (ex: cron toutes les minutes)

function collectMetrics() {
  const status = db.serverStatus()
  const timestamp = new Date().toISOString()

  const metrics = {
    timestamp: timestamp,

    // Operations
    operations: {
      insert: status.opcounters.insert,
      query: status.opcounters.query,
      update: status.opcounters.update,
      delete: status.opcounters.delete
    },

    // Connexions
    connections: {
      current: status.connections.current,
      available: status.connections.available,
      totalCreated: status.connections.totalCreated
    },

    // Cache
    cache: {
      used_bytes: status.wiredTiger.cache["bytes currently in the cache"],
      max_bytes: status.wiredTiger.cache["maximum bytes configured"],
      dirty_bytes: status.wiredTiger.cache["tracked dirty bytes in the cache"],
      pages_read: status.wiredTiger.cache["pages read into cache"],
      pages_written: status.wiredTiger.cache["pages written from cache"]
    },

    // Memory
    memory: {
      resident_mb: status.mem.resident,
      virtual_mb: status.mem.virtual,
      mapped_mb: status.mem.mapped || 0
    },

    // Network
    network: {
      bytes_in: status.network.bytesIn,
      bytes_out: status.network.bytesOut,
      requests: status.network.numRequests
    }
  }

  // Exporter vers votre système de monitoring
  // (ex: écrire dans un fichier, envoyer à Prometheus, etc.)
  print(JSON.stringify(metrics))

  return metrics
}

// Exécuter
collectMetrics()
```

### Alertes recommandées

```javascript
// alert_rules.js
// Règles d'alertes pour votre monitoring

const alertRules = [
  {
    name: "CPU_HIGH",
    condition: "cpu_usage > 80",
    duration: "5 minutes",
    severity: "WARNING",
    action: "Notify ops team"
  },
  {
    name: "CPU_CRITICAL",
    condition: "cpu_usage > 95",
    duration: "2 minutes",
    severity: "CRITICAL",
    action: "Page on-call engineer"
  },
  {
    name: "MEMORY_HIGH",
    condition: "cache_usage > 90",
    duration: "10 minutes",
    severity: "WARNING",
    action: "Investigate memory usage"
  },
  {
    name: "SLOW_QUERIES",
    condition: "slow_queries_count > 10",
    duration: "1 minute",
    severity: "WARNING",
    action: "Check query performance"
  },
  {
    name: "CONNECTIONS_HIGH",
    condition: "connections > 90% max",
    duration: "5 minutes",
    severity: "CRITICAL",
    action: "Scale up or investigate connection leaks"
  },
  {
    name: "REPLICATION_LAG",
    condition: "replication_lag > 10 seconds",
    duration: "5 minutes",
    severity: "CRITICAL",
    action: "Check replica set health"
  },
  {
    name: "DISK_SPACE",
    condition: "disk_usage > 85%",
    duration: "10 minutes",
    severity: "WARNING",
    action: "Plan capacity upgrade or archive data"
  },
  {
    name: "INDEX_SIZE_RATIO",
    condition: "index_size / data_size > 0.6",
    duration: "1 day",
    severity: "INFO",
    action: "Review index strategy"
  }
]

// Fonction de vérification des alertes
function checkAlerts() {
  const status = db.serverStatus()
  const stats = db.stats()

  alertRules.forEach(rule => {
    // Évaluer la condition
    // Déclencher l'alerte si nécessaire
    // Logique d'alerting personnalisée
  })
}
```

---

## Comparaison des outils

### Tableau comparatif

```
Comparaison des Outils de Monitoring MongoDB
════════════════════════════════════════════

Outil              Coût    Facilité  Détail  Temps-Réel  Alertes  Historique
─────────────────────────────────────────────────────────────────────────────
explain()          Gratuit  ★★★★★    ★★★★★    ❌          ❌       ❌
Profiler           Gratuit  ★★★★     ★★★★★    ✅          ❌       ✅ (limité)
currentOp()        Gratuit  ★★★★     ★★★★     ✅          ❌       ❌
mongostat          Gratuit  ★★★★★    ★★       ✅          ❌       ❌
mongotop           Gratuit  ★★★★★    ★★       ✅          ❌       ❌
Compass            Gratuit  ★★★★★    ★★★★     ❌          ❌       ❌
Atlas              $$$      ★★★★★    ★★★★★    ✅          ✅       ✅
Ops Manager        $$$$     ★★★      ★★★★★    ✅          ✅       ✅
Prometheus+Grafana Gratuit  ★★★      ★★★★     ✅          ✅       ✅
Datadog            $$$$     ★★★★★    ★★★★     ✅          ✅       ✅
ELK Stack          Gratuit  ★★       ★★★★     ✅          ✅       ✅
```

### Recommandations par contexte

```
Quel outil choisir ?
════════════════════

STARTUP / DÉVELOPPEMENT
└─ Compass + Profiler
   Raison : Gratuit, visuel, suffisant

PME / SCALE-UP
└─ Atlas (managed)
   OU Prometheus + Grafana (self-hosted)
   Raison : Bon compromis coût/features

ENTREPRISE
└─ Ops Manager OU Datadog
   Raison : Features avancées, support, SLA

DEBUGGING PONCTUEL
└─ explain() + currentOp() + Profiler
   Raison : Outils natifs, gratuits, détaillés

MONITORING 24/7
└─ Atlas, Datadog, ou Prometheus+Grafana
   Raison : Surveillance continue, alertes

MULTI-CLOUD / HYBRIDE
└─ Prometheus + Grafana
   Raison : Flexible, portable, open-source
```

---

## Checklist de monitoring

### ✅ Checklist : Monitoring de base

```
□ explain() configuré pour debug ponctuel
□ Profiler activé (niveau 1, slowms: 100)
□ mongostat pour surveillance temps réel
□ MongoDB Compass installé
□ dbStats() exécuté régulièrement
□ $indexStats vérifié mensuellement
□ Documentation des requêtes lentes
```

### ✅ Checklist : Monitoring production

```
□ Solution de monitoring 24/7 configurée
□ Dashboard avec métriques clés
□ Alertes configurées (CPU, RAM, Disk, Queries)
□ Historique des métriques conservé (30+ jours)
□ Runbook d'incident documenté
□ On-call rotation définie
□ Tests d'alertes réguliers
□ Revue mensuelle des métriques
□ Capacity planning trimestriel
```

---

## Concepts clés à retenir

### 🎯 Points essentiels

1. **Outils natifs MongoDB** sont gratuits et puissants
   - explain() pour diagnostic
   - Profiler pour historique
   - currentOp() pour temps réel
   - serverStatus() pour vue globale

2. **Outils graphiques** facilitent l'analyse
   - Compass pour développement
   - Atlas pour production managée
   - Ops Manager pour enterprise

3. **Solutions tierces** pour monitoring continu
   - Prometheus + Grafana (open-source)
   - Datadog, New Relic (SaaS)
   - ELK pour logs

4. **Monitoring en couches** :
   - Développement : Compass + explain()
   - Staging : Profiler + mongostat
   - Production : Atlas/Prometheus + Alertes

5. **Métriques critiques** à surveiller :
   - Latence des requêtes (P95, P99)
   - Utilisation ressources (CPU, RAM, Disk)
   - Cache hit ratio
   - Connexions actives
   - Index usage

6. **Alertes essentielles** :
   - Requêtes lentes
   - Ressources saturées
   - Replication lag
   - Espace disque faible

---

## Analogie finale

> **Les outils de monitoring MongoDB, c'est comme les instruments de bord d'un avion :**
>
> **Sans instruments** = Vol à vue
> - Vous voyez les problèmes quand c'est trop tard
> - Pas d'anticipation
> - Réaction au lieu de prévention
>
> **Avec instruments basiques** = Vol de jour
> - explain() = Altimètre (une mesure ponctuelle)
> - mongostat = Vitesse (ce qui se passe maintenant)
> - Vous volez, mais pas par mauvais temps
>
> **Avec instruments complets** = Vol de nuit / IFR
> - Dashboard = Cockpit complet
> - Alertes = Alarmes de sécurité
> - Historique = Boîte noire
> - Vous volez en toute confiance, 24/7
>
> **Un bon pilote (DevOps) :**
> - Surveille les instruments constamment
> - Anticipe les problèmes
> - Réagit rapidement aux alertes
> - Analyse les données post-vol
>
> **Les outils de monitoring sont vos yeux dans le cloud !** ✈️

---

**Vous maîtrisez maintenant les outils de monitoring des performances MongoDB !** 🚀

---


⏭️ [Framework d'Agrégation](/06-framework-agregation/README.md)
