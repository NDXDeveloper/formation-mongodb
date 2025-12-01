🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 2.7 Introduction à MongoDB Compass

## Introduction

**MongoDB Compass** est l'interface graphique officielle de MongoDB. Si vous préférez une interface visuelle plutôt que la ligne de commande, Compass est fait pour vous ! C'est un outil puissant, intuitif et gratuit qui vous permet d'explorer, manipuler et optimiser vos données MongoDB sans écrire une seule ligne de code.

> **💡 Pensez à Compass comme :** L'équivalent de phpMyAdmin pour MySQL, mais pour MongoDB. C'est votre tableau de bord visuel pour gérer vos bases de données.

Dans cette section, nous allons découvrir :
- Installation et configuration de Compass
- L'interface utilisateur
- Navigation et exploration des données
- Création et modification de documents
- Requêtes visuelles
- Analyse de schéma
- Gestion des index
- Import/Export de données

---

## Qu'est-ce que MongoDB Compass ?

### Définition

**MongoDB Compass** est :
- 🖥️ Une **interface graphique** (GUI) officielle MongoDB
- 🔍 Un **explorateur de données** visuel et intuitif
- 📊 Un **outil d'analyse** de schéma et de performance
- 🛠️ Un **client de développement** complet
- 🆓 Un outil **gratuit** disponible sur toutes les plateformes

### Pourquoi Utiliser Compass ?

**Avantages :**

| Avantage | Description |
|----------|-------------|
| **Visuel** | Pas besoin de mémoriser les commandes |
| **Intuitif** | Interface moderne et facile à prendre en main |
| **Productif** | Créez des requêtes complexes sans code |
| **Éducatif** | Parfait pour apprendre MongoDB |
| **Complet** | Toutes les fonctionnalités essentielles intégrées |

**Cas d'usage idéaux :**
- 🎓 **Apprentissage** : Découvrir MongoDB visuellement
- 🔍 **Exploration** : Naviguer dans vos données
- 📝 **Prototypage** : Tester rapidement des requêtes
- 🐛 **Débogage** : Inspecter les données problématiques
- 📊 **Analyse** : Comprendre la structure de vos données

---

## Versions de Compass

MongoDB propose plusieurs versions de Compass :

### Compass (Version Complète)

**La version standard recommandée.**

- ✅ Toutes les fonctionnalités
- ✅ Gratuite
- ✅ Idéale pour le développement et la production

### Compass Isolated Edition

- Fonctionnalités limitées
- Pas de connexion externe (sécurité renforcée)
- Pour environnements hautement sécurisés

### Compass Readonly Edition

- Lecture seule
- Aucune modification possible
- Pour auditeurs ou analystes

> **💡 Recommandation :** Utilisez la version **Compass** complète sauf si vous avez des besoins spécifiques.

---

## Installation de MongoDB Compass

### Téléchargement

**Site officiel :** https://www.mongodb.com/try/download/compass

### Installation selon votre OS

**Windows :**
```
1. Télécharger l'installateur (.exe)
2. Double-cliquer sur le fichier
3. Suivre l'assistant d'installation
4. Lancer Compass depuis le menu Démarrer
```

**macOS :**
```
1. Télécharger le fichier .dmg
2. Ouvrir le fichier .dmg
3. Glisser MongoDB Compass dans Applications
4. Lancer depuis Applications ou Launchpad
```

**Linux (Ubuntu/Debian) :**
```bash
# Télécharger le package .deb
wget https://downloads.mongodb.com/compass/mongodb-compass_1.40.4_amd64.deb

# Installer
sudo dpkg -i mongodb-compass_1.40.4_amd64.deb

# Lancer
mongodb-compass
```

### Première Ouverture

Au premier lancement, Compass vous accueille avec :
- Un écran de connexion
- Des exemples de connexion
- Des options de confidentialité
- Un guide de démarrage rapide

---

## Interface Utilisateur

### Vue d'Ensemble

L'interface Compass est organisée en plusieurs zones principales :

