🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Partie 4 : Architecture Distribuée (Avancé)

## 🎯 Architecture pour la haute disponibilité et la scalabilité

Vous maîtrisez maintenant la modélisation, l'optimisation et les garanties transactionnelles de MongoDB. Mais une question cruciale reste en suspens : **comment concevoir une architecture MongoDB qui reste disponible 24/7 malgré les pannes matérielles, et qui peut gérer des milliards de documents avec des performances constantes ?**

La Partie 4 est dédiée à l'**architecture distribuée de MongoDB**, le fondement qui permet aux applications modernes de scaler globalement et de tolérer les pannes. C'est ici que MongoDB révèle sa vraie puissance en tant que système distribué de classe mondiale.

## 🌐 Le défi du scale : Vertical vs Horizontal

### Les limites du scale vertical

**Scale vertical** (scale-up) : Ajouter plus de ressources à un seul serveur
- ➕ Simple à gérer (un seul nœud)
- ➕ Pas de complexité de distribution
- ➖ **Limites physiques** : On ne peut pas augmenter indéfiniment CPU/RAM/Disque
- ➖ **Single Point of Failure** : Si le serveur tombe, tout tombe
- ➖ **Coût exponentiel** : Les serveurs très puissants sont disproportionnellement chers
- ➖ **Downtime pour upgrade** : Mise à jour = interruption de service

**Réalité** : Le scale vertical atteint rapidement ses limites physiques et économiques.

### Le scale horizontal comme solution

**Scale horizontal** (scale-out) : Ajouter plus de serveurs (nœuds)
- ➕ **Pas de limite théorique** : Ajoutez autant de nœuds que nécessaire
- ➕ **Tolérance aux pannes** : La perte d'un nœud n'affecte pas le système
- ➕ **Coût linéaire** : Utilisation de commodity hardware
- ➕ **Upgrade sans downtime** : Rolling upgrades nœud par nœud
- ➖ **Complexité** : Distribution des données, cohérence, coordination
- ➖ **Latence réseau** : Communication entre nœuds

**MongoDB excelle dans le scale horizontal** grâce à deux mécanismes complémentaires :
1. **Réplication** : Pour la haute disponibilité (HA)
2. **Sharding** : Pour la scalabilité des données et du throughput

## 🏗️ Les deux piliers de l'architecture distribuée

### Réplication : La haute disponibilité

> **Objectif** : Garantir que votre système reste opérationnel même en cas de panne matérielle ou réseau.

**Concept** : Maintenir plusieurs copies identiques de vos données sur différents serveurs (Replica Set).

```
┌──────────────────────────────────────────────────┐
│           Replica Set (3 membres)                │
├──────────────────────────────────────────────────┤
│                                                  │
│   ┌─────────┐       ┌─────────┐       ┌─────────┐│
│   │ PRIMARY │─────▶ │SECONDARY│─────▶ │SECONDARY││
│   │ (write) │       │ (read?) │       │ (read?) ││
│   └─────────┘       └─────────┘       └─────────┘│
│        │                                         │
│        └──▶ Oplog (operations log)               │
│                                                  │
│   Si le PRIMARY tombe → Election automatique     │
│   Un SECONDARY devient PRIMARY en ~10 secondes   │
└──────────────────────────────────────────────────┘
```

**Bénéfices** :
- ✅ **Tolérance aux pannes** : Perte d'un ou deux nœuds sans interruption
- ✅ **Zero-downtime maintenance** : Rolling restart pour les upgrades
- ✅ **Durabilité des données** : Plusieurs copies sur différents serveurs
- ✅ **Disaster recovery** : Nœuds dans différents datacenters
- ✅ **Read scaling** : Possibilité de lire depuis les secondaries

**Cas d'usage typiques** :
- Applications critiques nécessitant 99.99%+ uptime
- Systèmes nécessitant des backups automatiques
- Applications multi-régionales

---

### Sharding : La scalabilité horizontale

> **Objectif** : Distribuer vos données sur plusieurs serveurs pour dépasser les limites d'un seul serveur.

**Concept** : Partitionner horizontalement vos données sur plusieurs shards (serveurs).

```
┌────────────────────────────────────────────────────────────┐
│                 Sharded Cluster                            │
├────────────────────────────────────────────────────────────┤
│                                                            │
│               ┌─────────┐                                  │
│         ┌───▶ │ Shard 1 │───┐  (users: id 0-333)           │
│         │     └─────────┘   │                              │
│    ┌────────┐               │                              │
│    │ mongos │─────────────▶ │  (users: id 334-666)         │
│    │(router)│               │  ┌─────────┐                 │
│    └────────┘         ┌───▶ │  │ Shard 2 │                 │
│         │             │     │  └─────────┘                 │
│         │             │     │                              │
│         └─────────────┼───▶ │  (users: id 667-999)         │
│                       │     │  ┌─────────┐                 │
│                       └───▶ │  │ Shard 3 │                 │
│                             │  └─────────┘                 │
│                             │                              │
│  Config Servers: Métadonnées sur la distribution           │
└────────────────────────────────────────────────────────────┘
```

