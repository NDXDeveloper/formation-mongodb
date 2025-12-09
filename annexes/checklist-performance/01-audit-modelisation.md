🔝 Retour au [Sommaire](/SOMMAIRE.md)

# E.1 - Audit de Modélisation

## Introduction

L'audit de modélisation évalue la **structure de vos documents** et la manière dont vos données sont organisées. Une bonne modélisation est la **fondation des performances** dans MongoDB.

### 🎯 Objectif

Vérifier que votre schéma de données est optimisé pour vos patterns d'accès réels et qu'il respecte les contraintes techniques de MongoDB.

### ⏱️ Durée estimée
- Audit rapide : 30 minutes
- Audit complet : 2-4 heures

---

## Checklist Générale

### 📊 Structure des Documents

#### ✅ Taille des Documents

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Documents < 16 Mo (limite BSON) | 🔴 | Obligatoire |
| Documents < 1 Mo (recommandé) | 🟠 | Fortement recommandé |
| Documents < 100 Ko (idéal) | 🟡 | Optimal pour la performance |
| Croissance des documents contrôlée | 🟠 | Éviter l'éclatement (document relocation) |

**Commande de vérification** :
```javascript
// Taille moyenne des documents
db.collection.stats().avgObjSize

// Taille maximale (approximative via échantillon)
db.collection.aggregate([
  { $sample: { size: 1000 } },
  { $project: { size: { $bsonSize: "$$ROOT" } } },
  { $sort: { size: -1 } },
  { $limit: 10 }
])

// Identifier les documents volumineux
db.collection.find().sort({ $natural: -1 }).limit(10).forEach(doc => {
  print(JSON.stringify(doc).length + " bytes - " + doc._id);
});
```

**Symptômes de problème** :
- ⚠️ Documents > 1 Mo régulièrement
- ⚠️ Croissance continue de la taille moyenne
- ⚠️ Logs indiquant document relocation

**Actions correctives** :
- Déplacer les grandes valeurs vers GridFS (fichiers > 16 Mo)
- Utiliser le pattern **Subset** pour limiter les tableaux
- Référencer les données volumineuses au lieu de les imbriquer
- Archiver les anciennes données

---

#### ✅ Profondeur d'Imbrication

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Profondeur < 100 niveaux (limite) | 🔴 | Obligatoire |
| Profondeur < 5 niveaux (recommandé) | 🟠 | Éviter la complexité |
| Pas d'imbrication dynamique infinie | 🟠 | Risque d'explosion |

**Bonnes pratiques** :
```javascript
// ❌ Mauvais : trop profond
{
  level1: {
    level2: {
      level3: {
        level4: {
          level5: {
            data: "value"
          }
        }
      }
    }
  }
}

// ✅ Bon : structure aplatie ou référencée
{
  path: "level1.level2.level3.level4.level5",
  data: "value"
}
```

---

#### ✅ Tableaux dans les Documents

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Tableaux < 1000 éléments (guideline) | 🟠 | Performance des requêtes |
| Pas de croissance illimitée | 🔴 | Utiliser le pattern Bucket |
| Index multikey pertinent | 🟡 | Si requêtes sur tableaux |

**Commande de vérification** :
```javascript
// Vérifier la taille des tableaux
db.collection.aggregate([
  { $project: {
      arrayField: 1,
      arraySize: { $size: { $ifNull: ["$arrayField", []] } }
    }
  },
  { $sort: { arraySize: -1 } },
  { $limit: 10 }
])
```

**Patterns recommandés** :
```javascript
// ❌ Mauvais : tableau sans limite
{
  _id: 1,
  comments: [ /* potentiellement des milliers */ ]
}

// ✅ Bon : pattern Subset
{
  _id: 1,
  recentComments: [ /* 10 derniers */ ],
  totalComments: 5247
}

// ✅ Bon : pattern Bucket
{
  _id: 1,
  bucket: "2024-01",
  comments: [ /* commentaires du mois */ ]
}
```

---

### 🔗 Relations et Références

#### ✅ Choix Embedded vs Referenced

| Relation | Embedded si... | Referenced si... |
|----------|----------------|------------------|
| **One-to-One** | Accès toujours ensemble | Données volumineuses ou rarement accédées |
| **One-to-Few** | < 100 éléments, lecture fréquente | Éléments indépendants |
| **One-to-Many** | Many < 1000, pas de croissance | Many > 1000 ou croissance illimitée |
| **Many-to-Many** | Rarement | Toujours (ou presque) |

