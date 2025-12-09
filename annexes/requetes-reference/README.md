🔝 Retour au [Sommaire](/SOMMAIRE.md)

# Annexe C : Requêtes MongoDB de Référence

## Vue d'ensemble

Cette annexe est une **bibliothèque de requêtes MongoDB prêtes à l'emploi**, organisée par cas d'usage. Elle contient des exemples concrets et testés que vous pouvez copier, adapter et utiliser immédiatement dans vos projets.

---

## Philosophie de cette annexe

### 🎯 Objectifs

- **Copier-coller immédiat** : Toutes les requêtes sont fonctionnelles et testées
- **Cas d'usage réels** : Exemples issus de situations de production courantes
- **Progressif** : Du simple au complexe, adapté à tous les niveaux
- **Commenté** : Chaque requête est expliquée et documentée
- **Best practices** : Les requêtes suivent les bonnes pratiques MongoDB

---

## Organisation de l'annexe

Cette référence est divisée en trois sections thématiques :

### C.1 - Requêtes d'administration
Requêtes pour gérer et surveiller votre infrastructure MongoDB :
- **Utilisation des index** : Identifier les index inutilisés ou sous-utilisés
- **Statistiques de collections** : Analyser la taille, le nombre de documents
- **Performance du serveur** : Métriques système et ressources
- **Gestion des utilisateurs** : Audit des permissions et rôles
- **Monitoring du Replica Set** : Santé et synchronisation
- **Monitoring du Sharding** : Distribution des données et balancer

### C.2 - Requêtes de monitoring
Requêtes pour surveiller la santé et les performances en temps réel :
- **Opérations en cours** : Identifier les opérations longues ou bloquantes
- **Profiler de requêtes** : Analyser les requêtes lentes
- **Métriques de performance** : CPU, mémoire, disque, réseau
- **Connexions actives** : Surveiller l'utilisation des connexions
- **Réplication lag** : Détecter les retards de synchronisation
- **Alerting** : Requêtes pour déclencher des alertes

### C.3 - Pipelines d'agrégation courants
Collections de pipelines réutilisables pour l'analyse de données :
- **Statistiques de base** : Compter, moyennes, sommes
- **Groupements et agrégations** : Par date, catégorie, région
- **Jointures (lookups)** : Combiner des collections liées
- **Transformations de données** : Restructuration, calculs
- **Analyses temporelles** : Tendances, historiques, time series
- **Rapports complexes** : Tableaux de bord, KPIs

---

## Comment utiliser cette annexe

### 📖 Pour les débutants

1. **Commencez par les requêtes simples** de chaque section
2. **Lisez les commentaires** pour comprendre chaque partie
3. **Testez sur des données non-critiques** avant la production
4. **Adaptez progressivement** à vos besoins spécifiques

### 🔧 Pour les développeurs

1. **Copiez la requête** qui correspond à votre besoin
2. **Remplacez les noms** de collections et champs
3. **Ajustez les filtres** selon vos critères
4. **Testez avec explain()** pour vérifier les performances
5. **Créez les index nécessaires** si identifiés

### 🚀 Pour les administrateurs

1. **Utilisez les requêtes d'administration** pour le monitoring
2. **Automatisez** les requêtes récurrentes via scripts
3. **Créez des dashboards** avec les métriques importantes
4. **Définissez des seuils d'alerte** basés sur les résultats
5. **Documentez** les requêtes personnalisées dans vos runbooks

### 💼 Pour les data analysts

1. **Explorez les pipelines d'agrégation** pour vos analyses
2. **Combinez plusieurs pipelines** pour des rapports complexes
3. **Optimisez** avec les index appropriés
4. **Exportez les résultats** vers vos outils BI
5. **Créez des vues** pour les analyses récurrentes

---

## Conventions utilisées

### Notation des requêtes

```javascript
// 📌 Titre de la requête
// 💡 Description de l'objectif
// ⚡ Note sur les performances
// 🔑 Index requis (si applicable)

db.collection.method({
  // Requête avec commentaires
})

// 📊 Résultat attendu
// Exemple de sortie
```

### Placeholders

Dans toutes les requêtes, remplacez :
- `<collection>` : Nom de votre collection
- `<field>` : Nom du champ
- `<value>` : Valeur à rechercher
- `<date>` : Date au format ISODate
- `<number>` : Valeur numérique
- `{...}` : Conditions supplémentaires

### Niveaux de complexité

Chaque requête est annotée avec son niveau :

