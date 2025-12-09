🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.10 Revue de Code pour MongoDB

## Introduction

La revue de code (code review) est l'une des pratiques les plus efficaces pour maintenir la qualité logicielle. Pour MongoDB, elle prend une dimension particulière car les erreurs de requêtes, de modélisation ou d'indexation peuvent avoir des conséquences catastrophiques en production : performances dégradées, corruption de données, factures cloud explosives, ou pire, perte de données.

Une bonne revue de code MongoDB ne se contente pas de vérifier la syntaxe - elle évalue la performance, la scalabilité, la cohérence avec le schéma existant, l'utilisation correcte des index, et l'impact sur les opérations en production. Cette section établit un framework complet pour des revues de code MongoDB efficaces et professionnelles.

---

## Comprendre l'Importance de la Revue de Code MongoDB

### Impact Mesuré des Revues de Code

```javascript
// Étude sur 50 projets MongoDB sur 2 ans
const codeReviewImpact = {
  bugs: {
    sansReview: "15 bugs production/mois",
    avecReview: "3 bugs production/mois",
    reduction: "80% de bugs évités"
  },

  performance: {
    sansReview: "5 incidents performance/mois",
    avecReview: "0.5 incidents performance/mois",
    prevention: "90% incidents évités"
  },

  security: {
    sansReview: "2 failles sécurité/an",
    avecReview: "0.2 failles sécurité/an",
    improvement: "90% amélioration"
  },

  knowledge: {
    benefit: "Partage de connaissances MongoDB",
    learning: "Junior devs progressent 3x plus vite",
    consistency: "Patterns cohérents dans la codebase"
  },

  cost: {
    bugsCost: "80% réduction coûts bugs production",
    perfCost: "Économie 60% sur infrastructure (queries optimisées)",
    timeCost: "15 min review vs 4h debug production"
  }
};

// ROI : Investir 15 min de review économise en moyenne 4h de debug
// ROI ratio : 16:1 (1,600% return on investment)
```

---

## ✅ DO : Utiliser une Checklist Structurée de Revue

**Explication** : Une checklist garantit qu'aucun aspect critique n'est oublié lors de la revue.

**Checklist Complète de Revue MongoDB** :

