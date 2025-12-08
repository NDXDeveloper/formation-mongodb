🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.1 Concepts de Réplication

## Introduction

La réplication est un mécanisme fondamental dans les systèmes de bases de données distribuées modernes, permettant de maintenir plusieurs copies des mêmes données sur différents nœuds d'un cluster. Dans MongoDB, la réplication ne se limite pas à une simple duplication passive des données : elle constitue l'infrastructure même qui garantit la disponibilité, la durabilité et la cohérence des données dans un environnement distribué potentiellement hostile (pannes matérielles, partitions réseau, latences variables).

Cette section explore les concepts théoriques et pratiques qui sous-tendent la réplication dans MongoDB, en les situant dans le contexte plus large de la théorie des systèmes distribués.

## Définition et Objectifs de la Réplication

### Définition Formelle

La **réplication** est le processus de création et de maintenance de copies multiples (réplicas) d'un ensemble de données sur différents nœuds physiquement séparés d'un système distribué. Chaque réplica contient idéalement une copie complète et à jour des données, bien que dans la pratique, un certain degré de divergence temporaire soit inévitable en raison des contraintes du réseau et du théorème CAP.

### Objectifs Fondamentaux

La réplication dans MongoDB poursuit plusieurs objectifs interconnectés :

#### 1. Haute Disponibilité (High Availability)

La haute disponibilité désigne la capacité d'un système à rester opérationnel et accessible même en présence de défaillances. Dans le contexte de MongoDB :

- **Tolérance aux pannes matérielles** : Si un serveur hébergeant un réplica subit une défaillance matérielle (disque dur, alimentation, carte mère), les autres réplicas continuent de servir les requêtes.

- **Maintenance sans interruption** : Les opérations de maintenance (mises à jour système, patches de sécurité, upgrades MongoDB) peuvent être effectuées sur un nœud à la fois sans arrêter le service global via une technique appelée "rolling restart".

- **Failover automatique** : En cas de perte du nœud Primary, le système détecte automatiquement la défaillance et promeut un nœud Secondary en nouveau Primary, généralement en quelques secondes (typiquement 2-12 secondes selon la configuration).

- **Disponibilité géographique** : En distribuant les réplicas sur plusieurs datacenters ou régions cloud, le système peut survivre à des pannes régionales entières (catastrophes naturelles, coupures électriques massives, pannes de datacenter).

#### 2. Durabilité des Données (Data Durability)

La durabilité garantit que les données committées survivent aux pannes :

- **Redondance** : En maintenant N copies des données, la probabilité de perte totale diminue exponentiellement avec N (en supposant des défaillances indépendantes).

- **Protection contre la corruption** : Si un réplica subit une corruption de données (corruption du système de fichiers, erreurs de mémoire ECC), les autres réplicas fournissent une source fiable pour la restauration.

- **Fenêtre de récupération étendue** : L'Oplog permet de "rejouer" les opérations sur plusieurs heures ou jours, facilitant la récupération point-in-time.

#### 3. Évolutivité en Lecture (Read Scalability)

- **Distribution de charge** : Les applications peuvent distribuer les requêtes de lecture entre plusieurs réplicas, augmentant ainsi le débit total de lecture du système.

- **Isolation des charges** : Les requêtes analytiques lourdes peuvent être dirigées vers des réplicas dédiés, protégeant le Primary des impacts sur les performances transactionnelles.

- **Proximité géographique** : Les lectures peuvent être servies depuis le réplica le plus proche géographiquement de l'utilisateur, réduisant la latence.

#### 4. Isolation et Ségrégation des Charges de Travail

- **Reporting et analytics** : Exécution de requêtes complexes et coûteuses sur des Secondaries dédiés sans impacter les opérations OLTP.

- **Backup opérationnels** : Utilisation de membres Hidden pour effectuer des sauvegardes sans perturber les performances du cluster.

- **Testing et développement** : Des réplicas peuvent servir de source de données pour les environnements de test.

## Modèles de Réplication : Contexte Théorique

### Réplication Synchrone vs Asynchrone

