🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.8 Scaling (Vertical et Horizontal)

## Introduction

Le scaling est l'art de **faire évoluer votre infrastructure** pour répondre à la demande croissante sans sacrifier les performances ni exploser les coûts. Atlas offre plusieurs stratégies de scaling : vertical (augmenter les ressources par nœud), horizontal (ajouter des nœuds via sharding), et auto-scaling (ajustement automatique). Cette section guide les équipes DevOps dans la planification de capacité, l'implémentation du scaling, et l'optimisation des coûts.

### 🎯 Objectifs de cette Section

- Comprendre les différents types de scaling
- Maîtriser le scaling vertical (tier changes)
- Implémenter le sharding pour le scaling horizontal
- Configurer l'auto-scaling intelligent
- Planifier la capacité à long terme
- Optimiser le ratio performance/coût

---

## 📊 Types de Scaling

### Vue d'Ensemble

```
┌───────────────────────────────────────────────────────────────────────┐
│                        SCALING STRATEGIES                             │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. VERTICAL SCALING (Scale Up/Down)                                  │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  M30 (8GB RAM)      →    M40 (16GB RAM)     →    M50 (32GB RAM)  │ │
│  │  ┌─────────┐            ┌─────────┐            ┌─────────┐       │ │
│  │  │  Node   │            │  Node   │            │  Node   │       │ │
│  │  │ 2 vCPU  │            │ 4 vCPU  │            │ 8 vCPU  │       │ │
│  │  │  8GB    │            │  16GB   │            │  32GB   │       │ │
│  │  └─────────┘            └─────────┘            └─────────┘       │ │
│  │                                                                  │ │
│  │  Pros: Simple, no code changes                                   │ │
│  │  Cons: Limits (max M700), brief downtime                         │ │
│  │  Cost: Linear increase                                           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  2. HORIZONTAL SCALING - READ REPLICAS (Scale Out)                    │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  3-node RS          →        5-node RS         →     7-node RS   │ │
│  │  ┌─────────┐              ┌─────────┐              ┌─────────┐   │ │
│  │  │ Primary │              │ Primary │              │ Primary │   │ │
│  │  └────┬────┘              └────┬────┘              └────┬────┘   │ │
│  │    ┌──┴──┐                  ┌──┴──┐                  ┌──┴──┐     │ │
│  │  ┌─▼─┐ ┌─▼─┐              ┌─▼─┐ ┌─▼─┐              ┌─▼─┐ ┌─▼─┐   │ │
│  │  │S1 │ │S2 │              │S1 │ │S2 │              │S1 │ │S2 │   │ │
│  │  └───┘ └───┘              └─┬─┘ └─┬─┘              └─┬─┘ └─┬─┘   │ │
│  │                           ┌─▼─┐ ┌─▼─┐              ┌─▼─┐ ┌─▼─┐   │ │
│  │                           │S3 │ │S4 │              │S3 │ │S4 │   │ │
│  │                           └───┘ └───┘              └─┬─┘ └─┬─┘   │ │
│  │                                                    ┌─▼─┐ ┌─▼─┐   │ │
│  │                                                    │S5 │ │S6 │   │ │
│  │                                                    └───┘ └───┘   │ │
│  │                                                                  │ │
│  │  Pros: Increase read capacity, better availability               │ │
│  │  Cons: No write scaling, storage still limited                   │ │
│  │  Cost: Linear per node                                           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  3. HORIZONTAL SCALING - SHARDING (Scale Out)                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  1 Shard              →         2 Shards        →    4 Shards    │ │
│  │  ┌─────────┐                ┌─────────┬─────────┐                │ │
│  │  │ Shard 0 │                │ Shard 0 │ Shard 1 │                │ │
│  │  │ 100GB   │                │  50GB   │  50GB   │                │ │
│  │  └─────────┘                └─────────┴─────────┘                │ │
│  │                              ┌──────┬──────┬──────┬──────┐       │ │
│  │                              │Shard0│Shard1│Shard2│Shard3│       │ │
│  │                              │ 25GB │ 25GB │ 25GB │ 25GB │       │ │
│  │                              └──────┴──────┴──────┴──────┘       │ │
│  │                                                                  │ │
│  │  Pros: Unlimited scaling, write scaling, data distribution       │ │
│  │  Cons: Complex, shard key crucial, rebalancing overhead          │ │
│  │  Cost: Linear per shard                                          │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  4. AUTO-SCALING (Automated)                                          │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  Normal Load    →    Peak Load    →    Normal Load               │ │
│  │  M40 (16GB)         M50 (32GB)        M40 (16GB)                 │ │
│  │                                                                  │ │
│  │  Automatic adjustment based on:                                  │ │
│  │  • CPU utilization                                               │ │
│  │  • Memory pressure                                               │ │
│  │  • Disk usage                                                    │ │
│  │                                                                  │ │
│  │  Pros: Responsive, cost-optimized, hands-off                     │ │
│  │  Cons: Potential brief downtime, cost variability                │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Matrice de Décision

```
┌────────────────────────────────────────────────────────────────────────┐
│                    SCALING DECISION MATRIX                             │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  SCENARIO                           SOLUTION                           │
│  ───────────────────────────────────────────────────────────────────── │
│  CPU constantly > 80%               Vertical scale up                  │
│  Memory constantly > 85%            Vertical scale up                  │
│  Disk > 80% full                    Auto-scale storage OR vertical up  │
│                                                                        │
│  Read-heavy workload                Add read replicas                  │
│  Analytics slowing production       Add analytics node                 │
│                                                                        │
│  Data > 2TB                         Consider sharding                  │
│  Write throughput maxed out         Implement sharding                 │
│  Single collection > 500GB          Shard that collection              │
│                                                                        │
│  Unpredictable traffic spikes       Enable auto-scaling                │
│  Daily/weekly traffic patterns      Enable auto-scaling                │
│                                                                        │
│  Budget constraints                 Right-size + auto-scale down       │
│  Steady predictable load            Reserved capacity                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## ⬆️ Scaling Vertical