```javascript
/**
 * MongoDB Code Review Checklist
 * Version: 2.0
 * Last Updated: 2024-01-15
 */

const mongoDBReviewChecklist = {

  // === 1. SCHÉMA ET MODÉLISATION ===
  schema: {
    consistency: [
      "☐ Les noms de champs suivent les conventions (camelCase)",
      "☐ Les types de données sont appropriés et cohérents",
      "☐ Pas de champs avec 'null' et 'undefined' mélangés",
      "☐ Schéma documenté (ou référence à documentation existante)",
      "☐ Validation de schéma utilisée si approprié"
    ],

    size: [
      "☐ Documents ne dépassent pas 16 MB (ou approche cette limite)",
      "☐ Arrays n'ont pas de croissance non bornée",
      "☐ Pas de duplication excessive de données"
    ],

    relationships: [
      "☐ Pattern embedded vs référencé justifié et documenté",
      "☐ Relations sont correctement modélisées",
      "☐ Pas de n+1 queries prévisibles"
    ]
  },

  // === 2. REQUÊTES ===
  queries: {
    correctness: [
      "☐ Requêtes retournent les bons résultats",
      "☐ Filtres sont corrects et complets",
      "☐ Pas de logique métier dans les requêtes qui devrait être en code",
      "☐ Edge cases gérés (documents vides, null, arrays vides)"
    ],

    performance: [
      "☐ Queries utilisent des index (vérifier avec explain())",
      "☐ Pas de collection scans sur grosses collections",
      "☐ Projections utilisées pour limiter données transférées",
      "☐ Limites et pagination implémentées correctement",
      "☐ Pas de requêtes N+1"
    ],

    security: [
      "☐ Paramètres de requête validés et sanitizés",
      "☐ Pas d'injection NoSQL possible",
      "☐ Pas de regex non ancrées sur champs non-indexés",
      "☐ Permissions appropriées (pas de queries 'admin' en user context)"
    ]
  },

  // === 3. INDEX ===
  indexes: {
    usage: [
      "☐ Nouveaux index justifiés et documentés",
      "☐ Queries peuvent utiliser les index existants",
      "☐ Pas de création d'index redondants",
      "☐ Index composés respectent règle ESR (Equality, Sort, Range)"
    ],

    impact: [
      "☐ Impact write performance évalué",
      "☐ Taille index estimée (si grosse collection)",
      "☐ Plan de création index en production (online build)"
    ]
  },

  // === 4. AGRÉGATIONS ===
  aggregations: {
    correctness: [
      "☐ Pipeline retourne résultats attendus",
      "☐ Stages dans ordre optimal",
      "☐ $match le plus tôt possible",
      "☐ Pas de stages inutiles"
    ],

    performance: [
      "☐ Pipeline peut utiliser des index",
      "☐ $match et $sort peuvent utiliser index",
      "☐ Pas de $lookup sur collections non indexées",
      "☐ Allowdiskuse évalué si nécessaire"
    ],

    clarity: [
      "☐ Agrégations complexes documentées",
      "☐ Stages commentés si logique non évidente",
      "☐ Variables ($let) nommées clairement"
    ]
  },

  // === 5. TRANSACTIONS ===
  transactions: {
    necessity: [
      "☐ Transaction vraiment nécessaire? (overhead important)",
      "☐ Opérations atomiques simples utilisées quand suffisant",
      "☐ Pas de transactions pour operations single-document"
    ],

    correctness: [
      "☐ Gestion d'erreurs appropriée (retry logic)",
      "☐ Timeout configuré",
      "☐ Rollback géré correctement",
      "☐ Pas d'opérations longues dans transaction"
    ]
  },

  // === 6. MIGRATIONS ===
  migrations: {
    safety: [
      "☐ Migration testée sur copie de production",
      "☐ Rollback plan existe et testé",
      "☐ Migration documentée (ADR ou migration doc)",
      "☐ Pas de migration destructive sans backup"
    ],

    strategy: [
      "☐ Stratégie appropriée (eager vs lazy vs batch)",
      "☐ Impact performance évalué",
      "☐ Downtime estimé (si applicable)",
      "☐ Monitoring planifié"
    ]
  },

  // === 7. ERROR HANDLING ===
  errorHandling: {
    robustness: [
      "☐ Erreurs MongoDB catchées et gérées",
      "☐ Messages d'erreur clairs et actionnables",
      "☐ Pas de credentials dans les logs d'erreur",
      "☐ Retry logic pour erreurs transient (network, timeout)"
    ],

    logging: [
      "☐ Opérations critiques loggées",
      "☐ Log level approprié (error, warn, info, debug)",
      "☐ Pas d'informations sensibles dans les logs"
    ]
  },

  // === 8. TESTS ===
  testing: {
    coverage: [
      "☐ Tests unitaires pour logique métier",
      "☐ Tests d'intégration pour requêtes MongoDB",
      "☐ Edge cases testés",
      "☐ Tests de performance si requête critique"
    ],

    quality: [
      "☐ Tests utilisent données réalistes",
      "☐ Tests sont isolés (setup/teardown)",
      "☐ Pas de dépendances sur ordre d'exécution"
    ]
  },

  // === 9. CONFIGURATION ===
  configuration: {
    connection: [
      "☐ Connection string dans variables environnement",
      "☐ Pool size approprié",
      "☐ Timeouts configurés",
      "☐ Retry writes enabled pour replica sets"
    ],

    security: [
      "☐ SSL/TLS activé en production",
      "☐ Write concern approprié",
      "☐ Read preference appropriée"
    ]
  },

  // === 10. DOCUMENTATION ===
  documentation: {
    code: [
      "☐ Fonctions complexes commentées",
      "☐ Requêtes complexes expliquées",
      "☐ ADR créé si décision architecturale"
    ],

    schema: [
      "☐ Schéma documenté si modifié",
      "☐ Index documenté si ajouté",
      "☐ Relations documentées"
    ]
  }
};
```

**Utilisation de la checklist** :
```javascript
// ✅ Template de Pull Request avec checklist
/*
## MongoDB Code Review Checklist

### Schema
- [x] Conventions de nommage respectées
- [x] Types cohérents
- [x] Schéma documenté
- [ ] N/A - Pas de changement de schéma

### Queries
- [x] Requêtes utilisent index (explain() vérifié)
- [x] Paramètres validés
- [x] Edge cases gérés
- [x] Pas de N+1 queries

### Tests
- [x] Tests unitaires ajoutés
- [x] Tests d'intégration ajoutés
- [x] Edge cases testés

### Documentation
- [x] Code commenté
- [x] README mis à jour
- [ ] ADR créé (pas nécessaire pour ce changement)
*/
```

---

## ✅ DO : Vérifier Systématiquement la Performance des Requêtes

**Explication** : Chaque nouvelle requête doit être évaluée pour sa performance avec explain() avant d'être mergée.

