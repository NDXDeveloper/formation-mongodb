🔝 Retour au [Sommaire](/SOMMAIRE.md)

# A.2 - Acronymes Courants

## Table des matières

- [A](#a) | [B](#b) | [C](#c) | [D](#d) | [E](#e) | [F](#f) | [G](#g) | [H](#h) | [I](#i) | [J](#j) | [K](#k) | [L](#l) | [M](#m) | [N](#n) | [O](#o) | [P](#p) | [R](#r) | [S](#s) | [T](#t) | [U](#u) | [W](#w)

---

## A

### ACID **[Intermédiaire]**
🔤 **Atomicity, Consistency, Isolation, Durability**

💡 **Définition** : Ensemble de propriétés garantissant la fiabilité des transactions dans les bases de données.

📌 **Détails** :
- **Atomicity** : Tout ou rien (transaction complète ou annulée)
- **Consistency** : Respect des règles d'intégrité
- **Isolation** : Les transactions concurrentes ne s'interfèrent pas
- **Durability** : Les données validées sont permanentes

💡 **Dans MongoDB** :
- ACID natif au niveau document
- ACID multi-documents via transactions (depuis v4.0)

🔗 **Voir aussi** : BASE, Transaction, Write Concern, Read Concern

---

### API **[Débutant]**
🔤 **Application Programming Interface**

💡 **Définition** : Interface permettant à des applications de communiquer entre elles.

💡 **Dans MongoDB** :
- Driver API pour les langages (Node.js, Python, Java, etc.)
- MongoDB Data API (REST)
- Atlas Admin API

📌 **Exemple** : Driver Node.js expose l'API MongoDB pour JavaScript.

---

### AWS **[Intermédiaire]**
🔤 **Amazon Web Services**

💡 **Définition** : Plateforme cloud d'Amazon, l'un des trois principaux fournisseurs cloud pour MongoDB Atlas.

💡 **Services pertinents** : EC2 (serveurs), EBS (stockage), VPC (réseau), S3 (backups).

🔗 **Voir aussi** : GCP, Azure, Atlas

---

### Azure **[Intermédiaire]**
🔤 **Microsoft Azure**

💡 **Définition** : Plateforme cloud de Microsoft, disponible pour MongoDB Atlas.

💡 **Services pertinents** : Virtual Machines, Managed Disks, Virtual Network, Blob Storage.

🔗 **Voir aussi** : AWS, GCP, Atlas

---

## B

### BASE **[Avancé]**
🔤 **Basically Available, Soft state, Eventually consistent**

💡 **Définition** : Modèle alternatif à ACID pour les systèmes distribués, privilégiant la disponibilité à la cohérence stricte.

📌 **Détails** :
- **Basically Available** : Système disponible même en cas de pannes partielles
- **Soft state** : État peut changer sans interaction (réplication en cours)
- **Eventually consistent** : Cohérence atteinte après un délai

💡 **Relation avec MongoDB** : MongoDB peut être configuré selon BASE ou ACID selon les besoins (Read/Write Concern).

🔗 **Voir aussi** : ACID, CAP, Eventual Consistency

---

### BI **[Intermédiaire]**
🔤 **Business Intelligence**

💡 **Définition** : Ensemble d'outils et processus d'analyse de données pour la prise de décision.

💡 **Dans MongoDB** : MongoDB Connector for BI permet d'utiliser des outils BI (Tableau, Power BI) avec MongoDB.

🔗 **Voir aussi** : Atlas Charts, ETL, OLAP

---

### BSON **[Débutant]**
🔤 **Binary JSON**

💡 **Définition** : Format de sérialisation binaire utilisé par MongoDB, extension de JSON avec types additionnels.

💡 **Avantages** :
- Plus compact que JSON
- Plus rapide à parser
- Support de types supplémentaires (Date, ObjectId, Binary, Decimal128, etc.)

📌 **Exemple** : Un document JSON est automatiquement converti en BSON lors du stockage.

⚠️ **Limite** : Taille maximale d'un document BSON = 16 Mo

🔗 **Voir aussi** : JSON, Document, ObjectId

---

## C

### CAP **[Avancé]**
🔤 **Consistency, Availability, Partition tolerance** (Théorème CAP ou théorème de Brewer)

💡 **Définition** : Théorème stipulant qu'un système distribué ne peut garantir simultanément que 2 des 3 propriétés suivantes :
- **Consistency** : Tous les nœuds voient les mêmes données au même moment
- **Availability** : Chaque requête reçoit une réponse (succès ou échec)
- **Partition tolerance** : Le système continue de fonctionner malgré des coupures réseau

💡 **Positionnement MongoDB** : CP (Consistency + Partition tolerance) par défaut, mais configurable vers AP selon Write Concern et Read Concern.

📌 **Exemple** :
- Write Concern "majority" → CP
- Read Preference "nearest" → AP

🔗 **Voir aussi** : ACID, BASE, Write Concern, Read Concern

---

### CI/CD **[Intermédiaire]**
🔤 **Continuous Integration / Continuous Deployment**

💡 **Définition** : Pratiques DevOps d'intégration et déploiement continus.

💡 **Dans MongoDB** :
- Migrations de schéma automatisées
- Déploiement de scripts d'administration
- Tests d'intégration avec MongoDB
- Gestion de configurations (Atlas Terraform Provider)

🔗 **Voir aussi** : DevOps, IaC, Terraform

---

### CLI **[Débutant]**
🔤 **Command-Line Interface**

💡 **Définition** : Interface en ligne de commande.

💡 **Dans MongoDB** :
- mongosh (MongoDB Shell)
- mongodump, mongorestore
- Atlas CLI
- Database Tools

🔗 **Voir aussi** : mongosh, GUI, Compass

---

### CPU **[Débutant]**
🔤 **Central Processing Unit**

💡 **Définition** : Processeur, ressource critique pour MongoDB.

💡 **Usage dans MongoDB** :
- Traitement des requêtes
- Calculs d'agrégation
- Indexation
- Compression/décompression

🎯 **Dimensionnement** : Multi-cœurs recommandé pour production (8+ cœurs).

🔗 **Voir aussi** : RAM, IOPS, Performance

---

### CRUD **[Débutant]**
🔤 **Create, Read, Update, Delete**

💡 **Définition** : Les quatre opérations de base sur les données.

💡 **Dans MongoDB** :
- **Create** : `insertOne()`, `insertMany()`
- **Read** : `find()`, `findOne()`
- **Update** : `updateOne()`, `updateMany()`, `replaceOne()`
- **Delete** : `deleteOne()`, `deleteMany()`

📌 **Exemple** :
```javascript
db.users.insertOne({name: "Alice"})  // Create
db.users.find({name: "Alice"})        // Read
db.users.updateOne({name: "Alice"}, {$set: {age: 30}}) // Update
db.users.deleteOne({name: "Alice"})   // Delete
```

---

### CSFLE **[Avancé]**
🔤 **Client-Side Field Level Encryption**

💡 **Définition** : Chiffrement des données au niveau des champs, effectué côté client avant envoi à MongoDB.

💡 **Avantages** :
- Sécurité maximale (MongoDB ne voit jamais les données en clair)
- Conformité RGPD, HIPAA, etc.
- Granularité au niveau du champ

⚠️ **Limitation** : Champs chiffrés non interrogeables (sauf Queryable Encryption).

🔗 **Voir aussi** : Queryable Encryption, TLS, Chiffrement

---

### CSV **[Débutant]**
🔤 **Comma-Separated Values**

💡 **Définition** : Format de fichier texte pour données tabulaires.

💡 **Dans MongoDB** :
- Import via `mongoimport --type csv`
- Export via `mongoexport --type csv`

📌 **Cas d'usage** : Migration depuis Excel, intégration avec outils analytics.

---

## D

### DBaaS **[Intermédiaire]**
🔤 **Database as a Service**

💡 **Définition** : Service de base de données géré dans le cloud (infrastructure et administration automatisées).

💡 **Exemple** : MongoDB Atlas est un DBaaS.

💡 **Avantages** : Pas de gestion serveurs, backups automatiques, scaling simplifié.

🔗 **Voir aussi** : Atlas, SaaS, PaaS, IaaS

---

### DDL **[Intermédiaire]**
🔤 **Data Definition Language**

💡 **Définition** : Langage de définition de structure de données (en SQL : CREATE, ALTER, DROP).

💡 **Dans MongoDB** :
- Création de collections
- Création d'index
- Validation de schéma
- Modification de structure

📌 **Exemple** : `db.createCollection("users", {validator: {...}})`

---

### DML **[Intermédiaire]**
🔤 **Data Manipulation Language**

💡 **Définition** : Langage de manipulation de données (en SQL : SELECT, INSERT, UPDATE, DELETE).

💡 **Dans MongoDB** : Équivalent aux opérations CRUD.

---

## E

### ETL **[Intermédiaire]**
🔤 **Extract, Transform, Load**

💡 **Définition** : Processus d'extraction, transformation et chargement de données.

💡 **Dans MongoDB** :
- Extraction : Change Streams, exports
- Transformation : Aggregation Pipeline
- Load : `insertMany()`, Bulk Operations

💡 **Outils** : Apache Kafka, Apache Spark, Airflow.

🔗 **Voir aussi** : BI, Data Pipeline, Change Streams

---

## F

### FIFO **[Intermédiaire]**
🔤 **First In, First Out**

💡 **Définition** : Premier entré, premier sorti.

💡 **Dans MongoDB** :
- Capped Collections (ordre FIFO)
- Oplog (ordre FIFO)
- TTL Index (expiration FIFO)

🔗 **Voir aussi** : Capped Collection, Oplog, TTL

---

### FTDC **[Avancé]**
🔤 **Full-Time Diagnostic Data Capture**

💡 **Définition** : Système de collecte automatique de métriques de diagnostic MongoDB.

💡 **Contenu** : CPU, RAM, requêtes, opérations, réplication, etc.

💡 **Usage** : Analyse de performance, troubleshooting, support MongoDB.

📌 **Localisation** : Fichiers dans `/data/db/diagnostic.data/`

🔗 **Voir aussi** : Monitoring, Profiler, Logs

---

## G

### GCP **[Intermédiaire]**
🔤 **Google Cloud Platform**

💡 **Définition** : Plateforme cloud de Google, disponible pour MongoDB Atlas.

💡 **Services pertinents** : Compute Engine, Persistent Disk, VPC, Cloud Storage.

🔗 **Voir aussi** : AWS, Azure, Atlas

---

### GUI **[Débutant]**
🔤 **Graphical User Interface**

💡 **Définition** : Interface graphique utilisateur.

💡 **Dans MongoDB** :
- MongoDB Compass
- Atlas UI (interface web)
- Studio 3T (tiers)
- Robo 3T (tiers)

🔗 **Voir aussi** : CLI, Compass, Atlas

---

## H

### HA **[Intermédiaire]**
🔤 **High Availability** (Haute Disponibilité)

💡 **Définition** : Capacité d'un système à rester opérationnel malgré des pannes.

💡 **Dans MongoDB** :
- Replica Set (3+ nœuds)
- Élection automatique
- Failover automatique
- Uptime typique : 99.9%+

🎯 **Production** : Replica Set obligatoire pour la HA.

🔗 **Voir aussi** : Replica Set, Failover, Uptime

---

### HTTP/HTTPS **[Débutant]**
🔤 **HyperText Transfer Protocol / HTTP Secure**

💡 **Définition** : Protocoles de communication web.

💡 **Dans MongoDB** :
- Atlas Data API (REST sur HTTPS)
- MongoDB Connector for BI (HTTPS)
- Monitoring endpoints

⚠️ **Sécurité** : Toujours utiliser HTTPS en production.

🔗 **Voir aussi** : REST, TLS, API

---

## I

### IaaS **[Intermédiaire]**
🔤 **Infrastructure as a Service**

💡 **Définition** : Location d'infrastructure cloud (serveurs, stockage, réseau).

💡 **Exemples** : AWS EC2, Azure VMs, GCP Compute Engine.

💡 **MongoDB** : Peut être déployé sur IaaS (vs DBaaS comme Atlas).

🔗 **Voir aussi** : DBaaS, PaaS, SaaS

---

### IaC **[Intermédiaire]**
🔤 **Infrastructure as Code**

💡 **Définition** : Gestion de l'infrastructure via code déclaratif.

💡 **Outils pour MongoDB** :
- Terraform (Atlas Provider)
- Ansible
- CloudFormation
- Kubernetes Operators

📌 **Exemple** : Déployer un cluster Atlas via Terraform.

🔗 **Voir aussi** : DevOps, CI/CD, Terraform

---

### IOPS **[Avancé]**
🔤 **Input/Output Operations Per Second**

💡 **Définition** : Nombre d'opérations de lecture/écriture disque par seconde.

💡 **Impact sur MongoDB** :
- Métrique critique si données > RAM
- Index sur disque → IOPS élevés nécessaires
- Agrégations complexes → IOPS intensif

🎯 **Recommandation** : SSD NVMe pour production (50K+ IOPS).

🔗 **Voir aussi** : SSD, Performance, Working Set

---

### IoT **[Intermédiaire]**
🔤 **Internet of Things** (Internet des Objets)

💡 **Définition** : Réseau d'objets connectés collectant et échangeant des données.

💡 **MongoDB pour IoT** :
- Time Series Collections
- Haute fréquence d'insertion
- Agrégation temps réel
- Scalabilité horizontale

📌 **Cas d'usage** : Capteurs, smart cities, véhicules connectés.

🔗 **Voir aussi** : Time Series, Sharding, Capped Collections

---

## J

### JSON **[Débutant]**
🔤 **JavaScript Object Notation**

💡 **Définition** : Format d'échange de données texte, lisible par l'humain.

💡 **Relation avec BSON** : MongoDB utilise BSON (binaire) en interne, mais les APIs exposent JSON pour la simplicité.

📌 **Exemple** :
```json
{
  "name": "Alice",
  "age": 30,
  "active": true
}
```

🔗 **Voir aussi** : BSON, Document

---

## K

### K8s **[Avancé]**
🔤 **Kubernetes** (abréviation : K8s = K + 8 lettres + s)

💡 **Définition** : Plateforme d'orchestration de conteneurs.

💡 **MongoDB sur Kubernetes** :
- MongoDB Community Kubernetes Operator
- MongoDB Enterprise Kubernetes Operator
- StatefulSets pour la persistance
- Helm Charts

⚠️ **Complexité** : Architecture avancée, expertise requise.

🔗 **Voir aussi** : Docker, Operators, Helm

---

## L

### LDAP **[Avancé]**
🔤 **Lightweight Directory Access Protocol**

💡 **Définition** : Protocole d'accès à des annuaires (Active Directory, OpenLDAP).

💡 **Dans MongoDB** : Mécanisme d'authentification enterprise (authentification centralisée).

⚠️ **Disponibilité** : MongoDB Enterprise uniquement.

🔗 **Voir aussi** : Authentification, Kerberos, SCRAM

---

## M

### MVCC **[Avancé]**
🔤 **Multi-Version Concurrency Control**

💡 **Définition** : Technique permettant plusieurs versions d'un document pour la concurrence.

💡 **Dans WiredTiger** : Implémentation MVCC pour isolation des lectures et écritures.

💡 **Avantage** : Lectures non bloquantes pendant les écritures.

🔗 **Voir aussi** : WiredTiger, Isolation, Snapshot

---

## N

### NoSQL **[Débutant]**
🔤 **Not Only SQL**

💡 **Définition** : Famille de bases de données non-relationnelles.

💡 **Types** :
- **Document** : MongoDB, Couchbase
- **Clé-Valeur** : Redis, DynamoDB
- **Colonnes** : Cassandra, HBase
- **Graphe** : Neo4j, ArangoDB

💡 **Caractéristiques** : Schema-flexible, scalabilité horizontale, haute performance.

🔗 **Voir aussi** : SQL, Document, BSON

---

## O

### OLAP **[Intermédiaire]**
🔤 **Online Analytical Processing**

💡 **Définition** : Traitement analytique de gros volumes de données (lectures massives, agrégations).

💡 **MongoDB pour OLAP** :
- Aggregation Pipeline
- Atlas Data Lake
- Connector for BI

🔗 **Voir aussi** : OLTP, BI, Aggregation

---

### OLTP **[Intermédiaire]**
🔤 **Online Transaction Processing**

💡 **Définition** : Traitement transactionnel en ligne (lectures/écritures fréquentes, transactions courtes).

💡 **MongoDB pour OLTP** :
- Opérations CRUD rapides
- Index optimisés
- Transactions ACID

🔗 **Voir aussi** : OLAP, CRUD, Transaction

---

### ORM/ODM **[Intermédiaire]**
🔤 **Object-Relational Mapping / Object-Document Mapping**

💡 **Définition** : Outil/bibliothèque mappant des objets du code aux documents de la base.

💡 **ODM pour MongoDB** :
- Mongoose (Node.js)
- MongoEngine (Python)
- Spring Data MongoDB (Java)
- Doctrine MongoDB ODM (PHP)

💡 **Avantages** : Abstraction, validation, hooks, relations.

⚠️ **Inconvénient** : Couche supplémentaire, parfois moins performant.

🔗 **Voir aussi** : Driver, Mongoose

---

## P

### PaaS **[Intermédiaire]**
🔤 **Platform as a Service**

💡 **Définition** : Plateforme cloud gérant l'infrastructure + runtime.

💡 **Exemples** : Heroku, Google App Engine.

💡 **MongoDB** : Atlas App Services est un PaaS avec MongoDB intégré.

🔗 **Voir aussi** : IaaS, DBaaS, SaaS

---

## R

### RAM **[Débutant]**
🔤 **Random Access Memory**

💡 **Définition** : Mémoire vive, ressource critique pour MongoDB.

💡 **Usage dans MongoDB** :
- Cache WiredTiger (50% RAM par défaut)
- Working Set (données + index fréquents)
- Opérations de tri en mémoire

🎯 **Dimensionnement** : RAM ≥ Working Set pour performances optimales.

⚠️ **Impact** : RAM insuffisante → swap → dégradation majeure.

🔗 **Voir aussi** : Working Set, WiredTiger, Performance

---

### RBAC **[Intermédiaire]**
🔤 **Role-Based Access Control**

💡 **Définition** : Contrôle d'accès basé sur les rôles.

💡 **Dans MongoDB** : Système d'autorisation natif (utilisateurs + rôles + privilèges).

📌 **Exemple** : Rôle "readWrite" sur la base "production".

🔗 **Voir aussi** : Autorisation, Rôle, Utilisateur

---

### REST **[Intermédiaire]**
🔤 **Representational State Transfer**

💡 **Définition** : Style d'architecture pour APIs web (HTTP + JSON).

💡 **Dans MongoDB** :
- Atlas Data API (REST)
- Atlas Admin API (REST)
- Realm/App Services API (REST)

📌 **Exemple** : `GET /api/data/v1/databases/mydb/collections/users/documents`

🔗 **Voir aussi** : HTTP, JSON, API

---

### RGPD **[Avancé]**
🔤 **Règlement Général sur la Protection des Données** (GDPR en anglais)

💡 **Définition** : Régulation européenne sur la protection des données personnelles.

💡 **Conformité MongoDB** :
- Chiffrement (CSFLE, au repos, en transit)
- Audit trails
- Droit à l'oubli (suppression)
- Anonymisation

🔗 **Voir aussi** : CSFLE, Audit, Chiffrement, Conformité

---

### RTO **[Avancé]**
🔤 **Recovery Time Objective**

💡 **Définition** : Durée maximale tolérable d'interruption de service.

💡 **Dans MongoDB** :
- Replica Set → RTO ~ 30s (élection)
- Backup/Restore → RTO dépend de la taille

🔗 **Voir aussi** : RPO, Backup, Disaster Recovery

---

### RPO **[Avancé]**
🔤 **Recovery Point Objective**

💡 **Définition** : Perte de données maximale acceptable (en temps).

💡 **Dans MongoDB** :
- Write Concern "majority" → RPO ~ 0
- Snapshot quotidien → RPO ≤ 24h

🔗 **Voir aussi** : RTO, Backup, Write Concern

---

## S

### SaaS **[Intermédiaire]**
🔤 **Software as a Service**

💡 **Définition** : Logiciel fourni comme service via internet.

💡 **Exemples MongoDB** : Atlas, Atlas Charts, Atlas Search.

🔗 **Voir aussi** : DBaaS, PaaS, IaaS

---

### SCRAM **[Intermédiaire]**
🔤 **Salted Challenge Response Authentication Mechanism**

💡 **Définition** : Mécanisme d'authentification par défaut dans MongoDB.

💡 **Versions** : SCRAM-SHA-1 (déprécié), SCRAM-SHA-256 (recommandé).

💡 **Avantage** : Sécurisé, ne transmet jamais le mot de passe en clair.

🔗 **Voir aussi** : Authentification, x.509, LDAP

---

### SQL **[Débutant]**
🔤 **Structured Query Language**

💡 **Définition** : Langage de requête pour bases de données relationnelles.

💡 **Comparaison avec MongoDB** :
- SQL : tables, lignes, colonnes, schéma fixe, jointures
- MongoDB : collections, documents, champs, schéma flexible, embedding

💡 **Pont** : MongoDB Connector for BI permet d'utiliser SQL sur MongoDB.

🔗 **Voir aussi** : NoSQL, Document, Collection

---

### SSD **[Intermédiaire]**
🔤 **Solid-State Drive**

💡 **Définition** : Disque de stockage sans pièce mécanique (mémoire flash).

💡 **Avantages pour MongoDB** :
- IOPS élevés (> 50K)
- Latence faible
- Durabilité

🎯 **Recommandation** : SSD NVMe obligatoire pour production.

🔗 **Voir aussi** : IOPS, Performance, Stockage

---

### SSL **[Intermédiaire]**
🔤 **Secure Sockets Layer**

💡 **Définition** : Protocole de sécurisation des communications (déprécié, remplacé par TLS).

⚠️ **Terminologie** : Souvent utilisé à la place de TLS par abus de langage.

🔗 **Voir aussi** : TLS, Chiffrement

---

## T

### TLS **[Intermédiaire]**
🔤 **Transport Layer Security**

💡 **Définition** : Protocole de sécurisation des communications réseau (successeur de SSL).

💡 **Dans MongoDB** :
- Chiffrement des connexions client-serveur
- Chiffrement inter-nœuds (Replica Set, Sharding)
- TLS 1.2+ recommandé

🎯 **Production** : TLS obligatoire.

🔗 **Voir aussi** : SSL, x.509, Chiffrement

---

### TTL **[Intermédiaire]**
🔤 **Time-To-Live**

💡 **Définition** : Durée de vie d'une donnée avant expiration automatique.

💡 **Dans MongoDB** : Index TTL pour suppression automatique de documents anciens.

📌 **Exemple** : Logs expirés après 7 jours :
```javascript
db.logs.createIndex({createdAt: 1}, {expireAfterSeconds: 604800})
```

💡 **Cas d'usage** : Sessions, caches, logs temporaires.

🔗 **Voir aussi** : Index, Capped Collection

---

## U

### URI/URL **[Débutant]**
🔤 **Uniform Resource Identifier / Uniform Resource Locator**

💡 **Définition** : Identifiant/Adresse d'une ressource.

💡 **MongoDB Connection String** (URI) :
```
mongodb://username:password@host:port/database?options
```

📌 **Exemple Atlas** :
```
mongodb+srv://user:pass@cluster0.mongodb.net/mydb
```

🔗 **Voir aussi** : Connection String, Driver

---

### UUID **[Intermédiaire]**
🔤 **Universally Unique Identifier**

💡 **Définition** : Identifiant unique de 128 bits (format standardisé).

💡 **Dans MongoDB** :
- Alternative à ObjectId pour `_id`
- Type BSON : UUID (Binary subtype 4)
- Génération côté client ou serveur

📌 **Exemple** : `550e8400-e29b-41d4-a716-446655440000`

💡 **Cas d'usage** : Intégration avec systèmes externes, conformité standards.

🔗 **Voir aussi** : ObjectId, _id

---

## W

### WiredTiger **[Avancé]**
🔤 Moteur de stockage par défaut de MongoDB (depuis version 3.2).

💡 **Caractéristiques** :
- Compression des données (Snappy, zlib, zstd)
- Compression des index (prefix compression)
- Checkpointing automatique
- Cache interne (50% RAM par défaut)
- MVCC pour concurrence

💡 **Configuration** :
- `storage.wiredTiger.engineConfig.cacheSizeGB`
- `storage.wiredTiger.collectionConfig.blockCompressor`

🔗 **Voir aussi** : Storage Engine, MVCC, Cache, Compression

---

## Navigation rapide

- **[← Retour au sommaire du glossaire](README.md)**
- **[← A.1 - Termes MongoDB essentiels](01-termes-mongodb-essentiels.md)**

---

## Légende des symboles

| Symbole | Signification |
|---------|---------------|
| 🔤 | Définition complète |
| 💡 | Information importante / Détail |
| 📌 | Exemple concret |
| ⚠️ | Attention / Limitation |
| 🎯 | Recommandation / Bonne pratique |
| 🔗 | Voir aussi (concepts liés) |

---

## Niveaux de complexité

- **[Débutant]** : Concepts essentiels pour débuter
- **[Intermédiaire]** : Notions pour un usage professionnel
- **[Avancé]** : Concepts pour production et optimisation
- **[Expert]** : Détails techniques approfondis

---

**💡 Astuce** : Utilisez Ctrl+F (ou Cmd+F) et la table des matières pour naviguer rapidement.

**📚 Ressources** : Pour plus de détails, consultez la [documentation officielle MongoDB](https://www.mongodb.com/docs/).

⏭️ [Commandes mongosh Essentielles](/annexes/commandes-mongosh/README.md)
