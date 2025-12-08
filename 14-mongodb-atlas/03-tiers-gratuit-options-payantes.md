🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.3 Tiers Gratuit (M0) et Options Payantes

## Introduction

Le modèle de pricing d'Atlas est conçu pour s'adapter à tous les stades d'une organisation : du développeur individuel apprenant MongoDB avec le **tier gratuit M0**, aux **entreprises gérant des pétaoctets** de données avec des clusters M700+ personnalisés. Cette section analyse en profondeur les options tarifaires, le calcul du TCO (Total Cost of Ownership), et les stratégies d'optimisation des coûts pour les équipes DevOps et FinOps.

### 🎯 Objectifs de cette Section

- Comprendre la structure de pricing complète d'Atlas
- Calculer le TCO pour différents scénarios
- Identifier les coûts cachés et les optimisations
- Développer une stratégie de sizing économique
- Maîtriser les options de réduction de coûts (Reserved Capacity, Pausing, etc.)

---

## 💚 Tier Gratuit : M0 (Free Forever)

### Spécifications Techniques

```
┌────────────────────────────────────────────────────────────────────┐
│                       M0 FREE TIER CLUSTER                         │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  COMPUTE                                                           │
│  ├─ RAM:              512 MB                                       │
│  ├─ vCPU:             Shared (contended)                           │
│  └─ Architecture:     Single replica set (3 nodes)                 │
│                                                                    │
│  STORAGE                                                           │
│  ├─ Capacity:         512 MB                                       │
│  ├─ Type:             Standard (shared disk)                       │
│  └─ IOPS:             Limited (shared)                             │
│                                                                    │
│  NETWORK                                                           │
│  ├─ Bandwidth:        Limited                                      │
│  ├─ Connections:      500 max connections                          │
│  └─ Data Transfer:    Limited (fair use)                           │
│                                                                    │
│  FEATURES                                                          │
│  ├─ Regions:          60+ (AWS/Azure/GCP)                          │
│  ├─ Backups:          ❌ Not included                              │
│  ├─ PITR:             ❌ Not included                              │
│  ├─ Auto-Scaling:     ❌ Not included                              │
│  ├─ VPC Peering:      ❌ Not included                              │
│  ├─ Private Endpoint: ❌ Not included                              │
│  ├─ Analytics Nodes:  ❌ Not included                              │
│  ├─ Atlas Search:     ❌ Not included                              │
│  ├─ Vector Search:    ❌ Not included                              │
│  ├─ Monitoring:       ✅ Basic metrics                             │
│  ├─ Alerting:         ✅ Basic alerts                              │
│  └─ Support:          Community forums only                        │
│                                                                    │
│  LIMITATIONS                                                       │
│  ├─ Max 1 M0 cluster per project                                   │
│  ├─ Shared infrastructure (noisy neighbors)                        │
│  ├─ No SLA                                                         │
│  ├─ May be paused after inactivity (60 days)                       │
│  └─ Subject to fair use policy                                     │
│                                                                    │
│  💰 COST: $0 FOREVER                                               │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Cas d'Usage Recommandés

```
✅ IDÉAL POUR:                          ❌ PAS RECOMMANDÉ POUR:

• Apprentissage MongoDB                 • Applications production
• Prototypes et POCs                    • Données sensibles
• Demos et présentations                • Workloads critiques
• Side projects personnels              • High availability requis
• Tests d'intégration CI/CD             • Performance garantie
• Sandbox développement                 • Conformité (HIPAA, PCI-DSS)
• Documentation et tutoriels            • Stockage > 500MB
• MVP ultra-légers                      • Trafic soutenu
```

### Comparaison M0 vs Self-Hosted Local

```
┌────────────────────────────────────────────────────────────────┐
│              M0 vs MONGODB LOCAL (Docker/Standalone)           │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  CRITÈRE              M0 ATLAS          LOCAL MONGODB          │
│  ────────────────────────────────────────────────────────────  │
│  Setup Time           2 minutes         10-30 minutes          │
│  Infrastructure       Cloud (managed)   Your machine           │
│  Availability         99%+ (shared)     Depends on machine     │
│  Backup               None              Manual                 │
│  Monitoring           ✅ Included        Manual setup          │
│  Internet Access      Required          Not required           │
│  Collaboration        Easy (URL)        Complex (port forward) │
│  Version Updates      Automatic         Manual                 │
│  Scalability          Upgrade to M2+    Requires migration     │
│  Cost                 $0                $0 (your electricity)  │
│                                                                │
│  RECOMMANDATION:                                               │
│  • M0 pour collaboration, démos, learning en ligne             │
│  • Local pour offline, performance locale, contrôle total      │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Monitoring du Quota M0

