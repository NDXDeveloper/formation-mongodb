🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.2 Documents imbriqués vs Références

## Introduction

L'une des décisions les plus importantes lors de la modélisation de données dans MongoDB est de choisir entre **imbriquer** les données dans un document ou utiliser des **références** vers d'autres documents. Ce choix fondamental influence directement les performances, la complexité des requêtes et la maintenabilité de votre application.

Dans cette section, nous allons explorer en profondeur ces deux approches, comprendre leurs avantages et inconvénients, et apprendre à faire le bon choix selon votre contexte.

---

## Comprendre les deux approches

### Documents imbriqués (Embedded Documents)

L'**imbrication** consiste à stocker des données liées directement à l'intérieur d'un document parent. Les données imbriquées font partie intégrante du document et sont stockées ensemble physiquement.

**Exemple - Utilisateur avec adresses imbriquées :**

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nom": "Martin",
  "prenom": "Sophie",
  "email": "sophie.martin@example.com",
  "adresses": [
    {
      "type": "domicile",
      "rue": "12 rue de la République",
      "ville": "Lyon",
      "codePostal": "69001",
      "pays": "France"
    },
    {
      "type": "bureau",
      "rue": "45 avenue des Champs",
      "ville": "Lyon",
      "codePostal": "69003",
      "pays": "France"
    }
  ],
  "dateInscription": ISODate("2024-01-10")
}
```

Dans cet exemple, les adresses sont **complètement intégrées** dans le document utilisateur.

### Références (References)

Les **références** stockent uniquement l'identifiant (`_id`) d'un document lié, similaire aux clés étrangères dans les bases relationnelles. Les données sont réparties dans plusieurs documents et collections.

**Exemple - Utilisateur avec références aux adresses :**

**Collection "utilisateurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nom": "Martin",
  "prenom": "Sophie",
  "email": "sophie.martin@example.com",
  "adresseIds": [
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
    ObjectId("60a1f1b2c3d4e5f6a7b8c9d1")
  ],
  "dateInscription": ISODate("2024-01-10")
}
```

**Collection "adresses" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  "utilisateurId": ObjectId("507f1f77bcf86cd799439011"),
  "type": "domicile",
  "rue": "12 rue de la République",
  "ville": "Lyon",
  "codePostal": "69001",
  "pays": "France"
}
```

```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d1"),
  "utilisateurId": ObjectId("507f1f77bcf86cd799439011"),
  "type": "bureau",
  "rue": "45 avenue des Champs",
  "ville": "Lyon",
  "codePostal": "69003",
  "pays": "France"
}
```

Dans cet exemple, les adresses sont dans une collection séparée et **référencées** par leur identifiant.

---

## Comparaison approfondie

### 1. Performances de lecture

#### Documents imbriqués ✅

**Avantage majeur :** Une seule requête suffit pour récupérer toutes les données.

```javascript
// Une seule requête pour tout obtenir
db.utilisateurs.findOne({ _id: ObjectId("507f1f77bcf86cd799439011") })
```

Résultat : L'utilisateur ET toutes ses adresses en une seule opération.

**Performance :** Excellente, car toutes les données sont stockées ensemble physiquement sur le disque.

#### Références ⚠️

**Nécessite plusieurs requêtes :**

```javascript
// Requête 1 : Récupérer l'utilisateur
const utilisateur = db.utilisateurs.findOne({
  _id: ObjectId("507f1f77bcf86cd799439011")
})

