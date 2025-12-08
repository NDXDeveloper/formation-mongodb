🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 14.7 Backups et Restauration

## Introduction

Les backups sont votre **dernière ligne de défense** contre la perte de données. Un backup n'a de valeur que s'il est testable, restaurable et répond à vos objectifs de RPO (Recovery Point Objective) et RTO (Recovery Time Objective). Atlas fournit un système de backup automatisé avec snapshots, Point-in-Time Recovery (PITR), et options de restauration flexibles. Cette section guide les équipes DevOps et SRE dans la mise en place d'une stratégie de backup et disaster recovery production-ready.

### 🎯 Objectifs de cette Section

- Comprendre l'architecture des backups Atlas
- Configurer les backups automatiques et PITR
- Maîtriser les différents scénarios de restauration
- Implémenter une stratégie de Disaster Recovery (DR)
- Tester régulièrement les procédures de restauration
- Respecter les exigences de compliance et retention

---

## 🏗️ Architecture des Backups Atlas

### Vue d'Ensemble

```
┌───────────────────────────────────────────────────────────────────────┐
│                    ATLAS BACKUP ARCHITECTURE                          │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│   PRODUCTION CLUSTER                                                  │
│   ┌──────────────────────────────────────────────────────────────────┐│
│   │                                                                  ││
│   │  Primary      Secondary     Secondary                            ││
│   │  [Node 1]     [Node 2]      [Node 3]                             ││
│   │     │            │              │                                ││
│   │     │            │              │                                ││
│   │     └────────────┴──────────────┘                                ││
│   │              Continuous Oplog Capture                            ││
│   │                        │                                         ││
│   └────────────────────────┼─────────────────────────────────────────┘│
│                            │                                          │
│                            ▼                                          │
│   BACKUP SERVICE (Atlas-managed)                                      │
│   ┌──────────────────────────────────────────────────────────────────┐│
│   │                                                                  ││
│   │  ┌─────────────────────────────────────────────────────────────┐ ││
│   │  │ SNAPSHOT SCHEDULER                                          │ ││
│   │  │ • Hourly snapshots (48h retention)                          │ ││
│   │  │ • Daily snapshots (7 days retention)                        │ ││
│   │  │ • Weekly snapshots (4 weeks retention)                      │ ││
│   │  │ • Monthly snapshots (12 months retention)                   │ ││
│   │  └─────────────────────────────────────────────────────────────┘ ││
│   │                                                                  ││
│   │  ┌─────────────────────────────────────────────────────────────┐ ││
│   │  │ OPLOG STORAGE (Point-in-Time Recovery)                      │ ││
│   │  │ • Continuous oplog capture                                  │ ││
│   │  │ • 72-hour window by default                                 │ ││
│   │  │ • Allows restore to any point in time                       │ ││
│   │  └─────────────────────────────────────────────────────────────┘ ││
│   │                                                                  ││
│   └──────────────────────────────────────────────────────────────────┘│
│                            ▼                                          │
│   STORAGE (Cloud Provider)                                            │
│   ┌──────────────────────────────────────────────────────────────────┐│
│   │                                                                  ││
│   │  Primary Region Storage         Cross-Region Storage (Optional)  ││
│   │  ┌──────────────────────┐       ┌──────────────────────┐         ││
│   │  │ AWS S3 (us-east-1)   │       │ AWS S3 (us-west-2)   │         ││
│   │  │ • Snapshots          │  ───► │ • Copy of snapshots  │         ││
│   │  │ • Oplog chunks       │       │ • Disaster recovery  │         ││
│   │  │ • Encrypted          │       │ • Compliance         │         ││
│   │  └──────────────────────┘       └──────────────────────┘         ││
│   │                                                                  ││
│   └──────────────────────────────────────────────────────────────────┘│
│                                                                       │
│   RESTORE OPTIONS                                                     │
│   ┌──────────────────────────────────────────────────────────────────┐│
│   │ 1. Automated Restore (new cluster from snapshot)                 ││
│   │ 2. Point-in-Time Restore (to specific timestamp)                 ││
│   │ 3. Download Snapshot (manual restore)                            ││
│   │ 4. Query Snapshot (without full restore)                         ││
│   └──────────────────────────────────────────────────────────────────┘│
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Types de Backups

```
┌────────────────────────────────────────────────────────────────────────┐
│                  BACKUP TYPES COMPARISON                               │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  TYPE                DESCRIPTION              AVAILABILITY             │
│  ───────────────────────────────────────────────────────────────────── │
│  Cloud Backup        • Automated snapshots    M10+ (Dedicated)         │
│  (Recommended)       • Point-in-Time Recovery                          │
│                      • Atlas-managed storage                           │
│                      • Encryption at rest                              │
│                      • Cross-region copy                               │
│                      • Query without restore                           │
│                                                                        │
│  Legacy Backup       • Deprecated             M10-M40 (Legacy)         │
│  (Deprecated)        • Being migrated         ⚠️ Migrate to Cloud      │
│                      • Less features                                   │
│                                                                        │
│  No Backup           • Manual responsibility  M0, M2, M5 (Shared)      │
│                      • Export via mongodump   ❌ Not for production    │
│                      • Client-side backups                             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Configuration des Backups

