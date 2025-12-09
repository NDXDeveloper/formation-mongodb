🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 19 : Migration et Intégration

## Vue d'ensemble du chapitre

La migration vers MongoDB et son intégration dans des écosystèmes de données existants représentent des défis majeurs pour les architectes de données. Ce chapitre explore les stratégies, outils et patterns éprouvés pour réussir des migrations complexes et établir des architectures hybrides performantes.

---

## 🎯 Objectifs du chapitre

À l'issue de ce chapitre, vous maîtriserez :

- **Stratégies de migration** : Approches big bang, incrémentales, parallèles et hybrides
- **Outils de migration** : MongoDB Relational Migrator, Kafka Connect, solutions tierces
- **Patterns d'intégration** : ETL/ELT, CDC, synchronisation bidirectionnelle, event-driven
- **Architecture polyglotte** : Coexistence harmonieuse entre bases relationnelles et NoSQL
- **Gouvernance des données** : Traçabilité, cohérence et qualité lors des migrations

---

## 📊 Contexte et enjeux

### Pourquoi migrer vers MongoDB ?

Les organisations migrent vers MongoDB pour diverses raisons stratégiques :

**Raisons techniques**
- **Scalabilité horizontale** : Gestion de volumes massifs (plusieurs To ou Po)
- **Flexibilité du schéma** : Adaptation rapide aux évolutions métier
- **Performance** : Latence réduite pour workloads documentaires et analytics
- **Haute disponibilité native** : Replica sets et géo-distribution intégrées

**Raisons business**
- **Time-to-market** : Développement agile avec modèles flexibles
- **Réduction des coûts** : TCO optimisé vs solutions relationnelles propriétaires
- **Innovation** : Exploitation de données non structurées, IA/ML, IoT
- **Cloud-native** : Migration vers architectures modernes (microservices, serverless)

### Défis typiques

**Défis techniques**
- Transformation de schémas normalisés en modèles dénormalisés
- Préservation de l'intégrité référentielle sans contraintes FK
- Migration de transactions complexes vers modèles orientés documents
- Conversion de procédures stockées en logique applicative

**Défis organisationnels**
- Formation des équipes aux paradigmes NoSQL
- Adaptation des processus DevOps et CI/CD
- Gestion du changement culturel (mindset relationnel → document)
- Gouvernance dans environnements polyglotte

**Défis opérationnels**
- Maintien de la disponibilité pendant la migration
- Garantie de cohérence des données migrant
- Validation exhaustive des données transformées
- Rollback et stratégies de repli

---

## 🏗️ Architecture de migration : Panorama

### 1. Migration Big Bang

**Principe** : Basculement complet en une seule opération (souvent un week-end)

**Scénario typique**
```
[Système SQL source] → [Arrêt] → [Migration complète] → [Démarrage MongoDB] → [Production]
                           ↓
                    [Freeze des écritures]
```

**Cas d'usage**
- Systèmes de taille modérée (<1 To)
- Applications pouvant tolérer une fenêtre de maintenance
- Budget limité pour infrastructure parallèle
- Besoin de simplification opérationnelle

**Avantages**
- ✅ Simplicité conceptuelle
- ✅ Coût d'infrastructure minimal
- ✅ Pas de synchronisation complexe
- ✅ Transition nette

**Inconvénients**
- ❌ Downtime important (heures à jours)
- ❌ Risque élevé (rollback difficile)
- ❌ Pression intense sur l'équipe
- ❌ Validation limitée en conditions réelles

**Exemple concret : Migration e-commerce**
Une PME avec 500 Go de données produit/commandes migre pendant un week-end de basse activité. Arrêt du site vendredi 22h, migration + validation, réouverture lundi 6h.

---

### 2. Migration incrémentale (Strangler Pattern)

**Principe** : Migration progressive par domaines métier ou modules

**Scénario typique**
```
Phase 1: [SQL] ← → [MongoDB] (Catalogue produits uniquement)
Phase 2: [SQL] ← → [MongoDB] (+ Gestion stock)
Phase 3: [SQL] ← → [MongoDB] (+ Commandes)
Phase N: [MongoDB seul] (Migration complète)
```

