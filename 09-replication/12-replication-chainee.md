🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 9.12 Réplication Chaînée

## Introduction

La **réplication chaînée** (chained replication) est un mécanisme dans MongoDB qui permet à un membre Secondary de répliquer les données depuis un autre Secondary plutôt que directement depuis le Primary. Ce concept est crucial pour optimiser la bande passante réseau et réduire la charge sur le Primary dans des architectures distribuées géographiquement ou à grande échelle.

## Concept et Fonctionnement

### Réplication Standard vs Chaînée

**Réplication Standard** :
```
┌──────────────────────────────────────────────┐
│         Réplication Standard                 │
│                                              │
│           ┌──────────┐                       │
│           │ PRIMARY  │                       │
│           └────┬─────┘                       │
│                │                             │
│      ┌─────────┴─────────┐                   │
│      │                   │                   │
│      ↓                   ↓                   │
│ ┌──────────┐       ┌──────────┐              │
│ │Secondary1│       │Secondary2│              │
│ └──────────┘       └──────────┘              │
│                                              │
│ Tous les Secondary répliquent depuis Primary │
└──────────────────────────────────────────────┘
```

**Réplication Chaînée** :
```
┌──────────────────────────────────────────────┐
│         Réplication Chaînée                  │
│                                              │
│           ┌──────────┐                       │
│           │ PRIMARY  │                       │
│           └────┬─────┘                       │
│                │                             │
│                ↓                             │
│          ┌──────────┐                        │
│          │Secondary1│                        │
│          └────┬─────┘                        │
│               │                              │
│               ↓                              │
│         ┌──────────┐                         │
│         │Secondary2│                         │
│         └──────────┘                         │
│                                              │
│ Secondary2 réplique depuis Secondary1        │
│ (qui réplique lui-même depuis Primary)       │
└──────────────────────────────────────────────┘
```

### Mécanisme de Sélection de la Source

MongoDB sélectionne automatiquement une source de synchronisation basée sur plusieurs critères :

```javascript
// Algorithme de sélection (simplifié)
function selectSyncSource() {
  const candidates = []

  // 1. Collecter les candidats potentiels
  for (const member of replicaSet.members) {
    if (member.state === PRIMARY || member.state === SECONDARY) {
      if (member.optime >= myOptime && member.health === 1) {
        candidates.push(member)
      }
    }
  }

  // 2. Trier par préférence
  candidates.sort((a, b) => {
    // Préférence 1: Primary (si chaining désactivé)
    if (!chainingAllowed) {
      if (a.isPrimary) return -1
      if (b.isPrimary) return 1
    }

    // Préférence 2: Ping le plus faible (latence réseau)
    if (a.pingMs !== b.pingMs) {
      return a.pingMs - b.pingMs
    }

    // Préférence 3: OpTime le plus récent
    return b.optime - a.optime
  })

  return candidates[0]
}
```

**Critères de sélection** :

| Critère | Priorité | Description |
|---------|----------|-------------|
| **OpTime** | Haute | La source doit être à jour ou plus avancée |
| **Health** | Haute | La source doit être healthy (health = 1) |
| **Latence réseau** | Moyenne | Préférence pour la latence la plus faible |
| **Chaining allowed** | Moyenne | Si false, seul Primary est éligible |
| **Tags** | Basse | Peut influencer via tags de configuration |

### Chaîne de Réplication Complexe

Dans un Replica Set de 5 membres :

```
Scénario avec chaîne complète :

DC1 (Primary)                  DC2                     DC3
    │                          │                       │
    │                          │                       │
┌───┴────┐                ┌────┴────┐           ┌─────┴─────┐
│Primary │                │Secondary│           │ Secondary │
│  Node  │ ───────────→   │   DC2   │ ──────→   │    DC3    │
└────────┘   100ms latence└─────────┘  150ms    └───────────┘
                               │
                               │ 50ms
                               ↓
                          ┌─────────┐
                          │Secondary│
                          │ DC2-bis │
                          └─────────┘

Flux de réplication :
1. Primary (DC1) → Secondary DC2 (réplication directe)
2. Secondary DC2 → Secondary DC3 (chained)
3. Secondary DC2 → Secondary DC2-bis (chained)

Avantage : Économise bande passante entre DC1 et DC3
           (une seule connexion DC1→DC2 au lieu de deux)
```

## Configuration

### Activation/Désactivation Globale

**Par défaut** : La réplication chaînée est **activée** (`chainingAllowed: true`)

