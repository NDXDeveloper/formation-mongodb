🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.5 Opérations CRUD de Base

## Introduction

Les **opérations CRUD** constituent le cœur de toute interaction avec une base de données. Que vous construisiez un blog, une boutique en ligne ou une application de gestion, vous utiliserez constamment ces quatre opérations fondamentales.

> **💡 CRUD** est un acronyme pour : **C**reate (Créer), **R**ead (Lire), **U**pdate (Mettre à jour), **D**elete (Supprimer)

Dans ce chapitre, nous allons découvrir comment manipuler vos données dans MongoDB avec ces opérations essentielles.

---

## Qu'est-ce que le CRUD ?

### Définition

**CRUD** désigne les quatre opérations de base que vous pouvez effectuer sur des données :

1. **Create (Créer)** : Ajouter de nouvelles données
2. **Read (Lire)** : Consulter et rechercher des données
3. **Update (Mettre à jour)** : Modifier des données existantes
4. **Delete (Supprimer)** : Retirer des données

### Analogie avec le Monde Réel

Imaginez un carnet de contacts :

- **Create** : Ajouter un nouveau contact
- **Read** : Consulter les informations d'un contact
- **Update** : Modifier le numéro de téléphone d'un contact
- **Delete** : Supprimer un contact qui n'est plus actif

Ces quatre actions couvrent **tout ce que vous faites** avec vos données au quotidien.

---

## Pourquoi le CRUD est Important ?

### Applications Universelles

**Toute application utilise le CRUD :**

**📱 Réseau Social**
- Create : Publier un post
- Read : Voir son fil d'actualité
- Update : Modifier sa bio
- Delete : Supprimer une photo

**🛒 E-commerce**
- Create : Ajouter un produit au panier
- Read : Parcourir le catalogue
- Update : Changer la quantité d'un article
- Delete : Retirer un article du panier

**📧 Application de Messagerie**
- Create : Envoyer un email
- Read : Lire ses messages
- Update : Marquer comme lu
- Delete : Supprimer un email

**📝 Application de Notes**
- Create : Créer une nouvelle note
- Read : Lire ses notes
- Update : Éditer une note
- Delete : Supprimer une note

### Fondations de Toute Base de Données

Les opérations CRUD sont **universelles** :
- ✅ Présentes dans toutes les bases de données (SQL, NoSQL)
- ✅ Concept indépendant de la technologie
- ✅ Une fois comprises, transférables partout

---

## Vue d'Ensemble des Opérations CRUD dans MongoDB

### Les Méthodes MongoDB

MongoDB fournit des méthodes spécifiques pour chaque opération CRUD :

| Opération | Méthode MongoDB | Description |
|-----------|-----------------|-------------|
| **Create** | `insertOne()` | Insérer un document |
| | `insertMany()` | Insérer plusieurs documents |
| **Read** | `find()` | Rechercher plusieurs documents |
| | `findOne()` | Rechercher un document |
| **Update** | `updateOne()` | Modifier un document |
| | `updateMany()` | Modifier plusieurs documents |
| | `replaceOne()` | Remplacer un document complet |
| **Delete** | `deleteOne()` | Supprimer un document |
| | `deleteMany()` | Supprimer plusieurs documents |

---

## 1. Create : Créer des Données

### Principe

**Ajouter de nouvelles données dans la base.**

### Méthodes

**insertOne()** : Insérer un seul document
```javascript
db.utilisateurs.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})
```

**insertMany()** : Insérer plusieurs documents en une fois
```javascript
db.utilisateurs.insertMany([
  { nom: "Martin", email: "martin@example.com" },
  { nom: "Bernard", email: "bernard@example.com" }
])
```

### Points Clés

- 📝 Ajoute des **nouveaux** documents
- 🔑 MongoDB génère automatiquement un `_id` unique
- ⚡ `insertMany()` est plus performant pour plusieurs documents
- 🎯 Vous définissez la structure des documents

### Cas d'Usage Courants

- Inscription d'un nouvel utilisateur
- Ajout d'un produit au catalogue
- Création d'un nouvel article de blog
- Enregistrement d'une nouvelle commande

---

## 2. Read : Lire des Données

### Principe

**Rechercher et consulter les données existantes.**

### Méthodes

**findOne()** : Trouver un seul document
```javascript
db.utilisateurs.findOne({ email: "dupont@example.com" })
```

**find()** : Trouver plusieurs documents
```javascript
db.utilisateurs.find({ ville: "Paris" })
```

### Points Clés

- 🔍 L'opération la **plus fréquente** (90% des requêtes)
- 📊 Supporte des filtres complexes
- 🎯 Retourne uniquement les champs demandés (projections)
- ⚡ Peut être optimisée avec des index

### Options Principales

