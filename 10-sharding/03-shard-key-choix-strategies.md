🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.3 Shard Key : Choix et Stratégies

## Introduction

Le choix de la **shard key** est la décision la plus critique et la plus difficile à modifier dans une architecture shardée MongoDB. Une shard key mal choisie peut compromettre définitivement les performances, la scalabilité et la maintenabilité du cluster, tandis qu'une shard key optimale permet une distribution équilibrée, des requêtes rapides et un scaling quasi-linéaire.

Cette section explore en profondeur les critères de sélection, les stratégies éprouvées, et les anti-patterns à éviter pour prendre la meilleure décision possible avant le déploiement en production.

> ⚠️ **ATTENTION CRITIQUE** : Avant MongoDB 5.0, la shard key est **totalement immuable** après sharding. Depuis MongoDB 5.0, il est possible de la raffiner (`refineCollectionShardKey`) mais au prix d'une opération lourde et coûteuse. **Le choix initial doit être extrêmement réfléchi.**

## Comprendre la Shard Key

### Définition et Rôle

```javascript
// Une shard key est un champ indexé (ou combinaison de champs)
// qui détermine COMMENT les documents sont distribués entre shards

// Exemple simple:
sh.shardCollection("ecommerce.orders", { customer_id: 1 })
//                                       ^^^^^^^^^^^^^^^^
//                                       Shard Key

// Chaque document est assigné à un chunk basé sur sa shard key:
{ customer_id: 100, order_date: "2024-12-08", ... }  → Chunk A → Shard 1
{ customer_id: 5000, order_date: "2024-12-08", ... } → Chunk B → Shard 2
{ customer_id: 9999, order_date: "2024-12-08", ... } → Chunk C → Shard 3
```

### Anatomie d'une Shard Key

```
┌────────────────────────────────────────────────────────┐
│              TYPES DE SHARD KEYS                       │
└────────────────────────────────────────────────────────┘

1. SIMPLE (Single Field)
   { field: 1 }

   Exemple: { customer_id: 1 }

   ✅ Avantages:
      - Simplicité
      - Requêtes ciblées faciles
   ❌ Inconvénients:
      - Risque de hotspots si monotone
      - Distribution peut être inégale

2. COMPOSÉE (Compound)
   { field1: 1, field2: 1, ... }

   Exemple: { region: 1, customer_id: 1 }

   ✅ Avantages:
      - Meilleure distribution
      - Ciblage par préfixe possible
      - Évite certains hotspots
   ❌ Inconvénients:
      - Requêtes doivent inclure préfixe
      - Plus complexe à comprendre

3. HASHED (Champ Haché)
   { field: "hashed" }

   Exemple: { _id: "hashed" }

   ✅ Avantages:
      - Distribution uniforme garantie
      - Pas de hotspots
   ❌ Inconvénients:
      - Impossibilité requêtes par plage
      - Toujours scatter-gather pour ranges

4. COMPOUND AVEC HASHED (MongoDB 4.4+)
   { field1: "hashed", field2: 1 }

   Exemple: { location: "hashed", timestamp: 1 }

   ✅ Avantages:
      - Distribution uniforme sur field1
      - Range queries possibles sur field2
   ❌ Inconvénients:
      - Complexité accrue
      - field1 doit avoir bonne cardinalité
```

## Le Framework CWT : Les 3 Piliers d'une Bonne Shard Key

Une shard key optimale doit satisfaire **trois propriétés fondamentales**, formant le framework **CWT** :

### 1. Cardinality (Cardinalité)

```
┌────────────────────────────────────────────────────────┐
│  CARDINALITÉ = Nombre de valeurs distinctes possibles  │
└────────────────────────────────────────────────────────┘

HAUTE CARDINALITÉ (✅ BON)
┌────────────────────────────────────┐
│ Field: user_id                     │
│ Valeurs: 10,000,000 users          │
│                                    │
│ → Permet 10M chunks théoriques     │
│ → Distribution très granulaire     │
│ → Scaling jusqu'à 10M shards (!)   │
└────────────────────────────────────┘

FAIBLE CARDINALITÉ (❌ MAUVAIS)
┌────────────────────────────────────┐
│ Field: status                      │
│ Valeurs: "active", "inactive" (2)  │
│                                    │
│ → Maximum 2 chunks possibles       │
│ → IMPOSSIBLE distribuer sur > 2    │
│ → Jumbo chunks garantis            │
└────────────────────────────────────┘
```

**Calcul de la Cardinalité :**

```javascript
// Analyser la cardinalité d'un champ
db.collection.aggregate([
  { $group: { _id: "$candidate_field", count: { $sum: 1 } } },
  { $count: "distinct_values" }
])

// Exemple:
db.orders.aggregate([
  { $group: { _id: "$customer_id", count: { $sum: 1 } } },
  { $count: "distinct_values" }
])
// Output: { distinct_values: 1250000 }  ✅ Excellente cardinalité

db.orders.aggregate([
  { $group: { _id: "$status", count: { $sum: 1 } } },
  { $count: "distinct_values" }
])
// Output: { distinct_values: 3 }  ❌ Cardinalité insuffisante
```

**Seuils de Cardinalité :**

```
┌──────────────────────┬──────────────┬─────────────────┐
│   Cardinalité        │  Évaluation  │  Nb Shards Max  │
├──────────────────────┼──────────────┼─────────────────┤
│   < 10               │      ❌      │  Inutilisable   │
│   10-100             │      ⚠️      │  < 10 shards    │
│   100-1,000          │      ⚠️      │  10-20 shards   │
│   1,000-10,000       │      ✅      │  100+ shards    │
│   > 10,000           │      ✅✅    │  Illimité       │
└──────────────────────┴──────────────┴─────────────────┘

Règle empirique:
Cardinalité minimale = Nombre de shards cible × 10
```

### 2. Write Distribution (Distribution des Écritures)

