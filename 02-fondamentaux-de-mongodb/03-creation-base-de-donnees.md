🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.3 Création d'une Base de Données

## Introduction

Bienvenue dans ce chapitre où nous allons apprendre à créer et gérer des bases de données dans MongoDB ! Si vous venez du monde SQL, vous allez découvrir que MongoDB adopte une approche différente et plus flexible.

> **💡 Particularité MongoDB :** Contrairement aux bases de données relationnelles où vous devez explicitement créer une base avant de l'utiliser, MongoDB crée automatiquement les bases de données et les collections dès que vous y insérez des données. C'est ce qu'on appelle la **création implicite**.

Dans cette section, nous allons voir :
- Comment créer une base de données
- Les commandes essentielles pour gérer les bases
- Les conventions de nommage
- Les bonnes pratiques

---

## Création Implicite vs Explicite

### Le Concept de Création Implicite

MongoDB suit le principe du **"lazy creation"** (création paresseuse) :

**Principe :** Une base de données n'est réellement créée que lorsque vous y stockez des données.

```javascript
// Vous "créez" une base appelée "mabase"
use mabase

// À ce stade, la base n'existe PAS encore !
// Elle apparaîtra seulement quand vous insérerez des données
```

**Pourquoi cette approche ?**
- 🚀 **Simplicité** : Pas besoin de commandes DDL complexes
- 💾 **Efficacité** : Pas de stockage pour des bases vides
- 🎯 **Rapidité** : Commencez à coder immédiatement

### Comparaison avec SQL

**Approche SQL traditionnelle :**
```sql
-- Étape 1 : Créer explicitement la base
CREATE DATABASE mabase;

-- Étape 2 : Sélectionner la base
USE mabase;

-- Étape 3 : Créer les tables
CREATE TABLE utilisateurs (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(100)
);

-- Étape 4 : Insérer des données
INSERT INTO utilisateurs VALUES (1, 'Dupont', 'dupont@example.com');
```

**Approche MongoDB :**
```javascript
// Étape 1 : Sélectionner la base (création implicite)
use mabase

// Étape 2 : Insérer des données (la base et la collection sont créées automatiquement)
db.utilisateurs.insertOne({
  nom: "Dupont",
  email: "dupont@example.com"
})
```

**Beaucoup plus simple, n'est-ce pas ? 😊**

---

## Commande `use` : Créer/Sélectionner une Base

### Syntaxe

```javascript
use nom_de_la_base
```

### Comportement

La commande `use` a deux fonctions :
1. **Si la base existe** → Elle la sélectionne (vous bascule dessus)
2. **Si la base n'existe pas** → Elle prépare sa création (effective après insertion)

### Exemples Pratiques

```javascript
// Exemple 1 : Créer/sélectionner une base pour un blog
use blog

// Exemple 2 : Base pour une boutique en ligne
use ecommerce

// Exemple 3 : Base pour un système de gestion
use gestion_entreprise
```

### Vérification de la Base Courante

```javascript
// Afficher la base de données actuellement sélectionnée
db

// Ou de manière plus explicite
db.getName()
```

**Sortie :**
```
blog
```

---

## Lister les Bases de Données

### Commande `show dbs` ou `show databases`

```javascript
// Afficher toutes les bases de données
show dbs

// Forme alternative (identique)
show databases
```

**Exemple de sortie :**
```
admin       40.00 KiB
config      60.00 KiB
local       80.00 KiB
blog        120.00 KiB
ecommerce   2.50 MiB
```

**Informations affichées :**
- **Nom de la base** : Nom que vous avez donné
- **Taille** : Espace disque utilisé

### Bases de Données Système

Vous remarquerez toujours trois bases spéciales :

| Base | Description | Usage |
|------|-------------|-------|
| **admin** | Base administrative | Gestion des utilisateurs, rôles, commandes système |
| **config** | Configuration | Métadonnées pour les clusters shardés |
| **local** | Données locales | Données spécifiques au serveur, non répliquées |

> **⚠️ Attention :** Ne modifiez pas ces bases système sauf si vous savez exactement ce que vous faites !

### Pourquoi ma base n'apparaît pas ?

```javascript
// Vous faites :
use nouvelle_base

// Puis vous listez :
show dbs
// nouvelle_base n'apparaît PAS !
```

**Raison :** La base n'a pas encore de données. MongoDB ne liste que les bases contenant au moins un document.

**Solution :** Insérez au moins un document :

