🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 19.1 Migration depuis SQL vers MongoDB

## Introduction

La migration d'une base de données relationnelle vers MongoDB représente bien plus qu'un simple transfert de données. Elle nécessite une refonte conceptuelle profonde, passant d'un modèle normalisé basé sur des relations rigides à un modèle orienté documents flexible et dénormalisé.

Cette section détaille les stratégies, patterns et considérations techniques pour réussir une migration SQL → MongoDB à l'échelle entreprise.

---

## 📋 Phase 1 : Analyse et Audit Pré-Migration

### 1.1 Cartographie de la base source

#### Inventaire complet

**Collecte des métadonnées**
```sql
-- Exemple PostgreSQL : Analyse du schéma
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    n_live_tup AS row_count
FROM pg_stat_user_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- Analyse des relations (Foreign Keys)
SELECT
    tc.table_schema,
    tc.constraint_name,
    tc.table_name,
    kcu.column_name,
    ccu.table_name AS foreign_table_name,
    ccu.column_name AS foreign_column_name
FROM information_schema.table_constraints AS tc
JOIN information_schema.key_column_usage AS kcu
    ON tc.constraint_name = kcu.constraint_name
JOIN information_schema.constraint_column_usage AS ccu
    ON ccu.constraint_name = tc.constraint_name
WHERE tc.constraint_type = 'FOREIGN KEY';
```

**Métriques à collecter**

| Métrique | Utilité | Outil |
|----------|---------|-------|
| **Volume par table** | Dimensionnement MongoDB, stratégie sharding | `pg_stat_user_tables`, `information_schema` |
| **Cardinalité relations** | Décision embedding vs referencing | Queries analytics |
| **Patterns d'accès** | Modélisation orientée queries | Logs applicatifs, `pg_stat_statements` |
| **Taux de croissance** | Planification capacité | Historique backups |
| **Indexes existants** | Transposition vers MongoDB | `pg_indexes` |
| **Triggers/Procédures** | Refactoring applicatif | `information_schema.routines` |
| **Contraintes CHECK** | Validation MongoDB schema | `information_schema.check_constraints` |

#### Analyse des dépendances

**Graph de dépendances**
```python
# Script Python pour générer le graphe de dépendances
import psycopg2
import networkx as nx
import matplotlib.pyplot as plt

def build_dependency_graph(conn):
    """Construit le graphe des relations entre tables"""
    cursor = conn.cursor()

    # Récupération des FK
    cursor.execute("""
        SELECT
            tc.table_name AS source,
            ccu.table_name AS target
        FROM information_schema.table_constraints tc
        JOIN information_schema.constraint_column_usage ccu
            ON tc.constraint_name = ccu.constraint_name
        WHERE tc.constraint_type = 'FOREIGN KEY'
    """)

    G = nx.DiGraph()
    for source, target in cursor.fetchall():
        G.add_edge(source, target)

    return G

# Identification des tables centrales (hubs)
def identify_hubs(G, top_n=10):
    """Identifie les tables les plus référencées"""
    in_degree = dict(G.in_degree())
    sorted_tables = sorted(in_degree.items(), key=lambda x: x[1], reverse=True)
    return sorted_tables[:top_n]

# Détection de cycles
def detect_cycles(G):
    """Détecte les relations circulaires"""
    try:
        cycles = list(nx.simple_cycles(G))
        return cycles
    except:
        return []
```

**Implications architecturales**
- **Tables centrales (hubs)** : Candidates pour collections principales avec embedding des satellites
- **Cycles de dépendances** : Nécessitent des références bidirectionnelles ou patterns spécifiques
- **Tables orphelines** : Migrations indépendantes possibles

---

### 1.2 Analyse des patterns d'accès

#### Profiling des requêtes

**PostgreSQL : Activation du query logging**
```sql
-- Configuration postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
pg_stat_statements.max = 10000

-- Analyse des queries les plus fréquentes
SELECT
    query,
    calls,
    mean_exec_time,
    total_exec_time,
    rows
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 50;

-- Analyse des queries les plus coûteuses
SELECT
    query,
    total_exec_time / calls AS avg_time,
    calls
FROM pg_stat_statements
WHERE calls > 100
ORDER BY total_exec_time DESC
LIMIT 50;
```

**MySQL : Slow query log**
```ini
# my.cnf
slow_query_log = 1
long_query_time = 0.1
slow_query_log_file = /var/log/mysql/slow.log
log_queries_not_using_indexes = 1
```

#### Patterns identifiés → Décisions de modélisation

**Pattern 1 : Jointures systématiques (1-to-1 ou 1-to-N faible cardinalité)**

```sql
-- Query typique observée
SELECT u.*, p.bio, p.avatar_url, p.phone
FROM users u
LEFT JOIN profiles p ON u.id = p.user_id
WHERE u.status = 'active';
```

**Décision MongoDB : Embedding**
```javascript
// Collection users avec profil embedded
{
    _id: ObjectId(),
    email: "user@example.com",
    status: "active",
    profile: {  // Embedded document
        bio: "Data Architect",
        avatar_url: "https://...",
        phone: "+33612345678"
    }
}
```

**Pattern 2 : Accès indépendant aux entités**

```sql
-- Query 1: Lecture orders sans détails
SELECT id, order_date, total, customer_id
FROM orders
WHERE order_date > '2024-01-01';

-- Query 2: Lecture order_items indépendamment
SELECT product_id, SUM(quantity)
FROM order_items
WHERE created_at > '2024-01-01'
GROUP BY product_id;
```

**Décision MongoDB : Referencing**
```javascript
// Collection orders (léger)
{ _id: ObjectId(), order_date: ISODate(), total: 150.50, customer_id: 123 }

// Collection order_items (séparée)
{ _id: ObjectId(), order_id: ObjectId(), product_id: 456, quantity: 2 }
```

**Pattern 3 : Agrégations complexes**

```sql
-- Analytics multi-tables
SELECT
    c.region,
    COUNT(DISTINCT o.id) as orders,
    SUM(oi.quantity * p.price) as revenue
FROM customers c
JOIN orders o ON c.id = o.customer_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.order_date >= '2024-01-01'
GROUP BY c.region;
```

**Décision MongoDB : Dénormalisation partielle + Aggregation Framework**
```javascript
// Order document avec données dénormalisées pour analytics
{
    _id: ObjectId(),
    order_date: ISODate("2024-01-15"),
    customer: {
        id: 123,
        region: "Europe"  // Dénormalisé pour analytics
    },
    items: [
        {
            product_id: 456,
            quantity: 2,
            unit_price: 25.50,  // Snapshot du prix à la commande
            subtotal: 51.00
        }
    ],
    total: 51.00
}

// Pipeline aggregation MongoDB
db.orders.aggregate([
    { $match: { order_date: { $gte: ISODate("2024-01-01") } } },
    { $group: {
        _id: "$customer.region",
        orders: { $sum: 1 },
        revenue: { $sum: "$total" }
    }}
])
```

---

### 1.3 Évaluation de la complexité

#### Matrice de complexité

| Dimension | Faible | Moyenne | Élevée |
|-----------|--------|---------|--------|
| **Volume données** | < 100 Go | 100 Go - 5 To | > 5 To |
| **Nombre de tables** | < 50 | 50 - 200 | > 200 |
| **Profondeur relations** | 2-3 niveaux | 4-5 niveaux | > 5 niveaux |
| **Logique métier SQL** | Minime (contraintes) | Triggers modérés | Procédures complexes |
| **Exigences disponibilité** | Downtime acceptable | Downtime limité | Zero downtime |
| **Hétérogénéité types données** | Standards | Mix standards/LOB | Spatial, JSON, binaires |

**Score de complexité global**
```
Score = (Volume * 1) + (Tables * 2) + (Profondeur * 3) + (Logique * 4) + (Disponibilité * 5) + (Hétérogénéité * 2)

< 30 : Migration simple (2-4 semaines)
30-60 : Migration modérée (2-3 mois)
> 60 : Migration complexe (6+ mois)
```

---

## 🔄 Phase 2 : Transformation de Schéma SQL → MongoDB

### 2.1 Principes de transformation

#### Règle d'or : Modéliser selon les queries, pas selon les relations