```
┌────────────────────────────────────────────────────────┐
│  DISTRIBUTION = Répartition uniforme des insertions   │
└────────────────────────────────────────────────────────┘

DISTRIBUTION UNIFORME (✅ BON)
┌────────────────────────────────────────────────────────┐
│ Shard Key: { user_id: "hashed" }                       │
│                                                        │
│ Insertions réparties uniformément:                     │
│   Shard A: ████████ (25% des writes)                   │
│   Shard B: ████████ (25% des writes)                   │
│   Shard C: ████████ (25% des writes)                   │
│   Shard D: ████████ (25% des writes)                   │
│                                                        │
│ Throughput total = Somme des shards                    │
│ Pas de bottleneck                                      │
└────────────────────────────────────────────────────────┘

HOTSPOT (❌ MAUVAIS)
┌────────────────────────────────────────────────────────┐
│ Shard Key: { timestamp: 1 }  (monotone croissant)      │
│                                                        │
│ TOUTES les insertions vont au chunk le plus récent:    │
│   Shard A: ░░░░░░░░ (0% des writes - ancien)           │
│   Shard B: ░░░░░░░░ (0% des writes - ancien)           │
│   Shard C: ░░░░░░░░ (0% des writes - ancien)           │
│   Shard D: ████████ (100% des writes - ACTUEL)         │
│                                                        │
│ Throughput = Capacité d'UN SEUL shard                  │
│ Shards A, B, C sous-utilisés                           │
│ Shard D surchargé → Bottleneck                         │
└────────────────────────────────────────────────────────┘
```

**Analyse de la Distribution :**

```javascript
// Simuler la distribution des insertions
db.collection.aggregate([
  { $sample: { size: 10000 } },  // Échantillon de 10k docs récents
  {
    $bucket: {
      groupBy: "$shard_key_field",
      boundaries: [
        MinKey,
        1000, 2000, 3000, 4000, 5000,
        MaxKey
      ],
      output: { count: { $sum: 1 } }
    }
  }
])

// Distribution idéale (uniforme):
[
  { _id: MinKey, count: 1666 },   // ~16.6%
  { _id: 1000, count: 1667 },     // ~16.7%
  { _id: 2000, count: 1666 },     // ~16.6%
  { _id: 3000, count: 1667 },     // ~16.7%
  { _id: 4000, count: 1667 },     // ~16.7%
  { _id: 5000, count: 1667 }      // ~16.7%
]

// Distribution problématique (hotspot):
[
  { _id: MinKey, count: 50 },     // ~0.5%
  { _id: 1000, count: 60 },       // ~0.6%
  { _id: 2000, count: 45 },       // ~0.45%
  { _id: 3000, count: 55 },       // ~0.55%
  { _id: 4000, count: 40 },       // ~0.4%
  { _id: 5000, count: 9750 }      // ~97.5% ❌ HOTSPOT!
]
```

**Patterns de Distribution :**

```
TYPE 1: MONOTONE CROISSANT (❌ Hotspot garanti)
─────────────────────────────────────────────────
timestamp, auto_increment_id, ObjectId, UUID v1

   Temps →
   ┌──────┬──────┬──────┬──────┬──────┬──────┐
   │      │      │      │      │      │██████│
   │      │      │      │      │      │██████│ ← Toutes les
   │      │      │      │      │      │██████│   insertions
   └──────┴──────┴──────┴──────┴──────┴──────┘
   Shard1 Shard2 Shard3 Shard4 Shard5 Shard6

TYPE 2: ALÉATOIRE (✅ Distribution idéale)
─────────────────────────────────────────────────
hash(field), UUID v4, random_string

   ┌──────┬──────┬──────┬──────┬──────┬──────┐
   │██████│██████│██████│██████│██████│██████│
   │██████│██████│██████│██████│██████│██████│ ← Uniforme
   │██████│██████│██████│██████│██████│██████│
   └──────┴──────┴──────┴──────┴──────┴──────┘
   Shard1 Shard2 Shard3 Shard4 Shard5 Shard6

TYPE 3: SKEWED (⚠️ Déséquilibre)
─────────────────────────────────────────────────
country (90% US), tenant_id (un gros client)

   ┌──────┬──────┬──────┬──────┬──────┬──────┐
   │██████│██    │███   │██    │███   │██    │
   │██████│      │      │      │      │      │ ← US domine
   │██████│      │      │      │      │      │
   └──────┴──────┴──────┴──────┴──────┴──────┘
    US     FR     DE     UK     IT     ES
```

### 3. Targetability (Ciblabilité)

```
┌────────────────────────────────────────────────────────┐
│  CIBLABILITÉ = Requêtes incluent la shard key          │
└────────────────────────────────────────────────────────┘

HAUTE CIBLABILITÉ (✅ BON)
┌────────────────────────────────────────────────────────┐
│ Shard Key: { customer_id: 1 }                          │
│                                                        │
│ Requête fréquente:                                     │
│   db.orders.find({ customer_id: 12345 })               │
│                    ^^^^^^^^^^^^^^^^^ ← Shard key!      │
│                                                        │
│ Routage: CIBLÉ vers 1 seul shard                       │
│ Latence: 5-10ms                                        │
│ Network: Minimal                                       │
└────────────────────────────────────────────────────────┘

FAIBLE CIBLABILITÉ (❌ MAUVAIS)
┌────────────────────────────────────────────────────────┐
│ Shard Key: { customer_id: 1 }                          │
│                                                        │
│ Requête fréquente:                                     │
│   db.orders.find({ product_id: "ABC123" })             │
│                    ^^^^^^^^^^^^^^^^ ← PAS shard key!   │
│                                                        │
│ Routage: SCATTER-GATHER (tous shards)                  │
│ Latence: 50-200ms (× N shards)                         │
│ Network: × N shards                                    │
└────────────────────────────────────────────────────────┘
```

**Analyse des Patterns de Requête :**

```javascript
// 1. Activer le profiler pour capturer requêtes (dev/staging)
db.setProfilingLevel(2)  // Log toutes les requêtes

// 2. Après quelques heures/jours, analyser:
db.system.profile.aggregate([
  { $match: { ns: "ecommerce.orders" } },
  {
    $group: {
      _id: "$command.filter",
      count: { $sum: 1 }
    }
  },
  { $sort: { count: -1 } },
  { $limit: 20 }
])

// Output exemple:
[
  { _id: { customer_id: "..." }, count: 150000 },  // 75% des requêtes!
  { _id: { order_id: "..." }, count: 30000 },      // 15%
  { _id: { status: "pending" }, count: 10000 },    // 5%
  { _id: { created_at: { $gte: ... } }, count: 6000 },  // 3%
  ...
]

// Décision:
// customer_id est dans 75% des requêtes
// → Excellente ciblabilité si choisi comme shard key

// 3. Désactiver profiler (impact performance)
db.setProfilingLevel(0)
```

