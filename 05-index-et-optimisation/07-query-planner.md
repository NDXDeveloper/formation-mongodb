🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.7 Le Query Planner

## Introduction

Le **Query Planner** (planificateur de requêtes) est le "cerveau" de MongoDB qui décide **comment** exécuter vos requêtes. C'est lui qui choisit quel index utiliser, dans quel ordre examiner les documents, et quelle stratégie employer pour obtenir les résultats le plus rapidement possible.

Comprendre le Query Planner vous permet de :
- 🧠 Savoir **pourquoi** un index est choisi plutôt qu'un autre
- 🎯 **Prédire** quel index sera utilisé
- ⚡ **Optimiser** la structure de vos index
- 🔧 **Diagnostiquer** les choix inattendus
- 📈 **Améliorer** les performances globales

Le Query Planner travaille en coulisses à chaque requête, et bien le comprendre est essentiel pour maîtriser l'optimisation MongoDB.

---

## Qu'est-ce que le Query Planner ?

### Définition

Le **Query Planner** est le composant de MongoDB responsable de :
1. Analyser une requête
2. Identifier les index disponibles
3. Évaluer plusieurs stratégies (plans) possibles
4. Choisir le meilleur plan d'exécution
5. Mettre en cache ce plan pour les requêtes similaires

### Analogie

```
Le Query Planner est comme un GPS intelligent :

Vous : "Je veux aller de Paris à Lyon"

GPS (Query Planner) :
├─ Analyse : Départ Paris, Arrivée Lyon
├─ Options disponibles :
│  • Autoroute A6 (rapide mais péages)
│  • Routes nationales (gratuit mais lent)
│  • Train (très rapide mais coûteux)
├─ Évaluation : Teste mentalement chaque option
├─ Choix : Sélectionne l'autoroute A6
└─ Mémorisation : Se souvient pour les prochains trajets similaires

MongoDB fait exactement la même chose avec vos requêtes !
```

### Position dans l'architecture MongoDB

```
Architecture d'exécution de requête
═══════════════════════════════════

1. Application envoie la requête
         ↓
2. MongoDB reçoit la requête
         ↓
3. ┌─────────────────────┐
   │   QUERY PLANNER     │ ← Nous sommes ici !
   │  (Planificateur)    │
   └──────────┬──────────┘
              ↓
   • Analyse la requête
   • Identifie les index candidats
   • Teste plusieurs plans
   • Choisit le meilleur
              ↓
4. ┌─────────────────────┐
   │  QUERY EXECUTOR     │
   │  (Exécuteur)        │
   └──────────┬──────────┘
              ↓
5. Résultats retournés à l'application
```

---

## Comment le Query Planner fonctionne

### Processus en 5 étapes

```
Processus de sélection de plan
═══════════════════════════════

Étape 1 : ANALYSE DE LA REQUÊTE
┌────────────────────────────────────┐
│ db.users.find({                    │
│   city: "Paris",                   │
│   age: { $gte: 25 }                │
│ })                                 │
└────────────────────────────────────┘
         ↓
Le Query Planner identifie :
• Filtres : city="Paris", age >= 25
• Tri : Aucun
• Projection : Tous les champs
• Limite : Aucune


Étape 2 : IDENTIFICATION DES INDEX CANDIDATS
┌────────────────────────────────────┐
│ Index disponibles :                │
│ 1. _id                             │
│ 2. city_1                          │
│ 3. age_1                           │
│ 4. city_1_age_1                    │
└────────────────────────────────────┘
         ↓
Index candidats (peuvent répondre à la requête) :
• city_1 (peut filtrer sur city)
• age_1 (peut filtrer sur age)
• city_1_age_1 (peut filtrer sur les deux)


Étape 3 : GÉNÉRATION DE PLANS D'EXÉCUTION
┌────────────────────────────────────┐
│ Plan A : Utiliser city_1           │
│ Plan B : Utiliser age_1            │
│ Plan C : Utiliser city_1_age_1     │
│ Plan D : COLLSCAN (pas d'index)    │
└────────────────────────────────────┘


Étape 4 : COMPÉTITION DES PLANS
┌────────────────────────────────────┐
│ Exécution partielle en parallèle : │
│                                    │
│ Plan A : 100 docs en 15ms          │
│ Plan B : 50 docs en 20ms           │
│ Plan C : 100 docs en 8ms ✅        │
│ Plan D : 30 docs en 45ms           │
└────────────────────────────────────┘
         ↓
Le Plan C est le plus rapide !


Étape 5 : SÉLECTION ET CACHE
┌────────────────────────────────────┐
│ Plan gagnant : city_1_age_1        │
│ Mise en cache du plan              │
│ Réutilisation pour requêtes        │
│ similaires                         │
└────────────────────────────────────┘
```