// Requête 2 : Récupérer les adresses
const adresses = db.adresses.find({
  _id: { $in: utilisateur.adresseIds }
})
```

**Performance :** Plus lente, nécessite plusieurs accès à la base de données (pas de jointures natives comme en SQL).

**Note :** MongoDB propose `$lookup` pour faire des jointures, mais c'est généralement plus lent que l'imbrication.

### 2. Performances d'écriture

#### Documents imbriqués ⚠️

**Mise à jour simple :**
```javascript
// Mettre à jour une adresse spécifique
db.utilisateurs.updateOne(
  {
    _id: ObjectId("507f1f77bcf86cd799439011"),
    "adresses.type": "domicile"
  },
  {
    $set: { "adresses.$.rue": "25 rue Neuve" }
  }
)
```

**Inconvénient :** Si vous devez mettre à jour la même information dupliquée dans plusieurs documents, cela nécessite plusieurs opérations.

#### Références ✅

**Mise à jour centralisée :**
```javascript
// Mettre à jour une adresse (affecte tous les utilisateurs qui la référencent)
db.adresses.updateOne(
  { _id: ObjectId("60a1f1b2c3d4e5f6a7b8c9d0") },
  { $set: { rue: "25 rue Neuve" } }
)
```

**Avantage :** Les données sont stockées une seule fois, donc une seule mise à jour suffit.

### 3. Duplication des données

#### Documents imbriqués ⚠️

**Duplication :** Les données imbriquées sont spécifiques à chaque document parent. Si plusieurs documents ont les mêmes données imbriquées, il y a duplication.

**Exemple problématique :**
```json
// Utilisateur 1
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "entreprise": {
    "nom": "TechCorp",
    "adresse": "10 rue du Commerce",
    "telephone": "+33 1 23 45 67 89"
  }
}

// Utilisateur 2
{
  "_id": ObjectId("..."),
  "nom": "Martin",
  "entreprise": {
    "nom": "TechCorp",
    "adresse": "10 rue du Commerce",
    "telephone": "+33 1 23 45 67 89"
  }
}
```

Si l'adresse de TechCorp change, il faut mettre à jour tous les documents utilisateurs.

#### Références ✅

**Pas de duplication :** Les données sont stockées une fois et référencées.

```json
// Utilisateur 1
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "entrepriseId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2")
}

// Utilisateur 2
{
  "_id": ObjectId("..."),
  "nom": "Martin",
  "entrepriseId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2")
}

// Entreprise (stockée une seule fois)
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2"),
  "nom": "TechCorp",
  "adresse": "10 rue du Commerce",
  "telephone": "+33 1 23 45 67 89"
}
```

### 4. Atomicité des opérations

#### Documents imbriqués ✅

**Atomicité garantie :** Toute modification d'un document (y compris ses sous-documents) est atomique.

```javascript
// Cette opération est entièrement atomique
db.utilisateurs.updateOne(
  { _id: ObjectId("...") },
  {
    $set: { "adresses.0.rue": "Nouvelle rue" },
    $inc: { version: 1 }
  }
)
```

Soit toutes les modifications réussissent, soit aucune.

#### Références ⚠️

**Atomicité limitée :** Chaque document est modifié atomiquement, mais pas l'ensemble.

```javascript
// Ces deux opérations ne sont PAS atomiques ensemble
db.utilisateurs.updateOne({ _id: ObjectId("...") }, { $set: { nom: "Nouveau" } })
db.adresses.updateOne({ _id: ObjectId("...") }, { $set: { rue: "Nouvelle" } })
```

Si vous avez besoin d'atomicité sur plusieurs documents, vous devrez utiliser des **transactions multi-documents** (disponibles depuis MongoDB 4.0).

### 5. Taille et croissance des documents

#### Documents imbriqués ⚠️

**Limite de 16 Mo :** Attention à la croissance des tableaux imbriqués !

**Exemple problématique :**
```json
{
  "_id": ObjectId("..."),
  "titre": "Article viral",
  "commentaires": [
    // Si des milliers de commentaires...
    // Le document peut atteindre 16 Mo !
  ]
}
```

**Solution :** Utiliser des références si le nombre d'éléments peut croître indéfiniment.

#### Références ✅

**Pas de limite globale :** Chaque document reste petit, quel que soit le nombre total d'éléments liés.

```json
// Article
{
  "_id": ObjectId("..."),
  "titre": "Article viral"
}

