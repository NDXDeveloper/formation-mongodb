🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.3 Membres d'un Replica Set

## Introduction

Un Replica Set MongoDB est composé de différents types de membres, chacun jouant un rôle spécifique dans l'architecture distribuée. Cette diversité de rôles permet de construire des topologies sophistiquées répondant à des besoins variés : haute disponibilité, isolation des charges de travail, protection contre les erreurs humaines, optimisation géographique, ou encore réduction des coûts d'infrastructure.

Comprendre les caractéristiques, les responsabilités et les compromis associés à chaque type de membre est essentiel pour concevoir des architectures de réplication robustes et performantes en production.

## Classification des Membres

Les membres d'un Replica Set peuvent être classifiés selon plusieurs dimensions orthogonales qui déterminent leur comportement et leur rôle dans le cluster.

### Classification par Rôle dans la Réplication

#### Membres Porteurs de Données (Data-Bearing Members)

Ce sont les membres qui maintiennent une copie complète ou partielle du dataset :

**Primary** :
- Unique membre acceptant les opérations d'écriture
- Génère l'Oplog à partir des modifications apportées aux données
- Sert de source autoritaire pour la réplication
- Peut également servir les lectures (selon la Read Preference)

**Secondary** :
- Réplique les données depuis le Primary (ou un autre Secondary via chaining)
- Applique les opérations de l'Oplog de manière asynchrone
- Peut servir les lectures en fonction de la Read Preference
- Éligible à devenir Primary lors d'une élection (selon la priority)

#### Membres Sans Données (Non-Data Members)

**Arbiter** :
- Ne maintient aucune copie des données
- Participe uniquement aux élections en fournissant un vote
- Consomme des ressources minimales (CPU, RAM, disque)
- Ne peut jamais devenir Primary

### Classification par Participation aux Élections

#### Membres Votants

- Participent activement au processus d'élection du Primary
- Leur vote compte dans le calcul du quorum (majorité)
- **Limite** : Maximum 7 membres votants par Replica Set (contrainte du protocole Raft)
- Propriété : `votes: 1` (valeur par défaut)

#### Membres Non-Votants

- Ne participent pas aux élections
- Utiles au-delà de la limite de 7 votants
- Répliquent normalement les données
- Propriété : `votes: 0`

**Cas d'usage** :
- Replica Sets avec plus de 7 membres (ex: 10 membres dont 7 votants)
- Membres géographiquement distants dont la latence rendrait les élections lentes
- Membres temporaires (analytics, backup) dont la disponibilité est secondaire

### Classification par Éligibilité à Devenir Primary

#### Membres Éligibles

- Peuvent devenir Primary lors d'une élection
- Ont une `priority > 0`
- Doivent être des membres votants et porteurs de données
- Participent au pool de candidats lors du failover

#### Membres Non-Éligibles (Priority 0)

- Ne peuvent jamais devenir Primary
- Ont une `priority: 0`
- Peuvent quand même voter (sauf si `votes: 0` également)
- Utiles pour des rôles spécialisés (reporting, backup, analytics)

**Exemples** :
- Hidden members : `priority: 0, hidden: true`
- Delayed members : `priority: 0, hidden: true, slaveDelay: N`
- Membres avec hardware insuffisant pour être Primary

### Classification par Visibilité

#### Membres Visibles

- Apparaissent dans la liste des serveurs disponibles pour les applications
- Peuvent recevoir des requêtes selon la Read Preference
- Comportement par défaut de tous les membres
- Propriété : `hidden: false` (défaut)

#### Membres Cachés (Hidden)

- Invisibles aux applications clientes
- N'apparaissent pas dans les résultats de `isMaster` / `hello`
- Ne reçoivent jamais de requêtes de lecture (même avec Read Preference secondary)
- Doivent avoir `priority: 0`
- Propriété : `hidden: true`

**Cas d'usage** :
- Serveurs de reporting et analytics dédiés
- Backups continus sans impact sur les opérations
- Tests de nouvelles versions ou configurations