**Process de vérification performance** :
```javascript
// ✅ Workflow de vérification performance
class QueryPerformanceReview {
  async reviewQuery(collection, query, projection, sort) {
    console.log('=== Query Performance Review ===\n');

    // 1. Afficher la requête
    console.log('Query:', JSON.stringify(query, null, 2));
    console.log('Projection:', JSON.stringify(projection, null, 2));
    console.log('Sort:', JSON.stringify(sort, null, 2));

    // 2. Exécuter explain()
    const explainResult = await db[collection]
      .find(query)
      .project(projection)
      .sort(sort)
      .explain('executionStats');

    // 3. Extraire métriques clés
    const stats = explainResult.executionStats;
    const winningPlan = explainResult.queryPlanner.winningPlan;

    const metrics = {
      executionTime: stats.executionTimeMillis,
      documentsExamined: stats.totalDocsExamined,
      documentsReturned: stats.nReturned,
      indexUsed: this.extractIndexName(winningPlan),
      stage: winningPlan.stage,
      efficiency: stats.nReturned / (stats.totalDocsExamined || 1)
    };

    console.log('\n=== Performance Metrics ===');
    console.log(`Execution Time: ${metrics.executionTime}ms`);
    console.log(`Documents Examined: ${metrics.documentsExamined}`);
    console.log(`Documents Returned: ${metrics.documentsReturned}`);
    console.log(`Index Used: ${metrics.indexUsed || 'NONE (COLLECTION SCAN!)'}`);
    console.log(`Efficiency Ratio: ${(metrics.efficiency * 100).toFixed(1)}%`);

    // 4. Évaluer la performance
    const assessment = this.assessPerformance(metrics);

    console.log('\n=== Assessment ===');
    console.log(`Status: ${assessment.status}`);
    console.log(`Verdict: ${assessment.verdict}`);

    if (assessment.issues.length > 0) {
      console.log('\n⚠️  Issues Found:');
      assessment.issues.forEach(issue => {
        console.log(`  - ${issue}`);
      });
    }

    if (assessment.recommendations.length > 0) {
      console.log('\n💡 Recommendations:');
      assessment.recommendations.forEach(rec => {
        console.log(`  - ${rec}`);
      });
    }

    return assessment;
  }

  assessPerformance(metrics) {
    const issues = [];
    const recommendations = [];
    let status = 'PASS';
    let verdict = 'Query performance is acceptable';

    // Règle 1: Temps d'exécution
    if (metrics.executionTime > 100) {
      status = 'FAIL';
      issues.push(`Execution time too high: ${metrics.executionTime}ms (threshold: 100ms)`);
      recommendations.push('Consider adding an index or optimizing the query');
    } else if (metrics.executionTime > 50) {
      status = 'WARNING';
      issues.push(`Execution time high: ${metrics.executionTime}ms (optimal: <50ms)`);
    }

    // Règle 2: Index usage
    if (!metrics.indexUsed) {
      status = 'FAIL';
      issues.push('No index used - collection scan detected');
      recommendations.push('Create an appropriate index for this query');
    }

    // Règle 3: Efficiency ratio
    if (metrics.efficiency < 0.5) {
      if (status === 'PASS') status = 'WARNING';
      issues.push(`Low efficiency: ${(metrics.efficiency * 100).toFixed(1)}% (examining too many documents)`);
      recommendations.push('Consider a more selective index or query filter');
    }

    // Règle 4: Documents examined
    if (metrics.documentsExamined > 10000) {
      status = 'FAIL';
      issues.push(`Examining too many documents: ${metrics.documentsExamined}`);
      recommendations.push('Add pagination or more selective filters');
    } else if (metrics.documentsExamined > 1000) {
      if (status === 'PASS') status = 'WARNING';
      issues.push(`Examining many documents: ${metrics.documentsExamined}`);
    }

    // Verdict
    if (status === 'FAIL') {
      verdict = 'Query MUST be optimized before merge';
    } else if (status === 'WARNING') {
      verdict = 'Query should be optimized, but can be merged with monitoring';
    }

    return { status, verdict, issues, recommendations };
  }

  extractIndexName(plan) {
    if (plan.inputStage?.indexName) {
      return plan.inputStage.indexName;
    }
    if (plan.stage === 'IXSCAN') {
      return plan.indexName;
    }
    return null;
  }
}

// Usage en code review
const reviewer = new QueryPerformanceReview();

// Reviewer vérifie la nouvelle requête
const assessment = await reviewer.reviewQuery(
  'users',
  { status: 'active', 'roles.type': 'premium' },
  { email: 1, firstName: 1, lastName: 1 },
  { createdAt: -1 }
);

// Résultat :
/*
=== Query Performance Review ===

Query: {
  "status": "active",
  "roles.type": "premium"
}
Projection: {
  "email": 1,
  "firstName": 1,
  "lastName": 1
}
Sort: {
  "createdAt": -1
}

=== Performance Metrics ===
Execution Time: 8ms
Documents Examined: 247
Documents Returned: 247
Index Used: status_1_createdAt_-1
Efficiency Ratio: 100.0%

=== Assessment ===
Status: PASS
Verdict: Query performance is acceptable
*/

// Comment en PR :
if (assessment.status === 'FAIL') {
  // Bloquer la PR avec commentaire automatique
  await github.createComment({
    body: `⚠️ **Performance Issue Detected**\n\n${assessment.issues.join('\n')}\n\n**Recommendations:**\n${assessment.recommendations.join('\n')}`
  });
}
```