- **Filtres** : Critères de recherche
- **Projections** : Sélectionner les champs à retourner
- **Tri** : Ordonner les résultats
- **Limite** : Limiter le nombre de résultats
- **Skip** : Sauter des résultats (pagination)

### Cas d'Usage Courants

- Afficher le profil d'un utilisateur
- Lister les produits d'une catégorie
- Rechercher des articles par mot-clé
- Consulter l'historique des commandes

---

## 3. Update : Mettre à Jour des Données

### Principe

**Modifier des données existantes.**

### Méthodes

**updateOne()** : Modifier un seul document
```javascript
db.utilisateurs.updateOne(
  { email: "dupont@example.com" },
  { $set: { ville: "Lyon" } }
)
```

**updateMany()** : Modifier plusieurs documents
```javascript
db.produits.updateMany(
  { categorie: "Électronique" },
  { $set: { enPromotion: true } }
)
```

**replaceOne()** : Remplacer un document complet
```javascript
db.utilisateurs.replaceOne(
  { _id: ObjectId("...") },
  { nom: "Nouveau", email: "nouveau@example.com" }
)
```

### Points Clés

- ✏️ Modifie des documents **existants**
- 🔧 Utilise des **opérateurs** spéciaux ($set, $inc, $push, etc.)
- 🎯 `updateOne()` modifie le premier document trouvé
- 📊 `updateMany()` modifie tous les documents correspondants

### Opérateurs Courants

- `$set` : Définir une valeur
- `$inc` : Incrémenter/décrémenter
- `$push` : Ajouter à un tableau
- `$pull` : Retirer d'un tableau
- `$unset` : Supprimer un champ

### Cas d'Usage Courants

- Mettre à jour le profil utilisateur
- Modifier le prix d'un produit
- Incrémenter un compteur de vues
- Ajouter un commentaire à un article

---

## 4. Delete : Supprimer des Données

### Principe

**Retirer définitivement des données.**

### Méthodes

**deleteOne()** : Supprimer un seul document
```javascript
db.utilisateurs.deleteOne({ email: "dupont@example.com" })
```

**deleteMany()** : Supprimer plusieurs documents
```javascript
db.logs.deleteMany({ date: { $lt: new Date("2024-01-01") } })
```

### Points Clés

- 🗑️ Suppression **définitive** (irréversible)
- ⚠️ Opération la plus **dangereuse**
- 🎯 `deleteOne()` supprime le premier document trouvé
- 📊 `deleteMany()` supprime tous les documents correspondants

### Précautions

- ✅ Toujours vérifier avec `find()` avant de supprimer
- ✅ Utiliser des filtres spécifiques
- ✅ Préférer le "soft delete" pour les données importantes
- ⚠️ JAMAIS `deleteMany({})` sans réfléchir (supprime TOUT)

### Cas d'Usage Courants

- Supprimer un compte utilisateur
- Nettoyer les anciens logs
- Retirer un produit obsolète
- Supprimer un brouillon d'article

---

## Comparaison CRUD : SQL vs MongoDB

### Correspondance des Opérations

| Opération | SQL | MongoDB |
|-----------|-----|---------|
| **Create** | `INSERT INTO` | `insertOne()` / `insertMany()` |
| **Read** | `SELECT` | `find()` / `findOne()` |
| **Update** | `UPDATE` | `updateOne()` / `updateMany()` |
| **Delete** | `DELETE` | `deleteOne()` / `deleteMany()` |

### Exemple Comparatif

**Créer :**
```sql
-- SQL
INSERT INTO utilisateurs (nom, email)
VALUES ('Dupont', 'dupont@example.com');
```
```javascript
// MongoDB
db.utilisateurs.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})
```

**Lire :**
```sql
-- SQL
SELECT * FROM utilisateurs
WHERE ville = 'Paris';
```
```javascript
// MongoDB
db.utilisateurs.find({ ville: "Paris" })
```

**Modifier :**
```sql
-- SQL
UPDATE utilisateurs
SET ville = 'Lyon'
WHERE email = 'dupont@example.com';
```
```javascript
// MongoDB
db.utilisateurs.updateOne(
  { email: "dupont@example.com" },
  { $set: { ville: "Lyon" } }
)
```

**Supprimer :**
```sql
-- SQL
DELETE FROM utilisateurs
WHERE email = 'dupont@example.com';
```
```javascript
// MongoDB
db.utilisateurs.deleteOne({ email: "dupont@example.com" })
```

---

## Particularités de MongoDB

### 1. Pas de Schéma Strict

**SQL :** Structure rigide définie à l'avance
```sql
CREATE TABLE utilisateurs (
  id INT PRIMARY KEY,
  nom VARCHAR(50),
  email VARCHAR(100)
);
```

**MongoDB :** Structure flexible
```javascript
// Aucune définition préalable nécessaire
db.utilisateurs.insertOne({ nom: "Dupont" })
db.utilisateurs.insertOne({ nom: "Martin", age: 30, ville: "Paris" })
// Les deux documents peuvent avoir des champs différents !
```

