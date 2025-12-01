🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 5.5 Création et suppression d'index

## Introduction

Maintenant que vous comprenez les types d'index et leurs options, il est temps d'apprendre à **créer** et **supprimer** des index de manière efficace et sécurisée.

La gestion des index est une opération critique qui peut :
- 📈 **Améliorer drastiquement** les performances (si bien fait)
- 🐌 **Dégrader les performances** (si mal fait)
- ⏱️ **Bloquer** temporairement votre application (si mal planifié)
- 💾 **Consommer** des ressources importantes (CPU, RAM, I/O)

Ce chapitre vous guidera à travers les commandes, les stratégies et les bonnes pratiques pour gérer les index en toute sécurité, aussi bien en développement qu'en production.

---

## Création d'index

### 1. Commande de base : createIndex()

#### Syntaxe générale

```javascript
db.collection.createIndex(
  { champ1: ordre, champ2: ordre, ... },  // Définition de l'index
  { option1: valeur, option2: valeur }    // Options (facultatif)
)
```

#### Exemples simples

```javascript
// Index simple ascendant
db.users.createIndex({ email: 1 })

// Index simple descendant
db.posts.createIndex({ publishedAt: -1 })

// Index composé
db.orders.createIndex({ userId: 1, createdAt: -1 })

// Index avec option unique
db.products.createIndex({ sku: 1 }, { unique: true })

// Index avec nom personnalisé
db.users.createIndex(
  { city: 1, age: 1 },
  { name: "city_age_idx" }
)
```

### 2. Retour de la commande

Quand vous créez un index, MongoDB retourne un objet de confirmation :

```javascript
db.users.createIndex({ email: 1 })

// Retour en cas de succès :
{
  "numIndexesBefore": 1,      // Nombre d'index avant
  "numIndexesAfter": 2,       // Nombre d'index après
  "createdCollectionAutomatically": false,
  "ok": 1                     // Succès
}

// Retour si l'index existe déjà (avec même définition) :
{
  "numIndexesBefore": 2,
  "numIndexesAfter": 2,       // Pas de changement
  "note": "all indexes already exist",
  "ok": 1
}
```

### 3. Gestion des erreurs

#### Erreur de doublon avec index unique

```javascript
// Création d'un index unique
db.users.createIndex({ email: 1 }, { unique: true })

// Si des doublons existent déjà :
{
  "ok": 0,
  "errmsg": "Index build failed: 1f23ab45-...: Collection test.users ( ... ) :: caused by :: E11000 duplicate key error",
  "code": 11000,
  "codeName": "DuplicateKey"
}
```

#### Solution pour les doublons

```javascript
// 1. Trouver les doublons
db.users.aggregate([
  { $group: {
      _id: "$email",
      count: { $sum: 1 }
  }},
  { $match: { count: { $gt: 1 } }}
])

// 2. Nettoyer les doublons (garder le premier)
db.users.aggregate([
  { $group: {
      _id: "$email",
      docs: { $push: "$$ROOT" }
  }},
  { $match: { "docs.1": { $exists: true } }}
]).forEach(doc => {
  // Garder le premier, supprimer les autres
  doc.docs.slice(1).forEach(duplicate => {
    db.users.deleteOne({ _id: duplicate._id })
  })
})

// 3. Créer l'index unique
db.users.createIndex({ email: 1 }, { unique: true })
```

### 4. Création d'index sur collection vide vs peuplée

#### Sur collection vide (rapide)

```javascript
// Collection vide
db.newCollection.createIndex({ field: 1 })
// → Création instantanée (< 1 seconde)
```

#### Sur collection peuplée (plus lent)

```javascript
// Collection avec 10 millions de documents
db.largeCollection.createIndex({ field: 1 })
// → Peut prendre plusieurs minutes/heures !

// Progression visible dans les logs :
// Index build: 1/3 - scanning collection
// Index build: 2/3 - sorting keys
// Index build: 3/3 - writing index
```

---

## Modes de création d'index

### 1. Foreground Index Build (par défaut)

#### Caractéristiques

```
Mode Foreground (Avant-plan)
════════════════════════════

Pendant la création :
├─ Database VERROUILLÉE (write lock)
├─ Lectures bloquées
├─ Écritures bloquées
├─ Construction RAPIDE
└─ Application INACCESSIBLE

Après la création :
└─ Index pleinement opérationnel
```

#### Exemple

