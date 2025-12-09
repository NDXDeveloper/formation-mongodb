🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.9 Documentation et Commentaires

## Introduction

"Code is read more often than it is written" - cette vérité est encore plus critique avec MongoDB où la flexibilité du schéma peut rapidement devenir une source de confusion. Une base de données MongoDB sans documentation est une bombe à retardement : les décisions de modélisation deviennent opaques, les index mystérieux, les migrations incompréhensibles, et l'onboarding de nouveaux développeurs un cauchemar.

La documentation n'est pas un luxe ou une corvée à faire "plus tard" - c'est un investissement qui se rentabilise dès le premier jour. Une bonne documentation réduit les bugs, accélère le développement, facilite la maintenance et permet à l'équipe de scaler. Cette section établit les standards de documentation pour un projet MongoDB professionnel.

---

## Comprendre l'Importance de la Documentation

### Impact Mesuré de la Documentation

```javascript
// Étude sur projet MongoDB de 2 ans sans documentation adéquate
const documentationImpact = {
  onboarding: {
    sansDocs: "2-3 semaines",
    avecDocs: "3-5 jours",
    gain: "70% temps économisé"
  },

  maintenance: {
    sansDocs: "30 min pour comprendre + 10 min pour modifier",
    avecDocs: "5 min pour comprendre + 10 min pour modifier",
    gain: "62% temps économisé"
  },

  bugs: {
    sansDocs: "25% causés par incompréhension du schéma",
    avecDocs: "5% causés par incompréhension",
    reduction: "80% bugs évités"
  },

  decisions: {
    sansDocs: "Décisions répétées, patterns incohérents",
    avecDocs: "Référence aux décisions passées, cohérence",
    benefit: "Architecture cohérente"
  }
};
```

### Coût de l'Absence de Documentation

```javascript
// Scénario réel : 10 développeurs sur 1 an
const costs = {
  tempsPerdu: {
    parDevParJour: "30 minutes à chercher de l'info",
    parAn: "10 devs × 250 jours × 0.5h = 1,250 heures",
    coutHoraire: 80,  // €/h
    total: "100,000 € de temps perdu"
  },

  bugsIncomprehension: {
    estimation: "5 bugs majeurs/an liés à incompréhension",
    tempsDebug: "40 heures par bug",
    total: "16,000 € de coûts bugs"
  },

  turnover: {
    frustrationDev: "Documentation absente = frustration",
    impactRetention: "Augmente le turnover de 20-30%",
    coutRemplacement: "50,000-100,000 € par développeur"
  }
};

// Total : ~120,000 € par an de coûts directs
// vs ~20,000 € pour maintenir documentation de qualité
// ROI : 500%
```

---

## ✅ DO : Documenter le Schéma de Chaque Collection

**Explication** : Chaque collection doit avoir une documentation claire de sa structure, ses champs, leurs types et leur signification.

