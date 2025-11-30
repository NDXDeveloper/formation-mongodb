🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.1 Qu'est-ce que MongoDB ?

## Introduction

MongoDB est une **base de données NoSQL** orientée documents, conçue pour stocker, gérer et interroger de grandes quantités de données de manière flexible et performante. Créée en 2007 par la société 10gen (devenue MongoDB Inc.), elle est aujourd'hui l'une des bases de données les plus populaires au monde.

Contrairement aux bases de données relationnelles traditionnelles comme MySQL ou PostgreSQL qui organisent les données dans des tables avec des lignes et des colonnes, MongoDB stocke les données sous forme de **documents** dans un format proche du JSON.

---

## Comprendre le concept de base de données orientée documents

### Qu'est-ce qu'un document ?

Dans MongoDB, un **document** est l'unité de base pour stocker des données. Il ressemble à un objet JSON (JavaScript Object Notation) et peut contenir différents types d'informations organisées en paires clé-valeur.

Voici un exemple simple d'un document représentant un utilisateur :

```json
{
  "_id": "507f1f77bcf86cd799439011",
  "nom": "Dupont",
  "prenom": "Marie",
  "age": 28,
  "email": "marie.dupont@email.com",
  "adresse": {
    "rue": "15 rue de la Paix",
    "ville": "Paris",
    "codePostal": "75001"
  },
  "hobbies": ["lecture", "voyage", "photographie"]
}
```

Comme vous pouvez le constater, un document peut contenir :