### 2. Documents Imbriqués

**MongoDB supporte nativement les objets imbriqués :**
```javascript
db.utilisateurs.insertOne({
  nom: "Dupont",
  adresse: {
    rue: "123 Rue Example",
    ville: "Paris",
    codePostal: "75001"
  },
  competences: ["JavaScript", "Python", "MongoDB"]
})
```

**SQL nécessiterait plusieurs tables et jointures.**

### 3. Tableaux Natifs

**MongoDB gère directement les tableaux :**
```javascript
db.articles.insertOne({
  titre: "Mon Article",
  tags: ["MongoDB", "NoSQL", "Base de données"],
  commentaires: [
    { auteur: "Alice", texte: "Super !" },
    { auteur: "Bob", texte: "Merci" }
  ]
})
```

**SQL nécessiterait des tables séparées.**

### 4. Opérations Atomiques

**Chaque opération sur un document est atomique :**
```javascript
// Cette mise à jour est atomique (tout ou rien)
db.comptes.updateOne(
  { _id: 1 },
  {
    $inc: { solde: 100 },
    $push: { transactions: { montant: 100, date: new Date() } }
  }
)
```

---

## Cycle de Vie d'une Application

### Flux Typique

```
1. L'utilisateur s'inscrit
   → CREATE (insertOne)

2. L'utilisateur se connecte
   → READ (findOne)

3. L'utilisateur met à jour son profil
   → UPDATE (updateOne)

4. L'utilisateur consulte des articles
   → READ (find)

5. L'utilisateur ajoute un commentaire
   → UPDATE (updateOne avec $push)

6. L'utilisateur supprime son compte
   → DELETE (deleteOne)
```

### Exemple : Application de Blog

```javascript
// 1. CREATE : Créer un nouvel article
db.articles.insertOne({
  titre: "Introduction à MongoDB",
  contenu: "MongoDB est...",
  auteur: "Alice",
  datePublication: new Date(),
  vues: 0,
  likes: 0
})

// 2. READ : Lire tous les articles publiés
db.articles.find({ publie: true })

// 3. UPDATE : Incrémenter les vues
db.articles.updateOne(
  { _id: ObjectId("...") },
  { $inc: { vues: 1 } }
)

// 4. UPDATE : Ajouter un commentaire
db.articles.updateOne(
  { _id: ObjectId("...") },
  {
    $push: {
      commentaires: {
        auteur: "Bob",
        texte: "Super article !",
        date: new Date()
      }
    }
  }
)

// 5. DELETE : Supprimer un brouillon
db.articles.deleteOne({
  _id: ObjectId("..."),
  statut: "brouillon"
})
```

---

## Bonnes Pratiques Générales

### ✅ À Faire

1. **Utilisez des filtres spécifiques**
   ```javascript
   // ✅ Bon : Filtre précis
   db.users.findOne({ _id: ObjectId("...") })

   // ⚠️ Risqué : Filtre vague
   db.users.findOne({ nom: "Dupont" })  // Plusieurs Dupont ?
   ```

2. **Vérifiez toujours les résultats**
   ```javascript
   let resultat = db.users.updateOne(...)
   if (resultat.modifiedCount === 0) {
     print("Aucune modification effectuée")
   }
   ```

3. **Testez avec find() avant delete()**
   ```javascript
   // 1. Voir ce qui sera supprimé
   db.logs.find({ old: true })

   // 2. Si OK, supprimer
   db.logs.deleteMany({ old: true })
   ```

4. **Utilisez les bonnes méthodes**
   - `insertMany()` au lieu de multiples `insertOne()`
   - `findOne()` au lieu de `find().limit(1)`
   - `updateOne()` pour un seul document

### ❌ À Éviter

1. **Opérations sans filtre**
   ```javascript
   // ❌ DANGEREUX
   db.users.deleteMany({})  // Supprime TOUT !
   ```

2. **Ignorer les erreurs**
   ```javascript
   // ❌ Pas de vérification
   db.users.insertOne(data)

   // ✅ Avec gestion d'erreur
   try {
     db.users.insertOne(data)
   } catch (e) {
     print("Erreur : " + e)
   }
   ```

3. **Modifications non atomiques**
   ```javascript
   // ❌ Race condition possible
   let doc = db.collection.findOne({ _id: 1 })
   doc.compteur = doc.compteur + 1
   db.collection.updateOne({ _id: 1 }, { $set: { compteur: doc.compteur } })

   // ✅ Atomique
   db.collection.updateOne({ _id: 1 }, { $inc: { compteur: 1 } })
   ```

---

## Performance et Optimisation

### Index : La Clé de la Performance

**Les opérations CRUD bénéficient grandement des index :**

