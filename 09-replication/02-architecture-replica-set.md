🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.2 Architecture Replica Set

## Introduction

L'architecture Replica Set de MongoDB représente une implémentation sophistiquée du modèle de réplication single-leader, conçue pour offrir haute disponibilité, durabilité des données et évolutivité en lecture dans un environnement de production. Cette section explore en profondeur les composants architecturaux, leurs interactions, et les principes de conception qui sous-tendent cette architecture distribuée.

## Vue d'Ensemble Architecturale

### Définition d'un Replica Set

Un **Replica Set** est un cluster de processus `mongod` interconnectés qui maintiennent collectivement le même dataset à travers un mécanisme de réplication asynchrone. Contrairement à une simple réplication master-slave, un Replica Set intègre nativement :

- **Élection automatique** : Sélection démocratique d'un nouveau Primary en cas de défaillance
- **Détection de pannes** : Mécanisme de heartbeat pour identifier les nœuds défaillants
- **Reconfiguration dynamique** : Ajout/suppression de membres sans arrêt du service
- **Gestion de la topologie** : Adaptation automatique aux changements de configuration

### Composants Principaux

L'architecture d'un Replica Set repose sur plusieurs composants fondamentaux :

#### 1. Nœuds de Données (Data-Bearing Nodes)

**Primary Node** :
- Unique nœud acceptant les opérations d'écriture
- Maintient le dataset autoritatif
- Génère l'Oplog à partir des écritures
- Envoie des heartbeats aux autres membres
- Se dégrade automatiquement (step down) s'il ne peut atteindre la majorité

**Secondary Nodes** :
- Répliquent les données depuis le Primary (ou une autre source)
- Appliquent les opérations de l'Oplog dans le même ordre
- Peuvent servir les lectures (selon Read Preference)
- Participent aux élections en cas de panne du Primary
- Maintiennent leur propre copie de l'Oplog

#### 2. Arbiters (Optionnels)

Les **Arbiters** sont des membres spéciaux du Replica Set qui :
- Participent aux élections (votent) mais ne stockent pas de données
- Ne peuvent jamais devenir Primary
- Nécessitent des ressources minimales (RAM, CPU, disque)
- Servent de "tie-breaker" dans les topologies à nombre pair de membres
- **Limitation critique** : Ne contribuent pas à la durabilité des données

**Note importante** : L'utilisation d'Arbiters est généralement déconseillée en production au profit de véritables Secondary nodes qui offrent à la fois redondance et disponibilité.

#### 3. Membres Spécialisés

**Hidden Members** :
- Répliquent les données mais sont invisibles aux applications
- Ne peuvent pas devenir Primary (priority: 0)
- Utilisés pour backup, reporting, analytics sans impact sur les opérations
- Votent dans les élections (sauf configuration contraire)

**Delayed Members** :
- Répliquent avec un délai configurable (ex: 1 heure, 4 heures)
- Protection contre les erreurs humaines (suppression accidentelle)
- Doivent être Hidden (car données obsolètes)
- Ne peuvent pas devenir Primary

**Non-Voting Members** :
- Participent à la réplication mais ne votent pas aux élections
- Utiles au-delà de 7 membres (limite de votants)
- Peuvent devenir Primary si configurés avec priority > 0

## Architecture de Communication

### Topologie Réseau

Un Replica Set forme un **graphe complet** (fully connected mesh) où chaque membre peut communiquer directement avec tous les autres membres.

```
         [Primary]
        /    |    \
       /     |     \
      /      |      \
[Secondary] [Secondary] [Secondary]
     \      /  \      /
      \    /    \    /
       \  /      \  /
    (tous connectés entre eux)
```

**Implications** :
- Pas de point de passage obligé (pas de SPOF au niveau réseau)
- Latence directe entre n'importe quelle paire de nœuds
- Complexité O(n²) en termes de connexions (n membres)
- Chaque membre maintient n-1 connexions persistantes

### Protocole de Heartbeat

