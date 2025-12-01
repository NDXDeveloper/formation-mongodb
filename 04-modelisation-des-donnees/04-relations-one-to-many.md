🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.4 Relations One-to-Many (Un-à-Plusieurs)

## Introduction

Les relations **one-to-many** (un-à-plusieurs) sont les plus courantes dans la modélisation de données. Elles existent lorsqu'un document d'une entité peut être associé à **plusieurs** documents d'une autre entité, mais chaque document de la seconde entité n'est lié qu'à **un seul** document de la première.

Dans MongoDB, le choix de la stratégie de modélisation pour ces relations dépend principalement du **nombre d'éléments** dans la partie "many" et des **patterns d'accès** de votre application.

---

## Comprendre les relations One-to-Many

### Définition

Une relation **one-to-many** signifie qu'une entité A peut être liée à **plusieurs** entités B, mais chaque entité B n'est liée qu'à **une seule** entité A.

**Exemples concrets :**

- **Auteur ↔ Articles** : Un auteur écrit plusieurs articles, mais chaque article a un seul auteur principal
- **Catégorie ↔ Produits** : Une catégorie contient plusieurs produits, mais chaque produit appartient à une catégorie
- **Client ↔ Commandes** : Un client passe plusieurs commandes, mais chaque commande appartient à un seul client
- **Département ↔ Employés** : Un département a plusieurs employés, mais chaque employé est dans un seul département
- **Album ↔ Photos** : Un album contient plusieurs photos, mais chaque photo appartient à un album
- **Article de blog ↔ Commentaires** : Un article reçoit plusieurs commentaires

### Les trois catégories de cardinalité

MongoDB distingue trois sous-types de relations one-to-many selon le nombre d'éléments :

1. **One-to-Few** : Le côté "many" contient **peu d'éléments** (généralement < 10-20)
2. **One-to-Many** : Le côté "many" contient un **nombre modéré** d'éléments (dizaines à centaines)
3. **One-to-Squillions** : Le côté "many" contient un **très grand nombre** d'éléments (milliers ou plus)

Cette distinction est **cruciale** pour choisir la bonne approche de modélisation.

---

## Stratégies de modélisation

### 1. Embedding : Documents imbriqués (One-to-Few)

#### Principe

Stocker les éléments de la partie "many" **directement dans le document parent** sous forme de tableau.

#### Quand utiliser l'embedding ?

✅ **Utilisez l'embedding pour "One-to-Few" quand :**

- Le nombre d'éléments est **limité et prévisible** (< 100)
- Les éléments sont **toujours consultés avec** le parent
- Les éléments **n'ont pas de sens** en dehors du parent
- Vous voulez des **performances optimales** en lecture
- Le total ne dépassera **jamais 16 Mo**

#### Exemple 1 : Article de blog avec commentaires (limités)

**Scénario :** Un blog où chaque article a quelques commentaires (< 50).

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "titre": "Introduction à MongoDB",
  "auteur": "Jean Martin",
  "contenu": "MongoDB est une base de données NoSQL orientée document...",
  "slug": "introduction-mongodb",
  "datePublication": ISODate("2024-01-15"),
  "tags": ["mongodb", "nosql", "database"],
  "commentaires": [
    {
      "id": 1,
      "auteur": "Sophie Dupont",
      "email": "sophie@example.com",
      "texte": "Excellent article, très clair !",
      "date": ISODate("2024-01-16T10:30:00Z"),
      "likes": 5
    },
    {
      "id": 2,
      "auteur": "Pierre Martin",
      "email": "pierre@example.com",
      "texte": "Merci pour ces explications détaillées.",
      "date": ISODate("2024-01-16T14:20:00Z"),
      "likes": 3
    },
    {
      "id": 3,
      "auteur": "Marie Leclerc",
      "email": "marie@example.com",
      "texte": "J'ai une question sur les index...",
      "date": ISODate("2024-01-17T09:15:00Z"),
      "likes": 1
    }
  ],
  "statistiques": {
    "vues": 1523,
    "likes": 89,
    "partages": 23
  }
}
```

**Avantages :**

- ✅ **Une seule requête** pour afficher l'article avec tous ses commentaires
- ✅ **Performance optimale** : données stockées ensemble
- ✅ **Atomicité** : ajout/modification de commentaire atomique avec l'article
- ✅ **Simplicité** : pas de gestion de jointures

**Requêtes :**

```javascript
// Lire l'article avec tous ses commentaires
db.articles.findOne({ slug: "introduction-mongodb" })

