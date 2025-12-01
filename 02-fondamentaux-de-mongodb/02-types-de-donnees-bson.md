🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.2 Types de Données BSON

## Introduction

Dans la section précédente, nous avons découvert la structure des documents BSON. Maintenant, plongeons dans les **types de données** que vous pouvez utiliser dans vos documents MongoDB.

BSON (Binary JSON) offre une palette riche de types de données, bien plus étendue que JSON standard. Cette variété vous permet de modéliser vos données de manière précise et efficace.

> **💡 Pourquoi c'est important ?** Choisir le bon type de données impacte directement les performances, la précision et la facilité d'utilisation de votre base de données.

---

## Vue d'Ensemble des Types BSON

MongoDB supporte une vingtaine de types de données différents. Voici un tableau récapitulatif :

| Catégorie | Types | Usage Principal |
|-----------|-------|-----------------|
| **Texte** | String | Chaînes de caractères |
| **Nombres** | Int32, Int64, Double, Decimal128 | Valeurs numériques |
| **Booléens** | Boolean | Vrai/Faux |
| **Dates** | Date, Timestamp | Dates et heures |
| **Identifiants** | ObjectId | Identifiants uniques |
| **Binaires** | Binary, BinData | Données binaires |
| **Null** | Null | Valeurs nulles/absentes |
| **Collections** | Array, Object (Document) | Listes et sous-documents |
| **Spéciaux** | Regular Expression, JavaScript, MinKey, MaxKey | Usages avancés |

Explorons chaque type en détail !

---

## 1. String (Chaîne de Caractères)

### Description

Le type **String** stocke du texte encodé en UTF-8.

### Syntaxe

```json
{
  "nom": "Dupont",
  "prenom": "Marie",
  "description": "Développeuse passionnée par MongoDB",
  "emoji": "🚀 MongoDB c'est génial !"
}
```

### Caractéristiques

- ✅ **Encodage UTF-8** : Support complet des caractères Unicode (accents, émojis, etc.)
- ✅ **Taille variable** : De 0 caractère à plusieurs mégaoctets
- ✅ **Indexable** : Peut être indexé pour des recherches rapides
- ✅ **Recherche textuelle** : Support des recherches full-text

### Cas d'Usage

```json
{
  "_id": 1,
  "titre": "Guide MongoDB",
  "auteur": "Jean Développeur",
  "description": "Un guide complet pour apprendre MongoDB",
  "langue": "fr",
  "tags": "base de données, NoSQL, tutoriel"
}
```

### Bonnes Pratiques

- Utilisez des chaînes pour tout texte lisible par l'humain
- Pour les identifiants, préférez ObjectId ou UUID
- Pour les énumérations, les chaînes sont appropriées mais pensez à valider

---

## 2. Les Types Numériques

MongoDB offre plusieurs types numériques selon vos besoins de précision et de plage.

### 2.1 Int32 (Entier 32 bits)

**Plage :** -2,147,483,648 à 2,147,483,647

```json
{
  "age": 28,
  "nombreVues": 1500,
  "stock": 42
}
```

**Quand l'utiliser :**
- Compteurs
- Ages
- Quantités
- Nombres entiers "normaux"

### 2.2 Int64 (Long - Entier 64 bits)

**Plage :** -9,223,372,036,854,775,808 à 9,223,372,036,854,775,807

```json
{
  "population": 7800000000,
  "distanceEnMetres": 384400000,
  "microsecondes": 1609459200000000
}
```

**Quand l'utiliser :**
- Très grands nombres entiers
- Timestamps en millisecondes
- Identifiants numériques longs
- Données scientifiques

### 2.3 Double (Nombre à Virgule Flottante)

**Précision :** 15-17 chiffres significatifs (64 bits IEEE 754)

```json
{
  "prix": 29.99,
  "temperature": -3.5,
  "coordonnees": {
    "latitude": 48.8566,
    "longitude": 2.3522
  },
  "ratio": 0.75
}
```

**⚠️ Attention : Précision limitée**
```javascript
// Problème de précision des flottants
0.1 + 0.2 = 0.30000000000000004
```

