🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.5 Relations Many-to-Many (Plusieurs-à-Plusieurs)

## Introduction

Les relations **many-to-many** (plusieurs-à-plusieurs) sont parmi les plus complexes à modéliser dans n'importe quelle base de données. Elles existent lorsque **plusieurs** documents d'une collection A peuvent être liés à **plusieurs** documents d'une collection B, et inversement.

Dans MongoDB, contrairement aux bases relationnelles qui utilisent systématiquement des tables de jonction, vous avez plusieurs stratégies possibles selon vos besoins et patterns d'accès.

---

## Comprendre les relations Many-to-Many

### Définition

Une relation **many-to-many** signifie qu'une entité A peut être liée à **plusieurs** entités B, et qu'une entité B peut être liée à **plusieurs** entités A.

**Exemples concrets :**

- **Étudiants ↔ Cours** : Un étudiant suit plusieurs cours, et un cours est suivi par plusieurs étudiants
- **Produits ↔ Tags** : Un produit a plusieurs tags, et un tag est associé à plusieurs produits
- **Acteurs ↔ Films** : Un acteur joue dans plusieurs films, et un film a plusieurs acteurs
- **Utilisateurs ↔ Groupes** : Un utilisateur appartient à plusieurs groupes, et un groupe contient plusieurs utilisateurs
- **Articles ↔ Auteurs** : Un article peut avoir plusieurs co-auteurs, et un auteur écrit plusieurs articles
- **Projets ↔ Développeurs** : Un projet implique plusieurs développeurs, et un développeur travaille sur plusieurs projets

### Caractéristiques

- **Cardinalité M:N** : Plusieurs occurrences de A liées à plusieurs occurrences de B
- **Bidirectionnalité** : La relation peut être consultée dans les deux sens
- **Métadonnées** : Souvent, la relation elle-même porte des informations (ex : date d'inscription d'un étudiant à un cours)

---

## Stratégies de modélisation

Il existe quatre approches principales pour modéliser une relation many-to-many dans MongoDB :

1. **Références bidirectionnelles** (Two-Way Embedding)
2. **Collection de jonction** (Junction Collection)
3. **Embedding d'un côté avec dénormalisation**
4. **Approche hybride**

---

## 1. Références bidirectionnelles : Two-Way Embedding

### Principe

Stocker un **tableau de références** dans chaque collection : A contient les IDs de B, et B contient les IDs de A.

### Quand utiliser Two-Way Embedding ?

✅ **Utilisez des références bidirectionnelles quand :**

- Vous avez besoin de **requêter dans les deux sens** fréquemment
- Le nombre d'associations est **modéré** (dizaines à centaines par document)
- Vous n'avez **pas besoin d'attributs** sur la relation elle-même
- Vous acceptez le **compromis de cohérence**

⚠️ **Attention :** Maintenir la cohérence des deux côtés nécessite des transactions ou une gestion applicative rigoureuse.

### Exemple 1 : Étudiants et Cours

**Scénario :** Système universitaire où les étudiants s'inscrivent à des cours.

**Collection "etudiants" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "matricule": "ETU-2024-001",
  "nom": "Dupont",
  "prenom": "Sophie",
  "email": "sophie.dupont@universite.fr",
  "dateNaissance": ISODate("2002-03-15"),
  "coursIds": [  // ← Tableau de références vers les cours
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d1"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d2")
  ],
  "niveau": "Licence 3",
  "anneeEntree": 2022
}
```

**Collection "cours" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  "code": "INFO-301",
  "titre": "Bases de données avancées",
  "description": "Étude approfondie des systèmes de bases de données...",
  "professeur": "Dr. Jean Martin",
  "credits": 6,
  "semestre": "Automne 2024",
  "etudiantIds": [  // ← Tableau de références vers les étudiants
    ObjectId("507f1f77bcf86cd799439011"),
    ObjectId("507f1f77bcf86cd799439012"),
    ObjectId("507f1f77bcf86cd799439013"),
    ObjectId("507f1f77bcf86cd799439014")
    // ... jusqu'à la capacité maximale
  ],
  "capaciteMax": 50,
  "salle": "Amphi B"
}
```

**Avantages :**