**Checklist** :
```markdown
✅ Documents liés accédés ensemble → Embedded
✅ Documents indépendants → Referenced
✅ Relation 1:N avec N > 1000 → Referenced
✅ Relation Many-to-Many → Referenced
✅ Duplication de données acceptable → Embedded (pattern Extended Reference)
✅ Besoin d'intégrité référentielle stricte → Referenced + validation
```

**Exemples** :

```javascript
// ✅ Bon : Embedded pour adresse (One-to-One)
{
  _id: 1,
  name: "Jean Dupont",
  address: {
    street: "123 rue de la Paix",
    city: "Paris",
    zip: "75001"
  }
}

// ✅ Bon : Referenced pour commandes (One-to-Many illimité)
// Collection users
{
  _id: 1,
  name: "Jean Dupont"
}

// Collection orders
{
  _id: 101,
  userId: 1,
  date: ISODate("2024-01-15"),
  total: 150.00
}

// ✅ Bon : Extended Reference (duplication partielle)
{
  _id: 101,
  user: {
    _id: 1,
    name: "Jean Dupont"  // Duplication pour performance
  },
  date: ISODate("2024-01-15"),
  total: 150.00
}
```

---

#### ✅ Intégrité Référentielle

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Validation de schéma en place | 🟡 | Pour contraintes critiques |
| Gestion des orphelins | 🟠 | Scripts de nettoyage |
| Cascade delete géré | 🟡 | Si nécessaire |

**Vérification des références cassées** :
```javascript
// Trouver les commandes avec userId inexistant
db.orders.aggregate([
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  },
  {
    $match: { user: { $size: 0 } }
  }
])
```

---

### 📐 Patterns de Modélisation

#### ✅ Patterns Appliqués

Vérifiez quels patterns sont utilisés et s'ils sont appropriés :

| Pattern | Cas d'usage | Checklist |
|---------|-------------|-----------|
| **Embedded** | Données toujours accédées ensemble | ✅ Documents < 1 Mo<br>✅ Pas de croissance illimitée |
| **Subset** | Tableaux volumineux | ✅ Top-N facilement identifiable<br>✅ Compteur total présent |
| **Extended Reference** | Jointures fréquentes | ✅ Champs dupliqués essentiels uniquement<br>✅ Stratégie de synchronisation |
| **Bucket** | Séries temporelles, IoT | ✅ Fenêtre temporelle définie<br>✅ Taille bucket prévisible |
| **Computed** | Agrégations coûteuses | ✅ Calculs rares ou batch<br>✅ Mécanisme de mise à jour |
| **Attribute** | Schéma polymorphe | ✅ Vraiment nécessaire<br>✅ Index Wildcard si requis |
| **Outlier** | Cas exceptionnels | ✅ Seuil défini<br>✅ Gestion dual-strategy |
| **Schema Versioning** | Évolution schéma | ✅ Champ version présent<br>✅ Migration planifiée |

**Vérification de l'utilisation des patterns** :
```javascript
// Exemple : vérifier si pattern Subset est bien appliqué
db.products.aggregate([
  {
    $project: {
      hasRecentReviews: { $ifNull: ["$recentReviews", false] },
      hasTotalCount: { $ifNull: ["$totalReviews", false] },
      reviewsCount: { $size: { $ifNull: ["$recentReviews", []] } }
    }
  },
  {
    $match: {
      $or: [
        { hasRecentReviews: true, hasTotalCount: false },
        { reviewsCount: { $gt: 50 } }
      ]
    }
  }
])
```

---

### 🚫 Anti-Patterns à Éviter

#### ❌ Anti-Patterns Courants

| Anti-Pattern | Impact | Solution |
|--------------|--------|----------|
| **Tableaux sans limite** | Documents > 16 Mo, performance | Pattern Bucket ou référence |
| **Imbrication excessive** | Complexité, maintien | Aplatir ou référencer |
| **Duplication non contrôlée** | Incohérence données | Extended Reference avec stratégie |
| **Collections par type** | Explosion collections | Pattern Polymorphic |
| **Normalisation excessive** | Multiples requêtes | Embedded ou Extended Reference |
| **Schéma rigide sans évolution** | Migration complexe | Schema Versioning |
| **Mono-document géant** | Contention, performance | Splitter en documents |

