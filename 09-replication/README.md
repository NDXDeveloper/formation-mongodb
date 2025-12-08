🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 9 : Réplication

## Vue d'ensemble

La réplication constitue l'un des piliers fondamentaux de l'architecture distribuée de MongoDB, permettant d'atteindre des niveaux élevés de disponibilité, de durabilité des données et de tolérance aux pannes. Ce chapitre explore en profondeur les mécanismes de réplication de MongoDB à travers le concept de **Replica Set**, une architecture sophistiquée qui garantit la redondance des données et la continuité de service même en cas de défaillance matérielle ou réseau.

## Contexte et Importance

Dans les environnements de production modernes, la disponibilité des données est critique. Une panne de serveur de base de données peut entraîner des pertes financières considérables, une dégradation de l'expérience utilisateur, voire des violations de SLA (Service Level Agreement). La réplication répond à plusieurs impératifs fondamentaux :

### Haute Disponibilité (High Availability - HA)

La réplication permet à MongoDB de continuer à servir les requêtes même lorsqu'un ou plusieurs nœuds deviennent indisponibles. Grâce à un mécanisme d'élection automatique, le système peut promouvoir un nœud secondaire en nœud primaire sans intervention humaine, minimisant ainsi le temps d'indisponibilité (downtime) à quelques secondes dans la plupart des scénarios.

### Durabilité et Redondance des Données

En maintenant plusieurs copies synchronisées des données sur différents serveurs physiques, la réplication protège contre la perte de données en cas de défaillance matérielle. Cette redondance géographique peut également être étendue à plusieurs datacenters ou régions cloud pour une protection optimale.

### Distribution Géographique

Les Replica Sets permettent de déployer des nœuds dans différentes zones géographiques, réduisant ainsi la latence pour les utilisateurs répartis mondialement tout en assurant une protection contre les défaillances régionales (catastrophes naturelles, pannes de datacenter).

### Isolation des Charges de Travail

La réplication facilite la séparation des charges de lecture et d'écriture, permettant par exemple de diriger les requêtes analytiques lourdes vers des nœuds secondaires dédiés, préservant ainsi les performances du nœud primaire pour les opérations transactionnelles critiques.

## Architecture Replica Set : Vue d'Ensemble

Un **Replica Set** MongoDB est un groupe de processus `mongod` qui maintiennent le même ensemble de données. L'architecture repose sur plusieurs principes clés :

### Topologie et Rôles

Un Replica Set typique comprend :

- **Un nœud Primary** : Le seul nœud acceptant les opérations d'écriture. Toutes les modifications de données passent obligatoirement par le Primary.
- **Plusieurs nœuds Secondary** : Répliquent de manière asynchrone les données du Primary via l'Oplog. Peuvent servir les lectures selon la configuration de Read Preference.
- **Optionnellement, des nœuds spécialisés** : Arbiters, Hidden members, Delayed members, chacun remplissant un rôle spécifique dans l'architecture.

### Mécanisme de Consensus

MongoDB utilise un protocole de consensus basé sur **Raft** (depuis la version 4.0, remplaçant le protocole initial basé sur Paxos) pour garantir la cohérence et gérer les élections. Ce protocole assure qu'à tout moment, il n'existe qu'un seul Primary dans le Replica Set, évitant ainsi les situations de "split-brain".

### Réplication Asynchrone et Oplog

La réplication dans MongoDB est **asynchrone** par défaut, ce qui signifie que les écritures sont d'abord commitées sur le Primary, puis propagées aux Secondaries. Cette propagation s'effectue via l'**Oplog** (Operations Log), une collection capped spéciale qui enregistre toutes les opérations de modification dans l'ordre chronologique.

L'asynchronisme offre de meilleures performances d'écriture mais introduit un concept important : le **replication lag**, c'est-à-dire le décalage temporel entre le Primary et les Secondaries. La compréhension et la gestion de ce lag sont essentielles pour les systèmes critiques.

## Garanties de Cohérence et Compromis CAP

Dans le cadre du théorème CAP (Consistency, Availability, Partition tolerance), MongoDB avec Replica Sets fait des choix architecturaux spécifiques :

### Modèle de Cohérence Réglable

MongoDB permet de configurer finement le niveau de cohérence via :

- **Write Concern** : Détermine le nombre de nœuds qui doivent accuser réception d'une écriture avant qu'elle soit considérée comme réussie.
- **Read Concern** : Spécifie le niveau de cohérence requis pour les lectures (local, available, majority, linearizable, snapshot).
- **Read Preference** : Définit quels membres du Replica Set peuvent servir les lectures (primary, primaryPreferred, secondary, secondaryPreferred, nearest).