- ✅ **Requêtes bidirectionnelles rapides** : pas besoin de jointure
- ✅ **Liste immédiate** : obtenir tous les cours d'un étudiant ou tous les étudiants d'un cours
- ✅ **Performance** : pas de collection supplémentaire à interroger

**Inconvénients :**

- ⚠️ **Cohérence difficile** : si on inscrit un étudiant, il faut mettre à jour les deux documents
- ⚠️ **Risque d'incohérence** : un étudiant pourrait être dans un cours sans que le cours le liste
- ⚠️ **Pas d'attributs sur la relation** : difficile de stocker la date d'inscription, la note, etc.

**Requêtes :**

```javascript
// Trouver tous les cours d'un étudiant
db.etudiants.findOne({ matricule: "ETU-2024-001" })
// Puis récupérer les cours
db.cours.find({
  _id: { $in: etudiant.coursIds }
})

// Trouver tous les étudiants d'un cours
db.cours.findOne({ code: "INFO-301" })
// Puis récupérer les étudiants
db.etudiants.find({
  _id: { $in: cours.etudiantIds }
})

// Inscrire un étudiant à un cours (avec transaction pour cohérence)
const session = db.getMongo().startSession()
session.startTransaction()

try {
  // Ajouter le cours à l'étudiant
  db.etudiants.updateOne(
    { _id: ObjectId("507f1f77bcf86cd799439011") },
    { $addToSet: { coursIds: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0") } },
    { session }
  )

  // Ajouter l'étudiant au cours
  db.cours.updateOne(
    { _id: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0") },
    { $addToSet: { etudiantIds: ObjectId("507f1f77bcf86cd799439011") } },
    { session }
  )

  session.commitTransaction()
} catch (error) {
  session.abortTransaction()
  throw error
} finally {
  session.endSession()
}

// Désinscrire un étudiant d'un cours
const session = db.getMongo().startSession()
session.startTransaction()

try {
  db.etudiants.updateOne(
    { _id: ObjectId("507f1f77bcf86cd799439011") },
    { $pull: { coursIds: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0") } },
    { session }
  )

  db.cours.updateOne(
    { _id: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0") },
    { $pull: { etudiantIds: ObjectId("507f1f77bcf86cd799439011") } },
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

### Exemple 2 : Produits et Tags

**Scénario :** E-commerce où les produits ont plusieurs tags pour la recherche.

**Collection "produits" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439015"),
  "nom": "Smartphone XYZ Pro",
  "description": "Smartphone haute performance...",
  "prix": 899.99,
  "marque": "TechBrand",
  "tagIds": [  // ← Tags du produit
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),  // "smartphone"
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d6"),  // "5G"
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d7"),  // "android"
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d8")   // "camera-professionnelle"
  ],
  "stock": 45
}
```

**Collection "tags" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
  "nom": "smartphone",
  "slug": "smartphone",
  "description": "Téléphones intelligents",
  "produitIds": [  // ← Produits ayant ce tag
    ObjectId("507f1f77bcf86cd799439015"),
    ObjectId("507f1f77bcf86cd799439016"),
    ObjectId("507f1f77bcf86cd799439017")
    // ... potentiellement des milliers de produits
  ],
  "nombreProduits": 1234,  // Dénormalisé pour performance
  "couleur": "#3498db"  // Pour affichage
}
```

**Note :** Pour ce cas, le tableau `produitIds` dans `tags` peut devenir **très grand**. Une collection de jonction serait peut-être plus appropriée.

---

## 2. Collection de jonction : Junction Collection

### Principe

Créer une **collection séparée** qui stocke les associations entre les deux entités, similaire aux tables de jonction en SQL.

### Quand utiliser une collection de jonction ?

✅ **Utilisez une collection de jonction quand :**

- Vous avez besoin de **stocker des attributs** sur la relation (date, statut, note, etc.)
- Le nombre d'associations est **très important** (milliers, millions)
- Vous voulez **faciliter la maintenance** de la cohérence
- Vous avez besoin de **requêtes complexes** sur les associations
- Les associations ont leur **propre cycle de vie**

### Exemple 1 : Étudiants et Cours avec métadonnées

**Collection "etudiants" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "matricule": "ETU-2024-001",
  "nom": "Dupont",
  "prenom": "Sophie",
  "email": "sophie.dupont@universite.fr",
  "niveau": "Licence 3",
  "anneeEntree": 2022
}
```