**Anti-pattern : Reproduction du schéma SQL**
```javascript
// ❌ MAUVAIS : Schéma normalisé "à la SQL"
// Collection users
{ _id: 1, name: "Alice" }

// Collection addresses
{ _id: 1, user_id: 1, street: "123 Main St", city: "Paris" }

// Collection orders
{ _id: 1, user_id: 1, date: ISODate() }

// Collection order_items
{ _id: 1, order_id: 1, product_id: 101, qty: 2 }
```

**Pattern correct : Modélisation orientée document**
```javascript
// ✅ BON : Agrégats métier cohérents
{
    _id: 1,
    name: "Alice",
    email: "alice@example.com",

    // Adresses embeddées (1-to-few)
    addresses: [
        { type: "billing", street: "123 Main St", city: "Paris", country: "FR" },
        { type: "shipping", street: "456 Oak Ave", city: "Lyon", country: "FR" }
    ],

    // Références vers orders (1-to-many, accès indépendant)
    recent_order_ids: [ObjectId("..."), ObjectId("...")],  // Cache des N dernières

    // Statistiques dénormalisées (évite COUNT queries)
    stats: {
        total_orders: 45,
        lifetime_value: 3500.00,
        last_order_date: ISODate("2024-01-15")
    }
}
```

---

### 2.2 Transformation systématique par type de relation

#### Relation 1-to-1 : User ↔ Profile

**Schéma SQL**
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    created_at TIMESTAMP
);

CREATE TABLE user_profiles (
    user_id INT PRIMARY KEY REFERENCES users(id),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    bio TEXT,
    avatar_url VARCHAR(500),
    date_of_birth DATE,
    phone VARCHAR(20)
);
```

**MongoDB : Embedding (recommandé)**
```javascript
{
    _id: ObjectId("507f1f77bcf86cd799439011"),
    email: "alice@example.com",
    password_hash: "$2b$10$...",
    created_at: ISODate("2023-01-15T10:30:00Z"),

    // Profile embedded
    profile: {
        first_name: "Alice",
        last_name: "Dupont",
        bio: "Senior Data Architect passionate about distributed systems",
        avatar_url: "https://cdn.example.com/avatars/alice.jpg",
        date_of_birth: ISODate("1985-03-20"),
        phone: "+33612345678"
    }
}
```

**Quand utiliser le referencing pour 1-to-1 ?**
- Profil très volumineux (> 1 Mo, ex: documents scannés)
- Accès profil rare par rapport aux accès user
- Cycles de mise à jour différents (user fréquent, profile rare)

**Exemple avec référence**
```javascript
// Collection users (léger, accès fréquent)
{
    _id: ObjectId("507f1f77bcf86cd799439011"),
    email: "alice@example.com",
    password_hash: "$2b$10$...",
    profile_id: ObjectId("507f1f77bcf86cd799439012")  // Référence
}

// Collection profiles (volumineux, accès rare)
{
    _id: ObjectId("507f1f77bcf86cd799439012"),
    user_id: ObjectId("507f1f77bcf86cd799439011"),  // Référence inverse
    first_name: "Alice",
    // ... autres champs ...
    scanned_documents: [
        { type: "id_card", url: "s3://...", size: 2048576 },
        { type: "proof_address", url: "s3://...", size: 1536000 }
    ]
}
```

---

#### Relation 1-to-Few : Order → Order Items (< 100 items)

**Schéma SQL**
```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    order_date TIMESTAMP,
    status VARCHAR(20),
    total_amount DECIMAL(10,2)
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    product_id INT REFERENCES products(id),
    quantity INT,
    unit_price DECIMAL(10,2),
    discount DECIMAL(5,2)
);
```

**MongoDB : Embedding avec dénormalisation**
```javascript
{
    _id: ObjectId(),
    order_number: "ORD-2024-00123",
    order_date: ISODate("2024-01-15T14:30:00Z"),
    status: "shipped",

    // Customer info (dénormalisé pour analytics)
    customer: {
        id: 12345,
        name: "Bob Martin",
        email: "bob@example.com",
        tier: "gold"  // Utile pour reporting
    },

    // Items embeddés (array)
    items: [
        {
            product_id: 101,
            product_name: "MongoDB Handbook",  // Dénormalisé (snapshot)
            sku: "BOOK-001",
            quantity: 2,
            unit_price: 45.00,
            discount: 10.0,
            subtotal: 81.00  // Pré-calculé
        },
        {
            product_id: 203,
            product_name: "USB-C Cable",
            sku: "ACC-023",
            quantity: 1,
            unit_price: 15.00,
            discount: 0,
            subtotal: 15.00
        }
    ],

    // Agrégats pré-calculés
    summary: {
        item_count: 3,
        subtotal: 96.00,
        tax: 19.20,
        shipping: 5.00,
        total: 120.20
    },

    // Shipping
    shipping_address: {
        street: "123 Rue de la Paix",
        city: "Paris",
        postal_code: "75001",
        country: "FR"
    },

    // Audit
    created_at: ISODate("2024-01-15T14:30:00Z"),
    updated_at: ISODate("2024-01-16T09:15:00Z"),
    shipped_at: ISODate("2024-01-16T09:15:00Z")
}
```

**Avantages de l'embedding**
- ✅ **Atomicité** : Une commande = un document (transaction implicite)
- ✅ **Performance lecture** : Pas de JOIN, une seule query
- ✅ **Snapshot historique** : Prix et noms produits figés à la commande
- ✅ **Analytics simplifiés** : Aggregation directe sans jointure

**Index recommandés**
```javascript
db.orders.createIndex({ "customer.id": 1, "order_date": -1 })
db.orders.createIndex({ "order_date": -1, "status": 1 })
db.orders.createIndex({ "items.product_id": 1 })  // Multikey index
db.orders.createIndex({ "order_number": 1 }, { unique: true })
```

---

#### Relation 1-to-Many : Blog Post → Comments (> 100 items)

**Schéma SQL**
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    content TEXT,
    author_id INT,
    published_at TIMESTAMP
);

CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INT REFERENCES posts(id),
    user_id INT,
    comment_text TEXT,
    created_at TIMESTAMP,
    parent_comment_id INT  -- Pour nested comments
);
```

**Problème avec embedding complet**
```javascript
// ❌ ANTI-PATTERN : Document peut dépasser 16 Mo
{
    _id: ObjectId(),
    title: "MongoDB vs PostgreSQL",
    content: "...",
    comments: [
        // Si post viral avec 10 000 comments → Document trop gros !
        { user_id: 1, text: "Great article!", created_at: ISODate() },
        // ... 9999 autres comments
    ]
}
```

**Solution 1 : Bucketing Pattern**
```javascript
// Collection posts (données principales)
{
    _id: ObjectId("POST001"),
    title: "MongoDB vs PostgreSQL",
    content: "Long article content...",
    author_id: 123,
    published_at: ISODate("2024-01-15"),

    stats: {
        comment_count: 1250,
        view_count: 15000,
        like_count: 350
    }
}

// Collection comment_buckets (buckets de ~50 comments)
{
    _id: ObjectId(),
    post_id: ObjectId("POST001"),
    bucket_number: 1,  // Bucket 1, 2, 3...
    count: 50,         // Nombre de comments dans ce bucket
    comments: [
        {
            comment_id: ObjectId(),
            user_id: 456,
            user_name: "Alice",  // Dénormalisé
            text: "Excellent comparison!",
            created_at: ISODate("2024-01-15T10:00:00Z"),
            likes: 12
        },
        // ... 49 autres comments
    ],
    created_at: ISODate("2024-01-15"),
    updated_at: ISODate("2024-01-15")
}

// Bucket suivant
{
    _id: ObjectId(),
    post_id: ObjectId("POST001"),
    bucket_number: 2,
    count: 50,
    comments: [ /* ... */ ]
}
```

**Queries sur le Bucketing Pattern**
```javascript
// 1. Récupérer post + premiers comments
const post = await db.posts.findOne({ _id: postId });
const firstBucket = await db.comment_buckets.findOne(
    { post_id: postId, bucket_number: 1 }
);

// 2. Ajouter un nouveau comment
// Si bucket courant < 50 comments → $push
await db.comment_buckets.updateOne(
    { post_id: postId, bucket_number: currentBucket, count: { $lt: 50 } },
    {
        $push: { comments: newComment },
        $inc: { count: 1 },
        $set: { updated_at: new Date() }
    }
);

// Si bucket plein → créer nouveau bucket
if (result.modifiedCount === 0) {
    await db.comment_buckets.insertOne({
        post_id: postId,
        bucket_number: currentBucket + 1,
        count: 1,
        comments: [newComment],
        created_at: new Date()
    });
}

// 3. Pagination des comments
const buckets = await db.comment_buckets.find(
    { post_id: postId },
    { sort: { bucket_number: 1 }, limit: 3 }  // 3 buckets = 150 comments
).toArray();
```