**Documentation de schéma complète** :
```javascript
/**
 * Collection: users
 *
 * Description:
 *   Stores user account information and authentication data.
 *   Each document represents a single user account in the system.
 *
 * Indexes:
 *   - email_1 (unique): Fast lookup by email for authentication
 *   - username_1 (unique): Fast lookup by username for profile pages
 *   - createdAt_-1: Recent users listing
 *   - "roles.type_1": Filter users by role
 *
 * Schema Version: 3 (updated: 2024-01-15)
 * Last Migration: 2024-01-15 - Split name into firstName/lastName
 *
 * Related Collections:
 *   - orders: users._id → orders.userId
 *   - sessions: users._id → sessions.userId
 *   - preferences: users._id → preferences.userId
 *
 * Data Retention:
 *   - Active users: Indefinite
 *   - Deleted users: Soft delete, 90 days retention
 *
 * Schema Definition:
 */
const userSchema = {
  // Unique identifier (MongoDB ObjectId)
  _id: {
    type: "ObjectId",
    required: true,
    description: "Unique user identifier, auto-generated"
  },

  // Schema version for migration tracking
  schemaVersion: {
    type: "number",
    required: true,
    default: 3,
    description: "Schema version, used for progressive migrations"
  },

  // Authentication & Identity
  email: {
    type: "string",
    required: true,
    unique: true,
    format: "email",
    description: "User email address, used for authentication",
    example: "alice@example.com",
    validation: "Must be valid email format"
  },

  username: {
    type: "string",
    required: true,
    unique: true,
    minLength: 3,
    maxLength: 30,
    pattern: "^[a-zA-Z0-9_]+$",
    description: "Unique username for public profile",
    example: "alice_smith"
  },

  passwordHash: {
    type: "string",
    required: true,
    description: "Bcrypt hash of user password (never store plaintext)",
    security: "Hashed with bcrypt, 10 rounds"
  },

  // Personal Information
  firstName: {
    type: "string",
    required: true,
    description: "User first name",
    example: "Alice",
    notes: "Split from 'name' field in schema v2"
  },

  lastName: {
    type: "string",
    required: true,
    description: "User last name",
    example: "Smith"
  },

  birthDate: {
    type: "Date",
    required: false,
    description: "User date of birth (optional)",
    usage: "Used for age verification, age calculation",
    notes: "Added in schema v2, calculated from age field"
  },

  // Contact Information
  phone: {
    type: "string",
    required: false,
    format: "E.164",
    description: "Phone number in international format",
    example: "+33612345678",
    validation: "Must start with + and country code"
  },

  // Account Status
  status: {
    type: "string",
    required: true,
    enum: ["active", "suspended", "pending_verification", "deleted"],
    default: "pending_verification",
    description: "Current account status",
    transitions: {
      "pending_verification → active": "Email verified",
      "active → suspended": "Admin action or policy violation",
      "* → deleted": "User deletion request"
    }
  },

  emailVerified: {
    type: "boolean",
    required: true,
    default: false,
    description: "Whether user has verified their email",
    notes: "Must be true before status can be 'active'"
  },

  // Roles & Permissions
  roles: {
    type: "array",
    required: true,
    default: [{ type: "user", grantedAt: "Date.now()" }],
    description: "User roles for authorization",
    items: {
      type: {
        type: "string",
        enum: ["user", "admin", "moderator", "premium"],
        description: "Role type"
      },
      grantedAt: {
        type: "Date",
        description: "When this role was granted"
      },
      grantedBy: {
        type: "string",
        description: "Who granted this role (userId or system)"
      }
    }
  },

  // Timestamps
  createdAt: {
    type: "Date",
    required: true,
    immutable: true,
    description: "Account creation timestamp",
    notes: "Set once at creation, never modified"
  },

  updatedAt: {
    type: "Date",
    required: true,
    description: "Last profile update timestamp",
    notes: "Updated on any document modification"
  },

  lastLoginAt: {
    type: "Date",
    required: false,
    description: "Last successful login timestamp",
    notes: "Null if user never logged in"
  },

  // Soft Delete
  deletedAt: {
    type: "Date",
    required: false,
    description: "Soft delete timestamp",
    notes: "Present if user deleted their account, null otherwise"
  },

  // Metadata
  metadata: {
    type: "object",
    required: false,
    description: "Additional metadata, flexible structure",
    example: {
      signupSource: "web",
      referralCode: "FRIEND123",
      locale: "en-US"
    }
  }
};

/**
 * Business Rules:
 *   - Email must be unique across all users (including deleted)
 *   - Username can be reused after 90 days of account deletion
 *   - Phone verification required for premium accounts
 *   - Users with 'deleted' status are kept for 90 days then hard-deleted
 *
 * Performance Notes:
 *   - Average document size: ~2 KB
 *   - 95th percentile read time: <5ms (indexed queries)
 *   - Write frequency: ~1 update per user per day
 *
 * Security:
 *   - passwordHash never returned in API responses
 *   - Email only visible to user themselves and admins
 *   - Phone only visible to user themselves
 *
 * Known Issues:
 *   - TODO: Add email change history for audit
 *   - TODO: Add password change history
 */
```

**Où documenter** :
```javascript
// ✅ Multiple emplacements pour accessibilité

// 1. Dans le code (proche du modèle)
// models/User.js
/**
 * User Model
 * See: docs/schema/users.md for complete schema documentation
 */

// 2. Documentation dédiée
// docs/schema/users.md
// (Documentation complète ci-dessus)

// 3. README de la collection (MongoDB Compass)
// Via MongoDB Compass ou script

// 4. Schema validation dans MongoDB
db.createCollection("users", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["email", "username", "passwordHash"],
      properties: {
        email: {
          bsonType: "string",
          description: "must be a string and is required"
        }
        // ... (validation complète)
      }
    }
  },
  validationLevel: "moderate",
  validationAction: "warn"
});
```

---

## ❌ DON'T : Laisser le Schéma Implicite et Non Documenté

**Explication** : Un schéma non documenté force chaque développeur à deviner la structure, causant incohérences et bugs.

**Anti-pattern** :
```javascript
// ❌ Aucune documentation
db.users.insertOne({
  email: "alice@example.com",
  pwd: "hashed_password",
  nm: "Alice",
  st: "a",
  cr: new Date(),
  roles: ["user"]
});

// Questions sans réponse :
// - Quel est le format du champ 'pwd' ? Hash algorithm?
// - 'nm' c'est quoi ? name? firstName? fullName?
// - 'st' = status? Quelles valeurs possibles? 'a' = active?
// - 'cr' = created? createdAt? creation?
// - roles est un tableau de quoi? strings? objects?
// - Quels champs sont obligatoires?
// - Y a-t-il des contraintes de validation?
```

**Conséquences** :