**Bénéfices** :
- ✅ **Capacité de stockage illimitée** : Ajoutez des shards pour plus d'espace
- ✅ **Throughput horizontal** : Plus de shards = plus de lectures/écritures parallèles
- ✅ **Isolation géographique** : Shards dans différentes régions
- ✅ **Évolutivité progressive** : Commencez petit, shardez quand nécessaire

**Cas d'usage typiques** :
- Datasets > 1 TB
- Applications avec millions d'utilisateurs
- IoT avec millions de capteurs
- SaaS multi-tenant avec isolation des données

---

### Combinaison : Replica Sets + Sharding

**En production**, chaque shard est lui-même un Replica Set :

```
┌──────────────────────────────────────────────────────────────┐
│        Production Sharded Cluster avec HA                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Shard 1 Replica Set                                    │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │  │
│  │  │ PRIMARY │  │SECONDARY│  │SECONDARY│                 │  │
│  │  └─────────┘  └─────────┘  └─────────┘                 │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Shard 2 Replica Set                                    │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │  │
│  │  │ PRIMARY │  │SECONDARY│  │SECONDARY│                 │  │
│  │  └─────────┘  └─────────┘  └─────────┘                 │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Config Servers Replica Set                             │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │  │
│  │  └─────────┘  └─────────┘  └─────────┘                 │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Mongos (Query Routers) - Stateless, peuvent être multiples  │
└──────────────────────────────────────────────────────────────┘
```

**Résultat** : Haute disponibilité **ET** scalabilité illimitée.

## 🎯 Haute disponibilité vs Scalabilité : Deux problèmes différents

Il est crucial de comprendre que la réplication et le sharding résolvent **des problèmes différents** :

| Dimension | Réplication (Replica Sets) | Sharding |
|-----------|----------------------------|----------|
| **Objectif principal** | Haute disponibilité (HA) | Scalabilité horizontale |
| **Problème résolu** | Tolérance aux pannes | Limites de capacité/throughput |
| **Copies des données** | Identiques sur tous les nœuds | Partitionnées entre shards |
| **Nombre de datasets** | 1 dataset répliqué N fois | 1 dataset distribué sur N shards |
| **Lecture** | Peut être distribuée (avec compromis) | Automatiquement distribuée |
| **Écriture** | Centralisée sur PRIMARY | Distribuée entre shards |
| **Capacité totale** | Limitée par un nœud | N × capacité d'un shard |
| **Complexité** | Faible à modérée | Élevée |
| **Quand l'utiliser** | Toujours (prod) | Quand un serveur ne suffit plus |

**Principe architectural** :
- Utilisez **toujours** un Replica Set en production (même sans sharding)
- N'ajoutez le sharding que **quand nécessaire** (> 1 TB ou throughput insuffisant)

## 📋 Prérequis

Cette partie s'adresse à des **architectes et ingénieurs système** ayant :

### Connaissances MongoDB requises
- ✅ **Maîtrise complète des Parties 1-3**
- ✅ Modélisation avancée et optimisation
- ✅ Compréhension des transactions et de la cohérence
- ✅ Expérience avec MongoDB en environnement de développement

### Connaissances en systèmes distribués
- ✅ **Théorème CAP** et ses implications pratiques
- ✅ **Consensus distribué** : Concepts de base (Raft, Paxos)
- ✅ **Partitionnement** : Stratégies de distribution de données
- ✅ **Cohérence éventuelle** vs cohérence forte
- ✅ **Network partitions** et split-brain scenarios
- ✅ **Quorum** et majorité

### Compétences opérationnelles
- 🛠️ Administration Linux/Unix (fichiers, processus, réseau)
- 🛠️ Réseau : TCP/IP, DNS, firewalls, load balancing
- 🛠️ Monitoring et observabilité
- 🛠️ Scripting (Bash, Python) pour l'automatisation
- 🛠️ Expérience avec des environnements de production

### État d'esprit
- 🧠 Pensée architecturale (trade-offs, patterns)
- 🧠 Approche pragmatique (pas de sur-ingénierie)
- 🧠 Culture DevOps (automatisation, monitoring)
- 🧠 Gestion de la complexité

**Si vous ne maîtrisez pas ces prérequis**, prenez le temps de les acquérir. L'architecture distribuée est complexe et les erreurs ont un impact direct sur la disponibilité et les performances en production.

## 🎓 Objectifs d'apprentissage

À la fin de cette partie, vous serez capable de :