**Solution 2 : Hybrid (Embedding partiel + Référence)**
```javascript
// Post avec N derniers comments embeddés
{
    _id: ObjectId("POST001"),
    title: "MongoDB vs PostgreSQL",
    content: "...",

    // 10 derniers comments embeddés (hot data)
    recent_comments: [
        { user_id: 789, text: "Just read this, amazing!", created_at: ISODate() },
        // ... 9 autres
    ],

    stats: {
        total_comments: 1250,
        comment_pages: 25  // Pour pagination
    }
}

// Collection comments (tous les comments, cold data)
db.comments (référencés par post_id)
```

---

#### Relation Many-to-Many : Users ↔ Groups

**Schéma SQL**
```sql
CREATE TABLE users (id SERIAL PRIMARY KEY, name VARCHAR);
CREATE TABLE groups (id SERIAL PRIMARY KEY, name VARCHAR);

-- Table de jonction
CREATE TABLE user_groups (
    user_id INT REFERENCES users(id),
    group_id INT REFERENCES groups(id),
    joined_at TIMESTAMP,
    role VARCHAR(50),
    PRIMARY KEY (user_id, group_id)
);
```

**MongoDB : Choix selon le pattern d'accès**

**Option A : Array de références dans chaque entité**
```javascript
// Collection users
{
    _id: ObjectId("USER001"),
    name: "Alice Dupont",
    group_ids: [  // Array de références
        ObjectId("GROUP001"),
        ObjectId("GROUP002"),
        ObjectId("GROUP003")
    ]
}

// Collection groups
{
    _id: ObjectId("GROUP001"),
    name: "Data Engineering Team",
    member_ids: [  // Array de références
        ObjectId("USER001"),
        ObjectId("USER002"),
        ObjectId("USER003")
    ]
}
```

**Avantages** : Simple, performant pour N-to-N avec cardinalité faible (<100)
**Inconvénients** :
- Pas d'attributs sur la relation (joined_at, role)
- Synchronisation bidirectionnelle complexe

**Option B : Collection de jonction (Extended Reference Pattern)**
```javascript
// Collection user_groups (similaire SQL)
{
    _id: ObjectId(),
    user_id: ObjectId("USER001"),
    group_id: ObjectId("GROUP001"),

    // Attributs de la relation
    joined_at: ISODate("2024-01-15"),
    role: "admin",

    // Dénormalisation pour queries
    user_name: "Alice Dupont",
    user_email: "alice@example.com",
    group_name: "Data Engineering Team",
    group_type: "technical"
}

// Index pour queries bidirectionnelles
db.user_groups.createIndex({ user_id: 1, group_id: 1 }, { unique: true })
db.user_groups.createIndex({ group_id: 1, joined_at: -1 })
db.user_groups.createIndex({ user_id: 1, joined_at: -1 })
```

**Queries sur Extended Reference**
```javascript
// Tous les groupes d'un user avec détails
db.user_groups.aggregate([
    { $match: { user_id: ObjectId("USER001") } },
    { $lookup: {
        from: "groups",
        localField: "group_id",
        foreignField: "_id",
        as: "group_details"
    }},
    { $unwind: "$group_details" },
    { $project: {
        group_name: "$group_details.name",
        group_type: "$group_details.type",
        role: 1,
        joined_at: 1
    }}
])

// Tous les membres d'un groupe
db.user_groups.aggregate([
    { $match: { group_id: ObjectId("GROUP001") } },
    { $lookup: {
        from: "users",
        localField: "user_id",
        foreignField: "_id",
        as: "user_details"
    }},
    { $unwind: "$user_details" },
    { $sort: { joined_at: -1 } }
])
```

**Option C : Dénormalisation complète (si queries read-heavy)**
```javascript
// Collection users avec groupes dénormalisés
{
    _id: ObjectId("USER001"),
    name: "Alice Dupont",
    memberships: [
        {
            group_id: ObjectId("GROUP001"),
            group_name: "Data Engineering Team",  // Dénormalisé
            group_type: "technical",
            role: "admin",
            joined_at: ISODate("2024-01-15")
        },
        {
            group_id: ObjectId("GROUP002"),
            group_name: "MongoDB Experts",
            group_type: "community",
            role: "member",
            joined_at: ISODate("2023-06-10")
        }
    ]
}

// Collection groups avec membres dénormalisés
{
    _id: ObjectId("GROUP001"),
    name: "Data Engineering Team",
    type: "technical",
    members: [
        {
            user_id: ObjectId("USER001"),
            user_name: "Alice Dupont",  // Dénormalisé
            user_email: "alice@example.com",
            role: "admin",
            joined_at: ISODate("2024-01-15")
        },
        // ... autres membres
    ],
    stats: {
        total_members: 25,
        admin_count: 3
    }
}
```

**Trade-offs dénormalisation**
- ✅ Performance lecture maximale (pas de $lookup)
- ✅ Queries simples
- ❌ Complexité sur les updates (maintenir cohérence)
- ❌ Duplication données (storage)

**Gestion des updates avec dénormalisation**
```javascript
// Changement de nom d'un user → Update en cascade
async function updateUserName(userId, newName) {
    const session = client.startSession();

    try {
        await session.withTransaction(async () => {
            // Update collection users
            await db.users.updateOne(
                { _id: userId },
                { $set: { name: newName } },
                { session }
            );

            // Update dans tous les groupes où il est membre
            await db.groups.updateMany(
                { "members.user_id": userId },
                { $set: { "members.$[elem].user_name": newName } },
                {
                    arrayFilters: [{ "elem.user_id": userId }],
                    session
                }
            );
        });
    } finally {
        await session.endSession();
    }
}
```

---

### 2.3 Gestion des types de données spéciaux

#### Types temporels (Date, Timestamp, Time zones)

**SQL → MongoDB**
```sql
-- SQL: Types variés
CREATE TABLE events (
    event_date DATE,                    -- Date seule
    event_datetime TIMESTAMP,           -- Date + heure (TZ-aware)
    event_time TIME,                    -- Heure seule
    created_at TIMESTAMP DEFAULT NOW()
);
```

**MongoDB : Type Date unifié (ISO 8601)**
```javascript
{
    _id: ObjectId(),

    // Date seule → ISODate à minuit UTC
    event_date: ISODate("2024-01-15T00:00:00Z"),

    // Datetime avec timezone
    event_datetime: ISODate("2024-01-15T14:30:00Z"),  // Stocké en UTC

    // Time seul → Stratégie 1: Epoch day + secondes
    event_time_seconds: 52200,  // 14h30 = 52200 secondes depuis minuit

    // Time seul → Stratégie 2: String (si pas de calculs)
    event_time_str: "14:30:00",

    // Stratégie 3: Date fictive (1970-01-01 + time)
    event_time_date: ISODate("1970-01-01T14:30:00Z"),

    // Timestamp d'audit
    created_at: ISODate("2024-01-15T10:20:30.123Z")
}
```

**Gestion timezone en applicatif**
```javascript
// Conversion UTC → Timezone local via aggregation
db.events.aggregate([
    {
        $project: {
            event_datetime_utc: "$event_datetime",
            event_datetime_paris: {
                $dateToString: {
                    format: "%Y-%m-%d %H:%M:%S",
                    date: "$event_datetime",
                    timezone: "Europe/Paris"
                }
            }
        }
    }
])
```

#### Types numériques (Decimal, Float, Integer)

**SQL**
```sql
CREATE TABLE products (
    id SERIAL,
    price DECIMAL(10,2),      -- Précision fixe (finance)
    weight FLOAT,             -- Approximatif
    stock_quantity INT,
    views_count BIGINT
);
```