```
┌─────────────────────────────────────────────────────────┐
│  [☰] MongoDB Compass            [?] [⚙️] [👤]           │  ← Barre supérieure
├─────────────────────────────────────────────────────────┤
│ Sidebar                │  Zone principale               │
│                        │                                │
│ • Connexions           │  Contenu selon la vue :        │
│ • Bases de données     │  - Liste des bases             │
│ • Collections          │  - Documents                   │
│ • Favoris              │  - Schéma                      │
│                        │  - Index                       │
│                        │  - etc.                        │
└─────────────────────────────────────────────────────────┘
```

### Barre Supérieure

**Éléments principaux :**
- **Menu hamburger (☰)** : Navigation principale
- **Nom de la connexion** : Serveur MongoDB actuel
- **Aide (?)** : Documentation et support
- **Paramètres (⚙️)** : Configuration de Compass
- **Profil (👤)** : Compte MongoDB (optionnel)

### Sidebar (Barre Latérale)

**Navigation hiérarchique :**
```
Connexion
├── Base_1
│   ├── Collection_A
│   ├── Collection_B
│   └── Collection_C
├── Base_2
│   ├── Collection_X
│   └── Collection_Y
└── Base_3
```

### Zone Principale

**Onglets disponibles (selon le contexte) :**
- **Documents** : Voir et éditer les documents
- **Aggregations** : Construire des pipelines d'agrégation
- **Schema** : Analyser la structure des données
- **Explain Plan** : Analyser les performances des requêtes
- **Indexes** : Gérer les index
- **Validation** : Définir des règles de validation

---

## Connexion à MongoDB

### Nouvelle Connexion

**Étape par étape :**

1. **Ouvrir Compass**
2. **Cliquer sur "New Connection"**
3. **Saisir la chaîne de connexion :**

```
mongodb://localhost:27017
```

### Types de Connexion

**Connexion Locale (par défaut) :**
```
mongodb://localhost:27017
```

**Avec authentification :**
```
mongodb://username:password@localhost:27017
```

**MongoDB Atlas (Cloud) :**
```
mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/
```

**Replica Set :**
```
mongodb://host1:27017,host2:27017,host3:27017/?replicaSet=myReplSet
```

### Options de Connexion

**Dans l'interface de connexion, vous pouvez configurer :**

**Onglet General :**
- Host : `localhost`
- Port : `27017`
- Authentication : `None` / `Username/Password` / `X.509` / etc.

**Onglet Advanced :**
- Read Preference : `Primary`, `Secondary`, etc.
- Replica Set Name : Nom du replica set
- SSL/TLS : Activer le chiffrement
- SSH Tunnel : Se connecter via SSH

**Onglet More Options :**
- Default Database : Base par défaut
- Connection Timeout : Délai de connexion
- Socket Timeout : Délai du socket

### Sauvegarder une Connexion

```
1. Configurer la connexion
2. Cliquer sur "Save & Connect"
3. Donner un nom : "Local Dev", "Production", "Atlas Cluster", etc.
4. Choisir une couleur (pour identifier visuellement)
5. Ajouter aux favoris si nécessaire
```

**Avantages :**
- 🚀 Reconnexion rapide
- 🎨 Code couleur pour les environnements
- ⭐ Favoris pour accès rapide
- 📝 Notes pour documenter

---

## Explorer les Données

### Vue Documents

**C'est la vue principale pour voir vos données.**

**Interface de la vue Documents :**
```
┌────────────────────────────────────────────────┐
│ [Filtre]  [Projet]  [Tri]  [Options]           │
├────────────────────────────────────────────────┤
│                                                │
│ Document 1                            [✏️] [🗑️] │
│ {                                              │
│   "_id": ObjectId("..."),                      │
│   "nom": "Dupont",                             │
│   "age": 30                                    │
│ }                                              │
│                                                │
│ Document 2                            [✏️] [🗑️] │
│ {                                              │
│   "_id": ObjectId("..."),                      │
│   "nom": "Martin",                             │
│   "age": 25                                    │
│ }                                              │
│                                                │
├────────────────────────────────────────────────┤
│ Page 1 sur 10      [◀️] [▶️]     20 documents    │
└────────────────────────────────────────────────┘
```