// Ajouter un nouveau commentaire
db.articles.updateOne(
  { _id: ObjectId("507f1f77bcf86cd799439011") },
  {
    $push: {
      commentaires: {
        id: 4,
        auteur: "Thomas Durand",
        email: "thomas@example.com",
        texte: "Super tutoriel !",
        date: new Date(),
        likes: 0
      }
    }
  }
)

// Modifier un commentaire spécifique
db.articles.updateOne(
  {
    _id: ObjectId("507f1f77bcf86cd799439011"),
    "commentaires.id": 2
  },
  {
    $set: { "commentaires.$.texte": "Texte modifié" }
  }
)

// Supprimer un commentaire
db.articles.updateOne(
  { _id: ObjectId("507f1f77bcf86cd799439011") },
  {
    $pull: { commentaires: { id: 2 } }
  }
)

// Compter les commentaires
db.articles.aggregate([
  { $match: { _id: ObjectId("507f1f77bcf86cd799439011") } },
  { $project: { nombreCommentaires: { $size: "$commentaires" } } }
])
```

#### Exemple 2 : Commande avec articles

**Scénario :** E-commerce où une commande contient plusieurs articles.

```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  "numeroCommande": "CMD-2024-001",
  "client": {
    "id": ObjectId("507f1f77bcf86cd799439012"),
    "nom": "Sophie Martin",
    "email": "sophie.martin@example.com"
  },
  "articles": [
    {
      "produitId": ObjectId("..."),
      "nom": "Livre MongoDB",
      "sku": "BOOK-001",
      "quantite": 2,
      "prixUnitaire": 29.99,
      "sousTotal": 59.98
    },
    {
      "produitId": ObjectId("..."),
      "nom": "Clavier mécanique",
      "sku": "KB-002",
      "quantite": 1,
      "prixUnitaire": 89.99,
      "sousTotal": 89.99
    },
    {
      "produitId": ObjectId("..."),
      "nom": "Souris ergonomique",
      "sku": "MS-003",
      "quantite": 1,
      "prixUnitaire": 39.99,
      "sousTotal": 39.99
    }
  ],
  "montantTotal": 189.96,
  "tva": 37.99,
  "totalTTC": 227.95,
  "statut": "en_preparation",
  "adresseLivraison": {
    "rue": "12 rue de la République",
    "ville": "Lyon",
    "codePostal": "69001",
    "pays": "France"
  },
  "dateCommande": ISODate("2024-01-15T10:30:00Z"),
  "dateLivraisonPrevue": ISODate("2024-01-18T00:00:00Z")
}
```

**Avantages :**

- ✅ **Snapshot historique** : même si les prix produits changent, la commande garde ses valeurs
- ✅ **Affichage rapide** : toutes les infos en une requête
- ✅ **Cohérence** : impossible d'avoir une commande incohérente

#### Exemple 3 : Utilisateur avec adresses

**Scénario :** Plateforme où un utilisateur a plusieurs adresses (domicile, bureau, livraison).

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439013"),
  "nom": "Dupont",
  "prenom": "Marie",
  "email": "marie.dupont@example.com",
  "telephone": "+33 6 12 34 56 78",
  "adresses": [
    {
      "id": 1,
      "type": "domicile",
      "principale": true,
      "rue": "15 rue de la Paix",
      "complementAdresse": "Appartement 3B",
      "ville": "Paris",
      "codePostal": "75002",
      "pays": "France",
      "instructions": "Code porte : 1234A"
    },
    {
      "id": 2,
      "type": "bureau",
      "principale": false,
      "rue": "45 avenue des Champs",
      "complementAdresse": "Bureau 501",
      "ville": "Paris",
      "codePostal": "75008",
      "pays": "France",
      "instructions": "Réception au 5ème étage"
    },
    {
      "id": 3,
      "type": "livraison",
      "principale": false,
      "rue": "10 rue du Commerce",
      "ville": "Lyon",
      "codePostal": "69001",
      "pays": "France",
      "instructions": "Livraison en semaine uniquement"
    }
  ],
  "dateInscription": ISODate("2023-06-10"),
  "statut": "actif"
}
```