### Compétences en réplication
- ✅ **Comprendre** l'architecture Replica Set en profondeur
- ✅ **Déployer** et configurer un Replica Set de production
- ✅ **Gérer** les différents types de membres (Primary, Secondary, Arbiter, Hidden, Delayed)
- ✅ **Comprendre** le mécanisme d'élection et le processus de failover
- ✅ **Maîtriser** l'oplog et la réplication des opérations
- ✅ **Configurer** Read Preference pour optimiser les lectures
- ✅ **Assurer** une haute disponibilité avec des SLAs de 99.99%+
- ✅ **Effectuer** une maintenance sans interruption de service
- ✅ **Monitorer** et diagnostiquer les problèmes de réplication
- ✅ **Gérer** le replication lag et les situations de split-brain

### Compétences en sharding
- ✅ **Comprendre** l'architecture d'un cluster shardé
- ✅ **Choisir** la shard key appropriée (critique pour la performance)
- ✅ **Déployer** un cluster shardé complet
- ✅ **Comprendre** les différents types de sharding (Range, Hashed, Zone)
- ✅ **Gérer** les chunks et le balancing automatique
- ✅ **Optimiser** les requêtes pour le sharding (targeted vs broadcast)
- ✅ **Résoudre** les problèmes de jumbo chunks
- ✅ **Monitorer** et maintenir un cluster shardé
- ✅ **Scaler** horizontalement en ajoutant des shards
- ✅ **Comprendre** l'impact du sharding sur les transactions

### Compétences architecturales
- ✅ **Concevoir** une architecture MongoDB pour la haute disponibilité
- ✅ **Dimensionner** les ressources nécessaires (CPU, RAM, disque, réseau)
- ✅ **Choisir** entre un Replica Set simple ou un cluster shardé
- ✅ **Planifier** la croissance et la scalabilité
- ✅ **Évaluer** les compromis entre performance, coût et disponibilité
- ✅ **Concevoir** des architectures multi-régionales
- ✅ **Implémenter** des stratégies de disaster recovery

### Compétences opérationnelles
- ✅ **Automatiser** le déploiement avec IaC (Infrastructure as Code)
- ✅ **Monitorer** les métriques critiques de réplication et sharding
- ✅ **Diagnostiquer** et résoudre les problèmes de production
- ✅ **Effectuer** des rolling upgrades sans downtime
- ✅ **Gérer** les incidents (failover, panne de shard, etc.)
- ✅ **Optimiser** les performances d'un cluster distribué

## 📚 Vue d'ensemble des modules

Cette partie contient **2 modules complémentaires** qui forment l'architecture distribuée de MongoDB :

### Module 9 : Réplication
**Durée estimée : 16-20 heures**

Le fondement de toute architecture MongoDB en production. La réplication garantit que votre système reste disponible malgré les pannes.

#### 9.1 Concepts de réplication
**Durée : 2 heures**

Les principes fondamentaux de la réplication dans MongoDB.

**Ce que vous maîtriserez :**
- Pourquoi la réplication est essentielle
- Architecture logique et physique
- Terminologie (Primary, Secondary, Arbiter, etc.)
- Différence avec d'autres systèmes de réplication

**Contexte** : Comprendre les concepts vous permet de prendre les bonnes décisions architecturales.

---

#### 9.2 Architecture Replica Set
**Durée : 2-3 heures**

Structure complète d'un Replica Set et ses composants.

**Ce que vous maîtriserez :**
- Topologie recommandée (3, 5, 7 membres)
- Rôle de chaque membre
- Configuration réseau et communication
- Quorum et majorité

**Principe clé** : Un Replica Set nécessite une majorité (N/2 + 1) pour élire un Primary. D'où l'importance d'un nombre impair de membres.

**Topologies courantes :**
```
3 membres : 1 Primary + 2 Secondary (tolère 1 panne)
5 membres : 1 Primary + 4 Secondary (tolère 2 pannes)
7 membres : 1 Primary + 6 Secondary (tolère 3 pannes)
```

---

#### 9.3 Membres d'un Replica Set
**Durée : 3-4 heures**

Détail de chaque type de membre et leurs cas d'usage.

**Types de membres :**

**Primary** : Le seul membre acceptant les écritures
- Point unique d'écriture
- Peut servir les lectures (par défaut)
- Source de vérité pour l'oplog

**Secondary** : Réplique le Primary
- Réplication asynchrone depuis l'oplog
- Peut servir les lectures (avec Read Preference)
- Candidat pour l'élection

**Arbiter** : Membre léger pour le quorum
- Ne stocke pas de données
- Participe uniquement aux élections
- Utilisé pour avoir un nombre impair de votants (controversé)

**Hidden** : Secondary caché
- Ne peut pas devenir Primary
- Invisible aux clients
- Utilisé pour les backups ou analytics

**Delayed** : Secondary avec retard temporel
- Réplique avec un délai intentionnel (ex: 1 heure)
- Protection contre les erreurs humaines (DELETE accidentel)
- Ne peut pas devenir Primary