La distinction entre réplication synchrone et asynchrone est cruciale pour comprendre les compromis de MongoDB :

#### Réplication Synchrone (Synchronous Replication)

**Principe** : Lorsqu'une opération d'écriture est soumise, le système attend que tous (ou un quorum) de réplicas aient confirmé avoir persisté l'opération avant de renvoyer un accusé de réception au client.

**Avantages** :
- Cohérence forte garantie : les lectures voient toujours les écritures précédentes
- Pas de perte de données en cas de panne du Primary (si quorum est utilisé)
- Simplicité du modèle mental pour les développeurs

**Inconvénients** :
- Latence élevée : le temps de réponse est limité par le réplica le plus lent
- Disponibilité réduite : si un réplica est inaccessible, les écritures peuvent être bloquées
- Throughput limité par la capacité des réplicas les plus lents

**Exemples de systèmes** : Google Spanner (avec TrueTime), certaines configurations de MySQL avec réplication semi-synchrone.

#### Réplication Asynchrone (Asynchronous Replication)

**Principe** : Le Primary accuse réception de l'écriture dès qu'elle est persistée localement, puis propage l'opération aux Secondaries de manière asynchrone, sans attendre leur confirmation.

**Avantages** :
- Latence minimale : le client n'attend que la persistance locale
- Haute performance : le débit d'écriture n'est pas limité par les réplicas
- Tolérance aux ralentissements : un réplica lent n'impacte pas les performances globales

**Inconvénients** :
- Fenêtre de perte potentielle : les écritures récentes peuvent être perdues si le Primary tombe avant propagation
- Cohérence éventuelle : les lectures sur Secondaries peuvent voir des données obsolètes
- Complexité accrue pour l'application (gestion du replication lag)

**Implémentation MongoDB** : Par défaut, MongoDB utilise la réplication asynchrone, mais offre une granularité fine via les Write Concerns.

#### Modèle Hybride de MongoDB : Write Concern

MongoDB transcende la dichotomie synchrone/asynchrone via le concept de **Write Concern**, permettant de spécifier par opération :

```javascript
db.collection.insertOne(
  { document },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
```

Paramètres :
- **w (write acknowledgement)** :
  - `w: 1` (défaut) - Asynchrone pur, ACK dès écriture sur le Primary
  - `w: "majority"` - Semi-synchrone, ACK après réplication sur la majorité
  - `w: <nombre>` - ACK après réplication sur N membres
  - `w: 0` - Fire-and-forget, pas d'ACK (déconseillé)

- **j (journal)** :
  - `j: true` - Garantit la persistance sur disque via le journal WiredTiger
  - `j: false` - ACK dès écriture en mémoire

- **wtimeout** : Timeout en millisecondes, évite le blocage indéfini

Ce modèle permet d'ajuster le compromis latence/durabilité au niveau de chaque opération.

### Single-Leader vs Multi-Leader vs Leaderless

#### Single-Leader (Master-Slave / Primary-Secondary)

**Architecture MongoDB** : C'est le modèle adopté par MongoDB avec les Replica Sets.

**Caractéristiques** :
- Un seul nœud (Primary) accepte les écritures
- Les Secondaries répliquent depuis le Primary
- Les lectures peuvent être distribuées (selon Read Preference)

**Avantages** :
- Pas de conflits d'écriture (une seule source de vérité)
- Modèle de cohérence plus simple
- Implémentation relativement directe

**Inconvénients** :
- Point de contention unique pour les écritures
- Dépendance à la disponibilité du Primary
- Latence pour les écritures distantes géographiquement du Primary

**Gestion des pannes** : Élection automatique d'un nouveau Primary selon un protocole de consensus (Raft dans MongoDB).

#### Multi-Leader (Multi-Master)

**Principe** : Plusieurs nœuds acceptent simultanément les écritures, avec réplication bidirectionnelle.

**Utilisé dans** : CouchDB, Cassandra (dans une certaine mesure), systèmes de réplication multi-régions.

**Challenges** :
- Détection et résolution de conflits complexe
- Cohérence éventuelle garantie uniquement
- Risque de divergence en cas de partition prolongée

