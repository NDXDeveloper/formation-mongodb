🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 9 : Cas d'Usage et Bonnes Pratiques (Transversal)

## 🎯 De la théorie à la pratique : Retours d'expérience et sagesse collective

Vous avez maintenant une connaissance approfondie de MongoDB : fondamentaux, architecture, sécurité, cloud, développement, performance et production. Vous possédez la **théorie complète**. Mais entre savoir comment faire quelque chose et savoir **quand, pourquoi et dans quel contexte** le faire, il y a un fossé immense qui ne se comble qu'avec l'expérience.

La Partie 9 est une **synthèse transversale** qui comble ce fossé. Elle consolide toutes vos connaissances à travers des cas d'usage réels, des architectures de référence, des bonnes pratiques éprouvées par l'industrie, et les erreurs courantes à éviter. C'est la **sagesse collective** accumulée par des milliers d'ingénieurs qui ont déjà fait les erreurs pour vous.

## 🌍 L'approche par cas d'usage : Apprendre de la réalité

### Pourquoi les cas d'usage sont essentiels

**Le problème de l'apprentissage académique :**
```
Formation traditionnelle :
- Concepts isolés
- Exemples simplifiés
- Environnements contrôlés
- Pas de contraintes réelles

Résultat : "Je connais MongoDB mais je ne sais pas comment l'appliquer"
```

**L'apprentissage par cas d'usage :**
```
Cas d'usage réels :
- Contexte complet (business, technique, contraintes)
- Décisions d'architecture avec trade-offs
- Solutions éprouvées en production
- Erreurs courantes et comment les éviter

Résultat : "Je sais exactement quand et comment utiliser MongoDB"
```

### La méthodologie : Du problème à la solution

**Framework de résolution :**

```
1. CONTEXTE BUSINESS
   - Quel problème résoudre ?
   - Quels sont les enjeux ?
   - Quelles sont les contraintes ?

2. EXIGENCES TECHNIQUES
   - Volume de données
   - Patterns d'accès (lecture vs écriture)
   - Latency requirements
   - Throughput requirements
   - Scalability needs
   - Budget constraints

3. DÉCISIONS D'ARCHITECTURE
   - Pourquoi MongoDB (vs alternatives) ?
   - Standalone / Replica Set / Sharded ?
   - Modélisation des données
   - Stratégie d'indexation
   - Caching strategy

4. IMPLÉMENTATION
   - Stack technique
   - Code patterns
   - Configuration optimale

5. RÉSULTATS & LEARNINGS
   - Métriques de performance
   - Challenges rencontrés
   - Solutions appliquées
   - Retour d'expérience
```

Cette approche systématique vous permet de **transférer les learnings** d'un cas d'usage à vos propres projets.

### L'écosystème des cas d'usage MongoDB

**Par domaine applicatif :**

```
┌─────────────────────────────────────────────────────────────┐
│                    MongoDB Use Cases                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  E-COMMERCE                                                 │
│  ├─ Product catalog (millions SKUs)                         │
│  ├─ Shopping cart (high concurrency)                        │
│  ├─ Order management (transactions)                         │
│  ├─ Personalization (recommendations)                       │
│  └─ Inventory tracking (real-time)                          │
│                                                             │
│  CONTENT MANAGEMENT                                         │
│  ├─ CMS / Blog platforms                                    │
│  ├─ Digital asset management                                │
│  ├─ User-generated content                                  │
│  ├─ Multi-tenant SaaS                                       │
│  └─ Version control systems                                 │
│                                                             │
│  REAL-TIME & STREAMING                                      │
│  ├─ IoT / Sensor data                                       │
│  ├─ Real-time analytics                                     │
│  ├─ Event sourcing                                          │
│  ├─ Live dashboards                                         │
│  └─ Chat / Messaging systems                                │
│                                                             │
│  SOCIAL / MOBILE                                            │
│  ├─ Social networks                                         │
│  ├─ Mobile backends                                         │
│  ├─ Gaming (player profiles, leaderboards)                  │
│  ├─ User profiles & preferences                             │
│  └─ Activity feeds                                          │
│                                                             │
│  ENTERPRISE                                                 │
│  ├─ Customer 360 (single view)                              │
│  ├─ Master Data Management (MDM)                            │
│  ├─ Catalog & Product Information Management (PIM)          │
│  ├─ Legacy modernization                                    │
│  └─ Multi-channel integration                               │
│                                                             │
│  ANALYTICS & AI                                             │
│  ├─ Operational analytics                                   │
│  ├─ Log aggregation                                         │
│  ├─ Metrics & monitoring                                    │
│  ├─ Machine Learning features store                         │
│  └─ Vector search / RAG applications                        │
│                                                             │
│  FINANCE & TRADING                                          │
│  ├─ Trading platforms                                       │
│  ├─ Risk management                                         │
│  ├─ Payment processing                                      │
│  ├─ Fraud detection                                         │
│  └─ Regulatory compliance                                   │
└─────────────────────────────────────────────────────────────┘
```

**Chaque cas d'usage a ses propres patterns, compromis et pièges.**

### Les facteurs de décision : Matrice de choix

**Quand MongoDB est le bon choix :**

