🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.8 Gestion des Environnements

## Introduction

La gestion rigoureuse des environnements est la pierre angulaire d'une application MongoDB professionnelle. Une séparation claire et stricte entre développement, staging et production n'est pas une option mais une nécessité absolue. Les incidents de production les plus catastrophiques résultent souvent d'une confusion d'environnements : une requête de test exécutée sur la production, des credentials partagés, ou des données de production utilisées en développement.

Cette section établit les pratiques essentielles pour maintenir une isolation complète entre environnements, garantir la reproductibilité des déploiements, et prévenir les erreurs coûteuses qui peuvent compromettre la sécurité, la conformité et la stabilité de votre système.

---

## Comprendre les Environnements

### Types d'Environnements Standard

```javascript
const environments = {
  local: {
    purpose: "Développement individuel",
    isolation: "Machine développeur",
    data: "Subset anonymisé ou synthétique",
    uptime: "Non critique"
  },

  development: {
    purpose: "Intégration continue",
    isolation: "Serveur partagé dev",
    data: "Données de test",
    uptime: "Heures de travail"
  },

  staging: {
    purpose: "Tests pré-production",
    isolation: "Infrastructure miroir prod",
    data: "Copie anonymisée de prod",
    uptime: "24/7 souhaité"
  },

  production: {
    purpose: "Utilisateurs finaux",
    isolation: "Infrastructure dédiée",
    data: "Données réelles",
    uptime: "99.9%+ requis"
  }
};
```

**Environnements optionnels** :
```javascript
const optionalEnvironments = {
  qa: {
    purpose: "Tests qualité manuels",
    between: "development et staging"
  },

  uat: {
    purpose: "Tests utilisateurs",
    between: "staging et production"
  },

  demo: {
    purpose: "Démonstrations clients",
    data: "Données fictives mais réalistes"
  },

  performance: {
    purpose: "Tests de charge",
    data: "Volume similaire à production"
  }
};
```

---

## ✅ DO : Maintenir une Isolation Stricte entre Environnements

**Explication** : Chaque environnement doit être complètement isolé des autres, avec ses propres ressources, credentials et données.

**Architecture d'isolation complète** :
```javascript
// ✅ Isolation par couche

// 1. Infrastructure séparée
const infrastructure = {
  production: {
    mongodbUrl: "mongodb+srv://prod-cluster.mongodb.net",
    region: "eu-west-1",
    instanceType: "M40",
    replicaSet: "prod-rs",
    backupSchedule: "hourly"
  },

  staging: {
    mongodbUrl: "mongodb+srv://staging-cluster.mongodb.net",
    region: "eu-west-1",
    instanceType: "M30",
    replicaSet: "staging-rs",
    backupSchedule: "daily"
  },

  development: {
    mongodbUrl: "mongodb+srv://dev-cluster.mongodb.net",
    region: "eu-west-1",
    instanceType: "M10",
    replicaSet: "dev-rs",
    backupSchedule: "weekly"
  }
};

// 2. Bases de données séparées
const databases = {
  production: {
    name: "production_app",
    users: "production_users",
    backups: "production_backups"
  },

  staging: {
    name: "staging_app",
    users: "staging_users",
    backups: "staging_backups"
  },

  development: {
    name: "dev_app",
    users: "dev_users",
    backups: "dev_backups"
  }
};

// 3. Credentials complètement différents
const credentials = {
  production: {
    username: "prod_app_user",
    password: process.env.PROD_DB_PASSWORD,  // Jamais en clair
    roles: ["readWrite", "dbAdmin"]
  },

  staging: {
    username: "staging_app_user",
    password: process.env.STAGING_DB_PASSWORD,
    roles: ["readWrite", "dbAdmin"]
  },

  development: {
    username: "dev_app_user",
    password: process.env.DEV_DB_PASSWORD,
    roles: ["readWrite", "dbAdmin", "dbOwner"]
  }
};

// 4. Réseau isolé
const networkSecurity = {
  production: {
    ipWhitelist: [
      "52.10.20.30/32",  // Production app servers
      "10.0.1.0/24"      // VPN production
    ],
    vpcPeering: "vpc-prod-123",
    privateEndpoint: true
  },

  staging: {
    ipWhitelist: [
      "52.20.30.40/32",  // Staging app servers
      "10.0.2.0/24"      // VPN staging
    ],
    vpcPeering: "vpc-staging-456",
    privateEndpoint: true
  },

  development: {
    ipWhitelist: [
      "0.0.0.0/0"  // Plus permissif pour dev (avec précautions)
    ],
    vpcPeering: "vpc-dev-789",
    privateEndpoint: false
  }
};
```