---

### 2. Références du côté "Many" : Child-Referencing

#### Principe

Stocker dans chaque document du côté "many" une **référence vers le parent** (côté "one").

#### Quand utiliser Child-Referencing ?

✅ **Utilisez Child-Referencing pour "One-to-Many" et "One-to-Squillions" quand :**

- Le nombre d'éléments est **important** (centaines à millions)
- Les éléments sont souvent **consultés indépendamment** du parent
- Vous voulez **paginer** les éléments
- Les éléments peuvent **grandir indéfiniment**
- Vous avez besoin de **rechercher** dans les éléments

#### Exemple 1 : Catégorie et produits

**Scénario :** E-commerce avec des catégories contenant de nombreux produits.

**Collection "categories" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439014"),
  "nom": "Électronique",
  "slug": "electronique",
  "description": "Appareils électroniques et accessoires",
  "icone": "electronics.svg",
  "ordre": 1,
  "statistiques": {
    "nombreProduits": 1523  // Dénormalisé pour performance
  }
}
```

**Collection "produits" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d1"),
  "nom": "Smartphone XYZ Pro",
  "categorieId": ObjectId("507f1f77bcf86cd799439014"),  // ← Référence vers catégorie
  "categoriePath": "Électronique > Smartphones",  // Dénormalisé
  "prix": 899.99,
  "marque": "TechBrand",
  "description": "Smartphone haute performance...",
  "stock": 45,
  "dateAjout": ISODate("2024-01-10")
}
```

```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2"),
  "nom": "Tablette ABC 10",
  "categorieId": ObjectId("507f1f77bcf86cd799439014"),  // ← Même catégorie
  "categoriePath": "Électronique > Tablettes",
  "prix": 449.99,
  "marque": "TechBrand",
  "description": "Tablette 10 pouces...",
  "stock": 28,
  "dateAjout": ISODate("2024-01-12")
}
```

**Avantages :**

- ✅ **Pas de limite** sur le nombre de produits par catégorie
- ✅ **Pagination facile** : récupérer les produits par lots
- ✅ **Recherche efficace** : chercher un produit spécifique
- ✅ **Index performants** : indexer `categorieId` pour rapidité

**Requêtes :**

```javascript
// Récupérer tous les produits d'une catégorie (avec pagination)
db.produits.find({
  categorieId: ObjectId("507f1f77bcf86cd799439014")
})
  .sort({ dateAjout: -1 })
  .limit(20)
  .skip(0)

// Compter les produits dans une catégorie
db.produits.countDocuments({
  categorieId: ObjectId("507f1f77bcf86cd799439014")
})

// Rechercher un produit spécifique dans une catégorie
db.produits.findOne({
  categorieId: ObjectId("507f1f77bcf86cd799439014"),
  nom: /smartphone/i
})

// Récupérer catégorie + premiers produits avec $lookup
db.categories.aggregate([
  { $match: { slug: "electronique" } },
  {
    $lookup: {
      from: "produits",
      localField: "_id",
      foreignField: "categorieId",
      as: "produits"
    }
  },
  { $project: {
      nom: 1,
      description: 1,
      produits: { $slice: ["$produits", 10] }  // Limiter à 10 produits
    }
  }
])
```

**Index recommandé :**
```javascript
db.produits.createIndex({ categorieId: 1 })
```

#### Exemple 2 : Auteur et articles

**Collection "auteurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439015"),
  "nom": "Jean Martin",
  "email": "jean.martin@blog.com",
  "bio": "Développeur passionné par MongoDB...",
  "photo": "https://exemple.com/photos/jean.jpg",
  "dateInscription": ISODate("2022-01-10"),
  "statistiques": {
    "nombreArticles": 47,
    "nombreAbonnes": 1250,
    "totalVues": 125430
  }
}
```

**Collection "articles" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d3"),
  "titre": "Modélisation MongoDB",
  "auteurId": ObjectId("507f1f77bcf86cd799439015"),  // ← Référence
  "auteurNom": "Jean Martin",  // Dénormalisé pour affichage rapide
  "slug": "modelisation-mongodb",
  "contenu": "Dans cet article...",
  "datePublication": ISODate("2024-01-15"),
  "statut": "publie",
  "tags": ["mongodb", "nosql"],
  "statistiques": {
    "vues": 523,
    "likes": 42
  }
}
```