- Des valeurs simples (texte, nombres)
- Des objets imbriqués (comme l'adresse)
- Des tableaux (comme la liste des hobbies)

Cette flexibilité est l'un des grands atouts de MongoDB.

### Qu'est-ce qu'une collection ?

Les documents sont regroupés dans des **collections**. Une collection est l'équivalent d'une table dans une base de données relationnelle, mais sans schéma rigide.

Par exemple, vous pourriez avoir :

- Une collection `utilisateurs` contenant tous les documents d'utilisateurs
- Une collection `produits` contenant tous les documents de produits
- Une collection `commandes` contenant tous les documents de commandes

### Qu'est-ce qu'une base de données ?

Une **base de données** MongoDB regroupe plusieurs collections. Une instance MongoDB peut héberger plusieurs bases de données, chacune contenant ses propres collections.

```
Instance MongoDB
├── Base de données "boutique_en_ligne"
│   ├── Collection "utilisateurs"
│   ├── Collection "produits"
│   └── Collection "commandes"
└── Base de données "blog"
    ├── Collection "articles"
    ├── Collection "commentaires"
    └── Collection "auteurs"
```

---

## Caractéristiques principales de MongoDB

### 1. Schéma flexible

Contrairement aux bases de données relationnelles où vous devez définir la structure de vos tables à l'avance, MongoDB permet une grande flexibilité. Les documents d'une même collection peuvent avoir des structures différentes.

Par exemple, dans une collection d'utilisateurs :

```json
// Document 1 : utilisateur particulier
{
  "nom": "Martin",
  "email": "martin@email.com"
}

// Document 2 : utilisateur professionnel avec plus d'informations
{
  "nom": "Société ABC",
  "email": "contact@abc.com",
  "siret": "12345678901234",
  "secteur": "Technologie",
  "nombreEmployes": 50
}
```

Cette flexibilité est particulièrement utile lorsque vos données évoluent au fil du temps.

### 2. Format BSON

MongoDB stocke les données en **BSON** (Binary JSON), une représentation binaire du format JSON. BSON offre plusieurs avantages :

- Stockage et transmission plus efficaces
- Support de types de données supplémentaires (dates, données binaires, ObjectId, etc.)
- Facilité de parcours et d'indexation

### 3. Requêtes puissantes

MongoDB propose un langage de requête riche permettant de :

- Filtrer les documents selon de nombreux critères
- Trier et paginer les résultats
- Effectuer des agrégations complexes
- Rechercher dans des documents imbriqués et des tableaux

### 4. Haute disponibilité

MongoDB intègre nativement des mécanismes de **réplication** (Replica Sets) permettant de maintenir plusieurs copies de vos données sur différents serveurs. En cas de panne d'un serveur, un autre prend automatiquement le relais.

### 5. Scalabilité horizontale

Grâce au **sharding** (partitionnement), MongoDB peut distribuer les données sur plusieurs serveurs pour gérer des volumes de données très importants et des charges de travail élevées.

### 6. Indexation

MongoDB supporte de nombreux types d'index pour accélérer les requêtes :

- Index simples et composés
- Index textuels pour la recherche full-text
- Index géospatiaux pour les données de localisation
- Et bien d'autres

---

## Pourquoi choisir MongoDB ?

### Avantages

| Avantage | Description |
|----------|-------------|
| **Flexibilité du schéma** | Adaptez facilement votre modèle de données sans migrations complexes |
| **Performance** | Excellentes performances en lecture et écriture grâce à l'architecture orientée documents |
| **Scalabilité** | Évoluez horizontalement en ajoutant des serveurs selon vos besoins |
| **Facilité de développement** | Le format JSON/BSON s'intègre naturellement avec les langages de programmation modernes |
| **Écosystème riche** | Drivers officiels pour tous les langages populaires, outils graphiques, cloud (Atlas) |
| **Documentation complète** | Documentation officielle exhaustive et communauté active |

### Cas d'utilisation courants

MongoDB excelle dans de nombreux contextes :

- **Applications web et mobiles** : Stockage de profils utilisateurs, sessions, préférences
- **Gestion de contenu** : CMS, blogs, catalogues de produits
- **Internet des objets (IoT)** : Collecte et analyse de données de capteurs
- **Analyse en temps réel** : Tableaux de bord, métriques, logs
- **Applications de gaming** : Scores, progressions, inventaires de joueurs
- **E-commerce** : Catalogues produits avec attributs variables

---

## MongoDB vs Bases de données relationnelles

Pour mieux comprendre MongoDB, comparons-le avec les bases de données relationnelles traditionnelles :

| Concept relationnel | Équivalent MongoDB |
|--------------------|--------------------|
| Base de données | Base de données |
| Table | Collection |
| Ligne (row) | Document |
| Colonne | Champ (field) |
| Clé primaire | `_id` (généré automatiquement) |
| Jointure (JOIN) | Documents imbriqués ou `$lookup` |
| Index | Index |

### Exemple comparatif

**Dans une base relationnelle (SQL)**, vous auriez typiquement plusieurs tables liées :

```
Table "utilisateurs"
+----+--------+-------------------+
| id | nom    | email             |
+----+--------+-------------------+
| 1  | Dupont | dupont@email.com  |
+----+--------+-------------------+

Table "adresses"
+----+---------------+--------+--------------+
| id | utilisateur_id| ville  | code_postal  |
+----+---------------+--------+--------------+
| 1  | 1             | Paris  | 75001        |
+----+---------------+--------+--------------+
```

**Dans MongoDB**, ces informations peuvent être regroupées dans un seul document :

```json
{
  "_id": ObjectId("..."),
  "nom": "Dupont",
  "email": "dupont@email.com",
  "adresse": {
    "ville": "Paris",
    "codePostal": "75001"
  }
}
```

Cette approche évite les jointures coûteuses et simplifie souvent le code applicatif.

---

## L'écosystème MongoDB

MongoDB ne se limite pas au moteur de base de données. L'écosystème comprend :

### Outils essentiels

- **mongosh** : Le shell interactif pour interagir avec MongoDB en ligne de commande
- **MongoDB Compass** : Interface graphique pour explorer et manipuler vos données
- **MongoDB Atlas** : Service cloud entièrement géré pour déployer MongoDB sans gérer l'infrastructure

### Drivers officiels

MongoDB propose des drivers officiels pour les principaux langages de programmation :

- JavaScript / Node.js
- Python (PyMongo)
- Java
- C# / .NET
- Go
- PHP
- Ruby
- Et bien d'autres

Ces drivers permettent d'intégrer MongoDB dans vos applications de manière native et performante.

---

## Conclusion

MongoDB est une base de données moderne, flexible et puissante qui répond aux besoins des applications actuelles. Son modèle orienté documents permet de stocker des données de manière naturelle et intuitive, tandis que ses fonctionnalités de réplication et de sharding assurent haute disponibilité et scalabilité.

Dans les sections suivantes, nous explorerons l'historique de MongoDB, comparerons plus en détail les bases NoSQL et SQL, puis nous passerons à l'installation et à la prise en main pratique.

---

## Points clés à retenir

- MongoDB est une base de données **NoSQL orientée documents**
- Les données sont stockées en **documents BSON** (similaires au JSON)
- Les documents sont regroupés dans des **collections**
- Le schéma est **flexible** : les documents peuvent avoir des structures différentes
- MongoDB offre **haute disponibilité** (réplication) et **scalabilité** (sharding)
- L'écosystème inclut des outils graphiques, un cloud managé et des drivers pour tous les langages

---

⏭️ [Historique et évolution](/01-introduction-a-mongodb/02-historique-et-evolution.md)
