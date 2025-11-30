🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.10 Présentation des outils : mongosh, MongoDB Compass, Atlas

## Introduction

MongoDB propose un écosystème d'outils complet pour interagir avec vos bases de données. Que vous préfériez la ligne de commande, une interface graphique ou une solution cloud, il existe un outil adapté à vos besoins.

Cette section présente les trois outils principaux que vous utiliserez au quotidien :

- **mongosh** : Le shell en ligne de commande
- **MongoDB Compass** : L'interface graphique de bureau
- **MongoDB Atlas** : La plateforme cloud

---

## Vue d'ensemble des outils

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Écosystème des outils MongoDB                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│   │     mongosh     │  │    Compass      │  │     Atlas       │     │
│   │                 │  │                 │  │                 │     │
│   │   ┌─────────┐   │  │   ┌─────────┐   │  │   ┌─────────┐   │     │
│   │   │  >_     │   │  │   │  🖥️     │   │  │   │  ☁️     │   │     │
│   │   └─────────┘   │  │   └─────────┘   │  │   └─────────┘   │     │
│   │                 │  │                 │  │                 │     │
│   │  Ligne de       │  │  Interface      │  │  Plateforme     │     │
│   │  commande       │  │  graphique      │  │  cloud          │     │
│   │                 │  │                 │  │                 │     │
│   │  • Scripts      │  │  • Visuel       │  │  • Hébergé      │     │
│   │  • Automation   │  │  • Exploration  │  │  • Managé       │     │
│   │  • DevOps       │  │  • Débutants    │  │  • Scalable     │     │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Comparaison rapide

| Outil | Type | Usage principal | Public cible |
|-------|------|-----------------|--------------|
| **mongosh** | CLI | Scripts, administration, développement | Développeurs, DevOps |
| **Compass** | GUI | Exploration, visualisation, requêtes | Débutants, analystes |
| **Atlas** | Cloud | Production, hosting, services managés | Tous profils |

---

## mongosh : Le shell MongoDB

### Qu'est-ce que mongosh ?

**mongosh** (MongoDB Shell) est l'interface en ligne de commande officielle pour interagir avec MongoDB. C'est un environnement JavaScript interactif qui permet d'exécuter des requêtes, d'administrer la base de données et d'écrire des scripts.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         mongosh                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   $ mongosh                                                         │
│   Current Mongosh Log ID: 6569f8c9e3b2a1d4c5e6f7a8                  │
│   Connecting to:          mongodb://127.0.0.1:27017                 │
│   Using MongoDB:          8.0.4                                     │
│   Using Mongosh:          2.3.0                                     │
│                                                                     │
│   For mongosh info see: https://docs.mongodb.com/mongodb-shell/     │
│                                                                     │
│   test> _                                                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Installation de mongosh

#### Windows

```powershell
# Via l'installateur MongoDB (inclus)
# Ou téléchargement séparé depuis mongodb.com/try/download/shell

# Via winget
winget install MongoDB.Shell
```

#### macOS

```bash
# Via Homebrew
brew install mongosh
```

#### Linux (Ubuntu/Debian)

```bash
# Si MongoDB est installé, mongosh est inclus
# Sinon, installation séparée :
wget https://downloads.mongodb.com/compass/mongosh-2.3.0-linux-x64.tgz
tar -zxvf mongosh-2.3.0-linux-x64.tgz
sudo cp mongosh-2.3.0-linux-x64/bin/* /usr/local/bin/
```

### Connexion à MongoDB

```bash
# Connexion locale (par défaut)
mongosh

# Connexion avec URI
mongosh "mongodb://localhost:27017"

# Connexion avec authentification
mongosh "mongodb://username:password@localhost:27017/database"

# Connexion à un Replica Set
mongosh "mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=myRS"

# Connexion à MongoDB Atlas
mongosh "mongodb+srv://username:password@cluster.xxxxx.mongodb.net/database"

# Options supplémentaires
mongosh --host localhost --port 27017 --username admin --password secret
```

### Commandes de navigation essentielles