**Bénéfices de l'isolation** :

### 1. Prévention des Erreurs Catastrophiques
```javascript
// ❌ Sans isolation : Erreur possible
db.users.deleteMany({});  // Exécuté par erreur sur prod!
// Résultat : Tous les utilisateurs supprimés en production

// ✅ Avec isolation : Erreur impossible
// Credentials de dev ne fonctionnent pas sur prod
// Connection string différente
// IP dev non autorisée sur prod
```

### 2. Sécurité Renforcée
```javascript
// Si un environnement est compromis
// Les autres restent protégés

// Scénario : Credentials dev volés
// Impact : Aucun sur production (credentials différents)
// Impact : Aucun sur staging (réseau séparé)
```

### 3. Tests Sans Risque
```javascript
// Tests destructifs possibles en dev/staging
await db.dropDatabase();  // OK en dev
await testMigration();    // OK en staging

// Impossible en production (protections en place)
```

---

## ❌ DON'T : Utiliser les Mêmes Credentials dans Plusieurs Environnements

**Explication** : Partager des credentials entre environnements est une faille de sécurité majeure et une source d'erreurs.

**Anti-pattern dangereux** :
```javascript
// ❌ DANGER : Credentials partagés
const config = {
  production: {
    host: "prod-cluster.mongodb.net",
    username: "app_user",        // Même username
    password: "SharedPassword123"  // Même password ⚠️
  },

  staging: {
    host: "staging-cluster.mongodb.net",
    username: "app_user",        // Même username
    password: "SharedPassword123"  // Même password ⚠️
  }
};

// Conséquences catastrophiques :
// 1. Si credentials compromis, tous les environnements exposés
// 2. Rotation de password affecte tous les environnements
// 3. Impossible de tracer qui a accès à quoi
// 4. Erreur de configuration peut exposer production
```

**Problèmes concrets** :

### 1. Impact d'une Compromission
```javascript
// Scénario : Credentials volés
// Avec credentials partagés :
// - Attaquant accède à TOUS les environnements
// - Données de production compromises
// - Données de test compromises
// - Impossible de limiter les dégâts

// Avec credentials séparés :
// - Compromission limitée à un environnement
// - Production protégée si dev compromis
// - Rotation facile et ciblée
```

### 2. Erreurs de Configuration
```javascript
// ❌ Erreur facile avec credentials partagés
const config = {
  host: process.env.DB_HOST,        // Correct
  username: "app_user",              // Correct
  password: "SharedPassword123",     // Correct
  database: "production_app"         // ERREUR! Dev sur prod!
};

// Application dev se connecte à production
// avec les mêmes credentials
// = Catastrophe silencieuse
```

**Solution appropriée** :
```javascript
// ✅ Credentials uniques par environnement
const getConfig = (env) => {
  const configs = {
    production: {
      host: process.env.PROD_DB_HOST,
      username: process.env.PROD_DB_USER,
      password: process.env.PROD_DB_PASSWORD,
      database: "production_app",
      ssl: true,
      replicaSet: "prod-rs"
    },

    staging: {
      host: process.env.STAGING_DB_HOST,
      username: process.env.STAGING_DB_USER,
      password: process.env.STAGING_DB_PASSWORD,
      database: "staging_app",
      ssl: true,
      replicaSet: "staging-rs"
    },

    development: {
      host: process.env.DEV_DB_HOST,
      username: process.env.DEV_DB_USER,
      password: process.env.DEV_DB_PASSWORD,
      database: "dev_app",
      ssl: true,
      replicaSet: "dev-rs"
    }
  };

  return configs[env];
};

// Validation stricte
const config = getConfig(process.env.NODE_ENV);

if (!config) {
  throw new Error(`Invalid environment: ${process.env.NODE_ENV}`);
}

// Vérification supplémentaire
if (process.env.NODE_ENV === 'production' &&
    config.host.includes('dev')) {
  throw new Error('CRITICAL: Dev host in production config!');
}
```

---

## ✅ DO : Utiliser des Variables d'Environnement pour la Configuration

**Explication** : La configuration doit être externalisée dans des variables d'environnement, jamais en dur dans le code.

