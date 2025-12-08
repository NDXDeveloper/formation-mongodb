🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.4 Élection du Primary

## Introduction

L'élection du Primary est un mécanisme critique dans l'architecture Replica Set de MongoDB qui garantit la haute disponibilité et la cohérence des données. Ce processus automatisé permet au cluster de maintenir ses opérations d'écriture même en cas de défaillance du nœud primaire actuel.

## Concepts Fondamentaux

### Quorum et Majorité

L'élection du Primary repose sur le principe de **majorité simple** :

```
Majorité = floor(nombre_total_membres / 2) + 1
```

**Exemples** :
- Replica Set de 3 membres : majorité = 2
- Replica Set de 5 membres : majorité = 3
- Replica Set de 7 membres : majorité = 4

Cette exigence de majorité garantit qu'il ne peut y avoir qu'un seul Primary à la fois, évitant ainsi le problème du "split-brain" dans les systèmes distribués.

### États des Membres

Durant le processus électoral, les membres peuvent se trouver dans différents états :

| État | Description |
|------|-------------|
| `PRIMARY` | Membre élu acceptant les écritures |
| `SECONDARY` | Membre répliquant les données du Primary |
| `ARBITER` | Membre participant aux votes uniquement |
| `RECOVERING` | Membre en cours de synchronisation |
| `STARTUP` | Membre en phase de démarrage |
| `STARTUP2` | Membre chargeant la configuration du Replica Set |
| `ROLLBACK` | Membre annulant des opérations non répliquées |
| `REMOVED` | Membre retiré du Replica Set |
| `DOWN` | Membre inaccessible |

## Protocole d'Élection : Raft

Depuis MongoDB 4.0, le protocole d'élection utilise une implémentation du **protocole Raft** (PV1 - Protocol Version 1).

### Termes et Concepts Raft

#### Term (Mandat)

Le **term** est un compteur monotone qui s'incrémente à chaque nouvelle élection :

```javascript
{
  "term": NumberLong(42),
  "lastCommittedOpTime": {
    "ts": Timestamp(1638360000, 1),
    "t": NumberLong(42)
  }
}
```

Chaque term peut avoir au plus un Primary. Un term sans Primary indique une élection ayant échoué.

#### OpTime

L'**OpTime** identifie de manière unique chaque opération dans l'oplog :

```javascript
{
  "ts": Timestamp(1638360000, 5),  // Timestamp de l'opération
  "t": NumberLong(42)               // Term associé
}
```

Le membre avec l'OpTime le plus récent a les données les plus à jour.

### Phases du Protocole Raft

#### 1. Heartbeats (Battements de cœur)

Le Primary envoie périodiquement des heartbeats aux membres secondaires :

- **Intervalle par défaut** : 2 secondes
- **Timeout** : 10 secondes (`electionTimeoutMillis`)

Si un Secondary ne reçoit pas de heartbeat pendant le timeout, il initie une élection.

#### 2. Initiation d'une Élection

Lorsqu'un membre détecte l'absence du Primary, il :

1. Incrémente son term local
2. Passe à l'état `CANDIDATE`
3. Vote pour lui-même
4. Envoie des requêtes de vote (`RequestVote`) aux autres membres

#### 3. Requêtes de Vote

La requête `RequestVote` contient :

```javascript
{
  "term": NumberLong(43),
  "candidateId": "mongodb-02:27017",
  "lastCommittedOpTime": {
    "ts": Timestamp(1638360000, 100),
    "t": NumberLong(42)
  },
  "lastAppliedOpTime": {
    "ts": Timestamp(1638360000, 105),
    "t": NumberLong(42)
  }
}
```

#### 4. Décision de Vote

Un membre accorde son vote si :

- ✅ Le term du candidat est supérieur ou égal à son term local
- ✅ Il n'a pas déjà voté pour un autre candidat dans ce term
- ✅ L'OpTime du candidat est au moins aussi récent que le sien
- ✅ Les contraintes de priorité sont respectées