```javascript
// Sans index : Lent sur une grosse collection
db.utilisateurs.find({ email: "test@example.com" })

// Créer un index
db.utilisateurs.createIndex({ email: 1 })

// Avec index : Rapide !
db.utilisateurs.find({ email: "test@example.com" })
```

### Conseils de Performance

**CREATE :**
- ✅ Utilisez `insertMany()` pour plusieurs documents
- ✅ Désactivez temporairement les index pour insertions massives

**READ :**
- ✅ Créez des index sur les champs fréquemment recherchés
- ✅ Utilisez des projections pour limiter les données retournées
- ✅ Limitez les résultats avec `.limit()`

**UPDATE :**
- ✅ Utilisez des opérateurs atomiques ($inc, $push, etc.)
- ✅ Préférez `updateOne()` si vous modifiez un seul document

**DELETE :**
- ✅ Supprimez par lots pour de grandes quantités
- ✅ Considérez le "soft delete" (marquer comme supprimé)

---

## Sécurité et Validation

### Validation des Données

**MongoDB permet de définir des règles de validation :**

```javascript
db.createCollection("utilisateurs", {
  validator: {
    $jsonSchema: {
      required: ["nom", "email"],
      properties: {
        nom: { bsonType: "string" },
        email: { bsonType: "string" },
        age: { bsonType: "int", minimum: 0 }
      }
    }
  }
})
```

### Injection NoSQL

**Attention aux injections dans les filtres :**

```javascript
// ❌ Vulnérable si userInput vient d'un utilisateur
db.users.findOne({ username: userInput })

// ✅ Valider et nettoyer les entrées utilisateur
function rechercherUtilisateur(username) {
  if (typeof username !== 'string') {
    throw new Error("Username invalide")
  }
  return db.users.findOne({ username: username })
}
```

---

## Structure du Chapitre

Dans les sections suivantes, nous allons explorer en détail chaque opération CRUD :

### 📝 2.5.1 Create : insertOne() et insertMany()
- Insérer un document
- Insérer plusieurs documents
- Génération de l'_id
- Options d'insertion
- Gestion des erreurs

### 🔍 2.5.2 Read : find() et findOne()
- Rechercher des documents
- Filtres et critères
- Projections
- Tri et limitation
- Pagination

### ✏️ 2.5.3 Update : updateOne() et updateMany()
- Modifier des documents
- Opérateurs de mise à jour
- Opérateurs pour tableaux
- Option upsert
- Modifications atomiques

### 🗑️ 2.5.4 Delete : deleteOne() et deleteMany()
- Supprimer des documents
- Précautions importantes
- Soft delete
- Suppression en cascade

### 🔄 2.5.5 Opérations Avancées : replaceOne()
- Remplacer un document complet
- Différences avec update
- Cas d'usage

---

## Points Clés à Retenir

### ✅ Essentiel

1. **CRUD** = Create, Read, Update, Delete
2. **4 opérations fondamentales** pour toute base de données
3. **MongoDB fournit des méthodes spécifiques** pour chaque opération
4. **Read est l'opération la plus fréquente** (90% des requêtes)
5. **Delete est irréversible** - prudence requise !

### 🎯 Méthodes Principales

| Opération | Méthode(s) |
|-----------|-----------|
| Create | `insertOne()`, `insertMany()` |
| Read | `find()`, `findOne()` |
| Update | `updateOne()`, `updateMany()`, `replaceOne()` |
| Delete | `deleteOne()`, `deleteMany()` |

### 💡 Principes Importants

- ✅ Utilisez des filtres spécifiques
- ✅ Vérifiez toujours les résultats
- ✅ Pensez aux index pour la performance
- ✅ Gérez les erreurs
- ⚠️ Soyez prudent avec les suppressions

---

## Prochaines Étapes

Maintenant que vous comprenez le concept global du CRUD, plongeons dans les détails de chaque opération :

➡️ **2.5.1 insertOne() et insertMany()** : Apprenez à créer des documents

Préparez-vous à manipuler vos données comme un pro ! 🚀

---

## Ressources Complémentaires

### Documentation Officielle

- [CRUD Operations - MongoDB Manual](https://docs.mongodb.com/manual/crud/)
- [Insert Documents](https://docs.mongodb.com/manual/tutorial/insert-documents/)
- [Query Documents](https://docs.mongodb.com/manual/tutorial/query-documents/)
- [Update Documents](https://docs.mongodb.com/manual/tutorial/update-documents/)
- [Delete Documents](https://docs.mongodb.com/manual/tutorial/remove-documents/)

### Concepts Liés

- Indexes pour optimiser les requêtes
- Validation de schéma
- Transactions pour garantir la cohérence
- Aggregation Framework pour des requêtes complexes

---


⏭️ [insertOne() et insertMany()](/02-fondamentaux-de-mongodb/05.1-insert.md)