```javascript
// Vérifier l'utilisation du M0 via Atlas API
GET /api/atlas/v1.0/groups/{projectId}/processes/{host}:{port}/databases

// Surveiller métriques clés
{
  "dataSize": 450000000,        // 450 MB / 512 MB
  "storageSize": 480000000,     // 480 MB / 512 MB
  "indexSize": 25000000,        // 25 MB
  "connections": 45,            // 45 / 500
  "warning": "Approaching storage limit"
}
```

**Stratégie de Graduation M0 → M10** :

```
Indicateurs de Migration M0 → M10:

├─ Storage > 400 MB (80% capacity)
├─ Connections régulières > 100
├─ Slow queries fréquentes
├─ Besoin de backups
├─ Passage en production
└─ Workload critique

→ Migration simple : Upgrade to M10 (1 click, ~5 min downtime)
```

---

## 💼 Tiers Shared : M2 et M5

### Spécifications et Pricing

```
┌───────────────────────────────────────────────────────────────────┐
│                     SHARED TIERS COMPARISON                       │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  TIER        M2                M5                                 │
│  ─────────────────────────────────────────────────────────────────│
│                                                                   │
│  RAM         2 GB              5 GB                               │
│  Storage     2 GB              5 GB                               │
│  vCPU        Shared            Shared                             │
│  IOPS        Limited           Limited                            │
│                                                                   │
│  Backups     ✅ Basic          ✅ Basic                           │
│  PITR        ❌ No             ❌ No                              │
│  Auto-Scale  ❌ No             ❌ No                              │
│  Monitoring  ✅ Standard       ✅ Standard                        │
│                                                                   │
│  Support     Community         Community                          │
│  SLA         None              None                               │
│                                                                   │
│  💰 MONTHLY COST (AWS us-east-1)                                  │
│  Standard    $9                $25                                │
│                                                                   │
│  USE CASE:                                                        │
│  • Small personal apps        • Medium personal apps              │
│  • Hobby projects             • Small team projects               │
│  • Non-critical workloads     • Development environments          │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### Limites des Tiers Shared

```
┌────────────────────────────────────────────────────────────────┐
│              SHARED INFRASTRUCTURE CONSTRAINTS                 │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  PROBLÈME              IMPACT                  MITIGATION      │
│  ────────────────────────────────────────────────────────────  │
│  Noisy Neighbors      Performance variable    → M10 Dedicated  │
│  Shared CPU           CPU throttling          → M10 Dedicated  │
│  Limited IOPS         Slow queries            → Optimize index │
│  No VPC Peering       Network latency         → M10 + Peering  │
│  No Private Link      Security limitations    → M10 + Private  │
│  Storage Cap          Growth blocked          → M10 Dedicated  │
│  No Auto-Scaling      Manual upgrades         → M10 + Auto-SC  │
│                                                                │
│  ⚠️ CRITICAL DECISION POINT:                                   │
│  If app is critical or going to prod → Skip M2/M5              │
│  Go directly to M10 (Dedicated) for $57/month                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## 🏢 Tiers Dedicated : M10 à M700+

### Pricing Matrix (AWS us-east-1)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                      DEDICATED TIERS PRICING MATRIX                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  TIER   RAM    vCPU  Storage    IOPS    Monthly    Hourly   Daily          │
│  ───────────────────────────────────────────────────────────────────────── │
│  M10    2GB    2     10-128GB   3K      $57        $0.08    $1.90          │
│  M20    4GB    2     10-256GB   3K      $140       $0.19    $4.67          │
│  M30    8GB    2     10-512GB   3K      $285       $0.39    $9.50          │
│  M40    16GB   4     10-1TB     6K      $630       $0.88    $21.00         │
│  M50    32GB   8     10-4TB     16K     $1,525     $2.12    $50.83         │
│  M60    64GB   16    10-4TB     16K     $3,050     $4.24    $101.67        │
│  M80    128GB  32    10-4TB     16K     $6,480     $9.00    $216.00        │
│  M140   192GB  48    10-4TB     25K     $10,800    $15.00   $360.00        │
│  M200   256GB  64    10-4TB     50K     $14,400    $20.00   $480.00        │
│  M300   384GB  96    10-4TB     50K     $23,040    $32.00   $768.00        │
│  M400+  Custom Custom Custom    Custom  Custom     Custom   Custom         │
│                                                                            │
│  * Prices for 3-node replica set                                           │
│  ** Storage pricing: $0.25/GB-month above included storage                 │
│  *** IOPS pricing: Additional IOPS available at extra cost                 │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

### Coûts par Provider et Région