| Symbole | Niveau | Description |
|---------|--------|-------------|
| 🟢 | Débutant | Requête simple, concepts de base |
| 🟡 | Intermédiaire | Requête avancée, concepts multiples |
| 🔴 | Avancé | Requête complexe, optimisation requise |
| ⚫ | Expert | Production uniquement, expertise nécessaire |

### Indicateurs de performance

| Symbole | Signification |
|---------|---------------|
| ⚡ | Requête rapide (< 100ms typiquement) |
| ⏱️ | Requête modérée (< 1s) |
| 🐌 | Requête lente (> 1s, optimisation nécessaire) |
| 🔥 | Requête intensive (impact sur le serveur) |

### Index requis

Quand un index est recommandé ou obligatoire :

```javascript
// 🔑 Index requis
db.collection.createIndex({ field: 1 })

// 🔑 Index composé recommandé
db.collection.createIndex({ field1: 1, field2: -1 })
```

---

## Structure type d'une requête

Chaque requête de cette annexe suit cette structure :

```javascript
// ============================================
// 📌 NOM DE LA REQUÊTE
// ============================================

// 💡 Objectif
// Description claire de ce que fait la requête

// 🎯 Cas d'usage
// Quand utiliser cette requête

// 🔑 Index recommandé (si applicable)
// db.collection.createIndex({...})

// ⚡ Performance
// Note sur les performances et impact

// ============================================
// REQUÊTE
// ============================================

db.collection.operation({
  // Filtre avec commentaires
  field: value  // Explication du filtre
}, {
  // Projection (si applicable)
  field1: 1,
  field2: 1,
  _id: 0
})
.sort({ field: -1 })  // Tri
.limit(10)            // Limitation

// ============================================
// 📊 RÉSULTAT ATTENDU
// ============================================

// Exemple de sortie
[
  { field1: "value1", field2: 100 },
  { field1: "value2", field2: 200 }
]

// ============================================
// 💡 NOTES ET VARIANTES
// ============================================

// Variante 1 : Description
// Code de la variante

// Variante 2 : Description
// Code de la variante

// ⚠️ Points d'attention
// Avertissements et précautions
```

---

## Catégories de requêtes

### Par fonctionnalité

| Catégorie | Section | Nombre de requêtes |
|-----------|---------|-------------------|
| Administration | C.1 | ~30 requêtes |
| Monitoring | C.2 | ~25 requêtes |
| Agrégation | C.3 | ~40 requêtes |
| **Total** | | **~95 requêtes** |

### Par niveau de difficulté

| Niveau | Pourcentage | Public cible |
|--------|-------------|--------------|
| 🟢 Débutant | 30% | Tous utilisateurs |
| 🟡 Intermédiaire | 40% | Développeurs, admins |
| 🔴 Avancé | 25% | Experts, production |
| ⚫ Expert | 5% | Spécialistes uniquement |

---

## Bonnes pratiques d'utilisation

### ✅ À faire

```javascript
// ✅ Tester d'abord avec limit()
db.collection.find(query).limit(10)

// ✅ Utiliser explain() pour vérifier les performances
db.collection.find(query).explain("executionStats")

// ✅ Créer les index nécessaires avant les grosses requêtes
db.collection.createIndex({ field: 1 })

// ✅ Utiliser des projections pour limiter les données
db.collection.find(query, { field1: 1, field2: 1, _id: 0 })

// ✅ Commenter vos requêtes pour la maintenance
db.collection.find(query).comment("Description de la requête")

// ✅ Sauvegarder les requêtes complexes dans des scripts
```

### ❌ À éviter

```javascript
// ❌ Requêtes sans limite sur de grandes collections
db.hugeLogs.find()  // Peut charger des millions de documents

// ❌ Regex sans ancrage au début (lent)
db.collection.find({ field: /pattern/ })  // Scan complet

// ❌ $where avec JavaScript (très lent)
db.collection.find({ $where: "this.field > 100" })

// ❌ Agrégations sans $match en premier
db.collection.aggregate([
  { $sort: { date: -1 } },  // ❌ Pas de filtre avant le tri
  { $match: { status: "active" } }
])

// ❌ Multiples requêtes quand une agrégation suffirait
// ❌ Ne pas vérifier l'existence d'index avant requêtes lourdes
```

---

## Performance et optimisation

### 🎯 Règles d'or

1. **Toujours utiliser des index** pour les filtres fréquents
2. **$match en premier** dans les pipelines d'agrégation
3. **Limiter les résultats** avec limit() quand possible
4. **Projeter uniquement** les champs nécessaires
5. **Tester avec explain()** avant mise en production
6. **Monitorer** l'utilisation des index (C.1)
7. **Profiler** les requêtes lentes (C.2)

