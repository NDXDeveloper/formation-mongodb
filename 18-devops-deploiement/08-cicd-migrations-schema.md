🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 18.8 CI/CD et Migrations de Schéma

## Introduction

Les migrations de schéma et l'intégration continue (CI/CD) sont essentielles pour maintenir la cohérence et la fiabilité des applications MongoDB en production. Bien que MongoDB soit schemaless, les applications ont besoin de structures de données cohérentes et évolutives, nécessitant des migrations contrôlées lors des changements de modèle.

```
┌────────────────────────────────────────────────────────────────────┐
│            CI/CD Pipeline for MongoDB Applications                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Source Control (Git)                      │  │
│  │                                                              │  │
│  │  • Application Code                                          │  │
│  │  • Migration Scripts                                         │  │
│  │  • Test Suites                                               │  │
│  │  • Infrastructure Code                                       │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │ Push/Merge                             │
│                           ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                   CI/CD Platform                             │  │
│  │         (GitLab CI, GitHub Actions, Jenkins)                 │  │
│  │                                                              │  │
│  │  ┌────────────────────────────────────────────────────────┐  │  │
│  │  │  Stage 1: Build & Test                                 │  │  │
│  │  │  • Lint code                                           │  │  │
│  │  │  • Unit tests                                          │  │  │
│  │  │  • Build Docker images                                 │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                           │                                  │  │
│  │  ┌────────────────────────▼───────────────────────────────┐  │  │
│  │  │  Stage 2: Integration Tests                            │  │  │
│  │  │  • Spin up MongoDB container                           │  │  │
│  │  │  • Run migration scripts                               │  │  │
│  │  │  • Integration tests                                   │  │  │
│  │  │  • API tests                                           │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                           │                                  │  │
│  │  ┌────────────────────────▼───────────────────────────────┐  │  │
│  │  │  Stage 3: Security & Quality                           │  │  │
│  │  │  • Dependency scanning                                 │  │  │
│  │  │  • Container scanning                                  │  │  │
│  │  │  • Code quality analysis                               │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                           │                                  │  │
│  │  ┌────────────────────────▼───────────────────────────────┐  │  │
│  │  │  Stage 4: Deploy to Staging                            │  │  │
│  │  │  • Run migrations (dry-run)                            │  │  │
│  │  │  • Deploy application                                  │  │  │
│  │  │  • Smoke tests                                         │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  │                           │                                  │  │
│  │  ┌────────────────────────▼───────────────────────────────┐  │  │
│  │  │  Stage 5: Deploy to Production                         │  │  │
│  │  │  • Pre-deployment backup                               │  │  │
│  │  │  • Run migrations                                      │  │  │
│  │  │  • Blue-Green/Canary deployment                        │  │  │
│  │  │  • Health checks                                       │  │  │
│  │  │  • Rollback on failure                                 │  │  │
│  │  └────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                           │                                        │
│                           ▼                                        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │               Production MongoDB Cluster                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### Défis des Migrations MongoDB

```yaml
# mongodb-migration-challenges.yaml
---
schemaless_nature:
  challenge: "MongoDB n'impose pas de schéma strict"
  implications:
    - "Documents peuvent avoir des structures différentes"
    - "Validation doit être gérée au niveau application"
    - "Migrations doivent gérer la coexistence de versions"

backward_compatibility:
  challenge: "Maintenir la compatibilité pendant le déploiement"
  implications:
    - "Ancienne et nouvelle versions de l'app coexistent"
    - "Migrations doivent être non-destructives"
    - "Rollback doit être possible"

large_datasets:
  challenge: "Migrations sur gros volumes de données"
  implications:
    - "Temps d'exécution long"
    - "Impact sur les performances"
    - "Besoin de migrations incrémentales"

zero_downtime:
  challenge: "Migrations sans interruption de service"
  implications:
    - "Stratégies de déploiement blue-green"
    - "Migrations en arrière-plan"
    - "Validation progressive"
```

---

## Stratégies de Migration de Schéma

### Approches de Migration

```javascript
// migration-strategies.js
/**
 * Stratégies de migration pour MongoDB
 */

// 1. Migration "Expand and Contract" (3 phases)
// Phase 1: Expand - Ajouter nouveaux champs
const expandPhase = async (db) => {
  // Ancienne structure coexiste avec la nouvelle
  await db.collection('users').updateMany(
    { newField: { $exists: false } },
    {
      $set: {
        newField: null,  // Placeholder
        migrationVersion: 2
      }
    }
  );
};

// Phase 2: Dual Write - Application écrit dans les deux
// L'application gère les deux formats pendant la transition
const dualWritePhase = async (user) => {
  return {
    // Old format
    firstName: user.firstName,
    lastName: user.lastName,

    // New format
    fullName: `${user.firstName} ${user.lastName}`,

    migrationVersion: 2
  };
};

// Phase 3: Contract - Supprimer anciens champs
const contractPhase = async (db) => {
  // Une fois que tous les documents sont migrés
  await db.collection('users').updateMany(
    {},
    {
      $unset: {
        firstName: "",
        lastName: ""
      },
      $set: {
        migrationVersion: 3
      }
    }
  );
};

// 2. Migration Lazy/On-Demand
const lazyMigration = async (db, userId) => {
  const user = await db.collection('users').findOne({ _id: userId });

  // Vérifier si la migration est nécessaire
  if (!user.fullName && user.firstName && user.lastName) {
    // Migrer à la volée
    await db.collection('users').updateOne(
      { _id: userId },
      {
        $set: {
          fullName: `${user.firstName} ${user.lastName}`,
          migrationVersion: 2
        },
        $unset: {
          firstName: "",
          lastName: ""
        }
      }
    );
  }

  return user;
};