### Process de Scaling Vertical

```
┌───────────────────────────────────────────────────────────────────────┐
│                    VERTICAL SCALING WORKFLOW                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  PHASE 1: ASSESSMENT (5 minutes)                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ • Current metrics: CPU, RAM, disk utilization                    │ │
│  │ • Workload trends over past 30 days                              │ │
│  │ • Peak usage patterns                                            │ │
│  │ • Growth projection                                              │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                              ▼                                        │
│  PHASE 2: TIER SELECTION (2 minutes)                                  │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Current: M40 (16GB RAM, 4 vCPU)                                  │ │
│  │ Target:  M50 (32GB RAM, 8 vCPU)                                  │ │
│  │                                                                  │ │
│  │ Validation:                                                      │ │
│  │ ✅ Target RAM > Working Set Size × 1.5                           │ │
│  │ ✅ Target CPU handles peak + 30% headroom                        │ │
│  │ ✅ Target storage > current usage × 1.3                          │ │
│  │ ✅ Cost increase acceptable ($630 → $1,525/month)                │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                              ▼                                        │
│  PHASE 3: PLANNING (10 minutes)                                       │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ • Schedule maintenance window (low traffic period)               │ │
│  │ • Notify stakeholders                                            │ │
│  │ • Prepare rollback plan                                          │ │
│  │ • Update capacity planning docs                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                              ▼                                        │
│  PHASE 4: EXECUTION (10-20 minutes)                                   │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ For each node in replica set:                                    │ │
│  │   1. Secondary 1: Scale, wait for healthy                        │ │
│  │   2. Secondary 2: Scale, wait for healthy                        │ │
│  │   3. Primary: Trigger election, then scale                       │ │
│  │                                                                  │ │
│  │ Brief downtime: ~30-60 seconds during primary election           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                              ▼                                        │
│  PHASE 5: VALIDATION (5 minutes)                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ ✅ All nodes healthy                                             │ │
│  │ ✅ Replication lag < 1s                                          │ │
│  │ ✅ Application connections restored                              │ │
│  │ ✅ No errors in logs                                             │ │
│  │ ✅ Metrics showing improved capacity                             │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  TOTAL TIME: ~30-40 minutes                                           │
│  DOWNTIME:   ~30-60 seconds                                           │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Configuration via Terraform

```hcl
# Terraform: Vertical Scaling
resource "mongodbatlas_cluster" "production" {
  project_id = var.atlas_project_id
  name       = "production-cluster"

  # Current tier
  provider_instance_size_name = "M40"

  # To scale up to M50, simply update:
  # provider_instance_size_name = "M50"
  # terraform apply

  provider_name = "AWS"
  provider_region_name = "US_EAST_1"

  # Replica set configuration
  replication_specs {
    num_shards = 1
    regions_config {
      region_name     = "US_EAST_1"
      electable_nodes = 3
      priority        = 7
      read_only_nodes = 0
    }
  }
}