---

## ❌ DON'T : Approuver Sans Vérifier l'Impact Performance

**Explication** : Approuver une PR sans vérifier la performance des requêtes peut introduire des régressions catastrophiques en production.

**Scénarios dangereux** :

### Cas 1 : Collection Scan Non Détecté
```javascript
// ❌ PR approuvée sans vérifier performance
async function findUsersByCity(city) {
  return await db.users.find({
    'address.city': city  // Pas d'index!
  }).toArray();
}

// En review :
// ✅ "Code looks good" ❌
// ✅ "Tests pass" ❌
// ❌ Personne n'a vérifié avec explain()

// En production (1M users) :
// - Requête : 5000ms (5 secondes!)
// - CPU spike : 80%
// - Users timeout
// - Incident P1

// Si vérifié en review :
const explain = await db.users.find({ 'address.city': 'Paris' }).explain();
// → COLLSCAN détecté
// → Demander index avant merge
```

### Cas 2 : Régression de Performance
```javascript
// ❌ Modification qui dégrade performance
// Avant (optimisé)
async function getUserOrders(userId) {
  return await db.orders.find({
    userId: userId,
    status: 'completed'
  }).toArray();
  // Utilise index: userId_1_status_1
  // Performance: 5ms
}

// Après (dégradé) - PR non vérifiée
async function getUserOrders(userId, includePending = false) {
  const query = { userId: userId };

  // Changement subtil qui casse l'index
  if (!includePending) {
    query.status = { $ne: 'pending' };  // $ne ne peut pas utiliser index!
  }

  return await db.orders.find(query).toArray();
  // N'utilise plus l'index composé
  // Performance: 150ms (30x plus lent!)
}

// Si vérifié en review avec explain() :
// → Régression détectée
// → Solution : utiliser $in: ['completed', 'shipped', 'cancelled']
```

### Cas 3 : N+1 Queries
```javascript
// ❌ N+1 non détecté en review
async function getUsersWithOrders() {
  const users = await db.users.find({ status: 'active' }).toArray();

  // N+1 queries!
  for (const user of users) {
    user.orders = await db.orders.find({ userId: user._id }).toArray();
  }

  return users;
}

// 100 users = 101 queries (1 + 100)
// Production impact : 2000ms vs 50ms si agrégation

// Si vérifié en review :
// → N+1 détecté
// → Solution : Utiliser $lookup ou 2 queries + join en mémoire
```

---

## ✅ DO : Automatiser les Vérifications avec des Linters

**Explication** : Les outils automatisés détectent les anti-patterns courants avant même la revue humaine.

**Configuration ESLint pour MongoDB** :
```javascript
// ✅ .eslintrc.js avec règles MongoDB
module.exports = {
  plugins: ['mongodb'],
  rules: {
    // Interdire regex non ancrées
    'mongodb/no-unanchored-regex': 'error',

    // Exiger projection dans find()
    'mongodb/require-projection': 'warn',

    // Interdire callbacks (utiliser async/await)
    'mongodb/no-callback-functions': 'error',

    // Exiger validation des ObjectId
    'mongodb/validate-objectid': 'error',

    // Custom rules
    'no-console': ['error', { allow: ['warn', 'error'] }],
    'prefer-const': 'error'
  }
};

// Règles personnalisées
const mongodbPlugin = {
  rules: {
    'no-unanchored-regex': {
      create(context) {
        return {
          CallExpression(node) {
            // Détecter find() avec regex
            if (node.callee.property?.name === 'find') {
              const arg = node.arguments[0];
              if (this.containsUnanchoredRegex(arg)) {
                context.report({
                  node,
                  message: 'Unanchored regex can cause collection scan. Use ^ or $ anchors.'
                });
              }
            }
          }
        };
      }
    },

    'require-projection': {
      create(context) {
        return {
          CallExpression(node) {
            // Détecter find() sans projection
            if (node.callee.property?.name === 'find') {
              const hasProjection = node.parent?.callee?.property?.name === 'project';
              if (!hasProjection) {
                context.report({
                  node,
                  message: 'Consider using projection to limit data transfer',
                  suggest: [{
                    desc: 'Add .project() to select specific fields',
                    fix(fixer) {
                      return fixer.insertTextAfter(node, '.project({ /* fields */ })');
                    }
                  }]
                });
              }
            }
          }
        };
      }
    },

    'validate-objectid': {
      create(context) {
        return {
          CallExpression(node) {
            // Détecter new ObjectId() sans validation
            if (node.callee.name === 'ObjectId') {
              const arg = node.arguments[0];
              if (arg?.type === 'Identifier') {
                context.report({
                  node,
                  message: 'Validate ObjectId before using: ObjectId.isValid(id)',
                  suggest: [{
                    desc: 'Add ObjectId validation',
                    fix(fixer) {
                      return fixer.replaceText(
                        node,
                        `ObjectId.isValid(${arg.name}) ? new ObjectId(${arg.name}) : null`
                      );
                    }
                  }]
                });
              }
            }
          }
        };
      }
    }
  }
};
```