**Quand l'utiliser :**
- Mesures physiques
- Coordonnées GPS
- Statistiques
- Calculs scientifiques (sans besoin de précision exacte)

### 2.4 Decimal128 (Décimal Haute Précision)

**Précision :** 34 chiffres décimaux significatifs

```json
{
  "prixPrecis": NumberDecimal("19.99"),
  "montantFacture": NumberDecimal("1234.56"),
  "tauxInteret": NumberDecimal("0.0325"),
  "crypto": NumberDecimal("0.00000123456789")
}
```

**Quand l'utiliser :**
- 💰 **Applications financières** (prix, montants)
- 💵 **Comptabilité** (factures, transactions)
- 📊 **Calculs monétaires** (éviter les erreurs d'arrondi)
- 🪙 **Crypto-monnaies** (précision extrême)

**Exemple Comparatif :**

```javascript
// ❌ Problème avec Double
db.produits.insertOne({
  nom: "Livre",
  prix: 19.99  // Stocké comme Double
});
// Risque : 19.990000000000002

// ✅ Solution avec Decimal128
db.produits.insertOne({
  nom: "Livre",
  prix: NumberDecimal("19.99")  // Précision exacte
});
```

### Tableau Comparatif des Types Numériques

| Type | Taille | Précision | Usage Recommandé |
|------|--------|-----------|------------------|
| **Int32** | 4 octets | Entier exact | Compteurs, ages, petits nombres |
| **Int64** | 8 octets | Entier exact | Très grands nombres, timestamps |
| **Double** | 8 octets | ~15 chiffres | Mesures, coordonnées, scientifique |
| **Decimal128** | 16 octets | 34 chiffres | Finance, comptabilité, monétaire |

---

## 3. Boolean (Booléen)

### Description

Représente une valeur **vraie** (`true`) ou **fausse** (`false`).

### Syntaxe

```json
{
  "actif": true,
  "estVerifie": false,
  "accepteNewsletters": true,
  "premium": false
}
```

### Cas d'Usage

```json
{
  "_id": 1,
  "email": "user@example.com",
  "emailVerifie": true,
  "compteActif": true,
  "abonnementPremium": false,
  "notifications": {
    "email": true,
    "sms": false,
    "push": true
  }
}
```

### Bonnes Pratiques

- Utilisez des noms de champs explicites : `estActif` plutôt que `actif` pour plus de clarté
- Pour les états avec plus de 2 valeurs, utilisez plutôt un String (enum)

```json
// ❌ Mauvais : trop de booléens
{
  "statutPending": false,
  "statutApproved": true,
  "statutRejected": false
}

// ✅ Bon : utiliser une énumération
{
  "statut": "approved"  // "pending" | "approved" | "rejected"
}
```

---

## 4. Types de Dates et Temps

### 4.1 Date (Type Date BSON)

Le type **Date** stocke les dates et heures avec une précision à la milliseconde.

**Format interne :** Nombre de millisecondes depuis l'epoch Unix (1er janvier 1970 00:00:00 UTC)

### Syntaxe dans le Shell

```javascript
// Création avec ISODate()
{
  "dateInscription": ISODate("2024-01-15T10:30:00Z"),
  "dernierLogin": ISODate("2024-12-01T08:45:30.123Z"),
  "dateNaissance": ISODate("1990-05-20T00:00:00Z")
}

// Création avec new Date()
{
  "dateCreation": new Date(),  // Date actuelle
  "expiration": new Date("2025-12-31")
}
```

### Exemples Concrets

```json
{
  "_id": 1,
  "utilisateur": "alice@example.com",
  "evenements": {
    "creation": ISODate("2024-01-15T10:30:00Z"),
    "derniereModification": ISODate("2024-11-20T14:22:00Z"),
    "dernierLogin": ISODate("2024-12-01T08:15:00Z")
  },
  "abonnement": {
    "dateDebut": ISODate("2024-01-15T00:00:00Z"),
    "dateFin": ISODate("2025-01-15T00:00:00Z")
  }
}
```

### Opérations Courantes

```javascript
// Date actuelle
db.logs.insertOne({
  message: "Utilisateur connecté",
  timestamp: new Date()
});

// Recherche par date
db.utilisateurs.find({
  dateInscription: {
    $gte: ISODate("2024-01-01"),
    $lt: ISODate("2025-01-01")
  }
});

// Extraction de parties de date (dans une agrégation)
db.commandes.aggregate([
  {
    $project: {
      annee: { $year: "$dateCommande" },
      mois: { $month: "$dateCommande" },
      jour: { $dayOfMonth: "$dateCommande" }
    }
  }
]);
```

### 4.2 Timestamp (Type Timestamp BSON)

**⚠️ Usage interne MongoDB** - Principalement utilisé par MongoDB lui-même pour la réplication.

```json
{
  "ts": Timestamp(1638360000, 1)
}
```

**Différences Date vs Timestamp :**

| Aspect | Date | Timestamp |
|--------|------|-----------|
| **Usage** | Applications | Réplication interne MongoDB |
| **Précision** | Millisecondes | Secondes + compteur |
| **Recommandation** | ✅ Utilisez pour vos dates | ❌ Réservé à MongoDB |

> **💡 Conseil :** Utilisez toujours le type **Date** pour vos applications. Laissez Timestamp à MongoDB pour ses besoins internes.

---

## 5. ObjectId

### Description

**ObjectId** est le type par défaut pour le champ `_id`. C'est un identifiant unique de **12 octets**.

### Structure d'un ObjectId

```
ObjectId("507f1f77bcf86cd799439011")
         └─────┬─────┘└┬┘└──┬──┘└┬┘
               │       │    │    └─ 3 octets : Compteur
               │       │    └────── 5 octets : Valeur aléatoire
               └───────┴─────────── 4 octets : Timestamp Unix
```

**Composition :**
1. **4 octets** : Timestamp (secondes depuis epoch Unix)
2. **5 octets** : Valeur aléatoire (identifie la machine/process)
3. **3 octets** : Compteur incrémental

### Avantages

- ✅ **Unique globalement** : Aucun risque de collision même dans des systèmes distribués
- ✅ **Tri chronologique** : Les ObjectId sont naturellement triés par date de création
- ✅ **Pas de coordination nécessaire** : Génération sans base de données
- ✅ **Contient le timestamp** : Date de création incluse

### Exemples

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "titre": "Article MongoDB"
}