```
┌────────────────────────────────────────────────────────────────────┐
│              M30 CLUSTER PRICING BY PROVIDER/REGION                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  PROVIDER   REGION              BASE COST    MULTIPLIER            │
│  ───────────────────────────────────────────────────────────────── │
│  AWS        us-east-1           $285         1.0x (baseline)       │
│  AWS        us-west-2           $285         1.0x                  │
│  AWS        eu-west-1           $310         1.09x                 │
│  AWS        eu-central-1        $310         1.09x                 │
│  AWS        ap-southeast-1      $315         1.11x                 │
│  AWS        ap-south-1          $320         1.12x                 │
│  AWS        sa-east-1           $340         1.19x                 │
│                                                                    │
│  Azure      East US             $290         1.02x                 │
│  Azure      West Europe         $315         1.11x                 │
│  Azure      Southeast Asia      $320         1.12x                 │
│                                                                    │
│  GCP        us-central1         $275         0.96x (cheapest!)     │
│  GCP        europe-west1        $300         1.05x                 │
│  GCP        asia-southeast1     $310         1.09x                 │
│                                                                    │
│  💡 TIP: GCP souvent 3-5% moins cher, mais AWS plus de régions     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Coûts Additionnels

#### 1. Storage Overages

```
┌──────────────────────────────────────────────────────────┐
│                 STORAGE PRICING                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Included Storage (par tier):                            │
│  • M10-M30:    10 GB included                            │
│  • M40+:       10 GB included (tier dépendant)           │
│                                                          │
│  Overage Pricing:                                        │
│  • Standard:   $0.25/GB-month                            │
│  • Low-Freq:   $0.02/GB-month (backups only)             │
│                                                          │
│  Exemple M30 avec 200 GB:                                │
│  • Base tier:        $285                                │
│  • Storage (190GB):  $47.50 (190 × $0.25)                │
│  • Total:            $332.50/month                       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

#### 2. IOPS Provisionnés

```
┌──────────────────────────────────────────────────────────┐
│                   IOPS PRICING                           │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Included IOPS (par tier):                               │
│  • M10-M30:   3,000 IOPS                                 │
│  • M40-M60:   6,000-16,000 IOPS                          │
│  • M80+:      16,000-50,000 IOPS                         │
│                                                          │
│  Additional IOPS:                                        │
│  • Cost: ~$0.10/IOPS-month (AWS EBS gp3 pricing)         │
│                                                          │
│  Exemple M30 avec 10,000 IOPS:                           │
│  • Base tier:              $285                          │
│  • Additional IOPS (7K):   $700 (7,000 × $0.10)          │
│  • Total:                  $985/month                    │
│                                                          │
│  💡 Si besoin > IOPS inclus → Consider higher tier       │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

#### 3. Data Transfer

```
┌──────────────────────────────────────────────────────────────┐
│                    DATA TRANSFER COSTS                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  DATA TRANSFER TYPE          COST (AWS)     COST (AZURE/GCP) │
│  ──────────────────────────────────────────────────────────  │
│  Inbound (Internet → Atlas)  FREE           FREE             │
│  Outbound (Atlas → Internet) $0.09/GB       $0.087-0.12/GB   │
│  Same Region (AZ → AZ)       FREE           FREE             │
│  Cross-Region                $0.02/GB       $0.02/GB         │
│  VPC Peering (same region)   FREE           FREE             │
│  PrivateLink                 $0.01/GB       N/A              │
│                                                              │
│  EXEMPLE CALCUL MENSUEL:                                     │
│  • Query results: 500 GB/month egress                        │
│  • Backups download: 100 GB/month                            │
│  • Total transfer: 600 GB                                    │
│  • Cost: 600 × $0.09 = $54/month                             │
│                                                              │
│  ⚠️ IMPACT: Pour read-heavy apps, data transfer peut         │
│  dépasser le coût du cluster lui-même !                      │
│                                                              │
│  OPTIMIZATION:                                               │
│  • Use compression                                           │
│  • Cache frequently accessed data                            │
│  • Deploy app servers in same region                         │
│  • Use VPC Peering (free transfer)                           │
│  • Consider Analytics Nodes for heavy reads                  │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

#### 4. Backup Storage

