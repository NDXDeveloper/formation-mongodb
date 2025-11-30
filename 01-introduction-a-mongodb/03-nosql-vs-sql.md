🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 1.3 Bases de données NoSQL vs SQL : Comparaison conceptuelle

## Introduction

Lorsque vous débutez avec MongoDB, l'une des premières questions qui se pose est : "Quelle est la différence entre les bases de données SQL et NoSQL ?" Cette section vous aidera à comprendre ces deux approches, leurs forces respectives et les situations où chacune excelle.

Comprendre cette distinction est essentiel pour faire des choix éclairés dans vos projets et tirer le meilleur parti de MongoDB.

---

## Qu'est-ce qu'une base de données SQL ?

### Définition

Les bases de données **SQL** (Structured Query Language) sont également appelées bases de données **relationnelles**. Elles organisent les données dans des **tables** structurées, composées de **lignes** (enregistrements) et de **colonnes** (attributs).

### Caractéristiques principales

- **Schéma fixe** : La structure des tables doit être définie avant d'insérer des données
- **Relations** : Les tables sont liées entre elles par des clés étrangères
- **Langage SQL** : Un langage standardisé pour interroger et manipuler les données
- **Transactions ACID** : Garanties fortes de cohérence des données

### Exemples de bases SQL populaires

| Base de données | Description |
|-----------------|-------------|
| **MySQL** | Open source, très répandue pour le web |
| **PostgreSQL** | Open source, riche en fonctionnalités |
| **Oracle Database** | Solution entreprise propriétaire |
| **Microsoft SQL Server** | Solution Microsoft pour l'entreprise |
| **SQLite** | Base légère embarquée |
| **MariaDB** | Fork de MySQL, open source |

### Exemple de structure SQL

Imaginons une application de gestion de commandes. En SQL, vous auriez typiquement plusieurs tables liées :

```sql
-- Table des clients
CREATE TABLE clients (
    id INT PRIMARY KEY,
    nom VARCHAR(100),
    email VARCHAR(100),
    date_inscription DATE
);

-- Table des commandes
CREATE TABLE commandes (
    id INT PRIMARY KEY,
    client_id INT,
    date_commande DATE,
    montant_total DECIMAL(10,2),
    FOREIGN KEY (client_id) REFERENCES clients(id)
);

-- Table des produits commandés
CREATE TABLE lignes_commande (
    id INT PRIMARY KEY,
    commande_id INT,
    produit_nom VARCHAR(100),
    quantite INT,
    prix_unitaire DECIMAL(10,2),
    FOREIGN KEY (commande_id) REFERENCES commandes(id)
);
```

Pour récupérer une commande complète, vous devez joindre ces tables :

```sql
SELECT c.nom, cmd.date_commande, lc.produit_nom, lc.quantite
FROM clients c
JOIN commandes cmd ON c.id = cmd.client_id
JOIN lignes_commande lc ON cmd.id = lc.commande_id
WHERE cmd.id = 123;
```

---

## Qu'est-ce qu'une base de données NoSQL ?

### Définition

**NoSQL** signifie "Not Only SQL" (et non pas "No SQL"). Ce terme regroupe diverses bases de données qui s'éloignent du modèle relationnel traditionnel pour offrir plus de flexibilité et de scalabilité.

### Les différentes familles NoSQL

Le terme NoSQL englobe plusieurs types de bases de données, chacune avec son propre modèle de données :

#### 1. Bases orientées documents

Stockent les données sous forme de documents (JSON, BSON, XML).

| Base de données | Description |
|-----------------|-------------|
| **MongoDB** | Leader du marché, documents BSON |
| **CouchDB** | Documents JSON, synchronisation |
| **Amazon DocumentDB** | Compatible MongoDB sur AWS |

#### 2. Bases clé-valeur

Stockent des paires clé-valeur simples, très rapides en lecture/écriture.

| Base de données | Description |
|-----------------|-------------|
| **Redis** | En mémoire, très performante |
| **Amazon DynamoDB** | Service managé AWS |
| **etcd** | Utilisée par Kubernetes |

#### 3. Bases orientées colonnes

Optimisées pour les requêtes analytiques sur de grandes quantités de données.

| Base de données | Description |
|-----------------|-------------|
| **Apache Cassandra** | Distribuée, haute disponibilité |
| **HBase** | Sur Hadoop, big data |
| **ScyllaDB** | Compatible Cassandra, très performante |

#### 4. Bases orientées graphes

Optimisées pour les données fortement connectées (réseaux sociaux, recommandations).

| Base de données | Description |
|-----------------|-------------|
| **Neo4j** | Leader des bases graphes |
| **Amazon Neptune** | Service managé AWS |
| **ArangoDB** | Multi-modèle |

### MongoDB : Une base orientée documents

