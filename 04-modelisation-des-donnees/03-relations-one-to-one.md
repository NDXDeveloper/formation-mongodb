🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 4.3 Relations One-to-One (Un-à-Un)

## Introduction

Une relation **one-to-one** (un-à-un) existe lorsqu'un document d'une collection est associé à **exactement un** document d'une autre collection (ou entité), et vice versa. Ce type de relation est le plus simple à modéliser dans MongoDB, mais le choix de la stratégie de modélisation dépend de vos besoins spécifiques.

Dans ce chapitre, nous allons explorer les différentes façons de modéliser des relations un-à-un dans MongoDB, avec des exemples concrets et des conseils pour choisir la meilleure approche.

---

## Comprendre les relations One-to-One

### Définition

Une relation **one-to-one** signifie qu'une entité A est liée à **une seule** entité B, et cette entité B n'est liée qu'à cette entité A.

**Exemples concrets :**

- **Utilisateur ↔ Profil détaillé** : Chaque utilisateur a un seul profil, et chaque profil appartient à un seul utilisateur
- **Personne ↔ Passeport** : Une personne a un seul passeport valide, et un passeport appartient à une seule personne
- **Employé ↔ Badge d'accès** : Chaque employé a un badge unique
- **Véhicule ↔ Immatriculation** : Un véhicule a une seule plaque d'immatriculation
- **Compte bancaire ↔ Carte de crédit principale** : Un compte peut avoir une carte principale désignée

### Caractéristiques

- **Cardinalité 1:1** : Une occurrence de A correspond à une occurrence de B
- **Bidirectionnalité** : La relation peut être consultée dans les deux sens
- **Optionnalité** : La relation peut être obligatoire ou facultative (ex : un utilisateur peut ne pas avoir de profil détaillé)

---

## Stratégies de modélisation

Il existe trois approches principales pour modéliser une relation one-to-one dans MongoDB :

1. **Imbrication (Embedded Documents)** - Recommandée dans la plupart des cas
2. **Références (References)** - Pour des cas spécifiques
3. **Approche hybride** - Combinaison des deux

---

## 1. Imbrication : Documents imbriqués

### Principe

Stocker les données liées **directement dans le même document** sous forme de sous-document.

### Quand utiliser l'imbrication ?

✅ **Utilisez l'imbrication quand :**

- Les données sont **toujours consultées ensemble**
- Les données liées sont **de taille raisonnable** (quelques Ko)
- Les données liées **n'ont pas de sens** en dehors du parent
- Vous voulez des **performances optimales** en lecture
- Vous avez besoin d'**atomicité** sur les modifications

### Exemple 1 : Utilisateur avec profil

**Scénario :** Un site web où chaque utilisateur a un profil avec des informations supplémentaires.

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nom": "Dupont",
  "prenom": "Marie",
  "email": "marie.dupont@example.com",
  "dateInscription": ISODate("2024-01-10"),
  "profil": {
    "photo": "https://exemple.com/photos/marie.jpg",
    "bio": "Développeuse passionnée par les bases de données NoSQL",
    "dateNaissance": ISODate("1995-03-15"),
    "ville": "Lyon",
    "site": "https://mariedupont.dev",
    "reseauxSociaux": {
      "twitter": "@mariedev",
      "linkedin": "marie-dupont",
      "github": "mariedupont"
    },
    "preferences": {
      "langue": "fr",
      "theme": "sombre",
      "newsletter": true
    }
  },
  "statut": "actif"
}
```

**Avantages de cette approche :**

- ✅ **Une seule requête** pour obtenir l'utilisateur et son profil
- ✅ **Atomicité garantie** : toute modification est cohérente
- ✅ **Performance optimale** : données stockées ensemble physiquement
- ✅ **Simplicité** : pas besoin de gérer des jointures

**Requêtes simples :**

```javascript
// Lire l'utilisateur avec son profil
db.utilisateurs.findOne({ email: "marie.dupont@example.com" })

