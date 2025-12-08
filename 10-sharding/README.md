🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 10 : Sharding (Partitionnement Horizontal)

## Introduction

Le **sharding** représente la solution de mise à l'échelle horizontale de MongoDB, permettant de distribuer les données sur plusieurs serveurs pour gérer des volumes massifs et des charges de travail intensives. Ce chapitre s'adresse à un niveau avancé et nécessite une maîtrise solide de la réplication (Replica Sets), car chaque shard dans une architecture shardée est typiquement implémenté comme un Replica Set.

### Pourquoi le Sharding ?

Lorsqu'une base de données atteint ses limites en termes de stockage, de mémoire ou de capacité de traitement, deux approches s'offrent aux architectes :

1. **Mise à l'échelle verticale (Scale-up)** : Augmenter les ressources matérielles d'un serveur unique
   - ✅ Simple à mettre en œuvre
   - ❌ Coûts exponentiels
   - ❌ Limites physiques infranchissables
   - ❌ Point de défaillance unique

2. **Mise à l'échelle horizontale (Scale-out)** : Distribuer les données sur plusieurs serveurs
   - ✅ Croissance quasi-linéaire
   - ✅ Coûts plus prévisibles
   - ✅ Haute disponibilité
   - ❌ Complexité architecturale accrue

Le sharding MongoDB implémente la deuxième approche en partitionnant automatiquement les données selon une clé de partitionnement (shard key).

### Principes Fondamentaux

Le sharding repose sur trois piliers conceptuels :

**1. Partitionnement des données**
Les collections sont divisées en **chunks** (morceaux) de données distribués sur différents shards. Chaque chunk contient un sous-ensemble des documents basé sur la valeur de la shard key.

**2. Distribution équilibrée**
MongoDB maintient automatiquement une distribution équilibrée des chunks entre les shards via un processus appelé **balancing**.

**3. Routage transparent**
Les applications interagissent avec le cluster via des **mongos** (routeurs) qui dirigent les requêtes vers les shards appropriés sans que le client n'ait à connaître la topologie du cluster.

### Architecture en un Coup d'Œil

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Cliente                    │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────┐
            │     Mongos (Routeurs)      │
            │   Query Router Layer       │
            └────────┬──────────┬────────┘
                     │          │
         ┌───────────┴──┐    ┌──┴───────────┐
         │              │    │              │
         ▼              ▼    ▼              ▼
    ┌────────┐     ┌────────┐         ┌────────┐
    │ Shard  │     │ Shard  │   ...   │ Shard  │
    │   A    │     │   B    │         │   N    │
    │(RS*)   │     │(RS*)   │         │(RS*)   │
    └────────┘     └────────┘         └────────┘
         ▲              ▲                  ▲
         │              │                  │
         └──────────────┴──────────────────┘
                        │
                        ▼
              ┌──────────────────┐
              │  Config Servers  │
              │  (Metadata RS)   │
              └──────────────────┘

* RS = Replica Set
```

### Quand Sharder ?

Le sharding n'est pas une solution universelle. Évaluez ces indicateurs :

#### ✅ Vous DEVRIEZ sharder si :

- **Volume de données** : Vos données dépassent la capacité de stockage d'un serveur unique (plusieurs To)
- **Débit d'écriture** : Le volume d'insertions/updates dépasse la capacité d'un nœud primaire
- **Ensemble de travail** : Le working set dépasse la RAM disponible sur un serveur
- **Distribution géographique** : Vous devez placer les données près des utilisateurs (latence)
- **Isolation des tenants** : Architecture multi-tenant nécessitant une isolation par shard

#### ❌ Vous NE DEVRIEZ PAS sharder si :

- Vos données tiennent confortablement sur un serveur (< 500 Go)
- Un Replica Set suffit à gérer votre charge
- Vous n'avez pas l'expertise pour gérer la complexité
- Vos requêtes sont majoritairement scatter-gather (inefficaces en sharding)
- Vous pouvez optimiser via indexation ou modélisation

### Stratégies de Partitionnement : Vue d'Ensemble

Le choix de la **shard key** est la décision la plus critique d'une architecture shardée. Une mauvaise shard key peut :
- Créer des hotspots (shards surchargés)
- Générer des jumbo chunks impossibles à déplacer
- Provoquer des scatter-gather queries systématiques
- Rendre le balancing inefficace

MongoDB propose trois stratégies principales de sharding :

#### 1. **Range Sharding** (Partitionnement par plage)

```
Shard Key: { "date": 1 }