// Avec références
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "auteur": ObjectId("507f191e810c19729de860ea"),
  "articles": [
    ObjectId("507f1f77bcf86cd799439012"),
    ObjectId("507f1f77bcf86cd799439013")
  ]
}
```

### Extraction du Timestamp

```javascript
// Dans mongosh
var objectId = ObjectId("507f1f77bcf86cd799439011");
objectId.getTimestamp();
// Retourne : ISODate("2012-10-17T20:46:22Z")
```

### Génération Manuelle

```javascript
// Génération d'un nouvel ObjectId
db.articles.insertOne({
  _id: ObjectId(),  // Nouveau ObjectId
  titre: "Mon article"
});

// Utilisation d'un ObjectId spécifique (rare)
db.articles.insertOne({
  _id: ObjectId("507f1f77bcf86cd799439011"),
  titre: "Article avec ID spécifique"
});
```

---

## 6. Binary (Données Binaires)

### Description

Le type **Binary** (ou **BinData**) permet de stocker des données binaires brutes.

### Cas d'Usage

- 🖼️ Images (petites uniquement, < 16 Mo)
- 📄 Fichiers PDF
- 🔐 Hash cryptographiques
- 🆔 UUID/GUID
- 🎵 Fichiers audio courts

### Syntaxe

```javascript
// Dans mongosh
db.fichiers.insertOne({
  nom: "logo.png",
  type: "image/png",
  donnees: BinData(0, "iVBORw0KGgoAAAANSUhEUgAA..."),
  taille: 2048
});
```

### Sous-types Binary

MongoDB définit plusieurs sous-types :

| Sous-type | Code | Description |
|-----------|------|-------------|
| **Generic** | 0 | Binaire générique |
| **Function** | 1 | Code fonction |
| **Binary (Old)** | 2 | Obsolète |
| **UUID (Old)** | 3 | UUID ancien format |
| **UUID** | 4 | UUID standard |
| **MD5** | 5 | Hash MD5 |
| **Encrypted** | 6 | Données chiffrées |
| **User Defined** | 128-255 | Défini par l'utilisateur |

### Exemple avec UUID

```javascript
// Stockage d'un UUID
db.utilisateurs.insertOne({
  _id: UUID("550e8400-e29b-41d4-a716-446655440000"),
  email: "user@example.com"
});
```

### ⚠️ Limitations

- Maximum 16 Mo par document (limitation BSON)
- Pour des fichiers volumineux, utilisez **GridFS**

---

## 7. Null

### Description

Représente l'**absence de valeur** ou une **valeur nulle**.

### Syntaxe

```json
{
  "nom": "Dupont",
  "prenom": "Marie",
  "telephone": null,
  "dateNaissance": null
}
```

### Null vs Champ Absent

```json
// Document 1 : Champ null
{
  "_id": 1,
  "nom": "Alice",
  "telephone": null
}