```javascript
// Vérifier l'état actuel
rs.conf().settings.chainingAllowed
// true (par défaut)

// Désactiver la réplication chaînée
cfg = rs.conf()
cfg.settings.chainingAllowed = false
rs.reconfig(cfg)

// Réactiver la réplication chaînée
cfg = rs.conf()
cfg.settings.chainingAllowed = true
rs.reconfig(cfg)
```

### Impact de la Modification

```javascript
// Avant désactivation
rs.status().members.forEach(m => {
  if (m.state === 2) {  // SECONDARY
    print(`${m.name} syncs from: ${m.syncSourceHost}`)
  }
})
// Exemple de sortie :
// mongodb-02:27017 syncs from: mongodb-01:27017 (Primary)
// mongodb-03:27017 syncs from: mongodb-02:27017 (Secondary - chaîné)
// mongodb-04:27017 syncs from: mongodb-02:27017 (Secondary - chaîné)

// Après désactivation (chainingAllowed = false)
// mongodb-02:27017 syncs from: mongodb-01:27017 (Primary)
// mongodb-03:27017 syncs from: mongodb-01:27017 (Primary)
// mongodb-04:27017 syncs from: mongodb-01:27017 (Primary)

// Tous les Secondary se synchronisent depuis le Primary
```

### Configuration par Tags

Bien que MongoDB ne permette pas de forcer explicitement la source de sync, on peut influencer le comportement avec les tags et la topologie réseau.

```javascript
// Configuration avec tags pour influencer le chaining
cfg = rs.conf()

cfg.members = [
  // DC1
  {
    _id: 0,
    host: "dc1-primary:27017",
    priority: 10,
    tags: { dc: "dc1", region: "us-east" }
  },

  // DC2 - Hub
  {
    _id: 1,
    host: "dc2-hub:27017",
    priority: 5,
    tags: { dc: "dc2", region: "eu-west", role: "hub" }
  },

  // DC2 - Leaf (devrait se synchroniser depuis dc2-hub)
  {
    _id: 2,
    host: "dc2-leaf:27017",
    priority: 1,
    tags: { dc: "dc2", region: "eu-west", role: "leaf" }
  },

  // DC3
  {
    _id: 3,
    host: "dc3-node:27017",
    priority: 1,
    tags: { dc: "dc3", region: "ap-south" }
  }
]

// Avec latence réseau optimisée :
// dc2-leaf préférera dc2-hub (même DC, faible latence)
// dc3-node pourrait chaîner via dc2-hub si latence DC1→DC3 élevée

rs.reconfig(cfg)
```

## Cas d'Usage

### 1. Architecture Multi-Datacenter

**Scénario** : Réplication entre 3 datacenters géographiquement distants

```
Configuration :
- DC1 (US-East) : Primary + 1 Secondary
- DC2 (EU-West) : 2 Secondary
- DC3 (Asia-Pacific) : 1 Secondary

Sans chaining :
┌────────┐
│  DC1   │
│Primary │
└───┬────┘
    │
    ├────────── Latence 100ms ───────→ DC2 (Secondary 1)
    ├────────── Latence 100ms ───────→ DC2 (Secondary 2)
    └────────── Latence 250ms ───────→ DC3 (Secondary)

Charge réseau Primary : 3 connexions sortantes
Bande passante DC1→DC2 : 2x flux complet
Bande passante DC1→DC3 : 1x flux complet

Avec chaining :
┌────────┐
│  DC1   │
│Primary │
└───┬────┘
    │
    ├────────── 100ms ──────→ DC2-Hub (Secondary 1)
    │                             │
    │                             ├── 5ms → DC2-Leaf (Secondary 2)
    │                             │
    └────────── 250ms ──────→     └── 200ms → DC3 (Secondary)

Charge réseau Primary : 2 connexions sortantes
Bande passante DC1→DC2 : 1x flux complet (économie 50%)
Bande passante DC2→DC3 : utilise la liaison EU-Asia
```

**Avantages** :
- Réduit la charge sur le Primary (2 connexions au lieu de 3)
- Économise la bande passante inter-datacenter
- Optimise le routage réseau (peut utiliser des liaisons régionales)