Shard A:  [MinKey, 2024-01-01)
Shard B:  [2024-01-01, 2024-06-01)
Shard C:  [2024-06-01, MaxKey]
```

**Caractéristiques :**
- ✅ Requêtes par plage très efficaces
- ✅ Bon pour les séries temporelles
- ❌ Risque de hotspots sur les insertions monotones
- ❌ Distribution inégale si les données ne sont pas uniformes

**Exemple d'utilisation :**
```javascript
// Collection de logs avec accès principalement par période
sh.shardCollection("logs.events", { "timestamp": 1 })
```

**Anti-pattern :**
```javascript
// ❌ MAUVAIS : timestamp seul crée un hotspot sur le shard le plus récent
sh.shardCollection("events.realtime", { "created_at": 1 })

// Toutes les nouvelles insertions vont au même shard (le plus récent)
```

#### 2. **Hashed Sharding** (Partitionnement par hachage)

```
Shard Key: { "_id": "hashed" }

Shard A:  [hash_min, hash_x)
Shard B:  [hash_x, hash_y)
Shard C:  [hash_y, hash_max]
```

**Caractéristiques :**
- ✅ Distribution uniforme garantie
- ✅ Excellent pour les insertions à haut débit
- ✅ Élimine les hotspots monotones
- ❌ Les requêtes par plage deviennent scatter-gather
- ❌ Impossible de cibler un shard spécifique

**Exemple d'utilisation :**
```javascript
// Collection avec insertions uniformes nécessaires
sh.shardCollection("users.profiles", { "_id": "hashed" })
```

**Anti-pattern :**
```javascript
// ❌ MAUVAIS : Hashed sharding sur une clé utilisée pour les requêtes par plage
sh.shardCollection("products.catalog", { "price": "hashed" })

// Les requêtes "find tous les produits entre 10€ et 50€"
// deviennent scatter-gather (tous les shards interrogés)
```

#### 3. **Zone Sharding** (Partitionnement par zone)

```
Zone "EU":     Shard A, Shard B  →  { "country": { $in: ["FR", "DE", "IT"] } }
Zone "US":     Shard C, Shard D  →  { "country": { $in: ["US", "CA"] } }
Zone "ASIA":   Shard E, Shard F  →  { "country": { $in: ["JP", "CN", "IN"] } }
```

**Caractéristiques :**
- ✅ Conformité réglementaire (RGPD, résidence des données)
- ✅ Optimisation de la latence (données près des utilisateurs)
- ✅ Isolation multi-tenant
- ❌ Configuration et maintenance complexes
- ❌ Risque de déséquilibre entre zones

**Exemple d'utilisation :**
```javascript
// Sharding géographique avec zones
sh.shardCollection("app.users", { "country": 1, "user_id": 1 })

// Configuration des zones
sh.addShardToZone("shard-eu-01", "EU")
sh.addShardToZone("shard-eu-02", "EU")
sh.updateZoneKeyRange("app.users",
  { country: "FR", user_id: MinKey },
  { country: "FR", user_id: MaxKey },
  "EU"
)
```

**Anti-pattern :**
```javascript
// ❌ MAUVAIS : Zones sans granularité suffisante
sh.shardCollection("orders.transactions", { "region": 1 })