**Règle du Préfixe pour Shard Keys Composées :**

```javascript
// Shard key composée: { region: 1, customer_id: 1 }

// ✅ CIBLÉ: Préfixe complet
db.orders.find({ region: "EU", customer_id: 12345 })
// → Route vers chunks avec region="EU" ET customer_id=12345
// → Probablement 1 seul shard

// ⚠️ PARTIELLEMENT CIBLÉ: Préfixe partiel
db.orders.find({ region: "EU" })
// → Route vers TOUS les chunks avec region="EU"
// → Peut être 1-N shards (selon distribution)

// ❌ SCATTER-GATHER: Pas de préfixe
db.orders.find({ customer_id: 12345 })
// → customer_id seul n'est PAS un préfixe valide
// → Doit interroger TOUS les shards

// ❌ SCATTER-GATHER: Champ non dans shard key
db.orders.find({ product_id: "ABC" })
// → TOUS les shards
```

## Matrice de Décision CWT

### MATRICE D'ÉVALUATION DES SHARD KEYS

| Cardinalité | Distribution | Ciblabilité | Score Global |
|-------------|--------------|-------------|--------------|
| ✅✅ | ✅✅ | ✅✅ | ⭐⭐⭐⭐⭐ PARFAIT |
| ✅✅ | ✅✅ | ⚠️ | ⭐⭐⭐⭐ Excellent |
| ✅✅ | ⚠️ | ✅✅ | ⭐⭐⭐⭐ Excellent |
| ✅✅ | ✅✅ | ❌ | ⭐⭐⭐ Bon (optimize) |
| ✅ | ✅ | ✅ | ⭐⭐⭐ Bon |
| ✅ | ❌ | ✅✅ | ⭐⭐ Acceptable |
| ⚠️ | ✅✅ | ✅✅ | ⭐⭐ Acceptable |
| ❌ | * | * | ❌ INUTILISABLE |
| * | ❌ | ✅✅ | ⚠️ Problématique |

**Légende :**
- ✅✅ = Excellent
- ✅ = Bon
- ⚠️ = Acceptable avec compromis
- ❌ = Problématique / Éliminatoire
- \* = N'importe

## Stratégies de Shard Key par Cas d'Usage

### Cas 1 : E-Commerce (Orders)

```javascript
// ═══════════════════════════════════════════════════════
// COLLECTION: orders (e-commerce)
// ═══════════════════════════════════════════════════════

// Document type:
{
  _id: ObjectId("..."),
  order_id: "ORD-2024-123456",
  customer_id: 1523456,
  order_date: ISODate("2024-12-08T10:30:00Z"),
  status: "pending",
  items: [...],
  total: 199.99,
  shipping_address: { country: "FR", ... }
}

// Patterns de requête (fréquence):
// 1. Par customer_id: 70%
//    "Afficher les commandes de cet utilisateur"
// 2. Par order_id: 20%
//    "Détails de la commande X"
// 3. Par status: 5%
//    "Toutes les commandes en attente"
// 4. Par date range: 5%
//    "Commandes du mois dernier"

// ───────────────────────────────────────────────────────
// OPTION 1: { customer_id: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅ Cardinalité: Haute (millions de customers)
// ⚠️ Distribution: Dépend du comportement clients
//    - Si quelques gros clients → Skew possible
//    - Si clients uniformes → OK
// ✅✅ Ciblabilité: 70% des requêtes (excellent)

// Verdict: ⭐⭐⭐⭐ Excellent choix
sh.shardCollection("ecommerce.orders", { customer_id: 1 })

// ───────────────────────────────────────────────────────
// OPTION 2: { customer_id: "hashed" }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Haute
// ✅✅ Distribution: Parfaite (hash garantit uniformité)
// ✅✅ Ciblabilité: 70% requêtes (customer_id exact)
// ❌ Ciblabilité: Impossible range sur customer_id

// Verdict: ⭐⭐⭐⭐⭐ PARFAIT pour gros clients
sh.shardCollection("ecommerce.orders", { customer_id: "hashed" })

// ───────────────────────────────────────────────────────
// OPTION 3: { order_date: 1, customer_id: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Très haute (date × customer)
// ❌ Distribution: Hotspot sur dates récentes
//    (Toutes nouvelles commandes sur même chunk)
// ⚠️ Ciblabilité:
//    - 70% requêtes avec customer_id seul → SCATTER!
//    - Nécessite { order_date, customer_id } pour cibler

// Verdict: ⭐⭐ Pas optimal
// ❌ NE PAS UTILISER (hotspot temporel)

// ───────────────────────────────────────────────────────
// OPTION 4: { customer_id: 1, order_date: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Très haute
// ✅ Distribution: Bonne (si customers distribués)
// ✅✅ Ciblabilité:
//    - customer_id seul → Ciblé (préfixe)
//    - customer_id + order_date → Très ciblé

// Verdict: ⭐⭐⭐⭐ Excellent compromis
sh.shardCollection("ecommerce.orders",
  { customer_id: 1, order_date: 1 }
)

// Requêtes:
db.orders.find({ customer_id: 12345 })  // ✅ Ciblé
db.orders.find({
  customer_id: 12345,
  order_date: { $gte: ISODate("2024-01-01") }
})  // ✅ Très ciblé

// ───────────────────────────────────────────────────────
// RECOMMANDATION FINALE
// ───────────────────────────────────────────────────────

// Petit/Moyen (< 10M orders):
sh.shardCollection("ecommerce.orders", { customer_id: 1 })

// Gros/Énorme (> 10M orders, gros clients):
sh.shardCollection("ecommerce.orders", { customer_id: "hashed" })

// Avec beaucoup de range queries par date ET customer:
sh.shardCollection("ecommerce.orders",
  { customer_id: 1, order_date: 1 }
)
```

### Cas 2 : IoT / Time Series