**Configuration** :
```javascript
{
  members: [
    // DC1
    { _id: 0, host: "dc1-primary:27017", priority: 10, tags: {dc: "dc1"} },
    { _id: 1, host: "dc1-secondary:27017", priority: 5, tags: {dc: "dc1"} },

    // DC2
    { _id: 2, host: "dc2-hub:27017", priority: 3, tags: {dc: "dc2", role: "hub"} },
    { _id: 3, host: "dc2-leaf:27017", priority: 1, tags: {dc: "dc2", role: "leaf"} },

    // DC3
    { _id: 4, host: "dc3-secondary:27017", priority: 1, tags: {dc: "dc3"} }
  ],

  settings: {
    chainingAllowed: true,
    electionTimeoutMillis: 30000  // Augmenté pour WAN
  }
}
```

### 2. Optimisation de Bande Passante

**Scénario** : Liaison réseau limitée depuis le Primary

```
Situation :
- Primary a une liaison de 1 Gbps
- Trafic applicatif consomme 600 Mbps
- 5 Secondary à synchroniser

Sans chaining :
Primary → 5 Secondary = 5 × 100 Mbps = 500 Mbps réplication
Total : 600 + 500 = 1100 Mbps (SATURATION !)

Avec chaining (arbre binaire) :
Primary → Secondary 1 = 100 Mbps
Primary → Secondary 2 = 100 Mbps
Secondary 1 → Secondary 3 = 100 Mbps (ne compte pas dans Primary)
Secondary 1 → Secondary 4 = 100 Mbps (ne compte pas dans Primary)
Secondary 2 → Secondary 5 = 100 Mbps (ne compte pas dans Primary)

Total sur Primary : 600 + 200 = 800 Mbps (OK)
```

**Topologie optimisée** :
```
                ┌──────────┐
                │ PRIMARY  │
                └────┬─────┘
                     │
          ┌──────────┴──────────┐
          │                     │
          ↓                     ↓
    ┌──────────┐          ┌──────────┐
    │Secondary1│          │Secondary2│
    └────┬─────┘          └────┬─────┘
         │                     │
    ┌────┴────┐                ↓
    │         │          ┌──────────┐
    ↓         ↓          │Secondary5│
┌──────┐ ┌──────┐        └──────────┘
│Sec 3 │ │Sec 4 │
└──────┘ └──────┘
```

### 3. Protection du Primary en Production

**Scénario** : Primary surchargé par les lectures/écritures applicatives

```javascript
// Configuration pour minimiser la charge de réplication sur Primary
cfg = rs.conf()

cfg.members = [
  // Primary - Gère uniquement les écritures applicatives
  { _id: 0, host: "primary:27017", priority: 10 },

  // Hub de réplication - Membre dédié à la distribution
  {
    _id: 1,
    host: "replication-hub:27017",
    priority: 5,
    tags: { role: "replication-hub" }
  },

  // Secondary classiques - Chaînent depuis le hub
  { _id: 2, host: "secondary-1:27017", priority: 1 },
  { _id: 3, host: "secondary-2:27017", priority: 1 },
  { _id: 4, host: "secondary-3:27017", priority: 1 },
  { _id: 5, host: "secondary-4:27017", priority: 1 }
]

cfg.settings.chainingAllowed = true

rs.reconfig(cfg)

// Résultat :
// - Primary → Hub uniquement (1 connexion)
// - Hub → 4 Secondary (distribue la charge de réplication)
// - Primary peut se concentrer sur les écritures applicatives
```

### 4. Réplication en Cascade pour Backup

```javascript
// Configuration avec delayed member chaîné
cfg = rs.conf()

cfg.members = [
  // Membres principaux
  { _id: 0, host: "primary:27017", priority: 10 },
  { _id: 1, host: "secondary-1:27017", priority: 5 },
  { _id: 2, host: "secondary-2:27017", priority: 5 },

  // Backup tier - Hidden, delayed, chaîné
  {
    _id: 3,
    host: "backup-secondary:27017",
    priority: 0,
    hidden: true,
    slaveDelay: 3600,  // 1 heure
    tags: { backup: "delayed" }
  }
]

// backup-secondary se synchronise depuis secondary-1 ou secondary-2
// avec 1h de délai, sans impacter le Primary
```

## Avantages et Inconvénients

### Avantages

**1. Réduction de la charge réseau sur le Primary**
```javascript
// Métriques avant/après activation chaining

// Sans chaining
{
  primaryNetworkOut: "500 MB/s",
  connectionsToPrimary: 5,
  primaryCPUReplication: "15%"
}

// Avec chaining
{
  primaryNetworkOut: "200 MB/s",  // -60%
  connectionsToPrimary: 2,         // -60%
  primaryCPUReplication: "6%"      // -60%
}
```