# Alternative: Use auto-scaling (recommended)
resource "mongodbatlas_cluster" "production_autoscale" {
  project_id = var.atlas_project_id
  name       = "production-cluster"

  provider_instance_size_name = "M40"

  # Auto-scaling configuration
  auto_scaling_compute_enabled = true
  auto_scaling_compute_scale_down_enabled = true

  provider_auto_scaling_compute_min_instance_size = "M40"
  provider_auto_scaling_compute_max_instance_size = "M60"
}
```

### Limites du Scaling Vertical

```
┌────────────────────────────────────────────────────────────────────────┐
│                 VERTICAL SCALING LIMITATIONS                           │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  LIMIT              CONSTRAINT                                         │
│  ───────────────────────────────────────────────────────────────────── │
│  Maximum Tier       M700+ (custom tiers available)                     │
│                     Typical max: M400 (512GB RAM)                      │
│                                                                        │
│  Cost               Exponential growth at high tiers                   │
│                     M200: $14,400/month                                │
│                     M400: ~$30,000/month                               │
│                                                                        │
│  Downtime           30-60 seconds per scaling operation                │
│                     (primary election)                                 │
│                                                                        │
│  Single Point       All data on one logical cluster                    │
│                     Cannot distribute writes                           │
│                                                                        │
│  Hardware           Physical limits of single machines                 │
│                     Max ~1TB RAM per node in practice                  │
│                                                                        │
│  WHEN TO STOP VERTICAL SCALING:                                        │
│  • Tier > M200 (consider sharding)                                     │
│  • Single collection > 500GB (must shard)                              │
│  • Write throughput maxed (need write distribution)                    │
│  • Cost > $10K/month (evaluate sharding economics)                     │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## ➡️ Scaling Horizontal : Sharding

### Architecture Sharded Cluster

```
┌───────────────────────────────────────────────────────────────────────┐
│                   SHARDED CLUSTER ARCHITECTURE                        │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│                           APPLICATION                                 │
│                                 │                                     │
│                                 ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                      MONGOS (Query Router)                      │  │
│  │  • Routes queries to appropriate shards                         │  │
│  │  • Merges results from multiple shards                          │  │
│  │  • Atlas-managed (automatic load balancing)                     │  │
│  └──────────────┬─────────────────┬─────────────────┬──────────────┘  │
│                 │                 │                 │                 │
│     ┌───────────┴────┐   ┌────────┴────────┐   ┌────┴────────────┐    │
│     ▼                ▼   ▼                 ▼   ▼                 ▼    │
│  ┌────────┐      ┌────────┐          ┌────────┐           ┌────────┐  │
│  │ SHARD 0│      │ SHARD 1│          │ SHARD 2│           │ SHARD 3│  │
│  ├────────┤      ├────────┤          ├────────┤           ├────────┤  │
│  │Primary │      │Primary │          │Primary │           │Primary │  │
│  │   ┌────┴┐     │   ┌────┴┐         │   ┌────┴┐          │   ┌────┴┐ │
│  │   │Sec 1│     │   │Sec 1│         │   │Sec 1│          │   │Sec 1│ │
│  │   │Sec 2│     │   │Sec 2│         │   │Sec 2│          │   │Sec 2│ │
│  └───┴─────┘     └───┴─────┘         └───┴─────┘          └───┴─────┘ │
│                                                                       │
│  Data Range:     Data Range:        Data Range:         Data Range:   │
│  userId:         userId:            userId:             userId:       │
│  0-250K          250K-500K          500K-750K           750K-1M       │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │              CONFIG SERVERS (3-node replica set)                │  │
│  │  • Store cluster metadata                                       │  │
│  │  • Shard key ranges                                             │  │
│  │  • Chunk distribution                                           │  │
│  │  • Atlas-managed                                                │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Choix du Shard Key

Le **shard key** est la décision la plus critique du sharding. Il détermine comment les données sont distribuées.

```
┌────────────────────────────────────────────────────────────────────────┐
│                      SHARD KEY SELECTION GUIDE                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  CRITERIA FOR GOOD SHARD KEY:                                          │
│  ─────────────────────────────                                         │
│                                                                        │
│  1. HIGH CARDINALITY                                                   │
│     • Many distinct values                                             │
│     • Example: userId (millions of users)                              │
│     ✅ Good: userId (1M+ unique values)                                │
│     ❌ Bad: country (200 values) → Hot spots                           │
│                                                                        │
│  2. EVEN DISTRIBUTION                                                  │
│     • Values spread evenly across range                                │
│     • No skew towards certain values                                   │
│     ✅ Good: UUID, hashed field                                        │
│     ❌ Bad: timestamp (monotonic) → All writes to one shard            │
│                                                                        │
│  3. QUERY TARGETING                                                    │
│     • Most queries include shard key                                   │
│     • Avoid scatter-gather queries                                     │
│     ✅ Good: Query by userId (targets 1 shard)                         │
│     ❌ Bad: Query without shard key (hits all shards)                  │
│                                                                        │
│  4. WRITE DISTRIBUTION                                                 │
│     • Writes spread across all shards                                  │
│     • No single "hot" shard                                            │
│     ✅ Good: Hashed _id                                                │
│     ❌ Bad: Incrementing orderId                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Exemples de Shard Keys