**Pattern recommandé** :
```javascript
// ✅ Configuration via variables d'environnement
// Fichier : .env.production
NODE_ENV=production
DB_HOST=prod-cluster.mongodb.net
DB_PORT=27017
DB_NAME=production_app
DB_USER=prod_app_user
DB_PASSWORD=<secret-from-vault>
DB_AUTH_SOURCE=admin
DB_SSL=true
DB_REPLICA_SET=prod-rs
DB_CONNECTION_TIMEOUT=30000
DB_MAX_POOL_SIZE=100

// Fichier : .env.staging
NODE_ENV=staging
DB_HOST=staging-cluster.mongodb.net
DB_PORT=27017
DB_NAME=staging_app
DB_USER=staging_app_user
DB_PASSWORD=<secret-from-vault>
DB_AUTH_SOURCE=admin
DB_SSL=true
DB_REPLICA_SET=staging-rs
DB_CONNECTION_TIMEOUT=30000
DB_MAX_POOL_SIZE=50

// Code application
require('dotenv').config();

const config = {
  mongodb: {
    url: buildMongoUrl(),
    options: {
      maxPoolSize: parseInt(process.env.DB_MAX_POOL_SIZE),
      serverSelectionTimeoutMS: parseInt(process.env.DB_CONNECTION_TIMEOUT),
      ssl: process.env.DB_SSL === 'true',
      replicaSet: process.env.DB_REPLICA_SET
    }
  }
};

function buildMongoUrl() {
  const {
    DB_USER,
    DB_PASSWORD,
    DB_HOST,
    DB_PORT,
    DB_NAME,
    DB_AUTH_SOURCE
  } = process.env;

  // Validation
  const required = ['DB_USER', 'DB_PASSWORD', 'DB_HOST', 'DB_NAME'];
  for (const key of required) {
    if (!process.env[key]) {
      throw new Error(`Missing required environment variable: ${key}`);
    }
  }

  return `mongodb://${DB_USER}:${DB_PASSWORD}@${DB_HOST}:${DB_PORT}/${DB_NAME}?authSource=${DB_AUTH_SOURCE}`;
}
```

**Avec gestionnaire de secrets** :
```javascript
// ✅ Intégration avec vault de secrets
const AWS = require('aws-sdk');

class ConfigManager {
  constructor(environment) {
    this.environment = environment;
    this.secretsManager = new AWS.SecretsManager({
      region: process.env.AWS_REGION
    });
  }

  async getConfig() {
    // Récupérer les secrets depuis AWS Secrets Manager
    const secretName = `${this.environment}/mongodb/credentials`;

    try {
      const data = await this.secretsManager
        .getSecretValue({ SecretId: secretName })
        .promise();

      const secrets = JSON.parse(data.SecretString);

      return {
        mongodb: {
          url: this.buildUrl(secrets),
          options: {
            maxPoolSize: secrets.maxPoolSize || 100,
            ssl: true,
            replicaSet: secrets.replicaSet
          }
        }
      };
    } catch (error) {
      console.error('Failed to retrieve secrets:', error);
      throw new Error('Configuration initialization failed');
    }
  }

  buildUrl(secrets) {
    const {
      username,
      password,
      host,
      database,
      port = 27017,
      authSource = 'admin'
    } = secrets;

    return `mongodb://${username}:${encodeURIComponent(password)}@${host}:${port}/${database}?authSource=${authSource}`;
  }
}

// Usage
const configManager = new ConfigManager(process.env.NODE_ENV);
const config = await configManager.getConfig();
```

---

## ❌ DON'T : Coder en Dur les Configurations dans le Code

**Explication** : Les configurations hardcodées rendent impossible le déploiement multi-environnement et exposent des informations sensibles.

**Anti-patterns** :
```javascript
// ❌ Configuration en dur dans le code
const mongoClient = new MongoClient(
  "mongodb://prod_user:P@ssw0rd123@prod-cluster.mongodb.net:27017/production_app",
  { ssl: true }
);

// Problèmes :
// 1. Credentials exposés dans le code source
// 2. Credentials dans Git (historique permanent)
// 3. Impossible de changer sans redéployer
// 4. Même code ne peut pas être déployé sur plusieurs environnements
// 5. Violation des audits de sécurité

// ❌ Configuration conditionnelle mais hardcodée
const config = process.env.NODE_ENV === 'production'
  ? {
      host: "prod-cluster.mongodb.net",
      username: "prod_user",
      password: "ProductionPassword123"  // Toujours en dur!
    }
  : {
      host: "dev-cluster.mongodb.net",
      username: "dev_user",
      password: "DevPassword123"  // Toujours en dur!
    };