// Mettre à jour la bio
db.utilisateurs.updateOne(
  { _id: ObjectId("507f1f77bcf86cd799439011") },
  { $set: { "profil.bio": "Nouvelle bio..." } }
)

// Mettre à jour les préférences
db.utilisateurs.updateOne(
  { _id: ObjectId("507f1f77bcf86cd799439011") },
  {
    $set: {
      "profil.preferences.theme": "clair",
      "profil.preferences.newsletter": false
    }
  }
)
```

### Exemple 2 : Produit avec spécifications techniques

**Scénario :** Un catalogue e-commerce où chaque produit a des spécifications techniques.

```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  "nom": "Smartphone XYZ Pro",
  "marque": "TechBrand",
  "prix": 899.99,
  "categorie": "Électronique",
  "stock": 45,
  "specifications": {
    "ecran": {
      "taille": "6.7 pouces",
      "resolution": "2778 x 1284",
      "technologie": "OLED"
    },
    "processeur": "Octa-core 3.2 GHz",
    "memoire": {
      "ram": "8 GB",
      "stockage": "256 GB"
    },
    "appareilPhoto": {
      "principal": "48 MP",
      "frontal": "12 MP",
      "video": "4K à 60 fps"
    },
    "batterie": "4500 mAh",
    "systeme": "Android 14",
    "dimensions": {
      "hauteur": "160.8 mm",
      "largeur": "78.1 mm",
      "epaisseur": "7.65 mm",
      "poids": "203 g"
    },
    "connectivite": ["5G", "WiFi 6", "Bluetooth 5.3", "NFC"]
  },
  "garantie": "2 ans",
  "dateAjout": ISODate("2024-01-15")
}
```

**Avantages :**

- ✅ Toutes les informations produit en une seule requête
- ✅ Facile à afficher sur une page produit
- ✅ Les spécifications font partie intégrante du produit

### Exemple 3 : Employé avec informations de paie

**Scénario :** Système RH où chaque employé a des informations de paie sensibles.

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439012"),
  "matricule": "EMP-2024-001",
  "nom": "Martin",
  "prenom": "Jean",
  "email": "jean.martin@entreprise.com",
  "poste": "Développeur Senior",
  "departement": "IT",
  "dateEmbauche": ISODate("2020-03-01"),
  "informationsPaie": {
    "salaireBrut": 4500,
    "devise": "EUR",
    "frequence": "mensuel",
    "iban": "FR76 1234 5678 9012 3456 7890 123",
    "numeroSecuriteSociale": "1 85 03 75 116 123 45",
    "tauxImposition": 30,
    "avantages": {
      "ticketsRestaurant": true,
      "mutuelle": "Premium",
      "transport": "50%"
    }
  },
  "statut": "actif"
}
```