**2. Optimisation multi-datacenter**
- Économie de bande passante inter-DC
- Routage réseau intelligent
- Latence réduite pour certains membres

**3. Scalabilité**
- Permet de scaler le nombre de Secondary sans surcharger le Primary
- Architecture en arbre pour grandes topologies

**4. Flexibilité**
- Adaptation automatique aux conditions réseau
- Résilience en cas de défaillance d'une source

### Inconvénients

**1. Latence de réplication augmentée**

```
Exemple de latence cumulée :

Sans chaining :
Primary → Secondary-3 : 50ms latence réseau
Total : 50ms + temps de traitement

Avec chaining :
Primary → Secondary-1 : 50ms
Secondary-1 → Secondary-2 : 30ms
Secondary-2 → Secondary-3 : 20ms
Total : 100ms + 3× temps de traitement

Latence doublée pour Secondary-3 !
```

**2. Risque de replication lag**

```javascript
// Scénario de lag amplifié

// Primary écrit 10000 ops/sec
// Secondary-1 peut répliquer 8000 ops/sec (limite I/O disque)
// Secondary-2 chaîne depuis Secondary-1

Résultat :
- Secondary-1 : lag de 2000 ops/sec (prend du retard)
- Secondary-2 : lag de 2000+ ops/sec (prend encore plus de retard)

// Effet cascade du lag !
```

**3. Complexité du troubleshooting**

```javascript
// Difficulté à identifier la source d'un problème

// Exemple : Secondary-4 a un lag de 120 secondes

// Sans chaining :
// Problème = Primary ou Secondary-4 ou réseau entre les deux

// Avec chaining :
// Problème = Primary ?
//         ou Secondary-1 (source de sync) ?
//         ou Secondary-2 (si chaîne) ?
//         ou Secondary-4 ?
//         ou réseau entre chaque maillon ?

// Plus de points de défaillance à investiguer
```

**4. Dépendance entre membres**

```
Si un maillon de la chaîne tombe :

Primary → Secondary-1 → Secondary-2 → Secondary-3
                ↓
              CRASH

Secondary-2 et Secondary-3 doivent :
1. Détecter la perte de Secondary-1
2. Trouver une nouvelle source (Primary ou autre)
3. Potentiellement re-synchroniser

Temps d'interruption : 10-30 secondes pour changer de source
```

## Monitoring et Troubleshooting

### Visualiser la Topologie de Réplication

```javascript
// Script pour visualiser la chaîne de réplication
function visualizeReplicationChain() {
  const status = rs.status()
  const graph = {}

  status.members.forEach(member => {
    const source = member.syncSourceHost || 'none'
    if (!graph[source]) {
      graph[source] = []
    }
    graph[source].push({
      name: member.name,
      state: member.stateStr,
      lag: member.optimeDate ?
           ((status.date - member.optimeDate) / 1000).toFixed(2) + 's' :
           'N/A'
    })
  })

  print('=== Replication Chain Topology ===\n')

  function printNode(nodeName, depth = 0) {
    const indent = '  '.repeat(depth)
    const children = graph[nodeName] || []

    if (depth === 0) {
      print(`${indent}${nodeName} (ROOT)`)
    }

    children.forEach(child => {
      print(`${indent}├─→ ${child.name} [${child.state}] lag: ${child.lag}`)
      printNode(child.name, depth + 1)
    })
  }

  // Trouver le Primary (root)
  const primary = status.members.find(m => m.state === 1)
  if (primary) {
    printNode(primary.name)
  }

  // Afficher les nœuds orphelins (si chaining cassé)
  Object.keys(graph).forEach(source => {
    if (source !== 'none' && !status.members.find(m => m.name === source)) {
      print(`\nWarning: Orphaned chain from ${source}:`)
      graph[source].forEach(child => {
        print(`  └─→ ${child.name} [${child.state}]`)
      })
    }
  })
}

// Exécution
visualizeReplicationChain()

// Exemple de sortie :
// === Replication Chain Topology ===
//
// mongodb-01:27017 (ROOT)
// ├─→ mongodb-02:27017 [SECONDARY] lag: 0.5s
//   ├─→ mongodb-03:27017 [SECONDARY] lag: 1.2s
//   └─→ mongodb-04:27017 [SECONDARY] lag: 1.5s
// ├─→ mongodb-05:27017 [SECONDARY] lag: 0.3s
```

### Détecter les Problèmes de Chaining

