🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.10 Gestion des index en production

## Introduction

La création d'index est une chose, mais les **gérer en production** en est une autre. Une base MongoDB en production nécessite une surveillance constante, une maintenance régulière et une gestion proactive des index pour garantir des performances optimales.

Ce chapitre couvre tous les aspects de la gestion des index en environnement de production :
- 📊 **Surveillance** et monitoring continu
- 🔍 **Détection** des problèmes avant qu'ils ne deviennent critiques
- 🛠️ **Maintenance** régulière et préventive
- 📈 **Optimisation** continue
- 🚨 **Réponse** aux incidents
- 📝 **Documentation** et gouvernance

Une gestion efficace des index est essentielle pour maintenir des performances élevées, contrôler les coûts et assurer la satisfaction des utilisateurs.

---

## Surveillance des index

### 1. Métriques clés à surveiller

#### a) Utilisation des index

```javascript
// Statistiques d'utilisation des index
db.collection.aggregate([{ $indexStats: {} }])
```

**Sortie type** :

```json
[
  {
    "name": "email_1",
    "key": { "email": 1 },
    "host": "mongodb-primary:27017",
    "accesses": {
      "ops": 125634,                    // Nombre d'utilisations
      "since": ISODate("2024-11-01T00:00:00Z")
    }
  },
  {
    "name": "city_1_age_1",
    "key": { "city": 1, "age": 1 },
    "host": "mongodb-primary:27017",
    "accesses": {
      "ops": 3,                         // ⚠️ Très peu utilisé !
      "since": ISODate("2024-11-01T00:00:00Z")
    }
  }
]
```

**Interprétation** :

```
Niveau d'utilisation :
═══════════════════════

> 10,000 ops     ★★★★★ Index critique (haute utilisation)
1,000 - 10,000   ★★★★  Index important
100 - 1,000      ★★★   Index modéré
10 - 100         ★★    Index peu utilisé
< 10             ★     Index inutilisé (candidat à suppression)
0                      Index jamais utilisé (supprimer !)
```

#### b) Taille des index

```javascript
// Taille totale des index
const stats = db.collection.stats()
console.log("Taille données :", (stats.size / 1024 / 1024).toFixed(2), "Mo")
console.log("Taille index :", (stats.totalIndexSize / 1024 / 1024).toFixed(2), "Mo")

// Détail par index
console.log("\nDétail des index :")
for (const [name, size] of Object.entries(stats.indexSizes)) {
  console.log(`  ${name}: ${(size / 1024 / 1024).toFixed(2)} Mo`)
}
```

**Sortie type** :

```
Taille données : 2500.00 Mo
Taille index : 850.00 Mo

Détail des index :
  _id_: 250.00 Mo
  email_1: 180.00 Mo
  city_1_age_1: 320.00 Mo
  status_1_createdAt_-1: 100.00 Mo
```

**Seuils d'alerte** :

```
Ratio index/données :
═════════════════════

< 20%      ✅ Excellent
20% - 40%  ✅ Normal
40% - 60%  ⚠️  À surveiller
60% - 80%  ⚠️  Attention
> 80%      🚨 Problème (trop d'index ou mal optimisés)
```

#### c) RAM disponible pour les index

```javascript
// Vérifier si les index tiennent en RAM
db.serverStatus().wiredTiger.cache
```

**Sortie pertinente** :

```json
{
  "maximum bytes configured": 8589934592,    // 8 Go max
  "bytes currently in the cache": 7516192768, // 7 Go utilisés
  "pages read into cache": 1234567,
  "pages written from cache": 987654
}
```

**Analyse** :

```
Working Set (Index + Données actives) vs RAM
═════════════════════════════════════════════

Working Set < 50% RAM    ✅ Excellent
Working Set < 80% RAM    ✅ Bon
Working Set < 100% RAM   ⚠️  Limite
Working Set > RAM        🚨 Problème - Swapping imminent
```

#### d) Performance des requêtes

```javascript
// Activer le profiler pour requêtes > 100ms
db.setProfilingLevel(1, { slowms: 100 })

// Analyser les requêtes lentes
db.system.profile.aggregate([
  { $match: {
      millis: { $gt: 100 },
      ts: { $gte: new Date(Date.now() - 3600000) }  // Dernière heure
  }},
  { $group: {
      _id: "$command.find",
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" },
      maxTime: { $max: "$millis" }
  }},
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

### 2. Dashboard de monitoring

#### Métriques essentielles à afficher

```
Dashboard MongoDB - Index
═════════════════════════

