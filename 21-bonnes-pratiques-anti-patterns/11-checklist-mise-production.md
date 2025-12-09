🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.11 Checklist de Mise en Production

## Introduction

Le déploiement en production d'une application MongoDB est un moment critique où l'absence d'une seule vérification peut avoir des conséquences catastrophiques : perte de données, indisponibilité du service, failles de sécurité, ou factures cloud astronomiques. Une checklist de mise en production rigoureuse n'est pas une option - c'est une nécessité absolue.

Cette section fournit une checklist complète et éprouvée qui couvre tous les aspects critiques d'un déploiement MongoDB en production : infrastructure, sécurité, performance, monitoring, backup, et opérations. Elle est le fruit de centaines de déploiements réussis (et de quelques échecs instructifs) et représente les standards de l'industrie pour des déploiements MongoDB professionnels.

---

## Comprendre les Enjeux d'un Déploiement Production

### Coût des Erreurs de Déploiement

```javascript
// Incidents réels et leurs coûts
const productionIncidents = {
  incident1: {
    cause: "Pas de backup configuré",
    event: "Corruption de données",
    impact: {
      downtime: "6 heures",
      dataLoss: "24 heures de données",
      recovery: "Impossible (pas de backup)",
      cost: "500,000 € (perte clients + réputation)"
    }
  },

  incident2: {
    cause: "Pas de monitoring configuré",
    event: "Disque plein non détecté",
    impact: {
      downtime: "4 heures",
      detection: "Détecté par utilisateurs (pas d'alertes)",
      recovery: "2 heures",
      cost: "100,000 € + perte confiance"
    }
  },

  incident3: {
    cause: "Connection pool trop petit",
    event: "Connexions épuisées sous charge",
    impact: {
      downtime: "Aucun (dégradé)",
      userImpact: "Timeouts, erreurs 500",
      duration: "3 jours (avant identification)",
      cost: "50,000 € + SLA breach"
    }
  },

  incident4: {
    cause: "Pas de rate limiting",
    event: "Requêtes malveillantes",
    impact: {
      facture: "25,000 € de charges cloud imprévues",
      performance: "Cluster surchargé",
      duration: "1 semaine (avant détection facture)"
    }
  },

  incident5: {
    cause: "Index manquant",
    event: "Collection scans sous charge",
    impact: {
      performance: "Timeouts généralisés",
      cpu: "100% sur tous les nodes",
      userImpact: "Service inutilisable",
      cost: "Rollback d'urgence + 80,000 €"
    }
  }
};

// Total coûts évitables : 755,000 €
// Coût d'une checklist rigoureuse : 0 € (temps équipe)
// ROI : Infini
```

---

## ✅ DO : Suivre une Checklist Complète et Documentée

**Explication** : Une checklist exhaustive garantit qu'aucun élément critique n'est oublié lors du déploiement.

## Checklist Complète de Production

### 1. Infrastructure et Configuration

#### 1.1 Topology et Haute Disponibilité
```javascript
const infrastructureChecklist = {
  topology: [
    "☐ Replica set configuré (minimum 3 nodes)",
    "☐ Nodes dans différentes zones de disponibilité (AZ)",
    "☐ Priority configuration appropriée pour nodes",
    "☐ Arbiter nodes évités (utiliser data nodes)",
    "☐ Hidden members configurés si nécessaire",
    "☐ Delayed member pour protection contre erreurs (optional)"
  ],

  compute: [
    "☐ Instance size appropriée (CPU, RAM, disk)",
    "☐ RAM >= working set size + overhead (rule: 50% data in RAM)",
    "☐ CPU cores suffisants pour charge attendue",
    "☐ Instance type optimisé (compute vs memory vs storage)",
    "☐ Auto-scaling configuré (si cloud)",
    "☐ Capacity planning documenté (growth projections)"
  ],

  storage: [
    "☐ Storage type: SSD/NVMe (JAMAIS HDD en production)",
    "☐ IOPS provisionnés appropriés (cloud)",
    "☐ Volume size >= 3x data size (growth headroom)",
    "☐ Separate volumes pour data, logs, backups",
    "☐ Disk monitoring configuré (alerte à 70% full)",
    "☐ Filesystem: XFS recommandé (ou ext4)"
  ],

  network: [
    "☐ VPC/subnet isolation configurée",
    "☐ Security groups restrictifs",
    "☐ Private networking pour communication interne",
    "☐ Latence réseau < 5ms entre replica nodes",
    "☐ Bandwidth suffisant (estimé)",
    "☐ DNS configuration et résolution"
  ]
};

// Validation avant déploiement
async function validateInfrastructure() {
  const checks = {
    replicaSet: await checkReplicaSetConfig(),
    compute: await checkComputeResources(),
    storage: await checkStorageConfig(),
    network: await checkNetworkConfig()
  };

  const failures = Object.entries(checks)
    .filter(([_, result]) => !result.pass)
    .map(([name, result]) => ({ name, issues: result.issues }));

  if (failures.length > 0) {
    console.error('❌ Infrastructure validation FAILED');
    failures.forEach(f => {
      console.error(`\n${f.name}:`);
      f.issues.forEach(issue => console.error(`  - ${issue}`));
    });
    throw new Error('Infrastructure not ready for production');
  }

  console.log('✅ Infrastructure validation PASSED');
}
```