### Détails de la compétition

Le Query Planner ne **devine** pas quel plan est le meilleur, il les **teste réellement** :

```
Compétition de plans (trial period)
════════════════════════════════════

Méthode :
1. Tous les plans candidats démarrent en parallèle
2. Chacun commence à exécuter la requête
3. Le premier à retourner N documents (ou tout examiner) gagne
4. Les autres plans sont abandonnés
5. Le plan gagnant est utilisé et mis en cache

Durée typique : Quelques millisecondes
Fréquence : Première exécution d'une nouvelle forme de requête
```

---

## Facteurs de décision du Query Planner

### 1. Sélectivité des index

La **sélectivité** est le pourcentage de documents qui correspondent au filtre.

```
Sélectivité = Documents correspondants / Total documents

Haute sélectivité (BON) :
═════════════════════════
Index sur email : 1 document sur 1M
Sélectivité : 1 / 1,000,000 = 0.0001% ✅
→ Index très efficace

Faible sélectivité (MOINS BON) :
══════════════════════════════════
Index sur gender : 500K documents sur 1M
Sélectivité : 500,000 / 1,000,000 = 50% ⚠️
→ Index moins efficace (COLLSCAN peut être choisi)
```

#### Exemple concret

```javascript
// Collection de 1 million d'utilisateurs
// 500,000 hommes, 500,000 femmes

// Index créés
db.users.createIndex({ gender: 1 })
db.users.createIndex({ email: 1 })

// Requête 1 : Haute sélectivité
db.users.find({ email: "alice@example.com" })
  .explain("executionStats")
// → Utilise l'index email_1 ✅
// Raison : Seulement 1 document correspond

// Requête 2 : Faible sélectivité
db.users.find({ gender: "male" })
  .explain("executionStats")
// → Peut faire un COLLSCAN ! ⚠️
// Raison : 50% des documents correspondent
// Le Query Planner estime qu'un COLLSCAN est plus rapide
// que parcourir 500,000 entrées d'index
```

### 2. Couverture des filtres

Le Query Planner préfère les index qui couvrent **tous** les filtres de la requête :

```javascript
// Requête avec deux filtres
db.orders.find({
  userId: 12345,
  status: "pending"
})

// Index disponibles :
// A) userId_1 (couvre 1 filtre)
// B) status_1 (couvre 1 filtre)
// C) userId_1_status_1 (couvre 2 filtres) ✅

// Le Query Planner choisira probablement C
// car il couvre TOUS les filtres
```

### 3. Support du tri

Si la requête inclut un tri, le Query Planner favorise les index qui peuvent **éviter le tri en mémoire** :

```javascript
// Requête avec tri
db.posts.find({ status: "published" })
  .sort({ publishedAt: -1 })

// Index disponibles :
// A) status_1 (filtre OK, mais tri en mémoire)
// B) status_1_publishedAt_-1 (filtre + tri via index) ✅

// Le Query Planner choisira B
// car il évite un SORT coûteux en mémoire
```

### 4. Ordre des champs dans l'index composé

Pour un index composé, le Query Planner peut l'utiliser si la requête utilise un **préfixe** de l'index :