**Cas d'usage**
- Systèmes monolithiques complexes
- Architectures microservices en transition
- Impossibilité de downtime prolongé
- Besoin de validation progressive

**Avantages**
- ✅ Zero downtime
- ✅ Réduction des risques (rollback par module)
- ✅ Apprentissage progressif des équipes
- ✅ ROI rapide sur modules critiques

**Inconvénients**
- ❌ Complexité architecturale (dual-writes, sync)
- ❌ Durée totale longue (mois à années)
- ❌ Coûts d'infrastructure doublés temporairement
- ❌ Maintenance de deux systèmes en parallèle

**Exemple concret : Migration bancaire**
Une banque migre son système CRM en 18 mois :
- **Mois 1-3** : Données clients (lecture seule)
- **Mois 4-8** : Historique transactions (analytics)
- **Mois 9-14** : Produits financiers et offres
- **Mois 15-18** : Décommissionnement progressif SQL

---

### 3. Migration avec période de synchronisation (Parallel Run)

**Principe** : Exécution parallèle avec synchronisation bidirectionnelle

**Scénario typique**
```
[Application]
     ↓
[Dual-Write Layer]
   ↙        ↘
[SQL]  ⟷  [MongoDB]
          (sync CDC)
```

**Cas d'usage**
- Applications critiques (finance, santé)
- Besoin de validation exhaustive
- Conformité réglementaire stricte
- Rollback garanti sans perte

**Avantages**
- ✅ Sécurité maximale (2 sources de vérité)
- ✅ Validation en conditions réelles
- ✅ Rollback instantané
- ✅ Comparaison exhaustive des résultats

**Inconvénients**
- ❌ Coût infrastructure maximal
- ❌ Complexité synchronisation bidirectionnelle
- ❌ Conflits de cohérence à gérer
- ❌ Performance dégradée (double écriture)

**Exemple concret : Système hospitalier**
Migration d'un dossier patient électronique avec période de 6 mois en parallèle :
- Tous les writes vont vers SQL ET MongoDB
- Reconciliation quotidienne automatisée
- Validation par équipes médicales sur MongoDB
- Bascule définitive après certification

---

## 🔄 Stratégies de synchronisation des données

### Change Data Capture (CDC)

**Principe** : Capture des modifications en temps réel depuis la base source

**Technologies courantes**
- **Debezium** : CDC open-source (MySQL, PostgreSQL, SQL Server, Oracle)
- **AWS DMS** : Service managé AWS
- **Oracle GoldenGate** : Solution enterprise
- **MongoDB Kafka Connector** : Intégration native Kafka

**Architecture CDC typique**
```
[Base SQL] → [Binlog/WAL] → [CDC Engine] → [Kafka/Queue] → [Consumer] → [MongoDB]
```

**Avantages CDC**
- Latence faible (quasi temps-réel)
- Impact minimal sur source
- Garantie de complétude (toutes les modifications)
- Scalabilité élevée

**Limitations**
- Configuration complexe (dépend du SGBD source)
- Gestion des DDL changes
- Résolution de conflits en cas de dual-write

---

### ETL Batch vs Streaming

**ETL Batch classique**
```
Extract (SQL) → Transform (Spark/Python) → Load (MongoDB)
Fréquence : Quotidienne, horaire
```

**Cas d'usage**
- Migration historique volumineuse
- Données analytics (non temps-réel)
- Transformations complexes (aggregations, enrichissement)

**ETL Streaming**
```
Stream (Kafka/Kinesis) → Transform (Spark Streaming/Flink) → MongoDB
Latence : Secondes
```

**Cas d'usage**
- Données IoT
- Événements utilisateurs (clickstream)
- Synchronisation quasi temps-réel

---

## 🛠️ Écosystème d'outils

### Outils MongoDB officiels