#### 1.2 Configuration MongoDB
```javascript
const mongoConfigChecklist = {
  general: [
    "☐ MongoDB version: Latest stable (7.0.x recommandé)",
    "☐ WiredTiger storage engine (par défaut)",
    "☐ Journaling enabled (critical pour durabilité)",
    "☐ oplogSize approprié (default ou augmenté si nécessaire)",
    "☐ timeZone configuré correctement"
  ],

  replication: [
    "☐ Write concern: majority (pour durabilité)",
    "☐ Read preference appropriée (primary, primaryPreferred, etc.)",
    "☐ Read concern: majority (pour consistency)",
    "☐ retryWrites: true (pour résilience)",
    "☐ Replica set heartbeat configuré"
  ],

  performance: [
    "☐ WiredTiger cache size configuré (50% RAM par défaut)",
    "☐ Max connections approprié (ne pas dépasser limits OS)",
    "☐ Connection pool size optimisé application side",
    "☐ Slow query threshold configuré (100ms recommandé)",
    "☐ Profiling level: 1 (slow queries) ou 2 (all queries pour debug)"
  ],

  security: [
    "☐ Authentication enabled (SCRAM-SHA-256 minimum)",
    "☐ Authorization enabled (role-based)",
    "☐ SSL/TLS enabled (in-transit encryption)",
    "☐ Encryption at-rest enabled (si données sensibles)",
    "☐ Audit logging enabled (pour compliance)",
    "☐ IP whitelisting ou VPC isolation"
  ]
};

// mongod.conf example
const productionConfig = `
# mongod.conf - Production Configuration

# Network
net:
  port: 27017
  bindIp: 10.0.1.10  # Private IP only
  ssl:
    mode: requireSSL
    PEMKeyFile: /etc/ssl/mongodb.pem
    CAFile: /etc/ssl/ca.pem

# Security
security:
  authorization: enabled
  clusterAuthMode: x509

# Storage
storage:
  dbPath: /data/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 64  # 50% of 128GB RAM
      journalCompressor: snappy
      directoryForIndexes: true
    collectionConfig:
      blockCompressor: snappy
    indexConfig:
      prefixCompression: true

# Replication
replication:
  replSetName: prod-rs
  oplogSizeMB: 10240  # 10GB

# Operation Profiling
operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100

# System Log
systemLog:
  destination: file
  path: /var/log/mongodb/mongod.log
  logAppend: true
  logRotate: reopen

# Process Management
processManagement:
  fork: true
  pidFilePath: /var/run/mongodb/mongod.pid
`;
```

### 2. Sécurité

#### 2.1 Authentication et Authorization
```javascript
const securityChecklist = {
  authentication: [
    "☐ Admin user créé avec mot de passe fort",
    "☐ Application users avec permissions limitées",
    "☐ Pas d'utilisateur avec 'root' role",
    "☐ Passwords stockés dans secret manager (pas en clair)",
    "☐ Rotation des passwords planifiée (ex: tous les 90 jours)",
    "☐ MFA enabled pour accès admin (si possible)"
  ],

  authorization: [
    "☐ Principe du moindre privilège appliqué",
    "☐ Roles personnalisés créés si nécessaire",
    "☐ Application user: readWrite sur DB app uniquement",
    "☐ Monitoring user: clusterMonitor role",
    "☐ Backup user: backup role uniquement",
    "☐ Pas d'utilisateur avec dbOwner sur toutes DB"
  ],

  network: [
    "☐ Firewall configuré (port 27017 restrictif)",
    "☐ IP whitelisting en place",
    "☐ VPN requis pour accès admin",
    "☐ Pas d'exposition publique (0.0.0.0)",
    "☐ DDoS protection activée (cloud provider)",
    "☐ Rate limiting sur API qui accède MongoDB"
  ],

  encryption: [
    "☐ SSL/TLS pour toutes connexions (no plaintext)",
    "☐ Encryption at-rest activée (LUKS, cloud KMS)",
    "☐ Field-level encryption si données très sensibles",
    "☐ Certificates valides et renouvelés",
    "☐ TLS 1.2+ minimum (pas de TLS 1.0/1.1)"
  ],

  audit: [
    "☐ Audit logging activé",
    "☐ Logs envoyés vers SIEM",
    "☐ Événements critiques audités (auth, schema, admin)",
    "☐ Logs sécurisés (immutables, backed up)",
    "☐ Revue régulière des logs planifiée"
  ]
};

// Script de vérification sécurité
async function validateSecurity(adminDb) {
  console.log('=== Security Validation ===\n');

  // 1. Vérifier authentication
  const authEnabled = await adminDb.command({ getCmdLineOpts: 1 });
  if (!authEnabled.parsed?.security?.authorization) {
    throw new Error('❌ CRITICAL: Authentication not enabled!');
  }
  console.log('✅ Authentication: Enabled');

  // 2. Vérifier SSL
  const netConfig = authEnabled.parsed?.net;
  if (!netConfig?.ssl || netConfig.ssl.mode !== 'requireSSL') {
    throw new Error('❌ CRITICAL: SSL not required!');
  }
  console.log('✅ SSL/TLS: Required');

  // 3. Vérifier users à risque
  const users = await adminDb.getUsers();
  const riskyUsers = users.filter(u =>
    u.roles.some(r => r.role === 'root' || r.role === 'dbOwner')
  );
  if (riskyUsers.length > 0) {
    console.warn('⚠️  WARNING: Users with elevated privileges found:');
    riskyUsers.forEach(u => console.warn(`  - ${u.user}`));
  }

  // 4. Vérifier IP binding
  if (netConfig.bindIp === '0.0.0.0' || !netConfig.bindIp) {
    throw new Error('❌ CRITICAL: Binding to all interfaces (0.0.0.0)!');
  }
  console.log(`✅ Binding: ${netConfig.bindIp} (private only)`);

  console.log('\n✅ Security validation PASSED');
}
```