### Modes d'Affichage

**1. List View (Vue Liste - par défaut) :**
- Documents affichés les uns sous les autres
- Format JSON lisible
- Facilite la lecture

**2. JSON View (Vue JSON) :**
- Format JSON brut
- Copier-coller facile
- Syntaxe colorée

**3. Table View (Vue Tableau) :**
- Affichage tabulaire (comme Excel)
- Colonnes pour chaque champ
- Facilite la comparaison

### Navigation

**Contrôles de pagination :**
- **Flèches ◀️ ▶️** : Page précédente/suivante
- **Limite** : Nombre de documents par page (20, 50, 100)
- **Total** : Nombre total de documents

**Raccourcis clavier :**
- `Ctrl+F` : Rechercher dans la page
- `Ctrl+K` : Barre de commande rapide
- `Esc` : Fermer les modales

---

## Filtrer les Documents

### Barre de Filtre

**En haut de la vue Documents, vous trouvez la barre de filtre.**

**Exemples de filtres :**

**1. Filtre simple :**
```json
{ "age": 30 }
```
→ Trouve tous les documents où age = 30

**2. Filtre avec opérateur :**
```json
{ "age": { "$gte": 18 } }
```
→ Trouve les documents où age ≥ 18

**3. Filtre avec plusieurs conditions :**
```json
{ "age": { "$gte": 18 }, "ville": "Paris" }
```
→ Age ≥ 18 ET ville = Paris

**4. Filtre sur champ imbriqué :**
```json
{ "adresse.ville": "Lyon" }
```

**5. Filtre sur tableau :**
```json
{ "tags": "mongodb" }
```
→ Le tableau tags contient "mongodb"

### Mode Visuel du Filtre

Compass propose aussi un **constructeur de requête visuel** :

```
[+] Ajouter un filtre

Champ: [age ▼]  Opérateur: [≥ ▼]  Valeur: [18]
Champ: [ville ▼]  Opérateur: [= ▼]  Valeur: [Paris]

[Appliquer]  [Réinitialiser]
```

**Avantage :** Pas besoin de connaître la syntaxe JSON !

---

## Créer et Modifier des Documents

### Insérer un Document

**Méthode 1 : Bouton "Add Data" :**

```
1. Cliquer sur "Add Data" → "Insert Document"
2. Une modal s'ouvre avec un document vide :

{
  "_id": {
    "$oid": "..." (généré automatiquement)
  }
}

3. Ajouter vos champs :

{
  "_id": {
    "$oid": "..."
  },
  "nom": "Nouveau",
  "email": "nouveau@example.com",
  "age": 28
}

4. Cliquer sur "Insert"
```

**Méthode 2 : Importer des données :**
```
1. "Add Data" → "Import File"
2. Choisir JSON ou CSV
3. Sélectionner le fichier
4. Configurer les options d'import
5. Importer
```

### Modifier un Document

**Deux modes d'édition :**

**1. Mode Table (recommandé pour débutants) :**
```
1. Cliquer sur un document
2. Modifier directement les valeurs dans les champs
3. Cliquer sur "Update" pour sauvegarder
```

**2. Mode JSON :**
```
1. Cliquer sur l'icône d'édition (✏️)
2. Modifier le JSON complet
3. Cliquer sur "Update"
```

**Exemple d'édition :**
```json
// Avant
{
  "nom": "Dupont",
  "age": 30
}

// Après modification
{
  "nom": "Dupont",
  "age": 31,
  "ville": "Paris"  // Nouveau champ ajouté
}
```

### Supprimer un Document

```
1. Cliquer sur l'icône de suppression (🗑️)
2. Confirmer la suppression dans la popup
3. Le document est supprimé
```

**⚠️ Attention :** La suppression est définitive !

### Cloner un Document

