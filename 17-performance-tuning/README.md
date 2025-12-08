🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 17 : Performance et Tuning

## Introduction

L'optimisation des performances MongoDB en environnement de production représente un défi multidimensionnel nécessitant une compréhension approfondie de l'architecture distribuée, du moteur de stockage WiredTiger, des patterns d'accès aux données et des contraintes matérielles. Ce chapitre s'adresse aux experts MongoDB responsables de systèmes critiques où chaque milliseconde compte et où la scalabilité horizontale doit être maîtrisée.

## Contexte et Enjeux de Performance

### Défis Spécifiques à MongoDB

Contrairement aux bases de données relationnelles traditionnelles, MongoDB présente des caractéristiques uniques qui influencent directement les stratégies d'optimisation :

**Architecture document-oriented**
- Les documents BSON peuvent contenir des structures imbriquées complexes nécessitant des stratégies de parsing et d'indexation spécifiques
- La limite de 16 Mo par document impose des contraintes architecturales fondamentales
- Le coût de désérialisation BSON peut devenir significatif sur des volumes élevés

**Distribution et réplication**
- La cohérence éventuelle (eventual consistency) dans les replica sets introduit des compromis performance/cohérence
- Le mécanisme de propagation via l'oplog peut créer des latences dans certains scénarios
- Les read preferences influencent directement la distribution de charge et la fraîcheur des données

**Sharding horizontal**
- Le choix de la shard key est irréversible et a un impact permanent sur les performances
- Les jumbo chunks non-splittables peuvent créer des déséquilibres critiques
- Les requêtes scatter-gather pénalisent fortement les performances

### Métriques de Performance Clés

Pour établir une stratégie d'optimisation efficace, il est essentiel de surveiller et d'analyser les métriques suivantes :

#### Métriques de requêtes
- **Latence moyenne et percentiles (P50, P95, P99)** : Indicateurs essentiels pour identifier les requêtes problématiques
- **Throughput (ops/sec)** : Mesure de la capacité de traitement
- **Query execution time** : Temps d'exécution incluant le planning et l'exécution
- **Index utilization ratio** : Pourcentage de requêtes utilisant efficacement les index

#### Métriques système
- **Working Set Size** : Portion de données fréquemment accédées devant résider en RAM
- **Cache hit ratio** : Ratio de lectures satisfaites par le cache WiredTiger
- **Disk I/O patterns** : IOPS, latence de lecture/écriture, patterns séquentiels vs aléatoires
- **Connection pool saturation** : Nombre de connexions actives vs disponibles

#### Métriques de réplication
- **Replication lag** : Retard entre primary et secondaries
- **Oplog window** : Durée couverte par l'oplog
- **Write concern acknowledgment time** : Temps d'acquittement des écritures

#### Métriques de sharding
- **Chunk migration rate** : Fréquence et durée des migrations
- **Data distribution skew** : Déséquilibre de distribution des données
- **Scatter-gather query ratio** : Proportion de requêtes nécessitant une interrogation multi-shards

## Méthodologie d'Analyse de Performance

### Approche Top-Down

L'analyse de performance suit une méthodologie structurée en plusieurs phases :

#### Phase 1 : Observation et Mesure
```
1. Établir une baseline de performance
2. Identifier les patterns d'utilisation
3. Capturer les métriques système et applicatives
4. Profiler les requêtes lentes
```

#### Phase 2 : Analyse et Corrélation
```
1. Corréler les pics de latence avec les métriques système
2. Identifier les patterns de requêtes problématiques
3. Analyser la distribution de charge dans un cluster
4. Évaluer l'efficacité de la topologie actuelle
```

#### Phase 3 : Hypothèses et Expérimentation
```
1. Formuler des hypothèses sur les causes racines
2. Prioriser les optimisations par impact potentiel
3. Tester les modifications en environnement isolé
4. Mesurer l'impact réel en production avec des déploiements progressifs
```

#### Phase 4 : Validation et Monitoring Continu
```
1. Valider l'amélioration des métriques cibles
2. Surveiller les effets secondaires potentiels
3. Documenter les changements et leurs impacts
4. Ajuster le monitoring pour détecter les régressions
```

### Outils d'Analyse Avancés

#### Profiling Multi-niveaux

**Database Profiler**
- Activation sélective par niveau (0: off, 1: slow queries, 2: all operations)
- Filtrage par seuil de latence configurable
- Analyse des execution stats détaillées