```javascript
// Détecter les chaînes trop longues
function detectLongChains() {
  const status = rs.status()
  const chains = {}

  // Construire les chaînes
  function getChainLength(memberName, visited = new Set()) {
    if (visited.has(memberName)) return 0  // Cycle détecté
    visited.add(memberName)

    const member = status.members.find(m => m.name === memberName)
    if (!member || member.state === 1) return 0  // Primary

    const source = member.syncSourceHost
    if (!source) return 0

    return 1 + getChainLength(source, visited)
  }

  status.members.forEach(member => {
    if (member.state === 2) {  // SECONDARY
      const chainLength = getChainLength(member.name)
      chains[member.name] = chainLength
    }
  })

  print('=== Chain Length Analysis ===\n')

  Object.entries(chains).forEach(([member, length]) => {
    const status = length === 0 ? '✓ Direct' :
                   length === 1 ? '○ One hop' :
                   length === 2 ? '⚠ Two hops' :
                   '✗ Too long!'
    print(`${status} ${member}: ${length} hop(s)`)
  })

  const maxChain = Math.max(...Object.values(chains))
  if (maxChain > 2) {
    print(`\nWARNING: Chain length exceeds recommended maximum (2)`)
    print(`Consider disabling chaining or restructuring topology`)
  }
}

// Exécution
detectLongChains()
```

### Mesurer l'Impact du Chaining

```javascript
// Comparer les performances avec/sans chaining
function measureChainingImpact() {
  const metrics = {
    withChaining: {},
    withoutChaining: {}
  }

  // Phase 1 : Mesurer avec chaining
  print('Phase 1: Measuring with chaining enabled...')

  const cfg1 = rs.conf()
  if (!cfg1.settings.chainingAllowed) {
    cfg1.settings.chainingAllowed = true
    rs.reconfig(cfg1)
    sleep(30000)  // Attendre stabilisation
  }

  metrics.withChaining = captureMetrics()

  // Phase 2 : Mesurer sans chaining
  print('Phase 2: Measuring with chaining disabled...')

  const cfg2 = rs.conf()
  cfg2.settings.chainingAllowed = false
  rs.reconfig(cfg2)
  sleep(30000)  // Attendre stabilisation

  metrics.withoutChaining = captureMetrics()

  // Phase 3 : Restaurer configuration originale
  const cfgRestore = rs.conf()
  cfgRestore.settings.chainingAllowed = cfg1.settings.chainingAllowed
  rs.reconfig(cfgRestore)

  // Analyse
  print('\n=== Chaining Impact Analysis ===\n')

  print('Primary Network Output:')
  print(`  With chaining:    ${metrics.withChaining.primaryNetworkOut} MB/s`)
  print(`  Without chaining: ${metrics.withoutChaining.primaryNetworkOut} MB/s`)
  print(`  Difference:       ${
    (metrics.withChaining.primaryNetworkOut - metrics.withoutChaining.primaryNetworkOut).toFixed(2)
  } MB/s`)

  print('\nMax Replication Lag:')
  print(`  With chaining:    ${metrics.withChaining.maxLag}s`)
  print(`  Without chaining: ${metrics.withoutChaining.maxLag}s`)
  print(`  Difference:       ${
    (metrics.withChaining.maxLag - metrics.withoutChaining.maxLag).toFixed(2)
  }s`)

  print('\nRecommendation:')
  if (metrics.withChaining.maxLag > metrics.withoutChaining.maxLag * 1.5) {
    print('  ⚠ Chaining significantly increases lag - consider disabling')
  } else if (metrics.withChaining.primaryNetworkOut < metrics.withoutChaining.primaryNetworkOut * 0.7) {
    print('  ✓ Chaining provides significant network savings - keep enabled')
  } else {
    print('  ○ Impact is moderate - evaluate based on specific needs')
  }
}

function captureMetrics() {
  const status = rs.status()
  const serverStatus = db.serverStatus()

  // Lag max
  const maxLag = Math.max(
    0,
    ...status.members
      .filter(m => m.state === 2 && m.optimeDate)
      .map(m => (status.date - m.optimeDate) / 1000)
  )

  // Network out (approximation)
  const networkOut = serverStatus.network.bytesOut / 1024 / 1024

  return {
    maxLag: maxLag,
    primaryNetworkOut: networkOut,
    timestamp: new Date()
  }
}
```

### Alerting sur Problèmes de Chaining

