🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 10.4 Types de Sharding

## Introduction

MongoDB propose **trois stratégies fondamentales de partitionnement** (sharding) qui déterminent comment les chunks sont créés et distribués entre les shards. Chaque stratégie présente des compromis différents entre distribution uniforme, performance des requêtes, et complexité de gestion. Le choix du type de sharding dépend des patterns d'accès aux données, des contraintes métier, et des objectifs de performance.

Cette section présente une vue d'ensemble comparative des trois types de sharding avant d'explorer chacun en détail dans les sous-sections suivantes.

## Les Trois Types de Sharding

```
┌─────────────────────────────────────────────────────────────┐
│           TYPES DE SHARDING MONGODB                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. RANGE SHARDING (Partitionnement par Plage)              │
│     ═══════════════════════════════════════════             │
│     Chunks basés sur des plages de valeurs contiguës        │
│                                                             │
│     Shard Key: { customer_id: 1 }                           │
│                                                             │
│     ┌─────────────┬─────────────┬─────────────┐             │
│     │  Chunk 1    │  Chunk 2    │  Chunk 3    │             │
│     │ [0, 1000)   │[1000, 2000) │[2000, 3000) │             │
│     │  Shard A    │  Shard B    │  Shard C    │             │
│     └─────────────┴─────────────┴─────────────┘             │
│                                                             │
│     ✅ Requêtes par plage efficaces                         │
│     ❌ Risque de hotspots si monotone                       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  2. HASHED SHARDING (Partitionnement par Hachage)           │
│     ═══════════════════════════════════════════════         │
│     Chunks basés sur hash de la shard key                   │
│                                                             │
│     Shard Key: { _id: "hashed" }                            │
│                                                             │
│     ┌─────────────┬─────────────┬─────────────┐             │
│     │  Chunk 1    │  Chunk 2    │  Chunk 3    │             │
│     │hash: [min,x)│hash: [x, y) │hash: [y,max]│             │
│     │  Shard A    │  Shard B    │  Shard C    │             │
│     └─────────────┴─────────────┴─────────────┘             │
│                                                             │
│     ✅ Distribution uniforme garantie                       │
│     ❌ Requêtes par plage → scatter-gather                  │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  3. ZONE SHARDING (Partitionnement par Zone)                │
│     ═══════════════════════════════════════════             │
│     Chunks assignés à des zones spécifiques                 │
│                                                             │
│     Shard Key: { country: 1, user_id: 1 }                   │
│                                                             │
│     Zone EU:  ┌─────────────┬─────────────┐                 │
│               │France chunks│ Germany chks│                 │
│               │  Shard EU-1 │  Shard EU-2 │                 │
│               └─────────────┴─────────────┘                 │
│                                                             │
│     Zone US:  ┌─────────────┬─────────────┐                 │
│               │  US chunks  │  CA chunks  │                 │
│               │  Shard US-1 │  Shard US-2 │                 │
│               └─────────────┴─────────────┘                 │
│                                                             │
│     ✅ Conformité géographique (RGPD)                       │
│     ❌ Complexité de configuration                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Comparaison Matricielle

### Tableau Comparatif Général

```
┌──────────────┬───────────────┬───────────────┬──────────────┐
│  Critère     │ Range Sharding│Hashed Sharding│Zone Sharding │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Distribution │     ⚠️        │      ✅✅     │      ⚠️      │
│ Uniforme     │ Dépend données│   Garantie    │  Contrôlée   │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Requêtes     │               │               │              │
│ par Plage    │      ✅✅     │      ❌       │      ✅      │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Requêtes     │               │               │              │
│ Exactes      │      ✅       │      ✅       │      ✅      │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Hotspots     │               │               │              │
│ Risque       │      ⚠️       │      ✅       │      ⚠️      │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Complexité   │               │               │              │
│ Setup        │     Faible    │    Faible     │    Élevée    │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Complexité   │               │               │              │
│ Maintenance  │     Faible    │    Faible     │    Élevée    │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Prévisibilité│               │               │              │
│ Localisation │      ✅✅     │      ❌       │      ✅✅    │
├──────────────┼───────────────┼───────────────┼──────────────┤
│ Cas d'Usage  │  Time-series, │  High-volume  │ Multi-tenant,│
│ Principal    │  Logs, IoT    │  Inserts      │ Géo-distribué│
└──────────────┴───────────────┴───────────────┴──────────────┘
```

### Performance Comparée

```
┌─────────────────────────────────────────────────────────────┐
│             PERFORMANCE PAR TYPE DE REQUÊTE                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  REQUÊTE PAR PLAGE (Range Query)                            │
│  ──────────────────────────────────                         │
│                                                             │
│  db.orders.find({ customer_id: { $gte: 1000, $lte: 2000 }})│
│                                                             │
│  Range Sharding:   ⭐⭐⭐⭐⭐ (Optimal)                     │
│    → Ciblé vers chunks dans la plage                        │
│    → 1-3 shards typiquement                                 │
│    → Latence: 5-15ms                                        │
│                                                             │
│  Hashed Sharding:  ⭐ (Très mauvais)                        │
│    → Scatter-gather OBLIGATOIRE                             │
│    → TOUS les shards interrogés                             │
│    → Latence: 50-200ms                                      │
│                                                             │
│  Zone Sharding:    ⭐⭐⭐⭐ (Bon)                           │
│    → Dépend de la zone                                      │
│    → Quelques shards dans la zone                           │
│    → Latence: 10-30ms                                       │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  REQUÊTE EXACTE (Point Query)                               │
│  ─────────────────────────────                              │
│                                                             │
│  db.orders.find({ customer_id: 1500 })                      │
│                                                             │
│  Range Sharding:   ⭐⭐⭐⭐⭐ (Optimal)                     │
│    → 1 seul chunk                                           │
│    → 1 seul shard                                           │
│    → Latence: 5ms                                           │
│                                                             │
│  Hashed Sharding:  ⭐⭐⭐⭐⭐ (Optimal)                     │
│    → 1 seul chunk (calculé par hash)                        │
│    → 1 seul shard                                           │
│    → Latence: 5ms                                           │
│                                                             │
│  Zone Sharding:    ⭐⭐⭐⭐⭐ (Optimal)                     │
│    → 1 seul chunk dans une zone                             │
│    → 1 seul shard                                           │
│    → Latence: 5ms                                           │
│                                                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  INSERTIONS (Write Performance)                             │
│  ─────────────────────────────                              │
│                                                             │
│  Range Sharding:   ⭐⭐⭐ (Variable)                        │
│    → Si monotone: Hotspot ❌                                │
│    → Si aléatoire: Bon ✅                                   │
│    → Throughput: Dépend du pattern                          │
│                                                             │
│  Hashed Sharding:  ⭐⭐⭐⭐⭐ (Optimal)                     │
│    → Distribution parfaite                                  │
│    → Pas de hotspot                                         │
│    → Throughput: Linéaire avec shards                       │
│                                                             │
│  Zone Sharding:    ⭐⭐⭐ (Variable)                        │
│    → Dépend de la distribution par zone                     │
│    → Risque de déséquilibre entre zones                     │
│    → Throughput: Dépend des zones actives                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Range Sharding : Vue d'Ensemble