**Note de sécurité :** Les informations sensibles doivent être protégées par des mécanismes de sécurité appropriés (chiffrement, contrôle d'accès).

---

## 2. Références : Documents séparés

### Principe

Stocker les données liées dans des **collections séparées** et utiliser des références (identifiants) pour les lier.

### Quand utiliser les références ?

✅ **Utilisez les références quand :**

- Les données liées sont **volumineuses** (plusieurs centaines de Ko ou plus)
- Les données sont **consultées indépendamment** du parent
- Vous avez besoin de **permissions différentes** sur chaque entité
- Les données liées sont **optionnelles** et rarement présentes
- Vous voulez **séparer les préoccupations** (separation of concerns)
- Les données sont **sensibles** et nécessitent une isolation

### Exemple 1 : Utilisateur et curriculum vitae

**Scénario :** Plateforme d'emploi où les CV peuvent être très détaillés et volumineux.

**Collection "utilisateurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "nom": "Dupont",
  "prenom": "Sophie",
  "email": "sophie.dupont@example.com",
  "telephone": "+33 6 12 34 56 78",
  "cvId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),  // ← Référence
  "dateInscription": ISODate("2024-01-10"),
  "statut": "actif"
}
```

**Collection "cv" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d0"),
  "utilisateurId": ObjectId("507f1f77bcf86cd799439011"),  // ← Référence inverse
  "titre": "Développeuse Full Stack Senior",
  "resume": "Plus de 10 ans d'expérience...",
  "experiences": [
    {
      "entreprise": "TechCorp",
      "poste": "Lead Developer",
      "debut": ISODate("2020-01-01"),
      "fin": ISODate("2024-01-01"),
      "description": "Développement d'applications web...",
      "technologies": ["React", "Node.js", "MongoDB"]
    },
    {
      "entreprise": "StartupXYZ",
      "poste": "Senior Developer",
      "debut": ISODate("2018-06-01"),
      "fin": ISODate("2019-12-31"),
      "description": "Architecture microservices...",
      "technologies": ["Java", "Spring Boot", "PostgreSQL"]
    }
    // ... potentiellement beaucoup d'expériences
  ],
  "formations": [
    {
      "etablissement": "Université de Lyon",
      "diplome": "Master Informatique",
      "annee": 2013
    }
    // ... formations
  ],
  "competences": [
    { "nom": "JavaScript", "niveau": "Expert", "anneesExperience": 10 },
    { "nom": "MongoDB", "niveau": "Avancé", "anneesExperience": 5 },
    // ... beaucoup de compétences
  ],
  "langues": [
    { "langue": "Français", "niveau": "Natif" },
    { "langue": "Anglais", "niveau": "Courant" }
  ],
  "projets": [
    // ... projets détaillés
  ],
  "certifications": [
    // ... certifications
  ],
  "dateModification": ISODate("2024-01-15")
}
```

**Avantages de cette approche :**

- ✅ Le document utilisateur reste **léger et rapide** à charger
- ✅ Le CV peut être très **détaillé sans impacter** les autres requêtes
- ✅ **Permissions séparées** : tout le monde peut voir le profil, mais le CV complet est privé
- ✅ Le CV peut être **chargé à la demande** (lazy loading)

**Requêtes :**

```javascript
// 1. Charger l'utilisateur (rapide)
const utilisateur = db.utilisateurs.findOne({
  email: "sophie.dupont@example.com"
})

// 2. Charger le CV seulement si nécessaire (ex : l'utilisateur clique sur "Voir CV")
if (utilisateur.cvId) {
  const cv = db.cv.findOne({ _id: utilisateur.cvId })
}

// Ou avec $lookup (jointure)
db.utilisateurs.aggregate([
  { $match: { email: "sophie.dupont@example.com" } },
  {
    $lookup: {
      from: "cv",
      localField: "cvId",
      foreignField: "_id",
      as: "cv"
    }
  },
  { $unwind: "$cv" }  // Convertir le tableau en objet
])
```

### Exemple 2 : Voiture et historique d'entretien

**Scénario :** Application de gestion de flotte où l'historique d'entretien peut devenir volumineux.

**Collection "vehicules" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439013"),
  "immatriculation": "AB-123-CD",
  "marque": "Renault",
  "modele": "Megane",
  "annee": 2022,
  "vin": "VF1BZFUE556789012",
  "couleur": "Bleu",
  "kilometrage": 35000,
  "historiqueEntretienId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d1"),
  "statut": "actif",
  "dateAchat": ISODate("2022-05-10")
}
```

**Collection "historiquesEntretien" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d1"),
  "vehiculeId": ObjectId("507f1f77bcf86cd799439013"),
  "interventions": [
    {
      "date": ISODate("2024-01-10"),
      "type": "Révision",
      "kilometrage": 35000,
      "cout": 250.00,
      "garage": "Garage Dupont",
      "details": "Vidange, changement filtres, contrôle freins",
      "pieceChangees": ["Huile moteur", "Filtre à huile", "Filtre à air"]
    },
    {
      "date": ISODate("2023-07-15"),
      "type": "Révision",
      "kilometrage": 25000,
      "cout": 200.00,
      "garage": "Garage Dupont",
      "details": "Vidange et contrôle général"
    }
    // ... historique complet depuis l'achat
  ],
  "prochainesMaintenances": [
    {
      "type": "Révision",
      "kilometragePrevu": 45000,
      "dateEstimee": ISODate("2024-07-01")
    }
  ],
  "garantie": {
    "type": "Constructeur",
    "expiration": ISODate("2025-05-10"),
    "kilometrageMax": 100000
  }
}
```