```javascript
// Création foreground (comportement par défaut)
db.users.createIndex({ email: 1 })

// Pendant la création :
// ⏸️  Application bloquée
// ⏸️  Pas de lecture/écriture possible
// ⚡ Construction rapide
```

#### Temps estimés

```
Collection 1 million docs   : 5-15 secondes
Collection 10 millions docs : 1-5 minutes
Collection 100 millions docs: 10-60 minutes

⚠️  Application INACCESSIBLE pendant toute cette durée !
```

### 2. Background Index Build (arrière-plan)

> ⚠️ **Note importante** : À partir de MongoDB 4.2, le comportement par défaut a été amélioré. Les index sont maintenant construits avec un système hybride qui minimise le verrouillage. L'option `background: true` est **dépréciée** depuis MongoDB 4.2.

#### Ancien comportement (MongoDB < 4.2)

```javascript
// Option background (dépréciée)
db.users.createIndex(
  { email: 1 },
  { background: true }
)

// Pendant la création :
// ✅ Lectures possibles
// ✅ Écritures possibles
// 🐌 Construction plus LENTE
// ⚠️  Verrouillages intermittents possibles
```

#### Nouveau comportement (MongoDB >= 4.2)

```javascript
// Création d'index moderne
db.users.createIndex({ email: 1 })

// MongoDB 4.2+ utilise un algorithme optimisé :
// ✅ Verrouillages minimisés
// ✅ Application reste responsive
// ⚡ Performance équilibrée
// 🎯 Meilleur des deux mondes
```

### 3. Rolling Index Build (déploiement progressif)

Pour les **replica sets** en production, la stratégie recommandée est le **rolling index build** :

```
Rolling Index Build - Stratégie de Production
══════════════════════════════════════════════

Étape 1 : Secondary 1
├─ Arrêter le membre
├─ Construire l'index
├─ Redémarrer le membre
└─ Attendre synchronisation

Étape 2 : Secondary 2
├─ Arrêter le membre
├─ Construire l'index
├─ Redémarrer le membre
└─ Attendre synchronisation

Étape 3 : Primary
├─ Faire un stepDown (devient secondary)
├─ Construire l'index sur le nouveau primary
├─ Construire l'index sur l'ancien primary (devenu secondary)
└─ Terminé

Avantages :
✅ Application toujours disponible
✅ Pas de downtime
✅ Risque minimisé
```

#### Commandes pour rolling build

```javascript
// Sur chaque secondary (un par un) :

// 1. Se connecter au secondary
mongosh --host secondary1.example.com

// 2. Vérifier qu'on est bien sur un secondary
rs.status()

// 3. Créer l'index
db.users.createIndex({ email: 1 })

// 4. Attendre la fin de la construction
// (surveiller avec db.currentOp())

// 5. Répéter pour secondary2, secondary3, etc.

// Sur le primary (en dernier) :

// 1. Forcer une élection (stepDown)
rs.stepDown(60)  // Le primary actuel devient secondary

// 2. Se connecter au nouveau primary
mongosh --host newPrimary.example.com

// 3. Créer l'index
db.users.createIndex({ email: 1 })

// 4. Créer l'index sur l'ancien primary (devenu secondary)
```

---

## Nommage des index

### 1. Nommage automatique

Par défaut, MongoDB génère un nom basé sur les champs et l'ordre :

```javascript
// Index créé
db.users.createIndex({ email: 1 })
// Nom automatique : "email_1"

db.users.createIndex({ city: 1, age: -1 })
// Nom automatique : "city_1_age_-1"

db.products.createIndex({ "specs.weight": 1 })
// Nom automatique : "specs.weight_1"
```

### 2. Nommage personnalisé

```javascript
// Nom personnalisé avec l'option "name"
db.users.createIndex(
  { email: 1 },
  { name: "idx_user_email" }
)

db.orders.createIndex(
  { userId: 1, status: 1, createdAt: -1 },
  { name: "idx_user_orders_recent" }
)
```

### 3. Bonnes pratiques de nommage

```
Convention recommandée :
────────────────────────

Format : idx_<collection>_<champs>_<info>

Exemples :
─────────
✅ "idx_users_email"
✅ "idx_orders_user_status"
✅ "idx_products_category_price"
✅ "idx_logs_timestamp_ttl"

À éviter :
──────────
❌ "index1"
❌ "temp"
❌ "test_idx"
❌ "aaaaa"

Avantages d'un bon nommage :
────────────────────────────
• Compréhension immédiate de l'usage
• Maintenance facilitée
• Documentation auto-descriptive
• Évite les confusions
```