```

**Conséquences mesurées** :
```javascript
// Impact sécurité
// Credentials dans Git =
// - Accessibles à tous les développeurs
// - Persistants dans l'historique (même si supprimés)
// - Potentiellement exposés si repo devient public
// - Violation RGPD/SOC2/ISO27001

// Impact opérationnel
// Changement de password =
// - Modifier le code
// - Commit
// - Review
// - Déploiement
// - Downtime potentiel
= Processus de 30 minutes au lieu de 30 secondes
```

---

## ✅ DO : Implémenter des Gardes de Sécurité contre les Erreurs

**Explication** : Des vérifications automatiques doivent prévenir l'exécution accidentelle de code dangereux en production.

**Gardes de sécurité** :
```javascript
// ✅ Vérifications de sécurité multiples
class EnvironmentGuard {
  constructor() {
    this.environment = process.env.NODE_ENV;
    this.isProduction = this.environment === 'production';
  }

  // Empêcher les opérations dangereuses en production
  assertNotProduction(operation) {
    if (this.isProduction) {
      throw new Error(
        `BLOCKED: Operation "${operation}" not allowed in production`
      );
    }
  }

  // Demander confirmation explicite pour opérations critiques
  requireExplicitConfirmation(operation) {
    if (this.isProduction) {
      const confirmation = process.env[`CONFIRM_${operation.toUpperCase()}`];

      if (confirmation !== 'YES_I_AM_SURE') {
        throw new Error(
          `BLOCKED: Operation "${operation}" requires explicit confirmation in production. ` +
          `Set CONFIRM_${operation.toUpperCase()}=YES_I_AM_SURE`
        );
      }
    }
  }

  // Vérifier que l'environnement est correct
  assertEnvironment(expected) {
    if (this.environment !== expected) {
      throw new Error(
        `BLOCKED: Operation expected ${expected} but running in ${this.environment}`
      );
    }
  }
}

const guard = new EnvironmentGuard();

// Usage
async function dropDatabase() {
  // Impossible en production
  guard.assertNotProduction('dropDatabase');

  await db.dropDatabase();
  console.log('Database dropped');
}

async function loadTestData() {
  // Seulement en dev
  guard.assertEnvironment('development');

  await db.collection('users').insertMany(testData);
}

async function runMigration() {
  // Demander confirmation en production
  guard.requireExplicitConfirmation('runMigration');

  await executeMigration();
}
```

**Protection au niveau base de données** :
```javascript
// ✅ Utilisateurs avec permissions limitées
const userPermissions = {
  production: {
    app_user: {
      roles: ['readWrite'],  // Pas de dropDatabase
      databases: ['production_app']
    },
    admin_user: {
      roles: ['dbOwner'],  // Tout mais nécessite MFA
      databases: ['production_app'],
      requireMFA: true
    }
  },

  development: {
    dev_user: {
      roles: ['root'],  // Tous les droits en dev
      databases: ['dev_app', 'dev_test']
    }
  }
};

// Protection avec validation de token
async function connectToProduction(credentials, mfaToken) {
  if (credentials.requireMFA && !mfaToken) {
    throw new Error('MFA token required for production access');
  }

  if (mfaToken) {
    const valid = await validateMFAToken(credentials.username, mfaToken);
    if (!valid) {
      throw new Error('Invalid MFA token');
    }
  }

  return await MongoClient.connect(buildUrl(credentials));
}
```

---

## ✅ DO : Anonymiser les Données pour les Environnements Non-Production

**Explication** : Les environnements de développement et staging ne doivent jamais contenir de données de production réelles.

**Processus d'anonymisation** :
```javascript
// ✅ Script d'anonymisation pour copie de production
class DataAnonymizer {
  async anonymizeCollection(sourceDb, targetDb, collectionName, rules) {
    console.log(`Anonymizing collection: ${collectionName}`);

    const cursor = sourceDb.collection(collectionName).find();
    const bulk = targetDb.collection(collectionName).initializeUnorderedBulkOp();

    let count = 0;

    await cursor.forEach(doc => {
      const anonymized = this.anonymizeDocument(doc, rules);
      bulk.insert(anonymized);
      count++;

      if (count % 1000 === 0) {
        console.log(`  Processed ${count} documents`);
      }
    });

    if (count > 0) {
      await bulk.execute();
    }

    console.log(`  Total: ${count} documents anonymized`);
  }