```
┌──────────────────────────────────────────────────────────┐
│                 BACKUP STORAGE COSTS                     │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  Continuous Cloud Backup:                                │
│  • Snapshot storage:  $0.20/GB-month                     │
│  • Oplog storage:     $0.20/GB-month                     │
│                                                          │
│  Calcul Estimation:                                      │
│  Formula = Data Size × Retention × Dedup Factor          │
│                                                          │
│  Exemple 200GB cluster, 7 daily + 4 weekly:              │
│  • Full snapshots: ~7 × 200GB = 1,400 GB                 │
│  • Deduplicated:   ~1,400 × 0.3 = 420 GB                 │
│  • Cost:           420 × $0.20 = $84/month               │
│                                                          │
│  ⚠️ Backup costs scale with data size and retention      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## ⚡ Serverless Instances

### Pricing Model

```
┌────────────────────────────────────────────────────────────────────┐
│                    SERVERLESS PRICING MODEL                        │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  PAY-PER-OPERATION (no fixed costs)                                │
│                                                                    │
│  OPERATION TYPE              COST                                  │
│  ───────────────────────────────────────────────────────────────── │
│  Read Operations             $0.10 per million                     │
│  Write Operations            $1.00 per million                     │
│  Storage                     $0.25/GB-month                        │
│  Backup Storage              $0.20/GB-month                        │
│  Data Transfer Out           $0.09/GB (AWS standard)               │
│                                                                    │
│  ────────────────────────────────────────────────────────────────  │
│                                                                    │
│  OPERATION DEFINITION:                                             │
│  • Read:   find(), findOne(), aggregate() that reads               │
│  • Write:  insert(), update(), delete()                            │
│  • Note:   Operations counted, not documents                       │
│                                                                    │
│  FREE TIER (per serverless instance):                              │
│  • 1 million reads/month                                           │
│  • 1 million writes/month                                          │
│  • 1 GB storage                                                    │
│  • 1 GB data transfer                                              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Comparaison Serverless vs Dedicated

```
┌────────────────────────────────────────────────────────────────────────┐
│              SERVERLESS vs M10 COST COMPARISON                         │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  WORKLOAD SCENARIO          SERVERLESS      M10 DEDICATED              │
│  ────────────────────────────────────────────────────────────────────  │
│  Scenario 1: Very Light Usage                                          │
│  • 5M reads/month           $0.40           $57.00                     │
│  • 500K writes/month        $0.00 (free)    $57.00                     │
│  • 2GB storage              $0.50           $57.00                     │
│  → TOTAL                    $0.90           $57.00  ✅ Serverless wins │
│                                                                        │
│  Scenario 2: Moderate Usage                                            │
│  • 50M reads/month          $4.90           $57.00                     │
│  • 10M writes/month         $9.00           $57.00                     │
│  • 10GB storage             $2.50           $57.00                     │
│  → TOTAL                    $16.40          $57.00  ✅ Serverless wins │
│                                                                        │
│  Scenario 3: Heavy Usage                                               │
│  • 200M reads/month         $19.90          $57.00                     │
│  • 50M writes/month         $49.00          $57.00                     │
│  • 20GB storage             $5.00           $57.00                     │
│  → TOTAL                    $73.90          $57.00  ✅ M10 wins        │
│                                                                        │
│  Scenario 4: Very Heavy Usage                                          │
│  • 1B reads/month           $99.90          $57.00                     │
│  • 200M writes/month        $199.00         $57.00                     │
│  • 50GB storage             $12.50          $57.00                     │
│  → TOTAL                    $311.40         $57.00  ✅ M10 wins        │
│                                                                        │
│  BREAK-EVEN POINT: ~150-200M operations/month                          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Cas d'Usage Serverless Optimal

```
✅ IDÉAL POUR:                          ⚠️ À ÉVITER POUR:

• Prototypes et MVPs                    • Workloads prévisibles constants
• Applications intermittentes           • High throughput sustained
• Trafic très variable/spiky            • Latency SLA stricte (<10ms)
• Side projects                         • Multi-région
• Webhooks et event processing          • VPC Peering requis
• Staging environments (utilisés 20%)   • Analytics workloads
• Microservices peu utilisés            • Batch processing intensif
• APIs peu fréquentées                  • Applications production critiques
```

---

## 💰 Calcul du TCO (Total Cost of Ownership)

### TCO sur 3 Ans : Exemple M40 Production

```
┌────────────────────────────────────────────────────────────────────────┐
│              TCO BREAKDOWN - M40 PRODUCTION (3 YEARS)                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ITEM                          MONTHLY      ANNUAL       3-YEAR        │
│  ────────────────────────────────────────────────────────────────────  │
│  Base Cluster (M40)            $630         $7,560       $22,680       │
│  Storage (200GB extra)         $50          $600         $1,800        │
│  Backup Storage (100GB)        $20          $240         $720          │
│  Data Transfer (500GB/mo)      $45          $540         $1,620        │
│  ────────────────────────────────────────────────────────────────────  │
│  SUBTOTAL                      $745         $8,940       $26,820       │
│                                                                        │
│  Support (Production - 10%)    $75          $900         $2,700        │
│  Reserved Capacity Discount    -$95         -$1,140      -$3,420       │
│  ────────────────────────────────────────────────────────────────────  │
│  TOTAL TCO                     $725         $8,700       $26,100       │
│                                                                        │
│  ────────────────────────────────────────────────────────────────────  │
│  EQUIVALENT SELF-HOSTED:                                               │
│  • Infrastructure              $200         $2,400       $7,200        │
│  • DBA Salary (20% time)       $2,500       $30,000      $90,000       │
│  • Tools & Monitoring          $100         $1,200       $3,600        │
│  • Backup Infrastructure       $150         $1,800       $5,400        │
│  ────────────────────────────────────────────────────────────────────  │
│  SELF-HOSTED TOTAL             $2,950       $35,400      $106,200      │
│                                                                        │
│  SAVINGS WITH ATLAS: $80,100 over 3 years (75% less)                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### TCO Calculator Formula