| Outil | Usage | Points forts |
|-------|-------|--------------|
| **MongoDB Relational Migrator** | Migration schema + data depuis SQL | Interface graphique, génération code |
| **mongodump/mongorestore** | Export/import MongoDB | Natif, rapide, format BSON |
| **mongoexport/mongoimport** | Export/import JSON/CSV | Lisibilité, interopérabilité |
| **MongoDB Kafka Connector** | Intégration Kafka | CDC, event-driven, scalable |
| **MongoDB Spark Connector** | ETL avec Spark | Big data, transformations complexes |
| **MongoDB Atlas Data Federation** | Fédération multi-sources | Queries cross-database sans migration physique |

### Solutions tierces

**Talend Open Studio / Talend Data Fabric**
- Interface drag-and-drop
- Connecteurs pré-construits (SQL → MongoDB)
- Transformations complexes
- Orchestration de pipelines

**Apache NiFi**
- Dataflow visuel
- Processeurs MongoDB natifs
- Gestion backpressure et retry
- Monitoring temps-réel

**Airbyte / Fivetran**
- ELT moderne cloud-native
- Catalogue de connecteurs (200+)
- Change data capture intégré
- Déploiement SaaS ou self-hosted

**Pentaho / Informatica**
- Solutions enterprise
- Gouvernance avancée
- Qualité des données
- Support commercial

---

## 📐 Patterns de transformation de schéma

### 1. One-to-One : Table → Collection

**SQL**
```sql
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);
```

**MongoDB (direct mapping)**
```javascript
{
    _id: ObjectId(),  // ou conservation de l'ID SQL
    id: 12345,
    name: "Alice Durand",
    email: "alice@example.com"
}
```

**Décision d'architecture**
- Conserver `id` SQL comme champ secondaire ?
- Utiliser `_id` natif MongoDB ?
- Index sur ancien `id` pour jointures transitoires ?

---

### 2. One-to-Many : FK → Embedded Documents

**SQL**
```sql
-- Table parent
CREATE TABLE orders (id INT, customer_id INT, date DATE);

-- Table enfant
CREATE TABLE order_items (
    id INT,
    order_id INT,  -- FK
    product_id INT,
    quantity INT
);
```

**MongoDB (embedding)**
```javascript
{
    _id: ObjectId("..."),
    customer_id: 5678,
    date: ISODate("2024-01-15"),
    items: [  // Embedded array
        { product_id: 101, quantity: 2 },
        { product_id: 203, quantity: 1 }
    ]
}
```

**Critères de décision**
- **Embedding si** : Relation forte, accès conjoint, taille limitée (<100 items typiquement)
- **Referencing si** : Items volumineux, accès indépendant, croissance illimitée

---

### 3. Many-to-Many : Tables de jonction → References

**SQL**
```sql
CREATE TABLE students (id INT, name VARCHAR);
CREATE TABLE courses (id INT, title VARCHAR);
CREATE TABLE enrollments (student_id INT, course_id INT);  -- Table de jonction
```

**MongoDB (approche 1 : array de références)**
```javascript
// Collection students
{
    _id: ObjectId("student1"),
    name: "Bob Martin",
    course_ids: [ObjectId("course1"), ObjectId("course2")]
}

// Collection courses
{
    _id: ObjectId("course1"),
    title: "MongoDB Avancé"
}
```

**MongoDB (approche 2 : dénormalisation partielle)**
```javascript
{
    _id: ObjectId("student1"),
    name: "Bob Martin",
    enrollments: [
        { course_id: ObjectId("course1"), title: "MongoDB Avancé", enrolled_date: ISODate() },
        { course_id: ObjectId("course2"), title: "Kubernetes", enrolled_date: ISODate() }
    ]
}
```

**Trade-offs**
- Approche 1 : Consistance forte, mais nécessite $lookup
- Approche 2 : Performance lecture, mais duplication (gestion des updates)

---

### 4. Héritage et polymorphisme

**SQL (Single Table Inheritance)**
```sql
CREATE TABLE vehicles (
    id INT,
    type VARCHAR(20),  -- 'car', 'truck', 'motorcycle'
    brand VARCHAR(50),
    num_doors INT,      -- Pour cars uniquement
    payload_capacity INT  -- Pour trucks uniquement
);
```