**MongoDB : Attention aux types numériques**
```javascript
{
    _id: ObjectId(),

    // Prix financiers → NumberDecimal (CRITICAL pour finance)
    price: NumberDecimal("19.99"),  // Exactitude garantie

    // Float approximatif
    weight: 1.75,  // Double JavaScript (perte précision possible)

    // Entiers
    stock_quantity: NumberInt(150),  // 32-bit
    views_count: NumberLong(123456789),  // 64-bit

    // Alternative pour finance : Stockage en centimes (int)
    price_cents: 1999,  // 19.99 EUR = 1999 cents
    currency: "EUR"
}
```

**Conversion depuis SQL**
```python
# Script Python pour migration avec types corrects
from decimal import Decimal
from bson import Decimal128

def migrate_product(sql_row):
    return {
        "_id": ObjectId(),
        "sku": sql_row['sku'],
        "price": Decimal128(Decimal(str(sql_row['price']))),  # SQL DECIMAL → BSON Decimal128
        "weight": float(sql_row['weight']),
        "stock_quantity": int(sql_row['stock_quantity']),
        "views_count": int(sql_row['views_count'])
    }
```

#### Types textes longs (TEXT, CLOB, Varchar(MAX))

**SQL**
```sql
CREATE TABLE articles (
    id SERIAL,
    title VARCHAR(255),
    summary TEXT,
    content TEXT,           -- Peut être très volumineux
    metadata JSONB
);
```

**MongoDB : Gestion documents volumineux**
```javascript
// Si content < 16 Mo (limite BSON) → Stockage direct
{
    _id: ObjectId(),
    title: "Complete MongoDB Guide",
    summary: "A comprehensive guide...",
    content: "Very long article content...",  // OK si < 16 Mo
    metadata: {
        author: "Jane Doe",
        tags: ["database", "nosql"]
    }
}

// Si content > 16 Mo → GridFS obligatoire
{
    _id: ObjectId(),
    title: "Complete MongoDB Guide",
    summary: "A comprehensive guide...",
    content_gridfs_id: ObjectId("..."),  // Référence vers GridFS
    content_size: 25000000,  // 25 Mo
    metadata: { /* ... */ }
}
```

**Utilisation GridFS**
```javascript
// Écriture dans GridFS
const bucket = new GridFSBucket(db, { bucketName: 'articles' });
const uploadStream = bucket.openUploadStream('article_content_123.txt', {
    metadata: { article_id: ObjectId("..."), content_type: "text/plain" }
});

uploadStream.write(largeContent);
uploadStream.end();

// Lecture depuis GridFS
const downloadStream = bucket.openDownloadStream(gridfsId);
downloadStream.on('data', (chunk) => {
    // Traiter chunk par chunk
});
```

#### Types binaires (BLOB, Bytea)

**Stratégies selon taille**

| Taille | Stratégie | Technologie |
|--------|-----------|-------------|
| < 16 Mo | BSON BinData | MongoDB natif |
| 16 Mo - 100 Mo | GridFS | MongoDB GridFS |
| > 100 Mo | Object Storage | S3, Azure Blob, GCS |

**Migration des BLOBs**
```python
def migrate_with_blob(sql_row):
    blob_data = sql_row['document_blob']
    blob_size = len(blob_data)

    if blob_size < 16_000_000:  # < 16 Mo
        return {
            "_id": ObjectId(),
            "filename": sql_row['filename'],
            "data": Binary(blob_data),  # BinData BSON
            "size": blob_size
        }
    elif blob_size < 100_000_000:  # < 100 Mo
        # Upload vers GridFS
        gridfs_id = upload_to_gridfs(blob_data, sql_row['filename'])
        return {
            "_id": ObjectId(),
            "filename": sql_row['filename'],
            "gridfs_id": gridfs_id,
            "size": blob_size,
            "storage": "gridfs"
        }
    else:  # > 100 Mo
        # Upload vers S3
        s3_url = upload_to_s3(blob_data, sql_row['filename'])
        return {
            "_id": ObjectId(),
            "filename": sql_row['filename'],
            "s3_url": s3_url,
            "size": blob_size,
            "storage": "s3"
        }
```

---

### 2.4 Transformation de la logique métier SQL

#### Triggers SQL → Application Logic / Change Streams

**SQL Trigger exemple**
```sql
-- Trigger pour audit automatique
CREATE OR REPLACE FUNCTION audit_product_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO product_audit_log (
        product_id,
        action,
        old_price,
        new_price,
        changed_by,
        changed_at
    ) VALUES (
        NEW.id,
        TG_OP,
        OLD.price,
        NEW.price,
        current_user,
        NOW()
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER product_change_trigger
AFTER UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION audit_product_changes();
```

**MongoDB : Stratégie 1 - Application logic**
```javascript
// Dans le code applicatif (Node.js exemple)
async function updateProductPrice(productId, newPrice, userId) {
    const session = client.startSession();

    try {
        await session.withTransaction(async () => {
            // Récupérer ancien prix
            const product = await db.products.findOne(
                { _id: productId },
                { session }
            );

            // Update produit
            await db.products.updateOne(
                { _id: productId },
                {
                    $set: { price: newPrice, updated_at: new Date() }
                },
                { session }
            );

            // Insérer audit log
            await db.product_audit_log.insertOne({
                product_id: productId,
                action: "UPDATE",
                old_price: product.price,
                new_price: newPrice,
                changed_by: userId,
                changed_at: new Date()
            }, { session });
        });
    } finally {
        await session.endSession();
    }
}
```

**MongoDB : Stratégie 2 - Change Streams (event-driven)**
```javascript
// Service d'audit écoutant les changements
const changeStream = db.products.watch([
    { $match: { "operationType": "update" } }
]);

changeStream.on('change', async (change) => {
    const productId = change.documentKey._id;
    const updatedFields = change.updateDescription.updatedFields;

    if ('price' in updatedFields) {
        // Extraire old_price depuis fullDocument (si activé)
        const oldPrice = change.fullDocumentBeforeChange?.price;

        // Insérer audit log
        await db.product_audit_log.insertOne({
            product_id: productId,
            action: "UPDATE",
            old_price: oldPrice,
            new_price: updatedFields.price,
            changed_at: new Date(),
            change_stream_id: change._id
        });
    }
});
```

**Comparaison**

| Aspect | SQL Trigger | App Logic | Change Streams |
|--------|-------------|-----------|----------------|
| **Couplage** | Fort (DB) | Moyen (code) | Faible (event) |
| **Performance** | Overhead sync | Contrôlable | Async (scalable) |
| **Testabilité** | Difficile | Facile | Moyenne |
| **Maintenance** | Complexe | Standard | Moderne |

#### Procédures stockées → Application Services

**SQL Stored Procedure**
```sql
CREATE PROCEDURE process_order(
    p_customer_id INT,
    p_product_ids INT[],
    p_quantities INT[]
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_order_id INT;
    v_total DECIMAL(10,2) := 0;
    v_idx INT;
BEGIN
    -- Créer order
    INSERT INTO orders (customer_id, order_date, status)
    VALUES (p_customer_id, NOW(), 'pending')
    RETURNING id INTO v_order_id;

    -- Insérer items et calculer total
    FOR v_idx IN 1..array_length(p_product_ids, 1) LOOP
        DECLARE
            v_price DECIMAL(10,2);
        BEGIN
            SELECT price INTO v_price
            FROM products
            WHERE id = p_product_ids[v_idx];

            INSERT INTO order_items (order_id, product_id, quantity, unit_price)
            VALUES (v_order_id, p_product_ids[v_idx], p_quantities[v_idx], v_price);

            v_total := v_total + (v_price * p_quantities[v_idx]);
        END;
    END LOOP;

    -- Update total
    UPDATE orders SET total_amount = v_total WHERE id = v_order_id;

    -- Décrémenter stock
    UPDATE products
    SET stock_quantity = stock_quantity - p_quantities[idx]
    WHERE id = ANY(p_product_ids);

    COMMIT;
END;
$$;
```