  anonymizeDocument(doc, rules) {
    const anonymized = { ...doc };

    for (const [field, rule] of Object.entries(rules)) {
      if (this.hasField(anonymized, field)) {
        anonymized[field] = this.applyRule(anonymized[field], rule);
      }
    }

    return anonymized;
  }

  applyRule(value, rule) {
    switch (rule.type) {
      case 'mask':
        return this.maskValue(value, rule.options);

      case 'fake':
        return this.generateFake(rule.dataType);

      case 'hash':
        return this.hashValue(value, rule.salt);

      case 'null':
        return null;

      case 'constant':
        return rule.value;

      default:
        return value;
    }
  }

  maskValue(value, options = {}) {
    if (typeof value !== 'string') return value;

    const { keepFirst = 0, keepLast = 0, maskChar = '*' } = options;

    if (value.length <= keepFirst + keepLast) {
      return maskChar.repeat(value.length);
    }

    const start = value.substring(0, keepFirst);
    const end = value.substring(value.length - keepLast);
    const middle = maskChar.repeat(value.length - keepFirst - keepLast);

    return start + middle + end;
  }

  generateFake(dataType) {
    const faker = require('faker');

    switch (dataType) {
      case 'email':
        return faker.internet.email();
      case 'name':
        return faker.name.findName();
      case 'phone':
        return faker.phone.phoneNumber();
      case 'address':
        return faker.address.streetAddress();
      case 'company':
        return faker.company.companyName();
      default:
        return faker.random.word();
    }
  }

  hashValue(value, salt) {
    const crypto = require('crypto');
    return crypto
      .createHmac('sha256', salt)
      .update(value.toString())
      .digest('hex');
  }

  hasField(obj, path) {
    const parts = path.split('.');
    let current = obj;

    for (const part of parts) {
      if (!current || !current.hasOwnProperty(part)) {
        return false;
      }
      current = current[part];
    }

    return true;
  }
}

// Configuration d'anonymisation
const anonymizationRules = {
  users: {
    email: { type: 'fake', dataType: 'email' },
    firstName: { type: 'fake', dataType: 'name' },
    lastName: { type: 'fake', dataType: 'name' },
    phone: { type: 'mask', options: { keepLast: 4 } },
    ssn: { type: 'hash', salt: process.env.ANONYMIZATION_SALT },
    address: { type: 'fake', dataType: 'address' },
    'payment.cardNumber': { type: 'mask', options: { keepLast: 4 } },
    'payment.cvv': { type: 'constant', value: '000' }
  },

  orders: {
    'customer.email': { type: 'fake', dataType: 'email' },
    'shipping.address': { type: 'fake', dataType: 'address' },
    'shipping.phone': { type: 'mask', options: { keepLast: 4 } }
  }
};

// Usage
async function refreshStagingData() {
  const prodClient = await MongoClient.connect(prodUrl);
  const stagingClient = await MongoClient.connect(stagingUrl);

  const prodDb = prodClient.db('production_app');
  const stagingDb = stagingClient.db('staging_app');

  const anonymizer = new DataAnonymizer();

  // Copier et anonymiser chaque collection
  for (const [collectionName, rules] of Object.entries(anonymizationRules)) {
    await anonymizer.anonymizeCollection(
      prodDb,
      stagingDb,
      collectionName,
      rules
    );
  }

  console.log('Staging data refresh completed');

  await prodClient.close();
  await stagingClient.close();
}
```

---

## ❌ DON'T : Utiliser des Données de Production en Développement

**Explication** : Utiliser des données réelles de production en développement est une violation majeure de sécurité et de conformité.

**Problèmes** :

### 1. Violations Légales et Réglementaires
```javascript
// RGPD Article 5(1)(f)
// "Les données doivent être traitées de manière à garantir
// une sécurité appropriée"

// Utiliser des données de production en dev =
// - Violation RGPD (amendes jusqu'à 4% du chiffre d'affaires)
// - Violation HIPAA (si données santé)
// - Violation PCI-DSS (si données bancaires)
// - Violation SOC 2

// Sanctions :
// - Amendes : Millions d'euros
// - Perte de certifications
// - Dommages réputationnels
// - Poursuites judiciaires
```

### 2. Risques de Sécurité
```javascript
// Développeur avec accès aux vraies données :
// - Emails clients réels exposés
// - Numéros de téléphone réels
// - Adresses réelles
// - Données financières réelles
// - Informations personnelles sensibles