**Collection "cours" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  "code": "INFO-301",
  "titre": "Bases de données avancées",
  "professeur": "Dr. Jean Martin",
  "credits": 6,
  "semestre": "Automne 2024",
  "capaciteMax": 50
}
```

**Collection "inscriptions" (jonction) :**
```json
{
  "_id": ObjectId("65a1f1b2c3d4e5f6a7b8c9e0"),
  "etudiantId": ObjectId("507f1f77bcf86cd799439011"),
  "coursId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  // ↓ Attributs de la relation
  "dateInscription": ISODate("2024-09-01"),
  "statut": "active",  // active, abandonnee, terminee
  "notes": {
    "partiel": 15,
    "projet": 17,
    "examenFinal": 16,
    "moyenne": 16
  },
  "presence": {
    "nombreSeances": 24,
    "nombreAbsences": 2,
    "tauxPresence": 91.7
  },
  "dateFinInscription": null
}
```

**Avantages :**

- ✅ **Attributs riches** sur la relation (notes, dates, statut)
- ✅ **Cohérence plus facile** : une seule entrée à créer/modifier/supprimer
- ✅ **Requêtes flexibles** : filtrer par date, statut, note, etc.
- ✅ **Scalabilité** : pas de limite sur le nombre d'associations
- ✅ **Historique** : conserver les anciennes inscriptions

**Inconvénients :**

- ⚠️ **Requêtes plus complexes** : nécessite des jointures avec `$lookup`
- ⚠️ **Performance** : 2 à 3 requêtes pour obtenir les données complètes
- ⚠️ **Collection supplémentaire** à gérer

**Requêtes :**

```javascript
// Trouver tous les cours d'un étudiant avec les détails
db.inscriptions.aggregate([
  { $match: {
      etudiantId: ObjectId("507f1f77bcf86cd799439011"),
      statut: "active"
    }
  },
  {
    $lookup: {
      from: "cours",
      localField: "coursId",
      foreignField: "_id",
      as: "coursDetails"
    }
  },
  { $unwind: "$coursDetails" },
  {
    $project: {
      coursCode: "$coursDetails.code",
      coursTitre: "$coursDetails.titre",
      professeur: "$coursDetails.professeur",
      dateInscription: 1,
      moyenne: "$notes.moyenne"
    }
  }
])

// Trouver tous les étudiants d'un cours
db.inscriptions.aggregate([
  { $match: {
      coursId: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
      statut: "active"
    }
  },
  {
    $lookup: {
      from: "etudiants",
      localField: "etudiantId",
      foreignField: "_id",
      as: "etudiantDetails"
    }
  },
  { $unwind: "$etudiantDetails" },
  {
    $project: {
      matricule: "$etudiantDetails.matricule",
      nom: "$etudiantDetails.nom",
      prenom: "$etudiantDetails.prenom",
      dateInscription: 1,
      moyenne: "$notes.moyenne",
      presence: "$presence.tauxPresence"
    }
  },
  { $sort: { "etudiantDetails.nom": 1 } }
])

// Inscrire un étudiant à un cours (simple !)
db.inscriptions.insertOne({
  etudiantId: ObjectId("507f1f77bcf86cd799439011"),
  coursId: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  dateInscription: new Date(),
  statut: "active",
  notes: {},
  presence: { nombreSeances: 0, nombreAbsences: 0 }
})

// Mettre à jour une note
db.inscriptions.updateOne(
  {
    etudiantId: ObjectId("507f1f77bcf86cd799439011"),
    coursId: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0")
  },
  {
    $set: {
      "notes.partiel": 15,
      "notes.moyenne": 15
    }
  }
)

// Désinscrire un étudiant
db.inscriptions.updateOne(
  {
    etudiantId: ObjectId("507f1f77bcf86cd799439011"),
    coursId: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0")
  },
  {
    $set: {
      statut: "abandonnee",
      dateFinInscription: new Date()
    }
  }
)