┌────────────────────────────────────────┐
│ TAILLE TOTALE DES INDEX                │
│ 850 Mo / 8000 Mo RAM (10.6%)           │
│ Tendance : ↗ +5% cette semaine         │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ INDEX LES PLUS UTILISÉS (24h)          │
│ 1. email_1           : 125K ops        │
│ 2. userId_1_status_1 : 89K ops         │
│ 3. createdAt_-1      : 67K ops         │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ INDEX PEU/PAS UTILISÉS (7j)            │
│ 1. oldField_1        : 0 ops  🚨       │
│ 2. tempIndex_1       : 2 ops  ⚠️       │
│ 3. city_1            : 15 ops ⚠️       │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ REQUÊTES LENTES (dernière heure)       │
│ Moyenne : 85ms                         │
│ Max : 1234ms                           │
│ > 100ms : 45 requêtes                  │
│ > 1000ms : 3 requêtes 🚨               │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│ ALERTES ACTIVES                        │
│ 🚨 Index "oldField_1" inutilisé        │
│ ⚠️  RAM cache à 85% (limite proche)    │
└────────────────────────────────────────┘
```

### 3. Outils de monitoring

#### MongoDB Atlas

```
Atlas Dashboard :
═════════════════

• Index Metrics (temps réel)
• Slow Query Analysis
• Index Suggestions (recommandations automatiques)
• Performance Advisor
• Alerting configurable
```

#### Prometheus + Grafana

```javascript
// Exemple de métrique Prometheus
mongodb_index_size_bytes{database="mydb",collection="users",index="email_1"}
mongodb_index_accesses_total{database="mydb",collection="users",index="email_1"}
```

#### MongoDB Ops Manager

```
Ops Manager :
═════════════

• Real-time Performance
• Index Analysis
• Query Optimization
• Automated Alerting
• Capacity Planning
```

---

## Maintenance régulière

### 1. Plan de maintenance mensuel

```
Plan de Maintenance MongoDB - Index
════════════════════════════════════

SEMAINE 1 : Analyse
├─ Jour 1-2 : Collecter $indexStats de toutes les collections
├─ Jour 3-4 : Analyser les requêtes lentes (profiler)
└─ Jour 5 : Identifier index inutilisés ou redondants

SEMAINE 2 : Planification
├─ Jour 1-2 : Évaluer l'impact de suppression d'index inutilisés
├─ Jour 3-4 : Concevoir nouveaux index si besoin
└─ Jour 5 : Préparer le plan d'exécution

SEMAINE 3 : Test
├─ Jour 1-3 : Tester en environnement staging
├─ Jour 4-5 : Valider les performances

SEMAINE 4 : Déploiement
├─ Jour 1-2 : Déploiement en production (hors heures de pointe)
├─ Jour 3-5 : Surveillance intensive post-déploiement
└─ Documenter les changements
```

### 2. Scripts de maintenance automatisés

#### Script : Détecter les index inutilisés

```javascript
// detect_unused_indexes.js

function detectUnusedIndexes(daysThreshold = 30) {
  const collections = db.getCollectionNames()
  const unusedIndexes = []

  collections.forEach(collName => {
    const indexStats = db[collName].aggregate([{ $indexStats: {} }]).toArray()

    indexStats.forEach(idx => {
      // Ignorer _id (toujours nécessaire)
      if (idx.name === "_id_") return

      const ops = idx.accesses.ops
      const daysSince = (new Date() - idx.accesses.since) / (1000 * 60 * 60 * 24)

      // Si pas utilisé depuis X jours
      if (ops === 0 && daysSince > daysThreshold) {
        unusedIndexes.push({
          collection: collName,
          index: idx.name,
          key: idx.key,
          daysSinceCreation: Math.floor(daysSince),
          operations: ops
        })
      }
    })
  })

  return unusedIndexes
}

// Exécution
const unused = detectUnusedIndexes(30)

