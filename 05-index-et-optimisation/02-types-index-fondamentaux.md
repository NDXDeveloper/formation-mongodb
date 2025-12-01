🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.2 Types d'Index Fondamentaux

## Introduction

Après avoir compris l'importance des index dans MongoDB (voir [5.1 Comprendre l'importance des index](01-importance-des-index.md)), il est temps d'explorer les **types d'index fondamentaux** qui constituent la base de toute stratégie d'optimisation dans MongoDB.

MongoDB propose trois types d'index fondamentaux qui couvrent la majorité des cas d'usage courants :

1. **Index Simple (Single Field)** - Index sur un seul champ
2. **Index Composé (Compound)** - Index sur plusieurs champs
3. **Index Multiclé (Multikey)** - Index sur des champs contenant des tableaux

Ces trois types d'index constituent les **piliers** de l'indexation dans MongoDB. Maîtriser leur fonctionnement, leurs avantages et leurs limitations est essentiel pour créer des applications performantes et scalables.

---

## Vue d'Ensemble des Types Fondamentaux

### 1. Index Simple (Single Field Index)

**Définition** : Un index créé sur un seul champ d'un document.

**Syntaxe** :
```javascript
db.collection.createIndex({ champ: 1 })
```

**Caractéristiques** :
- ✅ Le plus simple à comprendre et à utiliser
- ✅ Point de départ pour toute stratégie d'indexation
- ✅ Efficace pour les requêtes filtrant ou triant sur un seul champ
- ✅ Faible coût en termes d'espace et de maintenance

**Cas d'usage typique** :
```javascript
// Recherche par email
db.users.find({ email: "alice@example.com" })

// Index recommandé
db.users.createIndex({ email: 1 })
```

**Quand l'utiliser** :
- Requêtes fréquentes sur un seul champ
- Tri sur un seul champ
- Champs uniques (email, username, identifiant)
- Comme base avant de passer à des index plus complexes

---

### 2. Index Composé (Compound Index)

**Définition** : Un index créé sur plusieurs champs d'un document, dans un ordre spécifique.

**Syntaxe** :
```javascript
db.collection.createIndex({ champ1: 1, champ2: 1, champ3: -1 })
```

**Caractéristiques** :
- ✅ Optimise les requêtes multi-critères
- ✅ L'ordre des champs est crucial
- ✅ Supporte le concept de "préfixe d'index"
- ⚠️ Plus complexe à concevoir correctement
- ⚠️ Nécessite de comprendre la règle ESR (Equality, Sort, Range)

**Cas d'usage typique** :
```javascript
// Recherche par ville ET statut
db.users.find({ city: "Paris", isActive: true })

// Index recommandé
db.users.createIndex({ city: 1, isActive: 1 })
```

**Quand l'utiliser** :
- Requêtes fréquentes sur plusieurs champs combinés
- Requêtes avec filtrage ET tri
- Optimisation de requêtes complexes répétitives
- Création de "covered queries" (requêtes couvertes)

---

### 3. Index Multiclé (Multikey Index)

**Définition** : Un index spécialisé pour les champs contenant des tableaux, créant une entrée d'index pour chaque élément du tableau.

**Syntaxe** :
```javascript
// Syntaxe identique, MongoDB détecte automatiquement les tableaux
db.collection.createIndex({ champTableau: 1 })
```

**Caractéristiques** :
- ✅ Création automatique (transparente)
- ✅ Optimise les recherches dans les tableaux
- ✅ Fonctionne avec tableaux de valeurs simples ou de sous-documents
- ⚠️ Plus volumineux qu'un index simple
- ⚠️ Maximum un seul champ tableau par index composé

**Cas d'usage typique** :
```javascript
// Document avec tableau de tags
{
  _id: 1,
  name: "Article",
  tags: ["mongodb", "database", "nosql"]
}

// Recherche dans le tableau
db.articles.find({ tags: "mongodb" })

// Index multiclé (créé automatiquement)
db.articles.createIndex({ tags: 1 })
```

**Quand l'utiliser** :
- Systèmes de tags
- Catégories multiples
- Permissions et rôles
- Toute recherche sur des valeurs dans un tableau