**Checklist de détection** :
```markdown
❌ Tableaux qui grossissent indéfiniment (logs, comments, events)
❌ Documents qui approchent 16 Mo
❌ Requêtes nécessitant 5+ lookups
❌ Collections avec < 100 documents (sur-normalisation)
❌ Duplication manuelle sans synchronisation
❌ Pas de champ `schemaVersion` sur schémas évolutifs
❌ Documents hétérogènes sans pattern Attribute
```

---

### 📈 Croissance et Évolution

#### ✅ Anticipation de la Croissance

| Point de vérification | Priorité | Action |
|----------------------|----------|--------|
| Estimation de croissance documentée | 🟠 | Planification capacity |
| Tableaux bornés | 🔴 | Éviter débordement |
| Stratégie d'archivage définie | 🟡 | Pour données historiques |
| Migration de schéma planifiée | 🟡 | Schema Versioning |

**Calculs de projection** :
```javascript
// Analyser la croissance sur 30 jours
db.collection.aggregate([
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
      count: { $sum: 1 },
      avgSize: { $avg: { $bsonSize: "$$ROOT" } }
    }
  },
  { $sort: { _id: 1 } }
])

// Projection simple
// Si +1000 docs/jour avec 10 Ko/doc = +10 Mo/jour = +3.6 Go/an
```

---

## Checklist par Type d'Application

### 🛒 E-Commerce

```markdown
✅ Produits : Embedded (specs, images limitées) + Referenced (reviews)
✅ Panier : Embedded (items array bornée à ~100)
✅ Commandes : Extended Reference (user info dupliquée)
✅ Inventaire : Document séparé avec transactions
✅ Recherche : Index texte ou Atlas Search
```

### 📱 Application Mobile/Web

```markdown
✅ Utilisateurs : Embedded (profile, preferences)
✅ Sessions : TTL Index pour nettoyage auto
✅ Notifications : Capped Collection ou Bucket pattern
✅ Analytics : Time Series Collections (MongoDB 5.0+)
✅ Cache : Documents avec TTL
```

### 📊 IoT / Séries Temporelles

```markdown
✅ Mesures : Bucket pattern (par heure/jour)
✅ Métadonnées device : Embedded dans buckets
✅ Agrégations : Pattern Computed pour stats pré-calculées
✅ Alertes : Change Streams + Pattern Outlier
✅ Archivage : Time Series Collections + Atlas Data Lake
```

### 📝 CMS / Blog

```markdown
✅ Articles : Embedded (metadata) + GridFS (medias)
✅ Commentaires : Pattern Subset (10 derniers) + collection séparée
✅ Tags : Embedded array (< 50 tags)
✅ Auteurs : Referenced (ou Extended Reference)
✅ Versions : Schema Versioning ou document separé
```

---

## Outils d'Analyse

### 🔍 MongoDB Compass

**Schema Tab** :
- Analyse automatique de la structure
- Distribution des types
- Détection de patterns

**Explain Plan** :
- Impact de la modélisation sur les requêtes
- Suggestions d'amélioration

### 📊 Commandes Shell

```javascript
// 1. Analyser la structure des documents
db.collection.findOne()

// 2. Distribution des champs
db.collection.aggregate([
  { $project: {
      fields: { $objectToArray: "$$ROOT" }
    }
  },
  { $unwind: "$fields" },
  { $group: {
      _id: "$fields.k",
      count: { $sum: 1 },
      types: { $addToSet: { $type: "$fields.v" } }
    }
  }
])

// 3. Profondeur maximale (simplifié)
function getDepth(obj, currentDepth = 0) {
  if (typeof obj !== 'object' || obj === null) return currentDepth;
  return Math.max(...Object.values(obj).map(v => getDepth(v, currentDepth + 1)));
}

// 4. Cohérence du schéma
db.collection.aggregate([
  {
    $project: {
      fieldCount: { $size: { $objectToArray: "$$ROOT" } }
    }
  },
  {
    $group: {
      _id: "$fieldCount",
      count: { $sum: 1 }
    }
  }
])
```

### 📈 Scripts d'Audit

```javascript
// Script complet d'audit de collection
function auditCollection(collectionName) {
  const coll = db.getCollection(collectionName);
  const stats = coll.stats();

  print("=== Audit de " + collectionName + " ===");
  print("Documents totaux: " + stats.count);
  print("Taille moyenne: " + stats.avgObjSize + " bytes");
  print("Taille totale: " + (stats.size / 1024 / 1024).toFixed(2) + " Mo");

  // Échantillon pour analyse
  const sample = coll.aggregate([{ $sample: { size: 100 } }]).toArray();

  // Taille max
  const maxSize = Math.max(...sample.map(doc => JSON.stringify(doc).length));
  print("Taille max (échantillon): " + maxSize + " bytes");

  // Champs manquants
  const firstDoc = coll.findOne();
  const allFields = Object.keys(firstDoc);

  allFields.forEach(field => {
    const missing = coll.countDocuments({ [field]: { $exists: false } });
    if (missing > 0) {
      print("Champ '" + field + "' manquant dans " + missing + " documents");
    }
  });
}

// Utilisation
auditCollection("products");
```