### Classification par Délai de Réplication

#### Membres en Temps Réel

- Appliquent les opérations de l'Oplog sans délai artificiel
- Minimisent le replication lag naturel
- Comportement par défaut
- Propriété : `slaveDelay: 0` (ou absent)

#### Membres Retardés (Delayed)

- Appliquent les opérations de l'Oplog avec un délai configurable
- Maintiennent un état historique des données
- Doivent être Hidden (`hidden: true`) et non-éligibles (`priority: 0`)
- Propriété : `slaveDelay: N` (en secondes)

**Cas d'usage** :
- Protection contre les suppressions accidentelles
- Point de restauration pour les erreurs humaines
- Compliance et audit trail

## Matrice des Combinaisons de Propriétés

Les différentes propriétés peuvent être combinées pour créer des profils de membres adaptés à des besoins spécifiques. Voici une matrice des combinaisons courantes :

| Type de Membre | Priority | Votes | Hidden | SlaveDelay | Data-Bearing | Cas d'Usage |
|----------------|----------|-------|--------|------------|--------------|-------------|
| **Primary** | > 0 | 1 | false | 0 | ✓ | Membre principal, écritures |
| **Secondary Standard** | 1.0 (défaut) | 1 | false | 0 | ✓ | Réplication standard, failover |
| **Secondary Prioritaire** | > 1 | 1 | false | 0 | ✓ | Candidat préféré pour Primary |
| **Secondary Non-Éligible** | 0 | 1 | false | 0 | ✓ | Lecture seule, pas de failover |
| **Hidden** | 0 | 1 | true | 0 | ✓ | Reporting, analytics, backup |
| **Delayed** | 0 | 0-1 | true | > 0 | ✓ | Protection erreurs humaines |
| **Arbiter** | N/A | 1 | N/A | N/A | ✗ | Vote uniquement, tie-breaker |
| **Non-Voting Secondary** | ≥ 0 | 0 | false | 0 | ✓ | Au-delà de 7 membres |
| **Analytics Node** | 0 | 0 | true | 0 | ✓ | ETL, BI, pas de vote |

### Contraintes de Cohérence

Certaines combinaisons de propriétés ont des dépendances obligatoires :

1. **Hidden ⇒ Priority: 0** : Un membre caché ne peut pas devenir Primary
2. **SlaveDelay > 0 ⇒ Hidden: true** : Un membre retardé doit être caché
3. **SlaveDelay > 0 ⇒ Priority: 0** : Un membre retardé ne peut pas devenir Primary
4. **Arbiter ⇒ Votes: 1** : Un arbiter doit voter (sinon il est inutile)
5. **Priority > 0 ⇒ Votes: 1** : Un membre éligible doit pouvoir voter

Ces contraintes sont validées par MongoDB lors de la reconfiguration du Replica Set.

## Propriétés de Configuration des Membres

Chaque membre d'un Replica Set est défini par un ensemble de propriétés dans la configuration du Replica Set. Examinons les propriétés fondamentales :

### Propriétés d'Identité

#### _id (Member ID)

```javascript
{
  _id: 0,  // Identifiant unique du membre (0 à 255)
  ...
}
```

- **Type** : Entier (0-255)
- **Unicité** : Doit être unique au sein du Replica Set
- **Immutabilité** : Ne peut pas être modifié après la création
- **Utilisation** : Référence stable pour la reconfiguration, les logs, le monitoring

**Bonnes pratiques** :
- Utiliser des IDs séquentiels (0, 1, 2, ...) pour faciliter la lecture
- Documenter la correspondance ID ↔ Hostname dans la gestion de configuration
- Ne jamais réutiliser un ID après la suppression d'un membre (éviter la confusion)

#### host (Hostname)

```javascript
{
  _id: 0,
  host: "mongo1.example.com:27017",
  ...
}
```