// 3. Migration par Batch
const batchMigration = async (db, batchSize = 1000) => {
  let processed = 0;
  let hasMore = true;

  while (hasMore) {
    // Traiter par lots pour éviter la surcharge
    const batch = await db.collection('users')
      .find({ migrationVersion: { $lt: 2 } })
      .limit(batchSize)
      .toArray();

    if (batch.length === 0) {
      hasMore = false;
      continue;
    }

    // Préparer les opérations en bulk
    const bulkOps = batch.map(user => ({
      updateOne: {
        filter: { _id: user._id },
        update: {
          $set: {
            fullName: `${user.firstName} ${user.lastName}`,
            migrationVersion: 2
          },
          $unset: {
            firstName: "",
            lastName: ""
          }
        }
      }
    }));

    // Exécuter en bulk
    await db.collection('users').bulkWrite(bulkOps, { ordered: false });

    processed += batch.length;
    console.log(`Processed ${processed} documents`);

    // Pause pour ne pas surcharger la DB
    await new Promise(resolve => setTimeout(resolve, 100));
  }

  return processed;
};

// 4. Migration avec Versioning
const versionedMigration = {
  // Schéma version 1
  v1: {
    validate: (doc) => doc.firstName && doc.lastName,
    upgrade: async (db, doc) => {
      return {
        ...doc,
        fullName: `${doc.firstName} ${doc.lastName}`,
        _schemaVersion: 2
      };
    }
  },

  // Schéma version 2
  v2: {
    validate: (doc) => doc.fullName,
    upgrade: async (db, doc) => {
      // Future migration
      return doc;
    }
  }
};

// Fonction pour appliquer les migrations
const applyMigrations = async (db, doc) => {
  let currentDoc = doc;
  const currentVersion = doc._schemaVersion || 1;
  const targetVersion = 2;

  // Appliquer les migrations séquentiellement
  for (let v = currentVersion; v < targetVersion; v++) {
    const migration = versionedMigration[`v${v}`];
    if (migration && migration.upgrade) {
      currentDoc = await migration.upgrade(db, currentDoc);
    }
  }

  return currentDoc;
};

module.exports = {
  expandPhase,
  dualWritePhase,
  contractPhase,
  lazyMigration,
  batchMigration,
  versionedMigration,
  applyMigrations
};
```

---

## Outils de Migration

### Migrate-Mongo

```javascript
// Configuration migrate-mongo
// migrate-mongo-config.js
module.exports = {
  mongodb: {
    url: process.env.MONGODB_URI || "mongodb://localhost:27017",
    databaseName: process.env.MONGODB_DATABASE || "myapp",

    options: {
      useNewUrlParser: true,
      useUnifiedTopology: true,
      connectTimeoutMS: 3600000,
      socketTimeoutMS: 3600000,
    }
  },

  // Répertoire des migrations
  migrationsDir: "migrations",

  // Nom de la collection pour tracker les migrations
  changelogCollectionName: "changelog",

  // Fichier de status des migrations
  migrationFileExtension: ".js",

  // Module loader
  useFileHash: false,

  // Timeout
  moduleSystem: 'commonjs',
};
```

### Migration Script Example

```javascript
// migrations/20240115120000-add-fullname-field.js
/**
 * Migration: Ajouter le champ fullName en combinant firstName et lastName
 * Version: 2.0.0
 * Date: 2024-01-15
 */

module.exports = {
  /**
   * Migration Up - Appliquer les changements
   */
  async up(db, client) {
    console.log('Starting migration: add-fullname-field');

    // Créer un index temporaire pour la migration
    await db.collection('users').createIndex(
      { migrationVersion: 1 },
      { name: 'idx_migration_version' }
    );

    const session = client.startSession();

    try {
      await session.withTransaction(async () => {
        // Compter le nombre de documents à migrer
        const totalCount = await db.collection('users').countDocuments({
          migrationVersion: { $lt: 2 }
        });

        console.log(`Total documents to migrate: ${totalCount}`);

        const batchSize = 1000;
        let processed = 0;

        // Migration par batch
        while (processed < totalCount) {
          const batch = await db.collection('users')
            .find({
              migrationVersion: { $lt: 2 },
              firstName: { $exists: true },
              lastName: { $exists: true }
            })
            .limit(batchSize)
            .toArray();

          if (batch.length === 0) break;

          // Préparer les opérations bulk
          const bulkOps = batch.map(user => ({
            updateOne: {
              filter: { _id: user._id },
              update: {
                $set: {
                  fullName: `${user.firstName} ${user.lastName}`,
                  migrationVersion: 2,
                  migratedAt: new Date()
                }
              }
            }
          }));

          // Exécuter
          const result = await db.collection('users').bulkWrite(
            bulkOps,
            { ordered: false, session }
          );

          processed += result.modifiedCount;

          console.log(`Progress: ${processed}/${totalCount} (${Math.round(processed/totalCount*100)}%)`);

          // Pause pour ne pas surcharger
          await new Promise(resolve => setTimeout(resolve, 100));
        }

        console.log(`Migration completed: ${processed} documents migrated`);
      });
    } finally {
      await session.endSession();
    }

    // Créer un index sur le nouveau champ
    await db.collection('users').createIndex(
      { fullName: 'text' },
      { name: 'idx_fullname_text' }
    );

    // Nettoyer l'index temporaire
    await db.collection('users').dropIndex('idx_migration_version');
  },

  /**
   * Migration Down - Rollback
   */
  async down(db, client) {
    console.log('Rolling back migration: add-fullname-field');

    const session = client.startSession();

    try {
      await session.withTransaction(async () => {
        // Restaurer les champs firstName et lastName à partir de fullName
        const users = await db.collection('users')
          .find({ migrationVersion: 2 })
          .toArray();

        const bulkOps = users.map(user => {
          // Parser le fullName (simpliste)
          const parts = user.fullName ? user.fullName.split(' ') : ['', ''];
          const firstName = parts[0] || '';
          const lastName = parts.slice(1).join(' ') || '';

          return {
            updateOne: {
              filter: { _id: user._id },
              update: {
                $set: {
                  firstName,
                  lastName,
                  migrationVersion: 1
                },
                $unset: {
                  fullName: "",
                  migratedAt: ""
                }
              }
            }
          };
        });

        if (bulkOps.length > 0) {
          await db.collection('users').bulkWrite(bulkOps, { session });
        }

        console.log(`Rollback completed: ${bulkOps.length} documents restored`);
      });
    } finally {
      await session.endSession();
    }

    // Supprimer l'index fullName
    try {
      await db.collection('users').dropIndex('idx_fullname_text');
    } catch (error) {
      console.log('Index already dropped or does not exist');
    }
  }
};
```

### Migration avec Validation Schema

```javascript
// migrations/20240116100000-add-user-validation.js
/**
 * Migration: Ajouter validation schema pour users
 */