```javascript
// Index créé
db.users.createIndex({ country: 1, city: 1, age: 1 })

// Requête 1 : Utilise l'index (préfixe complet)
db.users.find({ country: "FR", city: "Paris", age: 30 })
// ✅ Utilise l'index complètement

// Requête 2 : Utilise l'index (préfixe partiel)
db.users.find({ country: "FR", city: "Paris" })
// ✅ Utilise l'index (prefix: country, city)

// Requête 3 : Utilise l'index (premier champ)
db.users.find({ country: "FR" })
// ✅ Utilise l'index (prefix: country)

// Requête 4 : N'utilise PAS l'index efficacement
db.users.find({ city: "Paris" })
// ❌ Ne peut pas utiliser l'index
// (ne commence pas par le premier champ)
```

### 5. Cardinalité des champs

La **cardinalité** est le nombre de valeurs distinctes dans un champ :

```
Haute cardinalité (BON pour index) :
═════════════════════════════════════
email : 1,000,000 valeurs uniques
→ Index très efficace ✅

Cardinalité moyenne :
═════════════════════
city : 1,000 valeurs distinctes
→ Index efficace ✅

Faible cardinalité (MOINS BON pour index) :
═══════════════════════════════════════════
gender : 2 valeurs (male, female)
→ Index peu efficace ⚠️
Le Query Planner peut préférer un COLLSCAN
```

### 6. Taille des données

Le Query Planner prend en compte si l'index tient en mémoire RAM :

```javascript
// Si l'index est en RAM : RAPIDE ✅
// Si l'index est sur disque : PLUS LENT ⚠️

// Le Query Planner peut préférer un COLLSCAN
// si les données sont déjà en cache mémoire
// plutôt qu'un index qui nécessite des lectures disque
```

---

## Le système de cache de plans

### Concept

Le Query Planner ne refait pas la compétition à **chaque** requête. Il met en cache les plans gagnants pour les réutiliser.

### Fonctionnement du cache

```
Cycle de vie d'un plan de requête
═════════════════════════════════

1. Première exécution
   ├─ Forme de requête : { city: ?, age: ? }
   ├─ Compétition entre plans
   ├─ Plan gagnant : city_1_age_1
   └─ Mise en CACHE

2. Exécutions suivantes (même forme)
   ├─ Forme de requête : { city: ?, age: ? }
   ├─ Plan trouvé dans le cache ✅
   └─ Utilisation directe (pas de compétition)

3. Invalidation du cache
   ├─ Après ~1000 écritures
   ├─ Après modification des index
   ├─ Après redémarrage du serveur
   └─ Avec db.collection.getPlanCache().clear()

4. Réinitialisation
   └─ Retour à l'étape 1 (nouvelle compétition)
```

### Forme de requête (Query Shape)

Le cache utilise la **forme** de la requête, pas les valeurs exactes :

```javascript
// Ces deux requêtes ont la MÊME forme
db.users.find({ city: "Paris", age: 30 })
db.users.find({ city: "Lyon", age: 25 })
// Forme : { city: <value>, age: <value> }
// → Utilisent le même plan en cache

// Ces requêtes ont des formes DIFFÉRENTES
db.users.find({ city: "Paris" })
// Forme : { city: <value> }

db.users.find({ city: "Paris", age: 30 })
// Forme : { city: <value>, age: <value> }
// → Plans en cache différents
```

### Visualisation du cache

```javascript
// Voir les plans en cache
db.users.getPlanCache().list()
```

**Exemple de sortie** :

```json
[
  {
    "queryHash": "ABC123",
    "planCacheKey": "DEF456",
    "isActive": true,
    "works": 125,
    "cachedPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "city_1_age_1"
      }
    },
    "creationExecStats": [
      // Statistiques de la compétition initiale
    ]
  }
]
```

### Effacer le cache de plans

```javascript
// Effacer tout le cache d'une collection
db.users.getPlanCache().clear()

// Effacer un plan spécifique
db.users.getPlanCache().clearPlansByQuery({
  city: "Paris",
  age: { $gte: 25 }
})
```

