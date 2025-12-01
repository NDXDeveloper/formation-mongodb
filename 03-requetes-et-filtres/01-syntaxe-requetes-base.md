🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 3.1 Syntaxe des Requêtes de Base

## Introduction

Les requêtes MongoDB permettent de rechercher et filtrer des documents dans vos collections. Contrairement au SQL où vous écrivez des instructions `SELECT`, MongoDB utilise des méthodes JavaScript et des objets JSON pour interroger vos données. Cette approche est plus naturelle pour les développeurs travaillant avec JavaScript et offre une grande flexibilité.

Dans ce chapitre, nous allons explorer la syntaxe fondamentale des requêtes MongoDB, en commençant par les bases pour construire progressivement votre compréhension.

---

## Structure Générale d'une Requête MongoDB

Une requête MongoDB suit généralement cette structure :

```javascript
db.collection.find(query, projection)
```

Où :
- **`db`** : fait référence à la base de données actuelle
- **`collection`** : le nom de votre collection
- **`query`** : un document JSON spécifiant les critères de recherche (optionnel)
- **`projection`** : un document JSON spécifiant quels champs inclure ou exclure (optionnel)

### Exemple Simple

```javascript
// Récupérer tous les documents d'une collection
db.users.find()

// Récupérer les utilisateurs ayant l'âge de 25 ans
db.users.find({ age: 25 })

// Récupérer uniquement le nom et l'email des utilisateurs
db.users.find({}, { name: 1, email: 1 })
```

---

## La Méthode `find()`

La méthode `find()` est la méthode principale pour interroger des documents dans MongoDB. Elle retourne un **curseur** qui pointe vers les documents correspondants.

### Syntaxe de Base

```javascript
db.collection.find(
    { champ: valeur },      // Critères de recherche
    { champ1: 1, champ2: 0 } // Projection (optionnelle)
)
```

### Retourner Tous les Documents

Pour récupérer tous les documents d'une collection sans aucun filtre :

```javascript
db.products.find()
```

Ou de manière équivalente avec un objet vide :

```javascript
db.products.find({})
```

### Recherche par Égalité

La forme la plus simple de requête est la recherche par égalité exacte :

```javascript
// Trouver tous les produits avec le nom "Laptop"
db.products.find({ name: "Laptop" })

// Trouver tous les utilisateurs avec le statut "active"
db.users.find({ status: "active" })

// Trouver les commandes avec un montant de 100
db.orders.find({ amount: 100 })
```

### Recherche avec Plusieurs Critères

Vous pouvez combiner plusieurs critères dans un même document de requête. Par défaut, MongoDB applique un **ET logique** (AND) entre les critères :

```javascript
// Trouver les utilisateurs actifs âgés de 25 ans
db.users.find({
    status: "active",
    age: 25
})

// Trouver les produits de catégorie "Electronics" avec un prix de 500
db.products.find({
    category: "Electronics",
    price: 500
})
```

Dans ces exemples, **tous les critères** doivent être satisfaits pour qu'un document soit retourné.

---

## La Méthode `findOne()`

La méthode `findOne()` fonctionne de manière similaire à `find()`, mais retourne **un seul document** au lieu d'un curseur. Elle est utile quand vous savez que vous ne voulez qu'un seul résultat.

### Syntaxe

```javascript
db.collection.findOne(query, projection)
```

### Exemples

```javascript
// Récupérer un seul utilisateur avec l'email spécifié
db.users.findOne({ email: "john@example.com" })

// Récupérer le premier produit de la collection
db.products.findOne()

// Récupérer un utilisateur par son _id
db.users.findOne({ _id: ObjectId("507f1f77bcf86cd799439011") })
```

**Note importante** : `findOne()` retourne `null` si aucun document ne correspond aux critères.

---

## Format des Documents de Requête

Les requêtes MongoDB sont exprimées sous forme de documents JSON (ou BSON). Ces documents suivent une structure clé-valeur où :
- La **clé** est le nom du champ à filtrer
- La **valeur** est la condition à appliquer

### Format Simple

```javascript
{ champ: valeur }
```

**Exemple** :
```javascript
db.books.find({ author: "George Orwell" })
```

### Format avec Opérateurs

Pour des conditions plus complexes, MongoDB utilise des opérateurs préfixés par `$` :

```javascript
{ champ: { $operateur: valeur } }
```

**Exemples** :
```javascript
// Trouver les produits avec un prix supérieur à 100
db.products.find({ price: { $gt: 100 } })

// Trouver les utilisateurs dont l'âge n'est pas 25
db.users.find({ age: { $ne: 25 } })
```

Nous explorerons ces opérateurs en détail dans les sections suivantes.

---

## Recherche dans des Documents Imbriqués

MongoDB permet de rechercher dans des champs imbriqués en utilisant la **notation pointée** (dot notation).

### Structure d'un Document Imbriqué

Imaginons une collection `users` avec des documents comme celui-ci :

```javascript
{
    _id: 1,
    name: "Alice",
    address: {
        street: "123 Main St",
        city: "Paris",
        zipCode: "75001"
    }
}
```