**MongoDB : Service applicatif (Node.js)**
```javascript
class OrderService {
    async processOrder(customerId, items) {
        // items: [{ productId, quantity }, ...]

        const session = this.client.startSession();

        try {
            return await session.withTransaction(async () => {
                // 1. Récupérer infos produits
                const productIds = items.map(i => ObjectId(i.productId));
                const products = await this.db.products.find(
                    { _id: { $in: productIds } },
                    { session }
                ).toArray();

                const productMap = new Map(products.map(p => [p._id.toString(), p]));

                // 2. Calculer total et vérifier stock
                let total = 0;
                const orderItems = [];

                for (const item of items) {
                    const product = productMap.get(item.productId);

                    if (!product) {
                        throw new Error(`Product ${item.productId} not found`);
                    }

                    if (product.stock_quantity < item.quantity) {
                        throw new Error(`Insufficient stock for ${product.name}`);
                    }

                    const subtotal = product.price * item.quantity;
                    total += subtotal;

                    orderItems.push({
                        product_id: product._id,
                        product_name: product.name,  // Dénormalisé
                        sku: product.sku,
                        quantity: item.quantity,
                        unit_price: product.price,
                        subtotal
                    });
                }

                // 3. Créer order
                const order = {
                    customer_id: customerId,
                    order_date: new Date(),
                    status: 'pending',
                    items: orderItems,
                    total,
                    created_at: new Date()
                };

                const orderResult = await this.db.orders.insertOne(order, { session });

                // 4. Décrémenter stocks
                const bulkOps = items.map(item => ({
                    updateOne: {
                        filter: { _id: ObjectId(item.productId) },
                        update: { $inc: { stock_quantity: -item.quantity } }
                    }
                }));

                await this.db.products.bulkWrite(bulkOps, { session });

                return { orderId: orderResult.insertedId, total };
            });
        } finally {
            await session.endSession();
        }
    }
}
```

#### Contraintes CHECK → Schema Validation

**SQL**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    price DECIMAL(10,2),
    stock_quantity INT,
    status VARCHAR(20),

    -- Contraintes CHECK
    CONSTRAINT price_positive CHECK (price >= 0),
    CONSTRAINT stock_non_negative CHECK (stock_quantity >= 0),
    CONSTRAINT valid_status CHECK (status IN ('active', 'discontinued', 'out_of_stock'))
);
```

**MongoDB : JSON Schema Validation**
```javascript
db.createCollection("products", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "price", "stock_quantity", "status"],
            properties: {
                name: {
                    bsonType: "string",
                    minLength: 1,
                    maxLength: 255,
                    description: "Product name is required"
                },
                price: {
                    bsonType: ["double", "decimal"],
                    minimum: 0,
                    description: "Price must be >= 0"
                },
                stock_quantity: {
                    bsonType: "int",
                    minimum: 0,
                    description: "Stock cannot be negative"
                },
                status: {
                    enum: ["active", "discontinued", "out_of_stock"],
                    description: "Status must be one of allowed values"
                },
                sku: {
                    bsonType: "string",
                    pattern: "^[A-Z]{3}-[0-9]{4}$",  // Format: ABC-1234
                    description: "SKU must match pattern"
                },
                created_at: {
                    bsonType: "date"
                }
            }
        }
    },
    validationLevel: "strict",    // strict ou moderate
    validationAction: "error"     // error ou warn
})
```

**Validation avancée avec $expr**
```javascript
// Contrainte complexe : discount ne peut pas dépasser 50% du prix
db.runCommand({
    collMod: "products",
    validator: {
        $expr: {
            $lte: ["$discount_amount", { $multiply: ["$price", 0.5] }]
        }
    }
})
```

---

## 🚀 Phase 3 : Stratégies de Migration des Données

### 3.1 Migration complète (Big Bang)

**Scénario : E-commerce PME (500 Go, downtime acceptable)**

**Étape 1 : Préparation (J-7)**
```bash
# 1. Backup complet SQL
pg_dump -h source_host -U postgres -Fc mydb > backup_full.dump

# 2. Dry-run migration (environnement test)
python migrate.py --source postgres://... --target mongodb://... --dry-run

# 3. Validation schéma MongoDB
mongosh --eval "db.getSiblingDB('mydb').products.findOne()" | jq
```

**Étape 2 : Migration (Vendredi 22h → Lundi 6h)**
```python
# Script migration complet
import psycopg2
from pymongo import MongoClient
from bson import ObjectId
from datetime import datetime
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class BigBangMigration:
    def __init__(self, pg_conn_str, mongo_conn_str):
        self.pg_conn = psycopg2.connect(pg_conn_str)
        self.mongo_client = MongoClient(mongo_conn_str)
        self.db = self.mongo_client['mydb']

    def migrate_table(self, table_name, transformer, batch_size=1000):
        """Migration générique d'une table"""
        cursor = self.pg_conn.cursor(name=f'cursor_{table_name}')
        cursor.execute(f"SELECT * FROM {table_name}")

        collection = self.db[table_name]
        batch = []
        total = 0

        logger.info(f"Starting migration of {table_name}")

        for row in cursor:
            doc = transformer(row)
            batch.append(doc)

            if len(batch) >= batch_size:
                collection.insert_many(batch, ordered=False)
                total += len(batch)
                logger.info(f"{table_name}: {total} documents migrated")
                batch = []

        if batch:
            collection.insert_many(batch, ordered=False)
            total += len(batch)

        logger.info(f"{table_name}: Migration complete. Total: {total}")
        cursor.close()

    def migrate_products(self):
        """Migration table products"""
        def transform(row):
            return {
                "_id": ObjectId(),
                "legacy_id": row[0],  # id SQL conservé
                "sku": row[1],
                "name": row[2],
                "price": Decimal128(Decimal(str(row[3]))),
                "stock_quantity": row[4],
                "status": row[5],
                "created_at": row[6],
                "migrated_at": datetime.utcnow()
            }

        self.migrate_table("products", transform)

        # Créer indexes
        self.db.products.create_index("legacy_id", unique=True)
        self.db.products.create_index("sku", unique=True)
        self.db.products.create_index([("name", "text")])

    def migrate_orders_with_items(self):
        """Migration orders avec embedding des items"""
        cursor = self.pg_conn.cursor(name='orders_cursor')
        cursor.execute("""
            SELECT
                o.id, o.customer_id, o.order_date, o.status, o.total_amount,
                array_agg(
                    json_build_object(
                        'product_id', oi.product_id,
                        'quantity', oi.quantity,
                        'unit_price', oi.unit_price
                    )
                ) as items
            FROM orders o
            LEFT JOIN order_items oi ON o.id = oi.order_id
            GROUP BY o.id
        """)

        batch = []
        for row in cursor:
            doc = {
                "_id": ObjectId(),
                "legacy_id": row[0],
                "customer_id": row[1],
                "order_date": row[2],
                "status": row[3],
                "total": Decimal128(Decimal(str(row[4]))),
                "items": row[5] if row[5] else [],
                "migrated_at": datetime.utcnow()
            }
            batch.append(doc)

            if len(batch) >= 1000:
                self.db.orders.insert_many(batch, ordered=False)
                batch = []

        if batch:
            self.db.orders.insert_many(batch)

        cursor.close()

        # Indexes
        self.db.orders.create_index("legacy_id", unique=True)
        self.db.orders.create_index([("customer_id", 1), ("order_date", -1)])

    def run_full_migration(self):
        """Exécution complète"""
        tables = [
            ("customers", self.migrate_customers),
            ("products", self.migrate_products),
            ("orders", self.migrate_orders_with_items)
        ]

        start = datetime.utcnow()

        for table_name, migrator in tables:
            logger.info(f"=== Migrating {table_name} ===")
            migrator()

        duration = (datetime.utcnow() - start).total_seconds()
        logger.info(f"Full migration completed in {duration:.2f} seconds")

        # Validation
        self.validate_migration()

    def validate_migration(self):
        """Validation row counts"""
        cursor = self.pg_conn.cursor()

        validations = [
            ("customers", "SELECT COUNT(*) FROM customers"),
            ("products", "SELECT COUNT(*) FROM products"),
            ("orders", "SELECT COUNT(*) FROM orders")
        ]

        all_valid = True
        for collection_name, sql_query in validations:
            cursor.execute(sql_query)
            sql_count = cursor.fetchone()[0]
            mongo_count = self.db[collection_name].count_documents({})

            status = "✓" if sql_count == mongo_count else "✗"
            logger.info(f"{status} {collection_name}: SQL={sql_count}, MongoDB={mongo_count}")

            if sql_count != mongo_count:
                all_valid = False

        cursor.close()
        return all_valid