```
1. Cliquer sur le document
2. Cliquer sur "Clone Document"
3. Un nouveau document identique est créé (avec un nouvel _id)
4. Modifier selon vos besoins
5. Insérer
```

---

## Projection et Tri

### Projection (Sélection de Champs)

**La projection vous permet de choisir quels champs afficher.**

**Dans la barre supérieure, onglet "Project" :**

```json
{
  "nom": 1,
  "email": 1
}
```
→ Affiche uniquement nom et email (+ _id par défaut)

```json
{
  "nom": 1,
  "email": 1,
  "_id": 0
}
```
→ Affiche nom et email sans _id

**Mode visuel :**
```
☑️ nom
☑️ email
☐ age
☐ ville
☐ _id
```

### Tri (Sort)

**Onglet "Sort" :**

```json
{ "age": 1 }
```
→ Tri croissant par age

```json
{ "age": -1 }
```
→ Tri décroissant par age

```json
{ "ville": 1, "age": -1 }
```
→ Tri par ville (asc), puis par age (desc)

**Mode visuel :**
```
Champ: [age ▼]  Ordre: [Décroissant ▼]

[Ajouter un tri]
```

---

## Analyse de Schéma

### Onglet "Schema"

**Compass peut analyser automatiquement la structure de vos documents.**

**Ce qu'il affiche :**

```
┌──────────────────────────────────────┐
│ Analyse de 1000 documents            │
├──────────────────────────────────────┤
│                                      │
│ _id (ObjectId)                       │
│ ━━━━━━━━━━━━━━━━━━━━━  100%          │
│ Présent dans tous les documents      │
│                                      │
│ nom (String)                         │
│ ━━━━━━━━━━━━━━━━━━━━━  98%           │
│ "Dupont", "Martin", "Bernard"...     │
│                                      │
│ age (Integer)                        │
│ ━━━━━━━━━━━━━━━━━━━━━  95%           │
│ Min: 18  Max: 65  Avg: 35.2          │
│                                      │
│ email (String)                       │
│ ━━━━━━━━━━━━━━━━━━━━━  100%          │
│ Tous uniques                         │
│                                      │
│ ville (String)                       │
│ ━━━━━━━━━━━━━━━━━━━━━  80%           │
│ "Paris", "Lyon", "Marseille"...      │
│                                      │
└──────────────────────────────────────┘
```

### Informations Fournies

Pour chaque champ :
- **Type de données** : String, Integer, ObjectId, etc.
- **Fréquence** : % de documents contenant ce champ
- **Valeurs** : Exemples ou statistiques
- **Distribution** : Graphique de répartition

### Utilité de l'Analyse

**Pourquoi c'est important :**
- 📊 **Comprendre vos données** : Structure et types
- 🔍 **Détecter les incohérences** : Champs manquants, types mixtes
- 🎯 **Optimiser** : Identifier les champs à indexer
- 📚 **Documenter** : Générer un schéma de référence

**Exemple d'insights :**
```
⚠️ Le champ "age" est présent dans seulement 80% des documents
   → Peut nécessiter un défaut ou une validation

⚠️ Le champ "prix" contient à la fois des String et des Number
   → Incohérence à corriger

✅ Le champ "email" est unique dans tous les documents
   → Bon candidat pour un index unique
```

---

## Gestion des Index

### Onglet "Indexes"

**Voir tous les index d'une collection.**

**Affichage typique :**
```
┌─────────────────────────────────────────────────┐
│ Index Name      │ Type     │ Size   │ Usage     │
├─────────────────────────────────────────────────┤
│ _id_            │ Single   │ 2.1 MB │ 100%      │
│ email_1         │ Single   │ 1.8 MB │ 85%       │
│ age_1_ville_1   │ Compound │ 3.2 MB │ 42%       │
└─────────────────────────────────────────────────┘

[+ Create Index]  [Refresh]
```

### Créer un Index

**Interface de création :**

```
1. Cliquer sur "Create Index"

2. Configurer l'index :

Nom: email_1
Champs: { "email": 1 }

Options:
☑️ Unique
☐ Partial
☐ TTL
☐ Sparse

3. Cliquer sur "Create Index"
```