**Cas d'usage pour chaque type :**
```
Standard: Primary + 2 Secondary
Budget: Primary + Secondary + Arbiter (non recommandé en prod)
Analytics: Primary + 2 Secondary + 1 Hidden
Protection: Primary + 2 Secondary + 1 Delayed
```

---

#### 9.4 Élection du Primary
**Durée : 2-3 heures**

Le mécanisme critique qui assure la haute disponibilité.

**Ce que vous maîtriserez :**
- Algorithme d'élection (basé sur Raft)
- Priorités et votes
- Durée d'élection (typiquement 10-12 secondes)
- Situations de split-brain et leur prévention
- Impact sur les applications pendant l'élection

**Scénario typique :**
```
t=0s : Primary tombe en panne
t=10s : Heartbeat timeout détecté
t=12s : Élection lancée
t=20s : Nouveau Primary élu
t=20s : Applications peuvent écrire à nouveau
```

**Impact :** 10-20 secondes d'indisponibilité en écriture. Les lectures depuis Secondary peuvent continuer.

---

#### 9.5 Oplog (Operations Log)
**Durée : 2-3 heures**

Le journal des opérations qui rend la réplication possible.

**Ce que vous maîtriserez :**
- Structure de l'oplog (capped collection)
- Taille de l'oplog et dimensionnement
- Idempotence des opérations
- Replication lag et monitoring
- Compaction et truncation

**Principe fondamental** : L'oplog doit être assez grand pour contenir plusieurs heures (voire jours) d'opérations, permettant aux Secondary de "rattraper" après une panne.

**Dimensionnement typique :**
```
Faible activité : 5-10 GB (plusieurs jours)
Activité moyenne : 20-50 GB (1-2 jours)
Haute activité : 100-200 GB (12-24 heures)
```

---

#### 9.6-9.7 Configuration et gestion
**Durée : 3-4 heures**

Déploiement pratique et opérations courantes.

**Ce que vous maîtriserez :**
- Initialisation d'un Replica Set
- Ajout et suppression de membres
- Modification de la configuration
- Reconfiguration sans downtime
- Validation et tests

---

#### 9.8 Read Preference
**Durée : 2 heures**

Contrôle d'où les lectures sont effectuées.

**Options :**
- `primary` (défaut) : Toutes les lectures depuis le Primary
- `primaryPreferred` : Primary si disponible, sinon Secondary
- `secondary` : Lectures uniquement depuis Secondary
- `secondaryPreferred` : Secondary si disponible, sinon Primary
- `nearest` : Le membre le plus proche (latence)

**Compromis :**
```
primary : Cohérence forte, charge centralisée
secondary : Distribution de charge, cohérence éventuelle
nearest : Latence minimale, cohérence éventuelle
```

**Cas d'usage :**
- Analytics : `secondary` (ne pas impacter le Primary)
- Applications critiques : `primary` (cohérence forte)
- Multi-région : `nearest` (latence optimale)

---

#### 9.9 Failover et haute disponibilité
**Durée : 2-3 heures**

Gestion automatique des pannes et continuité de service.

**Ce que vous maîtriserez :**
- Processus de failover automatique
- Rolling restart sans downtime
- Stratégies de disaster recovery
- Tests de résilience (chaos engineering)
- RTO et RPO (Recovery Time/Point Objective)

**SLA typiques :**
```
99.9% (3 nines) : ~8.7 heures downtime/an
99.95% : ~4.4 heures downtime/an
99.99% (4 nines) : ~52 minutes downtime/an
99.999% (5 nines) : ~5 minutes downtime/an
```

Un Replica Set bien configuré peut atteindre 99.99%+.

---

#### 9.10-9.12 Monitoring, maintenance et optimisations
**Durée : 3-4 heures**

Opérations avancées et optimisations.

**Métriques critiques :**
- Replication lag
- Oplog window
- Heartbeat latency
- Member state changes
- Network throughput entre membres

---

**Pourquoi ce module est crucial :** La réplication est **non négociable** en production. Même une petite application doit utiliser au minimum un Replica Set à 3 membres.

---

### Module 10 : Sharding (Partitionnement Horizontal)
**Durée estimée : 20-25 heures**

Le sharding permet de dépasser les limites d'un seul serveur en distribuant les données horizontalement.

#### 10.1 Concepts du sharding
**Durée : 2-3 heures**

Fondements théoriques du partitionnement horizontal.

**Ce que vous maîtriserez :**
- Pourquoi sharder (capacité, throughput)
- Quand sharder (seuils recommandés)
- Partitionnement horizontal vs vertical
- Compromis et complexité

**Seuils pour considérer le sharding :**
- Dataset > 1 TB sur un seul serveur
- Throughput > 100K ops/sec
- Croissance > 50% par an
- Besoins de scalabilité géographique

---

#### 10.2 Architecture shardée
**Durée : 3-4 heures**

Composants d'un cluster shardé et leurs interactions.