```javascript
// ═══════════════════════════════════════════════════════
// COLLECTION: sensor_data (IoT metrics)
// ═══════════════════════════════════════════════════════

// Document type:
{
  _id: ObjectId("..."),
  device_id: "sensor-12345",
  timestamp: ISODate("2024-12-08T10:30:15.123Z"),
  temperature: 22.5,
  humidity: 65,
  battery: 87
}

// Patterns de requête:
// 1. Par device_id + time range: 80%
//    "Métriques du capteur X des 24 dernières heures"
// 2. Par time range global: 15%
//    "Tous capteurs dans cette fenêtre"
// 3. Par device_id seul: 5%
//    "Dernière mesure du capteur X"

// Insertion pattern:
// - Insertions continues (streaming)
// - Toujours avec timestamp récent
// - Millions d'événements/jour

// ───────────────────────────────────────────────────────
// ❌ OPTION 1: { timestamp: 1 } - À ÉVITER!
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅ Cardinalité: Haute (timestamps uniques)
// ❌ Distribution: HOTSPOT PERMANENT
//    (Toutes insertions → chunk avec timestamp actuel)
// ✅ Ciblabilité: 95% requêtes avec timestamp

// Problème critique:
// À t=0:    [2024-12-08 00:00 - MaxKey] → Shard A
// À t=1h:   Chunk split, nouveau:
//           [2024-12-08 01:00 - MaxKey] → Shard A
// À t=2h:   Chunk split, nouveau:
//           [2024-12-08 02:00 - MaxKey] → Shard A
//
// Shard A reçoit TOUJOURS 100% des insertions
// Shards B, C, D idle

// Verdict: ❌ INUTILISABLE
// sh.shardCollection("iot.sensor_data", { timestamp: 1 })  ← NE PAS FAIRE

// ───────────────────────────────────────────────────────
// ✅ OPTION 2: { device_id: 1, timestamp: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Très haute (device × time)
// ✅✅ Distribution: Excellente
//    (Si 100k devices → insertions distribuées sur 100k buckets)
// ✅✅ Ciblabilité: 80% requêtes (device_id + time)

// Fonctionnement:
// Device 1 → Chunks [1-*] → Distribué sur shards
// Device 2 → Chunks [2-*] → Distribué sur shards
// ...
// Device N → Chunks [N-*] → Distribué sur shards

// Insertions parallèles de N devices → N shards actifs

// Verdict: ⭐⭐⭐⭐⭐ PARFAIT
sh.shardCollection("iot.sensor_data",
  { device_id: 1, timestamp: 1 }
)

// Requêtes:
db.sensor_data.find({
  device_id: "sensor-12345",
  timestamp: { $gte: ISODate("2024-12-07") }
})  // ✅ Ciblé vers chunks du device

// ───────────────────────────────────────────────────────
// ✅ OPTION 3: { device_id: "hashed", timestamp: 1 }
//              (MongoDB 4.4+)
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Très haute
// ✅✅ Distribution: Parfaite (hash garantit)
// ✅✅ Ciblabilité: 80% requêtes OK
// ❌ Ciblabilité: Impossible range sur device_id

// Avantage sur Option 2:
// - Distribution encore plus uniforme
// - Pas de risque si devices mal distribués

// Verdict: ⭐⭐⭐⭐⭐ PARFAIT (si pas de range sur device_id)
sh.shardCollection("iot.sensor_data",
  { device_id: "hashed", timestamp: 1 }
)

// ───────────────────────────────────────────────────────
// RECOMMANDATION FINALE
// ───────────────────────────────────────────────────────

// Standard (device_id bien distribué):
sh.shardCollection("iot.sensor_data",
  { device_id: 1, timestamp: 1 }
)

// Gros volume (> 1B events/jour, distribution incertaine):
sh.shardCollection("iot.sensor_data",
  { device_id: "hashed", timestamp: 1 }
)
```

### Cas 3 : Multi-Tenant SaaS

```javascript
// ═══════════════════════════════════════════════════════
// COLLECTION: documents (SaaS multi-tenant)
// ═══════════════════════════════════════════════════════

// Document type:
{
  _id: ObjectId("..."),
  tenant_id: "tenant-12345",
  document_id: "doc-67890",
  owner_id: "user-abc",
  created_at: ISODate("2024-12-08"),
  data: { ... }
}

// Contraintes:
// - Isolation par tenant (sécurité)
// - Tenants de tailles variées:
//   * 90% petits (< 1 GB)
//   * 9% moyens (1-100 GB)
//   * 1% énormes (> 1 TB) ← "Whale tenants"

// Patterns de requête:
// - 100% des requêtes incluent tenant_id
// - Jamais de requêtes cross-tenant

// ───────────────────────────────────────────────────────
// OPTION 1: { tenant_id: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ⚠️ Cardinalité: Dépend (10k tenants typique)
// ❌ Distribution: TRÈS DÉSÉQUILIBRÉE
//    - Whale tenant → Plusieurs chunks sur 1-2 shards
//    - Petits tenants → Partagent chunks

// Problème:
// Tenant "BigCorp" (1 TB):
//   → 8000 chunks (128 MB chacun)
//   → Concentrés sur 2-3 shards (balancer limite)
//   → Ces shards surchargés

// Verdict: ⭐⭐ Acceptable uniquement si:
// - Tous tenants de taille similaire
// - Pas de whale tenants

sh.shardCollection("saas.documents", { tenant_id: 1 })

// ───────────────────────────────────────────────────────
// ✅ OPTION 2: { tenant_id: 1, document_id: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Très haute (tenant × document)
// ✅ Distribution: Bonne
//    - Whale tenant → Chunks répartis (split sur document_id)
// ✅✅ Ciblabilité: 100% requêtes incluent tenant_id

// Fonctionnement:
// Tenant "BigCorp" avec 10M documents:
// - Range: [BigCorp, doc-0] à [BigCorp, doc-9999999]
// - Split automatique en N chunks
// - Distribués sur plusieurs shards

// Verdict: ⭐⭐⭐⭐ Excellent
sh.shardCollection("saas.documents",
  { tenant_id: 1, document_id: 1 }
)

// ───────────────────────────────────────────────────────
// ✅ OPTION 3: Zone Sharding (Isolation complète)
// ───────────────────────────────────────────────────────

// Pour Enterprise tenants: Shards dédiés

sh.shardCollection("saas.documents",
  { tenant_id: 1, document_id: 1 }
)

// Créer zones:
sh.addShardToZone("shard-premium-1", "enterprise-tenants")
sh.addShardToZone("shard-premium-2", "enterprise-tenants")
sh.addShardToZone("shard-standard-1", "standard-tenants")
sh.addShardToZone("shard-standard-2", "standard-tenants")

// Assigner whale tenants:
sh.updateZoneKeyRange(
  "saas.documents",
  { tenant_id: "BigCorp", document_id: MinKey },
  { tenant_id: "BigCorp", document_id: MaxKey },
  "enterprise-tenants"
)

// Assigner petits tenants:
sh.updateZoneKeyRange(
  "saas.documents",
  { tenant_id: "SmallCo-0000", document_id: MinKey },
  { tenant_id: "SmallCo-9999", document_id: MaxKey },
  "standard-tenants"
)

// Avantages:
// ✅ Isolation performance (whale n'impacte pas small)
// ✅ SLA différenciés possibles
// ✅ Facturation granulaire

// Verdict: ⭐⭐⭐⭐⭐ OPTIMAL pour SaaS Enterprise
```