// Document 2 : Champ absent
{
  "_id": 2,
  "nom": "Bob"
}
```

**Différences importantes :**

| Aspect | Valeur `null` | Champ absent |
|--------|---------------|--------------|
| **Existe** | Oui, avec valeur null | Non |
| **Requête `{ telephone: null }`** | ✅ Trouvé | ✅ Trouvé |
| **Requête `{ telephone: { $exists: false } }`** | ❌ Non trouvé | ✅ Trouvé |
| **Taille du document** | Légèrement plus grand | Plus petit |

### Exemples de Requêtes

```javascript
// Trouve les documents où telephone est null OU absent
db.utilisateurs.find({ telephone: null });

// Trouve uniquement les documents où telephone existe et vaut null
db.utilisateurs.find({
  telephone: { $type: "null" }
});

// Trouve les documents où telephone est absent
db.utilisateurs.find({
  telephone: { $exists: false }
});

// Trouve les documents où telephone existe (null ou non-null)
db.utilisateurs.find({
  telephone: { $exists: true }
});
```

---

## 8. Array (Tableau)

### Description

Les **tableaux** stockent des listes ordonnées de valeurs.

### Syntaxe

```json
{
  "prenoms": ["Marie", "Sophie", "Claire"],
  "notes": [15, 18, 12, 20],
  "competences": ["JavaScript", "Python", "MongoDB"],
  "vide": []
}
```

### Tableaux Hétérogènes

MongoDB permet des tableaux avec différents types :

```json
{
  "mixte": [
    "texte",
    42,
    true,
    null,
    { "objet": "imbriqué" },
    ["tableau", "imbriqué"]
  ]
}
```

### Tableaux de Documents

```json
{
  "_id": 1,
  "produit": "Laptop",
  "avis": [
    {
      "utilisateur": "Alice",
      "note": 5,
      "commentaire": "Excellent produit !",
      "date": ISODate("2024-11-15T10:00:00Z")
    },
    {
      "utilisateur": "Bob",
      "note": 4,
      "commentaire": "Très bien",
      "date": ISODate("2024-11-20T14:30:00Z")
    }
  ]
}
```

### Indexation des Tableaux

MongoDB peut indexer les tableaux :

```javascript
// Index sur un tableau
db.produits.createIndex({ "tags": 1 });

// Recherche dans un tableau
db.produits.find({ "tags": "mongodb" });

// Recherche dans un tableau de documents
db.produits.find({ "avis.note": 5 });
```

### Opérations sur les Tableaux

```javascript
// Ajouter un élément
db.utilisateurs.updateOne(
  { _id: 1 },
  { $push: { competences: "Docker" } }
);

// Ajouter plusieurs éléments
db.utilisateurs.updateOne(
  { _id: 1 },
  { $push: { competences: { $each: ["Kubernetes", "AWS"] } } }
);