**Composants :**

**Shards** : Serveurs de données (chacun un Replica Set)
- Stockent une partition des données
- Traitement local des requêtes
- Indépendants les uns des autres

**Config Servers** : Métadonnées du cluster (Replica Set)
- Stockent la cartographie des chunks
- Information sur la distribution
- Critique : si perdus, cluster inutilisable

**Mongos** : Routeurs de requêtes (stateless)
- Point d'entrée pour les clients
- Routing des requêtes vers les bons shards
- Agrégation des résultats
- Peuvent être multiples (load balancing)

**Architecture typique :**
```
Application
     ↓
┌────────────────────────────────┐
│  Mongos (2-3+ instances)       │
└────────────────────────────────┘
     ↓
┌────────────────────────────────┐
│  Config Servers (RS 3 membres) │
└────────────────────────────────┘
     ↓
┌──────────┬──────────┬──────────┐
│ Shard 1  │ Shard 2  │ Shard 3  │
│(RS 3mbr) │(RS 3mbr) │(RS 3mbr) │
└──────────┴──────────┴──────────┘
```

**Minimum pour un cluster shardé de production :**
- 2 shards × 3 membres = 6 serveurs de données
- 3 config servers
- 2+ mongos
- **Total : 11+ serveurs** (vs 3 pour un Replica Set simple)

**Conclusion :** Le sharding ajoute une complexité significative. Ne le faites que si nécessaire.

---

#### 10.3 Shard Key : Choix et stratégies
**Durée : 4-5 heures**

**LE** choix le plus important dans le sharding. Une mauvaise shard key peut ruiner les performances.

**Ce que vous maîtriserez :**
- Critères d'une bonne shard key (cardinalité, distribution, localité)
- Analyse des patterns d'accès
- Exemples de bonnes et mauvaises shard keys
- Impact sur les performances
- Shard key immutable (ne peut être changée facilement)

**Caractéristiques d'une bonne shard key :**

1. **Haute cardinalité** : Beaucoup de valeurs différentes
   - ✅ Bon : userId (millions de valeurs)
   - ❌ Mauvais : country (quelques centaines de valeurs)

2. **Distribution uniforme** : Données équilibrées entre shards
   - ✅ Bon : Hash de userId
   - ❌ Mauvais : timestamp (toutes les nouvelles données sur le même shard)

3. **Localité des requêtes** : Vos requêtes ciblent un seul shard
   - ✅ Bon : tenantId pour SaaS (requêtes par tenant)
   - ❌ Mauvais : Random hash (requêtes broadcast à tous les shards)

**Exemples :**

```javascript
// ❌ MAUVAIS : Timestamp monotone
{ _id: 1 }  // Toutes les insertions vont au même shard (le dernier)

// ❌ MAUVAIS : Faible cardinalité
{ country: 1 }  // Seulement ~200 valeurs possibles

// ✅ BON : Hash de _id
{ _id: "hashed" }  // Distribution uniforme, mais queries souvent broadcast

// ✅ BON : Identifiant d'entité avec haute cardinalité
{ userId: 1 }  // Queries isolées par user

// ✅ EXCELLENT : Compound key avec localité
{ tenantId: 1, timestamp: 1 }  // Isolation par tenant + ordre temporel
```

**Règle d'or :** Choisissez votre shard key en fonction de vos requêtes les plus fréquentes. Si 80% de vos requêtes filtrent par `tenantId`, c'est votre shard key.

---

#### 10.4 Types de sharding
**Durée : 3-4 heures**

Différentes stratégies de distribution.

**Range Sharding** : Distribution par plages de valeurs
- Chunks contigus (ex: 0-100, 101-200)
- Bon pour les requêtes par range
- Risque de hotspots si les données sont séquentielles

**Hashed Sharding** : Distribution par hash
- Distribution uniforme garantie
- Mauvais pour les range queries
- Pas de hotspots

**Zone Sharding** : Affectation manuelle de ranges à des shards
- Contrôle géographique (EU data en EU, US data aux US)
- Conformité réglementaire (GDPR, etc.)
- Isolation multi-tenant

---

#### 10.5 Chunks et balancing
**Durée : 3-4 heures**

Unité de distribution et équilibrage automatique.

**Ce que vous maîtriserez :**
- Concept de chunk (unité de migration)
- Taille de chunk (défaut 64 MB, configurable)
- Balancer automatique (équilibre les chunks entre shards)
- Migration de chunks
- Impact sur la performance

**Problèmes courants :**
- Jumbo chunks (> 64 MB, non migrables)
- Balancing pendant les heures de pointe
- Migrations lentes

---

#### 10.6-10.7 Déploiement et activation
**Durée : 4-5 heures**

Mise en place pratique d'un cluster shardé.

**Étapes :**
1. Déployer les config servers (Replica Set)
2. Déployer les shards (Replica Sets)
3. Déployer les mongos
4. Activer le sharding sur la base
5. Sharder les collections
6. Vérifier la distribution

