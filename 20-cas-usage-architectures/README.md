🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 20 : Cas d'Usage et Architectures

## Vue d'ensemble

Ce chapitre représente la synthèse pratique de l'ensemble des connaissances acquises tout au long de cette formation MongoDB. Plutôt que de présenter des concepts théoriques isolés, nous allons explorer des **architectures réelles** et des **décisions de conception justifiées** pour différents domaines d'application.

L'objectif est de vous permettre de :
- Comprendre comment les différentes fonctionnalités de MongoDB s'articulent dans des systèmes de production
- Identifier les patterns architecturaux adaptés à chaque contexte métier
- Analyser les compromis (trade-offs) inhérents à chaque décision technique
- Anticiper les défis de scalabilité et de performance propres à chaque domaine

## Méthodologie d'analyse

Pour chaque cas d'usage présenté dans ce chapitre, nous suivrons une approche structurée :

### 1. Contexte métier et contraintes
- Caractéristiques des données (volume, vélocité, variété)
- Exigences fonctionnelles et non-fonctionnelles
- Contraintes de performance (latence, throughput)
- Modèles d'accès aux données (lecture vs écriture)

### 2. Décisions de modélisation
- Choix entre documents imbriqués et références
- Application des patterns de modélisation appropriés
- Gestion des relations et de la dénormalisation
- Justification des compromis effectués

### 3. Stratégies d'indexation
- Index critiques pour les requêtes principales
- Équilibre entre performance en lecture et coût en écriture
- Index spécialisés (texte, géospatial, TTL, etc.)

### 4. Architecture de déploiement
- Configuration Replica Set ou Sharded Cluster
- Stratégies de sharding et choix de shard key
- Dimensionnement et scaling
- Considérations de haute disponibilité

### 5. Patterns d'intégration
- Intégration avec l'écosystème applicatif
- Gestion des événements et synchronisation
- Caching et optimisations

## Principes architecturaux transversaux

Avant d'explorer les cas d'usage spécifiques, rappelons les principes fondamentaux qui guident les décisions architecturales avec MongoDB :

### Principe 1 : Modéliser selon les patterns d'accès
> **"Model your data based on how your application queries and updates it"**

MongoDB n'impose pas de schéma normalisé. La modélisation doit être dictée par les requêtes de l'application, pas par une structure théorique idéale.

**Implications pratiques :**
- Analyser les requêtes les plus fréquentes avant de modéliser
- Privilégier la dénormalisation pour réduire les jointures
- Accepter la duplication de données si elle améliore les performances

### Principe 2 : Optimiser pour les opérations les plus fréquentes
Les systèmes réels sont rarement équilibrés entre lecture et écriture. Identifier le ratio lecture/écriture permet d'optimiser en conséquence.

**Exemples de compromis :**
- **Lecture intensive** : Dénormalisation agressive, index multiples, read replicas
- **Écriture intensive** : Normalisation relative, index minimaux, sharding efficace
- **Mixte** : Compromis documentés et mesurés

### Principe 3 : Anticiper la croissance
Les décisions architecturales doivent tenir compte de l'évolution prévisible du système.

**Facteurs de croissance :**
- Volume de données (storage)
- Nombre de requêtes (throughput)
- Nombre d'utilisateurs (concurrence)
- Complexité fonctionnelle (nouveaux use cases)

### Principe 4 : Privilégier la simplicité quand c'est possible
> **"Don't over-engineer for hypothetical scale"**

Il est tentant de concevoir immédiatement pour des millions d'utilisateurs. Une approche progressive est souvent plus pragmatique.

**Évolution typique :**
1. **Phase 1** : MongoDB standalone (dev/test)
2. **Phase 2** : Replica Set 3 nœuds (production simple)
3. **Phase 3** : Replica Set optimisé avec read preferences
4. **Phase 4** : Sharded Cluster (si nécessaire)

## Matrice de décision architecturale

Le tableau suivant aide à orienter les choix architecturaux selon les caractéristiques du système :

| Caractéristique | MongoDB Standalone | Replica Set | Sharded Cluster |
|-----------------|-------------------|-------------|-----------------|
| **Volume de données** | < 100 GB | < 2-3 TB | > 2 TB |
| **Throughput écriture** | < 1K ops/s | < 10K ops/s | > 10K ops/s |
| **Exigence HA** | Aucune (dev/test) | 99.9% uptime | 99.99%+ uptime |
| **Complexité opérationnelle** | Minimale | Modérée | Élevée |
| **Coût infrastructure** | Minimal | Moyen | Élevé |
| **Latence réseau acceptable** | N/A | < 10ms | < 20ms |

## Patterns architecturaux courants

### Pattern 1 : Architecture OLTP traditionnelle
**Contexte :** Applications transactionnelles avec forte cohérence