### Principe de Fonctionnement

```javascript
// Configuration Range Sharding
sh.shardCollection("analytics.events", { timestamp: 1 })

// MongoDB divise en chunks de plages contiguës:
// Chunk 1: [MinKey, 2024-01-01)         → Shard A
// Chunk 2: [2024-01-01, 2024-02-01)     → Shard B
// Chunk 3: [2024-02-01, 2024-03-01)     → Shard C
// Chunk 4: [2024-03-01, MaxKey]         → Shard A
```

### Visualisation de la Distribution

```
Range Sharding sur customer_id
════════════════════════════════════════════════════════

Shard Key Values: 0 ────────────────────────────── 10000

├──────────┬──────────┬──────────┬──────────┬──────────┤
│ Chunk 1  │ Chunk 2  │ Chunk 3  │ Chunk 4  │ Chunk 5  │
│ [0,2000) │[2k,4k)   │[4k,6k)   │[6k,8k)   │[8k,10k]  │
│ Shard A  │ Shard B  │ Shard C  │ Shard A  │ Shard B  │
└──────────┴──────────┴──────────┴──────────┴──────────┘

Requête: customer_id BETWEEN 3000 AND 5000
         └───────────────────────────────┘
              Touches: Chunk 2 et Chunk 3
              Shards interrogés: B et C (2 shards)
```