// Si 90% des transactions sont dans la région "US",
// les shards US seront surchargés
```

### Comparaison des Stratégies

| Critère | Range | Hashed | Zone |
|---------|-------|--------|------|
| **Distribution** | Inégale potentielle | Uniforme | Contrôlée |
| **Requêtes ciblées** | Excellentes (plages) | Mauvaises (scatter) | Bonnes (par zone) |
| **Insertions uniformes** | Risque de hotspot | Excellentes | Dépend de la distribution |
| **Complexité** | Faible | Faible | Élevée |
| **Cas d'usage typique** | Time-series, logs | High-throughput writes | Multi-tenant, RGPD |

### Anti-Patterns Critiques à Éviter

#### 🚫 **Anti-Pattern 1 : Shard Key à faible cardinalité**

```javascript
// ❌ MAUVAIS
sh.shardCollection("users.accounts", { "account_type": 1 })
// account_type peut avoir seulement 3 valeurs : "free", "premium", "enterprise"
// Maximum 3 chunks possibles → impossible de distribuer sur plus de 3 shards
```

**Impact :**
- Impossibilité de distribuer sur N shards si N > cardinalité
- Chunks indivisibles (jumbo chunks)
- Déséquilibre permanent

**Solution :**
```javascript
// ✅ BON : Combiner avec un champ à haute cardinalité
sh.shardCollection("users.accounts", { "account_type": 1, "user_id": 1 })
```

#### 🚫 **Anti-Pattern 2 : Shard Key monotone sans hachage**

```javascript
// ❌ MAUVAIS
sh.shardCollection("events.logs", { "timestamp": 1 })
// Tous les inserts vont au chunk le plus récent → 1 seul shard actif
```

**Impact :**
- Hotspot permanent sur le shard contenant les valeurs les plus récentes
- Les autres shards sont sous-utilisés
- Performance d'écriture limitée à la capacité d'un seul shard

**Solution :**
```javascript
// ✅ BON : Hashed ou composé
sh.shardCollection("events.logs", { "timestamp": "hashed" })
// ou
sh.shardCollection("events.logs", { "source_id": 1, "timestamp": 1 })
```

#### 🚫 **Anti-Pattern 3 : Shard Key mutable**

```javascript
// ❌ MAUVAIS
sh.shardCollection("products.catalog", { "category": 1, "subcategory": 1 })
// Si category ou subcategory change, le document doit migrer de chunk
```

**Impact :**
- Depuis MongoDB 5.0+, possible de changer la shard key mais avec overhead
- Complexité opérationnelle accrue
- Risque de fragmentation

**Solution :**
```javascript
// ✅ BON : Utiliser un champ immuable
sh.shardCollection("products.catalog", { "_id": 1 })
// ou stocker category_hash calculé à l'insertion
```

#### 🚫 **Anti-Pattern 4 : Shard Key non présente dans les requêtes**

```javascript
// ❌ MAUVAIS
sh.shardCollection("orders.transactions", { "order_date": 1 })

// Mais 90% des requêtes sont :
db.transactions.find({ "customer_id": "123456" })
// → Scatter-gather systématique sur tous les shards
```

**Impact :**
- Toutes les requêtes interrogent tous les shards
- Latence multipliée par le nombre de shards
- Charge réseau excessive

**Solution :**
```javascript
// ✅ BON : Aligner la shard key sur le pattern d'accès
sh.shardCollection("orders.transactions", { "customer_id": 1, "order_date": 1 })

// Les requêtes par customer_id ciblent maintenant un seul shard
```

#### 🚫 **Anti-Pattern 5 : Sur-sharding prématuré**

```javascript
// ❌ MAUVAIS : Déployer 20 shards pour une DB de 100 Go
// Overhead de coordination énorme pour peu de bénéfices
```

**Impact :**
- Complexité opérationnelle sans gain de performance
- Coûts d'infrastructure excessifs
- Overhead de balancing et de routage

**Règle empirique :**
- Commencer avec 2-3 shards
- Ajouter des shards lorsque chaque shard atteint 2-3 To ou 100k ops/sec

### Critères de Choix d'une Shard Key Idéale

Une shard key optimale doit satisfaire ces trois propriétés (paradigme **"CWT"**) :

#### 1. **Cardinalité (Cardinality)**
- ✅ Haute cardinalité : de nombreuses valeurs distinctes possibles
- ✅ Permet de créer de nombreux chunks
- ✅ Exemple : `user_id`, `email`, `uuid`

#### 2. **Distribution (Write distribution)**
- ✅ Les écritures sont uniformément réparties
- ✅ Évite les hotspots
- ✅ Exemple : `hashed(_id)`, `user_id` dans un système avec de nombreux utilisateurs actifs

#### 3. **Ciblabilité (Targetability)**
- ✅ Les requêtes fréquentes incluent la shard key
- ✅ Permet des requêtes ciblées (1 seul shard)
- ✅ Évite les scatter-gather queries

**Exemple de shard key équilibrée :**
```javascript
// Collection : e-commerce orders
sh.shardCollection("shop.orders", { "customer_id": 1, "order_date": 1 })