if (unused.length > 0) {
  print(`\n🚨 ${unused.length} index inutilisés détectés :\n`)
  unused.forEach(idx => {
    print(`Collection : ${idx.collection}`)
    print(`  Index : ${idx.index}`)
    print(`  Key : ${JSON.stringify(idx.key)}`)
    print(`  Jours : ${idx.daysSinceCreation}`)
    print(`  Ops : ${idx.operations}`)
    print(`  Action : Candidat à suppression\n`)
  })
} else {
  print("✅ Aucun index inutilisé détecté")
}
```

#### Script : Analyser la taille des index

```javascript
// analyze_index_size.js

function analyzeIndexSizes() {
  const collections = db.getCollectionNames()
  const analysis = []

  collections.forEach(collName => {
    const stats = db[collName].stats()

    if (!stats.indexSizes) return

    const dataSize = stats.size
    const totalIndexSize = stats.totalIndexSize
    const ratio = (totalIndexSize / dataSize * 100).toFixed(2)

    analysis.push({
      collection: collName,
      dataSize: (dataSize / 1024 / 1024).toFixed(2) + " Mo",
      totalIndexSize: (totalIndexSize / 1024 / 1024).toFixed(2) + " Mo",
      ratio: ratio + "%",
      indexes: Object.entries(stats.indexSizes).map(([name, size]) => ({
        name,
        size: (size / 1024 / 1024).toFixed(2) + " Mo"
      }))
    })
  })

  // Trier par ratio décroissant
  analysis.sort((a, b) => parseFloat(b.ratio) - parseFloat(a.ratio))

  print("\n📊 Analyse de la taille des index :\n")
  analysis.forEach(coll => {
    print(`Collection : ${coll.collection}`)
    print(`  Données : ${coll.dataSize}`)
    print(`  Index : ${coll.totalIndexSize}`)
    print(`  Ratio : ${coll.ratio}`)

    if (parseFloat(coll.ratio) > 50) {
      print(`  ⚠️  ALERTE : Ratio élevé !`)
    }

    print(`  Détail :`)
    coll.indexes.forEach(idx => {
      print(`    - ${idx.name}: ${idx.size}`)
    })
    print()
  })
}

analyzeIndexSizes()
```

#### Script : Identifier les index redondants

```javascript
// detect_redundant_indexes.js

function detectRedundantIndexes() {
  const collections = db.getCollectionNames()
  const redundant = []

  collections.forEach(collName => {
    const indexes = db[collName].getIndexes()

    // Comparer chaque paire d'index
    for (let i = 0; i < indexes.length; i++) {
      for (let j = i + 1; j < indexes.length; j++) {
        const idx1 = indexes[i]
        const idx2 = indexes[j]

        // Ignorer _id
        if (idx1.name === "_id_" || idx2.name === "_id_") continue

        const keys1 = Object.keys(idx1.key)
        const keys2 = Object.keys(idx2.key)

        // Vérifier si idx1 est un préfixe de idx2
        if (keys2.length > keys1.length) {
          const isPrefix = keys1.every((key, index) => {
            return keys2[index] === key && idx1.key[key] === idx2.key[key]
          })

          if (isPrefix) {
            redundant.push({
              collection: collName,
              redundant: idx1.name,
              redundantKey: idx1.key,
              covered: idx2.name,
              coveredKey: idx2.key,
              reason: `${idx1.name} est un préfixe de ${idx2.name}`
            })
          }
        }
      }
    }
  })

  if (redundant.length > 0) {
    print(`\n⚠️  ${redundant.length} index redondants détectés :\n`)
    redundant.forEach(r => {
      print(`Collection : ${r.collection}`)
      print(`  Redondant : ${r.redundant}`)
      print(`  Key : ${JSON.stringify(r.redundantKey)}`)
      print(`  Couvert par : ${r.covered}`)
      print(`  Key : ${JSON.stringify(r.coveredKey)}`)
      print(`  Raison : ${r.reason}`)
      print(`  Action : Envisager suppression de ${r.redundant}\n`)
    })
  } else {
    print("✅ Aucun index redondant détecté")
  }
}

detectRedundantIndexes()
```

### 3. Reconstruction d'index

#### Quand reconstruire ?

```
Signes qu'un index doit être reconstruit :
═══════════════════════════════════════════

✅ Après suppression massive de données (> 30%)
✅ Après import/migration important
✅ Fragmentation élevée détectée
✅ Performance dégradée inexpliquée
✅ Après corruption (rare)