// Des milliers de commentaires dans une collection séparée
{
  "_id": ObjectId("..."),
  "articleId": ObjectId("..."),
  "texte": "Commentaire 1"
}
```

---

## Quand utiliser l'imbrication ?

### ✅ Utilisez l'imbrication dans ces cas :

#### 1. Relation "contient" naturelle

Lorsque les données imbriquées **font partie intégrante** de l'entité principale.

**Exemple - Commande avec articles :**
```json
{
  "_id": ObjectId("..."),
  "numeroCommande": "CMD-2024-001",
  "client": "Sophie Martin",
  "articles": [
    {
      "produit": "Livre MongoDB",
      "quantite": 2,
      "prix": 29.99
    },
    {
      "produit": "Clavier",
      "quantite": 1,
      "prix": 89.99
    }
  ],
  "total": 149.97
}
```

Les articles **n'ont pas de sens** en dehors de la commande.

#### 2. Relation un-à-peu (one-to-few)

Quand le nombre d'éléments imbriqués est **limité** et **prévisible**.

**Exemple - Utilisateur avec contacts d'urgence (max 3) :**
```json
{
  "_id": ObjectId("..."),
  "nom": "Martin",
  "contactsUrgence": [
    { "nom": "Marie Martin", "relation": "épouse", "tel": "+33 6 12 34 56 78" },
    { "nom": "Jean Martin", "relation": "frère", "tel": "+33 6 23 45 67 89" }
  ]
}
```

#### 3. Données toujours consultées ensemble

Quand vous avez **toujours besoin** de ces données en même temps que le parent.

**Exemple - Profil utilisateur avec préférences :**
```json
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "email": "dupont@example.com",
  "preferences": {
    "langue": "fr",
    "theme": "sombre",
    "notifications": true,
    "newsletter": false
  }
}
```

#### 4. Besoin d'atomicité

Quand plusieurs données doivent être **modifiées ensemble** de manière cohérente.

**Exemple - Transaction bancaire :**
```json
{
  "_id": ObjectId("..."),
  "numeroTransaction": "TXN-2024-001",
  "montant": 100.00,
  "devise": "EUR",
  "compte": {
    "numero": "FR76 1234 5678",
    "soldeAvant": 1500.00,
    "soldeApres": 1400.00
  },
  "statut": "validee",
  "date": ISODate("2024-01-15")
}
```

#### 5. Performance de lecture critique

Quand la vitesse de lecture est **cruciale** et que vous voulez éviter les requêtes multiples.

---

## Quand utiliser les références ?

### ✅ Utilisez les références dans ces cas :

#### 1. Relation un-à-beaucoup (one-to-many) avec "beaucoup"

Quand le nombre d'éléments liés peut être **très grand** ou **illimité**.

**Exemple - Auteur avec articles :**

**Collection "auteurs" :**
```json
{
  "_id": ObjectId("..."),
  "nom": "Jean Martin",
  "email": "jean.martin@example.com",
  "specialite": "Bases de données"
}
```

**Collection "articles" :**
```json
{
  "_id": ObjectId("..."),
  "titre": "Introduction à MongoDB",
  "auteurId": ObjectId("..."),  // Référence
  "contenu": "...",
  "datePublication": ISODate("2024-01-15")
}
```

Un auteur peut avoir des centaines d'articles, impossible de les imbriquer.

#### 2. Relation plusieurs-à-plusieurs (many-to-many)

Quand plusieurs entités peuvent être liées à plusieurs autres entités.

**Exemple - Étudiants et cours :**

**Collection "etudiants" :**
```json
{
  "_id": ObjectId("..."),
  "nom": "Sophie Dupont",
  "coursIds": [
    ObjectId("60a1..."),  // MongoDB
    ObjectId("60a2..."),  // Node.js
    ObjectId("60a3...")   // React
  ]
}
```

**Collection "cours" :**
```json
{
  "_id": ObjectId("60a1..."),
  "titre": "MongoDB Avancé",
  "etudiantIds": [
    ObjectId("..."),  // Sophie
    ObjectId("..."),  // Pierre
    ObjectId("...")   // Marie
  ]
}
```

#### 3. Données partagées entre plusieurs entités

Quand les mêmes données sont utilisées par **plusieurs documents parents**.

**Exemple - Employés et entreprise :**

```json
// Employé 1
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "entrepriseId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2")
}

// Employé 2
{
  "_id": ObjectId("..."),
  "nom": "Martin",
  "entrepriseId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2")
}

// Entreprise (partagée)
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2"),
  "nom": "TechCorp",
  "adresse": "10 rue du Commerce",
  "telephone": "+33 1 23 45 67 89"
}
```

#### 4. Données consultées indépendamment

Quand les données liées sont souvent consultées **séparément** du parent.

**Exemple - Produits et catégories :**

```json
// Produit
{
  "_id": ObjectId("..."),
  "nom": "Smartphone XYZ",
  "prix": 599.99,
  "categorieId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d3")
}

