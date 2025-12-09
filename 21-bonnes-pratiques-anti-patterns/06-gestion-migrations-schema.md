🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.6 Gestion des Migrations de Schéma

## Introduction

L'un des mythes persistants autour de MongoDB est que son modèle "schema-less" signifie l'absence de gestion de schéma. En réalité, MongoDB est plus précisément "schema-flexible" : les schémas existent, évoluent, et doivent être gérés avec rigueur. La flexibilité du schéma est un avantage majeur, mais elle devient un piège si elle n'est pas accompagnée d'une stratégie claire de migration.

Une migration de schéma mal gérée peut corrompre des données, créer des incohérences, causer des downtimes prolongés, ou rendre impossible le rollback d'un déploiement. Cette section explore les stratégies éprouvées pour faire évoluer vos schémas MongoDB de manière sûre, progressive et réversible.

---

## Comprendre les Migrations dans MongoDB

### Différence avec les Bases Relationnelles

```javascript
// SQL : Migration obligatoire et immédiate
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
// Bloque la table, migration synchrone de TOUS les enregistrements

// MongoDB : Migration flexible et progressive
// Option 1 : Lazy migration (recommandée)
// Les nouveaux docs ont le champ, les anciens sont migrés au besoin

// Option 2 : Eager migration (si nécessaire)
db.users.updateMany({}, { $set: { phone: null } });
// Mais peut être fait progressivement, sans bloquer
```

### Types de Migrations

**1. Migrations Additives (Non-destructives)**
```javascript
// ✅ Ajouter un champ optionnel
{ name: "Alice" }
→ { name: "Alice", email: null }
```

**2. Migrations Transformatives**
```javascript
// ⚠️ Modifier la structure
{ name: "Alice Smith" }
→ { firstName: "Alice", lastName: "Smith" }
```

**3. Migrations Destructives**
```javascript
// ❌ Supprimer des données
{ name: "Alice", oldField: "deprecated" }
→ { name: "Alice" }
```

---

## ✅ DO : Utiliser le Pattern de Migration Lazy (Paresseuse)

**Explication** : La migration lazy transforme les documents progressivement lors de leur accès, évitant une migration massive coûteuse.

**Pattern recommandé** :
```javascript
// ✅ Migration lazy avec versioning
// Documents peuvent coexister avec différentes versions

// Version 1 (ancien format)
{
  _id: ObjectId("..."),
  schemaVersion: 1,
  name: "Alice Smith",
  age: 30
}

// Version 2 (nouveau format)
{
  _id: ObjectId("..."),
  schemaVersion: 2,
  firstName: "Bob",
  lastName: "Johnson",
  age: 25,
  birthDate: ISODate("1999-01-15")
}

// Code applicatif qui gère les deux versions
class User {
  constructor(doc) {
    this._raw = doc;
    this.migrated = false;
  }

  // Getters qui migrent à la volée
  get firstName() {
    if (this._raw.schemaVersion === 1) {
      // Migration à la volée
      return this._raw.name.split(' ')[0];
    }
    return this._raw.firstName;
  }

  get lastName() {
    if (this._raw.schemaVersion === 1) {
      return this._raw.name.split(' ')[1] || '';
    }
    return this._raw.lastName;
  }

  // Sauvegarder avec la nouvelle version
  async save() {
    if (this._raw.schemaVersion === 1) {
      // Migrer lors de la sauvegarde
      const migrated = {
        _id: this._raw._id,
        schemaVersion: 2,
        firstName: this.firstName,
        lastName: this.lastName,
        age: this._raw.age,
        birthDate: calculateBirthDate(this._raw.age)
      };

      await db.users.replaceOne(
        { _id: this._raw._id },
        migrated
      );

      this._raw = migrated;
      this.migrated = true;
    } else {
      // Sauvegarder normalement
      await db.users.replaceOne(
        { _id: this._raw._id },
        this._raw
      );
    }
  }
}
```

**Avantages mesurables** :

### 1. Pas de Downtime
```javascript
// Migration eager : 10M documents
// Temps : 2-3 heures
// Downtime : 2-3 heures

// Migration lazy :
// Temps initial : 0 seconde (juste déployer le code)
// Downtime : 0 seconde
// Migration progressive sur plusieurs jours/semaines
```

### 2. Réversibilité
```javascript
// Si problème détecté après déploiement
// Le code peut être rollback
// Les documents anciens continuent de fonctionner
// Les documents migrés peuvent être "démigrés" si nécessaire
```

### 3. Test en Production
```javascript
// Migration progressive = test progressif
// Surveiller les erreurs sur les premiers documents migrés
// Ajuster si nécessaire avant que tous soient migrés
```

