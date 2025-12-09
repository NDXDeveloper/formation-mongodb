🔝 Retour au [Sommaire](/SOMMAIRE.md)

# 21.1 Conventions de Nommage

## Introduction

Les conventions de nommage sont bien plus qu'une question de style : elles constituent le vocabulaire partagé de votre application. Un nommage cohérent et significatif réduit la charge cognitive, facilite la maintenance et prévient les erreurs. Dans un système distribué comme MongoDB, où les données peuvent être consultées par différentes équipes, services et outils, la clarté du nommage devient critique.

Cette section établit des conventions éprouvées qui équilibrent lisibilité, concision et compatibilité avec les écosystèmes JavaScript/JSON et les bonnes pratiques MongoDB.

---

## Principes Fondamentaux

### Principe de Clarté
Un nom doit révéler son intention sans ambiguïté. `usr` vs `authenticatedUser` : le second ne laisse aucun doute.

### Principe de Cohérence
Une fois une convention adoptée, elle doit être appliquée uniformément dans tout le projet. L'incohérence coûte plus cher qu'une convention imparfaite mais uniforme.

### Principe de Contexte
Le contexte fourni par la hiérarchie (base de données → collection → document) évite la redondance. Dans `users.profile.email`, pas besoin de `userProfileEmail`.

### Principe de Durabilité
Les noms survivent souvent aux implémentations. Privilégiez des noms qui resteront pertinents lors des évolutions.

---

## Bases de Données

### ✅ DO : Utiliser snake_case en Minuscules

**Explication** : Les noms de bases de données MongoDB sont sensibles à la casse sur certains systèmes de fichiers. Le snake_case en minuscules garantit la portabilité.

**Convention recommandée** :
```javascript
// ✅ Bonne pratique
production_store
analytics_data
user_management
reporting_db
```

**Bénéfices** :
- Compatibilité multi-plateforme (Windows, Linux, macOS)
- Pas de problèmes de casse
- Lisibilité avec séparateurs
- Standard dans l'écosystème MongoDB

---

### ❌ DON'T : Mélanger les Casses ou Utiliser des Espaces

**Explication** : Les noms avec casse mixte ou espaces créent des problèmes de portabilité et nécessitent des échappements.

**Anti-patterns** :
```javascript
// ❌ À éviter
ProductionStore       // Problèmes de casse
production-store      // Tirets créent de l'ambiguïté
"Production Store"    // Espaces nécessitent des quotes
PRODUCTION_STORE      // Majuscules = risque de confusion
```

**Conséquences** :
- Erreurs subtiles lors du déploiement cross-platform
- Nécessite des quotes dans certaines commandes
- Incompatibilité avec certains outils
- Source de bugs difficiles à tracer

---

### ✅ DO : Utiliser des Noms Descriptifs et Métier

**Explication** : Le nom de la base de données doit refléter son domaine métier ou son objectif fonctionnel.

**Bonnes pratiques** :
```javascript
// ✅ Noms qui communiquent l'intention
ecommerce_catalog     // Catalogue e-commerce
user_authentication   // Gestion authentification
financial_transactions // Transactions financières
analytics_events      // Événements analytiques
content_management    // Gestion de contenu
```

**Critères d'un bon nom** :
- Indique clairement le domaine
- Compréhensible par un nouveau développeur
- Reflète l'organisation métier
- Évite les acronymes obscurs

---

### ❌ DON'T : Utiliser des Noms Techniques Génériques

**Explication** : Les noms trop génériques ou techniques ne communiquent pas l'intention et créent de la confusion.

**Anti-patterns** :
```javascript
// ❌ Noms sans contexte métier
db1, db2, database    // Aucune information
test, temp, data      // Trop vague
mongo_db, nosql_db    // Redondant (on sait déjà que c'est MongoDB)
main, primary, app    // Générique, non descriptif
```

**Conséquences** :
- Impossible de comprendre l'objectif sans documentation
- Confusion dans les environnements multi-bases
- Difficulté pour nouveaux développeurs
- Risque d'erreurs lors des opérations

---