**Quand effacer le cache ?** :
```
Situations où effacer le cache :
════════════════════════════════

✅ Après création d'un nouvel index
   → Forcer la réévaluation des plans

✅ Après modification de la distribution des données
   → Les anciennes estimations peuvent être obsolètes

✅ Pour tester l'impact d'optimisations
   → Comparer avec un cache vierge

❌ En production sans raison
   → Cause une dégradation temporaire des performances
```

---

## Comprendre les choix du Query Planner

### Exemple 1 : Choix entre index simple et composé

```javascript
// Index disponibles
db.orders.getIndexes()
// [
//   { key: { userId: 1 }, name: "userId_1" },
//   { key: { userId: 1, createdAt: -1 }, name: "userId_1_createdAt_-1" }
// ]

// Requête
db.orders.find({ userId: 12345 })
  .sort({ createdAt: -1 })
  .explain("executionStats")
```

**Quel index sera choisi ?**

```
Analyse du Query Planner :
═══════════════════════════

Plan A : userId_1
├─ Filtre : ✅ OK (utilise l'index)
├─ Tri : ❌ En mémoire (SORT stage)
└─ Score : Moyen

Plan B : userId_1_createdAt_-1
├─ Filtre : ✅ OK (utilise l'index)
├─ Tri : ✅ Via l'index (pas de SORT)
└─ Score : Excellent ✅

Choix : Plan B (userId_1_createdAt_-1)
Raison : Évite le tri en mémoire
```

### Exemple 2 : COLLSCAN malgré un index disponible

```javascript
// Index disponible
db.users.createIndex({ isActive: 1 })

// Collection : 1 million d'utilisateurs
// 950,000 sont actifs (95%)

// Requête
db.users.find({ isActive: true })
  .explain("executionStats")
```

**Résultat surprenant** :

```json
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "COLLSCAN"           // ⚠️ COLLSCAN !?
    }
  }
}
```

**Explication** :

```
Pourquoi un COLLSCAN ?
══════════════════════

Analyse du Query Planner :
├─ 95% des documents correspondent au filtre
├─ Option A : Utiliser l'index isActive_1
│  └─ Parcourir 950,000 entrées d'index
│  └─ Récupérer 950,000 documents
│  └─ Temps estimé : 5 secondes
│
├─ Option B : COLLSCAN
│  └─ Parcourir 1,000,000 documents directement
│  └─ Temps estimé : 3 secondes
│
└─ Choix : COLLSCAN ✅
   Raison : Plus rapide que l'index
   quand la sélectivité est très faible
```

### Exemple 3 : Préférence pour un index moins évident

```javascript
// Index disponibles
db.products.createIndex({ category: 1 })
db.products.createIndex({ price: 1 })
db.products.createIndex({ inStock: 1 })

// Requête
db.products.find({
  category: "Electronics",    // 100,000 docs
  price: { $lt: 50 },        // 500,000 docs
  inStock: true              // 50,000 docs
})
```

**Quel index sera choisi ?**

```
Analyse de sélectivité :
════════════════════════

Index category_1 : 100,000 docs → 10% de sélectivité
Index price_1 : 500,000 docs → 50% de sélectivité
Index inStock_1 : 50,000 docs → 5% de sélectivité ✅

Choix probable : inStock_1
Raison : La plus haute sélectivité
Filtre le plus efficacement dès le départ
```

---

## Hints : Forcer l'utilisation d'un index

Parfois, vous savez mieux que le Query Planner quel index utiliser. Vous pouvez utiliser `hint()` pour **forcer** un index spécifique.

### Syntaxe

```javascript
// Forcer par nom d'index
db.collection.find({ ... }).hint("indexName")

// Forcer par définition
db.collection.find({ ... }).hint({ field: 1 })

// Forcer un COLLSCAN
db.collection.find({ ... }).hint({ $natural: 1 })
```

### Exemples

#### Forcer un index spécifique