```javascript
// Afficher l'aide
help

// Afficher la base de données actuelle
db

// Lister toutes les bases de données
show dbs

// Basculer vers une base de données
use mabase

// Lister les collections de la base actuelle
show collections

// Afficher les utilisateurs
show users

// Afficher les rôles
show roles

// Quitter le shell
exit
// ou Ctrl+C deux fois
// ou .exit
```

### Opérations CRUD de base

```javascript
// === CREATE ===
// Insérer un document
db.users.insertOne({ name: "Alice", age: 28, city: "Paris" })

// Insérer plusieurs documents
db.users.insertMany([
  { name: "Bob", age: 32, city: "Lyon" },
  { name: "Charlie", age: 25, city: "Marseille" }
])

// === READ ===
// Trouver tous les documents
db.users.find()

// Trouver avec un filtre
db.users.find({ city: "Paris" })

// Trouver un seul document
db.users.findOne({ name: "Alice" })

// Avec formatage lisible
db.users.find().pretty()

// === UPDATE ===
// Mettre à jour un document
db.users.updateOne(
  { name: "Alice" },
  { $set: { age: 29 } }
)

// Mettre à jour plusieurs documents
db.users.updateMany(
  { city: "Paris" },
  { $set: { country: "France" } }
)

// === DELETE ===
// Supprimer un document
db.users.deleteOne({ name: "Charlie" })

// Supprimer plusieurs documents
db.users.deleteMany({ age: { $lt: 25 } })

// Supprimer tous les documents d'une collection
db.users.deleteMany({})
```

### Fonctionnalités avancées de mongosh

#### Autocomplétion

mongosh propose une autocomplétion intelligente :

```javascript
db.us<TAB>     // Complète en db.users
db.users.fi<TAB>  // Propose find, findOne, findOneAndUpdate...
```

#### Historique des commandes

```bash
# Flèches haut/bas pour naviguer dans l'historique
# Ctrl+R pour rechercher dans l'historique
```

#### Éditeur multi-lignes

```javascript
// Ouvrir l'éditeur externe
.editor

// Ou utiliser les accolades pour les blocs multi-lignes
db.users.aggregate([
  { $match: { city: "Paris" } },
  { $group: { _id: "$city", count: { $sum: 1 } } }
])
```

#### Exécuter des scripts

```bash
# Exécuter un fichier JavaScript
mongosh script.js

# Exécuter avec une connexion spécifique
mongosh "mongodb://localhost:27017/mabase" script.js

# Exécuter une commande directement
mongosh --eval "db.users.countDocuments()"
```

#### Fichier de configuration .mongoshrc.js

Créez un fichier `~/.mongoshrc.js` pour personnaliser mongosh :

```javascript
// ~/.mongoshrc.js

// Prompt personnalisé
prompt = function() {
  return db.getName() + " > ";
}

// Alias utiles
const count = (collection) => db[collection].countDocuments();
const stats = () => db.stats();

// Message de bienvenue
print("Bienvenue dans MongoDB !");
print("Base actuelle : " + db.getName());
```

### Commandes d'administration

```javascript
// Statistiques de la base
db.stats()

// Statistiques d'une collection
db.users.stats()

// Informations sur le serveur
db.serverStatus()

// Version du serveur
db.version()

// Créer un index
db.users.createIndex({ email: 1 }, { unique: true })

// Lister les index
db.users.getIndexes()

// Profiling des requêtes
db.setProfilingLevel(1, { slowms: 100 })

// Voir les requêtes en cours
db.currentOp()

// Tuer une opération
db.killOp(opId)
```

### Raccourcis clavier

| Raccourci | Action |
|-----------|--------|
| `Tab` | Autocomplétion |
| `↑` / `↓` | Naviguer dans l'historique |
| `Ctrl+C` | Annuler la commande en cours |
| `Ctrl+D` | Quitter (comme `exit`) |
| `Ctrl+L` | Effacer l'écran |
| `Ctrl+R` | Recherche dans l'historique |

---

## MongoDB Compass : L'interface graphique

### Qu'est-ce que MongoDB Compass ?

