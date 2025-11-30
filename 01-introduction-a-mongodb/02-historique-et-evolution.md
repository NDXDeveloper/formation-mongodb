🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.2 Historique et évolution de MongoDB

## Introduction

Comprendre l'histoire de MongoDB permet de mieux appréhender ses choix de conception et son positionnement dans le paysage des bases de données. De ses débuts modestes en 2007 à son statut actuel de leader des bases de données NoSQL, MongoDB a connu une évolution remarquable qui reflète les transformations du monde du développement logiciel.

---

## Les origines : 2007-2009

### Le contexte de création

MongoDB est né au sein de **10gen**, une entreprise fondée en 2007 à New York par trois entrepreneurs :

- **Dwight Merriman** : cofondateur de DoubleClick (racheté par Google)
- **Eliot Horowitz** : ancien directeur technique de ShopWiki
- **Kevin Ryan** : ancien PDG de DoubleClick

À l'origine, 10gen développait une plateforme cloud complète (Platform as a Service) similaire à ce que propose aujourd'hui Google App Engine. Cette plateforme devait inclure un serveur d'applications et une base de données.

### La naissance de MongoDB

En travaillant sur leur plateforme, les fondateurs ont réalisé que les bases de données relationnelles existantes ne répondaient pas à leurs besoins de scalabilité et de flexibilité. Ils ont donc créé leur propre base de données orientée documents.

Le nom **MongoDB** vient du mot anglais "**humongous**" (qui signifie "énorme" ou "gigantesque"), reflétant l'ambition de gérer d'immenses volumes de données.

En 2009, 10gen décide d'abandonner le projet de plateforme cloud pour se concentrer exclusivement sur MongoDB, qu'ils publient en **open source** sous licence AGPL.

---

## Les premières années : 2009-2013

### Version 1.0 (août 2009)

La première version stable de MongoDB est publiée en août 2009. Elle pose les fondations du système :

- Stockage orienté documents au format BSON
- Requêtes dynamiques
- Indexation
- Réplication (replica sets primitifs)

### Adoption croissante

MongoDB gagne rapidement en popularité auprès des développeurs grâce à :

- Sa facilité d'utilisation
- Son modèle de données flexible
- Sa bonne intégration avec les langages de programmation modernes
- Sa documentation de qualité

Des entreprises comme **Foursquare**, **Craigslist** et **The New York Times** commencent à adopter MongoDB pour leurs applications.

### Versions majeures de cette période

| Version | Date | Nouveautés principales |
|---------|------|------------------------|
| 1.0 | Août 2009 | Première version stable |
| 1.2 | Décembre 2009 | Map-Reduce, améliorations des index |
| 1.4 | Mars 2010 | Requêtes géospatiales |
| 1.6 | Août 2010 | Sharding production-ready, Replica Sets |
| 2.0 | Septembre 2011 | Journaling par défaut, améliorations de performance |
| 2.2 | Août 2012 | Framework d'agrégation, TTL collections |
| 2.4 | Mars 2013 | Index textuels, améliorations géospatiales |

---

## La maturation : 2013-2017

### Changement de nom de l'entreprise

En 2013, l'entreprise **10gen** est renommée **MongoDB Inc.** pour s'aligner sur le nom de son produit phare, devenu mondialement reconnu.

### Version 2.6 (avril 2014)

Cette version marque un tournant avec l'introduction de nombreuses fonctionnalités orientées entreprise :

- Améliorations majeures du framework d'agrégation
- Opérations d'écriture en masse (bulk operations)
- Validation des requêtes
- Améliorations de sécurité

### Version 3.0 (mars 2015) : WiredTiger

La version 3.0 est une étape majeure avec l'introduction du moteur de stockage **WiredTiger** :

- Compression des données
- Verrouillage au niveau du document (au lieu de la collection)
- Performances d'écriture considérablement améliorées
- Meilleure gestion de la mémoire

WiredTiger deviendra le moteur de stockage par défaut à partir de la version 3.2.

### Version 3.2 (décembre 2015)