// Catégorie (peut être consultée seule pour lister tous les produits)
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d3"),
  "nom": "Électronique",
  "description": "Appareils électroniques et accessoires"
}
```

#### 5. Données volumineuses

Quand les données liées sont **volumineuses** et risquent de faire dépasser la limite de 16 Mo.

**Exemple - Vidéos et métadonnées :**

```json
// Vidéo (métadonnées seulement)
{
  "_id": ObjectId("..."),
  "titre": "Tutoriel MongoDB",
  "duree": 1800,
  "fichierVideoId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d4")
}

// Fichier vidéo (stocké séparément, peut-être dans GridFS)
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d4"),
  "fichier": BinData(...),  // Données volumineuses
  "taille": 524288000
}
```

#### 6. Mises à jour fréquentes sur les données liées

Quand les données liées changent souvent et sont partagées.

**Exemple - Prix de produits :**

```json
// Commande (garde une référence au produit)
{
  "_id": ObjectId("..."),
  "numeroCommande": "CMD-2024-001",
  "produitId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
  "prixAchat": 29.99,  // Prix au moment de l'achat (dénormalisé)
  "quantite": 2
}

// Produit (le prix courant peut changer)
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
  "nom": "Livre MongoDB",
  "prixActuel": 24.99  // Prix a changé depuis la commande
}
```

---

## Approche hybride : Le meilleur des deux mondes

Souvent, la meilleure solution combine imbrication et références avec **dénormalisation sélective**.

### Exemple - E-commerce optimisé

**Collection "commandes" :**
```json
{
  "_id": ObjectId("..."),
  "numeroCommande": "CMD-2024-001",
  "client": {
    "id": ObjectId("507f1f77bcf86cd799439011"),
    "nom": "Sophie Martin",  // ✅ Dénormalisé pour performance
    "email": "sophie.martin@example.com"  // ✅ Dénormalisé
  },
  "articles": [
    {
      "produitId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
      "nom": "Livre MongoDB",  // ✅ Dénormalisé (snapshot au moment de la commande)
      "prix": 29.99,  // ✅ Dénormalisé (prix au moment de l'achat)
      "quantite": 2
    }
  ],
  "montantTotal": 59.98,
  "statut": "livree",
  "adresseLivraison": {  // ✅ Imbriqué (spécifique à cette commande)
    "rue": "12 rue de la République",
    "ville": "Lyon",
    "codePostal": "69001"
  },
  "dateCommande": ISODate("2024-01-15")
}
```

**Avantages de cette approche :**
- ✅ **Une seule requête** pour afficher tous les détails de la commande
- ✅ **Snapshot historique** : même si le produit ou le client change, la commande garde ses valeurs
- ✅ **Performance optimale** pour l'affichage
- ✅ **Données essentielles dénormalisées**, références gardées pour aller plus loin si besoin

**Collection "clients" (séparée) :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nom": "Sophie Martin",
  "email": "sophie.martin@example.com",
  "telephone": "+33 6 12 34 56 78",
  "dateInscription": ISODate("2023-06-10"),
  "adresses": [
    // Adresses complètes du client
  ],
  "historique": {
    "nombreCommandes": 15,
    "montantTotal": 1249.85
  }
}
```

**Collection "produits" (séparée) :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
  "nom": "Livre MongoDB",
  "description": "Guide complet...",
  "prixActuel": 24.99,  // Peut changer
  "stock": 150,
  "categorie": "Livres techniques"
}
```

---

## Arbre de décision

Voici un guide pour choisir rapidement :

```
┌─────────────────────────────────────────────┐
│ Les données liées font-elles partie         │
│ intégrante de l'entité principale ?         │
└─────────────┬───────────────────────────────┘
              │
        Oui   │   Non
              │
     ┌────────▼────────┐
     │   IMBRIQUER     │         ┌────────────────────────┐
     └─────────────────┘         │ Nombre d'éléments ?    │
                                 └──────┬─────────────────┘
                                        │
                          Limité (<100) │  Illimité/Beaucoup
                                        │
                                 ┌──────▼─────────┐
                                 │ Consultées     │
                                 │ ensemble ?     │
                                 └──┬─────────────┘
                                    │
                            Oui     │     Non
                                    │
                          ┌─────────▼──────┐    ┌───────────────┐
                          │   IMBRIQUER    │    │  RÉFÉRENCER   │
                          │  (si < 16 Mo)  │    └───────────────┘
                          └────────────────┘