### 1. Incohérences de Données
```javascript
// Développeur A insère :
{ status: "active", roles: ["user"] }

// Développeur B insère :
{ status: "Active", roles: [{ type: "user" }] }

// Développeur C insère :
{ status: 1, roles: "user" }

// = Données incohérentes, queries qui échouent, bugs
```

### 2. Bugs de Validation
```javascript
// Sans documentation claire :
const user = await db.users.findOne({ email: email });

// Est-ce que user.status peut être null?
// Est-ce que user.roles existe toujours?
// Est-ce que user.email est normalisé (lowercase)?

if (user.status === "active") {  // Crash si status est undefined
  // ...
}
```

### 3. Temps Perdu
```javascript
// Chaque développeur doit :
// 1. Examiner des documents existants
// 2. Chercher dans le code
// 3. Demander aux collègues
// 4. Deviner et espérer

// 30 minutes perdues × 10 devs × 100 fois/an
// = 500 heures perdues = 40,000 €
```

---

## ✅ DO : Documenter les Index et Leur Justification

**Explication** : Chaque index doit avoir une documentation expliquant pourquoi il existe et quelles requêtes il optimise.

**Documentation d'index complète** :
```javascript
/**
 * Index Documentation - users collection
 *
 * Naming Convention: fieldName_direction_properties
 * Example: email_1_unique = field 'email', ascending (1), unique
 */

const userIndexes = {
  // Primary key (automatic)
  _id_: {
    type: "single",
    fields: { _id: 1 },
    unique: true,
    automatic: true,
    purpose: "Primary key, automatic MongoDB index",
    queries: "findById, updateById, deleteById",
    usage: "100% of single-document operations"
  },

  // Authentication index
  email_1_unique: {
    type: "single",
    fields: { email: 1 },
    unique: true,
    sparse: false,
    created: "2023-01-15",
    purpose: "Fast user lookup during authentication",
    queries: [
      "findOne({ email: email })",  // Login
      "countDocuments({ email: email })"  // Email existence check
    ],
    usage: "Every login attempt (~10,000/day)",
    performance: {
      without: "~500ms (collection scan)",
      with: "~2ms (index scan)",
      improvement: "250x faster"
    },
    size: "~15 MB",
    maintenance: "Updated on user creation/email change"
  },

  // Profile lookup index
  username_1_unique: {
    type: "single",
    fields: { username: 1 },
    unique: true,
    sparse: false,
    created: "2023-01-15",
    purpose: "Fast user profile lookup by username",
    queries: [
      "findOne({ username: username })",  // Profile page
      "find({ username: { $in: usernames } })"  // Bulk lookup
    ],
    usage: "Profile page views (~5,000/day)",
    performance: {
      without: "~300ms",
      with: "~1ms",
      improvement: "300x faster"
    }
  },

  // Recent users listing
  createdAt_-1: {
    type: "single",
    fields: { createdAt: -1 },
    unique: false,
    sparse: false,
    created: "2023-02-20",
    purpose: "List recently registered users (admin dashboard)",
    queries: [
      "find({}).sort({ createdAt: -1 }).limit(100)"
    ],
    usage: "Admin dashboard (~50/day)",
    performance: {
      without: "~1000ms (sort in memory)",
      with: "~10ms (sorted index scan)",
      improvement: "100x faster"
    },
    notes: "Also supports date range queries efficiently"
  },

  // Role-based queries
  roles_type_1: {
    type: "multikey",
    fields: { "roles.type": 1 },
    unique: false,
    sparse: false,
    created: "2023-03-10",
    purpose: "Filter users by role (admin queries)",
    queries: [
      "find({ 'roles.type': 'admin' })",
      "countDocuments({ 'roles.type': { $in: ['admin', 'moderator'] } })"
    ],
    usage: "Admin user management (~100/day)",
    performance: {
      without: "~800ms",
      with: "~15ms",
      improvement: "50x faster"
    },
    notes: "Multikey index because roles is an array"
  },

  // Compound index for filtered listing
  status_1_createdAt_-1: {
    type: "compound",
    fields: { status: 1, createdAt: -1 },
    unique: false,
    sparse: false,
    created: "2023-06-15",
    purpose: "List users by status, sorted by creation date",
    queries: [
      "find({ status: 'active' }).sort({ createdAt: -1 })",
      "find({ status: 'pending_verification' }).sort({ createdAt: -1 })"
    ],
    usage: "Admin dashboard, user management (~200/day)",
    performance: {
      without: "~600ms (index + sort)",
      with: "~8ms (covered query)",
      improvement: "75x faster"
    },
    notes: "Covers both filter and sort, very efficient",
    esrRule: "Equality (status) then Sort (createdAt)"
  },

  // Soft delete cleanup
  deletedAt_1: {
    type: "single",
    fields: { deletedAt: 1 },
    unique: false,
    sparse: true,  // Only indexes documents with deletedAt field
    partialFilterExpression: { deletedAt: { $exists: true } },
    created: "2023-08-01",
    purpose: "Efficient cleanup of old soft-deleted users",
    queries: [
      "find({ deletedAt: { $lt: ninetyDaysAgo } })",
      "deleteMany({ deletedAt: { $lt: ninetyDaysAgo } })"
    ],
    usage: "Nightly cleanup job (1/day)",
    performance: {
      without: "~5000ms (full scan)",
      with: "~50ms (sparse index)",
      improvement: "100x faster"
    },
    notes: "Sparse index saves space (only ~1% of users deleted)",
    size: "~200 KB (vs 15 MB if not sparse)"
  }
};

/**
 * Index Strategy Summary:
 *
 * Total Indexes: 7 (including _id)
 * Total Size: ~35 MB
 * Write Overhead: ~10-15% per insert/update
 *
 * Review Schedule:
 *   - Monthly: Check usage with $indexStats
 *   - Quarterly: Review slow queries for missing indexes
 *   - Annually: Full index audit
 *
 * Index Naming Convention:
 *   - Single: fieldName_direction[_properties]
 *   - Compound: field1_dir1_field2_dir2[_properties]
 *   - Properties: unique, sparse, partial, ttl, text, geo
 *
 * Unused Index Detection:
 *   db.users.aggregate([{ $indexStats: {} }])
 *   If ops: 0 for 30+ days → candidate for removal
 */
```