```python
# Atlas TCO Calculator
def calculate_atlas_tco(
    tier="M40",
    storage_gb=200,
    data_transfer_gb_month=500,
    backup_storage_gb=100,
    months=36,
    reserved_capacity=False,
    support_tier="production"
):
    """
    Calculate Total Cost of Ownership for Atlas
    """
    # Base pricing (example)
    tier_prices = {
        "M10": 57, "M20": 140, "M30": 285,
        "M40": 630, "M50": 1525, "M60": 3050,
        "M80": 6480, "M140": 10800, "M200": 14400
    }

    base_cost = tier_prices.get(tier, 0)

    # Storage cost (above included)
    storage_included = 10  # GB
    storage_cost = max(0, storage_gb - storage_included) * 0.25

    # Backup storage
    backup_cost = backup_storage_gb * 0.20

    # Data transfer
    transfer_cost = data_transfer_gb_month * 0.09

    # Monthly subtotal
    monthly = base_cost + storage_cost + backup_cost + transfer_cost

    # Support costs
    support_multiplier = {
        "none": 0,
        "developer": 0.05,
        "production": 0.10,
        "enterprise": 0.15
    }
    support_cost = monthly * support_multiplier.get(support_tier, 0)

    # Reserved capacity discount
    if reserved_capacity:
        discount = 0.15  # 15% discount
        monthly = monthly * (1 - discount)

    # Total
    total_monthly = monthly + support_cost
    total_tco = total_monthly * months

    return {
        "monthly": round(total_monthly, 2),
        "annual": round(total_monthly * 12, 2),
        "total_tco": round(total_tco, 2)
    }

# Example usage
cost = calculate_atlas_tco(
    tier="M40",
    storage_gb=200,
    data_transfer_gb_month=500,
    backup_storage_gb=100,
    months=36,
    reserved_capacity=True,
    support_tier="production"
)
print(f"Monthly: ${cost['monthly']}")
print(f"3-Year TCO: ${cost['total_tco']}")
```

---

## 🎯 Stratégies d'Optimisation des Coûts

### 1. Reserved Capacity (1-3 ans)

```
┌────────────────────────────────────────────────────────────────────┐
│                    RESERVED CAPACITY PRICING                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  COMMITMENT      DISCOUNT       EXAMPLE M40                        │
│  ───────────────────────────────────────────────────────────────── │
│  On-Demand       0%             $630/month                         │
│  1-Year          -14%           $542/month  (save $1,056/year)     │
│  3-Year          -30%           $441/month  (save $6,804/3-year)   │
│                                                                    │
│  ────────────────────────────────────────────────────────────────  │
│                                                                    │
│  ELIGIBILITY:                                                      │
│  • M10+ dedicated clusters only                                    │
│  • Minimum commitment per cluster                                  │
│  • Flexible: can change tier within commitment                     │
│                                                                    │
│  PAYMENT OPTIONS:                                                  │
│  • All Upfront (highest discount)                                  │
│  • Partial Upfront                                                 │
│  • No Upfront (monthly, lower discount)                            │
│                                                                    │
│  BEST PRACTICES:                                                   │
│  ✅ Use for stable production workloads                            │
│  ✅ Start with 1-year, renew if workload stable                    │
│  ❌ Avoid for dev/test (variable usage)                            │
│  ❌ Avoid for new apps (unknown growth)                            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 2. Cluster Pausing (Non-Production)

```
┌─────────────────────────────────────────────────────────┐
│              CLUSTER PAUSING STRATEGY                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  CONCEPT: Pause clusters when not in use                │
│  • Available for: M10-M40 (not M50+)                    │
│  • Paused state: No compute cost, only storage          │
│  • Resume time: ~30 seconds to 2 minutes                │
│                                                         │
│  COST SAVINGS EXAMPLE (M20 Dev Cluster):                │
│  ┌─────────────────────────────────────────────────┐    │
│  │ Without Pausing:                                │    │
│  │ • 24/7 operation: $140/month                    │    │
│  │                                                 │    │
│  │ With Pausing (nights + weekends):               │    │
│  │ • Active: 8 hours/day × 5 days = 160 hrs/mo     │    │
│  │ • Paused: 580 hrs/mo                            │    │
│  │ • Compute cost: $140 × (160/720) = $31          │    │
│  │ • Storage cost: $2.50 (10GB)                    │    │
│  │ • Total: $33.50/month                           │    │
│  │                                                 │    │
│  │ SAVINGS: $106.50/month (76% reduction!)         │    │
│  └─────────────────────────────────────────────────┘    │
│                                                         │
│  AUTOMATION:                                            │
│  ```bash                                                │
│  # Pause cluster at 18:00                               │
│  atlas clusters pause myDevCluster \                    │
│    --projectId xxx                                      │
│                                                         │
│  # Resume at 08:00                                      │
│  atlas clusters start myDevCluster \                    │
│    --projectId xxx                                      │
│  ```                                                    │
│                                                         │
│  USE CASES:                                             │
│  ✅ Development environments (pause nights/weekends)    │
│  ✅ Staging (pause when not testing)                    │
│  ✅ Demo environments                                   │
│  ❌ Production (always-on requirement)                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 3. Right-Sizing et Auto-Scaling