**Implémentation avec middleware** :
```javascript
// ✅ Middleware de migration automatique
async function migrateOnRead(doc) {
  if (!doc || doc.schemaVersion === CURRENT_VERSION) {
    return doc;
  }

  // Migrer selon la version
  let migrated = doc;

  if (doc.schemaVersion === 1) {
    migrated = migrateV1ToV2(migrated);
  }
  if (migrated.schemaVersion === 2) {
    migrated = migrateV2ToV3(migrated);
  }

  // Sauvegarder en arrière-plan (fire and forget)
  saveMigratedDocument(migrated).catch(err => {
    logger.error('Background migration failed', err);
  });

  return migrated;
}

// Utilisation
async function getUser(userId) {
  const doc = await db.users.findOne({ _id: userId });
  return migrateOnRead(doc);
}
```

---

## ❌ DON'T : Faire des Migrations "Big Bang"

**Explication** : Migrer tous les documents en une seule opération massive est dangereux et souvent inutile.

**Anti-pattern** :
```javascript
// ❌ Migration massive synchrone
async function migrateAllUsers() {
  console.log('Starting migration of 50M users...');

  const startTime = Date.now();

  // Migrer TOUS les documents d'un coup
  const result = await db.users.updateMany(
    { schemaVersion: 1 },
    [{
      $set: {
        schemaVersion: 2,
        firstName: { $arrayElemAt: [{ $split: ["$name", " "] }, 0] },
        lastName: { $arrayElemAt: [{ $split: ["$name", " "] }, 1] }
      }
    }]
  );

  const duration = Date.now() - startTime;
  console.log(`Migrated ${result.modifiedCount} users in ${duration}ms`);
  // Durée réelle : plusieurs heures
  // Application bloquée pendant tout ce temps
}
```

**Conséquences désastreuses** :

### 1. Downtime Prolongé
```javascript
// 50M documents à migrer
// Vitesse : ~5,000 docs/seconde
// Temps : 50,000,000 / 5,000 = 10,000 secondes = 2.8 heures

// Application inutilisable pendant 3 heures
// Coût : perte de revenus, utilisateurs frustrés
```

### 2. Charge Extrême sur la Base
```javascript
// CPU : 100% pendant toute la durée
// I/O disque : saturé
// Replica lag : peut atteindre plusieurs heures
// Autres opérations : extrêmement ralenties
```

### 3. Rollback Impossible
```javascript
// Si erreur découverte après 1 heure de migration
// 15M documents déjà migrés
// 35M documents restants

// Options :
// 1. Continuer la migration (et propager l'erreur)
// 2. Rollback (nécessite 1h+ supplémentaire)
// 3. Laisser dans un état incohérent (catastrophe)
```

### 4. Risque de Corruption
```javascript
// Si la migration plante au milieu
// État de la base : incohérent
// Certains docs migrés, d'autres non
// Difficile de savoir où reprendre
```

---

## ✅ DO : Implémenter des Migrations Progressives par Batch

**Explication** : Migrer par petits batches contrôlés permet de limiter l'impact et de pouvoir s'arrêter/reprendre à tout moment.

**Pattern recommandé** :
```javascript
// ✅ Migration progressive par batch
async function migrateInBatches() {
  const BATCH_SIZE = 1000;
  const DELAY_BETWEEN_BATCHES = 100; // ms

  let migratedCount = 0;
  let hasMore = true;

  while (hasMore) {
    // Récupérer un batch de documents à migrer
    const batch = await db.users.find({
      schemaVersion: 1
    }).limit(BATCH_SIZE).toArray();

    if (batch.length === 0) {
      hasMore = false;
      break;
    }

    // Migrer le batch
    const bulkOps = batch.map(doc => ({
      updateOne: {
        filter: { _id: doc._id, schemaVersion: 1 }, // Vérifier version
        update: {
          $set: {
            schemaVersion: 2,
            firstName: doc.name.split(' ')[0],
            lastName: doc.name.split(' ')[1] || '',
            migratedAt: new Date()
          },
          $unset: { name: "" }
        }
      }
    }));

    try {
      const result = await db.users.bulkWrite(bulkOps, { ordered: false });
      migratedCount += result.modifiedCount;

      console.log(`Migrated batch: ${result.modifiedCount} documents (total: ${migratedCount})`);

      // Métrique pour monitoring
      await recordMigrationProgress(migratedCount);

    } catch (error) {
      console.error('Batch migration failed:', error);
      // Décider : continuer ou arrêter
      if (isCriticalError(error)) {
        throw error;
      }
    }

    // Pause pour ne pas surcharger la base
    await sleep(DELAY_BETWEEN_BATCHES);

    // Vérifier si on doit arrêter (circuit breaker)
    if (await shouldStopMigration()) {
      console.log('Migration paused by circuit breaker');
      break;
    }
  }

  console.log(`Migration completed: ${migratedCount} documents migrated`);
  return migratedCount;
}

// Circuit breaker
async function shouldStopMigration() {
  const metrics = await getSystemMetrics();

  // Arrêter si :
  // - Trop d'erreurs détectées
  // - Replica lag trop important
  // - CPU trop élevé
  // - Erreur rate application en hausse

  return (
    metrics.migrationErrors > 100 ||
    metrics.replicaLag > 60 || // 60 secondes
    metrics.cpuUsage > 80 ||   // 80%
    metrics.errorRate > 0.05    // 5%
  );
}
```