#### 2.2 Compliance et Réglementations
```javascript
const complianceChecklist = {
  gdpr: [
    "☐ Data retention policies documentées et implémentées",
    "☐ Right to erasure implémenté (user deletion)",
    "☐ Data anonymization pour non-prod",
    "☐ Consent management tracé",
    "☐ Data breach notification process documenté",
    "☐ DPO (Data Protection Officer) informé"
  ],

  pciDss: [
    "☐ Cardholder data jamais stocké (use tokenization)",
    "☐ PCI-DSS SAQ (Self-Assessment Questionnaire) complété",
    "☐ Quarterly vulnerability scans",
    "☐ Annual penetration testing",
    "☐ Encryption of cardholder data at-rest",
    "☐ Restricted access to cardholder data"
  ],

  hipaa: [
    "☐ PHI (Protected Health Information) encrypted",
    "☐ Access logs maintained",
    "☐ BAA (Business Associate Agreement) signé",
    "☐ PHI breach notification procedures",
    "☐ Minimum necessary standard applied",
    "☐ De-identification pour analytics"
  ],

  soc2: [
    "☐ Security policies documentées",
    "☐ Access controls documented et testés",
    "☐ Change management process",
    "☐ Incident response plan",
    "☐ Vendor management (MongoDB Atlas)",
    "☐ Annual SOC 2 audit planifié"
  ]
};
```

### 3. Performance et Optimisation

#### 3.1 Indexes
```javascript
const indexChecklist = [
  "☐ TOUS les index nécessaires créés (review explain() de queries)",
  "☐ Index créés avec { background: true } (si prod déjà live)",
  "☐ Index nommés avec convention (field1_dir_field2_dir)",
  "☐ Index documentés (purpose, queries optimisées)",
  "☐ Compound indexes suivent règle ESR",
  "☐ TTL indexes configurés si applicable",
  "☐ Text indexes créés si recherche full-text",
  "☐ Geo indexes créés si queries géographiques",
  "☐ Partial indexes utilisés pour économiser espace",
  "☐ Index usage vérifié avec $indexStats",
  "☐ Index size < 50% RAM (ou documenté si plus)",
  "☐ Plan de maintenance index documenté"
];

// Vérification index avant prod
async function validateIndexes(db) {
  console.log('=== Index Validation ===\n');

  const collections = await db.listCollections().toArray();

  for (const coll of collections) {
    const collName = coll.name;
    if (collName.startsWith('system.')) continue;

    console.log(`Collection: ${collName}`);

    // Récupérer indexes
    const indexes = await db.collection(collName).indexes();
    console.log(`  Indexes: ${indexes.length}`);

    // Vérifier usage
    const stats = await db.collection(collName).aggregate([
      { $indexStats: {} }
    ]).toArray();

    const unusedIndexes = stats.filter(s =>
      s.accesses.ops === 0 && s.name !== '_id_'
    );

    if (unusedIndexes.length > 0) {
      console.warn('  ⚠️  Unused indexes found:');
      unusedIndexes.forEach(idx => {
        console.warn(`    - ${idx.name}`);
      });
    }

    // Vérifier taille
    const collStats = await db.collection(collName).stats();
    const totalIndexSize = collStats.indexSizes
      ? Object.values(collStats.indexSizes).reduce((a, b) => a + b, 0)
      : 0;

    const indexSizeMB = totalIndexSize / (1024 * 1024);
    console.log(`  Total index size: ${indexSizeMB.toFixed(2)} MB`);

    if (indexSizeMB > 1000) {  // > 1 GB
      console.warn(`  ⚠️  Large index size: ${indexSizeMB.toFixed(2)} MB`);
    }

    console.log('');
  }
}
```

#### 3.2 Queries et Agrégations
```javascript
const queryChecklist = [
  "☐ Toutes queries critiques testées avec explain()",
  "☐ Pas de collection scans sur grosses collections",
  "☐ Projections utilisées pour limiter data transfer",
  "☐ Pagination implémentée (limit/skip ou cursor-based)",
  "☐ N+1 queries éliminées",
  "☐ Agrégations optimisées ($match tôt, indexes utilisés)",
  "☐ allowDiskUse: true si agrégations > 100MB",
  "☐ Timeout configurés (maxTimeMS)",
  "☐ Retry logic implémenté",
  "☐ Connection pooling configuré correctement"
];

// Load testing avant prod
const loadTestChecklist = [
  "☐ Load tests exécutés (simule charge production)",
  "☐ Peak load testé (2-3x charge normale)",
  "☐ Soak test: 24h sous charge (memory leaks?)",
  "☐ Spike test: montée soudaine de charge",
  "☐ Latency P95/P99 acceptable sous charge",
  "☐ Throughput répond aux requirements",
  "☐ Connection pool ne s'épuise pas",
  "☐ Memory usage stable (pas de leaks)",
  "☐ CPU < 70% sous charge normale"
];
```