### ✅ DO : Inclure l'Environnement en Suffixe (optionnel mais recommandé)

**Explication** : Pour éviter les accidents, distinguez clairement les environnements dans les noms de bases.

**Convention recommandée** :
```javascript
// ✅ Environnements clairement identifiés
ecommerce_prod
ecommerce_staging
ecommerce_dev
ecommerce_test

// Ou avec préfixe
prod_ecommerce
staging_ecommerce
dev_ecommerce
```

**Bénéfices** :
- Protection contre les erreurs de connexion
- Clarté immédiate de l'environnement
- Sécurité accrue (moins de risques sur prod)

**Alternative** : Utiliser des instances MongoDB séparées par environnement (encore plus sûr).

---

## Collections

### ✅ DO : Utiliser camelCase au Pluriel

**Explication** : MongoDB recommande le camelCase pour les collections, avec un nom au pluriel pour refléter qu'elles contiennent plusieurs documents.

**Convention recommandée** :
```javascript
// ✅ Collections au pluriel en camelCase
users
products
orderItems
invoicePayments
customerAddresses
blogPosts
```

**Justification** :
- Cohérence avec l'écosystème JavaScript/JSON
- Reflète la nature "ensemble" de la collection
- Lisibilité sans séparateurs
- Standard MongoDB officiel

---

### ❌ DON'T : Mélanger Singulier/Pluriel ou Utiliser snake_case

**Explication** : L'incohérence entre singulier et pluriel crée de la confusion et des erreurs.

**Anti-patterns** :
```javascript
// ❌ Incohérence et styles inadaptés
user              // Singulier (alors que contient plusieurs users)
Products          // Majuscule initiale
order_items       // snake_case (base de données style)
Invoice-Payments  // Kebab-case avec majuscule
CUSTOMERS         // Tout en majuscules
```

**Conséquences** :
- Confusion : `user` est une collection ou un document ?
- Erreurs dans les requêtes par oubli du pluriel
- Code moins prévisible
- Friction avec les conventions JavaScript

---

### ✅ DO : Utiliser des Noms d'Entités Métier

**Explication** : Les noms de collections doivent correspondre aux concepts du domaine métier, pas aux détails d'implémentation.

**Bonnes pratiques** :
```javascript
// ✅ Vocabulaire métier clair
customers           // Pas "clients" si le métier parle de customers
subscriptions       // Pas "recurring_payments"
shipments          // Pas "delivery_records"
appointments       // Pas "calendar_slots"
```

**Avantages** :
- Communication fluide avec les équipes métier
- Documentation auto-explicative
- Alignement avec le langage ubiquitaire (DDD)
- Maintenance facilitée

---

### ❌ DON'T : Encoder des Métadonnées dans les Noms

**Explication** : Les préfixes techniques ou les métadonnées dans les noms de collections nuisent à la lisibilité.

**Anti-patterns** :
```javascript
// ❌ Métadonnées dans le nom
tbl_users           // Préfixe "table" (héritage SQL)
col_products        // Préfixe "collection" (redondant)
v2_orders          // Version dans le nom (problématique)
temp_customers     // État temporaire (collection devrait être permanente)
backup_users_20240115  // Date dans le nom
```

**Conséquences** :
- Noms encombrés et moins lisibles
- Difficulté de migration (v2 devient v3, etc.)
- Les "temp" deviennent permanentes
- Maintenance complexe

**Alternative pour les versions** : Utiliser un champ `schemaVersion` dans les documents.

---

### ✅ DO : Préfixer pour les Collections Système (si nécessaire)

**Explication** : Les collections techniques ou système peuvent bénéficier d'un préfixe pour les distinguer clairement.

**Usage approprié** :
```javascript
// ✅ Préfixe pour collections système/technique
_migrations         // Collections internes système
_audit_logs         // Logs système
_sessions          // Gestion des sessions techniques

// Collections métier (pas de préfixe)
users
products
orders
```

**Règle** : Seules les collections purement techniques méritent un préfixe. Les collections métier n'en ont jamais besoin.

---

## Champs (Propriétés des Documents)

### ✅ DO : Utiliser camelCase pour les Champs