```javascript
// EXAMPLE 1: E-commerce Order System
// =====================================

// ❌ BAD: Monotonic orderId
{
  shardKey: { orderId: 1 }
  // Problem: All new orders go to last shard
  // Result: Hot shard, poor write distribution
}

// ⚠️ MEDIUM: Country
{
  shardKey: { country: 1 }
  // Problem: Low cardinality (200 values)
  // Some shards have much more data (USA, China)
  // Result: Uneven distribution
}

// ✅ GOOD: Hashed userId
{
  shardKey: { userId: "hashed" }
  // Pros: Even distribution, good write scaling
  // Cons: Range queries on userId hit all shards
  // Use case: When most queries are equality (userId == X)
}

// ✅ BEST: Compound key (userId + orderId)
{
  shardKey: { userId: 1, orderId: 1 }
  // Pros:
  // - Targets single shard per user
  // - Range queries within user efficient
  // - Good write distribution (many users)
  // Use case: Query orders by user
}

// EXAMPLE 2: IoT Time-Series Data
// =================================

// ❌ BAD: timestamp only
{
  shardKey: { timestamp: 1 }
  // All writes go to last chunk (latest time)
}

// ✅ GOOD: deviceId + timestamp
{
  shardKey: { deviceId: 1, timestamp: 1 }
  // Distributes writes across devices
  // Time-range queries per device efficient
}

// ✅ ALTERNATIVE: Hashed deviceId
{
  shardKey: { deviceId: "hashed" }
  // Random distribution, maximum write throughput
  // Trade-off: Range queries less efficient
}

// EXAMPLE 3: Social Media Platform
// ==================================

// ❌ BAD: createdAt
{
  shardKey: { createdAt: 1 }
  // All new posts to one shard
}

// ✅ GOOD: authorId + createdAt
{
  shardKey: { authorId: 1, createdAt: -1 }
  // Posts distributed by author
  // Query "author's recent posts" hits one shard
}

// EXAMPLE 4: Multi-tenant SaaS
// ==============================

// ✅ EXCELLENT: tenantId + documentId
{
  shardKey: { tenantId: 1, documentId: 1 }
  // Complete tenant isolation per shard (if possible)
  // All queries for tenant hit single shard
  // Crucial for performance and data isolation
}
```

### Implémentation du Sharding

```javascript
// Step 1: Enable sharding on database
sh.enableSharding("mydb")

// Step 2: Create index on shard key
db.orders.createIndex({ userId: 1, orderId: 1 })

// Step 3: Shard the collection
sh.shardCollection("mydb.orders", { userId: 1, orderId: 1 })

// Step 4: Pre-split chunks (optional, for large collections)
// Prevents initial balancing overhead
for (let i = 0; i < 1000000; i += 100000) {
  sh.splitAt("mydb.orders", { userId: i, orderId: MinKey })
}

// Step 5: Monitor shard distribution
db.orders.getShardDistribution()
// Output:
// Shard shard0000 at shard0000/...
//  data : 25GiB docs : 5000000 chunks : 250
// Shard shard0001 at shard0001/...
//  data : 25GiB docs : 5000000 chunks : 250
// Shard shard0002 at shard0002/...
//  data : 25GiB docs : 5000000 chunks : 250
// Shard shard0003 at shard0003/...
//  data : 25GiB docs : 5000000 chunks : 250
//
// Totals
//  data : 100GiB docs : 20000000 chunks : 1000
```

### Configuration Atlas Sharding

```hcl
# Terraform: Sharded Cluster Configuration
resource "mongodbatlas_cluster" "production_sharded" {
  project_id = var.atlas_project_id
  name       = "production-sharded"

  cluster_type = "SHARDED"

  # Each shard is an M40 replica set
  provider_instance_size_name = "M40"
  provider_name               = "AWS"

  # Sharding configuration
  replication_specs {
    num_shards = 4  # Start with 4 shards

    regions_config {
      region_name     = "US_EAST_1"
      electable_nodes = 3  # 3 nodes per shard (HA)
      priority        = 7
      read_only_nodes = 0
    }
  }

  # Auto-scaling per shard
  auto_scaling_compute_enabled            = true
  auto_scaling_compute_scale_down_enabled = true
  provider_auto_scaling_compute_min_instance_size = "M40"
  provider_auto_scaling_compute_max_instance_size = "M60"
}

# Output monthly cost calculation
output "sharded_cluster_cost" {
  value = "Estimated: 4 shards × $630 (M40) = $2,520/month base"
}
```

### Ajout de Shards