**Exemple d'index composé :**
```json
{
  "ville": 1,
  "age": -1
}
```

**Options disponibles :**
- **Unique** : Valeurs uniques uniquement
- **Partial** : Index conditionnel
- **TTL** : Expiration automatique
- **Sparse** : Ignore les docs sans le champ
- **Text** : Index de recherche textuelle

### Analyser l'Utilisation

Compass affiche :
- **Taille de l'index** : Espace disque utilisé
- **Usage** : % d'utilisation dans les requêtes
- **Nombre d'accès** : Combien de fois l'index est utilisé

**Optimisation :**
```
✅ Index utilisé à 90% → Bien !
⚠️ Index utilisé à 5% → À supprimer ?
❌ Index jamais utilisé → Supprimer !
```

### Supprimer un Index

```
1. Cliquer sur l'icône de suppression (🗑️) à côté de l'index
2. Confirmer la suppression
```

**⚠️ Attention :** Ne supprimez pas l'index `_id_` (obligatoire) !

---

## Pipelines d'Agrégation

### Onglet "Aggregations"

**Interface de construction de pipeline visuelle.**

**Exemple de pipeline :**

```
Stage 1: $match
┌────────────────────────────┐
│ { "age": { "$gte": 18 } }  │
└────────────────────────────┘
        ↓
Stage 2: $group
┌──────────────────────────────────────┐
│ {                                    │
│   "_id": "$ville",                   │
│   "count": { "$sum": 1 },            │
│   "ageModyen": { "$avg": "$age" }    │
│ }                                    │
└──────────────────────────────────────┘
        ↓
Stage 3: $sort
┌────────────────────────────┐
│ { "count": -1 }            │
└────────────────────────────┘
```

### Créer un Pipeline

**Étape par étape :**

```
1. Cliquer sur "Add Stage"

2. Choisir le type : $match, $group, $project, etc.

3. Configurer en mode JSON ou visuel

4. Voir les résultats en temps réel

5. Ajouter d'autres stages si nécessaire

6. Exporter le code pour votre application
```

### Mode d'Édition

**Deux modes disponibles :**

**1. Mode Graphique :**
- Interface visuelle
- Glisser-déposer
- Pas besoin de connaître la syntaxe

**2. Mode JSON :**
- Syntaxe MongoDB pure
- Plus flexible
- Pour utilisateurs avancés

### Export du Pipeline

```
1. Une fois le pipeline créé
2. Cliquer sur "Export to Language"
3. Choisir votre langage :
   - Node.js
   - Python
   - Java
   - C#
   - etc.

4. Copier le code généré
5. Utiliser dans votre application
```

**Exemple de code généré (Node.js) :**
```javascript
const pipeline = [
  { $match: { age: { $gte: 18 } } },
  { $group: { _id: "$ville", count: { $sum: 1 } } },
  { $sort: { count: -1 } }
]

const aggCursor = collection.aggregate(pipeline)
```

---

## Explain Plan (Analyse de Performance)

### Onglet "Explain Plan"

**Analyse comment MongoDB exécute vos requêtes.**

**Informations affichées :**

```
┌─────────────────────────────────────────┐
│ Query Performance                       │
├─────────────────────────────────────────┤
│                                         │
│ Execution Time: 15 ms                   │
│ Documents Examined: 1,250               │
│ Documents Returned: 42                  │
│ Index Used: email_1                     │
│                                         │
│ ✅ Query uses index                     │
│ ✅ Efficient                            │
│                                         │
├─────────────────────────────────────────┤
│ Execution Stats                         │
│                                         │
│ Stage: IXSCAN                           │
│ Index: email_1                          │
│ Keys Examined: 42                       │
│                                         │
│ Stage: FETCH                            │
│ Documents: 42                           │
│                                         │
└─────────────────────────────────────────┘
```

### Interpréter les Résultats

**Indicateurs clés :**