module.exports = {
  async up(db) {
    console.log('Adding validation schema to users collection');

    await db.command({
      collMod: 'users',
      validator: {
        $jsonSchema: {
          bsonType: 'object',
          required: ['email', 'fullName', 'createdAt'],
          properties: {
            email: {
              bsonType: 'string',
              pattern: '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$',
              description: 'Valid email address required'
            },
            fullName: {
              bsonType: 'string',
              minLength: 2,
              maxLength: 100,
              description: 'Full name between 2-100 characters'
            },
            age: {
              bsonType: 'int',
              minimum: 0,
              maximum: 150,
              description: 'Age must be between 0 and 150'
            },
            status: {
              enum: ['active', 'inactive', 'suspended', 'deleted'],
              description: 'Status must be one of the enum values'
            },
            createdAt: {
              bsonType: 'date',
              description: 'Creation date required'
            },
            migrationVersion: {
              bsonType: 'int',
              minimum: 2,
              description: 'Migration version must be at least 2'
            }
          }
        }
      },
      validationLevel: 'moderate',  // strict ou moderate
      validationAction: 'warn'       // error ou warn
    });

    console.log('Validation schema added successfully');
  },

  async down(db) {
    console.log('Removing validation schema from users collection');

    await db.command({
      collMod: 'users',
      validator: {},
      validationLevel: 'off'
    });

    console.log('Validation schema removed');
  }
};
```

---

## Pipeline CI/CD Complet

### GitLab CI/CD

```yaml
# .gitlab-ci.yml
# Pipeline CI/CD complet pour application MongoDB

stages:
  - build
  - test
  - migrate
  - deploy-staging
  - test-staging
  - deploy-production
  - verify

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  NODE_VERSION: "20"
  MONGODB_VERSION: "7.0"

# Templates
.mongodb_service: &mongodb_service
  services:
    - name: mongo:${MONGODB_VERSION}
      alias: mongodb
  variables:
    MONGODB_URI: "mongodb://mongodb:27017/test"

# Build stage
build:
  stage: build
  image: node:${NODE_VERSION}

  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
      - .npm/

  script:
    - npm ci --cache .npm --prefer-offline
    - npm run build
    - npm run lint

  artifacts:
    paths:
      - dist/
      - node_modules/
    expire_in: 1 hour

# Unit Tests
test:unit:
  stage: test
  image: node:${NODE_VERSION}

  dependencies:
    - build

  script:
    - npm run test:unit -- --coverage

  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'

  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      junit: junit.xml
    paths:
      - coverage/
    expire_in: 1 week

# Integration Tests
test:integration:
  stage: test
  image: node:${NODE_VERSION}

  <<: *mongodb_service

  dependencies:
    - build

  script:
    # Attendre que MongoDB soit prêt
    - |
      echo "Waiting for MongoDB to be ready..."
      for i in {1..30}; do
        if nc -z mongodb 27017; then
          echo "MongoDB is ready!"
          break
        fi
        echo "Waiting... ($i/30)"
        sleep 2
      done

    # Exécuter les tests d'intégration
    - npm run test:integration

  artifacts:
    reports:
      junit: junit-integration.xml
    expire_in: 1 week

# Migration Dry-Run
migrate:dry-run:
  stage: migrate
  image: node:${NODE_VERSION}

  <<: *mongodb_service

  dependencies:
    - build

  script:
    - npm run migrate:status
    - npm run migrate:up -- --dry-run

  only:
    - merge_requests
    - main

# Security Scanning
security:dependency-scan:
  stage: test
  image: node:${NODE_VERSION}

  script:
    - npm audit --audit-level=moderate

  allow_failure: true

security:container-scan:
  stage: test
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]

  script:
    - trivy image --severity HIGH,CRITICAL --no-progress ${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHA}

  dependencies: []
  allow_failure: true

# Deploy to Staging
deploy:staging:
  stage: deploy-staging
  image: node:${NODE_VERSION}

  environment:
    name: staging
    url: https://staging.example.com

  dependencies:
    - build

  variables:
    MONGODB_URI: $STAGING_MONGODB_URI
    MONGODB_DATABASE: $STAGING_MONGODB_DATABASE

  script:
    - echo "Deploying to staging..."

    # Backup avant migration
    - npm run backup:create

    # Exécuter les migrations
    - npm run migrate:up

    # Vérifier les migrations
    - npm run migrate:status

    # Déployer l'application
    - npm run deploy:staging

    # Smoke tests
    - npm run test:smoke

  only:
    - main

# Staging Smoke Tests
test:staging-smoke:
  stage: test-staging
  image: node:${NODE_VERSION}

  dependencies: []

  variables:
    API_URL: https://staging.example.com

  script:
    - npm run test:smoke -- --env=staging

  only:
    - main