- **Format** : `hostname:port` ou `IP:port`
- **Résolution** : Doit être résolvable DNS depuis tous les autres membres
- **Port** : Par défaut 27017, peut être personnalisé
- **Modification** : Possible mais nécessite une reconfiguration

**Considérations** :
- Préférer les hostnames aux IPs (flexibilité DNS)
- Utiliser des FQDN (Fully Qualified Domain Names) pour éviter les ambiguïtés
- Attention aux changements d'IP si des IPs sont utilisées
- Le hostname doit correspondre au certificat TLS/SSL (si activé)

### Propriétés de Vote et Élection

#### priority (Priorité d'Élection)

```javascript
{
  _id: 0,
  priority: 1.0,  // Valeur par défaut
  ...
}
```

- **Plage** : 0 à 1000 (décimal)
- **Défaut** : 1.0
- **Sémantique** :
  - `priority: 0` → Jamais Primary (non-éligible)
  - `priority > 0` → Éligible, valeur plus élevée = préféré lors des élections
  - `priority: 1000` → Toujours préféré (sauf indisponibilité ou données obsolètes)

**Impact sur les élections** :
- Les membres avec `priority` élevée initient les élections plus rapidement
- Les autres membres préfèrent voter pour les candidats à priorité élevée
- En cas de split vote, les priorités départagent les candidats
- Un membre avec `priority: 0` vote mais ne se présente jamais comme candidat

**Cas d'usage** :
```javascript
// Configuration multi-datacenter avec préférence
cfg.members[0].priority = 2    // DC principal (préféré)
cfg.members[1].priority = 2    // DC principal (préféré)
cfg.members[2].priority = 1    // DC secondaire (normal)
cfg.members[3].priority = 0.5  // DC distant (backup uniquement)
cfg.members[4].priority = 0    // Hidden member (jamais Primary)
```

#### votes (Droit de Vote)

```javascript
{
  _id: 0,
  votes: 1,  // Valeur par défaut
  ...
}
```

- **Valeurs** : `0` ou `1` uniquement
- **Défaut** : 1 (votant)
- **Contrainte** : Maximum 7 membres avec `votes: 1` par Replica Set

**Quand utiliser `votes: 0`** :
- Replica Set avec plus de 7 membres (les membres supplémentaires doivent avoir `votes: 0`)
- Membres temporaires ou de test qui ne doivent pas influencer les élections
- Delayed members qui ne doivent pas voter (optionnel mais recommandé)
- Membres géographiquement très distants (latence réseau élevée)

**Implications** :
- Le quorum est calculé uniquement sur les membres votants
- Un membre non-votant peut quand même devenir Primary si `priority > 0`
- Réduire le nombre de votants accélère les élections mais réduit la tolérance aux pannes

### Propriétés de Visibilité et Comportement

#### hidden (Membre Caché)

```javascript
{
  _id: 0,
  priority: 0,    // Obligatoire si hidden: true
  hidden: true,
  ...
}
```

- **Valeurs** : `true` ou `false` (défaut)
- **Contrainte** : Nécessite `priority: 0`
- **Effet** : Le membre n'apparaît pas dans `isMaster` / `hello`, invisible aux clients

**Isolation des charges** :
```javascript
// Membre dédié au reporting
{
  _id: 3,
  host: "analytics.example.com:27017",
  priority: 0,
  hidden: true,
  votes: 0,  // Optionnel, mais recommandé
  tags: { usage: "analytics" }
}
```

#### slaveDelay (Délai de Réplication)

```javascript
{
  _id: 0,
  priority: 0,    // Obligatoire
  hidden: true,   // Obligatoire
  slaveDelay: 3600,  // 1 heure en secondes
  ...
}
```

- **Type** : Entier (secondes)
- **Plage** : 0 à 2^31 - 1 (environ 68 ans, mais valeurs pratiques : 0 à 86400)
- **Contraintes** : Nécessite `priority: 0` et `hidden: true`

**Mécanisme** :
- Le membre applique les opérations de l'Oplog avec un délai fixe
- Maintient un "snapshot" historique des données
- Le délai est basé sur le timestamp de l'opération, pas le temps d'arrivée