**MongoDB Compass** est l'interface graphique officielle de MongoDB. Elle permet d'explorer visuellement vos données, de construire des requêtes, d'analyser les performances et d'administrer vos bases de données sans écrire de code.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      MongoDB Compass                                │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ 🔌 New Connection                                           │    │
│  │                                                             │    │
│  │ URI: mongodb://localhost:27017                              │    │
│  │                                                             │    │
│  │ [    Connect    ]                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────┐ ┌────────────────────────────────────────────┐     │
│  │ Databases   │ │ Collections                                │     │
│  │             │ │                                            │     │
│  │ ▼ ecommerce │ │  users        (1,234 docs)                 │     │
│  │   products  │ │  products     (5,678 docs)                 │     │
│  │   users     │ │  orders       (12,345 docs)                │     │
│  │   orders    │ │                                            │     │
│  │             │ │                                            │     │
│  │ ▶ blog      │ │                                            │     │
│  │ ▶ analytics │ │                                            │     │
│  └─────────────┘ └────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Installation de Compass

#### Téléchargement

1. Rendez-vous sur [mongodb.com/try/download/compass](https://www.mongodb.com/try/download/compass)
2. Sélectionnez votre système d'exploitation
3. Téléchargez et installez

#### Windows

```powershell
# Via winget
winget install MongoDB.Compass.Full
```

#### macOS

```bash
# Via Homebrew
brew install --cask mongodb-compass
```

#### Linux

```bash
# Ubuntu/Debian (télécharger le .deb)
wget https://downloads.mongodb.com/compass/mongodb-compass_1.44.0_amd64.deb
sudo dpkg -i mongodb-compass_1.44.0_amd64.deb
```

### Éditions disponibles

| Édition | Description | Prix |
|---------|-------------|------|
| **Compass** | Version complète avec toutes les fonctionnalités | Gratuit |
| **Compass Readonly** | Lecture seule, pas de modification des données | Gratuit |
| **Compass Isolated** | Sans connexion réseau (environnements sécurisés) | Gratuit |

### Connexion à une base de données

#### Méthode 1 : URI de connexion

```
mongodb://localhost:27017
mongodb://username:password@localhost:27017/database
mongodb+srv://user:pass@cluster.mongodb.net/database
```

#### Méthode 2 : Formulaire détaillé

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Nouvelle connexion                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Hostname:     localhost                                           │
│   Port:         27017                                               │
│   Authentication:                                                   │
│     ○ None                                                          │
│     ● Username / Password                                           │
│         Username: admin                                             │
│         Password: ********                                          │
│         Auth DB:  admin                                             │
│   SSL:          ○ None  ● System CA  ○ Custom CA                    │
│                                                                     │
│   [ Save & Connect ]  [ Connect ]                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Fonctionnalités principales

#### 1. Exploration des données

Visualisez vos documents dans différents formats :

```
┌─────────────────────────────────────────────────────────────────────┐
│  Collection: users                                  Documents: 1,234│
├─────────────────────────────────────────────────────────────────────┤
│  View: [List] [JSON] [Table]                                        │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ {                                                           │    │
│  │   "_id": ObjectId("507f1f77bcf86cd799439011"),              │    │
│  │   "name": "Alice Dupont",                                   │    │
│  │   "email": "alice@example.com",                             │    │
│  │   "age": 28,                                                │    │
│  │   "address": {                                              │    │
│  │     "city": "Paris",                                        │    │
│  │     "country": "France"                                     │    │
│  │   }                                                         │    │
│  │ }                                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ {                                                           │    │
│  │   "_id": ObjectId("507f1f77bcf86cd799439012"),              │    │
│  │   "name": "Bob Martin",                                     │    │
│  │   ...                                                       │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 2. Constructeur de requêtes visuel

Construisez des requêtes sans écrire de code :

```
┌─────────────────────────────────────────────────────────────────────┐
│  Filter:  { age: { $gte: 25 }, city: "Paris" }                      │
│                                                                     │
│  Project: { name: 1, email: 1, age: 1 }                             │
│                                                                     │
│  Sort:    { age: -1 }                                               │
│                                                                     │
│  Limit:   10                                                        │
│                                                                     │
│  [ Find ]  [ Reset ]  [ Explain ]                                   │
└─────────────────────────────────────────────────────────────────────┘
```

#### 3. Pipeline d'agrégation visuel

Construisez des pipelines d'agrégation étape par étape :

```
┌─────────────────────────────────────────────────────────────────────┐
│  Aggregation Pipeline Builder                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Stage 1: $match                                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ { status: "active" }                                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│           │                                                         │
│           ▼                                                         │
│  Stage 2: $group                                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ { _id: "$city", total: { $sum: 1 } }                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│           │                                                         │
│           ▼                                                         │
│  Stage 3: $sort                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ { total: -1 }                                               │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  [+ Add Stage]                                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 4. Analyse des schémas

Compass analyse automatiquement la structure de vos documents :

```
┌─────────────────────────────────────────────────────────────────────┐
│  Schema Analysis: users                                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Field           Type        Probability                            │
│  ─────────────────────────────────────────                          │
│  _id             ObjectId    ████████████████████ 100%              │
│  name            String      ████████████████████ 100%              │
│  email           String      ████████████████████ 100%              │
│  age             Number      ██████████████████░░  90%              │
│  address         Object      ████████████████░░░░  80%              │
│  └─ city         String      ████████████████░░░░  80%              │
│  └─ country      String      ██████████████░░░░░░  70%              │
│  phone           String      ██████████░░░░░░░░░░  50%              │
│  createdAt       Date        ████████████████████ 100%              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 5. Gestion des index

Créez et analysez vos index visuellement :

```
┌─────────────────────────────────────────────────────────────────────┐
│  Indexes: users                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Name              Keys                    Size      Usage          │
│  ────────────────────────────────────────────────────────────────   │
│  _id_              { _id: 1 }              45 KB     Frequent       │
│  email_1           { email: 1 }            32 KB     Frequent       │
│  city_age_1        { city: 1, age: -1 }    28 KB     Rare           │
│                                                                     │
│  [+ Create Index]                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 6. Explain Plan

Analysez les performances de vos requêtes :

```
┌─────────────────────────────────────────────────────────────────────┐
│  Explain Plan                                                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Query: db.users.find({ city: "Paris" })                            │
│                                                                     │
│  ┌───────────────────────────────────────┐                          │
│  │           FETCH                       │                          │
│  │     Documents: 150                    │                          │
│  │     Time: 2ms                         │                          │
│  └──────────────┬────────────────────────┘                          │
│                 │                                                   │
│                 ▼                                                   │
│  ┌───────────────────────────────────────┐                          │
│  │          IXSCAN                       │                          │
│  │     Index: city_1                     │                          │
│  │     Keys Examined: 150                │                          │
│  │     Time: 1ms                         │                          │
│  └───────────────────────────────────────┘                          │
│                                                                     │
│  ✅ Index utilisé efficacement                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 7. Validation des schémas

Définissez des règles de validation visuellement :

```
┌─────────────────────────────────────────────────────────────────────┐
│  Schema Validation: users                                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Validation Level: [Strict ▼]                                       │
│  Validation Action: [Error ▼]                                       │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │ {                                                           │    │
│  │   "$jsonSchema": {                                          │    │
│  │     "bsonType": "object",                                   │    │
│  │     "required": ["name", "email"],                          │    │
│  │     "properties": {                                         │    │
│  │       "name": { "bsonType": "string" },                     │    │
│  │       "email": { "bsonType": "string" },                    │    │
│  │       "age": { "bsonType": "int", "minimum": 0 }            │    │
│  │     }                                                       │    │
│  │   }                                                         │    │
│  │ }                                                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  [ Update ]                                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Onglets et fonctionnalités

| Onglet | Fonctionnalité |
|--------|----------------|
| **Documents** | Visualiser, éditer, insérer, supprimer des documents |
| **Aggregations** | Construire des pipelines d'agrégation |
| **Schema** | Analyser la structure des documents |
| **Explain Plan** | Analyser les performances des requêtes |
| **Indexes** | Créer et gérer les index |
| **Validation** | Définir les règles de validation |

---

## MongoDB Atlas : La plateforme cloud

### Qu'est-ce que MongoDB Atlas ?

**MongoDB Atlas** est le service de base de données cloud entièrement géré par MongoDB Inc. Il vous permet de déployer, gérer et faire évoluer des clusters MongoDB sans gérer l'infrastructure.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       MongoDB Atlas                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │                    Atlas Dashboard                          │   │
│   │  ┌───────────────────────────────────────────────────────┐  │   │
│   │  │  Cluster: MyCluster                                   │  │   │
│   │  │  Status: ● Running                                    │  │   │
│   │  │  Region: AWS / eu-west-1                              │  │   │
│   │  │  Tier: M10 (General)                                  │  │   │
│   │  │                                                       │  │   │
│   │  │  Connections: 45 / 500                                │  │   │
│   │  │  Storage: 12.5 GB / 40 GB                             │  │   │
│   │  │  Operations: 1,234 ops/sec                            │  │   │
│   │  └───────────────────────────────────────────────────────┘  │   │
│   │                                                             │   │
│   │  [Connect]  [Browse Collections]  [Metrics]                 │   │
│   └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Avantages d'Atlas

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Pourquoi choisir Atlas ?                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ✅ Déploiement en quelques clics                                  │
│   ✅ Haute disponibilité automatique (Replica Sets)                 │
│   ✅ Sauvegardes automatiques avec point-in-time recovery           │
│   ✅ Scaling automatique (vertical et horizontal)                   │
│   ✅ Monitoring et alertes intégrés                                 │
│   ✅ Sécurité renforcée (chiffrement, authentification)             │
│   ✅ Multi-cloud (AWS, Azure, GCP)                                  │
│   ✅ Tier gratuit pour l'apprentissage (M0)                         │
│   ✅ Services additionnels (Search, Charts, Functions)              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Création d'un compte Atlas

#### Étape 1 : Inscription

1. Rendez-vous sur [mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)
2. Cliquez sur **Try Free**
3. Créez un compte avec :
   - Email + mot de passe
   - Ou Google / GitHub

#### Étape 2 : Créer une organisation et un projet

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Structure Atlas                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Organisation (ex: "Mon Entreprise")                               │
│   │                                                                 │
│   ├── Projet "Production"                                           │
│   │   ├── Cluster "prod-cluster"                                    │
│   │   └── Cluster "prod-analytics"                                  │
│   │                                                                 │
│   ├── Projet "Développement"                                        │
│   │   └── Cluster "dev-cluster"                                     │
│   │                                                                 │
│   └── Projet "Tests"                                                │
│       └── Cluster "test-cluster"                                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Création d'un cluster gratuit (M0)

#### Étape 1 : Choisir le type de cluster

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Choix du cluster                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ○ Serverless                                                      │
│     Facturation à l'usage, scaling automatique                      │
│                                                                     │
│   ○ Dedicated                                                       │
│     Ressources dédiées, haute performance                           │
│                                                                     │
│   ● Shared (M0, M2, M5)                                             │
│     Ressources partagées, idéal pour débuter                        │
│     ✓ M0 GRATUIT disponible                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Étape 2 : Configurer le cluster

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Configuration M0 (gratuit)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Cloud Provider:                                                   │
│   [AWS ●]  [Google Cloud ○]  [Azure ○]                              │
│                                                                     │
│   Region:                                                           │
│   [eu-west-1 (Ireland) ▼]                                           │
│                                                                     │
│   Cluster Tier:                                                     │
│   M0 Sandbox (Shared RAM, 512 MB Storage)                           │
│   Prix: GRATUIT                                                     │
│                                                                     │
│   Cluster Name: MyFirstCluster                                      │
│                                                                     │
│   [Create Cluster]                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Étape 3 : Configurer l'accès

**Créer un utilisateur de base de données :**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Database Access                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Username: myuser                                                  │
│   Password: ************                                            │
│                                                                     │
│   Built-in Role:                                                    │
│   ● Read and write to any database                                  │
│   ○ Only read any database                                          │
│   ○ Atlas admin                                                     │
│                                                                     │
│   [Add User]                                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Configurer l'accès réseau (IP Whitelist) :**

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Network Access                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   IP Access List                                                    │
│                                                                     │
│   ○ Add Current IP Address                                          │
│     (203.0.113.42)                                                  │
│                                                                     │
│   ○ Allow Access from Anywhere                                      │
│     (0.0.0.0/0) - Non recommandé en production                      │
│                                                                     │
│   ○ Add IP Address                                                  │
│     [________________]                                              │
│                                                                     │
│   [Add IP Address]                                                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Connexion à Atlas

Une fois le cluster créé, récupérez la chaîne de connexion :

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Connect to Cluster                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Choose a connection method:                                       │
│                                                                     │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│   │    mongosh      │  │    Compass      │  │  Application    │     │
│   │                 │  │                 │  │                 │     │
│   │   Shell CLI     │  │   GUI Tool      │  │   Driver        │     │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### Connexion avec mongosh

```bash
mongosh "mongodb+srv://mycluster.xxxxx.mongodb.net/" --apiVersion 1 --username myuser
```

#### Connexion avec Compass

```
mongodb+srv://myuser:password@mycluster.xxxxx.mongodb.net/
```

#### Connexion depuis une application (Node.js)

```javascript
const { MongoClient } = require('mongodb');

const uri = "mongodb+srv://myuser:password@mycluster.xxxxx.mongodb.net/?retryWrites=true&w=majority";
const client = new MongoClient(uri);

async function run() {
  try {
    await client.connect();
    console.log("Connecté à MongoDB Atlas !");

    const db = client.db("myDatabase");
    const collection = db.collection("users");

    // Insérer un document
    await collection.insertOne({ name: "Alice", age: 28 });

    // Lire les documents
    const users = await collection.find({}).toArray();
    console.log(users);

  } finally {
    await client.close();
  }
}

run().catch(console.dir);
```

### Services additionnels d'Atlas

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Services Atlas                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│   │Atlas Search │  │Atlas Charts │  │ Atlas Data  │                 │
│   │             │  │             │  │    Lake     │                 │
│   │ Recherche   │  │ Visualisa-  │  │ Analyse de  │                 │
│   │ full-text   │  │ tion de     │  │ données     │                 │
│   │ Lucene      │  │ données     │  │ volumineuses│                 │
│   └─────────────┘  └─────────────┘  └─────────────┘                 │
│                                                                     │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│   │   Atlas     │  │   Atlas     │  │   Atlas     │                 │
│   │  Triggers   │  │  Functions  │  │Vector Search│                 │
│   │             │  │             │  │             │                 │
│   │ Réactions   │  │ Serverless  │  │ Recherche   │                 │
│   │ automatiques│  │ JavaScript  │  │ vectorielle │                 │
│   │ aux events  │  │             │  │ (IA/ML)     │                 │
│   └─────────────┘  └─────────────┘  └─────────────┘                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Tarification Atlas

| Tier | RAM | Stockage | Prix approximatif |
|------|-----|----------|-------------------|
| **M0** (Shared) | Partagée | 512 MB | **Gratuit** |
| **M2** (Shared) | Partagée | 2 GB | ~$9/mois |
| **M5** (Shared) | Partagée | 5 GB | ~$25/mois |
| **M10** (Dedicated) | 2 GB | 10 GB | ~$60/mois |
| **M20** (Dedicated) | 4 GB | 20 GB | ~$140/mois |
| **M30+** | Variable | Variable | Variable |

> **Note** : Le tier M0 est parfait pour l'apprentissage et les petits projets. Passez aux tiers payants pour la production.

---

## Comparaison des outils

### Tableau récapitulatif

| Critère | mongosh | Compass | Atlas |
|---------|---------|---------|-------|
| **Type** | CLI | GUI Desktop | Cloud Platform |
| **Installation** | Locale | Locale | Aucune (web) |
| **Courbe d'apprentissage** | Moyenne | Faible | Faible |
| **Scripts/Automation** | ✅ Excellent | ❌ Non | ⚠️ Limité |
| **Exploration visuelle** | ❌ Non | ✅ Excellent | ✅ Bon |
| **Analyse de schéma** | ❌ Non | ✅ Oui | ✅ Oui |
| **Explain Plan visuel** | ❌ Non | ✅ Oui | ✅ Oui |
| **Agrégation visuelle** | ❌ Non | ✅ Oui | ✅ Oui |
| **Administration** | ✅ Complète | ⚠️ Partielle | ✅ Complète |
| **Gratuit** | ✅ Oui | ✅ Oui | ✅ Tier M0 |

### Quel outil choisir ?

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Guide de choix                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   Vous êtes débutant ?                                              │
│   └─► Commencez avec Compass pour explorer visuellement             │
│       puis apprenez mongosh progressivement                         │
│                                                                     │
│   Vous développez des scripts ou faites du DevOps ?                 │
│   └─► mongosh est indispensable                                     │
│                                                                     │
│   Vous voulez éviter la gestion d'infrastructure ?                  │
│   └─► Utilisez Atlas                                                │
│                                                                     │
│   Vous analysez des données ou créez des requêtes complexes ?       │
│   └─► Compass avec son constructeur d'agrégation                    │
│                                                                     │
│   En pratique : utilisez les trois !                                │
│   └─► Atlas pour l'hébergement                                      │
│   └─► Compass pour l'exploration                                    │
│   └─► mongosh pour les scripts et l'administration                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Autres outils utiles

### MongoDB Database Tools

Suite d'outils en ligne de commande pour l'import/export et la sauvegarde :

| Outil | Description |
|-------|-------------|
| `mongodump` | Exporter une base de données en format BSON |
| `mongorestore` | Restaurer une base depuis un dump BSON |
| `mongoexport` | Exporter en JSON ou CSV |
| `mongoimport` | Importer depuis JSON ou CSV |
| `mongostat` | Statistiques en temps réel |
| `mongotop` | Temps passé par collection |

```bash
# Exemples
mongodump --uri="mongodb://localhost:27017" --out=/backup
mongorestore --uri="mongodb://localhost:27017" /backup
mongoexport --db=test --collection=users --out=users.json
mongoimport --db=test --collection=users --file=users.json
```

### Mongo Express

Interface web légère pour MongoDB, idéale avec Docker :

```bash
docker run -d -p 8081:8081 \
  -e ME_CONFIG_MONGODB_URL="mongodb://host.docker.internal:27017" \
  mongo-express
```

Accessible sur `http://localhost:8081`

### Studio 3T (anciennement Robo 3T)

Alternative à Compass avec des fonctionnalités avancées :
- IntelliShell (autocomplétion avancée)
- Comparaison de données
- Migration SQL vers MongoDB
- Version gratuite disponible

---

## Conclusion

MongoDB offre un écosystème complet d'outils pour tous les profils d'utilisateurs :

- **mongosh** : Indispensable pour les scripts, l'automation et l'administration avancée
- **Compass** : Parfait pour l'exploration visuelle, l'apprentissage et l'analyse
- **Atlas** : Solution cloud complète pour éviter la gestion d'infrastructure

En tant que débutant, commencez par Compass pour vous familiariser avec MongoDB visuellement, puis apprenez mongosh pour maîtriser les opérations en ligne de commande. Si vous voulez un environnement sans configuration, Atlas avec son tier gratuit M0 est idéal.

---

## Points clés à retenir

- **mongosh** : Shell JavaScript interactif, idéal pour les scripts et l'administration
- **Compass** : Interface graphique pour explorer, requêter et analyser visuellement
- **Atlas** : Plateforme cloud managée avec tier gratuit (M0)
- Compass propose un **constructeur visuel d'agrégation** très utile
- Atlas offre des services additionnels : Search, Charts, Triggers, Functions
- Les trois outils sont **complémentaires** et peuvent être utilisés ensemble
- mongosh supporte les **fichiers de configuration** (`.mongoshrc.js`)
- Compass permet d'**analyser le schéma** automatiquement

---


⏭️ [Fondamentaux de MongoDB](/02-fondamentaux-de-mongodb/README.md)