```javascript
// Alertes pour problèmes de réplication chaînée
function checkChainingHealth() {
  const status = rs.status()
  const alerts = []

  // Check 1 : Chaînes trop longues
  status.members.forEach(member => {
    if (member.state === 2) {
      const chainLength = calculateChainLength(member.name, status)
      if (chainLength > 2) {
        alerts.push({
          severity: 'WARNING',
          component: 'Chaining',
          message: `${member.name} has chain length of ${chainLength} hops`,
          recommendation: 'Consider restructuring topology or disabling chaining'
        })
      }
    }
  })

  // Check 2 : Lag amplifié dans la chaîne
  const primaryOpTime = status.members.find(m => m.state === 1)?.optimeDate
  if (primaryOpTime) {
    status.members.forEach(member => {
      if (member.state === 2 && member.optimeDate) {
        const lag = (status.date - member.optimeDate) / 1000
        const source = member.syncSourceHost

        if (source) {
          const sourceMember = status.members.find(m => m.name === source)
          if (sourceMember && sourceMember.optimeDate) {
            const sourceLag = (status.date - sourceMember.optimeDate) / 1000

            // Si le lag du membre est > 2× le lag de sa source
            if (lag > sourceLag * 2 && lag > 10) {
              alerts.push({
                severity: 'WARNING',
                component: 'Replication Lag',
                message: `${member.name} lag (${lag.toFixed(2)}s) is 2× source lag (${sourceLag.toFixed(2)}s)`,
                recommendation: 'Investigate member performance or network issues'
              })
            }
          }
        }
      }
    })
  }

  // Check 3 : Sources instables
  const syncSources = {}
  status.members.forEach(m => {
    if (m.state === 2 && m.syncSourceHost) {
      syncSources[m.syncSourceHost] = (syncSources[m.syncSourceHost] || 0) + 1
    }
  })

  Object.entries(syncSources).forEach(([source, count]) => {
    if (count > 3) {
      alerts.push({
        severity: 'INFO',
        component: 'Load Distribution',
        message: `${source} is sync source for ${count} members`,
        recommendation: 'Consider if this member can handle the load'
      })
    }
  })

  return alerts
}

function calculateChainLength(memberName, status, visited = new Set()) {
  if (visited.has(memberName)) return 0
  visited.add(memberName)

  const member = status.members.find(m => m.name === memberName)
  if (!member || member.state === 1) return 0

  const source = member.syncSourceHost
  if (!source) return 0

  return 1 + calculateChainLength(source, status, visited)
}

// Utilisation
const alerts = checkChainingHealth()
if (alerts.length > 0) {
  print('=== Chaining Health Alerts ===\n')
  alerts.forEach(alert => {
    print(`[${alert.severity}] ${alert.component}`)
    print(`  ${alert.message}`)
    print(`  → ${alert.recommendation}\n`)
  })
}
```

## Alternatives et Comparaisons

### Chaining vs Direct Replication

**Comparaison des approches** :

| Aspect | Chaining Enabled | Chaining Disabled |
|--------|------------------|-------------------|
| **Charge Primary** | Faible (1-2 connexions) | Élevée (N connexions) |
| **Latence réplication** | Variable (cumulative) | Constante (directe) |
| **Bande passante** | Optimisée | Non optimisée |
| **Complexité** | Moyenne à élevée | Faible |
| **Résilience** | Points de défaillance multiples | Direct, plus simple |
| **Scalabilité** | Excellente | Limitée par Primary |

**Décision** :

```javascript
// Matrice de décision
function recommendChainingSetting(scenario) {
  const scores = {
    enableChaining: 0,
    disableChaining: 0
  }

  // Facteurs favorisant le chaining
  if (scenario.numberOfSecondaries > 5) scores.enableChaining += 3
  if (scenario.primaryBandwidthLimited) scores.enableChaining += 4
  if (scenario.multiDatacenter) scores.enableChaining += 3
  if (scenario.geographicallyDistributed) scores.enableChaining += 2

  // Facteurs favorisant le direct
  if (scenario.lowLatencyRequired) scores.disableChaining += 4
  if (scenario.criticalConsistency) scores.disableChaining += 3
  if (scenario.simpleTroubleshooting) scores.disableChaining += 2
  if (scenario.numberOfSecondaries <= 3) scores.disableChaining += 2

  print('=== Chaining Recommendation ===\n')
  print(`Enable Chaining Score:  ${scores.enableChaining}`)
  print(`Disable Chaining Score: ${scores.disableChaining}`)

  if (scores.enableChaining > scores.disableChaining) {
    print('\n✓ Recommendation: ENABLE chaining')
    print('  Benefits outweigh drawbacks for your scenario')
  } else if (scores.disableChaining > scores.enableChaining) {
    print('\n✓ Recommendation: DISABLE chaining')
    print('  Direct replication better suits your needs')
  } else {
    print('\n○ Recommendation: NEUTRAL')
    print('  Evaluate based on specific operational experience')
  }
}

// Exemple d'utilisation
recommendChainingSetting({
  numberOfSecondaries: 7,
  primaryBandwidthLimited: true,
  multiDatacenter: true,
  geographicallyDistributed: true,
  lowLatencyRequired: false,
  criticalConsistency: false,
  simpleTroubleshooting: false
})
```