---

## Comparaison des Types Fondamentaux

| Critère | Index Simple | Index Composé | Index Multiclé |
|---------|--------------|---------------|----------------|
| **Nombre de champs** | 1 | 2 à 32 | 1 (contenant un tableau) |
| **Complexité** | ⭐ Faible | ⭐⭐⭐ Moyenne | ⭐⭐ Moyenne |
| **Taille** | ⭐ Petite | ⭐⭐ Moyenne | ⭐⭐⭐ Grande |
| **Performance lecture** | ⭐⭐⭐ Excellente | ⭐⭐⭐ Excellente | ⭐⭐⭐ Excellente |
| **Impact écriture** | ⭐ Faible | ⭐⭐ Moyen | ⭐⭐⭐ Élevé |
| **Cas d'usage** | Requêtes simples | Requêtes complexes | Recherche dans tableaux |
| **Difficulté conception** | Facile | Difficile | Facile |

---

## Comment Choisir le Bon Type d'Index

### Arbre de Décision

```
Votre requête filtre sur...

├─ UN SEUL CHAMP (non-tableau)
│  └─→ Index Simple
│     Exemple : db.users.find({ email: "..." })
│
├─ PLUSIEURS CHAMPS combinés
│  └─→ Index Composé
│     Exemple : db.orders.find({ status: "...", customerId: ... })
│
└─ UN CHAMP CONTENANT UN TABLEAU
   └─→ Index Multiclé
      Exemple : db.articles.find({ tags: "mongodb" })
```

### Questions à Se Poser

#### 1. Combien de champs sont utilisés dans la requête ?

- **Un seul champ** → Index Simple
- **Plusieurs champs** → Index Composé

#### 2. Le champ contient-il un tableau ?

- **Oui** → Index Multiclé
- **Non** → Index Simple ou Composé

#### 3. La requête combine filtrage et tri ?

- **Oui, sur plusieurs champs** → Index Composé (avec règle ESR)
- **Non, juste filtrage** → Index Simple ou Multiclé selon le type de données

#### 4. La requête est-elle fréquente et critique ?

- **Très fréquente** → Investir du temps dans un index composé optimal
- **Occasionnelle** → Un index simple peut suffire

---

## Exemples de Scénarios Réels

### Scénario 1 : Application E-commerce

**Collection de produits** :
```javascript
{
  _id: ObjectId("..."),
  name: "Laptop Dell XPS",
  category: "Electronics",
  price: 1299.99,
  tags: ["laptop", "computer", "dell"],
  inStock: true,
  rating: 4.5
}
```

**Index recommandés** :

```javascript
// Index simple : Recherche par catégorie
db.products.createIndex({ category: 1 })

// Index composé : Recherche par catégorie avec tri par prix
db.products.createIndex({ category: 1, price: 1 })

// Index multiclé : Recherche par tags
db.products.createIndex({ tags: 1 })

// Index composé avec multiclé : Catégorie + tags + tri par note
db.products.createIndex({ category: 1, tags: 1, rating: -1 })
```

**Requêtes optimisées** :
```javascript
// Utilise l'index simple sur category
db.products.find({ category: "Electronics" })

// Utilise l'index composé category + price
db.products.find({ category: "Electronics" }).sort({ price: 1 })

// Utilise l'index multiclé sur tags
db.products.find({ tags: "laptop" })

// Utilise l'index composé avec multiclé
db.products.find({
  category: "Electronics",
  tags: "laptop"
}).sort({ rating: -1 })
```

### Scénario 2 : Gestion d'Utilisateurs

**Collection d'utilisateurs** :
```javascript
{
  _id: ObjectId("..."),
  email: "alice@example.com",
  username: "alice",
  city: "Paris",
  isActive: true,
  roles: ["user", "editor"],
  lastLoginAt: ISODate("2024-01-15T10:30:00Z")
}
```

**Index recommandés** :