**Explication** : Le camelCase est le standard dans l'écosystème JavaScript/JSON et facilite l'intégration avec le code applicatif.

**Convention recommandée** :
```javascript
// ✅ Champs en camelCase
{
  _id: ObjectId("..."),
  firstName: "Alice",
  lastName: "Smith",
  emailAddress: "alice@example.com",
  createdAt: ISODate("2024-01-15"),
  lastLoginDate: ISODate("2024-01-16"),
  isActive: true,
  accountBalance: 1500.50
}
```

**Avantages** :
- Mapping direct avec JavaScript (pas de conversion)
- Standard JSON et REST API
- Lisible et compact
- Cohérent avec les objets JavaScript

---

### ❌ DON'T : Utiliser snake_case ou Mélanger les Styles

**Explication** : Le snake_case force des conversions constantes entre base et application, source d'erreurs et de friction.

**Anti-patterns** :
```javascript
// ❌ Styles inadaptés ou incohérents
{
  first_name: "Alice",      // snake_case (style SQL)
  LastName: "Smith",         // PascalCase
  email_address: "...",      // Mélange avec camelCase
  created_at: "...",         // Incohérence
  IsActive: true             // PascalCase
}
```

**Conséquences** :
- Nécessite des transformations constantes (snake_case ↔ camelCase)
- Code de mapping complexe et source d'erreurs
- Incohérence avec les APIs JavaScript
- Performance dégradée par les conversions

**Note** : Si vous migrez depuis SQL, investissez dans une couche de transformation unique plutôt que de garder le snake_case.

---

### ✅ DO : Choisir des Noms Explicites et Non Ambigus

**Explication** : Un nom de champ doit être immédiatement compréhensible sans documentation.

**Bonnes pratiques** :
```javascript
// ✅ Noms clairs et précis
{
  orderDate: ISODate("..."),           // Date de la commande
  shippingAddress: {...},              // Adresse de livraison
  totalAmountInCents: 2999,           // Montant en centimes (précision)
  isEmailVerified: true,               // État de vérification
  lastPasswordChangeDate: ISODate("..."), // Date du dernier changement
  retryCount: 3                        // Nombre de tentatives
}
```

**Principes** :
- Éviter les ambiguïtés (date = création ? modification ?)
- Inclure l'unité si pertinent (InCents, InSeconds)
- Utiliser des booléens clairs (is*, has*, can*)
- Nommer les actions passées au passé (lastModified, not lastModify)

---

### ❌ DON'T : Utiliser des Abréviations Obscures

**Explication** : Les abréviations économisent quelques caractères mais coûtent cher en clarté et maintenance.

**Anti-patterns** :
```javascript
// ❌ Abréviations cryptiques
{
  fName: "Alice",           // firstName est plus clair
  addr: {...},              // address
  qty: 5,                   // quantity
  amt: 99.99,              // amount
  ts: 1705334400,          // timestamp
  usr: "alice123",         // user
  pwd: "...",              // password (et sensible!)
  dt: ISODate("..."),      // date (date de quoi?)
  no: "123",               // number (numéro de quoi?)
}
```

**Conséquences** :
- Charge cognitive pour déchiffrer
- Ambiguïté (dt = date? data? datetime?)
- Difficile pour nouveaux développeurs
- Erreurs lors de l'utilisation

**Exception** : Abréviations universelles et claires comme `id`, `url`, `html`, `api`.

---

### ✅ DO : Utiliser des Préfixes pour les Booléens

**Explication** : Les booléens avec préfixes `is`, `has`, `can`, `should` sont immédiatement identifiables et leur intention est claire.

**Convention recommandée** :
```javascript
// ✅ Booléens avec préfixes clairs
{
  isActive: true,
  isDeleted: false,
  isEmailVerified: true,
  hasSubscription: true,
  hasPremiumFeatures: false,
  canEditProfile: true,
  canAccessAdmin: false,
  shouldNotifyUser: true
}
```

**Avantages** :
- Type immédiatement identifiable
- Intention claire (état, capacité, possession)
- Code plus lisible : `if (user.isActive)`
- Convention universelle