- WiredTiger devient le moteur par défaut
- Validation des documents (schema validation)
- Opérateur `$lookup` pour les jointures
- Chiffrement au repos (Enterprise)

### Version 3.4 (novembre 2016)

- Vues (views)
- Collations pour le tri linguistique
- Zones de sharding
- Faceted search dans les agrégations

### Version 3.6 (novembre 2017)

- **Change Streams** : écoute des modifications en temps réel
- Sessions clients
- Améliorations du sharding
- JSON Schema validation

---

## L'ère des transactions : 2018-2020

### Introduction en bourse (octobre 2017)

MongoDB Inc. entre en bourse au NASDAQ sous le symbole **MDB**, valorisant l'entreprise à plus de 1,5 milliard de dollars. Cette introduction en bourse témoigne de la maturité et de l'adoption massive de MongoDB.

### Version 4.0 (juin 2018) : Transactions multi-documents

La version 4.0 répond à l'une des critiques historiques de MongoDB avec l'introduction des **transactions ACID multi-documents** :

- Transactions sur un Replica Set
- Propriétés ACID complètes
- Syntaxe familière (start_transaction, commit, abort)

Cette fonctionnalité permet à MongoDB de couvrir des cas d'utilisation auparavant réservés aux bases relationnelles.

### Version 4.2 (août 2019)

- **Transactions distribuées** (sur clusters shardés)
- Index Wildcard
- Chiffrement au niveau des champs (Field Level Encryption)
- Opérations de mise à jour avec pipeline d'agrégation

### Version 4.4 (juillet 2020)

- Améliorations des performances de sharding
- Hedged reads pour réduire la latence
- Améliorations du framework d'agrégation
- Custom aggregation expressions

---

## MongoDB moderne : 2021 à aujourd'hui

### Version 5.0 (juillet 2021)

La version 5.0 introduit des fonctionnalités innovantes :

- **Time Series Collections** : stockage optimisé pour les données temporelles
- **Versioning des API** : stabilité pour les applications
- **Resharding en ligne** : modification de la shard key sans interruption
- Window functions dans les agrégations

### Version 6.0 (juillet 2022)

- **Queryable Encryption** : recherche sur des données chiffrées
- Améliorations des time series
- Opérateurs d'agrégation supplémentaires
- Améliorations de Change Streams

### Version 7.0 (août 2023)

- **Compound Wildcard Indexes**
- Améliorations de performance significatives
- Sharding plus efficace
- Nouvelles fonctionnalités de sécurité

### Version 8.0 (2024)

- Améliorations de performance continues
- Nouvelles fonctionnalités d'agrégation
- Optimisations du moteur de stockage
- Support amélioré pour les architectures modernes

---

## L'évolution de MongoDB Atlas

### Lancement (2016)

En 2016, MongoDB lance **MongoDB Atlas**, son service de base de données cloud entièrement géré. Atlas permet de déployer, gérer et faire évoluer des clusters MongoDB sans gérer l'infrastructure.

### Expansion des fonctionnalités

Au fil des années, Atlas s'est enrichi de nombreux services :