```javascript
use nouvelle_base
db.test.insertOne({ message: "Hello MongoDB!" })

// Maintenant :
show dbs
// nouvelle_base apparaît ! ✅
```

---

## Conventions de Nommage

### Règles Obligatoires

MongoDB impose certaines règles pour nommer les bases de données :

✅ **Autorisé :**
```javascript
use monprojet
use mon_projet
use mon-projet
use projet2024
use PROJET
```

❌ **Interdit :**
```javascript
use mon projet        // ❌ Espaces interdits
use mon/projet        // ❌ Slash interdit
use mon\projet        // ❌ Backslash interdit
use mon.projet        // ❌ Point interdit
use ""                // ❌ Nom vide interdit
use $monprojet        // ❌ $ au début interdit (Windows)
```

### Contraintes Système

| Contrainte | Limite |
|------------|--------|
| **Longueur max** | 64 caractères |
| **Caractères interdits** | `/\. "$*<>:|?` |
| **Sensibilité à la casse** | Oui (selon OS) |
| **Nom réservé** | `admin`, `local`, `config` |

### Sensibilité à la Casse (Case Sensitivity)

**⚠️ Important :** Le comportement dépend du système d'exploitation :

```javascript
// Sur Linux/macOS : Sensible à la casse
use MonProjet  // Base différente de
use monprojet  // celle-ci

// Sur Windows : Insensible à la casse
use MonProjet  // Considéré comme la même base que
use monprojet  // celle-ci
```

**Bonne pratique :** Pour la portabilité, considérez toujours que les noms sont sensibles à la casse.

### Recommandations de Nommage

**✅ Bonnes pratiques :**

```javascript
// Snake case (recommandé)
use gestion_entreprise
use blog_personnel
use api_production

// Kebab case (acceptable)
use gestion-entreprise
use blog-personnel

// camelCase (moins courant mais OK)
use gestionEntreprise
use blogPersonnel

// Tout en minuscules (le plus simple)
use blog
use ecommerce
use gestion
```

**❌ À éviter :**

```javascript
use BLOG              // ❌ Tout en majuscules
use GestionEntreprise // ❌ PascalCase (confusion)
use blog_2024_v2_final // ❌ Trop long et complexe
use db                // ❌ Nom trop générique
use test              // ❌ Ambigu (dev vs prod ?)
```

### Conseils de Nommage

1. **Soyez descriptif** : `boutique_en_ligne` plutôt que `app`
2. **Utilisez des préfixes** pour organiser : `prod_ecommerce`, `dev_ecommerce`
3. **Cohérence** : Choisissez une convention et respectez-la
4. **Évitez les accents** : `gestion` plutôt que `géstion`
5. **Pas d'emojis** : Techniquement possible mais non recommandé 😅

---

## Créer une Base de Données : Pas à Pas

### Méthode 1 : Création Simple

**Étape par étape :**

```javascript
// 1. Démarrer mongosh (le shell MongoDB)
mongosh

// 2. Sélectionner/créer la base
use ma_premiere_base

// 3. Vérifier la base courante
db
// Sortie : ma_premiere_base

// 4. Insérer un premier document
db.ma_collection.insertOne({
  message: "Bienvenue dans MongoDB !",
  date: new Date()
})

// 5. Vérifier que la base existe maintenant
show dbs
// ma_premiere_base apparaît maintenant !
```

### Méthode 2 : Création avec Multiple Collections

```javascript
// Sélectionner la base
use blog

// Créer plusieurs collections avec des données initiales
db.articles.insertOne({
  titre: "Mon premier article",
  contenu: "Introduction à MongoDB",
  auteur: "Jean",
  datePublication: new Date()
})

db.auteurs.insertOne({
  nom: "Jean",
  email: "jean@example.com",
  bio: "Développeur passionné"
})

db.categories.insertOne({
  nom: "Tutoriels",
  description: "Articles pédagogiques"
})

// Vérifier les collections créées
show collections
```

**Sortie :**
```
articles
auteurs
categories
```

---

## Commandes Essentielles

### Afficher les Informations d'une Base

```javascript
// Sélectionner la base
use blog

// Obtenir des statistiques sur la base
db.stats()
```

**Exemple de sortie :**
```javascript
{
  db: 'blog',
  collections: 3,
  views: 0,
  objects: 3,           // Nombre total de documents
  avgObjSize: 156,      // Taille moyenne d'un document
  dataSize: 468,        // Taille totale des données
  storageSize: 36864,   // Espace de stockage
  indexes: 3,           // Nombre d'index
  indexSize: 36864,     // Taille des index
  totalSize: 73728,     // Taille totale
  scaleFactor: 1,
  fsUsedSize: 245760,
  fsTotalSize: 499963174912,
  ok: 1
}
```