```
┌─────────────────┐
│   Application   │
│    (Node.js)    │
└────────┬────────┘
         │
    ┌────┴────┐
    │ Primary │ ←─── Writes
    └────┬────┘
         │
    ┌────┴──────────┐
    │               │
┌───▼───┐     ┌─────▼───┐
│ Sec 1 │     │  Sec 2  │ ←─── Reads (optional)
└───────┘     └─────────┘
```

**Décisions clés :**
- Replica Set 3 nœuds minimum
- Write Concern `majority` pour cohérence forte
- Read Preference `primary` ou `primaryPreferred`
- Transactions multi-documents si nécessaire

### Pattern 2 : Architecture Read-Heavy avec séparation
**Contexte :** Applications avec ratio lecture/écriture élevé (90%+ lectures)

```
┌──────────────┐
│  Write API   │ ──────► Primary
└──────────────┘              │
                         ┌────┴────┐
┌──────────────┐         │         │
│   Read API   │ ─┬──► Sec 1   Sec 2 ◄──────┐
└──────────────┘  │    (analytics) (cache)  │
                  │                         │
┌──────────────┐  │                         │
│   Dashboard  │ ─┴─────────────────────────┘
└──────────────┘
```

**Décisions clés :**
- Read Preference `secondary` ou `nearest`
- Secondary dédié pour analytics (delayed replica optionnel)
- Index optimisés différemment selon les membres
- Write Concern `w:1` acceptable si tolérance légère

### Pattern 3 : Architecture géo-distribuée
**Contexte :** Utilisateurs répartis sur plusieurs régions

```
     Europe                US                  Asia
       │                   │                    │
   ┌───┴───┐           ┌───┴───┐           ┌───┴───┐
   │ Sec 1 │           │Primary│           │ Sec 2 │
   └───────┘           └───────┘           └───────┘
       │                   │                    │
       └───────────────────┴────────────────────┘
              Zone Sharding (optionnel)
```

**Décisions clés :**
- Replica Set multi-région avec membres dans chaque zone
- Read Preference `nearest` pour minimiser latence
- Write Concern adapté à la tolérance de latence
- Zone Sharding pour isolation géographique des données

### Pattern 4 : Architecture Sharded à grande échelle
**Contexte :** Plusieurs TB de données, > 50K ops/s

```
                  ┌──────────────┐
                  │   Mongos     │ ←─── Load Balancer
                  │ (Query Router)
                  └──────┬───────┘
                         │
        ┌────────────────┼───────────────┐
        │                │               │
   ┌────┴─────┐     ┌────┴─────┐    ┌────┴─────┐
   │ Shard 1  │     │ Shard 2  │    │ Shard 3  │
   │(Replica) │     │(Replica) │    │(Replica) │
   └──────────┘     └──────────┘    └──────────┘
        │                │               │
   ┌────┴──────┐   ┌─────┴──────┐  ┌─────┴──────┐
   │Config Srv │   │Config Srv  │  │Config Srv  │
   │(Replica)  │   │ (Replica)  │  │ (Replica)  │
   └───────────┘   └────────────┘  └────────────┘
```

**Décisions clés :**
- Shard key soigneusement choisie (éviter hotspots)
- Config servers en Replica Set
- Mongos colocalisés avec l'application
- Monitoring intensif du balancing

## Checklist de conception

Avant de finaliser une architecture MongoDB, validez les points suivants :

### ✅ Modélisation des données
- [ ] Les patterns d'accès principaux sont identifiés et documentés
- [ ] Le modèle de données optimise pour les requêtes les plus fréquentes
- [ ] Les limites de taille de document (16 Mo) sont respectées
- [ ] Les patterns de modélisation appropriés sont appliqués
- [ ] La stratégie de gestion des relations est définie

### ✅ Indexation
- [ ] Tous les index nécessaires sont identifiés
- [ ] L'impact des index sur les écritures est évalué
- [ ] Les index couvrants (covered queries) sont utilisés quand possible
- [ ] Une stratégie de maintenance des index est définie

### ✅ Performance et scaling
- [ ] Le dimensionnement initial est basé sur des métriques réelles ou estimées
- [ ] Une stratégie de scaling (vertical puis horizontal) est définie
- [ ] Les goulots d'étranglement potentiels sont identifiés
- [ ] Les mécanismes de caching sont intégrés si nécessaire

### ✅ Haute disponibilité
- [ ] Un Replica Set avec au moins 3 membres est configuré
- [ ] La stratégie de failover est testée
- [ ] Les Read/Write Concerns sont adaptés aux besoins
- [ ] Un plan de disaster recovery existe

### ✅ Sécurité
- [ ] L'authentification est activée et robuste
- [ ] Les rôles et permissions sont définis selon le principe du moindre privilège
- [ ] Le chiffrement en transit (TLS/SSL) est configuré
- [ ] Le chiffrement au repos est évalué selon les exigences
- [ ] L'audit est activé si nécessaire