Fréquence recommandée :
• Petites collections (< 1 Go) : Tous les 6 mois
• Moyennes collections (1-10 Go) : Tous les ans
• Grandes collections (> 10 Go) : Sur besoin uniquement
```

#### Processus de reconstruction sécurisé

```javascript
// Méthode recommandée : Drop & Recreate

// 1. Sauvegarder les définitions d'index
const indexes = db.collection.getIndexes()
print("Index sauvegardés :")
printjson(indexes)

// 2. Supprimer les index (sauf _id)
const indexNames = indexes
  .filter(idx => idx.name !== "_id_")
  .map(idx => idx.name)

indexNames.forEach(name => {
  print(`Suppression de ${name}...`)
  db.collection.dropIndex(name)
})

// 3. Recréer les index
indexes.forEach(idx => {
  if (idx.name === "_id_") return

  print(`Création de ${idx.name}...`)

  const options = {}
  if (idx.unique) options.unique = true
  if (idx.sparse) options.sparse = true
  if (idx.partialFilterExpression) {
    options.partialFilterExpression = idx.partialFilterExpression
  }
  if (idx.expireAfterSeconds !== undefined) {
    options.expireAfterSeconds = idx.expireAfterSeconds
  }

  db.collection.createIndex(idx.key, options)
})

print("✅ Reconstruction terminée")
```

---

## Gestion de la croissance

### 1. Planification de la capacité

```
Prévision de croissance des index
══════════════════════════════════

Formule simple :
Taille future = Taille actuelle × (1 + taux de croissance) ^ années

Exemple :
─────────
Taille actuelle : 100 Go
Croissance : 20% par an
Dans 3 ans : 100 × (1.2)^3 = 172.8 Go

Plan d'action :
1. Surveiller la croissance mensuelle
2. Extrapoler sur 6-12 mois
3. Provisionner la RAM en conséquence
4. Prévoir sharding si nécessaire
```

### 2. Stratégies de croissance

#### a) Optimisation des index existants

```javascript
// Avant : Index trop large
db.products.createIndex({
  category: 1,
  brand: 1,
  name: 1,
  description: 1,  // ← Champ long !
  price: 1
})
// Taille : 2 Go

// Après : Index optimisé
db.products.createIndex({
  category: 1,
  brand: 1,
  name: 1,         // Description exclu
  price: 1
})
// Taille : 800 Mo (économie de 60%)
```

#### b) Index partiels pour réduire la taille

```javascript
// Avant : Index complet
db.orders.createIndex({ status: 1, createdAt: -1 })
// 100% des documents indexés
// Taille : 1.5 Go

// Après : Index partiel (seulement orders actifs)
db.orders.createIndex(
  { status: 1, createdAt: -1 },
  {
    partialFilterExpression: {
      status: { $in: ["pending", "processing"] }
    }
  }
)
// 5% des documents indexés
// Taille : 75 Mo (économie de 95%)
```

#### c) Archivage des données anciennes

```javascript
// Stratégie d'archivage pour contrôler la taille

// 1. Créer une collection d'archives
db.orders.aggregate([
  { $match: {
      createdAt: { $lt: new Date("2024-01-01") }
  }},
  { $out: "orders_archive_2023" }
])

// 2. Supprimer de la collection principale
db.orders.deleteMany({
  createdAt: { $lt: new Date("2024-01-01") }
})

// 3. Reconstruire les index (maintenant plus petits)
// Les index sur orders sont maintenant 50% plus petits !
```

### 3. Seuils d'alerte automatiques

```javascript
// Script de monitoring avec alertes