### Politique de Backup par Défaut

```yaml
# Default Cloud Backup Policy (M10+)
backup_policy:
  # Snapshot Schedule
  snapshot_schedule:
    hourly:
      frequency: 1  # Every hour
      retention: 2  # 2 days (48 snapshots)

    daily:
      time: "03:00"  # UTC
      retention: 7   # 7 days

    weekly:
      day: "SATURDAY"
      time: "03:00"
      retention: 4   # 4 weeks

    monthly:
      day: 1  # First day of month
      time: "03:00"
      retention: 12  # 12 months

  # Point-in-Time Recovery
  pitr:
    enabled: true
    window_hours: 72  # 3 days

  # Encryption
  encryption:
    enabled: true
    type: "AES-256"

  # Cross-Region Copy (Optional)
  copy_settings:
    enabled: false
    regions: []
```

### Configuration Avancée

```hcl
# Terraform: Custom Backup Configuration
resource "mongodbatlas_cloud_backup_schedule" "production" {
  project_id   = var.atlas_project_id
  cluster_name = "production-cluster"

  # Policy for automated snapshots
  policy_item_hourly {
    frequency_interval = 2    # Every 2 hours (instead of 1)
    retention_unit     = "days"
    retention_value    = 3    # Keep for 3 days
  }

  policy_item_daily {
    frequency_interval = 1    # Every day
    retention_unit     = "days"
    retention_value    = 14   # Keep for 2 weeks
    time               = "04:00"  # 4 AM UTC
  }

  policy_item_weekly {
    frequency_interval = 1    # Every week
    retention_unit     = "weeks"
    retention_value    = 8    # Keep for 8 weeks
    day_of_week        = "SUNDAY"
    time               = "04:00"
  }

  policy_item_monthly {
    frequency_interval = 1    # Every month
    retention_unit     = "months"
    retention_value    = 24   # Keep for 2 years
    day_of_month       = 1
    time               = "04:00"
  }

  # Point-in-Time Recovery
  reference_hour_of_day    = 4
  reference_minute_of_hour = 0
  restore_window_days      = 7  # Extended to 7 days

  # Auto export to S3 (for compliance)
  auto_export_enabled = true

  export_bucket_id = mongodbatlas_cloud_backup_snapshot_export_bucket.compliance.id
}

# S3 Bucket for backup exports
resource "mongodbatlas_cloud_backup_snapshot_export_bucket" "compliance" {
  project_id   = var.atlas_project_id
  iam_role_id  = var.aws_iam_role_id
  bucket_name  = "mongodb-backups-compliance"
  cloud_provider = "AWS"
}
```

### Cross-Region Backup

```hcl
# Terraform: Cross-Region Backup Copy
resource "mongodbatlas_cloud_backup_schedule" "production_with_dr" {
  project_id   = var.atlas_project_id
  cluster_name = "production-cluster"

  # Standard policy
  policy_item_daily {
    frequency_interval = 1
    retention_unit     = "days"
    retention_value    = 7
    time               = "03:00"
  }

  # Copy snapshots to DR region
  copy_settings {
    cloud_provider     = "AWS"
    region_name        = "US_WEST_2"  # DR region
    should_copy_oplogs = true         # Include oplog for PITR
    frequencies        = ["DAILY", "WEEKLY", "MONTHLY"]
  }
}
```

---

## 🔄 Point-in-Time Recovery (PITR)

### Concept PITR

```
┌───────────────────────────────────────────────────────────────────────┐
│                   POINT-IN-TIME RECOVERY CONCEPT                      │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Timeline:                                                            │
│  ─────────────────────────────────────────────────────────────────────│
│                                                                       │
│  Day 1          Day 2          Day 3          Day 4 (Today)           │
│  │              │              │              │                       │
│  ├──────────────┼──────────────┼──────────────┼──────►                │
│  │              │              │              │                       │
│  Snapshot      Snapshot       Snapshot       Current                  │
│  00:00         00:00          00:00          State                    │
│                                                                       │
│  ───────────────────────────────────────────────────────              │
│          Continuous Oplog Capture (72 hours)                          │
│  ───────────────────────────────────────────────────────              │
│                                                                       │
│  PITR Window: Last 72 hours                                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │                                                                  │ │
│  │  Day 2, 14:35:27  ◄── You can restore to ANY point in this window│ │
│  │  Day 3, 09:12:45  ◄── Example: Before bad deployment             │ │
│  │  Day 3, 22:30:00  ◄── Example: Before corruption event           │ │
│  │                                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  HOW IT WORKS:                                                        │
│  1. Find the most recent snapshot BEFORE target time                  │
│  2. Apply oplog operations from snapshot to target timestamp          │
│  3. Result: Database state at exact target time                       │
│                                                                       │
│  USE CASES:                                                           │
│  ✅ Accidental data deletion                                          │
│  ✅ Bad deployment rollback                                           │
│  ✅ Corruption recovery                                               │
│  ✅ User error correction                                             │
│  ✅ Forensic analysis                                                 │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

### Restauration PITR

```bash
# Via Atlas UI:
# 1. Browse Backups → Select Cluster
# 2. Click "Restore"
# 3. Select "Point in Time"
# 4. Choose timestamp: 2025-12-08 14:35:27 UTC
# 5. Select restore method (new cluster or replace existing)