**Amélioration : Migration en arrière-plan** :
```javascript
// ✅ Worker de migration asynchrone
class MigrationWorker {
  constructor() {
    this.isRunning = false;
    this.stats = {
      total: 0,
      migrated: 0,
      failed: 0,
      startTime: null
    };
  }

  async start() {
    if (this.isRunning) {
      console.log('Migration already running');
      return;
    }

    this.isRunning = true;
    this.stats.startTime = Date.now();

    console.log('Starting background migration worker...');

    while (this.isRunning) {
      try {
        // Migrer un batch
        const count = await this.migrateBatch();

        if (count === 0) {
          console.log('Migration completed!');
          this.isRunning = false;
          break;
        }

        this.stats.migrated += count;

        // Pause adaptative selon la charge système
        const delay = await this.calculateDelay();
        await sleep(delay);

      } catch (error) {
        console.error('Migration batch failed:', error);
        this.stats.failed++;

        if (this.stats.failed > 10) {
          console.error('Too many failures, stopping migration');
          this.isRunning = false;
        }

        await sleep(5000); // Pause plus longue après erreur
      }
    }
  }

  async migrateBatch() {
    // Logique de migration par batch
    const batch = await db.users.find({ schemaVersion: 1 })
      .limit(500)
      .toArray();

    // ... migration logic

    return batch.length;
  }

  async calculateDelay() {
    const metrics = await getSystemMetrics();

    // Pause plus longue si système chargé
    if (metrics.cpuUsage > 70) return 1000;
    if (metrics.cpuUsage > 50) return 500;
    return 100;
  }

  stop() {
    console.log('Stopping migration worker...');
    this.isRunning = false;
  }

  getStats() {
    const duration = Date.now() - this.stats.startTime;
    return {
      ...this.stats,
      duration: duration,
      rate: this.stats.migrated / (duration / 1000) // docs/sec
    };
  }
}

// Utilisation
const worker = new MigrationWorker();
worker.start();

// Monitoring
setInterval(() => {
  console.log('Migration stats:', worker.getStats());
}, 60000); // Toutes les minutes
```

---

## ✅ DO : Versionner les Schémas de Documents

**Explication** : Inclure un champ `schemaVersion` dans chaque document permet de gérer proprement l'évolution du schéma.

**Pattern recommandé** :
```javascript
// ✅ Schéma versionné
{
  _id: ObjectId("..."),
  schemaVersion: 2,  // Version du schéma
  // ... autres champs
}

// Définir les versions de schéma
const SCHEMA_VERSIONS = {
  1: {
    description: "Initial schema",
    fields: ["name", "email", "age"],
    introduced: "2023-01-01"
  },
  2: {
    description: "Split name into firstName/lastName",
    fields: ["firstName", "lastName", "email", "age", "birthDate"],
    introduced: "2024-01-15",
    changes: [
      "Split name field",
      "Added birthDate field"
    ]
  },
  3: {
    description: "Added user preferences",
    fields: ["firstName", "lastName", "email", "birthDate", "preferences"],
    introduced: "2024-06-01",
    changes: [
      "Removed age field (calculated from birthDate)",
      "Added preferences object"
    ]
  }
};

const CURRENT_VERSION = 3;
```

**Migrations entre versions** :
```javascript
// ✅ Fonctions de migration entre versions
const migrations = {
  1: {
    up(doc) {
      // Migration V1 → V2
      const [firstName, ...lastNameParts] = doc.name.split(' ');
      return {
        ...doc,
        schemaVersion: 2,
        firstName: firstName,
        lastName: lastNameParts.join(' '),
        birthDate: calculateBirthDate(doc.age)
      };
    },
    down(doc) {
      // Rollback V2 → V1
      return {
        ...doc,
        schemaVersion: 1,
        name: `${doc.firstName} ${doc.lastName}`,
        age: calculateAge(doc.birthDate)
      };
    }
  },

  2: {
    up(doc) {
      // Migration V2 → V3
      const age = calculateAge(doc.birthDate);
      return {
        ...doc,
        schemaVersion: 3,
        preferences: {
          language: 'en',
          timezone: 'UTC',
          notifications: true
        }
      };
      // Note : age supprimé (sera calculé à la volée)
    },
    down(doc) {
      // Rollback V3 → V2
      return {
        ...doc,
        schemaVersion: 2,
        age: calculateAge(doc.birthDate)
      };
    }
  }
};

// Migrer un document à la version courante
function migrateToLatest(doc) {
  let currentDoc = { ...doc };
  let version = doc.schemaVersion || 1;

  while (version < CURRENT_VERSION) {
    if (!migrations[version]) {
      throw new Error(`No migration found for version ${version}`);
    }

    currentDoc = migrations[version].up(currentDoc);
    version++;
  }

  return currentDoc;
}

// Rollback à une version spécifique
function rollbackToVersion(doc, targetVersion) {
  let currentDoc = { ...doc };
  let version = doc.schemaVersion || CURRENT_VERSION;

  while (version > targetVersion) {
    if (!migrations[version - 1]) {
      throw new Error(`No rollback found for version ${version}`);
    }

    currentDoc = migrations[version - 1].down(currentDoc);
    version--;
  }

  return currentDoc;
}
```