// Statistiques : cours les plus populaires
db.inscriptions.aggregate([
  { $match: { statut: "active" } },
  { $group: {
      _id: "$coursId",
      nombreEtudiants: { $sum: 1 },
      moyenneGenerale: { $avg: "$notes.moyenne" }
    }
  },
  {
    $lookup: {
      from: "cours",
      localField: "_id",
      foreignField: "_id",
      as: "coursDetails"
    }
  },
  { $unwind: "$coursDetails" },
  { $sort: { nombreEtudiants: -1 } },
  { $limit: 10 }
])
```

**Index recommandés :**

```javascript
// Pour rechercher les cours d'un étudiant
db.inscriptions.createIndex({ etudiantId: 1, statut: 1 })

// Pour rechercher les étudiants d'un cours
db.inscriptions.createIndex({ coursId: 1, statut: 1 })

// Index composé pour éviter les doublons
db.inscriptions.createIndex(
  { etudiantId: 1, coursId: 1 },
  { unique: true }
)
```

### Exemple 2 : Acteurs et Films

**Collection "acteurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439020"),
  "nom": "Dujardin",
  "prenom": "Jean",
  "dateNaissance": ISODate("1972-06-19"),
  "nationalite": "Française",
  "photo": "https://exemple.com/photos/dujardin.jpg"
}
```

**Collection "films" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9e5"),
  "titre": "The Artist",
  "anneeSortie": 2011,
  "realisateur": "Michel Hazanavicius",
  "genre": ["Drame", "Comédie", "Romance"],
  "duree": 100
}
```

**Collection "roles" (jonction) :**
```json
{
  "_id": ObjectId("65a1f1b2c3d4e5f6a7b8c9f0"),
  "acteurId": ObjectId("507f1f77bcf86cd799439020"),
  "filmId": ObjectId("60a1f1b2c3d4e5f6a7b8c9e5"),
  // ↓ Métadonnées du rôle
  "nomPersonnage": "George Valentin",
  "typeRole": "principal",  // principal, secondaire, figurant
  "ordreGenerique": 1,
  "cachets": 500000,
  "dureeEcran": 85,  // minutes
  "recompenses": [
    {
      "nom": "Oscar du Meilleur Acteur",
      "annee": 2012
    },
    {
      "nom": "BAFTA du Meilleur Acteur",
      "annee": 2012
    }
  ]
}
```

### Exemple 3 : Projets et Développeurs

**Collection "projets" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439025"),
  "nom": "Refonte Application Mobile",
  "description": "Modernisation de l'application...",
  "dateDebut": ISODate("2024-01-01"),
  "dateFinPrevue": ISODate("2024-06-30"),
  "budget": 150000,
  "statut": "en_cours"
}
```

**Collection "developpeurs" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9f5"),
  "nom": "Martin",
  "prenom": "Sophie",
  "email": "sophie.martin@entreprise.com",
  "competences": ["React", "Node.js", "MongoDB"],
  "niveau": "Senior",
  "tauxJournalier": 650
}
```

**Collection "affectations" (jonction) :**
```json
{
  "_id": ObjectId("65a1f1b2c3d4e5f6a7b8c9f8"),
  "projetId": ObjectId("507f1f77bcf86cd799439025"),
  "developpeurId": ObjectId("60a1f1b2c3d4e5f6a7b8c9f5"),
  // ↓ Détails de l'affectation
  "role": "Lead Developer",
  "dateDebut": ISODate("2024-01-01"),
  "dateFin": ISODate("2024-06-30"),
  "allocation": 80,  // Pourcentage de temps (80% sur ce projet)
  "tauxJournalier": 650,
  "joursFactures": 85,
  "coutTotal": 55250,
  "technologiesPrincipales": ["React Native", "Node.js", "MongoDB"],
  "responsabilites": [
    "Architecture technique",
    "Revue de code",
    "Mentoring équipe"
  ],
  "statut": "actif"
}
```

---

## 3. Embedding d'un côté avec dénormalisation

### Principe

Imbriquer les données d'un côté de la relation en **dénormalisant** les informations importantes.

### Quand utiliser cette approche ?

✅ **Utilisez l'embedding dénormalisé quand :**

- L'une des directions est **beaucoup plus consultée** que l'autre
- Les données d'un côté sont **relativement stables**
- Vous voulez **optimiser les lectures** dans une direction
- Vous acceptez la **duplication de données**

### Exemple 1 : Articles et Auteurs

**Scénario :** Blog où les articles ont souvent plusieurs co-auteurs.

**Collection "auteurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439030"),
  "nom": "Martin",
  "prenom": "Jean",
  "email": "jean.martin@blog.com",
  "bio": "Développeur passionné...",
  "photo": "https://exemple.com/photos/jean.jpg",
  "specialites": ["MongoDB", "Node.js"],
  "statistiques": {
    "nombreArticles": 47,
    "nombreCoAuteurs": 12
  }
}
```