MongoDB appartient à la famille des bases **orientées documents**. Voici comment la même commande serait représentée :

```json
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "client": {
    "nom": "Dupont",
    "email": "dupont@email.com",
    "date_inscription": ISODate("2024-01-15")
  },
  "date_commande": ISODate("2024-03-20"),
  "montant_total": 150.00,
  "lignes": [
    {
      "produit_nom": "Clavier mécanique",
      "quantite": 1,
      "prix_unitaire": 89.99
    },
    {
      "produit_nom": "Souris ergonomique",
      "quantite": 2,
      "prix_unitaire": 30.00
    }
  ]
}
```

Toutes les informations sont regroupées dans un seul document, éliminant le besoin de jointures.

---

## Comparaison détaillée : SQL vs NoSQL

### 1. Modèle de données

| Aspect | SQL (Relationnel) | NoSQL (MongoDB) |
|--------|-------------------|-----------------|
| **Structure** | Tables avec lignes et colonnes | Documents JSON/BSON |
| **Schéma** | Fixe, défini à l'avance | Flexible, peut varier par document |
| **Relations** | Clés étrangères, jointures | Documents imbriqués ou références |
| **Normalisation** | Données normalisées (éviter la redondance) | Données souvent dénormalisées |

#### Illustration du schéma

**SQL - Schéma rigide :**
```
┌─────────────────────────────────┐
│         Table "users"           │
├────────┬──────────┬─────────────┤
│   id   │   name   │    email    │  ← Toutes les lignes ont
├────────┼──────────┼─────────────┤    les mêmes colonnes
│   1    │  Alice   │ alice@...   │
│   2    │   Bob    │  bob@...    │
└────────┴──────────┴─────────────┘
```

**MongoDB - Schéma flexible :**
```
┌─────────────────────────────────────────┐
│           Collection "users"            │
├─────────────────────────────────────────┤
│ { name: "Alice", email: "alice@..." }   │
├─────────────────────────────────────────┤
│ { name: "Bob", email: "bob@...",        │
│   phone: "0612345678",                  │  ← Ce document a des
│   preferences: { theme: "dark" } }      │    champs supplémentaires
└─────────────────────────────────────────┘
```

### 2. Langage de requête

| Aspect | SQL | MongoDB |
|--------|-----|---------|
| **Syntaxe** | Langage déclaratif standardisé | API basée sur des documents/objets |
| **Apprentissage** | Un seul langage à maîtriser | S'intègre naturellement avec le code |
| **Jointures** | Natives et puissantes | Possibles avec `$lookup`, mais moins courantes |

#### Exemples comparatifs

**Recherche simple :**

```sql
-- SQL
SELECT * FROM users WHERE age > 25;
```

```javascript
// MongoDB
db.users.find({ age: { $gt: 25 } })
```

**Recherche avec conditions multiples :**

```sql
-- SQL
SELECT name, email FROM users
WHERE age > 25 AND city = 'Paris'
ORDER BY name;
```

```javascript
// MongoDB
db.users.find(
  { age: { $gt: 25 }, city: "Paris" },
  { name: 1, email: 1 }
).sort({ name: 1 })
```

**Agrégation :**

```sql
-- SQL
SELECT city, COUNT(*) as total, AVG(age) as age_moyen
FROM users
GROUP BY city
HAVING COUNT(*) > 10;
```

```javascript
// MongoDB
db.users.aggregate([
  { $group: {
      _id: "$city",
      total: { $sum: 1 },
      age_moyen: { $avg: "$age" }
  }},
  { $match: { total: { $gt: 10 } } }
])
```

### 3. Scalabilité

| Aspect | SQL | NoSQL (MongoDB) |
|--------|-----|-----------------|
| **Scalabilité verticale** | Principale approche | Supportée |
| **Scalabilité horizontale** | Complexe à mettre en œuvre | Native (sharding) |
| **Distribution des données** | Difficile | Conçue pour |

#### Comprendre les types de scalabilité

**Scalabilité verticale (Scale Up) :**
- Ajouter plus de ressources à un seul serveur (CPU, RAM, stockage)
- Limitée par le matériel disponible
- Approche traditionnelle des bases SQL

```
Avant           Après
┌─────────┐     ┌─────────────┐
│ Serveur │ →   │   Serveur   │
│  4 CPU  │     │   16 CPU    │
│  8 GB   │     │   64 GB     │
└─────────┘     └─────────────┘
```

**Scalabilité horizontale (Scale Out) :**
- Ajouter plus de serveurs
- Théoriquement illimitée
- Approche native de MongoDB (sharding)

```
Avant              Après
┌─────────┐        ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Serveur │   →    │ Serveur │ │ Serveur │ │ Serveur │
└─────────┘        └─────────┘ └─────────┘ └─────────┘
                   Les données sont réparties
```