```
┌────────────────────────────────────────────────────────────────────────┐
│                    ADDING SHARDS WORKFLOW                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  INITIAL STATE: 2 Shards                                               │
│  ┌──────────┬──────────┐                                               │
│  │ Shard 0  │ Shard 1  │                                               │
│  │ 100GB    │ 100GB    │                                               │
│  └──────────┴──────────┘                                               │
│                                                                        │
│  STEP 1: Add 2 New Shards (via Atlas UI or Terraform)                  │
│  ┌──────────┬──────────┬──────────┬──────────┐                         │
│  │ Shard 0  │ Shard 1  │ Shard 2  │ Shard 3  │                         │
│  │ 100GB    │ 100GB    │  Empty   │  Empty   │                         │
│  └──────────┴──────────┴──────────┴──────────┘                         │
│                                                                        │
│  STEP 2: Balancer Redistributes Chunks (Automatic)                     │
│  Time: 2-24 hours depending on data size                               │
│  ┌──────────┬──────────┬──────────┬──────────┐                         │
│  │ Shard 0  │ Shard 1  │ Shard 2  │ Shard 3  │                         │
│  │ 75GB     │ 75GB     │ 25GB     │ 25GB     │  (Progressive)          │
│  └──────────┴──────────┴──────────┴──────────┘                         │
│                 ▼                                                      │
│  ┌──────────┬──────────┬──────────┬──────────┐                         │
│  │ Shard 0  │ Shard 1  │ Shard 2  │ Shard 3  │                         │
│  │ 50GB     │ 50GB     │ 50GB     │ 50GB     │  (Balanced)             │
│  └──────────┴──────────┴──────────┴──────────┘                         │
│                                                                        │
│  IMPACT DURING REBALANCING:                                            │
│  • Increased network traffic                                           │
│  • Slight performance impact (queries still work)                      │
│  • Automatic throttling to minimize disruption                         │
│                                                                        │
│  COST IMPACT:                                                          │
│  Before: 2 shards × $630 = $1,260/month                                │
│  After:  4 shards × $630 = $2,520/month                                │
│  Increase: 100% cost for 100% capacity                                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🤖 Auto-Scaling

### Configuration Auto-Scaling

```
┌───────────────────────────────────────────────────────────────────────┐
│                   AUTO-SCALING CONFIGURATION                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  COMPUTE AUTO-SCALING                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  Triggers:                                                       │ │
│  │  • CPU > 75% sustained for 1 hour      → Scale up                │ │
│  │  • Memory > 75% sustained for 1 hour   → Scale up                │ │
│  │  • CPU < 50% sustained for 24 hours    → Scale down              │ │
│  │                                                                  │ │
│  │  Configuration:                                                  │ │
│  │  • Min instance size: M40 (16GB)                                 │ │
│  │  • Max instance size: M80 (128GB)                                │ │
│  │  • Scale down enabled: Yes                                       │ │
│  │                                                                  │ │
│  │  Behavior:                                                       │ │
│  │  • Scales up: Within 1-2 hours of threshold                      │ │
│  │  • Scales down: After 24h below threshold (conservative)         │ │
│  │  • Brief downtime: ~30-60s per scale event                       │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  STORAGE AUTO-SCALING                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  Triggers:                                                       │ │
│  │  • Disk usage > 90%                    → Add storage             │ │
│  │                                                                  │ │
│  │  Behavior:                                                       │ │
│  │  • Adds 10-20% storage increment                                 │ │
│  │  • No downtime                                                   │ │
│  │  • Max limit: 4TB per node (configurable)                        │ │
│  │  • Cooldown: 6 hours between expansions                          │ │
│  │                                                                  │ │
│  │  Cost:                                                           │ │
│  │  • $0.25/GB-month (standard)                                     │ │
│  │  • Example: +100GB = +$25/month                                  │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Terraform Configuration

```hcl
# Terraform: Auto-Scaling Configuration
resource "mongodbatlas_cluster" "production" {
  project_id = var.atlas_project_id
  name       = "production-autoscale"

  provider_instance_size_name = "M40"

  # Compute Auto-Scaling
  auto_scaling_compute_enabled            = true
  auto_scaling_compute_scale_down_enabled = true

  provider_auto_scaling_compute_min_instance_size = "M40"
  provider_auto_scaling_compute_max_instance_size = "M80"

  # Storage Auto-Scaling
  auto_scaling_disk_gb_enabled = true

  # Disk size will auto-expand when > 90% full
  # Starts at disk_size_gb, can grow to max
  disk_size_gb = 100  # Starting size

  # Additional settings
  provider_name         = "AWS"
  provider_region_name  = "US_EAST_1"

  replication_specs {
    num_shards = 1
    regions_config {
      region_name     = "US_EAST_1"
      electable_nodes = 3
      priority        = 7
    }
  }
}
```

### Auto-Scaling en Action