**Collection "articles" (avec auteurs imbriqués) :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8ca00"),
  "titre": "Guide complet de MongoDB",
  "slug": "guide-complet-mongodb",
  "contenu": "Dans cet article détaillé...",
  "auteurs": [  // ← Auteurs imbriqués (dénormalisés)
    {
      "id": ObjectId("507f1f77bcf86cd799439030"),
      "nom": "Jean Martin",
      "photo": "https://exemple.com/photos/jean.jpg",
      "role": "Auteur principal"
    },
    {
      "id": ObjectId("507f1f77bcf86cd799439031"),
      "nom": "Sophie Dupont",
      "photo": "https://exemple.com/photos/sophie.jpg",
      "role": "Co-auteur"
    },
    {
      "id": ObjectId("507f1f77bcf86cd799439032"),
      "nom": "Pierre Leclerc",
      "photo": "https://exemple.com/photos/pierre.jpg",
      "role": "Contributeur"
    }
  ],
  "datePublication": ISODate("2024-01-15"),
  "tags": ["mongodb", "nosql", "tutorial"],
  "statut": "publie",
  "statistiques": {
    "vues": 5234,
    "likes": 342
  }
}
```

**Avantages :**

- ✅ **Affichage rapide** : toutes les infos pour afficher l'article en une requête
- ✅ **Byline complet** : noms et photos des auteurs immédiatement disponibles
- ✅ **Pas de jointure** pour la page article

**Inconvénients :**

- ⚠️ **Duplication** : les infos auteur sont dupliquées dans chaque article
- ⚠️ **Mise à jour complexe** : si un auteur change sa photo, faut mettre à jour tous ses articles
- ⚠️ **Requête inverse difficile** : trouver tous les articles d'un auteur nécessite une recherche

**Requêtes :**

```javascript
// Afficher un article avec tous ses auteurs (instantané)
db.articles.findOne({ slug: "guide-complet-mongodb" })

// Trouver tous les articles d'un auteur (recherche dans tableau)
db.articles.find({
  "auteurs.id": ObjectId("507f1f77bcf86cd799439030")
})

// Mettre à jour la photo d'un auteur dans tous ses articles
db.articles.updateMany(
  { "auteurs.id": ObjectId("507f1f77bcf86cd799439030") },
  {
    $set: {
      "auteurs.$[elem].photo": "https://exemple.com/photos/jean-new.jpg"
    }
  },
  {
    arrayFilters: [{ "elem.id": ObjectId("507f1f77bcf86cd799439030") }]
  }
)
```

**Index recommandé :**
```javascript
db.articles.createIndex({ "auteurs.id": 1 })
```

### Exemple 2 : Playlist et Chansons

**Collection "chansons" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439040"),
  "titre": "Imagine",
  "artiste": "John Lennon",
  "album": "Imagine",
  "annee": 1971,
  "duree": 183,
  "genre": "Rock",
  "fichier": "imagine.mp3"
}
```