```javascript
// Index simple unique : Email (authentification)
db.users.createIndex({ email: 1 }, { unique: true })

// Index simple unique : Username
db.users.createIndex({ username: 1 }, { unique: true })

// Index composé : Recherche utilisateurs actifs par ville
db.users.createIndex({ city: 1, isActive: 1 })

// Index multiclé : Recherche par rôles
db.users.createIndex({ roles: 1 })

// Index composé : Activité récente des utilisateurs d'une ville
db.users.createIndex({ city: 1, lastLoginAt: -1 })
```

### Scénario 3 : Système de Logs

**Collection de logs** :
```javascript
{
  _id: ObjectId("..."),
  timestamp: ISODate("2024-01-15T14:30:00Z"),
  level: "error",
  service: "authentication",
  message: "Login failed",
  tags: ["security", "authentication", "failed-login"],
  userId: 5678
}
```

**Index recommandés** :

```javascript
// Index composé : Recherche par niveau et service avec tri temporel
db.logs.createIndex({ level: 1, service: 1, timestamp: -1 })

// Index multiclé : Recherche par tags
db.logs.createIndex({ tags: 1 })

// Index composé : Logs d'un utilisateur par date
db.logs.createIndex({ userId: 1, timestamp: -1 })

// Index simple : Recherche par date (pour archivage)
db.logs.createIndex({ timestamp: -1 })
```

---

## Stratégie d'Indexation Progressive

### Étape 1 : Commencer Simple

Débutez avec des **index simples** sur les champs les plus fréquemment interrogés :

```javascript
// Identifier les champs les plus utilisés dans les requêtes
db.users.createIndex({ email: 1 })
db.orders.createIndex({ customerId: 1 })
db.products.createIndex({ category: 1 })
```

### Étape 2 : Analyser les Performances

Utilisez `explain()` pour identifier les requêtes lentes :

```javascript
// Analyser une requête
db.orders.find({
  customerId: 1234,
  status: "pending"
}).explain("executionStats")
```

### Étape 3 : Optimiser avec des Index Composés

Si les requêtes multi-critères sont fréquentes et lentes, créez des **index composés** :

```javascript
// Optimiser la requête précédente
db.orders.createIndex({ customerId: 1, status: 1 })
```

### Étape 4 : Gérer les Tableaux

Pour les champs contenant des tableaux, les **index multiclé** sont créés automatiquement :

```javascript
// Index multiclé automatique
db.products.createIndex({ tags: 1 })
```

### Étape 5 : Raffiner et Maintenir

- Surveillez l'utilisation des index avec `$indexStats`
- Supprimez les index inutilisés
- Ajustez l'ordre des champs dans les index composés si nécessaire

---

## Règles d'Or pour les Index Fondamentaux

### ✅ Bonnes Pratiques Générales

1. **Commencer petit**
   - Créez d'abord des index simples
   - Passez aux index composés seulement si nécessaire

2. **Analyser avant d'indexer**
   - Identifiez vos requêtes les plus fréquentes
   - Utilisez le profiler de requêtes

3. **Suivre la règle ESR pour les index composés**
   - **E**quality (égalité) → **S**ort (tri) → **R**ange (plage)

4. **Un index à la fois**
   - Créez un index
   - Mesurez l'impact
   - Ajustez si nécessaire

5. **Documenter vos index**
   - Expliquez pourquoi chaque index existe
   - Notez les requêtes qu'il optimise

### ❌ Erreurs Courantes à Éviter

1. **Sur-indexation**
   ```javascript
   // ❌ Trop d'index ralentit les écritures
   db.users.createIndex({ email: 1 })
   db.users.createIndex({ username: 1 })
   db.users.createIndex({ firstName: 1 })
   db.users.createIndex({ lastName: 1 })
   db.users.createIndex({ age: 1 })
   db.users.createIndex({ city: 1 })
   // ... 20 index au total = problème !
   ```

2. **Index redondants**
   ```javascript
   // ❌ Index redondant
   db.users.createIndex({ city: 1, age: 1 })
   db.users.createIndex({ city: 1 })  // ← Redondant !

   // ✅ Le premier index couvre déjà les requêtes sur city seul
   ```