| Année | Fonctionnalité |
|-------|----------------|
| 2016 | Lancement d'Atlas |
| 2018 | Atlas sur Azure et GCP |
| 2019 | **Atlas Search** (recherche full-text basée sur Lucene) |
| 2020 | Atlas Data Lake |
| 2021 | **Atlas Charts** (visualisation de données) |
| 2022 | Atlas Device Sync (synchronisation mobile) |
| 2023 | **Atlas Vector Search** (pour l'IA et le machine learning) |
| 2024 | Atlas Stream Processing |

### Position sur le marché

Aujourd'hui, Atlas représente une part significative du chiffre d'affaires de MongoDB Inc. et est utilisé par des milliers d'entreprises dans le monde.

---

## Chronologie visuelle

```
2007 ─── Fondation de 10gen
    │
2009 ─── MongoDB 1.0 (première version stable)
    │    Publication en open source
    │
2010 ─── MongoDB 1.6 : Sharding et Replica Sets
    │
2012 ─── MongoDB 2.2 : Framework d'agrégation
    │
2013 ─── 10gen devient MongoDB Inc.
    │
2015 ─── MongoDB 3.0 : Moteur WiredTiger
    │
2016 ─── Lancement de MongoDB Atlas
    │
2017 ─── Introduction en bourse (NASDAQ: MDB)
    │    MongoDB 3.6 : Change Streams
    │
2018 ─── MongoDB 4.0 : Transactions multi-documents
    │
2019 ─── MongoDB 4.2 : Transactions distribuées
    │
2021 ─── MongoDB 5.0 : Time Series Collections
    │
2022 ─── MongoDB 6.0 : Queryable Encryption
    │
2023 ─── MongoDB 7.0 : Améliorations de performance
    │    Atlas Vector Search
    │
2024 ─── MongoDB 8.0 : Dernière version majeure
```

---

## Évolution de la licence

### Licence AGPL (2009-2018)

À ses débuts, MongoDB était distribué sous licence **AGPL v3** (GNU Affero General Public License), une licence open source copyleft.

### Licence SSPL (2018)

En octobre 2018, MongoDB passe à la licence **SSPL** (Server Side Public License). Cette décision vise à empêcher les fournisseurs cloud (AWS, Azure, GCP) de proposer MongoDB "as a Service" sans contribuer au projet.

Cette décision a été controversée :

- Certains considèrent la SSPL comme non open source
- Des distributions Linux ont retiré MongoDB de leurs dépôts
- Des forks comme Amazon DocumentDB sont apparus

MongoDB reste cependant gratuit pour la plupart des usages, et le code source est toujours disponible.

---

## L'impact de MongoDB sur l'industrie

### Popularisation du NoSQL

MongoDB a joué un rôle majeur dans la popularisation des bases de données NoSQL. Il a démontré qu'il était possible de :

- Gérer des données sans schéma rigide
- Atteindre des performances élevées avec un modèle non relationnel
- Offrir une expérience développeur moderne

### Influence sur les autres bases de données

Le succès de MongoDB a poussé d'autres bases de données à évoluer :

- **PostgreSQL** a ajouté le support JSON/JSONB
- **MySQL** a introduit le type JSON
- De nouveaux produits comme **Amazon DocumentDB** et **Azure Cosmos DB** ont émergé

### Classements et reconnaissance

MongoDB figure régulièrement dans les classements des bases de données les plus populaires :

- Top 5 du classement **DB-Engines** depuis plusieurs années
- Base de données NoSQL documentaire #1 mondiale
- Utilisée par des entreprises du Fortune 500

---

## Conclusion

En moins de deux décennies, MongoDB est passé d'un composant d'un projet de plateforme cloud à l'une des bases de données les plus influentes au monde. Son évolution reflète les besoins changeants des développeurs et des entreprises :

- **2009-2013** : Fondations et adoption initiale
- **2013-2017** : Maturation et fonctionnalités entreprise
- **2018-2020** : Transactions ACID et parité avec le relationnel
- **2021+** : Innovation continue (time series, IA, chiffrement avancé)

Cette évolution constante, combinée à l'écosystème Atlas et à la communauté active, positionne MongoDB comme un choix de premier plan pour les applications modernes.

---

## Points clés à retenir

- MongoDB a été créé en **2007** par les fondateurs de 10gen
- Le nom vient de "**humongous**" (énorme)
- La version **1.0** est sortie en **août 2009**
- Le moteur **WiredTiger** (v3.0, 2015) a révolutionné les performances
- Les **transactions multi-documents** sont arrivées avec la v4.0 (2018)
- **MongoDB Atlas** (2016) est devenu un pilier de l'offre commerciale
- MongoDB continue d'innover avec les time series, le vector search et le chiffrement avancé

---


⏭️ [Bases de données NoSQL vs SQL : Comparaison conceptuelle](/01-introduction-a-mongodb/03-nosql-vs-sql.md)