### 4. Limite de longueur du nom

```javascript
// Limite : 128 caractères pour le nom complet
// Format : "<database>.<collection>.$<indexName>"

// ✅ OK
db.users.createIndex(
  { email: 1 },
  { name: "idx_users_email_unique" }
)

// ❌ ERREUR : Nom trop long
db.myVeryLongCollectionNameThatIsReallyTooLongForProduction.createIndex(
  { field: 1 },
  { name: "idx_myVeryLongCollectionNameThatIsReallyTooLongForProduction_field_unique_sparse_partial" }
)
// Error: index name too long
```

---

## Suppression d'index

### 1. Commande dropIndex()

#### Suppression par nom

```javascript
// Supprimer un index spécifique par son nom
db.users.dropIndex("email_1")

// Retour en cas de succès :
{
  "nIndexesWas": 3,     // Nombre d'index avant
  "ok": 1
}

// Retour en cas d'erreur (index inexistant) :
{
  "ok": 0,
  "errmsg": "index not found with name [email_1]",
  "code": 27,
  "codeName": "IndexNotFound"
}
```

#### Suppression par définition

```javascript
// Supprimer par la définition exacte de l'index
db.users.dropIndex({ email: 1 })

// Équivalent à :
db.users.dropIndex("email_1")
```

### 2. Commande dropIndexes()

#### Supprimer tous les index (sauf _id)

```javascript
// Supprimer TOUS les index de la collection
// (l'index _id est préservé)
db.users.dropIndexes()

// Retour :
{
  "nIndexesWas": 5,     // Avait 5 index
  "ok": 1
}
// Reste seulement l'index _id
```

#### Supprimer plusieurs index spécifiques

```javascript
// MongoDB 4.2+ : Supprimer plusieurs index en une commande
db.users.dropIndexes(["email_1", "city_1_age_1"])

// Retour :
{
  "nIndexesWas": 5,
  "ok": 1
}
```

### 3. L'index _id ne peut pas être supprimé

```javascript
// ❌ ERREUR : Impossible de supprimer _id
db.users.dropIndex("_id_")

// Retour :
{
  "ok": 0,
  "errmsg": "cannot drop _id index",
  "code": 72,
  "codeName": "InvalidOptions"
}
```

---

## Vérification des index

### 1. Lister tous les index

```javascript
// Obtenir la liste de tous les index
db.collection.getIndexes()
```

#### Exemple de sortie

```javascript
db.users.getIndexes()

// Retour :
[
  {
    "v": 2,
    "key": { "_id": 1 },
    "name": "_id_"
  },
  {
    "v": 2,
    "key": { "email": 1 },
    "name": "email_1",
    "unique": true
  },
  {
    "v": 2,
    "key": { "city": 1, "age": 1 },
    "name": "city_1_age_1"
  },
  {
    "v": 2,
    "key": { "createdAt": 1 },
    "name": "createdAt_1",
    "expireAfterSeconds": 86400
  }
]
```

### 2. Vérifier si un index existe

```javascript
// Vérifier l'existence d'un index par son nom
function indexExists(collectionName, indexName) {
  const indexes = db[collectionName].getIndexes()
  return indexes.some(idx => idx.name === indexName)
}

// Usage
if (indexExists("users", "email_1")) {
  print("L'index email_1 existe")
} else {
  print("L'index email_1 n'existe pas")
}
```

### 3. Obtenir des statistiques d'index

```javascript
// Statistiques détaillées de la collection
db.users.stats()

// Extrait pertinent :
{
  "nindexes": 4,                    // Nombre total d'index
  "indexSizes": {
    "_id_": 5242880,                // 5 Mo
    "email_1": 2097152,             // 2 Mo
    "city_1_age_1": 3145728,        // 3 Mo
    "createdAt_1": 1048576          // 1 Mo
  },
  "totalIndexSize": 11534336        // ~11 Mo au total
}
```

---

## Reconstruction d'index

### 1. Pourquoi reconstruire un index ?

Les index peuvent devenir **fragmentés** ou **inefficaces** avec le temps :

```
Raisons de reconstruire :
═════════════════════════

• Fragmentation après nombreuses écritures
• Corruption d'index (rare)
• Optimisation de l'espace disque
• Après migration de données massive
• Performance dégradée inexpliquée
```