---

## ❌ DON'T : Faire des Migrations Destructives Sans Backup

**Explication** : Supprimer ou transformer des données sans possibilité de retour est extrêmement risqué.

**Anti-pattern dangereux** :
```javascript
// ❌ DANGER : Migration destructive sans filet de sécurité
async function dangerousMigration() {
  // Supprimer un champ sans backup
  await db.users.updateMany(
    {},
    { $unset: { oldField: "" } }
  );

  // Transformer des données de façon irréversible
  await db.orders.updateMany(
    {},
    [{
      $set: {
        // Perte de précision
        totalInDollars: { $divide: ["$totalInCents", 100] }
      }
    }],
    { $unset: { totalInCents: "" } }
  );

  // Si erreur ou régression découverte plus tard
  // → Données perdues définitivement
}
```

**Conséquences** :
- Données perdues irrécupérables
- Impossibilité de rollback
- Perte de précision (cents → dollars)
- Corruption de données si la transformation échoue
- Violation de réglementations (RGPD, auditabilité)

**Solution appropriée** :
```javascript
// ✅ Migration destructive avec sauvegarde
async function safeMigration() {
  console.log('Step 1: Backup data before destructive operation');

  // 1. Copier les données dans une collection de backup
  await db.users.aggregate([
    { $match: { oldField: { $exists: true } } },
    { $out: "users_backup_20240115" }
  ]);

  console.log('Step 2: Add new field while keeping old one');

  // 2. Ajouter le nouveau champ SANS supprimer l'ancien
  await db.orders.updateMany(
    { totalInDollars: { $exists: false } },
    [{
      $set: {
        totalInDollars: { $divide: ["$totalInCents", 100] }
      }
    }]
  );

  console.log('Step 3: Verify migration success');

  // 3. Vérifier que la migration est correcte
  const invalidCount = await db.orders.countDocuments({
    $expr: {
      $ne: [
        { $multiply: ["$totalInDollars", 100] },
        "$totalInCents"
      ]
    }
  });

  if (invalidCount > 0) {
    throw new Error(`Migration verification failed: ${invalidCount} invalid documents`);
  }

  console.log('Step 4: Wait for confirmation (e.g., 7 days)');

  // 4. Attendre une période de sécurité
  // Ne supprimer l'ancien champ qu'après confirmation

  console.log('Migration completed. Old field kept for safety.');
  console.log('Run cleanup after verification period.');
}

// Nettoyage séparé (à exécuter manuellement après confirmation)
async function cleanupOldField() {
  console.log('Removing old field after verification period...');

  await db.orders.updateMany(
    { totalInCents: { $exists: true } },
    { $unset: { totalInCents: "" } }
  );

  console.log('Cleanup completed.');
}
```

---

## ✅ DO : Tester les Migrations en Environnement de Staging

**Explication** : Chaque migration doit être testée exhaustivement avant d'être appliquée en production.

**Processus de test** :
```javascript
// ✅ Pipeline de test complet
class MigrationTester {
  async testMigration(migration) {
    console.log(`Testing migration: ${migration.name}`);

    // 1. Préparer environnement de test
    await this.setupTestEnvironment();

    // 2. Charger données de test
    await this.loadTestData();

    // 3. Exécuter la migration
    const startTime = Date.now();
    await migration.up();
    const duration = Date.now() - startTime;

    // 4. Vérifications
    await this.verifyDataIntegrity();
    await this.verifyBusinessLogic();
    await this.verifyPerformance(duration);

    // 5. Tester le rollback
    await migration.down();
    await this.verifyRollback();

    // 6. Cleanup
    await this.cleanup();

    console.log(`Migration test completed successfully`);
  }

  async verifyDataIntegrity() {
    // Vérifier que tous les documents ont la bonne structure
    const invalidDocs = await db.users.find({
      $or: [
        { schemaVersion: { $exists: false } },
        { firstName: { $exists: false } },
        { lastName: { $exists: false } }
      ]
    }).limit(10).toArray();

    if (invalidDocs.length > 0) {
      throw new Error(`Data integrity check failed: ${invalidDocs.length} invalid documents`);
    }
  }

  async verifyBusinessLogic() {
    // Vérifier que la logique métier fonctionne toujours
    const testCases = [
      { input: {...}, expected: {...} },
      // ... autres cas de test
    ];

    for (const testCase of testCases) {
      const result = await businessLogic(testCase.input);
      if (!deepEqual(result, testCase.expected)) {
        throw new Error('Business logic verification failed');
      }
    }
  }

  async verifyPerformance(migrationDuration) {
    // Vérifier que les performances sont acceptables
    const sampleSize = 1000;
    const startTime = Date.now();

    await db.users.find().limit(sampleSize).toArray();

    const queryDuration = Date.now() - startTime;
    const avgPerDoc = queryDuration / sampleSize;

    if (avgPerDoc > 5) { // 5ms par document
      console.warn(`Performance degradation detected: ${avgPerDoc}ms per document`);
    }
  }
}
```