| Critère | MongoDB excellent | MongoDB acceptable | MongoDB à éviter |
|---------|------------------|-------------------|-----------------|
| **Schema** | Variable, évolutif | Semi-structuré | Strictement relationnel |
| **Relations** | Embedded, peu de JOINs | Références avec lookups limités | Nombreux JOINs complexes |
| **Scaling** | Horizontal requis | Vertical suffisant | Pas de scaling requis |
| **Workload** | Read-heavy ou balanced | Write-heavy modéré | Transactions complexes ACID |
| **Latency** | <100ms requis | <1s acceptable | >1s acceptable |
| **Data size** | TB-PB | GB-TB | <1 GB |
| **Queries** | Document queries | Aggregations simples | SQL complexe existant |
| **Team skills** | MongoDB expertise | NoSQL expérience | SQL-only |

**Exemples de décision :**

✅ **MongoDB est idéal :**
- Catalog produit e-commerce (millions SKUs, schema variable)
- User profiles (données hétérogènes par user)
- IoT sensor data (volume massif, time series)
- Content management (documents flexibles)

⚠️ **MongoDB acceptable mais challengeant :**
- ERP complexe (beaucoup de relations)
- Reporting complexe (nombreuses agrégations)
- Legacy migration (coût élevé)

❌ **MongoDB pas recommandé :**
- Banking core (ACID strict, transactions complexes)
- Warehouse inventory (JOINs complexes en temps réel)
- Système déjà optimal en SQL (pas de raison de migrer)

**Principe fondamental :** MongoDB n'est pas une solution universelle. Le bon choix dépend du contexte.

## 📐 Bonnes pratiques : Les règles d'or

### La pyramide des bonnes pratiques

```
┌─────────────────────────────────────────────────────────┐
│ 1. ARCHITECTURE (Impact maximal)                        │
│    - Design for your access patterns                    │
│    - Choose the right data model                        │
│    - Plan for scale from day one                        │
├─────────────────────────────────────────────────────────┤
│ 2. MODÉLISATION (Foundation)                            │
│    - Embed what you query together                      │
│    - Reference for large or frequently changing data    │
│    - Denormalize strategically                          │
├─────────────────────────────────────────────────────────┤
│ 3. PERFORMANCE (Optimization)                           │
│    - Index your queries                                 │
│    - Use explain() systematically                       │
│    - Monitor and measure continuously                   │
├─────────────────────────────────────────────────────────┤
│ 4. SÉCURITÉ (Non-negotiable)                            │
│    - Authentication always enabled                      │
│    - Principle of least privilege                       │
│    - Encryption in transit and at rest                  │
├─────────────────────────────────────────────────────────┤
│ 5. OPÉRATIONS (Reliability)                             │
│    - Automate everything                                │
│    - Monitor and alert intelligently                    │
│    - Test disaster recovery regularly                   │
└─────────────────────────────────────────────────────────┘
```

### Les 20 règles d'or MongoDB

**Modélisation :**
1. **Model for your queries, not your data** - L'accès prime sur la normalisation
2. **Embed for atomicity** - Si c'est modifié ensemble, c'est stocké ensemble
3. **Reference for flexibility** - Si c'est gros ou change souvent, référencez
4. **Keep documents < 16 MB** - Limite absolue, visez < 1 MB
5. **Denormalize intentionally** - Duplication n'est pas un crime si justifiée

**Index et performance :**
6. **Index everything you query** - Chaque query doit avoir un index
7. **ESR rule for compound indexes** - Equality, Sort, Range
8. **Cover your queries** - Index couvrants = pas de FETCH
9. **Monitor index usage** - Supprimez les index inutilisés
10. **Explain before optimize** - Mesurez, ne devinez pas

**Scalabilité :**
11. **Plan your shard key carefully** - Choix quasi-immutable
12. **High cardinality is crucial** - Shard key avec beaucoup de valeurs distinctes
13. **Avoid hot shards** - Distribution uniforme
14. **Test with production data** - Dev data ≠ prod data

**Sécurité :**
15. **Defense in depth** - Multiples couches de sécurité
16. **Validate everywhere** - Client, app, database
17. **Audit everything** - Logs pour forensics
18. **Encrypt sensitive data** - CSFLE pour données sensibles

**Opérations :**
19. **Automate relentlessly** - Si manuel, c'est fragile
20. **Test your backups** - Untested backup = no backup

### Anti-patterns : Les pièges à éviter

**Architecture :**
```
❌ ANTI-PATTERN : Massive documents (> 10 MB)
   Symptômes : Performance dégradée, mémoire saturée
   Fix : Référencer les gros objets, utiliser GridFS

❌ ANTI-PATTERN : Unbounded arrays
   Symptômes : Documents qui grossissent indéfiniment
   Fix : Limiter la taille ou références

❌ ANTI-PATTERN : Over-normalization (SQL thinking)
   Symptômes : Lookups partout, performance catastrophique
   Fix : Embed ce qui est queryé ensemble

❌ ANTI-PATTERN : Sharding trop tôt
   Symptômes : Complexité inutile
   Fix : Commencer avec Replica Set, sharding quand nécessaire
```