# Via Atlas API
curl -X POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/restoreJobs" \
  -u "${PUBLIC_KEY}:${PRIVATE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "deliveryType": "automated",
    "targetClusterName": "production-cluster-restore",
    "targetGroupId": "'${PROJECT_ID}'",
    "pointInTimeUTCSeconds": 1733665527
  }'
```

### Limitations PITR

```
┌────────────────────────────────────────────────────────────────────────┐
│                    PITR LIMITATIONS & CONSIDERATIONS                   │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  CONSTRAINT              DETAIL                                        │
│  ───────────────────────────────────────────────────────────────────── │
│  Time Window             72 hours by default                           │
│                          (configurable up to 7 days)                   │
│                                                                        │
│  Cluster Tier            M10+ only (not available on Shared)           │
│                                                                        │
│  Restore Time            Proportional to:                              │
│                          • Cluster size                                │
│                          • Oplog operations to replay                  │
│                          Typical: 15-60 minutes                        │
│                                                                        │
│  Oplog Coverage          Must have continuous oplog                    │
│                          Gaps = Cannot restore to those times          │
│                                                                        │
│  Precision               1-second granularity                          │
│                          Can specify exact timestamp                   │
│                                                                        │
│  Storage Cost            ~$0.20/GB-month for oplog storage             │
│                          Included in backup cost                       │
│                                                                        │
│  Sharded Clusters        Restores entire cluster                       │
│                          Cannot restore individual shards              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🔧 Scénarios de Restauration

### 1. Restauration vers Nouveau Cluster (Recommended)

```
┌────────────────────────────────────────────────────────────────────────┐
│               RESTORE TO NEW CLUSTER (Safe Method)                     │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  WORKFLOW:                                                             │
│  ─────────                                                             │
│                                                                        │
│  1. INITIATE RESTORE                                                   │
│     • Choose snapshot or PITR timestamp                                │
│     • Select "Restore to new cluster"                                  │
│     • Specify cluster name: "production-restore-20251208"              │
│                                                                        │
│  2. PROVISIONING (~10-20 minutes)                                      │
│     • Atlas creates new cluster                                        │
│     • Restores data from backup                                        │
│     • Original cluster remains untouched                               │
│                                                                        │
│  3. VALIDATION                                                         │
│     • Connect to restore cluster                                       │
│     • Verify data integrity                                            │
│     • Check row counts, sample queries                                 │
│     • Compare with production                                          │
│                                                                        │
│  4. DECISION                                                           │
│     Option A: Promote restore cluster                                  │
│     • Update connection strings in apps                                │
│     • Migrate traffic to restored cluster                              │
│     • Delete old cluster after validation                              │
│                                                                        │
│     Option B: Selective data recovery                                  │
│     • Export needed data from restore cluster                          │
│     • Import to production cluster                                     │
│     • Delete restore cluster                                           │
│                                                                        │
│  ADVANTAGES:                                                           │
│  ✅ Zero risk to production                                            │
│  ✅ Time to validate before switching                                  │
│  ✅ Can abort if issues found                                          │
│  ✅ Parallel operation (no downtime)                                   │
│                                                                        │
│  DISADVANTAGES:                                                        │
│  ⚠️ Requires connection string update                                  │
│  ⚠️ Extra cost during validation period                                │
│  ⚠️ Manual traffic migration                                           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 2. Restauration en Place (Replace Existing)

```
┌────────────────────────────────────────────────────────────────────────┐
│                RESTORE IN-PLACE (Replace Existing)                     │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  WORKFLOW:                                                             │
│  ─────────                                                             │
│                                                                        │
│  1. INITIATE RESTORE                                                   │
│     • Choose snapshot or PITR timestamp                                │
│     • Select "Replace existing cluster"                                │
│     • Confirmation required (destructive!)                             │
│                                                                        │
│  2. CLUSTER SHUTDOWN                                                   │
│     • Atlas stops the cluster                                          │
│     • All connections terminated                                       │
│     • ⚠️ DOWNTIME BEGINS                                               │
│                                                                        │
│  3. DATA REPLACEMENT (~15-60 minutes)                                  │
│     • Current data deleted                                             │
│     • Backup data restored                                             │
│     • Cannot abort once started                                        │
│                                                                        │
│  4. CLUSTER RESTART                                                    │
│     • Cluster comes back online                                        │
│     • Same connection strings work                                     │
│     • ⚠️ DOWNTIME ENDS                                                 │
│                                                                        │
│  ADVANTAGES:                                                           │
│  ✅ No connection string changes                                       │
│  ✅ Automatic traffic resumption                                       │
│  ✅ No extra costs                                                     │
│                                                                        │
│  DISADVANTAGES:                                                        │
│  ❌ Downtime (15-60+ minutes)                                          │
│  ❌ Cannot validate before replacing                                   │
│  ❌ Destructive (current data lost)                                    │
│  ❌ Cannot abort once started                                          │
│                                                                        │
│  USE ONLY WHEN:                                                        │
│  • Disaster recovery (production corrupted)                            │
│  • Downtime acceptable                                                 │
│  • 100% confident in backup validity                                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 3. Download Snapshot (Manual Restore)