**Avantages :**

- ✅ Le document véhicule reste léger pour les listes et recherches
- ✅ L'historique peut grandir sans limite (dans la limite de 16 Mo)
- ✅ Séparation des préoccupations : gestion véhicule vs maintenance

### Exemple 3 : Utilisateur et paramètres de confidentialité

**Scénario :** Application où les paramètres de confidentialité sont très détaillés.

**Collection "utilisateurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439014"),
  "nom": "Leclerc",
  "prenom": "Thomas",
  "email": "thomas.leclerc@example.com",
  "parametresConfidentialiteId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2"),
  "dateInscription": ISODate("2024-01-10")
}
```

**Collection "parametresConfidentialite" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d2"),
  "utilisateurId": ObjectId("507f1f77bcf86cd799439014"),
  "profil": {
    "visibilite": "amis",  // public, amis, privé
    "afficherEmail": false,
    "afficherTelephone": false,
    "afficherDateNaissance": false
  },
  "recherche": {
    "apparaitreResultats": true,
    "indexerProfil": true
  },
  "notifications": {
    "email": {
      "nouveauMessage": true,
      "nouvelAmi": true,
      "commentaire": false,
      "newsletter": false
    },
    "push": {
      "nouveauMessage": true,
      "nouvelAmi": false
    },
    "sms": {
      "codeVerification": true,
      "alerteSuspecte": true
    }
  },
  "partage": {
    "localisationTempsReel": false,
    "historiqueLieux": false,
    "activites": "amis"
  },
  "blocages": {
    "utilisateursBloques": [
      ObjectId("..."),
      ObjectId("...")
    ],
    "motsBloques": ["spam", "publicite"]
  },
  "donnees": {
    "autoriserAnalyse": false,
    "autoriserPartagePublicitaire": false,
    "conserverHistorique": true
  },
  "dateModification": ISODate("2024-01-15")
}
```

**Avantages :**

- ✅ **Isolation des données sensibles** : permissions strictes sur les paramètres
- ✅ **Performance** : pas besoin de charger ces paramètres à chaque connexion
- ✅ **Évolution** : peut grandir sans impacter le profil utilisateur

---

## 3. Approche hybride : Dénormalisation sélective

### Principe

Combiner les deux approches : stocker une **référence** tout en **dénormalisant** quelques champs importants.

### Quand utiliser l'approche hybride ?

✅ **Utilisez une approche hybride quand :**

- Vous voulez **éviter une jointure** pour les champs fréquemment consultés
- Les données complètes sont volumineuses mais vous avez besoin d'un **résumé**
- Vous voulez **optimiser les listes** tout en permettant l'accès aux détails

### Exemple 1 : Commande avec référence client et données dénormalisées

**Scénario :** E-commerce où vous voulez afficher les commandes rapidement sans charger tous les détails client.