function checkIndexHealth() {
  const alerts = []

  db.getCollectionNames().forEach(collName => {
    const stats = db[collName].stats()

    if (!stats.indexSizes) return

    const totalIndexSize = stats.totalIndexSize
    const dataSize = stats.size
    const ratio = totalIndexSize / dataSize

    // Alerte 1 : Ratio index/données trop élevé
    if (ratio > 0.6) {
      alerts.push({
        level: "WARNING",
        collection: collName,
        message: `Ratio index/données élevé : ${(ratio * 100).toFixed(2)}%`,
        action: "Vérifier si des index peuvent être optimisés"
      })
    }

    // Alerte 2 : Taille totale des index > 80% RAM
    const ramSize = db.serverStatus().wiredTiger.cache["maximum bytes configured"]
    if (totalIndexSize > ramSize * 0.8) {
      alerts.push({
        level: "CRITICAL",
        collection: collName,
        message: `Index trop volumineux : ${(totalIndexSize / 1024 / 1024 / 1024).toFixed(2)} Go`,
        action: "Augmenter RAM ou optimiser/archiver les données"
      })
    }

    // Alerte 3 : Trop d'index
    const indexCount = Object.keys(stats.indexSizes).length
    if (indexCount > 10) {
      alerts.push({
        level: "INFO",
        collection: collName,
        message: `Nombre d'index élevé : ${indexCount}`,
        action: "Vérifier si certains index peuvent être combinés ou supprimés"
      })
    }
  })

  // Afficher les alertes
  if (alerts.length > 0) {
    print(`\n🚨 ${alerts.length} alertes détectées :\n`)
    alerts.forEach((alert, i) => {
      print(`${i + 1}. [${alert.level}] ${alert.collection}`)
      print(`   ${alert.message}`)
      print(`   Action : ${alert.action}\n`)
    })
  } else {
    print("✅ Aucune alerte")
  }

  return alerts
}

// Exécuter la vérification
checkIndexHealth()
```

---

## Optimisation continue

### 1. Cycle d'amélioration continue

```
Cycle d'Optimisation MongoDB
════════════════════════════

1. MESURER (Baseline)
   ├─ Métriques actuelles
   ├─ Requêtes lentes
   └─ Utilisation index

2. ANALYSER
   ├─ Identifier problèmes
   ├─ Prioriser par impact
   └─ Concevoir solutions

3. TESTER
   ├─ Staging/Preprod
   ├─ Valider gains
   └─ Vérifier effets secondaires

4. DÉPLOYER
   ├─ Production (hors heures pointe)
   ├─ Rolling deployment
   └─ Monitoring intensif

5. VALIDER
   ├─ Confirmer améliorations
   ├─ Documenter changements
   └─ Retour à l'étape 1

Fréquence : Cycle mensuel ou trimestriel
```

### 2. Indicateurs de performance (KPIs)

```
KPIs Index MongoDB
══════════════════

Performance :
├─ Temps moyen de requête             < 50ms
├─ P95 temps de requête               < 100ms
├─ P99 temps de requête               < 500ms
└─ Requêtes > 1s par jour             < 10

Utilisation :
├─ Ratio index utilisés / total       > 80%
├─ Index jamais utilisés              0
└─ Ratio index/données                < 40%

Ressources :
├─ Working set dans RAM               > 80%
├─ Cache hit ratio                    > 95%
└─ Page faults                        < 100/sec

Maintenance :
├─ Analyse mensuelle                  ✅
├─ Documentation à jour               ✅
└─ Alertes configurées                ✅
```

### 3. Index Advisor

MongoDB propose des recommandations automatiques :

#### MongoDB Atlas Performance Advisor

```
Performance Advisor recommande :
════════════════════════════════

1. Créer index sur { userId: 1, createdAt: -1 }
   Impact : 1250 requêtes/jour seront 10x plus rapides
   Coût : +150 Mo d'espace disque

2. Supprimer index oldField_1
   Raison : Jamais utilisé depuis 90 jours
   Gain : -80 Mo d'espace disque

3. Combiner index city_1 et city_1_age_1
   Raison : city_1 est redondant
   Gain : -120 Mo, écritures plus rapides
```

#### Script maison d'analyse

```javascript
// Recommandations basées sur le profiler

function generateIndexRecommendations() {
  const recommendations = []

  // Analyser les requêtes lentes sans index
  db.system.profile.aggregate([
    { $match: {
        millis: { $gt: 100 },
        planSummary: "COLLSCAN"
    }},
    { $group: {
        _id: {
          ns: "$ns",
          filter: "$command.filter"
        },
        count: { $sum: 1 },
        avgTime: { $avg: "$millis" }
    }},
    { $match: { count: { $gt: 10 } }},
    { $sort: { count: -1 } }
  ]).forEach(query => {
    recommendations.push({
      type: "CREATE_INDEX",
      collection: query._id.ns,
      filter: query._id.filter,
      frequency: query.count,
      avgTime: Math.round(query.avgTime),
      priority: "HIGH",
      reason: `COLLSCAN détecté ${query.count} fois, temps moyen ${Math.round(query.avgTime)}ms`
    })
  })

  return recommendations
}