### Chaining vs Priority-Based Topology

**Alternative : Utiliser les priorités pour contrôler la topologie**

```javascript
// Au lieu de compter sur le chaining automatique,
// structurer explicitement via priorités et failover

cfg = rs.conf()

cfg.members = [
  // Tier 1 : Primary et backup Primary
  { _id: 0, host: "tier1-primary:27017", priority: 100 },
  { _id: 1, host: "tier1-backup:27017", priority: 90 },

  // Tier 2 : Secondary haute priorité (sources de chaining potentielles)
  { _id: 2, host: "tier2-hub-1:27017", priority: 50 },
  { _id: 3, host: "tier2-hub-2:27017", priority: 50 },

  // Tier 3 : Secondary normale
  { _id: 4, host: "tier3-node-1:27017", priority: 10 },
  { _id: 5, host: "tier3-node-2:27017", priority: 10 },
  { _id: 6, host: "tier3-node-3:27017", priority: 10 }
]

// Avec chaining:
// tier3 nodes préféreront se synchroniser depuis tier2 hubs (meilleure latence)
// ce qui réduit la charge sur tier1

// Sans chaining:
// Tous se synchronisent depuis tier1-primary
// Plus simple mais moins efficace
```

## Bonnes Pratiques

### 1. Quand Activer le Chaining

✅ **Activer dans ces cas** :

```javascript
const enableChainingWhen = {
  topology: [
    'Plus de 5 membres Secondary',
    'Multi-datacenter avec latences WAN',
    'Membres géographiquement distribués',
    'Architecture hiérarchique (tiers)'
  ],

  constraints: [
    'Bande passante limitée sur Primary',
    'Coûts de bande passante inter-DC élevés',
    'Besoin de scaler au-delà de 10 membres'
  ],

  acceptable: [
    'Latence de réplication de 5-10s acceptable',
    'Pas de requirement temps-réel strict',
    'Équipe capable de troubleshooter topologie complexe'
  ]
}
```

### 2. Quand Désactiver le Chaining

❌ **Désactiver dans ces cas** :

```javascript
const disableChainingWhen = {
  requirements: [
    'Latence de réplication < 2s requise',
    'Consistency forte et temps réel',
    'Nombre de membres < 5',
    'Tous les membres dans même datacenter'
  ],

  operational: [
    'Équipe de petite taille',
    'Troubleshooting simple requis',
    'Pas de contraintes de bande passante'
  ],

  issues: [
    'Lag de réplication problématique',
    'Chaînes trop longues (>2 hops)',
    'Sources instables'
  ]
}
```

### 3. Monitoring Essentiel

```javascript
// Métriques clés à surveiller avec chaining
const chainingMonitoring = {
  metrics: {
    // Topologie
    chainLength: {
      alert: '> 2 hops',
      check: 'every 60s'
    },

    // Latence
    replicationLag: {
      warning: '> 10s',
      critical: '> 60s',
      check: 'every 30s'
    },

    // Distribution
    syncSourceDistribution: {
      alert: 'Single source for > 4 members',
      check: 'every 300s'
    },

    // Stabilité
    syncSourceChanges: {
      alert: '> 3 changes in 10 minutes',
      check: 'continuously'
    }
  },

  dashboards: [
    'Replication chain topology graph',
    'Lag per member over time',
    'Sync source distribution',
    'Network bandwidth per member'
  ]
}
```

### 4. Configuration Optimale