// Retirer un élément
db.utilisateurs.updateOne(
  { _id: 1 },
  { $pull: { competences: "Docker" } }
);
```

---

## 9. Object (Document Imbriqué)

### Description

Un **document imbriqué** est un objet contenu dans un autre document.

### Syntaxe

```json
{
  "_id": 1,
  "nom": "Dupont",
  "adresse": {
    "rue": "123 Rue de la Paix",
    "ville": "Paris",
    "codePostal": "75001",
    "pays": "France"
  },
  "contact": {
    "email": "dupont@example.com",
    "telephone": {
      "fixe": "01 23 45 67 89",
      "mobile": "06 12 34 56 78"
    }
  }
}
```

### Imbrication Profonde

```json
{
  "entreprise": {
    "nom": "TechCorp",
    "siege": {
      "adresse": {
        "rue": "456 Avenue Innovation",
        "ville": "Lyon",
        "departement": {
          "code": "69",
          "nom": "Rhône"
        }
      }
    }
  }
}
```

**⚠️ Attention :** Évitez les imbrications trop profondes (> 4 niveaux) pour la lisibilité.

### Accès aux Champs Imbriqués

```javascript
// Recherche avec dot notation
db.utilisateurs.find({ "adresse.ville": "Paris" });

// Recherche multi-niveaux
db.entreprises.find({
  "siege.adresse.departement.code": "69"
});