**MongoDB (Pattern Polymorphic)**
```javascript
// Tous dans une collection vehicles
{ _id: 1, type: "car", brand: "Toyota", num_doors: 4 }
{ _id: 2, type: "truck", brand: "Volvo", payload_capacity: 15000 }
{ _id: 3, type: "motorcycle", brand: "Harley", engine_cc: 1200 }
```

**Avantages MongoDB**
- Schéma flexible natif
- Pas de colonnes NULL inutiles
- Validation conditionnelle par type
- Queries hétérogènes efficaces

---

## 🎯 Scénarios de migration réels

### Scénario 1 : E-commerce monolithique → Microservices

**Contexte**
- Système PHP/MySQL monolithique (8 ans)
- 2 To de données (produits, commandes, clients)
- 500 req/s en peak
- Objectif : Architecture microservices + scalabilité

**Stratégie choisie : Strangler Pattern incrémental**

**Phase 1 (Mois 1-2) : Catalogue produits**
- Migration read-only du catalogue vers MongoDB
- API Gateway route les lectures vers MongoDB
- Écritures restent sur MySQL avec CDC vers MongoDB
- Validation : Comparaison automatisée des résultats

**Phase 2 (Mois 3-5) : Service de recherche**
- Atlas Search pour recherche full-text avancée
- Facettes et filtres sur MongoDB
- Amélioration performance : 200ms → 50ms (latence P95)

**Phase 3 (Mois 6-9) : Service commandes**
- Modélisation Order Aggregate avec items embedded
- CDC bidirectionnel temporaire
- Tests de charge : 1000 req/s soutenus

**Phase 4 (Mois 10-12) : Décommissionnement MySQL**
- Migration données clients restantes
- Arrêt CDC
- Archive MySQL pour conformité

**Résultats**
- ✅ Réduction coûts infrastructure : -40%
- ✅ Performance améliorée : +4x
- ✅ Time-to-market : -60% (nouvelles features)
- ✅ Zero incident majeur

---

### Scénario 2 : IoT industriel temps-réel

**Contexte**
- Usine avec 10 000 capteurs
- Base Oracle (séries temporelles)
- 50 000 mesures/seconde
- Analyse en temps-réel requise

**Stratégie choisie : Architecture Lambda (batch + streaming)**

**Architecture cible**
```
[Capteurs] → [Kafka] → [Flink] → [MongoDB Time Series]
                ↓
            [S3/Parquet] (archive long-terme)
```

**Implémentation**
- **Couche streaming** : Flink pour aggregations temps-réel (moyennes 1min, alertes)
- **MongoDB Time Series Collections** : Optimisées pour données temporelles
- **Retention policy** : 30 jours données brutes, agrégats conservés indéfiniment
- **Alerting** : Change Streams pour détection anomalies

**Migration des historiques**
- Spark batch job : Oracle → Parquet → MongoDB
- 5 ans d'historique migrés en 48h
- Validation statistique (agrégats, min/max, distributions)

**Résultats**
- ✅ Latence requêtes analytiques : 5s → 200ms
- ✅ Coût stockage : -70% (compression native BSON)
- ✅ Scalabilité : Sharding automatique sur timestamp

---

### Scénario 3 : SaaS multi-tenant B2B

**Contexte**
- Application SaaS sur PostgreSQL
- 500 clients (tenants)
- Croissance : +50 nouveaux clients/mois
- Problème : Isolation tenants, schema rigide

**Stratégie choisie : Migration progressive avec architecture polyglotte**

**Architecture hybride**
```
[PostgreSQL]              [MongoDB]
- Authentification        - Données métier par tenant
- Facturation            - Configurations custom
- Audit logs             - Analytics temps-réel
```

**Pattern multi-tenancy MongoDB**
```javascript
// Option 1 : Database par tenant (choix retenu)
tenant_A / collections { orders, invoices, products }
tenant_B / collections { orders, invoices, products }

// Option 2 (non retenue) : Collection partagée avec discriminateur
{ _tenant_id: "A", order_id: 123, ... }
```

**Raison du choix "database par tenant"**
- Isolation forte (sécurité, performance)
- Backup/restore granulaire
- Customisation schéma par client
- Conformité RGPD (suppression tenant = drop database)