```javascript
// Le Query Planner choisit city_1
db.users.find({ city: "Paris", age: 30 })

// Forcer l'utilisation de age_1 à la place
db.users.find({ city: "Paris", age: 30 })
  .hint("age_1")

// Comparer les performances
const withoutHint = db.users.find({ city: "Paris", age: 30 })
  .explain("executionStats")

const withHint = db.users.find({ city: "Paris", age: 30 })
  .hint("city_1_age_1")
  .explain("executionStats")

print(`Sans hint : ${withoutHint.executionStats.executionTimeMillis}ms`)
print(`Avec hint : ${withHint.executionStats.executionTimeMillis}ms`)
```

#### Forcer un COLLSCAN (pour tests)

```javascript
// Forcer un scan complet (désactiver les index)
db.users.find({ city: "Paris" })
  .hint({ $natural: 1 })
  .explain("executionStats")

// Utile pour :
// - Comparer les performances avec/sans index
// - Tester si l'index fait vraiment une différence
```

### Quand utiliser hint() ?

```
✅ Utiliser hint() quand :
══════════════════════════

• Vous avez PROUVÉ avec explain() que le Query Planner
  fait un mauvais choix
• Les statistiques de collection sont obsolètes
• Cas d'usage spécifique où vous connaissez la meilleure stratégie
• Pour tests de performance (comparer différents index)

❌ NE PAS utiliser hint() :
═══════════════════════════

• Par défaut ou "au cas où"
• Sans avoir mesuré avec explain()
• En production sans tests approfondis
• Vous risquez de forcer un plan sous-optimal

⚠️  ATTENTION :
════════════════
• Si l'index hinted n'existe pas : ERREUR
• Le hint override le Query Planner même si mauvais choix
• Maintenir les hints peut devenir complexe
```

### Erreur avec hint inexistant

```javascript
// Index inexistant
db.users.find({ city: "Paris" })
  .hint("nonexistent_index")

// Erreur :
// Error: error: {
//   "ok": 0,
//   "errmsg": "error processing query: ... planner returned error: bad hint",
//   "code": 2
// }
```

---

## Influencer le Query Planner sans hint()

### 1. Créer l'index optimal

La meilleure façon d'influencer le Query Planner est de créer l'index parfait :

```javascript
// Au lieu de forcer avec hint()
db.orders.find({ userId: 123, status: "pending" })
  .sort({ createdAt: -1 })
  .hint("userId_1")

// Créer l'index optimal (règle ESR)
db.orders.createIndex({
  userId: 1,        // E - Equality
  status: 1,        // E - Equality
  createdAt: -1     // S - Sort
})

// Maintenant le Query Planner choisira naturellement
// le meilleur index
```

### 2. Supprimer les index redondants

Des index redondants peuvent confondre le Query Planner :

```javascript
// Index redondants
db.users.createIndex({ city: 1 })
db.users.createIndex({ city: 1, age: 1 })

// Problème : Le Query Planner doit choisir entre les deux
// Solution : Supprimer city_1, garder city_1_age_1
db.users.dropIndex("city_1")

// city_1_age_1 peut servir pour :
// • { city: "Paris" }
// • { city: "Paris", age: 30 }
```

### 3. Mettre à jour les statistiques de collection

MongoDB collecte des statistiques sur vos données. Si elles sont obsolètes, le Query Planner peut faire de mauvais choix.

```javascript
// Forcer la recollecte des statistiques
db.runCommand({
  validate: "users",
  full: false  // false = rapide, true = complet mais lent
})

// Reconstruire les index (recalcule les statistiques)
db.users.reIndex()
```

### 4. Utiliser des index partiels

Les index partiels peuvent améliorer les choix du Query Planner :

```javascript
// Au lieu d'un index complet
db.orders.createIndex({ status: 1 })
// → Inclut tous les statuts (pending, completed, cancelled)

// Créer un index partiel
db.orders.createIndex(
  { status: 1, createdAt: -1 },
  {
    partialFilterExpression: {
      status: { $in: ["pending", "processing"] }
    }
  }
)
// → Seulement les commandes actives
// → Plus petit, plus rapide, plus efficace
// → Le Query Planner le choisira pour les requêtes appropriées
```