---

### ❌ DON'T : Utiliser des Booléens Ambigus

**Explication** : Les booléens sans préfixe ou avec des noms ambigus créent de la confusion.

**Anti-patterns** :
```javascript
// ❌ Noms ambigus ou confus
{
  active: true,              // active ou isActive?
  deleted: false,            // État ou action?
  status: true,              // Quoi? Quel status?
  premium: 1,                // 1 = booléen? entier?
  verified: "yes",           // Devrait être booléen
  admin: false,              // isAdmin ou adminId?
  enabled: "true"            // String au lieu de boolean
}
```

**Conséquences** :
- Confusion sur le type (string? number? boolean?)
- Lecture ambiguë du code
- Erreurs lors des comparaisons
- Incohérence dans la base

---

### ✅ DO : Être Cohérent avec les Timestamps

**Explication** : Établissez une convention claire pour les dates et timestamps, et respectez-la partout.

**Convention recommandée** :
```javascript
// ✅ Convention cohérente pour les dates
{
  createdAt: ISODate("2024-01-15T10:30:00Z"),     // Date de création
  updatedAt: ISODate("2024-01-16T15:45:00Z"),     // Dernière modification
  deletedAt: null,                                 // Soft delete (null si actif)
  publishedAt: ISODate("2024-01-15T12:00:00Z"),   // Date de publication
  expiresAt: ISODate("2025-01-15T00:00:00Z")      // Date d'expiration
}
```

**Standards** :
- Suffixe `At` pour les timestamps
- Type `ISODate` (BSON date)
- UTC exclusivement
- null pour "pas encore défini"

---

### ❌ DON'T : Mélanger Formats et Conventions de Dates

**Explication** : L'incohérence dans les formats de dates est une source majeure de bugs.

**Anti-patterns** :
```javascript
// ❌ Incohérence de formats
{
  created: "2024-01-15",              // String ISO
  updated: 1705334400000,             // Timestamp Unix (ms)
  deleted: "15/01/2024",              // Format régional
  published: ISODate("..."),          // BSON Date
  expires: "2024-01-15 10:30:00",     // String non ISO
  lastLogin: new Date().toString(),   // String textuel
  createdDate: "...",                 // created_at mélangé avec createdDate
  modificationTime: "..."             // updated_at vs modificationTime
}
```

**Conséquences** :
- Bugs de timezone
- Comparaisons impossibles ou fausses
- Tri incorrect
- Parsing complexe et coûteux
- Erreurs lors des migrations

**Règle d'or** : Un seul format (ISODate recommandé), une seule convention de nommage.

---

### ✅ DO : Préfixer les Champs Privés/Internes avec Underscore

**Explication** : Les champs techniques ou internes peuvent être préfixés par `_` pour signaler qu'ils ne font pas partie de l'API publique.

**Usage approprié** :
```javascript
// ✅ Champs internes préfixés
{
  _id: ObjectId("..."),               // MongoDB standard
  _schemaVersion: 2,                  // Version interne du schéma
  _migrationDate: ISODate("..."),     // Métadonnées de migration
  _auditLog: [...],                   // Données d'audit interne

  // Champs publics/métier (pas de préfixe)
  userId: "user123",
  name: "Alice",
  email: "alice@example.com"
}
```

**Convention** :
- `_` uniquement pour les champs techniques/système
- Les champs métier n'ont jamais de `_` (sauf `_id`)
- Signal clair pour les développeurs

---

### ❌ DON'T : Utiliser des Caractères Spéciaux

**Explication** : Les caractères spéciaux dans les noms de champs causent des problèmes d'accès et de compatibilité.

**Anti-patterns** :
```javascript
// ❌ Caractères problématiques
{
  "user.name": "Alice",         // Point dans le nom (confusion avec nested)
  "user-id": "123",             // Tiret (problème avec opérateurs)
  "user$id": "456",             // Dollar (réservé MongoDB)
  "email@address": "...",       // @ (non standard)
  "first name": "Alice",        // Espace (nécessite quotes)
  "Prénom": "Alice"             // Accents (compatibilité)
}
```