### Compromis Performance vs Cohérence

- **Cohérence forte** : En utilisant `writeConcern: { w: "majority" }` et `readConcern: "majority"`, on obtient une cohérence forte au détriment de la performance.
- **Cohérence éventuelle** : Avec `writeConcern: { w: 1 }` et lecture sur les Secondaries, on privilégie la performance mais on accepte un délai de propagation.
- **Disponibilité durant les partitions** : En cas de partition réseau, le système peut temporairement devenir en lecture seule si le Primary ne peut atteindre une majorité de nœuds.

## Mécanismes Techniques Fondamentaux

### L'Oplog : Colonne Vertébrale de la Réplication

L'Oplog est une collection capped (`local.oplog.rs`) qui stocke un journal ordonné de toutes les opérations de modification. Caractéristiques clés :

- **Idempotence** : Les opérations de l'Oplog sont idempotentes, permettant leur application multiple sans effet de bord.
- **Taille configurable** : Doit être dimensionnée en fonction du volume d'écritures et de la fenêtre de maintenance souhaitée.
- **Fenêtre de réplication** : Détermine combien de temps un Secondary peut être désynchronisé avant de nécessiter une resynchronisation complète (initial sync).

### Heartbeats et Détection de Pannes

Les membres du Replica Set échangent des messages heartbeat toutes les 2 secondes par défaut. Ces heartbeats servent à :

- Détecter les nœuds défaillants ou inaccessibles
- Transmettre des informations d'état et de configuration
- Déclencher les élections en cas de perte du Primary

Le timeout par défaut (`electionTimeoutMillis`) est de 10 secondes, période après laquelle une élection peut être initiée si le Primary ne répond plus.

### Processus d'Élection

Lorsque le Primary devient indisponible ou qu'un nœud estime qu'une élection est nécessaire :