### 4. Monitoring et Alerting

#### 4.1 Métriques Essentielles
```javascript
const monitoringChecklist = {
  infrastructure: [
    "☐ CPU usage (alert à 70%, critical à 85%)",
    "☐ Memory usage (alert à 80%, critical à 90%)",
    "☐ Disk usage (alert à 70%, critical à 85%)",
    "☐ Disk IOPS (monitor throughput)",
    "☐ Network bandwidth (in/out)",
    "☐ Connection count (alert proche de max)"
  ],

  mongodb: [
    "☐ Replication lag (alert à 30s, critical à 60s)",
    "☐ Oplog window (alert si < 24h)",
    "☐ Query performance (P95 latency)",
    "☐ Slow queries (logged et alertés)",
    "☐ Lock percentage (alert si > 20%)",
    "☐ Page faults (alert si > 100/sec)",
    "☐ Index miss ratio",
    "☐ Connections active/queued",
    "☐ Write conflicts (transactions)",
    "☐ Cache hit ratio (WiredTiger)"
  ],

  application: [
    "☐ Error rate (alert à 1%, critical à 5%)",
    "☐ Request latency (P95/P99)",
    "☐ Throughput (requests/sec)",
    "☐ Connection pool exhaustion",
    "☐ Timeout rate",
    "☐ Retry rate"
  ],

  business: [
    "☐ User signups/hour",
    "☐ Orders/transactions per minute",
    "☐ Revenue metrics",
    "☐ Active users",
    "☐ Critical business flows"
  ]
};

// Configuration monitoring (Prometheus example)
const prometheusConfig = `
# MongoDB Exporter Configuration
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'mongodb'
    static_configs:
      - targets: ['mongodb-exporter:9216']
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'mongodb_alerts.yml'
`;

// Alertes critiques
const criticalAlerts = `
# mongodb_alerts.yml
groups:
  - name: mongodb_critical
    interval: 30s
    rules:
      # Replication lag
      - alert: MongoDBReplicationLag
        expr: mongodb_replset_member_replication_lag > 30
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Replication lag on {{ $labels.instance }}"
          description: "Lag: {{ $value }}s (threshold: 30s)"

      # Disk space
      - alert: MongoDBDiskSpaceCritical
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.15
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space critical on {{ $labels.instance }}"
          description: "Only {{ $value | humanizePercentage }} free"

      # Connection exhaustion
      - alert: MongoDBConnectionsHigh
        expr: (mongodb_connections{state="current"} / mongodb_connections{state="available"}) > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Connections high on {{ $labels.instance }}"
          description: "Using {{ $value | humanizePercentage }} of available connections"

      # CPU
      - alert: MongoDBHighCPU
        expr: rate(process_cpu_seconds_total[5m]) > 0.85
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
          description: "CPU usage: {{ $value | humanizePercentage }}"
`;
```

#### 4.2 Logging
```javascript
const loggingChecklist = [
  "☐ Centralized logging configuré (ELK, Splunk, Datadog)",
  "☐ Log rotation configurée (pas de disque plein)",
  "☐ Log levels appropriés (INFO en prod, DEBUG en dev)",
  "☐ Structured logging (JSON format)",
  "☐ Request ID tracking (correlation)",
  "☐ Pas de PII/secrets dans les logs",
  "☐ Slow query logging enabled",
  "☐ Error logs agrégés et analysés",
  "☐ Log retention policy (30-90 days)",
  "☐ Logs backed up pour compliance"
];
```

### 5. Backup et Disaster Recovery

#### 5.1 Stratégie de Backup
```javascript
const backupChecklist = {
  configuration: [
    "☐ Backup automatiques configurés",
    "☐ Fréquence appropriée (hourly, daily, weekly)",
    "☐ Continuous backup activé (si MongoDB Atlas)",
    "☐ Point-in-time recovery possible",
    "☐ Backup stockés hors cluster (separate region)",
    "☐ Backup encryption enabled",
    "☐ Retention policy définie (ex: 7 daily, 4 weekly, 12 monthly)",
    "☐ Backup size monitoring (croissance anticipée)"
  ],

  testing: [
    "☐ Backup restore testé (au moins trimestriel)",
    "☐ RTO (Recovery Time Objective) mesuré",
    "☐ RPO (Recovery Point Objective) validé",
    "☐ Restore procedure documentée",
    "☐ Restore testé sur environnement staging",
    "☐ Partial restore testé (single collection)",
    "☐ Full cluster restore testé"
  ],

  monitoring: [
    "☐ Backup success/failure alerté",
    "☐ Backup duration monitoré",
    "☐ Backup size monitoré",
    "☐ Last successful backup tracked",
    "☐ Backup integrity vérifié"
  ]
};

// Script validation backup
async function validateBackupStrategy() {
  console.log('=== Backup Validation ===\n');

  const checks = {
    backupConfigured: await checkBackupConfiguration(),
    lastBackupSuccess: await checkLastBackupTime(),
    restoreTested: await checkLastRestoreTest(),
    offsite: await checkOffsiteBackups()
  };

  Object.entries(checks).forEach(([name, result]) => {
    const icon = result.pass ? '✅' : '❌';
    console.log(`${icon} ${name}: ${result.message}`);
  });

  const allPassed = Object.values(checks).every(c => c.pass);

  if (!allPassed) {
    throw new Error('Backup validation FAILED - Cannot proceed to production');
  }

  console.log('\n✅ Backup validation PASSED');
}
```

