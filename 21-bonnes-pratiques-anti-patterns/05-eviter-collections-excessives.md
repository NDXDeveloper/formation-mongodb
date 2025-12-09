🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.5 Éviter les Collections Excessives

## Introduction

Si MongoDB offre une flexibilité totale sur le nombre de collections dans une base de données, cette liberté peut devenir un piège. Créer une nouvelle collection semble simple et sans coût, mais la multiplication incontrôlée des collections crée des problèmes de performance, de maintenance et d'architecture qui s'accumulent avec le temps.

Un nombre excessif de collections est souvent le symptôme d'une mauvaise modélisation, d'une sur-normalisation ou d'une incompréhension des capacités de MongoDB. Cette section explore quand créer une nouvelle collection est approprié, quand c'est un anti-pattern, et comment consolider efficacement une architecture avec trop de collections.

---

## Comprendre les Coûts des Collections

### Impact Technique

Chaque collection dans MongoDB a un coût :

```javascript
// Coût par collection
- Métadonnées de collection dans le catalogue
- Index _id automatique (consomme RAM et disque)
- Descripteurs de fichiers (limites OS)
- Overhead de gestion dans WiredTiger
- Entrées dans les statistiques et monitoring
- Espace dans les backups et dumps
```

**Impact mesuré** :
```javascript
// Base avec 10 collections vs 1,000 collections
// Même nombre total de documents (10M)

10 collections :
- Démarrage MongoDB : 2 secondes
- Backup (mongodump) : 30 secondes
- Mémoire métadonnées : 5 MB
- Complexité gestion : Faible

1,000 collections :
- Démarrage MongoDB : 45 secondes (20x plus lent)
- Backup (mongodump) : 15 minutes (30x plus lent)
- Mémoire métadonnées : 150 MB (30x plus)
- Complexité gestion : Élevée
```

### Limites Pratiques

MongoDB peut techniquement gérer des milliers de collections, mais les problèmes apparaissent :

| Nombre Collections | État | Symptômes |
|-------------------|------|-----------|
| **< 50** | ✅ Optimal | Aucun problème |
| **50 - 200** | ⚠️ Acceptable | Impact mineur |
| **200 - 500** | ⚠️ Problématique | Temps de démarrage accru |
| **500 - 1,000** | ❌ Excessif | Performance dégradée |
| **> 1,000** | ❌ Critique | Problèmes majeurs |

---

## ✅ DO : Consolider les Collections avec des Données Similaires

**Explication** : Les données qui partagent la même structure et le même cycle de vie doivent résider dans une seule collection, différenciées par un champ discriminant.

**Pattern recommandé - Polymorphisme** :
```javascript
// ❌ Anti-pattern : Collection par type de produit
// Collections: electronics, clothing, books, furniture, toys, ...
// Résultat : 50+ collections pour les produits

// Collection: electronics
{
  _id: ObjectId("..."),
  name: "Laptop",
  brand: "Dell",
  processor: "Intel i7",
  ram: "16GB"
}

// Collection: clothing
{
  _id: ObjectId("..."),
  name: "T-Shirt",
  brand: "Nike",
  size: "M",
  color: "Blue"
}

// ✅ Bonne pratique : Une collection avec discriminant
// Collection: products
{
  _id: ObjectId("..."),
  type: "electronics",  // Discriminant
  name: "Laptop",
  brand: "Dell",
  // Champs spécifiques aux electronics
  specs: {
    processor: "Intel i7",
    ram: "16GB"
  }
}

{
  _id: ObjectId("..."),
  type: "clothing",  // Discriminant
  name: "T-Shirt",
  brand: "Nike",
  // Champs spécifiques aux vêtements
  details: {
    size: "M",
    color: "Blue"
  }
}
```

**Avantages** :

### 1. Requêtes Simplifiées
```javascript
// ❌ Recherche dans 50 collections
const results = [];
for (const collection of ['electronics', 'clothing', 'books', ...]) {
  const items = await db[collection].find({ brand: "Nike" }).toArray();
  results.push(...items);
}

// ✅ Une seule requête
const results = await db.products.find({ brand: "Nike" }).toArray();
```