#### Exemple 3 : Département et employés

**Collection "departements" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439016"),
  "nom": "Développement",
  "code": "DEV",
  "responsable": "Sophie Martin",
  "budget": 500000,
  "statistiques": {
    "nombreEmployes": 25
  }
}
```

**Collection "employes" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d4"),
  "matricule": "EMP-001",
  "nom": "Dupont",
  "prenom": "Pierre",
  "email": "pierre.dupont@entreprise.com",
  "departementId": ObjectId("507f1f77bcf86cd799439016"),  // ← Référence
  "departementNom": "Développement",  // Dénormalisé
  "poste": "Développeur Senior",
  "salaire": 55000,
  "dateEmbauche": ISODate("2020-03-15"),
  "statut": "actif"
}
```

---

### 3. Références du côté "One" : Parent-Referencing

#### Principe

Stocker dans le document parent un **tableau de références** vers les documents enfants.

#### Quand utiliser Parent-Referencing ?

✅ **Utilisez Parent-Referencing quand :**

- Le nombre d'éléments est **modéré** (dizaines à centaines max)
- Vous avez souvent besoin de **lister tous les IDs** des enfants
- Vous voulez **éviter une requête** pour obtenir la liste des IDs
- Le nombre d'enfants ne dépassera **jamais 16 Mo**

⚠️ **Attention :** Moins courant que Child-Referencing, utilisez avec prudence.

#### Exemple 1 : Projet et tâches

**Collection "projets" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439017"),
  "nom": "Refonte site web",
  "description": "Modernisation complète du site",
  "responsable": "Sophie Martin",
  "tacheIds": [  // ← Tableau de références
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d6"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d7"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d8")
  ],
  "dateDebut": ISODate("2024-01-01"),
  "dateFinPrevue": ISODate("2024-06-30"),
  "budget": 50000,
  "statut": "en_cours"
}
```

**Collection "taches" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
  "nom": "Maquettes",
  "description": "Création des maquettes UI/UX",
  "assignee": "Pierre Dupont",
  "statut": "terminee",
  "priorite": "haute",
  "dateEcheance": ISODate("2024-01-31"),
  "heuresEstimees": 40,
  "heuresReelles": 38
}
```

**Requêtes :**

```javascript
// Récupérer le projet avec toutes ses tâches
const projet = db.projets.findOne({ nom: "Refonte site web" })
const taches = db.taches.find({
  _id: { $in: projet.tacheIds }
})

// Ajouter une nouvelle tâche
const nouvelleTacheId = db.taches.insertOne({
  nom: "Tests utilisateurs",
  description: "...",
  assignee: "Marie Leclerc",
  statut: "a_faire"
}).insertedId

db.projets.updateOne(
  { _id: ObjectId("507f1f77bcf86cd799439017") },
  { $push: { tacheIds: nouvelleTacheId } }
)

// Supprimer une tâche
db.projets.updateOne(
  { _id: ObjectId("507f1f77bcf86cd799439017") },
  { $pull: { tacheIds: ObjectId("60a1f1b2c3d4e5f6a7b8c9d5") } }
)
db.taches.deleteOne({ _id: ObjectId("60a1f1b2c3d4e5f6a7b8c9d5") })
```

#### Exemple 2 : Album et photos

**Collection "albums" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439018"),
  "titre": "Vacances été 2024",
  "description": "Photos de nos vacances en Bretagne",
  "proprietaire": "Sophie Martin",
  "photoIds": [  // ← Liste des photos
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d9"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9da"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9db")
    // ... max quelques centaines
  ],
  "photoCouverture": ObjectId("60a1f1b2c3d4e5f6a7b8c9d9"),
  "dateCreation": ISODate("2024-07-01"),
  "visibilite": "prive"
}
```

**Collection "photos" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d9"),
  "titre": "Coucher de soleil sur la plage",
  "url": "https://storage.exemple.com/photos/IMG_001.jpg",
  "miniature": "https://storage.exemple.com/thumbs/IMG_001_thumb.jpg",
  "dateCapture": ISODate("2024-07-15T20:30:00Z"),
  "localisation": {
    "latitude": 48.573405,
    "longitude": -3.982708,
    "ville": "Quiberon"
  },
  "camera": "iPhone 14 Pro",
  "taille": 2457600,
  "dimensions": {
    "largeur": 4032,
    "hauteur": 3024
  }
}
```