### Cas 4 : Social Network (Posts)

```javascript
// ═══════════════════════════════════════════════════════
// COLLECTION: posts (réseau social)
// ═══════════════════════════════════════════════════════

// Document type:
{
  _id: ObjectId("..."),
  post_id: "post-123456",
  author_id: 987654,
  created_at: ISODate("2024-12-08T10:30:00Z"),
  content: "...",
  likes: 1523,
  comments_count: 42
}

// Patterns de requête:
// 1. Par author_id: 60%
//    "Posts de cet utilisateur"
// 2. Timeline (following): 30%
//    "Posts des utilisateurs que je suis" (scatter!)
// 3. Par post_id: 10%
//    "Détails du post"

// Challenges:
// - Celebrity users (millions de followers)
// - Long tail (millions d'utilisateurs avec 10 posts)
// - Insertions continues

// ───────────────────────────────────────────────────────
// OPTION 1: { author_id: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅ Cardinalité: Haute (millions d'users)
// ⚠️ Distribution: Skewed
//    - Celebrity avec 100k posts → Plusieurs chunks
//    - Utilisateur lambda avec 10 posts → Partagent chunks
// ✅ Ciblabilité: 60% requêtes

// Problème:
// Celebrity "Influencer123":
//   - 500k posts
//   - 4000 chunks (si 128 MB chunks)
//   - Concentrés sur 1-2 shards
//   - Ces shards surchargés (millions de lectures)

// Verdict: ⭐⭐ Problématique pour celebrities

// ───────────────────────────────────────────────────────
// ✅ OPTION 2: { author_id: "hashed" }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Haute
// ✅✅ Distribution: Parfaite (hash)
//    - Celebrity posts distribués uniformément
// ✅✅ Ciblabilité: 60% requêtes avec author_id exact
// ❌ Ciblabilité: Timeline reste scatter-gather

// Verdict: ⭐⭐⭐⭐ Excellent
sh.shardCollection("social.posts", { author_id: "hashed" })

// ───────────────────────────────────────────────────────
// ✅ OPTION 3: { author_id: 1, created_at: 1 }
// ───────────────────────────────────────────────────────

// Analyse CWT:
// ✅✅ Cardinalité: Très haute
// ✅ Distribution: Bonne
// ✅✅ Ciblabilité:
//    - author_id seul → Ciblé (préfixe)
//    - author_id + created_at → Très ciblé
//    - "Posts récents de X" → Optimal

// Avantage:
// Celebrity posts automatiquement répartis par période

// Verdict: ⭐⭐⭐⭐ Excellent (si beaucoup de time range)
sh.shardCollection("social.posts",
  { author_id: 1, created_at: 1 }
)

// ───────────────────────────────────────────────────────
// RECOMMANDATION
// ───────────────────────────────────────────────────────

// Standard (pas de celebrities extrêmes):
sh.shardCollection("social.posts", { author_id: "hashed" })

// Avec celebrities + time range queries:
sh.shardCollection("social.posts",
  { author_id: 1, created_at: 1 }
)
```

## Anti-Patterns de Shard Key

### 🚫 Anti-Pattern 1 : Monotone Croissant Sans Mitigation

```javascript
// ❌ ANTI-PATTERN: timestamp, _id, auto_increment

sh.shardCollection("events.logs", { timestamp: 1 })
sh.shardCollection("users.sessions", { _id: 1 })  // ObjectId
sh.shardCollection("orders.transactions", { order_number: 1 })  // Auto-incr

// Problème:
┌────────────────────────────────────────────────────┐
│ Temps: T0                                          │
│   Chunk actuel: [2024-12-08 09:00, MaxKey]         │
│   → Shard A                                        │
│   → 100% des insertions sur Shard A                │
├────────────────────────────────────────────────────┤
│ Temps: T1 (1 heure plus tard)                      │
│   Chunk split:                                     │
│   - [2024-12-08 09:00, 2024-12-08 10:00] → Shard A │
│   - [2024-12-08 10:00, MaxKey] → Shard A (actif)   │
│   → 100% des insertions ENCORE sur Shard A         │
├────────────────────────────────────────────────────┤
│ Temps: T2                                          │
│   Migration: [10:00, MaxKey] → Shard B             │
│   Mais... nouvelles insertions encore sur Shard B  │
│   → HOTSPOT se déplace mais persiste               │
└────────────────────────────────────────────────────┘

// Impact mesurable:
// - Shard actif: 50k inserts/sec (limite atteinte)
// - Autres shards: 0 inserts/sec (idle)
// - Throughput cluster = Throughput 1 shard
// - Scaling horizontal inutile

// Solution 1: Hashed
sh.shardCollection("events.logs", { timestamp: "hashed" })
// ✅ Distribution parfaite
// ❌ Perte requêtes par plage

// Solution 2: Compound avec préfixe non-monotone
sh.shardCollection("events.logs", { source_id: 1, timestamp: 1 })
// ✅ Distribution si source_id varié
// ✅ Range queries sur timestamp OK (si source_id fourni)
```