### ✅ Monitoring et observabilité
- [ ] Les métriques clés sont suivies (ops/s, latence, utilisation disque)
- [ ] Des alertes sont configurées pour les seuils critiques
- [ ] Les logs sont centralisés et analysables
- [ ] Un système de profiling des requêtes lentes est en place

### ✅ Opérations et maintenance
- [ ] Une stratégie de backup automatique est mise en place
- [ ] Les procédures de restauration sont testées
- [ ] Un plan de maintenance (upgrades, patches) existe
- [ ] La documentation opérationnelle est à jour

## Anti-patterns architecturaux à éviter

### ❌ Anti-pattern 1 : "Penser SQL avec MongoDB"
**Erreur courante :** Normaliser excessivement et utiliser des références partout

**Conséquence :**
- Multiples requêtes pour obtenir des données liées
- Performance dégradée
- Complexité applicative inutile

**Solution :** Embrasser la dénormalisation et l'embedding quand approprié

### ❌ Anti-pattern 2 : "Index everything"
**Erreur courante :** Créer des index "au cas où" sans analyse des requêtes

**Conséquence :**
- Ralentissement des écritures
- Surcharge mémoire
- Maintenance complexe

**Solution :** Index guidés par l'analyse avec `explain()` et le profiler

### ❌ Anti-pattern 3 : "Sharding prématuré"
**Erreur courante :** Déployer un cluster shardé pour un petit volume de données

**Conséquence :**
- Complexité opérationnelle disproportionnée
- Coûts infrastructure élevés
- Overhead réseau inutile

**Solution :** Commencer par Replica Set, sharder uniquement quand nécessaire

### ❌ Anti-pattern 4 : "One size fits all"
**Erreur courante :** Utiliser la même architecture pour tous les cas d'usage

**Conséquence :**
- Sur-engineering ou sous-engineering selon les cas
- Coûts non optimisés
- Maintenance hétérogène

**Solution :** Adapter l'architecture aux contraintes spécifiques de chaque cas

### ❌ Anti-pattern 5 : "Ignorer la croissance"
**Erreur courante :** Concevoir uniquement pour le besoin immédiat

**Conséquence :**
- Migrations complexes et risquées en production
- Interruptions de service
- Refonte architecturale coûteuse

**Solution :** Planifier pour 2-3x la charge actuelle, mais sans over-engineering

## Méthodologie de migration progressive

Lorsqu'on adopte MongoDB ou qu'on fait évoluer une architecture existante, une approche progressive est recommandée :

### Phase 1 : Preuve de concept (PoC)
**Objectif :** Valider la faisabilité technique

- Environnement isolé (dev/test)
- Sous-ensemble représentatif des données
- Tests de performance sur cas d'usage critiques
- Validation de l'intégration avec l'existant

**Critères de succès :**
- Les exigences fonctionnelles sont satisfaites
- Les performances sont acceptables
- L'équipe maîtrise les concepts de base

### Phase 2 : Déploiement pilote
**Objectif :** Valider en conditions réelles contrôlées

- Replica Set en production pour un service non-critique
- Monitoring intensif
- Collecte de métriques réelles
- Ajustements de la configuration

**Critères de succès :**
- Stabilité sur plusieurs semaines
- Métriques dans les SLA
- Équipe opérationnelle formée

### Phase 3 : Déploiement progressif
**Objectif :** Généralisation à l'ensemble des services

- Migration service par service
- Coexistence temporaire avec systèmes legacy
- Synchronisation bidirectionnelle si nécessaire
- Documentation des patterns éprouvés

**Critères de succès :**
- Migration sans interruption de service
- Tous les services atteignent leurs SLA
- Réduction des coûts ou amélioration des performances

### Phase 4 : Optimisation continue
**Objectif :** Améliorer et faire évoluer

- Analyse régulière des performances
- Ajustement de la modélisation si nécessaire
- Scaling selon la croissance réelle
- Veille technologique sur les nouvelles fonctionnalités

## Considérations multi-environnements

Une architecture de production nécessite généralement plusieurs environnements :

### Environnement de développement
```yaml
Type: Standalone ou Replica Set minimal (1 nœud)
Données: Dataset réduit ou anonymisé
Configuration: Détendue (pas de sécurité stricte)
Coût: Minimal
```

### Environnement de test/staging
```yaml
Type: Replica Set (3 nœuds)
Données: Copie récente de production (anonymisée)
Configuration: Identique à production
Coût: Moyen (20-30% de production)
```

### Environnement de production
```yaml
Type: Replica Set ou Sharded Cluster
Données: Données réelles
Configuration: Optimisée et sécurisée
Coût: Selon les besoins réels
```