### 2. Index Unifiés
```javascript
// ✅ Un seul index pour tous les produits
db.products.createIndex({ brand: 1, name: 1 });

// ❌ 50 index identiques à maintenir
db.electronics.createIndex({ brand: 1, name: 1 });
db.clothing.createIndex({ brand: 1, name: 1 });
// ... 48 autres fois
```

### 3. Agrégations Globales
```javascript
// ✅ Statistiques globales simples
db.products.aggregate([
  {
    $group: {
      _id: "$type",
      count: { $sum: 1 },
      avgPrice: { $avg: "$price" }
    }
  }
]);

// ❌ Agrégations cross-collection complexes
// Nécessite $unionWith ou code applicatif
```

**Quand utiliser ce pattern** :
- Entités du même domaine métier
- Structure de base similaire
- Requêtes cross-type fréquentes
- Cycle de vie identique

---

## ❌ DON'T : Créer une Collection par Utilisateur

**Explication** : Créer une collection dédiée par utilisateur est un anti-pattern majeur qui mène rapidement à une explosion du nombre de collections.

**Anti-pattern catastrophique** :
```javascript
// ❌ Collection par utilisateur
// Base de données avec 100,000 utilisateurs = 100,000 collections!

// Collection: user_123_messages
{
  _id: ObjectId("..."),
  from: "alice",
  to: "bob",
  text: "Hello",
  timestamp: ISODate("...")
}

// Collection: user_456_messages
{
  _id: ObjectId("..."),
  from: "charlie",
  to: "alice",
  text: "Hi",
  timestamp: ISODate("...")
}

// ... 99,998 autres collections
```

**Conséquences désastreuses** :

### 1. Explosion des Métadonnées
```javascript
// 100,000 collections
// Chaque collection : ~2 KB de métadonnées
= 200 MB juste pour les métadonnées
= Temps de démarrage de plusieurs minutes
= Listing des collections prend 10+ secondes
```

### 2. Impossibilité de Requêtes Globales
```javascript
// ❌ Rechercher dans tous les messages
// Nécessite d'itérer 100,000 collections!
const allMessages = [];
for (let userId = 1; userId <= 100000; userId++) {
  const messages = await db[`user_${userId}_messages`]
    .find({ text: /important/ })
    .toArray();
  allMessages.push(...messages);
}
// Temps d'exécution : plusieurs minutes
```

### 3. Gestion Cauchemardesque
```javascript
// ❌ Opérations de maintenance
// Créer un index sur toutes les collections
for (let userId = 1; userId <= 100000; userId++) {
  await db[`user_${userId}_messages`].createIndex({ timestamp: -1 });
}
// Temps d'exécution : plusieurs heures

// Migration de schéma : impossible en pratique
// Backup/restore : extrêmement lent
```

### 4. Limites Système
```javascript
// Limites OS
- Descripteurs de fichiers : ~1024 par défaut
- Inodes : limités sur certains systèmes
- Mémoire : overhead significatif

// MongoDB atteint ses limites
- Namespace lock contention
- Catalogue surchargé
- Performance globale dégradée
```

**Solution appropriée** :
```javascript
// ✅ Collection unique avec champ userId
// Collection: messages (une seule)
{
  _id: ObjectId("..."),
  userId: "user_123",  // Propriétaire du message
  from: "alice",
  to: "bob",
  text: "Hello",
  timestamp: ISODate("...")
}

// Index optimisé
db.messages.createIndex({ userId: 1, timestamp: -1 });

// Requête d'un utilisateur spécifique
db.messages.find({ userId: "user_123" });

// Requête globale possible
db.messages.find({ text: /important/ });
```

---

## ✅ DO : Utiliser le Pattern Model per Tenant pour Multi-Tenancy

**Explication** : Pour les applications multi-tenant, il existe plusieurs stratégies avec différents compromis.