---

## Diagnostic des problèmes du Query Planner

### Problème 1 : Le Query Planner choisit le mauvais index

**Symptôme** :

```javascript
db.orders.find({ userId: 123, status: "pending" })
  .explain("executionStats")

// Utilise userId_1 au lieu de userId_1_status_1
```

**Diagnostic** :

```javascript
// 1. Vérifier les index disponibles
db.orders.getIndexes()

// 2. Vérifier le plan avec explain("allPlansExecution")
db.orders.find({ userId: 123, status: "pending" })
  .explain("allPlansExecution")

// 3. Analyser pourquoi l'index composé n'a pas été choisi
```

**Solutions possibles** :

```javascript
// Solution A : Vérifier que l'index composé existe bien
db.orders.createIndex({ userId: 1, status: 1 })

// Solution B : Effacer le cache de plans
db.orders.getPlanCache().clear()

// Solution C : Vérifier les statistiques
db.orders.stats()

// Solution D : En dernier recours, utiliser hint()
db.orders.find({ userId: 123, status: "pending" })
  .hint("userId_1_status_1")
```

### Problème 2 : Performance variable d'une requête

**Symptôme** :
```
Même requête, performances variables :
- Exécution 1 : 5ms
- Exécution 2 : 4ms
- Exécution 3 : 450ms  ← ?!
- Exécution 4 : 6ms
```

**Cause** : Le plan en cache a été invalidé à l'exécution 3, causant une nouvelle compétition de plans.

**Diagnostic** :

```javascript
// Vérifier l'historique du cache
db.orders.getPlanCache().list()

// Chercher "works" (nombre d'exécutions)
// Si "works" est faible, le plan est récent (nouvelle compétition)
```

**Solution** :

```javascript
// Si les plans changent trop souvent :
// 1. Optimiser les index
// 2. Réduire la fréquence des écritures
// 3. Augmenter le seuil de réinitialisation (MongoDB 4.2+)

// Surveiller les changements de plan
db.setProfilingLevel(1, { slowms: 100 })
```

### Problème 3 : COLLSCAN malgré un index

**Symptôme** :

```javascript
db.users.createIndex({ status: 1 })

db.users.find({ status: "active" })
  .explain("executionStats")

// Stage: "COLLSCAN"  ← ?!
```

**Diagnostic** :

```javascript
// Vérifier la sélectivité
const total = db.users.countDocuments()
const matching = db.users.countDocuments({ status: "active" })
const selectivity = (matching / total * 100).toFixed(2)

print(`Sélectivité : ${selectivity}%`)

// Si > 30% : Le COLLSCAN peut être plus rapide
```

**Solution** :

```javascript
// Si la sélectivité est vraiment faible :
// Créer un index partiel
db.users.createIndex(
  { status: 1 },
  {
    partialFilterExpression: {
      status: { $ne: "active" }
    },
    name: "status_inactive"
  }
)

// Pour les inactifs, l'index est efficace
db.users.find({ status: "inactive" })
// → Utilise l'index

// Pour les actifs, COLLSCAN reste optimal
db.users.find({ status: "active" })
// → COLLSCAN (voulu)
```

---

## Bonnes pratiques avec le Query Planner

### 1. Faire confiance au Query Planner (en général)

```
✅ Le Query Planner est sophistiqué et bien optimisé
✅ Il fait généralement les bons choix
✅ Utilisez explain() pour COMPRENDRE, pas pour CONTESTER
✅ N'utilisez hint() que si vous avez PROUVÉ qu'il se trompe
```

### 2. Créer des index optimaux

```
✅ Suivez la règle ESR (Equality, Sort, Range)
✅ Préférez les index composés aux multiples index simples
✅ Testez avec explain() avant le déploiement
✅ Supprimez les index redondants
```