**Conséquences** :
- Nécessite des syntaxes spéciales pour l'accès
- Incompatibilité avec certains drivers
- Confusion avec la notation pointée
- Problèmes dans les agrégations

**Règle** : Caractères alphanumériques et underscore uniquement.

---

## Noms d'Index

### ✅ DO : Nommer les Index de Façon Descriptive

**Explication** : Les noms d'index doivent clairement indiquer les champs indexés et leur ordre/type.

**Convention recommandée** :
```javascript
// ✅ Noms d'index descriptifs
db.users.createIndex(
  { email: 1 },
  { name: "email_1" }  // MongoDB génère automatiquement
);

db.products.createIndex(
  { category: 1, price: -1 },
  { name: "category_1_price_-1" }  // Ordre et direction clairs
);

db.orders.createIndex(
  { customerId: 1, createdAt: -1 },
  { name: "customerId_1_createdAt_-1" }
);

// Pour les index spécialisés
db.articles.createIndex(
  { title: "text", content: "text" },
  { name: "text_search_title_content" }
);

db.locations.createIndex(
  { coordinates: "2dsphere" },
  { name: "geo_coordinates_2dsphere" }
);
```

**Alternative personnalisée** :
```javascript
// Noms personnalisés descriptifs
db.users.createIndex(
  { email: 1 },
  { name: "idx_users_email_unique", unique: true }
);

db.orders.createIndex(
  { customerId: 1, status: 1, createdAt: -1 },
  { name: "idx_orders_customer_status_date" }
);
```

---

### ❌ DON'T : Laisser MongoDB Générer des Noms Trop Longs

**Explication** : MongoDB génère des noms d'index basés sur les champs, ce qui peut créer des noms très longs pour les index composés complexes.

**Problème** :
```javascript
// ❌ Nom auto-généré trop long
db.analytics.createIndex({
  userId: 1,
  eventType: 1,
  timestamp: -1,
  sessionId: 1,
  deviceType: 1
});
// Génère: "userId_1_eventType_1_timestamp_-1_sessionId_1_deviceType_1"
// Longueur: 61 caractères
```

**Conséquences** :
- Noms difficiles à manipuler dans les commandes
- Logs moins lisibles
- Limite de longueur de namespace (127 caractères total)

**Solution** :
```javascript
// ✅ Nom personnalisé concis
db.analytics.createIndex(
  { userId: 1, eventType: 1, timestamp: -1, sessionId: 1, deviceType: 1 },
  { name: "idx_analytics_user_events" }
);
```

---

### ✅ DO : Préfixer avec `idx_` pour Clarté

**Explication** : Un préfixe `idx_` identifie immédiatement les index dans les listings et les logs.

**Convention recommandée** :
```javascript
// ✅ Préfixe standard
idx_users_email
idx_orders_customer_date
idx_products_category_price
idx_sessions_token_expires

// Avec suffixe pour propriétés spéciales
idx_users_email_unique
idx_products_text_search
idx_locations_geo_2dsphere
idx_events_ttl
```

**Avantages** :
- Identification rapide dans les listings
- Tri naturel avec les préfixes
- Convention claire pour toute l'équipe

---

## Variables et Constantes dans le Code

### ✅ DO : Utiliser camelCase pour les Variables

**Explication** : Cohérence avec JavaScript et le reste de la codebase.

**Bonnes pratiques** :
```javascript
// ✅ Variables en camelCase
const userId = "user123";
const orderCollection = db.collection('orders');
const aggregationPipeline = [...];
const isValidUser = true;
const maxRetryCount = 3;
```

---

### ✅ DO : Utiliser UPPER_SNAKE_CASE pour les Constantes

**Explication** : Les vraies constantes (valeurs qui ne changent jamais) sont en majuscules.

**Convention** :
```javascript
// ✅ Constantes en UPPER_SNAKE_CASE
const MAX_DOCUMENT_SIZE = 16 * 1024 * 1024; // 16 MB
const DEFAULT_PAGE_SIZE = 20;
const CONNECTION_TIMEOUT_MS = 30000;
const BCRYPT_SALT_ROUNDS = 10;

// Configuration
const DB_NAME = process.env.DB_NAME || 'production_store';
const COLLECTION_USERS = 'users';
const COLLECTION_ORDERS = 'orders';
```