```

### Questions à se poser :

1. **Relation naturelle ?** Les données liées existent-elles indépendamment ?
2. **Cardinalité ?** Combien d'éléments liés (peu, beaucoup, illimité) ?
3. **Patterns d'accès ?** Consultées toujours ensemble ou séparément ?
4. **Taille ?** Risque de dépasser 16 Mo ?
5. **Mises à jour ?** Fréquentes et partagées, ou rares et spécifiques ?
6. **Atomicité ?** Besoin de modifier plusieurs données ensemble ?

---

## Exemples comparatifs détaillés

### Exemple 1 : Blog

#### ❌ Mauvais choix - Commentaires imbriqués (article viral)

```json
{
  "_id": ObjectId("..."),
  "titre": "Article viral",
  "commentaires": [
    // Risque : Des milliers de commentaires → dépasse 16 Mo !
    // Problème : Difficile de paginer les commentaires
    // Problème : Chaque ajout de commentaire réécrit tout le document
  ]
}
```

#### ✅ Bon choix - Commentaires référencés

```json
// Article
{
  "_id": ObjectId("..."),
  "titre": "Article viral",
  "auteur": "Jean Martin",
  "contenu": "...",
  "nombreCommentaires": 15234  // Compteur pour affichage
}

// Commentaires (collection séparée)
{
  "_id": ObjectId("..."),
  "articleId": ObjectId("..."),
  "auteur": "Sophie",
  "texte": "Super article !",
  "date": ISODate("2024-01-15")
}
```

**Requête pour afficher l'article avec les 10 premiers commentaires :**
```javascript
const article = db.articles.findOne({ _id: ObjectId("...") })
const commentaires = db.commentaires
  .find({ articleId: article._id })
  .sort({ date: -1 })
  .limit(10)
```

### Exemple 2 : Gestion de projets

#### ✅ Bon choix - Tâches imbriquées (projet avec peu de tâches)

```json
{
  "_id": ObjectId("..."),
  "nomProjet": "Refonte site web",
  "responsable": "Sophie Martin",
  "taches": [
    {
      "id": 1,
      "nom": "Maquettes",
      "statut": "terminee",
      "assignee": "Pierre",
      "echeance": ISODate("2024-01-15")
    },
    {
      "id": 2,
      "nom": "Développement frontend",
      "statut": "en_cours",
      "assignee": "Marie",
      "echeance": ISODate("2024-02-01")
    }
  ],
  "budget": 50000,
  "dateDebut": ISODate("2024-01-01")
}
```

**Avantage :** Une seule requête pour afficher le projet complet.

#### ✅ Alternative - Tâches référencées (projet avec beaucoup de tâches)

```json
// Projet
{
  "_id": ObjectId("..."),
  "nomProjet": "Migration infrastructure",
  "responsable": "Sophie Martin",
  "nombreTaches": 247,  // Trop pour imbriquer !
  "budget": 500000
}