### Avantages et Inconvénients

```
✅ AVANTAGES:
   • Requêtes par plage TRÈS efficaces
   • Localisation prévisible des données
   • Bon pour séries temporelles
   • Simplicité conceptuelle
   • Facilite archivage (drop chunks anciens)

❌ INCONVÉNIENTS:
   • Hotspots si shard key monotone
   • Distribution peut être inégale
   • Nécessite analyse de la distribution
   • Gros clients/valeurs → Jumbo chunks
```

### Cas d'Usage Idéaux

```javascript
// 1. Time-Series Data
sh.shardCollection("iot.metrics",
  { device_id: 1, timestamp: 1 }
)
// Range queries fréquentes sur timestamp

// 2. Logs avec Archivage
sh.shardCollection("app.logs",
  { date: 1, server_id: 1 }
)
// Permet de drop facilement chunks anciens

// 3. Sequential IDs (avec préfixe)
sh.shardCollection("orders.transactions",
  { region: 1, order_id: 1 }
)
// order_id séquentiel mais préfixé par region
```

## Hashed Sharding : Vue d'Ensemble

### Principe de Fonctionnement

```javascript
// Configuration Hashed Sharding
sh.shardCollection("users.profiles", { _id: "hashed" })

// MongoDB calcule hash(valeur) et crée chunks de hash ranges:
// Chunk 1: [hash_min, hash_x)           → Shard A
// Chunk 2: [hash_x, hash_y)             → Shard B
// Chunk 3: [hash_y, hash_max]           → Shard C

// Exemple:
// _id: "user-12345" → hash → -4567891234567890123 → Chunk 1 → Shard A
// _id: "user-67890" → hash →  8901234567890123456 → Chunk 3 → Shard C
```

### Visualisation de la Distribution

```
Hashed Sharding sur _id
════════════════════════════════════════════════════════

Hash Range: -9223372036854775808 ────────── 9223372036854775807
            (int64 min)                      (int64 max)

├────────────────┬────────────────┬────────────────┬────────────┤
│    Chunk 1     │    Chunk 2     │    Chunk 3     │  Chunk 4   │
│ [min, -5e18)   │[-5e18, 0)      │[0, 5e18)       │[5e18, max] │
│    Shard A     │    Shard B     │    Shard C     │   Shard D  │
└────────────────┴────────────────┴────────────────┴────────────┘

Distribution des insertions:
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Shard A  │  │ Shard B  │  │ Shard C  │  │ Shard D  │
│ ████████ │  │ ████████ │  │ ████████ │  │ ████████ │
│ 25%      │  │ 25%      │  │ 25%      │  │ 25%      │
└──────────┘  └──────────┘  └──────────┘  └──────────┘
     PARFAITEMENT UNIFORME (garantie mathématique)
```

### Avantages et Inconvénients

```
✅ AVANTAGES:
   • Distribution uniforme GARANTIE
   • Élimine complètement hotspots monotones
   • Excellent pour high-volume inserts
   • Simplicité opérationnelle
   • Prévisibilité performance

❌ INCONVÉNIENTS:
   • Range queries → TOUJOURS scatter-gather
   • Perte de localité des données
   • Impossible de drop chunks par plage
   • Archivage compliqué
   • $in queries moins efficaces
```

### Cas d'Usage Idéaux

```javascript
// 1. High-Volume Writes avec _id monotone
sh.shardCollection("logs.events", { _id: "hashed" })
// Élimine hotspot du ObjectId monotone

// 2. User Profiles (accès par ID)
sh.shardCollection("users.accounts", { user_id: "hashed" })
// Requêtes principalement par user_id exact

// 3. Cache distribué
sh.shardCollection("cache.sessions", { session_id: "hashed" })
// Accès aléatoire, pas de range queries
```

## Zone Sharding : Vue d'Ensemble

### Principe de Fonctionnement

```javascript
// Configuration Zone Sharding
sh.shardCollection("app.users", { country: 1, user_id: 1 })

// Définir zones géographiques
sh.addShardToZone("shard-eu-1", "EU")
sh.addShardToZone("shard-eu-2", "EU")
sh.addShardToZone("shard-us-1", "US")
sh.addShardToZone("shard-us-2", "US")

// Assigner ranges à zones
sh.updateZoneKeyRange("app.users",
  { country: "FR", user_id: MinKey },
  { country: "FR", user_id: MaxKey },
  "EU"
)

sh.updateZoneKeyRange("app.users",
  { country: "US", user_id: MinKey },
  { country: "US", user_id: MaxKey },
  "US"
)
```