---

#### 10.8-10.9 Opérations et requêtes
**Durée : 3-4 heures**

Opérations sur un cluster shardé et optimisation des requêtes.

**Types de requêtes :**
- **Targeted queries** : Incluent la shard key → routées vers 1 shard
- **Broadcast queries** : Sans shard key → envoyées à tous les shards

**Exemple :**
```javascript
// Collection sharded on { userId: 1 }

// ✅ Targeted query (fast)
db.orders.find({ userId: "user123" })

// ❌ Broadcast query (slow)
db.orders.find({ productId: "prod456" })
```

**Optimisation :** Incluez toujours la shard key dans vos requêtes fréquentes.

---

#### 10.10-10.12 Monitoring, jumbo chunks, bonnes pratiques
**Durée : 3-4 heures**

Gestion opérationnelle et optimisations avancées.

**Métriques critiques :**
- Distribution des chunks par shard
- Taille des chunks
- Balancer activity
- Query patterns (targeted vs broadcast)
- Hotspots (shards surchargés)

---

**Pourquoi ce module est optionnel au début :** Le sharding ajoute une complexité énorme. Commencez avec un Replica Set et shardez uniquement quand vous atteignez les limites (> 1 TB, throughput insuffisant).

## 🎯 Progression pédagogique

Cette partie suit une logique **haute disponibilité d'abord, puis scalabilité** :

```
Replica Set (HA) → Monitoring → Sharding (Scale) → Optimisation
```

### Semaines 1-3 : Maîtrise de la Réplication
**Focus : Construire des systèmes toujours disponibles**

**Semaine 1 : Concepts et architecture**
- Jours 1-2 : Concepts de réplication et architecture Replica Set
- Jours 3-4 : Types de membres et leurs cas d'usage
- Jours 5-7 : Élection, oplog et cohérence

**Semaine 2 : Déploiement et configuration**
- Jours 1-3 : Déploiement pratique d'un Replica Set
- Jours 4-5 : Read Preference et optimisation des lectures
- Jours 6-7 : Tests de failover et résilience

**Semaine 3 : Operations et monitoring**
- Jours 1-3 : Monitoring et métriques critiques
- Jours 4-5 : Maintenance sans downtime
- Jours 6-7 : Troubleshooting et optimisations

**Livrables :**
- Replica Set de production (3+ membres)
- Documentation de failover et recovery
- Dashboard de monitoring
- Procédures opérationnelles

---

### Semaines 4-7 : Maîtrise du Sharding
**Focus : Scaler horizontalement**

**Semaine 4 : Concepts et architecture**
- Jours 1-2 : Concepts de sharding et composants
- Jours 3-5 : Shard key : analyse et choix
- Jours 6-7 : Types de sharding et stratégies

**Semaine 5 : Déploiement**
- Jours 1-4 : Déploiement d'un cluster shardé complet
- Jours 5-7 : Activation du sharding et tests

**Semaine 6 : Optimisation**
- Jours 1-3 : Optimisation des requêtes (targeted vs broadcast)
- Jours 4-5 : Gestion des chunks et balancing
- Jours 6-7 : Résolution des jumbo chunks

**Semaine 7 : Production-ready**
- Jours 1-3 : Monitoring et métriques avancées
- Jours 4-5 : Bonnes pratiques et anti-patterns
- Jours 6-7 : Consolidation et révision

**Livrables :**
- Cluster shardé de production (2+ shards, chacun un RS)
- Analyse de shard key pour un cas d'usage réel
- Dashboard de monitoring du sharding
- Runbook opérationnel

---

**Rythme recommandé :** 3-5 heures par jour. Le sharding nécessite des sessions intensives de pratique.

## 🧠 Principes architecturaux fondamentaux

### 1. La règle d'or : Haute disponibilité d'abord

> **Toujours** déployez un Replica Set, même pour une petite application. Ne déployez **jamais** un seul mongod en production.

**Pourquoi :**
- Un seul serveur = Single Point of Failure
- Maintenance = Downtime
- Panne matérielle = Perte de données

**Minimum absolu en production :** Replica Set à 3 membres (1 Primary + 2 Secondary)

### 2. Le sharding est une optimisation, pas un prérequis

> Ne shardez que quand vous avez **vraiment** atteint les limites d'un Replica Set simple.

**Shardez si :**
- Dataset > 1 TB
- Throughput > 100K ops/sec sur un serveur
- Working set > RAM disponible
- Croissance rapide (> 50%/an)

**Ne shardez pas si :**
- Dataset < 500 GB
- Vous pouvez scale verticalement
- La complexité opérationnelle vous fait peur

**Réalité :** 80% des applications n'ont jamais besoin de sharding. Un Replica Set bien configuré peut gérer des TB de données et des dizaines de milliers d'ops/sec.