**Tests avec données réelles** :
```javascript
// ✅ Tester avec un subset de données production
async function testWithProductionData() {
  // 1. Exporter un sample de production
  // mongodump --collection=users --query='{"createdAt":{$gte:ISODate("2024-01-01")}}' --limit=10000

  // 2. Importer en staging
  // mongorestore --collection=users_test dump/users.bson

  // 3. Anonymiser les données sensibles
  await db.users_test.updateMany(
    {},
    [{
      $set: {
        email: { $concat: ["test_", "$_id", "@example.com"] },
        phone: "555-0000",
        // Garder la structure, anonymiser les valeurs
      }
    }]
  );

  // 4. Exécuter la migration sur les données de test
  await runMigration();

  // 5. Valider les résultats
  await validateResults();
}
```

---

## ✅ DO : Documenter Toutes les Migrations

**Explication** : Chaque migration doit être documentée avec son contexte, sa raison d'être et ses impacts.

**Template de documentation** :
```javascript
// ✅ Documentation complète de migration
const migration_002 = {
  // Métadonnées
  id: "002",
  name: "split-user-name-field",
  version: 2,
  date: "2024-01-15",
  author: "alice@company.com",

  // Description
  description: `
    Split the 'name' field into 'firstName' and 'lastName' fields
    for better data structure and international name support.
  `,

  // Raison métier
  businessReason: `
    Current single 'name' field causes issues with:
    - International names (different order)
    - Sorting by last name
    - Formal vs informal addressing
    - Integration with external systems
  `,

  // Changements
  changes: [
    "Add firstName field (extracted from name)",
    "Add lastName field (extracted from name)",
    "Add birthDate field (calculated from age)",
    "Keep name field temporarily for rollback safety"
  ],

  // Impact
  impact: {
    documentsAffected: 10000000,
    estimatedDuration: "2-3 days (background)",
    downtime: "none",
    breaking: false
  },

  // Dépendances
  dependencies: {
    minAppVersion: "2.5.0",
    maxAppVersion: null,
    requiredIndexes: [],
    incompatibleWith: []
  },

  // Rollback
  rollback: {
    possible: true,
    automatic: true,
    manual: false,
    instructions: "Deploy app version 2.4.x"
  },

  // Tests
  testing: {
    unit: true,
    integration: true,
    performance: true,
    staging: true,
    canary: true
  },

  // Validation
  validation: {
    dataIntegrity: "All documents have firstName and lastName",
    businessLogic: "User display names work correctly",
    performance: "Query performance not degraded"
  },

  // Migration
  async up() {
    // Code de migration
  },

  async down() {
    // Code de rollback
  },

  // Post-migration cleanup (optionnel, après période de sécurité)
  async cleanup() {
    // Supprimer le champ 'name' après confirmation
  }
};
```

**Registre des migrations** :
```javascript
// ✅ Collection pour tracker les migrations
// Collection: _migrations
{
  _id: "002",
  name: "split-user-name-field",
  version: 2,
  status: "completed",  // pending, running, completed, failed, rolled_back
  startedAt: ISODate("2024-01-15T10:00:00Z"),
  completedAt: ISODate("2024-01-17T15:30:00Z"),
  duration: 183000000,  // ms
  documentsProcessed: 10000000,
  documentsFailed: 0,
  result: {
    success: true,
    message: "Migration completed successfully"
  },
  appliedBy: "alice@company.com",
  environment: "production"
}

// Fonction pour enregistrer une migration
async function recordMigration(migration, status, result = {}) {
  await db._migrations.updateOne(
    { _id: migration.id },
    {
      $set: {
        name: migration.name,
        version: migration.version,
        status: status,
        ...(status === 'completed' && { completedAt: new Date() }),
        ...(status === 'running' && { startedAt: new Date() }),
        result: result
      }
    },
    { upsert: true }
  );
}
```

---

## ❌ DON'T : Appliquer des Migrations Sans Plan de Rollback

**Explication** : Chaque migration doit avoir un plan de rollback clair et testé avant d'être appliquée.