### Visualisation de la Distribution

```
Zone Sharding par Géographie
════════════════════════════════════════════════════════

┌───────────────────────────────────────────────────────┐
│  ZONE: EU (Paris)                                     │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │  Shard EU-1  │  │  Shard EU-2  │                   │
│  └──────────────┘  └──────────────┘                   │
│                                                       │
│  Chunks:                                              │
│  • FR users: [FR, 0] to [FR, ∞]                       │
│  • DE users: [DE, 0] to [DE, ∞]                       │
│  • IT users: [IT, 0] to [IT, ∞]                       │
│                                                       │
│  Latence app→shard: 1-5ms (local)                     │
└───────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────┐
│  ZONE: US (Virginia)                                  │
│  ┌──────────────┐  ┌──────────────┐                   │
│  │  Shard US-1  │  │  Shard US-2  │                   │
│  └──────────────┘  └──────────────┘                   │
│                                                       │
│  Chunks:                                              │
│  • US users: [US, 0] to [US, ∞]                       │
│  • CA users: [CA, 0] to [CA, ∞]                       │
│                                                       │
│  Latence app→shard: 1-5ms (local)                     │
└───────────────────────────────────────────────────────┘

Requête depuis app EU:
  find({ country: "FR", user_id: 12345 })
  → Routé vers Shard EU-1 ou EU-2 (local)
  → Latence totale: ~5ms
```

### Avantages et Inconvénients

```
✅ AVANTAGES:
   • Conformité réglementaire (RGPD, résidence données)
   • Latence optimale (données près utilisateurs)
   • Isolation multi-tenant
   • SLA différenciés possibles
   • Contrôle fin de la distribution

❌ INCONVÉNIENTS:
   • Configuration complexe
   • Maintenance continue nécessaire
   • Risque de déséquilibre entre zones
   • Requêtes cross-zone coûteuses
   • Overhead opérationnel
```

### Cas d'Usage Idéaux

```javascript
// 1. Multi-Région avec RGPD
sh.shardCollection("gdpr.user_data",
  { country: 1, user_id: 1 }
)
// Données EU restent en EU

// 2. Multi-Tenant avec Isolation
sh.shardCollection("saas.documents",
  { tenant_id: 1, doc_id: 1 }
)
// Whale tenants sur shards dédiés

// 3. Tiering (Hot/Warm/Cold)
sh.shardCollection("archive.documents",
  { year: 1, month: 1, doc_id: 1 }
)
// Années récentes sur SSD, anciennes sur HDD
```

## Sharding Hybride (Compound Hashed)

Depuis MongoDB 4.4, il est possible de combiner hashed et range sharding.

### Principe

```javascript
// Shard Key: Premier champ hashed, autres range
sh.shardCollection("iot.events",
  { device_id: "hashed", timestamp: 1 }
)

// Fonctionnement:
// 1. device_id haché → Distribution uniforme entre shards
// 2. timestamp en range → Queries temporelles efficaces par device
```

### Visualisation

```
Compound Hashed: { device_id: "hashed", timestamp: 1 }
════════════════════════════════════════════════════════════

Hash(device_id) distribue entre shards:
├─────────────────┬─────────────────┬─────────────────┐
│    Shard A      │    Shard B      │    Shard C      │
│ hash: [min, x)  │ hash: [x, y)    │ hash: [y, max]  │
└─────────────────┴─────────────────┴─────────────────┘

Sur chaque shard, chunks organisés par timestamp:
Shard A:
  ├─ [dev_hash_A, 2024-01-01] to [dev_hash_A, 2024-02-01]
  ├─ [dev_hash_A, 2024-02-01] to [dev_hash_A, 2024-03-01]
  └─ [dev_hash_B, 2024-01-01] to [dev_hash_B, 2024-02-01]

Avantages:
✅ Distribution uniforme (device_id haché)
✅ Range queries efficaces (timestamp) par device
```

### Exemple Pratique