**Collection "playlists" (avec chansons imbriquées) :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8ca10"),
  "nom": "Rock Classics",
  "description": "Les meilleurs classiques du rock",
  "proprietaire": "sophie.martin@example.com",
  "publique": true,
  "chansons": [  // ← Chansons imbriquées
    {
      "id": ObjectId("507f1f77bcf86cd799439040"),
      "titre": "Imagine",
      "artiste": "John Lennon",
      "duree": 183,
      "position": 1,
      "dateAjout": ISODate("2024-01-10")
    },
    {
      "id": ObjectId("507f1f77bcf86cd799439041"),
      "titre": "Bohemian Rhapsody",
      "artiste": "Queen",
      "duree": 354,
      "position": 2,
      "dateAjout": ISODate("2024-01-10")
    }
    // ... autres chansons
  ],
  "nombreChansons": 25,
  "dureeTotale": 5420,  // secondes
  "dateCreation": ISODate("2024-01-10"),
  "dateModification": ISODate("2024-01-15")
}
```

**Avantages :**
- ✅ **Lecture playlist optimale** : tout en une requête
- ✅ **Ordre préservé** : position de chaque chanson
- ✅ **Snapshot** : même si la chanson originale change, la playlist garde ses infos

---

## 4. Approche hybride

### Principe

Combiner plusieurs approches selon les besoins : références + dénormalisation + éventuellement collection de jonction.

### Exemple : Réseau social - Utilisateurs et Groupes

**Collection "utilisateurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439050"),
  "nom": "Martin",
  "prenom": "Sophie",
  "email": "sophie.martin@example.com",
  "photo": "https://exemple.com/photos/sophie.jpg",
  "groupeIds": [  // ← Références aux groupes
    ObjectId("60a1f1b2c3d4e5f6a7b8ca20"),
    ObjectId("60a1f1b2c3d4e5f6a7b8ca21")
  ],
  "groupesPrincipaux": [  // ← Top 3 groupes (dénormalisés)
    {
      "id": ObjectId("60a1f1b2c3d4e5f6a7b8ca20"),
      "nom": "Développeurs MongoDB",
      "logo": "https://exemple.com/logos/mongodb-dev.png"
    },
    {
      "id": ObjectId("60a1f1b2c3d4e5f6a7b8ca21"),
      "nom": "JavaScript Enthusiasts",
      "logo": "https://exemple.com/logos/js-enthu.png"
    }
  ],
  "statistiques": {
    "nombreGroupes": 15
  }
}
```

**Collection "groupes" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8ca20"),
  "nom": "Développeurs MongoDB",
  "description": "Groupe pour partager...",
  "logo": "https://exemple.com/logos/mongodb-dev.png",
  "type": "public",
  "createur": ObjectId("507f1f77bcf86cd799439050"),
  "nombreMembres": 1234,  // Dénormalisé
  "dateCreation": ISODate("2022-03-15")
}
```

**Collection "appartenances" (jonction avec métadonnées) :**
```json
{
  "_id": ObjectId("65a1f1b2c3d4e5f6a7b8ca25"),
  "utilisateurId": ObjectId("507f1f77bcf86cd799439050"),
  "groupeId": ObjectId("60a1f1b2c3d4e5f6a7b8ca20"),
  "role": "moderateur",  // membre, moderateur, admin
  "dateAdhesion": ISODate("2023-01-10"),
  "notifications": true,
  "frequenceDigest": "quotidien",
  "statistiques": {
    "nombrePublications": 42,
    "nombreCommentaires": 156,
    "dernierAcces": ISODate("2024-01-20")
  }
}
```

**Avantages de cette approche hybride :**

- ✅ **Affichage profil rapide** : groupes principaux imbriqués
- ✅ **Liste complète disponible** : via `groupeIds`
- ✅ **Métadonnées riches** : collection d'appartenances
- ✅ **Compteur dénormalisé** : `nombreMembres` pour affichage rapide

---

## Comparaison des approches

| Approche | Complexité | Performance lecture | Cohérence | Métadonnées | Scalabilité |
|----------|-----------|---------------------|-----------|-------------|-------------|
| **Références bidirectionnelles** | Moyenne | ✅ Excellente | ⚠️ Difficile | ❌ Non | ⚠️ Limitée |
| **Collection de jonction** | Faible | ⚠️ 2-3 requêtes | ✅ Facile | ✅ Oui | ✅ Excellente |
| **Embedding dénormalisé** | Moyenne | ✅ Excellente | ⚠️ Complexe | ⚠️ Limités | ⚠️ Moyenne |
| **Hybride** | Élevée | ✅ Très bonne | ⚠️ Moyenne | ✅ Oui | ✅ Bonne |

---

## Arbre de décision

```
Avez-vous besoin de stocker des attributs sur la relation ?
│
├─ OUI → COLLECTION DE JONCTION
│         (inscriptions, rôles, affectations)
│
└─ NON
   │
   └─ Dans quelle direction consultez-vous principalement ?
      │
      ├─ Les DEUX directions également
      │  │
      │  └─ Nombre d'associations par document ?
      │     │
      │     ├─ < 100 → RÉFÉRENCES BIDIRECTIONNELLES
      │     └─ > 100 → COLLECTION DE JONCTION
      │
      └─ UNE direction principalement
         │
         └─ Nombre d'associations ?
            │
            ├─ < 100 → EMBEDDING DÉNORMALISÉ
            └─ > 100 → HYBRIDE (références + cache)