**MongoDB et Multi-Leader** : MongoDB ne supporte pas nativement le multi-leader au niveau Replica Set, mais le sharding peut être vu comme une forme de multi-leader partitionné (chaque shard ayant son propre Primary).

#### Leaderless (Quorum-Based)

**Principe** : Pas de leader désigné, les lectures et écritures nécessitent un quorum de nœuds.

**Utilisé dans** : Dynamo, Cassandra, Riak.

**Caractéristiques** :
- W (write quorum) + R (read quorum) > N (total nœuds) garantit la cohérence
- Haute disponibilité en écriture même avec pannes
- Détection de conflits via vector clocks ou last-write-wins

**MongoDB et Leaderless** : MongoDB n'utilise pas ce modèle, préférant le single-leader pour simplifier la cohérence.

## Concepts Clés de la Réplication MongoDB

### Le Replica Set : Unité Fondamentale

Un **Replica Set** est un groupe de processus `mongod` qui maintiennent le même dataset. C'est l'abstraction de base de la réplication dans MongoDB.

**Composition typique** :
- 1 Primary : nœud leader acceptant les écritures
- N Secondaries : nœuds followers répliquant depuis le Primary
- Optionnel : Arbiters, Hidden members, Delayed members

**Invariant fondamental** : À tout moment, il existe au plus un Primary dans un Replica Set (single-leader invariant).

### L'Oplog : Journal de Réplication

L'**Oplog** (operations log) est une collection capped spéciale (`local.oplog.rs`) qui enregistre toutes les opérations de modification dans l'ordre chronologique.

**Propriétés critiques** :

1. **Idempotence** : Chaque opération de l'Oplog peut être appliquée plusieurs fois avec le même résultat. Par exemple :
   - `{ $set: { status: "active" } }` est idempotent
   - `{ $inc: { counter: 1 } }` est transformé en `{ $set: { counter: <valeur_finale> } }`

2. **Ordre total** : Les opérations sont strictement ordonnées par timestamp (optime), garantissant une réplication dans le même ordre.

3. **Compacité** : L'Oplog stocke l'état final, pas les étapes intermédiaires. Plusieurs mises à jour d'un même document sont consolidées.

4. **Circularité** : Étant une collection capped, l'Oplog réutilise l'espace en écrasant les entrées les plus anciennes.

**Optime** : Chaque opération est marquée par un optime, un tuple `(term, timestamp)` où :
- `term` : Numéro de mandat du Primary (incrémenté à chaque élection)
- `timestamp` : Horodatage BSON (8 octets : 4 pour epoch secondes, 4 pour ordinal)

### Protocole de Consensus : Raft

Depuis MongoDB 4.0, le protocole d'élection et de consensus est basé sur **Raft**, remplaçant l'ancien protocole propriétaire inspiré de Paxos.

**Principes Raft** :

1. **Leader Election** :
   - Les nœuds peuvent être dans trois états : Leader, Follower, Candidate
   - Si un Follower ne reçoit pas de heartbeat du Leader pendant `electionTimeout`, il devient Candidate
   - Le Candidate sollicite les votes des autres nœuds
   - Un Candidate devient Leader s'il obtient la majorité des votes

2. **Log Replication** :
   - Le Leader accepte les requêtes clients et les ajoute à son log
   - Le Leader propage les entrées aux Followers via des AppendEntries RPC
   - Une entrée est committée quand répliquée sur la majorité

3. **Safety** :
   - **Election Safety** : Au plus un Leader par term
   - **Leader Append-Only** : Le Leader n'écrase jamais son log
   - **Log Matching** : Si deux logs contiennent une entrée avec même index et term, tous les entrées précédentes sont identiques
   - **Leader Completeness** : Si une entrée est committée dans un term, elle sera présente dans tous les Leaders futurs
   - **State Machine Safety** : Si un nœud applique une entrée à un index, aucun autre nœud n'appliquera une entrée différente pour cet index

**Adaptation MongoDB** :
- Utilisation de la priorité pour influencer les élections
- Concept de "votes" (membres votants vs non-votants)
- Intégration avec Write Concern "majority"

