🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Chapitre 13 : Monitoring et Administration

## Vue d'ensemble

Le monitoring et l'administration efficaces d'une infrastructure MongoDB sont essentiels pour garantir la disponibilité, les performances et la fiabilité des systèmes en production. Ce chapitre s'adresse aux SRE (Site Reliability Engineers) et administrateurs système qui ont la responsabilité d'exploiter MongoDB à l'échelle, en fournissant une compréhension approfondie des métriques critiques, des outils d'observation et des pratiques d'administration.

## Importance du monitoring MongoDB

### Enjeux en production

Dans un environnement de production, l'absence de monitoring adéquat peut entraîner :

- **Dégradation silencieuse des performances** : Les requêtes lentes s'accumulent progressivement sans alerte précoce
- **Saturation de ressources** : Mémoire, CPU, disque ou connexions atteignent leurs limites sans anticipation
- **Indisponibilité non détectée** : Les failovers ou pannes partielles passent inaperçus jusqu'à impact utilisateur
- **Coûts imprévus** : Surconsommation de ressources cloud sans visibilité sur l'origine
- **Incidents prolongés** : Temps de résolution allongé par manque de données diagnostiques

### Objectifs du monitoring

Un système de monitoring MongoDB bien conçu doit permettre de :

1. **Détecter proactivement** les anomalies avant qu'elles n'impactent les utilisateurs
2. **Diagnostiquer rapidement** la cause racine des problèmes de performance
3. **Planifier la capacité** en anticipant les besoins futurs en ressources
4. **Optimiser continuellement** les performances des requêtes et de l'infrastructure
5. **Respecter les SLA** en maintenant une visibilité constante sur les métriques critiques
6. **Documenter l'historique** pour les post-mortems et analyses de tendances

## Les couches du monitoring MongoDB

### 1. Couche infrastructure (matérielle)

Surveillance des ressources système sous-jacentes :

```
┌─────────────────────────────────────────┐
│         INFRASTRUCTURE                  │
│  • CPU (utilisation, saturation)        │
│  • Mémoire (usage, swap, pression)      │
│  • Disque (IOPS, latence, espace)       │
│  • Réseau (bande passante, latence)     │
│  • Système de fichiers                  │
└─────────────────────────────────────────┘
```

**Exemple de métriques critiques** :
- CPU : `%user`, `%system`, `%iowait`, `%steal` (en cloud)
- Mémoire : Resident Set Size (RSS), cache WiredTiger, pages dirty
- Disque : Latence en lecture/écriture, taux d'utilisation, queue depth
- Réseau : Connexions établies, paquets perdus, bande passante saturée

### 2. Couche processus MongoDB

Surveillance du processus `mongod` et `mongos` :

```
┌─────────────────────────────────────────┐
│         PROCESSUS MONGODB               │
│  • Connexions actives/disponibles       │
│  • Opérations en cours                  │
│  • Locks et contentions                 │
│  • Cache et mémoire interne             │
│  • Latence des opérations               │
└─────────────────────────────────────────┘
```

**Exemple d'analyse** :

```javascript
// Commande serverStatus pour obtenir l'état du serveur
db.serverStatus()

// Résultat partiel simplifié :
{
  "connections": {
    "current": 847,
    "available": 51153,
    "totalCreated": 12453
  },
  "opcounters": {
    "insert": 425897,
    "query": 1847562,
    "update": 284563,
    "delete": 12453,
    "getmore": 45621,
    "command": 3254789
  },
  "wiredTiger": {
    "cache": {
      "bytes currently in the cache": 3221225472,
      "maximum bytes configured": 4294967296,
      "pages evicted by application threads": 12453
    }
  }
}
```

**Alertes recommandées** :
- `current connections / available connections > 80%` : Risque de saturation
- `cache eviction rate > seuil` : Pression mémoire excessive
- `lock wait time > 100ms` : Contention importante

### 3. Couche base de données et collections

Surveillance au niveau applicatif :