```bash
# Download snapshot for manual restore
# Use case: Restore to non-Atlas environment, forensic analysis

# Via Atlas API
curl -X POST \
  "https://cloud.mongodb.com/api/atlas/v1.0/groups/${PROJECT_ID}/clusters/${CLUSTER_NAME}/backup/restoreJobs" \
  -u "${PUBLIC_KEY}:${PRIVATE_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "deliveryType": "download",
    "snapshotId": "'${SNAPSHOT_ID}'"
  }'

# Response includes download URL
# Download and extract
wget "${DOWNLOAD_URL}" -O backup.tar.gz
tar -xzf backup.tar.gz

# Restore with mongorestore
mongorestore --uri="mongodb://localhost:27017" \
  --dir=./backup \
  --drop

# Use cases:
# ✅ Restore to self-hosted MongoDB
# ✅ Forensic analysis on separate system
# ✅ Data migration to different environment
# ✅ Compliance requirement to have offline backup
```

### 4. Query Snapshot (No Restore)

```
┌────────────────────────────────────────────────────────────────────────┐
│                  QUERYABLE BACKUPS (No Full Restore)                   │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Feature: Query snapshots directly without full restore                │
│                                                                        │
│  USE CASES:                                                            │
│  • Data forensics: "What was the value at time X?"                     │
│  • Selective recovery: "Restore just one document"                     │
│  • Compliance audits: "Show data state on date Y"                      │
│  • Analysis: "How did data evolve over time?"                          │
│                                                                        │
│  PROCESS:                                                              │
│  1. Atlas UI → Backups → Select Snapshot → "Query Snapshot"            │
│  2. Read-only connection string provided                               │
│  3. Connect with mongosh or application                                │
│  4. Query data as needed                                               │
│  5. Snapshot automatically deleted after 24 hours                      │
│                                                                        │
│  EXAMPLE:                                                              │
│  # Connect to queryable snapshot                                       │
│  mongosh "mongodb+srv://snapshot-xxxxx.mongodb.net/mydb"               │
│                                                                        │
│  # Query historical data                                               │
│  db.orders.findOne({ orderId: "12345" })                               │
│                                                                        │
│  # If data looks good, export it                                       │
│  mongoexport --uri="..." \                                             │
│    --collection=orders \                                               │
│    --query='{"orderId":"12345"}' \                                     │
│    --out=recovered_order.json                                          │
│                                                                        │
│  # Import to production                                                │
│  mongoimport --uri="production-uri" \                                  │
│    --collection=orders \                                               │
│    --file=recovered_order.json                                         │
│                                                                        │
│  LIMITATIONS:                                                          │
│  • Read-only access                                                    │
│  • 24-hour availability                                                │
│  • Additional cost per hour                                            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🚨 Disaster Recovery (DR)

### RPO et RTO Objectives

```
┌────────────────────────────────────────────────────────────────────────┐
│                    RPO & RTO DEFINITIONS                               │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  RPO (Recovery Point Objective)                                        │
│  ───────────────────────────────────                                   │
│  • Maximum acceptable data loss                                        │
│  • Measured in time: How old can restored data be?                     │
│                                                                        │
│  Examples:                                                             │
│  • RPO = 1 hour: Can lose max 1 hour of data                           │
│  • RPO = 5 minutes: Can lose max 5 minutes                             │
│  • RPO = 0: Zero data loss (synchronous replication)                   │
│                                                                        │
│  Atlas Capabilities:                                                   │
│  • Hourly snapshots: RPO = 1 hour                                      │
│  • PITR (72h window): RPO = ~1 minute                                  │
│  • Multi-region replica: RPO = ~0 seconds                              │
│                                                                        │
│  RTO (Recovery Time Objective)                                         │
│  ───────────────────────────────────                                   │
│  • Maximum acceptable downtime                                         │
│  • Measured in time: How quickly can we restore?                       │
│                                                                        │
│  Examples:                                                             │
│  • RTO = 4 hours: Service restored within 4h                           │
│  • RTO = 15 minutes: Service restored in 15min                         │
│  • RTO = 0: Instant failover (HA architecture)                         │
│                                                                        │
│  Atlas Capabilities:                                                   │
│  • Snapshot restore: RTO = 15-60 minutes                               │
│  • PITR restore: RTO = 30-90 minutes                                   │
│  • Replica failover: RTO = ~30 seconds                                 │
│  • Multi-region cluster: RTO = ~30 seconds                             │
│                                                                        │
│  ───────────────────────────────────────────────────────────────────── │
│                                                                        │
│  VISUALIZATION:                                                        │
│                                                                        │
│  Last Backup    Disaster Event    Service Restored                     │
│       │               │                   │                            │
│       ▼               ▼                   ▼                            │
│  ─────●───────────────●───────────────────●──────►  Time               │
│       │               │                   │                            │
│       └───── RPO ─────┘                   │                            │
│                       └─────── RTO ───────┘                            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Stratégies DR par Tier