# Deploy to Production (Manual)
deploy:production:
  stage: deploy-production
  image: node:${NODE_VERSION}

  environment:
    name: production
    url: https://example.com

  dependencies:
    - build

  variables:
    MONGODB_URI: $PRODUCTION_MONGODB_URI
    MONGODB_DATABASE: $PRODUCTION_MONGODB_DATABASE

  before_script:
    # Vérifications pré-déploiement
    - |
      echo "Pre-deployment checks..."
      npm run healthcheck:production || exit 1

  script:
    - echo "Deploying to production..."

    # 1. Backup complet
    - echo "Creating backup..."
    - npm run backup:create -- --compress --verify

    # 2. Vérifier les migrations en dry-run
    - echo "Dry-run migrations..."
    - npm run migrate:up -- --dry-run || exit 1

    # 3. Exécuter les migrations
    - echo "Running migrations..."
    - npm run migrate:up

    # 4. Vérifier le statut des migrations
    - npm run migrate:status

    # 5. Déploiement Blue-Green
    - echo "Blue-Green deployment..."
    - npm run deploy:production -- --strategy=blue-green

    # 6. Health checks
    - echo "Running health checks..."
    - npm run healthcheck:production -- --wait=30

    # 7. Smoke tests en production
    - echo "Running smoke tests..."
    - npm run test:smoke -- --env=production

  after_script:
    # Notifier sur Slack
    - |
      if [ $CI_JOB_STATUS == 'success' ]; then
        curl -X POST $SLACK_WEBHOOK_URL \
          -H 'Content-Type: application/json' \
          -d "{\"text\":\"✅ Production deployment successful: $CI_COMMIT_SHORT_SHA\"}"
      else
        curl -X POST $SLACK_WEBHOOK_URL \
          -H 'Content-Type: application/json' \
          -d "{\"text\":\"❌ Production deployment failed: $CI_COMMIT_SHORT_SHA\"}"
      fi

  when: manual
  only:
    - main

  # Retry automatique en cas d'échec transitoire
  retry:
    max: 2
    when:
      - script_failure

# Verification Post-Deployment
verify:production:
  stage: verify
  image: node:${NODE_VERSION}

  dependencies: []

  variables:
    API_URL: https://example.com

  script:
    - npm run test:e2e -- --env=production
    - npm run monitor:check

  only:
    - main

  needs:
    - deploy:production

# Rollback (Manual)
rollback:production:
  stage: deploy-production
  image: node:${NODE_VERSION}

  environment:
    name: production
    action: rollback

  variables:
    MONGODB_URI: $PRODUCTION_MONGODB_URI

  script:
    - echo "Rolling back production deployment..."

    # 1. Rollback de l'application
    - npm run deploy:rollback

    # 2. Rollback des migrations
    - npm run migrate:down -- --count=1

    # 3. Restore du backup
    - npm run backup:restore -- --latest

    # 4. Health checks
    - npm run healthcheck:production

  when: manual
  only:
    - main
```

### GitHub Actions

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '20'
  MONGODB_VERSION: '7.0'

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Build
        run: npm run build

      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-artifacts
          path: |
            dist/
            node_modules/
          retention-days: 1

  test-unit:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: build

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts

      - name: Run unit tests
        run: npm run test:unit -- --coverage

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
          flags: unittests

  test-integration:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: build

    services:
      mongodb:
        image: mongo:${{ env.MONGODB_VERSION }}
        ports:
          - 27017:27017
        options: >-
          --health-cmd "mongosh --eval 'db.adminCommand(\"ping\")'"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts

      - name: Run integration tests
        env:
          MONGODB_URI: mongodb://localhost:27017/test
        run: npm run test:integration

  migrate-dry-run:
    name: Migration Dry-Run
    runs-on: ubuntu-latest
    needs: build
    if: github.event_name == 'pull_request'

    services:
      mongodb:
        image: mongo:${{ env.MONGODB_VERSION }}
        ports:
          - 27017:27017

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts

      - name: Migration status
        env:
          MONGODB_URI: mongodb://localhost:27017/test
        run: npm run migrate:status

      - name: Migration dry-run
        env:
          MONGODB_URI: mongodb://localhost:27017/test
        run: npm run migrate:up -- --dry-run

  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [test-unit, test-integration]
    if: github.ref == 'refs/heads/main'

    environment:
      name: staging
      url: https://staging.example.com

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts

      - name: Create backup
        env:
          MONGODB_URI: ${{ secrets.STAGING_MONGODB_URI }}
        run: npm run backup:create

      - name: Run migrations
        env:
          MONGODB_URI: ${{ secrets.STAGING_MONGODB_URI }}
        run: npm run migrate:up

      - name: Deploy to staging
        run: npm run deploy:staging

      - name: Smoke tests
        run: npm run test:smoke -- --env=staging

  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'

    environment:
      name: production
      url: https://example.com

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}

      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts

      - name: Pre-deployment checks
        env:
          API_URL: https://example.com
        run: npm run healthcheck:production

      - name: Create backup
        env:
          MONGODB_URI: ${{ secrets.PRODUCTION_MONGODB_URI }}
        run: npm run backup:create -- --compress --verify

      - name: Migration dry-run
        env:
          MONGODB_URI: ${{ secrets.PRODUCTION_MONGODB_URI }}
        run: npm run migrate:up -- --dry-run

      - name: Run migrations
        env:
          MONGODB_URI: ${{ secrets.PRODUCTION_MONGODB_URI }}
        run: npm run migrate:up

      - name: Blue-Green deployment
        run: npm run deploy:production -- --strategy=blue-green

      - name: Health checks
        run: npm run healthcheck:production -- --wait=30

      - name: Smoke tests
        run: npm run test:smoke -- --env=production

      - name: Notify Slack
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment completed'
          webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Scripts de Migration et Déploiement

### package.json Scripts

```json
{
  "name": "mongodb-app",
  "version": "1.0.0",
  "scripts": {
    "build": "tsc",
    "lint": "eslint . --ext .ts,.js",
    "test": "jest",
    "test:unit": "jest --testPathPattern=unit",
    "test:integration": "jest --testPathPattern=integration --runInBand",
    "test:e2e": "jest --testPathPattern=e2e --runInBand",
    "test:smoke": "node scripts/smoke-tests.js",

    "migrate:create": "migrate-mongo create",
    "migrate:up": "migrate-mongo up",
    "migrate:down": "migrate-mongo down",
    "migrate:status": "migrate-mongo status",

    "backup:create": "node scripts/backup.js create",
    "backup:restore": "node scripts/backup.js restore",
    "backup:list": "node scripts/backup.js list",

    "deploy:staging": "node scripts/deploy.js --env=staging",
    "deploy:production": "node scripts/deploy.js --env=production",
    "deploy:rollback": "node scripts/rollback.js",

    "healthcheck:production": "node scripts/healthcheck.js --env=production",
    "monitor:check": "node scripts/monitor.js"
  }
}
```

### Backup Script

```javascript
// scripts/backup.js
const { MongoClient } = require('mongodb');
const { exec } = require('child_process');
const { promisify } = require('util');
const path = require('path');
const fs = require('fs').promises;
const AWS = require('aws-sdk');