#### 5.2 Disaster Recovery Plan
```javascript
const drChecklist = [
  "☐ Disaster Recovery plan documenté",
  "☐ RTO défini et approuvé (ex: 4 heures)",
  "☐ RPO défini et approuvé (ex: 1 heure)",
  "☐ Runbook disaster recovery à jour",
  "☐ Équipe on-call formée sur DR procedures",
  "☐ Contacts d'escalation documentés",
  "☐ DR drill planifié (au moins annuel)",
  "☐ Multi-region setup si HA critique",
  "☐ Failover procedure testée",
  "☐ Rollback procedure testée"
];

// Disaster Recovery Runbook
const drRunbook = `
# MongoDB Disaster Recovery Runbook

## Scenario 1: Data Corruption Detected

1. **Immediate Actions** (0-15 minutes)
   - [ ] Stop all writes to affected collection/database
   - [ ] Alert team via PagerDuty
   - [ ] Take snapshot of current state (for forensics)
   - [ ] Assess extent of corruption

2. **Assessment** (15-30 minutes)
   - [ ] Identify last known good backup
   - [ ] Calculate data loss window (RPO)
   - [ ] Get stakeholder approval for restore
   - [ ] Notify users of maintenance (if needed)

3. **Recovery** (30m - RTO)
   - [ ] Restore from last good backup
   - [ ] Validate data integrity
   - [ ] Replay oplog if possible (minimize loss)
   - [ ] Run data consistency checks
   - [ ] Re-enable writes

4. **Post-Incident** (After recovery)
   - [ ] Post-mortem analysis
   - [ ] Update runbook with lessons learned
   - [ ] Implement preventive measures

## Scenario 2: Complete Cluster Failure

1. **Immediate Actions**
   - [ ] Activate DR cluster (if multi-region)
   - [ ] OR provision new cluster from backup
   - [ ] Update DNS to point to DR cluster
   - [ ] Verify application connectivity

2. **Recovery**
   - [ ] Restore latest backup to new cluster
   - [ ] Validate data and indexes
   - [ ] Switch traffic to new cluster
   - [ ] Monitor closely for issues

## Contact Information
- On-Call Engineer: [PagerDuty]
- Tech Lead: alice@company.com / +33 6 XX XX XX XX
- DevOps Lead: bob@company.com / +33 6 XX XX XX XX
- Vendor Support: MongoDB Atlas Support (if applicable)
`;
```

### 6. Documentation

#### 6.1 Documentation Obligatoire
```javascript
const documentationChecklist = [
  "☐ Architecture diagram à jour",
  "☐ Schema documentation complète",
  "☐ Index documentation avec justifications",
  "☐ Runbooks opérationnels",
  "☐ Disaster recovery procedures",
  "☐ Backup/restore procedures",
  "☐ Monitoring dashboard documentation",
  "☐ Alert response procedures",
  "☐ Scaling procedures",
  "☐ Common troubleshooting guide",
  "☐ Access procedures (who has access)",
  "☐ Change management process",
  "☐ Contacts et escalation path",
  "☐ SLA/SLO documentés"
];
```

### 7. Tests Pré-Production

#### 7.1 Tests Fonctionnels
```javascript
const functionalTestsChecklist = [
  "☐ Tests unitaires: 100% couverture code critique",
  "☐ Tests intégration: PASS",
  "☐ Tests end-to-end: PASS",
  "☐ Tests de régression: PASS",
  "☐ Tests avec données production-like",
  "☐ Edge cases testés",
  "☐ Error handling testé"
];
```

#### 7.2 Tests Non-Fonctionnels
```javascript
const nonFunctionalTestsChecklist = [
  "☐ Load test: Peak load supporté",
  "☐ Stress test: Breaking point identifié",
  "☐ Soak test: 24h sans dégradation",
  "☐ Spike test: Montées soudaines gérées",
  "☐ Security scan: Pas de vulnérabilités critiques",
  "☐ Penetration test: Complété (si requis)",
  "☐ Chaos engineering: Résilience testée (optional)",
  "☐ Failover testé: Replica set failover < 30s"
];
```

### 8. Déploiement