**Migration**
- Script automatisé : Création DB MongoDB par tenant PostgreSQL
- CDC via Debezium pour synchronisation continue
- Période de parallel run : 3 mois
- Validation par tenant (tests fonctionnels clients pilotes)

**Résultats**
- ✅ Onboarding nouveau client : 2 jours → 2 heures
- ✅ Personnalisation schema : 0 friction
- ✅ Conformité RGPD : Simplified

---

## 🔐 Gouvernance et qualité des données

### Validation de la migration

**Stratégies de validation**

1. **Row count validation**
```javascript
// Script de vérification basique
const sqlCount = await sqlClient.query("SELECT COUNT(*) FROM users");
const mongoCount = await mongoDb.collection("users").countDocuments();
assert(sqlCount === mongoCount);
```

2. **Checksum validation**
- Calcul de hash sur colonnes critiques (SQL)
- Calcul de hash équivalent sur champs MongoDB
- Comparaison des agrégats

3. **Sampling validation**
- Sélection aléatoire de N records
- Comparaison champ par champ
- Alertes sur divergences

4. **Business rules validation**
- Tests fonctionnels critiques
- Validation contraintes métier
- Cohérence référentielle

### Traçabilité

**Métadonnées de migration**
```javascript
{
    _id: ObjectId(),
    // ... données métier ...
    _migration_metadata: {
        source_system: "postgresql",
        source_table: "orders",
        source_id: 12345,
        migrated_at: ISODate("2024-01-15T10:30:00Z"),
        migration_batch_id: "batch_2024_01_15_001",
        validation_status: "validated"
    }
}
```

**Avantages**
- Audit trail complet
- Rollback granulaire possible
- Debugging facilité
- Conformité réglementaire

---

## 📊 Monitoring et observabilité

### Métriques clés pendant migration

**Performance**
- Throughput migration (docs/seconde)
- Latence writes (SQL et MongoDB)
- Backlog CDC (lag)
- Taille queue messages (Kafka)

**Qualité**
- Taux d'erreur transformation
- Divergences SQL vs MongoDB
- Rejets validation schema
- Conflits synchronisation

**Infrastructure**
- CPU/RAM MongoDB
- IOPS disque
- Network bandwidth
- Oplog size (Replica Sets)

### Outils de monitoring

**Stack recommandée**
```
Prometheus + Grafana
    ↓
MongoDB Exporter (metrics)
    ↓
Alertmanager (notifications)
```

**Dashboards critiques**
- Migration progress (% completion)
- Data quality KPIs
- Performance comparison (SQL vs MongoDB)
- Error tracking

---

## 🚨 Gestion des risques

### Risques techniques majeurs

| Risque | Impact | Probabilité | Mitigation |
|--------|--------|-------------|------------|
| **Perte de données** | Critique | Faible | Backup avant migration, validation exhaustive, CDC reliable |
| **Downtime prolongé** | Élevé | Moyenne | Migration incrémentale, rollback plan, dry-runs |
| **Divergence données** | Élevé | Moyenne | Synchronisation bidirectionnelle, reconciliation automatisée |
| **Performance dégradée** | Moyen | Élevée | Benchmark pré-migration, indexation optimisée, testing charge |
| **Corruption transformation** | Critique | Faible | Pipeline testing, validation business rules |

### Plan de rollback

**Conditions de déclenchement**
- Erreurs validation > seuil acceptable (ex: 0.1%)
- Performance < baseline SQL (régression)
- Bugs critiques découverts en production
- Incident infrastructure MongoDB

**Procédure rollback type**
```
1. Freeze writes MongoDB
2. Re-route trafic vers SQL
3. Reverse CDC : MongoDB → SQL (récupération delta)
4. Validation cohérence SQL
5. Restore normal operations
6. Post-mortem et correctifs
```

**Critère de no-return point**
- Après décommissionnement SQL (archive uniquement)
- Généralement 3-6 mois post-bascule définitive

---

## 🎓 Compétences requises pour l'équipe

### Rôles clés

**Architecte de données**
- Design modèles MongoDB optimisés
- Stratégie migration globale
- Arbitrages techniques (embedding vs referencing)
- Validation architecture cible