```yaml
# DR Strategies Based on Business Requirements

# TIER 1 - CRITICAL (Financial, Healthcare)
tier_1:
  requirements:
    rpo: "< 1 minute"
    rto: "< 5 minutes"

  architecture:
    type: "Multi-region cluster"
    regions:
      - "US_EAST_1"  # Primary
      - "US_WEST_2"  # DR region
      - "EU_WEST_1"  # Geographic diversity

    read_preference: "primaryPreferred"
    write_concern: "majority"

  backup:
    snapshot_frequency: "hourly"
    pitr_enabled: true
    pitr_window: 168  # 7 days
    cross_region_copy: true

  cost: "$$$ High"

# TIER 2 - IMPORTANT (E-commerce, SaaS)
tier_2:
  requirements:
    rpo: "< 1 hour"
    rto: "< 30 minutes"

  architecture:
    type: "Single-region, cross-AZ"
    regions:
      - "US_EAST_1"
    availability_zones: 3

  backup:
    snapshot_frequency: "hourly"
    pitr_enabled: true
    pitr_window: 72  # 3 days
    cross_region_copy: false

  cost: "$$ Medium"

# TIER 3 - STANDARD (Internal tools)
tier_3:
  requirements:
    rpo: "< 24 hours"
    rto: "< 4 hours"

  architecture:
    type: "Single-region, cross-AZ"
    regions:
      - "US_EAST_1"
    availability_zones: 3

  backup:
    snapshot_frequency: "daily"
    pitr_enabled: false
    cross_region_copy: false

  cost: "$ Low"
```

### Plan de Disaster Recovery

```markdown
# DISASTER RECOVERY PLAN TEMPLATE

## 1. DISASTER SCENARIOS

### Scenario A: Accidental Data Deletion
- **Probability**: Medium
- **Impact**: High
- **Detection**: Application errors, user reports
- **Recovery**: PITR to timestamp before deletion

### Scenario B: Data Corruption
- **Probability**: Low
- **Impact**: Critical
- **Detection**: Data validation failures, integrity checks
- **Recovery**: PITR or snapshot restore

### Scenario C: Region Outage (AWS us-east-1)
- **Probability**: Very Low
- **Impact**: Critical
- **Detection**: Atlas alerts, monitoring systems
- **Recovery**: Failover to us-west-2 replica

### Scenario D: Cluster Configuration Error
- **Probability**: Low
- **Impact**: Medium
- **Detection**: Performance degradation, errors
- **Recovery**: Restore to new cluster from last snapshot

## 2. RESPONSE PROCEDURES

### Step 1: ASSESSMENT (5 minutes)
- [ ] Identify disaster type
- [ ] Assess scope of impact
- [ ] Determine affected data/users
- [ ] Activate incident response team

### Step 2: CONTAINMENT (10 minutes)
- [ ] Stop write operations if necessary
- [ ] Isolate affected systems
- [ ] Enable read-only mode if applicable
- [ ] Document incident details

### Step 3: RECOVERY (15-60 minutes)
- [ ] Select appropriate recovery method
- [ ] Initiate restore procedure
- [ ] Monitor restore progress
- [ ] Validate restored data

### Step 4: VERIFICATION (15 minutes)
- [ ] Run data integrity checks
- [ ] Compare row counts
- [ ] Test critical queries
- [ ] Verify indexes intact

### Step 5: RESUMPTION (10 minutes)
- [ ] Update connection strings (if new cluster)
- [ ] Enable write operations
- [ ] Monitor for errors
- [ ] Inform stakeholders

### Step 6: POST-MORTEM (24-48 hours)
- [ ] Document incident timeline
- [ ] Identify root cause
- [ ] Update procedures
- [ ] Implement preventive measures

## 3. CONTACT INFORMATION

| Role              | Name          | Phone         | Email                |
|-------------------|---------------|---------------|----------------------|
| On-Call Engineer  | John Doe      | +1-555-0001   | john@company.com     |
| Backup Engineer   | Jane Smith    | +1-555-0002   | jane@company.com     |
| Atlas Support     | MongoDB       | 24/7 Support  | support@mongodb.com  |
| Management        | CTO           | +1-555-0099   | cto@company.com      |

## 4. ESCALATION MATRIX

| Time Elapsed | Action                                              |
|--------------|-----------------------------------------------------|
| 0-15 min     | On-call engineer handles                            |
| 15-30 min    | Notify backup engineer                              |
| 30-60 min    | Engage Atlas support                                |
| 60+ min      | Escalate to management, consider external help      |

## 5. TESTING SCHEDULE

| Test Type           | Frequency    | Last Tested  | Next Test    |
|---------------------|--------------|--------------|--------------|
| Snapshot restore    | Quarterly    | 2025-09-15   | 2025-12-15   |
| PITR restore        | Bi-annual    | 2025-06-01   | 2025-12-01   |
| Full DR drill       | Annual       | 2025-03-01   | 2026-03-01   |
| Runbook review      | Quarterly    | 2025-10-01   | 2026-01-01   |
```