**Exemple** : Protection contre suppression accidentelle
```javascript
// Delayed member avec 4 heures de délai
{
  _id: 4,
  host: "delayed.example.com:27017",
  priority: 0,
  hidden: true,
  slaveDelay: 14400,  // 4 heures
  votes: 0,
  tags: { usage: "delayed_backup" }
}
```

Si une table est supprimée par erreur à 14h00, les données existent toujours sur le delayed member jusqu'à 18h00, permettant une récupération.

### Propriétés de Taggage

#### tags (Étiquettes de Membre)

```javascript
{
  _id: 0,
  tags: {
    dc: "east",
    rack: "A1",
    usage: "production",
    ssd: "true"
  },
  ...
}
```

- **Type** : Objet (clés-valeurs sous forme de chaînes)
- **Utilisation** : Routage des requêtes, configuration de Write Concern, organisation logique

**Cas d'usage multiples** :

1. **Localisation géographique** :
```javascript
tags: { region: "us-east-1", az: "us-east-1a" }
```

2. **Type de hardware** :
```javascript
tags: { storage: "ssd", ram: "128gb", cpu: "high" }
```

3. **Rôle fonctionnel** :
```javascript
tags: { usage: "production", workload: "oltp" }
tags: { usage: "analytics", workload: "olap" }
```

4. **Read Preference avec tags** :
```javascript
// Application cliente
db.collection.find().readPref("secondary", [
  { dc: "east", usage: "analytics" },  // Préférence 1
  { dc: "east" }                       // Fallback 1
])
```

5. **Write Concern avec tags** :
```javascript
// Garantir réplication dans deux datacenters
db.collection.insertOne(
  { ... },
  {
    writeConcern: {
      w: {
        "multiDC": 2  // Tag set custom défini dans la config
      }
    }
  }
)
```

**Configuration des tag sets pour Write Concern** :
```javascript
cfg = rs.conf()
cfg.settings = {
  getLastErrorModes: {
    multiDC: { dc: 2 },           // Au moins 2 DCs différents
    multiRack: { rack: 3 },       // Au moins 3 racks différents
    ssdReplica: { ssd: "true": 2 } // Au moins 2 membres SSD
  }
}
rs.reconfig(cfg)
```

### Propriétés Avancées

#### buildIndexes

```javascript
{
  _id: 0,
  priority: 0,     // Obligatoire si buildIndexes: false
  buildIndexes: false,
  ...
}
```

- **Valeurs** : `true` (défaut) ou `false`
- **Contrainte** : Si `false`, nécessite `priority: 0`
- **Effet** : Le membre ne construit pas d'index (sauf `_id`)