Les membres échangent périodiquement des messages de heartbeat pour :

#### Fréquence et Timing

- **heartbeatIntervalMillis** : 2000 ms (défaut)
  - Intervalle entre deux heartbeats consécutifs
  - Chaque membre envoie un heartbeat à tous les autres
  - Configurable mais rarement modifié en production

- **electionTimeoutMillis** : 10000 ms (défaut)
  - Délai sans heartbeat du Primary avant qu'un Secondary puisse initier une élection
  - Balance entre détection rapide et stabilité (éviter les élections intempestives)
  - Doit être supérieur à heartbeatIntervalMillis × N pour tolérer quelques heartbeats perdus

- **heartbeatTimeoutSecs** : 10 s (défaut)
  - Timeout pour considérer un heartbeat comme échoué
  - Un membre est marqué comme inaccessible après ce délai

#### Contenu des Heartbeats

Chaque message heartbeat transporte :
- **État du membre** : PRIMARY, SECONDARY, RECOVERING, ARBITER, etc.
- **OpTime** : Position dans l'Oplog (term, timestamp) → permet de mesurer le lag
- **Configuration version** : Détecte les désynchronisations de configuration
- **Health status** : Indicateurs de santé (charge, erreurs)
- **Election metadata** : Informations pour les protocoles d'élection

#### Détection de Défaillances

Un membre est considéré comme défaillant si :
1. Aucun heartbeat reçu pendant `electionTimeoutMillis`
2. Les heartbeats échouent systématiquement (timeout réseau)
3. Le membre rapporte explicitement une erreur fatale

**Stratégie de détection** :
- **Phi Accrual Failure Detector** : MongoDB utilise une variation de cet algorithme adaptatif
- Accumule l'historique des heartbeats pour estimer la probabilité de défaillance
- Évite les faux positifs dus aux variations temporaires de latence

### Canaux de Réplication

La réplication des données s'effectue via plusieurs canaux logiques :

#### 1. Oplog Streaming