---

### ❌ DON'T : Utiliser des Noms de Variables Trop Courts

**Explication** : Les noms de variables doivent être descriptifs, sauf dans des contextes très limités.

**Anti-patterns** :
```javascript
// ❌ Trop courts et ambigus
const u = db.collection('users');    // user? users? collection?
const d = new Date();                // date? document? data?
const r = await collection.find();   // result? record? rows?
const n = users.length;              // number? name?

// ✅ Descriptifs et clairs
const usersCollection = db.collection('users');
const currentDate = new Date();
const searchResults = await collection.find();
const userCount = users.length;
```

**Exceptions acceptables** :
```javascript
// ✅ Contexte très limité (boucles, callbacks)
for (let i = 0; i < items.length; i++) { ... }
users.map(u => u.name)  // Arrow function simple
items.forEach((item, idx) => { ... })
```

---

## Patterns Spéciaux

### ✅ DO : Utiliser des Suffixes Cohérents pour les Types

**Explication** : Les suffixes standardisés aident à identifier rapidement le type ou l'usage d'une variable.

**Suffixes recommandés** :
```javascript
// ✅ Suffixes standard
const usersCollection = db.collection('users');
const usersList = await users.find().toArray();
const usersArray = [...];
const userCount = users.countDocuments();
const userMap = new Map();
const userSet = new Set();

const orderDocument = await orders.findOne({ _id: orderId });
const ordersQuery = { status: 'pending' };
const ordersProjection = { _id: 0, orderId: 1, total: 1 };
const ordersPipeline = [
  { $match: { status: 'pending' } },
  { $group: { ... } }
];
```

---

### ✅ DO : Utiliser des Préfixes pour les Fonctions

**Explication** : Les verbes en préfixe clarifient l'action de la fonction.

**Conventions** :
```javascript
// ✅ Verbes d'action clairs
async function getUser(userId) { ... }
async function findUserByEmail(email) { ... }
async function createUser(userData) { ... }
async function updateUserProfile(userId, updates) { ... }
async function deleteUser(userId) { ... }
async function validateUserEmail(email) { ... }
async function calculateOrderTotal(items) { ... }
async function isUserActive(userId) { ... }
async function hasUserPermission(userId, permission) { ... }
```

**Verbes standard** :
- `get` : récupération par ID (une seule entité)
- `find` : recherche avec critères
- `create` : création nouvelle entité
- `update` : modification
- `delete` / `remove` : suppression
- `validate` : validation
- `calculate` : calcul
- `is` / `has` / `can` : vérifications booléennes

---

## Cas Spéciaux et Exceptions

### Champs MongoDB Réservés

**À respecter** :
```javascript
{
  _id: ObjectId("..."),      // ID MongoDB (obligatoire)
  _v: 1                       // Version (si vous utilisez Mongoose)
}
```

**Ne pas utiliser** :
- Champs commençant par `$` (réservé aux opérateurs)
- `__proto__`, `constructor` (dangereux en JavaScript)

---

### Collections Système MongoDB

**À ne pas modifier** :
```
system.indexes
system.users
system.roles
system.version
system.namespaces
```

---

## Checklist Conventions de Nommage

### Bases de Données
- [ ] snake_case en minuscules
- [ ] Noms descriptifs et métier
- [ ] Environnement identifié (optionnel)
- [ ] Pas d'espaces ni caractères spéciaux

### Collections
- [ ] camelCase au pluriel
- [ ] Vocabulaire métier clair
- [ ] Cohérence singulier/pluriel
- [ ] Pas de métadonnées dans le nom

### Champs
- [ ] camelCase pour les propriétés
- [ ] Noms explicites, pas d'abréviations obscures
- [ ] Booléens préfixés (is*, has*, can*)
- [ ] Timestamps cohérents (*At)
- [ ] Unités précisées si besoin (*InCents, *InSeconds)
- [ ] Caractères alphanumériques uniquement