---

## Matrice de Décision Rapide

### Embedded vs Referenced

```
Embedded si :
├─ Documents accédés toujours ensemble
├─ Relation 1:1 ou 1:Few (< 100)
├─ Données rarement modifiées séparément
├─ Performance lecture prioritaire
└─ Taille totale < 1 Mo

Referenced si :
├─ Documents accédés indépendamment
├─ Relation 1:Many (> 1000)
├─ Relation Many-to-Many
├─ Données modifiées fréquemment
├─ Réutilisation dans plusieurs contextes
└─ Risque de dépassement 16 Mo
```

### Pattern à Utiliser

```
Subset → Tableaux volumineux avec accès "top-N"
Bucket → Données temporelles ou séquentielles
Extended Reference → Jointures très fréquentes
Computed → Agrégations coûteuses répétées
Attribute → Schéma hautement variable
Outlier → Quelques cas très différents
Schema Versioning → Évolution schéma progressive
```

---

## Actions Prioritaires

### 🔴 Critique - À corriger immédiatement

```markdown
□ Documents approchant ou dépassant 16 Mo
□ Tableaux croissant sans limite définie
□ Références cassées impactant l'application
□ Profondeur > 20 niveaux d'imbrication
```

### 🟠 Important - À planifier sous 2 semaines

```markdown
□ Documents moyens > 1 Mo
□ Tableaux > 1000 éléments régulièrement
□ Duplication de données sans synchronisation
□ Sur-normalisation causant N+1 queries
□ Pas de stratégie d'archivage avec croissance continue
```

### 🟡 Modéré - À améliorer progressivement

```markdown
□ Documents > 100 Ko
□ Imbrication > 5 niveaux
□ Patterns non appliqués pour cas d'usage identifiés
□ Absence de Schema Versioning sur schéma évolutif
□ Collections avec variations importantes de structure
```

---

## Template de Rapport d'Audit

```markdown
# Rapport d'Audit de Modélisation
**Date** : [DATE]
**Collection(s)** : [NOMS]
**Auditeur** : [NOM]

## Résumé Exécutif
- Nombre de collections auditées : X
- Problèmes critiques identifiés : X
- Recommandations prioritaires : X

## Métriques Clés
| Métrique | Valeur | Statut |
|----------|--------|--------|
| Taille moy. documents | X Ko | 🟢/🟡/🔴 |
| Taille max documents | X Ko | 🟢/🟡/🔴 |
| Collections > 1M docs | X | 🟢/🟡/🔴 |

## Problèmes Identifiés
1. [PROBLÈME] - Priorité [🔴/🟠/🟡]
   - Impact : [DESCRIPTION]
   - Action : [RECOMMANDATION]

## Recommandations
1. Court terme (< 1 semaine)
   - [ACTION 1]
   - [ACTION 2]

2. Moyen terme (1-4 semaines)
   - [ACTION 3]
   - [ACTION 4]

3. Long terme (> 1 mois)
   - [ACTION 5]
   - [ACTION 6]

## Annexes
- Scripts utilisés
- Résultats détaillés
```

---

## Ressources Complémentaires

### Documentation Officielle
- [Data Modeling Introduction](https://www.mongodb.com/docs/manual/core/data-modeling-introduction/)
- [Data Model Design](https://www.mongodb.com/docs/manual/core/data-model-design/)
- [Model Relationships](https://www.mongodb.com/docs/manual/tutorial/model-embedded-one-to-many-relationships-between-documents/)

### Guides Avancés
- [Building with Patterns (Blog Series)](https://www.mongodb.com/blog/post/building-with-patterns-a-summary)
- [Schema Design Anti-Patterns](https://www.mongodb.com/developer/products/mongodb/schema-design-anti-pattern-summary/)

### Outils
- **MongoDB Compass** : Analyse visuelle du schéma
- **Atlas Schema Analyzer** : Recommandations automatiques
- **Variety.js** : Analyse de schéma OSS

---


⏭️ [Audit d'indexation](/annexes/checklist-performance/02-audit-indexation.md)