---

## 🧪 Testing des Backups

### Principe Fondamental

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│                   ⚠️ CRITICAL PRINCIPLE ⚠️                              │
│                                                                        │
│      "A backup you haven't tested is not a backup."                    │
│                                                                        │
│      "It's a Schrödinger's backup:                                     │
│       simultaneously working and broken                                │
│       until you try to restore it."                                    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Test Procedures

```bash
#!/bin/bash
# backup-test.sh - Automated Backup Testing

set -e

PROJECT_ID="your-project-id"
CLUSTER_NAME="production-cluster"
TEST_CLUSTER="backup-test-$(date +%Y%m%d-%H%M%S)"

echo "=== MongoDB Atlas Backup Test ==="
echo "Date: $(date)"
echo "Cluster: ${CLUSTER_NAME}"
echo ""

# Step 1: Get latest snapshot
echo "Step 1: Fetching latest snapshot..."
SNAPSHOT_ID=$(atlas backups snapshots list ${CLUSTER_NAME} \
  --projectId ${PROJECT_ID} \
  --limit 1 \
  --output json | jq -r '.[0].id')

echo "Latest snapshot: ${SNAPSHOT_ID}"
echo ""

# Step 2: Restore to new cluster
echo "Step 2: Initiating restore to test cluster..."
atlas backups restores start automated ${CLUSTER_NAME} \
  --projectId ${PROJECT_ID} \
  --snapshotId ${SNAPSHOT_ID} \
  --targetClusterName ${TEST_CLUSTER}

echo "Restore initiated. Waiting for completion..."

# Step 3: Wait for restore to complete
while true; do
  STATUS=$(atlas clusters describe ${TEST_CLUSTER} \
    --projectId ${PROJECT_ID} \
    --output json | jq -r '.stateName')

  if [ "$STATUS" = "IDLE" ]; then
    echo "Restore complete!"
    break
  fi

  echo "Status: ${STATUS}... waiting 30s"
  sleep 30
done

# Step 4: Get connection string
echo "Step 4: Getting connection string..."
CONN_STRING=$(atlas clusters connectionStrings describe ${TEST_CLUSTER} \
  --projectId ${PROJECT_ID} \
  --output json | jq -r '.standardSrv')

echo "Connection string: ${CONN_STRING}"
echo ""

# Step 5: Run validation queries
echo "Step 5: Running validation queries..."

mongosh "${CONN_STRING}" --eval "
  print('Testing database access...');
  const dbs = db.adminCommand({ listDatabases: 1 });
  print('Databases found: ' + dbs.databases.length);

  dbs.databases.forEach(dbInfo => {
    if (dbInfo.name === 'mydb') {
      db = db.getSiblingDB('mydb');
      const collections = db.getCollectionNames();
      print('Collections in mydb: ' + collections.length);

      collections.forEach(col => {
        const count = db[col].countDocuments();
        print(col + ': ' + count + ' documents');
      });
    }
  });

  print('Validation complete!');
"

# Step 6: Cleanup
echo ""
echo "Step 6: Cleanup (delete test cluster)..."
read -p "Delete test cluster ${TEST_CLUSTER}? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
  atlas clusters delete ${TEST_CLUSTER} \
    --projectId ${PROJECT_ID} \
    --force
  echo "Test cluster deleted."
fi

echo ""
echo "=== Backup Test Complete ==="
```