**Cas d'usage très limités** :
- Backup de données brutes sans index (gain d'espace disque)
- Membres temporaires pour export/archivage

**Attention** : Un membre avec `buildIndexes: false` ne peut **jamais** être reconfiguré avec `buildIndexes: true` sans réinitialisation complète.

#### horizons (DNS Horizons - MongoDB 4.2+)

```javascript
{
  _id: 0,
  host: "internal-mongo1.local:27017",
  horizons: {
    external: "mongo1.example.com:27017",
    vpc: "10.0.1.10:27017"
  },
  ...
}
```

- **Utilisation** : Environnements avec plusieurs réseaux (public/privé, VPC/Internet)
- **Mécanisme** : Les clients reçoivent le hostname correspondant à leur horizon
- **Cas d'usage** : Clusters hybrides cloud/on-premise, multi-VPC, NAT traversal

**Exemple** : Cluster accessible depuis Internet et VPC privé
```javascript
// Membre 1
{
  _id: 0,
  host: "10.0.1.10:27017",  // Hostname interne
  horizons: {
    public: "mongo1.example.com:27017",      // Accès public
    private: "mongo1.internal.vpc:27017"     // Accès VPC
  }
}

// Connexion client avec horizon
mongodb://mongo1.example.com/?replicaSet=rs0&horizon=public
```

## Dynamiques d'Interaction entre Membres

### Graphe de Réplication (Replication Graph)

Les membres d'un Replica Set forment un graphe orienté de réplication où :

- **Nœuds** : Membres du Replica Set
- **Arêtes** : Relations de réplication (source → destination)

#### Configuration par Défaut (Étoile)

```
         [Primary]
        /    |    \
       /     |     \
      /      |      \
  [Sec1]  [Sec2]  [Sec3]
```

Tous les Secondaries répliquent directement depuis le Primary.

**Avantages** :
- Latence de réplication minimale
- Simplicité de diagnostic
- Cohérence maximale entre Secondaries

**Inconvénients** :
- Charge réseau concentrée sur le Primary
- Bande passante du Primary limitante pour le nombre de Secondaries

#### Replication Chaining

```
    [Primary]
       |
    [Sec1] -----> [Sec2]
       |
    [Sec3]
```

Des Secondaries répliquent depuis d'autres Secondaries.

**Activation** (activé par défaut) :
```javascript
cfg = rs.conf()
cfg.settings.chainingAllowed = true
rs.reconfig(cfg)
```

**Sélection de la source** :
MongoDB sélectionne automatiquement la meilleure source de réplication basée sur :
- Proximité réseau (ping time)
- Fraîcheur des données (OpTime)
- Charge du nœud source

**Cas d'usage multi-datacenter** :
```
    DC-A                DC-B               DC-C
  [Primary] --------> [Sec1] --------> [Sec2]
     |
  [Sec3]
```

Sec1 réplique depuis Primary (DC-A → DC-B), Sec2 réplique depuis Sec1 (évite DC-A → DC-C).

**Désactivation du chaining** :
```javascript
cfg.settings.chainingAllowed = false
rs.reconfig(cfg)
```
Force tous les Secondaries à répliquer directement depuis le Primary.

### Équilibrage de Charge en Lecture

Les membres peuvent être sollicités pour servir les lectures selon plusieurs stratégies :

#### Read Preference Modes

1. **primary** : Uniquement le Primary (cohérence maximale)
2. **primaryPreferred** : Primary si disponible, sinon Secondary
3. **secondary** : Uniquement les Secondaries (décharge le Primary)
4. **secondaryPreferred** : Secondary si disponible, sinon Primary
5. **nearest** : Membre le plus proche (latence minimale)

#### Combinaison avec Tags

Isolation des charges de travail par tags :

```javascript
// Configuration
cfg.members[0].tags = { dc: "east", workload: "oltp" }
cfg.members[1].tags = { dc: "east", workload: "oltp" }
cfg.members[2].tags = { dc: "east", workload: "olap" }

// Application OLTP
db.getMongo().setReadPref("secondary", [
  { dc: "east", workload: "oltp" }
])

// Application Analytics
db.getMongo().setReadPref("secondary", [
  { dc: "east", workload: "olap" }
])
```

Garantit que les requêtes analytiques lourdes ne perturbent pas les opérations transactionnelles.

### Scénarios de Reconfiguration Dynamique

Les membres peuvent être ajoutés, supprimés ou reconfigurés dynamiquement :

#### Ajout d'un Membre

```javascript
cfg = rs.conf()
cfg.members.push({
  _id: 4,
  host: "mongo5.example.com:27017",
  priority: 1,
  votes: 1
})
rs.reconfig(cfg)
```

**Processus** :
1. Le nouveau membre effectue un Initial Sync complet
2. Reste en état STARTUP2 pendant la synchronisation
3. Passe à SECONDARY une fois à jour
4. Commence à participer aux élections

#### Suppression d'un Membre

```javascript
cfg = rs.conf()
cfg.members = cfg.members.filter(m => m._id !== 4)
rs.reconfig(cfg)
```

**Attention** : Si le membre supprimé est le Primary, une élection est déclenchée.

#### Modification des Propriétés

```javascript
cfg = rs.conf()
cfg.members[2].priority = 0  // Rendre non-éligible
cfg.members[2].hidden = true  // Cacher
cfg.members[2].tags = { usage: "backup" }
rs.reconfig(cfg)
```

Permet de transformer un Secondary standard en Hidden member sans réinitialisation.

## Considérations de Conception

### Dimensionnement du Nombre de Membres

**Règle générale** :
- **Production standard** : 3 membres (1 Primary + 2 Secondaries)
- **Haute criticité** : 5 membres (1 Primary + 4 Secondaries)
- **Multi-datacenter** : 5 membres minimum pour tolérer la perte d'un datacenter

**Compromis** :

**Avec 3 membres** :
- Quorum : 2/3
- Tolérance : Perte de 1 membre
- Coût : Modéré (3 serveurs)
- Complexité : Faible

**Avec 5 membres** :
- Quorum : 3/5
- Tolérance : Perte de 2 membres
- Coût : Élevé (5 serveurs)
- Complexité : Modérée

**Avec 7 membres (limite de votants)** :
- Quorum : 4/7
- Tolérance : Perte de 3 membres
- Coût : Très élevé
- Complexité : Élevée (considérer le sharding plutôt)

**Pourquoi pas un nombre pair ?**
- **Quorum identique** : 4 membres ont le même quorum que 3 (2/4 vs 2/3)
- **Coût sans bénéfice** : Aucune amélioration de la tolérance aux pannes
- **Élections potentiellement bloquées** : Risque de split vote accru

**Exception** : 4 membres avec 1 Arbiter (2 Secondaries + 1 Primary + 1 Arbiter), mais déconseillé (préférer 3 Secondaries complets).

### Optimisation Géographique

**Principe** : Placer les membres en fonction de la localisation des utilisateurs et des datacenters.

**Scénario 1 : Application mono-région avec DR**
```
  Région Principale (us-east-1)      Région Backup (us-west-2)
  [Primary, priority: 2]             [Secondary, priority: 0.5]
  [Secondary, priority: 2]
  [Secondary, priority: 1]
```

**Scénario 2 : Application multi-région avec utilisateurs distribués**
```
  US-East              US-West              EU
  [Primary, p: 2]      [Sec, p: 1]         [Sec, p: 1]
  [Sec, p: 1]          [Sec, p: 1]
```

Read Preference `nearest` pour minimiser la latence globale.

**Scénario 3 : Multi-datacenter avec quorum distribué**
```
  DC1 (2 membres)    DC2 (2 membres)    DC3 (1 membre, cloud)
  [Primary, p: 2]    [Sec, p: 1]        [Sec, p: 1]
  [Sec, p: 2]        [Sec, p: 1]
```

Permet de survivre à la perte complète d'un datacenter (quorum = 3/5).

### Isolation des Charges de Travail

**Stratégie** : Utiliser des membres spécialisés avec tags pour isoler différents types de requêtes.

**Architecture type** :
```javascript
// Membres OLTP (transactions)
members[0]: { tags: { workload: "oltp", dc: "east" } }
members[1]: { tags: { workload: "oltp", dc: "east" } }

// Membre OLAP (analytics)
members[2]: {
  priority: 0,
  hidden: true,
  tags: { workload: "olap", dc: "east" }
}

// Membre Backup
members[3]: {
  priority: 0,
  hidden: true,
  tags: { workload: "backup", dc: "west" }
}

// Membre Delayed (protection)
members[4]: {
  priority: 0,
  hidden: true,
  slaveDelay: 3600,
  tags: { workload: "delayed", dc: "west" }
}
```

**Routage applicatif** :
```javascript
// Application web (OLTP)
mongo.setReadPref("primaryPreferred", [{ workload: "oltp" }])

// Pipeline ETL (OLAP)
mongo.setReadPref("secondary", [{ workload: "olap" }])
```

## Surveillance et Monitoring des Membres

### Métriques Essentielles par Membre

**État du membre** :
```javascript
rs.status().members.forEach(m => {
  print(`${m.name}: ${m.stateStr} (lag: ${m.optimeDate})`)
})
```

**Informations clés** :
- `stateStr` : PRIMARY, SECONDARY, RECOVERING, STARTUP, etc.
- `health` : 0 (down) ou 1 (up)
- `optime` : Position dans l'Oplog
- `optimeDate` : Timestamp de la dernière opération appliquée
- `lastHeartbeat` : Date du dernier heartbeat reçu
- `pingMs` : Latence réseau vers ce membre

**Replication lag** :
```javascript
function getReplicationLag() {
  const status = rs.status()
  const primary = status.members.find(m => m.state === 1)

  status.members
    .filter(m => m.state === 2)
    .forEach(m => {
      const lag = (primary.optimeDate - m.optimeDate) / 1000
      print(`${m.name}: ${lag.toFixed(2)}s behind primary`)
    })
}
```

### Alerting sur Anomalies

**Conditions d'alerte critiques** :

1. **Membre DOWN** : `health: 0` pendant > 30 secondes
2. **Replication lag élevé** : Lag > 60 secondes (ajuster selon le contexte)
3. **Oplog window insuffisant** : < 6 heures de couverture
4. **Élections fréquentes** : Plus de 2 élections/heure
5. **Membre RECOVERING** : État prolongé (> 15 minutes)
6. **Heartbeat timeout** : Latence > 5 secondes

**Intégration monitoring** :
- MongoDB Cloud Manager / Ops Manager
- Prometheus + MongoDB Exporter
- Datadog / New Relic
- Scripts custom avec `rs.status()`

## Limitations et Contraintes

### Contraintes Numériques

- **Maximum 50 membres** par Replica Set (limite technique)
- **Maximum 7 membres votants** (contrainte du protocole de consensus)
- **Maximum 1 Arbiter** recommandé (plus est inutile et déconseillé)
- **Maximum ~500 tags** par membre (limite pratique, pas stricte)

### Contraintes de Configuration

- **Hidden ⇒ Priority 0** : Dépendance obligatoire
- **SlaveDelay ⇒ Hidden + Priority 0** : Contraintes en cascade
- **Arbiter** : Ne peut avoir aucune autre propriété spéciale
- **BuildIndexes: false** : Irréversible sans resync complet

### Contraintes Opérationnelles

- **Reconfiguration** : Nécessite un Primary disponible (sauf `force: true`)
- **Ajout de membre** : Déclenche un Initial Sync (coûteux)
- **Modification de votes** : Impact immédiat sur le quorum (risqué)
- **Modification de priority** : Peut déclencher une élection

## Conclusion

Les membres d'un Replica Set MongoDB offrent une flexibilité architecturale considérable grâce à leurs propriétés configurables. La compréhension approfondie de chaque type de membre et de leurs interactions est essentielle pour :

1. **Concevoir** des topologies adaptées aux besoins métier (HA, DR, performance)
2. **Optimiser** les compromis entre cohérence, disponibilité et coût
3. **Isoler** les charges de travail pour éviter les interférences
4. **Protéger** contre les erreurs humaines et les pannes matérielles
5. **Distribuer** géographiquement pour réduire la latence et améliorer la résilience

**Points clés** :

- Les **Primary et Secondary** forment le cœur du système de réplication
- Les **Arbiters** sont utiles mais déconseillés en production (préférer des Secondaries complets)
- Les **Hidden members** permettent l'isolation des charges sans impact sur les opérations
- Les **Delayed members** offrent une protection contre les erreurs humaines
- Les **tags** permettent un routage intelligent et une organisation logique du cluster
- Les **priorités** contrôlent l'éligibilité et la préférence lors des élections

Les sections suivantes détailleront chaque type de membre spécifique, leurs mécanismes internes, et les bonnes pratiques opérationnelles associées.

⏭️ [Primary](/09-replication/03.1-primary.md)