### 2. Commande reIndex()

```javascript
// Reconstruire TOUS les index d'une collection
db.users.reIndex()

// Retour :
{
  "nIndexesWas": 4,
  "nIndexes": 4,
  "indexes": [
    { "v": 2, "key": { "_id": 1 }, "name": "_id_" },
    { "v": 2, "key": { "email": 1 }, "name": "email_1" },
    // ...
  ],
  "ok": 1
}
```

### 3. Stratégie recommandée : Drop & Recreate

Au lieu de `reIndex()`, la méthode recommandée est :

```javascript
// 1. Noter les index existants
const indexes = db.users.getIndexes()

// 2. Supprimer les index (sauf _id)
db.users.dropIndexes()

// 3. Recréer les index un par un
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ city: 1, age: 1 })
// ...
```

**Avantages** :
- Plus de contrôle sur le processus
- Possibilité de modifier les définitions
- Meilleure visibilité
- Moins de risques

### 4. Attention en production !

```
⚠️  DANGER : reIndex() en production
═══════════════════════════════════════

reIndex() :
├─ VERROUILLE la collection
├─ BLOQUE toutes les opérations
├─ Peut prendre des HEURES
└─ Application INACCESSIBLE

Alternative sécurisée :
├─ Utiliser rolling index rebuild
├─ Créer un nouvel index
├─ Supprimer l'ancien
└─ Application reste disponible
```

---

## Gestion des index en production

### 1. Processus sécurisé de création

```
Processus recommandé pour créer un index en production
═══════════════════════════════════════════════════════

Étape 1 : PLANIFICATION
├─ Identifier le besoin (requêtes lentes)
├─ Analyser avec explain()
├─ Estimer la taille de l'index
└─ Choisir le moment (faible trafic)

Étape 2 : TEST EN DÉVELOPPEMENT
├─ Créer l'index en dev/staging
├─ Mesurer le temps de création
├─ Vérifier l'impact sur les écritures
└─ Valider l'amélioration des requêtes

Étape 3 : TEST EN PRÉ-PRODUCTION
├─ Créer sur environnement miroir
├─ Tester avec charge réaliste
└─ Mesurer impact sur performances

Étape 4 : DÉPLOIEMENT EN PRODUCTION
├─ Si standalone : créer directement (hors heures)
├─ Si replica set : rolling index build
├─ Si sharded : créer sur chaque shard
└─ Surveiller métriques en temps réel

Étape 5 : VALIDATION
├─ Vérifier utilisation avec explain()
├─ Mesurer amélioration des performances
├─ Surveiller métriques (CPU, RAM, I/O)
└─ Documenter l'ajout
```

### 2. Processus sécurisé de suppression

```
Processus recommandé pour supprimer un index en production
══════════════════════════════════════════════════════════

Étape 1 : ÉVALUATION
├─ Analyser l'utilisation ($indexStats)
├─ Vérifier les requêtes dépendantes
└─ Estimer le risque

Étape 2 : TEST AVEC HIDDEN
├─ Masquer l'index : hideIndex()
├─ Surveiller pendant 1-7 jours
├─ Vérifier métriques de performance
└─ Analyser requêtes lentes

Étape 3a : SI AUCUN IMPACT
├─ Supprimer définitivement : dropIndex()
└─ Documenter la suppression

Étape 3b : SI IMPACT NÉGATIF
├─ Rendre visible : unhideIndex()
├─ Analyser les requêtes affectées
└─ Conserver l'index
```

### 3. Surveillance pendant la création

```javascript
// Surveiller les opérations en cours
db.currentOp({
  $or: [
    { op: "command", "command.createIndexes": { $exists: true } },
    { msg: /^Index Build/ }
  ]
})

// Exemple de sortie :
{
  "inprog": [
    {
      "opid": 12345,
      "op": "command",
      "ns": "mydb.users",
      "command": {
        "createIndexes": "users",
        "indexes": [{
          "key": { "email": 1 },
          "name": "email_1"
        }]
      },
      "msg": "Index Build: scanning collection",
      "progress": {
        "done": 2500000,
        "total": 10000000
      },
      "numYields": 0,
      "secs_running": 45
    }
  ],
  "ok": 1
}
```

---

## Impact sur les performances

### 1. Impact pendant la création