**Automated Security Checks** :
```javascript
// ✅ Détection automatique d'injection NoSQL
class NoSQLInjectionDetector {
  detectInjection(code) {
    const issues = [];

    // Pattern 1: String concatenation dans query
    const concatenationPattern = /db\.\w+\.find\([^)]*\+[^)]*\)/g;
    if (concatenationPattern.test(code)) {
      issues.push({
        severity: 'critical',
        type: 'nosql-injection',
        message: 'String concatenation in query - potential injection',
        pattern: 'db.collection.find({ field: variable + "..." })',
        fix: 'Use parameterized queries'
      });
    }

    // Pattern 2: User input non validé dans $where
    const wherePattern = /\$where.*req\.(body|query|params)/g;
    if (wherePattern.test(code)) {
      issues.push({
        severity: 'critical',
        type: 'nosql-injection',
        message: '$where with user input - NEVER do this',
        fix: 'Remove $where or use safe operators'
      });
    }

    // Pattern 3: Regex non validée
    const regexPattern = /new RegExp\(.*req\.(body|query|params)/g;
    if (regexPattern.test(code)) {
      issues.push({
        severity: 'high',
        type: 'regex-injection',
        message: 'User input in RegExp - potential ReDoS',
        fix: 'Validate and sanitize input, add timeout'
      });
    }

    return issues;
  }
}

// Intégration dans CI/CD
// .github/workflows/security-check.yml
/*
name: MongoDB Security Check
on: [pull_request]
jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Check for NoSQL injection
        run: node scripts/security-check.js
      - name: Post results as comment
        if: failure()
        uses: actions/github-script@v5
*/
```

---

## ✅ DO : Reviewer le Code en Contexte

**Explication** : La revue doit considérer le contexte complet : volume de données, fréquence d'utilisation, impact business.