### 3. Surveiller les changements de plans

```javascript
// Activer le profiler pour détecter les requêtes lentes
db.setProfilingLevel(1, { slowms: 100 })

// Surveiller les changements de plans
db.system.profile.find({
  planSummary: { $exists: true }
}).sort({ ts: -1 }).limit(10)
```

### 4. Maintenir des statistiques à jour

```javascript
// En production, MongoDB maintient automatiquement les stats
// Mais après une grosse migration de données :

// Recalculer les statistiques
db.runCommand({ validate: "collection" })

// Effacer le cache de plans
db.collection.getPlanCache().clear()
```

### 5. Documenter les hints utilisés

```javascript
// Si vous DEVEZ utiliser hint() :

// ❌ Mauvais : Sans explication
db.orders.find({ ... }).hint("userId_1")

// ✅ Bon : Documenté
// Hint forcé car le Query Planner choisit userId_1_status_1
// mais userId_1 est plus rapide pour cette requête spécifique
// (vérifié le 2024-12-01 avec explain())
// TODO: Réévaluer après optimisation de la distribution des données
db.orders.find({ ... }).hint("userId_1")
```

---

## Le Query Planner et les agrégations

Le Query Planner fonctionne aussi avec les pipelines d'agrégation :

### Optimisations automatiques

```javascript
// Pipeline original
db.orders.aggregate([
  { $match: { status: "pending" } },
  { $sort: { createdAt: -1 } },
  { $limit: 10 }
])

// Le Query Planner réorganise automatiquement :
// 1. $match en premier (utilise l'index)
// 2. $sort via l'index (pas de SORT stage)
// 3. $limit appliqué tôt (pas besoin de tout charger)
```

### Voir le plan d'une agrégation

```javascript
db.orders.explain("executionStats").aggregate([
  { $match: { status: "pending" } },
  { $sort: { createdAt: -1 } },
  { $limit: 10 }
])

// Ou pour MongoDB 4.2+
db.orders.aggregate(
  [
    { $match: { status: "pending" } },
    { $sort: { createdAt: -1 } },
    { $limit: 10 }
  ],
  { explain: true }
)
```

### Index pour agrégations

```javascript
// Pipeline fréquent
db.orders.aggregate([
  { $match: { userId: 123 } },
  { $group: {
      _id: "$status",
      total: { $sum: "$amount" }
  }}
])

// Index optimal
db.orders.createIndex({ userId: 1, status: 1, amount: 1 })

// Le Query Planner utilisera cet index pour :
// 1. Filtrer par userId (IXSCAN)
// 2. Accéder rapidement à status et amount
// 3. Potentiellement une covered query si projection appropriée
```

---

## Outils de monitoring du Query Planner

### 1. Profiler MongoDB

```javascript
// Activer le profiler
db.setProfilingLevel(2)  // Log toutes les requêtes

// Ou seulement les lentes
db.setProfilingLevel(1, { slowms: 100 })

// Analyser les plans utilisés
db.system.profile.find({
  ns: "mydb.users",
  millis: { $gt: 100 }
}).sort({ ts: -1 }).pretty()
```

### 2. Commandes de diagnostic

```javascript
// Voir tous les plans en cache
db.collection.getPlanCache().list()

// Statistiques du cache
db.collection.aggregate([
  { $planCacheStats: {} }
])

// Plans actifs
db.collection.aggregate([
  { $planCacheStats: {} },
  { $match: { isActive: true } }
])
```

### 3. Logs MongoDB

```javascript
// Dans les logs MongoDB, rechercher :
// - "Slow query" : Requêtes lentes
// - "Plan cache" : Changements de plans
// - "Index selection" : Choix d'index

// Augmenter la verbosité des logs (temporaire)
db.setLogLevel(2, "query")

// Revenir au niveau normal
db.setLogLevel(0, "query")
```

---

## Checklist de diagnostic du Query Planner

### ✅ Ma requête est lente