### Statistiques Lisibles

```javascript
// Statistiques en Mo/Go
db.stats(1024 * 1024)  // En mégaoctets

// Ou
db.stats(1024 * 1024 * 1024)  // En gigaoctets
```

### Afficher les Collections

```javascript
// Méthode 1 : Commande shell
show collections

// Méthode 2 : Commande programmatique
db.getCollectionNames()

// Méthode 3 : Avec plus de détails
db.getCollectionInfos()
```

### Vérifier l'Existence d'une Base

```javascript
// Lister toutes les bases et chercher
show dbs

// Ou via une commande
db.adminCommand('listDatabases')
```

---

## Supprimer une Base de Données

### Commande `dropDatabase()`

**⚠️ ATTENTION : Cette action est IRRÉVERSIBLE !**

```javascript
// 1. Sélectionner la base à supprimer
use base_a_supprimer

// 2. Confirmer la base courante
db
// Sortie : base_a_supprimer

// 3. Supprimer la base
db.dropDatabase()
```

**Sortie :**
```javascript
{
  dropped: 'base_a_supprimer',
  ok: 1
}
```

### Vérification

```javascript
// Vérifier que la base a été supprimée
show dbs
// base_a_supprimer n'apparaît plus
```

### Sécurité

Pour éviter les suppressions accidentelles, plusieurs précautions :

```javascript
// ❌ Danger : Suppression sans vérification
use production_ecommerce
db.dropDatabase()  // CATASTROPHE !

// ✅ Bon : Vérification en plusieurs étapes
// 1. Vérifier la base courante
db.getName()

// 2. Vérifier le contenu
show collections

// 3. Peut-être créer une sauvegarde avant
// mongodump --db=production_ecommerce

// 4. Puis seulement supprimer si certain
db.dropDatabase()
```

---

## Travailler avec Plusieurs Bases

### Navigation entre les Bases

```javascript
// Créer et utiliser plusieurs bases
use blog
db.articles.insertOne({ titre: "Article 1" })

use ecommerce
db.produits.insertOne({ nom: "Produit 1" })

use gestion
db.employes.insertOne({ nom: "Employé 1" })

// Naviguer entre les bases
use blog
db  // blog

use ecommerce
db  // ecommerce
```

### Accéder à une Base Différente

```javascript
// Vous êtes dans la base 'blog'
use blog

// Accéder à une collection d'une autre base
db.getSiblingDB('ecommerce').produits.find()

// Ou via une variable
const ecommerceDb = db.getSiblingDB('ecommerce')
ecommerceDb.produits.find()
```

### Exemple Pratique : Connexions Multi-Bases

```javascript
// Base actuelle
use blog

// Insérer dans la base actuelle
db.articles.insertOne({ titre: "Article blog" })

// Insérer dans une autre base sans changer de contexte
db.getSiblingDB('ecommerce').produits.insertOne({
  nom: "Produit",
  prix: 29.99
})

// Vérifier que vous êtes toujours dans blog
db.getName()  // blog
```

---

## Exemples de Création par Secteur

### 1. Blog Personnel

```javascript
use blog_personnel

// Articles
db.articles.insertOne({
  titre: "Bienvenue sur mon blog",
  contenu: "Ceci est mon premier article...",
  auteur: "Marie",
  tags: ["bienvenue", "introduction"],
  datePublication: new Date(),
  publie: true
})

// Commentaires
db.commentaires.insertOne({
  articleId: ObjectId("..."),
  auteur: "Visiteur",
  texte: "Super article !",
  date: new Date()
})

// Statistiques
db.stats.insertOne({
  date: new Date(),
  visiteurs: 0,
  pages_vues: 0
})
```

### 2. E-commerce

```javascript
use boutique_en_ligne

// Produits
db.produits.insertOne({
  reference: "PROD-001",
  nom: "T-shirt MongoDB",
  prix: 19.99,
  stock: 100,
  categorie: "Vêtements"
})

// Clients
db.clients.insertOne({
  nom: "Dupont",
  prenom: "Pierre",
  email: "pierre.dupont@example.com",
  adresse: {
    rue: "123 Rue Example",
    ville: "Paris",
    codePostal: "75001"
  }
})

// Commandes
db.commandes.insertOne({
  clientId: ObjectId("..."),
  produits: [
    { produitId: ObjectId("..."), quantite: 2 }
  ],
  total: 39.98,
  statut: "en_attente",
  dateCommande: new Date()
})
```