```
┌────────────────────────────────────────────────────────────────┐
│                RIGHT-SIZING METHODOLOGY                        │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  STEP 1: BASELINE ANALYSIS                                     │
│  ────────────────────────                                      │
│  Monitor for 2-4 weeks:                                        │
│  • Peak CPU usage                                              │
│  • Peak RAM usage                                              │
│  • Peak IOPS                                                   │
│  • Peak Connections                                            │
│                                                                │
│  STEP 2: APPLY SIZING RULES                                    │
│  ─────────────────────────────                                 │
│  Rule of Thumb:                                                │
│  • Target 60-70% utilization (sustained)                       │
│  • Allow 30-40% headroom for spikes                            │
│                                                                │
│  Example Analysis:                                             │
│  Current: M60 (64GB RAM)                                       │
│  • Peak RAM: 28GB (44% utilization)                            │
│  • Avg RAM: 22GB (34% utilization)                             │
│  → OVER-PROVISIONED                                            │
│                                                                │
│  Recommendation: Downgrade to M40 (16GB)                       │
│  • With 22GB working set → Need 32GB tier                      │
│  → Downgrade to M50 (32GB)                                     │
│  → Savings: $1,525/month ($18,300/year)                        │
│                                                                │
│  STEP 3: ENABLE AUTO-SCALING                                   │
│  ──────────────────────────────                                │
│  ```yaml                                                       │
│  auto_scaling:                                                 │
│    compute:                                                    │
│      enabled: true                                             │
│      min_instance_size: "M40"                                  │
│      max_instance_size: "M60"                                  │
│      scale_down_enabled: true                                  │
│  ```                                                           │
│                                                                │
│  Benefits:                                                     │
│  • Runs at M40 during normal hours (60%)                       │
│  • Scales to M50/M60 during peaks (10%)                        │
│  • Average cost: ~M45 equivalent                               │
│  • Savings vs always-M60: ~$900/month                          │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 4. Data Tiering (Hot/Cold)

```
┌─────────────────────────────────────────────────────────────┐
│                   DATA TIERING STRATEGY                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CONCEPT: Separate hot (active) and cold (archival) data    │
│                                                             │
│  ARCHITECTURE:                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                                                     │    │
│  │  HOT DATA (Atlas Cluster M40)                       │    │
│  │  • Last 90 days                                     │    │
│  │  • Size: 200 GB                                     │    │
│  │  • Cost: $630/month                                 │    │
│  │                                                     │    │
│  │  ───────────► ETL Process ───────────►              │    │
│  │                                                     │    │
│  │  COLD DATA (Atlas Data Lake)                        │    │
│  │  • >90 days historical                              │    │
│  │  • Size: 2 TB (S3)                                  │    │
│  │  • Storage: 2000 × $0.023 = $46/month               │    │
│  │  • Query cost: $0.10/GB scanned (occasional)        │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
│  COST COMPARISON:                                           │
│                                                             │
│  All in Atlas Cluster (2.2TB):                              │
│  • Need M80+ tier: $6,480/month                             │
│  • Storage: 2,200 × $0.25 = $550/month                      │
│  • Total: ~$7,030/month                                     │
│                                                             │
│  With Data Tiering:                                         │
│  • M40 cluster: $630/month                                  │
│  • S3 storage: $46/month                                    │
│  • Data Lake queries: ~$20/month                            │
│  • Total: ~$696/month                                       │
│                                                             │
│  SAVINGS: $6,334/month (90% reduction!)                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 5. Multi-Environnement Optimization