### Stratégie 1 : Collection Partagée (Recommandée)
```javascript
// ✅ Tous les tenants dans la même collection
// Collection: documents
{
  _id: ObjectId("..."),
  tenantId: "tenant_abc",  // Discriminant de tenant
  documentName: "Contract.pdf",
  content: "...",
  createdAt: ISODate("...")
}

// Index composé pour isolation
db.documents.createIndex({ tenantId: 1, createdAt: -1 });

// Requête isolée par tenant
db.documents.find({ tenantId: "tenant_abc" });
```

**Avantages** :
- Scalabilité : Gère des milliers de tenants
- Performance : Index et requêtes optimisés
- Maintenance : Une seule structure à gérer
- Coût : Minimum d'overhead

**Inconvénients** :
- Sécurité : Dépend de l'isolation applicative
- Risque : Bug pourrait exposer données cross-tenant

### Stratégie 2 : Base de Données par Tenant
```javascript
// ✅ Pour petits nombres de tenants avec besoins spécifiques
// Base: tenant_abc
{
  collections: ["documents", "users", "settings"]
}

// Base: tenant_xyz
{
  collections: ["documents", "users", "settings"]
}
```

**Avantages** :
- Isolation forte
- Backup/restore par tenant
- Personnalisation par tenant

**Inconvénients** :
- Limite à ~100-200 tenants
- Requêtes cross-tenant impossibles
- Overhead par base de données

### Stratégie 3 : Collection par Tenant (À Éviter)
```javascript
// ⚠️ Uniquement pour < 50 tenants
// Collections: tenant_abc_documents, tenant_xyz_documents, ...
```

**Quand utiliser** : Très rare, seulement si :
- Moins de 50 tenants
- Isolation stricte requise
- Pas de requêtes cross-tenant
- Besoins de performances très spécifiques par tenant

---

## ❌ DON'T : Créer des Collections pour Chaque Date/Période

**Explication** : Créer une collection par jour, mois ou année est un pattern obsolète qui crée une explosion de collections.

**Anti-pattern** :
```javascript
// ❌ Collection par jour pour les logs
// Collections:
// logs_2024_01_01
// logs_2024_01_02
// logs_2024_01_03
// ... 365 collections par an
// ... 3,650 collections sur 10 ans!

// Collection: logs_2024_01_15
{
  _id: ObjectId("..."),
  level: "info",
  message: "User logged in",
  timestamp: ISODate("2024-01-15T10:30:00Z")
}
```

**Problèmes** :

### 1. Requêtes Cross-Période Complexes
```javascript
// ❌ Rechercher sur une semaine
const results = [];
for (let day = 1; day <= 7; day++) {
  const collection = `logs_2024_01_${day.toString().padStart(2, '0')}`;
  const logs = await db[collection].find({ level: "error" }).toArray();
  results.push(...logs);
}
```

### 2. Agrégations Impossibles
```javascript
// ❌ Statistiques mensuelles
// Nécessite d'agréger 30 collections manuellement
```

### 3. Maintenance Explosive
```javascript
// ❌ Créer les index quotidiennement
// Nettoyer les anciennes collections manuellement
// Gérer 365+ collections par an
```

**Solution moderne** :
```javascript
// ✅ Collection unique avec Time Series (MongoDB 5.0+)
db.createCollection("logs", {
  timeseries: {
    timeField: "timestamp",
    metaField: "metadata",
    granularity: "seconds"
  }
});

// Insertion
db.logs.insertOne({
  timestamp: new Date(),
  metadata: { level: "info", source: "app" },
  message: "User logged in"
});

// Requêtes sur n'importe quelle période
db.logs.find({
  timestamp: {
    $gte: new Date("2024-01-01"),
    $lt: new Date("2024-01-08")
  },
  "metadata.level": "error"
});

// Expiration automatique avec TTL
db.logs.createIndex(
  { timestamp: 1 },
  { expireAfterSeconds: 2592000 }  // 30 jours
);
```