const execPromise = promisify(exec);
const s3 = new AWS.S3();

class BackupManager {
  constructor(uri, database) {
    this.uri = uri;
    this.database = database;
    this.backupDir = process.env.BACKUP_DIR || '/tmp/backups';
    this.s3Bucket = process.env.S3_BACKUP_BUCKET;
  }

  async create(options = {}) {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const backupName = `backup-${this.database}-${timestamp}`;
    const backupPath = path.join(this.backupDir, backupName);

    console.log(`Creating backup: ${backupName}`);

    // Créer le répertoire de backup
    await fs.mkdir(backupPath, { recursive: true });

    try {
      // Exécuter mongodump
      const dumpCmd = [
        'mongodump',
        `--uri="${this.uri}"`,
        `--db=${this.database}`,
        `--out=${backupPath}`,
        '--gzip',
        '--oplog'
      ].join(' ');

      const { stdout, stderr } = await execPromise(dumpCmd);
      console.log('Mongodump output:', stdout);

      if (stderr) {
        console.error('Mongodump stderr:', stderr);
      }

      // Créer l'archive
      if (options.compress) {
        console.log('Compressing backup...');
        const tarCmd = `tar czf ${backupPath}.tar.gz -C ${this.backupDir} ${backupName}`;
        await execPromise(tarCmd);

        // Supprimer le répertoire non compressé
        await fs.rm(backupPath, { recursive: true });
      }

      // Vérifier le backup
      if (options.verify) {
        console.log('Verifying backup...');
        await this.verify(backupPath + (options.compress ? '.tar.gz' : ''));
      }

      // Upload vers S3
      if (this.s3Bucket) {
        await this.uploadToS3(
          backupPath + (options.compress ? '.tar.gz' : ''),
          `${backupName}${options.compress ? '.tar.gz' : ''}`
        );
      }

      console.log(`✅ Backup created successfully: ${backupName}`);

      return {
        name: backupName,
        path: backupPath + (options.compress ? '.tar.gz' : ''),
        timestamp: new Date()
      };

    } catch (error) {
      console.error('Backup failed:', error);
      throw error;
    }
  }

  async restore(backupName, options = {}) {
    console.log(`Restoring backup: ${backupName}`);

    let backupPath;

    // Télécharger depuis S3 si nécessaire
    if (backupName.startsWith('s3://')) {
      backupPath = await this.downloadFromS3(backupName);
    } else {
      backupPath = path.join(this.backupDir, backupName);
    }

    // Décompresser si nécessaire
    if (backupPath.endsWith('.tar.gz')) {
      console.log('Extracting backup...');
      const extractCmd = `tar xzf ${backupPath} -C ${this.backupDir}`;
      await execPromise(extractCmd);
      backupPath = backupPath.replace('.tar.gz', '');
    }

    try {
      // Exécuter mongorestore
      const restoreCmd = [
        'mongorestore',
        `--uri="${this.uri}"`,
        `--db=${this.database}`,
        `--dir=${backupPath}/${this.database}`,
        '--gzip',
        options.drop ? '--drop' : '',
        '--oplogReplay'
      ].filter(Boolean).join(' ');

      const { stdout, stderr } = await execPromise(restoreCmd);
      console.log('Mongorestore output:', stdout);

      if (stderr) {
        console.error('Mongorestore stderr:', stderr);
      }

      console.log(`✅ Backup restored successfully`);

    } catch (error) {
      console.error('Restore failed:', error);
      throw error;
    }
  }

  async list() {
    console.log('Listing backups...');

    // Lister les backups locaux
    const localBackups = await fs.readdir(this.backupDir);
    console.log('Local backups:', localBackups);

    // Lister les backups S3
    if (this.s3Bucket) {
      const s3Backups = await this.listS3Backups();
      console.log('S3 backups:', s3Backups);
    }
  }

  async verify(backupPath) {
    // Vérification basique de l'intégrité
    const stats = await fs.stat(backupPath);

    if (stats.size === 0) {
      throw new Error('Backup file is empty');
    }

    console.log(`Backup size: ${(stats.size / 1024 / 1024).toFixed(2)} MB`);
    return true;
  }

  async uploadToS3(localPath, s3Key) {
    console.log(`Uploading to S3: s3://${this.s3Bucket}/${s3Key}`);

    const fileContent = await fs.readFile(localPath);

    await s3.putObject({
      Bucket: this.s3Bucket,
      Key: s3Key,
      Body: fileContent,
      StorageClass: 'STANDARD_IA',
      Metadata: {
        'backup-date': new Date().toISOString(),
        'database': this.database
      }
    }).promise();

    console.log('✅ Uploaded to S3');
  }

  async downloadFromS3(s3Uri) {
    const s3Key = s3Uri.replace(`s3://${this.s3Bucket}/`, '');
    const localPath = path.join(this.backupDir, path.basename(s3Key));

    console.log(`Downloading from S3: ${s3Uri}`);

    const data = await s3.getObject({
      Bucket: this.s3Bucket,
      Key: s3Key
    }).promise();

    await fs.writeFile(localPath, data.Body);

    console.log('✅ Downloaded from S3');
    return localPath;
  }

  async listS3Backups() {
    const data = await s3.listObjectsV2({
      Bucket: this.s3Bucket,
      Prefix: 'backup-'
    }).promise();

    return data.Contents.map(obj => ({
      key: obj.Key,
      size: obj.Size,
      lastModified: obj.LastModified
    }));
  }
}