**Système de profiling applicatif**
- Instrumentation APM (Application Performance Monitoring)
- Distributed tracing pour les opérations multi-services
- Corrélation entre métriques applicatives et base de données

#### Analyse avec explain()

Le système explain() offre trois modes avec différents niveaux de détail :

**queryPlanner** : Analyse statique du plan d'exécution
- Stratégie de sélection d'index
- Estimation du nombre de documents examinés
- Présence de rejectedPlans

**executionStats** : Statistiques d'exécution réelles
- Documents examinés vs retournés (ratio critique)
- Temps d'exécution par stage
- Working memory utilisée

**allPlansExecution** : Comparaison de tous les plans candidats
- Scores de performance de chaque plan
- Raisons du rejet des plans alternatifs
- Coût d'exécution estimé vs réel

## Dimensions d'Optimisation

### 1. Optimisation de Modélisation

La modélisation des données est la fondation de toute stratégie de performance. Une mauvaise modélisation ne peut être compensée par l'indexation ou le hardware.

**Principes fondamentaux**
- **Data locality** : Minimiser les accès disques en co-localisant les données fréquemment accédées ensemble
- **Read-Write ratio optimization** : Adapter la stratégie d'embedding vs referencing selon les patterns d'accès
- **Cardinality management** : Gérer efficacement les relations one-to-many avec cardinalité variable
- **Update patterns** : Considérer la fréquence et la nature des mises à jour dans la structure documentaire

**Anti-patterns critiques à éviter**
- Unbounded arrays qui croissent indéfiniment
- Documents approchant la limite de 16 Mo
- Fragmentation excessive due à des modifications fréquentes
- Références circulaires complexes nécessitant de multiples lookups

### 2. Optimisation d'Indexation

L'indexation est l'outil le plus puissant pour améliorer les performances de lecture, mais chaque index a un coût en écriture et en stockage.

**Stratégies avancées**
- **Index covering** : Requêtes satisfaites uniquement par l'index sans accès au document
- **Index intersection** : Utilisation combinée de plusieurs index simples (optimisé depuis MongoDB 2.6)
- **Partial indexes** : Indexation sélective basée sur des prédicats pour réduire la taille
- **Compound index prefix utilization** : Exploitation maximale des index composés