// Afficher les recommandations
const recs = generateIndexRecommendations()
print(`\n💡 ${recs.length} recommandations :\n`)
recs.forEach((rec, i) => {
  print(`${i + 1}. [${rec.priority}] ${rec.type}`)
  print(`   Collection : ${rec.collection}`)
  print(`   Raison : ${rec.reason}`)
  print(`   Filtre suggéré : ${JSON.stringify(rec.filter)}\n`)
})
```

---

## Documentation et gouvernance

### 1. Documentation des index

Chaque index doit être documenté :

```javascript
// Exemple de documentation dans le code

/**
 * INDEX : email_1_name_1
 *
 * Collection : users
 * Champs : { email: 1, name: 1 }
 * Options : { unique: true }
 *
 * Objectif : Recherche rapide par email avec nom
 * Requête cible : find({ email: "..." }, { email: 1, name: 1, _id: 0 })
 * Covered query : Oui
 *
 * Performances :
 * - Avant : 3500ms (COLLSCAN)
 * - Après : 2ms (IXSCAN covered)
 * - Amélioration : 1750x
 *
 * Utilisation : ~10,000 requêtes/jour
 * Créé le : 2024-01-15
 * Créé par : équipe-backend
 * Dernière révision : 2024-11-01
 */
db.users.createIndex({ email: 1, name: 1 }, { unique: true })
```

### 2. Registre des index

Maintenir un registre central :

```markdown
# Registre des Index - MongoDB Production

## Collection : users

### Index : email_1
- **Champs** : { email: 1 }
- **Type** : Unique
- **Objectif** : Login utilisateur
- **Fréquence** : 50K req/jour
- **Taille** : 180 Mo
- **Créé** : 2023-06-01
- **Statut** : ✅ Actif

### Index : city_1_age_1
- **Champs** : { city: 1, age: 1 }
- **Type** : Composé
- **Objectif** : Recherche utilisateurs par localisation et âge
- **Fréquence** : 5K req/jour
- **Taille** : 320 Mo
- **Créé** : 2023-08-15
- **Statut** : ✅ Actif
- **Note** : Utilisé par feature de recommandation

### Index : tempIndex_1 [DEPRECATED]
- **Champs** : { tempField: 1 }
- **Type** : Simple
- **Objectif** : Migration temporaire
- **Fréquence** : 0 req/jour
- **Taille** : 50 Mo
- **Créé** : 2024-06-01
- **Statut** : ⚠️ À supprimer (inutilisé)
- **Action** : Suppression planifiée 2024-12-31
```

### 3. Processus d'approbation

```
Processus de Changement d'Index
════════════════════════════════

CRÉATION D'INDEX
────────────────
1. Développeur identifie le besoin
2. Analyse d'impact (taille, performances)
3. Test en dev/staging
4. Documentation de l'index
5. Revue par équipe DBA/SRE
6. Approbation formelle
7. Déploiement en production
8. Validation post-déploiement
9. Mise à jour du registre

SUPPRESSION D'INDEX
───────────────────
1. Identification (index inutilisé)
2. Analyse d'impact ($indexStats)
3. Test avec hideIndex() pendant 7 jours
4. Validation aucun impact négatif
5. Approbation formelle
6. Suppression en production
7. Surveillance 48h post-suppression
8. Mise à jour du registre

MODIFICATION D'INDEX
────────────────────
= Suppression + Création
Avec plan de rollback
```

---

## Réponse aux incidents

### 1. Index manquant détecté en production

**Symptômes** :
- Requêtes soudainement très lentes
- Timeouts fréquents
- CPU à 100%

**Diagnostic** :

```javascript
// 1. Vérifier les requêtes lentes
db.currentOp({
  "secs_running": { $gt: 5 },
  "planSummary": "COLLSCAN"
})

// 2. Identifier la requête problématique
db.system.profile.find({
  millis: { $gt: 1000 }
}).sort({ ts: -1 }).limit(10)
```

**Action immédiate** :

```javascript
// 3. Créer l'index manquant
// Sur un replica set : créer d'abord sur un secondary
db.collection.createIndex({ field: 1 })

// 4. Valider l'amélioration
db.collection.find({ field: "value" })
  .explain("executionStats")