**Ou collection classique** :
```javascript
// ✅ Collection unique avec index TTL
db.logs.createIndex({ timestamp: 1 }, { expireAfterSeconds: 2592000 });

// Partition virtuelle avec index
db.logs.createIndex({ year: 1, month: 1, timestamp: -1 });

// Requêtes efficaces
db.logs.find({
  year: 2024,
  month: 1,
  timestamp: { $gte: ISODate("2024-01-15") }
});
```

---

## ✅ DO : Limiter le Nombre de Collections Techniques

**Explication** : Les collections système et techniques doivent être maintenues au minimum et bien documentées.

**Collections techniques appropriées** :
```javascript
// ✅ Collections système essentielles
{
  "_migrations": "Historique des migrations de schéma",
  "_sessions": "Sessions utilisateur actives",
  "_jobs": "Queue de jobs asynchrones",
  "_audit_logs": "Logs d'audit sécurité",
  "_feature_flags": "Configuration des feature flags",
  "_cache": "Cache applicatif (si nécessaire)"
}

// Total : 6 collections techniques
// Collections métier : 15-20 collections
// Total global : ~25 collections ✓
```

**Anti-pattern** :
```javascript
// ❌ Prolifération de collections techniques
{
  "_migrations_v1": "...",
  "_migrations_v2": "...",
  "_sessions_active": "...",
  "_sessions_expired": "...",
  "_jobs_pending": "...",
  "_jobs_completed": "...",
  "_jobs_failed": "...",
  "_cache_users": "...",
  "_cache_products": "...",
  "_cache_orders": "...",
  "_logs_app": "...",
  "_logs_api": "...",
  "_logs_worker": "...",
  // ... 30+ collections techniques
}
```

**Consolidation** :
```javascript
// ✅ Une collection avec discriminant
// Collection: _jobs
{
  _id: ObjectId("..."),
  status: "pending",  // pending, completed, failed
  type: "email",
  payload: { ... },
  createdAt: ISODate("...")
}

// Index pour chaque status
db._jobs.createIndex({ status: 1, createdAt: 1 });

// Collection: _cache
{
  _id: "users:123",  // Type préfixé dans l'ID
  type: "user",
  data: { ... },
  expiresAt: ISODate("...")
}
```

---

## ❌ DON'T : Créer des Collections "Temporaires" Persistantes

**Explication** : Les collections créées "temporairement" pour des tests ou des migrations ont tendance à devenir permanentes et à s'accumuler.

**Anti-pattern** :
```javascript
// ❌ Collections temporaires qui s'accumulent
db.getCollectionNames();
[
  "users",
  "products",
  "orders",
  "users_backup_20231201",
  "users_backup_20231215",
  "users_backup_20240101",
  "products_migration_test",
  "products_migration_test_v2",
  "products_migration_final",
  "temp_analytics_2023",
  "temp_analytics_2024",
  "export_users_jan",
  "export_users_feb",
  "test_collection",
  "test_collection_2",
  "alice_test",
  "bob_experiment",
  // ... 30 collections "temporaires"
]
```

**Problèmes** :
- Confusion sur ce qui est actif
- Gaspillage d'espace disque
- Temps de backup augmenté
- Risque d'utilisation accidentelle
- Complexité de maintenance

**Solution** :
```javascript
// ✅ Convention de nommage et nettoyage
// Préfixe pour collections temporaires
const tempCollectionName = `_temp_${Date.now()}_migration`;

// Créer avec expiration implicite
await db.createCollection(tempCollectionName);

// Documenter dans une collection de tracking
await db._temp_collections.insertOne({
  name: tempCollectionName,
  purpose: "User migration test",
  createdBy: "alice",
  createdAt: new Date(),
  expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)  // 7 jours
});

// Script de nettoyage automatique
async function cleanupTempCollections() {
  const now = new Date();
  const expired = await db._temp_collections.find({
    expiresAt: { $lt: now }
  }).toArray();

  for (const coll of expired) {
    console.log(`Dropping expired collection: ${coll.name}`);
    await db[coll.name].drop();
    await db._temp_collections.deleteOne({ _id: coll._id });
  }
}

// Exécuter quotidiennement
```