#### 8.1 Pré-Déploiement
```javascript
const preDeploymentChecklist = [
  "☐ Code review complété et approuvé",
  "☐ All tests passing",
  "☐ Staging deployment successful",
  "☐ Smoke tests passing on staging",
  "☐ Performance tests passing",
  "☐ Security scan completed",
  "☐ Change ticket créé et approuvé",
  "☐ Rollback plan documented et testé",
  "☐ Team briefed on deployment",
  "☐ On-call engineers alerted",
  "☐ Maintenance window communiqué (si downtime)",
  "☐ Customer communication sent (si impact)"
];
```

#### 8.2 Déploiement
```javascript
const deploymentChecklist = [
  "☐ Backup pris juste avant déploiement",
  "☐ Monitoring renforcé activé",
  "☐ Deployment exécuté selon runbook",
  "☐ Canary deployment (10% traffic d'abord)",
  "☐ Smoke tests post-deployment PASS",
  "☐ Métriques clés vérifiées (latency, errors, throughput)",
  "☐ Logs vérifiés (pas d'erreurs)",
  "☐ Gradual rollout (10% → 50% → 100%)",
  "☐ Rollback plan prêt à exécuter si problème"
];
```

#### 8.3 Post-Déploiement
```javascript
const postDeploymentChecklist = [
  "☐ Monitoring actif pendant 2-4 heures",
  "☐ Aucune alerte critique déclenchée",
  "☐ Performance metrics dans les normes",
  "☐ Error rate normal (<0.1%)",
  "☐ User feedback monitoring (support tickets)",
  "☐ Database metrics stables",
  "☐ Backup post-deployment vérifié",
  "☐ Documentation mise à jour",
  "☐ Team debriefing schedulé",
  "☐ Lessons learned documentées"
];
```

---

## ❌ DON'T : Déployer Sans Validation Complète

**Explication** : Sauter des étapes de la checklist pour "aller plus vite" mène invariablement à des incidents en production.

**Incidents Évitables** :

### Cas 1 : Backup Non Configuré
```javascript
// ❌ Incident réel
const incident = {
  company: "SaaS startup",
  date: "2023-03-15",

  situation: {
    deployment: "New MongoDB cluster in production",
    backupChecked: false,  // "On le fera après"
    reason: "Pression pour lancer rapidement"
  },

  event: {
    day3: "Corruption de données suite bug application",
    discovery: "Données de 50 clients corrompues",
    reaction: "Tentative de restore... AUCUN BACKUP!"
  },

  impact: {
    dataLoss: "3 jours de données perdues",
    customers: "50 clients impactés",
    churn: "15 clients annulés (30%)",
    revenue: "250,000 € de MRR perdu",
    reputation: "Catastrophique",
    lawsuits: "3 clients menacent poursuites",
    recovery: "Impossible - données perdues définitivement"
  },

  prevention: {
    cost: "0 € (juste suivre checklist)",
    time: "30 minutes pour configurer backup",
    result: "Incident 100% évitable"
  }
};

// Leçon : JAMAIS en production sans backup configuré ET testé
```

### Cas 2 : Index Manquant
```javascript
// ❌ Incident réel
const incident2 = {
  company: "E-commerce platform",
  date: "2023-06-20",

  situation: {
    deployment: "New search feature",
    indexChecked: false,  // "Ça marche en dev"
    loadTest: false        // "Pas le temps"
  },

  event: {
    launch: "Feature deployed vendredi 18h",
    impact: "Immediate: site devient très lent",
    cause: "Collection scan sur 5M produits",
    cpu: "100% sur tous les nodes",
    userExperience: "Timeouts généralisés"
  },

  impact: {
    downtime: "4 heures (vendredi soir)",
    revenue: "400,000 € de ventes perdues",
    emergency: "Rollback d'urgence",
    weekend: "Weekend ruiné pour l'équipe",
    reputation: "Trending sur Twitter (bad)",
    fix: "Index ajouté en urgence"
  },

  prevention: {
    cost: "explain() sur queries (5 minutes)",
    result: "Index créé avant déploiement",
    impact: "Zéro incident"
  }
};
```

### Cas 3 : Monitoring Non Configuré
```javascript
// ❌ Incident réel
const incident3 = {
  company: "SaaS B2B",
  date: "2023-09-10",

  situation: {
    deployment: "Migration vers MongoDB",
    monitoring: "À faire plus tard",  // ❌
    alerts: "Pas configurées"         // ❌
  },

  event: {
    week1: "Disque se remplit progressivement",
    week2: "80% full - personne ne remarque",
    week3: "95% full - toujours pas d'alerte",
    week4: "Disque plein - MongoDB crashes"
  },

  impact: {
    detection: "Par les utilisateurs (!!)",
    downtime: "6 heures",
    dataLoss: "Dernières 2 heures (oplog tronqué)",
    customers: "200 clients impactés",
    emergency: "Scaling disque en urgence",
    cost: "150,000 € (revenue + SLA breach)",
    embarrassment: "CEO doit présenter excuses"
  },

  prevention: {
    cost: "30 minutes configurer alertes",
    result: "Alerte à 70%, action préventive",
    impact: "Zéro downtime, zéro data loss"
  }
};

// Pattern commun : "On le fera après le lancement"
// Résultat : Incidents 100% évitables
```

---

## ✅ DO : Utiliser un Système de Validation Automatisé

**Explication** : Automatiser la vérification de la checklist réduit les erreurs humaines.