**Monitoring d'utilisation** :
```javascript
// ✅ Script pour vérifier l'utilisation des index
async function auditIndexUsage(collectionName) {
  const stats = await db[collectionName].aggregate([
    { $indexStats: {} }
  ]).toArray();

  console.log(`\n=== Index Usage Report: ${collectionName} ===\n`);

  for (const indexStat of stats) {
    const { name, key, accesses } = indexStat;

    console.log(`Index: ${name}`);
    console.log(`  Fields: ${JSON.stringify(key)}`);
    console.log(`  Accesses: ${accesses.ops}`);
    console.log(`  Since: ${accesses.since}`);

    // Warning for unused indexes
    if (accesses.ops === 0) {
      console.log(`  ⚠️  WARNING: Never used! Consider removing.`);
    }

    console.log('');
  }
}
```

---

## ✅ DO : Documenter les Requêtes et Agrégations Complexes

**Explication** : Les requêtes et pipelines d'agrégation complexes doivent être documentés inline avec des explications claires.

**Documentation de requêtes complexes** :
```javascript
// ✅ Requête complexe bien documentée
/**
 * Get User Activity Summary
 *
 * Purpose:
 *   Generate a comprehensive activity summary for a user including
 *   their orders, reviews, and engagement metrics.
 *
 * Performance:
 *   - Uses compound index on orders: userId_1_createdAt_-1
 *   - Average execution time: ~25ms
 *   - Scales linearly with user's order count
 *
 * Parameters:
 *   @param {ObjectId} userId - User ID to summarize
 *   @param {Date} startDate - Start of date range (optional)
 *   @param {Date} endDate - End of date range (optional)
 *
 * Returns:
 *   {
 *     userId: ObjectId,
 *     totalOrders: number,
 *     totalSpent: number,
 *     averageOrderValue: number,
 *     reviewsCount: number,
 *     lastOrderDate: Date,
 *     favoriteCategory: string
 *   }
 *
 * Related Collections:
 *   - orders: Main data source
 *   - products: For category enrichment
 *   - reviews: For review count
 */
async function getUserActivitySummary(userId, startDate, endDate) {
  // Build date filter if provided
  const dateFilter = {};
  if (startDate || endDate) {
    dateFilter.createdAt = {};
    if (startDate) dateFilter.createdAt.$gte = startDate;
    if (endDate) dateFilter.createdAt.$lte = endDate;
  }

  // Main aggregation pipeline
  const pipeline = [
    // Stage 1: Filter orders for this user (uses index: userId_1_createdAt_-1)
    {
      $match: {
        userId: userId,
        status: { $in: ['completed', 'shipped'] },  // Only successful orders
        ...dateFilter
      }
    },

    // Stage 2: Unwind items array to process each product
    // Note: This may produce many documents if user has large orders
    {
      $unwind: {
        path: '$items',
        preserveNullAndEmptyArrays: false
      }
    },

    // Stage 3: Lookup product details for category information
    // Uses index on products: _id (automatic)
    {
      $lookup: {
        from: 'products',
        localField: 'items.productId',
        foreignField: '_id',
        as: 'productDetails'
      }
    },

    // Stage 4: Unwind product details (should always be 1 match)
    {
      $unwind: {
        path: '$productDetails',
        preserveNullAndEmptyArrays: true  // Handle deleted products
      }
    },

    // Stage 5: Group by user to aggregate statistics
    {
      $group: {
        _id: '$userId',

        // Count metrics
        totalOrders: { $addToSet: '$_id' },  // Unique order IDs
        totalItems: { $sum: '$items.quantity' },

        // Financial metrics
        totalSpent: { $sum: '$total' },

        // Category analysis
        categories: { $push: '$productDetails.category' },

        // Temporal metrics
        orderDates: { $push: '$createdAt' }
      }
    },

    // Stage 6: Calculate derived fields
    {
      $project: {
        _id: 0,
        userId: '$_id',

        // Order statistics
        totalOrders: { $size: '$totalOrders' },
        totalItems: 1,
        totalSpent: 1,
        averageOrderValue: {
          $divide: ['$totalSpent', { $size: '$totalOrders' }]
        },

        // Category analysis - find most frequent
        favoriteCategory: {
          $arrayElemAt: [
            {
              $reduce: {
                input: '$categories',
                initialValue: { category: null, count: 0 },
                in: {
                  $cond: [
                    { $gt: [
                      { $size: { $filter: {
                        input: '$categories',
                        cond: { $eq: ['$$this', '$$value.category'] }
                      }}},
                      '$$value.count'
                    ]},
                    {
                      category: '$$this',
                      count: { $size: { $filter: {
                        input: '$categories',
                        cond: { $eq: ['$$this', '$$value.category'] }
                      }}}
                    },
                    '$$value'
                  ]
                }
              }
            },
            0
          ]
        },

        // Last order date
        lastOrderDate: { $max: '$orderDates' }
      }
    },

    // Stage 7: Lookup review count (separate query for simplicity)
    {
      $lookup: {
        from: 'reviews',
        let: { userId: '$userId' },
        pipeline: [
          { $match: { $expr: { $eq: ['$userId', '$$userId'] } } },
          { $count: 'count' }
        ],
        as: 'reviewStats'
      }
    },

    // Stage 8: Final projection
    {
      $project: {
        userId: 1,
        totalOrders: 1,
        totalItems: 1,
        totalSpent: 1,
        averageOrderValue: { $round: ['$averageOrderValue', 2] },
        favoriteCategory: '$favoriteCategory.category',
        lastOrderDate: 1,
        reviewsCount: {
          $ifNull: [
            { $arrayElemAt: ['$reviewStats.count', 0] },
            0
          ]
        }
      }
    }
  ];

  // Execute with explain for performance monitoring
  const result = await db.orders.aggregate(pipeline).toArray();

  // Return single result or null
  return result[0] || null;
}

/**
 * Performance Optimization Notes:
 *
 * 1. Stage 1 uses index efficiently (userId + date range)
 * 2. Stage 3 lookup is fast due to _id index on products
 * 3. Bottleneck: Stage 2 unwind if user has many orders with many items
 *
 * Optimization Strategies:
 *   - If user has >1000 orders: Consider caching or pre-aggregation
 *   - If real-time not needed: Use materialized view updated nightly
 *   - For high-volume users: Implement bucket pattern
 *
 * Known Limitations:
 *   - Does not handle cancelled/refunded orders
 *   - Deleted products show as null in favoriteCategory
 *   - Performance degrades for users with >10k orders
 *
 * Testing:
 *   - Unit test with mock data: tests/unit/userActivity.test.js
 *   - Performance test with 10k orders: tests/perf/userActivity.perf.js
 *
 * Change History:
 *   - 2024-01-10: Initial implementation
 *   - 2024-02-15: Added favoriteCategory calculation
 *   - 2024-03-01: Optimized with compound index
 */
```