```
┌───────────────────────────────────────────────────────────────────────┐
│              AUTO-SCALING SCENARIO: BLACK FRIDAY                      │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Timeline: 24-hour period                                             │
│                                                                       │
│  00:00 - Normal Load                                                  │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Cluster: M40 (16GB RAM)                                          │ │
│  │ CPU: 45%    Memory: 60%    Cost: $630/month ($0.88/hour)         │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  08:00 - Traffic Increases                                            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ CPU: 68%    Memory: 72%                                          │ │
│  │ Status: Monitoring, approaching threshold                        │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  09:00 - Threshold Exceeded                                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ CPU: 78%    Memory: 76%    (sustained for 1 hour)                │ │
│  │ 🔔 AUTO-SCALE TRIGGERED: M40 → M50                               │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  10:00 - Scaled Up                                                    │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ Cluster: M50 (32GB RAM)                                          │ │
│  │ CPU: 45%    Memory: 48%    Cost: $1,525/month ($2.12/hour)       │ │
│  │ Performance: Smooth, handling increased load                     │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  12:00 - Peak Black Friday Traffic                                    │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ CPU: 72%    Memory: 68%                                          │ │
│  │ Status: Within acceptable range on M50                           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  18:00 - Traffic Subsiding                                            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ CPU: 38%    Memory: 45%                                          │ │
│  │ Status: Below threshold, monitoring for 24h before scale down    │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  Next Day 19:00 - Scale Down                                          │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ CPU < 50% for 24+ hours                                          │ │
│  │ 🔔 AUTO-SCALE TRIGGERED: M50 → M40                               │ │
│  │ Back to baseline configuration                                   │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  COST ANALYSIS:                                                       │
│  • M40 baseline: 22 hours × $0.88 = $19.36                            │
│  • M50 peak: 34 hours × $2.12 = $72.08                                │
│  • Total: $91.44 for 56-hour period                                   │
│                                                                       │
│  vs. Always M50: 56 hours × $2.12 = $118.72                           │
│  Savings: $27.28 (23% cost reduction)                                 │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 📊 Capacity Planning

### Sizing Formula

```python
# Capacity Planning Calculator
def calculate_required_tier(
    working_set_size_gb,
    peak_ops_per_sec,
    peak_connections,
    growth_rate_monthly=0.10,
    planning_horizon_months=12
):
    """
    Calculate required Atlas tier based on workload characteristics
    """
    # 1. Project future data size
    future_data_size = working_set_size_gb * (1 + growth_rate_monthly) ** planning_horizon_months

    # 2. RAM requirement (WSS + 50% headroom)
    required_ram = future_data_size * 1.5

    # 3. Operations requirement
    # Rule of thumb: M40 handles ~10K reads/s, 5K writes/s
    ops_factor = peak_ops_per_sec / 10000

    # 4. Connections requirement
    # Each tier supports different max connections
    connection_factor = peak_connections / 1000

    # 5. Select tier
    tiers = {
        "M10": {"ram": 2, "cost": 57},
        "M20": {"ram": 4, "cost": 140},
        "M30": {"ram": 8, "cost": 285},
        "M40": {"ram": 16, "cost": 630},
        "M50": {"ram": 32, "cost": 1525},
        "M60": {"ram": 64, "cost": 3050},
        "M80": {"ram": 128, "cost": 6480},
        "M140": {"ram": 192, "cost": 10800},
        "M200": {"ram": 256, "cost": 14400},
    }

    # Find minimum tier that satisfies requirements
    for tier_name, specs in tiers.items():
        if specs["ram"] >= required_ram:
            return {
                "recommended_tier": tier_name,
                "ram_gb": specs["ram"],
                "monthly_cost": specs["cost"],
                "future_data_size_gb": round(future_data_size, 2),
                "utilization": round(required_ram / specs["ram"] * 100, 1),
            }

    return {
        "recommended_tier": "M200+ or Sharding",
        "reason": "Data size exceeds single cluster capacity"
    }

# Example usage
result = calculate_required_tier(
    working_set_size_gb=50,      # Current: 50GB working set
    peak_ops_per_sec=5000,       # Peak: 5K ops/second
    peak_connections=500,         # Peak: 500 connections
    growth_rate_monthly=0.10,    # Growth: 10% per month
    planning_horizon_months=12   # Planning: 1 year ahead
)

print(f"Recommended Tier: {result['recommended_tier']}")
print(f"RAM: {result['ram_gb']}GB")
print(f"Monthly Cost: ${result['monthly_cost']}")
print(f"Projected Data (1yr): {result['future_data_size_gb']}GB")
print(f"Utilization: {result['utilization']}%")

