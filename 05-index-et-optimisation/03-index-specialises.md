🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.3 Index Spécialisés

## Introduction

Après avoir maîtrisé les **index fondamentaux** (simple, composé, multiclé) qui couvrent la majorité des cas d'usage courants, il est temps d'explorer les **index spécialisés** de MongoDB. Ces index sont conçus pour résoudre des problèmes spécifiques que les index classiques ne peuvent pas traiter efficacement.

MongoDB propose cinq types d'index spécialisés, chacun optimisé pour un type de données ou un cas d'usage particulier :

1. **Index Texte (Text)** - Recherche full-text dans du contenu textuel
2. **Index Géospatial (2d, 2dsphere)** - Requêtes sur des coordonnées géographiques
3. **Index Haché (Hashed)** - Distribution uniforme pour le sharding
4. **Index Wildcard** - Indexation de champs dynamiques et flexibles
5. **Index TTL (Time-To-Live)** - Expiration automatique de documents

Ces index spécialisés sont des outils puissants qui, lorsqu'ils sont utilisés correctement, peuvent transformer des opérations impossibles ou très lentes en requêtes ultra-rapides.

---

## Pourquoi des Index Spécialisés ?

### Limites des Index Classiques

Les index fondamentaux (simple, composé, multiclé) sont excellents pour :
- ✅ Recherches d'égalité et de plage
- ✅ Tri sur des champs
- ✅ Filtrage sur des valeurs connues
- ✅ Recherches dans des tableaux

Mais ils ont des limitations pour :
- ❌ Recherche de mots-clés dans du texte long
- ❌ Calculs de distance géographique
- ❌ Distribution uniforme garantie (sharding)
- ❌ Champs avec noms dynamiques/inconnus
- ❌ Suppression automatique de données temporaires

### La Solution : Index Spécialisés

Chaque index spécialisé résout un problème spécifique que les index classiques ne peuvent pas traiter efficacement :

```
Problème                          →  Index Spécialisé
─────────────────────────────────────────────────────────────
Recherche de mots dans texte      →  Index Texte
Coordonnées GPS et distances      →  Index Géospatial
Distribution uniforme (sharding)  →  Index Haché
Champs dynamiques/flexibles       →  Index Wildcard
Expiration automatique données    →  Index TTL
```

---

## Vue d'Ensemble des Index Spécialisés

### 1. Index Texte (Text Index)

**Objectif** : Recherche full-text dans du contenu textuel (blogs, e-commerce, documentation)

**Problème résolu** :
```javascript
// ❌ Recherche naïve - inefficace
db.articles.find({
  content: { $regex: /mongodb/i }
})

// ✅ Avec index texte - rapide et intelligent
db.articles.find({
  $text: { $search: "mongodb database" }
})
```

**Caractéristiques** :
- 🔍 Tokenisation et stemming automatique
- 🌍 Support multilingue (français, anglais, espagnol...)
- ⭐ Score de pertinence
- 🚫 Élimination des stop words ("le", "la", "de"...)

**Cas d'usage** :
- Moteur de recherche de blog/site
- Recherche de produits e-commerce
- Base de connaissances / FAQ
- Documentation technique

**Syntaxe** :
```javascript
db.articles.createIndex({ title: "text", content: "text" })
db.articles.find({ $text: { $search: "mongodb" } })
```

---

### 2. Index Géospatial (2d, 2dsphere)

**Objectif** : Requêtes sur des coordonnées géographiques (GPS, cartes)

**Problème résolu** :
```javascript
// ❌ Sans index géospatial - calcul manuel lent
// Trouver restaurants à < 1km → examiner tous les restaurants

// ✅ Avec index géospatial - recherche optimisée
db.restaurants.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [2.3522, 48.8566] },
      $maxDistance: 1000
    }
  }
})
```

**Caractéristiques** :
- 📍 Calculs de distance sphériques (Terre ronde)
- 🗺️ Support GeoJSON (Point, LineString, Polygon...)
- 🎯 Recherche par proximité, zone, intersection
- 📏 Résultats triés par distance automatiquement