### 4. Transactions et cohérence

| Aspect | SQL | NoSQL (MongoDB) |
|--------|-----|-----------------|
| **ACID** | Toujours garanti | Garanti au niveau document, multi-documents depuis v4.0 |
| **Cohérence** | Forte par défaut | Configurable (forte à éventuelle) |
| **Isolation** | Niveaux multiples | Snapshot isolation |

> **Note pour les débutants** : ACID signifie Atomicité, Cohérence, Isolation, Durabilité. Ces propriétés garantissent que les transactions sont fiables. Nous les détaillerons dans la section sur les fondements théoriques.

### 5. Performance

| Aspect | SQL | NoSQL (MongoDB) |
|--------|-----|-----------------|
| **Lectures simples** | Bonnes | Excellentes (pas de jointures) |
| **Lectures complexes (jointures)** | Optimisées | Moins performantes |
| **Écritures** | Bonnes | Excellentes |
| **Données non structurées** | Difficile | Naturel |

### 6. Cas d'utilisation typiques

| SQL est préférable pour | MongoDB est préférable pour |
|-------------------------|----------------------------|
| Données fortement structurées | Données semi-structurées ou variables |
| Relations complexes entre entités | Documents autonomes |
| Transactions financières critiques | Applications web/mobile à fort trafic |
| Reporting et BI traditionnels | Prototypage rapide |
| Systèmes legacy | Applications temps réel |
| Conformité réglementaire stricte | IoT et données de capteurs |

---

## Mythes et réalités

### Mythe 1 : "NoSQL signifie pas de SQL"

**Réalité** : NoSQL signifie "Not Only SQL". Beaucoup de bases NoSQL, dont MongoDB, supportent des fonctionnalités traditionnellement associées au SQL (jointures, transactions).

### Mythe 2 : "MongoDB ne supporte pas les transactions"

**Réalité** : Depuis la version 4.0 (2018), MongoDB supporte les transactions ACID multi-documents. Depuis la version 4.2, ces transactions fonctionnent même sur des clusters distribués (sharded).

### Mythe 3 : "Les bases NoSQL ne sont pas fiables pour les données critiques"

**Réalité** : MongoDB est utilisé par des banques, des entreprises de santé et des gouvernements pour des données critiques. Les fonctionnalités de réplication et de durabilité garantissent la fiabilité.

### Mythe 4 : "SQL et NoSQL sont mutuellement exclusifs"

**Réalité** : De nombreuses architectures modernes utilisent les deux approches. On parle de "polyglot persistence" : utiliser la base de données la plus adaptée à chaque besoin.

### Mythe 5 : "NoSQL est toujours plus rapide"

**Réalité** : Les performances dépendent du cas d'utilisation. Les bases SQL excellent pour les requêtes analytiques complexes avec de nombreuses jointures. MongoDB excelle pour les lectures/écritures simples sur des documents.

---

## Quand choisir SQL ?

Les bases de données relationnelles restent le meilleur choix dans plusieurs situations :

### 1. Données hautement structurées et stables

Si votre modèle de données est bien défini et change rarement, un schéma SQL rigide offre des garanties de qualité des données.

```
Exemple : Système comptable
- Les écritures comptables ont toujours la même structure
- Les règles de validation sont strictes
- La cohérence est primordiale
```

### 2. Relations complexes entre entités

Quand vos données sont fortement interconnectées avec de nombreuses relations many-to-many, le modèle relationnel est souvent plus efficace.

```
Exemple : Système de gestion universitaire
- Étudiants ↔ Cours ↔ Professeurs ↔ Salles
- Nombreuses relations croisées
- Requêtes impliquant plusieurs entités
```

### 3. Besoin de requêtes ad-hoc complexes

SQL excelle pour l'exploration de données et les requêtes analytiques non prévues à l'avance.

```sql
-- Requête complexe facilement exprimable en SQL
SELECT
    d.nom_departement,
    COUNT(DISTINCT e.id) as nb_employes,
    AVG(s.montant) as salaire_moyen,
    SUM(CASE WHEN e.anciennete > 5 THEN 1 ELSE 0 END) as seniors
FROM departements d
JOIN employes e ON d.id = e.dept_id
JOIN salaires s ON e.id = s.employe_id
WHERE s.annee = 2024
GROUP BY d.nom_departement
HAVING COUNT(DISTINCT e.id) > 10
ORDER BY salaire_moyen DESC;
```

### 4. Écosystème et outils établis

L'écosystème SQL dispose d'outils matures pour le reporting, la BI, l'ETL et l'administration.

---

## Quand choisir MongoDB ?

MongoDB est particulièrement adapté dans ces situations :

### 1. Schéma en évolution

Quand votre modèle de données évolue fréquemment, la flexibilité de MongoDB est un atout majeur.