// CLI
async function main() {
  const command = process.argv[2];
  const uri = process.env.MONGODB_URI;
  const database = process.env.MONGODB_DATABASE;

  const backup = new BackupManager(uri, database);

  switch (command) {
    case 'create':
      await backup.create({
        compress: process.argv.includes('--compress'),
        verify: process.argv.includes('--verify')
      });
      break;

    case 'restore':
      const backupName = process.argv[3];
      if (!backupName) {
        console.error('Backup name required');
        process.exit(1);
      }
      await backup.restore(backupName, {
        drop: process.argv.includes('--drop')
      });
      break;

    case 'list':
      await backup.list();
      break;

    default:
      console.error('Unknown command:', command);
      console.log('Usage: node backup.js <create|restore|list>');
      process.exit(1);
  }
}

if (require.main === module) {
  main().catch(error => {
    console.error(error);
    process.exit(1);
  });
}

module.exports = BackupManager;
```

### Deployment Script avec Blue-Green

```javascript
// scripts/deploy.js
const { MongoClient } = require('mongodb');
const k8s = require('@kubernetes/client-node');

class DeploymentManager {
  constructor(env) {
    this.env = env;
    this.namespace = env === 'production' ? 'production' : 'staging';

    // Kubernetes client
    const kc = new k8s.KubeConfig();
    kc.loadFromDefault();
    this.k8sAppsApi = kc.makeApiClient(k8s.AppsV1Api);
    this.k8sCoreApi = kc.makeApiClient(k8s.CoreV1Api);
  }

  async blueGreenDeploy(options = {}) {
    console.log(`Starting Blue-Green deployment to ${this.env}...`);

    const appName = 'myapp';
    const currentColor = await this.getCurrentColor(appName);
    const newColor = currentColor === 'blue' ? 'green' : 'blue';

    console.log(`Current: ${currentColor}, Deploying: ${newColor}`);

    try {
      // 1. Déployer la nouvelle version (green)
      await this.deployVersion(appName, newColor, options.version);

      // 2. Attendre que les pods soient prêts
      await this.waitForReady(appName, newColor);

      // 3. Health checks
      const healthy = await this.healthCheck(appName, newColor);

      if (!healthy) {
        throw new Error('Health check failed on new deployment');
      }

      // 4. Smoke tests
      await this.runSmokeTests(appName, newColor);

      // 5. Basculer le trafic
      await this.switchTraffic(appName, newColor);

      // 6. Vérifier le nouveau déploiement
      await this.monitorDeployment(appName, newColor, 300); // 5 minutes

      // 7. Supprimer l'ancienne version
      if (options.cleanup) {
        await this.cleanupOldVersion(appName, currentColor);
      }

      console.log(`✅ Blue-Green deployment completed successfully`);

    } catch (error) {
      console.error('Deployment failed:', error);

      // Rollback automatique
      if (currentColor) {
        console.log('Rolling back to previous version...');
        await this.switchTraffic(appName, currentColor);
      }

      throw error;
    }
  }

  async canaryDeploy(options = {}) {
    console.log(`Starting Canary deployment to ${this.env}...`);

    const appName = 'myapp';
    const canaryPercentage = options.canaryPercentage || 10;

    try {
      // 1. Déployer la version canary
      await this.deployCanary(appName, options.version, canaryPercentage);

      // 2. Surveiller les métriques
      const metricsOk = await this.monitorCanary(appName, 600); // 10 minutes

      if (!metricsOk) {
        throw new Error('Canary metrics show degradation');
      }

      // 3. Augmenter progressivement le trafic
      for (const percentage of [25, 50, 75, 100]) {
        console.log(`Increasing canary traffic to ${percentage}%`);
        await this.updateCanaryTraffic(appName, percentage);
        await this.monitorCanary(appName, 300); // 5 minutes
      }

      // 4. Cleanup
      await this.finalizeCanary(appName);

      console.log(`✅ Canary deployment completed successfully`);

    } catch (error) {
      console.error('Canary deployment failed:', error);

      // Rollback
      await this.rollbackCanary(appName);

      throw error;
    }
  }

  async getCurrentColor(appName) {
    try {
      const service = await this.k8sCoreApi.readNamespacedService(
        appName,
        this.namespace
      );

      return service.body.spec.selector.color || 'blue';
    } catch (error) {
      return 'blue'; // Default
    }
  }

  async deployVersion(appName, color, version) {
    console.log(`Deploying ${appName}-${color} version ${version}`);

    const deployment = {
      apiVersion: 'apps/v1',
      kind: 'Deployment',
      metadata: {
        name: `${appName}-${color}`,
        namespace: this.namespace,
        labels: {
          app: appName,
          color: color,
          version: version
        }
      },
      spec: {
        replicas: 3,
        selector: {
          matchLabels: {
            app: appName,
            color: color
          }
        },
        template: {
          metadata: {
            labels: {
              app: appName,
              color: color,
              version: version
            }
          },
          spec: {
            containers: [{
              name: appName,
              image: `myregistry/${appName}:${version}`,
              ports: [{
                containerPort: 3000
              }],
              env: [{
                name: 'MONGODB_URI',
                valueFrom: {
                  secretKeyRef: {
                    name: 'mongodb-uri',
                    key: 'uri'
                  }
                }
              }],
              readinessProbe: {
                httpGet: {
                  path: '/health',
                  port: 3000
                },
                initialDelaySeconds: 10,
                periodSeconds: 5
              },
              livenessProbe: {
                httpGet: {
                  path: '/health',
                  port: 3000
                },
                initialDelaySeconds: 30,
                periodSeconds: 10
              }
            }]
          }
        }
      }
    };

    await this.k8sAppsApi.createNamespacedDeployment(
      this.namespace,
      deployment
    );
  }