### 3. La shard key est immutable (quasi)

> Choisissez votre shard key avec soin. La changer après coup est extrêmement coûteux.

**Processus de choix :**
1. Analysez vos patterns de requêtes (80% de vos queries)
2. Identifiez les champs avec haute cardinalité
3. Testez sur des données réelles
4. Simulez la croissance à 5 ans
5. Validez avec un expert MongoDB

**Erreur courante :** Sharder trop tôt avec une mauvaise shard key, puis être coincé.

### 4. La localité est votre amie

> Concevez pour minimiser les communications entre shards.

**Bon :**
```javascript
// SaaS multi-tenant sharded on { tenantId: 1 }
// Query: db.data.find({ tenantId: "acme", date: ... })
// → Targeted à 1 shard
```

**Mauvais :**
```javascript
// Sharded on { _id: "hashed" }
// Query: db.data.find({ date: ... })
// → Broadcast à tous les shards
```

**Impact :** Queries broadcast sont 10-100x plus lentes.

### 5. Monitoring avant réactivité

> Vous ne pouvez pas gérer ce que vous ne mesurez pas.

**Métriques essentielles :**
- Replication lag
- Oplog window
- Member states
- Query performance (targeted vs broadcast)
- Chunk distribution
- Balancer activity

**Tooling :**
- MongoDB Atlas (monitoring intégré)
- Ops Manager / Cloud Manager
- Prometheus + Grafana
- Custom dashboards

### 6. Automatisation et Infrastructure as Code

> L'architecture distribuée est trop complexe pour des déploiements manuels.

**Outils recommandés :**
- Terraform (pour MongoDB Atlas)
- Ansible (pour on-premise)
- Kubernetes Operators (pour conteneurs)
- Scripts de déploiement versionés

**Bénéfice :** Reproductibilité, versioning, tests automatisés.

## 🚦 Validation des acquis

Avant de passer à la Partie 5, vous devez maîtriser :

### Checklist Réplication
- [ ] Je peux expliquer le rôle de chaque membre d'un Replica Set
- [ ] Je comprends le processus d'élection et de failover
- [ ] Je sais déployer un Replica Set de production
- [ ] Je peux configurer Read Preference selon les cas d'usage
- [ ] Je comprends l'oplog et son dimensionnement
- [ ] Je sais monitorer le replication lag
- [ ] Je peux effectuer une maintenance sans downtime
- [ ] J'ai testé un failover en conditions réelles

### Checklist Sharding
- [ ] Je comprends quand sharder (et quand ne pas sharder)
- [ ] Je peux analyser et choisir une shard key appropriée
- [ ] Je connais les différences entre Range et Hashed sharding
- [ ] Je sais déployer un cluster shardé complet
- [ ] Je comprends l'architecture (shards, config, mongos)
- [ ] Je peux optimiser les requêtes pour le sharding
- [ ] Je sais gérer les chunks et le balancing
- [ ] Je peux diagnostiquer et résoudre les jumbo chunks

### Checklist Architecture
- [ ] Je peux concevoir une architecture HA pour une application
- [ ] Je sais dimensionner un Replica Set (nombre de membres, ressources)
- [ ] Je peux justifier le choix entre RS simple et cluster shardé
- [ ] Je comprends les compromis entre coût, complexité et performance
- [ ] Je peux concevoir une architecture multi-région
- [ ] J'ai un plan de disaster recovery documenté

### Checklist Opérationnelle
- [ ] Je peux monitorer tous les composants critiques
- [ ] Je sais diagnostiquer les problèmes de réplication et sharding
- [ ] J'ai des runbooks pour les incidents courants
- [ ] Je peux effectuer un rolling restart sans downtime
- [ ] J'ai testé mes procédures de recovery

**Objectif :** Cocher 90%+ de ces cases. L'architecture distribuée est critique et les erreurs coûtent cher.

## 🎯 Projets pratiques recommandés

### Projet 1 : Replica Set de production
**Durée : 15-20 heures**

**Objectif :** Déployer et opérer un Replica Set production-ready.

**Tâches :**
1. Déployer un RS à 3 membres (Docker ou VMs)
2. Configurer monitoring (Grafana + Prometheus)
3. Tester le failover (kill Primary, observer élection)
4. Implémenter rolling restart
5. Simuler replication lag et le résoudre
6. Documenter les runbooks

**Livrables :**
- Infrastructure as Code (Terraform/Ansible)
- Dashboards de monitoring
- Tests de résilience documentés
- Procédures opérationnelles

---

### Projet 2 : Cluster shardé avec shard key analysis
**Durée : 25-30 heures**

**Objectif :** Déployer un cluster shardé et optimiser la shard key.

**Tâches :**
1. Analyser un dataset réel (logs, e-commerce, etc.)
2. Choisir et justifier une shard key
3. Déployer le cluster (2 shards, config servers, mongos)
4. Sharder les collections
5. Benchmarker targeted vs broadcast queries
6. Monitorer la distribution des chunks
7. Documenter l'architecture et les choix