// Projection de champs imbriqués
db.utilisateurs.find(
  {},
  { "contact.email": 1, "adresse.ville": 1 }
);
```

---

## 10. Regular Expression (Expression Régulière)

### Description

Stocke des **expressions régulières** pour des recherches de patterns.

### Syntaxe

```json
{
  "pattern": /^[A-Z]/,
  "emailPattern": /^[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$/i
}
```

### Utilisation dans les Requêtes

```javascript
// Recherche avec regex
db.utilisateurs.find({
  email: /^alice/i  // Commence par "alice" (insensible à la casse)
});

// Regex avec $regex
db.produits.find({
  nom: { $regex: "laptop", $options: "i" }
});

// Pattern complexe
db.articles.find({
  titre: /^MongoDB.*guide$/i
});
```

### Options des Regex

| Option | Signification |
|--------|---------------|
| **i** | Insensible à la casse |
| **m** | Multi-lignes |
| **x** | Ignore les espaces |
| **s** | Dot (.) correspond aussi aux retours à la ligne |

---

## 11. Types Spéciaux

### 11.1 MinKey et MaxKey

Types spéciaux utilisés pour les comparaisons et le sharding.

```javascript
// MinKey : Plus petite valeur possible
{ valeur: MinKey }

// MaxKey : Plus grande valeur possible
{ valeur: MaxKey }
```

**Usage :** Principalement dans les configurations de sharding.

### 11.2 JavaScript / JavaScript with Scope

Stockage de code JavaScript (rarement utilisé).

```javascript
{
  fonction: function() { return this.nom; }
}
```

**⚠️ Attention :** Évitez d'utiliser ce type pour des raisons de sécurité.

### 11.3 Undefined (Obsolète)

❌ **Déprécié** - N'utilisez plus ce type. Utilisez `null` à la place.

---

## Déterminer le Type d'un Champ

### Opérateur $type

```javascript
// Trouver les documents où 'age' est un nombre
db.utilisateurs.find({
  age: { $type: "number" }
});

// Trouver les documents où 'email' est une chaîne
db.utilisateurs.find({
  email: { $type: "string" }
});

// Trouver les documents où 'tags' est un tableau
db.produits.find({
  tags: { $type: "array" }
});
```

### Codes des Types BSON

| Type | Alias | Code Numérique |
|------|-------|----------------|
| Double | "double" | 1 |
| String | "string" | 2 |
| Object | "object" | 3 |
| Array | "array" | 4 |
| Binary | "binData" | 5 |
| ObjectId | "objectId" | 7 |
| Boolean | "bool" | 8 |
| Date | "date" | 9 |
| Null | "null" | 10 |
| Regular Expression | "regex" | 11 |
| JavaScript | "javascript" | 13 |
| Int32 | "int" | 16 |
| Timestamp | "timestamp" | 17 |
| Int64 | "long" | 18 |
| Decimal128 | "decimal" | 19 |
| MinKey | "minKey" | -1 |
| MaxKey | "maxKey" | 127 |

---

## Conversion de Types

### Opérateurs de Conversion (Agrégation)

```javascript
// Convertir en entier
db.produits.aggregate([
  {
    $project: {
      prixEntier: { $toInt: "$prix" }
    }
  }
]);

// Convertir en chaîne
db.utilisateurs.aggregate([
  {
    $project: {
      ageTexte: { $toString: "$age" }
    }
  }
]);

// Convertir en date
db.logs.aggregate([
  {
    $project: {
      dateConvertie: { $toDate: "$timestamp" }
    }
  }
]);
```

### Opérateurs de Conversion Disponibles

- `$toInt` : Convertir en Int32
- `$toLong` : Convertir en Int64
- `$toDouble` : Convertir en Double
- `$toDecimal` : Convertir en Decimal128
- `$toString` : Convertir en String
- `$toDate` : Convertir en Date
- `$toBool` : Convertir en Boolean
- `$toObjectId` : Convertir en ObjectId

---

## Bonnes Pratiques par Type

### 🔤 Strings
- ✅ Utilisez pour tout texte lisible
- ✅ Indexez les champs fréquemment recherchés
- ❌ N'utilisez pas pour des IDs (préférez ObjectId)

### 🔢 Nombres
- ✅ Int32/Int64 pour les entiers
- ✅ Decimal128 pour la finance
- ❌ Évitez Double pour l'argent

### 📅 Dates
- ✅ Utilisez toujours le type Date BSON
- ✅ Stockez en UTC
- ❌ N'utilisez pas de chaînes pour les dates

### 🆔 ObjectId
- ✅ Parfait pour les `_id`
- ✅ Utile pour les références
- ❌ Ne le modifiez jamais une fois créé

### 📦 Arrays
- ✅ Parfaits pour les listes courtes/moyennes
- ❌ Attention aux tableaux qui grandissent indéfiniment
- ❌ Évitez les tableaux de > 1000 éléments

### 🏢 Documents Imbriqués
- ✅ Parfaits pour les relations 1-à-1
- ✅ Évitez > 3-4 niveaux d'imbrication
- ❌ Ne dupliquez pas excessivement les données

---

## Exemples de Modélisation par Secteur

### E-commerce

```json
{
  "_id": ObjectId("..."),
  "reference": "PROD-12345",
  "nom": "Laptop Pro 15",
  "description": "Ordinateur portable haute performance",
  "prix": NumberDecimal("1299.99"),
  "prixPromo": NumberDecimal("1099.99"),
  "stock": 42,
  "enLigne": true,
  "categories": ["informatique", "laptops", "pro"],
  "specifications": {
    "processeur": "Intel i7",
    "ram": 16,
    "stockage": 512
  },
  "images": [
    "https://cdn.example.com/img1.jpg",
    "https://cdn.example.com/img2.jpg"
  ],
  "notes": [4.5, 5.0, 4.0, 5.0],
  "moyenneNotes": 4.625,
  "dateAjout": ISODate("2024-01-15T10:00:00Z"),
  "derniereMaj": ISODate("2024-11-20T14:30:00Z")
}
```

### Blog

```json
{
  "_id": ObjectId("..."),
  "titre": "Introduction à MongoDB",
  "slug": "introduction-mongodb",
  "contenu": "MongoDB est une base de données NoSQL...",
  "auteur": ObjectId("507f191e810c19729de860ea"),
  "publie": true,
  "vues": 1523,
  "likes": 89,
  "tags": ["mongodb", "database", "nosql", "tutoriel"],
  "categorie": "Tutoriels",
  "datePublication": ISODate("2024-11-01T09:00:00Z"),
  "derniereModification": ISODate("2024-11-15T16:45:00Z"),
  "commentaires": [
    {
      "auteur": "Alice",
      "texte": "Excellent article !",
      "date": ISODate("2024-11-02T10:30:00Z"),
      "likes": 5
    }
  ],
  "metadonnees": {
    "motsCles": ["mongodb", "tutoriel", "debutant"],
    "description": "Guide complet pour débuter avec MongoDB",
    "image": "https://cdn.example.com/og-image.jpg"
  }
}
```

### Application Bancaire

```json
{
  "_id": ObjectId("..."),
  "numeroCompte": "FR76 1234 5678 9012 3456 7890 123",
  "titulaire": ObjectId("507f191e810c19729de860ea"),
  "type": "compte-courant",
  "solde": NumberDecimal("2547.83"),
  "devise": "EUR",
  "actif": true,
  "dateOuverture": ISODate("2020-03-15T00:00:00Z"),
  "plafonds": {
    "carteDebit": NumberDecimal("1000.00"),
    "virementJournalier": NumberDecimal("5000.00")
  },
  "dernieresTransactions": [
    {
      "date": ISODate("2024-11-30T14:23:00Z"),
      "type": "virement",
      "montant": NumberDecimal("-50.00"),
      "libelle": "Paiement facture",
      "soldeApres": NumberDecimal("2547.83")
    }
  ]
}
```

---

## Tableau Récapitulatif : Choisir le Bon Type

| Besoin | Type Recommandé | Exemple |
|--------|-----------------|---------|
| Texte simple | String | `"Paris"` |
| Petit nombre entier | Int32 | `42` |
| Grand nombre entier | Int64 | `9876543210` |
| Nombre décimal approximatif | Double | `3.14159` |
| Montant financier | Decimal128 | `NumberDecimal("99.99")` |
| Vrai/Faux | Boolean | `true` |
| Date/Heure | Date | `ISODate("2024-12-01")` |
| Identifiant unique | ObjectId | `ObjectId("...")` |
| Liste | Array | `["a", "b", "c"]` |
| Sous-structure | Object | `{ "rue": "...", "ville": "..." }` |
| Valeur absente | Null | `null` |
| Données binaires | Binary | `BinData(0, "...")` |
| Pattern de recherche | RegExp | `/^test/i` |

---

## Points Clés à Retenir

### ✅ Essentiel

1. **Types riches** : BSON offre bien plus de types que JSON
2. **Précision financière** : Utilisez Decimal128 pour l'argent
3. **Dates natives** : Type Date pour toutes les dates/heures
4. **ObjectId unique** : Identifiant par défaut, unique et ordonné chronologiquement
5. **Tableaux puissants** : Indexables et requêtables
6. **Documents imbriqués** : Modélisation naturelle des relations

### 🎯 Bonnes Pratiques

- Choisissez le type le plus approprié dès le départ
- Soyez cohérent dans votre base de données
- Documentez vos choix de types
- Validez les types avec des schémas (nous verrons cela plus tard)

### ⚠️ Pièges à Éviter

- ❌ Utiliser Double pour les montants financiers
- ❌ Stocker des dates comme des chaînes
- ❌ Mélanger les types dans le même champ selon les documents
- ❌ Créer des tableaux qui grandissent sans limite

---

## Prochaines Étapes

Maintenant que vous maîtrisez les types de données BSON, vous êtes prêt pour :

➡️ **2.3 Création d'une base de données** : Créer votre première base MongoDB

Cette connaissance des types est fondamentale pour bien modéliser vos données et optimiser les performances !

---

## Ressources Complémentaires

### Documentation Officielle

- [BSON Types - MongoDB Manual](https://docs.mongodb.com/manual/reference/bson-types/)
- [Type Operators - $type](https://docs.mongodb.com/manual/reference/operator/query/type/)
- [Type Conversion Operators](https://docs.mongodb.com/manual/reference/operator/aggregation/#type-conversion-operators)

### Pour Aller Plus Loin

- Compression et stockage des différents types
- Indexation selon les types
- Conversion de types dans les pipelines d'agrégation

---


⏭️ [Création d'une base de données](/02-fondamentaux-de-mongodb/03-creation-base-de-donnees.md)