**Queries :**
```
❌ ANTI-PATTERN : $where ou $regex sans ancrage
   Symptômes : Full collection scans
   Fix : Index approprié, regex avec ^

❌ ANTI-PATTERN : N+1 queries
   Symptômes : Latence multipliée par N
   Fix : Aggregation avec $lookup ou embedding

❌ ANTI-PATTERN : Ignorer explain()
   Symptômes : Queries lentes non diagnostiquées
   Fix : explain() systématique en développement
```

**Développement :**
```
❌ ANTI-PATTERN : Connection per request
   Symptômes : Latence énorme, épuisement de connexions
   Fix : Connection pooling

❌ ANTI-PATTERN : No error handling
   Symptômes : Crashes silencieux, data loss
   Fix : Try/catch, retry logic, fallbacks

❌ ANTI-PATTERN : Trust user input
   Symptômes : Injection attacks, data corruption
   Fix : Validation stricte, parameterized queries
```

**Opérations :**
```
❌ ANTI-PATTERN : Manual deployments
   Symptômes : Configuration drift, errors
   Fix : Infrastructure as Code

❌ ANTI-PATTERN : No monitoring
   Symptômes : Blind flying, incidents non détectés
   Fix : Monitoring complet avec alerting

❌ ANTI-PATTERN : Single region
   Symptômes : Pas de disaster recovery
   Fix : Multi-région au moins pour prod
```

**Chacun de ces anti-patterns coûte cher en production.** Apprenez-les pour les éviter.

## 🔧 Dépannage : Résoudre les problèmes réels

### La méthodologie de troubleshooting

**Framework systématique :**

```
┌──────────────────────────────────────────────────────┐
│         Troubleshooting Methodology                  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  1. DEFINE THE PROBLEM                               │
│     - What is broken? (symptoms)                     │
│     - When did it start? (timeline)                  │
│     - What changed? (deployments, traffic)           │
│     - Impact? (users affected, severity)             │
│                                                      │
│  2. GATHER DATA                                      │
│     - Logs (application + MongoDB)                   │
│     - Metrics (CPU, RAM, disk, network, queries)     │
│     - Recent changes (code, config, data)            │
│     - User reports                                   │
│                                                      │
│  3. FORM HYPOTHESES                                  │
│     - Most likely causes (based on symptoms)         │
│     - Ranked by probability                          │
│     - Testable predictions                           │
│                                                      │
│  4. TEST HYPOTHESES                                  │
│     - One at a time                                  │
│     - Measure impact                                 │
│     - Isolate variables                              │
│                                                      │
│  5. IMPLEMENT FIX                                    │
│     - Quick mitigation (if needed)                   │
│     - Root cause fix                                 │
│     - Validation                                     │
│                                                      │
│  6. DOCUMENT & PREVENT                               │
│     - Postmortem (what, why, how to prevent)         │
│     - Runbook update                                 │
│     - Monitoring improvement                         │
└──────────────────────────────────────────────────────┘
```

### Problèmes courants et diagnostics

**Catégories de problèmes :**

**1. Performance dégradée**
```
Symptômes :
- Queries lentes (latency élevée)
- Throughput réduit
- Timeouts

Diagnostics :
- explain() sur queries lentes
- Vérifier index (missing, unused)
- Vérifier ressources (CPU, RAM, disk)
- Vérifier locks (currentOp)

Solutions courantes :
- Ajouter index manquants
- Optimiser queries
- Scale up/out
```

**2. Erreurs de connexion**
```
Symptômes :
- Connection timeout
- Connection refused
- Too many connections

Diagnostics :
- Vérifier network (firewalls, DNS)
- Vérifier maxConnections
- Vérifier connection pool config
- Vérifier MongoDB logs

Solutions courantes :
- Ajuster pool size
- Augmenter maxConnections
- Vérifier network config
- Restart MongoDB (last resort)
```

**3. Problèmes de réplication**
```
Symptômes :
- Replication lag
- Secondary down
- Election fréquentes

Diagnostics :
- rs.status() et rs.printReplicationInfo()
- Vérifier network latency entre nodes
- Vérifier charge sur secondary
- Vérifier oplog size

Solutions courantes :
- Augmenter oplog size
- Réduire charge sur secondary
- Améliorer network
- Resynch si nécessaire
```

**4. Problèmes de disque**
```
Symptômes :
- Disk space warning
- Slow queries (disk I/O)
- High disk utilization

Diagnostics :
- df -h (espace disque)
- iostat (IOPS, throughput)
- Vérifier data size et index size
- Vérifier oplog size

Solutions courantes :
- Compacter collections
- Supprimer index inutilisés
- Augmenter disque
- Archive old data
```

**5. Problèmes applicatifs**
```
Symptômes :
- Data inconsistency
- Duplicate key errors
- Validation errors

Diagnostics :
- Vérifier application logs
- Vérifier schema validation
- Vérifier indexes uniques
- Vérifier transactions

Solutions courantes :
- Fix application logic
- Add/update validation rules
- Handle errors properly
- Implement retry logic
```

### Les outils de diagnostic

**Commandes essentielles :**
```javascript
// Status général
db.serverStatus()
db.currentOp()

// Réplication
rs.status()
rs.printReplicationInfo()

// Index et queries
db.collection.stats()
db.collection.getIndexes()
db.collection.explain()

// Profiling
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find().sort({ ts: -1 })

// Sharding
sh.status()
db.collection.getShardDistribution()
```