**Considérations de maintenance**
- Impact sur les write operations (chaque index augmente le coût d'écriture de ~10%)
- Fragmentation des index et rebuild periodiques
- Index build strategy (foreground vs background vs rolling)
- Index size monitoring et working set implications

### 3. Optimisation des Requêtes

Au-delà de l'indexation, la construction des requêtes elles-mêmes peut être optimisée.

**Techniques avancées**
- **Query selectivity** : Ordonner les prédicats du plus sélectif au moins sélectif
- **Projection minimale** : Ne récupérer que les champs strictement nécessaires
- **Hint forcing** : Forcer l'utilisation d'un index spécifique dans les cas complexes
- **Aggregation pipeline optimization** : Repositionnement des stages pour early filtering

**Patterns problématiques**
- Négations qui empêchent l'utilisation d'index ($ne, $nin)
- Regular expressions sans ancrage initial
- $where et $expr qui forcent un full collection scan
- Sorts sans index supportant l'ordre de tri

### 4. Optimisation du Framework d'Agrégation

Les pipelines d'agrégation peuvent être extrêmement performants ou désastreux selon leur construction.

**Règles d'optimisation des pipelines**
- **$match early** : Filtrer le plus tôt possible dans le pipeline
- **$project minimization** : Projeter uniquement les champs nécessaires pour les stages suivants
- **Index-supported stages** : Utiliser des stages pouvant exploiter les index ($match, $sort)
- **Allowdisuse consideration** : Gérer la limite de 100 Mo de RAM par stage

**Optimisations automatiques du Query Optimizer**
- Réorganisation automatique de $match avant $project
- Fusion de stages adjacents compatibles
- Pipeline splitting pour exploitation des index
- Predicate pushdown vers les sources de données

### 5. Optimisation Architectural et Infrastructure

#### Configuration WiredTiger

Le moteur de stockage WiredTiger offre de nombreux paramètres ajustables :

**Cache Configuration**
- Cache size (défaut : 50% de RAM - 1 Go, minimum 256 Mo)
- Éviction strategy et dirty ratio thresholds
- Cache pressure monitoring et tuning

**Checkpoint Management**
- Checkpoint interval (défaut : 60 secondes ou 2 GB de journaling)
- Impact sur les performances d'écriture
- Balance entre durabilité et performance

**Compression**
- Algorithmes disponibles : snappy (défaut), zlib, zstd, none
- Trade-off CPU vs disk I/O vs storage
- Compression effectiveness par type de données

#### Dimensionnement Matériel

**RAM Requirements**
- Working set doit tenir intégralement en RAM
- Formule estimative : (Data Size × Access Frequency) + (Index Size) + (OS Cache) + (WiredTiger Cache) + (Connection Overhead)
- Monitoring du page faults pour détecter la saturation

**Storage Considerations**
- SSD NVMe pour les workloads à forte latence-sensibilité
- RAID 10 pour balance performance/redondance
- Separate volumes pour data, journal, et logs
- IOPS requirements estimation basée sur le workload

**CPU et Network**
- CPU scaling pour les aggregations et index builds
- Network bandwidth pour replica sets et sharded clusters
- Network latency impact sur la réplication et les transactions distribuées

### 6. Optimisation de la Topologie

#### Replica Sets

**Configuration optimale**
- Nombre impair de membres votants pour éviter les split-brain
- Geographic distribution pour disaster recovery
- Hidden members pour backups et analytics
- Delayed members pour protection contre les erreurs logiques

**Read Preference Strategy**
- Primary : Cohérence maximale, charge concentrée
- PrimaryPreferred : Failover automatique sur secondary
- Secondary : Décharge le primary, données potentiellement stales
- Nearest : Latence minimale, utile pour applications distribuées

#### Sharded Clusters

**Shard Key Selection**
- High cardinality pour distribution uniforme
- Query isolation pour éviter les scatter-gather
- Write scaling : éviter les monotonic values (hotspots)
- Range vs Hashed : trade-offs entre query targeting et distribution

**Zone Sharding**
- Geographic data locality
- Workload isolation (OLTP vs OLAP)
- Data lifecycle management (hot/warm/cold)

**Balancer Configuration**
- Balancing window configuration pour minimiser l'impact
- Chunk size tuning (défaut : 64 MB)
- Maximum parallel migrations

## Stratégies de Monitoring Continu

### Monitoring Proactif

Un système de monitoring efficace doit être :
- **Temps réel** : Détection des anomalies en cours
- **Prédictif** : Anticipation des problèmes avant qu'ils n'impactent les utilisateurs
- **Contextuel** : Corrélation entre différentes métriques
- **Actionnable** : Alertes avec contexte suffisant pour diagnostic rapide

### Outils de Monitoring

**Solutions natives MongoDB**
- MongoDB Atlas monitoring (cloud)
- MongoDB Ops Manager (on-premise)
- Cloud Manager (legacy)

**Solutions tierces**
- Prometheus + Grafana avec MongoDB exporter
- Datadog, New Relic, AppDynamics
- ELK Stack pour log aggregation et analysis

**Métriques essentielles à monitorer**
```
Performance:
- Query execution times (P50, P95, P99)
- Operations per second
- Index hit ratio

Resources:
- CPU utilization
- Memory usage (resident, virtual, mapped)
- Disk I/O (read/write latency, IOPS)
- Network throughput

Replication:
- Replication lag
- Oplog window
- Election frequency

Sharding:
- Chunk distribution
- Migration queue length
- Scatter-gather ratio
```

### Alerting Strategy

**Niveaux d'alerte**
- **Critical** : Impact utilisateur immédiat, intervention urgente
- **Warning** : Tendance préoccupante, investigation requise
- **Info** : Événement notable, documentation

**Seuils adaptatifs**
- Baselines dynamiques selon les patterns temporels
- Détection d'anomalies par machine learning
- Réduction des false positives par corrélation multi-métriques

## Méthodologie de Test de Performance

### Types de Tests

**Load Testing**
- Simulation de charge normale
- Validation de la capacité nominale
- Identification du point de breaking avant saturation

**Stress Testing**
- Dépassement intentionnel des capacités
- Observation du comportement sous contrainte extrême
- Validation des mécanismes de protection (back-pressure, circuit breakers)

**Endurance Testing**
- Charge soutenue sur longue période
- Détection des memory leaks et resource exhaustion
- Validation de la stabilité long-terme

**Spike Testing**
- Variations brusques de charge
- Validation de l'élasticité et auto-scaling
- Comportement lors des pics métier (Black Friday, etc.)

### Outils de Benchmarking

**YCSB (Yahoo! Cloud Serving Benchmark)**
- Framework standard pour benchmarking NoSQL
- Workloads prédéfinis (A, B, C, D, E, F)
- Extensible pour workloads personnalisés

**POCDriver**
- Outil MongoDB officiel pour proof-of-concepts
- Simulation de patterns d'accès réalistes
- Génération de rapports détaillés

**Custom Load Generators**
- Reproduction fidèle des patterns applicatifs réels
- Utilisation des drivers natifs
- Injection de données représentatives

## Optimisations Avancées Spécifiques

### Optimisation pour Workloads Write-Heavy

**Stratégies**
- Write concern relaxé (w:1) pour workloads non-critiques
- Batching des écritures (insertMany vs insertOne)
- Unordered bulk operations pour parallélisation
- Journal write concern (j:false) si acceptable

**Trade-offs**
- Durabilité vs throughput
- Risque de perte de données en cas de crash
- Complexité de la gestion d'erreur en mode unordered

### Optimisation pour Workloads Read-Heavy

**Stratégies**
- Index covering queries maximales
- Read from secondaries pour distribution de charge
- Caching applicatif (Redis, Memcached)
- Materialized views pour agrégations fréquentes

**Considérations**
- Cohérence des données en lecture sur secondary
- Invalidation de cache lors de mises à jour
- Coût de maintenance des vues matérialisées

### Optimisation pour Workloads Analytiques

**Approches**
- Dedicated hidden secondary pour analytics
- Time-series collections pour données temporelles
- Atlas Data Lake pour requêtes sur données archivées
- Columnstore indexes (prévu futures versions)

### Optimisation Géospatiale

**Index 2dsphere optimization**
- Precision tuning via geospatial index options
- Compound indexes avec attributs non-géospatiaux
- GeoJSON vs legacy coordinate pairs

### Optimisation Text Search

**Atlas Search vs Text Indexes**
- Atlas Search pour recherche full-text avancée (Apache Lucene)
- Text indexes pour recherche simple intégrée
- Trade-offs : fonctionnalités vs complexité vs coût

## Patterns d'Anti-Optimisation

Certaines "optimisations" peuvent paradoxalement dégrader les performances :

**Over-indexing**
- Trop d'index ralentit les écritures sans bénéfice pour les lectures
- Index inutilisés consomment de la RAM précieuse
- Maintenance overhead (compact, rebuild)

**Premature sharding**
- Complexité opérationnelle accrue
- Overhead de coordination entre shards
- Migrations coûteuses si shard key mal choisie

**Excessive denormalization**
- Documents gigantesques difficiles à maintenir
- Updates multiples nécessaires pour cohérence
- Consommation mémoire excessive

**Misconfigured read/write concerns**
- Write concern trop strict pour le use case
- Read concern inadapté aux besoins de cohérence
- Impact sur latency et throughput

## Conclusion du Chapitre

L'optimisation des performances MongoDB est un processus itératif et continu nécessitant :

1. **Compréhension profonde** de l'architecture et des mécanismes internes
2. **Monitoring rigoureux** des métriques pertinentes
3. **Méthodologie scientifique** d'hypothèse-test-validation
4. **Vision holistique** intégrant modélisation, indexation, infrastructure et topologie
5. **Culture de la mesure** : toute optimisation doit être quantifiable

Les sections suivantes de ce chapitre approfondiront chacune de ces dimensions avec des techniques concrètes et des exemples détaillés pour transformer un système MongoDB en une plateforme hautement performante et scalable.

---

**Points clés à retenir :**
- La performance MongoDB résulte de l'optimisation coordonnée de multiples dimensions
- Le monitoring continu est essentiel pour maintenir des performances optimales
- Chaque optimisation représente un trade-off qui doit être mesuré et validé
- La modélisation des données est la fondation : un mauvais modèle ne peut être compensé
- L'approche doit être data-driven, basée sur des métriques réelles et non des suppositions

**Progression du chapitre :**
Les sections suivantes détailleront méthodiquement chaque aspect évoqué dans cette introduction, fournissant les techniques concrètes et les bonnes pratiques pour atteindre l'excellence opérationnelle en production.

⏭️ [Identification des goulots d'étranglement](/17-performance-tuning/01-identification-goulots.md)
