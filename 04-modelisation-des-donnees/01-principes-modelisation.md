🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.1 Principes de modélisation orientée document

## Introduction

La modélisation des données dans MongoDB diffère fondamentalement de l'approche traditionnelle des bases de données relationnelles. Alors que les bases relationnelles organisent les données en tables avec des relations strictes, MongoDB adopte une approche orientée document qui offre plus de flexibilité et s'aligne naturellement avec la façon dont les applications modernes manipulent les données.

Dans ce chapitre, nous allons explorer les principes fondamentaux qui guident la modélisation des données dans MongoDB, en comprenant comment penser différemment et tirer parti de la nature orientée document de cette base de données.

---

## Qu'est-ce que la modélisation orientée document ?

### Le document comme unité de base

Dans MongoDB, **le document est l'unité fondamentale de stockage**. Un document est une structure de données composée de paires clé-valeur, similaire à un objet JSON. Contrairement aux bases relationnelles où les données sont dispersées dans plusieurs tables, un document peut contenir toutes les informations liées à une entité.

**Exemple d'un document utilisateur :**

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nom": "Dupont",
  "prenom": "Marie",
  "email": "marie.dupont@example.com",
  "age": 28,
  "adresse": {
    "rue": "15 rue de la Paix",
    "ville": "Paris",
    "codePostal": "75002",
    "pays": "France"
  },
  "telephones": [
    "+33 6 12 34 56 78",
    "+33 1 23 45 67 89"
  ],
  "dateInscription": ISODate("2023-01-15T10:30:00Z"),
  "actif": true
}
```

### Différence avec le modèle relationnel

**Modèle relationnel (SQL) :**
- Les données sont organisées en **tables** avec des colonnes fixes
- Les relations entre entités nécessitent des **clés étrangères** et des **jointures**
- Le schéma est **rigide** et défini à l'avance
- La **normalisation** est encouragée pour éviter la redondance

**Modèle orienté document (MongoDB) :**
- Les données sont organisées en **documents** flexibles
- Les relations peuvent être représentées par **imbrication** ou **référence**
- Le schéma est **flexible** et peut évoluer
- La **dénormalisation** est souvent préférée pour optimiser les performances

---

## Principes fondamentaux de la modélisation

### 1. Modéliser selon les besoins de l'application

**Principe clé :** La structure de vos documents doit refléter la façon dont votre application accède et utilise les données.

Au lieu de concevoir un schéma normalisé abstrait, demandez-vous :
- Quelles données l'application lit-elle ensemble ?
- Quelles sont les requêtes les plus fréquentes ?
- Quelles données sont modifiées ensemble ?

**Exemple :** Si votre application affiche toujours un article de blog avec ses commentaires, il peut être judicieux d'imbriquer les commentaires dans le document de l'article.

```json
{
  "_id": ObjectId("..."),
  "titre": "Introduction à MongoDB",
  "contenu": "MongoDB est une base de données...",
  "auteur": "Jean Martin",
  "datePublication": ISODate("2024-01-10"),
  "commentaires": [
    {
      "auteur": "Sophie L.",
      "texte": "Excellent article !",
      "date": ISODate("2024-01-11")
    },
    {
      "auteur": "Pierre D.",
      "texte": "Très instructif, merci.",
      "date": ISODate("2024-01-12")
    }
  ]
}
```

### 2. Privilégier l'atomicité des opérations

MongoDB garantit l'**atomicité au niveau du document**. Cela signifie que toutes les modifications apportées à un seul document sont atomiques (tout ou rien).

**Implication pratique :** Si plusieurs données doivent être modifiées de manière cohérente, envisagez de les placer dans le même document.

**Exemple - Commande e-commerce :**

```json
{
  "_id": ObjectId("..."),
  "numeroCommande": "CMD-2024-001",
  "client": {
    "nom": "Durand",
    "email": "durand@example.com"
  },
  "articles": [
    {
      "produit": "Livre MongoDB",
      "quantite": 2,
      "prixUnitaire": 29.99
    },
    {
      "produit": "Clavier mécanique",
      "quantite": 1,
      "prixUnitaire": 89.99
    }
  ],
  "montantTotal": 149.97,
  "statut": "en_preparation",
  "dateCommande": ISODate("2024-01-15")
}
```

Toute la commande peut être mise à jour de manière atomique sans risque d'incohérence.

### 3. Accepter la dénormalisation

Contrairement aux bases relationnelles où la normalisation est la norme, MongoDB encourage souvent la **dénormalisation** pour améliorer les performances.

**Normalisation (SQL) :** Éviter la duplication de données
**Dénormalisation (MongoDB) :** Dupliquer certaines données pour éviter les jointures coûteuses

**Exemple - Commandes avec informations client :**

Au lieu de stocker uniquement l'identifiant du client, vous pouvez dupliquer certaines informations :

```json
{
  "_id": ObjectId("..."),
  "numeroCommande": "CMD-2024-002",
  "client": {
    "id": ObjectId("507f1f77bcf86cd799439011"),
    "nom": "Martin",
    "prenom": "Sophie",
    "email": "sophie.martin@example.com"
  },
  "articles": [ /* ... */ ],
  "montantTotal": 299.99
}
```

**Avantage :** Une seule requête suffit pour afficher toutes les informations de la commande, sans jointure.

**Considération :** Si le client change son email, il faudra potentiellement mettre à jour plusieurs documents de commandes. C'est un compromis à évaluer selon vos besoins.

### 4. Utiliser des documents imbriqués pour les relations un-à-plusieurs

Lorsqu'une entité possède une relation un-à-plusieurs avec une autre entité et que :
- Le nombre d'éléments liés est **limité** (pas de croissance infinie)
- Ces éléments sont **toujours consultés ensemble** avec l'entité parent

Alors l'**imbrication** est souvent la meilleure solution.

**Exemple - Utilisateur avec adresses :**

```json
{
  "_id": ObjectId("..."),
  "nom": "Leclerc",
  "prenom": "Thomas",
  "email": "thomas.leclerc@example.com",
  "adresses": [
    {
      "type": "domicile",
      "rue": "25 avenue des Champs",
      "ville": "Lyon",
      "codePostal": "69001",
      "principale": true
    },
    {
      "type": "livraison",
      "rue": "10 rue du Commerce",
      "ville": "Lyon",
      "codePostal": "69002",
      "principale": false
    }
  ]
}
```

### 5. Utiliser des références pour les relations complexes

Lorsque :
- Les données liées sont **volumineuses** ou **nombreuses**
- Les données liées sont **consultées indépendamment**
- Plusieurs entités partagent la même référence

Alors utilisez des **références** (identifiants) entre documents, similaire aux clés étrangères SQL.

**Exemple - Articles de blog et auteurs :**

**Collection "auteurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nom": "Martin",
  "prenom": "Jean",
  "email": "jean.martin@example.com",
  "bio": "Développeur passionné..."
}
```