  async waitForReady(appName, color, timeout = 300) {
    console.log(`Waiting for ${appName}-${color} to be ready...`);

    const startTime = Date.now();

    while (Date.now() - startTime < timeout * 1000) {
      const deployment = await this.k8sAppsApi.readNamespacedDeployment(
        `${appName}-${color}`,
        this.namespace
      );

      const status = deployment.body.status;

      if (status.readyReplicas === status.replicas) {
        console.log(`✅ ${appName}-${color} is ready`);
        return true;
      }

      await new Promise(resolve => setTimeout(resolve, 5000));
    }

    throw new Error(`Timeout waiting for ${appName}-${color} to be ready`);
  }

  async healthCheck(appName, color) {
    console.log(`Running health check on ${appName}-${color}...`);

    // Get pod IPs
    const pods = await this.k8sCoreApi.listNamespacedPod(
      this.namespace,
      undefined,
      undefined,
      undefined,
      undefined,
      `app=${appName},color=${color}`
    );

    // Check health on each pod
    for (const pod of pods.body.items) {
      const podIp = pod.status.podIP;
      const response = await fetch(`http://${podIp}:3000/health`);

      if (!response.ok) {
        console.error(`Health check failed on pod ${pod.metadata.name}`);
        return false;
      }
    }

    console.log('✅ Health check passed');
    return true;
  }

  async runSmokeTests(appName, color) {
    console.log(`Running smoke tests on ${appName}-${color}...`);

    // Exécuter les smoke tests via le service interne
    const serviceUrl = `http://${appName}-${color}.${this.namespace}.svc.cluster.local:3000`;

    // Tests basiques
    const tests = [
      { path: '/health', expectedStatus: 200 },
      { path: '/api/version', expectedStatus: 200 },
      { path: '/api/users', expectedStatus: 200 }
    ];

    for (const test of tests) {
      const response = await fetch(`${serviceUrl}${test.path}`);

      if (response.status !== test.expectedStatus) {
        throw new Error(`Smoke test failed: ${test.path} returned ${response.status}`);
      }
    }

    console.log('✅ Smoke tests passed');
  }

  async switchTraffic(appName, newColor) {
    console.log(`Switching traffic to ${newColor}...`);

    const service = {
      apiVersion: 'v1',
      kind: 'Service',
      metadata: {
        name: appName,
        namespace: this.namespace
      },
      spec: {
        selector: {
          app: appName,
          color: newColor
        },
        ports: [{
          port: 80,
          targetPort: 3000
        }]
      }
    };

    await this.k8sCoreApi.patchNamespacedService(
      appName,
      this.namespace,
      service,
      undefined,
      undefined,
      undefined,
      undefined,
      { headers: { 'Content-Type': 'application/strategic-merge-patch+json' } }
    );

    console.log(`✅ Traffic switched to ${newColor}`);
  }

  async monitorDeployment(appName, color, duration) {
    console.log(`Monitoring deployment for ${duration} seconds...`);

    const startTime = Date.now();
    const interval = 30000; // 30 seconds

    while (Date.now() - startTime < duration * 1000) {
      // Check error rate
      const errorRate = await this.getErrorRate(appName, color);

      if (errorRate > 5) { // 5% error threshold
        throw new Error(`High error rate detected: ${errorRate}%`);
      }

      // Check response time
      const responseTime = await this.getAvgResponseTime(appName, color);

      if (responseTime > 1000) { // 1 second threshold
        console.warn(`High response time: ${responseTime}ms`);
      }

      await new Promise(resolve => setTimeout(resolve, interval));
    }

    console.log('✅ Monitoring completed successfully');
  }

  async getErrorRate(appName, color) {
    // Query Prometheus or metrics system
    // Placeholder implementation
    return Math.random() * 2; // Simulate error rate
  }

  async getAvgResponseTime(appName, color) {
    // Query Prometheus or metrics system
    // Placeholder implementation
    return Math.random() * 500; // Simulate response time
  }

  async cleanupOldVersion(appName, color) {
    console.log(`Cleaning up ${appName}-${color}...`);

    await this.k8sAppsApi.deleteNamespacedDeployment(
      `${appName}-${color}`,
      this.namespace
    );

    console.log(`✅ Cleanup completed`);
  }
}

// CLI
async function main() {
  const env = process.argv.find(arg => arg.startsWith('--env='))?.split('=')[1] || 'staging';
  const strategy = process.argv.find(arg => arg.startsWith('--strategy='))?.split('=')[1] || 'blue-green';
  const version = process.argv.find(arg => arg.startsWith('--version='))?.split('=')[1] || 'latest';

  const deployment = new DeploymentManager(env);

  if (strategy === 'blue-green') {
    await deployment.blueGreenDeploy({ version, cleanup: true });
  } else if (strategy === 'canary') {
    await deployment.canaryDeploy({ version, canaryPercentage: 10 });
  } else {
    console.error('Unknown strategy:', strategy);
    process.exit(1);
  }
}

if (require.main === module) {
  main().catch(error => {
    console.error(error);
    process.exit(1);
  });
}

module.exports = DeploymentManager;
```

---

## Tests Automatisés

### Tests d'Intégration avec Migrations

```javascript
// tests/integration/migrations.test.js
const { MongoClient } = require('mongodb');
const { Migrator } = require('migrate-mongo');