// ✅ Cardinalité : customer_id offre une haute cardinalité
// ✅ Distribution : Si les clients sont distribués uniformément
// ✅ Ciblabilité : Les requêtes typiques sont :
//    - "Commandes du client X" → ciblée (1 shard)
//    - "Commandes du client X en 2024" → ciblée (1 shard)
```

### Cas d'Usage par Stratégie

| Stratégie | Cas d'Usage Idéal | Exemple Concret |
|-----------|-------------------|-----------------|
| **Range** | Time-series, IoT, logs avec accès chronologique | Métriques système, événements temporels |
| **Hashed** | Collections avec insertions uniformes nécessaires | User profiles, inventory items |
| **Zone** | Multi-tenant, conformité géographique | SaaS multi-régions, données RGPD |

### Métriques de Santé d'un Sharding

Surveillez ces indicateurs pour évaluer l'efficacité de votre sharding :

1. **Distribution des chunks :**
   ```javascript
   sh.status()
   // Vérifier que la distribution est équilibrée (±10%)
   ```

2. **Taux de scatter-gather :**
   ```javascript
   db.collection.explain("executionStats").find({ ... })
   // nShards touched devrait être minimal (idéalement 1)
   ```

3. **Taux de migrations :**
   - < 5 migrations/heure en production stable
   - Pics tolérés lors d'ajout de shards

4. **Jumbo chunks :**
   ```javascript
   db.chunks.find({ jumbo: true }).count()
   // Devrait être 0
   ```

### Structure du Chapitre

Ce chapitre est organisé en 12 sections pour une compréhension progressive et complète du sharding :

- **10.1** : Concepts fondamentaux du sharding
- **10.2** : Architecture détaillée (shards, config servers, mongos)
- **10.3** : Shard Key – Choix et stratégies (approfondissement)
- **10.4** : Types de sharding (Range, Hashed, Zone)
- **10.5** : Chunks et mécanisme de balancing
- **10.6** : Déploiement pratique d'un cluster shardé
- **10.7** : Activation du sharding sur bases et collections
- **10.8** : Migration des chunks et opérations de maintenance
- **10.9** : Opérations spécifiques aux clusters shardés
- **10.10** : Monitoring et maintenance continue
- **10.11** : Résolution des jumbo chunks
- **10.12** : Bonnes pratiques et récapitulatif

### Prérequis

Avant d'aborder ce chapitre, vous devez maîtriser :

- ✅ **Replica Sets** (Chapitre 9) : Architecture, élection, oplog
- ✅ **Modélisation des données** (Chapitre 4) : Patterns, références vs embedded
- ✅ **Index et optimisation** (Chapitre 5) : Stratégies d'indexation
- ✅ **Monitoring** (Chapitre 13) : Métriques, alerting

### Avertissement

> ⚠️ **Le sharding est irréversible** : Une fois une collection shardée, il est extrêmement difficile et coûteux de revenir en arrière. Prenez le temps de concevoir soigneusement votre shard key et testez en profondeur avant la mise en production.

> 🔐 **Sécurité** : Un cluster shardé expose une surface d'attaque plus large (mongos, config servers, multiples shards). Assurez-vous d'appliquer toutes les bonnes pratiques de sécurité (Chapitre 11).

---

**Dans les sections suivantes**, nous allons explorer en détail chaque composant de l'architecture shardée, apprendre à déployer et configurer un cluster en production, et maîtriser les techniques avancées de gestion et d'optimisation.

---


⏭️ [Concepts du sharding](/10-sharding/01-concepts-sharding.md)