### 🚫 Anti-Pattern 2 : Faible Cardinalité

```javascript
// ❌ ANTI-PATTERN: Champs avec peu de valeurs

sh.shardCollection("users.accounts", { account_type: 1 })
// account_type: "free", "premium", "enterprise" (3 valeurs)

sh.shardCollection("orders.transactions", { status: 1 })
// status: "pending", "completed", "cancelled", "refunded" (4 valeurs)

sh.shardCollection("products.catalog", { category: 1 })
// category: 15 catégories

// Problème:
// Maximum de chunks = Cardinalité de la shard key
//
// account_type (3 valeurs):
//   → Maximum 3 chunks possibles
//   → Impossible distribuer sur > 3 shards
//   → Si 90% "free" → 1 chunk énorme (jumbo)
//
// Exemple avec 100 GB données:
//   - "free": 90 GB → 1 jumbo chunk sur Shard A
//   - "premium": 8 GB → 64 chunks sur Shard B
//   - "enterprise": 2 GB → 16 chunks sur Shard C
//
// Shard A: Surchargé, impossible balancer (jumbo)
// Shards B, C: Sous-utilisés

// Impact:
db.chunks.find({ ns: "users.accounts", jumbo: true })
// { min: { account_type: "free" }, max: { account_type: "premium" },
//   jumbo: true, shard: "shard-a" }

// Tentative de split:
sh.splitAt("users.accounts", { account_type: "free" })
// Error: Cannot split chunk with only one distinct value

// Solution: Shard key composée avec haute cardinalité
sh.shardCollection("users.accounts",
  { account_type: 1, user_id: 1 }
)
// account_type × user_id = 3 × millions = suffisant
```

### 🚫 Anti-Pattern 3 : Shard Key Non Présente dans Requêtes

```javascript
// ❌ ANTI-PATTERN: Shard key rarement utilisée

// Shard key choisie:
sh.shardCollection("products.catalog", { created_at: 1 })

// Mais requêtes réelles:
db.catalog.find({ sku: "ABC-123" })              // 40%
db.catalog.find({ category: "electronics" })     // 30%
db.catalog.find({ name: /laptop/i })             // 20%
db.catalog.find({ price: { $lt: 500 } })         // 10%

// Résultat:
// 100% des requêtes = SCATTER-GATHER

// Impact sur 10 shards:
// - Requête simple: 10 × latency shard + merge
//   * 1 shard: 5ms
//   * 10 shards: max(5, 7, 4, 6, 5, 8, 5, 6, 7, 5) = 8ms
//   * Merge: 2ms
//   * Total: 10ms
//   * vs ciblé: 5ms → 2× plus lent
//
// - Charge réseau: × 10
// - Connexions utilisées: × 10
// - CPU mongos: élevé (merge)

// Solution: Aligner shard key sur requêtes
sh.shardCollection("products.catalog", { sku: 1 })

// Maintenant:
db.catalog.find({ sku: "ABC-123" })  // ✅ Ciblé (1 shard)
```

### 🚫 Anti-Pattern 4 : Shard Key Mutable

```javascript
// ❌ ANTI-PATTERN: Champs qui changent

sh.shardCollection("users.profiles", { email: 1 })

// Problème:
// User change d'email:
db.profiles.updateOne(
  { _id: userId },
  { $set: { email: "new_email@example.com" } }
)

// Avant MongoDB 5.0:
// WriteError: Cannot modify shard key field

// MongoDB 5.0+:
// ✅ Possible, MAIS:
//    1. Document doit migrer de chunk (coûteux)
//    2. Transaction automatique lancée
//    3. Lock sur chunk source et destination
//    4. Si échec → retry ou échec update

// Impact:
// - Update ~50× plus lent (1ms → 50ms)
// - Contention locks
// - Balancer peut être bloqué

// Solution: Shard key immuable
sh.shardCollection("users.profiles", { _id: 1 })
// _id est TOUJOURS immuable

// Ou: Stocker version hachée immuable
sh.shardCollection("users.profiles", { email_hash: 1 })
// email_hash calculé à l'insertion, jamais changé
// Email original peut changer (champ non-shard-key)
```

### 🚫 Anti-Pattern 5 : Shard Key Trop Complexe

```javascript
// ❌ ANTI-PATTERN: Shard key avec 4+ champs

sh.shardCollection("analytics.events", {
  country: 1,
  region: 1,
  city: 1,
  user_segment: 1,
  event_type: 1
})

// Problèmes:

// 1. Ciblabilité: Requête doit inclure TOUS les préfixes
db.events.find({ country: "FR" })
// ❌ Scatter (préfixe incomplet)

db.events.find({ country: "FR", region: "IDF" })
// ❌ Encore scatter

db.events.find({ country: "FR", region: "IDF", city: "Paris" })
// ⚠️ Partiellement ciblé (3/5 préfixe)

db.events.find({
  country: "FR", region: "IDF", city: "Paris",
  user_segment: "premium", event_type: "click"
})
// ✅ Ciblé... mais requête très spécifique (rare!)

// 2. Maintenance:
// - Index multi-champs très gros
// - Difficile à comprendre
// - Splits complexes

// 3. Distribution:
// - Combinaisons rares créent chunks vides
// - Déséquilibre potentiel

// Solution: Limiter à 2-3 champs maximum
sh.shardCollection("analytics.events",
  { country: 1, user_id: 1 }
)
// Plus simple, plus efficace
```

## Raffiner une Shard Key (MongoDB 5.0+)