---

### 4. Références bidirectionnelles : Two-Way Referencing

#### Principe

Stocker des références **dans les deux sens** : le parent connaît ses enfants, et les enfants connaissent leur parent.

#### Quand utiliser Two-Way Referencing ?

✅ **Utilisez des références bidirectionnelles quand :**

- Vous avez besoin de **chercher dans les deux sens** fréquemment
- Le nombre d'éléments est **modéré**
- Vous voulez **optimiser les requêtes** dans les deux directions

⚠️ **Inconvénient majeur :** Maintenir la **cohérence** des deux références.

#### Exemple : Étudiants et cours (many-to-many traité comme two one-to-many)

**Collection "etudiants" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439019"),
  "nom": "Dupont",
  "prenom": "Sophie",
  "email": "sophie.dupont@universite.fr",
  "coursIds": [  // ← Références vers cours suivis
    ObjectId("60a1f1b2c3d4e5f6a7b8c9dc"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9dd"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9de")
  ],
  "anneeScolaire": 2024,
  "niveau": "Master 1"
}
```

**Collection "cours" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9dc"),
  "code": "INFO-401",
  "titre": "Bases de données avancées",
  "professeur": "Dr. Martin",
  "etudiantIds": [  // ← Références vers étudiants inscrits
    ObjectId("507f1f77bcf86cd799439019"),
    ObjectId("507f1f77bcf86cd799439020"),
    ObjectId("507f1f77bcf86cd799439021")
    // ... autres étudiants
  ],
  "credits": 6,
  "semestre": "Automne 2024",
  "capaciteMax": 50
}
```

**Avantages :**

- ✅ Trouver rapidement **les cours d'un étudiant**
- ✅ Trouver rapidement **les étudiants d'un cours**
- ✅ Pas de requête supplémentaire dans l'autre collection

**Inconvénients :**

- ⚠️ Maintenir la cohérence : si on ajoute un étudiant à un cours, il faut mettre à jour les deux documents
- ⚠️ Risque d'incohérence si une mise à jour échoue

**Requêtes avec transactions (pour cohérence) :**

```javascript
// Inscrire un étudiant à un cours
const session = db.getMongo().startSession()
session.startTransaction()

try {
  // Ajouter le cours à l'étudiant
  db.etudiants.updateOne(
    { _id: ObjectId("507f1f77bcf86cd799439019") },
    { $addToSet: { coursIds: ObjectId("60a1f1b2c3d4e5f6a7b8c9dc") } },
    { session }
  )

  // Ajouter l'étudiant au cours
  db.cours.updateOne(
    { _id: ObjectId("60a1f1b2c3d4e5f6a7b8c9dc") },
    { $addToSet: { etudiantIds: ObjectId("507f1f77bcf86cd799439019") } },
    { session }
  )

  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  throw error
} finally {
  session.endSession()
}
```

---

## Comparaison des approches

| Approche | Cardinalité idéale | Avantages | Inconvénients |
|----------|-------------------|-----------|---------------|
| **Embedding** | One-to-Few (< 100) | ✅ 1 requête<br>✅ Atomicité<br>✅ Performance | ⚠️ Limite 16 Mo<br>⚠️ Croissance limitée<br>⚠️ Duplication |
| **Child-Referencing** | One-to-Many, One-to-Squillions | ✅ Illimité<br>✅ Pagination<br>✅ Recherche facile | ⚠️ 2+ requêtes<br>⚠️ Index nécessaires |
| **Parent-Referencing** | One-to-Many modéré | ✅ Liste IDs immédiate<br>✅ Requête unique pour IDs | ⚠️ Limite du tableau<br>⚠️ Cohérence à gérer |
| **Two-Way Referencing** | Many-to-Many modéré | ✅ Recherche bidirectionnelle | ⚠️ Cohérence complexe<br>⚠️ Double stockage |