```
┌─────────────────────────────────────────┐
│         DATABASE & COLLECTIONS          │
│  • Taille des bases et collections      │
│  • Nombre de documents                  │
│  • Fragmentation                        │
│  • Performance des index                │
│  • Slow queries                         │
└─────────────────────────────────────────┘
```

**Exemple de commande dbStats** :

```javascript
db.stats()

// Résultat :
{
  "db": "production",
  "collections": 24,
  "views": 2,
  "objects": 45789234,      // Nombre total de documents
  "avgObjSize": 1247.56,    // Taille moyenne d'un document (bytes)
  "dataSize": 57123456789,  // Taille totale des données
  "storageSize": 62345678901, // Espace disque alloué
  "indexes": 48,
  "indexSize": 4567890123,
  "totalSize": 66913568024,
  "scaleFactor": 1,
  "fsUsedSize": 445678901234,
  "fsTotalSize": 1099511627776
}
```

**Métriques dérivées importantes** :
- **Ratio de fragmentation** : `storageSize / dataSize` (optimal proche de 1.0)
- **Index overhead** : `indexSize / dataSize` (surveiller si > 50%)
- **Taux de croissance** : Évolution de `dataSize` par jour/semaine

### 4. Couche réplication et distribution

Pour les déploiements Replica Set et Sharded Cluster :

```
┌─────────────────────────────────────────┐
│         REPLICATION & SHARDING          │
│  • Lag de réplication                   │
│  • Health des membres                   │
│  • Élections et failovers               │
│  • Distribution des chunks              │
│  • Migration en cours                   │
└─────────────────────────────────────────┘
```

**Exemple d'analyse de replication lag** :

```javascript
rs.printSlaveReplicationInfo()

// Résultat typique :
source: secondary1.domain.com:27017
    syncedTo: Mon Dec 08 2025 14:32:45 GMT+0100
    0 secs (0 hrs) behind the primary

source: secondary2.domain.com:27017
    syncedTo: Mon Dec 08 2025 14:32:43 GMT+0100
    2 secs (0 hrs) behind the primary  // ⚠️ Lag détecté
```

**Seuils d'alerte critiques** :
- Replication lag > 10 secondes : Warning
- Replication lag > 60 secondes : Critical
- Membre en état RECOVERING > 5 minutes : Investigation requise
- Chunk migration stalled > 30 minutes : Problème de balancing

## Architecture de monitoring recommandée

### Approche multi-couches

```
┌──────────────────────────────────────────────────────────┐
│                    VISUALISATION                         │
│              Grafana / Atlas Charts                      │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│                  AGRÉGATION / ALERTING                   │
│           Prometheus / Atlas Monitoring                  │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│                   COLLECTEURS                            │
│    MongoDB Exporter / mongodb_exporter                   │
│    Atlas Monitoring Agent / Cloud Watch                  │
└───────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│                  SOURCES DE DONNÉES                      │
│  serverStatus / dbStats / replSetGetStatus               │
│  currentOp / logs / FTDC                                 │
└──────────────────────────────────────────────────────────┘
```

### Outils essentiels par catégorie

#### Outils natifs MongoDB

| Outil | Usage | Fréquence |
|-------|-------|-----------|
| `serverStatus` | État global du serveur | Temps réel / 1s |
| `dbStats` | Statistiques par base | Périodique / 5min |
| `currentOp` | Opérations en cours | Diagnostic à la demande |
| `explain()` | Analyse de requêtes | Optimisation continue |
| Profiler | Requêtes lentes | Troubleshooting |
| FTDC | Diagnostics continus | Analyse post-mortem |

#### Outils de monitoring externe