---

## ✅ DO : Utiliser le Pattern Computed pour Éviter les Collections Dérivées

**Explication** : Plutôt que créer des collections pour stocker des données calculées, utiliser des agrégations et des vues.

**Anti-pattern** :
```javascript
// ❌ Collections dérivées multiples
// Collection: orders (source)
{
  _id: ObjectId("..."),
  customerId: "CUST123",
  total: 150.00,
  date: ISODate("2024-01-15")
}

// Collection: daily_sales (dérivée)
{
  _id: "2024-01-15",
  totalSales: 15000.00,
  orderCount: 100
}

// Collection: customer_totals (dérivée)
{
  _id: "CUST123",
  totalSpent: 1500.00,
  orderCount: 10
}

// Collection: monthly_stats (dérivée)
// ... et ainsi de suite
// Résultat : 5-10 collections dérivées à maintenir
```

**Problèmes** :
- Synchronisation complexe
- Risque d'incohérence
- Stockage redondant
- Maintenance multi-collections

**Solution - MongoDB Views** :
```javascript
// ✅ Vue pour statistiques quotidiennes
db.createView("daily_sales", "orders", [
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%d", date: "$date" }
      },
      totalSales: { $sum: "$total" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { _id: -1 } }
]);

// Utilisation (identique à une collection)
db.daily_sales.find({ _id: "2024-01-15" });

// ✅ Vue pour totaux clients
db.createView("customer_totals", "orders", [
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$total" },
      orderCount: { $sum: 1 },
      lastOrderDate: { $max: "$date" }
    }
  }
]);

// Requête
db.customer_totals.find({ totalSpent: { $gt: 1000 } });
```

**Avantages des vues** :
- Toujours à jour automatiquement
- Pas de synchronisation nécessaire
- Pas de stockage redondant
- Maintenance simplifiée

**Quand utiliser des collections matérialisées** :
```javascript
// ✅ Si les agrégations sont trop coûteuses
// Utiliser $out ou $merge périodiquement
db.orders.aggregate([
  {
    $group: {
      _id: "$customerId",
      totalSpent: { $sum: "$total" },
      orderCount: { $sum: 1 }
    }
  },
  {
    $merge: {
      into: "customer_stats",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
]);

// Rafraîchir toutes les heures ou quotidiennement
// Mais : 1 collection matérialisée, pas 10
```

---

## ✅ DO : Auditer et Nettoyer Régulièrement

**Explication** : Maintenir un inventaire des collections et nettoyer régulièrement celles qui sont obsolètes.

**Audit régulier** :
```javascript
// ✅ Script d'audit des collections
async function auditCollections() {
  const collections = await db.listCollections().toArray();
  const report = [];

  for (const collInfo of collections) {
    const collName = collInfo.name;
    const stats = await db[collName].stats();

    const info = {
      name: collName,
      size: stats.size,
      count: stats.count,
      avgObjSize: stats.avgObjSize,
      indexes: stats.nindexes,
      // Dernière modification (approximation)
      lastModified: await getLastModified(collName)
    };

    report.push(info);
  }

  // Identifier les collections suspectes
  const suspicious = report.filter(c =>
    c.count === 0 ||  // Vide
    c.lastModified < Date.now() - 90 * 24 * 60 * 60 * 1000 ||  // Pas modifiée depuis 90 jours
    c.name.includes('temp') ||  // Nom suspect
    c.name.includes('test') ||
    c.name.includes('backup')
  );

  return { report, suspicious };
}

async function getLastModified(collName) {
  const doc = await db[collName].findOne({}, { sort: { _id: -1 } });
  return doc ? doc._id.getTimestamp() : null;
}
```