### Test Schedule

```yaml
# Backup Testing Schedule
testing_schedule:
  # Basic restore test
  snapshot_restore_test:
    frequency: "QUARTERLY"  # Every 3 months
    duration: "2 hours"
    procedure:
      - "Restore latest snapshot to new cluster"
      - "Verify data integrity"
      - "Run sample queries"
      - "Compare row counts with production"
      - "Delete test cluster"

  # Point-in-Time Recovery test
  pitr_test:
    frequency: "BI_ANNUAL"  # Every 6 months
    duration: "3 hours"
    procedure:
      - "Choose random timestamp in PITR window"
      - "Restore to that timestamp"
      - "Verify data consistency"
      - "Test application compatibility"
      - "Document any issues"

  # Full disaster recovery drill
  dr_drill:
    frequency: "ANNUAL"
    duration: "4-8 hours"
    procedure:
      - "Simulate region failure"
      - "Execute full DR procedure"
      - "Measure RTO and RPO"
      - "Test failover process"
      - "Validate all applications"
      - "Document lessons learned"

  # Download and manual restore test
  download_test:
    frequency: "BI_ANNUAL"
    duration: "3 hours"
    procedure:
      - "Download snapshot"
      - "Restore to local MongoDB"
      - "Verify data integrity"
      - "Test mongorestore process"
```

### Validation Checklist

```
┌────────────────────────────────────────────────────────────────────────┐
│                   BACKUP VALIDATION CHECKLIST                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  DATABASE STRUCTURE                                                    │
│  ☐ All databases present                                               │
│  ☐ All collections present                                             │
│  ☐ Collection counts match production                                  │
│  ☐ Document counts per collection match                                │
│                                                                        │
│  INDEXES                                                               │
│  ☐ All indexes restored                                                │
│  ☐ Index definitions match production                                  │
│  ☐ Unique indexes enforced                                             │
│  ☐ TTL indexes functional                                              │
│                                                                        │
│  DATA INTEGRITY                                                        │
│  ☐ Sample queries return expected results                              │
│  ☐ Aggregation pipelines work correctly                                │
│  ☐ Foreign key relationships intact (if using $lookup)                 │
│  ☐ No corrupted documents                                              │
│                                                                        │
│  CONFIGURATION                                                         │
│  ☐ User accounts restored (if applicable)                              │
│  ☐ Roles and permissions correct                                       │
│  ☐ Replica set configuration valid                                     │
│  ☐ Sharding configuration (if applicable)                              │
│                                                                        │
│  APPLICATION COMPATIBILITY                                             │
│  ☐ Application can connect                                             │
│  ☐ Read operations successful                                          │
│  ☐ Write operations successful                                         │
│  ☐ No schema version mismatches                                        │
│                                                                        │
│  PERFORMANCE                                                           │
│  ☐ Query performance acceptable                                        │
│  ☐ No obvious performance degradation                                  │
│  ☐ Indexes being used effectively                                      │
│                                                                        │
│  DOCUMENTATION                                                         │
│  ☐ Test results documented                                             │
│  ☐ Any issues noted                                                    │
│  ☐ RTO/RPO metrics recorded                                            │
│  ☐ Next test scheduled                                                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Best Practices

### Configuration Best Practices

```yaml
# Recommended Backup Configuration
best_practices:
  # 1. Enable PITR for all production clusters
  pitr:
    enabled: true
    window: 168  # 7 days for compliance

  # 2. Configure appropriate retention
  retention:
    hourly: 48    # 2 days
    daily: 14     # 2 weeks
    weekly: 8     # 2 months
    monthly: 24   # 2 years (compliance)

  # 3. Enable cross-region copy for DR
  cross_region:
    enabled: true
    target_region: "DR_REGION"
    copy_oplogs: true

  # 4. Automate backup exports for compliance
  auto_export:
    enabled: true
    target: "S3_BUCKET"
    frequency: "MONTHLY"

  # 5. Test backups regularly
  testing:
    frequency: "QUARTERLY"
    automated: true

  # 6. Monitor backup health
  monitoring:
    - alert_on_backup_failure
    - alert_on_pitr_gap
    - alert_on_low_oplog_window