| Catégorie | Outils | Cas d'usage |
|-----------|--------|-------------|
| **Collecteurs** | mongodb_exporter, Telegraf | Export métriques vers Prometheus |
| **Stockage** | Prometheus, InfluxDB, TimescaleDB | Time-series database |
| **Visualisation** | Grafana, Kibana, Atlas Charts | Dashboards et exploration |
| **Alerting** | Alertmanager, PagerDuty, OpsGenie | Notifications et escalade |
| **Logs** | ELK Stack, Splunk, Loki | Agrégation et analyse de logs |
| **APM** | New Relic, Datadog, Dynatrace | Monitoring applicatif bout-en-bout |

#### Solutions managées

**MongoDB Atlas** offre une solution intégrée avec :
- Monitoring en temps réel (rafraîchissement 10s-1min)
- Alertes personnalisables sur 50+ métriques
- Query Performance Insights (requêtes lentes automatiques)
- Real-Time Performance Panel
- Logs accessibles via interface et API
- Intégration avec Datadog, New Relic, PagerDuty

**Exemple de configuration d'alerte Atlas** :

```json
{
  "eventTypeName": "REPLICATION_OPLOG_WINDOW_RUNNING_OUT",
  "enabled": true,
  "notifications": [
    {
      "typeName": "PAGER_DUTY",
      "intervalMin": 5,
      "delayMin": 0
    }
  ],
  "threshold": {
    "operator": "LESS_THAN",
    "threshold": 1,
    "units": "HOURS"
  }
}
```

## Stratégie de monitoring recommandée

### 1. Établir une baseline

Avant de définir des alertes, comprendre le comportement normal :

```
📊 Collecte pendant 2-4 semaines :
├── Patterns quotidiens (heures de pointe)
├── Variations hebdomadaires (weekend vs semaine)
├── Pics mensuels (facturation, batch jobs)
└── Saisonnalité (périodes métier)
```

**Métriques de baseline critiques** :
- Latence P50, P95, P99 des opérations
- Throughput moyen et pics (ops/sec)
- Utilisation mémoire en conditions normales
- Taille quotidienne de l'oplog consommée

### 2. Définir des SLI/SLO/SLA

**Service Level Indicators (SLI)** - Métriques mesurées :
- Disponibilité : % de temps avec réponse valide
- Latence : Percentile P95 < 100ms
- Throughput : Capacité à gérer X ops/sec

**Service Level Objectives (SLO)** - Objectifs internes :
- 99.9% de disponibilité sur 30 jours
- P95 latency < 100ms pour 99.5% des requêtes
- Replication lag < 5s en conditions normales

**Service Level Agreements (SLA)** - Engagements contractuels :
- 99.95% uptime mensuel avec pénalités si non respecté

### 3. Hiérarchiser les alertes

```
┌─────────────────────────────────────────────┐
│  CRITICAL - Intervention immédiate requise  │
│  • Primary down                             │
│  • Replication stopped                      │
│  • Disk full (>95%)                         │
│  • OOM imminent                             │
└─────────────────────────────────────────────┘
┌─────────────────────────────────────────────┐
│  WARNING - Dégradation anticipée            │
│  • Replication lag > 30s                    │
│  • Slow queries > seuil                     │
│  • Disk usage > 80%                         │
│  • Connection pool > 70%                    │
└─────────────────────────────────────────────┘
┌─────────────────────────────────────────────┐
│  INFO - Surveillance et tendances           │
│  • Index usage stats                        │
│  • Growth rate analysis                     │
│  • Query patterns changes                   │
└─────────────────────────────────────────────┘
```

### 4. Implémenter l'observabilité complète

**Les trois piliers** :

1. **Metrics** (Métriques)
   - Séries temporelles quantitatives
   - Agrégation et percentiles
   - Dashboards et graphiques

2. **Logs** (Journaux)
   - Events détaillés
   - Contexte et tracing
   - Corrélation avec métriques

3. **Traces** (Traces distribuées)
   - Suivi de requêtes bout-en-bout
   - Identification des goulots d'étranglement
   - Dépendances inter-services

**Exemple de corrélation** :