// Risques :
// - Laptop développeur volé = fuite de données
// - Backup non chiffré = exposition
// - Développeur malveillant = vol de données
// - Logs avec données réelles = exposition
```

### 3. Risques de Corruption
```javascript
// Tests en dev sur données prod :
await db.users.updateMany(
  {},
  { $set: { email: 'test@example.com' } }
);
// Si exécuté par erreur sur prod = catastrophe

// Opérations destructives :
await db.collection.deleteMany({});
// En dev = OK
// Sur prod par erreur = désastre
```

**Solution appropriée** :
```javascript
// ✅ Données synthétiques pour développement
class TestDataGenerator {
  async generateUsers(count = 1000) {
    const faker = require('faker');
    const users = [];

    for (let i = 0; i < count; i++) {
      users.push({
        _id: new ObjectId(),
        email: faker.internet.email(),
        firstName: faker.name.firstName(),
        lastName: faker.name.lastName(),
        phone: faker.phone.phoneNumber(),
        address: {
          street: faker.address.streetAddress(),
          city: faker.address.city(),
          country: faker.address.country(),
          zipCode: faker.address.zipCode()
        },
        createdAt: faker.date.past(2),
        // Clairement marqué comme test
        __test_data: true
      });
    }

    return users;
  }

  async seedDatabase() {
    // S'assurer qu'on est en dev
    if (process.env.NODE_ENV === 'production') {
      throw new Error('Cannot seed database in production');
    }

    console.log('Generating test data...');

    const users = await this.generateUsers(1000);
    await db.users.insertMany(users);

    console.log('Test data generated successfully');
  }
}
```

---

## ✅ DO : Utiliser des Tags pour Identifier les Environnements

**Explication** : Marquer clairement chaque instance, connexion et log avec l'environnement facilite le monitoring et prévient les erreurs.

**Tagging complet** :
```javascript
// ✅ Tags sur toutes les couches
// 1. Tags MongoDB
const client = new MongoClient(mongoUrl, {
  appName: `myapp-${process.env.NODE_ENV}`,  // Visible dans MongoDB logs
  monitorCommands: true
});

// 2. Tags dans les logs
const logger = winston.createLogger({
  defaultMeta: {
    environment: process.env.NODE_ENV,
    service: 'myapp',
    host: os.hostname()
  },
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  )
});

// Chaque log inclut l'environnement
logger.info('Database connection established', {
  database: config.database,
  host: config.host
});

// 3. Tags dans le monitoring
const prometheus = require('prom-client');

const dbQueriesCounter = new prometheus.Counter({
  name: 'mongodb_queries_total',
  help: 'Total number of MongoDB queries',
  labelNames: ['environment', 'collection', 'operation']
});

dbQueriesCounter.inc({
  environment: process.env.NODE_ENV,
  collection: 'users',
  operation: 'find'
});

// 4. Tags dans les headers HTTP (pour API)
app.use((req, res, next) => {
  res.setHeader('X-Environment', process.env.NODE_ENV);
  next();
});

// 5. Tags visibles dans l'UI (non-production uniquement)
if (process.env.NODE_ENV !== 'production') {
  app.use((req, res, next) => {
    res.locals.environmentBanner = {
      text: `${process.env.NODE_ENV.toUpperCase()} ENVIRONMENT`,
      color: process.env.NODE_ENV === 'staging' ? 'orange' : 'red'
    };
    next();
  });
}
```

**Tags dans MongoDB Atlas** :
```javascript
// ✅ Tags au niveau cluster
const clusterTags = {
  production: {
    Environment: 'production',
    Criticality: 'high',
    BackupFrequency: 'hourly',
    MaintenanceWindow: 'sunday-02:00',
    CostCenter: 'engineering'
  },

  staging: {
    Environment: 'staging',
    Criticality: 'medium',
    BackupFrequency: 'daily',
    MaintenanceWindow: 'any',
    CostCenter: 'engineering'
  },

  development: {
    Environment: 'development',
    Criticality: 'low',
    BackupFrequency: 'weekly',
    MaintenanceWindow: 'any',
    CostCenter: 'engineering',
    AutoShutdown: 'true'  // Peut être éteint la nuit
  }
};
```

---

## ✅ DO : Implémenter des Déploiements Progressifs

**Explication** : Déployer progressivement à travers les environnements avec validation à chaque étape.

**Pipeline de déploiement** :
```javascript
// ✅ Pipeline de déploiement structuré
const deploymentPipeline = {
  stages: [
    {
      name: 'local',
      validation: ['unit-tests', 'linting'],
      manual: false
    },
    {
      name: 'development',
      validation: ['unit-tests', 'integration-tests'],
      manual: false,
      autoRollback: true
    },
    {
      name: 'staging',
      validation: [
        'integration-tests',
        'e2e-tests',
        'performance-tests',
        'security-scan'
      ],
      manual: true,  // Approbation requise
      smokeTests: true,
      canary: {
        enabled: true,
        percentage: 10,
        duration: '30m'
      }
    },
    {
      name: 'production',
      validation: [
        'staging-approval',
        'change-ticket',
        'rollback-plan'
      ],
      manual: true,
      requireApprovals: 2,
      canary: {
        enabled: true,
        percentage: 5,
        duration: '1h'
      },
      blueGreen: true,
      autoRollback: true,
      rollbackThreshold: {
        errorRate: 0.01,  // 1%
        latencyP99: 1000   // 1s
      }
    }
  ]
};

