🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E. Checklist de Performance

## Vue d'ensemble

Cette annexe fournit des **checklists pratiques et actionnables** pour auditer et optimiser les performances de vos déploiements MongoDB. Ces checklists sont conçues pour être utilisées à différents moments du cycle de vie de votre application :

- **Lors de la conception** : avant la mise en production
- **En production** : audits réguliers et diagnostics
- **Lors de problèmes** : identification rapide des goulots d'étranglement
- **Lors de la croissance** : anticipation des besoins de scaling

---

## Objectif et Utilisation

### 🎯 Objectif

Fournir un guide de référence rapide pour :
- Identifier les problèmes de performance courants
- Vérifier que les meilleures pratiques sont appliquées
- Diagnostiquer rapidement les ralentissements
- Prioriser les optimisations

### 📋 Comment utiliser ces checklists

1. **Parcourez chaque section** selon votre contexte
2. **Cochez les points vérifiés** ✅
3. **Identifiez les points critiques** ⚠️
4. **Priorisez les actions** selon l'impact
5. **Documentez vos conclusions** pour référence future

### 🔄 Fréquence recommandée

| Audit | Fréquence |
|-------|-----------|
| Modélisation | À la conception, puis lors de modifications majeures |
| Indexation | Mensuel ou lors de nouveaux patterns de requêtes |
| Requêtes | Hebdomadaire en développement, mensuel en production |
| Infrastructure | Trimestriel ou lors de problèmes de performance |

---

## Structure des Audits

Cette annexe est divisée en **4 audits complémentaires** :

### E.1 - Audit de Modélisation
**Objectif** : Vérifier que votre schéma de données est optimisé pour vos cas d'usage

**Points clés** :
- Relations et structures de documents
- Normalisation vs dénormalisation
- Patterns de modélisation appliqués
- Taille et croissance des documents

**Quand l'utiliser** : Lors de la conception initiale et lors de refactoring majeurs

---

### E.2 - Audit d'Indexation
**Objectif** : Optimiser les index pour maximiser les performances de lecture

**Points clés** :
- Couverture des requêtes par les index
- Index inutilisés ou redondants
- Stratégie d'indexation composée
- Impact sur les écritures

**Quand l'utiliser** : Régulièrement en production, surtout après ajout de nouvelles fonctionnalités

---

### E.3 - Audit de Requêtes
**Objectif** : Identifier et optimiser les requêtes lentes

**Points clés** :
- Requêtes non indexées
- Pipelines d'agrégation inefficaces
- Patterns anti-performants
- Utilisation du cache

**Quand l'utiliser** : Lors de ralentissements ou avant optimisation

---

### E.4 - Audit d'Infrastructure
**Objectif** : Vérifier la configuration matérielle et système

**Points clés** :
- Ressources (CPU, RAM, Disque, Réseau)
- Configuration MongoDB
- Architecture (Replica Set, Sharding)
- Monitoring et alertes

**Quand l'utiliser** : Lors du déploiement initial et lors de scaling

---

## Méthodologie d'Audit

### 1️⃣ Préparation

```markdown
✅ Définir le périmètre de l'audit
✅ Rassembler les métriques actuelles
✅ Identifier les objectifs de performance
✅ Préparer les outils de monitoring
```

### 2️⃣ Collecte de Données

**Commandes essentielles** :
```javascript
// Statistiques serveur
db.serverStatus()

// Statistiques de base
db.stats()

// Statistiques de collection
db.collection.stats()

// Opérations en cours
db.currentOp()

// Requêtes lentes (profiler)
db.system.profile.find().sort({ts: -1}).limit(10)
```

### 3️⃣ Analyse

- Comparez avec les **valeurs de référence** (baseline)
- Identifiez les **écarts significatifs**
- Priorisez selon l'**impact business**
- Estimez le **coût d'optimisation**

### 4️⃣ Action