```
┌───────────────────────────────────────────────────────────────────┐
│            ENVIRONMENT-BASED SIZING STRATEGY                      │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ENVIRONMENT    REQUIREMENTS         TIER        MONTHLY COST     │
│  ─────────────────────────────────────────────────────────────────│
│                                                                   │
│  Production     • High availability   M60        $3,050           │
│                 • Multi-region        3 regions                   │
│                 • 24/7 uptime         Auto-scale                  │
│                 • PITR backups        Support                     │
│                 • VPC Peering                                     │
│                                                                   │
│  Staging        • Similar to prod     M30        $285             │
│                 • Single region       1 region                    │
│                 • Testing workload    Basic backup                │
│                 • Can tolerate brief                              │
│                   downtime                                        │
│                                                                   │
│  QA/Testing     • Functional testing  M10        $57              │
│                 • Single region       1 region                    │
│                 • Low usage           Pause nights                │
│                 • No backup needed    + weekends                  │
│                                                                   │
│  Development    • Developer use       Serverless $5-10            │
│                 • Highly variable     or M0      or FREE          │
│                 • Pause when idle     Auto-pause                  │
│                                                                   │
│  CI/CD          • Pipeline tests      M0 (Free)  $0               │
│                 • Ephemeral           Create +                    │
│                 • Short-lived         Delete                      │
│                                                                   │
│  ───────────────────────────────────────────────────────────────  │
│  TOTAL MONTHLY COST:        $3,397 - $3,402                       │
│                                                                   │
│  ❌ BAD PRACTICE (same tier everywhere):                          │
│  5 × M60 = $15,250/month                                          │
│                                                                   │
│  ✅ OPTIMIZED APPROACH:                                           │
│  Savings: $11,848/month (78% reduction!)                          │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 6. Compression et Query Optimization

```
┌──────────────────────────────────────────────────────────┐
│          STORAGE & TRANSFER OPTIMIZATION                 │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  TECHNIQUE           SAVINGS      IMPLEMENTATION         │
│  ────────────────────────────────────────────────────    │
│                                                          │
│  1. WiredTiger       30-80%       Enabled by default     │
│     Compression      storage      (snappy)               │
│                      reduction                           │
│                                                          │
│  2. Projection       50-90%       Always project only    │
│     (select fields)  network      needed fields:         │
│                      bandwidth    find({}, {name: 1})    │
│                                                          │
│  3. Index-Covered    90%+         Use covered queries    │
│     Queries          disk I/O     (no document fetch)    │
│                      reduction                           │
│                                                          │
│  4. Aggregation      40-60%       $project early in      │
│     $project early   memory       pipeline               │
│                      usage                               │
│                                                          │
│  5. TTL Indexes      100%         Auto-delete old docs:  │
│     (auto-delete)    storage      createIndex(           │
│                      for old data   {createdAt: 1},      │
│                                     {expireAfterSec}     │
│                                   )                      │
│                                                          │
│  EXAMPLE IMPACT:                                         │
│  Before optimization:                                    │
│  • Storage: 500 GB × $0.25 = $125/month                  │
│  • Transfer: 1 TB × $0.09 = $90/month                    │
│  • Total: $215/month                                     │
│                                                          │
│  After optimization:                                     │
│  • Storage: 200 GB × $0.25 = $50/month (TTL, compress)   │
│  • Transfer: 300 GB × $0.09 = $27/month (projections)    │
│  • Total: $77/month                                      │
│                                                          │
│  SAVINGS: $138/month (64% reduction)                     │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 📊 Pricing Models Comparison

### Atlas vs Competitors