**Outils externes :**
- **mongostat** : Real-time stats
- **mongotop** : Collection-level statistics
- **mtools** : Log analysis
- **Percona Monitoring** : Comprehensive monitoring
- **FTDC files** : Diagnostic data capture

## 📋 Prérequis

Cette partie s'adresse à des **praticiens MongoDB** ayant :

### Connaissances MongoDB requises
- ✅ **Compréhension globale** des Parties 1-8
- ✅ Maîtrise des fondamentaux (CRUD, index, agrégations)
- ✅ Connaissance de l'architecture (réplication, sharding)
- ✅ Expérience avec au moins un projet MongoDB
- ✅ Familiarité avec les concepts de performance

**Note :** Cette partie est **transversale** - pas besoin de maîtriser toutes les parties précédentes en profondeur, mais une vue d'ensemble est essentielle.

### Expérience pratique
- 💼 Au moins un projet MongoDB de bout en bout
- 💼 Expérience de debugging en développement
- 💼 Compréhension des problèmes courants
- 💼 Curiosité pour les patterns d'architecture

### État d'esprit
- 🎯 **Pragmatisme** : "What works in practice"
- 🎯 **Ouverture** : Apprendre des erreurs des autres
- 🎯 **Synthèse** : Connecter les concepts
- 🎯 **Application** : Transférer les learnings à vos projets

**Niveau cible :** Intermédiaire à avancé (Parties 1-4 suffisent, 5-8 sont un plus).

## 🎓 Objectifs d'apprentissage

À la fin de cette partie, vous serez capable de :

### Compétences en cas d'usage

**Analyse de cas d'usage :**
- ✅ **Identifier** les patterns d'accès dans un contexte business
- ✅ **Choisir** MongoDB vs alternatives selon le contexte
- ✅ **Concevoir** l'architecture appropriée pour un use case
- ✅ **Anticiper** les challenges potentiels
- ✅ **Évaluer** les trade-offs architecture/performance/coût

**Architectures de référence :**
- ✅ **Comprendre** les architectures éprouvées par use case
- ✅ **Adapter** ces architectures à vos besoins
- ✅ **Justifier** vos choix d'architecture
- ✅ **Documenter** les décisions et trade-offs

**Domaines applicatifs :**
- ✅ **E-commerce** : Catalog, cart, orders, inventory
- ✅ **Content management** : CMS, blogs, DAM
- ✅ **IoT & Real-time** : Sensor data, analytics, event sourcing
- ✅ **Social & Mobile** : Profiles, feeds, chat
- ✅ **Enterprise** : Customer 360, MDM, legacy modernization
- ✅ **Analytics & AI** : Operational analytics, ML features, RAG

### Compétences en bonnes pratiques

**Patterns et anti-patterns :**
- ✅ **Reconnaître** les patterns efficaces
- ✅ **Identifier** les anti-patterns avant qu'ils causent des problèmes
- ✅ **Refactorer** du code existant pour suivre les best practices
- ✅ **Évaluer** du code MongoDB (code review)

**Design patterns :**
- ✅ **Appliquer** les 9 patterns de modélisation MongoDB
- ✅ **Choisir** le pattern approprié selon le contexte
- ✅ **Combiner** plusieurs patterns dans une même application
- ✅ **Documenter** l'utilisation des patterns

**Performance patterns :**
- ✅ **Implémenter** des stratégies de caching efficaces
- ✅ **Optimiser** les queries selon les patterns d'accès
- ✅ **Concevoir** pour la scalabilité dès le départ
- ✅ **Monitorer** et améliorer continuellement

**Security patterns :**
- ✅ **Appliquer** defense in depth
- ✅ **Prévenir** les vulnérabilités courantes
- ✅ **Auditer** la sécurité d'une application
- ✅ **Implémenter** l'encryption appropriée

### Compétences en troubleshooting

**Diagnostic systématique :**
- ✅ **Appliquer** une méthodologie de troubleshooting
- ✅ **Utiliser** les outils de diagnostic (mongostat, explain, logs)
- ✅ **Interpréter** les métriques et logs
- ✅ **Former** des hypothèses testables
- ✅ **Isoler** la cause racine

**Problèmes courants :**
- ✅ **Diagnostiquer** et résoudre les problèmes de performance
- ✅ **Gérer** les problèmes de connexion
- ✅ **Résoudre** les issues de réplication
- ✅ **Traiter** les problèmes de disque et ressources
- ✅ **Debugger** les erreurs applicatives

**Prévention :**
- ✅ **Écrire** des postmortems constructifs
- ✅ **Créer** des runbooks opérationnels
- ✅ **Améliorer** le monitoring pour détecter les problèmes tôt
- ✅ **Partager** les learnings avec l'équipe

### Compétences transversales

**Decision making :**
- ✅ **Évaluer** les trade-offs de façon systématique
- ✅ **Justifier** les décisions techniques
- ✅ **Communiquer** les compromis aux stakeholders
- ✅ **Documenter** les choix d'architecture