- Appliquez les correctifs **par ordre de priorité**
- Testez en **environnement de staging**
- Mesurez l'**impact réel**
- Documentez les **changements**

### 5️⃣ Suivi

- Surveillez les **métriques clés**
- Ajustez si **nécessaire**
- Planifiez le **prochain audit**

---

## Niveaux de Priorité

Utilisez ce système pour classer vos actions :

| Niveau | Icône | Description | Action |
|--------|-------|-------------|--------|
| **Critique** | 🔴 | Impact majeur sur les performances ou la disponibilité | Correction immédiate requise |
| **Important** | 🟠 | Dégradation notable des performances | Planifier sous 1-2 semaines |
| **Modéré** | 🟡 | Optimisation bénéfique | Inclure dans le prochain cycle |
| **Mineur** | 🟢 | Amélioration marginale | Opportuniste |
| **Info** | 🔵 | Point d'information, pas d'action requise | Documentation |

---

## Outils Recommandés

### Monitoring en Temps Réel

```bash
# mongostat - Vue d'ensemble des opérations
mongostat --host localhost:27017

# mongotop - Temps passé par collection
mongotop --host localhost:27017 5

# Logs en temps réel
tail -f /var/log/mongodb/mongod.log
```

### Analyse de Performance

```javascript
// Activer le profiler (niveau 2 = toutes les requêtes)
db.setProfilingLevel(2)

// Analyser une requête spécifique
db.collection.find({...}).explain("executionStats")

// Vérifier les index utilisés
db.collection.find({...}).explain("allPlansExecution")
```

### MongoDB Compass

- **Visual Explain Plan** : analyse graphique des requêtes
- **Index Performance** : suggestions d'index
- **Schema Analysis** : analyse de la structure

### Atlas (Cloud)

- **Performance Advisor** : recommandations automatiques
- **Query Profiler** : interface visuelle des requêtes lentes
- **Real-time Performance Panel** : monitoring en direct

---

## Métriques Clés à Surveiller

### Performances Générales

| Métrique | Valeur Cible | Outil |
|----------|--------------|-------|
| Opérations/sec | Selon charge | `mongostat` |
| Latence moyenne | < 10ms (lecture) | Logs, Atlas |
| Latence P95 | < 50ms | Profiler |
| Latence P99 | < 100ms | Profiler |
| Cache hit ratio | > 95% | `serverStatus` |

### Ressources Système

| Métrique | Valeur Cible | Outil |
|----------|--------------|-------|
| CPU | < 70% | `top`, `htop` |
| RAM utilisée | < 80% | `free -h` |
| Working Set | < RAM disponible | `serverStatus` |
| I/O Wait | < 10% | `iostat` |
| Connexions | < 80% du max | `serverStatus` |

### Index et Requêtes

| Métrique | Valeur Cible | Outil |
|----------|--------------|-------|
| Index scans/s | > 90% des ops | `mongostat` |
| Collection scans | Minimiser | Profiler |
| Index size | < RAM | `db.collection.stats()` |
| Slow queries | < 1% | Profiler |

---

## Workflow d'Optimisation

```
┌─────────────────────┐
│  Identifier le      │
│  goulot             │
│  d'étranglement     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Mesurer l'état     │
│  actuel (baseline)  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Appliquer          │
│  l'optimisation     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Mesurer            │
│  l'amélioration     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│  Documenter et      │
│  déployer           │
└─────────────────────┘
```

---

## Points d'Attention Généraux

### ⚠️ Pièges Courants

1. **Sur-indexation** : trop d'index ralentissent les écritures
2. **Absence d'index** : scans complets de collections
3. **Requêtes N+1** : multiples requêtes au lieu de jointures/agrégations
4. **Working set > RAM** : swapping et ralentissements
5. **Connexions non poolées** : overhead de connexion
6. **Absence de monitoring** : problèmes détectés trop tard

### ✅ Meilleures Pratiques