```javascript
// Collection IoT avec millions de devices
sh.shardCollection("sensors.readings",
  { device_id: "hashed", timestamp: 1 }
)

// Query 1: Lectures d'un device (ciblée)
db.readings.find({
  device_id: "sensor-12345",
  timestamp: { $gte: ISODate("2024-12-01") }
})
// → 1 seul shard (hash de device_id)
// → Range scan efficace sur timestamp

// Query 2: Toutes lectures récentes (scatter-gather)
db.readings.find({
  timestamp: { $gte: ISODate("2024-12-01") }
})
// → TOUS les shards (timestamp sans device_id)

// Insertions: Distribution parfaite
// Chaque device_id haché vers shard différent
// → Pas de hotspot même si timestamps monotones
```

## Matrice de Décision

### Critères de Choix

```
┌────────────────────────────────────────────────────────────┐
│         ARBRE DE DÉCISION: QUEL TYPE CHOISIR ?             │
└────────────────────────────────────────────────────────────┘

Question 1: Vos requêtes incluent-elles des range queries ?
│
├─ NON (seulement point queries / $in)
│  └─→ HASHED SHARDING
│      • Distribution optimale
│      • Simplicité
│
└─ OUI (range queries fréquentes)
   │
   Question 2: Vos insertions sont-elles monotones ?
   │
   ├─ NON (aléatoires ou bien distribuées)
   │  └─→ RANGE SHARDING
   │      • Range queries efficaces
   │      • Bonne distribution
   │
   └─ OUI (timestamp, auto-increment, ObjectId)
      │
      Question 3: Pouvez-vous ajouter préfixe non-monotone ?
      │
      ├─ OUI (ex: device_id, region, user_id)
      │  │
      │  Question 4: Besoin de distribution garantie ?
      │  │
      │  ├─ OUI
      │  │  └─→ COMPOUND HASHED
      │  │      { field: "hashed", timestamp: 1 }
      │  │
      │  └─ NON
      │     └─→ RANGE COMPOUND
      │         { field: 1, timestamp: 1 }
      │
      └─ NON
         └─→ HASHED SHARDING
             (sacrifice range queries)

Question 5: Avez-vous besoin d'isolation géographique/tenant ?
│
└─ OUI
   └─→ ZONE SHARDING
       • Sur base de Range ou Hashed
       • Configuration zones
```

### Tableau de Scoring

```
┌─────────────────┬────────┬────────┬────────┬───────────┐
│   Critère       │ Range  │ Hashed │  Zone  │ Compound  │
│                 │        │        │        │  Hashed   │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ Range Queries   │   10   │   0    │   8    │     9     │
│ Efficaces       │        │        │        │           │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ Distribution    │   5    │   10   │   6    │    10     │
│ Uniforme        │        │        │        │           │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ Pas de Hotspot  │   4    │   10   │   5    │    10     │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ Simplicité      │   9    │   10   │   3    │     7     │
│ Config          │        │        │        │           │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ Simplicité      │   9    │   10   │   5    │     7     │
│ Maintenance     │        │        │        │           │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ Localité        │   10   │   0    │   10   │     5     │
│ Données         │        │        │        │           │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ Conformité      │   5    │   3    │   10   │     6     │
│ Géo/Tenant      │        │        │        │           │
├─────────────────┼────────┼────────┼────────┼───────────┤
│ TOTAL           │  52/70 │  43/70 │  47/70 │   54/70   │
└─────────────────┴────────┴────────┴────────┴───────────┘

Interprétation:
• 60-70: Excellent pour le cas d'usage
• 50-59: Bon choix
• 40-49: Acceptable avec compromis
• < 40:  Probablement pas adapté
```

## Combinaisons et Patterns Avancés

### Pattern 1 : Range avec Zone (Géo + Time)

```javascript
// Cas: Application globale avec time-series
sh.shardCollection("metrics.global",
  { region: 1, timestamp: 1 }
)

// Zones géographiques
sh.addShardToZone("shard-eu", "EU")
sh.addShardToZone("shard-us", "US")
sh.addShardToZone("shard-asia", "ASIA")

// Chunks EU restent en EU
sh.updateZoneKeyRange("metrics.global",
  { region: "EU", timestamp: MinKey },
  { region: "EU", timestamp: MaxKey },
  "EU"
)

// Avantages:
// ✅ Range queries efficaces sur timestamp
// ✅ Conformité RGPD (données locales)
// ✅ Latence optimale
```