### ⚡ Checklist de performance

Avant d'exécuter une requête en production :

- [ ] Index appropriés créés
- [ ] Requête testée avec explain()
- [ ] Projection limitée aux champs nécessaires
- [ ] Limite (limit) définie si approprié
- [ ] Pipeline d'agrégation optimisé ($match en premier)
- [ ] Testé sur un échantillon de données
- [ ] Impact sur le serveur évalué (currentOp)
- [ ] Plan de rollback en cas de problème

---

## Personnalisation des requêtes

### Adaptation à votre schéma

Toutes les requêtes de cette annexe utilisent des noms génériques. Voici comment les adapter :

```javascript
// ========== EXEMPLE GÉNÉRIQUE ==========
db.users.find({
  status: "active",
  age: { $gte: 18 }
})

// ========== VOTRE ADAPTATION ==========
db.customers.find({
  accountStatus: "verified",     // Votre champ status
  membershipAge: { $gte: 30 }    // Votre champ age
})
```

### Création de bibliothèques personnalisées

```javascript
// 📁 my-queries.js
// Bibliothèque personnalisée de requêtes

const MyQueries = {
  // Requêtes utilisateurs
  activeUsers: () => db.users.find({ active: true }),

  // Requêtes commandes
  recentOrders: (days) => db.orders.find({
    createdAt: { $gte: new Date(Date.now() - days * 86400000) }
  }),

  // Agrégations
  salesByMonth: () => db.orders.aggregate([
    { $group: {
        _id: { $dateToString: { format: "%Y-%m", date: "$date" } },
        total: { $sum: "$amount" }
    }},
    { $sort: { _id: -1 } }
  ])
};

// Usage
MyQueries.activeUsers()
MyQueries.recentOrders(7)
MyQueries.salesByMonth()
```

---

## Intégration avec les outils

### MongoDB Compass

1. Copiez la requête depuis cette annexe
2. Collez dans l'onglet "Documents" → Filter
3. Utilisez l'onglet "Aggregations" pour les pipelines
4. Exportez les résultats au format JSON/CSV

### Scripts d'automatisation

```javascript
// monitoring-script.js
// Charge les requêtes de monitoring automatiquement

load("annexes/requetes-reference/02-requetes-monitoring.md");

// Exécuter toutes les heures via cron
// 0 * * * * mongosh --quiet monitoring-script.js
```

### Intégration dans le code applicatif

```javascript
// Node.js avec le driver MongoDB
const { MongoClient } = require('mongodb');

// Copier la requête de l'annexe
const query = { status: "active", age: { $gte: 18 } };
const projection = { name: 1, email: 1, _id: 0 };

const results = await collection
  .find(query)
  .project(projection)
  .limit(10)
  .toArray();
```

---

## Maintenance et mises à jour

### Versions MongoDB supportées

Les requêtes de cette annexe sont compatibles avec :
- ✅ MongoDB 6.0+
- ✅ MongoDB 7.0+
- ✅ MongoDB 8.0+

Les fonctionnalités spécifiques à certaines versions sont annotées.

### Mises à jour de l'annexe

Cette annexe est régulièrement mise à jour pour :
- Ajouter de nouvelles requêtes courantes
- Optimiser les requêtes existantes
- Supporter les nouvelles fonctionnalités MongoDB
- Corriger les erreurs ou imprécisions

**Dernière mise à jour** : Novembre 2025

---

## Ressources complémentaires

### Documentation officielle