**Collection "clients" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439015"),
  "nom": "Martin",
  "prenom": "Sophie",
  "email": "sophie.martin@example.com",
  "telephone": "+33 6 12 34 56 78",
  "adresses": [
    {
      "type": "facturation",
      "rue": "12 rue de la Paix",
      "ville": "Paris",
      "codePostal": "75001"
    },
    {
      "type": "livraison",
      "rue": "25 avenue des Champs",
      "ville": "Lyon",
      "codePostal": "69001"
    }
  ],
  "dateInscription": ISODate("2023-06-10"),
  "statut": "premium"
}
```

**Collection "commandes" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d3"),
  "numeroCommande": "CMD-2024-001",
  "clientId": ObjectId("507f1f77bcf86cd799439015"),  // ← Référence
  // ↓ Données dénormalisées pour performance
  "clientNomComplet": "Sophie Martin",
  "clientEmail": "sophie.martin@example.com",
  "articles": [
    {
      "produitId": ObjectId("..."),
      "nom": "Livre MongoDB",
      "quantite": 2,
      "prixUnitaire": 29.99
    }
  ],
  "montantTotal": 59.98,
  "statut": "livree",
  "dateCommande": ISODate("2024-01-15"),
  "dateLivraison": ISODate("2024-01-18"),
  "adresseLivraison": {  // Snapshot au moment de la commande
    "rue": "25 avenue des Champs",
    "ville": "Lyon",
    "codePostal": "69001"
  }
}
```

**Avantages :**

- ✅ **Liste rapide** : afficher toutes les commandes sans jointure
- ✅ **Snapshot historique** : l'adresse au moment de la commande est conservée
- ✅ **Détails disponibles** : possibilité de charger le profil client complet via la référence
- ✅ **Évolution** : si le client change son email, les anciennes commandes gardent l'ancien

**Requêtes :**

```javascript
// Liste des commandes (rapide, sans jointure)
db.commandes.find({ clientId: ObjectId("507f1f77bcf86cd799439015") })
  .sort({ dateCommande: -1 })

// Détails d'une commande avec profil client complet (si nécessaire)
const commande = db.commandes.findOne({ numeroCommande: "CMD-2024-001" })
const client = db.clients.findOne({ _id: commande.clientId })
```

### Exemple 2 : Article de blog avec auteur

**Collection "auteurs" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439016"),
  "nom": "Jean Martin",
  "email": "jean.martin@blog.com",
  "bio": "Développeur passionné par MongoDB et les bases NoSQL...",
  "photo": "https://exemple.com/photos/jean-martin.jpg",
  "site": "https://jeanmartin.dev",
  "reseauxSociaux": {
    "twitter": "@jeandev",
    "linkedin": "jean-martin"
  },
  "statistiques": {
    "nombreArticles": 47,
    "nombreAbonnes": 1250
  }
}
```

**Collection "articles" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d4"),
  "titre": "Modélisation avancée dans MongoDB",
  "slug": "modelisation-avancee-mongodb",
  "contenu": "Dans cet article, nous allons explorer...",
  "auteurId": ObjectId("507f1f77bcf86cd799439016"),  // ← Référence
  // ↓ Données dénormalisées pour affichage
  "auteurNom": "Jean Martin",
  "auteurPhoto": "https://exemple.com/photos/jean-martin.jpg",
  "datePublication": ISODate("2024-01-15"),
  "tags": ["mongodb", "nosql", "modelisation"],
  "statut": "publie",
  "statistiques": {
    "vues": 1523,
    "likes": 89,
    "partages": 23
  }
}
```

**Avantages :**

- ✅ **Liste d'articles** affichable sans jointure (nom et photo auteur disponibles)
- ✅ **Page article** complète sans requête supplémentaire
- ✅ **Profil auteur détaillé** accessible via référence si besoin

---

## Comparaison des approches

| Critère | Imbrication | Références | Hybride |
|---------|-------------|------------|---------|
| **Performance lecture** | ✅ Excellente (1 requête) | ⚠️ 2 requêtes minimum | ✅ Bonne (1 requête pour le résumé) |
| **Taille du document** | ⚠️ Peut devenir volumineux | ✅ Documents légers | ✅ Parent léger |
| **Atomicité** | ✅ Garantie | ⚠️ Nécessite transactions | ⚠️ Selon le besoin |
| **Séparation des données** | ❌ Tout ensemble | ✅ Collections séparées | ⚠️ Dépend des champs |
| **Permissions différentes** | ❌ Difficile | ✅ Facile | ⚠️ Possible |
| **Évolution du schéma** | ⚠️ Modifie le parent | ✅ Indépendant | ✅ Flexible |
| **Cohérence des données** | ✅ Automatique | ⚠️ À gérer | ⚠️ Duplication acceptée |
| **Complexité des requêtes** | ✅ Simple | ⚠️ Plus complexe | ⚠️ Moyenne |