---

## Patterns courants

### Pattern 1 : Subset Pattern

Imbriquer les **éléments les plus récents/importants**, stocker le reste en référence.

**Exemple : Article avec aperçu des commentaires**

```json
{
  "_id": ObjectId("..."),
  "titre": "MongoDB Performance",
  "contenu": "...",
  "commentairesRecents": [  // ← 3 derniers commentaires imbriqués
    {
      "auteur": "Sophie",
      "texte": "Excellent article !",
      "date": ISODate("2024-01-18")
    },
    {
      "auteur": "Pierre",
      "texte": "Très instructif.",
      "date": ISODate("2024-01-17")
    }
  ],
  "nombreCommentairesTotal": 247,  // Total dans collection séparée
  "datePublication": ISODate("2024-01-15")
}
```

**Collection "commentaires" (tous les commentaires) :**
```json
{
  "_id": ObjectId("..."),
  "articleId": ObjectId("..."),
  "auteur": "Sophie",
  "texte": "Excellent article !",
  "date": ISODate("2024-01-18")
}
```

### Pattern 2 : Extended Reference Pattern

Dénormaliser quelques champs du parent dans les enfants.

**Exemple : Produit avec informations catégorie**

```json
{
  "_id": ObjectId("..."),
  "nom": "Smartphone XYZ",
  "categorieId": ObjectId("..."),  // ← Référence
  "categorieNom": "Électronique",  // ← Dénormalisé
  "categoriePath": "Électronique > Smartphones",  // ← Dénormalisé
  "prix": 899.99
}
```

**Avantages :**
- ✅ Afficher la liste de produits sans jointure
- ✅ Breadcrumb navigation immédiat
- ⚠️ Si catégorie renommée, faut mettre à jour tous les produits

### Pattern 3 : Bucketing Pattern

Regrouper les éléments en "seaux" pour limiter la taille.

**Exemple : Mesures IoT par heure**

Au lieu de :
```json
// ❌ Un document par mesure (des millions !)
{
  "capteurId": "SENSOR-001",
  "temperature": 22.5,
  "timestamp": ISODate("2024-01-15T10:00:00Z")
}
```

Regrouper par heure :
```json
{
  "_id": ObjectId("..."),
  "capteurId": "SENSOR-001",
  "date": ISODate("2024-01-15T10:00:00Z"),
  "mesures": [
    { "timestamp": ISODate("2024-01-15T10:00:00Z"), "temperature": 22.5 },
    { "timestamp": ISODate("2024-01-15T10:01:00Z"), "temperature": 22.6 },
    { "timestamp": ISODate("2024-01-15T10:02:00Z"), "temperature": 22.4 }
    // ... 60 mesures par heure
  ],
  "nombreMesures": 60,
  "temperatureMoyenne": 22.5,
  "temperatureMin": 22.1,
  "temperatureMax": 22.9
}
```

---

## Arbre de décision

```
Combien d'éléments dans la partie "many" ?
│
├─ < 100 éléments (One-to-Few)
│  │
│  ├─ Toujours consultés ensemble ? → EMBEDDING
│  └─ Consultés indépendamment ? → CHILD-REFERENCING
│
├─ 100 à 1000 éléments (One-to-Many)
│  │
│  ├─ Besoin de pagination ? → CHILD-REFERENCING
│  ├─ Besoin de liste complète des IDs ? → PARENT-REFERENCING
│  └─ Mix : éléments récents + total ? → SUBSET PATTERN
│
└─ > 1000 éléments (One-to-Squillions)
   │
   └─ TOUJOURS CHILD-REFERENCING
      ├─ Avec dénormalisation si nécessaire
      └─ Ou BUCKETING si données temporelles
```

---

## Exemples complets par domaine

### E-commerce : Catégories et Produits

**Choix : Child-Referencing** (des milliers de produits par catégorie)

```javascript
// Collection "categories"
{
  "_id": ObjectId("cat_electronique"),
  "nom": "Électronique",
  "slug": "electronique",
  "niveauHierarchie": 1
}

// Collection "produits"
{
  "_id": ObjectId("prod_smartphone_xyz"),
  "nom": "Smartphone XYZ Pro",
  "categorieId": ObjectId("cat_electronique"),
  "prix": 899.99,
  "stock": 45
}

// Index pour performance
db.produits.createIndex({ categorieId: 1, prix: 1 })
```