```javascript
// ═══════════════════════════════════════════════════════
// REFINE COLLECTION SHARD KEY (MongoDB 5.0+)
// ═══════════════════════════════════════════════════════

// Situation:
// Shard key actuelle: { customer_id: 1 }
// Problème: Jumbo chunks pour gros clients
// Objectif: Ajouter champ supplémentaire

// ───────────────────────────────────────────────────────
// PROCÉDURE
// ───────────────────────────────────────────────────────

// Étape 1: Créer index pour nouvelle shard key
db.orders.createIndex({ customer_id: 1, order_date: 1 })

// Étape 2: Raffiner la shard key
db.adminCommand({
  refineCollectionShardKey: "ecommerce.orders",
  key: { customer_id: 1, order_date: 1 }
})

// Étape 3: Attendre rebalancing automatique
sh.isBalancerRunning()

// ───────────────────────────────────────────────────────
// IMPACT ET CONSIDÉRATIONS
// ───────────────────────────────────────────────────────

// ✅ Avantages:
// - Permet de corriger erreur de design
// - Pas besoin de resharder toute la collection
// - Opération online (pas de downtime)

// ❌ Limitations:
// - Peut UNIQUEMENT ajouter des suffixes (pas modifier préfixe)
//   Valide:   {a:1} → {a:1, b:1}
//   Invalide: {a:1} → {b:1}
//   Invalide: {a:1} → {b:1, a:1}

// - Tous documents doivent avoir les nouveaux champs
//   (ou migrer d'abord)

// ⚠️ Coûts:
// - Metadata updates (config.collections, config.chunks)
// - Possible rebalancing (migrations de chunks)
// - Index rebuild sur tous shards
// - Cache invalidation (tous mongos)

// Durée typique:
// - Petite collection (< 1 GB): Minutes
// - Moyenne collection (1-100 GB): Heures
// - Grosse collection (> 1 TB): Jours

// ───────────────────────────────────────────────────────
// EXEMPLE COMPLET
// ───────────────────────────────────────────────────────

// État initial:
sh.status()
// shard key: { product_id: 1 }
// Problem: Quelques produits populaires → jumbo chunks

// Étape 1: Ajouter timestamp aux documents si manquant
db.products.updateMany(
  { created_at: { $exists: false } },
  { $set: { created_at: new Date() } }
)

// Étape 2: Créer nouvel index
db.products.createIndex({ product_id: 1, created_at: 1 })

// Étape 3: Raffiner
db.adminCommand({
  refineCollectionShardKey: "shop.products",
  key: { product_id: 1, created_at: 1 }
})

// Output:
{
  "ok": 1,
  "oldKey": { "product_id": 1 },
  "newKey": { "product_id": 1, "created_at": 1 }
}

// Étape 4: Forcer split des jumbo chunks
db.adminCommand({
  split: "shop.products",
  middle: {
    product_id: "popular-product-123",
    created_at: ISODate("2024-06-01")
  }
})

// Les chunks se splitent maintenant sur created_at aussi
```

## Checklist de Décision

```
┌────────────────────────────────────────────────────────┐
│       CHECKLIST: CHOISIR UNE SHARD KEY                 │
├────────────────────────────────────────────────────────┤
│
│  PHASE 1: ANALYSE DES DONNÉES
│  ─────────────────────────────
│  ☐ Volumétrie actuelle et projection 2-5 ans
│  ☐ Taux de croissance (inserts/jour)
│  ☐ Distribution des valeurs (histogramme)
│  ☐ Identifier champs à haute cardinalité (> 10k)
│  ☐ Identifier champs immuables
│
│  PHASE 2: ANALYSE DES REQUÊTES
│  ─────────────────────────────
│  ☐ Activer profiler (dev/staging) 7 jours minimum
│  ☐ Identifier top 10 requêtes (fréquence)
│  ☐ Calculer % requêtes par type de filtre
│  ☐ Identifier pattern majoritaire (> 60%)
│  ☐ Vérifier présence de range queries
│
│  PHASE 3: ÉVALUATION CANDIDATS
│  ─────────────────────────────
│  Pour chaque candidat:
│
│  ☐ Cardinalité:
│     Mesure: db.collection.distinct("field").length
│     Cible: > 10k valeurs
│     Score: ___/10
│
│  ☐ Distribution:
│     Test: Simuler insertions récentes
│     Vérifier: Uniforme ou skew
│     Score: ___/10
│
│  ☐ Ciblabilité:
│     Mesure: % requêtes incluant le champ
│     Cible: > 70%
│     Score: ___/10
│
│  ☐ Score Global CWT: ___/30
│
│  PHASE 4: DÉCISION
│  ─────────────────────────────
│  ☐ Candidat #1: _______ (Score: __)
│  ☐ Candidat #2: _______ (Score: __)
│  ☐ Candidat #3: _______ (Score: __)
│
│  ☐ Meilleur candidat sélectionné: _____________
│
│  ☐ Type:
│     [ ] Simple
│     [ ] Composée (__ champs)
│     [ ] Hashed
│     [ ] Compound hashed
│
│  ☐ Valider compromis acceptables:
│     [ ] Scatter-gather pour ___% requêtes OK
│     [ ] Hotspot temporaire acceptable
│     [ ] Distribution 80/20 acceptable
│
│  PHASE 5: VALIDATION
│  ─────────────────────────────
│  ☐ Test sur staging avec données réelles
│  ☐ Load test (3× charge prod anticipée)
│  ☐ Mesurer distribution chunks
│  ☐ Mesurer latence P99
│  ☐ Vérifier absence jumbo chunks
│  ☐ Valider comportement balancer
│
│  PHASE 6: DOCUMENTATION
│  ─────────────────────────────
│  ☐ Documenter choix et rationale
│  ☐ Lister compromis acceptés
│  ☐ Définir métriques de monitoring
│  ☐ Planifier review post-déploiement (3 mois)
│
│  ☐ GO / NO-GO PRODUCTION
│
└────────────────────────────────────────────────────────┘
```

## Outils d'Analyse