**Collection "articles" :**
```json
{
  "_id": ObjectId("507f191e810c19729de860ea"),
  "titre": "MongoDB avancé",
  "contenu": "Dans cet article...",
  "auteurId": ObjectId("507f1f77bcf86cd799439011"),
  "datePublication": ISODate("2024-01-20")
}
```

L'application devra faire deux requêtes pour obtenir l'article et les détails de l'auteur, mais cela offre plus de flexibilité.

---

## Flexibilité du schéma

### Schéma dynamique

L'un des grands avantages de MongoDB est son **schéma flexible**. Les documents d'une même collection peuvent avoir des structures différentes.

**Exemple - Produits avec attributs variables :**

```json
// Produit 1 : Livre
{
  "_id": ObjectId("..."),
  "type": "livre",
  "nom": "Guide MongoDB",
  "prix": 29.99,
  "auteur": "Jean Martin",
  "isbn": "978-3-16-148410-0",
  "nombrePages": 350
}

// Produit 2 : Ordinateur portable
{
  "_id": ObjectId("..."),
  "type": "electronique",
  "nom": "Laptop Pro 15",
  "prix": 1299.99,
  "marque": "TechBrand",
  "processeur": "Intel i7",
  "ram": "16GB",
  "stockage": "512GB SSD"
}
```