```
┌────────────────────────────────────────────────────────────────────────┐
│          MONGODB ATLAS vs AWS DOCUMENTDB vs COSMOS DB                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  PROVIDER       TIER          MONTHLY COST    NOTES                    │
│  ────────────────────────────────────────────────────────────────────  │
│  MongoDB        M40           $630            • Official MongoDB       │
│  Atlas          (16GB, 4vCPU)                 • Full features          │
│                                                • Multi-cloud           │
│                                                                        │
│  AWS            db.r5.xlarge  ~$540           • MongoDB 4.0 compatible │
│  DocumentDB     (32GB, 4vCPU)                 • Limited features       │
│                                                • AWS only              │
│                                                • No transactions       │
│                                                                        │
│  Azure          400 RU/s      ~$600           • Different model        │
│  Cosmos DB      (~16GB equiv)                 • Pay per RU/s           │
│                                                • Global distribution   │
│                                                • Multi-model           │
│                                                                        │
│  ────────────────────────────────────────────────────────────────────  │
│                                                                        │
│  KEY DIFFERENTIATORS:                                                  │
│  • Atlas: True MongoDB, all features, multi-cloud flexibility          │
│  • DocumentDB: Cheaper but limited MongoDB compatibility               │
│  • Cosmos DB: Different API, global by default, more expensive         │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🎯 Guide de Décision : Quel Modèle Choisir ?

```
┌────────────────────────────────────────────────────────────────────┐
│                    PRICING MODEL DECISION TREE                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  START: What's your use case?                                      │
│    │                                                               │
│    ├──► Learning MongoDB                                           │
│    │    └──► M0 (Free Forever)                                     │
│    │                                                               │
│    ├──► Prototype / POC                                            │
│    │    ├──► Variable traffic  → Serverless                        │
│    │    └──► Constant light    → M0 or M10                         │
│    │                                                               │
│    ├──► Development Environment                                    │
│    │    ├──► Team < 5          → M0 or Serverless                  │
│    │    ├──► Team 5-20         → M10 (with pausing)                │
│    │    └──► Team 20+          → M20 (with pausing)                │
│    │                                                               │
│    ├──► Staging / QA                                               │
│    │    └──► M10-M30 (based on prod size, 1 tier lower)            │
│    │                                                               │
│    ├──► Production (Small)                                         │
│    │    ├──► Data < 50GB       → M20-M30                           │
│    │    └──► Consistent load   → Consider Reserved Capacity        │
│    │                                                               │
│    ├──► Production (Medium)                                        │
│    │    ├──► Data 50-500GB     → M30-M50                           │
│    │    ├──► Variable load     → Enable Auto-Scaling               │
│    │    └──► Multi-region      → M40+ with replication             │
│    │                                                               │
│    ├──► Production (Large)                                         │
│    │    ├──► Data 500GB-5TB    → M60-M140                          │
│    │    ├──► High IOPS         → M140+ (25K-50K IOPS)              │
│    │    └──► Analytics         → Add Analytics Nodes               │
│    │                                                               │
│    └──► Production (Enterprise)                                    │
│         ├──► Data > 5TB        → M200+ with Sharding               │
│         ├──► Global users      → Multi-region clusters             │
│         ├──► Compliance        → Enterprise features               │
│         └──► 24/7 support      → Enterprise support plan           │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🏁 Résumé et Best Practices

### Récapitulatif des Coûts

| Tier | RAM | Monthly Cost | Ideal For |
|------|-----|--------------|-----------|
| **M0** | 512MB | **$0** | Learning, demos, POCs |
| **M2** | 2GB | $9 | Small hobby projects |
| **M5** | 5GB | $25 | Personal apps |
| **M10** | 2GB | $57 | Dev/test, small prod |
| **M20** | 4GB | $140 | Small production |
| **M30** | 8GB | $285 | Medium production |
| **M40** | 16GB | $630 | Production |
| **M50** | 32GB | $1,525 | Large production |
| **M60** | 64GB | $3,050 | Enterprise |
| **M80+** | 128GB+ | $6,480+ | Very large enterprise |
| **Serverless** | Auto | Variable | Intermittent, variable |

### Top 10 Cost Optimization Tips

```
1. ✅ Start Small, Scale Up
   → Begin with M10, add resources as needed

2. ✅ Use Reserved Capacity for Production
   → Save 14-30% on stable workloads

3. ✅ Pause Non-Production Clusters
   → Save 70-80% on dev/test environments

4. ✅ Enable Auto-Scaling
   → Pay only for what you use

5. ✅ Right-Size Regularly
   → Review utilization monthly, adjust tiers

6. ✅ Use Data Tiering
   → Move cold data to Data Lake (90% cheaper)

7. ✅ Optimize Queries
   → Reduce data transfer costs with projections

8. ✅ Use VPC Peering
   → Eliminate data transfer costs (same region)

9. ✅ Monitor and Alert on Budget
   → Set budget alerts to avoid surprises

10. ✅ Choose Appropriate Support Tier
    → Don't pay for enterprise support if not needed
```

### Red Flags (Coûts Inattendus)

```
⚠️ WATCH OUT FOR:

❌ Data Transfer Costs
   → Can exceed cluster cost for read-heavy apps
   → Solution: VPC Peering, caching, compression

❌ Backup Storage Accumulation
   → Snapshots + PITR can add up
   → Solution: Optimize retention policy

❌ Over-Provisioned Clusters
   → Running M60 with 20% utilization
   → Solution: Right-size + auto-scaling

❌ Too Many Environments at High Tiers
   → Dev/staging on M40+ unnecessarily
   → Solution: Tier appropriately per environment

❌ Idle Clusters Running 24/7
   → Dev clusters on weekends/nights
   → Solution: Implement pausing automation
```

---


⏭️ [Configuration réseau et sécurité](/14-mongodb-atlas/04-configuration-reseau-securite.md)