1. **Mesurez avant d'optimiser** : pas d'optimisation prématurée
2. **Optimisez par ordre d'impact** : gains rapides vs efforts
3. **Testez en staging** : avant tout déploiement en production
4. **Documentez les changements** : pour référence future
5. **Automatisez le monitoring** : alertes proactives
6. **Revisitez régulièrement** : les patterns changent

---

## Seuils d'Alerte Recommandés

### 🚨 Alertes Critiques

```javascript
// CPU > 90% pendant 5 minutes
// RAM > 95% pendant 3 minutes
// Disk I/O wait > 50% pendant 2 minutes
// Connexions > 90% du maximum
// Replication lag > 10 secondes
// Oplog < 1 heure de données
```

### ⚠️ Alertes Warning

```javascript
// CPU > 70% pendant 10 minutes
// RAM > 80% pendant 5 minutes
// Cache hit ratio < 90%
// Slow queries > 100ms
// Index size > 50% de la RAM
// Collection scans > 10% des requêtes
```

---

## Ressources Complémentaires

### Documentation Officielle
- [MongoDB Performance Best Practices](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)
- [Production Notes](https://www.mongodb.com/docs/manual/administration/production-notes/)
- [Query Plans and Performance](https://www.mongodb.com/docs/manual/core/query-plans/)

### Outils Externes
- **Percona Monitoring and Management (PMM)** : monitoring avancé
- **MongoDB Compass** : analyse visuelle
- **mongo-perf** : benchmarking

### Formations
- MongoDB University : M201 (MongoDB Performance)
- MongoDB University : M312 (Diagnostics and Debugging)

---

## Structure de cette Annexe

Cette annexe contient les 4 audits suivants :

1. **[E.1 - Audit de Modélisation](./01-audit-modelisation.md)**
   - Vérification de la structure des documents
   - Relations et références
   - Patterns appliqués

2. **[E.2 - Audit d'Indexation](./02-audit-indexation.md)**
   - Couverture des index
   - Index inutilisés
   - Stratégie d'indexation

3. **[E.3 - Audit de Requêtes](./03-audit-requetes.md)**
   - Analyse des requêtes lentes
   - Optimisation des agrégations
   - Patterns de requêtes

4. **[E.4 - Audit d'Infrastructure](./04-audit-infrastructure.md)**
   - Configuration système
   - Ressources matérielles
   - Architecture distribuée

---

## Exemple d'Utilisation

### Scénario : Application de e-commerce ralentie

**1. Symptômes observés**
- Temps de réponse de 2-3 secondes sur la page produit
- CPU à 85% en permanence
- Plaintes utilisateurs

**2. Audits à effectuer dans l'ordre**

```markdown
✅ E.3 - Audit de Requêtes
   → Identifier les requêtes lentes spécifiques

✅ E.2 - Audit d'Indexation
   → Vérifier si les requêtes sont indexées

✅ E.1 - Audit de Modélisation
   → Vérifier la structure des documents produits

✅ E.4 - Audit d'Infrastructure
   → Vérifier les ressources système
```

**3. Actions prises**
- Ajout d'index composé sur `{category: 1, price: 1}`
- Activation du cache de requêtes
- Optimisation du pipeline d'agrégation des recommandations
- Augmentation de la RAM de 8GB à 16GB

**4. Résultats**
- Temps de réponse : **300ms** (division par 7)
- CPU moyen : **35%** (gain de 50%)
- Satisfaction utilisateurs restaurée

---

## Note Importante

> **Ces checklists sont des guides, pas des règles absolues.**
>
> Chaque application a des besoins spécifiques. Adaptez ces recommandations à votre contexte : charge, données, contraintes business, et ressources disponibles.
>
> L'optimisation est un **processus continu**, pas une action ponctuelle.

---

**Version** : 1.0
**Compatible avec** : MongoDB 6.x, 7.x, 8.x

⏭️ [Audit de modélisation](/annexes/checklist-performance/01-audit-modelisation.md)