**Continuous improvement :**
- ✅ **Identifier** les opportunités d'amélioration
- ✅ **Mesurer** l'impact des changements
- ✅ **Apprendre** des erreurs (postmortems)
- ✅ **Partager** les connaissances (documentation, mentoring)

## 📚 Vue d'ensemble des modules

Cette partie contient **3 modules complémentaires et transversaux** :

### Module 20 : Cas d'Usage et Architectures
**Durée estimée : 16-20 heures**

Exploration approfondie des cas d'usage réels avec architectures de référence.

#### 20.1 Vue d'ensemble des cas d'usage
**Durée : 2 heures**

Introduction aux principaux domaines d'application de MongoDB.

**Ce que vous maîtriserez :**
- Mapping use cases → MongoDB capabilities
- Facteurs de décision (MongoDB vs alternatives)
- Matrice de choix par contexte

---

#### 20.2-20.3 E-commerce et Catalog
**Durée : 3-4 heures**

Architecture e-commerce complète.

**Cas d'usage couverts :**
- Product catalog (millions SKUs, faceted search)
- Shopping cart (high concurrency, session management)
- Order management (transactions, workflow)
- Inventory tracking (real-time, consistency)
- Recommendations (personalization)

**Architecture de référence :**
```
Product Catalog Service (MongoDB)
  - Sharded by category or product_id
  - Atlas Search pour faceted search
  - Cache layer (Redis) pour hot products

Shopping Cart Service (MongoDB)
  - Document per user
  - Embedded line items
  - TTL indexes pour abandoned carts

Order Management Service (MongoDB + PostgreSQL)
  - MongoDB pour orders (flexible schema)
  - PostgreSQL pour payments (ACID strict)
  - Event bus (Kafka) pour communication

Inventory Service (MongoDB)
  - Time Series pour stock movements
  - Transactions pour updates critiques
```

**Learnings :**
- Quand utiliser transactions vs atomicité document
- Trade-offs consistency vs performance
- Sharding strategy pour catalog

---

#### 20.4 Content Management Systems
**Durée : 2-3 heures**

CMS et gestion de contenu.

**Cas d'usage :**
- Blog/News platforms
- Digital Asset Management (DAM)
- User-generated content
- Multi-tenant SaaS

**Patterns clés :**
- Schema flexible pour types de contenu variés
- Version control (pattern Schema Versioning)
- GridFS pour assets > 16 MB
- Workflow approval (embedded status)

---

#### 20.5 IoT et Time Series
**Durée : 3-4 heures**

Données temporelles et IoT.

**Cas d'usage :**
- Sensor data (millions de points/sec)
- Metrics et monitoring
- Log aggregation
- Predictive maintenance

**Architecture :**
```
Ingestion Layer (Kafka)
  ↓
MongoDB Time Series Collections
  - Automatic compression (90% savings)
  - TTL for data expiration
  - Pre-aggregation (rollups)
  ↓
Analytics Layer (Spark)
  ↓
Dashboards (Grafana, Atlas Charts)
```

**Optimisations :**
- Batching inserts (bulk writes)
- Pre-aggregation (hourly/daily rollups)
- Archival strategy (Data Lake pour old data)

---

#### 20.6 Social et Mobile
**Durée : 2-3 heures**

Applications sociales et mobiles.

**Cas d'usage :**
- User profiles (flexible schema)
- Activity feeds (time-ordered)
- Chat/Messaging (real-time)
- Leaderboards (gaming)

**Patterns :**
- Fan-out on write vs fan-out on read
- Change Streams pour real-time updates
- Realm Sync pour mobile offline-first

---

#### 20.7 Customer 360 et MDM
**Durée : 2-3 heures**

Vue unifiée client et Master Data Management.

**Challenge :** Agréger données de sources multiples (CRM, ERP, Web, Mobile).

**Architecture :**
```
Multiple Data Sources
  ↓
ETL Pipeline (Kafka Connect, Airflow)
  ↓
MongoDB (Master Data)
  - Flexible schema pour sources hétérogènes
  - Reference architecture
  - Denormalization stratégique
  ↓
Analytics & BI (Connector for BI)
```

---

#### 20.8-20.9 Analytics et AI/ML
**Durée : 3-4 heures**

MongoDB pour analytics et intelligence artificielle.

**Use cases :**
- Operational analytics (real-time)
- Feature store pour ML
- Vector Search pour RAG
- Recommendations engines

**Architecture RAG (Retrieval-Augmented Generation) :**
```
Documents → Embeddings (OpenAI) → MongoDB (Vector Search)
                                         ↓
User Query → Embedding → Similar docs → LLM → Response
```

---

**Pourquoi ce module est crucial :** Voir MongoDB appliqué à des cas réels avec architectures complètes vous permet de transférer ces patterns à vos projets.

---

### Module 21 : Bonnes Pratiques et Anti-patterns
**Durée estimée : 12-16 heures**

Distillation de la sagesse collective : ce qui marche et ce qui échoue.

#### 21.1 Bonnes pratiques de modélisation
**Durée : 3-4 heures**

Synthèse des patterns de modélisation.