```
Exemple : Startup en phase de développement
- Fonctionnalités ajoutées régulièrement
- Modèle de données qui évolue
- Pas de migrations de schéma complexes
```

### 2. Documents autonomes

Quand les données forment naturellement des unités autonomes.

```json
// Un article de blog avec ses commentaires : un document autonome
{
  "titre": "Introduction à MongoDB",
  "auteur": { "nom": "Marie", "bio": "..." },
  "contenu": "...",
  "tags": ["mongodb", "nosql", "tutorial"],
  "commentaires": [
    { "auteur": "Pierre", "texte": "Super article !" },
    { "auteur": "Sophie", "texte": "Très clair, merci" }
  ]
}
```

### 3. Besoin de scalabilité horizontale

Pour les applications à fort trafic nécessitant une distribution des données.

```
Exemple : Application mobile avec millions d'utilisateurs
- Charge variable et imprévisible
- Besoin de scaler rapidement
- Données géographiquement distribuées
```

### 4. Données semi-structurées ou hétérogènes

Quand les données n'ont pas toutes la même structure.

```json
// Catalogue produits avec attributs variables
{ "type": "laptop", "cpu": "i7", "ram": "16GB", "ecran": "15\"" }
{ "type": "tshirt", "taille": "M", "couleur": "bleu", "matiere": "coton" }
{ "type": "livre", "isbn": "978-...", "pages": 350, "auteur": "..." }
```

### 5. Développement rapide

L'absence de schéma rigide accélère les itérations.

```
Exemple : Prototype ou MVP
- Time-to-market critique
- Besoin de flexibilité
- Équipe de développement agile
```

---

## L'approche hybride : Polyglot Persistence

Dans les architectures modernes, il est courant d'utiliser plusieurs types de bases de données, chacune pour ses forces :

```
┌─────────────────────────────────────────────────────────┐
│                   Application                           │
└─────────────────────────────────────────────────────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
    ┌─────────┐   ┌────────────┐   ┌─────────┐   ┌──────────┐
    │ MongoDB │   │PostgreSQL  │   │  Redis  │   │  Neo4j   │
    │         │   │            │   │         │   │          │
    │Catalogue│   │Transactions│   │ Cache   │   │Relations │
    │Produits │   │Financières │   │Sessions │   │ Sociales │
    └─────────┘   └────────────┘   └─────────┘   └──────────┘
```

Cette approche permet de tirer le meilleur de chaque technologie.

---

## Tableau récapitulatif

| Critère | SQL | MongoDB (NoSQL) |
|---------|-----|-----------------|
| **Modèle** | Tables relationnelles | Documents JSON/BSON |
| **Schéma** | Rigide, défini à l'avance | Flexible, dynamique |
| **Scalabilité** | Verticale principalement | Horizontale native |
| **Transactions** | ACID complet | ACID (depuis v4.0) |
| **Jointures** | Natives, optimisées | Possibles, moins courantes |
| **Requêtes** | SQL standardisé | API orientée objet |
| **Courbe d'apprentissage** | SQL à maîtriser | Proche du code applicatif |
| **Cas typiques** | ERP, finance, BI | Web, mobile, IoT, temps réel |

---

## Conclusion

SQL et NoSQL ne sont pas des technologies concurrentes mais **complémentaires**. Le choix entre les deux dépend de vos besoins spécifiques :

- **Choisissez SQL** pour des données structurées, des relations complexes et des besoins de reporting traditionnels
- **Choisissez MongoDB** pour la flexibilité, la scalabilité horizontale et le développement agile

MongoDB a considérablement évolué et offre aujourd'hui des fonctionnalités (transactions, jointures) autrefois réservées aux bases relationnelles, tout en conservant sa flexibilité et sa scalabilité natives.

Dans les sections suivantes, nous approfondirons les fondements théoriques (théorème CAP, ACID) pour mieux comprendre les compromis inhérents à chaque approche.

---

## Points clés à retenir

- **SQL** = bases relationnelles avec tables, schéma fixe et langage SQL
- **NoSQL** = "Not Only SQL", regroupe plusieurs familles (documents, clé-valeur, colonnes, graphes)
- **MongoDB** est une base NoSQL **orientée documents**
- Le **schéma flexible** de MongoDB permet d'adapter facilement le modèle de données
- MongoDB excelle en **scalabilité horizontale** (sharding)
- Les **transactions ACID** sont supportées depuis MongoDB 4.0
- L'approche **polyglot persistence** combine plusieurs types de bases selon les besoins
- Le choix SQL vs NoSQL dépend du **cas d'utilisation**, pas d'une supériorité absolue

---


⏭️ [Fondements théoriques](/01-introduction-a-mongodb/04-fondements-theoriques.md)