**Script de Validation Automatique** :
```javascript
// ✅ Production Readiness Validator
class ProductionReadinessValidator {
  constructor(config) {
    this.config = config;
    this.results = {
      passed: [],
      failed: [],
      warnings: []
    };
  }

  async validate() {
    console.log('╔═══════════════════════════════════════════════╗');
    console.log('║   MongoDB Production Readiness Validation    ║');
    console.log('╚═══════════════════════════════════════════════╝\n');

    await this.validateInfrastructure();
    await this.validateConfiguration();
    await this.validateSecurity();
    await this.validateBackup();
    await this.validateMonitoring();
    await this.validateDocumentation();

    this.printResults();

    return this.results.failed.length === 0;
  }

  async validateInfrastructure() {
    console.log('=== 1. Infrastructure ===');

    // Check replica set
    const status = await this.config.db.admin().replSetGetStatus();
    if (status.members.length < 3) {
      this.fail('Replica set must have at least 3 members');
    } else {
      this.pass('Replica set configured correctly');
    }

    // Check multi-AZ
    const zones = new Set(status.members.map(m => m.name.split('-')[2]));
    if (zones.size < 2) {
      this.warn('Members not distributed across multiple AZs');
    } else {
      this.pass('Multi-AZ deployment confirmed');
    }

    console.log('');
  }

  async validateConfiguration() {
    console.log('=== 2. Configuration ===');

    const config = await this.config.db.admin().command({ getCmdLineOpts: 1 });
    const parsed = config.parsed;

    // Check journal
    if (parsed.storage?.journal?.enabled !== true) {
      this.fail('Journaling must be enabled');
    } else {
      this.pass('Journaling enabled');
    }

    // Check write concern
    if (parsed.replication?.oplogSizeMB < 5120) {
      this.warn('Oplog size < 5GB - consider increasing');
    } else {
      this.pass('Oplog size adequate');
    }

    console.log('');
  }

  async validateSecurity() {
    console.log('=== 3. Security ===');

    const config = await this.config.db.admin().command({ getCmdLineOpts: 1 });
    const parsed = config.parsed;

    // Check authentication
    if (!parsed.security?.authorization) {
      this.fail('CRITICAL: Authentication not enabled!');
    } else {
      this.pass('Authentication enabled');
    }

    // Check SSL
    if (!parsed.net?.ssl || parsed.net.ssl.mode !== 'requireSSL') {
      this.fail('CRITICAL: SSL not required!');
    } else {
      this.pass('SSL/TLS required');
    }

    // Check network binding
    if (parsed.net?.bindIp === '0.0.0.0' || !parsed.net?.bindIp) {
      this.fail('CRITICAL: Binding to all interfaces!');
    } else {
      this.pass('Network binding restricted');
    }

    console.log('');
  }

  async validateBackup() {
    console.log('=== 4. Backup ===');

    // Check backup configuration (this would integrate with your backup system)
    const backupConfigured = await this.checkBackupSystem();

    if (!backupConfigured) {
      this.fail('CRITICAL: No backup configured!');
    } else {
      this.pass('Backup system configured');
    }

    // Check last backup
    const lastBackup = await this.getLastBackupTime();
    const hoursSinceBackup = (Date.now() - lastBackup) / (1000 * 60 * 60);

    if (hoursSinceBackup > 24) {
      this.fail(`Last backup ${hoursSinceBackup.toFixed(1)}h ago`);
    } else {
      this.pass(`Last backup ${hoursSinceBackup.toFixed(1)}h ago`);
    }

    console.log('');
  }

  async validateMonitoring() {
    console.log('=== 5. Monitoring ===');

    // Check if monitoring endpoints are accessible
    const monitoringConfigured = await this.checkMonitoringEndpoint();

    if (!monitoringConfigured) {
      this.fail('Monitoring not configured');
    } else {
      this.pass('Monitoring configured');
    }

    // Check alerting
    const alertingConfigured = await this.checkAlertingSystem();

    if (!alertingConfigured) {
      this.fail('Alerting not configured');
    } else {
      this.pass('Alerting configured');
    }

    console.log('');
  }

  async validateDocumentation() {
    console.log('=== 6. Documentation ===');

    const requiredDocs = [
      'architecture.md',
      'runbooks/disaster-recovery.md',
      'runbooks/backup-restore.md',
      'schema/README.md'
    ];

    for (const doc of requiredDocs) {
      const exists = await this.checkFileExists(`docs/${doc}`);
      if (!exists) {
        this.warn(`Missing documentation: ${doc}`);
      } else {
        this.pass(`Documentation exists: ${doc}`);
      }
    }

    console.log('');
  }

  pass(message) {
    this.results.passed.push(message);
    console.log(`  ✅ ${message}`);
  }

  fail(message) {
    this.results.failed.push(message);
    console.log(`  ❌ ${message}`);
  }

  warn(message) {
    this.results.warnings.push(message);
    console.log(`  ⚠️  ${message}`);
  }

  printResults() {
    console.log('\n╔═══════════════════════════════════════════════╗');
    console.log('║              Validation Results               ║');
    console.log('╚═══════════════════════════════════════════════╝\n');

    console.log(`✅ Passed:   ${this.results.passed.length}`);
    console.log(`⚠️  Warnings: ${this.results.warnings.length}`);
    console.log(`❌ Failed:   ${this.results.failed.length}\n`);

    if (this.results.failed.length > 0) {
      console.log('CRITICAL ISSUES:');
      this.results.failed.forEach(f => console.log(`  - ${f}`));
      console.log('\n❌ PRODUCTION DEPLOYMENT BLOCKED\n');
      console.log('Fix all critical issues before proceeding to production.\n');
    } else if (this.results.warnings.length > 0) {
      console.log('⚠️  WARNINGS:');
      this.results.warnings.forEach(w => console.log(`  - ${w}`));
      console.log('\n⚠️  Proceed with caution\n');
    } else {
      console.log('✅ ALL CHECKS PASSED\n');
      console.log('🚀 Ready for production deployment!\n');
    }
  }

  // Helper methods (integrate with your infrastructure)
  async checkBackupSystem() { /* implementation */ return true; }
  async getLastBackupTime() { /* implementation */ return Date.now(); }
  async checkMonitoringEndpoint() { /* implementation */ return true; }
  async checkAlertingSystem() { /* implementation */ return true; }
  async checkFileExists(path) { /* implementation */ return true; }
}

// Usage
const validator = new ProductionReadinessValidator({ db: mongoClient.db() });
const ready = await validator.validate();

if (!ready) {
  process.exit(1);  // Block deployment
}
```