```
□ J'ai exécuté explain("executionStats")
□ J'ai vérifié le stage (IXSCAN vs COLLSCAN)
□ J'ai calculé le ratio (nReturned / totalDocsExamined)
□ J'ai vérifié quel index est utilisé (ou absence d'index)
□ J'ai comparé avec les index disponibles
□ J'ai testé avec un hint() pour confirmer
□ J'ai vérifié explain("allPlansExecution") pour voir les alternatives
□ J'ai créé/modifié l'index si nécessaire
□ J'ai effacé le cache de plans après modification
□ J'ai re-testé et validé l'amélioration
```

### ✅ Le Query Planner ne choisit pas mon index

```
□ L'index existe vraiment (getIndexes())
□ L'index couvre les champs de ma requête
□ La sélectivité de mon index est bonne (< 30%)
□ L'index n'est pas hidden
□ Le cache de plans n'est pas obsolète
□ J'ai testé avec hint() pour confirmer que l'index est meilleur
□ J'ai vérifié explain("allPlansExecution") pour comprendre pourquoi
□ Les statistiques de collection sont à jour
```

---

## Concepts clés à retenir

### 🎯 Points essentiels

1. **Le Query Planner** choisit comment exécuter vos requêtes
   - Analyse la requête
   - Identifie les index candidats
   - Teste plusieurs plans
   - Choisit et met en cache le meilleur

2. **Facteurs de décision** :
   - Sélectivité des index (plus c'est sélectif, mieux c'est)
   - Couverture des filtres
   - Support du tri
   - Cardinalité des champs

3. **Cache de plans** :
   - Réutilise les plans gagnants
   - Basé sur la "forme" de la requête
   - Invalidé après ~1000 écritures ou modifications d'index

4. **Faire confiance au Query Planner** :
   - Il fait généralement les bons choix
   - Utilisez `hint()` seulement si prouvé nécessaire
   - Créez plutôt l'index optimal

5. **Diagnostiquer avec explain()** :
   - Mode "allPlansExecution" pour voir tous les plans testés
   - Vérifier les plans rejetés et comprendre pourquoi

6. **Influencer sans forcer** :
   - Créer l'index optimal (règle ESR)
   - Supprimer les index redondants
   - Utiliser des index partiels
   - Maintenir les statistiques à jour

---

## Ressources et commandes utiles

### Commandes essentielles

```javascript
// Voir le plan choisi
db.collection.find({ ... }).explain("executionStats")

// Voir tous les plans testés
db.collection.find({ ... }).explain("allPlansExecution")

// Forcer un index
db.collection.find({ ... }).hint("indexName")

// Cache de plans
db.collection.getPlanCache().list()
db.collection.getPlanCache().clear()

// Statistiques
db.collection.stats()
db.collection.aggregate([{ $planCacheStats: {} }])
```

---

## Analogie finale

> **Le Query Planner est comme un chef cuisinier expérimenté :**
>
> **Vous** : "Je veux un plat délicieux rapidement"
>
> **Chef (Query Planner)** :
> - Regarde les ingrédients disponibles (index)
> - Évalue plusieurs recettes possibles (plans)
> - Teste mentalement chaque approche (compétition)
> - Choisit la méthode la plus rapide et efficace
> - Se souvient de ce choix pour les commandes similaires (cache)
>
> **Vous pouvez** :
> ✅ Lui fournir de meilleurs ingrédients (créer index optimaux)
> ✅ Faire confiance à son expertise (généralement)
> ⚠️ Suggérer une recette spécifique (hint) si vous SAVEZ qu'elle est meilleure
> ❌ Mais ne le forcez pas sans raison, c'est lui l'expert !
>
> **Le Query Planner cuisine des requêtes rapides avec les ingrédients (index) que vous lui donnez !** 👨‍🍳

---

**Vous maîtrisez maintenant le fonctionnement du Query Planner !** 🚀

---


⏭️ [Stratégies d'optimisation des requêtes](/05-index-et-optimisation/08-strategies-optimisation.md)