**Documentation des collections** :
```javascript
// ✅ Maintenir un catalogue de collections
// Collection: _collection_registry
{
  _id: "users",
  purpose: "User accounts and profiles",
  owner: "auth-team",
  createdAt: ISODate("2023-01-01"),
  status: "active",
  retentionPolicy: "indefinite",
  dependencies: ["orders", "sessions"]
}

{
  _id: "orders",
  purpose: "Customer orders",
  owner: "commerce-team",
  createdAt: ISODate("2023-01-01"),
  status: "active",
  retentionPolicy: "7 years",
  dependencies: ["users", "products"]
}

{
  _id: "_temp_migration_20240115",
  purpose: "Temporary migration workspace",
  owner: "alice",
  createdAt: ISODate("2024-01-15"),
  status: "temporary",
  retentionPolicy: "7 days",
  dependencies: []
}
```

**Process de nettoyage** :
```javascript
// ✅ Process de review trimestriel
async function quarterlyCleanup() {
  const { suspicious } = await auditCollections();

  console.log(`Found ${suspicious.length} collections to review:`);

  for (const coll of suspicious) {
    console.log(`\nCollection: ${coll.name}`);
    console.log(`  Documents: ${coll.count}`);
    console.log(`  Size: ${(coll.size / 1024 / 1024).toFixed(2)} MB`);
    console.log(`  Last modified: ${coll.lastModified}`);

    // Demander confirmation avant suppression
    const registry = await db._collection_registry.findOne({ _id: coll.name });

    if (!registry || registry.status === 'temporary') {
      console.log(`  ⚠️  CANDIDATE FOR DELETION`);
    }
  }
}
```

---

## ❌ DON'T : Créer des Collections pour Chaque Variation de Structure

**Explication** : Même avec des structures légèrement différentes, privilégier une collection polymorphe.

**Anti-pattern** :
```javascript
// ❌ Collection par variation mineure
// Collections:
// - events_click
// - events_pageview
// - events_purchase
// - events_signup
// - events_logout
// ... 50 types d'événements = 50 collections

// Collection: events_click
{
  _id: ObjectId("..."),
  timestamp: ISODate("..."),
  userId: "user123",
  elementId: "btn_submit",
  pageUrl: "/checkout"
}

// Collection: events_pageview
{
  _id: ObjectId("..."),
  timestamp: ISODate("..."),
  userId: "user123",
  pageUrl: "/products",
  referrer: "google.com"
}
```

**Solution polymorphe** :
```javascript
// ✅ Collection unique avec structure flexible
// Collection: events
{
  _id: ObjectId("..."),
  type: "click",  // Discriminant
  timestamp: ISODate("..."),
  userId: "user123",
  // Champs communs
  sessionId: "sess_456",
  // Champs spécifiques dans un sous-objet
  data: {
    elementId: "btn_submit",
    pageUrl: "/checkout"
  }
}

{
  _id: ObjectId("..."),
  type: "pageview",
  timestamp: ISODate("..."),
  userId: "user123",
  sessionId: "sess_456",
  data: {
    pageUrl: "/products",
    referrer: "google.com"
  }
}

// Index efficace
db.events.createIndex({ type: 1, timestamp: -1 });
db.events.createIndex({ userId: 1, timestamp: -1 });

// Requêtes simples
db.events.find({ type: "click" });
db.events.find({ userId: "user123" });

// Agrégations puissantes
db.events.aggregate([
  {
    $group: {
      _id: "$type",
      count: { $sum: 1 }
    }
  }
]);
```

---

## ✅ DO : Établir une Politique de Gouvernance

**Explication** : Définir des règles claires sur quand créer une nouvelle collection.