```javascript
// ═══════════════════════════════════════════════════════
// SCRIPT D'ANALYSE DE CANDIDATS SHARD KEY
// ═══════════════════════════════════════════════════════

function analyzeShardKeyCandidate(collection, field) {
  print("═══════════════════════════════════════════")
  print("Analysing shard key candidate: " + field)
  print("Collection: " + collection)
  print("═══════════════════════════════════════════\n")

  const coll = db.getSiblingDB(collection.split('.')[0])
                 .getCollection(collection.split('.')[1])

  // ───────────────────────────────────────────────────
  // 1. CARDINALITÉ
  // ───────────────────────────────────────────────────
  print("1. CARDINALITÉ")
  print("─────────────────")

  const distinctCount = coll.distinct(field).length
  const totalDocs = coll.countDocuments()
  const cardinalityRatio = (distinctCount / totalDocs * 100).toFixed(2)

  print("Distinct values: " + distinctCount)
  print("Total documents: " + totalDocs)
  print("Ratio: " + cardinalityRatio + "%")

  let cardinalityScore
  if (distinctCount < 10) cardinalityScore = 0
  else if (distinctCount < 100) cardinalityScore = 2
  else if (distinctCount < 1000) cardinalityScore = 5
  else if (distinctCount < 10000) cardinalityScore = 7
  else cardinalityScore = 10

  print("Score: " + cardinalityScore + "/10")

  if (cardinalityScore < 5) {
    print("⚠️  WARNING: Cardinalité insuffisante pour sharding")
  }
  print("")

  // ───────────────────────────────────────────────────
  // 2. DISTRIBUTION
  // ───────────────────────────────────────────────────
  print("2. DISTRIBUTION")
  print("─────────────────")

  const distribution = coll.aggregate([
    { $sample: { size: 10000 } },
    { $group: { _id: "$" + field, count: { $sum: 1 } } },
    { $sort: { count: -1 } },
    { $limit: 10 }
  ]).toArray()

  print("Top 10 values distribution:")
  distribution.forEach((d, i) => {
    const pct = (d.count / 10000 * 100).toFixed(2)
    print(`  ${i+1}. Value: ${d._id}, Count: ${d.count} (${pct}%)`)
  })

  const topValuePct = distribution[0].count / 10000 * 100
  let distributionScore
  if (topValuePct > 50) distributionScore = 0  // Très skewed
  else if (topValuePct > 30) distributionScore = 3
  else if (topValuePct > 20) distributionScore = 5
  else if (topValuePct > 10) distributionScore = 7
  else distributionScore = 10  // Uniforme

  print("\nTop value represents: " + topValuePct.toFixed(2) + "%")
  print("Score: " + distributionScore + "/10")

  if (distributionScore < 5) {
    print("⚠️  WARNING: Distribution déséquilibrée (skew)")
  }
  print("")

  // ───────────────────────────────────────────────────
  // 3. MONOTONIE (pour timestamps)
  // ───────────────────────────────────────────────────
  if (field.match(/date|time|created|updated/i)) {
    print("3. MONOTONIE (Timestamp détecté)")
    print("─────────────────")

    const recent = coll.find()
                       .sort({ [field]: -1 })
                       .limit(1000)
                       .toArray()

    const recentValues = recent.map(d => d[field])
    const isMonotonic = recentValues.every((val, i) =>
      i === 0 || val <= recentValues[i-1]
    )

    print("Is monotonic increasing: " + isMonotonic)

    if (isMonotonic) {
      print("⚠️  WARNING: Clé monotone → HOTSPOT garanti!")
      print("   Recommandation: Utiliser hashed ou compound")
    }
    print("")
  }

  // ───────────────────────────────────────────────────
  // 4. RÉSUMÉ
  // ───────────────────────────────────────────────────
  const totalScore = cardinalityScore + distributionScore

  print("═══════════════════════════════════════════")
  print("RÉSUMÉ")
  print("═══════════════════════════════════════════")
  print("Cardinalité Score: " + cardinalityScore + "/10")
  print("Distribution Score: " + distributionScore + "/10")
  print("Total Score: " + totalScore + "/20")
  print("")

  if (totalScore >= 16) {
    print("✅ EXCELLENT candidat pour shard key")
  } else if (totalScore >= 12) {
    print("✅ BON candidat (avec réserves)")
  } else if (totalScore >= 8) {
    print("⚠️  ACCEPTABLE (compromis nécessaires)")
  } else {
    print("❌ MAUVAIS candidat - NE PAS UTILISER")
  }

  print("═══════════════════════════════════════════\n")

  return {
    field: field,
    cardinalityScore: cardinalityScore,
    distributionScore: distributionScore,
    totalScore: totalScore,
    distinctValues: distinctCount,
    recommendation: totalScore >= 12 ? "GOOD" : "BAD"
  }
}

// Utilisation:
analyzeShardKeyCandidate("ecommerce.orders", "customer_id")
analyzeShardKeyCandidate("ecommerce.orders", "status")
analyzeShardKeyCandidate("ecommerce.orders", "created_at")
```

## Résumé

Le choix de la **shard key** est la décision la plus critique en sharding MongoDB :

**Framework CWT (Les 3 Piliers) :**
1. **Cardinalité** : > 10k valeurs distinctes (idéal)
2. **Write Distribution** : Insertions uniformément réparties
3. **Targetability** : > 70% requêtes incluent la shard key

**Stratégies par Cas d'Usage :**
- **E-Commerce** : `{ customer_id: "hashed" }` ou `{ customer_id: 1, order_date: 1 }`
- **IoT/Time Series** : `{ device_id: 1, timestamp: 1 }` ou `{ device_id: "hashed", timestamp: 1 }`
- **Multi-Tenant SaaS** : `{ tenant_id: 1, document_id: 1 }` + Zone Sharding
- **Social Network** : `{ author_id: "hashed" }` ou `{ author_id: 1, created_at: 1 }`

**Anti-Patterns à Éviter :**
1. ❌ Monotone croissant sans mitigation (hotspot permanent)
2. ❌ Faible cardinalité (< 100 valeurs)
3. ❌ Shard key absente des requêtes (scatter-gather)
4. ❌ Shard key mutable (impact performance)
5. ❌ Trop complexe (> 3 champs)

**Processus de Décision :**
1. Analyser données (cardinalité, distribution)
2. Analyser requêtes (profiler 7+ jours)
3. Évaluer candidats (score CWT)
4. Tester en staging (load test)
5. Documenter et valider

**Raffinage (MongoDB 5.0+) :**
- Possible d'ajouter suffixes à la shard key
- Opération lourde mais online
- Dernier recours (mieux vaut bien choisir initialement)

> 💡 **Règle d'Or** : Investir 80% du temps de conception sur le choix de la shard key. C'est la décision la plus difficile à changer et la plus impactante sur les performances long-terme.

---


⏭️ [Types de sharding](/10-sharding/04-types-de-sharding.md)