#### 5. Élection du Primary

Un candidat devient Primary s'il obtient :

- La **majorité des votes** du Replica Set
- Confirmation que son OpTime est suffisamment à jour

## Priorités et Configuration

### Priority (Priorité)

La priorité détermine la préférence d'un membre à devenir Primary :

```javascript
{
  "_id": "rs0",
  "members": [
    { "_id": 0, "host": "mongodb-01:27017", "priority": 2 },  // Préféré
    { "_id": 1, "host": "mongodb-02:27017", "priority": 1 },  // Standard
    { "_id": 2, "host": "mongodb-03:27017", "priority": 0.5 } // Moins prioritaire
  ]
}
```

**Règles** :
- Valeur par défaut : `1`
- Plage : `0` à `1000`
- `priority: 0` → Le membre ne peut jamais devenir Primary (membre passif)
- Plus la priorité est élevée, plus le membre a de chances d'être élu

### Votes

Chaque membre peut avoir 0 ou 1 vote :

```javascript
{
  "_id": "rs0",
  "members": [
    { "_id": 0, "host": "mongodb-01:27017", "votes": 1 },
    { "_id": 1, "host": "mongodb-02:27017", "votes": 1 },
    { "_id": 2, "host": "mongodb-03:27017", "votes": 0 }  // Non-votant
  ]
}
```

**Contraintes** :
- Maximum 7 membres votants par Replica Set
- Un membre avec `votes: 0` ne participe pas aux élections
- Souvent utilisé pour les membres géographiquement distants

### Priorité 0 + Votes 0

Configuration pour un membre en lecture seule (analytics, reporting) :

```javascript
{
  "_id": 3,
  "host": "mongodb-analytics:27017",
  "priority": 0,
  "votes": 0,
  "hidden": true  // Caché des applications
}
```

## Scénarios d'Élection

### Scénario 1 : Défaillance du Primary

**Situation initiale** :
```
[PRIMARY] mongodb-01 (term: 42)
[SECONDARY] mongodb-02
[SECONDARY] mongodb-03
```

**Séquence d'événements** :

1. **T0** : mongodb-01 devient inaccessible
2. **T0 + 10s** : mongodb-02 et mongodb-03 détectent le timeout
3. **T0 + 10s** : mongodb-02 initie une élection (term: 43)
4. **T0 + 10.5s** : mongodb-02 envoie `RequestVote` à mongodb-03
5. **T0 + 11s** : mongodb-03 vote pour mongodb-02
6. **T0 + 11.5s** : mongodb-02 obtient la majorité (2/3) et devient Primary

**Résultat** :
```
[DOWN] mongodb-01
[PRIMARY] mongodb-02 (term: 43)
[SECONDARY] mongodb-03
```

**Temps total** : ~1-2 secondes après détection de la défaillance

### Scénario 2 : Partition Réseau (Split-Brain Prevention)

**Situation initiale** :
```
DC1: [PRIMARY] mongodb-01, [SECONDARY] mongodb-02
DC2: [SECONDARY] mongodb-03
```

**Partition réseau** : Séparation entre DC1 et DC2

**Partition A (DC1)** :
- mongodb-01 et mongodb-02 peuvent communiquer
- Majorité : 2/3 ✅
- mongodb-01 reste PRIMARY

**Partition B (DC2)** :
- mongodb-03 isolé
- Pas de majorité : 1/3 ❌
- mongodb-03 devient SECONDARY en lecture seule

**Résultat** :
```
DC1: [PRIMARY] mongodb-01, [SECONDARY] mongodb-02  → Opérations d'écriture OK
DC2: [SECONDARY] mongodb-03                         → Lecture seule
```

Le système évite le split-brain car seule la partition avec majorité peut élire un Primary.

### Scénario 3 : Élection avec Priorités