```
Ressources consommées pendant la création d'index
══════════════════════════════════════════════════

CPU :
████████████████████ 80-100%
└─ Lecture, tri, écriture des données

RAM :
████████████ 60-80%
└─ Buffer pour tri et construction

Disque I/O :
███████████████████████████ 90-100%
└─ Lecture collection + écriture index

Réseau (replica set) :
██████████ 50%
└─ Réplication vers secondaries
```

### 2. Temps de création estimés

```
Temps de création selon la taille
═══════════════════════════════════

Collection      | Docs      | Taille | Temps estimé
────────────────|-----------|--------|──────────────
Petite          | 10K       | 5 Mo   | < 1 seconde
Moyenne         | 100K      | 50 Mo  | 5-10 secondes
Grande          | 1M        | 500 Mo | 30-60 secondes
Très grande     | 10M       | 5 Go   | 5-15 minutes
Énorme          | 100M      | 50 Go  | 1-3 heures

⚠️  Variables influençant le temps :
• Matériel (CPU, RAM, SSD vs HDD)
• Taille moyenne des documents
• Complexité de l'index (simple vs composé)
• Charge du serveur
```

### 3. Impact après la création

```
Impact permanent d'un index
═══════════════════════════

Lectures :
✅ Plus rapides (IXSCAN vs COLLSCAN)
└─ Gain : 10x à 1000x

Écritures :
⚠️  Plus lentes (mise à jour index + document)
└─ Surcoût : +10% à +50% par index

Espace disque :
⚠️  Augmentation
└─ Index ≈ 5-20% de la taille des données

RAM :
⚠️  Consommation accrue
└─ Working set inclut les index actifs
```

---

## Bonnes pratiques

### 1. Création d'index

```
✅ À FAIRE
══════════

• Créer en dehors des heures de pointe
• Tester d'abord en dev/staging
• Utiliser rolling build pour replica sets
• Nommer explicitement les index importants
• Documenter l'objectif de chaque index
• Mesurer l'impact avec explain()
• Surveiller les métriques pendant la création
• Avoir un plan de rollback

❌ À ÉVITER
═══════════

• Créer sur le primary d'un replica set directement
• Créer plusieurs index en même temps
• Créer aux heures de forte charge
• Créer sans tester l'impact
• Créer "au cas où" sans analyse
• Créer des index redondants
• Ignorer les erreurs de doublon
```

### 2. Suppression d'index

```
✅ À FAIRE
══════════

• Analyser l'utilisation avant de supprimer
• Utiliser hidden pour tester
• Conserver les définitions (sauvegarde)
• Supprimer en dehors des heures de pointe
• Surveiller les performances après
• Documenter la raison de la suppression

❌ À ÉVITER
═══════════

• Supprimer sans analyse préalable
• Supprimer l'index _id
• Supprimer plusieurs index d'un coup
• Supprimer pendant les heures de pointe
• Supprimer sans test préalable (hidden)
```

### 3. Gestion quotidienne

```
✅ ROUTINE DE MAINTENANCE
══════════════════════════

Quotidien :
├─ Surveiller requêtes lentes
├─ Vérifier utilisation des index
└─ Analyser métriques de performance

Hebdomadaire :
├─ Revoir les index peu utilisés
├─ Identifier les requêtes sans index
└─ Planifier optimisations

Mensuel :
├─ Analyser $indexStats
├─ Nettoyer index inutilisés
└─ Documenter changements
```

---

## Commandes utiles de diagnostic

### 1. Statistiques d'utilisation des index

```javascript
// Obtenir les statistiques d'utilisation (MongoDB 3.2+)
db.users.aggregate([{ $indexStats: {} }])

// Exemple de sortie :
[
  {
    "name": "email_1",
    "key": { "email": 1 },
    "host": "server1:27017",
    "accesses": {
      "ops": 15234,           // Nombre d'utilisations
      "since": ISODate("2024-11-01T00:00:00Z")
    }
  },
  {
    "name": "city_1_age_1",
    "key": { "city": 1, "age": 1 },
    "host": "server1:27017",
    "accesses": {
      "ops": 3,               // Très peu utilisé !
      "since": ISODate("2024-11-01T00:00:00Z")
    }
  }
]
```

### 2. Taille des index