# Output:
# Recommended Tier: M80
# RAM: 128GB
# Monthly Cost: $6480
# Projected Data (1yr): 157.69GB
# Utilization: 61.8%
```

### Growth Projection

```
┌────────────────────────────────────────────────────────────────────────┐
│                      GROWTH PROJECTION MODEL                           │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Current State: M40 cluster, 50GB data                                 │
│  Growth Rate: 10% monthly                                              │
│                                                                        │
│  PROJECTION (12 MONTHS):                                               │
│                                                                        │
│  Month    Data Size    Tier Needed    Monthly Cost    Action           │
│  ────────────────────────────────────────────────────────────────────  │
│  0 (Now)  50 GB        M40            $630            Current          │
│  3        66 GB        M50            $1,525          Scale in Q1      │
│  6        89 GB        M50            $1,525          OK               │
│  9        118 GB       M60            $3,050          Scale in Q3      │
│  12       157 GB       M80            $6,480          Scale in Q4      │
│                                                                        │
│  ALTERNATIVE STRATEGY: Enable Auto-Scaling                             │
│  • Min: M40                                                            │
│  • Max: M80                                                            │
│  • Let Atlas handle scaling automatically                              │
│  • Average cost: ~$3,500/month (vs. $6,480 always-on M80)              │
│                                                                        │
│  ALTERNATIVE STRATEGY 2: Shard at M60                                  │
│  • At month 9, implement 2-shard cluster                               │
│  • Cost: 2 × M40 = $1,260/month (cheaper than M60!)                    │
│  • Better scalability path                                             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Stratégies de Scaling

### Par Type d'Application

```yaml
# Scaling Strategies by Application Type

# 1. E-COMMERCE
e_commerce:
  characteristics:
    - High read/write ratio: 10:1
    - Spiky traffic (sales, Black Friday)
    - User-centric queries

  strategy:
    initial: "M30-M40"
    scaling_approach: "Vertical + Auto-scaling"
    auto_scale:
      min: "M30"
      max: "M60"

    shard_when: "Data > 500GB OR Tier > M60"
    shard_key: "{ userId: 1, orderId: 1 }"

  expected_cost:
    baseline: "$285-630/month"
    peak: "$3,050/month (auto-scaled to M60)"
    average: "$1,200/month"

# 2. SOCIAL MEDIA / CONTENT PLATFORM
social_media:
  characteristics:
    - Very high read ratio: 100:1
    - Large media metadata
    - User timeline queries

  strategy:
    initial: "M40"
    scaling_approach: "Read replicas + Sharding"
    read_replicas: 5  # Add analytics nodes

    shard_when: "Early (proactive)"
    shard_key: "{ authorId: 1, createdAt: -1 }"

  expected_cost:
    baseline: "$630/month (M40)"
    with_replicas: "$3,150/month (5 nodes)"
    sharded: "$2,520/month (4 shards M40)"

# 3. IOT / TIME-SERIES
iot_timeseries:
  characteristics:
    - Very high write ratio: 1:100
    - Time-based queries
    - Predictable growth

  strategy:
    initial: "M20-M30"
    scaling_approach: "Horizontal sharding (early)"

    shard_immediately: true
    shard_key: "{ deviceId: 1, timestamp: 1 }"
    num_shards: 4  # Start with 4

    data_lifecycle:
      hot_data: "7 days (Atlas cluster)"
      warm_data: "30 days (Atlas Data Lake)"
      cold_data: "1+ year (S3 Glacier)"

  expected_cost:
    baseline: "$1,140/month (4 × M30)"
    with_tiering: "$600/month (reduced hot data)"

# 4. ANALYTICS / BI
analytics:
  characteristics:
    - Read-only (mostly)
    - Complex aggregations
    - Large scans

  strategy:
    initial: "M40"
    scaling_approach: "Dedicated analytics nodes"

    analytics_nodes: 2-3
    read_preference: "secondary"

    consider: "Atlas Data Lake for historical data"

  expected_cost:
    baseline: "$630/month (M40)"
    with_analytics: "$1,890/month (3 nodes M40)"

# 5. MULTI-TENANT SAAS
multi_tenant:
  characteristics:
    - Tenant isolation critical
    - Variable per-tenant usage
    - Compliance requirements

  strategy:
    initial: "M30-M40"
    scaling_approach: "Shard by tenant"

    shard_key: "{ tenantId: 1, documentId: 1 }"

    large_tenants: "Dedicated clusters"
    small_tenants: "Shared sharded cluster"

  expected_cost:
    shared_cluster: "$630-1,260/month"
    per_large_tenant: "$285-630/month"
```

---

## 📋 Best Practices

### Scaling Checklist