describe('Migration Integration Tests', () => {
  let client;
  let db;

  beforeAll(async () => {
    const uri = process.env.MONGODB_URI || 'mongodb://localhost:27017';
    client = await MongoClient.connect(uri);
    db = client.db('test_migrations');
  });

  afterAll(async () => {
    await db.dropDatabase();
    await client.close();
  });

  beforeEach(async () => {
    // Nettoyer la base avant chaque test
    const collections = await db.listCollections().toArray();
    for (const coll of collections) {
      await db.collection(coll.name).deleteMany({});
    }
  });

  describe('Migration: add-fullname-field', () => {
    it('should migrate firstName and lastName to fullName', async () => {
      // Arrange - Insérer des données avec l'ancien format
      await db.collection('users').insertMany([
        { firstName: 'John', lastName: 'Doe', migrationVersion: 1 },
        { firstName: 'Jane', lastName: 'Smith', migrationVersion: 1 }
      ]);

      // Act - Exécuter la migration
      const migration = require('../../migrations/20240115120000-add-fullname-field');
      await migration.up(db, client);

      // Assert
      const users = await db.collection('users').find({}).toArray();

      expect(users).toHaveLength(2);
      expect(users[0].fullName).toBe('John Doe');
      expect(users[0].migrationVersion).toBe(2);
      expect(users[1].fullName).toBe('Jane Smith');
    });

    it('should handle rollback correctly', async () => {
      // Arrange
      await db.collection('users').insertMany([
        { fullName: 'John Doe', migrationVersion: 2 }
      ]);

      // Act - Rollback
      const migration = require('../../migrations/20240115120000-add-fullname-field');
      await migration.down(db, client);

      // Assert
      const users = await db.collection('users').find({}).toArray();

      expect(users[0].firstName).toBe('John');
      expect(users[0].lastName).toBe('Doe');
      expect(users[0].migrationVersion).toBe(1);
      expect(users[0].fullName).toBeUndefined();
    });

    it('should be idempotent', async () => {
      // Arrange
      await db.collection('users').insertOne({
        firstName: 'John',
        lastName: 'Doe',
        migrationVersion: 1
      });

      // Act - Exécuter la migration deux fois
      const migration = require('../../migrations/20240115120000-add-fullname-field');
      await migration.up(db, client);
      await migration.up(db, client);

      // Assert - Doit rester cohérent
      const users = await db.collection('users').find({}).toArray();
      expect(users).toHaveLength(1);
      expect(users[0].fullName).toBe('John Doe');
    });
  });

  describe('Migration Performance', () => {
    it('should handle large datasets efficiently', async () => {
      // Arrange - Insérer beaucoup de documents
      const users = Array.from({ length: 10000 }, (_, i) => ({
        firstName: `User${i}`,
        lastName: `Test${i}`,
        migrationVersion: 1
      }));

      await db.collection('users').insertMany(users);

      // Act - Mesurer le temps d'exécution
      const startTime = Date.now();
      const migration = require('../../migrations/20240115120000-add-fullname-field');
      await migration.up(db, client);
      const duration = Date.now() - startTime;

      // Assert
      console.log(`Migration duration: ${duration}ms`);
      expect(duration).toBeLessThan(30000); // 30 seconds max

      const count = await db.collection('users').countDocuments({
        migrationVersion: 2
      });
      expect(count).toBe(10000);
    });
  });
});
```

---

## Best Practices

### Checklist CI/CD MongoDB

```yaml
# cicd-mongodb-best-practices.yaml
---
migrations:
  - ✅ Migrations versionnées et ordonnées
  - ✅ Up et down fonctions implémentées
  - ✅ Migrations idempotentes
  - ✅ Tests pour chaque migration
  - ✅ Dry-run avant production
  - ✅ Backup avant migration
  - ✅ Migration par batch pour gros volumes
  - ✅ Monitoring pendant migration

backward_compatibility:
  - ✅ Expand-contract pattern utilisé
  - ✅ Ancienne et nouvelle versions coexistent
  - ✅ Dual-write period définie
  - ✅ Rollback testé
  - ✅ Feature flags pour nouvelles fonctionnalités

testing:
  - ✅ Tests unitaires (> 80% coverage)
  - ✅ Tests d'intégration avec MongoDB
  - ✅ Tests de migration
  - ✅ Tests de performance
  - ✅ Smoke tests post-déploiement
  - ✅ Tests e2e en staging

deployment:
  - ✅ Blue-green ou canary deployment
  - ✅ Health checks configurés
  - ✅ Monitoring actif pendant déploiement
  - ✅ Rollback automatique en cas d'échec
  - ✅ Notifications (Slack, email)
  - ✅ Zero-downtime deployment

backup_and_recovery:
  - ✅ Backup automatique avant déploiement
  - ✅ Backup versionnés et datés
  - ✅ Backup stockés off-site (S3)
  - ✅ Restore testé régulièrement
  - ✅ Point-in-time recovery disponible
  - ✅ RPO/RTO documentés

security:
  - ✅ Secrets via variables d'environnement
  - ✅ Credentials jamais en clair
  - ✅ Dependency scanning automatique
  - ✅ Container scanning
  - ✅ RBAC sur pipelines
  - ✅ Audit logs activés

monitoring:
  - ✅ Métriques applicatives (latence, erreurs)
  - ✅ Métriques MongoDB (connexions, queries)
  - ✅ Alertes configurées
  - ✅ Dashboards disponibles
  - ✅ Logs centralisés
  - ✅ Tracing distribué

documentation:
  - ✅ README à jour
  - ✅ Runbooks pour incidents
  - ✅ Migration guide
  - ✅ Rollback procedures
  - ✅ Architecture diagrams
  - ✅ Contact d'escalation
```

---

## Conclusion

L'intégration de CI/CD avec les migrations de schéma MongoDB nécessite une approche méthodique pour garantir la fiabilité et la disponibilité :

**Principes clés :**
- **Backward compatibility** : Expand-contract pattern
- **Zero-downtime** : Blue-green/canary deployments
- **Safety** : Backup, dry-run, rollback automatique
- **Testing** : Tests à tous les niveaux
- **Monitoring** : Observabilité continue

**Stratégies de migration :**
- Expand and Contract (3 phases)
- Lazy/On-demand migration
- Batch migration pour gros volumes
- Versioning avec schéma

**Pipeline CI/CD :**
- Build, test, migrate, deploy
- Validation à chaque étape
- Automated rollback sur échec
- Notifications multi-canal

**Outils :**
- migrate-mongo pour migrations
- Jest pour tests
- Docker pour isolation
- Kubernetes pour orchestration

Une approche disciplinée du CI/CD et des migrations garantit des déploiements fiables et réversibles, minimisant les risques pour les applications MongoDB en production.

---


⏭️ [Blue/Green deployments](/18-devops-deploiement/09-blue-green-deployments.md)