```

### Compliance Considerations

```
┌────────────────────────────────────────────────────────────────────────┐
│                     COMPLIANCE REQUIREMENTS                            │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  REGULATION        REQUIREMENTS                                        │
│  ───────────────────────────────────────────────────────────────────── │
│  GDPR              • Data deletion on request                          │
│  (Europe)          • Backup encryption                                 │
│                    • Geographic data residency                         │
│                    • Retention limits (typically 7 years max)          │
│                                                                        │
│  HIPAA             • Encryption at rest and in transit                 │
│  (Healthcare)      • Access logging                                    │
│                    • BAA with MongoDB                                  │
│                    • 6-year retention minimum                          │
│                                                                        │
│  PCI-DSS           • 90-day backup retention minimum                   │
│  (Payments)        • Quarterly restore testing                         │
│                    • Encryption                                        │
│                    • Secure backup storage                             │
│                                                                        │
│  SOX               • 7-year retention for financial records            │
│  (Finance)         • Immutable backups                                 │
│                    • Audit trail                                       │
│                    • Tested DR procedures                              │
│                                                                        │
│  IMPLEMENTATION IN ATLAS:                                              │
│  ✅ Encryption: Enabled by default (AES-256)                           │
│  ✅ Geographic control: Choose backup region                           │
│  ✅ Retention: Configure per policy (up to custom)                     │
│  ✅ Audit: Atlas audit logs available                                  │
│  ✅ BAA: Available for HIPAA customers                                 │
│  ✅ Immutability: Backups cannot be modified                           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Cost Optimization

```yaml
# Backup Cost Optimization Strategies
cost_optimization:
  # 1. Right-size retention
  strategy_1:
    description: "Don't over-retain if not needed"
    example:
      before:
        monthly_retention: 24  # 2 years
        cost: "$2,400/year"
      after:
        monthly_retention: 12  # 1 year
        cost: "$1,200/year"
      savings: "50%"

  # 2. Disable PITR for non-critical environments
  strategy_2:
    description: "PITR adds ~30% to backup costs"
    example:
      dev_cluster:
        pitr: false
        savings: "$500/year"
      staging_cluster:
        pitr: false
        savings: "$300/year"

  # 3. Optimize snapshot frequency
  strategy_3:
    description: "Hourly may be overkill for some workloads"
    example:
      before:
        hourly_snapshots: true
        daily_snapshots: true
        storage: "500 GB"
        cost: "$100/month"
      after:
        hourly_snapshots: false
        daily_snapshots: true
        storage: "250 GB"
        cost: "$50/month"

  # 4. Use lifecycle policies
  strategy_4:
    description: "Archive old backups to cheaper storage"
    example:
      auto_export:
        enabled: true
        frequency: "MONTHLY"
        target: "S3_GLACIER"
        savings: "70% on long-term retention"
```

---

## 🏁 Résumé

### Points Clés

1. **Architecture Backups**
   - Snapshots automatiques (hourly, daily, weekly, monthly)
   - PITR avec fenêtre de 72h (extensible à 7 jours)
   - Cross-region copy pour DR
   - Encryption AES-256 par défaut

2. **Types de Restauration**
   - Nouveau cluster (recommandé, zéro risque)
   - Remplacement en place (downtime requis)
   - Download manuel (forensics, migration)
   - Query snapshot (récupération sélective)

3. **Disaster Recovery**
   - Définir RPO et RTO objectives
   - Multi-region pour RTO < 5 min
   - PITR pour RPO < 1 min
   - Plan DR documenté et testé

4. **Testing Impératif**
   - Tester backups au moins trimestriellement
   - DR drill annuel obligatoire
   - Mesurer RTO/RPO réels
   - Documenter procédures

5. **Compliance**
   - GDPR: Geographic residency
   - HIPAA: Encryption + BAA
   - PCI-DSS: 90 jours minimum
   - SOX: 7 ans retention

### Configuration Minimale Production

```hcl
resource "mongodbatlas_cloud_backup_schedule" "production" {
  project_id   = var.project_id
  cluster_name = "production"

  # Hourly for 2 days
  policy_item_hourly {
    frequency_interval = 1
    retention_unit     = "days"
    retention_value    = 2
  }

  # Daily for 14 days
  policy_item_daily {
    frequency_interval = 1
    retention_unit     = "days"
    retention_value    = 14
  }

  # Weekly for 8 weeks
  policy_item_weekly {
    frequency_interval = 1
    retention_unit     = "weeks"
    retention_value    = 8
  }

  # Monthly for 12 months
  policy_item_monthly {
    frequency_interval = 1
    retention_unit     = "months"
    retention_value    = 12
  }

  # PITR 7 days
  restore_window_days = 7

  # Cross-region copy
  copy_settings {
    region_name        = "US_WEST_2"
    should_copy_oplogs = true
  }
}
```

### Checklist Production

```
☐ Backups automatiques configurés
☐ PITR activé (7 jours minimum)
☐ Cross-region copy activé
☐ Retention policy définie selon compliance
☐ Alertes backup configurées
☐ Test de restauration documenté
☐ DR plan rédigé et partagé
☐ Test trimestriel planifié
☐ DR drill annuel planifié
☐ Équipe formée aux procédures
```

---


⏭️ [Scaling (vertical et horizontal)](/14-mongodb-atlas/08-scaling.md)