**Anti-pattern** :
```javascript
// ❌ Migration sans rollback possible
async function pointOfNoReturn() {
  // Transformation irréversible
  await db.users.updateMany(
    {},
    [{
      $set: {
        // Hasher les emails (perte d'information)
        emailHash: { $function: {
          body: function(email) { return hash(email); },
          args: ["$email"],
          lang: "js"
        }}
      }
    }],
    { $unset: { email: "" } }  // Email original supprimé
  );

  // Impossible de récupérer les emails originaux!
}
```

**Solution avec rollback** :
```javascript
// ✅ Migration avec rollback possible
const migration = {
  async up() {
    console.log('Migration UP: Adding emailHash field');

    // Phase 1 : Ajouter le nouveau champ SANS supprimer l'ancien
    await db.users.updateMany(
      { emailHash: { $exists: false } },
      [{
        $set: {
          emailHash: { $function: {
            body: function(email) { return hash(email); },
            args: ["$email"],
            lang: "js"
          }}
        }
      }]
    );

    // email original conservé
    // Rollback possible en supprimant simplement emailHash
  },

  async down() {
    console.log('Migration DOWN: Removing emailHash field');

    // Rollback simple
    await db.users.updateMany(
      { emailHash: { $exists: true } },
      { $unset: { emailHash: "" } }
    );

    // email original toujours présent
  },

  async cleanup() {
    // Seulement après confirmation (ex: 30 jours)
    console.log('CLEANUP: Removing original email field');

    // Vérifier que le rollback n'est plus nécessaire
    const migrationRecord = await db._migrations.findOne({ _id: this.id });
    const daysSinceMigration = (Date.now() - migrationRecord.completedAt) / (1000 * 60 * 60 * 24);

    if (daysSinceMigration < 30) {
      throw new Error('Cannot cleanup: safety period not elapsed');
    }

    // Supprimer l'ancien champ
    await db.users.updateMany(
      { email: { $exists: true } },
      { $unset: { email: "" } }
    );
  }
};
```

---

## ✅ DO : Utiliser des Feature Flags pour les Migrations Applicatives

**Explication** : Les feature flags permettent de contrôler progressivement le déploiement d'une nouvelle version du schéma.

**Pattern recommandé** :
```javascript
// ✅ Migration contrôlée par feature flags
class UserService {
  constructor() {
    this.featureFlags = new FeatureFlags();
  }

  async getUser(userId) {
    const doc = await db.users.findOne({ _id: userId });

    // Vérifier si la nouvelle version du schéma est activée
    if (this.featureFlags.isEnabled('use_schema_v2')) {
      // Utiliser le nouveau schéma
      return this.mapToSchemaV2(doc);
    } else {
      // Utiliser l'ancien schéma
      return this.mapToSchemaV1(doc);
    }
  }

  async createUser(userData) {
    // Vérifier quelle version de schéma utiliser
    if (this.featureFlags.isEnabled('use_schema_v2')) {
      // Créer avec le nouveau schéma
      return await db.users.insertOne({
        schemaVersion: 2,
        firstName: userData.firstName,
        lastName: userData.lastName,
        ...
      });
    } else {
      // Créer avec l'ancien schéma
      return await db.users.insertOne({
        schemaVersion: 1,
        name: `${userData.firstName} ${userData.lastName}`,
        ...
      });
    }
  }
}

// Rollout progressif
const rolloutPlan = {
  phase1: { // Canary
    percentage: 1,
    duration: '24 hours',
    users: ['internal_testers']
  },
  phase2: { // Early adopters
    percentage: 10,
    duration: '48 hours'
  },
  phase3: { // General availability
    percentage: 100,
    duration: 'indefinite'
  }
};
```

**Feature flags dynamiques** :
```javascript
// ✅ Configuration des feature flags
// Collection: _feature_flags
{
  _id: "use_schema_v2",
  enabled: true,
  rolloutPercentage: 10,  // 10% des utilisateurs
  rolloutUsers: [         // Utilisateurs spécifiques
    "user123",
    "user456"
  ],
  enabledAt: ISODate("2024-01-15T10:00:00Z"),
  disabledAt: null,
  metadata: {
    owner: "alice@company.com",
    jiraTicket: "MIGRATION-123",
    description: "Enable schema v2 for user documents"
  }
}

class FeatureFlags {
  async isEnabled(flagName, context = {}) {
    const flag = await db._feature_flags.findOne({ _id: flagName });

    if (!flag || !flag.enabled) {
      return false;
    }

    // Vérifier si l'utilisateur est dans le rollout
    if (context.userId && flag.rolloutUsers?.includes(context.userId)) {
      return true;
    }

    // Vérifier le pourcentage de rollout
    if (flag.rolloutPercentage !== undefined) {
      const hash = this.hashUserId(context.userId);
      return (hash % 100) < flag.rolloutPercentage;
    }

    return true;
  }

  hashUserId(userId) {
    // Simple hash pour distribution uniforme
    return parseInt(userId.toString().substring(0, 8), 16) % 100;
  }
}
```