### Requêtes sur Champs Imbriqués

Pour rechercher dans un champ imbriqué, utilisez la notation pointée entre guillemets :

```javascript
// Trouver les utilisateurs habitant à Paris
db.users.find({ "address.city": "Paris" })

// Trouver les utilisateurs avec un code postal spécifique
db.users.find({ "address.zipCode": "75001" })
```

**Important** : Les guillemets autour du chemin sont obligatoires en notation pointée.

---

## Recherche dans des Tableaux

MongoDB facilite la recherche dans des tableaux de valeurs.

### Structure avec Tableau

```javascript
{
    _id: 1,
    name: "Alice",
    hobbies: ["reading", "swimming", "coding"]
}
```

### Recherche d'une Valeur dans un Tableau

Pour vérifier si un tableau contient une valeur spécifique :

```javascript
// Trouver les utilisateurs ayant "reading" comme hobby
db.users.find({ hobbies: "reading" })
```

MongoDB vérifie automatiquement si la valeur existe dans le tableau.

### Recherche Exacte de Tableau

Pour trouver un document avec un tableau contenant **exactement** les valeurs spécifiées dans le même ordre :

```javascript
// Tableau exact avec le même ordre
db.users.find({ hobbies: ["reading", "swimming", "coding"] })
```

Cette requête ne correspondra qu'aux documents ayant exactement ce tableau dans cet ordre.

---

## Affichage des Résultats

Par défaut, `find()` retourne un curseur. Dans le shell MongoDB (mongosh), les résultats sont automatiquement itérés et affichés.

### Méthodes Utiles pour Formater l'Affichage

#### `pretty()`

Affiche les documents de manière formatée et lisible :

```javascript
db.users.find().pretty()
```

**Note** : Dans les versions récentes de mongosh, l'affichage est automatiquement formaté.

#### `limit()`

Limite le nombre de documents retournés :

```javascript
// Afficher seulement 5 utilisateurs
db.users.find().limit(5)
```

#### `count()` ou `countDocuments()`

Compte le nombre de documents correspondants sans les afficher :

```javascript
// Compter tous les utilisateurs
db.users.countDocuments()

// Compter les utilisateurs actifs
db.users.countDocuments({ status: "active" })
```

#### `toArray()`

Convertit le curseur en tableau JavaScript (utile dans les applications) :

```javascript
const users = db.users.find({ status: "active" }).toArray()
```

---

## Chaînage des Méthodes

MongoDB permet de chaîner plusieurs méthodes pour construire des requêtes plus complexes :

```javascript
// Trouver les 10 premiers utilisateurs actifs, triés par nom
db.users
    .find({ status: "active" })
    .sort({ name: 1 })
    .limit(10)
```

Les méthodes sont exécutées de gauche à droite, permettant une construction intuitive des requêtes.

---

## Le Champ Spécial `_id`

Chaque document MongoDB possède un champ unique `_id`. Par défaut, MongoDB génère automatiquement un `ObjectId` pour ce champ.

### Recherche par `_id`

```javascript
// Recherche par ObjectId
db.users.findOne({ _id: ObjectId("507f1f77bcf86cd799439011") })

// Si l'_id est une chaîne personnalisée
db.products.findOne({ _id: "PROD-001" })
```

**Important** : Les ObjectId doivent être créés avec le constructeur `ObjectId()` lors des requêtes.

---

## Documents Retournés

### Structure du Résultat avec `find()`

`find()` retourne un **curseur**, pas directement les documents. Le curseur est un pointeur vers les résultats qui peuvent être itérés.

```javascript
// Le curseur peut être stocké dans une variable
const cursor = db.users.find({ status: "active" })

// Itération manuelle
while (cursor.hasNext()) {
    printjson(cursor.next())
}
```

### Structure du Résultat avec `findOne()`

`findOne()` retourne directement **un document** ou `null` :

```javascript
const user = db.users.findOne({ email: "john@example.com" })

if (user) {
    print("Utilisateur trouvé: " + user.name)
} else {
    print("Aucun utilisateur trouvé")
}
```

---

## Exemples Pratiques Complets

### Exemple 1 : Collection de Produits

```javascript
// Collection: products
{
    _id: ObjectId("..."),
    name: "Laptop",
    brand: "Dell",
    price: 899.99,
    category: "Electronics",
    inStock: true
}
```

**Requêtes** :

```javascript
// Tous les produits en stock
db.products.find({ inStock: true })

// Produits de la marque Dell
db.products.find({ brand: "Dell" })

// Produits électroniques en stock
db.products.find({
    category: "Electronics",
    inStock: true
})
```

### Exemple 2 : Collection d'Utilisateurs

```javascript
// Collection: users
{
    _id: ObjectId("..."),
    username: "johndoe",
    email: "john@example.com",
    age: 30,
    status: "active",
    profile: {
        firstName: "John",
        lastName: "Doe"
    },
    tags: ["developer", "javascript"]
}
```

**Requêtes** :