| Indicateur | Signification | Optimal |
|------------|---------------|---------|
| **Execution Time** | Temps d'exécution | < 100ms |
| **Docs Examined** | Documents scannés | ≈ Docs Returned |
| **Index Used** | Index utilisé | Oui ✅ |
| **Stage COLLSCAN** | Scan complet | Non ❌ |
| **Stage IXSCAN** | Scan d'index | Oui ✅ |

**Bonnes performances :**
```
✅ Execution Time: 5 ms
✅ Documents Examined: 10
✅ Documents Returned: 10
✅ Index Used: email_1
```

**Mauvaises performances :**
```
❌ Execution Time: 5000 ms
❌ Documents Examined: 1,000,000
❌ Documents Returned: 10
❌ No index used (COLLSCAN)
```

### Optimisation

**Si les performances sont mauvaises :**

1. **Créer un index** sur les champs filtrés
2. **Modifier la requête** pour utiliser les index existants
3. **Limiter les résultats** avec `.limit()`
4. **Utiliser des projections** pour réduire les données

---

## Validation de Schéma

### Onglet "Validation"

**Définir des règles de validation pour vos documents.**

**Interface de configuration :**

```
┌─────────────────────────────────────────────┐
│ Validation Rules                            │
├─────────────────────────────────────────────┤
│                                             │
│ {                                           │
│   "$jsonSchema": {                          │
│     "bsonType": "object",                   │
│     "required": ["nom", "email"],           │
│     "properties": {                         │
│       "nom": {                              │
│         "bsonType": "string",               │
│         "minLength": 2                      │
│       },                                    │
│       "email": {                            │
│         "bsonType": "string",               │
│         "pattern": "^.+@.+\\..+$"           │
│       },                                    │
│       "age": {                              │
│         "bsonType": "int",                  │
│         "minimum": 0,                       │
│         "maximum": 150                      │
│       }                                     │
│     }                                       │
│   }                                         │
│ }                                           │
│                                             │
├─────────────────────────────────────────────┤
│ Validation Level: [ Strict ▼ ]              │
│ Validation Action: [ Error ▼ ]              │
│                                             │
│ [Save]  [Test]  [Cancel]                    │
└─────────────────────────────────────────────┘
```

### Tester la Validation

**Compass permet de tester vos règles :**

```
1. Définir les règles de validation
2. Cliquer sur "Test"
3. Entrer un document de test :

{
  "nom": "D",         // ❌ Trop court (min: 2)
  "email": "invalid", // ❌ Format invalide
  "age": 200          // ❌ Trop grand (max: 150)
}

4. Voir les erreurs de validation
5. Corriger et retester
```

---

## Import et Export

### Importer des Données

**Formats supportés :**
- JSON (documents MongoDB)
- CSV (données tabulaires)

**Procédure d'import :**

```
1. Sélectionner une collection
2. Cliquer sur "Add Data" → "Import File"
3. Choisir le fichier
4. Configurer les options :
   - Délimiteur (pour CSV)
   - Mapping des champs
   - Ignorer les erreurs
5. Prévisualiser
6. Importer
```

**Exemple CSV :**
```csv
nom,email,age
Dupont,dupont@example.com,30
Martin,martin@example.com,25
Bernard,bernard@example.com,35
```

### Exporter des Données

**Formats d'export :**
- JSON (documents complets)
- CSV (format tabulaire)

**Procédure d'export :**

```
1. Filtrer les documents à exporter (optionnel)
2. Cliquer sur "Export Data"
3. Choisir le format : JSON ou CSV
4. Configurer les options :
   - Tous les documents ou filtrés uniquement
   - Tous les champs ou projection
5. Choisir le nom et l'emplacement du fichier
6. Exporter
```

**Options d'export :**
- **Exporter la requête** : Inclure le filtre dans le fichier
- **Exporter tout** : Tous les documents de la collection
- **Limiter** : Exporter seulement X documents

---

## Fonctionnalités Avancées

### Workspace (Espace de Travail)

**Sauvegarder vos requêtes et pipelines.**