**Cas d'usage** :
- Applications de livraison (restaurants, taxis)
- Immobilier (recherche par zone)
- Tourisme (points d'intérêt)
- Tracking GPS (flottes de véhicules)

**Syntaxe** :
```javascript
db.places.createIndex({ location: "2dsphere" })
db.places.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [lon, lat] },
      $maxDistance: 5000
    }
  }
})
```

---

### 3. Index Haché (Hashed Index)

**Objectif** : Distribution uniforme des données (principalement pour sharding)

**Problème résolu** :
```javascript
// ❌ Range sharding - déséquilibre
// Shard 1 : IDs 1-10000 (peu actif)
// Shard 2 : IDs 10001-20000 (très actif) → HOTSPOT !

// ✅ Hashed sharding - équilibré
// Shard 1 : ~50% des IDs (mélangés)
// Shard 2 : ~50% des IDs (mélangés)
```

**Caractéristiques** :
- 🔀 Fonction de hachage pour mélanger les valeurs
- ⚖️ Distribution uniforme garantie mathématiquement
- 🚀 Parfait pour éviter les hotspots
- ⚠️ Perd l'ordre des valeurs (pas de tri ni plage)

**Cas d'usage** :
- Sharding de collections à forte croissance
- IDs séquentiels ou temporels
- Distribution uniforme de charge
- Cache distribué

**Syntaxe** :
```javascript
db.users.createIndex({ userId: "hashed" })
sh.shardCollection("mydb.users", { userId: "hashed" })
```

---

### 4. Index Wildcard

**Objectif** : Indexer des champs avec noms dynamiques ou structure flexible

**Problème résolu** :
```javascript
// Collection avec attributs variables
// Laptop : { brand, processor, ram, storage }
// T-shirt : { brand, size, color, material }
// Livre : { author, isbn, pages, publisher }

// ❌ Créer 50+ index différents ? Impossible à maintenir !

// ✅ Un seul index wildcard - s'adapte automatiquement
db.products.createIndex({ "attributes.$**": 1 })

// Fonctionne pour TOUS les attributs
db.products.find({ "attributes.processor": "Intel i7" })
db.products.find({ "attributes.color": "Blue" })
db.products.find({ "attributes.author": "John Doe" })
```

**Caractéristiques** :
- 🌟 Indexe automatiquement tous les sous-champs
- 🔄 S'adapte aux nouveaux champs sans reconfiguration
- 📊 Parfait pour schémas flexibles et évolutifs
- ⚠️ Plus volumineux qu'un index classique

**Cas d'usage** :
- Attributs produits variables (e-commerce)
- Métadonnées extensibles
- Configuration multi-tenant
- Documents hétérogènes dans une collection

**Syntaxe** :
```javascript
db.products.createIndex({ "attributes.$**": 1 })
db.users.createIndex({ "customFields.$**": 1 })
```

---

### 5. Index TTL (Time-To-Live)

**Objectif** : Suppression automatique de documents après un délai

**Problème résolu** :
```javascript
// ❌ Script cron manuel - complexe et peu fiable
// Exécuter toutes les heures :
// db.sessions.deleteMany({ createdAt: { $lt: thirtyMinutesAgo } })

// ✅ Index TTL - automatique et garanti
db.sessions.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 1800 }  // 30 minutes
)
// MongoDB supprime automatiquement les vieux documents !
```

**Caractéristiques** :
- ⏰ Suppression automatique en arrière-plan
- 🔄 Thread MongoDB vérifie toutes les 60 secondes
- 📅 Basé sur un champ de type Date
- 🎯 Idéal pour données temporaires

**Cas d'usage** :
- Sessions utilisateur (expiration après inactivité)
- Cache temporaire
- Logs (rétention limitée)
- Tokens de vérification
- Conformité RGPD (suppression automatique)

**Syntaxe** :
```javascript
db.sessions.createIndex(
  { lastActivityAt: 1 },
  { expireAfterSeconds: 1800 }  // 30 minutes
)
```

---

## Tableau Comparatif des Index Spécialisés

| Index | Problème Résolu | Cas d'Usage Principal | Limitation Principale |
|-------|-----------------|----------------------|----------------------|
| **Texte** | Recherche full-text | Blogs, e-commerce, docs | 1 seul par collection |
| **Géospatial** | Coordonnées GPS | Livraison, tourisme, maps | Format GeoJSON requis |
| **Haché** | Distribution uniforme | Sharding, hotspots | Pas de plage ni tri |
| **Wildcard** | Champs dynamiques | Schémas flexibles | Volumineux |
| **TTL** | Expiration auto | Sessions, cache, logs | Délai jusqu'à 60s |

---

## Comment Choisir le Bon Index Spécialisé ?

### Arbre de Décision

```
Votre besoin est...

├─ Rechercher des MOTS-CLÉS dans du TEXTE
│  └─→ Index Texte
│     "Recherche d'articles contenant 'MongoDB'"
│
├─ Travailler avec des COORDONNÉES GPS
│  └─→ Index Géospatial
│     "Restaurants à moins de 2 km"
│
├─ DISTRIBUER uniformément les données (sharding)
│  └─→ Index Haché
│     "Éviter les hotspots avec IDs séquentiels"
│
├─ Indexer des CHAMPS DYNAMIQUES/VARIABLES
│  └─→ Index Wildcard
│     "Attributs produits différents par catégorie"
│
└─ SUPPRIMER automatiquement après un DÉLAI
   └─→ Index TTL
      "Sessions expirées après 30 minutes"
```

### Questions à Se Poser

#### 1. Quel type de données avez-vous ?

- **Texte long** (articles, descriptions) → Index Texte
- **Coordonnées GPS** (latitude/longitude) → Index Géospatial
- **IDs séquentiels** → Index Haché
- **Champs flexibles** (noms inconnus) → Index Wildcard
- **Données temporaires** (avec date) → Index TTL

#### 2. Quel type d'opération effectuez-vous ?

- **Recherche de mots** → Index Texte
- **Calcul de distance** → Index Géospatial
- **Distribution de données** → Index Haché
- **Requête sur champs variables** → Index Wildcard
- **Nettoyage automatique** → Index TTL

#### 3. Quelle est votre contrainte principale ?

- **Pertinence des résultats** → Index Texte (scoring)
- **Précision géographique** → Index Géospatial
- **Équilibrage de charge** → Index Haché
- **Évolution du schéma** → Index Wildcard
- **Croissance de la collection** → Index TTL

---

## Combinaison d'Index Spécialisés

### Index Spécialisés + Index Fondamentaux

Vous pouvez (et devez souvent) combiner index spécialisés et fondamentaux :

```javascript
// Collection de produits
{
  _id: 1,
  name: "Laptop Dell XPS",
  category: "Electronics",
  price: 1299,
  description: "Powerful laptop with Intel i7...",
  location: { type: "Point", coordinates: [2.3522, 48.8566] },
  attributes: { brand: "Dell", ram: "16GB" },
  createdAt: ISODate("2024-01-15")
}

// Index fondamentaux
db.products.createIndex({ category: 1, price: -1 })  // Composé
db.products.createIndex({ name: 1 })                 // Simple

// Index spécialisés
db.products.createIndex({ description: "text" })           // Texte
db.products.createIndex({ location: "2dsphere" })         // Géospatial
db.products.createIndex({ "attributes.$**": 1 })          // Wildcard
db.products.createIndex({ createdAt: 1 }, { expireAfterSeconds: 7776000 })  // TTL
```

### Plusieurs Index Spécialisés

Certaines collections peuvent bénéficier de plusieurs index spécialisés :

```javascript
// Site e-commerce
db.products.createIndex({ name: "text", description: "text" })  // Recherche
db.products.createIndex({ storeLocation: "2dsphere" })          // Géoloc
db.products.createIndex({ "dynamicAttrs.$**": 1 })              // Attributs

// Système avec sessions et cache
db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 1800 })  // TTL
db.cache.createIndex({ key: "hashed" })                                   // Haché
db.cache.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })    // TTL
```

---

## Stratégie d'Adoption des Index Spécialisés

### Étape 1 : Identifier les Besoins

Analysez vos cas d'usage :

```javascript
// Quelles requêtes sont lentes ou impossibles ?
db.collection.find(...).explain("executionStats")

// Quels types de données avez-vous ?
// - Texte long ? → Index Texte
// - GPS ? → Index Géospatial
// - Temporaire ? → Index TTL
```

### Étape 2 : Choisir le(s) Index Approprié(s)

Utilisez l'arbre de décision ci-dessus pour identifier le ou les index spécialisés nécessaires.

### Étape 3 : Tester en Environnement de Développement

```javascript
// 1. Créer l'index
db.collection.createIndex(...)

// 2. Tester les requêtes
db.collection.find(...).explain("executionStats")

// 3. Vérifier les performances
// - Temps de requête
// - Documents examinés
// - Utilisation de l'index
```

### Étape 4 : Monitorer en Production

```javascript
// Surveiller l'utilisation
db.collection.aggregate([{ $indexStats: {} }])

// Surveiller la taille
db.collection.stats().indexSizes

// Surveiller les performances
db.currentOp()
```

### Étape 5 : Optimiser et Ajuster

- Ajuster les paramètres (TTL duration, weights texte, etc.)
- Supprimer les index inutilisés
- Affiner selon les patterns réels d'utilisation

---

## Limitations Communes

### Restrictions Générales

Certaines limitations s'appliquent à plusieurs index spécialisés :

#### Un Seul Index de Certains Types

- **Index Texte** : 1 seul par collection maximum
- **Index Wildcard** : Pas de limitation de nombre, mais attention à la taille

#### Incompatibilités

```javascript
// ❌ Impossible : Index composé avec certains types
db.collection.createIndex({
  category: 1,
  description: "text"  // Index texte doit être seul ou avec préfixe
})

// ❌ Impossible : Index TTL composé
db.collection.createIndex(
  { createdAt: 1, userId: 1 },
  { expireAfterSeconds: 3600 }
)
```

#### Types de Données

- **Index Géospatial** : Nécessite format GeoJSON strict
- **Index TTL** : Nécessite champ de type Date
- **Index Haché** : Perd l'ordre (pas de range queries)

---

## Performance et Coûts

### Taille des Index Spécialisés

Les index spécialisés peuvent être plus volumineux que les index classiques :

```
Type d'Index          Taille Relative    Notes
─────────────────────────────────────────────────────────
Index Simple          ⭐ (référence)     ~5-15% des données
Index Texte           ⭐⭐⭐             ~20-50% (tokenisation)
Index Géospatial      ⭐⭐               ~10-25%
Index Haché           ⭐                 ~5-15% (similaire simple)
Index Wildcard        ⭐⭐⭐⭐           ~30-100% (tous les chemins)
Index TTL             ⭐                 ~5-15% (similaire simple)
```

### Impact sur les Écritures

Les index spécialisés ont des impacts variés sur les performances d'écriture :

```
Type d'Index          Impact Écriture    Raison
─────────────────────────────────────────────────────────
Index Simple          ⭐                 Mise à jour simple
Index Texte           ⭐⭐⭐             Tokenisation + stemming
Index Géospatial      ⭐⭐               Calculs géométriques
Index Haché           ⭐                 Calcul de hash rapide
Index Wildcard        ⭐⭐⭐             Analyse structure + multiple entrées
Index TTL             ⭐                 Identique index simple
```

### Recommandations

✅ **Utilisez les index spécialisés quand ils sont nécessaires**
- Ne pas hésiter si le cas d'usage correspond
- Les bénéfices dépassent largement les coûts

⚠️ **Mais n'en abusez pas**
- Pas d'index "au cas où"
- Chaque index a un coût
- Privilégiez les index réellement utilisés

---

## Exemples de Scénarios Réels

### Scénario 1 : Site E-commerce

**Besoins** :
- Recherche de produits par mots-clés
- Recherche de magasins proches
- Attributs variables par catégorie

**Solution** :
```javascript
// Index texte pour recherche produits
db.products.createIndex({
  name: "text",
  description: "text"
}, {
  weights: { name: 10, description: 1 }
})

// Index géospatial pour magasins
db.stores.createIndex({ location: "2dsphere" })

// Index wildcard pour attributs flexibles
db.products.createIndex({ "specs.$**": 1 })
```

### Scénario 2 : Application de Livraison

**Besoins** :
- Trouver restaurants proches
- Chercher restaurants par nom/spécialité
- Sessions utilisateur temporaires

**Solution** :
```javascript
// Index géospatial pour proximité
db.restaurants.createIndex({ location: "2dsphere" })

// Index texte pour recherche
db.restaurants.createIndex({
  name: "text",
  cuisine: "text"
})

// Index TTL pour sessions
db.sessions.createIndex(
  { lastActivityAt: 1 },
  { expireAfterSeconds: 1800 }
)
```

### Scénario 3 : Plateforme IoT

**Besoins** :
- Données de capteurs avec attributs variables
- Localisation géographique
- Rétention limitée (30 jours)
- Distribution uniforme (sharding)

**Solution** :
```javascript
// Index wildcard pour mesures variables
db.sensor_data.createIndex({ "readings.$**": 1 })

// Index géospatial pour localisation
db.sensors.createIndex({ location: "2dsphere" })

// Index TTL pour rétention
db.sensor_data.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 2592000 }  // 30 jours
)

// Index haché pour sharding
db.sensor_data.createIndex({ sensorId: "hashed" })
sh.shardCollection("iot.sensor_data", { sensorId: "hashed" })
```

---

## Outils et Commandes Utiles

### Lister Tous les Index

```javascript
db.collection.getIndexes()
```

### Identifier les Index Spécialisés

```javascript
// Index texte
db.collection.getIndexes().filter(idx => idx.key._fts === "text")

// Index géospatial
db.collection.getIndexes().filter(idx =>
  Object.values(idx.key).includes("2dsphere") ||
  Object.values(idx.key).includes("2d")
)

// Index haché
db.collection.getIndexes().filter(idx =>
  Object.values(idx.key).includes("hashed")
)

// Index TTL
db.collection.getIndexes().filter(idx => idx.expireAfterSeconds !== undefined)
```

### Statistiques d'Utilisation

```javascript
// Par index
db.collection.aggregate([{ $indexStats: {} }])

// Tailles des index
db.collection.stats().indexSizes
```

---

## Bonnes Pratiques Générales

### ✅ À Faire

1. **Identifier le besoin avant de créer**
   - Analyser les requêtes problématiques
   - Choisir l'index adapté au cas d'usage

2. **Tester en environnement de dev**
   - Vérifier les performances avec explain()
   - Mesurer l'impact sur les écritures

3. **Combiner intelligemment**
   - Index spécialisés + index fondamentaux
   - Couvrir tous les patterns de requêtes

4. **Documenter les choix**
   - Expliquer pourquoi chaque index existe
   - Noter les requêtes qu'il optimise

5. **Monitorer l'utilisation**
   - Vérifier régulièrement avec $indexStats
   - Supprimer les index inutilisés

### ❌ À Éviter

1. **Ne pas créer "au cas où"**
   - Chaque index a un coût
   - Créer uniquement si besoin prouvé

2. **Ne pas ignorer les limitations**
   - 1 seul index texte par collection
   - Index TTL nécessite type Date
   - Index haché ne supporte pas les plages

3. **Ne pas oublier les index fondamentaux**
   - Index spécialisés ne remplacent pas les classiques
   - Combiner pour couvrir tous les besoins

4. **Ne pas négliger la taille**
   - Certains index sont très volumineux (wildcard, texte)
   - Surveiller l'espace disque

5. **Ne pas sous-estimer l'impact sur les écritures**
   - Index texte et wildcard ralentissent les insertions
   - Mesurer et accepter le compromis

---

## Progression dans l'Apprentissage

Cette section introduit les cinq types d'index spécialisés. Pour maîtriser chacun d'eux en profondeur, consultez les sections détaillées suivantes :

### 📖 Sections Détaillées

1. **[5.3.1 Index Texte (Text)](./03.1-index-texte.md)**
   - Recherche full-text avancée
   - Tokenisation, stemming, stop words
   - Score de pertinence
   - Support multilingue
   - Poids des champs

2. **[5.3.2 Index Géospatial (2d, 2dsphere)](./03.2-index-geospatial.md)**
   - Format GeoJSON
   - Opérateurs géospatiaux ($near, $geoWithin, $geoIntersects)
   - Calculs de distance
   - Types de géométries
   - 2d vs 2dsphere

3. **[5.3.3 Index Haché (Hashed)](./03.3-index-hache.md)**
   - Fonction de hachage
   - Hashed sharding
   - Distribution uniforme
   - Limitations (pas de plage)
   - Cas d'usage sharding

4. **[5.3.4 Index Wildcard](./03.4-index-wildcard.md)**
   - Champs dynamiques
   - WildcardProjection
   - Schémas flexibles
   - Performance et taille
   - Alternatives

5. **[5.3.5 Index TTL (Time-To-Live)](./03.5-index-ttl.md)**
   - Expiration automatique
   - expireAfterSeconds
   - Thread TTL en arrière-plan
   - Modification avec collMod
   - Cas d'usage temporels

### 🎯 Parcours Recommandé

**Pour les débutants** :
1. **Index TTL** (5.3.5) - Le plus simple à comprendre
2. **Index Texte** (5.3.1) - Cas d'usage courant
3. **Index Géospatial** (5.3.2) - Si besoin de géolocalisation
4. **Index Wildcard** (5.3.4) - Pour schémas flexibles
5. **Index Haché** (5.3.3) - Pour sharding avancé

**Pour les utilisateurs intermédiaires** :
1. Identifier vos besoins spécifiques
2. Approfondir les index correspondants
3. Combiner avec les index fondamentaux
4. Tester en environnement de staging

**Pour les experts** :
- Utilisez cette section comme référence rapide
- Consultez les sections détaillées pour cas avancés
- Optimisez les combinaisons d'index
- Passez aux sections suivantes sur l'optimisation

---

## Conclusion

Les **index spécialisés** étendent considérablement les capacités de MongoDB en permettant de traiter efficacement des cas d'usage que les index fondamentaux ne peuvent pas gérer. Chaque type d'index spécialisé résout un problème spécifique :

- 🔍 **Texte** → Recherche full-text intelligente
- 📍 **Géospatial** → Requêtes géographiques précises
- 🔀 **Haché** → Distribution uniforme garantie
- 🌟 **Wildcard** → Flexibilité maximale pour schémas dynamiques
- ⏰ **TTL** → Gestion automatique du cycle de vie

### Points Clés à Retenir

- 🔑 Chaque index spécialisé résout un problème spécifique
- 🔑 Ne remplacent pas les index fondamentaux, les complètent
- 🔑 Peuvent être combinés entre eux et avec index classiques
- 🔑 Ont des coûts (taille, écritures) mais bénéfices valent largement
- 🔑 Nécessitent de choisir le bon outil pour le bon problème
- 🔑 Chacun a ses propres limitations à connaître
- 🔑 Testez et mesurez avant déploiement en production

### Prochaines Étapes

Après avoir exploré les index spécialisés, vous serez prêt pour :
- **[5.4 Options et Modificateurs d'Index](./04-options-modificateurs-index.md)** : Unique, partiel, sparse, caché
- **[5.5 Création et Suppression d'Index](./05-creation-suppression-index.md)** : Gestion en production
- **[5.6 Analyse avec explain()](./06-analyse-explain.md)** : Diagnostic approfondi des requêtes
- **[5.8 Stratégies d'Optimisation](./08-strategies-optimisation.md)** : Techniques avancées

---

**📚 Ressources Complémentaires**
- [Documentation officielle MongoDB - Indexes](https://docs.mongodb.com/manual/indexes/)
- [Index Types Overview](https://docs.mongodb.com/manual/indexes/#index-types)
- [Indexing Strategies](https://docs.mongodb.com/manual/applications/indexes/)
- [Performance Best Practices](https://www.mongodb.com/basics/best-practices)

⏭️ [Index texte (Text)](/05-index-et-optimisation/03.1-index-texte.md)