// Processus de déploiement
class DeploymentManager {
  async deploy(version, targetEnvironment) {
    const stage = deploymentPipeline.stages.find(
      s => s.name === targetEnvironment
    );

    if (!stage) {
      throw new Error(`Invalid environment: ${targetEnvironment}`);
    }

    console.log(`Starting deployment to ${targetEnvironment}`);

    // 1. Validations pré-déploiement
    await this.runValidations(stage.validation);

    // 2. Approbations manuelles si nécessaire
    if (stage.manual) {
      await this.requestApproval(targetEnvironment, stage.requireApprovals);
    }

    // 3. Backup pre-deployment
    await this.createBackup(targetEnvironment);

    // 4. Déploiement
    if (stage.canary?.enabled) {
      await this.canaryDeployment(version, targetEnvironment, stage.canary);
    } else if (stage.blueGreen) {
      await this.blueGreenDeployment(version, targetEnvironment);
    } else {
      await this.rollingDeployment(version, targetEnvironment);
    }

    // 5. Smoke tests post-déploiement
    if (stage.smokeTests) {
      await this.runSmokeTests(targetEnvironment);
    }

    // 6. Monitoring renforcé
    await this.enableEnhancedMonitoring(targetEnvironment, '1h');

    console.log(`Deployment to ${targetEnvironment} completed`);
  }

  async canaryDeployment(version, env, config) {
    console.log(`Canary deployment: ${config.percentage}% for ${config.duration}`);

    // Déployer sur un subset
    await this.deployToPercentage(version, env, config.percentage);

    // Monitorer pendant la durée canary
    const metrics = await this.monitorDeployment(env, config.duration);

    // Décider si continuer
    if (metrics.errorRate > 0.01) {
      console.error('Canary failed: error rate too high');
      await this.rollback(env);
      throw new Error('Canary deployment failed');
    }

    // Déployer sur le reste
    await this.deployToPercentage(version, env, 100);
  }