```
[15:42:13] METRIC: Latency spike P95 = 2.3s (normally 85ms)
             ↓
[15:42:10] LOG: "[SLOW] Query on users.profiles took 2145ms"
             ↓
[15:42:10] TRACE: Span "db.query" → Missing index on {email: 1}
             ↓
           ROOT CAUSE: Full collection scan après déploiement
```

## Métriques clés par persona

### Pour les SRE (Site Reliability Engineers)

**Focus : Disponibilité et performance système**

```yaml
Métriques prioritaires:
  - Uptime et availability (%)
  - Latency percentiles (P50, P95, P99)
  - Error rate (failed operations %)
  - Resource saturation (CPU, RAM, Disk)
  - Replication lag
  - Failover events et durée

Dashboards typiques:
  - Service health overview
  - Resource utilization trends
  - Incident timeline
  - Capacity planning projection
```

### Pour les DBA (Database Administrators)

**Focus : Optimisation des requêtes et données**

```yaml
Métriques prioritaires:
  - Slow queries count et exemples
  - Index usage et efficacité
  - Lock contention et wait time
  - Collection growth rate
  - Fragmentation ratio
  - Query execution plans

Dashboards typiques:
  - Query performance analysis
  - Index effectiveness report
  - Schema evolution tracking
  - Maintenance windows planning
```

### Pour les développeurs

**Focus : Performance applicative**

```yaml
Métriques prioritaires:
  - Query latency par collection
  - Operation success rate
  - Connection pool exhaustion
  - Application error correlation
  - Transaction abort rate

Dashboards typiques:
  - Per-collection performance
  - API endpoint latency breakdown
  - Error rate by operation type
```

## Checklist de mise en place du monitoring

### Phase 1 : Infrastructure (Jour 1-3)

- [ ] Déployer les agents de collecte (node_exporter, mongodb_exporter)
- [ ] Configurer Prometheus ou solution time-series équivalente
- [ ] Vérifier la collecte des métriques système (CPU, RAM, Disk, Network)
- [ ] Configurer la rétention des métriques (30-90 jours recommandé)
- [ ] Sécuriser les endpoints de métriques (authentification, TLS)

### Phase 2 : MongoDB (Jour 4-7)

- [ ] Activer le monitoring MongoDB (serverStatus, dbStats)
- [ ] Configurer le profiler avec seuil approprié (100-200ms)
- [ ] Activer les logs structurés (JSON format)
- [ ] Configurer l'agrégation des logs (Loki, ELK, Splunk)
- [ ] Documenter la topologie (Replica Set / Sharded Cluster)

### Phase 3 : Visualisation (Jour 8-10)

- [ ] Déployer Grafana ou Atlas Charts
- [ ] Importer des dashboards de référence MongoDB
- [ ] Créer des dashboards personnalisés par service
- [ ] Configurer les variables de dashboard (env, cluster, node)
- [ ] Établir les permissions d'accès

### Phase 4 : Alerting (Jour 11-14)

- [ ] Définir les SLO par service
- [ ] Configurer les alertes critiques (downtime, replication issues)
- [ ] Configurer les alertes de warning (resource saturation)
- [ ] Intégrer avec PagerDuty / OpsGenie
- [ ] Documenter les runbooks d'intervention
- [ ] Tester les alertes et escalations

### Phase 5 : Optimisation continue (Ongoing)

- [ ] Réviser hebdomadairement les alertes non pertinentes
- [ ] Ajuster les seuils selon la baseline observée
- [ ] Documenter les incidents et post-mortems
- [ ] Automatiser les remediations courantes
- [ ] Mettre à jour la documentation d'exploitation

## Exemples de dashboards essentiels

### Dashboard 1 : Health Overview