# Exécution
if __name__ == "__main__":
    migrator = BigBangMigration(
        pg_conn_str="postgresql://user:pass@localhost/mydb",
        mongo_conn_str="mongodb://localhost:27017"
    )
    migrator.run_full_migration()
```

**Étape 3 : Validation (Samedi matin)**
```javascript
// Script validation MongoDB
// 1. Row counts
db.customers.countDocuments()  // vs SQL count
db.products.countDocuments()
db.orders.countDocuments()

// 2. Sampling validation (100 random orders)
const sqlOrders = [...];  // From SQL export
const mongoOrders = db.orders.aggregate([
    { $sample: { size: 100 } }
]).toArray();

// Compare field by field

// 3. Business validation
// Exécuter test suite applicative complète
```

**Étape 4 : Cutover (Dimanche)**
```bash
# 1. Dernière synchro incrémentale (si données modifiées pendant migration)
# 2. Update connection strings dans app
# 3. Déploiement nouvelle version app (MongoDB)
# 4. Smoke tests production
# 5. Monitoring intensif
```

---

### 3.2 Migration incrémentale avec CDC (Change Data Capture)

**Scénario : Banque (5 To, zero downtime requis)**

**Architecture**
```
[PostgreSQL] → [Debezium] → [Kafka] → [MongoDB Sink Connector] → [MongoDB]
     ↓
[Application] (dual-read: MongoDB, fallback SQL si nécessaire)
```

**Étape 1 : Setup CDC avec Debezium**
```yaml
# docker-compose.yml pour Debezium stack
version: '3.8'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on: [zookeeper]
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092

  debezium:
    image: debezium/connect:latest
    depends_on: [kafka]
    environment:
      BOOTSTRAP_SERVERS: kafka:9092
      GROUP_ID: 1
      CONFIG_STORAGE_TOPIC: debezium_configs
      OFFSET_STORAGE_TOPIC: debezium_offsets
      STATUS_STORAGE_TOPIC: debezium_statuses
```

**Configuration Debezium Connector PostgreSQL**
```json
{
  "name": "postgres-source-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres-host",
    "database.port": "5432",
    "database.user": "debezium_user",
    "database.password": "secret",
    "database.dbname": "mybank",
    "database.server.name": "bankdb",

    "table.include.list": "public.accounts,public.transactions,public.customers",

    "plugin.name": "pgoutput",
    "publication.name": "debezium_publication",

    "snapshot.mode": "initial",
    "snapshot.locking.mode": "none",

    "transforms": "route",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "$3"
  }
}
```

**Consumer Kafka → MongoDB (Node.js)**
```javascript
const { Kafka } = require('kafkajs');
const { MongoClient } = require('mongodb');

class CDCProcessor {
    constructor(kafkaConfig, mongoUri) {
        this.kafka = new Kafka(kafkaConfig);
        this.consumer = this.kafka.consumer({ groupId: 'mongodb-sink' });
        this.mongoClient = new MongoClient(mongoUri);
    }

    async start() {
        await this.mongoClient.connect();
        this.db = this.mongoClient.db('mybank');

        await this.consumer.connect();
        await this.consumer.subscribe({
            topics: ['accounts', 'transactions', 'customers'],
            fromBeginning: true
        });

        await this.consumer.run({
            eachMessage: async ({ topic, partition, message }) => {
                await this.processMessage(topic, message);
            }
        });
    }

    async processMessage(topic, message) {
        const change = JSON.parse(message.value.toString());
        const operation = change.op;  // c=create, u=update, d=delete

        const collection = this.db.collection(topic);

        try {
            switch (operation) {
                case 'c':  // INSERT
                case 'r':  // Snapshot read
                    await this.handleInsert(collection, change.after);
                    break;

                case 'u':  // UPDATE
                    await this.handleUpdate(collection, change.before, change.after);
                    break;

                case 'd':  // DELETE
                    await this.handleDelete(collection, change.before);
                    break;
            }

            // Commit offset
            await this.consumer.commitOffsets([{
                topic: message.topic,
                partition: message.partition,
                offset: (parseInt(message.offset) + 1).toString()
            }]);

        } catch (error) {
            console.error(`Error processing message:`, error);
            // Dead letter queue or retry logic
        }
    }

    async handleInsert(collection, data) {
        const doc = this.transformDocument(collection.collectionName, data);
        await collection.insertOne(doc);
    }

    async handleUpdate(collection, before, after) {
        const filter = { legacy_id: before.id };
        const update = this.buildUpdateOperation(before, after);
        await collection.updateOne(filter, update);
    }

    async handleDelete(collection, data) {
        await collection.deleteOne({ legacy_id: data.id });
    }

    transformDocument(collectionName, sqlRow) {
        // Transformation spécifique par collection
        const transformers = {
            accounts: this.transformAccount,
            transactions: this.transformTransaction,
            customers: this.transformCustomer
        };

        return transformers[collectionName].call(this, sqlRow);
    }

    transformAccount(sqlRow) {
        return {
            _id: new ObjectId(),
            legacy_id: sqlRow.id,
            account_number: sqlRow.account_number,
            customer_id: sqlRow.customer_id,
            balance: Decimal128.fromString(sqlRow.balance.toString()),
            currency: sqlRow.currency,
            status: sqlRow.status,
            created_at: new Date(sqlRow.created_at),
            updated_at: new Date()
        };
    }

    buildUpdateOperation(before, after) {
        const changes = {};

        for (const key in after) {
            if (before[key] !== after[key]) {
                changes[key] = after[key];
            }
        }

        return {
            $set: { ...changes, updated_at: new Date() }
        };
    }
}

// Démarrage
const processor = new CDCProcessor(
    { brokers: ['localhost:9092'] },
    'mongodb://localhost:27017'
);

processor.start();
```

**Étape 2 : Migration initiale (Snapshot)**

Le snapshot Debezium migre toutes les données existantes, puis passe en mode streaming pour les changements futurs.

**Monitoring du snapshot**
```bash
# Check Debezium connector status
curl -X GET http://localhost:8083/connectors/postgres-source-connector/status

# Check Kafka lag
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group mongodb-sink --describe

# MongoDB: Vérifier progression
mongosh --eval "
  db.accounts.countDocuments();
  db.transactions.countDocuments();
"
```

**Étape 3 : Phase de validation (1-2 semaines)**

Application en **dual-read mode** :
- Lectures depuis MongoDB (performance)
- Comparaison périodique avec SQL (validation)
- Écritures toujours sur SQL (source de vérité)

```javascript
// Dual-read implementation
class DualReadService {
    async getAccount(accountId) {
        // Read from MongoDB (fast)
        const mongoResult = await this.mongoDb.accounts.findOne(
            { account_number: accountId }
        );

        // En parallèle: Read from SQL (validation)
        const sqlResult = await this.sqlDb.query(
            'SELECT * FROM accounts WHERE account_number = $1',
            [accountId]
        );

        // Compare results (async, non-blocking)
        this.compareAndLog(mongoResult, sqlResult);

        // Return MongoDB result (fast path)
        return mongoResult;
    }

    async compareAndLog(mongoDoc, sqlRow) {
        const differences = this.findDifferences(mongoDoc, sqlRow);

        if (differences.length > 0) {
            await this.db.validation_errors.insertOne({
                timestamp: new Date(),
                collection: 'accounts',
                mongo_id: mongoDoc._id,
                sql_id: sqlRow.id,
                differences,
                mongo_data: mongoDoc,
                sql_data: sqlRow
            });

            // Alert si taux d'erreur > seuil
            const errorRate = await this.getErrorRate();
            if (errorRate > 0.01) {  // > 1%
                this.alertOps(`High validation error rate: ${errorRate}%`);
            }
        }
    }
}
```

**Étape 4 : Bascule finale (cutover)**

Après validation réussie (taux d'erreur < 0.1% pendant 1 semaine) :

```javascript
// Switch to MongoDB-only mode
class ProductionService {
    async getAccount(accountId) {
        // MongoDB devient source unique
        return await this.mongoDb.accounts.findOne(
            { account_number: accountId }
        );
    }

    async updateAccount(accountId, updates) {
        // Écriture sur MongoDB uniquement
        return await this.mongoDb.accounts.updateOne(
            { account_number: accountId },
            { $set: updates }
        );
    }
}