---

## Critères de décision

### Choisir l'imbrication quand :

1. ✅ Les données liées sont **toujours consultées avec** le parent
2. ✅ Les données liées sont de **taille modérée** (< 100 Ko)
3. ✅ Vous avez besoin d'**atomicité** sur les modifications
4. ✅ Les données liées **n'ont pas de sens** séparément
5. ✅ Vous voulez des **performances optimales**

**Exemples types :**
- Utilisateur et profil
- Produit et spécifications
- Document et métadonnées

### Choisir les références quand :

1. ✅ Les données liées sont **volumineuses** (> 100 Ko)
2. ✅ Les données sont **consultées indépendamment**
3. ✅ Vous avez besoin de **permissions différentes**
4. ✅ Les données liées sont **optionnelles** et rarement présentes
5. ✅ Vous voulez **séparer les préoccupations**

**Exemples types :**
- Utilisateur et CV détaillé
- Véhicule et historique d'entretien
- Utilisateur et paramètres avancés

### Choisir l'approche hybride quand :

1. ✅ Vous voulez **optimiser les listes** avec quelques informations
2. ✅ Vous avez besoin d'un **snapshot historique**
3. ✅ Les données complètes sont volumineuses mais vous avez besoin d'un **résumé**
4. ✅ Vous voulez **éviter les jointures** sur les champs fréquents

**Exemples types :**
- Commande avec résumé client
- Article avec résumé auteur
- Projet avec résumé équipe

---

## Patterns courants

### Pattern 1 : Champs calculés imbriqués

Imbriquer des données calculées qui dépendent uniquement du parent.

```json
{
  "_id": ObjectId("..."),
  "numeroCommande": "CMD-2024-001",
  "articles": [
    { "nom": "Produit A", "prix": 29.99, "quantite": 2 },
    { "nom": "Produit B", "prix": 19.99, "quantite": 1 }
  ],
  "calculsTotaux": {  // ← Imbriqué car calculé à partir des articles
    "sousTotal": 79.97,
    "tva": 15.99,
    "fraisPort": 5.00,
    "total": 100.96
  }
}
```

### Pattern 2 : Référence avec cache

Stocker une référence tout en cachant certaines données pour éviter les requêtes répétées.

```json
{
  "_id": ObjectId("..."),
  "titre": "Message du forum",
  "auteurId": ObjectId("..."),  // Référence complète
  "auteurCache": {  // Cache pour éviter les requêtes
    "nom": "Jean Martin",
    "avatar": "https://...",
    "dateMiseAJour": ISODate("2024-01-15")
  },
  "contenu": "...",
  "datePublication": ISODate("2024-01-15")
}
```

**Note :** Il faut gérer la synchronisation du cache si l'auteur modifie son profil.

### Pattern 3 : Référence optionnelle

Utiliser une référence qui peut ne pas exister.

```json
{
  "_id": ObjectId("..."),
  "nom": "Martin",
  "email": "martin@example.com",
  "cvId": null,  // ← Optionnel : l'utilisateur n'a pas encore créé de CV
  "dateInscription": ISODate("2024-01-10")
}
```

---

## Cas particuliers

### Relation one-to-one bidirectionnelle

Parfois, vous voulez pouvoir chercher dans les deux sens facilement.

**Exemple : Employé ↔ Badge d'accès**

**Collection "employes" :**
```json
{
  "_id": ObjectId("507f1f77bcf86cd799439017"),
  "nom": "Dupont",
  "matricule": "EMP-001",
  "badgeId": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5")  // ← Référence vers badge
}
```