**Les 9 patterns revisités :**
1. **Embedded** : Données toujours lues ensemble
2. **Subset** : Limiter la taille en embarquant un subset
3. **Extended Reference** : Embed les champs souvent accédés
4. **Outlier** : Traiter différemment les cas extrêmes
5. **Computed** : Pré-calculer les agrégations coûteuses
6. **Bucket** : Grouper les time series
7. **Schema Versioning** : Gérer l'évolution du schéma
8. **Attribute** : Schema flexible avec paires key-value
9. **Polymorphic** : Plusieurs types dans une collection

**Quand utiliser chaque pattern :** Matrice de décision complète.

---

#### 21.2-21.3 Anti-patterns (modélisation et queries)
**Durée : 3-4 heures**

Les erreurs courantes et comment les éviter.

**Top 10 anti-patterns :**
1. Massive documents (> 10 MB)
2. Unbounded arrays
3. Over-normalization
4. $where et $regex non ancrés
5. N+1 queries
6. Missing indexes
7. Ignoring explain()
8. Connection per request
9. No error handling
10. Sharding trop tôt (ou trop tard)

**Pour chaque anti-pattern :**
- Symptômes reconnaissables
- Impact sur performance/scalabilité
- Comment le détecter
- Comment le corriger
- Comment le prévenir

---

#### 21.4 Bonnes pratiques de performance
**Durée : 2-3 heures**

Synthèse des optimisations.

**Checklist performance :**
- [ ] Index pour chaque query pattern
- [ ] Compound indexes suivant ESR rule
- [ ] Covering indexes quand possible
- [ ] Projections pour limiter data transfer
- [ ] Aggregation pipeline optimisé ($match tôt)
- [ ] Working set tient en RAM
- [ ] Connection pooling configuré
- [ ] Caching pour hot data
- [ ] Monitoring continu

---

#### 21.5 Bonnes pratiques de sécurité
**Durée : 2-3 heures**

Defense in depth appliqué.

**Les 8 couches de sécurité :**
1. Network (firewall, VPC)
2. Authentication (SCRAM, x.509)
3. Authorization (RBAC granulaire)
4. Encryption in transit (TLS 1.2+)
5. Encryption at rest (AES-256)
6. Audit logging
7. Input validation
8. Backups (chiffrés)

---

#### 21.6 Bonnes pratiques opérationnelles
**Durée : 2-3 heures**

Excellence opérationnelle.

**Principes :**
- Infrastructure as Code (Terraform)
- Monitoring et alerting complets
- Automated backups et tests de restore
- Disaster recovery plan testé
- Runbooks pour incidents courants
- Postmortems après incidents
- Documentation à jour

---

**Pourquoi ce module est crucial :** Apprendre des erreurs des autres vous fait gagner des années d'expérience.

---

### Module 22 : Dépannage et Résolution de Problèmes
**Durée estimée : 14-18 heures**

Diagnostics et résolution systématiques.

#### 22.1 Méthodologie de troubleshooting
**Durée : 2-3 heures**

Framework structuré pour résoudre les problèmes.

**Les 6 étapes :**
1. Define (quoi, quand, impact)
2. Gather data (logs, metrics)
3. Form hypotheses (causes probables)
4. Test (valider hypothèses)
5. Fix (mitigation + root cause)
6. Document (postmortem, prévention)

**Tools du troubleshooter :**
- Logs (MongoDB + application)
- Metrics (CPU, RAM, disk, network)
- mongostat, mongotop
- explain(), profiler
- currentOp()
- FTDC files

---

#### 22.2 Diagnostics de performance
**Durée : 3-4 heures**

Identifier et résoudre les problèmes de performance.

**Symptômes courants :**
- Queries lentes (> 100ms)
- High latency (p99 > 1s)
- Low throughput (< 1000 ops/sec)
- CPU/RAM/Disk saturés

**Diagnostics par couche :**
- **Application** : APM tools, profiling
- **MongoDB** : explain(), profiler, slow query log
- **Système** : top, iostat, vmstat
- **Network** : netstat, tcpdump

**Solutions types :**
- Ajouter/optimiser index
- Refactorer queries
- Augmenter ressources
- Optimiser schema

---

#### 22.3-22.4 Problèmes réseau et réplication
**Durée : 3-4 heures**

Résoudre les issues de connectivité et réplication.

**Problèmes réseau :**
- Connection timeouts
- Too many connections
- High network latency

**Problèmes réplication :**
- Replication lag
- Secondary can't keep up
- Elections fréquentes
- Oplog insuffisant

**Diagnostics et solutions.**

---

#### 22.5 Problèmes de disque et ressources
**Durée : 2-3 heures**

Gérer les contraintes de ressources.

**Problèmes courants :**
- Disk space full
- High disk I/O
- RAM insuffisante
- CPU saturé

**Stratégies de résolution :**
- Compression
- Archival (Data Lake)
- Compaction
- Scaling (vertical/horizontal)

---

#### 22.6 Problèmes applicatifs
**Durée : 2-3 heures**

Debugger les erreurs applicatives.

**Erreurs fréquentes :**
- Duplicate key errors
- Validation errors
- Transaction aborts
- Deadlocks

**Root causes et fixes.**

---

#### 22.7 Recovery et restauration
**Durée : 2-3 heures**

Récupérer d'un incident majeur.