### Index
- [ ] Noms descriptifs
- [ ] Préfixe `idx_` (optionnel mais recommandé)
- [ ] Pas trop longs
- [ ] Propriétés spéciales en suffixe (_unique, _ttl)

### Code
- [ ] Variables en camelCase
- [ ] Constantes en UPPER_SNAKE_CASE
- [ ] Fonctions avec verbes d'action
- [ ] Pas de noms trop courts (sauf contextes limités)
- [ ] Suffixes cohérents pour les types

---

## Tableaux de Référence Rapide

### Comparaison des Styles

| Style | Exemple | Usage MongoDB |
|-------|---------|---------------|
| camelCase | `firstName`, `orderDate` | ✅ **Collections et champs** |
| PascalCase | `FirstName`, `OrderDate` | ❌ Éviter |
| snake_case | `first_name`, `order_date` | ✅ **Bases de données uniquement** |
| kebab-case | `first-name`, `order-date` | ❌ Éviter |
| UPPER_SNAKE_CASE | `MAX_SIZE`, `DB_NAME` | ✅ **Constantes code** |

---

### Préfixes et Suffixes Standards

| Préfixe/Suffixe | Usage | Exemple |
|-----------------|-------|---------|
| `is*` | Booléen - état | `isActive`, `isDeleted` |
| `has*` | Booléen - possession | `hasSubscription`, `hasPaid` |
| `can*` | Booléen - capacité | `canEdit`, `canAccess` |
| `*At` | Timestamp | `createdAt`, `updatedAt` |
| `*Date` | Date (sans heure) | `birthDate`, `expiryDate` |
| `*Count` | Nombre/quantité | `retryCount`, `pageCount` |
| `*Id` | Identifiant | `userId`, `orderId` |
| `*List` / `*Array` | Collection | `usersList`, `itemsArray` |
| `idx_*` | Index | `idx_users_email` |
| `_*` | Champ interne | `_schemaVersion`, `_audit` |

---

### Verbes d'Action Recommandés

| Verbe | Signification | Exemple |
|-------|--------------|---------|
| `get` | Récupérer par ID | `getUser(id)` |
| `find` | Rechercher | `findUsersByStatus(status)` |
| `create` | Créer | `createOrder(data)` |
| `update` | Modifier | `updateProfile(id, data)` |
| `delete` / `remove` | Supprimer | `deleteUser(id)` |
| `validate` | Valider | `validateEmail(email)` |
| `calculate` | Calculer | `calculateTotal(items)` |
| `fetch` | Récupérer (API externe) | `fetchUserData(api)` |
| `build` | Construire | `buildQuery(filters)` |
| `parse` | Parser | `parseDate(string)` |

---

## Impact sur la Qualité du Projet

### Bénéfices Mesurables

**Cohérence des conventions** :
- ⏱️ Réduction de 30-50% du temps de compréhension du code
- 🐛 Diminution de 20-40% des bugs de nommage/typo
- 👥 Onboarding nouveaux développeurs 2-3x plus rapide
- 📖 Documentation auto-explicative

**Maintenance** :
- Refactoring facilité
- Recherche dans le code plus efficace
- Moins de confusion dans les revues de code

**Collaboration** :
- Communication équipe améliorée
- Moins de questions/clarifications nécessaires
- Code reviews plus rapides

---

## Conclusion

Les conventions de nommage ne sont pas une contrainte arbitraire mais un investissement dans la qualité et la maintenabilité du projet. Une base de code avec un nommage cohérent et clair est :

- **Plus facile à comprendre** : Un nouveau développeur peut naviguer le code sans documentation extensive
- **Plus sûre** : Les intentions claires réduisent les erreurs
- **Plus maintenable** : Les changements sont moins risqués
- **Plus professionnelle** : Reflète la rigueur de l'équipe

**Règle d'or** : Établissez vos conventions tôt, documentez-les, et appliquez-les rigoureusement. La cohérence vaut mieux que la perfection.

---


⏭️ [Gestion des _id](/21-bonnes-pratiques-anti-patterns/02-gestion-ids.md)