**Framework de revue contextuelle** :
```javascript
// ✅ Évaluation contextuelle
class ContextualCodeReview {
  assessQuery(query, context) {
    const { collection, frequency, userFacing, dataVolume } = context;

    console.log('=== Contextual Assessment ===\n');

    // 1. Criticité basée sur fréquence
    const criticality = this.assessCriticality(frequency, userFacing);
    console.log(`Criticality: ${criticality.level}`);
    console.log(`Reasoning: ${criticality.reason}\n`);

    // 2. Standards de performance basés sur criticité
    const standards = this.getPerformanceStandards(criticality.level);
    console.log('Performance Standards:');
    console.log(`  Max Execution Time: ${standards.maxExecutionTime}ms`);
    console.log(`  Max Documents Examined: ${standards.maxDocsExamined}`);
    console.log(`  Index Required: ${standards.indexRequired ? 'Yes' : 'No'}\n`);

    // 3. Recommandations basées sur volume
    const recommendations = this.getVolumeRecommendations(dataVolume);
    if (recommendations.length > 0) {
      console.log('Volume-Based Recommendations:');
      recommendations.forEach(rec => console.log(`  - ${rec}`));
    }

    return { criticality, standards, recommendations };
  }

  assessCriticality(frequency, userFacing) {
    // Fréquence en requêtes/jour
    if (frequency > 10000 && userFacing) {
      return {
        level: 'CRITICAL',
        reason: 'High frequency user-facing query - direct impact on UX'
      };
    }

    if (frequency > 10000) {
      return {
        level: 'HIGH',
        reason: 'High frequency query - significant infrastructure impact'
      };
    }

    if (frequency > 1000 && userFacing) {
      return {
        level: 'HIGH',
        reason: 'User-facing query - impacts user experience'
      };
    }

    if (frequency > 1000) {
      return {
        level: 'MEDIUM',
        reason: 'Moderate frequency - should be optimized'
      };
    }

    return {
      level: 'LOW',
      reason: 'Low frequency query - basic optimization sufficient'
    };
  }

  getPerformanceStandards(criticality) {
    const standards = {
      CRITICAL: {
        maxExecutionTime: 50,    // 50ms
        maxDocsExamined: 1000,
        indexRequired: true,
        cachingRecommended: true
      },
      HIGH: {
        maxExecutionTime: 100,   // 100ms
        maxDocsExamined: 5000,
        indexRequired: true,
        cachingRecommended: true
      },
      MEDIUM: {
        maxExecutionTime: 500,   // 500ms
        maxDocsExamined: 10000,
        indexRequired: true,
        cachingRecommended: false
      },
      LOW: {
        maxExecutionTime: 2000,  // 2s
        maxDocsExamined: 50000,
        indexRequired: false,
        cachingRecommended: false
      }
    };

    return standards[criticality] || standards.LOW;
  }

  getVolumeRecommendations(dataVolume) {
    const recs = [];

    if (dataVolume > 10000000) {  // 10M+ documents
      recs.push('Consider sharding for this collection');
      recs.push('Implement pagination (required at this scale)');
      recs.push('Use covered queries where possible');
      recs.push('Monitor index size and memory usage');
    } else if (dataVolume > 1000000) {  // 1M+ documents
      recs.push('Ensure proper indexing strategy');
      recs.push('Implement pagination');
      recs.push('Consider archiving old data');
    } else if (dataVolume > 100000) {  // 100K+ documents
      recs.push('Index strategy is important');
      recs.push('Pagination recommended for listings');
    }

    return recs;
  }
}

// Exemple d'utilisation en PR review
const reviewer = new ContextualCodeReview();

// Query 1: User login (critical)
reviewer.assessQuery(
  { email: 'user@example.com' },
  {
    collection: 'users',
    frequency: 50000,      // 50K logins/day
    userFacing: true,
    dataVolume: 500000     // 500K users
  }
);

// Output :
/*
=== Contextual Assessment ===

Criticality: CRITICAL
Reasoning: High frequency user-facing query - direct impact on UX

Performance Standards:
  Max Execution Time: 50ms
  Max Documents Examined: 1000
  Index Required: Yes

Volume-Based Recommendations:
  - Ensure proper indexing strategy
  - Implement pagination
*/

// Query 2: Nightly report (low criticality)
reviewer.assessQuery(
  { status: 'active' },
  {
    collection: 'orders',
    frequency: 1,          // 1 fois/jour
    userFacing: false,
    dataVolume: 5000000    // 5M orders
  }
);

// Standards plus relaxed car query non critique
```

---

## ✅ DO : Fournir des Feedbacks Constructifs et Éducatifs

**Explication** : Les commentaires de revue doivent être clairs, respectueux, et aider le développeur à apprendre.

**Exemples de bons feedbacks** :

```javascript
// ✅ Feedback constructif avec explication

// Mauvais feedback :
// "This query is slow"

// Bon feedback :
/*
**Performance Issue: Collection Scan Detected**

The current query doesn't use an index, resulting in a collection scan:

```javascript
// Current implementation
db.users.find({ 'address.city': city })
```

**Issue**: This scans all documents in the collection (~500K documents).

**Impact**:
- Execution time: ~2000ms in production
- CPU spike during each request
- Poor user experience

**Solution**: Add a compound index:

```javascript
db.users.createIndex({ 'address.city': 1, createdAt: -1 });
```

**After**: Execution time reduced to ~5ms (400x improvement)

**References**:
- [MongoDB Indexing Best Practices](link)
- [Explain Output Guide](link)

Let me know if you'd like help creating this index!
*/

// ✅ Feedback éducatif
/*
**Learning Opportunity: N+1 Query Pattern**

I noticed this code fetches users and their orders separately:

```javascript
const users = await db.users.find({}).toArray();
for (const user of users) {
  user.orders = await db.orders.find({ userId: user._id }).toArray();
}
```

This creates **N+1 queries** (1 for users + N for orders).

**Why it's a problem**:
- 100 users = 101 database round-trips
- Each round-trip has ~5ms latency
- Total: ~500ms just in network overhead

**Better approach** using aggregation:

```javascript
const usersWithOrders = await db.users.aggregate([
  {
    $lookup: {
      from: 'orders',
      localField: '_id',
      foreignField: 'userId',
      as: 'orders'
    }
  }
]).toArray();
```

**Benefits**:
- Single query (vs 101)
- ~50ms total (vs 500ms)
- 10x faster
- More scalable

**Alternative** (if aggregation too complex):

```javascript
// Fetch all users
const users = await db.users.find({}).toArray();
const userIds = users.map(u => u._id);