**Collection "badges" :**
```json
{
  "_id": ObjectId("60a1f1b2c3d4e5f6a7b8c9d5"),
  "numero": "BADGE-12345",
  "employeId": ObjectId("507f1f77bcf86cd799439017"),  // ← Référence vers employé
  "dateActivation": ISODate("2024-01-10"),
  "statut": "actif"
}
```

**Avantages :**
- ✅ Recherche rapide dans les deux sens
- ✅ Utile si vous scannez un badge et voulez trouver l'employé

**Inconvénient :**
- ⚠️ Maintenir la cohérence des deux références

### Relation optionnelle avec valeur par défaut

**Exemple : Utilisateur avec thème personnalisé**

```json
{
  "_id": ObjectId("..."),
  "nom": "Martin",
  "email": "martin@example.com",
  "themePersonnalise": {  // ← Optionnel, si absent = thème par défaut
    "couleurPrimaire": "#3498db",
    "couleurSecondaire": "#2ecc71",
    "police": "Roboto"
  }
}
```

Si le champ `themePersonnalise` n'existe pas, l'application utilise le thème par défaut.

---

## Recommandations pratiques

### ✅ Bonnes pratiques

1. **Privilégiez l'imbrication par défaut** pour les relations one-to-one
2. **Mesurez la taille** de vos documents imbriqués
3. **Documentez vos choix** : expliquez pourquoi vous avez séparé ou imbriqué
4. **Pensez aux permissions** : données sensibles → références
5. **Anticipez la croissance** : si les données peuvent grossir, utilisez des références
6. **Testez les performances** avec des volumes réalistes

### ⚠️ Pièges à éviter

1. **Séparer systématiquement** comme en SQL (perte des avantages MongoDB)
2. **Imbriquer des données volumineuses** sans réfléchir à la limite de 16 Mo
3. **Ne pas gérer la cohérence** des données dénormalisées
4. **Oublier les index** sur les champs de référence
5. **Ne pas considérer les patterns d'accès** réels de l'application

---

## Exemples de refactoring

### Avant : Tout imbriqué (problématique)

```json
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "email": "dupont@example.com",
  "cv": {
    // ⚠️ 200 Ko de données CV ici
    "experiences": [ /* 50 expériences détaillées */ ],
    "formations": [ /* ... */ ],
    "projets": [ /* ... */ ]
  }
}
```

**Problème :** Chaque requête sur l'utilisateur charge le CV volumineux.

### Après : Références (amélioré)

```json
// Collection "utilisateurs"
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "email": "dupont@example.com",
  "cvId": ObjectId("...")  // ← Référence
}

// Collection "cv"
{
  "_id": ObjectId("..."),
  "utilisateurId": ObjectId("..."),
  "experiences": [ /* ... */ ],
  "formations": [ /* ... */ ],
  "projets": [ /* ... */ ]
}
```

**Amélioration :** Le profil utilisateur est léger, le CV est chargé à la demande.

---

## Conclusion

Les relations **one-to-one** dans MongoDB sont simples à modéliser, mais le choix entre imbrication et références dépend de votre contexte :

- **Imbrication** : Solution par défaut pour les données consultées ensemble, atomiques et de taille modérée
- **Références** : Pour les données volumineuses, sensibles ou consultées indépendamment
- **Hybride** : Compromis optimal pour les cas où vous voulez performances ET flexibilité

**Règle d'or :** Privilégiez l'imbrication sauf si vous avez une bonne raison de séparer (taille, permissions, consultation indépendante).

---

**Points clés à retenir :**

- ✅ L'imbrication est la solution naturelle pour les relations one-to-one
- ✅ Utilisez des références pour les données volumineuses (> 100 Ko)
- ✅ Pensez aux patterns d'accès de votre application
- ✅ L'approche hybride offre un bon compromis
- ✅ Attention à la limite de 16 Mo par document
- ✅ Documentez vos choix de modélisation
- ✅ Testez avec des volumes réalistes

---


⏭️ [Relations One-to-Many](/04-modelisation-des-donnees/04-relations-one-to-many.md)