### 3. Application de Gestion

```javascript
use gestion_entreprise

// Employés
db.employes.insertOne({
  matricule: "EMP001",
  nom: "Martin",
  prenom: "Sophie",
  poste: "Développeuse",
  departement: "IT",
  dateEmbauche: new Date("2024-01-15"),
  salaire: 45000
})

// Projets
db.projets.insertOne({
  nom: "Migration vers MongoDB",
  responsable: ObjectId("..."),
  dateDebut: new Date("2024-01-01"),
  dateFin: new Date("2024-06-30"),
  budget: 50000,
  statut: "en_cours"
})

// Tâches
db.taches.insertOne({
  projetId: ObjectId("..."),
  titre: "Mise en place de la réplication",
  assigneA: ObjectId("..."),
  priorite: "haute",
  statut: "todo",
  dateCreation: new Date()
})
```

### 4. Système de Logs

```javascript
use logs_application

// Logs d'application
db.logs.insertMany([
  {
    niveau: "INFO",
    message: "Application démarrée",
    timestamp: new Date(),
    service: "api"
  },
  {
    niveau: "ERROR",
    message: "Erreur de connexion à la base",
    timestamp: new Date(),
    service: "database",
    stack: "Error: Connection refused..."
  },
  {
    niveau: "WARNING",
    message: "Utilisation mémoire élevée",
    timestamp: new Date(),
    service: "monitoring",
    memoire: "85%"
  }
])
```

---

## Gestion Environnements (Dev, Test, Prod)

### Stratégie de Nommage

```javascript
// Environnement de développement
use dev_blog
use dev_ecommerce
use dev_api

// Environnement de test
use test_blog
use test_ecommerce
use test_api

// Environnement de production
use prod_blog
use prod_ecommerce
use prod_api
```

### Ou avec des Bases Séparées par Suffixe

```javascript
// Une base, plusieurs suffixes
use blog_dev
use blog_test
use blog_prod
```

### Connexion selon l'Environnement

```javascript
// Dans votre code applicatif (exemple Node.js)
const env = process.env.NODE_ENV || 'dev';
const dbName = `${env}_blog`;

// Se connecter à la bonne base
const client = new MongoClient(uri);
const database = client.db(dbName);
```

---

## Bonnes Pratiques

### ✅ À Faire

1. **Noms descriptifs et cohérents**
   ```javascript
   use blog_entreprise      // ✅ Clair
   use ecommerce_france     // ✅ Spécifique
   ```

2. **Préfixes pour les environnements**
   ```javascript
   use prod_application
   use dev_application
   use test_application
   ```

3. **Documentation**
   ```javascript
   // Créer une collection metadata pour documenter
   use mon_projet
   db.metadata.insertOne({
     nom: "mon_projet",
     description: "Application de gestion des stocks",
     version: "1.0",
     dateCreation: new Date(),
     responsable: "Équipe Dev"
   })
   ```

4. **Vérification avant suppression**
   ```javascript
   // Toujours vérifier avant de supprimer
   db.getName()
   show collections
   db.stats()
   // Puis seulement :
   db.dropDatabase()
   ```

### ❌ À Éviter

1. **Noms génériques**
   ```javascript
   use db        // ❌ Trop vague
   use test      // ❌ Ambigu
   use temp      // ❌ Pas clair
   ```

2. **Caractères problématiques**
   ```javascript
   use mon projet    // ❌ Espaces
   use mon/projet    // ❌ Slash
   use 2024-projet   // ⚠️ Commence par un nombre (déconseillé)
   ```

3. **Trop de bases**
   ```javascript
   // ❌ Évitez de créer une base par utilisateur
   use user_1
   use user_2
   use user_3
   // ...
   // ✅ Préférez une base avec une collection
   use application
   db.users.insertOne({ userId: 1, ... })
   ```

4. **Duplication inutile**
   ```javascript
   // ❌ Bases redondantes
   use blog
   use blog2
   use blog_new
   use blog_final
   use blog_final_v2
   ```

---

## Commandes Administratives Avancées

### Clone d'une Base

MongoDB ne fournit pas de commande native pour cloner une base, mais vous pouvez le faire ainsi :