### Heartbeats et Détection de Défaillances

Les membres d'un Replica Set échangent périodiquement des messages **heartbeat** pour :

1. **Surveiller la santé** : Détecter les nœuds inaccessibles ou défaillants
2. **Partager l'état** : Propager les informations de configuration et d'état
3. **Synchroniser la topologie** : Maintenir une vue cohérente du cluster

**Paramètres clés** :
- `heartbeatIntervalMillis` : 2000 ms par défaut (fréquence des heartbeats)
- `electionTimeoutMillis` : 10000 ms par défaut (timeout avant élection)
- `heartbeatTimeoutSecs` : 10 s par défaut (timeout de réponse heartbeat)

**Mécanisme de détection** :
- Si un membre ne reçoit pas de heartbeat du Primary pendant `electionTimeoutMillis`, il peut initier une élection
- Si le Primary ne peut communiquer avec une majorité de membres, il se dégrade automatiquement en Secondary (step down)

### Replication Lag : Inévitabilité et Gestion

Le **replication lag** (décalage de réplication) est le délai entre l'application d'une opération sur le Primary et sa propagation sur un Secondary.

**Causes du lag** :

1. **Latence réseau** : Délai physique de transmission des données
2. **Charge du Secondary** : Saturation CPU/disque du Secondary
3. **Volume d'écritures** : Débordement si le taux d'écriture dépasse la capacité de réplication
4. **Opérations lourdes** : Indexes builds, large transactions
5. **Configuration sous-optimale** : Oplog trop petit, insuffisance de ressources

**Mesure du lag** :
```javascript
rs.status()
```
Champ `optimeDate` pour chaque membre : différence entre Primary et Secondary.

**Implications** :

- **Lectures obsolètes** : Les lectures sur Secondaries peuvent voir des données périmées
- **Failover retardé** : Un Secondary très en retard peut retarder l'élection
- **Fenêtre de perte** : Lag important augmente le risque de perte en cas de panne du Primary

**Stratégies d'atténuation** :