**Configuration** :
```javascript
{
  "members": [
    { "_id": 0, "host": "mongodb-01:27017", "priority": 10 },  // Préféré
    { "_id": 1, "host": "mongodb-02:27017", "priority": 5 },
    { "_id": 2, "host": "mongodb-03:27017", "priority": 1 }
  ]
}
```

**Si mongodb-02 est initialement Primary** :

1. mongodb-01 redémarre après maintenance
2. mongodb-01 constate qu'il a une priorité supérieure
3. Après 10 secondes (`priorityTakeoverDelayMillis`), mongodb-01 déclenche une élection
4. mongodb-01 devient Primary car il a la priorité la plus élevée

**Mécanisme** : **Priority Takeover Election**

### Scénario 4 : Catchup Phase

Lorsqu'un nouveau Primary est élu, il peut entrer en **catchup phase** :

```
[SECONDARY] mongodb-02 (dernier optime: t=42, ts=100)
        ↓ Élu Primary
[PRIMARY (CATCHUP)] mongodb-02
        ↓ Rattrapage des opérations
[PRIMARY] mongodb-02 (dernier optime: t=43, ts=150)
```

**Paramètre** :
```javascript
cfg.settings.catchUpTimeoutMillis = 30000  // 30 secondes par défaut
```

Durant la catchup phase :
- Le Primary ne peut pas encore accepter d'écritures
- Il réplique les opérations manquantes depuis les autres membres
- Évite la perte de données récemment écrites

## Timeouts et Paramètres de Configuration

### electionTimeoutMillis

Durée avant qu'un Secondary initie une élection :

```javascript
cfg = rs.conf()
cfg.settings = cfg.settings || {}
cfg.settings.electionTimeoutMillis = 10000  // 10 secondes (défaut)
rs.reconfig(cfg)
```

**Recommandations** :
- **Réseaux rapides** : 5000-10000 ms
- **Réseaux lents/WAN** : 15000-30000 ms
- Trop court → élections fréquentes et instabilité
- Trop long → disponibilité réduite lors de défaillances

### heartbeatIntervalMillis

Intervalle entre les heartbeats :

```javascript
cfg.settings.heartbeatIntervalMillis = 2000  // 2 secondes (défaut)
```

**Note** : Ce paramètre n'est généralement pas modifiable dans les versions récentes.

### catchUpTimeoutMillis

Durée maximale de la catchup phase :

```javascript
cfg.settings.catchUpTimeoutMillis = -1  // Désactivé
cfg.settings.catchUpTimeoutMillis = 0   // Pas de catchup
cfg.settings.catchUpTimeoutMillis = 30000  // 30 secondes (défaut)
```

### catchUpTakeoverDelayMillis

Délai avant qu'un Secondary avec priorité plus élevée déclenche une élection :

```javascript
cfg.settings.catchUpTakeoverDelayMillis = 30000  // 30 secondes (défaut)
```

## Cas Particuliers et Edge Cases

### Élection Impossible (Pas de Majorité)

**Replica Set de 2 membres** :
```
[PRIMARY] mongodb-01
[SECONDARY] mongodb-02
```

Si mongodb-01 tombe :
- mongodb-02 ne peut pas obtenir la majorité (1/2)
- Pas de nouveau Primary élu
- Le cluster est en **lecture seule**

**Solution** : Toujours avoir un nombre impair de membres (3, 5, 7) ou utiliser un arbiter.

### Arbiter pour Résoudre les Votes

```javascript
{
  "members": [
    { "_id": 0, "host": "mongodb-01:27017" },
    { "_id": 1, "host": "mongodb-02:27017" },
    { "_id": 2, "host": "arbiter:27017", "arbiterOnly": true }
  ]
}
```

L'arbiter :
- Ne stocke aucune donnée
- Participe uniquement aux votes
- Permet d'obtenir un nombre impair de votants
- Ressources minimales requises