---

## ❌ DON'T : Écrire des Requêtes Complexes Sans Explication

**Explication** : Une requête d'agrégation de 50 lignes sans commentaires est incompréhensible et impossible à maintenir.

**Anti-pattern** :
```javascript
// ❌ Pipeline complexe sans documentation
const result = await db.orders.aggregate([
  { $match: { userId: userId, status: { $in: ['completed', 'shipped'] } } },
  { $unwind: '$items' },
  { $lookup: { from: 'products', localField: 'items.productId', foreignField: '_id', as: 'p' } },
  { $unwind: '$p' },
  { $group: { _id: '$userId', t: { $addToSet: '$_id' }, ti: { $sum: '$items.quantity' },
    ts: { $sum: '$total' }, c: { $push: '$p.category' }, d: { $push: '$createdAt' } } },
  { $project: { _id: 0, userId: '$_id', totalOrders: { $size: '$t' }, totalItems: '$ti',
    totalSpent: '$ts', avgOrder: { $divide: ['$ts', { $size: '$t' }] },
    favCat: { $arrayElemAt: [{ $reduce: { input: '$c', initialValue: { c: null, cnt: 0 },
    in: { $cond: [{ $gt: [{ $size: { $filter: { input: '$c', cond: { $eq: ['$$this', '$$value.c'] } }}}, '$$value.cnt'] },
    { c: '$$this', cnt: { $size: { $filter: { input: '$c', cond: { $eq: ['$$this', '$$value.c'] } }}}}, '$$value'] }}}, 0] },
    lastOrder: { $max: '$d' } } }
]).toArray();

// Questions sans réponse :
// - Que calcule ce pipeline exactement?
// - Pourquoi ces stages spécifiques?
// - Quels index sont utilisés?
// - Quelle est la performance attendue?
// - Quelles sont les limitations?
// - Que signifient 't', 'ti', 'ts', 'c', 'd', 'favCat'?
```