**Mécanisme** :
- Les Secondaries ouvrent un **tailable cursor** sur `local.oplog.rs` du Primary (ou d'une source de réplication)
- Le cursor reste ouvert et "suit" les nouvelles entrées au fur et à mesure
- Transmission des opérations par batches pour optimiser la bande passante
- Compression optionnelle (snappy, zlib, zstd) pour réduire le trafic réseau

**Optimisations** :
- **Oplog batching** : Les Secondaries récupèrent plusieurs opérations en une seule requête
- **Parallel application** : Depuis MongoDB 4.0, application parallèle des opérations non conflictuelles
- **Streaming replication** : Introduit dans 4.2, réduit la latence en envoyant les données dès leur écriture

#### 2. Initial Sync

Lorsqu'un nouveau Secondary rejoint ou qu'un Secondary est trop en retard (Oplog gap) :

**Phases de l'Initial Sync** :
1. **Clone des données** :
   - Copie complète de toutes les bases (sauf `local`)
   - Utilise des cursors parallèles pour accélérer le transfert
   - Peut être interrompu et repris (resumable depuis MongoDB 4.4)

2. **Application Oplog** :
   - Pendant le clone, les nouvelles opérations s'accumulent
   - Une fois le clone terminé, application des opérations manquées
   - Peut nécessiter plusieurs itérations si le débit d'écriture est élevé

3. **Final catch-up** :
   - Application des dernières opérations
   - Transition vers l'état SECONDARY
   - Début de la réplication continue

**Coût** : L'Initial Sync est très coûteux en ressources (CPU, disque, réseau) et peut prendre des heures/jours pour de gros datasets.

#### 3. Replication Chaining (Chaînage)

Par défaut, les Secondaries répliquent depuis le Primary, mais le **chaining** permet à un Secondary de répliquer depuis un autre Secondary.

**Avantages** :
- Réduit la charge réseau sur le Primary
- Optimise la bande passante dans les topologies multi-datacenters
- Permet des topologies hiérarchiques (ex: DC1 → DC2 → DC3)

**Configuration** :
- Activé par défaut (`settings.chainingAllowed: true`)
- MongoDB sélectionne automatiquement la meilleure source (nearest avec Oplog à jour)

**Risques** :
- Augmente le replication lag (effet cascade)
- Complexifie le diagnostic en cas de problème
- Peut être désactivé pour forcer la réplication depuis le Primary uniquement

## Architecture de Données

### Organisation du Stockage

Chaque membre d'un Replica Set maintient plusieurs bases et collections critiques :

#### Base de Données `local`

La base `local` est spéciale : **elle n'est jamais répliquée**. Elle contient :

**local.oplog.rs** :
- Collection capped (taille fixe, FIFO)
- Journal ordonné de toutes les opérations de modification
- Taille configurable (`oplogSizeMB` au démarrage)
- Crucial pour la réplication et la récupération

**local.replset.minvalid** :
- Stocke l'OpTime minimal valide après un crash
- Garantit la cohérence après un redémarrage
- Utilisé pour éviter la lecture de données partiellement appliquées

**local.replset.oplogTruncateAfterPoint** :
- Point jusqu'où l'Oplog peut être tronqué en toute sécurité
- Utilisé lors de la récupération après panne

**local.system.replset** :
- Configuration actuelle du Replica Set
- Versionnée pour détecter les changements de configuration
- Partagée entre tous les membres (répliquée via un mécanisme spécial)

**local.startup_log** :
- Journal des démarrages du `mongod`
- Informations de diagnostic (version, configuration, hardware)

#### Bases de Données Utilisateur

Toutes les bases autres que `local` sont répliquées :
- Stockage WiredTiger avec journalisation
- Isolation MVCC (Multi-Version Concurrency Control)
- Compression configurable (snappy, zlib, zstd, none)
- Checkpoints périodiques pour durabilité

### L'Oplog : Structure et Sémantique

L'Oplog est au cœur de l'architecture de réplication. Explorons sa structure en détail.

#### Format des Entrées Oplog

Chaque opération dans l'Oplog est un document BSON :

```javascript
{
  "ts": Timestamp(1638360000, 1),     // Timestamp BSON (secondes, ordinal)
  "t": NumberLong(3),                  // Term number (mandat du Primary)
  "h": NumberLong("8124378213847"),    // Hash de l'opération (legacy)
  "v": 2,                              // Version du format Oplog
  "op": "i",                           // Type d'opération (i, u, d, c, n)
  "ns": "mydb.users",                  // Namespace (base.collection)
  "ui": UUID("..."),                   // UUID de la collection
  "wall": ISODate("2021-12-01T12:00:00Z"), // Timestamp mural
  "o": {                               // Objet opération
    "_id": ObjectId("..."),
    "name": "Alice",
    "email": "alice@example.com"
  },
  "o2": { "_id": ObjectId("...") }     // Objet critère (pour updates)
}
```

#### Types d'Opérations (op)

- **"i"** (insert) : Insertion d'un document, `o` contient le document complet
- **"u"** (update) : Mise à jour, `o` contient les modificateurs, `o2` le critère de sélection
- **"d"** (delete) : Suppression, `o2` contient le critère
- **"c"** (command) : Commande administrative (createIndex, dropCollection, etc.)
- **"n"** (noop) : Opération no-op (keepalive, marqueurs)

#### Idempotence des Opérations

**Principe fondamental** : Toutes les opérations de l'Oplog doivent être **idempotentes**, c'est-à-dire qu'elles peuvent être appliquées plusieurs fois avec le même résultat.

**Transformation pour l'idempotence** :

Original (client) :
```javascript
db.counters.updateOne(
  { _id: "page_views" },
  { $inc: { count: 1 } }
)
```

Oplog (transformé) :
```javascript
{
  "op": "u",
  "ns": "stats.counters",
  "o": { $v: 1, $set: { count: 42 } },  // Valeur finale, pas l'incrément
  "o2": { _id: "page_views" }
}
```

Cette transformation garantit qu'appliquer l'opération multiple fois (ex: après un crash) produit le même état final.

#### Dimensionnement de l'Oplog

La taille de l'Oplog est un paramètre critique :

**Calcul de la taille** :
```
OplogSize (GB) = Taux_écriture (GB/h) × Fenêtre_souhaitée (h) × Facteur_sécurité
```

**Exemple** :
- Taux d'écriture : 5 GB/h
- Fenêtre souhaitée : 24h (permettre une panne de 24h sans resync)
- Facteur de sécurité : 2x
- **OplogSize = 5 × 24 × 2 = 240 GB**

**Taille par défaut** :
- 5% de l'espace disque disponible
- Minimum 1 GB, maximum 50 GB
- Souvent insuffisant pour la production

**Redimensionnement** :
- Depuis MongoDB 4.0 : `replSetResizeOplog` en ligne (sans redémarrage)
- Avant 4.0 : nécessite arrêt et modification des fichiers de données

## Topologies de Replica Set

### Topologie Standard : 3 Membres

La configuration la plus courante et recommandée :

```
    [Primary]
      /    \
     /      \
[Secondary] [Secondary]
```

**Propriétés** :
- **Quorum** : 2 membres sur 3 (majorité)
- **Tolérance aux pannes** : 1 membre peut tomber sans perte de disponibilité
- **Redondance** : 3 copies des données
- **Élections** : Toujours possible même avec 1 panne

**Cas d'usage** : Production standard, équilibre optimal entre coût et résilience.

### Topologie à 5 Membres

Pour une résilience accrue :

```
       [Primary]
      /    |    \
     /     |     \
[Sec1]  [Sec2]  [Sec3]  [Sec4]
```

**Propriétés** :
- **Quorum** : 3 membres sur 5
- **Tolérance aux pannes** : 2 membres peuvent tomber
- **Coût** : 5 serveurs complets (ou Secondaries + Arbiters, déconseillé)
- **Cas d'usage** : Applications critiques, multi-datacenters

### Topologie avec Arbiter (À Éviter)

```
    [Primary]
      /    \
     /      \
[Secondary] [Arbiter]
```

**Problèmes** :
- Pas de redondance si le Primary tombe (1 seule copie des données récentes)
- Risque de perte de données avec `w: 1`
- Protection minimale contre la corruption

**Alternative recommandée** : 3 Secondaries complets, même si plus coûteux.

### Topologie Multi-Datacenter

Configuration pour haute disponibilité géographique :

#### Distribution 2-1

```
    Datacenter A              Datacenter B
    [Primary]                 [Secondary]
    [Secondary]
```

**Avantages** :
- Survie à la perte d'un datacenter entier
- Latence locale pour les lectures (Read Preference: nearest)

**Inconvénients** :
- Si DC-A tombe, pas de quorum possible (1/3) → read-only
- Nécessite un 3ème site ou un arbiter cloud pour le quorum

#### Distribution 2-2-1 (Recommandée)

```
    DC-A              DC-B            DC-C (Cloud)
   [Primary]        [Secondary]      [Secondary]
   [Secondary]      [Secondary]
```

**Avantages** :
- Survie à la perte d'un DC complet avec quorum
- Élections possibles après panne de DC
- Meilleure distribution géographique

**Considération** : Latence inter-DC impacte le write concern "majority".

#### Distribution avec Priority

Contrôler quel datacenter héberge le Primary :

```javascript
cfg = rs.conf()
cfg.members[0].priority = 2  // DC-A: priorité haute
cfg.members[1].priority = 2  // DC-A
cfg.members[2].priority = 1  // DC-B: priorité normale
cfg.members[3].priority = 1  // DC-B
cfg.members[4].priority = 0.5 // DC-C: priorité basse (backup)
rs.reconfig(cfg)
```

Le Primary sera préférentiellement dans DC-A (latence minimale pour les applications principales).

### Topologie avec Membres Hidden

Pour isoler les charges analytiques :

```
[Primary] ← [Secondary] ← [Secondary (Reporting)]
                ↑              (Hidden, Priority: 0)
                |
            [Secondary]
```

**Configuration du membre Hidden** :
```javascript
cfg = rs.conf()
cfg.members[3].priority = 0
cfg.members[3].hidden = true
cfg.members[3].tags = { usage: "reporting" }
rs.reconfig(cfg)
```

**Cas d'usage** :
- Requêtes analytiques lourdes sans impact sur les opérations OLTP
- Backups continus via mongodump/mongorestore
- Extraction ETL vers data warehouses

### Topologie avec Delayed Member

Protection contre les erreurs humaines :

```
[Primary] → [Secondary] → [Secondary (Delayed 4h)]
                            (Hidden, Priority: 0,
                             SlaveDelay: 14400)
```

**Configuration** :
```javascript
cfg = rs.conf()
cfg.members[4].priority = 0
cfg.members[4].hidden = true
cfg.members[4].slaveDelay = 14400  // 4 heures en secondes
cfg.members[4].votes = 0  // Optionnel: ne vote pas
rs.reconfig(cfg)
```

**Cas d'usage** :
- Protection contre `db.collection.drop()` accidentel
- Récupération de données supprimées par erreur (si détecté dans les 4h)
- Ne remplace PAS les backups réguliers

## Élection et Consensus : Architecture du Protocole

### Protocole Raft dans MongoDB

MongoDB implémente une variation du protocole Raft pour gérer le consensus et les élections.

#### États des Nœuds

Chaque membre peut être dans un des états suivants :

1. **Leader (Primary)** :
   - Accepte les écritures
   - Envoie des heartbeats réguliers (AppendEntries)
   - Réplique l'Oplog vers les Followers

2. **Follower (Secondary)** :
   - Réplique depuis le Leader ou une autre source
   - Répond aux heartbeats
   - Initie une élection si pas de heartbeat pendant `electionTimeout`

3. **Candidate** :
   - État transitoire pendant une élection
   - Sollicite les votes des autres membres
   - Devient Leader si majorité de votes obtenus

#### Mécanisme d'Élection

**Déclencheurs d'élection** :
1. Démarrage initial du Replica Set (tous en état Candidate)
2. Perte de heartbeat du Primary pendant `electionTimeoutMillis`
3. Commande manuelle `rs.stepDown()`
4. Reconfigurations avec changement de priorité

**Processus d'élection** :

1. **Transition vers Candidate** :
   - Un Secondary ne recevant plus de heartbeat du Primary incrémente son **term number**
   - Entre en état CANDIDATE
   - Vote pour lui-même
   - Envoie des **RequestVote RPC** à tous les autres membres

2. **Critères de Vote** :
   Un membre vote pour un Candidate si :
   - Le term du Candidate est supérieur ou égal au term local
   - Le membre n'a pas déjà voté dans ce term
   - L'Oplog du Candidate est au moins aussi à jour (basé sur OpTime)
   - La priorité du Candidate le permet (priority > 0)
   - Les contraintes de tags sont respectées (si configurées)

3. **Obtention de la Majorité** :
   - Si le Candidate reçoit la majorité des votes (quorum), il devient PRIMARY
   - Commence immédiatement à envoyer des heartbeats (AppendEntries)
   - Accepte les écritures

4. **Échec d'Élection** :
   - Si aucun Candidate n'obtient la majorité (split vote)
   - Nouveau timeout aléatoire (`electionTimeout` × random[1.0, 1.5])
   - Nouvelle tentative d'élection (term incrémenté)

**Randomisation** : Le timeout aléatoire évite les élections synchronisées qui conduiraient à des split votes répétés.

#### Term Numbers

Le **term** est un compteur logique monotone qui :
- S'incrémente à chaque nouvelle élection
- Identifie de manière unique chaque "mandat" d'un Primary
- Est inclus dans chaque opération de l'Oplog (champ `t`)
- Permet de détecter les configurations obsolètes

**Propriété de sécurité** : Pour un term donné, il existe au plus un Leader.

### Priorités et Éligibilité

La **priority** influence la probabilité qu'un membre devienne Primary :

**Valeurs de priority** :
- **priority: 0** → Jamais Primary (Hidden, Delayed, Arbiters)
- **0 < priority ≤ 1000** → Éligible, valeur plus élevée = préféré
- **priority par défaut** : 1.0

**Impact sur les élections** :
- Un membre avec priority élevée sollicite l'élection plus rapidement (timeout réduit)
- Les autres membres préfèrent voter pour les membres de priorité élevée
- Permet de contrôler quel datacenter héberge le Primary

**Exemple** :
```javascript
cfg = rs.conf()
// DC principal (priorité haute)
cfg.members[0].priority = 2
cfg.members[1].priority = 2
// DC secondaire (priorité normale)
cfg.members[2].priority = 1
cfg.members[3].priority = 1
// DC distant (priorité basse, backup)
cfg.members[4].priority = 0.5
rs.reconfig(cfg)
```

### Votes et Quorum

#### Membres Votants vs Non-Votants

- **Votants** : Participent aux élections (max 7 dans un Replica Set)
- **Non-votants** : Répliquent mais ne votent pas (`votes: 0`)

**Limitation à 7 votants** : Compromis entre temps de convergence et résilience.

#### Calcul du Quorum

**Formule** : Majorité = ⌊N/2⌋ + 1 (où N = nombre de membres votants)

**Exemples** :
- 3 membres → quorum = 2
- 5 membres → quorum = 3
- 7 membres → quorum = 4

**Implications** :
- Un Replica Set de 3 membres tolère la perte de 1 membre
- Un Replica Set de 5 membres tolère la perte de 2 membres
- **Ajouter un 4ème membre n'améliore PAS la tolérance aux pannes** (quorum passe de 2 à 3)

**Recommandation** : Toujours utiliser un nombre impair de membres votants.

## Architecture de Reconfiguration

### Configuration Dynamique

La configuration d'un Replica Set est stockée dans un document BSON versionné :

```javascript
{
  _id: "myReplicaSet",
  version: 3,  // Incrémenté à chaque modification
  members: [
    {
      _id: 0,
      host: "mongo1.example.com:27017",
      priority: 2,
      tags: { dc: "east", rack: "1" }
    },
    {
      _id: 1,
      host: "mongo2.example.com:27017",
      priority: 1,
      tags: { dc: "east", rack: "2" }
    },
    {
      _id: 2,
      host: "mongo3.example.com:27017",
      priority: 1,
      tags: { dc: "west", rack: "1" }
    }
  ],
  settings: {
    chainingAllowed: true,
    heartbeatIntervalMillis: 2000,
    electionTimeoutMillis: 10000,
    catchUpTimeoutMillis: -1,
    getLastErrorDefaults: { w: "majority", wtimeout: 5000 }
  }
}
```

### Processus de Reconfiguration

Toute modification de la configuration passe par `rs.reconfig()` :

**Étapes** :
1. **Validation** : Le Primary valide la nouvelle configuration
2. **Propagation** : La configuration est répliquée comme une opération spéciale
3. **Application** : Chaque membre applique la nouvelle configuration
4. **Version** : Le numéro de version est incrémenté

**Reconfiguration sécurisée** :
- Nécessite un Primary disponible (sauf `force: true`)
- La nouvelle configuration doit avoir un quorum valide
- Les membres existants doivent reconnaître leur nouvelle identité

**Reconfiguration forcée** (`force: true`) :
- Utilisée en cas de perte de quorum (disaster recovery)
- Peut entraîner une perte de données (rollback)
- Ne doit être utilisée qu'en dernier recours

### Tags et Read Preference

Les **tags** permettent d'annoter les membres pour le routage intelligent des requêtes :

```javascript
cfg = rs.conf()
cfg.members[0].tags = { dc: "nyc", usage: "production" }
cfg.members[1].tags = { dc: "nyc", usage: "production" }
cfg.members[2].tags = { dc: "sfo", usage: "analytics" }
rs.reconfig(cfg)
```

**Utilisation** :
```javascript
db.collection.find().readPref("secondary", [
  { dc: "sfo", usage: "analytics" }  // Préférence
])
```

Permet d'isoler les charges de travail et d'optimiser la latence réseau.

## Considérations de Performance

### Impact de la Topologie sur les Performances

#### Latence Réseau Inter-Membres

La latence entre nœuds impacte directement :

- **Write Concern "majority"** : Le temps d'ACK est limité par le Secondary le plus lent
- **Élections** : Une latence élevée augmente le temps de convergence
- **Replication Lag** : Les liens lents créent du retard de réplication

**Recommandations** :
- Latence intra-DC : < 1 ms (idéal)
- Latence inter-DC : < 50 ms (acceptable pour w: majority)
- Latence inter-continent : > 100 ms (problématique, nécessite tuning)

#### Bandwidth Requirements

**Estimation de la bande passante** :
```
BW (Mbps) = Taux_écriture (MB/s) × 8 × Nombre_Secondaries × Overhead
```

**Exemple** :
- Taux d'écriture : 10 MB/s
- 2 Secondaries
- Overhead (compression, protocole) : 1.5x
- **BW = 10 × 8 × 2 × 1.5 = 240 Mbps**

**Optimisations** :
- Compression Oplog (snappy, zstd) : réduction de 50-70%
- Replication chaining : réduit la charge sur le Primary
- Déduplication implicite (batching)

### Dimensionnement des Ressources

#### CPU

- **Primary** : Charge élevée (écritures, lectures, génération Oplog)
- **Secondaries** : Charge modérée (application Oplog, lectures si Read Preference)
- **Recommandation** : CPU équivalent entre Primary et Secondaries pour faciliter le failover

#### RAM

- **Working Set** : Doit tenir en RAM pour performances optimales
- **WiredTiger Cache** : Par défaut 50% de RAM - 1 GB (configurable)
- **Recommandation** : RAM(Secondary) ≥ RAM(Primary)

#### Disque

- **IOPS** : Crucial pour les écritures et l'application de l'Oplog
- **Latency** : SSD fortement recommandé en production
- **Espace** : Prévoir pour l'Oplog + données + indexes + marge (20%)

**Formule Oplog** :
```
Espace_Oplog = Max(
  5% × Espace_disque_total,
  Taux_écriture × Fenêtre_souhaitée × 1.5
)
```

## Surveillance et Observabilité

### Métriques Architecturales Clés

**Santé du Replica Set** :
- État de chaque membre (PRIMARY, SECONDARY, RECOVERING, etc.)
- Connectivité inter-membres (heartbeat success rate)
- Version de configuration (config version mismatch)

**Performances de Réplication** :
- Replication lag par Secondary
- Oplog window (temps avant que l'Oplog ne soit plein)
- Throughput de réplication (ops/sec appliquées)

**Élections** :
- Nombre d'élections (haute fréquence = instabilité)
- Durée des élections (> 30s = problème)
- Raison des élections (heartbeat timeout, priority change, etc.)

### Commandes de Monitoring

```javascript
// État complet du Replica Set
rs.status()

// Configuration actuelle
rs.conf()

// Informations sur la réplication (vue depuis un membre)
db.getReplicationInfo()  // Info sur l'Oplog local
db.printSlaveReplicationInfo()  // Lag des Secondaries

// Oplog window
db.getReplicationInfo().timeDiff  // En secondes
```

## Sécurité de l'Architecture

### Authentication Inter-Membres

Les membres d'un Replica Set doivent s'authentifier mutuellement via :

#### Keyfile Authentication

Méthode simple pour environnements non critiques :
```bash
# Générer une keyfile
openssl rand -base64 756 > /path/to/keyfile
chmod 400 /path/to/keyfile

# Configuration mongod
security:
  keyFile: /path/to/keyfile
  authorization: enabled
```

**Limitations** : Sécurité basique (shared secret), pas de révocation granulaire.

#### x.509 Certificate Authentication

Méthode recommandée pour la production :
```yaml
security:
  clusterAuthMode: x509
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /path/to/member-cert.pem
    CAFile: /path/to/ca.pem
```

**Avantages** :
- Cryptographie à clé publique (pas de secret partagé)
- Identité vérifiable par certificat
- Révocation possible via CRL

### Encryption en Transit

Toutes les communications inter-membres doivent être chiffrées :

```yaml
net:
  tls:
    mode: requireTLS
    certificateKeyFile: /path/to/cert.pem
    CAFile: /path/to/ca.pem
    allowConnectionsWithoutCertificates: false
```

**Protocoles** : TLS 1.2+ (TLS 1.3 recommandé)

### Network Isolation

**Bonnes pratiques** :
- Réseau privé dédié pour la communication inter-membres
- Firewalls restreignant l'accès au port MongoDB (27017)
- VPN ou peering privé pour les connexions inter-DC
- Pas d'exposition publique des membres du Replica Set

## Limitations et Contraintes Architecturales

### Limites Numériques

- **Maximum 50 membres** par Replica Set
- **Maximum 7 membres votants** (compromis consensus/performance)
- **Maximum 1 Arbiter** recommandé (plus = inutile)
- **Taille maximale Oplog** : Limitée par l'espace disque disponible

### Contraintes de Conception

- **Single-Leader** : Un seul Primary, point de contention pour les écritures
- **Réplication asynchrone** : Replication lag inévitable (cohérence éventuelle)
- **Quorum requis** : Perte de la majorité → cluster read-only
- **Latence inter-DC** : Impacte les performances avec w: "majority"

### Trade-offs Architecturaux

**Nombre de membres** :
- Plus de membres = plus de redondance mais plus de complexité et coût
- 3 membres (production standard) vs 5 membres (haute criticité)

**Distribution géographique** :
- Latence réduite (membres proches) vs résilience (distribution large)
- Compromis entre performance locale et disaster recovery

**Write Concern** :
- w:1 (rapide mais risqué) vs w:"majority" (lent mais durable)
- À ajuster selon les besoins de l'application

## Conclusion

L'architecture Replica Set de MongoDB est une construction sophistiquée qui intègre :

- **Consensus distribué** via Raft pour garantir l'élection sûre d'un leader unique
- **Réplication asynchrone** avec Oplog idempotent pour la propagation efficace des données
- **Détection de pannes** via heartbeats et timeouts adaptatifs
- **Flexibilité topologique** pour s'adapter à divers besoins (HA, DR, performance)
- **Configurabilité fine** via Write/Read Concerns, priorities, tags

**Points clés à retenir** :

1. Un Replica Set forme un **graphe complet** où chaque membre communique avec tous les autres
2. Le **protocole Raft** garantit qu'il n'existe qu'un seul Primary à tout moment
3. L'**Oplog** est l'épine dorsale de la réplication, avec des propriétés d'idempotence et d'ordre total
4. La **topologie** doit être conçue en fonction des besoins de disponibilité, latence et coût
5. Les **membres spécialisés** (Hidden, Delayed) permettent d'isoler les charges de travail
6. La **latence réseau** et la **bande passante** sont des facteurs critiques de performance
7. Le **quorum** (majorité) est essentiel pour les élections et le write concern "majority"

La maîtrise de cette architecture est fondamentale pour concevoir des systèmes MongoDB résilients, performants et adaptés aux exigences de production. Les sections suivantes approfondiront les aspects opérationnels : types de membres, mécanismes d'élection, gestion de l'Oplog, et pratiques de maintenance.

⏭️ [Membres d'un Replica Set](/09-replication/03-membres-replica-set.md)