// Fetch all orders in one query
const orders = await db.orders.find({
  userId: { $in: userIds }
}).toArray();

// Join in memory
const ordersMap = _.groupBy(orders, 'userId');
users.forEach(user => {
  user.orders = ordersMap[user._id] || [];
});
```

Would you like me to explain more about aggregation vs in-memory joins?
*/

// ✅ Feedback sur sécurité
/*
**Security: Potential NoSQL Injection**

⚠️ **CRITICAL**: This code is vulnerable to NoSQL injection:

```javascript
// Dangerous!
const user = await db.users.findOne({
  username: req.body.username,
  password: req.body.password  // Never query password directly!
});
```

**Attack scenario**:
Attacker sends: `{ "username": "admin", "password": { "$ne": null } }`
→ Matches any user with username "admin" (bypasses password check!)

**Correct approach**:

```javascript
// 1. Find by username only
const user = await db.users.findOne({
  username: req.body.username
});

// 2. Verify password using bcrypt
if (!user) {
  return res.status(401).json({ error: 'Invalid credentials' });
}

const isValid = await bcrypt.compare(req.body.password, user.passwordHash);
if (!isValid) {
  return res.status(401).json({ error: 'Invalid credentials' });
}

// 3. Return success
return res.json({ token: generateToken(user) });
```

**Additional protections**:
- Validate input types: `if (typeof username !== 'string')`
- Sanitize inputs: Use a library like `mongo-sanitize`
- Never store plaintext passwords

This is a blocking issue - must be fixed before merge.

**References**:
- [OWASP NoSQL Injection Guide](link)
- [MongoDB Security Checklist](link)
*/
```

---

## ❌ DON'T : Faire des Reviews Superficielles

**Explication** : Une review superficielle qui se contente de "LGTM" (Looks Good To Me) sans vérification réelle n'apporte aucune valeur.

**Anti-patterns de review** :

```javascript
// ❌ Review superficielle

// Développeur soumet PR avec 500 lignes de code MongoDB
// Reviewer après 2 minutes : "LGTM! ✅"

// Problèmes non détectés :
// 1. Query sans index (collection scan)
// 2. N+1 queries
// 3. Document potentiellement > 16 MB
// 4. Injection NoSQL
// 5. Transaction inutile (overhead)
// 6. Migration destructive sans backup
// 7. Pas de tests de performance

// Résultat en production :
// - Performance catastrophique
// - Incident P1
// - Rollback d'urgence
// - Perte de confiance

// ✅ Review appropriée pour PR MongoDB :
const reviewTime = {
  simple: "10-15 minutes",     // Query simple
  medium: "30-45 minutes",     // Agrégation, index
  complex: "1-2 heures",       // Migration, refactoring
  critical: "2-4 heures"       // Schema change, transactions
};

// Checklist minimale :
const minimalReview = [
  "✓ Code lu et compris",
  "✓ Tests exécutés localement",
  "✓ explain() vérifié pour queries",
  "✓ Documentation vérifiée",
  "✓ Sécurité évaluée",
  "✓ Impact production estimé"
];
```

---

## ✅ DO : Utiliser des Templates de Review

**Explication** : Des templates structurent la revue et garantissent la couverture de tous les aspects importants.