**Conséquences** :
- Impossible à maintenir
- Bugs lors des modifications
- Personne n'ose toucher le code
- Duplication plutôt que réutilisation
- Temps perdu à déchiffrer

---

## ✅ DO : Documenter les Décisions Architecturales (ADR)

**Explication** : Les décisions importantes de conception doivent être documentées avec leur contexte et leurs alternatives.

**Architecture Decision Record (ADR)** :
```markdown
# ADR-003: Utilisation du Pattern Embedded pour les Commentaires d'Articles

## Statut
Accepté (2024-01-15)

## Contexte
Les articles de blog peuvent recevoir des commentaires des utilisateurs.
Nous devons décider comment modéliser cette relation.

### Données du Problème
- Volume articles : ~10,000
- Commentaires moyens par article : 15
- Commentaires max observé : 250
- 95% des articles ont < 50 commentaires
- 90% des lectures d'articles incluent la lecture des commentaires
- Les commentaires sont rarement modifiés après création
- Pas de recherche cross-article sur les commentaires

### Contraintes
- Les pages d'article doivent charger en < 200ms
- Pas de budget pour infrastructure complexe
- Équipe de 4 développeurs (expérience MongoDB moyenne)

## Décision
Nous utilisons le **pattern Embedded** avec limite de 50 commentaires
les plus récents imbriqués dans le document article.

### Structure Choisie
```javascript
// Collection: articles
{
  _id: ObjectId("..."),
  title: "Article Title",
  content: "...",
  author: "...",

  // 50 commentaires les plus récents imbriqués
  recentComments: [
    {
      _id: ObjectId("..."),
      userId: ObjectId("..."),
      username: "alice",
      text: "Great article!",
      createdAt: ISODate("...")
    },
    // ... jusqu'à 50 commentaires
  ],

  // Métadonnées
  commentCount: 127,
  lastCommentDate: ISODate("...")
}

// Collection: article_comments (pour historique complet si >50)
{
  _id: ObjectId("..."),
  articleId: ObjectId("..."),
  userId: ObjectId("..."),
  username: "bob",
  text: "Comment text",
  createdAt: ISODate("...")
}
```

## Alternatives Considérées

### Alternative 1: Références (Collection Séparée)
```javascript
// Collection articles
{ _id: ObjectId("..."), title: "...", ... }

// Collection comments
{ _id: ObjectId("..."), articleId: ObjectId("..."), ... }
```

**Rejeté car :**
- Requiert 2 requêtes pour charger un article avec commentaires
- Ajout de ~100ms de latence (network + query)
- Complexifie le code (jointure applicative)
- 90% des cas d'usage pénalisés

### Alternative 2: Embedded Illimité
```javascript
// Tous les commentaires imbriqués
{
  _id: ObjectId("..."),
  title: "...",
  comments: [/* tous les commentaires */]
}
```

**Rejeté car :**
- Document peut dépasser 16 MB (250+ commentaires)
- Performance dégradée sur articles populaires
- Mise à jour du document entier à chaque nouveau commentaire

### Alternative 3: GridFS
**Rejeté car :** Over-engineering pour ce cas d'usage

## Conséquences

### Positives
- ✅ Performance optimale (1 requête pour 90% des cas)
- ✅ Simplicité du code applicatif
- ✅ Pas de problème de taille (50 commentaires ≈ 10 KB)
- ✅ Facile à implémenter et maintenir

### Négatives
- ⚠️ Les articles avec >50 commentaires nécessitent requête supplémentaire
- ⚠️ Duplication des commentaires (50 dans article + tous dans collection)
- ⚠️ Mise à jour pour maintenir le subset synchronisé

### Mitigations
- Articles avec >50 commentaires : <5% (acceptable)
- Duplication : Cost/benefit favorable (10 KB vs performance)
- Synchronisation : Gérée par fonction dédiée avec tests

## Métriques de Succès
- Temps de chargement article : < 200ms ✓ (moyenne 45ms)
- Complexité code : Simple ✓
- Satisfaction développeurs : 8/10

## Implémentation
- Code: `src/models/Article.js`
- Tests: `tests/models/Article.test.js`
- Migration: `migrations/003-embed-comments.js`

## Review
À revoir si :
- Commentaires moyens dépasse 40 par article
- Patterns d'accès changent (recherche cross-article)
- Performance se dégrade

Prochaine review : 2024-07-15 (6 mois)

## Références
- MongoDB Best Practices: Embedded Documents
- Pattern: Subset Pattern
- Discussion Slack: #architecture-2024-01-10

## Auteurs
- Proposé par : Alice (alice@company.com)
- Reviewé par : Bob, Charlie
- Approuvé par : Tech Lead David
```