```

---

## Patterns avancés

### Pattern 1 : Subset avec références

Imbriquer les N éléments les plus importants, tout en gardant la liste complète en référence.

**Exemple : Produit avec tags principaux**

```json
{
  "_id": ObjectId("..."),
  "nom": "Smartphone XYZ",
  "prix": 899.99,
  "tagsPrincipaux": [  // ← 3 tags les plus importants (imbriqués)
    {
      "id": ObjectId("..."),
      "nom": "smartphone",
      "populaire": true
    },
    {
      "id": ObjectId("..."),
      "nom": "5G",
      "populaire": true
    }
  ],
  "tousLesTagIds": [  // ← Tous les tags (références)
    ObjectId("..."),
    ObjectId("..."),
    ObjectId("..."),
    ObjectId("..."),
    ObjectId("...")
  ],
  "nombreTags": 12
}
```

### Pattern 2 : Collection de jonction avec cache

Maintenir une collection de jonction tout en cachant des compteurs.

**Exemple : Groupes avec compteur de membres**

```json
// Collection "groupes"
{
  "_id": ObjectId("..."),
  "nom": "Développeurs MongoDB",
  "nombreMembres": 1234,  // ← Cache du compteur
  "derniereMiseAJourCompteur": ISODate("2024-01-20")
}

// Collection "appartenances"
{
  "_id": ObjectId("..."),
  "groupeId": ObjectId("..."),
  "utilisateurId": ObjectId("..."),
  "dateAdhesion": ISODate("2023-01-10")
}

// Mise à jour du cache périodiquement ou via trigger
db.groupes.updateOne(
  { _id: groupeId },
  {
    $set: {
      nombreMembres: db.appartenances.countDocuments({ groupeId }),
      derniereMiseAJourCompteur: new Date()
    }
  }
)
```

### Pattern 3 : Références avec métadonnées minimales

Stocker la référence ET quelques métadonnées dans le tableau.

**Exemple : Utilisateur avec historique de groupes**

```json
{
  "_id": ObjectId("..."),
  "nom": "Sophie Martin",
  "groupes": [
    {
      "groupeId": ObjectId("..."),
      "dateAdhesion": ISODate("2023-01-10"),
      "role": "membre",
      "actif": true
    },
    {
      "groupeId": ObjectId("..."),
      "dateAdhesion": ISODate("2022-05-15"),
      "role": "moderateur",
      "actif": true
    }
  ]
}
```

---

## Cas d'usage détaillés

### Cas 1 : Réseau social - Amis

**Problème :** Les relations d'amitié sont bidirectionnelles.

**Solution :** Collection de jonction pour gérer le statut.

```json
// Collection "amities"
{
  "_id": ObjectId("..."),
  "utilisateur1Id": ObjectId("..."),  // Toujours le plus petit ID
  "utilisateur2Id": ObjectId("..."),  // Toujours le plus grand ID
  "statut": "acceptee",  // en_attente, acceptee, bloquee
  "initiateur": ObjectId("..."),  // Qui a envoyé la demande
  "dateDemandeAmitie": ISODate("2024-01-10"),
  "dateAcceptation": ISODate("2024-01-11")
}
```

**Index pour éviter les doublons :**
```javascript
db.amities.createIndex(
  { utilisateur1Id: 1, utilisateur2Id: 1 },
  { unique: true }
)
```

### Cas 2 : E-commerce - Wishlist

**Solution :** Embedding avec références.

```json
{
  "_id": ObjectId("..."),
  "utilisateurId": ObjectId("..."),
  "nom": "Ma wishlist",
  "produits": [
    {
      "produitId": ObjectId("..."),
      "nom": "Smartphone XYZ",  // Dénormalisé
      "prix": 899.99,  // Dénormalisé
      "image": "https://...",  // Dénormalisé
      "dateAjout": ISODate("2024-01-15"),
      "priorite": "haute",
      "notes": "Pour mon anniversaire"
    }
  ],
  "dateCreation": ISODate("2023-12-01")
}
```

### Cas 3 : Système de recommandations

**Solution :** Collection de jonction avec scores.

```json
{
  "_id": ObjectId("..."),
  "utilisateurId": ObjectId("..."),
  "produitId": ObjectId("..."),
  "scoreRecommandation": 0.87,
  "raisons": [
    "Basé sur vos achats précédents",
    "Populaire parmi les utilisateurs similaires"
  ],
  "dateCalcul": ISODate("2024-01-20"),
  "affiche": false,
  "clique": false
}
```

---

## Recommandations pratiques

### ✅ Bonnes pratiques

1. **Privilégiez la collection de jonction** si vous avez des métadonnées
2. **Utilisez des transactions** pour les références bidirectionnelles
3. **Créez des index** sur tous les champs de référence
4. **Dénormalisez intelligemment** : seulement les champs fréquents et stables
5. **Maintenez des compteurs** pour éviter les COUNT coûteux
6. **Documentez votre stratégie** : expliquez pourquoi vous avez choisi telle approche
7. **Testez les performances** avec des volumes réalistes

### ⚠️ Pièges à éviter

1. **Références bidirectionnelles sans transactions** → incohérences
2. **Embedding de données qui changent souvent** → mises à jour massives
3. **Oublier les index** sur les champs de recherche
4. **Collection de jonction sans attributs** → utiliser des références serait plus simple
5. **Ne pas anticiper la croissance** → tableaux qui explosent la limite de 16 Mo

### 🔧 Maintenance et cohérence

```javascript
// Script de vérification de cohérence (références bidirectionnelles)
db.etudiants.find().forEach(etudiant => {
  etudiant.coursIds.forEach(coursId => {
    const cours = db.cours.findOne({ _id: coursId })
    if (!cours || !cours.etudiantIds.includes(etudiant._id)) {
      print(`Incohérence détectée : Étudiant ${etudiant._id} référence cours ${coursId}`)
    }
  })
})