3. **Mauvais ordre dans les index composés**
   ```javascript
   // ❌ Mauvais ordre (range en premier)
   db.products.createIndex({ price: 1, category: 1 })

   // ✅ Bon ordre (equality en premier)
   db.products.createIndex({ category: 1, price: 1 })
   ```

4. **Ignorer les index multiclé**
   ```javascript
   // ❌ Ne pas indexer les tableaux fréquemment recherchés
   db.articles.find({ tags: "mongodb" })  // Lent sans index

   // ✅ Créer l'index multiclé
   db.articles.createIndex({ tags: 1 })
   ```

5. **Créer des index sans les tester**
   ```javascript
   // ❌ Créer un index sans vérifier son utilisation
   db.collection.createIndex({ field: 1 })

   // ✅ Toujours vérifier avec explain()
   db.collection.find({ field: "value" }).explain("executionStats")
   ```

---

## Impact des Index sur les Performances

### Lectures (Queries)

**Sans index** :
```
Collection Scan (COLLSCAN)
- Examine tous les documents (100%)
- Temps : O(n) - linéaire
- Lent sur grandes collections
```

**Avec index approprié** :
```
Index Scan (IXSCAN)
- Examine seulement les documents pertinents (<1%)
- Temps : O(log n) - logarithmique
- Rapide même sur très grandes collections
```

**Amélioration typique** : 10x à 1000x plus rapide

### Écritures (Insert/Update/Delete)

**Sans index** :
```
INSERT → Écriture document uniquement
Temps : ~1ms
```

**Avec N index** :
```
INSERT → Écriture document + Mise à jour N index
Temps : ~1ms + (N × 0.5ms)

Exemple avec 5 index : ~1ms + 2.5ms = 3.5ms
```

**Impact** : Les écritures sont plus lentes, mais généralement acceptable

### Espace Disque

**Taille typique des index** :
```
Index simple : 5-15% de la taille des données
Index composé : 10-25% de la taille des données
Index multiclé : 15-40% de la taille des données
```

**Exemple** :
```
Collection : 10 GB de données
3 index simples : ~1.5 GB
2 index composés : ~4 GB
1 index multiclé : ~2 GB
Total : ~17.5 GB (données + index)
```

---

## Quand Utiliser Chaque Type

### Utilisez un Index Simple quand...

- ✅ Vous filtrez ou triez sur **un seul champ**
- ✅ C'est un champ **unique** (email, username)
- ✅ Vous débutez votre stratégie d'indexation
- ✅ Le champ a une **haute cardinalité** (nombreuses valeurs distinctes)

**Exemple** :
```javascript
db.users.find({ email: "alice@example.com" })
// → Index simple sur email
```

### Utilisez un Index Composé quand...

- ✅ Vous filtrez sur **plusieurs champs combinés**
- ✅ Vous combinez **filtrage et tri**
- ✅ Vous voulez des **covered queries**
- ✅ Les requêtes multi-critères sont **fréquentes**

**Exemple** :
```javascript
db.orders.find({
  status: "pending",
  customerId: 1234
}).sort({ createdAt: -1 })
// → Index composé sur { status: 1, customerId: 1, createdAt: -1 }
```

### Utilisez un Index Multiclé quand...

- ✅ Vous recherchez dans des **tableaux**
- ✅ Vous avez des **systèmes de tags**
- ✅ Vous gérez des **catégories multiples**
- ✅ Vous utilisez des **permissions/rôles**

**Exemple** :
```javascript
db.articles.find({ tags: "mongodb" })
// → Index multiclé sur tags
```

---

## Outils et Commandes de Base

### Créer un Index

```javascript
// Index simple
db.collection.createIndex({ field: 1 })

// Index composé
db.collection.createIndex({ field1: 1, field2: -1 })

// Index avec options
db.collection.createIndex(
  { field: 1 },
  { name: "idx_custom_name", unique: true }
)
```

### Lister les Index

```javascript
db.collection.getIndexes()
```

### Analyser une Requête

```javascript
db.collection.find({ field: "value" }).explain("executionStats")
```

### Supprimer un Index