```javascript
// Utilisateur par email
db.users.findOne({ email: "john@example.com" })

// Utilisateurs actifs
db.users.find({ status: "active" })

// Utilisateurs avec le tag "developer"
db.users.find({ tags: "developer" })

// Recherche par prénom (champ imbriqué)
db.users.find({ "profile.firstName": "John" })
```

### Exemple 3 : Collection de Commandes

```javascript
// Collection: orders
{
    _id: ObjectId("..."),
    orderNumber: "ORD-12345",
    customerId: ObjectId("..."),
    amount: 150.00,
    status: "completed",
    orderDate: ISODate("2024-01-15T10:30:00Z")
}
```

**Requêtes** :

```javascript
// Commande par numéro
db.orders.findOne({ orderNumber: "ORD-12345" })

// Commandes complétées
db.orders.find({ status: "completed" })

// Commandes d'un client spécifique
db.orders.find({ customerId: ObjectId("507f1f77bcf86cd799439011") })

// Commandes avec montant exact
db.orders.find({ amount: 150.00 })
```

---

## Bonnes Pratiques

### 1. Toujours Utiliser des Guillemets pour la Notation Pointée

```javascript
// ✅ Correct
db.users.find({ "address.city": "Paris" })

// ❌ Incorrect (ne fonctionnera pas)
db.users.find({ address.city: "Paris" })
```

### 2. Utiliser `findOne()` Quand Vous N'Avez Besoin Que d'Un Document

`findOne()` est plus performant car il s'arrête après avoir trouvé le premier document correspondant.

```javascript
// ✅ Bon choix pour un seul résultat
const user = db.users.findOne({ email: "john@example.com" })

// ⚠️ Moins optimal si vous voulez un seul résultat
const user = db.users.find({ email: "john@example.com" }).limit(1)
```

### 3. Spécifier les Champs Nécessaires avec les Projections

Récupérer uniquement les champs dont vous avez besoin améliore les performances :

```javascript
// Récupérer seulement le nom et l'email
db.users.find(
    { status: "active" },
    { name: 1, email: 1, _id: 0 }
)
```

### 4. Utiliser des Index pour les Champs Fréquemment Interrogés

Si vous recherchez souvent par un champ spécifique, créez un index dessus :

```javascript
// Créer un index sur le champ email
db.users.createIndex({ email: 1 })
```

Les index seront détaillés dans le chapitre 5.

### 5. Tester Vos Requêtes

Avant de les utiliser en production, testez vos requêtes dans le shell MongoDB :

```javascript
// Vérifier combien de documents correspondent
db.users.countDocuments({ status: "active" })

// Examiner quelques résultats
db.users.find({ status: "active" }).limit(3)
```

---

## Différences avec SQL

Pour ceux qui viennent du monde SQL, voici quelques équivalences :

| SQL | MongoDB |
|-----|---------|
| `SELECT * FROM users` | `db.users.find()` |
| `SELECT * FROM users WHERE age = 25` | `db.users.find({ age: 25 })` |
| `SELECT name, email FROM users` | `db.users.find({}, { name: 1, email: 1 })` |
| `SELECT * FROM users WHERE age = 25 AND status = 'active'` | `db.users.find({ age: 25, status: "active" })` |
| `SELECT * FROM users LIMIT 10` | `db.users.find().limit(10)` |
| `SELECT COUNT(*) FROM users` | `db.users.countDocuments()` |

---

## Points Clés à Retenir

✅ **`find()`** retourne un curseur vers plusieurs documents, **`findOne()`** retourne un seul document ou null

✅ Les requêtes MongoDB utilisent des documents JSON pour spécifier les critères

✅ Plusieurs critères dans un même document sont combinés avec un **ET logique** (AND)

✅ La **notation pointée** (`"champ.souschamp"`) permet d'interroger des documents imbriqués

✅ Les recherches dans les tableaux vérifient automatiquement si la valeur existe dans le tableau

✅ Le champ **`_id`** est unique pour chaque document et est automatiquement indexé

✅ Les méthodes peuvent être **chaînées** pour construire des requêtes complexes

✅ Les **projections** permettent de sélectionner uniquement les champs nécessaires

---

## Résumé

Dans ce chapitre, vous avez appris :

- La structure de base des requêtes MongoDB avec `find()` et `findOne()`
- Comment rechercher des documents par égalité simple ou multiple critères
- Comment interroger des champs imbriqués avec la notation pointée
- Comment rechercher des valeurs dans des tableaux
- Les méthodes utiles pour formater et limiter les résultats
- Les bonnes pratiques pour écrire des requêtes efficaces

Dans les prochains chapitres, nous approfondirons les **opérateurs de comparaison**, les **opérateurs logiques**, et d'autres techniques avancées de requêtage qui vous permettront de créer des filtres beaucoup plus puissants et sophistiqués.

---


⏭️ [Opérateurs de comparaison ($eq, $ne, $gt, $gte, $lt, $lte, $in, $nin)](/03-requetes-et-filtres/02-operateurs-comparaison.md)