// Recalculer les compteurs dénormalisés
db.groupes.find().forEach(groupe => {
  const nombreMembres = db.appartenances.countDocuments({ groupeId: groupe._id })
  if (groupe.nombreMembres !== nombreMembres) {
    db.groupes.updateOne(
      { _id: groupe._id },
      { $set: { nombreMembres } }
    )
    print(`Groupe ${groupe.nom} : compteur corrigé (${groupe.nombreMembres} → ${nombreMembres})`)
  }
})
```

---

## Conclusion

Les relations **many-to-many** dans MongoDB offrent plusieurs options de modélisation, chacune avec ses avantages et inconvénients :

**Résumé des recommandations :**

1. **Avec métadonnées** → **Collection de jonction** (meilleur choix général)
2. **Sans métadonnées + requêtes bidirectionnelles** → **Références bidirectionnelles** (avec transactions)
3. **Sans métadonnées + une direction prioritaire** → **Embedding dénormalisé**
4. **Cas complexes** → **Approche hybride**

**Facteurs de décision :**

- 📊 **Métadonnées** : Oui → Collection de jonction
- 📊 **Direction de requête** : Une seule → Embedding / Les deux → Références ou jonction
- 📊 **Nombre d'associations** : < 100 → Références / > 100 → Jonction
- 📊 **Fréquence de modification** : Élevée → Jonction / Faible → Embedding

N'oubliez pas : il n'y a pas de solution universelle. Analysez vos patterns d'accès et choisissez l'approche qui correspond le mieux à vos besoins !

---

**Points clés à retenir :**

- ✅ Les relations many-to-many sont les plus complexes à modéliser
- ✅ La collection de jonction est souvent le meilleur choix
- ✅ Les références bidirectionnelles nécessitent des transactions
- ✅ L'embedding dénormalisé optimise les lectures d'une direction
- ✅ Les approches hybrides offrent le meilleur des deux mondes
- ✅ Toujours créer des index sur les champs de référence
- ✅ Tester avec des volumes réalistes avant de décider

---


⏭️ [Patterns de modélisation](/04-modelisation-des-donnees/06-patterns-modelisation.md)