**Ingénieur Data**
- Implémentation pipelines ETL/CDC
- Scripting transformation (Python, Spark)
- Monitoring qualité données
- Optimisation performance

**DBA MongoDB**
- Configuration Replica Sets / Sharding
- Tuning performance (indexes, query optimization)
- Backup/restore
- Troubleshooting production

**DevOps / SRE**
- Automation déploiements
- Infrastructure as Code (Terraform)
- Monitoring et alerting
- Incident management

### Formation recommandée

**MongoDB University (gratuit)**
- M121: MongoDB Aggregation Framework
- M201: MongoDB Performance
- M320: MongoDB Data Modeling

**Certifications**
- MongoDB Certified Developer Associate
- MongoDB Certified DBA Associate

---

## 📚 Structure du chapitre

Ce chapitre est organisé en sections détaillées couvrant chaque aspect de la migration et de l'intégration :

### Sections à venir

- **19.1** : Migration depuis SQL vers MongoDB
- **19.2** : Outils de migration (comparatif détaillé)
- **19.3** : Relational Migrator (guide complet)
- **19.4** : Stratégies de migration incrémentale
- **19.5** : Synchronisation bidirectionnelle
- **19.6** : MongoDB Connector for BI
- **19.7** : Intégration avec Apache Kafka
- **19.8** : Intégration avec Apache Spark
- **19.9** : ETL et Data Pipelines
- **19.10** : Coexistence avec des bases relationnelles

---

## 🎯 Points clés à retenir

### Principes fondamentaux

1. **Il n'existe pas de stratégie universelle** : Chaque migration est unique (contexte, contraintes, objectifs)

2. **La modélisation est critique** : 80% du succès dépend d'un modèle MongoDB adapté (ne pas faire du SQL sur MongoDB)

3. **Progressivité réduit les risques** : Privilégier approches incrémentales sauf cas simples

4. **Validation est non négociable** : Investir massivement dans tests et comparaisons

5. **Architecture polyglotte est viable** : MongoDB n'a pas à remplacer toutes les bases existantes

### Anti-patterns à éviter

- ❌ **Migration sans refonte de modèle** : Reproduire schéma SQL normalisé dans MongoDB
- ❌ **Big bang sans dry-run** : Bascule en production sans test grandeur nature
- ❌ **Sous-estimation de la complexité** : Transformation schéma + logique métier + tests
- ❌ **Négliger la formation** : Équipes non formées aux paradigmes MongoDB
- ❌ **Absence de rollback plan** : Aucune stratégie de repli en cas de problème

### Checklist de démarrage

Avant de lancer une migration, valider :

- [ ] Business case solide (ROI, bénéfices métier)
- [ ] Sponsor exécutif engagé
- [ ] Équipe dédiée avec compétences MongoDB
- [ ] Budget (infrastructure, outils, formation)
- [ ] Architecture cible documentée
- [ ] POC réalisé avec succès
- [ ] Stratégie de migration choisie et validée
- [ ] Plan de rollback défini
- [ ] Métriques de succès établies
- [ ] Gouvernance et processus définis

---

## 🔗 Ressources complémentaires

### Documentation officielle
- [MongoDB Migration Guide](https://www.mongodb.com/cloud/atlas/migrate)
- [Relational Migrator Documentation](https://www.mongodb.com/products/relational-migrator)
- [Change Streams](https://www.mongodb.com/docs/manual/changeStreams/)

### Études de cas
- MongoDB Customer Success Stories (mongodb.com/customers)
- Architecture patterns (mongodb.com/blog)

### Outils open-source
- Debezium (debezium.io)
- Apache NiFi (nifi.apache.org)
- Airbyte (airbyte.com)

---

**Dans les sections suivantes**, nous détaillerons chaque aspect de la migration avec des exemples techniques concrets, des configurations détaillées et des patterns éprouvés en production.

**Prochaine section** : 19.1 Migration depuis SQL vers MongoDB - Guide technique complet des transformations de schéma et stratégies de migration.

⏭️ [Migration depuis SQL vers MongoDB](/19-migration-integration/01-migration-sql-vers-mongodb.md)