### Pattern 2 : Hashed avec Zone (Distribution + Isolation)

```javascript
// Cas: SaaS multi-tenant avec whale tenants
sh.shardCollection("saas.data",
  { tenant_id: "hashed", doc_id: 1 }
)

// Zone pour whale tenants
sh.addShardToZone("shard-premium", "WHALE")
sh.addShardToZone("shard-standard-1", "STANDARD")
sh.addShardToZone("shard-standard-2", "STANDARD")

// BigCorp a son propre hash range
sh.updateZoneKeyRange("saas.data",
  { tenant_id: hash("BigCorp"), doc_id: MinKey },
  { tenant_id: hash("BigCorp"), doc_id: MaxKey },
  "WHALE"
)

// Avantages:
// ✅ Distribution uniforme (hashed)
// ✅ Isolation whale tenant
// ✅ SLA différencié possible
```

### Pattern 3 : Tiering Temporel avec Zone

```javascript
// Cas: Archive avec tiers de stockage
sh.shardCollection("archive.documents",
  { year: 1, month: 1, doc_id: 1 }
)

// Zones par tier
sh.addShardToZone("shard-nvme-1", "HOT")   // 2024
sh.addShardToZone("shard-ssd-1", "WARM")   // 2023
sh.addShardToZone("shard-hdd-1", "COLD")   // 2022 et avant

// HOT: Année courante sur NVMe
sh.updateZoneKeyRange("archive.documents",
  { year: 2024, month: 1, doc_id: MinKey },
  { year: 2024, month: 12, doc_id: MaxKey },
  "HOT"
)

// WARM: Année précédente sur SSD
sh.updateZoneKeyRange("archive.documents",
  { year: 2023, month: 1, doc_id: MinKey },
  { year: 2023, month: 12, doc_id: MaxKey },
  "WARM"
)

// COLD: Archives sur HDD
sh.updateZoneKeyRange("archive.documents",
  { year: MinKey, month: MinKey, doc_id: MinKey },
  { year: 2022, month: 12, doc_id: MaxKey },
  "COLD"
)

// Migration annuelle:
// - Janvier 2025: 2024 → WARM, 2023 → COLD
```

## Anti-Patterns par Type

### 🚫 Anti-Pattern 1 : Range sur Monotone Sans Mitigation

```javascript
// ❌ ANTI-PATTERN
sh.shardCollection("events.logs", { timestamp: 1 })

// Problème:
// TOUS les inserts vont au chunk avec MaxKey
// = 1 seul shard actif en écriture
// = Hotspot permanent
// = Scaling horizontal inutile

// Impact mesuré:
// - Shard actif: 50k writes/sec (CPU 100%)
// - Autres shards: 0 writes/sec (idle)

// Solution: Compound avec préfixe non-monotone
sh.shardCollection("events.logs",
  { source_id: 1, timestamp: 1 }  // ✅
)
```

### 🚫 Anti-Pattern 2 : Hashed sur Range-Heavy Workload

```javascript
// ❌ ANTI-PATTERN
sh.shardCollection("timeseries.metrics", { timestamp: "hashed" })

// Requêtes typiques:
db.metrics.find({
  timestamp: { $gte: ISODate("2024-12-07") }
})
// → Scatter-gather SYSTÉMATIQUE

// Impact:
// - 100% des requêtes → tous shards
// - Latence × N shards
// - Network × N shards

// Solution: Range sharding approprié
sh.shardCollection("timeseries.metrics",
  { device_id: 1, timestamp: 1 }  // ✅
)
```

### 🚫 Anti-Pattern 3 : Zone Sharding Sans Monitoring

```javascript
// ❌ ANTI-PATTERN
// Configurer zones et oublier

sh.shardCollection("app.users", { country: 1, user_id: 1 })
sh.addShardToZone("shard-us", "US")
sh.addShardToZone("shard-eu", "EU")
// ... Configuration initiale

// 6 mois plus tard:
// - 90% des users sont US → Shards US surchargés
// - 10% des users sont EU → Shards EU idle
// - Déséquilibre non détecté

// Solution: Monitoring continu
db.getSiblingDB("config").chunks.aggregate([
  { $match: { ns: "app.users" } },
  { $group: { _id: "$shard", count: { $sum: 1 } } }
])

// + Alertes si déséquilibre > 20%
```