**Scénarios :**
- Data corruption
- Accidental deletion
- Disaster (datacenter down)
- Ransomware attack

**Procédures de recovery :**
- Point-in-Time Restore
- Oplog replay
- Backup restoration
- Disaster recovery cross-region

---

**Pourquoi ce module est crucial :** Savoir diagnostiquer et résoudre rapidement les problèmes en production est ce qui différencie un junior d'un senior.

## 🎯 Progression pédagogique

Cette partie suit une logique **comprendre → appliquer → diagnostiquer** :

```
Cas d'usage → Best practices → Troubleshooting
```

### Semaines 1-2 : Cas d'usage et architectures
**Focus : Apprendre des architectures réelles**

**Semaine 1 : Use cases principaux**
- Jours 1-2 : E-commerce (catalog, cart, orders)
- Jours 3-4 : CMS et content management
- Jours 5-7 : IoT et time series

**Semaine 2 : Use cases avancés**
- Jours 1-3 : Social, mobile, gaming
- Jours 4-5 : Customer 360, MDM
- Jours 6-7 : Analytics, AI/ML, RAG

**Livrables :**
- Analyse de 5+ cas d'usage
- Architecture diagrams
- Décisions d'architecture justifiées
- Identification de patterns réutilisables

---

### Semaines 3-4 : Bonnes pratiques et anti-patterns
**Focus : Do's and Don'ts**

**Semaine 3 : Best practices**
- Jours 1-2 : Modélisation (9 patterns)
- Jours 3-4 : Performance optimization
- Jours 5-7 : Security et operations

**Semaine 4 : Anti-patterns**
- Jours 1-3 : Anti-patterns modélisation et queries
- Jours 4-5 : Anti-patterns développement
- Jours 6-7 : Anti-patterns opérationnels

**Livrables :**
- Checklist des best practices
- Liste des anti-patterns avec fixes
- Code review d'un projet existant
- Refactoring d'un anti-pattern identifié

---

### Semaines 5-6 : Troubleshooting
**Focus : Diagnostics et résolution**

**Semaine 5 : Méthodologie et performance**
- Jours 1-2 : Framework de troubleshooting
- Jours 3-7 : Diagnostics de performance (hands-on)

**Semaine 6 : Problèmes spécifiques**
- Jours 1-2 : Réplication et réseau
- Jours 3-4 : Disque et ressources
- Jours 5-7 : Recovery et disaster response

**Livrables :**
- Runbooks pour problèmes courants
- 3+ problèmes diagnostiqués et résolus
- Postmortem d'un incident simulé
- Amélioration du monitoring

---

**Rythme recommandé :** 2-3 heures par jour avec pratique sur cas réels.

## 🧠 Principes de pensée transversale

### 1. Context is king

> Il n'y a pas de "bonne" architecture, seulement des architectures appropriées au contexte.

**Application :** Toujours commencer par comprendre les exigences business et techniques avant de concevoir.

### 2. Trade-offs are unavoidable

> Chaque décision a un coût. L'art est de choisir consciemment.

**Application :** Documentez vos trade-offs. "On a choisi X parce que Y, sachant que Z est le compromis."

### 3. Simple is better than complex

> La solution la plus simple qui résout le problème est la meilleure.

**Application :** Résistez à la tentation de sur-ingénierie. KISS.

### 4. Learn from failures (yours and others')

> Les erreurs sont des opportunités d'apprentissage, pas des échecs.

**Application :** Postmortems blameless, documentation des anti-patterns.

### 5. Patterns accelerate, but don't constrain

> Les patterns sont des guides, pas des dogmes.

**Application :** Adaptez les patterns à votre contexte, ne les appliquez pas aveuglément.

### 6. Measure, don't assume

> Les données battent les opinions.

**Application :** Baseez vos décisions sur des métriques, pas sur des intuitions.

## 🚦 Validation des acquis

Avant de passer à la Partie 10, vous devez maîtriser :

### Checklist Cas d'usage
- [ ] Je peux analyser un use case et proposer une architecture MongoDB
- [ ] Je connais les architectures de référence pour 5+ domaines
- [ ] Je peux justifier le choix MongoDB vs alternatives
- [ ] Je sais adapter les patterns à mes besoins spécifiques
- [ ] Je comprends les trade-offs de chaque choix

### Checklist Bonnes pratiques
- [ ] Je connais les 9 patterns de modélisation et quand les utiliser
- [ ] Je peux identifier 10+ anti-patterns courants
- [ ] J'ai une checklist de best practices pour chaque aspect
- [ ] Je peux faire une code review MongoDB compétente
- [ ] J'applique systématiquement les best practices

### Checklist Troubleshooting
- [ ] J'ai une méthodologie structurée de diagnostic
- [ ] Je peux diagnostiquer un problème de performance
- [ ] Je sais utiliser tous les outils de diagnostic
- [ ] J'ai résolu 5+ problèmes réels de production
- [ ] Je peux écrire un runbook opérationnel
- [ ] J'ai conduit un postmortem complet

### Checklist Transversale
- [ ] Je peux connecter les concepts entre les différentes parties
- [ ] Je sais transférer les learnings d'un cas d'usage à l'autre
- [ ] Je peux former des collègues sur ces pratiques
- [ ] Je contribue activement à la documentation
- [ ] J'améliore continuellement mes pratiques