---

## ✅ DO : Maintenir un README Technique Complet

**Explication** : Le README du projet doit être la porte d'entrée pour comprendre l'architecture MongoDB.

**Structure README recommandée** :
```markdown
# Project Name - MongoDB Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Collections](#collections)
4. [Indexes](#indexes)
5. [Queries](#queries)
6. [Migrations](#migrations)
7. [Development](#development)
8. [Deployment](#deployment)

## Overview

### Database Architecture
- **Provider**: MongoDB Atlas (M40, us-east-1)
- **Replica Set**: 3-node cluster with auto-failover
- **Sharding**: Enabled on `orders` collection (userId shard key)
- **Current Version**: MongoDB 7.0.x
- **Schema Version**: 3 (last updated: 2024-01-15)

### Collections Summary
| Collection | Documents | Avg Size | Purpose |
|------------|-----------|----------|---------|
| users | ~500K | 2 KB | User accounts |
| articles | ~10K | 15 KB | Blog articles |
| orders | ~2M | 5 KB | Customer orders |
| products | ~50K | 3 KB | Product catalog |

## Architecture

### Data Model Diagram
```
[users] ─┬─> [orders]
         ├─> [reviews]
         └─> [sessions]

[articles] ──> [article_comments]

[products] <── [orders.items]
```

### Design Patterns Used
- **Embedded Documents**: Comments in articles (Subset pattern)
- **References**: Users → Orders (too large to embed)
- **Bucketing**: Time-series data in analytics collection
- **Computed Pattern**: Daily statistics pre-aggregated

## Collections

### users
**Purpose**: Store user account and authentication data

**Schema**: [docs/schema/users.md](docs/schema/users.md)

**Key Fields**:
- `email` (unique): User email for authentication
- `username` (unique): Public username
- `roles[]`: Array of role objects

**Indexes**:
- `email_1_unique`: Authentication lookup
- `username_1_unique`: Profile pages
- `createdAt_-1`: Recent users admin view

**Related**: orders, sessions, reviews

### orders
**Purpose**: Customer purchase orders

**Schema**: [docs/schema/orders.md](docs/schema/orders.md)

**Sharding**: Yes (shard key: `userId`)

**Key Fields**:
- `userId`: Reference to users
- `items[]`: Array of ordered products
- `status`: Order status (pending → completed)

**Indexes**:
- `userId_1_createdAt_-1`: User order history
- `status_1`: Order management queries

## Queries

### Common Queries

#### Get User with Recent Orders
```javascript
// See: src/queries/getUserDashboard.js
const user = await db.users.findOne({ _id: userId });
const recentOrders = await db.orders
  .find({ userId: userId })
  .sort({ createdAt: -1 })
  .limit(10)
  .toArray();
```

#### Search Products
```javascript
// See: src/queries/searchProducts.js
// Uses text index on: name, description
const products = await db.products
  .find({ $text: { $search: query } })
  .project({ score: { $meta: "textScore" } })
  .sort({ score: { $meta: "textScore" } })
  .limit(20)
  .toArray();
```

### Performance Notes
- Most queries use indexes (check with explain())
- Slow query threshold: 100ms (logged automatically)
- Query profiling enabled at level 1

## Migrations

### Schema Versioning
- Current schema version: 3
- Migrations are tracked in `_migrations` collection
- Tool: migrate-mongo

### Recent Migrations
- **003**: Split user.name into firstName/lastName (2024-01-15)
- **002**: Add orders.shippingAddress (2024-01-10)
- **001**: Initial schema (2023-11-01)

### Running Migrations
```bash
# Check status
npm run migrate:status

# Run pending migrations
npm run migrate:up

# Rollback last migration
npm run migrate:down
```

## Development

### Local Setup
```bash
# 1. Start MongoDB locally
docker-compose up -d mongodb

# 2. Seed test data
npm run seed:dev

# 3. Run application
npm run dev
```

### Environment Variables
```bash
NODE_ENV=development
DB_HOST=localhost:27017
DB_NAME=myapp_dev
DB_USER=dev_user
DB_PASSWORD=dev_password
```

### Testing
```bash
# Unit tests
npm test

# Integration tests (requires MongoDB)
npm run test:integration