// Tâches (collection séparée)
{
  "_id": ObjectId("..."),
  "projetId": ObjectId("..."),
  "nom": "Audit sécurité",
  "statut": "en_cours",
  "assignee": "Pierre",
  "echeance": ISODate("2024-02-15")
}
```

---

## Tableau récapitulatif

| Critère | Imbrication | Références |
|---------|-------------|------------|
| **Performance lecture** | ✅ Excellente (1 requête) | ⚠️ Moyenne (plusieurs requêtes) |
| **Performance écriture** | ⚠️ Peut nécessiter réécriture document | ✅ Modification ciblée |
| **Duplication données** | ⚠️ Possible (accepté) | ✅ Pas de duplication |
| **Atomicité** | ✅ Garantie | ⚠️ Nécessite transactions |
| **Limite taille** | ⚠️ Attention aux 16 Mo | ✅ Pas de limite globale |
| **Cardinalité** | ✅ One-to-few | ✅ One-to-many, Many-to-many |
| **Données partagées** | ❌ Non recommandé | ✅ Idéal |
| **Complexité requêtes** | ✅ Simple | ⚠️ Plus complexe |
| **Cohérence** | ✅ Automatique | ⚠️ À gérer manuellement |

---

## Patterns courants

### Pattern : Extended Reference

Combiner référence et dénormalisation des champs les plus utilisés.

```json
{
  "_id": ObjectId("..."),
  "titre": "Commande #1234",
  "client": {
    "id": ObjectId("..."),  // ✅ Référence
    "nom": "Sophie Martin",  // ✅ Dénormalisé (champ fréquent)
    "email": "sophie@example.com"  // ✅ Dénormalisé
    // Autres détails dans la collection "clients"
  }
}
```

### Pattern : Subset

Imbriquer un sous-ensemble des données les plus pertinentes.

```json
{
  "_id": ObjectId("..."),
  "titre": "Nouveau smartphone",
  "avis": [
    // Les 5 avis les plus récents (imbriqués)
    { "auteur": "Jean", "note": 5, "texte": "Excellent !" }
  ],
  "nombreAvisTotal": 1523,  // Total des avis
  "noteGlobale": 4.5
  // Tous les avis sont dans une collection séparée
}
```

### Pattern : Two-Way Reference

Références bidirectionnelles pour faciliter les requêtes dans les deux sens.

```json
// Étudiant
{
  "_id": ObjectId("..."),
  "nom": "Sophie Dupont",
  "coursIds": [
    ObjectId("60a1..."),
    ObjectId("60a2...")
  ]
}

// Cours
{
  "_id": ObjectId("60a1..."),
  "titre": "MongoDB Avancé",
  "etudiantIds": [
    ObjectId("..."),
    ObjectId("...")
  ]
}
```

Permet de trouver facilement :
- Les cours d'un étudiant
- Les étudiants d'un cours

---

## Recommandations pratiques

### ✅ Bonnes pratiques

1. **Commencez simple** : Préférez l'imbrication quand c'est possible
2. **Mesurez** : Utilisez `explain()` pour évaluer les performances
3. **Pensez croissance** : Anticipez l'évolution du volume de données
4. **Dénormalisez intelligemment** : Dupliquez uniquement les données fréquemment consultées
5. **Documentez vos choix** : Expliquez pourquoi vous avez choisi telle approche

### ⚠️ Pièges à éviter

1. **Normaliser systématiquement** comme en SQL (perdre les bénéfices de MongoDB)
2. **Imbriquer sans limite** (risque de dépasser 16 Mo)
3. **Négliger les patterns d'accès** (modéliser sans connaître les besoins réels)
4. **Oublier les index** sur les champs référencés
5. **Ne pas tester les performances** avec des volumes réalistes

---

## Conclusion

Le choix entre documents imbriqués et références est **contextuel** et dépend de nombreux facteurs :

- **Type de relation** (un-à-peu, un-à-beaucoup, plusieurs-à-plusieurs)
- **Patterns d'accès** (lecture ensemble ou séparément)
- **Volume de données** (limité ou illimité)
- **Fréquence des modifications** (stables ou changeantes)
- **Besoins d'atomicité** (cohérence stricte ou relâchée)

**Règle d'or :** Privilégiez l'imbrication pour optimiser les **lectures**, utilisez les références pour les **données volumineuses, partagées ou fréquemment mises à jour**.

N'oubliez pas qu'une approche **hybride** avec dénormalisation sélective offre souvent le meilleur compromis entre performance et maintenabilité.

---

**Points clés à retenir :**

- ✅ Imbrication = données toujours consultées ensemble, en nombre limité
- ✅ Références = données volumineuses, partagées ou consultées indépendamment
- ✅ La dénormalisation est votre alliée pour les performances
- ✅ Pensez "comment l'application accède-t-elle aux données ?"
- ✅ Attention à la limite de 16 Mo par document
- ✅ Utilisez des approches hybrides pour combiner les avantages
- ✅ Mesurez et ajustez selon vos besoins réels

---


⏭️ [Relations One-to-One](/04-modelisation-des-donnees/03-relations-one-to-one.md)