1. **Initiation** : Un nœud éligible initie une élection en incrémentant son term number
2. **Vote** : Chaque nœud vote pour au maximum un candidat par term, basé sur la priorité et la fraîcheur des données (basé sur l'optime)
3. **Majorité** : Le candidat recevant la majorité des votes devient le nouveau Primary
4. **Convergence** : Le nouveau Primary commence à accepter les écritures, les anciens Secondaries se synchronisent

Le protocole garantit qu'à tout moment, au plus un nœud peut être Primary (Single Leader).

## Considérations de Production

### Nombre de Membres et Quorum

- **Nombre impair recommandé** : Pour éviter les situations d'égalité lors des votes (3, 5, 7 membres typiquement).
- **Majorité (Quorum)** : Pour un Replica Set de N membres, la majorité est calculée comme `floor(N/2) + 1`.
- **Maximum de 50 membres** : Limite technique MongoDB, avec un maximum de 7 membres votants.

### Topologie Géographique

Pour une résilience maximale :

- **Multi-datacenter** : Distribuer les membres sur plusieurs datacenters ou availability zones
- **Distribution asymétrique** : Par exemple, 2 membres dans le datacenter principal, 1 dans le datacenter secondaire, plus éventuellement un Arbiter
- **Latence réseau** : Impact critique sur les performances de réplication et d'élection

### Monitoring et Alerting

Métriques essentielles à surveiller :

- **Replication Lag** : Écart entre le Primary et chaque Secondary
- **Oplog Window** : Temps pendant lequel l'Oplog peut absorber les écritures
- **Élections** : Fréquence et durée des élections (indicateur de stabilité)
- **Santé des membres** : État de chaque nœud (PRIMARY, SECONDARY, RECOVERING, etc.)

## Structure du Chapitre

Ce chapitre est organisé en sections progressives qui couvrent tous les aspects de la réplication MongoDB :

### 9.1 Concepts de réplication
Introduction aux principes fondamentaux de la réplication, motivations et objectifs.

### 9.2 Architecture Replica Set
Étude détaillée de l'architecture, des composants et de leur interaction.

### 9.3 Membres d'un Replica Set
Exploration approfondie des différents types de membres (Primary, Secondary, Arbiter, Hidden, Delayed) et de leurs rôles spécifiques.

### 9.4 Élection du Primary
Mécanisme de consensus, protocole Raft, critères d'éligibilité et processus de vote.

### 9.5 Oplog (Operations Log)
Structure, fonctionnement, dimensionnement et gestion de l'Oplog.

### 9.6 Configuration d'un Replica Set
Déploiement pratique, initialisation, fichiers de configuration.

### 9.7 Ajout et suppression de membres
Opérations de maintenance, scaling horizontal, procédures sécurisées.

### 9.8 Read Preference
Stratégies de lecture, distribution de charge, compromis cohérence-performance.

### 9.9 Failover et haute disponibilité
Scénarios de défaillance, temps de récupération, stratégies de résilience.

### 9.10 Monitoring d'un Replica Set
Outils, métriques, dashboards, alerting.

### 9.11 Maintenance et opérations courantes
Rolling restarts, upgrades, backup sans interruption.

### 9.12 Réplication chaînée
Topologies avancées, réplication en cascade, implications de performance.

## Prérequis pour ce Chapitre

Pour tirer pleinement parti de ce chapitre, vous devriez avoir :

- Une compréhension solide des concepts de bases de données distribuées
- Une connaissance des fondamentaux MongoDB (documents, collections, opérations CRUD)
- Une familiarité avec les concepts de cohérence, disponibilité et tolérance aux partitions (théorème CAP)
- Des notions de réseau et de systèmes distribués
- Une expérience pratique avec le déploiement et l'administration de MongoDB

## Objectifs d'Apprentissage

À l'issue de ce chapitre, vous serez capable de :

1. **Concevoir** une architecture Replica Set adaptée à vos besoins de disponibilité et performance
2. **Déployer** et configurer un Replica Set en production avec les bonnes pratiques
3. **Comprendre** les mécanismes internes de réplication, d'élection et de consensus
4. **Optimiser** les paramètres de Write Concern, Read Concern et Read Preference pour votre cas d'usage
5. **Surveiller** et maintenir un Replica Set en production
6. **Diagnostiquer** et résoudre les problèmes courants de réplication
7. **Planifier** la récupération en cas de défaillance et la continuité de service

## Cas d'Usage de la Réplication

La réplication MongoDB est particulièrement pertinente pour :

- **Applications critiques** nécessitant une disponibilité 24/7/365
- **Systèmes à fort trafic** avec des besoins de scaling en lecture
- **Applications multi-régions** avec des utilisateurs géographiquement distribués
- **Conformité réglementaire** requérant la redondance et la durabilité des données
- **Environnements de développement/test** nécessitant des copies de production
- **Analytics et reporting** isolés de la charge transactionnelle
- **Backup opérationnels** via des membres cachés ou retardés

## Notes Importantes

⚠️ **Attention** : La réplication n'est PAS une stratégie de backup à elle seule. Bien qu'elle protège contre les défaillances matérielles, elle ne protège pas contre la corruption de données, les suppressions accidentelles ou les bugs applicatifs qui se propagent à tous les membres. Des sauvegardes régulières (mongodump, snapshots) restent indispensables.

💡 **Performance** : La réplication a un coût. Chaque écriture doit être propagée à tous les Secondaries, et les garanties de cohérence forte (write concern majority) augmentent la latence d'écriture. Un dimensionnement et une configuration appropriés sont cruciaux.

🔒 **Sécurité** : Dans un Replica Set, tous les membres doivent être sécurisés de manière équivalente. La communication inter-membres doit être chiffrée (keyfile ou x.509) et authentifiée pour éviter l'ajout de membres non autorisés.

## Évolutions Récentes

Les versions récentes de MongoDB ont apporté des améliorations significatives à la réplication :

- **MongoDB 4.0** : Adoption du protocole Raft pour le consensus, transactions multi-documents
- **MongoDB 4.2** : Amélioration de la vitesse d'initial sync, streaming replication
- **MongoDB 4.4** : Réplication en miroir pour les opérations de lecture, amélioration du hedge reads
- **MongoDB 5.0** : Snapshots plus légers, réduction de la latence d'élection
- **MongoDB 6.0** : Amélioration du change stream, sharding et réplication plus intégrés
- **MongoDB 7.0** : Optimisation de l'Oplog, compression améliorée
- **MongoDB 8.0** : Réduction du replication lag, monitoring avancé

## Ressources Complémentaires

Pour approfondir vos connaissances sur la réplication MongoDB :

- **Documentation officielle** : [MongoDB Replication Documentation](https://docs.mongodb.com/manual/replication/)
- **Blog MongoDB** : Articles techniques sur les internals de la réplication
- **MongoDB University** : Cours M103 (Basic Cluster Administration)
- **Papers académiques** : "Raft Consensus Algorithm" (Diego Ongaro, John Ousterhout)
- **Conférences MongoDB** : MongoDB World, MongoDB.local sessions

---

La maîtrise de la réplication est essentielle pour tout administrateur ou architecte MongoDB travaillant sur des systèmes de production. Les sections suivantes exploreront chaque aspect en détail, combinant théorie des systèmes distribués et pratiques opérationnelles éprouvées.

**Passons maintenant à l'étude des concepts fondamentaux de la réplication.**

⏭️ [Concepts de réplication](/09-replication/01-concepts-replication.md)