```
1. Créer une requête complexe
2. Cliquer sur "Save"
3. Donner un nom : "Utilisateurs majeurs à Paris"
4. Retrouver dans "My Queries"
5. Réutiliser d'un clic
```

### Favoris

**Marquer des collections en favori :**

```
1. Clic droit sur une collection
2. "Add to Favorites"
3. La collection apparaît en haut de la sidebar
4. Accès rapide
```

### Mode Sombre

```
Settings ⚙️ → Theme → Dark
```

**Confort visuel :** Réduit la fatigue oculaire en travaillant la nuit 🌙

### Raccourcis Clavier

| Raccourci | Action |
|-----------|--------|
| `Ctrl+K` | Barre de commande |
| `Ctrl+F` | Rechercher |
| `Ctrl+,` | Paramètres |
| `Ctrl+N` | Nouvelle connexion |
| `Ctrl+W` | Fermer l'onglet |
| `Ctrl+T` | Nouvel onglet |

### Plugins et Extensions

Compass supporte des plugins pour étendre ses fonctionnalités :
- Plugins de validation personnalisés
- Plugins d'analyse de données
- Plugins de visualisation

---

## Compass vs mongosh

### Comparaison

| Aspect | MongoDB Compass | mongosh |
|--------|----------------|---------|
| **Interface** | Graphique (GUI) | Ligne de commande |
| **Courbe d'apprentissage** | Facile 🟢 | Moyenne 🟡 |
| **Productivité débutants** | Élevée ✅ | Moyenne |
| **Productivité experts** | Moyenne | Élevée ✅ |
| **Visualisation** | Excellente 📊 | Limitée |
| **Scripting** | Non | Oui ✅ |
| **Automatisation** | Limitée | Excellente ✅ |
| **Performance** | Bonne | Excellente |

### Quand Utiliser Compass ?

✅ **Utilisez Compass pour :**
- Explorer et découvrir vos données
- Créer des requêtes visuellement
- Analyser le schéma de vos collections
- Prototyper rapidement
- Déboguer des problèmes de données
- Présenter des données à des non-techniques

### Quand Utiliser mongosh ?

✅ **Utilisez mongosh pour :**
- Automatiser des tâches (scripts)
- Opérations rapides en ligne de commande
- Administration serveur
- Déploiements en production
- CI/CD et DevOps
- Quand vous êtes à l'aise avec la CLI

### Approche Hybride

**La meilleure approche : Utiliser les deux !**

```
🖥️ Compass pour :
   - Développement
   - Exploration
   - Prototypage

⌨️ mongosh pour :
   - Scripts
   - Automatisation
   - Production
```

---

## Performances et Limitations

### Performances

**Compass est optimisé pour :**
- ✅ Collections de taille moyenne (< 1 million de docs)
- ✅ Requêtes simples et modérées
- ✅ Exploration et développement

**Peut être lent avec :**
- ⚠️ Très grosses collections (> 10 millions)
- ⚠️ Requêtes très complexes
- ⚠️ Agrégations lourdes sur tous les documents

### Limitations

**Quota de requêtes :**
- Compass limite le nombre de documents affichés par page
- Par défaut : 20 documents
- Maximum : 1000 documents par page

**Pas pour la production :**
- Compass n'est pas conçu pour gérer la production
- Utilisez les drivers dans vos applications
- Compass = outil de développement et d'administration

---

## Bonnes Pratiques

### ✅ À Faire

1. **Utilisez des connexions sauvegardées**
   ```
   - Nommez vos connexions clairement
   - Utilisez des couleurs pour les environnements
   - "Dev Local" (vert), "Prod" (rouge), etc.
   ```

2. **Exploitez l'analyse de schéma**
   ```
   - Lancez-la régulièrement
   - Détectez les incohérences tôt
   - Optimisez vos modèles
   ```

3. **Testez vos requêtes avec Explain**
   ```
   - Vérifiez les performances AVANT la production
   - Créez des index si nécessaire
   - Optimisez les requêtes lentes
   ```