- [MongoDB Query Documentation](https://www.mongodb.com/docs/manual/tutorial/query-documents/)
- [Aggregation Pipeline](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/)
- [Performance Best Practices](https://www.mongodb.com/docs/manual/administration/analyzing-mongodb-performance/)

### Outils recommandés

| Outil | Usage | Lien |
|-------|-------|------|
| **MongoDB Compass** | GUI pour tester les requêtes | [Télécharger](https://www.mongodb.com/products/compass) |
| **Studio 3T** | IDE avancé avec IntelliQuery | [Site officiel](https://studio3t.com/) |
| **NoSQLBooster** | Client avec auto-complétion SQL | [Site officiel](https://nosqlbooster.com/) |
| **mongosh** | Shell officiel | Intégré à MongoDB |

### Formations complémentaires

- MongoDB University (gratuit)
- Certification MongoDB Developer
- Certification MongoDB DBA

---

## Structure de navigation

Cette annexe contient les sections suivantes :

### 📊 Sections principales

- **[C.1 - Requêtes d'administration](./01-requetes-administration.md)**
  - Gestion des index
  - Statistiques de collections
  - Monitoring du serveur
  - Audit des utilisateurs
  - Replica Set et Sharding

- **[C.2 - Requêtes de monitoring](./02-requetes-monitoring.md)**
  - Opérations en cours
  - Profiler de requêtes
  - Métriques temps réel
  - Connexions et ressources
  - Alerting

- **[C.3 - Pipelines d'agrégation courants](./03-pipelines-agregation-courants.md)**
  - Statistiques de base
  - Groupements et analyses
  - Jointures ($lookup)
  - Transformations de données
  - Analyses temporelles
  - Rapports complexes

---

## Exemples d'utilisation

### Scénario 1 : Identifier les requêtes lentes

```javascript
// 1. Consulter C.2 - Requêtes de monitoring
// 2. Copier la requête "Requêtes lentes du profiler"
db.system.profile.find({ millis: { $gt: 100 } })
  .sort({ millis: -1 })
  .limit(10)

// 3. Analyser les résultats
// 4. Consulter C.1 pour vérifier les index
```

### Scénario 2 : Créer un rapport de ventes mensuel

```javascript
// 1. Consulter C.3 - Pipelines d'agrégation
// 2. Copier le pipeline "Ventes par mois"
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $group: {
      _id: { $dateToString: { format: "%Y-%m", date: "$date" } },
      total: { $sum: "$amount" },
      count: { $sum: 1 }
  }},
  { $sort: { _id: -1 } }
])

// 3. Adapter à votre schéma
// 4. Créer une vue pour réutilisation
```

### Scénario 3 : Audit de sécurité

```javascript
// 1. Consulter C.1 - Requêtes d'administration
// 2. Copier "Lister tous les utilisateurs avec leurs rôles"
db.getSiblingDB("admin").system.users.find()

// 3. Exporter les résultats
// 4. Analyser les permissions
```

---

## FAQ - Foire Aux Questions

### Q: Puis-je utiliser ces requêtes en production directement ?

**R:** Oui, mais toujours après :
1. Test sur un environnement de dev/staging
2. Vérification avec explain()
3. Création des index nécessaires
4. Validation de l'impact avec currentOp()

### Q: Comment savoir quelle requête utiliser ?

**R:**
1. Identifiez votre besoin (administration, monitoring, analyse)
2. Consultez la section correspondante (C.1, C.2, ou C.3)
3. Lisez les descriptions "Cas d'usage"
4. Choisissez la requête la plus proche
5. Adaptez à votre schéma

### Q: Les requêtes sont-elles optimisées ?

**R:** Oui, toutes les requêtes suivent les bonnes pratiques MongoDB. Cependant, l'optimisation finale dépend de :
- Votre schéma de données
- Vos index existants
- La taille de vos collections
- Votre architecture (standalone, replica set, sharded)

### Q: Comment contribuer avec mes propres requêtes ?

**R:** Les requêtes de cette annexe sont issues de cas d'usage réels et de bonnes pratiques communautaires. Documentez vos requêtes utiles dans votre propre bibliothèque en suivant le format de cette annexe.

---

## Notes importantes

> **⚠️ Environnements de production**
> - Testez toujours en dev/staging d'abord
> - Vérifiez l'impact avec explain() et currentOp()
> - Créez les index nécessaires avant les requêtes lourdes
> - Planifiez les maintenances pour les opérations impactantes
> - Gardez des backups à jour

> **💡 Optimisation**
> - Une requête lente n'est pas toujours un problème de code
> - Vérifiez d'abord si les index appropriés existent
> - Considérez la modélisation de données
> - Surveillez la croissance des collections

> **🔐 Sécurité**
> - Les requêtes d'administration nécessitent des privilèges
> - Ne loggez jamais les credentials dans les scripts
> - Utilisez des variables d'environnement pour les connexions
> - Auditez régulièrement les permissions

---

## Prêt à explorer les requêtes ?

Commencez par la section qui correspond à votre besoin :

- 🔧 **Administration** → [C.1 - Requêtes d'administration](./01-requetes-administration.md)
- 📊 **Monitoring** → [C.2 - Requêtes de monitoring](./02-requetes-monitoring.md)
- 📈 **Analyse de données** → [C.3 - Pipelines d'agrégation](./03-pipelines-agregation-courants.md)

---

**Bonne exploration ! 🚀**

*Cette annexe est conçue pour être votre compagnon quotidien dans le travail avec MongoDB. Ajoutez-la à vos favoris !*

⏭️ [Requêtes d'administration (index usage, collection stats)](/annexes/requetes-reference/01-requetes-administration.md)