Les deux documents sont dans la même collection mais ont des champs différents adaptés à leur type de produit.

### Évolution du schéma

Le schéma peut **évoluer progressivement** sans nécessiter de migrations complexes. Vous pouvez ajouter de nouveaux champs aux nouveaux documents sans modifier les anciens.

**Exemple - Ajout d'un champ :**

```json
// Ancien document
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "email": "dupont@example.com"
}

// Nouveau document avec champ supplémentaire
{
  "_id": ObjectId("..."),
  "nom": "Martin",
  "email": "martin@example.com",
  "newsletter": true  // Nouveau champ
}
```

Votre application peut gérer les deux formats simultanément.

---

## Considérations importantes

### 1. Limite de taille des documents

Un document MongoDB ne peut **pas dépasser 16 Mo**. Cette limite est généralement suffisante pour la plupart des cas d'usage, mais elle influence vos décisions de modélisation.

**Implications :**
- Évitez d'imbriquer un nombre illimité d'éléments (ex : commentaires d'un article viral)
- Pour des données volumineuses (fichiers, images), utilisez GridFS ou des services de stockage externes
- Faites attention à la croissance potentielle des tableaux imbriqués

### 2. Performances de lecture vs écriture

**Dénormalisation :**
- ✅ Améliore les **performances de lecture** (moins de requêtes)
- ⚠️ Peut ralentir les **écritures** (mises à jour multiples si données dupliquées)

**Normalisation (références) :**
- ✅ Facilite les **mises à jour** (données stockées une seule fois)
- ⚠️ Nécessite plus de **requêtes de lecture** (jointures applicatives)

Choisissez en fonction de votre cas d'usage : votre application lit-elle plus qu'elle n'écrit, ou l'inverse ?

### 3. Requêtes fréquentes

Votre modèle doit optimiser les **opérations les plus courantes** de votre application.

**Questions à se poser :**
- Quelles données sont affichées ensemble sur une page ?
- Quelles recherches sont les plus fréquentes ?
- Quelles données sont rarement consultées séparément ?

### 4. Cohérence des données

Réfléchissez aux **garanties de cohérence** nécessaires :
- Les données dupliquées peuvent-elles être temporairement désynchronisées ?
- Quelles opérations doivent être atomiques ?
- Avez-vous besoin de transactions multi-documents (disponibles depuis MongoDB 4.0) ?

---

## Méthodologie de modélisation

Voici une approche étape par étape pour modéliser vos données :

### Étape 1 : Identifier les entités

Listez les principales entités de votre application (utilisateurs, produits, commandes, etc.).

### Étape 2 : Identifier les relations

Déterminez comment ces entités sont liées :
- Un-à-un (un utilisateur a un profil)
- Un-à-plusieurs (un auteur a plusieurs articles)
- Plusieurs-à-plusieurs (des étudiants suivent plusieurs cours)

### Étape 3 : Analyser les patterns d'accès

Documentez les requêtes principales :
- Afficher un utilisateur avec toutes ses commandes
- Lister les articles d'un auteur
- Rechercher des produits par catégorie

### Étape 4 : Choisir entre imbrication et références

Pour chaque relation, décidez :
- **Imbriquer** si les données sont toujours consultées ensemble et en nombre limité
- **Référencer** si les données sont volumineuses, consultées séparément ou partagées

### Étape 5 : Valider avec des cas d'usage réels

Simulez les opérations principales de votre application pour vérifier que votre modèle est performant.

### Étape 6 : Itérer et ajuster

La modélisation n'est pas figée. Ajustez votre schéma en fonction des retours d'expérience et des évolutions de l'application.

---

## Exemples comparatifs

### Exemple 1 : Blog

**Modèle relationnel (SQL) :**
- Table `auteurs` (id, nom, email)
- Table `articles` (id, titre, contenu, auteur_id)
- Table `commentaires` (id, texte, article_id, auteur_id)

**Modèle MongoDB (imbrication) :**

```json
{
  "_id": ObjectId("..."),
  "titre": "MongoDB pour débutants",
  "contenu": "Dans cet article...",
  "auteur": {
    "nom": "Sophie Martin",
    "email": "sophie@example.com"
  },
  "datePublication": ISODate("2024-01-15"),
  "commentaires": [
    {
      "auteur": "Jean D.",
      "texte": "Super article !",
      "date": ISODate("2024-01-16")
    }
  ],
  "tags": ["mongodb", "nosql", "database"]
}
```

**Avantage :** Une seule requête pour obtenir l'article complet avec ses commentaires.

### Exemple 2 : E-commerce

**Modèle relationnel (SQL) :**
- Table `clients`
- Table `produits`
- Table `commandes`
- Table `lignes_commandes` (table de jointure)

**Modèle MongoDB (hybride) :**

```json
// Collection "produits"
{
  "_id": ObjectId("..."),
  "nom": "Smartphone XYZ",
  "prix": 599.99,
  "description": "...",
  "stock": 50
}

// Collection "commandes"
{
  "_id": ObjectId("..."),
  "numeroCommande": "CMD-2024-001",
  "client": {
    "id": ObjectId("..."),
    "nom": "Durand",
    "email": "durand@example.com"
  },
  "articles": [
    {
      "produitId": ObjectId("..."),
      "nom": "Smartphone XYZ",  // Dénormalisé
      "prix": 599.99,            // Dénormalisé
      "quantite": 1
    }
  ],
  "montantTotal": 599.99,
  "statut": "livree",
  "dateCommande": ISODate("2024-01-15")
}
```

**Approche :**
- Les produits sont dans une collection séparée (normalisé)
- Les commandes dénormalisent certaines infos produit pour éviter les jointures
- Le prix au moment de la commande est conservé (même si le prix produit change)

---

## Récapitulatif des principes

| Principe | Description |
|----------|-------------|
| **Orientation application** | Modéliser selon les besoins et patterns d'accès de l'application |
| **Atomicité** | Regrouper les données modifiées ensemble dans le même document |
| **Dénormalisation** | Accepter la duplication pour améliorer les performances de lecture |
| **Imbrication** | Utiliser pour les relations un-à-plusieurs avec un nombre limité d'éléments |
| **Références** | Utiliser pour les données volumineuses ou consultées indépendamment |
| **Flexibilité** | Tirer parti du schéma flexible pour évoluer progressivement |
| **Limite 16 Mo** | Garder à l'esprit la taille maximale des documents |

---

## Conclusion

La modélisation orientée document dans MongoDB représente un changement de paradigme par rapport aux bases de données relationnelles. Au lieu de normaliser systématiquement, vous devez **penser en termes de documents autonomes** qui reflètent la structure naturelle de vos données et les besoins de votre application.

Les principes clés à retenir :
- Modélisez pour votre application, pas pour une structure abstraite
- Privilégiez l'imbrication pour les données toujours consultées ensemble
- Utilisez des références pour les relations complexes ou les données volumineuses
- Acceptez la dénormalisation comme un outil d'optimisation
- Profitez de la flexibilité du schéma pour faire évoluer votre modèle

La modélisation MongoDB est un **art de l'équilibre** entre performance, cohérence et simplicité. Il n'y a pas de solution universelle : chaque application a ses propres besoins qui guideront vos choix de conception.

Dans les sections suivantes, nous explorerons en détail les différents types de relations et les patterns de modélisation avancés qui vous aideront à affiner votre maîtrise de la modélisation orientée document.

---

**Points clés à retenir :**

- ✅ Le document est l'unité fondamentale de MongoDB
- ✅ Modéliser selon les besoins de l'application, pas selon une structure abstraite
- ✅ L'atomicité est garantie au niveau du document
- ✅ La dénormalisation est souvent préférable pour les performances
- ✅ Choisir entre imbrication et références selon le contexte
- ✅ Le schéma est flexible et peut évoluer
- ✅ Attention à la limite de 16 Mo par document

---


⏭️ [Documents imbriqués vs Références](/04-modelisation-des-donnees/02-imbriques-vs-references.md)