1. **Write Concern "majority"** : Attend la réplication sur la majorité avant ACK
2. **Read Concern "majority"** : Lit uniquement les données répliquées sur la majorité
3. **Monitoring** : Alertes sur lag > seuil acceptable (ex: 10 secondes)
4. **Dimensionnement** : Oplog suffisamment large (couvre plusieurs heures d'écritures)
5. **Hardware** : Secondaries avec ressources équivalentes au Primary
6. **Network** : Bande passante et latence adéquates entre nœuds

### Cohérence et Isolation : Read et Write Concerns

MongoDB offre un contrôle granulaire sur les garanties de cohérence via Read et Write Concerns.

#### Write Concern

Contrôle le niveau d'accusé de réception requis pour les écritures :

**w: 1** (défaut avant MongoDB 5.0)
- ACK dès écriture sur le Primary (en mémoire ou journal selon `j`)
- Minimum de latence
- Risque de perte si Primary tombe avant réplication

**w: "majority"** (défaut depuis MongoDB 5.0)
- ACK après réplication sur la majorité des membres votants
- Durabilité garantie : survit à la panne d'une minorité
- Latence accrue (attente de la réplication)

**w: \<nombre\>**
- ACK après réplication sur N membres spécifiques
- Utile pour garantir la réplication sur des membres géographiquement distribués

**w: 0**
- Fire-and-forget, pas d'ACK
- Performance maximale mais aucune garantie
- Déconseillé sauf pour données non critiques

**j: true/false**
- Contrôle la persistance sur disque via le journal WiredTiger
- `j: true` garantit la durabilité même en cas de crash brutal
- `j: false` accepte l'écriture en mémoire (cache WiredTiger)

#### Read Concern

Contrôle le niveau de cohérence requis pour les lectures :

**local** (défaut)
- Lit les données locales du nœud, sans garantie de durabilité
- Peut lire des données non répliquées (susceptibles d'être rollback)
- Latence minimale

**available**
- Similaire à `local` mais optimisé pour les clusters shardés
- Peut retourner des documents orphelins (après migration de chunk)

**majority**
- Lit uniquement les données répliquées sur la majorité
- Garantit qu'une donnée lue ne sera pas rollback
- Combiné avec `writeConcern: majority`, fournit une cohérence causale

**linearizable**
- Cohérence la plus forte : équivalent à une lecture depuis un registre unique
- Garantit que la lecture voit toutes les écritures précédentes (global order)
- Latence élevée, utilisé pour les cas critiques (ex: leader election)
- Disponible uniquement pour les lectures sur Primary avec `maxTimeMS`

**snapshot**
- Lecture depuis un snapshot cohérent (point-in-time)
- Utilisé dans les transactions multi-documents
- Garantit l'isolation entre transactions (snapshot isolation)

#### Combinaisons Typiques

**Cohérence éventuelle** (performance maximale) :
```javascript
{ writeConcern: { w: 1 }, readConcern: { level: "local" } }
```

**Cohérence forte** (durabilité maximale) :
```javascript
{ writeConcern: { w: "majority", j: true }, readConcern: { level: "majority" } }
```

**Cohérence linéarisable** (cas critiques) :
```javascript
{ writeConcern: { w: "majority", j: true }, readConcern: { level: "linearizable" } }
```

### Read Preference : Routage des Lectures

La **Read Preference** détermine quels membres du Replica Set peuvent servir les lectures.

**Modes** :

1. **primary** (défaut)
   - Toutes les lectures depuis le Primary
   - Cohérence maximale (voit toutes les écritures)
   - Charge concentrée sur le Primary

2. **primaryPreferred**
   - Primary si disponible, sinon Secondary
   - Fallback automatique lors de maintenance/panne
   - Utile pour applications tolérantes au lag

3. **secondary**
   - Lectures exclusivement depuis les Secondaries
   - Décharge le Primary
   - Risque de lectures obsolètes

4. **secondaryPreferred**
   - Secondaries si disponibles, sinon Primary
   - Équilibre charge/disponibilité

5. **nearest**
   - Membre avec latence réseau la plus faible
   - Optimal pour réduire la latence globale
   - Peut être Primary ou Secondary

**Tag Sets** : Permettent de cibler des membres spécifiques selon des tags (datacenter, région, type de hardware).

## Théorème CAP et Positionnement de MongoDB

Le **théorème CAP** (Consistency, Availability, Partition tolerance), énoncé par Eric Brewer, stipule qu'un système distribué ne peut garantir simultanément :

- **C (Consistency)** : Tous les nœuds voient les mêmes données au même moment
- **A (Availability)** : Chaque requête reçoit une réponse (succès ou échec)
- **P (Partition tolerance)** : Le système continue de fonctionner malgré les partitions réseau

En pratique, **P est obligatoire** (les partitions réseau sont inévitables), donc le choix se résume à CP vs AP.

### MongoDB : CP ou AP ?

**MongoDB est configurable** sur le spectre CP-AP :

**Configuration CP** (Consistency prioritaire) :
```javascript
writeConcern: { w: "majority", j: true }
readConcern: { level: "majority" }
readPreference: "primary"
```
- Durant une partition, si le Primary ne peut atteindre la majorité, il se dégrade → indisponibilité temporaire en écriture
- Les lectures voient toujours des données cohérentes
- Privilégie la cohérence sur la disponibilité

**Configuration AP** (Availability prioritaire) :
```javascript
writeConcern: { w: 1 }
readConcern: { level: "local" }
readPreference: "secondaryPreferred"
```
- Les écritures réussissent tant que le Primary est accessible (même isolé)
- Les lectures peuvent être servies par n'importe quel membre accessible
- Privilégie la disponibilité sur la cohérence (cohérence éventuelle)

**En pratique**, MongoDB offre un continuum configurable, permettant d'ajuster les compromis selon les besoins de chaque opération.

## Comparaison avec d'Autres Systèmes

### MongoDB vs MySQL Replication

**MySQL (réplication asynchrone classique)** :
- Single-leader (master-slave)
- Réplication basée sur binlog (similaire à l'Oplog)
- Pas de failover automatique natif (nécessite ProxySQL, Orchestrator, etc.)
- Cohérence éventuelle sans contrôle granulaire

**MongoDB** :
- Single-leader avec élection automatique (Raft)
- Oplog idempotent et ordonné
- Failover automatique en secondes
- Write/Read Concerns pour contrôle granulaire de la cohérence

### MongoDB vs Cassandra

**Cassandra** :
- Leaderless (quorum-based)
- Haute disponibilité en écriture (multi-master)
- Cohérence éventuelle par défaut (tunable avec quorum)
- Résolution de conflits via timestamps (last-write-wins)

**MongoDB** :
- Single-leader par Replica Set
- Pas de conflits d'écriture (un seul Primary)
- Cohérence configurable (local à linearizable)
- Modèle plus simple pour les développeurs

### MongoDB vs PostgreSQL (réplication synchrone)

**PostgreSQL** :
- Single-leader avec réplication streaming
- Support réplication synchrone (similaire à w: "majority")
- Failover via outils externes (Patroni, repmgr)
- Performances élevées mais moins de flexibilité géographique

**MongoDB** :
- Réplication asynchrone par défaut, configurable
- Failover automatique intégré
- Architecture pensée pour la distribution géographique
- Trade-off performance/cohérence plus fin

## Limitations et Considérations

### Limitations Théoriques

1. **Overhead de la réplication** : Chaque écriture doit être propagée, consommant bande passante et CPU
2. **Replication lag inévitable** : La réplication asynchrone implique toujours un décalage temporel
3. **Complexité de consensus** : Le protocole Raft, bien que robuste, ajoute de la latence et de la complexité
4. **Borne supérieure d'élection** : Le temps de failover est borné par `electionTimeoutMillis` (généralement 10s)

### Limitations Pratiques MongoDB

1. **Maximum 50 membres** par Replica Set (dont 7 votants maximum)
2. **Coût de w: "majority"** : Latence accrue, particulièrement sur WAN
3. **Oplog continu** : Nécessite un dimensionnement soigné pour éviter les resync complets
4. **Pas de multi-leader natif** : Les écritures sont toujours single-point (jusqu'au sharding)

### Considérations Opérationnelles

1. **Topologie réseau** : La latence inter-nœuds impacte directement les performances
2. **Dimensionnement matériel** : Les Secondaries doivent avoir des ressources similaires au Primary
3. **Oplog sizing** : Calculer la taille en fonction du taux d'écriture et de la fenêtre de maintenance
4. **Monitoring** : Surveillance continue du lag, de la santé des membres, des élections

## Conclusion

La réplication dans MongoDB repose sur des fondements théoriques solides issus de la recherche en systèmes distribués (consensus Raft, théorème CAP, modèles de cohérence), tout en offrant une flexibilité pratique via les Write Concerns, Read Concerns et Read Preferences.

**Points clés à retenir** :

1. MongoDB utilise un modèle **single-leader** (Primary-Secondary) avec élection automatique via Raft
2. La réplication est **asynchrone par défaut**, mais configurable finement via Write Concern
3. Le **théorème CAP** s'applique : MongoDB permet de choisir sur le spectre CP-AP selon les besoins
4. L'**Oplog** est l'épine dorsale de la réplication, garantissant l'ordre et l'idempotence
5. Le **replication lag** est une réalité inévitable, nécessitant monitoring et gestion proactive
6. Les **Read et Write Concerns** offrent un contrôle granulaire sur les compromis cohérence/performance/disponibilité

La maîtrise de ces concepts est essentielle pour concevoir, déployer et opérer des systèmes MongoDB en production, en particulier pour les applications critiques nécessitant haute disponibilité et durabilité des données.

Dans les sections suivantes, nous approfondirons l'architecture concrète des Replica Sets, les mécanismes d'élection, la gestion de l'Oplog, et les aspects opérationnels de la réplication.

⏭️ [Architecture Replica Set](/09-replication/02-architecture-replica-set.md)