**Attention** : MongoDB recommande d'utiliser un vrai membre de données plutôt qu'un arbiter lorsque possible.

### Rollback Automatique

Si un ancien Primary redémarre avec des opérations non répliquées :

```
Ancien Primary (mongodb-01) :
  Oplog: [op1, op2, op3, op4, op5]  // op5 non répliquée

Nouveau Primary (mongodb-02) :
  Oplog: [op1, op2, op3, op4, op6]  // Nouvelle branche

Résultat pour mongodb-01 :
  1. Détection de la divergence
  2. État ROLLBACK
  3. Annulation de op5 (sauvegarde dans rollback/)
  4. Application de op6
  5. État SECONDARY
```

**Fichiers de rollback** :
```bash
/data/db/rollback/
  ├── 2024-01-15T10-30-00.0.bson
  └── 2024-01-15T10-30-00.0.metadata.json
```

### Write Concern et Élection

Utilisation de `w: "majority"` pour éviter les rollbacks :

```javascript
db.orders.insertOne(
  { orderId: 12345, amount: 500 },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)
```

Garantit que l'écriture est répliquée sur la majorité avant de confirmer, rendant le rollback quasiment impossible.

## Monitoring des Élections

### Commandes de Diagnostic

#### rs.status()

```javascript
rs.status()
```

Informations clés :
```javascript
{
  "set": "rs0",
  "myState": 1,  // 1 = PRIMARY, 2 = SECONDARY
  "term": NumberLong(43),
  "members": [
    {
      "name": "mongodb-01:27017",
      "state": 1,
      "stateStr": "PRIMARY",
      "electionTime": Timestamp(1638360000, 1),
      "electionDate": ISODate("2024-01-15T10:00:00Z")
    }
  ]
}
```

#### rs.isMaster()

```javascript
rs.isMaster()
```

Retourne :
```javascript
{
  "ismaster": true,
  "primary": "mongodb-01:27017",
  "setName": "rs0",
  "electionId": ObjectId("7fffffff000000000000002b")
}
```

### Logs d'Élection

Recherche dans les logs MongoDB :

```bash
grep "election" /var/log/mongodb/mongod.log
```

Exemple de log :
```
2024-01-15T10:00:00.000+0000 I REPL     [replexec-0] Starting an election, since we've seen no PRIMARY in the past 10000ms
2024-01-15T10:00:00.100+0000 I REPL     [replexec-0] conducting a dry run election to see if we could be elected
2024-01-15T10:00:00.200+0000 I REPL     [replexec-0] dry election run succeeded, running for election in term 43
2024-01-15T10:00:01.000+0000 I REPL     [replexec-0] election succeeded, assuming primary role in term 43
```

### Métriques Importantes

Métriques à surveiller :

| Métrique | Description | Alerte si |
|----------|-------------|-----------|
| `electionTime` | Timestamp de la dernière élection | Changements fréquents |
| `term` | Numéro du mandat actuel | Incrémentation rapide |
| `pingMs` | Latence réseau entre membres | > 100ms |
| `optime.ts` | Timestamp du dernier oplog | Lag important |

## Optimisation et Bonnes Pratiques

### 1. Topologie Géographique

**Distribution recommandée** pour 5 membres :

```
DC Principal (Région A) :
  - mongodb-01 (priority: 10)
  - mongodb-02 (priority: 5)
  - mongodb-03 (priority: 5)

DC Secondaire (Région B) :
  - mongodb-04 (priority: 1)
  - mongodb-05 (priority: 1)
```

Avantages :
- Majorité dans le DC principal
- Basculement rapide en cas de défaillance locale
- Protection contre la perte du DC principal

### 2. Configuration des Priorités

Stratégie de priorités :