```javascript
// Méthode 1 : Via mongodump et mongorestore (en dehors du shell)
// mongodump --db=source_db
// mongorestore --db=destination_db dump/source_db/

// Méthode 2 : Copier collection par collection (dans le shell)
use source_db
const collections = db.getCollectionNames()

collections.forEach(collection => {
  db.getSiblingDB('destination_db')[collection].insertMany(
    db[collection].find().toArray()
  )
})
```

### Renommer une Base

MongoDB ne permet pas de renommer directement une base. Voici la procédure :

```javascript
// 1. Créer la nouvelle base avec les données
use ancienne_base
const collections = db.getCollectionNames()

collections.forEach(collection => {
  db.getSiblingDB('nouvelle_base')[collection].insertMany(
    db[collection].find().toArray()
  )
})

// 2. Vérifier que tout est OK
use nouvelle_base
show collections

// 3. Supprimer l'ancienne base
use ancienne_base
db.dropDatabase()
```

> **💡 Note :** Pour les grosses bases, utilisez plutôt `mongodump` et `mongorestore` avec l'option `--nsFrom` et `--nsTo`.

---

## MongoDB Compass : Interface Graphique

### Créer une Base via Compass

Si vous préférez une interface graphique :

1. **Ouvrir MongoDB Compass**
2. **Se connecter** à votre serveur MongoDB
3. **Cliquer sur "Create Database"**
4. **Remplir le formulaire :**
   - Database Name : `mon_projet`
   - Collection Name : `utilisateurs`
5. **Cliquer sur "Create Database"**

**Avantage :** La base ET la collection sont créées immédiatement, même vides !

### Navigation dans Compass

- Liste des bases sur la gauche
- Statistiques visibles pour chaque base
- Recherche et filtrage faciles
- Interface conviviale pour les débutants

---

## Commandes Récapitulatives

Voici un aide-mémoire des commandes essentielles :

| Commande | Description | Exemple |
|----------|-------------|---------|
| `use <nom>` | Créer/sélectionner une base | `use blog` |
| `db` | Afficher la base courante | `db` |
| `db.getName()` | Obtenir le nom de la base | `db.getName()` |
| `show dbs` | Lister toutes les bases | `show dbs` |
| `show databases` | Lister toutes les bases | `show databases` |
| `show collections` | Lister les collections | `show collections` |
| `db.stats()` | Statistiques de la base | `db.stats()` |
| `db.dropDatabase()` | Supprimer la base | `db.dropDatabase()` |
| `db.getCollectionNames()` | Liste des collections | `db.getCollectionNames()` |
| `db.getSiblingDB()` | Accéder à une autre base | `db.getSiblingDB('autre')` |

---

## Points Clés à Retenir

### ✅ Essentiel

1. **Création implicite** : Les bases sont créées automatiquement à l'insertion
2. **Commande `use`** : Sélectionne ou prépare la création d'une base
3. **`show dbs`** : Liste uniquement les bases contenant des données
4. **Nommage** : 64 caractères max, pas d'espaces, cohérence importante
5. **Suppression** : `dropDatabase()` est irréversible, toujours vérifier !

### 🎯 Workflow Standard

```javascript
// 1. Créer/sélectionner la base
use mon_projet

// 2. Insérer des données
db.ma_collection.insertOne({ test: "hello" })

// 3. Vérifier
show dbs
db.getName()
show collections
```

### ⚠️ Pièges Courants

- ❌ Oublier d'insérer des données (la base n'apparaît pas)
- ❌ Utiliser des caractères interdits dans le nom
- ❌ Supprimer la mauvaise base
- ❌ Créer trop de bases au lieu d'utiliser des collections

---

## Prochaines Étapes

Maintenant que vous savez créer des bases de données, passons aux collections :

➡️ **2.4 Création et gestion des collections** : Organiser vos données en collections

La maîtrise des bases de données est la fondation de votre travail avec MongoDB !

---

## Ressources Complémentaires

### Documentation Officielle

- [MongoDB Databases](https://docs.mongodb.com/manual/core/databases-and-collections/)
- [Database Commands](https://docs.mongodb.com/manual/reference/command/)
- [Naming Restrictions](https://docs.mongodb.com/manual/reference/limits/#naming-restrictions)

### Pour Aller Plus Loin

- Sécurisation des bases avec l'authentification
- Gestion des utilisateurs par base
- Monitoring des bases en production

---


⏭️ [Création et gestion des collections](/02-fondamentaux-de-mongodb/04-creation-gestion-collections.md)