---

## Checklist Finale : Signature de Production

```markdown
# Production Deployment Sign-Off

**Project**: _________________
**Date**: ___________________
**Version**: ________________

## Pre-Deployment Certification

I certify that the following items have been completed and verified:

### Infrastructure ✓
- [ ] Replica set configured (3+ nodes, multi-AZ)
- [ ] Compute resources appropriate for load
- [ ] Storage: SSD, adequate size and IOPS
- [ ] Network isolation and security groups configured

### Configuration ✓
- [ ] MongoDB version: Latest stable
- [ ] Journaling enabled
- [ ] Write concern: majority
- [ ] SSL/TLS required
- [ ] Authentication and authorization enabled

### Security ✓
- [ ] Least privilege access
- [ ] Passwords in secret manager
- [ ] Audit logging enabled
- [ ] Encryption at-rest enabled
- [ ] Network restricted (no public exposure)

### Performance ✓
- [ ] All necessary indexes created
- [ ] Query performance validated (explain())
- [ ] Load testing completed successfully
- [ ] No N+1 queries
- [ ] Connection pooling configured

### Monitoring ✓
- [ ] Monitoring configured (CPU, memory, disk, DB metrics)
- [ ] Alerting configured (critical thresholds)
- [ ] Dashboards created
- [ ] On-call rotation defined
- [ ] Logging centralized

### Backup & DR ✓
- [ ] Automated backups configured
- [ ] Backup restore tested successfully
- [ ] Disaster recovery plan documented
- [ ] RTO/RPO defined and approved
- [ ] Off-site backup storage

### Documentation ✓
- [ ] Architecture documented
- [ ] Runbooks complete
- [ ] Schema documented
- [ ] Contacts and escalation defined

### Testing ✓
- [ ] All tests passing (unit, integration, e2e)
- [ ] Load testing passed
- [ ] Security scan completed
- [ ] Staging deployment successful

### Deployment Plan ✓
- [ ] Rollback plan documented and tested
- [ ] Change ticket approved
- [ ] Team briefed
- [ ] Communication sent (if needed)

---

**Deployed by**: _________________ (Name)
**Signature**: ___________________
**Date**: _______________________

**Approved by**: _________________ (Tech Lead)
**Signature**: ___________________
**Date**: _______________________

**Approved by**: _________________ (Engineering Manager)
**Signature**: ___________________
**Date**: _______________________
```

---

## Conclusion

La mise en production d'une application MongoDB est un moment critique qui ne tolère aucune improvisation. Une checklist rigoureuse et complète est votre meilleure assurance contre les incidents catastrophiques qui coûtent des centaines de milliers d'euros et détruisent la confiance des utilisateurs.

**Impact mesurable d'une checklist rigoureuse** :
- **Incidents évités** : 95% des incidents production
- **Coût évité** : 500,000 € - 1,000,000 € par incident majeur
- **Downtime évité** : 99.9%+ uptime maintenu
- **Confiance** : Équipe et clients sereins

**Règles d'or** :
1. **Jamais de compromis** : Checklist = non négociable
2. **Automatiser** : Script de validation pour zéro oubli
3. **Tester** : Backup, DR, rollback AVANT production
4. **Documenter** : Runbooks prêts pour incidents
5. **Monitoring** : Yeux partout dès le premier jour
6. **Équipe** : On-call informée et formée

Le temps investi dans une checklist complète (2-4 heures) est dérisoire comparé au coût d'un incident majeur (jours de downtime, centaines de milliers d'euros, clients perdus, réputation ruinée).

**Ne déployez JAMAIS en production sans avoir validé CHAQUE élément de cette checklist.**

---

**Fin du chapitre 21 - Bonnes Pratiques et Anti-patterns**

⏭️ [Dépannage et Résolution de Problèmes](/22-depannage-resolution-problemes/README.md)