// Arrêt CDC après période de grâce (1 mois)
// Conservation SQL en archive read-only
```

---

### 3.3 Migration avec transformation ETL complexe

**Scénario : Legacy ERP (10 ans, schéma fortement normalisé, 200 tables)**

**Architecture ETL avec Apache Spark**
```
[Oracle DB] → [Spark ETL Job] → [MongoDB]
                   ↓
           [S3 Staging Area]
           [Data Quality Checks]
```

**Spark ETL Job (Scala/PySpark)**
```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
import pymongo

class ERPToMongoETL:
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("ERP_MongoDB_Migration") \
            .config("spark.mongodb.output.uri", "mongodb://localhost/erp") \
            .config("spark.jars.packages", "org.mongodb.spark:mongo-spark-connector_2.12:3.0.2") \
            .getOrCreate()

        self.oracle_props = {
            "user": "erp_reader",
            "password": "secret",
            "driver": "oracle.jdbc.driver.OracleDriver"
        }

    def extract_customers(self):
        """Extract customers avec 5 tables jointes"""

        # Lecture depuis Oracle avec jointures
        query = """
            SELECT
                c.customer_id,
                c.customer_name,
                c.tax_id,
                c.created_date,

                -- Adresses (multiple)
                ca.address_id,
                ca.address_type,
                ca.street,
                ca.city,
                ca.country,

                -- Contacts (multiple)
                cc.contact_id,
                cc.contact_type,
                cc.contact_value,

                -- Credit info
                cr.credit_limit,
                cr.credit_rating,

                -- Preferences
                cp.preference_key,
                cp.preference_value

            FROM customers c
            LEFT JOIN customer_addresses ca ON c.customer_id = ca.customer_id
            LEFT JOIN customer_contacts cc ON c.customer_id = cc.customer_id
            LEFT JOIN customer_credit cr ON c.customer_id = cr.customer_id
            LEFT JOIN customer_preferences cp ON c.customer_id = cp.customer_id
        """

        df = self.spark.read.jdbc(
            url="jdbc:oracle:thin:@oracle-host:1521:ORCL",
            table=f"({query})",
            properties=self.oracle_props
        )

        return df

    def transform_customers(self, df):
        """Transformation vers format MongoDB"""

        # Grouper par customer_id et agréger les relations
        result = df.groupBy("customer_id", "customer_name", "tax_id", "created_date",
                            "credit_limit", "credit_rating") \
            .agg(
                # Agréger adresses en array de structs
                collect_list(
                    struct(
                        col("address_id"),
                        col("address_type"),
                        col("street"),
                        col("city"),
                        col("country")
                    )
                ).alias("addresses"),

                # Agréger contacts en array
                collect_list(
                    when(col("contact_id").isNotNull(),
                        struct(
                            col("contact_id"),
                            col("contact_type"),
                            col("contact_value")
                        )
                    )
                ).alias("contacts"),

                # Agréger preferences en map
                map_from_entries(
                    collect_list(
                        struct(
                            col("preference_key"),
                            col("preference_value")
                        )
                    )
                ).alias("preferences")
            ) \
            .select(
                col("customer_id").alias("legacy_id"),
                col("customer_name").alias("name"),
                col("tax_id"),
                col("created_date"),
                struct(
                    col("credit_limit"),
                    col("credit_rating")
                ).alias("credit_info"),
                col("addresses"),
                col("contacts"),
                col("preferences"),
                current_timestamp().alias("migrated_at")
            )

        return result

    def validate_and_cleanse(self, df):
        """Data quality checks"""

        # Remove duplicates
        df = df.dropDuplicates(["legacy_id"])

        # Filter invalid records
        df = df.filter(col("name").isNotNull())
        df = df.filter(col("tax_id").rlike("^[A-Z]{2}[0-9]{9}$"))  # Regex validation

        # Fix NULL arrays
        df = df.withColumn("addresses",
                          when(size(col("addresses")) == 0, array()).otherwise(col("addresses")))

        # Log validation failures
        invalid_count = df.filter(col("name").isNull()).count()
        print(f"Filtered {invalid_count} invalid records")

        return df

    def load_to_mongodb(self, df):
        """Load vers MongoDB"""

        df.write \
            .format("mongo") \
            .mode("overwrite") \
            .option("database", "erp") \
            .option("collection", "customers") \
            .save()

    def run_customer_migration(self):
        """Pipeline complet"""

        # Extract
        print("Extracting from Oracle...")
        raw_df = self.extract_customers()

        # Transform
        print("Transforming data...")
        transformed_df = self.transform_customers(raw_df)

        # Validate
        print("Validating and cleansing...")
        clean_df = self.validate_and_cleanse(transformed_df)

        # Load
        print("Loading to MongoDB...")
        self.load_to_mongodb(clean_df)

        # Post-load indexing
        print("Creating indexes...")
        self.create_indexes()

        print("Migration complete!")

    def create_indexes(self):
        """Création indexes MongoDB post-load"""
        client = pymongo.MongoClient("mongodb://localhost:27017")
        db = client['erp']

        db.customers.create_index("legacy_id", unique=True)
        db.customers.create_index("name")
        db.customers.create_index("tax_id", unique=True)
        db.customers.create_index([("name", "text")])

# Exécution
if __name__ == "__main__":
    etl = ERPToMongoETL()
    etl.run_customer_migration()
```

**Optimisations Spark pour gros volumes**
```python
# Configuration pour cluster Spark
spark_conf = {
    "spark.executor.memory": "8g",
    "spark.executor.cores": "4",
    "spark.executor.instances": "10",
    "spark.sql.shuffle.partitions": "200",

    # MongoDB write configuration
    "spark.mongodb.output.maxBatchSize": "1024",
    "spark.mongodb.output.writeConcern.w": "1",

    # Optimization
    "spark.sql.adaptive.enabled": "true",
    "spark.sql.adaptive.coalescePartitions.enabled": "true"
}
```

---

## 📊 Phase 4 : Validation et Tests Post-Migration

### 4.1 Validation quantitative

**Script de validation automatisé**
```python
class MigrationValidator:
    def __init__(self, sql_conn, mongo_client):
        self.sql_conn = sql_conn
        self.mongo_db = mongo_client['mydb']
        self.report = []

    def validate_row_counts(self):
        """Validation des counts"""
        mappings = [
            ("customers", "customers"),
            ("products", "products"),
            ("orders", "orders")
        ]

        for sql_table, mongo_collection in mappings:
            sql_count = self.get_sql_count(sql_table)
            mongo_count = self.mongo_db[mongo_collection].count_documents({})

            match = sql_count == mongo_count
            self.report.append({
                "check": "row_count",
                "entity": sql_table,
                "sql_count": sql_count,
                "mongo_count": mongo_count,
                "match": match,
                "diff": abs(sql_count - mongo_count)
            })

    def validate_aggregates(self):
        """Validation agrégats métier"""

        # Exemple: Total des commandes par statut
        sql_query = """
            SELECT status, COUNT(*), SUM(total_amount)
            FROM orders
            GROUP BY status
        """

        mongo_pipeline = [
            {
                "$group": {
                    "_id": "$status",
                    "count": { "$sum": 1 },
                    "total": { "$sum": { "$toDouble": "$total" } }
                }
            }
        ]

        sql_results = self.execute_sql(sql_query)
        mongo_results = list(self.mongo_db.orders.aggregate(mongo_pipeline))

        # Comparaison
        for sql_row in sql_results:
            status, sql_count, sql_sum = sql_row
            mongo_row = next((m for m in mongo_results if m['_id'] == status), None)

            if mongo_row:
                count_match = sql_count == mongo_row['count']
                sum_match = abs(sql_sum - mongo_row['total']) < 0.01  # Tolérance décimale

                self.report.append({
                    "check": "aggregate",
                    "entity": f"orders_by_status_{status}",
                    "sql_count": sql_count,
                    "mongo_count": mongo_row['count'],
                    "sql_sum": sql_sum,
                    "mongo_sum": mongo_row['total'],
                    "match": count_match and sum_match
                })

    def validate_sample_records(self, sample_size=1000):
        """Validation échantillon de records"""

        # Sélection aléatoire de IDs depuis SQL
        sql_ids = self.get_random_ids("orders", sample_size)

        mismatches = []

        for sql_id in sql_ids:
            sql_record = self.get_sql_record("orders", sql_id)
            mongo_record = self.mongo_db.orders.find_one({"legacy_id": sql_id})

            if not mongo_record:
                mismatches.append({"id": sql_id, "error": "missing_in_mongodb"})
                continue

            # Comparaison champ par champ
            diffs = self.compare_records(sql_record, mongo_record)
            if diffs:
                mismatches.append({"id": sql_id, "differences": diffs})

        self.report.append({
            "check": "sample_validation",
            "sample_size": sample_size,
            "mismatches": len(mismatches),
            "accuracy": (sample_size - len(mismatches)) / sample_size * 100
        })

        return mismatches

    def generate_report(self):
        """Génération rapport HTML"""

        html = "<html><body><h1>Migration Validation Report</h1>"
        html += f"<p>Generated: {datetime.now()}</p>"

        html += "<h2>Summary</h2>"
        total_checks = len(self.report)
        passed = sum(1 for r in self.report if r.get('match', True))
        html += f"<p>Total checks: {total_checks}, Passed: {passed}, Failed: {total_checks - passed}</p>"

        html += "<table border='1'><tr><th>Check</th><th>Entity</th><th>Status</th><th>Details</th></tr>"

        for entry in self.report:
            status = "✓ PASS" if entry.get('match', True) else "✗ FAIL"
            html += f"<tr><td>{entry['check']}</td><td>{entry.get('entity', 'N/A')}</td><td>{status}</td><td>{entry}</td></tr>"

        html += "</table></body></html>"

        with open("migration_validation_report.html", "w") as f:
            f.write(html)