---

## ✅ DO : Monitorer les Migrations en Temps Réel

**Explication** : Une migration en cours doit être surveillée en continu pour détecter rapidement les problèmes.

**Métriques à surveiller** :
```javascript
// ✅ Dashboard de monitoring de migration
class MigrationMonitor {
  constructor(migrationId) {
    this.migrationId = migrationId;
    this.startTime = Date.now();
    this.metrics = {
      processed: 0,
      succeeded: 0,
      failed: 0,
      skipped: 0,
      rate: 0,
      errors: []
    };
  }

  async recordProgress(batch) {
    this.metrics.processed += batch.length;
    this.metrics.succeeded += batch.filter(d => d.success).length;
    this.metrics.failed += batch.filter(d => !d.success).length;

    // Calculer le rate
    const elapsed = (Date.now() - this.startTime) / 1000;
    this.metrics.rate = this.metrics.processed / elapsed;

    // Enregistrer dans la base
    await db._migration_progress.updateOne(
      { _id: this.migrationId },
      {
        $set: {
          ...this.metrics,
          lastUpdate: new Date(),
          estimatedCompletion: this.estimateCompletion()
        }
      },
      { upsert: true }
    );

    // Vérifier les seuils d'alerte
    await this.checkAlerts();
  }

  async checkAlerts() {
    // Alert si taux d'erreur trop élevé
    const errorRate = this.metrics.failed / this.metrics.processed;
    if (errorRate > 0.01) { // 1%
      await this.sendAlert('HIGH_ERROR_RATE', {
        errorRate: errorRate,
        failedCount: this.metrics.failed
      });
    }

    // Alert si rate trop lent
    if (this.metrics.rate < 100) { // < 100 docs/sec
      await this.sendAlert('SLOW_MIGRATION', {
        rate: this.metrics.rate
      });
    }

    // Alert si replica lag
    const replicaLag = await this.getReplicaLag();
    if (replicaLag > 30) { // > 30 secondes
      await this.sendAlert('HIGH_REPLICA_LAG', {
        lag: replicaLag
      });
    }
  }

  estimateCompletion() {
    const remaining = await db.users.countDocuments({
      schemaVersion: { $lt: 2 }
    });

    if (this.metrics.rate === 0) return null;

    const secondsRemaining = remaining / this.metrics.rate;
    return new Date(Date.now() + (secondsRemaining * 1000));
  }

  async sendAlert(type, data) {
    console.error(`MIGRATION ALERT: ${type}`, data);

    // Envoyer notification (Slack, email, PagerDuty, etc.)
    await notificationService.send({
      channel: '#migrations',
      severity: 'warning',
      title: `Migration Alert: ${type}`,
      data: data
    });
  }
}
```

**Tableau de bord de migration** :
```javascript
// ✅ Requête pour afficher le statut de migration
async function getMigrationDashboard() {
  const migrations = await db._migrations.find({
    status: { $in: ['running', 'pending'] }
  }).toArray();

  const dashboard = [];

  for (const migration of migrations) {
    const progress = await db._migration_progress.findOne({
      _id: migration._id
    });

    const total = await db.users.countDocuments({});
    const remaining = await db.users.countDocuments({
      schemaVersion: { $lt: migration.version }
    });

    dashboard.push({
      id: migration._id,
      name: migration.name,
      status: migration.status,
      progress: {
        total: total,
        processed: progress?.processed || 0,
        remaining: remaining,
        percentage: ((progress?.processed || 0) / total * 100).toFixed(2)
      },
      performance: {
        rate: progress?.rate || 0,
        estimatedCompletion: progress?.estimatedCompletion
      },
      errors: {
        count: progress?.failed || 0,
        rate: (progress?.failed || 0) / (progress?.processed || 1)
      }
    });
  }

  return dashboard;
}

// Affichage console
async function displayMigrationStatus() {
  const dashboard = await getMigrationDashboard();

  console.log('\n=== MIGRATION DASHBOARD ===\n');

  for (const migration of dashboard) {
    console.log(`Migration: ${migration.name} (${migration.id})`);
    console.log(`Status: ${migration.status}`);
    console.log(`Progress: ${migration.progress.percentage}% (${migration.progress.processed}/${migration.progress.total})`);
    console.log(`Rate: ${migration.performance.rate.toFixed(0)} docs/sec`);
    console.log(`ETA: ${migration.performance.estimatedCompletion || 'calculating...'}`);
    console.log(`Errors: ${migration.errors.count} (${(migration.errors.rate * 100).toFixed(2)}%)`);
    console.log('---\n');
  }
}
```

---

## Frameworks et Outils

### ✅ DO : Utiliser des Outils de Migration Dédiés