**Politique recommandée** :
```javascript
/**
 * POLITIQUE DE CRÉATION DE COLLECTIONS
 *
 * === AUTORISATION AUTOMATIQUE ===
 * Une nouvelle collection peut être créée SI :
 * 1. Domaine métier clairement distinct
 * 2. Aucune collection existante ne convient
 * 3. Plus de 1,000 documents attendus
 * 4. Cycle de vie indépendant
 *
 * === REVIEW REQUISE ===
 * Une review est obligatoire SI :
 * 1. Doute sur la pertinence
 * 2. Structure similaire à collection existante
 * 3. Collection "temporaire" ou "test"
 * 4. Nombre total de collections > 30
 *
 * === INTERDICTIONS ===
 * JAMAIS créer de collection pour :
 * 1. Chaque utilisateur individuel
 * 2. Chaque date/période temporelle
 * 3. Chaque variation mineure de structure
 * 4. Données dérivées (utiliser vues)
 * 5. Cache (utiliser collection cache unique)
 *
 * === NOMMAGE ===
 * - Métier : camelCase (users, orderItems)
 * - Technique : _prefixe (_migrations, _cache)
 * - Temporaire : _temp_* avec date (_temp_20240115_test)
 *
 * === DOCUMENTATION ===
 * Chaque nouvelle collection DOIT :
 * 1. Être documentée dans _collection_registry
 * 2. Avoir un owner désigné
 * 3. Avoir une politique de rétention
 * 4. Être revue trimestriellement
 */
```

**Process d'approbation** :
```javascript
// ✅ Template de demande de nouvelle collection
const collectionRequest = {
  name: "proposedCollectionName",
  purpose: "Clear business purpose",
  justification: "Why existing collections don't work",
  owner: "team-name",
  estimatedSize: {
    documents: 100000,
    averageDocSize: 2048,  // bytes
    growth: "10% monthly"
  },
  indexes: [
    { keys: { field1: 1, field2: -1 }, rationale: "..." }
  ],
  retentionPolicy: "3 years",
  alternativesConsidered: [
    "Why not reuse collection X?",
    "Why not use polymorphic pattern?"
  ],
  requestedBy: "developer-name",
  requestDate: new Date()
};

// Review par l'équipe architecture
```

---

## Stratégies de Consolidation

### ✅ DO : Migrer Progressivement vers Moins de Collections

**Explication** : Si vous avez déjà trop de collections, consolider progressivement.

**Plan de consolidation** :
```javascript
// Phase 1 : Inventaire et catégorisation
const collections = await db.listCollections().toArray();
const categorized = {
  active: [],      // Collections actives et nécessaires
  temporary: [],   // Collections temporaires à supprimer
  candidates: [],  // Collections candidates à la consolidation
  technical: []    // Collections techniques
};

for (const coll of collections) {
  // Catégoriser chaque collection
  if (coll.name.startsWith('_temp')) {
    categorized.temporary.push(coll.name);
  } else if (canConsolidate(coll)) {
    categorized.candidates.push(coll.name);
  }
  // ...
}

// Phase 2 : Supprimer les temporaires
for (const collName of categorized.temporary) {
  await db[collName].drop();
}

// Phase 3 : Consolider les candidates
// Exemple : Consolider product_* en products
const productCollections = categorized.candidates.filter(
  name => name.startsWith('product_')
);

// Créer la collection consolidée
db.createCollection('products');

// Migrer chaque collection
for (const collName of productCollections) {
  const type = collName.replace('product_', '');

  // Ajouter le champ type et copier
  const cursor = db[collName].find();
  await cursor.forEach(async doc => {
    doc.type = type;
    await db.products.insertOne(doc);
  });

  // Vérifier et supprimer l'ancienne
  const oldCount = await db[collName].countDocuments();
  const newCount = await db.products.countDocuments({ type });

  if (oldCount === newCount) {
    await db[collName].drop();
  }
}
```