  async monitorDeployment(env, duration) {
    // Surveiller métriques clés
    const startTime = Date.now();
    const durationMs = this.parseDuration(duration);

    const metrics = {
      errorRate: 0,
      latencyP99: 0,
      throughput: 0
    };

    while (Date.now() - startTime < durationMs) {
      const current = await this.getMetrics(env);

      // Rollback automatique si seuils dépassés
      if (current.errorRate > 0.01) {
        console.error('Auto-rollback triggered: high error rate');
        await this.rollback(env);
        throw new Error('Deployment rolled back automatically');
      }

      // Mettre à jour métriques
      Object.assign(metrics, current);

      await this.sleep(60000);  // Check every minute
    }

    return metrics;
  }
}
```

---

## ✅ DO : Documenter les Différences entre Environnements

**Explication** : Maintenir une documentation claire des différences de configuration entre environnements.

**Documentation des environnements** :
```javascript
// ✅ Documentation structurée
const environmentDocumentation = {
  production: {
    description: "Production environment serving real users",

    infrastructure: {
      provider: "MongoDB Atlas",
      tier: "M40",
      region: "eu-west-1",
      replicaSet: "3 nodes",
      sharding: "enabled (2 shards)",
      backup: "Continuous backup + snapshots every 6h"
    },

    configuration: {
      connectionPoolSize: 100,
      connectionTimeout: 30000,
      queryTimeout: 60000,
      retryWrites: true,
      w: "majority",
      journal: true
    },

    security: {
      ssl: true,
      authentication: "SCRAM-SHA-256",
      encryption: "at-rest + in-transit",
      ipWhitelist: "VPC peering only",
      auditLog: "enabled"
    },

    monitoring: {
      alerting: "24/7 PagerDuty",
      logging: "CloudWatch + Datadog",
      apm: "enabled",
      errorTracking: "Sentry"
    },

    access: {
      developers: "Read-only via VPN + MFA",
      devops: "Admin via VPN + MFA + approval",
      automated: "Service accounts with rotation"
    },

    dataRetention: {
      logs: "90 days",
      backups: "90 days",
      auditLogs: "7 years"
    }
  },

  staging: {
    description: "Pre-production testing environment",

    infrastructure: {
      provider: "MongoDB Atlas",
      tier: "M30",
      region: "eu-west-1",
      replicaSet: "3 nodes",
      sharding: "disabled",
      backup: "Daily snapshots"
    },

    // Différences par rapport à production
    differences: [
      "Smaller instance size (M30 vs M40)",
      "No sharding (simpler setup for testing)",
      "Less frequent backups (daily vs continuous)",
      "Anonymized data from production",
      "Lower connection pool (50 vs 100)",
      "More permissive access control"
    ],

    refreshSchedule: {
      data: "Weekly from production (anonymized)",
      schema: "After each production deployment"
    }
  },

  development: {
    description: "Development and testing environment",

    infrastructure: {
      provider: "MongoDB Atlas",
      tier: "M10",
      region: "eu-west-1",
      replicaSet: "Single node",
      sharding: "disabled",
      backup: "None"
    },

    // Simplifications par rapport à production
    simplifications: [
      "Single node (vs replica set)",
      "Smaller instance",
      "No backups",
      "Test data only",
      "Permissive access",
      "Can be reset anytime"
    ]
  }
};
```

---

## Checklist Gestion des Environnements

### Isolation
- [ ] Instances MongoDB séparées par environnement
- [ ] Bases de données distinctes
- [ ] Credentials uniques par environnement
- [ ] Réseau isolé (VPC/IP whitelisting)
- [ ] Pas de partage de ressources entre envs

### Configuration
- [ ] Variables d'environnement pour toute config
- [ ] Secrets dans vault (pas en clair)
- [ ] Configuration versionée (Git)
- [ ] Pas de hardcoding dans le code
- [ ] Validation des configs au démarrage

### Données
- [ ] Jamais de données prod en dev/staging
- [ ] Données anonymisées pour non-prod
- [ ] Processus de refresh documenté
- [ ] Marqueurs clairs (\_\_test_data)
- [ ] Conformité RGPD/réglementaire

### Sécurité
- [ ] Gardes contre opérations dangereuses
- [ ] MFA pour accès production
- [ ] Audit logging activé
- [ ] Chiffrement en transit (SSL/TLS)
- [ ] Chiffrement au repos (production)

### Déploiement
- [ ] Pipeline structuré (dev → staging → prod)
- [ ] Validation à chaque étape
- [ ] Approbations manuelles pour prod
- [ ] Tests automatisés
- [ ] Plan de rollback documenté

### Monitoring
- [ ] Tags environnement sur toutes les ressources
- [ ] Logs avec identification env
- [ ] Alertes différenciées par env
- [ ] Métriques séparées
- [ ] Dashboard par environnement

### Documentation
- [ ] Différences entre envs documentées
- [ ] Procédures d'accès claires
- [ ] Contacts et escalation définis
- [ ] Architecture à jour
- [ ] Runbooks disponibles

---

## Conclusion

La gestion rigoureuse des environnements est non négociable pour :

- **Sécurité** : Protection des données production
- **Conformité** : Respect RGPD, SOC2, ISO27001
- **Stabilité** : Prévention des erreurs catastrophiques
- **Efficacité** : Tests sans risque
- **Professionnalisme** : Standards de l'industrie

**Règles d'or** :
1. **Isolation stricte** : Zéro partage entre environnements
2. **Credentials uniques** : Un par environnement
3. **Variables d'environnement** : Jamais de hardcoding
4. **Données anonymisées** : Jamais de prod en dev
5. **Pipeline structuré** : Validation progressive
6. **Documentation complète** : Traçabilité et clarté

Une erreur d'environnement peut détruire en secondes ce qui a pris des mois à construire. L'investissement dans une gestion rigoureuse est infiniment rentable.

---


⏭️ [Documentation et commentaires](/21-bonnes-pratiques-anti-patterns/09-documentation-commentaires.md)