**Livrables :**
- Document d'analyse de shard key
- Cluster fonctionnel
- Benchmarks de performance
- Monitoring complet
- Recommandations d'optimisation

---

### Projet 3 : Architecture multi-région
**Durée : 20-25 heures**

**Objectif :** Concevoir une architecture avec contraintes géographiques.

**Contraintes :**
- Données EU doivent rester en EU (GDPR)
- Latence < 100ms pour les utilisateurs locaux
- Tolérance à la panne d'une région entière

**Livrables :**
- Design doc de l'architecture
- Déploiement sur 3 régions (simulé)
- Tests de failover inter-région
- Analyse de latence

---

Ces projets vous donneront une expérience pratique complète et constitueront d'excellents ajouts à votre portfolio.

## 📊 Comparaison des architectures

| Architecture | Coût | Complexité | HA | Scalabilité | Maintenance | Cas d'usage |
|--------------|------|------------|-----|-------------|-------------|-------------|
| **Standalone** | € | ⭐ | ❌ | ❌ | Simple | Dev/test uniquement |
| **Replica Set (3)** | €€€ | ⭐⭐ | ✅✅✅ | ⭐ | Moyenne | 90% des apps prod |
| **Replica Set (5)** | €€€€€ | ⭐⭐ | ✅✅✅✅ | ⭐ | Moyenne | Apps critiques |
| **Sharded (2 shards)** | €€€€€€ | ⭐⭐⭐⭐ | ✅✅✅ | ✅✅ | Complexe | > 1 TB |
| **Sharded (5+ shards)** | €€€€€€€€ | ⭐⭐⭐⭐⭐ | ✅✅✅ | ✅✅✅✅✅ | Très complexe | Hyper-scale |

**Conseil :** Commencez simple, évoluez selon les besoins réels.

## 🌟 Conseils d'architecte

### 1. KISS (Keep It Simple, Stupid)
Utilisez l'architecture la plus simple qui répond à vos besoins. La complexité a un coût (opérationnel, bugs, maintenance).

### 2. Plan for failure
Tout tombe : hardware, réseau, datacenter. Concevez pour la résilience dès le début.

### 3. Measure twice, cut once
Testez intensivement avant la production. Un failover qui échoue en production est catastrophique.

### 4. Document everything
Dans 6 mois, vous aurez oublié pourquoi vous avez fait certains choix. Documentez votre architecture et vos décisions.

### 5. Automate relentlessly
Tout processus manuel sera oublié ou mal exécuté sous stress. Automatisez les déploiements, les backups, le monitoring.

### 6. Learn from others' mistakes
Lisez les post-mortems publics (MongoDB, GitHub, etc.). Les erreurs d'architecture distribuée sont coûteuses.

## 📚 Ressources complémentaires

### Documentation officielle
- [MongoDB Replication](https://www.mongodb.com/docs/manual/replication/)
- [MongoDB Sharding](https://www.mongodb.com/docs/manual/sharding/)
- [Production Notes](https://www.mongodb.com/docs/manual/administration/production-notes/)

### Livres essentiels
- *Designing Data-Intensive Applications* par Martin Kleppmann
- *Database Internals* par Alex Petrov
- *MongoDB: The Definitive Guide* (3rd ed.)

### Cours et certifications
- MongoDB University (cours M103, M201, M320)
- MongoDB Certified DBA Associate

### Communauté
- MongoDB Community Forums
- MongoDB User Groups
- Conférences (MongoDB World, MongoDB.local)

## 🚀 Et après ?

Une fois cette partie maîtrisée, vous serez capable de **concevoir et opérer des architectures MongoDB distribuées** pour des applications de classe mondiale. Vous comprendrez :

- Comment garantir 99.99%+ uptime avec les Replica Sets
- Comment scaler horizontalement avec le sharding
- Les compromis entre disponibilité, cohérence et performance
- Comment opérer des systèmes distribués en production

La **Partie 5** vous enseignera la sécurité et l'administration avancée, essentielles pour protéger vos données et gérer vos clusters en production.

La **Partie 6** couvrira MongoDB Atlas et le cloud, vous permettant de déployer des architectures distribuées sans gérer l'infrastructure.

Mais d'abord, **maîtrisez cette Partie 4**. L'architecture distribuée est le fondement de toute application MongoDB à grande échelle. Une architecture bien conçue dès le départ vous fera économiser des années de problèmes.

---

**Prêt à construire des systèmes distribués de classe mondiale ? Allons-y ! 🌍**

---

**Prochaine étape :** [Module 9 - Réplication →](/09-replication/README.md)

---

*💡 Citation du jour : "A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable." - Leslie Lamport (inventeur de Paxos)*

⏭️ [Module 9 - Réplication →](/09-replication/README.md)