### Blog : Articles et Commentaires

**Choix : Hybride (Subset Pattern)**

```javascript
// Collection "articles"
{
  "_id": ObjectId("art_001"),
  "titre": "MongoDB Guide",
  "contenu": "...",
  "commentairesRecents": [/* 5 derniers */],
  "nombreCommentaires": 247
}

// Collection "commentaires" (tous)
{
  "_id": ObjectId("com_001"),
  "articleId": ObjectId("art_001"),
  "auteur": "Sophie",
  "texte": "Super article !"
}
```

### RH : Départements et Employés

**Choix : Child-Referencing**

```javascript
// Collection "departements"
{
  "_id": ObjectId("dep_dev"),
  "nom": "Développement",
  "responsable": "Sophie Martin"
}

// Collection "employes"
{
  "_id": ObjectId("emp_001"),
  "nom": "Dupont Pierre",
  "departementId": ObjectId("dep_dev"),
  "departementNom": "Développement"  // Dénormalisé
}

db.employes.createIndex({ departementId: 1 })
```

---

## Recommandations pratiques

### ✅ Bonnes pratiques

1. **Commencez par l'embedding** si le nombre est < 100
2. **Utilisez Child-Referencing** pour la plupart des cas > 100 éléments
3. **Créez des index** sur les champs de référence
4. **Dénormalisez intelligemment** les champs fréquents
5. **Utilisez le Subset Pattern** pour les "top N"
6. **Comptez et agrégez** pour éviter de charger tous les éléments
7. **Testez avec des volumes réalistes**

### ⚠️ Pièges à éviter

1. **Imbriquer des milliers d'éléments** (risque 16 Mo + performance)
2. **Ne pas créer d'index** sur les références
3. **Dupliquer trop de données** (difficile à maintenir)
4. **Oublier la pagination** pour les grandes collections
5. **Ne pas anticiper la croissance** (partir sur embedding alors que ça grandira)
6. **Utiliser Two-Way Referencing** sans transactions

### 📊 Métriques à surveiller

```javascript
// Surveiller la taille des tableaux imbriqués
db.articles.aggregate([
  {
    $project: {
      titre: 1,
      nombreCommentaires: { $size: "$commentaires" }
    }
  },
  { $sort: { nombreCommentaires: -1 } },
  { $limit: 10 }
])

// Trouver les documents proches de 16 Mo
db.articles.find().forEach(doc => {
  const size = Object.bsonsize(doc)
  if (size > 15 * 1024 * 1024) {
    print(`Document ${doc._id} : ${size} bytes`)
  }
})
```

---

## Conclusion

Les relations **one-to-many** sont omniprésentes dans la modélisation MongoDB. Le choix de la stratégie dépend principalement de :

1. **Cardinalité** : Combien d'éléments ? (few, many, squillions)
2. **Patterns d'accès** : Consultés ensemble ou séparément ?
3. **Croissance** : Nombre fixe ou croissance illimitée ?
4. **Performance** : Lectures ou écritures prioritaires ?

**Règles d'or :**

- 📊 **< 100 éléments** → Embedding
- 📊 **100-1000 éléments** → Child-Referencing (ou Subset Pattern)
- 📊 **> 1000 éléments** → Child-Referencing obligatoire

N'oubliez pas : vous pouvez toujours **combiner les approches** avec des patterns hybrides pour obtenir le meilleur compromis performance/flexibilité.

---

**Points clés à retenir :**

- ✅ Privilégiez l'embedding pour les relations one-to-few
- ✅ Utilisez child-referencing pour les relations one-to-many/squillions
- ✅ Créez toujours des index sur les champs de référence
- ✅ Le Subset Pattern est excellent pour afficher les "top N"
- ✅ Dénormalisez les champs fréquemment consultés
- ✅ Attention à la limite de 16 Mo avec l'embedding
- ✅ Testez vos choix avec des volumes réalistes

---


⏭️ [Relations Many-to-Many](/04-modelisation-des-donnees/05-relations-many-to-many.md)