**Migration avec double-écriture** :
```javascript
// ✅ Migration sans downtime
// Phase 1 : Double écriture
async function createProduct(data) {
  const type = data.category;

  // Écriture dans l'ancienne collection (legacy)
  await db[`product_${type}`].insertOne(data);

  // Écriture dans la nouvelle collection (nouvelle)
  await db.products.insertOne({ ...data, type });
}

// Phase 2 : Migration des données historiques
// (script de migration en arrière-plan)

// Phase 3 : Switch lecture vers nouvelle collection
async function getProduct(id) {
  // Lire depuis la nouvelle collection
  return await db.products.findOne({ _id: id });
}

// Phase 4 : Arrêter double écriture
async function createProduct(data) {
  // Écriture uniquement dans la nouvelle collection
  await db.products.insertOne({ ...data, type: data.category });
}

// Phase 5 : Supprimer anciennes collections
for (const collName of productCollections) {
  await db[collName].drop();
}
```

---

## Métriques et Monitoring

### ✅ DO : Suivre l'Évolution du Nombre de Collections

**Métriques importantes** :
```javascript
// ✅ Dashboard de monitoring
const metrics = {
  totalCollections: 42,

  // Par catégorie
  business: 25,        // Collections métier
  technical: 8,        // Collections techniques
  temporary: 9,        // À nettoyer

  // Tendance
  lastMonth: 38,
  growth: +4,          // +4 collections ce mois

  // Distribution taille
  empty: 5,            // Collections vides (candidates à suppression)
  small: 15,           // < 1,000 docs
  medium: 18,          // 1,000 - 1M docs
  large: 4,            // > 1M docs

  // Alertes
  over50: false,       // ✅ Sous la limite de 50
  hasTemp: true,       // ⚠️ Collections temporaires présentes
  hasEmpty: true       // ⚠️ Collections vides présentes
};

// Alertes automatiques
if (metrics.totalCollections > 50) {
  alert("Too many collections - review needed");
}

if (metrics.temporary > 5) {
  alert("Too many temporary collections - cleanup needed");
}
```

---

## Checklist Collections

### Prévention
- [ ] Politique de création documentée
- [ ] Process d'approbation en place
- [ ] Limite cible < 50 collections
- [ ] Pas de collections par user/date/type mineur
- [ ] Pattern polymorphe utilisé quand approprié

### Documentation
- [ ] Registre de collections maintenu
- [ ] Owner identifié pour chaque collection
- [ ] Purpose clairement documenté
- [ ] Politique de rétention définie

### Monitoring
- [ ] Nombre de collections suivi
- [ ] Alertes configurées (>50, temporaires)
- [ ] Audit trimestriel planifié
- [ ] Collections vides identifiées

### Consolidation
- [ ] Collections temporaires nettoyées
- [ ] Collections similaires consolidées
- [ ] Vues utilisées pour données dérivées
- [ ] Process de migration documenté

---

## Tableau de Décision : Créer une Nouvelle Collection ?

| Critère | Nouvelle Collection | Collection Existante |
|---------|-------------------|---------------------|
| **Domaine métier** | Distinct et indépendant | Même domaine/similaire |
| **Structure** | Fondamentalement différente | Variations mineures |
| **Volume** | > 10,000 documents | < 10,000 documents |
| **Cycle de vie** | Indépendant | Synchronisé |
| **Requêtes** | Toujours isolées | Parfois cross-type |
| **Partage de données** | Aucun | Fréquent |
| **Croissance** | Prévisible et contrôlée | Risque d'explosion |

---

## Conclusion

Le nombre de collections dans MongoDB est un indicateur de la qualité de l'architecture :

- **< 50 collections** : Architecture propre et maintenable
- **50-100 collections** : Attention requise
- **> 100 collections** : Problème architectural probable

**Règles d'or** :
1. **Consolidation** : Préférer moins de collections avec discriminants
2. **Gouvernance** : Politique claire de création
3. **Nettoyage** : Audit et suppression réguliers
4. **Documentation** : Registre de collections maintenu
5. **Monitoring** : Suivre l'évolution du nombre

Une architecture avec trop de collections est souvent le symptôme d'une sur-normalisation ou d'une transposition directe d'un modèle relationnel. MongoDB excelle avec des collections consolidées et des documents riches.

---


⏭️ [Gestion des migrations de schéma](/21-bonnes-pratiques-anti-patterns/06-gestion-migrations-schema.md)