```
┌────────────────────────────────────────────────────────────────────────┐
│                     SCALING BEST PRACTICES                             │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  MONITORING & ALERTS                                                   │
│  ☐ Set up alerts for CPU > 70%, Memory > 75%                           │
│  ☐ Monitor trends, not just absolutes                                  │
│  ☐ Track growth rate (weekly/monthly)                                  │
│  ☐ Dashboard with capacity planning metrics                            │
│                                                                        │
│  VERTICAL SCALING                                                      │
│  ☐ Scale proactively, not reactively                                   │
│  ☐ Keep 20-30% headroom for unexpected spikes                          │
│  ☐ Schedule scaling during low-traffic windows                         │
│  ☐ Test application resilience to brief disconnects                    │
│  ☐ Enable auto-scaling for production clusters                         │
│                                                                        │
│  HORIZONTAL SCALING (SHARDING)                                         │
│  ☐ Choose shard key carefully (cannot change easily)                   │
│  ☐ Test shard key with production query patterns                       │
│  ☐ Validate even data distribution before full migration               │
│  ☐ Start with 2-4 shards (not 1, not 10)                               │
│  ☐ Plan for future shard additions                                     │
│  ☐ Monitor balancer impact during rebalancing                          │
│                                                                        │
│  CAPACITY PLANNING                                                     │
│  ☐ Review capacity quarterly                                           │
│  ☐ Project growth 6-12 months ahead                                    │
│  ☐ Budget for scaling costs                                            │
│  ☐ Consider sharding at Tier > M60                                     │
│  ☐ Evaluate multi-region for DR + capacity                             │
│                                                                        │
│  COST OPTIMIZATION                                                     │
│  ☐ Right-size clusters (don't over-provision)                          │
│  ☐ Use auto-scaling to handle variability                              │
│  ☐ Consider reserved capacity for stable workloads                     │
│  ☐ Archive old data to Data Lake                                       │
│  ☐ Review and remove unused indexes                                    │
│                                                                        │
│  TESTING                                                               │
│  ☐ Load test before scaling events                                     │
│  ☐ Validate application handles scaling gracefully                     │
│  ☐ Test failover scenarios                                             │
│  ☐ Document scaling procedures                                         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Anti-Patterns

```
❌ SCALING ANTI-PATTERNS:

1. Reactive Scaling
   • Waiting until CPU hits 100%
   • Scaling during peak traffic
   • No capacity planning
   → Solution: Proactive monitoring, scale at 70-80%

2. Over-Provisioning
   • Running M200 with 20% utilization
   • "Just in case" mentality
   → Solution: Use auto-scaling, right-size

3. Poor Shard Key Choice
   • Using monotonic fields (timestamp, auto-increment)
   • Low cardinality fields (country, status)
   → Solution: Research, test, validate distribution

4. Too Many Shards
   • Starting with 10+ shards for 100GB data
   • Each shard < 50GB
   → Solution: Start small (2-4 shards), grow as needed

5. Ignoring Query Patterns
   • Sharding without analyzing queries
   • Scatter-gather on every query
   → Solution: Analyze slow query log, optimize for common patterns

6. No Testing
   • Scaling in production without testing
   • Assuming application handles disconnects
   → Solution: Test in staging first
```

---

## 🏁 Résumé

### Points Clés

1. **Types de Scaling**
   - Vertical : Augmenter les ressources par nœud (simple, limité)
   - Horizontal : Ajouter des nœuds/shards (complexe, illimité)
   - Auto-scaling : Ajustement automatique (recommandé)

2. **Vertical Scaling**
   - Simple : Update tier dans Terraform
   - Downtime : ~30-60s (primary election)
   - Limite : M700 pratique max
   - Coût : Exponentiel aux tiers élevés

3. **Sharding**
   - Shard key = décision critique
   - Cardinality, distribution, query targeting
   - Start small : 2-4 shards
   - Ajouter shards = 2x capacity, 2x cost

4. **Auto-Scaling**
   - Compute : M40-M80 range recommandé
   - Storage : Auto-expand à 90%
   - Scale up : 1h après threshold
   - Scale down : 24h après threshold

5. **Capacity Planning**
   - Projeter 6-12 mois
   - WSS × 1.5 = RAM requis
   - Considérer sharding à M60+
   - Budget pour croissance

### Configuration Recommandée Production

```hcl
resource "mongodbatlas_cluster" "production" {
  project_id = var.project_id
  name       = "production"

  # Start with appropriate tier
  provider_instance_size_name = "M40"

  # Enable auto-scaling
  auto_scaling_compute_enabled            = true
  auto_scaling_compute_scale_down_enabled = true
  provider_auto_scaling_compute_min_instance_size = "M40"
  provider_auto_scaling_compute_max_instance_size = "M80"

  # Storage auto-scaling
  auto_scaling_disk_gb_enabled = true
  disk_size_gb                 = 100

  # 3-node replica set (HA)
  replication_specs {
    num_shards = 1
    regions_config {
      region_name     = "US_EAST_1"
      electable_nodes = 3
      priority        = 7
    }
  }
}
```

### Decision Tree

```
Start
  │
  ├─ Data < 500GB AND Tier < M60?
  │  └─ YES → Use vertical scaling + auto-scaling
  │  └─ NO  → ↓
  │
  ├─ Write-heavy OR Data > 2TB?
  │  └─ YES → Implement sharding
  │  └─ NO  → ↓
  │
  ├─ Read-heavy analytics?
  │  └─ YES → Add analytics nodes OR Data Lake
  │  └─ NO  → ↓
  │
  └─ Variable traffic patterns?
     └─ YES → Enable auto-scaling
     └─ NO  → Reserved capacity
```

---


⏭️ [Atlas Search](/14-mongodb-atlas/09-atlas-search.md)