// 5. Documenter l'incident
```

### 2. RAM saturée par les index

**Symptômes** :
- Page faults élevés
- Performances dégradées
- Swapping

**Diagnostic** :

```javascript
// Vérifier l'utilisation mémoire
const cache = db.serverStatus().wiredTiger.cache
const used = cache["bytes currently in the cache"]
const max = cache["maximum bytes configured"]
const percentage = (used / max * 100).toFixed(2)

print(`Cache utilisé : ${percentage}%`)
```

**Actions** :

```javascript
// Option 1 : Supprimer index inutilisés
const unused = detectUnusedIndexes(30)
// Supprimer après validation

// Option 2 : Optimiser les index existants
// Convertir en index partiels si possible

// Option 3 : Augmenter la RAM (court terme)
// Option 4 : Sharding (long terme)
```

### 3. Index corrompu

**Symptômes** :
- Erreurs lors des requêtes
- Résultats incohérents
- Logs d'erreur MongoDB

**Actions** :

```javascript
// 1. Valider la collection
db.runCommand({ validate: "collection", full: true })

// 2. Si corruption confirmée, reconstruire l'index
const indexes = db.collection.getIndexes()
// Sauvegarder les définitions

db.collection.dropIndex("indexName")
db.collection.createIndex({ ... })

// 3. Valider à nouveau
db.runCommand({ validate: "collection" })
```

---

## Checklist de production

### ✅ Checklist quotidienne

```
□ Vérifier les alertes de monitoring
□ Consulter les requêtes lentes (> 100ms)
□ Vérifier l'utilisation du cache RAM
□ Surveiller les page faults
□ Vérifier les logs d'erreur
```

### ✅ Checklist hebdomadaire

```
□ Analyser les tendances de performance
□ Vérifier la croissance de la taille des index
□ Identifier les nouvelles requêtes lentes
□ Vérifier les backups
□ Réviser les alertes déclenchées
```

### ✅ Checklist mensuelle

```
□ Exécuter $indexStats sur toutes les collections
□ Identifier les index inutilisés (ops = 0)
□ Analyser le profiler MongoDB (requêtes lentes)
□ Vérifier le ratio index/données
□ Évaluer le besoin de nouveaux index
□ Mettre à jour la documentation
□ Planifier les optimisations du mois suivant
```

### ✅ Checklist trimestrielle

```
□ Revue complète de tous les index
□ Analyse de la croissance (tendances)
□ Évaluation du besoin de sharding
□ Test de restauration depuis backup
□ Revue de la capacité (RAM, disque)
□ Formation équipe sur les changements
□ Mise à jour des runbooks
```

---

## Bonnes pratiques de production

### ✅ À faire

```
1. Surveiller en continu
   └─ Dashboard temps réel
   └─ Alertes configurées

2. Documenter chaque changement
   └─ Raison de l'index
   └─ Impact attendu
   └─ Résultats obtenus

3. Tester avant production
   └─ Staging avec données réalistes
   └─ Validation des performances

4. Déploiement progressif
   └─ Rolling deployment
   └─ Hors heures de pointe
   └─ Surveillance intensive

5. Maintenir un plan de rollback
   └─ Définitions d'index sauvegardées
   └─ Procédure de retour arrière

6. Communiquer avec l'équipe
   └─ Changements planifiés
   └─ Fenêtres de maintenance
   └─ Post-mortems d'incidents

7. Automatiser ce qui peut l'être
   └─ Détection d'index inutilisés
   └─ Alertes automatiques
   └─ Rapports hebdomadaires
```

### ❌ À éviter

```
1. Créer des index sans analyse
   └─ Toujours mesurer le besoin

2. Ne jamais supprimer d'index
   └─ Accumulation d'index obsolètes

3. Ignorer les alertes
   └─ Les petits problèmes deviennent gros

4. Pas de documentation
   └─ Impossible de comprendre à posteriori

5. Changements en heures de pointe
   └─ Risque d'impact utilisateurs

6. Ne pas tester en staging
   └─ Surprises désagréables en prod

7. Index "au cas où"
   └─ Gaspillage de ressources
```

---

## Outils et scripts utiles

### Script complet de maintenance

```javascript
// maintenance_report.js
// À exécuter mensuellement