**Explication** : Des frameworks existent pour gérer les migrations de manière structurée.

**Options populaires** :
```javascript
// 1. migrate-mongo
// https://github.com/seppevs/migrate-mongo

// Configuration: migrate-mongo-config.js
module.exports = {
  mongodb: {
    url: process.env.MONGODB_URL,
    databaseName: process.env.DB_NAME,
  },
  migrationsDir: "migrations",
  changelogCollectionName: "_migrations"
};

// Migration: migrations/20240115-split-name-field.js
module.exports = {
  async up(db) {
    await db.collection('users').updateMany(
      { schemaVersion: 1 },
      [{
        $set: {
          schemaVersion: 2,
          firstName: { $arrayElemAt: [{ $split: ["$name", " "] }, 0] },
          lastName: { $arrayElemAt: [{ $split: ["$name", " "] }, 1] }
        }
      }]
    );
  },

  async down(db) {
    await db.collection('users').updateMany(
      { schemaVersion: 2 },
      [{
        $set: {
          schemaVersion: 1,
          name: { $concat: ["$firstName", " ", "$lastName"] }
        }
      }]
    );
  }
};

// Usage
// migrate-mongo up       # Apply migrations
// migrate-mongo down     # Rollback last migration
// migrate-mongo status   # Show migration status
```

```javascript
// 2. Mongoose Migrations
// Avec Mongoose (ODM)

import mongoose from 'mongoose';

const migrationSchema = new mongoose.Schema({
  version: Number,
  name: String,
  appliedAt: Date,
  status: String
});

const Migration = mongoose.model('Migration', migrationSchema);

class MigrationRunner {
  constructor() {
    this.migrations = [];
  }

  register(migration) {
    this.migrations.push(migration);
  }

  async run() {
    const appliedMigrations = await Migration.find({
      status: 'completed'
    });

    const appliedVersions = new Set(
      appliedMigrations.map(m => m.version)
    );

    for (const migration of this.migrations) {
      if (appliedVersions.has(migration.version)) {
        continue;
      }

      console.log(`Running migration: ${migration.name}`);

      try {
        await migration.up();

        await Migration.create({
          version: migration.version,
          name: migration.name,
          appliedAt: new Date(),
          status: 'completed'
        });

        console.log(`Migration completed: ${migration.name}`);
      } catch (error) {
        console.error(`Migration failed: ${migration.name}`, error);
        throw error;
      }
    }
  }
}
```

---

## Checklist Migrations

### Préparation
- [ ] Migration documentée (raison, impact, rollback)
- [ ] Tests unitaires écrits
- [ ] Tests d'intégration écrits
- [ ] Backup de la base créé
- [ ] Plan de rollback défini et testé
- [ ] Feature flags configurés (si applicable)

### Testing
- [ ] Test en environnement local
- [ ] Test avec données anonymisées de production
- [ ] Test de performance effectué
- [ ] Test de rollback validé
- [ ] Review de code effectuée

### Déploiement
- [ ] Migration testée en staging
- [ ] Monitoring configuré
- [ ] Alertes configurées
- [ ] Équipe informée et disponible
- [ ] Documentation à jour
- [ ] Canary deployment planifié

### Exécution
- [ ] Backup pré-migration créé
- [ ] Migration lancée (batch/lazy)
- [ ] Monitoring actif
- [ ] Métriques surveillées
- [ ] Logs vérifiés
- [ ] Aucune alerte critique

### Post-Migration
- [ ] Validation des données
- [ ] Tests fonctionnels passés
- [ ] Performance vérifiée
- [ ] Aucune régression détectée
- [ ] Documentation mise à jour
- [ ] Équipe informée du succès

### Cleanup (après période de sécurité)
- [ ] Période de sécurité écoulée (ex: 30 jours)
- [ ] Aucun problème signalé
- [ ] Rollback non nécessaire confirmé
- [ ] Cleanup des anciens champs exécuté
- [ ] Backup de pré-cleanup créé

---

## Conclusion

La gestion des migrations de schéma dans MongoDB requiert :

- **Stratégie** : Lazy vs eager, versioning, rollback
- **Prudence** : Tests, backups, monitoring
- **Progressivité** : Batches, feature flags, canary
- **Réversibilité** : Toujours pouvoir faire marche arrière
- **Documentation** : Traçabilité complète

**Règles d'or** :
1. **Lazy migrations** : Privilégier la migration progressive
2. **Versioning** : Toujours inclure schemaVersion
3. **Réversibilité** : Plan de rollback testé
4. **Tests** : Staging avant production
5. **Monitoring** : Surveillance en temps réel
6. **Prudence** : Conserver les anciennes données temporairement

Une migration bien gérée est invisible pour les utilisateurs. Une migration mal gérée peut causer des heures de downtime et des pertes de données.

---


⏭️ [Versioning des documents](/21-bonnes-pratiques-anti-patterns/07-versioning-documents.md)