```
┌─────────────────────────────────────────────────────────┐
│ MongoDB Cluster Health                     [Last 1h ▼]  │
├─────────────────────────────────────────────────────────┤
│ ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│ │Uptime   │  │Avg      │  │Active   │  │Repl     │      │
│ │99.98%   │  │Latency  │  │Conn     │  │Lag      │      │
│ │  🟢     │  │47ms 🟢  │  │142 🟢   │  │0.2s 🟢  │      │
│ └─────────┘  └─────────┘  └─────────┘  └─────────┘      │
├─────────────────────────────────────────────────────────┤
│ Operations Per Second                                   │
│ ▁▂▃▄▅▆█▇▆▅▄▃▂▁▁▂▃▄▅▆█▇▆▅▄▃▂▁▁▂▃▄▅▆█▇▆▅▄▃▂▁ 1.2K ops/s   │
├─────────────────────────────────────────────────────────┤
│ Replica Set Members                                     │
│ primary1     [PRIMARY]    ⬤ Healthy    Lag: 0ms        │
│ secondary1   [SECONDARY]  ⬤ Healthy    Lag: 150ms      │
│ secondary2   [SECONDARY]  ⬤ Healthy    Lag: 180ms      │
└─────────────────────────────────────────────────────────┘
```

### Dashboard 2 : Resource Utilization

```
┌─────────────────────────────────────────────────────────┐
│ Resource Utilization - node1.cluster.local              │
├─────────────────────────────────────────────────────────┤
│ CPU Usage %                        RAM Usage GB         │
│ ██████████████████░░ 75%          ██████████████░░ 28/32│
│                                                         │
│ Disk IOPS                          Network MB/s         │
│ ▁▂▃█▆▅▄▃▂▁ 4.2K                  ▁▃▅█▆▄▃▂▁ 145 MB/s     │
│                                                         │
│ WiredTiger Cache                   Page Faults/s        │
│ ██████████████░░░░ 3.2/4 GB       ▁▁▁▂▁▁▁▁ 12/s         │
└─────────────────────────────────────────────────────────┘
```

### Dashboard 3 : Query Performance

```
┌─────────────────────────────────────────────────────────────┐
│ Top 5 Slowest Operations (Last 1h)                          │
├─────────────────────────────────────────────────────────────┤
│ Collection      │ Operation │ Avg Time │ Count │ Index      │
│ users.profiles  │ find      │ 2.4s     │ 234   │ ❌ SCAN    │
│ orders.history  │ aggregate │ 1.8s     │ 89    │ ⚠️ PARTIAL │
│ products.catalog│ find      │ 890ms    │ 1.2K  │ ✅ IXSCAN  │
│ logs.events     │ insert    │ 450ms    │ 45K   │ N/A        │
│ sessions.active │ update    │ 380ms    │ 3.4K  │ ✅ IXSCAN  │
└─────────────────────────────────────────────────────────────┘
```

## Prochaines sections

Ce chapitre continue avec des sections détaillées sur :

- **13.1** Métriques clés à surveiller en détail
- **13.2** Commandes d'administration essentielles
- **13.3** Profiler de requêtes et optimisation
- **13.4** Gestion et analyse des logs
- **13.5** MongoDB Database Tools
- **13.6** mongostat et mongotop
- **13.7** Intégration Prometheus et Grafana
- **13.8** MongoDB Ops Manager
- **13.9** Alerting et notifications
- **13.10** Diagnostics avec FTDC
- **13.11** Gestion de la mémoire et du cache WiredTiger

---

**Points clés à retenir** :

✅ Le monitoring MongoDB nécessite une approche multi-couches (infrastructure, processus, application, distribution)

✅ Établir une baseline avant de configurer des alertes pour éviter les faux positifs

✅ Prioriser les métriques selon le rôle (SRE, DBA, Dev) et les objectifs de service (SLO)

✅ Combiner metrics, logs et traces pour une observabilité complète

✅ Automatiser la collecte, visualisation et alerting dès le départ

✅ Documenter les runbooks et procédures d'intervention pour chaque alerte

---

**Prochaine section** : 13.1 Métriques clés à surveiller

⏭️ [Métriques clés à surveiller](/13-monitoring-administration/01-metriques-cles.md)
