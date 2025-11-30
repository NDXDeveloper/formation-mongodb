# 📚 Sommaire - Formation MongoDB Complète
**Un guide progressif pour développeurs et DevOps — Du débutant à l'expert**

---

## **[Partie 1 : Introduction et Concepts Fondamentaux (Débutant)](/partie-01-introduction-concepts-fondamentaux.md)**

### 1. [Introduction à MongoDB](01-introduction-a-mongodb/README.md)
- 1.1 [Qu'est-ce que MongoDB ?](01-introduction-a-mongodb/01-quest-ce-que-mongodb.md)
- 1.2 [Historique et évolution](01-introduction-a-mongodb/02-historique-et-evolution.md)
- 1.3 [Bases de données NoSQL vs SQL : Comparaison conceptuelle](01-introduction-a-mongodb/03-nosql-vs-sql.md)
- 1.4 [Fondements théoriques](01-introduction-a-mongodb/04-fondements-theoriques.md)
    - 1.4.1 [Le théorème CAP (Consistency, Availability, Partition tolerance)](01-introduction-a-mongodb/04.1-theoreme-cap.md)
    - 1.4.2 [Positionnement de MongoDB dans le théorème CAP](01-introduction-a-mongodb/04.2-positionnement-mongodb-cap.md)
    - 1.4.3 [Eventual Consistency vs Strong Consistency](01-introduction-a-mongodb/04.3-eventual-vs-strong-consistency.md)
- 1.5 [Cas d'usage et quand choisir MongoDB](01-introduction-a-mongodb/05-cas-usage-et-choix.md)
- 1.6 [Architecture générale de MongoDB](01-introduction-a-mongodb/06-architecture-generale.md)
- 1.7 [Terminologie : Documents, Collections, Bases de données](01-introduction-a-mongodb/07-terminologie.md)
- 1.8 [Installation de MongoDB (Windows, Linux, macOS)](01-introduction-a-mongodb/08-installation-mongodb.md)
- 1.9 [Installation via Docker](01-introduction-a-mongodb/09-installation-docker.md)
- 1.10 [Présentation des outils : mongosh, MongoDB Compass, Atlas](01-introduction-a-mongodb/10-presentation-outils.md)

### 2. [Fondamentaux de MongoDB](02-fondamentaux-de-mongodb/README.md)
- 2.1 [Structure des documents BSON](02-fondamentaux-de-mongodb/01-structure-documents-bson.md)
- 2.2 [Types de données BSON](02-fondamentaux-de-mongodb/02-types-de-donnees-bson.md)
- 2.3 [Création d'une base de données](02-fondamentaux-de-mongodb/03-creation-base-de-donnees.md)
- 2.4 [Création et gestion des collections](02-fondamentaux-de-mongodb/04-creation-gestion-collections.md)
- 2.5 [Opérations CRUD de base](02-fondamentaux-de-mongodb/05-operations-crud.md)
    - 2.5.1 [insertOne() et insertMany()](02-fondamentaux-de-mongodb/05.1-insert.md)
    - 2.5.2 [find() et findOne()](02-fondamentaux-de-mongodb/05.2-find.md)
    - 2.5.3 [updateOne() et updateMany()](02-fondamentaux-de-mongodb/05.3-update.md)
    - 2.5.4 [deleteOne() et deleteMany()](02-fondamentaux-de-mongodb/05.4-delete.md)
    - 2.5.5 [replaceOne()](02-fondamentaux-de-mongodb/05.5-replace.md)
- 2.6 [Le shell MongoDB (mongosh)](02-fondamentaux-de-mongodb/06-shell-mongosh.md)
- 2.7 [Introduction à MongoDB Compass](02-fondamentaux-de-mongodb/07-introduction-compass.md)

### 3. [Requêtes et Filtres](03-requetes-et-filtres/README.md)
- 3.1 [Syntaxe des requêtes de base](03-requetes-et-filtres/01-syntaxe-requetes-base.md)
- 3.2 [Opérateurs de comparaison ($eq, $ne, $gt, $gte, $lt, $lte, $in, $nin)](03-requetes-et-filtres/02-operateurs-comparaison.md)
- 3.3 [Opérateurs logiques ($and, $or, $not, $nor)](03-requetes-et-filtres/03-operateurs-logiques.md)
- 3.4 [Opérateurs d'éléments ($exists, $type)](03-requetes-et-filtres/04-operateurs-elements.md)
- 3.5 [Opérateurs d'évaluation ($regex, $expr, $mod, $text, $where)](03-requetes-et-filtres/05-operateurs-evaluation.md)
- 3.6 [Opérateurs de tableaux ($all, $elemMatch, $size)](03-requetes-et-filtres/06-operateurs-tableaux.md)
- 3.7 [Projections : Sélection des champs](03-requetes-et-filtres/07-projections.md)
- 3.8 [Tri, limite et pagination (sort, limit, skip)](03-requetes-et-filtres/08-tri-limite-pagination.md)
- 3.9 [Comptage de documents (countDocuments, estimatedDocumentCount)](03-requetes-et-filtres/09-comptage-documents.md)
- 3.10 [Requêtes sur documents imbriqués](03-requetes-et-filtres/10-requetes-documents-imbriques.md)
- 3.11 [Requêtes sur tableaux](03-requetes-et-filtres/11-requetes-tableaux.md)

---

## **[Partie 2 : Modélisation et Conception (Intermédiaire)](/partie-02-modelisation-conception.md)**

### 4. [Modélisation des Données](04-modelisation-des-donnees/README.md)
- 4.1 [Principes de modélisation orientée document](04-modelisation-des-donnees/01-principes-modelisation.md)
- 4.2 [Documents imbriqués vs Références](04-modelisation-des-donnees/02-imbriques-vs-references.md)
- 4.3 [Relations One-to-One](04-modelisation-des-donnees/03-relations-one-to-one.md)
- 4.4 [Relations One-to-Many](04-modelisation-des-donnees/04-relations-one-to-many.md)
- 4.5 [Relations Many-to-Many](04-modelisation-des-donnees/05-relations-many-to-many.md)
- 4.6 [Patterns de modélisation](04-modelisation-des-donnees/06-patterns-modelisation.md)
    - 4.6.1 [Pattern Embedded](04-modelisation-des-donnees/06.1-pattern-embedded.md)
    - 4.6.2 [Pattern Subset](04-modelisation-des-donnees/06.2-pattern-subset.md)
    - 4.6.3 [Pattern Extended Reference](04-modelisation-des-donnees/06.3-pattern-extended-reference.md)
    - 4.6.4 [Pattern Outlier](04-modelisation-des-donnees/06.4-pattern-outlier.md)
    - 4.6.5 [Pattern Computed](04-modelisation-des-donnees/06.5-pattern-computed.md)
    - 4.6.6 [Pattern Bucket](04-modelisation-des-donnees/06.6-pattern-bucket.md)
    - 4.6.7 [Pattern Schema Versioning](04-modelisation-des-donnees/06.7-pattern-schema-versioning.md)
    - 4.6.8 [Pattern Attribute](04-modelisation-des-donnees/06.8-pattern-attribute.md)
    - 4.6.9 [Pattern Polymorphic](04-modelisation-des-donnees/06.9-pattern-polymorphic.md)
- 4.7 [Anti-patterns à éviter](04-modelisation-des-donnees/07-anti-patterns.md)
- 4.8 [Limite de taille des documents (16 Mo)](04-modelisation-des-donnees/08-limite-taille-documents.md)
- 4.9 [Conception pour la performance](04-modelisation-des-donnees/09-conception-performance.md)

### 5. [Index et Optimisation](05-index-et-optimisation/README.md)
- 5.1 [Comprendre l'importance des index](05-index-et-optimisation/01-importance-des-index.md)
- 5.2 [Types d'index fondamentaux](05-index-et-optimisation/02-types-index-fondamentaux.md)
    - 5.2.1 [Index simple (Single Field)](05-index-et-optimisation/02.1-index-simple.md)
    - 5.2.2 [Index composé (Compound)](05-index-et-optimisation/02.2-index-compose.md)
    - 5.2.3 [Index multiclé (Multikey)](05-index-et-optimisation/02.3-index-multicle.md)
- 5.3 [Index spécialisés](05-index-et-optimisation/03-index-specialises.md)
    - 5.3.1 [Index texte (Text)](05-index-et-optimisation/03.1-index-texte.md)
    - 5.3.2 [Index géospatial (2d, 2dsphere)](05-index-et-optimisation/03.2-index-geospatial.md)
    - 5.3.3 [Index haché (Hashed)](05-index-et-optimisation/03.3-index-hache.md)
    - 5.3.4 [Index Wildcard](05-index-et-optimisation/03.4-index-wildcard.md)
    - 5.3.5 [Index TTL (Time-To-Live)](05-index-et-optimisation/03.5-index-ttl.md)
- 5.4 [Options et modificateurs d'index](05-index-et-optimisation/04-options-modificateurs-index.md)
    - 5.4.1 [Index unique](05-index-et-optimisation/04.1-index-unique.md)
    - 5.4.2 [Index partiel (Partial)](05-index-et-optimisation/04.2-index-partiel.md)
    - 5.4.3 [Index sparse](05-index-et-optimisation/04.3-index-sparse.md)
    - 5.4.4 [Index caché (Hidden)](05-index-et-optimisation/04.4-index-cache.md)
    - 5.4.5 [Combinaison d'options](05-index-et-optimisation/04.5-combinaison-options.md)
- 5.5 [Création et suppression d'index](05-index-et-optimisation/05-creation-suppression-index.md)
- 5.6 [Analyse des requêtes avec explain()](05-index-et-optimisation/06-analyse-explain.md)
- 5.7 [Le Query Planner](05-index-et-optimisation/07-query-planner.md)
- 5.8 [Stratégies d'optimisation des requêtes](05-index-et-optimisation/08-strategies-optimisation.md)
- 5.9 [Index couvrants (Covered Queries)](05-index-et-optimisation/09-index-couvrants.md)
- 5.10 [Gestion des index en production](05-index-et-optimisation/10-gestion-index-production.md)
- 5.11 [Outils de monitoring des performances](05-index-et-optimisation/11-outils-monitoring-performances.md)

### 6. [Framework d'Agrégation](06-framework-agregation/README.md)
- 6.1 [Introduction au framework d'agrégation](06-framework-agregation/01-introduction-agregation.md)
- 6.2 [Concept de pipeline](06-framework-agregation/02-concept-pipeline.md)
- 6.3 [Étapes de base](06-framework-agregation/03-etapes-de-base.md)
    - 6.3.1 [$match](06-framework-agregation/03.1-match.md)
    - 6.3.2 [$project](06-framework-agregation/03.2-project.md)
    - 6.3.3 [$group](06-framework-agregation/03.3-group.md)
    - 6.3.4 [$sort](06-framework-agregation/03.4-sort.md)
    - 6.3.5 [$limit et $skip](06-framework-agregation/03.5-limit-skip.md)
    - 6.3.6 [$count](06-framework-agregation/03.6-count.md)
- 6.4 [Étapes avancées](06-framework-agregation/04-etapes-avancees.md)
    - 6.4.1 [$lookup (jointures)](06-framework-agregation/04.1-lookup.md)
    - 6.4.2 [$unwind](06-framework-agregation/04.2-unwind.md)
    - 6.4.3 [$addFields et $set](06-framework-agregation/04.3-addfields-set.md)
    - 6.4.4 [$replaceRoot et $replaceWith](06-framework-agregation/04.4-replaceroot-replacewith.md)
    - 6.4.5 [$facet](06-framework-agregation/04.5-facet.md)
    - 6.4.6 [$bucket et $bucketAuto](06-framework-agregation/04.6-bucket-bucketauto.md)
    - 6.4.7 [$graphLookup](06-framework-agregation/04.7-graphlookup.md)
    - 6.4.8 [$merge et $out](06-framework-agregation/04.8-merge-out.md)
    - 6.4.9 [$redact](06-framework-agregation/04.9-redact.md)
    - 6.4.10 [$sample](06-framework-agregation/04.10-sample.md)
    - 6.4.11 [$unionWith](06-framework-agregation/04.11-unionwith.md)
- 6.5 [Opérateurs d'agrégation](06-framework-agregation/05-operateurs-agregation.md)
    - 6.5.1 [Opérateurs arithmétiques](06-framework-agregation/05.1-operateurs-arithmetiques.md)
    - 6.5.2 [Opérateurs de chaînes](06-framework-agregation/05.2-operateurs-chaines.md)
    - 6.5.3 [Opérateurs de dates](06-framework-agregation/05.3-operateurs-dates.md)
    - 6.5.4 [Opérateurs de tableaux](06-framework-agregation/05.4-operateurs-tableaux.md)
    - 6.5.5 [Opérateurs conditionnels](06-framework-agregation/05.5-operateurs-conditionnels.md)
    - 6.5.6 [Accumulateurs](06-framework-agregation/05.6-accumulateurs.md)
- 6.6 [Optimisation des pipelines d'agrégation](06-framework-agregation/06-optimisation-pipelines.md)
- 6.7 [Vues (Views) et vues matérialisées](06-framework-agregation/07-vues-materialisees.md)

### 7. [Validation des Schémas](07-validation-des-schemas/README.md)
- 7.1 [Introduction à la validation de schéma](07-validation-des-schemas/01-introduction-validation.md)
- 7.2 [JSON Schema dans MongoDB](07-validation-des-schemas/02-json-schema-mongodb.md)
- 7.3 [Règles de validation ($jsonSchema)](07-validation-des-schemas/03-regles-validation-jsonschema.md)
- 7.4 [Niveaux de validation (strict, moderate)](07-validation-des-schemas/04-niveaux-validation.md)
- 7.5 [Actions de validation (error, warn)](07-validation-des-schemas/05-actions-validation.md)
- 7.6 [Modification des règles de validation](07-validation-des-schemas/06-modification-regles.md)
- 7.7 [Validation des types de données](07-validation-des-schemas/07-validation-types-donnees.md)
- 7.8 [Validation des champs obligatoires](07-validation-des-schemas/08-validation-champs-obligatoires.md)
- 7.9 [Validation personnalisée avec $expr](07-validation-des-schemas/09-validation-personnalisee-expr.md)
- 7.10 [Bonnes pratiques de validation](07-validation-des-schemas/10-bonnes-pratiques-validation.md)

---

## **[Partie 3 : Transactions et Concurrence (Avancé)](/partie-03-transactions-concurrence.md)**

### 8. [Transactions](08-transactions/README.md)
- 8.1 [Rappel : ACID (Atomicité, Cohérence, Isolation, Durabilité)](08-transactions/01-rappel-acid.md)
    - 8.1.1 [Définition des propriétés ACID](08-transactions/01.1-definition-proprietes-acid.md)
    - 8.1.2 [ACID dans le contexte NoSQL](08-transactions/01.2-acid-contexte-nosql.md)
    - 8.1.3 [Comparaison avec les bases relationnelles](08-transactions/01.3-comparaison-bases-relationnelles.md)
- 8.2 [Atomicité native : Transactions mono-document](08-transactions/02-transactions-mono-document.md)
- 8.3 [Transactions multi-documents](08-transactions/03-transactions-multi-documents.md)
    - 8.3.1 [Cas d'usage et nécessité](08-transactions/03.1-cas-usage-necessite.md)
    - 8.3.2 [Sessions et transactions](08-transactions/03.2-sessions-transactions.md)
    - 8.3.3 [Syntaxe et API](08-transactions/03.3-syntaxe-api.md)
    - 8.3.4 [Commit et rollback](08-transactions/03.4-commit-rollback.md)
- 8.4 [Niveaux de cohérence et d'isolation](08-transactions/04-niveaux-coherence-isolation.md)
    - 8.4.1 [Read Concern (local, available, majority, linearizable, snapshot)](08-transactions/04.1-read-concern.md)
    - 8.4.2 [Write Concern (w, j, wtimeout)](08-transactions/04.2-write-concern.md)
    - 8.4.3 [Compromis performance vs cohérence](08-transactions/04.3-compromis-performance-coherence.md)
- 8.5 [Transactions distribuées](08-transactions/05-transactions-distribuees.md)
- 8.6 [Limites et considérations de performance](08-transactions/06-limites-considerations-performance.md)
- 8.7 [Bonnes pratiques transactionnelles](08-transactions/07-bonnes-pratiques-transactionnelles.md)

---

## **[Partie 4 : Architecture Distribuée (Avancé)](/partie-04-architecture-distribuee.md)**

### 9. [Réplication](09-replication/README.md)
- 9.1 [Concepts de réplication](09-replication/01-concepts-replication.md)
- 9.2 [Architecture Replica Set](09-replication/02-architecture-replica-set.md)
- 9.3 [Membres d'un Replica Set](09-replication/03-membres-replica-set.md)
    - 9.3.1 [Primary](09-replication/03.1-primary.md)
    - 9.3.2 [Secondary](09-replication/03.2-secondary.md)
    - 9.3.3 [Arbiter](09-replication/03.3-arbiter.md)
    - 9.3.4 [Hidden](09-replication/03.4-hidden.md)
    - 9.3.5 [Delayed](09-replication/03.5-delayed.md)
- 9.4 [Élection du Primary](09-replication/04-election-primary.md)
- 9.5 [Oplog (Operations Log)](09-replication/05-oplog.md)
- 9.6 [Configuration d'un Replica Set](09-replication/06-configuration-replica-set.md)
- 9.7 [Ajout et suppression de membres](09-replication/07-ajout-suppression-membres.md)
- 9.8 [Read Preference](09-replication/08-read-preference.md)
- 9.9 [Failover et haute disponibilité](09-replication/09-failover-haute-disponibilite.md)
- 9.10 [Monitoring d'un Replica Set](09-replication/10-monitoring-replica-set.md)
- 9.11 [Maintenance et opérations courantes](09-replication/11-maintenance-operations-courantes.md)
- 9.12 [Réplication chaînée](09-replication/12-replication-chainee.md)

### 10. [Sharding (Partitionnement Horizontal)](10-sharding/README.md)
- 10.1 [Concepts du sharding](10-sharding/01-concepts-sharding.md)
- 10.2 [Architecture shardée](10-sharding/02-architecture-shardee.md)
    - 10.2.1 [Shards](10-sharding/02.1-shards.md)
    - 10.2.2 [Config Servers](10-sharding/02.2-config-servers.md)
    - 10.2.3 [Mongos (Query Routers)](10-sharding/02.3-mongos-query-routers.md)
- 10.3 [Shard Key : Choix et stratégies](10-sharding/03-shard-key-choix-strategies.md)
- 10.4 [Types de sharding](10-sharding/04-types-de-sharding.md)
    - 10.4.1 [Range Sharding](10-sharding/04.1-range-sharding.md)
    - 10.4.2 [Hashed Sharding](10-sharding/04.2-hashed-sharding.md)
    - 10.4.3 [Zone Sharding](10-sharding/04.3-zone-sharding.md)
- 10.5 [Chunks et balancing](10-sharding/05-chunks-balancing.md)
- 10.6 [Déploiement d'un cluster shardé](10-sharding/06-deploiement-cluster-sharde.md)
- 10.7 [Activer le sharding sur une base et une collection](10-sharding/07-activer-sharding.md)
- 10.8 [Migration des chunks](10-sharding/08-migration-chunks.md)
- 10.9 [Opérations sur un cluster shardé](10-sharding/09-operations-cluster-sharde.md)
- 10.10 [Monitoring et maintenance](10-sharding/10-monitoring-maintenance.md)
- 10.11 [Jumbo Chunks et résolution](10-sharding/11-jumbo-chunks-resolution.md)
- 10.12 [Bonnes pratiques de sharding](10-sharding/12-bonnes-pratiques-sharding.md)

---

## **[Partie 5 : Sécurité et Administration (Avancé)](/partie-05-securite-administration.md)**

### 11. [Sécurité](11-securite/README.md)
- 11.1 [Vue d'ensemble de la sécurité MongoDB](11-securite/01-vue-ensemble-securite.md)
- 11.2 [Authentification](11-securite/02-authentification.md)
    - 11.2.1 [SCRAM (défaut)](11-securite/02.1-scram.md)
    - 11.2.2 [x.509](11-securite/02.2-x509.md)
    - 11.2.3 [LDAP](11-securite/02.3-ldap.md)
    - 11.2.4 [Kerberos](11-securite/02.4-kerberos.md)
- 11.3 [Autorisation et rôles](11-securite/03-autorisation-roles.md)
    - 11.3.1 [Rôles intégrés](11-securite/03.1-roles-integres.md)
    - 11.3.2 [Rôles personnalisés](11-securite/03.2-roles-personnalises.md)
    - 11.3.3 [Privilèges et actions](11-securite/03.3-privileges-actions.md)
- 11.4 [Gestion des utilisateurs](11-securite/04-gestion-utilisateurs.md)
- 11.5 [Chiffrement](11-securite/05-chiffrement.md)
    - 11.5.1 [Chiffrement en transit (TLS/SSL)](11-securite/05.1-chiffrement-transit.md)
    - 11.5.2 [Chiffrement au repos](11-securite/05.2-chiffrement-repos.md)
    - 11.5.3 [Client-Side Field Level Encryption (CSFLE)](11-securite/05.3-csfle.md)
    - 11.5.4 [Queryable Encryption](11-securite/05.4-queryable-encryption.md)
- 11.6 [Audit](11-securite/06-audit.md)
- 11.7 [Network Security et IP Whitelisting](11-securite/07-network-security-ip-whitelisting.md)
- 11.8 [Security Checklist](11-securite/08-security-checklist.md)
- 11.9 [Conformité et certifications](11-securite/09-conformite-certifications.md)

### 12. [Sauvegarde et Restauration](12-sauvegarde-restauration/README.md)
- 12.1 [Stratégies de sauvegarde](12-sauvegarde-restauration/01-strategies-sauvegarde.md)
- 12.2 [mongodump et mongorestore](12-sauvegarde-restauration/02-mongodump-mongorestore.md)
- 12.3 [Sauvegarde de Replica Sets](12-sauvegarde-restauration/03-sauvegarde-replica-sets.md)
- 12.4 [Sauvegarde de clusters shardés](12-sauvegarde-restauration/04-sauvegarde-clusters-shardes.md)
- 12.5 [Snapshots du système de fichiers](12-sauvegarde-restauration/05-snapshots-systeme-fichiers.md)
- 12.6 [MongoDB Atlas Backup](12-sauvegarde-restauration/06-mongodb-atlas-backup.md)
- 12.7 [Point-in-Time Recovery](12-sauvegarde-restauration/07-point-in-time-recovery.md)
- 12.8 [Oplog pour la récupération](12-sauvegarde-restauration/08-oplog-recuperation.md)
- 12.9 [Restauration complète vs partielle](12-sauvegarde-restauration/09-restauration-complete-partielle.md)
- 12.10 [Automatisation des sauvegardes](12-sauvegarde-restauration/10-automatisation-sauvegardes.md)
- 12.11 [Tests de restauration](12-sauvegarde-restauration/11-tests-restauration.md)
- 12.12 [Bonnes pratiques](12-sauvegarde-restauration/12-bonnes-pratiques.md)

### 13. [Monitoring et Administration](13-monitoring-administration/README.md)
- 13.1 [Métriques clés à surveiller](13-monitoring-administration/01-metriques-cles.md)
- 13.2 [Commandes d'administration](13-monitoring-administration/02-commandes-administration.md)
    - 13.2.1 [serverStatus](13-monitoring-administration/02.1-serverstatus.md)
    - 13.2.2 [dbStats](13-monitoring-administration/02.2-dbstats.md)
    - 13.2.3 [collStats](13-monitoring-administration/02.3-collstats.md)
    - 13.2.4 [currentOp](13-monitoring-administration/02.4-currentop.md)
    - 13.2.5 [killOp](13-monitoring-administration/02.5-killop.md)
- 13.3 [Profiler de requêtes](13-monitoring-administration/03-profiler-requetes.md)
- 13.4 [Logs MongoDB](13-monitoring-administration/04-logs-mongodb.md)
- 13.5 [MongoDB Database Tools](13-monitoring-administration/05-mongodb-database-tools.md)
- 13.6 [mongostat et mongotop](13-monitoring-administration/06-mongostat-mongotop.md)
- 13.7 [Intégration avec Prometheus et Grafana](13-monitoring-administration/07-prometheus-grafana.md)
- 13.8 [MongoDB Ops Manager](13-monitoring-administration/08-mongodb-ops-manager.md)
- 13.9 [Alerting et notifications](13-monitoring-administration/09-alerting-notifications.md)
- 13.10 [Diagnostics avec FTDC](13-monitoring-administration/10-diagnostics-ftdc.md)
- 13.11 [Gestion de la mémoire et du cache WiredTiger](13-monitoring-administration/11-memoire-cache-wiredtiger.md)

---

## **[Partie 6 : MongoDB Atlas et Cloud (Avancé)](/partie-06-mongodb-atlas-cloud.md)**

### 14. [MongoDB Atlas](14-mongodb-atlas/README.md)
- 14.1 [Présentation de MongoDB Atlas](14-mongodb-atlas/01-presentation-mongodb-atlas.md)
- 14.2 [Création d'un cluster Atlas](14-mongodb-atlas/02-creation-cluster-atlas.md)
- 14.3 [Tiers gratuit (M0) et options payantes](14-mongodb-atlas/03-tiers-gratuit-options-payantes.md)
- 14.4 [Configuration réseau et sécurité](14-mongodb-atlas/04-configuration-reseau-securite.md)
- 14.5 [Connexion à Atlas](14-mongodb-atlas/05-connexion-atlas.md)
- 14.6 [Monitoring et alertes dans Atlas](14-mongodb-atlas/06-monitoring-alertes-atlas.md)
- 14.7 [Backups et restauration](14-mongodb-atlas/07-backups-restauration.md)
- 14.8 [Scaling (vertical et horizontal)](14-mongodb-atlas/08-scaling.md)
- 14.9 [Atlas Search](14-mongodb-atlas/09-atlas-search.md)
- 14.10 [Atlas Data Lake](14-mongodb-atlas/10-atlas-data-lake.md)
- 14.11 [Atlas Charts (visualisation)](14-mongodb-atlas/11-atlas-charts.md)
- 14.12 [Atlas App Services](14-mongodb-atlas/12-atlas-app-services.md)
- 14.13 [Atlas Vector Search](14-mongodb-atlas/13-atlas-vector-search.md)
- 14.14 [Triggers et fonctions serverless](14-mongodb-atlas/14-triggers-fonctions-serverless.md)
- 14.15 [Data API](14-mongodb-atlas/15-data-api.md)
- 14.16 [Atlas CLI](14-mongodb-atlas/16-atlas-cli.md)

---

## **[Partie 7 : Développement et Intégration (Intermédiaire à Avancé)](/partie-07-developpement-integration.md)**

### 15. [Drivers et Intégration Applicative](15-drivers-integration-applicative/README.md)
- 15.1 [Vue d'ensemble des drivers officiels](15-drivers-integration-applicative/01-vue-ensemble-drivers.md)
- 15.2 [Driver Node.js / JavaScript](15-drivers-integration-applicative/02-driver-nodejs-javascript.md)
- 15.3 [Driver Python (PyMongo)](15-drivers-integration-applicative/03-driver-python-pymongo.md)
- 15.4 [Driver Java](15-drivers-integration-applicative/04-driver-java.md)
- 15.5 [Driver C# / .NET](15-drivers-integration-applicative/05-driver-csharp-dotnet.md)
- 15.6 [Driver Go](15-drivers-integration-applicative/06-driver-go.md)
- 15.7 [Driver PHP](15-drivers-integration-applicative/07-driver-php.md)
- 15.8 [Driver Ruby](15-drivers-integration-applicative/08-driver-ruby.md)
- 15.9 [Connection String et options](15-drivers-integration-applicative/09-connection-string-options.md)
- 15.10 [Connection Pooling](15-drivers-integration-applicative/10-connection-pooling.md)
- 15.11 [Gestion des erreurs et retry](15-drivers-integration-applicative/11-gestion-erreurs-retry.md)
- 15.12 [ODM et ORM](15-drivers-integration-applicative/12-odm-orm.md)
    - 15.12.1 [Mongoose (Node.js)](15-drivers-integration-applicative/12.1-mongoose-nodejs.md)
    - 15.12.2 [Motor (Python async)](15-drivers-integration-applicative/12.2-motor-python-async.md)
    - 15.12.3 [Spring Data MongoDB (Java)](15-drivers-integration-applicative/12.3-spring-data-mongodb.md)
- 15.13 [Bonnes pratiques d'intégration](15-drivers-integration-applicative/13-bonnes-pratiques-integration.md)

### 16. [Fonctionnalités Avancées](16-fonctionnalites-avancees/README.md)
- 16.1 [Change Streams](16-fonctionnalites-avancees/01-change-streams.md)
    - 16.1.1 [Principes et cas d'usage](16-fonctionnalites-avancees/01.1-principes-cas-usage.md)
    - 16.1.2 [Configuration et filtres](16-fonctionnalites-avancees/01.2-configuration-filtres.md)
    - 16.1.3 [Resume tokens](16-fonctionnalites-avancees/01.3-resume-tokens.md)
- 16.2 [GridFS (stockage de fichiers volumineux)](16-fonctionnalites-avancees/02-gridfs.md)
- 16.3 [Capped Collections](16-fonctionnalites-avancees/03-capped-collections.md)
- 16.4 [Time Series Collections](16-fonctionnalites-avancees/04-time-series-collections.md)
- 16.5 [Clustered Collections](16-fonctionnalites-avancees/05-clustered-collections.md)
- 16.6 [Requêtes géospatiales avancées](16-fonctionnalites-avancees/06-requetes-geospatiales-avancees.md)
- 16.7 [Recherche Full-Text avancée](16-fonctionnalites-avancees/07-recherche-full-text-avancee.md)
- 16.8 [Atlas Search et Lucene](16-fonctionnalites-avancees/08-atlas-search-lucene.md)
- 16.9 [Vector Search et IA](16-fonctionnalites-avancees/09-vector-search-ia.md)
- 16.10 [MongoDB et GraphQL](16-fonctionnalites-avancees/10-mongodb-graphql.md)

---

## **[Partie 8 : Performance et Production (Expert)](/partie-08-performance-production.md)**

### 17. [Performance et Tuning](17-performance-tuning/README.md)
- 17.1 [Identification des goulots d'étranglement](17-performance-tuning/01-identification-goulots.md)
- 17.2 [Analyse avec explain() approfondie](17-performance-tuning/02-analyse-explain-approfondie.md)
- 17.3 [Optimisation de la modélisation](17-performance-tuning/03-optimisation-modelisation.md)
- 17.4 [Optimisation des index](17-performance-tuning/04-optimisation-index.md)
- 17.5 [Optimisation des agrégations](17-performance-tuning/05-optimisation-agregations.md)
- 17.6 [Configuration du moteur WiredTiger](17-performance-tuning/06-configuration-wiredtiger.md)
- 17.7 [Dimensionnement matériel](17-performance-tuning/07-dimensionnement-materiel.md)
- 17.8 [Paramètres de configuration avancés](17-performance-tuning/08-parametres-configuration-avances.md)
- 17.9 [Compression des données](17-performance-tuning/09-compression-donnees.md)
- 17.10 [Read/Write splitting](17-performance-tuning/10-read-write-splitting.md)
- 17.11 [Caching strategies](17-performance-tuning/11-caching-strategies.md)
- 17.12 [Benchmarking et tests de charge](17-performance-tuning/12-benchmarking-tests-charge.md)

### 18. [DevOps et Déploiement](18-devops-deploiement/README.md)
- 18.1 [Infrastructure as Code pour MongoDB](18-devops-deploiement/01-infrastructure-as-code.md)
- 18.2 [Déploiement avec Docker](18-devops-deploiement/02-deploiement-docker.md)
- 18.3 [Docker Compose pour environnements de dev](18-devops-deploiement/03-docker-compose-dev.md)
- 18.4 [Kubernetes et MongoDB](18-devops-deploiement/04-kubernetes-mongodb.md)
    - 18.4.1 [MongoDB Community Kubernetes Operator](18-devops-deploiement/04.1-community-kubernetes-operator.md)
    - 18.4.2 [MongoDB Enterprise Kubernetes Operator](18-devops-deploiement/04.2-enterprise-kubernetes-operator.md)
- 18.5 [Helm Charts](18-devops-deploiement/05-helm-charts.md)
- 18.6 [Ansible pour MongoDB](18-devops-deploiement/06-ansible-mongodb.md)
- 18.7 [Terraform et MongoDB Atlas](18-devops-deploiement/07-terraform-mongodb-atlas.md)
- 18.8 [CI/CD et migrations de schéma](18-devops-deploiement/08-cicd-migrations-schema.md)
- 18.9 [Blue/Green deployments](18-devops-deploiement/09-blue-green-deployments.md)
- 18.10 [Gestion des configurations](18-devops-deploiement/10-gestion-configurations.md)
- 18.11 [Environnements multi-régions](18-devops-deploiement/11-environnements-multi-regions.md)

### 19. [Migration et Intégration](19-migration-integration/README.md)
- 19.1 [Migration depuis SQL vers MongoDB](19-migration-integration/01-migration-sql-vers-mongodb.md)
- 19.2 [Outils de migration](19-migration-integration/02-outils-migration.md)
- 19.3 [Relational Migrator](19-migration-integration/03-relational-migrator.md)
- 19.4 [Stratégies de migration incrémentale](19-migration-integration/04-strategies-migration-incrementale.md)
- 19.5 [Synchronisation bidirectionnelle](19-migration-integration/05-synchronisation-bidirectionnelle.md)
- 19.6 [MongoDB Connector for BI](19-migration-integration/06-mongodb-connector-bi.md)
- 19.7 [Intégration avec Apache Kafka](19-migration-integration/07-integration-apache-kafka.md)
- 19.8 [Intégration avec Apache Spark](19-migration-integration/08-integration-apache-spark.md)
- 19.9 [ETL et Data Pipelines](19-migration-integration/09-etl-data-pipelines.md)
- 19.10 [Coexistence avec des bases relationnelles](19-migration-integration/10-coexistence-bases-relationnelles.md)

---

## **[Partie 9 : Cas d'Usage et Bonnes Pratiques (Transversal)](/partie-09-cas-usage-bonnes-pratiques.md)**

### 20. [Cas d'Usage et Architectures](20-cas-usage-architectures/README.md)
- 20.1 [Applications web et mobiles](20-cas-usage-architectures/01-applications-web-mobiles.md)
- 20.2 [Gestion de contenu (CMS)](20-cas-usage-architectures/02-gestion-contenu-cms.md)
- 20.3 [Catalogue produits (e-commerce)](20-cas-usage-architectures/03-catalogue-produits-ecommerce.md)
- 20.4 [Internet des objets (IoT)](20-cas-usage-architectures/04-internet-des-objets-iot.md)
- 20.5 [Gaming et leaderboards](20-cas-usage-architectures/05-gaming-leaderboards.md)
- 20.6 [Analyse en temps réel](20-cas-usage-architectures/06-analyse-temps-reel.md)
- 20.7 [Gestion des logs](20-cas-usage-architectures/07-gestion-logs.md)
- 20.8 [Personnalisation et recommandations](20-cas-usage-architectures/08-personnalisation-recommandations.md)
- 20.9 [Applications financières](20-cas-usage-architectures/09-applications-financieres.md)
- 20.10 [Architecture microservices](20-cas-usage-architectures/10-architecture-microservices.md)
- 20.11 [Event Sourcing avec MongoDB](20-cas-usage-architectures/11-event-sourcing-mongodb.md)
- 20.12 [CQRS et MongoDB](20-cas-usage-architectures/12-cqrs-mongodb.md)

### 21. [Bonnes Pratiques et Anti-patterns](21-bonnes-pratiques-anti-patterns/README.md)
- 21.1 [Conventions de nommage](21-bonnes-pratiques-anti-patterns/01-conventions-nommage.md)
- 21.2 [Gestion des _id](21-bonnes-pratiques-anti-patterns/02-gestion-ids.md)
- 21.3 [Gestion des null et valeurs manquantes](21-bonnes-pratiques-anti-patterns/03-gestion-null-valeurs-manquantes.md)
- 21.4 [Éviter les documents trop volumineux](21-bonnes-pratiques-anti-patterns/04-eviter-documents-volumineux.md)
- 21.5 [Éviter les collections excessives](21-bonnes-pratiques-anti-patterns/05-eviter-collections-excessives.md)
- 21.6 [Gestion des migrations de schéma](21-bonnes-pratiques-anti-patterns/06-gestion-migrations-schema.md)
- 21.7 [Versioning des documents](21-bonnes-pratiques-anti-patterns/07-versioning-documents.md)
- 21.8 [Gestion des environnements](21-bonnes-pratiques-anti-patterns/08-gestion-environnements.md)
- 21.9 [Documentation et commentaires](21-bonnes-pratiques-anti-patterns/09-documentation-commentaires.md)
- 21.10 [Revue de code pour MongoDB](21-bonnes-pratiques-anti-patterns/10-revue-code-mongodb.md)
- 21.11 [Checklist de mise en production](21-bonnes-pratiques-anti-patterns/11-checklist-mise-production.md)

### 22. [Dépannage et Résolution de Problèmes](22-depannage-resolution-problemes/README.md)
- 22.1 [Problèmes de connexion](22-depannage-resolution-problemes/01-problemes-connexion.md)
- 22.2 [Problèmes de performance](22-depannage-resolution-problemes/02-problemes-performance.md)
- 22.3 [Problèmes de réplication](22-depannage-resolution-problemes/03-problemes-replication.md)
- 22.4 [Problèmes de sharding](22-depannage-resolution-problemes/04-problemes-sharding.md)
- 22.5 [Corruption de données](22-depannage-resolution-problemes/05-corruption-donnees.md)
- 22.6 [Récupération après panne](22-depannage-resolution-problemes/06-recuperation-apres-panne.md)
- 22.7 [Analyse des logs d'erreurs](22-depannage-resolution-problemes/07-analyse-logs-erreurs.md)
- 22.8 [Support MongoDB et ressources](22-depannage-resolution-problemes/08-support-mongodb-ressources.md)
- 22.9 [Communauté et forums](22-depannage-resolution-problemes/09-communaute-forums.md)

---

## **[Partie 10 : Conclusion et Perspectives](/partie-10-conclusion-perspectives.md)**

### 23. [Nouveautés et Évolutions](23-nouveautes-evolutions/README.md)
- 23.1 [Historique des versions majeures](23-nouveautes-evolutions/01-historique-versions-majeures.md)
- 23.2 [Nouveautés MongoDB 6.x](23-nouveautes-evolutions/02-nouveautes-mongodb-6x.md)
- 23.3 [Nouveautés MongoDB 7.x](23-nouveautes-evolutions/03-nouveautes-mongodb-7x.md)
- 23.4 [Nouveautés MongoDB 8.x](23-nouveautes-evolutions/04-nouveautes-mongodb-8x.md)
- 23.5 [Roadmap et fonctionnalités futures](23-nouveautes-evolutions/05-roadmap-fonctionnalites-futures.md)
- 23.6 [Veille technologique et ressources](23-nouveautes-evolutions/06-veille-technologique-ressources.md)

---

## **Annexes**

### A. [Glossaire des Termes Techniques](annexes/glossaire/README.md)
- A.1 [Termes MongoDB essentiels (BSON, Replica Set, Shard, Chunk, etc.)](annexes/glossaire/01-termes-mongodb-essentiels.md)
- A.2 [Acronymes courants (CAP, ACID, TTL, WiredTiger, etc.)](annexes/glossaire/02-acronymes-courants.md)

### B. [Commandes mongosh Essentielles](annexes/commandes-mongosh/README.md)
- B.1 [Navigation (show dbs, use, show collections, etc.)](annexes/commandes-mongosh/01-navigation.md)
- B.2 [CRUD rapide](annexes/commandes-mongosh/02-crud-rapide.md)
- B.3 [Administration (rs.status(), sh.status(), etc.)](annexes/commandes-mongosh/03-administration.md)
- B.4 [Helpers et raccourcis](annexes/commandes-mongosh/04-helpers-raccourcis.md)

### C. [Requêtes MongoDB de Référence](annexes/requetes-reference/README.md)
- C.1 [Requêtes d'administration (index usage, collection stats)](annexes/requetes-reference/01-requetes-administration.md)
- C.2 [Requêtes de monitoring (currentOp, profiler)](annexes/requetes-reference/02-requetes-monitoring.md)
- C.3 [Pipelines d'agrégation courants](annexes/requetes-reference/03-pipelines-agregation-courants.md)

### D. [Configuration de Référence par Cas d'Usage](annexes/configuration-reference/README.md)
- D.1 [Configuration Replica Set (3 nœuds)](annexes/configuration-reference/01-configuration-replica-set.md)
- D.2 [Configuration Sharded Cluster](annexes/configuration-reference/02-configuration-sharded-cluster.md)
- D.3 [Configuration développement local](annexes/configuration-reference/03-configuration-developpement-local.md)
- D.4 [Configuration haute performance](annexes/configuration-reference/04-configuration-haute-performance.md)

### E. [Checklist de Performance](annexes/checklist-performance/README.md)
- E.1 [Audit de modélisation](annexes/checklist-performance/01-audit-modelisation.md)
- E.2 [Audit d'indexation](annexes/checklist-performance/02-audit-indexation.md)
- E.3 [Audit de requêtes](annexes/checklist-performance/03-audit-requetes.md)
- E.4 [Audit d'infrastructure](annexes/checklist-performance/04-audit-infrastructure.md)

### F. [Docker et Docker Compose](annexes/docker-compose/README.md)
- F.1 [MongoDB standalone](annexes/docker-compose/01-mongodb-standalone.md)
- F.2 [Replica Set avec Docker Compose](annexes/docker-compose/02-replica-set-docker-compose.md)
- F.3 [Sharded Cluster avec Docker Compose](annexes/docker-compose/03-sharded-cluster-docker-compose.md)
- F.4 [Stack complète (MongoDB + Mongo Express + Application)](annexes/docker-compose/04-stack-complete.md)

### G. [Scripts et Automatisation](annexes/scripts-automatisation/README.md)
- G.1 [Scripts de backup](annexes/scripts-automatisation/01-scripts-backup.md)
- G.2 [Scripts de monitoring](annexes/scripts-automatisation/02-scripts-monitoring.md)
- G.3 [Scripts de maintenance](annexes/scripts-automatisation/03-scripts-maintenance.md)
- G.4 [Playbooks Ansible](annexes/scripts-automatisation/04-playbooks-ansible.md)

---

## **Points Forts de Cette Formation**

| Caractéristique | Description |
|-----------------|-------------|
| ✅ **Progressive** | Parcours structuré du débutant à l'expert en 10 parties |
| ✅ **Complète** | 23 chapitres + 7 annexes couvrant tous les aspects |
| ✅ **Théorique solide** | CAP, ACID, patterns de modélisation |
| ✅ **Orientée production** | Sécurité, monitoring, HA, disaster recovery |
| ✅ **Cloud-native** | MongoDB Atlas, Kubernetes, Docker |
| ✅ **DevOps-friendly** | CI/CD, IaC, Terraform, Ansible |
| ✅ **Pratique** | Annexes, checklists et configurations de référence |

---

**Durée estimée** : 50-70 heures de formation théorique
**Public** : Développeurs, DevOps, SRE — Débutants à experts
**Prérequis** : Notions de base en programmation et systèmes Unix/Linux

---

**Version** : 2.0 — Novembre 2025
**MongoDB** : Versions 6.x / 7.x / 8.x
**Licence** : Creative Commons BY-NC-SA 4.0