```javascript
db.collection.dropIndex("index_name")
// ou
db.collection.dropIndex({ field: 1 })
```

### Statistiques d'Utilisation

```javascript
db.collection.aggregate([{ $indexStats: {} }])
```

---

## Progression dans l'Apprentissage

Cette section introduit les trois types d'index fondamentaux. Pour maîtriser chacun d'eux en profondeur, consultez les sections détaillées suivantes :

### 📖 Sections Détaillées

1. **[5.2.1 Index Simple (Single Field)](./02.1-index-simple.md)**
   - Syntaxe détaillée et exemples
   - Ordre ascendant vs descendant
   - Index sur champs imbriqués
   - Index unique et options
   - Cas d'usage et bonnes pratiques

2. **[5.2.2 Index Composé (Compound)](./02.2-index-compose.md)**
   - Ordre des champs et son importance
   - Concept de préfixe d'index
   - Règle ESR (Equality, Sort, Range)
   - Ordre de tri dans les index composés
   - Stratégies d'optimisation avancées

3. **[5.2.3 Index Multiclé (Multikey)](./02.3-index-multicle.md)**
   - Fonctionnement avec les tableaux
   - Index sur tableaux de sous-documents
   - Limitations et considérations
   - Opérateurs de tableaux ($elemMatch, $all)
   - Alternatives et cas avancés

### 🎯 Parcours Recommandé

**Pour les débutants** :
1. Commencez par les **index simples** (5.2.1)
2. Pratiquez la création et l'analyse
3. Passez ensuite aux **index composés** (5.2.2)
4. Terminez avec les **index multiclé** (5.2.3)

**Pour les utilisateurs intermédiaires** :
1. Révisez rapidement les **index simples**
2. Concentrez-vous sur les **index composés** et la règle ESR
3. Explorez les **index multiclé** pour les cas avancés

**Pour les experts** :
- Utilisez cette section comme référence rapide
- Consultez les sections détaillées pour des cas spécifiques
- Passez directement aux **index spécialisés** (Section 5.3)

---

## Conclusion

Les trois types d'index fondamentaux (simple, composé, multiclé) forment la base de toute stratégie d'optimisation dans MongoDB. Comprendre quand et comment utiliser chacun d'eux est essentiel pour :

- ✅ Améliorer drastiquement les performances de lecture
- ✅ Réduire la charge serveur et les temps de réponse
- ✅ Permettre le passage à l'échelle de votre application
- ✅ Optimiser l'utilisation des ressources (CPU, RAM, disque)

### Points Clés à Retenir

- 🔑 **Index Simple** : Un champ, simple et efficace
- 🔑 **Index Composé** : Plusieurs champs, puissant mais nécessite de la réflexion
- 🔑 **Index Multiclé** : Pour les tableaux, automatique et transparent
- 🔑 Commencez simple, optimisez ensuite
- 🔑 Toujours mesurer avec `explain()`
- 🔑 Équilibrez performances lecture vs écriture
- 🔑 Documentez vos choix d'indexation

### Prochaines Étapes

Après avoir maîtrisé ces types fondamentaux, vous serez prêt à explorer :
- **[5.3 Index Spécialisés](./03-index-specialises.md)** : Index texte, géospatial, haché, wildcard, TTL
- **[5.4 Options et Modificateurs d'Index](./04-options-modificateurs-index.md)** : Unique, partiel, sparse, caché
- **[5.5 Création et Suppression d'Index](./05-creation-suppression-index.md)** : Gestion en production
- **[5.6 Analyse avec explain()](./06-analyse-explain.md)** : Diagnostic approfondi

---

**📚 Ressources Complémentaires**
- [Documentation officielle MongoDB - Indexes](https://docs.mongodb.com/manual/indexes/)
- [Index Types Overview](https://docs.mongodb.com/manual/indexes/#index-types)
- [Indexing Strategies](https://docs.mongodb.com/manual/applications/indexes/)
- [Performance Best Practices](https://www.mongodb.com/basics/best-practices)

⏭️ [Index simple (Single Field)](/05-index-et-optimisation/02.1-index-simple.md)