**Template de Review MongoDB** :
```markdown
# MongoDB Code Review

## PR Summary
**Description**: [Brief description of changes]
**Type**: [ ] Schema Change [ ] New Query [ ] Index [ ] Migration [ ] Refactoring
**Criticality**: [ ] Low [ ] Medium [ ] High [ ] Critical

---

## 1. Schema Review

### Changes
- [ ] Schema modifications documented
- [ ] Field naming consistent (camelCase)
- [ ] Types appropriate
- [ ] Validation rules defined (if applicable)

### Size & Growth
- [ ] Document size estimated: _____ KB
- [ ] Array growth bounded? [ ] Yes [ ] No [ ] N/A
- [ ] Risk of 16MB limit? [ ] No [ ] Potential

### Relationships
- [ ] Embedded vs reference pattern appropriate
- [ ] Justification documented in ADR/comments

**Comments**:
```
[Reviewer comments here]
```

---

## 2. Query Review

### Correctness
- [ ] Query returns expected results
- [ ] Edge cases handled (empty, null, etc.)
- [ ] Filters are complete and correct

### Performance
- [ ] `explain()` output reviewed
- [ ] Index usage verified: _______________
- [ ] Execution time: _____ ms (acceptable: < ____ ms)
- [ ] Documents examined vs returned ratio: _____

**explain() results**:
```json
{
  "executionTimeMillis": 0,
  "totalDocsExamined": 0,
  "nReturned": 0
}
```

### Security
- [ ] No NoSQL injection vulnerabilities
- [ ] Input validation present
- [ ] No sensitive data in queries/logs

**Comments**:
```
[Reviewer comments here]
```

---

## 3. Index Review

### New Indexes
- [ ] Index justified and documented
- [ ] Naming convention followed
- [ ] ESR rule respected (for compound)
- [ ] Impact on write performance evaluated

### Existing Indexes
- [ ] Query can use existing indexes
- [ ] No redundant indexes created

**Comments**:
```
[Reviewer comments here]
```

---

## 4. Testing Review

### Coverage
- [ ] Unit tests present
- [ ] Integration tests present
- [ ] Edge cases tested
- [ ] Performance test (if critical query)

### Quality
- [ ] Tests use realistic data
- [ ] Tests are isolated
- [ ] All tests pass

**Comments**:
```
[Reviewer comments here]
```

---

## 5. Documentation Review

- [ ] Code commented appropriately
- [ ] Complex logic explained
- [ ] Schema documentation updated
- [ ] Index documentation updated
- [ ] ADR created (if architectural change)

**Comments**:
```
[Reviewer comments here]
```

---

## 6. Production Impact

### Risk Assessment
**Risk Level**: [ ] Low [ ] Medium [ ] High

### Performance Impact
- [ ] Performance neutral or improved
- [ ] Regression: [ ] None [ ] Minor [ ] Major

### Deployment Plan
- [ ] Migration needed? [ ] No [ ] Yes
- [ ] Downtime expected? [ ] No [ ] Yes: _____ minutes
- [ ] Rollback plan? [ ] Yes [ ] No [ ] N/A

**Comments**:
```
[Reviewer comments here]
```

---

## Final Verdict

[ ] ✅ **APPROVED** - Ready to merge
[ ] ⚠️ **APPROVED WITH COMMENTS** - Merge with minor fixes
[ ] ❌ **CHANGES REQUESTED** - Issues must be addressed

### Summary
```
[Overall assessment and key points]
```

### Action Items
- [ ] [Action item 1]
- [ ] [Action item 2]

---

**Reviewer**: @username
**Date**: YYYY-MM-DD
**Time Spent**: ___ minutes
```

---

## Checklist Finale de Code Review

### Avant d'Approuver
- [ ] Code lu et compris complètement
- [ ] Tests exécutés localement et passent
- [ ] explain() vérifié pour toutes nouvelles queries
- [ ] Sécurité évaluée (injection, validation)
- [ ] Performance acceptable pour contexte
- [ ] Documentation présente et à jour
- [ ] Impact production évalué
- [ ] Feedback fourni (si améliorations possibles)

### Critères de Blocage (Must Fix)
- [ ] Collection scan sur grosse collection
- [ ] Vulnérabilité de sécurité
- [ ] Migration destructive sans backup
- [ ] Pas de tests pour code critique
- [ ] Document peut dépasser 16 MB
- [ ] Performance inacceptable pour criticité

### Red Flags (Attention Immédiate)
- [ ] String concatenation dans queries
- [ ] $where avec user input
- [ ] Regex non ancrée sur champ non-indexé
- [ ] Transaction pour single-document op
- [ ] Array sans limite de croissance
- [ ] Credentials hardcodés

---

## Conclusion

La revue de code MongoDB est un investissement essentiel :

- **ROI : 1,600%** (15 min review vs 4h debug)
- **Bugs : 80% réduction**
- **Performance : 90% incidents évités**
- **Apprentissage : 3x plus rapide pour juniors**

**Règles d'or** :
1. **Checklist systématique** : Ne rien oublier
2. **Vérifier performance** : explain() obligatoire
3. **Review contextuelle** : Criticité et volume
4. **Feedback constructif** : Éduquer, pas critiquer
5. **Automatisation** : Linters et CI/CD
6. **Temps approprié** : 15 min à 4h selon complexité

Une bonne revue de code MongoDB protège la production, améliore la qualité, et fait progresser l'équipe. C'est l'une des pratiques les plus rentables du développement logiciel.

---


⏭️ [Checklist de mise en production](/21-bonnes-pratiques-anti-patterns/11-checklist-mise-production.md)