4. **Sauvegardez vos requêtes fréquentes**
   ```
   - Créez une bibliothèque de requêtes
   - Partagez avec l'équipe
   - Gagnez du temps
   ```

5. **Utilisez les filtres avant d'exporter**
   ```
   - N'exportez que ce dont vous avez besoin
   - Économisez du temps et de l'espace
   ```

### ❌ À Éviter

1. **Ne faites pas de modifications directes en production**
   ```
   ❌ Modifier des documents critiques via Compass en prod
   ✅ Utilisez des scripts testés avec des sauvegardes
   ```

2. **N'analysez pas de très grosses collections**
   ```
   ❌ Analyser 100 millions de documents
   ✅ Échantillonner ou utiliser des outils dédiés
   ```

3. **Ne supprimez pas d'index sans vérifier**
   ```
   ❌ Supprimer un index utilisé
   ✅ Vérifier l'utilisation avec Explain d'abord
   ```

4. **N'importez pas sans vérifier**
   ```
   ❌ Importer 1 million de lignes sans validation
   ✅ Tester avec un petit échantillon d'abord
   ```

---

## Dépannage

### Problèmes de Connexion

**Compass ne se connecte pas :**

```
❌ Problème : "Connection refused"
✅ Solutions :
   - Vérifier que MongoDB est démarré
   - Vérifier l'hôte et le port
   - Vérifier le pare-feu
   - Tester avec mongosh d'abord
```

**Authentification échoue :**

```
❌ Problème : "Authentication failed"
✅ Solutions :
   - Vérifier username/password
   - Vérifier la base d'authentification
   - Vérifier les privilèges utilisateur
```

### Performances Lentes

**Compass est lent :**

```
✅ Solutions :
   - Réduire le nombre de documents par page
   - Utiliser des filtres pour limiter les données
   - Fermer les onglets inutilisés
   - Redémarrer Compass
   - Vérifier les ressources système (RAM, CPU)
```

### Erreurs d'Import

**L'import échoue :**

```
✅ Solutions :
   - Vérifier le format du fichier (JSON/CSV)
   - Vérifier l'encodage (UTF-8)
   - Valider la structure des données
   - Importer par petits lots
   - Vérifier les logs d'erreur
```

---

## Points Clés à Retenir

### ✅ Essentiel

1. **Compass = GUI officielle** : Interface visuelle pour MongoDB
2. **Gratuit et multiplateforme** : Windows, macOS, Linux
3. **Exploration visuelle** : Parfait pour comprendre vos données
4. **Analyse de schéma** : Détecte automatiquement la structure
5. **Gestion des index** : Créer, analyser, optimiser
6. **Pipelines visuels** : Construire des agrégations sans code
7. **Export de code** : Générer du code pour vos applications

### 🎯 Cas d'Usage Idéaux

- 🎓 **Apprentissage** : Découvrir MongoDB
- 🔍 **Exploration** : Comprendre vos données
- 🐛 **Débogage** : Trouver et corriger des problèmes
- 📊 **Analyse** : Étudier la structure et les performances
- 🚀 **Prototypage** : Tester rapidement des idées

### ⚠️ Limitations

- Pas adapté pour les très grosses collections
- Pas pour l'automatisation (utilisez mongosh)
- Pas pour la production (utilisez les drivers)

---

## Ressources Complémentaires

### Documentation Officielle

- [MongoDB Compass Documentation](https://docs.mongodb.com/compass/)
- [Compass Download](https://www.mongodb.com/try/download/compass)
- [Compass Tutorial](https://docs.mongodb.com/compass/current/tutorial/)

### Tutoriels Vidéo

- MongoDB University (cours gratuits)
- YouTube : MongoDB Official Channel
- Tutoriels interactifs sur le site MongoDB

### Alternatives

- **Studio 3T** : Client tiers payant avec plus de fonctionnalités
- **Robo 3T** : Client léger et gratuit
- **NoSQLBooster** : Client avec support SQL

---


⏭️ [Requêtes et Filtres](/03-requetes-et-filtres/README.md)