print("=" .repeat(60))
print("RAPPORT DE MAINTENANCE MONGODB - INDEX")
print("Date : " + new Date().toISOString())
print("=" .repeat(60))

// 1. Statistiques globales
print("\n📊 1. STATISTIQUES GLOBALES\n")
const collections = db.getCollectionNames()
let totalIndexSize = 0
let totalDataSize = 0

collections.forEach(coll => {
  const stats = db[coll].stats()
  totalIndexSize += stats.totalIndexSize || 0
  totalDataSize += stats.size || 0
})

print(`Collections : ${collections.length}`)
print(`Données totales : ${(totalDataSize / 1024 / 1024 / 1024).toFixed(2)} Go`)
print(`Index totaux : ${(totalIndexSize / 1024 / 1024 / 1024).toFixed(2)} Go`)
print(`Ratio : ${(totalIndexSize / totalDataSize * 100).toFixed(2)}%`)

// 2. Index inutilisés
print("\n🔍 2. INDEX INUTILISÉS\n")
const unused = detectUnusedIndexes(30)
print(`Trouvés : ${unused.length}`)
unused.slice(0, 5).forEach(idx => {
  print(`  - ${idx.collection}.${idx.index}`)
})

// 3. Collections avec ratio élevé
print("\n⚠️  3. COLLECTIONS AVEC RATIO INDEX/DONNÉES ÉLEVÉ\n")
collections.forEach(coll => {
  const stats = db[coll].stats()
  const ratio = stats.totalIndexSize / stats.size
  if (ratio > 0.5) {
    print(`  - ${coll}: ${(ratio * 100).toFixed(2)}%`)
  }
})

// 4. Recommandations
print("\n💡 4. RECOMMANDATIONS\n")
print("Voir rapport détaillé...")

print("\n" + "=".repeat(60))
print("FIN DU RAPPORT")
print("=".repeat(60))
```

---

## Concepts clés à retenir

### 🎯 Points essentiels

1. **Surveillance continue** est indispensable
   - Métriques en temps réel
   - Alertes configurées
   - Dashboard de monitoring

2. **Maintenance régulière** prévient les problèmes
   - Analyse mensuelle
   - Scripts automatisés
   - Documentation à jour

3. **Gestion de la croissance** est proactive
   - Planification de capacité
   - Optimisation continue
   - Archivage si nécessaire

4. **Documentation** est cruciale
   - Registre des index
   - Raison de chaque index
   - Processus d'approbation

5. **Réponse aux incidents** doit être rapide
   - Procédures documentées
   - Plan de rollback
   - Post-mortem systématique

6. **Automatisation** économise du temps
   - Scripts de détection
   - Alertes automatiques
   - Rapports réguliers

7. **Communication** avec l'équipe
   - Changements planifiés
   - Incidents partagés
   - Knowledge base

---

## Ressources pour aller plus loin

### Commandes essentielles

```javascript
// Monitoring
db.collection.aggregate([{ $indexStats: {} }])
db.collection.stats()
db.serverStatus().wiredTiger.cache

// Maintenance
db.collection.getIndexes()
db.collection.dropIndex("indexName")
db.collection.reIndex()  // Utiliser avec précaution

// Diagnostic
db.currentOp()
db.system.profile.find().sort({ ts: -1 })
db.setProfilingLevel(1, { slowms: 100 })
```

---

## Analogie finale

> **Gérer des index en production, c'est comme entretenir un jardin :**
>
> **Sans entretien** = Mauvaises herbes envahissantes
> - Index inutilisés qui prennent de l'espace
> - Performance qui se dégrade
> - Coûts qui augmentent
>
> **Avec entretien régulier** = Jardin florissant
> - Surveillance constante (arroser, observer)
> - Maintenance préventive (tailler, désherber)
> - Planification (saisons, croissance)
> - Documentation (journal du jardinier)
>
> **Un bon jardinier (DBA) sait :**
> - Quand planter (créer des index)
> - Quand tailler (optimiser)
> - Quand arracher (supprimer)
> - Quand laisser pousser (ne pas sur-optimiser)
>
> **Résultat : Un système MongoDB sain et performant toute l'année !** 🌱

---

**Vous maîtrisez maintenant la gestion des index en production !** 🚀

---


⏭️ [Outils de monitoring des performances](/05-index-et-optimisation/11-outils-monitoring-performances.md)