```javascript
// Taille de tous les index
db.users.stats().indexSizes

// Exemple :
{
  "_id_": 10485760,           // 10 Mo
  "email_1": 5242880,         // 5 Mo
  "city_1_age_1": 8388608     // 8 Mo
}

// Taille totale
db.users.stats().totalIndexSize  // 23 Mo
```

### 3. Identifier les index inutilisés

```javascript
// Script pour trouver les index peu/pas utilisés
db.users.aggregate([
  { $indexStats: {} },
  { $match: { "accesses.ops": { $lt: 100 } }},  // Moins de 100 utilisations
  { $project: {
      name: 1,
      ops: "$accesses.ops",
      since: "$accesses.since"
  }}
])
```

---

## Scénarios courants

### Scénario 1 : Ajout d'index unique sur données existantes

```javascript
// Problème : Créer index unique mais des doublons existent

// Étape 1 : Trouver les doublons
const duplicates = db.users.aggregate([
  { $group: {
      _id: "$email",
      count: { $sum: 1 },
      ids: { $push: "$_id" }
  }},
  { $match: { count: { $gt: 1 } }}
]).toArray()

print(`${duplicates.length} emails en doublon trouvés`)

// Étape 2 : Décider de la stratégie
// Option A : Garder le plus ancien
duplicates.forEach(dup => {
  const idsToDelete = dup.ids.slice(1)  // Garder le premier
  db.users.deleteMany({ _id: { $in: idsToDelete } })
})

// Option B : Fusionner les données
duplicates.forEach(dup => {
  // Logique métier pour fusionner...
})

// Étape 3 : Créer l'index unique
db.users.createIndex({ email: 1 }, { unique: true })
```

### Scénario 2 : Migration d'index

```javascript
// Objectif : Remplacer un index par un meilleur

// Index actuel (pas optimal)
db.orders.getIndexes()
// { key: { userId: 1 }, name: "userId_1" }
// { key: { status: 1 }, name: "status_1" }

// Nouvelle requête fréquente nécessite un index composé
db.orders.find({ userId: 123, status: "pending" }).sort({ createdAt: -1 })

// Stratégie :
// 1. Créer le nouvel index composé
db.orders.createIndex(
  { userId: 1, status: 1, createdAt: -1 },
  { name: "idx_user_status_date" }
)

// 2. Valider que le nouvel index est utilisé
db.orders.find({ userId: 123, status: "pending" })
  .sort({ createdAt: -1 })
  .explain("executionStats")
// → Vérifier indexName: "idx_user_status_date"

// 3. Masquer les anciens index (test)
db.orders.hideIndex("userId_1")
db.orders.hideIndex("status_1")

// 4. Surveiller pendant quelques jours
// ...

// 5. Supprimer les anciens index
db.orders.dropIndex("userId_1")
db.orders.dropIndex("status_1")
```

### Scénario 3 : Index en urgence (production en feu 🔥)

```javascript
// Symptôme : Application très lente, timeout
// Diagnostic : db.currentOp() montre beaucoup de COLLSCAN

// 1. Identifier la requête problématique
db.currentOp({
  "secs_running": { $gt: 5 },
  "ns": "mydb.orders"
})

// 2. Analyser une requête typique
db.orders.find({ status: "pending" }).explain("executionStats")
// → COLLSCAN sur 10M documents !

// 3. Créer l'index en urgence
// Sur un replica set : Créer d'abord sur un secondary
db.orders.createIndex({ status: 1 })

// 4. Valider l'amélioration immédiate
db.orders.find({ status: "pending" }).explain("executionStats")
// → IXSCAN : 10000 docs examinés au lieu de 10M

// 5. Documenter l'incident et la solution
```

---

## Erreurs courantes et solutions

### Erreur 1 : Index déjà existant

```javascript
// Erreur
db.users.createIndex({ email: 1 })
// Retour : "all indexes already exist"

// Solution : Vérifier d'abord
if (!indexExists("users", "email_1")) {
  db.users.createIndex({ email: 1 })
}
```

### Erreur 2 : Nom d'index trop long

```javascript
// Erreur : Index name too long

// Solution : Utiliser un nom personnalisé plus court
db.collection.createIndex(
  { veryLongFieldName: 1 },
  { name: "idx_short_name" }
)
```

### Erreur 3 : Doublons avec index unique

```javascript
// Erreur : E11000 duplicate key error

// Solution : Nettoyer les doublons AVANT de créer l'index
// (voir Scénario 1 ci-dessus)
```