### Environnement de disaster recovery
```yaml
Type: Réplica ou snapshot dans région secondaire
Données: Synchronisation continue ou périodique
Configuration: Standby ou active-passive
Coût: Selon RPO/RTO requis
```

## Outils d'aide à la décision

### 1. Calculateurs de dimensionnement
- **MongoDB Atlas Sizing Calculator** : Estimation des ressources
- **Formules empiriques** :
  - RAM = (Working Set × 1.2) + Index Size
  - IOPS = (Ops/s × 1.5) pour des accès aléatoires
  - Throughput réseau = Data Size × Replication Factor / Time

### 2. Outils de validation
- **MongoDB Compass** : Exploration et validation du schéma
- **explain() et profiler** : Validation des performances
- **mongosh avec scripts** : Tests d'intégration
- **Load testing tools** : YCSB, Apache JMeter, artillery.io

### 3. Frameworks de migration
- **Relational Migrator** : Migration depuis SQL
- **mongomirror** : Synchronisation entre clusters
- **Change Streams** : Synchronisation en temps réel

## Synthèse : Arbre de décision

```
Quel est votre cas d'usage principal ?
│
├─ OLTP transactionnel
│  ├─ Volume < 1TB → Replica Set 3 nœuds
│  ├─ Volume > 1TB → Évaluer sharding
│  └─ Forte cohérence → Transactions + Write Concern majority
│
├─ Analyse / Reporting
│  ├─ Temps réel → Secondary avec read preference
│  ├─ Batch → Delayed secondary ou Atlas Data Lake
│  └─ BI Tools → MongoDB Connector for BI
│
├─ Catalogue / CMS
│  ├─ Recherche full-text → Index texte ou Atlas Search
│  ├─ Hiérarchies → Pattern Embedded ou Materialized Path
│  └─ Multi-langue → Champs localisés ou documents séparés
│
├─ IoT / Time Series
│  ├─ Volume élevé → Time Series Collections
│  ├─ Retention → TTL Index
│  └─ Agrégations → Pipeline avec $bucket
│
├─ Géolocalisation
│  ├─ Requêtes spatiales → Index 2dsphere
│  ├─ Proximité → $geoNear
│  └─ Zones → Zone Sharding
│
└─ Logs / Events
   ├─ Écriture intensive → Minimal indexing + Sharding
   ├─ Rotation → Capped Collections ou TTL
   └─ Agrégations complexes → Pipeline + $merge
```

## Prochaines sections

Les sections suivantes de ce chapitre détaillent les architectures spécifiques pour chaque domaine d'application :

- **20.1** : Applications web et mobiles (CRUD, sessions, caching)
- **20.2** : Gestion de contenu (CMS, hiérarchies, versioning)
- **20.3** : Catalogue produits (e-commerce, recherche, personnalisation)
- **20.4** : Internet des objets (IoT, séries temporelles, high throughput)
- **20.5** : Gaming et leaderboards (classements, scores, matchmaking)
- **20.6** : Analyse en temps réel (streaming, agrégations, dashboards)
- **20.7** : Gestion des logs (ingestion massive, retention, recherche)
- **20.8** : Personnalisation et recommandations (profils, ML, real-time)
- **20.9** : Applications financières (transactions, audit, compliance)
- **20.10** : Architecture microservices (découplage, événements, CQRS)
- **20.11** : Event Sourcing avec MongoDB (immutabilité, replay, audit)
- **20.12** : CQRS et MongoDB (séparation lecture/écriture, projections)

Chaque section présentera des architectures de référence, des décisions de conception justifiées, et des exemples concrets issus de l'industrie.

---

## Références et ressources complémentaires

### Documentation officielle
- [MongoDB Architecture Guide](https://www.mongodb.com/docs/manual/core/architecture/)
- [MongoDB Best Practices](https://www.mongodb.com/docs/manual/administration/production-notes/)
- [MongoDB University - M320: Data Modeling](https://university.mongodb.com/)

### Articles et études de cas
- MongoDB Blog: Customer case studies
- InfoQ: Articles sur les architectures à grande échelle
- High Scalability: Architectures de systèmes distribués

### Livres recommandés
- "MongoDB: The Definitive Guide" - Shannon Bradshaw et al.
- "Designing Data-Intensive Applications" - Martin Kleppmann
- "Building Microservices" - Sam Newman

### Communauté
- MongoDB Community Forums
- Stack Overflow (tag: mongodb)
- MongoDB User Groups locaux

---

**Note importante :** Les architectures présentées dans ce chapitre sont des modèles de référence. Chaque système de production doit être adapté à son contexte spécifique, ses contraintes, et ses objectifs métier. L'analyse, le test et le monitoring sont essentiels pour valider les choix architecturaux.

⏭️ [Applications web et mobiles](/20-cas-usage-architectures/01-applications-web-mobiles.md)