```javascript
{
  "members": [
    // Serveurs haute performance
    { "_id": 0, "host": "ssd-server-01:27017", "priority": 10 },
    { "_id": 1, "host": "ssd-server-02:27017", "priority": 9 },

    // Serveurs standard
    { "_id": 2, "host": "standard-01:27017", "priority": 5 },

    // Serveurs analytics (ne deviennent jamais Primary)
    { "_id": 3, "host": "analytics-01:27017", "priority": 0 },
    { "_id": 4, "host": "backup-01:27017", "priority": 0 }
  ]
}
```

### 3. Éviter les Élections Fréquentes

**Causes courantes** :
- ❌ Réseau instable
- ❌ Ressources insuffisantes (CPU, mémoire)
- ❌ electionTimeoutMillis trop court
- ❌ Charge trop importante sur le Primary

**Solutions** :
- ✅ Augmenter `electionTimeoutMillis` sur réseaux WAN
- ✅ Monitoring proactif des ressources
- ✅ Répartition de charge avec Read Preference
- ✅ Hardware adéquat pour le Primary

### 4. Test de Failover

Scénario de test :

```bash
# 1. Vérifier l'état initial
mongosh --eval "rs.status()"

# 2. Simuler la défaillance du Primary
# (sur le nœud Primary)
mongosh --eval "db.adminCommand({shutdown: 1})"

# 3. Observer l'élection (sur un Secondary)
mongosh --eval "while(true) { print(new Date(), rs.isMaster().primary); sleep(1000); }"

# 4. Mesurer le temps de basculement
# Devrait être < 30 secondes pour une configuration standard
```

### 5. Write Concern pour la Cohérence

Configuration recommandée pour les applications critiques :

```javascript
// Niveau global (MongoDB 5.0+)
db.adminCommand({
  setDefaultRWConcern: 1,
  defaultWriteConcern: {
    w: "majority",
    wtimeout: 5000
  }
})

// Niveau collection
db.createCollection("criticalData", {
  writeConcern: { w: "majority" }
})
```

## Limitations et Considérations

### Limites Techniques

| Limite | Valeur | Impact |
|--------|--------|--------|
| Membres maximum par Replica Set | 50 | Mais seulement 7 votants |
| Membres votants maximum | 7 | Au-delà, utiliser `votes: 0` |
| Taille minimale recommandée | 3 membres | Pour avoir une majorité |
| Temps typique d'élection | 10-30 secondes | Dépend de la latence réseau |

### Considérations de Performance

L'élection peut impacter :

1. **Disponibilité en écriture** : Arrêt des écritures pendant 10-30 secondes
2. **Lectures** : Avec `readPreference: primary`, les lectures sont également bloquées
3. **Connexions applicatives** : Nécessitent une gestion du retry

**Mitigation** :
```javascript
// Configuration du driver avec retry automatique
const client = new MongoClient(uri, {
  retryWrites: true,
  retryReads: true,
  serverSelectionTimeoutMS: 30000
})
```

## Conclusion

L'élection du Primary dans MongoDB est un mécanisme sophistiqué basé sur le protocole Raft qui garantit :

- ✅ **Cohérence** : Un seul Primary à la fois
- ✅ **Disponibilité** : Basculement automatique en cas de défaillance
- ✅ **Tolérance aux pannes** : Fonctionne tant que la majorité est disponible
- ✅ **Prévention du split-brain** : Grâce à l'exigence de majorité

La compréhension approfondie de ce processus est essentielle pour :
- Concevoir des topologies Replica Set résilientes
- Diagnostiquer les problèmes de disponibilité
- Optimiser les configurations pour différents scénarios
- Garantir la haute disponibilité des applications critiques

---

**Points clés à retenir** :
1. La majorité (n/2 + 1) est indispensable pour élire un Primary
2. Les priorités influencent mais ne garantissent pas l'élection
3. Le protocole Raft (term + OpTime) garantit la cohérence
4. Un monitoring actif des élections est crucial en production
5. Toujours préférer un nombre impair de membres votants

⏭️ [Oplog (Operations Log)](/09-replication/05-oplog.md)