**Objectif :** Cocher 80%+ de ces cases.

## 🎯 Projet de synthèse : Architecture complète

### Projet : Design et troubleshooting complet
**Durée : 30-40 heures**

**Objectif :** Concevoir, implémenter et opérer une architecture complète basée sur un cas d'usage réel.

**Phase 1 : Analyse et design (10h)**
1. Analyser un cas d'usage fourni (e-commerce, IoT, ou social)
2. Définir les exigences (fonctionnelles, non-fonctionnelles)
3. Concevoir l'architecture complète
4. Justifier tous les choix (modélisation, indexation, sharding, etc.)
5. Identifier les anti-patterns potentiels et comment les éviter
6. Documenter les trade-offs

**Phase 2 : Implémentation (15h)**
7. Implémenter l'architecture (IaC + application)
8. Appliquer les best practices de code
9. Tests (unit, integration, load)
10. Documentation complète

**Phase 3 : Operations et troubleshooting (15h)**
11. Déployer sur environnement de staging
12. Monitoring et alerting complets
13. Simuler 3 incidents différents
14. Diagnostiquer et résoudre les incidents
15. Écrire runbooks et postmortems

**Livrables :**
- Document d'architecture (avec diagrammes)
- Code complet et documenté
- Tests avec >80% coverage
- Infrastructure as Code
- Monitoring dashboards
- 3 runbooks opérationnels
- 3 postmortems d'incidents
- Présentation des learnings (30 min)

**Critères de validation :**
- ✅ Architecture justifiée et documentée
- ✅ Best practices appliquées partout
- ✅ Aucun anti-pattern majeur
- ✅ Performance validée (load tests)
- ✅ Incidents diagnostiqués en <30 min
- ✅ Documentation professionnelle
- ✅ Présentation claire des learnings

Ce projet démontre une maîtrise transversale de MongoDB.

## 📊 Matrice de maturité MongoDB

| Dimension | Junior | Intermédiaire | Senior | Expert |
|-----------|--------|---------------|--------|--------|
| **Architecture** | CRUD basique | Modélisation correcte | Architectures complexes | Architectures à l'échelle |
| **Patterns** | N'en connaît aucun | Utilise 2-3 patterns | Utilise 5+ patterns | Crée de nouveaux patterns |
| **Anti-patterns** | Ne les voit pas | Reconnaît après coup | Anticipe et évite | Forme les autres |
| **Troubleshooting** | Panic mode | Debug basique | Diagnostic structuré | Root cause expert |
| **Best practices** | Ignore | Applique parfois | Applique systématiquement | Enseigne et améliore |
| **Use cases** | Connait 1-2 | Connait 3-5 | Connait 8+ | Adapte à tout contexte |

**Objectif de cette partie :** Vous faire passer d'intermédiaire à senior/expert.

## 🌟 Conseils de praticien

### 1. Start with the user, not the database
Concevez pour les besoins utilisateurs, pas pour la technologie.

### 2. Document your "why", not just your "what"
Les futurs mainteneurs doivent comprendre vos décisions.

### 3. Every production issue is a learning opportunity
Postmortem blameless, focus sur la prévention.

### 4. Share knowledge aggressively
La meilleure documentation est celle qui est utilisée. Partagez vos runbooks.

### 5. Stay curious
Les meilleures architectures viennent de ceux qui explorent et expérimentent.

### 6. Simplify ruthlessly
Si vous ne pouvez pas expliquer simplement, vous n'avez pas compris.

## 📚 Ressources complémentaires

### Études de cas officielles
- [MongoDB Case Studies](https://www.mongodb.com/customers)
- Architecture blogs (eBay, Uber, LinkedIn)

### Best practices
- [MongoDB Best Practices](https://www.mongodb.com/basics/best-practices)
- Building with Patterns series

### Troubleshooting
- MongoDB Support knowledge base
- Community forums (Stack Overflow, MongoDB Community)

### Livres recommandés
- *MongoDB Applied Design Patterns* par Rick Copeland
- *50 Tips and Tricks for MongoDB Developers* par Kristina Chodorow

## 🚀 Et après ?

Une fois cette partie maîtrisée, vous serez un **praticien MongoDB complet**. Vous saurez :

- Concevoir des architectures MongoDB pour n'importe quel use case
- Appliquer les best practices systématiquement
- Éviter les anti-patterns courants
- Diagnostiquer et résoudre les problèmes en production
- Apprendre de l'expérience collective

La **Partie 10** conclura votre formation avec une synthèse complète et les perspectives d'évolution.

**Félicitations d'être arrivé ici. Vous êtes maintenant armé de la connaissance théorique ET pratique pour exceller avec MongoDB.**

---

**Prêt à synthétiser tout ce que vous avez appris ? C'est parti ! 🎯**

---

**Prochaine étape :** [Module 20 - Cas d'Usage et Architectures →](/20-cas-usage-architectures/README.md)

---

*💡 Citation du jour : "Experience is simply the name we give our mistakes." - Oscar Wilde*

⏭️ [Module 20 - Cas d'Usage et Architectures →](/20-cas-usage-architectures/README.md)