### Erreur 4 : Mémoire insuffisante

```javascript
// Erreur : Sort exceeded memory limit

// Solution 1 : Augmenter la limite (temporaire)
db.adminCommand({
  setParameter: 1,
  internalQueryExecMaxBlockingSortBytes: 335544320  // 320 Mo
})

// Solution 2 : Créer l'index par étapes (plus sûr)
// Créer l'index normalement (MongoDB gère automatiquement)
db.collection.createIndex({ field: 1 })
```

---

## Checklist de création/suppression

### ✅ Checklist : Avant de créer un index

```
□ J'ai identifié une requête lente qui bénéficierait de cet index
□ J'ai utilisé explain() pour confirmer le besoin
□ J'ai testé l'index en développement
□ J'ai estimé la taille de l'index
□ J'ai vérifié qu'un index similaire n'existe pas déjà
□ J'ai choisi un nom d'index explicite
□ J'ai planifié la création en dehors des heures de pointe
□ J'ai préparé un plan de rollback
□ J'ai les permissions nécessaires
□ J'ai prévenu l'équipe
```

### ✅ Checklist : Avant de supprimer un index

```
□ J'ai analysé l'utilisation de l'index ($indexStats)
□ J'ai identifié les requêtes qui utilisent cet index
□ J'ai testé avec hideIndex() pendant plusieurs jours
□ J'ai confirmé qu'aucune dégradation n'a été observée
□ J'ai sauvegardé la définition de l'index
□ J'ai planifié la suppression en dehors des heures de pointe
□ J'ai un plan de recréation rapide si nécessaire
□ J'ai les permissions nécessaires
□ J'ai prévenu l'équipe
```

---

## Concepts clés à retenir

### 🎯 Points essentiels

1. **Création d'index** :
   - `createIndex()` : Commande principale
   - Nom automatique ou personnalisé avec `name`
   - Retour de confirmation avec `numIndexesBefore/After`

2. **Suppression d'index** :
   - `dropIndex()` : Supprimer un index spécifique
   - `dropIndexes()` : Supprimer tous sauf _id
   - L'index _id ne peut JAMAIS être supprimé

3. **Stratégies de production** :
   - **Replica set** : Rolling index build
   - **Teste avant suppression** : Utiliser `hidden: true`
   - **Surveiller** : `currentOp()`, `$indexStats`

4. **Impact** :
   - Création : Consomme CPU, RAM, I/O
   - Temps : De secondes à heures selon la taille
   - Permanent : Espace disque + écritures plus lentes

5. **Bonnes pratiques** :
   - Tester en dev/staging d'abord
   - Créer en dehors des heures de pointe
   - Nommer explicitement les index importants
   - Documenter chaque changement
   - Avoir un plan de rollback

6. **Diagnostic** :
   - `getIndexes()` : Lister les index
   - `$indexStats` : Statistiques d'utilisation
   - `explain()` : Vérifier l'utilisation effective

---

## Ressources pour aller plus loin

### Commandes utiles à mémoriser

```javascript
// Gestion de base
db.collection.createIndex({ field: 1 })
db.collection.dropIndex("indexName")
db.collection.getIndexes()

// Diagnostics
db.collection.aggregate([{ $indexStats: {} }])
db.collection.stats().indexSizes
db.currentOp({ op: "command", "command.createIndexes": { $exists: true } })

// Tests
db.collection.hideIndex("indexName")
db.collection.unhideIndex("indexName")

// Analyse
db.collection.find({ ... }).explain("executionStats")
```

---

## Analogie finale

> **Créer et supprimer des index, c'est comme gérer des étagères dans un entrepôt :**
>
> **Créer un index** = Installer une nouvelle étagère
> - Prend du temps à installer
> - Nécessite de l'espace
> - Rend le rangement futur plus long (un article de plus à classer)
> - MAIS facilite grandement la recherche
>
> **Supprimer un index** = Retirer une étagère
> - Libère de l'espace
> - Accélère le rangement
> - MAIS rend la recherche plus difficile
>
> **Le bon équilibre** : Avoir juste assez d'étagères pour trouver rapidement ce dont on a besoin, sans surcharger l'entrepôt ! 📦

---

**Vous maîtrisez maintenant la création et la suppression d'index de manière sécurisée et efficace !** 🚀

---


⏭️ [Analyse des requêtes avec explain()](/05-index-et-optimisation/06-analyse-explain.md)