```javascript
// Configuration recommandée pour multi-DC avec chaining
const optimalChainingConfig = {
  members: [
    // DC1 (Primary datacenter)
    {
      _id: 0,
      host: "dc1-primary:27017",
      priority: 100,
      tags: { dc: "dc1", role: "primary" }
    },
    {
      _id: 1,
      host: "dc1-secondary:27017",
      priority: 90,
      tags: { dc: "dc1", role: "secondary" }
    },

    // DC2 (Secondary datacenter - hub)
    {
      _id: 2,
      host: "dc2-hub:27017",
      priority: 50,
      tags: { dc: "dc2", role: "hub" }
    },
    {
      _id: 3,
      host: "dc2-node-1:27017",
      priority: 10,
      tags: { dc: "dc2", role: "node" }
    },
    {
      _id: 4,
      host: "dc2-node-2:27017",
      priority: 10,
      tags: { dc: "dc2", role: "node" }
    },

    // DC3 (Tertiary datacenter)
    {
      _id: 5,
      host: "dc3-secondary:27017",
      priority: 10,
      tags: { dc: "dc3", role: "secondary" }
    },

    // Arbiter (pour majorité impaire)
    {
      _id: 6,
      host: "arbiter:27017",
      arbiterOnly: true,
      priority: 0
    }
  ],

  settings: {
    chainingAllowed: true,
    electionTimeoutMillis: 30000,  // 30s pour WAN
    heartbeatIntervalMillis: 2000,

    getLastErrorDefaults: {
      w: "majority",
      wtimeout: 5000
    },

    // Write concern pour multi-DC
    getLastErrorModes: {
      multiDC: { dc: 2 }  // Au moins 2 DC
    }
  }
}
```

### 5. Tests de Validation

```bash
#!/bin/bash
# test-chaining-performance.sh

echo "=== Testing Chaining Performance ==="

# Test 1 : Mesurer baseline sans chaining
echo "Test 1: Baseline (no chaining)..."
mongosh --eval "
  cfg = rs.conf()
  cfg.settings.chainingAllowed = false
  rs.reconfig(cfg)
"

sleep 30
BASELINE_LAG=$(mongosh --quiet --eval "
  Math.max(...rs.status().members
    .filter(m => m.state === 2 && m.optimeDate)
    .map(m => (rs.status().date - m.optimeDate) / 1000))
")

echo "Baseline max lag: ${BASELINE_LAG}s"

# Test 2 : Avec chaining
echo "Test 2: With chaining..."
mongosh --eval "
  cfg = rs.conf()
  cfg.settings.chainingAllowed = true
  rs.reconfig(cfg)
"

sleep 30
CHAINING_LAG=$(mongosh --quiet --eval "
  Math.max(...rs.status().members
    .filter(m => m.state === 2 && m.optimeDate)
    .map(m => (rs.status().date - m.optimeDate) / 1000))
")

echo "Chaining max lag: ${CHAINING_LAG}s"

# Analyse
echo ""
echo "=== Results ==="
echo "Baseline:  ${BASELINE_LAG}s"
echo "Chaining:  ${CHAINING_LAG}s"
echo "Difference: $(echo "$CHAINING_LAG - $BASELINE_LAG" | bc)s"

if (( $(echo "$CHAINING_LAG > $BASELINE_LAG * 1.5" | bc -l) )); then
  echo "⚠ WARNING: Chaining increases lag by >50%"
else
  echo "✓ OK: Lag increase is acceptable"
fi
```

## Conclusion

La réplication chaînée est un mécanisme puissant pour optimiser la topologie d'un Replica Set MongoDB, particulièrement dans des architectures multi-datacenter ou à grande échelle.

**Points clés** :

- ✅ **Activation par défaut** : `chainingAllowed: true`
- ✅ **Avantages** : Réduit charge Primary, optimise bande passante, scalabilité
- ⚠️ **Inconvénients** : Latence accrue, complexité, dépendances
- 🔧 **Configuration** : Simple (un seul paramètre global)
- 📊 **Monitoring** : Essentiel pour détecter problèmes (chaînes longues, lag)

**Décision** :

| Scénario | Recommandation |
|----------|----------------|
| < 5 membres, même DC | **Désactiver** |
| 5-10 membres, multi-DC | **Activer** avec monitoring |
| > 10 membres, géo-distribué | **Activer** (essentiel) |
| Latence critique < 2s | **Désactiver** |
| Bande passante limitée | **Activer** |

La réplication chaînée doit être considérée comme un outil d'optimisation à utiliser judicieusement en fonction des contraintes spécifiques de votre architecture et de vos exigences opérationnelles.

⏭️ [Sharding (Partitionnement Horizontal)](/10-sharding/README.md)