### 🚫 Anti-Pattern 4 : Mélanger Types Incompatibles

```javascript
// ❌ ANTI-PATTERN
// Range sharding initial
sh.shardCollection("data.collection", { field_a: 1 })

// Plus tard: Tenter d'ajouter hashing
// IMPOSSIBLE! Pas de migration automatique

// Solution: Décider le type AVANT sharding
// Si besoin de changer: Re-shard complet nécessaire
```

### 🚫 Anti-Pattern 5 : Zones Trop Granulaires

```javascript
// ❌ ANTI-PATTERN
// 1 zone par ville (des centaines de zones)
sh.addShardToZone("shard-001", "Paris")
sh.addShardToZone("shard-002", "Lyon")
sh.addShardToZone("shard-003", "Marseille")
// ... 100+ zones

// Problèmes:
// - Configuration cauchemardesque
// - Maintenance impossible
// - Overhead metadata énorme
// - Balancer confus

// Solution: Zones au niveau pays/région
sh.addShardToZone("shard-eu-1", "EU")
sh.addShardToZone("shard-eu-2", "EU")
// Max 5-10 zones total
```

## Considérations de Migration

### Changer de Type de Sharding

```
┌────────────────────────────────────────────────────────┐
│     MIGRATION ENTRE TYPES (Complexité)                 │
├────────────────────────────────────────────────────────┤
│
│  Non-Shardé → Range:        ⭐⭐ Facile
│  Non-Shardé → Hashed:       ⭐⭐ Facile
│  Non-Shardé → Zone:         ⭐⭐⭐ Moyen
│
│  Range → Hashed:            ❌ Impossible*
│  Range → Zone:              ⭐⭐⭐⭐ Complexe
│  Hashed → Range:            ❌ Impossible*
│  Hashed → Zone:             ⭐⭐⭐⭐ Complexe
│  Zone → Range/Hashed:       ⭐⭐⭐ Moyen
│
│  * Nécessite re-shard complet (export/import)
└────────────────────────────────────────────────────────┘
```

### Procédure de Re-Sharding

```javascript
// Si changement de type nécessaire (dernier recours):

// Étape 1: Créer nouvelle collection avec nouveau type
sh.shardCollection("data.collection_v2", { new_key: "hashed" })

// Étape 2: Migration dual-write
// Application écrit dans les deux collections

// Étape 3: Backfill historique
// Script pour copier données anciennes

// Étape 4: Validation
// Vérifier cohérence

// Étape 5: Cutover
// Basculer lectures vers nouvelle collection

// Étape 6: Cleanup
// Supprimer ancienne collection

// Durée typique: Jours à semaines selon volume
```

## Résumé

Les **trois types de sharding** MongoDB offrent des compromis différents :

**Range Sharding :**
- ✅ Range queries efficaces
- ✅ Localité des données
- ❌ Risque de hotspots
- 🎯 Idéal pour : Time-series, logs, données séquentielles

**Hashed Sharding :**
- ✅ Distribution uniforme garantie
- ✅ Pas de hotspots
- ❌ Range queries scatter-gather
- 🎯 Idéal pour : High-volume inserts, accès par clé exacte

**Zone Sharding :**
- ✅ Conformité géographique
- ✅ Isolation multi-tenant
- ❌ Complexité configuration
- 🎯 Idéal pour : Multi-région, whale tenants, tiering

**Compound Hashed (MongoDB 4.4+) :**
- ✅ Distribution + Range queries
- ✅ Meilleur des deux mondes
- 🎯 Idéal pour : IoT, time-series distribués

**Critères de décision :**
1. Analyser patterns de requête (range vs point)
2. Évaluer distribution des insertions (monotone vs aléatoire)
3. Considérer contraintes métier (géo, tenant)
4. Tester en staging avec données réelles

**Anti-patterns critiques :**
- ❌ Range sur monotone sans préfixe
- ❌ Hashed sur workload range-heavy
- ❌ Zones sans monitoring
- ❌ Sur-granularité des zones

Le choix du type de sharding est une décision architecturale majeure qui doit être prise en fonction des caractéristiques spécifiques du workload et des données.

---


⏭️ [Range Sharding](/10-sharding/04.1-range-sharding.md)