```

### 4.2 Tests fonctionnels

**Suite de tests applicatifs**
```javascript
// Exemple avec Jest (Node.js)
describe('Post-Migration Functional Tests', () => {
    let db;

    beforeAll(async () => {
        const client = await MongoClient.connect('mongodb://localhost:27017');
        db = client.db('mydb');
    });

    test('Customer CRUD operations', async () => {
        // Create
        const customer = {
            name: "Test Customer",
            email: "test@example.com",
            addresses: [{ city: "Paris" }]
        };
        const result = await db.customers.insertOne(customer);
        expect(result.insertedId).toBeDefined();

        // Read
        const found = await db.customers.findOne({ _id: result.insertedId });
        expect(found.name).toBe("Test Customer");

        // Update
        await db.customers.updateOne(
            { _id: result.insertedId },
            { $set: { name: "Updated Name" } }
        );

        // Delete
        await db.customers.deleteOne({ _id: result.insertedId });
    });

    test('Order with items creation', async () => {
        const order = {
            customer_id: 123,
            order_date: new Date(),
            items: [
                { product_id: 1, quantity: 2, unit_price: 25.00 }
            ],
            total: 50.00
        };

        const result = await db.orders.insertOne(order);
        expect(result.insertedId).toBeDefined();

        // Verify atomicity (items embedded)
        const savedOrder = await db.orders.findOne({ _id: result.insertedId });
        expect(savedOrder.items.length).toBe(1);
        expect(savedOrder.total).toBe(50.00);
    });

    test('Aggregation pipeline (top customers)', async () => {
        const topCustomers = await db.orders.aggregate([
            { $match: { order_date: { $gte: new Date('2024-01-01') } } },
            { $group: {
                _id: "$customer_id",
                total_spent: { $sum: "$total" },
                order_count: { $sum: 1 }
            }},
            { $sort: { total_spent: -1 } },
            { $limit: 10 }
        ]).toArray();

        expect(topCustomers.length).toBeLessThanOrEqual(10);
        expect(topCustomers[0].total_spent).toBeGreaterThan(0);
    });

    test('Performance: Query with index', async () => {
        const start = Date.now();

        const results = await db.products.find({
            status: "active",
            price: { $gte: 10, $lte: 100 }
        }).limit(100).toArray();

        const duration = Date.now() - start;

        expect(results.length).toBeGreaterThan(0);
        expect(duration).toBeLessThan(100);  // < 100ms
    });
});
```

---

## 🚨 Phase 5 : Problèmes Courants et Résolutions

### 5.1 Performance dégradée post-migration

**Symptôme** : Queries MongoDB plus lentes que SQL

**Diagnostic**
```javascript
// Vérifier si index utilisé
db.products.find({ status: "active" }).explain("executionStats")

// Si COLLSCAN → Manque d'index !
```

**Résolution**
```javascript
// Créer indexes manquants
db.products.createIndex({ status: 1 })
db.products.createIndex({ status: 1, price: 1 })  // Compound

// Revoir modélisation si trop de $lookup
// → Considérer dénormalisation
```

### 5.2 Documents dépassant 16 Mo

**Symptôme** : Erreur "Document exceeds maximum size"

**Résolution**
```javascript
// Solution 1: Bucketing pour arrays volumineuses
// (voir section 2.2 - Bucketing Pattern)

// Solution 2: GridFS pour champs > 16 Mo
// Solution 3: Référencement + collection séparée
```

### 5.3 Perte de contraintes FK

**Symptôme** : Données orphelines, incohérences

**Résolution**
```javascript
// 1. Validation applicative stricte
// 2. Schema validation MongoDB
// 3. Script nettoyage périodique

// Exemple: Détecter commandes orphelines
db.orders.aggregate([
    {
        $lookup: {
            from: "customers",
            localField: "customer_id",
            foreignField: "_id",
            as: "customer"
        }
    },
    { $match: { customer: [] } },  // Customer inexistant
    { $project: { _id: 1, customer_id: 1 } }
])
```

### 5.4 Gestion des transactions

**Symptôme** : Besoin de transactions multi-documents non anticipé

**Résolution**
```javascript
// MongoDB 4.0+ : Transactions multi-documents
const session = client.startSession();

try {
    await session.withTransaction(async () => {
        await db.accounts.updateOne(
            { _id: sourceAccount },
            { $inc: { balance: -amount } },
            { session }
        );

        await db.accounts.updateOne(
            { _id: targetAccount },
            { $inc: { balance: amount } },
            { session }
        );
    });
} finally {
    await session.endSession();
}
```

---

## 🎯 Checklist Complète de Migration

### Pré-Migration
- [ ] Audit complet schéma SQL (tables, relations, contraintes)
- [ ] Analyse patterns d'accès (queries fréquentes)
- [ ] Dimensionnement MongoDB (CPU, RAM, stockage)
- [ ] POC sur dataset réel (1-10% des données)
- [ ] Design modèle MongoDB validé par équipe
- [ ] Choix stratégie migration (big bang / incrémental / CDC)
- [ ] Scripts migration développés et testés
- [ ] Plan de rollback documenté
- [ ] Environnements test/staging préparés

### Pendant Migration
- [ ] Backup complet SQL avant démarrage
- [ ] Monitoring en temps réel (throughput, lag, erreurs)
- [ ] Validation incrémentale (samples, row counts)
- [ ] Communication équipes (statut, ETA)
- [ ] Logs détaillés de toute opération

### Post-Migration
- [ ] Validation exhaustive données (100% row counts, agrégats métier)
- [ ] Tests fonctionnels complets (suite de tests automatisés)
- [ ] Tests performance (benchmarks)
- [ ] Création indexes nécessaires
- [ ] Configuration monitoring production
- [ ] Formation équipes MongoDB
- [ ] Documentation modèle MongoDB
- [ ] Décommissionnement SQL planifié (si applicable)

---

## 📚 Conclusion

La migration SQL → MongoDB est un projet d'envergure nécessitant :

1. **Analyse approfondie** : Ne pas sous-estimer la complexité du schéma source
2. **Refonte modélisation** : Penser "document-oriented", pas "table-oriented"
3. **Stratégie adaptée** : Choisir entre big bang / incrémental selon contexte
4. **Validation rigoureuse** : Tests quantitatifs ET qualitatifs
5. **Approche itérative** : POC → pilot → production progressive

**Points critiques de succès** :
- ✅ Sponsorship management fort
- ✅ Équipe mixte (SQL + MongoDB experts)
- ✅ Formation développeurs (mindset change)
- ✅ Investissement validation (30-40% du temps projet)
- ✅ Plan de rollback solide

**Dans la section suivante** : 19.2 Outils de migration - Analyse détaillée des outils disponibles pour faciliter la migration.

⏭️ [Outils de migration](/19-migration-integration/02-outils-migration.md)