# Test coverage
npm run test:coverage
```

## Deployment

### Environments
- **Development**: localhost / dev cluster
- **Staging**: MongoDB Atlas M30 (staging-cluster)
- **Production**: MongoDB Atlas M40 (prod-cluster)

### Deployment Process
1. Run tests
2. Deploy to staging
3. Run smoke tests
4. Manual approval
5. Deploy to production with canary (10% → 100%)

### Monitoring
- **Application**: Datadog APM
- **Database**: MongoDB Atlas monitoring
- **Alerts**: PagerDuty integration
- **Logs**: CloudWatch Logs

## Additional Resources
- [MongoDB Schema Documentation](docs/schema/)
- [Architecture Decision Records](docs/adr/)
- [API Documentation](docs/api/)
- [Runbooks](docs/runbooks/)

## Team Contacts
- **Tech Lead**: Alice (alice@company.com)
- **DevOps**: Bob (bob@company.com)
- **On-Call**: #oncall-engineering (Slack)
```

---

## ❌ DON'T : Négliger la Documentation ou la Laisser Obsolète

**Explication** : Une documentation obsolète est pire qu'une absence de documentation car elle trompe et crée de la confusion.

**Signes de documentation obsolète** :
```javascript
// ❌ Documentation qui ne correspond plus au code

/**
 * Get user by email
 * @param {string} email - User email
 * @returns {Object} User object
 */
async function getUserByEmail(email) {
  // Code actuel utilise username, pas email!
  return await db.users.findOne({ username: email });
}

// Conséquences :
// - Développeur utilise mal la fonction
// - Bugs introduits
// - Perte de confiance dans la documentation
// - Développeurs ignorent la doc (cercle vicieux)
```

**Maintenir la documentation à jour** :
```javascript
// ✅ Processus de maintenance de documentation

// 1. Code review checklist
const pullRequestChecklist = [
  "Code reviewed",
  "Tests added/updated",
  "Documentation updated",  // ← Obligatoire
  "Changelog updated",
  "ADR created if architectural change"
];

// 2. Automated checks
// .github/workflows/docs-check.yml
/*
name: Documentation Check
on: [pull_request]
jobs:
  check-docs:
    - Check if schema files modified → Require docs update
    - Check if new queries added → Require docs update
    - Lint markdown files
    - Check broken links
*/

// 3. Documentation debt tracking
// docs/TODO.md
/*
## Documentation TODO
- [ ] Update users.md for schema v3 (Alice, 2024-01-20)
- [ ] Document new search aggregation (Bob, 2024-01-18)
- [ ] Add performance metrics to indexes.md (Charlie, 2024-01-15)
*/

// 4. Regular documentation review
// Schedule: Monthly
const documentationReview = {
  frequency: "monthly",
  checklist: [
    "Verify schema docs match actual schema",
    "Check index documentation completeness",
    "Review and update ADRs",
    "Update metrics and performance numbers",
    "Check for broken links",
    "Remove obsolete documentation"
  ]
};
```

---

## Checklist Documentation

### Schéma
- [ ] Chaque collection documentée
- [ ] Champs avec types et descriptions
- [ ] Contraintes et validations
- [ ] Relations avec autres collections
- [ ] Exemples de documents
- [ ] Business rules expliquées

### Index
- [ ] Chaque index documenté
- [ ] Purpose et justification
- [ ] Requêtes optimisées
- [ ] Metrics de performance
- [ ] Date de création
- [ ] Revue d'utilisation régulière

### Requêtes
- [ ] Requêtes complexes commentées
- [ ] Agrégations expliquées étape par étape
- [ ] Performance attendue
- [ ] Index utilisés
- [ ] Limitations connues

### Architecture
- [ ] ADR pour décisions majeures
- [ ] Diagrammes à jour
- [ ] Patterns utilisés documentés
- [ ] Trade-offs expliqués

### Code
- [ ] Fonctions avec JSDoc
- [ ] Commentaires inline pour logique complexe
- [ ] TODOs avec contexte et date
- [ ] Warnings pour comportements non-évidents

### Projet
- [ ] README complet et à jour
- [ ] Guide de setup
- [ ] Guide de déploiement
- [ ] Contacts et escalation
- [ ] Changelog maintenu

---

## Conclusion

La documentation est un investissement, pas un coût :

- **ROI mesuré** : 500% (20k€ doc vs 100k€ temps perdu)
- **Onboarding** : 70% plus rapide
- **Bugs** : 80% de réduction
- **Maintenance** : 60% plus rapide
- **Turnover** : Réduction significative

**Règles d'or** :
1. **Documenter en écrivant** : Pas "plus tard"
2. **Proche du code** : Doc et code ensemble
3. **Maintenir à jour** : Doc obsolète = pire que rien
4. **Exemples concrets** : Code > prose
5. **Contexte et pourquoi** : Pas seulement quoi
6. **Accessible** : README, inline, wiki

Une équipe qui documente bien est une équipe qui scale. La documentation est le code que vous écrivez pour les humains.

---


⏭️ [Revue de code pour MongoDB](/21-bonnes-pratiques-anti-patterns/10-revue-code-mongodb.md)
